# Java Core Questions

## Questions

1. What are the main features of Java?
2. What is platform independence in Java?
3. What is JVM, JRE, and JDK?
4. What is bytecode?
5. What is the difference between stack and heap memory?
6. What is a class?
7. What is an object?
8. What are constructors?
9. What is constructor overloading?
10. What is method overloading?
11. What is method overriding?
12. What is inheritance?
13. What is encapsulation?
14. What is abstraction?
15. What is polymorphism?
16. What is the difference between compile-time and runtime polymorphism?
17. What is an interface?
18. What is an abstract class?
19. Difference between abstract class and interface.
20. Can an interface have default methods?
21. Can an interface have static methods?
22. What is the diamond problem?
23. What is the difference between `==` and `.equals()`?
24. What is the contract between `equals()` and `hashCode()`?
25. What happens if `hashCode()` is not overridden?
26. What is immutable class?
27. How do you create an immutable class?
28. Why is `String` immutable?
29. Difference between `String`, `StringBuilder`, and `StringBuffer`.
30. What is the String constant pool?
31. What is exception handling?
32. Difference between checked and unchecked exceptions.
33. Difference between `throw` and `throws`.
34. Difference between `final`, `finally`, and `finalize`.
35. What is try-with-resources?
36. What is a custom exception?
37. What are access modifiers in Java?
38. What is static keyword?
39. What is final keyword?
40. What is transient keyword?
41. What is volatile keyword?
42. What is serialization?
43. What is marker interface?
44. What is cloning?
45. What is shallow copy and deep copy?
46. What are wrapper classes?
47. What is autoboxing and unboxing?
48. What are annotations?
49. What is reflection?
50. What are generics?
51. What is type erasure?
52. What is varargs?
53. What is enum?
54. What is garbage collection?
55. What are GC roots?
56. What is memory leak in Java?
57. How can memory leaks happen in Java?
58. What are strong, weak, soft, and phantom references?
59. What is classloader?
60. What are Java records?

---

## Answers

1. What are the main features of Java?
   - **Answer:**
      - Java is platform-independent, object-oriented, has automatic memory management, strong typing, and built-in multithreading
      - OOP principles are relied upon for modular service/controller layers, and garbage collection means manual memory management is rarely a concern in Spring Boot applications
      - Platform independence works because Java source compiles to bytecode (.class files), which the JVM interprets or JIT-compiles on any OS — write once, run anywhere
      - The classloader subsystem loads .class files into memory using a parent-delegation model: bootstrap classloader loads core JDK classes, extension classloader loads endorsed libs, and application classloader loads application classes from the classpath
      - Java's memory model differs from C++ in that there is no manual malloc/free — objects are heap-allocated and garbage-collected, with no pointer arithmetic and bounds-checked arrays
      - Try-with-resources (introduced in Java 7) ensures AutoCloseable resources like JDBC connections or file handles are closed in a finally block automatically, reducing boilerplate and preventing resource leaks
