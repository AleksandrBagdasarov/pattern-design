# Future / Promise

**Category:** Concurrency · **Also known as:** Deferred, Task, Eventual value, CompletableFuture · **Related:** Thread Pool, Active Object, Guarded Suspension, Observer, Event Loop

## Intent
Represent a result that is not available yet as a first-class object, so the caller can start an operation, keep going, and later wait for, combine, or react to its value or error.

## Problem
- Callback nesting ("callback hell"): each async step needs the previous one's result, and error handling is duplicated at every level.
- Code waits on slow calls one after another although they are independent (three 300 ms service calls take 900 ms instead of 300).
- Results and errors from background threads are passed back through shared variables and flags, with races.
- Several callers trigger the same expensive load; there is no object to share.

Scenario: an order-details endpoint needs the order, the customer profile and the shipment status from three services. Sequential awaits cost the sum of latencies; the page needs the max.

## Solution and structure
```
producer side:  promise/future ──set_result(v) | set_exception(e)──►  state: pending → fulfilled | rejected
consumer side:  await f / f.result() / f.then(cb) / f.add_done_callback(cb)
combinators:    all / gather, race / wait(FIRST_COMPLETED), any, allSettled, timeout
```
- **Future** (read side): pending, then exactly once fulfilled with a value or rejected with an error.
- **Promise / resolver** (write side): the producer's handle to complete it. In JS the `Promise` constructor's `resolve/reject` (or `Promise.withResolvers()`, ES2024); in Python `asyncio.Future.set_result` / `concurrent.futures.Future.set_result`.
- **Continuations**: callbacks or `await` run when the value is ready.
- **Combinators**: join many futures, race them, add timeouts, cancel.

Python has two unrelated Future types: `concurrent.futures.Future` (thread-safe, blocking `.result()`) and `asyncio.Future` (loop-bound, awaitable, not thread-safe). Bridge with `asyncio.wrap_future` / `loop.run_in_executor` and `asyncio.run_coroutine_threadsafe`.

## When to use
- Any asynchronous operation with a single result: an HTTP call, a DB query, a job submitted to a pool.
- Running independent operations concurrently and joining their results.
- Sharing one in-flight operation among several waiters (memoize the future).
- Bridging callback-style APIs into `async`/`await`.

## When NOT to use
- **Multiple values over time** (websocket messages, user input, progress updates): use a stream (async iterator, RxJS Observable, signals). A Promise resolves once.
- **Cancellation is essential and the platform's futures cannot cancel** the work: JS Promises are not cancellable; pass an `AbortSignal`, or use an Observable whose unsubscribe aborts the request (Angular `HttpClient`).
- Purely synchronous, fast code: wrapping it in a Promise adds a microtask hop and complexity.
- Manually creating `asyncio.Future`/`new Promise` around something that already returns one ("explicit construction anti-pattern"). Just `await`/return it.

## Example — Python
Fan-out with structured concurrency (`TaskGroup`, 3.11+), a timeout, and a hand-made future bridging a callback API.
```python
import asyncio
from dataclasses import dataclass
from typing import Any
import httpx

@dataclass
class OrderDetails:
    order: dict[str, Any]
    customer: dict[str, Any]
    shipment: dict[str, Any] | None

async def get_json(client: httpx.AsyncClient, url: str) -> dict[str, Any]:
    r = await client.get(url)
    r.raise_for_status()
    return r.json()

async def order_details(client: httpx.AsyncClient, order_id: str, customer_id: str) -> OrderDetails:
    async with asyncio.timeout(2.0):                         # one deadline for the whole fan-out
        async with asyncio.TaskGroup() as tg:                # if one fails, the others are cancelled
            order = tg.create_task(get_json(client, f"/orders/{order_id}"))
            customer = tg.create_task(get_json(client, f"/customers/{customer_id}"))
            shipment = tg.create_task(get_json(client, f"/shipments?order={order_id}"))
    return OrderDetails(order.result(), customer.result(), shipment.result())

def on_payment_confirmed(gateway: Any, payment_id: str) -> asyncio.Future[str]:
    """Wrap a callback-style SDK; callbacks may arrive on another thread."""
    loop = asyncio.get_running_loop()
    fut: asyncio.Future[str] = loop.create_future()
    def callback(status: str) -> None:                       # called from the SDK's thread
        loop.call_soon_threadsafe(fut.set_result, status)    # asyncio.Future is not thread-safe
    gateway.subscribe(payment_id, callback)
    return fut
```
Use `asyncio.gather(..., return_exceptions=True)` when partial results are acceptable; TaskGroup is all-or-nothing.

