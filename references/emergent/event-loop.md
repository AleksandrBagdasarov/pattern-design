# Event Loop

**Category:** Emergent / modern · **Also known as:** Reactor, run loop, message loop · **Related:** Future / Promise, Producer-Consumer, Thread Pool, Game Loop, Active Object

## Intent
Run many concurrent I/O-bound activities on a single thread by multiplexing non-blocking I/O: one loop waits for readiness events (sockets, timers, signals) and dispatches short callbacks or resumes coroutines, so no thread is parked per connection.

## Problem
- Thousands of concurrent connections, each mostly waiting on the network. A thread per connection costs memory and context switches; shared state across threads needs locks.
- Scenario: a FastAPI service fans out to three internal APIs per request. With sync `requests` in an `async def` route, one slow upstream freezes *every* request on that worker, because the loop thread is blocked.
- Scenario: an Angular page computes a 50k-row aggregate synchronously in a click handler; the UI freezes, no spinner renders.
- Smells: `time.sleep`, sync DB drivers or CPU-heavy loops inside `async def`; "unexpected" ordering of logs between `setTimeout`, `Promise.then` and `await`.

## Solution and structure
```
        ┌──────────────── one thread ────────────────┐
        │  run one macrotask (I/O callback, timer)   │
        │  drain ALL microtasks (promise reactions)  │◄─ JS
        │  (browser) render if due: rAF, style, paint│
        │  poll OS for ready I/O (epoll/kqueue/IOCP) │
        └────────────────────────────────────────────┘
```
- **Loop**: owns a ready queue, timers and an I/O selector.
- **Tasks/callbacks**: short, non-blocking units; long work must yield (`await`) or move elsewhere.
- **Futures/promises**: placeholders resolved by I/O completion, resuming suspended coroutines.

**JS ordering rules** (browser and Node): the current synchronous code runs to completion; then *all* microtasks (`Promise.then`, `await` continuations, `queueMicrotask`, `MutationObserver`) are drained; then one macrotask (`setTimeout`, `setInterval`, I/O, `MessageChannel`, UI events). Node adds `process.nextTick`, which runs before promise microtasks, and phases (timers → poll → check (`setImmediate`) → close). So: `sync → nextTick → microtasks → timers/I/O`.

**asyncio**: the loop runs ready callbacks FIFO; `await` on an incomplete future suspends the coroutine and returns control; `call_soon` ≈ microtask-ish (next iteration), `call_later` is a timer. There is no separate microtask queue — an `await` on an already-done coroutine does not yield at all. `await asyncio.sleep(0)` forces a yield.

**Blocking the loop**: anything that holds the thread (CPU loop, sync I/O) delays every other task. Offload: `asyncio.to_thread` / `run_in_executor` (blocking I/O), `ProcessPoolExecutor` (CPU), Web Workers / `worker_threads` (JS CPU).

## When to use
- I/O-bound servers and clients with high concurrency (API gateways, websockets, scrapers, chat).
- UI threads (browser, Angular) — you don't choose it, but you must design for it.
- Coordinating many timers and network calls with shared state and no locks.

## When NOT to use
- CPU-bound work (image processing, ML inference, big aggregations): use processes, workers or native extensions.
- Codebases built on blocking libraries (sync SQLAlchemy, `requests`, boto3): a thread pool or sync framework (Flask, sync FastAPI `def` routes, which run in a threadpool) is simpler than half-async code.
- Low-concurrency scripts: sequential sync code is easier to read and debug.

## Example — Python
Fan-out with a timeout, offloading a blocking SDK and a CPU step correctly.

```python
import asyncio
from concurrent.futures import ProcessPoolExecutor
import httpx

CPU_POOL = ProcessPoolExecutor(max_workers=4)

def score(order_lines: list[dict]) -> float:           # CPU-bound, pure
    return sum(l["qty"] * l["price"] for l in order_lines) ** 0.5

def legacy_fraud_check(order_id: str) -> bool:          # blocking SDK call
    ...

async def enrich_order(client: httpx.AsyncClient, order: dict) -> dict:
    async with asyncio.timeout(2.0):                     # 3.11+
        async with asyncio.TaskGroup() as tg:            # concurrent, cancels siblings on error
            user = tg.create_task(client.get(f"/users/{order['user_id']}"))
            stock = tg.create_task(client.get(f"/stock/{order['sku']}"))
            fraud = tg.create_task(asyncio.to_thread(legacy_fraud_check, order["id"]))
    loop = asyncio.get_running_loop()
    risk = await loop.run_in_executor(CPU_POOL, score, order["lines"])
    return {**order, "user": user.result().json(), "stock": stock.result().json(),
            "fraud": fraud.result(), "risk": risk}

# Wrong: blocks every request on this worker
# async def bad(): time.sleep(1); requests.get(url)
```
Enable `asyncio` debug mode (`PYTHONASYNCIODEBUG=1`) to log callbacks that run longer than 100 ms.

