# Idempotent Receiver

**Category:** Emergent / modern · **Also known as:** Idempotent consumer, deduplication, idempotency key · **Related:** Retry, Message Queue, Saga, Task Queue, Event-Driven

## Intent
Make a receiver (HTTP endpoint, message consumer, job handler) safe to invoke more than once with the same request, so that duplicates caused by retries and at-least-once delivery produce the same effect and the same response as a single invocation.

## Problem
- Networks give you at-least-once, never exactly-once delivery. The client times out after the server committed; the broker redelivers after the consumer crashed before ack; the user double-clicks "Pay".
- Scenario: `POST /payments` charges a card, then the response is lost. The mobile app retries and the customer is charged twice.
- Scenario: an SQS consumer sends a welcome email, then crashes before deleting the message. The message reappears after the visibility timeout and the user gets two emails.
- Smells: retries added on the client with no server-side dedup; "it only happens under load"; reconciliation scripts that delete duplicate rows.

## Solution and structure
Pick the cheapest technique that makes the effect idempotent:

1. **Natural idempotency** — design the operation so repeats are harmless: `PUT /users/42 {…}` (set, not increment), `UPSERT … ON CONFLICT DO NOTHING`, `status = 'shipped' WHERE status = 'paid'`, set membership instead of append.
2. **Business unique key** — a unique constraint on something the request already carries (`order_id` for a shipment, `provider_event_id` for a webhook).
3. **Idempotency key + dedup table** — the client generates a unique key per *logical* operation (UUID) and sends it (`Idempotency-Key` header, message ID). The receiver stores `(key, request_fingerprint, status, response)`:

```
request(key) ─► INSERT key (status=in_progress) ──conflict?──► existing row:
                     │                                         completed → replay stored response
                     ▼                                         in_progress → 409 / retry later
               do the work + store response                     different fingerprint → 422
               in the SAME transaction (status=completed)
```
The key row and the side effect must commit atomically. When the side effect is external (a payment API), pass the key downstream too (Stripe accepts its own `Idempotency-Key`), so the whole chain is idempotent.

Stripe-style semantics worth copying: keys are client-generated, scoped per account and endpoint, expire after a retention window (Stripe: 24 hours); a replay returns the original status code and body, including errors; reusing a key with different parameters is an error. An IETF draft (`draft-ietf-httpapi-idempotency-key-header`) standardises the header; check its current status before citing it as final.

## When to use
- Any handler behind a retry policy, a queue with at-least-once delivery, or a webhook source (Stripe, GitHub, Shopify all redeliver).
- Non-idempotent HTTP methods (`POST`, `PATCH` with increments) that move money, send notifications or create resources.
- Saga steps and compensations, outbox relays, Kafka consumers that may re-read after a rebalance.

## When NOT to use
- Operations that are already naturally idempotent (`GET`, `PUT` full replace, `DELETE` by id, upserts): a dedup table adds a write for nothing.
- Pure, side-effect-free computations: just recompute.
- When the broker or platform already gives you dedup for your window (SQS FIFO 5-minute dedup, Kafka idempotent producer for the *producer* side) — but note these never cover your consumer's own side effects.

## Example — Python
FastAPI + SQLAlchemy async; the idempotency row and the payment are written in one transaction.

```python
import hashlib, json
from fastapi import APIRouter, Depends, Header, HTTPException
from fastapi.responses import JSONResponse
from sqlalchemy import select
from sqlalchemy.exc import IntegrityError
from sqlalchemy.ext.asyncio import AsyncSession

router = APIRouter()

@router.post("/payments")
async def create_payment(body: PaymentIn, idempotency_key: str = Header(...),
                         db: AsyncSession = Depends(get_db), psp: Psp = Depends(get_psp)):
    fp = hashlib.sha256(json.dumps(body.model_dump(mode="json"), sort_keys=True).encode()).hexdigest()
    try:
        async with db.begin_nested():
            db.add(IdemKey(key=idempotency_key, fingerprint=fp, status="in_progress"))
    except IntegrityError:
        row = await db.scalar(select(IdemKey).where(IdemKey.key == idempotency_key))
        if row.fingerprint != fp:
            raise HTTPException(422, "Idempotency-Key reused with a different body")
        if row.status != "completed":
            raise HTTPException(409, "Request with this key is still in progress")
        return JSONResponse(row.response_body, status_code=row.response_status)

    charge = await psp.charge(body.amount, body.currency, idempotency_key=idempotency_key)
    payment = Payment(amount=body.amount, psp_charge_id=charge.id)
    db.add(payment)
    await db.flush()
    resp = {"id": payment.id, "status": charge.status}
    await db.execute(IdemKey.__table__.update().where(IdemKey.key == idempotency_key)
                     .values(status="completed", response_status=201, response_body=resp))
    await db.commit()
    return JSONResponse(resp, status_code=201)
```
`PaymentIn`, `IdemKey`, `Payment`, `get_db`, `get_psp` are the app's own models and dependencies. Passing the same key to the PSP covers a crash between `charge` and `commit`.

