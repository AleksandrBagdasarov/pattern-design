# Object Mother

**Category:** Creational (testing) · **Also known as:** Test Fixture Factory, Example Objects · **Related:** Fluent Builder, Builder, Prototype, Fake Object, Arrange-Act-Assert, Data-Driven Testing

## Intent
Provide a central set of named factory methods that return canonical, valid domain objects for tests ("a paid order", "a suspended user"), so tests stop hand-building the same objects and read in domain terms.

## Problem
- Every test builds an `Order` with eight fields by hand; when a required field is added, 200 tests break.
- Tests are hard to read: the reader cannot tell which of the eight fields matters for this test.
- Inconsistent fixtures: one test's "valid user" has no email and passes only because validation was skipped.
- Scenario: a payments service adds a mandatory `currency` to `Payment`. A sweeping change across 140 test files follows. With an Object Mother, one function changes.

## Solution and structure
- **Object Mother**: a module or class of functions, each returning a ready object with a *domain name*: `paid_order()`, `refunded_payment()`, `admin_user()`.
- Functions accept keyword overrides for the fields a test cares about, and default everything else to valid values.
- Unique values (IDs, emails) are generated per call to avoid collisions in DB-backed tests.
- Classic Object Mother returns finished objects; the modern variant combines it with a Test Data Builder: the mother returns a builder preset to a canonical state, the test tweaks it.

```
test ──► mothers.paid_order(currency="USD") ──► Order(valid, defaults + override)
                    │
                    └─ built on: OrderBuilder / factory_boy / polyfactory
```

## When to use
- Many tests need the same kinds of domain objects in a few canonical states.
- Domain objects have many required fields and invariants.
- You want test names and arrange blocks to speak the domain language.

## When NOT to use
- Small codebase with a handful of tests: inline construction is more explicit.
- Objects have one or two fields; a constructor call is already clear.
- When the mother grows dozens of near-duplicate methods (`paid_order_in_usd_with_two_items_shipped_to_de`). Switch to a builder with overrides.
- As a hidden way to share mutable fixtures between tests. Each call must return a fresh object.

## Example — Python
Plain functions in a `tests/mothers.py` module, with keyword overrides and unique IDs. pytest fixtures can wrap these when DB persistence is needed.

```python
# tests/mothers.py
from dataclasses import replace
from datetime import UTC, datetime
from decimal import Decimal
from itertools import count
from app.domain import Customer, Order, OrderLine, OrderStatus

_ids = count(1)

def a_customer(**overrides: object) -> Customer:
    n = next(_ids)
    base = Customer(id=f"cus-{n}", email=f"customer{n}@example.test", country="DE", is_blocked=False)
    return replace(base, **overrides)

def a_line(sku: str = "SKU-1", qty: int = 1, price: str = "19.99") -> OrderLine:
    return OrderLine(sku=sku, qty=qty, unit_price=Decimal(price))

def a_new_order(**overrides: object) -> Order:
    base = Order(
        id=f"ord-{next(_ids)}",
        customer=a_customer(),
        lines=(a_line(),),
        currency="EUR",
        status=OrderStatus.NEW,
        created_at=datetime(2026, 1, 15, 10, 0, tzinfo=UTC),
    )
    return replace(base, **overrides)

def a_paid_order(**overrides: object) -> Order:
    return a_new_order(status=OrderStatus.PAID, **overrides)

def an_order_from_blocked_customer() -> Order:
    return a_new_order(customer=a_customer(is_blocked=True))

# tests/test_refunds.py
def test_cannot_refund_unpaid_order() -> None:
    order = a_new_order()                     # only "unpaid" matters here; the rest is valid noise
    assert refund_policy.can_refund(order) is False
```

## Example — TypeScript
Vitest with an Angular service test: the mother returns plain objects matching the API DTOs, with `Partial` overrides.

