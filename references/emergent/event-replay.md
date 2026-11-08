# Event Replay

**Category:** Emergent / modern · **Also known as:** Projection rebuild, replay, reprocessing, rehydration · **Related:** Event Sourcing, CQRS, Snapshot, Idempotent Receiver, Data Pipeline

## Intent
Reconstruct state — an aggregate, a read model, a search index, a bug's exact conditions — by re-applying a stored, ordered log of events from the beginning (or from a snapshot), instead of depending on a mutable current-state copy.

## Problem
- Read models drift from the truth (a bug in a projection, a missed message) and there is no way to fix them except manual SQL patching.
- Scenario: an order-history projection had a bug that ignored `OrderPartiallyRefunded` for three weeks. Dashboards show wrong revenue. With events stored, you fix the handler and rebuild the projection; with only current state, the information is gone.
- Scenario: product wants a new "average time from cart to checkout" report for last year. With an event log you replay history into a new projection; with CRUD tables you start collecting today.
- Smells: "backfill" scripts that reimplement business logic; projections that can't be rebuilt; bugs that can't be reproduced because the state that triggered them is overwritten.

## Solution and structure
```
 event store / log (append-only, ordered, immutable)
   e1 e2 e3 ... eN ─► replayer ─► handler(state, event) ─► new projection (shadow table)
                       │ position/checkpoint                 │
                       └─ resume from checkpoint             └─ swap when caught up
```
- **Event log**: durable, ordered, immutable; per-stream order (aggregate) or global order (log offset). Kafka topic with sufficient retention, an events table, EventStoreDB.
- **Handlers / projectors**: deterministic `apply(state, event) → state`, no side effects to the outside world during replay.
- **Checkpoint**: last processed position, stored *with* the projection so they commit together.
- **Snapshot (optional)**: periodic state dump plus its position, to shorten replays.
- **Cut-over**: build into a new table/index (blue/green), catch up to live, then switch readers.

Replay is used for: (1) rehydrating an aggregate on each command (Event Sourcing), (2) rebuilding or creating read models (CQRS), (3) reprocessing after a bug fix, (4) debugging/time travel and test fixtures.

## When to use
- You already have an append-only event log (event sourcing, Kafka with long retention, CDC stream, audit table).
- Read models or search indexes that must be rebuildable and evolvable.
- Analytics or features that need historical behavior, not just current state.
- Reproducing production bugs deterministically.

## When NOT to use
- You only have current-state CRUD tables; building an event log just to replay it is Event Sourcing — a big decision, not a side feature.
- Small projections cheaply recomputable with a SQL query or materialized view (`REFRESH MATERIALIZED VIEW`).
- Logs with external side effects baked into handlers (sending emails, charging cards) unless you can disable them during replay.
- Very long histories with no snapshotting and tight rebuild time budgets — plan snapshots first.

## Example — Python
A projector that can run live or in replay mode, with checkpoint and projection committed atomically, and side effects suppressed on replay.

```python
from collections.abc import AsyncIterator, Callable, Awaitable
from dataclasses import dataclass
from decimal import Decimal

@dataclass(frozen=True)
class Event:
    position: int
    type: str
    order_id: str
    data: dict

class RevenueProjection:
    def __init__(self, db, table: str, notify: Callable[[str], Awaitable[None]] | None) -> None:
        self.db, self.table, self.notify = db, table, notify    # notify=None during replay

    async def apply(self, e: Event) -> None:
        match e.type:
            case "OrderPaid":
                await self.db.execute(f"INSERT INTO {self.table} (order_id, amount) VALUES ($1, $2) "
                                      "ON CONFLICT (order_id) DO NOTHING", e.order_id, Decimal(e.data["amount"]))
                if self.notify:
                    await self.notify(e.order_id)
            case "OrderPartiallyRefunded":
                await self.db.execute(f"UPDATE {self.table} SET amount = amount - $2 WHERE order_id = $1",
                                      e.order_id, Decimal(e.data["amount"]))
            case _:
                pass                                            # ignore unknown types: forward-compatible

async def replay(events: AsyncIterator[Event], proj: RevenueProjection, name: str) -> None:
    async for e in events:                                      # ordered from checkpoint
        async with proj.db.transaction():                       # projection + checkpoint atomically
            await proj.apply(e)
            await proj.db.execute("INSERT INTO checkpoints (name, position) VALUES ($1, $2) "
                                  "ON CONFLICT (name) DO UPDATE SET position = $2", name, e.position)

# Rebuild: replay(store.read_all(from_=0), RevenueProjection(db, "revenue_v2", notify=None), "revenue_v2")
# then point readers at revenue_v2 and drop revenue_v1.
```
Table names must come from code, never user input (they are interpolated).

