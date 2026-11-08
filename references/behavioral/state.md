# State

**Category:** Behavioral · **Also known as:** Objects for States, State machine (when table-driven) · **Related:** Strategy, Balking, Command, Memento, Saga

## Intent
Let an object change its behaviour when its internal state changes, by delegating state-specific behaviour to state objects (or a transition table) instead of branching on a status field everywhere.

## Problem
- Methods start with `if self.status == "draft": ... elif self.status == "paid": ...`, and the same branching repeats across `pay()`, `ship()`, `cancel()`, `refund()`.
- Illegal transitions slip through: an order gets shipped before it is paid, or cancelled after delivery, because one branch forgot a check.
- New states (e.g. `partially_refunded`) require editing every method.
- Scenario: an order moves `draft → awaiting_payment → paid → shipped → delivered`, with `cancelled` reachable only before shipping and `refunded` only after payment. Support keeps finding orders in impossible states.

## Solution and structure
Two common shapes:

1. **Transition table** (usually best in Python/TS): an enum of states, a mapping `(state, event) → next_state`, and one `transition()` method that rejects anything not in the table. Side effects hang off transitions.
2. **GoF state objects**: a `State` interface with one method per event; each concrete state implements the events legal in it and returns the next state. The **Context** holds the current state and delegates.

```
            pay            ship           deliver
 AWAITING ───────▶ PAID ───────▶ SHIPPED ───────▶ DELIVERED
    │ cancel        │ cancel / refund
    ▼               ▼
 CANCELLED       REFUNDED
```

Choose state objects when each state has substantial, different behaviour (not just "allowed or not"). Choose the table when behaviour is mostly "is this transition legal, and what is next".

## When to use
- An entity has a lifecycle with rules about which operations are legal in which state (orders, payments, tickets, subscriptions, connections).
- The same `status` switch appears in three or more methods.
- You need to document and test the lifecycle explicitly (the table is the documentation).
- UI flows: wizards, connection status (connecting / online / reconnecting), upload progress.

## When NOT to use
- Two states with one boolean check (`is_active`). Keep the boolean.
- The "state" is really a configuration choice that does not change during the object's life. That is Strategy.
- Long-running, multi-service workflows with timeouts and compensation. Use a workflow engine or Saga (Temporal, Step Functions), not an in-memory state object.
- Do not build a class per state when every state class only says "not allowed". Use the table.

## Example — Python
Table-driven, with the enum as the persisted value.

```python
from dataclasses import dataclass, field
from enum import StrEnum

class OrderStatus(StrEnum):
    AWAITING_PAYMENT = "awaiting_payment"
    PAID = "paid"
    SHIPPED = "shipped"
    DELIVERED = "delivered"
    CANCELLED = "cancelled"
    REFUNDED = "refunded"

class Event(StrEnum):
    PAY = "pay"; SHIP = "ship"; DELIVER = "deliver"; CANCEL = "cancel"; REFUND = "refund"

S, E = OrderStatus, Event
TRANSITIONS: dict[tuple[OrderStatus, Event], OrderStatus] = {
    (S.AWAITING_PAYMENT, E.PAY): S.PAID,
    (S.AWAITING_PAYMENT, E.CANCEL): S.CANCELLED,
    (S.PAID, E.SHIP): S.SHIPPED,
    (S.PAID, E.CANCEL): S.REFUNDED,
    (S.PAID, E.REFUND): S.REFUNDED,
    (S.SHIPPED, E.DELIVER): S.DELIVERED,
}

class IllegalTransition(Exception):
    pass

@dataclass
class Order:
    id: str
    status: OrderStatus = OrderStatus.AWAITING_PAYMENT
    history: list[OrderStatus] = field(default_factory=list)

    def apply(self, event: Event) -> None:
        try:
            nxt = TRANSITIONS[(self.status, event)]
        except KeyError:
            raise IllegalTransition(f"{event} not allowed in {self.status}") from None
        self.history.append(self.status)
        self.status = nxt
```

Persist transitions with an optimistic check (`UPDATE ... WHERE id = :id AND status = :expected`) so two concurrent requests cannot both move from `PAID`.

