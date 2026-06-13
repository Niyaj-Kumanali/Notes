# Sealed Classes

## Overview

- **Definition** — Sealed classes and interfaces restrict which other classes or interfaces may extend or implement them, creating a fixed or bounded hierarchy.
- **Why It Exists** — To give the author of a base type control over its subtype hierarchy, enabling exhaustive pattern matching and preventing unknown extensions that could break invariants.
- **Historical Context** — Inspired by sealed types in Scala and Kotlin, introduced as a preview in JDK 15, received a second preview in JDK 16, and were finalized in JDK 17.
- **Key Concepts** — **sealed** keyword on class/interface; **permits** clause lists permitted subtypes; permitted subtypes must be in the same module or same named package; each subtype must be **final**, **sealed**, or **non-sealed**; **sealed interfaces** follow the same rules; **exhaustive pattern matching** works when all permitted subtypes are covered in a switch; compiler verifies completeness.

## Core Concepts

- Declaration uses the `sealed` modifier and a `permits` clause:

  ```java
  public sealed class Shape permits Circle, Square, Triangle { }
  ```

- Each class named in the `permits` clause must extend the sealed class and reside in the same module (or the same named package if no module declaration exists).
- A permitted subclass must declare one of three modifiers:
  - **final** — closes that branch; no further subclasses allowed.
  - **sealed** — continues the restriction with its own `permits` list.
  - **non-sealed** — reopens the hierarchy; any class can extend this subclass.
- Sealed interfaces work identically to sealed classes:

  ```java
  public sealed interface JsonValue permits JsonString, JsonNumber, JsonObject { }
  ```

- Pattern matching exhaustiveness: when a switch covers all permitted subtypes, the compiler does not require a `default` clause. If a new subtype is added to the `permits` clause, the compiler flags incomplete switches.
- Migration from non-sealed hierarchies: identify all known subtypes, declare the base class as `sealed`, list them in `permits`, and mark each as `final`, `sealed`, or `non-sealed` depending on whether you want to allow further growth.

## Common Mistakes

- **Forgetting to mark permitted subclasses as final, sealed, or non-sealed**
  - The compiler rejects the subclass with an error that it must specify one of the three modifiers.
  - **Why it looks correct:** In a standard hierarchy, a subclass needs only `extends`.
  - The fix: add `final`, `sealed`, or `non-sealed` to each permitted subclass.

- **Putting the sealed class and its subclasses in different modules**
  - The `permits` clause only works when subtypes are in the same module (or same unnamed package within the same package).
  - **Why it looks correct:** A regular public class can be extended anywhere on the classpath.
  - The fix: place all permitted subtypes in the same module as the sealed parent, or use module exports carefully.

- **Adding a new subtype without updating the permits clause**
  - The compiler rejects the new subtype because it is not listed in the parent's `permits`.
  - **Why it looks correct:** You only modified the subclass file; the parent seems unchanged.
  - The fix: always add the new subclass name to the parent's `permits` clause and recompile the parent.

- **Expecting sealed classes to prevent reflection-based instantiation**
  - Sealed classes restrict compile-time extension but do not stop runtime instantiation via reflection.
  - **Why it looks correct:** The word "sealed" implies a security boundary.
  - The fix: combine sealed classes with module system access controls or security manager rules if runtime enforcement is needed.

## Real-World Scenarios

### Domain Modeling with Fixed Variants

- Model a payment method where only CreditCard, PayPal, and Crypto are valid.

  ```java
  public sealed interface PaymentMethod permits CreditCard, PayPal, Crypto { }
  public record CreditCard(String lastFour, String expiry) implements PaymentMethod { }
  public record PayPal(String email) implements PaymentMethod { }
  public record Crypto(String walletAddress) implements PaymentMethod { }
  ```

- Every code path that handles `PaymentMethod` must cover all three variants, enforced by the compiler.

### Algebraic Data Types

- Represent a JSON value as a union of mutually exclusive types.

  ```java
  public sealed interface JsonNode permits JsonObject, JsonArray, JsonString, JsonNumber, JsonNull { }
  ```

- Switches on `JsonNode` are exhaustive; adding a new variant forces all switch sites to update.

### State Machine with Known States

- Model order lifecycle states where transitions are constrained.

  ```java
  public sealed interface OrderState permits Pending, Shipped, Delivered, Cancelled { }
  public enum Pending implements OrderState { INSTANCE }
  public enum Shipped implements OrderState { INSTANCE }
  public enum Delivered implements OrderState { INSTANCE }
  public enum Cancelled implements OrderState { INSTANCE }
  ```

- Exhaustive switches ensure no state is forgotten in business logic.

## Scenario-Based Questions

**Q: You are designing an API that processes geometric shapes. Currently, you have Circle, Square, and Triangle. Other developers in your organization may want to add new shapes later. Should you use a sealed class?**

