# Lazy Load

**Category:** Optimization · **Also known as:** Deferred loading, on-demand loading, code splitting (frontend) · **Related:** Lazy Initialization, Proxy, Caching, Virtual Proxy, Data Mapper

## Intent
Do not load data or code until it is actually needed — an ORM relation fetched on first access, a route's JavaScript downloaded on first navigation, an image loaded when it scrolls into view.

## Problem
- Loading a `Customer` eagerly pulls all orders, addresses and invoices, though most screens show only the name.
- The Angular initial bundle is 3 MB because the admin area, charts and PDF export ship to every visitor; Lighthouse reports poor LCP and TBT.
- A list page loads 200 full-size images below the fold.

Profile first: SQL logging (`echo=True`, `pg_stat_statements`) and `py-spy` for backend; Chrome DevTools Coverage tab (unused JS/CSS), Network panel, Lighthouse, and `ng build --stats-json` + `esbuild` analyzer / `source-map-explorer` for bundle composition.

## Solution and structure
```
caller ──accesses──▶ placeholder (proxy / lazy attribute / dynamic import)
                         │ first access only
                         ▼
                    real load (SQL / HTTP / chunk download) ──▶ cached in placeholder
```
Fowler's four forms: **lazy initialization** (field null until first use), **virtual proxy** (object stand-in), **value holder** (wrapper with `get()`), **ghost** (partially loaded object filling itself in). Frontend equivalents: dynamic `import()`, `loadComponent`/`loadChildren`, `@defer` blocks, `loading="lazy"` images.

The key design decision is the boundary: *what is likely to be needed together* should be loaded together; lazy everything leads to N+1.

## When to use
- Large or rarely used relations/fields (blobs, history, audit trails).
- Routes, features and heavy widgets (charts, editors, maps) not needed for first paint.
- Below-the-fold images, iframes, and heavy components.
- Expensive optional resources (ML model, large config) in CLI tools or workers.

## When NOT to use
- You will iterate over the relation for every row: lazy loading causes **N+1 queries**. Use eager loading (`selectinload`, `joinedload`) for that query.
- Async code paths: SQLAlchemy lazy loads under `AsyncSession` raise `MissingGreenlet`; load explicitly.
- Content needed for first paint/LCP (hero image, primary route): lazy-loading it makes Core Web Vitals worse.
- Tiny chunks: splitting a 5 KB component into its own request adds latency for nothing.

## Example — Python
```python
from sqlalchemy import ForeignKey, select
from sqlalchemy.orm import DeclarativeBase, Mapped, mapped_column, relationship, selectinload, raiseload
from sqlalchemy.ext.asyncio import AsyncSession
from functools import cached_property

class Base(DeclarativeBase): ...

class Customer(Base):
    __tablename__ = "customers"
    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str]
    # lazy="raise": accidental lazy access fails loudly instead of silently issuing N+1 queries
    orders: Mapped[list["Order"]] = relationship(back_populates="customer", lazy="raise")

class Order(Base):
    __tablename__ = "orders"
    id: Mapped[int] = mapped_column(primary_key=True)
    customer_id: Mapped[int] = mapped_column(ForeignKey("customers.id"))
    customer: Mapped[Customer] = relationship(back_populates="orders")

async def customer_names(s: AsyncSession) -> list[str]:
    return list((await s.scalars(select(Customer.name))).all())        # orders never loaded

async def customers_with_orders(s: AsyncSession) -> list[Customer]:
    stmt = select(Customer).options(selectinload(Customer.orders))      # 2 queries total, not N+1
    return list((await s.scalars(stmt)).all())

class ReportGenerator:
    @cached_property                     # lazy initialization of an expensive resource
    def templates(self) -> dict[str, str]:
        return load_all_templates()      # runs only if a report is actually rendered

def load_all_templates() -> dict[str, str]: ...
```
Making relations `lazy="raise"` by default and choosing a loader per query is the safest pattern with async SQLAlchemy.

