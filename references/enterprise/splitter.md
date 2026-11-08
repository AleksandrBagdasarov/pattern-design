# Splitter

**Category:** Enterprise integration · **Also known as:** Fan-out, debatching, Composed Message Processor (with Aggregator) · **Related:** Aggregator, Message Queue, Pipes and Filters, Iterator, Data Pipeline

## Intent
Break a composite message into individual messages, one per element, so each can be routed, processed, retried and scaled independently.

## Problem
- An order with 40 lines is processed as one unit; one bad line fails the whole order and the retry redoes all 40.
- A nightly file with 2 million records is loaded into memory and processed in one transaction that holds locks for an hour.
- Different line types go to different systems (physical goods to the warehouse, licenses to the provisioning service).
- Scenario: a bulk `POST /payouts` with 5,000 payouts must call the bank per payout with rate limits and individual retries.

## Solution and structure
The splitter reads the composite, emits one message per element, and copies the context the parts need: a **correlation ID** (parent ID), the **sequence number** and the **total count**, so a downstream Aggregator can reassemble or detect completion.

```
Batch(id=B1, items=[i1, i2, i3]) ──Splitter──> {B1, 1/3, i1}  ──> worker ──┐
                                           ├─> {B1, 2/3, i2}  ──> worker ──┼──> Aggregator(B1) ──> BatchResult
                                           └─> {B1, 3/3, i3}  ──> worker ──┘
```
Variants: **iterating splitter** (stream elements, don't materialize all), **content-based split** (different element types to different channels), **chunking** (groups of N instead of single elements to balance per-message overhead).

## When to use
- Elements are independent and benefit from parallelism, isolation of failures or per-item retries.
- Different parts of a message go to different consumers.
- Large files/streams must be processed incrementally with bounded memory.

## When NOT to use
- The whole set must succeed or fail atomically (a ledger posting with balanced lines): keep it one unit/transaction.
- Elements are tiny and numerous: per-message overhead dominates; chunk instead, or use a set-based SQL statement.
- Ordering across elements matters and the transport doesn't preserve it.

## Example — Python
Streaming splitter over a CSV upload, chunked, with correlation metadata, publishing to a queue.

```python
import csv
import json
from collections.abc import Callable, Iterator
from itertools import batched          # Python 3.12+
from typing import IO, TypedDict

class Payout(TypedDict):
    payout_id: str
    iban: str
    amount_cents: int

def read_payouts(f: IO[str]) -> Iterator[Payout]:
    for row in csv.DictReader(f):                     # streams, never loads the whole file
        yield Payout(payout_id=row["id"], iban=row["iban"], amount_cents=int(row["amount_cents"]))

def split_batch(batch_id: str, f: IO[str], send: Callable[[str, bytes], None],
                chunk_size: int = 100) -> int:
    count = 0
    for seq, chunk in enumerate(batched(read_payouts(f), chunk_size), start=1):
        msg = {"batch_id": batch_id, "seq": seq, "items": list(chunk)}
        send(f"{batch_id}:{seq}", json.dumps(msg).encode())   # key = idempotency key
        count += 1
    send(f"{batch_id}:end", json.dumps({"batch_id": batch_id, "total_chunks": count}).encode())
    return count
```
Because the total is unknown until the stream ends, an "end" marker carries it for the aggregator. If you can count first (e.g. a DB query), put `total` in every part instead.

## Example — TypeScript
RxJS: split an order into lines, process each with bounded concurrency, then reassemble.

```ts
import { from, mergeMap, Observable, toArray, map, catchError, of } from 'rxjs';

interface OrderLine { sku: string; qty: number; kind: 'physical' | 'license' }
interface Order { id: string; lines: OrderLine[] }
type LineResult = { sku: string; ok: boolean; error?: string };

export function fulfil(
  order: Order,
  ship: (l: OrderLine) => Observable<void>,
  provision: (l: OrderLine) => Observable<void>,
): Observable<{ orderId: string; results: LineResult[] }> {
  return from(order.lines).pipe(                           // split
    mergeMap((line) => {
      const work$ = line.kind === 'physical' ? ship(line) : provision(line);  // content-based route
      return work$.pipe(
        map((): LineResult => ({ sku: line.sku, ok: true })),
        catchError((e: unknown) => of<LineResult>({ sku: line.sku, ok: false, error: String(e) })),
      );
    }, 4),                                                 // at most 4 lines in flight
    toArray(),                                             // aggregate
    map((results) => ({ orderId: order.id, results })),
  );
}
```
`ship`/`provision` must complete after emitting once (HTTP calls do); a never-completing inner observable stalls `toArray`.

## In the wild
- Apache Camel `split()` (with `streaming()`, `parallelProcessing()`), Spring Integration Splitter.
- AWS Step Functions `Map`/Distributed Map state (S3 objects or JSON arrays fanned out), SQS `SendMessageBatch`.
- Kafka Streams `flatMap`, Apache Beam `ParDo`, Python `itertools.batched`, RxJS `from`/`mergeMap`, Celery `group`/`chord` (split + aggregate).

## Trade-offs
| Pros | Cons |
|---|---|
| Per-item failure isolation and retries | Loses atomicity of the original batch |
| Parallelism and independent scaling | Needs correlation and completion tracking to reassemble |
| Streaming keeps memory bounded | More messages: overhead, cost, monitoring |
| Different parts routed to different consumers | Ordering across parts is lost unless preserved by key |

## Pitfalls
- **Missing correlation data** in parts: the aggregator cannot reassemble. Fix: parent ID, sequence and total/end marker in every part.
- **Materializing the whole input** (`list(reader)`). Fix: generators/streams.
- **Unbounded fan-out** (`mergeMap` without concurrency, 5,000 parallel HTTP calls). Fix: concurrency limits, rate limiter.
- **Duplicate parts on splitter retry.** Fix: deterministic part IDs (`batch:seq`) and idempotent consumers.
- **Splitting work that must be atomic.** Fix: keep it whole, or use a Saga with compensations.

## Refactoring path
- **Introduce it:** in a batch job that fails as a whole, extract per-item processing into a function; feed it from a generator; then publish items (or chunks) to a queue with correlation IDs; add an aggregator or a status table for completion.
- **Remove it / simplify:** if parts are always processed together and failures are rare, process the batch in-process with a loop and per-item error collection; or replace with a set-based SQL statement.

## Related patterns
- [Aggregator](../enterprise/aggregator.md) — reassembles the parts; split + aggregate = Composed Message Processor.
- [Message Queue](../enterprise/message-queue.md) — distributes the split parts to competing workers.
- [Pipes and Filters](../architectural/pipes-and-filters.md) — the splitter is one filter in a pipeline.
- [Iterator](../behavioral/iterator.md) — generators make streaming splitters trivial.
- [Data Pipeline](../emergent/data-pipeline.md) — debatching large inputs in ETL.
