# Saga

**Category:** Cloud / distributed · **Also known as:** Long-running transaction, Process manager (orchestrated form), Compensating transaction · **Related:** Idempotent Receiver, Message Queue, Publish-Subscribe, CQRS, Event-Driven, Retry, Leader Election

## Intent
Implement a business transaction that spans several services or databases as a sequence of local transactions, each publishing an event or message that triggers the next, and undo completed steps with compensating actions when a later step fails.

## Problem
- Placing an order touches Orders, Payments, Inventory and Shipping, each with its own database. A distributed ACID transaction (2PC/XA) is unavailable, too slow, or couples services' availability.
- Code calls the services one after another over HTTP; when shipping fails after payment succeeded, the customer is charged for an order that does not exist.
- A process crash between "write to DB" and "publish event" leaves the system inconsistent (the dual-write problem).
- Retries create duplicate charges because steps are not idempotent.

Scenario: checkout must create the order, reserve stock, charge the card, and book a courier. If booking fails, release stock, refund the payment and mark the order cancelled.

## Solution and structure
Each step = local transaction + compensation. Steps after the **pivot** (the point of no return, often the payment capture) must be retriable until they succeed, since they can no longer be compensated.

**Choreography**: no coordinator; each service reacts to events and emits its own.
```
Orders ──OrderCreated──► Inventory ──StockReserved──► Payments ──PaymentCaptured──► Shipping
   ◄──────────────── StockReleased ◄──── PaymentFailed ◄──────────────────────────────┘ (on failure)
```
**Orchestration**: a saga orchestrator (state machine) sends commands and handles replies.
```
             ┌─ReserveStock──► Inventory ─┐
Orchestrator ├─ChargeCard────► Payments  ─┼─► replies ──► next command or compensations in reverse
 (state DB)  └─BookCourier───► Shipping  ─┘
```
| | Choreography | Orchestration |
|---|---|---|
| Coupling | services know each other's events | services know only the orchestrator's commands |
| Visibility | flow is implicit, spread across services | flow is explicit in one place |
| Best for | 2–4 steps, simple flows | many steps, branches, timeouts, human steps |
| Risk | cyclic event dependencies, hard debugging | orchestrator becomes a "god service" |

**Required building blocks:**
- **Transactional outbox**: write the business change and the outgoing message to the same DB transaction (an `outbox` table); a relay (poller or CDC such as Debezium) publishes it. This fixes the dual write.
- **Idempotent receivers**: messages are delivered at least once; dedupe by message id or make handlers naturally idempotent.
- **Compensations**: semantic undo (refund, release, cancel), not a rollback; they must be idempotent and retriable.
- **Isolation countermeasures**: sagas are ACD without I. Use semantic locks (status `PENDING`), commutative updates, or re-reading values to avoid anomalies visible to other transactions.
- **Timeouts**: a step that never replies must eventually trigger compensation.

## When to use
- A business process spans services with separate databases and must end in a consistent outcome.
- Steps are long-running (seconds to days: waiting for payment confirmation, manual approval).
- Every step has a meaningful compensation, or can be placed after the pivot and retried until success.

## When NOT to use
- All data is in one database: use a single local ACID transaction. If your microservices need sagas for every operation, the service boundaries may be wrong.
- No meaningful compensation exists and the step cannot be retried safely (sending a physical letter): redesign (reorder steps, add a reservation step).
- Simple fire-and-forget side effects (send email after signup): an outbox + idempotent consumer is enough; no saga state machine needed.
- Hand-building an orchestrator with retries, timers and state persistence when a workflow engine fits (Temporal, AWS Step Functions, Azure Durable Functions, Camunda).

