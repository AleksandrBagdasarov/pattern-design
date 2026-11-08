# Data-Oriented Design

**Category:** Optimization · **Also known as:** DOD, struct-of-arrays (SoA), columnar layout · **Related:** SIMD, Entity Component System, Object Recycling, Flyweight, Micro-Optimizations

## Intent
Organize data around how it is accessed in bulk — contiguous, homogeneous arrays per field (columns) processed in batches — rather than around objects that bundle every field of one entity, so that CPU caches, vectorization and native libraries can do their work.

## Problem
- A list of 2M `Order` objects; computing revenue by region loops over Python objects, touching dict-backed attributes and pointer-chasing across the heap. It takes 20 s and 1.5 GB.
- Each object carries fields the hot computation never reads, wasting memory bandwidth.
- Per-object method calls (`order.total()`) prevent vectorization.
- Frontend: a chart/simulation with 100k points as `{x, y, color}` objects causes GC churn and slow frames.

Profile first: `py-spy`/`scalene` show time in the interpreter loop over objects; `memray`/`tracemalloc` show per-object overhead; Chrome DevTools Performance and Memory (heap snapshot object counts). If the work is I/O-bound, DOD will not help.

## Solution and structure
```
Array of structs (AoS)                    Struct of arrays (SoA / columnar)
[ {id, region, cents, status}, ... ]       ids:     [1, 2, 3, ...]        int64
                                           region:  [0, 2, 0, ...]        int8 (dictionary-encoded)
                                           cents:   [1250, 990, ...]      int64
                                           status:  [...]                 (not loaded if unused)
```
Principles:
- **Separate hot and cold fields**; the hot loop reads only the columns it needs.
- **Homogeneous, contiguous, typed arrays** (NumPy, Arrow, typed arrays) instead of heterogeneous objects.
- **Batch transforms** over whole columns instead of per-entity methods.
- **Encode** categorical data (dictionary/categorical types), use the narrowest numeric types.
- **Identity by index** rather than by object reference.

Python dissolves much of the manual work: Polars, pandas, NumPy, PyArrow and DuckDB are columnar engines. Using them *is* applying DOD.

## When to use
- Analytics, ETL, reporting, aggregations over large datasets.
- Simulations, games (ECS), particle systems, physics, charts with many points.
- Numeric or columnar workloads where the same operation applies to every element.
- Memory-bound workloads where per-object overhead dominates.

## When NOT to use
- Business logic over a handful of rich entities per request (a checkout, an order workflow): domain objects are clearer and the data is tiny.
- Highly heterogeneous data with per-entity polymorphic behavior.
- Workloads dominated by DB/network latency.
- When SQL can do it: push the aggregation to Postgres/DuckDB (a columnar engine already) instead of moving data into application memory.

## Example — Python
```python
# Before (AoS): 2M dataclass instances, Python-level loop  →  ~seconds, ~GBs
from dataclasses import dataclass
from collections import defaultdict

@dataclass(slots=True)
class Order:
    id: int
    region: str
    cents: int
    status: str

def revenue_by_region_aos(orders: list[Order]) -> dict[str, int]:
    out: defaultdict[str, int] = defaultdict(int)
    for o in orders:
        if o.status == "paid":
            out[o.region] += o.cents
    return dict(out)

# After (SoA / columnar): Polars reads only needed columns and runs vectorized, multi-threaded
import polars as pl

def revenue_by_region(path: str) -> dict[str, int]:
    df = (
        pl.scan_parquet(path)                                   # lazy: projection + predicate pushdown
        .filter(pl.col("status") == "paid")
        .group_by("region")
        .agg(pl.col("cents").sum())
        .collect()
    )
    return dict(zip(df["region"].to_list(), df["cents"].to_list()))

# Manual SoA with NumPy when you own the arrays
import numpy as np

def revenue_numpy(region_codes: np.ndarray, cents: np.ndarray, paid: np.ndarray, n_regions: int) -> np.ndarray:
    return np.bincount(region_codes[paid], weights=cents[paid], minlength=n_regions).astype(np.int64)
```
`np.bincount` with `weights` returns floats; for very large sums convert carefully or use `np.add.at` on an `int64` array to keep exact integer cents.

