# Design Patterns

## 1. Executive Summary

Design patterns are reusable, proven solutions to commonly occurring problems in software design. They represent best practices evolved over time by experienced developers. Cataloged by the "Gang of Four" (GoF) in 1994, patterns are categorized into Creational, Structural, and Behavioral types. In modern backend development, these patterns remain fundamental, with many Spring Boot frameworks implementing them internally.

## 2. Core Theory

### Three Categories

**Creational Patterns:** Deal with object creation mechanisms, making the system independent of how objects are created, composed, and represented.
- Singleton, Factory Method, Abstract Factory, Builder, Prototype

**Structural Patterns:** Concern class and object composition, forming larger structures while keeping them flexible and efficient.
- Adapter, Bridge, Composite, Decorator, Facade, Flyweight, Proxy

**Behavioral Patterns:** Characterize communication between objects, distributing responsibility and managing algorithms.
- Chain of Responsibility, Command, Interpreter, Iterator, Mediator, Memento, Observer, State, Strategy, Template Method, Visitor

### Why Patterns Matter in Backend Development

- **Common vocabulary**: "We need a Strategy pattern here" communicates design intent clearly.
- **Proven solutions**: Avoid reinventing the wheel.
- **Code quality**: Patterns encourage loose coupling, high cohesion, and maintainability.
- **Framework alignment**: Spring Boot itself is built on patterns (Singleton, Proxy, Template Method, etc.).

## 3. Under-the-Hood Deep Dive

### Creational Patterns

**Singleton Pattern**
Ensures a class has only one instance and provides a global access point.

```java
// Java Singleton (Bill Pugh)
public class DatabaseConnectionPool {
    private DatabaseConnectionPool() {}

    private static class Holder {
        private static final DatabaseConnectionPool INSTANCE =
            new DatabaseConnectionPool();
    }

    public static DatabaseConnectionPool getInstance() {
        return Holder.INSTANCE;
    }
}

// Spring Singleton (default scope)
@Service
@Scope("singleton") // Default in Spring
public class UserService {
    // One instance per Spring context
}

// Thread-safe Singleton
public class ConfigManager {
    private static volatile ConfigManager instance;
    private final Properties config = new Properties();

    private ConfigManager() {
        loadConfig();
    }

    public static ConfigManager getInstance() {
        if (instance == null) {
            synchronized (ConfigManager.class) {
                if (instance == null) {
                    instance = new ConfigManager();
                }
            }
        }
        return instance;
    }
}
```

**Factory Method Pattern**
Defines an interface for creating an object but lets subclasses decide which class to instantiate.

```java
// Product interface
public interface PaymentGateway {
    PaymentResponse process(PaymentRequest request);
}

// Concrete products
public class StripeGateway implements PaymentGateway {
    public PaymentResponse process(PaymentRequest request) { /* Stripe */ }
}

public class PayPalGateway implements PaymentGateway {
    public PaymentResponse process(PaymentRequest request) { /* PayPal */ }
}

// Creator
public abstract class PaymentGatewayFactory {
    public abstract PaymentGateway createGateway();

    public PaymentResponse processPayment(PaymentRequest request) {
        PaymentGateway gateway = createGateway();
        return gateway.process(request);
    }
}

public class StripeFactory extends PaymentGatewayFactory {
    @Override
    public PaymentGateway createGateway() {
        return new StripeGateway();
    }
}
```

**Builder Pattern**
Separates the construction of a complex object from its representation.

```java
// Classic Builder
public class Order {
    private final String id;
    private final String customerId;
    private final List<OrderItem> items;
    private final BigDecimal total;
    private final String shippingAddress;
    private final String couponCode;

    private Order(Builder builder) {
        this.id = builder.id;
        this.customerId = builder.customerId;
        this.items = builder.items;
        this.total = builder.total;
        this.shippingAddress = builder.shippingAddress;
        this.couponCode = builder.couponCode;
    }

    public static class Builder {
        private String id;
        private String customerId;
        private List<OrderItem> items = new ArrayList<>();
        private BigDecimal total = BigDecimal.ZERO;
        private String shippingAddress;
        private String couponCode;

        public Builder(String customerId) {
            this.customerId = customerId;
        }

        public Builder id(String id) { this.id = id; return this; }
        public Builder addItem(OrderItem item) { this.items.add(item); return this; }
        public Builder total(BigDecimal total) { this.total = total; return this; }
        public Builder shippingAddress(String addr) { this.shippingAddress = addr; return this; }
        public Builder couponCode(String code) { this.couponCode = code; return this; }

        public Order build() {
            if (customerId == null) throw new IllegalStateException("customerId required");
            return new Order(this);
        }
    }
}

// Usage
Order order = new Order.Builder("cust-123")
    .id("order-456")
    .addItem(new OrderItem("product-1", 2))
    .shippingAddress("123 Main St")
    .build();

// Lombok @Builder
@Data
@Builder
public class UserProfile {
    private String id;
    private String name;
    private String email;
    private String phone;
    private Address address;
}
```

