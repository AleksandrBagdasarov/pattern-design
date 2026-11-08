# Null Object

**Category:** Behavioral · **Also known as:** Active Nothing, Special Case (Fowler, when the default is not "nothing") · **Related:** Black Hole, Strategy, Optional, Dependency Injection, Test Stub

## Intent
Replace the absence of a collaborator (`None`/`null`) with an object that implements the same interface and does a sensible "nothing", so callers never need to check for absence.

## Problem
- A collaborator is optional: a discount policy that may not apply, a notifier that is off in some tenants, a parent node at the root of a tree, an anonymous user without a profile.
- Callers fill with `if x is not None:` / `x?.foo()` checks. Every new call site must remember the check; one forgotten check is an `AttributeError` / `TypeError` in production.
- Smells: the same `None` guard repeated across many methods; defaults computed ad hoc at each call (`discount.rate if discount else 0`); `Optional[...]` fields whose only purpose is "feature off".

Scenario: checkout applies an optional `DiscountPolicy`. Most orders have none. Ten functions do `if policy: total = policy.apply(total)`.

## Solution and structure
Define the interface once and provide an implementation whose methods return neutral values (identity, zero, empty collection, no-op).

```
          DiscountPolicy (Protocol)
           ├── PercentOff(10)      apply(t) -> t * 0.9
           ├── FixedOff(500)       apply(t) -> t - 500
           └── NoDiscount          apply(t) -> t        <- Null Object
```

Participants:
- **Abstraction** (Protocol / interface).
- **Real implementations**.
- **Null implementation**: stateless, usually one shared instance; behavior is the neutral element of the operation.
- **Client**: receives the abstraction, never `None`; the decision "absent → null object" is made once, at construction or at the boundary.

The neutral behavior must be *correct* for the domain, not merely silent. "No discount" is a legitimate business answer; "no payment gateway" is not.

## When to use
- The collaborator is genuinely optional and "do nothing / neutral value" is a correct outcome.
- Many call sites would otherwise repeat the same `None` check.
- Recursive structures (tree root's parent, leaf's children) where a neutral terminator simplifies traversal.
- Defaults for injected hooks: `on_progress`, `logger`, `metrics`, `cache`.

## When NOT to use
- Absence is an error the caller must handle (user not found, config missing): return `None`/raise and let the type system force the check (Optional pattern), or raise a domain exception. A null object here hides bugs.
- The caller needs to *know* whether the thing exists (show "Add a card" vs "Visa •••• 4242"): use `T | None` and narrow explicitly.
- A single call site: a one-line `if` is simpler than a new class.
- The "neutral" result is ambiguous (what does a null `PriceProvider` return: 0? That gives free products). If there is no safe neutral value, don't use it.

## Example — Python
A Protocol plus a tiny null class. Python's duck typing means the null object needs no inheritance; a module-level instance is enough.

```python
from dataclasses import dataclass
from decimal import Decimal
from typing import Protocol, Final

class DiscountPolicy(Protocol):
    def apply(self, subtotal: Decimal) -> Decimal: ...
    def describe(self) -> str: ...

@dataclass(frozen=True)
class PercentOff:
    percent: Decimal
    def apply(self, subtotal: Decimal) -> Decimal:
        return subtotal * (1 - self.percent / 100)
    def describe(self) -> str:
        return f"{self.percent}% off"

class _NoDiscount:
    def apply(self, subtotal: Decimal) -> Decimal:
        return subtotal                      # identity: the neutral element
    def describe(self) -> str:
        return ""
    def __repr__(self) -> str:
        return "NO_DISCOUNT"

NO_DISCOUNT: Final[DiscountPolicy] = _NoDiscount()

def policy_for(code: str | None, catalog: dict[str, DiscountPolicy]) -> DiscountPolicy:
    # The only place that deals with absence.
    return catalog.get(code, NO_DISCOUNT) if code else NO_DISCOUNT

@dataclass
class Checkout:
    discount: DiscountPolicy = NO_DISCOUNT
    def total(self, subtotal: Decimal) -> Decimal:
        return self.discount.apply(subtotal).quantize(Decimal("0.01"))   # no None check
```

