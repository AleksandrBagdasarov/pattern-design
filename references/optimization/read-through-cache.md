# Read-Through Cache

**Category:** Optimization · **Also known as:** Loading cache, self-populating cache · **Related:** Caching (cache-aside), Write-Behind Cache, Proxy, Decorator, Repository

## Intent
Make the cache the only thing callers talk to: on a miss, the cache itself invokes a configured loader, stores the result and returns it. Loading, TTL, invalidation and stampede protection live in one component instead of every call site.

## Problem
- Cache-aside code (`get → if None: load → set`) is duplicated across 30 call sites, each with slightly different TTLs, key formats and error handling.
- Some call sites forgot stampede protection or negative caching; one caches exceptions.
- You want to swap Redis for an in-process cache, or add metrics, without touching callers.

Profile first: confirm the reads are hot and repetitive (APM traces, `pg_stat_statements`, `py-spy`, DevTools Network panel showing duplicate requests). Read-through is a structural improvement over cache-aside once caching is justified.

## Solution and structure
```
caller ──get(key)──▶ ReadThroughCache ──hit──▶ value
                          │ miss (single-flight per key)
                          ▼
                       loader(key) ──▶ DB / API
                          │
                          └──store with TTL+jitter──▶ backing cache (memory / Redis)
```
- **Cache**: owns the backing store, TTL, eviction, single-flight per key, negative caching and metrics.
- **Loader**: a function `key → value` bound at construction. The caller never calls it.
- **Invalidation API**: `invalidate(key)` / `invalidate_all()` called by the write path or by change events.
- Optional **refresh-ahead**: reload hot entries in the background before they expire (Caffeine `refreshAfterWrite`), so readers rarely wait.

Versus cache-aside: same runtime behavior, but the responsibility moves into the cache. The cache becomes a Proxy/Decorator in front of the repository or client.

## When to use
- Many callers read the same kind of data by key (products by id, user permissions, tenant config).
- You want consistent caching policy and observability in one place.
- The loader is uniform (one repository/client method per key type).

## When NOT to use
- One or two call sites: plain cache-aside is clearer.
- Loading needs call-site context the cache cannot have (request-specific auth headers, varying query options) — or include those in the key and accept lower hit rates.
- Bulk/query-shaped reads ("orders where status=open and …"): caching arbitrary query results invites invalidation nightmares; use a read model (CQRS) or DB indexes.
- Strong consistency requirements.

## Example — Python
```python
import asyncio, random, time
from collections import OrderedDict
from collections.abc import Awaitable, Callable, Hashable

class ReadThroughCache[K: Hashable, V]:
    def __init__(self, loader: Callable[[K], Awaitable[V | None]],
                 ttl_s: float = 300, max_size: int = 10_000, negative_ttl_s: float = 15) -> None:
        self._load, self._ttl, self._max, self._neg_ttl = loader, ttl_s, max_size, negative_ttl_s
        self._data: OrderedDict[K, tuple[float, V | None]] = OrderedDict()
        self._inflight: dict[K, asyncio.Task[V | None]] = {}
        self.hits = self.misses = 0

    async def get(self, key: K) -> V | None:
        if (entry := self._data.get(key)) and entry[0] > time.monotonic():
            self._data.move_to_end(key)                         # LRU touch
            self.hits += 1
            return entry[1]
        self.misses += 1
        task = self._inflight.get(key)
        if task is None:                                        # single-flight per key
            task = asyncio.create_task(self._fill(key))
            self._inflight[key] = task
            task.add_done_callback(lambda _: self._inflight.pop(key, None))
        return await asyncio.shield(task)                       # a cancelled caller doesn't cancel the load

    async def _fill(self, key: K) -> V | None:
        value = await self._load(key)                           # exceptions propagate; nothing cached
        ttl = (self._ttl if value is not None else self._neg_ttl) * random.uniform(0.9, 1.1)
        self._data[key] = (time.monotonic() + ttl, value)
        self._data.move_to_end(key)
        while len(self._data) > self._max:
            self._data.popitem(last=False)
        return value

    def invalidate(self, key: K) -> None:
        self._data.pop(key, None)

# usage: products = ReadThroughCache(repo.get_product, ttl_s=120); await products.get(42)
```
PEP 695 generic class syntax requires Python 3.12+. For production in-process caches consider `cachetools` (+ `asyncache`) or `aiocache`; for distributed caches, keep this interface and back it with Redis plus a lock for cross-process single-flight.

