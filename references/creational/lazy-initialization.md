# Lazy Initialization

**Category:** Creational · **Also known as:** Lazy Instantiation, Initialize-on-Demand · **Related:** Lazy Load, Singleton, Multiton, Proxy, Double-Checked Locking, Caching, Future/Promise

## Intent
Defer creating an expensive object or computing a value until the first time it is actually needed, then reuse the result.

## Problem
- Startup does work that most runs never use: loading an ML model, opening a connection to a rarely used service, parsing a large schema, importing a heavy SDK.
- Scenario: a CLI tool imports `boto3`, `pandas` and loads a 300 MB model at import time. `mytool --help` takes 4 s. Only one subcommand needs the model.
- Scenario (frontend): an Angular app bundles a charting library used on one admin page, adding 400 KB to the initial download.
- Smell: slow cold start, high baseline memory, or failures at startup caused by dependencies that the current code path never touches.

## Solution and structure
- **Holder**: a property, cached function, or wrapper that checks "already created?" on access.
- **Initializer**: the factory function that creates the value on first access.
- First access runs the initializer and stores the result; later accesses return the stored value.
- In concurrent code, the check-and-create must be atomic (lock, `asyncio` task/future, or language guarantees).

```
access ─► initialized? ──yes──► return cached value
                └─no──► run initializer ─► store ─► return
```

Variants: lazy attribute (`functools.cached_property`), lazy global (`@cache` on a zero-arg function), lazy import, lazy module/route (dynamic `import()`), lazy async value (memoized promise / task).

## When to use
- The resource is costly and not needed on every path (optional features, rarely used endpoints).
- Startup time or cold start matters (CLIs, serverless, mobile/SPA initial load).
- The configuration needed to create the object is only available later (after login, after settings load).