For callables, the Python null object is often just `lambda *_: None` or `contextlib.nullcontext()` for an optional context manager.

## Example — TypeScript
An Angular DI token with a null implementation as the default provider; features override it when enabled.

```typescript
import { Injectable, InjectionToken, inject } from '@angular/core';

export interface Analytics {
  track(event: string, props?: Record<string, unknown>): void;
  identify(userId: string): void;
}

export const NULL_ANALYTICS: Analytics = Object.freeze({
  track: () => {},
  identify: () => {},
});

export const ANALYTICS = new InjectionToken<Analytics>('ANALYTICS', {
  providedIn: 'root',
  factory: () => NULL_ANALYTICS,          // default: tracking off (dev, tests, no consent)
});

@Injectable({ providedIn: 'root' })
export class CheckoutService {
  private readonly analytics = inject(ANALYTICS);

  placeOrder(orderId: string): void {
    // ... place order ...
    this.analytics.track('order_placed', { orderId });   // never `this.analytics?.track`
  }
}

// app.config.ts, when the user consented:
// providers: [{ provide: ANALYTICS, useClass: SegmentAnalytics }]
```

## In the wild
- Python: `contextlib.nullcontext`, `logging.NullHandler` (libraries add it so they never print unless configured), `unittest.mock` is *not* a null object (it records), `io` sinks like `open(os.devnull, "w")`.
- Angular: `NoopAnimationsModule` / `provideNoopAnimations()`, `NoopInterceptor` examples, DI tokens with no-op defaults.
- RxJS: `EMPTY` and `NEVER` are null observables; `noop` function.
- OpenTelemetry: `NoOpTracer` / `NoOpMeter` returned by the API when no SDK is installed.
- Collections: returning `[]` instead of `None` is the most common null object of all.

## Trade-offs
| Pros | Cons |
|---|---|
| Removes repeated `None` checks; simpler client code | Can hide real absence bugs (silent failure) |
| Absence decided once, at the boundary | Callers cannot distinguish "off" from "on" without extra API |
| Polymorphic: fits DI and Strategy naturally | Another class per interface to maintain as it grows |
| Safe defaults for optional hooks | Wrong neutral value causes subtle domain errors |

## Pitfalls
- **Null object for something that must exist.** A `NullPaymentGateway` that "succeeds" ships free orders. Fix: only use it where doing nothing is correct; otherwise fail fast.
- **Returning another `None` from a null object's method.** The checks move one level down. Fix: return neutral values of the declared type (empty list, identity, another null object).
- **Interface drift.** A new method is added to the Protocol but not to the null class; static typing catches it only if the null object is annotated as the Protocol type. Fix: annotate (`Final[DiscountPolicy]`) or subclass the ABC.
- **Stateful null objects.** Fix: make them immutable singletons (`Object.freeze`, module constant).
- **Using `isinstance(x, NoDiscount)` checks.** Reintroduces the conditionals. Fix: if callers need to branch, add a query method (`is_active()`) or use `T | None` instead.

## Refactoring path
- **Introduce it** (Fowler's "Introduce Special Case"): (1) find the repeated `if x is None` for one collaborator; (2) create the null class implementing the interface with neutral behavior; (3) at the source (factory, constructor default, DI default) return the null object instead of `None`; (4) delete the checks one call site at a time; (5) narrow the type from `T | None` to `T`.
- **Remove it / simplify:** if callers keep checking `isinstance(..., NullX)`, or absence turns out to be an error case, change the type back to `T | None`, return `None`, and let the type checker force handling.

## Related patterns
- [Black Hole](black-hole.md) — a null object specialized for sinks that swallow everything (loggers, metrics).
- [Strategy](strategy.md) — the null object is often the "do nothing" strategy.
- [Optional](../emergent/optional.md) — the opposite choice: make absence explicit so callers must handle it.
- [Dependency Injection](dependency-injection.md) — null objects are the natural default binding for optional dependencies.
- [Test Stub](../testing/test-stub.md) — similar shape, but exists for tests and returns canned answers, not neutral ones.
