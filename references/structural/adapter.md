# Adapter

**Category:** Structural · **Also known as:** Wrapper (loosely), Translator · **Related:** Facade, Decorator, Proxy, Bridge, Hexagonal (Ports and Adapters), Message Translator

## Intent
Convert the interface of an existing class into the interface your code expects, so that classes with incompatible interfaces can work together without changing either of them.

## Problem
- Your domain code expects `PaymentGateway.charge(order_id, amount: Decimal)`, but the vendor SDK exposes `stripe.PaymentIntent.create(amount=int_cents, currency=..., metadata=...)`.
- Vendor types (`stripe.StripeError`, `requests.Response`, AWS `ClientError`) leak into business logic, so swapping or upgrading a vendor touches dozens of files.
- Two libraries model the same concept differently (cents vs. `Decimal`, epoch seconds vs. `datetime`, callbacks vs. promises).
- Tests need to patch deep SDK internals because there is no seam of your own.

Scenario: you integrate Stripe for card payments and later must add Adyen for EU customers. Without an adapter, `if provider == "stripe"` branches spread across the checkout service.

## Solution and structure
- **Target**: the interface your code depends on (a `Protocol` / TS `interface`), shaped by *your* needs.
- **Adaptee**: the existing, incompatible class (third-party SDK, legacy module).
- **Adapter**: implements Target, holds an Adaptee, translates calls, data types and errors.
- **Client**: depends only on Target.

```
Client ──> Target (Protocol)
              ▲
              │ implements
           Adapter ──holds──> Adaptee (vendor SDK)
```

Prefer the *object adapter* (composition) over the *class adapter* (multiple inheritance from Target and Adaptee); composition keeps the adaptee's surface hidden.

## When to use
- Integrating a third-party SDK, legacy module or external API whose shape you do not control.
- You want vendor-independence at a boundary (ports and adapters / anti-corruption layer).
- Several implementations of the same capability must look identical to the caller (Stripe/Adyen, S3/GCS, SMTP/SES).
- You need a test seam: the Target can be faked without touching the SDK.

## When NOT to use
- You own both sides: change one interface instead of translating between them.
- The mismatch is a single call site: a small helper function is enough.
- You only need to *simplify* a large subsystem, not translate it: use a Facade.
- You only need to add behavior with the same interface: use a Decorator.
- "Adapter per class just in case" around stable stdlib types (`datetime`, `pathlib`) is over-engineering.

## Example — Python
In Python, the Target is a `typing.Protocol`; the adapter is a plain class that satisfies it structurally.

```python
from dataclasses import dataclass
from decimal import Decimal
from typing import Protocol

import stripe  # third-party adaptee


class PaymentDeclined(Exception): ...


@dataclass(frozen=True)
class ChargeResult:
    provider_ref: str
    amount: Decimal


class PaymentGateway(Protocol):  # Target, owned by the domain
    def charge(self, order_id: str, amount: Decimal, currency: str) -> ChargeResult: ...


class StripeGatewayAdapter:
    def __init__(self, client: stripe.StripeClient) -> None:
        self._client = client

    def charge(self, order_id: str, amount: Decimal, currency: str) -> ChargeResult:
        cents = int((amount * 100).to_integral_value())
        try:
            intent = self._client.payment_intents.create(
                params={"amount": cents, "currency": currency.lower(),
                        "metadata": {"order_id": order_id}, "confirm": True}
            )
        except stripe.CardError as exc:          # translate vendor errors
            raise PaymentDeclined(str(exc)) from exc
        return ChargeResult(provider_ref=intent.id, amount=amount)


def checkout(gateway: PaymentGateway, order_id: str, total: Decimal) -> str:
    return gateway.charge(order_id, total, "EUR").provider_ref
```

The Stripe SDK API (`StripeClient` vs. module-level `stripe.PaymentIntent`) has changed across major versions; check the current docs before copying the call shape.

## Example — TypeScript
An Angular service adapting a backend's snake_case DTO to the frontend model, with the HTTP shape isolated in one place.

