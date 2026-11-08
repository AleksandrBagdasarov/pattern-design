# Builder

**Category:** Creational · **Also known as:** — · **Related:** Fluent Builder, Factory, Abstract Factory, Object Mother, Prototype, Composite

## Intent
Separate the step-by-step assembly and validation of a complex object from its final representation, so the same construction process is readable, can enforce invariants at the end, and can produce an immutable result.

## Problem
- A constructor with many parameters, most optional, several mutually dependent ("telescoping constructor"): `Order(customer, items, None, None, True, "EUR", None, ...)`.
- The object must be valid as a whole, but parts arrive at different times (from a multi-step form, a parser, several services).
- You want the final object immutable, but assembling it requires mutation.
- Scenario: building an outbound email with recipients, CC/BCC, a subject, plain and HTML bodies, attachments and headers. Rules: at least one recipient, at least one body, attachments under 10 MB total. Validation is currently spread across three callers.

## Solution and structure
- **Builder**: mutable object that collects parts through methods (`add_recipient`, `attach`). Holds no business logic other than assembly rules.
- **Product**: the immutable result, created only by `build()`.
- **build()**: validates cross-field invariants and returns the Product (or raises).
- **Director** (optional, GoF): encodes a reusable recipe (`build_password_reset_email(builder)`). In practice this is usually just a function.

```
client ─► builder.add_x() ─► builder.add_y() ─► builder.build() ─► Product (frozen, valid)
                                                    └─ validates invariants here
```

## When to use
- Many optional parts or complex nested structure (documents, requests, queries, reports, config trees).
- Invariants span several fields and must be checked once, before the object exists.
- The product should be immutable but its parts are gathered incrementally.
- The same steps should produce different representations (HTML vs PDF report from the same builder calls).

## When NOT to use
- The object has a handful of fields with defaults. In Python, use keyword arguments with defaults (a `@dataclass` or a Pydantic model already *is* the builder: named, optional, validated). In TS, use an options object `{ ... }`.
- Validation is per-field. Pydantic validators or a schema handle that without a builder.
- You need a copy with one field changed: `dataclasses.replace()` or object spread `{ ...o, x }`.
- The builder just mirrors setters one-to-one with no invariants. It adds a class that does nothing.

## Example — Python
Python's keyword arguments remove most reasons for a builder. It remains useful when parts are accumulated (lists) and validated together.

```python
from dataclasses import dataclass, field
from email.headerregistry import Address

MAX_ATTACH = 10 * 1024 * 1024

@dataclass(frozen=True, slots=True)
class Email:
    to: tuple[str, ...]
    subject: str
    text: str | None
    html: str | None
    attachments: tuple[tuple[str, bytes], ...]

@dataclass
class EmailBuilder:
    subject: str
    _to: list[str] = field(default_factory=list)
    _text: str | None = None
    _html: str | None = None
    _files: list[tuple[str, bytes]] = field(default_factory=list)

    def to(self, addr: str) -> None: self._to.append(str(Address(addr_spec=addr)))
    def text(self, body: str) -> None: self._text = body
    def html(self, body: str) -> None: self._html = body
    def attach(self, name: str, data: bytes) -> None: self._files.append((name, data))

    def build(self) -> Email:
        if not self._to:
            raise ValueError("email needs at least one recipient")
        if self._text is None and self._html is None:
            raise ValueError("email needs a text or html body")
        if sum(len(d) for _, d in self._files) > MAX_ATTACH:
            raise ValueError("attachments exceed 10 MB")
        return Email(tuple(self._to), self.subject, self._text, self._html, tuple(self._files))

def invoice_email(customer_email: str, pdf: bytes) -> Email:   # a "director"
    b = EmailBuilder(subject="Your invoice")
    b.to(customer_email)
    b.text("Invoice attached.")
    b.attach("invoice.pdf", pdf)
    return b.build()
```

## Example — TypeScript
Parts collected across steps of a wizard, then validated and frozen once. A chainable variant is in [Fluent Builder](fluent-builder.md).

