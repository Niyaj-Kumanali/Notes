# Object-Oriented Programming (OOPs)

---

## 1. Executive Summary

### What Is It?
Object-Oriented Programming (OOP) is a programming paradigm that organizes software design around **objects** rather than functions and logic. An object is a self-contained entity that contains **data** (fields/attributes) and **behavior** (methods/procedures that operate on that data).

### Why Does It Exist?
OOP was created to address the limitations of procedural programming as software grew in complexity. Procedural code becomes difficult to maintain, extend, and debug at scale because data and logic are scattered across functions with global state. OOP solves this by:

- **Encapsulation** — Bundling data with the code that operates on it, hiding internal state
- **Modularity** — Breaking systems into discrete, interchangeable components
- **Reusability** — Creating once, using everywhere through inheritance and composition
- **Maintainability** — Localizing changes to specific objects rather than ripple effects across the codebase

### Real-World Use Cases
| Domain | Example |
|--------|---------|
| **Enterprise Applications** | Banking systems where `Account`, `Customer`, `Transaction` are natural objects |
| **GUI Frameworks** | Swing, JavaFX, WPF — where `Button`, `Window`, `TextField` are objects with state and behavior |
| **Game Development** | `Player`, `Enemy`, `Weapon`, `Level` with inheritance hierarchies |
| **E-Commerce** | `Order`, `Cart`, `Product`, `PaymentGateway`, `InventoryManager` |
| **ORM Frameworks** | Hibernate, Entity Framework — map database tables to domain objects |
| **Web Frameworks** | Spring MVC controllers, ASP.NET Core controllers as objects handling HTTP requests |
| **Simulation Systems** | Aircraft simulators where `Engine`, `Flap`, `Radar` are modeled as objects |

### When to Use It
- System has identifiable real-world entities that map naturally to objects
- Codebase is expected to grow and evolve over years
- Multiple developers will work on the codebase simultaneously
- Business logic changes frequently and needs to be isolated
- You need to model complex relationships between entities

### When NOT to Use It
- Extremely simple scripts (< 100 lines) where functions suffice
- Performance-critical hot paths in game engines or embedded systems where virtual dispatch overhead matters
- Pure data transformation pipelines (ETL, map-reduce) where functional programming is more natural
- Stateless utility classes with no behavior (these are just namespaces in disguise)
- Micro-ORM thin layers where full domain models are overkill (use Dapper-style raw SQL instead)

---

## 2. Core Theory

### The Four Pillars of OOP

#### 2.1 Encapsulation
Encapsulation is the bundling of data and methods that operate on that data within a single unit (class), restricting direct access to an object's internal state.

**Key Mechanism:** Access modifiers (`private`, `protected`, `public`, `default`/`internal`)

```java
// Encapsulation example
public class BankAccount {
    private String accountNumber;     // Hidden state
    private double balance;           // Hidden state
    private List<Transaction> transactions = new ArrayList<>();

    public BankAccount(String accountNumber, double initialDeposit) {
        this.accountNumber = accountNumber;
        this.balance = initialDeposit;
        this.transactions.add(new Transaction("OPEN", initialDeposit));
    }

    // Controlled access through public methods
    public double getBalance() {
        return balance;
    }

    public void deposit(double amount) {
        if (amount <= 0) throw new IllegalArgumentException("Amount must be positive");
        balance += amount;
        transactions.add(new Transaction("DEPOSIT", amount));
    }

    public void withdraw(double amount) {
        if (amount <= 0) throw new IllegalArgumentException("Amount must be positive");
        if (amount > balance) throw new InsufficientFundsException(balance, amount);
        balance -= amount;
        transactions.add(new Transaction("WITHDRAW", amount));
    }

    // Internal detail hidden from callers
    private void auditLog(String action) {
        // Writing to audit database — encapsulating infrastructure concern
    }
}
```

**Benefits:**
- Protects invariants (e.g., balance can never go negative without business logic approval)
- Allows implementation to change without affecting callers
- Reduces coupling between components
- Makes thread safety easier to reason about

#### 2.2 Inheritance
Inheritance allows a class to acquire the properties and behaviors of another class, forming a parent-child hierarchy.

```java
// Inheritance example
public class Vehicle {
    protected String registrationNumber;
    protected String make;
    protected String model;

    public void start() { System.out.println("Vehicle starting..."); }
    public void stop() { System.out.println("Vehicle stopping..."); }
}

public class Car extends Vehicle {
    private int numberOfDoors;

    @Override
    public void start() {
        System.out.println("Turning ignition key...");
        super.start(); // Calls Vehicle.start()
    }
}

public class ElectricCar extends Car {
    private Battery battery;

    @Override
    public void start() {
        System.out.println("Powering electrical system...");
        // Calling super.start() deliberately omitted — electric cars don't
        // have ignition keys
    }
}
```

**Inheritance vs Composition trade-off:** See Section 13 for a detailed comparison.

#### 2.3 Polymorphism
Polymorphism allows objects of different types to respond to the same method call in different ways.

**Compile-time polymorphism (Method Overloading):**
```java
public class Calculator {
    public int add(int a, int b) { return a + b; }
    public double add(double a, double b) { return a + b; }
    public int add(int a, int b, int c) { return a + b + c; }
}
```

**Runtime polymorphism (Method Overriding):**
```java
public interface PaymentProcessor {
    boolean processPayment(Order order);
}

public class CreditCardProcessor implements PaymentProcessor {
    @Override
    public boolean processPayment(Order order) {
        // Charge credit card via Stripe API
        return stripeClient.charge(order.getTotal());
    }
}

public class PayPalProcessor implements PaymentProcessor {
    @Override
    public boolean processPayment(Order order) {
        // Redirect to PayPal
        return paypalClient.executePayment(order.getTotal());
    }
}

// Client code — polymorphic, doesn't care about concrete type
public class CheckoutService {
    private PaymentProcessor processor;

    public CheckoutService(PaymentProcessor processor) {
        this.processor = processor;
    }

    public OrderResult checkout(Cart cart) {
        Order order = new Order(cart);
        boolean paid = processor.processPayment(order);
        return paid ? OrderResult.success(order) : OrderResult.failed(order);
    }
}
```

#### 2.4 Abstraction
Abstraction hides complex implementation details and exposes only the essential features of an object.

```java
// Abstraction via abstract class
public abstract class DatabaseConnection {
    protected String connectionString;

    public DatabaseConnection(String connectionString) {
        this.connectionString = connectionString;
    }

    public abstract Connection open();
    public abstract void close();
    public abstract ResultSet executeQuery(String sql);
}

public class MySqlConnection extends DatabaseConnection {
    @Override
    public Connection open() {
        // MySQL-specific connection logic
        return DriverManager.getConnection("jdbc:mysql://" + connectionString);
    }

    @Override
    public void close() { /* MySQL cleanup */ }

    @Override
    public ResultSet executeQuery(String sql) { /* MySQL query execution */ }
}
```

### Concepts Not Everyone Gets Right

#### Association, Aggregation, Composition
These three form a hierarchy of relationship strength:

| Relationship | Ownership | Lifetime | Example |
|-------------|-----------|----------|---------|
| **Association** | None (uses-a) | Independent | `Professor` ↔ `Student` |
| **Aggregation** | Weak (has-a) | Independent | `Department` → `Professor` |
| **Composition** | Strong (owns-a) | Dependent | `House` → `Room` |

```java
// Association — both exist independently
public class Doctor {
    private List<Patient> patients = new ArrayList<>(); // Association
}

// Aggregation — Department contains Professors, but Professors exist without Department
public class Department {
    private List<Professor> professors;
    public Department(List<Professor> professors) {
        this.professors = professors; // External references
    }
}

// Composition — Rooms are created and destroyed with House
public class House {
    private List<Room> rooms = new ArrayList<>();
    public House() {
        rooms.add(new Room("Living Room"));
        rooms.add(new Room("Bedroom"));
        // Rooms are part of House's lifecycle
    }
}
```

---

## 3. Under-the-Hood Deep Dive

### JVM Memory Behavior with Objects

When you write `new Customer()`, the JVM does the following:

1. **Class loading** — Load `.class` file, verify bytecode, allocate static memory
2. **Heap allocation** — Memory allocated on Eden space (young generation)
3. **Object header** — 12–16 bytes (mark word + klass pointer + optional padding)
4. **Instance fields** — Memory for each field (aligned to 8-byte boundaries)
5. **Constructor** — Instance initializer methods `<init>` invoked
6. **Reference** — Stack variable points to heap address

**Memory layout of an object:**
```
|-- Mark Word (8 bytes) --|-- Klass Pointer (4 bytes compressed) --|-- Fields --|
|   (hash, GC info, locks) |   (pointer to Class metadata)         |   data     |
```

**Object size estimation:**
```java
// In HotSpot JVM with compressed OOPs (typical):
class Point {
    int x;    // 4 bytes
    int y;    // 4 bytes
    // Header: 12 bytes
    // Total before alignment: 20 -> rounds to 24 bytes
}
```

### Virtual Method Dispatch (Polymorphism)
The JVM uses **virtual method tables (vtable)** for dynamic dispatch:

```text
Object vtable:
  [0] hashCode()
  [1] equals()
  [2] toString()

Customer vtable extends Object:
  [0] hashCode()       -> overridden
  [1] equals()          -> overridden
  [2] toString()         -> inherited
  [3] calculateDiscount() -> new method
```

When calling `customer.calculateDiscount()`, the JVM:
1. Loads the object reference
2. Looks up the class pointer from the object header
3. Indexes into the vtable at the correct offset
4. Jumps to the method implementation

**Performance implication:** Virtual dispatch costs ~1–5ns more than static dispatch. HotSpot's inline caching optimizes monomorphic call sites to near-zero cost.

### .NET CLR Behavior
Similar to JVM but with differences:
- **Object header:** 8 bytes (sync block index + method table pointer)
- **Value types (structs)** are allocated on stack or inline in arrays — no heap allocation, no GC pressure
- **Reference types** use the managed heap with generational GC
- **Virtual call** uses the Method Table (equivalent to vtable)
- **Constrained virtual calls** — .NET can devirtualize sealed classes at JIT time

