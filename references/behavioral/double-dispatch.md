# Double Dispatch

**Category:** Behavioral · **Also known as:** Multiple dispatch (general form), Multimethods · **Related:** Visitor, Strategy, Interpreter, Composite

## Intent
Select the operation to run based on the runtime types of two objects, not just the receiver, without a chain of `isinstance` checks.

## Problem
- Single dispatch (`obj.method(arg)`) picks the implementation by `obj`'s type only. The argument's type is ignored at runtime, so behaviour for pairs ends up in `isinstance` ladders.
- Smell: `if isinstance(a, Card) and isinstance(b, Refund): ... elif isinstance(a, Card) and isinstance(b, Charge): ...` repeated and growing quadratically.
- Scenario: a payments service applies `Charge`, `Refund` and `Chargeback` operations to `CardAccount`, `WalletAccount` and `InvoiceAccount`. Each pair has its own rules (wallets cannot be charged back, invoices refund by credit note). The rules live in one 200-line function.

## Solution and structure
Classic OO technique: two virtual calls.
1. The client calls `a.interact(b)`. Dispatch #1 picks `a`'s concrete method.
2. Inside it, `a` calls `b.interact_with_<A's type>(self)`. Dispatch #2 picks `b`'s concrete method, which now knows both types.

```
client ─▶ op.apply_to(account)         # dispatch on op type
               └─▶ account.apply_refund(op)  # dispatch on account type
```

In Python and TypeScript, the cleaner modern forms are:
- **A dispatch table** keyed by `(type(a), type(b))`.
- **Structural pattern matching** (`match (a, b): case (Card(), Refund()):`) with exhaustiveness checked by a type checker.
- **`functools.singledispatch`** for one-argument dispatch; nest it or use a library (`plum`, `multipledispatch`) for true multiple dispatch.
- **Discriminated unions** in TS with a `switch` on both `kind` fields.

## When to use
- The behaviour genuinely depends on the combination of two types (collision handling, currency/instrument pairs, shape intersections, permission × resource rules).
- Both type families are reasonably stable and the number of pairs is manageable.
- It is the mechanism behind Visitor (node type × operation type).

## When NOT to use
- Behaviour depends on one type. Normal polymorphism or a single `singledispatch` is enough.
- The pairs share most logic and differ by data. Use a lookup table of parameters (fees per pair) rather than code per pair.
- Types are open on both sides and plugins add new ones; every new type requires N new methods. Use a registry with a sensible default.
- Few pairs and a closed domain: a single `match` statement is clearer than the two-hop OO form.

## Example — Python
Pattern matching on a tuple of types; the language makes the classic two-call trick unnecessary.

```python
from dataclasses import dataclass
from decimal import Decimal

@dataclass(frozen=True)
class CardAccount:   id: str; card_token: str
@dataclass(frozen=True)
class WalletAccount: id: str; balance: Decimal
@dataclass(frozen=True)
class InvoiceAccount: id: str; customer_id: str

@dataclass(frozen=True)
class Charge:     amount: Decimal
@dataclass(frozen=True)
class Refund:     amount: Decimal
@dataclass(frozen=True)
class Chargeback: amount: Decimal; reason: str

type Account = CardAccount | WalletAccount | InvoiceAccount
type Operation = Charge | Refund | Chargeback

class Unsupported(Exception): pass

def apply(op: Operation, acct: Account) -> str:
    match op, acct:
        case Charge(amount=a), CardAccount(card_token=t):
            return f"psp.capture({t}, {a})"
        case Charge(amount=a), WalletAccount(balance=b) if b >= a:
            return f"wallet.debit({acct.id}, {a})"
        case Refund(amount=a), CardAccount() | WalletAccount():
            return f"reverse {a} to {acct.id}"
        case Refund(amount=a), InvoiceAccount(customer_id=c):
            return f"credit_note({c}, {a})"
        case Chargeback(), CardAccount():
            return f"open dispute: {op.reason}"
        case _:
            raise Unsupported(f"{type(op).__name__} on {type(acct).__name__}")
```

Because there is a `case _` fallback, a type checker will not flag a missing pair; drop the wildcard and end with `assert_never` only if every combination must be handled explicitly.

