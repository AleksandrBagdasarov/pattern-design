# PAC (Presentation-Abstraction-Control)

**Category:** UI · **Also known as:** Hierarchical MVC (HMVC, a close relative), agent hierarchy · **Related:** MVC, MVVM, Presentation Model, Mediator, Composite, Microservices

## Intent
Structure an interactive system as a tree of cooperating **agents**, each with its own Presentation (UI), Abstraction (data/model) and Control (mediator), where agents talk to each other only through their Control parts along the hierarchy.

## Problem
- A large dashboard or workspace consists of semi-independent widgets (orders chart, live alerts, inventory table, map), each with its own data source, refresh policy and UI.
- A single global MVC model becomes a god object; every widget depends on everything.
- Widgets must coordinate (selecting a region on the map filters the chart and table) without importing each other.
- Different teams own different widgets, maybe even ship them separately (micro-frontends).

Scenario: an operations dashboard. The top-level agent owns global filters (date range, region). Each child agent (Sales, Alerts, Inventory) owns its data fetching and view. When the map agent reports "region = EU", the top control broadcasts the new filter to the children; they never reference each other.

## Solution and structure
```
                  ┌───────────── Top-level agent ─────────────┐
                  │ P: shell layout   A: global filters  C ◀──┼── coordinates children
                  └──────────┬───────────────┬────────────────┘
                     C ◀─────┘       C ◀──────┘        C ◀───── (only Controls talk)
          ┌── Sales agent ──┐  ┌── Alerts agent ──┐  ┌── Map agent ──┐
          │ P  chart        │  │ P  list          │  │ P  map        │
          │ A  sales data   │  │ A  ws stream     │  │ A  geo data   │
          └─────────────────┘  └──────────────────┘  └───────────────┘
```
- **Presentation**: rendering and input for this agent only.
- **Abstraction**: the agent's data and domain access; no UI knowledge.
- **Control**: mediates between its P and A, and is the only part that communicates with parent and child agents.
- Communication goes up and down the tree (events up, state/commands down), not sideways.

## When to use
- Complex dashboards, IDE-like workspaces, admin consoles with many independent panels.
- Micro-frontend shells where each fragment must be autonomous but react to shared context.
- Systems where parts have different data sources/lifecycles (polling, websockets, static).

## When NOT to use
- Ordinary SPAs with a few pages: component trees with inputs/outputs plus a shared store already give you the useful part.
- When every widget needs most of the same data: a single store (NgRx/SignalStore) with selectors is simpler than duplicated per-agent abstractions.
- As vocabulary for its own sake: in Angular, "a feature component + its provided store + a parent that mediates" *is* PAC; you rarely need to name the layers explicitly.

## Example — Python
A server-side (or Textual TUI) dashboard: each agent has a Control that receives context from its parent.
```python
from dataclasses import dataclass, field
from typing import Protocol
from datetime import date

@dataclass(frozen=True)
class Context:
    start: date
    end: date
    region: str | None = None

class Agent(Protocol):
    def on_context(self, ctx: Context) -> None: ...      # downward
    def render(self) -> str: ...

class SalesAgent:
    def __init__(self, repo: "SalesRepo") -> None:
        self._repo = repo                                  # Abstraction access
        self._rows: list[tuple[str, int]] = []
    def on_context(self, ctx: Context) -> None:            # Control
        self._rows = self._repo.totals(ctx.start, ctx.end, ctx.region)
    def render(self) -> str:                               # Presentation
        return "\n".join(f"{day}: {cents / 100:.2f}" for day, cents in self._rows)

@dataclass
class DashboardAgent:
    children: list[Agent]
    ctx: Context = field(default_factory=lambda: Context(date.today(), date.today()))

    def select_region(self, region: str) -> None:          # event bubbled up from MapAgent
        self.ctx = Context(self.ctx.start, self.ctx.end, region)
        for child in self.children:                        # broadcast down; no sibling links
            child.on_context(self.ctx)

    def render(self) -> str:
        return "\n---\n".join(c.render() for c in self.children)

class SalesRepo(Protocol):
    def totals(self, start: date, end: date, region: str | None) -> list[tuple[str, int]]: ...
```

