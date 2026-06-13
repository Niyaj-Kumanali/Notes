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

## Interview Questions

- **What is the difference between a sealed class and a final class?**
  - A final class cannot be extended at all. A sealed class can be extended only by the classes listed in its `permits` clause, allowing a controlled, bounded hierarchy.

- **What does the non-sealed modifier mean in a permitted subclass?**
  - `non-sealed` re-opens the hierarchy. Any class can extend the `non-sealed` subclass, effectively removing the sealing restriction for that branch of the hierarchy.

- **How does the compiler use sealed types to verify exhaustiveness in switch expressions and pattern matching?**
  - The compiler knows the complete set of permitted subtypes. It checks that every permitted subtype has a matching case in the switch. If a case is missing or a new subtype is added later, the compiler reports an error.

- **Can a sealed class be abstract?**
  - Yes, a sealed class can be abstract. The abstract class defines a restricted set of concrete subtypes via the `permits` clause.

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
