# Bulkhead

**Category:** Cloud / distributed · **Also known as:** Resource isolation, Compartmentalization, Pool partitioning · **Related:** Circuit Breaker, Rate Limiter, Thread Pool, Lock Striping, Sharding

## Intent
Partition resources (threads, connections, concurrency slots, instances) into isolated pools per dependency, tenant or workload, so that exhaustion in one partition cannot sink the others, like watertight compartments in a ship's hull.

## Problem
- A slow reporting endpoint holds all 40 threadpool slots, and health checks and checkout requests time out too.
- One recommendation service hangs; every request thread waits on it, and the app can no longer reach the healthy inventory service.
- A single enterprise tenant runs a bulk export that consumes the shared DB connection pool; all other tenants see errors.
- Batch jobs and interactive traffic share the same workers, so a nightly import ruins daytime latency when it runs late.

Scenario: a product page calls inventory (critical), reviews (nice to have) and recommendations (nice to have). When recommendations become slow, the product page must still render with inventory.

## Solution and structure
```
               ┌─ pool A: inventory   (max 20 concurrent) ──► inventory svc
 requests ─────┼─ pool B: reviews     (max 5 concurrent)  ──► reviews svc
               └─ pool C: recommend.  (max 5, queue 0)    ──► recommendations svc
```
- **Partition key**: dependency, tenant, priority class (interactive vs batch) or endpoint.
- **Isolated resource per partition**: semaphore, thread pool, connection pool, queue, or entire deployments / node pools.
- **Overflow policy**: reject immediately, wait with a short timeout, or queue with a bound.
- **Sizing**: from Little's law: concurrency ≈ throughput × latency, plus headroom.

Levels from fine to coarse: a semaphore per client → separate thread/connection pools → separate worker processes or queues → separate deployments, node pools, DB replicas or cells per tenant group.

## When to use
- The service calls several dependencies with different criticality and failure modes.
- Multi-tenant systems where one tenant must not starve others.
- Mixing latency-sensitive and batch workloads.
- Together with a Circuit Breaker: the bulkhead caps damage while the breaker detects the failure.

## When NOT to use
- One dependency, low traffic: a timeout and a single pool are enough.
- Partitions so small they waste capacity: idle reserved slots in one pool cannot help a busy one. Start with a few coarse partitions.
- When the platform already isolates (separate Lambda functions with reserved concurrency, separate Kubernetes deployments); do not duplicate in code without reason.
- When the real problem is unbounded rate from callers: use a Rate Limiter at the edge.

## Example — Python
asyncio semaphores per dependency, with fail-fast overflow, plus separate `httpx` connection pools.
```python
import asyncio
from contextlib import asynccontextmanager
from collections.abc import AsyncIterator
import httpx

class BulkheadFull(Exception): ...

class Bulkhead:
    def __init__(self, name: str, max_concurrent: int, max_wait: float = 0.0) -> None:
        self.name, self._sem, self._max_wait = name, asyncio.Semaphore(max_concurrent), max_wait

    @asynccontextmanager
    async def slot(self) -> AsyncIterator[None]:
        if self._sem.locked() and not self._max_wait:
            raise BulkheadFull(self.name)                # fail fast: no free slot, no waiting
        try:
            await asyncio.wait_for(self._sem.acquire(), timeout=self._max_wait or None)
        except TimeoutError:
            raise BulkheadFull(self.name) from None      # waited max_wait: do not queue forever
        try:
            yield
        finally:
            self._sem.release()

def limits(n: int) -> httpx.Limits:                     # separate connection pool per dependency
    return httpx.Limits(max_connections=n, max_keepalive_connections=n)

inventory = httpx.AsyncClient(base_url="http://inventory", limits=limits(20), timeout=1.0)
recs = httpx.AsyncClient(base_url="http://recs", limits=limits(5), timeout=0.5)
inventory_bh = Bulkhead("inventory", 20, max_wait=0.2)
recs_bh = Bulkhead("recommendations", 5)

async def product_page(sku: str) -> dict:
    async with inventory_bh.slot():                      # critical: errors propagate
        stock = (await inventory.get(f"/stock/{sku}")).json()
    try:
        async with recs_bh.slot():                       # optional: degrade on overflow or error
            related = (await recs.get(f"/related/{sku}")).json()
    except (BulkheadFull, httpx.HTTPError):
        related = []
    return {"sku": sku, "stock": stock, "related": related}
```
In FastAPI, sync endpoints share one AnyIO threadpool (40 tokens by default); a slow sync dependency there is a shared-resource risk. Make it async or give it its own executor.

