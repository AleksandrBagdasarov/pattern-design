# Active Object

**Category:** Concurrency · **Also known as:** Concurrent Object, Actor (close relative) · **Related:** Future / Promise, Producer-Consumer, Monitor Object, Command, Scheduler

## Intent
Decouple method invocation from method execution: callers' method calls become request messages placed on a queue and executed one at a time on the object's own thread (or task), and each call immediately returns a Future for its result.

## Problem
- A non-thread-safe resource (a serial device, a SQLite connection, a GUI toolkit, a legacy SDK client) is used from many threads, and sprinkling locks around every call is error-prone.
- Callers should not block while a slow operation runs, but the operations must execute in order.
- Lock-based code deadlocks because objects call each other while holding their locks.

Scenario: an audit logger writes to a local SQLite file. SQLite connections must not be shared freely across threads, and writes should be batched and ordered. Request threads should hand off a log entry and continue.

## Solution and structure
```
client ──proxy.method(args)──► [ activation queue ] ──► scheduler thread ──► servant.method(args)
   ◄──── Future ─────────────────────────────────────────────────────────── result / exception
```
- **Proxy**: public interface; turns each call into a request (a Command) and returns a Future.
- **Activation queue**: holds pending requests (Producer-Consumer).
- **Scheduler**: a single thread/task that dequeues and dispatches requests; may reorder by guard or priority.
- **Servant**: the real implementation; only ever touched by the scheduler thread, so it needs no locks.
- **Future**: carries the result back.

The **Actor model** generalizes this: each actor has a mailbox, processes one message at a time, and talks to others only by messages.

## When to use
- One resource must be accessed serially, and you want "thread confinement" rather than locks.
- Callers should fire requests asynchronously and optionally await results.
- Bridging a blocking library into an asyncio program: one dedicated worker thread owns the library.
- Stateful entities with many concurrent callers (actor systems: per-user session, per-device controller).

## When NOT to use
- In asyncio or JS, a single-threaded event loop already confines state; an `asyncio.Lock` or a simple queue usually suffices. Do not build proxy/scheduler/servant classes for code that is already single-threaded.
- The operation is fast and contention is low: a mutex (Monitor Object) is simpler and has lower latency.
- You need parallel execution of the operations themselves. Active Object serializes them; use a Thread Pool.
- Simple fire-and-forget background jobs: a `ThreadPoolExecutor(max_workers=1)` already is an active object without the ceremony.

## Example — Python
The idiomatic form: a single-worker executor owns the servant. `max_workers=1` guarantees FIFO, one-at-a-time execution on one thread.
```python
import sqlite3, threading
from concurrent.futures import Future, ThreadPoolExecutor
from dataclasses import dataclass
from datetime import datetime, UTC

@dataclass(frozen=True)
class AuditEntry:
    user_id: str
    action: str
    at: datetime

class AuditLog:
    """Active Object: public methods enqueue; the private thread owns the connection."""
    def __init__(self, path: str) -> None:
        self._local = threading.local()
        self._exec = ThreadPoolExecutor(max_workers=1, thread_name_prefix="audit",
                                        initializer=self._open, initargs=(path,))

    def _open(self, path: str) -> None:                 # runs on the worker thread
        self._local.db = sqlite3.connect(path)
        self._local.db.execute("CREATE TABLE IF NOT EXISTS audit(user_id, action, at)")

    def _write(self, e: AuditEntry) -> int:             # servant method: no locks needed
        cur = self._local.db.execute("INSERT INTO audit VALUES (?,?,?)",
                                     (e.user_id, e.action, e.at.isoformat()))
        self._local.db.commit()
        return cur.lastrowid or 0

    def record(self, user_id: str, action: str) -> Future[int]:   # proxy method
        return self._exec.submit(self._write, AuditEntry(user_id, action, datetime.now(UTC)))

    def close(self) -> None:
        self._exec.submit(lambda: self._local.db.close())
        self._exec.shutdown(wait=True)

log = AuditLog("audit.db")
fut = log.record("u-42", "refund.approved")     # returns immediately
# in async code: row_id = await asyncio.wrap_future(fut)
```
For actors in asyncio, the same idea is a class with an `asyncio.Queue` mailbox and one consumer task.