### Structural Patterns

**Adapter Pattern**
Allows incompatible interfaces to work together.

```java
// Target interface
public interface PaymentProvider {
    void pay(String amount, String currency);
}

// Adaptee - third-party library with different interface
public class LegacyPaymentSystem {
    public void makePayment(double amountInCents) {
        // Process in cents
    }
}

// Adapter
public class LegacyPaymentAdapter implements PaymentProvider {
    private final LegacyPaymentSystem legacySystem;

    public LegacyPaymentAdapter(LegacyPaymentSystem legacySystem) {
        this.legacySystem = legacySystem;
    }

    @Override
    public void pay(String amount, String currency) {
        double cents = Double.parseDouble(amount) * 100;
        legacySystem.makePayment(cents);
    }
}

// Usage
@Service
public class CheckoutService {
    private final PaymentProvider paymentProvider;

    public CheckoutService() {
        this.paymentProvider = new LegacyPaymentAdapter(new LegacyPaymentSystem());
    }

    public void checkout(Order order) {
        paymentProvider.pay(order.getTotal().toString(), "USD");
    }
}
```

**Decorator Pattern**
Attaches additional responsibilities to an object dynamically.

```java
// Component interface
public interface DataSource {
    void write(String data);
    String read();
}

// Concrete component
public class FileDataSource implements DataSource {
    private final String filename;

    public FileDataSource(String filename) {
        this.filename = filename;
    }

    public void write(String data) {
        Files.write(Paths.get(filename), data.getBytes());
    }

    public String read() {
        return new String(Files.readAllBytes(Paths.get(filename)));
    }
}

// Base decorator
public abstract class DataSourceDecorator implements DataSource {
    protected DataSource wrappee;

    public DataSourceDecorator(DataSource source) {
        this.wrappee = source;
    }

    public void write(String data) {
        wrappee.write(data);
    }

    public String read() {
        return wrappee.read();
    }
}

// Concrete decorators
public class EncryptionDecorator extends DataSourceDecorator {
    public EncryptionDecorator(DataSource source) { super(source); }

    @Override
    public void write(String data) {
        super.write(encrypt(data));
    }

    @Override
    public String read() {
        return decrypt(super.read());
    }

    private String encrypt(String data) { /* AES encryption */ }
    private String decrypt(String data) { /* AES decryption */ }
}

public class CompressionDecorator extends DataSourceDecorator {
    public CompressionDecorator(DataSource source) { super(source); }

    @Override
    public void write(String data) {
        super.write(compress(data));
    }

    @Override
    public String read() {
        return decompress(super.read());
    }
}

// Usage: layered decorators
DataSource source = new FileDataSource("data.txt");
source = new CompressionDecorator(source);
source = new EncryptionDecorator(source);
// Writing: encrypt -> compress -> file
// Reading: file -> decompress -> decrypt
```

**Proxy Pattern**
Provides a surrogate or placeholder for another object to control access.

```java
// Subject interface
public interface ProductService {
    Product getProduct(String id);
}

// Real subject
public class RealProductService implements ProductService {
    public Product getProduct(String id) {
        // Expensive DB call
        return database.findProduct(id);
    }
}

// Proxy - adds caching
public class CachingProductServiceProxy implements ProductService {
    private final RealProductService realService;
    private final Map<String, Product> cache = new ConcurrentHashMap<>();

    public CachingProductServiceProxy(RealProductService realService) {
        this.realService = realService;
    }

    @Override
    public Product getProduct(String id) {
        return cache.computeIfAbsent(id, realService::getProduct);
    }
}
```

### Behavioral Patterns

**Strategy Pattern**
Defines a family of algorithms, encapsulates each one, and makes them interchangeable.

