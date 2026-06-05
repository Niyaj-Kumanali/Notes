# SOLID Principles

## 1. Executive Summary

SOLID is a mnemonic acronym for five design principles intended to make object-oriented designs more understandable, flexible, and maintainable. Introduced by Robert C. Martin, these principles guide developers to create systems that are easy to extend, refactor, and test. While originally conceived for OOP, SOLID principles apply equally well to modern backend architectures including microservices and event-driven systems.

## 2. Core Theory

### The Five Principles

**S - Single Responsibility Principle (SRP)**
A class should have one, and only one, reason to change. Each class or module should be responsible for a single part of the system's functionality.

**O - Open/Closed Principle (OCP)**
Software entities should be open for extension but closed for modification. You should be able to add new functionality without changing existing code.

**L - Liskov Substitution Principle (LSP)**
Derived classes must be substitutable for their base classes. Subtypes should behave in a way that does not break the program when used in place of their parent types.

**I - Interface Segregation Principle (ISP)**
Clients should not be forced to depend on interfaces they do not use. Many specific interfaces are better than one general-purpose interface.

**D - Dependency Inversion Principle (DIP)**
High-level modules should not depend on low-level modules. Both should depend on abstractions. Abstractions should not depend on details. Details should depend on abstractions.

## 3. Under-the-Hood Deep Dive

### Single Responsibility Principle

SRP states: "A class should have only one reason to change." This does not mean a class can do only one thing; it means a class should encapsulate one cohesive responsibility.

**Violation:**
```java
// Violates SRP - has multiple responsibilities
public class OrderService {
    public void createOrder(Order order) { /* order creation */ }
    public void sendEmail(Order order) { /* email logic */ }
    public void generateInvoice(Order order) { /* invoice generation */ }
    public void saveToDatabase(Order order) { /* persistence */ }
}
```

**Compliance:**
```java
// Each class has one responsibility
@Service
public class OrderService {
    private final OrderRepository repository;
    private final EmailService emailService;
    private final InvoiceService invoiceService;

    @Transactional
    public Order createOrder(CreateOrderRequest request) {
        Order order = new Order(request);
        repository.save(order);
        emailService.sendOrderConfirmation(order);
        invoiceService.generateInvoice(order);
        return order;
    }
}

@Service
public class EmailService {
    public void sendOrderConfirmation(Order order) { /* email logic */ }
}

@Service
public class InvoiceService {
    public Invoice generateInvoice(Order order) { /* invoice logic */ }
}
```

### Open/Closed Principle

OCP is achieved through abstraction (interfaces, abstract classes) and polymorphism.

**Violation:**
```java
// Violates OCP - adding new payment method requires modifying this class
public class PaymentProcessor {
    public void process(String type, Payment payment) {
        if (type.equals("CREDIT_CARD")) {
            processCreditCard(payment);
        } else if (type.equals("PAYPAL")) {
            processPayPal(payment);
        }
        // Must modify this class to add new payment types
    }
}
```

**Compliance:**
```java
// Complies with OCP - new payment methods extend, not modify
public interface PaymentMethod {
    boolean process(Payment payment);
}

@Component
public class CreditCardPayment implements PaymentMethod {
    @Override
    public boolean process(Payment payment) {
        // Credit card processing logic
        return true;
    }
}

@Component
public class PayPalPayment implements PaymentMethod {
    @Override
    public boolean process(Payment payment) {
        // PayPal processing logic
        return true;
    }
}

@Service
public class PaymentProcessor {
    private final Map<String, PaymentMethod> paymentMethods;

    public PaymentProcessor(List<PaymentMethod> methods) {
        this.paymentMethods = methods.stream()
            .collect(Collectors.toMap(
                m -> m.getClass().getSimpleName().replace("Payment", "").toUpperCase(),
                Function.identity()
            ));
    }

    public boolean process(String type, Payment payment) {
        PaymentMethod method = paymentMethods.get(type.toUpperCase());
        if (method == null) {
            throw new UnsupportedPaymentException(type);
        }
        return method.process(payment);
    }
}

// Adding a new payment method doesn't modify existing code
@Component
public class ApplePayPayment implements PaymentMethod {
    @Override
    public boolean process(Payment payment) {
        // Apple Pay processing logic
        return true;
    }
}
```

### Liskov Substitution Principle

LSP ensures that inheritance is used correctly. Subtypes must satisfy the behavioral contract of their base type.

