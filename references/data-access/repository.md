# Repository

**Category:** Data access · **Also known as:** — · **Related:** Unit of Work, Data Mapper, Domain Model, Query Object, Specification, Fake Object

## Intent
Give the domain a collection-like interface for loading and storing aggregates, hiding the persistence technology behind it.

## Problem
- Service code is full of `session.execute(select(...).where(...))`; business logic and SQL are interleaved and untestable without a database.
- The same query (`active orders for customer`) is copy-pasted in five places with small differences.
- Scenario: you want to unit-test `checkout()` but it needs Postgres because it builds queries inline.
- You plan to move one aggregate to DynamoDB or an external API and every caller would have to change.

## Solution and structure
Define an interface in domain terms (`get`, `add`, `list_overdue`) that accepts and returns domain objects. A concrete implementation (SQLAlchemy, HTTP, in-memory) lives in the infrastructure layer.

```
Service ──uses──> OrderRepository (Protocol: get, add, list_unpaid_since)
                        ▲                 ▲
          SqlOrderRepository     InMemoryOrderRepository (tests)
                 │
            Session (Unit of Work)
```
- One repository **per aggregate root**, not per table.
- The repository does not commit; the Unit of Work does.
- Methods return domain objects or plain values, never ORM `Query`/`Select` objects.

## When to use
- You have a Domain Model and want it persistence-ignorant.
- Business logic must be tested fast with an in-memory fake.
- Queries need a single named home (`list_unpaid_since(date)`).
- The storage backend is likely to differ across aggregates or change.

## When NOT to use
- Simple CRUD services where the ORM is the model: Django/Active Record managers already are repository-like; wrapping them adds nothing.
- Reporting and complex read queries: use a read model/Query Object or raw SQL in a query service (CQRS read side).
- A generic `Repository<T>` with `find_by_anything(**kwargs)` — this is the ORM again with extra steps. Prefer specific methods.
- Tiny scripts: just use the session.

## Example — Python
`Protocol` for the port, SQLAlchemy 2.x for the adapter, a dict for tests.

```python
from datetime import datetime
from typing import Protocol
from uuid import UUID
from sqlalchemy import select
from sqlalchemy.orm import Session
from app.domain import Order, Status  # mapped imperatively elsewhere

class OrderRepository(Protocol):
    def get(self, order_id: UUID) -> Order | None: ...
    def add(self, order: Order) -> None: ...
    def list_unpaid_since(self, since: datetime) -> list[Order]: ...

class SqlOrderRepository:
    def __init__(self, session: Session) -> None:
        self._s = session
    def get(self, order_id: UUID) -> Order | None:
        return self._s.get(Order, order_id)
    def add(self, order: Order) -> None:
        self._s.add(order)            # no commit: the Unit of Work owns that
    def list_unpaid_since(self, since: datetime) -> list[Order]:
        stmt = select(Order).where(Order.status == Status.DRAFT, Order.created_at >= since)
        return list(self._s.scalars(stmt))

class InMemoryOrderRepository:
    def __init__(self) -> None:
        self._items: dict[UUID, Order] = {}
    def get(self, order_id: UUID) -> Order | None:
        return self._items.get(order_id)
    def add(self, order: Order) -> None:
        self._items[order.id] = order
    def list_unpaid_since(self, since: datetime) -> list[Order]:
        return [o for o in self._items.values()
                if o.status is Status.DRAFT and o.created_at >= since]
```
In FastAPI, provide the repository with `Depends(get_order_repo)`, and override it in tests via `app.dependency_overrides`.

## Example — TypeScript
NestJS/Node style: an abstract class doubles as a DI token.

