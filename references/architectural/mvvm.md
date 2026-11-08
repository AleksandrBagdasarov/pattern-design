# Model-View-ViewModel (MVVM)

**Category:** Architectural · **Also known as:** Model-View-Binder · **Related:** MVC, MVP, Presentation Model, Observer, Property Change Listener

## Intent
Put all view state and presentation logic into a **ViewModel** that exposes observable properties and commands; the **View** declaratively binds to it, so the view has no imperative update code and the view model is testable without a UI.

## Problem
- Components push values into the DOM by hand (`el.textContent = …`), and state gets out of sync.
- Presentation logic ("disable Submit while saving", "show 3 of 120 selected") is entangled with the template and untestable.
- The same screen state is needed by two views (list + summary badge).
- Scenario: an order checkout page with a cart, a coupon field, async price recalculation and a disabled pay button. Imperative jQuery-style updates miss cases; tests require a browser.

## Solution and structure
```
  View (template) ──binds──► ViewModel (signals/observables + commands) ──calls──► Model (services, API)
        ▲                           │
        └───── change notification─┘   (data binding, no reference from VM to View)
```
- **Model:** domain data and services (HTTP clients, stores).
- **ViewModel:** state (signals, observables), derived state (computed), commands (methods). Knows nothing about the DOM.
- **View:** template with bindings; forwards events to commands.
- **Binder:** the framework mechanism (Angular templates + change detection, Vue reactivity, WPF bindings).

## When to use
- Frameworks with data binding: Angular, Vue, Svelte, Knockout, WPF/MAUI, SwiftUI, Jetpack Compose.
- Screens with significant derived state and async workflows.
- You want unit tests on presentation logic without rendering.

## When NOT to use
- Static or server-rendered pages: MVC is simpler.
- Tiny components with one input and one output: a separate view-model service is ceremony; the component class *is* the view model.
- Do not force a separate ViewModel class per component in Angular; extract one only when logic is shared or heavy.

## Example — Python
```python
# Python has no built-in binder; a UI-agnostic view model with change notification
# is still useful (Textual, PySide, or tests). Observers are plain callables.
from collections.abc import Callable
from dataclasses import dataclass, field
from decimal import Decimal

Listener = Callable[[str], None]

@dataclass
class CartViewModel:
    items: dict[str, Decimal] = field(default_factory=dict)
    coupon_pct: int = 0
    saving: bool = False
    _listeners: list[Listener] = field(default_factory=list, repr=False)

    def subscribe(self, fn: Listener) -> None:
        self._listeners.append(fn)

    def _changed(self, prop: str) -> None:
        for fn in self._listeners:
            fn(prop)

    @property
    def total(self) -> Decimal:                       # derived state
        subtotal = sum(self.items.values(), Decimal(0))
        return (subtotal * (100 - self.coupon_pct) / 100).quantize(Decimal("0.01"))

    @property
    def can_pay(self) -> bool:
        return bool(self.items) and not self.saving

    def add(self, sku: str, price: Decimal) -> None:  # command
        self.items[sku] = price
        self._changed("total")

    def apply_coupon(self, pct: int) -> None:
        self.coupon_pct = max(0, min(pct, 50))
        self._changed("total")

vm = CartViewModel()
vm.add("sku-1", Decimal("40"))
vm.apply_coupon(10)
assert vm.total == Decimal("36.00") and vm.can_pay
```

## Example — TypeScript
```ts
// Angular: the component class with signals is the ViewModel; the template is the View.
import { Component, computed, inject, signal } from '@angular/core';
import { CartApi } from './cart-api';                    // Model (HTTP service)

interface Line { sku: string; price: number }

@Component({
  selector: 'app-cart',
  template: `
    @for (line of lines(); track line.sku) { <p>{{ line.sku }} – {{ line.price }}</p> }
    <input type="number" [value]="couponPct()" (input)="setCoupon($any($event.target).valueAsNumber)" />
    <p>Total: {{ total() }}</p>
    <button [disabled]="!canPay()" (click)="pay()">Pay</button>
  `,
})
export class Cart {
  private api = inject(CartApi);
  protected readonly lines = signal<Line[]>([]);
  protected readonly couponPct = signal(0);
  protected readonly saving = signal(false);
  protected readonly total = computed(() => {
    const sub = this.lines().reduce((s, l) => s + l.price, 0);
    return Math.round(sub * (100 - this.couponPct())) / 100;
  });
  protected readonly canPay = computed(() => this.lines().length > 0 && !this.saving());

  setCoupon(pct: number) { this.couponPct.set(Math.max(0, Math.min(pct || 0, 50))); }
  async pay() {
    this.saving.set(true);
    try { await this.api.pay(this.lines()); this.lines.set([]); } finally { this.saving.set(false); }
  }
}
```
Do not mutate a signal's array in place (`lines().push(x)`); `set`/`update` with a new array, or OnPush/zoneless rendering will not see the change.

## In the wild
- Angular components + signals (`signal`, `computed`, `linkedSignal`, `resource` — the last two are newer APIs; check angular.dev for current status).
- Vue `ref`/`computed` with templates, Knockout observables, WPF `INotifyPropertyChanged`, SwiftUI `@Observable`, Android `ViewModel` + `StateFlow`.
- Python: Textual `reactive` attributes, Qt `Property` with `notify` signals.

## Trade-offs
| Pros | Cons |
|---|---|
| No manual DOM sync; state drives the view | Binding magic can hide performance problems |
| Presentation logic unit-testable | Debugging binding chains is harder |
| Derived state declared once (computed) | Large view models become god objects |
| Works naturally with reactive streams | Two-way binding can create update loops |

## Pitfalls
- **God ViewModel:** one component holding 30 signals. Split into child components or a feature store/facade service.
- **DOM access in the view model:** `ElementRef` manipulation inside logic breaks testability; use bindings or directives.
- **In-place mutation:** OnPush and signals compare by reference; always produce new objects/arrays.
- **Subscriptions not cleaned up:** prefer `toSignal`, `async` pipe or `takeUntilDestroyed()` over manual `subscribe`.
- **Business rules in the view model:** keep pricing rules in the backend/domain; the VM handles presentation state.

## Refactoring path
- **Introduce it:** replace imperative DOM updates with bound state; move fields into signals; turn derived values into `computed`; move event handlers into methods with no DOM access; then extract shared state into an injectable facade if two components need it.
- **Remove it / simplify:** if a separate view-model service backs a single trivial component, fold its state back into the component class.

## Related patterns
- [MVC](../architectural/mvc.md) — controller-driven; no binding.
- [MVP](../architectural/mvp.md) — presenter calls the view through an interface instead of binding.
- [Presentation Model](../ui/presentation-model.md) — Fowler's framework-neutral ancestor of MVVM.
- [Observer](../behavioral/observer.md) — the notification mechanism under data binding.
- [Property Change Listener](../behavioral/property-change-listener.md) — per-property notifications a binder consumes.