2. What is platform independence in Java?
   - **Answer:**
      - Java achieves platform independence through bytecode that runs on the JVM
      - The same Spring Boot JAR can be deployed on Windows dev machines and Linux servers without recompiling, which is critical for CI/CD pipelines
      - The compilation process goes: .java source → javac compiler → .class bytecode file → JVM interprets/JIT-compiles on the target OS. Each platform has its own JVM implementation that translates the same bytecode to native instructions
      - Portability challenges include file path separators (`/` vs `\`), line endings (`\n` vs `\r\n`), character encodings (UTF-8 vs platform default), and timezone/locale differences that can surface in cross-platform deployments
3. What is JVM, JRE, and JDK?
   - **Answer:**
      - JVM executes bytecode, JRE includes JVM plus core libraries, and JDK adds development tools like javac and javap
      - Typically, only JDK is installed on dev machines while JRE-based Docker images are used for production
      - JVM architecture includes the class loader subsystem (loads .class files), runtime data areas (method area, heap, stack, PC registers, native method stacks), and execution engine (interpreter, JIT compiler, garbage collector)
      - JIT compilation converts frequently executed bytecode (hot spots) into native machine code at runtime, caching compiled code in the code cache for faster subsequent execution
      - Choosing the right JVM implementation (HotSpot, OpenJ9, GraalVM) matters for server performance — GraalVM can ahead-of-time compile to native images for faster startup, while HotSpot's mature JIT produces better peak throughput
4. What is bytecode?
   - **Answer:**
      - Bytecode is the intermediate representation of Java source code, stored in .class files and executed by the JVM
      - While bytecode is not typically worked with directly, understanding it helps debug classpath issues when compiled classes are stale
      - javac produces bytecode by parsing source code into an AST, performing semantic analysis, type checking, and then emitting stack-based bytecode instructions into .class files
      - The JVM verifies bytecode before execution — it checks magic numbers, version compatibility, constant pool validity, stack integrity, and type safety to prevent malicious or malformed code from executing
      - Tools like javap can disassemble .class files to inspect bytecode, which is useful for understanding generic type erasure (type parameters replaced with Object or bounds), lambda desugaring (lambdas become private methods invoked via invokedynamic), and compiler optimizations
5. What is the difference between stack and heap memory?
   - **Answer:**
      - Stack stores primitive values and object references per thread, while heap stores all actual objects and is shared across threads
      - Large collections of records live on the heap while local loop variables stay on the stack
      - Each method call creates a stack frame containing local variables, operand stack, and frame data — when the method returns, the frame is popped. Deep or infinite recursion exhausts stack space and throws StackOverflowError
      - The heap is divided into generations: young generation (Eden + Survivor spaces for newly allocated objects) and old generation (tenured space for long-lived objects). Minor GC collects the young generation frequently; major GC collects the old generation less often but with longer pause times
      - JVM heap settings are tuned with -Xms (initial heap), -Xmx (max heap), -XX:NewRatio (young/old ratio), and -XX:MaxTenuringThreshold to avoid OutOfMemoryError during bulk data processing
6. What is a class?
   - **Answer:**
      - A class is a blueprint for creating objects in Java
      - In backend code, every entity like `Partner`, `SensorReading`, or `InventoryRecord` is a class with fields and methods
      - Class members include fields (state), methods (behavior), constructors (initialization), and nested types. Static members belong to the class itself; instance members belong to each object created from the class
      - A class is loaded into memory by the classloader: the JVM reads the .class file, allocates memory in the method area for class metadata, and creates a java.lang.Class object on the heap as the runtime representation
      - In a layered Spring Boot architecture, classes are structured by responsibility — entity classes for data, repository interfaces for persistence, service classes for business logic, and controller classes for HTTP endpoints — keeping each class focused on a single concern
7. What is an object?
   - **Answer:**
      - An object is a runtime instance of a class with its own state
      - When calling `repository.findById(id)`, the returned entity object has specific field values representing a real database row
      - Object creation uses the `new` keyword: memory is allocated on the heap, the constructor initializes the object's fields, and a reference to the object is returned. The object persists until no references point to it, at which point the garbage collector reclaims its memory
      - Object lifecycle: class loading → constructor execution → field initialization → use (method calls) → becomes eligible for GC when unreachable → garbage collector reclaims heap memory
8. What are constructors?
   - **Answer:**
      - Constructors initialize object state when an instance is created
      - In DTOs and entities, constructors are used to set required fields so the object is never in an invalid state
      - A default no-arg constructor is provided by Java only if no other constructor is defined. Parameterized constructors enforce required values at creation time. Constructor chaining with `this()` delegates to another constructor in the same class, and the `this()` call must be the first statement
      - Constructor injection is preferred over field injection in Spring services because it makes dependencies explicit, enables immutability with final fields, and makes unit testing straightforward without requiring the Spring context
9. What is constructor overloading?
   - **Answer:**
      - Constructor overloading allows defining multiple constructors with different parameters for flexible object creation
      - For example, an `Alert` class could have overloaded constructors to accept either sensor ID alone or full sensor reading data
      - Java differentiates overloaded constructors by parameter list (number, types, and order of parameters) — not by return type or parameter names. The compiler matches the `new` call to the constructor with the closest matching parameter signature
      - The `this()` call within a constructor must be the first statement because it chains to another constructor, and the object must be fully initialized before any other logic executes
      - Builder patterns are often preferred over constructor overloading for classes with many optional parameters, as they provide named parameters and avoid confusing constructor signatures with many similar types
10. What is method overloading?
    - **Answer:**
       - Method overloading means multiple methods share the same name but differ in parameters
       - It is commonly used in service layers for lookup methods with different filter criteria, like `findById()` and `findByIdAndDate()`
       - This is compile-time polymorphism — the compiler resolves which method to call based on the argument types at compile time, binding the call statically
       - Method resolution follows JLS rules: exact match first, then widening, then varargs. Return type alone cannot distinguish overloaded methods — two methods with the same name and parameters but different return types cause a compile error
       - Autoboxing complicates overload resolution: calling `method(int)` vs `method(Integer)` are distinct overloads, and Java may auto-box or unbox implicitly, sometimes choosing an unexpected overload or failing to compile when ambiguous
11. What is method overriding?
    - **Answer:**
       - Method overriding allows a subclass to provide a specific implementation of a parent class method
       - `equals()` and `hashCode()` are commonly overridden in entity classes to ensure correct behavior in collections and JPA identity management
       - This is runtime polymorphism — the JVM determines which method to call at runtime based on the actual object type, not the reference type, using vtable dispatch
       - The `@Override` annotation is not required but catches mistakes at compile time (wrong signature, typos). Covariant return types allow overriding methods to return a subclass of the parent's return type
       - Overridden methods cannot be more restrictive in access — if the parent method is `public`, the override cannot be `protected` or `private`. Spring AOP uses proxy-based overriding where a proxy wraps the bean and intercepts method calls
12. What is inheritance?
    - **Answer:**
       - Inheritance lets a class acquire fields and methods from a parent class
       - For example, a base `AbstractReportService` can be extended to share common reporting logic across multiple report types
       - Java supports single inheritance with classes (one `extends` only) but multiple inheritance of type through interfaces (a class can implement many interfaces). The `extends` keyword establishes the parent-child relationship, and inherited members (except constructors and private members) are accessible in the subclass
       - Method overriding via inheritance is how subclasses customize behavior — the subclass inherits the method signature but provides its own implementation
       - The diamond problem arises when a class inherits conflicting default methods from multiple interfaces; the implementing class must explicitly override the conflicting method. Composition is generally preferred over inheritance for maintainability because inheritance creates tight coupling between parent and child
13. What is encapsulation?
    - **Answer:**
       - Encapsulation bundles data and methods together while hiding internal state via access modifiers
       - In Spring Boot services, entity fields are kept private with behavior exposed through public methods, preventing direct field manipulation
       - Getters and setters control access, but the principle of information hiding means exposing only what's necessary — a getter that returns a mutable internal collection breaks encapsulation because external code can modify internal state through the returned reference
       - Encapsulation supports maintainability by allowing internal implementation changes without breaking external callers. If how a field is stored or computed changes, callers using the getter are unaffected as long as the contract is preserved
       - In entity classes, defensive copies are returned from getters for mutable fields (like List or Date) and inputs are validated in setters to enforce invariants
14. What is abstraction?
    - **Answer:**
       - Abstraction hides complex implementation details and shows only essential features
       - Service interfaces in Spring Boot allow controllers to depend on contracts, not concrete implementations
       - Abstract classes vs interfaces: abstract classes can provide partial implementation and hold state; interfaces define pure contracts (with Java 8 default methods adding limited implementation). Both achieve abstraction but serve different design purposes
       - Abstraction reduces coupling — when a `ReportService` is defined as an interface and `PdfReportService` is injected, the controller knows nothing about PDF generation details, making it easy to swap implementations or add new ones
       - Spring's dependency injection supports programming to interfaces by resolving implementations at runtime, which is the foundation of the layered architecture pattern
15. What is polymorphism?
    - **Answer:**
       - Polymorphism allows objects to take multiple forms, behaving differently based on their actual type
       - A `NotificationSender` interface could have `EmailSender` and `SmsSender` implementations triggered based on alert severity
       - Compile-time polymorphism (method overloading) resolves at compile time based on parameter signatures. Runtime polymorphism (method overriding) resolves at runtime based on the actual object type using vtable dispatch — the JVM looks up the method in the object's actual class hierarchy
       - The JVM maintains a virtual method table (vtable) per class that maps method signatures to implementations; at runtime, the call is dispatched through the vtable based on the receiver's actual type
       - Spring leverages polymorphism for dependency injection: when a field of type `NotificationSender` is declared, Spring resolves the concrete bean at runtime, enabling loose coupling between components
16. What is the difference between compile-time and runtime polymorphism?
    - **Answer:**
       - Compile-time polymorphism is resolved during compilation through method overloading, while runtime polymorphism is resolved at runtime through method overriding
       - Overloading is used for convenience methods and overriding for interface implementations
       - javac resolves overloaded methods by matching argument types against available signatures, preferring exact matches, then widening conversions, then autoboxing, then varargs — this is all static resolution
       - For overridden methods, the JVM uses dynamic dispatch: it looks up the actual class of the receiver object and finds the method implementation in that class or its parents. This has a small performance cost compared to static dispatch, but JIT compilers devirtualize hot calls when the type is monomorphic
       - Both forms commonly appear together in Spring Boot code — for example, a service may overload `save()` with different parameter lists (compile-time) while also overriding a base class `save()` method (runtime)
17. What is an interface?
    - **Answer:**
       - An interface is a contract that defines method signatures without implementation
       - Repository and service interfaces are commonly used to decouple layers and enable Spring to inject appropriate implementations
       - Since Java 8, interfaces can have default methods (with a body, providing backward-compatible method additions) and static methods (utility methods not inherited by implementors)
       - Functional interfaces (single abstract method) enable lambda expressions — `@FunctionalInterface` is a marker annotation that enforces the single-method constraint. Marker interfaces like `Serializable` have no methods but signal special behavior to the JVM or frameworks
       - Interfaces support multiple inheritance of type: a class can implement many interfaces, allowing it to be polymorphically assigned to any of those types. Interfaces are preferred over abstract classes when defining contracts across unrelated class hierarchies
18. What is an abstract class?
    - **Answer:**
       - An abstract class cannot be instantiated and may contain both abstract and concrete methods
       - An abstract `BaseValidationService` with common validation logic can leave specific rules to subclasses
       - Abstract methods have no body and must be implemented by concrete subclasses. A class with even one abstract method must be declared abstract. Abstract classes can have constructors even though they cannot be instantiated — constructors run when a subclass calls `super()`
       - Access modifiers on abstract methods can be more restrictive in subclasses but not less. Unlike interfaces, abstract classes can have instance fields (state), multiple constructors, and static methods
       - Abstract classes are preferred when related classes share significant common state and behavior, while interfaces are used when defining contracts across unrelated hierarchies or when multiple inheritance of type is needed
19. Difference between abstract class and interface.
    - **Answer:**
       - Abstract classes can have state and constructors, while interfaces support only constants and abstract/default methods
       - Abstract classes are used for shared logic across related classes and interfaces for defining contracts across unrelated classes
       - Java 8 blurred the line by adding default and static methods to interfaces, but key differences remain: interfaces cannot have instance fields (only `public static final` constants), cannot have constructors, and a class can implement multiple interfaces but extend only one abstract class
       - When to prefer one over the other: use interfaces for defining capability contracts (like `Comparable`, `Serializable`), use abstract classes for template method patterns where base class controls the algorithm skeleton and subclasses fill in steps
       - In Spring, `JpaRepository` is an interface because it defines a persistence contract any class can implement, while `AbstractPaginationHelper` is an abstract class because it provides shared pagination logic that concrete helpers extend
20. Can an interface have default methods?
    - **Answer:**
       - Yes, interfaces can have default methods with a body introduced in Java 8
       - This is useful when creating a custom functional interface for data transformation, providing a default no-op implementation so implementers only override what they need
       - Default methods can cause the diamond problem when a class implements two interfaces with the same default method signature. Java resolves this by requiring the implementing class to explicitly override the conflicting method
       - Conflict resolution: if class A extends B and implements C, and both B and C have a default method `foo()`, the class must override `foo()`. If only one interface provides a default, that default wins. If both interfaces provide defaults and there is no class hierarchy, the compiler forces an explicit override
       - Default methods were introduced primarily for backward compatibility — they allowed the Java 8 Streams API to add methods to existing collection interfaces without breaking billions of lines of existing code. Their limitation is that they cannot access instance fields of the implementing class
21. Can an interface have static methods?
    - **Answer:**
       - Yes, interfaces can have static methods with a body since Java 8
       - Static helper methods in interfaces are useful for utility functions like `ValidationUtils.isValidSerial()` that belong conceptually to a specific domain
       - Interface static methods are not inherited by implementing classes — they must be called via `InterfaceName.method()`, not through an implementing class instance. This differs from class static methods which are inherited
       - Use cases include factory methods (e.g., `List.of()` creates immutable lists), utility methods grouped by domain, and companion objects for interfaces
       - Java 9 added private methods in interfaces, allowing default methods to share internal helper logic without exposing it publicly. This was a further step toward making interfaces capable of encapsulating implementation details
22. What is the diamond problem?
    - **Answer:**
       - The diamond problem occurs when a class inherits from multiple sources with conflicting default method implementations
       - Java avoids it with classes using single inheritance, and for interfaces, the implementing class must override the conflicting method
       - Java 8's default methods reintroduced this risk because a class implementing two interfaces with the same default method has no automatic way to choose one implementation
       - Explicit override resolves it: the implementing class provides its own implementation, or chooses one via `InterfaceA.super.method()`. The compiler enforces this — ambiguity is a compile error, not a runtime error
       - In C++, multiple inheritance allows a class to inherit from multiple base classes directly, and virtual inheritance mitigates the diamond problem at the language level. Java's approach is simpler: single class inheritance plus explicit interface default resolution. A real scenario is implementing both `MouseListener` and `KeyListener` which both define `keyPressed()` — the override must be explicit
23. What is the difference between `==` and `.equals()`?
    - **Answer:**
       - `==` compares object references (memory addresses), while `.equals()` compares logical content
       - `.equals()` should be used on Strings to check logical equality, never `==`
       - The default `equals()` behavior from `Object` mimics `==` — it compares memory addresses. To compare logical content, classes must override `equals()`. String, Integer, and other wrapper classes override it to compare values
       - String interning affects `==`: two String literals with the same value share the same reference in the String pool, so `==` returns true. But `new String("hello")` creates a new object on the heap, so `==` returns false even for identical content
       - Best practice: always use `.equals()` for object comparison. Wrapper class comparisons like `Integer a = 200; Integer b = 200;` fail with `==` because values outside -128 to 127 are not cached, creating separate objects
24. What is the contract between `equals()` and `hashCode()`?
    - **Answer:**
       - If two objects are equal via `equals()`, they must have the same hash code
       - Both should be overridden in entity classes so that HashSets and HashMaps work correctly for deduplication and lookup operations
       - The full contract: if `a.equals(b)` is true, then `a.hashCode() == b.hashCode()` must be true. The reverse is not required — unequal objects CAN share hash codes (hash collisions). Hash-based collections use hashCode to locate the bucket, then equals to find the exact entry
       - Breaking the contract causes silent failures: putting an object in a HashSet, then modifying it, may cause the HashSet to never find it again because it's in the wrong bucket. This is why mutable objects as HashMap keys are dangerous
       - Common implementation uses `Objects.hash()` for hashCode and `Objects.equals()` for equals. Lombok's `@EqualsAndHashCode` generates both from specified fields, reducing boilerplate in entity classes
25. What happens if `hashCode()` is not overridden?
    - **Answer:**
       - Without overriding `hashCode()`, the default Object implementation uses memory address
       - Two logically equal objects would have different hash codes, causing issues in HashSets and HashMaps like duplicates or lookup failures
       - HashMap internally works by computing `hashCode()`, mapping to a bucket (array index via modulo), and storing key-value entries in that bucket's linked list or tree. When looking up, it computes the hash to find the bucket, then calls `equals()` on entries in that bucket
       - If hashCode is not overridden, two equal objects land in different buckets, so HashMap treats them as distinct keys. This causes memory leaks from duplicate entries — the same logical key is put twice and both remain, with no way to retrieve or remove the first
       - Debugging such issues in production requires heap dumps (Eclipse MAT, VisualVM) to find unexpected duplicate keys. IDE-generated hash code implementations use prime multipliers (e.g., 31 * result + field.hashCode()) to distribute hash values well across the bucket array
26. What is immutable class?
    - **Answer:**
       - An immutable class cannot be modified after creation
       - Immutable DTOs for sensor readings, for example, ensure thread-safe data transfer across Kafka consumer threads without synchronization
       - Rules for immutability: class must be final (or all fields final), all fields must be private and final, initialized via constructor, no setters, return defensive copies for mutable field getters, and prevent subclasses from overriding methods
       - Benefits: thread safety (no synchronization needed), caching (can be freely shared and interned), safe hash codes (computed once, never change), and security (no tampering of String class names or DB URLs)
       - Immutability can hurt performance when objects are large and need frequent updates — creating a new copy for each modification generates garbage. Builder patterns or records mitigate this by providing efficient construction
27. How do you create an immutable class?
    - **Answer:**
       - Declare the class final, make all fields private final, initialize them through the constructor, provide only getters without setters, and return defensive copies for mutable fields
       - This approach is commonly used for value objects in domain-driven design
       - Builder pattern alternative: for classes with many fields, a builder separates construction from representation. The builder collects values via setter-like methods and constructs the immutable object when `build()` is called, avoiding constructors with dozens of parameters
       - Java 14+ records simplify immutability: `record Point(int x, int y)` automatically generates a final class with private final fields, constructor, getters, equals, hashCode, and toString. Records are the idiomatic choice for simple immutable data carriers
       - `Collections.unmodifiableList()` provides a read-only view but is not full immutability — the underlying list can still be modified through other references. True immutability requires an unmodifiable collection wrapping a private copy. Serialization concerns: deserialization creates a new object, so `readResolve()` or `readObject()` should prevent invalid state
28. Why is `String` immutable?
    - **Answer:**
       - String immutability enables caching (String pool), security (no tampering of class names or DB URLs), thread safety, and efficient hash code caching
       - String immutability is relied upon when using Strings as HashMap keys
       - The String pool mechanism: String literals are interned in a shared pool on the heap. Multiple references to the same literal point to one object, saving memory. Immutability guarantees that sharing is safe — no thread can modify a shared String
       - Pre-Java 7, substrings shared the internal char[] array (offset + count), which could cause memory leaks when a small substring kept a large char[] alive. Java 7u6 changed String to use a byte array with coder field, and substrings now copy the data
       - StringBuilder is needed for concatenation in loops because each `+` operation creates a new String object, generating excessive garbage. StringBuilder modifies an internal buffer in place. Reflection can technically break immutability via `setAccessible(true)` on the internal field, but this is a security violation
29. Difference between `String`, `StringBuilder`, and `StringBuffer`.
    - **Answer:**
       - String is immutable, StringBuilder is mutable and not thread-safe, StringBuffer is mutable and thread-safe
       - String is used for fixed values, StringBuilder for single-threaded concatenation in loops, and StringBuffer is rarely used in modern code
       - All three use an internal char[] (String uses byte[] with compact strings since Java 9). Capacity management: StringBuilder and StringBuffer start with a default capacity (16) and grow by doubling + 2 when full. String has no capacity concept since it's immutable
       - Performance benchmarks show StringBuilder is significantly faster than StringBuffer because StringBuffer synchronizes every method call. In single-threaded contexts, the synchronization overhead is pure waste
       - javac optimizes simple `+` concatenation by converting it into StringBuilder.append() calls. For example, `"a" + b + c` becomes `new StringBuilder().append("a").append(b).append(c).toString()`. However, this compiler optimization does not apply inside loops — it creates a new StringBuilder per iteration
30. What is the String constant pool?
    - **Answer:**
       - The String constant pool is a special heap region that caches String literals to save memory
       - When `String s = "partner"` is written in multiple places, all references point to the same pooled object
       - `String.intern()` explicitly adds a String to the pool and returns the pooled reference. `"hello" == "hello".intern()` is always true. But using `new String("hello")` creates a heap object outside the pool — only literals are automatically pooled
       - Before Java 7, the pool lived in PermGen (fixed size, caused OOM). Java 7 moved it to the regular heap, allowing it to grow dynamically. Java 8 removed PermGen entirely (replaced by Metaspace)
       - G1 GC supports String deduplication (-XX:+UseStringDeduplication), which automatically merges equal Strings in the heap even if they are not in the pool. For large datasets with repeated Strings (like serial numbers), this reduces memory footprint significantly
31. What is exception handling?
    - **Answer:**
       - Exception handling uses try-catch-finally to manage runtime errors gracefully
       - In Spring Boot APIs, global exception handlers with `@ControllerAdvice` return consistent error responses instead of stack traces
       - The exception hierarchy: Throwable is the root, with Exception (checked, recoverable) and Error (unchecked, JVM problems like OutOfMemoryError) as subclasses. RuntimeException extends Exception and covers unchecked programming bugs
       - Checked exceptions must be caught or declared in the method signature with `throws`. Unchecked exceptions (RuntimeException and its subclasses) are not required to be caught. Try-with-resources (Java 7) ensures AutoCloseable resources are closed even if an exception occurs, replacing verbose finally blocks
       - Custom exceptions are commonly defined for business logic failures (like `InvalidSerialException`, `SensorDataNotFoundException`) so that controllers and global exception handlers can distinguish business errors from system errors and return appropriate HTTP status codes
32. Difference between checked and unchecked exceptions.
    - **Answer:**
       - Checked exceptions are checked at compile time and must be handled or declared; unchecked exceptions (RuntimeException) are not
       - Checked exceptions are used for recoverable conditions like file not found, and unchecked for programming bugs like null pointer
       - Use each type appropriately: checked exceptions represent conditions a well-written application should anticipate and recover from (IOException, SQLException). Unchecked exceptions represent programming errors that should be fixed, not caught (NullPointerException, IllegalArgumentException)
       - Best practice in Spring Boot: prefer unchecked exceptions for transactional methods because only unchecked exceptions trigger automatic rollback by default. Checked exceptions do not trigger rollback unless explicitly configured with `@Transactional(rollbackFor = Exception.class)`
       - Spring wraps checked exceptions in unchecked DataAccessException (a RuntimeException hierarchy) so that repository code doesn't leak checked exceptions to service layers. Custom exception design: extend RuntimeException for business logic errors and define meaningful exception hierarchies
33. Difference between `throw` and `throws`.
    - **Answer:**
       - `throw` actually throws an exception instance, while `throws` declares that a method might throw certain checked exceptions
       - Custom exceptions are thrown from service methods with `throw`, and `throws` is declared in method signatures
       - Exception propagation: when `throw` is executed, the JVM searches up the call stack for a matching catch block. If none is found, the thread terminates with an unhandled exception. The `throws` declaration tells the compiler (and callers) that a method may throw checked exceptions that must be handled upstream
       - `throws` with overriding: an overriding method cannot declare new checked exceptions, but can declare narrower, same, or no checked exceptions. It can declare unchecked exceptions freely. This ensures subclass methods don't surprise callers with unexpected checked exceptions
       - Chained exceptions wrap a cause: `throw new ServiceException("msg", cause)` preserves the original exception. Spring's declarative transaction management handles rollback for runtime exceptions thrown from @Transactional methods — checked exceptions require explicit rollbackFor configuration
34. Difference between `final`, `finally`, and `finalize`.
    - **Answer:**
       - `final` is a keyword for constants, non-overridable methods, and non-inheritable classes
       - `finally` is a try-catch block that always executes
       - `finalize()` is a deprecated GC callback
       - `final` is commonly used for constants and `finally` for resource cleanup
       - `finally` interaction with return: if both try and finally have return statements, the finally return wins. This is a common source of bugs. `finally` always executes even if an exception is thrown, caught, or the method returns — but it does NOT execute if the JVM exits (System.exit()) or the thread is killed
       - `finalize()` should never be relied upon for resource cleanup because: it's called at an unpredictable time by the GC, it's not guaranteed to be called at all, it can delay GC, and it can resurrect objects. Alternatives: `AutoCloseable` with try-with-resources for deterministic cleanup, `Cleaner` (Java 9+) for lightweight phantom-reference-based cleanup, and `PhantomReference` for post-mortem resource management
35. What is try-with-resources?
    - **Answer:**
       - Try-with-resources automatically closes resources that implement AutoCloseable
       - It is essential for JDBC connections and file I/O, ensuring resources are closed even if an exception occurs, without needing a finally block
       - Multi-resource syntax: `try (var r1 = ...; var r2 = ...)` declares multiple resources. They are closed in reverse declaration order (r2 first, then r1), ensuring correct cleanup order for dependent resources
       - Suppressed exceptions: if both the try block and the close() method throw exceptions, the close exception is added as a suppressed exception on the primary exception. This preserves all error information without losing the original cause
       - To make custom resources AutoCloseable, implement `close()` which is called automatically. In Spring Boot, JPA template methods like `JpaTemplate.execute()` handle connection management internally, but for raw JDBC or file operations, try-with-resources is essential
36. What is a custom exception?
    - **Answer:**
       - A custom exception is a user-defined class extending Exception or RuntimeException
       - For example, `InvalidSerialException` can be created to handle specific validation failures distinctly from generic system errors
       - Extend RuntimeException for unchecked exceptions (programming bugs, business logic errors that callers shouldn't be forced to catch) and Exception for checked exceptions (recoverable conditions callers must handle). Most Spring Boot applications use unchecked custom exceptions exclusively
       - Constructor best practice: always provide constructors for (message), (message, cause), (cause), and (message, cause, enableSuppression, writableStackTrace). The cause parameter chains exceptions for debugging. Custom fields like error codes or HTTP status codes provide structured error information
       - Integration with global exception handlers: `@ControllerAdvice` classes catch specific custom exceptions and return structured responses. Lombok's `@ResponseStatus` can annotate exceptions with HTTP status codes. Include `serialVersionUID` for custom exceptions that might be serialized across service boundaries
37. What are access modifiers in Java?
    - **Answer:**
       - Access modifiers control visibility: `private` (class only), `default` (package), `protected` (package + subclasses), `public` (everywhere)
       - In a layered architecture, fields are kept `private` and method visibility is decided based on which layer needs access
       - Access modifiers affect inheritance: `private` members are not inherited, `protected` members are accessible in subclasses even in different packages, and default (package-private) members are accessible within the same package only
       - Package-private (no modifier) is the most commonly forgotten modifier — it's useful for implementation classes that should only be visible within the package, like internal helper classes
       - Java 9's module system adds a layer of encapsulation beyond access modifiers: `exports` controls which packages are visible to other modules, and `open` allows reflection. Reflection can bypass access control via `setAccessible(true)`, but the module system restricts this for non-opened packages
38. What is static keyword?
    - **Answer:**
       - `static` means a member belongs to the class, not instances
       - Static constants are used for configuration keys, static utility methods for validation helpers, and static inner classes to group related types
       - Static initialization blocks run once when the class is loaded, used for complex static field initialization. They execute in order of declaration and cannot throw checked exceptions or reference instance members
       - Static methods cannot be overridden — they are hidden, not overridden, because they belong to the class, not the instance. Calling a static method on a subclass reference uses compile-time binding (the declared type's method), not runtime polymorphism
       - Static variables are stored in the method area (metaspace in Java 8+) and are shared across all instances. Thread safety concerns arise when static mutable state is accessed from multiple threads — synchronization or atomic classes are needed. Anti-patterns include static service classes in Spring (hinders testing, hides dependencies, breaks DI)
39. What is final keyword?
    - **Answer:**
       - `final` on a variable makes it a constant, on a method prevents overriding, and on a class prevents inheritance
       - Service dependencies are commonly marked as `final` for immutability, and `final` constants are used for magic strings in configuration
       - Blank final variables: final instance fields can be uninitialized at declaration but must be assigned exactly once in every constructor. Final local variables can be assigned once, often used in lambda expressions and anonymous classes
       - Final parameters are useful in anonymous classes and lambdas because they must be effectively final (not reassigned after initialization). This is why lambdas can only capture local variables that are not modified
       - JIT optimization: final methods can be devirtualized (inlined) by the JVM because they cannot be overridden, eliminating vtable lookup overhead. Final fields enable safe publication in multi-threaded contexts — the JVM guarantees that final fields are visible to all threads after construction without synchronization
40. What is transient keyword?
    - **Answer:**
       - `transient` marks fields that should not be serialized
       - Derived or cache fields are commonly marked as transient when entities are serialized for caching or cross-service communication
       - Java serialization mechanism: when `ObjectOutputStream.writeObject()` is called, it writes non-transient fields to the byte stream. Transient fields are skipped and initialized to their default values (null for objects, 0 for primitives) upon deserialization
       - Transient interacts with Externalizable (the alternative to Serializable): Externalizable gives full control over serialization via `writeExternal()` and `readExternal()`, so transient is less relevant when using Externalizable
       - Alternatives in Spring Boot: `@JsonIgnore` in Jackson excludes fields from JSON serialization without affecting Java serialization. `@JsonInclude(Include.NON_NULL)` conditionally excludes nulls. Security concern: never serialize sensitive data (passwords, tokens, connection strings) — mark them transient or exclude via custom serializers
41. What is volatile keyword?
    - **Answer:**
       - `volatile` guarantees visibility of changes to a variable across threads, preventing thread-local caching
       - Volatile flags are commonly used for graceful shutdown signals across threads
       - Happens-before guarantees: a write to a volatile variable happens-before every subsequent read of that same variable by any thread. This ensures visibility — when thread A writes to a volatile flag, thread B immediately sees the updated value, not a stale cached copy
       - Volatile does NOT provide atomicity: `volatile int count; count++` is not thread-safe because increment is read-modify-write. For atomic operations, use AtomicInteger, AtomicLong, or synchronized blocks. Volatile is suitable for single-writer scenarios (one thread writes, others read)
       - Common use cases: status flags (shutdown, ready), double-checked locking for lazy initialization (`volatile Singleton instance`), and version counters. Compared to synchronized: volatile is lighter (no blocking), but synchronized provides both visibility and atomicity. Volatile reads/writes have memory fence semantics that prevent CPU reordering
42. What is serialization?
    - **Answer:**
       - Serialization converts an object into a byte stream for storage or transmission
       - Sensor data objects, for example, are serialized when sent to Kafka topics and deserialized by consumer applications
       - Serializable interface: a class must implement `Serializable` (a marker interface) to be serialized. `serialVersionUID` is a version identifier — if the class structure changes without updating the UID, deserialization of old data fails with InvalidClassException
       - Custom serialization: `writeObject()` and `readObject()` methods control exactly what gets serialized and how. `defaultWriteObject()` and `defaultReadObject()` handle non-transient fields. `writeReplace()` and `readResolve()` can substitute objects during serialization (useful for singletons)
       - JSON serialization (Jackson in Spring Boot) is preferred over Java serialization for most use cases: it's human-readable, language-independent, and avoids the security risks of Java deserialization (remote code execution via crafted serialized objects). Spring Boot uses Jackson by default with `@RequestBody` and `@ResponseBody`
43. What is marker interface?
    - **Answer:**
       - A marker interface has no methods but signals special behavior to the JVM or framework
       - `Serializable` and `Cloneable` are classic examples
       - In Spring Boot, `@Configuration` annotations serve a similar signaling purpose
       - The JVM checks for `Serializable` during serialization — if the class doesn't implement it, `NotSerializableException` is thrown. Similarly, `Cloneable` allows `Object.clone()` to work; without it, `CloneNotSupportedException` is thrown
       - Marker interfaces are considered a pre-annotation design pattern. The shift towards annotations provides more flexibility: annotations can carry metadata (like `@SerialVersionUID` or `@SuppressWarnings`), can be applied to specific elements, and don't pollute the type hierarchy. However, marker interfaces still serve a purpose when the runtime needs to check type identity (e.g., `instanceof Serializable`)
       - Tradeoffs: marker interfaces enforce a type-level contract (objects implementing it are guaranteed to be serializable), while annotations are metadata that code must explicitly read and act upon
44. What is cloning?
    - **Answer:**
       - Cloning creates a copy of an object using the `clone()` method of Object
       - For example, baseline configuration objects can be cloned before applying entity-specific overrides to preserve the original
       - The Cloneable interface contract: if a class doesn't implement Cloneable, calling `clone()` throws `CloneNotSupportedException`. The `clone()` method is protected, so subclasses must override it and call `super.clone()` to get Object's bitwise copy behavior
       - Shallow vs deep copy: `Object.clone()` performs a shallow copy — primitive fields are copied, but object references point to the same nested objects. Deep copy requires recursively cloning nested objects or using serialization-based cloning
       - Why copy constructors or factory methods are preferred over Cloneable: Cloneable breaks the constructor contract, `clone()` is protected and awkward to use externally, and the Cloneable interface has no clone() method in its API (it just suppresses the exception). Copy constructors like `new Foo(original)` are clearer and more flexible. Serialization-based cloning (`deepCloneViaSerialization()`) handles complex object graphs but has performance overhead
45. What is shallow copy and deep copy?
    - **Answer:**
       - Shallow copy copies only the top-level object, sharing references to nested objects
       - Deep copy recursively duplicates all referenced objects
       - When copying entity data for reports, deep copy is needed to avoid modifying cached data through references
       - `Object.clone()` performs shallow copy: if an object has a `List<String>` field, the cloned object points to the same List. Modifying the list in one object affects the other. For primitive fields and immutable references (String, Integer), shallow copy is sufficient
       - Deep copy implementations: manually cloning each nested object (error-prone for complex graphs), serialization-based cloning (`.writeObject()` to a ByteArrayOutputStream then `.readObject()` — handles cycles but requires all objects to be Serializable), or using libraries like Apache Commons BeanUtils `BeanUtils.cloneBean()` and JSON round-trip (`new Gson().fromJson(new Gson().toJson(obj), Class.class)`)
       - Performance overhead of deep copy scales with object graph size and complexity. For simple DTOs, it's negligible. For deeply nested structures with cycles, it can be significant. Libraries like Apache Commons provide `SerializationUtils.clone()` for convenient deep copy via serialization
46. What are wrapper classes?
    - **Answer:**
       - Wrapper classes (Integer, Double, Boolean, etc.) box primitives into objects
       - They are used when collections require objects, like `Map<String, Integer>` for counting or caching
       - Autoboxing/unboxing: Java automatically converts between primitives and wrappers (int ↔ Integer). This is syntactic sugar — the compiler generates `Integer.valueOf()` for boxing and `.intValue()` for unboxing
       - Caching ranges: Integer caches -128 to 127, Byte caches all values, Character caches 0-127, Long/Short cache -128 to 127. Values outside these ranges create new objects, so `Integer.valueOf(200) == Integer.valueOf(200)` is false. `Integer a = 127; Integer b = 127; a == b` is true, but with 128 it's false
       - Performance overhead: boxing creates objects on the heap, causing GC pressure in tight loops. Prefer primitive streams (`IntStream`, `LongStream`) over wrapper streams for performance. Comparison gotcha: `Integer a = 128; Integer b = 128; a.equals(b)` is true, but `a == b` is false. OptionalInt vs Optional<Integer>: OptionalInt avoids boxing overhead for int values
47. What is autoboxing and unboxing?
    - **Answer:**
       - Autoboxing automatically converts primitives to wrapper objects, unboxing does the reverse
       - Java automatically converts when putting an `int` into a `List<Integer>` or using a wrapper in arithmetic
       - Compiler-generated code: `Integer x = 5;` compiles to `Integer x = Integer.valueOf(5);` (boxing). `int y = x;` compiles to `int y = x.intValue();` (unboxing). This happens transparently but has real performance implications
       - Performance cost in tight loops: each iteration creates a new wrapper object (outside cache range), generating garbage. In a loop of millions of iterations, this can cause noticeable GC pauses. Use primitives directly when possible
       - Null pointer risks: unboxing a null wrapper throws NullPointerException. `Integer x = null; int y = x;` crashes. This commonly happens with Optional or collection lookups returning null wrappers. To avoid pitfalls: check for null before unboxing, use Optional<Integer>, or prefer primitive streams (IntStream) for numeric operations
48. What are annotations?
    - **Answer:**
       - Annotations are metadata tags added to code elements
       - In Spring Boot, annotations like `@Service`, `@RestController`, `@Transactional`, and `@Cacheable` are heavily used to declaratively configure behavior without XML
       - Retention policies: SOURCE (discarded after compilation, like `@Override`), CLASS (in .class file but not available at runtime, like some framework annotations), RUNTIME (available via reflection, like Spring's `@Autowired`). Most Spring annotations are RUNTIME retention
       - Target types specify where annotations can be applied: `@Target(ElementType.TYPE)` for classes, `METHOD` for methods, `FIELD` for fields, `PARAMETER` for parameters, `CONSTRUCTOR` for constructors
       - Spring processes annotations via reflection at startup: component scanning reads `@Component`/`@Service`/`@Repository`, dependency injection reads `@Autowired`, AOP reads `@Transactional`/`@Cacheable`. Custom annotations for cross-cutting concerns: create a custom annotation + Aspect to implement logging, timing, or security checks. Meta-annotations: `@Service` is meta-annotated with `@Component`, so Spring treats both the same
49. What is reflection?
    - **Answer:**
       - Reflection allows inspecting and invoking classes, methods, and fields at runtime
       - Spring Boot uses reflection heavily for dependency injection, though direct use is rare — for example, a dynamic field-mapping utility
       - `Class.forName("com.example.MyClass")` loads a class by name and returns its Class object. `getMethod()` / `getDeclaredMethod()` retrieves methods. `Method.invoke()` calls methods dynamically. `Field.get()` / `Field.set()` reads/writes fields
       - Performance overhead: reflection is 10-50x slower than direct method calls because it bypasses JIT optimizations (inlining, devirtualization). Spring minimizes this cost by caching reflected metadata (BeanDefinition caches, MethodHandle caches) so reflection happens once at startup
       - Security restrictions: SecurityManager (deprecated in Java 17) could block reflective access. The module system (Java 9+) restricts reflective access to non-exported packages — `--add-opens` flags are needed for frameworks. Alternatives: method handles (Java 7+) provide near-direct-call performance with some reflection flexibility, and lambda metafactory (used by MethodHandles.Lookup) is faster for dynamic invocation
50. What are generics?
    - **Answer:**
       - Generics enable type-safe collections and classes by parameterizing types
       - For example, `List<SerialRecord>` ensures only SerialRecord objects are added, eliminating casting and catching type errors at compile time
       - Type parameters: `<T>` declares a type variable. Generic methods: `public <T> T getFirst(List<T> list)` infers T from the argument. Bounded wildcards: `? extends T` (upper bound, read-only) and `? super T` (lower bound, write-only) provide flexibility
       - The PECS principle (Producer Extends, Consumer Super) guides wildcard usage: when a generic type produces values for you to read, use `? extends T`. When it consumes values you provide, use `? super T`. For example, `copyAll(List<? super T> dest, List<? extends T> src)` — src produces, dest consumes
       - Generics improve code reusability in Spring's `JpaRepository<T, ID>` — each entity type gets a type-safe repository without casting. `@Repository public interface PartnerRepository extends JpaRepository<Partner, Long>` inherits all CRUD methods typed to Partner
51. What is type erasure?
    - **Answer:**
       - Type erasure removes generic type information at runtime, so `List<String>` and `List<Integer>` both become just `List`
       - This means generic types cannot be checked at runtime, which affects reflection-based field mapper designs
       - javac replaces type parameters with their upper bound (e.g., `<T extends Comparable>` becomes `Comparable`) or `Object` if unbounded. Bridge methods are generated to maintain polymorphism — when a subclass overrides a generic method, the bridge method delegates to the implementation with the erased signature
       - You cannot create `new T()` because T doesn't exist at runtime. Workarounds: pass `Class<T>` as a parameter and use `clazz.getDeclaredConstructor().newInstance()`, or use `TypeToken`/`TypeReference` (Guava) to capture generic types via anonymous class inheritance
       - Implications for serialization: Jackson loses generic type information during serialization due to type erasure. This is why `List<MyDto>` loses its element type during JSON parsing. Solutions: `TypeReference<List<MyDto>>` in Jackson, or annotating fields with `@JsonTypeInfo` for polymorphic types
52. What is varargs?
    - **Answer:**
       - Varargs allow methods to accept variable number of arguments using `...` syntax
       - Varargs are useful in validation frameworks to pass multiple error codes to a logging utility without overloading methods
       - Internal array creation: `method(String... args)` is compiled to `method(String[] args)` — the compiler creates a new array for each varargs call. This means `method("a", "b")` allocates `new String[]{"a", "b"}`
       - Varargs must be the last parameter in the method signature. A method can have at most one varargs parameter. If mixed with regular parameters, the varargs collects all remaining arguments into the array
       - Heap pollution warnings: when varargs are combined with generics, the compiler warns about potential heap pollution because the varargs array could be modified. Use `@SafeVarargs` on final/static methods to suppress the warning when you're sure the array won't be modified. Avoid varargs for clarity when the method takes a fixed, small number of parameters — named parameters or overloading is more readable
53. What is enum?
    - **Answer:**
       - Enums define a fixed set of named constants, and in Java they are full classes with fields and methods
       - For example, an `AlertSeverity` enum (LOW, MEDIUM, HIGH, CRITICAL) can have threshold values and action methods attached
       - Enum singleton pattern: each enum constant is a single instance. `AlertSeverity.HIGH == AlertSeverity.HIGH` is always true. Enums implement `Comparable` and `Serializable` by default. You can add fields, constructors (private), and methods to enums
       - Switch-case with enums: `switch(severity) { case LOW: ...; case HIGH: ...; }` — exhaustiveness is not enforced (missing cases are allowed), but the compiler warns. `values()` returns all constants as an array for iteration
       - EnumSet and EnumMap are specialized, high-performance implementations backed by bit vectors. `EnumSet.of(LOW, HIGH)` is faster than `HashSet` for enum values. When to use enums vs constants: enums provide type safety (you can't pass an invalid int), names, values(), and can carry behavior. The JVM ensures enum instantiation safety — reflection cannot create new enum instances (`IllegalArgumentException` on `Enum.newInstance()`)
54. What is garbage collection?
    - **Answer:**
       - GC automatically reclaims memory from objects no longer reachable
       - In Spring Boot apps, GC is relied upon to clean up request-scoped objects and DTOs, though heap settings may need tuning during bulk data processing to avoid pauses
       - Mark-sweep-compact algorithm: GC marks all reachable objects starting from GC roots, sweeps (reclaims) unmarked objects, and compacts surviving objects to eliminate memory fragmentation. Compaction is expensive but prevents fragmentation
       - Generational collection: young generation (new objects, collected frequently via minor GC) and old generation (long-lived objects, collected less frequently via major GC). Most objects die young — copying collectors in young gen are efficient because they only copy survivors
       - Common collectors: G1 (default since Java 9, balanced throughput/latency with predictable pause times), ZGC (ultra-low latency <10ms, scalable heap up to 16TB), Shenandoah (low-pause concurrent collector). Monitor GC with flags: `-XX:+PrintGCDetails`, `-Xlog:gc*` (Java 9+), JFR (Java Flight Recorder). Tuning for low-latency Kafka consumers: increase young gen size to reduce minor GC frequency, use G1 with `-XX:MaxGCPauseMillis=50`
55. What are GC roots?
    - **Answer:**
       - GC roots are special objects from which the GC traces reachability, including active thread stacks, static fields, JNI references, and monitor locks
       - Objects not reachable from any root are candidates for collection
       - Root scanning process: GC starts by scanning thread stacks (local variables and parameters of active methods), then static fields of loaded classes, then JNI (native method) references, then monitor locks (objects held by synchronized blocks). Root scanning contributes to GC pause time
       - Common root categories: running threads, static variables in loaded classes, JNI global/local references, monitors used for synchronization, and objects pending finalization (finalizer queue). JVM internal roots include class loading data and JIT compiler references
       - Memory leak analysis: use Eclipse MAT (Memory Analyzer Tool) to take heap dumps and trace from GC roots to leaked objects via "Path to GC Roots". VisualVM and JFR also provide heap dump capabilities. Identify unwanted references from roots — for example, a static collection holding references to objects that should have been garbage collected
56. What is memory leak in Java?
    - **Answer:**
       - A memory leak occurs when objects are no longer needed but remain reachable
       - A common example is a cached map in a static HashMap that is never cleared, causing heap growth over time
       - Common leak patterns: unclosed resources (InputStream, Connection) that hold native memory, ThreadLocal variables not removed after use (thread pools keep threads alive), inner class references (non-static inner classes implicitly reference their outer class), and listener registrations that are never unregistered
       - Detection via heap dumps: compare heap dumps over time to identify growing object counts. Use Eclipse MAT's Leak Suspects report to automatically identify potential leaks. JFR can record allocation patterns to find where leaked objects were created
       - Weak references as cleanup tools: WeakHashMap automatically removes entries when keys are garbage collected, preventing memory leaks in caches. SoftReference provides a memory-sensitive cache that the GC clears before throwing OOM. PhantomReference tracks object finalization for cleanup of external resources
57. How can memory leaks happen in Java?
    - **Answer:**
       - Common causes include forgetting to close resources, holding objects in static collections, ThreadLocal not removed after use, JVM cached String intern, and unclosed streams
       - A static cache map, for example, can cause gradual heap exhaustion
       - WeakHashMap for incident-driven cleanup: entries are automatically removed when keys become weakly reachable, preventing unbounded cache growth. Use it for caches where entries should be collected when no other references exist
       - JDBC connection leaks crash applications: unclosed connections exhaust the connection pool, causing new requests to block or fail. Leaked connections may also hold database server resources. Prevention: use try-with-resources for all JDBC operations, monitor connection pool metrics, and set connection timeouts
       - Profiling with VisualVM or JFR: VisualVM's sampler tracks heap usage over time and identifies object types with the most instances. JFR records allocation events and GC activity with low overhead. Preventive patterns: try-with-resources for all AutoCloseable resources, bounded caches (Caffeine with maximumSize), and always remove ThreadLocal variables in finally blocks (especially in thread pools)
58. What are strong, weak, soft, and phantom references?
    - **Answer:**
       - Strong references prevent GC collection; soft references are collected before OOM (useful for caches); weak references are collected at next GC (used by WeakHashMap); phantom references track object finalization
       - WeakHashMap is useful for temporary caching in data aggregation scenarios
       - ReferenceQueue interaction: when a soft/weak/phantom referenced object is collected, the reference is enqueued in a ReferenceQueue. You poll the queue to perform cleanup (e.g., removing entries from a cache, releasing native resources). Phantom references use `ref.get()` which always returns null — they track pre-mortem cleanup, not the object itself
       - WeakHashMap works internally as a canonical mapping: entries have weakly-referenced keys. When a key becomes unreachable, GC collects it, enqueues the weak reference, and WeakHashMap's expungeStaleEntries() removes the entry during the next get/put/size call
       - Real framework use cases: Guava Cache uses weak/soft references for evictable caches. WeakHashMap is used for metadata caches (like `Class.getEnclosingClass()`). SoftReference is ideal for memory-sensitive caches — the GC clears soft references only when memory is low, giving the cache maximum lifetime. PhantomReference is used by cleaners (Java 9+) for post-mortem cleanup of external resources (file handles, native memory)
59. What is classloader?
    - **Answer:**
       - The classloader loads .class files into the JVM memory
       - In Spring Boot, the classloader handles loading from BOOT-INF/lib, which is why fat JARs work — classloader issues may arise when migrating from plain JAR to Spring Boot
       - Delegation model (parent-delegation): Bootstrap classloader loads core JDK classes (rt.jar, java.lang.*). Platform/Extension classloader loads endorsed libraries and JDK extensions. Application/System classloader loads application classes from the classpath. Custom classloaders extend this hierarchy. Each classloader first delegates to its parent before loading a class itself
       - Custom classloaders enable hot deployment: a custom classloader can load a new version of a class at runtime without restarting the JVM. This is how application servers and plugin systems work. You extend ClassLoader and override `findClass()` to load bytecode from custom sources
       - Tomcat's per-webapp classloader: each web application gets its own classloader to provide class isolation — different apps can use different versions of the same library. ClassNotFoundException troubleshooting: the class is not on the classpath, the wrong classloader is loading it, or parent-delegation is loading an older version. Debug with `ClassLoader.getResource()` and `-verbose:class` JVM flag
60. What are Java records?
    - **Answer:**
       - Records are concise data carriers introduced in Java 14
       - They automatically generate constructor, getters, equals, hashCode, and toString
       - Records are well-suited for DTO layers as immutable transfer objects to reduce boilerplate code
       - Canonical constructor: `record Point(int x, int y)` generates `Point(int x, int y)`, `x()`, `y()`, `equals()`, `hashCode()`, `toString()`. Compact constructor: `record Range(int start, int end) { Range { if (end < start) throw new IllegalArgumentException(); } }` validates without repeating parameters
       - Restrictions: records are implicitly final (cannot be extended), fields are implicitly final (no setters), cannot declare instance fields beyond the record components, and cannot extend other classes (they implicitly extend `java.lang.Record`). They can implement interfaces, have static fields/methods, and have instance methods
       - JPA interaction issue: JPA requires a no-arg constructor for entity hydration, but records only have the canonical constructor. Records work well as read-only DTOs, API response objects, and value types. For JPA entities, stick with traditional classes or use `@ConstructorProperties` with a constructor. Records support serialization correctly — they implement Serializable automatically if specified, and the component fields are serialized. Use cases for API response DTOs: `record ApiResponse<T>(boolean success, T data, String message)` eliminates boilerplate in controller return types
