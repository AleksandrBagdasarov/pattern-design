# View Helper

**Category:** UI · **Also known as:** Template helper, filter, pipe, formatter · **Related:** Presentation Model, Decorator, Value Object, MVC

## Intent
Move formatting, conversion and small display decisions out of templates into named, reusable, testable helpers (pipes, template filters, formatter functions), so templates express layout and helpers express presentation rules.

## Problem
- Templates contain expressions like `{{ (o.total / 100).toFixed(2) + ' ' + (o.currency === 'EUR' ? '€' : o.currency) }}`.
- The same date/money/status formatting is copy-pasted across 15 templates and drifts (some show `12.5`, some `12.50`).
- Methods called from templates (`{{ formatStatus(o) }}`) run on every change detection pass.
- Display rules cannot be unit-tested without rendering.

Scenario: order status codes (`PAID`, `SHIPPED`, `RTS`) must be shown as localized labels with a color. A `orderStatus` pipe returns the label and a small helper returns the CSS class; both are tested with plain unit tests.

## Solution and structure
```
Template ──{{ value | helper:args }}──▶ View Helper (pure function / pipe / filter) ──▶ display string / class / URL
```
- **Helper**: a pure, stateless function of its inputs (value, locale, options).
- **Template** calls it by name; it never contains the logic itself.
- Optionally, helpers wrap platform formatters (`Intl.NumberFormat`, Babel) to keep locale handling in one place.

## When to use
- Repeated formatting: money, dates, relative time, file sizes, phone numbers, status labels, initials, truncation.
- Display-only mapping (enum → label/icon/color).
- URL building for links and assets (`url_for`, `routerLink` arrays from ids).

## When NOT to use
- Logic that affects behavior or data (can the user edit? what is the total?): that belongs in a Presentation Model or domain layer, not a formatter.
- One-off formatting used once: a `computed()` in the component is fine.
- The framework already has it: Angular `DatePipe`, `CurrencyPipe`, `DecimalPipe`, `PercentPipe`, `I18nPluralPipe`; Jinja `format`, Babel `format_currency`. Do not rewrite these.
- Helpers that call HTTP or read global state: they stop being pure and become hidden services.

## Example — Python
```python
from datetime import datetime, timezone
from decimal import Decimal
from babel.numbers import format_currency
from babel.dates import format_timedelta
from fastapi.templating import Jinja2Templates

STATUS_LABELS = {"PAID": "Paid", "SHIPPED": "Shipped", "RTS": "Returned to sender"}

def money(cents: int, currency: str = "EUR", locale: str = "de_DE") -> str:
    return format_currency(Decimal(cents) / 100, currency, locale=locale)

def status_label(code: str) -> str:
    return STATUS_LABELS.get(code, code.title())

def ago(ts: datetime, now: datetime | None = None) -> str:
    now = now or datetime.now(timezone.utc)
    return format_timedelta(ts - now, add_direction=True, locale="en")

templates = Jinja2Templates(directory="templates")
templates.env.filters.update(money=money, status_label=status_label, ago=ago)
# template: <td>{{ order.total_cents | money(order.currency) }}</td>
#           <td>{{ order.status | status_label }}</td>

def test_money_formats_german_locale() -> None:
    assert money(123456, "EUR") == "1.234,56\xa0€"

def test_unknown_status_falls_back() -> None:
    assert status_label("ON_HOLD") == "On_Hold"
```
In Python a view helper is just a function registered as a template filter; no class is needed.

## Example — TypeScript
```ts
import { Pipe, PipeTransform, Component, input, LOCALE_ID, inject } from '@angular/core';
import { CurrencyPipe } from '@angular/common';

export type OrderStatus = 'PAID' | 'SHIPPED' | 'RTS';
const LABELS: Record<OrderStatus, string> = { PAID: 'Paid', SHIPPED: 'Shipped', RTS: 'Returned to sender' };

@Pipe({ name: 'orderStatus' })      // pure by default: re-runs only when the input reference changes
export class OrderStatusPipe implements PipeTransform {
  transform(status: OrderStatus): string { return LABELS[status] ?? status; }
}

@Pipe({ name: 'fileSize' })
export class FileSizePipe implements PipeTransform {
  private readonly fmt = new Intl.NumberFormat(inject(LOCALE_ID), { maximumFractionDigits: 1 });
  transform(bytes: number): string {
    const units = ['B', 'KB', 'MB', 'GB'];
    const i = bytes > 0 ? Math.min(Math.floor(Math.log(bytes) / Math.log(1024)), units.length - 1) : 0;
    return `${this.fmt.format(bytes / 1024 ** i)} ${units[i]}`;
  }
}

@Component({
  selector: 'app-order-row',
  imports: [OrderStatusPipe, FileSizePipe, CurrencyPipe],
  template: `
    <span>{{ status() | orderStatus }}</span>
    <span>{{ total() | currency: 'EUR' }}</span>
    <span>{{ invoiceBytes() | fileSize }}</span>`,
})
export class OrderRow {
  readonly status = input.required<OrderStatus>();
  readonly total = input.required<number>();
  readonly invoiceBytes = input(0);
}
```
Test: `new OrderStatusPipe().transform('RTS')` — no TestBed needed. `FileSizePipe` uses `inject()` in a field initializer, so instantiate it via `TestBed.runInInjectionContext(() => new FileSizePipe())`.

## In the wild
- Angular built-in pipes (`date`, `currency`, `async`, `json`, `keyvalue`); custom `@Pipe`.
- Jinja2 filters, Django template filters/tags (`{% load humanize %}`), Rails helpers, Laravel Blade directives.
- `Intl.*` APIs, Babel (Python i18n), `humanize` package, `date-fns`.
- Core J2EE Patterns (View Helper, 2001) — the origin of the name.

## Trade-offs
| Pros | Cons |
|---|---|
| One source of truth for display rules | Indirection: reader must look up the helper |
| Trivially unit-testable pure functions | Helper sprawl (dozens of tiny pipes) if not organized |
| Pure pipes are memoized per input in Angular | Impure pipes run every change-detection cycle |
| Templates become readable | Temptation to hide business logic in "formatters" |

## Pitfalls
- **Impure pipes (`pure: false`) or template method calls** for expensive work: they run on every CD pass. Keep pipes pure; for derived state use `computed()`.
- **Mutating inputs** of a pure pipe (pushing into the same array) — the pipe will not re-run. Pass new references.
- **Locale hard-coded** inside helpers. Inject `LOCALE_ID` / pass locale explicitly.
- **Helpers with side effects** (logging, HTTP). Keep them pure; move effects to services.

## Refactoring path
- **Introduce it:** grep templates for repeated expressions or template method calls; extract a pure function, test it, wrap as pipe/filter, replace usages.
- **Remove it / simplify:** delete custom helpers duplicated by built-ins (`DatePipe`, `Intl`); inline single-use helpers into a `computed`.

## Related patterns
- [Presentation Model](../ui/presentation-model.md) — owns screen state; helpers only format values.
- [Decorator](../structural/decorator.md) — older "presenter/decorator" objects (Draper) wrap a model to add display methods; pipes are the functional form.
- [Value Object](../emergent/value-object.md) — a `Money` value object can carry its own `format()`; then a pipe just delegates.
- [MVC](../architectural/mvc.md) — helpers keep the V in MVC free of logic.
