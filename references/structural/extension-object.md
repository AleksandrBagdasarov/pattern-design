# Extension Object

**Category:** Structural · **Also known as:** Extension Interface, Capability query, Adapter registry · **Related:** Decorator, Adapter, Visitor, Role Object, Microkernel, Marker Interface, Component-Based Design

## Intent
Let an object expose additional interfaces at runtime without changing its class: clients ask the object "do you support capability X?" and get back an extension object implementing X, or nothing.

## Problem
- A core class (`Document`, `Order`, `User`) keeps growing interfaces as plugins and features need new behavior from it: `to_pdf()`, `to_xml()`, `index_for_search()`, `audit_snapshot()`…
- Different clients need different, unrelated views of the same object, and most clients need none of them.
- Plugins written by other teams must add capabilities to core objects without editing core code.

Scenario: an e-commerce `Order` must be exportable to an accounting system, searchable, and printable as an invoice — features owned by different teams and enabled per tenant. Adding all methods to `Order` bloats it and couples it to every team.

## Solution and structure
- **Subject**: the core object; offers `get_extension(kind) -> Extension | None`.
- **Extension (abstract)**: a capability interface (`InvoiceRenderable`, `Searchable`).
- **Concrete extension**: implements the capability for one subject type, usually holding a reference to the subject.
- **Registry** (common variant): maps `(subject type, capability) -> factory`, so plugins register extensions without touching the subject.

```
client ──get_extension(Searchable)──> Order
                                        │ looks up registry
                                        ▼
                         OrderSearchExtension(order) implements Searchable
```
Compared with Decorator: decorators wrap and keep the same interface; extension objects add *new* interfaces alongside the object.

## When to use
- Plugin architectures where third parties add capabilities to core entities.
- Many optional, unrelated interfaces for one class, each used by few clients.
- Capabilities that depend on configuration (enabled per tenant, per feature flag).

## When NOT to use
- The capabilities are known, few and stable: just implement them as methods or Protocols.
- Only one client needs the extra behavior: use a plain function `render_invoice(order)` in that client's module.
- In Python, `functools.singledispatch` gives you "add an operation for a type from outside" with far less machinery; prefer it when you need an operation, not an object with state.
- When callers would need many `if ext is None` checks: the design is probably hiding required behavior.

## Example — Python
A registry-based variant; plugins register extensions for core types.

```python
from collections.abc import Callable
from dataclasses import dataclass
from decimal import Decimal
from typing import Any, Protocol, cast


class Searchable(Protocol):
    def search_document(self) -> dict[str, Any]: ...

class InvoiceRenderable(Protocol):
    def render_invoice(self) -> str: ...


_REGISTRY: dict[tuple[type, type], Callable[[Any], object]] = {}

def register_extension(subject: type, capability: type):
    def deco[F: Callable[[Any], object]](factory: F) -> F:
        _REGISTRY[(subject, capability)] = factory
        return factory
    return deco


class Extensible:
    def get_extension[T](self, capability: type[T]) -> T | None:
        for cls in type(self).__mro__:                    # supports subclasses
            if (factory := _REGISTRY.get((cls, capability))) is not None:
                return cast(T, factory(self))
        return None


@dataclass
class Order(Extensible):
    id: int
    customer: str
    total: Decimal


# --- in plugin module search_plugin.py ---
@register_extension(Order, Searchable)
@dataclass
class OrderSearch:
    order: Order
    def search_document(self) -> dict[str, Any]:
        return {"id": self.order.id, "customer": self.order.customer.lower()}


# --- client code ---
order = Order(1, "ACME Ltd", Decimal("99.90"))
if (s := order.get_extension(Searchable)) is not None:
    print(s.search_document())
print(order.get_extension(InvoiceRenderable))            # None: plugin not loaded
```

## Example — TypeScript
Using `InjectionToken`s as capability keys, an Angular-flavored registry.

