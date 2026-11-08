# Memento

**Category:** Behavioral · **Also known as:** Snapshot, Token · **Related:** Command, Snapshot, Prototype, Event Sourcing, State, Private Class Data

## Intent
Capture an object's internal state as an opaque snapshot so it can be restored later, without exposing the object's internals to whoever stores the snapshot.

## Problem
- You need undo, "cancel changes", rollback after a failed multi-step operation, or a restore point, and the state to save includes private fields.
- Smell: an editor or service makes all fields public just so an undo manager can copy them, or the undo manager copies fields one by one and misses a new one.
- Smell: "discard changes" in a form re-fetches from the server because nobody kept the original.
- Scenario: a pricing admin edits a rule set with 40 rules, tries a simulation, and wants to revert to the state before the experiment. The rule set has derived caches that must not be saved.

## Solution and structure
- **Originator**: the object whose state is saved. It creates a memento (`save()`) and restores from one (`restore(m)`). Only it knows what goes inside.
- **Memento**: an immutable value holding the snapshot. Opaque to everyone except the originator.
- **Caretaker**: keeps mementos (undo stack, checkpoint list) and decides when to save or restore, without reading them.

```
Caretaker ──save()──▶ Originator ──creates──▶ Memento (frozen)
    │  stack of mementos                         ▲
    └──restore(m)──▶ Originator ◀──reads─────────┘
```

In Python and TypeScript, "opaque" is enforced by convention (a frozen dataclass or `Readonly` type the caretaker never inspects). With **immutable state** (frozen dataclasses, signals holding immutable objects, Redux stores), the current value *is* the memento: keep the old reference and you can restore it for free.

## When to use
- Undo/redo and "discard changes" in editors, forms, drawing tools.
- Checkpoint/rollback around a risky operation on an in-memory model (simulation, what-if analysis).
- Saving and resuming progress (game saves, wizard drafts, long computation checkpoints).
- Optimistic UI: snapshot before an optimistic update, restore on server error.

## When NOT to use
- State is large and changes are small. Full snapshots waste memory; use Command with inverse operations, or structural sharing.
- Persistent data in a database. Use a transaction or savepoint, not an in-memory copy.
- You need a full history for audit or temporal queries. Use Event Sourcing.
- The object is already immutable. Just keep references; no extra pattern is needed.

## Example — Python
A frozen memento with only the persistent part of the state; derived caches are rebuilt on restore.

```python
from copy import deepcopy
from dataclasses import dataclass, field
from decimal import Decimal

@dataclass(frozen=True)
class RuleSetMemento:
    _rules: tuple[tuple[str, Decimal], ...]   # leading underscore: caretaker must not read
    _version: int

class PricingRuleSet:
    def __init__(self) -> None:
        self._rules: dict[str, Decimal] = {}
        self._version = 0
        self._cache: dict[str, Decimal] = {}  # derived, not part of the snapshot

    def set_rule(self, sku: str, discount: Decimal) -> None:
        self._rules[sku] = discount
        self._version += 1
        self._cache.clear()

    def save(self) -> RuleSetMemento:
        return RuleSetMemento(tuple(sorted(self._rules.items())), self._version)

    def restore(self, m: RuleSetMemento) -> None:
        self._rules = dict(m._rules)
        self._version = m._version
        self._cache.clear()

@dataclass
class Checkpoints:                         # caretaker
    _stack: list[RuleSetMemento] = field(default_factory=list)
    def push(self, m: RuleSetMemento) -> None: self._stack.append(m)
    def pop(self) -> RuleSetMemento: return self._stack.pop()

rules, history = PricingRuleSet(), Checkpoints()
rules.set_rule("SKU-1", Decimal("0.10"))
history.push(rules.save())
rules.set_rule("SKU-1", Decimal("0.50"))   # experiment
rules.restore(history.pop())               # back to 10%
```

