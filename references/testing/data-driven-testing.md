# Data-Driven Testing

**Category:** Testing · **Also known as:** Parameterized Test, Table-Driven Test · **Related:** Property-Based Testing, Golden Master, Arrange-Act-Assert, BDD

## Intent
Write the test logic once and run it over a table of (input, expected output) rows, so adding a case means adding a row, not copying a test.

## Problem
- Ten near-identical tests differ only in the literal values: `test_vat_de`, `test_vat_fr`, `test_vat_pl`…
- Copy-paste tests drift: one gets a fix, the others do not.
- Edge cases (empty string, zero, boundary values) are missing because adding each costs a whole test function.
- One loop inside a single test hides which case failed and stops at the first failure.

Scenario: a shipping-cost function depends on country, weight band and membership. Business gives you a table of 20 rows. A parametrized test turns that table into 20 independent, individually-named test cases.

## Solution and structure
```
cases = [ (input…, expected), … ]   ──▶  one test function  ──▶  N independent test results
```
- **Case table**: inline literals, a list of dataclasses, or an external file (CSV/JSON/YAML) owned by domain experts.
- **Test template**: Arrange from the row, Act once, Assert against the row's expectation.
- **Case ids**: a readable name per row so failures say which case broke.

## When to use
- Pure functions or rules with many input/output examples: pricing, validation, parsing, mapping, permission matrices.
- Boundary-value and equivalence-class testing.
- The same contract tested against several implementations (parametrize over fixtures).
- Business-owned examples in a spreadsheet.

## When NOT to use
- Cases need different Arrange or Assert logic: forcing them into one template produces `if case.kind == …` branches inside the test. Write separate tests.
- The input space is large and you care about invariants rather than specific examples: use Property-Based Testing.
- Two or three cases with distinct meaning: separate, well-named tests read better.
- Giant cross-product parametrization (`3 × 5 × 4 × 6` = 360 cases) that runs slowly and tests nothing extra; choose representative combinations (pairwise).

## Example — Python
```python
from dataclasses import dataclass
from decimal import Decimal
import pytest
from shipping import shipping_cost, UnknownCountry

@dataclass(frozen=True)
class Case:
    country: str
    weight_kg: Decimal
    member: bool
    expected: Decimal

CASES = [
    Case("DE", Decimal("0.5"), False, Decimal("4.90")),
    Case("DE", Decimal("0.5"), True,  Decimal("0.00")),
    Case("DE", Decimal("2.0"), False, Decimal("6.90")),   # boundary: exactly 2 kg
    Case("DE", Decimal("2.01"), False, Decimal("9.90")),
    Case("PL", Decimal("0.5"), False, Decimal("7.50")),
]

@pytest.mark.parametrize("case", CASES, ids=lambda c: f"{c.country}-{c.weight_kg}kg-member={c.member}")
def test_shipping_cost(case: Case) -> None:
    assert shipping_cost(case.country, case.weight_kg, member=case.member) == case.expected

@pytest.mark.parametrize(
    "country, weight",
    [
        pytest.param("XX", Decimal("1"), id="unknown-country"),
        pytest.param("de", Decimal("1"), id="lowercase-code"),
    ],
)
def test_rejects_invalid_country(country: str, weight: Decimal) -> None:
    with pytest.raises(UnknownCountry):
        shipping_cost(country, weight, member=False)
```
Valid and invalid cases live in separate tests because their Assert step differs. `pytest.param(..., marks=pytest.mark.xfail)` marks known bugs per row.

## Example — TypeScript
```ts
import { describe, it, expect } from 'vitest';
import { FormControl } from '@angular/forms';
import { ibanValidator } from './iban.validator';

describe('ibanValidator', () => {
  it.each([
    { value: 'DE89370400440532013000', valid: true },
    { value: 'DE89 3704 0044 0532 0130 00', valid: true },   // spaces allowed
    { value: 'DE89370400440532013001', valid: false },       // bad checksum
    { value: '', valid: true },                               // empty is Validators.required's job
    { value: 'XX00', valid: false },
  ])('$value → valid=$valid', ({ value, valid }) => {
    const control = new FormControl(value, { validators: ibanValidator });
    expect(control.valid).toBe(valid);
  });

  it.for([['  ', 'whitespace'], ['DE89-3704', 'dash']] as const)(
    'rejects %s (%s)',
    ([value]) => {
      expect(ibanValidator(new FormControl(value))).toEqual({ iban: true });
    },
  );
});
```
`it.each` with objects gives `$field` interpolation in titles. `it.for` (Vitest 2+) passes the row as one argument and works with `test.extend` fixtures.

## In the wild
- pytest `@pytest.mark.parametrize`, parametrized fixtures (`@pytest.fixture(params=[...])`), `pytest-cases`.
- Vitest/Jest `it.each` / `describe.each` / `it.for`; Jasmine has no built-in equivalent (loops over `it`).
- Go table-driven tests (the canonical idiom); JUnit 5 `@ParameterizedTest` with `@CsvSource`.
- Gherkin `Scenario Outline` + `Examples:` tables in BDD tools.

## Trade-offs
| Pros | Cons |
|---|---|
| New case = one row; less duplication | One template can hide case-specific intent |
| Each row reported separately with its own id | Large tables become hard to review without good ids |
| Encourages boundary and edge cases | Branching inside the template is a common smell |
| Tables can come from domain experts | External data files drift from code and lose type checking |

## Pitfalls
- **Unreadable ids** (`test_x[case3]`): always supply `ids=` / title templates.
- **Logic in the test template** (`if expected is None: ...`): split into separate parametrized tests per outcome kind.
- **Expected values computed with the same formula as the SUT:** the test then proves nothing. Expected values must be literals from the spec.
- **Mutable shared rows:** a test mutates a dict from the table and pollutes later cases. Use frozen dataclasses or tuples.

## Refactoring path
- **Introduce it:** find tests that differ only in literals; extract the varying values into a list of rows, keep one function, add ids, delete the copies.
- **Remove it / simplify:** if the template grew conditionals, split rows by behavior into separate tests; if rows exploded to hundreds, switch to property-based generation plus a few named examples.

## Related patterns
- [Property-Based Testing](../testing/property-based-testing.md) — generated inputs and invariants instead of hand-picked rows.
- [Golden Master](../testing/golden-master.md) — runs a recorded input corpus; expectations come from the previous output.
- [BDD](../testing/bdd.md) — Scenario Outline is data-driven testing in Gherkin.
- [Arrange-Act-Assert](../testing/arrange-act-assert.md) — each row should map to one clean AAA run.
