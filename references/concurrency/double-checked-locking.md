# Double-Checked Locking

**Category:** Concurrency · **Also known as:** DCL, Double-checked locking optimization · **Related:** Lazy Initialization, Singleton, Monitor Object, Read-Write Lock, Future / Promise

## Intent
Lazily initialize a shared resource exactly once without paying for a lock on every access: check without the lock, and only if the resource is missing, take the lock and check again before creating it.

## Problem
- An expensive shared object (DB engine, ML model, parsed config, HTTP client) should be created on first use, not at import time.
- A naive lazy getter (`if self._x is None: self._x = build()`) creates it twice when two threads race, which may leak connections or double-load a 2 GB model.
- Wrapping the whole getter in a lock fixes the race but makes every access contend on the lock, long after initialization is done.

Scenario: a FastAPI app loads a fraud-scoring model on first request. Sync endpoints run in a threadpool, so 40 threads may hit the getter at once on startup.

## Solution and structure
```
get():
  if instance is not None: return instance        # 1st check, no lock (fast path)
  with lock:
      if instance is None:                        # 2nd check, under lock
          instance = create()
  return instance
```
Correctness depends on **safe publication**: a thread that sees the reference non-null must also see the fully constructed object. In Java this requires `volatile`; in C++ `std::atomic` with acquire/release. Classic DCL without those is broken in those languages because writes can be reordered.

In **CPython** (GIL build), assigning a reference is atomic and the object is fully built before the assignment, so the pattern is safe. On the **free-threaded build** (3.13t/3.14t) CPython is designed to keep plain reference reads and writes safe (no torn pointers, internal per-object locking for dicts), and the pattern as written is expected to stay correct; this is newer territory, so check the free-threading docs for your exact version if correctness is critical. Never port it to C extensions or other runtimes without explicit memory-ordering guarantees.

## When to use
- Initialization is expensive, must happen at most once, must be lazy, and the getter is on a hot path accessed from multiple threads.
- In practice, prefer a ready-made primitive (below) that implements it for you.

## When NOT to use
- **Eager initialization is fine.** Create the object at startup (FastAPI `lifespan`, module import, Angular `APP_INITIALIZER` / `provideAppInitializer`). This removes the problem entirely and fails fast on misconfiguration.
- **A module-level global or `functools.cache` on a zero-arg function** is enough when creating twice is harmless (it is thread-safe for the cache dict, but can call the function more than once under a race).
- **asyncio**: no threads, so no DCL. The async problem is "two coroutines both await the slow init"; solve it by caching the Task/Future (see example) or an `asyncio.Lock`.
- **Single-threaded JS**: synchronous lazy init cannot race. Async init has the same fix as asyncio: memoize the Promise.
- Cheap objects: just lock every time, or create per use.

## Example — Python
Threaded version with DCL, and the asyncio version that caches the in-flight task instead.
```python
import asyncio, threading
from collections.abc import Awaitable, Callable

class FraudModel:
    def __init__(self, path: str) -> None:
        self.weights = open(path, "rb").read()      # slow, large

    def score(self, features: dict[str, float]) -> float:
        return 0.0

class LazyModel:
    def __init__(self, path: str) -> None:
        self._path = path
        self._model: FraudModel | None = None
        self._lock = threading.Lock()

    def get(self) -> FraudModel:
        model = self._model                          # read once into a local
        if model is None:
            with self._lock:
                model = self._model                  # second check under the lock
                if model is None:
                    model = FraudModel(self._path)
                    self._model = model              # publish only the fully built object
        return model

class AsyncLazy[T]:
    """asyncio: memoize the Task so concurrent awaiters share one initialization."""
    def __init__(self, factory: Callable[[], Awaitable[T]]) -> None:
        self._factory = factory
        self._task: asyncio.Task[T] | None = None

    async def get(self) -> T:
        if self._task is None or (self._task.done() and self._task.exception()):
            self._task = asyncio.ensure_future(self._factory())  # retry after a failed init
        return await self._task
```
Note: `functools.cached_property` stopped taking a lock in Python 3.12, so it no longer prevents double computation under threads. Use the explicit pattern above if "exactly once" matters.

## Example — TypeScript
Synchronous lazy init in JS never races. The async analog of DCL is memoizing the Promise; Angular DI already gives lazily created root singletons.
```ts
import { Injectable, inject } from '@angular/core';
import { HttpClient } from '@angular/common/http';
import { firstValueFrom } from 'rxjs';

interface FeatureFlags { [key: string]: boolean }

@Injectable({ providedIn: 'root' })          // created once, on first inject()
export class FlagsService {
  private readonly http = inject(HttpClient);
  private flags?: Promise<FeatureFlags>;

  load(): Promise<FeatureFlags> {
    // "first check": reuse the in-flight or settled Promise
    this.flags ??= firstValueFrom(this.http.get<FeatureFlags>('/api/flags')).catch((err) => {
      this.flags = undefined;                // allow retry after failure
      throw err;
    });
    return this.flags;
  }
}
```
RxJS `shareReplay({ bufferSize: 1, refCount: false })` gives the same "fetch once, share" behavior for observables. Across Node `worker_threads`, each worker has its own module instance, so there is nothing shared to double-check.

## In the wild
- Java: the canonical `volatile` DCL singleton, or the holder-class idiom; Kotlin `by lazy` (synchronized by default); C# `Lazy<T>`.
- C++11: function-local statics are initialized thread-safely (`std::call_once` under the hood).
- Python: module import is guarded by the import lock (a module global is a safe one-time init); `functools.cache`; Django's lazy settings.
- Go: `sync.Once` / `sync.OnceValue`. Rust: `std::sync::OnceLock`, `LazyLock`.
- Angular: `providedIn: 'root'` services are lazily instantiated singletons per injector.

## Trade-offs
| Pros | Cons |
|---|---|
| Lock-free fast path after initialization | Subtle; broken in languages without safe publication |
| Exactly-once creation | More code than eager init |
| Keeps startup fast | First caller pays the latency spike |
| Failure on init can be retried | Errors appear at first use, not at startup |

## Pitfalls
- **Checking the field twice without a local** and returning the field: harmless in CPython, but a bug pattern in other languages. Read into a local.
- **Publishing a half-built object**: assigning `self._model = FraudModel.__new__(...)` then filling it in. Build fully, then assign.
- **Porting the Python version to Java/C++ without `volatile` / atomics.** It is the textbook example of a broken pattern there.
- **Caching a failed async init forever.** Clear the memoized Promise/Task on failure.
- **Using DCL when eager init in a lifespan hook would do.** Lazy init hides config errors until traffic arrives.

## Refactoring path
- **Introduce it:** a lazy getter shows a race (double creation) or a profile shows contention on a lock around a getter. Read into a local, add the lock and second check, and test with many threads hitting the getter at startup.
- **Remove it / simplify:** move creation to startup (`lifespan`, DI provider, module import) and delete the lock and checks. Or replace with a platform primitive: `sync.Once`, `Lazy<T>`, memoized Promise, Angular root provider.

## Related patterns
- [Lazy Initialization](../creational/lazy-initialization.md): the general idea DCL makes thread-safe.
- [Singleton](../creational/singleton.md): DCL is a classic (and often unnecessary) way to build a lazy singleton.
- [Monitor Object](../concurrency/monitor-object.md): the "always lock" alternative.
- [Future / Promise](../concurrency/future-promise.md): memoizing the future is the async form of DCL.
- [Read-Write Lock](../concurrency/read-write-lock.md): another way to make read-mostly access cheap.
