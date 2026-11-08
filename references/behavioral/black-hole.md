# Black Hole

**Category:** Behavioral · **Also known as:** No-op sink, Null sink, `/dev/null` object · **Related:** Null Object, Dependency Injection, Decorator, Test Stub, Observer

## Intent
Provide a sink that accepts any input through its interface and silently discards it, so producers can always emit (logs, metrics, traces, events) without checking whether anybody is listening.

## Problem
- Cross-cutting outputs (logging, metrics, tracing, audit, progress reporting) are optional depending on environment, consent or configuration.
- Producers get littered with `if self.metrics: self.metrics.increment(...)`, or library code crashes because the host app never configured a handler.
- Tests and CLI tools want the side channel off without patching globals.
- Smells: `Optional[MetricsClient]` everywhere; feature flags checked at every emit site; libraries printing "No handlers could be found for logger X" (old Python 2 warning).

Scenario: a payment library emits `payments.charge.latency_ms` and audit events. Some host apps run Prometheus, some run StatsD, many run nothing. The library must work identically in all cases.

## Solution and structure
A Black Hole is a Null Object restricted to *write-only* interfaces: all methods return `None`/`void` (or the sink itself, for chaining), so "do nothing" is always a correct answer.

```
Producer ──emit──> Sink (interface)
                     ├── PrometheusSink   (real)
                     ├── StatsdSink       (real)
                     └── BlackHoleSink    (accept, discard)
```

Difference from Null Object: a null object returns neutral *values* that influence the caller (identity discount, empty list). A black hole returns nothing the caller uses; it is purely a terminal sink. That makes it nearly always safe, which is the point.

A variant is a **counting black hole** for tests: discards but records counts, which is really a Test Spy.

## When to use
- Optional observability: logger, metrics, tracer, audit trail, progress callback, event publisher in tests.
- Libraries that must not emit anything unless the host opts in.
- Benchmarks and load tests that need to remove output cost.
- Default implementations in DI containers for write-only ports.

## When NOT to use
- Anything whose output is required for correctness or compliance (audit logs in a regulated system, outbox events): a silent sink there is data loss. Fail loudly at startup if it is not configured.
- Interfaces with return values: use Null Object and pick neutral values deliberately.
- When you want to *verify* emissions in tests: use a Test Spy or in-memory fake.
- When the language/framework already gives you one (below): don't write your own.

## Example — Python
Most of the time the stdlib already has the black hole. Write one only for your own ports.

```python
import logging
from typing import Protocol

# 1. Library logging: the stdlib black hole. Host apps configure real handlers.
logging.getLogger("payments").addHandler(logging.NullHandler())

# 2. Your own write-only port.
class Metrics(Protocol):
    def increment(self, name: str, value: int = 1, **tags: str) -> None: ...
    def timing(self, name: str, ms: float, **tags: str) -> None: ...

class BlackHoleMetrics:
    __slots__ = ()
    def increment(self, name: str, value: int = 1, **tags: str) -> None:
        pass
    def timing(self, name: str, ms: float, **tags: str) -> None:
        pass

BLACK_HOLE_METRICS: Metrics = BlackHoleMetrics()

class PaymentClient:
    def __init__(self, http: "HttpClient", metrics: Metrics = BLACK_HOLE_METRICS) -> None:
        self._http = http
        self._metrics = metrics

    async def charge(self, order_id: str, cents: int) -> str:
        resp = await self._http.post("/charges", {"order_id": order_id, "amount": cents})
        self._metrics.increment("payments.charge", status=str(resp.status))  # no None check
        return resp.json()["id"]

class HttpClient(Protocol):
    async def post(self, path: str, body: dict) -> "Response": ...
class Response(Protocol):
    status: int
    def json(self) -> dict: ...
```

Other Python black holes: `open(os.devnull, "w")`, `contextlib.redirect_stdout(io.StringIO())`, `collections.deque(iterable, maxlen=0)` to consume an iterator and discard its items.

## Example — TypeScript
In Angular, the black hole is the default value of an `InjectionToken` for a write-only port.

```typescript
import { Injectable, InjectionToken, inject } from '@angular/core';
import { Writable } from 'node:stream';

export interface AuditSink {
  record(action: string, details: Record<string, unknown>): void;
}

const BLACK_HOLE_AUDIT: AuditSink = { record: () => {} };

export const AUDIT_SINK = new InjectionToken<AuditSink>('AUDIT_SINK', {
  providedIn: 'root',
  factory: () => BLACK_HOLE_AUDIT,
});

@Injectable({ providedIn: 'root' })
export class AdminUsersService {
  private readonly audit = inject(AUDIT_SINK);

  deactivate(userId: string): void {
    // ... call API ...
    this.audit.record('user.deactivated', { userId });
  }
}

// Node: a stream black hole (in a server-side module)
export const devNull = new Writable({ write(_chunk, _enc, cb) { cb(); } });
```

## In the wild
- Python: `logging.NullHandler`, `os.devnull`, `subprocess.DEVNULL`, OpenTelemetry API no-op tracer/meter (used automatically when no SDK is configured), `structlog` can drop events via a processor raising `DropEvent`.
- JS/Node: `stream.Writable` sinks, `/dev/null`, `pino` with `enabled: false`, RxJS `noop` as an `Observer.next`.
- Angular: `provideNoopAnimations()`; tests often provide no-op `ErrorHandler` or analytics.
- Unix `/dev/null` itself is the archetype.

## Trade-offs
| Pros | Cons |
|---|---|
| Producers emit unconditionally; simpler code | Silent: misconfiguration looks like "no traffic" |
| Almost always safe (no return values) | Easy to ship to prod with observability accidentally off |
| Zero-dependency default for libraries | Work to build the payload still happens unless guarded |
| Trivial to implement and test | Doesn't help tests that need to assert emissions |

## Pitfalls
- **Black hole in production by accident.** Missing env var → metrics silently off for months. Fix: log a startup warning when the black hole is bound outside dev/test, or require explicit opt-out.
- **Expensive payload construction still runs** (`json.dumps(big)` before `debug()`). Fix: lazy arguments (`log.debug("x %s", obj)`), `isEnabledFor`, or an `enabled` property on the port.
- **Using it for a port that returns data.** Fix: that's a Null Object; choose neutral return values consciously.
- **Black-holing errors.** A sink for exceptions (`except: pass`) is not this pattern; it is swallowing bugs. Fix: never black-hole error channels in production.
- **Interface drift.** Fix: type the instance as the Protocol/interface so new methods fail type-checking.

## Refactoring path
- **Introduce it:** from `Optional[Sink]` with guards at every emit. (1) Define the write-only Protocol. (2) Add the black hole class/constant. (3) Make it the default in the constructor or DI token. (4) Remove guards. (5) Narrow the type to non-optional.
- **Remove it / simplify:** if the output turned out to be mandatory, delete the default so construction fails without a real sink. If only one implementation is ever used, inject it directly and drop the abstraction.

## Related patterns
- [Null Object](null-object.md) — the general pattern; Black Hole is its write-only, always-safe specialization.
- [Dependency Injection](dependency-injection.md) — the black hole is the default binding; real sinks override it.
- [Test Spy](../testing/test-spy.md) — records instead of discards; use when tests must assert emissions.
- [Decorator](../structural/decorator.md) — sampling/filtering sinks wrap a real one; a black hole is the terminal case.
- [Observer](observer.md) — a black hole is a valid subscriber that ignores notifications.
