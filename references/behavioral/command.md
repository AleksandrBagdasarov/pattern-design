# Command

**Category:** Behavioral · **Also known as:** Action, Transaction, Request object · **Related:** Memento, Strategy, Chain of Responsibility, Event Sourcing, Task Queue, CQS, CQRS

## Intent
Turn a request into a standalone object that holds everything needed to perform it, so it can be passed around, queued, logged, retried or undone independently of whoever triggered it.

## Problem
- The trigger (button, HTTP endpoint, CLI, cron) is tightly bound to the code that does the work, so the same operation is duplicated across triggers.
- You need undo/redo, an audit log of "who asked for what", or replay after a crash.
- Work must run later or elsewhere: a background worker, a retry after a failure, a batch.
- Scenario: an admin panel lets support staff refund orders, change addresses and cancel subscriptions. Every action must be audited, some must be undoable, and refunds above 500 EUR must wait for approval.

## Solution and structure
- **Command** holds the parameters of one request and exposes `execute()` (and optionally `undo()`).
- **Receiver** is the domain object or service that actually does the work.
- **Invoker** triggers commands without knowing what they do: a button, a job runner, an undo stack.
- **Client** builds the command and hands it to the invoker.

```
Client ─builds─▶ Command(params) ─execute()─▶ Receiver
                    ▲
Invoker (queue / history / retry loop) ─holds─┘
```

Two flavours exist in practice. **GoF command**: the object carries behaviour (`execute`). **Command message** (CQRS, message buses): a plain data record (`RefundOrder(order_id, amount)`) dispatched to a separate handler. The second is easier to serialize and is the common modern form.

## When to use
- Undo/redo in editors, design tools, form wizards.
- Queueing and deferred execution: job queues, outbox, offline-first sync.
- Audit and replay: persist commands, re-run them, or show a history.
- One operation reachable from several triggers (REST, CLI, scheduled job) and you want a single entry point.
- Macro / batch: compose several commands into one.

## When NOT to use
- A simple call with no queueing, undo or logging needs. A function call is a command already.
- Just to pass behaviour as a parameter. That is a callable (Strategy).
- You need guaranteed delivery across processes. A Command object is not enough; you need a durable Message Queue or Task Queue.
- CRUD endpoints wrapped one-to-one in `CreateXCommand` + handler + bus with no extra behaviour. That is ceremony; call the service.

## Example — Python
Command messages as frozen dataclasses plus handlers; the invoker adds audit and undo.

```python
from dataclasses import dataclass, field
from typing import Protocol

@dataclass(frozen=True)
class ChangeShippingAddress:
    order_id: str
    new_address: str
    requested_by: str

class Orders(Protocol):
    def get_address(self, order_id: str) -> str: ...
    def set_address(self, order_id: str, address: str) -> None: ...

@dataclass
class AddressChange:
    cmd: ChangeShippingAddress
    orders: Orders
    _previous: str | None = None

    def execute(self) -> None:
        self._previous = self.orders.get_address(self.cmd.order_id)
        self.orders.set_address(self.cmd.order_id, self.cmd.new_address)

    def undo(self) -> None:
        assert self._previous is not None, "undo before execute"
        self.orders.set_address(self.cmd.order_id, self._previous)

@dataclass
class CommandHistory:
    done: list[AddressChange] = field(default_factory=list)
    audit: list[ChangeShippingAddress] = field(default_factory=list)

    def run(self, action: AddressChange) -> None:
        action.execute()
        self.done.append(action)
        self.audit.append(action.cmd)          # serializable record of intent

    def undo_last(self) -> None:
        if self.done:
            self.done.pop().undo()
```

## Example — TypeScript
An undo stack for an editor, held in a signal so the UI can bind `canUndo`.

