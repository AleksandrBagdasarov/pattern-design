---
name: pattern-design
description: Catalog of software design patterns grouped by topic (creational, structural, behavioral, dependency injection, architectural, data access, security, concurrency, testing, UI, enterprise integration, cloud/distributed, optimization, emergent) with use cases, trade-offs and when NOT to use each, plus one detailed reference file per pattern with Python and TypeScript examples. Use this skill whenever planning or researching an implementation, designing a system, module, service, API or feature from scratch, optimizing performance or scalability, refactoring, restructuring or decoupling existing code, choosing between architectural approaches, or reviewing a design. Also use when the user names any pattern (Singleton, Factory, Strategy, Observer, Repository, Unit of Work, CQRS, Saga, Circuit Breaker, Hexagonal, Clean Architecture, MVVM, DI, Event Sourcing, Producer-Consumer, Mock vs Stub, etc.), asks "how should I structure this?", "what's the best way to design…", or mentions coupling, extensibility, god classes, spaghetti code or scaling problems, even if the word "pattern" never appears.
license: MIT
---

# Pattern design

A design pattern is a **named, reusable answer to a recurring set of forces**. A force is a pressure on the design, for example "callers must not know the concrete class", "this must survive a downstream outage" or "reads outnumber writes 1000:1". Patterns earn their keep in two ways. They are a shared vocabulary ("put a Circuit Breaker in front of the billing client"), and they are tested solutions whose costs are already known. They are never a goal in themselves. Code that uses no named patterns but solves the problem simply is better than code that shows off five.

This file is the **map**: every pattern grouped by topic, each with a one-line use case. The **territory** lives in `references/<category>/<pattern>.md`. Each reference file covers intent, the problem, structure, when to use it and when not to, Python and TypeScript examples, trade-offs, pitfalls, a refactoring path and related patterns.

## When you must load the detailed references

Load the reference files (not just this overview) whenever the task is one of these:

| Task mode | Signals | What to load |
|---|---|---|
| **Plan / research** | "plan", "propose an approach", "how should we…", "compare options", writing an ADR or design doc, plan mode | the 2–4 candidate patterns you are weighing |
| **Design from scratch** | new service, module, feature, library, API, data model or app skeleton | the patterns you intend to use, plus the architectural style chosen |
| **Optimization** | slow, does not scale, high memory, high latency, too many DB calls, throughput, cost | the relevant `optimization/`, `concurrency/`, `cloud/` or `data-access/` files |
| **Restructuring / refactoring** | god class, tangled dependencies, hard to test, "clean this up", split a monolith, migrate the architecture | the target patterns *and* the pattern currently in use, so you can read its refactoring path |

Why it matters: one-line summaries are enough to *recognize* a candidate, but not to *apply* it. Most bad pattern use comes from the details: a Singleton that breaks tests, a Repository that leaks ORM queries, a retry with no idempotency, an Event Sourcing system with no snapshotting plan. Those details are in the references. A small bug fix or a factual question answered from this overview needs no reference.

**Procedure for those four modes:**
1. **Name the forces first.** What varies, what must stay stable, what scale, what failure modes, who calls whom. Write them down in one to five bullets.
2. **Shortlist** 2–4 candidates using the selector and the category tables below.
3. **Read their reference files.** Check the "When NOT to use" and "Pitfalls" sections against your forces.
4. **Decide and record it.** In the plan or design, state which pattern was chosen, why it fits the forces, what was rejected and why, and the cost you accept. If no pattern clears the bar, say so and keep the plain solution.
5. **Fit the codebase.** If the project already uses a pattern for this concern, such as an existing repository layer or event bus, extend it rather than introducing a competing one.

## Ground rules

- **Simplest thing first.** Introduce a pattern when a real force exists *now*, or is certain soon. An interface with one implementation, a factory that builds one type and an event bus with one subscriber are usually indirection without benefit. The `code` skill's "simplicity first" rule applies on top of this one.
- **Language idioms dissolve some patterns.** In Python and TypeScript, functions are first-class values. Strategy, Command and Template Method are often just a callable parameter. A Python module is already a Singleton. A context manager or `try/finally` replaces many lifecycle patterns. Generators are Iterators. Decorators (`@functools.cache`, `@retry`) cover Proxy and Decorator. Frameworks also supply patterns for you: FastAPI `Depends` and Angular `inject()` are DI, SQLAlchemy `Session` is Unit of Work plus Identity Map, and RxJS is Observer plus Iterator. Use the built-in form before hand-rolling the classic GoF one.
- **Patterns compose.** Real designs are clusters (see "Common combinations"). Choose the cluster deliberately so the pieces don't fight, for example Active Record mixed with a Repository plus Unit of Work.
- **Name it in code.** Once you use a pattern, name the classes or modules after their role (`OrderRepository`, `PaymentGatewayAdapter`, `RetryPolicy`) so the next reader recognizes it.
- **Measure before optimizing.** The optimization patterns trade complexity for speed. Profile first, apply one pattern, then measure again.

