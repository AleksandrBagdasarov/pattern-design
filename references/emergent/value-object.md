# Value Object

**Category:** Emergent / modern · **Also known as:** Value type, domain primitive, tiny type · **Related:** DTO, Domain Model, Flyweight, Specification, Private Class Data

## Intent
Model a domain concept that is defined entirely by its attributes (Money, Email, DateRange, Address) as a small immutable type with value-based equality and built-in validation, so invalid values cannot exist and the concept's rules live in one place.

## Problem
- Primitive obsession: amounts as `float`, currencies as `str`, emails as `str`, date ranges as two loose `date`s.
- Scenario: `total = order.amount + refund.amount` silently adds EUR to USD. Elsewhere `0.1 + 0.2` produces `0.30000000000000004` on an invoice. Email validation is copy-pasted in five handlers, two of which lower-case it.
- Scenario: `transfer(from_id, to_id, amount)` is called with arguments swapped — both are `int`, the type checker cannot help.
- Smells: the same validation/normalization repeated; functions taking 4+ primitives; bugs from mutated shared objects (`start_date` changed through an alias).

## Solution and structure
- **Immutable**: no setters; "changes" return new instances (`money.add(other)`).
- **Equality by value**: two `Money(10, "EUR")` are equal and hash the same; identity doesn't matter (contrast with an Entity, which has an id and a lifecycle).
- **Self-validating**: the constructor/factory rejects invalid states; once you hold one, it is valid.
- **Behavior-rich**: operations that belong to the concept (`Money.allocate`, `DateRange.overlaps`, `Email.domain`).
- **Side-effect free**: methods are pure.

Entities hold value objects: `Order(id, total: Money, ship_to: Address)`. Persist them as embedded columns (SQLAlchemy `composite`, JSON column) rather than separate tables with ids.

## When to use
- Concepts with rules: money and currency, quantities with units, emails, phone numbers, IBANs, date ranges, coordinates, percentages.
- Values that are passed around a lot and whose invalid forms cause bugs.
- Distinguishing same-shaped primitives (`UserId` vs `OrderId`) at type-check time.
- Keys in dicts/sets and cache keys (immutability + hashing).

## When NOT to use
- Plain data with no rules that just crosses a boundary: a DTO/Pydantic model or a `TypedDict` is enough.
- Things with identity and lifecycle (a Customer whose email changes is still the same customer): use an Entity.
- Ultra-hot numeric loops over millions of items: wrapper objects cost allocations; use arrays (NumPy, typed arrays) and wrap at the edges.
- Don't wrap every string "just in case". Start with concepts that have actual invariants or have caused bugs.

## Example — Python
`@dataclass(frozen=True, slots=True)` gives immutability, `__eq__` and `__hash__`; `Decimal` for exact money. For IDs, `typing.NewType` is a zero-cost alternative.

```python
from dataclasses import dataclass
from decimal import Decimal, ROUND_HALF_EVEN
from typing import NewType, Self

OrderId = NewType("OrderId", int)          # type-checker-only distinction, no runtime cost

@dataclass(frozen=True, slots=True)
class Money:
    amount: Decimal
    currency: str

    def __post_init__(self) -> None:
        if len(self.currency) != 3 or not self.currency.isupper():
            raise ValueError(f"bad currency {self.currency!r}")
        q = self.amount.quantize(Decimal("0.01"), rounding=ROUND_HALF_EVEN)
        object.__setattr__(self, "amount", q)          # normalize inside frozen dataclass

    @classmethod
    def of(cls, amount: str | int, currency: str) -> Self:
        return cls(Decimal(amount), currency)          # never from float

    def __add__(self, other: "Money") -> "Money":
        if other.currency != self.currency:
            raise ValueError(f"cannot add {self.currency} and {other.currency}")
        return Money(self.amount + other.amount, self.currency)

    def allocate(self, parts: int) -> list["Money"]:
        """Split without losing cents: 10.00 / 3 -> 3.34, 3.33, 3.33."""
        cents = int(self.amount * 100)
        base, rem = divmod(cents, parts)
        return [Money(Decimal(base + (i < rem)) / 100, self.currency) for i in range(parts)]

assert Money.of("10", "EUR") == Money.of("10.00", "EUR")
assert sum(m.amount for m in Money.of("10", "EUR").allocate(3)) == Decimal("10.00")
```
With Pydantic, `class Money(BaseModel, frozen=True)` or an `Annotated[str, AfterValidator(...)]` type gives the same at API boundaries.

