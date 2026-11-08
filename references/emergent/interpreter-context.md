# Interpreter Context

**Category:** Emergent / modern · **Also known as:** Context object, ambient context, environment, request context, continuation-local storage · **Related:** Interpreter, Dependency Injection, Service Locator, Chain of Responsibility, Singleton

## Intent
Carry state that belongs to the current evaluation (an expression's variable bindings, a request's user and trace id, a transaction, the active injector) implicitly through a flow of execution, scoped to that flow, without threading it as a parameter through every function and without leaking it to concurrent flows.

## Problem
- Data needed deep in the call stack (request id for logs, tenant id for queries, current user for audit) but irrelevant to the functions in between.
- Scenario: a FastAPI service must include `request_id` and `tenant_id` in every log line and DB query. Passing them as parameters changes 60 function signatures; storing them in a module global breaks immediately under asyncio, because 200 concurrent requests overwrite each other.
- Scenario: a rules engine evaluates `price * qty > limit` and needs variable bindings and a function table during evaluation.
- Smells: "tramp data" parameters passed through 5 layers unused; thread-locals in async code; globals reset in `finally` blocks.

## Solution and structure
- **Context**: a small object or a set of variables holding evaluation-scoped state.
- **Scope**: established at an entry point (request middleware, task start, `interpret(expr, env)`) and torn down when it ends.
- **Propagation**: the runtime copies or inherits the context into child work (asyncio tasks copy the current `contextvars.Context`; Node's `AsyncLocalStorage` follows async resources; Angular's injection context exists during construction/factory execution).
- **Access**: read-only lookups near the leaves (`request_id.get()`, `inject(Token)`).

```
 middleware: token = ctx.set(req_ctx) ─► handler ─► service ─► repo/logger reads ctx.get()
             finally: ctx.reset(token)   (each concurrent request sees its own value)
```
In the classic Interpreter pattern (GoF), the Context is the explicit environment passed into `evaluate(context)` — that explicit form is still best when you control the evaluator.

## When to use
- Cross-cutting, request-scoped data: correlation/trace ids, tenant, locale, auth principal, deadline/cancellation.
- Interpreters, template engines, rule engines, query builders that need bindings during evaluation.
- Framework hooks: transactions or sessions bound to the current unit of work.