**Violation:**
```java
// Violates LSP - Square is not substitutable for Rectangle
public class Rectangle {
    protected int width;
    protected int height;

    public void setWidth(int width) { this.width = width; }
    public void setHeight(int height) { this.height = height; }
    public int getArea() { return width * height; }
}

public class Square extends Rectangle {
    @Override
    public void setWidth(int width) {
        super.setWidth(width);
        super.setHeight(width); // Side effect!
    }

    @Override
    public void setHeight(int height) {
        super.setWidth(height); // Side effect!
        super.setHeight(height);
    }
}

// Client code that breaks with Square
public void resize(Rectangle rect) {
    rect.setWidth(5);
    rect.setHeight(10);
    assert rect.getArea() == 50; // Fails for Square (gets 100)
}
```

**Compliance:**
```java
// Better design: use abstraction without inheritance violation
public interface Shape {
    int getArea();
}

public class Rectangle implements Shape {
    private int width;
    private int height;

    public Rectangle(int width, int height) {
        this.width = width;
        this.height = height;
    }

    @Override
    public int getArea() { return width * height; }
}

public class Square implements Shape {
    private int side;

    public Square(int side) {
        this.side = side;
    }

    @Override
    public int getArea() { return side * side; }
}
```

### Interface Segregation Principle

**Violation:**
```java
// Fat interface - forces clients to depend on methods they don't use
public interface Worker {
    void work();
    void eat();
    void sleep();
}

public class HumanWorker implements Worker {
    public void work() { /* working */ }
    public void eat() { /* eating */ }
    public void sleep() { /* sleeping */ }
}

public class RobotWorker implements Worker {
    public void work() { /* working */ }
    public void eat() { throw new UnsupportedOperationException(); }
    public void sleep() { throw new UnsupportedOperationException(); }
}
```

**Compliance:**
```java
// Segregated interfaces
public interface Workable {
    void work();
}

public interface Eatable {
    void eat();
}

public interface Sleepable {
    void sleep();
}

public class HumanWorker implements Workable, Eatable, Sleepable {
    public void work() { }
    public void eat() { }
    public void sleep() { }
}

public class RobotWorker implements Workable {
    public void work() { }
}
```

### Dependency Inversion Principle

**Violation:**
```java
// Violates DIP - high level depends on low level
public class OrderService {
    private MySQLDatabase database; // Concrete class dependency

    public OrderService() {
        this.database = new MySQLDatabase(); // Tight coupling
    }
}
```

**Compliance:**
```java
// Complies with DIP - both depend on abstraction
public interface OrderRepository {
    Order save(Order order);
    Optional<Order> findById(Long id);
}

@Service
public class OrderService {
    private final OrderRepository repository; // Depends on abstraction

    public OrderService(OrderRepository repository) { // Injection
        this.repository = repository;
    }
}

@Repository
public class JpaOrderRepository implements OrderRepository {
    // Implementation depends on abstraction
}

@Repository
public class MongoOrderRepository implements OrderRepository {
    // Alternative implementation
}
```

## 4. Production Code Examples

### SRP in Spring Boot Controllers

```java
// Controller: only handles HTTP concerns
@RestController
@RequestMapping("/api/users")
public class UserController {

    private final UserService userService;
    private final UserAssembler userAssembler;

    @GetMapping("/{id}")
    public ResponseEntity<UserResponse> getUser(@PathVariable Long id) {
        User user = userService.findById(id);
        return ResponseEntity.ok(userAssembler.toResponse(user));
    }
}

// Service: business logic
@Service
public class UserService {
    private final UserRepository repository;
    private final ValidationService validator;

    @Transactional
    public User createUser(CreateUserRequest request) {
        validator.validate(request);
        User user = new User(request);
        return repository.save(user);
    }
}

// Assembler: DTO conversion
@Component
public class UserAssembler {
    public UserResponse toResponse(User user) {
        return UserResponse.builder()
            .id(user.getId())
            .name(user.getName())
            .email(user.getEmail())
            .build();
    }
}
```

### OCP with Strategy Pattern

