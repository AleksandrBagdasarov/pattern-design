# Fluent Builder

**Category:** Creational · **Also known as:** Fluent Interface, Method Chaining, Test Data Builder (in tests) · **Related:** Builder, Object Mother, Query Object, Specification, Prototype

## Intent
A Builder whose methods return the builder (or a new builder) so construction reads as one chained expression: `select(Order).where(...).order_by(...).limit(20)`.

## Problem
- Builder calls on separate lines (`b.to(x); b.subject(y); b.attach(z)`) are noisy, and the meaning of the whole is lost.
- Query or request construction has many optional clauses applied conditionally (filters from a search form).
- Tests need many slight variations of a valid object: "a paid order with two items in EUR".
- Scenario: a reporting endpoint accepts ten optional filters. The handler builds SQL by string concatenation, with a bug when two filters are combined. A chainable query builder makes each clause a method and keeps SQL generation in one place.

## Solution and structure
- **Fluent builder**: each configuring method returns `self` (mutable style) or a *new* builder (immutable style), so calls chain.
- **Terminal method**: `build()`, `execute()`, `toString()` ends the chain and produces the product.
- Optionally, **typed stages** (step builder): each method returns a different type that only exposes the legal next steps, so the compiler enforces order and required parts.

```
QueryBuilder ─where()─► QueryBuilder ─orderBy()─► QueryBuilder ─build()─► Query
  (each step returns a builder; terminal step returns the product)
```

Mutable vs immutable chaining:
- **Mutable** (`return self`): cheap, but a builder shared between two branches gets both branches' changes.
- **Immutable** (return a copy): safe to branch and reuse as a base (`base.where(a)` and `base.where(b)`), small allocation cost. SQLAlchemy `Select`, Angular `HttpParams` and RxJS `pipe` use this style.

## When to use
- Construction is a sequence of optional, composable clauses (queries, HTTP requests, pipelines, validation chains).
- Readability of the call site matters (DSL-like APIs, public libraries).
- Test data: a Test Data Builder with sensible defaults plus `.with_x()` overrides.

## When NOT to use
- Few fields: keyword args (`Order(status="paid")`) or `{ ...defaults, status: 'paid' }` are clearer and type-check better.
- Each step has side effects (I/O). Chains hide where effects happen; use explicit calls.
- Debuggability matters more than brevity: long chains give one-line stack traces and are hard to breakpoint.
- Python codebases where it fights the idiom: a chain of setters returning `self` is less Pythonic than kwargs and `dataclasses.replace`.

## Example — Python
An immutable fluent Test Data Builder. Each `with_` returns a new builder via `dataclasses.replace`, so a base builder can be shared across tests safely.

```python
from dataclasses import dataclass, field, replace
from decimal import Decimal
from typing import Self

@dataclass(frozen=True)
class OrderLine:
    sku: str
    qty: int
    price: Decimal

@dataclass(frozen=True)
class Order:
    id: str
    customer_id: str
    currency: str
    status: str
    lines: tuple[OrderLine, ...]

@dataclass(frozen=True)
class OrderBuilder:
    id: str = "ord-1"
    customer_id: str = "cus-1"
    currency: str = "EUR"
    status: str = "new"
    lines: tuple[OrderLine, ...] = field(default=())

    def with_status(self, status: str) -> Self: return replace(self, status=status)
    def in_currency(self, cur: str) -> Self: return replace(self, currency=cur)
    def with_line(self, sku: str, qty: int = 1, price: str = "9.99") -> Self:
        return replace(self, lines=(*self.lines, OrderLine(sku, qty, Decimal(price))))

    def build(self) -> Order:
        lines = self.lines or (OrderLine("SKU-DEFAULT", 1, Decimal("9.99")),)
        return Order(self.id, self.customer_id, self.currency, self.status, lines)

def an_order() -> OrderBuilder:
    return OrderBuilder()

# test
paid_usd = an_order().with_status("paid").in_currency("USD").with_line("SKU-1", qty=2).build()
```

