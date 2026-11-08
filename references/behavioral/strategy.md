# Strategy

**Category:** Behavioral · **Also known as:** Policy · **Related:** State, Template Method, Command, Bridge, Factory, Dependency Injection, Specification

## Intent
Define a family of interchangeable algorithms behind one interface so the caller can pick one at runtime without changing its own code.

## Problem
- One function holds several algorithms selected by a flag: `if provider == "stripe": ... elif provider == "paypal": ...`, and every new provider edits it.
- The same `if/elif` on the same key is repeated in several places (fee calculation, refund, webhook parsing).
- Tests need to force a specific algorithm (a fixed clock, a deterministic pricing rule) but it is hard-coded.
- Scenario: shipping cost is flat for domestic orders, weight-based for EU and carrier-API-based for the rest. Marketing wants a "free over 50 EUR" promo for one week.

## Solution and structure
- **Strategy** is the interface: one operation, such as `cost(order) -> Money`.
- **Concrete strategies** implement it. Each one is independent and testable alone.
- **Context** holds a strategy and delegates to it. It does not know which concrete one it has.
- A **registry or factory** usually maps a key (config value, country, plan) to a strategy.

```
Context ──uses──▶ «Strategy» cost(order)
                     ▲        ▲          ▲
                 Flat    WeightBased   CarrierApi
```

In Python and TypeScript a strategy is very often **just a function**. Use a class only when the strategy carries configuration or several related methods.

## When to use
- Two or more real algorithms for the same job exist today, chosen by data or config.
- Callers or tests must inject the behaviour (sorting key, retry policy, pricing rule, auth scheme).
- The same switch appears in several places; collecting each branch into one object removes the duplication.
- Plugins: third parties add algorithms without editing your code.

## When NOT to use
- Only one algorithm exists and no second one is planned. Write the function.
- Two branches that differ by a constant. Use a parameter.
- The algorithm changes because the object's own lifecycle state changes (draft, paid, shipped). That is State.
- The variants share 90% of the steps and differ in one hook. A callable parameter for that hook is enough; Template Method is the class-based alternative.
- A class hierarchy of `XStrategy` with single `execute()` methods in Python. Pass a callable.

## Example — Python
The language dissolves the pattern: strategies are callables, the registry is a dict.

```python
from collections.abc import Callable
from dataclasses import dataclass
from decimal import Decimal

@dataclass(frozen=True)
class Order:
    country: str
    weight_kg: Decimal
    subtotal: Decimal

type ShippingPolicy = Callable[[Order], Decimal]

def flat_domestic(order: Order) -> Decimal:
    return Decimal("4.90")

def weight_based(order: Order) -> Decimal:
    return Decimal("3.00") + order.weight_kg * Decimal("1.20")

def free_over(threshold: Decimal, fallback: ShippingPolicy) -> ShippingPolicy:
    def policy(order: Order) -> Decimal:
        return Decimal(0) if order.subtotal >= threshold else fallback(order)
    return policy

POLICIES: dict[str, ShippingPolicy] = {
    "UA": flat_domestic,
    "EU": free_over(Decimal(50), weight_based),
}

def shipping_cost(order: Order, policies: dict[str, ShippingPolicy] = POLICIES) -> Decimal:
    region = "UA" if order.country == "UA" else "EU"
    return policies[region](order)

assert shipping_cost(Order("PL", Decimal(2), Decimal(60))) == 0
```

Use a `Protocol` class instead of `Callable` when a strategy needs more than one method (e.g. `quote()` and `label()`).

## Example — TypeScript
In Angular, a strategy is commonly an injection token so the app or a test can swap it.

```typescript
import { Injectable, InjectionToken, inject } from '@angular/core';

export interface Order { country: string; weightKg: number; subtotal: number }
export type ShippingPolicy = (order: Order) => number;

export const flatDomestic: ShippingPolicy = () => 4.9;
export const weightBased: ShippingPolicy = o => 3 + o.weightKg * 1.2;
export const freeOver = (threshold: number, fallback: ShippingPolicy): ShippingPolicy =>
  o => (o.subtotal >= threshold ? 0 : fallback(o));

export const SHIPPING_POLICIES = new InjectionToken<Record<string, ShippingPolicy>>(
  'SHIPPING_POLICIES',
  { providedIn: 'root', factory: () => ({ UA: flatDomestic, EU: freeOver(50, weightBased) }) },
);

@Injectable({ providedIn: 'root' })
export class ShippingCalculator {
  private readonly policies = inject(SHIPPING_POLICIES);

  cost(order: Order): number {
    const policy = this.policies[order.country === 'UA' ? 'UA' : 'EU'];
    if (!policy) throw new Error(`No shipping policy for ${order.country}`);
    return policy(order);
  }
}
// test: TestBed.configureTestingModule({ providers: [{ provide: SHIPPING_POLICIES, useValue: { UA: () => 0, EU: () => 0 } }] })
```

## In the wild
- `sorted(key=...)`, `max(key=...)`, `json.dumps(default=...)`, `functools.cmp_to_key`.
- `Array.prototype.sort(compareFn)`, Angular `RouteReuseStrategy`, `TitleStrategy`, `PreloadingStrategy`, `ErrorHandler`.
- Passport.js strategies, NestJS `AuthGuard('jwt')`, Python `httpx.Auth` subclasses.
- `tenacity` wait/stop strategies, `asyncio` event loop policies, Pydantic `field_serializer`.
- Kafka partitioners, AWS SDK retry strategies (`StandardRetryStrategy`).

## Trade-offs
| Pros | Cons |
|---|---|
| New algorithm = new function, no edits to the context (open/closed) | More units to name and find; the selection logic moves somewhere else |
| Each algorithm is tested in isolation | Callers or config must know which strategies exist |
| Removes duplicated switches on the same key | Overkill when there are only two stable branches |
| Easy to inject fakes in tests | Strategies needing very different inputs force a bloated shared signature |

## Pitfalls
- **Strategy class with one method in Python/TS.** Fix: use a function type; upgrade to a Protocol/interface only when needed.
- **The switch just moves.** The `if/elif` now picks the strategy in five places. Fix: one registry or factory, built once in the composition root.
- **Leaky interface.** One strategy needs the DB, another the HTTP client, so the interface takes everything. Fix: inject dependencies into each strategy at construction (closure or constructor), keep the call signature small.
- **Missing key fails late.** `policies[key]` raises `KeyError` in production. Fix: validate config at startup, or use an explicit default strategy.
- **Stateful strategies shared between requests.** Fix: keep strategies pure, or build one per use.

## Refactoring path
- **Introduce it:** from a conditional that selects an algorithm. (1) Extract each branch into a function with the same signature. (2) Replace the branch bodies with calls to those functions. (3) Replace the conditional with a dict lookup. (4) Make the dict injectable (parameter, DI token) and test each function alone.
- **Remove it / simplify:** if the registry has one entry for a year, inline the function and delete the indirection. If strategies multiply but are configuration variants of one algorithm, collapse them into one function with parameters.

## Related patterns
- [State](state.md) — same structure, but the object switches its own strategy as its state changes.
- [Template Method](template-method.md) — inheritance-based variation of steps; Strategy uses composition and is usually preferred.
- [Command](command.md) — also an object wrapping behaviour, but it represents a request to execute, often with undo or queueing.
- [Factory](../creational/factory.md) — commonly chooses which strategy to build from config.
- [Dependency Injection](dependency-injection.md) — the usual way a strategy reaches the context.
- [Specification](specification.md) — composable boolean strategies for business rules.
- [Bridge](../structural/bridge.md) — structural cousin; separates an abstraction from a whole implementation hierarchy.
