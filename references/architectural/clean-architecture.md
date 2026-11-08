# Clean Architecture

**Category:** Architectural · **Also known as:** Uncle Bob's Clean Architecture · **Related:** Hexagonal, Onion, Layered, Dependency Injection, Repository, Service Layer

## Intent
Organize code in concentric rings (entities → use cases → interface adapters → frameworks and drivers) and enforce the **Dependency Rule**: source-code dependencies point only inward, so business rules know nothing about the web framework, database or UI.

## Problem
- Business logic imports `fastapi`, `sqlalchemy` or `boto3`; unit tests need a running DB or heavy mocking.
- A framework upgrade (SQLAlchemy 1.x → 2.x, Pydantic v1 → v2) ripples through the core logic.
- The same use case must be driven from HTTP, a CLI, a queue consumer and tests.
- Scenario: a payments service must switch from Stripe to Adyen, and from Postgres to DynamoDB for one aggregate. Every rule that calls the SDK or ORM directly has to be rewritten and retested.

## Solution and structure
```
┌──────────── Frameworks & drivers (FastAPI, SQLAlchemy, Stripe SDK) ────────────┐
│ ┌──────── Interface adapters (controllers, presenters, repo impls) ─────────┐ │
│ │ ┌──────── Use cases (application business rules, ports) ───────────────┐ │ │
│ │ │          Entities (enterprise business rules)                        │ │ │
│ │ └──────────────────────────────────────────────────────────────────────┘ │ │
│ └──────────────────────────────────────────────────────────────────────────┘ │
└────────────────────────────────────────────────────────────────────────────────┘
          every arrow of `import` points inward
```
- **Entities:** domain objects and invariants, pure Python/TS.
- **Use cases (interactors):** one class or function per application action; define the *ports* (interfaces) they need, e.g. `PaymentGateway`, `OrderRepository`.
- **Interface adapters:** controllers convert requests to use-case input; presenters format output; repository/gateway implementations fulfil ports.
- **Frameworks and drivers:** wiring, DI composition root, config.
- Control flow goes outward (use case calls the gateway), but the dependency points inward: dependency inversion via ports.

## When to use
- Domain-heavy services expected to live for years and outlive frameworks.
- Several delivery mechanisms (REST, gRPC, CLI, workers) over the same use cases.
- You need fast unit tests of business rules with in-memory fakes.

## When NOT to use
- CRUD with thin rules: you will write four classes per endpoint to move a row. Use Layered or plain FastAPI + SQLAlchemy.
- Prototypes, scripts, short-lived services.
- Over-engineering smells: request/response models duplicated per ring with identical fields; a `Presenter` that only returns its input; interfaces with exactly one implementation and no test fake.

## Directory layout
```
src/payments/
  domain/        entities.py, errors.py              # imports nothing outside stdlib
  application/   ports.py, charge_order.py           # imports domain only
  adapters/      http_api.py, sql_repo.py, stripe_gateway.py
  main.py        composition root: builds adapters, injects into use cases
```

## Example — Python
```python
# application/ports.py
from typing import Protocol
from decimal import Decimal
from dataclasses import dataclass

@dataclass
class Order:                      # domain/entities.py
    id: str
    amount: Decimal
    paid: bool = False
    def mark_paid(self) -> None:
        if self.paid:
            raise ValueError("already paid")
        self.paid = True

class OrderRepository(Protocol):
    def get(self, order_id: str) -> Order: ...
    def save(self, order: Order) -> None: ...

class PaymentGateway(Protocol):
    def charge(self, order_id: str, amount: Decimal) -> str: ...

# application/charge_order.py — use case, no framework imports
@dataclass
class ChargeOrder:
    orders: OrderRepository
    gateway: PaymentGateway
    def __call__(self, order_id: str) -> str:
        order = self.orders.get(order_id)
        tx_id = self.gateway.charge(order.id, order.amount)
        order.mark_paid()
        self.orders.save(order)
        return tx_id

# adapters/http_api.py — outer ring depends on inner ring
from fastapi import APIRouter, Depends
router = APIRouter()

def charge_order_uc() -> ChargeOrder:             # overridden in main.py / tests
    raise NotImplementedError

@router.post("/orders/{order_id}/charge")
def charge(order_id: str, uc: ChargeOrder = Depends(charge_order_uc)) -> dict[str, str]:
    return {"transaction_id": uc(order_id)}
```
`typing.Protocol` gives structural ports without inheritance; tests pass an in-memory `FakeOrders` and `FakeGateway`.

