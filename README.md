
# Pattern Design: The Ultimate Catalog of Software Design Patterns

Welcome to **Pattern Design**, a production-grade, highly structured catalog of software design patterns designed for modern engineers using **Python** and **TypeScript**.

A design pattern is a **named, reusable answer to a recurring set of forces**. Whether you are dealing with conflicting pressures like *"callers must not know the concrete class,"* *"this service must survive a downstream outage,"* or *"reads outnumber writes 1000:1,"* this skill provides battle-tested solutions with explicit trade-offs, anti-patterns, and language-idiomatic implementations.

---

## 🧭 Quick Navigation: Problem → Pattern Selector

Stuck on a design problem? Find your pattern quickly using this high-level map:

| You need to… | Consider |
| --- | --- |
| **Hide which concrete class gets created** | Factory, Abstract Factory, DI |
| **Build a complex object step-by-step** | Builder, Fluent Builder |
| **Ensure a single shared instance** | DI Container Scope, Singleton (last resort), Multiton |
| **Make an incompatible API fit yours** | Adapter, Facade, Anti-Corruption Layer |
| **Add cross-cutting behavior without modifying classes** | Decorator, Proxy, Chain of Responsibility, Middleware |
| **Swap an algorithm or behavior at runtime** | Strategy (callable), State |
| **React to asynchronous changes or events** | Observer, Pub-Sub, Event-Driven, Event Sourcing |
| **Decouple domain logic from DB & frameworks** | Clean Architecture, Hexagonal, Repository, Unit of Work |
| **Scale reads separately from writes** | CQRS, Read-Through Cache, Sharding |
| **Survive flaky or slow network dependencies** | Retry with Backoff, Circuit Breaker, Bulkhead, Rate Limiter |
| **Test code safely with external dependencies** | Fakes, Stubs, Spies, Mocks, Dependency Injection |

---

## 📚 Complete Pattern Catalog by Topic

Every pattern maps to a dedicated reference file covering its core intent, problem statement, structural architecture, Python/TypeScript code examples, trade-offs, pitfalls, and refactoring paths.

### 1. Creational Patterns (Object Creation)

Control *who* decides which concrete type is created, *when* it is created, and *how many* instances exist.

* **Singleton:** Ensure process-wide uniqueness when DI isn't available.
* **Factory (Simple / Factory Method):** Hide concrete class selection logic.
* **Abstract Factory:** Create families of related objects (e.g., UI themes, cloud SDK kits).
* **Builder & Fluent Builder:** Step-by-step construction of complex or optional objects.
* **Prototype:** Clone a pre-configured instance instead of rebuilding from scratch.
* **Object Pool / Multiton / Lazy Initialization:** Manage expensive or keyed resources efficiently.

### 2. Structural Patterns (Object Composition)

Compose classes and objects into larger, loosely coupled structures at integration seams.

* **Adapter / Bridge / Wrapper:** Reconcile mismatched interfaces.
* **Decorator:** Dynamically stack responsibilities (logging, caching, security).
* **Facade:** Expose a simplified entry point to a complex subsystem.
* **Proxy & Dynamic Proxy:** Control access to objects lazily, remotely, or via reflection (`__getattr__`, JS `Proxy`).
* **Composite & Component-Based Design:** Treat hierarchies and parts uniformly (UI trees, file systems, ECS).

### 3. Behavioral Patterns (Object Collaboration)

Distribute responsibility and communication cleanly without tight coupling.

* **Observer & Pub-Sub:** Reactive notifications (Signals, RxJS, Event Aggregators).
* **Strategy & State:** Swap algorithms or alter behavior dynamically based on internal state.
* **Command & Memento:** Wrap requests as objects for queuing, logging, undo/redo capabilities.
* **Chain of Responsibility & Mediator:** Pass requests through pipeline handlers or centralize hub communications.
* **Template Method & Visitor:** Fixed algorithmic skeletons or operations over stable AST structures.
* **Specification & Null Object:** Clean up business rule combinations and eliminate `None`/`null` checks.

### 4. Architectural Patterns (System Topology)

Decide where business logic lives and dictate dependency direction.

* **Layered (n-tier) & Clean Architecture:** Domain-centric design with inward-pointing dependencies.
* **Hexagonal (Ports and Adapters):** Isolate core domain logic from external HTTP, CLI, and database drivers.
* **MVC, MVVM & MVP:** Presentation-layer organization for server-rendered or reactive single-page applications.
* **Event-Driven & Event Sourcing:** Asynchronous temporal decoupling and append-only audit logs.
* **Microservices & Microkernel (Plug-in):** Modular, independently deployable boundaries and extensible cores.

### 5. Data Access & Persistence

Bridge the gap between domain logic and underlying databases.

* **Domain Model vs. Active Record vs. Data Mapper:** Choose your persistence paradigm (Django ORM vs. SQLAlchemy/TypeORM DataMapper).
* **Repository & Unit of Work:** Collection-like aggregate interfaces backed by atomic transaction tracking.
* **Query Object & Snapshot:** Composable query builders and point-in-time state records.

### 6. Security, Concurrency & Cloud Resilience

Protect system boundaries, scale workers safely, and ensure high availability over unstable networks.

* **Security:** RBAC/ABAC Authorization, Secure Proxy, Intercepting Validators.
* **Concurrency:** Thread Pools, Producer-Consumer queues, Read-Write Locks, Futures/Promises, Rate Limiters, and Event Loops.
* **Cloud & Distributed:** Circuit Breakers, Bulkheads, Leader Election, Sharding, CQRS, and Distributed Sagas.
* **Optimization:** Caching (Cache-Aside, Write-Behind), Lazy Loading, Object Recycling, and Data-Oriented Design.

---

## 🛠️ How to Use This Skill in Your Workflow

1. **Name the Forces First:** Before writing code, list 1 to 5 constraints (e.g., what must vary, scaling requirements, failure modes, caller/callee boundaries).
2. **Shortlist Candidates:** Use the selectors above to pick 2–4 candidate patterns.
3. **Check Language Idioms:** Remember that Python and TypeScript features often replace classic GoF boilerplate (e.g., first-class functions act as strategies, generators act as iterators, decorators replace proxies, and framework hooks like FastAPI `Depends` handle DI).
4. **Apply Simplicity First:** Introduce a pattern *only* when a real force exists today or is guaranteed tomorrow. Avoid speculative over-engineering.