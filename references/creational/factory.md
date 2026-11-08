# Factory (Simple Factory and Factory Method)

**Category:** Creational · **Also known as:** Simple Factory, Factory Method, Virtual Constructor · **Related:** Abstract Factory, Builder, Strategy, Dependency Injection, Template Method, Secure Factory / Builder

## Intent
Move the decision "which concrete class do I construct, and how" out of the caller and into one place. The caller asks for something that satisfies an interface; the factory picks and builds the implementation.

## Problem
- `if provider == "stripe": ... elif provider == "adyen": ...` is copied into every place that needs a payment gateway.
- Adding a new implementation means editing every caller (shotgun surgery).
- Construction needs knowledge the caller should not have: credentials, base URLs, retry settings.
- Scenario: an order service chooses a payment gateway by the merchant's country. Three handlers and a background job each duplicate the branching and the constructor arguments. A fourth provider is coming next quarter.

## Solution and structure
Two related forms:

- **Simple Factory**: one function or static method takes a key (config, input) and returns an instance of the right class. Not a GoF pattern, but the one you will write most.
- **Factory Method** (GoF): a base class declares an abstract `create_x()` and uses it in its own logic; subclasses override it to choose the product. Creation varies by subclass, while the algorithm around it stays in the base.

```
Simple Factory:   caller ──key──► make_gateway(key) ──► StripeGateway | AdyenGateway
                                                        (both satisfy PaymentGateway)

Factory Method:   ReportExporter.export()  calls self.create_writer()
                     ├─ CsvExporter.create_writer()  -> CsvWriter
                     └─ XlsxExporter.create_writer() -> XlsxWriter
```

Participants: **Product** (interface), **Concrete products**, **Creator** (factory function or the class with the factory method), **Client** (depends only on Product).

## When to use
- The concrete type is chosen at runtime from config, input or environment.
- Construction is non-trivial and must be the same everywhere (credentials, wiring, defaults).
- You expect more implementations and want one place to register them.
- A framework base class needs subclasses to supply a collaborator (Factory Method).

## When NOT to use
- There is one implementation and no concrete second one coming. Call the constructor.
- The caller already knows the exact type. `StripeGateway(cfg)` is clearer than `factory.create("stripe")`.
- You are only passing dependencies around. A DI container already is a factory; register the type there.
- Factory Method via inheritance when a callable parameter would do: in Python and TS, pass `writer_factory: Callable[[], Writer]` instead of subclassing.

## Example — Python
A registry-backed Simple Factory. In Python, classes and functions are already factories (callables), so a dict of callables replaces a class hierarchy of creators.

```python
from dataclasses import dataclass
from decimal import Decimal
from typing import Callable, Protocol

class PaymentGateway(Protocol):
    def charge(self, order_id: str, amount: Decimal, currency: str) -> str: ...

@dataclass(frozen=True)
class GatewayConfig:
    api_key: str
    sandbox: bool = False

class StripeGateway:
    def __init__(self, cfg: GatewayConfig) -> None: self.cfg = cfg
    def charge(self, order_id: str, amount: Decimal, currency: str) -> str:
        return f"stripe:{order_id}"

class AdyenGateway:
    def __init__(self, cfg: GatewayConfig) -> None: self.cfg = cfg
    def charge(self, order_id: str, amount: Decimal, currency: str) -> str:
        return f"adyen:{order_id}"

_REGISTRY: dict[str, Callable[[GatewayConfig], PaymentGateway]] = {
    "stripe": StripeGateway,
    "adyen": AdyenGateway,
}

def make_gateway(provider: str, cfg: GatewayConfig) -> PaymentGateway:
    try:
        return _REGISTRY[provider](cfg)
    except KeyError:
        raise ValueError(f"unknown payment provider: {provider!r}") from None

# Caller knows only the Protocol:
gateway = make_gateway("adyen", GatewayConfig(api_key="..."))
```

## Example — TypeScript
Angular's `useFactory` provider is a Factory resolved by DI, so the choice happens once at bootstrap.

```typescript
import { InjectionToken, Provider } from '@angular/core';

export interface StorageAdapter { get(key: string): string | null; set(key: string, v: string): void; }

class LocalStorageAdapter implements StorageAdapter {
  get(k: string) { return localStorage.getItem(k); }
  set(k: string, v: string) { localStorage.setItem(k, v); }
}
class MemoryStorageAdapter implements StorageAdapter {
  private readonly map = new Map<string, string>();
  get(k: string) { return this.map.get(k) ?? null; }
  set(k: string, v: string) { this.map.set(k, v); }
}

export const STORAGE = new InjectionToken<StorageAdapter>('STORAGE');

export function provideStorage(): Provider {
  return {
    provide: STORAGE,
    useFactory: (): StorageAdapter =>
      typeof localStorage !== 'undefined' ? new LocalStorageAdapter() : new MemoryStorageAdapter(), // SSR-safe
  };
}

// consumer: private readonly storage = inject(STORAGE);
```

## In the wild
- Python: `logging.getLogger`, `open()` (returns a text or binary file class depending on mode), `pathlib.Path()` (returns `PosixPath`/`WindowsPath`), `dataclasses.field(default_factory=...)`, SQLAlchemy `create_engine(url)` (picks dialect from the URL).
- FastAPI: a dependency function is a factory for request-scoped objects.
- Angular: `useFactory` providers, `InjectionToken({ factory })`.
- Node/AWS: `new S3Client(config)` resolving credentials providers; `crypto.createHash(alg)`.
- Factory Method: `unittest.TestCase` hooks, Django class-based views' `get_queryset()`, `get_form_class()`.

## Trade-offs
| Pros | Cons |
|---|---|
| Callers depend on the interface, not concrete classes | One more indirection to follow when reading code |
| One place to add an implementation (open/closed) | String keys lose type safety unless you use enums/literals |
| Construction details (secrets, defaults) stay centralized | Registry can become a dumping ground for unrelated types |
| Easy to swap in fakes for tests | Factory Method via inheritance couples product choice to class hierarchy |

## Pitfalls
- **The factory still has the `if/elif` ladder and grows forever.** Fix: a registry dict, or registration by plugins/entry points.
- **Stringly-typed keys** with typos discovered at runtime. Fix: `Literal["stripe", "adyen"]` or an `Enum`; TS union types.
- **Factory that takes ten parameters** to configure every product. That is a Builder problem; split it.
- **Returning concrete types from the factory signature**, so callers start using concrete methods. Fix: return the interface/Protocol type.
- **Factory called on every request for a stateless product.** Fix: build once and share (DI singleton scope).
- **Hiding errors**: unknown key returns `None`. Fix: raise with the list of valid keys.

## Refactoring path
- **Introduce it:** find duplicated `if type == ...: X(...)` blocks. Extract one function returning the common interface; replace each call site; then convert the branch into a registry; finally inject the factory (or its product) via DI.
- **Remove it / simplify:** if the registry has had one entry for a long time, inline the constructor at the call sites and delete the factory. If the factory merely wraps DI, register the type in the container directly.

## Related patterns
- [Abstract Factory](abstract-factory.md) — creates a *family* of matching products, not one.
- [Builder](builder.md) — for one complex product assembled in steps rather than chosen by type.
- [Strategy](../behavioral/strategy.md) — factories often select which strategy to use.
- [Template Method](../behavioral/template-method.md) — Factory Method is a Template Method whose step creates an object.
- [Dependency Injection](../behavioral/dependency-injection.md) — a container is a generalized factory; prefer it for wiring.
- [Secure Factory / Builder](../security/secure-factory-builder.md) — a factory that embeds credentials and security policy.
