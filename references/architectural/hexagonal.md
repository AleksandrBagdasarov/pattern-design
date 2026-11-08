# Hexagonal Architecture (Ports and Adapters)

**Category:** Architectural · **Also known as:** Ports and Adapters · **Related:** Clean Architecture, Onion, Adapter, Dependency Injection, Repository, Anti-corruption layer

## Intent
Put the application core in the middle and let it talk to the outside world only through **ports** (interfaces owned by the core). **Adapters** plug technologies into those ports, so the core can be driven by HTTP, CLI, queues or tests and can drive any database or vendor without changing.

## Problem
- The same logic must be triggered by a REST call, a Kafka message, a cron job and a test, and each entry point currently duplicates it.
- Business code calls `requests`, `boto3` or the ORM directly, so tests either hit the network or drown in `mock.patch`.
- A third-party API's data model leaks into your domain names and types.
- Scenario: a shipping service gets quotes from three carriers, receives orders over HTTP and SQS, and must be testable offline. Each carrier SDK has its own error types and field names.

## Solution and structure
```
   driving (primary) side                     driven (secondary) side
  HTTP adapter ─┐                           ┌─► SqlShipmentRepo
  SQS consumer ─┼─► [ inbound port ]        │
  CLI / tests  ─┘    QuoteShipment use case ├─► [ outbound port ] CarrierClient ─► DhlAdapter
                     (application core)     └─► [ outbound port ] Clock          ─► FedexAdapter
```
- **Inbound (driving) ports:** the API the core offers (use-case functions/classes).
- **Outbound (driven) ports:** interfaces the core needs (`CarrierClient`, `ShipmentRepository`, `Clock`).
- **Primary adapters** call inbound ports; **secondary adapters** implement outbound ports.
- A composition root wires adapters to ports. The hexagon shape has no meaning; it just shows "many sides, all equal".

## When to use
- Many I/O channels into and out of one core.
- Integrations with unstable or multiple vendors (payments, carriers, LLM providers).
- You want the test suite to run the real core against fake adapters.

## When NOT to use
- A thin proxy or CRUD API with no logic: ports add a layer of pure forwarding.
- A single adapter per port forever and no tests using fakes: plain functions with clear module boundaries suffice.
- When the "port" would just mirror one SDK one-to-one; wrap only what the core needs, or skip the port.

## Directory layout
```
shipping/
  core/        model.py, ports.py, use_cases.py      # no third-party imports
  adapters/
    inbound/   http.py, sqs_consumer.py, cli.py
    outbound/  dhl.py, fedex.py, sql_repo.py
  bootstrap.py                                       # wires adapters to ports
tests/fakes.py                                       # FakeCarrier, InMemoryRepo
```

## Example — Python
```python
# core/ports.py and core/use_cases.py
from dataclasses import dataclass
from decimal import Decimal
from typing import Protocol

@dataclass(frozen=True)
class Quote:
    carrier: str
    price: Decimal
    days: int

class CarrierClient(Protocol):            # outbound port, in the core's vocabulary
    name: str
    async def quote(self, weight_kg: float, dest: str) -> Quote: ...

@dataclass
class QuoteShipment:                      # inbound port / use case
    carriers: list[CarrierClient]
    async def cheapest(self, weight_kg: float, dest: str) -> Quote:
        quotes = [await c.quote(weight_kg, dest) for c in self.carriers]
        return min(quotes, key=lambda q: (q.price, q.days))

# adapters/outbound/dhl.py — translates vendor shapes into core types
import httpx

class DhlAdapter:
    name = "dhl"
    def __init__(self, client: httpx.AsyncClient) -> None:
        self.client = client
    async def quote(self, weight_kg: float, dest: str) -> Quote:
        r = await self.client.post("/rates", json={"wt": weight_kg, "cc": dest})
        r.raise_for_status()
        body = r.json()
        return Quote("dhl", Decimal(body["totalPrice"]["amount"]), body["transitDays"])

# tests/fakes.py
@dataclass
class FakeCarrier:
    name: str
    price: Decimal
    async def quote(self, weight_kg: float, dest: str) -> Quote:
        return Quote(self.name, self.price, 3)
```
`QuoteShipment([FakeCarrier("a", Decimal(5)), FakeCarrier("b", Decimal(4))])` tests the core with zero I/O. In real code, use `asyncio.gather` for concurrent quotes.

