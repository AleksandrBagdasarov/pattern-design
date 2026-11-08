# Broker

**Category:** Architectural · **Also known as:** Object Request Broker (ORB), message broker architecture · **Related:** Message Broker, Client-Server, Publish-Subscribe, Proxy, Mediator, ESB

## Intent
Decouple distributed components by routing all requests through an intermediary **broker**: clients send a request to the broker, the broker locates a suitable server, forwards the request and returns the response (or delivers the message), so neither side needs to know the other's location, implementation or availability.

## Problem
- Clients hard-code server hostnames; moving or scaling a server breaks clients.
- Producers and workers run at different speeds and must not block each other.
- Components written in different languages must interoperate.
- Scenario: a document platform where the web API must hand PDF rendering, OCR and virus scanning to worker fleets that scale from 2 to 200 instances. The API must not track worker addresses or health.

## Solution and structure
```
 Client ─► client-side proxy ─► BROKER ─► server-side proxy (skeleton) ─► Server
                                  │  registry, routing, queues, retries
 Client ◄──────── response ◄──────┘
```
- **Broker:** registers servers, routes requests, buffers messages, may balance load and handle failover.
- **Client-side proxy (stub):** makes a remote call look local; marshals the request.
- **Server-side proxy (skeleton):** unmarshals and invokes the server.
- **Bridges:** connect brokers of different networks/protocols.
- Two flavors in practice: **request routing brokers** (CORBA ORB, gRPC with a proxy/load balancer, service mesh) and **message brokers** (RabbitMQ, Kafka, SQS) where delivery is asynchronous.

## When to use
- Many distributed producers and consumers with changing locations and counts.
- Work distribution with buffering and competing consumers.
- Language/platform-heterogeneous systems needing a neutral protocol (AMQP, gRPC, Kafka protocol).

## When NOT to use
- A single client talking to a single stable server: a direct HTTP call with DNS is enough.
- Ultra-low-latency paths where an extra hop matters.
- Teams without capacity to operate a broker cluster: prefer a managed service (SQS, Pub/Sub, Amazon MQ) or skip it.
- Smell: putting business rules in broker routing configuration.

## Example — Python
```python
# Celery: the broker (RabbitMQ/Redis) routes tasks to whichever worker is available.
# Producers never know worker hostnames; workers scale independently.
from celery import Celery

app = Celery("docs", broker="amqp://guest@rabbitmq//", backend="redis://redis/0")
app.conf.task_routes = {"docs.render_pdf": {"queue": "rendering"}, "docs.ocr": {"queue": "ocr"}}
app.conf.task_acks_late = True              # ack after success: a crashed worker's task is redelivered
app.conf.worker_prefetch_multiplier = 1     # fair dispatch for long tasks

@app.task(name="docs.render_pdf", bind=True, autoretry_for=(ConnectionError,),
          retry_backoff=True, max_retries=5)
def render_pdf(self, document_id: str) -> str:
    # must be idempotent: acks_late + retries mean it can run more than once
    output_key = f"pdf/{document_id}.pdf"
    ...                                     # fetch doc, render, upload to output_key
    return output_key

# API process (the client): enqueue via the broker, optionally wait on the result backend
def request_render(document_id: str) -> str:
    async_result = render_pdf.delay(document_id)
    return async_result.id                  # client polls /jobs/{id} instead of blocking

# worker processes:  celery -A docs worker -Q rendering --concurrency=8
```

## Example — TypeScript
```ts
// Request-reply through RabbitMQ (amqplib): client and server know only queue names, not each other.
import amqp from 'amqplib';
import { randomUUID } from 'node:crypto';

export async function quoteViaBroker(sku: string, qty: number): Promise<number> {
  const conn = await amqp.connect('amqp://rabbitmq');
  const ch = await conn.createChannel();
  const { queue: replyTo } = await ch.assertQueue('', { exclusive: true });
  const correlationId = randomUUID();

  const reply = new Promise<number>((resolve, reject) => {
    const timer = setTimeout(() => reject(new Error('pricing timeout')), 2000);
    void ch.consume(replyTo, msg => {
      if (msg?.properties.correlationId !== correlationId) return;
      clearTimeout(timer);
      resolve(JSON.parse(msg.content.toString()).price);
    }, { noAck: true });
  });

  ch.sendToQueue('pricing.requests', Buffer.from(JSON.stringify({ sku, qty })), { correlationId, replyTo });
  try { return await reply; } finally { await conn.close(); }
}
// A pricing worker consumes 'pricing.requests' and replies to msg.properties.replyTo with the same correlationId.
// In production reuse one connection/channel; opening one per call is only for brevity.
```

## In the wild
- Message brokers: RabbitMQ, ActiveMQ/Amazon MQ, Kafka, NATS, AWS SQS/SNS, Azure Service Bus, Google Pub/Sub.
- Request brokers: CORBA ORBs, Java RMI registry, gRPC behind Envoy, service meshes, API gateways.
- Python task queues (Celery, Dramatiq, RQ) and Node BullMQ use a broker underneath.

## Trade-offs
| Pros | Cons |
|---|---|
| Location transparency; servers move and scale freely | Extra hop: latency and another component to run |
| Buffering decouples speeds and availability | Broker is a critical dependency (needs HA, monitoring) |
| Load balancing via competing consumers | Debugging spans processes; needs tracing/correlation ids |
| Interoperability across languages | Delivery semantics (duplicates, ordering) leak to app code |

## Pitfalls
- **Assuming exactly-once:** brokers redeliver. Make handlers idempotent; use idempotency keys.
- **Acking before work is done:** a crash loses the message. Ack after success (`acks_late`, manual ack).
- **Unbounded queues:** a slow consumer fills disk/memory. Set TTLs, max lengths, alerts on queue depth.
- **No dead-letter queue:** a poison message retries forever. Configure DLQs and inspect them.
- **Synchronous request-reply everywhere:** you rebuilt RPC with more latency. Use direct calls for sync needs, the broker for async work.
- **Large payloads in messages:** store blobs in object storage, pass references (claim check).

## Refactoring path
- **Introduce it:** take a slow synchronous call (PDF rendering in the request handler), move it into a task consumed from a queue, return a job id, and poll or push completion. Add idempotency before adding retries.
- **Remove it / simplify:** if a broker carries only synchronous request-reply between two fixed services, replace it with direct HTTP/gRPC plus service discovery; for low volume, a database-backed job table can replace a broker.

## Related patterns
- [Message Broker](../enterprise/message-broker.md) — the integration-level component this style relies on.
- [Client-Server](../architectural/client-server.md) — direct style without an intermediary.
- [Publish-Subscribe](../architectural/publish-subscribe.md) — the broker's fan-out mode.
- [Proxy](../structural/proxy.md) — client stubs and server skeletons are remote proxies.
- [Mediator](../behavioral/mediator.md) — in-process analogue: objects talk through a hub.
- [ESB](../enterprise/esb.md) — a broker plus transformation and orchestration.
