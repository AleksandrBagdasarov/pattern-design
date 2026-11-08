# Secure Proxy

**Category:** Security · **Also known as:** Protection Proxy, Guard, Access Proxy · **Related:** Proxy, Authorization, Decorator, Dynamic Proxy, Single Access Point, Ambassador, Sidecar

## Intent
Put a stand-in with the same interface in front of a sensitive object or service, and perform security checks (authorization, auditing, masking, rate limits) there before delegating, so the protected object never sees an unchecked call.

## Problem
- A powerful object (a payments client, a document store, an admin service) is passed around, and every caller is trusted to check permissions first.
- The protected code is third-party or shared and cannot be modified to add checks.
- Sensitive results must be filtered or masked per caller (PII, card numbers) and auditing must be consistent.
- Scenario: a `DocumentStore` with `read`, `write` and `delete` is injected into five services. Two of them forget the tenant check on `delete`. Wrapping the store in a proxy that enforces tenant and role on every call closes all five at once.

## Solution and structure
- **Subject interface**: what clients call (`DocumentStore`).
- **Real subject**: the unprotected implementation; not handed to untrusted callers.
- **Secure proxy**: implements the same interface, holds the real subject and the caller's principal, checks the policy (and optionally audits, masks, throttles), then delegates or refuses.
- **Composition root** wires clients to the proxy only.

```
client ─► SecureDocumentStore (same interface) ── check(principal, action, resource)
                         │ allowed                 └─ denied ─► raise / 403 + audit log
                         ▼
                   S3DocumentStore (real subject)
```

In-process proxy vs out-of-process: the same idea at network level is an API gateway, a service mesh sidecar (Envoy with an authz filter) or an authenticating reverse proxy (oauth2-proxy). Choose by where trust boundaries are.

## When to use
- One object or service is the sensitive resource and must be guarded regardless of who calls it.
- You cannot or should not change the real implementation (vendor SDK, shared library).
- You need consistent cross-cutting security on an object's methods: audit trail, PII masking, per-principal rate limits.

## When NOT to use
- Checks belong at the request boundary and apply to all routes: use a route-level PEP / middleware ([Authorization](authorization.md), [Single Access Point](single-access-point.md)).
- The real subject is still reachable by other paths (imported directly elsewhere). A bypassable proxy gives false confidence; fix the wiring first.
- The check needs data only the real subject has after doing the work (e.g. filtering a huge result set). Push the filter into the query instead.
- Simple single-method protection: a decorator (`@requires(...)`) on the function is simpler than a class proxy.

## Example — Python
A proxy class implementing the same `Protocol`. It receives the principal per request (constructed in a FastAPI dependency), so it is request-scoped.

```python
import logging
from dataclasses import dataclass
from typing import Protocol

audit = logging.getLogger("audit")

class DocumentStore(Protocol):
    def read(self, tenant_id: str, doc_id: str) -> bytes: ...
    def delete(self, tenant_id: str, doc_id: str) -> None: ...

@dataclass(frozen=True)
class Principal:
    user_id: str
    tenant_id: str
    roles: frozenset[str]

class PermissionDenied(Exception): ...

class SecureDocumentStore:
    """Same interface as DocumentStore; enforces tenant isolation and roles."""
    def __init__(self, inner: DocumentStore, principal: Principal) -> None:
        self._inner = inner
        self._p = principal

    def _check(self, action: str, tenant_id: str, doc_id: str, allowed_roles: set[str]) -> None:
        ok = tenant_id == self._p.tenant_id and bool(self._p.roles & allowed_roles)
        audit.info("doc.%s user=%s doc=%s allowed=%s", action, self._p.user_id, doc_id, ok)
        if not ok:
            raise PermissionDenied(f"{action} denied on {doc_id}")

    def read(self, tenant_id: str, doc_id: str) -> bytes:
        self._check("read", tenant_id, doc_id, {"viewer", "editor", "admin"})
        return self._inner.read(tenant_id, doc_id)

    def delete(self, tenant_id: str, doc_id: str) -> None:
        self._check("delete", tenant_id, doc_id, {"admin"})
        self._inner.delete(tenant_id, doc_id)

# FastAPI wiring: handlers only ever receive the proxy.
# def get_documents(p: Principal = Depends(current_principal)) -> DocumentStore:
#     return SecureDocumentStore(s3_store, p)
```

## Example — TypeScript
A NestJS/Node service-level proxy that also masks PII in results for callers without the `pii:read` permission.

