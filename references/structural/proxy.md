# Proxy

**Category:** Structural · **Also known as:** Surrogate, Stub (remote) · **Related:** Decorator, Adapter, Dynamic Proxy, Secure Proxy, Lazy Load, Caching, Ambassador

## Intent
Provide a stand-in object with the same interface as a real subject, to control access to it: delay its creation, reach it over a network, check permissions, or cache its results.

## Problem
- The real object is expensive to create (loads a 200 MB model, opens a connection pool) and many code paths never use it.
- The real object lives in another process or machine, but callers should use it like a local object.
- Some callers may use the object and others may not, and you do not want permission checks scattered across callers.
- Repeated identical calls to a slow, pure-ish service waste time and money.

Scenario: an `ImageService` thumbnails images from S3. A list page with 500 items should not download 500 originals; a virtual proxy loads only when `.pixels()` is first called.

## Solution and structure
- **Subject**: the interface both real object and proxy implement.
- **Real Subject**: does the actual work.
- **Proxy**: holds (or knows how to obtain) the real subject and controls access to it.

Common kinds:
| Kind | Controls | Example |
|---|---|---|
| Virtual (lazy) | when the real object is created | lazy ORM relationship, lazy route |
| Remote | where it lives | gRPC stub, RPC client |
| Protection | who may call | permission-checking wrapper |
| Caching / smart reference | how often it is called; ref counting | memoized client, `weakref.proxy` |

