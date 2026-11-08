# Wrapper

**Category:** Structural · **Also known as:** Wrapper family, Delegation · **Related:** Adapter, Decorator, Proxy, Facade, Dynamic Proxy, Secure Proxy, Ambassador

## Intent
"Wrapper" is not one pattern but an umbrella term for any object that holds another object and forwards calls to it. This page helps you pick the right member of the family: Adapter, Decorator, Proxy or Facade. They look identical in code (a class with a private field and forwarding methods); they differ in *why* they wrap and *what interface* they expose.

## Problem
- You need to put something between a caller and an object: to translate it, add behavior, control access or hide complexity.
- Code reviews argue "is this a Proxy or a Decorator?" while the real question is unanswered: should the interface change, and who decides what gets wrapped?
- Misnamed wrappers mislead readers: a `StripeProxy` that actually translates types, or a `CacheAdapter` that adds caching, hides the design intent.

## Solution and structure
Ask three questions:

1. **Is the exposed interface the same as the wrapped one?**
   - Different, and you must fit an *existing* expected interface → **Adapter**.
   - Different, and *new and simpler* than a whole subsystem → **Facade**.
   - Same → go to 2.
2. **Is the point to add behavior or to control access?**
   - Add responsibilities (logging, metrics, retry, compression), stackable, chosen by the client → **Decorator**.
   - Control access (lazy creation, remote call, permission check, caching of an expensive object), usually chosen by the system, not the client → **Proxy**.
3. **Must it work for any interface without writing forwarding methods?** → **Dynamic Proxy** (`__getattr__`, JS `Proxy`).

| | Adapter | Decorator | Proxy | Facade |
|---|---|---|---|---|
| Interface exposed | the one the client expects (differs from wrappee) | same as wrappee | same as wrappee | new, simplified |
| Wraps | one object | one object (stackable) | one object | many objects / a subsystem |
| Purpose | translate | add behavior | control access | simplify |
| Who composes it | integration/boundary code | client or DI wiring, often several layers | system/framework, usually one layer | subsystem author |
| Creates wrappee? | no, receives it | no, receives it | often yes (lazy/virtual proxy) | often yes |
| Typical name | `StripeGatewayAdapter` | `RetryingHttpClient` | `LazyImage`, `RemoteUserService` | `CheckoutFacade`, `OrdersApi` |
| Python idiom | Protocol + class | function decorator or class | `__getattr__`, `functools.cached_property` | module with a few functions |
| TS/Angular idiom | service mapping DTO→model | `HttpInterceptor`, RxJS operators | JS `Proxy`, `@defer`, lazy routes | store facade service |

## When to use
- Use this page when you know you need indirection and must choose its kind.
- Use the specific pattern's page for implementation details.

## When NOT to use
- No wrapper at all is often right: if you own the class, change it.
- A function is enough when the wrapper has one method: `with_retry(fn)` beats a `RetryingClient` class with one forwarding method.
- Avoid wrapping "just in case" around stable libraries; wrap at real boundaries (vendor SDKs, network, persistence).

## Example — Python
The same `HttpClient` wrapped four different ways. Note how only the interface and the purpose differ.

```python
import time
from typing import Any, Protocol

import httpx


class JsonClient(Protocol):
    def get_json(self, path: str) -> dict[str, Any]: ...


class HttpxAdapter:                    # Adapter: httpx.Client -> JsonClient
    def __init__(self, client: httpx.Client) -> None:
        self._c = client
    def get_json(self, path: str) -> dict[str, Any]:
        r = self._c.get(path)
        r.raise_for_status()
        return r.json()


class TimedClient:                     # Decorator: same interface, adds timing
    def __init__(self, inner: JsonClient) -> None:
        self._inner = inner
    def get_json(self, path: str) -> dict[str, Any]:
        t = time.perf_counter()
        try:
            return self._inner.get_json(path)
        finally:
            print(f"GET {path} {1000 * (time.perf_counter() - t):.1f} ms")


class LazyClient:                      # Proxy: same interface, controls creation
    def __init__(self, base_url: str) -> None:
        self._base_url, self._real = base_url, None
    def get_json(self, path: str) -> dict[str, Any]:
        if self._real is None:
            self._real = HttpxAdapter(httpx.Client(base_url=self._base_url))
        return self._real.get_json(path)


class UsersFacade:                     # Facade: new, simpler interface over several calls
    def __init__(self, api: JsonClient) -> None:
        self._api = api
    def profile_with_orders(self, user_id: int) -> dict[str, Any]:
        user = self._api.get_json(f"/users/{user_id}")
        user["orders"] = self._api.get_json(f"/users/{user_id}/orders")["items"]
        return user


users = UsersFacade(TimedClient(LazyClient("https://api.example.com")))
```