## Example — TypeScript
Dispatch table keyed by both discriminants; the type `Handlers` forces every pair to be filled in.

```typescript
type Shape =
  | { kind: 'circle'; x: number; y: number; r: number }
  | { kind: 'rect'; x: number; y: number; w: number; h: number };

type Kind = Shape['kind'];
type Of<K extends Kind> = Extract<Shape, { kind: K }>;
type Handlers = { [A in Kind]: { [B in Kind]: (a: Of<A>, b: Of<B>) => boolean } };

const overlaps: Handlers = {
  circle: {
    circle: (a, b) => Math.hypot(a.x - b.x, a.y - b.y) <= a.r + b.r,
    rect: (c, r) => circleRect(c, r),
  },
  rect: {
    circle: (r, c) => circleRect(c, r),                   // reuse symmetric case
    rect: (a, b) => a.x < b.x + b.w && b.x < a.x + a.w && a.y < b.y + b.h && b.y < a.y + a.h,
  },
};

function circleRect(c: Of<'circle'>, r: Of<'rect'>): boolean {
  const nx = Math.max(r.x, Math.min(c.x, r.x + r.w));
  const ny = Math.max(r.y, Math.min(c.y, r.y + r.h));
  return Math.hypot(c.x - nx, c.y - ny) <= c.r;
}

export function intersects(a: Shape, b: Shape): boolean {
  const fn = overlaps[a.kind][b.kind] as (x: Shape, y: Shape) => boolean;  // one cast at the boundary
  return fn(a, b);
}
```

Adding a third shape is a compile error until every new pair is defined.

## In the wild
- Python binary operators: `a + b` tries `type(a).__add__`, then `type(b).__radd__` on `NotImplemented`. That is double dispatch built into the language.
- `functools.singledispatch` / `singledispatchmethod` (single argument), `plum-dispatch` and Julia (true multiple dispatch).
- Visitor in compilers: `node.accept(visitor)` → `visitor.visit_BinOp(node)` (`ast.NodeVisitor` uses name lookup instead).
- Game engines' collision matrices (Box2D contact handlers keyed by shape pairs).
- SQLAlchemy dialect compilation (`compiler.visit_<construct>` per dialect).

## Trade-offs
| Pros | Cons |
|---|---|
| Pair-specific logic is explicit and found in one place | N × M combinations grow fast |
| Removes `isinstance` ladders and their ordering bugs | Classic two-hop form is hard to read and debug |
| Typed tables/`match` can enforce exhaustiveness | Adding a type touches every counterpart |
| Symmetric cases can be reused deliberately | Default/fallback behaviour is easy to get wrong |

## Pitfalls
- **Subclass ordering in `isinstance` chains** (a `case Base()` above `case Sub()`). Fix: most specific cases first, or exact-type table keys.
- **Forgetting asymmetry.** `(Card, Refund)` handled, `(Refund, Card)` not, because call sites order arguments differently. Fix: normalise argument order at the entry point.
- **Silent wildcard fallback** hides missing pairs. Fix: raise on the fallback, and test the full matrix with a parametrised test.
- **Using it for data differences.** Fix: a table of parameters, not a table of functions.
- **`NotImplemented` vs `NotImplementedError`** in Python operator overloads. Return `NotImplemented` so the reflected method gets its turn.

## Refactoring path
- **Introduce it:** from an `isinstance` ladder over two arguments. (1) Make the type families explicit (union types / sealed hierarchies). (2) Write a parametrised test over all pairs, recording current behaviour. (3) Replace the ladder with a `match` on the tuple or a typed dispatch table. (4) Remove the wildcard where you want completeness enforced.
- **Remove it / simplify:** if most pairs return the same thing, collapse to single dispatch on the type that matters plus a small exceptions table.

## Related patterns
- [Visitor](visitor.md) — the best-known application of double dispatch: node type × operation.
- [Strategy](strategy.md) — single-axis variation; use it when only one side varies.
- [Interpreter](interpreter.md) — evaluates AST nodes, often via dispatch on node type.
- [Composite](../structural/composite.md) — the tree structure that dispatch-based operations usually walk.
