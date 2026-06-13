# Pattern Matching

## Overview

- **Definition** — Pattern matching allows you to test a value against a pattern, binding variables and extracting components in a single construct, reducing the need for casts and manual decomposition.
- **Why It Exists** — To eliminate verbose, error-prone chains of `instanceof` checks followed by casts, enabling concise and safe data exploration in conditionals and switch constructs.
- **Historical Context** — Pattern matching for `instanceof` was previewed in JDK 14, finalized in JDK 16. Record patterns previewed in JDK 19, finalized in JDK 21. Pattern matching for switch was previewed in JDK 17 and continued to evolve through JDK 21.
- **Key Concepts** — **Type pattern** (`obj instanceof String s`) binds a variable in scope; **guarded patterns** combine a pattern with `&&` for additional conditions; **record patterns** deconstruct records in patterns; **nested patterns** match multi-level record structures; **switch pattern matching** with sealed classes enables **exhaustiveness**; **null handling** requires an explicit `null` case; **var** in patterns matches any type.

## Core Concepts

- Pattern matching for `instanceof` (JDK 16 final):

  ```java
  if (obj instanceof String s) {
    System.out.println(s.length());
  }
  ```

  The variable `s` is in scope only when the pattern matches.
- Guarded patterns combine a type pattern with a boolean condition using `&&`:

  ```java
  if (obj instanceof String s && s.length() > 5) {
    System.out.println("Long string: " + s);
  }
  ```

- Record patterns (JDK 21 final) deconstruct a record into its components:

  ```java
  if (obj instanceof Point(int x, int y)) {
    System.out.println("x=" + x + ", y=" + y);
  }
  ```

- Nested patterns allow matching at multiple levels:

  ```java
  if (obj instanceof Rectangle(Point(int x, int y), var size)) {
    System.out.println("Rectangle at " + x + "," + y);
  }
  ```

- Pattern matching in switch (JDK 17+):

  ```java
  String formatted = switch (obj) {
    case Integer i -> "int " + i;
    case String s -> "string " + s;
    case null     -> "null";
    default       -> "unknown";
  };
  ```

- With sealed classes, the compiler can verify exhaustiveness:

  ```java
  String area = switch (shape) {
    case Circle c   -> Math.PI * c.radius() * c.radius();
    case Square s   -> s.side() * s.side();
    case Triangle t -> t.base() * t.height() / 2;
  };
  ```

- `null` does not match any pattern by default. A dedicated `case null` is required to handle null values without a `default` trap.
- The `var` keyword in a record pattern matches any type for that component:

  ```java
  if (obj instanceof Pair(var left, var right)) { }
  ```

## Common Mistakes

- **Forgetting that null does not match any pattern**
  - Writing `if (obj instanceof String s)` when `obj` can be null will skip the branch even if the type matches.
  - **Why it looks correct:** `instanceof` traditionally returns false for null, and the pattern syntax looks similar.
  - The fix: add a separate null check before or use `case null` in a switch.

- **Using guarded patterns with `&` instead of `&&`**
  - The guard operator in patterns is `&&`, not a single `&`.
  - **Why it looks correct:** `&` is a valid operator in Java for bitwise AND or non-short-circuit boolean AND.
  - The fix: always use `&&` in guarded patterns.

- **Assuming pattern variables are in scope after the if-else statement**
  - Pattern variables are scoped to the block where the pattern matches. They are not available after the if statement unless the compiler can prove they were assigned.
  - **Why it looks correct:** Regular variable declarations are scoped to the enclosing block.
  - The fix: use the pattern variable inside the matching block or combine with `&&` in a single condition.

- **Forgetting to handle all sealed subtypes in a switch expression**
  - A switch expression must be exhaustive. Missing a permitted subtype causes a compile error.
  - **Why it looks correct:** Traditional switch statements do not require exhaustiveness.
  - The fix: add cases for each sealed subtype, or add a `default` branch to handle unknown types.

## Real-World Scenarios

### Type-Safe JSON Processing

