# SOLID Principles

---

## Overview

- **Definition**
  - A mnemonic acronym for five design principles (Single Responsibility, Open/Closed, Liskov Substitution, Interface Segregation, Dependency Inversion) that make object-oriented designs more understandable, flexible, and maintainable.

  **Why It Exists**
    - Without SOLID, code becomes rigid (hard to change), fragile (changes break other parts), and immobile (hard to reuse).
    - These principles guide developers to create systems that are easy to extend, refactor, and test.

- **Key Concepts**
  - **SRP** — one reason to change per class.
  - **OCP** — open for extension, closed for modification.
  - **LSP** — subtypes must be substitutable for base types.
  - **ISP** — many specific interfaces over one general interface.
  - **DIP** — depend on abstractions, not concretions.

---

## Core Concepts

- **Single Responsibility Principle (SRP)**
  - A class should have only one reason to change. This doesn't mean one method — it means one cohesive responsibility.
  - Extract separate concerns into separate classes (e.g., separate `OrderService` into `OrderService`, `EmailService`, `InvoiceService`).

- **Open/Closed Principle (OCP)**
  - Software entities should be open for extension but closed for modification.
  - Achieved through abstraction (interfaces) and polymorphism. Add new behavior by creating new implementations, not by modifying existing code.
  - Strategy pattern is a classic OCP implementation.

- **Liskov Substitution Principle (LSP)**
  - Derived classes must be substitutable for their base classes without altering program correctness.
  - Subtypes must satisfy the behavioral contract of the base type. Classic violation: `Square` extending `Rectangle` (setting width also changes height).

- **Interface Segregation Principle (ISP)**
  - Clients should not be forced to depend on interfaces they do not use.
  - Many specific interfaces are better than one general-purpose interface. `RobotWorker` should not implement `eat()` and `sleep()` methods it doesn't need.

- **Dependency Inversion Principle (DIP)**
  - High-level modules should not depend on low-level modules. Both should depend on abstractions.
  - Abstractions should not depend on details — details should depend on abstractions.
  - Spring's constructor injection with interfaces is a practical implementation.

```java
// SRP — each class has one responsibility
@Service
public class OrderService {
    // Order creation logic only
}

@Service
public class EmailService {
    // Email sending logic only
}

// OCP — Strategy pattern: extend without modifying
public interface PaymentMethod {
    boolean process(Payment payment);
}

@Component
public class CreditCardPayment implements PaymentMethod {
    public boolean process(Payment payment) { /* credit card logic */ }
}

@Component
public class ApplePayPayment implements PaymentMethod {
    public boolean process(Payment payment) { /* apple pay logic */ } // New, no existing code changed
}

// DIP — depend on abstractions
@Service
public class OrderConfirmationService {
    private final NotificationSender notificationSender; // Interface

    public OrderConfirmationService(NotificationSender sender) { // Injection
        this.notificationSender = sender;
    }
}

public interface NotificationSender {
    void send(Notification notification);
}
```

---

## Common Mistakes

- **Over-Engineering with SOLID**
  - Applying all principles everywhere creates unnecessary complexity. Apply where change is expected; for stable code, simpler solutions work.
  - Applying best practices universally seems disciplined, and the cost of unnecessary abstraction is invisible until someone has to trace through five indirection layers to debug a simple bug.

- **SRP Taken Too Far**
  - A class with one method isn't necessarily SRP. Cohesion matters: related behaviors can live in one class.
  - Splitting each method into its own class seems like the ultimate application of "single responsibility" — the cost of scattered logic across dozens of tiny files only becomes apparent when you need to understand a workflow that spans 15 classes.

- **OCP via Inheritance Instead of Composition**
  - Deep inheritance hierarchies (`DiscountedOrder extends Order extends BaseOrder`) are rigid. Prefer composition with Strategy pattern.
  - Inheritance is the most intuitive OOP mechanism for code reuse, and deep hierarchies seem clean at first — the rigidity only surfaces when a change in `BaseOrder` unexpectedly breaks all three levels of subclasses.

