# Optional

**Category:** Emergent / modern · **Also known as:** Maybe, Option, nullable type, `T | None` · **Related:** Null Object, Value Object, Specification, DTO

## Intent
Make the possible absence of a value explicit in the type, so the type checker forces every caller to handle the "nothing" case, instead of absence leaking through as `None`/`null`/`undefined` and crashing far from its cause.

## Problem
- `AttributeError: 'NoneType' object has no attribute 'email'` / `TypeError: Cannot read properties of undefined` in production, from a lookup that returned nothing.
- Scenario: `get_user(id)` returns `None` when not found, but its signature says `-> User`. A billing job calls `get_user(id).email` and crashes at 3 a.m. for a deleted user.
- Scenario: an Angular template shows `{{ order.customer.address.city }}`; the API omits `address` for digital orders and the whole view errors.
- Smells: defensive `if x is not None` everywhere *or* nowhere; magic sentinel values (`-1`, `""`, `0`) meaning "missing"; docstrings saying "may return None" that the type doesn't.

## Solution and structure
- **Declare** absence in the signature: `def find(id) -> User | None`, `find(id): User | undefined`.
- **Enable** strict checking: mypy/pyright strict (Optional checking is on by default), TS `strict` (`strictNullChecks`). Now unguarded access is a compile error.
- **Narrow** once, close to the source: `if user is None: raise NotFound(...)`, early return, or a default. After narrowing, the rest of the function works with `User`.
- **Propagate safely** where absence is normal: `?.` and `??` in TS; in Python, explicit conditionals or `getattr(x, "a", None)` (Python has no `?.`).
- **Container `Maybe` types** (`Option` with `map`/`flat_map`) are the functional variant; in Python and TS, union-with-None plus the type checker gives most of the benefit with no library.

Distinguish three things that look alike: *absent* (`None`), *empty* (`[]`, `""`), and *unknown/not yet loaded* (often better as a state union: `{ status: 'loading' } | { status: 'ok', data }`).

## When to use
- Lookups that legitimately find nothing (`find_by_email`, cache `get`, `dict.get`).
- Optional fields in models and API schemas (middle name, `deleted_at`, PATCH bodies).
- Configuration values with no sensible default.

## When NOT to use
- Errors: "not found because the DB was down" is not absence. Raise an exception or return a Result/discriminated union carrying the error.
- When a safe neutral object exists (empty list, no-op logger, guest user): return it — that is the Null Object pattern, and callers need no checks.
- When absence is impossible by contract: don't widen the type "just in case"; that pushes pointless checks onto every caller. Validate at the boundary and keep internal types non-optional.
- Collections: return `[]`, not `None`.

## Example — Python
`T | None` with narrowing at the edge; FastAPI turns absence into a 404 once, and the domain code works with non-optional types.

```python
from dataclasses import dataclass
from fastapi import APIRouter, Depends, HTTPException

@dataclass(frozen=True)
class User:
    id: int
    email: str
    display_name: str | None = None        # genuinely optional field

class UserRepo:
    def __init__(self, rows: dict[int, User]) -> None:
        self._rows = rows
    def find(self, user_id: int) -> User | None:      # absence is in the type
        return self._rows.get(user_id)

def greeting(user: User) -> str:                       # takes a narrowed, non-optional User
    return f"Hi, {user.display_name or user.email}"

router = APIRouter()

def get_repo() -> UserRepo: ...

@router.get("/users/{user_id}/greeting")
def read_greeting(user_id: int, repo: UserRepo = Depends(get_repo)) -> dict[str, str]:
    user = repo.find(user_id)
    if user is None:                                    # narrow once, at the boundary
        raise HTTPException(status_code=404, detail="user not found")
    return {"message": greeting(user)}                  # pyright knows user: User here

# Without the check, pyright/mypy reports:
#   Argument of type "User | None" cannot be assigned to parameter "user" of type "User"
```
`display_name or user.email` treats `""` as missing too; use `if user.display_name is not None` when empty strings are meaningful.