## Example — TypeScript
In Node the event loop already serializes callbacks, but async methods interleave at every `await`. A mailbox (a Promise chain) gives strict one-at-a-time execution of async operations.
```ts
export class SerialExecutor {
  private tail: Promise<unknown> = Promise.resolve();

  run<T>(task: () => Promise<T>): Promise<T> {           // returns the Future
    const result = this.tail.then(task);
    this.tail = result.catch(() => undefined);           // a failure must not poison the queue
    return result;
  }
}

export class DeviceController {                           // proxy
  private readonly mailbox = new SerialExecutor();
  constructor(private readonly port: { write(cmd: string): Promise<string> }) {}

  move(x: number, y: number): Promise<string> {
    return this.mailbox.run(() => this.port.write(`MOVE ${x} ${y}`)); // servant call, serialized
  }
  home(): Promise<string> {
    return this.mailbox.run(() => this.port.write('HOME'));
  }
}
```
In the browser, a Web Worker is an active object: `postMessage` is the request, the worker's message loop is the scheduler, and libraries like Comlink generate the Promise-returning proxy.

## In the wild
- Python: `ThreadPoolExecutor(max_workers=1)`; Qt's `QObject` thread affinity with queued signals; Tkinter's "only the main thread touches widgets" rule.
- Actor frameworks: Akka / Pekko, Orleans (virtual actors), Erlang/Elixir processes, Ray actors (`@ray.remote` classes), Dapr actors, Thespian (Python).
- JS: Web Workers + Comlink, Node `worker_threads` with `postMessage`.
- Android `Handler`/`Looper`; Swift `actor` types (language-level active objects).

## Trade-offs
| Pros | Cons |
|---|---|
| Servant code needs no locks (thread confinement) | All calls serialized: throughput capped at one worker |
| Callers never block on the operation | Queue growth under load; needs a bound or backpressure |
| Natural ordering of requests | Extra latency per call (enqueue, context switch) |
| Deadlock-free between objects that only message each other | Debugging is harder; stack traces cross threads |

## Pitfalls
- **Unbounded mailbox.** A slow servant plus fast callers means memory growth. Bound the queue or apply backpressure to callers.
- **Calling the servant's own proxy from inside the servant and waiting** on the Future: self-deadlock, because the only worker is busy waiting. Call the servant method directly.
- **Leaking servant state.** Returning mutable internals lets other threads touch them; return copies.
- **Ignored Futures** lose exceptions. Log in a done-callback.
- **Serial executor poisoned by a rejection** (TS): chain `.catch` on the tail as shown.
- **Forgetting shutdown** leaves a non-daemon thread alive at exit.

## Refactoring path
- **Introduce it:** find a shared resource guarded by many scattered locks. Move all access into one class. Give it a single-worker executor. Change public methods to `submit` and return Futures. Delete the locks. Update callers to await or ignore results deliberately.
- **Remove it / simplify:** if contention was low, replace the executor with a `Lock` and synchronous methods. If you need parallelism, switch to a Thread Pool and make the servant thread-safe or per-thread.

## Related patterns
- [Future / Promise](../concurrency/future-promise.md): what proxy methods return.
- [Producer-Consumer](../concurrency/producer-consumer.md): the activation queue is a single-consumer producer-consumer.
- [Monitor Object](../concurrency/monitor-object.md): serializes access with a lock on the caller's thread instead of a separate thread.
- [Command](../behavioral/command.md): each request in the queue is a command object.
- [Scheduler](../concurrency/scheduler.md): the dispatcher may reorder requests by priority.
- [Thread Pool](../concurrency/thread-pool.md): a single-worker pool is the minimal active object.
