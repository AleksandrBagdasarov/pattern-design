# Guarded Suspension

**Category:** Concurrency · **Also known as:** Guarded wait, Condition wait, Wait-until · **Related:** Monitor Object, Balking, Producer-Consumer, Future / Promise, Barrier

## Intent
Make an operation wait (suspend) until a precondition on shared state becomes true, instead of failing or busy-polling, and resume it when another thread or task changes that state.

## Problem
- Code polls in a loop: `while not ready: time.sleep(0.1)`. It wastes CPU, adds up to 100 ms latency, and still races with the writer.
- A method has a precondition that cannot hold yet (the buffer is empty, the connection is not authenticated, the config is not loaded), and the caller would rather wait than get an error.
- A naive `if not ready: event.wait()` misses wake-ups or proceeds on a stale check because the state changed between the check and the action.

Scenario: request handlers need an OAuth access token. A background task refreshes it. While a refresh is in progress, handlers should wait for the new token rather than use an expired one or start their own refresh.

## Solution and structure
```
caller ──acquire lock──► while not guard(): cond.wait()  ──► act on state ──► release
writer ──acquire lock──► mutate state ──► cond.notify_all() ──► release
```
- **Guarded object**: holds the shared state and a lock.
- **Guard**: a predicate over that state (`self._token is not None and not expired`).
- **Condition variable**: lets waiters release the lock and sleep until notified.
- **Notifier**: any method that changes state relevant to the guard calls `notify`/`notify_all` while holding the lock.

Two rules make it correct: check the guard **in a loop** (spurious wake-ups, and another waiter may have consumed the state first), and check and act **under the same lock**. `Condition.wait_for(predicate)` does the loop for you.

## When to use
- A caller can do nothing useful until some shared state is ready, and the wait is expected to be short or bounded.
- Implementing blocking structures: bounded buffers, one-shot latches, "wait until the cache is warm".
- Coordinating asyncio tasks around shared state (`asyncio.Condition`, `asyncio.Event`).

## When NOT to use
- The precondition is unlikely to become true soon. Fail fast or skip instead (see Balking).
- Waiting for a single result of a single operation. A Future / Promise is simpler than a condition variable.
- A one-way "it happened" flag. `threading.Event` / `asyncio.Event` is enough; you do not need a full Condition.
- In browser JS there is no blocking wait on the main thread. Model it as a Promise that resolves when the state changes, or an observable you `filter` + `take(1)`.

## Example — Python
asyncio version; the threading version is identical with `threading.Condition` and no `await`.
```python
import asyncio, time
from dataclasses import dataclass
from collections.abc import Awaitable, Callable

@dataclass(frozen=True)
class Token:
    value: str
    expires_at: float

    def valid(self) -> bool:
        return time.monotonic() < self.expires_at - 30   # 30 s safety margin

class TokenHolder:
    def __init__(self, fetch: Callable[[], Awaitable[Token]]) -> None:
        self._fetch = fetch
        self._token: Token | None = None
        self._refreshing = False
        self._cond = asyncio.Condition()

    async def get(self, timeout: float = 5.0) -> str:
        async with self._cond:
            if (self._token is None or not self._token.valid()) and not self._refreshing:
                self._refreshing = True
                asyncio.create_task(self._refresh())      # keep a reference in real code
            # Guarded suspension: release the lock and sleep until the guard holds.
            await asyncio.wait_for(
                self._cond.wait_for(lambda: self._token is not None and self._token.valid()),
                timeout,
            )
            assert self._token is not None
            return self._token.value

    async def _refresh(self) -> None:
        token: Token | None = None
        try:
            token = await self._fetch()
        finally:
            async with self._cond:
                self._refreshing = False
                if token is not None:
                    self._token = token
                self._cond.notify_all()                   # wake every waiter to re-check the guard
```
If `_fetch` fails, waiters wake, find the guard still false, and keep waiting until the timeout; production code should also store the error and raise it to the waiters.

## Example — TypeScript
Single-threaded JS cannot block, so the "suspension" is an awaited Promise. Concurrent callers share the same in-flight Promise.
```ts
interface Token { value: string; expiresAt: number }

export class TokenHolder {
  private token: Token | null = null;
  private inflight: Promise<Token> | null = null;

  constructor(private readonly fetchToken: () => Promise<Token>) {}

  async get(): Promise<string> {
    if (this.token && Date.now() < this.token.expiresAt - 30_000) return this.token.value;
    // Guard is false: suspend on the shared Promise instead of polling.
    this.inflight ??= this.fetchToken()
      .then((t) => (this.token = t))
      .finally(() => (this.inflight = null));
    return (await this.inflight).value;
  }
}

// Reactive form in Angular: wait until a signal/observable satisfies the guard.
// const ready$ = toObservable(this.configLoaded).pipe(filter(Boolean), take(1));
// await firstValueFrom(ready$);
```

## In the wild
- Python: `threading.Condition.wait_for`, `asyncio.Condition.wait_for`, `threading.Event`, `asyncio.Event`; `queue.Queue.get()` is a guarded suspension on "not empty".
- Java `Object.wait/notifyAll`, `java.util.concurrent.locks.Condition`; POSIX `pthread_cond_wait`.
- JS: `Atomics.wait` (worker threads only; disallowed on the browser main thread) and `Atomics.waitAsync`; RxJS `filter(...)`, `take(1)`; Angular route guards and resolvers that wait on an observable.
- Kubernetes readiness probes and `kubectl wait --for=condition=...` are the same idea at the infrastructure level.

## Trade-offs
| Pros | Cons |
|---|---|
| No busy-waiting; wake-up latency near zero | Risk of waiting forever if nobody notifies |
| Callers get a simple blocking API | Lock + condition logic is easy to get subtly wrong |
| Precondition and action are atomic under one lock | Waiters hold threads (with threads) while sleeping |
| Composes into queues, latches, pools | Hidden blocking makes latency harder to reason about |

## Pitfalls
- **`if` instead of `while`.** After waking, the guard may be false again. Always loop, or use `wait_for`.
- **Notifying without the lock** or mutating state outside it. Both cause lost wake-ups. Mutate and notify under the same lock.
- **`notify()` when several kinds of waiter exist.** The one woken may not be the one whose guard became true. Prefer `notify_all()` unless you have a single waiter type.
- **No timeout.** A failed writer leaves waiters stuck. Pass a timeout and surface an error.
- **Calling slow I/O while holding the lock.** In the example, the fetch runs outside the lock; only the state update is inside.
- **Lost task reference.** `asyncio.create_task` results can be garbage-collected if not referenced; keep them in a set or use a TaskGroup.

## Refactoring path
- **Introduce it:** find a `sleep`-polling loop or a "check then act" race. Move the state and the check into one object with a lock. Replace polling with `Condition.wait_for(pred)`. Add `notify_all()` in every mutator. Add a timeout.
- **Remove it / simplify:** if the condition is "a single operation finished", replace the Condition with a Future/Promise or an Event. If the waiting is really work hand-off, use a `Queue`, which already contains the guarded suspension.

## Related patterns
- [Monitor Object](../concurrency/monitor-object.md): packages several guarded methods behind one lock.
- [Balking](../concurrency/balking.md): the opposite reaction to a false guard: return immediately.
- [Producer-Consumer](../concurrency/producer-consumer.md): its queue is two guarded suspensions (not empty, not full).
- [Future / Promise](../concurrency/future-promise.md): the simpler form when you wait for one result.
- [Barrier](../concurrency/barrier.md): a guarded suspension where the guard is "all N parties arrived".