## Example — TypeScript
```ts
// core: port as an InjectionToken so Angular DI can swap adapters
import { InjectionToken, Injectable, inject } from '@angular/core';
import { HttpClient } from '@angular/common/http';
import { Observable, map } from 'rxjs';

export interface Quote { carrier: string; price: number; days: number }
export interface CarrierPort { quote(weightKg: number, dest: string): Observable<Quote> }
export const CARRIERS = new InjectionToken<CarrierPort[]>('CARRIERS');

// secondary adapter
@Injectable({ providedIn: 'root' })
export class DhlAdapter implements CarrierPort {
  private http = inject(HttpClient);
  quote(weightKg: number, dest: string) {
    return this.http.post<{ totalPrice: { amount: number }; transitDays: number }>(
      '/carriers/dhl/rates', { wt: weightKg, cc: dest },
    ).pipe(map(b => ({ carrier: 'dhl', price: b.totalPrice.amount, days: b.transitDays })));
  }
}

// app.config.ts wiring:  { provide: CARRIERS, useFactory: () => [inject(DhlAdapter)] }
// test wiring:           { provide: CARRIERS, useValue: [{ quote: () => of({ carrier: 'x', price: 1, days: 1 }) }] }
```

## In the wild
- FastAPI `Depends` + `dependency_overrides`, Angular `InjectionToken` providers, NestJS custom providers: the wiring mechanism.
- Spring's `@Repository` interfaces with multiple implementations; Python `typing.Protocol` for structural ports.
- Alistair Cockburn's original 2005 article; DDD's anti-corruption layer is a secondary adapter with translation.

## Trade-offs
| Pros | Cons |
|---|---|
| Core is testable with fakes, no network | Extra interface + adapter per external system |
| Vendors and transports swappable | Mapping code between vendor and core types |
| Entry points (HTTP, queue, CLI) share one core | Indirection makes "go to definition" land on a Protocol |
| Vendor models stay out of the domain | Teams over-port trivial dependencies (logging, time) |

## Pitfalls
- **Leaky ports:** a port returning `httpx.Response` or an ORM row is not a port. Return core types; raise core exceptions.
- **Port per adapter method:** design ports from the core's needs, not by mirroring an SDK.
- **Business logic in adapters:** adapters translate and transport only; a rule in `DhlAdapter` will be missing from `FedexAdapter`.
- **Composition scattered everywhere:** keep wiring in one composition root (`bootstrap.py`, `app.config.ts`).
- **Mocks instead of fakes:** a hexagon pays off when you write a reusable in-memory fake per port.

## Refactoring path
- **Introduce it:** pick the most painful external call; define a Protocol with the minimal method the core needs; move the existing call into an adapter implementing it; inject it; write a fake and move tests to it. Then do inbound: make the HTTP handler call a use-case function instead of holding logic.
- **Remove it / simplify:** inline ports that have one adapter and no fake; let handlers call a module function directly; keep the adapter only where vendor translation is real.

## Related patterns
- [Clean Architecture](../architectural/clean-architecture.md) — adds named rings (entities, use cases) on the same rule.
- [Onion](../architectural/onion.md) — concentric domain layers; see its Clean vs Hexagonal vs Onion comparison.
- [Adapter](../structural/adapter.md) — the object-level pattern each secondary adapter implements.
- [Dependency Injection](../behavioral/dependency-injection.md) — how adapters are plugged in.
- [Repository](../data-access/repository.md) — the classic persistence port.
- [Message Translator](../enterprise/message-translator.md) — the translation role inside an anti-corruption adapter.
