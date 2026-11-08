# CQRS

**Category:** Cloud / distributed · **Also known as:** Command Query Responsibility Segregation, Read/write model split · **Related:** CQS, Event Sourcing, Publish-Subscribe, Sharding, Read-Through Cache, Domain Model, Saga

## Intent
Use separate models for changing state (commands) and for reading it (queries), so each can have its own shape, storage, consistency and scaling, instead of forcing one model to serve both.

## Problem
- The domain model is rich (aggregates enforcing invariants), but list screens need flat, denormalized data across 6 tables; the ORM produces N+1 queries or a 12-way join.
- Reads outnumber writes 100:1 and need different indexes, caching or a search engine; tuning for reads hurts writes and vice versa.
- DTOs, serializers and "view" fields leak into the domain entities.
- Dashboards and full-text search require data from several services.

Scenario: an order management system. Placing and changing orders runs complex rules (stock, credit limits). The back-office order list must filter by customer name, status, SKU and date across millions of orders in under 100 ms, and support full-text search.

## Solution and structure
```
            Commands (PlaceOrder, CancelOrder)                    Queries (SearchOrders, GetOrderView)
client ──► command handler ──► domain model ──► write DB          client ──► query handler ──► read model
                                   │                                                          ▲
                                   └── domain events / CDC / outbox ──► projector ────────────┘
                                                                       (denormalize, index)
```
- **Command side**: validates intent, loads aggregates, enforces invariants, persists; returns little (an id or nothing).
- **Query side**: thin handlers reading purpose-built views (SQL views, materialized tables, Elasticsearch, Redis); no business rules.
- **Synchronization**: same transaction (same DB, simplest), or asynchronously via domain events, transactional outbox, or change data capture (Debezium). Asynchronous means **eventual consistency** between write and read.
- **Levels** of separation, from cheap to expensive: separate classes/handlers on one DB → separate read tables/materialized views in the same DB → separate read store updated asynchronously → combined with Event Sourcing.

CQRS is distinct from **CQS** (a method either changes state or returns data, not both), which is a coding rule at the method level.

## When to use
- Read and write workloads differ strongly in shape, volume or scaling needs.
- A rich domain model on the write side, with list/search/report screens that do not fit it.
- Read models assembled from several services or bounded contexts.
- Already using Event Sourcing (the event store is unreadable for queries without projections).

## When NOT to use
- CRUD applications where the screens map onto the tables: a single model with an ORM and some tuned queries is far simpler. CQRS on simple CRUD is a well-known anti-pattern.
- When the team cannot handle eventual consistency in the UI ("I saved but the list doesn't show it").
- As a system-wide architecture. Apply it per bounded context, where it pays.
- When a database view, a covering index or a read replica would solve the read problem.

## Example — Python
Level 2: separate command and query paths, read model kept in the same Postgres and updated in the same transaction. Moving to an async projector later does not change the handlers' interfaces.
```python
from dataclasses import dataclass
from uuid import UUID, uuid4
from sqlalchemy import text
from sqlalchemy.ext.asyncio import AsyncSession

@dataclass(frozen=True)
class PlaceOrder:                      # command: intent, validated input
    customer_id: UUID
    lines: list[tuple[str, int]]       # (sku, qty)

@dataclass(frozen=True)
class OrderListItem:                   # read DTO: exactly what the screen needs
    id: UUID
    customer_name: str
    status: str
    total_cents: int

async def handle_place_order(db: AsyncSession, cmd: PlaceOrder) -> UUID:
    order_id = uuid4()
    async with db.begin():
        total = await price_and_reserve(db, cmd.lines)          # domain rules live here only
        await db.execute(text("INSERT INTO orders(id, customer_id, status, total_cents) "
                              "VALUES (:i, :c, 'placed', :t)"), {"i": order_id, "c": cmd.customer_id, "t": total})
        await db.execute(text(                                  # projection, same transaction
            "INSERT INTO order_list_view(id, customer_name, status, total_cents) "
            "SELECT :i, name, 'placed', :t FROM customers WHERE id = :c"),
            {"i": order_id, "c": cmd.customer_id, "t": total})
    return order_id

async def search_orders(db: AsyncSession, status: str, q: str, limit: int = 50) -> list[OrderListItem]:
    rows = await db.execute(text(                               # query: no domain model, no joins
        "SELECT id, customer_name, status, total_cents FROM order_list_view "
        "WHERE status = :s AND customer_name ILIKE :q ORDER BY id DESC LIMIT :l"),
        {"s": status, "q": f"%{q}%", "l": limit})
    return [OrderListItem(**r._mapping) for r in rows]

async def price_and_reserve(db: AsyncSession, lines: list[tuple[str, int]]) -> int: ...
```
In FastAPI, map `POST /orders` to the command handler and `GET /orders` to the query handler; they can depend on different sessions (primary vs replica).

