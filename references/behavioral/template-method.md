# Template Method

**Category:** Behavioral · **Also known as:** Hook methods, Hollywood principle ("don't call us, we'll call you") · **Related:** Strategy, Factory, Decorator, Chain of Responsibility, Pipes and Filters

## Intent
Define the skeleton of an algorithm in a base class method and let subclasses override specific steps without changing the overall order.

## Problem
- Several classes implement the same multi-step process with the same order and only one or two steps differ, and the shared steps are copy-pasted.
- Bug fixes to the shared part (logging, error handling, retries, transactions) get applied to one copy and forgotten in the others.
- Scenario: CSV, Excel and JSON importers each `open → parse rows → validate → upsert → report`. Only parsing differs, but each importer has its own slightly different validation and reporting, and the JSON importer forgot to wrap the upsert in a transaction.

## Solution and structure
- **Abstract class** has a final `run()` (the template method) that calls steps in a fixed order.
- **Primitive operations** are abstract steps each subclass must implement.
- **Hooks** are optional steps with a default (often a no-op) that subclasses may override.
- **Concrete classes** fill in the steps; they do not call `run()`'s internals themselves.

```
BaseImporter.run():            CsvImporter    JsonImporter
   rows = self.parse(src)  ──▶  parse()        parse()
   self.validate(rows)          (default)      validate() override
   self.upsert(rows)            (shared)       (shared)
   self.after_import()          hook           hook
```

In Python and TypeScript the same result is often achieved with **composition**: a plain function that takes the variable steps as callables. That avoids inheritance and is usually preferable unless the steps share a lot of state.

## When to use
- A stable algorithm with a few well-defined variation points, and those variations come in a known family (importers, exporters, report generators, test base classes).
- Frameworks that call user code at fixed lifecycle points (setup, handle, teardown).
- You want to enforce invariants the subclass cannot skip (transaction, audit log, timing metric).

## When NOT to use
- Only one step varies. Pass a function (Strategy) instead of subclassing.
- Variations need to be mixed and matched (CSV parsing + strict validation + no reporting). Inheritance combinations explode; compose strategies.
- Deep hierarchies (`Base → AbstractCsv → SemicolonCsv → LegacySemicolonCsv`). Flatten into composition.
- The steps are independent stages over data. A pipeline (list of functions) reads better.

## Example — Python
Class form with `abc`; note the `@final` template and the hook.

```python
from abc import ABC, abstractmethod
from collections.abc import Iterable
from dataclasses import dataclass
from contextlib import AbstractContextManager
from typing import Protocol, final
import csv, io, json

@dataclass(frozen=True)
class Customer:
    email: str
    name: str

class CustomerRepo(Protocol):
    def transaction(self) -> AbstractContextManager[None]: ...
    def upsert_many(self, rows: list[Customer]) -> None: ...

class CustomerImporter(ABC):
    def __init__(self, repo: CustomerRepo) -> None:
        self.repo = repo

    @final
    def run(self, raw: str) -> int:
        rows = list(self.parse(raw))
        valid = [r for r in rows if self.is_valid(r)]
        with self.repo.transaction():
            self.repo.upsert_many(valid)
        self.after_import(len(rows), len(valid))
        return len(valid)

    @abstractmethod
    def parse(self, raw: str) -> Iterable[Customer]: ...

    def is_valid(self, c: Customer) -> bool:              # overridable default
        return "@" in c.email

    def after_import(self, seen: int, imported: int) -> None:  # hook, no-op
        pass

class CsvCustomerImporter(CustomerImporter):
    def parse(self, raw: str) -> Iterable[Customer]:
        for row in csv.DictReader(io.StringIO(raw)):
            yield Customer(row["email"].strip().lower(), row["name"])

class JsonCustomerImporter(CustomerImporter):
    def parse(self, raw: str) -> Iterable[Customer]:
        return [Customer(d["email"].lower(), d["name"]) for d in json.loads(raw)]
```

The composition alternative: `def import_customers(raw, repo, parse: Callable[[str], Iterable[Customer]], is_valid=default_is_valid) -> int`. Prefer it when there is one varying step.

