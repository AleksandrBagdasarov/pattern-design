# Single Access Point

**Category:** Security · **Also known as:** Single Point of Entry, Gatekeeper, API Gateway (at network level), Check Point (related) · **Related:** Authorization, Secure Proxy, Intercepting Validator, Front Controller, Facade, Chain of Responsibility, Ambassador, Sidecar

## Intent
Force every external request into a system (or subsystem) through one well-defined entry point, where authentication, authorization, validation, rate limiting and auditing happen once, so there is no side door that skips them.

## Problem
- Security checks live in each handler or each service, and one of them is always missing.
- The system has several entry points (REST, admin panel, webhooks, internal debug endpoint, a gRPC port) with different levels of protection; attackers use the weakest.
- Scenario: 12 microservices each validate JWTs with their own library version. One still accepts `alg: none`; another exposes `/actuator` publicly. A gateway that verifies tokens and only routes declared paths removes both issues at the edge.
- Smell: "which ports/paths are reachable from the internet?" has no quick answer.

## Solution and structure
- **Access point**: the single entry (API gateway, reverse proxy, front controller, app-level middleware stack, login page for sessions).
- **Checks at the access point**: TLS termination, authentication, coarse authorization (route/scope), schema validation, rate limiting, request size limits, audit logging, correlation IDs.
- **Protected services**: not reachable except through the access point (private network, security groups, mTLS from the gateway only).
- **Defense in depth**: services still enforce object-level authorization; the access point does not replace it.

```
internet ─► [ Access point: TLS │ authN │ rate limit │ validate │ route ] ─► orders-svc
                                                                       └─► billing-svc
                         ✗ no direct route from internet to services (private subnet / mTLS)
```

Within one application the same idea applies: one middleware/dependency chain (router-level) that all routes inherit, rather than per-route opt-in.

## When to use
- Multiple services or modules are exposed externally and must share auth, limits and logging.
- Compliance needs a single place for audit and access logs.
- You want to shrink the attack surface: one hardened, monitored component instead of many.

## When NOT to use
- A single small service: the app's own middleware is the access point; adding a separate gateway adds a hop and an ops burden.
- Never rely on it *alone* for object-level checks: a gateway knows "this token has `orders:read`", not "this order belongs to this user".
- Don't put business logic in the gateway (pricing, orchestration); it becomes a distributed monolith's bottleneck. Keep it to cross-cutting concerns.
- If internal services can be called directly (same VPC without network policy), the "single" point is fiction. Fix the network first.

## Example — Python
One FastAPI app whose routers all inherit a mandatory security dependency. Public routes are an explicit, short allowlist rather than per-route opt-in.

```python
from typing import Annotated
from fastapi import APIRouter, Depends, FastAPI, HTTPException, Request
from fastapi.security import HTTPAuthorizationCredentials, HTTPBearer

bearer = HTTPBearer(auto_error=False)

async def authenticate(request: Request,
                       creds: Annotated[HTTPAuthorizationCredentials | None, Depends(bearer)]) -> None:
    if creds is None:
        raise HTTPException(401, headers={"WWW-Authenticate": "Bearer"})
    principal = verify_jwt(creds.credentials)      # signature, exp, aud, iss, allowed algs
    request.state.principal = principal            # downstream handlers read it from here

# Every protected router is mounted through this single gate.
protected = APIRouter(prefix="/api", dependencies=[Depends(authenticate)])
public = APIRouter(prefix="/public")               # explicit allowlist: health, login, webhooks*

@public.get("/health")
async def health() -> dict[str, str]:
    return {"status": "ok"}

@protected.get("/orders/{order_id}")
async def get_order(order_id: str, request: Request) -> dict[str, str]:
    return {"id": order_id, "viewer": request.state.principal.user_id}

app = FastAPI(docs_url=None, redoc_url=None, openapi_url=None)  # no public docs in prod
app.include_router(public)
app.include_router(protected)
# * webhooks are public routes but must verify the provider's HMAC signature themselves.
# verify_jwt: your token verification (e.g. PyJWT/joserfc with a pinned algorithm list).

def test_every_api_route_is_protected() -> None:   # guard against side doors
    # route.dependant is a FastAPI internal; adjust if your FastAPI version changes it.
    for route in app.routes:
        if getattr(route, "path", "").startswith("/api"):
            assert any(d.call is authenticate for d in route.dependant.dependencies)
```