```java
// OCP-compliant discount calculation
public interface DiscountStrategy {
    BigDecimal calculate(Order order);
    boolean applies(Order order);
}

@Component
public class NoDiscount implements DiscountStrategy {
    @Override
    public BigDecimal calculate(Order order) {
        return BigDecimal.ZERO;
    }

    @Override
    public boolean applies(Order order) {
        return true; // Default
    }
}

@Component
public class LoyaltyDiscount implements DiscountStrategy {
    @Override
    public BigDecimal calculate(Order order) {
        return order.getTotal().multiply(new BigDecimal("0.10"));
    }

    @Override
    public boolean applies(Order order) {
        return order.getCustomer().getAgeInDays() > 365;
    }
}

@Component
public class BulkDiscount implements DiscountStrategy {
    @Override
    public BigDecimal calculate(Order order) {
        return order.getTotal().multiply(new BigDecimal("0.15"));
    }

    @Override
    public boolean applies(Order order) {
        return order.getItemCount() >= 10;
    }
}

@Service
public class DiscountCalculator {
    private final List<DiscountStrategy> strategies;

    public DiscountCalculator(List<DiscountStrategy> strategies) {
        this.strategies = strategies;
    }

    public BigDecimal calculate(Order order) {
        return strategies.stream()
            .filter(s -> s.applies(order))
            .map(s -> s.calculate(order))
            .reduce(BigDecimal.ZERO, BigDecimal::add);
    }
}
```

### ISP in Interface Design

```java
// Good: segregated interfaces for different concerns
public interface ReadableRepository<T, ID> {
    Optional<T> findById(ID id);
    List<T> findAll();
    long count();
}

public interface WriteableRepository<T, ID> {
    T save(T entity);
    void deleteById(ID id);
    void delete(T entity);
}

public interface AuditableRepository<T, ID> {
    Optional<T> findWithAuditLog(ID id);
    List<AuditEntry> getAuditHistory(ID id);
}

// Implement only what's needed
@Repository
public interface ProductRepository extends ReadableRepository<Product, Long>,
        WriteableRepository<Product, Long> {
    // Product-specific queries
}

@Repository
public interface ReportRepository extends ReadableRepository<Report, Long> {
    // Read-only, no write methods needed
}
```

### DIP with Dependency Injection

```java
// High-level module
@Service
public class OrderConfirmationService {

    private final NotificationSender notificationSender;

    public OrderConfirmationService(NotificationSender notificationSender) {
        this.notificationSender = notificationSender;
    }

    public void confirmOrder(Order order) {
        // business logic
        notificationSender.send(new OrderConfirmation(order));
    }
}

// Abstraction
public interface NotificationSender {
    void send(Notification notification);
}

// Low-level implementations
@Component
@ConditionalOnProperty(name = "notification.type", havingValue = "email")
public class EmailNotificationSender implements NotificationSender {
    public void send(Notification notification) {
        // Send email
    }
}

@Component
@ConditionalOnProperty(name = "notification.type", havingValue = "sms")
public class SmsNotificationSender implements NotificationSender {
    public void send(Notification notification) {
        // Send SMS
    }
}
```

## 5. Real-World Scenarios

### Refactoring Legacy Code with SOLID

1. **SRP**: Extract email, invoice, and logging from OrderService.
2. **OCP**: Replace if-else validation with strategy pattern.
3. **LSP**: Replace inheritance hierarchies with interfaces.
4. **ISP**: Split fat repository interfaces into read/write/audit.
5. **DIP**: Introduce repository abstractions and dependency injection.

### Microservice Design with SOLID

- **SRP**: Each microservice has one business capability.
- **OCP**: Add features via new services, not by modifying existing ones.
- **LSP**: APIs are substitutable (versioning, backward compatibility).
- **ISP**: Service interfaces match client needs (BFF pattern).
- **DIP**: Services depend on message abstractions (events), not concrete services.

## 6. Performance

### SOLID and Performance Trade-offs

| Principle | Potential Overhead | Benefit |
|-----------|-------------------|---------|
| SRP | More classes, more indirection | Easier to optimize specific concerns |
| OCP | Abstraction overhead | No modification cost for new features |
| ISP | More interfaces | Smaller, more focused implementations |
| DIP | Runtime resolution cost | Swap implementations without code changes |

### Guidelines
- Don't over-engineer: apply SOLID where complexity exists.
- Use DI frameworks (Spring) to minimize DIP overhead.
- Profile before optimizing: abstraction overhead is usually negligible.

## 7. Security

### SOLID for Security
- **SRP**: Keep authentication, authorization, and audit in separate classes.
- **DIP**: Security services depend on abstractions, allowing pluggable security providers.
- **ISP**: Security interfaces should expose only required methods (avoid broad SecurityManager).

