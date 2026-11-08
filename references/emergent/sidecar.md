# Sidecar

**Category:** Emergent / modern · **Also known as:** Sidekick, co-process, service-mesh proxy · **Related:** Ambassador, Decorator, Proxy, Microservices, Bulkhead

## Intent
Deploy a helper process next to the application process, sharing its lifecycle, network namespace and/or filesystem, to provide cross-cutting infrastructure (mTLS, log shipping, metrics, config reload, secrets) without changing or linking into the application.

## Problem
- Every service, in several languages, re-implements the same plumbing: TLS certificates, log forwarding, metrics export, secret rotation, auth token refresh.
- Scenario: a platform runs FastAPI, NestJS and a legacy Java service. Security mandates mTLS between all of them and certificate rotation every 24 hours. Adding a TLS library and rotation logic to each codebase means three implementations, three bug sets, and redeploying every service when the policy changes.
- Scenario: an app writes logs to local files; shipping them to Elasticsearch requires code nobody wants inside the app.
- Smells: a shared "platform SDK" pinned at different versions per service; infra changes that require application releases; infra logic that cannot be reused across languages.

## Solution and structure
Package the infrastructure concern as its own container/process, scheduled together with the app as one unit (a Kubernetes Pod, an ECS task, a systemd unit pair). They communicate over `localhost`, a shared volume, or a Unix socket.

```
┌──────────────── Pod ────────────────┐
│  app container      sidecar         │
│  :8000  ◄──localhost──► :15001       │◄── network (mTLS)
│  /var/log/app ──shared volume──► log shipper ──► Loki/ES
└─────────────────────────────────────┘
```
- **Application**: plain business code, speaks plain HTTP to localhost or writes to stdout/files.
- **Sidecar**: proxy, agent or watcher; versioned and released by the platform team independently.
- **Shared resources**: network namespace, volumes, lifecycle (start/stop together, scaled together).

Sidecars can sit on inbound traffic (auth, mTLS termination), outbound traffic (then it is also an Ambassador), or off the request path (log shipping, config sync).

## When to use
- Polyglot fleets where a concern must behave identically everywhere (mTLS, tracing, policy).
- Infra owned by a platform team that needs to roll out independently of app releases.
- Legacy or third-party apps you cannot modify but need to observe, secure or adapt.
- Concerns with a separate resource/failure profile (a log shipper must not crash the app).

## When NOT to use
- One service, one language: a library or middleware (FastAPI middleware, an Angular `HttpInterceptor`, a NestJS interceptor) is simpler and has no network hop.
- Latency-critical hot paths where an extra proxy hop (typically sub-millisecond to a few ms) matters.
- Small clusters where per-pod sidecar CPU/memory overhead multiplied by replicas exceeds the benefit; consider node-level agents (DaemonSet) or sidecar-less meshes (Istio ambient mode, Cilium) instead.
- Serverless functions: there is no pod to attach to (Lambda Extensions are the closest analogue).

## Example — Python
App side stays ignorant: it logs JSON to stdout and talks plain HTTP to the mesh. The sidecar is configured in the manifest (below).

```python
import logging, sys
import httpx
from fastapi import FastAPI
from pythonjsonlogger.json import JsonFormatter   # python-json-logger 3.x path

handler = logging.StreamHandler(sys.stdout)       # the log-shipper sidecar tails stdout/files
handler.setFormatter(JsonFormatter())
logging.basicConfig(level=logging.INFO, handlers=[handler])
log = logging.getLogger("orders")

app = FastAPI()
# Plain HTTP: the mesh sidecar transparently upgrades to mTLS and adds retries/tracing.
inventory = httpx.AsyncClient(base_url="http://inventory.shop.svc.cluster.local", timeout=2.0)

@app.post("/orders")
async def create_order(order: dict) -> dict:
    r = await inventory.post("/reservations", json={"sku": order["sku"], "qty": order["qty"]})
    r.raise_for_status()
    log.info("order created", extra={"sku": order["sku"]})
    return {"status": "created"}
```