```java
// Strategy interface
public interface ShippingCostStrategy {
    BigDecimal calculate(Order order);
}

// Concrete strategies
public class StandardShipping implements ShippingCostStrategy {
    public BigDecimal calculate(Order order) {
        return new BigDecimal("5.99");
    }
}

public class ExpressShipping implements ShippingCostStrategy {
    public BigDecimal calculate(Order order) {
        return new BigDecimal("14.99");
    }
}

public class FreeShipping implements ShippingCostStrategy {
    public BigDecimal calculate(Order order) {
        return order.getTotal().compareTo(new BigDecimal("50")) >= 0
            ? BigDecimal.ZERO
            : new BigDecimal("5.99");
    }
}

// Context
@Service
public class ShippingCalculator {
    private final Map<String, ShippingCostStrategy> strategies;

    public ShippingCalculator(List<ShippingCostStrategy> strategyList) {
        this.strategies = strategyList.stream()
            .collect(Collectors.toMap(
                s -> s.getClass().getSimpleName()
                    .replace("Shipping", "").toLowerCase(),
                Function.identity()
            ));
    }

    public BigDecimal calculate(String type, Order order) {
        return strategies.getOrDefault(type.toLowerCase(), new StandardShipping())
            .calculate(order);
    }
}
```

**Observer Pattern**
Defines a one-to-many dependency between objects so that when one changes state, all dependents are notified.

```java
// Observer interface
public interface OrderObserver {
    void onOrderCreated(Order order);
    void onOrderShipped(Order order);
    void onOrderCancelled(Order order);
}

// Concrete observers
@Component
public class EmailNotificationObserver implements OrderObserver {
    public void onOrderCreated(Order order) {
        sendEmail(order.getCustomerEmail(), "Order Confirmation");
    }

    public void onOrderShipped(Order order) {
        sendEmail(order.getCustomerEmail(), "Order Shipped");
    }

    public void onOrderCancelled(Order order) {
        sendEmail(order.getCustomerEmail(), "Order Cancelled");
    }
}

@Component
public class AuditLogObserver implements OrderObserver {
    public void onOrderCreated(Order order) {
        log.info("Order created: {}", order.getId());
    }
    // ... other methods
}

// Subject
@Component
public class OrderSubject {
    private final List<OrderObserver> observers;

    public OrderSubject(List<OrderObserver> observers) {
        this.observers = observers;
    }

    public void notifyOrderCreated(Order order) {
        observers.forEach(o -> o.onOrderCreated(order));
    }

    public void notifyOrderShipped(Order order) {
        observers.forEach(o -> o.onOrderShipped(order));
    }
}

// Usage in service
@Service
public class OrderService {
    private final OrderSubject subject;

    @Transactional
    public Order createOrder(CreateOrderRequest request) {
        Order order = new Order(request);
        orderRepository.save(order);
        subject.notifyOrderCreated(order);
        return order;
    }
}
```

**Template Method Pattern**
Defines the skeleton of an algorithm, deferring some steps to subclasses.

```java
// Abstract class with template method
public abstract class PaymentFlow {
    // Template method - defines algorithm skeleton
    public final PaymentResult execute(PaymentRequest request) {
        validateRequest(request);
        PaymentAuthorization auth = authorize(request);
        PaymentCapture capture = capture(auth);
        notify(request, capture);
        return buildResult(capture);
    }

    // Steps with default implementation
    protected void validateRequest(PaymentRequest request) {
        if (request.getAmount() == null || request.getAmount().signum() <= 0) {
            throw new ValidationException("Invalid amount");
        }
    }

    // Steps that subclasses must implement
    protected abstract PaymentAuthorization authorize(PaymentRequest request);
    protected abstract PaymentCapture capture(PaymentAuthorization auth);

    // Hook method - optional override
    protected void notify(PaymentRequest request, PaymentCapture capture) {
        // Default: no notification
    }

    private PaymentResult buildResult(PaymentCapture capture) {
        return new PaymentResult(capture.getId(), capture.getStatus());
    }
}

// Concrete implementation
public class CreditCardPaymentFlow extends PaymentFlow {
    @Override
    protected PaymentAuthorization authorize(PaymentRequest request) {
        // Call credit card authorization API
        return paymentGateway.authorize(request);
    }

    @Override
    protected PaymentCapture capture(PaymentAuthorization auth) {
        return paymentGateway.capture(auth);
    }

    @Override
    protected void notify(PaymentRequest request, PaymentCapture capture) {
        emailService.sendPaymentConfirmation(request.getEmail(), capture);
    }
}
```

**State Pattern**
Allows an object to alter its behavior when its internal state changes.

