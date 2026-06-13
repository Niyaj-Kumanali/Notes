# Switch Expressions

- Arrow syntax case -> is concise and does not fall through
- Colon syntax case : behaves like old switch with fall-through and requires break
- Switch as expression assigns a value to a variable: int result = switch (x) { ... }
- Yield keyword returns a value from a case block when colon syntax is used
- Break returns a value in switch expressions only when used as a labeled break (uncommon)
- Arrow cases yield implicitly (no yield needed for single expression; yield needed for blocks)
- Exhaustive: switch expression must cover all possible input values
- Default case covers unhandled values if the type is not sealed
- Null case: case null -> handles null directly (JDK 17 preview, JDK 21 final)
- Pattern matching in switch: case String s -> or case Point(int x, int y) ->
- When expressions are cleaner than nested if-else chains for type-based dispatch
- Breaking changes: old switch-with-break does not return a value; new switch expression always does