## Example — TypeScript
Angular front end consuming a CQRS backend: commands return an id, the read model may lag, so the UI applies an optimistic update and re-fetches.
```ts
import { Injectable, inject, signal } from '@angular/core';
import { HttpClient } from '@angular/common/http';
import { firstValueFrom } from 'rxjs';

interface OrderListItem { id: string; customerName: string; status: string; totalCents: number }
interface PlaceOrder { customerId: string; lines: { sku: string; qty: number }[] }

@Injectable({ providedIn: 'root' })
export class OrdersFacade {
  private http = inject(HttpClient);
  readonly list = signal<OrderListItem[]>([]);

  async search(status: string): Promise<void> {                      // query side
    this.list.set(await firstValueFrom(this.http.get<OrderListItem[]>('/api/orders', { params: { status } })));
  }

  async place(cmd: PlaceOrder, customerName: string): Promise<string> { // command side
    const { id } = await firstValueFrom(this.http.post<{ id: string }>('/api/orders/commands/place', cmd));
    this.list.update((l) => [{ id, customerName, status: 'placed', totalCents: 0 }, ...l]); // optimistic
    return id;                                         // read model catches up; next search() replaces this row
  }
}
```
NestJS ships `@nestjs/cqrs` (CommandBus, QueryBus, EventBus) if you want the buses on the Node backend; plain service methods are often enough.

## In the wild
- Libraries: NestJS `@nestjs/cqrs`, MediatR (.NET), Axon Framework (JVM), Marten (.NET, with event sourcing), `eventsourcing` (Python).
- Projections and sync: Debezium CDC → Kafka → Elasticsearch/OpenSearch; DynamoDB Streams → Lambda → read table; PostgreSQL materialized views (`REFRESH MATERIALIZED VIEW CONCURRENTLY`).
- Read stores: Elasticsearch, Redis, ClickHouse, read replicas.
- GraphQL BFFs often act as the query side composing read models.

## Trade-offs
| Pros | Cons |
|---|---|
| Each side optimized and scaled independently | Two models to build, test and keep in sync |
| Simpler queries; domain model free of view concerns | Eventual consistency visible to users (async variant) |
| Read store can be rebuilt from the source of truth | Projection bugs and replay tooling add operational work |
| Fits event-driven and multi-service read needs | Much more code than CRUD for simple domains |

## Pitfalls
- **Applying it everywhere.** Use it in the one bounded context that needs it.
- **Read-your-own-writes** surprises. Return the new state from the command, apply optimistic UI updates, or read from the write store for the author for a short time.
- **Dual writes** (write DB, then publish event) lose events on crash. Use a transactional outbox or CDC.
- **Non-idempotent projectors** double-count on redelivery. Track the last processed event/version per projection.
- **Business logic in the query side** (or queries hitting the domain model). Keep queries dumb.
- **No rebuild path.** Make projections replayable from events or from the write DB.

## Refactoring path
- **Introduce it:** start by splitting handlers (command vs query classes) on the same DB. Move heavy list queries onto purpose-built views or tables updated in the same transaction. Only when scale or shape demand it, make the projection asynchronous via outbox/CDC into a separate store, and handle lag in the UI.
- **Remove it / simplify:** if the read model mirrors the write tables one-to-one, drop the projection and query the write tables. Collapse async back to synchronous if consistency complaints outweigh the scaling benefit.

## Related patterns
- [CQS](../ui/cqs.md): the method-level rule CQRS scales up to the architecture.
- [Event Sourcing](../architectural/event-sourcing.md): commonly paired; events feed the projections.
- [Publish-Subscribe](../architectural/publish-subscribe.md): how projections receive changes.
- [Read-Through Cache](../optimization/read-through-cache.md): a lighter alternative when reads are just slow.
- [Domain Model](../data-access/domain-model.md): lives on the command side.
- [Sharding](../cloud/sharding.md): write side sharded by key; read model built for cross-shard queries.
- [Saga](../cloud/saga.md): coordinates commands across services.