```java
// State interface
public interface OrderState {
    void next(Order order);
    void cancel(Order order);
    String getStatus();
}

// Concrete states
public class PendingState implements OrderState {
    public void next(Order order) {
        order.setState(new PaidState());
    }

    public void cancel(Order order) {
        order.setState(new CancelledState());
    }

    public String getStatus() { return "PENDING"; }
}

public class PaidState implements OrderState {
    public void next(Order order) {
        order.setState(new ShippedState());
    }

    public void cancel(Order order) {
        order.setState(new RefundingState());
    }

    public String getStatus() { return "PAID"; }
}

public class ShippedState implements OrderState {
    public void next(Order order) {
        order.setState(new DeliveredState());
    }

    public void cancel(Order order) {
        throw new IllegalStateException("Cannot cancel shipped order");
    }

    public String getStatus() { return "SHIPPED"; }
}

// Context
@Entity
public class Order {
    @Id
    private Long id;
    private String status;

    @Transient
    private OrderState state;

    public Order() {
        this.state = new PendingState();
        this.status = state.getStatus();
    }

    public void setState(OrderState state) {
        this.state = state;
        this.status = state.getStatus();
    }

    public void next() {
        state.next(this);
    }

    public void cancel() {
        state.cancel(this);
    }
}
```

## 4. Production Code Examples

### Spring Boot Framework Patterns

```java
// Factory Pattern in Spring: @Bean creates objects
@Configuration
public class AppConfig {
    @Bean
    public RestTemplate restTemplate() {
        return new RestTemplateBuilder()
            .setConnectTimeout(Duration.ofMillis(500))
            .build();
    }
}

// Proxy Pattern in Spring: @Transactional creates proxy
@Service
public class UserService {
    @Transactional // Spring creates a transactional proxy
    public void updateUser(User user) {
        userRepository.save(user);
    }
}

// Template Method in Spring: JdbcTemplate, JpaRepository
public interface UserRepository extends JpaRepository<User, Long> {
    // Spring Data JPA provides implementation
}

// Observer Pattern in Spring: @EventListener
@Component
public class OrderEventListener {
    @EventListener
    public void handleOrderCreated(OrderCreatedEvent event) {
        // React to event
    }
}

// Strategy Pattern in Spring: authentication providers
@Configuration
public class SecurityConfig {
    @Bean
    public AuthenticationManager authenticationManager(
            List<AuthenticationProvider> providers) {
        return new ProviderManager(providers);
    }
}
```

### Command Pattern for Request Processing

```java
// Command interface
public interface Command<R> {
    R execute();
}

// Concrete commands
public class CreateUserCommand implements Command<User> {
    private final UserRepository repository;
    private final CreateUserRequest request;

    public CreateUserCommand(UserRepository repo, CreateUserRequest req) {
        this.repository = repo;
        this.request = req;
    }

    @Override
    public User execute() {
        return repository.save(new User(request));
    }
}

// Command invoker
public class CommandInvoker {
    private final List<Command<?>> history = new ArrayList<>();

    public <R> R executeCommand(Command<R> command) {
        R result = command.execute();
        history.add(command);
        return result;
    }
}
```

## 5. Real-World Scenarios

### E-Commerce Checkout Pipeline
- **Builder**: Build Order from multiple steps (items, shipping, payment).
- **Strategy**: Different shipping cost calculations, payment methods.
- **State**: Order lifecycle (Pending -> Paid -> Shipped -> Delivered).
- **Observer**: Notify email, SMS, analytics on order changes.
- **Chain of Responsibility**: Validation pipeline (user -> items -> payment -> fraud).

### API Gateway Architecture
- **Proxy**: Gateway proxies requests to backend services.
- **Decorator**: Add authentication, rate limiting, logging to requests.
- **Chain of Responsibility**: Request filter chain.
- **Factory**: Create service-specific client adapters.

### Caching Layer
- **Proxy**: Caching proxy for expensive operations.
- **Decorator**: Layered caching (L1 Caffeine, L2 Redis).
- **Flyweight**: Share cached objects across requests.
- **Strategy**: Different eviction policies (LRU, LFU, TTL).

## 6. Performance

### Pattern Overhead

| Pattern | Overhead | When to Use |
|---------|----------|-------------|
| Singleton | None | Always (shared state) |
| Factory | Low (method call) | When creation varies |
| Builder | Low (object creation) | Complex objects |
| Proxy | Low-Medium (indirection) | Lazy loading, caching |
| Decorator | Low (wrapper chain) | Dynamic behavior |
| Observer | Low (list iteration) | Event handling |
| Strategy | Low (interface call) | Algorithm selection |
| State | Low (state delegation) | State machine |

