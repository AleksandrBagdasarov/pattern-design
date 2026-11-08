# Metadata Mapping

**Category:** Emergent / modern · **Also known as:** Declarative mapping, annotation-driven configuration, attribute-driven design · **Related:** Data Mapper, DTO, Intercepting Validator, Dynamic Proxy, Interpreter

## Intent
Describe *what* the mapping or behavior is (field ↔ column, field ↔ JSON key, validation rules, routes, DI tokens) as declarative metadata attached to code, and let one generic engine read that metadata to do the work, instead of writing the same mapping code by hand for every class.

## Problem
- Repetitive, hand-written translation code for every entity: `row["created_at"] → order.created_at`, `to_dict`, `from_dict`, validators, OpenAPI schemas — all kept in sync by discipline.
- Scenario: an API has 40 resource types. Each has a hand-written serializer, a validator and a SQL mapper. Renaming one field requires changing four files; one is always forgotten and the API returns `null` for the renamed field.
- Smells: parallel structures that must stay in sync (model + schema + docs + validator); copy-paste `to_dict` methods; field lists repeated as strings.

## Solution and structure
```
  class + metadata ──► metadata registry ──► generic engine
  (type hints,          (fields, types,       (ORM, serializer,
   Field(...),          constraints,           validator, router,
   decorators)          aliases, columns)      DI container)
```
- **Metadata**: type annotations, field descriptors (`Field(alias=...)`, `mapped_column(...)`), decorators (`@Component`, `@app.get`), or external files (XML/YAML mappings).
- **Registry/introspection**: collected once at class creation or import time (metaclass, `__init_subclass__`, decorator, `typing.get_type_hints`, TS decorators, compile-time codegen).
- **Engine**: one generic implementation that interprets metadata — the ORM's `Data Mapper`, Pydantic's validator core, FastAPI's OpenAPI generator.

Fowler's original "Metadata Mapping" is the ORM case (field ↔ column). The modern generalization is any framework driven by declarations: one source of truth generates validation, serialization, docs and persistence.

## When to use
- Many classes need the same kind of transformation (persistence, serialization, validation, docs).
- A single declaration should feed several consumers (OpenAPI schema + validation + TS client generation).
- Library/framework code that must work on user-defined types it has never seen.

## When NOT to use
- 2–3 types with stable shapes: an explicit function `order_to_row(order)` is clearer, greppable and debuggable.
- Mappings with heavy, case-by-case business logic: metadata turns into a mini-language of hooks and escapes; write code.
- Hot paths where reflection cost matters and the framework does not compile/cache the metadata (Pydantic v2 and SQLAlchemy do cache; a hand-rolled `getattr` loop may not).
- Don't build your own engine when Pydantic, SQLAlchemy, dataclasses, `attrs`/`cattrs`, class-validator or Zod already exist.

## Example — Python
A tiny engine reading metadata from `Annotated` — the same mechanism Pydantic and FastAPI use. In real code, use Pydantic; this shows the pattern.

```python
from dataclasses import dataclass, fields
from datetime import datetime
from typing import Annotated, Any, get_type_hints

@dataclass(frozen=True)
class Column:
    name: str
    to_db: Any = None      # optional converters
    from_db: Any = None

@dataclass
class Order:
    id: Annotated[int, Column("order_id")]
    customer_email: Annotated[str, Column("email")]
    created_at: Annotated[datetime, Column("created_ts", to_db=datetime.timestamp,
                                           from_db=datetime.fromtimestamp)]

def _columns(cls: type) -> dict[str, Column]:
    hints = get_type_hints(cls, include_extras=True)
    return {f.name: next(m for m in hints[f.name].__metadata__ if isinstance(m, Column))
            for f in fields(cls)}

def to_row(obj: Any) -> dict[str, Any]:
    return {c.name: (c.to_db or (lambda v: v))(getattr(obj, attr))
            for attr, c in _columns(type(obj)).items()}

def from_row[T](cls: type[T], row: dict[str, Any]) -> T:
    return cls(**{attr: (c.from_db or (lambda v: v))(row[c.name])
                  for attr, c in _columns(cls).items()})
```
With Pydantic the same declaration is `customer_email: EmailStr = Field(alias="email")`, and it gives validation, `model_dump(by_alias=True)` and a JSON Schema for free. With SQLAlchemy 2.0: `customer_email: Mapped[str] = mapped_column("email")`.

