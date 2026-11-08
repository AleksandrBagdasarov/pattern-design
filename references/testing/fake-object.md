# Fake Object

**Category:** Testing · **Also known as:** Fake, in-memory implementation · **Related:** Test Stub, Test Spy, Mock Object, Repository, Hexagonal Architecture

## Intent
Provide a real, working but simplified implementation of a dependency (in-memory repository, SQLite instead of Postgres, local filesystem instead of S3) so tests exercise genuine behavior without the cost of the production component.

## Problem
- Tests stub `repo.get()` and `repo.save()` separately, and the stubs do not remember what was saved, so multi-step flows (create, then fetch, then update) cannot be tested honestly.
- Every test has a different elaborate `Mock` setup for the same dependency; the setups disagree with each other and with production.
- The real dependency (database, queue, object store) makes the unit suite slow or requires Docker.

Scenario: an order service creates an order, reserves stock, and later cancels it. With stubs you hand-craft each return value. With a `FakeOrderRepository` backed by a dict, the test calls the service three times and checks the final state, exactly like production.

## Solution and structure
```
             ┌──────────── OrderRepository (Protocol / port) ─────────────┐
             │                                                           │
   SqlOrderRepository (prod)                           FakeOrderRepository (tests)
   SQLAlchemy + Postgres                               dict[int, Order]
```
- **Port**: the interface the SUT depends on.
- **Fake**: a full implementation of the port with shortcuts (memory, no durability, no concurrency control).
- **Contract tests**: the same test suite runs against both the fake and the real adapter, so they cannot drift.

## When to use
- Ports-and-adapters / hexagonal code: repositories, message buses, email senders, blob stores, clocks.
- Stateful interactions across several calls.
- Many tests share a dependency; one fake replaces dozens of ad-hoc stubs.
- You want state-based assertions ("the outbox contains one email") instead of interaction-based ones.

## When NOT to use
- The fake would need to reimplement complex semantics (SQL query planner, transactions, Kafka partitions). Use the real thing in a container (Testcontainers) for those tests.
- Only one test needs one canned answer: a stub is simpler.
- The behavior under test *is* the adapter (the SQL query itself): test against the real database.
- SQLite as a fake for Postgres when the code uses Postgres-specific features (JSONB, `ON CONFLICT` nuances, array types): you get false greens.

## Example — Python
```python
from dataclasses import dataclass, replace
from typing import Protocol
import pytest

@dataclass(frozen=True)
class Order:
    id: int
    customer_id: int
    status: str = "new"

class OrderRepository(Protocol):
    def add(self, order: Order) -> None: ...
    def get(self, order_id: int) -> Order | None: ...
    def update(self, order: Order) -> None: ...

class FakeOrderRepository:
    def __init__(self) -> None:
        self._rows: dict[int, Order] = {}
    def add(self, order: Order) -> None:
        if order.id in self._rows:
            raise ValueError("duplicate id")   # mimic the unique constraint
        self._rows[order.id] = order
    def get(self, order_id: int) -> Order | None:
        return self._rows.get(order_id)
    def update(self, order: Order) -> None:
        self._rows[order.id] = order

def cancel_order(repo: OrderRepository, order_id: int) -> Order:
    order = repo.get(order_id)
    if order is None or order.status == "shipped":
        raise ValueError("cannot cancel")
    cancelled = replace(order, status="cancelled")
    repo.update(cancelled)
    return cancelled

@pytest.fixture
def repo() -> FakeOrderRepository:
    return FakeOrderRepository()

def test_cancel_persists_status(repo: FakeOrderRepository) -> None:
    repo.add(Order(1, customer_id=7))
    cancel_order(repo, 1)
    assert repo.get(1) == Order(1, 7, "cancelled")
```
For contract tests, parametrize a fixture over `[FakeOrderRepository, SqlOrderRepository]` and run the same repository tests against both.

## Example — TypeScript
```ts
import { Injectable, signal } from '@angular/core';
import { Observable, of, throwError } from 'rxjs';
import { TestBed } from '@angular/core/testing';
import { expect, it } from 'vitest';
import { TasksApi, Task, TaskListStore } from './tasks';

@Injectable()
export class FakeTasksApi implements TasksApi {
  private readonly tasks = signal<Task[]>([]);
  private nextId = 1;
  list(): Observable<Task[]> { return of(this.tasks()); }
  create(title: string): Observable<Task> {
    if (!title.trim()) return throwError(() => new Error('422'));  // mimic server validation
    const task = { id: this.nextId++, title, done: false };
    this.tasks.update((t) => [...t, task]);
    return of(task);
  }
}

it('created task appears after reload', async () => {
  TestBed.configureTestingModule({
    providers: [TaskListStore, { provide: TasksApi, useClass: FakeTasksApi }],
  });
  const store = TestBed.inject(TaskListStore);
  await store.create('Write tests');
  await store.reload();
  expect(store.tasks().map((t) => t.title)).toEqual(['Write tests']);
});
```

## In the wild
- SQLAlchemy with SQLite in-memory (`sqlite://`) for simple schemas; `fakeredis`; `moto` (fake AWS services); `pyfakefs` (fake filesystem); `mongomock`.
- Django's `locmem` email backend and cache backend; Starlette/FastAPI `TestClient` with an in-memory repository via `dependency_overrides`.
- Angular `HttpClientTestingModule` is a stub/spy hybrid; in-memory web API (`angular-in-memory-web-api`) is a fake backend.
- Testcontainers is the alternative when a fake would be too inaccurate.

## Trade-offs
| Pros | Cons |
|---|---|
| Tests real multi-step behavior, state-based assertions | Is code that must be maintained and can have bugs |
| One fake replaces many fragile mocks; refactor-friendly | Can drift from production semantics unless contract-tested |
| Fast, deterministic, no infrastructure | Does not catch adapter-level bugs (SQL, serialization) |
| Doubles as a dev/demo backend | Temptation to grow it into a second production system |

## Pitfalls
- **Fake more lenient than prod:** no unique constraints, no validation, no ordering guarantees. Mirror the constraints the domain relies on, and run shared contract tests.
- **Shared mutable fake across tests:** state leaks. Create a new fake per test (function-scoped fixture, fresh TestBed).
- **Returning internal references:** callers mutate the fake's stored objects. Store immutable values or copy on read.
- **Faking what you do not own** with guessed semantics (a fake Stripe). Wrap the third party in your own adapter first and fake *your* adapter.

## Refactoring path
- **Introduce it:** spot repeated `Mock()` setups for the same dependency. Extract a Protocol/interface, write the fake, migrate tests one by one, then add contract tests.
- **Remove it / simplify:** if the fake keeps diverging, replace its usages with Testcontainers-based integration tests for those paths, and keep stubs for the few remaining unit tests.

## Related patterns
- [Test Stub](../testing/test-stub.md) — canned answers; no memory between calls.
- [Test Spy](../testing/test-spy.md) — interaction recording; fakes favour state checks instead.
- [Mock Object](../testing/mock-object.md) — the opposite end: verifies calls, not state.
- [Repository](../data-access/repository.md) — the port most often faked.
- [Hexagonal](../architectural/hexagonal.md) — ports make fakes possible.
