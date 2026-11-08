# Authorization (RBAC / ABAC / policy-based)

**Category:** Security · **Also known as:** Access Control, Policy Enforcement Point / Policy Decision Point (PEP/PDP), Permission Check · **Related:** Secure Proxy, Single Access Point, Intercepting Validator, Specification, Chain of Responsibility, Decorator, Front Controller

## Intent
Decide *who may do what to which resource* in one place (the policy), and enforce that decision consistently at every entry point, so no handler can forget or reimplement the check differently.

## Problem
- Checks like `if user.role == "admin" or order.owner_id == user.id` are copied into dozens of handlers, each slightly different.
- One forgotten check is a breach: Broken Object Level Authorization (IDOR, "change the ID in the URL") is #1 in the OWASP API Security Top 10.
- Rules change (a new "support" role may read but not refund), and nobody can list where to update them.
- Scenario: `GET /orders/{id}` checks ownership, but the newer `GET /orders/{id}/invoice.pdf` was added by another team without it. Any logged-in user can download any invoice by iterating IDs.

## Solution and structure
Separate four roles (XACML vocabulary, used informally):
- **PEP (enforcement point)**: where the request is stopped: a route dependency, guard, middleware, or service method decorator. Every protected action passes through one.
- **PDP (decision point)**: a pure function `can(subject, action, resource, context) -> allow/deny`. No I/O inside if possible; data is loaded before.
- **PIP (information point)**: loads attributes the PDP needs (user roles, resource owner, tenant).
- **PAP (administration)**: where policies are defined (code, config, a policy engine).

Models for the decision:
- **RBAC**: permissions attach to roles, roles to users. Simple, auditable; breaks down for "own resources only".
- **ABAC**: decisions on attributes of subject, resource, action and environment (`resource.tenant == user.tenant and amount < user.limit`).
- **ReBAC** (relationship-based, Google Zanzibar): "user is editor of folder that contains doc". Fits sharing/collaboration.
- **Policy-as-code**: rules in a dedicated language/engine (OPA/Rego, Cedar, Casbin, Oso, OpenFGA).

```
request ─► authN (who) ─► PEP ──(subject, action, resource)──► PDP: policy ─► allow / deny (403/404)
                           ▲                                      ▲
                           └──── PIP loads resource & attributes ─┘
```

Rules: **deny by default**, check on the **server** (UI checks are only UX), check **object-level** access, not just "is logged in", and log denials.

## When to use
- Any system with more than one kind of user or any per-resource ownership.
- Multi-tenant SaaS: tenant isolation is an authorization rule enforced on every query.
- Rules change independently of features, or must be audited ("who can approve refunds?").

## When NOT to use
- A single-user tool or internal script with no untrusted callers: authentication alone may suffice.
- Don't build a full policy engine for three roles and five rules. Start with an in-code PDP (a function per resource type); adopt OPA/Cedar/OpenFGA when rules are numerous, shared across services, or must be edited without deploys.
- Don't use authorization as input validation: shape/format checks belong to [Intercepting Validator](intercepting-validator.md).

## Example — Python
FastAPI: the PDP is a pure function; the PEP is a dependency that loads the resource and authorizes before the handler body runs.

```python
from dataclasses import dataclass
from enum import StrEnum
from typing import Annotated
from fastapi import Depends, FastAPI, HTTPException, status

class Action(StrEnum):
    READ = "order:read"; REFUND = "order:refund"

ROLE_PERMS: dict[str, frozenset[Action]] = {                      # RBAC part
    "customer": frozenset({Action.READ}),
    "support":  frozenset({Action.READ}),
    "finance":  frozenset({Action.READ, Action.REFUND}),
}

@dataclass(frozen=True)
class Principal: user_id: str; tenant_id: str; roles: frozenset[str]

@dataclass(frozen=True)
class Order: id: str; tenant_id: str; owner_id: str; total_cents: int

def can(p: Principal, action: Action, order: Order) -> bool:       # PDP: pure, unit-testable
    if order.tenant_id != p.tenant_id:                             # ABAC: tenant isolation first
        return False
    if not any(action in ROLE_PERMS.get(r, frozenset()) for r in p.roles):
        return False
    if p.roles == {"customer"}:                                    # customers: own orders only
        return order.owner_id == p.user_id
    if action is Action.REFUND and order.total_cents > 100_000:
        return False                                               # large refunds need another flow
    return True

def authorized(action: Action):                                    # PEP factory
    async def dep(order_id: str, p: Annotated[Principal, Depends(current_principal)],
                  repo: Annotated[OrderRepo, Depends(get_order_repo)]) -> Order:
        order = await repo.get(order_id)
        if order is None or not can(p, action, order):
            raise HTTPException(status.HTTP_404_NOT_FOUND)         # don't reveal existence
        return order
    return dep

app = FastAPI()

@app.post("/orders/{order_id}/refund")
async def refund(order: Annotated[Order, Depends(authorized(Action.REFUND))]) -> dict[str, str]:
    return {"status": "refund-started", "order": order.id}
# current_principal, OrderRepo, get_order_repo: auth and data-access dependencies.
```

