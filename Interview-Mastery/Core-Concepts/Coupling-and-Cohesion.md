# Coupling & Cohesion

---

## 1. Executive Summary

### What Is It?
**Coupling** measures how much one module/class depends on other modules/classes. **Cohesion** measures how closely related the responsibilities of a single module/class are to each other.

These are two sides of the same coin: high cohesion + low coupling is the goal of good software design.

| | Low Coupling | High Coupling |
|--|-------------|---------------|
| **High Cohesion** | ✅ **Ideal** — Easy to maintain, test, and evolve | ⚠️ Classes are focused but too dependent on each other |
| **Low Cohesion** | ⚠️ Classes are independent but each does too much | ❌ **Worst** — Everything depends on everything; nothing is focused |

### Why Does It Exist?
These concepts exist to measure and guide design quality. Without them, systems degrade into:
- **Spaghetti code** — high coupling + low cohesion
- **Rigid systems** — changing one thing breaks many others
- **Fragile systems** — seemingly unrelated changes cause failures
- **Immobile systems** — code can't be reused because it's too entangled

### Real-World Use Cases
- **Microservices decomposition** — Services should be highly cohesive (focused on one business capability) and loosely coupled (communicate via APIs, not shared databases)
- **Package/Module design** — High cohesion within packages, low coupling between them
- **Refactoring legacy code** — Measure coupling & cohesion to identify extract candidates
- **Team ownership** — Assign teams to highly cohesive modules to reduce coordination overhead
- **Testing strategy** — Low coupling → easy mocking; high cohesion → focused tests

### When to Use It (as a design tool)
- During code reviews — "This class has low cohesion, let's split it"
- During architecture design — "These two services are tightly coupled, let's introduce an API"
- During refactoring — "This method belongs to another class (feature envy)"
- During dependency analysis — "This module has too many outgoing dependencies"

### When NOT to Obsess About It
- In very small codebases (< 1000 lines) — simplicity matters more
- In performance-critical inner loops — occasional coupling is acceptable for speed
- In glue/configuration code — Spring Boot `@Configuration` classes are naturally coupled to many beans
- Across anti-corruption layers — coupling at bounded context boundaries is expected and managed

---

## 2. Core Theory

### Coupling — From Worst to Best

#### Level 1: Content Coupling ❌ (Worst)
One module directly modifies internal data of another.

```java
// Content coupling — one class accesses private data of another
class OrderService {
    public void process(Order order) {
        order.total = order.calculateTotal(); // Direct field access
        order.status = "PROCESSED"; // Direct status mutation
    }
}
```

#### Level 2: Common Coupling ❌
Multiple modules share the same global data.

```java
// Common coupling — global shared state
public class ApplicationState {
    public static User currentUser;
    public static boolean isMaintenanceMode;
}
// 20 different classes read/write these — impossible to reason about
```

#### Level 3: External Coupling ⚠️
Modules depend on an external system (file system, database, API).

```java
// External coupling — but managed through interface
@Service
public class PaymentService {
    @Autowired
    private PaymentGateway gateway; // Depends on external system
}
```
**Note:** This is acceptable when isolated behind an interface.

#### Level 4: Control Coupling ⚠️
One module passes control flags to another, dictating its behavior.

```java
// Control coupling — flag tells the method what to do
public void processOrder(Order order, boolean sendEmail, boolean applyDiscount) {
    // ...
    if (sendEmail) { emailService.send(order); }
    if (applyDiscount) { order.applyPromotion(); }
}
```
**Better:** Multiple focused methods instead of flags.

#### Level 5: Stamp Coupling ⚡
Modules share a composite data structure but only use part of it.

```java
// Stamp coupling — passing entire object when only one field is needed
public void sendInvoice(Order order) {
    // Only uses order.getCustomerEmail()
    emailService.send(order.getCustomerEmail(), generateInvoice(order));
}
```
**Better:** `sendInvoice(CustomerEmail email)` — pass only what's needed.

#### Level 6: Data Coupling ✅ (Good)
Modules communicate through simple data parameters.

```java
// Data coupling — only necessary data is passed
public void transferFunds(AccountId from, AccountId to, Money amount) {
    // Both services only need what they use
}
```

#### Level 7: Message Coupling ✅ (Best)
Modules communicate only through messages/events. No shared state, no method calls.

```java
// Message coupling — event-driven
public class OrderService {
    @Autowired
    private EventPublisher eventPublisher;

    public void placeOrder(Cart cart) {
        Order order = new Order(cart);
        orderRepository.save(order);
        eventPublisher.publish(new OrderPlacedEvent(order.getId()));
    }
}

public class InventoryService {
    @EventHandler
    public void on(OrderPlacedEvent event) {
        // React independently — zero coupling to OrderService
    }
}
```

### Cohesion — From Worst to Best

#### Level 1: Coincidental Cohesion ❌
Elements are grouped arbitrarily in the same module.

```java
// Coincidental — "MiscellaneousUtils" with random unrelated methods
public class Utils {
    public static String formatDate(LocalDate date) { /* ... */ }
    public static double calculateTax(double amount) { /* ... */ }
    public static void sendEmail(String to, String body) { /* ... */ }
    public static String encryptPassword(String raw) { /* ... */ }
}
```

#### Level 2: Logical Cohesion ⚠️
Elements are grouped because they're logically similar (not because they work together).

```java
// Logical cohesion — "all input/output operations in one class"
public class IOOperations {
    public void readFromFile(String path) { /* ... */ }
    public void writeToDatabase(Record r) { /* ... */ }
    public String readFromApi(String url) { /* ... */ }
    public void writeToQueue(Message m) { /* ... */ }
}
```
**Better:** Separate into `FileReader`, `DatabaseWriter`, `ApiClient`, `MessagePublisher`.

#### Level 3: Temporal Cohesion ⚠️
Elements are grouped because they happen at the same time.

```java
// Temporal cohesion — "things that run at startup"
public class ApplicationInitializer {
    public void init() {
        loadConfiguration();
        connectToDatabase();
        registerShutdownHook();
        warmupCache();
        startMetricsReporter();
    }
}
```
**Better:** Each initialization concern in its own lifecycle-aware component.

#### Level 4: Procedural Cohesion ⚡
Elements are grouped because they follow a procedure, but they don't share data.

```java
// Procedural cohesion — steps of a workflow, not necessarily related
public class OrderWorkflow {
    public void processOrder(Order order) {
        validateOrder(order);       // Validation
        calculateTax(order);        // Tax calculation
        sendConfirmationEmail(order); // Notification
        updateInventory(order);     // Inventory
        generateInvoice(order);     // Document generation
    }
}
```
**Better:** Each concern extracted to its own service. The workflow orchestrates via events.

#### Level 5: Communicational Cohesion ✅
Elements operate on the same data.

```java
// Communicational cohesion — all methods work on the same data
public class Invoice {
    private Money subtotal;
    private List<LineItem> items;
    private TaxRate taxRate;

    public Money calculateTotal() { /* uses subtotal, items, taxRate */ }
    public Money calculateTax() { /* uses subtotal, taxRate */ }
    public void addItem(LineItem item) { /* modifies items */ }
    // All methods share the same data — good cohesion
}
```

#### Level 6: Sequential Cohesion ✅
Output of one element is input to another.

```java
// Sequential cohesion — data flows through stages
public class OrderImportPipeline {
    public ImportResult process(InputStream source) {
        List<String> rawLines = readLines(source);
        List<Record> parsed = parseLines(rawLines);
        List<Record> validated = validateRecords(parsed);
        return persistRecords(validated);
    }
}
```

#### Level 7: Functional Cohesion ✅ (Best)
All elements contribute to a single, well-defined function.

```java
// Functional cohesion — one clear purpose
public class TaxCalculator {
    // Every method in this class exists to calculate tax
    public Money calculateTax(Order order, TaxCode code) { /* ... */ }
    public Money calculateSalesTax(Money subtotal, Address address) { /* ... */ }
    public Money calculateVAT(Money subtotal) { /* ... */ }
    public Money calculateDuty(Money subtotal, Product product) { /* ... */ }
}
```

