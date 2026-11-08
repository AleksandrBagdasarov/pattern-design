# Event Sourcing

**Category:** Architectural · **Also known as:** ES · **Related:** CQRS, Snapshot, Event Replay, Event-Driven, Memento, Domain Model

## Intent
Store the state of an entity as an **append-only sequence of domain events** instead of overwriting its current row. Current state is derived by replaying (folding) the events; read models are built from the same stream as projections.

## Problem
- You must answer "how did this balance get to −120?" and "what did the account look like last Tuesday?"; an `UPDATE` destroys history.
- Audit logs are written separately from state and drift from it.
- Several read shapes (statement, monthly totals, fraud features) are needed from the same writes.
- Scenario: a wallet/ledger service under financial regulation. Every deposit, withdrawal, hold and reversal must be traceable and replayable; a bug in fee calculation must be corrected by rebuilding balances.

## Solution and structure
```
 command ─► Aggregate.load(events) ─► decide ─► new events ─► EventStore.append(stream, expected_version)
                                                                    │
                                     projector ◄────── subscribe ───┘ ─► read models (SQL, Elastic, cache)
```
- **Event:** immutable past-tense fact (`FundsDeposited{amount}`), versioned schema.
- **Stream:** events of one aggregate instance (`wallet-123`), ordered by version.
- **Aggregate:** rebuilds state with `apply(event)`, validates commands, emits new events. No I/O.
- **Event store:** append with **optimistic concurrency** (`expected_version`), read stream, subscribe to all.
- **Projections:** consumers that build query-optimized views; can be dropped and rebuilt by replay.
- **Snapshots:** periodic cached state to avoid replaying thousands of events.

## When to use
- Audit, compliance and "time travel" are core requirements (finance, healthcare, logistics tracking).
- The domain is naturally a sequence of facts (ledgers, orders with lifecycle, collaborative editing).
- You need multiple, evolving read models from the same history (with CQRS).

## When NOT to use
- CRUD domains with no use for history: a normal table plus an `updated_at`/audit table is far simpler.
- When you only need an audit log: add an append-only audit table or CDC to a regular schema.
- Data that must be truly deleted (GDPR erasure) without a crypto-shredding plan.
- Teams without experience in eventual consistency and schema versioning on a deadline.

## Example — Python
```python
from dataclasses import dataclass
from decimal import Decimal
from typing import Protocol

@dataclass(frozen=True)
class FundsDeposited:  amount: Decimal
@dataclass(frozen=True)
class FundsWithdrawn:  amount: Decimal
Event = FundsDeposited | FundsWithdrawn

class ConcurrencyError(Exception): ...

class EventStore(Protocol):
    def load(self, stream: str) -> list[Event]: ...
    def append(self, stream: str, events: list[Event], expected_version: int) -> None: ...

@dataclass
class Wallet:
    balance: Decimal = Decimal(0)
    version: int = 0

    @classmethod
    def from_events(cls, events: list[Event]) -> "Wallet":
        w = cls()
        for e in events:
            w.apply(e)
        return w

    def apply(self, e: Event) -> None:
        match e:
            case FundsDeposited(amount): self.balance += amount
            case FundsWithdrawn(amount): self.balance -= amount
        self.version += 1

    def withdraw(self, amount: Decimal) -> list[Event]:       # decide: pure, returns new events
        if amount > self.balance:
            raise ValueError("insufficient funds")
        return [FundsWithdrawn(amount)]

def withdraw(store: EventStore, wallet_id: str, amount: Decimal) -> None:
    stream = f"wallet-{wallet_id}"
    wallet = Wallet.from_events(store.load(stream))
    new = wallet.withdraw(amount)
    store.append(stream, new, expected_version=wallet.version)  # raises ConcurrencyError on race
```
`match` on frozen dataclasses gives a readable fold; the decision logic is a pure function you can test with "given events, when command, then events".

