# Publish-Subscribe

**Category:** Architectural · **Also known as:** Pub/Sub, topic-based messaging · **Related:** Observer, Event-Driven, Message Broker, Message Queue, Event Aggregator, Mediator

## Intent
Let **publishers** send messages to named **topics** (or channels) without knowing who receives them; **subscribers** register interest in topics and each receives its own copy. An intermediary handles delivery, so publishers and subscribers are decoupled in identity, number and often time.

## Problem
- A service calls N other services to tell them the same thing, and the list keeps growing.
- Consumers come and go (new analytics job, a temporary audit consumer) and the publisher should not be redeployed for each.
- Different consumers need the same message at their own pace.
- Scenario: a user-profile service. When a user changes email, billing, CRM sync, search indexing and the notifications service all need to know. Today `update_email()` calls four HTTP endpoints.

## Solution and structure
```
 Publisher ──► topic "user.email_changed" ──┬──► subscription: billing   (own queue/offset)
                                            ├──► subscription: crm-sync
                                            └──► subscription: search-indexer
```
- **Topic:** named channel; messages are fanned out to every subscription.
- **Subscription:** a durable (survives subscriber downtime) or transient (live only) attachment to a topic; often backed by a per-subscriber queue (SNS → SQS) or a consumer group offset (Kafka).
- **Filtering:** by topic hierarchy (`orders.*`, MQTT `sensors/+/temp`) or content attributes.
- **Delivery semantics:** at-most-once or at-least-once in practice; "exactly-once" only within specific systems (e.g. Kafka transactions) and still requires idempotent side effects.
- Contrast with **point-to-point queue:** one message → one consumer from a competing pool.

## When to use
- One-to-many notification of facts across modules or services.
- Dynamic or unknown sets of consumers.
- Frontend: cross-component notifications (WebSocket topics, in-app event streams).

## When NOT to use
- One producer, one consumer, work to be done once: use a work queue (point-to-point).
- Request/response needing an answer: use RPC/HTTP (or request-reply over messaging, explicitly).
- In-process, synchronous notification within one object graph: Observer is simpler.
- Smell: topics with a single subscriber that the publisher depends on — hidden RPC.

## Example — Python
```python
# Redis Pub/Sub is transient (no persistence); Redis Streams or Kafka give durable subscriptions.
import asyncio
import json
from redis.asyncio import Redis

TOPIC = "user.email_changed"

async def publish_email_changed(redis: Redis, user_id: str, new_email: str) -> None:
    payload = {"event_id": f"{user_id}:{new_email}", "user_id": user_id, "email": new_email, "v": 1}
    await redis.publish(TOPIC, json.dumps(payload))

async def run_subscriber(redis: Redis, name: str) -> None:
    pubsub = redis.pubsub()
    await pubsub.subscribe(TOPIC)
    try:
        async for msg in pubsub.listen():
            if msg["type"] != "message":
                continue
            event = json.loads(msg["data"])
            print(f"[{name}] sync email for {event['user_id']}")   # idempotent handler here
    finally:
        await pubsub.unsubscribe(TOPIC)
        await pubsub.aclose()      # method name differs across redis-py versions (close/aclose)

async def main() -> None:
    redis = Redis()
    subs = [asyncio.create_task(run_subscriber(redis, n)) for n in ("billing", "crm")]
    await asyncio.sleep(0.1)                     # transient pub/sub: subscribers must be listening first
    await publish_email_changed(redis, "u-1", "new@example.com")
    await asyncio.sleep(0.1)
    for t in subs:
        t.cancel()
```
The `sleep` illustrates the classic trap: with non-durable pub/sub, messages published before a subscriber connects are lost.

## Example — TypeScript
```ts
// Angular: topic-based pub/sub over one WebSocket (rxjs/webSocket multiplex).
import { Injectable } from '@angular/core';
import { webSocket, WebSocketSubject } from 'rxjs/webSocket';
import { Observable, share } from 'rxjs';

interface Envelope<T = unknown> { topic: string; data?: T; op?: 'sub' | 'unsub' }

@Injectable({ providedIn: 'root' })
export class TopicClient {
  private socket: WebSocketSubject<Envelope> = webSocket<Envelope>('wss://api.example.com/ws');

  topic<T>(name: string): Observable<T> {
    return this.socket.multiplex(
      () => ({ topic: name, op: 'sub' }),        // sent on first subscription
      () => ({ topic: name, op: 'unsub' }),      // sent when the last subscriber leaves
      msg => msg.topic === name,
    ).pipe(share()) as Observable<T>;
  }

  publish<T>(name: string, data: T) { this.socket.next({ topic: name, data }); }
}

// usage in a component:
// prices = toSignal(inject(TopicClient).topic<{ sku: string; price: number }>('prices.eu'));
```

## In the wild
- AWS SNS (+ SQS per subscriber), EventBridge, Google Cloud Pub/Sub, Azure Service Bus topics.
- Kafka topics with consumer groups (each group is a subscription), RabbitMQ fanout/topic exchanges, NATS, MQTT, Redis Pub/Sub and Streams.
- Postgres `LISTEN/NOTIFY`, browser `BroadcastChannel`, Node `EventEmitter`, RxJS `Subject`.

## Trade-offs
| Pros | Cons |
|---|---|
| Publishers unaware of subscribers; add consumers freely | Hard to know who depends on a message (hidden coupling on schema) |
| Natural fan-out and parallel processing | Delivery guarantees vary; duplicates and loss must be designed for |
| Durable subscriptions absorb consumer downtime | Ordering only within partition/key, if at all |
| Scales horizontally by topic/partition | Broker becomes critical infrastructure to operate |

## Pitfalls
- **Assuming transient pub/sub is durable:** Redis Pub/Sub, `LISTEN/NOTIFY` and plain `Subject` drop messages when nobody listens. Use durable subscriptions where loss matters.
- **Non-idempotent subscribers:** redelivery causes double charges/emails. Dedupe on an event id.
- **Schema changes breaking unknown subscribers:** version payloads, add fields only, maintain a schema registry/contract tests.
- **Poison messages blocking a subscription:** configure retries with a dead-letter queue.
- **Topic explosion or one mega-topic:** name topics by business fact (`orders.placed`), partition by entity id.
- **Leaked subscriptions in UIs:** unsubscribe on destroy (`takeUntilDestroyed`, `toSignal`).

## Refactoring path
- **Introduce it:** replace the list of direct notifications with one publish of a fact; migrate one consumer at a time to subscribe; keep the direct call behind a flag until the subscriber is verified; remove the calls.
- **Remove it / simplify:** a topic with one subscriber that needs a reply → turn into a direct call or a point-to-point queue; in-process pub/sub with one listener → a function call or Observer.

## Related patterns
- [Observer](../behavioral/observer.md) — in-process, the subject knows its observers directly.
- [Event-Driven](../architectural/event-driven.md) — the architectural style built on pub/sub.
- [Message Broker](../enterprise/message-broker.md) — the intermediary implementing topics.
- [Message Queue](../enterprise/message-queue.md) — point-to-point counterpart (competing consumers).
- [Event Aggregator](../behavioral/event-aggregator.md) — in-app single hub for many event sources.
- [Idempotent Receiver](../emergent/idempotent-receiver.md) — required for at-least-once subscribers.