- **LSP Violation Through Collection Types**
  - `List<String>` is not substitutable for `List<Object>` in Java. More subtly, subclass methods throwing `UnsupportedOperationException` violate LSP.
  - Java's type system allows the assignment and it compiles without error — the `ClassCastException` only appears at runtime when code expecting `List<Object>` tries to insert a non-String type.

- **DIP Leading to Yo-Yo Problem**
  - Excessive abstraction layers make code hard to follow. Use the right level of abstraction for the problem.
  - Depending on abstractions is a fundamental principle, and more layers seem like better design — the indirection cost only becomes visible when you need to open five files to trace what a single method call actually does.

---

## Key Design Considerations

- **SOLID in Microservices**
  - SRP: one business capability per service.
  - OCP: add features via new services/events, not modifying existing ones.
  - LSP: service API backward compatibility.
  - ISP: BFF per client type.
  - DIP: services depend on event schemas (abstractions), not concrete service URLs.

- **SOLID and Testing**
  - SRP makes classes easier to unit test (fewer dependencies).
  - OCP allows testing new features without retesting existing code.
  - DIP enables mocking abstractions for isolated tests.

- **Violation Detection Indicators**
  - SRP: class >500 lines with many unrelated imports.
  - OCP: switch/if-else chains checking type.
  - LSP: `instanceof` checks.
  - ISP: empty method bodies with `throw UnsupportedOperationException()`.
  - DIP: `new ConcreteClass()` in business logic.

- **SOLID as a Compass, Not a Rulebook**
  - Apply pragmatically: if a class has one reason to change, it's SRP; if adding a feature requires modifying existing code, it violates OCP; if a subclass breaks parent behavior, it violates LSP; if a client depends on methods it doesn't use, ISP violation; if high-level depends on low-level concrete classes, DIP violation.

---

## Real-World Scenarios

### Scenario 1: SRP Violation in an Order Service

- **Context**
  - A `UserService` class is 800 lines long. It handles user CRUD, password hashing, email sending, SMS notifications, audit logging, and session management. Adding a new feature (e.g., two-factor auth) requires modifying 5 different methods in this single class. Testing is painful because every test needs mocks for email, SMS, and database.

- **Resolution**
  - Apply SRP. Split into: `UserService` (user CRUD, business logic), `PasswordService` (hashing, validation, complexity rules), `EmailService` (send emails via SES/SendGrid), `SmsService` (send SMS via Twilio), `AuditService` (log user actions), `SessionService` (manage tokens, expiry).
  - Each class has one reason to change. Testing `UserService` now only needs to mock the narrow dependencies it actually uses.

```java
// Before: God class violating SRP
@Service
public class UserService {
    public User createUser(String name, String email, String rawPassword) {
        String hash = BCrypt.hashpw(rawPassword, BCrypt.gensalt());  // Should be in PasswordService
        User user = userRepository.save(new User(name, email, hash));
        sendEmail(email, "Welcome!");  // Should be in EmailService
        auditLog("USER_CREATED", user.getId());  // Should be in AuditService
        return user;
    }
}

// After: Each class has one responsibility
@Service
public class UserService {
    public User createUser(String name, String email, String rawPassword) {
        String hash = passwordService.hash(rawPassword);
        User user = userRepository.save(new User(name, email, hash));
        eventBus.publish(new UserCreatedEvent(user.getId(), email));
        return user;
    }
}
```

### Scenario 2: OCP Violation with Payment Processing

- **Context**
  - A `PaymentProcessor` class has a switch statement checking payment type to determine processing logic. Adding a new payment method (e.g., Apple Pay) requires modifying the switch, risking breaking existing methods.

