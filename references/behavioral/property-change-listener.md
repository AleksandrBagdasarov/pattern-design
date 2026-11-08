# Property Change Listener

**Category:** Behavioral · **Also known as:** Property Observer, PropertyChangeSupport (Java Beans), INotifyPropertyChanged (.NET), Observable property · **Related:** Observer, Event Aggregator, MVVM, Presentation Model, Mediator

## Intent
Notify interested parties when a *specific property* of an object changes, delivering the property name together with the old and new values, so they can react without polling.

## Problem
- A view model or form has many fields, and different parts of the UI or logic care about different fields: the shipping-cost panel cares about `country`, the tax line cares about `country` and `vat_id`, the submit button cares about `is_valid`.
- Generic Observer ("something changed") forces every listener to recompute everything or diff the whole object.
- Smells: polling (`setInterval(() => if (model.x !== last) ...)`), setters that directly call five unrelated components (`set country(v) { this._c = v; this.tax.recalc(); this.shipping.recalc(); ... }`), or listeners that fire on assignments that did not actually change the value.

Scenario: a checkout form where changing `country` must recompute shipping options and tax, and clear `postalCode` if the format no longer matches, while editing `notes` must trigger nothing.

## Solution and structure
The subject owns a registry of listeners keyed by property name (or a wildcard). Each setter compares old and new; if different, it stores the new value and fires a change event.

```
Subject (bean / view model)
  ├─ set country(v):  old = _country; if old != v: _country = v; fire("country", old, v)
  └─ listeners: { "country": [shipping, tax], "*": [dirtyTracker] }

Listener(event: {name, old, new}) -> None
```

Participants:
- **Subject**: holds state, owns listener registration and firing.
- **PropertyChangeEvent**: `{source, name, old_value, new_value}`.
- **Listener**: callable registered for one property or all.
- Optional **vetoable** variant: a listener may reject the change before it is applied (validation).

In modern frontends the pattern is largely provided by the reactivity system: Angular signals, Vue `ref`/`watch`, MobX observables. You get per-property dependency tracking without writing the registry.

## When to use
- View models / presentation models where different UI parts depend on different fields.
- Configuration objects whose individual settings must be propagated live (log level, feature flags).
- Dirty tracking, undo stacks, audit logs that need old and new values per field.
- Bridging a non-reactive domain object into a reactive UI.

## When NOT to use
- You are in a framework with fine-grained reactivity (Angular signals, Vue, Svelte, MobX): use `signal`/`computed`/`effect` instead of hand-rolled listener maps.
- Only one consumer exists: call it directly or pass a callback.
- Changes come in batches and listeners should see a consistent snapshot: per-property events fire mid-update. Use an immutable state + single "state changed" event (store, reducer) instead.
- Domain entities on the backend: emit explicit domain events (`OrderShipped`) rather than generic "field X changed", which leaks structure and loses intent.

## Example — Python
`__setattr__` on a dataclass gives every field change notification without writing setters. Equality check prevents no-op events.

```python
from collections import defaultdict
from collections.abc import Callable
from dataclasses import dataclass, field
from typing import Any

type Listener = Callable[[str, Any, Any], None]   # (name, old, new)

@dataclass
class Observable:
    _listeners: dict[str, list[Listener]] = field(
        default_factory=lambda: defaultdict(list), init=False, repr=False, compare=False)

    def on_change(self, name: str, listener: Listener) -> Callable[[], None]:
        self._listeners[name].append(listener)
        return lambda: self._listeners[name].remove(listener)   # unsubscribe handle

    def __setattr__(self, name: str, value: Any) -> None:
        old = getattr(self, name, None)
        super().__setattr__(name, value)
        if name.startswith("_") or old == value or "_listeners" not in self.__dict__:
            return
        for listener in [*self._listeners[name], *self._listeners["*"]]:
            listener(name, old, value)

@dataclass
class CheckoutForm(Observable):
    country: str = "US"
    postal_code: str = ""
    notes: str = ""

form = CheckoutForm()
unsubscribe = form.on_change("country", lambda n, old, new: print(f"recalc shipping {old}->{new}"))
form.on_change("*", lambda n, *_: print(f"dirty: {n}"))
form.country = "DE"   # recalc shipping US->DE, dirty: country
form.notes = "leave at door"   # dirty: notes
form.country = "DE"   # nothing: value unchanged
unsubscribe()
```

## Example — TypeScript
In Angular, a signal per property *is* a property change listener with dependency tracking; `computed` recalculates only when its inputs change.

