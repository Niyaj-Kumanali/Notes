# Coupling & Cohesion

---

## Overview

- **Definition:** Coupling measures how much one module depends on others. Cohesion measures how closely related the responsibilities within a single module are.
- **Why It Exists:** These concepts guide design quality — without them, systems degrade into spaghetti code that is rigid, fragile, and immobile.

- **Historical Context:** Coupling and cohesion were formalized by Larry Constantine in the late 1960s as part of structured design, later incorporated into OOP by Grady Booch and others. Before these concepts, there was no vocabulary to describe *why* one system was easier to maintain than another — teams relied on intuition and experience. The key insight was that these two properties are independent yet complementary: you can have low coupling with low cohesion (many small unrelated modules that barely talk to each other), and you can have high cohesion with high coupling (one module that does one thing well but everything depends on it). The goal of "high cohesion, low coupling" is an optimization problem, not a binary state.
- **Key Concepts:** **Coupling levels** (Content → Common → External → Control → Stamp → Data → Message), **Cohesion levels** (Coincidental → Logical → Temporal → Procedural → Communicational → Sequential → Functional), **High cohesion + low coupling** is the goal.

---

## Core Concepts

### Coupling Types (Worst to Best)

- **Content Coupling:** One module directly modifies another's internal data.
- **Common Coupling:** Multiple modules share the same global data.
- **External Coupling:** Modules depend on an external system (DB, API, filesystem).
- **Control Coupling:** One module passes flags dictating another's behavior.
- **Stamp Coupling:** Passing an entire data structure when only part is needed.
- **Data Coupling:** Communicating through simple data parameters only.
- **Message Coupling:** Modules communicate only through events/messages.

```java
// Content coupling — direct field access
class OrderService {
    public void process(Order order) {
        order.total = order.calculateTotal(); // Bad
    }
}

// Message coupling — event-driven (best)
public class OrderService {
    private EventPublisher eventPublisher;
    public void placeOrder(Cart cart) {
        Order order = new Order(cart);
        orderRepository.save(order);
        eventPublisher.publish(new OrderPlacedEvent(order.getId()));
    }
}
```

### Cohesion Types (Worst to Best)

- **Coincidental:** Arbitrarily grouped elements (e.g., `Utils.java`).
- **Logical:** Grouped by logical similarity, not collaboration.
- **Temporal:** Elements grouped because they run at the same time.
- **Procedural:** Elements follow a procedure but don't share data.
- **Communicational:** Elements operate on the same data.
- **Sequential:** Output of one element is input to another.
- **Functional:** All elements contribute to a single well-defined purpose.

```java
// Coincidental cohesion — random unrelated methods
public class Utils {
    public static String formatDate(LocalDate date) { }
    public static void sendEmail(String to, String body) { }
}

// Functional cohesion — one clear purpose
public class TaxCalculator {
    public Money calculateTax(Order order, TaxCode code) { }
    public Money calculateSalesTax(Money subtotal, Address address) { }
}
```

---

## Common Mistakes

- **God classes** — Adding "just one more method" to an already-large class because it seems related. It looks correct because the method operates on the same data as the existing methods, and creating a new class seems like unnecessary ceremony. In production, the symptom is a 30-minute CI build because every test in the module instantiates the god class and its 15 dependencies — even for a one-line validation change.

- **Shared mutable singletons** — A single `AppCache` instance shared across the entire application, read and written by different threads and modules. It looks correct because it is convenient — any code can read or write cache without wiring it through constructors. In production, the symptom is a race condition that manifests every few days under load: two requests simultaneously update the same cache key, one write is silently lost, and a user sees stale data 30 minutes later. Reproducing it locally is impossible because the timing window is too narrow.