```ts
import { Injectable, InjectionToken } from '@angular/core';

export interface Order { id: number; customer: string; total: number }

export interface Exportable { toCsvRow(): string }
export interface Printable { toPrintHtml(): string }

type Factory<T> = (order: Order) => T;
export const EXPORTABLE = new InjectionToken<Factory<Exportable>>('Exportable');
export const PRINTABLE = new InjectionToken<Factory<Printable>>('Printable');

@Injectable({ providedIn: 'root' })
export class OrderExtensions {
  private readonly factories = new Map<InjectionToken<unknown>, Factory<unknown>>();

  register<T>(capability: InjectionToken<Factory<T>>, factory: Factory<T>): void {
    this.factories.set(capability, factory);
  }

  get<T>(order: Order, capability: InjectionToken<Factory<T>>): T | null {
    const f = this.factories.get(capability) as Factory<T> | undefined;
    return f ? f(order) : null;
  }
}

// feature module / provideAppInitializer registers what this tenant has enabled:
//   ext.register(EXPORTABLE, (o) => ({ toCsvRow: () => `${o.id},${o.customer},${o.total}` }));
//
// In a component:
//   const csv = this.ext.get(order, EXPORTABLE)?.toCsvRow();
//   @if (canPrint()) { <button (click)="print()">Print</button> }
```

## In the wild
- Eclipse `IAdaptable.getAdapter(Class)` — the canonical implementation, used throughout the Eclipse IDE.
- Zope/Pyramid `zope.interface` adapters and `queryAdapter(obj, IFace)`; Plone is built on it.
- COM `QueryInterface`; .NET `IServiceProvider.GetService(Type)` on objects.
- Python `functools.singledispatch` and `pluggy` hooks (pytest plugins) solve the same "add capabilities from outside" problem.
- VS Code extension API "contribution points" and Angular multi-providers (`multi: true`) are registry-based relatives.

## Trade-offs
| Pros | Cons |
|---|---|
| Core class stays small and stable | Capability discovery at runtime; missing extensions surface late |
| Plugins add interfaces without touching the core | Clients must handle "not supported" (`None`) |
| Capabilities can be enabled per configuration | Registry is global-ish state; import order matters |
| Separates unrelated concerns cleanly | Harder to navigate than plain methods; less static typing |

## Pitfalls
- **Plugin not imported, extension silently missing.** Fix: explicit plugin loading (entry points, `provideX()` functions) and a startup check of required capabilities.
- **Extensions mutate the subject's internals.** Fix: extensions use the subject's public API only.
- **Everything becomes an extension**, including core behavior. Fix: required behavior belongs on the class or a Protocol it implements.
- **Subclass lookup forgotten**: registered for `Order`, not found for `SubscriptionOrder`. Fix: walk the MRO / prototype chain, as shown.
- **Stateful extensions created on every call** lose state. Fix: cache per subject (`WeakKeyDictionary`) if they must keep state.

## Refactoring path
- **Introduce it:** when a core class keeps gaining methods owned by other teams/plugins, add `get_extension`, move one capability out as an extension registered by its owning module, then migrate the rest.
- **Remove it / simplify:** if all extensions are always present, move them back as methods or Protocol implementations; if extensions are stateless operations, replace with `singledispatch` functions.

## Related patterns
- [Decorator](../structural/decorator.md) — wraps to add behavior to the same interface; extensions add new interfaces.
- [Adapter](../structural/adapter.md) — an extension is often an adapter from the subject to the capability.
- [Visitor](../behavioral/visitor.md) — adds operations over a fixed type hierarchy; Extension Object adds interfaces per type.
- [Role Object](../behavioral/role-object.md) — context-specific roles of an entity; a close cousin.
- [Microkernel](../architectural/microkernel.md) — the architecture that typically hosts extension registries.
- [Component-Based Design](../structural/component-based-design.md) — entities as bags of components, queried by type.
