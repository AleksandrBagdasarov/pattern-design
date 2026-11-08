# Facade

**Category:** Structural · **Also known as:** — · **Related:** Adapter, Mediator, Service Layer, Module, Singleton, Abstract Factory

## Intent
Provide one simple, higher-level interface to a complex subsystem, so clients can do the common things without knowing the subsystem's classes, call order or configuration.

## Problem
- Clients must orchestrate many collaborators in the right order: create the inventory reservation, then charge payment, then create the shipment, then send the email, and roll back on failure.
- The same orchestration is copy-pasted across controllers, CLI commands and background jobs.
- Callers import ten internal classes, so any internal refactor breaks them.
- New team members cannot find "the one function to call" for a use case.

Scenario: in an Angular app, every component injects `Store`, dispatches three actions and selects four slices to show a cart. A `CartFacade` with `items`, `total`, `add(id)` and `checkout()` hides the store shape.

## Solution and structure
- **Facade**: knows which subsystem classes handle a request and delegates; contains coordination, not domain rules.
- **Subsystem classes**: do the real work; unaware of the facade; still directly accessible for advanced clients.
- **Clients**: talk to the facade for common cases.

```
Controller / CLI / Job
        │
     Facade ──> Inventory
        ├────> Payments
        ├────> Shipping
        └────> Notifications
```
A facade does not forbid direct subsystem access; it makes the common path easy.

## When to use
- A subsystem has many parts but most clients need a few common operations.
- You want to layer a system: each layer talks to the one below through a facade.
- You want to decouple clients from a library or legacy module you plan to replace.
- In Angular: hide the state-management implementation (NgRx, signal store, plain services) behind a component-facing API.

## When NOT to use
- The subsystem is already simple; a facade that forwards one-to-one adds a layer for nothing.
- You need to match an *existing* interface: use Adapter.
- The "facade" starts holding business rules and state: it is becoming a god object; move rules into the domain or into use-case classes.
- One giant `AppFacade` for the entire app: split by use case or feature.

## Example — Python
In Python a module of functions is often the best facade; a class is useful when it holds injected dependencies.

```python
from dataclasses import dataclass
from decimal import Decimal
from typing import Protocol


class Inventory(Protocol):
    def reserve(self, sku: str, qty: int) -> str: ...
    def release(self, reservation_id: str) -> None: ...

class Payments(Protocol):
    def charge(self, customer_id: str, amount: Decimal) -> str: ...

class Shipping(Protocol):
    def create_shipment(self, order_id: str, address: str) -> str: ...

class Mailer(Protocol):
    def send(self, to: str, template: str, **ctx: str) -> None: ...


@dataclass(frozen=True)
class PlaceOrder:
    order_id: str
    customer_id: str
    email: str
    address: str
    sku: str
    qty: int
    amount: Decimal


@dataclass
class CheckoutFacade:
    inventory: Inventory
    payments: Payments
    shipping: Shipping
    mailer: Mailer

    def place_order(self, cmd: PlaceOrder) -> str:
        reservation = self.inventory.reserve(cmd.sku, cmd.qty)
        try:
            self.payments.charge(cmd.customer_id, cmd.amount)
        except Exception:
            self.inventory.release(reservation)
            raise
        tracking = self.shipping.create_shipment(cmd.order_id, cmd.address)
        self.mailer.send(cmd.email, "order_placed", tracking=tracking)
        return tracking
```
A FastAPI route then becomes `return checkout.place_order(cmd)` with `checkout` provided via `Depends`.

## Example — TypeScript
Angular store facade with signals; components never see the store shape.

```ts
import { Injectable, computed, inject, signal } from '@angular/core';
import { HttpClient } from '@angular/common/http';
import { firstValueFrom } from 'rxjs';

interface CartItem { productId: number; name: string; price: number; qty: number }

@Injectable({ providedIn: 'root' })
export class CartFacade {
  private readonly http = inject(HttpClient);
  private readonly _items = signal<CartItem[]>([]);

  readonly items = this._items.asReadonly();
  readonly total = computed(() => this._items().reduce((s, i) => s + i.price * i.qty, 0));
  readonly isEmpty = computed(() => this._items().length === 0);

  add(item: Omit<CartItem, 'qty'>): void {
    this._items.update((items) => {
      const found = items.find((i) => i.productId === item.productId);
      return found
        ? items.map((i) => (i === found ? { ...i, qty: i.qty + 1 } : i))
        : [...items, { ...item, qty: 1 }];
    });
  }

  async checkout(): Promise<string> {
    const { orderId } = await firstValueFrom(
      this.http.post<{ orderId: string }>('/api/orders', { items: this._items() }),
    );
    this._items.set([]);
    return orderId;
  }
}
```
Swapping the internals to NgRx SignalStore later changes only this file.

## In the wild
- `requests` over `urllib3`/`http.client`; `shutil` over `os`/file primitives; `subprocess.run` over `Popen`.
- boto3 resources (`s3.Bucket(...).upload_file`) over the low-level client and the transfer manager.
- FastAPI `FastAPI` app object over Starlette routing, middleware and Pydantic validation.
- Angular `HttpClient` over `XMLHttpRequest`/`fetch` backends; "facade service" over NgRx is a widely used Angular convention.
- jQuery `$.ajax`; Node `fs/promises` convenience functions; AWS Amplify libraries over Cognito/S3/AppSync.

## Trade-offs
| Pros | Cons |
|---|---|
| Clients depend on a small, stable API | Can grow into a god object that knows everything |
| Internal refactors don't ripple to callers | Another layer to maintain and document |
| Orchestration lives in one testable place | Hides capabilities advanced clients need, tempting them to bypass it |
| Natural boundary for layering and migration | Pure pass-through facades add indirection without value |

## Pitfalls
- **God facade**: every feature adds a method. Fix: one facade per use case or feature (`CheckoutFacade`, `CartFacade`), not per app.
- **Business rules in the facade** (discount calculation, validation). Fix: keep rules in domain objects; the facade only coordinates.
- **Facade leaks subsystem types** in its signatures. Fix: accept/return simple DTOs or domain types.
- **Bypassing**: half the code calls the subsystem directly. Fix: make internals private to the module (see Module) or enforce with import-lint rules.
- **No transaction/compensation thinking** in orchestration. Fix: define failure handling explicitly (try/rollback, Unit of Work, Saga for distributed steps).

## Refactoring path
- **Introduce it:** search for the same sequence of calls in several callers; extract it into a facade method with intention-revealing name; replace callers one by one; then narrow what the module exports.
- **Remove it / simplify:** if most methods are one-line forwards, delete the facade and let callers use the subsystem; if it grew rules, extract them into domain services and leave a thin facade or none.

## Related patterns
- [Adapter](../structural/adapter.md) — conforms one object to an existing interface; Facade defines a new simple interface over many.
- [Mediator](../behavioral/mediator.md) — colleagues know and talk through the mediator; subsystem classes do not know the facade.
- [Service Layer](../enterprise/service-layer.md) — an application-wide set of facades that define use-case boundaries.
- [Module](../structural/module.md) — a module's public surface often is its facade.
- [Singleton](../creational/singleton.md) — facades are often single instances (`providedIn: 'root'`), via DI rather than a global.
- [Wrapper](../structural/wrapper.md) — how Facade compares with Adapter, Decorator and Proxy.
