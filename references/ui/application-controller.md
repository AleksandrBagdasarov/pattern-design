# Application Controller

**Category:** UI · **Also known as:** Flow controller, wizard coordinator, navigation coordinator · **Related:** Front Controller, Page Controller, State, Mediator, Saga

## Intent
Centralize screen flow — which view comes next and which command runs — in one object, instead of hard-coding navigation inside each page or component.

## Problem
- Multi-step flows (checkout, onboarding, KYC, loan application) where the next step depends on state: "skip shipping for digital goods", "require ID upload if the amount > €10k".
- Each step component calls `router.navigate(['/step-3'])` itself, so the flow is spread across files and nobody can see it whole.
- Adding or reordering a step means editing several components; deep links into step 4 without steps 1–3 done break the app.

Scenario: checkout has Cart → Address → Shipping → Payment → Review. Digital-only carts skip Address and Shipping; corporate customers add a PO-number step. An application controller owns the transition table and validates entry to each step.

## Solution and structure
```
Step views ──"next"/"back" + data──▶ Application Controller ──decides──▶ next view / command
                                        │  flow definition (state machine or table)
                                        │  flow state (what's completed)
                                        ▼
                                   Domain services
```
- **Application Controller**: holds flow state and rules; answers "what is the next step?" and "may the user enter step X?".
- **Views/pages**: render their step and report outcomes (`completed(step, data)`), never navigate themselves.
- **Flow definition**: ideally declarative (a list/graph or a finite state machine), so it is readable and testable.

## When to use
- Wizards and workflows with conditional branching or skipping.
- The same flow must run in different shells (web and mobile, modal and full page).
- Flow rules change often and must be tested independently of UI.
- Guarding deep links: entering step N requires steps before N.

## When NOT to use
- Linear 2–3 step flows with no branching: a `@switch` on a `step` signal in one component is enough.
- Simple navigation between unrelated pages: the router (Front Controller) and links are sufficient.
- Long-running server-side business processes spanning services and days: that is a Saga / workflow engine (Temporal, Step Functions), not a UI controller.

## Example — Python
A server-side wizard flow (e.g. FastAPI + session) with a declarative transition function:
```python
from dataclasses import dataclass, field
from enum import StrEnum

class Step(StrEnum):
    CART = "cart"; ADDRESS = "address"; SHIPPING = "shipping"
    PO_NUMBER = "po-number"; PAYMENT = "payment"; REVIEW = "review"

@dataclass
class CheckoutState:
    digital_only: bool
    corporate: bool
    completed: set[Step] = field(default_factory=set)

class CheckoutFlow:
    def steps(self, s: CheckoutState) -> list[Step]:
        flow = [Step.CART]
        if not s.digital_only:
            flow += [Step.ADDRESS, Step.SHIPPING]
        if s.corporate:
            flow.append(Step.PO_NUMBER)
        return flow + [Step.PAYMENT, Step.REVIEW]

    def next(self, s: CheckoutState, current: Step) -> Step | None:
        s.completed.add(current)
        flow = self.steps(s)
        i = flow.index(current)
        return flow[i + 1] if i + 1 < len(flow) else None

    def can_enter(self, s: CheckoutState, step: Step) -> bool:
        flow = self.steps(s)
        return step in flow and all(p in s.completed for p in flow[: flow.index(step)])

def test_digital_corporate_flow_skips_shipping() -> None:
    flow, s = CheckoutFlow(), CheckoutState(digital_only=True, corporate=True)
    assert flow.next(s, Step.CART) == Step.PO_NUMBER
    assert not flow.can_enter(s, Step.PAYMENT)
```
Page handlers call `flow.next(...)` and redirect to the returned step; guards call `can_enter`.

## Example — TypeScript
```ts
import { Injectable, computed, inject, signal } from '@angular/core';
import { CanActivateFn, Router } from '@angular/router';

type Step = 'cart' | 'address' | 'shipping' | 'po' | 'payment' | 'review';

@Injectable({ providedIn: 'root' })
export class CheckoutFlow {
  private readonly router = inject(Router);
  readonly digitalOnly = signal(false);
  readonly corporate = signal(false);
  private readonly completed = signal<ReadonlySet<Step>>(new Set());

  readonly steps = computed<Step[]>(() => [
    'cart',
    ...(this.digitalOnly() ? [] : (['address', 'shipping'] as const)),
    ...(this.corporate() ? (['po'] as const) : []),
    'payment', 'review',
  ]);

  complete(step: Step): Promise<boolean> {
    this.completed.update((s) => new Set(s).add(step));
    const flow = this.steps();
    const next = flow[flow.indexOf(step) + 1];
    return this.router.navigate(['/checkout', next ?? 'done']);
  }

  canEnter(step: Step): boolean {
    const flow = this.steps();
    const i = flow.indexOf(step);
    return i >= 0 && flow.slice(0, i).every((s) => this.completed().has(s));
  }
}

export const checkoutStepGuard: CanActivateFn = (route) => {
  const flow = inject(CheckoutFlow);
  return flow.canEnter(route.paramMap.get('step') as Step) || inject(Router).parseUrl('/checkout/cart');
};
```
Step components call `flow.complete('address')` and never touch the router.

## In the wild
- Spring Web Flow, Struts/J2EE Application Controller (Core J2EE Patterns), ASP.NET wizard controls.
- XState (state machines for UI flows, works with Angular/React), iOS "Coordinator" pattern, Android Navigation graph.
- Angular route guards + a flow service as above; NgRx effects can also act as the flow controller.

## Trade-offs
| Pros | Cons |
|---|---|
| Whole flow visible and testable in one place | Another layer for simple navigation |
| Steps are reusable and unaware of each other | Controller can become a god object across many flows |
| Easy to add/skip/reorder steps | Must keep browser back button and URLs in sync with flow state |
| Guards deep links consistently | Flow state persistence (refresh) needs explicit handling |

## Pitfalls
- **Flow state lost on reload:** persist it (session storage, server session, URL) or rebuild it from server data.
- **Components still navigating directly:** grep for `router.navigate` in step components; route all transitions through the controller.
- **One controller for every flow:** create one per flow (checkout, onboarding), provided at the flow's route level if state must reset.
- **Imperative if/else spaghetti inside the controller:** express transitions as data or a state machine.

## Refactoring path
- **Introduce it:** collect all `navigate` calls in a flow into one service with `next(step)`; replace in components; add a guard using `canEnter`; extract the step list into data.
- **Remove it / simplify:** if the flow became linear, replace the controller with a `step` signal and `@switch` in one component.

## Related patterns
- [Front Controller](../ui/front-controller.md) — handles every request; application controller decides flow within it.
- [Page Controller](../ui/page-controller.md) — per-page handlers that delegate flow decisions here.
- [State](../behavioral/state.md) — the flow is often a state machine.
- [Mediator](../behavioral/mediator.md) — coordinates step components without them knowing each other.
- [Saga](../cloud/saga.md) — the backend analogue for long-running multi-service processes.
