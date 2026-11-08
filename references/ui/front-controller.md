# Front Controller

**Category:** UI · **Also known as:** Single entry point, dispatcher, router + middleware pipeline · **Related:** Page Controller, Application Controller, Chain of Responsibility, Single Access Point, Intercepting Validator

## Intent
Route every incoming request through one handler that performs common processing (auth, logging, locale, error handling) and then dispatches to the specific handler for that request.

## Problem
- Each page/endpoint repeats the same steps: authenticate, parse locale, open a DB session, log, map exceptions to error pages.
- A missed check in one handler becomes a security hole.
- URL structure is scattered across files, so there is no single place to see or change routing.
- Cross-cutting changes (add request-id tracing) require editing every handler.

Scenario: an API needs request ids, JWT auth, CORS, structured logging and uniform error JSON on every endpoint. A front controller (the ASGI app with its middleware stack and router) does these once; endpoints only contain endpoint logic.

## Solution and structure
```
request ─▶ Front Controller ─▶ [middleware/filters: trace → auth → locale → errors] ─▶ Dispatcher/Router
                                                                                         │ match route
                                                                     ┌───────────────────┼───────────────────┐
                                                                handler A           handler B            handler C
```
- **Front controller**: the single entry point (ASGI/WSGI app, Express app, Angular `Router`).
- **Filters / middleware**: chain of cross-cutting steps, each may short-circuit (401) or decorate the request.
- **Dispatcher**: maps URL/method to a handler (route table).
- **Handlers / commands**: Page Controllers for the specific work.

In practice you almost never write a front controller: every web framework and SPA router is one. The design work is choosing what goes into middleware versus individual handlers.

## When to use
- Any web app or API (you get it from the framework).
- Cross-cutting policies that must apply uniformly: authentication, CSRF, rate limiting, tracing, error mapping, i18n.
- Centralized route table needed for docs, permissions, or link generation.
- SPA navigation: one router that resolves URLs to components, guards and lazy chunks.

## When NOT to use
- Do not build your own on top of a framework that already has one (no custom `dispatch(path)` switch inside a FastAPI app).
- Endpoint-specific logic in global middleware (`if request.path == '/orders'`): that belongs in the handler or a route-scoped dependency/guard.
- Expensive middleware applied to every request when only a few routes need it: scope it to a router/route group.

## Example — Python
```python
import time, uuid, logging
from fastapi import FastAPI, Request, Depends, APIRouter
from fastapi.responses import JSONResponse
from app.auth import current_user          # dependency: validates JWT, raises 401
from app.errors import DomainError

log = logging.getLogger("api")
app = FastAPI()                             # the front controller

@app.middleware("http")
async def trace_and_time(request: Request, call_next):
    request_id = request.headers.get("x-request-id", str(uuid.uuid4()))
    start = time.perf_counter()
    response = await call_next(request)
    response.headers["x-request-id"] = request_id
    log.info("%s %s %s %.1fms", request.method, request.url.path,
             response.status_code, (time.perf_counter() - start) * 1000,
             extra={"request_id": request_id})
    return response

@app.exception_handler(DomainError)
async def domain_error(_: Request, exc: DomainError) -> JSONResponse:
    return JSONResponse({"error": exc.code, "detail": str(exc)}, status_code=exc.status)

# route-group policy: everything under /api/orders requires a user
orders = APIRouter(prefix="/api/orders", dependencies=[Depends(current_user)])

@orders.get("/{order_id}")
async def get_order(order_id: int) -> dict:  # page controller: only endpoint logic
    ...

app.include_router(orders)
```
Middleware for truly global concerns; `dependencies=[...]` on a router for group policies; per-endpoint `Depends` for the rest.

## Example — TypeScript
In Angular, the `Router` is the client-side front controller; HTTP interceptors are the equivalent filter chain for outgoing requests.
```ts
import { ApplicationConfig, inject } from '@angular/core';
import { provideRouter, Routes, CanMatchFn, Router, withComponentInputBinding } from '@angular/router';
import { provideHttpClient, withInterceptors, HttpInterceptorFn } from '@angular/common/http';
import { AuthService } from './auth.service';

const isLoggedIn: CanMatchFn = () =>
  inject(AuthService).isLoggedIn() || inject(Router).parseUrl('/login');

const authHeader: HttpInterceptorFn = (req, next) => {
  const token = inject(AuthService).token();
  return next(token ? req.clone({ setHeaders: { Authorization: `Bearer ${token}` } }) : req);
};

const routes: Routes = [
  { path: 'login', loadComponent: () => import('./login/login').then((m) => m.Login) },
  {
    path: 'orders',
    canMatch: [isLoggedIn],                                  // group policy, like router dependencies
    loadChildren: () => import('./orders/orders.routes').then((m) => m.ORDER_ROUTES),
  },
  { path: '', pathMatch: 'full', redirectTo: 'orders' },
  { path: '**', loadComponent: () => import('./not-found').then((m) => m.NotFound) },
];

export const appConfig: ApplicationConfig = {
  providers: [
    provideRouter(routes, withComponentInputBinding()),
    provideHttpClient(withInterceptors([authHeader])),
  ],
};
```
Node/Express: `app.use(cors()); app.use(requestId); app.use('/api/orders', requireAuth, ordersRouter); app.use(errorHandler)`.

## In the wild
- ASGI/WSGI apps (FastAPI/Starlette, Django `urls.py` + middleware, Flask), Express/Fastify/NestJS (middleware, guards, interceptors, exception filters).
- Angular `Router` + guards + resolvers, `HttpInterceptorFn`; React Router; Vue Router.
- Edge: API Gateway, nginx, AWS ALB listener rules, Envoy — infrastructure-level front controllers.
- Spring `DispatcherServlet`, ASP.NET Core middleware pipeline.

## Trade-offs
| Pros | Cons |
|---|---|
| Cross-cutting policy implemented once, applied uniformly | Middleware order bugs are subtle (auth before CORS preflight, etc.) |
| One place to see all routes | Global middleware cost paid on every request |
| Handlers stay small and focused | Hidden behavior: reading a handler doesn't show what ran before it |
| Easy to add tracing/metrics globally | Single point of failure if the entry pipeline crashes |

## Pitfalls
- **Wrong middleware order:** e.g. error handler registered inside the auth layer so auth errors are not formatted; CORS after auth so preflight gets 401. Document and test the order.
- **Path-matching inside middleware** instead of router-scoped dependencies/guards: fragile and easy to bypass with a new route.
- **Blocking work in async middleware** (sync DB calls, `time.sleep`): stalls every request.
- **Reading the request body in middleware** in Starlette can consume it; prefer dependencies for body-related checks.
- **Angular guards using `canActivate` on lazy routes** still downloads the chunk; use `canMatch` to block both loading and activation.

## Refactoring path
- **Introduce it:** find checks copied into many handlers; move global ones into middleware/interceptors, group ones into router dependencies/guards; delete the copies; add a test per policy.
- **Remove it / simplify:** if middleware has grown `if path...` branches, move that logic to route-scoped dependencies or the handlers themselves.

## Related patterns
- [Page Controller](../ui/page-controller.md) — the handlers the front controller dispatches to.
- [Application Controller](../ui/application-controller.md) — decides flow between screens after dispatch.
- [Chain of Responsibility](../behavioral/chain-of-responsibility.md) — the middleware pipeline structure.
- [Single Access Point](../security/single-access-point.md) — the security view of the same single entry.
- [Intercepting Validator](../security/intercepting-validator.md) — validation as a front-controller filter.
