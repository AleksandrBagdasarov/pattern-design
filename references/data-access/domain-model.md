# Domain Model

**Category:** Data access · **Also known as:** Rich domain model, DDD tactical design · **Related:** Repository, Unit of Work, Data Mapper, Value Object, Service Layer, Specification

## Intent
Put business rules and invariants inside objects that carry both data and behavior, so the rules live in one place and cannot be bypassed by callers.

## Problem
- Rules are scattered across endpoints, services and SQL: "an order cannot be paid twice" is checked in three handlers and missed in the fourth.
- Entities are bags of public fields (anemic model); every service mutates them directly, so invariants hold only by convention.
- Scenario: an `Order` gets lines added after it was shipped because a new admin endpoint set `order.lines.append(...)` without checking `status`.
- Signals: long `if order.status == ...` blocks in services, duplicated validation, bugs that are "invalid state reached through a different code path".

## Solution and structure
Model the domain as objects whose methods enforce invariants. State changes only through intention-revealing methods (`order.add_line()`, `order.pay()`), never by assigning fields from outside.

**DDD tactical building blocks:**
- **Entity** — has identity that persists across changes (`Order#42` is the same order after edits). Equality by ID.
- **Value Object** — no identity, immutable, equality by value (`Money`, `Email`, `Address`). Most fields should be value objects.
- **Aggregate** — a cluster of entities and value objects with one **aggregate root** that is the only entry point. Invariants are guaranteed inside the aggregate boundary; one transaction modifies one aggregate. Other aggregates are referenced by ID, not by object reference.
- **Domain Event** — a fact that happened (`OrderPaid`), recorded by the aggregate and published after commit so other aggregates or services react (eventual consistency between aggregates).
- **Domain Service** — a stateless operation that does not naturally belong to one entity (e.g. pricing across several aggregates).
- **Bounded Context** — the boundary inside which a model and its language are consistent. "Customer" in Billing and in Shipping are different models; translate at the edge (anti-corruption layer, Message Translator).

```
 Service Layer ──> Repository.get(id) ──> Order (aggregate root)
                                           ├── OrderLine (entity)
                                           ├── Money (value object)
                                           └── events: [OrderPaid]
 Service Layer ──> uow.commit() ──> publish events
```

## When to use
- Non-trivial business rules and invariants that change often and must hold regardless of entry point.
- Several use cases manipulate the same concepts (orders, ledgers, subscriptions, claims).
- Domain experts exist and a shared vocabulary pays off.
- You want to unit-test rules without a database.

## When NOT to use
- CRUD with little logic (admin panels, settings tables): use Active Record or Table Data Gateway plus DTOs.
- Reporting/analytics: query directly with SQL or a read model; aggregates are for writes.
- Small scripts or one-off ETL: a Transaction Script (a function per use case) is simpler.
- Smell of over-application: aggregates that are just getters/setters, or every table given an aggregate root with no invariants.

## Example — Python
Plain dataclasses, no ORM imports; persistence is a Data Mapper's job.

```python
from dataclasses import dataclass, field
from decimal import Decimal
from enum import StrEnum
from uuid import UUID, uuid4

@dataclass(frozen=True)
class Money:
    amount: Decimal
    currency: str
    def __add__(self, other: "Money") -> "Money":
        if other.currency != self.currency:
            raise ValueError("currency mismatch")
        return Money(self.amount + other.amount, self.currency)

class Status(StrEnum):
    DRAFT = "draft"; PAID = "paid"; SHIPPED = "shipped"

@dataclass(frozen=True)
class OrderPaid:
    order_id: UUID
    total: Money

@dataclass
class Order:  # aggregate root
    customer_id: UUID                     # other aggregate referenced by ID
    currency: str = "EUR"
    id: UUID = field(default_factory=uuid4)
    status: Status = Status.DRAFT
    _lines: list[tuple[str, int, Money]] = field(default_factory=list)
    events: list[object] = field(default_factory=list)

    def add_line(self, sku: str, qty: int, price: Money) -> None:
        if self.status is not Status.DRAFT:
            raise ValueError("cannot modify a non-draft order")
        if qty <= 0:
            raise ValueError("qty must be positive")
        self._lines.append((sku, qty, price))

    def total(self) -> Money:
        total = Money(Decimal(0), self.currency)
        for _, qty, price in self._lines:
            total += Money(price.amount * qty, price.currency)
        return total

    def pay(self) -> None:
        if self.status is not Status.DRAFT or not self._lines:
            raise ValueError("order not payable")
        self.status = Status.PAID
        self.events.append(OrderPaid(self.id, self.total()))
```

