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

## Interview Questions

- **What is the difference between break and yield in a switch expression?**
  - `break` exits a switch statement without producing a value. `yield` returns a value from a colon-syntax case in a switch expression and transfers control to the caller of the expression. Arrow-syntax cases do not use `yield` for single-expression bodies.

- **Is a switch expression required to be exhaustive?**
  - Yes. A switch expression must cover all possible values of the selector. For enums, the compiler checks that every constant has a case or that a `default` exists. For sealed types, every permitted subtype must have a case. For other types, a `default` is required.

- **How does null behave in a switch expression?**
  - If the selector is null and no `case null` is present, the switch expression throws `NullPointerException`. A `case null` label must be the first case when used. Traditional switch statements also throw NPE on null selectors.

- **Can you use pattern matching with switch for non-sealed types?**
  - Yes, you can use type patterns, record patterns, and guarded patterns in a switch regardless of whether the selector type is sealed. However, the switch must still be exhaustive, which typically means adding a `default` branch for non-sealed types.

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
