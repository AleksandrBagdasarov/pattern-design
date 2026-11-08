# Micro-Optimizations

**Category:** Optimization · **Also known as:** Hot-loop tuning, peephole optimization (by hand) · **Related:** Data-Oriented Design, SIMD, Object Recycling, Caching, Flyweight

## Intent
Make small, local code changes in a **measured** hot path — fewer allocations, fewer attribute lookups, better built-ins, cheaper data structures — to shave constant factors after algorithmic and I/O fixes are exhausted.

## Problem
- A profiler shows one function or loop consuming most CPU time (e.g. 60% in a CSV-to-model transform running over 5M rows).
- Algorithmic complexity is already right, I/O is not the bottleneck, and the code still misses its budget.
- Frontend: a long task (>50 ms) in a scroll or input handler causes jank; DevTools shows a specific function dominating.

The overriding rule: **profile first, change one thing, measure again**. Most code is not hot. Knuth's "premature optimization" quote is about the 97% that isn't.

Tools:
- Python: `cProfile` + `snakeviz`/`pstats`, `py-spy record -o profile.svg` (sampling, production-safe), `scalene` (CPU + memory + "Python vs native" split), `line_profiler` for a single function, `timeit`/`pyperf` for microbenchmarks, `tracemalloc` / `memray` for allocations.
- JS/TS: Chrome DevTools Performance panel (flame chart, long tasks), Angular DevTools profiler (change detection cost), `node --cpu-prof`, `0x`, `clinic flame`, `mitata`/`tinybench` or Vitest `bench` for microbenchmarks.

## Solution and structure
Order of attack (stop as soon as the budget is met):
1. **Do less work**: avoid repeated computation, hoist invariants out of loops, short-circuit, cache.
2. **Better algorithm / data structure**: `set`/`dict` membership instead of list scans; `heapq` for top-k; `bisect` for sorted lookups. Usually the biggest win and not "micro".
3. **Use built-ins implemented in C**: `sum`, `min`, `sorted(key=)`, `str.join`, `collections.Counter`, `itertools`, `operator.attrgetter`; in JS, typed arrays and native methods in hot paths.
4. **Reduce interpreter overhead**: local variable aliases for globals/attributes in tight loops, comprehensions instead of `append` loops, `__slots__`, avoid per-item function calls and exceptions for control flow.
5. **Reduce allocations**: generators instead of intermediate lists, reuse buffers, avoid string concatenation in loops.
6. **Go native / vectorize** if still too slow: NumPy, Polars, Cython, Rust (PyO3), WebAssembly — see Data-Oriented Design and SIMD.

Always keep a benchmark (pytest-benchmark, Vitest `bench`) next to the optimized code so regressions are visible.

## When to use
- A profiler attributes significant time to the code you are changing, in a realistic workload.
- Hot inner loops: parsers, serializers, per-row transforms, render/diff loops, request middleware executed on every call.
- Library code used by many callers where constant factors multiply.

## When NOT to use
- Without a profile. Guessing is usually wrong (the slow part is the DB call, not the loop).
- When the cost is I/O, N+1 queries, lock contention, or network: fix those first.
- When the change hurts readability for a 2% gain in non-critical code.
- When a newer runtime already does it: CPython 3.11+ specializing interpreter made many old tricks (local aliasing, avoiding `.` lookups) much less valuable; V8 inlines small functions. Re-measure tricks on your current runtime.

## Example — Python
```python
# Profile: py-spy record -o prof.svg -- python -m etl.load orders.csv  → 70% in to_rows()
from dataclasses import dataclass
from decimal import Decimal
from operator import itemgetter

@dataclass(slots=True)                    # smaller objects, faster attribute access
class OrderRow:
    order_id: int
    sku: str
    cents: int

# Before: 2 dict lookups/row, list scan for membership, float→Decimal per row
def to_rows_slow(records: list[dict[str, str]], blocked: list[str]) -> list[OrderRow]:
    out = []
    for r in records:
        if r["sku"] not in blocked:                                    # O(len(blocked)) per row
            out.append(OrderRow(int(r["order_id"]), r["sku"], int(Decimal(r["price"]) * 100)))
    return out

# After: set membership, hoisted callables, comprehension, integer cents parsing
def to_rows_fast(records: list[dict[str, str]], blocked: list[str]) -> list[OrderRow]:
    blocked_set = frozenset(blocked)                                    # O(1) membership
    get = itemgetter("order_id", "sku", "price")
    def cents(price: str) -> int:
        whole, _, frac = price.partition(".")
        return int(whole) * 100 + int((frac + "00")[:2])               # avoids Decimal per row
    return [
        OrderRow(int(oid), sku, cents(price))
        for oid, sku, price in map(get, records)
        if sku not in blocked_set
    ]

# Verify: same output (property test) + speed (benchmark)
def test_fast_matches_slow(benchmark) -> None:                         # pytest-benchmark fixture
    rows = [{"order_id": str(i), "sku": f"S{i % 50}", "price": "12.30"} for i in range(10_000)]
    assert to_rows_fast(rows, ["S1", "S2"]) == to_rows_slow(rows, ["S1", "S2"])
    benchmark(to_rows_fast, rows, ["S1", "S2"])
```
The `cents` shortcut assumes non-negative prices with at most two decimals — exactly the kind of correctness trade-off micro-optimizations introduce. Encode it in a test.

