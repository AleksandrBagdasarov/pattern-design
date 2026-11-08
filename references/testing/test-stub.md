# Test Stub

**Category:** Testing · **Also known as:** Stub, canned-answer double · **Related:** Test Spy, Fake Object, Mock Object, Dependency Injection, Arrange-Act-Assert

## Intent
Replace a depended-on component (DOC) with an object that returns pre-set answers, so the test can drive the system under test (SUT) down a specific path — including paths that are hard to reach with the real dependency.

## Problem
- The SUT reads from something slow, non-deterministic or hard to put into a given state: an HTTP API, the clock, a random generator, a payment gateway.
- You need to test the "gateway returns 503" or "exchange rate is 0" branch, and the real service will not cooperate on demand.
- Tests are flaky because they depend on time, network or shared data.

Scenario: `checkout()` must fall back to "pending" when the fraud API times out. You cannot make the real fraud API time out on command. A stub that raises `TimeoutError` gives you that branch in one line.

Meszaros' terms: a stub controls the SUT's **indirect inputs**. It does not check how it was called; that is a Spy or Mock.

## Solution and structure
```
test ──configures──▶ Stub (returns X / raises E)
  │                    ▲
  └──exercises──▶ SUT ─┘ calls DOC interface
  └──asserts on SUT's return value / state
```
- **SUT** depends on an interface (Protocol, abstract class, injected function).
- **Stub** implements just enough of that interface to return the values the test needs.
- The test **injects** the stub (constructor argument, `Depends` override, Angular provider) and asserts on the SUT's output or state, never on the stub.

Variants: *Responder* (returns valid values), *Saboteur* (raises errors), *Temporary Test Stub* (stands in for code not written yet).

## When to use
- The test needs a specific indirect input: a fixed time, a canned API response, an error.
- The real DOC is slow, costs money, or has side effects you do not want in a unit test.
- You test the SUT's decisions, not its conversation with the DOC.

## When NOT to use
- The dependency is a pure, fast function (a formatter, a calculator): call the real one.
- You need to verify an outgoing call happened (an email was sent): use a Spy or a Fake with inspectable state.
- The stub would need to emulate real behavior across many calls (save then load): write a Fake Object instead; a stub that grows logic is a fake in disguise.
- You are stubbing the code you are testing. Stub only the boundary.

## Example — Python
```python
from dataclasses import dataclass
from datetime import datetime, timezone
from typing import Protocol
import pytest

class FxRates(Protocol):
    def rate(self, base: str, quote: str) -> float: ...

@dataclass
class PriceService:
    rates: FxRates

    def convert(self, cents: int, base: str, quote: str) -> int:
        try:
            return round(cents * self.rates.rate(base, quote))
        except TimeoutError:
            raise PriceUnavailable(f"{base}->{quote}")

class PriceUnavailable(Exception): ...

class StubRates:                       # Responder
    def __init__(self, value: float) -> None: self.value = value
    def rate(self, base: str, quote: str) -> float: return self.value

class TimeoutRates:                    # Saboteur
    def rate(self, base: str, quote: str) -> float: raise TimeoutError

def test_converts_using_rate() -> None:
    assert PriceService(StubRates(1.1)).convert(1000, "EUR", "USD") == 1100

def test_timeout_maps_to_domain_error() -> None:
    with pytest.raises(PriceUnavailable):
        PriceService(TimeoutRates()).convert(1000, "EUR", "USD")

def test_clock_stub(monkeypatch: pytest.MonkeyPatch) -> None:
    import billing  # module that calls billing.now()
    monkeypatch.setattr(billing, "now", lambda: datetime(2026, 1, 1, tzinfo=timezone.utc))
    assert billing.invoice_month() == "2026-01"
```
In Python a stub is often just a lambda, a small class, or `Mock(return_value=...)`. A hand-written class is clearer and type-checked; `unittest.mock` is quicker but accepts any attribute unless you use `create_autospec`.

## Example — TypeScript
```ts
import { TestBed } from '@angular/core/testing';
import { of, throwError } from 'rxjs';
import { describe, it, expect } from 'vitest';
import { UserApi, ProfileStore } from './profile';

describe('ProfileStore', () => {
  it('shows the display name from the API', async () => {
    const stubApi: Pick<UserApi, 'getUser'> = {
      getUser: () => of({ id: 1, firstName: 'Ada', lastName: 'Lovelace' }),
    };
    TestBed.configureTestingModule({
      providers: [ProfileStore, { provide: UserApi, useValue: stubApi }],
    });
    const store = TestBed.inject(ProfileStore);
    await store.load(1);
    expect(store.displayName()).toBe('Ada Lovelace');
  });

  it('exposes an error state when the API fails', async () => {
    TestBed.configureTestingModule({
      providers: [ProfileStore, { provide: UserApi, useValue: { getUser: () => throwError(() => new Error('503')) } }],
    });
    const store = TestBed.inject(ProfileStore);
    await store.load(1);
    expect(store.error()).toBe('Could not load profile');
  });
});
```

## In the wild
- `unittest.mock.Mock(return_value=…, side_effect=…)`, `pytest`'s `monkeypatch.setattr`.
- FastAPI `app.dependency_overrides[get_rates] = lambda: StubRates(1.1)`.
- `respx` / `responses` / `pytest-httpx` stub HTTP at the transport level; `freezegun` / `time-machine` stub the clock.
- Vitest `vi.fn().mockReturnValue()`, `vi.useFakeTimers()`, `vi.setSystemTime()`; Angular `HttpTestingController` (`req.flush(...)`) stubs HTTP responses; MSW stubs network in browser/Node.

## Trade-offs
| Pros | Cons |
|---|---|
| Makes hard-to-reach branches (errors, edge values) trivial to test | Can drift from the real DOC's contract; tests pass, prod breaks |
| Fast, deterministic, no I/O | Tells you nothing about whether the SUT called the DOC correctly |
| Minimal code, easy to read | Many stubs per test signal a SUT with too many dependencies |

## Pitfalls
- **Contract drift:** the stub returns a shape the real API never returns. Fix: type the stub against the same Protocol/interface, use `create_autospec`, and keep a few contract or integration tests against the real thing.
- **Asserting on the stub** (`stub.rate.assert_called_with(...)`): that makes it a mock and couples the test to implementation. Assert on the SUT's result.
- **Stubbing too deep:** `mock.a.b.c.return_value` chains mean the SUT violates the Law of Demeter. Fix the design, not the test.
- **Patching the wrong name:** `mock.patch("requests.get")` when the module did `from requests import get`. Patch where it is looked up (`mymodule.get`), or better, inject.

## Refactoring path
- **Introduce it:** from a test that hits the network or the clock. Extract the call behind a parameter or Protocol (seam), inject the real one by default, pass a stub in the test.
- **Remove it / simplify:** if the stubbed thing is pure and fast, delete the stub and use the real function. If several tests build elaborate stubs with internal state, replace them with one shared Fake.

## Related patterns
- [Test Spy](../testing/test-spy.md) — a stub that also records calls for later assertions.
- [Fake Object](../testing/fake-object.md) — a working implementation instead of canned answers.
- [Mock Object](../testing/mock-object.md) — verifies outgoing calls; stubs only feed inputs.
- [Dependency Injection](../behavioral/dependency-injection.md) — the seam that lets you swap the stub in.
- [Arrange-Act-Assert](../testing/arrange-act-assert.md) — stubs are configured in Arrange.
