# Service-Oriented Architecture (SOA)

**Category:** Architectural · **Also known as:** SOA, enterprise services · **Related:** Microservices, ESB, Broker, Message Translator, Facade, Client-Server

## Intent
Expose business capabilities of an enterprise as reusable, network-accessible **services** with formal, shared contracts, so that many applications across the organization can compose them instead of re-implementing them. Integration typically flows through shared infrastructure such as an Enterprise Service Bus (ESB) and service registry.

## Problem
- Five departmental applications each keep their own customer master data and pricing logic; they disagree.
- Integrations are point-to-point (N×(N−1) links), each with its own format and error handling.
- Legacy systems (mainframe, ERP, CRM) must be reused by new channels without rewriting them.
- Scenario: an insurer with a COBOL policy system, SAP billing and Salesforce CRM wants a new customer portal and partner API that reuse "get policy", "create quote" and "bill customer" without each project wiring three systems again.

## Solution and structure
```
   Portal     Partner API     Call-center app          (consumers / composite apps)
      │            │                │
      └────────────┴──── ESB / integration layer ───────┐  routing, transformation,
                         (registry, security, orchestration) protocol mediation
      ┌──────────────────┬──────────────────┬───────────┘
 PolicyService      BillingService     CustomerService      (enterprise services, contracts: WSDL/OpenAPI)
   (mainframe)         (SAP)              (CRM)
```
- **Service contract:** formal interface and canonical data model (historically WSDL/XSD + SOAP; today often OpenAPI/AsyncAPI).
- **Service provider/consumer**, **service registry** (discovery, versioning), **governance** (standards, reuse, security policies).
- **ESB/integration layer:** routing, message transformation, protocol bridging, orchestration (BPEL/BPMN).
- Services are typically coarse-grained, enterprise-wide, and can share databases with the legacy system they wrap.

## When to use
- Large enterprises integrating many existing (often legacy, vendor) systems.
- A canonical enterprise capability must be reused by many channels (customer master, pricing).
- Strong central governance and compliance requirements over integrations.

## When NOT to use
- Greenfield product built by autonomous teams: microservices or a modular monolith fit better.
- Small organizations: an ESB and registry are heavy infrastructure for a few integrations; direct HTTP APIs suffice.
- Smell: business logic accumulating inside the ESB ("smart pipes"), making it a bottleneck and a single point of failure.

## Example — Python
```python
# A service facade wrapping a legacy SOAP system behind a canonical contract (zeep is a SOAP client).
from dataclasses import dataclass
from datetime import date
from decimal import Decimal
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel
from zeep import Client
from zeep.exceptions import Fault

class Policy(BaseModel):                          # canonical enterprise model
    policy_id: str
    holder_name: str
    premium: Decimal
    valid_until: date

@dataclass
class LegacyPolicyAdapter:                        # translation to/from the mainframe schema
    client: Client
    def get(self, policy_id: str) -> Policy:
        raw = self.client.service.GetPolicyDetails(PolNo=policy_id.zfill(10))
        return Policy(
            policy_id=raw.PolNo.lstrip("0"),
            holder_name=f"{raw.HldrFirst} {raw.HldrLast}".title(),
            premium=Decimal(raw.PremAmt) / 100,   # legacy stores cents as string
            valid_until=raw.ExpDt,
        )

app = FastAPI(title="PolicyService", version="2.1.0")   # contract published as OpenAPI
adapter = LegacyPolicyAdapter(Client("http://mainframe-gw/policy?wsdl"))

@app.get("/v2/policies/{policy_id}", response_model=Policy)
def get_policy(policy_id: str) -> Policy:
    try:
        return adapter.get(policy_id)
    except Fault as f:
        raise HTTPException(404 if "NOTFOUND" in str(f) else 502, "policy backend error")
```

## Example — TypeScript
```ts
// Consumer side: a composite app orchestrating enterprise services via a generated typed client.
// (Types would normally be generated from the services' OpenAPI contracts, e.g. openapi-typescript.)
interface Policy { policyId: string; holderName: string; premium: number; validUntil: string }
interface Invoice { invoiceId: string; amount: number; dueDate: string }

const ESB = 'https://esb.internal.example.com';

async function call<T>(path: string, init?: RequestInit): Promise<T> {
  const res = await fetch(`${ESB}${path}`, {
    ...init,
    headers: { 'Content-Type': 'application/json', 'X-Consumer-Id': 'customer-portal', ...init?.headers },
    signal: AbortSignal.timeout(3000),
  });
  if (!res.ok) throw new Error(`${path} failed: ${res.status}`);
  return res.json() as Promise<T>;
}

export async function renewPolicy(policyId: string): Promise<{ policy: Policy; invoice: Invoice }> {
  const policy = await call<Policy>(`/policy/v2/policies/${policyId}`);
  const invoice = await call<Invoice>('/billing/v1/invoices', {
    method: 'POST',
    body: JSON.stringify({ customerRef: policy.policyId, amount: policy.premium }),
  });
  return { policy, invoice };
}
```

## In the wild
- MuleSoft, IBM Integration Bus/App Connect, Oracle Service Bus, WSO2, TIBCO, Apache Camel/ServiceMix.
- SOAP/WS-* stacks, UDDI registries (largely obsolete), BPEL engines; modern equivalents: API management platforms (Apigee, Azure API Management) and iPaaS.
- AWS's 2002 "API mandate" is often cited as SOA done with HTTP services instead of an ESB.

## Trade-offs
| Pros | Cons |
|---|---|
| Reuse of enterprise capabilities across channels | Central ESB becomes a bottleneck and single point of failure |
| Wraps legacy systems without rewriting | Heavy governance slows change |
| Canonical model reduces point-to-point integrations | Canonical data models become bloated compromises |
| Centralized security, monitoring, policy | Vendor lock-in to integration platforms |

## Pitfalls
- **Business logic in the bus:** transformations grow into rules nobody owns. Keep the bus to routing/translation; logic in services.
- **Over-generic canonical model:** one `Customer` with 300 optional fields. Use per-context models with translators at the edges.
- **Chatty fine-grained services over SOAP:** latency explodes. Design coarse-grained operations.
- **Versioning by breaking change:** publish versioned contracts (`/v2/...`) and support old versions for a deprecation period.
- **Reuse as a goal:** services nobody reuses still pay the governance cost.

## Refactoring path
- **Introduce it:** from point-to-point integrations, define a contract for the most duplicated capability, wrap the system of record with a service facade, route consumers through it, then retire direct links.
- **Remove it / simplify:** move toward microservices by pushing ESB logic back into owning services ("smart endpoints, dumb pipes"), replacing the ESB with an API gateway plus a message broker, and splitting services by bounded context.

## Related patterns
- [Microservices](../architectural/microservices.md) — finer-grained, team-owned, decentralized data and governance.
- [ESB](../enterprise/esb.md) — the integration backbone typical of SOA.
- [Broker](../architectural/broker.md) — the general intermediary style an ESB implements.
- [Message Translator](../enterprise/message-translator.md) — converting legacy formats to the canonical model.
- [Facade](../structural/facade.md) — service facades in front of legacy systems.
