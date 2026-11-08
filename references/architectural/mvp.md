# Model-View-Presenter (MVP)

**Category:** Architectural · **Also known as:** Passive View, Supervising Controller (variants) · **Related:** MVC, MVVM, Presentation Model, Humble Object

## Intent
Make the View as dumb as possible (a "humble" object behind an interface) and put all presentation logic in a **Presenter** that reads the Model and tells the View exactly what to display, so the presenter is fully unit-testable with a fake view.

## Problem
- UI toolkit code (widgets, activities, forms) is hard or slow to test, and logic is buried in event handlers.
- There is no data-binding framework, or binding is too limited.
- Scenario: a desktop (Qt/Tkinter) or legacy Android login screen. Validation, error messages, spinner state and navigation are all written in button-click handlers. Tests require instantiating real widgets.

## Solution and structure
```
  View (widgets) ──user events──► Presenter ──calls──► Model (services)
       ▲                              │
       └──── view.show_error(...) ────┘   via IView interface
```
- **View:** implements an interface (`LoginView`) with methods like `show_error`, `set_busy`, `navigate_home`; forwards events to the presenter. No logic.
- **Presenter:** holds a reference to the view *interface*, reacts to events, calls the model, decides what the view shows.
- **Model:** services, domain, API clients.
- **Passive View** variant: the presenter sets every field. **Supervising Controller:** simple bindings allowed; presenter handles complex logic.
- Difference from MVVM: the presenter knows the view (via interface) and pushes updates; a view model does not know the view.

## When to use
- UI toolkits without good data binding (Tkinter, WinForms, classic Android Views, GWT).
- When presentation logic must be tested exhaustively without the UI runtime.
- Legacy UIs where you want to extract testable logic incrementally.

## When NOT to use
- Angular, Vue, React, SwiftUI, Compose: their reactive models make MVVM or component state the idiom; an extra presenter interface fights the framework.
- Trivial dialogs: the interface + presenter doubles code for no benefit.
- Smell: a view interface with 25 methods; split the screen or switch to a bound view model.

## Example — Python
```python
from typing import Protocol
from dataclasses import dataclass

class LoginView(Protocol):
    def set_busy(self, busy: bool) -> None: ...
    def show_error(self, message: str) -> None: ...
    def go_to_dashboard(self, user_name: str) -> None: ...

class AuthService(Protocol):
    def login(self, email: str, password: str) -> str: ...   # returns display name; raises PermissionError

@dataclass
class LoginPresenter:
    view: LoginView
    auth: AuthService

    def on_submit(self, email: str, password: str) -> None:
        if "@" not in email:
            self.view.show_error("Enter a valid email")
            return
        self.view.set_busy(True)
        try:
            name = self.auth.login(email, password)
        except PermissionError:
            self.view.show_error("Wrong email or password")
        else:
            self.view.go_to_dashboard(name)
        finally:
            self.view.set_busy(False)

# Test with a recording fake view — no Qt/Tk needed
@dataclass
class FakeView:
    calls: list[tuple[str, object]]
    def set_busy(self, busy: bool) -> None: self.calls.append(("busy", busy))
    def show_error(self, message: str) -> None: self.calls.append(("error", message))
    def go_to_dashboard(self, user_name: str) -> None: self.calls.append(("nav", user_name))

class DenyAll:
    def login(self, email: str, password: str) -> str: raise PermissionError

view = FakeView([])
LoginPresenter(view, DenyAll()).on_submit("a@b.c", "x")
assert ("error", "Wrong email or password") in view.calls
```
The real Qt/Tk window implements `LoginView` and calls `presenter.on_submit(...)` from its button handler.

## Example — TypeScript
```ts
// Framework-free DOM view + presenter (e.g. a widget embedded in a legacy page)
export interface LoginView {
  setBusy(busy: boolean): void;
  showError(msg: string): void;
  goToDashboard(name: string): void;
}
export interface AuthService { login(email: string, pw: string): Promise<string> }

export class LoginPresenter {
  constructor(private readonly view: LoginView, private readonly auth: AuthService) {}
  async submit(email: string, pw: string): Promise<void> {
    if (!email.includes('@')) return this.view.showError('Enter a valid email');
    this.view.setBusy(true);
    try { this.view.goToDashboard(await this.auth.login(email, pw)); }
    catch { this.view.showError('Wrong email or password'); }
    finally { this.view.setBusy(false); }
  }
}

export class DomLoginView implements LoginView {
  private presenter: LoginPresenter;
  constructor(private root: HTMLFormElement, auth: AuthService) {
    this.presenter = new LoginPresenter(this, auth);
    root.addEventListener('submit', e => {
      e.preventDefault();
      const f = new FormData(root);
      void this.presenter.submit(String(f.get('email')), String(f.get('password')));
    });
  }
  setBusy(b: boolean) { this.root.querySelector('button')!.disabled = b; }
  showError(m: string) { this.root.querySelector('.error')!.textContent = m; }
  goToDashboard(name: string) { location.assign(`/dashboard?u=${encodeURIComponent(name)}`); }
}
```

## In the wild
- Classic Android (pre-Jetpack ViewModel) apps, GWT, WinForms, Delphi; Mosby/Moxy libraries.
- Microsoft's Smart Client guidance; Fowler's "Passive View" and "Supervising Controller" write-ups.
- In modern web frameworks it is rare; Angular "smart/container + dumb/presentational components" resembles Supervising Controller but uses binding, not a view interface.

## Trade-offs
| Pros | Cons |
|---|---|
| Presentation logic 100% testable with fakes | Boilerplate: view interface + presenter per screen |
| Works without any binding framework | Presenter-to-view calls are imperative and chatty |
| View can be swapped (desktop/web/test) | Presenters grow large on complex screens |
| Clear incremental refactoring target for legacy UIs | Lifecycle (rotation, view recreation) must be handled manually |

## Pitfalls
- **Logic leaking back into the view:** "just one `if` in the click handler". Every decision belongs in the presenter.
- **Presenter referencing concrete widgets:** depend on the view interface only, or tests need the toolkit.
- **Memory leaks:** presenter keeps a reference to a destroyed view (Android). Detach in the view's teardown.
- **Async callbacks after view teardown:** check an `attached` flag or cancel tasks on detach.

## Refactoring path
- **Introduce it:** pick one screen; write the view interface from what the handlers currently do to widgets; move handler bodies into a presenter that calls the interface; implement the interface in the existing window; add tests with a fake view.
- **Remove it / simplify:** when migrating to a binding framework, turn presenter fields into observable view-model state and delete the view interface (MVP → MVVM).

## Related patterns
- [MVVM](../architectural/mvvm.md) — replaces the view interface with data binding.
- [MVC](../architectural/mvc.md) — controller handles input; view may read the model directly.
- [Presentation Model](../ui/presentation-model.md) — state-holding alternative where the view pulls from the model.
- [Test Spy](../testing/test-spy.md) — the recording fake view used to test presenters.
