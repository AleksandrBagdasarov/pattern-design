# BDD (Behavior-Driven Development)

**Category:** Testing · **Also known as:** Specification by Example, Given/When/Then, Gherkin scenarios · **Related:** Arrange-Act-Assert, Data-Driven Testing, Fake Object

## Intent
Describe system behavior as concrete examples in a shared, structured language (Given / When / Then) agreed with product and QA, and turn those examples into executable tests.

## Problem
- Requirements live in tickets and Slack threads; developers, QA and product interpret "discount for loyal customers" differently.
- Acceptance tests are written after the fact by QA, in a separate tool, and drift from the code.
- Test names describe methods (`test_apply_discount_2`) rather than behavior, so nobody outside the team can read them.

Scenario: "Gold members get free shipping on orders over €50" — what about exactly €50, or after a coupon is applied? A Three Amigos session (dev, QA, product) writes three scenarios with concrete numbers; those become the acceptance tests.

## Solution and structure
```
Discovery (examples workshop) ─▶ Formulation (Gherkin .feature) ─▶ Automation (step definitions ─▶ app)
```
- **Feature file**: `Feature`, `Scenario`, `Given` (context), `When` (one event), `Then` (observable outcome), `Scenario Outline` + `Examples` for tables.
- **Step definitions**: code bound to step text by pattern; they call the application through its public API (service layer, HTTP, UI).
- **World/context**: per-scenario state shared between steps (pytest fixtures in pytest-bdd).

BDD is first a collaboration practice. Gherkin tooling is optional; the Given/When/Then structure can be used inside ordinary pytest or Vitest tests.

## When to use
- Business rules with stakeholder involvement: pricing, eligibility, workflows, compliance.
- Acceptance tests at service or API level that non-developers should read and review.
- Teams where misunderstood requirements, not code bugs, cause most rework.

## When NOT to use
- No one outside the dev team will read the feature files: plain tests with descriptive names and Given/When/Then comments give 90% of the value without the step-glue layer.
- Technical/unit-level behavior (a parser, a cache): Gherkin adds indirection and nothing else.
- UI-click-level scenarios ("When I click the blue button") — brittle and not behavior. Express intent ("When I submit the order") and keep UI details in step code.
- Heavy step reuse across hundreds of scenarios with regex parameters: becomes a hard-to-refactor DSL.

## Example — Python
```gherkin
# tests/features/shipping.feature
Feature: Free shipping for gold members
  Scenario Outline: Shipping fee by tier and subtotal
    Given a <tier> customer
    And a cart with subtotal <subtotal> EUR
    When the customer checks out
    Then the shipping fee is <fee> EUR

    Examples:
      | tier     | subtotal | fee  |
      | gold     | 50.01    | 0.00 |
      | gold     | 50.00    | 4.90 |
      | standard | 80.00    | 4.90 |
```
```python
# tests/test_shipping_bdd.py
from decimal import Decimal
import pytest
from pytest_bdd import scenarios, given, when, then, parsers
from shop.checkout import Cart, Customer, checkout

scenarios("features/shipping.feature")

@given(parsers.parse("a {tier} customer"), target_fixture="customer")
def customer(tier: str) -> Customer:
    return Customer(id=1, tier=tier)

@given(parsers.parse("a cart with subtotal {subtotal} EUR"), target_fixture="cart")
def cart(subtotal: str) -> Cart:
    return Cart(subtotal=Decimal(subtotal))

@when("the customer checks out", target_fixture="receipt")
def do_checkout(customer: Customer, cart: Cart):
    return checkout(customer, cart)

@then(parsers.parse("the shipping fee is {fee} EUR"))
def shipping_fee(receipt, fee: str) -> None:
    assert receipt.shipping == Decimal(fee)
```
Steps call the application service, not the database or UI. Without pytest-bdd, a plain parametrized test with `# Given / # When / # Then` comments expresses the same spec.

## Example — TypeScript
```ts
import { describe, it, expect, beforeEach } from 'vitest';
import { TestBed } from '@angular/core/testing';
import { CheckoutStore } from './checkout.store';
import { CustomerApi } from './customer.api';
import { FakeCustomerApi } from './testing/fake-customer.api';

describe('Feature: free shipping for gold members', () => {
  let store: CheckoutStore;
  let customers: FakeCustomerApi;

  beforeEach(() => {
    TestBed.configureTestingModule({
      providers: [CheckoutStore, { provide: CustomerApi, useClass: FakeCustomerApi }],
    });
    store = TestBed.inject(CheckoutStore);
    customers = TestBed.inject(CustomerApi) as FakeCustomerApi;
  });

  describe('Scenario: gold member above the threshold', () => {
    it('Given a gold customer with a 50.01 EUR cart, When they check out, Then shipping is free', async () => {
      customers.seed({ id: 1, tier: 'gold' });       // Given
      store.setCart({ customerId: 1, subtotal: 50.01 });

      await store.checkout();                          // When

      expect(store.receipt()?.shipping).toBe(0);       // Then
    });
  });
});
```
For Gherkin in JS, `@cucumber/cucumber` is the standard runner; `quickpickle` and `vitest-cucumber` bring `.feature` files to Vitest (check their maintenance status before adopting). Playwright + `playwright-bdd` is common for end-to-end scenarios.

## In the wild
- Python: `pytest-bdd`, `behave`; Robot Framework (keyword-driven, similar intent).
- JS/TS: Cucumber.js, `playwright-bdd`, Cypress with Cucumber preprocessor; Jasmine/Mocha's `describe`/`it` naming came from BDD (RSpec).
- SpecFlow/Reqnroll (.NET), Cucumber-JVM; Gojko Adzic, *Specification by Example*.

## Trade-offs
| Pros | Cons |
|---|---|
| Shared language with product/QA; fewer misunderstood requirements | Step-definition glue layer to maintain |
| Living documentation that is executed | Feature files rot if stakeholders stop reading them |
| Concrete examples surface edge cases early | Regex/parse-based step matching is weakly typed and hard to refactor |
| Scenario Outline tables are natural for rules | Slow if scenarios run through UI by default |

## Pitfalls
- **Imperative UI scripts in Gherkin** ("click", "type", "wait 2s"). Write declarative business steps; push mechanics into step code or page objects.
- **Multiple When steps per scenario**: tests several behaviors at once. One When per scenario.
- **Gherkin without collaboration**: developers writing feature files alone is just an expensive test format.
- **Shared mutable world** between scenarios: reset per scenario (fixtures, fresh TestBed).
- **Then steps that check internals** (DB rows) instead of observable outcomes.

## Refactoring path
- **Introduce it:** start with Given/When/Then structure and behavior-focused names in existing tests; add Gherkin only for the rules stakeholders actually review; run discovery workshops before writing code.
- **Remove it / simplify:** if nobody outside dev reads `.feature` files, inline step bodies into ordinary parametrized tests with descriptive names and delete the glue layer.

## Related patterns
- [Arrange-Act-Assert](../testing/arrange-act-assert.md) — Given/When/Then maps one-to-one to Arrange/Act/Assert.
- [Data-Driven Testing](../testing/data-driven-testing.md) — Scenario Outline + Examples is a parameterized test.
- [Fake Object](../testing/fake-object.md) — fast acceptance tests run against fakes behind the service layer.
