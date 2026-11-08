# Scheduler

**Category:** Concurrency · **Also known as:** Job scheduler, Task scheduler, Timer queue, Cron · **Related:** Producer-Consumer, Active Object, Leader Election, Task Queue, Balking

## Intent
Separate *what* a task does from *when* and *in what order* it runs: a scheduler holds pending tasks with their due time or priority and dispatches them to workers according to an explicit policy.

## Problem
- Periodic work is implemented as `while True: do(); time.sleep(3600)` loops scattered across the code. Drift accumulates, errors kill the loop silently, and nobody knows what runs when.
- Urgent jobs (password reset emails) wait behind bulk jobs (newsletter) because everything is FIFO.
- With several app replicas, every replica runs the same "nightly" job, so customers get three invoices.
- Delayed actions ("cancel the order if unpaid in 30 minutes") are implemented with long sleeps that do not survive a restart.

Scenario: an e-commerce backend must (a) expire unpaid orders 30 minutes after creation, (b) rebuild a sitemap nightly, and (c) prioritize transactional emails over marketing ones.

## Solution and structure
```
schedule(task, at|every|priority) ──► [ priority queue ordered by (due_time, priority) ]
                                              │ timer wakes at next due time
                                              ▼
                                       dispatcher ──► worker(s) / executor
```
- **Task**: a callable plus metadata (due time, period, priority, id for dedup/cancel).
- **Queue**: a min-heap keyed by due time and/or priority (`heapq`), or a durable store (DB table, Redis sorted set, broker with delayed delivery).
- **Dispatcher**: sleeps until the earliest due time (or until a new earlier task arrives), pops due tasks, hands them to workers.
- **Policy**: FIFO, priority, deadline (EDF), fair-share per tenant, rate-limited.
- **Durability and singleton-ness** for distributed setups: persist tasks, and make sure only one node fires each occurrence (Leader Election or a DB row lock).

## When to use
- Time-based work: delays, periodic jobs, timeouts, retries with backoff.
- Ordering by priority or deadline among queued work.
- Any background job that must run once per period across a cluster.

## When NOT to use
- Building your own durable distributed scheduler. Use the platform: Kubernetes CronJob, EventBridge Scheduler, Cloud Scheduler, Celery beat, APScheduler with a DB job store, Temporal schedules, SQS delay queues.
- A single in-process timer: `loop.call_later`, `asyncio.sleep` in a task, `setTimeout`, RxJS `timer`/`interval` are enough.
- "Expire after N minutes" checks that can be lazy: store `expires_at` and filter on read, or use a DB/Redis TTL, instead of scheduling a job per entity.
- Heavy priority logic for a queue that never has more than a few items.

## Example — Python
An in-process asyncio scheduler with delayed and periodic tasks, built on `heapq`. Good for a single process; for clusters, persist tasks and add a leader.
```python
import asyncio, heapq, itertools, time, logging
from collections.abc import Awaitable, Callable
from dataclasses import dataclass, field

Job = Callable[[], Awaitable[None]]

@dataclass(order=True)
class Entry:
    due: float
    priority: int
    seq: int                                     # tie-breaker keeps heap order stable
    job: Job = field(compare=False)
    every: float | None = field(compare=False, default=None)

class Scheduler:
    def __init__(self) -> None:
        self._heap: list[Entry] = []
        self._seq = itertools.count()
        self._wakeup = asyncio.Event()

    def at(self, delay: float, job: Job, *, priority: int = 10, every: float | None = None) -> None:
        heapq.heappush(self._heap, Entry(time.monotonic() + delay, priority, next(self._seq), job, every))
        self._wakeup.set()                       # a new task may be earlier than the current sleep

    async def run(self) -> None:
        async with asyncio.TaskGroup() as tg:
            while True:
                timeout = self._heap[0].due - time.monotonic() if self._heap else None
                if timeout is None or timeout > 0:
                    self._wakeup.clear()
                    try:
                        await asyncio.wait_for(self._wakeup.wait(), timeout)
                    except TimeoutError:
                        pass
                    continue
                e = heapq.heappop(self._heap)
                tg.create_task(self._safe(e.job))
                if e.every is not None:          # schedule from the due time, not "now": no drift
                    heapq.heappush(self._heap, Entry(e.due + e.every, e.priority, next(self._seq), e.job, e.every))

    async def _safe(self, job: Job) -> None:
        try:
            await job()
        except Exception:
            logging.exception("scheduled job failed")   # one failure must not stop the scheduler
```
Same-due-time tasks run by `priority` (lower first). For real deployments prefer APScheduler / Celery beat / a cloud scheduler; the code above shows the mechanics.