```typescript
// testing/mothers.ts
import type { OrderDto, UserDto } from '../app/api/types';

let seq = 0;
const nextId = (p: string) => `${p}-${++seq}`;

export const aUser = (o: Partial<UserDto> = {}): UserDto => ({
  id: nextId('usr'), email: `user${seq}@example.test`, roles: ['customer'], active: true, ...o,
});
export const anAdmin = (o: Partial<UserDto> = {}): UserDto => aUser({ roles: ['admin'], ...o });

export const anOrder = (o: Partial<OrderDto> = {}): OrderDto => ({
  id: nextId('ord'), userId: aUser().id, status: 'new', currency: 'EUR',
  lines: [{ sku: 'SKU-1', qty: 1, unitPrice: 19.99 }], ...o,
});
export const aPaidOrder = (o: Partial<OrderDto> = {}) => anOrder({ status: 'paid', ...o });

// orders.store.spec.ts
import { TestBed } from '@angular/core/testing';
import { provideHttpClient } from '@angular/common/http';
import { provideHttpClientTesting, HttpTestingController } from '@angular/common/http/testing';
import { OrdersStore } from './orders.store';
import { aPaidOrder, anOrder } from '../../testing/mothers';

it('shows only paid orders in the revenue total', () => {
  TestBed.configureTestingModule({ providers: [provideHttpClient(), provideHttpClientTesting()] });
  const store = TestBed.inject(OrdersStore);
  store.load();
  TestBed.inject(HttpTestingController).expectOne('/api/orders')
    .flush([aPaidOrder({ lines: [{ sku: 'A', qty: 2, unitPrice: 10 }] }), anOrder()]);
  expect(store.revenue()).toBe(20);
});
```

## In the wild
- Python: factory_boy (`Factory`, `SubFactory`, `Sequence`, traits), polyfactory (from Pydantic models/dataclasses), model_bakery for Django; pytest fixtures wrapping mothers (`@pytest.fixture def paid_order(): ...`).
- JS/TS: fishery, `@faker-js/faker`, `@mswjs/data`; Rails' FactoryBot is the origin of most of these APIs.
- The term comes from ThoughtWorks (Schuh & Punke, 2001); Nat Pryce later proposed Test Data Builders to fix mothers that grow too many methods.

## Trade-offs
| Pros | Cons |
|---|---|
| One place to change when the domain model changes | Hidden defaults: a test may pass because of a value it doesn't show |
| Tests read in domain language | Mother bloat: many near-duplicate methods |
| Fixtures are valid by construction | Coupling: many tests depend on one module |
| Consistent unique IDs avoid collisions | Shared mutable instances cause flaky tests |

## Pitfalls
- **Tests relying on implicit defaults** ("currency is EUR" without saying so). Fix: pass every value the assertion depends on explicitly, even if it equals the default.
- **Returning shared instances** (module-level constants mutated by tests). Fix: every call builds a fresh object; freeze domain objects.
- **Mother bypasses domain validation** (constructs invalid states the real code can't reach). Fix: build through the same constructors/factories as production code.
- **Explosion of named variants.** Fix: keep 5–10 canonical states; use overrides or a fluent builder for the rest.
- **Randomized data (faker) without a seed** creates unreproducible failures. Fix: seed per test run and print the seed.
- **DB-persisting mothers** used in unit tests. Fix: separate `build_*` (in-memory) from `create_*` (persisted) like factory_boy's `build()`/`create()`.

## Refactoring path
- **Introduce it:** grep tests for repeated constructor calls of the same domain type. Extract a function with valid defaults; replace call sites file by file; name variants by business state; add overrides as needed.
- **Remove it / simplify:** when the mother has dozens of variants, convert it into a single fluent Test Data Builder plus 3–5 named starting points. If only a few tests use it, inline the objects.

## Related patterns
- [Fluent Builder](fluent-builder.md) — Test Data Builders complement or replace a bloated mother.
- [Builder](builder.md) — the mother may sit on top of the production builder to keep fixtures valid.
- [Prototype](prototype.md) — mothers often copy a canonical object with overrides (`replace`, spread).
- [Fake Object](../testing/fake-object.md) — fakes hold the objects mothers create (in-memory repositories).
- [Arrange-Act-Assert](../testing/arrange-act-assert.md) — mothers keep the Arrange block short and intention-revealing.
- [Data-Driven Testing](../testing/data-driven-testing.md) — parametrized cases often use mother calls as inputs.
