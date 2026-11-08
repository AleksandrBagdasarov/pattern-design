# Pipes and Filters

**Category:** Architectural · **Also known as:** Pipeline · **Related:** Data Pipeline, Chain of Responsibility, Producer-Consumer, Decorator, Iterator, Splitter

## Intent
Structure processing as a chain of independent **filters**, each transforming its input into output, connected by **pipes** that carry data from one filter to the next. Filters share no state and do not know their neighbors, so they can be reordered, reused, tested and scaled separately.

## Problem
- One function parses, validates, enriches, deduplicates and writes records; changing one step risks all others.
- Some steps are slow (geocoding, ML inference) and should run concurrently while others are trivial.
- The same steps are reused in a different order for another feed.
- Scenario: importing supplier CSVs. Each file must be decoded, rows parsed, prices normalized to EUR, invalid rows sent to a dead-letter file, valid rows upserted in batches of 500.

## Solution and structure
```
 source ─pipe─► [parse] ─pipe─► [validate] ─pipe─► [to_eur] ─pipe─► [batch] ─pipe─► sink
                                   └──► reject pipe ──► dead-letter sink
```
- **Filter:** a pure-ish transformation `Iterable[In] -> Iterable[Out]` (or per-item `In -> Out | None`). Stateless or with only local state (a batch buffer).
- **Pipe:** the connector: a generator chain, a queue, an OS pipe, a Kafka topic.
- **Source / sink:** the ends (file reader, DB writer).
- **Pull** (generators, iterators) vs **push** (callbacks, RxJS, streams). Pipes may be in-process or across processes/services.

## When to use
- ETL, log processing, media transcoding, compilers (lex → parse → optimize → emit), request middleware.
- Steps are naturally sequential and independently testable.
- You need to scale or parallelize individual stages (bounded queues between them).

## When NOT to use
- Interactive request/response with lots of shared context: filters would pass a huge "context" dict, which is a hidden shared state.
- Steps that need random access to the whole dataset (global sort, joins): use a database or a batch framework.
- Only two fixed steps: a plain function call `write(parse(f))` is clearer.
- Smell: filters that read/write a shared mutable object, or that must run in exactly one order with no reuse.

## Example — Python
```python
# Generators are pipes; each filter is a function from iterator to iterator (lazy, constant memory).
import csv
from collections.abc import Callable, Iterable, Iterator
from decimal import Decimal, InvalidOperation
from itertools import batched          # Python 3.12+
from typing import TextIO

Row = dict[str, str]
RATES = {"EUR": Decimal(1), "USD": Decimal("0.92")}

def parse(f: TextIO) -> Iterator[Row]:
    yield from csv.DictReader(f)

def validate(rows: Iterable[Row], rejects: list[Row]) -> Iterator[Row]:
    for r in rows:
        try:
            Decimal(r["price"])
        except (KeyError, InvalidOperation):
            rejects.append(r)
            continue
        if r.get("currency") in RATES:
            yield r
        else:
            rejects.append(r)

def to_eur(rows: Iterable[Row]) -> Iterator[Row]:
    for r in rows:
        eur = Decimal(r["price"]) * RATES[r["currency"]]
        yield {**r, "price": str(eur.quantize(Decimal("0.01"))), "currency": "EUR"}

def run_import(f: TextIO, upsert: Callable[[list[Row]], None]) -> list[Row]:
    rejects: list[Row] = []
    for chunk in batched(to_eur(validate(parse(f), rejects)), 500):
        upsert(list(chunk))                  # sink
    return rejects
```
Swap a filter for a `multiprocessing`/`asyncio.Queue` stage only when profiling shows it is the bottleneck.

## Example — TypeScript
```ts
// Node streams: each filter is a Transform; stream.pipeline wires pipes and propagates errors/backpressure.
import { createReadStream, createWriteStream } from 'node:fs';
import { pipeline } from 'node:stream/promises';
import { Transform } from 'node:stream';
import { createGunzip } from 'node:zlib';
import split2 from 'split2';               // line splitter (third-party)

const parseJson = new Transform({
  readableObjectMode: true,
  transform(line: string, _enc, cb) {
    try { cb(null, JSON.parse(line)); } catch { cb(); }          // drop malformed lines
  },
});

const onlyErrors = new Transform({
  objectMode: true,
  transform(evt: { level: string; msg: string }, _enc, cb) {
    cb(null, evt.level === 'error' ? `${evt.msg}\n` : undefined);
  },
});

await pipeline(
  createReadStream('app.log.gz'), createGunzip(), split2(), parseJson, onlyErrors,
  createWriteStream('errors.txt'),
);
```
In Angular, an RxJS `pipe(filter(...), map(...), debounceTime(...))` is the same style applied to event streams.

## In the wild
- Unix shells (`cat | grep | sort | uniq`), Python generators, `itertools`, Node streams, RxJS operators.
- Apache Beam, Kafka Streams, Spark transformations, GStreamer/FFmpeg filter graphs, compilers.
- HTTP middleware stacks (Starlette/ASGI, Express) are pipes and filters over requests.

## Trade-offs
| Pros | Cons |
|---|---|
| Filters reusable, testable in isolation | Data must be serialized between process-level filters |
| Easy to add, remove, reorder steps | Error handling across stages is harder |
| Stages scale or parallelize independently | Shared context tempts "smart" filters with hidden coupling |
| Lazy pull pipelines run in constant memory | Debugging a long chain needs per-stage observability |

## Pitfalls
- **Losing errors:** a failed filter silently stops the stream. Use `stream.pipeline` (not `.pipe()`), and route bad items to a reject pipe instead of raising.
- **No backpressure:** unbounded in-memory lists between stages blow up memory. Use generators or bounded queues.
- **Materializing in the middle:** `list(rows)` in a filter defeats laziness.
- **Stateful filters:** a dedupe filter with a growing `set` is fine only with bounded keys; otherwise use an external store.
- **Exhausted iterators:** a generator pipeline can be consumed once; don't iterate it twice.

## Refactoring path
- **Introduce it:** take the monolithic loop body; extract each step into a function `rows -> rows`; compose them; add tests per filter; then make stages concurrent where profiling demands.
- **Remove it / simplify:** if the pipeline has two or three steps that never change, inline them into one well-named function; collapse cross-process queues back into an in-process generator chain if throughput allows.

## Related patterns
- [Data Pipeline](../emergent/data-pipeline.md) — the operational version with checkpoints and scheduling.
- [Chain of Responsibility](../behavioral/chain-of-responsibility.md) — each handler may stop the chain; filters usually all run.
- [Producer-Consumer](../concurrency/producer-consumer.md) — the queue-based pipe between concurrent filters.
- [Decorator](../structural/decorator.md) — wraps one object; a pipeline composes many transformations.
- [Iterator](../behavioral/iterator.md) — generators provide the pull-based pipes.
- [Splitter](../enterprise/splitter.md) — a filter that turns one message into many.