## Example — TypeScript
In the browser and Angular, RxJS is the scheduler. Periodic polling with no overlap, pausing when the tab is hidden:
```ts
import { Injectable, inject, DOCUMENT } from '@angular/core'; // DOCUMENT moved to core in v20; older: @angular/common
import { HttpClient } from '@angular/common/http';
import { fromEvent, timer, switchMap, exhaustMap, startWith, map, EMPTY, shareReplay, catchError } from 'rxjs';

interface OrderStatus { id: string; state: 'pending' | 'paid' | 'expired' }

@Injectable({ providedIn: 'root' })
export class OrderStatusPoller {
  private http = inject(HttpClient);
  private doc = inject(DOCUMENT);

  status$(orderId: string) {
    const visible$ = fromEvent(this.doc, 'visibilitychange').pipe(
      startWith(null),
      map(() => this.doc.visibilityState === 'visible'),
    );
    return visible$.pipe(
      switchMap((visible) => (visible ? timer(0, 10_000) : EMPTY)),     // schedule: every 10 s
      exhaustMap(() =>                                                   // skip a tick if the last call is still running
        this.http.get<OrderStatus>(`/api/orders/${orderId}/status`).pipe(catchError(() => EMPTY)),
      ),
      shareReplay({ bufferSize: 1, refCount: true }),
    );
  }
}
```
In Node, `node-cron` or `croner` handle cron expressions in-process; BullMQ repeatable/delayed jobs (Redis) survive restarts and run once across replicas.

## In the wild
- Python: `sched`, `asyncio` `call_later`/`call_at`, APScheduler, Celery beat and `apply_async(countdown=...)`, RQ Scheduler, Airflow/Prefect/Dagster (DAG schedulers).
- JS: `setTimeout`/`setInterval`, RxJS schedulers (`asyncScheduler`, `animationFrameScheduler`), `requestIdleCallback`, BullMQ, Agenda.
- Infrastructure: cron, systemd timers, Kubernetes CronJob (with `concurrencyPolicy: Forbid`), AWS EventBridge Scheduler, GCP Cloud Scheduler, Temporal schedules.
- OS and runtimes: the OS CPU scheduler, Go's goroutine scheduler, the asyncio event loop itself.

## Trade-offs
| Pros | Cons |
|---|---|
| One place defines timing and ordering policy | In-memory schedules vanish on restart |
| Priorities stop bulk work starving urgent work | Priority can starve low-priority work (needs aging) |
| Drift-free periodic execution | Clustered runs need dedup / leader election |
| Easy to observe and cancel jobs | Clock changes, DST and time zones bite cron jobs |

## Pitfalls
- **Duplicate runs with multiple replicas.** Use a single scheduler deployment, a leader, `SELECT ... FOR UPDATE SKIP LOCKED`, or a managed scheduler; make jobs idempotent anyway.
- **Overlapping runs** when a job takes longer than its period. Skip (balk), queue, or set `concurrencyPolicy: Forbid` / `max_instances=1`.
- **Drift** from `sleep(period)` after the work. Compute the next due time from the previous due time.
- **Wall clock vs monotonic clock.** Use monotonic time for delays; use time-zone-aware wall clock (and know DST behavior) for cron.
- **Silent death** of a scheduler loop on an exception. Catch per job, log, alert on missed runs (heartbeat metric).
- **Thundering herd** at midnight. Add jitter.

## Refactoring path
- **Introduce it:** inventory every `sleep` loop and `setInterval`. Move them into one scheduler (library or platform) with named jobs, then add error handling, overlap policy and metrics. When you scale to more than one replica, add persistence and dedup.
- **Remove it / simplify:** replace per-entity timers with TTLs or lazy expiry checks; replace a homegrown scheduler with the platform's (CronJob, EventBridge, Celery beat).

## Related patterns
- [Producer-Consumer](../concurrency/producer-consumer.md): the scheduler produces due tasks for a worker pool.
- [Active Object](../concurrency/active-object.md): its dispatcher is a scheduler over method requests.
- [Leader Election](../cloud/leader-election.md): ensures a clustered schedule fires once.
- [Task Queue](../emergent/task-queue.md): durable delayed and retried jobs.
- [Balking](../concurrency/balking.md): skip a run when the previous one is still active.
- [Rate Limiter](../concurrency/rate-limiter.md): a scheduler policy that caps throughput.