## Example — TypeScript
```ts
// Profile: DevTools Performance → long task in filterRows() on each keystroke (10k rows)
interface Row { id: number; name: string; tags: string[] }

// Before: lowercases every row name on every keystroke; Array.includes inside the loop
export function filterSlow(rows: Row[], q: string, tags: string[]): Row[] {
  return rows.filter((r) => r.name.toLowerCase().includes(q.toLowerCase()) && r.tags.some((t) => tags.includes(t)));
}

// After: precompute once per data change, Set lookup, hoist invariants, plain loop
export interface IndexedRow extends Row { nameLc: string }
export const indexRows = (rows: Row[]): IndexedRow[] => rows.map((r) => ({ ...r, nameLc: r.name.toLowerCase() }));

export function filterFast(rows: IndexedRow[], q: string, tags: ReadonlySet<string>): Row[] {
  const needle = q.toLowerCase();
  const out: Row[] = [];
  for (let i = 0; i < rows.length; i++) {
    const r = rows[i]!;
    if (!r.nameLc.includes(needle)) continue;          // cheap check first
    for (const t of r.tags) if (tags.has(t)) { out.push(r); break; }
  }
  return out;
}

// filter.bench.ts — run with `vitest bench`
import { bench, describe } from 'vitest';
const data = Array.from({ length: 10_000 }, (_, i) => ({ id: i, name: `Order ${i}`, tags: [`t${i % 20}`] }));
const indexed = indexRows(data);
describe('filter', () => {
  bench('slow', () => { filterSlow(data, 'order 9', ['t3', 't4']); });
  bench('fast', () => { filterFast(indexed, 'order 9', new Set(['t3', 't4'])); });
});
```
In Angular, `indexRows` belongs in a `computed()` over the data signal, so it reruns only when data changes, not per keystroke. Often the bigger UI win is not the loop at all but rendering fewer rows (virtual scroll, `track` in `@for`).

## In the wild
- CPython's own speedups (3.11 specializing adaptive interpreter, 3.13 experimental JIT, free-threaded build) — check current release notes before hand-tuning.
- `orjson`/`msgspec` vs `json`, `pydantic-core` (Rust) — replacing hot code with native libraries is often the best micro-optimization.
- V8 hidden classes and inline caches (keep object shapes consistent), Angular `track` in `@for`, OnPush/zoneless change detection.

## Trade-offs
| Pros | Cons |
|---|---|
| Measurable wins in genuinely hot code | Readability and maintainability cost |
| No architectural change needed | Gains are constant-factor; often small |
| Benchmarks document performance intent | Runtime upgrades can erase or invert gains |
| Can defer expensive rewrites to native code | Easy to introduce subtle correctness changes |

## Pitfalls
- **Optimizing without a profile** or with an unrealistic micro-benchmark (tiny inputs, JIT not warmed up, dead-code elimination in JS benches).
- **Behavior changes** hidden in speedups (float vs Decimal rounding, Unicode case folding vs `toLowerCase`). Guard with differential/property tests.
- **Optimizing the wrong layer:** the loop is fast, but it runs 100× because of a missing memo or repeated change detection.
- **Stale folklore:** "use `map` not comprehension", "cache `len()`" — measure on your interpreter version.
- **No regression guard:** add `pytest-benchmark --benchmark-compare` or Vitest `bench` to CI for critical paths.

## Refactoring path
- **Introduce it:** profile realistic load → pick the top frame → write a benchmark + equivalence test → apply one change → re-measure → keep only changes with significant wins.
- **Remove it / simplify:** when code is no longer hot (profile again after changes) or a runtime/library upgrade makes the straightforward version as fast, revert to the readable version, keeping the benchmark.

## Related patterns
- [Data-Oriented Design](../optimization/data-oriented-design.md) — the next step when per-object overhead dominates.
- [SIMD](../optimization/simd.md) — vectorize numeric hot loops instead of tuning them element by element.
- [Object Recycling](../optimization/object-recycling.md) — when allocation/GC is the measured cost.
- [Caching](../optimization/caching.md) — "do less work" by not repeating it at all.
- [Flyweight](../structural/flyweight.md) — share immutable state to cut memory per object.