## Example — TypeScript
```ts
import { Injectable, inject } from '@angular/core';
import { HttpClient, HttpInterceptorFn } from '@angular/common/http';
import { Observable, forkJoin, map, tap } from 'rxjs';

// Decorator: functional interceptor adds behavior to every request, same contract.
export const timingInterceptor: HttpInterceptorFn = (req, next) => {
  const t = performance.now();
  return next(req).pipe(tap({ complete: () => console.log(req.url, performance.now() - t) }));
};

// Adapter: backend DTO -> frontend model.
interface UserDto { user_id: number; full_name: string }
export interface User { id: number; name: string }
const toUser = (d: UserDto): User => ({ id: d.user_id, name: d.full_name });

// Facade: one simple method over several endpoints.
@Injectable({ providedIn: 'root' })
export class ProfileFacade {
  private readonly http = inject(HttpClient);
  load(id: number): Observable<{ user: User; orderCount: number }> {
    return forkJoin({
      user: this.http.get<UserDto>(`/api/users/${id}`).pipe(map(toUser)),
      orders: this.http.get<unknown[]>(`/api/users/${id}/orders`),
    }).pipe(map(({ user, orders }) => ({ user, orderCount: orders.length })));
  }
}

// Proxy: lazy route; the real component chunk is loaded on first access.
export const routes = [
  { path: 'reports', loadComponent: () => import('./reports').then((m) => m.Reports) },
];
```

## In the wild
- Adapter: `io.TextIOWrapper`, `util.promisify`, RxJS `from()`, Angular `ControlValueAccessor`.
- Decorator: `functools.lru_cache`, `tenacity.retry`, ASGI/Express middleware, Angular `HttpInterceptorFn`, RxJS `pipe()` operators.
- Proxy: `weakref.proxy`, `unittest.mock.Mock`, SQLAlchemy lazy-loaded relationships, JS `Proxy` (Vue 3 reactivity), gRPC/HTTP client stubs, Angular `@defer` blocks.
- Facade: `requests.get` over `urllib3`, boto3 resources, NgRx/signal-store facade services.

## Trade-offs
| Pros | Cons |
|---|---|
| Separates concerns without editing the wrapped class | Every layer adds indirection and stack-trace depth |
| Correct naming communicates intent instantly | Identical code shape makes misuse and misnaming easy |
| Wrappers compose (decorate an adapter, facade over decorators) | Deep stacks hide where behavior lives and in what order |
| Natural test seams | Identity checks (`is`, `===`, `isinstance`) break through wrappers |

## Pitfalls
- **Naming by structure instead of intent.** Fix: name by role (`Adapter`, `Retrying…`, `Lazy…`, `…Facade`).
- **One wrapper doing three jobs** (translate + retry + cache). Fix: split into an adapter plus decorators.
- **Order bugs in stacks**: cache outside retry vs. inside retry produce different behavior. Fix: compose in one place (DI wiring) and test the order.
- **Forgotten methods**: a hand-written wrapper forwards 5 of 6 methods. Fix: narrow the Protocol, or use a dynamic proxy deliberately.

## Refactoring path
- **Introduce it:** first decide whether the interface must change (Adapter/Facade) or not (Decorator/Proxy), then follow that pattern's page.
- **Remove it / simplify:** collapse single-method wrappers into functions; remove pass-through wrappers that add nothing; flatten stacks where two layers do the same thing.

## Related patterns
- [Adapter](../structural/adapter.md) — translate to an expected interface.
- [Decorator](../structural/decorator.md) — add stackable behavior, same interface.
- [Proxy](../structural/proxy.md) — control access, same interface.
- [Facade](../structural/facade.md) — simple entry point to a subsystem.
- [Dynamic Proxy](../structural/dynamic-proxy.md) — generic wrappers generated at runtime.
- [Secure Proxy](../security/secure-proxy.md) — Proxy specialised for authorization.
- [Ambassador](../emergent/ambassador.md) — the out-of-process wrapper for remote calls.
