# Coupling & Cohesion

---

## Overview

- **Definition:** Coupling measures how much one module depends on others. Cohesion measures how closely related the responsibilities within a single module are.
- **Why It Exists:** These concepts guide design quality — without them, systems degrade into spaghetti code that is rigid, fragile, and immobile.
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

- **God classes** — "One more method" mentality leads to low cohesion and high coupling
- **Shared mutable singletons** — Creates common coupling and race conditions
- **Passing entire objects for one field** — Stamp coupling
- **Boolean parameter flags** — Control coupling; split into specific methods
- **Everything in a utils package** — Coincidental cohesion
- **Extending framework classes** — High coupling to framework
- **Circular dependencies** — No architecture planning
- **Event-driven overuse** — Unnecessary complexity for simple synchronous flows
- **Anemic domain model** — Data separate from behavior, low cohesion
- **Fat interfaces** — Interface pollution forces implementing classes to have empty methods

---

## Key Design Considerations

- **Cost of coupling** — Every dependency is a liability: changes to B may break A, testing A requires B, deploying A may require B
- **Compile-time vs. runtime coupling** — Interfaces/DI reduce runtime coupling; direct imports create compile-time coupling
- **Measuring coupling** — CBO (Coupling Between Objects), DIT (Depth of Inheritance Tree), RFC (Response for a Class), LCOM (Lack of Cohesion)
- **Architecture trade-offs** — Monolith (high coupling, medium cohesion), Modular monolith (low within boundaries, high per module), Microservices (very low across, very high per service)
- **Conway's Law** — System structure mirrors org structure; coupling defines team coordination needs
- **YAGNI on decoupling** — Over-decoupling leads to indirection, harder debugging, unnecessary complexity

---

## Real-World Scenarios

### Scenario 1: Payment Gateway Integration
A fintech startup integrates Stripe, PayPal, and Razorpay. Initially, payment logic is scattered across `OrderService`, `InvoiceService`, and `RefundService`. Adding a new gateway (e.g., Adyen) requires modifying all three classes. **Fix:** Introduce a `PaymentGateway` interface. Each gateway implements it. `OrderService` depends only on the interface. Adding Adyen = one new class, zero changes to existing code. This moves from Stamp/Common coupling to Message coupling.

### Scenario 2: Shared Database in Microservices
Two microservices (Order Service and Inventory Service) share a PostgreSQL database. A schema change in the `orders` table (adding a column) causes Inventory Service to crash because its ORM auto-maps all columns. **Fix:** Each service gets its own database. Inventory Service exposes an API for stock queries. Order Service calls this API instead of querying the DB directly. This eliminates Common coupling entirely.

### Scenario 3: Monolith to Modular Monolith Migration
A 500K-line monolith has no clear module boundaries — changing anything breaks something unrelated. The team wants microservices but lacks the resources. **Fix:** Identify bounded contexts (e.g., Billing, Shipping, Auth). Extract each into a Maven/Gradle module with strict dependency rules. Use ArchUnit tests to enforce that Billing never imports Shipping classes. Result: build time drops from 45 min to 8 min, teams own modules independently.

---

## Scenario-Based Questions

1. **Q: You are building a system where a single database table is read and written by 6 different services. A schema migration for one service breaks all 6. How do you refactor?**
   A: This is Common coupling. Each service must own its data. Identify bounded contexts, split the table per service, and introduce API boundaries. Use the Strangler Fig pattern: create new endpoints, migrate consumers one by one. Add contract tests at each service boundary to prevent regressions.

2. **Q: You are designing an event-driven order processing system. The Order Service publishes events; Notification, Inventory, and Shipping subscribe. A bug in Notification causes the event bus to stall, delaying all subscribers. How do you prevent this?**
   A: Use message brokers with consumer isolation (separate queues per subscriber). Implement dead-letter queues, timeouts, and circuit breakers per consumer. Use eventual consistency — the Order Service should not wait for all subscribers to complete. Add monitoring for consumer lag.