- **Passing entire objects for one field** — A method calls `calculateTax(User user)` when it only needs `user.getAddress().getZipCode()`. This looks correct because it keeps the method signature simple — one parameter instead of three — and avoids changing callers if the zip code moves later. In production, the symptom is a NullPointerException from `user.getAddress()` because the caller passed a partially-initialized `User` that has an address loaded in one code path but not another. The method did not document that it needed an address to be loaded, so the bug surfaces at runtime, not compile time.

- **Boolean parameter flags** — A method like `sendEmail(String to, String body, boolean isUrgent)` where the boolean changes behavior internally. This looks correct because it avoids writing two nearly identical methods, keeping the code DRY. In production, the developer who added the flag is now gone, and the new team member cannot tell from call sites which calls are urgent and which are not — `sendEmail(a, b, true)`, `sendEmail(c, d, false)` — the boolean is an opaque signal. A change to the urgent path accidentally breaks the non-urgent path because they share the same method body.

- **Everything in a utils package** — A `Utils.java` class with 200 unrelated methods: date formatting, encryption, file I/O, email validation, math utilities. It grows because every developer needs a place for "helper" code and `Utils` is the path of least resistance — creating a new class requires justifying its existence. In production, the symptom is that any change to any utility method requires re-deploying all services that depend on `Utils` — and because it is a single JAR, they all have the same version. A security patch to the encryption utility forces a release of the email service that only uses the date formatting method.

- **Extending framework classes** — Writing `class MyController extends BaseController` where `BaseController` is from a third-party framework. This looks correct because the documentation itself shows extending framework classes as the primary integration point, and it requires the least code. In production, a framework upgrade deprecates a method in `BaseController` that `MyController` overrides — the upgrade is blocked for months because the new framework version removes the method entirely, and migrating off the inheritance requires rewriting all controllers.

- **Circular dependencies between modules** — Module A imports from Module B, and Module B imports from Module A, either directly or transitively. This happens because the boundary between modules was not clearly defined upfront — the natural tendency is to put related code wherever is convenient. It looks correct during development because both modules compile and tests pass. In production, the symptom is a build that takes 45 minutes because Maven/Gradle cannot parallelize compilation across circularly-dependent modules, and any change triggers a full rebuild of the entire dependency cycle.

- **Event-driven overuse** — Publishing an event for every state change even when the only subscriber is the same process, synchronous. This looks correct because "events are decoupled" and the pattern is trendy — teams adopt it to appear modern. In production, the symptom is a 10-line code path that you cannot follow without searching for event subscribers across the entire codebase, and debugging requires reading event logs rather than stack traces. A simple synchronous call would be traceable in a single stack frame.

- **Anemic domain model** — Domain objects that are data bags with getters and setters while all business logic lives in service classes. This looks correct because it follows the JavaBean convention and separates "data" from "logic" — a habit many developers learn from early Spring/Hibernate tutorials that put everything in services. In production, the symptom is business logic duplicated across services because the logic naturally belongs to the domain object but was placed in a service instead — now `OrderService.calculateTotal()` and `InvoiceService.calculateTotal()` have drifted apart after two independent bug fixes.

- **Fat interfaces** — A `CrudRepository` interface with `save`, `findById`, `findAll`, `update`, `delete`, `count`, `existsById` that every repository must implement. This looks correct because it provides a complete data access API in one place, and implementing all methods is straightforward with a base class. In production, a read-only reporting service that implements this interface must provide `delete()` — either as an empty method (silently doing nothing when called) or throwing `UnsupportedOperationException` (crashing at runtime). Both lie about the contract. The solution is to split into `ReadableRepository`, `WritableRepository`, `DeletableRepository`.

---

## Key Design Considerations

- **Cost of coupling** — Every dependency is a liability: changes to B may break A, testing A requires B, deploying A may require B