### Garbage Collection Implications
- Objects are garbage-collected; primitives inside objects are too
- Short-lived objects die in Young Generation (minor GC ~1ms)
- Long-lived objects promote to Old Generation (major GC ~10–100ms)
- Object pooling can reduce GC pressure for frequently created objects
- `String` and primitive wrapper classes create allocation pressure

---

## 4. Production Code Examples

### 4.1 Basic Example — Value Object

```java
// GOOD: Immutable value object
public final class Money {
    private final BigDecimal amount;
    private final Currency currency;

    public Money(BigDecimal amount, Currency currency) {
        this.amount = Objects.requireNonNull(amount);
        this.currency = Objects.requireNonNull(currency);
        if (amount.scale() > currency.getDefaultFractionDigits()) {
            throw new IllegalArgumentException("Excessive precision");
        }
    }

    public Money add(Money other) {
        if (!this.currency.equals(other.currency)) {
            throw new CurrencyMismatchException(this.currency, other.currency);
        }
        return new Money(this.amount.add(other.amount), this.currency);
    }

    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (!(o instanceof Money)) return false;
        Money money = (Money) o;
        return amount.compareTo(money.amount) == 0 &&
               currency.equals(money.currency);
    }

    @Override
    public int hashCode() {
        return Objects.hash(amount.stripTrailingZeros(), currency);
    }

    // Getters only — no setters (immutable)
    public BigDecimal getAmount() { return amount; }
    public Currency getCurrency() { return currency; }
}

// BAD: Mutable, exposes internals
public class BadMoney {
    public double amount;       // Exposed field, no encapsulation
    public String currency;     // String instead of Currency type
    // No validation, no immutability
}
```

### 4.2 Intermediate Example — Strategy Pattern with Polymorphism

```java
// Abstraction
public interface NotificationStrategy {
    void send(String recipient, String message);
    NotificationChannel getChannel();
}

// Concrete implementations
@Component
public class EmailNotification implements NotificationStrategy {
    private final JavaMailSender mailSender;

    @Override
    public void send(String recipient, String message) {
        mailSender.send(prepareMessage(recipient, message));
    }

    @Override
    public NotificationChannel getChannel() { return NotificationChannel.EMAIL; }
}

@Component
public class SMSNotification implements NotificationStrategy {
    private final TwilioClient twilio;

    @Override
    public void send(String recipient, String message) {
        twilio.sendMessage(recipient, message);
    }

    @Override
    public NotificationChannel getChannel() { return NotificationChannel.SMS; }
}

@Component
public class PushNotification implements NotificationStrategy {
    @Override
    public void send(String deviceToken, String message) {
        // Send via Firebase Cloud Messaging
    }

    @Override
    public NotificationChannel getChannel() { return NotificationChannel.PUSH; }
}

// Production service using polymorphism
@Service
public class NotificationService {
    private final Map<NotificationChannel, NotificationStrategy> strategies;

    @Autowired
    public NotificationService(List<NotificationStrategy> strategies) {
        this.strategies = strategies.stream()
            .collect(Collectors.toMap(
                NotificationStrategy::getChannel,
                Function.identity()
            ));
    }

    public void notify(User user, NotificationMessage message) {
        for (NotificationChannel channel : user.getPreferredChannels()) {
            NotificationStrategy strategy = strategies.get(channel);
            if (strategy != null) {
                try {
                    strategy.send(user.getContactFor(channel), message.getText());
                } catch (NotificationException e) {
                    log.warn("Failed to send via {}: {}", channel, e.getMessage());
                    // Fallback logic
                }
            }
        }
    }
}
```

### 4.3 Advanced Example — Domain-Driven Aggregate Root

```java
// Aggregate root with encapsulation, invariants, and domain events
@Entity
@Table(name = "orders")
public class Order {
    @EmbeddedId
    private OrderId id;

    @Version
    private Long version; // Optimistic locking

    @Embedded
    private OrderStatus status;

    @OneToMany(cascade = ALL, orphanRemoval = true)
    @JoinColumn(name = "order_id", nullable = false)
    private List<OrderLine> lines = new ArrayList<>();

    @Embedded
    private Money total;

    @Embedded
    private ShippingAddress shippingAddress;

    @Transient
    private List<DomainEvent> domainEvents = new ArrayList<>();

    // Private constructor — use factory method
    private Order() {}

    // Factory method (static factory)
    public static Order create(CustomerId customerId, Cart cart, ShippingAddress address) {
        Order order = new Order();
        order.id = OrderId.generate();
        order.status = OrderStatus.PENDING;
        order.shippingAddress = address;

        for (CartItem item : cart.getItems()) {
            order.addLine(OrderLine.fromCartItem(item));
        }

        order.recalculateTotal();
        order.domainEvents.add(new OrderPlacedEvent(order.id, order.total));
        return order;
    }

    // Command method that enforces invariants
    public void markShipped(TrackingNumber trackingNumber) {
        if (!this.status.canTransitionTo(OrderStatus.SHIPPED)) {
            throw new IllegalStateException(
                "Cannot ship order in status: " + this.status
            );
        }
        this.status = OrderStatus.SHIPPED;
        this.domainEvents.add(new OrderShippedEvent(this.id, trackingNumber));
    }

    // Public getters (immutable from outside)
    public OrderId getId() { return id; }
    public OrderStatus getStatus() { return status; }
    public Money getTotal() { return total; }
    public List<OrderLine> getLines() {
        return Collections.unmodifiableList(lines);
    }

    // Package-private for infrastructure
    List<DomainEvent> releaseEvents() {
        List<DomainEvent> events = new ArrayList<>(this.domainEvents);
        this.domainEvents.clear();
        return events;
    }

    private void addLine(OrderLine line) {
        this.lines.add(line);
    }

    private void recalculateTotal() {
        this.total = lines.stream()
            .map(OrderLine::getSubtotal)
            .reduce(Money.ZERO, Money::add);
    }
}
```

### 4.4 Bad Implementation — God Object (Anti-Pattern)

```java
// BAD: God Object — violates Single Responsibility and Encapsulation
public class OrderService {
    private Database db;
    private EmailService email;
    private PaymentGateway payment;

    public void processOrder(OrderData data) {
        // Validate
        if (data.getItems().isEmpty()) throw new ValidationException();

        // Calculate
        double total = 0;
        for (ItemData item : data.getItems()) {
            total += item.getPrice() * item.getQuantity();
            // Discount logic mixed here
            if (item.getCategory().equals("ELECTRONICS")) {
                total *= 0.95; // 5% off
            }
            // Tax logic also here
            total *= 1.08;
        }

        // Persist
        db.saveOrder(data, total);

        // Send email — coupling business logic to delivery mechanism
        email.sendOrderConfirmation(data.getEmail(), data.getOrderId());

        // Payment — error handling mixed with everything
        boolean paid = payment.charge(data.getCreditCard(), total);
        if (!paid) {
            // Rollback
            db.deleteOrder(data.getOrderId());
            email.sendPaymentFailed(data.getEmail(), data.getOrderId());
        }
    }
}

// GOOD: Separated responsibilities
@Service
public class OrderProcessingService {
    private final OrderFactory orderFactory;
    private final OrderRepository orderRepository;
    private final PaymentService paymentService;
    private final DomainEventPublisher eventPublisher;

    @Transactional
    public Order processOrder(Cart cart, PaymentDetails payment) {
        Order order = orderFactory.createFromCart(cart);
        orderRepository.save(order);

        PaymentResult result = paymentService.charge(order, payment);
        if (result.isSuccess()) {
            order.markPaid(result.getTransactionId());
        } else {
            order.markFailed(result.getFailureReason());
        }

        orderRepository.save(order);
        eventPublisher.publish(order.releaseEvents());
        return order;
    }
}
```

---

## 5. Real-World Scenarios

### Scenario 1: Refactoring Spaghetti Procedural Code
**Problem:** A 5000-line `ProcessOrder()` function handles validation, pricing, tax calculation, inventory check, payment, email, and shipping — all in one function. Any change breaks everything.

**Analysis:** Violates SRP, OCP, and basic encapsulation. No single developer understands the full function.

**Solution:** Extract domain objects (`Order`, `LineItem`, `Customer`, `Payment`, `Shipment`), each with focused responsibilities. Move logic into the relevant object.

**Why it works:** Changes to pricing logic only affect `PricingCalculator`. Changes to shipping only affect `ShipmentService`. Testing becomes isolated.

**Alternative:** Start with a Facade that wraps the legacy function, then incrementally extract objects behind interfaces.

### Scenario 2: Deep Inheritance Hierarchy Causing "Diamond Problem"
**Problem:** A UI framework has `Control -> ScrollableControl -> ContainerControl -> Form`. Adding a new `DraggableControl` requires duplicating code across branches.

**Analysis:** Deep inheritance creates fragile hierarchies where changes at the top ripple destructively.

**Solution:** Favor composition over inheritance. Extract `Draggable` behavior into an interface + delegate to a `DragBehavior` helper object.

**Why it works:** `DraggableTextBox` implements `Draggable` interface and delegates to `DragHandler`. No hierarchy changes needed.

**Alternative:** Use the Decorator pattern to add behavior at runtime.

### Scenario 3: Mutable Domain Objects Causing Race Conditions
**Problem:** `Cart item.setQuantity(qty)` in a web app causes inconsistent state when two requests hit simultaneously.

**Analysis:** Mutable state + no synchronization = race condition. Encapsulation was violated.

**Solution:** Immutable objects + command pattern. `AdjustQuantityCommand` creates a new `Cart` with the adjusted item rather than mutating in place.

**Why it works:** Immutable objects are thread-safe by definition. No locks needed.

### Scenario 4: Leaky Abstractions in Database Layer
**Problem:** `CustomerRepository` exposes `save()`, but callers must call `flush()` and `clear()` to avoid memory issues.