## Example — TypeScript
TS compares objects by reference, so value equality must be explicit. Branded types give nominal IDs at zero runtime cost.

```ts
export type OrderId = number & { readonly __brand: 'OrderId' };
export const OrderId = (n: number) => n as OrderId;

export class Money {
  private constructor(readonly minor: bigint, readonly currency: string) { Object.freeze(this); }

  static of(amount: string, currency: string): Money {
    if (!/^[A-Z]{3}$/.test(currency)) throw new Error(`bad currency ${currency}`);
    if (!/^-?\d+(\.\d{1,2})?$/.test(amount)) throw new Error(`bad amount ${amount}`);
    const [int, frac = ''] = amount.split('.');
    const sign = int.startsWith('-') ? -1n : 1n;
    return new Money(BigInt(int) * 100n + sign * BigInt(frac.padEnd(2, '0')), currency);
  }
  add(o: Money): Money {
    if (o.currency !== this.currency) throw new Error('currency mismatch');
    return new Money(this.minor + o.minor, this.currency);
  }
  equals(o: Money): boolean { return this.minor === o.minor && this.currency === o.currency; }
  toJSON() { return { amount: (Number(this.minor) / 100).toFixed(2), currency: this.currency }; }
}
```
In Angular, immutable value objects work naturally with signals and `OnPush`: replacing the object is the change signal. Give `signal()`/`computed()` a custom `equal: (a, b) => a.equals(b)` to avoid needless updates.

## In the wild
- Python: `datetime`, `Decimal`, `UUID`, `pathlib.Path`, `ipaddress.IPv4Address`, `NamedTuple`, frozen dataclasses, Pydantic `EmailStr`, `py-moneyed`.
- TS/JS: `Temporal` types (immutable dates; check runtime support), `URL` (mutable, a counter-example), Dinero.js for money, Zod branded types.
- DDD literature (Evans, Vernon); SQLAlchemy `composite()`; Java records and C# `record` as language-level value types.

## Trade-offs
| Pros | Cons |
|---|---|
| Invalid states unrepresentable after construction | More types and conversions at boundaries |
| Rules centralised; fewer duplicated checks | Serialization/ORM mapping needs configuration |
| Immutability makes sharing and caching safe | Allocation cost in hot loops |
| Type checker catches argument mix-ups | Over-wrapping primitives adds ceremony |

## Pitfalls
- Using `float` for money; use `Decimal` (Python) or integer minor units / `bigint` (TS).
- "Immutable" objects with mutable fields (`list` inside a frozen dataclass): use `tuple`/`frozenset`, or `ReadonlyArray`.
- Forgetting value equality in TS (`===` compares references), or forgetting `__hash__` consistency in Python when overriding `__eq__` on a non-dataclass.
- Validation bypassed by deserializers that set fields directly (`model_construct`, `Object.assign`). Route all creation through the validating factory.
- Giving value objects an id or persisting them in their own table with foreign keys — that turns them into entities.

## Refactoring path
- **Introduce it:** pick one primitive that caused bugs; create the value type with validation; convert it at boundaries (API parse, DB load); push it inward function by function, letting the type checker find call sites; delete duplicated validation.
- **Remove it / simplify:** if a wrapper has no rules and no behavior, replace it with `NewType`/branded type (still type-safe) or the plain primitive.

## Related patterns
- [DTO](../enterprise/dto.md) — boundary data carriers; value objects carry domain rules.
- [Domain Model](../data-access/domain-model.md) — entities and aggregates composed of value objects.
- [Flyweight](../structural/flyweight.md) — immutable values can be safely shared/interned.
- [Specification](../behavioral/specification.md) — rules over values, often composed with them.
- [Private Class Data](../security/private-class-data.md) — the immutability mechanism in classic OO form.