```java
// SRP: Separate security concerns
@Component
public class AuthenticationService { }

@Component
public class AuthorizationService { }

@Component
public class AuditService { }
```

## 8. Common Mistakes

### Mistake 1: Over-Engineering with SOLID
Applying all principles everywhere creates unnecessary complexity. Apply where change is expected.

### Mistake 2: SRP Taken Too Far
A class with one method isn't necessarily SRP. Cohesion matters: related behaviors can be in one class.

### Mistake 3: OCP via Inheritance Instead of Composition
```java
// WRONG: OCP via deep inheritance
class DiscountedOrder extends Order { }
class SeasonalDiscountedOrder extends DiscountedOrder { }

// RIGHT: OCP via composition + strategy
class Order {
    private DiscountStrategy discountStrategy;
}
```

### Mistake 4: LSP Violation Through Collection Types
```java
// LSP violation: List<String> is not substitutable for List<Object>
List<Object> objects = new ArrayList<String>(); // Compile error in Java
```

### Mistake 5: DIP Leading to Yo-Yo Problem
Excessive abstraction layers make code hard to follow. Use the right level of abstraction.

## 9. Senior Engineer Perspective

### SOLID as a Compass, Not a Rulebook
SOLID principles are guidelines that reduce maintenance cost. Apply pragmatically:
- If a class has one reason to change, it's SRP.
- If adding a feature requires modifying existing code, it violates OCP.
- If a subclass breaks parent behavior, it violates LSP.
- If a client depends on methods it doesn't use, ISP violation.
- If a high-level module depends on low-level concrete classes, DIP violation.

### SOLID in Microservices
- **SRP**: Service per bounded context.
- **OCP**: Extend via new services, events, plugins.
- **LSP**: Service API backward compatibility.
- **ISP**: BFF per client type.
- **DIP**: Events as abstractions, service implementations as concretions.

### SOLID Testing Benefits
- SRP: Easy to unit test (few dependencies).
- OCP: Test new features without retesting existing.
- DIP: Mock abstractions for isolated tests.

## 10. Interview Questions (20: 10 easy + 10 medium)

### Easy

1. **Q:** What does SOLID stand for?
   **A:** Single Responsibility, Open-Closed, Liskov Substitution, Interface Segregation, Dependency Inversion.

2. **Q:** What is the Single Responsibility Principle?
   **A:** A class should have only one reason to change, meaning it should have only one responsibility.

3. **Q:** What is the Open/Closed Principle?
   **A:** Classes should be open for extension but closed for modification.

4. **Q:** What is the Liskov Substitution Principle?
   **A:** Derived classes must be substitutable for their base classes without altering the correctness of the program.

5. **Q:** What is Interface Segregation Principle?
   **A:** Clients should not be forced to depend on interfaces they do not use.

6. **Q:** What is Dependency Inversion Principle?
   **A:** Depend on abstractions, not concretions. High-level modules should not depend on low-level modules.

7. **Q:** Which principle is violated by a "fat interface" with many methods?
   **A:** Interface Segregation Principle (ISP).

8. **Q:** Which principle suggests using interfaces/abstract classes rather than concrete classes?
   **A:** Dependency Inversion Principle (DIP).

9. **Q:** Does SRP mean a class should only have one method?
   **A:** No. It means the class should have one responsibility (coherent set of behaviors).

10. **Q:** How does Dependency Injection relate to DIP?
    **A:** DI is a technique that implements DIP by injecting dependencies through constructors, setters, or interfaces.

### Medium

11. **Q:** Give an example of LSP violation using collections.
    **A:** Passing `ArrayList` where `List` is expected is fine. But creating a subclass that throws `UnsupportedOperationException` for methods defined in the parent violates LSP.

12. **Q:** How does SRP relate to microservices?
    **A:** Each microservice should have one business capability (SRP at service level).

13. **Q:** Explain how the Strategy pattern supports OCP.
    **A:** Strategy pattern allows adding new algorithms (strategies) without modifying the context class that uses them.

14. **Q:** How do you refactor a class that violates SRP?
    **A:** Identify the different responsibilities and extract each into its own class.

15. **Q:** What is the difference between DIP and DI?
    **A:** DIP is a principle (depend on abstractions). DI is a pattern to achieve DIP (injecting dependencies).

