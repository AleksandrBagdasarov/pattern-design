# Module

**Category:** Structural · **Also known as:** Module pattern, Revealing Module, Package-by-feature, Modular monolith (at system scale) · **Related:** Facade, Singleton, Layered, Microservices, Microkernel, Private Class Data

## Intent
Group related code and state behind a small public surface, keeping internals private, so the rest of the system depends only on what the module deliberately exports.

## Problem
- Everything imports everything: `from app.orders.repository import _build_query` in a reporting script; refactoring any file breaks unrelated code.
- No visible boundary between "API of this feature" and "implementation details".
- Code is organised by technical layer (`models/`, `services/`, `utils/`) so one feature change touches six folders.
- Global state and helper functions leak into the global namespace (historically, browser JS before ES modules).

Scenario: a FastAPI monolith has `billing`, `orders`, `catalog`. `orders` reaches into `billing.models.Invoice` and its SQLAlchemy session directly. Splitting billing into a service later is impossible without untangling dozens of imports.

## Solution and structure
- **Public surface**: the names a module exports (`__init__.py` + `__all__`; ES `export`; Angular public API of a library).
- **Internals**: everything else, private by convention (`_name`, `_internal/` package) or by language/tooling (unexported ES symbols, lint rules).
- **Dependencies**: a module declares what it needs (imports, DI); dependency direction between modules is explicit and acyclic.

```
orders/                       billing/
  __init__.py  (public API) ──> __init__.py (public API)
  _service.py                   _invoices.py
  _repository.py                _stripe_adapter.py
  _models.py                    _models.py
```
Modern languages make the old "module pattern" (IIFE closures in JS) unnecessary: Python modules and ES modules *are* modules. The pattern today is about **designing boundaries**, not emulating privacy.

## When to use
- Always, at some granularity: every non-trivial codebase needs boundaries.
- Package by feature/bounded context when features evolve independently.
- Before considering microservices: a modular monolith with enforced boundaries gives most of the benefit at a fraction of the cost.
- In Angular: feature folders with a single public entry (`index.ts`), lazy-loaded routes per feature, and libraries with an explicit public API.

## When NOT to use
- Tiny scripts and prototypes: one file is fine.
- Splitting into many micro-modules with one function each: navigation cost outweighs encapsulation.
- Barrel files (`index.ts` re-exporting everything) that export internals defeat the purpose and can hurt tree-shaking and cause circular imports.

## Example — Python
A feature package exposing a small API; internals are underscore-prefixed. A Python module is also a natural Singleton: it is imported once per process.

```python
# orders/__init__.py  — the module's public surface
from orders._service import OrderService, place_order
from orders._schemas import OrderIn, OrderOut

__all__ = ["OrderService", "place_order", "OrderIn", "OrderOut"]


# orders/_schemas.py
from decimal import Decimal
from pydantic import BaseModel

class OrderIn(BaseModel):
    sku: str
    qty: int

class OrderOut(BaseModel):
    id: int
    total: Decimal


# orders/_service.py
from decimal import Decimal
import billing                       # depend on billing's public API only
from orders._schemas import OrderIn, OrderOut

_PRICES = {"SKU-1": Decimal("12.50")}  # private module state

class OrderService:
    def __init__(self, invoices: billing.InvoiceApi) -> None:
        self._invoices = invoices

    def place(self, data: OrderIn) -> OrderOut:
        total = _PRICES[data.sku] * data.qty
        order_id = self._invoices.create(amount=total)
        return OrderOut(id=order_id, total=total)

def place_order(svc: OrderService, data: OrderIn) -> OrderOut:
    return svc.place(data)
```
Enforce the boundary with tooling: `import-linter` contracts ("orders may import billing, not billing._*"; "no cycles"), or Tach/pydeps.

## Example — TypeScript
An Angular feature with a single public entry point and an ESLint boundary rule.

```ts
// src/app/orders/index.ts — public API of the feature
export { ORDERS_ROUTES } from './orders.routes';
export type { OrderSummary } from './models';
export { OrdersFacade } from './orders.facade';
// NOT exported: order-row component, mappers, HTTP DTOs

// src/app/orders/orders.routes.ts
import { Routes } from '@angular/router';
export const ORDERS_ROUTES: Routes = [
  { path: '', loadComponent: () => import('./order-list').then((m) => m.OrderList) },
  { path: ':id', loadComponent: () => import('./order-detail').then((m) => m.OrderDetail) },
];

// src/app/app.routes.ts — other features use only the public API
import { Routes } from '@angular/router';
export const routes: Routes = [
  { path: 'orders', loadChildren: () => import('./orders').then((m) => m.ORDERS_ROUTES) },
];

// eslint.config.js (sketch): forbid deep imports into features
// 'no-restricted-imports': ['error', { patterns: ['**/orders/*', '!**/orders/index'] }]
// Nx users get this from @nx/enforce-module-boundaries with tags.
```

## In the wild
- Python packages, `__all__`, `_private` convention; `importlib.metadata` entry points for plugin modules.
- ES modules (`import`/`export`), Node `package.json` `"exports"` field (hides deep paths from consumers).
- Angular standalone features + lazy routes; Angular libraries' `public-api.ts`; Nx module boundaries; historically `NgModule`.
- Java Platform Module System, .NET assemblies with `internal`, Go `internal/` directories.
- Modular monolith tooling: `import-linter` (Python), Tach, dependency-cruiser and Nx (TS).

## Trade-offs
| Pros | Cons |
|---|---|
| Changes stay local; internals refactor freely | Designing good boundaries takes domain knowledge |
| Clear ownership per feature/team | Cross-module features need explicit APIs, more ceremony |
| Enables lazy loading and code splitting (frontend) | Wrong boundaries are costly to move later |
| Path to extracting services later | Python privacy is by convention; needs lint enforcement |

## Pitfalls
- **Circular imports between modules.** Fix: make dependencies one-directional; extract a shared kernel module or invert with an interface/event.
- **Shared `utils`/`common` module that grows forever.** Fix: move helpers to the feature that owns them; keep `common` tiny and stable.
- **Leaking ORM models across modules.** Fix: export DTOs or domain types; keep persistence private.
- **Barrel files that re-export everything.** Fix: export only the API; avoid deep `export *` chains.
- **Boundaries by technical layer only** (`services/`, `models/`). Fix: package by feature, layer inside the feature.

## Refactoring path
- **Introduce it:** pick one feature; move its files into one package; add an explicit public surface; run an import linter to list violations; fix callers to use the public API; add the lint contract to CI; repeat per feature.
- **Remove it / simplify:** merge modules that always change together and have chatty APIs between them; delete pass-through re-export layers.

## Related patterns
- [Facade](../structural/facade.md) — a module's public API is often a facade.
- [Singleton](../creational/singleton.md) — a Python/ES module is a process-wide single instance.
- [Layered](../architectural/layered.md) — layers inside a module or modules as layers.
- [Microservices](../architectural/microservices.md) — modules with network boundaries; get the module boundaries right first.
- [Microkernel](../architectural/microkernel.md) — a core plus plugin modules.
- [Private Class Data](../security/private-class-data.md) — encapsulation at class level rather than module level.
