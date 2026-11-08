# Message Queue

**Category:** Enterprise integration · **Also known as:** Point-to-point channel, work queue · **Related:** Message Broker, Task Queue, Producer-Consumer, Idempotent Receiver, Retry, Saga

## Intent
Deliver each message to exactly one of the competing consumers through a durable buffer, so work is decoupled in time, retried on failure and spread across workers.

## Problem
- A request handler does slow work (PDF generation, webhooks, payment capture) and times out under load.
- A downstream system is intermittently unavailable; synchronous calls fail the user's request.
- Updating the DB and notifying another service must both happen, but a crash between them loses the notification.
- Scenario: `POST /payouts` writes a payout row and must call the bank API; the bank has rate limits and outages.

## Solution and structure
Producers enqueue messages; the queue persists them; workers receive, process and **acknowledge**. Unacknowledged messages become visible again after a timeout (visibility timeout / ack deadline) and are redelivered.

```
producer ──send──> [ queue ] ──receive──> worker 1 ─ack─┐
                     │       └─receive──> worker 2 ─ack─┤ (competing consumers)
                     └── after N failures ──> [ DLQ ] ──> alert / inspect / redrive
```
**Delivery semantics.** Most queues are **at-least-once**: a message may be delivered more than once (worker crashed after doing the work but before ack; timeout too short). At-most-once loses messages; exactly-once *processing* is achieved only by idempotent consumers or by transactional coupling of consume + side effect within one system (e.g. Kafka transactions for Kafka-to-Kafka). Design every consumer to be idempotent.

**Dead-letter queue (DLQ).** After `maxReceiveCount` failed attempts, the message moves to a DLQ instead of blocking or retrying forever. Monitor DLQ depth, fix the cause, then redrive.

**Transactional outbox.** To change the DB and send a message atomically, write the message into an `outbox` table in the *same* DB transaction as the business change. A relay (poller or CDC such as Debezium) reads unsent rows, publishes them, and marks them sent. Publication is at-least-once, so consumers dedupe by message ID.

```
BEGIN; UPDATE payouts ...; INSERT INTO outbox(id, topic, payload) ...; COMMIT;
relay: SELECT ... FROM outbox WHERE sent_at IS NULL FOR UPDATE SKIP LOCKED → publish → UPDATE sent_at
```

## When to use
- Background work, rate-limited or flaky downstream calls, load leveling.
- Each message must be processed once by one worker (commands, jobs), not broadcast.
- Cross-service workflows that must survive crashes (with outbox + idempotency).

## When NOT to use
- The client needs the result synchronously and quickly.
- Broadcasting an event to many consumers: use pub-sub (topic → one queue per consumer).
- Strict global ordering with high throughput: standard queues reorder; FIFO queues and partitions limit parallelism.
- Small apps: a DB table polled with `FOR UPDATE SKIP LOCKED` (or a library on top of it) can replace a separate broker.

## Example — Python
Outbox write in the same transaction, plus an idempotent SQS consumer (`boto3`).

```python
import json
from uuid import uuid4
import boto3
from sqlalchemy import text
from sqlalchemy.orm import Session

def request_payout(session: Session, payout_id: str, amount_cents: int) -> None:
    with session.begin():                                   # one atomic transaction
        session.execute(text("INSERT INTO payouts (id, amount_cents, status) "
                             "VALUES (:id, :a, 'requested')"), {"id": payout_id, "a": amount_cents})
        session.execute(text("INSERT INTO outbox (id, topic, payload) VALUES (:id, :t, :p)"),
                        {"id": str(uuid4()), "t": "payouts",
                         "p": json.dumps({"payout_id": payout_id, "amount_cents": amount_cents})})

sqs = boto3.client("sqs")

def consume(queue_url: str, session: Session) -> None:
    resp = sqs.receive_message(QueueUrl=queue_url, MaxNumberOfMessages=10,
                               WaitTimeSeconds=20, VisibilityTimeout=60)
    for m in resp.get("Messages", []):
        body = json.loads(m["Body"])
        with session.begin():
            # dedupe: unique constraint on processed_messages.id makes redelivery a no-op
            inserted = session.execute(text(
                "INSERT INTO processed_messages (id) VALUES (:id) ON CONFLICT DO NOTHING"),
                {"id": m["MessageId"]}).rowcount
            if inserted:
                send_to_bank(body["payout_id"], body["amount_cents"])  # bank call must accept an idempotency key
        sqs.delete_message(QueueUrl=queue_url, ReceiptHandle=m["ReceiptHandle"])  # = ack
        # on exception: no delete → redelivered after timeout → DLQ after maxReceiveCount
```
Prefer a business idempotency key (the outbox row id or `payout_id`) over the transport's `MessageId` when the producer may publish the same logical message twice (outbox relays can).

