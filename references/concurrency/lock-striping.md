# Lock Striping

**Category:** Concurrency · **Also known as:** Lock splitting, Striped locks, Fine-grained locking, Partitioned locks · **Related:** Monitor Object, Read-Write Lock, Sharding, Bulkhead

## Intent
Reduce lock contention by protecting a shared structure with N locks, each guarding a partition ("stripe") chosen by hashing the key, so operations on different keys rarely block each other.

## Problem
- One global lock guards a large map (per-user session store, per-account balance cache, per-key rate-limit counters). The profiler shows threads spending most of their time waiting for it, yet they touch unrelated keys.
- A per-key lock dictionary (`locks[key] = Lock()`) grows without bound and itself needs a lock to create entries.
- Lock splitting by concern (one lock for `users`, one for `orders`) helps but still leaves one hot lock per concern.

Scenario: a payment gateway's in-process idempotency cache records request keys from thousands of concurrent threads. Each operation touches one key, but all serialize on one mutex.

## Solution and structure
```
key ──hash(key) % N──► stripe i ──► locks[i] guards buckets[i]

locks:   [L0] [L1] [L2] ... [LN-1]
buckets: {..} {..} {..} ... {..}
```
- **Stripes**: a fixed array of N locks (often a power of two, sized to the expected parallelism, e.g. 16–64).
- **Stripe function**: `hash(key) & (N - 1)`; must be stable for a key during the object's lifetime.
- **Partitioned data** (optional): each stripe can own its own sub-dict, so even the container's internal resizing is per stripe.
- **Whole-structure operations** (size, clear, snapshot) must acquire all stripes in a fixed order (0..N-1) to avoid deadlock, or accept an approximate answer.

Related variant: **lock splitting** uses different locks for independent fields of one object rather than hashed partitions.

## When to use
- Real thread parallelism with measured contention on one lock, and operations that touch a single key.
- Keys are reasonably uniform so stripes get similar load.
- Bounded memory for locks is required (unlike lock-per-key).
- Per-key mutual exclusion (serialize work for the same customer) without a global bottleneck.

## When NOT to use
- **Standard CPython with short pure-Python critical sections.** The GIL already serializes bytecode, so striping rarely helps throughput; it can help when critical sections release the GIL (I/O, C extensions) or on the free-threaded build. Measure first.
- **asyncio / JS.** No parallel threads; per-key `asyncio.Lock` (or a per-key Promise chain) is for ordering async steps, not for contention.
- Operations commonly span many keys (transfers between accounts): you need multi-stripe locking with ordering, and complexity grows fast. Consider a single lock, a DB transaction, or an actor per entity.
- A concurrent collection already exists in your platform (`ConcurrentHashMap`, Go `sync.Map`): use it.
- Across processes or machines: that is Sharding plus distributed locks, not in-memory striping.

## Example — Python
Per-key serialization with a bounded number of locks. Also usable as an asyncio variant by swapping in `asyncio.Lock`.
```python
import threading
from collections.abc import Hashable, Iterator
from contextlib import contextmanager, ExitStack

class StripedLocks:
    def __init__(self, stripes: int = 32) -> None:
        if stripes & (stripes - 1):
            raise ValueError("stripes must be a power of two")
        self._mask = stripes - 1
        self._locks = [threading.Lock() for _ in range(stripes)]

    def _index(self, key: Hashable) -> int:
        return hash(key) & self._mask   # str hashes vary per process (PYTHONHASHSEED); fine in-process

    @contextmanager
    def for_key(self, key: Hashable) -> Iterator[None]:
        with self._locks[self._index(key)]:
            yield

    @contextmanager
    def for_keys(self, *keys: Hashable) -> Iterator[None]:
        """Lock several keys without deadlock: unique stripes, always in ascending order."""
        with ExitStack() as stack:
            for i in sorted({self._index(k) for k in keys}):
                stack.enter_context(self._locks[i])
            yield

class IdempotencyCache:
    def __init__(self) -> None:
        self._locks = StripedLocks(64)
        self._seen: dict[str, str] = {}          # dict ops are atomic; the lock guards check-then-set

    def claim(self, request_key: str, response_id: str) -> str:
        with self._locks.for_key(request_key):
            existing = self._seen.get(request_key)
            if existing is not None:
                return existing                  # duplicate request: return original response
            self._seen[request_key] = response_id
            return response_id

    def transfer(self, from_acct: str, to_acct: str) -> None:
        with self._locks.for_keys(from_acct, to_acct):
            ...                                  # both accounts locked, no deadlock
```

