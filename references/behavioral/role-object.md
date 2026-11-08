# Role Object

**Category:** Behavioral (domain modeling) · **Also known as:** Roles, Party/Role model · **Related:** Extension Object, Decorator, State, Component-Based Design, Domain Model, Bounded Context

## Intent
Model an entity that plays different roles in different contexts by attaching separate role objects to a stable core, instead of encoding every role into one class or a subclass hierarchy.

## Problem
- One real-world party is a customer, a supplier and an employee at once, or becomes one later. Each role has its own data (credit limit, payment terms, salary) and behavior.
- Inheritance fails: `Person -> Customer`, `Person -> Supplier` cannot represent a person who is both; multiple inheritance combinations explode, and an object cannot change class when a role is gained or lost.
- One fat class with all role fields is worse: `User` has 60 nullable columns, half of the methods only make sense when some flag is set, and every team edits the same file.
- Smells: `if user.is_supplier:` guarding supplier fields; nullable groups of columns that are "all set or all null"; duplicated identity records (`customers` and `suppliers` tables with the same company twice and no link).

Scenario: a B2B marketplace where an organization can buy (buyer role: credit limit, billing address), sell (seller role: payout account, commission rate), and later be suspended as a seller while still buying.

## Solution and structure
Split the entity into a **core** (identity and role-independent data) and **role objects** (role-specific state and behavior), each holding a reference back to the core. The core manages attaching, detaching and looking up roles.

```
             ┌───────────────┐ roles  ┌───────────────┐
             │ Organization  │◇──────>│ Role          │ core: org
             │ (core)        │        └──────┬────────┘
             │ id, name, vat │        ┌──────┴─────────┐
             │ add_role/get  │      BuyerRole      SellerRole
             └───────────────┘      credit_limit   payout_account
                                    place_order()  commission_rate
```

Participants:
- **Component core**: identity, shared attributes, role registry (`add_role`, `remove_role`, `role(Type)`, `has_role`).
- **Role**: role-specific data and behavior, with a back-reference to the core.
- **Client**: asks the core for the role it needs in its context; the buying module only deals with `BuyerRole`.

In a database this usually maps to a core table plus one table per role with a foreign key (`organizations`, `buyer_profiles`, `seller_profiles`).

In bounded-context architectures the same idea appears at a larger scale: each context keeps its own model of the party (Sales' Customer, Billing's Account) keyed by a shared ID.

## When to use
- Entities gain and lose roles at runtime, and multiple roles can coexist.
- Roles have substantial distinct data and behavior, owned by different modules or teams.
- You want each module to depend only on the role it uses (interface segregation for domain entities).

## When NOT to use
- Roles never change and never combine: use a single class or a discriminated union (`type: "buyer" | "seller"`).
- Roles differ only in a flag or two fields: keep them on the entity.
- Roles are really permissions (can edit, can approve): use authorization (RBAC/ABAC), not domain role objects.
- The role behavior is just a state machine of one entity: use State.
- You are in separate services: don't share a role-holding core object across services; use separate models per bounded context linked by ID.

## Example — Python
```python
from __future__ import annotations
from dataclasses import dataclass, field
from decimal import Decimal

class RoleError(LookupError): ...

@dataclass(eq=False)
class Role:
    org: Organization = field(repr=False)

@dataclass(eq=False)
class BuyerRole(Role):
    credit_limit: Decimal = Decimal("0")
    def can_order(self, amount: Decimal, open_balance: Decimal) -> bool:
        return open_balance + amount <= self.credit_limit

@dataclass(eq=False)
class SellerRole(Role):
    payout_iban: str = ""
    commission_rate: Decimal = Decimal("0.10")
    suspended: bool = False
    def net_payout(self, gross: Decimal) -> Decimal:
        return gross * (1 - self.commission_rate)

@dataclass(eq=False)
class Organization:
    id: str
    name: str
    _roles: dict[type[Role], Role] = field(default_factory=dict, repr=False)

    def add_role[R: Role](self, role_type: type[R], **data: object) -> R:
        if role_type in self._roles:
            raise ValueError(f"{self.name} already has {role_type.__name__}")
        role = role_type(org=self, **data)
        self._roles[role_type] = role
        return role

    def remove_role(self, role_type: type[Role]) -> None:
        self._roles.pop(role_type, None)

    def has_role(self, role_type: type[Role]) -> bool:
        return role_type in self._roles

    def role[R: Role](self, role_type: type[R]) -> R:
        try:
            return self._roles[role_type]   # type: ignore[return-value]
        except KeyError:
            raise RoleError(f"{self.name} is not a {role_type.__name__}") from None

acme = Organization("org-1", "Acme GmbH")
acme.add_role(BuyerRole, credit_limit=Decimal("10000"))
acme.add_role(SellerRole, payout_iban="DE89...")
# ordering module only sees BuyerRole:
assert acme.role(BuyerRole).can_order(Decimal("500"), open_balance=Decimal("0"))
```