- **Resolution**
  - Apply OCP with the Strategy pattern. Define a `PaymentMethod` interface. Each payment type is a separate implementation implementing `process(Payment payment)`. The `PaymentProcessor` uses a registry of strategies, selected by payment type. Adding Apple Pay means creating `ApplePayProcessor` — zero changes to existing code.

### Scenario 3: DIP Violation in an Order Confirmation Service

- **Context**
  - An `OrderConfirmationService` directly instantiates `SmtpEmailSender` to send confirmation emails. Testing requires an actual SMTP server. Switching to a different email provider (SendGrid, SES) means changing code.

- **Resolution**
  - Apply DIP. `OrderConfirmationService` depends on the `NotificationSender` interface, not `SmtpEmailSender`. The concrete implementation is injected via constructor. Spring's DI makes this seamless. Testing injects a `MockNotificationSender`. Switching providers requires only a new implementation class.

---

## Scenario-Based Questions

**Q: Your team has a `ReportService` class that generates PDFs, sends emails, archives old reports, and calculates report metrics. Changes to email formatting require redeploying the entire service. How do you refactor?**

- **Solution**
  - Apply SRP. Split into `ReportGenerator` (PDF creation), `EmailService` (sending), `ReportArchiver` (archive/storage), and `ReportAnalytics` (metrics). Each has one reason to change.
  - Email formatting changes only affect `EmailService`. Testing individual components becomes straightforward — no more mocking 5 dependencies for a single test.

---

**Q: You have a `ShippingCostCalculator` with a switch statement on shipping type (STANDARD, EXPRESS, OVERNIGHT). A new shipping type (SAME_DAY) is requested. How do you add it without modifying existing code?**

- **Solution**
  - Apply OCP. Define a `ShippingStrategy` interface with `calculate(Order)`. Create implementations for each type. Use a strategy registry (Map<String, ShippingStrategy> injected by Spring). Adding SAME_DAY means creating `SameDayShipping` class and registering it — the `ShippingCostCalculator` doesn't change.

> **Follow-up:** Your `SameDayShipping` strategy needs access to the inventory service to check if the item is in a nearby warehouse — a dependency none of the other strategies need. How does this affect the strategy interface design? Should all strategies now take an optional inventory service dependency?

---

**Q: A `Square` class extends `Rectangle`. When setting width on a Square, the height also changes. Code that works with Rectangle breaks when passed a Square. Which principle is violated?**

- **Solution**
  - LSP. The Square is not substitutable for Rectangle — setting width on a Rectangle doesn't change height, but on a Square it does.
  - Fix by not inheriting from Rectangle. Use a separate `Shape` interface with `area()` method, and let both Rectangle and Square implement it independently. Classic LSP violation example.

---

**Q: Your `UserService` interface has 15 methods: `create()`, `update()`, `delete()`, `findById()`, `sendEmail()`, `sendSMS()`, `resetPassword()`, `validatePassword()`, `generateToken()`, `refreshToken()`, etc. A mobile client only needs 3 methods. Which principle is violated?**

- **Solution**
  - ISP — the mobile client depends on methods it doesn't use. Split into focused interfaces: `UserCrudService` (create/update/delete/find), `NotificationService` (email/SMS), `AuthenticationService` (password reset, token management). The mobile client depends only on the small interfaces it needs.

> **Follow-up:** Six months later, a common use case needs to call methods from all three interfaces in sequence — the web client now depends on three separate interfaces while the mobile client depends on two. Has ISP made the client code more complex, and when does interface proliferation become its own anti-pattern?

---

**Q: A junior developer writes: `public class OrderService { private final SmtpEmailSender emailSender = new SmtpEmailSender(); }`. What's wrong and how do you fix it?**

- **Solution**
  - DIP violation — the high-level `OrderService` depends on a low-level concrete class (`SmtpEmailSender`). If you switch to SendGrid or mock for testing, you must change code.
  - Fix: depend on an abstraction (`NotificationSender` interface), inject via constructor. Spring handles the concrete implementation.

---

