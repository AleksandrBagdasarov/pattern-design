# Specification

**Category:** Behavioral (domain) · **Also known as:** Specification pattern (Evans & Fowler), Predicate object, Criteria · **Related:** Composite, Interpreter, Query Object, Repository, Strategy, Chain of Responsibility

## Intent
Encapsulate a business rule as an object (a predicate) that answers "does this candidate satisfy the rule?", and make rules combinable with `and`, `or`, `not` so complex conditions are built from small, named, testable pieces.

## Problem
- The same business rule ("customer is eligible for free shipping") appears as inline conditions in a service, a query, a validator and a UI guard, and they drift apart.
- Rules combine: eligible = active AND not banned AND (premium OR lifetime spend ≥ 500). Inline, this becomes long boolean expressions that nobody can name or test in isolation.
- The same rule must run in memory (validate one object) and in the database (select all matching rows).
- Smells: copy-pasted `if c.status == "active" and not c.banned and ...`; repository methods multiplying (`find_active_premium_not_banned_in_eu`); rules hard-coded in SQL that differ from the Python version.

Scenario: marketing defines campaign audiences as combinations of rules. Each rule must be reusable, explainable ("why was this customer excluded?") and executable as a SQL `WHERE` over millions of customers.

## Solution and structure
A Specification exposes `is_satisfied_by(candidate) -> bool`. Composite specifications (`And`, `Or`, `Not`) combine others. Optionally each spec also knows how to translate itself into a query expression.

```
Specification[T]
  is_satisfied_by(t) -> bool
  to_sql() -> ColumnElement     (optional: query translation)
  __and__ / __or__ / __invert__
     ├── IsActive, IsBanned, MinSpend(500)        leaf rules
     └── And(a, b), Or(a, b), Not(a)              composites
```

Three uses (Evans): **validation** (does this object meet the rule?), **selection** (query all objects that meet it) and **building to order** (construct something that satisfies it).

## When to use
- Business rules that are named by the domain, reused in several places, and combined in many ways.
- Rules must be evaluated both in memory and as database queries, and must stay identical.
- Rules are configured at runtime (admin-defined audiences, alerting rules, access policies).
- You need explanations: which sub-rule failed.

## When NOT to use
- A single condition used in one place: a function or a property (`customer.is_eligible`) is enough.
- The combinations are fixed and few: a few named predicate functions composed with `all()`/`any()` give the same benefit without a class hierarchy.
- Your ORM already gives composable expressions (SQLAlchemy `and_`, Django `Q`, Prisma `where`): name and reuse those expressions directly instead of wrapping them.
- Performance-critical filtering over large in-memory data: object-per-rule evaluation is slower than vectorized filters; use the database or NumPy/pandas.

## Example — Python
Specs as small frozen dataclasses with operator overloading, each able to produce both an in-memory check and a SQLAlchemy expression.

```python
from __future__ import annotations
from dataclasses import dataclass
from decimal import Decimal
from sqlalchemy import ColumnElement, and_, not_, or_
from app.models import Customer   # SQLAlchemy mapped class

class Spec:
    def is_satisfied_by(self, c: Customer) -> bool: raise NotImplementedError
    def to_sql(self) -> ColumnElement[bool]: raise NotImplementedError
    def __and__(self, other: Spec) -> Spec: return And(self, other)
    def __or__(self, other: Spec) -> Spec: return Or(self, other)
    def __invert__(self) -> Spec: return Not(self)

@dataclass(frozen=True)
class And(Spec):
    a: Spec; b: Spec
    def is_satisfied_by(self, c: Customer) -> bool: return self.a.is_satisfied_by(c) and self.b.is_satisfied_by(c)
    def to_sql(self) -> ColumnElement[bool]: return and_(self.a.to_sql(), self.b.to_sql())

@dataclass(frozen=True)
class Or(Spec):
    a: Spec; b: Spec
    def is_satisfied_by(self, c: Customer) -> bool: return self.a.is_satisfied_by(c) or self.b.is_satisfied_by(c)
    def to_sql(self) -> ColumnElement[bool]: return or_(self.a.to_sql(), self.b.to_sql())

@dataclass(frozen=True)
class Not(Spec):
    a: Spec
    def is_satisfied_by(self, c: Customer) -> bool: return not self.a.is_satisfied_by(c)
    def to_sql(self) -> ColumnElement[bool]: return not_(self.a.to_sql())

@dataclass(frozen=True)
class IsActive(Spec):
    def is_satisfied_by(self, c: Customer) -> bool: return c.status == "active"
    def to_sql(self) -> ColumnElement[bool]: return Customer.status == "active"

@dataclass(frozen=True)
class MinLifetimeSpend(Spec):
    amount: Decimal
    def is_satisfied_by(self, c: Customer) -> bool: return c.lifetime_spend >= self.amount
    def to_sql(self) -> ColumnElement[bool]: return Customer.lifetime_spend >= self.amount

@dataclass(frozen=True)
class IsBanned(Spec):
    def is_satisfied_by(self, c: Customer) -> bool: return c.banned
    def to_sql(self) -> ColumnElement[bool]: return Customer.banned.is_(True)

FREE_SHIPPING = IsActive() & ~IsBanned() & MinLifetimeSpend(Decimal("500"))
# in memory:  FREE_SHIPPING.is_satisfied_by(customer)
# in the DB:  session.scalars(select(Customer).where(FREE_SHIPPING.to_sql()))
```

