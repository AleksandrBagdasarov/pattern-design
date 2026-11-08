# Data Pipeline

**Category:** Emergent / modern · **Also known as:** ETL / ELT, batch pipeline, streaming pipeline, data flow · **Related:** Pipes and Filters, Dependency Graph, Producer-Consumer, Event Replay, Splitter

## Intent
Move data from sources to destinations through explicit stages (extract/ingest → validate → transform → load), each stage with a clear contract, so the flow is restartable from checkpoints, observable, testable in isolation and safe to re-run.

## Problem
- One script reads from the API, cleans, joins, aggregates and writes to the warehouse in a single function. When it fails at 80%, the only option is to start over — and the first 80% is now written twice.
- Scenario: a nightly job imports orders from a partner SFTP, converts currencies, and loads them into Postgres for reporting. A malformed row crashes the job; a re-run duplicates yesterday's rows; nobody knows how many records were dropped.
- Smells: no checkpoints; re-runs not idempotent; validation mixed with transformation; memory blow-ups from loading entire files; "the cron job silently failed for a week".

## Solution and structure
```
 source ─► extract ─► validate ─► transform ─► load ─► sink
            │            │ bad rows ─► dead-letter / quarantine
            └─ checkpoint (cursor, offset, watermark, partition date)
```
- **Stages**: small functions with typed input/output; pure where possible (transform), I/O at the edges (extract/load).
- **Streaming between stages**: iterate in batches/chunks (generators, async iterators, Arrow record batches) to bound memory.
- **Checkpoint/watermark**: the last processed offset/timestamp/partition, committed with or after the load, so a restart continues.
- **Idempotent load**: upsert by natural key, or overwrite a partition (`DELETE WHERE date = X; INSERT`), so re-runs don't duplicate.
- **Dead-letter**: invalid records go to a quarantine table/file with the reason; the pipeline continues and reports counts.
- **Orchestrator** (for multi-step batch): schedules, retries and orders tasks as a DAG; the pipeline code stays independent of it.

ETL transforms before loading; ELT loads raw data first and transforms inside the warehouse (dbt) — the modern default for analytics because raw data is kept and transforms are re-runnable.

## When to use
- Recurring movement of data between systems (APIs, files, DBs, warehouses, search indexes).
- Volumes that don't fit in memory, or runs long enough that partial failures are expected.
- Data that must be validated and quarantined rather than crash the job.
- Backfills and reprocessing over historical partitions.

## When NOT to use
- A one-off migration of a few thousand rows: a single script or SQL `INSERT … SELECT` is fine.
- Synchronous request/response features: users waiting on a pipeline need an API, not a batch job.
- When the database can do it: `COPY`, materialized views, `INSERT … SELECT` inside the DB beat pulling data into Python.
- Don't build your own orchestrator, scheduler or connector framework: use Airflow/Dagster/Prefect, dbt, Airbyte/Fivetran, Kafka Connect, AWS Glue/Step Functions.

## Example — Python
Generator stages with chunked, idempotent loading, a dead-letter path and a watermark.

```python
import csv
from collections.abc import Iterable, Iterator
from datetime import date
from decimal import Decimal
from itertools import batched           # 3.12+
from pydantic import BaseModel, ValidationError

class OrderRow(BaseModel):
    order_id: str
    amount: Decimal
    currency: str
    order_date: date

def extract(path: str) -> Iterator[dict[str, str]]:
    with open(path, newline="") as f:
        yield from csv.DictReader(f)                     # streaming, constant memory

def validate(rows: Iterable[dict[str, str]], dead: list[tuple[dict, str]]) -> Iterator[OrderRow]:
    for raw in rows:
        try:
            yield OrderRow.model_validate(raw)
        except ValidationError as e:
            dead.append((raw, str(e)))                    # quarantine, don't crash

def transform(rows: Iterable[OrderRow], fx: dict[str, Decimal]) -> Iterator[dict]:
    for r in rows:
        yield {"order_id": r.order_id, "order_date": r.order_date,
               "amount_eur": (r.amount * fx[r.currency]).quantize(Decimal("0.01"))}

def load(rows: Iterable[dict], conn, run_date: date, chunk: int = 1000) -> int:
    n = 0
    with conn.transaction(), conn.cursor() as cur:        # partition overwrite = idempotent re-run
        cur.execute("DELETE FROM fact_orders WHERE load_date = %s", (run_date,))
        for batch in batched(rows, chunk):
            cur.executemany("INSERT INTO fact_orders (order_id, order_date, amount_eur, load_date) "
                             "VALUES (%(order_id)s, %(order_date)s, %(amount_eur)s, %(load_date)s)",
                             [r | {"load_date": run_date} for r in batch])
            n += len(batch)
        cur.execute("UPDATE watermarks SET last_date = %s WHERE name = 'orders'", (run_date,))
    return n

def run(path: str, conn, run_date: date, fx: dict[str, Decimal]) -> dict[str, int]:
    dead: list[tuple[dict, str]] = []
    loaded = load(transform(validate(extract(path), dead), fx), conn, run_date)
    return {"loaded": loaded, "rejected": len(dead)}     # emit as metrics; alert on ratio
```
`conn` is assumed to be a psycopg 3 connection.

