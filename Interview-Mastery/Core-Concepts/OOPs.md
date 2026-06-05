# Object-Oriented Programming (OOP)

---

## Overview

- **Definition:** OOP is a programming paradigm that organizes software around objects — containers of data (fields) and behavior (methods).
- **Why It Exists:** Procedural code becomes hard to maintain at scale. OOP provides encapsulation (bundle data + operations), modularity (interchangeable components), reusability (inheritance & composition), and maintainability (localized changes).
- **Key Concepts:** **Encapsulation** (private fields, public methods), **Abstraction** (interfaces/abstract classes hide details), **Inheritance** (is-a relationships), **Polymorphism** (same interface, different behavior), **Composition** (has-a relationships — prefer over inheritance).

---

## Core Concepts

### Encapsulation

Hide internal state; all interaction through public methods. This protects invariants and allows internal refactoring without affecting callers.

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

### Abstraction

Simplify by exposing only what's necessary. Interfaces define contracts; abstract classes provide partial implementation.

```java
interface PaymentGateway {
    PaymentResult charge(amount);
}
class StripeGateway implements PaymentGateway {
    public PaymentResult charge(amount) {
        return StripeAPI.charge(amount);
    }
}
```

### Inheritance

Derive new classes from existing ones. Child inherits parent's members and can override methods.

```java
abstract class Vehicle {
    abstract void start();
}
class Car extends Vehicle {
    void start() { System.out.println("Engine on"); }
}
```

**LSP:** Subtypes must be substitutable for base types. `Square extends Rectangle` violates LSP if `setWidth` also changes height — client code breaks.

### Polymorphism

**Compile-time:** Method overloading (same name, different params). **Runtime:** Method overriding (JVM dispatches by actual type).

```java
void print(String s) { System.out.println(s); }
void print(int i)    { System.out.println(i); }

// Runtime polymorphism
Vehicle v = new Car();
v.start();  // "Engine on"
```

### Relationship Types

- **Association:** General "uses-a" (Teacher ↔ Student)
- **Aggregation:** Weak "has-a" (Department has Professors; Professor lives on without Department)
- **Composition:** Strong "has-a" (House has Rooms; deleting House deletes Rooms)

### SOLID Principles

| Principle | What it means | Violation example |
|-----------|--------------|-------------------|
| **S**ingle Responsibility | One reason to change | `OrderService` handles DB, email, logging |
| **O**pen/Closed | Extend without modifying | Adding report export requires altering `ReportGenerator` |
| **L**iskov Substitution | Subtypes replace base types | `Square` inherits `Rectangle` with broken `setWidth` |
| **I**nterface Segregation | Don't force unused methods | `Worker` interface with `eat()` on `Robot` |
| **D**ependency Inversion | Depend on abstractions | `OrderService` instantiates `MySQLRepository` |

---

## Common Mistakes

- **God classes** — A single class doing everything (validation, DB, email, logging). Low cohesion, hard to test.
- **Deep inheritance** — More than 3 levels creates fragile hierarchies. Change a grandparent, break all descendants.
- **`@Data` on JPA entities** — `hashCode`/`equals` based on auto-generated `id` changes on persist.
- **Violating LSP** — `Square extends Rectangle` causes subtle bugs.
- **Feature envy** — Method in class A constantly calls getters on class B. The method belongs in B.
- **Mocking concrete classes** — Tests coupled to implementation details.

---

## Key Design Considerations

- **Dependency Injection:** Pass dependencies via constructor, don't create inside class. Enables loose coupling and easy testing.
- **Factory Pattern:** Encapsulates object creation. Use when construction logic is complex.
- **Strategy Pattern:** Encapsulate interchangeable algorithms. Replace `if-else` chains.
- **Observer Pattern:** One-to-many notification. Subject publishes events; subscribers react.
- **Null Object Pattern:** Return a no-op implementation instead of null to prevent NPEs.
- **Domain-Driven Design:** Ubiquitous language between devs and domain experts. Aggregates enforce consistency. Value Objects have no identity.

---

## Real-World Scenarios

### Scenario 1: Plugin System for a Photo Editor
Users can extend GIMP/Photoshop with plugins (filters, export formats). Each plugin must integrate without modifying the core application. **Design:** Define a `Plugin` interface with `apply(image): Image`. Plugins are loaded via service loader (SPI) or reflection. The core application discovers and invokes plugins without knowing their concrete types. **Trade-off:** Reflection adds startup cost and reduces type safety. Alternative: compile-time code generation with annotation processors.

### Scenario 2: Multi-Channel Notification System
An e-commerce app needs to send order confirmations via email, SMS, push notification, and in-app toast. New channels must be addable without changing existing code. **Design:** `NotificationChannel` interface with `send(message)`. Each channel implements it. A `NotificationService` accepts a list of `NotificationChannel` instances (strategy pattern injected via DI). Adding WebSocket = one new class. **Trade-off:** Strategy pattern increases class count but eliminates switch/if-else chains.