16. **Q:** How does ISP improve code maintainability?
    **A:** Smaller interfaces mean changes affect fewer clients, and implementations don't need empty method bodies.

17. **Q:** Can you violate OCP while still using inheritance?
    **A:** Yes. If you modify a base class to add new functionality, you violate OCP. Extension should be through new subclasses.

18. **Q:** What is the relationship between LSP and polymorphism?
    **A:** LSP defines the correct use of polymorphism: subtypes must behave correctly when used in place of their parent types.

19. **Q:** How do you detect a LSP violation?
    **A:** If substituting a derived class causes unexpected behavior, throws new exceptions, or changes method semantics, LSP is violated.

20. **Q:** Which SOLID principle helps with unit testing?
    **A:** DIP - depending on abstractions allows mocking and isolation. SRP - smaller classes are easier to test.

## 11. Advanced Interview Questions (20: 10 hard + 10 system design)

### Hard

1. **Q:** Explain how the Template Method pattern supports OCP while violating DIP.
    **A:** Template Method defines the skeleton in a base class (violating DIP if high-level depends on base class). However, subclasses fill in details. It supports OCP (extend via subclass) but can violate DIP if base class contains high-level logic.

2. **Q:** How does functional programming relate to SOLID principles?
    **A:** FP achieves SRP naturally (small functions). OCP via higher-order functions. ISP via small, focused function signatures. DIP via dependency injection in closures.

3. **Q:** Design a validation framework that follows OCP.
    **A:** Validator interface with `validate(T input)`. CompositeValidator aggregates validators. New validators implement interface, added to composite via configuration without modifying existing code.

4. **Q:** How do you handle cross-cutting concerns (logging, metrics) without violating SRP?
    **A:** Use AOP (aspect-oriented programming) with Spring @Aspect. Logging and metrics aspects are separate from business logic.

5. **Q:** Explain a scenario where following SRP could lead to OCP violation.
    **A:** If you split a class into many small ones (SRP), but new requirements require modifying multiple small classes (OCP violation). Balance is needed.

6. **Q:** How does Spring's @Autowired support DIP?
    **A:** @Autowired injects dependencies through constructor/setter, allowing the consuming class to depend on interfaces rather than concrete implementations.

7. **Q:** What is the role of the Adapter pattern in supporting OCP?
    **A:** Adapter allows integrating new systems without modifying existing code (client expects interface, adapter converts from new system).

8. **Q:** How do you test for LSP compliance?
    **A:** Write a base test suite that runs against both base class and derived classes. All tests should pass for all implementations.

9. **Q:** Explain how the Decorator pattern supports OCP.
    **A:** Decorator wraps an object and adds behavior without modifying the original class. New decorators can be added without changing existing code.

10. **Q:** How does SOLID apply to database schema design?
    **A:** SRP: each table represents one entity. OCP: add columns (extend) without breaking existing queries (if backward compatible). ISP: views expose only needed columns. DIP: application depends on views/repositories, not direct table access.

### System Design

11. **Q:** Design a notification system using SOLID principles.
    **A:** SRP: separate services for email, SMS, push. OCP: new notification types via interface. ISP: NotificationSender interface with send method. DIP: NotificationService depends on NotificationSender interface.

12. **Q:** Design a plugin system following OCP.
    **A:** Plugin interface loaded via ServiceLoader or Spring's plugin mechanism. Host application discovers and invokes plugins without code changes. New plugins added by dropping jar file.

13. **Q:** Design a flexible pricing engine using SOLID.
    **A:** SRP: separate pricing rules. OCP: new rules implement PriceRule interface. ISP: PriceRule has `apply(Order, Price)` method. DIP: PricingEngine depends on List<PriceRule>.

14. **Q:** Design a data export system following SRP and OCP.
    **A:** SRP: ExportService orchestrates, FormatWriter handles format, DataProvider supplies data. OCP: new formats implement FormatWriter. ISP: each interface has focused methods.

15. **Q:** Design a workflow engine with SOLID.
    **A:** SRP: steps are separate classes. OCP: new steps implement WorkflowStep. ISP: Step interface with execute method. DIP: WorkflowEngine depends on List<WorkflowStep>.

16. **Q:** Apply SOLID to microservice decomposition.
    **A:** SRP: one business capability per service. OCP: new features as new services (not modifying existing). ISP: BFF per client type. DIP: services communicate via event interfaces.

