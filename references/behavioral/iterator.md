# Iterator

**Category:** Behavioral · **Also known as:** Cursor, Generator, Enumerator · **Related:** Composite, Visitor, Observer, Lazy Load, Value List Handler, Producer-Consumer, Pipes and Filters

## Intent
Provide a way to access the elements of a collection one at a time, without exposing how the collection is stored, and without materialising everything at once.

## Problem
- Callers reach into a collection's internals (`tree._children[0]._children`, `page["data"]["items"]`) and break when the storage changes.
- Loading everything into memory first: `rows = cursor.fetchall()` on 10 million rows, or `list(all_pages())` from a paginated API that never ends.
- The same traversal (pagination loop, tree walk, retry-until-empty) is copy-pasted at every call site.
- Scenario: an export job needs every order from a REST API that returns 100 per page with a `next_cursor`. Three services each implement the cursor loop, one of them forgets the last page.

## Solution and structure
- **Iterable** (the aggregate) produces an iterator: `__iter__` in Python, `[Symbol.iterator]` in JS.
- **Iterator** keeps the traversal position and returns the next element: `__next__` / `next()`, signalling the end with `StopIteration` / `{ done: true }`.
- The **client** only uses the protocol (`for x in ...`, `for...of`), so storage can change freely.

In Python and TypeScript you almost never write the iterator class by hand: **generators** (`yield`) produce one, and **async generators** cover I/O-bound sources like paginated APIs and DB cursors. The language has fully absorbed this pattern.

```
client ── for item in source ──▶ generator (holds cursor/page state) ──▶ storage / API
```

## When to use
- Large or unbounded sequences: DB cursors, files, paginated APIs, Kafka partitions, event streams.
- Hide a non-trivial traversal (tree, graph, pagination, retry) behind a simple loop.
- Lazy pipelines: filter, map, batch without building intermediate lists.
- Several independent traversals of the same collection at the same time.

## When NOT to use
- The data is small and already in a list. Return the list; it is easier to index, re-iterate and debug.
- The caller needs random access, `len()` or multiple passes. A generator is single-pass; return a sequence.
- Push-based sources (UI events, websockets) where the producer decides timing. Use Observer / RxJS; or bridge with an async generator over a queue.
- A custom iterator class where a generator function would do.

## Example — Python
An async generator hiding cursor pagination, plus a batching helper.

```python
from collections.abc import AsyncIterator
from itertools import batched          # 3.12+
import httpx

async def iter_orders(client: httpx.AsyncClient, since: str) -> AsyncIterator[dict]:
    cursor: str | None = None
    while True:
        params = {"since": since, "limit": 100} | ({"cursor": cursor} if cursor else {})
        resp = await client.get("/orders", params=params)
        resp.raise_for_status()
        page = resp.json()
        for order in page["items"]:
            yield order
        cursor = page.get("next_cursor")
        if not cursor:
            return

async def export_orders(since: str) -> int:
    count = 0
    async with httpx.AsyncClient(base_url="https://api.example.com", timeout=10) as client:
        async for order in iter_orders(client, since):
            if order["status"] == "paid":
                count += 1          # memory stays flat regardless of total size
    return count

# sync data: lazy chunks for bulk inserts
def insert_in_chunks(rows: list[tuple[str, int]], size: int = 500) -> None:
    for chunk in batched(rows, size):
        print(f"INSERT {len(chunk)} rows")
```

If the consumer may stop early (`break`), close the generator (`await agen.aclose()` or `contextlib.aclosing`) so its `finally` and `async with` blocks run deterministically.

## Example — TypeScript
Async generator in Node or the browser; in Angular you would wrap it with RxJS `from()` when you need operators.