```ts
import { Injectable } from '@nestjs/common';
import { Pool } from 'pg';
import { Order } from './order';

export abstract class OrderRepository {
  abstract get(id: string): Promise<Order | undefined>;
  abstract save(order: Order): Promise<void>;
  abstract listUnpaidSince(since: Date): Promise<Order[]>;
}

@Injectable()
export class PgOrderRepository extends OrderRepository {
  constructor(private readonly pool: Pool) { super(); }

  async get(id: string): Promise<Order | undefined> {
    const { rows } = await this.pool.query('SELECT * FROM orders WHERE id = $1', [id]);
    return rows[0] ? Order.fromRow(rows[0]) : undefined;
  }
  async save(order: Order): Promise<void> {
    const r = order.toRow();
    await this.pool.query(
      `INSERT INTO orders (id, customer_id, status, total_cents) VALUES ($1,$2,$3,$4)
       ON CONFLICT (id) DO UPDATE SET status = EXCLUDED.status, total_cents = EXCLUDED.total_cents`,
      [r.id, r.customerId, r.status, r.totalCents]);
  }
  async listUnpaidSince(since: Date): Promise<Order[]> {
    const { rows } = await this.pool.query(
      `SELECT * FROM orders WHERE status = 'draft' AND created_at >= $1`, [since]);
    return rows.map(Order.fromRow);
  }
}
// module: providers: [{ provide: OrderRepository, useClass: PgOrderRepository }]
```
In Angular, a "repository" is usually an `@Injectable({ providedIn: 'root' })` service wrapping `HttpClient`; name it for its role (`OrdersApi`) and keep it free of view state.

## In the wild
- Spring Data repositories, TypeORM `Repository<Entity>`, MikroORM `EntityRepository`, Prisma delegates (`prisma.order`) — framework-provided, mostly generic.
- Django model managers (`Order.objects`) play the same role for Active Record.
- cosmicpython.com chapter 2 — the canonical Python treatment.

## Trade-offs
| Pros | Cons |
|---|---|
| Domain and services testable with an in-memory fake | One more layer and interface to maintain |
| Queries named and centralized | Easy to leak ORM details (lazy loading, query objects) |
| Storage can change per aggregate | Hides performance characteristics (N+1, missing indexes) |
| Clear seam for caching or auditing (Decorator) | Generic repositories tempt you into a leaky abstraction |

## Pitfalls
- **Returning ORM queries or `IQueryable`:** callers build SQL outside the repository. Fix: return lists/domain objects; add a named method.
- **Committing inside `add()`:** breaks multi-step atomicity. Fix: Unit of Work commits.
- **Repository per table:** `OrderLineRepository` lets callers bypass the aggregate root. Fix: one per aggregate.
- **Lazy-loaded relations accessed after the session closes** (`DetachedInstanceError`). Fix: eager-load what the aggregate needs (`selectinload`).
- **Fake drifts from real behavior** (uniqueness, ordering). Fix: run a shared contract test suite against both implementations.

## Refactoring path
- **Introduce it:** find repeated inline queries in services; extract each into a named method on a new `XRepository` class taking the session; switch services to it; extract a `Protocol`; add an in-memory fake and move service tests to it.
- **Remove it / simplify:** if every method is a one-line pass-through to the ORM and you have no fake-based tests, inline the calls and delete the interface, or use the framework's built-in repository.

## Related patterns
- [Unit of Work](../data-access/unit-of-work.md) — owns the transaction that repositories participate in.
- [Data Mapper](../data-access/data-mapper.md) — does the actual object↔row mapping beneath the repository.
- [Domain Model](../data-access/domain-model.md) — repositories exist per aggregate root.
- [Query Object](../data-access/query-object.md) — pass criteria into a repository without leaking SQL.
- [Specification](../behavioral/specification.md) — domain-level criteria a repository can translate to a query.
- [Table Data Gateway](../data-access/table-data-gateway.md) — table-centric, returns rows instead of aggregates.
- [Fake Object](../testing/fake-object.md) — the in-memory repository used in tests.
- [Hexagonal](../architectural/hexagonal.md) — the repository interface is a driven port.