## Example — TypeScript
In Angular, the single access point for outbound API calls is a functional HTTP interceptor: tokens are attached in one place, and 401s are handled in one place.

```typescript
import { HttpErrorResponse, HttpInterceptorFn } from '@angular/common/http';
import { inject } from '@angular/core';
import { Router } from '@angular/router';
import { catchError, throwError } from 'rxjs';
import { AuthStore } from './auth.store';
import { environment } from '../environments/environment';

export const apiGateInterceptor: HttpInterceptorFn = (req, next) => {
  const auth = inject(AuthStore);
  const router = inject(Router);
  // Attach credentials only to our own API: never leak tokens to third-party hosts.
  if (!req.url.startsWith(environment.apiUrl)) return next(req);

  const token = auth.accessToken();
  const authed = token ? req.clone({ setHeaders: { Authorization: `Bearer ${token}` } }) : req;
  return next(authed).pipe(
    catchError((err: unknown) => {
      if (err instanceof HttpErrorResponse && err.status === 401) {
        auth.clear();
        router.navigate(['/login'], { queryParams: { returnUrl: router.url } });
      }
      return throwError(() => err);
    }),
  );
};

// app.config.ts
// providers: [provideHttpClient(withInterceptors([apiGateInterceptor]))]
```

The real access point is server-side; this interceptor only makes the client consistent.

## In the wild
- Gateways: AWS API Gateway (+ Cognito/JWT/Lambda authorizers), AWS ALB with OIDC, Azure API Management, Kong, Apigee, Traefik, NGINX, Envoy; Kubernetes Ingress / Gateway API.
- Identity: a single login (OIDC provider: Keycloak, Auth0, Cognito, Entra ID) as the only place credentials are entered; BFF (backend-for-frontend) pattern for SPAs.
- App-level: FastAPI router/app `dependencies=[...]`, Starlette/Express middleware stacks, NestJS global guards (`APP_GUARD`), Django middleware; Angular `withInterceptors`.
- Network: bastion hosts / AWS SSM Session Manager as the single admin entry; private subnets + security groups.

## Trade-offs
| Pros | Cons |
|---|---|
| One place to harden, monitor and audit | Single point of failure; needs HA and capacity planning |
| Consistent authN, limits and logging | Extra network hop and latency |
| Smaller attack surface | Can become a bottleneck for teams (shared config) |
| Services stay simpler | False sense of safety if services skip object-level checks |

## Pitfalls
- **Side doors**: debug endpoints, metrics/actuator, direct service DNS, old API versions, open admin ports. Fix: network policy so only the gateway reaches services; an automated route inventory.
- **Opt-in protection per route** (`@login_required` forgotten on one). Fix: protect by default at router/app level, allowlist public routes.
- **Trusting headers set by the gateway** (`X-User-Id`) while services are also reachable directly. Fix: mTLS or signed tokens between gateway and services; services verify.
- **Gateway does all authorization**, services none: one misconfigured route exposes everything. Fix: object-level checks in services.
- **Webhooks bypass auth** without signature verification. Fix: verify HMAC/signatures and timestamps.

## Refactoring path
- **Introduce it:** inventory all entry points (ports, paths, hosts). Put a gateway or global middleware in front; move authN and rate limiting there; make services private; add a test or scanner that fails on unprotected routes; keep object-level authz in services.
- **Remove it / simplify:** if a separate gateway fronts one service with only JWT checks, fold the checks into the app's global middleware and remove the hop, keeping protected-by-default routing.

## Related patterns
- [Authorization](authorization.md) — fine-grained decisions behind the access point.
- [Secure Proxy](secure-proxy.md) — guards one resource; Single Access Point guards the whole system.
- [Intercepting Validator](intercepting-validator.md) — input validation often runs at the access point.
- [Front Controller](../ui/front-controller.md) — the in-app form: one handler dispatching all requests.
- [Facade](../structural/facade.md) — a gateway often also presents a simplified API over many services.
- [Chain of Responsibility](../behavioral/chain-of-responsibility.md) — the middleware chain inside the access point.
- [Ambassador](../emergent/ambassador.md), [Sidecar](../emergent/sidecar.md) — per-service proxies that complement an edge gateway.
