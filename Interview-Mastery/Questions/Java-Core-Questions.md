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
      - In my projects, I rely on OOP for modular service/controller layers and on garbage collection so I rarely worry about manual memory management in Spring Boot apps
   - **If asked more:**
      - I can dive into platform independence via bytecode and JVM, explain how the classloader works, and contrast Java's memory model with languages like C++
      - I would also mention how features like try-with-resources helped me write cleaner DB resource handling
2. What is platform independence in Java?
   - **Answer:**
      - Java achieves platform independence through bytecode that runs on the JVM
      - I deploy the same Spring Boot JAR on Windows dev machines and Linux EC2 servers without recompiling, which is critical for our CI/CD pipeline
   - **If asked more:**
      - I can explain the compilation process (.java to .class), how JVMs exist per platform, and discuss portability challenges like file path separators or character encodings I encountered in cross-platform deployments
3. What is JVM, JRE, and JDK?
   - **Answer:**
      - JVM executes bytecode, JRE includes JVM plus core libraries, and JDK adds development tools like javac and javap
      - In my daily work, I only install JDK on dev machines and use JRE-based Docker images for production
   - **If asked more:**
      - I can explain the JVM architecture (class loader, runtime data areas, execution engine), how JIT compilation works, and why choosing the right JVM implementation matters for server performance
4. What is bytecode?
   - **Answer:**
      - Bytecode is the intermediate representation of Java source code, stored in .class files and executed by the JVM
      - I never work with bytecode directly, but understanding it helped me debug classpath issues in my CDMS project when compiled classes were stale
   - **If asked more:**
      - I can explain how javac produces bytecode, how the JVM verifies it before execution, and how tools like javap can inspect bytecode for troubleshooting generic type erasure or lambda desugaring
5. What is the difference between stack and heap memory?
   - **Answer:**
      - Stack stores primitive values and object references per thread, while heap stores all actual objects and is shared across threads
      - In my inventory project, large collections of serial records lived on heap while local loop variables stayed on stack
   - **If asked more:**
      - I can explain stack frames, how recursion causes StackOverflowError, how heap is divided into young/old generations, and how I tuned JVM heap settings to avoid OutOfMemoryError during bulk data processing
6. What is a class?
   - **Answer:**
      - A class is a blueprint for creating objects in Java
      - In my backend code, every entity like `Partner`, `SensorReading`, or `InventoryRecord` is a class with fields and methods
   - **If asked more:**
      - I can explain class members (fields, methods, constructors), static vs instance context, how a class is loaded into memory, and how I structure classes using layered architecture in Spring Boot
7. What is an object?
   - **Answer:**
      - An object is a runtime instance of a class with its own state
      - When I call `partnerRepository.findById(id)`, the returned `Partner` object has specific field values representing a real database row
   - **If asked more:**
      - I can explain object creation with `new`, memory allocation on heap, how the constructor initializes state, and object lifecycle from creation to garbage collection
8. What are constructors?
   - **Answer:**
      - Constructors initialize object state when an instance is created
      - In my DTOs and entities, I use constructors to set required fields so the object is never in an invalid state
   - **If asked more:**
      - I can explain default constructors, parameterized constructors, constructor chaining with `this()`, and why I prefer constructor injection over field injection in Spring services
9. What is constructor overloading?
   - **Answer:**
      - Constructor overloading lets me define multiple constructors with different parameters for flexible object creation
      - In my cold-chain project, I overloaded `Alert` constructors to accept either sensor ID alone or full sensor reading data
   - **If asked more:**
      - I can explain how Java differentiates overloaded constructors, why the `this()` call must be the first statement, and how it differs from builder patterns for complex object creation
10. What is method overloading?
    - **Answer:**
       - Method overloading means multiple methods share the same name but differ in parameters
       - I use it in my service layer when I need lookup methods with different filter criteria, like `findByPartnerId()` and `findByPartnerIdAndDate()`
    - **If asked more:**
       - I can explain compile-time polymorphism, how method resolution works, why return type alone cannot distinguish overloaded methods, and how autoboxing complicates overload resolution
11. What is method overriding?
    - **Answer:**
       - Method overriding allows a subclass to provide a specific implementation of a parent class method
       - In my projects, I override `equals()` and `hashCode()` in entity classes to ensure correct behavior in collections and JPA identity management
    - **If asked more:**
       - I can explain runtime polymorphism, the `@Override` annotation, covariant return types, the rule that overridden methods cannot be more restrictive, and how Spring AOP uses proxy-based overriding
