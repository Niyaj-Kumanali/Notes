# Sealed Classes

- Purpose: restrict which classes or interfaces may extend or implement a given type
- Declared with sealed keyword and permits clause: sealed class Shape permits Circle, Square {}
- Permitted subclasses must be in the same module or the same named package
- Each permitted subclass must be declared final, sealed, or non-sealed
- Final subclass closes the hierarchy branch further
- Sealed subclass continues the restriction to its own permitted types
- Non-sealed subclass opens the hierarchy to unknown subtypes
- Sealed interfaces follow the same rules as sealed classes
- Pattern matching exhaustiveness: when all sealed subtypes are covered in switch, no default needed
- Compiler verifies that no subtype is missed
- Migration: existing non-sealed hierarchies can be refactored to sealed if subtypes are known
- JDK 17 preview, finalized in JDK 17
- Use cases: domain modeling with fixed variants, algebraic data types, state machines with known states
- Enables safer API design by controlling extensibility