17. **Q:** Design a rule engine with LSP compliance.
    **A:** Base Rule class/interface with `evaluate(Context)` and `execute(Context)`. All rule subtypes implement both methods correctly. Pre/post conditions documented and honored.

18. **Q:** Apply ISP to a large enterprise application with 10+ client types.
    **A:** Backend For Frontend (BFF) per client type. Each BFF exposes interface matching client needs. Common services behind shared interface. Clients never get methods they don't use.

19. **Q:** Design a caching layer following DIP.
    **A:** CacheService depends on CacheProvider interface. Implementations: RedisCache, CaffeineCache, NoOpCache. Switch via configuration without changing business code.

20. **Q:** How would you refactor a 10K-line controller class using SOLID?
    **A:** 1) Extract services per domain (SRP). 2) Extract validation strategies (OCP). 3) Split into focused interfaces (ISP). 4) Inject dependencies via constructors (DIP). 5) Remove inheritance violations (LSP).

## 12. Expert-Level Interview Questions (10: architect-level)

1. **Q:** How do SOLID principles apply to event-driven architectures?
    **A:** SRP: each event handler handles one event type. OCP: add new event handlers for new event types without modifying existing handlers. ISP: event interfaces contain only relevant data. DIP: services depend on event abstractions (interface), not concrete events.

2. **Q:** Design a framework that enforces SOLID compliance through architectural tests.
    **A:** ArchUnit tests: verify classes have correct dependencies, interfaces are segregated, no cyclic dependencies, high-level modules don't depend on low-level modules, concrete classes implement interfaces.

3. **Q:** How do SOLID principles interact with Domain-Driven Design?
    **A:** SRP: aggregate root has single responsibility. OCP: add new behaviors via domain events. ISP: domain interfaces specific to bounded context. DIP: domain layer depends on interfaces implemented by infrastructure.

