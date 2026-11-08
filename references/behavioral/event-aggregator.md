# Event Aggregator

**Category:** Behavioral · **Also known as:** Event Bus (in-process), Message Hub · **Related:** Observer, Mediator, Publish-Subscribe, Property Change Listener, Message Broker

## Intent
Provide one in-process channel through which many publishers raise events and many subscribers receive them, so publishers and subscribers only know the aggregator and the event types, not each other.

## Problem
- With plain Observer, each subscriber must find and attach to each subject. With N sources and M listeners you get N×M wiring, and listeners must know where events come from.
- Smells: a component injects five unrelated services only to subscribe to their events; a "notify everything" method that grows each time a new feature needs to react; dynamic sources (plugins, dynamically created widgets) that listeners cannot know in advance.

Scenario: in an admin SPA, "user logged out" is raised by the auth service, by an HTTP interceptor on 401, and by an idle-timeout timer. The cart, the websocket connection, the analytics tracker and the toast service must all react. Nobody should import everybody.

## Solution and structure
A single aggregator object maps event *types* to subscriber lists. Publishers call `publish(event)`; subscribers call `subscribe(EventType, handler)`.

```
AuthService ─┐                        ┌─> CartStore
Http401 ─────┼─ publish(LoggedOut) ─> │ EventAggregator │ ─┼─> SocketClient
IdleTimer ───┘                        └─> Analytics
```

Participants:
- **Event**: a small immutable message type (`LoggedOut`, `OrderPlaced`), typed per event rather than stringly.
- **EventAggregator**: registry `type -> handlers`, `publish`, `subscribe` returning an unsubscribe handle.
- **Publisher / Subscriber**: depend only on the aggregator and event types; injected via DI.

It differs from Pub/Sub architecture in scope: an Event Aggregator lives in one process, typically delivers synchronously or on the same event loop, and offers no durability.

## When to use
- Many-to-many notifications inside one application (SPA, desktop app, modular monolith).
- Cross-module signals in a modular monolith where modules must not import each other (`OrderPlaced` → loyalty, email, analytics modules).
- Plugin systems where publishers and subscribers are discovered at runtime.

## When NOT to use
- One publisher, few known subscribers: use Observer directly (a signal or Subject on the owning service).
- Request/response or anything where the publisher needs a result: call a service.
- Ordered workflows ("do A, then B, then C"): orchestrate explicitly (Service Layer, Saga); an event bus hides the sequence.
- Across processes, or when events must survive crashes: use a Message Broker / transactional outbox.
- Shared state that UI renders: use a store/signal state; a bus spreads state across handlers.
- As a way to avoid thinking about dependencies: a global bus with 200 event types is a hidden coupling graph.

## Example — Python
A typed, async in-process bus. Handlers run concurrently and one failing handler does not stop the others.

```python
import asyncio
import logging
from collections import defaultdict
from collections.abc import Awaitable, Callable
from dataclasses import dataclass

log = logging.getLogger(__name__)

@dataclass(frozen=True, slots=True)
class OrderPlaced:
    order_id: str
    customer_id: str
    total_cents: int

type Handler[E] = Callable[[E], Awaitable[None]]

class EventAggregator:
    def __init__(self) -> None:
        self._handlers: defaultdict[type, list[Handler]] = defaultdict(list)

    def subscribe[E](self, event_type: type[E], handler: Handler[E]) -> Callable[[], None]:
        self._handlers[event_type].append(handler)
        return lambda: self._handlers[event_type].remove(handler)

    async def publish(self, event: object) -> None:
        handlers = list(self._handlers[type(event)])          # copy: handlers may unsubscribe
        results = await asyncio.gather(*(h(event) for h in handlers), return_exceptions=True)
        for handler, result in zip(handlers, results):
            if isinstance(result, Exception):
                log.error("handler %s failed for %r", handler.__qualname__, event, exc_info=result)

bus = EventAggregator()

async def award_points(e: OrderPlaced) -> None: ...
async def send_receipt(e: OrderPlaced) -> None: ...

bus.subscribe(OrderPlaced, award_points)
bus.subscribe(OrderPlaced, send_receipt)
# in the order service, after the DB commit:
# await bus.publish(OrderPlaced("o-1", "c-9", 4999))
```

Publish *after* the transaction commits; otherwise subscribers act on data that may roll back.

## Example — TypeScript
An Angular root service built on an RxJS `Subject`, with type-safe filtering by event class.

