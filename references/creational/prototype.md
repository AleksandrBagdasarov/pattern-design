# Prototype

**Category:** Creational · **Also known as:** Clone · **Related:** Builder, Fluent Builder, Factory, Object Mother, Flyweight, Memento

## Intent
Create new objects by copying a configured existing instance (the prototype) and adjusting the copy, instead of building each one from scratch.

## Problem
- Building an object is expensive or long-winded (parsed templates, a config tree assembled from several sources, a default dashboard with 20 widgets), and many objects differ only slightly.
- Callers need "one like this, but with X changed" and don't know the concrete class.
- Scenario: a reporting app ships preset report definitions (filters, columns, grouping). Users click "duplicate" and tweak one field. Rebuilding from a factory loses user customizations; copying the saved definition is the natural operation.
- Smell: code that reads all fields of an object just to pass them into a constructor with one changed.

## Solution and structure
- **Prototype**: an object that can produce a copy of itself (`clone()`, `copy.copy`, `structuredClone`, `dataclasses.replace`).
- **Registry** (optional): named prototypes (`"monthly-sales"`, `"default-dashboard"`) from which clients clone.
- **Client**: clones and adjusts, never needing the concrete class.

```
registry["monthly-sales"] ──clone──► copy ──adjust(period="Q3")──► new report
```

Key decision: **shallow vs deep copy**.
- Shallow: new top-level object, shared nested objects. Safe only if nested parts are immutable.
- Deep: everything copied. Correct for mutable trees, slower, and must handle cycles and non-copyable members (locks, sockets, DB sessions).

The cleanest modern form is **immutable values plus copy-with-changes**: if nothing is mutable, a shallow copy with overrides is always safe.

## When to use
- Many objects share most of their configuration and differ in a few fields.
- Construction is costly (parsing, I/O, heavy computation) and the result can be reused as a template.
- Users or code need "duplicate" semantics (documents, report definitions, form presets, game entities).
- Concrete class is not known to the caller (polymorphic copy).

## When NOT to use
- Construction is cheap: just call the constructor or factory.
- Objects own resources (connections, file handles, subscriptions). Copying them produces two owners of one resource. Create new ones instead.
- You need history/undo of state: use [Memento](../behavioral/memento.md) or event sourcing.
- Objects are immutable value objects: no clone method needed, use `dataclasses.replace` / spread.

## Example — Python
`dataclasses.replace` is Prototype for frozen dataclasses. `copy.deepcopy` is the general tool; customize with `__deepcopy__` when needed.

```python
from copy import deepcopy
from dataclasses import dataclass, field, replace
from datetime import date

@dataclass(frozen=True)
class ReportFilter:
    region: str | None = None
    product_line: str | None = None

@dataclass(frozen=True)
class ReportDefinition:
    name: str
    columns: tuple[str, ...]
    filters: ReportFilter
    period_start: date
    period_end: date
    group_by: tuple[str, ...] = ()

PRESETS: dict[str, ReportDefinition] = {
    "monthly-sales": ReportDefinition(
        name="Monthly sales",
        columns=("date", "region", "net_revenue"),
        filters=ReportFilter(),
        period_start=date(2026, 1, 1), period_end=date(2026, 1, 31),
        group_by=("region",),
    ),
}

def from_preset(key: str, **changes: object) -> ReportDefinition:
    # Frozen + tuples => a shallow copy with overrides is safe.
    return replace(PRESETS[key], **changes)

q3_emea = from_preset("monthly-sales", name="Q3 EMEA",
                      period_start=date(2026, 7, 1), period_end=date(2026, 9, 30),
                      filters=ReportFilter(region="EMEA"))

# Mutable legacy objects: deep-copy, never share nested dicts/lists.
@dataclass
class DashboardLayout:
    widgets: list[dict[str, object]] = field(default_factory=list)

DEFAULT_LAYOUT = DashboardLayout(widgets=[{"type": "kpi", "metric": "revenue"}])
user_layout = deepcopy(DEFAULT_LAYOUT)   # copy.copy would share the widgets list
```

## Example — TypeScript
`structuredClone` deep-copies plain data (Maps, Dates, nested arrays) but drops class prototypes and throws on functions; spread is a shallow copy.

