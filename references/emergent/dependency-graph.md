# Dependency Graph

**Category:** Emergent / modern · **Also known as:** DAG, topological ordering, dependency resolution, reactive graph · **Related:** Dependency Injection, Data Pipeline, Observer, Scheduler, Lazy Load

## Intent
Model units of work (tasks, modules, services, computed values) as nodes with explicit "depends on" edges, then derive execution order, parallelism, invalidation and cycle detection from the graph instead of hardcoding them.

## Problem
- Ordering is encoded by hand: a `setup()` that calls ten initializers "in the right order", and every new one has to be slotted in by guesswork.
- Scenario: a reporting job runs `load_customers → load_orders → join → enrich → export`. Someone adds `load_fx_rates`; it must run before `enrich`, but nobody knows whether it can run in parallel with `load_orders`. Re-running after a failure recomputes everything.
- Scenario: a UI derives `total`, `tax` and `discountedTotal` from a cart; manual recomputation in event handlers misses one path and the UI shows stale numbers.
- Smells: circular import or circular dependency errors discovered at runtime; "run this script before that one" in a README; recomputing everything because you can't tell what changed.

## Solution and structure
Declare nodes and their dependencies; an engine does the rest.

```
 load_customers ─┐
                 ├─► join ─► enrich ─► export
 load_orders ────┘           ▲
 load_fx_rates ──────────────┘
```
- **Node**: a task/provider/computed value with an identity.
- **Edges**: declared dependencies (explicit list, function parameters, or tracked reads).
- **Resolver**: topological sort (Kahn's algorithm or DFS) gives a valid order and detects cycles; nodes with in-degree 0 at the same time can run in parallel.
- **Invalidation (optional)**: when a node changes, only its downstream nodes are marked dirty and recomputed (incremental builds, reactive signals).

Two flavours: **static** graphs declared up front (build systems, Airflow DAGs, DI containers) and **dynamic** graphs discovered at runtime by tracking reads (signals, spreadsheets, MobX).

## When to use
- Tasks with partial ordering where some can run in parallel (builds, ETL, migrations, CI jobs).
- Initialization of many components with inter-dependencies (DI containers, plugin loading).
- Derived state that must stay consistent and recompute minimally (spreadsheets, signals, incremental compilers).
- Package/version resolution and change impact analysis ("which services are affected by this schema change?").

## When NOT to use
- A linear sequence of 3–5 steps: just call them in order. A graph engine adds nothing.
- Only 2–3 derived values in a component: a `computed()` or a property is the graph already — don't build your own.
- When the framework already resolves it (FastAPI `Depends`, Angular DI, Airflow, Make): declare edges, don't re-implement the sort.
- Graphs with cycles by nature (feedback loops, workflows with retries) need a state machine or Saga, not a DAG.

## Example — Python
`graphlib` is in the stdlib (3.9+) and supports parallel scheduling directly.

```python
import asyncio
from collections.abc import Awaitable, Callable
from graphlib import TopologicalSorter, CycleError

Task = Callable[[dict[str, object]], Awaitable[object]]

async def run_dag(tasks: dict[str, Task], deps: dict[str, set[str]]) -> dict[str, object]:
    sorter = TopologicalSorter(deps)
    try:
        sorter.prepare()                      # raises CycleError early
    except CycleError as e:
        raise ValueError(f"dependency cycle: {e.args[1]}") from e
    results: dict[str, object] = {}
    running: dict[asyncio.Task[object], str] = {}
    while sorter.is_active():
        for name in sorter.get_ready():       # all nodes whose deps are done
            running[asyncio.create_task(tasks[name](results))] = name
        done, _ = await asyncio.wait(running, return_when=asyncio.FIRST_COMPLETED)
        for t in done:
            name = running.pop(t)
            results[name] = t.result()        # re-raises task failure
            sorter.done(name)
    return results

deps = {
    "join": {"load_customers", "load_orders"},
    "enrich": {"join", "load_fx_rates"},
    "export": {"enrich"},
}
```
Pass the `tasks` mapping with one coroutine per node; `load_*` have no entry in `deps` keys but appear as predecessors, which `graphlib` accepts.

## Example — TypeScript
Angular signals are a dynamic dependency graph: edges are discovered by reads, and only dirty nodes recompute, lazily.

```ts
import { Injectable, computed, signal } from '@angular/core';

export interface Line { sku: string; price: number; qty: number }

@Injectable({ providedIn: 'root' })
export class CartStore {
  readonly lines = signal<Line[]>([]);
  readonly taxRate = signal(0.2);
  readonly couponPct = signal(0);

  // edges: subtotal ← lines; discounted ← subtotal, couponPct; total ← discounted, taxRate
  readonly subtotal = computed(() => this.lines().reduce((s, l) => s + l.price * l.qty, 0));
  readonly discounted = computed(() => this.subtotal() * (1 - this.couponPct() / 100));
  readonly total = computed(() => +(this.discounted() * (1 + this.taxRate())).toFixed(2));

  add(line: Line): void {
    this.lines.update(ls => [...ls, line]);   // new array: graph sees the change
  }
}
```
Changing `taxRate` recomputes only `total`; `subtotal` and `discounted` stay memoized. Reading a computed inside itself throws a cycle error.

## In the wild
- Python: `graphlib.TopologicalSorter`, FastAPI `Depends` (resolves a DI DAG per request, caches each dependency once per request), pytest fixtures, Airflow/Prefect/Dagster DAGs, Alembic revision graph, `uv`/pip resolvers.
- JS/TS: Angular signals and DI injector tree, Nx/Turborepo task graphs, webpack/Vite module graph, MobX, Excel recalculation.
- Build/infra: Make, Bazel, Terraform resource graph, Kubernetes init containers (coarse), dbt model DAG.

## Trade-offs
| Pros | Cons |
|---|---|
| Order and parallelism derived, not hand-maintained | Indirection: harder to see "what runs when" without tooling |
| Cycles detected early and explicitly | Dynamic graphs can hide accidental dependencies |
| Incremental recompute: only affected nodes rerun | Caching/invalidation logic is subtle |
| Adding a node doesn't require editing a central sequence | Failure handling (skip downstream? retry?) must be designed |

## Pitfalls
- Hidden edges: a task reads a file or global that isn't declared, so the order is "accidentally right" until parallelism exposes it. Make inputs explicit.
- Mutating a signal's array/object in place: the graph sees no change. Always produce new references or use `update` with a copy.
- Side effects inside computed nodes (HTTP calls, writes) — they may run zero or many times. Keep nodes pure; put effects in `effect()` or the task runner.
- No failure policy: one failed node should cancel or skip its descendants, not leave the run hanging.
- Diamond dependencies recomputed twice in naive push-based observers (glitches); use a graph engine that orders recompute topologically (signals do).

## Refactoring path
- **Introduce it:** list every step and what it reads/writes; write the edges as data; run a topological sort to validate the current hand order; then let the sorter drive execution; finally enable parallelism and incremental recompute.
- **Remove it / simplify:** if the graph is a straight line with no parallelism or caching benefit, inline it back into sequential calls; replace a home-grown engine with `graphlib`, the framework's DI, or an orchestrator.

## Related patterns
- [Dependency Injection](../behavioral/dependency-injection.md) — containers resolve a dependency graph of providers.
- [Data Pipeline](../emergent/data-pipeline.md) — pipelines are usually DAGs of stages.
- [Observer](../behavioral/observer.md) — push notification; a reactive graph adds ordering and memoization.
- [Scheduler](../concurrency/scheduler.md) — decides *when*; the graph decides *in what order*.
- [Lazy Load](../optimization/lazy-load.md) — pull-based graphs compute nodes only when read.
