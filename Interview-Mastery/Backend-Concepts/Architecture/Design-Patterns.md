# Design Patterns

---

## Overview

- **Definition:** Reusable, proven solutions to commonly occurring problems in software design, cataloged by the Gang of Four (GoF) in 1994.
- **Why It Exists:** Patterns provide a common vocabulary for developers, promote best practices, encourage loose coupling and high cohesion, and align with frameworks like Spring Boot which implement many patterns internally.
- **Key Concepts:** **Creational Patterns** (Singleton, Factory, Builder — object creation), **Structural Patterns** (Adapter, Decorator, Proxy — object composition), **Behavioral Patterns** (Observer, Strategy, State — object communication)
- **Patterns vs Principles vs Idioms** — Design patterns are reusable solutions (Strategy, Observer). Principles are guidelines (SOLID, KISS, DRY). Idioms are language-specific conventions (JavaBean pattern, try-with-resources). Patterns implement principles; idioms implement patterns in a specific language.
- **When NOT to Use a Pattern** — Patterns add indirection, complexity, and maintenance overhead. If a simple if-else for 3 cases solves the problem, don't add a Strategy pattern. If a single class with 2 methods handles the task, don't add Abstract Factory. Apply patterns where change is expected, not preemptively. YAGNI (You Ain't Gonna Need It) applies to patterns too.

---

## Core Concepts

- **Singleton Pattern:** Ensures a class has only one instance with a global access point. Spring beans are singletons by default. Use Bill Pugh initialization holder pattern or enums for thread safety.
- **Factory Method Pattern:** Defines an interface for creating objects but lets subclasses decide which class to instantiate. Spring's `@Bean` methods are a form of factory.
- **Builder Pattern:** Separates construction of a complex object from its representation. Useful for objects with many optional parameters. Lombok's `@Builder` provides a concise alternative.
- **Adapter Pattern:** Allows incompatible interfaces to work together by converting one interface to another. Used when integrating legacy systems or third-party libraries.
- **Decorator Pattern:** Attaches additional responsibilities to an object dynamically without modifying its class. Layered decorators compose behaviors like encryption, compression, and logging.
- **Proxy Pattern:** Provides a surrogate for another object to control access. Spring uses proxies for `@Transactional`, `@Cacheable`, and AOP. Caching proxies and security proxies are common.
- **Observer Pattern:** Defines a one-to-many dependency where when one object changes state, all dependents are notified. Spring's `@EventListener` implements this pattern.
- **Strategy Pattern:** Defines a family of interchangeable algorithms. Used for payment methods, shipping costs, tax calculations, and authentication providers.
- **Template Method Pattern:** Defines the skeleton of an algorithm, deferring some steps to subclasses. Spring's `JdbcTemplate`, `RestTemplate`, and `JpaRepository` follow this pattern.
- **State Pattern:** Allows an object to alter its behavior when its internal state changes. Order lifecycle management (Pending → Paid → Shipped → Delivered) is a classic example.
- **Command Pattern:** Encapsulates a request as an object, allowing parameterization, queuing, logging, and undoable operations. Implemented via `Runnable`, `Callable`, and functional interfaces in Java. Used in task queues, transactional behavior, and menu systems.
- **Chain of Responsibility:** Passes a request along a chain of handlers. Each handler decides to process or pass to the next. Spring Security's `SecurityFilterChain` is the canonical example — authentication filters, CSRF filters, CORS filters, each processing or delegating.
- **Facade Pattern:** Provides a unified interface to a set of interfaces in a subsystem. REST controllers often act as facades over complex business logic and service layers. The controller exposes a simple API while hiding the complexity of service orchestration behind it.

```java
// Strategy Pattern for shipping costs
public interface ShippingCostStrategy {
    BigDecimal calculate(Order order);
}

public class StandardShipping implements ShippingCostStrategy {
    public BigDecimal calculate(Order order) { return new BigDecimal("5.99"); }
}

public class ExpressShipping implements ShippingCostStrategy {
    public BigDecimal calculate(Order order) { return new BigDecimal("14.99"); }
}

@Service
public class ShippingCalculator {
    private final Map<String, ShippingCostStrategy> strategies;

    public ShippingCalculator(List<ShippingCostStrategy> strategyList) {
        this.strategies = strategyList.stream()
            .collect(Collectors.toMap(s -> s.getClass().getSimpleName().replace("Shipping", "").toLowerCase(), Function.identity()));
    }

    public BigDecimal calculate(String type, Order order) {
        return strategies.getOrDefault(type.toLowerCase(), new StandardShipping()).calculate(order);
    }
}

// Decorator Pattern with layered behavior
DataSource source = new FileDataSource("data.txt");
source = new CompressionDecorator(source);
source = new EncryptionDecorator(source);
// Writing: encrypt -> compress -> file
// Reading: file -> decompress -> decrypt
```

---

## Common Mistakes

- **Pattern Overuse** — applying patterns where a simpler solution suffices. "Hello World" doesn't need Abstract Factory, Builder, Visitor, and Mediator. This *looks correct* because: using patterns shows design sophistication and prepares for future extensibility — the complexity of Abstract Factory for what could be a simple `new` call only becomes technical debt when someone has to trace through four indirection layers to understand the code.
- **Singleton Abuse** — singletons cause hidden dependencies and testability issues. Let Spring manage singleton scope instead of implementing your own. This *looks correct* because: having exactly one instance of a service seems efficient and ensures consistent state — the hidden coupling only surfaces when unit tests can't isolate the singleton's state between test runs.
- **Misapplying Inheritance** — using inheritance when composition is more appropriate. Prefer Strategy over subclassing for varying behaviors. This *looks correct* because: inheritance is the most intuitive OOP mechanism for code reuse, and a base class with overrides seems clean — the coupling becomes problematic when a change to the base class unexpectedly breaks multiple subclasses in different ways.
- **Pattern Rigidity** — treating patterns as rigid rules instead of guidelines. Adapt the pattern to your specific context. This *looks correct* because: patterns are proven solutions documented by experts, and following them strictly seems like the disciplined approach — the inflexibility only becomes a problem when the textbook pattern doesn't quite fit the real-world problem.
- **Implementing Patterns From Scratch** — Spring Boot already implements Proxy, Template Method, Singleton, Factory, and Observer. Leverage the framework. This *looks correct* because: implementing a pattern yourself gives you full control and understanding of how it works — the custom implementation only becomes a maintenance burden when the next developer has to debug a hand-rolled proxy instead of using Spring's well-tested AOP support.
- **Proxies Without Interface (CGLIB vs JDK Dynamic)** — Spring uses JDK Dynamic Proxy when the target implements an interface, CGLIB when it doesn't. CGLIB creates a subclass at runtime. Both have the same performance characteristics. Spring Boot defaults to CGLIB for `@EnableAspectJAutoProxy`. Ensure `@Configuration` classes are not `final` if CGLIB proxies are used. This *looks correct* because: marking a class `final` is a good practice for immutability and design intent — CGLIB's requirement for non-final classes only becomes an issue when Spring fails to create a proxy at startup with an obscure error message.

---

## Key Design Considerations

- **Spring Boot Pattern Mapping** — Singleton (`@Service`, `@Component` default scope), Factory (`@Bean`, `BeanFactory`), Proxy (`@Transactional`, `@Cacheable`, AOP), Template Method (`JdbcTemplate`, `JpaRepository`), Observer (`@EventListener`), Chain of Responsibility (`SecurityFilterChain`), Strategy (`AuthenticationProvider`, `MessageConverter`).
- **Modern Alternatives** — Strategy → lambda expressions and method references; Observer → reactive streams; Command → `Runnable`/`Callable`/`Supplier`; Template Method → functional interface composition; Builder → Lombok `@Builder` or Kotlin data classes.
- **Patterns in Distributed Systems** — Circuit Breaker (Resilience4j), Saga (distributed transactions), CQRS (command/query separation), Event Sourcing (state as event stream), Bulkhead (resource isolation), Sidecar (service mesh deployment).
- **Reactive Patterns** — Observer evolved into Reactive Streams (Publisher/Subscriber). Spring WebFlux uses this pattern for non-blocking I/O. Backpressure (subscriber signals demand to publisher) prevents overwhelming consumers. Hot vs Cold publishers: Cold = each subscriber gets its own stream; Hot = subscribers share the same stream (like a broadcast).
- **Functional Programming as Pattern Alternative** — Many GoF patterns can be replaced with functional constructs: Strategy → lambda/predicate, Command → `Function`/`Consumer`, Template Method → method reference, Visitor → pattern matching (in languages with pattern matching). Functional composition often achieves the same goal with less boilerplate than traditional OOP patterns.
- **Composition Over Inheritance** — prefer composing objects with interfaces over deep class hierarchies. Strategies, Decorators, and Adapters all use composition.

---

## Real-World Scenarios

### Scenario 1: Payment Processing with Strategy Pattern
**Context:** An e-commerce platform needs to support multiple payment methods (credit card, PayPal, Apple Pay, cryptocurrency). Each method has different validation, processing, and error handling flows.

**Resolution:** Use the Strategy pattern. Define a `PaymentStrategy` interface with a `pay(Order order)` method. Each payment method is a separate implementation. The `PaymentService` selects the appropriate strategy at runtime based on user choice. Adding a new payment method requires only a new implementation class — zero changes to existing code.

```java
public interface PaymentStrategy {
    PaymentResult pay(Order order);
}

@Component
public class CreditCardStrategy implements PaymentStrategy {
    public PaymentResult pay(Order order) {
        // Validate card, charge via gateway, handle 3D Secure
    }
}

@Component("paymentStrategyFactory")
public class PaymentStrategyFactory {
    private final Map<String, PaymentStrategy> strategies;
    public PaymentStrategyFactory(List<PaymentStrategy> strategyList) {
        this.strategies = strategyList.stream()
            .collect(Collectors.toMap(s -> s.getClass().getSimpleName()
                .replace("Strategy", "").toLowerCase(), Function.identity()));
    }
    public PaymentStrategy getStrategy(String type) {
        return strategies.getOrDefault(type.toLowerCase(), 
            strategies.get("default"));
    }
}
```

### Scenario 2: Order Processing Pipeline with Chain of Responsibility
**Context:** An order processing system needs to apply multiple validation and enrichment steps: validate inventory → check fraud → apply discounts → calculate tax → charge payment. Each step can stop the pipeline.

**Resolution:** Use the Chain of Responsibility pattern. Each order handler implements `OrderHandler` with `handle(Order, OrderContext)` and a reference to the next handler. The chain is assembled in the correct order. Spring's `SecurityFilterChain` follows the same pattern.

```java
public interface OrderHandler {
    void handle(Order order, OrderContext context);
    OrderHandler setNext(OrderHandler next);
}

@Component
@Order(1)
public class InventoryCheckHandler implements OrderHandler { /* ... */ }

@Component
@Order(2)
public class FraudDetectionHandler implements OrderHandler { /* ... */ }

@Bean
public OrderHandler orderPipeline(List<OrderHandler> handlers) {
    // Spring injects ordered handlers, chain them
    for (int i = 0; i < handlers.size() - 1; i++) {
        handlers.get(i).setNext(handlers.get(i + 1));
    }
    return handlers.get(0);
}
```

### Scenario 3: Notification System with Observer Pattern
**Context:** A banking application must notify multiple channels (email, SMS, push, audit log) when a transaction occurs. New notification channels are added frequently.

**Resolution:** Use the Observer pattern. A `TransactionSubject` publishes `TransactionEvent`. Multiple `NotificationObserver` implementations (EmailNotifier, SMSNotifier, AuditLogger) subscribe to events. Spring's `@EventListener` makes this trivial.

```java
@Component
public class TransactionEventPublisher {
    private final ApplicationEventPublisher publisher;
    public void publish(Transaction transaction) {
        publisher.publishEvent(new TransactionCompleteEvent(transaction));
    }
}

@Component
public class EmailNotifier {
    @EventListener
    public void handle(TransactionCompleteEvent event) {
        // Send email receipt
    }
}

@Component
public class SMSNotifier {
    @EventListener
    @Order(2) // Runs after email
    public void handle(TransactionCompleteEvent event) {
        // Send SMS alert
    }
}
```

---

## Scenario-Based Questions

1. **Q: You are building a notification system where the same event (e.g., order placed) needs to trigger email, SMS, push notification, and audit logging. New channels are added every quarter. How do you design this?**
   - A: Use the Observer pattern. Define a domain event (`OrderPlacedEvent`) published by the order service. Each notification channel is a separate observer/event listener that reacts independently. Spring's `@EventListener` or a message broker (Kafka/RabbitMQ) can implement this. Adding a new channel means adding one class with zero changes to existing code. This follows OCP.

> **Interview follow-up:** You use `@EventListener` and all three notifications (email, SMS, push) execute in the same thread — if the email service is slow, it delays SMS and push delivery. How would you make the observers run asynchronously without losing the event if the application crashes between notifications?

2. **Q: Your team is building a PDF report generator that creates reports in HTML then converts to PDF. Some reports need digital signatures, others need watermarks, others need both. You don't know what future enhancements will be needed. What pattern do you use?**
   - A: The Decorator pattern. Define a `Report` interface with `generate()`. A `BaseReport` generates the core PDF. `SignedReportDecorator` adds digital signatures. `WatermarkedReportDecorator` adds watermarks. Compose them at runtime: `new SignedReportDecorator(new WatermarkedReportDecorator(new BaseReport(data)))`. This gives unlimited combinations without subclass explosion.

> **Interview follow-up:** Your report pipeline now has 5 decorators (signing, watermarking, encryption, compression, audit stamp), and the order of decoration matters — how do you ensure the decorators are applied in the correct order without relying on the caller to stack them manually?

3. **Q: Your application has a complex object with 15 optional parameters (database connection: host, port, credentials, pool size, SSL config, etc.). Constructors with 15 parameters are unreadable and error-prone. How do you solve this?**
   - A: Builder pattern. Create a `DatabaseConfigBuilder` with fluent setter methods that return `this`. A `build()` method validates all required fields and constructs the immutable `DatabaseConfig`. Lombok's `@Builder` annotation auto-generates this. The builder also catches configuration errors at build time rather than runtime.

4. **Q: Your REST API needs to run the same request through authentication, rate limiting, request logging, and input validation before reaching the controller. How does Spring implement this, and what pattern is it?**
   - A: Chain of Responsibility pattern. Spring Security's `SecurityFilterChain` passes the request through a chain of filter objects (`AuthenticationFilter` → `RateLimitFilter` → `LoggingFilter` → `ValidationFilter`). Each filter decides to process the request and/or pass it to the next filter. Filters are ordered and can short-circuit by throwing exceptions.

5. **Q: You have a legacy `XMLOrderService` that your new system needs to use, but your system works with `Order` objects, not XML. You can't modify the legacy service. What pattern bridges this gap?**
   - A: Adapter pattern. Create an `OrderServiceAdapter` that implements your `OrderService` interface. Internally, it converts `Order` objects to XML, calls `XMLOrderService`, and converts the XML response back to `Order`. The adapter encapsulates the conversion logic, making the legacy system invisible to the rest of your code.

6. **Q: A controller method is called from 10 different places. You need to add caching to this method without modifying it or its callers. What pattern does Spring use for this?**
   - A: Proxy pattern. Adding `@Cacheable("products")` to the method causes Spring to create a runtime proxy (CGLIB or JDK Dynamic Proxy). The proxy intercepts calls, checks the cache, and either returns cached data or executes the method and caches the result. The original method and callers are completely unaware of the caching.

7. **Q: Your order processing system needs to execute different tax calculation algorithms based on the customer's country, and new countries are added monthly. How do you avoid a giant if-else chain?**
   - A: Strategy pattern. Define a `TaxCalculator` interface with `calculateTax(Order)`. Each country has its own implementation (`USTaxCalculator`, `EUTaxCalculator`, `UKTaxCalculator`). A factory selects the right strategy based on the customer's country code. Adding a new country is a single new class — the if-else chain is eliminated.

> **Interview follow-up:** Two months later, the EU changes its tax rules so that the calculation now depends on the customer's total order history, not just the current order — how do you handle strategies that need different input data without breaking the existing interface contract?

8. **Q: A document workflow system has states: DRAFT → PENDING_REVIEW → APPROVED → PUBLISHED. The behavior of `publish()` differs in each state. What pattern models this cleanly?**
   - A: State pattern. Each state is a separate class implementing `DocumentState` interface with methods like `publish()`, `reject()`, `submitForReview()`. The `Document` class delegates to its current state object. State transitions happen inside the state methods. This isolates state-specific behavior and makes adding new states straightforward.

9. **Q: Your application needs to support multiple data export formats (CSV, JSON, Excel, PDF). Currently, you have a `DataExporter` with a switch statement. New formats are requested every sprint. What pattern do you refactor to?**
   - A: Strategy pattern combined with Factory. Create a `FileExporter` interface with `export(Data data, OutputStream out)`. Implementations: `CsvExporter`, `JsonExporter`, `ExcelExporter`, `PdfExporter`. A factory or registry selects the exporter based on format string. The switch statement disappears. OCP is satisfied — extend by adding classes, not modifying existing ones.

10. **Q: How would you design a request-scoped cache that caches user details during a single HTTP request to avoid N+1 database calls, but clears at the end of the request?**
    - A: Proxy pattern with request scope. Create a `UserCacheProxy` that implements `UserRepository`. It holds a `HashMap<String, User>` that checks before calling the real repository. Configure the proxy as request-scoped in Spring (`@Scope("request")`). The proxy wraps the real repository transparently — callers don't know caching exists.

---

## Interview Questions

1. **What are the three categories of Gang of Four design patterns?**
   - A: Creational (object creation mechanisms — Singleton, Factory, Builder), Structural (object composition — Adapter, Decorator, Proxy), Behavioral (object communication — Observer, Strategy, State).

2. **What is the difference between Factory Method and Abstract Factory?**
   - A: Factory Method creates a single product through inheritance — subclasses override a factory method. Abstract Factory creates families of related products through composition — a factory interface has methods for each product type.

3. **How does the Proxy pattern differ from the Decorator pattern?**
   - A: Proxy controls access to an object (lazy loading, security, caching). Decorator adds behavior to an existing object. Proxy often creates the underlying object; Decorator wraps an already-existing instance.

4. **What pattern does Spring Data JPA's Repository use?**
   - A: Template Method pattern. `JpaRepository` and `SimpleJpaRepository` define the algorithm skeleton (save, find, delete) with abstract query methods. Custom `findBy*` methods fill in the specifics via derived query generation.

5. **How does the State pattern differ from the Strategy pattern?**
   - A: State changes the object's behavior when its internal state changes — the context delegates to a state object that can transition to other states. Strategy lets the client select an algorithm at runtime — the strategy is set once and doesn't change the context's state.

6. **Give an example of the Builder pattern in the Java standard library or popular frameworks.**
   - A: `StringBuilder`, `Stream.Builder`, Lombok's `@Builder`, Spring's `UriComponentsBuilder`, and `MockMvcRequestBuilders` in Spring MVC testing.

7. **How does Spring implement the Proxy pattern for @Transactional?**
   - A: Spring creates a CGLIB or JDK Dynamic Proxy for the bean. When `@Transactional` methods are called, the proxy intercepts the call, begins a transaction, invokes the actual method, and commits/rolls back based on the outcome. The method itself is unaware of the transaction management.

8. **What is the Chain of Responsibility pattern and where is it used in Spring?**
   - A: Passes a request along a chain of handlers until one processes it. Spring Security's `SecurityFilterChain` is the prime example — each filter (authentication, CSRF, CORS, etc.) decides to process or pass to the next.

9. **What pattern would you use for a plugin architecture where plugins can be added at runtime?**
   - A: Strategy pattern for the plugin interface, Factory pattern for plugin discovery (Java `ServiceLoader`), Decorator for plugin composition, and Observer for plugin lifecycle events.

10. **How do design patterns improve testability?**
    - A: Patterns like Strategy, Observer, and DIP encourage programming to interfaces rather than concrete classes. This loose coupling makes it trivial to mock dependencies in unit tests and test components in isolation.

---

## Developer Recommendations

- **Prefer composition over inheritance** — Inheritance creates rigid hierarchy: a change to a base class ripples through all subclasses. Composition (Strategy, Decorator, Adapter) lets you assemble behavior from interchangeable components. Test components in isolation and replace them without side effects. The mantra "favor composition over inheritance" is the single most impactful design principle. A SaaS company built their notification system with deep inheritance (`BaseNotifier` → `AsyncBaseNotifier` → `EmailNotifier`, `SMSNotifier`, `PushNotifier`); when they needed to add Slack notifications that combined email and push behavior, the inheritance hierarchy collapsed and required a complete rewrite to composition-based design.

- **Use the Strategy pattern instead of switch/if-else chains for varying algorithms** — A switch on `paymentType` or `exportFormat` violates OCP — adding a new case requires modifying existing code. Strategy extracts each algorithm into its own class. Spring injection makes this seamless: inject a `List<PaymentStrategy>` and map by type. New strategies = new classes, zero modifications. A travel booking platform used a 400-line switch statement for payment processing across 15 gateways; a developer adding the 16th gateway accidentally broke the PayPal case because the switch had a fall-through bug that went unnoticed for 3 weeks in production.

- **Leverage Spring's built-in pattern implementations** — Spring already implements Proxy (`@Transactional`, `@Cacheable`), Template Method (`JdbcTemplate`, `JpaRepository`), Factory (`@Bean`, `BeanFactory`), and Observer (`@EventListener`). Don't reimplement these patterns. Instead, understand which pattern Spring uses and extend it. For custom proxies, use Spring AOP rather than generating CGLIB proxies manually.

- **Don't force patterns where simpler solutions work** — A simple `if-else` for 2-3 cases is more readable than a full Strategy pattern with interfaces, implementations, and a factory. Add patterns as complexity grows, not preemptively. YAGNI applies to design patterns too — premature abstraction adds indirection without benefit.

- **Use Builder for objects with >4 parameters, especially when many are optional** — Telescoping constructors (one for each parameter combination) are unreadable and error-prone. The Builder pattern with fluent API makes construction self-documenting. Lombok's `@Builder` eliminates boilerplate. Validation in `build()` catches configuration errors early. Examples: HTTP request builders, query specifications, configuration objects.

- **Match the pattern to the volatility point** — Identify what changes most frequently in your system and apply the appropriate pattern there. If payment methods change quarterly, use Strategy. If notification channels change, use Observer. If database access patterns change, use Template Method. Over-engineering stable code with patterns adds complexity without payoff.
- **Document the pattern intent, not just the structure** — When using a pattern, document why it was chosen and what problem it solves. A comment like "Strategy pattern for shipping cost calculation — new shipping methods implement the interface" is more valuable than just the structural code. Future maintainers need to know the design rationale, not just the pattern name.
- **Patterns are vocabulary, not a checklist** — The primary value of patterns is communication. When a developer says "let's use Strategy here," everyone understands the intent: extract algorithm into interchangeable implementations. Don't force code into a pattern structure if it doesn't fit — the problem drives the pattern choice, not vice versa.
