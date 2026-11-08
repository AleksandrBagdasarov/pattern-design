# Property-Based Testing

**Category:** Testing · **Also known as:** PBT, generative testing, QuickCheck-style testing · **Related:** Data-Driven Testing, Golden Master, Arrange-Act-Assert

## Intent
State invariants that must hold for *all* valid inputs, let a framework generate hundreds of inputs (including nasty edge cases), and automatically shrink any failure to a minimal counterexample.

## Problem
- Example tests only cover the cases you thought of; bugs live in the ones you did not (empty lists, unicode, negative zero, duplicates, huge numbers).
- Code with a clear algebraic contract — serialization round-trips, sorting, money arithmetic, parsers, state machines — is under-tested by a handful of examples.
- Two implementations must agree (old vs new, optimized vs naive) over the whole input space.

Scenario: `split_payment(total_cents, n)` must split a bill into `n` shares. Examples for 100/3 pass. Hypothesis finds that for `total=1, n=3` the shares sum to 0 because of floor division — no human wrote that case.

## Solution and structure
```
strategy/arbitrary ──generates──▶ inputs ──▶ SUT ──▶ property check (bool / assert)
        ▲                                                │ fail
        └──────────── shrinker: minimize failing input ◀─┘ ──▶ report + replay seed
```
Common property shapes:
- **Round-trip:** `decode(encode(x)) == x` (JSON, Pydantic model ↔ dict, URL params).
- **Invariant:** output always satisfies a condition (sum preserved, sorted, no negatives).
- **Oracle / differential:** `fast(x) == naive(x)` or `new(x) == legacy(x)`.
- **Idempotence:** `f(f(x)) == f(x)` (normalize, dedupe, upsert).
- **Metamorphic:** relation between outputs for related inputs (`search(q + filter) ⊆ search(q)`).
- **Stateful / model-based:** random command sequences against the SUT and a simple model.

## When to use
- Pure logic with clear invariants: parsers, serializers, money, date math, collections, encoders.
- Refactors and optimizations (differential property against the old implementation).
- Validation code: generate valid and invalid inputs, check the validator agrees with a simple spec.
- Stateful components (caches, carts, queues) via rule-based state machines.

## When NOT to use
- You cannot state a property without re-implementing the SUT (`assert f(x) == same_formula(x)` is circular).
- Behavior is mostly I/O orchestration with little logic: example tests with fakes are clearer.
- Slow SUT (seconds per call): 100 examples make the suite slow; limit `max_examples` or test the pure core only.
- Communicating specific business rules to readers: keep a few named examples next to properties.

## Example — Python
```python
from decimal import Decimal
from hypothesis import given, strategies as st, example
from pydantic import BaseModel

def split_payment(total_cents: int, n: int) -> list[int]:
    base, remainder = divmod(total_cents, n)
    return [base + (1 if i < remainder else 0) for i in range(n)]

@given(total=st.integers(min_value=0, max_value=10**9), n=st.integers(min_value=1, max_value=50))
@example(total=1, n=3)                       # pin a known tricky case
def test_split_preserves_total_and_is_fair(total: int, n: int) -> None:
    shares = split_payment(total, n)
    assert sum(shares) == total
    assert len(shares) == n
    assert max(shares) - min(shares) <= 1

class Invoice(BaseModel):
    number: str
    amount: Decimal
    tags: list[str]

invoices = st.builds(
    Invoice,
    number=st.text(min_size=1, max_size=20),
    amount=st.decimals(min_value=0, max_value=10**6, places=2, allow_nan=False),
    tags=st.lists(st.text(max_size=10), max_size=5),
)

@given(invoices)
def test_invoice_json_round_trip(inv: Invoice) -> None:
    assert Invoice.model_validate_json(inv.model_dump_json()) == inv
```
Hypothesis stores failing examples in `.hypothesis/` and replays them first on the next run. Use `@settings(max_examples=…, deadline=…)` to tune, and `hypothesis.stateful.RuleBasedStateMachine` for model-based tests.

## Example — TypeScript
```ts
import { describe, it, expect } from 'vitest';
import fc from 'fast-check';
import { toQueryParams, fromQueryParams, type Filters } from './filters';

const filtersArb: fc.Arbitrary<Filters> = fc.record({
  search: fc.string({ maxLength: 30 }),
  status: fc.constantFrom('open', 'closed', 'all'),
  page: fc.integer({ min: 1, max: 1000 }),
  tags: fc.uniqueArray(fc.stringMatching(/^[a-z]{1,8}$/), { maxLength: 5 }),
});

describe('filters ↔ URL', () => {
  it('round-trips through query params', () => {
    fc.assert(
      fc.property(filtersArb, (filters) => {
        expect(fromQueryParams(toQueryParams(filters))).toEqual(filters);
      }),
    );
  });

  it('sortBy is idempotent and preserves length', () => {
    fc.assert(
      fc.property(fc.array(fc.record({ id: fc.nat(), price: fc.integer() })), (rows) => {
        const once = sortByPrice(rows);
        expect(sortByPrice(once)).toEqual(once);
        expect(once).toHaveLength(rows.length);
      }),
      { numRuns: 200 },
    );
  });
});

declare function sortByPrice<T extends { price: number }>(rows: T[]): T[];
```
On failure fast-check prints the shrunk counterexample and a `seed`/`path` to replay deterministically. `@fast-check/vitest` adds `test.prop([...])` sugar.

## In the wild
- Python: Hypothesis (with `hypothesis-jsonschema`, `schemathesis` for OpenAPI-driven API fuzzing of FastAPI apps).
- JS/TS: fast-check (used by Jest, Ramda, io-ts test suites), `@fast-check/vitest`.
- Haskell QuickCheck (origin), Erlang PropEr/QuickCheck for stateful systems, Rust `proptest`.

## Trade-offs
| Pros | Cons |
|---|---|
| Finds edge cases humans do not think of | Requires thinking in invariants; learning curve |
| Shrinking gives minimal, debuggable counterexamples | Slower than example tests; needs tuning in CI |
| One property replaces dozens of examples | Weak properties ("doesn't crash") give false confidence |
| Excellent for refactors via differential testing | Generators for complex domain objects take effort |

## Pitfalls
- **Over-filtering:** `assume()` / `.filter()` discarding most inputs makes Hypothesis give up (`FailedHealthCheck`). Build valid data with constrained strategies instead.
- **Restating the implementation** as the property. Prefer round-trip, invariant, or comparison with a deliberately naive oracle.
- **Non-reproducible CI failures:** commit the Hypothesis example database or print/pin seeds; add the shrunk case as an `@example` / regression unit test.
- **Floats:** NaN and ±0 break equality-based properties; exclude them deliberately or compare with tolerance.
- **Hidden global state** (clock, DB) between examples: Hypothesis runs many examples inside one test; function-scoped pytest fixtures are *not* reset per example.

## Refactoring path
- **Introduce it:** take an existing parametrized table, ask "what is true for every row?", write that as a property with a generator, keep the table rows as `@example`s.
- **Remove it / simplify:** if properties are slow or flaky, move them to a nightly job with higher `max_examples` and keep a few fast example tests in the PR suite.

## Related patterns
- [Data-Driven Testing](../testing/data-driven-testing.md) — hand-picked rows; complements generated inputs.
- [Golden Master](../testing/golden-master.md) — differential properties against legacy code are a generative golden master.
- [Arrange-Act-Assert](../testing/arrange-act-assert.md) — generation replaces Arrange; the property is the Assert.