```typescript
interface Widget { readonly id: string; type: 'kpi' | 'chart' | 'table'; config: Record<string, unknown>; }
interface DashboardTemplate { name: string; widgets: Widget[]; refreshSec: number; createdAt: Date; }

const DEFAULT_SALES: DashboardTemplate = {
  name: 'Sales overview',
  refreshSec: 60,
  createdAt: new Date('2026-01-01'),
  widgets: [
    { id: 'w1', type: 'kpi', config: { metric: 'revenue' } },
    { id: 'w2', type: 'chart', config: { metric: 'orders', range: '30d' } },
  ],
};

export function duplicateDashboard(src: DashboardTemplate, name: string): DashboardTemplate {
  const copy = structuredClone(src);              // deep: widgets and configs are independent
  copy.name = name;
  copy.createdAt = new Date();
  copy.widgets = copy.widgets.map(w => ({ ...w, id: crypto.randomUUID() })); // fresh identities
  return copy;
}

// Shallow spread shares nested arrays -- mutating shallow.widgets[0].config affects DEFAULT_SALES:
const shallow = { ...DEFAULT_SALES, name: 'oops' };
const mine = duplicateDashboard(DEFAULT_SALES, 'My sales');
```

## In the wild
- Python: `copy.copy`/`copy.deepcopy` with `__copy__`/`__deepcopy__` hooks; `dataclasses.replace`; Pydantic `model.model_copy(update={...}, deep=True)` (note `update` values are not validated); `dict.copy()`.
- JS: `structuredClone` (available in modern browsers and Node 17+), object/array spread, `Object.create(proto)` (JS's prototype chain is a different, language-level meaning of "prototype").
- Angular: `FormGroup` values copied via `getRawValue()` and spread; NgRx/signal state updates with spread (`update(s => ({ ...s, x }))`).
- Infrastructure: VM/AMI images and container images are prototypes; Kubernetes pod templates.

## Trade-offs
| Pros | Cons |
|---|---|
| Cheap creation of variants of an expensive object | Deep copy is slow on large graphs and must handle cycles |
| Caller doesn't need the concrete class | Shallow copies silently share mutable state |
| Natural fit for "duplicate" and preset features | Copying identity fields (IDs, timestamps) creates duplicates by accident |
| Pairs well with immutable data | Resource-owning members cannot be meaningfully cloned |

## Pitfalls
- **Shallow copy of mutable nested data** (`copy.copy`, `{...obj}`), then mutating the copy changes the prototype. Fix: freeze nested data or deep-copy.
- **Copied identity**: the clone keeps the original's `id`, `created_at`, version or ETag. Fix: reset identity fields explicitly in the clone function.
- **`structuredClone` on class instances**: you get plain objects without methods; functions throw `DataCloneError`. Fix: clone data, re-wrap in classes, or implement `clone()`.
- **Deep-copying ORM entities** copies session state and lazy-load proxies. Fix: build a new entity from the fields you want.
- **Pydantic `model_copy(update=...)`** skips validation. Fix: `Model.model_validate({**m.model_dump(), **changes})` when the changes come from input.

## Refactoring path
- **Introduce it:** find code that reconstructs an object from another's fields to change one value. Step 1: make the object immutable or decide deep-copy semantics. Step 2: add a `clone`/`with_` function (or use `replace`). Step 3: move presets into a registry and clone from it.
- **Remove it / simplify:** if cloning is only used to apply defaults, replace the registry with a factory function with keyword defaults. If clones are rare and cheap to build, delete the clone method.

## Related patterns
- [Builder](builder.md) — builds from parts; Prototype starts from a finished instance.
- [Fluent Builder](fluent-builder.md) — immutable fluent builders clone themselves on each step.
- [Factory](factory.md) — a factory may return clones from a registry of prototypes.
- [Object Mother](object-mother.md) — test fixtures often clone a canonical object and adjust it.
- [Flyweight](../structural/flyweight.md) — shares one instance instead of copying it.
- [Memento](../behavioral/memento.md) — snapshots state for restore, not for creating new objects.