## Example — TypeScript
State objects for a WebSocket connection, where each state behaves differently; a discriminated union keeps it type-safe.

```typescript
import { signal } from '@angular/core';

type ConnState =
  | { kind: 'idle' }
  | { kind: 'connecting'; attempt: number }
  | { kind: 'open'; socket: WebSocket }
  | { kind: 'backoff'; attempt: number; retryAt: number };

export class LiveFeed {
  readonly state = signal<ConnState>({ kind: 'idle' });

  connect(url: string): void {
    const s = this.state();
    if (s.kind === 'open' || s.kind === 'connecting') return;         // balk
    const attempt = s.kind === 'backoff' ? s.attempt + 1 : 1;
    this.state.set({ kind: 'connecting', attempt });
    const socket = new WebSocket(url);
    socket.onopen = () => this.state.set({ kind: 'open', socket });
    socket.onclose = () => {
      const delay = Math.min(30_000, 2 ** attempt * 500);
      this.state.set({ kind: 'backoff', attempt, retryAt: Date.now() + delay });
      setTimeout(() => this.connect(url), delay);
    };
  }

  send(msg: string): void {
    const s = this.state();
    switch (s.kind) {
      case 'open': s.socket.send(msg); return;                        // narrowed: socket exists
      case 'idle': case 'connecting': case 'backoff':
        throw new Error(`cannot send while ${s.kind}`);
      default: { const never: never = s; return never; }              // exhaustiveness check
    }
  }
}
```

## In the wild
- `python-statemachine`, `transitions` (Python), XState (TS/JS), Spring State Machine.
- AWS Step Functions, Temporal workflows (state machines with durability).
- TCP connection states in the OS, `asyncio.Future` (pending / cancelled / finished), `Promise` states.
- Angular `HttpEventType` progress events, Angular Router navigation lifecycle events.
- Stripe `PaymentIntent.status`, GitHub PR states (domain examples of explicit lifecycles).

## Trade-offs
| Pros | Cons |
|---|---|
| Illegal transitions are rejected in one place | More upfront structure than an `if` |
| The lifecycle is visible and testable (the table is the spec) | State objects scatter behaviour over many small classes |
| Adding a state touches the table, not every method | Shared data between states lives in the context and can get messy |
| Discriminated unions make invalid data unrepresentable in TS | Persisting and migrating state values needs care |

## Pitfalls
- **Status as a free string column.** Typos create ghost states. Fix: enum in code plus a DB check constraint or enum type.
- **Setting `order.status = ...` directly** outside the machine, bypassing rules. Fix: make status private/read-only and expose only `apply(event)`.
- **Race conditions on transitions.** Fix: optimistic locking (version column or `WHERE status = expected`) or row lock.
- **Side effects inside the transition table logic** (sending email before commit). Fix: emit a domain event after commit; let handlers react.
- **Booleans that encode state** (`is_paid`, `is_shipped`, `is_cancelled`) allow impossible combinations. Fix: one enum.
- **Missing exhaustiveness checks** when adding a state. Fix: `never` check in TS, `match` with `assert_never` in Python.

## Refactoring path
- **Introduce it:** from scattered `if status ==` checks. (1) Replace string/booleans with an enum. (2) Write the transition table from current behaviour, with tests per legal and illegal transition. (3) Route all status changes through `apply(event)`. (4) Only if states have rich, different behaviour, move it into per-state objects.
- **Remove it / simplify:** if the machine has two states and one transition, replace it with a boolean and a guard. If you built per-state classes that only allow or deny, collapse them into a table.

## Related patterns
- [Strategy](strategy.md) — same delegation structure; Strategy is chosen from outside, State changes itself.
- [Balking](../concurrency/balking.md) — ignore a call when the object is in the wrong state (`connect()` while connecting).
- [Saga](../cloud/saga.md) — a distributed, durable state machine with compensations.
- [Memento](memento.md) — snapshot the state to roll back a failed transition.
- [Command](command.md) — events that drive the machine are often commands.
- [Event Sourcing](../architectural/event-sourcing.md) — state as a fold over the transition events.
