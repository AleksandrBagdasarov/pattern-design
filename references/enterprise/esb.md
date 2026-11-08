# ESB (Enterprise Service Bus)

**Category:** Enterprise integration · **Also known as:** Integration bus, integration middleware, iPaaS (cloud form) · **Related:** Message Broker, SOA, Message Translator, Content-Based Router, Aggregator, API Gateway

## Intent
Provide one central integration backbone that connects heterogeneous applications through adapters, routing, transformation and protocol mediation, so applications do not integrate point-to-point.

## Problem
- An enterprise has an ERP (SOAP), a CRM (REST), a mainframe (fixed-width files over MQ) and a warehouse system (FTP CSVs); each pair is wired directly with custom scripts.
- Every format change in one system breaks several integrations owned by different teams.
- Cross-cutting needs (auditing, security, retries, monitoring) are re-implemented per connection.
- Scenario: a new e-commerce site must send orders to ERP, CRM and warehouse, each with its own protocol and schema.

## Solution and structure
A central bus hosts:
- **Adapters/connectors** per system and protocol (SOAP, REST, JMS, FTP, JDBC, SAP).
- **Routing** (content-based, itinerary) and **transformation** (Message Translator, XSLT/DataWeave/JSONata) to and from a canonical data model.
- **Mediation** of protocols, sync/async, security, retries, auditing and monitoring.
- Sometimes **orchestration** of multi-step processes (BPEL, flows).

```
 Web shop ─┐                  ┌──────────── ESB ─────────────┐          ┌─> ERP (SOAP)
 CRM ──────┼──> adapters ──>  │ route ─ transform ─ enrich   │ ──> ─────┼─> Warehouse (FTP CSV)
 Mainframe ┘                  │ canonical model, audit, retry│          └─> CRM (REST)
                              └──────────────────────────────┘
```
Modern guidance ("smart endpoints, dumb pipes"): keep routing and translation, but keep **business logic in the services**, not in the bus.

## When to use
- Integrating many packaged/legacy systems you cannot change (SAP, mainframes, COTS) with differing protocols.
- Central governance, auditing and monitoring of integrations is a hard requirement (banking, insurance, government).
- An integration team owns the connections as a product; a managed iPaaS (MuleSoft, Boomi, Azure Logic Apps) is an option.

## When NOT to use
- Greenfield services you control: use a message broker (pub-sub, queues) plus per-service translators; keep logic in services.
- A handful of integrations: a few small adapters or a lightweight integration library (Apache Camel, Spring Integration) inside a service.
- To orchestrate core business processes: logic in the bus becomes an unversioned, untestable "god middleware"; use services, sagas or a workflow engine (Temporal, Step Functions).

## Example — Python
There is no idiomatic Python ESB product; the core of one is an adapter → translate → route pipeline. This sketch shows the shape, for a small in-house integration hub.

```python
from collections.abc import Callable
from dataclasses import dataclass
import csv, io

@dataclass(frozen=True)
class CanonicalOrder:                        # canonical data model
    order_id: str
    customer_email: str
    lines: list[tuple[str, int]]

Translator = Callable[[CanonicalOrder], bytes]
Sender = Callable[[bytes], None]

def to_erp_xml(o: CanonicalOrder) -> bytes:
    items = "".join(f'<item sku="{s}" qty="{q}"/>' for s, q in o.lines)
    return f'<order id="{o.order_id}">{items}</order>'.encode()

def to_warehouse_csv(o: CanonicalOrder) -> bytes:
    buf = io.StringIO()
    csv.writer(buf).writerows([o.order_id, s, q] for s, q in o.lines)
    return buf.getvalue().encode()

@dataclass
class Route:
    applies: Callable[[CanonicalOrder], bool]
    translate: Translator
    send: Sender

def dispatch(order: CanonicalOrder, routes: list[Route], audit: Callable[[str], None]) -> None:
    for r in routes:
        if r.applies(order):
            r.send(r.translate(order))
            audit(f"{order.order_id} -> {r.send.__name__}")
```
(Escape XML properly with `xml.sax.saxutils` or an XML library in real code.) For anything beyond this, use an integration framework rather than growing it.

