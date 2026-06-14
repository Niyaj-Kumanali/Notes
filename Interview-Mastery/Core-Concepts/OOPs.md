# Object-Oriented Programming (OOP)

---

## Overview

- **Definition**
  - OOP is a programming paradigm that organizes software around objects — containers of data (fields) and behavior (methods).

  **Why It Exists**
    - Procedural code becomes hard to maintain at scale because behavior and the data it operates on are separated.
    - A change to a data structure forces you to find and update every function that touches it.
    - OOP bundles data and behavior together, so changes are localized to the object that owns them.

  **Historical Context**
    - Before OOP went mainstream in the 1990s (C++, Java, C#), large systems were written in C or COBOL with modules that operated on shared structs.
    - A change as simple as adding a field to a `Customer` struct required auditing every function that accessed it — and because the data was globally visible, invariants were impossible to guarantee (any function could set `balance` to any value).
    - Simula (1967) introduced classes and objects, but it was Smalltalk and later C++ that proved the paradigm could scale to large systems.
    - The key insight was not "put code with data" — it was encapsulation as a correctness mechanism: by controlling access through methods, you could protect invariants the way a database protects its rows through transactions.

- **Key Concepts**
  - **Encapsulation** — private fields, public methods; protects invariants.
  - **Abstraction** — interfaces/abstract classes hide implementation details.
  - **Inheritance** — is-a relationships; derive new types from existing ones.
  - **Polymorphism** — same interface, different behavior depending on runtime type.
  - **Composition** — has-a relationships; prefer over inheritance for flexibility.

---

## Core Concepts

- **Encapsulation**
  - Hide internal state; all interaction through public methods. This protects invariants and allows internal refactoring without affecting callers.

```java
public class BankAccount {
    private double balance;

    public void deposit(double amount) {
        if (amount <= 0) throw new IllegalArgumentException("Amount must be positive");
        balance += amount;
    }

    public double getBalance() { return balance; }
}
```

  - The `balance` field being private is not just a naming convention — it means no external code can set `balance = -9999` directly.
  - The `deposit` method is the only entry point, and it enforces the rule that deposits must be positive.
  - If you later add overdraft protection, interest accrual, or audit logging, you change one method in one class. Without encapsulation, every caller that increments `balance` directly would need to be found and updated.

- **Abstraction**
  - Simplify by exposing only what is necessary. Interfaces define contracts; abstract classes provide partial implementation.

```java
interface PaymentGateway {
    PaymentResult charge(double amount);
}

class StripeGateway implements PaymentGateway {
    public PaymentResult charge(double amount) {
        return StripeAPI.charge(amount);
    }
}
```

  - The value of abstraction is that `OrderService` can depend on `PaymentGateway` without knowing whether it is Stripe, PayPal, or a test double. Switching payment providers becomes a configuration change, not a code change.

- **Inheritance**
  - Derive new classes from existing ones. A child inherits the parent's members and can override methods.

```java
abstract class Vehicle {
    abstract void start();
}

class Car extends Vehicle {
    void start() { System.out.println("Engine on"); }
}
```

  - **Liskov Substitution Principle (LSP)** — Subtypes must be substitutable for their base types. The classic violation is `Square extends Rectangle`: if `setWidth` also changes height to maintain the square invariant, client code that assumes width and height are independent will silently produce wrong results.
  - The test is simple — if a subtype needs to override a method in a way that weakens or invalidates an assumption the parent established, the inheritance relationship is wrong.

- **Polymorphism**
  - **Compile-time polymorphism** is method overloading — same name, different parameter types. The compiler resolves which method to call at compile time based on the declared argument types.
  - **Runtime polymorphism** is method overriding — the JVM dispatches to the actual type at runtime, not the declared type.

```java
void print(String s) { System.out.println(s); }
void print(int i)    { System.out.println(i); }

// Runtime polymorphism
Vehicle v = new Car();
v.start();  // dispatches to Car.start() at runtime
```

  - The runtime case is why you can write `List<Vehicle> fleet` and call `start()` on each element without a single `instanceof` check — the JVM handles dispatch to the right implementation automatically.

- **Relationship Types**
  - **Association** — General "uses-a" relationship. `Teacher` interacts with `Student`, but neither owns the other (e.g., a teacher can exist without any students).
  - **Aggregation** — Weak "has-a." `Department` has `Professor` instances, but professors exist independently. Deleting the department does not delete the professors.
  - **Composition** — Strong "has-a." `House` has `Room` instances; the rooms have no meaningful existence outside the house. Deleting the house deletes the rooms. In code, this usually means the child object is created inside the parent's constructor and not shared.
  - The distinction between aggregation and composition matters for lifecycle management and for communicating intent to other developers. If you see composition, you know the child should not be shared or retained elsewhere.

- **SOLID Principles**

| Principle | What it means | Violation example |
|-----------|--------------|-------------------|
| **S**ingle Responsibility | One reason to change | `OrderService` handles DB, email, logging |
| **O**pen/Closed | Extend without modifying | Adding report export requires altering `ReportGenerator` |
| **L**iskov Substitution | Subtypes replace base types | `Square` inherits `Rectangle` with broken `setWidth` |
| **I**nterface Segregation | Don't force unused methods | `Worker` interface with `eat()` on `Robot` |
| **D**ependency Inversion | Depend on abstractions | `OrderService` instantiates `MySQLRepository` directly |

  **SRP nuance**
    - "One responsibility" does not mean one method per class — it means one reason to change.
    - A `UserService` with `createUser`, `updateUser`, and `deleteUser` is fine because all three change together when user requirements change.
    - The violation is when DB logic, email sending, and PDF generation all live in the same class — a change to the email template requires touching the same file as a change to the database schema.

---

## Common Mistakes

- **God classes**
  - A single class handling validation, DB access, email, logging, and PDF generation. This grows organically: one method is added for "just this one thing," then another, until the class is 3000 lines.
  - During development all the logic is in one place and easy to find. The practical problem is not just that it violates SRP — it is that every test must deal with all of those dependencies, and a change to any one path risks breaking something completely unrelated.
  - In production, the symptom is a deployment that changes an email template and causes an unrelated validation bug — and the team spends two days debugging because they cannot test the validation path independently of the email dependency.

- **Deep inheritance**
  - More than 3 levels creates fragile hierarchies. A change to a grandparent method can silently break grandchildren.
  - The deeper the hierarchy, the harder it is to reason about what a method call actually does without tracing through multiple levels of overrides.
  - This is a scale inflection point: at 2 levels, you can keep the full call graph in your head. At 4 levels, each method invocation has 3+ override candidates depending on the runtime type, and reasoning about a single path requires holding a 4-deep mental call stack.
  - The fragility grows exponentially with depth because each level of override can introduce or break behavior that the level above assumed.

- **`@Data` on JPA entities**
  - Lombok's `@Data` generates `hashCode` and `equals` based on all fields, including the auto-generated `id`.
  - Before an entity is persisted, `id` is null. After persistence, `id` has a value. If you store the entity in a `HashSet` before saving it, you will never be able to find it again after saving because its hash code changed.

- **Violating LSP**
  - `Square extends Rectangle` causes subtle, hard-to-trace bugs because the violation only manifests in client code that uses the base type — not in the subtype itself.

- **Feature envy**
  - A method in class A that calls five getters on class B to do its calculation. This starts when a method needs data from another class to compute something, and the path of least resistance is to call getters rather than move the logic.
  - The method still works and returns the right result. Over time, class B's interface becomes a procedural data bag with getters for everything, and class A accumulates business logic that should belong to B.
  - When the data structure changes, every envious method across the codebase must be updated — a single field rename becomes a week-long refactoring project.

- **Mocking concrete classes**
  - Tests that mock `MySQLRepository` directly are coupled to the implementation. When the class is refactored or replaced, the test breaks even if the behavior is correct. Mock interfaces instead.

---

## Key Design Considerations

- **Dependency Injection**
  - Pass dependencies via constructor; don't instantiate them inside the class.
  - This decouples the class from its dependencies and makes them substitutable in tests.
  - The trade-off is slightly more boilerplate, but it is the single most impactful practice for making classes testable in isolation.

- **Factory Pattern**
  - Encapsulates object creation. Use when construction logic involves conditionals, configuration, or multiple steps that callers should not need to know about.

- **Strategy Pattern**
  - Encapsulate interchangeable algorithms behind an interface. Replace `if-else` chains where the branching logic is about "which behavior to apply" rather than "what data to use."

- **Observer Pattern**
  - One-to-many notification. A subject publishes events; subscribers react. The subject does not know who is listening, which keeps it decoupled from the notification channels.

- **Null Object Pattern**
  - Return a no-op implementation instead of null. Callers do not need to null-check; they just call the method and nothing happens. Eliminates NPEs at the cost of one extra class.

- **Domain-Driven Design**
  - Use a shared ubiquitous language between developers and domain experts so code reflects the business model directly.
  - Aggregates enforce consistency boundaries.
  - Value Objects (like `Money` or `Address`) have no identity — two `Money(100, USD)` instances are equal; two `Order` instances with the same contents are not.

---

## Real-World Scenarios

### Scenario 1: Plugin System for a Photo Editor

- **Context**
  - Users can extend GIMP/Photoshop with plugins (filters, export formats). Each plugin must integrate without modifying the core application.

- **Design**
  - Define a `Plugin` interface with `apply(image): Image`. Plugins are loaded via the Service Provider Interface (SPI) or reflection. The core application discovers and invokes plugins without knowing their concrete types.

- **Trade-off**
  - Reflection adds startup cost and reduces type safety. If performance is a constraint, compile-time code generation via annotation processors is an alternative — but it requires a build step and more infrastructure.

> **Interview follow-up:** How would you ensure a single misbehaving plugin (infinite loop, memory leak) does not crash the entire editor?

### Scenario 2: Multi-Channel Notification System

- **Context**
  - An e-commerce app needs to send order confirmations via email, SMS, push, and in-app toast. New channels must be addable without changing existing code.

- **Design**
  - `NotificationChannel` interface with `send(message)`. Each channel implements it. A `NotificationService` accepts a list of `NotificationChannel` instances injected via DI. Adding WebSocket is one new class and a registration call — no changes to `NotificationService` or any existing channel.

- **Trade-off**
  - Notifications become eventually consistent — the ticket/order is updated before notifications fire. If a notification fails, you need retry logic; the simpler direct-call approach fails synchronously and more visibly.

> **Interview follow-up:** How would you prioritize channels so that SMS notifications (expensive, per-message cost) do not flood your budget during a spam attack, while email remains unaffected?

### Scenario 3: Legacy God Class Refactoring

- **Context**
  - `OrderService` is 3000 lines with validation, persistence, email, PDF generation, and logging all mixed together. Adding a feature breaks something unrelated 30% of the time.

- **Fix**
  - Apply SRP — extract `OrderValidator`, `OrderRepository`, `EmailService`, `PdfGenerator`, `OrderLogger`. The original `OrderService` becomes a facade that delegates. Each extracted class has one reason to change and can be tested with focused, fast unit tests.

- **Trade-off**
  - More indirection — you now have to trace through a delegation chain to understand the full flow. The benefit is that changing the email template requires touching exactly one file, with no risk to the database layer.

## Use Cases

- Reach for OOP when you need to bundle data with behavior in a system large enough that uncoordinated changes to shared data structures become a risk. The paradigm shines when you have multiple developers working on the same codebase and need to protect invariants.

- **Modeling domain entities** — encapsulate data and behavior together
  - Use classes like `Customer`, `Order`, `Product` to keep related fields and methods in one place. Encapsulation prevents callers from corrupting internal state (e.g., a negative `balance`).
  - **Avoid when:** the object is a pure data carrier (DTO, configuration POJO) with no behavior — a record or a simple struct is clearer.

- **Polymorphic dispatch** — treat different types through a common interface
  - Define an interface or abstract class when multiple implementations exist (e.g., `PaymentProcessor` with `CreditCardProcessor`, `PayPalProcessor`). Callers work against the abstraction, not the concrete type.
  - **Avoid when:** the "is-a" relationship is forced — prefer composition over inheritance to avoid fragile base class syndrome.

- **Controlled mutation with invariants** — guard object state transitions
  - Use private fields with business methods (not bare setters) to ensure valid state transitions. A `Ticket.setStatus(Status)` method can reject invalid transitions that a setter would silently accept.
  - **Avoid when:** the object's state is simple and trusted (e.g., an in-memory cache entry) — direct field access in a tight loop avoids method dispatch overhead.

- **Code reuse via inheritance** — share common behavior
  - Extract shared fields and lifecycle methods into a base class (e.g., `BaseEntity` with `id`, `createdAt`, `updatedAt`). Subclasses inherit the boilerplate.
  - **Avoid when:** the shared behavior is unrelated to the class hierarchy — composition (delegate to a helper object) keeps the coupling looser.

- **Namespace and organization** — group related constants and utilities
  - A class can serve as a namespace for related static methods and constants when top-level packages would be overkill (e.g., `MathUtils`, `ValidationRules`).
  - **Avoid when:** the class is just a bag of unrelated static methods — split into focused utility classes.

---

## Scenario-Based Questions

**Q: You are building a customer support ticket system. A new notification channel requires changing the core `Ticket` class. How do you redesign?**

- **Solution**
  - Apply the Observer pattern with dependency injection. Define a `TicketObserver` interface. `TicketService` emits events when tickets are created or updated. Each notification channel implements `TicketObserver`. Adding WebSocket notifications is one new class with zero changes to existing code.
  - Trade-off: notifications become eventually consistent — the ticket is updated before observers fire, so a failed notification needs its own retry mechanism.
  - **Non-obvious constraint:** The Observer pattern introduces a hidden dependency — `TicketService` does not know who is listening, but it must guarantee that observers complete quickly or risk blocking the ticket update path.

> **Interview follow-up:** An SMS provider is down. How do you prevent the slow SMS observer from delaying all other notification channels and the ticket update itself?

---

**Q: Every new carrier in your shipping cost calculator requires adding `if-else` in `ShippingService`. How do you refactor?**

- **Solution**
  - Apply the Strategy pattern. Define a `ShippingStrategy` interface with `calculate(Order): Money`. Each carrier and tier combination is a strategy implementation. A registry maps `(carrier, tier)` → strategy.
  - Trade-off: more classes, but each strategy is independently testable and can be handed to the business team to verify in isolation.
  - **Non-obvious constraint:** The registry needs a fallback for unknown carrier/tier combinations — returning a null strategy would require null checks everywhere. A `DefaultShippingStrategy` (Null Object pattern) returns the highest rate and logs a warning.

> **Interview follow-up:** A new carrier has 5 tier levels with different pricing formulas. How do you design the strategy hierarchy so that adding a tier does not require creating a new class?

---

**Q: You're building a plugin marketplace. Plugins are third-party JARs. Plugins must interact with the canvas, but must not access the filesystem. How?**

- **Solution**
  - Apply the Facade pattern combined with the Proxy pattern. Expose a `CanvasAPI` interface that covers only legitimate operations (draw shapes, read selection). Plugins run in a separate `ClassLoader` with a `SecurityManager` that restricts filesystem and network access.
  - Trade-off: sandboxing adds complexity and limits plugin performance — plugins cannot get direct GPU access through the restricted API.
  - **Non-obvious constraint:** `SecurityManager` is deprecated in Java 17+ and removed in Java 21. For modern JVMs, the sandbox must be implemented at the OS level (subprocess with restricted permissions, container, seccomp) rather than at the JVM level.

> **Interview follow-up:** A plugin needs to load a font file from a specific directory to render text. How do you grant limited filesystem access without opening the entire filesystem to the plugin?

---

**Q: A 5000-line `ReportGenerator` handles PDF, Excel, CSV, and HTML. Adding JSON risks breaking existing formats. What do you do?**

- **Solution**
  - Apply Template Method combined with Strategy. Create a `ReportFormatter` interface where each format contains only format-specific logic. `ReportGenerator` orchestrates: fetch data → transform → pass to formatter. Adding JSON means one new class, no modifications to any existing formatter.
  - Trade-off: some shared transformation logic may be duplicated across formatters; extract that into a shared utility if it diverges.
  - **Non-obvious constraint:** The Template Method assumes the report generation flow (fetch → transform → format) is the same for all formats. If a future format needs a completely different flow (e.g., streaming output for huge datasets), forcing it into this template makes it harder, not easier.

> **Interview follow-up:** How do you handle reports that are too large to fit in memory — what pattern supports streaming the output row by row rather than building the entire report in memory?

---

**Q: Your app has 20+ microservices, each with its own `User` model. Adding a field requires updating all 20 models. How do you decouple?**

- **Solution**
  - Each service should own only the slice of user data it needs. When a user profile changes, a `UserUpdated` event fires; each service updates only its relevant subset.
  - Trade-off: eventual consistency — for a window after a profile update, some services will show stale data. This is usually acceptable for profile data but would not be acceptable for security-critical fields like permissions or payment status.
  - **Non-obvious constraint:** The `UserUpdated` event schema becomes a shared contract between 20 services. Changing the event schema (adding a field, renaming) requires coordinated updates across all consumers even though each only uses a subset.

> **Interview follow-up:** How would you roll out a breaking change to the `UserUpdated` event schema without requiring all 20 services to deploy simultaneously?

---

**Q: Orders arrive from web, mobile, and POS with different validation rules. How do you avoid `if (source == WEB)` scattered throughout?**

- **Solution**
  - Use a Factory pattern with a validation strategy per source. `OrderProcessorFactory.create(source)` returns an `OrderProcessor` configured with the correct `OrderValidator`. Each validator implements the same interface with source-specific rules.
  - Validation logic is centralized per source — adding a new source channel is a new class and a factory entry, not a grep-and-edit across the codebase.
  - **Non-obvious constraint:** The Factory must choose the correct validator before the request is fully parsed — you need the `source` field early enough in the pipeline to route correctly, which constrains how you structure your API endpoints.

> **Interview follow-up:** If a single order can contain items from both web and POS inventory, how do you handle mixed-source validation?

---

**Q: `ShoppingCart` has methods for add, remove, applyDiscount, calculateTax, checkout, saveForLater, and shareWithFriends. What's the design smell and fix?**

- **Solution**
  - Low cohesion — seven unrelated responsibilities that each have their own reason to change. Split into `CartManager` (add/remove), `PricingService` (discount/tax), `CheckoutService` (checkout), `CartPersistence` (save), `CartSharing` (share).
  - Trade-off: more objects to wire together via DI. Benefit: changing the discount algorithm requires touching exactly one class with no risk to checkout or sharing logic.

---

**Q: `OrderService` uses `new PayPalService()` directly. Management wants to add Stripe. How do you migrate without breaking production?**

- **Solution**
  - Apply the Strangler Fig pattern. Introduce a `PaymentService` interface first. Create `PaypalAdapter` implementing it, delegating all calls to the existing `PayPalService`. All new code references only the interface. Add `StripeAdapter` when ready. Eventually remove `PayPalService` entirely.
  - Trade-off: double maintenance during the transition period — both the old class and the adapter exist simultaneously. The benefit is zero production downtime and a rollback path at every step.

---

**Q: `Character`, `Monster`, `NPC`, and `Item` all need `serialize()`, `deserialize()`, `display()`, and `clone()`. Copy-pasting leads to inconsistency. How do you DRY this?**

- **Solution**
  - Use the Prototype pattern for cloning (`clone()` in a base interface). For serialization, use the Visitor pattern — a `SerializerVisitor` visits each object type and handles its serialization logic without the object needing to know about the format. For display, use a `Renderer` strategy.
  - Trade-off: Visitor requires updating all visitors when a new type is added; if new types are added frequently, consider reflection-based serialization to avoid the maintenance overhead.

---

**Q: Your library is used by 50 internal teams. Every breaking change forces all 50 to update. How do you design the public API?**

- **Solution**
  - Apply the Facade pattern — expose a single `PublicAPI` class that delegates to internal implementations. Mark all internal classes as package-private or enforce module boundaries with JPMS. Use semantic versioning strictly: breaking changes only in major versions.
  - When a breaking change is necessary, introduce versioned facades (`v1`, `v2`) using the Adapter pattern and deprecate the old facade with a migration window.
  - Trade-off: the facade becomes a bottleneck for new capabilities — plan module boundaries carefully so the public surface stays small.
  - **Non-obvious constraint:** A facade that delegates to everything creates a monolithic dependency — teams that only need one feature must still depend on the entire library. Consider splitting into multiple facades by domain.

> **Interview follow-up:** One team reports that your library's `v2` facade is missing a method they relied on in `v1`. How do you handle the compatibility gap without rolling back the major version?

---

## Interview Questions

- **What are the four pillars of OOP?**
  - Encapsulation (hide data), Abstraction (hide complexity), Inheritance (is-a relationships), Polymorphism (same interface, different behavior).
  - Most interviewers consider this answer incomplete — a strong candidate also explains why encapsulation and polymorphism together are more powerful than either alone: encapsulation guarantees invariants, and polymorphism lets you vary behavior without bypassing those guarantees.

- **What is the difference between composition and inheritance?**
  - Composition (has-a) is more flexible — behaviors can be swapped at runtime and there is no tight coupling between classes.
  - Inheritance (is-a) creates a parent-child dependency where changing the parent can break all children.
  - Prefer composition except for true is-a relationships where the child will never need to deviate from the parent's contract.

- **What is the Liskov Substitution Principle?**
  - Subtypes must be substitutable for their base types without altering correctness.
  - `Square extends Rectangle` violates LSP because `setWidth` also changes height, breaking client code that assumes those dimensions are independent.

- **What is the difference between method overloading and method overriding?**
  - Overloading (compile-time polymorphism) — same method name, different parameter signatures in the same class. The compiler resolves the call at compile time.
  - Overriding (runtime polymorphism) — a child class redefines a parent method. The JVM uses dynamic dispatch to resolve the call at runtime based on the actual object type.

- **What is the purpose of an interface vs an abstract class?**
  - An interface defines a contract — what to do — with no implementation.
  - An abstract class provides partial implementation — both what and how.
  - Use interfaces for capabilities that unrelated classes may share (`Serializable`, `Comparable`).
  - Use abstract classes when there is genuinely shared base logic that subclasses should inherit rather than re-implement.

- **What is the Dependency Inversion Principle?**
  - High-level modules should not depend on low-level modules; both should depend on abstractions.
  - `OrderService` should depend on a `PaymentGateway` interface, not directly on `StripeGateway`. This means the payment provider can be changed or mocked without touching `OrderService`.

- **What is a design pattern? Give three examples.**
  - A reusable solution to a commonly recurring design problem.
  - Factory (creational — encapsulates object creation), Adapter (structural — makes incompatible interfaces work together), Observer (behavioral — one-to-many event notification).

- **What is the difference between aggregation and composition?**
  - Both are "has-a." Composition implies ownership and shared lifecycle — destroying the parent destroys the child (House → Rooms). Aggregation is weaker — the child can exist independently of the parent (Department → Professors).

- **What is the Open/Closed Principle?**
  - Classes should be open for extension but closed for modification.
  - Add new behavior by creating new implementations or subclasses, not by editing existing code. This protects existing, tested behavior from being inadvertently broken.

- **What is a god class and why is it a problem?**
  - A class with too many responsibilities — a 3000-line `OrderService` doing validation, DB access, email, and logging.
  - Every change touches the same file, making bugs from unrelated changes common. It is impossible to test any one responsibility in isolation because all the others are entangled with it.

- **What is the difference between shallow copy and deep copy?**
  - Shallow copy creates a new object but does not clone its reference-type fields — both the original and copy share the same nested objects.
  - Deep copy clones the entire object graph, creating independent copies of all referenced objects. In Java, `clone()` performs shallow copy by default; deep copy requires manual implementation or serialization-deserialization.

- **Can you override a private method in Java?**
  - No. Private methods are not inherited, so they cannot be overridden. If a child class defines a method with the same signature, it is a new method, not an override — there is no polymorphic dispatch.
  - This is a common trick question in interviews: the two methods are unrelated, so calling the method on a child reference (declared as the child type) executes the child's version, but calling it on a parent reference (declared as the parent type) executes the parent's version only.

- **What is the difference between `final`, `finally`, and `finalize`?**
  - `final` is a keyword that prevents inheritance (class), overriding (method), or reassignment (variable).
  - `finally` is a block in exception handling that always executes regardless of whether an exception is thrown.
  - `finalize()` is a deprecated method called by the garbage collector before reclaiming an object — never rely on it for resource cleanup; use `AutoCloseable` or `Cleaner` instead.

- **What is a covariant return type?**
  - An overriding method may return a subtype of the return type declared in the parent method. Java 5 introduced this for generics; Java 7+ extended it to all overrides.
  - Example: `Parent clone()` can be overridden as `Child clone()` in the subclass without needing a cast at the call site.

- **What is the difference between static and dynamic binding?**
  - Static binding (early binding) resolves at compile time for `private`, `static`, and `final` methods based on the declared reference type.
  - Dynamic binding (late binding) resolves at runtime using the actual object type for overridden instance methods. The JVM uses vtable dispatch to find the correct method implementation.

- **What is the diamond problem in multiple inheritance?**
  - A class inherits from two classes that both inherit from the same base, causing ambiguity over which inherited method to invoke.
  - Java avoids this by disallowing multiple inheritance of classes. Interfaces with `default` methods solve it via explicit disambiguation: the implementing class must override the conflicting method or use `SuperInterface.super.method()` to pick one.

- **What is a marker interface? Does it violate OOP principles?**
  - A marker interface has no methods — `Serializable`, `Cloneable`, `RandomAccess`. It tags a class with metadata discoverable via `instanceof`.
  - Some argue it violates OOP because it uses a type declaration where an annotation would suffice. In modern Java, `@Serializable` or custom annotations are preferred because they can carry configuration and are processed at compile time.

- **What is the difference between coupling and cohesion?**
  - Coupling measures how dependent one class is on another — low coupling means changes in one class rarely require changes in another.
  - Cohesion measures how focused a class's responsibilities are — high cohesion means all methods work toward a single purpose.
  - Good design targets high cohesion and low coupling: a `BankAccount` class should only do account-related things (high cohesion) and should not depend on `EmailService` directly (low coupling).

- **When would you choose an abstract class over an interface in Java 8+?**
  - Choose an abstract class when the base type has shared state (fields, constructors) or when some methods have common implementation that subclasses must inherit rather than override.
  - Choose an interface when you define a capability that unrelated classes can implement, or when multiple inheritance of type is needed.
  - Java 8+ interfaces can have `default` and `static` methods, which has made abstract classes less necessary — but they still cannot hold mutable state or constructors.

- **What is a virtual method?**
  - A method that can be overridden and participates in runtime polymorphic dispatch. In Java, all non-`static`, non-`private`, non-`final` instance methods are virtual by default — no `virtual` keyword is needed.
  - The JVM maintains a vtable (virtual method table) per class. At runtime, it reads the correct method pointer from the vtable based on the actual object type, enabling polymorphic behavior with minimal overhead — typically one indirection per call.

---

## Developer Recommendations

- **Prefer composition over inheritance**
  - Inheritance creates tight coupling: changing the parent class can break all children in non-obvious ways.
  - Composition lets you swap behaviors at runtime via the strategy pattern.
  - Reserve inheritance for true is-a relationships where the child will never need to violate the parent's contract.
  - A real case: a team built a `BaseEntity` with `save()`, `delete()`, and `validate()` that every domain model extended. When the audit-logging requirement changed from "log on save" to "log on every field change," modifying `BaseEntity.save()` broke the `Report` subclass that never needed auditing and had its own batch-save logic. The hierarchy was too coupled to decouple without rewriting all 30 subclasses.

- **Apply SOLID principles rigorously, but don't dogmatize**
  - SRP means one reason to change, not one method per class. Too many tiny classes create indirection hell where tracing a single operation requires jumping through ten files.
  - A `UserService` with `createUser`, `updateUser`, and `deleteUser` is fine — they all change together when user requirements change.

- **Use dependency injection for testability**
  - `new Database()` inside a constructor makes unit testing impossible without a real database.
  - Constructor injection makes dependencies explicit and substitutable.
  - Trade-off: more boilerplate (constructor params, DI container config), but the ability to test every class in complete isolation is worth it.
  - **Production story:** A service had a `new EmailClient()` inside its constructor. When the email server went down during a deployment, every health check instantiated the class, which tried to connect to the email server, which timed out after 30 seconds — the deployment took 20 minutes because the readiness probe failed on every pod until the email server recovered.

- **Prefer immutable objects**
  - Mutable shared state is the primary source of bugs in OOP: race conditions, unexpected mutation across call sites, objects in partially-updated states.
  - Make fields `final`, use builders for complex construction, return defensive copies from getters.
  - Trade-off: immutable objects create more garbage (a new object per mutation), but they eliminate entire categories of bugs.

- **Use the strategy pattern to eliminate `if-else` chains**
  - `if (type.equals("PDF")) ... else if (type.equals("CSV"))` is fragile: adding a new type requires modifying existing code.
  - Replace with a `Formatter` interface and a registry of `type → formatter`. Adding a new format is a new class and a registry entry, with no risk to existing formats.

- **Prefer small interfaces over large ones (Interface Segregation)**
  - A `Worker` interface with `work()`, `eat()`, and `sleep()` forces `Robot` to implement `eat()` with either an empty body or a thrown exception — both are lies.
  - Split into `Workable`, `Eatable`, `Sleepable`. No class is forced to pretend to support behavior it does not have.

- **Use the factory pattern to centralize complex construction**
  - When object creation involves conditionals, configuration, or dependency resolution, a factory encapsulates that complexity in one place. Callers do not need to know how objects are built, and construction logic is not duplicated across call sites.

- **Keep inheritance hierarchies shallow (≤ 3 levels)**
  - Deep hierarchies are fragile and hard to reason about. To understand what a method call does, you may need to trace through several levels of overrides.
  - Prefer implementing interfaces over extending classes; use `default` methods or shared helper classes when you need to share implementation across implementors.
