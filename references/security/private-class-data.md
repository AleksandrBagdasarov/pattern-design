# Private Class Data

**Category:** Security · **Also known as:** Encapsulated Data, Immutable State, Data Hiding · **Related:** Value Object, Secure Factory / Builder, Builder, Memento, Secure Proxy, Optional

## Intent
Restrict write access (and, where needed, read access) to an object's attributes after construction by moving them into a private, typically immutable data holder, so sensitive or invariant-bearing state cannot be changed or leaked by the object's own methods or by callers.

## Problem
- Attributes that must never change after creation (an API key, a user's tenant ID, an order's currency, a price once quoted) are plain mutable fields; any method or caller can reassign them.
- Sensitive values leak through `repr`, `__dict__`, JSON serialization, debug logs or error trackers.
- Scenario: a `PaymentIntent` object carries `amount`, `currency` and the customer's `card_token`. A refactor added `intent.amount = adjusted` in a discount helper after the amount had been authorized, and Sentry captured `card_token` in a local-variables dump. Separating the immutable, redacted data from the behavior removes both problems.
- Smell: "this field must not change" written as a comment instead of being enforced.

## Solution and structure
- **Data class**: holds the protected attributes; immutable (frozen) and with a redacted representation for secrets.
- **Main class**: receives the data object at construction, keeps it in a private attribute, exposes read-only accessors (or nothing) and behavior that uses the data.
- Changes produce a *new* object (copy-with-changes) rather than mutating, which keeps invariants checked at one point (construction).

```
PaymentIntent ──has──► _data: PaymentData (frozen: amount, currency, card_token[redacted])
   .amount (read-only)                   ▲
   .capture() uses _data                 └── created once, validated in __post_init__
```

Language reality: Python has no true private attributes (`_x` is convention, `__x` is name mangling) and TS `private` is compile-time only. Use enforcement that actually holds at runtime: frozen dataclasses/`slots`, Pydantic `frozen` + `SecretStr`, JS `#private` fields and `Object.freeze`. None of these protect against code running in the same process with malicious intent; they protect against mistakes and accidental leaks.

## When to use
- Fields carry invariants or security meaning that must be fixed after creation (identity, tenant, money, credentials).
- The object is passed widely and you cannot review every caller.
- Secrets live in memory and must not appear in logs, `repr`, serialization or crash reports.

## When NOT to use
- Plain DTOs that are meant to be mutated and serialized (form models, request bodies). Use a normal dataclass/model.
- When a frozen dataclass is already enough: don't add a second "data" class plus wrapper just to follow the GoF shape. In Python and TS, `@dataclass(frozen=True)` or `readonly` + `#private` usually *is* the pattern.
- Hot paths that rebuild large objects on every change; consider a mutable internal structure with a narrow API instead.

## Example — Python
A frozen, slotted data holder with a redacted secret, wrapped by a behavior class with read-only properties.

```python
from dataclasses import dataclass, field, replace
from decimal import Decimal
from pydantic import SecretStr

@dataclass(frozen=True, slots=True)
class PaymentData:
    intent_id: str
    tenant_id: str
    amount: Decimal
    currency: str
    card_token: SecretStr = field(repr=False)          # repr shows nothing; SecretStr prints '**********'

    def __post_init__(self) -> None:
        if self.amount <= 0:
            raise ValueError("amount must be positive")
        if len(self.currency) != 3:
            raise ValueError("currency must be an ISO 4217 code")

class PaymentIntent:
    __slots__ = ("_data", "_captured")

    def __init__(self, data: PaymentData) -> None:
        self._data = data
        self._captured = False

    @property
    def amount(self) -> Decimal: return self._data.amount      # read-only, no setter
    @property
    def currency(self) -> str: return self._data.currency

    def with_amount(self, new_amount: Decimal) -> "PaymentIntent":
        if self._captured:
            raise RuntimeError("cannot change a captured intent")
        return PaymentIntent(replace(self._data, amount=new_amount))  # re-runs validation

    def capture(self, gateway: "Gateway") -> None:
        gateway.capture(self._data.intent_id, self._data.card_token.get_secret_value())
        self._captured = True

    def __repr__(self) -> str:
        return f"PaymentIntent({self._data.intent_id}, {self.amount} {self.currency})"
# intent.amount = 1  -> AttributeError; intent._data.amount = 1 -> FrozenInstanceError
```

## Example — TypeScript
JS `#private` fields are enforced at runtime (unlike TS `private`), and `toJSON` controls serialization so the secret never reaches logs or HTTP responses.