```ts
import { Injectable, inject } from '@angular/core';
import { HttpClient } from '@angular/common/http';
import { Observable, map } from 'rxjs';

interface TaskDto { id: number; title: string; is_done: boolean; due_at: string | null }
export interface Task { id: number; title: string; done: boolean; due: Date | null }

export abstract class TaskSource {               // Target (abstract class = DI token)
  abstract list(): Observable<Task[]>;
}

const toTask = (d: TaskDto): Task => ({
  id: d.id, title: d.title, done: d.is_done,
  due: d.due_at ? new Date(d.due_at) : null,
});

@Injectable()
export class HttpTaskSourceAdapter extends TaskSource {
  private readonly http = inject(HttpClient);    // Adaptee
  list(): Observable<Task[]> {
    return this.http.get<TaskDto[]>('/api/tasks').pipe(map((ds) => ds.map(toTask)));
  }
}

// app.config.ts: providers: [{ provide: TaskSource, useClass: HttpTaskSourceAdapter }]
```

## In the wild
- Python: `io.TextIOWrapper` adapts a binary stream to text; `csv.reader` over any iterable of strings; `asyncio.to_thread` / `loop.run_in_executor` adapt blocking calls to `await`; `contextlib.closing` adapts `.close()` to the context-manager protocol.
- SQLAlchemy dialects and DB-API drivers adapt each database to one interface; Starlette `WSGIMiddleware` adapts WSGI apps to ASGI.
- Node: `util.promisify` adapts callback APIs to promises; RxJS `from()` / `fromEvent()` adapt promises, iterables and DOM events to Observables.
- Angular: `toSignal()` / `toObservable()` adapt between RxJS and signals; `ControlValueAccessor` adapts any widget to Angular forms.
- Cloud SDKs: boto3 resource vs. client layers; Terraform providers adapt each cloud API to one resource model.

## Trade-offs
| Pros | Cons |
|---|---|
| Vendor types and errors stay out of the domain | One more class and one more indirection |
| Swap or add providers without touching callers | Lowest-common-denominator Target can hide useful vendor features |
| Clean test seam: fake the Target, not the SDK | Translation code must be kept in sync with SDK upgrades |
| Single place to translate units, errors, naming | Easy to drift into a leaky adapter that returns vendor objects |

## Pitfalls
- **Leaky adapter**: returning the vendor object or raising vendor exceptions. Fix: map to your own types and exception hierarchy.
- **Target shaped like the vendor**: copying the SDK's method list into your Protocol. Fix: design the Target from the caller's needs, then adapt.
- **Silent data loss** in translation (rounding money, dropping timezones). Fix: use `Decimal`/integer minor units consistently, test the mapping with edge values.
- **Adapter grows business logic** (retries, pricing rules). Fix: keep it a translator; put retries in a Decorator and rules in the domain.
- **Class adapter via inheritance** from the SDK class, which exposes every SDK method. Fix: compose.

## Refactoring path
- **Introduce it:** find every import of the vendor SDK; define a Protocol with only the operations the domain uses; move the SDK calls into one adapter class; inject it; replace vendor exceptions with domain ones; add a fake for tests.
- **Remove it / simplify:** if there is one implementation, no test seam needed and the "adapter" is a one-line pass-through, inline it into a helper function or call the SDK directly at the edge.

## Related patterns
- [Facade](../structural/facade.md) — simplifies a subsystem with a new interface; Adapter conforms to an existing one.
- [Decorator](../structural/decorator.md) — same interface plus behavior; Adapter changes the interface.
- [Proxy](../structural/proxy.md) — same interface, controls access.
- [Bridge](../structural/bridge.md) — designed up front to separate abstraction from implementation; Adapter is retrofitted.
- [Wrapper](../structural/wrapper.md) — how to choose among the wrapper family.
- [Hexagonal](../architectural/hexagonal.md) — adapters are the outer ring around domain ports.
- [Message Translator](../enterprise/message-translator.md) — Adapter for messages instead of method calls.
