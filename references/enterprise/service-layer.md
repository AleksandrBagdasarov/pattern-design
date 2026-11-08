# Service Layer

**Category:** Enterprise integration · **Also known as:** Application service, use-case layer, interactor · **Related:** Facade, Unit of Work, Repository, DTO, Domain Model, Layered

## Intent
Define the application's boundary as a set of use-case operations that coordinate domain objects, repositories, transactions and side effects, so every entry point (HTTP, CLI, queue consumer) reuses the same logic.

## Problem
- Business workflows live inside FastAPI route functions or Angular-facing controllers; a Celery task that needs the same workflow copy-pastes it.
- Transactions start and end in random places; some paths send the confirmation email, others forget.
- Scenario: "place order" is implemented in the REST handler; now the same flow must run from a Kafka consumer and an admin CLI.
- Signals: route handlers longer than ~20 lines, `session.commit()` in controllers, tests that need an HTTP client to test a business rule.

## Solution and structure
One function or method per use case: `place_order(cmd) -> OrderId`. It loads aggregates via repositories, calls domain methods, commits through the Unit of Work, and triggers side effects (events, emails via ports). Adapters (HTTP, CLI, consumers) only translate input/output DTOs and call it.

```
HTTP route ─┐
CLI command ├─> OrderService.place_order(PlaceOrder DTO)
Kafka consumer┘      ├─ uow.customers.get / uow.orders.add
                    ├─ order.add_line(...), order.place()   (rules in the domain)
                    ├─ uow.commit()
                    └─ publisher / notifier (ports)
```
Keep it **thin**: orchestration only. If a service method has `if` branches about business rules, those rules probably belong to the domain model.

## When to use
- More than one entry point for the same workflows, or likely to be.
- Use cases spanning several aggregates/repositories plus transactions and side effects.
- You want to test use cases with fakes, without HTTP or a database.
- A clear API for the frontend team: each endpoint maps to one use case.

## When NOT to use
- Pure CRUD where the handler calls the ORM once: a service method that just forwards adds a layer without behavior.
- When the framework already provides the layer with conventions you follow (e.g. NestJS providers *are* your services; don't add another layer above them).
- A "god service" `OrderService` with 60 methods: split per use case or per sub-area instead of growing it.

## Example — Python
FastAPI adapter → plain service function → UoW. The service has no FastAPI imports.

```python
# services.py
from dataclasses import dataclass
from typing import Protocol
from uuid import UUID
from app.domain import Order
from app.uow import UnitOfWork

@dataclass(frozen=True)
class PlaceOrder:                     # input command (internal DTO)
    customer_id: UUID
    lines: list[tuple[str, int]]

class Notifier(Protocol):
    def order_placed(self, order_id: UUID, email: str) -> None: ...

class CustomerNotFound(Exception): ...

def place_order(cmd: PlaceOrder, uow: UnitOfWork, notifier: Notifier) -> UUID:
    with uow:
        customer = uow.customers.get(cmd.customer_id)
        if customer is None:
            raise CustomerNotFound(cmd.customer_id)
        order = Order(customer_id=customer.id)
        for sku, qty in cmd.lines:
            order.add_line(sku, qty, uow.prices.current(sku))
        order.place()
        uow.orders.add(order)
        uow.commit()
    notifier.order_placed(order.id, customer.email)   # after commit
    return order.id

# api.py — adapter only
from fastapi import APIRouter, Depends, HTTPException
from pydantic import BaseModel

router = APIRouter()

class PlaceOrderIn(BaseModel):
    customer_id: UUID
    lines: list[tuple[str, int]]

@router.post("/orders", status_code=201)
def post_order(body: PlaceOrderIn, uow: UnitOfWork = Depends(get_uow),
               notifier: Notifier = Depends(get_notifier)) -> dict[str, UUID]:
    try:
        return {"id": place_order(PlaceOrder(body.customer_id, body.lines), uow, notifier)}
    except CustomerNotFound:
        raise HTTPException(404, "customer not found")
```

## Example — TypeScript
NestJS: the controller is an adapter; the service owns the use case.

```ts
import { Body, Controller, Injectable, NotFoundException, Post } from '@nestjs/common';
import { DataSource } from 'typeorm';
import { Customer, Order } from './entities';
import { Mailer } from './mailer';

export class PlaceOrderDto { customerId!: string; lines!: { sku: string; qty: number }[] }

@Injectable()
export class PlaceOrderService {
  constructor(private readonly db: DataSource, private readonly mailer: Mailer) {}

  async execute(dto: PlaceOrderDto): Promise<string> {
    const { order, email } = await this.db.transaction(async (em) => {
      const customer = await em.findOneBy(Customer, { id: dto.customerId });
      if (!customer) throw new NotFoundException('customer not found');
      const order = Order.place(customer.id, dto.lines);   // rules in the domain
      await em.save(order);
      return { order, email: customer.email };
    });
    await this.mailer.orderPlaced(order.id, email);        // after commit
    return order.id;
  }
}

@Controller('orders')
export class OrdersController {
  constructor(private readonly placeOrder: PlaceOrderService) {}
  @Post() create(@Body() dto: PlaceOrderDto) { return this.placeOrder.execute(dto); }
}
```
In Angular, the analogous layer is a facade service (`inject()`-ed) that components call instead of talking to `HttpClient` and the store directly.

## In the wild
- NestJS `@Injectable()` providers, Spring `@Service` + `@Transactional`, ASP.NET application services, Django "services.py" convention (HackSoft styleguide).
- Clean Architecture "use case interactors"; cosmicpython "service layer" chapter.
- Angular facade services over NgRx/signals stores.

## Trade-offs
| Pros | Cons |
|---|---|
| One implementation of each use case for every entry point | Extra layer; pass-through methods for trivial CRUD |
| Clear transaction and side-effect boundary | Tends to absorb domain logic (anemic model) |
| Use cases testable with fakes, no HTTP | Large service classes become god objects |
| Thin, boring controllers | Error mapping between layers must be designed |

## Pitfalls
- **Framework types in services** (`Request`, `HTTPException`). Fix: raise domain/application exceptions; map them in the adapter.
- **Business rules in the service** while entities are data bags. Fix: push rules into domain methods.
- **Services calling other services' internals** in chains. Fix: share domain logic, or orchestrate from one use case; consider domain events.
- **Side effects inside the transaction** (email before commit). Fix: after commit or outbox.
- **One giant `XService`.** Fix: one class/function per use case or per cohesive group.

## Refactoring path
- **Introduce it:** pick the fattest route handler; extract its body into a function taking plain inputs and dependencies; move commits there; make the handler translate DTOs and errors only; reuse the function from the second entry point; add tests with fakes.
- **Remove it / simplify:** for pass-through methods with no orchestration, call the repository/ORM from the handler directly and delete the method.

## Related patterns
- [Facade](../structural/facade.md) — a service layer is a facade over the domain and infrastructure.
- [Unit of Work](../data-access/unit-of-work.md) — one service method ≈ one unit of work.
- [Repository](../data-access/repository.md) — what services use to load and save aggregates.
- [Domain Model](../data-access/domain-model.md) — where rules belong; the service orchestrates.
- [DTO](../enterprise/dto.md) — the input/output shapes of service methods and adapters.
- [Layered](../architectural/layered.md) / [Clean Architecture](../architectural/clean-architecture.md) — the service layer is the application ring.