12. What is inheritance?
    - **Answer:**
       - Inheritance lets a class acquire fields and methods from a parent class
       - In my CDMS project, I extended a base `AbstractReportService` to share common reporting logic across multiple report types
    - **If asked more:**
       - I can explain single inheritance in Java, the `extends` keyword, method overriding via inheritance, diamond problem with interfaces, and why I prefer composition over inheritance for maintainability
13. What is encapsulation?
    - **Answer:**
       - Encapsulation bundles data and methods together while hiding internal state via access modifiers
       - In my Spring Boot services, I keep entity fields private and expose behavior through public methods, preventing direct field manipulation
    - **If asked more:**
       - I can explain getters/setters, the principle of information hiding, how encapsulation supports maintainability, and why exposing internal collections directly can break encapsulation
14. What is abstraction?
    - **Answer:**
       - Abstraction hides complex implementation details and shows only essential features
       - I use abstraction when defining service interfaces in Spring Boot so controllers depend on contracts, not concrete implementations
    - **If asked more:**
       - I can explain abstract classes vs interfaces, how abstraction reduces coupling, how Spring's dependency injection supports programming to interfaces, and real examples from my layered architecture
15. What is polymorphism?
    - **Answer:**
       - Polymorphism allows objects to take multiple forms, behaving differently based on their actual type
       - In my cold-chain project, a single `NotificationSender` interface had `EmailSender` and `SmsSender` implementations triggered based on alert severity
    - **If asked more:**
       - I can explain compile-time (method overloading) vs runtime (method overriding) polymorphism, how the JVM uses vtable dispatch, and how Spring leverages polymorphism for dependency injection
16. What is the difference between compile-time and runtime polymorphism?
    - **Answer:**
       - Compile-time polymorphism is resolved during compilation through method overloading, while runtime polymorphism is resolved at runtime through method overriding
       - I use overloading for convenience methods and overriding for interface implementations in my projects
    - **If asked more:**
       - I can explain how javac resolves overloaded methods, how the JVM uses dynamic dispatch for overridden methods, performance implications, and how both forms appear together in Spring Boot code
17. What is an interface?
    - **Answer:**
       - An interface is a contract that defines method signatures without implementation
       - In my projects, I define repository and service interfaces to decouple layers and enable Spring to inject appropriate implementations
    - **If asked more:**
       - I can explain default and static methods in interfaces, functional interfaces vs marker interfaces, how interfaces support multiple inheritance of type, and when I choose an interface over an abstract class
18. What is an abstract class?
    - **Answer:**
       - An abstract class cannot be instantiated and may contain both abstract and concrete methods
       - In my CDMS project, I used an abstract `BaseValidationService` with common validation logic, leaving specific rules to subclasses
    - **If asked more:**
       - I can explain abstract method rules, constructors in abstract classes, access modifiers, how they differ from interfaces in terms of state and multiple inheritance, and when I choose abstract classes
19. Difference between abstract class and interface.
    - **Answer:**
       - Abstract classes can have state and constructors, while interfaces support only constants and abstract/default methods
       - I use abstract classes for shared logic across related classes and interfaces for defining contracts across unrelated classes
    - **If asked more:**
       - I can explain how Java 8 blurred the line with default methods, when to prefer one over the other, and examples from Spring like `JpaRepository` (interface) vs `AbstractPaginationHelper` (abstract class)
20. Can an interface have default methods?
    - **Answer:**
       - Yes, interfaces can have default methods with a body introduced in Java 8
       - I used this when creating a custom functional interface for data transformation, providing a default no-op implementation so implementers only override what they need
    - **If asked more:**
       - I can explain the diamond problem with default methods, how to resolve conflicts with explicit override, why default methods were introduced for backward compatibility in Streams, and their limitations
21. Can an interface have static methods?
    - **Answer:**
       - Yes, interfaces can have static methods with a body since Java 8
       - I use static helper methods in interfaces for utility functions like `ValidationUtils.isValidSerial()` that belong conceptually to the validation domain
    - **If asked more:**
       - I can explain that interface static methods are not inherited, how they differ from class static methods, use cases like factory methods, and Java 9's private methods in interfaces
