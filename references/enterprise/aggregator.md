# Aggregator

**Category:** Enterprise integration · **Also known as:** Scatter-Gather (with fan-out), API composition, BFF aggregation · **Related:** Splitter, Message Translator, Backend for Frontend, Facade, Future/Promise, Saga

## Intent
Collect related messages or responses and combine them into one result once a completeness condition is met (all arrived, enough arrived, or timeout).

## Problem
- A page needs data from five services; the browser makes five sequential requests, so it is slow and chatty.
- A Splitter broke an order into items processed separately; something must reassemble the results.
- Quotes are requested from several suppliers; the best answer within 2 seconds should win.
- Scenario: an Angular order-details page needs order, customer, shipment tracking and invoice status from four backends.

## Solution and structure
An aggregator is **stateful** and answers three questions:
1. **Correlation:** which messages belong together? (correlation ID, order ID)
2. **Completeness:** when is the aggregate done? (all N expected, first K, timeout, external "end" marker)
3. **Aggregation:** how to combine? (merge, pick best, sum, collect list)

```
              ┌─> orders-svc ─────┐
request ──fan─┼─> customers-svc ──┼──> Aggregator (correlate, wait ≤ 800 ms, merge) ──> response
              └─> shipping-svc ───┘          partial result + flags if one is slow
```
Two settings:
- **Synchronous (API composition / BFF):** fan out concurrently, await with a timeout, merge in memory.
- **Asynchronous (messaging):** store partial state keyed by correlation ID (Redis/DB), emit the aggregate when complete, expire incomplete groups.

## When to use
- One client view needs data from several services (BFF, GraphQL resolvers, API gateway composition).
- Reassembling results after a Splitter or parallel workers.
- Scatter-gather: best quote, fastest replica, quorum reads.
- Batching many small messages into one (bulk inserts, digest emails).

## When NOT to use
- The data lives in one database: a JOIN or a read model is simpler and consistent.
- Composition across many services on every request at high QPS: consider a precomputed read model (CQRS) instead.
- You cannot tolerate partial results but also cannot wait: rethink the boundary.

## Example — Python
Async API composition with `httpx`: one mandatory call, then concurrent optional calls with per-call timeouts and partial results.

```python
import asyncio
from typing import Any
import httpx

async def _fetch(client: httpx.AsyncClient, url: str, timeout: float) -> dict[str, Any] | None:
    try:
        r = await asyncio.wait_for(client.get(url), timeout)
        r.raise_for_status()
        return r.json()
    except (httpx.HTTPError, TimeoutError):
        return None                               # degrade instead of failing the page

async def order_details(client: httpx.AsyncClient, order_id: str) -> dict[str, Any]:
    order = await _fetch(client, f"http://orders/orders/{order_id}", 0.5)
    if order is None:
        raise LookupError(order_id)               # the core part is mandatory
    customer, shipment, invoice = await asyncio.gather(
        _fetch(client, f"http://customers/customers/{order['customer_id']}", 0.5),
        _fetch(client, f"http://shipping/shipments?order_id={order_id}", 0.8),
        _fetch(client, f"http://billing/invoices?order_id={order_id}", 0.8),
    )
    return {
        "order": order,
        "customer": customer,
        "shipment": shipment,
        "invoice": invoice,
        "partial": any(x is None for x in (customer, shipment, invoice)),
    }
```
`gather` is safe here because `_fetch` never raises; with raising calls prefer `asyncio.TaskGroup` (3.11+) so siblings are cancelled on failure.

## Example — TypeScript
Angular: aggregate several HTTP calls with RxJS `forkJoin`, tolerating a failing optional source.

```ts
import { Injectable, inject } from '@angular/core';
import { HttpClient } from '@angular/common/http';
import { catchError, forkJoin, map, Observable, of, switchMap, timeout } from 'rxjs';

interface Order { id: string; customerId: string }
interface Customer { id: string; name: string }
interface Shipment { status: string; eta: string }
export interface OrderDetails { order: Order; customer: Customer | null; shipment: Shipment | null }

@Injectable({ providedIn: 'root' })
export class OrderDetailsFacade {
  private readonly http = inject(HttpClient);

  private optional<T>(src: Observable<T>): Observable<T | null> {
    return src.pipe(timeout(800), catchError(() => of(null)));
  }

  load(orderId: string): Observable<OrderDetails> {
    return this.http.get<Order>(`/api/orders/${orderId}`).pipe(
      switchMap((order) => forkJoin({
        customer: this.optional(this.http.get<Customer>(`/api/customers/${order.customerId}`)),
        shipment: this.optional(this.http.get<Shipment>(`/api/shipments/${orderId}`)),
      }).pipe(map((parts) => ({ order, ...parts })))),
    );
  }
}
```
`forkJoin` emits only when every source completes; each must emit at least once (HTTP calls do). Better still, do this aggregation in a BFF so the browser makes one request.

## In the wild
- GraphQL servers/federation (Apollo Router), API gateways with composition, BFF services.
- Apache Camel `aggregate()` with completion size/timeout; Spring Integration Aggregator.
- AWS Step Functions `Parallel`/`Map` states; Kafka Streams windowed aggregations; RxJS `forkJoin`/`combineLatest`/`zip`; `Promise.allSettled`.

## Trade-offs
| Pros | Cons |
|---|---|
| One round trip for the client; parallel fan-out | Latency = slowest dependency (unless timeouts) |
| Clients decoupled from service topology | Aggregator becomes a coupling point for many services |
| Partial results give graceful degradation | Async aggregators hold state: expiry, persistence, memory |
| Natural place for caching and shaping responses | Consistency across sources is not guaranteed |

## Pitfalls
- **No timeout:** one slow service stalls everything. Fix: per-call timeouts and an overall deadline.
- **All-or-nothing by accident** (`Promise.all`, `gather` with raising calls). Fix: mark mandatory vs optional parts; `allSettled`/`catchError`.
- **Unbounded async state** for groups that never complete. Fix: completion timeout and TTL on stored partials.
- **Sequential calls** where parallel is possible. Fix: fan out after resolving only the true dependencies.
- **N+1 fan-out** (per list item call). Fix: batch endpoints or DataLoader-style batching.

## Refactoring path
- **Introduce it:** find screens making several dependent requests; add a BFF/aggregator endpoint that fans out concurrently with timeouts; switch the client to it; then cache or replace hot paths with a read model.
- **Remove it / simplify:** if the aggregated data ends up in one service/DB, replace the fan-out with a single query or read model and delete the composition code.

## Related patterns
- [Splitter](../enterprise/splitter.md) — the inverse; often paired (split → process → aggregate).
- [Message Translator](../enterprise/message-translator.md) — normalizes the parts before merging.
- [Facade](../structural/facade.md) — a BFF/aggregator is a facade over several services.
- [Future / Promise](../concurrency/future-promise.md) — the mechanism behind concurrent fan-out.
- [CQRS](../cloud/cqrs.md) — precomputed read models replace runtime aggregation at scale.