- If the set of shapes is fixed and known at design time, sealed is appropriate. If you want to allow future unknown shapes, use `non-sealed` on a base subclass, or skip sealed entirely and use an open hierarchy with a default case in switches.
- **Interview follow-up:** How would you design the hierarchy if some shapes are known now but you want to allow a future plugin system to register new shapes?

**Q: You need to add a new permitted subtype to a sealed class after the API has been released. What happens to existing switch expressions in client code?**

- The compiler will flag any switch on the sealed type that is not exhaustive (i.e., does not cover the new subtype). Clients must add a case for the new subtype or add a `default` branch. This is the key benefit: the compiler tells every consumer what to update.
- **Interview follow-up:** Can you avoid breaking clients by using a `default` branch in your own code?

**Q: You are modeling a payment system where each payment method has different validation rules. How would you use sealed classes to enforce that all payment methods implement a `validate()` method?**

- Define a `sealed interface PaymentMethod` with an abstract `validate()` method, list permitted subtypes (e.g., `CreditCard`, `PayPal`, `Crypto`), and have each record implement `validate()`. Any switch on `PaymentMethod` must cover all subtypes, ensuring no method is missed.
- **Interview follow-up:** How would you add a new payment method type without recompiling existing code?

**Q: You have a sealed interface `Expression` with subtypes `Constant`, `Add`, `Subtract`, `Multiply`, `Divide`. How do you ensure that division by zero is handled at compile time?**

- Add a `Divide` record with `Expression left, Expression right` and handle the zero case with a guarded pattern: `case Divide(var l, var r) when r.eval() != 0`. The sealed hierarchy guarantees that no unknown expression type can appear, so the switch is exhaustive with these cases.
- **Interview follow-up:** If you add a `Modulo` subtype later, what compiler guarantees do you get?

**Q: You are migrating an existing class hierarchy to sealed classes. One of the subclasses is used as a base for third-party extensions. How do you handle this?**

- Mark that subclass as `non-sealed`. This allows third parties to extend it while the rest of the hierarchy remains sealed. Document the contract clearly to indicate which branches are open for extension.
- **Interview follow-up:** What are the security implications of using `non-sealed` in a sealed hierarchy?

**Q: How do sealed classes interact with Java records? Can a record be a permitted subtype?**

- Yes, records are perfect for permitted subtypes because they are implicitly final and provide transparent data carriers. For example, `record Circle(double radius) implements Shape { }` can be a permitted subtype of a sealed `Shape`.
- **Interview follow-up:** Can a sealed class itself be a record?

**Q: You have a sealed class `Vehicle` permitted to `Car`, `Bike`, `Truck`. A developer accidentally creates a new class `Bus` that extends `Vehicle` without adding it to the `permits` clause. What happens?**

- The compiler rejects `Bus` with an error stating that `Bus` is not allowed to extend `Vehicle` because it is not listed in the `permits` clause. The developer must add `Bus` to the `permits` clause and recompile `Vehicle`.
- **Interview follow-up:** What if `Bus` is in a different module?

**Q: You are designing a serialization framework that needs to handle a fixed set of data types. How would sealed classes help?**

- Define a `sealed interface DataType permits IntType, StringType, FloatType, BooleanType`. The framework can switch exhaustively over all types. Adding a new type requires updating the `permits` clause and all switch sites, preventing silent deserialization failures.
- **Interview follow-up:** How would you extend this design to support user-defined custom types?

**Q: How would you use sealed classes to model a Try monad (Success/Failure) pattern in Java?**

- Define `sealed interface Try<T> permits Success<T>, Failure<T>`. `Success` holds the value, `Failure` holds the exception. Switches on `Try` must cover both cases, eliminating forgotten error handling.
- **Interview follow-up:** How would you add a `Loading` state to this Try monad?

**Q: Your team uses an enum for days of the week, but you need to attach different data to each day. How would sealed classes provide a better solution?**

- Replace the enum with a `sealed interface DayOfWeek` permitting `Weekday` and `Weekend` records. Each record can hold custom data (e.g., `Weekday(boolean isEarlyShift)`). The hierarchy remains exhaustive and more flexible than an enum.
- **Interview follow-up:** What do you lose by switching from enum to sealed class in this case?

**Q: A junior developer uses a sealed class but forgets to add the sealed modifier to the parent class, only adding it to the subclasses. What error occurs?**

- If the parent class is not declared `sealed`, the `permits` clause is invalid. The compiler will report that the parent class cannot restrict its subclasses without the `sealed` modifier. All classes in a sealed hierarchy must start with the sealed parent.
- **Interview follow-up:** Can a non-sealed parent class have a sealed child?

## Interview Questions

- **What is the difference between a sealed class and a final class?**
  - A final class cannot be extended at all. A sealed class can be extended only by the classes listed in its `permits` clause, allowing a controlled, bounded hierarchy.

