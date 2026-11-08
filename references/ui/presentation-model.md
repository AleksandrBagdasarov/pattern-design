# Presentation Model

**Category:** UI · **Also known as:** Application Model; close cousin of the MVVM ViewModel · **Related:** MVVM, MVP, View Helper, Observer, CQS, Facade

## Intent
Pull the state and behavior of a screen out of the view into a plain, UI-framework-agnostic class that the view merely mirrors, so the screen's logic can be tested without rendering anything.

## Problem
- Components/templates contain business-ish logic: when the Save button is enabled, which fields are visible, what the error banner says, how a filter combines with paging.
- Testing that logic requires rendering the component, querying the DOM and clicking.
- The same screen logic is needed in two views (desktop table and mobile list) and is duplicated.
- Component classes grow to 400 lines mixing HTTP, form wiring, derived state and DOM concerns.

Scenario: an order-search page has filters, pagination, a "Clear filters" button enabled only when a filter is set, and a summary line "12 of 340 orders". All of that is presentation state and can live in an `OrderSearchModel` with signals; the component only binds to it.

## Solution and structure
```
View (template/component)  ──binds to / forwards events──▶  Presentation Model
   thin: layout, DOM, a11y                                    view state (signals)
                                                              derived state (computed)
                                                              commands (methods)
                                                                   │ uses
                                                                   ▼
                                                     Domain services / API clients
```
- **Presentation Model** holds everything the view shows (including UI-only state like `isLoading`, `selectedIds`, `canSubmit`) and exposes intent methods (`applyFilter`, `nextPage`).
- **View** binds fields to the model and forwards user events; contains no decisions.
- Synchronization: Fowler's original leaves view↔model sync to the view; in Angular, signals and templates make that sync automatic, which is why Angular components + signal stores read as MVVM.

## When to use
- Screens with non-trivial interaction logic: enabling rules, derived summaries, multi-step filters, optimistic updates.
- You want fast unit tests of screen behavior with no DOM.
- Two or more views share the same screen logic.

## When NOT to use
- Simple display components (a card, a badge): a component with `input()` and a template is enough. An extra class is ceremony.
- When the logic is truly domain logic (pricing, eligibility): put it in the domain/service layer, not in a presentation model.
- If you already use a store (NgRx SignalStore, NgRx Store) with selectors per screen, that store is your presentation model; do not add another layer on top.

## Example — Python
Presentation models also apply to server-rendered or CLI/TUI apps. Here the model is a pure dataclass consumed by a Jinja template or Textual widget:
```python
from dataclasses import dataclass, field
from typing import Literal

Status = Literal["all", "open", "shipped"]

@dataclass
class OrderSearchModel:
    total: int = 0
    rows: list[dict] = field(default_factory=list)
    query: str = ""
    status: Status = "all"
    page: int = 1
    page_size: int = 20

    @property
    def can_clear(self) -> bool:
        return bool(self.query) or self.status != "all"

    @property
    def summary(self) -> str:
        return f"{len(self.rows)} of {self.total} orders"

    @property
    def has_next(self) -> bool:
        return self.page * self.page_size < self.total

    def clear(self) -> None:
        self.query, self.status, self.page = "", "all", 1

    def set_status(self, status: Status) -> None:
        self.status, self.page = status, 1     # changing a filter resets paging

def test_changing_filter_resets_page() -> None:
    m = OrderSearchModel(total=100, page=3)
    m.set_status("open")
    assert (m.page, m.can_clear) == (1, True)
```

## Example — TypeScript
```ts
import { Injectable, computed, inject, signal, Component } from '@angular/core';
import { rxResource } from '@angular/core/rxjs-interop';
import { OrdersApi, OrderStatus } from './orders.api';

@Injectable()   // provided per component: one model per screen instance
export class OrderSearchModel {
  private readonly api = inject(OrdersApi);
  readonly query = signal('');
  readonly status = signal<OrderStatus | 'all'>('all');
  readonly page = signal(1);

  readonly result = rxResource({
    params: () => ({ q: this.query(), status: this.status(), page: this.page() }),
    stream: ({ params }) => this.api.search(params),
  });
  readonly canClear = computed(() => this.query() !== '' || this.status() !== 'all');
  readonly summary = computed(() => {
    const r = this.result.value();
    return r ? `${r.rows.length} of ${r.total} orders` : '';
  });

  setStatus(s: OrderStatus | 'all') { this.status.set(s); this.page.set(1); }
  clear() { this.query.set(''); this.status.set('all'); this.page.set(1); }
}

@Component({
  selector: 'app-order-search',
  providers: [OrderSearchModel],
  template: `
    <p>{{ vm.summary() }}</p>
    <button [disabled]="!vm.canClear()" (click)="vm.clear()">Clear filters</button>`,
})
export class OrderSearch { protected readonly vm = inject(OrderSearchModel); }
```
`rxResource`/`resource` option names changed across Angular 19–20 (`request`/`loader` became `params`/`stream`); check angular.dev for your version. Tests can `TestBed.inject(OrderSearchModel)` with a fake `OrdersApi` and never render the template.

## In the wild
- Angular: services provided at component level, NgRx `signalStore` / `ComponentStore` per feature.
- Vue composables (`useOrderSearch()`), React custom hooks — same idea as functions.
- WPF/Avalonia ViewModels, Android Jetpack `ViewModel`, Fowler's Presentation Model (2004).
- Server-side: view models passed to Jinja/Django templates; FastAPI response models shaped for a specific page.

## Trade-offs
| Pros | Cons |
|---|---|
| Screen logic testable without DOM | Extra class per screen |
| View becomes declarative and thin | Risk of duplicating domain logic in the UI layer |
| Same model reusable across views | Sync boilerplate in frameworks without bindings |
| Clear place for UI-only state | Can turn into a god object for complex screens |

## Pitfalls
- **`providedIn: 'root'` for a per-screen model:** state survives navigation and leaks between instances. Provide it in the component's `providers`.
- **Model knows about the DOM** (`ElementRef`, `document`): breaks testability. Keep DOM work in the component or a directive.
- **Domain rules in the model:** move them to the domain/service layer; the model only adapts them for display.
- **Mutable arrays in signals** (`rows().push(x)`): change detection does not see it. Use `update(r => [...r, x])`.

## Refactoring path
- **Introduce it:** pick a fat component; move fields to signals in an `@Injectable()` model, move derived values to `computed`, move handlers to methods; the component injects it; then move tests from DOM-based to model-based.
- **Remove it / simplify:** if the model only forwards inputs to the template, inline it back into the component.

## Related patterns
- [MVVM](../architectural/mvvm.md) — Presentation Model with framework-provided data binding; Angular signals fit here.
- [MVP](../architectural/mvp.md) — presenter pushes into a passive view instead of the view pulling from a model.
- [View Helper](../ui/view-helper.md) — formatting of single values; the model handles screen state.
- [CQS](../ui/cqs.md) — model methods are commands; signals/computed are queries.
- [Observer](../behavioral/observer.md) — the mechanism behind signal/RxJS bindings.
- [Facade](../structural/facade.md) — a model often fronts several services for the view.