```typescript
interface ApiCredentialsData { readonly clientId: string; readonly tenantId: string; readonly secret: string; }

export class ApiCredentials {
  readonly #data: Readonly<ApiCredentialsData>;

  constructor(data: ApiCredentialsData) {
    if (!/^[a-z0-9-]{8,}$/.test(data.clientId)) throw new Error('invalid clientId');
    this.#data = Object.freeze({ ...data });          // defensive copy + freeze
  }

  get clientId(): string { return this.#data.clientId; }
  get tenantId(): string { return this.#data.tenantId; }

  /** The only way to use the secret: it is never returned to callers. */
  authorizationHeader(): string {
    return `Basic ${btoa(`${this.#data.clientId}:${this.#data.secret}`)}`;
  }

  toJSON() { return { clientId: this.clientId, tenantId: this.tenantId, secret: '[REDACTED]' }; }
  toString() { return `ApiCredentials(${this.clientId})`; }
}

const creds = new ApiCredentials({ clientId: 'shop-frontend', tenantId: 't-42', secret: 's3cr3t' });
console.log(JSON.stringify(creds));          // {"clientId":"shop-frontend","tenantId":"t-42","secret":"[REDACTED]"}
// (creds as any).#data -> SyntaxError; creds.clientId = 'x' -> TypeError in strict mode (getter only)

// Angular: expose state read-only from services the same way.
// private readonly _cart = signal<Cart>(empty); readonly cart = this._cart.asReadonly();
```

## In the wild
- Python: `@dataclass(frozen=True, slots=True)`, `typing.NamedTuple`, Pydantic `ConfigDict(frozen=True)` and `SecretStr`/`SecretBytes`, `types.MappingProxyType` (read-only dict view), properties without setters.
- JS/TS: `#private` fields, `Object.freeze`, `readonly`/`ReadonlyArray`, Angular `signal().asReadonly()` and `BehaviorSubject` exposed as `asObservable()`.
- Frameworks: Django `SECRET_KEY` handling, Sentry/structlog scrubbers (`send_default_pii=False`, `before_send`), AWS SDK credential objects that redact in `repr`.
- Other languages: Java records, Kotlin `val`, C# `init`-only properties and records.

## Trade-offs
| Pros | Cons |
|---|---|
| Invariants enforced after construction, not by convention | Updates require copies; more allocation |
| Secrets don't leak through repr/serialization | Two classes (data + behavior) where one may do |
| Safer to share objects across threads/requests | Python/TS privacy is bypassable by determined code |
| Clear single point of validation | Frozen nested containers need care (lists inside frozen classes are still mutable) |

## Pitfalls
- **Frozen outer object, mutable inner list/dict.** Fix: tuples/`frozenset`/`MappingProxyType`; `ReadonlyArray` plus `Object.freeze` copies.
- **TS `private` assumed to be runtime protection.** It is erased at compile time; use `#private`.
- **Getter returning the internal mutable object** (`get items() { return this.#items; }`). Fix: return copies or readonly views.
- **Secrets still serialized** by `dataclasses.asdict`, `model_dump()` or `JSON.stringify` of a nested plain object. Fix: `SecretStr` (dumps as `'**********'` unless explicitly revealed), `toJSON`, and scrubbing in error trackers.
- **`object.__setattr__` used to "fix" a frozen object** in a hurry. Fix: create a new instance via `replace`.
- **Secrets kept longer than needed** in long-lived objects. Fix: fetch per use from a secret provider, or clear references after use.

## Refactoring path
- **Introduce it:** identify fields that are set once and must not change, and fields that are secret. Move them into a frozen data class with validation in `__post_init__`/constructor; make the main class hold it privately; replace setters with copy-with-changes methods; add redaction for secrets and a test that `repr`/JSON output contains no secret.
- **Remove it / simplify:** if the wrapper class adds no behavior, merge it into the frozen data class itself; if the data must legitimately change often, replace with a mutable class with a narrow, validated API.

## Related patterns
- [Value Object](../emergent/value-object.md) — immutable, compared by value; often the data holder here.
- [Secure Factory / Builder](secure-factory-builder.md) — creates objects with secrets correctly; Private Class Data keeps them protected afterwards.
- [Builder](../creational/builder.md) — collects parts mutably, then emits the frozen product.
- [Memento](../behavioral/memento.md) — also an opaque state holder, used for restore rather than protection.
- [Secure Proxy](secure-proxy.md) — restricts who can call methods; Private Class Data restricts what can change.
- [Optional](../emergent/optional.md) — explicit absence for fields that may legitimately be missing.