## 7. Security

### Security-Related Pattern Usage

- **Proxy**: Authentication proxy, authorization proxy.
- **Decorator**: Add encryption/decryption to data streams.
- **Chain of Responsibility**: Security filter chain in Spring Security.
- **Factory**: Create different security providers (OAuth, SAML, LDAP).

```java
// Chain of Responsibility in Spring Security
@Configuration
@EnableWebSecurity
public class SecurityConfig {
    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) {
        return http
            .addFilterBefore(new RateLimitFilter(), BasicAuthenticationFilter.class)
            .addFilterAfter(new AuditLogFilter(), BasicAuthenticationFilter.class)
            .build();
    }
}
```

## 8. Common Mistakes

### Mistake 1: Pattern Overuse
"Hello World" doesn't need Abstract Factory, Builder, Visitor, and Mediator. Apply patterns where the problem matches the pattern's intent.

### Mistake 2: Singleton Abuse
Singletons can cause hidden dependencies and testability issues. Spring manages singletons for you.

### Mistake 3: Misapplying Inheritance
Using inheritance when composition is more appropriate. Prefer Strategy over subclassing.

### Mistake 4: Pattern Rigidity
Patterns are guidelines, not rules. Adapt the pattern to your context.

### Mistake 5: Implementing Patterns From Scratch
Spring Boot already implements many patterns (Proxy, Template, Singleton, Factory). Leverage the framework.

## 9. Senior Engineer Perspective

### Patterns in Modern Backend Development

**Patterns Everywhere in Spring Boot:**
- Singleton: Beans (default scope)
- Proxy: @Transactional, @Cacheable, AOP
- Template Method: JdbcTemplate, JmsTemplate, RestTemplate
- Factory: BeanFactory, @Bean methods
- Observer: @EventListener, ApplicationListener
- Chain of Responsibility: filter chains, HandlerInterceptor
- Strategy: authentication providers, message converters

### Patterns in Distributed Systems
- **Circuit Breaker**: Resilience pattern (Resilience4j).
- **Saga**: Distributed transaction pattern.
- **CQRS**: Command/query separation pattern.
- **Event Sourcing**: State change as event stream.
- **Bulkhead**: Resource isolation pattern.
- **Sidecar**: Deployment pattern (service mesh).

### When NOT to Use a Pattern
- The solution is simpler without the pattern.
- The pattern adds overhead without benefit.
- The team doesn't understand the pattern.
- The framework already handles the concern.

## 10. Interview Questions (20: 10 easy + 10 medium)

### Easy

1. **Q:** What are design patterns?
   **A:** Reusable solutions to common software design problems. They are templates for how to solve problems, not finished code.

2. **Q:** Who wrote the original design patterns catalog?
   **A:** The "Gang of Four" (GoF): Erich Gamma, Richard Helm, Ralph Johnson, John Vlissides.

3. **Q:** What are the three categories of design patterns?
   **A:** Creational, Structural, and Behavioral.

4. **Q:** What is the Singleton pattern?
   **A:** Ensures a class has only one instance and provides a global point of access to it.

5. **Q:** What is the Factory pattern?
   **A:** Defines an interface for creating objects but lets subclasses decide which class to instantiate.

6. **Q:** What is the Observer pattern?
   **A:** A one-to-many dependency where when one object changes state, all its dependents are notified.

7. **Q:** What is the Strategy pattern?
   **A:** Defines a family of interchangeable algorithms, encapsulating each one and making them interchangeable.

8. **Q:** What is the Decorator pattern?
   **A:** Attaches additional responsibilities to an object dynamically. Provides a flexible alternative to subclassing.

9. **Q:** What is the Adapter pattern?
   **A:** Allows incompatible interfaces to work together by converting one interface to another.

10. **Q:** Give an example of Singleton in Spring Boot.
    **A:** All Spring beans are singletons by default (@Scope("singleton")).

### Medium

11. **Q:** Explain the difference between Factory Method and Abstract Factory.
    **A:** Factory Method creates one product via inheritance. Abstract Factory creates families of related products via composition.

12. **Q:** How does the Proxy pattern differ from the Decorator pattern?
    **A:** Proxy controls access to an object (lazy loading, security). Decorator adds behavior to an object. Proxy creates the object; Decorator wraps an existing object.

13. **Q:** What pattern does Spring Data JPA's Repository use?
    **A:** Template Method pattern (the framework provides the algorithm skeleton; custom queries fill in the details).