## Example — TypeScript
The classic ordering trap, then keeping an Angular component responsive.

```ts
console.log('1 sync');
setTimeout(() => console.log('7 timeout (macrotask)'), 0);
Promise.resolve().then(() => console.log('4 microtask'));
queueMicrotask(() => console.log('5 microtask'));
(async () => { console.log('2 async fn body runs sync'); await null; console.log('6 after await (microtask)'); })();
console.log('3 sync');
// Output: 1, 2, 3, 4, 5, 6, 7. (Node: process.nextTick callbacks would run before 4.)

import { Component, signal } from '@angular/core';

@Component({
  selector: 'app-report',
  template: `<button (click)="run()">Build</button> @if (busy()) {<span>Working…</span>} {{ total() }}`,
})
export class Report {
  busy = signal(false);
  total = signal<number | null>(null);

  run(): void {
    this.busy.set(true);
    const worker = new Worker(new URL('./report.worker', import.meta.url), { type: 'module' });
    worker.onmessage = ({ data }: MessageEvent<number>) => {
      this.total.set(data); this.busy.set(false); worker.terminate();
    };
    worker.postMessage({ rows: 50_000 });    // heavy loop runs off the UI thread
  }
}
```

## In the wild
- Python: `asyncio` (selector/proactor loops), `uvloop` (libuv-based), Trio/AnyIO, uvicorn, aiohttp, FastAPI `async def` routes (sync `def` routes run in a threadpool).
- JS: browser HTML event loop, Node.js on libuv, Deno/Bun; Angular change detection scheduled from loop events (zoneless uses microtask/rAF scheduling).
- Elsewhere: nginx, Redis (single-threaded command loop), Netty, Tokio, GUI toolkits (Qt, Tk) main loops.

## Trade-offs
| Pros | Cons |
|---|---|
| Huge concurrency with little memory | One blocking call stalls everything |
| No locks for state touched only between awaits | Function coloring: async spreads through call stacks |
| Deterministic, single-threaded reasoning | Ordering subtleties (microtasks vs macrotasks) |
| Cheap timers and cancellation | CPU work needs separate processes/workers |

## Pitfalls
- Calling blocking code in `async def` (sync DB drivers, `requests`, `time.sleep`, big JSON parse). Use async libraries or `asyncio.to_thread`.
- Fire-and-forget `asyncio.create_task` without keeping a reference: the task may be garbage-collected mid-flight. Keep references or use `TaskGroup`.
- Assuming `await` always yields: awaiting an already-completed future or a coroutine with no real suspension doesn't. Races still exist across real awaits (check-then-act).
- Microtask starvation: recursive promise chains or `queueMicrotask` loops prevent rendering and timers.
- `setTimeout(fn, 0)` is not "immediately": browsers clamp nested timers to ≥4 ms and it runs after microtasks.
- Mixing loops: calling `asyncio.run` inside a running loop, or creating clients bound to a different loop in tests (use `pytest-asyncio`/`anyio` fixtures consistently).

## Refactoring path
- **Introduce it:** start at the edges — switch the HTTP/DB clients to async versions, convert call chains to `async` bottom-up, wrap remaining blocking calls with `to_thread`, then add concurrency (`TaskGroup`, `gather`) where calls are independent.
- **Remove it / simplify:** if most work is CPU-bound or the libraries are sync, go back to sync handlers with a thread/process pool; one coherent model beats half-async code.

## Related patterns
- [Future / Promise](../concurrency/future-promise.md) — the result handles the loop resolves.
- [Thread Pool](../concurrency/thread-pool.md) — where blocking work is offloaded.
- [Producer-Consumer](../concurrency/producer-consumer.md) — `asyncio.Queue` gives backpressure between loop tasks.
- [Game Loop](../emergent/game-loop.md) — a time-driven loop, often running on top of an event loop.
- [Active Object](../concurrency/active-object.md) — method calls queued to one executor thread, a close cousin.
