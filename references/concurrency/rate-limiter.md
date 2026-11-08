# Rate Limiter

**Category:** Concurrency · **Also known as:** Throttling, Token bucket, Leaky bucket, Quota, Debounce/Throttle (client side) · **Related:** Bulkhead, Circuit Breaker, Balking, Scheduler, Retry

## Intent
Cap how often an operation may run per unit of time, per caller or globally, to protect a resource, enforce fair use, or stay inside a downstream quota.

## Problem
- One tenant's script sends 2,000 requests/second and degrades the API for everyone.
- A vendor API allows 100 requests/minute; the import job gets HTTP 429 and is temporarily banned.
- Login endpoints are brute-forced.
- On the client, a search box fires an HTTP request on every keystroke; a scroll handler runs 200 times per second.

## Solution and structure
Pick the algorithm by the behavior you want at the boundary:

| Algorithm | How it works | Bursts | Memory | Typical use |
|---|---|---|---|---|
| **Token bucket** | bucket of capacity B refills at rate r; each request takes a token | allows bursts up to B | 2 numbers per key | public APIs (AWS API Gateway, Stripe) |
| **Leaky bucket** (as queue) | requests enter a queue drained at constant rate r | smoothed; excess queued or dropped | queue per key | outbound calls to a strict downstream, nginx `limit_req` |
| **Fixed window** | counter per key per window (`INCR key:12:05`) | up to 2× limit at window edges | 1 counter | simple quotas, daily limits |
| **Sliding window log** | store timestamps, count those within the last W | exact | O(requests) per key | low-volume, exact limits (login attempts) |
| **Sliding window counter** | weighted mix of current and previous fixed windows | approximate, smooth | 2 counters | high-volume APIs (Cloudflare-style) |
| **GCRA** | token bucket expressed as a "theoretical arrival time" | like token bucket | 1 timestamp | `redis-cell`, many API gateways |

Participants: a **key** (user id, API key, IP, tenant, route), a **limit policy**, a **state store** (process memory, or Redis for multiple replicas), and a **rejection response** (HTTP 429 with `Retry-After`, or wait/queue for outbound calls).

**Distributed:** with N replicas, per-process limits allow N× the limit. Put the state in Redis and make the check-and-update atomic with a Lua script (or `INCR` + `EXPIRE` for fixed windows), or enforce at the gateway (API Gateway, Envoy, nginx, Kong).

**Client side:** **debounce** waits for a quiet period then fires once (search-as-you-type); **throttle** fires at most once per interval (scroll, resize); **audit** is throttle that emits the latest value at the end of the interval.

## When to use
- Inbound: protect your service from abuse and noisy tenants; enforce plan quotas.
- Outbound: stay under a third-party quota instead of reacting to 429s.
- UI: collapse bursts of events before doing expensive work.

## When NOT to use
- The problem is concurrency (too many simultaneous slow calls), not rate. Use a semaphore / Bulkhead.
- The downstream is failing, not overloaded by you. Use a Circuit Breaker.
- Internal batch work you control: process it with a bounded pool instead of throttling every call.
- Writing your own distributed limiter when the gateway or the platform already offers one.

## Example — Python
Async token bucket for outbound calls (waits instead of rejecting), plus the Redis fixed-window check for inbound limits across replicas.
```python
import asyncio, time
from redis.asyncio import Redis

class TokenBucket:
    def __init__(self, rate: float, capacity: int) -> None:
        self.rate, self.capacity = rate, capacity
        self._tokens = float(capacity)
        self._updated = time.monotonic()
        self._lock = asyncio.Lock()                 # serializes waiters: FIFO-ish fairness

    async def acquire(self) -> None:
        async with self._lock:
            while True:
                now = time.monotonic()
                self._tokens = min(self.capacity, self._tokens + (now - self._updated) * self.rate)
                self._updated = now
                if self._tokens >= 1:
                    self._tokens -= 1
                    return
                await asyncio.sleep((1 - self._tokens) / self.rate)  # time until the next token

vendor_limit = TokenBucket(rate=100 / 60, capacity=10)   # 100/min, bursts of 10

async def call_vendor(client, path: str):
    await vendor_limit.acquire()
    return await client.get(path)

async def allow(redis: Redis, key: str, limit: int, window_s: int) -> tuple[bool, int]:
    """Inbound fixed window shared by all replicas. Returns (allowed, retry_after_seconds)."""
    bucket = f"rl:{key}:{int(time.time() // window_s)}"
    async with redis.pipeline(transaction=True) as p:     # MULTI/EXEC: INCR and EXPIRE together
        count, _ = await p.incr(bucket).expire(bucket, window_s, nx=True).execute()
    return count <= limit, window_s - int(time.time() % window_s)
```
`expire(..., nx=True)` needs Redis 7+. For a distributed token bucket or sliding window, use a Lua script so the read-modify-write is atomic, or a library such as `limits` (used by `slowapi` for FastAPI). `aiolimiter` provides an asyncio leaky bucket.