## Example — TypeScript
```ts
import { Component, Injectable, inject, input, signal, computed, effect } from '@angular/core';
import { rxResource } from '@angular/core/rxjs-interop';
import { SalesApi } from './sales.api';

export interface DashContext { from: string; to: string; region: string | null }

@Injectable()                                         // Abstraction of the Sales agent
class SalesData {
  private readonly api = inject(SalesApi);
  readonly ctx = signal<DashContext | null>(null);
  readonly totals = rxResource({
    params: () => this.ctx() ?? undefined,           // undefined params = idle
    stream: ({ params }) => this.api.totals(params),
  });
}

@Component({                                          // Control + Presentation of the Sales agent
  selector: 'app-sales-panel',
  providers: [SalesData],                             // agent-private state
  template: `@for (t of data.totals.value() ?? []; track t.day) { <div>{{ t.day }}: {{ t.total }}</div> }`,
})
export class SalesPanel {
  protected readonly data = inject(SalesData);
  readonly context = input.required<DashContext>();
  constructor() {
    effect(() => this.data.ctx.set(this.context()));  // Control: parent context -> abstraction
  }
}

@Component({                                          // top-level agent
  selector: 'app-dashboard',
  imports: [SalesPanel /*, MapPanel, AlertsPanel */],
  template: `
    <!-- <app-map-panel (regionSelected)="region.set($event)" /> -->
    <app-sales-panel [context]="ctx()" />`,
})
export class Dashboard {
  readonly region = signal<string | null>(null);
  readonly ctx = computed<DashContext>(() => ({ from: '2026-01-01', to: '2026-01-31', region: this.region() }));
}
```
Children emit `output()` events up; the parent pushes context down via inputs — never sibling-to-sibling. Instead of the `effect`, you can have `SalesData` expose a `linkedSignal` or let the component pass `params: () => this.context()` directly. `rxResource` option names changed across Angular 19–20 (`request`/`loader` → `params`/`stream`); check angular.dev for your version.

## In the wild
- Angular/React component trees with component-scoped providers/stores and parent mediation; Angular micro-frontends via Native Federation with a shell coordinating remotes.
- HMVC in PHP frameworks (Kohana, CodeIgniter modules); Joëlle Coutaz's original PAC (1987) for interactive systems.
- Grafana/Kibana dashboards: panels with their own query + visualization, coordinated by dashboard-level variables.

## Trade-offs
| Pros | Cons |
|---|---|
| Agents are autonomous: own data, own lifecycle, own team | Communication through the hierarchy adds indirection and boilerplate |
| No sibling coupling; easy to add/remove panels | Duplicated data fetching if agents share data |
| Fits micro-frontends and plug-in panels | Hard to trace a change across many levels |
| Failures can be isolated per agent | Overkill for small UIs |

## Pitfalls
- **Siblings talking directly** (injecting each other's services): breaks the hierarchy. Route through the parent control or a scoped event bus.
- **Agent stores provided in root:** state is shared globally and agents lose autonomy. Provide per component.
- **Deep hierarchies** where context passes through five levels of inputs: provide a context service at the level that owns it and inject it below.
- **Each agent fetching the same data:** share via a parent-level cache or store when overlap is large.

## Refactoring path
- **Introduce it:** in a god-dashboard component, carve out one panel with its own component + provided data service; give it an input for shared context and outputs for events; repeat panel by panel.
- **Remove it / simplify:** if agents mostly mirror one shared state, collapse their abstractions into a single feature store with selectors and keep panels as presentational components.

## Related patterns
- [MVC](../architectural/mvc.md) — single triad; PAC is a hierarchy of triad-like agents with a mediating Control.
- [MVVM](../architectural/mvvm.md) — each agent internally is often MVVM in Angular.
- [Presentation Model](../ui/presentation-model.md) — the Abstraction + Control of an agent often take this form.
- [Mediator](../behavioral/mediator.md) — the Control role.
- [Composite](../structural/composite.md) — the agent tree structure.
- [Microservices](../architectural/microservices.md) — micro-frontends are PAC agents owned by separate teams.