```typescript
import { Injectable, computed, signal } from '@angular/core';

export interface Command {
  readonly label: string;
  execute(): void;
  undo(): void;
}

export class RenameTask implements Command {
  readonly label: string;
  private previous = '';
  constructor(private readonly task: { title: string }, private readonly next: string) {
    this.label = `Rename to "${next}"`;
  }
  execute(): void { this.previous = this.task.title; this.task.title = this.next; }
  undo(): void { this.task.title = this.previous; }
}

@Injectable({ providedIn: 'root' })
export class History {
  private readonly undoStack = signal<Command[]>([]);
  private readonly redoStack = signal<Command[]>([]);
  readonly canUndo = computed(() => this.undoStack().length > 0);
  readonly canRedo = computed(() => this.redoStack().length > 0);

  run(cmd: Command): void {
    cmd.execute();
    this.undoStack.update(s => [...s, cmd]);
    this.redoStack.set([]);                       // a new action invalidates redo
  }
  undo(): void {
    const cmd = this.undoStack().at(-1);
    if (!cmd) return;
    cmd.undo();
    this.undoStack.update(s => s.slice(0, -1));
    this.redoStack.update(s => [...s, cmd]);
  }
}
```

## In the wild
- Celery / RQ / Dramatiq tasks, `concurrent.futures.Executor.submit(fn, *args)`, `asyncio.Queue` of work items.
- NgRx / Redux actions (command-like messages handled by reducers and effects), NestJS `@nestjs/cqrs` `CommandBus`.
- AWS SQS messages and Step Functions tasks, Kafka records used as commands.
- Database migrations (Alembic `upgrade`/`downgrade`), `git` commits, VS Code commands registry.
- `prosemirror` / `Tiptap` transactions, browser `document.execCommand` (deprecated).

## Trade-offs
| Pros | Cons |
|---|---|
| Decouples trigger from execution; one operation, many triggers | One class or record per operation adds files and indirection |
| Enables undo, redo, queueing, retry, audit, replay | Undo is hard when effects are external (emails, payments) |
| Command records serialize well for logs and queues | Command bus hides call targets, making navigation harder |
| Macro commands compose operations | Stale commands: replaying old parameters on new state can be wrong |

## Pitfalls
- **Undo that does not restore everything.** Fix: capture the previous state inside `execute`, or use a Memento; test `execute` then `undo` returns to the original snapshot.
- **Non-undoable side effects.** You cannot "unsend" an email. Fix: model a compensating command (refund, correction email) instead of `undo`, as in a Saga.
- **Retrying non-idempotent commands.** Fix: give every command an id and make the handler idempotent (Idempotent Receiver).
- **Commands that hold live objects** (DB sessions, ORM entities) can't be queued or serialized. Fix: store ids and plain values, and load in the handler.
- **Forgetting to clear redo** after a new command. Fix: as in the example.
- **Bus with one handler per command and nothing else.** Fix: call the service directly until you need middleware (auth, logging, transactions) on the bus.

## Refactoring path
- **Introduce it:** from duplicated operation logic across controllers, or from a request for undo/audit. (1) Create a frozen record of the operation's inputs. (2) Move the logic into a handler taking that record. (3) Make each trigger build the record and call the handler. (4) Add the invoker features you need (history, queue, audit) in one place.
- **Remove it / simplify:** if no command is ever queued, logged or undone, replace `bus.dispatch(DoX(...))` with a direct service call and delete the bus and handler registry.

## Related patterns
- [Memento](memento.md) — stores the state a command needs to undo itself.
- [Strategy](strategy.md) — also wraps behaviour, but describes *how* to do something, not a request to do it.
- [Event Sourcing](../architectural/event-sourcing.md) — persists the resulting events (facts), whereas commands are intentions that may be rejected.
- [Task Queue](../emergent/task-queue.md) — the infrastructure that runs commands asynchronously.
- [CQRS](../cloud/cqrs.md) — separates command handling from queries at the architecture level.
- [CQS (Command-Query Separation)](../ui/cqs.md) — method-level rule: commands change state and return nothing.
- [Chain of Responsibility](chain-of-responsibility.md) — middleware around a command bus (auth, validation, transactions).
