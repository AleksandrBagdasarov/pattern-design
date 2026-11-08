# Sharding

**Category:** Cloud / distributed · **Also known as:** Horizontal partitioning, Partitioning (Kafka, DynamoDB), Cells (at service level) · **Related:** CQRS, Bulkhead, Lock Striping, Leader Election, Caching

## Intent
Split a dataset horizontally across multiple independent nodes (shards) by a shard key, so storage, write throughput and load scale out beyond what one node can handle.

## Problem
- The primary database is at its vertical limit: biggest instance size, write IOPS saturated, and replicas only help reads.
- A single table holds billions of rows; indexes no longer fit in memory, vacuum/backups take hours.
- Data residency rules require EU customers' data to live in EU storage.
- One noisy tenant's load affects every other tenant on the same database.

Scenario: a multi-tenant SaaS invoicing platform with 40k tenants writes 20k invoices/second at peak. One Postgres primary cannot keep up; almost all queries are scoped to one tenant.

## Solution and structure
```
             ┌──────────── router (app library, proxy, or the DB itself) ────────────┐
 request ──► │ shard_key = tenant_id → shard = f(shard_key)                            │
             └──────┬───────────────────┬───────────────────┬─────────────────────────┘
                    ▼                   ▼                   ▼
                shard 0             shard 1             shard 2   (each with its own replicas)
```
- **Shard key**: the attribute that decides placement. It should match the dominant access pattern (most queries include it) and have high cardinality with even load.
- **Placement strategy**:
  - **Hash**: `hash(key) mod N` spreads evenly but resharding moves most keys; **consistent hashing** or many **virtual shards** (e.g. 4096 logical shards mapped onto a few physical nodes) make rebalancing cheap.
  - **Range**: key ranges per shard; good for range scans and time series, prone to hot spots on the newest range.
  - **Directory / lookup**: a mapping table tenant → shard; flexible (move a big tenant alone) but the directory must be highly available and cached.
  - **Geo**: by region for latency or residency.
- **Router**: resolves the shard for each query.
- **Cross-shard operations**: scatter-gather queries, global secondary indexes, or a separate read model for analytics.

Not the same as **partitioning within one node** (PostgreSQL declarative partitioning), which helps maintenance and pruning but not write capacity beyond the node.

## When to use
- Write throughput or data size exceeds one node after you have optimized queries, added indexes, used read replicas and caching, and scaled up.
- Access is naturally scoped by a key (tenant, user, account, device).
- Tenant isolation or data residency is a requirement.

## When NOT to use
- Before exhausting simpler options: indexing, query fixes, connection pooling, read replicas, caching, archiving cold data, a bigger instance, table partitioning. Sharding is a one-way door with permanent complexity.
- Workloads full of cross-entity joins and transactions with no natural key (an ERP general ledger across all entities).
- Small datasets "for future scale".
- When a managed distributed database handles it transparently for your scale (DynamoDB, Spanner, CockroachDB, Aurora Limitless, Citus): choose the key well and let it shard.

## Example — Python
A directory-backed router with virtual shards: tenants hash to one of 1024 virtual shards, and a small map assigns virtual shards to physical databases. Moving a virtual shard changes one map entry.
```python
import hashlib
from dataclasses import dataclass
from sqlalchemy import create_engine, text
from sqlalchemy.engine import Engine

VIRTUAL_SHARDS = 1024

def virtual_shard(tenant_id: str) -> int:
    # stable across processes and versions (unlike built-in hash() for str)
    digest = hashlib.blake2b(tenant_id.encode(), digest_size=8).digest()
    return int.from_bytes(digest) % VIRTUAL_SHARDS

@dataclass
class ShardMap:
    engines: dict[str, Engine]                    # physical name -> engine
    assignment: dict[int, str]                    # virtual shard -> physical name (load from config/DB)
    overrides: dict[str, str]                     # big tenants pinned to a dedicated shard

    def engine_for(self, tenant_id: str) -> Engine:
        name = self.overrides.get(tenant_id) or self.assignment[virtual_shard(tenant_id)]
        return self.engines[name]

shards = ShardMap(
    engines={n: create_engine(f"postgresql+psycopg://app@{n}/invoices") for n in ("db-a", "db-b")},
    assignment={v: ("db-a" if v < 512 else "db-b") for v in range(VIRTUAL_SHARDS)},
    overrides={"tenant-megacorp": "db-b"},
)

def invoices_for(tenant_id: str, limit: int = 50) -> list[dict]:
    with shards.engine_for(tenant_id).connect() as conn:    # every query carries the shard key
        rows = conn.execute(
            text("SELECT id, total FROM invoices WHERE tenant_id = :t ORDER BY created_at DESC LIMIT :l"),
            {"t": tenant_id, "l": limit},
        )
        return [dict(r._mapping) for r in rows]
```
Keep `tenant_id` in every table's primary key and every query, even within a shard, so tenants can later be moved.