### Scenario 3: Legacy God Class Refactoring
`OrderService` is 3000 lines with validation, persistence, email, PDF generation, and logging all mixed together. Adding a feature breaks something unrelated 30% of the time. **Fix:** Apply SRP — extract `OrderValidator`, `OrderRepository`, `EmailService`, `PdfGenerator`, `OrderLogger`. Each new class has a single responsibility. The original `OrderService` becomes a facade that delegates. **Trade-off:** More indirection but each class is independently testable and changeable.

---

## Scenario-Based Questions

1. **Q: You are building a customer support ticket system. Agents can be from different teams, and notification preferences (email, SMS, Slack, in-app) vary. A new notification channel requires changing the core `Ticket` class. How do you redesign?**
   A: Apply the Observer pattern with dependency injection. Define a `TicketObserver` interface. `TicketService` emits events. Each notification channel implements `TicketObserver`. Adding WebSocket notifications = one new class, zero changes to existing code. Trade-off: notifications become eventual consistent; the ticket is updated before notifications fire.

2. **Q: Your e-commerce platform needs to calculate shipping costs based on weight, destination, carrier, and membership tier. Every new carrier requires adding `if-else` in `ShippingService`. How do you refactor?**
   A: Apply the Strategy pattern. Define a `ShippingStrategy` interface with `calculate(Order): Money`. Each carrier + tier combo is a strategy. A strategy registry maps (carrier, tier) → strategy. Trade-off: more classes but each strategy is independently testable and verifiable by business.

3. **Q: You're building a plugin marketplace for a design tool. Plugins are JARs uploaded by third-party developers. Plugins must interact with the canvas, but you must prevent malicious plugins from accessing the filesystem. How?**
   A: Apply the Facade pattern combined with the Proxy pattern. Expose a `CanvasAPI` interface that provides limited capabilities (draw shapes, read selection). Plugins run in a separate classloader with a security manager that restricts filesystem and network access. Trade-off: sandboxing adds complexity and limits plugin performance (no direct GPU access).

4. **Q: A 5000-line `ReportGenerator` handles PDF, Excel, CSV, and HTML output, each in separate methods but sharing a lot of state. Adding a new format (JSON) risks breaking existing formats. What do you do?**
   A: Apply Template Method + Strategy. Create a `ReportFormatter` interface. Each format implements it with only format-specific logic. The `ReportGenerator` orchestrates: fetch data → transform → pass to formatter. Trade-off: some shared logic duplication. Benefit: adding JSON creates no risk to PDF.

5. **Q: Your app has 20+ microservices, each with its own `User` model. Adding a field to the user profile requires updating all 20 models. How do you decouple?**
   A: Each service should own its user data subset. Use a shared `User` context bounded context boundary, but distributed via events. When user profile changes, a `UserUpdated` event fires; services update only their relevant subset. Trade-off: eventual consistency — after a profile update, some services may temporarily show stale data.

6. **Q: You need to process orders from web, mobile, and POS systems. Each has slightly different validation rules (web requires email, mobile requires phone, POS requires cashier ID). How do you avoid `if (source == WEB)` all over?**
   A: Use a Factory pattern with a validation strategy per source. `OrderProcessorFactory.create(source)` returns an `OrderProcessor` with the correct `OrderValidator`. Each validator implements the same interface but with different rules. Trade-off: more classes but validation logic is centralized per source.

7. **Q: A `ShoppingCart` class has methods for add, remove, applyDiscount, calculateTax, checkout, saveForLater, and shareWithFriends. What's the design smell and fix?**
   A: Low cohesion — seven unrelated responsibilities. Split into `CartManager` (add/remove), `PricingService` (discount/tax), `CheckoutService` (checkout), `CartPersistence` (save), `CartSharing` (share). Trade-off: more objects to wire together via DI. Benefit: each class is independently testable.

8. **Q: Your legacy system uses `new PayPalService()` directly in `OrderService`. Management wants to add Stripe. All code referencing PayPalService must change. How do you migrate without breaking production?**
   A: Apply the Strangler Fig pattern. First, introduce a `PaymentService` interface. Create `PaypalAdapter` implementing it. Delegate all calls from old `PayPalService` to the adapter. Now add `StripeAdapter`. All new code uses the interface. Later, remove `PayPalService` entirely. Trade-off: double maintenance during transition. Benefit: no production downtime.

9. **Q: A game has Character, Monster, NPC, and Item classes. Every class needs serialize(), deserialize(), display(), and clone(). Copy-pasting across classes leads to inconsistency. How do you DRY this?**
   A: Use the Prototype pattern for cloning (`clone()` in a base class or interface). For serialization/deserialization, use the Visitor pattern — a `SerializerVisitor` visits each object type. For display, use a `Renderer` strategy. Trade-off: Visitor adds complexity for each new type; consider reflection-based serialization if performance allows.