## Example — TypeScript
Node with BullMQ (Redis-backed queue): retries with backoff, failed jobs kept as a DLQ equivalent.

```ts
import { Queue, Worker } from 'bullmq';

const connection = { host: 'localhost', port: 6379 };
export const payouts = new Queue<{ payoutId: string; amountCents: number }>('payouts', { connection });

export async function enqueuePayout(payoutId: string, amountCents: number): Promise<void> {
  await payouts.add('send', { payoutId, amountCents }, {
    jobId: payoutId,                     // duplicate adds with the same id are ignored
    attempts: 5,
    backoff: { type: 'exponential', delay: 2_000 },
    removeOnComplete: true,
    removeOnFail: false,                 // keep failures for inspection / retry (DLQ role)
  });
}

new Worker<{ payoutId: string; amountCents: number }>('payouts', async (job) => {
  await bank.send(job.data.payoutId, job.data.amountCents, { idempotencyKey: job.data.payoutId });
}, { connection, concurrency: 5, limiter: { max: 10, duration: 1_000 } });

declare const bank: {
  send(id: string, cents: number, o: { idempotencyKey: string }): Promise<void>;
};
```

## In the wild
- Amazon SQS (standard: at-least-once, best-effort order; FIFO: ordered per group, dedup window), Azure Service Bus queues, Google Cloud Tasks, RabbitMQ queues.
- Celery, RQ, Dramatiq, arq, Procrastinate (Postgres), BullMQ, Sidekiq.
- Outbox: Debezium outbox event router, `pg-boss`, MassTransit/NServiceBus outbox, AWS DynamoDB Streams as outbox.

## Trade-offs
| Pros | Cons |
|---|---|
| Fast responses; slow work moves to workers | Eventual consistency; users need status/polling or push |
| Survives downstream outages with retries | Duplicates under at-least-once; idempotency is mandatory |
| Horizontal scaling via competing consumers | Ordering is weak or costs throughput |
| DLQ isolates poison messages | Another system to monitor (depth, age, DLQ) |
| Outbox gives atomic DB change + message | Outbox relay adds latency and an operational component |

## Pitfalls
- **Dual write** (commit DB, then publish): crash in between loses the message. Fix: transactional outbox.
- **Visibility timeout shorter than processing time:** the same message is processed twice concurrently. Fix: timeout > p99 processing, or extend it while working (heartbeat).
- **Non-idempotent side effects** (charging twice). Fix: dedupe table or idempotency keys on downstream APIs.
- **No DLQ / unbounded retries:** poison messages loop forever. Fix: `maxReceiveCount` + DLQ + alerts on DLQ depth.
- **Acking before work completes:** crash loses the message. Fix: ack after success.
- **Huge payloads in messages.** Fix: store the blob (S3) and send a reference (claim check).

## Refactoring path
- **Introduce it:** take the slowest synchronous side effect; write an outbox row instead of calling it; add a relay and a worker; make the worker idempotent; add DLQ and metrics; return `202 Accepted` with a status resource.
- **Remove it / simplify:** if the work is fast and the dependency reliable, call it inline; if you only need deferred jobs in one app, replace a separate broker with a DB-backed job queue.

## Related patterns
- [Message Broker](../enterprise/message-broker.md) — the infrastructure that hosts queues and topics.
- [Task Queue](../emergent/task-queue.md) — a queue specialized for background jobs in one application.
- [Producer-Consumer](../concurrency/producer-consumer.md) — the in-process version with a bounded buffer.
- [Idempotent Receiver](../emergent/idempotent-receiver.md) — makes at-least-once delivery safe.
- [Retry](../behavioral/retry.md) — backoff and jitter policy for redeliveries.
- [Saga](../cloud/saga.md) — multi-step workflows coordinated over queues with compensations.