14. **Q:** How does the State pattern differ from the Strategy pattern?
    **A:** State changes the object's behavior when its internal state changes (state transitions). Strategy lets the client select an algorithm at runtime.

15. **Q:** What is the Chain of Responsibility pattern?
    **A:** Passes a request along a chain of handlers. Each handler decides to process the request or pass it to the next handler.

16. **Q:** Give an example of the Builder pattern in Java standard library.
    **A:** StringBuilder, StringBuffer, Stream.Builder, Optional.IntBuilder.

17. **Q:** How does Spring implement the Proxy pattern?
    **A:** Using CGLIB or JDK dynamic proxies. @Transactional, @Cacheable create proxy objects that intercept method calls.

18. **Q:** What is the difference between Composition and Inheritance?
    **A:** Composition: has-a relationship (object contains another object). Inheritance: is-a relationship (class extends another class). Favor composition over inheritance.

19. **Q:** What pattern does the Command pattern represent?
    **A:** Encapsulates a request as an object, parameterizing clients with queues, requests, and operations.

20. **Q:** How do design patterns improve testability?
    **A:** Patterns like Strategy, Observer, and DIP encourage loose coupling, making it easier to mock dependencies and test in isolation.

## 11. Advanced Interview Questions (20: 10 hard + 10 system design)

### Hard

1. **Q:** Implement a thread-safe Singleton in Java without synchronization overhead for reads.
    **A:** Using Bill Pugh initialization holder pattern or enum Singleton.
    ```java
    public enum DataSource {
        INSTANCE;
        public Connection getConnection() { ... }
    }
    ```

2. **Q:** How would you implement a Flyweight pattern for an e-commerce product catalog?
    **A:** Share immutable product metadata (name, description, specs) across all product instances. Vary only mutable state (price, inventory) per product.

3. **Q:** Design a validation framework using Chain of Responsibility.
    **A:** Validator interface with `validate(T input) throws ValidationException`. Each validator checks one rule and passes to next. CompositeValidator chains validators.

4. **Q:** What pattern would you use for plugin architecture?
    **A:** Strategy pattern for plugin interface. Factory pattern for plugin discovery (ServiceLoader). Decorator for plugin composition. Observer for plugin events.

5. **Q:** How does Spring's @Async use the Proxy pattern?
    **A:** Spring creates a CGLIB proxy. When you call @Async method, the proxy submits the call to a TaskExecutor and returns immediately.

6. **Q:** Implement a Mediator pattern for a chat room.
    **A:** Mediator interface: `sendMessage(User, String)`. ChatRoom (mediator) receives messages and broadcasts to all users. Users don't know about each other.

7. **Q:** How would you refactor a God class using patterns?
    **A:** Extract: Strategy (for algorithms), Command (for operations), Observer (for notifications), Facade (for complex subsystems), State (for state-dependent behavior).

8. **Q:** Compare Visitor pattern with Pattern Matching in Java 17+.
    **A:** Visitor separates algorithm from object structure (GoF). Pattern matching with sealed classes provides similar functionality with less boilerplate.

9. **Q:** Design a retry mechanism using the Decorator pattern.
    **A:** RetryDecorator wraps any service. On failure, retries N times with backoff. Can wrap any interface implementation.
    ```java
    public class RetryDecorator<T> implements T {
        private final T delegate;
        private final int maxRetries;
        // Proxy method call with retry logic
    }
    ```

10. **Q:** What is the Memento pattern and where is it useful?
    **A:** Captures and externalizes an object's internal state without violating encapsulation. Useful for undo/redo, checkpoints in long-running processes, snapshots.

### System Design

11. **Q:** Design a notification system using the Observer pattern.
    **A:** Subject: OrderService. Observers: EmailNotifier, SMSNotifier, PushNotifier, AnalyticsTracker. On order creation, subject notifies all observers.

12. **Q:** Design a flexible tax calculation system using patterns.
    **A:** Strategy pattern per tax type (VAT, GST, SalesTax). Decorator for compound tax calculation. Factory for creating tax strategies per region.

13. **Q:** Design a document processing pipeline using patterns.
    **A:** Chain of Responsibility for document validation. Strategy for different output formats (PDF, HTML, DOCX). Decorator for adding headers, footers, watermarks.

14. **Q:** Design a caching system using the Proxy and Strategy patterns.
    **A:** CachingProxy wraps data access. Strategy for cache eviction (LRU, LFU, TTL). Decorator for multi-level cache (L1 local, L2 distributed).

