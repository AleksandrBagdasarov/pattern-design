# Setter Injection

**Category:** Behavioral (injection) · **Also known as:** Property injection, Method injection (loosely) · **Related:** Dependency Injection, Constructor Injection, Interface Injection, Null Object, Strategy

## Intent
Supply a dependency after construction through a setter or writable property, for dependencies that are optional, have a sensible default, or must be replaceable during the object's lifetime.

## Problem
- Some collaborators are genuinely optional (a progress callback, a custom formatter, an exception handler) and forcing them through every constructor call adds noise.
- Some must change at runtime: swap a log formatter, change a retry policy after reading remote config, set an exception handler on an already-running event loop.
- Some objects are created by a framework or a deserializer that only calls a no-arg constructor, so dependencies can only be supplied afterwards.
- Smells that push toward setters: constructors with many optional `None` parameters; a constructor-injection cycle (A needs B, B needs A); objects that must exist before their collaborator is ready.

Scenario: a report exporter has a sensible default formatter and no progress reporting. A CLI wants to plug a progress bar; an admin endpoint wants to switch the CSV dialect per tenant at runtime.

## Solution and structure
- The consumer declares a writable property or `set_x()` method for the dependency.
- The property starts with a safe default (often a Null Object) so the object is valid even if nobody calls the setter.
- The injector (composition root, container, framework or caller) calls the setter when it has the dependency.

```
exporter = ReportExporter()                    # valid: default formatter, no-op progress
exporter.progress = TqdmProgress(total)        # inject later (optional)
exporter.formatter = TenantCsvFormatter(...)   # replace at runtime
```

Key rule: **required dependencies do not belong here.** If the object cannot work without it, use Constructor Injection; setter injection for a required dependency creates a temporal coupling ("call `set_repo` before `run`").

## When to use
- Optional dependencies with a good default (Null Object / Black Hole).
- Dependencies that legitimately change during the object's lifetime (strategy swap, reconfiguration, hot reload).
- Framework-created objects that need collaborators after instantiation (some plugin systems, Java Beans, serializers).
- As a temporary workaround to break a construction cycle while you refactor the cycle away.

## When NOT to use
- Required dependencies: use the constructor.
- Values that differ per call: pass them as method parameters.
- Thread-shared services: swapping a dependency while other threads/tasks use it is a race. Prefer building a new instance.
- When the only motivation is "too many constructor parameters": that is a responsibility problem; split the class.
- In Angular services: `inject()` handles optional deps (`inject(TOKEN, { optional: true })`); don't add setters for DI. Component `input()` covers parent-supplied configuration.

## Example — Python
Properties with defaults. The stdlib itself uses this form widely (`Handler.setFormatter`, `loop.set_exception_handler`).

```python
import csv
import io
from collections.abc import Callable, Iterable
from dataclasses import dataclass

type Progress = Callable[[int], None]

def _no_progress(_: int) -> None:            # Null Object default
    pass

@dataclass(frozen=True)
class CsvFormat:
    delimiter: str = ","
    quoting: int = csv.QUOTE_MINIMAL

class ReportExporter:
    def __init__(self, rows_source: Callable[[], Iterable[dict]]) -> None:
        self._rows = rows_source              # required -> constructor
        self._format = CsvFormat()            # optional -> setter, with defaults
        self._progress: Progress = _no_progress

    @property
    def format(self) -> CsvFormat:
        return self._format

    @format.setter
    def format(self, value: CsvFormat) -> None:
        self._format = value

    def set_progress(self, callback: Progress | None) -> None:
        self._progress = callback or _no_progress     # None resets to the default

    def export(self) -> str:
        buf = io.StringIO()
        writer: csv.DictWriter | None = None
        for i, row in enumerate(self._rows(), start=1):
            if writer is None:
                writer = csv.DictWriter(buf, row.keys(), delimiter=self._format.delimiter,
                                        quoting=self._format.quoting)
                writer.writeheader()
            writer.writerow(row)
            self._progress(i)
        return buf.getvalue()

exporter = ReportExporter(lambda: [{"id": 1, "total": "9.99"}])
exporter.format = CsvFormat(delimiter=";")    # tenant uses European Excel
exporter.set_progress(lambda n: print(f"{n} rows"))
```

