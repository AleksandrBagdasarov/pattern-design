# Leader Election

**Category:** Cloud / distributed · **Also known as:** Single active instance, Coordinator election, Lease-based leadership · **Related:** Scheduler, Balking, Sharding, Leader-Follower, Saga

## Intent
Among several identical instances of a service, pick exactly one to perform a task that must not run concurrently (a scheduled job, a coordinator role, a single writer), and automatically pick a new one when it dies.

## Problem
- Scaling the API from 1 to 3 replicas made the "send daily digest" cron fire three times.
- A stream consumer or migration runner must be active on one node only, but the service runs in an autoscaling group.
- A hand-rolled "leader" flag in the DB stays set after the leader crashed, and nothing runs until someone clears it.
- Two nodes both believe they are leader after a network partition or a long GC pause (split brain) and both write.

Scenario: an order service has 4 replicas. One of them must poll the payment provider for settlement reports every 5 minutes and write them to the ledger.

## Solution and structure
```
 node A ──┐                       ┌── holds lease (leader) ──► runs the job, renews every T/3
 node B ──┼─ try acquire lease ──►│ lease store: key, owner, expiry, fencing token
 node C ──┘                       └── others: watch / retry; take over after expiry
```
- **Lease**: a record with owner id and expiry, stored in a strongly consistent store (etcd, ZooKeeper, Consul, Kubernetes `Lease`, a SQL row, or with caveats Redis).
- **Acquire**: atomic compare-and-set ("set if absent or expired").
- **Renew**: the leader extends its lease well before expiry; if renewal fails, it must **stop acting as leader immediately**.
- **Fencing token**: a monotonically increasing number returned with each acquisition; downstream writes include it, and the store rejects tokens lower than the highest seen. This is what actually prevents split-brain damage from a paused old leader.
- **Followers**: retry acquisition periodically or watch the key.

Consensus algorithms (Raft, Paxos, ZAB) implement election inside the coordination store; application code should rely on them, not reimplement them.

## When to use
- Singleton background work in a horizontally scaled service: schedulers, report pollers, cleanup jobs.
- A coordinator that assigns partitions or work to others.
- Active-passive failover for a component that cannot run active-active.

## When NOT to use
- The platform can run the job once for you: Kubernetes CronJob (`concurrencyPolicy: Forbid`), EventBridge Scheduler → one Lambda, Cloud Scheduler, Celery beat as a single deployment. Prefer that.
- Work can be partitioned: let every node take a share (Kafka consumer groups, `SELECT ... FOR UPDATE SKIP LOCKED` job claiming) instead of one leader doing everything.
- The job is idempotent and cheap: running it on every node with a dedup key may be simpler than election.
- You only need mutual exclusion for one short operation: a DB advisory lock or row lock in the same transaction is enough.

## Example — Python
PostgreSQL session-level advisory lock: simple and strongly consistent if you already have Postgres. The lock is released automatically when the session's connection dies.
```python
import asyncio, logging
import asyncpg

LOCK_ID = 7_420_001          # app-wide constant identifying "settlement poller"
log = logging.getLogger("leader")

async def poll_settlements(conn: asyncpg.Connection) -> None:
    ...                       # fetch reports and write them to the ledger

async def run_as_leader(dsn: str, interval: float = 300) -> None:
    while True:
        conn = await asyncpg.connect(dsn)                 # dedicated connection: the lock lives in its session
        try:
            if not await conn.fetchval("SELECT pg_try_advisory_lock($1)", LOCK_ID):
                await asyncio.sleep(30)                   # follower: retry later
                continue
            log.info("became leader")
            while True:
                await conn.fetchval("SELECT 1")           # liveness check: lost connection = lost leadership
                await poll_settlements(conn)
                await asyncio.sleep(interval)
        except (asyncpg.PostgresError, OSError):
            log.warning("lost leadership or DB connection; stepping down")
        finally:
            await conn.close()                            # closing the session releases the lock
```
Caveats: connection poolers in transaction mode (PgBouncer) break session-level locks; use a direct connection. Between a network failure and detection, the old leader may still act; make writes idempotent or fenced. For etcd/ZooKeeper, use their election recipes (`etcd3` client lease + `Election`, `kazoo` `Election`).