## Example — TypeScript
A step builder: the types allow `build()` only after the required `to()` step, so a missing recipient is a compile error.

```typescript
interface Notification { readonly channel: 'email' | 'sms'; readonly to: string; readonly body: string; readonly priority: 'low' | 'high'; }

class NotificationDraft {
  constructor(private readonly n: Omit<Notification, 'to'>) {}
  to(recipient: string): ReadyNotification { return new ReadyNotification({ ...this.n, to: recipient }); }
}

class ReadyNotification {
  constructor(private readonly n: Notification) {}
  urgent(): ReadyNotification { return new ReadyNotification({ ...this.n, priority: 'high' }); }
  body(text: string): ReadyNotification { return new ReadyNotification({ ...this.n, body: text }); }
  build(): Notification {
    if (this.n.channel === 'sms' && this.n.body.length > 160) throw new Error('SMS body > 160 chars');
    return Object.freeze({ ...this.n });
  }
}

export const notification = (channel: Notification['channel']) =>
  new NotificationDraft({ channel, body: '', priority: 'low' });

// notification('sms').build();                          // compile error: no build() on draft
const n = notification('sms').to('+4915112345678').body('Your order shipped').urgent().build();
```

## In the wild
- SQLAlchemy 2.x `select(User).where(...).order_by(...)` (immutable, generative); Django QuerySets (lazy, chainable).
- pandas method chaining (`df.assign(...).query(...).groupby(...)`); `polars` lazy frames.
- Angular `HttpParams`/`HttpHeaders` (`.set()` returns a new instance; ignoring the return value is a classic bug); RxJS `pipe(...)`; `Validators.compose`.
- Node: Knex, Kysely, Drizzle, TypeORM `QueryBuilder`; `zod` schemas (`z.string().min(1).email()`); Express `res.status(404).json(...)`.
- Testing: factory_boy, polyfactory, `@faker-js/faker` combined with test data builders.

## Trade-offs
| Pros | Cons |
|---|---|
| Very readable, DSL-like call sites | Long chains are hard to debug and step through |
| Optional parts composed conditionally (`if f: q = q.where(...)`) | Mutable chaining causes aliasing bugs when branched |
| Immutable style allows safe reuse of partial builders | Immutable style allocates a copy per step |
| Step builders encode required order in types | Step-builder typing is verbose to write and maintain |

## Pitfalls
- **Ignoring the return value of an immutable builder**: `params.set('page', '2')` without reassigning does nothing. Fix: always `params = params.set(...)`; lint for unused results.
- **Mutable builder reused as a base**: `base.where(a)` then `base.where(b)` yields both filters. Fix: immutable builders or explicit `copy()`.
- **Terminal method forgotten**: passing the builder where the product was expected. Fix: distinct types; strict typing catches it.
- **Chains with hidden I/O** (`.save()` in the middle). Fix: only the terminal method may perform effects.
- **Over-fluent APIs** where every field has a `with_x`. Keep the methods for domain-meaningful variations (`paid()`, `shipped_to("DE")`).

## Refactoring path
- **Introduce it:** start from a plain Builder whose call sites are repetitive. Make setters return `self`/a copy; add `build()` as terminal; convert call sites; for public APIs, consider immutable style before publishing, since changing it later breaks callers.
- **Remove it / simplify:** if chains only set fields once, replace with a constructor or options object with defaults; keep the domain-named helpers as plain functions (`paid_order()`).

## Related patterns
- [Builder](builder.md) — the non-chained base pattern and its invariants rules.
- [Object Mother](object-mother.md) — named canonical objects; often returns fluent builders for tweaking.
- [Query Object](../data-access/query-object.md) — the product of most fluent query builders.
- [Specification](../behavioral/specification.md) — combinable predicates, often exposed fluently (`a.and_(b)`).
- [Prototype](prototype.md) — immutable fluent builders are effectively prototypes copied with changes.