### The Relationship Between Coupling and Cohesion

```
High Cohesion  │  ⚠️ Isolated but       │  ✅ Ideal (Target)
               │  unfocused             │
               │                        │
Low Cohesion   │  ❌ Worst              │  ⚠️ Connected but
               │                        │  scattered
               └────────────────────────┴──────────────────
                  High Coupling            Low Coupling
```

**Key insight:** You cannot achieve low coupling without first achieving high cohesion. A class that does 20 unrelated things necessarily couples to 20 other classes.

---

## 3. Under-the-Hood Deep Dive

### Compile-Time vs Runtime Coupling

**Compile-time coupling:** Class A references Class B directly in source code.
- Java: `import com.example.B;`
- C#: `using Example.B;`
- Creates a hard dependency — if B changes, A must recompile

**Runtime coupling:** A depends on B through abstraction (interface/abstract class).
- `ServiceA` depends on `InterfaceB`, resolved at runtime
- Looser coupling — A doesn't need recompilation when B changes
- Foundation of DI containers (Spring, Guice, .NET DI)

### Measuring Coupling

| Metric | What It Measures | Formula |
|--------|-----------------|---------|
| **CBO** (Coupling Between Objects) | Number of classes a class depends on | Count of distinct classes referenced |
| **DIT** (Depth of Inheritance Tree) | Inheritance depth | Levels from root to class |
| **RFC** (Response for a Class) | Set of methods callable in response to a message | Methods in class + methods called from outside |
| **LCOM** (Lack of Cohesion of Methods) | How unrelated methods are within a class | Higher = worse cohesion |

**Tooling:**
- Java: JDepend, SonarQube, IntelliJ Metrics
- C#: NDepend, Visual Studio Code Metrics
- IDE: IntelliJ "Dependency Matrix", VS "Code Map"

### Memory/Performance Implications

| Design Quality | Impact |
|---------------|--------|
| **Low cohesion** | More objects instantiated per operation → GC pressure |
| **Stamp coupling** | Unnecessary data serialization → bandwidth/memory waste |
| **Common coupling** | Contended locks on global state → thread contention |
| **Control coupling** | Complex conditional branches → branch misprediction, poor JIT optimization |
| **High cohesion + low coupling** | Smaller, focused objects → better cache locality, clearer hot paths |

---

## 4. Production Code Examples

### 4.1 Refactoring from Tight Coupling to Loose Coupling

**BAD — Tight coupling:**
```java
@Service
public class OrderService {
    private final MySqlOrderRepository repository = new MySqlOrderRepository();
    private final SmtpEmailService emailService = new SmtpEmailService();
    private final StripePaymentGateway paymentGateway = new StripePaymentGateway();

    public Order placeOrder(Cart cart) {
        // Logic coupled to specific implementations
        Order order = new Order(cart);
        repository.save(order);                // Coupled to MySQL
        paymentGateway.charge(cart.getTotal());// Coupled to Stripe
        emailService.send(order);              // Coupled to SMTP

        // Hard to test — all real implementations run
        // Hard to change — swapping to PostgreSQL means changing this class
        // Hard to extend — adding audit log means modifying this class
        return order;
    }
}
```

**GOOD — Loose coupling:**
```java
@Service
public class OrderService {
    private final OrderRepository orderRepository;
    private final PaymentGateway paymentGateway;
    private final NotificationService notificationService;
    private final DomainEventPublisher eventPublisher;

    // Dependencies injected — not created
    public OrderService(OrderRepository orderRepository,
                        PaymentGateway paymentGateway,
                        NotificationService notificationService,
                        DomainEventPublisher eventPublisher) {
        this.orderRepository = orderRepository;
        this.paymentGateway = paymentGateway;
        this.notificationService = notificationService;
        this.eventPublisher = eventPublisher;
    }

    public Order placeOrder(Cart cart) {
        Order order = Order.create(cart);
        orderRepository.save(order);
        eventPublisher.publish(new OrderPlacedEvent(order.getId(), cart.getTotal()));

        // Side effects handled by event subscribers
        return order;
    }
}

// Payment happens via event — not direct call
@Component
public class PaymentHandler {
    @EventListener
    public void handle(OrderPlacedEvent event) {
        paymentGateway.charge(event.getTotal());
    }
}

// Email happens via event
@Component
public class EmailHandler {
    @EventListener
    public void handle(OrderPlacedEvent event) {
        notificationService.sendOrderConfirmation(event.getOrderId());
    }
}
```

### 4.2 High Cohesion — Extracting Focused Classes

**BEFORE — Low cohesion:**
```java
@Service
public class UserService {
    // 30 methods covering registration, login, password reset, profile, preferences,
    // billing, roles, permissions, audit, and reporting
    public User register(String email, String password) { /* ... */ }
    public AuthToken login(String email, String password) { /* ... */ }
    public void resetPassword(String email) { /* ... */ }
    public void updateProfile(UserId id, ProfileData data) { /* ... */ }
    public void updatePreferences(UserId id, Preferences prefs) { /* ... */ }
    public BillingInfo getBilling(UserId id) { /* ... */ }
    public void assignRole(UserId id, Role role) { /* ... */ }
    public boolean checkPermission(UserId id, Permission perm) { /* ... */ }
    public AuditLogEntry[] getAuditLog(UserId id, DateRange range) { /* ... */ }
    public Report generateUserReport(DateRange range) { /* ... */ }
    // ... 20 more methods
}
```

**AFTER — High cohesion:**
```java
// Each class has ONE reason to change
@Service
public class UserRegistrationService {
    public User register(RegistrationRequest request) { /* registration only */ }
}

@Service
public class AuthenticationService {
    public AuthToken login(LoginRequest request) { /* login only */ }
    public void initiatePasswordReset(Email email) { /* password reset only */ }
}

@Service
public class UserProfileService {
    public void updateProfile(UserId id, ProfileData data) { /* profile only */ }
    public void updatePreferences(UserId id, Preferences prefs) { /* preferences only */ }
}

@Service
public class UserBillingService {
    public BillingInfo getBilling(UserId id) { /* billing only */ }
}

@Service
public class AuthorizationService {
    public void assignRole(UserId id, Role role) { /* authorization only */ }
    public boolean checkPermission(UserId id, Permission perm) { /* permissions only */ }
}

@Service
public class UserReportingService {
    public Report generateUserReport(DateRange range) { /* reporting only */ }
}
```

### 4.3 Measuring and Enforcing — Production Pattern

```java
// Strategy: Define coupling boundaries with ArchUnit (Java)
// This test fails if coupling rules are violated

@RunWith(ArchUnitRunner.class)
public class ArchitectureTest {

    @Test
    public void services_should_not_depend_on_controllers() {
        classes().that().resideInAPackage("..service..")
            .should().onlyDependOnClassesThat()
            .resideInAnyPackage("..service..", "..domain..", "java..", "org.springframework..")
            .check(importedClasses);
    }

    @Test
    public void domain_objects_should_have_high_cohesion() {
        // Domain objects should only access their own fields
        classes().that().resideInAPackage("..domain..")
            .and().areAnnotatedWith(Entity.class)
            .should().haveOnlyAccessorsThatAccessTheirOwnFields()
            .check(importedClasses);
    }

    @Test
    public void repositories_should_only_be_used_by_services() {
        classes().that().resideInAPackage("..repository..")
            .should().onlyBeAccessed()
            .byClassesThat().resideInAnyPackage("..service..", "..repository..")
            .check(importedClasses);
    }
}
```

---

## 5. Real-World Scenarios (10)

### Scenario 1: "But It Works on My Machine" — Tightly Coupled Configuration
**Problem:** Every microservice reads configuration from hard-coded environment variable names. Changing one variable name requires updating 20 services.

**Analysis:** Common coupling through environment variables. All services depend on the same global configuration keys.

