# Caching (Cache-Aside)

**Category:** Optimization · **Also known as:** Cache-aside, lazy caching, memoization (in-process) · **Related:** Read-Through Cache, Write-Behind Cache, Lazy Load, Proxy, Decorator, CQRS

## Intent
Keep the results of expensive reads in a faster store, and have the application check that store first, load from the source on a miss, and populate the cache itself — trading bounded staleness and memory for latency and load.

## Problem
- Profiling (not guessing) shows the same expensive read dominating: a product-catalog query runs 2,000 times/min with identical parameters; an external FX-rate API is called per request and rate-limits you.
- p95 latency is driven by a downstream you cannot speed up.
- Database CPU is saturated by repeated reads, while data changes rarely.

Profile first: `py-spy top --pid …` or `cProfile` + `snakeviz` for Python, APM traces (OpenTelemetry, Datadog), `pg_stat_statements` for top queries, Chrome DevTools Network/Performance panels for duplicate HTTP calls in the frontend. Cache the measured hotspot, then measure hit ratio and latency again.

## Solution and structure
```
get(key):  app ──1. GET key──▶ cache ──hit──▶ return
                 └─miss─▶ 2. load from DB/API ─▶ 3. SET key (TTL) ─▶ return
update:    app ──1. write DB──▶ 2. DELETE key (invalidate) ──▶ next read repopulates
```
Decisions you must make explicitly:
- **Key design**: include every input that changes the result (tenant, locale, version, user role). Prefix with a schema version (`v3:product:42`) to invalidate everything on deploy.
- **Invalidation**: TTL (bounded staleness), explicit delete on write, event-driven invalidation (CDC/outbox events), or versioned keys. Prefer *delete* over *update* on writes to avoid racing writers putting stale values.
- **Stampede protection**: when a hot key expires, N concurrent requests miss and all hit the DB. Use single-flight (one loader per key, others await), a short distributed lock (`SET key:lock NX PX 5000`), probabilistic early refresh (XFetch), or stale-while-revalidate.
- **TTL jitter**: randomize TTL ±10–20% so keys populated together do not expire together.
- **Negative caching**: cache "not found" briefly to stop repeated misses for non-existent keys (and cache-penetration attacks).
- **Bounds**: max size + eviction policy (LRU/LFU) for in-process caches; `maxmemory-policy` in Redis.

## When to use
- Read-heavy data with tolerable staleness: catalogs, config, permissions, FX rates, rendered fragments.
- Expensive computations with repeated inputs (pure functions → memoization).
- Shielding rate-limited or slow third-party APIs.
- Frontend: deduplicating identical HTTP requests across components in one session.

## When NOT to use
- You have not measured: a cache can hide a missing index or an N+1 query. Fix those first (`EXPLAIN ANALYZE`, eager loading).
- Data must be strongly consistent (balances, inventory at checkout): read from the source of truth.
- Low hit ratio (highly unique keys): you pay cache round-trips for nothing.
- The cache becomes the only copy of data: that is a datastore, not a cache.
- Single-process simple memoization needs: `functools.cache` beats a Redis cluster.

## Example — Python
```python
import asyncio, json, random
from collections.abc import Awaitable, Callable
from redis.asyncio import Redis

class CacheAside:
    def __init__(self, redis: Redis, ttl_s: int = 300, prefix: str = "v1") -> None:
        self._r, self._ttl, self._prefix = redis, ttl_s, prefix
        self._inflight: dict[str, asyncio.Future] = {}        # in-process single-flight

    async def get(self, key: str, load: Callable[[], Awaitable[dict | None]]) -> dict | None:
        k = f"{self._prefix}:{key}"
        if (raw := await self._r.get(k)) is not None:
            return json.loads(raw) or None                     # "{}" = cached miss
        if (fut := self._inflight.get(k)) is not None:
            return await fut                                   # stampede: wait for the leader
        fut = asyncio.get_running_loop().create_future()
        self._inflight[k] = fut
        try:
            value = await load()
            ttl = self._ttl if value else 30                   # short negative caching
            jitter = int(ttl * random.uniform(0.9, 1.1))
            await self._r.set(k, json.dumps(value or {}), ex=jitter)
            fut.set_result(value)
            return value
        except Exception as e:
            fut.set_exception(e)
            raise
        finally:
            del self._inflight[k]

    async def invalidate(self, key: str) -> None:
        await self._r.delete(f"{self._prefix}:{key}")          # call AFTER the DB commit
```
For pure in-process functions use `functools.cache` / `lru_cache(maxsize=…)`; for async with TTL, `cachetools` + a lock, or `aiocache`. Single-flight above protects one process; across many pods, add a Redis `SET NX` lock or serve stale while one worker refreshes.

