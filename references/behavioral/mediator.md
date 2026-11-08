# Mediator

**Category:** Behavioral · **Also known as:** Controller, Coordinator, Hub · **Related:** Observer, Event Aggregator, Facade, Command, Publish-Subscribe, Broker, Saga

## Intent
Encapsulate how a set of objects interact in one mediator object, so the participants talk to the mediator instead of referencing each other directly.

## Problem
- N components each hold references to several others, giving up to N×(N−1) dependencies. Changing one forces edits in many.
- Interaction rules ("when country changes, reset region, reload shipping options, re-validate VAT") are scattered across the participants, and nobody can see the full flow.
- Components cannot be reused elsewhere because they know their neighbours.
- Scenario: a checkout page has address form, shipping picker, payment picker, promo code box and order summary. Address affects shipping; shipping affects summary; promo affects summary and may disable some payment methods. Each component injects three others and calls them directly; a change in one breaks another.

## Solution and structure
- **Colleagues** know only the mediator. They report what happened (`addressChanged(addr)`) and expose simple inputs.
- **Mediator** knows all colleagues and contains the coordination rules: given this event, update those colleagues.

```
Before:  A ⇄ B ⇄ C ⇄ D  (mesh)        After:  A   B   C   D
         ⇅ ╳ ⇅                                  ╲  │  │  ╱
                                                 Mediator
```

Difference from Observer / Event Aggregator: those only route events and do not know what reactions happen. A mediator **owns the interaction logic** and talks to specific colleagues. In Angular, a parent "smart" component or a component-scoped store service with signals is usually the mediator, and child components communicate via inputs/outputs.

## When to use
- A screen or workflow where several parts must stay consistent and the rules span them (checkout, wizard, dashboard filters).
- Many-to-many communication between objects that you want to reuse independently.
- Orchestrating a multi-step process across services from one place (an orchestration-style Saga is a distributed mediator).
- Chat rooms, air-traffic-style coordination, game lobbies.

## When NOT to use
- Two or three objects with simple, stable interactions. Direct calls or inputs/outputs are clearer.
- Reactions are independent of each other and order does not matter. Observer or an event bus is lighter.
- When the mediator would become a god object holding all business logic. Split it by sub-flow, or push rules down into the domain.
- In-process "mediator libraries" (`MediatR`-style `send(request)` buses) used only to call one handler per request add indirection without coordination. Call the service.

## Example — Python
A checkout coordinator mediating between independent domain services.

```python
from dataclasses import dataclass
from decimal import Decimal
from typing import Protocol

@dataclass(frozen=True)
class Address:   country: str; postcode: str
@dataclass(frozen=True)
class Quote:     subtotal: Decimal; shipping: Decimal; discount: Decimal; vat: Decimal

class Shipping(Protocol):
    def options(self, addr: Address) -> dict[str, Decimal]: ...
class Promotions(Protocol):
    def discount(self, code: str | None, subtotal: Decimal) -> Decimal: ...
class Tax(Protocol):
    def vat(self, addr: Address, net: Decimal) -> Decimal: ...

class CheckoutMediator:
    """Owns the rules that connect address, shipping, promo and tax."""
    def __init__(self, shipping: Shipping, promos: Promotions, tax: Tax, subtotal: Decimal) -> None:
        self._shipping, self._promos, self._tax = shipping, promos, tax
        self.subtotal = subtotal
        self.address: Address | None = None
        self.method: str | None = None
        self.promo: str | None = None

    def address_changed(self, addr: Address) -> dict[str, Decimal]:
        self.address = addr
        options = self._shipping.options(addr)
        if self.method not in options:          # rule: invalid method resets
            self.method = None
        return options

    def method_chosen(self, method: str) -> None:
        self.method = method

    def promo_applied(self, code: str) -> None:
        self.promo = code

    def quote(self) -> Quote:
        if self.address is None or self.method is None:
            raise ValueError("address and shipping method required")
        ship = self._shipping.options(self.address)[self.method]
        disc = self._promos.discount(self.promo, self.subtotal)
        net = self.subtotal - disc + ship
        return Quote(self.subtotal, ship, disc, self._tax.vat(self.address, net))
```

None of the services knows about the others; the rules are readable in one class.

## Example — TypeScript
Angular: a component-scoped store mediates; children only use inputs/outputs or the store.

