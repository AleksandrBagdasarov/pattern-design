# Dependency Injection

**Category:** Behavioral (injection) · **Also known as:** DI, Inversion of Control (IoC, the broader principle) · **Related:** Constructor Injection, Setter Injection, Interface Injection, Service Locator, Factory, Singleton, Strategy, Hexagonal

## Intent
Give an object the collaborators it needs from the outside instead of letting it create or look them up, so the wiring is decided in one place and the object can be reused, configured and tested with different implementations.

## Problem
- A class does `self.db = PostgresClient(os.environ["DB_URL"])` or `new HttpClient()` inside itself. It cannot be tested without a real database, cannot share a connection pool, and cannot switch to another implementation without editing it.
- Global singletons and module-level clients spread hidden dependencies: you can't tell from a signature what a function touches.
- Smells: `mock.patch("app.services.orders.requests.post")` in every test; import-time side effects (connecting to Redis on import); constructors reading env vars; "works on my machine" because tests share a global client.

Scenario: `OrderService` needs a repository, a payment gateway and a clock. In production these are Postgres, Stripe and the system clock; in tests they are an in-memory repo, a fake gateway and a fixed clock.

## Solution and structure
Three roles:
- **Consumer (client)**: declares what it needs, by type/interface, and does not construct it.
- **Dependency (service)**: an implementation of that interface.
- **Injector / composition root**: the single place, at the application's entry point, that builds the object graph and hands dependencies to consumers. This can be plain code ("pure DI") or a **container** that resolves the graph from registrations.

```
main.py / app.config.ts  (composition root)
   repo   = PostgresOrderRepo(pool)
   pay    = StripeGateway(api_key)
   svc    = OrderService(repo, pay, clock=SystemClock())   <- inject
   app    = create_app(svc)
```

Delivery mechanisms: **constructor** (default), **setter/property** (optional or late), **interface/method** (the consumer implements an injector method), plus **parameter injection** per call (FastAPI endpoint params, function arguments).

**Lifetimes (scopes)** decide how many instances exist:
| Scope | One instance per | Typical deps |
|---|---|---|
| Singleton / app | process (or root injector) | config, HTTP client, connection pool, cache |
| Scoped / request | request, job, unit of work, component subtree | DB session, current user, transaction |
| Transient | each injection | cheap stateless helpers |

Rule: a longer-lived object must not hold a shorter-lived one (a singleton capturing a request-scoped DB session is the "captive dependency" bug).

## When to use
- The dependency does I/O, is expensive, has state, or has more than one implementation (prod, test, per-tenant).
- You want unit tests without patching modules.
- Frameworks where DI is the native mechanism (FastAPI, Angular, NestJS, Spring): use it rather than fighting it.
- Ports-and-adapters/clean architectures: DI is how adapters get plugged into the core.

## When NOT to use
- Pure functions and value objects: just call them / construct them (`Decimal`, `datetime`, `Money`). Injecting `json` or a math helper is noise.
- Small scripts and CLIs: pass arguments to functions; no container.
- A container for a 10-class app: pure DI in a `main()` is clearer and type-checked. Add a container when the graph and scopes outgrow hand wiring.
- Injecting everything "for testability" when the tests would use the real implementation anyway (e.g. an in-process pure calculator).

## Example — Python
FastAPI `Depends` is a DI container keyed by callables: it resolves a graph per request, caches each dependency once per request by default, and runs `yield` dependencies' teardown after the response. App-lifetime objects come from `lifespan`.

```python
from collections.abc import AsyncIterator
from contextlib import asynccontextmanager
from typing import Annotated
import httpx
from fastapi import Depends, FastAPI, Request
from sqlalchemy.ext.asyncio import AsyncSession, async_sessionmaker, create_async_engine
from app.orders import OrderService, SqlOrderRepo          # plain classes, no FastAPI imports
from app.payments import PaymentGateway, StripeGateway

@asynccontextmanager
async def lifespan(app: FastAPI) -> AsyncIterator[None]:        # composition root: app scope
    engine = create_async_engine("postgresql+asyncpg://...")
    app.state.sessions = async_sessionmaker(engine, expire_on_commit=False)
    async with httpx.AsyncClient(base_url="https://api.stripe.com") as http:
        app.state.http = http
        yield
    await engine.dispose()

app = FastAPI(lifespan=lifespan)

async def get_session(request: Request) -> AsyncIterator[AsyncSession]:   # request scope
    async with request.app.state.sessions() as session:
        yield session

def get_gateway(request: Request) -> PaymentGateway:
    return StripeGateway(request.app.state.http)

def get_order_service(
    session: Annotated[AsyncSession, Depends(get_session)],
    gateway: Annotated[PaymentGateway, Depends(get_gateway)],
) -> OrderService:
    return OrderService(SqlOrderRepo(session), gateway)               # constructor injection

@app.post("/orders/{order_id}/pay")
async def pay(order_id: str, svc: Annotated[OrderService, Depends(get_order_service)]) -> dict:
    return await svc.pay(order_id)

# tests: app.dependency_overrides[get_gateway] = lambda: FakeGateway()
```

`OrderService` itself is plain Python with constructor parameters; it does not import FastAPI. Keep `Depends` at the edge.

