# Bridge

**Category:** Structural · **Also known as:** Handle/Body · **Related:** Adapter, Strategy, Abstract Factory, Dependency Injection

## Intent
Split an abstraction from its implementation into two separate hierarchies connected by composition, so each can vary and grow independently.

## Problem
- Two independent dimensions of variation are being encoded in one inheritance tree, producing a Cartesian product of subclasses: `EmailOrderAlert`, `SmsOrderAlert`, `EmailPaymentAlert`, `SmsPaymentAlert`, `SlackPaymentAlert`…
- Adding one new value on either axis requires N new classes.
- High-level logic (what to say, when, to whom) is tangled with low-level mechanics (how bytes are sent, how pixels are drawn).

Scenario: a notification system has message kinds (order shipped, payment failed, password reset) and delivery channels (email, SMS, push). With inheritance you need kinds × channels classes; with Bridge you need kinds + channels.

## Solution and structure
- **Abstraction**: the high-level API clients use; holds a reference to an Implementor.
- **Refined Abstraction**: variants of the abstraction (`PaymentFailedAlert`).
- **Implementor**: the low-level interface (`Channel.send(recipient, subject, body)`).
- **Concrete Implementor**: `EmailChannel`, `SmsChannel`.

```
Abstraction ────has-a────> Implementor
    ▲                          ▲
RefinedAbstractionA      ConcreteImplA
RefinedAbstractionB      ConcreteImplB
```

The difference from Strategy is intent and scope: Bridge is a structural decision that the *whole* abstraction is built on a pluggable implementation layer, usually chosen at construction time and rarely swapped; Strategy swaps one algorithm, often per call.

## When to use
- Two (or more) orthogonal axes of variation that would otherwise multiply subclasses.
- You want to ship a platform-independent API over platform-specific backends (drivers, renderers, storage engines).
- Implementations must be selectable at runtime or configuration time without touching the high-level code.

## When NOT to use
- There is only one axis of variation: plain polymorphism or Strategy is enough.
- There is only one implementation and no realistic second one: the extra interface is speculative.
- You are integrating an existing incompatible class: that is Adapter.
- In Python/TS, if the "implementor" is one function, pass a callable instead of a class hierarchy.

## Example — Python
```python
from dataclasses import dataclass
from typing import Protocol


class Channel(Protocol):                       # Implementor
    def send(self, to: str, subject: str, body: str) -> None: ...


class EmailChannel:
    def __init__(self, smtp_host: str) -> None:
        self.smtp_host = smtp_host

    def send(self, to: str, subject: str, body: str) -> None:
        print(f"[smtp {self.smtp_host}] {to}: {subject}\n{body}")


class SmsChannel:
    def send(self, to: str, subject: str, body: str) -> None:
        print(f"[sms] {to}: {subject} - {body[:140]}")


@dataclass
class Notification:                           # Abstraction
    channel: Channel

    def notify(self, to: str) -> None:
        self.channel.send(to, self.subject(), self.body())

    def subject(self) -> str: raise NotImplementedError
    def body(self) -> str: raise NotImplementedError


@dataclass
class PaymentFailed(Notification):            # Refined abstraction
    order_id: str = ""

    def subject(self) -> str: return "Payment failed"
    def body(self) -> str: return f"We could not charge order {self.order_id}."


@dataclass
class OrderShipped(Notification):
    tracking: str = ""

    def subject(self) -> str: return "Your order is on its way"
    def body(self) -> str: return f"Track it: {self.tracking}"


PaymentFailed(SmsChannel(), order_id="A-17").notify("+380501112233")
OrderShipped(EmailChannel("smtp.local"), tracking="UA123").notify("a@b.com")
```
Three kinds × three channels = six classes, not nine.

## Example — TypeScript
Angular version: the abstraction is a chart component API; the implementor is a renderer injected via a token, so the same chart logic draws to Canvas or SVG.

```ts
import { InjectionToken, Injectable, inject } from '@angular/core';

export interface ChartRenderer {                       // Implementor
  line(points: ReadonlyArray<[number, number]>, color: string): void;
  clear(): void;
}
export const CHART_RENDERER = new InjectionToken<ChartRenderer>('ChartRenderer');

export class CanvasRenderer implements ChartRenderer {
  constructor(private readonly ctx: CanvasRenderingContext2D) {}
  clear(): void { this.ctx.clearRect(0, 0, this.ctx.canvas.width, this.ctx.canvas.height); }
  line(pts: ReadonlyArray<[number, number]>, color: string): void {
    this.ctx.strokeStyle = color;
    this.ctx.beginPath();
    pts.forEach(([x, y], i) => (i ? this.ctx.lineTo(x, y) : this.ctx.moveTo(x, y)));
    this.ctx.stroke();
  }
}

@Injectable()
export abstract class Chart {                          // Abstraction
  protected readonly renderer = inject(CHART_RENDERER);
  abstract draw(values: number[]): void;
}

@Injectable()
export class RevenueChart extends Chart {              // Refined abstraction
  draw(values: number[]): void {
    this.renderer.clear();
    this.renderer.line(values.map((v, i) => [i * 20, 200 - v]), '#2a7');
  }
}
```

## In the wild
- JDBC / Python DB-API / SQLAlchemy `Engine` + dialect: one query abstraction over many drivers.
- Python `logging`: `Logger` (abstraction: levels, hierarchy) × `Handler` (implementor: file, stream, syslog, HTTP). Adding a handler does not change loggers.
- Angular `Renderer2`: components use one rendering API; the platform supplies browser, server (SSR) or other renderers.
- Node `fs` abstraction over libuv per-OS implementations; ORMs like Prisma/TypeORM over database connectors.

## Trade-offs
| Pros | Cons |
|---|---|
| Kinds + channels classes instead of kinds × channels | Up-front design cost; must identify the axes correctly |
| Implementations swappable at configuration time | Implementor interface tends toward lowest common denominator |
| High-level and low-level code evolve and are tested separately | Extra indirection makes simple flows harder to trace |
| Hides platform details from clients | Wrong split is expensive to undo |

## Pitfalls
- **Fat implementor interface** that mirrors one platform. Fix: define primitive operations every implementation can support; build richer behavior in the abstraction.
- **Leaking implementation checks** (`if isinstance(self.channel, SmsChannel)`) into the abstraction. Fix: add a capability method or move the difference into the implementor.
- **Bridge with one implementation forever**. Fix: inline it until a second implementation exists.
- **Confusing Bridge with Adapter**: Bridge is designed before the parts exist; Adapter retrofits existing ones.

## Refactoring path
- **Introduce it:** spot a class hierarchy whose names combine two words from two lists (`SmsPaymentAlert`). Extract the second axis into an interface, move its code into implementor classes, pass an implementor into the abstraction's constructor, then collapse the combined subclasses.
- **Remove it / simplify:** if only one implementor survives, inline its methods into the abstraction and delete the interface; if only one abstraction exists, the implementor is likely just a Strategy or a function parameter.

## Related patterns
- [Adapter](../structural/adapter.md) — makes existing incompatible parts fit after the fact; Bridge separates them by design.
- [Strategy](../behavioral/strategy.md) — swaps one algorithm; Bridge splits an entire abstraction from its implementation layer.
- [Abstract Factory](../creational/abstract-factory.md) — can create and configure the right implementor family.
- [Dependency Injection](../behavioral/dependency-injection.md) — the usual way to hand the implementor to the abstraction.
- [Wrapper](../structural/wrapper.md) — overview of composition-based structural patterns.