## Example — TypeScript
The cloud-native alternative to an ESB: EventBridge rules route by content, each target does its own translation (input transformers or a small Lambda). Shown with AWS SDK v3.

```ts
import { EventBridgeClient, PutRuleCommand, PutTargetsCommand } from '@aws-sdk/client-eventbridge';

const eb = new EventBridgeClient({});

export async function routeLargeOrdersToErp(erpAdapterLambdaArn: string): Promise<void> {
  await eb.send(new PutRuleCommand({
    Name: 'orders-large-to-erp',
    EventBusName: 'integration',
    EventPattern: JSON.stringify({
      source: ['shop.orders'],
      'detail-type': ['OrderPlaced'],
      detail: { totalCents: [{ numeric: ['>=', 100000] }] },   // content-based routing
    }),
  }));
  await eb.send(new PutTargetsCommand({
    Rule: 'orders-large-to-erp',
    EventBusName: 'integration',
    Targets: [{ Id: 'erp-adapter', Arn: erpAdapterLambdaArn }], // adapter translates to SOAP
  }));
}
```
Typically defined in IaC (CDK/Terraform) rather than imperative calls; the routing and per-target adapter split is the point.

## In the wild
- Products: MuleSoft Anypoint, IBM App Connect/Integration Bus, Oracle Service Bus, TIBCO BusinessWorks, WSO2, Red Hat Fuse; Microsoft BizTalk (end of mainstream support — check current status).
- Frameworks implementing the EIP catalog: Apache Camel, Spring Integration.
- Cloud/iPaaS: Azure Logic Apps + Service Bus, AWS EventBridge + Step Functions, Boomi, Workato.

## Trade-offs
| Pros | Cons |
|---|---|
| One place for adapters, transformation, audit, security | Central bottleneck and single point of failure |
| Point-to-point integrations replaced by hub-and-spoke | Integration team becomes a delivery bottleneck |
| Protocol mediation for legacy systems | Business logic leaks into the bus, hard to test and version |
| Centralized monitoring and governance | Expensive licenses and specialist skills |
| Canonical model reduces translators (n instead of n²) | Canonical model becomes a lowest-common-denominator compromise |

## Pitfalls
- **Business rules in bus flows.** Fix: the bus routes and translates; services decide.
- **Huge canonical model** everyone must agree on. Fix: smaller per-domain contracts; translate at bounded-context edges.
- **Synchronous chains through the bus** for user requests: latency and coupled failure. Fix: async where possible; timeouts.
- **No versioning or tests for flows.** Fix: flows in source control, CI, contract tests.
- **Using an ESB as a microservices backbone.** Fix: broker + smart endpoints.

## Refactoring path
- **Introduce it:** inventory point-to-point integrations; define canonical messages for the busiest flows; move adapters and translators into an integration layer one flow at a time; add central audit and monitoring.
- **Remove it / simplify:** strangle the bus: for each flow, move business logic into the owning service, replace bus routing with broker topics and per-consumer translators, then retire the flow; keep the bus only for legacy adapters.

## Related patterns
- [Message Broker](../enterprise/message-broker.md) — the "dumb pipe" that replaces an ESB in modern designs.
- [Service-Oriented (SOA)](../architectural/soa.md) — the architecture ESBs were built to support.
- [Message Translator](../enterprise/message-translator.md) — the transformation step inside the bus.
- [Aggregator](../enterprise/aggregator.md) / [Splitter](../enterprise/splitter.md) — common bus-hosted routing steps.
- [Content Filter](../enterprise/content-filter.md) — trimming messages per target system.
- [Microservices](../architectural/microservices.md) — prefers decentralized smart endpoints over a central bus.
