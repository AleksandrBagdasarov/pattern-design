# Intercepting Validator

**Category:** Security · **Also known as:** Input Validation Filter, Validation Middleware, Validation Pipe · **Related:** Single Access Point, Authorization, Chain of Responsibility, Decorator, DTO, Front Controller, Specification

## Intent
Validate, normalize and reject untrusted input in one interception layer *before* it reaches handlers or business logic, so every handler receives data that is already well-formed and typed.

## Problem
- Handlers each re-check (or forget to check) lengths, formats, ranges and required fields.
- Untrusted input reaches SQL, shell, file paths, templates or downstream APIs and causes injection, path traversal, SSRF or crashes on unexpected types.
- Error responses are inconsistent (some 400 with details, some 500 stack traces).
- Scenario: an endpoint accepts `{"amount": "100", "currency": "usd "}`. One handler parses `amount` as float, another as string, a third forgets that `-100` is possible and issues a negative refund. A single schema that coerces to `Decimal`, requires `> 0` and normalizes currency to an ISO code removes the whole class of bugs.

## Solution and structure
- **Interceptor**: sits between the transport and the handler (framework pipe, middleware, decorator, parameter binding).
- **Validation schema**: declarative description of allowed input (types, ranges, lengths, enums, patterns, allowlists); unknown fields rejected.
- **Normalization**: canonical forms (trim, case-fold, Unicode normalize, parse dates to UTC) applied once.
- **Rejection**: a consistent 400/422 response with field-level errors, without echoing dangerous input or internals.
- **Handler**: receives typed, validated objects only.

```
request ─► [ parse ─► validate schema ─► normalize ] ─► handler(typed DTO)
                         │ invalid
                         └──► 422 { field errors }   (handler never runs)
```

Validation is not sanitization of output. Injection is prevented at the *sink*: parameterized queries, template auto-escaping, `subprocess` with argument lists, URL allowlists for SSRF. Input validation narrows what reaches sinks; it does not replace sink-side defenses.

## When to use
- Any boundary receiving untrusted data: HTTP APIs, message consumers, webhooks, file uploads, CLI arguments, config files.
- Many handlers share the same input types (IDs, money, emails, pagination).
- You want uniform, documented error responses (and OpenAPI schemas generated from the same definitions).

## When NOT to use
- Internal calls between your own already-validated layers: re-validating everywhere costs time and hides where trust is established. Validate at the boundary, then use typed domain objects.
- Business rules that need state (e.g. "stock is available", "user may refund") belong in the domain/service layer or [Authorization](authorization.md), not in the input validator.
- Don't hand-roll an interceptor when the framework provides one (FastAPI/Pydantic, NestJS `ValidationPipe`, Angular form validators).

## Example — Python
FastAPI + Pydantic v2: the request model *is* the interceptor. The handler is never called with invalid data, and the error format is uniform (422).

```python
from decimal import Decimal
from typing import Annotated, Literal
from uuid import UUID
from fastapi import FastAPI, Query
from pydantic import BaseModel, BeforeValidator, ConfigDict, EmailStr, Field, StringConstraints

def _normalize_code(v: object) -> object:
    return v.strip().upper() if isinstance(v, str) else v

# Normalize first (" usd" -> "USD"), then check against an allowlist.
Currency = Annotated[Literal["EUR", "USD", "GBP"], BeforeValidator(_normalize_code)]
Sku = Annotated[str, StringConstraints(strip_whitespace=True, pattern=r"^[A-Z0-9-]{3,32}$")]

class RefundRequest(BaseModel):
    model_config = ConfigDict(extra="forbid", frozen=True)   # unknown fields -> 422
    order_id: UUID
    amount: Annotated[Decimal, Field(gt=0, le=Decimal("10000"), decimal_places=2)]
    currency: Currency
    reason: Annotated[str, StringConstraints(strip_whitespace=True, min_length=3, max_length=500)]
    notify_email: EmailStr | None = None                       # needs `pydantic[email]`

app = FastAPI()

@app.post("/refunds", status_code=202)
async def create_refund(body: RefundRequest) -> dict[str, str]:
    # body.amount is a positive Decimal, body.currency is one of three codes: no re-checks here.
    return {"order_id": str(body.order_id), "status": "queued"}

SORT_COLUMNS = {"price": "price ASC", "-price": "price DESC", "name": "name ASC"}

@app.get("/products")
async def list_products(sku: Sku | None = None,
                        limit: Annotated[int, Query(ge=1, le=100)] = 20,
                        sort: Literal["price", "-price", "name"] = "name") -> dict[str, object]:
    order_by = SORT_COLUMNS[sort]       # allowlist maps input to SQL; never interpolate raw input
    return {"sku": sku, "limit": limit, "order_by": order_by}
```

