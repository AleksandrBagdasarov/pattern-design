# Onion Architecture

**Category:** Architectural · **Also known as:** — · **Related:** Clean Architecture, Hexagonal, Layered, Domain Model, Repository, Dependency Injection

## Intent
Arrange the application as concentric rings with the **domain model at the center**, surrounded by domain services, then application services, then infrastructure/UI/tests at the edge. Every ring may depend only on rings closer to the center; infrastructure is pushed to the outermost ring.

## Problem
- Classic layering makes the domain depend on data access, so the most valuable code is coupled to the least stable code.
- Domain rules are smeared across "services" because there is no explicit place for logic that spans several entities.
- Scenario: an insurance pricing engine has rich rules (risk factors, discounts, regional regulations). The team wants a domain that compiles and tests with no DB, plus a clear spot for cross-entity rules such as "family discount".

## Solution and structure
```
            ┌──────────────── Infrastructure · UI · Tests ────────────────┐
            │  ┌────────────── Application services ──────────────────┐   │
            │  │  ┌─────────── Domain services ───────────────────┐   │   │
            │  │  │   ┌──── Domain model (entities, VOs) ────┐    │   │   │
            │  │  │   └──────────────────────────────────────┘    │   │   │
            │  │  │   repository interfaces live here or above    │   │   │
            │  │  └───────────────────────────────────────────────┘   │   │
            │  └──────────────────────────────────────────────────────┘   │
            └─────────────────────────────────────────────────────────────┘
```
- **Domain model:** entities and value objects, invariants. No dependencies.
- **Domain services:** logic spanning entities; declare repository interfaces.
- **Application services:** use cases, transactions, authorization checks; orchestrate domain services.
- **Outer ring:** ORM repositories, web controllers, message consumers, tests, DI wiring. Implements inner interfaces.
- Coupling is toward the center; the outer ring is where all I/O lives.

## When to use
- Rich, long-lived domain models (DDD), where entity behavior matters more than CRUD.
- Teams that want explicit guidance on *where domain services go* versus application services.
- .NET/Java-style codebases with project-per-ring; Python packages per ring also work.

## When NOT to use
- Anemic CRUD domains: rings will be empty pass-throughs. Use Layered.
- Small services where a single `service.py` + `repo.py` is readable.
- If the team already uses Clean or Hexagonal vocabulary, do not introduce Onion terms alongside them; pick one vocabulary.

## Clean vs Hexagonal vs Onion
All three apply the same core rule: **business logic in the center, dependencies point inward, I/O behind interfaces implemented at the edge.** They differ in emphasis:

| Aspect | Hexagonal (Cockburn, 2005) | Onion (Palermo, 2008) | Clean (Martin, 2012) |
|---|---|---|---|
| Central metaphor | inside vs outside, ports and adapters | concentric rings around the domain model | concentric rings + Dependency Rule |
| Inner structure | unspecified ("application") | domain model → domain services → application services | entities → use cases |
| Emphasis | symmetry of driving/driven sides, testability | domain model at the very center (DDD) | use cases as first-class, framework independence |
| Typical names | `ports/`, `adapters/inbound`, `adapters/outbound` | `Domain`, `Domain.Services`, `Application`, `Infrastructure` | `entities/`, `use_cases/`, `interface_adapters/`, `frameworks/` |

Practical advice: choose by vocabulary the team already speaks. A codebase can be described as all three at once; what matters is enforcing the inward dependency rule.

## Directory layout
```
pricing/
  domain/model/        policy.py, money.py
  domain/services/     discounts.py, repositories.py   # repo Protocols
  application/         quote_policy.py
  infrastructure/      sql_policy_repo.py, api.py, container.py
```