## Example — TypeScript
```ts
type Status = 'draft' | 'paid' | 'shipped';
export interface OrderPaid { type: 'OrderPaid'; orderId: string; totalCents: number }

export class Order {
  private readonly lines: { sku: string; qty: number; priceCents: number }[] = [];
  private _status: Status = 'draft';
  readonly events: OrderPaid[] = [];

  constructor(readonly id: string, readonly customerId: string) {}

  get status(): Status { return this._status; }

  addLine(sku: string, qty: number, priceCents: number): void {
    if (this._status !== 'draft') throw new Error('cannot modify a non-draft order');
    if (!Number.isInteger(qty) || qty <= 0) throw new Error('qty must be a positive integer');
    this.lines.push({ sku, qty, priceCents });
  }

  totalCents(): number {
    return this.lines.reduce((sum, l) => sum + l.qty * l.priceCents, 0);
  }

  pay(): void {
    if (this._status !== 'draft' || this.lines.length === 0) throw new Error('order not payable');
    this._status = 'paid';
    this.events.push({ type: 'OrderPaid', orderId: this.id, totalCents: this.totalCents() });
  }
}
```
On the Angular side a domain model rarely belongs in the browser; the frontend usually holds DTOs and view state. Keep rules on the server and mirror only what the UI needs for instant feedback.

## In the wild
- SQLAlchemy classical/imperative mapping (`registry.map_imperatively`) keeps domain classes free of ORM base classes.
- "Architecture Patterns with Python" (Percival & Gregory, cosmicpython.com) — FastAPI/Flask-style Domain Model + Repository + UoW + events.
- NestJS `@nestjs/cqrs` `AggregateRoot` with `apply()`/`commit()` for domain events.
- Axon (Java), EventStoreDB, Marten — aggregate-centric frameworks for event-sourced models.

## Trade-offs
| Pros | Cons |
|---|---|
| Invariants enforced in one place, impossible to bypass | More classes and mapping code than CRUD |
| Business rules unit-testable without I/O | Requires mapping to/from persistence (Data Mapper) |
| Ubiquitous language makes code readable to domain experts | Aggregate boundaries are hard to get right first time |
| Domain events decouple side effects | Cross-aggregate consistency becomes eventual |

## Pitfalls
- **Anemic model:** classes with only fields and services doing all logic. Fix: move the rule into a method on the object whose state it guards.
- **Huge aggregates** (Customer containing all Orders): lock contention and slow loads. Fix: small aggregates, reference by ID.
- **Modifying two aggregates in one transaction:** couples them. Fix: domain event plus a handler (or a Saga across services).
- **ORM leaking in** (`Base` subclasses, lazy relationships inside invariants). Fix: imperative mapping or separate persistence models.
- **Publishing events before commit:** consumers see changes that later roll back. Fix: collect on the aggregate, publish after commit or via an outbox.
- **Floats for money:** use `Decimal` or integer minor units.

## Refactoring path
- **Introduce it:** pick one rule that is duplicated across services; add a method on the entity that enforces it; route all callers through that method; make the field private/read-only; repeat. Introduce aggregates and events only when cross-object invariants appear.
- **Remove it / simplify:** if methods are only setters and there are no invariants, collapse to DTO + Active Record or plain functions (Transaction Script); delete the mapping layer.

## Related patterns
- [Repository](../data-access/repository.md) — loads and saves whole aggregates.
- [Unit of Work](../data-access/unit-of-work.md) — commits one aggregate's changes atomically and triggers event publishing.
- [Data Mapper](../data-access/data-mapper.md) — keeps the model persistence-ignorant.
- [Active Record](../data-access/active-record.md) — the simpler alternative for CRUD-shaped domains.
- [Value Object](../emergent/value-object.md) — the immutable building blocks of the model.
- [Service Layer](../enterprise/service-layer.md) — orchestrates use cases around the model; thin, not where rules live.
- [Specification](../behavioral/specification.md) — composable domain rules usable in queries.
- [Event Sourcing](../architectural/event-sourcing.md) — persists the aggregate as its domain events.
- [Message Translator](../enterprise/message-translator.md) — translation between bounded contexts.