**Solution:** Centralized configuration service (Spring Cloud Config / Azure App Configuration). Each service requests its config by key prefix. Config changes are managed in one place.

**Why it works:** Reduces common coupling to a single, versioned, auditable config source.

### Scenario 2: The God Package — Low Cohesion
**Problem:** A Java package `com.company.utils` has 80 classes: `StringUtils`, `DateUtils`, `EmailUtils`, `PaymentUtils`, `ShippingUtils`, `PdfUtils`... Everything depends on utils.

**Analysis:** Coincidental cohesion — unrelated utilities grouped by "we don't know where else to put them."

**Solution:** Split into domain-specific packages: `com.company.string`, `com.company.email`, `com.company.payment`, `com.company.shipping`, `com.company.document`. Each package has clear purpose.

**Alternative:** If truly generic, use separate Maven/Gradle modules: `foundation-string`, `foundation-email`, etc.

### Scenario 3: API Gateway Becoming God Class
**Problem:** An API Gateway has 500 routes, all defined in one `RouteConfig` class with inline authentication, rate limiting, and transformation logic.

**Analysis:** Low cohesion (routing + auth + transformation + rate limiting in one class) leads to high coupling (gateway coupled to every backend).

**Solution:** Plugin architecture. Authentication plugins, rate-limit plugins, transform plugins. Each is a highly cohesive module. Gateway orchestrates.

### Scenario 4: Shared Database Between Microservices
**Problem:** Six microservices all access the same `orders` table directly.

**Analysis:** Content coupling via shared database. Schema changes must coordinate across six teams. Impossible to deploy independently.

**Solution:** Each microservice owns its data. Order service owns `orders` table; other services call Order Service API.

### Scenario 5: DTO Overuse Creating Stamp Coupling
**Problem:** Every method passes `UserDTO` even when it only needs `user.getEmail()`. Changing `UserDTO` breaks 100 methods.

**Analysis:** Stamp coupling — passing more data than needed.

**Solution:** Pass only required parameters. For simple needs, pass `Email`. For complex needs, create purpose-specific interfaces.

### Scenario 6: Switch Statement Overload (Low Cohesion)
**Problem:** A `NotificationService` has a 200-line switch for 15 notification types.

**Analysis:** Low cohesion — notification type logic should be in the type, not the service.

**Solution:** Strategy pattern. Each notification type is its own class. `NotificationService` only orchestrates.

### Scenario 7: Framework Lock-In
**Problem:** Every class extends `BaseController` which imports Spring MVC classes. Switching to a different framework means rewriting everything.

**Analysis:** High coupling to framework base classes.

**Solution:** Separate framework code from business code. Controllers are thin; business logic lives in framework-agnostic services.

### Scenario 8: Service Layer That Does Nothing
**Problem:** Services are anemic — they just call repositories. Controllers call services that call repositories. Three layers of indirection with no value.

**Analysis:** Low cohesion in the service layer — it has no business logic, no cohesion.

**Solution:** Remove anemic services. Let controllers call repositories directly for simple CRUD. Add services only when business logic exists.

### Scenario 9: Feature Envy Across Modules
**Problem:** `OrderService` has 15 methods that compute things from `Customer` data. Every time `Customer` changes, `OrderService` breaks.

**Analysis:** Feature envy — `OrderService` is more interested in `Customer` data than `Order` data.

**Solution:** Move those methods into `Customer` (where they belong). `OrderService` calls `customer.calculateLifetimeValue()`.

### Scenario 10: Test Suite That Takes 6 Hours
**Problem:** Every test creates the full Spring context because beans are tightly coupled.

**Analysis:** High coupling forces bootstrap of entire system for every test.

**Solution:** Architectural refactoring: interfaces + test doubles. Focused unit tests with mocks. Integration tests only for boundary modules.

---

## 6. Performance Considerations

| Design Quality | Performance Impact |
|---------------|-------------------|
| Stamp coupling | Excess data serialization → bandwidth + CPU waste |
| Common coupling | Synchronized access to shared state → thread contention |
| Control coupling | Complex conditionals → branch misprediction |
| Low cohesion | Too many objects/operations per request → GC pressure |
| Tight coupling (runtime) | Reflection-based DI resolution → startup time cost |

### Optimization Strategies
1. **Profile before optimizing** — Coupling/cohesion improvements usually help maintainability first; performance is secondary
2. **Hot paths should prefer data coupling** — Message coupling via events adds latency; use direct data coupling for synchronous hot paths
3. **Batch operations** — Low cohesion often leads to N+1 object creation per request; batch operations reduce overhead
4. **Startup optimization** — High coupling through component scanning increases startup time; explicit wiring reduces it

---

## 7. Security Considerations

| Issue | Coupling/Cohesion Factor | Mitigation |
|-------|-------------------------|------------|
| Sensitive data leak | Stamp coupling — passing too much data | Minimal data transfer; DTOs with only needed fields |
| Privilege escalation | Common coupling — shared mutable state | Isolate security contexts; no shared states between trust boundaries |
| Deserialization attacks | Tight coupling to external libraries | Abstract library behind interface; validate deserialized data |
| Audit gaps | Low cohesion — audit scattered across classes | High-cohesion audit module that intercepts all operations |
| Supply chain | High coupling to many dependencies | Limit dependencies; use interfaces for abstraction |

---

## 8. Common Mistakes (20)

| # | Mistake | Why It Happens | Consequence | Fix |
|---|---------|---------------|-------------|-----|
| 1 | God classes | "One more method" mentality | Low cohesion, high coupling, impossible to test | Extract classes by responsibility |
| 2 | Shared mutable singletons | Convenience | Common coupling; race conditions | Immutable singletons or DI |
| 3 | Passing entire objects for one field | Lazy method signatures | Stamp coupling; tight dependency | Pass only required values |
| 4 | Boolean parameter flags | "Reuse" | Control coupling; unclear API | Specific methods per behavior |
| 5 | JDBC code mixed with business logic | Quick and dirty | Content coupling (direct DB access) | Repository abstraction |
| 6 | Everything in a `utils` package | No organization | Coincidental cohesion | Domain-specific packages |
| 7 | Extending framework classes | "That's how the tutorial did it" | High coupling to framework | Composition over inheritance |
| 8 | Static utility methods everywhere | "DRY" taken too far | Hidden coupling; untestable | Injected services |
| 9 | Circular dependencies | No architecture planning | Compile/runtime issues; untestable | Dependency inversion |
| 10 | Event-driven overuse for simple flows | "It's decoupled!" | Unnecessary complexity + debugging nightmare | Direct calls for synchronous flows |
| 11 | Magic strings for feature flags | Quick feature toggles | Common coupling; untestable | Type-safe enums + config service |
| 12 | Anemic domain model | "Data class is cleaner" | Low cohesion — data separate from behavior | Move logic into domain objects |
| 13 | Fat interfaces | "All payment methods support these" | Interface pollution; implementing classes implement empty methods | Interface segregation |
| 14 | Catch-all handler classes | "One place for cross-cutting" | Low cohesion | AOP or dedicated interceptor |
| 15 | Mixing concerns in constructors | Convenience | Constructor does too much; hard to test | Separate construction from initialization |
| 16 | Feature envy | Developer unfamiliar with domain | Low cohesion — wrong class has the logic | Move behavior to the correct class |
| 17 | Copy-paste reuse | Deadlines | Duplication leads to divergent evolution | Extract shared code; reduce coupling |
| 18 | Over-engineering with abstractions | "Future-proofing" | Unnecessary indirection; harder to understand | YAGNI — add abstractions when needed |
| 19 | Repository pattern as anti-pattern for simple CRUD | "Best practice" | Unnecessary coupling to interfaces | Use direct access for simple cases |
| 20 | Not measuring coupling | "It feels fine" | Gradual decay into tightly coupled mess | Automated architecture tests (ArchUnit) |

---

## 9. Senior Engineer Perspective

### How a Senior Engineer Thinks About Coupling & Cohesion