## Example — TypeScript
A Node streaming pipeline using `stream/promises.pipeline` with async generator stages; backpressure and error propagation are handled by Node.

```ts
import { createReadStream } from 'node:fs';
import { pipeline } from 'node:stream/promises';
import { createInterface } from 'node:readline';

interface Event { userId: string; type: string; ts: number }

async function* parse(lines: AsyncIterable<string>): AsyncGenerator<Event> {
  for await (const line of lines) {
    try { yield JSON.parse(line) as Event; }
    catch { process.stderr.write(`dead-letter: ${line.slice(0, 200)}\n`); }
  }
}

async function* onlyPurchases(events: AsyncIterable<Event>): AsyncGenerator<Event> {
  for await (const e of events) if (e.type === 'purchase') yield e;
}

function loadInBatches(bulkInsert: (rows: Event[]) => Promise<void>, size = 500) {
  return async (events: AsyncIterable<Event>): Promise<void> => {
    let batch: Event[] = [];
    for await (const e of events) {
      batch.push(e);
      if (batch.length >= size) { await bulkInsert(batch); batch = []; }   // awaits = backpressure
    }
    if (batch.length) await bulkInsert(batch);
  };
}

export async function run(path: string, bulkInsert: (rows: Event[]) => Promise<void>) {
  const lines = createInterface({ input: createReadStream(path), crlfDelay: Infinity });
  await pipeline(lines, parse, onlyPurchases, loadInBatches(bulkInsert));
}
```

## In the wild
- Orchestration: Apache Airflow, Dagster, Prefect, AWS Step Functions + Glue, Azure Data Factory, Google Cloud Composer.
- Transform: dbt (ELT in SQL), Spark/PySpark, Polars, DuckDB, pandas (small data), Apache Beam/Dataflow (batch + streaming).
- Streaming/ingest: Kafka + Kafka Connect/Streams, Flink, Debezium CDC, Airbyte, Fivetran, AWS Kinesis Firehose.
- Language-level: Python generators/`itertools`, Node streams, RxJS operators for in-app data flows.

## Trade-offs
| Pros | Cons |
|---|---|
| Restartable, observable, testable stages | More moving parts than a script |
| Bounded memory via streaming/batching | Idempotency and checkpoints must be designed |
| Bad data quarantined instead of failing everything | Data latency (batch) or operational load (streaming) |
| Stages reusable and independently scalable | Schema changes ripple through stages |

## Pitfalls
- Non-idempotent loads (`INSERT` without dedup) turn every retry into duplicates. Overwrite partitions or upsert by key.
- Watermark advanced before the load commits: a crash skips data. Commit them together, or advance after success.
- Loading everything into memory (`list(...)`, `pd.read_csv` on a huge file). Stream or chunk.
- Silent drops: invalid rows discarded without counting. Emit loaded/rejected metrics and alert on thresholds.
- Late-arriving data ignored by a strict "since last run" watermark; use a lookback window and idempotent upserts.
- Business logic duplicated between the app and the pipeline; share the domain functions or validate with the same schema.
- Orchestrator-specific code mixed into transforms, making them untestable; keep stages plain functions.

## Refactoring path
- **Introduce it:** split the monolithic script into extract/validate/transform/load functions with typed records; add tests per transform; make the load idempotent; add a watermark and dead-letter; then hand scheduling and retries to an orchestrator.
- **Remove it / simplify:** if the "pipeline" is just a copy between two tables in the same DB, replace it with an `INSERT … SELECT`, a view or a materialized view; if it's a one-off, keep it as a script.

## Related patterns
- [Pipes and Filters](../architectural/pipes-and-filters.md) — the architectural style each pipeline instantiates.
- [Dependency Graph](../emergent/dependency-graph.md) — orchestrators run pipeline tasks as a DAG.
- [Producer-Consumer](../concurrency/producer-consumer.md) — bounded queues between stages for backpressure.
- [Event Replay](../emergent/event-replay.md) — backfills replay history through the pipeline.
- [Splitter](../enterprise/splitter.md) — breaking files/batches into records for per-item processing.