**Q: Your team has a `DataExporter` that exports to CSV. Now they need JSON, XML, and Excel support. The current code has a single method with a format parameter and a giant if-else chain. How do you make this extensible?**

- **Solution**
  - Apply OCP + Strategy. Create an `Exporter<T>` interface with `export(Data, OutputStream)`. Implement `CsvExporter`, `JsonExporter`, `XmlExporter`, `ExcelExporter`. The `DataExporter` delegates to the appropriate strategy based on format.
  - New formats require only a new implementation — no changes to the `DataExporter` class.

---

**Q: You encounter a class that's 1500 lines long and handles user management, billing, notifications, and reporting. Every team member is afraid to touch it. Which principle is most violated?**

- **Solution**
  - SRP — the class has 4+ reasons to change (user rules, billing rules, notification rules, reporting rules). A change to billing logic risks breaking user management. Refactor by extracting each concern into its own class with a focused responsibility.

---

**Q: Your `PaymentService` calls `creditCardProcessor.charge()`, `payPalProcessor.charge()`, and `applePayProcessor.charge()` in separate if-blocks. Adding a new processor requires adding another if-block. How do you fix this?**

- **Solution**
  - Apply DIP and OCP. Define a `PaymentProcessor` interface with `charge(Payment)`. Each provider implements it. Use a `PaymentProcessorRegistry` (Map<String, PaymentProcessor> injected by Spring). The `PaymentService` just calls `registry.get(type).charge(payment)`. Adding a new processor means adding a new implementation and registering it.

> **Follow-up:** Each payment processor now needs to initialize its own connection pool, API credentials, and health checks — adding a new processor is still a new class, but it also needs configuration, secret management, and monitoring setup. Does OCP guarantee that adding a new implementation is zero-modification if the supporting infrastructure requires changes in multiple places?

---

**Q: A subclass overrides a method to throw `UnsupportedOperationException`. Why is this a SOLID violation and which principle?**

- **Solution**
  - LSP violation. The subclass is not substitutable for the base class because calling the method on the subclass breaks the contract (throws exception instead of returning a value).
  - Fix by splitting the interface: base interface with shared methods, separate sub-interfaces for optional behaviors. Use the ISP to avoid fat interfaces that force subclasses to implement methods they don't support.

---

**Q: Your notification system has `EmailNotification` and `SMSNotification` classes. Now you need `PushNotification`. The current design has an abstract `Notification` class with `send()` method. Is this OCP-compliant?**

- **Solution**
  - Yes, if you add `PushNotification extends Notification` without modifying existing classes. OCP is satisfied — the system is open for extension (new notification type) and closed for modification (existing types unchanged).
  - However, if adding push requires modifying the `NotificationProcessor` to add a new case, that violates OCP. Use a strategy pattern for the processor too.

---

## Interview Questions

- **What does SOLID stand for?**
  - Single Responsibility, Open-Closed, Liskov Substitution, Interface Segregation, Dependency Inversion.

- **What is the Single Responsibility Principle (SRP)?**
  - A class should have only one reason to change. It should have one cohesive responsibility. Not "does one thing" — it owns a single concern.
  - Example: `EmailService` handles email; `OrderService` handles orders.