**1. The Cost of Coupling**
Every dependency is a liability. When Module A depends on Module B:
- Changes to B may break A
- Testing A requires setting up B
- Understanding A requires understanding B
- Deploying A may require deploying B

**2. The Value of Cohesion**
A cohesive module is understandable on its own. A developer can read 3–5 classes and understand the module's purpose. Low cohesion forces context-switching across the codebase.

**3. Architectural Trade-offs**

| Strategy | Coupling | Cohesion | When |
|----------|----------|----------|------|
| Monolith | High (everything coupled) | Medium | Early-stage, small team |
| Modular monolith | Low within boundaries | High per module | Established product, 2–5 teams |
| Microservices | Very low across services | Very high per service | 10+ teams, independent deployment |
| Event-driven | Lowest (temporal decoupling) | High per handler | Async workflows, cross-system |

**4. Operational Concerns**
- **High coupling → deployment coordination** — Must deploy A and B together
- **Low cohesion → on-call confusion** — A single JIRA ticket requires 3 teams to investigate
- **Coupling to slow dependencies** — Thread pool exhaustion, cascading failures
- **Cohesion defines team boundaries** — Conway's Law: system structure mirrors org structure

**5. Rules of Thumb**
- If changing one class requires changing 5 others → coupling is too high
- If you can't describe a class's purpose in one sentence → cohesion is too low
- If a method has more than 2 parameters that are objects containing many fields → possible stamp coupling
- If a method has boolean flags → control coupling, split it

---

## 10. Interview Questions

### Beginner Questions (10)

**Q1: What is coupling in software engineering?**
**A:** Coupling is the degree of interdependence between software modules. Low coupling means modules are independent; high coupling means modules depend heavily on each other.

**Q2: What is cohesion?**
**A:** Cohesion measures how closely related the elements within a single module/class are to each other. High cohesion means elements work together toward a single purpose.

**Q3: Why is low coupling desirable?**
**A:** Low coupling makes modules independent, easier to change, test, and deploy independently. It reduces the ripple effect of changes.

**Q4: What is the ideal relationship between coupling and cohesion?**
**A:** High cohesion + low coupling. Each module has a focused purpose (high cohesion) and minimal dependencies on other modules (low coupling).

**Q5: What is a God class?**
**A:** A class with too many responsibilities (low cohesion). It typically has hundreds of methods, many dependencies (high coupling), and is very difficult to maintain or test.

**Q6: What is data coupling?**
**A:** The best form of coupling. Modules share only simple data parameters. Each module uses only what it receives.

**Q7: What is content coupling?**
**A:** The worst form of coupling. One module directly modifies internal data of another module (e.g., accessing `private` fields or modifying internal variables).

**Q8: What is coincidental cohesion?**
**A:** The worst form of cohesion. Elements are grouped arbitrarily (e.g., `Utils.java` with completely unrelated methods).

**Q9: What is functional cohesion?**
**A:** The best form of cohesion. All elements contribute to a single, well-defined function.

**Q10: How does the Single Responsibility Principle relate to cohesion?**
**A:** SRP directly supports high cohesion — a class should have one reason to change, meaning its elements are all focused on that one responsibility.

### Intermediate Questions (20)

**Q11: What's the difference between stamp coupling and data coupling?**
**A:** Stamp coupling passes an entire data structure when only part of it is needed. Data coupling passes only the required parameters. Stamp coupling creates unnecessary dependency on the data structure.

**Q12: How would you measure coupling in a Java project?**
**A:** Use tools like JDepend, SonarQube, or IntelliJ Metrics. Key metrics: CBO (Coupling Between Objects), Afferent/Efferent Coupling. Or use ArchUnit for automated architecture tests.

**Q13: What's the difference between logical cohesion and communicational cohesion?**
**A:** Logical cohesion groups elements by category (e.g., "all validation"), while communicational cohesion groups elements that operate on the same data (e.g., all methods on `Invoice` that use `Invoice.items`).

**Q14: Can a class have both high cohesion and high coupling?**
**A:** Yes. Example: a `TaxCalculator` has high cohesion (all tax logic) but high coupling to 10 different tax authority APIs. It's focused but has many dependencies.

**Q15: How does the Dependency Inversion Principle reduce coupling?**
**A:** DIP says depend on abstractions, not concretions. This means high-level modules depend on interfaces, not on specific implementations. Changes to implementations don't affect high-level modules.

**Q16: What's the relationship between cohesion and testability?**
**A:** High cohesion improves testability — a focused class has fewer dependencies, so it's easier to set up and test. Low cohesion forces tests to mock many unrelated dependencies.

**Q17: How do microservices aim to improve coupling and cohesion?**
**A:** Microservices aim for:
- High cohesion: each service focuses on one business capability
- Low coupling: services communicate via APIs/events, not shared databases

**Q18: What is feature envy and how does it relate to cohesion?**
**A:** Feature envy is when a method is more interested in another class's data than its own. It indicates low cohesion — the method belongs in the envied class.

**Q19: How would you refactor a class with low cohesion but you're afraid of breaking changes?**
**A:** Use Strangler Fig pattern: 1) Create new focused classes, 2) Delegate from the old class to new classes, 3) Gradually update callers, 4) Remove old class.

**Q20: What is a "leaky abstraction" and how does it relate to coupling?**
**A:** A leaky abstraction exposes implementation details of the underlying system. It increases coupling because callers depend on those details. Example: Hibernate's `LazyInitializationException` leaks the ORM session management.

**Q21: How do you handle coupling to third-party libraries?**
**A:** Create an anti-corruption layer (interface + adapter) around the library. Your code depends on the interface, not the library. Changes to the library are isolated to the adapter.

**Q22: What is common coupling and why is it dangerous?**
**A:** Common coupling is when multiple modules share the same global data. It's dangerous because any module can modify the shared state, making it impossible to reason about data flow.

**Q23: How would you detect God classes using metrics?**
**A:** Look for: high LCOM (Lack of Cohesion of Methods), high CBO (Coupling Between Objects), high RFC (Response for a Class), and large method count (>20 methods).

**Q24: What is the difference between logical and physical coupling?**
**A:** Logical coupling is about source code dependencies (imports). Physical coupling is about deployment dependencies (JARs, DLLs, services).

**Q25: How does Spring's DI container help with coupling?**
**A:** DI container manages dependency wiring externally. Classes declare their dependencies (via constructor), and the container injects them. This reduces coupling because:
- Classes don't create their own dependencies
- Binding is configurable (different implementations in different environments)
- Testing can inject mocks

**Q26: What is "Inappropriate Intimacy" in coupling terms?**
**A:** When one class knows too much about another's internal implementation. For example, always calling `getX().getY().getZ()` chain. Solution: Law of Demeter.

**Q27: How do events reduce coupling compared to direct method calls?**
**A:** With events, the publisher doesn't know who subscribes. No direct dependency. The subscriber may be in a different service, even a different system. Temporal decoupling — they don't need to run at the same time.

**Q28: What is the downside of excessive decoupling?**
**A:** Over-decoupling leads to indirection — interfaces everywhere, event pipelines with no clear flow, difficulty debugging (cause-effect isn't obvious), and unnecessary complexity.

**Q29: How would you explain coupling and cohesion to a non-technical stakeholder?**
**A:** "Think of a car: high coupling would mean replacing the tires requires rewiring the engine. Low cohesion would mean the steering wheel also controls the radio. Good design means each part does one thing well, and changing one part doesn't require changing others."

**Q30: What's the relationship between coupling and Conway's Law?**
**A:** Conway's Law says systems mirror communication structures. High coupling between modules leads to high coordination needs between teams. Low coupling enables independent team ownership.

### Senior-Level Questions (20)

**Q31: Design an architecture test suite that enforces coupling rules across 50 microservices.**
**A:** Use a centralized testing framework (ArchUnit or custom). Define rules per bounded context: "services in `payment` context may only depend on `payment.api`, not `shipping.api`". Run in CI pipeline. Fail builds on violations.

**Q32: How would you measure and improve cohesion across a 5-year-old 500K LOC monolith?**
**A:** 1) Use static analysis to measure LCOM per class, 2) Identify clusters of classes with high afferent coupling, 3) Apply DDD bounded context discovery, 4) Extract modules by context, 5) Enforce package boundaries with ArchUnit, 6) Track trend over time.

