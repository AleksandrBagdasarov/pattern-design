# Read-Write Lock

**Category:** Concurrency · **Also known as:** Shared-Exclusive lock, RWLock, Readers-Writer lock · **Related:** Monitor Object, Lock Striping, Double-Checked Locking, CQRS, Snapshot

## Intent
Let any number of readers access shared data at the same time while giving a writer exclusive access, so read-heavy data is not serialized behind a single mutex.

## Problem
- An in-memory structure (routing table, feature-flag set, price cache) is read on every request and updated rarely, but a plain `Lock` serializes all readers and becomes the hot spot in a profile.
- Readers must never see a half-applied update (two related fields changed in separate steps).
- Writers starve when a constant stream of readers holds the lock.

Scenario: a pricing service keeps a 50k-entry price table in memory. Thousands of threads read prices per second; a background job swaps in updates every minute.

## Solution and structure
```
readers ──acquire_read()──► (shared, many at once) ──release_read()
writer  ──acquire_write()─► (waits for readers to drain, blocks new ones) ──release_write()
```
- **Lock state**: active reader count, writer-active flag, waiting-writer count.
- **Read acquire**: proceeds unless a writer holds the lock (or, in writer-preferring locks, is waiting).
- **Write acquire**: waits until no readers and no writer.
- **Policy**: reader-preferring (writers can starve), writer-preferring (readers can starve under write bursts), or fair (FIFO). Pick deliberately.

Before reaching for an RWLock, consider **copy-on-write with an atomic reference swap**: readers take the current immutable snapshot without any lock; the writer builds a new object and replaces the reference. In CPython, rebinding an attribute is atomic, which makes this the idiomatic choice.

## When to use
- Reads vastly outnumber writes, read critical sections are long enough that contention on a mutex actually shows up in profiling, and the data cannot simply be replaced as an immutable snapshot.
- Real parallel readers exist: OS threads in Java/Go/Rust/C++, Python free-threaded builds, or readers that release the GIL inside the critical section (NumPy, I/O).
- Database / file level: shared vs exclusive row or table locks, `fcntl.flock(LOCK_SH / LOCK_EX)`.

## When NOT to use
- **Standard CPython with pure-Python readers.** The GIL already serializes bytecode, so an RWLock rarely adds throughput and its own bookkeeping costs more than a `Lock`. Use a plain `Lock` or copy-on-write.
- **asyncio.** Code between `await`s is already atomic relative to other tasks. You only need a lock if a read spans an `await` and a writer must not interleave; usually an immutable snapshot is still simpler.
- **Single-threaded JS.** No shared memory between callbacks, so no RWLock. Across workers you share data by message passing; locks over `SharedArrayBuffer` are rare and low-level.
- Short critical sections: a mutex is cheaper than RWLock bookkeeping. Measure first.
- Write-heavy data: shard it (Lock Striping) instead.

## Example — Python
Preferred: copy-on-write snapshot. Shown with a minimal writer-preferring RWLock for the case where in-place mutation is unavoidable.
```python
import threading
from contextlib import contextmanager
from collections.abc import Iterator, Mapping
from types import MappingProxyType

class PriceTable:
    """Copy-on-write: readers never lock; one writer at a time builds and swaps."""
    def __init__(self, prices: dict[str, int]) -> None:
        self._snapshot: Mapping[str, int] = MappingProxyType(dict(prices))
        self._write_lock = threading.Lock()

    def get(self, sku: str) -> int | None:
        return self._snapshot.get(sku)             # one attribute read: always a consistent snapshot

    def apply(self, changes: dict[str, int]) -> None:
        with self._write_lock:                      # serialize writers so no update is lost
            new = dict(self._snapshot) | changes
            self._snapshot = MappingProxyType(new)  # atomic reference swap

class RWLock:
    """Writer-preferring: new readers wait while a writer is waiting."""
    def __init__(self) -> None:
        self._cond = threading.Condition()
        self._readers = 0
        self._writer = False
        self._waiting_writers = 0

    @contextmanager
    def read(self) -> Iterator[None]:
        with self._cond:
            self._cond.wait_for(lambda: not self._writer and self._waiting_writers == 0)
            self._readers += 1
        try:
            yield
        finally:
            with self._cond:
                self._readers -= 1
                if self._readers == 0:
                    self._cond.notify_all()

    @contextmanager
    def write(self) -> Iterator[None]:
        with self._cond:
            self._waiting_writers += 1
            self._cond.wait_for(lambda: not self._writer and self._readers == 0)
            self._waiting_writers -= 1
            self._writer = True
        try:
            yield
        finally:
            with self._cond:
                self._writer = False
                self._cond.notify_all()
```
The stdlib has no RWLock; if you need one, `readerwriterlock` on PyPI provides tested variants. Neither lock above is reentrant: do not take `read()` inside `write()`.

