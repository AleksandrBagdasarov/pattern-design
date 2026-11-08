# Retry

**Category:** Behavioral (resilience) · **Also known as:** Retry with backoff, Exponential backoff · **Related:** Circuit Breaker, Idempotent Receiver, Rate Limiter, Bulkhead, Decorator, Ambassador

## Intent
Re-attempt an operation that failed for a *transient* reason, waiting longer between attempts (backoff), randomizing the wait (jitter), and giving up after a bounded budget.

## Problem
- Network calls fail occasionally for reasons that fix themselves: a dropped TCP connection, a 503 during a deploy, a 429 throttle, a DB failover, a lock timeout.
- Without retry, one blip becomes a user-visible error or a failed batch job.
- Naive retry is worse than none: `while True: try ... except: continue` hammers a struggling service, every client retries at the same instant (thundering herd), and a non-idempotent `POST /payments` retried after a timeout charges the card twice.
- Smells: hand-written `for attempt in range(3)` loops copy-pasted around every client; `time.sleep(1)` constants; `except Exception` that also retries `ValueError` and 400s; incidents where retries multiplied load 3^n across service layers.

Scenario: a checkout service calls a payment gateway. The gateway returns 503 for ~2 s during its rolling deploys. You want checkout to survive that window, never double-charge, and not add load when the gateway is really down.

## Solution and structure
A retry policy wraps the call and decides, per failure, whether and when to try again.

```
caller ──> RetryPolicy ──> operation
              │  classify error: retryable? ── no ──> raise immediately
              │  budget left?    (attempts / total deadline) ── no ──> raise last error
              │  wait = min(cap, base * 2**attempt) * random()   ("full jitter")
              └─ sleep(wait), try again
```

Key parameters:
- **Classification.** Retry: connection errors, timeouts, 408, 429, 502, 503, 504, DB deadlocks/serialization failures. Do not retry: 400, 401, 403, 404, 409, 422, validation errors, programming errors. Honor `Retry-After` on 429/503.
- **Backoff.** Exponential (`base * 2**n`) spreads attempts out; linear is acceptable for short local waits.
- **Jitter.** Randomize the delay so synchronized clients desynchronize. "Full jitter" (`uniform(0, backoff)`) is the AWS-recommended default.
- **Caps.** A max single delay, a max attempt count, and ideally a total deadline that is smaller than the caller's own timeout.
- **Idempotency.** Only retry operations that are safe to repeat: GET/PUT/DELETE by design, or POST with an idempotency key the server deduplicates.

## When to use
- Calls across a network or process boundary with known transient failure modes.
- Idempotent operations, or operations made idempotent with a key.
- Background jobs and consumers, where latency matters less than eventual success.
- Optimistic-concurrency conflicts (`serialization_failure`) that succeed on re-read and re-apply.

## When NOT to use
- Errors that are deterministic (bad input, auth failure, 404): retrying only adds latency.
- Non-idempotent side effects without an idempotency key: fix idempotency first.
- When the dependency is down for minutes: put a Circuit Breaker in front so you fail fast instead of piling retries on.
- At every layer of a call chain. Retry at one layer (usually the outermost client or the edge); 3 retries × 3 layers = 27 calls per user request.
- Interactive requests with a tight latency budget: one fast retry at most, then fail.
- In-process code that "sometimes fails": that's a bug or a race, not a transient fault.

## Example — Python
Use `tenacity` rather than a hand-rolled loop. The idempotency key is generated once, outside the retried function, so every attempt sends the same key.

```python
import uuid
import httpx
from tenacity import (
    retry, retry_if_exception, stop_after_attempt, stop_after_delay,
    wait_random_exponential,
)

RETRYABLE_STATUS = {408, 429, 502, 503, 504}

def is_transient(exc: BaseException) -> bool:
    if isinstance(exc, (httpx.ConnectError, httpx.ReadTimeout, httpx.RemoteProtocolError)):
        return True
    return isinstance(exc, httpx.HTTPStatusError) and exc.response.status_code in RETRYABLE_STATUS

@retry(
    retry=retry_if_exception(is_transient),
    wait=wait_random_exponential(multiplier=0.2, max=5),   # full jitter, capped at 5 s
    stop=stop_after_attempt(5) | stop_after_delay(10),     # attempt cap and total deadline
    reraise=True,                                          # surface the real error, not RetryError
)
async def _post_charge(client: httpx.AsyncClient, body: dict, key: str) -> dict:
    resp = await client.post("/charges", json=body, headers={"Idempotency-Key": key}, timeout=3)
    resp.raise_for_status()
    return resp.json()

async def charge(client: httpx.AsyncClient, order_id: str, amount_cents: int) -> dict:
    key = f"charge-{order_id}-{uuid.uuid4()}"   # or derive deterministically from order_id
    return await _post_charge(client, {"order_id": order_id, "amount": amount_cents}, key)
```

Without a library, the same thing is a 10-line loop with `await asyncio.sleep(random.uniform(0, min(cap, base * 2**n)))`. `tenacity` also supports `wait_exponential_jitter` and `before_sleep` hooks for logging.

## Example — TypeScript
RxJS 7+ `retry` takes a config object (`retryWhen` is deprecated). Angular `HttpClient` observables are cold, so resubscribing re-sends the request.

