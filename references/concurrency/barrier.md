# Barrier

**Category:** Concurrency · **Also known as:** Rendezvous, Phase barrier, CyclicBarrier, CountDownLatch (one-shot variant) · **Related:** Guarded Suspension, Thread Pool, Future / Promise, Work Stealing

## Intent
Make a fixed group of N workers wait at a synchronization point until all of them have arrived, then release them together, optionally running one action between phases.

## Problem
- A simulation or batch computation runs in phases; phase k+1 reads results that every worker writes in phase k. Without a barrier, fast workers read half-finished data.
- A load test must start 200 clients at exactly the same moment, not staggered by their setup time.
- Workers each load a shard of a model or cache; serving must not start until all shards are loaded.
- Hand-rolled counters with `sleep` loops ("wait until done_count == N") are racy and slow.

Scenario: a nightly reconciliation splits 8 bank accounts files among 8 workers. Phase 1 parses and normalizes; phase 2 matches transactions across all accounts and needs every file normalized first.

## Solution and structure
```
worker 1 ──phase 1──┐
worker 2 ──phase 1──┼──► [ barrier: wait until N arrived ] ──► action() ──► all start phase 2
worker N ──phase 1──┘
```
- **Barrier(parties=N, action=None)**: counts arrivals; the N-th arrival releases everyone and runs the optional action once.
- **Party / worker**: calls `wait()` at the end of each phase.
- **Cyclic** barriers reset automatically for the next phase; a **latch** (count down to zero) is one-shot and lets non-participants wait.
- **Broken state**: if one party times out or fails, the barrier breaks and every waiter gets an error instead of hanging.

## When to use
- Fixed, known number of participants working in lock-step phases (iterative numeric methods, simulations, parallel test setup).
- "Start all at once" coordination for benchmarks and race-condition tests.
- Fan-out followed by a join where all participants keep running afterward.