```typescript
interface Page<T> { items: T[]; nextCursor?: string }
interface Order { id: string; status: 'paid' | 'pending'; totalCents: number }

export async function* paginate<T>(
  fetchPage: (cursor?: string) => Promise<Page<T>>,
  signal?: AbortSignal,
): AsyncGenerator<T, void, undefined> {
  let cursor: string | undefined;
  do {
    signal?.throwIfAborted();
    const page = await fetchPage(cursor);
    yield* page.items;
    cursor = page.nextCursor;
  } while (cursor);
}

const fetchOrders = async (cursor?: string): Promise<Page<Order>> => {
  const url = new URL('https://api.example.com/orders');
  if (cursor) url.searchParams.set('cursor', cursor);
  const res = await fetch(url);
  if (!res.ok) throw new Error(`HTTP ${res.status}`);
  return (await res.json()) as Page<Order>;
};

export async function paidRevenue(): Promise<number> {
  let total = 0;
  for await (const o of paginate(fetchOrders)) {
    if (o.status === 'paid') total += o.totalCents;
  }
  return total;
}
// RxJS: from(paginate(fetchOrders)).pipe(filter(o => o.status === 'paid'), bufferCount(50))
```

## In the wild
- Python: generators, `itertools` (`islice`, `batched`, `chain`), file objects, `csv.reader`, DB-API cursors, SQLAlchemy `Result.yield_per()` / `stream()`, `os.scandir`.
- boto3 paginators (`client.get_paginator("list_objects_v2").paginate(...)`), Stripe `auto_paging_iter()`.
- JS: iteration protocols, `Map`/`Set` iterators, `for await` over Node streams, `ReadableStream` async iteration (newer runtimes), AWS SDK v3 `paginateListObjectsV2`.
- RxJS `from(iterable)`, Angular `@for` over any iterable.
- Kafka consumers (`for msg in consumer`), `aiokafka`.

## Trade-offs
| Pros | Cons |
|---|---|
| Constant memory over huge or infinite sources | Single pass; re-iterating a spent generator silently yields nothing |
| Hides storage and traversal details | Errors surface lazily, far from where the iterator was created |
| Composes into lazy pipelines | Resources (connections, files) stay open while iteration is paused |
| Built into both languages; no classes needed | Harder to debug and inspect than a list |

## Pitfalls
- **Consuming a generator twice.** The second loop is empty and nothing fails. Fix: return a list when callers need multiple passes, or return an iterable object whose `__iter__` creates a fresh generator.
- **Abandoned generators holding resources.** Fix: `with closing(...)` / `contextlib.aclosing`, or keep the resource's `with` outside the generator.
- **Modifying a collection while iterating** (`RuntimeError: dictionary changed size`). Fix: iterate over a copy or collect changes and apply after.
- **Accidental materialisation**: `list()`, `sorted()`, or `len()` on a lazy stream. Fix: keep the pipeline lazy end-to-end, or accept the cost explicitly.
- **Lazy evaluation outliving its context**: a generator using a DB session that has been closed by the time it is consumed. Fix: consume inside the session scope.

## Refactoring path
- **Introduce it:** from copy-pasted pagination or traversal loops, or from `fetchall()` memory spikes. (1) Move one loop into a generator that yields items. (2) Replace the call site's loop body with a `for` over the generator. (3) Reuse it at the other call sites. (4) Add batching/limits as separate generator helpers.
- **Remove it / simplify:** if the data is always small and consumers call `list()` on it anyway, return a list and drop the laziness.

## Related patterns
- [Observer](observer.md) — the push-based dual; the producer calls you instead of you pulling.
- [Visitor](visitor.md) — adds type-specific behaviour during a traversal; an iterator only provides order.
- [Composite](../structural/composite.md) — trees are often exposed through a depth-first iterator.
- [Value List Handler](../data-access/value-list-handler.md) — server-side paging of large result sets.
- [Lazy Load](../optimization/lazy-load.md) — defers loading data until it is needed; iterators defer per element.
- [Producer-Consumer](../concurrency/producer-consumer.md) — iterator across threads or tasks via a queue.
- [Pipes and Filters](../architectural/pipes-and-filters.md) — chains of generators form a pipeline.
