# CQS (Command-Query Separation)

**Category:** UI · **Also known as:** Command-Query Separation principle (Bertrand Meyer) · **Related:** CQRS, Presentation Model, Command, Observer, Idempotent Receiver

## Intent
Every method is either a **command** that changes state and returns nothing, or a **query** that returns data and has no observable side effects — never both. "Asking a question should not change the answer."

## Problem
- `getUser()` lazily creates the user if missing; `nextInvoiceNumber()` both returns and increments; `validate()` normalizes the input as a side effect.
- Calling a "getter" twice gives different results, so it cannot be cached, memoized, or safely called from a template or a `computed()`.
- Tests and readers cannot tell from a name whether a call is safe to repeat.
- In UIs, template expressions or selectors that mutate state cause change-detection loops (`ExpressionChangedAfterItHasBeenCheckedError`) or infinite effect cycles.

Scenario: a cart service has `addItem(item): Cart` which mutates and returns the new cart, and `total()` which also records an analytics event. A `computed(() => cart.total())` fires analytics on every recompute. Splitting into `addItem(): void` and a pure `total` query fixes both.

## Solution and structure
```
Commands:  add_item(), apply_coupon(), checkout()      → change state, return None/void (or only an ack/id)
Queries:   total, items, can_checkout                  → return data, no side effects, safe to repeat
```
- Queries can be cached, memoized, called in any order, and used in reactive derivations.
- Commands are where side effects, validation and events live.
- Pragmatic exceptions (Meyer acknowledged them): `stack.pop()`, `queue.get()`, atomic `increment_and_get`, `dict.setdefault`. When atomicity matters, a combined operation is correct; name it so it's obviously a command.
- CQS is a method-level principle. **CQRS** scales it to separate models/stores for reads and writes.

## When to use
- Designing any service, store, or class API.
- Angular signal stores: queries as `signal`/`computed`, commands as methods. `computed` *must* be side-effect free.
- APIs: GET must be safe; POST/PUT/PATCH/DELETE are commands (HTTP's own CQS).
- Anywhere you want caching or memoization of reads.

## When NOT to use (or bend it)
- Atomic read-modify-write on shared state (`pop`, compare-and-swap, `INSERT … RETURNING id`): splitting into query + command introduces race conditions.
- Commands returning a created id or a result object is widely accepted (`create_order() -> OrderId`); returning the *entire mutated state* from every command is the smell.
- Fluent builders (`builder.with_x().with_y()`) return `self` by design.

## Example — Python
```python
from dataclasses import dataclass, field
from decimal import Decimal

@dataclass(frozen=True)
class Line:
    sku: str
    qty: int
    unit_price: Decimal

@dataclass
class Cart:
    _lines: dict[str, Line] = field(default_factory=dict)
    _coupon_percent: int = 0

    # --- commands: mutate, return None -------------------------------
    def add_item(self, sku: str, qty: int, unit_price: Decimal) -> None:
        if qty <= 0:
            raise ValueError("qty must be positive")
        existing = self._lines.get(sku)
        new_qty = qty + (existing.qty if existing else 0)
        self._lines[sku] = Line(sku, new_qty, unit_price)

    def apply_coupon(self, percent: int) -> None:
        self._coupon_percent = percent

    # --- queries: pure, repeatable -----------------------------------
    @property
    def lines(self) -> tuple[Line, ...]:
        return tuple(self._lines.values())      # copy: callers can't mutate state

    @property
    def total(self) -> Decimal:
        gross = sum((l.unit_price * l.qty for l in self._lines.values()), Decimal(0))
        return (gross * (100 - self._coupon_percent) / 100).quantize(Decimal("0.01"))

# Pragmatic exception — atomic in the DB, so not split:
# INSERT INTO invoice_seq DEFAULT VALUES RETURNING id   -> next_invoice_number()
```

## Example — TypeScript
```ts
import { Injectable, computed, signal } from '@angular/core';

interface Line { sku: string; qty: number; unitPrice: number }

@Injectable({ providedIn: 'root' })
export class CartStore {
  private readonly _lines = signal<readonly Line[]>([]);
  private readonly couponPercent = signal(0);

  // queries: read-only signals and pure computeds
  readonly lines = this._lines.asReadonly();
  readonly total = computed(() => {
    const gross = this._lines().reduce((s, l) => s + l.unitPrice * l.qty, 0);
    return Math.round(gross * (100 - this.couponPercent())) / 100;
  });
  readonly isEmpty = computed(() => this._lines().length === 0);

  // commands: change state, return void
  addItem(sku: string, qty: number, unitPrice: number): void {
    this._lines.update((lines) => {
      const i = lines.findIndex((l) => l.sku === sku);
      return i < 0
        ? [...lines, { sku, qty, unitPrice }]
        : lines.map((l, j) => (j === i ? { ...l, qty: l.qty + qty } : l));
    });
  }
  applyCoupon(percent: number): void { this.couponPercent.set(percent); }
}
```
Never write signals or call commands inside `computed()`; Angular throws or warns because computeds must be pure. Side effects that react to state belong in `effect()` or in the command itself.

## In the wild
- HTTP method semantics (GET/HEAD safe; POST not), GraphQL `query` vs `mutation`.
- Angular signals (`computed` must be pure), NgRx (selectors = queries, actions/reducers = commands), Redux.
- Python: `list.sort()` returns `None` (command) while `sorted()` returns a new list (query) — CQS in the stdlib; `dict.setdefault` and `list.pop` are deliberate exceptions.
- CQRS frameworks (MediatR, NestJS `@nestjs/cqrs`) take it to the architecture level.

## Trade-offs
| Pros | Cons |
|---|---|
| Queries are safe to cache, memoize, repeat and reorder | Atomic "do and tell" operations must be justified exceptions |
| Readers know which calls have side effects | Sometimes an extra call to read state after a command |
| Fits reactive UIs (signals, selectors) perfectly | Strict purists reject returning ids from commands, adding friction |
| Simplifies testing: assert on queries after commands | Not enforced by languages; relies on discipline and naming |

## Pitfalls
- **Getters with side effects** (lazy creation, logging analytics, incrementing counters). Move effects into explicit commands.
- **Commands returning internal mutable state** (`add_item()` returns `self._lines`): callers mutate it. Return nothing, or an immutable snapshot.
- **Splitting atomic operations** into `get_next()` + `increment()` and creating a race. Keep them atomic and name the method as a command.
- **Effects writing signals that their own sources read** (feedback loops). Prefer `computed` for derivation; use `effect` only for external side effects.

## Refactoring path
- **Introduce it:** find methods that both mutate and return; split into a void command and a query; update callers to call the command then read the query; make queries return copies or readonly views.
- **Remove it / simplify:** where a split created races or round-trips (two DB calls), merge into one atomic command that returns the minimal result (id, version), clearly named.

## Related patterns
- [CQRS](../cloud/cqrs.md) — CQS at the architecture level, with separate read and write models.
- [Presentation Model](../ui/presentation-model.md) — exposes queries as signals and commands as methods.
- [Command](../behavioral/command.md) — reifies commands as objects for queues, undo, logging.
- [Observer](../behavioral/observer.md) — queries as observable state; views subscribe.
- [Idempotent Receiver](../emergent/idempotent-receiver.md) — makes repeated commands safe, as queries already are.
