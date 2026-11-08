# Marker Interface

**Category:** Structural · **Also known as:** Tag interface, Marker type, Branded type (TS) · **Related:** Metadata Mapping, Extension Object, Specification, Value Object, Strategy

## Intent
Tag a type with a capability or property by having it implement an interface with no methods, so code (or a type checker) can test for that tag and treat the type differently.

## Problem
- Some objects need special handling based on a *property of their type*, not their data: "this event may be retried", "this command is idempotent", "this DTO contains PII and must be masked in logs".
- Ad-hoc checks spread through the code: `if type(cmd).__name__ in {"CreateOrder", "ChargeCard"}` or a hard-coded list of class names.
- You want the type checker, not a runtime string list, to prevent mistakes like passing a raw `str` where a validated `UserId` is required.

Scenario: a message bus must deduplicate only commands that are safe to deduplicate. Marking them `Idempotent` lets the bus check `isinstance(cmd, Idempotent)`.

## Solution and structure
- **Marker**: an empty interface/base class (Python: an empty ABC or class; TS: a brand property).
- **Marked types**: inherit from or implement the marker.
- **Consumers**: check for the marker (`isinstance`, type narrowing, overloads) and branch.

Two flavors:
1. **Runtime marker** — checked with `isinstance`/`instanceof`. Python and classes in TS.
2. **Compile-time marker (branded/nominal type)** — exists only for the type checker: Python `typing.NewType`, TS branded types (`string & { __brand: 'UserId' }`). Zero runtime cost.

Modern alternative: **metadata** (Python class attributes or decorators, TS/Angular decorators, Java annotations). Metadata can carry parameters (`@retryable(max=3)`); a marker can only say yes/no.

## When to use
- A yes/no capability that is truly a property of the *type* and stable over its lifetime.
- You want the type system to enforce it (only `Idempotent` commands may be passed to `dedupe()`).
- Preventing primitive obsession at zero runtime cost (branded IDs, validated strings).

## When NOT to use
- The tag needs parameters or varies per instance: use a field, an enum, or metadata.
- Behavior differs by tag: put a method on the interface (polymorphism) instead of `isinstance` switches elsewhere.
- TypeScript structural typing: an empty `interface Serializable {}` matches *every* object, so it marks nothing; use a brand or a class with `instanceof`.
- Python `typing.Protocol` with no members: every object satisfies it structurally; use an ABC or class for a real marker.

## Example — Python
Runtime marker plus compile-time `NewType` brand.

```python
from dataclasses import dataclass
from typing import NewType


class Idempotent:
    """Marker: handling this command twice has the same effect as once."""
    __slots__ = ()


class ContainsPII:
    """Marker: never log field values of this type."""
    __slots__ = ()


UserId = NewType("UserId", int)          # compile-time marker, erased at runtime


@dataclass(frozen=True)
class SetEmail(Idempotent, ContainsPII):
    user_id: UserId
    email: str


@dataclass(frozen=True)
class ChargeCard:                        # not idempotent: no marker
    user_id: UserId
    amount_cents: int


def log_command(cmd: object) -> str:
    if isinstance(cmd, ContainsPII):
        return f"{type(cmd).__name__}(<redacted>)"
    return repr(cmd)


def dispatch(cmd: object, seen: set[object]) -> None:
    if isinstance(cmd, Idempotent) and cmd in seen:
        return                           # safe to drop duplicates
    seen.add(cmd)
    print("handling", log_command(cmd))


dispatch(SetEmail(UserId(7), "a@b.com"), set())
# dispatch(ChargeCard(7, 100), set())   # mypy/pyright: int is not UserId
```

## Example — TypeScript
Branded types are the idiomatic TS marker, since interfaces are structural.

```ts
declare const brand: unique symbol;
type Brand<T, B extends string> = T & { readonly [brand]: B };

export type UserId = Brand<string, 'UserId'>;
export type OrderId = Brand<string, 'OrderId'>;
export type SafeHtml = Brand<string, 'SafeHtml'>;

export const UserId = (raw: string): UserId => {
  if (!/^usr_[a-z0-9]{8}$/.test(raw)) throw new Error(`invalid user id: ${raw}`);
  return raw as UserId;                       // the only place a cast is allowed
};

export function loadOrders(user: UserId): Promise<OrderId[]> {
  return fetch(`/api/users/${user}/orders`).then((r) => r.json());
}

loadOrders(UserId('usr_ab12cd34'));            // ok
// loadOrders('usr_ab12cd34');                 // error: string is not UserId
// loadOrders(someOrderId);                    // error: OrderId is not UserId

// Runtime marker needs a class (structural typing makes empty interfaces useless):
export abstract class Retryable {}
export class SyncInventory extends Retryable { constructor(readonly sku: string) { super(); } }
const shouldRetry = (job: object) => job instanceof Retryable;
```
Angular's `SafeHtml`/`SafeUrl` from `DomSanitizer` work in a similar spirit: a value type that marks "already sanitized".

## In the wild
- Python: `typing.NewType`, `collections.abc.Hashable` (an ABC checked via `__hash__`), `typing.Final`/`Annotated` metadata, Pydantic `SecretStr` (a wrapper type marking secrets).
- Java: `Serializable`, `Cloneable`, `RandomAccess` — the textbook markers, now often replaced by annotations.
- TS: branded types in Zod (`.brand<'UserId'>()`), Effect, io-ts.
- Angular: `SafeValue` types from `DomSanitizer`; NestJS uses decorators/metadata instead of markers.

## Trade-offs
| Pros | Cons |
|---|---|
| Type-level, explicit, discoverable in IDE | Only a boolean; cannot carry parameters |
| Compile-time brands cost nothing at runtime | Consumers must remember to check it |
| Prevents mixing IDs and primitive strings | Inherited by subclasses even when not appropriate |
| Simple: no framework needed | Encourages `isinstance` switches instead of polymorphism |

## Pitfalls
- **Empty TS interface or empty Python Protocol as a marker**: structural typing makes everything match. Fix: brands, classes or ABCs.
- **Unchecked casts everywhere** (`raw as UserId`). Fix: one validating constructor per brand; lint against other casts.
- **Marker drives growing `if isinstance` ladders**. Fix: move the varying behavior into a method or Strategy.
- **Subclass inherits a marker it should not** (a subclass of an idempotent command adds side effects). Fix: mark leaf classes, or use metadata that is not inherited.

## Refactoring path
- **Introduce it:** replace hard-coded type-name lists with a marker; for IDs, introduce `NewType`/brand at the boundary (parsing) and let the type checker find every call site.
- **Remove it / simplify:** if the marker needs options, replace with a class attribute or decorator metadata (`retry_policy = RetryPolicy(3)`); if consumers branch on it heavily, replace with a polymorphic method.

## Related patterns
- [Metadata Mapping](../emergent/metadata-mapping.md) — annotations/decorators carrying parameters; the modern replacement.
- [Extension Object](../structural/extension-object.md) — capabilities added at runtime rather than declared on the type.
- [Value Object](../emergent/value-object.md) — a richer alternative to a branded primitive when behavior is needed.
- [Specification](../behavioral/specification.md) — rule-based selection when the criterion depends on data, not type.
- [Strategy](../behavioral/strategy.md) — replace marker-driven branching with polymorphism.