22. What is the diamond problem?
    - **Answer:**
       - The diamond problem occurs when a class inherits from multiple sources with conflicting default method implementations
       - Java avoids it with classes using single inheritance, and for interfaces, the implementing class must override the conflicting method
    - **If asked more:**
       - I can explain how Java 8's default methods reintroduced this risk, how explicit override resolves it, C++'s approach vs Java's, and real scenarios like extending multiple event listener interfaces
23. What is the difference between `==` and `.equals()`?
    - **Answer:**
       - `==` compares object references (memory addresses), while `.equals()` compares logical content
       - In my inventory validation, I used `.equals()` on serial numbers (Strings) to check logical equality, never `==`
    - **If asked more:**
       - I can explain how `equals()` default behavior mimics `==` for objects unless overridden, how String interning affects `==`, best practices for overriding `equals()`, and pitfalls with wrapper class comparisons
24. What is the contract between `equals()` and `hashCode()`?
    - **Answer:**
       - If two objects are equal via `equals()`, they must have the same hash code
       - I override both in my entity classes so that HashSets and HashMaps work correctly for deduplication and lookup operations
    - **If asked more:**
       - I can explain the full contract including the reverse (unequal objects CAN share hash codes), why using only one breaks hash-based collections, common implementations using Objects utility, and Lombok's `@EqualsAndHashCode`
25. What happens if `hashCode()` is not overridden?
    - **Answer:**
       - Without overriding `hashCode()`, the default Object implementation uses memory address
       - Two logically equal objects would have different hash codes, causing issues in HashSets and HashMaps like duplicates or lookup failures
    - **If asked more:**
       - I can explain how HashMap buckets work, consequences like memory leaks from duplicate entries, debugging such issues in production, and tools like IDE-generated hash code using prime numbers
26. What is immutable class?
    - **Answer:**
       - An immutable class cannot be modified after creation
       - In my cold-chain project, I used immutable DTOs for sensor readings to ensure thread-safe data transfer across Kafka consumer threads without synchronization
    - **If asked more:**
       - I can explain the rules (final class, private final fields, no setters, defensive copying in getters), why String is immutable, benefits like caching and thread safety, and when immutability hurts performance
27. How do you create an immutable class?
    - **Answer:**
       - I declare the class final, make all fields private final, initialize them through the constructor, provide only getters without setters, and return defensive copies for mutable fields
       - I used this approach for value objects in my inventory system
    - **If asked more:**
       - I can explain the builder pattern alternative for classes with many fields, how records simplify immutability in Java 14+, why Collections.unmodifiableList() is not full immutability, and serialization concerns
28. Why is `String` immutable?
    - **Answer:**
       - String immutability enables caching (String pool), security (no tampering of class names or DB URLs), thread safety, and efficient hash code caching
       - In my projects, I rely on String immutability when using Strings as HashMap keys
    - **If asked more:**
       - I can explain the String pool mechanism, how substring memory works pre-Java 7 vs now, why StringBuilder is needed for concatenation in loops, and how reflection could technically break immutability
29. Difference between `String`, `StringBuilder`, and `StringBuffer`.
    - **Answer:**
       - String is immutable, StringBuilder is mutable and not thread-safe, StringBuffer is mutable and thread-safe
       - I use String for fixed values, StringBuilder for single-threaded concatenation in loops, and rarely use StringBuffer in modern code
    - **If asked more:**
       - I can explain internal char[], capacity management, performance benchmarks, why StringBuilder is faster than StringBuffer, and how javac optimizes simple `+` concatenation using StringBuilder
30. What is the String constant pool?
    - **Answer:**
       - The String constant pool is a special heap region that caches String literals to save memory
       - When I write `String s = "partner"` in multiple places, all references point to the same pooled object
    - **If asked more:**
       - I can explain `intern()`, how literals vs `new String()` behave, pool location changes from permgen to heap, how String deduplication works in G1 GC, and memory implications for large datasets
31. What is exception handling?
    - **Answer:**
       - Exception handling uses try-catch-finally to manage runtime errors gracefully
       - In my Spring Boot APIs, I use global exception handlers with `@ControllerAdvice` to return consistent error responses instead of stack traces
    - **If asked more:**
       - I can explain the exception hierarchy (Throwable -> Exception/RuntimeException), checked vs unchecked, try-with-resources for closing DB connections, and custom exceptions I defined for business logic failures