## Example — TypeScript
Redis lease with a token, safe release via Lua. Fine for "avoid duplicate cron runs"; not a correctness guarantee under partitions (see Pitfalls).
```ts
import { createClient } from 'redis';
import { randomUUID } from 'node:crypto';

const redis = createClient({ url: process.env.REDIS_URL });
await redis.connect();
const KEY = 'leader:settlement-poller';
const TTL_MS = 15_000;
const me = randomUUID();

const RENEW = `if redis.call('get', KEYS[1]) == ARGV[1] then
  return redis.call('pexpire', KEYS[1], ARGV[2]) else return 0 end`;

let isLeader = false;

async function tick(): Promise<void> {
  if (isLeader) {
    const ok = await redis.eval(RENEW, { keys: [KEY], arguments: [me, String(TTL_MS)] });
    isLeader = ok === 1;                       // renewal failed: step down at once
  } else {
    isLeader = (await redis.set(KEY, me, { NX: true, PX: TTL_MS })) === 'OK';
  }
}

setInterval(() => void tick().catch(() => (isLeader = false)), TTL_MS / 3);

export async function maybePollSettlements(poll: () => Promise<void>): Promise<void> {
  if (isLeader) await poll();                 // only the lease holder does the work
}
```
In Kubernetes, the `Lease` API (`coordination.k8s.io/v1`) is the native choice; client libraries in Go (`client-go/leaderelection`) are the reference implementation, and there are community ports for Node and Python.

## In the wild
- Coordination stores: etcd (lease + election API), ZooKeeper (ephemeral sequential znodes; Curator `LeaderLatch`), Consul sessions + KV lock, Kubernetes `Lease` objects (used by kube-controller-manager and operators).
- Databases: PostgreSQL `pg_try_advisory_lock`; DynamoDB lock client (conditional writes + lease).
- Built-in election: Kafka controller (KRaft), Redis Sentinel, MongoDB replica sets, Elasticsearch master election.
- Schedulers that avoid the need: Kubernetes CronJob, ShedLock (JVM), Quartz clustered mode, APScheduler with a shared job store (still needs care to avoid duplicates).

## Trade-offs
| Pros | Cons |
|---|---|
| Exactly one active worker for singleton tasks | Failover delay up to the lease TTL |
| Automatic failover when the leader dies | Split-brain risk without fencing |
| Uses infrastructure you often already run | Extra dependency on a consistent store |
| Simpler than distributed coordination of every step | Leader becomes a throughput bottleneck and hotspot |

## Pitfalls
- **Assuming a lease guarantees exclusivity.** A leader paused (GC, VM freeze) past its TTL wakes up and keeps writing. Use fencing tokens or idempotent, conditional writes.
- **Redis/Redlock for correctness-critical leadership.** It depends on timing assumptions; acceptable for efficiency (avoid duplicate work), not for safety. Use etcd/ZooKeeper/Postgres for safety.
- **Renewing too late** (at TTL − ε). Renew at about TTL/3 and stop work when renewal fails.
- **Doing long work without checking leadership** between steps. Check before each side effect.
- **Leader lock in a pooled connection** (Postgres advisory locks + PgBouncer transaction mode) silently does not work.
- **No observability**: expose "am I leader" as a metric and alert when no leader exists.

## Refactoring path
- **Introduce it:** when duplicate scheduled runs appear after scaling out, first ask whether the platform scheduler can run the job once. If not, add a lease on an existing consistent store, wrap the job in "only if leader", make side effects idempotent, then add fencing for writes that must not be duplicated.
- **Remove it / simplify:** move the job to a managed scheduler or partition the work across all nodes (queue with `SKIP LOCKED`, consumer groups) and delete the election code.

## Related patterns
- [Scheduler](../concurrency/scheduler.md): the most common reason to elect a leader.
- [Balking](../concurrency/balking.md): non-leaders balk; an advisory try-lock is a distributed balk.
- [Sharding](../cloud/sharding.md): partitioning work removes the need for one leader; each shard may still elect its own.
- [Leader-Follower](../concurrency/leader-follower.md): an unrelated in-process threading pattern with a similar name.
- [Saga](../cloud/saga.md): an orchestrator often runs as a single elected instance or as a durable workflow engine.
