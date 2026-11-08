# Interface Injection

**Category:** Behavioral (injection) · **Also known as:** Injection via callback interface, "Aware" interfaces (Spring), Type 1 IoC · **Related:** Dependency Injection, Setter Injection, Constructor Injection, Template Method, Microkernel, Observer

## Intent
Let the dependency's provider define an *injector interface* (for example `inject_clock(clock)` or `registerOnChange(fn)`) that consumers implement; the injector detects the interface and calls that method to hand the dependency over.

## Problem
- A framework or host must give many independently written components (plugins, form controls, handlers) a dependency, but it does not construct them through a constructor it controls, or the components are created by users with arbitrary constructors.
- The host needs a uniform, discoverable contract: "if you implement this, I will call it with X".
- Smells: plugins reaching into globals to find the host (`app = current_app()` everywhere); ad-hoc `setup()` methods with inconsistent signatures; host code full of `hasattr(plugin, "set_db")` checks without a declared contract.

Scenario: a custom Angular form control (a star rating) must receive a callback from the forms API so it can report value changes. Angular doesn't construct it with that callback; instead the control implements `ControlValueAccessor`, and the forms directive calls `registerOnChange(fn)` and `registerOnTouched(fn)`.

## Solution and structure
1. The **injector interface** declares one method per dependency: `register_x(x)`, `inject_x(x)`, `set_x_aware(x)`.
2. The **consumer** implements it.
3. The **injector** (framework, container, host) checks whether an object implements the interface (`isinstance` against a runtime-checkable Protocol, or a structural check) and calls the method.

```
          «interface» ClockAware
            inject_clock(clock)
                  ▲ implements
   ScheduledReport ─┘

Host/Container:  for obj in components:
                     if isinstance(obj, ClockAware): obj.inject_clock(clock)
```

Difference from Setter Injection: the setter is not chosen by the consumer; it is a contract defined by the injector, which is what lets the injector apply it generically to unknown classes. Fowler listed it as the third DI form (from the Avalon framework); today it survives mainly as framework callback interfaces.

## When to use
- Building a framework/host that injects a few well-known dependencies into third-party or user-defined components (plugins, form controls, handlers).
- The consumer's construction is outside your control, but you can require it to implement an interface.
- Opt-in capabilities: only components that need X implement `XAware`; others are untouched.

## When NOT to use
- Application code with a DI container or framework DI (FastAPI, Angular `inject()`, NestJS): use constructor injection or `inject()`; interface injection adds a contract per dependency for no gain.
- When you control construction: constructor injection gives stronger guarantees (valid after construction).
- Many dependencies: one interface per dependency multiplies boilerplate. Pass a single context object to a `setup(ctx)` hook instead (Microkernel style).
- When you are *consuming* a framework that already defines such interfaces: implement theirs, don't invent your own parallel one.

## Example — Python
A host that injects services into plugins that opt in via runtime-checkable Protocols.

```python
from datetime import datetime, timezone
from typing import Protocol, runtime_checkable
import httpx

@runtime_checkable
class ClockAware(Protocol):
    def inject_clock(self, now: "Clock") -> None: ...

@runtime_checkable
class HttpAware(Protocol):
    def inject_http(self, client: httpx.AsyncClient) -> None: ...

class Clock(Protocol):
    def __call__(self) -> datetime: ...

class PluginHost:
    def __init__(self, http: httpx.AsyncClient, clock: Clock) -> None:
        self._http, self._clock = http, clock
        self.plugins: list[object] = []

    def register(self, plugin: object) -> None:
        # The injector detects the interfaces and calls the injector methods.
        if isinstance(plugin, ClockAware):
            plugin.inject_clock(self._clock)
        if isinstance(plugin, HttpAware):
            plugin.inject_http(self._http)
        self.plugins.append(plugin)

class SlackDigest:                       # third-party plugin, arbitrary constructor
    def __init__(self, channel: str) -> None:
        self.channel = channel
    def inject_clock(self, now: Clock) -> None:
        self._now = now
    def inject_http(self, client: httpx.AsyncClient) -> None:
        self._http = client
    async def run(self) -> None:
        await self._http.post("https://slack.example/api", json={
            "channel": self.channel, "text": f"Digest {self._now():%Y-%m-%d}"})

# host = PluginHost(httpx.AsyncClient(), lambda: datetime.now(timezone.utc))
# host.register(SlackDigest("#sales"))
```

`isinstance` with `runtime_checkable` checks only method presence, not signatures; type checkers verify signatures statically.