## Selector: problem → candidate patterns

| You need to… | Consider |
|---|---|
| hide which concrete class gets created | Factory, Abstract Factory, DI |
| build a complex object step by step or with many optional parts | Builder, Fluent Builder |
| ensure one shared instance (configuration, connection pool) | DI container scope first, Singleton as a last resort, Multiton for one instance per key |
| make an incompatible API fit yours (a third-party SDK) | Adapter, Facade, Anti-corruption layer (see Hexagonal) |
| add behavior without changing the class (logging, caching, auth) | Decorator, Proxy, Chain of Responsibility, middleware (Intercepting Validator) |
| swap an algorithm at runtime | Strategy (a callable), State when behavior depends on the object's state |
| react to changes or events | Observer, Pub-Sub, Event Aggregator, Property Change Listener, Event-Driven |
| undo/redo, queue or log operations | Command, Memento, Event Sourcing |
| treat a tree and a leaf uniformly | Composite, Visitor for operations over the tree |
| encode business rules that combine | Specification, Chain of Responsibility, Interpreter |
| decouple the domain from the DB, web layer and vendors | Hexagonal / Onion / Clean, Repository, DI |
| make reads fast and scale them separately from writes | CQRS, Read-Through Cache, Sharding, materialized views |
| keep a full audit trail or time travel | Event Sourcing, Snapshot, Event Replay |
| run a multi-step business transaction across services | Saga, Idempotent Receiver, Message Queue (outbox) |
| survive a flaky or slow dependency | Retry with backoff, Circuit Breaker, Bulkhead, Rate Limiter, Ambassador |
| process a stream or batch in stages | Pipes and Filters, Data Pipeline, Producer-Consumer, Splitter/Aggregator |
| do work concurrently without races | Thread Pool, Producer-Consumer, Future/Promise, Read-Write Lock, Active Object, Event Loop |
| limit how often something runs | Rate Limiter, Balking, Scheduler, debounce/throttle (Rate Limiter reference) |
| reduce memory or allocation churn | Flyweight, Object Pool, Object Recycling, Data-Oriented Design |
| avoid repeated expensive work | Caching, Lazy Load, Lazy Initialization, Write-Behind Cache |
| test code with external dependencies | Fake, Stub, Spy, Mock, DI to inject them, Golden Master for legacy code |
| make a codebase pluggable | Microkernel, Extension Object, Strategy plus a registry, Service Locator (with care) |
| structure a UI | MVVM (Angular/Vue signals), MVC, MVP, Presentation Model, Front Controller (router) |

## Catalog by topic

Paths are relative to this skill's directory. Every row links to its reference file.

### Creational: how objects get made
These control *who* decides which concrete type is created, *when* it is created, and *how many* instances exist. Reach for them when construction logic is duplicated, conditional, expensive or tied to concrete classes.

| Pattern | Use when | Reference |
|---|---|---|
| Singleton | exactly one instance must exist process-wide and DI isn't available (prefer DI scope) | `references/creational/singleton.md` |
| Factory (Simple / Factory Method) | the caller shouldn't know or choose the concrete class; selection by config or input | `references/creational/factory.md` |
| Abstract Factory | you create *families* of related objects that must match (a UI theme, a cloud provider kit) | `references/creational/abstract-factory.md` |
| Builder | an object has many optional parts or must be assembled and validated in steps | `references/creational/builder.md` |
| Fluent Builder | the same as Builder, with a chainable readable API (query builders, test data) | `references/creational/fluent-builder.md` |
| Prototype | cloning a configured instance is cheaper or simpler than building one | `references/creational/prototype.md` |
| Object Pool | creation is expensive and instances are reusable (DB connections, workers) | `references/creational/object-pool.md` |
| Lazy Initialization | a costly resource may never be needed; defer it until first use | `references/creational/lazy-initialization.md` |
| Multiton | one instance *per key* (per tenant, per region client) | `references/creational/multiton.md` |
| Object Mother | tests need canonical, named, ready-made domain objects | `references/creational/object-mother.md` |

