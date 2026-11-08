# Work Stealing

**Category:** Concurrency · **Also known as:** Work-stealing scheduler, Fork/Join pool · **Related:** Thread Pool, Producer-Consumer, Scheduler, Barrier

## Intent
Balance uneven workloads across workers by giving each worker its own task deque; a worker pushes and pops its own tasks at one end, and an idle worker steals tasks from the other end of a busy worker's deque.

## Problem
- A thread pool with one shared queue becomes a contention point when tasks are tiny and numerous (millions of sub-millisecond tasks all hitting one lock).
- Recursive divide-and-conquer work (parallel sort, tree traversal, crawling a directory tree) spawns subtasks dynamically; static partitioning ("worker i takes items i, i+N, …") leaves some workers idle while one grinds through a deep subtree.
- Tasks have very different costs, so a fixed split finishes at the speed of the unluckiest worker.

Scenario: indexing a document store where a few customers have 10 million documents and most have a hundred. Splitting by customer leaves 7 of 8 workers idle after a minute.

## Solution and structure
```
worker 1: deque [t1 t2 t3 t4] ◄─ push/pop (LIFO, owner end)
worker 2: deque [ ]            ─── idle → steal from the top (FIFO end) of worker 1 ─► t1
worker 3: deque [t9]
```
- **Per-worker deque**: the owner pushes new subtasks and pops from the same end (LIFO: good cache locality, depth-first, bounded memory).
- **Thieves** take from the opposite end (oldest tasks, usually the largest chunks of remaining work), which minimizes contention with the owner.
- **Steal policy**: pick a random victim; back off or park when nothing is found.
- **Join**: a task waiting on its children helps by running other tasks rather than blocking ("help-first" / "work-first" strategies).

Implementing a correct lock-free deque (Chase-Lev) is hard. Application developers should **use a runtime that already does this** rather than build one.

## When to use
- CPU-bound, fine-grained, recursive or highly uneven parallel work.
- You choose a runtime: Java ForkJoinPool, .NET TPL, Go, Tokio, Rayon, TBB all use it internally. Knowing the pattern helps you write tasks that suit it (small, independent, splittable).
- Distributed batch: Dask and Ray schedulers steal tasks between workers to rebalance.

## When NOT to use
- **Python application code.** The stdlib executors use one shared queue, and the GIL (default build) prevents CPU parallelism in threads anyway. Use `ProcessPoolExecutor.map(..., chunksize=...)` with reasonably sized chunks, NumPy/Polars (parallel in native code), or Dask/Ray. Building work stealing in pure Python costs more than it saves.
- **I/O-bound work.** The bottleneck is waiting, not balancing CPUs. asyncio or a simple pool is enough.
- **Node / browser.** One event loop per thread; with worker_threads or Web Workers, a shared pool (Piscina) with a central queue is usually fine because tasks are coarse.
- Uniform, coarse tasks: a plain pool with a shared queue balances them already.

## Example — Python
Pure-Python work stealing is not worthwhile; the practical way to get its benefit is dynamic, small-ish chunks pulled from a shared queue (a degenerate "everyone steals from the center" scheme), which balances uneven tasks well with processes.
```python
from concurrent.futures import ProcessPoolExecutor
from collections.abc import Iterator
from dataclasses import dataclass

@dataclass(frozen=True)
class Chunk:
    customer_id: str
    offset: int
    limit: int

def plan_chunks(doc_counts: dict[str, int], chunk_size: int = 5_000) -> Iterator[Chunk]:
    """Split big customers into many chunks so no single task dominates."""
    for cid, n in doc_counts.items():
        for off in range(0, n, chunk_size):
            yield Chunk(cid, off, chunk_size)

def index_chunk(chunk: Chunk) -> int:
    # CPU-bound: tokenize + build postings for documents [offset, offset+limit)
    return min(chunk.limit, 5_000)

def index_all(doc_counts: dict[str, int], workers: int = 8) -> int:
    chunks = list(plan_chunks(doc_counts))   # if chunk costs differ, sort largest first to shorten the tail
    with ProcessPoolExecutor(max_workers=workers) as pool:
        # chunksize=1: each worker pulls the next task as soon as it is free (dynamic balancing)
        return sum(pool.map(index_chunk, chunks, chunksize=1))

if __name__ == "__main__":
    print(index_all({"big-co": 10_000_000, "small-a": 120, "small-b": 80}))
```
For real recursive parallelism in Python, Dask (`dask.distributed` steals tasks between workers) and Ray are the tools; they schedule a task graph rather than hand-written deques.