## Example — TypeScript
```ts
// Before: 100k objects {x, y, vx, vy}; per-frame GC and pointer chasing
// After: struct of arrays in typed arrays; one contiguous buffer per field
export class Points {
  readonly x: Float32Array; readonly y: Float32Array;
  readonly vx: Float32Array; readonly vy: Float32Array;
  readonly color: Uint32Array;                          // cold for the physics step
  count = 0;

  constructor(readonly capacity: number) {
    this.x = new Float32Array(capacity); this.y = new Float32Array(capacity);
    this.vx = new Float32Array(capacity); this.vy = new Float32Array(capacity);
    this.color = new Uint32Array(capacity);
  }

  add(x: number, y: number, vx: number, vy: number, rgba: number): number {
    const i = this.count++;
    this.x[i] = x; this.y[i] = y; this.vx[i] = vx; this.vy[i] = vy; this.color[i] = rgba;
    return i;                                           // identity = index
  }

  step(dt: number, width: number): void {               // touches only hot columns
    const { x, y, vx, vy } = this;
    for (let i = 0; i < this.count; i++) {
      x[i] = x[i]! + vx[i]! * dt;                       // `!` only needed with noUncheckedIndexedAccess
      y[i] = y[i]! + vy[i]! * dt;
      if (x[i]! < 0 || x[i]! > width) vx[i] = -vx[i]!;
    }
  }
}
```
Typed arrays can be transferred to a Web Worker with zero copy (`postMessage(buf, [buf.buffer])`) or uploaded directly to WebGL. Keep the Angular side to a signal with summary stats, not 100k reactive objects. Apache Arrow JS gives the same columnar layout for tabular data in the browser.

## In the wild
- Python: NumPy, pandas, Polars, PyArrow, DuckDB, Vaex; `array` module and `struct` for compact homogeneous storage.
- Databases/formats: Parquet, ORC, Arrow, ClickHouse, DuckDB, Snowflake, Redshift, BigQuery (columnar storage).
- Games/engines: Unity DOTS/ECS, Bevy ECS, EnTT; Mike Acton's "Data-Oriented Design and C++" (CppCon 2014).
- Web: typed arrays, WebGL/WebGPU buffers, Apache Arrow JS, Perspective (FINOS) for large grids.

## Trade-offs
| Pros | Cons |
|---|---|
| Orders-of-magnitude speedups for bulk operations | Loses object encapsulation and per-entity methods |
| Much lower memory per record | Index-based identity is error-prone (stale indices after deletes) |
| Enables SIMD/vectorization and multi-threading in native engines | Harder to express complex branching business rules |
| Columnar files/engines skip unneeded columns | Adding/removing entities requires compaction or free lists |

## Pitfalls
- **Converting back to objects inside the hot loop** (`df.iter_rows()`, `.apply(lambda …)`, `.to_list()` then loop): you lose the benefit. Stay in column expressions.
- **Wrong dtypes:** `object` dtype in pandas, float for money, int64 where int8 categories suffice. Choose types deliberately.
- **Stale indices** after swap-remove compaction. Use generational ids or rebuild index maps.
- **Applying DOD to small data:** 500 rows per request do not need Polars; overhead may exceed gains.
- **Moving data out of the DB just to aggregate it:** let SQL do it.

## Refactoring path
- **Introduce it:** profile → identify a bulk computation over many objects → extract only the needed fields into columns (or load directly into Polars/NumPy) → rewrite the computation as column operations → verify equality with the old version on sample data → measure.
- **Remove it / simplify:** if the dataset is small or the logic is mostly per-entity branching, return to domain objects; keep columnar processing at the analytics/reporting edge.

## Related patterns
- [SIMD](../optimization/simd.md) — contiguous columns are what makes vectorized instructions possible.
- [Entity Component System](../structural/entity-component-system.md) — DOD applied to game/simulation architecture.
- [Object Recycling](../optimization/object-recycling.md) — SoA removes most per-object allocations outright.
- [Flyweight](../structural/flyweight.md) — dictionary encoding of repeated values is the columnar analogue.
- [Micro-Optimizations](../optimization/micro-optimizations.md) — the step before restructuring data.
