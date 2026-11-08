# Task Queue

**Category:** Emergent / modern · **Also known as:** Job queue, background jobs, work queue, worker queue · **Related:** Message Queue, Producer-Consumer, Idempotent Receiver, Retry, Rate Limiter, Scheduler

## Intent
Move slow, unreliable or deferred work out of the request path: the web process enqueues a small job description, returns immediately, and separate worker processes execute the job with retries, scheduling and concurrency limits.

## Problem
- HTTP handlers do work users shouldn't wait for, or work that can fail independently: sending emails, generating PDFs, calling slow partner APIs, resizing images, recomputing reports.
- Scenario: `POST /orders` sends a confirmation email, generates an invoice PDF and notifies the warehouse. The SMTP server is slow (3 s) and the warehouse API sometimes times out; checkout latency spikes, and a timeout rolls back a successfully paid order.
- Scenario: a FastAPI app uses `BackgroundTasks` for invoice generation; a deploy restarts the pod and in-flight invoices are silently lost.
- Smells: request timeouts caused by side work; `asyncio.create_task` fire-and-forget for important work; retry loops inside handlers; cron jobs that scan tables for "pending" rows.

## Solution and structure
```
 web/API ──enqueue(job, args)──► broker (Redis / RabbitMQ / SQS / Postgres)
   │ 202 + job id                       │
   ▼                                    ▼
 client polls / websocket       workers (N processes, concurrency limits)
                                   ├─ success → result backend / DB
                                   ├─ failure → retry with backoff
                                   └─ exhausted → dead-letter queue + alert
```
- **Producer**: enqueues a job with *identifiers*, not big payloads or ORM objects (`order_id`, not `order`).
- **Broker**: durable storage with delivery semantics (usually at-least-once), visibility timeouts/acks.
- **Workers**: separate deployable processes, scaled on queue depth; each job is idempotent.
- **Policies**: retries with exponential backoff and jitter, max attempts, timeouts, rate limits per queue, priorities, scheduled/delayed jobs.
- **Results/status**: stored in the DB (or result backend) so the client can poll `GET /jobs/{id}`.
- **Transactional enqueue**: to avoid "order committed but job lost" (or the reverse), enqueue via a transactional outbox, or use a DB-backed queue in the same transaction.

## When to use
- Work that takes longer than a request should (> ~500 ms) or that users need not wait for.
- Calls to unreliable third parties that need retries without holding a connection.
- Spiky workloads you want to smooth (bursty uploads, bulk imports) with controlled concurrency.
- Delayed or scheduled work (reminder emails in 24 h, nightly reports).

## When NOT to use
- The user needs the result now and it is fast: just do it inline.
- Fire-and-forget work whose loss is acceptable and which is short: FastAPI `BackgroundTasks` or a simple `asyncio` task is enough (accept loss on restart).
- Cross-service event distribution where many consumers react: that's Pub-Sub/Message Broker, not a task queue.
- Long multi-step business processes with compensation: use a workflow engine (Temporal, Step Functions) or a Saga, not chains of ad hoc tasks.

## Example — Python
Celery task with retries, backoff, idempotency and rate limiting; enqueue only after the DB commit.

```python
from celery import Celery
import httpx

app = Celery("shop", broker="redis://redis:6379/0")
app.conf.update(task_acks_late=True,              # ack after success: at-least-once
                task_reject_on_worker_lost=True,
                worker_prefetch_multiplier=1)

@app.task(bind=True, autoretry_for=(httpx.HTTPError,), retry_backoff=True,
          retry_backoff_max=600, retry_jitter=True, max_retries=8,
          rate_limit="20/m", soft_time_limit=30)
def notify_warehouse(self, order_id: int) -> None:
    with session_scope() as db:                    # app's SQLAlchemy session helper
        order = db.get(Order, order_id)
        if order is None or order.warehouse_notified_at is not None:
            return                                 # idempotent: already done or gone
        r = httpx.post("https://wh.example.com/shipments",
                       json={"order_id": order_id, "lines": order.lines_payload()},
                       headers={"Idempotency-Key": f"order-{order_id}"}, timeout=10)
        r.raise_for_status()
        order.mark_warehouse_notified()

# In the FastAPI handler: commit first, then enqueue with an id, never the ORM object.
async def create_order(payload, db) -> dict:
    order = await place_order(db, payload)
    await db.commit()
    notify_warehouse.delay(order.id)               # or write to an outbox in the same txn
    return {"id": order.id, "status": "accepted"}
```
Alternatives: Dramatiq, RQ, arq (asyncio-native), Taskiq, or Postgres-backed queues (procrastinate) when you want transactional enqueue.

