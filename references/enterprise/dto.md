# DTO (Data Transfer Object)

**Category:** Enterprise integration · **Also known as:** Schema, request/response model, view model (on the client), contract · **Related:** Domain Model, Message Translator, Content Filter, Intercepting Validator, Data Mapper

## Intent
A plain, behavior-free object that defines exactly the shape of data crossing a boundary (HTTP API, message, process), decoupled from internal models.

## Problem
- Endpoints return ORM entities directly: a new `password_hash` column leaks into the API, and renaming a column breaks the mobile app.
- Input is accepted as `dict`/`any` and validated ad hoc, deep in the code.
- Frontend and backend disagree about field names and nullability; bugs show up at runtime.
- Scenario: `GET /users/{id}` serializes the SQLAlchemy `User`, including `is_admin` and `stripe_customer_id`; `PATCH /users/{id}` lets a client set `is_admin`.

## Solution and structure
Define separate, explicit types per boundary and direction, and map to/from the domain at the edge.

```
JSON ──validate──> UserCreateIn (DTO) ──map──> User (domain/ORM) ──map──> UserOut (DTO) ──serialize──> JSON
```
- **Input DTOs** validate and coerce untrusted data (types, ranges, formats) and contain only fields a client may set.
- **Output DTOs** whitelist fields to expose.
- **DTO vs domain model:** a DTO has no behavior and no invariants beyond shape/format; it is versioned with the API contract. A domain model carries rules and changes with the business. Never reuse one as the other except in trivial CRUD.
- In TypeScript, interfaces/types are erased at runtime: they give compile-time safety only. For untrusted input (API responses, form data, messages) pair them with a runtime validator (zod, valibot, class-validator) and infer the type from the schema.

## When to use
- Every public API, message schema and cross-service contract.
- Separating what clients can write (`UserUpdateIn`) from what they can read (`UserOut`).
- Aggregating data from several sources into one response shape.
- Generating clients/types (OpenAPI → TypeScript) for the frontend.

## When NOT to use
- Between functions inside one module: pass domain objects or primitives.
- A DTO per layer per entity "just in case" (`UserDto`, `UserModel`, `UserEntity`, `UserVm` all identical): map only at real boundaries.
- Internal admin tools where the ORM model *is* the contract and nobody else consumes it (accept the coupling knowingly).

## Example — Python
Pydantic v2 schemas: separate in/out models, strict validation, built from ORM attributes.

```python
from datetime import datetime
from typing import Annotated
from uuid import UUID
from fastapi import FastAPI
from pydantic import BaseModel, ConfigDict, EmailStr, Field, StringConstraints

Name = Annotated[str, StringConstraints(strip_whitespace=True, min_length=1, max_length=100)]

class UserCreateIn(BaseModel):
    model_config = ConfigDict(extra="forbid")       # reject unknown fields like "is_admin"
    email: EmailStr
    full_name: Name
    password: Annotated[str, Field(min_length=12)]

class UserUpdateIn(BaseModel):
    model_config = ConfigDict(extra="forbid")
    full_name: Name | None = None

class UserOut(BaseModel):
    model_config = ConfigDict(from_attributes=True)  # read from ORM objects
    id: UUID
    email: EmailStr
    full_name: str
    created_at: datetime

app = FastAPI()

@app.post("/users", response_model=UserOut, status_code=201)
def create_user(body: UserCreateIn, svc: UserServiceDep) -> UserOut:
    user = svc.register(body.email, body.full_name, body.password)  # domain/ORM object
    return UserOut.model_validate(user)    # whitelist: no password_hash, no is_admin
```
`EmailStr` requires the `email-validator` extra (`pip install "pydantic[email]"`).

## Example — TypeScript
Angular: schema-first with zod, type inferred from the schema, runtime-validated API response.

```ts
import { Injectable, inject } from '@angular/core';
import { HttpClient } from '@angular/common/http';
import { map, Observable } from 'rxjs';
import { z } from 'zod';

export const UserOut = z.object({
  id: z.string().uuid(),
  email: z.string().email(),
  full_name: z.string(),
  created_at: z.coerce.date(),
});
export type UserOut = z.infer<typeof UserOut>;

export interface UserUpdateIn { full_name?: string }   // compile-time only, we produce it

@Injectable({ providedIn: 'root' })
export class UsersApi {
  private readonly http = inject(HttpClient);

  get(id: string): Observable<UserOut> {
    return this.http.get<unknown>(`/api/users/${id}`).pipe(map((raw) => UserOut.parse(raw)));
  }
  update(id: string, body: UserUpdateIn): Observable<UserOut> {
    return this.http.patch<unknown>(`/api/users/${id}`, body).pipe(map((raw) => UserOut.parse(raw)));
  }
}
```
`http.get<T>()` alone is an unchecked cast. Zod's API changed between v3 and v4 (e.g. string formats like `z.email()` became top-level); check the zod docs for your version. Alternative: generate types from FastAPI's OpenAPI (`openapi-typescript`, `orval`).

## In the wild
- Pydantic models in FastAPI, DRF serializers, marshmallow, msgspec `Struct`.
- NestJS DTO classes with `class-validator`; zod/valibot/ArkType; tRPC input schemas.
- Protobuf/Avro/JSON Schema messages, GraphQL input/output types, OpenAPI components.

## Trade-offs
| Pros | Cons |
|---|---|
| API contract decoupled from internal models | Mapping code and near-duplicate classes |
| Whitelisting prevents data leaks and mass assignment | Must keep frontend/backend types in sync (use codegen) |
| Validation at the boundary, trusted data inside | Runtime validation has a (small) CPU cost |
| Independent versioning of contract and model | Too many DTO layers create noise |

## Pitfalls
- **Returning ORM/domain objects directly:** leaks fields, couples API to schema. Fix: `response_model`/explicit output DTO.
- **One DTO for create, update and read:** clients can send `id` or `is_admin`. Fix: separate In/Out models; `extra="forbid"`.
- **Trusting TS interfaces on API data:** a cast is not validation. Fix: validate with a schema at the boundary.
- **Logic in DTOs** (methods computing prices). Fix: move to the domain model or service.
- **Floats and naive datetimes in contracts.** Fix: decimals as strings, ISO-8601 with timezone.
- **Breaking changes** by renaming fields. Fix: additive changes, versioned DTOs, deprecation period.

## Refactoring path
- **Introduce it:** for each endpoint returning an ORM object, add an `XOut` model mirroring the current JSON (no client change); then add `XIn` models with `extra="forbid"`; then generate TS types from OpenAPI for the frontend.
- **Remove it / simplify:** collapse identical internal DTO layers that never cross a boundary; keep only the API/message contracts.

## Related patterns
- [Domain Model](../data-access/domain-model.md) — behavior and invariants; DTOs only carry data.
- [Message Translator](../enterprise/message-translator.md) — converts between DTO schemas of different systems.
- [Content Filter](../enterprise/content-filter.md) — an output DTO is a content filter over the model.
- [Intercepting Validator](../security/intercepting-validator.md) — DTO validation as a central boundary check.
- [Data Mapper](../data-access/data-mapper.md) — mapping at the DB boundary, analogous to DTO mapping at the API boundary.