### Structural: how objects fit together
These compose classes and objects into larger structures while keeping them loosely coupled. Reach for them at integration seams, when wrapping or adapting, and when modeling part/whole hierarchies.

| Pattern | Use when | Reference |
|---|---|---|
| Adapter | an existing interface doesn't match the one your code expects | `references/structural/adapter.md` |
| Bridge | abstraction and implementation must vary independently (shape × renderer) | `references/structural/bridge.md` |
| Wrapper | umbrella term: choosing between Adapter, Decorator, Proxy and Facade | `references/structural/wrapper.md` |
| Decorator | add responsibilities dynamically and stackably (logging, caching, retry) | `references/structural/decorator.md` |
| Facade | give a simple entry point to a complex subsystem | `references/structural/facade.md` |
| Proxy | control access to an object (lazy, remote, protection, caching) | `references/structural/proxy.md` |
| Dynamic Proxy | generate proxies at runtime for cross-cutting concerns (`__getattr__`, JS `Proxy`) | `references/structural/dynamic-proxy.md` |
| Flyweight | very many similar objects; share the immutable intrinsic state | `references/structural/flyweight.md` |
| Marker Interface | tag types with a capability without adding methods | `references/structural/marker-interface.md` |
| Extension Object | add interfaces to an object at runtime without changing its class | `references/structural/extension-object.md` |
| Module | group related code behind a public surface with private internals | `references/structural/module.md` |

**Component patterns** (part/whole hierarchies and composition over inheritance):

| Pattern | Use when | Reference |
|---|---|---|
| Composite | treat individual objects and groups uniformly (trees: UI, file system, org chart) | `references/structural/composite.md` |
| Component-Based Design | build entities from pluggable components instead of deep inheritance | `references/structural/component-based-design.md` |
| ECS (Entity-Component-System) | many entities, data-heavy simulation and games; data separated from systems | `references/structural/entity-component-system.md` |
| COP (Component-Oriented Programming) | independently deployable, contract-defined components (plugins, Angular components) | `references/structural/component-oriented-programming.md` |

### Behavioral: how objects collaborate
These distribute responsibility and communication between objects. Reach for them when conditionals multiply, when behavior must be swappable, or when objects must react to each other without tight coupling.

| Pattern | Use when | Reference |
|---|---|---|
| Observer | dependents must be notified when a subject changes (signals, RxJS, events) | `references/behavioral/observer.md` |
| Strategy | interchangeable algorithms selected at runtime | `references/behavioral/strategy.md` |
| Command | wrap a request as an object: queue, log, undo, retry | `references/behavioral/command.md` |
| State | behavior changes with internal state; replace a big `if state ==` block | `references/behavioral/state.md` |
| Template Method | fixed algorithm skeleton, with steps overridable by subclasses | `references/behavioral/template-method.md` |
| Double Dispatch | the operation depends on the runtime types of *two* objects | `references/behavioral/double-dispatch.md` |
| Visitor | add operations over a stable object structure (AST, document tree) | `references/behavioral/visitor.md` |
| Iterator | traverse a collection without exposing its internals (generators) | `references/behavioral/iterator.md` |
| Chain of Responsibility | pass a request along handlers until one handles it (middleware) | `references/behavioral/chain-of-responsibility.md` |
| Mediator | many objects talk through one hub instead of to each other | `references/behavioral/mediator.md` |
| Memento | snapshot and restore state without breaking encapsulation (undo) | `references/behavioral/memento.md` |
| Interpreter | evaluate sentences of a small language or rule DSL | `references/behavioral/interpreter.md` |
| Retry | transient failures should be retried with backoff and jitter | `references/behavioral/retry.md` |
| Property Change Listener | observe changes to individual properties (form fields, view models) | `references/behavioral/property-change-listener.md` |
| Event Aggregator | one channel collects events from many sources for many listeners | `references/behavioral/event-aggregator.md` |

**Specialized behaviors** (small patterns that remove special cases):

| Pattern | Use when | Reference |
|---|---|---|
| Null Object | replace `None` checks with a do-nothing implementation | `references/behavioral/null-object.md` |
| Black Hole | a sink that accepts and silently discards everything (no-op logger, metrics) | `references/behavioral/black-hole.md` |
| Specification | combinable business rules (`is_active & ~is_banned`), reusable in queries | `references/behavioral/specification.md` |
| Role Object | one entity plays different roles in different contexts (customer, supplier) | `references/behavioral/role-object.md` |

