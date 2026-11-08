# Content Filter

**Category:** Enterprise integration · **Also known as:** Projection, field filter, sparse fieldset · **Related:** DTO, Message Translator, Content Enricher, Claim Check, Splitter

## Intent
Remove everything from a message that the recipient does not need, leaving a smaller, simpler message with only the relevant fields.

## Problem
- A full `Customer` record (address, payment methods, notes, internal risk score) is published on every change, but the email service needs only `id`, `email`, `locale`.
- Sensitive fields (PII, tokens, internal flags) leak to consumers or logs that should not see them.
- Large nested payloads cost bandwidth, storage and serialization time; consumers couple to fields they never use.
- Scenario: a third-party analytics webhook receives full orders including customer addresses, creating a GDPR problem.

## Solution and structure
A filter step projects the incoming message onto a whitelist of fields (and may flatten nested structures). It sits on the producer side (publish a slim event), in the integration layer (per-subscriber transformation), or on the API boundary (response DTO, sparse fieldsets).

```
Customer{id, email, locale, address, cards, risk_score, notes}
        ──ContentFilter(id, email, locale)──> CustomerContact{id, email, locale} ──> email-svc
```
Prefer **whitelisting** (pick fields) over blacklisting (remove fields): new sensitive fields are then excluded by default.

## When to use
- Consumers need a small subset of a large message.
- Data minimization and privacy: PII must not cross a boundary (third parties, logs, analytics).
- Reducing payload size for mobile clients or high-volume streams.
- Hiding internal structure so consumers cannot couple to it.

## When NOT to use
- Consumers genuinely need the full state (event-carried state transfer into a replica): filtering forces them to call back.
- When filtering hides what should be a different event: emit a purpose-specific event instead of filtering a generic one.
- Field-level filtering of every response by client request (`?fields=`) when nobody needs it: extra complexity and cache-key explosion.

## Example — Python
Pydantic projection models act as whitelisting filters; nested fields are flattened explicitly.

```python
from pydantic import BaseModel, ConfigDict, EmailStr

class Address(BaseModel):
    street: str
    city: str
    country: str

class Customer(BaseModel):                        # full internal model
    id: str
    email: EmailStr
    locale: str
    address: Address
    risk_score: float
    notes: str

class CustomerContact(BaseModel):                 # filtered message for email-svc
    model_config = ConfigDict(extra="ignore", frozen=True)
    id: str
    email: EmailStr
    locale: str

class CustomerRegion(BaseModel):                  # filtered + flattened for analytics
    id: str
    country: str

def to_contact(c: Customer) -> CustomerContact:
    return CustomerContact.model_validate(c.model_dump(include={"id", "email", "locale"}))

def to_region(c: Customer) -> CustomerRegion:
    return CustomerRegion(id=c.id, country=c.address.country)

# Ad hoc: c.model_dump(include={"id": True, "address": {"country"}})
#   -> {"id": "...", "address": {"country": "UA"}}
```
`extra="ignore"` is Pydantic's default; spelling it out documents that unknown input fields are dropped rather than rejected.

## Example — TypeScript
A typed `pick` helper: the compiler checks the field list, the runtime copies only those keys.

```ts
export function pick<T extends object, K extends keyof T>(obj: T, keys: readonly K[]): Pick<T, K> {
  const out = {} as Pick<T, K>;
  for (const k of keys) if (k in obj) out[k] = obj[k];
  return out;
}

interface Customer {
  id: string; email: string; locale: string;
  address: { street: string; city: string; country: string };
  riskScore: number; notes: string;
}

const CONTACT_FIELDS = ['id', 'email', 'locale'] as const;
export type CustomerContact = Pick<Customer, (typeof CONTACT_FIELDS)[number]>;

export const toContact = (c: Customer): CustomerContact => pick(c, CONTACT_FIELDS);

export const toRegion = (c: Customer) => ({ id: c.id, country: c.address.country });

// NestJS alternative on responses: class-transformer @Exclude()/@Expose() with
// ClassSerializerInterceptor; whitelist with excludeExtraneousValues: true.
```
Note that `Pick<T, K>` alone is compile-time only: casting a full object to `CustomerContact` still sends every field over the wire. The runtime copy is what filters.

## In the wild
- SQL `SELECT col1, col2` (the cheapest filter: don't load what you don't need), MongoDB projections, DynamoDB `ProjectionExpression`.
- GraphQL field selection; JSON:API sparse fieldsets (`?fields[articles]=title`); Google APIs `fields=` partial responses.
- FastAPI `response_model` / `response_model_include`, DRF serializers with `fields`, EventBridge input transformers, Kafka Connect `ReplaceField` SMT, Debezium column filters, logging redaction filters.

## Trade-offs
| Pros | Cons |
|---|---|
| Smaller payloads, less bandwidth and storage | Consumers needing more must call back (chatty) |
| Data minimization; fewer PII leaks | One filtered variant per consumer can proliferate |
| Consumers decoupled from internal fields | Changing which fields a consumer gets is a contract change |
| Simpler consumer code | Filtering after loading still pays the load cost |

## Pitfalls
- **Blacklisting** (`exclude={"password"}`): new sensitive fields leak automatically. Fix: whitelist.
- **Type-only filtering in TS** (`as Pick<...>`). Fix: copy fields at runtime or validate with a schema that strips unknown keys.
- **Filtering after `SELECT *`** of large rows/blobs. Fix: push the projection into the query.
- **Filtered event lacks the ID/version** consumers need to fetch more or order updates. Fix: always keep identity, version and timestamp.
- **Logs bypass the filter.** Fix: log the filtered DTO, or use redaction in the logging pipeline.

## Refactoring path
- **Introduce it:** list what each consumer actually reads (grep, contract tests); define a whitelist model per consumer; publish/return it instead of the full object; then push the projection into the DB query.
- **Remove it / simplify:** if every consumer ends up with the full object anyway, drop the per-consumer variants and publish one well-defined DTO.

## Related patterns
- [DTO](../enterprise/dto.md) — an output DTO is the most common content filter.
- [Message Translator](../enterprise/message-translator.md) — the general transformation; a filter only removes data.
- [Splitter](../enterprise/splitter.md) — splits a message into parts rather than dropping fields.
- [Aggregator](../enterprise/aggregator.md) — the opposite direction: combining data into a larger message.
- Content Enricher (EIP, no separate reference) — adds missing data; the inverse of a filter. Claim Check — replaces a large payload with a reference.