**Analysis:** Leaky abstraction — internal ORM details exposed to callers.

**Solution:** Repository pattern with unit of work. `CustomerRepository.save()` handles everything internally.

**Why it works:** Callers don't know about ORM. You could swap JPA for JDBC without changing callers.

### Scenario 5: Anemic Domain Model
**Problem:** `Order` has only getters/setters. All business logic lives in `OrderService` classes. OOP is used as a data container.

**Analysis:** This is procedural programming with classes. No encapsulation, no behavior cohesion.

**Solution:** Move business logic into domain objects. `Order.calculateTotal()`, `Order.applyDiscount()`, `Order.validate()`.

**Why it works:** Logic is where the data is. Changing how an order works only touches the `Order` class.

### Scenario 6: Circular Dependencies Between Classes
**Problem:** `OrderService` depends on `InventoryService`, which depends on `NotificationService`, which depends on `OrderService`.

**Analysis:** Tight coupling leads to circular compilation and runtime issues. Impossible to test independently.

**Solution:** Introduce an interface (`InventoryObserver`) and event-driven communication. `OrderService` publishes `OrderPlacedEvent`; `InventoryService` subscribes.

**Why it works:** Direction of dependency is inverted. Classes only depend on abstractions.

### Scenario 7: Testing Difficulty Due to Hidden Dependencies
**Problem:** `EmailService emailService = new EmailService();` is instantiated inside `OrderService.processOrder()`. Cannot mock for unit tests.

**Analysis:** Constructor does work; dependencies are hidden. Violates Dependency Inversion Principle.

**Solution:** Constructor injection. `OrderService(EmailService emailService, PaymentGateway gateway)`.

**Why it works:** Dependencies are explicit. Mocks can be injected. Testability improves dramatically.

### Scenario 8: Performance Issues from Object Over-Creation
**Problem:** Creating thousands of `BigDecimal` objects per request causes GC pauses and slow throughput.

**Analysis:** Value objects for every calculation create allocation pressure.

**Solution:** Object pooling for expensive objects, or primitive-based calculation where precision allows. Use `int` (cents) instead of `BigDecimal` for money in non-financial contexts.

**Why it works:** Reduces GC pressure by 10-100x in hot paths.

### Scenario 9: Feature Envy Across Services
**Problem:** `OrderService.extractCustomerReport()` accesses 20 getters on `Customer` and computes statistics — the method belongs in `Customer`.

**Analysis:** Feature envy — a method is more interested in another class's data than its own.

**Solution:** Move the method to `Customer`. `Customer.generateReport()`.

**Why it works:** Cohesion increases. If `Customer` structure changes, only one place needs updating.

### Scenario 10: God Class Violating Single Responsibility
**Problem:** `User` class has 50 methods covering authentication, profile, billing, preferences, and admin functions.

**Analysis:** If a class has "and" in its description, it's a God class. `User` is responsible for everything "and" everything else.

**Solution:** Split: `UserProfile`, `UserCredentials`, `UserBilling`, `UserPreferences`, `UserRole`.

**Why it works:** Each class has one reason to change. Security changes don't touch billing code.

---

## 6. Performance Considerations

| Concern | Impact | Mitigation |
|---------|--------|------------|
| Virtual dispatch | 1–5ns overhead per call | Inline caching in JVM/JIT; prefer final/sealed classes |
| Object allocation | Heap + GC pressure | Object pooling; stack allocation for structs (.NET); primitive collections |
| Deep inheritance | Larger vtable, slower dispatch | Prefer composition; limit hierarchy depth to ≤5 levels |
| Reflection-heavy OOP | 10-100x slower than direct calls | Cache MethodHandles; use code generation at compile time |
| Boxing/unboxing | Heap allocation for value types | Use specialized collections (IntArrayList, etc.) |
| Getter/setter chains | Method call overhead per access | Law of Demeter — don't chain; expose computed values |

### Time Complexity of OOP Patterns
| Pattern | Runtime Overhead | Memory Overhead |
|---------|-----------------|-----------------|
| Direct method call | O(1) | 0 |
| Virtual method call | O(1) (~1–5ns) | 1 pointer per object |
| Interface dispatch | O(1) (~2–10ns) | 1–2 pointers per object |
| Dynamic proxy | O(1) (~10–100ns) | 1 proxy object |
| Reflection invocation | O(1) (~100–1000ns) | Metadata objects |

### Optimization Strategies
1. **HotSpot inlining** — Write small, frequently-called methods that JIT can inline
2. **Primitive collections** — Use `IntArrayList` instead of `ArrayList<Integer>`
3. **Flyweight pattern** — Share immutable state across many objects
4. **Struct over class (.NET)** — Use `struct` for small, immutable, frequently-allocated types
5. **Avoid deep hierarchies** — Each level adds dispatch cost
6. **Prefer composition** — Reduces indirection compared to dynamic dispatch

---

## 7. Security Considerations

### Common OOP Security Issues

| Issue | Example | Mitigation |
|-------|---------|------------|
| Mutable shared state | Thread-unsafe `SimpleDateFormat` | Use immutable objects, thread-local, or synchronized blocks |
| Insecure deserialization | `ObjectInputStream.readObject()` can execute arbitrary code | Validate inputs; use whitelist of allowed classes |
| IDOR (Insecure Direct Object Reference) | `order.cancel(userId)` lets user cancel any order | Authorization check in every method |
| Reflection abuse | `field.setAccessible(true)` breaks encapsulation | SecurityManager (Java); Code Access Security (.NET) |
| Abstract class impersonation | Extending restricted classes to access protected members | Mark sensitive classes `final`/`sealed` |

### Enterprise Best Practices
- Always use `private` fields with controlled access
- Prefer `final` classes unless extension is explicitly designed for
- Never expose mutable internal collections — return `Collections.unmodifiableList()` or copies
- Validate all inputs at the boundary of every public method
- Use `Optional`/nullable annotations to make null contracts explicit

---

## 8. Common Mistakes (20+)

| # | Mistake | Why It Happens | Consequences | Correct Approach |
|---|---------|---------------|--------------|------------------|
| 1 | Exposing internal state with public fields | Laziness, C struct habit | No control; invariants violated | Private fields + getters |
| 2 | Returning mutable references to internal collections | Convenience | Callers modify internal state | Return unmodifiable view or defensive copy |
| 3 | Using inheritance for code reuse only | "My teacher said inheritance is good" | Fragile base class; deep hierarchies | Composition + delegation |
| 4 | God classes (too many responsibilities) | "Just add one more method" | Hard to test, maintain, understand | Single Responsibility Principle |
| 5 | Feature envy | Convenience | Low cohesion | Move method to the class that owns the data |
| 6 | Using `instanceof` instead of polymorphism | Don't know interface-based design | Brittle code; violates OCP | Polymorphic dispatch via interface |
| 7 | Anemic domain models | "Data + service is cleaner" | Procedural code in OOP clothing | Move behavior into domain objects |
| 8 | Violating LSP with incorrect subtypes | Misunderstanding substitutability | Runtime failures with polymorphic code | Liskov Substitution Principle validation |
| 9 | Constructor doing too much work | Eager initialization | Hard to test; slow instantiation | Separate construction from initialization |
| 10 | Circular dependencies | Lack of dependency inversion | Uncompilable; untestable | DIP, interfaces, events |
| 11 | Not overriding `equals()`/`hashCode()` | Forgotten or rushed | Broken collections (Set, Map) | Always override both together |
| 12 | Overusing static methods | "It's just a utility" | Hidden dependencies; untestable | Instance methods with injected dependencies |
| 13 | Ignoring `hashCode()` contract | Not understanding HashMap | Objects lost in collections | Consistent hash + equals |
| 14 | Exposing too many public methods | "Someone might need it" | Large API surface; tight coupling | Minimal public API; package-private internals |
| 15 | Leaky abstractions | Implementation details showing through | Brittle code; hidden dependencies | Clean interface design |
| 16 | Not using composition over inheritance | "Inheritance is OOP!" | Rigid class hierarchies | Favor composition |
| 17 | Mutable value objects | "I need to change one field" | Thread-safety issues; aliasing bugs | Make value objects immutable |
| 18 | Using inheritance for implementation sharing | "Saves typing" | Breaks encapsulation; coupling | Static helper methods or composition |
| 19 | Over-engineering with patterns | "We need an AbstractFactoryFactory" | Unnecessary complexity | YAGNI — simple first, patterns as needed |
| 20 | Not closing resources in base classes | Forget to call super.close() | Resource leaks | Template Method pattern; try-with-resources |
| 21 | Protected fields in base classes | "Subclasses need access" | Tight coupling between parent and child | Private fields with protected accessors |
| 22 | Serialization bypassing constructors | Implementing Serializable | Invariants not enforced | Custom serialization logic; `readResolve()` |
| 23 | Assuming `final` isn't important | "I'll make it final later" | Security issues; unintended overriding | Default to final; open only when designed for extension |
| 24 | Diamond problem with default methods | Multiple interface inheritance | Ambiguous method resolution | Explicit override; composition |

---

## 9. Senior Engineer Perspective

### How a Senior Engineer Thinks About OOP

**Beyond syntax:**
A senior engineer doesn't think about OOP as "classes and objects." They think about:

1. **Contracts** — What guarantees does this abstraction provide? What are the preconditions, postconditions, and invariants?
2. **Boundaries** — Where does this module end and the next begin? What is the cost of crossing that boundary?
3. **Change** — If requirements change (they will), what breaks? How localized is the impact?
4. **Cost** — What is the cognitive cost of this abstraction? The maintenance cost? The performance cost?

### Architectural Trade-offs