**Injection patterns** (how a dependency reaches its consumer):

| Pattern | Use when | Reference |
|---|---|---|
| Dependency Injection | consumers receive dependencies instead of creating them (testability, swapping) | `references/behavioral/dependency-injection.md` |
| Constructor Injection | the default form: dependencies are required and immutable | `references/behavioral/constructor-injection.md` |
| Setter Injection | the dependency is optional or must change after construction | `references/behavioral/setter-injection.md` |
| Interface Injection | the dependency supplies an injector method the consumer implements | `references/behavioral/interface-injection.md` |

### Architectural: how the whole system is shaped
These are system-level styles that decide where business logic lives and which way dependencies point. Choose one *primary* style per deployable and apply it consistently.

| Pattern | Use when | Reference |
|---|---|---|
| Layered (n-tier) | a conventional app; presentation → application → domain → infrastructure | `references/architectural/layered.md` |
| Clean Architecture | the domain must be independent of frameworks, DB and UI; dependencies point inward | `references/architectural/clean-architecture.md` |
| Hexagonal (Ports and Adapters) | many I/O channels (HTTP, CLI, queue, tests) driving one core | `references/architectural/hexagonal.md` |
| Onion | Clean or Hexagonal variant with explicit concentric domain rings | `references/architectural/onion.md` |
| MVC | server-rendered or classic UI; separate the model, the view and input handling | `references/architectural/mvc.md` |
| MVVM | data-bound UIs (Angular, Vue, WPF); the view binds to view-model state | `references/architectural/mvvm.md` |
| MVP | a passive view with a testable presenter (legacy UI, Android) | `references/architectural/mvp.md` |
| ECB (Entity-Control-Boundary) | use-case-driven analysis; boundaries for I/O, controls for use cases | `references/architectural/ecb.md` |
| Pipes and Filters | data flows through independent transformation stages | `references/architectural/pipes-and-filters.md` |
| Event-Driven | components react to events asynchronously; loose temporal coupling | `references/architectural/event-driven.md` |
| Event Sourcing | store state as an append-only log of events; audit, replay, projections | `references/architectural/event-sourcing.md` |
| Publish-Subscribe | publishers broadcast to topics without knowing the subscribers | `references/architectural/publish-subscribe.md` |
| Microservices | independent teams and deployables per bounded context; scale them separately | `references/architectural/microservices.md` |
| Service-Oriented (SOA) | enterprise services with shared contracts, often through an ESB | `references/architectural/soa.md` |
| Service Locator | a registry looked up at runtime (plugins); usually prefer DI | `references/architectural/service-locator.md` |
| Client-Server | clients request, a central server responds (the basis of web apps) | `references/architectural/client-server.md` |
| Broker | an intermediary routes requests between distributed components | `references/architectural/broker.md` |
| Blackboard | several specialist modules incrementally solve a problem on shared state | `references/architectural/blackboard.md` |
| Microkernel (Plug-in) | a minimal core plus plug-ins (IDEs, CLIs, pipelines) | `references/architectural/microkernel.md` |

### Data access: how the domain meets persistence
Choose these as a set. Active Record and Data Mapper are competing foundations. Repository, Unit of Work and Query Object sit on top of a Data Mapper.

| Pattern | Use when | Reference |
|---|---|---|
| Domain Model | rich business rules live in behavior-carrying domain objects | `references/data-access/domain-model.md` |
| Repository | a collection-like interface over aggregates; hides the persistence technology | `references/data-access/repository.md` |
| Unit of Work | track changes and commit them atomically in one transaction (SQLAlchemy Session) | `references/data-access/unit-of-work.md` |
| Data Mapper | domain objects stay persistence-ignorant; a mapper moves the data (SQLAlchemy ORM, TypeORM DataMapper) | `references/data-access/data-mapper.md` |
| Active Record | simple CRUD where each object saves itself (Django ORM, Rails) | `references/data-access/active-record.md` |
| Query Object | queries built as composable objects instead of raw strings | `references/data-access/query-object.md` |
| Table Data Gateway | one object per table that handles all its rows (SQL-centric code) | `references/data-access/table-data-gateway.md` |
| Row Data Gateway | one object per row that handles its own SQL, without domain logic | `references/data-access/row-data-gateway.md` |
| Value List Handler | page or iterate large result sets for the client | `references/data-access/value-list-handler.md` |
| Snapshot | a point-in-time copy of state (event-store snapshots, versioned reads) | `references/data-access/snapshot.md` |

