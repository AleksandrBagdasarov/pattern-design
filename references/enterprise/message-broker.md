# Message Broker

**Category:** Enterprise integration · **Also known as:** Message-oriented middleware (MOM), event broker · **Related:** Message Queue, Publish-Subscribe, Broker, ESB, Event-Driven, Idempotent Receiver

## Intent
Put an intermediary between producers and consumers that receives, stores and routes messages, so services communicate asynchronously without knowing each other's location, availability or number.

## Problem
- Service A calls B, C and D synchronously after each order; if D is down, A fails, and adding E means changing A.
- Traffic spikes overwhelm downstream services that cannot scale as fast as the frontend.
- Point-to-point integrations multiply: n services → up to n² connections, each with its own retry logic.
- Scenario: `OrderPlaced` must reach billing, inventory, email and analytics; analytics is slow and sometimes offline.

## Solution and structure
Producers publish to a named destination (exchange/topic/queue); the broker persists messages and delivers them to consumers according to routing rules. Producers and consumers depend only on the broker and the message schema.

```
order-svc ──publish OrderPlaced──> [ Broker ]
                                    ├─ topic/exchange "orders"
                                    │    ├─> queue billing    ──> billing-svc
                                    │    ├─> queue inventory  ──> inventory-svc
                                    │    └─> queue analytics  ──> analytics (catches up later)
```
Two broker families with different models:
- **Queue/routing brokers (RabbitMQ, ActiveMQ, Amazon MQ, SQS+SNS):** messages are deleted once acknowledged; rich routing (direct, topic, headers, fanout); per-message acks, priorities, TTL, DLQ.
- **Log-based brokers (Kafka, Redpanda, Kinesis, Pulsar):** an append-only partitioned log retained for days or forever; consumers track offsets; replay is possible; ordering per partition; scales to very high throughput.
- **Lightweight/cloud:** Redis Streams, NATS/JetStream, Google Pub/Sub, Azure Service Bus, EventBridge (routing rules on event content).

## When to use
- One event, many independent consumers (pub-sub) or work distributed to many workers (queue).
- Temporal decoupling: consumers may be down or slow without failing producers.
- Load leveling and backpressure for bursty traffic.
- Event streaming, replay, or feeding analytics (log-based).

## When NOT to use
- The caller needs the answer now (pricing, auth checks): use a synchronous call with timeouts/retries.
- A monolith or a couple of services where an in-process event bus or a DB-backed job table suffices.
- Your team cannot operate it: prefer a managed service (SQS/SNS, Pub/Sub, Confluent/MSK) over self-hosting Kafka.
- To fake request/response everywhere ("RPC over queues"): you get async complexity with sync coupling.

## Example — Python
RabbitMQ with `aio-pika`: a topic exchange fanned out to per-service queues.

```python
import json
import aio_pika
from aio_pika import DeliveryMode, ExchangeType, Message

async def publish_order_placed(conn: aio_pika.abc.AbstractRobustConnection,
                               order_id: str, total_cents: int) -> None:
    async with conn.channel(publisher_confirms=True) as ch:
        ex = await ch.declare_exchange("orders", ExchangeType.TOPIC, durable=True)
        body = json.dumps({"order_id": order_id, "total_cents": total_cents}).encode()
        await ex.publish(
            Message(body, content_type="application/json", message_id=order_id,
                    delivery_mode=DeliveryMode.PERSISTENT, type="OrderPlaced.v1"),
            routing_key="order.placed")

async def run_billing_consumer(conn: aio_pika.abc.AbstractRobustConnection) -> None:
    ch = await conn.channel()
    await ch.set_qos(prefetch_count=20)                     # bounded in-flight work
    ex = await ch.declare_exchange("orders", ExchangeType.TOPIC, durable=True)
    q = await ch.declare_queue("billing.order-placed", durable=True,
                               arguments={"x-dead-letter-exchange": "orders.dlx"})
    await q.bind(ex, routing_key="order.placed")
    async with q.iterator() as messages:
        async for msg in messages:
            async with msg.process(requeue=False):          # ack on success, reject → DLX on error
                await charge(json.loads(msg.body))           # must be idempotent

# conn = await aio_pika.connect_robust("amqp://guest:guest@localhost/")
```

