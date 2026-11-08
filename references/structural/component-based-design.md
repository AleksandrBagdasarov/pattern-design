# Component-Based Design

**Category:** Structural (component patterns) · **Also known as:** Composition over inheritance, Entity-component (object form), Mixins-as-parts · **Related:** Entity-Component-System, Strategy, Extension Object, Composite, Decorator, Component-Oriented Programming

## Intent
Build objects from small, pluggable components, each owning one aspect of behavior and state, instead of from deep inheritance hierarchies; behavior is changed by adding, removing or swapping components.

## Problem
- Inheritance trees grow sideways and down: `User` → `PremiumUser` → `PremiumBusinessUser` → `PremiumBusinessUserWithSso`; a "free business user with SSO" does not fit anywhere.
- Base classes become dumping grounds (the "God base class") because every subclass needs a bit of shared code.
- Feature combinations are decided at runtime (tenant config, feature flags, game items), but classes are decided at compile time.
- Changing one aspect (pricing) forces changes in an unrelated hierarchy (notifications).

Scenario: a SaaS account has a billing plan, an authentication method, a storage quota policy and a notification preference. Each varies independently and can change while the account lives. Composition: `Account(billing=..., auth=..., quota=..., notifier=...)`.

## Solution and structure
- **Container / entity**: a thin object that owns components and delegates to them; little or no logic of its own.
- **Components**: small objects implementing one aspect behind an interface (`BillingPlan`, `AuthMethod`).
- **Wiring**: a factory, builder, configuration or DI container assembles the right components.
- Optional: components can be looked up by type (`entity.get(Health)`), as in Unity/Godot game objects.

```
Account ──billing──> FlatRatePlan | UsagePlan
        ──auth─────> PasswordAuth | SsoAuth
        ──quota────> FixedQuota   | TieredQuota
```
This is "favor object composition over class inheritance" (GoF) applied as an organising principle. Each slot is essentially a Strategy.

## When to use
- Several independent axes of variation on one concept.
- Combinations decided by configuration or at runtime, or changing over the object's life.
- Game objects, UI widgets, workflow steps, accounts/tenants with feature toggles.
- Teams own different aspects; components let them work without touching one shared class.

## When NOT to use
- One axis of variation: a single Strategy parameter or simple subclassing is clearer.
- Small, stable "is-a" hierarchies (e.g. `Shape` → `Circle`): inheritance is fine.
- Tens of thousands of entities updated every frame: use ECS for data locality instead of objects-with-components.
- Components that constantly reach into each other: they are one component in disguise; merge them.

## Example — Python
Slots are Protocols; the account delegates and can swap a component at runtime.

```python
from dataclasses import dataclass
from decimal import Decimal
from typing import Protocol


class BillingPlan(Protocol):
    def monthly_charge(self, usage_gb: Decimal) -> Decimal: ...

class QuotaPolicy(Protocol):
    def allows(self, used_gb: Decimal, extra_gb: Decimal) -> bool: ...

class Notifier(Protocol):
    def notify(self, account_id: str, message: str) -> None: ...


@dataclass(frozen=True)
class FlatRate:
    price: Decimal
    def monthly_charge(self, usage_gb: Decimal) -> Decimal: return self.price

@dataclass(frozen=True)
class PayPerGb:
    per_gb: Decimal
    def monthly_charge(self, usage_gb: Decimal) -> Decimal: return usage_gb * self.per_gb

@dataclass(frozen=True)
class HardQuota:
    limit_gb: Decimal
    def allows(self, used_gb: Decimal, extra_gb: Decimal) -> bool:
        return used_gb + extra_gb <= self.limit_gb

class EmailNotifier:
    def notify(self, account_id: str, message: str) -> None:
        print(f"email to {account_id}: {message}")


@dataclass
class Account:                                   # container: delegates, little logic
    id: str
    billing: BillingPlan
    quota: QuotaPolicy
    notifier: Notifier
    used_gb: Decimal = Decimal("0")

    def upload(self, size_gb: Decimal) -> bool:
        if not self.quota.allows(self.used_gb, size_gb):
            self.notifier.notify(self.id, "Storage quota exceeded")
            return False
        self.used_gb += size_gb
        return True


acc = Account("acme", FlatRate(Decimal("49")), HardQuota(Decimal("100")), EmailNotifier())
acc.billing = PayPerGb(Decimal("0.4"))           # plan change, no new class needed
```

