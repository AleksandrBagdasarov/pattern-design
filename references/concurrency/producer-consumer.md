# Producer-Consumer

**Category:** Concurrency · **Also known as:** Bounded Buffer, Work Queue, Pipeline stage · **Related:** Thread Pool, Guarded Suspension, Message Queue, Pipes and Filters, Task Queue

## Intent
Decouple the code that produces work items from the code that consumes them with a queue between them, so each side runs at its own pace and a bounded queue applies backpressure when consumers fall behind.

## Problem
- A crawler fetches pages faster than the parser can process them; memory grows until the process is OOM-killed.
- A webhook handler does slow work inline (send email, resize image), so request latency depends on the slowest downstream.
- Producers and consumers have different concurrency needs (1 Kafka reader, 20 HTTP writers) but are written as one loop.
- Shutdown loses in-flight items, or the program hangs because consumers wait forever on an empty queue.

Scenario: an import job reads a 5 GB CSV of orders and posts each row to a slow internal API. Reading is fast, posting is slow and should run 10-wide, and the file must not be loaded into RAM.

## Solution and structure
```
producer(s) ──put──► [ bounded queue, maxsize=N ] ──get──► consumer(s)
     ▲ blocks when full                         blocks when empty ▲
```
- **Producer**: creates items and `put`s them; blocks (or awaits) when the queue is full. That block is the backpressure.
- **Queue**: thread-safe or coroutine-safe FIFO with a **bound**. Unbounded queues only move the memory problem.
- **Consumer**: loops `get` → process → `task_done`.
- **Termination protocol**: sentinel values (one per consumer), `Queue.shutdown()` (3.13+), or cancelling consumers after `queue.join()`.

## When to use
- Rates differ between stages, or bursts must be smoothed.
- You want independent concurrency per stage (1 reader, N writers).
- Stages should be testable in isolation, connected only by a queue.
- In-process only. Across processes or machines, the same idea is a Message Queue / broker (SQS, RabbitMQ, Kafka).

## When NOT to use
- A simple `for` loop over a list where both sides are fast; or `asyncio.gather` / `TaskGroup` over a known set of items. A queue adds lifecycle code you then must get right.
- Work must survive a crash or deploy. In-memory queues lose items; use a durable Task Queue or broker.
- Strict request/response with a caller waiting on the result: use a Future or a Thread Pool `submit`.
- In the browser or an Angular component, an RxJS stream (`mergeMap(fn, concurrency)`) already is a producer-consumer pipeline; do not hand-roll a queue class.

## Example — Python
asyncio, bounded queue, N consumers, clean shutdown using `TaskGroup` and `Queue.shutdown()` (Python 3.13+; on 3.12 send one `None` sentinel per consumer instead).
```python
import asyncio, csv
from collections.abc import Iterator
import httpx

def read_orders(path: str) -> Iterator[dict[str, str]]:
    with open(path, newline="") as f:
        yield from csv.DictReader(f)

async def producer(q: asyncio.Queue[dict[str, str]], path: str) -> None:
    for row in read_orders(path):
        await q.put(row)          # waits while the queue is full: backpressure
    q.shutdown()                  # 3.13+: consumers drain remaining items, then get() raises QueueShutDown

async def consumer(q: asyncio.Queue[dict[str, str]], client: httpx.AsyncClient, failed: list[str]) -> None:
    while True:
        try:
            order = await q.get()
        except asyncio.QueueShutDown:
            return
        try:
            r = await client.post("/orders/import", json=order)
            r.raise_for_status()
        except httpx.HTTPError:
            failed.append(order["order_id"])  # never let one item kill the consumer
        finally:
            q.task_done()

async def import_orders(path: str, workers: int = 10) -> list[str]:
    q: asyncio.Queue[dict[str, str]] = asyncio.Queue(maxsize=workers * 2)
    failed: list[str] = []
    async with httpx.AsyncClient(base_url="https://orders.internal", timeout=5) as client:
        async with asyncio.TaskGroup() as tg:
            tg.create_task(producer(q, path))
            for _ in range(workers):
                tg.create_task(consumer(q, client, failed))
    return failed
```
For threads, `queue.Queue(maxsize=...)` has the same API (blocking `put`/`get`, `task_done`, `join`, and `shutdown` since 3.13). Note that the CSV read here is blocking file I/O on the loop; for large files offload it or accept it if reads are fast.