- **What is the Open/Closed Principle (OCP)?**
  - Software entities should be open for extension (add new behavior) but closed for modification (don't change existing code).
  - Achieved via abstraction and polymorphism — the Strategy pattern is a classic implementation.

- **What is the Liskov Substitution Principle (LSP)?**
  - Derived classes must be substitutable for their base classes without altering program correctness.
  - Subtypes must satisfy the behavioral contract of the base type. Classic violation: `Square extends Rectangle` (setting width also changes height).

- **Which principle is violated by a "fat interface" with methods that many clients don't use?**
  - Interface Segregation Principle (ISP). Many client-specific interfaces are better than one general-purpose interface.
  - A `Worker` interface with `work()` and `eat()` forces `RobotWorker` to implement `eat()` unnecessarily.

- **What is the difference between Dependency Inversion Principle (DIP) and Dependency Injection (DI)?**
  - DIP is a principle: high-level modules should depend on abstractions, not concretions.
  - DI is a pattern to achieve DIP: dependencies are injected (via constructor, setter, or interface) rather than created internally.

- **How do you detect an LSP violation?**
  - If substituting a derived class causes unexpected behavior, throws new exceptions (`UnsupportedOperationException`), or silently ignores method calls — LSP is violated.
  - Common indicators: `instanceof` checks for subclasses, overridden methods that do nothing, or overriding to narrow the contract.

- **Which SOLID principles help most with unit testing?**
  - SRP — smaller classes with focused responsibilities are easier to test with fewer mocks.
  - DIP — depending on abstractions allows injecting mocks instead of real implementations.
  - ISP — small interfaces mean tests only mock what they need.

- **How does SRP apply to microservices?**
  - Each microservice should have one business capability — SRP at the service boundary.
  - An `OrderService` shouldn't handle payments or user management. Service boundaries should align with business capabilities and bounded contexts.

- **How do SOLID principles relate to each other?**
  - SRP keeps classes focused. OCP lets you extend without modifying (via interfaces). LSP ensures subclasses work correctly through those interfaces. ISP keeps interfaces focused so clients don't depend on what they don't need. DIP lets high-level code depend on those interfaces rather than concretions.
  - They form a cohesive set for maintainable OO design.

---

## Developer Recommendations

- **Watch for SRP violations: classes >300 lines or with "and" in their name**
  - A class named `UserAndEmailAndPaymentManager` is a clear SRP violation. If a class has imports from 5+ different packages, it likely has too many responsibilities.
  - Split by identifying the reasons for change — each reason to change should map to one class. Use package-by-feature (one package per feature) to naturally enforce SRP.
  - **Production story:** A fintech company's `TransactionService` grew to 2,500 lines handling validation, fraud checks, ledger updates, notification dispatch, and audit logging — when a compliance requirement changed the audit format, a bug in the audit code accidentally skipped fraud checks for 48 hours before anyone noticed, because unrelated logic shared the same class.

- **Use the Strategy pattern to satisfy OCP instead of switch/if-else**
  - When you add a new payment method, export format, or notification channel, you shouldn't modify existing classes.
  - Extract varying behavior into Strategy interfaces. Spring injection automatically collects all implementations. Adding a new strategy is a new class + registration only — the core logic never changes.

- **Prefer composition over inheritance to avoid LSP violations**
  - Inheritance hierarchies are rigid. `Square extends Rectangle` seems natural but violates LSP.
  - Instead of inheritance, compose behaviors: a `ShippingCalculator` has a `List<FeeRule>` rather than extending from a base `FeeCalculator`.
  - Composition with interfaces satisfies OCP and avoids LSP pitfalls.

- **Design small, focused interfaces following ISP**
  - An interface with 15 methods forces every implementer to provide all 15, even if some are irrelevant.
  - Split into role-based interfaces: `Readable`, `Writable`, `Deletable`, `Exportable`. Spring's `JpaRepository` already does this — `CrudRepository`, `PagingAndSortingRepository`, `JpaSpecificationExecutor` are separate interfaces.

- **Use constructor injection for DIP**
  - Constructor injection makes dependencies explicit (the constructor signature shows what the class needs), enables immutability (`final` fields), and fails at compile time if a required dependency is missing.
  - Field injection (`@Autowired` on fields) hides dependencies and fails only at runtime with `NullPointerException`. Never use field injection in production code.
  - **Production story:** A payment service used field injection throughout — when a refactoring renamed the `PaymentGateway` class, the old field-injected reference silently became `null` at runtime (because Spring couldn't match the field type), causing `NullPointerException` in production for 20 minutes while developers traced through bytecode proxies to understand why a `@Autowired` field was null.