```typescript
export interface Customer { id: string; name: string; email: string; iban: string; }
export interface CustomerRepo { findById(id: string): Promise<Customer | null>; }
export interface Caller { userId: string; tenantId: string; permissions: ReadonlySet<string>; }

export class ForbiddenError extends Error {}

const mask = (s: string) => (s.length <= 4 ? '****' : `${'*'.repeat(s.length - 4)}${s.slice(-4)}`);

export class SecureCustomerRepo implements CustomerRepo {
  constructor(
    private readonly inner: CustomerRepo,
    private readonly caller: Caller,
    private readonly audit: (event: string, data: Record<string, unknown>) => void,
  ) {}

  async findById(id: string): Promise<Customer | null> {
    if (!this.caller.permissions.has('customer:read')) {
      this.audit('customer.read.denied', { userId: this.caller.userId, id });
      throw new ForbiddenError('customer:read required');
    }
    const c = await this.inner.findById(id);
    if (!c) return null;
    this.audit('customer.read', { userId: this.caller.userId, id });
    return this.caller.permissions.has('pii:read')
      ? c
      : { ...c, email: mask(c.email), iban: mask(c.iban) };   // response-level data minimization
  }
}

// Request-scoped wiring (e.g. Nest REQUEST scope or a per-request factory):
// const repo: CustomerRepo = new SecureCustomerRepo(pgCustomerRepo, req.caller, auditLog);
```

## In the wild
- Network level: API gateways (AWS API Gateway with Lambda/JWT authorizers, Kong), oauth2-proxy, Envoy `ext_authz`, Istio `AuthorizationPolicy`, AWS ALB OIDC authentication.
- Data level: PostgreSQL Row-Level Security and security-barrier views act as a proxy in front of tables; S3 Access Points and presigned URLs are scoped proxies to objects.
- Code level: Spring Security method security (`@PreAuthorize`), NestJS guards/interceptors, Django's `PermissionRequiredMixin`; Python `__getattr__`-based wrappers and JS `Proxy` for dynamic variants.
- OS level: `sudo`, browser sandbox brokers.

## Trade-offs
| Pros | Cons |
|---|---|
| Real subject is unchanged; checks centralized per resource | Bypassable if the real subject is reachable elsewhere |
| Transparent to clients (same interface) | Each interface method must be wrapped; easy to miss a new one |
| Natural home for audit, masking, throttling | Request-scoped proxies add allocation and wiring per request |
| Composable with other proxies/decorators | Hidden behavior: callers may not expect `PermissionDenied` |

## Pitfalls
- **Leaking the real subject** (also registered in DI, or exposed via an attribute like `proxy.inner`). Fix: only the proxy is injectable; keep the real one private to the composition root.
- **New methods added to the interface but not to the proxy** (with `__getattr__` fallthrough, they delegate unchecked). Fix: explicit methods, no generic passthrough; a test that every interface method is wrapped.
- **Trusting arguments for the check** (`tenant_id` from the client) instead of the principal and the loaded resource. Fix: compare against the authenticated principal and the resource's real owner.
- **Masking only in one path**: proxy masks `find_by_id` but `search` returns raw data. Fix: all read methods go through the same masking function.
- **Audit logs that contain the sensitive data** they protect. Fix: log IDs and decisions, not payloads.

## Refactoring path
- **Introduce it:** extract an interface for the sensitive object; write the proxy implementing it with explicit checks; change the composition root so callers receive the proxy; remove direct imports of the real subject; add tests for denied paths.
- **Remove it / simplify:** if all access already flows through a route-level PEP and the proxy's checks duplicate it, delete the proxy and keep auditing in a decorator or middleware.

## Related patterns
- [Proxy](../structural/proxy.md) — the general structural pattern; Secure Proxy is its protection variant.
- [Authorization](authorization.md) — the policy the proxy consults; the proxy is one kind of enforcement point.
- [Decorator](../structural/decorator.md) — same shape; decorators add behavior, a secure proxy restricts access.
- [Dynamic Proxy](../structural/dynamic-proxy.md) — generates proxies at runtime; risky for security because of implicit passthrough.
- [Single Access Point](single-access-point.md) — the system-wide gate; a secure proxy guards one resource.
- [Ambassador](../emergent/ambassador.md), [Sidecar](../emergent/sidecar.md) — out-of-process proxies that can enforce authN/authZ and mTLS.
