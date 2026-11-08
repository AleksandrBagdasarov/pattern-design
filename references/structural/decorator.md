# Decorator

**Category:** Structural · **Also known as:** Wrapper · **Related:** Proxy, Adapter, Chain of Responsibility, Composite, Strategy, Intercepting Validator

## Intent
Attach additional responsibilities to an object dynamically by wrapping it in another object with the same interface; wrappers can be stacked in any order, giving a flexible alternative to subclassing.

## Problem
- Cross-cutting behavior (logging, metrics, retry, caching, auth, compression) must be added to some instances but not others.
- Subclassing explodes: `RetryingHttpClient`, `CachingHttpClient`, `RetryingCachingHttpClient`, `LoggingRetryingCachingHttpClient`…
- The core class is third-party or must stay focused on one job (Single Responsibility).
- The combination of behaviors must be chosen at runtime or in configuration.

Scenario: an exchange-rate client must be cached in production, retried on network errors, timed in staging, and bare in unit tests.

## Solution and structure
- **Component**: the shared interface (`RatesClient.get_rate(base, quote)`).
- **Concrete Component**: the real implementation.
- **Decorator**: implements Component, holds a Component, forwards calls and adds behavior before/after.

```
client = Timing( Retrying( Caching( HttpRatesClient ) ) )
call ─> Timing ─> Retrying ─> Caching ─> HttpRatesClient
```
Order matters: here a cache hit skips retry and HTTP; timing measures everything including retries.

## Three things called "decorator"
- **GoF Decorator** (this page): an *object* wraps another object at runtime, same interface.
- **Python `@decorator`**: syntax sugar for `f = deco(f)`, applied at definition time to a function or class. A function decorator that wraps calls (`@functools.cache`, `@tenacity.retry`) *is* the GoF pattern applied to a callable. Python's language dissolves the class boilerplate: a callable is the Component.
- **TypeScript/Angular decorators** (`@Component`, `@Injectable`, `@Input`): metadata annotations used by the framework/compiler. They mostly attach configuration, not runtime wrapping, so they are closer to Metadata Mapping than to GoF Decorator. TS 5 implements TC39 standard decorators, while Angular's own decorators are processed by the Angular compiler (and its tsconfig has historically used `experimentalDecorators`); this area has been changing, so check angular.dev before writing custom ones.

## When to use
- Optional, combinable behaviors around a stable interface.
- You cannot or should not modify the wrapped class.
- Behavior must be configured per environment or per instance (DI wiring decides the stack).
- Pipelines of transformations over a stream or request (middleware, I/O streams).

## When NOT to use
- One behavior, applied everywhere, in code you own: just put it in the class or function.
- The wrapper changes the interface: that is Adapter.
- The interface is wide (20+ methods): hand-written decorators must forward all of them; consider a dynamic proxy, a smaller interface, or framework hooks (middleware/interceptors).
- Behavior depends on the concrete type of the wrapped object: decorators should be type-agnostic.

## Example — Python
Idiomatic Python: decorate the callable, preserve metadata with `functools.wraps`, keep the signature with `ParamSpec`.

```python
import asyncio
import functools
import time
from collections.abc import Awaitable, Callable
from decimal import Decimal

import httpx


def retry[**P, R](attempts: int = 3, delay: float = 0.2,
                   on: tuple[type[Exception], ...] = (httpx.TransportError,)):
    def deco(fn: Callable[P, Awaitable[R]]) -> Callable[P, Awaitable[R]]:
        @functools.wraps(fn)
        async def wrapper(*args: P.args, **kwargs: P.kwargs) -> R:
            for i in range(attempts):
                try:
                    return await fn(*args, **kwargs)
                except on:
                    if i == attempts - 1:
                        raise
                    await asyncio.sleep(delay * 2**i)
            raise AssertionError("unreachable")
        return wrapper
    return deco


def timed[**P, R](fn: Callable[P, Awaitable[R]]) -> Callable[P, Awaitable[R]]:
    @functools.wraps(fn)
    async def wrapper(*args: P.args, **kwargs: P.kwargs) -> R:
        t = time.perf_counter()
        try:
            return await fn(*args, **kwargs)
        finally:
            print(f"{fn.__qualname__} took {time.perf_counter() - t:.3f}s")
    return wrapper


@timed
@retry(attempts=3)
async def get_rate(client: httpx.AsyncClient, base: str, quote: str) -> Decimal:
    r = await client.get(f"/rates/{base}/{quote}")
    r.raise_for_status()
    return Decimal(r.json()["rate"])
```
Decorators apply bottom-up: `get_rate = timed(retry(attempts=3)(get_rate))`. Do not stack `@functools.cache` on an `async def`: it caches the coroutine object, which can only be awaited once.

