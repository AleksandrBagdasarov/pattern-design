# Chain of Responsibility

**Category:** Behavioral · **Also known as:** Middleware, Pipeline, Interceptor chain, Handler chain · **Related:** Decorator, Command, Pipes and Filters, Intercepting Validator, Template Method, Strategy

## Intent
Pass a request along an ordered chain of handlers. Each handler either handles it, transforms it and passes it on, or stops the chain, so the sender does not need to know which handler does the work.

## Problem
- One request goes through many cross-cutting checks (auth, rate limit, validation, logging, caching) and they are hard-coded inside every endpoint.
- A big `if/elif` decides who handles a request, and new cases are added by editing it: "if amount < 100 auto-approve, elif < 5000 team lead, else CFO".
- The order of steps matters and must be configurable per route or tenant.
- Scenario: every outgoing HTTP call to a partner API must add an auth header, retry on 503, log timing and refresh the token on 401. Each service client re-implements this slightly differently.

## Solution and structure
- **Handler** has one method, `handle(request, next)`. It may act before calling `next`, after it, both, or not call it at all (short-circuit).
- The **chain** is an ordered list of handlers, built once in configuration.
- The **client** sends the request to the first handler.

```
request ─▶ [auth] ─▶ [rate limit] ─▶ [log] ─▶ [endpoint]
response ◀──────────────────────────────────────┘ (passes back through in reverse)
```

Two variants:
- **Classic GoF**: the first handler that can handle the request handles it and stops (approval levels, event bubbling, exception handler lookup).
- **Middleware / interceptor**: every handler wraps the next one, like an onion (web frameworks, HTTP clients). This is the common modern form and is structurally a chain of Decorators.

In Python and TS a handler is just a function taking `(request, next)`, and the chain is built with a fold.

## When to use
- Cross-cutting concerns around a core operation: auth, logging, metrics, tracing, retries, caching, compression, CORS.
- Several candidate handlers with a priority order, and you want to add or reorder them without touching the sender.
- The pipeline must be configured per route, tenant or environment.

## When NOT to use
- Exactly one handler always handles the request. Call it.
- Steps are independent data transforms with no short-circuit. A list of functions (Pipes and Filters) is simpler.
- The decision is a plain lookup by key. Use a dict of strategies, not a chain that asks each handler "is this yours?".
- Debuggability matters more than flexibility and there are only two concerns. Inline them, or use one decorator.

## Example — Python
An HTTP client middleware chain, built from plain async functions.

```python
from collections.abc import Awaitable, Callable
from functools import reduce
import asyncio, logging, time
import httpx

type Next = Callable[[httpx.Request], Awaitable[httpx.Response]]
type Middleware = Callable[[httpx.Request, Next], Awaitable[httpx.Response]]

def bearer(token: Callable[[], str]) -> Middleware:
    async def mw(req: httpx.Request, nxt: Next) -> httpx.Response:
        req.headers["Authorization"] = f"Bearer {token()}"
        return await nxt(req)
    return mw

async def timing(req: httpx.Request, nxt: Next) -> httpx.Response:
    start = time.perf_counter()
    try:
        return await nxt(req)
    finally:
        logging.info("%s %s %.0fms", req.method, req.url, (time.perf_counter() - start) * 1000)

async def retry_503(req: httpx.Request, nxt: Next) -> httpx.Response:
    for attempt in range(3):
        resp = await nxt(req)
        if resp.status_code != 503:
            return resp
        await asyncio.sleep(0.2 * 2 ** attempt)
    return resp

def build_chain(middlewares: list[Middleware], terminal: Next) -> Next:
    # first in list = outermost
    return reduce(lambda nxt, mw: (lambda r, mw=mw, nxt=nxt: mw(r, nxt)), reversed(middlewares), terminal)

async def call_partner(client: httpx.AsyncClient) -> httpx.Response:
    send = build_chain([timing, bearer(lambda: "tkn"), retry_503], client.send)
    return await send(client.build_request("GET", "https://partner.example.com/v1/rates"))
```

In practice, httpx already offers `event_hooks` and custom transports; FastAPI/Starlette middleware and `Depends` chains cover the server side.

## Example — TypeScript
Angular functional HTTP interceptors are exactly this pattern (Angular 15+; check angular.dev for current signatures).

