# Service Locator

**Category:** Architectural · **Also known as:** registry lookup · **Related:** Dependency Injection, Singleton, Microkernel, Factory, Multiton

## Intent
Provide a central **registry** that code asks at runtime for the implementation of a service (`locator.get(PaymentGateway)`), instead of constructing it or receiving it from the caller.

## Problem
- Many classes need the same infrastructure (logger, config, cache), and passing them through every constructor seems tedious.
- Plug-ins loaded at runtime must find host services they cannot receive via constructors.
- Legacy code with `new ConcreteThing()` everywhere needs a seam for tests without a full DI rewrite.
- Scenario: a CLI tool with dynamically discovered plug-ins; each plug-in needs the HTTP client and config, but the host does not know the plug-in's constructor signature.

## Solution and structure
```
 composition root ──register──► ServiceLocator { type/key → instance or factory }
                                       ▲
 consumer.method() ───── get(Key) ─────┘
```
- **Locator/registry:** maps keys (types, tokens, strings) to instances or factories; optional scopes.
- **Registration:** done once at startup (composition root).
- **Consumer:** calls `get` when needed. Its dependencies are therefore **hidden** in method bodies, not visible in its signature.
- **DI vs Service Locator:** with DI, the container *pushes* dependencies into constructors; with a locator, the consumer *pulls*. The same container can be used either way; pulling from inside business code is the anti-pattern part.

## When to use
- Plug-in hosts and framework boundaries where constructors are not under your control (plug-ins, framework-instantiated classes).
- Inside a composition root or factory that resolves things by key at runtime.
- As a transitional seam in legacy code before introducing constructor injection.

## When NOT to use
- Normal application and domain code: use constructor/parameter injection; dependencies become explicit and tests need no global setup.
- Python: a module-level function or FastAPI `Depends` usually replaces it. Angular/NestJS: use `inject()`/constructor DI.
- Smell: `locator.get(...)` calls scattered through business methods; tests that must reset a global registry between cases.

## Example — Python
```python
# A minimal, typed locator for a plug-in host. Business code should not import it.
from collections.abc import Callable
from typing import Any, TypeVar, cast
from importlib.metadata import entry_points
import httpx

T = TypeVar("T")

class ServiceLocator:
    def __init__(self) -> None:
        self._factories: dict[type, Callable[[], Any]] = {}
        self._instances: dict[type, Any] = {}

    def register(self, key: type[T], factory: Callable[[], T]) -> None:
        self._factories[key] = factory

    def get(self, key: type[T]) -> T:
        if key not in self._instances:
            try:
                self._instances[key] = self._factories[key]()   # lazy singleton per key
            except KeyError:
                raise LookupError(f"no service registered for {key.__name__}") from None
        return cast(T, self._instances[key])

# composition root of the CLI host
locator = ServiceLocator()
locator.register(httpx.Client, lambda: httpx.Client(timeout=5.0))

def load_plugins() -> None:
    for ep in entry_points(group="mycli.plugins"):   # third-party packages register here
        plugin_init = ep.load()
        plugin_init(locator)                         # plug-in pulls what it needs

# inside a plug-in package:
def init(locator: ServiceLocator) -> None:
    client = locator.get(httpx.Client)
    ...
```
Note the plug-in's `init` still receives the locator as a parameter; avoid a module-global `locator` imported everywhere.

## Example — TypeScript
```ts
// Angular's Injector is a container; calling injector.get() from arbitrary code is service-location.
import { Injectable, Injector, inject, Type, InjectionToken } from '@angular/core';

export interface ExportFormat { id: string; export(rows: unknown[]): Blob }
// keyed by explicit ids: class names are minified in production builds
export const EXPORT_FORMATS = new InjectionToken<Record<string, Type<ExportFormat>>>('EXPORT_FORMATS');

// Legitimate use: a factory that resolves an implementation by runtime key.
@Injectable({ providedIn: 'root' })
export class ExporterRegistry {
  private injector = inject(Injector);
  private formats = inject(EXPORT_FORMATS);
  get(id: string): ExportFormat {
    const type = this.formats[id];
    if (!type) throw new Error(`Unknown export format: ${id}`);
    return this.injector.get(type);                     // resolved lazily, by key
  }
}

// Anti-pattern: hidden dependency pulled from a global injector inside business code.
// let appInjector: Injector;  // set in main.ts
// export function priceWithTax(p: number) { return p * appInjector.get(TaxService).rate; }
// Prefer: class PriceCalc { private tax = inject(TaxService); ... }
```
`inject()` in a field initializer is DI (resolved at construction, visible at the top of the class), not a locator.

## In the wild
- Angular `Injector.get`, NestJS `ModuleRef.get`/`resolve`, .NET `IServiceProvider.GetService`, Spring `ApplicationContext.getBean`.
- Python `importlib.metadata.entry_points` (plug-in registry), `codecs.lookup`, `logging.getLogger` (a locator by name), Flask `current_app` / `g`.
- Android `getSystemService`, JNDI in Java EE.

## Trade-offs
| Pros | Cons |
|---|---|
| Decouples consumer from concrete classes | Dependencies hidden; signatures lie about what a class needs |
| Works where constructors are not controllable (plug-ins) | Runtime "not registered" errors instead of startup/type errors |
| Lazy resolution by runtime key | Global state complicates tests and parallel test runs |
| Easy to retrofit into legacy code | Encourages reaching for everything from anywhere |

## Pitfalls
- **Global locator imported everywhere:** pass the locator only into composition roots, factories and plug-in entry points.
- **Late failures:** validate registrations at startup (resolve every required key once).
- **Test leakage:** a test registers a fake and another test sees it. Create a fresh locator per test or provide scoped overrides.
- **Lifetimes:** mixing singleton and per-request objects in one registry leaks request state. Make scope explicit.

## Refactoring path
- **Introduce it:** only as a seam: replace `new Concrete()` with `locator.get(Interface)` in legacy code to enable fakes in tests, as an intermediate step.
- **Remove it / simplify:** for each `locator.get(X)` inside a class, add `X` as a constructor/parameter dependency, resolve it in the composition root, and delete the call; once no business code calls `get`, keep the container only at the root.

## Related patterns
- [Dependency Injection](../behavioral/dependency-injection.md) — the preferred alternative: dependencies are pushed and explicit.
- [Singleton](../creational/singleton.md) — a locator is often a singleton holding singletons; shares its testing problems.
- [Microkernel](../architectural/microkernel.md) — plug-in hosts are the main legitimate use.
- [Factory](../creational/factory.md) — a keyed factory is often the honest name for a "locator" used at the root.
- [Multiton](../creational/multiton.md) — one instance per key, the storage model of many locators.
