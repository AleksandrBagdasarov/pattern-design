# Golden Master

**Category:** Testing · **Also known as:** Approval Testing, Snapshot Testing, Characterization Test · **Related:** Data-Driven Testing, Property-Based Testing, Fake Object, Strangler (refactoring)

## Intent
Capture the current output of a piece of code for a broad set of inputs, store it as the approved "master", and fail whenever future output differs — locking in behavior (right or wrong) before you refactor code you do not fully understand.

## Problem
- Legacy code with no tests, unclear requirements, and many branches: a 600-line `calculate_invoice()` nobody dares touch.
- You need to restructure it, but writing precise assertions requires knowing what it should do, which nobody does.
- Output is large and structured (a PDF-ready invoice dict, an HTML email, a CSV report, a rendered component), so hand-written assertions are tedious.

Scenario: the billing module must be split before adding a new tax regime. Run `calculate_invoice` over 500 recorded production-like orders, save each result to JSON, commit it. Refactor freely; any diff in the JSON is a behavior change you must explain.

## Solution and structure
```
inputs (recorded / generated) ──▶ SUT ──▶ output ──serialize──▶ received.json
                                                                   │ diff
                                            approved/master.json ◀─┘
```
1. **Input corpus**: real or combinatorially generated inputs that cover the branches (use coverage to check).
2. **Deterministic serialization**: stable key order, fixed clock/random seed, normalized floats and ids ("scrubbers").
3. **Approval step**: a human reviews the first output (or a diff) and promotes it to the master.
4. **Comparison**: test fails on diff and shows it.

## When to use
- Before refactoring untested legacy code (Feathers' characterization tests).
- Large or structured outputs where the whole output is the contract: reports, serializers, code generators, rendered templates, API response shapes.
- Migrating implementations (Python 2 → 3, old pricing engine → new) where "same output" is the requirement.

## When NOT to use
- New code with known requirements: write explicit assertions that state intent.
- Output that is inherently non-deterministic and cannot be scrubbed.
- UI component snapshots of full DOM trees that change on every styling tweak: people start updating snapshots blindly and the test stops protecting anything. Prefer targeted assertions or visual regression for real pixel checks.
- As a permanent test strategy for code you have already understood: replace with intent-revealing tests once the refactor is done.

## Example — Python
```python
import json
from decimal import Decimal
from pathlib import Path
import pytest
from billing.legacy import calculate_invoice   # the untested legacy function

CASES = json.loads(Path("tests/golden/orders.json").read_text())   # recorded inputs
MASTER_DIR = Path("tests/golden/approved")

def canonical(value: object) -> str:
    def default(o: object) -> object:
        if isinstance(o, Decimal):
            return str(o)
        raise TypeError(type(o))
    return json.dumps(value, sort_keys=True, indent=2, default=default)

@pytest.mark.parametrize("case", CASES, ids=lambda c: c["order_id"])
def test_invoice_matches_golden_master(case: dict, request: pytest.FixtureRequest) -> None:
    received = canonical(calculate_invoice(case))
    master = MASTER_DIR / f"{case['order_id']}.json"
    if request.config.getoption("--update-golden", default=False) or not master.exists():
        master.parent.mkdir(parents=True, exist_ok=True)
        master.write_text(received)          # review this diff in the PR before merging
        pytest.skip("golden master written")
    assert received == master.read_text()
```
Register `--update-golden` in `conftest.py` via `pytest_addoption`. Libraries such as `syrupy` (`assert result == snapshot`, `--snapshot-update`) or `approvaltests` give you this plus nicer diffs; prefer them over hand-rolling.

## Example — TypeScript
```ts
import { describe, it, expect, beforeEach, vi } from 'vitest';
import { TestBed } from '@angular/core/testing';
import { InvoiceView } from './invoice-view';
import orders from './fixtures/orders.json';

describe('InvoiceView golden master', () => {
  beforeEach(() => {
    vi.useFakeTimers();
    vi.setSystemTime(new Date('2026-01-15T00:00:00Z'));    // scrub the clock
  });

  it.each(orders.map((o) => [o.orderId, o] as const))('renders invoice %s', async (_id, order) => {
    const fixture = TestBed.createComponent(InvoiceView);
    fixture.componentRef.setInput('order', order);
    await fixture.whenStable();
    const text = (fixture.nativeElement as HTMLElement).textContent?.replace(/\s+/g, ' ').trim();
    expect(text).toMatchSnapshot();                          // stored in __snapshots__/
  });

  it('pure calculation, file-based master', async () => {
    const { calculateTotals } = await import('./legacy-totals');
    await expect(JSON.stringify(calculateTotals(orders), null, 2))
      .toMatchFileSnapshot('./golden/totals.json');
  });
});
```
Snapshotting normalized text content instead of raw HTML keeps the master stable across markup-only changes. Update with `vitest -u` only after reviewing the diff.

## In the wild
- Python: `syrupy`, `pytest-snapshot`, `approvaltests`, `pytest-regressions` (`data_regression`, `dataframe_regression`).
- JS/TS: Vitest/Jest `toMatchSnapshot`, `toMatchInlineSnapshot`, `toMatchFileSnapshot`; Playwright `toHaveScreenshot` for visual masters; Storybook + Chromatic.
- Michael Feathers, *Working Effectively with Legacy Code* (characterization tests); Emily Bache's Gilded Rose kata.

## Trade-offs
| Pros | Cons |
|---|---|
| Safety net for legacy code in hours, without understanding it | Locks in existing bugs as "approved" |
| Covers huge output surface with little test code | Failing diff tells you *what* changed, not *whether it is wrong* |
| Excellent for migrations and refactors | Brittle on non-deterministic data unless scrubbed |
| Reviewable diffs in PRs | Teams rubber-stamp `-u` updates, eroding value |

## Pitfalls
- **Blind updates:** treat snapshot changes as code changes; require reviewer sign-off and keep snapshots small enough to read.
- **Non-determinism:** timestamps, UUIDs, dict/set ordering, float formatting, locale. Freeze time, seed RNGs, sort keys, scrub ids.
- **Thin input corpus:** coverage report shows branches never hit. Add inputs until branch coverage of the legacy code is high, or combine with combinatorial generation.
- **Masters that never retire:** once refactored, replace golden tests with explicit unit tests and delete the masters.

## Refactoring path
- **Introduce it:** pick the seam (a function or endpoint), record inputs (logs, prod samples with PII scrubbed), freeze non-determinism, generate masters, check coverage, then refactor in small commits with the suite green.
- **Remove it / simplify:** after the code is understood and restructured, write intention-revealing tests per rule, then delete the snapshot files. Keep only snapshots whose whole output is genuinely the contract (serializers, codegen).

## Related patterns
- [Data-Driven Testing](../testing/data-driven-testing.md) — the input corpus is usually run as a parametrized table.
- [Property-Based Testing](../testing/property-based-testing.md) — differential testing ("old == new for all inputs") generalizes the master.
- [Fake Object](../testing/fake-object.md) — used to cut legacy code off from DB and network while characterizing it.
- [Arrange-Act-Assert](../testing/arrange-act-assert.md) — the Assert step is "matches master".