## Example — TypeScript
NestJS (Node) global `ValidationPipe` with class-validator DTOs: every controller parameter passes the same interceptor. Angular reactive-form validators are the client-side mirror (UX only).

```typescript
import { ValidationPipe } from '@nestjs/common';
import { NestFactory } from '@nestjs/core';
import { Transform, Type } from 'class-transformer';
import { IsEmail, IsIn, IsOptional, IsPositive, IsUUID, Length, Max } from 'class-validator';
import { AppModule } from './app.module';

export class RefundRequestDto {
  @IsUUID() orderId!: string;
  @Type(() => Number) @IsPositive() @Max(10_000) amount!: number;
  @Transform(({ value }) => (typeof value === 'string' ? value.trim().toUpperCase() : value))
  @IsIn(['EUR', 'USD', 'GBP']) currency!: string;
  @Length(3, 500) reason!: string;
  @IsOptional() @IsEmail() notifyEmail?: string;
}

async function bootstrap() {
  const app = await NestFactory.create(AppModule);
  app.useGlobalPipes(new ValidationPipe({
    whitelist: true,              // strip properties without decorators
    forbidNonWhitelisted: true,   // ...or reject them (400) instead of silently stripping
    transform: true,              // give controllers real DTO instances with converted types
  }));
  await app.listen(3000);
}
bootstrap();

// Controller: @Post('refunds') create(@Body() dto: RefundRequestDto) { /* dto is validated */ }
// Lighter alternative in plain Node/Express: a zod schema + one middleware calling schema.safeParse(req.body).
```

## In the wild
- Python: Pydantic models in FastAPI (request bodies, query/path params), `pydantic-settings` for config, Django Forms/DRF serializers, marshmallow, `attrs` validators.
- Node: NestJS `ValidationPipe`, zod / valibot / ArkType with Express or tRPC, Fastify JSON-schema validation (Ajv), `express-validator`.
- Angular: `Validators.*` and custom `ValidatorFn`/`AsyncValidatorFn` on reactive forms; signal-based forms are newer and their API may still be changing, check angular.dev.
- Edge: AWS API Gateway request validators (JSON Schema), WAF rules (OWASP CRS), OpenAPI-driven validation (Connexion, `openapi-backend`).

## Trade-offs
| Pros | Cons |
|---|---|
| One consistent place for input rules and errors | Schema can drift from the domain if duplicated in many layers |
| Handlers get typed, trustworthy data | Over-strict schemas break clients on harmless changes |
| Shrinks attack surface (allowlists, size limits) | Gives false confidence if sinks aren't also safe |
| Schemas double as docs (OpenAPI) and client types | Validation cost on large payloads (mitigate with size limits) |

## Pitfalls
- **Silently ignoring unknown fields** enables mass assignment (`"is_admin": true`). Fix: `extra="forbid"` / `forbidNonWhitelisted`, and separate input DTOs from DB models.
- **Blocklists instead of allowlists** (stripping `<script>`). Fix: allowlist formats; encode output at the sink.
- **Using validated input in raw SQL / shell anyway.** Fix: parameterized queries, argument arrays, ORM; map sort fields through an allowlist.
- **Client-side validation only.** Fix: the server is the authority; Angular validators are UX.
- **No size limits**: a 50 MB JSON body or a 10k-element array exhausts memory. Fix: body size limits at the server/gateway; `max_length` on lists.
- **Error messages echoing input** (reflected XSS in error pages, log injection). Fix: return field names and codes, not raw values; escape logs.
- **Float for money.** Fix: `Decimal` / integer minor units.

## Refactoring path
- **Introduce it:** list endpoints and their ad-hoc checks. Define one schema per input type; move checks into it; turn on the framework's global validation; delete per-handler checks once covered by tests (including invalid-input tests); add body-size limits.
- **Remove it / simplify:** if validation is duplicated at several internal layers, keep it at the boundary and pass typed domain objects inward; remove re-validation of already-typed values.

## Related patterns
- [Single Access Point](single-access-point.md) — validation commonly runs at the entry point for all requests.
- [Authorization](authorization.md) — validation checks shape; authorization checks permission. Run authN, then validation, then authZ on the loaded resource.
- [Chain of Responsibility](../behavioral/chain-of-responsibility.md) — middleware/pipe chains host the validator.
- [Decorator](../structural/decorator.md) — `@validate_call` style per-function validation.
- [DTO](../enterprise/dto.md) — the validated input type the interceptor produces.
- [Front Controller](../ui/front-controller.md) — the dispatcher where a global validator is attached.
- [Specification](../behavioral/specification.md) — for stateful business rules beyond input shape.
