# Switch Expressions

## Overview

- **Definition** — A switch expression is a multi-way branch that can be used as an expression (assigned to a variable) or as a statement, supporting both arrow syntax and colon syntax with yield.
- **Why It Exists** — To provide a more concise, less error-prone alternative to traditional switch statements by eliminating fall-through, supporting exhaustiveness checks, and enabling pattern matching.
- **Historical Context** — Previewed in JDK 12, refined in JDK 13 (replacing `break` with `yield` for value return), and finalized in JDK 14. Pattern matching for switch was previewed starting JDK 17 and was finalized in JDK 21.
- **Key Concepts** — **Arrow syntax** (`case ->`) has no fall-through; **colon syntax** (`case :`) retains fall-through and uses `yield` to return a value; **switch as expression** assigns a value to a variable; **yield** returns a value from a colon-syntax block; **exhaustive** switch must cover all possible inputs; **null handling** via `case null`; **pattern matching in switch** allows type-based dispatch; **when clauses** are cleaner than if-else chains.

## Core Concepts

- Arrow syntax (`case ->`) is concise and does not fall through:

  ```java
  String result = switch (day) {
    case MONDAY, FRIDAY -> "Work";
    case SATURDAY, SUNDAY -> "Rest";
    default -> "Midweek";
  };
  ```

- Colon syntax (`case :`) retains traditional fall-through and uses `yield` to return a value:

  ```java
  String result = switch (day) {
    case MONDAY:
    case FRIDAY:
      yield "Work";
    case SATURDAY:
    case SUNDAY:
      yield "Rest";
    default:
      yield "Midweek";
  };
  ```

- Switch as an expression: the entire `switch` produces a value that can be assigned, passed as an argument, or used in a larger expression.
- `yield` returns a value from a colon-syntax case block. Arrow-syntax cases do not need `yield` for single expressions; they need `yield` only inside a block:

  ```java
  int val = switch (x) {
    case 1 -> 10;
    case 2 -> { yield 20; }
    default -> 0;
  };
  ```

- Exhaustiveness: a switch expression must cover all possible values of the selector. For enums and sealed types, the compiler verifies that every constant or permitted subtype has a case.
- Null handling: use `case null ->` to match null directly; without it, a null selector throws `NullPointerException`.

  ```java
  String s = switch (obj) {
    case null  -> "null";
    case String str -> str;
    default    -> "other";
  };
  ```

- Pattern matching in switch (JDK 17+): cases can use type patterns, record patterns, and guarded patterns:

  ```java
  String describe = switch (obj) {
    case Integer i && i > 0 -> "positive int";
    case Integer i          -> "non-positive int";
    case String s           -> "string of length " + s.length();
    case null               -> "null";
    default                 -> "unknown";
  };
  ```

- When expressions using switch are cleaner than nested if-else chains for multi-way type dispatch:

  ```java
  // Cleaner than if-else instanceof chain
  return switch (shape) {
    case Circle c   -> Math.PI * c.radius() * c.radius();
    case Square s   -> s.side() * s.side();
    case null       -> 0.0;
  };
  ```

## Common Mistakes

- **Using break instead of yield in a colon-syntax switch expression**
  - `break` with a value is not valid in a switch expression; the compiler expects `yield`.
  - **Why it looks correct:** Traditional switch statements use `break` to exit a case.
  - The fix: use `yield value;` to return a value from a colon-syntax case, or switch to arrow syntax.

- **Assuming arrow syntax cases fall through**
  - Arrow syntax cases do not fall through. Each case handles only its own label.
  - **Why it looks correct:** Colon syntax has always had fall-through; the arrow looks similar.
  - The fix: list multiple labels separated by commas on a single arrow case, e.g. `case MONDAY, FRIDAY ->`.

- **Omitting a default branch for non-sealed or non-enum types**
  - If the selector type is not sealed and not an enum, a switch expression must have a `default` branch to be exhaustive.
  - **Why it looks correct:** Traditional switch statements do not require exhaustiveness.
  - The fix: always provide a `default` for non-sealed or non-enum types, or use `default -> throw new IllegalArgumentException()`.