## Example — TypeScript
Angular's own composition tools: injectable services as components plus host directives to compose UI behavior.

```ts
import { Component, Directive, signal, input, inject, ElementRef } from '@angular/core';

@Directive({ selector: '[appTooltip]', host: { '[attr.title]': 'text()' } })
export class Tooltip { readonly text = input('', { alias: 'appTooltip' }); }

@Directive({ selector: '[appDisableable]', host: { '[attr.aria-disabled]': 'disabled()' } })
export class Disableable { readonly disabled = signal(false); }

@Directive({
  selector: '[appRipple]',
  host: { '(click)': 'ripple()' },
})
export class Ripple {
  private readonly el = inject<ElementRef<HTMLElement>>(ElementRef);
  ripple() { this.el.nativeElement.animate([{ opacity: 0.6 }, { opacity: 1 }], 200); }
}

// The button is composed of behaviors instead of inheriting BaseButton → IconButton → ...
@Component({
  selector: 'app-action-button',
  hostDirectives: [Ripple, Disableable, { directive: Tooltip, inputs: ['appTooltip: hint'] }],
  template: `<ng-content />`,
})
export class ActionButton {
  private readonly disableable = inject(Disableable);
  lock() { this.disableable.disabled.set(true); }
}
// <app-action-button hint="Save the order">Save</app-action-button>
```
`hostDirectives` (Angular 15+) attaches directives to a component's host; check angular.dev for current input/output aliasing syntax.

## In the wild
- Unity `GameObject` + `MonoBehaviour` components, Godot nodes, Unreal actor components.
- Angular `hostDirectives`, DI-injected services; React hooks (`useForm`, `useAuth`) and Vue composables are function-level components.
- Python: Django class-based views via mixins (a weaker, inheritance-based form), `logging` (logger + handlers + formatters + filters), Starlette app assembled from middleware and routers.
- SQLAlchemy composite columns, Pydantic models composed of nested models.

## Trade-offs
| Pros | Cons |
|---|---|
| Independent axes vary without class explosion | Behavior is spread over several objects; harder to read in one place |
| Components swapped at runtime or per config | Wiring/assembly code becomes important and can grow complex |
| Components reusable and testable alone | Inter-component communication needs a design (events, container) |
| Teams own separate components | Type-based lookup (`get(Component)`) loses static guarantees |

## Pitfalls
- **Components talking to each other directly** via the container (`self.owner.billing.quota...`). Fix: the container coordinates, or components communicate via events.
- **Mixins mistaken for composition**: multiple inheritance of mixins still couples at class definition and fights over `__init__`/MRO. Fix: hold components as fields.
- **Anemic container that leaks components**: callers do `acc.quota.allows(...)` everywhere. Fix: expose intention-revealing methods on the container.
- **Required components missing at runtime.** Fix: required constructor arguments, not optional lookups; validate at assembly.

## Refactoring path
- **Introduce it:** find a hierarchy whose subclasses differ along two or more aspects; extract one aspect into an interface + implementations; replace subclasses by passing components; repeat per aspect; collapse the hierarchy.
- **Remove it / simplify:** if a slot only ever has one implementation, inline it; if all components are always used together, merge them back into one class.

## Related patterns
- [Entity-Component-System](../structural/entity-component-system.md) — data-oriented variant: components are plain data, logic lives in systems.
- [Strategy](../behavioral/strategy.md) — each component slot is a strategy.
- [Extension Object](../structural/extension-object.md) — capabilities attached and queried at runtime.
- [Composite](../structural/composite.md) — containers of children forming trees; can be combined with components.
- [Decorator](../structural/decorator.md) — adds behavior by wrapping rather than by adding parts.
- [Component-Oriented Programming](../structural/component-oriented-programming.md) — components as deployable units with contracts, at a larger scale.