32. Difference between checked and unchecked exceptions.
    - **Answer:**
       - Checked exceptions are checked at compile time and must be handled or declared; unchecked exceptions (RuntimeException) are not
       - I use checked exceptions for recoverable conditions like file not found, and unchecked for programming bugs like null pointer
    - **If asked more:**
       - I can explain when to use each type, best practices in Spring Boot (unchecked preferred for transactional rollback), how Spring wraps checked exceptions in DataAccessException, and custom exception design
33. Difference between `throw` and `throws`.
    - **Answer:**
       - `throw` actually throws an exception instance, while `throws` declares that a method might throw certain checked exceptions
       - In my code, I `throw` custom exceptions from service methods and declare `throws` in method signatures
    - **If asked more:**
       - I can explain exception propagation, how throws works with overriding, chained exceptions, and how Spring's declarative transaction management handles rollback for runtime exceptions
34. Difference between `final`, `finally`, and `finalize`.
    - **Answer:**
       - `final` is a keyword for constants, non-overridable methods, and non-inheritable classes
       - `finally` is a try-catch block that always executes
       - `finalize()` is a deprecated GC callback
       - In my projects, I use `final` for constants and `finally` for resource cleanup
    - **If asked more:**
       - I can explain how `finally` interacts with return statements, why `finalize()` should never be relied upon, alternatives like Cleaner and AutoCloseable, and real cleanup patterns in Spring Boot
35. What is try-with-resources?
    - **Answer:**
       - Try-with-resources automatically closes resources that implement AutoCloseable
       - I use it in my CDMS project for JDBC connections and file I/O, ensuring resources are closed even if an exception occurs, without needing a finally block
    - **If asked more:**
       - I can explain the multi-resource syntax, how resources are closed in reverse order, suppressed exceptions, how to make custom resources AutoCloseable, and Spring Boot's equivalent in JPA template methods
36. What is a custom exception?
    - **Answer:**
       - A custom exception is a user-defined class extending Exception or RuntimeException
       - In my inventory project, I created `InvalidSerialException` to handle specific validation failures distinctly from generic system errors
    - **If asked more:**
       - I can explain when to extend RuntimeException vs Exception, constructor best practices (message, cause), custom fields for error codes, integration with global exception handlers, and serialization UID
37. What are access modifiers in Java?
    - **Answer:**
       - Access modifiers control visibility: `private` (class only), `default` (package), `protected` (package + subclasses), `public` (everywhere)
       - In my layered architecture, I keep fields `private` and decide method visibility based on which layer needs access
    - **If asked more:**
       - I can explain how access modifiers affect inheritance, package-private as the default, encapsulation benefits, how reflection bypasses access control, and module system (Java 9) restrictions
38. What is static keyword?
    - **Answer:**
       - `static` means a member belongs to the class, not instances
       - I use static constants for configuration keys, static utility methods for validation helpers, and static inner classes to group related types
    - **If asked more:**
       - I can explain static initialization blocks, why static methods cannot be overridden, how static variables are stored in the method area, thread safety concerns, and common anti-patterns like static service classes
39. What is final keyword?
    - **Answer:**
       - `final` on a variable makes it a constant, on a method prevents overriding, and on a class prevents inheritance
       - I mark service dependencies as `final` for immutability and use `final` constants for magic strings in configuration
    - **If asked more:**
       - I can explain blank final variables, why final parameters are useful in anonymous classes, how the JIT optimizes final methods, and how final helps with thread safety via safe publication
40. What is transient keyword?
    - **Answer:**
       - `transient` marks fields that should not be serialized
       - In my projects, I mark derived or cache fields as transient when entities are serialized for caching or cross-service communication
    - **If asked more:**
       - I can explain the serialization mechanism, how transient interacts with Externalizable, alternatives like `@JsonIgnore` in Jackson, and security concerns of serializing sensitive data
41. What is volatile keyword?
    - **Answer:**
       - `volatile` guarantees visibility of changes to a variable across threads, preventing thread-local caching
       - In my Kafka consumer configurations, I used volatile flags for graceful shutdown signals across threads
    - **If asked more:**
       - I can explain happens-before guarantees, why volatile does not provide atomicity (use AtomicInteger instead), common use cases (flags, double-checked locking), and comparison with synchronized