## Example — Python
Orchestrated saga with persistence-friendly steps and compensations in reverse order. In production, run this inside a durable workflow engine (e.g. Temporal's Python SDK) so a crash resumes mid-saga.
```python
from collections.abc import Awaitable, Callable
from dataclasses import dataclass, field
import logging

log = logging.getLogger("saga")
Action = Callable[[dict], Awaitable[None]]

@dataclass
class Step:
    name: str
    action: Action
    compensation: Action | None = None       # None: step is after the pivot, retry until success

@dataclass
class Saga:
    steps: list[Step]
    done: list[Step] = field(default_factory=list)

    async def run(self, ctx: dict) -> bool:
        for step in self.steps:
            try:
                await step.action(ctx)        # each action: local tx + idempotency key = ctx["order_id"]
                self.done.append(step)
            except Exception:
                log.exception("step %s failed; compensating", step.name)
                await self._compensate(ctx)
                return False
        return True

    async def _compensate(self, ctx: dict) -> None:
        for step in reversed(self.done):
            if step.compensation:
                await step.compensation(ctx)  # must be idempotent; on failure, retry/park for ops

async def checkout(order_id: str, inventory, payments, shipping, orders) -> bool:
    saga = Saga([
        Step("create_order", orders.create_pending, orders.cancel),
        Step("reserve_stock", inventory.reserve, inventory.release),
        Step("charge_card", payments.charge, payments.refund),       # pivot in practice
        Step("book_courier", shipping.book),                           # must be retriable
        Step("confirm_order", orders.confirm),
    ])
    return await saga.run({"order_id": order_id})
```
The in-memory `done` list is lost on a crash; that is exactly why real orchestrators persist saga state after each step (a `sagas` table or a workflow engine's event history).

## Example — TypeScript
Choreography side: an inventory consumer that is idempotent and uses a transactional outbox (Node, `pg`).
```ts
import { Pool } from 'pg';

const db = new Pool({ connectionString: process.env.DATABASE_URL });

interface OrderCreated { messageId: string; orderId: string; items: { sku: string; qty: number }[] }

export async function onOrderCreated(evt: OrderCreated): Promise<void> {
  const client = await db.connect();
  try {
    await client.query('BEGIN');
    // Idempotent receiver: a unique key on processed_messages rejects duplicates.
    const seen = await client.query(
      'INSERT INTO processed_messages(id) VALUES ($1) ON CONFLICT DO NOTHING', [evt.messageId]);
    if (seen.rowCount === 0) { await client.query('ROLLBACK'); return; }

    await client.query('SAVEPOINT reserve');
    let ok = true;
    for (const { sku, qty } of evt.items) {
      const r = await client.query(
        'UPDATE stock SET reserved = reserved + $2 WHERE sku = $1 AND available - reserved >= $2', [sku, qty]);
      if (r.rowCount === 0) { ok = false; break; }
    }
    if (!ok) await client.query('ROLLBACK TO SAVEPOINT reserve'); // undo partial reservations, keep dedupe row
    // Outbox: the event is committed atomically with the state change; a relay publishes it.
    await client.query('INSERT INTO outbox(topic, key, payload) VALUES ($1, $2, $3)', [
      ok ? 'inventory.stock-reserved' : 'inventory.stock-rejected', evt.orderId, JSON.stringify({ orderId: evt.orderId }),
    ]);
    await client.query('COMMIT');
  } catch (e) {
    await client.query('ROLLBACK');
    throw e;                                     // broker redelivers; handler is safe to rerun
  } finally {
    client.release();
  }
}
```

## In the wild
- Workflow engines (orchestration): Temporal (Python/TS/Go/Java SDKs), AWS Step Functions, Azure Durable Functions, Camunda/Zeebe, Netflix Conductor, Restate. AWS has also been adding durable-execution features to Lambda; check current docs.
- Frameworks: Eventuate Tram Sagas, Axon (JVM), MassTransit and NServiceBus sagas (.NET).
- Outbox and CDC: Debezium outbox event router, Kafka Connect; AWS DynamoDB Streams / EventBridge Pipes.
- Classic origin: Garcia-Molina & Salem, "Sagas" (1987); popularized for microservices by Chris Richardson.

## Trade-offs
| Pros | Cons |
|---|---|
| Business consistency across services without 2PC | Only eventual consistency; no isolation between sagas |
| Services stay independently available | Compensations add design and test effort per step |
| Handles long-running and human-in-the-loop flows | Debugging spans many services and messages |
| Orchestration makes the flow explicit and observable | Requires outbox, idempotency, timeouts, monitoring |

## Pitfalls
- **Dual write** (commit DB, then publish) loses or duplicates events. Use an outbox or CDC.
- **Non-idempotent steps/compensations** cause double charges or double refunds under redelivery. Use idempotency keys (payment providers such as Stripe support them).
- **Compensation that can fail** with no plan. Retry with backoff, then park in a dead-letter state with an alert for manual action.
- **Missing timeouts**: a lost reply leaves the saga hanging forever. Every awaited reply needs a deadline.
- **Dirty reads by other sagas** (stock looks available while reserved by a saga that will fail). Use explicit `PENDING`/`RESERVED` states.
- **Choreography sprawl**: 8 services reacting to each other's events with no one owning the flow. Switch to orchestration.
- **Ordering assumptions**: events for the same order on different partitions arrive out of order. Partition by the saga/aggregate id.

## Refactoring path
- **Introduce it:** start from sequential cross-service HTTP calls in one handler. Add idempotency keys to each downstream call. Add an outbox for events. Define a compensation per step, order steps so irreversible ones come last, then move the flow into an orchestrator (a workflow engine or persisted state machine) with timeouts and monitoring.
- **Remove it / simplify:** if the steps' data can live in one service and one database, merge them and use a local transaction. If only one side effect remains, replace the saga with an outbox + idempotent consumer.

## Related patterns
- [Idempotent Receiver](../emergent/idempotent-receiver.md): mandatory for every saga participant.
- [Message Queue](../enterprise/message-queue.md): carries commands and events between steps (with the outbox).
- [Publish-Subscribe](../architectural/publish-subscribe.md): the transport for choreographed sagas.
- [Event-Driven](../architectural/event-driven.md): the architectural style sagas usually live in.
- [CQRS](../cloud/cqrs.md): sagas issue commands; read models show saga progress.
- [Retry](../behavioral/retry.md): steps after the pivot are retried until they succeed.
- [Leader Election](../cloud/leader-election.md): needed if a hand-built orchestrator must run as a single instance.
