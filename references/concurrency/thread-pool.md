# Thread Pool

**Category:** Concurrency · **Also known as:** Worker Pool, Executor, Replicated Workers · **Related:** Producer-Consumer, Future / Promise, Work Stealing, Bulkhead, Object Pool

## Intent
Run many short tasks on a fixed, reusable set of worker threads (or processes) fed by a shared task queue, so concurrency is bounded and the cost of creating workers is paid once.

## Problem
- Code spawns a new `threading.Thread` (or process) per request or per item. Under load you get thousands of threads, memory blow-up, context-switch thrash, or "can't start new thread".
- A batch job calls a blocking library (boto3, `requests`, a DB driver without async support) for 10,000 items sequentially and takes an hour, although the work is 95% waiting on I/O.
- CPU-heavy work (PDF rendering, image resizing, hashing) blocks an asyncio event loop or an Express handler, so every other request stalls.
- Nobody can answer "how many of these run at once?" because the number is unbounded.

Scenario: a nightly job downloads 20,000 invoice PDFs from a vendor API that only offers a blocking SDK. Sequentially it takes 3 hours; one thread per invoice crashes the box.

## Solution and structure
```
submit(task) ──► [ task queue ] ──► worker 1 ─┐
                                   worker 2 ─┼─► Future (result / exception)
                                   worker N ─┘
```
- **Pool / Executor**: owns N workers and an internal queue; exposes `submit()` / `map()` and `shutdown()`.
- **Task**: a callable plus arguments.
- **Worker**: loops forever: take task, run it, store result or exception in the task's Future.
- **Future**: handle the caller uses to get the result (see Future / Promise).

Choose the worker type by workload, not by habit:
- **I/O-bound, blocking library** → threads (`ThreadPoolExecutor`). The GIL is released during blocking I/O.
- **I/O-bound, async library available** → no pool; asyncio with a `Semaphore` for the bound.
- **CPU-bound pure Python** → processes (`ProcessPoolExecutor`). On the standard (GIL) build threads do not run Python bytecode in parallel. The free-threaded build (3.13t experimental, 3.14t officially supported but not the default) removes that limit, but C extensions must declare support, and single-thread speed is somewhat lower. Measure before relying on it. 3.14 also adds `InterpreterPoolExecutor` (sub-interpreters, each with its own GIL).
- **CPU-bound in NumPy / C code that releases the GIL** → threads usually work.

## When to use
- Many independent, short tasks, where the bound on concurrency matters (DB connections, vendor rate limits, RAM).
- Blocking calls you cannot make async; offload them from an event loop with `asyncio.to_thread` / `loop.run_in_executor`.
- CPU work in Node: `worker_threads` behind a pool (Piscina) so the event loop stays responsive.

## When NOT to use
- The work is I/O and an async client exists. Use asyncio / Promises with a semaphore: cheaper than threads, no shared-state races.
- One long-lived background loop. A single dedicated thread or an asyncio task is clearer.
- Tasks that depend on each other's results inside the same pool. They can deadlock when all workers wait on queued tasks; restructure as a pipeline or use a work-stealing / fork-join runtime.
- Durable work that must survive a restart. Use a Task Queue (Celery, RQ, SQS consumers), not an in-memory pool.
- Browser code doing I/O. `fetch` is already non-blocking; a Web Worker pool only helps CPU-bound work.

## Example — Python
Stdlib only. `Executor` is a context manager; leaving the block waits for all tasks.
```python
from concurrent.futures import ThreadPoolExecutor, as_completed
from dataclasses import dataclass
import urllib.request

@dataclass(frozen=True)
class DownloadResult:
    invoice_id: str
    size: int

def download_invoice(invoice_id: str) -> DownloadResult:
    url = f"https://vendor.example.com/invoices/{invoice_id}.pdf"
    with urllib.request.urlopen(url, timeout=10) as resp:  # blocking I/O releases the GIL
        body = resp.read()
    return DownloadResult(invoice_id, len(body))

def download_all(ids: list[str], max_workers: int = 16) -> tuple[list[DownloadResult], dict[str, Exception]]:
    ok: list[DownloadResult] = []
    failed: dict[str, Exception] = {}
    with ThreadPoolExecutor(max_workers=max_workers, thread_name_prefix="invoice") as pool:
        futures = {pool.submit(download_invoice, i): i for i in ids}
        for fut in as_completed(futures):
            try:
                ok.append(fut.result())
            except Exception as exc:  # the worker's exception is re-raised here
                failed[futures[fut]] = exc
    return ok, failed

# From async code, offload a single blocking call instead:
#   result = await asyncio.to_thread(download_invoice, "INV-1")
```
For 1M items, do not `submit` all at once (each Future holds memory); feed in chunks or put a bounded queue in front (Producer-Consumer).

