# Microservices

**Category:** Architectural · **Also known as:** microservice architecture · **Related:** SOA, Event-Driven, Saga, Circuit Breaker, API Gateway (Single Access Point), Sidecar, CQRS

## Intent
Build a system as a set of small, independently deployable services, each owning one **bounded context** and its own data, communicating over the network via APIs or events, so that teams can develop, deploy and scale their parts independently.

## Problem
- 40 engineers in one repo and one deployable; a release train every two weeks blocked by any failing module.
- One component needs 50× the compute of the rest (image processing, search), but scaling means scaling everything.
- Different parts need different tech (Python ML scoring next to a Node BFF).
- Scenario: an e-commerce monolith where catalog, checkout, payments and recommendations are owned by four teams who constantly break each other's code and wait on a shared deploy.

## Solution and structure
```
   clients ─► API gateway / BFF ─┬─► catalog-svc ──► catalog DB
                                 ├─► orders-svc  ──► orders DB ──outbox──► broker ──► shipping-svc
                                 └─► payments-svc ─► payments DB                  └─► email-svc
```
- **Service per bounded context**, owned by one team; **database per service** (no shared tables).
- **Communication:** synchronous (HTTP/gRPC) for queries that need an answer now; asynchronous events for state propagation.
- **Platform:** CI/CD per service, containers/orchestration, service discovery, centralized logs, metrics, distributed tracing, config and secrets.
- **Resilience:** timeouts, retries with backoff, circuit breakers, bulkheads.
- **Data consistency:** sagas and eventual consistency instead of distributed transactions.

## When to use
- Multiple teams blocked by a shared deployable (Conway's law pressure).
- Clear, stable domain boundaries already proven inside a modular monolith.
- Genuinely different scaling, availability or technology needs per part.
- The organization can afford platform engineering and on-call per service.

## When NOT to use
- Startups and new products whose boundaries are still moving: start with a **modular monolith** (packages per bounded context, enforced imports), split later.
- One small team: operational cost dominates.
- Operations that need strong cross-entity consistency on every request.
- Smell: "distributed monolith" — services that must deploy together, share a DB, or call each other synchronously in long chains.

## Directory layout
```
services/
  orders/      app/ (FastAPI), migrations/, Dockerfile, openapi.yaml, tests/
  payments/    app/, migrations/, Dockerfile, tests/
  web-bff/     src/ (NestJS or Express), Dockerfile
contracts/     events/*.json (JSON Schema/Avro), shared OpenAPI specs
deploy/        helm/ or terraform/ per service
```
Separate repos or a monorepo both work; independence comes from separate pipelines and data, not repos.

## Example — Python
```python
# orders-svc: owns its DB; calls payments synchronously with timeout + retry; publishes an event via outbox.
import httpx
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel
from tenacity import RetryError, retry, stop_after_attempt, wait_exponential_jitter, retry_if_exception_type

app = FastAPI()
payments = httpx.AsyncClient(base_url="http://payments-svc", timeout=httpx.Timeout(2.0))

class PlaceOrder(BaseModel):
    order_id: str
    customer_id: str
    amount_cents: int

@retry(stop=stop_after_attempt(3), wait=wait_exponential_jitter(0.1, 1.0),
       retry=retry_if_exception_type(httpx.TransportError))
async def authorize(cmd: PlaceOrder) -> str:
    r = await payments.post("/authorizations", json=cmd.model_dump(),
                            headers={"Idempotency-Key": cmd.order_id})   # safe to retry
    r.raise_for_status()
    return r.json()["authorization_id"]

@app.post("/orders", status_code=201)
async def place_order(cmd: PlaceOrder) -> dict[str, str]:
    try:
        auth_id = await authorize(cmd)
    except (httpx.HTTPError, RetryError) as exc:       # RetryError after the last failed attempt
        raise HTTPException(503, "payments unavailable") from exc
    # in one local transaction: insert order + insert outbox row {"type": "OrderPlaced", ...}
    # a relay process publishes outbox rows to the broker
    return {"order_id": cmd.order_id, "authorization_id": auth_id}
```

## Example — TypeScript
```ts
// web-bff (NestJS): aggregates two services for the Angular client; degrades if one is down.
import { Controller, Get, Param } from '@nestjs/common';

type Product = { id: string; name: string; price: number };
type Stock = { id: string; available: number };

async function getJson<T>(url: string, ms = 800): Promise<T> {
  const res = await fetch(url, { signal: AbortSignal.timeout(ms) });
  if (!res.ok) throw new Error(`${url} -> ${res.status}`);
  return res.json() as Promise<T>;
}

@Controller('products')
export class ProductPageController {
  @Get(':id')
  async page(@Param('id') id: string) {
    const [product, stock] = await Promise.allSettled([
      getJson<Product>(`http://catalog-svc/products/${id}`),
      getJson<Stock>(`http://inventory-svc/stock/${id}`),
    ]);
    if (product.status === 'rejected') throw product.reason;          // catalog is essential
    return {
      ...product.value,
      available: stock.status === 'fulfilled' ? stock.value.available : null,  // optional: degrade
    };
  }
}
```

## In the wild
- Kubernetes/ECS, service meshes (Istio, Linkerd), API gateways (Kong, AWS API Gateway), OpenTelemetry for tracing.
- Netflix, Amazon, Uber (and Uber's later move to "domain-oriented microservices" to tame thousands of services).
- Several companies (e.g. Amazon Prime Video's monitoring team, publicly in 2023) moved parts back to monoliths for cost; evaluate per workload.

## Trade-offs
| Pros | Cons |
|---|---|
| Independent deploys and team autonomy | Network failures, latency, partial outages become normal |
| Scale and choose tech per service | Distributed data: no joins, sagas, eventual consistency |
| Fault isolation (if resilience is built in) | Heavy platform and observability investment |
| Smaller codebases easier to understand | Cross-service changes need versioned contracts and coordination |
| | Local development and end-to-end testing are harder |

## Pitfalls
- **Shared database:** couples schemas and deploys. One owner per table; others use APIs/events.
- **Splitting by technical layer** (a "DB service", an "auth logic service"): split by business capability.
- **Synchronous call chains** A→B→C→D: latency and availability multiply. Use async events, caching, or merge services.
- **No idempotency on retries:** double charges. Send idempotency keys; consumers dedupe.
- **Too small ("nanoservices"):** every feature touches five services. Merge chatty services.
- **Missing correlation ids and tracing:** incidents become guesswork. Propagate trace context (W3C `traceparent`) from day one.

## Refactoring path
- **Introduce it:** first make a modular monolith with enforced boundaries and per-module schemas; pick the module with the clearest boundary and highest independent change rate; put it behind an interface; extract it with the **Strangler Fig** approach (route traffic gradually through a gateway), with its own DB and an event feed for data it needs.
- **Remove it / simplify:** merge services that always change and deploy together into one deployable with internal modules; replace sync chatter with in-process calls; keep separate only what has a different scale or owner.

## Related patterns
- [SOA](../architectural/soa.md) — predecessor: shared enterprise contracts and an ESB vs smart endpoints, dumb pipes.
- [Saga](../cloud/saga.md) — business transactions across services.
- [Circuit Breaker](../cloud/circuit-breaker.md) — isolates failing downstream services.
- [Event-Driven](../architectural/event-driven.md) — asynchronous state propagation between services.
- [Single Access Point](../security/single-access-point.md) — API gateway in front of services.
- [Sidecar](../emergent/sidecar.md) — infrastructure concerns (mTLS, retries, telemetry) beside each service.