- **Scale inflection point** — In a codebase under 50K lines with 3 developers, tight coupling is an annoyance but rarely blocking. The team can keep the full dependency graph in their heads, and coordination is a conversation away. At 500K lines with 50 developers, coupling becomes the primary driver of build time, test fragility, deployment risk, and onboarding difficulty. The same `Utils` class that was "fine" at small scale becomes a release coordination nightmare at large scale because every team has a different stake in its contents. This is why modular architecture pays no dividends early — the benefits only appear when the system crosses the threshold where no single person can hold it in their head.

- **Compile-time vs. runtime coupling** — Interfaces/DI reduce runtime coupling; direct imports create compile-time coupling
- **Measuring coupling** — CBO (Coupling Between Objects), DIT (Depth of Inheritance Tree), RFC (Response for a Class), LCOM (Lack of Cohesion)
- **Architecture trade-offs** — Monolith (high coupling, medium cohesion), Modular monolith (low within boundaries, high per module), Microservices (very low across, very high per service)
- **Conway's Law** — System structure mirrors org structure; coupling defines team coordination needs
- **YAGNI on decoupling** — Over-decoupling leads to indirection, harder debugging, unnecessary complexity

---

## Real-World Scenarios

### Scenario 1: Payment Gateway Integration
A fintech startup integrates Stripe, PayPal, and Razorpay. Initially, payment logic is scattered across `OrderService`, `InvoiceService`, and `RefundService`. Adding a new gateway (e.g., Adyen) requires modifying all three classes. **Fix:** Introduce a `PaymentGateway` interface. Each gateway implements it. `OrderService` depends only on the interface. Adding Adyen = one new class, zero changes to existing code. This moves from Stamp/Common coupling to Message coupling.

**Interview follow-up:** The new Adyen integration has different error codes and retry semantics than Stripe. How do you handle gateway-specific behavior without leaking Adyen details into the generic `PaymentGateway` interface?

### Scenario 2: Shared Database in Microservices
Two microservices (Order Service and Inventory Service) share a PostgreSQL database. A schema change in the `orders` table (adding a column) causes Inventory Service to crash because its ORM auto-maps all columns. **Fix:** Each service gets its own database. Inventory Service exposes an API for stock queries. Order Service calls this API instead of querying the DB directly. This eliminates Common coupling entirely.

**Non-obvious constraint:** The API call adds network latency that the direct DB query did not have. If Inventory Service's API is down, Order Service cannot validate stock at all — the direct DB query at least returned stale but functional data.

**Interview follow-up:** How do you split a database transaction that previously spanned both Order and Inventory tables — how does the Order Service guarantee atomicity when Inventory now has its own database?

### Scenario 3: Monolith to Modular Monolith Migration
A 500K-line monolith has no clear module boundaries — changing anything breaks something unrelated. The team wants microservices but lacks the resources. **Fix:** Identify bounded contexts (e.g., Billing, Shipping, Auth). Extract each into a Maven/Gradle module with strict dependency rules. Use ArchUnit tests to enforce that Billing never imports Shipping classes. Result: build time drops from 45 min to 8 min, teams own modules independently.

---

## Scenario-Based Questions

1. **Q: You are building a system where a single database table is read and written by 6 different services. A schema migration for one service breaks all 6. How do you refactor?**
   A: This is Common coupling. Each service must own its data. Identify bounded contexts, split the table per service, and introduce API boundaries. Use the Strangler Fig pattern: create new endpoints, migrate consumers one by one. Add contract tests at each service boundary to prevent regressions.

   **Non-obvious constraint:** Splitting a shared table into per-service tables means that queries that previously joined across services in-database now require API calls — what was a 2ms join becomes a 50ms network round trip.

2. **Q: You are designing an event-driven order processing system. The Order Service publishes events; Notification, Inventory, and Shipping subscribe. A bug in Notification causes the event bus to stall, delaying all subscribers. How do you prevent this?**
   A: Use message brokers with consumer isolation (separate queues per subscriber). Implement dead-letter queues, timeouts, and circuit breakers per consumer. Use eventual consistency — the Order Service should not wait for all subscribers to complete. Add monitoring for consumer lag.

   **Non-obvious constraint:** Separate queues guarantee notification cannot block shipping, but the Order Service still publishes to a single topic — if the broker itself is the bottleneck, queue isolation does not help.