15. **Q:** Design an authentication system using patterns.
    **A:** Strategy for authentication methods (OAuth, JWT, Basic). Chain of Responsibility for auth filters. Factory for creating auth providers. Proxy for securing resource access.

16. **Q:** Design a workflow engine using the State and Strategy patterns.
    **A:** State pattern for workflow state (Pending, Active, Completed, Failed). Strategy for task execution. Command for workflow actions.

17. **Q:** Design an API rate limiter using patterns.
    **A:** Strategy for rate limiting algorithms (Token Bucket, Leaky Bucket, Sliding Window). Proxy pattern wraps API calls with rate limit check. Decorator for adding rate limiting to any service.

18. **Q:** Design a data export service using Template Method.
    **A:** AbstractExport defines skeleton: query data, transform, format, write. Subclasses: CsvExport, JsonExport, ExcelExport. Hook methods for format-specific processing.

19. **Q:** Design an event sourcing system using patterns.
    **A:** Command pattern for commands. Event pattern for events. Observer for event handlers. Builder for event reconstruction. Memento for aggregate snapshots.

20. **Q:** Design a multi-tenant SaaS platform using patterns.
    **A:** Strategy for tenant isolation strategies. Factory for creating tenant-specific services. Proxy for tenant context resolution. Decorator for adding tenant filtering to queries.

## 12. Expert-Level Interview Questions (10: architect-level)

1. **Q:** Design a pattern-based framework for microservice orchestration.
    **A:** Command pattern for saga steps. Observer/Event for choreography. State pattern for saga state machine. Strategy for compensation strategies. Builder for saga definition.

2. **Q:** How do design patterns map to cloud-native patterns?
    **A:** Singleton -> Kubernetes Pod (one instance). Factory -> Helm Charts. Proxy -> Service Mesh sidecar. Observer -> Event-driven with Kafka. Strategy -> Feature flags. Decorator -> Envoy filters.

3. **Q:** Analyze Spring Framework's use of patterns and how they interact.
    **A:** BeanFactory (Factory, Singleton). AOP (Proxy, Decorator). Events (Observer). Transaction (Template Method, Proxy). MVC (Front Controller, View Helper). Security (Chain of Responsibility). The patterns compose: @Transactional uses Proxy, which uses Template Method for transaction management.

4. **Q:** Design a pattern-based refactoring strategy for a legacy monolith.
    **A:** Phase 1: Extract interfaces (Strategy, DIP). Phase 2: Add caching (Proxy, Decorator). Phase 3: Extract modules (Facade). Phase 4: Add event hooks (Observer). Phase 5: Strangler Fig (Proxy for migration).

5. **Q:** How do design patterns relate to reactive programming?
    **A:** Observer pattern is fundamental to reactive streams. Strategy pattern for backpressure strategies. Decorator for reactive operators (map, filter, flatMap). Factory for creating publishers.

