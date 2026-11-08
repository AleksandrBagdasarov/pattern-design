# Value List Handler

**Category:** Data access · **Also known as:** Pagination, paged list, cursor pagination · **Related:** Query Object, Iterator, DTO, Repository, CQRS

## Intent
Serve a large result set to a client one page at a time, with a stable way to fetch the next page, instead of loading or returning everything.

## Problem
- `GET /orders` returns 400k rows; the API times out and the browser freezes.
- Offset pagination on a big table gets slower with every page (`OFFSET 200000` scans and discards 200k rows).
- Rows are inserted while the user pages, so items are duplicated or skipped between pages.
- Scenario: an Angular dashboard lists transactions with infinite scroll; a nightly export walks all of them.

## Solution and structure
The handler takes a query (filters, sort), a page size and a position token, and returns one page plus the token for the next one.

```
client ──GET /orders?limit=50&cursor=abc──> handler ──> query WHERE (sort_key, id) < (cursor) LIMIT 51
       <── { items: [...50], next_cursor: "xyz" | null } ──
```
**Offset vs keyset (cursor):**

| | Offset/limit | Keyset / cursor |
|---|---|---|
| Query | `ORDER BY created_at DESC, id DESC LIMIT 50 OFFSET 1000` | `WHERE (created_at, id) < ($ts, $id) ORDER BY created_at DESC, id DESC LIMIT 50` |
| Cost per page | grows with offset (rows are scanned and discarded) | constant with a matching index |
| Jump to page N | yes | no (next/previous only) |
| Concurrent inserts/deletes | duplicates or skipped rows | stable |
| Total count | often shown (`COUNT(*)` is itself expensive) | usually omitted or estimated |
| Fits | small tables, admin grids with page numbers | feeds, infinite scroll, APIs, exports |

The cursor is an opaque, encoded tuple of the last row's sort key and its unique ID. Always include a unique tiebreaker in the sort.

## When to use
- Any list endpoint over a table that can grow unboundedly.
- Infinite scroll, activity feeds, public APIs, sync/export jobs.
- Batch jobs iterating a large table without holding one huge result in memory.

