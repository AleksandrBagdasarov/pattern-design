# Abstract Factory

**Category:** Creational · **Also known as:** Kit · **Related:** Factory, Builder, Bridge, Adapter, Dependency Injection, Hexagonal Architecture

## Intent
Provide one interface for creating a *family* of related objects, so that the objects a client gets always belong together, without the client naming any concrete class.

## Problem
- Several objects must be chosen *together*: if storage is S3 then the queue is SQS and the secrets come from Secrets Manager; on GCP, all three change.
- Mixing members of different families is a real bug (an S3 blob store with a GCP Pub/Sub queue that cannot read the bucket's events).
- Scenario: a SaaS backend deploys to AWS for most customers and to Azure for one enterprise customer. Code is littered with `if CLOUD == "aws"` checks in five places, and a new hire wired `AzureBlobStore` with `SqsQueue` in a background worker.
- Smell: parallel `if/elif` ladders on the same key in several factories.

## Solution and structure
- **AbstractFactory**: an interface with one creation method per product (`blob_store()`, `queue()`, `secrets()`).
- **ConcreteFactory**: one per family (`AwsKit`, `AzureKit`), returning matching products.
- **AbstractProducts**: interfaces for each product kind.
- **Client**: receives one factory and uses only the abstract interfaces.

```
              CloudKit (interface)
      blob_store()   queue()   secrets()
          ▲                         ▲
   AwsKit │                         │ AzureKit
  S3Store, SqsQueue, AwsSecrets     AzureBlobStore, ServiceBusQueue, KeyVault
```

The family choice is made once (at startup, from config), and the factory is injected. Consistency follows from the structure.

## When to use
- Two or more product kinds vary together along one axis (cloud provider, UI theme, database vendor, test vs production).
- Mixing families must be impossible or at least hard.
- You want to switch the whole family with one config value or one DI binding.

## When NOT to use
- Only one product varies. Use a plain [Factory](factory.md).
- Only one family exists and a second is speculative. Construct directly; add the kit when the second family is real.
- The "family" is just configuration values (URLs, regions). Pass a settings object.
- Your DI container can bind a whole module of providers per environment (Angular `provideX()` functions, NestJS dynamic modules). That *is* the abstract factory; don't add a second layer on top.
- Adding a new product kind is frequent: every concrete factory must change, which is the pattern's main cost.

## Example — Python
A `Protocol` for the kit and frozen dataclasses for concrete kits. In Python a module exposing the same functions per family can also serve as the kit.

```python
from dataclasses import dataclass
from typing import Protocol

class BlobStore(Protocol):
    def put(self, key: str, data: bytes) -> str: ...

class JobQueue(Protocol):
    def send(self, body: dict[str, str]) -> None: ...

class CloudKit(Protocol):
    def blob_store(self) -> BlobStore: ...
    def job_queue(self) -> JobQueue: ...

@dataclass(frozen=True)
class AwsKit:
    bucket: str
    queue_url: str
    def blob_store(self) -> BlobStore: return S3BlobStore(self.bucket)
    def job_queue(self) -> JobQueue: return SqsJobQueue(self.queue_url)

@dataclass(frozen=True)
class LocalKit:                      # family used in dev and tests
    root: str = "/tmp/blobs"
    def blob_store(self) -> BlobStore: return FileBlobStore(self.root)
    def job_queue(self) -> JobQueue: return InMemoryQueue()

def kit_from_env(env: str) -> CloudKit:
    match env:
        case "aws":   return AwsKit(bucket="invoices-prod", queue_url="https://sqs/...")
        case "local": return LocalKit()
        case _:       raise ValueError(f"unknown cloud env {env!r}")

def export_invoice(kit: CloudKit, invoice_id: str, pdf: bytes) -> None:
    url = kit.blob_store().put(f"invoices/{invoice_id}.pdf", pdf)
    kit.job_queue().send({"type": "invoice.exported", "url": url})
# S3BlobStore, SqsJobQueue, FileBlobStore, InMemoryQueue implement the Protocols.
```

## Example — TypeScript
In Angular, an environment-specific `provideX()` function that registers several matching providers acts as the abstract factory.

```typescript
import { InjectionToken, Provider, makeEnvironmentProviders, EnvironmentProviders } from '@angular/core';

export interface AnalyticsClient { track(event: string, props?: Record<string, unknown>): void; }
export interface ErrorReporter { report(err: unknown): void; }

export const ANALYTICS = new InjectionToken<AnalyticsClient>('ANALYTICS');
export const ERRORS = new InjectionToken<ErrorReporter>('ERRORS');

class SegmentAnalytics implements AnalyticsClient { track(e: string) { /* segment SDK */ } }
class SentryReporter implements ErrorReporter { report(err: unknown) { /* Sentry.captureException */ } }
class ConsoleAnalytics implements AnalyticsClient { track(e: string, p?: object) { console.debug(e, p); } }
class ConsoleReporter implements ErrorReporter { report(err: unknown) { console.error(err); } }

/** One call picks the whole family: products can't be mixed by accident. */
export function provideTelemetry(kind: 'prod' | 'dev'): EnvironmentProviders {
  const family: Provider[] = kind === 'prod'
    ? [{ provide: ANALYTICS, useClass: SegmentAnalytics }, { provide: ERRORS, useClass: SentryReporter }]
    : [{ provide: ANALYTICS, useClass: ConsoleAnalytics }, { provide: ERRORS, useClass: ConsoleReporter }];
  return makeEnvironmentProviders(family);
}

// app.config.ts: providers: [provideTelemetry(environment.production ? 'prod' : 'dev')]
```

## In the wild
- SQLAlchemy dialects: one dialect supplies a matching compiler, type mapper and DBAPI adapter.
- Python DB-API modules (`sqlite3`, `psycopg`): each module is a kit of connection, cursor and exception classes.
- Angular `provideHttpClient(withFetch())`, `provideRouter(...)`: provider bundles that wire matching collaborators.
- NestJS dynamic modules (`TypeOrmModule.forRoot`), Terraform/Pulumi provider packages, AWS SDK clients sharing one credential/region config.
- GUI toolkits: Qt styles, Material vs Cupertino widgets in Flutter.

## Trade-offs
| Pros | Cons |
|---|---|
| Guarantees products from one family are used together | Adding a new product kind changes every concrete factory |
| Switch an entire family with one binding | More types and indirection than direct construction |
| Clients depend only on interfaces | Tempting to build for a second family that never comes |
| Natural seam for test doubles (a `LocalKit`) | Lowest-common-denominator interfaces can hide useful vendor features |

## Pitfalls
- **Kit grows into a god-factory** creating unrelated things. Fix: one kit per cohesive family; split by concern.
- **Clients downcast** (`isinstance(store, S3BlobStore)`) to reach vendor features. Fix: add the capability to the interface or use a separate port.
- **Choosing the family in several places.** Fix: decide once at the composition root and inject the kit.
- **Creating a new product on every call** when products are expensive clients. Fix: cache products inside the concrete kit (`functools.cached_property`).
- **Leaky abstractions**: an interface shaped after one vendor (S3 multipart details) that the other cannot implement. Design it from the use case, not the SDK.

## Refactoring path
- **Introduce it:** find parallel conditionals on the same discriminator in several factories. Define product interfaces first; group the matching constructors into one class per family; replace the conditionals with a single `kit_from_env()` at startup; inject the kit.
- **Remove it / simplify:** if one family remains, inline its products into DI bindings and delete the kit interface. If only one product varies, collapse to a Simple Factory.

## Related patterns
- [Factory](factory.md) — one product chosen at a time; Abstract Factory is a set of factory methods.
- [Builder](builder.md) — assembles one complex object step by step; often a product of a kit.
- [Bridge](../structural/bridge.md) — an abstract factory can supply the implementor side of a Bridge.
- [Adapter](../structural/adapter.md) — concrete products are often adapters over vendor SDKs.
- [Dependency Injection](../behavioral/dependency-injection.md) — provider bundles per environment are the DI form of this pattern.
- [Hexagonal](../architectural/hexagonal.md) — a kit is a convenient way to supply all adapters for one environment.
