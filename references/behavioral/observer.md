# Observer

**Category:** Behavioral · **Also known as:** Listener, Dependents, Event handler · **Related:** Publish-Subscribe, Event Aggregator, Property Change Listener, Mediator, Iterator

## Intent
Define a one-to-many dependency so that when one object (the subject) changes, all registered dependents (observers) are notified automatically, without the subject knowing their concrete types.

## Problem
- A domain object has to tell several other parts of the system that something happened, and the list keeps growing.
- Smell: `OrderService.place()` ends with `email.send(...)`, `analytics.track(...)`, `inventory.reserve(...)`, `cache.invalidate(...)`. Every new reaction edits the service and adds an import, so the core now depends on everything downstream.
- Smell: UI code polls a value in a loop or a timer "to see if it changed".
- Scenario: a cart total must update the header badge, the checkout button state and a free-shipping banner. Each component re-reads the cart on its own schedule and they drift out of sync.

## Solution and structure
- **Subject** keeps a list of observers and offers `subscribe` / `unsubscribe`. After a state change it calls `notify`.
- **Observer** is anything with an `update(event)` method, or just a callable.
- Observers register themselves (or are registered by a composition root). The subject only knows the observer interface.

```
Subject ──notify(event)──▶ Observer A
   │      (in-process,  ─▶ Observer B
   └─ list of observers  ─▶ Observer C
```

Push model: the event carries the data. Pull model: the event only says "changed" and the observer reads the subject. Push is usually simpler and avoids races.

Observer is **in-process and synchronous by default**. Once events cross a process or need durability, you want Publish-Subscribe with a broker.

## When to use
- One change, many reactions, and the set of reactions varies independently of the subject.
- UI state propagation: model to view, store to components (signals, RxJS, Vue refs).
- Domain events inside one service (an order was placed, so recompute loyalty points).
- Lifecycle hooks in libraries ("on connect", "on retry") that callers want to extend.

## When NOT to use
- Exactly one reaction that will always happen. Call it directly; it is easier to read and debug.
- The caller needs a result or must know whether the reaction succeeded (charging a card). Use a direct call, Command, or a Saga.
- Reactions must survive crashes or run in another service. Use an outbox plus a Message Queue or Publish-Subscribe.
- Order between observers matters and is load-bearing. Make it an explicit pipeline (Chain of Responsibility, Pipes and Filters).
- Cascading observers that update other subjects create hidden control flow. Use a Mediator or derive state with `computed`.

## Example — Python
The idiomatic form is a list of callables, not an `Observer` base class.

```python
from collections.abc import Callable
from dataclasses import dataclass, field
import logging

log = logging.getLogger(__name__)

@dataclass(frozen=True)
class OrderPlaced:
    order_id: str
    customer_email: str
    total_cents: int

type Handler = Callable[[OrderPlaced], None]

@dataclass
class OrderEvents:
    _handlers: list[Handler] = field(default_factory=list)

    def subscribe(self, handler: Handler) -> Callable[[], None]:
        self._handlers.append(handler)
        return lambda: self._handlers.remove(handler)   # unsubscribe handle

    def publish(self, event: OrderPlaced) -> None:
        for handler in list(self._handlers):            # copy: handlers may unsubscribe
            try:
                handler(event)
            except Exception:
                log.exception("handler %r failed for %s", handler, event.order_id)

events = OrderEvents()
events.subscribe(lambda e: print(f"email receipt to {e.customer_email}"))
unsubscribe = events.subscribe(lambda e: print(f"track revenue {e.total_cents}"))
events.publish(OrderPlaced("o-1", "ann@example.com", 4_200))
unsubscribe()
```

For async handlers, `await asyncio.gather(*(h(e) for h in handlers), return_exceptions=True)` keeps one failure from cancelling the rest.

## Example — TypeScript
In Angular, signals are the Observer pattern for synchronous state; RxJS is the Observer pattern for event streams.

