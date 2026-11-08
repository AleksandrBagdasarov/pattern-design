# Secure Factory / Builder

**Category:** Security · **Also known as:** Secure Factory, Secure Builder, Hardened Client Factory · **Related:** Factory, Builder, Abstract Factory, Authorization, Secure Proxy, Private Class Data, Dependency Injection

## Intent
Centralize the creation of security-sensitive objects (clients, sessions, tokens, cookies, crypto contexts, outbound requests) so that credentials, secure defaults and policy are applied identically every time, and callers cannot construct an insecure variant by accident.

## Problem
- Every place that creates an HTTP client, DB connection or JWT chooses its own TLS settings, timeouts, algorithms and secret source; one of them gets it wrong (`verify=False`, `alg: none`, no timeout).
- Secrets are read from environment variables in many modules, logged in error messages, or hard-coded in tests that end up in production.
- Objects need a security context (tenant, principal, scopes) attached at creation and it is easy to forget.
- Scenario: three services call a partner API. One copies a snippet with `httpx.Client(verify=False)` "temporarily" during an incident; a year later it is still there. A shared factory that always builds the client with TLS verification, pinned timeouts, and a token from the secrets manager removes the choice.

## Solution and structure
- **Secure factory**: the only sanctioned way to obtain the object. It reads secrets from a secret provider, applies hardened defaults (TLS, timeouts, algorithms, cookie flags), attaches the security context, and returns the product.
- **Secure builder**: when the product has options, the builder exposes only safe knobs and validates in `build()` that the combination is secure (e.g. `SameSite=None` requires `Secure`).
- **Secret provider**: abstraction over the secrets manager (Vault, AWS Secrets Manager, env in dev) with caching and rotation.
- **Product**: ideally immutable, with secrets not exposed via `repr`/serialization.

```
caller ─► SecureClientFactory.for_partner("acme", principal)
              ├─ secrets.get("acme/api-token")      (rotation-aware, never logged)
              ├─ TLS verify on, min TLS 1.2, timeouts, retries capped
              └─ attach tenant/principal headers
          ─► httpx.Client (hardened)
```

## When to use
- The same kind of sensitive object is created in several places.
- Defaults matter for security and must not depend on each developer's memory (TLS, cookie flags, JWT algorithms, password hashing parameters).
- Credentials must come from one managed source and rotate without code changes.

## When NOT to use
- A single creation site: write it carefully there and add a test; a factory adds indirection without benefit.
- The library's own defaults are already secure and not overridable insecurely (e.g. `secrets.token_urlsafe`, `argon2-cffi`'s `PasswordHasher` defaults). Use them directly.
- Do not let the factory become a place where policy decisions about *access* are made; that belongs to [Authorization](authorization.md).

## Example — Python
A factory for outbound partner clients and for signed tokens. `pydantic.SecretStr` keeps secrets out of `repr` and logs.

```python
from dataclasses import dataclass
from datetime import UTC, datetime, timedelta
from functools import cache
from typing import Protocol
import httpx, jwt   # PyJWT
from pydantic import SecretStr

class SecretProvider(Protocol):
    def get(self, name: str) -> SecretStr: ...

@dataclass(frozen=True)
class PartnerConfig:
    base_url: str          # validated to be https:// at load time
    token_secret: str      # name in the secret store, not the secret itself

class SecureClientFactory:
    def __init__(self, secrets: SecretProvider, partners: dict[str, PartnerConfig]) -> None:
        self._secrets = secrets
        self._partners = partners

    @cache                                   # one pooled client per partner
    def partner_client(self, partner: str) -> httpx.Client:
        cfg = self._partners[partner]
        if not cfg.base_url.startswith("https://"):
            raise ValueError(f"{partner}: plaintext HTTP is not allowed")
        token = self._secrets.get(cfg.token_secret).get_secret_value()
        return httpx.Client(
            base_url=cfg.base_url,
            verify=True,                                     # no opt-out parameter exposed
            timeout=httpx.Timeout(5.0, connect=2.0),
            headers={"Authorization": f"Bearer {token}"},
            follow_redirects=False,                          # avoid leaking the token on redirect
        )

    def service_token(self, subject: str, audience: str, ttl: timedelta = timedelta(minutes=5)) -> str:
        key = self._secrets.get("jwt/signing-key").get_secret_value()
        now = datetime.now(UTC)
        claims = {"sub": subject, "aud": audience, "iat": now, "exp": now + min(ttl, timedelta(minutes=15))}
        return jwt.encode(claims, key, algorithm="EdDSA")    # algorithm fixed here, not by callers
```