## Example — TypeScript
Kubernetes manifest (YAML) adding a log-shipper sidecar using native sidecar containers, plus the Node app writing to the shared volume.

```yaml
apiVersion: v1
kind: Pod
metadata: { name: orders, labels: { app: orders } }
spec:
  initContainers:
    - name: log-shipper              # native sidecar: init container with restartPolicy Always
      image: fluent/fluent-bit:3.1
      restartPolicy: Always          # starts before app, stops after it (K8s >= 1.29 beta, GA 1.33)
      volumeMounts: [{ name: logs, mountPath: /var/log/app }]
  containers:
    - name: app
      image: registry.example.com/orders-node:1.4.2
      volumeMounts: [{ name: logs, mountPath: /var/log/app }]
      resources: { requests: { cpu: 200m, memory: 256Mi } }
  volumes: [{ name: logs, emptyDir: {} }]
```

```ts
import { createWriteStream } from 'node:fs';
import express from 'express';

const out = createWriteStream('/var/log/app/orders.log', { flags: 'a' });
const log = (msg: string, fields: Record<string, unknown> = {}) =>
  out.write(JSON.stringify({ ts: new Date().toISOString(), msg, ...fields }) + '\n');

const app = express();
app.use(express.json());
app.post('/orders', (req, res) => {
  log('order created', { sku: req.body.sku });
  res.status(201).json({ status: 'created' });
});
app.listen(8000);
```
For a mesh sidecar (Envoy), you usually add nothing to the manifest: the mesh injects it via a namespace label (e.g. `istio-injection=enabled`). Kubernetes versions above are from memory; check the K8s docs for your cluster.

## In the wild
- Service meshes: Istio/Envoy, Linkerd proxy, Consul Connect; Dapr sidecar (state, pub/sub, bindings over localhost HTTP/gRPC).
- Logging/metrics: Fluent Bit, Vector, OpenTelemetry Collector as sidecar; Vault Agent injector for secrets; cloud-sql-proxy for GCP Cloud SQL.
- AWS: ECS task containers with `dependsOn`, App Mesh Envoy (being retired in favour of ECS Service Connect — check AWS docs), Lambda Extensions as the serverless analogue.

## Trade-offs
| Pros | Cons |
|---|---|
| Language-agnostic; one implementation for the fleet | Extra CPU/memory per replica |
| Infra evolves without app releases | Extra hop and failure point on the request path |
| Isolates failures and resource limits from the app | Startup/shutdown ordering issues |
| Works for apps you cannot modify | Debugging spans two processes; more moving parts |

## Pitfalls
- Startup races: the app sends traffic before the proxy is ready, or the sidecar exits first on shutdown and in-flight requests fail. Use native sidecars (init container with `restartPolicy: Always`) or readiness gates / `holdApplicationUntilProxyStarts` in Istio.
- Jobs that never finish because the sidecar keeps running after the main container exits — native sidecars fix this; older clusters need a shutdown signal.
- Business logic creeping into the sidecar (routing by customer tier, data transformation). Keep it infrastructure-only.
- No resource requests/limits on the sidecar, so it starves the app under load.
- Double retries: app retries *and* sidecar retries multiply load. Configure retries in one place.

## Refactoring path
- **Introduce it:** identify plumbing duplicated across services; standardize the app-side contract (stdout logs, plain HTTP to localhost, files on a volume); deploy the sidecar to one service; remove the in-app implementation; roll out via injection.
- **Remove it / simplify:** if the fleet collapsed to one stack, move the concern into a shared middleware library; if per-pod overhead hurts, move to a node agent (DaemonSet) or a sidecar-less mesh.

## Related patterns
- [Ambassador](../emergent/ambassador.md) — a sidecar specialised for outbound calls on the client's behalf.
- [Decorator](../structural/decorator.md) — the in-process equivalent: wrap behavior without changing the core.
- [Proxy](../structural/proxy.md) — sidecars are often network proxies.
- [Microservices](../architectural/microservices.md) — the context where sidecars pay off.
- [Bulkhead](../cloud/bulkhead.md) — separate containers isolate resource failures.