## Example — TypeScript
Angular has a hierarchical injector: environment injectors (root, route) and element injectors (component `providers`). `inject()` works only in an injection context (field initializers, constructor, factory functions, `runInInjectionContext`).

```typescript
import { ApplicationConfig, Component, Injectable, InjectionToken, inject, signal } from '@angular/core';
import { HttpClient, provideHttpClient, withInterceptors } from '@angular/common/http';

export interface Clock { now(): Date }
export const CLOCK = new InjectionToken<Clock>('CLOCK', {
  providedIn: 'root',
  factory: () => ({ now: () => new Date() }),
});

@Injectable({ providedIn: 'root' })                 // app-wide singleton, tree-shakable
export class OrdersApi {
  private readonly http = inject(HttpClient);
  pay(id: string) { return this.http.post<{ status: string }>(`/api/orders/${id}/pay`, {}); }
}

@Injectable()                                        // no providedIn: must be provided explicitly
export class CheckoutState {                         // one instance per component subtree
  private readonly api = inject(OrdersApi);
  private readonly clock = inject(CLOCK);
  readonly paidAt = signal<Date | null>(null);
  pay(id: string) { this.api.pay(id).subscribe(() => this.paidAt.set(this.clock.now())); }
}

@Component({
  selector: 'app-checkout',
  providers: [CheckoutState],                        // component scope: destroyed with the component
  template: `<button (click)="state.pay('o-1')">Pay</button>`,
})
export class Checkout { protected readonly state = inject(CheckoutState); }

export const appConfig: ApplicationConfig = {       // composition root
  providers: [provideHttpClient(withInterceptors([]))],
};
// tests: TestBed.configureTestingModule({ providers: [{ provide: CLOCK, useValue: fixedClock }] })
```

## In the wild
- Python: FastAPI `Depends` + `dependency_overrides`; `dependency-injector`, `punq`, `lagom`, `svcs`, `dishka` containers; pytest fixtures are DI for tests (scopes: function/module/session); Django has none built in (settings + import strings are closer to Service Locator).
- TypeScript: Angular DI (`inject()`, `providers`, `InjectionToken`, hierarchical injectors); NestJS (`@Injectable`, module providers, `Scope.REQUEST`); InversifyJS, tsyringe; React Context is DI by tree.
- Others: Spring, .NET `Microsoft.Extensions.DependencyInjection` (Singleton/Scoped/Transient), Guice, Dagger.

## Trade-offs
| Pros | Cons |
|---|---|
| Testable without patching: pass fakes | Indirection: "where does this come from?" |
| Wiring and lifetimes in one place | Containers add magic, runtime resolution errors |
| Implementations swappable by config/environment | Scope bugs (captive dependencies) are subtle |
| Explicit dependencies in signatures | Over-injection bloats constructors and interfaces |
| Enables ports-and-adapters architecture | Learning curve for framework-specific rules (injection context) |

## Pitfalls
- **Service Locator in disguise**: passing the container around and calling `container.get(X)` inside business code. Fix: resolve only at the composition root; consumers declare parameters.
- **Captive dependency**: singleton holds a request-scoped session/user. Fix: inject a factory (`async_sessionmaker`) or make the consumer request-scoped.
- **Framework leaking into the domain** (`Depends` defaults in service constructors, Angular decorators in pure domain code). Fix: framework DI only at the edge; core classes take plain parameters.
- **Constructor over-injection** (8+ deps). It signals too many responsibilities. Fix: split the class, or introduce a Facade.
- **Interfaces with one implementation everywhere.** Fix: inject concrete classes until a second implementation or a test fake actually needs the seam; Python `Protocol` makes adding one later cheap.
- **Angular: `inject()` outside an injection context** (inside `setTimeout`, `subscribe`, async methods) throws NG0203. Fix: inject in field initializers; capture `Injector` and use `runInInjectionContext` when needed.
- **Angular: `providedIn: 'root'` vs component `providers` confusion.** Providing a root service again in a component creates a second instance for that subtree. Fix: be deliberate about which injector owns state.
- **Import-time construction** of clients (`client = Redis()` at module top). Fix: create in lifespan/composition root.

## Refactoring path
- **Introduce it:** (1) find `new`/constructor calls and global lookups inside a class; (2) turn each into a constructor parameter with the current object as default (temporary); (3) build the object in the composition root (`main`, `lifespan`, `app.config.ts`); (4) remove the defaults; (5) replace `mock.patch` in tests with injected fakes; (6) introduce a Protocol/InjectionToken only where a second implementation exists.
- **Remove it / simplify:** if a container is used for a handful of objects, replace it with explicit wiring in `main()`. Stop injecting pure, deterministic helpers; import and call them.

## Related patterns
- [Constructor Injection](constructor-injection.md) — the default delivery mechanism.
- [Setter Injection](setter-injection.md) — for optional or reconfigurable dependencies.
- [Interface Injection](interface-injection.md) — consumer exposes an injector method; rare today.
- [Service Locator](../architectural/service-locator.md) — the pull-based alternative; hides dependencies, usually inferior.
- [Factory](../creational/factory.md) — inject a factory when the consumer must create instances at runtime.
- [Singleton](../creational/singleton.md) — prefer a DI singleton scope over a hand-rolled Singleton.
- [Hexagonal](../architectural/hexagonal.md) — DI is how adapters are plugged into ports.