## When NOT to use
- The resource is always needed. Create it at startup so failures surface immediately, not on the first user request. Eager init plus a readiness probe is easier to operate.
- The first-request latency spike is unacceptable (a 2 s model load on a user's request). Warm it in the background or at startup.
- It hides a configuration error until 3 a.m. Validate config eagerly even if you build the client lazily.
- Cheap objects: the check costs more than it saves.

## Example — Python
`functools.cached_property` for per-instance laziness and lazy imports for heavy modules. In async code, memoize the *task* so concurrent callers share one initialization.

```python
import asyncio
from collections.abc import Awaitable, Callable
from functools import cached_property
from typing import TYPE_CHECKING

if TYPE_CHECKING:
    from sentence_transformers import SentenceTransformer   # type-only, no import cost

class SearchService:
    def __init__(self, model_name: str) -> None:
        self._model_name = model_name

    @cached_property            # first access loads it; stored in the instance __dict__
    def model(self) -> "SentenceTransformer":
        from sentence_transformers import SentenceTransformer   # lazy import (~2 s)
        return SentenceTransformer(self._model_name)

    def embed(self, text: str) -> list[float]:
        return self.model.encode(text).tolist()

class ExchangeRates:
    """Async lazy value: concurrent first callers await the same fetch."""
    def __init__(self, fetch: Callable[[], Awaitable[dict[str, float]]]) -> None:
        self._fetch = fetch
        self._task: asyncio.Task[dict[str, float]] | None = None

    async def get(self) -> dict[str, float]:
        if self._task is None:
            self._task = asyncio.create_task(self._fetch())
        try:
            return await self._task
        except Exception:
            self._task = None           # don't cache failures forever; next call retries
            raise
```

Note: `cached_property` is not thread-safe since Python 3.12 removed its internal lock; guard it with your own lock if threads share the instance.

## Example — TypeScript
Angular lazy routes and `@defer` blocks are framework-level lazy initialization; in services, memoize the promise.

```typescript
import { Injectable } from '@angular/core';
import type { Routes } from '@angular/router';

export const routes: Routes = [
  { path: '', loadComponent: () => import('./home/home').then(m => m.Home) },
  // chart library is only downloaded when /admin/reports is visited
  { path: 'admin/reports', loadComponent: () => import('./reports/reports').then(m => m.Reports) },
];

type PdfLib = typeof import('pdf-lib');

@Injectable({ providedIn: 'root' })
export class PdfExportService {
  private lib?: Promise<PdfLib>;

  private load(): Promise<PdfLib> {
    // Memoize the promise, not the result, so concurrent callers share one download.
    this.lib ??= import('pdf-lib').catch(err => { this.lib = undefined; throw err; });
    return this.lib;
  }

  async exportInvoice(lines: string[]): Promise<Uint8Array> {
    const { PDFDocument } = await this.load();
    const doc = await PDFDocument.create();
    const page = doc.addPage();
    lines.forEach((l, i) => page.drawText(l, { x: 40, y: 800 - i * 16, size: 11 }));
    return doc.save();
  }
}
// Template-level: @defer (on viewport) { <app-sales-chart/> } @placeholder { <p>Loading…</p> }
```

## In the wild
- Python: `functools.cached_property`, `functools.cache` on zero-arg getters, module-level `__getattr__` (PEP 562) for lazy attributes, `importlib.util.LazyLoader`; PEP 810 explicit lazy imports were accepted for Python 3.15 (check current docs before relying on the syntax).
- SQLAlchemy/Django lazy relationship loading (see Lazy Load); Django `settings` (`LazyObject`); pydantic-settings loaded on first use via `@cache`.
- Angular: `loadComponent`/`loadChildren`, `@defer` blocks (Angular 17+), `inject()` with `providedIn: 'root'` (services are instantiated on first injection, not at bootstrap).
- JS: dynamic `import()`, `??=` memoization, RxJS `defer()` + `shareReplay(1)`.
- AWS Lambda: clients created in global scope are lazily reused across warm invocations.

## Trade-offs
| Pros | Cons |
|---|---|
| Faster startup / smaller initial bundle | First user pays the latency spike |
| No cost for unused features | Errors surface late, at first use, not at deploy |
| Can defer until config is available | Concurrency: double initialization or races without guarding |
| Lower baseline memory | Harder to reason about when side effects happen |

## Pitfalls
- **Race on first access** (threads or concurrent coroutines create two instances). Fix: lock (threads), memoize the task/promise (async), or initialize eagerly.
- **Caching a failure** (a rejected promise stored forever). Fix: clear the memo on error, as in both examples.
- **Lazy + mutable global** turns into a hidden Singleton. Fix: make the holder an injected instance.
- **Laziness hiding misconfiguration**: missing env var discovered on first request. Fix: validate settings eagerly, create clients lazily.
- **`cached_property` on classes with `__slots__`** fails (no `__dict__`). Fix: add `__dict__` to slots or use an explicit field.
- **Lazy-loading everything in Angular** creates waterfalls of chunks. Fix: lazy-load per route/feature, preload likely routes (`withPreloading(PreloadAllModules)`).

## Refactoring path
- **Introduce it:** measure startup (`python -X importtime`, bundle analyzer, Lambda init duration). Move the costly creation behind a function; memoize it; replace direct references with the accessor; add concurrency guarding; add a warm-up hook if latency matters.
- **Remove it / simplify:** if the lazy value is accessed on every request anyway, move creation to startup (`lifespan`, `APP_INITIALIZER`/`provideAppInitializer`) and delete the memo logic.

## Related patterns
- [Lazy Load](../optimization/lazy-load.md) — lazy fetching of *data* (ORM relations, routes), the data-access cousin.
- [Singleton](singleton.md) — often created lazily; laziness is a creation detail, not the pattern.
- [Multiton](multiton.md) — lazy creation per key.
- [Proxy](../structural/proxy.md) — a virtual proxy is lazy initialization behind the real object's interface.
- [Double-Checked Locking](../concurrency/double-checked-locking.md) — thread-safe lazy init without locking every access.
- [Caching](../optimization/caching.md) — laziness with eviction and invalidation.
- [Future/Promise](../concurrency/future-promise.md) — memoizing the future is how async lazy init avoids duplicate work.