## Example — TypeScript
In Angular/Node the equivalent of "readers never block" is an immutable value behind a signal or `BehaviorSubject`: readers get the current snapshot, writers replace it.
```ts
import { Injectable, signal, computed } from '@angular/core';

type Prices = Readonly<Record<string, number>>;

@Injectable({ providedIn: 'root' })
export class PriceStore {
  private readonly prices = signal<Prices>({});          // the "snapshot reference"
  readonly count = computed(() => Object.keys(this.prices()).length);

  price(sku: string): number | undefined {
    return this.prices()[sku];                           // read: no lock, consistent snapshot
  }

  apply(changes: Prices): void {
    this.prices.update((cur) => ({ ...cur, ...changes })); // write: build new, swap reference
  }
}
```
Mutating the object in place (`this.prices()[sku] = 1`) breaks both consistency and change detection; always replace.

## In the wild
- Java `ReentrantReadWriteLock`, `StampedLock` (optimistic reads); Go `sync.RWMutex`; Rust `RwLock`; C++ `std::shared_mutex`.
- Linux kernel `rwlock_t` and RCU (read-copy-update: the copy-on-write idea at kernel scale).
- PostgreSQL `SELECT ... FOR SHARE` vs `FOR UPDATE`; SQLite WAL mode lets readers proceed during a write.
- `fcntl.flock` shared/exclusive file locks.
- Copy-on-write: Python `MappingProxyType` over a fresh dict, Immutable.js / Immer, Angular signals, NgRx state.

## Trade-offs
| Pros | Cons |
|---|---|
| Parallel reads where real parallelism exists | More expensive than a mutex per acquire |
| Readers see consistent state | Starvation unless policy is chosen well |
| Writers get exclusive access without stopping the world forever | Upgrade read→write deadlocks easily |
| Copy-on-write variant makes reads lock-free | Copy-on-write costs a full copy per write |

## Pitfalls
- **Lock upgrade.** Two readers both trying to upgrade to write deadlock. Release the read lock and acquire the write lock, then re-validate.
- **Writer starvation** with reader-preferring locks under constant read load. Use writer preference or fairness.
- **Assuming a speed-up under the GIL.** Benchmark against a plain `Lock` and against copy-on-write.
- **Returning references to mutable internals** from inside the read section; the caller then reads them after release. Return copies or immutable views.
- **Lost update in copy-on-write** when two writers copy the same snapshot. Serialize writers with a mutex (as in the example) or compare-and-swap.

## Refactoring path
- **Introduce it:** profile shows contention on a mutex guarding read-mostly data. First try copy-on-write (immutable snapshot + swap). If writes must be in place and are rare, replace the mutex with an RWLock, mark each section read or write, and load-test for starvation.
- **Remove it / simplify:** if benchmarks show no gain, return to a single `Lock`. If writes became frequent, shard the data (Lock Striping) or move reads to a separate read model (CQRS).

## Related patterns
- [Monitor Object](../concurrency/monitor-object.md): the simpler "one lock for everything" alternative.
- [Lock Striping](../concurrency/lock-striping.md): reduces contention by partitioning rather than by read/write mode.
- [Double-Checked Locking](../concurrency/double-checked-locking.md): another trick for read-mostly lazy state.
- [CQRS](../cloud/cqrs.md): the architectural version: separate read and write models entirely.
- [Snapshot](../data-access/snapshot.md): the immutable-snapshot idea behind copy-on-write.
