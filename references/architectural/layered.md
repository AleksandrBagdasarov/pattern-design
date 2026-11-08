# Layered Architecture (n-tier)

**Category:** Architectural · **Also known as:** n-tier, n-layer, three-layer architecture · **Related:** Clean Architecture, Hexagonal, Onion, Service Layer, Repository, MVC

## Intent
Split an application into horizontal layers, each with one kind of responsibility, where a layer may only call the layer(s) below it. The usual stack is presentation → application/service → domain → infrastructure (data access).

## Problem
- HTTP handlers contain SQL, business rules and response formatting in one 200-line function.
- The same rule ("an order over 10 000 needs approval") is copied into the API handler, the CLI command and the nightly job.
- Changing the database schema forces edits in route handlers; changing the JSON shape forces edits in queries.
- Scenario: a FastAPI app grew from 5 to 60 endpoints; every endpoint opens its own session, builds queries inline and validates business rules ad hoc. Nobody can find "where the rules live".

## Solution and structure
```
  Presentation   (routers, controllers, templates, DTOs)
       │ calls
  Application    (use cases / services, transactions, orchestration)
       │ calls
  Domain         (entities, business rules)
       │ calls
  Infrastructure (repositories, ORM, HTTP clients, queues)
```
- **Strict layering:** a layer calls only the one directly below. **Relaxed layering:** it may call any lower layer (common; e.g. presentation reads infrastructure for simple queries).
- Data crosses layers as DTOs or domain objects, never as framework request objects.
- In the classic form the domain depends on infrastructure (it calls repositories directly). Clean/Hexagonal/Onion invert exactly that arrow.

## When to use
- Conventional CRUD-plus-rules business apps, internal tools, most first versions of a service.
- Teams that need an obvious, widely understood structure with low ceremony.
- When the database really is the center of the app and is not expected to be swapped.

## When NOT to use
- A 300-line script or a single-purpose Lambda: one module with functions is enough.
- The domain is complex and must be tested without a DB, or must outlive framework/DB changes: use Clean/Hexagonal so dependencies point inward.
- Data-flow problems (ETL, streaming): Pipes and Filters fits better than request/response layers.
- Smell of over-application: every layer just forwards the call ("sinkhole" layers) with no logic.

## Example — Python
```python
# app/api/orders.py            (presentation)
# app/services/orders.py       (application)
# app/domain/orders.py         (domain)
# app/infra/order_repo.py      (infrastructure)
from dataclasses import dataclass
from decimal import Decimal
from fastapi import APIRouter, Depends, HTTPException
from sqlalchemy.orm import Session

@dataclass
class Order:                                   # domain
    id: int
    total: Decimal
    status: str = "new"
    def approve(self) -> None:
        if self.total > Decimal("10000") and self.status != "reviewed":
            raise ValueError("large orders need review first")
        self.status = "approved"

class OrderRepository:                         # infrastructure
    def __init__(self, session: Session) -> None:
        self.session = session
    def get(self, order_id: int) -> Order | None: ...
    def save(self, order: Order) -> None: ...

class OrderService:                            # application
    def __init__(self, repo: OrderRepository, session: Session) -> None:
        self.repo, self.session = repo, session
    def approve(self, order_id: int) -> Order:
        order = self.repo.get(order_id)
        if order is None:
            raise LookupError(order_id)
        order.approve()
        self.repo.save(order)
        self.session.commit()
        return order

router = APIRouter()                           # presentation

def get_service(session: Session = Depends(get_session)) -> OrderService:  # get_session: your DB dependency
    return OrderService(OrderRepository(session), session)

@router.post("/orders/{order_id}/approve")
def approve(order_id: int, svc: OrderService = Depends(get_service)) -> dict[str, str]:
    try:
        return {"status": svc.approve(order_id).status}
    except LookupError:
        raise HTTPException(404)
```
Handlers translate HTTP; the service owns the transaction; the rule sits on the entity.

## Example — TypeScript
```ts
// Angular frontend layering: component (presentation) → facade/service (application) → api client (infrastructure)
import { Injectable, inject, signal, Component } from '@angular/core';
import { HttpClient } from '@angular/common/http';
import { firstValueFrom } from 'rxjs';

export interface Order { id: number; total: number; status: string }

@Injectable({ providedIn: 'root' })
export class OrdersApi {                                  // infrastructure
  private http = inject(HttpClient);
  approve(id: number) { return this.http.post<Order>(`/api/orders/${id}/approve`, {}); }
}

@Injectable({ providedIn: 'root' })
export class OrdersFacade {                               // application
  private api = inject(OrdersApi);
  readonly error = signal<string | null>(null);
  async approve(id: number): Promise<void> {
    try { await firstValueFrom(this.api.approve(id)); this.error.set(null); }
    catch { this.error.set('Approval failed'); }
  }
}

@Component({ selector: 'app-approve', template: `<button (click)="facade.approve(42)">Approve</button>` })
export class ApproveButton { protected facade = inject(OrdersFacade); }   // presentation
```

## In the wild
- Django apps (views → forms/services → models), Spring/NestJS "controller → service → repository" scaffolds.
- FastAPI full-stack templates: `api/`, `crud/` or `services/`, `models/`.
- OSI network stack and TCP/IP are layered architectures in the strict sense.

## Trade-offs
| Pros | Cons |
|---|---|
| Simple, universally understood | Domain usually depends on the DB layer (hard to test in isolation) |
| Clear place for each kind of code | Features are spread across all layers (one change touches 4 folders) |
| Layers can be replaced or tested separately if interfaces are clean | Sinkhole layers add boilerplate without value |
| Enforceable with import-lint tools | Tends toward an anemic domain (rules drift into services) |

## Pitfalls
- **Layer skipping without a rule:** decide strict vs relaxed and enforce it (`import-linter` in Python, `eslint-plugin-boundaries` / Nx tags in TS).
- **Leaking ORM objects to the presentation layer:** lazy loads fire during serialization. Map to DTOs (Pydantic models) at the boundary.
- **Transactions in repositories:** each repo call commits separately. Own the transaction in the application layer (Unit of Work).
- **Fat services, anemic entities:** keep invariants on domain objects; services orchestrate.
- **Organizing only by layer in a large app:** combine with vertical slicing (`orders/api.py`, `orders/service.py`) so features stay together.

## Refactoring path
- **Introduce it:** pick one endpoint; extract SQL into a repository function; extract the rule into a service or entity method; leave the handler doing parse → call → format. Repeat per endpoint; add an import-lint contract last.
- **Remove it / simplify:** if layers only forward calls, collapse service and repository into one module per feature, or let handlers call the ORM directly for pure CRUD and keep a service only where rules exist.

## Related patterns
- [Clean Architecture](../architectural/clean-architecture.md) — same layers, but dependencies point inward to the domain.
- [Hexagonal](../architectural/hexagonal.md) — replaces "top/bottom" with "inside/outside" via ports.
- [Onion](../architectural/onion.md) — concentric version of layering with the domain at the center.
- [Service Layer](../enterprise/service-layer.md) — the application layer in this stack.
- [Repository](../data-access/repository.md) — the usual shape of the infrastructure layer.
- [MVC](../architectural/mvc.md) — often the presentation layer's internal structure.
