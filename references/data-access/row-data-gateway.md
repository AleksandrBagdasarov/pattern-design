# Row Data Gateway

**Category:** Data access · **Also known as:** — · **Related:** Table Data Gateway, Active Record, Data Mapper, Repository

## Intent
An object that mirrors exactly one database row: it holds the row's fields and can insert, update and delete itself, but contains no domain logic.

## Problem
- You want object-style access to rows (`row.status = "x"; row.update()`) instead of passing dicts around, but you do not want domain logic coupled to persistence.
- Business logic lives elsewhere (Transaction Scripts or a separate domain layer) and needs a uniform way to read/write individual records.
- Scenario: a legacy billing script loads a `customer_accounts` row, changes the balance through a service function, then writes it back; each step currently hand-builds SQL.

## Solution and structure
Two parts:
- A **finder** (often static/class methods or a separate finder object) returns gateway instances for rows.
- The **gateway instance** has one field per column plus `insert()`, `update()`, `delete()`.

```
Finder.find(id) ──> AccountRow(id, balance, version)  ── row.update() ──> UPDATE ... WHERE id=? AND version=?
                          ▲
           service functions read/write fields, hold all logic
```
Compared with **Active Record**: identical shape, but Active Record also holds domain logic. When a Row Data Gateway starts gaining business methods, it has become Active Record — decide consciously.

## When to use
- Transaction-script code that wants typed row objects without an ORM.
- Wrapping legacy tables during a gradual migration to a domain model (the gateway becomes the persistence model a Data Mapper later maps from).
- Simple per-record edit flows where the table is the model.

## When NOT to use
- You already use an ORM: its mapped entities are row gateways (or more); don't hand-roll.
- Set-based operations (bulk updates, reports): use a Table Data Gateway with SQL statements.
- Rich domain logic: Domain Model + Data Mapper.

## Example — Python
`sqlite3`/DB-API style with optimistic locking via a `version` column.

```python
from __future__ import annotations
import sqlite3
from dataclasses import dataclass

class StaleRowError(Exception): ...

@dataclass
class AccountRow:
    id: int
    owner_email: str
    balance_cents: int
    version: int

    @classmethod
    def find(cls, conn: sqlite3.Connection, account_id: int) -> AccountRow | None:
        cur = conn.execute(
            "SELECT id, owner_email, balance_cents, version FROM accounts WHERE id = ?",
            (account_id,))
        row = cur.fetchone()
        return cls(*row) if row else None

    def insert(self, conn: sqlite3.Connection) -> None:
        cur = conn.execute(
            "INSERT INTO accounts (owner_email, balance_cents, version) VALUES (?, ?, 1)",
            (self.owner_email, self.balance_cents))
        self.id, self.version = cur.lastrowid, 1

    def update(self, conn: sqlite3.Connection) -> None:
        cur = conn.execute(
            """UPDATE accounts SET owner_email = ?, balance_cents = ?, version = version + 1
               WHERE id = ? AND version = ?""",
            (self.owner_email, self.balance_cents, self.id, self.version))
        if cur.rowcount != 1:
            raise StaleRowError(f"account {self.id} changed concurrently")
        self.version += 1

# logic lives outside the gateway (transaction script)
def deposit(conn: sqlite3.Connection, account_id: int, cents: int) -> None:
    if cents <= 0:
        raise ValueError("deposit must be positive")
    with conn:                                   # commits or rolls back
        acc = AccountRow.find(conn, account_id)
        if acc is None:
            raise LookupError(account_id)
        acc.balance_cents += cents
        acc.update(conn)
```

## Example — TypeScript
```ts
import type { Pool } from 'pg';

export class StaleRowError extends Error {}

export class AccountRow {
  private constructor(
    readonly id: number,
    public ownerEmail: string,
    public balanceCents: number,
    private version: number,
  ) {}

  static async find(db: Pool, id: number): Promise<AccountRow | undefined> {
    const { rows } = await db.query(
      'SELECT id, owner_email, balance_cents, version FROM accounts WHERE id = $1', [id]);
    const r = rows[0];
    return r && new AccountRow(r.id, r.owner_email, Number(r.balance_cents), r.version);
  }

  async update(db: Pool): Promise<void> {
    const res = await db.query(
      `UPDATE accounts SET owner_email = $1, balance_cents = $2, version = version + 1
       WHERE id = $3 AND version = $4`,
      [this.ownerEmail, this.balanceCents, this.id, this.version]);
    if (res.rowCount !== 1) throw new StaleRowError(`account ${this.id} changed concurrently`);
    this.version += 1;
  }
}
```

## In the wild
- Mapped ORM entities used without logic (SQLAlchemy declarative models as "persistence models" in a layered app).
- Java/.NET "entity bean"/POCO with a DAO per row; Rails models before logic is added.
- Most codebases use this implicitly via an ORM; hand-written row gateways are mainly in legacy or driver-only code.

## Trade-offs
| Pros | Cons |
|---|---|
| Typed row objects without an ORM | One query per row: poor for bulk work |
| Persistence separated from business logic | Drifts into Active Record as logic creeps in |
| Natural place for optimistic locking | Manual mapping and SQL per table |
| Easy stepping stone towards a Data Mapper | Relations/joins must be handled manually |

## Pitfalls
- **Lost updates** from read-modify-write without locking. Fix: version column (shown) or `SELECT ... FOR UPDATE`.
- **Loading rows in a loop** (N+1). Fix: a finder that returns many rows in one query.
- **Business rules sneaking in** (`row.can_withdraw()`). Fix: keep in the service, or consciously adopt Active Record.
- **Each `update()` auto-committing** so multi-row changes are not atomic. Fix: commit in the caller's transaction.

## Refactoring path
- **Introduce it:** wrap one table's hand-built SQL in a row class with `find/insert/update`; replace dict passing with the row object; add a version column if concurrent edits are possible.
- **Remove it / simplify:** replace with ORM-mapped models (same shape, less code), or with a Table Data Gateway when most operations are set-based.

## Related patterns
- [Table Data Gateway](../data-access/table-data-gateway.md) — one object per table; better for set-based operations.
- [Active Record](../data-access/active-record.md) — a row gateway plus domain logic.
- [Data Mapper](../data-access/data-mapper.md) — maps between row objects/tables and a separate domain model.
- [Repository](../data-access/repository.md) — aggregate-level access built on top of mappers or gateways.
