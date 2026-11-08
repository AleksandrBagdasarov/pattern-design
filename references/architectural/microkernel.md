# Microkernel (Plug-in Architecture)

**Category:** Architectural · **Also known as:** plug-in architecture, extension architecture · **Related:** Service Locator, Strategy, Extension Object, Component-Oriented Programming, Hexagonal, Blackboard

## Intent
Keep a minimal, stable **core** that provides the essential workflow and extension points, and add features as independent **plug-ins** that register with the core through a defined contract, so the product can be extended (even by third parties) without changing the core.

## Problem
- Every customer or integration adds an `elif provider == "..."` branch to the core.
- Third parties need to extend the product without access to its source.
- Features must be enabled per installation or tenant.
- Scenario: a data-import tool supports CSV, Excel and three ERP formats. Each new format edits `importer.py`, risks regressions in others and ships with a full release. Partners want to add their own formats.

## Solution and structure
```
               ┌──────────── Core (kernel) ─────────────┐
 plug-in A ───►│ registry   lifecycle   extension points │◄─── plug-in C
 plug-in B ───►│ (discover, validate, load, call hooks)  │
               └─────────────────────────────────────────┘
```
- **Core:** the minimal workflow, plug-in discovery and lifecycle, shared services offered to plug-ins.
- **Plug-in contract:** interface/Protocol plus metadata (name, version, capabilities, API version).
- **Registry:** maps capability → plug-in; discovery via entry points, config, a directory scan or DI multi-providers.
- **Plug-ins:** independent modules implementing the contract; they depend on the core's contract, never on each other (unless the core models dependencies explicitly).

## When to use
- Products meant to be extended: IDEs, browsers, build tools, CLIs, CMSs, ETL tools, test frameworks.
- A family of interchangeable integrations (formats, payment providers, notification channels) growing over time.
- Per-customer feature sets in on-premise or white-label products.

## When NOT to use
- Two or three variants inside your own codebase: a dict of strategies is enough; you don't need discovery, versioning and lifecycle.
- Variants need deep access to core internals: the contract will leak until plug-ins are just core code in another folder.
- Smell: a "plug-in system" with one plug-in, or plug-ins that must be released together with the core every time.

## Directory layout
```
importer/
  core/        contract.py (FormatPlugin Protocol), registry.py, run.py
  plugins/     csv_format.py, xlsx_format.py           # built-in plug-ins use the same contract
pyproject.toml   [project.entry-points."importer.formats"] csv = "importer.plugins.csv_format:CsvFormat"
partner-pkg/pyproject.toml  [project.entry-points."importer.formats"] sap = "sap_fmt:SapFormat"
```

## Example — Python
```python
# core/contract.py + core/registry.py
from collections.abc import Iterator
from importlib.metadata import entry_points
from typing import BinaryIO, ClassVar, Protocol, runtime_checkable

CORE_API_VERSION = 2

@runtime_checkable
class FormatPlugin(Protocol):
    name: ClassVar[str]
    api_version: ClassVar[int]
    def accepts(self, filename: str) -> bool: ...
    def rows(self, data: BinaryIO) -> Iterator[dict[str, str]]: ...

class Registry:
    def __init__(self) -> None:
        self._plugins: dict[str, FormatPlugin] = {}

    def register(self, plugin: FormatPlugin) -> None:
        if not isinstance(plugin, FormatPlugin) or plugin.api_version != CORE_API_VERSION:
            raise TypeError(f"incompatible plug-in: {plugin!r}")
        self._plugins[plugin.name] = plugin

    def discover(self) -> None:
        for ep in entry_points(group="importer.formats"):   # installed packages advertise plug-ins
            try:
                self.register(ep.load()())
            except Exception as exc:                        # one broken plug-in must not kill the core
                print(f"skipping plug-in {ep.name}: {exc}")

    def for_file(self, filename: str) -> FormatPlugin:
        for p in self._plugins.values():
            if p.accepts(filename):
                return p
        raise LookupError(f"no format plug-in for {filename}")

# plugins/csv_format.py — a built-in plug-in, same contract as third-party ones
import csv, io

class CsvFormat:
    name: ClassVar[str] = "csv"
    api_version: ClassVar[int] = 2
    def accepts(self, filename: str) -> bool: return filename.endswith(".csv")
    def rows(self, data: BinaryIO) -> Iterator[dict[str, str]]:
        yield from csv.DictReader(io.TextIOWrapper(data, encoding="utf-8"))
```

