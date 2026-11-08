# Ambassador

**Category:** Emergent / modern · **Also known as:** Out-of-process client proxy, egress proxy, local proxy · **Related:** Sidecar, Proxy, Circuit Breaker, Retry, Adapter

## Intent
Put a helper proxy, co-located with the client, between the client and a remote service, so that connection concerns (retries, timeouts, circuit breaking, TLS, auth, routing, sharding, monitoring) are handled outside the application code and in one consistent place.

## Problem
- The app needs resilient outbound calls, but the resilience logic is tangled into business code and varies by language and team.
- Scenario: a legacy PHP service and a new FastAPI service both call a rate-limited partner API that requires OAuth client-credentials, mTLS and a regional endpoint. Each has its own token cache, retry policy and timeout; one hammers the partner during an incident because its retries have no backoff.
- Scenario: an app talks to a sharded Redis/Memcached; the client must know the shard map. Changing the map means redeploying every client.
- Smells: copy-pasted retry/backoff code; partner credentials in many apps; legacy code you cannot change that needs TLS or new auth.

## Solution and structure
Run a proxy next to the client (same pod/host, as a sidecar, or as a local daemon). The app talks to `localhost:PORT` in the simplest protocol; the ambassador handles the remote side.

```
 app ──plain HTTP──► ambassador (localhost:9000) ──TLS+OAuth+retry+CB──► partner API
                       │  timeouts, retries w/ jitter, circuit breaker,
                       │  token refresh, metrics, shard routing
```
- **Client**: calls a local endpoint; stays simple; may still set an overall deadline.
- **Ambassador**: Envoy/Nginx/HAProxy/a small custom proxy; configured per remote service.
- **Remote service**: unaware; sees well-behaved traffic.

Ambassador vs Sidecar: Sidecar is the deployment shape (co-located helper); Ambassador is a role (client-side proxy for outbound calls). Most ambassadors are deployed as sidecars. Ambassador vs API gateway: a gateway sits in front of the *server* for many clients; an ambassador sits next to *one client*.

## When to use
- Polyglot or legacy clients that need uniform resilience/auth/TLS for outbound calls.
- Partner APIs with complex connection requirements (mTLS, token exchange, request signing, egress IP allow-listing).
- Client-side routing that should change without client redeploys (sharding, canary, regional failover).
- Central visibility of outbound traffic (latency, error rates per dependency).

## When NOT to use
- One service in one language calling a dependency: use an in-process client wrapper (httpx with `tenacity`, a typed client class, Angular `HttpInterceptor` + RxJS `retry`). No extra hop, easier to debug.
- When a service mesh already gives every pod an outbound proxy: configure it (Istio `DestinationRule`/`VirtualService`) instead of adding another.
- When the client needs domain-aware decisions (retry only some business errors, fall back to cached domain data): that logic belongs in the app.
- Browser clients: you cannot co-deploy a proxy; use a BFF/backend proxy instead.

## Example — Python
Minimal ambassador (httpx + Starlette) that adds OAuth, timeout, retry with jitter; the app just calls `http://localhost:9000/...`. In production prefer Envoy; this shows the responsibilities.

```python
import asyncio, random, time
import httpx
from starlette.applications import Starlette
from starlette.requests import Request
from starlette.responses import Response
from starlette.routing import Route

UPSTREAM = "https://api.partner.example.com"
client = httpx.AsyncClient(base_url=UPSTREAM, timeout=httpx.Timeout(2.0), cert=("/certs/c.pem", "/certs/k.pem"))
_token: tuple[str, float] = ("", 0.0)

async def token() -> str:
    global _token
    if time.monotonic() > _token[1]:
        r = await client.post("/oauth/token", data={"grant_type": "client_credentials"})
        body = r.raise_for_status().json()
        _token = (body["access_token"], time.monotonic() + body["expires_in"] - 30)
    return _token[0]

async def forward(req: Request) -> Response:
    body = await req.body()
    for attempt in range(3):
        try:
            r = await client.request(req.method, req.url.path, params=req.query_params, content=body,
                                     headers={"Authorization": f"Bearer {await token()}",
                                              "Content-Type": req.headers.get("content-type", "")})
            if r.status_code < 500 and r.status_code != 429:
                return Response(r.content, r.status_code, media_type=r.headers.get("content-type"))
        except httpx.TransportError:
            pass
        await asyncio.sleep(min(2.0, 0.1 * 2**attempt) * random.random())   # full jitter
    return Response(b'{"error":"upstream unavailable"}', 503, media_type="application/json")

app = Starlette(routes=[Route("/{path:path}", forward, methods=["GET", "POST", "PUT", "DELETE"])])
```
Only retry non-idempotent methods if the upstream honours an idempotency key.

