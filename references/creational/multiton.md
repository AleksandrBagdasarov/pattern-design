# Multiton

**Category:** Creational · **Also known as:** Registry of Singletons, Keyed Singleton · **Related:** Singleton, Lazy Initialization, Object Pool, Flyweight, Factory, Caching

## Intent
Guarantee exactly one instance *per key* (per tenant, region, database, logger name) and give one place to obtain it, creating it on first request for that key.

## Problem
- You need a shared, expensive instance, but not one in total: one S3 client per AWS region, one DB engine per tenant database, one rate limiter per API key.
- Creating a new instance per call wastes connections; one global instance is wrong because configuration differs per key.
- Scenario: a multi-tenant SaaS where each enterprise tenant has its own Postgres database. Handlers call `create_engine(tenant.dsn)` per request, which creates a new connection pool per request. Connections pile up until Postgres refuses new clients. One engine per tenant, created once and reused, solves it.
- Smell: a dict at module level mapping keys to clients, populated from several places with no locking or cleanup.

## Solution and structure
- **Multiton registry**: a map `key → instance`, plus a `get(key)` that returns the existing instance or creates one through a factory.
- **Factory**: builds an instance for a key (looks up per-key config).
- **Lifecycle policy**: are keys bounded (regions: a handful) or unbounded (tenants, users)? Unbounded keys need eviction and cleanup, which turns the Multiton into a cache.

```
get("eu-west-1") ─► registry ── hit ──► existing client
                         └─ miss ─► factory("eu-west-1") ─► store ─► client
```

## When to use
- The key space is small and known or slowly growing (regions, tenant DBs for tens to hundreds of tenants, named loggers).
- Each instance is expensive and safe to share among callers with the same key.
- Per-key configuration differs (credentials, endpoints, limits).

## When NOT to use
- Unbounded keys (per user, per request ID): the registry grows forever. Use a cache with TTL/LRU (`cachetools.TTLCache`, `lru_cache(maxsize=...)`) and close evicted resources.
- The instances are cheap: just create them.
- One instance with a key parameter on each call works (`client.get(region=...)`). Fewer objects, simpler.
- Only one key will ever exist. That is a [Singleton](singleton.md) (or just DI).
- You need cross-process uniqueness (one leader per tenant across pods): that requires a distributed lock or leader election, not an in-process map.

## Example — Python
`functools.cache` on a keyed factory is a Multiton. For resources that must be closed, keep an explicit registry with a shutdown method.

```python
import asyncio
from functools import cache
import boto3
from sqlalchemy.ext.asyncio import AsyncEngine, create_async_engine

@cache                                   # one client per region (keys are few and bounded)
def s3_client(region: str):
    return boto3.client("s3", region_name=region)   # boto3 clients are thread-safe

class TenantEngines:
    """One AsyncEngine (with its own pool) per tenant database."""
    def __init__(self, dsn_for: dict[str, str], pool_size: int = 5) -> None:
        self._dsn_for = dsn_for
        self._pool_size = pool_size
        self._engines: dict[str, AsyncEngine] = {}
        self._lock = asyncio.Lock()

    async def get(self, tenant_id: str) -> AsyncEngine:
        if (engine := self._engines.get(tenant_id)) is not None:
            return engine
        async with self._lock:                       # avoid two engines for one tenant
            if tenant_id not in self._engines:
                dsn = self._dsn_for[tenant_id]       # KeyError = unknown tenant, fail loudly
                self._engines[tenant_id] = create_async_engine(
                    dsn, pool_size=self._pool_size, pool_pre_ping=True)
            return self._engines[tenant_id]

    async def dispose_all(self) -> None:             # call from FastAPI lifespan shutdown
        for engine in self._engines.values():
            await engine.dispose()
        self._engines.clear()
```

## Example — TypeScript
A per-room WebSocket stream in Angular: each room key maps to one shared observable, removed when the last subscriber leaves.

```typescript
import { Injectable } from '@angular/core';
import { Observable, finalize, share } from 'rxjs';
import { webSocket } from 'rxjs/webSocket';

export interface ChatMessage { roomId: string; author: string; text: string; sentAt: string; }

@Injectable({ providedIn: 'root' })
export class RoomStreams {
  private readonly streams = new Map<string, Observable<ChatMessage>>();

  room(roomId: string): Observable<ChatMessage> {
    let stream = this.streams.get(roomId);
    if (!stream) {
      stream = webSocket<ChatMessage>(`wss://chat.example.com/rooms/${encodeURIComponent(roomId)}`).pipe(
        finalize(() => this.streams.delete(roomId)), // evict when the socket closes
        share(),                                     // one socket per room, many subscribers
      );
      this.streams.set(roomId, stream);
    }
    return stream;
  }
}
// Two components calling room('general') share one WebSocket connection.
```

## In the wild
- Python `logging.getLogger(name)`: the canonical Multiton (one logger per dotted name, never removed).
- `functools.cache`/`lru_cache` on keyed factories; `weakref.WeakValueDictionary` for registries that release unused instances.
- boto3: you build per-region clients yourself; botocore caches loaded service models internally.
- SQLAlchemy: one `Engine` per database URL is the documented practice (the engine owns the pool).
- Node: `ioredis` cluster per-node connections; `Intl.NumberFormat` instances cached per locale for performance.
- Angular: `inject()` with a token per key is not built in; keyed services are usually a root service holding a `Map`.

## Trade-offs
| Pros | Cons |
|---|---|
| One expensive instance per key, reused | Registry can grow without bound (memory/connection leak) |
| Per-key config and isolation (pools per tenant) | Global mutable state, same testing problems as Singleton |
| Lazy creation per key | Needs locking for concurrent first access |
| Central place for shutdown of all instances | Eviction requires closing resources safely while in use |

## Pitfalls
- **Unbounded growth** (keys from user input). Fix: bounded cache (LRU/TTL) that calls `close()`/`dispose()` on eviction, or validate keys against a known set.
- **Keying on mutable or unnormalized values** (`"EU-West-1"` vs `"eu-west-1"`). Fix: normalize keys; use enums.
- **Race on first access** creates two instances, leaking one pool. Fix: lock with a re-check, as above.
- **Using user-controlled keys without authorization**: tenant A asks for tenant B's engine. Fix: resolve the key from the authenticated principal, never from a raw header.
- **No shutdown path**, so dozens of pools stay open on redeploy. Fix: `dispose_all` in the app lifecycle.
- **`@cache` in tests** keeps clients between tests. Fix: `cache_clear()` in fixtures or inject the registry.

## Refactoring path
- **Introduce it:** find per-call creation of a keyed expensive resource. Wrap creation in one `get(key)` function; memoize it; add locking if concurrent; add shutdown; then decide on eviction if keys are unbounded.
- **Remove it / simplify:** if one key dominates, collapse to a Singleton/DI binding. If keys are unbounded and short-lived, replace the registry with a TTL cache or create per request with an external pooler.

## Related patterns
- [Singleton](singleton.md) — the single-key special case.
- [Lazy Initialization](lazy-initialization.md) — instances are created on the first request for each key.
- [Object Pool](object-pool.md) — a Multiton often holds one pool per key.
- [Flyweight](../structural/flyweight.md) — shares immutable instances by key to save memory; Multiton shares stateful resources.
- [Factory](factory.md) — the per-key creation logic.
- [Caching](../optimization/caching.md) — a Multiton with eviction is a cache; reuse cache tooling.