## Example — TypeScript
Single-threaded JS has no lock contention, but per-key serialization of async work is common (e.g. saving the same document). A striped/keyed mutex keeps memory bounded.
```ts
export class KeyedSerializer {
  private readonly tails = new Map<string, Promise<unknown>>();

  run<T>(key: string, task: () => Promise<T>): Promise<T> {
    const prev = this.tails.get(key) ?? Promise.resolve();
    const result = prev.then(task, task);              // run after the previous task, even if it failed
    const tail = result.catch(() => undefined);
    this.tails.set(key, tail);
    void tail.then(() => {
      if (this.tails.get(key) === tail) this.tails.delete(key); // bounded memory: drop idle keys
    });
    return result;
  }
}

// usage in a service: saves for the same document never overlap, different documents run concurrently
// this.serializer.run(doc.id, () => firstValueFrom(this.http.put(`/api/docs/${doc.id}`, doc)));
```
Here the map entry is dropped when idle, which avoids unbounded growth without fixed stripes. With `worker_threads` sharing a `SharedArrayBuffer`, a true striped lock would use an `Int32Array` of lock words and `Atomics.compareExchange`.

## In the wild
- Java `ConcurrentHashMap` (segment locks up to Java 7; per-bin locking + CAS since Java 8); Guava `Striped<Lock>`.
- Go: sharded maps (`orcaman/concurrent-map`); `sync.Map` for specific access patterns.
- Databases: row-level locks instead of table locks; PostgreSQL lock manager partitions (`NUM_LOCK_PARTITIONS`).
- Python: free-threaded CPython uses per-object locks internally rather than a global lock, the same idea at interpreter level.
- Kafka partitions and per-key ordering: the distributed analog.

## Trade-offs
| Pros | Cons |
|---|---|
| Contention drops roughly by the stripe count for uniform keys | Hot keys still contend; skew defeats it |
| Bounded memory for locks | Whole-structure operations need all locks |
| Per-key serialization without a global bottleneck | Multi-key operations need careful ordering |
| Simple to retrofit behind the same API | Unrelated keys sharing a stripe still block each other |

## Pitfalls
- **Deadlock on multi-key operations** acquired in arbitrary order. Always sort and deduplicate stripe indices.
- **Unstable hash** (mutable keys, hash depending on mutable fields). Use immutable keys.
- **Too few stripes** with many threads, or **too many**, wasting memory and cache lines. Size to expected parallel threads, not to key count.
- **Lock-per-key dict** without eviction: memory leak. Stripe, or evict idle entries like the TS example.
- **Assuming a speed-up under the GIL.** Benchmark before and after.

## Refactoring path
- **Introduce it:** confirm contention on one lock with profiling. Make sure each operation touches one key. Replace `self._lock` with `self._locks.for_key(key)`. Handle multi-key operations with ordered acquisition, and whole-structure operations with "acquire all".
- **Remove it / simplify:** if benchmarks show no gain, revert to a single lock. If the data moved to a database or Redis, rely on its row/key-level atomicity and delete the in-process locks.

## Related patterns
- [Monitor Object](../concurrency/monitor-object.md): the single-lock design striping refines.
- [Read-Write Lock](../concurrency/read-write-lock.md): reduces contention by read/write mode instead of by key.
- [Sharding](../cloud/sharding.md): the same partition-by-key idea across machines.
- [Bulkhead](../cloud/bulkhead.md): partitions resources to isolate failures rather than contention.