### Security: protecting the boundary
Centralize these. Security spread across every handler eventually gets one handler wrong.

| Pattern | Use when | Reference |
|---|---|---|
| Authorization (RBAC / ABAC / policy) | decide *who may do what* in one place, enforced everywhere | `references/security/authorization.md` |
| Secure Proxy | gate access to a sensitive object or service behind checks | `references/security/secure-proxy.md` |
| Single Access Point | one entry point (gateway, login) through which all access flows | `references/security/single-access-point.md` |
| Intercepting Validator | validate and sanitize all input centrally before handlers (Pydantic, pipes) | `references/security/intercepting-validator.md` |
| Secure Factory / Builder | object creation that embeds security credentials or policy consistently | `references/security/secure-factory-builder.md` |
| Private Class Data | restrict write access to sensitive attributes after construction | `references/security/private-class-data.md` |

### Concurrency: doing many things at once safely
First choose the model: asyncio or the event loop for I/O-bound work, processes for CPU-bound work, threads for blocking libraries. Then choose the pattern.

| Pattern | Use when | Reference |
|---|---|---|
| Thread Pool | reuse a bounded set of workers for many tasks (`ThreadPoolExecutor`) | `references/concurrency/thread-pool.md` |
| Producer-Consumer | decouple production from consumption with a bounded queue (backpressure) | `references/concurrency/producer-consumer.md` |
| Guarded Suspension | wait until a condition holds before proceeding (`Condition.wait_for`) | `references/concurrency/guarded-suspension.md` |
| Barrier | N workers must all reach a point before any continues | `references/concurrency/barrier.md` |
| Read-Write Lock | many concurrent readers, exclusive writers | `references/concurrency/read-write-lock.md` |
| Active Object | method calls become asynchronous requests run on the object's own thread | `references/concurrency/active-object.md` |
| Monitor Object | an object whose methods are mutually exclusive, with condition waits | `references/concurrency/monitor-object.md` |
| Double-Checked Locking | lazy init of a shared resource without locking on every access | `references/concurrency/double-checked-locking.md` |
| Lock Striping (Lock Split) | reduce contention by locking partitions instead of the whole structure | `references/concurrency/lock-striping.md` |
| Scheduler | control when and in what order tasks run (cron, priority queues) | `references/concurrency/scheduler.md` |
| Balking | skip an operation when the object isn't in the right state (already running) | `references/concurrency/balking.md` |
| Future / Promise | a placeholder for a result computed asynchronously (`asyncio.Future`, JS `Promise`) | `references/concurrency/future-promise.md` |
| Work Stealing | idle workers take tasks from busy workers' queues (load balancing) | `references/concurrency/work-stealing.md` |
| Rate Limiter | cap the request rate (token bucket, sliding window, debounce/throttle) | `references/concurrency/rate-limiter.md` |
| Leader-Follower | a pool of threads takes turns demultiplexing and handling events | `references/concurrency/leader-follower.md` |

### Testing: making code verifiable
Test doubles differ in what they *verify*. Stubs and fakes supply state. Spies and mocks verify interactions. Prefer fakes and state checks; overuse of mocks couples tests to implementation details.

| Pattern | Use when | Reference |
|---|---|---|
| Test Stub | return canned answers to drive the code under test down a path | `references/testing/test-stub.md` |
| Test Spy | record calls for later assertions while behaving like a stub | `references/testing/test-spy.md` |
| Fake Object | a working lightweight implementation (in-memory repository, SQLite) | `references/testing/fake-object.md` |
| Mock Object | pre-programmed expectations about calls, verified by the mock itself | `references/testing/mock-object.md` |
| Golden Master (Approval / Snapshot) | lock in current behavior of legacy code before refactoring it | `references/testing/golden-master.md` |
| Data-Driven Testing | the same test logic over a table of inputs (`pytest.mark.parametrize`, `it.each`) | `references/testing/data-driven-testing.md` |
| Property-Based Testing | assert invariants over generated inputs (Hypothesis, fast-check) | `references/testing/property-based-testing.md` |
| BDD | specify behavior as Given/When/Then scenarios shared with stakeholders | `references/testing/bdd.md` |
| Arrange-Act-Assert | the structure of every single test: setup, one action, then checks | `references/testing/arrange-act-assert.md` |

