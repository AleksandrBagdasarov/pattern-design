# Balking

**Category:** Concurrency · **Also known as:** Skip-if-busy, Try-lock-or-return, Idempotent trigger · **Related:** Guarded Suspension, Monitor Object, Scheduler, Rate Limiter, State

## Intent
If an object is not in the right state to perform an operation (already running, already done, not yet ready), return immediately without doing anything instead of waiting or failing loudly.

## Problem
- A "Sync now" button or a cron tick starts a long export while the previous one is still running; two runs overlap and corrupt state or double-charge.
- `close()` / `start()` / `save()` are called multiple times and the second call breaks (double close, double subscription).
- A periodic job is triggered every minute but sometimes takes 3 minutes; queued runs pile up.

Scenario: an admin dashboard has a "Recalculate commissions" button. The calculation takes 2 minutes. Clicking twice, or a cron firing while an admin clicked, must not run it twice.

## Solution and structure
```
request() ──► with lock:
                 if state != READY: return  (balk)
                 state = RUNNING
              do work (outside the lock)
              with lock: state = READY
```
- **Guard**: a state check (`running`, `dirty`, `closed`).
- **Atomic test-and-set**: check and change the state under one lock, or use a try-lock (`lock.acquire(blocking=False)`), so two callers cannot both pass.
- **Balk response**: return `None`/`False`, return the in-flight result, or log; it is a normal outcome, not an error.

Balking differs from Guarded Suspension only in what happens when the guard is false: suspension waits, balking leaves.

## When to use
- Operations that must not overlap and where a skipped call loses nothing (the running call will produce an up-to-date result anyway).
- Idempotent lifecycle methods: `start`, `stop`, `close`, `dispose`.
- "Save if dirty": skip writes when nothing changed.
- UI actions that should ignore repeated clicks while a request is in flight.

## When NOT to use
- The caller needs the work to happen (each payment must be processed). Queue it (Producer-Consumer) or wait (Guarded Suspension).
- Every trigger carries different input that must not be dropped (each webhook event). Balking would lose data.
- Callers need the result of the in-flight run: share the in-flight Future/Promise instead of returning nothing.
- Across processes or replicas, an in-memory flag does not help; use a DB lock (`pg_try_advisory_lock`), a Redis `SET NX PX` lock, or `concurrencyPolicy: Forbid`.

## Example — Python
A non-blocking try-lock is the cleanest balk; the asyncio variant uses `Lock.locked()` safely because there is no `await` between check and acquire.
```python
import asyncio, threading, logging
from collections.abc import Awaitable, Callable

log = logging.getLogger(__name__)

class CommissionJob:
    """Threaded: overlapping calls balk."""
    def __init__(self, recalc: Callable[[], int]) -> None:
        self._recalc = recalc
        self._running = threading.Lock()

    def trigger(self) -> int | None:
        if not self._running.acquire(blocking=False):     # atomic test-and-set
            log.info("commission recalculation already running; skipped")
            return None
        try:
            return self._recalc()
        finally:
            self._running.release()

class AsyncSingleFlight:
    """asyncio: balk, but let late callers await the run already in progress."""
    def __init__(self, work: Callable[[], Awaitable[int]]) -> None:
        self._work = work
        self._current: asyncio.Task[int] | None = None

    async def trigger(self) -> int:
        if self._current is None or self._current.done():   # check-and-set: no await in between
            self._current = asyncio.create_task(self._work())
        return await self._current

class Connection:
    def __init__(self) -> None:
        self._closed = False

    def close(self) -> None:
        if self._closed:                                    # idempotent close: balk on second call
            return
        self._closed = True
        ...                                                 # release sockets
```

## Example — TypeScript
RxJS `exhaustMap` is balking as an operator: while an inner request is active, new triggers are ignored.
```ts
import { Component, inject, signal } from '@angular/core';
import { HttpClient } from '@angular/common/http';
import { Subject, exhaustMap, finalize, tap } from 'rxjs';
import { takeUntilDestroyed } from '@angular/core/rxjs-interop';

@Component({
  selector: 'app-commissions',
  template: `
    <button (click)="recalc$.next()" [disabled]="running()">Recalculate</button>
    @if (running()) { <span>Running…</span> }
  `,
})
export class Commissions {
  private http = inject(HttpClient);
  protected running = signal(false);
  protected recalc$ = new Subject<void>();

  constructor() {
    this.recalc$
      .pipe(
        exhaustMap(() => {                       // clicks during an active request are dropped
          this.running.set(true);
          return this.http.post<{ total: number }>('/api/commissions/recalculate', {}).pipe(
            finalize(() => this.running.set(false)),
          );
        }),
        tap((r) => console.log('total', r.total)),
        takeUntilDestroyed(),
      )
      .subscribe();
  }
}
```
Disabling the button is UX; `exhaustMap` is the correctness guard (keyboard, double events). The server endpoint must still balk or be idempotent, because two browser tabs bypass both.

## In the wild
- RxJS `exhaustMap`, `throttleTime` with `leading: true`.
- Python: `Lock.acquire(blocking=False)`, idempotent `close()` on files and sockets, `asyncio.Task` single-flight; `golang.org/x/sync/singleflight` is the Go equivalent.
- APScheduler `max_instances=1` (skips and logs "maximum number of running instances reached"), Celery with `celery-once` or a Redis lock, Kubernetes CronJob `concurrencyPolicy: Forbid`.
- PostgreSQL `pg_try_advisory_lock` (returns false instead of waiting); Redis `SET key value NX PX 30000`.
- Editors' autosave that skips when the document is not dirty.

## Trade-offs
| Pros | Cons |
|---|---|
| Prevents overlapping, duplicate work cheaply | Silently drops calls; callers may not notice |
| No waiting threads, no queues | Wrong when every request matters |
| Makes lifecycle methods idempotent | In-memory guard does not span processes |
| Simple to reason about | Lost trigger if state changed after the running job read its input |

## Pitfalls
- **Non-atomic check-then-set** with threads (`if not self.running: self.running = True`). Two threads pass. Use a try-lock or set under a lock.
- **Missed update.** A trigger arrives while a run is in progress but after it read the data; the balked trigger's change is never processed. Fix with a "rerun requested" flag checked at the end of the run.
- **Flag never reset** after an exception. Reset in `finally`.
- **Silent skips.** Log or count balks so operators can see them; return a distinct value to the caller.
- **Process-local guard in a multi-replica deployment.** Use a distributed lock with a TTL, and handle the lock expiring while work continues (fencing).

## Refactoring path
- **Introduce it:** find overlapping-run bugs or double-close errors. Add a state flag guarded by a try-lock, return early when busy, reset in `finally`. Add a counter/log for balks. For UI, switch the trigger stream to `exhaustMap`.
- **Remove it / simplify:** if skipped calls turned out to matter, replace balking with a queue (serialize) or with sharing the in-flight result (single-flight). If overlap is impossible by construction (single scheduler with `max_instances=1`), drop the hand-written guard.

## Related patterns
- [Guarded Suspension](../concurrency/guarded-suspension.md): waits on a false guard instead of returning.
- [Monitor Object](../concurrency/monitor-object.md): provides the lock for the atomic state check.
- [Scheduler](../concurrency/scheduler.md): overlap policy for periodic jobs is often balking.
- [Rate Limiter](../concurrency/rate-limiter.md): throttle-leading drops calls by time, balking drops them by state.
- [State](../behavioral/state.md): when the object has many states with different allowed operations.
