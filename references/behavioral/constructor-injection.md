# Constructor Injection

**Category:** Behavioral (injection) · **Also known as:** Ctor injection, Initializer injection · **Related:** Dependency Injection, Setter Injection, Interface Injection, Factory, Builder, Facade

## Intent
Pass every required dependency through the constructor (or `__init__`), so an object is fully usable the moment it exists and its dependencies are visible in its signature and immutable afterwards.

## Problem
- Objects that create their own collaborators can't be tested or reconfigured (see Dependency Injection).
- Objects that receive collaborators *later* (setters, `init()` methods, global lookups) can exist in a half-built state: `service.gateway` is `None` until someone remembers to set it, and the bug surfaces as `AttributeError` at 3 a.m.
- Smells: `def __init__(self): self.repo = None` followed by `set_repo()`; two-phase `init()`/`configure()` calls; methods that start with `assert self.client is not None`; hidden dependencies found only by reading the body.

Scenario: `InvoiceService` needs an invoice repository, a PDF renderer and an email sender. Every method needs all three. There is no valid InvoiceService without them.

## Solution and structure
- Declare dependencies as constructor parameters typed by abstraction (Protocol/interface) or concrete class.
- Store them in read-only fields (`Final`, `readonly`, frozen dataclass, `private readonly`).
- Don't do work in the constructor beyond assignment and cheap validation: no I/O, no network calls.
- The composition root (or DI container) calls the constructor with real implementations; tests call it with fakes.

```
InvoiceService(repo: InvoiceRepo, pdf: PdfRenderer, mail: Mailer)
       ▲                  ▲                ▲              ▲
       └──── composition root / container / test builds and passes them
```

Invariant: after `__init__` returns, the object is valid. That is the main advantage over the other injection forms.

## When to use
- The default for every *required* dependency. Most dependencies are required.
- Immutable, thread-safe services (no reassignment after construction).
- Whenever you want the type checker and the container to fail fast on missing wiring.

## When NOT to use
- The dependency is optional with a sensible default: still constructor injection, but with a default value (Null Object), or setter injection if it must change at runtime.
- Circular dependencies (A needs B, B needs A): constructor injection cannot build them. The real fix is to break the cycle (extract a third class, use events); setter/lazy injection is a workaround.
- Framework-instantiated objects you don't construct (some ORM entities, Angular components in old patterns, pytest test classes): use the framework's mechanism (`inject()`, fixtures).
- Values that vary per call (current user, request ID): pass them as method parameters, not constructor dependencies, unless the object is request-scoped.

## Example — Python
A frozen dataclass is the most compact form: the generated `__init__` *is* constructor injection, and `frozen=True` makes the dependencies immutable.

```python
from dataclasses import dataclass
from datetime import date
from typing import Protocol

class InvoiceRepo(Protocol):
    async def get(self, invoice_id: str) -> "Invoice": ...
    async def mark_sent(self, invoice_id: str, on: date) -> None: ...

class PdfRenderer(Protocol):
    def render(self, invoice: "Invoice") -> bytes: ...

class Mailer(Protocol):
    async def send(self, to: str, subject: str, attachment: bytes) -> None: ...

class Clock(Protocol):
    def today(self) -> date: ...

@dataclass(frozen=True, slots=True)
class InvoiceService:
    repo: InvoiceRepo
    pdf: PdfRenderer
    mail: Mailer
    clock: Clock

    async def send(self, invoice_id: str) -> None:
        invoice = await self.repo.get(invoice_id)
        await self.mail.send(invoice.customer_email, f"Invoice {invoice.number}",
                             self.pdf.render(invoice))
        await self.repo.mark_sent(invoice_id, on=self.clock.today())

# composition root
# service = InvoiceService(SqlInvoiceRepo(session), WeasyPdf(), SesMailer(ses), SystemClock())
# test
# service = InvoiceService(InMemoryInvoiceRepo([inv]), FakePdf(), RecordingMailer(), FixedClock(date(2026, 1, 31)))
```

`Invoice` is assumed to be a domain dataclass defined elsewhere. Use keyword arguments at the call site when there are more than two or three dependencies.

## Example — TypeScript
TypeScript parameter properties give one-line constructor injection in plain/Node code. In Angular, `inject()` in field initializers is the current idiom; Angular still supports constructor parameter injection, and both resolve at construction time, so the "valid after construction" guarantee holds.