## Example — TypeScript
BullMQ (Redis) producer and worker in Node; the Angular side shows status by polling.

```ts
import { Queue, Worker, type Job } from 'bullmq';

const connection = { host: 'redis', port: 6379 };
export const invoices = new Queue<{ orderId: string }>('invoices', { connection });

export async function requestInvoice(orderId: string): Promise<string> {
  const job = await invoices.add('generate', { orderId }, {
    jobId: `invoice-${orderId}`,                 // dedup: same id is not enqueued twice
    attempts: 5,
    backoff: { type: 'exponential', delay: 2_000 },
    removeOnComplete: 1_000,
  });
  return job.id!;
}

new Worker<{ orderId: string }>('invoices', async (job: Job<{ orderId: string }>) => {
  const pdf = await renderInvoicePdf(job.data.orderId);   // app function; must be idempotent
  await storeInvoice(job.data.orderId, pdf);
}, { connection, concurrency: 4, limiter: { max: 10, duration: 1_000 } });

declare function renderInvoicePdf(id: string): Promise<Uint8Array>;
declare function storeInvoice(id: string, pdf: Uint8Array): Promise<void>;
```
In Angular, poll the status endpoint with `timer(0, 2000).pipe(switchMap(() => http.get<JobStatus>(url)), takeWhile(s => s.state !== 'done', true))`, or push updates via SSE/WebSocket.

## In the wild
- Python: Celery, RQ, Dramatiq, arq, Huey, Taskiq, procrastinate; Django-Q; FastAPI `BackgroundTasks` (in-process, not durable).
- Node: BullMQ, Bee-Queue, Agenda, pg-boss, Graphile Worker; NestJS `@nestjs/bullmq`.
- Cloud: AWS SQS + Lambda/ECS workers, Google Cloud Tasks, Azure Queue Storage; Sidekiq (Ruby), Hangfire (.NET); Temporal for durable workflows.

## Trade-offs
| Pros | Cons |
|---|---|
| Fast responses; slow work isolated | Eventual completion: UX needs status/polling |
| Retries and backoff without blocking users | At-least-once delivery forces idempotent tasks |
| Workers scale independently on queue depth | Broker and workers to deploy, monitor and alert on |
| Smooths spikes; rate limits third-party calls | Debugging spans processes; ordering not guaranteed |

## Pitfalls
- Enqueuing before the DB commit: the worker runs, can't find the row (or the transaction rolls back and the job still runs). Enqueue after commit, or use an outbox / DB-backed queue.
- Passing large payloads or ORM objects as task arguments: stale data, serialization errors, broker bloat. Pass ids and re-read.
- Non-idempotent tasks with `acks_late`/retries: duplicate emails or charges. Check-and-mark, unique constraints, idempotency keys downstream.
- Visibility timeout (SQS) or broker ack timeout shorter than the task duration: the job is redelivered while still running.
- No dead-letter queue or alerting: failures disappear after max retries. Monitor queue depth, age of oldest job and DLQ size.
- Unbounded retries hammering a down dependency; cap attempts, add jitter, combine with a circuit breaker.
- Using in-process background tasks (`BackgroundTasks`, `create_task`) for work that must not be lost.

## Refactoring path
- **Introduce it:** find slow or flaky side effects in handlers; make each an idempotent function taking ids; enqueue it after commit; return 202 with a status resource; add retries, DLQ and queue-depth metrics; scale workers separately.
- **Remove it / simplify:** if tasks are fast, rarely fail and users wait for them anyway, call them inline; if only one tiny non-critical job exists, use the framework's in-process background tasks and drop the broker.

## Related patterns
- [Message Queue](../enterprise/message-queue.md) — the delivery mechanism underneath; task queues add job semantics on top.
- [Producer-Consumer](../concurrency/producer-consumer.md) — the in-process version of the same shape.
- [Idempotent Receiver](../emergent/idempotent-receiver.md) — required because jobs are retried and redelivered.
- [Retry](../behavioral/retry.md) and [Rate Limiter](../concurrency/rate-limiter.md) — per-task policies.
- [Scheduler](../concurrency/scheduler.md) — delayed and periodic jobs (Celery beat, BullMQ repeatable jobs).