## Example — TypeScript
```ts
// domain + application: no Angular, no HttpClient
export interface Order { id: string; amount: number; paid: boolean }
export abstract class OrderGateway {              // port (abstract class doubles as DI token)
  abstract load(id: string): Promise<Order>;
  abstract pay(id: string): Promise<Order>;
}
export class PayOrder {                           // use case
  constructor(private readonly gateway: OrderGateway) {}
  async execute(id: string): Promise<Order> {
    const order = await this.gateway.load(id);
    if (order.paid) throw new Error('already paid');
    return this.gateway.pay(id);
  }
}

// adapters: Angular-specific
import { Injectable, inject, makeEnvironmentProviders } from '@angular/core';
import { HttpClient } from '@angular/common/http';
import { firstValueFrom } from 'rxjs';

@Injectable()
export class HttpOrderGateway extends OrderGateway {
  private http = inject(HttpClient);
  load(id: string) { return firstValueFrom(this.http.get<Order>(`/api/orders/${id}`)); }
  pay(id: string) { return firstValueFrom(this.http.post<Order>(`/api/orders/${id}/pay`, {})); }
}

export const provideOrders = () => makeEnvironmentProviders([
  { provide: OrderGateway, useClass: HttpOrderGateway },
  { provide: PayOrder, useFactory: () => new PayOrder(inject(OrderGateway)) },
]);
```

## In the wild
- FastAPI `Depends` + `app.dependency_overrides` acts as the composition root and lets tests swap adapters.
- "Cosmic Python" (Percival & Gregory, *Architecture Patterns with Python*) is a close Python rendering.
- NestJS custom providers and Angular DI tokens play the same wiring role.

## Trade-offs
| Pros | Cons |
|---|---|
| Business rules testable in milliseconds without I/O | More files, interfaces and mapping code |
| Frameworks and vendors become replaceable details | Steep for juniors; ring boundaries get debated |
| Use cases document what the system does | Mapping between ring models costs time and can drift |
| Several delivery mechanisms reuse one core | Easy to apply ceremonially where CRUD would do |

## Pitfalls
- **ORM models as entities:** SQLAlchemy declarative classes in `domain/` drag the ORM inward. Either accept it consciously (pragmatic) or use imperative mapping / separate models.
- **Pydantic in the domain:** acceptable as a pragmatic choice for validation, but then the domain depends on Pydantic; decide explicitly.
- **Use cases calling other use cases deeply:** extract a domain service instead.
- **Ports shaped like the vendor SDK:** define ports in the use case's language (`charge(order_id, amount)`), not `create_payment_intent(...)`.
- **No enforcement:** add import contracts (`import-linter`) or ESLint boundaries; the rule decays otherwise.

## Refactoring path
- **Introduce it:** start from the core rule most painful to test. Extract it into a pure function/entity; define a Protocol for the I/O it needs; implement the Protocol with the existing code; wire it in the composition root; add fakes in tests. Grow ring by ring.
- **Remove it / simplify:** collapse presenter + controller into the handler; drop single-implementation ports that no test uses; merge request/response models with identical fields into one DTO.

## Related patterns
- [Hexagonal](../architectural/hexagonal.md) — same dependency inversion, phrased as ports and adapters without prescribed rings.
- [Onion](../architectural/onion.md) — near-identical rings; see its comparison table.
- [Layered](../architectural/layered.md) — the simpler style it evolves from; dependencies point down instead of in.
- [Dependency Injection](../behavioral/dependency-injection.md) — how adapters reach use cases.
- [Repository](../data-access/repository.md) — the most common port.
- [Service Layer](../enterprise/service-layer.md) — use cases are a fine-grained service layer.
