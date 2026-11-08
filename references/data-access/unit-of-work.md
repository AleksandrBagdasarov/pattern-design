# Unit of Work

**Category:** Data access · **Also known as:** — (SQLAlchemy `Session`, EF `DbContext`) · **Related:** Repository, Data Mapper, Identity Map, Service Layer, Transactional Outbox

## Intent
Track every object changed during a business operation and write all the changes in one atomic transaction, or none of them.

## Problem
- A use case touches several objects (debit account, credit account, write ledger entry) and each repository call commits on its own; a crash in the middle leaves the data inconsistent.
- Services call `session.commit()` in random places; nobody knows where a transaction starts or ends.
- Scenario: `transfer()` debits account A, then the credit to B fails a constraint, but A's debit is already committed.
- Repeated loads of the same row in one request return different Python objects that overwrite each other's changes.

## Solution and structure
The Unit of Work (UoW) is opened at the start of a use case, hands out repositories that share its connection/session, records new, dirty and deleted objects, and on `commit()` flushes them in the correct order inside one transaction. Leaving the block without committing rolls back.

```
service ──with uow:──> uow.orders / uow.accounts (repositories on the same session)
                       modify domain objects
                       uow.commit()  ──> INSERT/UPDATE/DELETE in one transaction
         exit without commit ──> rollback
```
A real UoW usually includes an **Identity Map**: one object per primary key per unit, so two loads return the same instance.

## When to use
- A use case changes more than one row/object and must be atomic.
- You use Repositories and need a single owner of the transaction boundary.
- You want service code to say "commit" once, explicitly, at the end of the use case.
- You want to publish domain events only after a successful commit.

## When NOT to use
- You already have one: SQLAlchemy `Session`, Django `transaction.atomic()`, EF `DbContext`, TypeORM `EntityManager.transaction`, Prisma `$transaction`. Wrap thinly or use it directly; do not reimplement change tracking.
- Single-statement writes: the DB's autocommit is the unit.
- Across services or databases: a UoW cannot span them. Use a Saga or a transactional outbox.
- Long-running user workflows (minutes): do not hold a transaction open; use optimistic locking and separate short transactions.

## Example — Python
An explicit UoW over SQLAlchemy as a context manager. Repositories never commit.

```python
from collections.abc import Callable
from typing import Self
from uuid import UUID
from sqlalchemy.orm import Session
from app.repositories import SqlAccountRepository, SqlOrderRepository
from app.events import publish

class SqlUnitOfWork:
    def __init__(self, session_factory: Callable[[], Session]) -> None:
        self._factory = session_factory

    def __enter__(self) -> Self:
        self.session = self._factory()
        self.orders = SqlOrderRepository(self.session)
        self.accounts = SqlAccountRepository(self.session)
        return self

    def __exit__(self, *exc: object) -> None:
        self.session.rollback()        # no-op after a successful commit
        self.session.close()

    def commit(self) -> None:
        events = [e for o in self.session.identity_map.values()
                  for e in getattr(o, "events", [])]
        self.session.commit()
        for event in events:           # only after commit succeeded
            publish(event)

def pay_order(uow: SqlUnitOfWork, order_id: UUID, account_id: UUID) -> None:
    with uow:
        order = uow.orders.get(order_id)
        account = uow.accounts.get(account_id)
        if order is None or account is None:
            raise LookupError("not found")
        account.withdraw(order.total())
        order.pay()
        uow.commit()                   # both changes or neither
```
For in-process `publish()` this is fine; if publishing to a broker, write events to an outbox table inside the same commit instead (see Message Queue). Async variant: `AsyncSession` with `async with`.

## Example — TypeScript
TypeORM's `DataSource.transaction` already is a UoW; the callback scope is the unit.

```ts
import { DataSource, EntityManager } from 'typeorm';
import { Account } from './account.entity';
import { Order } from './order.entity';

export class PaymentService {
  constructor(private readonly db: DataSource) {}

  async payOrder(orderId: string, accountId: string): Promise<void> {
    await this.db.transaction(async (em: EntityManager) => {
      const order = await em.findOneByOrFail(Order, { id: orderId });
      const account = await em.findOneOrFail(Account, {
        where: { id: accountId },
        lock: { mode: 'pessimistic_write' },
      });
      account.withdraw(order.totalCents);
      order.pay();
      await em.save([account, order]);   // committed when the callback resolves
    });                                   // rolled back if it throws
  }
}
```
Prisma equivalent: `await prisma.$transaction(async (tx) => { ... })`. Check your ORM's docs for isolation level and lock options; they differ by version and driver.

## In the wild
- SQLAlchemy `Session` (UoW + Identity Map); `with Session.begin():` commits or rolls back.
- Django `transaction.atomic()` (transaction boundary, no change tracking — Active Record saves immediately).
- Entity Framework `DbContext.SaveChanges()`, Hibernate `Session`, MikroORM `em.flush()`.
- FastAPI: a `Depends` generator that yields a session/UoW and closes it after the response.

## Trade-offs
| Pros | Cons |
|---|---|
| All-or-nothing writes per use case | Hidden flushes can surprise (autoflush before queries) |
| One obvious transaction boundary | Long units hold locks and connections |
| Identity Map prevents duplicate in-memory copies | Change tracking costs memory on large batches |
| Natural hook for post-commit events | Cannot span services, queues or multiple DBs |

## Pitfalls
- **Repositories that commit:** destroy atomicity. Fix: only the UoW commits.
- **Publishing events/sending emails before commit:** side effects for rolled-back data. Fix: after commit, or outbox.
- **Network calls inside the transaction** (payment API while holding row locks). Fix: call out before or after; record intent in the DB.
- **One session per application (global):** shared state across requests and threads. Fix: one UoW per request/use case.
- **Bulk loads through a tracking UoW** (100k rows): memory blow-up. Fix: bulk statements (`insert().values()`), `yield_per`, or chunked units.
- **Forgetting async sessions are not concurrency-safe:** do not share one `AsyncSession` across `asyncio.gather` tasks.

## Refactoring path
- **Introduce it:** find all `commit()` calls; remove them from repositories and helpers; open one session per request/use case; commit once at the end of each service method; then wrap that in an explicit `UnitOfWork` if you want an abstract, fakeable boundary.
- **Remove it / simplify:** if your custom UoW only wraps `Session` with no events or fakes, delete it and use `with Session.begin():` or `transaction.atomic()` directly.

## Related patterns
- [Repository](../data-access/repository.md) — UoW hands out repositories sharing its transaction.
- [Data Mapper](../data-access/data-mapper.md) — the UoW tells the mapper what to insert/update/delete.
- [Domain Model](../data-access/domain-model.md) — one transaction should modify one aggregate.
- [Service Layer](../enterprise/service-layer.md) — each service method is typically one unit of work.
- [Message Queue](../enterprise/message-queue.md) — transactional outbox extends the UoW to messaging.
- [Saga](../cloud/saga.md) — the replacement when the "transaction" spans services.