## Example — TypeScript
`cockatiel` provides a bulkhead policy for Node; the same idea can be a small semaphore.
```ts
export class Bulkhead {
  private active = 0;
  private readonly waiters: Array<() => void> = [];

  constructor(readonly name: string, private readonly limit: number, private readonly maxQueue = 0) {}

  async run<T>(task: () => Promise<T>): Promise<T> {
    if (this.active >= this.limit) {
      if (this.waiters.length >= this.maxQueue) throw new Error(`bulkhead ${this.name} full`);
      await new Promise<void>((resolve) => this.waiters.push(resolve));
    } else {
      this.active++;
    }
    try {
      return await task();
    } finally {
      const next = this.waiters.shift();
      if (next) next();            // hand the slot directly to the next waiter
      else this.active--;
    }
  }
}

const recsBulkhead = new Bulkhead('recommendations', 5);
export async function related(sku: string): Promise<string[]> {
  try {
    return await recsBulkhead.run(() =>
      fetch(`http://recs/related/${sku}`, { signal: AbortSignal.timeout(500) }).then((r) => r.json()));
  } catch {
    return [];                     // degrade instead of failing the page
  }
}
```
Node's `http.Agent({ maxSockets })` per upstream is a connection-level bulkhead. In the browser, separate concerns rather than threads: e.g. RxJS `mergeMap(fn, 3)` caps concurrent uploads so they do not saturate the connection for API calls.

## In the wild
- Python: `asyncio.Semaphore`, AnyIO `CapacityLimiter`, separate `ThreadPoolExecutor`s, httpx `Limits`, SQLAlchemy separate engines/pools, Celery dedicated queues and workers per job type.
- JS: `cockatiel` bulkhead, `p-limit`, `http.Agent` `maxSockets`, BullMQ queue-per-workload.
- JVM/.NET: Resilience4j Bulkhead / ThreadPoolBulkhead, Polly Bulkhead (now "concurrency limiter" in Polly v8).
- Infrastructure: Kubernetes resource requests/limits, namespaces and node pools; Envoy `max_connections` / `max_pending_requests`; AWS Lambda reserved concurrency per function; cell-based architectures (AWS) and per-tenant DB shards.

## Trade-offs
| Pros | Cons |
|---|---|
| Failure in one dependency or tenant stays contained | Reserved capacity can sit idle while another pool is starved |
| Critical paths keep resources under load | More pools to size, monitor and tune |
| Clear per-dependency limits double as backpressure | Fail-fast rejections need graceful handling upstream |
| Works at every level (code, process, infra) | Too many tiny partitions fragment capacity |

## Pitfalls
- **Shared resource underneath the partitions**: separate semaphores but one connection pool or one DB. Isolate the actual bottleneck.
- **Unbounded waiting** in the overflow queue turns a bulkhead into a slow-motion outage. Bound the queue or fail fast.
- **No timeouts**: slots are held forever by hung calls. Pair with timeouts and a Circuit Breaker.
- **Sizing by guess.** Use measured latency and throughput (Little's law) and revisit after changes.
- **Not distinguishing rejections in metrics**: count bulkhead rejections separately from dependency errors.

## Refactoring path
- **Introduce it:** find the incident where one slow dependency or tenant took everything down. Put a timeout on that call, then a semaphore with a fail-fast limit, then a fallback. Expand to separate pools or deployments for workloads with different SLOs.
- **Remove it / simplify:** merge partitions that never fill or never fail; if the infrastructure already isolates (separate service, reserved concurrency), delete the in-code limiter.

## Related patterns
- [Circuit Breaker](../cloud/circuit-breaker.md): detects failure and stops calls; the bulkhead limits damage until it does.
- [Rate Limiter](../concurrency/rate-limiter.md): caps rate; bulkhead caps concurrency.
- [Thread Pool](../concurrency/thread-pool.md): separate pools per dependency are the classic bulkhead.
- [Lock Striping](../concurrency/lock-striping.md): partitions to reduce contention rather than to isolate failure.
- [Sharding](../cloud/sharding.md): partitioning data also isolates tenants' load and blast radius.