## Example — TypeScript
```ts
import { Injectable, inject, InjectionToken } from '@angular/core';
import { Observable, shareReplay, defer, timer, takeUntil } from 'rxjs';
import { HttpClient } from '@angular/common/http';

export class ReadThroughCache<K, V> {
  private readonly entries = new Map<K, { expires: number; value$: Observable<V> }>();
  constructor(private readonly loader: (key: K) => Observable<V>, private readonly ttlMs = 60_000) {}

  get(key: K): Observable<V> {
    const hit = this.entries.get(key);
    if (hit && hit.expires > Date.now()) return hit.value$;
    const value$ = defer(() => this.loader(key)).pipe(
      shareReplay({ bufferSize: 1, refCount: false }),       // one request shared by all subscribers
    );
    this.entries.set(key, { expires: Date.now() + this.ttlMs, value$ });
    value$.subscribe({ error: () => this.entries.delete(key) });   // don't keep failed loads
    return value$;
  }
  invalidate(key: K): void { this.entries.delete(key); }
}

export interface Permissions { roles: string[] }

@Injectable({ providedIn: 'root' })
export class PermissionsService {
  private readonly http = inject(HttpClient);
  private readonly cache = new ReadThroughCache<string, Permissions>(
    (tenantId) => this.http.get<Permissions>(`/api/tenants/${tenantId}/permissions`), 5 * 60_000);

  forTenant(tenantId: string): Observable<Permissions> { return this.cache.get(tenantId); }
  onPermissionsChanged(tenantId: string): void { this.cache.invalidate(tenantId); }
}
```
Expired entries in this sketch are only replaced on next access; add a size bound if keys are unbounded.

## In the wild
- Java Caffeine `LoadingCache` / Guava `CacheLoader` (the canonical read-through API, with refresh-ahead), Ehcache, Hazelcast `MapLoader`.
- AWS DynamoDB Accelerator (DAX) is a read-through/write-through cache in front of DynamoDB; CDNs are read-through caches for HTTP origins.
- Python: `dogpile.cache` regions with `get_or_create` and creator functions; `functools.lru_cache` is a read-through memoizer for pure functions.
- Frontend: TanStack Query (`queryFn` is the loader), Apollo Client cache policies.

## Trade-offs
| Pros | Cons |
|---|---|
| Caching policy centralized; callers stay simple | Hides I/O behind what looks like a cheap lookup |
| Stampede protection and metrics implemented once | Loader must be expressible as `key → value` |
| Easy to swap backing stores | Cache becomes a critical dependency in the read path |
| Enables refresh-ahead for hot keys | Generic layer can mask per-use staleness requirements |

## Pitfalls
- **Caching exceptions** or permanently caching `None`: drop failed entries; give negative entries a short TTL.
- **Cancellation propagating into the shared load** (asyncio): one caller timing out cancels everyone's load. Shield the shared task as above.
- **Invalidation not wired:** the write path must call `invalidate` after commit, or publish an event the cache subscribes to.
- **Unbounded key space** (search strings as keys): set `max_size` and evict LRU.
- **Per-instance caches across many pods** diverge after invalidation on one pod. Use a shared cache or broadcast invalidations (Redis pub/sub).

## Refactoring path
- **Introduce it:** find duplicated cache-aside blocks; create one read-through cache per entity with the loader from the repository; replace call sites; centralize invalidation in the write service.
- **Remove it / simplify:** if hit ratio is low, delete the cache and call the repository directly; if only one caller remains, inline back to cache-aside or `functools.lru_cache`.

## Related patterns
- [Caching](../optimization/caching.md) — cache-aside: the same idea with loading done by the caller.
- [Write-Behind Cache](../optimization/write-behind-cache.md) — the write-side counterpart; together they make the cache the system of access.
- [Proxy](../structural/proxy.md) / [Decorator](../structural/decorator.md) — a read-through cache wrapping a repository is a caching proxy.
- [Repository](../data-access/repository.md) — the usual loader source and the interface the cache can implement.