## Example — Python
```python
# domain/model/policy.py
from dataclasses import dataclass
from decimal import Decimal

@dataclass(frozen=True)
class Policy:
    holder_id: str
    base_premium: Decimal

# domain/services/repositories.py + discounts.py
from typing import Protocol

class PolicyRepository(Protocol):
    def by_household(self, household_id: str) -> list[Policy]: ...

def family_discount(policies: list[Policy]) -> Decimal:      # domain service: spans entities
    return Decimal("0.10") if len(policies) >= 3 else Decimal("0")

# application/quote_policy.py
@dataclass
class QuotePolicy:
    policies: PolicyRepository
    def __call__(self, household_id: str, new: Policy) -> Decimal:
        existing = self.policies.by_household(household_id)
        discount = family_discount([*existing, new])
        return (new.base_premium * (1 - discount)).quantize(Decimal("0.01"))

# infrastructure/sql_policy_repo.py — outer ring depends inward
from sqlalchemy import text
from sqlalchemy.orm import Session

class SqlPolicyRepository:
    def __init__(self, session: Session) -> None:
        self.session = session
    def by_household(self, household_id: str) -> list[Policy]:
        rows = self.session.execute(
            text("SELECT holder_id, premium FROM policies WHERE household_id = :h"),
            {"h": household_id},
        )
        return [Policy(r.holder_id, Decimal(r.premium)) for r in rows]
```

## Example — TypeScript
```ts
// domain/model + domain/services (no imports from outer rings)
export interface Policy { holderId: string; basePremium: number }
export abstract class PolicyRepository { abstract byHousehold(id: string): Promise<Policy[]> }
export const familyDiscount = (ps: Policy[]) => (ps.length >= 3 ? 0.1 : 0);

// application
export class QuotePolicy {
  constructor(private readonly repo: PolicyRepository) {}
  async execute(householdId: string, p: Policy): Promise<number> {
    const all = [...(await this.repo.byHousehold(householdId)), p];
    return Math.round(p.basePremium * (1 - familyDiscount(all)) * 100) / 100;
  }
}

// infrastructure (Node + NestJS-style provider wiring)
import { Pool } from 'pg';
export class PgPolicyRepository extends PolicyRepository {
  constructor(private readonly pool: Pool) { super(); }
  async byHousehold(id: string): Promise<Policy[]> {
    const { rows } = await this.pool.query('SELECT holder_id, premium FROM policies WHERE household_id=$1', [id]);
    return rows.map(r => ({ holderId: r.holder_id, basePremium: Number(r.premium) }));
  }
}
// providers: [{ provide: PolicyRepository, useFactory: (pool: Pool) => new PgPolicyRepository(pool), inject: [Pool] }]
```

## In the wild
- Common in ASP.NET solution templates (`Domain`, `Application`, `Infrastructure`, `WebApi` projects).
- Python: same layout as "Cosmic Python"; enforce rings with `import-linter` "layers" contracts.
- TS: Nx module boundary tags (`type:domain`, `type:app`, `type:infra`).

## Trade-offs
| Pros | Cons |
|---|---|
| Domain is the most stable, most testable code | More projects/packages and mapping |
| Explicit home for domain services | Domain vs application service boundary is often argued |
| Infrastructure fully replaceable | Overkill for CRUD |
| Maps well onto DDD tactical patterns | Ring rules need tooling to stay enforced |

## Pitfalls
- **Repository interface placed in infrastructure:** the domain then imports infrastructure. Put interfaces in the domain-services or application ring.
- **Application services containing domain rules:** push rules into entities or domain services; application services orchestrate.
- **Shared "Common" project everyone imports:** it becomes a hidden outer-to-inner dependency. Keep it tiny and pure.
- **Mixing vocabularies:** do not have `ports/` *and* `Domain.Services`; choose one style name.

## Refactoring path
- **Introduce it:** from a Layered app, move repository interfaces up into the domain, make the data layer implement them (dependency inversion), then extract cross-entity rules from services into domain services.
- **Remove it / simplify:** merge domain services and application services into one service module when the distinction holds no logic; collapse rings into feature folders.

## Related patterns
- [Clean Architecture](../architectural/clean-architecture.md) — same rule, use-case-centric naming.
- [Hexagonal](../architectural/hexagonal.md) — same rule, ports/adapters naming, no prescribed inner rings.
- [Layered](../architectural/layered.md) — the style Onion inverts.
- [Domain Model](../data-access/domain-model.md) — the center ring.
- [Repository](../data-access/repository.md) — interface inside, implementation outside.
- [Dependency Injection](../behavioral/dependency-injection.md) — wires outer implementations into inner interfaces.