## Example — TypeScript
In Node, I/O needs no threads. The "pool" for I/O is a concurrency limit over Promises. For CPU work use `worker_threads` via Piscina.
```ts
// Bounded concurrency over async I/O: N "workers" pull from a shared iterator.
export async function mapPool<T, R>(
  items: readonly T[],
  limit: number,
  fn: (item: T) => Promise<R>,
): Promise<PromiseSettledResult<R>[]> {
  const results: PromiseSettledResult<R>[] = new Array(items.length);
  let next = 0;
  async function worker(): Promise<void> {
    while (next < items.length) {
      const i = next++; // safe: single-threaded, no await between read and increment
      try {
        results[i] = { status: 'fulfilled', value: await fn(items[i]) };
      } catch (reason) {
        results[i] = { status: 'rejected', reason };
      }
    }
  }
  await Promise.all(Array.from({ length: Math.min(limit, items.length) }, worker));
  return results;
}

// CPU-bound: a real thread pool (npm i piscina)
// import Piscina from 'piscina';
// const pool = new Piscina({ filename: new URL('./resize.worker.js', import.meta.url).href, maxThreads: 4 });
// const thumb = await pool.run({ path: '/tmp/a.png', width: 200 });
```

## In the wild
- Python: `concurrent.futures.ThreadPoolExecutor` / `ProcessPoolExecutor` / `InterpreterPoolExecutor` (3.14), `asyncio.to_thread`, `loop.run_in_executor`. FastAPI/Starlette run plain `def` endpoints and sync dependencies in a threadpool (AnyIO, default 40 tokens).
- Uvicorn/Gunicorn worker processes; Celery `--concurrency` with prefork/threads/gevent pools.
- Node: libuv threadpool (default 4, `UV_THREADPOOL_SIZE`) runs `fs`, `crypto.pbkdf2`, `dns.lookup`, zlib. `worker_threads`, Piscina.
- Browser: Web Workers (hand-rolled pool or Comlink).
- DB connection pools (SQLAlchemy `QueuePool`, `pg.Pool`) are the same idea applied to connections (see Object Pool).

## Trade-offs
| Pros | Cons |
|---|---|
| Bounded resource use; predictable load on downstreams | Pool size is a tuning knob that is easy to get wrong |
| Worker creation cost paid once | Shared mutable state still needs locks; races remain possible |
| Simple Futures API for results and errors | GIL: threads give no CPU parallelism on the default build |
| Easy to swap thread / process / interpreter executors | Processes need picklable tasks and pay serialization cost |

## Pitfalls
- **Swallowed exceptions.** A task's exception is stored in its Future; if you never call `.result()` it is silently lost. Always collect results or add a done-callback that logs.
- **Pool-in-pool deadlock.** A task that `submit`s to the same pool and waits on the result can block forever once all workers do it. Use a separate pool or no nested waits.
- **Wrong size.** Rule of thumb: CPU-bound ≈ number of cores; I/O-bound ≈ cores × (1 + wait/compute), capped by the downstream's limits (DB pool size, API quota). Measure.
- **Unbounded submission.** `submit` in a loop over millions of items queues them all in memory. Chunk, or use a bounded queue.
- **Thread-unsafe clients.** Some SDK objects are not thread-safe (e.g. a `requests.Session` is commonly shared but not documented as safe; boto3 `Session` is not, clients are). Create per-thread objects with `threading.local()` or the executor's `initializer=`.
- **Forgetting shutdown.** Use `with` so threads are joined; on Ctrl-C use `shutdown(cancel_futures=True)`.
- **Blocking the event loop anyway.** Calling `fut.result()` inside an `async def` blocks the loop; `await asyncio.wrap_future(fut)` or use `to_thread`.

## Refactoring path
- **Introduce it:** find the spot that spawns threads per item or runs blocking calls sequentially. Extract the per-item work into a pure function that returns a value. Wrap the loop in `with ThreadPoolExecutor(max_workers=small) as pool: pool.map(...)`. Add error collection, then tune the size under realistic load.
- **Remove it / simplify:** if the downstream library gains an async API, replace the pool with `asyncio.TaskGroup` plus a `Semaphore`. If profiling shows the pool never runs more than one task, go back to a plain loop.

## Related patterns
- [Producer-Consumer](../concurrency/producer-consumer.md): the pool is a consumer group behind a queue; add a bounded queue for backpressure.
- [Future / Promise](../concurrency/future-promise.md): what `submit` returns.
- [Work Stealing](../concurrency/work-stealing.md): a pool with per-worker queues for recursive or uneven tasks.
- [Bulkhead](../cloud/bulkhead.md): separate pools per dependency so one slow service cannot take every worker.
- [Object Pool](../creational/object-pool.md): reuses resources (connections) rather than executors.
- [Leader-Follower](../concurrency/leader-follower.md): a pool variant where threads take turns waiting on the event source.