- Parse a JSON tree using a sealed `JsonNode` hierarchy and pattern matching:

  ```java
  String extractText(JsonNode node) {
    return switch (node) {
      case JsonString s  -> s.value();
      case JsonNumber n  -> n.value().toString();
      case JsonNull _    -> "null";
      case JsonArray a   -> a.elements().stream().map(this::extractText).collect(Collectors.joining(","));
      case JsonObject o  -> "object";
    };
  }
  ```

### Hierarchical Data Decomposition

- Process a nested tree of nodes where each node is a record:

  ```java
  record Node(String name, List<Node> children) { }

  void print(Node node) {
    if (node instanceof Node(String name, List<Node> children)) {
      System.out.println(name + " has " + children.size() + " children");
    }
  }
  ```

### Expression Evaluator

- Evaluate an abstract syntax tree using record patterns and sealed types:

  ```java
  sealed interface Expr permits Const, Add, Mul { }
  record Const(int value) implements Expr { }
  record Add(Expr left, Expr right) implements Expr { }
  record Mul(Expr left, Expr right) implements Expr { }

  int eval(Expr e) {
    return switch (e) {
      case Const(var v)         -> v;
      case Add(var l, var r)    -> eval(l) + eval(r);
      case Mul(var l, var r)    -> eval(l) * eval(r);
    };
  }
  ```

## Scenario-Based Questions

**Q: You have a method that receives an Object and needs to extract data from a deeply nested record structure like `Wrapper(Container(Point(int x, int y), String label))`. How would you implement this without explicitly calling getters?**

- Use a nested record pattern: `if (obj instanceof Wrapper(Container(Point(var x, var y), var label)))`. This deconstructs all levels in a single pattern match, binding `x`, `y`, and `label` directly.
- **Interview follow-up:** What happens if `Container` is null when you use a nested record pattern?

**Q: You are refactoring a series of if-else instanceof checks into a switch expression. One of the checked types has subtypes that are not covered. How does the compiler help you?**

- The compiler reports an error that the switch is not exhaustive. If the type is sealed, the error lists the missing permitted subtypes. If it is not sealed, you must add a `default` branch.
- **Interview follow-up:** When would you intentionally omit a `default` branch in a switch expression?

## Interview Questions

- **What is the scope of a pattern variable bound in an instanceof expression?**
  - The pattern variable is in scope where the compiler can prove the pattern matched. For `if (x instanceof String s)`, `s` is in scope inside the if block and any subsequent `&&` conditions. It is not available after the if-else chain unless the compiler can prove all branches assign it.

- **How do record patterns interact with generic record types?**
  - Record patterns use type inference. If you have `record Box<T>(T value) { }`, you can write `if (obj instanceof Box(var v))` to infer the type of `v` from the component type at runtime. However, generics are erased, so the pattern matches only the raw record type; the component type is unchecked.

- **What is a guarded pattern and how does it differ from a regular type pattern with an if condition?**
  - A guarded pattern combines the type check and the condition in a single pattern: `case String s && s.length() > 5`. In a switch, this avoids nesting a second condition inside the case body. The guard is part of the pattern, so a failed guard moves to the next case rather than falling through.

- **Can you use pattern matching with arrays?**
  - No. Array types are not records and do not support deconstruction patterns. You must manually access array elements.

## Developer Recommendations

- **Replace instanceof-cast chains with pattern matching for instanceof**
  - Reduces duplication and eliminates the risk of casting to the wrong type after the check.
  - Use `if (obj instanceof String s)` instead of `if (obj instanceof String) { String s = (String) obj; ... }`.

- **Use record patterns for deconstructing nested data**
  - When processing tree or composite structures, nested record patterns make the code read like the data shape.
  - Combine with sealed types and switch to ensure all variants are handled.

- **Use guarded patterns for precise case selection in switch**
  - Instead of filtering inside the case body, add the condition as a guard in the pattern label.
  - Keeps each case focused on a single pattern-condition pair and makes missed conditions visible at a glance.
  - **Production story:** A team building an expression evaluator replaced a chain of 20 if-else instanceof checks with a sealed interface and pattern matching switch, reducing the method length from 80 lines to 15 and eliminating a bug where a new expression type was silently ignored.
