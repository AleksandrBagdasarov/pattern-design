# Table Data Gateway

**Category:** Data access · **Also known as:** DAO (Data Access Object), table module gateway · **Related:** Row Data Gateway, Repository, Active Record, DTO, Query Object

## Intent
One object holds all the SQL for one table (or view): it finds, inserts, updates and deletes rows and returns plain records, with no domain logic.

## Problem
- SQL strings are scattered across handlers, scripts and jobs; changing a column means grepping the whole codebase.
- The code is SQL-centric by choice (performance-sensitive, DBA-owned schema, stored procedures) and an ORM would get in the way.
- Scenario: a FastAPI service over a reporting database uses hand-tuned SQL; `payments` queries are duplicated in three endpoints and a nightly job.

## Solution and structure
A gateway class per table exposes intention-named methods; each method runs one statement with bound parameters and returns rows as simple data (dicts, dataclasses, TypedDicts, Pydantic models). Business logic stays in the caller (often a Transaction Script or Service Layer).

```
Service / script ──> PaymentsGateway.find_by_customer(id) ──> SELECT ... FROM payments
                 ──> PaymentsGateway.insert(...)          ──> INSERT INTO payments
                         returns PaymentRecord (plain data)
```
Difference from Repository: a gateway is **table-shaped** and returns records; a repository is **aggregate-shaped** and returns domain objects.

## When to use
- SQL-first codebases, raw drivers (`asyncpg`, `psycopg`, `pg`), stored procedures or views.
- Thin services and data pipelines with little domain logic.
- You want all SQL for a table in one reviewable, testable file.
- Reporting/read side of CQRS where rows go straight to DTOs.

## When NOT to use
- A rich Domain Model with aggregates spanning tables: use Repository + Data Mapper.
- An ORM is already in use: its query API on the model is the gateway; don't wrap it again.
- One-off scripts: inline SQL is fine.

## Example — Python
`asyncpg` with a frozen dataclass record; all `payments` SQL in one place.

```python
from dataclasses import dataclass
from datetime import datetime
from decimal import Decimal
import asyncpg

@dataclass(frozen=True, slots=True)
class PaymentRecord:
    id: int
    customer_id: int
    amount: Decimal
    status: str
    created_at: datetime

class PaymentsGateway:
    def __init__(self, pool: asyncpg.Pool) -> None:
        self._pool = pool

    async def find(self, payment_id: int) -> PaymentRecord | None:
        row = await self._pool.fetchrow(
            "SELECT id, customer_id, amount, status, created_at FROM payments WHERE id = $1",
            payment_id)
        return PaymentRecord(**row) if row else None

    async def find_by_customer(self, customer_id: int, limit: int = 50) -> list[PaymentRecord]:
        rows = await self._pool.fetch(
            """SELECT id, customer_id, amount, status, created_at FROM payments
               WHERE customer_id = $1 ORDER BY created_at DESC, id DESC LIMIT $2""",
            customer_id, limit)
        return [PaymentRecord(**r) for r in rows]

    async def insert(self, customer_id: int, amount: Decimal) -> int:
        return await self._pool.fetchval(
            "INSERT INTO payments (customer_id, amount, status) VALUES ($1, $2, 'pending') RETURNING id",
            customer_id, amount)

    async def mark_settled(self, payment_id: int) -> bool:
        result = await self._pool.execute(
            "UPDATE payments SET status = 'settled' WHERE id = $1 AND status = 'pending'",
            payment_id)
        return result == "UPDATE 1"
```
For transactions spanning several gateways, pass a connection (`async with pool.acquire() as conn, conn.transaction():`) instead of the pool.

## Example — TypeScript
Node with `pg`; the gateway returns typed records.

```ts
import type { Pool, PoolClient } from 'pg';

export interface PaymentRecord {
  id: number; customerId: number; amount: string; status: 'pending' | 'settled'; createdAt: Date;
}
type Db = Pool | PoolClient;   // accept a client to join an outer transaction

const COLS = `id, customer_id AS "customerId", amount, status, created_at AS "createdAt"`;

export class PaymentsGateway {
  constructor(private readonly db: Db) {}

  async find(id: number): Promise<PaymentRecord | undefined> {
    const { rows } = await this.db.query<PaymentRecord>(
      `SELECT ${COLS} FROM payments WHERE id = $1`, [id]);
    return rows[0];
  }

  async findByCustomer(customerId: number, limit = 50): Promise<PaymentRecord[]> {
    const { rows } = await this.db.query<PaymentRecord>(
      `SELECT ${COLS} FROM payments WHERE customer_id = $1
       ORDER BY created_at DESC, id DESC LIMIT $2`, [customerId, limit]);
    return rows;
  }

  async markSettled(id: number): Promise<boolean> {
    const r = await this.db.query(
      `UPDATE payments SET status = 'settled' WHERE id = $1 AND status = 'pending'`, [id]);
    return r.rowCount === 1;
  }
}
```
Note: `pg` returns `NUMERIC` as a string by default to avoid float precision loss; keep it a string or use a decimal library.

## In the wild
- Java DAOs, Spring `JdbcTemplate`-based DAOs, MyBatis mappers.
- Python: `aiosql`/`PugSQL` (SQL files become gateway methods), `databases`, plain `asyncpg`.
- Node: `pgtyped`, Kysely/Knex query modules, `sqlc` (Go, also generates TS/Python) — gateways generated from SQL.

## Trade-offs
| Pros | Cons |
|---|---|
| All SQL for a table in one place; easy to review and tune | Logic tends to accumulate in callers (Transaction Script sprawl) |
| No ORM magic; predictable queries | Manual mapping, manual joins across tables |
| Easy to fake per method in tests | Multi-table operations don't fit one gateway |
| Works with views and stored procedures | Schema changes still ripple through record types |

## Pitfalls
- **String-formatting values into SQL.** Fix: always bound parameters; only whitelisted identifiers formatted in.
- **`SELECT *`** returning columns your record type doesn't expect after a migration. Fix: explicit column lists.
- **Each method grabbing its own connection** so multi-step writes aren't atomic. Fix: accept a connection/transaction from the caller.
- **Gateway grows business rules** (`if amount > limit`). Fix: keep rules in a service or domain object.
- **N gateways called in a loop** (N+1). Fix: add a batch method (`find_many(ids)` with `= ANY($1)`).

## Refactoring path
- **Introduce it:** grep for SQL touching one table; move each statement into a named method on `XGateway`; replace call sites; add tests per method against a real test database.
- **Remove it / simplify:** if you adopt an ORM, map the table and replace gateway methods with ORM queries or a Repository; delete the gateway once no callers remain.

## Related patterns
- [Row Data Gateway](../data-access/row-data-gateway.md) — one object per row instead of per table.
- [Repository](../data-access/repository.md) — aggregate-oriented, returns domain objects.
- [Active Record](../data-access/active-record.md) — gateway plus domain logic on the row object.
- [DTO](../enterprise/dto.md) — the plain records a gateway returns.
- [Query Object](../data-access/query-object.md) — for dynamic filters on a gateway's table.