```typescript
import { Component, DestroyRef, Injectable, computed, effect, inject, signal } from '@angular/core';
import { Subject } from 'rxjs';
import { takeUntilDestroyed } from '@angular/core/rxjs-interop';

export interface CartItem { sku: string; priceCents: number; qty: number }

@Injectable({ providedIn: 'root' })
export class CartStore {
  readonly items = signal<CartItem[]>([]);
  readonly totalCents = computed(() =>
    this.items().reduce((sum, i) => sum + i.priceCents * i.qty, 0));
  readonly checkedOut = new Subject<{ totalCents: number }>();   // one-off events

  add(item: CartItem): void {
    this.items.update(list => [...list, item]);                   // new array, not push()
  }
}

@Component({ selector: 'app-cart-badge', template: `{{ cart.totalCents() / 100 }} EUR` })
export class CartBadge {
  protected readonly cart = inject(CartStore);
  constructor() {
    effect(() => console.log('total changed', this.cart.totalCents()));
    this.cart.checkedOut
      .pipe(takeUntilDestroyed(inject(DestroyRef)))
      .subscribe(e => console.log('checked out', e.totalCents));
  }
}
```

## In the wild
- Angular signals (`signal`, `computed`, `effect`), RxJS `Subject`/`BehaviorSubject`, `EventEmitter` behind `output()`.
- Node `EventEmitter`, DOM `addEventListener`, `AbortSignal` `abort` event.
- SQLAlchemy ORM events (`event.listen(Session, "after_commit", fn)`), Django signals, Blinker (used by Flask).
- `asyncio.Future.add_done_callback`, `logging.Handler` (loggers notify handlers).
- Vue `watch`/`ref`, MobX, Redux `store.subscribe`.

## Trade-offs
| Pros | Cons |
|---|---|
| Subject does not depend on concrete reactions (open for extension) | Control flow is implicit; "who reacts to this?" needs a search |
| Reactions can be added and removed at runtime | Forgotten subscriptions leak memory and keep dead objects alive |
| Natural fit for UI state and domain events | Notification order is usually unspecified |
| Easy to test the subject in isolation | Cascades (observer updates another subject) can loop or glitch |
| | Synchronous handlers block the publisher; exceptions need a policy |

## Pitfalls
- **Leaked subscriptions.** Components subscribe and never unsubscribe. Fix: return an unsubscribe handle; in Angular use `takeUntilDestroyed`, the `async` pipe, or `toSignal`; in Python hold `weakref.WeakMethod` for bound-method observers when lifetimes differ.
- **One failing observer stops the rest.** Fix: catch and log per handler, or use `gather(..., return_exceptions=True)`.
- **Mutating the list while iterating.** A handler unsubscribes during notify. Fix: iterate over a copy.
- **In-place mutation under signals/OnPush.** `items().push(x)` does not notify. Fix: always set a new reference with `update(list => [...list, x])`.
- **Infinite loops.** Observer A writes to B which notifies A. Fix: derive with `computed`, or guard with equality checks. Angular forbids writing signals in `computed`.
- **Using it for must-happen side effects** (payment, emails with guarantees). Fix: transactional outbox plus a queue.

## Refactoring path
- **Introduce it:** start from a method that calls three or more unrelated collaborators after its main work. (1) Define an immutable event dataclass. (2) Add a tiny subject with `subscribe`/`publish`. (3) Move one direct call into a handler, registered in the composition root; keep tests green. (4) Move the rest one at a time. (5) Delete the now-unused imports from the core.
- **Remove it / simplify:** if a subject has one subscriber that never changes, inline the call. If state is being copied between subjects by observers, replace the chain with one source of truth plus derived values (`computed`, properties).

## Related patterns
- [Publish-Subscribe](../architectural/publish-subscribe.md) — Observer across process boundaries, decoupled by topic and broker; adds durability and async delivery.
- [Event Aggregator](event-aggregator.md) — one central channel instead of subscribing to each subject separately.
- [Property Change Listener](property-change-listener.md) — Observer specialised to individual property changes.
- [Mediator](mediator.md) — replaces a web of observers with explicit coordination logic in one place.
- [Iterator](iterator.md) — the pull-based dual of Observer; RxJS combines both.
- [Event-Driven](../architectural/event-driven.md) — the architectural style built on these notifications.