## When NOT to use
- Small bounded lists (countries, a user's 10 addresses): return all.
- Analytics over the whole set: aggregate in the DB, don't page through it in the app.
- Keyset when the UI truly needs arbitrary page jumps on a small/medium table: offset is simpler.

## Example — Python
Keyset pagination with SQLAlchemy and an opaque base64 cursor.

```python
import base64, json
from datetime import datetime
from pydantic import BaseModel
from sqlalchemy import select, tuple_
from sqlalchemy.orm import Session
from app.orm import TxRow

class Page[T](BaseModel):
    items: list[T]
    next_cursor: str | None

class TxOut(BaseModel):
    id: int
    amount: str
    created_at: datetime

def _encode(ts: datetime, id_: int) -> str:
    return base64.urlsafe_b64encode(json.dumps([ts.isoformat(), id_]).encode()).decode()

def _decode(cursor: str) -> tuple[datetime, int]:
    ts, id_ = json.loads(base64.urlsafe_b64decode(cursor))
    return datetime.fromisoformat(ts), int(id_)

def list_transactions(session: Session, account_id: int, limit: int = 50,
                      cursor: str | None = None) -> Page[TxOut]:
    limit = min(limit, 200)
    stmt = (select(TxRow).where(TxRow.account_id == account_id)
            .order_by(TxRow.created_at.desc(), TxRow.id.desc()).limit(limit + 1))
    if cursor:
        stmt = stmt.where(tuple_(TxRow.created_at, TxRow.id) < _decode(cursor))
    rows = session.scalars(stmt).all()
    has_more = len(rows) > limit
    rows = rows[:limit]
    items = [TxOut(id=r.id, amount=str(r.amount), created_at=r.created_at) for r in rows]
    nxt = _encode(rows[-1].created_at, rows[-1].id) if has_more else None
    return Page[TxOut](items=items, next_cursor=nxt)
```
Index to match: `CREATE INDEX ON transactions (account_id, created_at DESC, id DESC)`. Row-value comparison `(a, b) < (x, y)` works in PostgreSQL, MySQL 8 and SQLite; on others expand it to `a < x OR (a = x AND b < y)`. For internal batch jobs, `session.execute(stmt).yield_per(1000)` streams without a cursor API.

## Example — TypeScript
Angular: infinite scroll that follows `next_cursor`, with signals.

```ts
import { Injectable, inject, signal } from '@angular/core';
import { HttpClient, HttpParams } from '@angular/common/http';

export interface Page<T> { items: T[]; next_cursor: string | null }
export interface Tx { id: number; amount: string; created_at: string }

@Injectable({ providedIn: 'root' })
export class TransactionsList {
  private readonly http = inject(HttpClient);
  private cursor: string | null = null;
  readonly items = signal<Tx[]>([]);
  readonly done = signal(false);
  readonly loading = signal(false);

  loadMore(accountId: number): void {
    if (this.loading() || this.done()) return;          // guard against double-fire on scroll
    this.loading.set(true);
    let params = new HttpParams().set('limit', 50);
    if (this.cursor) params = params.set('cursor', this.cursor);
    this.http.get<Page<Tx>>(`/api/accounts/${accountId}/transactions`, { params })
      .subscribe({
        next: (page) => {
          this.items.update((xs) => [...xs, ...page.items]);
          this.cursor = page.next_cursor;
          this.done.set(page.next_cursor === null);
        },
        complete: () => this.loading.set(false),
        error: () => this.loading.set(false),
      });
  }
}
```
For very long lists also virtualize rendering (CDK `cdk-virtual-scroll-viewport`) so the DOM stays small.

## In the wild
- Stripe (`starting_after`), GitHub/Slack/Shopify GraphQL (Relay connection spec: `edges`, `pageInfo.endCursor`), DynamoDB `LastEvaluatedKey`, S3 `ContinuationToken`, Elasticsearch `search_after`.
- Django REST Framework `CursorPagination`, `fastapi-pagination`, Prisma `cursor` option.
- AWS SDK and `boto3` paginators wrap continuation tokens into iterators.

## Trade-offs
| Pros | Cons |
|---|---|
| Bounded memory and response size | Extra API surface (cursor, page DTO) |
| Keyset gives constant-time pages and stability under writes | Keyset cannot jump to page N; sorting by arbitrary columns needs matching indexes |
| Works naturally with streaming/infinite scroll | Totals are expensive or approximate |
| Same handler serves UI and batch export | Cursors must be validated and versioned |

## Pitfalls
- **Non-unique sort key** (`created_at` only): rows skipped or duplicated. Fix: add the primary key as tiebreaker in both `ORDER BY` and the cursor.
- **No max page size:** clients request `limit=100000`. Fix: clamp.
- **`COUNT(*)` on every request** for a big table. Fix: drop it, cache it, or use an estimate.
- **Cursor tied to a different sort/filter** than the request. Fix: embed sort/filter in the cursor or reject mismatches.
- **Transparent cursors** (raw IDs) that clients start constructing. Fix: opaque encoding; sign it if tampering matters.
- **Offset on large tables** for exports. Fix: keyset or server-side streaming.

## Refactoring path
- **Introduce it:** add `limit` with a sane default and max to the unbounded endpoint; return a `Page` envelope; then replace offset with keyset once tables grow, keeping `offset` temporarily for old clients.
- **Remove it / simplify:** if a list is provably small and bounded, return the plain array and delete the paging machinery.

## Related patterns
- [Query Object](../data-access/query-object.md) — defines what is being paged.
- [Iterator](../behavioral/iterator.md) — a paginator exposed as an iterator/async generator.
- [DTO](../enterprise/dto.md) — the page envelope and item schema.
- [Repository](../data-access/repository.md) — a natural home for `list_page(cursor, limit)`.
- [CQRS](../cloud/cqrs.md) — read models designed for the paged query.