## Example — TypeScript
Rehydrating an aggregate is a fold (`reduce`) over its events; the same reducer powers NgRx/Redux devtools time travel.

```ts
type CartEvent =
  | { type: 'ItemAdded'; sku: string; qty: number; price: number }
  | { type: 'ItemRemoved'; sku: string }
  | { type: 'CheckedOut'; at: string };

interface Cart { items: ReadonlyMap<string, { qty: number; price: number }>; checkedOut: boolean }
const empty: Cart = { items: new Map(), checkedOut: false };

export function apply(state: Cart, e: CartEvent): Cart {     // pure, deterministic
  switch (e.type) {
    case 'ItemAdded': {
      const items = new Map(state.items);
      const prev = items.get(e.sku)?.qty ?? 0;
      items.set(e.sku, { qty: prev + e.qty, price: e.price });
      return { ...state, items };
    }
    case 'ItemRemoved': {
      const items = new Map(state.items); items.delete(e.sku);
      return { ...state, items };
    }
    case 'CheckedOut':
      return { ...state, checkedOut: true };
  }
}

export const rehydrate = (events: readonly CartEvent[], from: Cart = empty): Cart =>
  events.reduce(apply, from);                                  // `from` can be a snapshot
```
Exhaustive `switch` on the discriminated union makes TS flag unhandled event types when one is added.

## In the wild
- EventStoreDB/KurrentDB subscriptions with catch-up, Marten (projections rebuild), Axon (replay with tracking tokens), Kafka consumer offset reset (`kafka-consumer-groups --reset-offsets`), Kafka Streams state store restore from changelog.
- Debezium CDC + replay into new sinks; AWS Kinesis/EventBridge Archive & Replay.
- Redux/NgRx devtools time travel; database WAL replay and point-in-time recovery; Git (commits replayed by rebase).

## Trade-offs
| Pros | Cons |
|---|---|
| Read models are disposable and fixable | Replays of long histories are slow without snapshots |
| New views over full history, retroactively | Handlers must stay compatible with old event versions |
| Deterministic reproduction of bugs | Side effects must be suppressed during replay |
| Natural audit trail | Requires log retention and storage cost |

## Pitfalls
- Side effects re-firing during replay (emails, webhooks, payments). Separate projections from reactions, or gate them with a replay flag.
- Non-deterministic handlers (reading `now()`, calling external services, random ids). Use data from the event only.
- Checkpoint stored separately from projection writes — crash leads to double-applied or skipped events. Commit together, and make handlers idempotent.
- Event schema evolution ignored: old events no longer parse. Version events and upcast them on read.
- Rebuilding in place while serving reads: users see half-built data. Build a new table/index and swap.
- Kafka retention shorter than replay needs — use compacted topics, tiered storage, or archive to S3.

## Refactoring path
- **Introduce it:** make projectors pure functions of events; store checkpoints with projections; add a "rebuild into new table" command; test by rebuilding in staging and diffing against the live projection; add snapshots when replay time grows.
- **Remove it / simplify:** if a projection is cheap to compute from current state, replace it with a SQL view or materialized view and drop its replay machinery.

## Related patterns
- [Event Sourcing](../architectural/event-sourcing.md) — the event log that replay consumes.
- [CQRS](../cloud/cqrs.md) — read models are the main replay target.
- [Snapshot](../data-access/snapshot.md) — bounds replay time.
- [Idempotent Receiver](../emergent/idempotent-receiver.md) — handlers must tolerate re-delivery during replay.
- [Data Pipeline](../emergent/data-pipeline.md) — backfills are replays through a pipeline.