## Example — TypeScript
```ts
import { Injectable, inject } from '@angular/core';
import { HttpClient } from '@angular/common/http';
import { Observable, shareReplay, timer, tap } from 'rxjs';

export interface Product { id: number; name: string; priceCents: number }

@Injectable({ providedIn: 'root' })
export class ProductCache {
  private readonly http = inject(HttpClient);
  private readonly entries = new Map<number, Observable<Product>>();
  private readonly ttlMs = 60_000;

  get(id: number): Observable<Product> {
    let entry = this.entries.get(id);
    if (!entry) {
      const created = this.http.get<Product>(`/api/products/${id}`).pipe(
        tap({ error: () => this.entries.delete(id) }),     // never keep a failed load
        shareReplay({ bufferSize: 1, refCount: false }),    // one request for all concurrent subscribers
      );
      this.entries.set(id, created);
      timer(this.ttlMs).subscribe(() => {                   // TTL eviction (only of this exact entry)
        if (this.entries.get(id) === created) this.entries.delete(id);
      });
      entry = created;
    }
    return entry;
  }

  invalidate(id: number): void { this.entries.delete(id); }   // call after a successful PUT
}
```
In RxJS 7 `shareReplay` resets on error (it does not cache the error), and the map entry is also removed so a failed id does not linger. Do not use `finalize` for this: an HTTP observable completes after the first response, so `finalize` would evict every entry immediately. Libraries such as TanStack Query (Angular adapter) give keys, stale-while-revalidate, dedupe and invalidation out of the box.

## In the wild
- Python: `functools.cache`/`lru_cache`/`cached_property`, `cachetools` (TTLCache), `aiocache`, Django cache framework, `fastapi-cache2`, `dogpile.cache` (built-in stampede lock).
- Redis / Valkey / Memcached; AWS ElastiCache; CDN and HTTP caching (`Cache-Control`, `ETag`, `stale-while-revalidate`).
- Frontend: TanStack Query, Apollo cache, Angular `withHttpTransferCache` (SSR hydration), service workers.

## Trade-offs
| Pros | Cons |
|---|---|
| Large latency and load reductions on hot reads | Stale data; invalidation is genuinely hard |
| Simple, explicit, works with any store | Cache logic spread across call sites |
| Cache failure degrades to source reads | Stampedes on expiry of hot keys |
| Shields fragile/rate-limited downstreams | Extra infrastructure, memory and failure modes |

## Pitfalls
- **No invalidation strategy** (SKILL anti-pattern). Every cached value needs a TTL *and* a write-path rule.
- **Invalidate-before-commit race:** delete, then another reader repopulates the old value before your transaction commits. Invalidate after commit (or use outbox/CDC events); consider a delayed second delete.
- **Set-on-write races:** two writers set the cache in the opposite order of their DB commits. Delete on write; let reads repopulate.
- **Missing key dimensions:** user A sees user B's data because the key lacks tenant/user. Review keys in code review like SQL.
- **Unbounded in-process caches** (`@cache` on a method with unbounded args, or on instance methods keeping `self` alive). Use `lru_cache(maxsize=…)` and avoid caching bound methods.
- **Caching errors or `None`** forever. Negative-cache only briefly.

## Refactoring path
- **Introduce it:** profile → pick top read → add cache-aside with TTL + metrics (hit ratio, load time) → add write-path invalidation → add stampede protection if the key is hot → measure again.
- **Remove it / simplify:** if hit ratio is low or staleness bugs pile up, fix the underlying query (index, denormalized read model), then delete the cache. Prefer moving cache logic behind a Read-Through abstraction if call sites duplicate it.

## Related patterns
- [Read-Through Cache](../optimization/read-through-cache.md) — the cache, not the caller, loads on miss.
- [Write-Behind Cache](../optimization/write-behind-cache.md) — caches writes too, flushing asynchronously.
- [Lazy Load](../optimization/lazy-load.md) — defers loading; caching avoids reloading.
- [Proxy](../structural/proxy.md) / [Decorator](../structural/decorator.md) — common wrappers for adding caching transparently.
- [CQRS](../cloud/cqrs.md) — a precomputed read model is often a better answer than a cache.