```typescript
import { Injectable, inject, DestroyRef } from '@angular/core';
import { takeUntilDestroyed } from '@angular/core/rxjs-interop';
import { Observable, Subject, filter } from 'rxjs';

export class LoggedOut { constructor(readonly reason: 'manual' | 'expired' | 'idle') {} }
export class CartChanged { constructor(readonly itemCount: number) {} }

type Ctor<T> = new (...args: never[]) => T;

@Injectable({ providedIn: 'root' })
export class EventAggregator {
  private readonly events$ = new Subject<object>();

  publish(event: object): void {
    this.events$.next(event);
  }

  on<T extends object>(type: Ctor<T>): Observable<T> {
    return this.events$.pipe(filter((e): e is T => e instanceof type));
  }
}

// Subscriber in a component or service; cleaned up automatically on destroy.
@Injectable({ providedIn: 'root' })
export class SocketClient {
  private readonly bus = inject(EventAggregator);
  constructor() {
    this.bus.on(LoggedOut).pipe(takeUntilDestroyed(inject(DestroyRef)))
      .subscribe(e => this.disconnect(e.reason));
  }
  private disconnect(reason: string): void { /* close socket */ }
}
```

## In the wild
- Angular: a root service with a `Subject` is the idiom; NgRx actions stream is a structured event aggregator plus a store.
- Node `EventEmitter` (one emitter shared app-wide), NestJS `@nestjs/event-emitter` (`@OnEvent('order.placed')`), `mitt` in Vue apps.
- Python: `blinker` signals (Flask uses them), Django signals (`post_save`), `pyee`.
- Prism (WPF) `IEventAggregator`, Guava `EventBus`, MediatR notifications in .NET.
- Out of process, the same idea becomes Pub/Sub (Redis pub/sub, SNS, Kafka topics).

## Trade-offs
| Pros | Cons |
|---|---|
| Publishers and subscribers fully decoupled | Control flow is invisible; "who handles this?" needs a search |
| Easy to add a new reaction without touching the publisher | Hard to reason about ordering and error handling |
| Works with dynamically created sources/listeners | Global bus becomes a dumping ground for every interaction |
| Simple to test publishers (assert published events) | No durability or delivery guarantees in-process |

## Pitfalls
- **Stringly-typed event names** (`'user:logout'` vs `'user.logout'`). Fix: event classes/discriminated unions; subscribe by type.
- **Subscriber leaks.** Components subscribe and never unsubscribe. Fix: unsubscribe handle; `takeUntilDestroyed`.
- **Publishing before commit.** Handlers send emails for orders that rolled back. Fix: publish after commit, or use the outbox pattern for anything external.
- **One handler's exception kills the rest** (synchronous loops, or an RxJS subscriber that throws). Fix: isolate handler errors (`return_exceptions=True`, `catchError` per subscription).
- **Event ping-pong.** Handler A publishes an event that triggers B, which publishes one that triggers A. Fix: keep events as facts about the past, not commands; review cycles.
- **Using events as commands** (`SendEmailPlease`). Fix: if exactly one handler must run and the caller cares, call it directly.
- **Late subscribers miss events** with a plain `Subject`. Fix: decide explicitly; use `ReplaySubject(1)` or state (signal) when "current value" matters.

## Refactoring path
- **Introduce it:** when a service has grown many outward calls to unrelated modules on one occurrence. (1) Define an event type for the occurrence. (2) Create the aggregator and register it in DI. (3) Move each outward call into a subscriber in the module that owns it. (4) Replace the calls with one `publish`. (5) Add a test that the publisher emits the event and separate tests per subscriber.
- **Remove it / simplify:** if an event type has exactly one publisher and one subscriber, replace it with a direct call or an Observable exposed by the publisher. If handlers must run in order, replace with an explicit orchestrating function.

## Related patterns
- [Observer](observer.md) — subscribers attach to a specific subject; the aggregator adds an indirection to decouple from sources.
- [Mediator](mediator.md) — also a hub, but contains coordination logic; the aggregator just routes.
- [Publish-Subscribe](../architectural/publish-subscribe.md) — the distributed, often durable, version.
- [Message Broker](../enterprise/message-broker.md) — out-of-process infrastructure when events cross service boundaries.
- [Property Change Listener](property-change-listener.md) — fine-grained per-object notification; the aggregator is app-wide.
