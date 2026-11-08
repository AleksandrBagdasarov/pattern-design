# Component-Oriented Programming (COP)

**Category:** Structural (component patterns) · **Also known as:** Component-based software engineering (CBSE), Plugin components · **Related:** Module, Microkernel, Component-Based Design, Dependency Injection, Hexagonal, Microservices, Facade

## Intent
Build systems from independently developed, replaceable **components**: units with explicit provided and required interfaces (a contract), that can be composed, versioned and deployed without the consumer knowing their internals.

## Problem
- Features are built as tightly coupled code: replacing the payment provider or the date picker means touching many files across the app.
- Teams cannot ship pieces independently; every change needs a full coordinated release.
- Reuse across projects happens by copy-paste because there is no unit with a stable contract.
- Dependencies are hidden (a component reaches for globals), so it cannot be moved or tested in isolation.

Scenario: a company has three Angular apps that all need the same address-entry widget with validation and autocomplete. A component with a contract (inputs, outputs, form integration, required `AddressLookup` service) can be published once as a library and used by all three.

## Solution and structure
A component in COP has:
- **Provided interface**: what it offers (public methods, inputs/outputs, events, HTTP API).
- **Required interface**: what it needs from the environment, declared explicitly (constructor/DI dependencies, injection tokens, config), never fetched from globals.
- **Contract**: types plus behavioral guarantees (pre/postconditions, events emitted, error semantics), versioned (semver).
- **Container / framework**: instantiates components, satisfies their required interfaces (DI), manages lifecycle.
- **Composition**: an assembler wires components together; components do not know each other's concrete types.

```
 [AddressForm] provides: value, valid, (addressChange)
               requires: ADDRESS_LOOKUP ───> [GoogleLookup] | [NominatimLookup] | [FakeLookup]
```
Scale varies: a UI component, a library, a plugin, a service. Szyperski's definition: "a unit of composition with contractually specified interfaces and explicit context dependencies only."

## When to use
- Reusable building blocks shared by several apps or teams (design systems, SDKs, plugins).
- Systems that must allow third-party extensions or swapping implementations without recompiling the host.
- Independent release cycles per component.
- Frontend apps in general: Angular, React and Vue are component-oriented by design.

## When NOT to use
- Code used in exactly one place, by one team, that changes with its caller: a plain module/class is cheaper than a versioned contract.
- Premature extraction to a shared library before the API is understood: you will version-churn. Duplicate first, extract on the third use.
- Network-level components (microservices) when in-process modules would do: start with in-process components.

## Example — Python
A component with an explicit provided interface, explicit required interfaces and composition at the edge (FastAPI). Python packages + entry points make it deployable as a plugin.

```python
from collections.abc import Callable
from dataclasses import dataclass
from typing import Protocol

from fastapi import APIRouter, Depends, FastAPI


class GeoLookup(Protocol):                      # required interface
    def suggest(self, query: str, limit: int) -> list[str]: ...


@dataclass(frozen=True)
class AddressConfig:                            # required configuration
    max_suggestions: int = 5


def create_address_component(                   # provided interface: a router factory
    get_lookup: Callable[[], GeoLookup],
    config: AddressConfig = AddressConfig(),
) -> APIRouter:
    router = APIRouter(prefix="/address", tags=["address"])

    @router.get("/suggest")
    def suggest(q: str, lookup: GeoLookup = Depends(get_lookup)) -> list[str]:
        if len(q) < 3:
            return []
        return lookup.suggest(q, config.max_suggestions)

    return router


# --- host application: the assembler ---
class FakeLookup:
    def suggest(self, query: str, limit: int) -> list[str]:
        return [f"{query} St. {i}" for i in range(1, limit + 1)]

app = FastAPI()
app.include_router(create_address_component(get_lookup=FakeLookup))
# A plugin host could discover components via importlib.metadata.entry_points(group="myapp.components").
```

## Example — TypeScript
An Angular standalone component whose contract is its inputs/outputs plus a required injection token; the host supplies the implementation.

