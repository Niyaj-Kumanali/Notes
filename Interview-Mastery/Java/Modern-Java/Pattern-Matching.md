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

## Use Cases

Pattern matching transforms type checks and destructuring from verbose, error-prone idioms into concise, compiler-verified expressions.

- **Type-safe `instanceof` chains** — Replace cascading `if-else` or `Visitor` patterns with a single expression.
  - Use `if (obj instanceof String s)` or pattern-matching `switch` to bind the variable and check the type atomically. Example: processing a heterogeneous `List<Object>` where each element needs specific handling.
  - **Avoid when:** the type set is open and a default fallback is acceptable — pattern matching with sealed types gives compile-time exhaustiveness; an open hierarchy needs a `default` branch.

- **Record deconstruction** — Extract nested fields from a record hierarchy in one step.
  - Match `Wrapping(Container(Point(int x, int y), String label))` to destructure all levels at once. Example: JSON-like nested data structures processed in a rules engine.
  - **Avoid when:** only a top-level field is needed — a simple accessor call is clearer than a nested pattern.

- **Exhaustive switches over sealed types** — Guarantee every permitted subtype is handled; the compiler flags any omission.
  - Write a switch expression over a sealed interface without a `default` branch. Example: processing each `Shape` variant (Circle, Rect, Triangle) in a drawing application.
  - **Avoid when:** the type hierarchy is not sealed — a `default` branch is required and the compiler cannot enforce exhaustiveness.

- **Guarded patterns** — Combine a type check with an additional condition without nested `if` statements.
  - Write `case Circle c when c.radius() > 0 -> ...` to apply the branch only when the radius is positive. Example: validating domain objects during deserialization.
  - **Avoid when:** the guard condition is expensive or has side effects — guards are evaluated for every matching case, so pre-compute the condition if possible.

## Scenario-Based Questions

**Q: You have a method that receives an Object and needs to extract data from a deeply nested record structure like `Wrapper(Container(Point(int x, int y), String label))`. How would you implement this without explicitly calling getters?**

- Use a nested record pattern: `if (obj instanceof Wrapper(Container(Point(var x, var y), var label)))`. This deconstructs all levels in a single pattern match, binding `x`, `y`, and `label` directly.
- **Interview follow-up:** What happens if `Container` is null when you use a nested record pattern?

**Q: You are refactoring a series of if-else instanceof checks into a switch expression. One of the checked types has subtypes that are not covered. How does the compiler help you?**

- The compiler reports an error that the switch is not exhaustive. If the type is sealed, the error lists the missing permitted subtypes. If it is not sealed, you must add a `default` branch.
- **Interview follow-up:** When would you intentionally omit a `default` branch in a switch expression?

**Q: You are processing an AST where nodes can be `BinaryOp`, `UnaryOp`, `Literal`, or `Variable`. How would you use pattern matching to evaluate this tree?**

- Define a sealed `Expr` interface with those four permitted subtypes, each as a record. Use a switch expression with record patterns: `case BinaryOp(var left, var op, var right)`, `case UnaryOp(var op, var operand)`, `case Literal(var val)`, `case Variable(var name)`. The switch is exhaustive and each case deconstructs the node.
- **Interview follow-up:** How would you add short-circuit evaluation for Boolean AND/OR using guarded patterns?

**Q: A method receives a `Map<String, Object>` and needs to extract values of different types with null-safe handling. How can pattern matching help?**

- Use pattern matching in a switch on the map value: `case String s -> processString(s)`, `case Integer i -> processInt(i)`, `case null -> handleNull()`. Combined with `Map.getOrDefault`, you can handle missing keys by providing a sentinel value that maps to a default case.
- **Interview follow-up:** How would you handle the case where the map returns a `List` that might need different processing based on element types?

**Q: You have a deeply nested record structure representing an XML document. How do you extract specific elements without writing multiple nested if-statements?**

- Use nested record patterns in a single `if` or `switch`: `if (doc instanceof Document(Root(var children)))` binds the children directly. For conditional extraction, combine with guarded patterns: `case Element(String name, var attrs) when name.equals("target")`.
- **Interview follow-up:** How does the compiler handle nulls in nested record patterns?

**Q: You are using pattern matching in a switch that handles both `String` and `Integer`. What happens if the selector is null without a `case null`?**

- Without a `case null`, the switch expression throws `NullPointerException`. Pattern matching in switch does not match null by default. Always add `case null ->` as the first case to handle null selectors explicitly.
- **Interview follow-up:** What is the ordering requirement for `case null` relative to other string patterns?

**Q: A team member writes `if (obj instanceof String s & s.length() > 5)` using `&` instead of `&&`. Why does this fail?**

- The guard operator in pattern matching is `&&`, not `&`. `&` is a bitwise operator that requires both sides to be boolean, but `instanceof String s` is not a boolean expression — it is a pattern. The compiler will reject the code with a syntax error.
- **Interview follow-up:** Can you use `|` as a guard operator for alternative patterns?

**Q: You need to handle different subtypes of a sealed `Shape` class. How does pattern matching with sealed types improve safety over traditional instanceof checks?**

- With sealed types and pattern matching in switch, the compiler verifies exhaustiveness at compile time. Traditional instanceof checks can silently miss a subtype, leading to runtime errors. The switch expression also deconstructs records in the same line.
- **Interview follow-up:** What performance characteristics does pattern matching have compared to manual instanceof checks?