## Example — TypeScript
Kafka partitions are sharding for streams: the message key determines the partition, which gives per-key ordering and parallelism across consumers.
```ts
import { Kafka, CompressionTypes } from 'kafkajs';

interface InvoiceIssued { tenantId: string; invoiceId: string; total: number }

const kafka = new Kafka({ clientId: 'invoicing', brokers: process.env.KAFKA_BROKERS!.split(',') });
const producer = kafka.producer({ idempotent: true });
await producer.connect();

export async function publishIssued(e: InvoiceIssued): Promise<void> {
  await producer.send({
    topic: 'invoices.issued',            // e.g. 48 partitions = 48 shards
    compression: CompressionTypes.GZIP,
    messages: [{ key: e.tenantId, value: JSON.stringify(e) }], // same tenant -> same partition -> ordered
  });
}

// A consumer group spreads partitions across instances; one partition is read by one consumer at a time.
const consumer = kafka.consumer({ groupId: 'ledger-writer' });
await consumer.connect();
await consumer.subscribe({ topic: 'invoices.issued' });
await consumer.run({
  eachMessage: async ({ partition, message }) => {
    const e = JSON.parse(message.value!.toString()) as InvoiceIssued;
    console.log(`partition ${partition}: ${e.tenantId}/${e.invoiceId}`);
  },
});
```
`kafkajs` is in maintenance mode at the time of writing; Confluent's `@confluentinc/kafka-javascript` is the actively developed client. Check before choosing.

## In the wild
- Databases: Citus (Postgres), Vitess (MySQL), MongoDB sharded clusters, Cassandra/ScyllaDB (consistent hashing ring), DynamoDB and Cosmos DB partition keys, Spanner/CockroachDB (automatic range splits), Aurora Limitless, Elasticsearch shards.
- Streams: Kafka/Kinesis partitions by key.
- Caches: Redis Cluster (16384 hash slots), Memcached client-side consistent hashing.
- Architecture: AWS cell-based architecture, "silo per tenant" in SaaS.

## Trade-offs
| Pros | Cons |
|---|---|
| Scales writes and storage near-linearly | Cross-shard joins, transactions and uniqueness become hard |
| Smaller indexes and faster maintenance per shard | Choosing the wrong key is very expensive to undo |
| Failure and noisy neighbors contained per shard | Rebalancing and resharding are operational projects |
| Enables data residency and per-tenant isolation | More infrastructure, backups, migrations, monitoring |

## Pitfalls
- **Low-cardinality or skewed key** (country, status, created date): hot shards. Use high-cardinality keys, add a suffix for known hot keys, or pin whales to dedicated shards.
- **`hash(key) % N` with physical N**: adding a node remaps most keys. Use virtual shards or consistent hashing.
- **Python's built-in `hash()` for routing**: randomized per process for `str`. Use a stable hash.
- **Queries without the shard key** silently become scatter-gather to all shards. Make the key mandatory in the data access layer.
- **Global uniqueness and auto-increment IDs** across shards collide. Use UUIDv7/ULID or Snowflake-style IDs.
- **Schema migrations** must run on every shard; tooling must handle partial failure.
- **Cross-shard business transactions**: use a Saga or redesign aggregates to live on one shard.

## Refactoring path
- **Introduce it:** add the future shard key (`tenant_id`) to every table, key and query while still on one database. Route all DB access through one function that takes the key. Move to a distributed database or add physical shards behind that router, starting with a dual-write or logical replication migration per tenant. Build cross-tenant reporting on a separate read model.
- **Remove it / simplify:** consolidate shards when hardware growth outpaced data growth (fewer, larger shards behind the same router), or migrate to a managed distributed database and delete the routing layer.

## Related patterns
- [CQRS](../cloud/cqrs.md): a separate read model handles cross-shard queries and analytics.
- [Bulkhead](../cloud/bulkhead.md): shards double as isolation compartments for tenants.
- [Lock Striping](../concurrency/lock-striping.md): the same partition-by-key idea inside one process.
- [Leader Election](../cloud/leader-election.md): each shard's replica set elects its own primary.
- [Saga](../cloud/saga.md): coordinates business operations that span shards.
- [Caching](../optimization/caching.md): often enough to delay sharding for read-heavy load.