**Q33: How do you balance the need for low coupling with performance requirements?**
**A:** 1) Identify hot paths via profiling, 2) In hot paths, allow tighter coupling for performance (direct calls over events), 3) Isolate hot paths behind a clear module boundary, 4) Use CQRS — decoupled writes (events), tightly coupled reads (direct joins).

**Q34: A legacy system has 1000 static methods that access shared mutable state. How do you reduce coupling without rewriting?**
**A:** 1) Wrap shared state in a module with explicit interface, 2) Extract one static method at a time into an injectable service, 3) Use a gradual strangler pattern, 4) Add synchronized blocks to prevent race conditions in the interim, 5) Write characterization tests before each extraction.

**Q35: How do you prevent coupling creep in a growing microservice architecture?**
**A:** 1) **Explicit API contracts** — versioned, published, reviewed, 2) **No shared databases**, 3) **Synchronous calls only through API gateways**, 4) **Async events for cross-service workflows**, 5) **Contract testing** (PACT) between services, 6) **Architecture tests** in CI, 7) **Team ownership maps** — each service owned by one team.

**Q36: What coupling patterns would you use for a multi-cloud deployment (AWS + Azure)?**
**A:** 1) Abstract cloud services behind interfaces (`ObjectStorage`, `QueueService`, `Database`), 2) Implement adapters for AWS (S3, SQS, RDS) and Azure (Blob, Queue Storage, SQL), 3) Use DI to inject the right implementation per environment, 4) Test against cloud-agnostic test doubles.

**Q37: How do you design a module so that it can be extracted into a microservice later without changes?**
**A:** 1) High cohesion — the module owns one business capability, 2) Low coupling — depends only on interfaces/events, not internal classes, 3) Owns its data — no shared database tables, 4) Published API — versioned interface, 5) No framework-specific annotations leaking into the domain.

**Q38: How does C4 model help with visualizing coupling and cohesion?**
**A:** C4 (Context, Containers, Components, Code) provides hierarchical views. System Context shows system-level coupling. Container diagram shows service coupling. Component diagram shows intra-module coupling. Class diagram shows code-level cohesion.

**Q39: How would you handle a situation where two modules have high functional cohesion but are tightly coupled?**
**A:** They likely belong together as one module. If they're both cohesive but coupled, they're probably part of the same bounded context. Merge them rather than force decoupling.

**Q40: What's the difference between semantic coupling and syntactic coupling?**
**A:** Syntactic coupling: A depends on B's interface/API. Semantic coupling: A depends on B's behavior (timing, ordering, side effects). Semantic coupling is harder to detect — even with clean interfaces, if A relies on B processing events in order, they're semantically coupled.

**Q41: How do you handle temporal coupling in event-driven systems?**
**A:** Temporal coupling is when the ordering of events matters. Solutions: 1) Event versioning with timestamps, 2) Event sourcing (replay from beginning), 3) Saga pattern with compensating actions, 4) Idempotent consumers.

**Q42: Design a package structure for a payment system with high cohesion and low coupling.**
**A:**
```
com.company.payment/
├── api/            — Public interfaces
├── domain/         — Aggregates, value objects, domain events
├── infrastructure/ — Database, external API adapters
├── application/    — Use cases / services
└── test/           — Unit + integration tests
```
Dependency rule: `api` → nothing; `domain` → nothing; `application` → `domain` + `api`; `infrastructure` → `api` + `domain`.

**Q43: How do you detect and fix "shotgun surgery" in terms of coupling?**
**A:** Shotgun surgery = one change affects many classes. Detection: git history analysis (when a commit touches many files, those files are coupled). Fix: move related code into a single module (increase cohesion) or introduce abstraction.

**Q44: What's your approach to coupling when building a library vs an application?**
**A:** Libraries: extremely low coupling — depend on nothing or only on stable foundations (JDK, .NET BCL). Applications: reasonable coupling is fine — Spring Boot services will couple to Spring, that's acceptable.

**Q45: How do you explain the cost of coupling to junior developers?**
**A:** "Every dependency you add is a promise to update this code when that code changes. It's also a promise that bugs in that code will be bugs in your code. The fewer promises you make, the less maintenance you have."

**Q46: How does domain-driven design's bounded context concept relate to coupling and cohesion?**
**A:** Bounded contexts are the ultimate expression of high cohesion + low coupling. Each bounded context is highly cohesive (focused on one domain) and loosely coupled to other contexts (communicates via events/APIs).

**Q47: When is tight coupling acceptable?**
**A:** 1) Inside a bounded context: classes within are naturally coupled, 2) Glue code (config, wiring), 3) Framework code that your app is built on (Spring, ASP.NET), 4) Performance-critical paths where abstraction costs too much.

**Q48: How do you handle coupling to test infrastructure?**
**A:** 1) TestContainers for databases — decouple from specific DB setups, 2) WireMock for external APIs, 3) Embedded Kafka/RabbitMQ for message queues. These are infrastructure-level decoupling strategies.

**Q49: What metrics do you use to track coupling over time in CI/CD?**
**A:** 1) CBO per class (trend line), 2) Number of modules with afferent coupling > threshold, 3) Cyclic dependency count, 4) Testability score (how many classes need mocking per test), 5) Build time (more coupling = more recompilation).

**Q50: Design a migration strategy from a tightly coupled monolith to a modular monolith.**
**A:** 1) **Characterize** — measure current coupling and cohesion, 2) **Boundaries** — identify bounded contexts via event storming, 3) **Interfaces** — extract interfaces between contexts, 4) **Data isolation** — separate database schemas, 5) **Gateways** — replace direct calls with API calls, 6) **Test** — contract tests at each extracted boundary, 7) **Validate** — metrics show coupling decreased, cohesion increased.

### Architect-Level Questions (10)

**Q51: How would you design a coupling budget for a 1000-microservice architecture?**
**A:** Each service has a coupling budget:
- Max 5 synchronous dependencies (REST/gRPC)
- Max 10 async event subscriptions
- Max 3 shared libraries
- Zero shared databases
- API version compatibility window: 2 versions
Budget violations trigger architecture review.

**Q52: How do you align organization structure with coupling and cohesion goals?**
**A:** Conway's Law: each team owns one or more highly cohesive bounded contexts. Two teams should never own the same module (increases coupling overhead). Teams are decoupled via API contracts. Cross-team changes require API versioning, not code changes.

**Q53: Design a system where 5 teams can work independently without merge conflicts or coordination.**
**A:** 1) **Bounded context per team** — each team owns their domain, 2) **Open API first** — API contracts published before implementation, 3) **Contract testing** — PACT tests verify API compatibility, 4) **Shared kernel** — minimal, versioned, change-controlled, 5) **Feature flags** — each team ships independently, 6) **Event catalog** — cross-team communication via documented events.

**Q54: How would you handle a distributed monolith disguised as microservices?**
**A:** Detection: services share databases, deploy together, have synchronous request chains 5+ deep, can't deploy independently. Fix: 1) Database isolation per service, 2) Eventual consistency for cross-service data, 3) API gateway for synchronous flows, 4) Team ownership changes.

