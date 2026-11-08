# Circuit Breaker

**Category:** Cloud / distributed · **Also known as:** Breaker, Fail-fast proxy · **Related:** Retry, Bulkhead, Rate Limiter, Proxy, Ambassador, Sidecar

## Intent
Wrap calls to a remote dependency in an object that tracks failures; when failures cross a threshold it "opens" and fails fast without calling the dependency, then after a cool-down lets a few trial calls through to detect recovery.

## Problem
- The payment provider is down. Every checkout request waits 30 s for a timeout, worker threads and DB connections pile up, and the whole API goes down with it (cascading failure).
- Retries multiply load on a dependency that is already struggling, delaying its recovery.
- Users wait for timeouts instead of getting an immediate "try again later" or a degraded response.

Scenario: a checkout service calls a fraud-check API. When the fraud API is slow or erroring, checkout should skip the call within milliseconds and fall back to "manual review" instead of hanging.

## Solution and structure
```
          failures >= threshold
 CLOSED ───────────────────────────► OPEN ──(reset timeout elapsed)──► HALF-OPEN
   ▲  calls pass, failures counted     │ calls fail fast                  │ allow N trial calls
   │                                   ◄───────── trial fails ────────────┤
   └──────────────── trial calls succeed ─────────────────────────────────┘
```
- **Breaker**: state machine (Closed / Open / Half-Open) keyed per dependency (often per host or per operation).
- **Failure policy**: what counts (timeouts, 5xx, connection errors; not 4xx validation errors), threshold as count or rate over a sliding window, minimum call volume.
- **Reset timeout**: how long to stay open before probing.
- **Fallback**: cached value, default, degraded feature, or a fast error to the caller.
- **Telemetry**: state changes must emit metrics/logs; an open breaker is an incident signal.

Ordering in the "resilient outbound call" stack: **timeout** inside **retry** inside **circuit breaker** inside **bulkhead** (the breaker sees one logical failure after retries are exhausted; some teams put retry outside instead so each attempt is counted, and either is valid if chosen deliberately).

## When to use
- Synchronous calls to remote dependencies that can fail or slow down independently (HTTP APIs, gRPC, databases, caches).
- A fallback exists, or failing fast is better than waiting.
- Many callers share the dependency, so one slow downstream could exhaust your resources.

## When NOT to use
- Local in-process calls or resources that fail deterministically (a bad input will fail every time; a breaker just hides the bug).
- Asynchronous message consumption: queues already decouple you; use retry with backoff, dead-letter queues and consumer pause instead.
- When a service mesh (Istio/Envoy outlier detection, Linkerd) already provides breaking at the network layer; do not stack a second, differently tuned breaker without reason.
- Very low traffic: a breaker needs enough calls to judge failure rate; a timeout plus retry may be enough.

## Example — Python
A compact async breaker with a failure-count threshold. Production libraries add sliding windows, half-open call limits and metrics.
```python
import asyncio, time
from collections.abc import Awaitable, Callable
from contextlib import nullcontext
from enum import Enum
import httpx

class State(Enum):
    CLOSED = "closed"; OPEN = "open"; HALF_OPEN = "half_open"

class CircuitOpenError(Exception): ...

class CircuitBreaker:
    def __init__(self, threshold: int = 5, reset_after: float = 30.0,
                 counts_as_failure: tuple[type[BaseException], ...] = (Exception,)) -> None:
        self.threshold, self.reset_after, self.counts = threshold, reset_after, counts_as_failure
        self.state, self.failures, self.opened_at = State.CLOSED, 0, 0.0
        self._probe = asyncio.Lock()                      # one trial call at a time in HALF_OPEN

    async def call[T](self, fn: Callable[[], Awaitable[T]]) -> T:
        if self.state is State.OPEN:
            if time.monotonic() - self.opened_at < self.reset_after:
                raise CircuitOpenError("fraud-api circuit open")
            self.state = State.HALF_OPEN
        if self.state is State.HALF_OPEN and self._probe.locked():
            raise CircuitOpenError("probe in progress")
        async with self._probe if self.state is State.HALF_OPEN else nullcontext():
            try:
                result = await fn()
            except self.counts:
                self.failures += 1
                if self.state is State.HALF_OPEN or self.failures >= self.threshold:
                    self.state, self.opened_at = State.OPEN, time.monotonic()
                raise
            self.state, self.failures = State.CLOSED, 0
            return result

fraud_breaker = CircuitBreaker(threshold=5, reset_after=20, counts_as_failure=(httpx.TransportError,))

async def fraud_check(client: httpx.AsyncClient, order: dict) -> str:
    try:
        r = await fraud_breaker.call(lambda: client.post("/score", json=order, timeout=2.0))
        return r.json()["decision"]
    except (CircuitOpenError, httpx.TransportError):     # TransportError includes timeouts
        return "manual_review"                            # fallback: degrade, don't fail checkout
```
Count only failures that indicate dependency health (transport errors, timeouts, 5xx), not your own 4xx bugs; to count 5xx, raise a dedicated exception for them inside `fn` and add it to `counts_as_failure`.