## When NOT to use
- Business inputs a function needs to do its job (order, amount, user being edited): pass them explicitly. Hidden inputs make functions untestable and surprising.
- As a general service locator for dependencies: use DI (`Depends`, `inject()` in constructors).
- Mutable shared state written by many layers: it becomes a hidden global. Contexts should be set once at the entry point and read below.
- Thread-local storage in async code (`threading.local` doesn't follow coroutines; all coroutines on the loop thread share it).

## Example — Python
`contextvars` is async-safe: each task gets a copy of the context at creation.

```python
import logging, uuid
from contextvars import ContextVar
from dataclasses import dataclass
from fastapi import FastAPI, Request

@dataclass(frozen=True, slots=True)
class RequestCtx:
    request_id: str
    tenant_id: str

_current: ContextVar[RequestCtx | None] = ContextVar("request_ctx", default=None)

def current() -> RequestCtx:
    ctx = _current.get()
    if ctx is None:
        raise RuntimeError("no request context (called outside a request?)")
    return ctx

class CtxFilter(logging.Filter):
    def filter(self, record: logging.LogRecord) -> bool:
        ctx = _current.get()
        record.request_id = ctx.request_id if ctx else "-"
        return True

app = FastAPI()

@app.middleware("http")
async def bind_context(request: Request, call_next):
    token = _current.set(RequestCtx(
        request_id=request.headers.get("x-request-id", str(uuid.uuid4())),
        tenant_id=request.headers["x-tenant-id"]))
    try:
        return await call_next(request)
    finally:
        _current.reset(token)          # restore; never leave it set for the next request

async def list_orders(db) -> list[dict]:          # deep in the service layer
    return await db.fetch("SELECT * FROM orders WHERE tenant_id = $1", current().tenant_id)
```
Note: `run_in_executor` does not copy context automatically; `asyncio.to_thread` does. For thread pools, wrap with `contextvars.copy_context().run`.

## Example — TypeScript
Node uses `AsyncLocalStorage`; Angular's injection context is the framework's own ambient context.

```ts
// Node / Express
import { AsyncLocalStorage } from 'node:async_hooks';
import { randomUUID } from 'node:crypto';
import express from 'express';

interface ReqCtx { requestId: string; tenantId: string }
const als = new AsyncLocalStorage<ReqCtx>();
export const ctx = (): ReqCtx => {
  const c = als.getStore();
  if (!c) throw new Error('no request context');
  return c;
};

const app = express();
app.use((req, _res, next) =>
  als.run({ requestId: req.header('x-request-id') ?? randomUUID(), tenantId: req.header('x-tenant-id')! }, next));
```

```ts
// Angular: inject() works only inside an injection context
import { DestroyRef, EnvironmentInjector, inject, runInInjectionContext } from '@angular/core';

export function autoLog(name: string) {                 // a reusable "inject function"
  inject(DestroyRef).onDestroy(() => console.log(`${name} destroyed`));
}

export class LateSetup {
  private env = inject(EnvironmentInjector);            // field initializer: inside context
  later(): void {
    // inject() here would throw NG0203; re-enter a context explicitly:
    runInInjectionContext(this.env, () => autoLog('late'));
  }
}
```

## In the wild
- Python: `contextvars` (asyncio, `decimal.localcontext`), Starlette/FastAPI request scope, structlog `contextvars` binding, OpenTelemetry context, SQLAlchemy `scoped_session` (thread/task scope), Flask `g`/`request` (context-local proxies).
- Node: `AsyncLocalStorage` (used by OpenTelemetry, NestJS CLS, Next.js `headers()`); TC39 `AsyncContext` proposal (check its stage).
- Angular: injection context (`inject()`, `runInInjectionContext`, `assertInInjectionContext`), `DestroyRef`; React context; Go `context.Context` (explicit-parameter style).

## Trade-offs
| Pros | Cons |
|---|---|
| Removes tramp parameters through many layers | Hidden inputs: signatures stop telling the truth |
| Concurrency-safe isolation per request/task | Breaks when propagation is lost (thread pools, callbacks) |
| Consistent logging/tracing with no code changes | Tests must set up context explicitly |
| Framework hooks become simple (`inject()`) | Easy to abuse as a global service locator |

## Pitfalls
- Using module globals or `threading.local` in asyncio — values bleed across concurrent requests.
- Forgetting `reset(token)` (or using `set` without scoping): context leaks into later work on reused tasks/threads.
- Mutating an object stored in a ContextVar: child tasks share the same object reference; store immutable values and `set` new ones.
- Calling Angular `inject()` in methods, `setTimeout` callbacks or after `await` — throws NG0203. Inject in constructors/field initializers, or capture an `Injector` and use `runInInjectionContext`.
- Losing context across raw thread pools, `EventEmitter` listeners registered elsewhere, or callback-based libraries; bind explicitly (`copy_context().run`, `AsyncResource.bind`).

## Refactoring path
- **Introduce it:** find tramp parameters (request id, tenant) that pass through without use; create a ContextVar/ALS with a typed accessor; set it in one middleware; replace the parameter at the leaves; delete it from intermediate signatures.
- **Remove it / simplify:** when a "context" value turns out to be business input, make it an explicit parameter again; when contexts multiply, consolidate into one immutable request-context object, or move dependencies to DI.

## Related patterns
- [Interpreter](../behavioral/interpreter.md) — the original home of the explicit evaluation context.
- [Dependency Injection](../behavioral/dependency-injection.md) — for dependencies; context is for evaluation-scoped data.
- [Service Locator](../architectural/service-locator.md) — what an abused ambient context degrades into.
- [Chain of Responsibility](../behavioral/chain-of-responsibility.md) — middleware chains typically establish the context.
- [Singleton](../creational/singleton.md) — process-wide; context vars are the per-flow alternative.