## Example — TypeScript
Node with `kafkajs`: log-based broker, consumer group, key = order ID for per-order ordering.

```ts
import { Kafka } from 'kafkajs';

const kafka = new Kafka({ clientId: 'order-svc', brokers: ['localhost:9092'] });

export async function publishOrderPlaced(orderId: string, totalCents: number): Promise<void> {
  const producer = kafka.producer({ idempotent: true });   // safe producer retries
  await producer.connect();
  await producer.send({
    topic: 'orders.v1',
    messages: [{
      key: orderId,                                         // same key → same partition → ordered
      value: JSON.stringify({ type: 'OrderPlaced', orderId, totalCents }),
      headers: { 'schema-version': '1' },
    }],
  });
  await producer.disconnect();                              // reuse one producer in real code
}

export async function runInventoryConsumer(): Promise<void> {
  const consumer = kafka.consumer({ groupId: 'inventory' }); // each group gets every message
  await consumer.connect();
  await consumer.subscribe({ topic: 'orders.v1', fromBeginning: false });
  await consumer.run({
    eachMessage: async ({ message }) => {
      const event = JSON.parse(message.value!.toString()) as { orderId: string };
      await reserveStock(event.orderId);                    // offset committed after success
    },
  });
}
declare function reserveStock(orderId: string): Promise<void>;
```
kafkajs is in low-maintenance mode; Confluent's `@confluentinc/kafka-javascript` is the actively developed alternative — check current status before choosing.

## In the wild
- RabbitMQ, Kafka, Redpanda, Pulsar, NATS, ActiveMQ/Artemis, Redis Streams.
- Managed: Amazon SNS/SQS, EventBridge, MSK, Kinesis; Google Pub/Sub; Azure Service Bus/Event Hubs.
- Frameworks: Celery/Dramatiq (on RabbitMQ/Redis), FastStream, NestJS microservices transports, Spring Cloud Stream, MassTransit.

## Trade-offs
| Pros | Cons |
|---|---|
| Producers and consumers decoupled in space, time and number | Eventual consistency; harder to reason about flows |
| Absorbs spikes; consumers scale independently | Another critical stateful system to run and monitor |
| Add consumers without changing producers | At-least-once delivery requires idempotent consumers |
| Replay and audit (log-based) | Debugging/tracing across hops needs correlation IDs and tracing |
| Built-in retries, DLQs, routing | Schema evolution must be managed explicitly |

## Pitfalls
- **Assuming exactly-once delivery.** Fix: idempotent consumers (dedupe by message ID) — see Idempotent Receiver.
- **Publishing after the DB commit without an outbox** → lost events on crash; before commit → phantom events. Fix: transactional outbox.
- **Global ordering expectations.** Fix: order only per key/partition; design for out-of-order across keys.
- **Unbounded prefetch / no backpressure** → consumer OOM. Fix: prefetch/`max.poll.records` limits.
- **Poison messages retried forever.** Fix: retry limit + dead-letter queue + alerting.
- **No schema versioning.** Fix: version field or schema registry; additive changes only.

## Refactoring path
- **Introduce it:** pick one fire-and-forget side effect currently called synchronously (email, analytics); publish an event instead, via an outbox; move the call into a consumer; add DLQ and monitoring; repeat for other side effects.
- **Remove it / simplify:** if there is one producer and one consumer in the same deployable, replace the broker with an in-process call or a DB-backed task queue.

## Related patterns
- [Message Queue](../enterprise/message-queue.md) — the point-to-point channel a broker provides; outbox, DLQ, delivery semantics.
- [Publish-Subscribe](../architectural/publish-subscribe.md) — the one-to-many messaging style brokers implement.
- [Broker](../architectural/broker.md) — the general architectural intermediary pattern.
- [ESB](../enterprise/esb.md) — a broker plus central transformation and orchestration.
- [Event-Driven](../architectural/event-driven.md) — the architecture built on top of a broker.
- [Idempotent Receiver](../emergent/idempotent-receiver.md) — required for at-least-once delivery.