If you never need `to_sql`, drop the classes: `type Pred = Callable[[Customer], bool]` plus `def both(a, b): return lambda c: a(c) and b(c)` is the whole pattern.

## Example — TypeScript
A functional form: a spec is a typed predicate with combinators. Useful for client-side filtering and route/feature guards.

```typescript
export type Spec<T> = ((candidate: T) => boolean) & { readonly label: string };

export const spec = <T>(label: string, fn: (t: T) => boolean): Spec<T> =>
  Object.assign((t: T) => fn(t), { label });

export const and = <T>(...specs: Spec<T>[]) =>
  spec<T>(specs.map(s => s.label).join(' AND '), t => specs.every(s => s(t)));
export const or = <T>(...specs: Spec<T>[]) =>
  spec<T>(`(${specs.map(s => s.label).join(' OR ')})`, t => specs.some(s => s(t)));
export const not = <T>(s: Spec<T>) => spec<T>(`NOT ${s.label}`, t => !s(t));

/** Explain which leaf rules failed (for "why am I not eligible?" UI). */
export const failing = <T>(t: T, ...specs: Spec<T>[]) => specs.filter(s => !s(t)).map(s => s.label);

interface Customer { status: 'active' | 'closed'; banned: boolean; lifetimeSpend: number; tier: 'basic' | 'premium' }

const isActive = spec<Customer>('active', c => c.status === 'active');
const isBanned = spec<Customer>('banned', c => c.banned);
const isPremium = spec<Customer>('premium', c => c.tier === 'premium');
const bigSpender = spec<Customer>('spend >= 500', c => c.lifetimeSpend >= 500);

export const freeShipping = and(isActive, not(isBanned), or(isPremium, bigSpender));

// Angular: customers().filter(freeShipping) inside a computed();
// failing(customer, isActive, not(isBanned), or(isPremium, bigSpender)) -> ['premium OR spend >= 500']
```

## In the wild
- SQLAlchemy column expressions and Django `Q` objects (`Q(active=True) & ~Q(banned=True)`) are specifications for the DB side.
- Spring Data JPA `Specification<T>` with `and`/`or`/`not`; .NET `Ardalis.Specification`.
- Python: `operator`/`functools` for predicate composition; Pydantic validators for validation-only uses.
- Policy engines (OPA/Rego, AWS IAM conditions, Cedar) are specifications as a DSL.
- Elasticsearch `bool` query (`must`, `should`, `must_not`) is a serialized composite spec.

## Trade-offs
| Pros | Cons |
|---|---|
| Business rules named, reused and unit-testable | Class explosion if every condition becomes a spec |
| One rule definition for memory and DB | Dual implementations (`is_satisfied_by` / `to_sql`) can drift |
| Composable at runtime (admin-defined rules) | Translating arbitrary specs to SQL gets hard (joins, aggregates) |
| Explainability: report failing leaves | Indirection for simple conditions |

## Pitfalls
- **Drift between in-memory and SQL versions** (NULL handling, case sensitivity, time zones). Fix: a parametrized test that runs each spec both ways over the same fixture rows and compares.
- **Specs that load the whole table and filter in Python.** Fix: push down via `to_sql`/ORM expressions; fall back to memory only for small sets.
- **Leaking ORM into the domain.** Fix: keep leaf specs in the domain and put SQL translation in an infrastructure visitor/mapper if strict layering matters.
- **Every `if` becomes a spec.** Fix: only extract rules the business names and reuses.
- **Specs with side effects or I/O** (calling an API inside `is_satisfied_by`). Fix: specs are pure; load data first.
- **Python `and`/`or` keywords on spec objects.** They use truthiness, not `__and__`. Fix: use `&`, `|`, `~`, and parenthesize because `&` binds tighter than comparisons.

## Refactoring path
- **Introduce it:** (1) find the duplicated condition and give it a name as a function; (2) replace all copies with calls; (3) when combinations multiply or the DB needs the same rule, promote functions to spec objects with combinators; (4) add `to_sql` and the dual-execution test; (5) collapse `find_x_y_z` repository methods into `find(spec)`.
- **Remove it / simplify:** if specs are never combined or reused, inline them back into named functions or model properties; if only the DB side is used, keep plain ORM expressions as module-level constants.

## Related patterns
- [Composite](../structural/composite.md) — `And`/`Or`/`Not` are a composite of specs.
- [Interpreter](interpreter.md) — a spec tree is an AST for a tiny boolean rule language; use Interpreter when rules are parsed from text.
- [Query Object](../data-access/query-object.md) — focuses on building queries; a spec is a rule that can also produce a query.
- [Repository](../data-access/repository.md) — `repo.find(spec)` keeps the repository interface small.
- [Strategy](strategy.md) — a spec is a strategy that returns a boolean, but specs are combined, strategies are swapped.
- [Chain of Responsibility](chain-of-responsibility.md) — for sequential rule handling with early exit instead of boolean composition.