3. **Q: Your team maintains a `Utils` class with 200 methods (formatting, math, I/O, encryption, networking). Every developer adds to it. How do you fix this without a rewrite?**
   A: This is Coincidental cohesion (worst type). Use the Strangler pattern: identify groups of related methods (e.g., DateUtils, StringUtils, CryptoUtils), create new classes, deprecate old methods, migrate callers. Use ArchUnit to forbid imports from `Utils` after migration.

4. **Q: A junior developer adds a feature by modifying 15 existing classes. The feature is conceptually one unit. What's the design problem and fix?**
   A: Low cohesion — the feature's logic is scattered across classes that don't own it. Extract the feature into its own class. The 15 existing classes should only change if their public API needs to accommodate the new feature, not to contain its logic.

5. **Q: Two microservices use Redis pub/sub for communication. A message schema change breaks the consumer. How do you make this robust?**
   A: Use a schema registry (e.g., Avro, Protobuf with Schema Registry). Version all messages. The producer publishes v2 while consumers still consume v1 until they migrate. Add consumer-side contract tests that validate message schema compatibility.

6. **Q: Adding a new field to a configuration file requires updating 20 classes that read it. What pattern addresses this?**
   A: Centralize configuration access behind a `ConfigService` interface. Each component reads only the config it needs. New fields are added to the `ConfigService` without changing consumers. This moves from Stamp coupling (passing entire config object) to Data coupling (specific parameters).

7. **Q: An API gateway has grown to 10,000 lines with request validation, auth, rate limiting, logging, and routing all mixed together. What's the cohesion problem and fix?**
   A: Temporal cohesion — everything runs at the same time but for different purposes. Apply the Pipeline pattern: each concern is a separate middleware/filter. Validation, auth, rate limiting, and routing become independent, testable, and reusable components (Functional cohesion).

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

- **Prefer Message or Data coupling over Common coupling** — Shared mutable state (global variables, singletons, shared databases) is the #1 source of production bugs. Each module should own its data and expose it through APIs or events. The cost: more boilerplate for serialization/deserialization. The benefit: modules can be developed, tested, and deployed independently.

- **Target Functional cohesion in every class** — A class should have exactly one reason to change. If "add logging" touches `OrderService`, `PaymentService`, and `ShippingService`, you have cross-cutting concern scattering. Use AOP or middleware instead. The trade-off: more classes in your project. The benefit: each class is independently testable, understandable, and replaceable.

- **Use the Strangler Fig pattern to refactor low-cohesion code** — Don't rewrite 200-method `Utils` classes overnight. Create focused replacements, route new callers, then deprecate. Trade-off: you carry dead code during transition. Benefit: zero risk of breaking production, continuous delivery during refactoring.

- **Enforce architecture rules with ArchUnit** — Manual code reviews miss coupling violations. Automate them: "Services must not access repositories directly", "Controllers must not call other controllers". Trade-off: build time increases by seconds. Benefit: architectural integrity is machine-enforced, not dependent on reviewer vigilance.

- **Prefer interface-based dependencies over concrete classes** — `PaymentProcessor processor` (interface) vs `StripeProcessor processor` (concrete). The interface lets you swap implementations, mock in tests, and reduce compile-time coupling. Trade-off: one extra file per abstraction. Benefit: testability + flexibility without changing callers.

- **Use consumer-driven contracts for service boundaries** — When Service A calls Service B, A defines what it expects. B must pass all consumer contract tests before deploying. Trade-off: contract test maintenance overhead. Benefit: breaking changes are caught in CI, not production.

- **Monitor coupling trends over time** — Track CBO, RFC, and LCOM metrics per release. If CBO of a core module rises above 10, it's a red flag. Trade-off: metric collection tooling. Benefit: early warning system for architectural degradation.

- **Apply the Dependency Inversion Principle consistently** — High-level modules should not depend on low-level modules. Both should depend on abstractions. Trade-off: more indirection. Benefit: low-level details (DB, API, filesystem) can change without affecting business logic.
