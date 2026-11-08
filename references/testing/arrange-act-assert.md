# Arrange-Act-Assert

**Category:** Testing · **Also known as:** AAA, Given-When-Then, Four-Phase Test (Setup-Exercise-Verify-Teardown) · **Related:** BDD, Test Stub, Test Spy, Fake Object, Data-Driven Testing

## Intent
Structure every test as three visually separate phases — set up the world, perform exactly one action, check the outcome — so each test reads as a single, obvious example of one behavior.

## Problem
- Tests interleave setup, actions and assertions ("act, assert, act again, assert"), so a failure does not say which behavior broke.
- A single test checks five behaviors; its name is `test_order_flow`.
- Readers cannot find the line that is actually under test among 30 lines of setup.
- Assertions are hidden inside helper functions or fixtures.

Scenario: a 60-line `test_cart` creates a user, adds items, removes one, applies a coupon, checks totals after each step. When it fails at line 48, nobody knows whether the coupon or the removal broke. Split into four AAA tests, each with one Act.

## Solution and structure
```
# Arrange   build SUT + inputs + doubles (fixtures, builders, Object Mother)
# Act       one call on the SUT — the behavior named in the test
# Assert    check result / state / recorded interactions
(# Teardown handled by fixtures / afterEach / context managers)
```
- **Arrange** may be large, but should be delegated to fixtures, factories and builders so only the details relevant to *this* test are visible.
- **Act** is usually one line. If you need two, ask whether you are testing two things.
- **Assert** can contain several assertions about the *same outcome* (status and body of one response).

## When to use
- Always, for unit and integration tests. It is the default shape.
- When reviewing tests: missing or blurred phases point to tests that test too much.

## When NOT to use
- Property-based tests: Arrange is replaced by generation; keep the Act/Assert separation but do not force comments.
- End-to-end journey tests where a multi-step flow *is* the behavior (signup → verify email → first login). Still keep each step's assertion next to its action, and keep these tests few.
- Don't add `# Arrange` comments to three-line tests where blank lines already make the phases obvious.

## Example — Python
```python
from decimal import Decimal
import pytest
from shop.cart import Cart, Coupon, CouponExpired
from tests.builders import a_product   # Test Data Builder

@pytest.fixture
def cart() -> Cart:
    return Cart(customer_id=7)

def test_percentage_coupon_reduces_total(cart: Cart) -> None:
    # Arrange
    cart.add(a_product(price=Decimal("40.00")), qty=2)
    coupon = Coupon(code="SAVE10", percent=10)

    # Act
    cart.apply(coupon)

    # Assert
    assert cart.total == Decimal("72.00")
    assert cart.applied_coupons == ["SAVE10"]

def test_expired_coupon_is_rejected(cart: Cart) -> None:
    cart.add(a_product(), qty=1)
    expired = Coupon(code="OLD", percent=10, expired=True)

    with pytest.raises(CouponExpired):     # Act + Assert for exceptions
        cart.apply(expired)

    assert cart.applied_coupons == []      # state unchanged
```
`pytest.raises` merges Act and Assert syntactically; keep only the single acting call inside the `with` block.

## Example — TypeScript
```ts
import { describe, it, expect, beforeEach } from 'vitest';
import { TestBed } from '@angular/core/testing';
import { provideHttpClient } from '@angular/common/http';
import { HttpTestingController, provideHttpClientTesting } from '@angular/common/http/testing';
import { TodoStore } from './todo.store';

describe('TodoStore', () => {
  let store: TodoStore;
  let http: HttpTestingController;

  beforeEach(() => {
    TestBed.configureTestingModule({
      providers: [TodoStore, provideHttpClient(), provideHttpClientTesting()],
    });
    store = TestBed.inject(TodoStore);
    http = TestBed.inject(HttpTestingController);
  });

  it('marks a todo as done optimistically', () => {
    // Arrange
    store.set([{ id: 1, title: 'Ship', done: false }]);

    // Act
    store.toggle(1);

    // Assert
    expect(store.todos()[0]?.done).toBe(true);
    http.expectOne({ method: 'PATCH', url: '/api/todos/1' });
  });
});
```

## In the wild
- Bill Wake's AAA (2001); Meszaros' Four-Phase Test in *xUnit Test Patterns*; Given/When/Then from BDD (Dan North).
- pytest fixtures and Vitest `beforeEach` are the framework-provided Arrange/Teardown; `yield` fixtures and context managers give Teardown.
- Linters: `eslint-plugin-jest`/`vitest` rules like `max-expects`, `no-conditional-in-test`; `pytest-aaa` / `flake8-aaa` enforce the layout in Python.

## Trade-offs
| Pros | Cons |
|---|---|
| One behavior per test; failures are precise | More, smaller tests (usually a benefit) |
| Readable by anyone; test name + Act line say it all | Repeated Arrange unless factored into fixtures/builders |
| Pushes setup into reusable fixtures and builders | Over-abstracted Arrange (fixture chains) hides relevant details |
| Makes "testing too much" visible | Rigid application to journey tests adds noise |

## Pitfalls
- **Multiple Acts** per test: split the test, or name it as a scenario and accept it is an integration test.
- **Mystery Guest:** relevant data hidden in a distant fixture or JSON file. Keep values the assertion depends on visible in Arrange (builders with explicit overrides).
- **Assertion roulette:** many unrelated asserts without messages. Group assertions about one outcome; split otherwise.
- **Logic in tests** (`if`, loops computing expected values): use Data-Driven Testing or literal expectations.
- **Act inside Arrange fixtures:** the behavior runs before the test body, making failures confusing. Fixtures arrange; the test acts.

## Refactoring path
- **Introduce it:** in a long test, mark each Act; split at every second Act into a new test with a behavior-named title; move common setup into fixtures/builders.
- **Remove it / simplify:** if Arrange became a deep fixture graph, inline the few values that matter back into the test and keep only generic plumbing in fixtures.

## Related patterns
- [BDD](../testing/bdd.md) — Given/When/Then is AAA in business language.
- [Test Stub](../testing/test-stub.md) — configured in Arrange.
- [Test Spy](../testing/test-spy.md) — inspected in Assert.
- [Fake Object](../testing/fake-object.md) — seeded in Arrange, inspected in Assert.
- [Data-Driven Testing](../testing/data-driven-testing.md) — each row is one AAA run.
- [Object Mother](../creational/object-mother.md) — keeps Arrange short and intention-revealing.