| Decision | Pros | Cons | When to Choose |
|----------|------|------|----------------|
| Deep inheritance | Reuse, polymorphism | Rigid hierarchy, fragile base class | Stable hierarchies (UI components) |
| Composition | Flexible, testable | More boilerplate (delegation) | Evolving domains (business logic) |
| Rich domain model | Cohesive, expressive | Performance overhead for simple cases | Complex business rules |
| Anemic + services | Simple, fast to develop | Procedural, low cohesion | CRUD-heavy apps, simple rules |
| Immutable objects | Thread-safe, safe aliasing | Allocation overhead | Value objects, DTOs, events |
| Mutable objects | Performant, low allocation | Thread-safety complexity | Performance-critical internal state |

### Maintainability Concerns
- **Readability:** Code is read 10x more than it's written. Optimize for the reader.
- **Discoverability:** Well-named classes and methods should make the domain model obvious.
- **Testability:** If a class is hard to test, it's poorly designed. Not vice versa.
- **Evolvability:** The best OOP design is the one that makes the next change easy.

### Operational Concerns
- **Serialization:** Domain objects crossing process boundaries need careful versioning
- **Lazy loading:** ORM proxies can cause `LazyInitializationException` in disconnected scenarios
- **Proxy limitations:** `final` methods can't be proxied by Spring AOP / Hibernate
- **Memory leaks:** Inner classes hold implicit references to outer classes
- **Startup time:** Class loading + DI container scanning can take seconds at scale

---

## 10. Interview Questions

### Beginner Questions (10)

**Q1: What are the four pillars of OOP?**
**A:** Encapsulation, Inheritance, Polymorphism, Abstraction. (Define each briefly.)

**Q2: What is the difference between a class and an object?**
**A:** A class is a blueprint/template. An object is an instance of that class — it has actual state (field values) and a memory location.

**Q3: What is encapsulation and why is it important?**
**A:** Encapsulation bundles data and methods together, hiding internal state behind a controlled interface. It protects invariants, reduces coupling, and allows implementation change without affecting callers.

**Q4: What is the difference between method overloading and method overriding?**
**A:** Overloading is compile-time polymorphism — same method name, different parameters. Overriding is runtime polymorphism — subclass provides a specific implementation of a parent method, same signature.

**Q5: What is a constructor? Can it be private?**
**A:** A constructor initializes an object's state. Yes, private constructors are used in singleton pattern, factory methods, and utility classes (prevent instantiation).

**Q6: What is the `super` keyword?**
**A:** `super` refers to the immediate parent class. Used to call parent constructors and access overridden methods.

**Q7: What is the difference between `==` and `.equals()`?**
**A:** `==` compares reference equality (memory address) for objects. `.equals()` compares content/structural equality — can be overridden.

**Q8: What is a static method?**
**A:** A method that belongs to the class, not to any instance. Called via `ClassName.method()`. Cannot access instance fields.

**Q9: What is the default access modifier in Java?**
**A:** Package-private (no explicit modifier). Accessible only within the same package.

**Q10: What is `final` in Java?**
**A:** `final` class = cannot be extended. `final` method = cannot be overridden. `final` variable = cannot be reassigned (constant).

### Intermediate Questions (20)

**Q11: What is the Liskov Substitution Principle?**
**A:** Subtypes must be substitutable for their base types without altering correctness. If `S` is a subtype of `T`, then objects of type `T` should be replaceable with objects of type `S` without breaking the program. Classic violation: `Square extends Rectangle` — changing width on a square also changes height, violating `Rectangle` behavior.

**Q12: Why should `equals()` and `hashCode()` be overridden together?**
**A:** The `hashCode()` contract says equal objects must have equal hash codes. If two objects are `equals()` but have different hash codes, they can't be found in `HashMap`/`HashSet`. Always override both.

**Q13: What is the difference between composition and aggregation?**
**A:** Composition = strong ownership, child cannot exist without parent (e.g., `House` → `Room`). Aggregation = weak has-a relationship, child can exist independently (e.g., `Department` → `Professor`).

**Q14: What is the Open/Closed Principle?**
**A:** Classes should be open for extension but closed for modification. Add new behavior through extension (inheritance, composition) rather than modifying existing, tested code.

**Q15: What is the difference between an abstract class and an interface?**
**A:** Abstract class: can have state (fields), partial implementation, single inheritance. Interface: no state (Java 7), multiple inheritance, full abstraction. Java 8+ interfaces can have `default` methods.

**Q16: What is the Dependency Inversion Principle?**
**A:** High-level modules should not depend on low-level modules. Both should depend on abstractions. Abstractions should not depend on details; details should depend on abstractions.

**Q17: How do you prevent a class from being subclassed?**
**A:** Declare it `final` in Java, `sealed` or non-`open` in C#, or use a private constructor with a static factory.

**Q18: What is the difference between shallow copy and deep copy?**
**A:** Shallow copy copies reference values — both objects share mutable child objects. Deep copy recursively copies all objects — complete independence.

**Q19: What is a covariant return type?**
**A:** An overriding method can return a more specific type than the parent method. Java 5+ feature.

**Q20: Explain the concept of a marker interface.**
**A:** An interface with no methods (`Serializable`, `Cloneable`). It marks a class as having a certain capability, detected via `instanceof`.

**Q21: What is the diamond problem and how does Java resolve it?**
**A:** When a class inherits from two interfaces with the same default method. Java requires the implementing class to override the method explicitly.

**Q22: What is the difference between `String`, `StringBuilder`, and `StringBuffer`?**
**A:** `String` is immutable. `StringBuilder` is mutable and not thread-safe (preferred). `StringBuffer` is mutable and thread-safe (synchronized, slower).

**Q23: What is a sealed class in Java 17?**
**A:** A sealed class restricts which classes can extend it. `sealed class Shape permits Circle, Rectangle`. Provides controlled inheritance.

**Q24: Explain the concept of a record in Java 14+.**
**A:** `record` is a transparent carrier of immutable data. Automatically generates constructor, `equals()`, `hashCode()`, `toString()`. It's a value class (DO NOT use `@Entity` with records).

**Q25: What is the difference between a POJO, a JavaBean, and a Spring Bean?**
**A:** POJO: Plain Old Java Object, no constraints. JavaBean: POJO with no-arg constructor, getters/setters, serializable. Spring Bean: object managed by the Spring IoC container.

**Q26: Why is it bad to return a reference to a mutable field?**
**A:** Callers can modify internal state, breaking invariants. Return a defensive copy or an unmodifiable view.

**Q27: What is the law of Demeter?**
**A:** "Don't talk to strangers." An object should only call methods on: itself, its fields, parameters, or locally created objects. Avoids deep coupling chains.

**Q28: What is a value object?**
**A:** An immutable object that has no identity — two value objects are equal if their fields are equal. Examples: `Money`, `Color`, `Address`.

**Q29: What is an entity?**
**A:** An object with a distinct identity that persists over time and changes. Two entities are equal if their IDs match, not their fields. Example: `Customer(id=123)`.

**Q30: How does the JVM handle method dispatch for `private`, `static`, and `virtual` methods?**
**A:** `private` — static dispatch (compile-time). `static` — static dispatch. Virtual — dynamic dispatch via vtable at runtime.

### Senior-Level Questions (20)

**Q31: Design a thread-safe object pool. What patterns would you use?**
**A:** Object Pool pattern + Semaphore for blocking access + `Synchronized` or `ReentrantLock` for internal bookkeeping + WeakReferences or eviction policy. Consider generics for type safety.

**Q32: How would you design an immutable class? What are the gotchas?**
**A:** 1) Declare class `final`, 2) Make all fields `final` and `private`, 3) No setters, 4) Initialize via constructor, 5) Return defensive copies in getters for mutable fields. Gotcha: `Date` field needs copying, arrays need `clone()`, collections need `Collections.unmodifiableList()`.

**Q33: Explain how you'd refactor a God class with 5000 lines and 20 dependencies.**
**A:** 1) Identify distinct responsibilities (SRP), 2) Extract interfaces for each, 3) Extract classes using Extract Class refactoring, 4) Use Facade pattern temporarily for backward compatibility, 5) Wire dependencies via DI, 6) Test each extracted class.

**Q34: When would you choose inheritance over composition?**
**A:** When the relationship is truly "is-a" AND the subclass genuinely needs to reuse the parent's implementation AND the hierarchy is stable. UI component frameworks are a good fit. For most business logic, composition wins.

**Q35: How does the JVM handle default method dispatch in interfaces?**
**A:** Default methods are compiled as static methods with a synthetic receiver parameter. At runtime, the JVM resolves the most specific override using the vtable. Diamond resolution follows: class wins over interface, most specific interface wins.

**Q36: Design a multi-tenant application's domain model. How do you isolate tenant data?**
**A:** Options: 1) Separate database per tenant, 2) Separate schema per tenant, 3) Shared table with tenant discriminator column. Entity design: `TenantAware` abstract base class with `tenantId`, injected via Spring RequestScope or interceptor.

**Q37: How do you handle optimistic locking in a distributed domain model?**
**A:** Use `@Version` attribute in JPA. When updating, JPA includes `WHERE version = :oldVersion`. If version changed, `OptimisticLockException` is thrown. Retry logic with exponential backoff.

**Q38: Explain the performance implications of deep inheritance in a high-throughput service.**
**A:** Each inheritance level adds: vtable lookup overhead (~1ns), larger object header, more class metadata, slower JIT compilation. Beyond 5 levels, consider composition. In hot paths (10k+ calls/sec), sealed/final classes are measurably faster.

**Q39: How do you handle circular dependencies in Spring beans?**
**A:** 1) Setter injection (Spring can handle this), 2) `@Lazy` on one dependency, 3) Refactor to break the cycle — introduce an interface or event-based communication. Setter + `@Lazy` is the quick fix; refactoring is the right fix.

**Q40: How does JIT inlining interact with virtual method dispatch?**
**A:** HotSpot profiles call sites. If a call site is monomorphic (always calls the same class), JIT devirtualizes and inlines. If bi/megamorphic, it uses a vtable lookup with inline cache. Writing small, non-overridden methods maximizes inlining potential.

**Q41: Design a type-safe builder pattern for a complex domain object.**
**A:** Use the stepped builder pattern — each step's interface exposes only the next valid method. Enforces compile-time correctness. Example: `OrderBuilder.addProduct().withQuantity(2).withDiscount(10%).build()`.