10. **Q: Your team ships a library used by 50 internal teams. Every breaking change to the public API requires 50 teams to update. How do you design the public API?**
    A: Apply the Facade pattern. The library has a single `PublicAPI` class that delegates to internal implementations. Mark all internal classes as package-private or use OSGi/JPMS module system. Use semantic versioning — breaking changes only in major versions. Trade-off: facade becomes a bottleneck; plan for versioned facades (v1, v2) using the Adapter pattern.

---

## Interview Questions

1. **What are the four pillars of OOP?**
   A: Encapsulation (hide data), Abstraction (hide complexity), Inheritance (is-a), Polymorphism (same interface, different behavior).

2. **What is the difference between composition and inheritance?**
   A: Composition (has-a) is more flexible — you can swap behaviors at runtime and avoid tight coupling. Inheritance (is-a) creates parent-child dependency; changing the parent can break all children.

3. **What is the Liskov Substitution Principle?**
   A: Subtypes must be substitutable for their base types without altering correctness. `Square extends Rectangle` violates LSP because changing width also changes height, breaking client expectations.

4. **What is the difference between method overloading and method overriding?**
   A: Overloading (compile-time polymorphism) — same method name, different parameters within the same class. Overriding (runtime polymorphism) — child redefines a parent method. Overriding uses dynamic dispatch; overloading uses static binding.

5. **What is the purpose of an interface vs an abstract class?**
   A: Interface defines a contract (what to do), no implementation. Abstract class provides partial implementation (both what and how). Use interfaces for capabilities, abstract classes for shared base logic.

6. **What is the Dependency Inversion Principle?**
   A: High-level modules should not depend on low-level modules. Both should depend on abstractions. E.g., `OrderService` depends on `PaymentGateway` interface, not `StripeGateway`.

7. **What is a design pattern? Give three examples.**
   A: Reusable solution to a common problem. Creational (Factory — object creation), Structural (Adapter — incompatible interfaces), Behavioral (Observer — one-to-many notification).

8. **What is the difference between aggregation and composition?**
   A: Both are "has-a" relationships. Composition implies ownership — if the parent is destroyed, the child is too (e.g., House → Rooms). Aggregation is weaker — the child can exist independently (e.g., Department → Professors).

9. **What is the Open/Closed Principle?**
   A: Classes should be open for extension but closed for modification. Add new behavior by creating new subclasses/implementations, not by modifying existing code.

10. **What is a god class and why is it a problem?**
    A: A class with too many responsibilities (e.g., 3000-line `OrderService` doing validation, DB, email, logging). Violates SRP — hard to test, understand, and change. Fix: split into focused classes.

---

## Developer Recommendations

- **Prefer composition over inheritance** — Inheritance creates tight coupling: changing the parent class can break all children. Composition (has-a) is more flexible — you can swap behaviors at runtime via strategy pattern. Use inheritance only for true is-a relationships where the child will never need to deviate.

- **Apply SOLID principles rigorously, but don't dogmatize** — SRP means one reason to change, not one method per class. Too many tiny classes create indirection hell. A `UserService` with `createUser`, `updateUser`, `deleteUser` is fine — they all change together when user requirements change.

- **Use dependency injection for testability** — `new Database()` in a constructor makes unit testing impossible without a real database. Constructor injection allows substituting mocks or test doubles. Trade-off: more boilerplate (constructor params, DI container config) vs the benefit of being able to test in isolation.

- **Prefer immutable objects** — Mutable shared state is the #1 source of bugs in OOP. Make fields `final`, use builders for complex construction, return defensive copies. Trade-off: immutable objects create more garbage (new object per mutation) but eliminate entire categories of bugs (race conditions, unexpected mutation).

- **Use the strategy pattern to eliminate if-else chains** — `if (type == "PDF") ... else if (type == "CSV")` for formatting logic. Replace with a `Formatter` interface and a map of type → formatter. Trade-off: more classes. Benefit: adding a new format is a new class without changing existing code.

- **Prefer small interfaces over large ones (Interface Segregation)** — A `Worker` interface with `work()`, `eat()`, `sleep()` forces `Robot` to implement `eat()` (or throw). Split into `Workable`, `Eatable`, `Sleepable`. Trade-off: more interfaces to manage. Benefit: no classes with empty or throwing method implementations.

- **Use the factory pattern to centralize complex construction** — When object creation requires conditionals, configuration, or dependency resolution, a factory encapsulates that complexity. Trade-off: one more layer of indirection. Benefit: callers don't need to know creation details, and construction logic isn't duplicated.

- **Keep inheritance hierarchies shallow (≤ 3 levels)** — Deep hierarchies are fragile: changing a grandparent method can break grandchildren. Prefer interface implementation over class extension. Trade-off: interfaces provide no shared implementation, so you may need default methods or helper classes.