## Example — TypeScript
Promise combinators in Node, and in Angular the Observable form that supports cancellation.
```ts
import { Injectable, inject } from '@angular/core';
import { HttpClient } from '@angular/common/http';
import { forkJoin, timeout, catchError, of } from 'rxjs';

interface Order { id: string; customerId: string }
interface Customer { id: string; name: string }
interface Shipment { status: string }

// Node / plain TS: concurrent fan-out with a deadline
export async function orderDetails(base: string, id: string, signal?: AbortSignal) {
  const deadline = AbortSignal.any([AbortSignal.timeout(2000), ...(signal ? [signal] : [])]);
  const get = <T>(path: string) =>
    fetch(`${base}${path}`, { signal: deadline })
      .then((r) => (r.ok ? (r.json() as Promise<T>) : Promise.reject(new Error(`${r.status} ${path}`))));
  const [order, shipment] = await Promise.allSettled([get<Order>(`/orders/${id}`), get<Shipment>(`/shipments/${id}`)]);
  if (order.status === 'rejected') throw order.reason;       // order is required, shipment is optional
  return { order: order.value, shipment: shipment.status === 'fulfilled' ? shipment.value : null };
}

// Angular: Observables are lazy and cancel the HTTP request on unsubscribe
@Injectable({ providedIn: 'root' })
export class OrderDetailsService {
  private http = inject(HttpClient);
  load(order: Order) {
    return forkJoin({                                          // Promise.all for observables
      order: of(order),
      customer: this.http.get<Customer>(`/api/customers/${order.customerId}`),
      shipment: this.http.get<Shipment>(`/api/shipments/${order.id}`).pipe(catchError(() => of(null))),
    }).pipe(timeout(2000));
  }
}
```
Use `firstValueFrom(obs$)` when an API needs a Promise. Angular's `resource()` / `httpResource()` wrap async loading into signals; their API has been evolving across recent versions, so check angular.dev for the current status.

## In the wild
- Python: `asyncio.Task`/`Future`, `asyncio.gather`, `TaskGroup`, `asyncio.timeout`, `concurrent.futures.Future`, `as_completed`, AnyIO task groups, Trio nurseries.
- JS: `Promise`, `Promise.all/allSettled/race/any`, `Promise.withResolvers`, `AbortController`; RxJS `forkJoin`, `firstValueFrom`, `lastValueFrom`.
- Java `CompletableFuture`; C# `Task<T>`; Rust `Future`; Scala `Future`/`Promise`.
- Distributed: Celery `AsyncResult`, Ray `ObjectRef`, Dask futures.

## Trade-offs
| Pros | Cons |
|---|---|
| Linear async code with `await`; errors propagate | "Function coloring": async spreads up the call stack |
| Easy concurrency via combinators | JS Promises are eager and not cancellable |
| A shareable handle to in-flight work | Unobserved rejections / un-awaited tasks hide errors |
| Clean bridge from callbacks and threads | Two Future types in Python are easy to confuse |

## Pitfalls
- **Sequential awaits for independent work** (`a = await f(); b = await g()`). Start both, then await (TaskGroup / `Promise.all`).
- **Fire-and-forget without a reference** (`asyncio.create_task(x())` discarded): the task can be garbage-collected and its exception is lost. Keep references or use a TaskGroup.
- **Floating Promises** in TS: enable `@typescript-eslint/no-floating-promises`.
- **Blocking `.result()` inside async code** stalls the loop. Await instead.
- **Setting an asyncio Future from another thread** directly. Use `call_soon_threadsafe`.
- **`Promise.all` fails fast but does not cancel** the other operations. Pass an `AbortSignal` to stop them.
- **`async` inside `forEach`** does not wait. Use `for...of` with `await` or `Promise.all(items.map(...))`.

## Refactoring path
- **Introduce it:** convert callback APIs to return Promises/futures (`util.promisify`, `loop.create_future`). Replace nested callbacks with `await`. Find independent sequential awaits and run them concurrently with a combinator. Add timeouts.
- **Remove it / simplify:** if a "future" only wraps synchronous work, make it a plain function. If a Promise is used for a stream of values, switch to an Observable or async iterator.

## Related patterns
- [Thread Pool](../concurrency/thread-pool.md): `submit` returns a future for work on another thread.
- [Active Object](../concurrency/active-object.md): every method call returns a future.
- [Guarded Suspension](../concurrency/guarded-suspension.md): what `.result()` does internally.
- [Observer](../behavioral/observer.md): the multi-value cousin (Observables).
- [Event Loop](../emergent/event-loop.md): the runtime that schedules promise continuations.