## Example — TypeScript
In JS, a worker-thread pool with a shared queue gives the same balancing for coarse tasks. Splitting work into many small jobs is what matters.
```ts
// npm i piscina — a worker_threads pool; idle workers take the next queued task.
import Piscina from 'piscina';
import { availableParallelism } from 'node:os';

interface Chunk { customerId: string; offset: number; limit: number }

const pool = new Piscina({
  filename: new URL('./index-chunk.worker.js', import.meta.url).href,
  maxThreads: availableParallelism(),
});

export function planChunks(counts: Record<string, number>, size = 5_000): Chunk[] {
  return Object.entries(counts).flatMap(([customerId, n]) =>
    Array.from({ length: Math.ceil(n / size) }, (_, i) => ({ customerId, offset: i * size, limit: size })),
  );
}

export async function indexAll(counts: Record<string, number>): Promise<number> {
  const chunks = planChunks(counts);
  const done = await Promise.all(chunks.map((c) => pool.run(c) as Promise<number>));
  return done.reduce((a, b) => a + b, 0);
}
```
True work stealing across JS workers would require `SharedArrayBuffer` deques with `Atomics`; this is almost never worth it in application code.

## In the wild
- Java `ForkJoinPool` (also behind parallel streams and virtual-thread carriers), .NET ThreadPool local queues, Intel TBB, Cilk.
- Go runtime scheduler (per-P run queues with stealing); Tokio multi-threaded runtime; Rust Rayon (`par_iter`).
- Dask distributed (work stealing between workers), Ray scheduler spillback; Spark speculative execution is a related straggler fix.
- Python: none in the stdlib executors (shared queue); Python itself benefits indirectly through native libraries.

## Trade-offs
| Pros | Cons |
|---|---|
| Near-optimal load balance for uneven, recursive work | Very hard to implement correctly (lock-free deques) |
| Low contention: owners work on their own deque | Task order is not FIFO; latency of individual tasks varies |
| Good cache locality (LIFO on owner side) | Overhead dominates if tasks are too tiny |
| Scales with core count | Little benefit for I/O-bound or coarse uniform tasks |

## Pitfalls
- **Hand-rolling it** instead of choosing a runtime that has it. Use ForkJoinPool, Rayon, Tokio, Go, Dask.
- **Tasks too fine-grained** (a few instructions): scheduling overhead dominates. Add a sequential threshold below which you stop splitting.
- **Blocking inside tasks** (I/O, locks) starves the pool. Keep stealing pools for CPU work; do I/O elsewhere.
- **Expecting ordering.** Results arrive out of order; collect and sort if needed.
- **Python: one huge task** dominates no matter how the pool schedules. Split large inputs into chunks first, and sort big chunks first.

## Refactoring path
- **Introduce it:** profiling shows idle workers while one is busy (stragglers). First split work into smaller, independent chunks and use a pool with dynamic pull (`chunksize=1`). Only if contention on the shared queue shows up, move to a runtime with work stealing (Dask/Ray in Python, ForkJoin/Rayon elsewhere).
- **Remove it / simplify:** if tasks are coarse and uniform, a plain `ThreadPoolExecutor`/`ProcessPoolExecutor` with a shared queue is simpler and as fast.

## Related patterns
- [Thread Pool](../concurrency/thread-pool.md): the base design; work stealing replaces its single queue with per-worker deques.
- [Producer-Consumer](../concurrency/producer-consumer.md): the shared-queue alternative.
- [Scheduler](../concurrency/scheduler.md): work stealing is a scheduling policy.
- [Barrier](../concurrency/barrier.md): lock-step phases are the opposite approach, wasteful on uneven work.