42. What is serialization?
    - **Answer:**
       - Serialization converts an object into a byte stream for storage or transmission
       - In my cold-chain project, sensor data objects were serialized when sent to Kafka topics and deserialized by consumer applications
    - **If asked more:**
       - I can explain Serializable interface, serialVersionUID importance, custom writeObject/readObject methods, how JSON serialization differs from Java serialization, and why Jackson is preferred in Spring Boot
43. What is marker interface?
    - **Answer:**
       - A marker interface has no methods but signals special behavior to the JVM or framework
       - `Serializable` and `Cloneable` are classic examples
       - In Spring Boot, `@Configuration` annotations serve a similar signaling purpose
    - **If asked more:**
       - I can explain how JVM checks for Serializable, why marker interfaces are considered a design pattern, the shift towards annotations as markers, and tradeoffs of custom marker interfaces vs annotations
44. What is cloning?
    - **Answer:**
       - Cloning creates a copy of an object using the `clone()` method of Object
       - In my inventory project, I cloned baseline configuration objects before applying partner-specific overrides to preserve the original
    - **If asked more:**
       - I can explain the Cloneable interface contract, why clone() is protected, shallow vs deep copy behavior, why copy constructors or factory methods are preferred over Cloneable, and serialization-based cloning
45. What is shallow copy and deep copy?
    - **Answer:**
       - Shallow copy copies only the top-level object, sharing references to nested objects
       - Deep copy recursively duplicates all referenced objects
       - In my entity copying for reports, I needed deep copy to avoid modifying cached data through references
    - **If asked more:**
       - I can explain Object.clone() doing shallow copy, how to implement deep copy via serialization or manual recursion, performance overhead of deep copy, and libraries like Apache Commons for cloning utilities
46. What are wrapper classes?
    - **Answer:**
       - Wrapper classes (Integer, Double, Boolean, etc.) box primitives into objects
       - I use them when collections require objects, like `Map<String, Integer>` for partner counts in inventory caching
    - **If asked more:**
       - I can explain autoboxing/unboxing, caching ranges (Integer cache -128 to 127), performance overhead of boxing in loops, comparison gotchas with `==`, and OptionalInt vs Optional<Integer>
47. What is autoboxing and unboxing?
    - **Answer:**
       - Autoboxing automatically converts primitives to wrapper objects, unboxing does the reverse
       - In my code, Java automatically converts when I put an `int` into a `List<Integer>` or use a wrapper in arithmetic
    - **If asked more:**
       - I can explain compiler-generated boxing/unboxing code, performance cost in tight loops, null pointer risks with unboxing null wrappers, and how to avoid pitfalls with primitive streams
48. What are annotations?
    - **Answer:**
       - Annotations are metadata tags added to code elements
       - In Spring Boot, I heavily use annotations like `@Service`, `@RestController`, `@Transactional`, and `@Cacheable` to declaratively configure behavior without XML
    - **If asked more:**
       - I can explain retention policies (SOURCE, CLASS, RUNTIME), target types, how Spring processes annotations via reflection, creating custom annotations for cross-cutting concerns, and meta-annotations
49. What is reflection?
    - **Answer:**
       - Reflection allows inspecting and invoking classes, methods, and fields at runtime
       - Spring Boot uses reflection heavily for dependency injection, but in my direct code I rarely use it — once for a dynamic field-mapping utility in my CDMS project
    - **If asked more:**
       - I can explain Class.forName(), getMethod/invoke, performance overhead, security restrictions with SecurityManager, how Spring minimizes reflection cost with caching, and alternatives like method handles
50. What are generics?
    - **Answer:**
       - Generics enable type-safe collections and classes by parameterizing types
       - In my projects, `List<SerialRecord>` ensures only SerialRecord objects are added, eliminating casting and catching type errors at compile time
    - **If asked more:**
       - I can explain type parameters, generic methods, bounded wildcards (? extends T / ? super T), how generics improve code reusability in Spring's JpaRepository<T, ID>, and the PECS principle
51. What is type erasure?
    - **Answer:**
       - Type erasure removes generic type information at runtime, so `List<String>` and `List<Integer>` both become just `List`
       - This means I cannot check generic types at runtime, which affected my reflection-based field mapper design
    - **If asked more:**
       - I can explain how javac replaces type parameters with bounds or Object, bridge methods, why you cannot create `new T()`, workarounds using TypeToken or Class<T> parameters, and implications for serialization
