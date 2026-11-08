# Entity-Control-Boundary (ECB)

**Category:** Architectural · **Also known as:** Boundary-Control-Entity (BCE), robustness analysis (Jacobson) · **Related:** Clean Architecture, Hexagonal, MVC, Service Layer, Domain Model

## Intent
Classify every object derived from a use case into one of three stereotypes: **Boundary** (talks to actors: UI, API, external systems), **Control** (orchestrates one use case) and **Entity** (long-lived business data and rules). The classification turns use-case text into a first-cut design.

## Problem
- Requirements are written as use cases or user stories, but the team jumps straight to tables and endpoints; workflow logic ends up scattered in controllers and models.
- No agreed rule for "where does this method go?"
- Scenario: "Customer requests a refund" involves a web form, a payment provider, a refund policy check and an order record. Without a rule, the policy lands in the HTTP handler in one place and in the order model in another.

## Solution and structure
```
  Actor ──► Boundary ──► Control ──► Entity
                            │
                            └──► Boundary (to external system)
```
Robustness rules (Jacobson):
- Actors talk only to Boundaries.
- Boundaries talk to Controls (and to actors), not directly to Entities for business decisions.
- Controls talk to Boundaries, Entities and other Controls.
- Entities talk only to Controls (and to other Entities in rich models).

Mapping to modern code:
- **Boundary:** FastAPI routers, Angular components, CLI commands, payment gateway clients, message consumers.
- **Control:** a use-case function/class or application service (one per use case, named with a verb).
- **Entity:** domain objects and their repositories.

## When to use
- Analysis and early design from use cases (workshops, ADRs, interview whiteboard design).
- A lightweight vocabulary for "where does this go" in small-to-medium services.
- As a stepping stone toward Clean Architecture: Boundaries ≈ adapters, Controls ≈ use cases, Entities ≈ entities.

## When NOT to use
- As a folder structure for its own sake (`boundaries/`, `controls/`, `entities/` with one file each in a CRUD app).
- When the team already uses Layered/Clean vocabulary: map to it rather than adding a third naming scheme.
- Data-processing or event-streaming systems, which are not use-case shaped; use Pipes and Filters or Event-Driven.

## Example — Python
```python
# Use case: "Customer requests a refund"
from dataclasses import dataclass
from datetime import datetime, timedelta, UTC
from decimal import Decimal
from typing import Protocol
from fastapi import APIRouter, Depends, HTTPException

@dataclass
class Order:                                         # Entity
    id: str
    paid_amount: Decimal
    paid_at: datetime
    refunded: bool = False
    def refundable(self, now: datetime) -> bool:
        return not self.refunded and now - self.paid_at <= timedelta(days=30)

class Orders(Protocol):                              # Entity store
    def get(self, order_id: str) -> Order: ...
    def save(self, order: Order) -> None: ...

class PaymentProvider(Protocol):                     # Boundary to an external system
    def refund(self, order_id: str, amount: Decimal) -> None: ...

@dataclass
class RequestRefund:                                 # Control: one use case
    orders: Orders
    payments: PaymentProvider
    def __call__(self, order_id: str) -> None:
        order = self.orders.get(order_id)
        if not order.refundable(datetime.now(UTC)):
            raise ValueError("refund window closed")
        self.payments.refund(order.id, order.paid_amount)
        order.refunded = True
        self.orders.save(order)

router = APIRouter()                                 # Boundary to the actor

def request_refund() -> RequestRefund: ...           # wired in the composition root

@router.post("/orders/{order_id}/refund", status_code=202)
def refund(order_id: str, uc: RequestRefund = Depends(request_refund)) -> None:
    try:
        uc(order_id)
    except ValueError as e:
        raise HTTPException(409, str(e))
```

## Example — TypeScript
```ts
// Angular frontend slice of the same use case
import { Component, Injectable, inject, signal } from '@angular/core';
import { HttpClient } from '@angular/common/http';
import { firstValueFrom } from 'rxjs';

export interface Order { id: string; paidAmount: number; refundable: boolean }   // Entity (client view)

@Injectable({ providedIn: 'root' })
export class RefundApi {                                                       // Boundary (to backend)
  private http = inject(HttpClient);
  request(id: string) { return firstValueFrom(this.http.post<void>(`/api/orders/${id}/refund`, {})); }
}

@Injectable({ providedIn: 'root' })
export class RequestRefundControl {                                            // Control
  private api = inject(RefundApi);
  readonly status = signal<'idle' | 'pending' | 'done' | 'rejected'>('idle');
  async run(order: Order) {
    if (!order.refundable) { this.status.set('rejected'); return; }
    this.status.set('pending');
    try { await this.api.request(order.id); this.status.set('done'); }
    catch { this.status.set('rejected'); }
  }
}

@Component({                                                                   // Boundary (to actor)
  selector: 'app-refund-button',
  template: `<button (click)="control.run(order)">Refund</button> {{ control.status() }}`,
})
export class RefundButton {
  protected control = inject(RequestRefundControl);
  order: Order = { id: 'o-1', paidAmount: 50, refundable: true };
}
```

## In the wild
- UML robustness diagrams (ICONIX process), Jacobson's OOSE; Java EE "BCE" packaging (Adam Bien's `boundary/control/entity` per component).
- Clean Architecture explicitly credits ECB for its interactor/entity split.

## Trade-offs
| Pros | Cons |
|---|---|
| Direct path from use-case text to classes | Analysis-level: silent on persistence, DI, transactions |
| Simple three-way rule anyone can apply | Can produce one Control per trivial use case |
| Keeps workflow out of UI and entities | Vocabulary less known among web developers today |
| Maps cleanly onto Clean/Hexagonal | Folder-per-stereotype splits features apart |

## Pitfalls
- **Boundaries calling entities to make decisions:** route through a Control, or the rule duplicates across boundaries.
- **Controls becoming god services:** one Control per use case, named with a verb (`RequestRefund`, not `OrderManager`).
- **Anemic entities:** rules about the entity itself (`refundable`) belong on the entity; Controls orchestrate.
- **Package by stereotype in large systems:** package by feature first (`refunds/boundary`, `refunds/control`, `refunds/entity`).

## Refactoring path
- **Introduce it:** list the use cases; for each, draw actor → boundary → control → entity; move workflow out of handlers into one control per use case; move entity rules onto entities.
- **Remove it / simplify:** if controls only forward a call to a repository, let the boundary call the repository directly for pure reads and keep controls only for real workflows.

## Related patterns
- [Clean Architecture](../architectural/clean-architecture.md) — formalizes Control as use-case interactor and adds the dependency rule.
- [Hexagonal](../architectural/hexagonal.md) — Boundaries become adapters behind ports.
- [MVC](../architectural/mvc.md) — MVC's controller is a UI-input controller, not an ECB use-case Control.
- [Service Layer](../enterprise/service-layer.md) — groups Controls into service objects.
- [Domain Model](../data-access/domain-model.md) — the rich form of ECB Entities.
