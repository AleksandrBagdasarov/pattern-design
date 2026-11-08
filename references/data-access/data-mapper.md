# Data Mapper

**Category:** Data access · **Also known as:** Mapper (SQLAlchemy ORM, Hibernate, TypeORM DataMapper mode) · **Related:** Active Record, Repository, Unit of Work, Domain Model, Metadata Mapping, DTO

## Intent
A separate layer moves data between in-memory objects and the database, so the objects know nothing about persistence and the database schema knows nothing about the objects.

## Problem
- Domain classes inherit from an ORM base and carry `save()`, column types and session state; you cannot test or reason about them without a database.
- The object shape and the table shape diverge: value objects (`Money` as two columns), an aggregate stored over three tables, renamed legacy columns.
- Scenario: a `Subscription` aggregate has a `BillingPeriod` value object and a list of `Discount`s; the DB has `subscriptions`, `subscription_discounts` and a legacy `bill_start_dt` column.

## Solution and structure
The mapper knows both sides; neither side knows the mapper.

```
Domain object (pure)  <──  Mapper (load/insert/update/delete)  ──>  Tables
                               ▲
                     Repository / Unit of Work call it
```
Two common forms:
- **Metadata-driven** (ORM): you declare the mapping once and the ORM generates SQL (SQLAlchemy imperative mapping, Hibernate XML/annotations).
- **Hand-written**: `to_row()` / `from_row()` functions per aggregate. Fine for a few aggregates or for non-SQL stores.

## When to use
- A rich Domain Model whose shape differs from the schema.
- You want domain objects testable and importable without the ORM.
- Legacy or shared schemas you cannot change.
- The same domain object is stored in different backends (SQL, document store, cache).

## When NOT to use
- Objects mirror tables one-to-one and logic is thin: Active Record (Django, Rails) is faster to build.
- Read-only reporting: map rows straight into DTOs (`select(...)` into Pydantic/TypedDict), no domain objects.
- A hand-written mapper for every table when an ORM declarative mapping would do — that is boilerplate you maintain forever.

## Example — Python
SQLAlchemy 2.x imperative mapping: the domain class has no ORM import; the mapping lives in infrastructure.

```python
# domain.py — no SQLAlchemy here
from dataclasses import dataclass, field
from decimal import Decimal
from uuid import UUID

@dataclass(frozen=True)
class Money:
    amount: Decimal
    currency: str

@dataclass
class Invoice:
    id: UUID
    customer_id: UUID
    total: Money
    lines: list["InvoiceLine"] = field(default_factory=list)

@dataclass
class InvoiceLine:
    sku: str
    qty: int

# orm.py — infrastructure
from sqlalchemy import Column, ForeignKey, Integer, Numeric, String, Table, Uuid
from sqlalchemy.orm import composite, registry, relationship

mapper_registry = registry()
invoices = Table("invoices", mapper_registry.metadata,
    Column("id", Uuid, primary_key=True), Column("customer_id", Uuid, nullable=False),
    Column("total_amount", Numeric(12, 2)), Column("total_currency", String(3)))
invoice_lines = Table("invoice_lines", mapper_registry.metadata,
    Column("id", Integer, primary_key=True, autoincrement=True),
    Column("invoice_id", ForeignKey("invoices.id")), Column("sku", String), Column("qty", Integer))

def start_mappers() -> None:
    mapper_registry.map_imperatively(InvoiceLine, invoice_lines)
    mapper_registry.map_imperatively(Invoice, invoices, properties={
        "total": composite(Money, invoices.c.total_amount, invoices.c.total_currency),
        "lines": relationship(InvoiceLine, lazy="selectin", cascade="all, delete-orphan"),
    })
```
Caveat: mapping frozen dataclasses with `composite` works for value objects; mapped entity classes must allow attribute assignment. Check the SQLAlchemy docs for your version's dataclass integration details.

## Example — TypeScript
A hand-written mapper for a store without an ORM (also works for mapping API payloads).

```ts
export interface InvoiceRow {
  id: string; customer_id: string; total_amount: string; total_currency: string;
  lines: { sku: string; qty: number }[];   // JSONB column
}

export class Invoice {
  constructor(
    readonly id: string,
    readonly customerId: string,
    readonly total: { amount: number; currency: string },
    readonly lines: ReadonlyArray<{ sku: string; qty: number }>,
  ) {}
}

export const InvoiceMapper = {
  toDomain(r: InvoiceRow): Invoice {
    return new Invoice(r.id, r.customer_id,
      { amount: Number(r.total_amount), currency: r.total_currency }, r.lines);
  },
  toRow(i: Invoice): InvoiceRow {
    return {
      id: i.id, customer_id: i.customerId,
      total_amount: i.total.amount.toFixed(2), total_currency: i.total.currency,
      lines: [...i.lines],
    };
  },
};
```
TypeORM and MikroORM support data-mapper style (`em.save(entity)`), though their entities still carry decorators — a pragmatic compromise.

## In the wild
- SQLAlchemy ORM (always a data mapper, even with declarative classes; imperative mapping gives full persistence ignorance).
- Hibernate/JPA, Doctrine (PHP), MikroORM, TypeORM in DataMapper mode, Entity Framework.
- `pydantic`'s `model_validate(row, from_attributes=True)` is a tiny read-only mapper into DTOs.

## Trade-offs
| Pros | Cons |
|---|---|
| Domain objects are plain, fast to test | Extra layer, mapping config to keep in sync |
| Object shape and schema evolve independently | Harder to see the SQL that runs |
| Supports value objects, inheritance, legacy schemas | Steeper learning curve than Active Record |
| Works with UoW for change tracking and batching | Hand-written mappers become boilerplate quickly |

## Pitfalls
- **Lazy loading surprises:** N+1 queries or access after the session closes. Fix: set `lazy="selectin"`/`joinedload` for what the aggregate always needs.
- **Leaky persistence fields** (`_sa_instance_state`, ORM-only IDs) used in domain logic. Fix: keep domain identity explicit.
- **Mapping both ways in API handlers** too: the mapper becomes a dumping ground. Fix: separate DTO mapping (API) from persistence mapping.
- **Silent precision loss** (Decimal ↔ float, timezone-naive datetimes). Fix: explicit types (`Numeric`, `DateTime(timezone=True)`), test round-trips.

## Refactoring path
- **Introduce it:** from Active Record models that grew business logic: create plain domain classes with the logic; map them to the existing tables (imperative mapping or `to_domain/to_row`); move callers through a Repository; delete logic from ORM models.
- **Remove it / simplify:** if domain classes equal tables and the mapper is identity mapping, switch to declarative ORM models (effectively Active Record style) and drop the hand-written layer.

## Related patterns
- [Active Record](../data-access/active-record.md) — the alternative where objects save themselves; see its decision table.
- [Repository](../data-access/repository.md) — the domain-facing interface that uses the mapper.
- [Unit of Work](../data-access/unit-of-work.md) — tells the mapper what changed.
- [Domain Model](../data-access/domain-model.md) — what the mapper keeps persistence-ignorant.
- [Metadata Mapping](../emergent/metadata-mapping.md) — the declarative form ORMs use to drive the mapper.
- [DTO](../enterprise/dto.md) — mapping at the API boundary rather than the DB boundary.