## Example — TypeScript
```ts
// Angular: plug-ins as multi-providers on an InjectionToken; the core never imports them.
import { InjectionToken, Injectable, inject, Type, makeEnvironmentProviders } from '@angular/core';

export interface WidgetPlugin {
  readonly id: string;
  readonly title: string;
  loadComponent(): Promise<Type<unknown>>;          // lazy: plug-in code split into its own chunk
}
export const WIDGET_PLUGINS = new InjectionToken<WidgetPlugin[]>('WIDGET_PLUGINS', { factory: () => [] });

@Injectable({ providedIn: 'root' })
export class WidgetRegistry {                       // core
  private plugins = inject(WIDGET_PLUGINS);
  list() { return this.plugins.map(p => ({ id: p.id, title: p.title })); }
  async component(id: string) {
    const p = this.plugins.find(x => x.id === id);
    if (!p) throw new Error(`Unknown widget ${id}`);
    return p.loadComponent();                       // render with NgComponentOutlet
  }
}

// a plug-in package exposes a provider function:
export const provideSalesWidget = () => makeEnvironmentProviders([{
  provide: WIDGET_PLUGINS, multi: true,
  useValue: { id: 'sales', title: 'Sales', loadComponent: () => import('./sales-widget').then(m => m.SalesWidget) },
}]);
// app.config.ts: providers: [provideSalesWidget(), provideOpsWidget()]
```

## In the wild
- VS Code extensions, Eclipse (OSGi), browsers, Jenkins, WordPress, Grafana panels.
- pytest (pluggy hooks + `pytest11` entry points), Flake8/Ruff-style rule sets, Sphinx extensions, Airflow providers, Babel/ESLint/Vite/Rollup plug-ins, webpack loaders.
- Python `importlib.metadata.entry_points`; Angular multi-providers and `ENVIRONMENT_INITIALIZER`; NestJS dynamic modules.
- OS microkernels (Mach, QNX, seL4) are the origin of the name.

## Trade-offs
| Pros | Cons |
|---|---|
| Core stays small and stable; features isolated | Designing a good, stable contract is hard |
| Third parties extend without core changes | Contract versioning and compatibility forever |
| Per-installation feature sets | Plug-in faults can crash or slow the host |
| Plug-ins testable in isolation | Discovery and dynamic loading complicate debugging and packaging |

## Pitfalls
- **Contract too wide:** exposing core internals freezes them. Expose a narrow facade of host services.
- **No version check:** old plug-ins break silently. Declare an API version; reject or adapt incompatible ones.
- **Plug-in failure takes down the host:** isolate loading and calls (try/except, timeouts, separate processes for untrusted code).
- **Hidden ordering dependencies between plug-ins:** make ordering explicit (priorities, `before`/`after` like pluggy's `tryfirst`).
- **Security:** third-party plug-ins run with your privileges. Sandbox or review untrusted code.
- **Eager loading of everything:** slow startup. Load plug-in code lazily on first use.

## Refactoring path
- **Introduce it:** from a growing `if/elif` over variants, extract a Protocol from what branches have in common, move each branch into a class implementing it, register them in a dict (Strategy + registry), and only when third parties or separate releases are needed, add entry-point discovery and versioning.
- **Remove it / simplify:** if all plug-ins live in the repo and ship with the core, drop discovery and versioning; keep a plain dict of strategies.

## Related patterns
- [Service Locator](../architectural/service-locator.md) — plug-ins often look up host services; prefer passing a narrow host API.
- [Strategy](../behavioral/strategy.md) — the in-repo, non-discoverable version of a plug-in.
- [Extension Object](../structural/extension-object.md) — adds interfaces to objects at runtime; plug-ins add features to the system.
- [Component-Oriented Programming](../structural/component-oriented-programming.md) — contract-defined deployable components.
- [Hexagonal](../architectural/hexagonal.md) — adapters are plug-ins for ports.
- [Blackboard](../architectural/blackboard.md) — knowledge sources are commonly loaded as plug-ins.