### UI: presentation-layer organization
These complement MVC, MVVM and MVP from the architectural section. In a modern SPA (Angular), the router acts as the Front Controller, components with signals act as the view model, and services act as the presentation model and store.

| Pattern | Use when | Reference |
|---|---|---|
| Presentation Model | view state and behavior extracted into a UI-agnostic, testable class | `references/ui/presentation-model.md` |
| View Helper | formatting and display logic out of templates (pipes, template filters) | `references/ui/view-helper.md` |
| Page Controller | one controller per page or route (simple server apps) | `references/ui/page-controller.md` |
| Application Controller | centralized screen-flow and navigation logic (wizards, workflows) | `references/ui/application-controller.md` |
| Front Controller | a single handler for all requests that dispatches to handlers (routers, middleware) | `references/ui/front-controller.md` |
| CQS (Command-Query Separation) | methods either change state or return data, never both | `references/ui/cqs.md` |
| PAC (Presentation-Abstraction-Control) | a hierarchy of agents, each with its own presentation, abstraction and control (complex dashboards, micro-frontends) | `references/ui/pac.md` |

### Enterprise integration: systems talking to systems
Messaging decouples systems in time and space. The price is eventual consistency and the need for idempotency.

| Pattern | Use when | Reference |
|---|---|---|
| Service Layer | an application boundary that orchestrates use cases and transactions | `references/enterprise/service-layer.md` |
| DTO (Data Transfer Object) | shape the data that crosses a boundary (API schemas, Pydantic models) | `references/enterprise/dto.md` |
| Message Broker | middleware that routes messages between producers and consumers (RabbitMQ, Kafka) | `references/enterprise/message-broker.md` |
| Message Queue | point-to-point async delivery with buffering and retry (SQS, Celery) | `references/enterprise/message-queue.md` |
| ESB (Enterprise Service Bus) | central integration backbone with routing and transformation (legacy enterprise) | `references/enterprise/esb.md` |
| Aggregator | combine related messages or responses into one (scatter-gather, BFF) | `references/enterprise/aggregator.md` |
| Splitter | break a composite message into parts processed individually | `references/enterprise/splitter.md` |
| Content Filter | strip a message down to the fields a consumer needs | `references/enterprise/content-filter.md` |
| Message Translator | convert between message formats or schemas (anti-corruption layer) | `references/enterprise/message-translator.md` |

### Cloud / distributed: resilience and scale
Assume the network fails, latency spikes and nodes die. Each of these patterns handles one of those facts.

| Pattern | Use when | Reference |
|---|---|---|
| Circuit Breaker | stop calling a failing dependency and fail fast, probing for recovery | `references/cloud/circuit-breaker.md` |
| Bulkhead | isolate resources per dependency or tenant so one failure can't sink all | `references/cloud/bulkhead.md` |
| Leader Election | exactly one node should perform a task (cron, coordinator) | `references/cloud/leader-election.md` |
| Sharding | split data horizontally across nodes by a key to scale writes and storage | `references/cloud/sharding.md` |
| CQRS | separate write and read models (different shapes, stores, scaling) | `references/cloud/cqrs.md` |
| Saga | a distributed transaction as local steps with compensations | `references/cloud/saga.md` |
| Serverless | event-triggered functions, pay per use, no server management | `references/cloud/serverless.md` |

### Optimization: making it fast and lean
Profile first. Each of these trades simplicity or freshness for speed.

| Pattern | Use when | Reference |
|---|---|---|
| Caching (cache-aside) | repeated expensive reads with tolerable staleness | `references/optimization/caching.md` |
| Lazy Load | load related data only when accessed (ORM relations, lazy routes) | `references/optimization/lazy-load.md` |
| Read-Through Cache | the cache itself loads on a miss; callers only see the cache | `references/optimization/read-through-cache.md` |
| Write-Behind Cache | writes go to the cache and are flushed to the store asynchronously in batches | `references/optimization/write-behind-cache.md` |
| Micro-Optimizations | hot-loop tweaks after profiling (locals, comprehensions, avoiding allocations) | `references/optimization/micro-optimizations.md` |
| Object Recycling | reuse objects instead of allocating them in hot paths (GC pressure) | `references/optimization/object-recycling.md` |
| Data-Oriented Design | lay out data for access patterns (struct-of-arrays, columnar, NumPy) | `references/optimization/data-oriented-design.md` |
| SIMD | one instruction over many data elements (NumPy vectorization, typed arrays) | `references/optimization/simd.md` |