## When NOT to use
- You only need to wait for all tasks to finish once: `asyncio.TaskGroup`, `asyncio.gather`, `concurrent.futures.wait`, or `Promise.all` is the join. That covers most real cases.
- The number of participants is dynamic. Barriers need a fixed count (Java's `Phaser` handles dynamic parties; Python has no stdlib equivalent).
- Workers progress at very different speeds: everyone waits for the slowest each phase. Consider a pipeline (Producer-Consumer) or work stealing.
- Across machines: use a coordination service (ZooKeeper double barrier, a DB counter) or restructure as a job DAG (Airflow, Step Functions).

## Example — Python
`threading.Barrier` for threads; `asyncio.Barrier` (3.11+) has the same shape for tasks. Processes use `multiprocessing.Barrier`.
```python
import threading
from collections import defaultdict
from concurrent.futures import ThreadPoolExecutor
from decimal import Decimal

ACCOUNTS = ["acc-1", "acc-2", "acc-3", "acc-4"]
normalized: dict[str, list[tuple[str, Decimal]]] = {}
matches: dict[str, list[str]] = defaultdict(list)
lock = threading.Lock()

def build_index() -> None:            # runs once, in exactly one thread, between phases
    print(f"phase 1 done: {sum(map(len, normalized.values()))} transactions")

barrier = threading.Barrier(len(ACCOUNTS), action=build_index, timeout=60)

def load(account: str) -> list[tuple[str, Decimal]]:
    return [(f"{account}-tx{i}", Decimal(i)) for i in range(3)]  # stand-in for parsing a file

def worker(account: str) -> None:
    rows = load(account)                         # phase 1: independent work
    with lock:
        normalized[account] = rows
    barrier.wait()                               # BrokenBarrierError if another party fails/times out
    for other, txs in normalized.items():        # phase 2: reads everyone's phase-1 output
        if other != account:
            hits = [tx for tx, amt in txs if any(amt == a for _, a in rows)]
            with lock:
                matches[account].extend(hits)

with ThreadPoolExecutor(max_workers=len(ACCOUNTS)) as pool:  # pool must have >= N workers
    for f in [pool.submit(worker, a) for a in ACCOUNTS]:
        f.result()
```
If you only needed "parse all, then match", two separate `pool.map` calls would be simpler; the barrier pays off when workers keep local state across many phases.

## Example — TypeScript
On the main JS thread, a barrier is just `Promise.all` between phases. A true barrier is needed only across `worker_threads` / Web Workers, built on `SharedArrayBuffer` + `Atomics`.
```ts
// Phase barrier for async tasks in one thread: each phase is a join.
export async function runPhases<S>(
  parties: readonly S[],
  phases: ReadonlyArray<(party: S) => Promise<void>>,
): Promise<void> {
  for (const phase of phases) {
    await Promise.all(parties.map((p) => phase(p))); // nobody starts phase k+1 early
  }
}

// Cross-thread barrier (worker_threads), one-shot, using Atomics.
// sab = new SharedArrayBuffer(4) shared with all workers; parties = number of workers.
export function barrierWait(sab: SharedArrayBuffer, parties: number): void {
  const counter = new Int32Array(sab);
  const arrived = Atomics.add(counter, 0, 1) + 1;
  if (arrived === parties) {
    Atomics.notify(counter, 0);                  // release all waiters
  } else {
    for (let seen = arrived; seen < parties; seen = Atomics.load(counter, 0)) {
      Atomics.wait(counter, 0, seen); // sleeps only while the value is still `seen`; blocks this worker only
    }
  }
}
```
`Atomics.wait` throws on the browser main thread; there use `Atomics.waitAsync` or plain Promises. A reusable (cyclic) Atomics barrier also needs a generation counter.

## In the wild
- Python: `threading.Barrier`, `asyncio.Barrier`, `multiprocessing.Barrier`; `torch.distributed.barrier()` and MPI `MPI_Barrier` in ML/HPC.
- Java: `CyclicBarrier`, `CountDownLatch`, `Phaser`; Go `sync.WaitGroup` (latch-like).
- JS: `Promise.all`, RxJS `forkJoin` / `combineLatest` (join, not cyclic).
- Distributed: ZooKeeper barrier recipe; Spark stage boundaries behave like a barrier between shuffles.

## Trade-offs
| Pros | Cons |
|---|---|
| Simple, correct phase coordination | Throughput limited by the slowest party each phase |
| Optional action runs once between phases | Fixed party count; a crashed party breaks everyone |
| Replaces racy counters and sleeps | Deadlock if the pool has fewer threads than parties |
| Useful for reproducible concurrency tests | Rarely needed in I/O-bound app code |

## Pitfalls
- **Pool smaller than parties.** With a 4-worker pool and a 5-party barrier, the fifth party never runs. Give the pool at least N workers, or use dedicated threads.
- **No timeout.** One stuck worker hangs all. Set `timeout`, catch `BrokenBarrierError`, and abort the job.
- **Heavy action.** The barrier action runs while every party waits; keep it short.
- **Using a barrier as a join.** If workers end after the barrier, you wanted `TaskGroup` / `gather` / `Promise.all`.
- **Exceptions skipping `wait()`.** A party that raises before `wait()` leaves others waiting; call `barrier.abort()` in an `except` so others fail fast.

## Refactoring path
- **Introduce it:** spot a shared counter plus sleep loop or a manual "all loaded?" flag. Fix the party count, replace the loop with `Barrier(N).wait()`, add a timeout and `abort()` on error.
- **Remove it / simplify:** if workers do not keep state across phases, split the work into sequential joins (`gather` per phase) and delete the barrier. If phases form a DAG, model it as a pipeline or job graph instead.

## Related patterns
- [Guarded Suspension](../concurrency/guarded-suspension.md): a barrier is a guarded wait on "arrived == N".
- [Future / Promise](../concurrency/future-promise.md): `gather` / `Promise.all` over futures is the one-shot join most code needs.
- [Thread Pool](../concurrency/thread-pool.md): barriers inside a pool require the pool to be large enough.
- [Work Stealing](../concurrency/work-stealing.md): an alternative for uneven workloads, where lock-step phases waste time.