4. **Q:** Explain how to balance SOLID principles with YAGNI (You Ain't Gonna Need It).
    **A:** Apply SOLID where change is expected. For stable code, simpler solutions work. Start with simple design, refactor to SOLID when actual change requests appear.

5. **Q:** How would you teach SOLID to a team of junior developers?
    **A:** Start with symptoms of violation: "This class has many reasons to change" (SRP). "We modified this file 10 times for new features" (OCP). "This subclass throws NotImplemented" (LSP, ISP). "This service directly instantiates database" (DIP).

6. **Q:** Analyze a real-world open-source framework (Spring) for SOLID compliance.
    **A:** SRP: BeanFactory, ApplicationContext, ResourceLoader are separate. OCP: extensible via BeanPostProcessor, ImportBeanDefinitionRegistrar. ISP: many focused interfaces (InitializingBean, DisposableBean, BeanFactoryAware). DIP: IoC container injects dependencies.

7. **Q:** How do SOLID principles apply to configuration management?
    **A:** SRP: separate configuration sources (DB config, file config, vault config). OCP: add new config sources via ConfigSource interface. ISP: ConfigSource has getValue(key) method. DIP: ConfigManager depends on ConfigSource list.

8. **Q:** Design a testing strategy that validates SOLID compliance.
    **A:** ArchUnit tests for structural compliance. Property-based testing for LSP (base class contract holds for derived classes). Mutation testing for SRP (changes to one responsibility shouldn't break others).

9. **Q:** How do SOLID principles relate to the evolution of software architecture (monolith -> microservices)?
    **A:** SRP guides service boundaries. OCP enables adding services without modifying existing ones. ISP informs BFF design. DIP suggests depending on event schemas (abstractions) rather than concrete service URLs.

10. **Q:** Critique SOLID principles in the context of modern backend development.
    **A:** Strengths: maintainability, testability, flexibility. Limitations: increased complexity, not all principles apply equally (LSP less relevant in composition-heavy code), can lead to over-abstraction. Best used pragmatically with awareness of trade-offs.

## 13. Debugging & Troubleshooting

### Issues Caused by SOLID Violations

**SRP Violation Symptoms:**
- Large class files (>500 lines).
- Class has many imports from unrelated domains.
- Changes to one feature break another feature.

**OCP Violation Symptoms:**
- Adding new feature requires modifying many existing files.
- Switch/if-else chains checking type.
- Difficult to add new variations.

**LSP Violation Symptoms:**
- instanceof checks before casting.
- Methods throwing UnsupportedOperationException.
- Conditional logic checking subclass type.

**ISP Violation Symptoms:**
- Empty method implementations.
- Interfaces with many methods (>10).
- Clients importing interfaces for one method.

**DIP Violation Symptoms:**
- new keyword for concrete classes in business logic.
- Static factory methods in business code.
- Changes to database class break business logic.

### Detection Tools
```java
// ArchUnit example for SOLID compliance
@AnalyzeClasses(packages = "com.example")
public class SolidComplianceTest {

    @ArchTest
    static final ArchRule srp_rule = classes()
        .that().resideInAPackage("..service..")
        .should().haveOnlyOneMethodWithName("execute")
        .orShould().beAnnotatedWith("@Service");

    @ArchTest
    static final ArchRule dip_rule = classes()
        .that().resideInAPackage("..service..")
        .should().onlyDependOnClassesThat()
        .resideInAnyPackage("..service..", "java..", "org.springframework..");
}
```

## 14. Comparison Section

### SOLID vs GRASP

| Principle | SOLID | GRASP |
|-----------|-------|-------|
| Focus | Class-level design | Responsibility assignment |
| Key Ideas | SRP, OCP, LSP, ISP, DIP | Controller, Creator, Polymorphism, etc. |
| Scope | Object-oriented design | General responsibility-driven design |
| Origin | Robert C. Martin | Craig Larman |

### SOLID vs Clean Architecture

| Clean Architecture Layer | SOLID Principle |
|--------------------------|-----------------|
| Entities | SRP (business rules) |
| Use Cases | SRP (application-specific rules) |
| Interface Adapters | DIP (depend on use case interfaces) |
| Frameworks/Drivers | OCP (plug into ports) |

## 15. Revision Notes

### Quick Recap
- **SRP**: One reason to change per class. Extract separate concerns.
- **OCP**: Extend via abstraction, not modification. Use strategy/template/ decorator patterns.
- **LSP**: Subtypes must behave as their base types. Square-Rectangle is the classic violation.
- **ISP**: Small, focused interfaces over large, general ones. Role interfaces.
- **DIP**: Depend on abstractions, not concretions. Use dependency injection.

### Common Violations to Spot
- "God class" with many methods/fields = SRP violation.
- `switch`/`if-else` on type = OCP violation.
- `instanceof` checks = LSP violation.
- Interface with `throws UnsupportedOperationException` = ISP violation.
- `new ConcreteClass()` in business logic = DIP violation.

## 16. Cheat Sheet

```
+-------------------------------------------------------------------+
|                     SOLID PRINCIPLES CHEAT SHEET                   |
+-------------------------------------------------------------------+
| LETTER | NAME                      | DESCRIPTION                    |
+--------+---------------------------+-------------------------------+
| S      | Single Responsibility    | One reason to change per class |
| O      | Open/Closed              | Open for extension, closed for |
|        |                          | modification                   |
| L      | Liskov Substitution      | Subtypes replaceable for base  |
| I      | Interface Segregation    | Many specific > one general    |
| D      | Dependency Inversion     | Depend on abstractions         |
+--------+---------------------------+-------------------------------+
| VIOLATION INDICATORS                                               |
+-------------------------------------------------------------------+
| SRP: "This class has 2000 lines and 50 methods"                    |
| OCP: "Add a new type? Modify this switch statement"                |
| LSP: "instanceof checks all over the code"                         |
| ISP: "This interface has 15 methods I don't need"                 |
| DIP: "Business logic creates new MySQLConnection()"               |
+-------------------------------------------------------------------+
| APPLYING SOLID IN SPRING BOOT                                      |
+-------------------------------------------------------------------+
| SRP: @Service, @Repository, @Controller separation                |
| OCP: Strategy pattern with List<Interface> injection               |
| LSP: Interface-based design, no deep inheritance                   |
| ISP: Small focused interfaces per role                            |
| DIP: Constructor injection with @Autowired                        |
+-------------------------------------------------------------------+
| DESIGN PATTERNS THAT SUPPORT SOLID                                 |
+-------------------------------------------------------------------+
| SRP   | Facade, Command, Interpreter                              |
| OCP   | Strategy, Template Method, Decorator, Visitor             |
| LSP   | Abstract Factory, Builder, Prototype                      |
| ISP   | Adapter, Proxy, Bridge                                    |
| DIP   | Factory Method, Service Locator, DI Container             |
+-------------------------------------------------------------------+
```
