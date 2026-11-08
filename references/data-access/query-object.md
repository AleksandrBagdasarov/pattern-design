# Query Object

**Category:** Data access · **Also known as:** Criteria object, query builder · **Related:** Repository, Specification, Fluent Builder, Interpreter, Value List Handler

## Intent
Represent a database query as an object built from composable criteria, instead of concatenating SQL strings or adding one repository method per filter combination.

## Problem
- A search endpoint has 8 optional filters and sort options; the code is a chain of `if filters.status: sql += " AND status = %s"`.
- String building invites SQL injection and broken `WHERE ... AND` syntax.
- The repository grows `find_by_status`, `find_by_status_and_customer`, `find_by_status_and_customer_since`, …
- Scenario: an admin orders list filtered by status, customer, date range, min total, free-text, sorted by any column, paginated.

## Solution and structure
A Query Object collects criteria (field, operator, value), ordering and paging, then is translated into the backend's query language by one piece of code (the repository or an ORM).

```
OrderSearch(status=..., since=..., sort=...)  ──>  repository.search(query)
                                                     └─ translate criteria → SQL / ORM statement
```
Usually you do not write the translator: SQLAlchemy `select()`, Django `Q`/QuerySet, Knex, Prisma `where` objects and Kysely are query objects already. Your job is a typed, domain-level *criteria* object and a small function that applies it.

## When to use
- Many optional filters/sorts combined at runtime (search screens, list APIs).
- The same criteria must be applied to different stores (SQL and an in-memory fake, or SQL and Elasticsearch).
- You want to validate, log, cache-key or serialize the query itself.

## When NOT to use
- A handful of fixed queries: named repository methods are clearer.
- Writing your own SQL AST when the ORM's expression language exists — you would build a worse SQLAlchemy.
- Complex analytical SQL (window functions, CTEs): write the SQL, test it, keep it in a query service.

## Example — Python
A Pydantic criteria object (also the FastAPI query DTO) applied to a SQLAlchemy `Select`.

```python
from datetime import datetime
from decimal import Decimal
from typing import Literal
from pydantic import BaseModel, Field
from sqlalchemy import Select, select
from app.orm import OrderRow

SORTABLE = {"created_at": OrderRow.created_at, "total": OrderRow.total}

class OrderSearch(BaseModel):
    status: Literal["draft", "paid", "shipped"] | None = None
    customer_id: int | None = None
    created_from: datetime | None = None
    created_to: datetime | None = None
    min_total: Decimal | None = Field(default=None, ge=0)
    sort: Literal["created_at", "total"] = "created_at"
    desc: bool = True

    def apply(self, stmt: Select[tuple[OrderRow]]) -> Select[tuple[OrderRow]]:
        if self.status is not None:
            stmt = stmt.where(OrderRow.status == self.status)
        if self.customer_id is not None:
            stmt = stmt.where(OrderRow.customer_id == self.customer_id)
        if self.created_from is not None:
            stmt = stmt.where(OrderRow.created_at >= self.created_from)
        if self.created_to is not None:
            stmt = stmt.where(OrderRow.created_at < self.created_to)
        if self.min_total is not None:
            stmt = stmt.where(OrderRow.total >= self.min_total)
        col = SORTABLE[self.sort]                  # whitelist: no user strings in SQL
        return stmt.order_by(col.desc() if self.desc else col.asc(), OrderRow.id)

# FastAPI: def list_orders(q: Annotated[OrderSearch, Query()], session: SessionDep): ...
#          rows = session.scalars(q.apply(select(OrderRow)).limit(50)).all()
```
Pydantic models as `Query()` parameters are supported in recent FastAPI versions (0.115+); check the FastAPI docs for your version.

## Example — TypeScript
A typed criteria object translated to Prisma's `where` (itself a query object).

```ts
import type { Prisma } from '@prisma/client';

export interface OrderSearch {
  status?: 'draft' | 'paid' | 'shipped';
  customerId?: number;
  createdFrom?: Date;
  createdTo?: Date;
  minTotal?: number;
  sort?: 'createdAt' | 'total';
  desc?: boolean;
}

export function toPrisma(q: OrderSearch): Prisma.OrderFindManyArgs {
  const where: Prisma.OrderWhereInput = {};
  if (q.status) where.status = q.status;
  if (q.customerId !== undefined) where.customerId = q.customerId;
  if (q.createdFrom || q.createdTo) where.createdAt = { gte: q.createdFrom, lt: q.createdTo };
  if (q.minTotal !== undefined) where.total = { gte: q.minTotal };
  const dir = q.desc === false ? 'asc' : 'desc';
  return { where, orderBy: [{ [q.sort ?? 'createdAt']: dir }, { id: dir }] };
}
```
On the Angular side the same `OrderSearch` interface types the filter form and is serialized into `HttpParams`; keep it in sync with the backend schema (OpenAPI codegen).

## In the wild
- SQLAlchemy Core/ORM `select()`, Django `Q` objects and QuerySets, Knex/Kysely, Prisma `where`, TypeORM `QueryBuilder`, MongoDB filter documents, Elasticsearch Query DSL.
- `django-filter`, `fastapi-filter` — query objects generated from declared filter fields.
- GraphQL input types and OData `$filter` are query objects on the wire.

## Trade-offs
| Pros | Cons |
|---|---|
| Composable filters without method explosion | Can become a leaky "query anything" API |
| Injection-safe when values are bound and columns whitelisted | Translator must keep up with every new operator |
| Serializable: cache keys, saved searches, audit | Arbitrary filters may hit unindexed columns |
| Same criteria usable on fakes and real stores | Harder to see final SQL than a handwritten query |

## Pitfalls
- **User-supplied column names** in `ORDER BY`/`WHERE`. Fix: whitelist mapping from allowed names to columns.
- **No stable tiebreaker in sort:** pagination duplicates/skips rows. Fix: always append the primary key.
- **Unbounded results:** a query object without a limit. Fix: enforce a default and maximum page size.
- **Filters on unindexed columns** in production. Fix: only expose filters you index; check `EXPLAIN`.
- **Generic `filters: dict[str, Any]`:** loses typing and validation. Fix: an explicit typed model.

## Refactoring path
- **Introduce it:** take the method with the most `if` branches building SQL; define a typed criteria model with those fields; move the branching into one `apply()`; replace the sibling `find_by_*` methods with calls passing criteria.
- **Remove it / simplify:** if only two or three combinations are actually used, replace the query object with named repository methods and delete the generic translator.

## Related patterns
- [Repository](../data-access/repository.md) — accepts query objects so callers don't see SQL.
- [Specification](../behavioral/specification.md) — domain rules as objects; a query object is its persistence-oriented cousin.
- [Fluent Builder](../creational/fluent-builder.md) — a chainable API for constructing queries.
- [Interpreter](../behavioral/interpreter.md) — translating a criteria tree into SQL is interpretation.
- [Value List Handler](../data-access/value-list-handler.md) — pages the results a query object selects.
