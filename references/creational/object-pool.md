# Object Pool

**Category:** Creational · **Also known as:** Resource Pool, Connection Pool · **Related:** Thread Pool, Object Recycling, Flyweight, Singleton, Multiton, Bulkhead, Rate Limiter

## Intent
Keep a bounded set of expensive, reusable objects ready, lend them out on demand and take them back after use, instead of creating and destroying one per use.

## Problem
- Creating the object is expensive: TCP + TLS + auth handshake for a DB connection (often 5–50 ms), a browser instance, a GPU context, an SFTP session.
- The downstream system limits concurrent instances (PostgreSQL `max_connections`, a vendor's 10 concurrent sessions).
- Scenario: a FastAPI service opens a new `asyncpg` connection per request. At 300 rps, p99 latency is dominated by connection setup and Postgres hits `too many clients`. A pool of 20 connections, shared by all requests, fixes both latency and the hard limit.
- Smell: object creation shows up in the profiler, or the downstream reports connection storms.

## Solution and structure
- **Pool**: owns up to `max_size` objects; tracks idle and in-use; creates lazily up to the limit; blocks or fails when exhausted.
- **Reusable object**: must be *reset* to a clean state on return (rollback open transaction, clear session variables).
- **Client**: `acquire` → use → `release`, always via a context manager / `try/finally`.
- Policies: min/max size, acquire timeout, idle timeout, max lifetime (recycle), health check on borrow ("pre-ping").

```
client ─acquire()─► Pool ──idle?──► hand out existing
                     │   └─none & size<max─► create new
                     │   └─none & size=max─► wait (timeout) / raise
client ─release()──► reset + health-check ─► back to idle (or discard)
```

## When to use
- Creation cost is high relative to use, and objects are reusable after a reset.
- You must cap concurrency against a downstream (the pool size is also a throttle).
- Long-running service with steady load (web servers, workers).

## When NOT to use
- Objects are cheap (ordinary Python/JS objects, small buffers): pooling adds locking and bugs with no gain. Modern GCs handle short-lived objects well.
- A library already pools: SQLAlchemy engines, `asyncpg.create_pool`, `httpx.AsyncClient`, Node `http.Agent` with `keepAlive`, `pg.Pool`. Configure theirs; do not pool on top.
- Short-lived processes (Lambda invocations, CLI scripts) where the pool never warms. Use an external pooler (RDS Proxy, PgBouncer) instead.
- State cannot be reliably reset; a leak between users becomes a security bug.

## Example — Python
Usually you configure an existing pool. When you do need one (e.g. a vendor SDK session), an `asyncio.Queue` plus a context manager is enough.

```python
import asyncio
from collections.abc import AsyncIterator, Awaitable, Callable
from contextlib import asynccontextmanager
from typing import Protocol

class Session(Protocol):
    async def reset(self) -> None: ...
    async def close(self) -> None: ...
    def healthy(self) -> bool: ...

class SessionPool[S: Session]:
    def __init__(self, factory: Callable[[], Awaitable[S]], max_size: int = 10) -> None:
        self._factory = factory
        self._idle: asyncio.Queue[S] = asyncio.Queue()
        self._slots = asyncio.Semaphore(max_size)   # caps total live sessions

    @asynccontextmanager
    async def acquire(self, timeout: float = 5.0) -> AsyncIterator[S]:
        await asyncio.wait_for(self._slots.acquire(), timeout)
        try:
            session = self._idle.get_nowait() if not self._idle.empty() else await self._factory()
        except BaseException:
            self._slots.release()
            raise
        ok = False
        try:
            yield session
            ok = True
        finally:
            if ok and session.healthy():
                await session.reset()            # never hand dirty state to the next borrower
                self._idle.put_nowait(session)
            else:
                await session.close()            # discard broken or errored sessions
            self._slots.release()

# async with pool.acquire() as s: await s.query(...)
```

## Example — TypeScript
Node with `pg`: the library's `Pool` is the pattern. The important part is releasing in `finally`.

```typescript
import { Pool, PoolClient } from 'pg';

export const pool = new Pool({
  connectionString: process.env.DATABASE_URL,
  max: 20,                        // per process; multiply by replicas when sizing against max_connections
  idleTimeoutMillis: 30_000,
  connectionTimeoutMillis: 2_000, // fail fast when exhausted instead of queueing forever
});

export async function withTransaction<T>(fn: (c: PoolClient) => Promise<T>): Promise<T> {
  const client = await pool.connect();
  let broken = false;
  try {
    await client.query('BEGIN');
    const result = await fn(client);
    await client.query('COMMIT');
    return result;
  } catch (err) {
    try { await client.query('ROLLBACK'); } catch { broken = true; }
    throw err;
  } finally {
    client.release(broken);       // release(true) destroys the connection instead of reusing it
  }
}

// Single statements need no explicit checkout: pool.query('SELECT ...', [id])
```

## In the wild
- DB: SQLAlchemy `QueuePool` (`pool_size`, `max_overflow`, `pool_pre_ping`, `pool_recycle`), `asyncpg.create_pool`, psycopg `ConnectionPool`, Node `pg.Pool`, Prisma, HikariCP; external poolers PgBouncer and AWS RDS Proxy.
- HTTP: `httpx`/`requests.Session` and urllib3 connection pools, Node `http.Agent({ keepAlive: true })`, undici.
- Workers: `concurrent.futures.ThreadPoolExecutor`/`ProcessPoolExecutor`, Celery prefork, Puppeteer/Playwright browser pools (`generic-pool`).
- Redis clients (`redis.ConnectionPool`, ioredis).

## Trade-offs
| Pros | Cons |
|---|---|
| Removes repeated creation cost (latency, CPU) | Leaks (not released) exhaust the pool and hang the app |
| Caps concurrency against downstream limits | State leaking between borrowers if reset is incomplete |
| Smooths load spikes via waiting | Sizing is subtle (per process × replicas × workers) |
| Health checks/recycling hide transient failures | Stale objects after network blips unless pre-ping/recycle |

## Pitfalls
- **Not releasing on error paths.** Fix: only expose acquire as a context manager / `try/finally`.
- **Pool size × replicas × worker processes exceeds the DB limit.** Fix: compute total = pool_max × processes × pods; use PgBouncer/RDS Proxy for many small clients.
- **Oversized pools**: more connections than DB cores makes the DB slower. Start small (e.g. 2–4 × cores at the DB in total) and measure.
- **Sharing a pool across `fork()`** (gunicorn preload, multiprocessing). Fix: create pools after fork; SQLAlchemy `engine.dispose(close=False)` in the child.
- **Holding a connection during slow non-DB work** (HTTP call inside a transaction). Fix: acquire late, release early.
- **No acquire timeout**, so exhaustion becomes an outage with no error. Fix: set one and alert on it.
- **Asyncio: pool created on one event loop and used on another** (common in tests). Fix: create pools in `lifespan`/fixtures on the running loop.

## Refactoring path
- **Introduce it:** profile and confirm creation cost or connection limits. Prefer the library's pool; move creation to app startup (`lifespan`); wrap use in a context manager; add timeouts and metrics (in-use, waiters).
- **Remove it / simplify:** delete hand-rolled pools when a library offers one; in serverless, drop in-process pools for an external proxy; for cheap objects, just construct them.

## Related patterns
- [Thread Pool](../concurrency/thread-pool.md) — a pool of workers rather than of resources.
- [Object Recycling](../optimization/object-recycling.md) — reuse of memory/objects to cut allocation, usually without borrowing semantics.
- [Flyweight](../structural/flyweight.md) — shares immutable instances concurrently; a pool lends mutable ones exclusively.
- [Bulkhead](../cloud/bulkhead.md) — separate pools per dependency isolate failures.
- [Rate Limiter](../concurrency/rate-limiter.md) — limits rate over time; a pool limits concurrency.
- [Singleton](singleton.md) — the pool itself is usually one shared instance per process.
- [Multiton](multiton.md) — one pool per database/tenant.
