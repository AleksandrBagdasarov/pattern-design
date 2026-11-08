# Write-Behind Cache

**Category:** Optimization · **Also known as:** Write-back cache, buffered writes, write coalescing · **Related:** Read-Through Cache, Caching, Producer-Consumer, Message Queue, Unit of Work

## Intent
Accept writes into a fast cache or buffer, acknowledge immediately, and flush them to the durable store asynchronously — batched and coalesced — trading durability and consistency for write throughput and latency.

## Problem
- A high-frequency write path overwhelms the database: view counters, "last seen" timestamps, telemetry, game scores, rate-limit counters, autosave drafts.
- Each write is a round-trip and a transaction; most are immediately superseded by the next write to the same key.
- The client does not need the write to be durable before getting a response.

Scenario: an article page increments `views` on every hit — 5,000 UPDATEs/second on hot rows, with lock contention. Incrementing in Redis (or in-process) and flushing aggregated deltas every 5 s turns 25,000 updates into a few hundred.

Profile first: confirm writes are the bottleneck (DB wait events, `pg_stat_statements` sorted by calls/total time, lock waits, `py-spy dump` showing workers blocked on commits) before introducing an asynchronous durability gap.

## Solution and structure
```
writer ──put(k, v)──▶ cache/buffer (dirty set) ──ack──▶ writer
                            │ every N ms or M entries (flusher)
                            ▼
                    coalesce by key ──batch upsert──▶ database
                            │ on failure: retry with backoff, keep dirty
```
- **Buffer**: holds dirty entries; coalesces multiple writes to the same key (last-write-wins or summed deltas).
- **Flusher**: periodic or size-triggered background task writing batches; retries; handles shutdown flush.
- **Reads**: must go through the cache (read-through) or accept that the DB lags behind.
- **Durability**: decide what loss is acceptable. Options: accept loss on crash (counters), use a durable buffer (Redis AOF, Kafka, a WAL), or don't use write-behind.

## When to use
- Write-heavy, loss-tolerant or aggregatable data: counters, metrics, presence, analytics events, leaderboards.
- Writes to the same key are frequent and coalescable.
- The store is much slower than the cache, or charges per write (DynamoDB WCU, external APIs).
- Frontend: autosave of drafts (debounced), batching analytics beacons.

## When NOT to use
- Money, orders, inventory, anything that must be durable when you return 200: write synchronously (write-through or plain DB write).
- Other systems read the DB directly and expect fresh data.
- Constraints/validation enforced by the DB must reject the write synchronously (unique email).
- Low write volume: the complexity buys nothing. Batch writes in the request instead (`executemany`, bulk insert).
- When you actually need a durable async pipeline: use a message queue/outbox explicitly rather than a cache with a flusher.

## Example — Python
```python
import asyncio, logging
from collections import Counter
from collections.abc import Awaitable, Callable

log = logging.getLogger(__name__)
Flush = Callable[[dict[int, int]], Awaitable[None]]

class ViewCounterBuffer:
    """Coalesces increments in memory and flushes deltas in batches. Loses at most one interval on crash."""
    def __init__(self, flush: Flush, interval_s: float = 5.0, max_pending: int = 10_000) -> None:
        self._flush, self._interval, self._max = flush, interval_s, max_pending
        self._pending: Counter[int] = Counter()
        self._task: asyncio.Task[None] | None = None
        self._wake = asyncio.Event()

    def increment(self, article_id: int, n: int = 1) -> None:   # O(1), no I/O on the request path
        self._pending[article_id] += n
        if len(self._pending) >= self._max:
            self._wake.set()

    async def _flush_once(self) -> None:
        batch, self._pending = self._pending, Counter()           # swap: new writes go to a fresh buffer
        if not batch:
            return
        try:
            await self._flush(dict(batch))
        except Exception:
            log.exception("flush failed; re-queueing %d keys", len(batch))
            self._pending.update(batch)                           # deltas merge safely; retry next tick

    async def _run(self) -> None:
        while True:
            try:
                await asyncio.wait_for(self._wake.wait(), self._interval)
            except TimeoutError:
                pass
            self._wake.clear()
            await self._flush_once()

    async def __aenter__(self) -> "ViewCounterBuffer":
        self._task = asyncio.create_task(self._run())
        return self

    async def __aexit__(self, *exc: object) -> None:
        if self._task:
            self._task.cancel()
        await self._flush_once()                                  # drain on graceful shutdown

# flush impl: UPDATE articles a SET views = a.views + d.delta FROM (VALUES …) d(id, delta) WHERE a.id = d.id
# wire into FastAPI lifespan: async with ViewCounterBuffer(flush=repo.add_views) as buf: app.state.views = buf; yield
```
Summed deltas are safe to retry only if the flush itself is atomic; if a batch may partially apply, make it idempotent (batch ids) or flush in one transaction.

