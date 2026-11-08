# Event-Driven Architecture

**Category:** Architectural · **Also known as:** EDA, reactive architecture · **Related:** Publish-Subscribe, Event Sourcing, Message Broker, Message Queue, Saga, Observer, CQRS

## Intent
Let components communicate by emitting **events** (facts that something happened) and reacting to events from others, instead of calling each other directly. Producers do not know who consumes, and consumers process asynchronously, so components are decoupled in time and in knowledge.

## Problem
- `place_order()` synchronously calls inventory, email, analytics, loyalty and fraud services; one slow or down service fails or slows checkout.
- Every new feature ("also notify the warehouse") edits the order service.
- Spiky traffic needs buffering.
- Scenario: an e-commerce backend where checkout latency is the sum of six downstream calls and a marketing outage blocks orders.

## Solution and structure
```
 Order service ──OrderPlaced──► [ event channel / broker ] ──► Inventory (reserve stock)
      │                                                    ├─► Email (send receipt)
      └── writes order + outbox row in one transaction     └─► Analytics (record sale)
```
- **Event:** immutable, past-tense fact with an id, type, timestamp, version and payload (`OrderPlaced{order_id, total}`).
- **Producer/emitter**, **channel** (in-process bus, Kafka topic, SNS, EventBridge), **consumer/handler**.
- Topologies: **broker topology** (choreography: consumers react independently) vs **mediator topology** (an orchestrator emits commands and listens for results).
- Event styles: **event notification** (thin: "order 42 placed", fetch details), **event-carried state transfer** (fat payload), **event sourcing** (events are the source of truth).

## When to use
- Several independent reactions to one business fact.
- Teams/services must evolve and deploy independently.
- Workloads that need buffering, retries and async processing (emails, webhooks, ETL triggers).
- UI: user interactions and server pushes (WebSocket) drive state changes.

## When NOT to use
- The caller needs the result immediately to continue (price lookup, auth check): use a synchronous call.
- Strong consistency across steps is required and you cannot design compensations.
- Small monolith with one consumer per event: a direct function call is easier to read and debug.
- Smell: "event" names that are commands in disguise (`SendEmail`) consumed by exactly one service — that is a queue/RPC, call it so.

## Example — Python
```python
# In-process event bus (asyncio). For cross-service, replace with a broker + outbox.
import asyncio
from collections import defaultdict
from collections.abc import Awaitable, Callable
from dataclasses import dataclass, field
from datetime import datetime, UTC
from decimal import Decimal
from uuid import UUID, uuid4

@dataclass(frozen=True)
class OrderPlaced:
    order_id: str
    total: Decimal
    event_id: UUID = field(default_factory=uuid4)
    occurred_at: datetime = field(default_factory=lambda: datetime.now(UTC))

Handler = Callable[[object], Awaitable[None]]

class EventBus:
    def __init__(self) -> None:
        self._handlers: dict[type, list[Handler]] = defaultdict(list)
    def subscribe(self, event_type: type, handler: Handler) -> None:
        self._handlers[event_type].append(handler)
    async def publish(self, event: object) -> None:
        results = await asyncio.gather(
            *(h(event) for h in self._handlers[type(event)]), return_exceptions=True
        )
        for r in results:                      # one failing handler must not hide others
            if isinstance(r, Exception):
                print(f"handler failed for {event!r}: {r!r}")   # log + retry/DLQ in real code

bus = EventBus()

async def reserve_stock(e: object) -> None: ...
async def send_receipt(e: object) -> None: ...

bus.subscribe(OrderPlaced, reserve_stock)
bus.subscribe(OrderPlaced, send_receipt)
# in the use case, after commit:  await bus.publish(OrderPlaced("o-42", Decimal("99.90")))
```

## Example — TypeScript
```ts
// Angular: a typed in-app event stream; components react without referencing each other.
import { Injectable, inject, DestroyRef } from '@angular/core';
import { Subject, filter, Observable } from 'rxjs';
import { takeUntilDestroyed } from '@angular/core/rxjs-interop';

export type AppEvent =
  | { type: 'cart/itemAdded'; sku: string; qty: number }
  | { type: 'auth/loggedOut' };

@Injectable({ providedIn: 'root' })
export class EventBus {
  private readonly events$ = new Subject<AppEvent>();
  emit(e: AppEvent) { this.events$.next(e); }
  on<T extends AppEvent['type']>(type: T): Observable<Extract<AppEvent, { type: T }>> {
    return this.events$.pipe(filter((e): e is Extract<AppEvent, { type: T }> => e.type === type));
  }
}

@Injectable({ providedIn: 'root' })
export class CartAnalytics {
  constructor() {
    inject(EventBus).on('cart/itemAdded')
      .pipe(takeUntilDestroyed(inject(DestroyRef)))
      .subscribe(e => console.log('track add_to_cart', e.sku, e.qty));
  }
}
```
For a Node backend, the same shape with Kafka (`kafkajs`) or AWS EventBridge replaces the `Subject`.

## In the wild
- Kafka, RabbitMQ, AWS SNS/SQS/EventBridge, Google Pub/Sub, NATS; Debezium (CDC) turns DB changes into events.
- Django signals, SQLAlchemy ORM events, Node `EventEmitter`, DOM events, RxJS, NgRx actions.
- Transactional outbox libraries and CDC for reliable publishing.

## Trade-offs
| Pros | Cons |
|---|---|
| Producers decoupled from consumers; add consumers without changes | Eventual consistency; harder to reason about end state |
| Resilience: buffering absorbs spikes and outages | Flow is implicit; debugging needs tracing (correlation ids) |
| Independent scaling and deployment | Duplicate/out-of-order delivery must be handled |
| Natural audit trail of business facts | Event schema evolution is a long-term contract |

## Pitfalls
- **Dual write:** committing to the DB and publishing to the broker separately loses events on crash. Use the transactional outbox or CDC.
- **Non-idempotent consumers:** at-least-once delivery means duplicates. Dedupe by `event_id` (Idempotent Receiver).
- **Ordering assumptions:** order is per partition/key at best. Partition by aggregate id; tolerate gaps.
- **Event chains nobody can follow:** A → B → C → A loops. Document flows, add correlation/causation ids and distributed tracing.
- **Fat events leaking internals:** publish a versioned public contract, not your ORM row.
- **Subscriptions never cleaned up (frontend):** use `takeUntilDestroyed`/`DestroyRef`.

## Refactoring path
- **Introduce it:** pick one side effect in a synchronous flow (sending the receipt email); publish a domain event after commit and move the side effect into a handler; add an outbox when the handler moves out of process; repeat for other side effects.
- **Remove it / simplify:** if an event has one consumer that the producer actually waits on, replace it with a direct call or a command on a queue; collapse in-process buses with one subscriber into function calls.

## Related patterns
- [Publish-Subscribe](../architectural/publish-subscribe.md) — the delivery topology most EDA uses.
- [Event Sourcing](../architectural/event-sourcing.md) — events become the system of record, not just notifications.
- [Message Broker](../enterprise/message-broker.md) — the infrastructure carrying events.
- [Saga](../cloud/saga.md) — coordinates multi-step workflows over events.
- [Idempotent Receiver](../emergent/idempotent-receiver.md) — mandatory for at-least-once consumers.
- [Observer](../behavioral/observer.md) — the in-process, synchronous ancestor.
