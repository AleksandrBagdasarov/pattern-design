# Snapshot

**Category:** Data access · **Also known as:** Point-in-time copy, versioned state, aggregate snapshot · **Related:** Event Sourcing, Event Replay, Memento, CQRS, Caching

## Intent
Store a point-in-time copy of an object's or dataset's state, so you can rebuild it quickly or read it as it was at that moment.

## Problem
Two common forms of the same force: "recomputing current/past state is too slow or impossible".
- **Event-sourced aggregates:** loading an account means replaying 80,000 events; latency grows with age.
- **Historical reads:** an invoice must show the customer's address and prices *as they were* when issued, but the `customers` and `products` rows have since changed.
- **Consistent reads:** a report or export must see one consistent state while writes continue.
- Scenario: a ledger service replays every transaction since 2019 on each request; p99 latency is 2s.

## Solution and structure
**Event-store snapshot:** every N events (or on a schedule) persist `(aggregate_id, version, state)`. To load: read the latest snapshot, then replay only events with `version > snapshot.version`.

```
events:   e1 e2 ... e1000 | e1001 ... e1037
snapshot:            [state@1000]
load = snapshot@1000 + apply(e1001..e1037)
```
**Business snapshot (denormalized copy):** copy the data you must preserve into the record at the moment of the fact (`invoice.billing_address`, `order_line.unit_price`). This is not duplication; it is a different fact.

**Storage/DB snapshot:** MVCC transaction snapshots (`REPEATABLE READ`), table versioning (temporal tables), volume/DB backups.

## When to use
- Event Sourcing with long-lived aggregates (measure replay time first).
- Documents that must be immutable once issued: invoices, contracts, orders, audit records.
- Reports or exports needing a consistent view.
- Undo/restore points for user-edited documents.

## When NOT to use
- Event-sourced aggregates with short streams (dozens of events): replay is cheap; snapshots add schema-versioning burden.
- As a cache for ordinary CRUD reads: use caching with invalidation instead.
- Copying data that should always be current (a user's display name in a comment feed): reference it.

## Example — Python
Snapshot store for an event-sourced account; snapshot every 100 events.

```python
from dataclasses import dataclass, asdict
from typing import Protocol

@dataclass
class Account:
    id: str
    balance_cents: int = 0
    version: int = 0
    def apply(self, event: dict) -> None:
        match event["type"]:
            case "Deposited": self.balance_cents += event["cents"]
            case "Withdrawn": self.balance_cents -= event["cents"]
        self.version = event["version"]

class EventStore(Protocol):
    def read(self, stream: str, after_version: int) -> list[dict]: ...

class SnapshotStore(Protocol):
    def latest(self, stream: str) -> dict | None: ...
    def save(self, stream: str, version: int, schema: int, state: dict) -> None: ...

SNAPSHOT_EVERY = 100
SCHEMA = 2  # bump when Account's shape changes; old snapshots are ignored

def load_account(account_id: str, events: EventStore, snaps: SnapshotStore) -> Account:
    snap = snaps.latest(account_id)
    if snap and snap["schema"] == SCHEMA:
        acc = Account(**snap["state"])
    else:
        acc = Account(id=account_id)               # fall back to full replay
    for e in events.read(account_id, after_version=acc.version):
        acc.apply(e)
    if acc.version - (snap["version"] if snap else 0) >= SNAPSHOT_EVERY:
        snaps.save(account_id, acc.version, SCHEMA, asdict(acc))  # could run async
    return acc
```

## Example — TypeScript
Business snapshot: an invoice copies customer and price data at issue time.

```ts
export interface Customer { id: string; name: string; vatId: string; address: string }
export interface Product { sku: string; name: string; priceCents: number; vatRate: number }

export interface InvoiceLine {
  readonly sku: string; readonly name: string;
  readonly unitPriceCents: number; readonly vatRate: number; readonly qty: number;
}
export interface Invoice {
  readonly id: string; readonly issuedAt: string;
  readonly customerId: string;                    // reference for navigation
  readonly billTo: Readonly<Pick<Customer, 'name' | 'vatId' | 'address'>>; // snapshot
  readonly lines: readonly InvoiceLine[];
}

export function issueInvoice(id: string, c: Customer,
                             items: { product: Product; qty: number }[]): Invoice {
  return Object.freeze({
    id, issuedAt: new Date().toISOString(), customerId: c.id,
    billTo: { name: c.name, vatId: c.vatId, address: c.address },
    lines: items.map(({ product: p, qty }) => ({
      sku: p.sku, name: p.name, unitPriceCents: p.priceCents, vatRate: p.vatRate, qty,
    })),
  });
}
```
Later price or address changes do not alter issued invoices, which is a legal requirement in most jurisdictions.

## In the wild
- EventStoreDB/Kurrent, Marten, Axon and Akka Persistence snapshot support; Kafka Streams state-store changelog compaction.
- PostgreSQL MVCC snapshots (`REPEATABLE READ`, `pg_export_snapshot` for consistent parallel dumps); SQL Server/MariaDB system-versioned temporal tables.
- AWS EBS/RDS snapshots, DynamoDB point-in-time recovery, Git commits, NgRx/Redux state snapshots for time-travel debugging.

## Trade-offs
| Pros | Cons |
|---|---|
| Bounded load time for long event streams | Snapshot schema must be versioned with the model |
| Correct historical documents | Extra storage; data duplicated by design |
| Consistent reads without blocking writers | Another write path that can fail or lag |
| Enables restore points and time travel | Stale snapshot if invalidation/frequency is wrong |

## Pitfalls
- **Snapshot treated as source of truth** in Event Sourcing. Fix: events are the truth; snapshots are disposable and rebuildable.
- **No schema version:** a refactored aggregate cannot read old snapshots. Fix: store a schema number; ignore and rebuild mismatches.
- **Snapshotting synchronously on the hot path** adds latency. Fix: write snapshots asynchronously or in a background job.
- **Snapshotting too early:** complexity without measured benefit. Fix: measure replay time and stream lengths first.
- **Referencing instead of copying** data that must be frozen (prices on orders). Fix: copy at the moment of the fact.

## Refactoring path
- **Introduce it:** measure aggregate load time vs. stream length; add a snapshot table keyed by `(stream, version)`; write snapshots from a background job; change the loader to snapshot + tail replay; keep full replay as fallback.
- **Remove it / simplify:** if streams are short, stop writing snapshots, drop the table and load by full replay; the system stays correct because snapshots were never authoritative.

## Related patterns
- [Event Sourcing](../architectural/event-sourcing.md) — snapshots bound its replay cost.
- [Event Replay](../emergent/event-replay.md) — snapshots are the starting point for partial replay.
- [Memento](../behavioral/memento.md) — in-process snapshot for undo, same idea at object level.
- [CQRS](../cloud/cqrs.md) — read models are continuously maintained snapshots of the write side.
- [Caching](../optimization/caching.md) — a cache is a disposable snapshot with an expiry policy.