## Example — TypeScript
Node Kafka/SQS-style consumer with a dedup table keyed by message ID (Postgres via `pg`).

```ts
import type { Pool } from 'pg';

interface OrderPlaced { messageId: string; orderId: string; email: string }

export function makeHandler(pool: Pool, mailer: { sendConfirmation(email: string, orderId: string): Promise<void> }) {
  return async (msg: OrderPlaced): Promise<void> => {
    const client = await pool.connect();
    try {
      await client.query('BEGIN');
      const { rowCount } = await client.query(
        'INSERT INTO processed_messages (message_id, consumer) VALUES ($1, $2) ON CONFLICT DO NOTHING',
        [msg.messageId, 'order-confirmation'],
      );
      if (rowCount === 0) { await client.query('ROLLBACK'); return; }  // duplicate: ack and skip
      // Natural idempotency for the DB effect:
      await client.query(
        "UPDATE orders SET status = 'confirmed' WHERE id = $1 AND status = 'placed'", [msg.orderId]);
      await client.query(
        'INSERT INTO outbox (topic, payload) VALUES ($1, $2)',
        ['email.confirmation', JSON.stringify({ email: msg.email, orderId: msg.orderId })]);
      await client.query('COMMIT');   // dedup row + effects commit together; email sent by outbox relay
    } catch (e) {
      await client.query('ROLLBACK');
      throw e;                         // broker redelivers; dedup row was rolled back too
    } finally {
      client.release();
    }
  };
}
```
On the Angular side, generate one key per user intent (`crypto.randomUUID()` when the form opens or on first submit) and reuse it for retries; never generate a fresh key inside a `retry()` operator.

## In the wild
- Stripe, Adyen, PayPal, Square `Idempotency-Key` headers; AWS APIs `ClientToken`/`ClientRequestToken` (EC2, ECS); AWS Lambda Powertools `@idempotent` (Python/TS) backed by DynamoDB.
- Kafka idempotent producer and transactions (`enable.idempotence`, exactly-once within Kafka only); SQS FIFO `MessageDeduplicationId`; Celery `acks_late` requires idempotent tasks.
- SQL: `INSERT … ON CONFLICT`, unique constraints; DynamoDB conditional writes (`attribute_not_exists`).

## Trade-offs
| Pros | Cons |
|---|---|
| Retries become safe; enables at-least-once everywhere | Extra write per request and a table to expire/clean |
| Clients get the same response on replay | Requires atomicity between dedup record and effect |
| Removes duplicate-cleanup scripts | External side effects need downstream idempotency too |
| Clear contract for API consumers | In-progress/concurrent duplicates need explicit handling |

## Pitfalls
- Checking "seen?" then writing in two steps without a unique constraint: two concurrent duplicates both pass. Rely on the constraint or a conditional write.
- Storing the dedup row in a different store/transaction than the effect: a crash in between loses or duplicates work. Use the same DB, or the outbox.
- Generating the key server-side or per retry attempt — it must identify the logical operation and come from the caller.
- Not fingerprinting the payload: a buggy client reuses a key for a different request and silently gets the old response.
- Sending emails/charging cards inline after commit with no key: use an outbox or pass the key to the provider.
- Never expiring keys (unbounded table) or expiring them sooner than the maximum retry window.

## Refactoring path
- **Introduce it:** first look for natural idempotency (turn increments into sets, add a business unique key); then add a client-supplied key and a dedup table on the riskiest endpoint or consumer; move effects into the same transaction or an outbox; add a TTL cleanup job.
- **Remove it / simplify:** if the operation became naturally idempotent (upsert by business key), drop the dedup table; replace hand-rolled dedup with Powertools or the broker's feature where it covers the whole effect.

## Related patterns
- [Retry](../behavioral/retry.md) — retry is only safe against an idempotent receiver.
- [Message Queue](../enterprise/message-queue.md) — at-least-once delivery is why consumers must dedup.
- [Saga](../cloud/saga.md) — every step and compensation must be idempotent.
- [Task Queue](../emergent/task-queue.md) — background jobs are retried and redelivered.
- [Event-Driven](../architectural/event-driven.md) — the cluster where this pattern is mandatory.