## Example — TypeScript
TypeScript erases types at runtime, so metadata must exist as values. The idiomatic modern form is a schema object (Zod) from which the static type is *derived*, avoiding drift.

```ts
import { z } from 'zod';
import { HttpClient } from '@angular/common/http';
import { Injectable, inject } from '@angular/core';
import { map, type Observable } from 'rxjs';

// One declaration: runtime validation + wire mapping + static type.
const OrderWire = z.object({
  order_id: z.number().int(),
  email: z.string().email(),          // Zod 4 prefers z.email()
  created_ts: z.number(),
});
const Order = OrderWire.transform(w => ({
  id: w.order_id,
  customerEmail: w.email,
  createdAt: new Date(w.created_ts * 1000),
}));
export type Order = z.output<typeof Order>;

@Injectable({ providedIn: 'root' })
export class OrdersApi {
  private http = inject(HttpClient);
  get(id: number): Observable<Order> {
    return this.http.get<unknown>(`/api/orders/${id}`).pipe(map(raw => Order.parse(raw)));
  }
}
```
Angular itself is metadata-driven (`@Component({ selector, template, ... })` compiled ahead of time). Decorator support differs between TS "experimental" and standard (TC39) decorators; `reflect-metadata`-based libraries (class-validator, TypeORM, NestJS) rely on `emitDecoratorMetadata`, which standard decorators don't provide — check before mixing.

## In the wild
- Python: Pydantic models and `Field`, dataclasses, `attrs`, SQLAlchemy declarative `Mapped`/`mapped_column`, FastAPI route decorators and `Annotated[..., Depends()]`, Django models, marshmallow.
- TS/JS: Angular `@Component`/`@Input`/`input()`, NestJS decorators, TypeORM/MikroORM entities, class-validator/class-transformer, Zod/Valibot schemas, OpenAPI codegen.
- Elsewhere: JPA/Hibernate annotations, Protobuf/Avro schemas (metadata outside code), JSON Schema.

## Trade-offs
| Pros | Cons |
|---|---|
| One source of truth; removes boilerplate | "Magic": behavior not visible at the call site |
| Several outputs from one declaration (validation, docs, DB) | Errors surface at import/runtime, sometimes cryptically |
| Consistent behavior across all types | Escape hatches for odd cases complicate the metadata |
| Easy to add fields | Debugging requires understanding the engine |

## Pitfalls
- Using the same model for DB, API input and API output: metadata for one concern leaks into the others (exposing `password_hash`). Use separate DTOs/schemas.
- Relying on TS types at runtime (`http.get<Order>` does not validate anything). Validate with a runtime schema at the boundary.
- `from __future__ import annotations` / string annotations with locally defined types break `get_type_hints`; keep types importable at module level.
- Reflection per call in hot loops; precompute and cache the metadata once per class.
- Metadata DSLs that grow conditionals — when it reads like a program, write a function instead.

## Refactoring path
- **Introduce it:** find the duplicated mapping code; pick an existing engine (Pydantic, SQLAlchemy, Zod); convert one type, with a test comparing old and new output (golden master); migrate the rest; delete the hand-written mappers.
- **Remove it / simplify:** for a few types or highly irregular mappings, replace metadata with explicit mapper functions; keep the schema only at the trust boundary.

## Related patterns
- [Data Mapper](../data-access/data-mapper.md) — the ORM engine that consumes mapping metadata.
- [DTO](../enterprise/dto.md) — boundary shapes usually declared with metadata (Pydantic, Zod).
- [Intercepting Validator](../security/intercepting-validator.md) — validation rules declared as metadata, enforced centrally.
- [Dynamic Proxy](../structural/dynamic-proxy.md) — another reflection-driven generic mechanism.
- [Interpreter](../behavioral/interpreter.md) — metadata is effectively a small language the engine interprets.
