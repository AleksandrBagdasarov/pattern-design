# Dynamic Proxy

**Category:** Structural · **Also known as:** Runtime proxy, Interceptor, Invocation handler · **Related:** Proxy, Decorator, Secure Proxy, Mock Object, Metadata Mapping

## Intent
Create proxy objects at runtime that intercept every attribute access or method call on any target, routing them through one generic handler, so cross-cutting behavior is written once instead of in a hand-coded proxy per interface.

## Problem
- You would need a hand-written proxy for each of 30 service interfaces just to add tracing or auditing, and they must be updated every time a method is added.
- The set of methods is not known at compile time (RPC stubs generated from a schema, configuration objects, ORM attributes).
- You need to observe reads and writes on plain objects (reactivity, change tracking) without asking users to call `set_x()`.
- Test doubles must accept any call and record it.

Scenario: every repository method should emit an OpenTelemetry span named `Repo.method`. A dynamic proxy wraps any repository instance and covers all methods, including ones added next month.

## Solution and structure
- **Target**: the real object (optional; some proxies have none, like RPC stubs or mocks).
- **Handler**: a single function/trap invoked for every access: `(target, name, args) -> result`.
- **Proxy**: a runtime-created object that routes accesses to the handler.

Language mechanisms:
- Python: `__getattr__` (called only when normal lookup fails), `__getattribute__` (every access; easy to recurse infinitely), `__setattr__`, `__call__`; or generate classes with `type()`.
- JavaScript/TypeScript: the built-in `Proxy` with traps (`get`, `set`, `apply`, `has`, `deleteProperty`…) and `Reflect` for default behavior.
- Java: `java.lang.reflect.Proxy` (interfaces only), ByteBuddy/cglib for classes — mentioned only for vocabulary.

## When to use
- Cross-cutting concerns (tracing, metrics, auditing, permission checks) uniformly across many interfaces.
- Clients for APIs described by data (OpenAPI, JSON-RPC, gRPC reflection) where methods map 1:1 to remote operations.
- Reactivity/change tracking on plain objects (Vue 3, MobX, Immer drafts).
- Test doubles and record/replay tools.

## When NOT to use
- One or two interfaces: a hand-written Proxy/Decorator is clearer and type-checked.
- Behavior that differs per method: a big `if name == ...` in the handler is a worse class.
- Hot loops: interception adds overhead to every access.
- When static typing matters at call sites: dynamic proxies erase types unless you cast/annotate carefully (`typing.cast`, TS generics).
- In Python, if a decorator on the class methods or a metaclass/`__init_subclass__` hook is enough, prefer that explicit form.

## Example — Python
A generic tracing proxy that keeps the static type of the wrapped object for callers.

```python
import functools
import inspect
import time
from typing import Any, cast


class Traced:
    """Wrap any object; time every public method call (sync or async)."""

    def __init__(self, target: object, sink=print) -> None:
        object.__setattr__(self, "_target", target)   # bypass our own __setattr__ rules
        object.__setattr__(self, "_sink", sink)

    def __getattr__(self, name: str) -> Any:          # only called for missing attrs
        attr = getattr(self._target, name)
        if name.startswith("_") or not callable(attr):
            return attr
        label = f"{type(self._target).__name__}.{name}"

        if inspect.iscoroutinefunction(attr):
            @functools.wraps(attr)
            async def async_wrapper(*a: Any, **kw: Any) -> Any:
                t = time.perf_counter()
                try:
                    return await attr(*a, **kw)
                finally:
                    self._sink(f"{label} {time.perf_counter() - t:.4f}s")
            return async_wrapper

        @functools.wraps(attr)
        def wrapper(*a: Any, **kw: Any) -> Any:
            t = time.perf_counter()
            try:
                return attr(*a, **kw)
            finally:
                self._sink(f"{label} {time.perf_counter() - t:.4f}s")
        return wrapper


def traced[T](target: T) -> T:
    return cast(T, Traced(target))   # callers keep full type hints


class OrderRepository:
    def get(self, order_id: int) -> dict[str, Any]:
        return {"id": order_id, "status": "paid"}


repo = traced(OrderRepository())
repo.get(42)          # prints "OrderRepository.get 0.0000s"
```
`isinstance(repo, OrderRepository)` is now `False`; that is the price of the generic form.

