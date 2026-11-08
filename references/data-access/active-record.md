# Active Record

**Category:** Data access · **Also known as:** — (Django ORM, Rails ActiveRecord, Laravel Eloquent) · **Related:** Data Mapper, Row Data Gateway, Repository, Domain Model, DTO

## Intent
An object wraps one database row, carries its data and a little domain logic, and knows how to load and save itself.

## Problem
- You are building a CRUD-heavy app (admin, CMS, internal tool) and a full Domain Model + Repository + Mapper stack would triple the code.
- The object model mirrors the tables almost exactly.
- Scenario: a back-office app with `Customer`, `Product`, `Coupon`; most endpoints are list/create/update/delete with a couple of validation rules.

## Solution and structure
One class per table; an instance is a row. Class methods find rows, instance methods save/delete, and simple derived logic sits on the object.

```
Coupon (class)  ── Coupon.objects.filter(...) / Coupon.find(id) ──> SELECT
coupon (row)    ── coupon.save() / coupon.delete()              ──> INSERT/UPDATE/DELETE
                ── coupon.is_valid_for(order_total)             (light domain logic)
```

## When to use
- CRUD and simple workflows; schema and objects match one-to-one.
- A framework built around it (Django, Rails, Laravel) where the ecosystem (admin, forms, serializers) assumes it.
- Small teams and fast iteration matter more than persistence ignorance.

## When NOT to use
- Rich invariants spanning several objects, or an object shape that differs from the schema: use Domain Model + Data Mapper.
- You need unit tests of business rules without a database at scale.
- Heavy reporting: write queries/read models, not per-row objects.

## Active Record vs Data Mapper — decision
| Question | Leans Active Record | Leans Data Mapper |
|---|---|---|
| Business logic complexity | low, mostly validation | high, invariants across objects |
| Object ↔ table shape | 1:1 | value objects, aggregates over several tables, legacy schema |
| Framework | Django, Rails, Laravel | SQLAlchemy, Hibernate, MikroORM, FastAPI + DDD |
| Test strategy | DB-backed tests are acceptable (pytest-django, transactions per test) | fast pure unit tests with fakes |
| Expected lifespan / growth | short-lived or stable scope | long-lived core domain that will grow |
| Team | familiar with the framework conventions | comfortable with layering and mapping |
A codebase can mix both per bounded context: Active Record for the admin/catalog, Data Mapper for billing. Do not mix them for the *same* aggregate.

## Example — Python
Django: the model is the Active Record. Keep logic small and query logic in a custom manager/QuerySet.

```python
from decimal import Decimal
from django.db import models, transaction
from django.db.models import F
from django.utils import timezone

class CouponQuerySet(models.QuerySet["Coupon"]):
    def active(self) -> "CouponQuerySet":
        return self.filter(expires_at__gt=timezone.now(), uses_left__gt=0)

class Coupon(models.Model):
    code = models.CharField(max_length=32, unique=True)
    percent_off = models.PositiveSmallIntegerField()
    min_total = models.DecimalField(max_digits=10, decimal_places=2, default=Decimal(0))
    uses_left = models.PositiveIntegerField(default=1)
    expires_at = models.DateTimeField()

    objects = CouponQuerySet.as_manager()

    def is_valid_for(self, total: Decimal) -> bool:
        return self.uses_left > 0 and self.expires_at > timezone.now() and total >= self.min_total

    def discount(self, total: Decimal) -> Decimal:
        return (total * self.percent_off / 100).quantize(Decimal("0.01"))

    def redeem(self) -> None:
        # atomic decrement in SQL avoids the read-modify-write race
        with transaction.atomic():
            updated = Coupon.objects.filter(pk=self.pk, uses_left__gt=0).update(
                uses_left=F("uses_left") - 1)
            if not updated:
                raise ValueError("coupon exhausted")
            self.refresh_from_db(fields=["uses_left"])
```
FastAPI has no built-in Active Record; SQLModel/SQLAlchemy declarative models are often *used* AR-style (`session.add(obj)`), but saving still goes through the session (a Data Mapper underneath).

## Example — TypeScript
TypeORM supports Active Record by extending `BaseEntity`.

```ts
import { BaseEntity, Column, Entity, MoreThan, PrimaryGeneratedColumn } from 'typeorm';

@Entity('coupons')
export class Coupon extends BaseEntity {
  @PrimaryGeneratedColumn('uuid') id!: string;
  @Column({ unique: true }) code!: string;
  @Column('smallint') percentOff!: number;
  @Column('int') usesLeft!: number;
  @Column('timestamptz') expiresAt!: Date;

  static findActive(code: string): Promise<Coupon | null> {
    return this.findOneBy({ code, usesLeft: MoreThan(0), expiresAt: MoreThan(new Date()) });
  }

  discountCents(totalCents: number): number {
    return Math.round((totalCents * this.percentOff) / 100);
  }
}

// usage
const c = await Coupon.findActive('WELCOME10');
if (c) { c.usesLeft -= 1; await c.save(); }   // racy under concurrency, see Pitfalls
```

## In the wild
- Django ORM, Ruby on Rails ActiveRecord, Laravel Eloquent, Peewee, Tortoise ORM, TypeORM `BaseEntity`, Sequelize models.
- Pydantic models are *not* Active Record — they have no persistence; pair them as DTOs.

## Trade-offs
| Pros | Cons |
|---|---|
| Minimal code; very fast to build CRUD | Domain objects coupled to DB schema and framework |
| Ecosystem tooling (admin, migrations, forms) | Business rules tested only with a DB |
| Easy to read for newcomers | Logic sprawls into fat models over time |
| Querying is ergonomic | Hard to model aggregates, value objects, legacy schemas |

## Pitfalls
- **Fat models:** every rule lands on the model until it is 2000 lines. Fix: extract services/domain classes for workflows; keep model logic local to one row.
- **Read-modify-write races** (`obj.count -= 1; obj.save()`). Fix: `F()` expressions/atomic `UPDATE`, `select_for_update`, or optimistic version columns.
- **N+1 queries** from relation access in loops. Fix: `select_related`/`prefetch_related`, or `relations` in TypeORM.
- **Returning models straight from APIs:** leaks columns, couples API to schema. Fix: DTOs/serializers.
- **Signals/hooks (`post_save`) doing business work:** hidden side effects. Fix: explicit service calls.

## Refactoring path
- **Introduce it:** from raw SQL scattered across handlers in a CRUD app: define models for the tables, replace SQL with model queries, move per-row helpers onto the model.
- **Remove it / simplify (towards Data Mapper):** when invariants grow, extract a plain domain class for the core aggregate; keep the AR model as the persistence model; add `to_domain()/from_domain()`; route writes through a Repository; finally stop calling `.save()` from services.

## Related patterns
- [Data Mapper](../data-access/data-mapper.md) — the persistence-ignorant alternative.
- [Row Data Gateway](../data-access/row-data-gateway.md) — the same "object per row" without domain logic.
- [Repository](../data-access/repository.md) — usually redundant over AR; Django managers already fill the role.
- [Domain Model](../data-access/domain-model.md) — where you go when AR models get fat.
- [DTO](../enterprise/dto.md) — keep API shapes separate from AR models.