## Example — TypeScript
Angular functional guard and a permission signal for UX. This only hides UI; the API must enforce the same rule.

```typescript
import { Injectable, computed, inject, signal } from '@angular/core';
import { CanMatchFn, Router } from '@angular/router';

export type Permission = 'order:read' | 'order:refund' | 'admin:users';
interface Session { userId: string; permissions: readonly Permission[]; }

@Injectable({ providedIn: 'root' })
export class AuthzService {
  private readonly session = signal<Session | null>(null);  // set after login from /api/me
  readonly permissions = computed(() => new Set(this.session()?.permissions ?? []));
  setSession(s: Session | null) { this.session.set(s); }
  can(p: Permission): boolean { return this.permissions().has(p); }  // deny by default
}

export const requirePermission = (p: Permission): CanMatchFn => () => {
  const authz = inject(AuthzService);
  return authz.can(p) || inject(Router).createUrlTree(['/forbidden']);
};

// routes: { path: 'admin/users', canMatch: [requirePermission('admin:users')],
//           loadComponent: () => import('./admin/users').then(m => m.Users) }
// template: @if (authz.can('order:refund')) { <button (click)="refund()">Refund</button> }
```

## In the wild
- FastAPI `Depends`/`Security` with OAuth2 scopes; Django permissions and `django-guardian` (object-level); Starlette `AuthenticationMiddleware` (authN only).
- NestJS guards + CASL; Express middleware; Angular `CanActivateFn`/`CanMatchFn` (UX only).
- Engines: Open Policy Agent (Rego), AWS Cedar / Amazon Verified Permissions, OpenFGA and SpiceDB (Zanzibar-style ReBAC), Casbin, Oso, Permit.io.
- Databases: PostgreSQL Row-Level Security as defense in depth for tenant isolation; AWS IAM policies are ABAC/RBAC at the cloud layer.

## Trade-offs
| Pros | Cons |
|---|---|
| One auditable source of truth for access rules | Central PDP becomes a dependency on every request (latency, availability) |
| Consistent enforcement, fewer IDOR holes | Object-level checks require loading the resource first |
| Rules testable as pure functions | ABAC/ReBAC policies are hard to reason about at scale |
| Rules can change without touching handlers | Policy engines add a language and infrastructure to learn |

## Pitfalls
- **Checking only the role, not the object** (IDOR). Fix: every by-ID access goes through `can(principal, action, resource)`.
- **Client-side-only checks** (hidden button, route guard). Fix: the server enforces; UI mirrors for UX.
- **List endpoints unfiltered**: single-item fetch is checked, but `GET /orders?tenant=` isn't. Fix: push tenant/owner filters into the query (repository scoping, RLS).
- **Allow by default** when a rule is missing. Fix: deny unless an explicit rule allows; add a test that enumerates routes and asserts each has a PEP.
- **Trusting identity from request body/headers** (`X-User-Id`). Fix: derive the principal only from the verified token/session.
- **Stale permissions** in long-lived JWTs. Fix: short token lifetime, or look up roles server-side for sensitive actions.
- **403 vs 404 leaks existence.** Use 404 for resources the caller may not know about.

## Refactoring path
- **Introduce it:** inventory every route and its current checks (a table). Write the PDP as pure functions with tests per rule. Add a PEP (dependency/guard/decorator) and migrate routes one by one. Add a CI test that fails if a route lacks the PEP. Consider RLS for tenant isolation as a second layer.
- **Remove it / simplify:** if a policy engine is used for a handful of static role checks, replace it with an in-code permission map and keep the PEP interface so it can return later.

## Related patterns
- [Secure Proxy](secure-proxy.md) — a PEP wrapped around one object or service.
- [Single Access Point](single-access-point.md) — ensures all traffic passes the PEP.
- [Intercepting Validator](intercepting-validator.md) — validates input shape; authorization decides permission.
- [Specification](../behavioral/specification.md) — composable predicates; a natural shape for ABAC rules and query filters.
- [Chain of Responsibility](../behavioral/chain-of-responsibility.md) — middleware chains host authN → authZ → validation.
- [Decorator](../structural/decorator.md) — `@requires("order:refund")` on service methods.
- [Front Controller](../ui/front-controller.md) — a single dispatcher where route-level checks attach.