## Example — TypeScript
Client side with RxJS in Angular: debounce the search box, throttle scroll.
```ts
import { Component, inject, signal } from '@angular/core';
import { HttpClient } from '@angular/common/http';
import { toObservable, toSignal } from '@angular/core/rxjs-interop';
import { debounceTime, distinctUntilChanged, filter, switchMap, catchError, of } from 'rxjs';

interface Product { id: string; name: string }

@Component({
  selector: 'app-product-search',
  template: `
    <input [value]="query()" (input)="query.set($any($event.target).value)" placeholder="Search" />
    @for (p of results(); track p.id) { <div>{{ p.name }}</div> }
  `,
})
export class ProductSearch {
  private http = inject(HttpClient);
  protected query = signal('');

  protected results = toSignal(
    toObservable(this.query).pipe(
      debounceTime(300),                 // wait for 300 ms of silence
      filter((q) => q.length >= 2),
      distinctUntilChanged(),
      switchMap((q) =>                   // cancel the previous in-flight request
        this.http.get<Product[]>('/api/products', { params: { q } }).pipe(catchError(() => of([]))),
      ),
    ),
    { initialValue: [] as Product[] },
  );
}
// Scroll: fromEvent(window, 'scroll').pipe(throttleTime(100, undefined, { leading: true, trailing: true }))
```
In Node/NestJS use `@nestjs/throttler` or `express-rate-limit` with a Redis store for multi-instance deployments.

## In the wild
- Python: `slowapi` / `limits` (FastAPI/Starlette), `aiolimiter`, `django-ratelimit`; Celery `rate_limit` task option.
- JS: RxJS `debounceTime`, `throttleTime`, `auditTime`, `sampleTime`; lodash `debounce`/`throttle`; `express-rate-limit`, `rate-limiter-flexible`, `@nestjs/throttler`, `bottleneck` (outbound).
- Infrastructure: nginx `limit_req` (leaky bucket), Envoy local/global rate limit, AWS API Gateway usage plans (token bucket), AWS WAF rate-based rules, Cloudflare rate limiting, Kong.
- Redis: `INCR`/`EXPIRE`, sorted sets for sliding logs, `redis-cell` (GCRA) module.

## Trade-offs
| Pros | Cons |
|---|---|
| Protects services from overload and abuse | Legitimate bursts can be rejected |
| Enforces fairness and plan quotas | Distributed limits add a Redis round-trip per request |
| Keeps you under third-party quotas | Choosing keys and limits needs real traffic data |
| Client-side debounce cuts wasted requests | Debounce adds perceived latency |

## Pitfalls
- **Per-process limits behind a load balancer** multiply the effective limit. Use a shared store or the gateway.
- **Non-atomic read-modify-write** in Redis (`GET` then `SET`). Use `INCR`, `MULTI`, or Lua.
- **Fixed-window edge bursts**: 2× the limit across a boundary. Use sliding window or token bucket if it matters.
- **Keying by IP only**: NAT and corporate proxies share IPs; attackers rotate them. Key by API key / user where possible.
- **No `Retry-After` / rate-limit headers**, so clients retry immediately and make it worse.
- **Fail-closed when Redis is down** takes the whole API down. Decide fail-open vs fail-closed explicitly.
- **Debounce vs throttle confusion.** Debounce for "after the user stops", throttle for "regularly while it continues". `switchMap` after debounce to drop stale responses.

## Refactoring path
- **Introduce it:** measure current per-key rates. Start at the edge (gateway or middleware) with generous limits in log-only mode, then enforce. For outbound quotas, wrap the client with a token bucket before you see 429s, and honor `Retry-After` on the ones you still get.
- **Remove it / simplify:** if a hand-rolled limiter duplicates what the gateway does, delete it. If the real problem was concurrency, replace it with a semaphore.

## Related patterns
- [Bulkhead](../cloud/bulkhead.md): limits concurrency, not rate; often used together.
- [Circuit Breaker](../cloud/circuit-breaker.md): stops calls to a failing dependency; rate limiting protects a healthy one.
- [Balking](../concurrency/balking.md): drops calls by state rather than by time.
- [Scheduler](../concurrency/scheduler.md): spreads work over time; a leaky bucket is a constant-rate scheduler.
- [Retry](../behavioral/retry.md): must respect the limiter and `Retry-After`, with backoff and jitter.