- **Forgetting that null throws NPE in a switch expression without case null**
  - A switch expression that doesn't have `case null` will throw `NullPointerException` if the selector is null.
  - **Why it looks correct:** In a traditional switch, null also throws NPE, but the developer may not realize this applies to expressions too.
  - The fix: add `case null ->` at the top of the switch to handle the null case explicitly.

- **Using switch expression as a statement but forgetting to handle the return value**
  - A switch expression produces a value; ignoring it is allowed but usually signals a bug.
  - **Why it looks correct:** A switch statement does not produce a value.
  - The fix: if you don't need the value, use a traditional switch statement, or explicitly discard the value with a comment.

## Real-World Scenarios

### HTTP Status Code Mapping

- Map status code enums to user-friendly messages using arrow syntax:

  ```java
  String message = switch (status) {
    case 200 -> "OK";
    case 201 -> "Created";
    case 301, 302 -> "Redirect";
    case 400 -> "Bad Request";
    case 401 -> "Unauthorized";
    case 403 -> "Forbidden";
    case 404 -> "Not Found";
    case 500 -> "Internal Server Error";
    default -> "Unknown " + status;
  };
  ```

### Command Dispatch

- Dispatch commands based on parsed input without if-else chains:

  ```java
  Result execute(Command cmd) {
    return switch (cmd) {
      case Add(var a, var b) -> Result.of(a + b);
      case Sub(var a, var b) -> Result.of(a - b);
      case Mul(var a, var b) -> Result.of(a * b);
      case Div(var a, var b) when b != 0 -> Result.of(a / b);
      case Div(var a, var b) -> Result.error("Division by zero");
      case null -> Result.error("No command");
    };
  }
  ```

### Enum-Based State Transitions

- Compute next state in a state machine using switch expression:

  ```java
  OrderState nextState(OrderState current, Event event) {
    return switch (current) {
      case PENDING -> switch (event) {
        case PAY -> CONFIRMED;
        case CANCEL -> CANCELLED;
        default -> throw new IllegalStateException();
      };
      case CONFIRMED -> switch (event) {
        case SHIP -> SHIPPED;
        case CANCEL -> CANCELLED;
        default -> throw new IllegalStateException();
      };
      default -> throw new IllegalStateException("No transition from " + current);
    };
  }
  ```

## Scenario-Based Questions

**Q: You are reviewing a pull request where the developer used a switch expression to replace an if-else chain that maps user roles to permissions. There are three roles (ADMIN, USER, GUEST) and the enum is not sealed. What exhaustiveness issue might exist?**

- If the switch expression has no `default` branch, adding a new role in the future will cause a compile error. If it has a `default` branch, new roles are silently handled by the default, which may be incorrect. The best practice is to use a `default` that throws an exception or to ensure the enum is never extended.
- **Interview follow-up:** If the role enum is later changed to a sealed interface to support custom roles, how would the switch expression change?

**Q: How would you handle a switch expression where two different patterns need the same implementation but one pattern has a guard condition?**

- Combine the labels with a comma for the non-guarded case and use a separate guarded case for the condition. Arrow syntax evaluates cases in order, so the guarded case must come before the unguarded one to avoid the guard being shadowed.

  ```java
  String desc = switch (obj) {
    case String s && s.isEmpty() -> "empty";
    case String s -> "non-empty";
    default -> "other";
  };
  ```

- **Interview follow-up:** What happens if you swap the order of the two String cases?

**Q: You have a legacy switch statement that uses fall-through with multiple `case` labels and a shared block. How do you migrate it to a modern switch expression?**

- Replace the fall-through pattern with a single `case` using comma-separated labels: `case MONDAY, TUESDAY, WEDNESDAY -> "weekday"`. If the original had intentional fall-through to execute shared code and then continue, restructure the logic to avoid needing fall-through behavior.
- **Interview follow-up:** What if you need to execute some common code before returning a value specific to each case?

**Q: You are writing a switch expression that maps HTTP status codes to messages. How do you handle ranges of values efficiently?**

- List individual codes in comma-separated labels for fixed mappings and use `default` for the rest. Switch expressions do not support range patterns, so you must enumerate each code or use a catch-all default: `case 200 -> "OK"; case 201 -> "Created"; case 301, 302, 307 -> "Redirect"; default -> "Unknown"`.
- **Interview follow-up:** How would you handle hundreds of status codes without a giant switch?