## Example — TypeScript
Node with `opossum`, the common breaker library. Its options and events have been stable for years, but check the README for the version you install.
```ts
import CircuitBreaker from 'opossum';

interface Order { id: string; total: number }
type Decision = 'approve' | 'reject' | 'manual_review';

async function scoreOrder(order: Order): Promise<Decision> {
  const res = await fetch('https://fraud.internal/score', {
    method: 'POST',
    body: JSON.stringify(order),
    headers: { 'content-type': 'application/json' },
    signal: AbortSignal.timeout(2000),
  });
  if (res.status >= 500) throw new Error(`fraud api ${res.status}`);  // only 5xx trips the breaker
  return ((await res.json()) as { decision: Decision }).decision;
}

export const fraudBreaker = new CircuitBreaker(scoreOrder, {
  timeout: 2500,                  // breaker-level timeout counts as failure
  errorThresholdPercentage: 50,   // open when >= 50% of calls in the rolling window fail
  volumeThreshold: 10,            // ...and at least 10 calls were made
  resetTimeout: 20_000,           // then try a half-open probe after 20 s
});
fraudBreaker.fallback((): Decision => 'manual_review');
fraudBreaker.on('open', () => console.warn('fraud breaker OPEN'));

// usage: const decision = await fraudBreaker.fire(order);
```
In an Angular client, a breaker is rarely worth it; use `retry({ count, delay })` with backoff plus a user-visible error state. The server or the mesh owns breaking.

## In the wild
- Python: `pybreaker`, `purgatory` (async), `aiobreaker`. (`tenacity` is retry, not a breaker.)
- JS: `opossum`, `cockatiel` (breaker + retry + bulkhead + timeout policies); NestJS commonly wraps one of these.
- JVM/.NET: Resilience4j, Polly; Netflix Hystrix (retired, origin of the popular model).
- Infrastructure: Envoy/Istio outlier detection and connection limits, Linkerd failure accrual, AWS App Mesh; AWS SDKs have adaptive retry modes but not full breakers.

## Trade-offs
| Pros | Cons |
|---|---|
| Stops cascading failures; frees threads and connections | Thresholds are hard to tune; false opens hurt availability |
| Fast failure and graceful fallbacks | Per-instance state: each replica opens independently |
| Gives the dependency room to recover | Adds a state machine to test and observe |
| Clear operational signal when a dependency is sick | Can hide errors if fallbacks are silent |

## Pitfalls
- **No timeout underneath.** A breaker counts failures; without a timeout a hung call never fails. Always set timeouts.
- **Counting client errors** (400, 404, validation) as failures opens the circuit on your own bugs.
- **One breaker for all endpoints** of a large service: one bad endpoint blocks healthy ones. Key per host + operation where it matters.
- **Unlimited half-open traffic**: all requests rush in at once and knock the recovering service over again. Allow one or a few probes.
- **Silent fallbacks** with no metric: the system "works" while a dependency is down for days. Alert on state changes.
- **Shared mutable breaker across threads** without synchronization in threaded code; use a library or a lock.

## Refactoring path
- **Introduce it:** first add timeouts to every outbound call. Then wrap the client of the dependency that has caused cascading incidents with a breaker in log-only mode, tune thresholds from observed error rates, then enforce and add a fallback. Put it in the client adapter, not scattered through business code.
- **Remove it / simplify:** if a service mesh now does outlier detection, remove the in-code breaker or keep only the fallback logic. If the dependency became async (queue), drop the breaker for retry + DLQ.

## Related patterns
- [Retry](../behavioral/retry.md): handles transient blips; the breaker stops retries from hammering a dead service.
- [Bulkhead](../cloud/bulkhead.md): limits how many resources a dependency can consume; complements the breaker.
- [Rate Limiter](../concurrency/rate-limiter.md): protects a healthy dependency from too much load.
- [Proxy](../structural/proxy.md): a breaker is a protection proxy around the client.
- [Ambassador](../emergent/ambassador.md) / [Sidecar](../emergent/sidecar.md): move breaking out of the app into an out-of-process proxy.
