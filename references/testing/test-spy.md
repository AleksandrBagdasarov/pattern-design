# Test Spy

**Category:** Testing · **Also known as:** Recording stub · **Related:** Test Stub, Mock Object, Fake Object, Arrange-Act-Assert

## Intent
A test double that records how it was called (arguments, count, order) so the test can assert on the SUT's **indirect outputs** after the action, while optionally returning canned values like a stub.

## Problem
- The SUT's observable effect is a call to another component: send an email, publish an event, write an audit log, call an analytics SDK.
- The return value of the SUT does not show whether that happened.
- You want to check the call *after* Act, in normal Assert style, instead of declaring expectations upfront.

Scenario: `register_user()` must publish exactly one `UserRegistered` event with the user's id. The function returns the new user; the event is a side effect. A spy publisher records `publish(event)` calls, and the test checks the list.

Spy versus mock (Meszaros): a spy is passive and the **test** asserts on what it recorded; a mock holds expectations and **verifies itself**. Python's `unittest.mock.Mock` and Vitest's `vi.fn()` are spies by this definition, despite their names.

## Solution and structure
```
Arrange: spy = SpyPublisher()        Act: sut.do(spy)        Assert: spy.calls == [...]
                    ▲                         │
                    └──── records call ◀──────┘
```
- **Spy** implements the DOC interface, stores each call, returns a default or canned value.
- **Test** inspects the recording in the Assert phase.
- A *partial spy* (`vi.spyOn`, `mock.patch.object(..., wraps=...)`) wraps a real object and records calls while still running real code.

## When to use
- The outgoing call *is* the behavior: notifications, events, metrics, audit records, cache invalidation.
- You want readable Arrange-Act-Assert tests, with assertions at the end.
- You need to check call count or arguments of a boundary you do not own.

## When NOT to use
- The effect can be checked through state: use a Fake (in-memory outbox, repository) and assert on its contents. This survives refactors better.
- Asserting calls between your own internal classes. That tests implementation, not behavior; tests will break when you restructure.
- The SUT returns the answer directly: assert on the return value.

## Example — Python
```python
from dataclasses import dataclass, field
from typing import Protocol
from unittest.mock import create_autospec

@dataclass(frozen=True)
class UserRegistered:
    user_id: int
    email: str

class EventBus(Protocol):
    def publish(self, event: object) -> None: ...

@dataclass
class SpyBus:
    published: list[object] = field(default_factory=list)
    def publish(self, event: object) -> None:
        self.published.append(event)

def register_user(email: str, bus: EventBus, next_id: int = 42) -> int:
    bus.publish(UserRegistered(next_id, email.lower()))
    return next_id

def test_publishes_one_event_hand_written_spy() -> None:
    bus = SpyBus()
    user_id = register_user("Ada@Example.com", bus)
    assert bus.published == [UserRegistered(user_id, "ada@example.com")]

def test_publishes_with_autospec_spy() -> None:
    bus = create_autospec(EventBus, instance=True)   # rejects unknown methods/signatures
    register_user("ada@example.com", bus)
    bus.publish.assert_called_once_with(UserRegistered(42, "ada@example.com"))
```
The hand-written spy compares dataclasses with `==` and needs no mocking library. `create_autospec` is preferable to a bare `Mock()` because a typo like `assert_called_onse` or a wrong signature fails loudly.

## Example — TypeScript
```ts
import { TestBed } from '@angular/core/testing';
import { describe, it, expect, vi } from 'vitest';
import { CartStore } from './cart.store';
import { Analytics } from './analytics';

describe('CartStore', () => {
  it('tracks add_to_cart with sku and quantity', () => {
    const track = vi.fn<(name: string, props: Record<string, unknown>) => void>();
    TestBed.configureTestingModule({
      providers: [CartStore, { provide: Analytics, useValue: { track } }],
    });
    const cart = TestBed.inject(CartStore);

    cart.add({ sku: 'SKU-1', qty: 2 });

    expect(track).toHaveBeenCalledTimes(1);
    expect(track).toHaveBeenCalledWith('add_to_cart', { sku: 'SKU-1', qty: 2 });
    expect(cart.count()).toBe(2);
  });

  it('partial spy on a real method', () => {
    TestBed.configureTestingModule({ providers: [CartStore, Analytics] });
    const analytics = TestBed.inject(Analytics);
    const spy = vi.spyOn(analytics, 'track').mockImplementation(() => {});
    TestBed.inject(CartStore).add({ sku: 'SKU-2', qty: 1 });
    expect(spy.mock.calls[0]?.[0]).toBe('add_to_cart');
  });
});
```
Restore partial spies (`vi.restoreAllMocks()` in `afterEach`, or `restoreMocks: true` in Vitest config) so they do not leak between tests.

## In the wild
- Python: `unittest.mock.Mock`/`MagicMock`/`AsyncMock` (`call_args_list`, `assert_awaited_once_with`), `mock.patch.object(obj, "m", wraps=obj.m)`, pytest-mock's `mocker.spy`.
- Vitest/Jest: `vi.fn()`, `vi.spyOn()`; Jasmine: `jasmine.createSpyObj`, `spyOn` (legacy Angular setups).
- Sinon.js `sinon.spy()`; logging: pytest `caplog` is effectively a spy on the logging system.

## Trade-offs
| Pros | Cons |
|---|---|
| Verifies side effects that have no return value | Couples tests to call shape; refactors that keep behavior can break them |
| Assertions stay in the Assert phase (readable AAA) | Easy to over-specify (exact args, order) beyond what matters |
| Library spies need zero boilerplate | Untyped spies (`Mock()`) accept typos and wrong signatures silently |
| Partial spies observe real objects without replacing them | Partial spies leak if not restored |

## Pitfalls
- **Over-specification:** asserting every argument and call order. Assert only what the requirement states; use `ANY` / `expect.objectContaining`.
- **Silent false positives with bare `Mock`:** misspelled assertion methods used to pass silently (Python 3.12+ raises `AttributeError` for common misspellings, but not all). Use `create_autospec` or hand-written spies.
- **Async:** checking `assert_called` on an `AsyncMock` that was called but never awaited. Use `assert_awaited_*`.
- **Spying on internals** (private helpers of the SUT). Spy only at architectural boundaries.

## Refactoring path
- **Introduce it:** a test currently checks a side effect by reading logs or a real outbox table. Inject the DOC, pass a spy, assert on recorded calls.
- **Remove it / simplify:** if many tests assert sequences of calls on the same DOC, replace the spy with a Fake that keeps state (`FakeOutbox.messages`) and assert on that state. If the call is internal, delete the assertion and test the public outcome.

## Related patterns
- [Test Stub](../testing/test-stub.md) — controls inputs; a spy additionally records outputs.
- [Mock Object](../testing/mock-object.md) — verifies expectations itself instead of letting the test inspect a recording.
- [Fake Object](../testing/fake-object.md) — state-based alternative; often more refactor-proof.
- [Arrange-Act-Assert](../testing/arrange-act-assert.md) — spies keep verification in the Assert block.