## Example — TypeScript
RxJS expresses producer-consumer with bounded concurrency; `mergeMap`'s second argument is the consumer count. Backpressure is pull-based only if the source is lazy.
```ts
import { Injectable, inject } from '@angular/core';
import { HttpClient } from '@angular/common/http';
import { from, mergeMap, catchError, of, toArray, map } from 'rxjs';

interface OrderRow { orderId: string; total: number }

@Injectable({ providedIn: 'root' })
export class OrderImportService {
  private http = inject(HttpClient);

  importAll(rows: OrderRow[], workers = 4) {
    return from(rows).pipe(                                   // producer
      mergeMap(                                               // up to `workers` consumers in flight
        (row) => this.http.post('/api/orders/import', row).pipe(
          map(() => ({ id: row.orderId, ok: true as const })),
          catchError(() => of({ id: row.orderId, ok: false as const })),
        ),
        workers,
      ),
      toArray(),
    );
  }
}
```
In Node with an unbounded source (a stream), use `for await` over a readable stream plus a concurrency limiter, or `stream.Readable.prototype.map(fn, { concurrency })` (marked experimental in Node docs); Node streams already apply backpressure via `highWaterMark`.

## In the wild
- Python: `queue.Queue`, `asyncio.Queue`, `multiprocessing.Queue`, `janus` (bridges thread and asyncio queues).
- Node: streams (`highWaterMark` backpressure), `p-queue`, BullMQ (Redis-backed, durable).
- RxJS: `mergeMap(fn, n)`, `concatMap` (n = 1), `bufferCount`.
- Distributed: Kafka consumer groups, SQS + Lambda, RabbitMQ prefetch (`basic_qos`) as the bound, Celery.
- Go channels; Java `BlockingQueue`; Kotlin `Channel`.

## Trade-offs
| Pros | Cons |
|---|---|
| Stages decoupled in speed and concurrency | Termination and error handling are easy to get wrong |
| Bounded queue gives natural backpressure | In-memory items are lost on crash |
| Smooths bursts | Extra latency per hop; ordering lost with N consumers |
| Each stage testable alone | Harder to trace an item end-to-end |

## Pitfalls
- **Unbounded queue** (`maxsize=0`). Hides the slowness until OOM. Always set a bound.
- **Consumer dies on first exception** and the producer then blocks forever on a full queue. Catch per item, always `task_done()` in `finally`.
- **Hang on shutdown.** Consumers wait on `get()` forever. Use `shutdown()`, one sentinel per consumer, or cancel them after `q.join()`.
- **Ordering assumptions.** With more than one consumer, completion order differs from input order. Use a single consumer per key (partition by key, as Kafka does) if order matters.
- **Poison message** that fails every retry. Cap retries and route it to a dead-letter list.
- **Blocking work in async consumers.** A sync DB call inside an asyncio consumer stalls every consumer. Use async drivers or `to_thread`.

## Refactoring path
- **Introduce it:** start from an inline loop "read → process". Extract `process(item)`. Put a bounded queue between reading and processing with one consumer and verify behavior. Raise the consumer count. Add the shutdown protocol and per-item error handling. Move to a durable broker only when items must survive restarts.
- **Remove it / simplify:** if the item set is small and known, replace producer, queue and consumers with `TaskGroup` + `Semaphore`, or a `ThreadPoolExecutor.map`.

## Related patterns
- [Thread Pool](../concurrency/thread-pool.md): the consumers are often a pool; the executor hides its own queue.
- [Guarded Suspension](../concurrency/guarded-suspension.md): the blocking `get`/`put` are guarded suspensions on "not empty"/"not full".
- [Message Queue](../enterprise/message-queue.md): the durable, cross-process form.
- [Pipes and Filters](../architectural/pipes-and-filters.md): a chain of producer-consumer stages.
- [Task Queue](../emergent/task-queue.md): durable background jobs with retries.
- [Rate Limiter](../concurrency/rate-limiter.md): throttles consumers against a downstream quota.