**Q42: How would you model a state machine using OOP principles?**
**A:** State pattern: `OrderState` interface with methods like `next()`, `cancel()`, `refund()`. Each state is a class (`PendingState`, `ShippedState`, `DeliveredState`). The `Order` delegates to its current state object.

**Q43: Explain how you'd handle serialization of a complex object graph.**
**A:** 1) DTOs for API boundaries (not domain objects), 2) Custom serialization (`writeObject`/`readObject`) for backward compatibility, 3) `@JsonIgnore` for circular references, 4) Versioning via `serialVersionUID`, 5) Test serialization compatibility in CI.

**Q44: What are the trade-offs of using records vs classes for domain objects?**
**A:** Records: immutable, concise, perfect for DTOs/value objects. Cannot extend, no custom logic (well, limited), no JPA entities. Classes: full flexibility, mutable, JPA entities. Use records for value objects and DTOs; use classes for entities and rich domain models.

**Q45: How do you implement cross-cutting concerns without violating OOP?**
**A:** Aspect-Oriented Programming (Spring AOP, AspectJ). Logging, transaction management, security, auditing are cross-cutting. AOP keeps domain objects clean while injecting behavior declaratively.

**Q46: Explain how Hibernate proxies relate to OOP principles.**
**A:** Hibernate uses dynamic proxies (or bytecode enhancement) to create lazy-loading proxies. These override methods to load data on first access. Violates LSP if the proxy behaves differently (e.g., `instanceof` on uninitialized proxy fails; calling `getClass()` returns the proxy class).

**Q47: How would you design an event-driven aggregate that publishes domain events?**
**A:** Aggregate root maintains a list of domain events (transient). Before `save()`, the repository collects events and publishes them. `AbstractAggregateRoot` in Spring Data provides this pattern. Ensures atomicity: event publication happens only if the aggregate is persisted.

**Q48: When should you use a domain event vs a service method?**
**A:** Domain event: when multiple unrelated side effects must occur (email, audit, cache invalidation), when other aggregates need to react, when you need event sourcing. Service method: when the action is synchronous and single-purpose.

**Q49: How do you evolve a domain model over multiple releases?**
**A:** 1) Never delete, deprecate, 2) Parallel models for migration, 3) Anti-corruption layer, 4) Feature toggles, 5) Adapter patterns for legacy consumers, 6) Incremental strangler fig pattern.

**Q50: Explain how you'd implement a CQRS architecture with separate read/write models.**
**A:** Write model: rich domain objects with behavior, JPA entities, transactional. Read model: flat DTOs, optimized queries, possibly denormalized. Eventual consistency via domain events → event handlers → update read model.

### Architect-Level Questions (10)

**Q51: How would you design a domain model for a multi-bank payment system handling 10k TPS?**
**A:** 1) Aggregate roots per bounded context (`Payment`, `Settlement`, `Reconciliation`), 2) Event sourcing for audit trail, 3) CQRS for read vs write separation, 4) Idempotency keys, 5) Saga pattern for distributed transactions, 6) Anti-corruption layer for each bank's model, 7) Versioned events for backward compatibility.

**Q52: Design an authorization framework that supports RBAC, ABAC, and custom policies.**
**A:** `AuthorizationService` interface. `RBACStrategy` checks role-permission mapping. `ABACStrategy` evaluates attribute-based rules (user.age > 18, resource.owner == user). `PolicyEngine` composes strategies via chain-of-responsibility. Domain model: `User`, `Role`, `Permission`, `Policy`, `Resource`. Annotations (`@PreAuthorize`) at the controller level.

**Q53: How would you modernize a legacy monolithic domain model without a rewrite?**
**A:** Strangler Fig pattern: 1) Identify bounded contexts via DDD Event Storming, 2) Wrap legacy model in anti-corruption layer, 3) Extract one bounded context at a time behind new API, 4) Route traffic progressively, 5) Keep monolithic facade until all consumers migrate. 6) Eventually decommission legacy model.

**Q54: Design a domain model for a multi-currency, multi-country e-commerce platform.**
**A:** `Money` value object (amount + currency + currency conversion). `CountrySpecificPrice` entity. `TaxCalculator` strategy per country. `ShippingRule` per region. `Order` aggregate root with country-specific validation strategy injected via DI. `PaymentMethod` per country.

**Q55: How would you handle eventual consistency in an event-driven microservices architecture?**
**A:** 1) Saga pattern for multi-step transactions, 2) Outbox pattern for reliable event publication, 3) Idempotent consumers with deduplication keys, 4) Compensating actions for failures, 5) `SagaStateMachine` with persistence, 6) Monitoring via saga logs.

**Q56: Compare domain-driven design with data-driven design in enterprise systems.**
**A:** DDD: rich domain model, behavior in entities, ubiquitous language, complex business rules. Better for systems with significant business logic (trading, insurance, healthcare). Data-driven: anemic models, service layer handles logic, simpler, faster initial development. Better for CRUD-heavy systems (admin panels, simple catalog). Both can coexist in the same system across bounded contexts.

**Q57: Design a versioned API where the domain model must support multiple API versions simultaneously.**
**A:** 1) Internal domain model (canonical), independent of API, 2) Version-specific DTOs and mappers (MapStruct), 3) Version negotiation via Accept header or URL path, 4) Adapter pattern per version, 5) Deprecation timeline for each version, 6) Backward-compatible domain changes.

**Q58: How do you ensure data consistency across aggregates in a DDD system?**
**A:** One aggregate per transaction. Use domain events for eventual consistency across aggregates. Sagas for multi-aggregate workflows. Eventual consistency is accepted — the domain model reflects "business transaction" rather than "database transaction." Compensating actions for failures.

**Q59: Design a domain model for a healthcare system with 99.999% uptime and auditing requirements.**
**A:** 1) Append-only event store for audit, 2) CQRS with separate read replicas, 3) `Patient` aggregate root with medical record events, 4) `Appointment` aggregate with scheduling rules, 5) `Provider`, `Facility` value objects, 6) Time-based versioning for clinical data, 7) Read model optimized for different queries (diagnosis search, appointment view). All mutations go through domain events.

**Q60: How would you model a workflow engine using OOP and design patterns?**
**A:** State pattern for workflow states. Chain-of-Responsibility for sequential tasks. Strategy for task execution. Visitor for workflow analysis/reporting. Observer for workflow event notifications. Composite for sub-workflows. `WorkflowEngine` FSM with persistence, recovery, and monitoring.

---

## 11. Scenario-Based Interview Questions (20)

### Scenario 1: Lombok-Generated equals/hashCode on JPA Entities
**Problem:** A developer uses `@Data` (Lombok) on a `@Entity`. After persisting and re-fetching, an entity added to a `HashSet` pre-persist cannot be found post-persist.

**Thought process:** The `id` field is null before persist and non-null after. Lombok's `hashCode()` uses the `id` field. When `id` changes, the hash bucket changes, and the object is "lost" in the `HashSet`.

**Investigation:** Check `hashCode()` implementation. Confirm it uses the database-generated `id`.

**Solution:** Use `@EqualsAndHashCode(onlyExplicitlyIncluded = true)` with `@EqualsAndHashCode.Include` on business key fields (not `id`). Or use `@Getter @Setter` instead of `@Data`.

**Interview-quality answer:** "JPA entities should not use `@Data` because `hashCode()` changes when the entity is persisted (id goes from null to a value). This breaks collections. I use `@Getter @Setter` and manually implement `equals()`/`hashCode()` based on natural/business keys."

### Scenario 2: Service Layer With No Business Logic
**Problem:** The service layer just delegates to repositories. Every "business rule" is in SQL stored procedures.

**Thought process:** This is an anemic domain model — OOP is used only as data containers; all logic is elsewhere.

**Investigation:** Check if domain objects have any behavior. If all methods are getters/setters, you have procedural code in OOP clothing.

**Solution:** Move business rules from stored procedures to domain objects where they belong. Validate, calculate, enforce invariants in the domain.

### Scenario 3: Spring AOP Can't Proxy a final Method
**Problem:** `@Transactional` on a `final` method doesn't work.

**Thought process:** Spring AOP uses JDK dynamic proxies or CGLIB proxies. CGLIB creates a subclass, but `final` methods can't be overridden.

**Solution:** Remove `final` from methods that need AOP proxying. Alternatively, use AspectJ compile-time weaving instead of Spring AOP.

### Scenario 4: Circular Dependency in a Object Graph (JSON Serialization)
**Problem:** `@JsonIgnoreProperties` not configured — infinite recursion during serialization.

**Thought process:** Bidirectional relationships cause circular references.

**Solution:** `@JsonManagedReference` / `@JsonBackReference` or `@JsonIgnore` on one side. Better: use DTOs instead of serializing entities directly.

### Scenario 5: Large Object Graph Causes OutOfMemoryError
**Problem:** Loading one `Order` loads the entire customer's 10-year history via lazy loading that triggers unnecessarily.

**Thought process:** Lazy loading + iteration across associations at the view layer loads everything.

**Investigation:** Enable SQL logging, see hundreds of queries per request.

**Solution:** Explicit fetch plans. Use DTO projections (`SELECT new OrderDTO(...)`). Close the session/entity manager before rendering.

### Scenario 6: EntityListener Can't Inject Spring Beans
**Problem:** `@PostLoad` in `@EntityListener` tries to `@Autowired` but gets null.

**Thought process:** Entity listeners are instantiated by Hibernate, not Spring. Spring DI doesn't work.

**Solution:** Register listener with `SpringBeanContainer` in Hibernate config. Or use `ApplicationContextAware` / static context holder.

### Scenario 7: NullPointerException When Using Method Chaining
**Problem:** `order.getCustomer().getAddress().getCity()` — NPE when any intermediate is null.

**Thought process:** Law of Demeter violation. Deep traversal without null checks.