```typescript
import { HttpClient, HttpErrorResponse } from '@angular/common/http';
import { Injectable, inject } from '@angular/core';
import { Observable, throwError, timer } from 'rxjs';
import { retry } from 'rxjs/operators';

const RETRYABLE = new Set([0, 408, 429, 502, 503, 504]); // 0 = network error / CORS / offline

function backoff(maxDelayMs = 5000, baseMs = 200) {
  return (error: unknown, retryCount: number): Observable<number> => {
    if (!(error instanceof HttpErrorResponse) || !RETRYABLE.has(error.status)) {
      return throwError(() => error);                 // non-retryable: stop now
    }
    const retryAfter = Number(error.headers?.get('Retry-After'));
    const exp = Math.min(maxDelayMs, baseMs * 2 ** retryCount);
    const delay = retryAfter > 0 ? retryAfter * 1000 : Math.random() * exp; // full jitter
    return timer(delay);
  };
}

@Injectable({ providedIn: 'root' })
export class OrdersApi {
  private readonly http = inject(HttpClient);

  getOrder(id: string) {
    return this.http.get<Order>(`/api/orders/${id}`).pipe(
      retry({ count: 4, delay: backoff() }),
    );
  }
}
interface Order { id: string; total: number }
```

## In the wild
- Python: `tenacity`, `backoff`, `urllib3.util.Retry` (via `requests` `HTTPAdapter`), `httpx` transport `retries=` (connect errors only), `stamina` (opinionated tenacity wrapper).
- AWS SDKs (boto3 `retries={"mode": "standard"|"adaptive"}`, JS SDK v3 `maxAttempts`): already retry with jitter; don't wrap another retry loop around them.
- Celery `autoretry_for`, `retry_backoff`, `retry_jitter`; SQS visibility timeout + redrive policy to a DLQ; Kafka consumer retry topics.
- RxJS `retry({ count, delay, resetOnSuccess })`; Angular `HttpClient` + `retry`; `p-retry` / `async-retry` in Node.
- Service meshes (Envoy, Istio `retries`), gRPC retry policy in service config: retry at the infrastructure layer (Ambassador/Sidecar).
- Stripe and many payment APIs: `Idempotency-Key` header precisely so clients can retry POSTs.

## Trade-offs
| Pros | Cons |
|---|---|
| Hides short transient failures from users | Adds tail latency (worst case = sum of delays) |
| Simple to add as a decorator/operator | Amplifies load during real outages without a breaker |
| Jitter smooths synchronized load spikes | Requires idempotency, which may need server changes |
| Policy is configurable and testable in one place | Can mask real bugs if classification is too broad |

## Pitfalls
- **Retrying non-idempotent writes.** Fix: idempotency key generated *once per logical operation*, stored server-side with the result (see Idempotent Receiver).
- **Generating the idempotency key inside the retried function.** Every attempt gets a new key, so deduplication never works. Fix: create it before the retry boundary.
- **No jitter.** Thousands of clients retry at t=1s, 2s, 4s in lockstep. Fix: full jitter.
- **Retry inside retry.** SDK retries + your retries + mesh retries multiply. Fix: pick one layer; disable the others or reduce their counts.
- **Total time exceeds the caller's timeout.** The caller has already given up while you still retry. Fix: a total deadline (`stop_after_delay`) shorter than the upstream timeout.
- **`except Exception` classification.** Retries `KeyError` and 400s. Fix: an explicit allowlist of transient errors.
- **`reraise=False` in tenacity.** Callers get `RetryError` and lose the original exception type. Fix: `reraise=True` or unwrap.
- **Retrying hot observables / shared streams in RxJS.** `retry` resubscribes; on a hot or `shareReplay`ed source it does not re-run the request. Fix: put `retry` on the cold HTTP observable, before sharing.
- **Silent retries.** Fix: log/metric every retry (`before_sleep`), so a degrading dependency is visible before it fails outright.

## Refactoring path
- **Introduce it:** find copy-pasted try/sleep loops around clients. (1) Centralize classification in one `is_transient` function. (2) Replace each loop with a shared decorator/operator configured with caps and jitter. (3) Audit each wrapped call for idempotency; add keys where needed. (4) Add retry metrics. (5) Add a Circuit Breaker if the dependency has long outages.
- **Remove it / simplify:** if retries never succeed in metrics (every retried call eventually fails), the errors are not transient; remove the retry and fail fast. If an SDK or mesh already retries, delete your outer layer.

## Related patterns
- [Circuit Breaker](../cloud/circuit-breaker.md) — stops retrying altogether when failure is sustained; use both together, breaker outside retry.
- [Idempotent Receiver](../emergent/idempotent-receiver.md) — the server-side half that makes retries safe.
- [Rate Limiter](../concurrency/rate-limiter.md) — 429 handling and client-side throttling; retries must respect it.
- [Bulkhead](../cloud/bulkhead.md) — caps concurrent retries so one slow dependency cannot exhaust all workers.
- [Decorator](../structural/decorator.md) — the usual implementation shape (`@retry`).
- [Ambassador](../emergent/ambassador.md) — moves retry out of process into a proxy or mesh.
- [Command](command.md) — a request reified as an object can be queued and retried later.