**Q: A developer uses `switch` as a statement (not expression) with pattern matching but forgets to handle all sealed subtypes. Does the compiler catch this?**

- For switch statements, exhaustiveness is not required — missing cases simply do nothing at runtime. However, with pattern matching, the compiler will issue a warning if the switch is not exhaustive. Switch expressions enforce exhaustiveness as a compile error.
- **Interview follow-up:** When would you intentionally use a switch statement over a switch expression?

**Q: You need to implement a discount calculator where different customer tiers get different percentage discounts. How would you use a switch expression for this?**

- Define a sealed `CustomerTier` hierarchy or enum, then: `return switch (tier) { case PREMIUM -> 0.2; case GOLD -> 0.15; case SILVER -> 0.1; case BRONZE -> 0.05; case null -> 0.0; }`. The switch is exhaustive and handles null safely.
- **Interview follow-up:** How would you add a loyalty points factor that modifies the discount?

**Q: Your team uses a switch expression that yields a `String` from a pattern match. One case throws an exception. Does the compiler require that throwing case to also have a return value?**

- No. A case that throws an exception (or returns via `throw`) does not need to yield a value. The compiler accepts `case Integer i -> throw new IllegalArgumentException("unsupported")` because throwing terminates normally in the compiler's analysis.
- **Interview follow-up:** What about a case that has a `return` statement instead of yield?

**Q: You have a switch expression on an enum where every constant is covered. Is a `default` branch required?**

- If every enum constant is explicitly covered, the compiler accepts the switch without a `default`. However, if the enum is later extended, the switch will break. A `default` that throws an exception is often safer for API enums.
- **Interview follow-up:** How do you document that adding a new enum constant requires updating the switch?

**Q: You need a switch expression that returns different types based on the input, but switch must have a consistent return type. How do you handle heterogeneous return types?**

- Use a common supertype or sealed interface for the return value. Each case returns a subtype of that common type. For example, `sealed interface Result permits TextResult, NumberResult` with switch cases returning `TextResult` or `NumberResult`.
- **Interview follow-up:** What if you need to return completely unrelated types?

**Q: A junior developer writes a switch expression with colon syntax and uses `break` instead of `yield`. How do you explain the difference?**

- Explain that `break` exits a switch statement without producing a value. `yield` returns a value from a colon-syntax case in a switch expression. The compiler will reject `break value;` in a switch expression, producing a clear error message.
- **Interview follow-up:** Can you mix arrow and colon syntax in the same switch?

**Q: You are writing a switch with many cases that all share some logic before returning a distinct value. How do you avoid duplication?**

- Extract the shared logic into a helper method called from each case, or use a `default` delegation pattern: `case A, B, C -> sharedProcessing("specificResult")`. For complex cases, consider a `Map` of handlers instead of a large switch.
- **Interview follow-up:** When would a Map of handlers be preferable to a switch expression?

## Interview Questions

- **What is the difference between break and yield in a switch expression?**
  - `break` exits a switch statement without producing a value. `yield` returns a value from a colon-syntax case in a switch expression and transfers control to the caller of the expression. Arrow-syntax cases do not use `yield` for single-expression bodies.

- **Is a switch expression required to be exhaustive?**
  - Yes. A switch expression must cover all possible values of the selector. For enums, the compiler checks that every constant has a case or that a `default` exists. For sealed types, every permitted subtype must have a case. For other types, a `default` is required.

- **How does null behave in a switch expression?**
  - If the selector is null and no `case null` is present, the switch expression throws `NullPointerException`. A `case null` label must be the first case when used. Traditional switch statements also throw NPE on null selectors.

- **Can you use pattern matching with switch for non-sealed types?**
  - Yes, you can use type patterns, record patterns, and guarded patterns in a switch regardless of whether the selector type is sealed. However, the switch must still be exhaustive, which typically means adding a `default` branch for non-sealed types.

- **What is the difference between a switch expression and a switch statement?**
  - A switch expression produces a value and must be exhaustive. A switch statement does not produce a value and does not require exhaustiveness. Switch expressions use `yield` or arrow syntax with single expressions; switch statements use `break` or fall-through.

- **Can a switch expression throw an exception instead of returning a value?**
  - Yes. A case can throw an exception via `throw`. The compiler accepts this because the method terminates exceptionally, which satisfies the requirement that every execution path produces a result or throws.