52. What is varargs?
    - **Answer:**
       - Varargs allow methods to accept variable number of arguments using `...` syntax
       - I used varargs in my validation framework to pass multiple error codes to a logging utility without overloading methods
    - **If asked more:**
       - I can explain the internal array creation, how varargs must be the last parameter, heap pollution warnings with generics, and when to avoid varargs for clarity
53. What is enum?
    - **Answer:**
       - Enums define a fixed set of named constants, and in Java they are full classes with fields and methods
       - In my cold-chain project, I used `AlertSeverity` enum (LOW, MEDIUM, HIGH, CRITICAL) with threshold values and action methods attached
    - **If asked more:**
       - I can explain enum singleton pattern, switch-case with enums, EnumSet/EnumMap for performance, when to use enums vs constants, and how JVM ensures enum instantiation safety against reflection
54. What is garbage collection?
    - **Answer:**
       - GC automatically reclaims memory from objects no longer reachable
       - In my Spring Boot apps, I rely on GC to clean up request-scoped objects and DTOs, but I had to tune heap settings during bulk inventory processing to avoid pauses
    - **If asked more:**
       - I can explain the mark-sweep-compact algorithm, generational collection (young/old), common collectors (G1, ZGC), how to monitor GC with JVM flags, and GC tuning for low-latency Kafka consumers
55. What are GC roots?
    - **Answer:**
       - GC roots are special objects from which the GC traces reachability, including active thread stacks, static fields, JNI references, and monitor locks
       - Objects not reachable from any root are candidates for collection
    - **If asked more:**
       - I can explain the root scanning process in GC, how it impacts pause times, common root categories, heap dump analysis tools (Eclipse MAT), and how memory leak analysis works by identifying unwanted references from roots
56. What is memory leak in Java?
    - **Answer:**
       - A memory leak occurs when objects are no longer needed but remain reachable
       - In my inventory project, I fixed a leak where cached serial record maps in a static HashMap were never cleared, causing heap growth over time
    - **If asked more:**
       - I can explain common leak patterns (unclosed streams, ThreadLocal misuse, inner class references, listener registrations), how to detect leaks via heap dumps, and weak references as cleanup tools
57. How can memory leaks happen in Java?
    - **Answer:**
       - Common causes include forgetting to close resources, holding objects in static collections, ThreadLocal not removed after use, JVM cached String intern, and unclosed streams
       - In my project, a static cache map caused gradual heap exhaustion
    - **If asked more:**
       - I can explain incident-driven cleanup with WeakHashMap, how JDBC connection leaks crash applications, profiling with VisualVM or JFR, and preventive patterns like try-with-resources and bounded caches
58. What are strong, weak, soft, and phantom references?
    - **Answer:**
       - Strong references prevent GC collection; soft references are collected before OOM (useful for caches); weak references are collected at next GC (used by WeakHashMap); phantom references track object finalization
       - I used WeakHashMap for temporary cache in cold-chain data aggregation
    - **If asked more:**
       - I can explain ReferenceQueue interaction, how WeakHashMap works internally for canonical mappings, soft reference as memory-sensitive cache, phantom reference for pre-mortem cleanup, and real use cases in frameworks like Guava cache
59. What is classloader?
    - **Answer:**
       - The classloader loads .class files into the JVM memory
       - In Spring Boot, the classloader handles loading from BOOT-INF/lib, which is why fat JARs work — I had to debug classloader issues when migrating from plain JAR to Spring Boot
    - **If asked more:**
       - I can explain the delegation model (Bootstrap -> Platform -> System -> custom), how custom classloaders enable hot deployment, Tomcat's per-webapp classloader, and common ClassNotFoundException troubleshooting
60. What are Java records?
    - **Answer:**
       - Records are concise data carriers introduced in Java 14
       - They automatically generate constructor, getters, equals, hashCode, and toString
       - In my DTO layers, I plan to use records for immutable transfer objects to reduce boilerplate code
    - **If asked more:**
       - I can explain canonical and compact constructors, restrictions (no extends, final fields), how records interact with JPA (issue with no-arg constructor), serialization behavior, and use cases for API response DTOs
