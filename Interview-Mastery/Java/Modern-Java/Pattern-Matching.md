# Pattern Matching

- Pattern matching for instanceof introduced in JDK 16 as final
- Type pattern: if (obj instanceof String s) binds variable s in scope of the if block
- Guarded patterns combine pattern with boolean condition using &&: if (obj instanceof String s && s.length() > 5)
- Record patterns introduced in JDK 19 preview, finalized in JDK 21
- Deconstruction: if (obj instanceof Point(int x, int y)) extracts components directly
- Nested patterns: if (obj instanceof Rectangle(Point(int x, int y), var size))
- Switch expression pattern matching with sealed classes for exhaustive matching
- Exhaustiveness checked at compile time: every permitted subtype must have a case
- Null handling: null does not match any pattern by default; an explicit null case is needed
- Var keyword in patterns matches any type: case var x -> handles all remaining cases
- Improved readability by removing casts and nested conditions