- **How do you handle multiple conditions that yield the same result in arrow syntax?**
  - Use a comma-separated list of labels: `case MONDAY, TUESDAY, WEDNESDAY -> "weekday"`. This is cleaner than having separate cases that fall through to the same block.

- **What is the purpose of the `yield` keyword in a switch expression?**
  - `yield` returns a value from a colon-syntax case block in a switch expression. It was introduced in JDK 13 as a replacement for `break value` and is scoped to the switch expression only.

- **Can you use a switch expression in a method argument?**
  - Yes. Since a switch expression is an expression, it can be passed directly as an argument: `process(switch (val) { case 1 -> "one"; default -> "other"; })`.

- **How does the compiler determine the type of a switch expression?**
  - The compiler computes the least upper bound of all yielded values. If cases yield `String` and `null`, the type is `String`. If cases yield `Integer` and `Double`, the type is `Number & Comparable<?>`.

- **What happens if a switch expression has unreachable cases?**
  - The compiler reports an error for unreachable cases. For example, placing `case String s` after `case CharSequence cs` is unreachable because `String` is a subtype of `CharSequence` and the first pattern dominates.

- **Can you use `var` as the selector type in a switch expression?**
  - No, `var` cannot be used as the selector type because the switch selector must have a known type at compile time to check exhaustiveness and perform pattern matching.

- **How do you handle enums with switch expressions across library versions?**
  - Adding a new enum constant breaks switch expressions without `default`. The best practice is to include a `default` branch that throws an exception, documenting that enum consumers must update their switches.

- **What is the difference between `case null` and a null check before the switch?**
  - `case null` integrates null handling into the switch structure, making it visible and part of the exhaustive check. A separate null check before the switch is equivalent but less cohesive.

- **Can a switch expression have empty case bodies?**
  - No. Each case in a switch expression must produce a value or throw. Empty bodies are not allowed. In a switch statement, empty bodies are allowed and fall through.

- **How does scope work for variables declared inside a switch expression case?**
  - Variables declared in a case block are scoped to that block. They are not visible in other cases. In arrow syntax, the scope is the expression or block on the right side of `->`.

- **Can you use switch expressions with `String` selectors?**
  - Yes. Switch expressions support `String` selectors. String comparison uses `equals()` matching, and all the same exhaustiveness rules apply with a required `default` branch.

- **What is the difference between pattern matching dominance and traditional switch ordering?**
  - Traditional switch ordering matters only for fall-through. Pattern matching ordering matters for which case matches first. A more general pattern (e.g., `Object o`) placed before a specific pattern (e.g., `String s`) dominates it and makes the specific case unreachable.

- **How do you convert a traditional switch with fall-through to a modern switch expression?**
  - Replace fall-through patterns with comma-separated labels on a single arrow case. Replace shared code blocks with helper methods. Replace `break value` with `yield` for colon syntax or arrow syntax.

- **What happens if a switch expression case throws an exception in arrow syntax?**
  - The exception propagates normally. The arrow syntax with a block allows `throw`: `case 1 -> { throw new RuntimeException("error"); }`. This is valid because throwing terminates the expression normally in compiler analysis.

## Developer Recommendations

- **Prefer arrow syntax over colon syntax for new code**
  - Arrow syntax eliminates accidental fall-through, is more readable, and does not require `yield` for simple expressions.
  - Use colon syntax only when migrating legacy switch statements where fall-through behavior is intentionally relied upon.

- **Always include a default branch for non-sealed types**
  - Even when you believe all cases are covered, a `default` branch protects against future additions to the type hierarchy.
  - Throw an `IllegalArgumentException` or `IllegalStateException` in the default to make unexpected values visible immediately.

- **Add an explicit case null to handle null selectors**
  - Prevents silent `NullPointerException` and makes the null behavior visible in the switch structure.
  - Place `case null` as the first case for clarity.

- **Use switch expressions to replace if-else instanceof chains**
  - A switch with type patterns is more concise and visually separates each type branch.
  - Combine with sealed types to get compile-time exhaustiveness guarantees.
  - **Production story:** A payment processing service replaced a 40-line if-else chain with a switch expression on a sealed PaymentMethod type, reducing the code to 12 lines and immediately catching a missing payment method branch at compile time.