## Example — TypeScript
Plain TS with a setter and a default. In Angular, the parent-supplied variant is a component `input()`; optional services use `inject(..., { optional: true })` instead of setters.

```typescript
import { Component, InjectionToken, computed, inject, input } from '@angular/core';

export type PriceFormatter = (cents: number) => string;
const defaultFormatter: PriceFormatter = c => (c / 100).toFixed(2);

export class PriceLabelRenderer {
  private _formatter: PriceFormatter = defaultFormatter;

  set formatter(f: PriceFormatter | null) {           // setter injection with reset
    this._formatter = f ?? defaultFormatter;
  }
  render(cents: number): string { return this._formatter(cents); }
}

// Angular: optional dependency without a setter
export const CURRENCY_FORMATTER = new InjectionToken<PriceFormatter>('CURRENCY_FORMATTER');

@Component({
  selector: 'app-price',
  template: `<span>{{ label() }}</span>`,
})
export class Price {
  readonly cents = input.required<number>();
  readonly formatter = input<PriceFormatter | undefined>(undefined);   // parent may "set" it
  private readonly injected = inject(CURRENCY_FORMATTER, { optional: true });

  protected readonly label = computed(() =>
    (this.formatter() ?? this.injected ?? defaultFormatter)(this.cents()));
}
```

## In the wild
- Python stdlib: `logging.Handler.setFormatter`, `Logger.setLevel`, `asyncio` `loop.set_exception_handler` / `set_task_factory`, `sys.excepthook` (a module-level property), `http.cookiejar.CookieJar.set_policy`.
- Spring `@Autowired` on setters (now discouraged for required deps); Java Beans; .NET property injection (Autofac).
- Angular: `@Input()` setters in older code, `input()` in current code, `@ViewChild` setters for late-available children.
- Node: `server.setTimeout`, `emitter.setMaxListeners`; Express `app.set('json replacer', fn)`.

## Trade-offs
| Pros | Cons |
|---|---|
| Optional dependencies without constructor noise | Object may be used before the setter is called |
| Dependencies replaceable at runtime | Mutable state: races under concurrency |
| Works with frameworks that need a no-arg constructor | Hidden temporal coupling ("call setX first") |
| Can break construction cycles (as a stopgap) | Dependencies less visible than in the constructor |

## Pitfalls
- **Required dependency via setter.** `run()` fails with `None` because nobody called `set_repo`. Fix: move to the constructor.
- **No default.** The field is `None` until set, so every method checks. Fix: default to a Null Object; let the setter accept `None` to reset.
- **Swapping under concurrency.** A request reads half-old, half-new config. Fix: replace one immutable config object atomically (a single reference assignment), or build a new instance.
- **Setters used to hide a cycle permanently.** Fix: extract the shared part into a third object, or communicate through events.
- **Validation skipped.** A setter accepts anything. Fix: validate in the setter (raise on invalid values) or type it strictly.
- **Angular `@Input` setter side effects** running before other inputs are set. Fix: use signal inputs plus `computed`, which recompute when any input changes.

## Refactoring path
- **Introduce it:** from a constructor with many optional `None` params or a hard-coded optional collaborator. (1) Give the field a Null Object default. (2) Add a setter/property. (3) Move optional constructor args to setter calls at the call sites that need them. (4) Delete the `None` checks.
- **Remove it / simplify:** if a setter is called exactly once right after construction everywhere, move the dependency into the constructor (keyword arg with default) and delete the setter; if it is never called, delete it and keep the default.

## Related patterns
- [Dependency Injection](dependency-injection.md) — overview, scopes, composition root.
- [Constructor Injection](constructor-injection.md) — the default; use it for anything required.
- [Interface Injection](interface-injection.md) — the setter is mandated by an interface the injector recognizes.
- [Null Object](null-object.md) — the default value that makes setter injection safe.
- [Strategy](strategy.md) — runtime strategy swapping is the most common legitimate use of a setter.