```typescript
export interface ShipmentRequest {
  readonly orderId: string;
  readonly address: Readonly<{ line1: string; city: string; country: string; zip: string }>;
  readonly parcels: ReadonlyArray<{ weightKg: number }>;
  readonly insuredValue?: number;
}

export class ShipmentRequestBuilder {
  private address?: ShipmentRequest['address'];
  private readonly parcels: { weightKg: number }[] = [];
  private insuredValue?: number;

  constructor(private readonly orderId: string) {}

  setAddress(a: ShipmentRequest['address']): void { this.address = a; }
  addParcel(weightKg: number): void { this.parcels.push({ weightKg }); }
  insure(value: number): void { this.insuredValue = value; }

  build(): ShipmentRequest {
    if (!this.address) throw new Error('address is required');
    if (this.parcels.length === 0) throw new Error('at least one parcel');
    if (this.parcels.some(p => p.weightKg <= 0 || p.weightKg > 30)) throw new Error('parcel weight 0-30 kg');
    if (this.address.country !== 'DE' && this.insuredValue === undefined) {
      throw new Error('international shipments must be insured');
    }
    return Object.freeze({
      orderId: this.orderId,
      address: Object.freeze({ ...this.address }),
      parcels: Object.freeze(this.parcels.map(p => ({ ...p }))),
      insuredValue: this.insuredValue,
    });
  }
}
```

## In the wild
- Python: `email.message.EmailMessage` (incremental assembly), `str.join` over a list (the idiomatic string builder), `io.StringIO`, `argparse.ArgumentParser` (build a parser, then `parse_args`), `httpx.Request` construction.
- SQLAlchemy `select(...).where(...).order_by(...)`; Pydantic `model_validate` acts as the `build()` step over raw input.
- Angular: `FormBuilder` / `NonNullableFormBuilder` building `FormGroup` trees; `HttpParams`/`HttpHeaders` (immutable, each `set` returns a new instance).
- Node: `URLSearchParams`, Knex/Prisma query builders, AWS CDK constructs assembled before `synth`.

## Trade-offs
| Pros | Cons |
|---|---|
| Readable assembly of complex objects | Extra class that duplicates the product's fields |
| Cross-field validation in one place (`build()`) | Product and builder must be kept in sync as fields change |
| Immutable product with incremental construction | Overkill for objects that keyword args/option objects cover |
| Reusable recipes (directors) for common variants | Mutable builder shared across threads/requests is a bug source |

## Pitfalls
- **`build()` doesn't validate**, so invalid products escape. Fix: put every invariant in `build()`; make the product constructor private-by-convention or validate there too.
- **Product holds references to builder's mutable lists**, so later `add_x` mutates a "built" object. Fix: copy to tuples / `Object.freeze` copies in `build()`.
- **Reusing one builder for several products** and getting leftovers. Fix: one builder per product, or a `reset()` that is tested.
- **Builder as a shared singleton** in DI. Fix: builders are short-lived, created per use.
- **Builders for DTOs that Pydantic/dataclasses already handle.** Delete them.

## Refactoring path
- **Introduce it:** start from a constructor with many optional params or from callers that assemble then validate. Step 1: move validation into one function. Step 2: create a builder holding the parts with that function as `build()`. Step 3: switch callers one at a time. Step 4: freeze the product.
- **Remove it / simplify:** if the builder has no cross-field rules and only sets fields, replace it with keyword arguments / an options object plus a model validator, then delete the builder.

## Related patterns
- [Fluent Builder](fluent-builder.md) — the same pattern with chainable methods returning the builder.
- [Factory](factory.md) — selects *which* type to create; Builder controls *how* one complex instance is assembled.
- [Abstract Factory](abstract-factory.md) — returns finished family members; builders can be one of its products.
- [Object Mother](object-mother.md) — test fixtures; often implemented on top of a test-data builder.
- [Prototype](prototype.md) — clone a configured instance instead of rebuilding it from parts.
- [Composite](../structural/composite.md) — builders are a common way to assemble composite trees.
