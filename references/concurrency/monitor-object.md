# Monitor Object

**Category:** Concurrency · **Also known as:** Monitor, Thread-safe class, Synchronized object · **Related:** Guarded Suspension, Read-Write Lock, Active Object, Lock Striping, Balking

## Intent
Encapsulate shared state together with a single lock so that only one thread at a time executes any of the object's methods, and let methods wait on condition variables for state changes.

## Problem
- Shared state (a counter, an inventory map, a connection list) is mutated from several threads, with locks acquired by callers in some places and forgotten in others.
- Invariants spanning two fields ("reserved <= stock") break under concurrent updates.
- Locking is done externally (`with lock: inventory.reserve(...)`), so every caller must remember the protocol.

Scenario: an in-process inventory for a flash sale. Many request threads reserve and release items. The invariant `0 <= reserved <= stock` must always hold, and reservation should optionally wait briefly for stock to be released.

## Solution and structure
```
             +---------------- Monitor -----------------+
thread A ──► | lock                                     |
thread B ──► | state (private)                          |
             | method1(): with lock: ... cond.wait() ...|
             | method2(): with lock: ... cond.notify()  |
             +------------------------------------------+
```
- **Monitor object**: private state + one lock (+ zero or more condition variables built on that lock).
- **Synchronized methods**: every public method acquires the lock on entry and releases it on exit; private helpers assume the lock is held.
- **Condition variables**: let a method wait for a guard (see Guarded Suspension) and let mutators notify waiters.

The lock is internal; callers just call methods. This is what Java's `synchronized` keyword gives every object.

## When to use
- A small object whose state is shared across threads and whose operations are short.
- Invariants across several fields must hold atomically.
- You want one obvious place to look for all synchronization of a piece of state.

## When NOT to use
- **asyncio and JS.** Code between `await`s is atomic; a plain class is already a monitor for synchronous methods. Use `asyncio.Lock` only for critical sections that span an `await`.
- **Read-mostly data** under heavy contention: copy-on-write or a Read-Write Lock.
- **Long-running or I/O-bound methods.** Holding the monitor during I/O serializes everything; do I/O outside the lock or use an Active Object.
- **Operations that span several monitors.** Calling monitor B from inside monitor A's lock invites deadlock (lock ordering). Restructure or use a single coarser lock.
- The data can be made immutable or owned by one thread: no lock is better than a correct lock.

## Example — Python
`threading.Condition` wraps an `RLock` by default; here a plain `Lock` makes accidental reentry fail loudly instead of hiding a design problem.
```python
import threading
from dataclasses import dataclass, field

class OutOfStock(Exception): ...

@dataclass
class Inventory:
    stock: dict[str, int]
    _reserved: dict[str, int] = field(default_factory=dict)
    _lock: threading.Lock = field(default_factory=threading.Lock, repr=False)

    def __post_init__(self) -> None:
        self._released = threading.Condition(self._lock)   # condition shares the monitor lock

    def available(self, sku: str) -> int:
        with self._lock:
            return self._available(sku)

    def _available(self, sku: str) -> int:                 # helper: caller holds the lock
        return self.stock.get(sku, 0) - self._reserved.get(sku, 0)

    def reserve(self, sku: str, qty: int, wait: float = 0.0) -> None:
        with self._lock:
            if not self._released.wait_for(lambda: self._available(sku) >= qty, timeout=wait):
                raise OutOfStock(sku)
            self._reserved[sku] = self._reserved.get(sku, 0) + qty   # invariant holds on exit

    def release(self, sku: str, qty: int) -> None:
        with self._lock:
            self._reserved[sku] = max(0, self._reserved.get(sku, 0) - qty)
            self._released.notify_all()

    def commit(self, sku: str, qty: int) -> None:          # reservation became a sale
        with self._lock:
            self._reserved[sku] -= qty
            self.stock[sku] -= qty
```
`wait_for(..., timeout=0)` checks once and returns immediately, so `reserve` balks by default and waits only when asked. Note that `stock` is a public field here for brevity; in real code keep it private so nobody mutates it without the lock.

## Example — TypeScript
In a single JS thread synchronous methods cannot interleave, so the monitor reduces to a mutex for async critical sections that cross an `await`.
```ts
export class Mutex {
  private last: Promise<void> = Promise.resolve();
  async lock(): Promise<() => void> {
    let release!: () => void;
    const next = new Promise<void>((r) => (release = r));
    const prev = this.last;
    this.last = prev.then(() => next);
    await prev;
    return release;
  }
}

export class Wallet {                                 // the monitor
  private balance = 0;
  private readonly mutex = new Mutex();
  constructor(private readonly ledger: { append(delta: number): Promise<void> }) {}

  async withdraw(amount: number): Promise<void> {
    const release = await this.mutex.lock();
    try {
      if (this.balance < amount) throw new Error('insufficient funds');
      await this.ledger.append(-amount);              // the await is why we need the lock
      this.balance -= amount;
    } finally {
      release();
    }
  }
}
```
Libraries such as `async-mutex` provide this (plus semaphores) ready-made. The Web Locks API (`navigator.locks.request`) coordinates across tabs and workers in the browser.

## In the wild
- Java `synchronized` + `wait/notifyAll`; C# `lock` + `Monitor`; Go `sync.Mutex` + `sync.Cond` embedded in a struct.
- Python: `queue.Queue` is a monitor (one mutex, three conditions: not_empty, not_full, all_tasks_done). `threading.Condition`, `asyncio.Lock`, `asyncio.Condition`.
- JS: `async-mutex`, Web Locks API, `Atomics`-based mutexes over `SharedArrayBuffer`.
- Databases: a row lock (`SELECT ... FOR UPDATE`) acts as a monitor around a single row.

## Trade-offs
| Pros | Cons |
|---|---|
| Synchronization is encapsulated; callers cannot forget it | One lock serializes all methods, even unrelated ones |
| Easy to reason about invariants | Nested monitor calls risk deadlock |
| Condition waits integrate cleanly | Holding the lock during slow work kills throughput |
| Small, well-understood code | Coarse-grained; may need striping or RW locks later |

## Pitfalls
- **Leaking the lock's protection**: returning a reference to an internal dict lets callers mutate it unlocked. Return copies or immutable views.
- **Calling out while locked**: invoking callbacks, logging to a network sink or calling another monitor under the lock. Copy what you need, release, then call out.
- **Reentrancy surprises**: a public method calling another public method deadlocks with `Lock` and silently works with `RLock`. Use private "lock held" helpers instead.
- **Check-then-act across two calls** (`if inv.available(s) > 0: inv.reserve(s, 1)`) is still racy. Provide a single atomic method.
- **Using a threading lock in asyncio** blocks the event loop. Use `asyncio.Lock`.

## Refactoring path
- **Introduce it:** collect the shared state and the scattered external locks into one class. Make fields private. Wrap each public method body in `with self._lock`. Replace caller-side check-then-act with combined methods. Replace sleep-polling with a condition.
- **Remove it / simplify:** if the object turns out to be used from one thread or task only, drop the lock. If contention appears, split into Lock Striping, a Read-Write Lock, or an Active Object that owns the state.

## Related patterns
- [Guarded Suspension](../concurrency/guarded-suspension.md): the condition-wait part of a monitor.
- [Balking](../concurrency/balking.md): a monitor method that returns when its guard is false instead of waiting.
- [Active Object](../concurrency/active-object.md): serializes via a queue and a thread rather than a lock.
- [Read-Write Lock](../concurrency/read-write-lock.md): relaxes the monitor's exclusivity for readers.
- [Lock Striping](../concurrency/lock-striping.md): splits one monitor lock into many.