```
Client ──> Subject
              ▲
     ┌────────┴────────┐
   Proxy ──controls──> RealSubject
```
Proxy vs. Decorator: same structure. A Proxy is about *access* and is usually one layer the system inserts (often managing the subject's lifecycle); a Decorator adds *behavior* and the client stacks several.

## When to use
- Lazy initialization of heavy resources behind an interface callers already use.
- Hiding a network boundary behind a local-looking interface (with care, see Pitfalls).
- Centralising access control or quota checks for an object.
- Caching results of an expensive, deterministic call.

## When NOT to use
- Laziness that a simple `functools.cached_property` or a lazy `import()` already gives you.
- Remote proxies that make network calls look free: chatty, fine-grained remote calls kill performance; design a coarse-grained API (DTOs) instead.
- Authorization that depends on the business context: put it in the use case/service where the context is known, not in a generic proxy.
- When you need to add several behaviors: that is Decorator territory.

## Example — Python
A virtual + caching proxy for a heavy ML model client. Python's `__getattr__` could make it generic (see Dynamic Proxy); here it is explicit for type safety.

```python
import threading
from functools import lru_cache
from typing import Protocol


class FraudScorer(Protocol):
    def score(self, user_id: int, amount_cents: int) -> float: ...


class OnnxFraudScorer:                     # Real subject: slow to construct
    def __init__(self, model_path: str) -> None:
        print(f"loading {model_path} ...")  # e.g. onnxruntime.InferenceSession
        self._path = model_path

    def score(self, user_id: int, amount_cents: int) -> float:
        return min(1.0, amount_cents / 1_000_000)


class LazyFraudScorer:                     # Virtual + caching proxy
    def __init__(self, model_path: str) -> None:
        self._model_path = model_path
        self._real: FraudScorer | None = None
        self._lock = threading.Lock()
        self.score = lru_cache(maxsize=10_000)(self._score)

    def _subject(self) -> FraudScorer:
        if self._real is None:
            with self._lock:               # double-checked: create once under threads
                if self._real is None:
                    self._real = OnnxFraudScorer(self._model_path)
        return self._real

    def _score(self, user_id: int, amount_cents: int) -> float:
        return self._subject().score(user_id, amount_cents)


scorer: FraudScorer = LazyFraudScorer("/models/fraud.onnx")  # app starts fast
```

## Example — TypeScript
A protection proxy for an admin API service, plus Angular's built-in virtual proxies.

```ts
import { Injectable, inject } from '@angular/core';
import { HttpClient } from '@angular/common/http';
import { Observable, throwError } from 'rxjs';

export interface UserAdminApi {
  deactivate(userId: number): Observable<void>;
}

@Injectable({ providedIn: 'root' })
export class HttpUserAdminApi implements UserAdminApi {
  private readonly http = inject(HttpClient);
  deactivate(id: number) { return this.http.post<void>(`/api/admin/users/${id}/deactivate`, {}); }
}

@Injectable({ providedIn: 'root' })
export class AuthSession { roles = new Set<string>(); }

@Injectable({ providedIn: 'root' })
export class GuardedUserAdminApi implements UserAdminApi {   // Protection proxy
  private readonly real = inject(HttpUserAdminApi);
  private readonly session = inject(AuthSession);
  deactivate(id: number): Observable<void> {
    return this.session.roles.has('admin')
      ? this.real.deactivate(id)
      : throwError(() => new Error('Forbidden: admin role required'));
  }
}
// Client-side checks are UX only; the server must enforce the same rule.

// Virtual proxies Angular gives you for free:
//   { path: 'admin', loadComponent: () => import('./admin').then(m => m.Admin) }
//   @defer (on viewport) { <heavy-chart /> } @placeholder { <p>Loading…</p> }
```

## In the wild
- Python: `weakref.proxy`, `unittest.mock.Mock`/`MagicMock`, `xmlrpc.client.ServerProxy`, `multiprocessing.managers` proxies, `functools.cached_property`.
- SQLAlchemy lazy-loaded relationships and `AssociationProxy`; Django `SimpleLazyObject` (used for `request.user`).
- gRPC generated stubs, boto3 clients, OpenAPI-generated clients (remote proxies).
- JS `Proxy` (Vue 3 and MobX reactivity), Angular lazy routes and `@defer`, Webpack/Vite dynamic `import()`.
- Infrastructure: Nginx/Envoy reverse proxies, CloudFront, API Gateway — the same idea at network level.

## Trade-offs
| Pros | Cons |
|---|---|
| Transparent to clients (same interface) | Transparency can hide cost (network, lazy load latency) |
| Faster startup, lower memory with lazy loading | First call is slow and may fail late, far from config |
| One place for access control or caching | Thread-safety of lazy init must be handled |
| Can manage lifecycle of the real subject | Stale cache and invalidation problems |

## Pitfalls
- **Fallacies of distributed computing**: remote proxies make callers forget latency and partial failure. Fix: coarse-grained methods, explicit timeouts, typed errors, retries/circuit breaker around the proxy.
- **Race on lazy init** in threaded code. Fix: a lock (double-checked) or create eagerly; in asyncio, guard with `asyncio.Lock`.
- **Caching per-user data in a shared proxy**. Fix: include the principal in the cache key, or cache only public data.
- **Client-side protection proxies treated as security**. Fix: always enforce on the server; client checks are UX.
- **`lru_cache` on methods keeps `self` alive** (memory leak). Fix: cache per instance as shown, or use `cachetools` with explicit keys.

## Refactoring path
- **Introduce it:** find an expensive constructor or remote client used directly; extract a Subject interface; make the proxy implement it and create the real object on demand; swap the binding in DI; keep callers unchanged.
- **Remove it / simplify:** if the subject is always used right after startup, create it eagerly and delete the proxy; replace hand-written caching proxies with `functools.cache`, `cachetools` or HTTP caching.

## Related patterns
- [Decorator](../structural/decorator.md) — same shape, adds behavior and stacks; Proxy controls access.
- [Adapter](../structural/adapter.md) — changes the interface; Proxy keeps it.
- [Dynamic Proxy](../structural/dynamic-proxy.md) — proxies generated at runtime for any interface.
- [Secure Proxy](../security/secure-proxy.md) — protection proxy in depth.
- [Lazy Load](../optimization/lazy-load.md) — the data-access form of the virtual proxy.
- [Caching](../optimization/caching.md) — cache strategies a caching proxy should follow.
- [Ambassador](../emergent/ambassador.md) — a remote proxy moved out of process.