## Example — TypeScript
```ts
import { Injectable, inject, DestroyRef } from '@angular/core';
import { HttpClient } from '@angular/common/http';
import { Subject, bufferTime, filter, concatMap, catchError, tap, EMPTY } from 'rxjs';

interface AnalyticsEvent { name: string; ts: number; props?: Record<string, unknown> }

@Injectable({ providedIn: 'root' })
export class AnalyticsBuffer {
  private readonly http = inject(HttpClient);
  private readonly events$ = new Subject<AnalyticsEvent>();
  private pending: AnalyticsEvent[] = [];

  constructor() {
    const sub = this.events$.pipe(
      bufferTime(5_000, null, 50),                      // flush every 5 s or every 50 events
      filter((batch) => batch.length > 0),
      concatMap((batch) =>                              // keep order; one request at a time
        this.http.post('/api/analytics/batch', batch).pipe(
          tap(() => (this.pending = this.pending.filter((e) => !batch.includes(e)))),
          catchError(() => EMPTY),                      // loss-tolerant: keep in pending for the beacon
        )),
    ).subscribe();
    inject(DestroyRef).onDestroy(() => sub.unsubscribe());
    addEventListener('pagehide', () => {               // last-chance flush; survives unload
      if (this.pending.length) navigator.sendBeacon('/api/analytics/batch', JSON.stringify(this.pending));
    });
  }

  track(name: string, props?: Record<string, unknown>): void {
    const e = { name, ts: Date.now(), props };
    this.pending.push(e);
    if (this.pending.length > 500) this.pending.shift();  // bound memory
    this.events$.next(e);
  }
}
```
This sketch tolerates loss (analytics): `pending` holds unconfirmed events so `pagehide` can beacon them; failed batches are not retried in-session. For drafts, use `debounceTime` + `switchMap` to a save endpoint plus `localStorage` as the durable buffer.

## In the wild
- Hardware CPU and disk caches (write-back), OS page cache (`fsync` forces the flush), PostgreSQL WAL + background writer.
- Hazelcast `MapStore` with `write-delay-seconds`, Apache Ignite/Coherence write-behind, Redis as counter store with periodic DB sync.
- Kafka producer `linger.ms`/`batch.size` (batching), Elasticsearch `refresh_interval`, OpenTelemetry `BatchSpanProcessor`, Python `logging.handlers.QueueHandler` + `MemoryHandler`.
- Browser `navigator.sendBeacon`, service-worker Background Sync.

## Trade-offs
| Pros | Cons |
|---|---|
| Very low write latency on the request path | Data loss window on crash unless the buffer is durable |
| Massive reduction in DB writes via coalescing/batching | DB is eventually consistent; direct DB readers see stale data |
| Smooths write spikes | Errors surface asynchronously, after the client got 200 |
| Cheaper on per-write-priced stores | Ordering, retries, idempotency and shutdown draining are tricky |

## Pitfalls
- **Unbounded buffer when the DB is down:** memory grows until OOM. Cap it; shed load or fall back to synchronous writes.
- **No drain on shutdown:** every deploy loses data. Flush in lifespan/SIGTERM handlers; give pods a termination grace period.
- **Non-idempotent retries** (re-adding deltas after a partially applied batch): single-transaction batches or idempotency keys.
- **Multiple instances each buffering the same keys:** fine for summed deltas, wrong for last-write-wins values (out-of-order overwrites). Include versions/timestamps and use conditional upserts.
- **Silent failures:** export metrics for buffer size, flush latency, flush errors, oldest dirty entry age.

## Refactoring path
- **Introduce it:** measure write load → confirm loss tolerance with product → buffer one hot write type with periodic batch flush → add metrics, cap, shutdown drain → measure DB load again.
- **Remove it / simplify:** if durability incidents appear or volume dropped, replace with synchronous batched writes per request, or move to a durable queue (Kafka/SQS) + consumer when you need both async and durability.

## Related patterns
- [Read-Through Cache](../optimization/read-through-cache.md) — read side; readers must see buffered writes.
- [Caching](../optimization/caching.md) — cache-aside for reads; write-behind changes the write path.
- [Producer-Consumer](../concurrency/producer-consumer.md) — the buffer + flusher is a producer-consumer pipeline.
- [Message Queue](../enterprise/message-queue.md) — the durable alternative when loss is unacceptable.
- [Unit of Work](../data-access/unit-of-work.md) — batches writes within one transaction; write-behind batches across requests.