```typescript
import { Component, computed, effect, signal } from '@angular/core';

@Component({
  selector: 'app-checkout',
  template: `
    <select [value]="country()" (change)="country.set($any($event.target).value)">
      <option>US</option><option>DE</option>
    </select>
    <input [value]="postalCode()" (input)="postalCode.set($any($event.target).value)" />
    <p>Shipping: {{ shippingCost() }}</p>
  `,
})
export class Checkout {
  readonly country = signal('US');
  readonly postalCode = signal('');
  readonly notes = signal('');

  // Only re-runs when `country` changes; editing notes/postalCode does not trigger it.
  readonly shippingCost = computed(() => (this.country() === 'US' ? 5 : 12));

  constructor() {
    let previous = this.country();
    effect(() => {
      const current = this.country();            // tracked dependency
      if (current !== previous) {
        this.postalCode.set('');                  // side effect on a specific property change
        previous = current;
      }
    });
  }
}
```

For Reactive Forms, `form.get('country')!.valueChanges` is the per-control listener (an Observable); pipe through `distinctUntilChanged()` to suppress no-op emissions. Signal-based forms are newer and their API may still be changing: check angular.dev.

## In the wild
- Java Beans `PropertyChangeSupport` / `VetoableChangeSupport`; .NET `INotifyPropertyChanged` (WPF/MAUI binding).
- Angular signals (`signal`, `computed`, `effect`), Reactive Forms `valueChanges` per control; Vue `watch(() => obj.field, ...)`; MobX `observe`/`reaction`.
- Python: `traitlets` (Jupyter widgets: `obj.observe(handler, names=["value"])`), `param`, Pydantic has no change hooks (models are data, `validate_assignment=True` validates but does not notify).
- SQLAlchemy attribute events (`@event.listens_for(User.email, "set")`).
- JS `Proxy` set trap: the underlying mechanism of Vue 3 reactivity.

## Trade-offs
| Pros | Cons |
|---|---|
| Listeners react only to the fields they care about | Event storms when many fields change in one logical update |
| Old/new values enable undo, audit, dirty tracking | Listeners observe intermediate, inconsistent states |
| Decouples UI parts from each other | Implicit control flow: hard to trace who reacts to what |
| Supported natively by most reactive frameworks | Leaks if listeners are never removed |

## Pitfalls
- **Firing when the value did not change.** Causes loops and wasted work. Fix: compare before firing (`old == new` / `Object.is`, or `distinctUntilChanged`).
- **Listener mutates the same object, causing cascades or infinite loops.** Fix: guard re-entrancy, or derive values (`computed`) instead of writing in listeners. In Angular, avoid writing signals inside `effect` when `computed` or `linkedSignal` expresses it.
- **Mid-update inconsistency.** Setting `country` then `postal_code` fires a listener between them that sees a mismatched pair. Fix: batch updates (one `update()` on a state object) or debounce.
- **Memory leaks.** A long-lived model holds references to destroyed components' listeners. Fix: return an unsubscribe handle; in Angular, `takeUntilDestroyed()` / `DestroyRef`; effects created in an injection context are cleaned up automatically.
- **Stringly-typed property names.** Typos silently register for a non-existent field. Fix: `keyof T` in TS, or field references rather than strings.
- **Mutating a nested object in place.** `form.address.city = "X"` does not go through the outer setter; in signals, mutating an object inside a signal does not notify. Fix: replace immutably (`sig.update(a => ({...a, city: 'X'}))`).

## Refactoring path
- **Introduce it:** from setters that call many components directly or from polling. (1) Add a listener registry to the subject. (2) Move each direct call out of the setter into a registered listener. (3) Add the equality guard. (4) Add unsubscribe and wire it to component teardown. In Angular, go straight to signals instead of steps 1–3.
- **Remove it / simplify:** if listeners react to every property anyway, collapse to one "state changed" event over an immutable state object. If only one listener exists, replace with a direct call or callback parameter.

## Related patterns
- [Observer](observer.md) — the general form; Property Change Listener is Observer with per-property topics and old/new payloads.
- [Event Aggregator](event-aggregator.md) — centralizes events from many objects; property listeners are attached to one object.
- [MVVM](../architectural/mvvm.md) — data binding is built on property change notification.
- [Presentation Model](../ui/presentation-model.md) — the object that typically exposes observable properties.
- [Mediator](mediator.md) — coordinates reactions between fields when listener-to-listener chains get tangled.