## Example — TypeScript
Object form with DI: Angular decides the stack, callers only see `RatesClient`.

```ts
import { Injectable, InjectionToken, inject } from '@angular/core';
import { HttpClient } from '@angular/common/http';
import { Observable, map, retry, shareReplay, timer } from 'rxjs';

export interface RatesClient { rate(base: string, quote: string): Observable<number> }
export const RATES = new InjectionToken<RatesClient>('RATES');

@Injectable({ providedIn: 'root' })
export class HttpRatesClient implements RatesClient {
  private readonly http = inject(HttpClient);
  rate(b: string, q: string) {
    return this.http.get<{ rate: string }>(`/api/rates/${b}/${q}`).pipe(map((r) => Number(r.rate)));
  }
}

export class RetryingRates implements RatesClient {
  constructor(private readonly inner: RatesClient) {}
  rate(b: string, q: string) {
    return this.inner.rate(b, q).pipe(retry({ count: 3, delay: (_, i) => timer(200 * 2 ** i) }));
  }
}

export class CachingRates implements RatesClient {
  private readonly cache = new Map<string, Observable<number>>();
  constructor(private readonly inner: RatesClient) {}
  rate(b: string, q: string) {
    const key = `${b}/${q}`;
    if (!this.cache.has(key)) this.cache.set(key, this.inner.rate(b, q).pipe(shareReplay(1)));
    return this.cache.get(key)!;
  }
}

// app.config.ts
export const ratesProvider = {
  provide: RATES,
  useFactory: () => new CachingRates(new RetryingRates(inject(HttpRatesClient))),
};
```

## In the wild
- Python: `functools.cache`/`lru_cache`, `functools.wraps`, `contextlib.contextmanager`, `tenacity.retry`, `io.BufferedReader` over raw streams, `gzip.GzipFile` wrapping a file object.
- FastAPI/Starlette middleware and `APIRouter(dependencies=[...])`; pytest fixtures wrapping setup/teardown.
- Angular `HttpInterceptorFn` chain; RxJS operators (`retry`, `timeout`, `shareReplay`) decorate an Observable.
- Node streams `pipe()` (gzip, cipher); Express/Koa middleware; NestJS interceptors.
- AWS Lambda Powertools decorators (`@logger.inject_lambda_context`, `@tracer.capture_method`).

## Trade-offs
| Pros | Cons |
|---|---|
| Combine behaviors at runtime without subclass explosion | Many small objects; debugging means walking the wrapper stack |
| Each decorator has one responsibility and is testable alone | Order dependence is implicit and easy to get wrong |
| Works on classes you cannot modify | Wide interfaces require lots of forwarding boilerplate |
| Behavior chosen per environment via DI | Identity/type checks see the decorator, not the wrapped object |

## Pitfalls
- **Losing metadata** in Python (`__name__`, docstring, signature). Fix: `functools.wraps`; use `ParamSpec` for typing.
- **Caching coroutines or Observables wrongly**: caching a cold Observable without `shareReplay` re-runs the request. Fix: cache the *result* (or a shared, replaying stream).
- **Wrong order**: retry outside a circuit breaker, cache outside auth checks (serving one user's data to another). Fix: define the stack in one place and test it.
- **Decorator relies on internals** of the concrete component. Fix: depend only on the interface.
- **Partial forwarding**: new interface methods are not forwarded. Fix: keep interfaces narrow; add a test that every interface method passes through.

## Refactoring path
- **Introduce it:** find cross-cutting code duplicated in many methods (try/except retry loops, timing, cache lookups). Extract an interface if none exists; move the concern into one wrapper; wire it in DI; delete the duplicated code from the core.
- **Remove it / simplify:** if a decorator is always applied and never configured, inline it into the component; collapse several one-line decorators into a single middleware; replace hand-written wrappers with a library (`tenacity`, `cachetools`, RxJS operators).

## Related patterns
- [Proxy](../structural/proxy.md) — same shape, but controls access rather than adding behavior; usually one layer, chosen by the system.
- [Adapter](../structural/adapter.md) — changes the interface; Decorator preserves it.
- [Chain of Responsibility](../behavioral/chain-of-responsibility.md) — handlers may stop the chain; decorators always forward.
- [Composite](../structural/composite.md) — also recursive composition, but aggregates many children instead of wrapping one.
- [Strategy](../behavioral/strategy.md) — changes the guts; Decorator changes the skin.
- [Intercepting Validator](../security/intercepting-validator.md) — a decorator/middleware specialised for input validation.
- [Metadata Mapping](../emergent/metadata-mapping.md) — what TS/Angular decorators mostly are.