## Example — TypeScript
Angular's `ControlValueAccessor` is interface injection: the forms API injects callbacks through `registerOnChange` / `registerOnTouched`.

```typescript
import { Component, forwardRef, signal } from '@angular/core';
import { ControlValueAccessor, NG_VALUE_ACCESSOR } from '@angular/forms';

@Component({
  selector: 'app-star-rating',
  providers: [{ provide: NG_VALUE_ACCESSOR, useExisting: forwardRef(() => StarRating), multi: true }],
  template: `
    @for (star of [1, 2, 3, 4, 5]; track star) {
      <button type="button" [disabled]="disabled()" (click)="select(star)" (blur)="onTouched()">
        {{ star <= value() ? '★' : '☆' }}
      </button>
    }
  `,
})
export class StarRating implements ControlValueAccessor {
  protected readonly value = signal(0);
  protected readonly disabled = signal(false);
  private onChange: (v: number) => void = () => {};     // Null Object until injected
  protected onTouched: () => void = () => {};

  // Injector methods called by the forms directive:
  registerOnChange(fn: (v: number) => void): void { this.onChange = fn; }
  registerOnTouched(fn: () => void): void { this.onTouched = fn; }
  writeValue(v: number | null): void { this.value.set(v ?? 0); }
  setDisabledState(isDisabled: boolean): void { this.disabled.set(isDisabled); }

  protected select(star: number): void {
    this.value.set(star);
    this.onChange(star);
  }
}
// usage: <app-star-rating formControlName="rating" />
```

Signal-based forms in newer Angular versions may offer a different custom-control contract; check angular.dev for the current API.

## In the wild
- Angular: `ControlValueAccessor` (`registerOnChange`, `registerOnTouched`), `Validator.registerOnValidatorChange`.
- Spring `*Aware` interfaces: `ApplicationContextAware.setApplicationContext`, `BeanNameAware`, `EnvironmentAware`.
- Python: Flask extensions' `init_app(app)` convention, Sphinx/pytest plugin hooks (`setup(app)`, `pytest_configure(config)`), `logging.Handler.setFormatter` called generically by `dictConfig`, `asyncio.Protocol.connection_made(transport)` (the event loop injects the transport).
- Node: `stream` implementers receive callbacks via `_write(chunk, enc, callback)`; Express middleware receives `next`.

## Trade-offs
| Pros | Cons |
|---|---|
| Host injects into classes it didn't construct | One interface per dependency: boilerplate |
| Opt-in: components declare exactly what they need | Object is incomplete until the injector calls it |
| Uniform, discoverable contract for plugins | Consumer coupled to the injector's interfaces |
| Works with arbitrary constructors | Rarely the right choice in application code |

## Pitfalls
- **Using the dependency before injection** (in `__init__` or before `register`). Fix: default to a Null Object (as `onChange = () => {}`) or fail with a clear error.
- **Inventing interface injection in app code** where framework DI exists. Fix: constructor injection / `inject()`.
- **`runtime_checkable` false positives**: any object with a method of that name passes. Fix: distinctive method names; static typing for signatures.
- **Forgetting `multi: true` / `NG_VALUE_ACCESSOR` provider** in Angular, so the forms API never injects (error "No value accessor for form control"). Fix: register the provider as shown.
- **Calling the injected `onChange` from `writeValue`.** Causes the form to mark itself dirty / loop. Fix: `writeValue` only updates the view; `onChange` only on user interaction.
- **Many `XAware` interfaces per plugin.** Fix: a single `setup(ctx)` hook with a typed context object.

## Refactoring path
- **Introduce it:** in a host with ad-hoc plugin wiring. (1) Define one Protocol per injectable service. (2) Make the host check and call them on registration. (3) Migrate plugins from globals to the injector methods. (4) Document the contract.
- **Remove it / simplify:** if you control construction of all consumers, replace each `inject_x` with a constructor parameter and build them in the composition root; if plugins need many services, collapse the interfaces into one `setup(ctx)`.

## Related patterns
- [Dependency Injection](dependency-injection.md) — the umbrella concept and its other forms.
- [Setter Injection](setter-injection.md) — the same mechanics without an injector-defined contract.
- [Constructor Injection](constructor-injection.md) — preferred whenever you control construction.
- [Microkernel](../architectural/microkernel.md) — plugin hosts are the main home of interface injection.
- [Template Method](template-method.md) — framework calls hooks you implement; interface injection is a hook whose purpose is delivering a dependency.
- [Observer](observer.md) — `registerOnChange(fn)` is also a subscription; the injected callback is the observer.