### Emergent / modern: patterns from current practice
| Pattern | Use when | Reference |
|---|---|---|
| Game Loop | fixed-cadence update and render cycle (games, simulations, polling agents) | `references/emergent/game-loop.md` |
| Dependency Graph | order work by dependencies (build systems, DI resolution, task DAGs, signals) | `references/emergent/dependency-graph.md` |
| Idempotent Receiver | processing the same message twice has the same effect as once | `references/emergent/idempotent-receiver.md` |
| Sidecar | cross-cutting infrastructure in a co-deployed process (proxy, log shipper) | `references/emergent/sidecar.md` |
| Ambassador | an out-of-process proxy that handles a client's remote-call concerns (retry, TLS, routing) | `references/emergent/ambassador.md` |
| Event Loop | single-threaded concurrency over non-blocking I/O (asyncio, Node, browser) | `references/emergent/event-loop.md` |
| Metadata Mapping | mapping or behavior driven by declarative metadata (decorators, Pydantic fields, ORM annotations) | `references/emergent/metadata-mapping.md` |
| Value Object | immutable, equality-by-value domain types (Money, Email) | `references/emergent/value-object.md` |
| Optional | make absence explicit in the type (`T \| None`, `Maybe`, `?.`) | `references/emergent/optional.md` |
| Interpreter Context | state passed through an evaluation (context vars, request context, environments) | `references/emergent/interpreter-context.md` |
| Event Replay | rebuild state or projections by replaying stored events | `references/emergent/event-replay.md` |
| Spatial Partitioning | index objects by location for fast proximity queries (grids, quadtrees, geohash) | `references/emergent/spatial-partitioning.md` |
| Data Pipeline | ingest → transform → load stages with checkpoints (ETL/ELT, streaming) | `references/emergent/data-pipeline.md` |
| Task Queue | offload slow or deferred work to background workers (Celery, RQ, BullMQ) | `references/emergent/task-queue.md` |

**Domain-specific patterns.** Mature domains have their own catalogs: DDD tactical patterns (Aggregate, Domain Event, Bounded Context, see `data-access/domain-model.md`), game programming, ML systems (feature store, model registry), finance (double-entry ledger) and others. When working in such a domain, name its canonical patterns and use them alongside the general ones above.

## Common combinations

| Cluster | Pieces | Typical fit |
|---|---|---|
| Classic backend service | Layered or Clean + Service Layer + Repository + Unit of Work + DTO + DI | FastAPI or NestJS CRUD-plus-rules apps |
| Ports and adapters core | Hexagonal + DI + Adapter + Fake repositories in tests | domain-heavy services with many integrations |
| Event-driven system | Pub-Sub + Message Queue + Idempotent Receiver + Saga + transactional outbox | microservices exchanging state changes |
| Audit and time travel | Event Sourcing + CQRS + Snapshot + Event Replay | ledgers, compliance, collaborative editing |
| Resilient outbound call | Timeout + Retry (backoff and jitter) + Circuit Breaker + Bulkhead (+ Ambassador or Sidecar) | any network dependency |
| Modern SPA (Angular) | MVVM (signals) + DI services as Presentation Model + Observer (RxJS) + Front Controller (router) + Facade over the store | Angular 17+ apps |
| Background processing | Task Queue + Producer-Consumer + Retry + Idempotent Receiver + Rate Limiter | email, reports, webhooks |
| Testable legacy refactor | Golden Master → extract seams → DI → Fakes → Strangler-style replacement | restructuring old code |

## Anti-pattern alarms

Stop and reconsider when you see any of these:
- A pattern introduced "for future flexibility" with one implementation and no concrete second use case.
- A Singleton or Service Locator used to avoid passing dependencies. This creates hidden global state and painful tests.
- Microservices adopted before the module boundaries are clear inside a monolith (a distributed monolith).
- CQRS or Event Sourcing applied to simple CRUD.
- A Repository that returns ORM query objects, or one generic `Repository<T>` with every method, which leaks persistence anyway.
- Retry without idempotency, or retry without a cap and jitter (a retry storm).
- Cache added with no invalidation or TTL strategy.
- Mocks asserting every internal call, so tests break on every refactor.