## Example — TypeScript
Envoy as the ambassador (config excerpt) and the Node client that stays trivial.

```yaml
# envoy.yaml (excerpt) — listener on 127.0.0.1:9000 routes to the partner cluster
route_config:
  virtual_hosts:
    - name: partner
      domains: ["*"]
      routes:
        - match: { prefix: "/" }
          route:
            cluster: partner_api
            timeout: 3s
            retry_policy: { retry_on: "5xx,reset,connect-failure", num_retries: 2,
                            per_try_timeout: 1s }
clusters:
  - name: partner_api
    type: LOGICAL_DNS
    circuit_breakers: { thresholds: [{ max_connections: 100, max_pending_requests: 50 }] }
    outlier_detection: { consecutive_5xx: 5, base_ejection_time: 30s }
    transport_socket: { name: envoy.transport_sockets.tls }   # mTLS details omitted
```

```ts
// The app knows only the local ambassador.
const PARTNER = process.env.PARTNER_URL ?? 'http://127.0.0.1:9000';

export async function getQuote(sku: string, signal?: AbortSignal): Promise<{ price: number }> {
  const res = await fetch(`${PARTNER}/quotes/${encodeURIComponent(sku)}`, { signal });
  if (!res.ok) throw new Error(`quote failed: ${res.status}`);
  return (await res.json()) as { price: number };
}
```

## In the wild
- Envoy/Istio egress and per-pod outbound proxy; Linkerd; Dapr service invocation; Consul Connect upstreams.
- `cloud-sql-proxy` / AlloyDB Auth Proxy (IAM auth + TLS to Cloud SQL), AWS RDS Proxy (pooling, though deployed centrally), Twemproxy/mcrouter for cache sharding, PgBouncer as a local connection pooler.
- Microsoft Azure Architecture Center documents "Ambassador" by this name.

## Trade-offs
| Pros | Cons |
|---|---|
| Uniform resilience/auth across languages | Extra hop and latency, another process to run |
| Change policy without redeploying clients | Error semantics get lossy (503 from proxy vs upstream) |
| Legacy apps gain TLS/auth/retries unmodified | Retries invisible to app can violate idempotency |
| Central metrics per dependency | Configuration sprawl; needs its own testing |

## Pitfalls
- Retrying in both the app and the ambassador: attempts multiply (3 × 3 = 9). Choose one layer; keep the app's deadline as the outer bound.
- Retrying `POST` without idempotency keys — duplicate payments. Restrict retries to idempotent methods or require keys.
- The ambassador's timeout longer than the client's: the client gives up while the proxy keeps retrying. Make per-try and total timeouts nest correctly.
- Putting business fallbacks into proxy config; keep domain decisions in the app.
- Treating it as a single shared box: a central proxy is a gateway, not an ambassador, and becomes a bottleneck/SPOF.

## Refactoring path
- **Introduce it:** inventory outbound calls and their duplicated retry/auth code; stand up the proxy for one dependency; point the client's base URL to localhost; delete the in-app retry/auth for that dependency; repeat.
- **Remove it / simplify:** if only one client remains, fold the policy back into an in-process client wrapper with `tenacity`/RxJS `retry` and drop the proxy hop.

## Related patterns
- [Sidecar](../emergent/sidecar.md) — the deployment shape most ambassadors use.
- [Proxy](../structural/proxy.md) — the in-process version of the same idea.
- [Circuit Breaker](../cloud/circuit-breaker.md) and [Retry](../behavioral/retry.md) — the policies an ambassador typically hosts.
- [Adapter](../structural/adapter.md) — when the ambassador also translates protocols.
- [Idempotent Receiver](../emergent/idempotent-receiver.md) — required upstream for safe proxy-level retries.