```typescript
import { HttpErrorResponse, HttpInterceptorFn, provideHttpClient, withInterceptors } from '@angular/common/http';
import { inject } from '@angular/core';
import { catchError, finalize, retry, throwError, timer } from 'rxjs';
import { AuthService } from './auth.service';

export const authInterceptor: HttpInterceptorFn = (req, next) => {
  const token = inject(AuthService).token();
  return next(token ? req.clone({ setHeaders: { Authorization: `Bearer ${token}` } }) : req);
};

export const timingInterceptor: HttpInterceptorFn = (req, next) => {
  const started = performance.now();
  return next(req).pipe(finalize(() => console.debug(req.url, performance.now() - started)));
};

export const retry503Interceptor: HttpInterceptorFn = (req, next) =>
  next(req).pipe(
    retry({ count: 2, delay: (err: HttpErrorResponse, i) => (err.status === 503 ? timer(200 * 2 ** i) : throwError(() => err)) }),
  );

export const unauthorizedInterceptor: HttpInterceptorFn = (req, next) => {
  const auth = inject(AuthService);                              // inject synchronously, not inside catchError
  return next(req).pipe(catchError((err: HttpErrorResponse) => {
    if (err.status === 401) auth.logout();
    return throwError(() => err);
  }));
};

// app.config.ts — order matters: first listed sees the request first and the response last
export const httpProviders = provideHttpClient(
  withInterceptors([timingInterceptor, authInterceptor, retry503Interceptor, unauthorizedInterceptor]),
);
```

`inject()` is only valid synchronously in the interceptor body (its injection context); calling it later inside an RxJS callback throws `NG0203`, so capture services first.

## In the wild
- ASGI/Starlette/FastAPI middleware, Django middleware, WSGI middleware stacks.
- Express/Koa/NestJS middleware, NestJS guards → interceptors → pipes, Angular `HttpInterceptorFn`, Angular router guards.
- `logging` (records propagate up the logger hierarchy), DOM event bubbling, Python exception handling up the call stack.
- `httpx` transports/event hooks, `requests` adapters, Axios interceptors, AWS SDK v3 middleware stack.
- Servlet filters, gRPC interceptors, Celery task signals.

## Trade-offs
| Pros | Cons |
|---|---|
| Cross-cutting concerns live once, applied everywhere | Order dependencies are implicit and easy to break |
| Handlers can be added, removed and reordered by config | A request may fall off the end unhandled |
| Each handler is small and unit-testable | Stack traces and debugging span many layers |
| Sender is decoupled from who handles the request | Long chains add latency on every request |

## Pitfalls
- **Wrong order.** Retry outside auth means a refreshed token is never applied; logging inside retry logs each attempt. Fix: document the order next to the chain and test it.
- **Forgetting to call `next`** (request silently dropped) or calling it twice (duplicate side effects). Fix: tests per handler for both paths.
- **No terminal handler** in a GoF chain: unhandled requests return `None`. Fix: a final default handler that raises or returns a clear error.
- **Late-binding closures** in Python lambdas building the chain. Fix: bind loop variables as defaults (`mw=mw`) as above, or use `functools.partial`.
- **Retrying non-idempotent requests** in a retry handler. Fix: retry only safe methods or requests with idempotency keys.
- **Business logic in middleware.** Fix: keep middleware for cross-cutting concerns; domain rules belong in the service.

## Refactoring path
- **Introduce it:** from the same boilerplate (auth, logging, retry) at the top of many functions. (1) Extract one concern into a `(request, next)` function. (2) Build a chain with that handler plus the original code as the terminal. (3) Move the next concern. (4) Prefer the framework's built-in middleware hook over a custom chain.
- **Remove it / simplify:** if a chain has one handler, or every handler applies to exactly one endpoint, inline them into that endpoint or a single decorator.

## Related patterns
- [Decorator](../structural/decorator.md) — middleware chains are stacked decorators; a decorator always delegates, a GoF chain may stop.
- [Pipes and Filters](../architectural/pipes-and-filters.md) — data transformation stages without short-circuit or wrapping.
- [Intercepting Validator](../security/intercepting-validator.md) — a validation handler placed in the chain before the endpoint.
- [Command](command.md) — middleware around a command bus (transactions, auth, logging).
- [Retry](retry.md) — commonly implemented as one handler in the chain.
- [Strategy](strategy.md) — pick one handler by key instead of asking each in turn.