Note: `@cache` on a method keys on `self` too and keeps the instance alive; fine for an app-lifetime factory, not for short-lived objects.

## Example — TypeScript
A secure cookie builder for an Express/Node backend: only safe options are exposed, and `build()` enforces combinations.

```typescript
import type { CookieOptions, Response } from 'express';

type SameSite = 'strict' | 'lax' | 'none';

export class SessionCookieBuilder {
  private sameSite: SameSite = 'lax';
  private maxAgeMs = 30 * 60 * 1000;           // 30 min default
  private path = '/';

  constructor(private readonly name: string, private readonly value: string) {
    if (!/^__Host-/.test(name)) throw new Error('session cookies must use the __Host- prefix');
  }

  crossSite(): this { this.sameSite = 'none'; return this; }   // still forced Secure below
  ttlMinutes(m: number): this {
    if (m <= 0 || m > 24 * 60) throw new Error('session TTL must be 1 min .. 24 h');
    this.maxAgeMs = m * 60 * 1000;
    return this;
  }

  build(): [string, string, CookieOptions] {
    return [this.name, this.value, {
      httpOnly: true,            // not readable from JS (XSS can't exfiltrate it)
      secure: true,              // HTTPS only; required for SameSite=None and __Host-
      sameSite: this.sameSite,
      path: this.path,           // __Host- requires Path=/ and no Domain
      maxAge: this.maxAgeMs,
    }];
  }
}

export function setSession(res: Response, sessionId: string): void {
  res.cookie(...new SessionCookieBuilder('__Host-sid', sessionId).ttlMinutes(60).build());
}
```

## In the wild
- Python: `ssl.create_default_context()` (a secure factory for TLS contexts), `secrets` module, `argon2.PasswordHasher`, `boto3.Session` credential provider chain, `itsdangerous` serializers, Django `SESSION_COOKIE_SECURE`/`CSRF_*` settings applied by one framework factory.
- Node: `tls.createSecureContext`, `crypto.createCipheriv` wrappers, `helmet()` (secure header defaults), `express-session` cookie config.
- AWS SDKs: default credential provider chains (no keys in code); Secrets Manager / Parameter Store caching clients.
- Angular: `provideHttpClient(withXsrfConfiguration(...))` centralizes XSRF token handling; `DomSanitizer` is the only sanctioned way to create trusted HTML values.

## Trade-offs
| Pros | Cons |
|---|---|
| Secure defaults applied everywhere, by construction | Callers lose flexibility; legitimate edge cases need factory changes |
| Single source of credentials; rotation in one place | Factory becomes critical shared code needing careful review |
| Easier audit ("all partner clients come from here") | Bypass is still possible unless lint/review enforces usage |
| Secrets less likely to leak through ad-hoc code | Caching secrets in clients complicates rotation |

## Pitfalls
- **Exposing an "insecure" escape hatch** (`verify: bool = True` parameter). Fix: no such parameter; dev-only overrides through explicit, environment-gated configuration.
- **Secrets in `repr`, logs, exceptions or URLs.** Fix: `SecretStr`, redacting log filters, headers instead of query strings.
- **Caching clients forever** after the token rotates. Fix: refresh on 401 or TTL, or use auth hooks (`httpx.Auth`) that fetch tokens per request with caching.
- **Allowing redirects with credentials** to other hosts. Fix: disable redirects or strip auth on cross-origin redirects.
- **Developers bypassing the factory** with `httpx.Client(...)`. Fix: lint rule (ruff banned-api / ESLint `no-restricted-imports`) and code review checklist.
- **Weak options accepted silently** (`SameSite=None` without `Secure`). Fix: validate combinations in `build()` and raise.

## Refactoring path
- **Introduce it:** grep for constructor calls of sensitive types (`httpx.Client`, `jwt.encode`, `res.cookie`, `boto3.client`). Write the factory with hardened defaults; replace call sites; add a lint rule banning direct construction outside the factory module; move secrets to a provider.
- **Remove it / simplify:** if a library now ships equally secure defaults and the factory only forwards arguments, replace it with direct use plus a lint/config check, keeping the secret provider.

## Related patterns
- [Factory](../creational/factory.md) — the general creational pattern this specializes.
- [Builder](../creational/builder.md) — the builder form validates secure option combinations in `build()`.
- [Abstract Factory](../creational/abstract-factory.md) — environment-specific families (dev vs prod secret providers).
- [Private Class Data](private-class-data.md) — protects secrets inside the created objects after construction.
- [Secure Proxy](secure-proxy.md) — guards use of the object; the secure factory guards its creation.
- [Dependency Injection](../behavioral/dependency-injection.md) — inject the factory or its products so callers never construct them.