6. **Q:** What patterns are anti-patterns in microservices?
    **A:** Singleton (centralized state doesn't scale). In-process Observer becomes distributed event bus. State pattern becomes Saga. Proxy becomes API Gateway.

7. **Q:** Design a pattern-based approach to adding observability to an existing system.
    **A:** Decorator pattern wraps services with logging/metrics/tracing. Proxy for auto-instrumentation. Observer for event-driven monitoring. Strategy for different monitoring backends.

8. **Q:** How do functional programming concepts replace traditional patterns?
    **A:** Strategy -> Higher-order functions. Command -> Lambda/Function. Observer -> Reactive streams. Template Method -> Function composition. Factory -> Supplier. Builder -> Currying.

9. **Q:** Design a pattern catalog for a specific domain (e-commerce).
    **A:** Creational: ProductBuilder, PaymentGatewayFactory. Structural: InventoryProxy, PricingDecorator. Behavioral: ShippingStrategy, OrderState, CartObserver, DiscountStrategy, CheckoutCommand.

10. **Q:** How do you document and govern design pattern usage across a large organization?
    **A:** Architecture Decision Records (ADRs). Pattern library with examples. Code review checklist for pattern usage. Automated checks (ArchUnit). Training and office hours.

## 13. Debugging & Troubleshooting

### Pattern-Related Issues

**Singleton Issues:** Hidden global state, thread safety, test difficulties.
**Proxy Issues:** Stack traces showing proxy lines, debugger skipping proxy.
**Decorator Issues:** Deep wrapper chains hard to debug.
**Observer Issues:** Unexpected notification order, memory leaks (unregistered observers).
**State Issues:** Missing state transitions, invalid states.

### Debugging Patterns in Spring
```yaml
# Enable proxy debugging
logging:
  level:
    org.springframework.aop: DEBUG
    org.springframework.cglib: DEBUG

# See proxy details in health endpoint
management:
  endpoints:
    web:
      exposure:
        include: beans, caches, beans
```

## 14. Comparison Section

### GoF Patterns vs Modern Alternatives

| GoF Pattern | Modern Alternative |
|-------------|-------------------|
| Singleton | DI container (Spring manages scope) |
| Factory | @Bean, Supplier<T>, constructors |
| Builder | Lombok @Builder, Kotlin data classes |
| Observer | @EventListener, Reactive Streams |
| Strategy | Lambda expressions, method references |
| Command | Runnable, Callable, Supplier |
| Iterator | Stream API, for-each loop |
| Template Method | Functional interface composition |
| Proxy | AOP, @Transactional, service mesh |
| Adapter | Functional interfaces, method references |

### Pattern Category Summary

| Category | Focus | Key Patterns |
|----------|-------|--------------|
| Creational | Object creation | Singleton, Factory, Builder |
| Structural | Object composition | Adapter, Decorator, Proxy |
| Behavioral | Object communication | Observer, Strategy, State |

## 15. Revision Notes

### Quick Recap
- **Singleton**: One instance (Spring default scope).
- **Factory**: Delegate object creation to subclasses.
- **Builder**: Step-by-step complex object construction.
- **Adapter**: Make incompatible interfaces work together.
- **Decorator**: Add behavior dynamically (wrapper).
- **Proxy**: Control access to an object.
- **Observer**: Event notification to multiple listeners.
- **Strategy**: Interchangeable algorithms.
- **Template Method**: Algorithm skeleton with overridable steps.
- **State**: Object behavior changes with its state.

### Framework Mappings
- Spring Beans: Singleton, Factory, Proxy
- Spring Data: Template Method, Proxy
- Spring AOP: Proxy, Decorator
- Spring Events: Observer
- Spring Security: Chain of Responsibility, Strategy

## 16. Cheat Sheet

```
+-------------------------------------------------------------------+
|                    DESIGN PATTERNS CHEAT SHEET                     |
+-------------------------------------------------------------------+
| CREATIONAL    | STRUCTURAL     | BEHAVIORAL                        |
+---------------+----------------+-----------------------------------+
| Singleton     | Adapter        | Chain of Responsibility           |
| Factory       | Bridge         | Command                           |
| Abstract      | Composite      | Interpreter                       |
|   Factory     | Decorator      | Iterator                          |
| Builder       | Facade         | Mediator                          |
| Prototype     | Flyweight      | Memento                           |
|               | Proxy          | Observer                          |
|               |                | State                             |
|               |                | Strategy                          |
|               |                | Template Method                   |
|               |                | Visitor                           |
+---------------+----------------+-----------------------------------+
| COMMON SPRING BOOT PATTERNS                                        |
+-------------------------------------------------------------------+
| Singleton     | @Bean, @Service, @Component (default scope)        |
| Factory       | @Bean methods, ApplicationContext.getBean()        |
| Proxy         | @Transactional, @Cacheable, @Async                |
| Template      | JdbcTemplate, RestTemplate, JmsTemplate           |
| Method        | JpaRepository, MongoRepository, etc.              |
| Observer      | @EventListener, ApplicationListener, Application- |
|               | EventPublisher                                    |
| Chain of      | SecurityFilterChain, HandlerInterceptor            |
| Responsibility|                                                    |
| Strategy      | AuthenticationProvider, MessageConverter          |
| Decorator     | HttpMessageConverter customization                |
| Adapter       | WebMvcConfigurer, HandlerAdapter                  |
+-------------------------------------------------------------------+
| WHEN TO USE                                                       |
+-------------------------------------------------------------------+
| Pattern       | Use When                                           |
+---------------+----------------------------------------------------+
| Singleton     | Exactly one instance needed (caches, factories)    |
| Factory       | Object creation varies by type/configuration       |
| Builder       | Object has many optional parameters               |
| Adapter       | Existing class has wrong interface                 |
| Decorator     | Need to add responsibilities dynamically           |
| Proxy         | Need to control access to an object               |
| Observer      | Multiple objects need to react to events           |
| Strategy      | Multiple interchangeable algorithms exist          |
| Template      | Algorithm steps vary but structure is fixed        |
| State         | Object behavior depends on internal state          |
+-------------------------------------------------------------------+
```