3. **Q: Your team maintains a `Utils` class with 200 methods (formatting, math, I/O, encryption, networking). Every developer adds to it. How do you fix this without a rewrite?**
   A: This is Coincidental cohesion (worst type). Use the Strangler pattern: identify groups of related methods (e.g., DateUtils, StringUtils, CryptoUtils), create new classes, deprecate old methods, migrate callers. Use ArchUnit to forbid imports from `Utils` after migration.

4. **Q: A junior developer adds a feature by modifying 15 existing classes. The feature is conceptually one unit. What's the design problem and fix?**
   A: Low cohesion — the feature's logic is scattered across classes that don't own it. Extract the feature into its own class. The 15 existing classes should only change if their public API needs to accommodate the new feature, not to contain its logic.

5. **Q: Two microservices use Redis pub/sub for communication. A message schema change breaks the consumer. How do you make this robust?**
   A: Use a schema registry (e.g., Avro, Protobuf with Schema Registry). Version all messages. The producer publishes v2 while consumers still consume v1 until they migrate. Add consumer-side contract tests that validate message schema compatibility.

   **Interview follow-up:** Redis pub/sub does not persist messages. If the consumer is down during a schema migration window, it misses messages. How do you guarantee delivery during the migration?

6. **Q: Adding a new field to a configuration file requires updating 20 classes that read it. What pattern addresses this?**
   A: Centralize configuration access behind a `ConfigService` interface. Each component reads only the config it needs. New fields are added to the `ConfigService` without changing consumers. This moves from Stamp coupling (passing entire config object) to Data coupling (specific parameters).

7. **Q: An API gateway has grown to 10,000 lines with request validation, auth, rate limiting, logging, and routing all mixed together. What's the cohesion problem and fix?**
   A: Temporal cohesion — everything runs at the same time but for different purposes. Apply the Pipeline pattern: each concern is a separate middleware/filter. Validation, auth, rate limiting, and routing become independent, testable, and reusable components (Functional cohesion).

   **Interview follow-up:** A specific endpoint needs different middleware than the rest — e.g., the health check endpoint should skip auth and rate limiting. How do you design middleware to support per-endpoint overrides without sacrificing cohesion?

8. **Q: Your team uses a shared mutable singleton for caching across the entire application. Race conditions appear sporadically. How to fix?**
   A: This is Common coupling with the singleton as global state. Solutions: (1) Dependency inject the cache — each component gets its own instance or a reference. (2) Use immutable cache entries. (3) Use a distributed cache (Redis) with atomic operations. (4) Write tests that run in parallel to detect race conditions.

9. **Q: A cleanup script sends `null` for all optional parameters. The downstream service crashes with NPE. What coupling type and fix?**
   A: Control coupling — `null` flags are acting as implicit control signals. Fix: (1) Use `Optional<T>` parameters. (2) Better: split the method — one overload with all options, another with defaults. Or use the Null Object pattern: pass a `NoOpHandler` instead of null.

10. **Q: You inherit a system where the UI layer calls the database layer directly, bypassing the service layer. What's the coupling problem and how do you fix it?**
    A: Content coupling — the UI directly accesses data it shouldn't, creating a fragile dependency. Fix: Enforce layered architecture with ArchUnit tests. Move data access behind service interfaces. Use DTOs instead of entities in the UI layer. Each layer communicates only with the layer below through interfaces.

---

## Interview Questions

1. **What is the difference between coupling and cohesion?**
   A: Coupling measures inter-module dependency; cohesion measures intra-module focus. High cohesion + low coupling is the ideal design goal.

2. **List the coupling types from worst to best.**
   A: Content → Common → External → Control → Stamp → Data → Message.