**Solution:** `Optional.map()` chains or use `order.getCustomerCity()` that handles nulls internally. Better: restructure so callers don't need this chain.

### Scenario 8: Inconsistent State After Partial Save
**Problem:** Saving an `Order` with `OrderLines` fails halfway through; database is inconsistent.

**Thought process:** No transactional boundary. Partial persistence.

**Solution:** `@Transactional` on the service method with proper cascade settings. Test rollback behavior.

### Scenario 9: Unmodifiable Collection Mutation via Iterator
**Problem:** Dev returns `Collections.unmodifiableList()`, but someone gets an iterator and calls `remove()`.

**Thought process:** The list is unmodifiable — all mutating operations throw `UnsupportedOperationException`.

**Solution:** Catch the exception early in testing. Use code review to enforce immutability patterns.

### Scenario 10: Clone Method Returns Shallow Copy
**Problem:** Clone a `Customer` with an `Address` list; modifying one affects the other.

**Thought process:** `clone()` in Java is shallow by default.

**Solution:** Implement deep copy manually or use copy constructors. Prefer copy constructors over `Cloneable` — they're type-safe and don't throw `CloneNotSupportedException`.

### Scenario 11: Performance Degradation From Getters/Setters
**Problem:** Reflection-based frameworks calling setters 100k times in a loop for a batch operation.

**Thought process:** Reflection is 10-100x slower than direct access.

**Solution:** Use direct field access via `Unsafe` (internal) or code generation at compile time. MapStruct/Record-style instead of reflective getter chains.

### Scenario 12: Concurrent Modification Exception in Collections
**Problem:** Iterating a `HashMap` with multiple threads causes `ConcurrentModificationException`.

**Thought process:** `HashMap` is not thread-safe. Structural modifications during iteration throw.

**Solution:** Use `ConcurrentHashMap`, `Collections.synchronizedMap()`, or copy-on-write collections.

### Scenario 13: Hibernate N+1 Query Problem
**Problem:** Loading 100 `Orders` triggers 101 SQL queries — 1 for the list, 100 for each order's lines.

**Thought process:** Default lazy loading fetches associations on access.

**Solution:** `JOIN FETCH`, `@EntityGraph`, or batch fetching. Use DTO projections for read-only queries.

### Scenario 14: Static Method Hides Dependency
**Problem:** `EmailUtil.send()` is called from 50 places. Switching from SendGrid to AWS SES requires finding all call sites.

**Thought process:** Static utility methods create hidden dependencies.

**Solution:** `EmailService` interface with injected implementation. Tests can mock the interface.

### Scenario 15: Protected Method Exposed by Subclass
**Problem:** A protected method in a base class was meant for internal use, but a subclass makes it public.

**Thought process:** Access modifiers can be widened, not narrowed, in Java. Protected doesn't guarantee encapsulation from subclasses.

**Solution:** Use package-private + keep subclasses in the same package, or use `@Override` with documentation.

### Scenario 16: Failure to Handle Version Conflicts in Optimistic Locking
**Problem:** Two users edit the same customer record; second save silently overwrites the first.

**Thought process:** No optimistic locking. Last write wins.

**Solution:** Add `@Version` field. Handle `OptimisticLockException` with retry logic or conflict resolution UI.

### Scenario 17: DTO-Entity Mapping Leaks Across Layers
**Problem:** Entity `@JsonIgnore` annotations control DTO serialization. Adding a field to the entity breaks the API contract.

**Thought process:** No separation of concerns between persistence and API models.

**Solution:** Separate DTOs per API version. Use MapStruct for mapping. Entity never leaves the service layer.

### Scenario 18: Mocking Concrete Classes Is Hard
**Problem:** `new EmailService()` inside a method makes unit testing impossible.

**Thought process:** Hard-coded dependencies prevent substitution.

**Solution:** Constructor injection. Use interfaces. The class depends on abstraction, not concrete implementation.

### Scenario 19: Default Method Conflict in Multiple Interfaces
**Problem:** `class MyService implements A, B` where both A and B define `default void log()` — compilation error.

**Thought process:** Java resolves conflicts by requiring explicit override.

**Solution:** Override `log()` in `MyService` and delegate to either `A.super.log()` or `B.super.log()` as appropriate.

### Scenario 20: Memory Leak From Anonymous Inner Classes
**Problem:** A listener registered in a Servlet context retains a reference to a web app classloader — causes permgen/memory leak on redeploy.

**Thought process:** Anonymous inner classes hold an implicit reference to the enclosing instance. If the enclosing object is the classloader, it can't be GC'd.

**Solution:** Use static inner classes or lambda expressions that don't capture the outer instance. Deregister listeners on context destroy.

---

## 12. Debugging & Troubleshooting

### Issue 1: LazyInitializationException in Spring MVC
- **Symptoms:** Error when rendering a view that accesses a lazy-loaded association.
- **Root cause:** The Hibernate session is closed when the view renders (transaction ended before view rendering).
- **Investigation:** Enable Hibernate SQL logging; check if `org.hibernate.LazyInitializationException` stack trace shows view rendering.
- **Resolution:** 1) Open Session in View (OSIV) — simple but performance costs, 2) DTO projections with all needed data fetched in the service layer, 3) Use `@Transactional` in the controller.

### Issue 2: Object Not Found in HashSet After Persist
- **Symptoms:** An object added to a `HashSet` before persisting is not found after persisting.
- **Root cause:** `hashCode()` uses the database-generated `id`. Before persist, `id` is null. After persist, `id` has a value. The hash bucket changes.
- **Investigation:** Check `equals()` and `hashCode()` implementations. Verify they use `id`.
- **Resolution:** Use business keys (not `id`) for `equals()`/`hashCode()`. Never use mutable fields.

### Issue 3: StackOverflowError During JSON Serialization
- **Symptoms:** Infinite recursion when serializing a bidirectional relationship.
- **Root cause:** Parent serializes child, child serializes parent — loop.
- **Investigation:** Check the serialization stack trace for the repeating pattern.
- **Resolution:** `@JsonIgnore` on one side, `@JsonManagedReference`/`@JsonBackReference`, or DTOs.

### Issue 4: ClassCastException With Hibernate Proxy
- **Symptoms:** `j.l.ClassCastException: com.sun.proxy.$Proxy123 cannot be cast to CustomEntity`.
- **Root cause:** Hibernate generates a proxied subclass. Direct casting with class name fails; only interface casting works.
- **Investigation:** Check `object.getClass().getName()` to see the proxy class name.
- **Resolution:** Operate on interfaces. Use `@Proxy(lazy=false)` on the entity if absolutely needed.

### Issue 5: TransientPropertyValueException
- **Symptoms:** Saving an entity with a reference to a transient (unsaved) child entity.
- **Root cause:** Cascade type not configured or child not persisted first.
- **Investigation:** Check `cascade` annotations and the entity's persistence state.
- **Resolution:** `cascade = CascadeType.PERSIST` or `ALL`. Or explicitly save the child first.

### Issue 6: ConstraintViolationException — Optimistic Lock
- **Symptoms:** Batch update returns unexpected row count.
- **Root cause:** Another transaction modified the same entity concurrently.
- **Investigation:** Check `@Version` field and concurrent access logs.
- **Resolution:** `@Retryable` on the service method with exponential backoff. Inform user of conflict.

### Issue 7: Detached Entity Passed to Persist
- **Symptoms:** `PersistentObjectException: detached entity passed to persist`.
- **Root cause:** An entity with an existing `id` (from a deserialized form) is passed to `persist()` instead of `merge()`.
- **Investigation:** Check the entity's state before the persist call.
- **Resolution:** Use `merge()` for potentially detached entities. Use `@Transactional` consistently.

### Issue 8: N+1 Query Performance
- **Symptoms:** 100 orders load, 101 SQL queries executed.
- **Root cause:** Default lazy loading + iteration triggers per-entity queries.
- **Investigation:** Enable Hibernate SQL logging; watch for repeated identical queries.
- **Resolution:** `JOIN FETCH`, `@EntityGraph`, batch fetching, DTO projections.

### Issue 9: JDK Dynamic Proxy Limitation
- **Symptoms:** Spring AOP `@Transactional` doesn't work when the method is called from within the same class.
- **Root cause:** Self-invocation bypasses the proxy.
- **Investigation:** Check if the transaction manager reports "no active transaction."
- **Resolution:** Use `(SelfService) AopContext.currentProxy()`, extract the method into a separate bean, or use AspectJ weaving.

### Issue 10: Session Closed When Using @Async
- **Symptoms:** `LazyInitializationException` in an `@Async` method.
- **Root cause:** `@Async` runs in a different thread. The Hibernate session is not propagated.
- **Investigation:** Check the thread name in the stack trace.
- **Resolution:** Open a new session before accessing lazy data. Use `@Transactional(propagation = REQUIRES_NEW)` on the async method.

---

## 13. Comparison Section

### Inheritance vs Composition

| Aspect | Inheritance | Composition |
|--------|-------------|-------------|
| Relationship | "is-a" | "has-a" / "uses-a" |
| Reuse | Implementation reuse via extends | Behavioral reuse via delegation |
| Flexibility | Static (compile-time) | Dynamic (runtime) |
| Coupling | Tight — subclass depends on parent | Loose — depends on interface |
| Testability | Harder (requires parent) | Easier (mockable delegates) |
| Hierarchy depth | Fragile beyond 3 levels | Unlimited |
| Polymorphism | Via method overriding | Via interface implementation |
| State sharing | Protected fields accessible | Private fields, controlled access |
| Typical use | UI frameworks, base classes | Business logic, evolving domains |

**Verdict:** Prefer composition by default. Use inheritance only when:
1. Clear "is-a" relationship
2. Subclass genuinely reuses most parent behavior
3. Hierarchy is stable
4. You control both parent and child

### Abstract Class vs Interface

