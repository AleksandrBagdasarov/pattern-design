# Message Translator

**Category:** Enterprise integration · **Also known as:** Anti-corruption layer (ACL, DDD), mapper, normalizer, canonical data model translator · **Related:** Adapter, DTO, Content Filter, Data Mapper, ESB, Domain Model

## Intent
Convert a message from one system's format, schema or vocabulary into another's, so neither side has to adopt the other's model.

## Problem
- A payment provider's webhook payload (`{"obj": {"amt": 1999, "cur": "usd", "st": "succeeded"}}`) is parsed directly in business logic; their field renames break your domain code.
- Two bounded contexts use the same word for different things ("account" in billing vs identity).
- Version 2 of your own event schema must coexist with consumers that still expect version 1.
- Scenario: you integrate Stripe and Adyen; each has its own payment-status vocabulary, and the order service should see only `PaymentSucceeded` / `PaymentFailed`.

## Solution and structure
A translator is a pure function (or a small class) from the source schema to the target schema. Place it at the boundary, owned by the side that wants protection.

```
Stripe webhook JSON ──validate──> StripeEvent (their DTO) ──translate──> PaymentResult (our model) ──> domain
Adyen notification  ──validate──> AdyenItem   (their DTO) ──translate──┘
```
Layers of translation (EIP): **data types** (cents vs decimal, epoch vs ISO-8601), **data structure** (nesting, renames), **semantics** (status vocabularies, units), **transport** (XML/JSON/Avro/CSV). A **Normalizer** routes several source formats to their translators and emits one canonical form. In DDD, a translator set at a context boundary is the **anti-corruption layer**.

## When to use
- Integrating third-party APIs, webhooks, legacy systems or another team's events.
- Bounded contexts that must keep their own models.
- Schema versioning: upcasting old event versions to the current one on read.
- Multiple vendors behind one internal interface (with Adapter).

## When NOT to use
- Both sides are yours and can share one contract (same service, same DTO): translation adds mapping without protection.
- Chains of translators A→B→C→D for one hop: translate once, directly to what the consumer needs.
- Putting business decisions in the translator (approving, rejecting): it should only map; decisions belong in the domain.

## Example — Python
Validate the external shape with Pydantic, then translate to an internal model; vendor vocabulary does not leak.

```python
from dataclasses import dataclass
from decimal import Decimal
from enum import StrEnum
from typing import Literal
from pydantic import BaseModel

class StripeObj(BaseModel):          # their schema (only the fields we use)
    id: str
    amount: int                      # minor units
    currency: str                    # lowercase ISO code
    status: Literal["succeeded", "processing", "requires_payment_method", "canceled"]
    metadata: dict[str, str]

class StripeEvent(BaseModel):
    type: str
    data: dict[str, StripeObj]

class PaymentStatus(StrEnum):
    SUCCEEDED = "succeeded"; PENDING = "pending"; FAILED = "failed"

@dataclass(frozen=True)
class PaymentResult:                 # our model
    order_id: str
    provider_ref: str
    amount: Decimal
    currency: str
    status: PaymentStatus

_STATUS = {"succeeded": PaymentStatus.SUCCEEDED, "processing": PaymentStatus.PENDING,
           "requires_payment_method": PaymentStatus.FAILED, "canceled": PaymentStatus.FAILED}

def from_stripe(raw: bytes) -> PaymentResult:
    obj = StripeEvent.model_validate_json(raw).data["object"]
    return PaymentResult(
        order_id=obj.metadata["order_id"],
        provider_ref=obj.id,
        amount=Decimal(obj.amount) / 100,            # wrong for zero-decimal currencies (JPY)
        currency=obj.currency.upper(),
        status=_STATUS[obj.status],
    )
```
The minor-unit conversion depends on the currency's exponent; use a currency table in real code. Field names above are simplified; consult the provider's current API reference.

## Example — TypeScript
Angular: translate a backend DTO (snake_case, strings) into a view model at the service boundary, so components never see the wire format.

```ts
import { Injectable, inject } from '@angular/core';
import { HttpClient } from '@angular/common/http';
import { map, Observable } from 'rxjs';

interface InvoiceDto {               // backend contract (e.g. generated from OpenAPI)
  invoice_id: string; total: string; currency: string;
  issued_at: string; status: 'open' | 'paid' | 'void';
}

export interface InvoiceVm {         // what components consume
  id: string; total: number; currency: string; issuedAt: Date;
  isPayable: boolean; label: string;
}

export function toInvoiceVm(d: InvoiceDto): InvoiceVm {
  return {
    id: d.invoice_id,
    total: Number(d.total),
    currency: d.currency,
    issuedAt: new Date(d.issued_at),
    isPayable: d.status === 'open',
    label: { open: 'Awaiting payment', paid: 'Paid', void: 'Cancelled' }[d.status],
  };
}

@Injectable({ providedIn: 'root' })
export class InvoicesApi {
  private readonly http = inject(HttpClient);
  list(): Observable<InvoiceVm[]> {
    return this.http.get<InvoiceDto[]>('/api/invoices').pipe(map((xs) => xs.map(toInvoiceVm)));
  }
}
```
Converting money to `number` is fine for display; keep decimal strings (or integer cents) for anything that calculates.

## In the wild
- Kafka Connect Single Message Transforms, Confluent Schema Registry compatibility rules, Avro/Protobuf schema evolution, event upcasters in Axon/Marten.
- Apache Camel/Spring Integration transformers, MuleSoft DataWeave, XSLT, JSONata, AWS EventBridge input transformers, AWS AppSync resolvers.
- Pydantic `model_validate` + mapping functions, `class-transformer`, Angular service `map()` operators, OpenAPI codegen + mapping layers.

## Trade-offs
| Pros | Cons |
|---|---|
| Internal model protected from external changes | Mapping code to write, test and maintain |
| One place to absorb vendor/version differences | Lossy translations hide information if not careful |
| Enables independent evolution of bounded contexts | Semantic mismatches can't always be mapped cleanly |
| Pure functions: easy to unit test with fixtures | Extra hop/latency when deployed as a separate component |

## Pitfalls
- **Translating without validating the input first:** `KeyError`/`undefined` deep in mapping. Fix: validate the source schema, then map.
- **Silent defaults for unknown values** (`status` unmapped → "pending"). Fix: fail loudly or route to a DLQ; exhaustive mappings (`Literal`, TS union + `Record`).
- **Lossy number/time conversions** (float money, timezone drop, currency exponents). Fix: decimals/minor units, aware datetimes.
- **Business rules hidden in the translator.** Fix: map only; decide in the domain.
- **Shared "canonical" model that every context must accept.** Fix: translate per context pair at boundaries you own.

## Refactoring path
- **Introduce it:** find business code reading vendor fields directly; define the internal model; write `from_vendor()` with fixture-based tests from real payloads; switch callers; forbid vendor DTO imports outside the integration module (lint rule/import-linter).
- **Remove it / simplify:** if source and target schemas became identical (you own both and merged them), delete the identity mapping and share the DTO.

## Related patterns
- [Adapter](../structural/adapter.md) — adapts an interface (calls); the translator adapts data (messages). Often used together.
- [DTO](../enterprise/dto.md) — the source and target shapes being translated.
- [Content Filter](../enterprise/content-filter.md) — a translator that only removes fields.
- [Data Mapper](../data-access/data-mapper.md) — translation between domain objects and the DB.
- [ESB](../enterprise/esb.md) — hosts translators centrally against a canonical model.
- [Domain Model](../data-access/domain-model.md) — what the anti-corruption layer protects at a bounded-context edge.