## Example — TypeScript
Frontend code usually receives roles as optional sub-objects from the API; a typed lookup keeps components role-specific.

```typescript
import { Component, computed, input } from '@angular/core';

interface BuyerRole { kind: 'buyer'; creditLimit: number; openBalance: number }
interface SellerRole { kind: 'seller'; commissionRate: number; suspended: boolean }
type Role = BuyerRole | SellerRole;

export interface Organization { id: string; name: string; roles: Role[] }

export function roleOf<K extends Role['kind']>(
  org: Organization, kind: K,
): Extract<Role, { kind: K }> | undefined {
  return org.roles.find((r): r is Extract<Role, { kind: K }> => r.kind === kind);
}

@Component({
  selector: 'app-seller-panel',
  template: `
    @if (seller(); as s) {
      <p>Commission: {{ s.commissionRate * 100 }}%</p>
      @if (s.suspended) { <p class="warn">Selling suspended</p> }
    } @else {
      <button>Become a seller</button>
    }
  `,
})
export class SellerPanel {
  readonly org = input.required<Organization>();
  readonly seller = computed(() => roleOf(this.org(), 'seller'));   // only the seller role matters here
}
```

## In the wild
- Party/Role models in ERP and CRM systems (SAP Business Partner with BP roles; Odoo `res.partner` with customer/supplier ranks).
- Django: one-to-one "profile" models per role (`CustomerProfile`, `SellerProfile`) linked to `User`, a pragmatic Role Object.
- SQLAlchemy: one-to-one relationships per role table; joined-table inheritance is the alternative when roles don't combine.
- DDD bounded contexts: the same person modeled separately per context, linked by ID.
- Not to be confused with RBAC roles in auth libraries (Keycloak, Casbin), which model permissions.

## Trade-offs
| Pros | Cons |
|---|---|
| Roles added/removed at runtime; combinations are free | More objects and indirection than one class |
| Each module depends only on its role | Identity questions: is a role equal to its core? |
| Role data lives in its own table, owned by its team | Cross-role rules need the core or a service to coordinate |
| Avoids subclass explosion and nullable-column sprawl | Lookups (`role(Type)`) can fail at runtime |

## Pitfalls
- **Role objects reaching into each other** (`seller.org.role(BuyerRole).credit_limit`). Fix: cross-role rules belong in a domain service or the core.
- **Treating roles as permissions.** Fix: keep authorization separate; a buyer role is domain data, "can approve orders" is a policy.
- **Missing-role errors at runtime.** Fix: `has_role` checks at the module boundary, or typed optionals (`roleOf` returning `undefined`) that force handling.
- **Duplicated core data in role tables** (name, address copied). Fix: core owns shared attributes; roles reference the core.
- **Over-generic role registry** (string keys, `dict[str, Any]`). Fix: type-keyed lookup with generics.
- **Equality confusion.** Fix: decide identity is the core's ID; roles compare by core plus role type.

## Refactoring path
- **Introduce it:** from a fat entity with role flags. (1) Identify groups of fields/methods used only when a flag is set. (2) Move each group into a role class with a back-reference; keep delegating methods on the entity temporarily. (3) Move clients to `entity.role(X)`. (4) Split the table into core + role tables with a migration. (5) Delete the flags; `has_role` replaces them.
- **Remove it / simplify:** if an entity never has more than one role and roles never change, collapse into a discriminated union or subclass per type; if roles carry almost no data, fold the fields back into the core.

## Related patterns
- [Extension Object](../structural/extension-object.md) — attaches interfaces at runtime; Role Object is its domain-modeling application with role state.
- [Decorator](../structural/decorator.md) — wraps to add behavior transparently; roles are explicitly looked up, not transparent.
- [State](state.md) — one active state that changes behavior; roles can coexist.
- [Component-Based Design](../structural/component-based-design.md) — same composition-over-inheritance idea for game/UI entities.
- [Domain Model](../data-access/domain-model.md) — role objects live inside a rich domain model and its bounded contexts.