```typescript
import { Injectable, inject } from '@angular/core';
import { HttpClient } from '@angular/common/http';

// Plain TS / Node (NestJS uses this form with decorators)
export interface InvoiceRepo { get(id: string): Promise<Invoice>; markSent(id: string, on: Date): Promise<void> }
export interface Mailer { send(to: string, subject: string, pdf: Uint8Array): Promise<void> }
export interface PdfRenderer { render(invoice: Invoice): Uint8Array }
export interface Invoice { id: string; number: string; customerEmail: string }

export class InvoiceService {
  constructor(
    private readonly repo: InvoiceRepo,
    private readonly pdf: PdfRenderer,
    private readonly mail: Mailer,
    private readonly now: () => Date = () => new Date(),   // optional dep with default
  ) {}

  async send(id: string): Promise<void> {
    const invoice = await this.repo.get(id);
    await this.mail.send(invoice.customerEmail, `Invoice ${invoice.number}`, this.pdf.render(invoice));
    await this.repo.markSent(id, this.now());
  }
}

// Angular: same guarantee via inject() in field initializers
@Injectable({ providedIn: 'root' })
export class InvoicesApi {
  private readonly http = inject(HttpClient);          // resolved during construction
  send(id: string) { return this.http.post<void>(`/api/invoices/${id}/send`, {}); }
}
```

## In the wild
- FastAPI: dependency functions that build a service with its constructor (`OrderService(repo, gateway)`); `Depends` on a class uses its `__init__` parameters as sub-dependencies.
- Angular: constructor parameters with type tokens, and `inject()` in field initializers (preferred in current docs); both run at instantiation.
- NestJS, Spring, .NET, Guice: constructor injection is the recommended default.
- Python stdlib style: `logging.Handler(level)`, `http.client.HTTPConnection(host, timeout=...)`; dataclasses and attrs generate injection-ready `__init__`s.
- pytest: fixtures construct the system under test with fakes, which is constructor injection in tests.

## Trade-offs
| Pros | Cons |
|---|---|
| Object is always fully initialized | Long parameter lists when a class does too much |
| Dependencies explicit and immutable | Cannot express circular dependencies |
| Fails fast on missing wiring (type checker, container) | Changing a dependency requires a new instance |
| Easiest form to test: just call the constructor | Subclasses must forward all base constructor args |

## Pitfalls
- **Constructor does I/O** (connects, fetches config). Construction becomes slow and fails unpredictably; tests can't build the object offline. Fix: construct cheap, connect in an explicit `start()`/lifespan or lazily.
- **Too many parameters.** Five or more is a design smell, not a DI problem. Fix: split responsibilities, or group cohesive deps behind a Facade; don't hide them in a "context" bag object.
- **Injecting the container** (`def __init__(self, container)`). That is Service Locator. Fix: inject the actual dependencies.
- **Mutable public fields** (`self.repo = repo` reassigned later). Fix: `frozen=True`, `Final`, `readonly`.
- **Positional-argument mix-ups** between deps of the same type. Fix: keyword-only parameters (`*,`) or distinct types/tokens.
- **Defaults that hit production resources** (`mail: Mailer = SesMailer()`) evaluated at import time. Fix: default to a Null Object or `None` + build in the composition root.
- **Angular: calling `inject()` later** (in `ngOnInit`, callbacks). Fix: inject in field initializers or the constructor only.

## Refactoring path
- **Introduce it:** (1) for each `self.x = Something()` or global lookup inside a class, add a constructor parameter defaulting to the old expression; (2) update the composition root/tests to pass the dependency; (3) remove the default once all callers pass it; (4) make fields read-only; (5) if setters existed for required deps, delete them.
- **Remove it / simplify:** if a dependency is only used by one method, consider passing it to that method instead; if a class is a bag of pure functions, turn it into module functions with no constructor at all.

## Related patterns
- [Dependency Injection](dependency-injection.md) — the overall principle, containers, scopes and composition root.
- [Setter Injection](setter-injection.md) — for optional or changeable dependencies; weaker initialization guarantee.
- [Interface Injection](interface-injection.md) — injector method defined by an interface; rarely needed.
- [Factory](../creational/factory.md) — inject a factory when the consumer must create short-lived instances itself.
- [Builder](../creational/builder.md) — when construction needs many optional parts, build stepwise, then construct.
- [Facade](../structural/facade.md) — reduces constructor over-injection by grouping a subsystem.