## Example — TypeScript
```ts
import { Routes } from '@angular/router';
import { Component } from '@angular/core';

export const routes: Routes = [
  { path: '', loadComponent: () => import('./home/home').then((m) => m.Home) },
  {
    path: 'admin',                     // whole admin area in a separate chunk
    loadChildren: () => import('./admin/admin.routes').then((m) => m.ADMIN_ROUTES),
  },
];

@Component({
  selector: 'app-order-page',
  imports: [/* SalesChart, OrderHistory are standalone and only referenced inside @defer */],
  template: `
    <app-order-summary />
    @defer (on viewport; prefetch on idle) {
      <app-sales-chart />              <!-- chart lib downloaded when scrolled into view -->
    } @placeholder (minimum 200ms) {
      <div class="chart-skeleton"></div>
    } @loading {
      <p>Loading chart…</p>
    }
    <img ngSrc="/img/banner.webp" width="800" height="200" priority />   <!-- LCP: not lazy -->
    <img src="/img/footer.webp" loading="lazy" width="400" height="100" />`,
})
export class OrderPage {}

// Node: defer a heavy dependency until the code path runs
export async function exportPdf(html: string): Promise<Uint8Array> {
  const { default: puppeteer } = await import('puppeteer');
  const browser = await puppeteer.launch();
  try { const page = await browser.newPage(); await page.setContent(html); return await page.pdf(); }
  finally { await browser.close(); }
}
```
`@defer` only splits components that are standalone and not referenced elsewhere eagerly; `ngSrc` requires `NgOptimizedImage` in `imports`. Verify chunking in the `ng build` output.

## In the wild
- SQLAlchemy relationship `lazy=` strategies (`select`, `selectin`, `joined`, `raise`, `noload`), Django `select_related`/`prefetch_related` vs default lazy FK access, Hibernate proxies.
- Python `functools.cached_property`, module-level `__getattr__` (PEP 562) for lazy imports; PEP 810 explicit lazy imports targeted for Python 3.15 — check status.
- Angular `loadComponent`/`loadChildren`, `@defer`, `NgOptimizedImage`; React `lazy` + `Suspense`; native `loading="lazy"`, `IntersectionObserver`.

## Trade-offs
| Pros | Cons |
|---|---|
| Less work and memory when data/code is not used | N+1 queries and request waterfalls when it is |
| Smaller initial bundle; faster first load | Latency moved to the moment of use (spinner on click) |
| Simple to apply at route/relation boundaries | Hidden I/O on attribute access is surprising |
| Pairs well with prefetching | Errors happen later, far from where loading was configured |

## Pitfalls
- **N+1 queries:** enable SQL logging in tests, use `lazy="raise"`, or assert query counts (`django_assert_num_queries`, a SQLAlchemy event counter).
- **Accessing lazy attributes after the session closes** (`DetachedInstanceError`): load what the response needs inside the unit of work; return DTOs.
- **Lazy-loading the LCP element** (hero image/route): use `priority`/`fetchpriority="high"` instead.
- **Chunk waterfalls:** lazy route → lazy child → lazy component, each a round trip. Use `withPreloading(PreloadAllModules)` or `@defer (prefetch on idle)`.
- **Layout shift** when deferred content appears: reserve space via placeholders and image dimensions.

## Refactoring path
- **Introduce it:** measure (bundle analyzer, Coverage tab, SQL log) → split the biggest unused-on-first-load route or relation → add placeholder/prefetch → re-run Lighthouse or query counts.
- **Remove it / simplify:** if logs show N+1 or users always wait on a deferred chunk, switch that path to eager loading or prefetch, or merge tiny chunks back into the parent.

## Related patterns
- [Lazy Initialization](../creational/lazy-initialization.md) — the object-level form for single expensive resources.
- [Proxy](../structural/proxy.md) — virtual proxy is the classic lazy-load mechanism.
- [Caching](../optimization/caching.md) — once loaded, keep it; lazy load decides *when*, caching decides *how long*.
- [Data Mapper](../data-access/data-mapper.md) — ORMs implement lazy loading inside the mapper.