- **What does the non-sealed modifier mean in a permitted subclass?**
  - `non-sealed` re-opens the hierarchy. Any class can extend the `non-sealed` subclass, effectively removing the sealing restriction for that branch of the hierarchy.

- **How does the compiler use sealed types to verify exhaustiveness in switch expressions and pattern matching?**
  - The compiler knows the complete set of permitted subtypes. It checks that every permitted subtype has a matching case in the switch. If a case is missing or a new subtype is added later, the compiler reports an error.

- **Can a sealed class be abstract?**
  - Yes, a sealed class can be abstract. The abstract class defines a restricted set of concrete subtypes via the `permits` clause.

- **Can a sealed interface extend another sealed interface?**
  - Yes. A sealed interface can extend another sealed interface. The extending interface may add further restrictions or open branches via its own `permits` clause.

- **What happens if a permitted subclass is in a different package than the sealed class?**
  - Permitted subclasses must be in the same module. If no module declaration exists, they must be in the same named package. Cross-package inheritance in the same module is allowed.

- **Can a sealed class be instantiated directly if it is not abstract?**
  - If a sealed class is concrete (not abstract), it can be instantiated directly via its constructor. However, its permitted subclasses can also be instantiated.

- **How do you reflectively discover all permitted subclasses of a sealed type at runtime?**
  - Use `Class.isSealed()` and `Class.getPermittedSubclasses()` to get an array of `Class<?>` objects representing the permitted subtypes at runtime.

- **What is the relationship between sealed classes and pattern matching exhaustiveness?**
  - Sealed classes enable compile-time exhaustiveness checking in switch expressions and statements. The compiler knows all permitted subtypes and can verify that every possible case is handled.

- **Can a permitted subclass be a local class or anonymous class?**
  - No. Permitted subclasses must be top-level classes or nested classes. Anonymous and local classes cannot be listed in a `permits` clause because they do not have a name that can be referenced.

- **What is the difference between a sealed class and a package-private class hierarchy?**
  - A package-private class can only be extended within the same package, but this is a visibility restriction, not a hierarchy restriction. Sealed classes explicitly list permitted subtypes and work across packages within a module.

- **Can a sealed class be a nested class?**
  - Yes. A sealed class can be nested inside another class or interface. The `permits` clause refers to other nested types or top-level types as usual.

- **Does the compiler generate a synthetic constructor for sealed classes?**
  - No. Sealed classes use regular constructors. The sealing information is stored in the class file as a `PermittedSubclasses` attribute, not in the constructor.

- **How do sealed classes improve API design compared to traditional inheritance?**
  - They provide a documented, compiler-enforced contract of which subtypes exist. API consumers can rely on exhaustive matching, and the API author controls evolution of the hierarchy.

- **Can a sealed class be used with Java modules to restrict visibility further?**
  - Yes. Combining sealed classes with module exports gives fine-grained control: the sealed type can be exported while specific permitted subtypes are not, forcing users to program against the sealed type only.

- **What is the purpose of the `non-sealed` modifier?**
  - The `non-sealed` modifier reopens a sealed hierarchy branch. Any class can extend a `non-sealed` subclass, allowing third-party extension while other branches remain closed.

- **Can a sealed class have multiple levels of hierarchy?**
  - Yes. A sealed class can have permitted subclasses that are themselves sealed, creating a multi-level hierarchy where each level restricts its own subtypes.

- **How does the `permits` clause affect compilation order?**
  - The sealed class must be compiled before or at the same time as its permitted subclasses. If the sealed class is a library dependency, adding new permitted subclasses requires recompiling the sealed class.

- **Can a sealed class be declared without a `permits` clause if subclasses are in the same file?**
  - Yes. If all permitted subclasses are declared in the same source file, the `permits` clause can be omitted. The compiler infers the permitted subtypes from the subclasses in the file.

- **What happens if a permitted subclass is declared `final`?**
  - A `final` permitted subclass closes that branch of the hierarchy. No further subclasses are allowed under that branch.

## Developer Recommendations

- **Use sealed classes for domain models with a fixed set of variants**
  - The compiler enforces that all variants are handled everywhere, eliminating runtime "unknown type" errors.
  - Declare the base type as `sealed`, list all variants in `permits`, and mark each variant as `final` if the tree should not grow further.

- **Use non-sealed to create extension points within a sealed hierarchy**
  - When most variants are known but one branch should remain open, mark that subclass as `non-sealed`.
  - Document that the `non-sealed` branch allows third-party extensions while the rest of the hierarchy is fixed.

- **Combine sealed classes with pattern matching for exhaustive switches**
  - Write switch expressions on the sealed type without a `default` clause; the compiler verifies completeness.
  - When a new variant is added, compiler errors guide you to every switch site that needs updating.
  - **Production story:** A team modeling financial instrument types as sealed interfaces caught a missing variant at compile time during a refactor, preventing a production outage that would have occurred with an enum-based approach.