**Q: You have a `Pair` record and want to match only pairs where both elements are the same type and equal. How would you express this?**

- Use a nested guarded pattern: `if (obj instanceof Pair(String a, String b) && a.equals(b))`. The guard `&& a.equals(b)` ensures both strings are equal. For generic `Pair(var a, var b)` you cannot use `a.equals(b)` because types are erased.
- **Interview follow-up:** How would you match a `Pair` where both elements are non-null strings without using a guard?

**Q: You are building a validation framework that checks input objects for specific patterns. How would pattern matching help simplify the validation logic?**

- Use record patterns to destructure inputs in validation rules. Each rule is a pattern: `case User(var name, var email) when name != null && email != null -> valid`. Multiple patterns can be combined in a switch, making validation declarative.
- **Interview follow-up:** How would you compose multiple patterns for complex validation rules?

**Q: A method needs to return different result types based on input, similar to a discriminated union. How would pattern matching in switch help?**

- Define a sealed `Result` type with `Success<T>` and `Failure` records. Pattern matching in switch deconstructs each variant: `case Success(var value) -> value`, `case Failure(var error) -> throw error`. The compiler ensures all variants are handled.
- **Interview follow-up:** How would you add a `Loading` variant without breaking existing code?

## Interview Questions

- **What is the scope of a pattern variable bound in an instanceof expression?**
  - The pattern variable is in scope where the compiler can prove the pattern matched. For `if (x instanceof String s)`, `s` is in scope inside the if block and any subsequent `&&` conditions. It is not available after the if-else chain unless the compiler can prove all branches assign it.

- **How do record patterns interact with generic record types?**
  - Record patterns use type inference. If you have `record Box<T>(T value) { }`, you can write `if (obj instanceof Box(var v))` to infer the type of `v` from the component type at runtime. However, generics are erased, so the pattern matches only the raw record type; the component type is unchecked.

- **What is a guarded pattern and how does it differ from a regular type pattern with an if condition?**
  - A guarded pattern combines the type check and the condition in a single pattern: `case String s && s.length() > 5`. In a switch, this avoids nesting a second condition inside the case body. The guard is part of the pattern, so a failed guard moves to the next case rather than falling through.

- **Can you use pattern matching with arrays?**
  - No. Array types are not records and do not support deconstruction patterns. You must manually access array elements.

- **What is the difference between a type pattern and a record pattern?**
  - A type pattern (`String s`) checks the type and binds a variable. A record pattern (`Point(int x, int y)`) additionally deconstructs the record into its components. Record patterns can be nested.

- **Can pattern variables be reassigned?**
  - Pattern variables are effectively final within their scope. They cannot be reassigned after being bound by the pattern.

- **How does the compiler handle dominance in pattern matching switch?**
  - A pattern dominates another if it is more general (e.g., `Object o` dominates `String s`). The compiler reports an error if a case is dominated by a preceding case that would always match first.

- **What is the scope of a pattern variable in a switch case?**
  - The pattern variable is in scope within the case block. It is not visible in other cases. For arrow syntax, it is in scope in the expression or block after `->`.

- **Can you use pattern matching with var in record patterns?**
  - Yes. `var` in a record pattern infers the type of the component: `if (obj instanceof Pair(var left, var right))`. The inferred type is the component type from the record declaration.

- **How do guarded patterns affect pattern matching dominance?**
  - A guarded pattern does not dominate the same unguarded pattern. For example, `case String s && s.length() > 0` does not dominate `case String s`. The compiler allows both, and the guarded case must come first.

- **Can you use pattern matching in a traditional switch statement (not expression)?**
  - Yes, pattern matching works in both switch statements and switch expressions starting from JDK 21. The same patterns can be used in both constructs.

- **What happens if a record pattern component does not match the runtime type?**
  - The pattern does not match, and the next case or else branch is evaluated. Record patterns include an implicit type check for each component that is a type pattern.

- **Can you use pattern matching with enums?**
  - Yes, but enums typically use constant cases directly. Pattern matching with enums is useful when enum constants carry state (via fields or methods).

- **How do you handle null in record patterns?**
  - If the top-level value is null, no record pattern matches. You need a `case null` in a switch or a separate null check for instanceof. Nested null in record components depends on whether the component type allows null.

- **What is the difference between `case null` and `default` in a pattern matching switch?**
  - `case null` matches only null values. `default` matches any value not matched by previous cases, excluding null (unless no `case null` exists, in which case the switch throws NPE before reaching default).

- **Can you use pattern matching in a lambda or method reference?**
  - Not directly. Pattern matching is a statement/expression construct and cannot be used inside lambdas that expect functional interfaces. You must use a full switch expression as the lambda body.

- **How does type inference work in nested record patterns with generics?**
  - Due to erasure, nested generic record components are unchecked. The pattern matches the raw type and the component type is inferred from the erasure. Var patterns help avoid unchecked warnings.

- **What is a sealed type's role in exhaustiveness checking for pattern matching?**
  - The compiler uses the `permits` clause to determine the complete set of subtypes. If the switch covers all permitted subtypes, no default is needed. Missing a subtype causes a compile error.

- **Can you use pattern matching in a catch clause?**
  - No. Catch clauses use exception types directly. Pattern matching is not supported in catch clauses as of JDK 21.

- **How does the compiler determine which pattern in a switch matches a given value?**
  - The compiler evaluates patterns in declaration order. The first pattern that matches the selector value is executed. Dominance rules prevent a more general pattern from appearing before a more specific one.

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