**Q55: Compare event-driven architecture vs synchronous API in terms of coupling.**
**A:** Event-driven: temporal decoupling (publisher doesn't wait for subscriber), resilient (queue buffers failures), but harder to debug and trace. Synchronous API: tight temporal coupling (caller blocks), simple to debug, but cascading failures. Use events for cross-domain workflows, sync APIs for within-domain operations.

**Q56: How do you ensure high cohesion in a CQRS/Event Sourcing system?**
**A:** Commands (write side) are highly cohesive per aggregate — one command handler modifies one aggregate. Queries (read side) are highly cohesive per view — one projection handler updates one read model. No mixing of read/write concerns.

**Q57: How would you evolve a data mesh architecture using coupling and cohesion principles?**
**A:** Each domain owns its data (high cohesion: data + logic together). Data is shared via API/event contracts (low coupling). No shared data lakes. Domain data products have clear ownership and service-level objectives.

**Q58: What coupling patterns exist at the infrastructure level (networks, databases, message brokers)?**
**A:** 1) **Network coupling** — service mesh (istio) vs direct pod-to-pod, 2) **Database coupling** — shared DB (tight) vs per-service DB (loose), 3) **Message broker** — shared broker (medium) vs per-domain broker (loose), 4) **Schema registry** — tightly couples producers to schema versioning.

**Q59: How does the stability of dependencies affect coupling decisions?**
**A:** Depend on stable abstractions (Low Coupling principle). Stable = JDK/.NET core, well-established libraries. Unstable = experimental libraries, internal projects undergoing refactoring. Abstract unstable dependencies behind interfaces.

**Q60: Design a platform that allows 100 teams to share infrastructure without creating coupling.**
**A:** 1) **Platform APIs** — infrastructure exposed via APIs, not shared config, 2) **Tenant isolation** — logical or physical per-team separation, 3) **Self-service** — teams provision their own resources, 4) **Guardrails** — automated policy enforcement, 5) **Observability** — distributed traces across teams, 6) **Governing body** — platform team that manages shared evolution.

---

## 11. Scenario-Based Interview Questions (10)

### Scenario 1: Refactoring the God Utils Package
**Problem:** `com.company.util` has 200 classes used across 80 microservices. Any change to any util class requires re-deploying all 80 services.

**Solution:** Split into domain-focused packages. Move string utilities to `com.company.text`, date utilities to `com.company.time`, etc. Each package becomes its own Maven module. Services depend only on what they need.

### Scenario 2: New Payment Provider Integration
**Problem:** The code has `StripePaymentGateway` referenced in 15 places. Adding PayPal requires touching all 15.

**Solution:** Extract `PaymentGateway` interface. The 15 locations depend on the interface. Add `PayPalPaymentGateway` implementing the interface. Inject via DI config.

### Scenario 3: Microservices Share a Database
**Problem:** Two services (Order and Inventory) share the `inventory` table. Inventory schema change requires coordinating two deployments.

**Solution:** Inventory service owns the table. Order service calls Inventory API. This increases latency but decouples deployment.

### Scenario 4: Feature Envy in Reports
**Problem:** `ReportService.generateUserReport()` calls 15 getters on `User`, 10 on `Order`, and 8 on `Payment`.

**Solution:** Move report generation into each domain object. `User.toReport()`, `Order.toReport()`, `Payment.toReport()`. `ReportService` orchestrates composition only.

### Scenario 5: Service Layer Becoming God Class
**Problem:** `OrderService` started as order processing. Now it handles validation, pricing, tax, inventory, shipping, email, invoice, fraud detection — 2000 lines, 15 dependencies.

**Solution:** Extract each cross-cutting concern into its own service. Use events for side effects. `OrderService` focuses on order orchestration.

### Scenario 6: APIs That Take Too Much
**Problem:** `OrderResponse` has 40 fields. Every API consumer gets everything regardless of need. Bandwidth waste, coupling to full schema.

**Solution:** GraphQL for flexible queries or versioned DTOs per consumer. Better: use the Interface Segregation Principle — each consumer gets a tailored response.

### Scenario 7: Third-Party Library Obsolescence
**Problem:** Direct use of deprecated `Apache HttpClient` in 500 places. Migrating to `OkHttp` requires 500 changes.

**Solution:** Create `HttpClient` interface. Wrap Apache in one adapter class. All 500 places depend on the interface. Migration is one adapter rewrite.

### Scenario 8: Event Overload
**Problem:** A simple user registration publishes 15 events. Subscribers are hard to find. Debugging requires checking 20 different event handlers.

**Solution:** Use events only for cross-domain concerns. Within the same bounded context, use direct method calls. Document events in an event catalog.

### Scenario 9: Circular Package Dependency
**Problem:** `com.company.order` imports from `com.company.customer` and vice versa. Compilation order impossible.

**Solution:** Extract the common types to `com.company.core`. `order` depends on `core`, `customer` depends on `core`. No cross-dependency.

### Scenario 10: All Tests Require Full Context
**Problem:** Every unit test loads the entire Spring Boot application context because of tight coupling.

**Solution:** Extract interfaces. Use constructor injection. Write focused unit tests with mocks. Integration tests only for the integration boundaries.

---

## 12. Debugging & Troubleshooting

### Issue 1: Build Order Failures in Multi-Module Projects
- **Symptom:** Maven/Gradle can't resolve compile order
- **Root cause:** Circular module dependencies
- **Investigation:** `jdepend -file` shows cycles
- **Resolution:** Extract shared types into a new module. Break the cycle with interfaces.

### Issue 2: Test Suite Takes 6 Hours in CI
- **Symptom:** CI pipeline too slow
- **Root cause:** All tests create full application context due to tight coupling
- **Investigation:** Check test class dependencies. If every test creates everything, coupling is the problem.
- **Resolution:** Interface-based design + mocks. Only integration tests load context.

### Issue 3: One Schema Change Breaks 50 Services
- **Symptom:** Adding a column to `customers` table breaks 50 services
- **Root cause:** All services access the same database table directly
- **Investigation:** Git grep for `customers` across repositories
- **Resolution:** Each service owns its data. Other services call the owning service's API.

### Issue 4: Feature Toggle Everywhere
- **Symptom:** `if (featureToggle.isEnabled("new-checkout"))` in 200 places
- **Root cause:** Control coupling — feature flags scattered
- **Investigation:** Grep for feature toggle references
- **Resolution:** Strategy pattern for toggled behavior. One toggle point, polymorphic implementations.

### Issue 5: Can't Upgrade Spring Boot Version
- **Symptom:** Upgrading Spring Boot breaks 30 modules
- **Root cause:** High coupling to Spring specific APIs in business code
- **Investigation:** Check imports — `org.springframework.*` in domain classes
- **Resolution:** Anti-corruption layer. Spring annotations only in `infrastructure` package. Domain code is Spring-agnostic.

### Issue 6: Production Incident — Global Variable Corruption
- **Symptom:** One request corrupts data for another request
- **Root cause:** Common coupling — shared mutable state (static cache, ThreadLocal misuse)
- **Investigation:** Identify shared state in stack traces
- **Resolution:** Make shared state immutable or request-scoped.

### Issue 7: Refactoring Takes Forever
- **Symptom:** Any refactoring requires touching 20+ files
- **Root cause:** Low cohesion + high coupling — responsibilities spread across classes
- **Investigation:** Measure CBO per class
- **Resolution:** Extract cohesive modules. One change per module.

---

## 13. Comparison Section

### Coupling Types Comparison

| Coupling Type | Level | Flexibility | Testability | Change Impact |
|---------------|-------|-------------|-------------|---------------|
| Content | ❌ Worst | None | Impossible | Changes everywhere |
| Common | ❌ Bad | None | Very hard | Global |
| External | ⚠️ Poor | Low | Hard | Across system |
| Control | ⚠️ Poor | Low | Moderate | Flag changes affect callers |
| Stamp | ⚡ Fair | Moderate | Moderate | Structure changes propagate |
| Data | ✅ Good | High | Easy | Minimal |
| Message | ✅ Best | Maximum | Easy | None (temporal) |

### Cohesion Types Comparison

