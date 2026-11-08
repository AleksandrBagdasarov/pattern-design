# Singleton

**Category:** Creational · **Also known as:** — · **Related:** Multiton, Lazy Initialization, Dependency Injection, Service Locator, Double-Checked Locking, Module

## Intent
Ensure a class has exactly one instance per process and give a well-known way to reach it. Use it as a last resort: a DI container scope usually gives the same "one shared instance" without the global access.

## Problem
- Several parts of the code need the *same* stateful resource: a connection pool, an HTTP client with keep-alive, a metrics registry, loaded configuration.
- Creating it twice is wrong (two pools exhaust DB connections) or wasteful (re-parsing a 2 MB config on every request).
- Scenario: a FastAPI service creates `httpx.AsyncClient()` inside each handler. Every request opens new TCP/TLS connections and latency doubles under load. The client should be created once and shared.
- Smell that pushes people to Singleton: "I need this object everywhere and don't want to pass it around". That smell is usually better fixed by DI than by a global.

## Solution and structure
- **Singleton**: holds the single instance and controls its creation (private constructor + static accessor in classic OO).
- **Clients**: obtain the instance through the accessor instead of constructing it.

```
client A ──┐
client B ──┼──► Singleton.instance() ──► the one instance (created on first call)
client C ──┘
```

Modern forms, in order of preference:
1. **DI container scope**: register the object as app-scoped / `providedIn: 'root'`; callers receive it by injection.
2. **Module-level instance** (Python) or exported `const` (TS): the module system guarantees one evaluation per process.
3. **Cached factory**: `@functools.cache` on a zero-argument function, created lazily.
4. Classic class with a hidden constructor: rarely needed in Python or TS.

## When to use
- The resource is genuinely process-wide and stateless from the caller's view (logger, metrics registry, config snapshot, a pooled client).
- Duplicates would be incorrect, not just slower (a single file lock, a single in-process event loop scheduler).
- No DI mechanism is available at that point (module import time, a CLI script, a library entry point).

## When NOT to use
- You only want to avoid passing a parameter. Pass it, or use DI (`Depends`, `inject()`).
- The object holds per-user or per-request state. That is a request-scoped dependency, not a global one.
- You need different instances in tests. A hard-coded global forces monkeypatching; constructor injection is simpler.
- You need one instance *per key* (per tenant, per region). Use [Multiton](multiton.md).
- Multi-process deployments (gunicorn workers, Node cluster, serverless): a Singleton is per process. If you need one per *system*, you need leader election or an external store, not a Singleton.

## Example — Python
A module is already a Singleton. The idiomatic form is a cached factory, which also lets FastAPI inject it and tests override it.

```python
# app/clients.py
from functools import cache
import httpx
from pydantic_settings import BaseSettings

class Settings(BaseSettings):
    payments_url: str = "https://payments.internal"
    timeout_s: float = 5.0

@cache                      # first call builds it; later calls return the same object
def get_settings() -> Settings:
    return Settings()

@cache
def get_payments_client() -> httpx.AsyncClient:
    s = get_settings()
    return httpx.AsyncClient(base_url=s.payments_url, timeout=s.timeout_s)

# app/main.py
from contextlib import asynccontextmanager
from typing import Annotated
from fastapi import Depends, FastAPI

@asynccontextmanager
async def lifespan(app: FastAPI):
    yield
    await get_payments_client().aclose()   # explicit shutdown of the shared resource

app = FastAPI(lifespan=lifespan)

@app.get("/orders/{order_id}/payment")
async def payment(order_id: str,
                  client: Annotated[httpx.AsyncClient, Depends(get_payments_client)]):
    r = await client.get(f"/payments/by-order/{order_id}")
    return r.json()

# tests: app.dependency_overrides[get_payments_client] = lambda: fake_client
```

## Example — TypeScript
In Angular, `providedIn: 'root'` *is* the Singleton, owned by the injector, so tests can replace it.

```typescript
import { Injectable, inject, signal } from '@angular/core';
import { HttpClient } from '@angular/common/http';
import { firstValueFrom } from 'rxjs';

export interface AppConfig { apiUrl: string; featureFlags: Record<string, boolean>; }

@Injectable({ providedIn: 'root' })        // one instance per root injector
export class ConfigService {
  private readonly http = inject(HttpClient);
  private readonly _config = signal<AppConfig | null>(null);
  readonly config = this._config.asReadonly();

  async load(): Promise<void> {
    this._config.set(await firstValueFrom(this.http.get<AppConfig>('/config.json')));
  }
  isEnabled(flag: string): boolean {
    return this._config()?.featureFlags[flag] ?? false;
  }
}

// Node: an exported instance is a Singleton per module graph.
// export const prisma = new PrismaClient();
// Test: TestBed.configureTestingModule({ providers: [{ provide: ConfigService, useValue: fake }] })
```

## In the wild
- Python: modules themselves; `logging.getLogger(name)` (a Multiton keyed by name); `functools.cache`; FastAPI app state / `lifespan`.
- Angular: `@Injectable({ providedIn: 'root' })`; providers in `bootstrapApplication`. Note that lazy-loaded route providers create a *child* injector and therefore a second instance.
- NestJS: providers are singleton-scoped by default (`Scope.DEFAULT`).
- Node: the module cache (`require`/ESM) evaluates a module once per path; two copies of a package in `node_modules` give two "singletons".

## Trade-offs
| Pros | Cons |
|---|---|
| One shared expensive resource (pool, client) | Hidden global state; dependencies are not visible in signatures |
| Simple access from anywhere | Tests leak state into each other unless reset |
| Controlled lifecycle when owned by DI | Per-process only; misleading under multi-worker deploys |
| Lazy creation is easy | Concurrency on first creation must be handled (threads) |

## Pitfalls
- **Calling the global directly from business code** (`Settings()` deep inside a service). Fix: resolve it at the edge and inject it.
- **Mutable singleton state shared across requests** (a "current user" field). Fix: keep singletons stateless or immutable; put per-request data in request scope.
- **Thread-unsafe lazy creation** in threaded Python or Java-style code. Fix: create eagerly at startup, or guard with a lock (`functools.cache` is not a lock; two threads may both build it once).
- **No shutdown**: clients and pools never closed, warnings at exit. Fix: tie close to `lifespan` / `ngOnDestroy` of the owner / `app.enableShutdownHooks()`.
- **Test pollution**: `@cache` keeps the instance across tests. Fix: `get_settings.cache_clear()` in a fixture, or use DI overrides.
- **Angular: `providers: [MyService]` on a component** silently creates a separate instance for that subtree.

## Refactoring path
- **Introduce it:** spot repeated construction of the same expensive object. Step 1: move creation into one factory function. Step 2: cache it (`@cache` / `providedIn: 'root'`). Step 3: inject it at call sites instead of importing the global. Step 4: add shutdown in the app lifecycle.
- **Remove it / simplify:** replace `Thing.instance()` calls with a constructor parameter, one call site at a time; register the object in the DI container with app scope; delete the static accessor when no callers remain.

## Related patterns
- [Multiton](multiton.md) — one instance per key instead of one in total.
- [Lazy Initialization](lazy-initialization.md) — how the single instance is often created on first use.
- [Dependency Injection](../behavioral/dependency-injection.md) — the preferred way to share one instance without global access.
- [Service Locator](../architectural/service-locator.md) — a registry of singletons; shares the hidden-dependency problem.
- [Double-Checked Locking](../concurrency/double-checked-locking.md) — thread-safe lazy creation in threaded runtimes.
- [Module](../structural/module.md) — in Python and TS, the module is the natural Singleton.