```typescript
import { Injectable, computed, inject, signal, Component } from '@angular/core';

interface Address { country: string; postcode: string }
const RATES: Record<string, Record<string, number>> = { UA: { nova: 3 }, PL: { dpd: 6, inpost: 4 } };

@Injectable()                                         // provided per checkout page, not root
export class CheckoutMediator {
  readonly subtotal = signal(80);
  readonly address = signal<Address | null>(null);
  readonly method = signal<string | null>(null);
  readonly promo = signal<string | null>(null);

  readonly options = computed(() => RATES[this.address()?.country ?? ''] ?? {});
  readonly discount = computed(() => (this.promo() === 'SPRING10' ? this.subtotal() * 0.1 : 0));
  readonly total = computed(() => {
    const ship = this.options()[this.method() ?? ''] ?? 0;
    return this.subtotal() - this.discount() + ship;
  });

  addressChanged(a: Address): void {
    this.address.set(a);
    const m = this.method();
    if (m !== null && !(m in this.options())) this.method.set(null);    // rule lives here
  }
  methodChosen(m: string): void { this.method.set(m); }
  promoApplied(code: string): void { this.promo.set(code.trim().toUpperCase()); }
}

@Component({
  selector: 'app-checkout',
  providers: [CheckoutMediator],
  template: `<!-- <app-address (changed)="m.addressChanged($event)" />
                  <app-shipping [options]="m.options()" (chosen)="m.methodChosen($event)" /> -->
             Total: {{ m.total() }}`,
})
export class Checkout { protected readonly m = inject(CheckoutMediator); }
```

Child components never inject each other; they receive data via inputs and report via outputs, and the mediator decides what changes. Test the mediator alone, without rendering.

## In the wild
- Angular smart/container components and component-scoped services (`providers: [...]` on a component), NgRx ComponentStore / SignalStore.
- Form libraries: Angular `FormGroup` coordinates validation and value changes across child controls.
- Orchestrators: AWS Step Functions, Temporal workflows, Camunda (distributed mediators).
- `asyncio.TaskGroup` coordinating child tasks; Kubernetes controllers reconciling several resources.
- .NET MediatR and NestJS `@nestjs/cqrs` call themselves mediators but are mostly request dispatchers.

## Trade-offs
| Pros | Cons |
|---|---|
| Replaces a mesh of dependencies with a star | The mediator can become a god class |
| Interaction rules are visible and testable in one place | Single point of change for every new rule (merge conflicts) |
| Colleagues are reusable and simpler | Adds a layer; simple flows become indirect |
| Easy to change the flow without touching participants | Hides direct cause and effect from readers of a colleague |

## Pitfalls
- **God mediator.** All business rules accumulate in one class. Fix: one mediator per screen or flow; domain rules stay in domain objects; the mediator only coordinates.
- **Colleagues still calling each other** "just this once". Fix: enforce through DI scope (colleagues cannot inject each other) and code review.
- **Mediator as root singleton** for a per-page flow, leaking state between visits. Fix: provide it at the component level so it is created and destroyed with the page.
- **Two-way loops**: the mediator updates A, A emits a change, the mediator updates A again. Fix: derived state (`computed`) instead of imperative syncing, and equality checks.
- **Confusing it with an event bus.** If the hub has no logic, call it an Event Aggregator and keep it dumb.

## Refactoring path
- **Introduce it:** from components that inject or reference each other. (1) Create a mediator and inject it into all participants. (2) Move one interaction rule into it; the participant now calls `mediator.x_changed()`. (3) Repeat until participants no longer reference each other. (4) Convert imperative syncing into derived state where possible.
- **Remove it / simplify:** if the mediator only forwards calls one-to-one, delete it and wire the two parties directly (or via inputs/outputs). If it has grown huge, split it by sub-flow.

## Related patterns
- [Observer](observer.md) — colleagues often notify the mediator via events; the mediator adds the coordination logic.
- [Event Aggregator](event-aggregator.md) — a logic-free hub; Mediator is a hub with rules.
- [Facade](../structural/facade.md) — simplifies access *to* a subsystem; Mediator coordinates *between* peers.
- [Saga](../cloud/saga.md) — orchestration sagas are distributed mediators with compensation.
- [Broker](../architectural/broker.md) — mediates between remote clients and servers at the architecture level.
- [Command](command.md) — colleagues may send commands to the mediator instead of calling methods.