```ts
import { Component, InjectionToken, inject, input, output, signal } from '@angular/core';
import { toObservable, toSignal } from '@angular/core/rxjs-interop';
import { Observable, debounceTime, distinctUntilChanged, filter, switchMap } from 'rxjs';

export interface AddressLookup { suggest(q: string): Observable<string[]> }      // required
export const ADDRESS_LOOKUP = new InjectionToken<AddressLookup>('ADDRESS_LOOKUP');

@Component({
  selector: 'lib-address-input',
  template: `
    <input [placeholder]="placeholder()" (input)="query.set($any($event.target).value)" />
    <ul>
      @for (s of suggestions(); track s) { <li (click)="picked.emit(s)">{{ s }}</li> }
    </ul>
  `,
})
export class AddressInput {                                   // provided contract:
  readonly placeholder = input('Start typing an address');  //   inputs
  readonly picked = output<string>();                       //   outputs
  private readonly lookup = inject(ADDRESS_LOOKUP);         //   explicit dependency

  protected readonly query = signal('');
  protected readonly suggestions = toSignal(
    toObservable(this.query).pipe(
      debounceTime(300), distinctUntilChanged(), filter((q) => q.length >= 3),
      switchMap((q) => this.lookup.suggest(q)),
    ),
    { initialValue: [] as string[] },
  );
}

// Host app: providers: [{ provide: ADDRESS_LOOKUP, useClass: NominatimLookup }]
// Template:  <lib-address-input (picked)="form.patchValue({ address: $event })" />
```
For form integration, implement `ControlValueAccessor` so the component also satisfies Angular's forms contract.

## In the wild
- Angular/React/Vue/Web Components (Custom Elements: a browser-standard component contract).
- Angular libraries built with ng-packagr and published to npm; Angular Material and CDK.
- Python: packages with entry points (pytest plugins via `pluggy`, Flask blueprints, FastAPI routers), Airflow providers.
- Older and enterprise: COM, CORBA, JavaBeans/EJB, OSGi bundles, .NET assemblies.
- Module Federation / Native Federation for runtime-loaded micro-frontends (a larger-grained, higher-cost form).

## Trade-offs
| Pros | Cons |
|---|---|
| Reuse across apps and teams via stable contracts | Contract design and versioning are real, ongoing work |
| Replace implementations without touching consumers | Generic components accumulate options ("prop explosion") |
| Independent development, testing and release | Integration issues move to composition time; need contract tests |
| Explicit dependencies make testing with fakes easy | Indirection and packaging overhead for small teams |

## Pitfalls
- **Hidden dependencies** (component reads a global store, `window`, or a root singleton it did not declare). Fix: declare every dependency as an input or injection token.
- **Leaky contracts** exposing internal state or DOM structure that consumers start relying on. Fix: minimal public API; style hooks via CSS custom properties, not deep selectors.
- **Breaking changes without semver.** Fix: version, changelog, deprecate before removing; contract tests in CI.
- **Over-configurable components** with dozens of inputs. Fix: composition (content projection, slots, smaller components) over configuration.
- **Shared mutable state between component instances** via `providedIn: 'root'` when per-instance state was intended. Fix: component-level `providers`.

## Refactoring path
- **Introduce it:** find a piece duplicated across apps or a feature with a clear boundary; list what it provides and what it needs; replace implicit dependencies with explicit inputs/tokens; move it to a library; publish with semver; migrate consumers one by one.
- **Remove it / simplify:** if a "shared" component has one consumer and changes with it every release, move it back into that app; collapse over-generic components into specific ones.

## Related patterns
- [Module](../structural/module.md) — encapsulation boundary; a component adds required interfaces, versioning and deployability.
- [Microkernel](../architectural/microkernel.md) — a host core loading plugin components.
- [Component-Based Design](../structural/component-based-design.md) — composing one object from parts; COP composes systems from deployable units.
- [Dependency Injection](../behavioral/dependency-injection.md) — how a container satisfies required interfaces.
- [Hexagonal](../architectural/hexagonal.md) — ports are a component's required/provided interfaces.
- [Microservices](../architectural/microservices.md) — components separated by a network boundary.
- [Facade](../structural/facade.md) — a component's provided interface often acts as a facade over its internals.