| Aspect | Abstract Class | Interface (Java 8+) |
|--------|---------------|---------------------|
| Constructor | Yes | No |
| State (fields) | Yes | No (static final only) |
| Method implementations | Yes (partial) | Yes (default/static) |
| Multiple inheritance | No (single class) | Yes (multiple interfaces) |
| Access modifiers | Full range | Public only |
| When to use | Shared state + partial implementation | Contract/capability definition |

**Modern guidance:** Start with an interface. Use an abstract class only when you need shared state or constructors.

### OOP vs Functional Programming

| Aspect | OOP | Functional (FP) |
|--------|-----|-----------------|
| State | Mutable (encapsulated) | Immutable (preferred) |
| Core unit | Objects | Functions |
| Composition | Object composition + inheritance | Function composition |
| Side effects | Methods modify state | Pure functions preferred |
| Concurrency | Shared mutable state (hard) | Immutable state (safe) |
| Polymorphism | Subtype polymorphism | Parametric polymorphism (generics) |
| Learning curve | Moderate (concepts) | Higher (monads, functors) |

**Hybrid approach:** Java 8+ / C# / Kotlin supports both. Use OOP for domain modeling, FP for data processing (Stream API, LINQ, optionals).

### OOP vs Procedural

| Aspect | OOP | Procedural |
|--------|-----|------------|
| Data & logic | Bundled | Separate (data structures + functions) |
| Modularity | Class-based | Function-based |
| State | Per-object | Global / shared |
| Scaling | Better for large systems | Better for small/medium |
| Testing | Easy (mock objects) | Moderate (mock functions harder) |
| Code organization | Domain-driven | Action-driven |

---

## 14. Revision Notes

### Key Concepts
- **Encapsulation:** Bundle data + behavior; hide internal state
- **Inheritance:** "is-a" relationship; code reuse + polymorphism
- **Polymorphism:** Same interface, different implementations (compile-time = overloading, runtime = overriding)
- **Abstraction:** Hide complexity, expose essentials

### SOLID Principles (Quick Reference)
| Letter | Principle | Meaning |
|--------|-----------|---------|
| **S** | Single Responsibility | One reason to change per class |
| **O** | Open/Closed | Open for extension, closed for modification |
| **L** | Liskov Substitution | Subtypes must be substitutable for base types |
| **I** | Interface Segregation | Small, focused interfaces |
| **D** | Dependency Inversion | Depend on abstractions, not concretions |

### Important Formulas/Rules
- **Law of Demeter:** A method can only call methods on: itself, its fields, its parameters, or new objects
- **Composition over inheritance:** Prefer delegation unless clear "is-a" with stable hierarchy
- **equals + hashCode:** Always override together; use business keys not database IDs
- **Immutable class recipe:** `final` class + `final` fields + no setters + defensive copies

### Rules of Thumb
- One `public` method per `if` branch → extracted into its own method/class
- If a class has "and" in its description, split it
- If a method has more than 3 parameters, create a parameter object
- If a class has more than 200 lines, it probably violates SRP
- If you're writing `instanceof`, you're probably missing polymorphism

### Interview Checkpoints
1. Know the four pillars with real code examples
2. Understand SOLID with practical violations and fixes
3. Know `equals()` / `hashCode()` contract by heart
4. Distinguish composition vs inheritance with trade-offs
5. Be ready to refactor a God class on a whiteboard
6. Know when OOP is the wrong tool (ETL, simple scripts, FP-preferable contexts)
7. Understand DDD aggregates, value objects, entities, domain events
8. Know how JVM handles object memory, vtables, GC

---

## 15. Cheat Sheet (One Page)

```
═══ OOP QUICK REFERENCE ═══════════════════════════════════════

┌─ FOUR PILLARS ──────────────────────────────────────────────┐
│ Encapsulation : hide state behind methods                   │
│ Inheritance   : class B extends A (is-a)                    │
│ Polymorphism  : same interface, diff behavior               │
│                          ↳ compile-time (overloading)       │
│                          ↳ runtime      (overriding)        │
│ Abstraction   : hide complexity, expose essentials          │
└─────────────────────────────────────────────────────────────┘

┌─ SOLID ─────────────────────────────────────────────────────┐
│ S = Single Responsibility                                   │
│ O = Open for extension, Closed for modification             │
│ L = Liskov - subtypes must be substitutable                 │
│ I = Interface Segregation - keep interfaces small           │
│ D = Dependency Inversion - depend on abstractions           │
└─────────────────────────────────────────────────────────────┘

┌─ RELATIONSHIPS ─────────────────────────────────────────────┐
│ Association  : uses-a   (Doctor ↔ Patient)                  │
│ Aggregation  : has-a    (Dept → Prof)   [weak ownership]    │
│ Composition  : owns-a   (House → Room)  [strong ownership]  │
└─────────────────────────────────────────────────────────────┘

┌─ DESIGN RULES ──────────────────────────────────────────────┐
│ Favor composition over inheritance                          │
│ Law of Demeter: don't talk to strangers                     │
│ Program to interfaces, not implementations                  │
│ Open for extension, closed for modification                 │
│ One reason to change per class                              │
│ equals + hashCode = always override together                │
└─────────────────────────────────────────────────────────────┘

┌─ COMMON MISTAKES ───────────────────────────────────────────┐
│ ❌ Public fields                    → ✅ Private + getters   │
│ ❌ Returning mutable internals      → ✅ Defensive copy      │
│ ❌ instanceof checks                → ✅ Polymorphism        │
│ ❌ Deep inheritance                 → ✅ Composition         │
│ ❌ God class                        → ✅ Split by SRP        │
│ ❌ Anemic domain model              → ✅ Add behavior        │
│ ❌ Mutable value objects            → ✅ Immutable           │
└─────────────────────────────────────────────────────────────┘

┌─ INTERVIEW TIPS ────────────────────────────────────────────┐
│ "This violates the Open/Closed principle because..."        │
│ "I'd refactor this using the Strategy pattern, which..."    │
│ "The Liskov Substitution Principle is violated because..."  │
│ "For this use case, composition is better because..."       │
│ "I'd apply DDD here: Order is an aggregate root, Money is  │
│  a value object, and OrderPlaced is a domain event."        │
└─────────────────────────────────────────────────────────────┘
```

---

## 16. Knowledge Validation

### 20 Multiple Choice Questions

**Q1: Which principle states that a class should have only one reason to change?**
- A) Open/Closed Principle
- B) Liskov Substitution Principle
- C) **Single Responsibility Principle**
- D) Dependency Inversion Principle

**Q2: What happens if you don't override `hashCode()` when overriding `equals()`?**
- A) Compilation error
- B) Runtime exception
- C) **Objects that are `equals()` may not work correctly in Hash-based collections**
- D) Nothing — it's optional

**Q3: Which relationship is strongest?**
- A) Association
- B) Aggregation
- C) **Composition**
- D) They're all the same

**Q4: Can a `private` method be overridden?**
- A) Yes, always
- B) **No, private methods are not visible to subclasses**
- C) Yes, if you annotate with `@Override`
- D) Only in Java 8+

**Q5: What is the main advantage of composition over inheritance?**
- A) **More flexible and less coupling**
- B) Simpler code
- C) Better performance
- D) Less boilerplate

**Q6: Which SOLID principle is violated by a fat interface with 20 methods?**
- A) Single Responsibility
- B) Open/Closed
- C) **Interface Segregation**
- D) Liskov Substitution

**Q7: In Java, what does `final` on a method mean?**
- A) The method returns a constant
- B) **The method cannot be overridden**
- C) The method runs in a transaction
- D) The method is optimized by JIT

**Q8: What is an anemic domain model?**
- A) A model with too many fields
- B) **A model with only data, no behavior**
- C) A model that uses inheritance poorly
- D) A model that can't be serialized

**Q9: Which pattern solves the "diamond problem" in Java?**
- A) Visitor pattern
- B) **Explicit method override in the implementing class**
- C) Singleton pattern
- D) Template method

**Q10: What does Liskov Substitution Principle ensure?**
- A) **Subtypes can replace base types without breaking correctness**
- B) Classes should be open for extension
- C) Interfaces should be small
- D) Depend on abstractions

**Q11: A `House` class creates `Room` objects in its constructor. This is:**
- A) Association
- B) Aggregation
- C) **Composition**
- D) Inheritance

**Q12: Which access modifier provides the most encapsulation?**
- A) public
- B) protected
- C) default
- D) **private**

**Q13: What is a marker interface?**
- A) **An interface with no methods**
- B) An interface with one method
- C) An interface used for annotations
- D) An interface that marks boundaries

**Q14: Which of these violates the Law of Demeter?**
- A) `customer.getOrders()`
- B) `customer.getOrder(123)`
- C) **`customer.getOrders().getTotal()`**
- D) `orderService.getCustomerOrders(customerId)`

**Q15: If class `Square extends Rectangle`, setting width on Square also changes height. This violates:**
- A) **Liskov Substitution Principle**
- B) Open/Closed Principle
- C) Interface Segregation
- D) Single Responsibility

**Q16: What is a value object?**
- A) **An object identified by its field values, not by identity**
- B) An object with a database ID
- C) A primitive wrapper
- D) A static utility

**Q17: Which of the following is NOT a benefit of encapsulation?**
- A) Hides implementation details
- B) **Improves runtime performance**
- C) Protects invariants
- D) Reduces coupling

**Q18: What does the `super` keyword do?**
- A) Creates a parent instance
- B) **Calls the parent class constructor or method**
- C) Refers to the current object
- D) Marks a method as overriding

**Q19: In Java, can an interface have a `private` method?**
- A) No
- B) **Yes, since Java 9**
- C) Only if static
- D) Only in abstract classes

**Q20: Which of the following correctly implements the Dependency Inversion Principle?**
- A) `class OrderService extends Database`
- B) **`class OrderService { private final OrderRepository repo; }`**
- C) `class OrderService { Database db = new Database(); }`
- D) `class OrderService { static Database db; }`

### 10 Coding Questions