| Cohesion Type | Level | Understandability | Maintainability | When to Use |
|---------------|-------|-------------------|-----------------|-------------|
| Coincidental | ❌ Worst | Very low | Very low | Never |
| Logical | ❌ Poor | Low | Low | Rarely (if grouping by analogy) |
| Temporal | ⚠️ Fair | Moderate | Moderate | Startup/shutdown sequences |
| Procedural | ⚡ Fair | Moderate | Moderate | Workflow steps (orchestration) |
| Communicational | ✅ Good | High | High | Domain objects (entity + behavior) |
| Sequential | ✅ Good | High | High | Pipelines (ETL, data processing) |
| Functional | ✅ Best | Very high | Very high | Single-responsibility modules |

### Coupling & Cohesion Across Architecture Styles

| Architecture | Coupling | Cohesion | Coordination Cost |
|-------------|----------|----------|-------------------|
| Monolith | High | Medium | Low (one team) |
| Modular monolith | Medium | High | Medium (per module) |
| Microservices | Low | Very high | High (cross-service) |
| Serverless | Very low | Very high | Very high (function coordination) |
| Event-driven | Lowest | High per handler | Medium (event tracing) |

---

## 14. Revision Notes

### Key Principles
- **High cohesion + low coupling = good design**
- **Content coupling** — worst; **Message coupling** — best
- **Coincidental cohesion** — worst; **Functional cohesion** — best
- **SRP** drives high cohesion; **DIP** drives low coupling
- **Law of Demeter** — "don't talk to strangers" (reduces coupling)

### Detection Heuristics
- God class → low cohesion, high coupling
- Feature envy → belongs in another class (reduces cohesion)
- Boolean parameters → control coupling (split the method)
- `instanceof` chains → missing polymorphism (low cohesion)
- Long parameter lists → possible stamp coupling

### Coupling/Cohesion "Smells"
```
❌ Utils/Helper class with 20 unrelated methods → coincidental cohesion
❌ if/switch on type code → missing polymorphism, low cohesion
❌ getX().getY().getZ() chain → inappropriate intimacy, high coupling
❌ Too many constructor parameters → high coupling to dependencies
❌ One method using 1 field, another using 10 → low cohesion
```

### Interview Checkpoints
1. Know the 7 levels of coupling (content → message)
2. Know the 7 levels of cohesion (coincidental → functional)
3. Understand trade-offs — over-decoupling is also bad
4. Be ready to refactor a tightly coupled code snippet
5. Know how DI, events, and interfaces reduce coupling
6. Understand Conway's Law and team boundaries

---

## 15. Cheat Sheet

```
═══ COUPLING & COHESION ═══════════════════════════════════════

GOAL: High Cohesion + Low Coupling

┌─ COUPLING (worst → best) ───────────────────────────────────┐
│ ❌ Content    — direct internal access                       │
│ ❌ Common     — shared global state                          │
│ ⚠️ External   — file system, DB, network                    │
│ ⚠️ Control    — flag parameters                             │
│ ⚡ Stamp      — passing entire object for one field         │
│ ✅ Data       — just the data you need                      │
│ ✅ Message    — events, no direct dependency                │
└─────────────────────────────────────────────────────────────┘

┌─ COHESION (worst → best) ───────────────────────────────────┐
│ ❌ Coincidental — random grouping (Utils)                    │
│ ❌ Logical      — "all X-related things"                    │
│ ⚠️ Temporal     — "things that happen at the same time"    │
│ ⚡ Procedural   — steps of a procedure                      │
│ ✅ Communicational — operate on same data                   │
│ ✅ Sequential   — output→input                              │
│ ✅ Functional   — one clear purpose                         │
└─────────────────────────────────────────────────────────────┘

┌─ DESIGN RULES ──────────────────────────────────────────────┐
│ • SRP  → high cohesion     (one reason to change)           │
│ • DIP  → low coupling      (depend on abstractions)         │
│ • ISP  → focused interfaces (no fat interfaces)             │
│ • LoD  → don't chain deep   (law of Demeter)                │
│ • YAGNI → don't over-decouple                               │
└─────────────────────────────────────────────────────────────┘

┌─ REFACTORING RECIPES ───────────────────────────────────────┐
│ God class           → Extract Class (by responsibility)      │
│ Feature envy        → Move Method (to the envied class)      │
│ Boolean flags       → Extract Method (one per behavior)      │
│ instanceof chain    → Replace with Polymorphism              │
│ Stamp coupling      → Pass only needed parameters            │
│ Common coupling     → Encapsulate Global State               │
│ Long parameter list → Introduce Parameter Object             │
└─────────────────────────────────────────────────────────────┘
```

---

## 16. Knowledge Validation

### 20 Multiple Choice Questions

**Q1: Which is the best form of coupling?**
- A) Content coupling
- B) Common coupling
- C) **Message coupling**
- D) Control coupling

**Q2: Which is the worst form of coupling?**
- A) **Content coupling**
- B) Data coupling
- C) Stamp coupling
- D) Message coupling

**Q3: Which is the best form of cohesion?**
- A) Temporal cohesion
- B) Procedural cohesion
- C) **Functional cohesion**
- D) Logical cohesion

**Q4: Which is the worst form of cohesion?**
- A) Coincidental cohesion
- B) Logical cohesion
- C) Temporal cohesion
- D) **Coincidental cohesion**

**Q5: A class that validates, persists, emails, and generates PDFs for orders has:**
- A) High cohesion
- B) **Low cohesion**
- C) Low coupling
- D) Good design

**Q6: Passing a boolean flag `boolean sendEmail` to a method is an example of:**
- A) Content coupling
- B) **Control coupling**
- C) Stamp coupling
- D) Data coupling

**Q7: `Utils.java` with `formatDate()`, `encryptPassword()`, `sendSMS()`, and `parseXML()` is:**
- A) Functional cohesion
- B) **Coincidental cohesion**
- C) Sequential cohesion
- D) Communicational cohesion

**Q8: Law of Demeter helps reduce:**
- A) Cohesion problems
- B) **Coupling problems**
- C) Performance problems
- D) Security problems

**Q9: Which pattern reduces coupling by making modules communicate through events?**
- A) **Event-driven architecture**
- B) Singleton pattern
- C) Template method
- D) Factory pattern

**Q10: Which SOLID principle directly improves cohesion?**
- A) Liskov Substitution
- B) **Single Responsibility**
- C) Open/Closed
- D) Dependency Inversion

**Q11: Feature envy indicates:**
- A) High coupling
- B) **Low cohesion**
- C) High cohesion
- D) Low coupling

**Q12: A microservice that accesses another service's database directly is:**
- A) **Content coupling**
- B) Data coupling
- C) Message coupling
- D) Stamp coupling

**Q13: A 5000-line class with 30 methods and 20 dependencies likely has:**
- A) **Low cohesion and high coupling**
- B) High cohesion and low coupling
- C) High cohesion and high coupling
- D) Low cohesion and low coupling

**Q14: Which testing practice is affected by high coupling?**
- A) **Unit testing (difficult to mock)**
- B) Performance testing
- C) Load testing
- D) Security testing

**Q15: What is the output of one module used as input to another?**
- A) Functional cohesion
- B) **Sequential cohesion**
- C) Communicational cohesion
- D) Temporal cohesion

**Q16: Services sharing a global configuration file is an example of:**
- A) Content coupling
- B) **Common coupling**
- C) External coupling
- D) Stamp coupling

**Q17: Which refactoring reduces stamp coupling?**
- A) Extract Interface
- B) **Pass only required parameters**
- C) Move Method
- D) Extract Class

**Q18: Conway's Law states that:**
- A) **Systems mirror communication structures**
- B) More developers make projects slower
- C) Adding people to late projects makes it later
- D) Code complexity increases over time

**Q19: The Dependency Inversion Principle reduces:**
- A) Cohesion problems
- B) **Coupling problems**
- C) Both coupling and cohesion
- D) Neither

**Q20: A class whose methods all operate on the same instance variables has:**
- A) **Communicational cohesion**
- B) Functional cohesion
- C) Sequential cohesion
- D) Temporal cohesion

### 10 Coding Questions

**Q1:** Identify coupling type: `user.getAddress().getCity().getZipCode()`