3. **What is Stamp coupling? Give an example.**
   A: Passing an entire object when only one field is needed. E.g., passing a `User` object to a method that only needs `user.email`.

4. **What is Common coupling and why is it dangerous?**
   A: Multiple modules sharing global data. Dangerous because any module can mutate the data, causing unpredictable behavior and race conditions.

5. **What is the highest form of cohesion?**
   A: Functional cohesion — all elements contribute to a single well-defined purpose.

6. **How does the Interface Segregation Principle relate to coupling?**
   A: ISP (the I in SOLID) reduces coupling by ensuring clients depend only on methods they use. Fat interfaces create Stamp coupling.

7. **What metrics measure coupling?**
   A: CBO (Coupling Between Objects), DIT (Depth of Inheritance Tree), RFC (Response for a Class), LCOM (Lack of Cohesion of Methods).

8. **How do events reduce coupling compared to direct calls?**
   A: Events decouple publisher from subscriber — the publisher doesn't know who listens. This enables temporal decoupling (async), location transparency, and independent deployability.

9. **What's the downside of excessive decoupling?**
   A: Indirection, harder debugging, event pipelines with unclear flow, performance overhead, and complexity for simple synchronous flows.

10. **How does Conway's Law relate to coupling?**
    A: System structure mirrors org communication structure. If two teams need to coordinate frequently, their services should be loosely coupled (API/events), not tightly coupled (shared DB).

---

## Developer Recommendations

- **Prefer Message or Data coupling over Common coupling** — Shared mutable state (global variables, singletons, shared databases) is the #1 source of production bugs. Each module should own its data and expose it through APIs or events. A production incident: a shared Redis cache between Payment and Fraud services meant that a bug in Fraud service accidentally flushed keys that Payment service relied on, causing all payment transactions to be re-processed as "first time" — 200K duplicate charges sent to customers before the incident was detected. The cost: more boilerplate for serialization/deserialization. The benefit: modules can be developed, tested, and deployed independently.

- **Target Functional cohesion in every class** — A class should have exactly one reason to change. If "add logging" touches `OrderService`, `PaymentService`, and `ShippingService`, you have cross-cutting concern scattering. Use AOP or middleware instead. The trade-off: more classes in your project. The benefit: each class is independently testable, understandable, and replaceable.

- **Use the Strangler Fig pattern to refactor low-cohesion code** — Don't rewrite 200-method `Utils` classes overnight. Create focused replacements, route new callers, then deprecate. Trade-off: you carry dead code during transition. Benefit: zero risk of breaking production, continuous delivery during refactoring.

- **Enforce architecture rules with ArchUnit** — Manual code reviews miss coupling violations. Automate them: "Services must not access repositories directly", "Controllers must not call other controllers". Trade-off: build time increases by seconds. Benefit: architectural integrity is machine-enforced, not dependent on reviewer vigilance.

- **Prefer interface-based dependencies over concrete classes** — `PaymentProcessor processor` (interface) vs `StripeProcessor processor` (concrete). The interface lets you swap implementations, mock in tests, and reduce compile-time coupling. Trade-off: one extra file per abstraction. Benefit: testability + flexibility without changing callers.

- **Use consumer-driven contracts for service boundaries** — When Service A calls Service B, A defines what it expects. B must pass all consumer contract tests before deploying. Trade-off: contract test maintenance overhead. Benefit: breaking changes are caught in CI, not production.

- **Monitor coupling trends over time** — Track CBO, RFC, and LCOM metrics per release. If CBO of a core module rises above 10, it's a red flag. Trade-off: metric collection tooling. Benefit: early warning system for architectural degradation.

- **Apply the Dependency Inversion Principle consistently** — High-level modules should not depend on low-level modules. Both should depend on abstractions. Trade-off: more indirection. Benefit: low-level details (DB, API, filesystem) can change without affecting business logic.