## Example — TypeScript
A JS `Proxy` that records changes to a form model (change tracking without setters).

```ts
type Patch = { path: string; from: unknown; to: unknown };

export function tracked<T extends object>(target: T, patches: Patch[] = [], base = ''): T {
  return new Proxy(target, {
    get(obj, key, receiver) {
      const value = Reflect.get(obj, key, receiver);
      return typeof value === 'object' && value !== null
        ? tracked(value as object, patches, `${base}${String(key)}.`)
        : value;
    },
    set(obj, key, value, receiver) {
      const from = Reflect.get(obj, key, receiver);
      if (from !== value) patches.push({ path: base + String(key), from, to: value });
      return Reflect.set(obj, key, value, receiver);
    },
  });
}

interface Profile { name: string; address: { city: string; zip: string } }
const patches: Patch[] = [];
const profile = tracked<Profile>({ name: 'Ann', address: { city: 'Kyiv', zip: '01001' } }, patches);
profile.address.city = 'Lviv';
// patches -> [{ path: 'address.city', from: 'Kyiv', to: 'Lviv' }]
// PATCH /api/profile with only the changed fields.
```
Note: nested objects get a new Proxy on every `get`, so `profile.address !== profile.address`. Cache proxies in a `WeakMap` if identity matters. Angular signals do not use Proxies; mutating a proxied object does not notify a signal.

## In the wild
- Python: `unittest.mock.Mock`/`MagicMock`, `xmlrpc.client.ServerProxy`, `multiprocessing` manager proxies, `wrapt.ObjectProxy` (a robust base for transparent proxies), SQLAlchemy instrumented attributes.
- JS: Vue 3 `reactive()`, MobX observables, Immer `produce()` drafts, Valtio; Jest/Vitest `vi.fn()` and module mocks.
- Java/Spring AOP, `@Transactional`, Hibernate lazy entities — the canonical server-side users.
- OpenTelemetry auto-instrumentation libraries patch or wrap clients dynamically.

## Trade-offs
| Pros | Cons |
|---|---|
| One handler covers every method, now and future | Loses static typing and IDE navigation unless cast carefully |
| Removes boilerplate forwarding code | "Magic": behavior not visible where methods are defined |
| Enables reactivity, mocks, RPC stubs | Runtime overhead on every access |
| Easy to apply/remove at wiring time | Breaks identity, `isinstance`, `instanceof`, private fields (`#x` in JS) |

## Pitfalls
- **Infinite recursion** with `__getattribute__`/`__setattr__` touching `self.x`. Fix: use `object.__getattribute__`/`object.__setattr__`, or prefer `__getattr__`.
- **Dunder methods bypass `__getattr__`**: `len(proxy)`, `proxy[0]`, `with proxy:` look up on the type, not the instance. Fix: define those explicitly or use `wrapt.ObjectProxy`.
- **JS private fields and built-ins**: `Map`, `Date`, class `#private` fields throw through a Proxy because the receiver is wrong. Fix: bind methods to the target in the `get` trap or avoid proxying such objects.
- **Async functions wrapped synchronously** (timing measures coroutine creation, not execution). Fix: detect coroutine functions as shown.
- **Silent typos**: mocks/proxies accept any attribute name. Fix: `Mock(spec=Cls)` / `create_autospec`; typed wrappers.

## Refactoring path
- **Introduce it:** when the third hand-written proxy repeats the same before/after code, extract the shared handler; apply it at the composition root (DI) behind a typed factory like `traced[T]`.
- **Remove it / simplify:** if only a few methods need the behavior, replace the dynamic proxy with explicit decorators on those methods; for reactivity in Angular, use signals instead of proxied objects.

## Related patterns
- [Proxy](../structural/proxy.md) — the static, hand-written form.
- [Decorator](../structural/decorator.md) — explicit, typed, stackable alternative.
- [Secure Proxy](../security/secure-proxy.md) — authorization via (often dynamic) proxies.
- [Mock Object](../testing/mock-object.md) — the most common dynamic proxy in daily work.
- [Metadata Mapping](../emergent/metadata-mapping.md) — handlers often read metadata to decide what to do.
