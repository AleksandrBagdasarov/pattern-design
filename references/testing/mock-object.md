# Mock Object

**Category:** Testing · **Also known as:** Mock (strict sense, Meszaros) · **Related:** Test Spy, Test Stub, Fake Object, Dependency Injection

## Intent
A test double pre-programmed with **expectations** about the calls it will receive, which verifies itself — failing when an unexpected call arrives or an expected one is missing. It tests the SUT's indirect outputs by behavior verification.

## Problem
- The only observable effect of the SUT is a conversation with a collaborator (charge a card, send an SMS, call a hardware API), and the protocol matters: exact arguments, exactly once, possibly order.
- A wrong call is expensive or irreversible in production (double charge), so the test must fail on *any* unexpected call, not just check the ones it thought of.

Scenario: `refund(order)` must call `gateway.refund(charge_id, amount)` exactly once and never `gateway.charge`. A strict mock that rejects unexpected calls catches the regression where someone "fixes" a negative amount by calling `charge`.

Terminology: libraries blur the words. `unittest.mock.Mock` and `vi.fn()` are spies by default (record, then the test asserts). They act as mocks when you configure them strictly: `create_autospec(..., spec_set=True)`, or explicit "expect this, reject everything else" setups (Sinon `mock().expects()`, `jest-mock-extended` `calledWith`, Mockito-style libraries).

## Solution and structure
```
Arrange: mock.expect(refund(c1, 500)).times(1)    ← expectations declared up front
Act:     sut.refund(order)  ──calls──▶ mock (checks each call as it arrives)
Assert:  mock.verify()                             ← fails on missing / extra calls
```
- **Mock** knows the expected calls and raises on deviations.
- **Test** sets expectations before Act and triggers verification after.

## When to use
- Interaction is the requirement and it crosses a boundary you own as a port (payment, messaging, hardware).
- Protocol correctness matters: once-only, ordering, nothing else called.
- Outside-in TDD ("London school"): discovering a collaborator's interface by mocking it before it exists.

## When NOT to use
- The effect can be observed as state (use a Fake) or return value (assert directly).
- Mocking types you do not own (`requests.Session`, `HttpClient`, an AWS SDK): you encode guesses about their behavior. Wrap them in your own adapter and mock or fake the adapter.
- Mocking value objects, data classes, or the SUT's own private helpers.
- When every test sets five expectations on internal calls: that is "mockist overkill"; tests break on any refactor. Use state-based tests.

## Example — Python
```python
from dataclasses import dataclass
from typing import Protocol
from unittest.mock import create_autospec, call
import pytest

class PaymentGateway(Protocol):
    def charge(self, customer_id: str, cents: int) -> str: ...
    def refund(self, charge_id: str, cents: int) -> None: ...

@dataclass(frozen=True)
class Order:
    id: int
    charge_id: str
    total_cents: int

def refund_order(order: Order, gateway: PaymentGateway, partial: int | None = None) -> None:
    amount = order.total_cents if partial is None else partial
    if not 0 < amount <= order.total_cents:
        raise ValueError("invalid refund amount")
    gateway.refund(order.charge_id, amount)

@pytest.fixture
def gateway() -> PaymentGateway:
    # spec_set: unknown attributes raise; autospec: wrong signatures raise
    return create_autospec(PaymentGateway, instance=True, spec_set=True)

def test_full_refund_calls_gateway_exactly_once(gateway) -> None:
    refund_order(Order(1, "ch_1", 500), gateway)
    assert gateway.mock_calls == [call.refund("ch_1", 500)]   # nothing else, e.g. no charge()

def test_invalid_amount_never_touches_gateway(gateway) -> None:
    with pytest.raises(ValueError):
        refund_order(Order(1, "ch_1", 500), gateway, partial=900)
    assert gateway.mock_calls == []
```
Comparing the full `mock_calls` list is how you get strict "no unexpected calls" behavior from `unittest.mock`, which otherwise records leniently.

## Example — TypeScript
```ts
import { describe, it, expect, vi, afterEach } from 'vitest';

interface PaymentGateway {
  charge(customerId: string, cents: number): Promise<string>;
  refund(chargeId: string, cents: number): Promise<void>;
}

async function refundOrder(order: { chargeId: string; totalCents: number }, gw: PaymentGateway) {
  await gw.refund(order.chargeId, order.totalCents);
}

function strictGateway() {
  const unexpected = (name: string) => vi.fn(() => { throw new Error(`unexpected call: ${name}`); });
  return {
    charge: unexpected('charge'),
    refund: vi.fn<PaymentGateway['refund']>().mockResolvedValue(undefined),
  } satisfies PaymentGateway;
}

describe('refundOrder', () => {
  afterEach(() => vi.restoreAllMocks());

  it('refunds exactly once and never charges', async () => {
    const gw = strictGateway();
    await refundOrder({ chargeId: 'ch_1', totalCents: 500 }, gw);
    expect(gw.refund).toHaveBeenCalledExactlyOnceWith('ch_1', 500);
    expect(gw.charge).not.toHaveBeenCalled();
  });
});
```
`toHaveBeenCalledExactlyOnceWith` exists in recent Vitest (3.x+); on older versions use `toHaveBeenCalledTimes(1)` plus `toHaveBeenCalledWith`.

## In the wild
- Python: `unittest.mock` (`create_autospec`, `spec_set`, `mock_calls`), `pytest-mock`, `flexmock`, `doublex`.
- JS/TS: Vitest/Jest `vi.fn()` / `jest.fn()`, `jest-mock-extended`, Sinon mocks (`sinon.mock(obj).expects('m').once()`), `ts-mockito`.
- Angular: `HttpTestingController.expectOne(url)` + `verify()` is a true mock: it fails on unmatched or outstanding requests.

## Trade-offs
| Pros | Cons |
|---|---|
| Catches unexpected or duplicate calls to dangerous collaborators | Tightly couples tests to implementation; refactors break tests |
| Tests interaction protocols that have no observable state | Easy to write tests that pass while the real integration is broken |
| Drives interface discovery in outside-in TDD | Setup can dwarf the behavior under test |
| No infrastructure needed | Mocking third-party APIs encodes wrong assumptions |

## Pitfalls
- **Mocking what you don't own:** wrap it, then mock your wrapper; cover the wrapper with an integration test.
- **Lenient mocks pretending to be strict:** a bare `Mock()` returns another `Mock` for any attribute, so typos pass. Use `spec_set`/autospec or typed hand-written doubles.
- **`mock.patch` in the wrong namespace:** patch where the name is looked up, not where it is defined.
- **Verifying every internal call:** the SKILL anti-pattern "mocks asserting every internal call". Mock only at boundaries; verify only calls the requirement mentions.
- **Forgetting verify:** with explicit-expectation libraries, a missing `verify()` makes the test vacuous. Put it in `afterEach` (`httpMock.verify()`).

## Refactoring path
- **Introduce it:** a test currently asserts on a real side effect you cannot afford (a real charge). Put the collaborator behind a port, inject a strict mock, assert the call list.
- **Remove it / simplify:** when mock-heavy tests break on harmless refactors, replace the mock with a Fake that records state (`FakeGateway.refunds`) and assert outcomes; keep strict mocks only for the few "must never happen" calls.

## Related patterns
- [Test Spy](../testing/test-spy.md) — records calls and lets the test assert; less strict, more readable.
- [Test Stub](../testing/test-stub.md) — supplies inputs; never verifies.
- [Fake Object](../testing/fake-object.md) — prefer it when state can be observed.
- [Dependency Injection](../behavioral/dependency-injection.md) — required to substitute the mock without patching.