**Q1:** Implement an immutable `Person` class with `name`, `birthDate`, and `addresses` (list).
**A:**
```java
public final class Person {
    private final String name;
    private final LocalDate birthDate;
    private final List<Address> addresses;

    public Person(String name, LocalDate birthDate, List<Address> addresses) {
        this.name = name;
        this.birthDate = birthDate;
        this.addresses = List.copyOf(addresses); // Defensive copy + immutable list
    }

    public String getName() { return name; }
    public LocalDate getBirthDate() { return birthDate; }
    public List<Address> getAddresses() { return addresses; } // Already immutable
}
```

**Q2:** Implement a thread-safe `BankAccount` class using encapsulation.
**A:**
```java
public class BankAccount {
    private final Object lock = new Object();
    private long balance;

    public void deposit(long amount) {
        synchronized (lock) {
            if (amount <= 0) throw new IllegalArgumentException();
            balance += amount;
        }
    }

    public boolean withdraw(long amount) {
        synchronized (lock) {
            if (amount > balance) return false;
            balance -= amount;
            return true;
        }
    }

    public long getBalance() {
        synchronized (lock) { return balance; }
    }
}
```

**Q3:** Refactor this code to follow OOP principles:
```java
// Given
public class OrderProcessor {
    public void process(String type, double amount) {
        if (type.equals("CREDIT_CARD")) {
            // charge card
        } else if (type.equals("PAYPAL")) {
            // redirect to PayPal
        } else if (type.equals("CRYPTO")) {
            // blockchain payment
        }
    }
}
```
**A:**
```java
public interface PaymentMethod {
    boolean process(double amount);
}

public class CreditCardPayment implements PaymentMethod {
    public boolean process(double amount) { /* charge card */ }
}

public class PayPalPayment implements PaymentMethod {
    public boolean process(double amount) { /* PayPal */ }
}

public class OrderProcessor {
    private final PaymentMethod paymentMethod;
    public OrderProcessor(PaymentMethod paymentMethod) {
        this.paymentMethod = paymentMethod;
    }
    public void process(double amount) {
        paymentMethod.process(amount);
    }
}
```

**Q4:** Implement `equals()` and `hashCode()` for an `Employee` class using business key (`employeeId`).
**A:**
```java
public class Employee {
    private String employeeId; // Business key — set at creation, never changes
    private String name;
    private String department;

    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (!(o instanceof Employee)) return false;
        Employee employee = (Employee) o;
        return Objects.equals(employeeId, employee.employeeId);
    }

    @Override
    public int hashCode() {
        return Objects.hash(employeeId);
    }
}
```

**Q5:** Demonstrate the Singleton pattern (thread-safe, lazy).
**A:**
```java
public class DatabaseConnectionPool {
    private DatabaseConnectionPool() {}

    private static class Holder {
        static final DatabaseConnectionPool INSTANCE = new DatabaseConnectionPool();
    }

    public static DatabaseConnectionPool getInstance() {
        return Holder.INSTANCE;
    }
}
```

**Q6:** Show the difference between method overloading and overriding.
**A:**
```java
// Overloading (compile-time polymorphism)
class Calculator {
    public int add(int a, int b) { return a + b; }
    public double add(double a, double b) { return a + b; }
    public int add(int a, int b, int c) { return a + b + c; }
}

// Overriding (runtime polymorphism)
interface DiscountStrategy {
    double calculate(double price);
}

class NoDiscount implements DiscountStrategy {
    public double calculate(double price) { return price; }
}

class SeasonalDiscount implements DiscountStrategy {
    public double calculate(double price) { return price * 0.9; }
}
```

**Q7:** Write a builder pattern for a `UserRegistrationRequest`.
**A:**
```java
public class UserRegistrationRequest {
    private final String email;
    private final String password;
    private final String firstName;
    private final String lastName;

    private UserRegistrationRequest(Builder builder) {
        this.email = builder.email;
        this.password = builder.password;
        this.firstName = builder.firstName;
        this.lastName = builder.lastName;
    }

    public static class Builder {
        private String email;
        private String password;
        private String firstName;
        private String lastName;

        public Builder email(String email) { this.email = email; return this; }
        public Builder password(String password) { this.password = password; return this; }
        public Builder firstName(String firstName) { this.firstName = firstName; return this; }
        public Builder lastName(String lastName) { this.lastName = lastName; return this; }

        public UserRegistrationRequest build() {
            validate();
            return new UserRegistrationRequest(this);
        }

        private void validate() {
            if (email == null || password == null) {
                throw new IllegalStateException("Email and password are required");
            }
        }
    }
}
```

**Q8:** Implement the Template Method pattern for a data import process.
**A:**
```java
public abstract class DataImporter {
    public final void importData(InputStream source) {
        openConnection();
        List<String> lines = readLines(source);
        List<Record> records = parseLines(lines);
        List<Record> validated = validate(records);
        persist(validated);
        closeConnection();
    }

    protected abstract void openConnection();
    protected abstract List<String> readLines(InputStream source);
    protected abstract List<Record> parseLines(List<String> lines);
    protected List<Record> validate(List<Record> records) { return records; } // Hook
    protected abstract void persist(List<Record> records);
    protected abstract void closeConnection();
}

class CsvImporter extends DataImporter {
    // Implement abstract methods...
}
```

**Q9:** Fix this code that violates OCP:
```java
class AreaCalculator {
    public double area(Object shape) {
        if (shape instanceof Circle) { ... }
        else if (shape instanceof Rectangle) { ... }
    }
}
```
**A:**
```java
interface Shape { double area(); }
class Circle implements Shape {
    private double radius;
    public double area() { return Math.PI * radius * radius; }
}
class Rectangle implements Shape {
    private double width, height;
    public double area() { return width * height; }
}
// AreaCalculator doesn't need to change when new shapes are added
```

**Q10:** Write a Strategy pattern for file compression.
**A:**
```java
interface CompressionStrategy {
    byte[] compress(byte[] data);
    byte[] decompress(byte[] compressed);
}

class GzipCompression implements CompressionStrategy { /* ... */ }
class ZipCompression implements CompressionStrategy { /* ... */ }

class FileCompressor {
    private final CompressionStrategy strategy;
    FileCompressor(CompressionStrategy strategy) { this.strategy = strategy; }
    byte[] compress(byte[] data) { return strategy.compress(data); }
}
```

### 10 Scenario Questions

**Scenario 1:** You're asked to add a new payment method (Stripe) to a system that currently hard-codes `CreditCardProcessor` in 15 services. How do you refactor this using OOP principles?

**Answer:** Create a `PaymentProcessor` interface. Have `CreditCardProcessor` and `StripeProcessor` implement it. Inject via DI. Use Factory or Strategy pattern for selection. This applies DIP (depend on abstraction) and OCP (open for extension).

**Scenario 2:** Your team has `OrderService` with 30 methods including `processOrder`, `cancelOrder`, `generateInvoice`, `sendEmail`, `calculateShipping`, `validateAddress`, `applyPromotion`. What's wrong and how do you fix it?

**Answer:** Violates SRP (too many responsibilities). Split into: `OrderProcessingService`, `InvoiceService`, `NotificationService`, `ShippingService`, `PromotionService`. Each class has one reason to change.

**Scenario 3:** In a high-traffic system, creating new `BigDecimal` objects for each price calculation is causing GC pressure. How do you balance OOP purity with performance?

**Answer:** Use `int` (cents) for internal calculations in hot paths. Keep `Money` value object at the API boundary. Profile to confirm the bottleneck. Apply Flyweight for frequently used values.

**Scenario 4:** Two developers disagree: one wants deep inheritance (`Vehicle -> Car -> Sedan -> SportsSedan`), another wants composition (`Car { Engine, Transmission, Brakes }`). How do you decide?

**Answer:** The inheritance chain has 4 levels — fragile. A `SportsSedan` is a `Sedan` with different behavior, but if requirements change, a hierarchy above 3 levels becomes hard to maintain. Use composition. Define `Car` with modular components. Use interfaces for shared contracts.

**Scenario 5:** A junior developer writes `if (order.getStatus() == "SHIPPED")` everywhere. How do you teach them the OOP approach?

**Answer:** This is feature envy and primitive obsession. The status logic belongs in the `Order` object or a `ShippingState` state machine. `order.ship(trackingNumber)` encapsulates the state transition. Use enum types, not strings.

**Scenario 6:** Your `User` class has `@OneToMany List<Order>`, and serializing it causes infinite recursion. How do you fix it without modifying the entity?

**Answer:** Create DTO classes specifically for the API. Map entity → DTO using MapStruct. Don't serialize entities at all. This separates persistence concerns from API concerns.

**Scenario 7:** A Spring `@Service` with `@Transactional` doesn't work when method `a()` calls `b()` within the same class. Why?

**Answer:** Self-invocation bypasses the proxy. Spring AOP creates a proxy, but `a()` calls `b()` on `this` (the actual object), not the proxy. Solution: extract `b()` into a separate `@Service` bean, or use `AopContext.currentProxy()`.

**Scenario 8:** Your domain model has `Customer extends Person` but now you need a `Customer` that is a `Company` (not a person). Inheritance breaks.

**Answer:** Refactor: `Customer` is a role, not a type. `Person implements CustomerRole`. `Company implements CustomerRole`. Favor composition: `Customer { CustomerRole role, String name, TaxInfo tax }`.

**Scenario 9:** An entity's `hashCode()` uses a mutable field. Objects in a `HashSet` become "invisible" after field mutation. How do you solve this for JPA entities?

**Answer:** Never use mutable fields for `hashCode()` or `equals()`. For JPA entities, use a stable business key (e.g., `customerNumber`, `email`) or a UUID generated at creation. Never use the database-generated `id` for hash codes.

**Scenario 10:** A legacy system has a 10,000-line `MainController` with all business logic, SQL, and HTML rendering. How do you introduce OOP incrementally?

**Answer:** Strangler Fig pattern: 1) Identify natural boundaries (authentication, orders, reporting), 2) Extract one at a time behind an interface, 3) Create domain objects for the extracted module, 4) Write tests for the extracted code, 5) Route new features through the new module. Don't rewrite — extract.