**A:** Inappropriate intimacy / Law of Demeter violation. It's tight coupling to the full navigation chain. Fix: `user.getShippingZipCode()` — expose computed value.

**Q2:** Refactor this to reduce coupling:
```java
public void process(Employee emp, boolean sendEmail) {
    double tax = calculateTax(emp.getSalary());
    emp.setTaxWithheld(tax);
    if (sendEmail) emailService.send(emp.getEmail(), "Tax: " + tax);
}
```

**A:** Remove boolean flag (control coupling). Split:
```java
public void process(Employee emp) { /* ... */ }
public void processAndNotify(Employee emp) {
    process(emp);
    emailService.send(emp.getEmail(), "Tax: " + calculateTax(emp.getSalary()));
}
```

**Q3:** Identify the cohesion type:
```java
public class StartupTasks {
    public void initializeDatabase() { /* ... */ }
    public void warmupCache() { /* ... */ }
    public void startMetricsServer() { /* ... */ }
    public void registerHealthChecks() { /* ... */ }
}
```

**A:** Temporal cohesion — methods run at the same time (startup) but have no other relationship.

**Q4:** Fix this feature envy:
```java
public class OrderService {
    public double calculateCustomerDiscount(Customer customer) {
        double total = 0;
        for (Order o : customer.getOrders()) total += o.getTotal();
        return total > 1000 ? 0.1 : 0;
    }
}
```

**A:** Move the method to `Customer`:
```java
public class Customer {
    public double calculateDiscount() {
        double total = orders.stream().mapToDouble(Order::getTotal).sum();
        return total > 1000 ? 0.1 : 0;
    }
}
// OrderService: customer.calculateDiscount()
```

**Q5:** Write an ArchUnit test that prevents controllers from depending on repositories.

**A:**
```java
@Test
public void controllers_should_not_depend_on_repositories() {
    classes().that().resideInAPackage("..controller..")
        .should().onlyDependOnClassesThat()
        .resideOutsideOfPackage("..repository..")
        .check(importedClasses);
}
```

**Q6:** Identify coupling and fix:
```java
@Service
public class UserService {
    public void register(RegistrationRequest request) {
        User user = new User();
        user.setName(request.getName());
        user.setEmail(request.getEmail());
        user.setPassword(hashPassword(request.getPassword())); // control coupling via flag hidden
        userRepository.save(user);
        emailService.sendWelcomeEmail(user); // Stamp coupling — passing whole user
        auditService.log("USER_REGISTERED", user.getId()); // Should only need ID and action
    }
}
```

**A:** Stamp coupling to `emailService` — pass only `email` and `name`. Data coupling to `auditService` — OK but audit data extractable.

**Q7:** Fix common coupling:
```java
public class Config {
    public static int MAX_RETRIES = 3;
    public static String DB_URL = "jdbc:mysql://localhost:3306/db";
}
// 50 classes read these; 10 classes write to them
```

**A:** Make immutable + use DI:
```java
@Component
@ConfigurationProperties(prefix = "app")
public class AppConfig {
    private int maxRetries;
    private String dbUrl;
    // getters only — no setters
}
```

**Q8:** Refactor to higher cohesion:
```java
public class ProductController {
    @GetMapping public List<Product> getAll() { /* ... */ }
    @PostMapping public Product create(@RequestBody Product p) { /* ... */ }
    @DeleteMapping public void delete(@PathVariable Long id) { /* ... */ }
    @GetMapping("/report") public Report generateReport() { /* ... */ }
    @PostMapping("/bulk-import") public void bulkImport(@RequestBody File f) { /* ... */ }
}
```

**A:** Split:
```java
@RestController
@RequestMapping("/products")
public class ProductCrudController {
    @GetMapping public List<Product> getAll() { /* ... */ }
    @PostMapping public Product create(@RequestBody Product p) { /* ... */ }
    @DeleteMapping("/{id}") public void delete(@PathVariable Long id) { /* ... */ }
}
@RestController
@RequestMapping("/products/reports")
public class ProductReportController { /* ... */ }
@RestController
@RequestMapping("/products/admin")
public class ProductAdminController { /* ... */ }
```

**Q9:** Identify coupling in this code:
```java
public class PaymentHandler {
    public void handlePayment(PaymentEvent event) {
        Order order = orderRepository.findById(event.getOrderId());
        order.setStatus("PAID");
        order.setPaymentTransactionId(event.getTransactionId());
        orderRepository.save(order);
    }
}
```

**A:** Temporal coupling — `handlePayment` assumes the event arrives in the right order. Also, modifying `Order` from a `PaymentHandler` raises cohesion concerns.

**Q10:** Design a test for coupling between service layers.

**A:**
```java
// Using ArchUnit
@Test
public void domain_should_not_depend_on_infrastructure() {
    noClasses().that().resideInAPackage("..domain..")
        .should().dependOnClassesThat()
        .resideInAPackage("..infrastructure..")
        .check(importedClasses);
}
```

### 10 Scenario Questions

**Scenario 1:** A junior developer asks you to review a PR where they added a new feature by modifying 15 existing classes. What's wrong and how do you fix it?

**Answer:** This indicates low cohesion — the feature logic is scattered across 15 classes instead of being encapsulated in one. Extract the feature into a new class. Each of the 15 classes should change only if they need different interfaces to the feature.

**Scenario 2:** Your team's build time has gone from 5 minutes to 45 minutes. What's the likely cause and how do you fix it?

**Answer:** Likely cause: high coupling means any change recompiles the entire project. Fix: modularize — split into Maven/Gradle modules with clear dependency direction. Measure compilation scope before and after.

**Scenario 3:** A new team member changed one line in `PaymentService` and accidentally broke `NotificationService`. Why?

**Answer:** High coupling — `PaymentService` probably throws an exception that `NotificationService` catches, or they share a mutable object. Fix: reduce shared state, use events, add contract tests.

**Scenario 4:** Management wants you to estimate "how long to add one new report." You can't give an estimate because the report generation logic is spread across 20 classes. What do you say?

**Answer:** "Our reporting logic has low cohesion — it's spread across 20 classes with high coupling. I recommend a 2-week refactoring to extract reporting into a cohesive module. After that, new reports take 2-3 days each."

**Scenario 5:** Two microservices share a Redis cache. Changing the cache key format breaks both. How do you decouple?

**Answer:** Each service should own its cache entries. Use namespaced keys (`serviceA:orders:*`, `serviceB:products:*`). Better: separate Redis instances. Use a cache abstraction layer.

**Scenario 6:** The QA team reports that fixing one bug introduces two more. What design issue is this and how do you fix it?

**Answer:** High coupling + low cohesion = fragile code. Fixing one thing breaks others because responsibilities are entangled. Solution: extract cohesive modules with clear interfaces. Add characterization tests before refactoring.

**Scenario 7:** You're architecting a system for a startup that needs to move fast. How do you balance coupling and cohesion with speed?

**Answer:** Start with a modular monolith. Use package boundaries and interfaces but keep deployment as one unit. As the team grows and needs independent shipping, extract modules into services. Don't over-decouple early.

**Scenario 8:** A senior engineer says "we don't need interfaces, they're just extra code." Your system has 20 microservices. What do you say?

**Answer:** "Interfaces are contracts. Without them, we have implicit coupling — services depend on concrete implementations. If a service changes its payment provider, every direct reference breaks. Interfaces localize that change."

**Scenario 9:** Your codebase has 80% code coverage but your tests take 3 hours and you still get production bugs. Why?

**Answer:** Likely: tests are integration tests that test everything together (due to high coupling). They're slow and test the wrong things. Fix: refactor for low coupling, write focused unit tests for logic, integration tests only for boundaries.

**Scenario 10:** A monolithic system must be split into microservices, but the business can't pause feature development for a rewrite. What's your approach?

**Answer:** Strangler Fig pattern. Identify one bounded context with high internal cohesion and relatively low coupling to the rest. Extract it behind an API. Migrate consumers one by one. Don't stop feature development — allocate 20% of each sprint to extraction.