## Example — TypeScript
A NestJS/Node style base job with a fixed lifecycle.

```typescript
export abstract class ScheduledJob<T> {
  constructor(protected readonly name: string) {}

  /** Template method: do not override. */
  async run(): Promise<void> {
    const started = performance.now();
    try {
      const items = await this.fetch();
      for (const item of items) await this.process(item);
      await this.onSuccess(items.length);
    } catch (err) {
      console.error(`[${this.name}] failed`, err);
      throw err;
    } finally {
      console.info(`[${this.name}] took ${Math.round(performance.now() - started)}ms`);
    }
  }

  protected abstract fetch(): Promise<T[]>;
  protected abstract process(item: T): Promise<void>;
  protected async onSuccess(_count: number): Promise<void> {}   // hook
}

interface Invoice { id: string; email: string }

export class OverdueInvoiceReminder extends ScheduledJob<Invoice> {
  constructor(private readonly api: { overdue(): Promise<Invoice[]>; remind(i: Invoice): Promise<void> }) {
    super('overdue-invoice-reminder');
  }
  protected fetch(): Promise<Invoice[]> { return this.api.overdue(); }
  protected process(item: Invoice): Promise<void> { return this.api.remind(item); }
}
```

TypeScript has no `final` keyword for methods; document it, or make `run` a standalone function that receives the steps.

## In the wild
- `unittest.TestCase` (`setUp`/`tearDown` around `test_*`), `socketserver.BaseRequestHandler.handle`, `threading.Thread.run`, `http.server.BaseHTTPRequestHandler.do_GET`.
- `logging.Handler.emit`, `json.JSONEncoder.default`, `collections.abc.Mapping` (implement 3 methods, get the rest).
- Angular lifecycle hooks (`ngOnInit`, `ngOnDestroy`) are framework-called hooks of the same spirit.
- Django class-based views (`get_queryset`, `get_context_data`), DRF `APIView`.
- Pydantic `model_post_init`, SQLAlchemy `TypeDecorator.process_bind_param`.

## Trade-offs
| Pros | Cons |
|---|---|
| Shared steps and invariants live in one place | Inheritance couples subclasses to the base class's internals |
| Subclasses cannot reorder or skip the skeleton | Hard to combine variations (no mixing of two subclasses' steps) |
| Clear, documented extension points | Readers jump between base and subclass to follow the flow |
| Natural fit for framework lifecycle hooks | Fragile base class: changes to the skeleton ripple to every subclass |

## Pitfalls
- **Subclasses overriding the template method itself.** Fix: `@typing.final` (checked by type checkers), or composition.
- **Too many hooks.** Ten optional hooks means nobody knows which combination is safe. Fix: keep two or three; split the class if more are needed.
- **Subclass must call `super()` in the right place.** That is a sign the step boundaries are wrong. Fix: add a hook so the base calls the subclass, not the other way round.
- **Deep hierarchies for variations.** Fix: replace the varying step with an injected strategy.
- **Hidden state shared between steps via `self`.** Fix: pass data between steps as return values.

## Refactoring path
- **Introduce it:** from two or more classes with copy-pasted procedures. (1) Make the procedures identical in shape by extracting the differing parts into methods with the same names. (2) Pull the shared procedure up into a base class as `run()`. (3) Mark the differing methods abstract. (4) Delete the duplicates.
- **Remove it / simplify:** replace inheritance with delegation: turn the abstract steps into constructor parameters (callables or Protocol objects), make `run()` a function, delete the subclasses that only implemented one step.

## Related patterns
- [Strategy](strategy.md) — composition-based alternative; vary a step by passing an object or function.
- [Factory](../creational/factory.md) — Factory Method is a template method whose step creates an object.
- [Decorator](../structural/decorator.md) — adds behaviour around a call without subclassing; good for cross-cutting steps (timing, logging).
- [Pipes and Filters](../architectural/pipes-and-filters.md) — when the steps are independent data transformations.
- [Chain of Responsibility](chain-of-responsibility.md) — a configurable sequence of handlers instead of a fixed skeleton.