## Example — TypeScript
```ts
type WalletEvent =
  | { type: 'FundsDeposited'; amount: number; at: string }
  | { type: 'FundsWithdrawn'; amount: number; at: string };

interface WalletState { balance: number; version: number }

export const evolve = (s: WalletState, e: WalletEvent): WalletState => {
  switch (e.type) {
    case 'FundsDeposited': return { balance: s.balance + e.amount, version: s.version + 1 };
    case 'FundsWithdrawn': return { balance: s.balance - e.amount, version: s.version + 1 };
  }
};
export const rehydrate = (events: WalletEvent[]) => events.reduce(evolve, { balance: 0, version: 0 });

export function decideWithdraw(s: WalletState, amount: number): WalletEvent[] {
  if (amount > s.balance) throw new Error('insufficient funds');
  return [{ type: 'FundsWithdrawn', amount, at: new Date().toISOString() }];
}

// Projection: monthly totals read model, rebuilt from scratch by replaying all events
export function projectMonthly(events: WalletEvent[]): Map<string, number> {
  const m = new Map<string, number>();
  for (const e of events) {
    const key = e.at.slice(0, 7);
    const delta = e.type === 'FundsDeposited' ? e.amount : -e.amount;
    m.set(key, (m.get(key) ?? 0) + delta);
  }
  return m;
}
```
A frontend reducer (NgRx, Redux) has the same `(state, event) => state` shape, but only in memory; it is not event sourcing unless the events are the persisted source of truth.

## In the wild
- EventStoreDB (Kurrent), Marten (Postgres), Axon (Java), `eventsourcing` (Python library), Kafka as a log (needs care: no per-stream optimistic concurrency).
- Git (commits are events), bank ledgers, database WALs.
- Plain Postgres works: `events(stream_id, version, type, data jsonb, UNIQUE(stream_id, version))`.

## Trade-offs
| Pros | Cons |
|---|---|
| Complete, trustworthy audit trail and time travel | Significant complexity; unfamiliar to most teams |
| New read models built retroactively from history | Eventual consistency between writes and projections |
| Bugs fixable by correcting projections and replaying | Event schema versioning (upcasting) forever |
| Natural fit for event-driven integration | Querying current state needs projections; ad-hoc SQL is hard |
| Optimistic concurrency per aggregate | Deleting personal data needs crypto-shredding or stream rewriting |

## Pitfalls
- **No optimistic concurrency check:** two writers append conflicting events. Always append with `expected_version`, enforced by a unique constraint.
- **Events as CRUD diffs** (`WalletUpdated{balance}`): you lose intent. Model business facts.
- **Side effects in `apply`:** replay would resend emails. `apply` must be pure; side effects go in projectors/handlers with idempotency.
- **No snapshot plan:** long streams make loads slow. Snapshot every N events; keep snapshots disposable.
- **Changing old event shapes in place:** never edit stored events; add versions and upcasters.
- **Reading projections inside command handlers:** they are stale; validate against the aggregate's own stream.

## Refactoring path
- **Introduce it:** start with one aggregate where history matters. Keep the existing table as a projection; write events alongside (outbox), then make the event stream the source and rebuild the table from it; migrate other aggregates only if they need it.
- **Remove it / simplify:** if nobody uses history or replay, keep the latest projection as the primary table, archive the event store, and write state directly plus an audit table.

## Related patterns
- [CQRS](../cloud/cqrs.md) — almost always paired: events on the write side, projections on the read side.
- [Snapshot](../data-access/snapshot.md) — bounds rehydration time.
- [Event Replay](../emergent/event-replay.md) — rebuilding projections or debugging by replaying.
- [Event-Driven](../architectural/event-driven.md) — ES events can be published to other services (as a public, versioned contract).
- [Memento](../behavioral/memento.md) — stores snapshots of state instead of the facts that produced it.
- [Domain Model](../data-access/domain-model.md) — the aggregate that decides and applies events.