## Example — TypeScript
Strict null checks, `?.` / `??`, and Angular's `@if (x; as y)` narrowing in templates.

```ts
import { Component, computed, input } from '@angular/core';

interface Address { city: string; zip: string }
interface Order { id: string; customer: { name: string; address?: Address }; discount?: number | null }

@Component({
  selector: 'app-order-card',
  template: `
    <h3>{{ order().customer.name }}</h3>
    @if (order().customer.address; as addr) {
      <p>{{ addr.city }}, {{ addr.zip }}</p>          <!-- addr: Address, narrowed -->
    } @else {
      <p>Digital delivery</p>
    }
    <p>Discount: {{ discountPct() }}%</p>`,
})
export class OrderCard {
  order = input.required<Order>();
  // ?? keeps 0 as a valid value; || would turn 0 into the default
  discountPct = computed(() => (this.order().discount ?? 0) * 100);
}

export function cityOf(o: Order): string | undefined {
  return o.customer.address?.city;                 // optional chaining short-circuits
}
```
Avoid the non-null assertion `!` except where an invariant is proven; each `!` is an unchecked claim.

## In the wild
- Python: `typing.Optional[T]` ≡ `T | None` (3.10+ syntax), `dict.get`, `re.match` returning `Match | None`, Pydantic optional fields, SQLAlchemy `Mapped[str | None]` making the column nullable.
- TS/JS: `strictNullChecks`, `?.`, `??`, `??=`, `Array.prototype.find` returning `T | undefined`, `noUncheckedIndexedAccess` for index lookups, Angular `@if (...; as x)`, RxJS `filter((x): x is T => x != null)`.
- Elsewhere: Rust `Option`, Kotlin `T?`, Swift optionals, Java `Optional<T>` (return values only), Haskell `Maybe`.

## Trade-offs
| Pros | Cons |
|---|---|
| Absence handled at compile time, not in production | More narrowing code at boundaries |
| Signatures document behavior honestly | Over-use spreads `| None` checks everywhere |
| Zero runtime cost with union types | Doesn't carry *why* something is absent |
| Works with existing tools (pyright, tsc) | Gradual codebases with `Any`/`any` bypass the checks |

## Pitfalls
- Truthiness checks that mistreat valid falsy values: `if not count` rejects `0`; `x || default` replaces `0` and `""`. Use `is None` / `??`.
- `Optional` used to signal failure: callers can't distinguish "not found" from "error". Use exceptions or a `Result` union (`{ ok: true, value } | { ok: false, error }`).
- Optional fields in internal domain types because the API made them optional — validate and narrow at the boundary.
- In Pydantic, `x: str | None` without a default is *required but nullable*; write `x: str | None = None` for truly optional.
- TS `undefined` vs `null` mixed in one model; pick one convention (often `undefined` for absent properties, `null` from JSON APIs) and normalize at the boundary.
- `!` assertions and `cast` silencing the checker instead of handling absence.

## Refactoring path
- **Introduce it:** turn on strict null checking (pyright strict or mypy, TS `strict`); fix signatures that lie (`-> User` that returns `None`); add narrowing at the boundary; replace sentinel values with `None`; enable `noUncheckedIndexedAccess` for extra safety.
- **Remove it / simplify:** where most callers immediately supply the same default, return a Null Object or the default from the source; where absence is impossible after validation, narrow the type and delete downstream checks.

## Related patterns
- [Null Object](../behavioral/null-object.md) — replace absence with a do-nothing implementation so callers need no checks.
- [Value Object](../emergent/value-object.md) — validated values remove "maybe invalid" states; optional removes "maybe missing" ones.
- [Specification](../behavioral/specification.md) — composable predicates often used to filter `T | None` streams.
- [DTO](../enterprise/dto.md) — where optional API fields belong, before narrowing into domain types.