Use `deepcopy` only when the state contains nested mutable objects you cannot convert to immutable tuples; it is slow and copies things you may not want (locks, connections).

## Example — TypeScript
Optimistic update in an Angular store: the previous immutable value is the memento.

```typescript
import { Injectable, inject, signal } from '@angular/core';
import { HttpClient } from '@angular/common/http';

export interface Task { id: string; title: string; done: boolean }

@Injectable({ providedIn: 'root' })
export class TaskStore {
  private readonly http = inject(HttpClient);
  readonly tasks = signal<readonly Task[]>([]);
  readonly error = signal<string | null>(null);

  toggle(id: string): void {
    const snapshot = this.tasks();                              // memento: an immutable reference
    this.tasks.update(list => list.map(t => (t.id === id ? { ...t, done: !t.done } : t)));
    const updated = this.tasks().find(t => t.id === id);
    if (!updated) return;

    this.http.patch<Task>(`/api/tasks/${id}`, { done: updated.done }).subscribe({
      error: () => {
        this.tasks.set(snapshot);                               // restore
        this.error.set('Could not save, change reverted');
      },
    });
  }
}
```

Restoring a whole-list snapshot can overwrite other changes made meanwhile; for concurrent edits, revert only the affected item, or refetch.

## In the wild
- Immutable state stores (Redux, NgRx, Angular signals with immutable values); Redux DevTools time travel.
- `copy.copy` / `copy.deepcopy`, `__getstate__` / `__setstate__` (pickle protocol), `structuredClone` in JS.
- Database savepoints (`SAVEPOINT`, SQLAlchemy `session.begin_nested()`), which are the persistent analogue.
- Editors: ProseMirror/Tiptap history, browser `history.pushState` state objects, VS Code hot exit backups.
- ML training checkpoints (`torch.save(model.state_dict())`), Flink/Spark checkpoints.

## Trade-offs
| Pros | Cons |
|---|---|
| Undo/rollback without exposing internals | Memory cost grows with state size × number of snapshots |
| Originator controls exactly what is captured | Deep copies are slow and may copy unshareable resources |
| Trivial with immutable state | Restoring stale snapshots can discard concurrent changes |
| Caretaker logic stays generic | Snapshot format couples to the originator's internals (versioning when persisted) |

## Pitfalls
- **Shallow copies sharing mutable data.** The memento changes when the original does. Fix: store immutable structures (tuples, frozen dataclasses, `readonly` copies) or copy deeply at the boundary.
- **Unbounded history.** Fix: cap the stack (`collections.deque(maxlen=50)`), or coalesce small edits.
- **Snapshotting derived or external state** (caches, open connections, timers). Fix: capture only the source of truth; rebuild the rest on restore.
- **Persisted mementos without a version.** An old save cannot be loaded after a refactor. Fix: include a schema version and a migration.
- **Restoring the whole state on an optimistic error** while other updates happened. Fix: revert the specific change, or reconcile with the server.

## Refactoring path
- **Introduce it:** from an undo manager that reads public fields, or a "cancel" that refetches. (1) Add `save()` returning a frozen value with the essential state. (2) Add `restore()`. (3) Make the caretaker use only these two calls. (4) Make the fields private again.
- **Remove it / simplify:** if the state is already immutable, delete memento classes and keep previous references. If snapshots are too large, switch to Command with inverse operations or Event Sourcing.

## Related patterns
- [Command](command.md) — undo via inverse operations; often combined, with the command holding a memento.
- [Snapshot](../data-access/snapshot.md) — the persistence-level point-in-time copy (event stores, versioned reads).
- [Prototype](../creational/prototype.md) — also copies objects, but to create new ones, not to restore.
- [Event Sourcing](../architectural/event-sourcing.md) — keeps every change instead of selected snapshots.
- [State](state.md) — snapshot before a transition to roll back on failure.
- [Private Class Data](../security/private-class-data.md) — protects the internals the memento must not expose.
