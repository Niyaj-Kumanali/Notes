# Java 8 and Modern Java Questions

## Questions

1. What are Java 8 features?
2. What is lambda expression?
3. What is functional interface?
4. What is `@FunctionalInterface`?
5. What is Stream API?
6. Difference between collection and stream.
7. Difference between intermediate and terminal operations.
8. What is lazy evaluation in streams?
9. Difference between `map()` and `flatMap()`.
10. Difference between `filter()` and `map()`.
11. Difference between `findFirst()` and `findAny()`.
12. Difference between sequential and parallel streams.
13. When should you avoid parallel streams?
14. What is `Optional`?
15. Why should we not use `Optional.get()` directly?
16. Difference between `orElse()` and `orElseGet()`.
17. What are method references?
18. What are default methods?
19. What is `CompletableFuture`?
20. Difference between `thenApply()` and `thenCompose()`.
21. Difference between `thenApply()` and `thenAccept()`.
22. How do you handle exceptions in `CompletableFuture`?
23. What is Java Date and Time API?
24. Difference between `LocalDateTime`, `ZonedDateTime`, and `Instant`.
25. What are records in Java?
26. What are sealed classes?
27. What is pattern matching?
28. What are switch expressions?
29. What are text blocks?
30. What are virtual threads?
31. Given a `List<Employee>` where each employee has Name, ID, Salary, and Experience, filter employees based on experience and salary criteria, then print the results in ascending or descending order. Write the Java code.

---

## Answers

1. What are Java 8 features?
   - **Answer:**
      - Java 8 introduced lambdas, Stream API, Optional, default methods, CompletableFuture, and the new Date/Time API
      - In my projects, I use lambdas and streams daily for collection processing — filtering partners, mapping DTOs, and collecting reports without loops
      - Java 8 was motivated by the need for functional-style programming in Java — lambdas enabled passing behavior as arguments without anonymous classes, which was critical for Stream API's design
      - The Date/Time API replaced java.util.Date and java.util.Calendar because those classes were mutable, not thread-safe, and used zero-based months (Month.JANUARY = 0)
      - The features I use most are Stream API for declarative collection processing and Optional for null-safety in repository queries
2. What is lambda expression?
   - **Answer:**
      - Lambda expressions let me pass behavior as a method argument concisely
      - In my cold-chain project, I used lambdas in Stream API to filter sensor readings above a threshold and map them to alert objects without writing verbose anonymous classes
      - Lambda syntax is `(parameters) -> expression` for single expressions or `(parameters) -> { statements; }` for multi-line bodies
      - The compiler infers types from the target functional interface — e.g., `(s) -> s.length()` infers `s` is a String if targeting `Function<String, Integer>`
      - Lambdas can only capture effectively final local variables — variables that are not modified after initialization
      - Under the hood, lambdas are compiled using `invokedynamic` (the `LambdaMetafactory`), which creates a class at runtime that implements the target functional interface
      - A common pitfall is trying to modify a captured variable inside the lambda, which causes a compile error — if you need to mutate state, use an `AtomicReference` or a collection
3. What is functional interface?
   - **Answer:**
      - A functional interface has exactly one abstract method and can be implemented by a lambda
      - Built-in ones like Predicate, Function, Consumer, and Supplier are used heavily in Stream API operations in my data processing pipelines
      - To create a custom functional interface, define an interface with a single abstract method — e.g., `interface AlertMapper<T, R> { R map(T sensor); }`
      - The `@FunctionalInterface` annotation enforces this contract at compile time — adding a second abstract method triggers a compilation error
      - Default methods do not count toward the abstract method count, so an interface with one abstract method and several default methods is still a functional interface
      - Example: `Comparator` has one abstract method `compare(T o1, T o2)` and many default methods like `thenComparing()`, so it is a functional interface — I can use it as `Comparator.comparing(Employee::getSalary)` with a lambda
4. What is `@FunctionalInterface`?
   - **Answer:**
      - @FunctionalInterface is an annotation that marks an interface as intended for lambda use
      - The compiler enforces exactly one abstract method
      - I use it when defining custom functional interfaces for specific transformation logic in my validation framework
      - Overriding `equals()` or `hashCode()` from `Object` does not count as abstract methods — the annotation only tracks the interface's own abstract methods, so adding `equals()` is safe
      - If you accidentally add a second abstract method, the compiler produces: "Multiple non-overriding abstract methods found in interface"
      - `Runnable` (with `run()`) and `Callable<V>` (with `call()`) are both functional interfaces despite not being annotated — the annotation is optional but serves as documentation and compile-time safety
5. What is Stream API?
   - **Answer:**
      - Stream API processes sequences of data declaratively using functional operations
      - In my inventory project, I used streams to filter, sort, and collect serial records by partner, replacing complex for-loops with readable one-liners
      - A stream pipeline consists of a source, zero or more intermediate operations, and a terminal operation
      - Stream sources include collections (`stream()`, `parallelStream()`), arrays (`Arrays.stream()`), I/O (`BufferedReader.lines()`), and generators (`Stream.generate()`, `Stream.iterate()`)
      - Streams do not modify the underlying source — filtering a stream never removes elements from the original collection
      - `Collectors.toMap()`, `Collectors.groupingBy()`, and `Collectors.partitioningBy()` are essential for aggregation — I use `groupingBy` to bucket serial records by partner in my inventory reports
6. Difference between collection and stream.
   - **Answer:**
      - Collections store all elements in memory, while streams compute elements on demand and cannot be reused
      - I use collections as data storage and streams for transformation pipelines — the data stays in the collection, streams just process it once
      - Collections are about data — you store, retrieve, and manage elements; streams are about computation — you describe a pipeline of operations on data
      - Collections use external iteration (for-each loops), while streams use internal iteration — the stream library controls traversal, enabling optimizations like fusion and short-circuiting
      - Streams can be parallelized easily by calling `.parallelStream()` or `.parallel()` because the framework handles splitting, thread coordination, and merging without user-managed threads
      - A stream is consumed after a terminal operation — calling any operation on the same stream reference after `collect()` or `forEach()` throws `IllegalStateException`
7. Difference between intermediate and terminal operations.
   - **Answer:**
      - Intermediate operations (filter, map, sorted) return a new stream and are lazy
      - Terminal operations (collect, forEach, count) trigger processing
      - In my filter-and-collect pattern, filter and map just build a pipeline until collect executes everything
      - Intermediate operations are fused — the JVM optimizes the pipeline so that for each element, all intermediate steps execute before moving to the next element, reducing passes over the data
      - Order matters: filtering first with `filter()` reduces the number of elements that `map()` must process, so always place `filter()` before `map()` when possible
      - Short-circuiting terminal operations like `findFirst()`, `anyMatch()`, and `limit()` stop processing once the condition is met, avoiding unnecessary work on remaining elements
      - `peek()` is an intermediate operation for debugging — it performs a side effect without altering elements, unlike `forEach()` which is terminal and consumes the stream
8. What is lazy evaluation in streams?
   - **Answer:**
      - Lazy evaluation means intermediate operations are not executed until a terminal operation is invoked
      - In my code, chaining multiple filters on a large collection does not process elements until collect() is called, enabling optimization like operation fusion
      - Laziness enables infinite streams — `Stream.generate(Math::random).limit(10)` generates exactly 10 random numbers without attempting infinite computation
      - The JVM optimizes the pipeline by fusing consecutive intermediate operations — e.g., `filter().map().filter()` processes each element through all three steps in a single pass rather than creating intermediate streams
      - You can debug lazy pipelines using `peek()` to inspect elements as they flow through the pipeline — `.filter(x -> x > 5).peek(System.out::println).map(...)`
      - Lazy evaluation improves performance for early-terminating operations because `findFirst()` stops processing after the first match, and `anyMatch()` stops at the first true — remaining elements are never touched
9. Difference between `map()` and `flatMap()`.
   - **Answer:**
      - map transforms each element 1-to-1
      - flatMap transforms 1-to-many and flattens the result
      - I use map for simple DTO conversion (Entity->ResponseDTO) and flatMap when each sensor reading produces multiple alert objects in my cold-chain project
      - `flatMap` works with nested collections — given a `List<List<Integer>>`, `flatMap` flattens it into a single `Stream<Integer>`, whereas `map` would produce `Stream<List<Integer>>`
      - `Optional.flatMap` is used for chaining operations that themselves return Optional — e.g., `optionalUser.flatMap(User::findDepartment)` avoids `Optional<Optional<Department>>`
      - With `Collectors.groupingBy`, you can use downstream flatMap-like behavior by grouping and then aggregating nested lists
      - In streams, `flatMap` differs from `Optional.flatMap` only in context: both flatten nested structures, but Stream's flatMap takes a function returning `Stream<T>` while Optional's returns `Optional<T>`
10. Difference between `filter()` and `map()`.
    - **Answer:**
       - filter selects elements matching a predicate (narrowing)
       - map transforms each element (changing)
       - I chain filter before map to reduce processing — e.g., filter valid serial records then map to response DTOs in inventory APIs
       - `filter` takes a `Predicate<T>` (returns boolean) and keeps elements where the predicate is true; `map` takes a `Function<T, R>` and transforms each element to a new value
       - Combining `filter` + `map` is idiomatic in Stream pipelines — filter first to reduce the dataset, then map to extract or transform the fields you need
       - Placing filter first reduces downstream work because fewer elements flow through `map`, `sorted`, and other subsequent operations
       - `distinct()` and `limit()` are also filtering operations — `distinct()` filters duplicates (uses `equals`/`hashCode`) and `limit()` truncates the stream to at most n elements
11. Difference between `findFirst()` and `findAny()`.
    - **Answer:**
       - findFirst returns the first element in encounter order
       - findAny returns any element non-deterministically
       - In sequential streams they behave identically, but findAny is optimized for parallel streams because it does not enforce ordering
       - Encounter order depends on the source — `List` and `LinkedList` have encounter order; `HashSet` does not; `TreeSet` uses natural ordering
       - `findFirst` is slower in parallel streams because the framework must process elements in encounter order, which adds synchronization overhead — `findAny` can return any thread's result immediately
       - Real use cases: use `findAny()` when any valid partner suffices (e.g., "find any available supplier") and `findFirst()` when order matters (e.g., "find the earliest alert by timestamp")
       - Both return `Optional<T>`, so combine with `orElseThrow()` for cases where the element must exist, or `orElse(defaultValue)` for fallback behavior
12. Difference between sequential and parallel streams.
    - **Answer:**
       - Sequential streams process elements in a single thread
       - Parallel streams split work across multiple threads using ForkJoinPool
       - I avoid parallel streams in my projects because most of my collections are small or involve I/O where parallel adds overhead
       - Parallel streams use the common `ForkJoinPool` (with `Runtime.getRuntime().availableProcessors() - 1` threads by default), which is shared across the entire JVM — long-running parallel tasks can starve other parallel operations
       - The `Spliterator` interface divides the data source into chunks for parallel processing — `ArrayList` has an efficient spliterator, but `LinkedList` does not, making parallelism inefficient on linked structures
       - Parallel streams help with large datasets (100k+ elements) and CPU-intensive operations like complex mathematical computations or large-scale filtering
       - Pitfalls include shared mutable state (use `collect` with thread-safe collectors like `ConcurrentHashMap.newKeySet()`), blocking I/O (threads get blocked waiting for DB/network), and non-deterministic ordering
13. When should you avoid parallel streams?
    - **Answer:**
       - I avoid parallel streams for small datasets, I/O-bound operations (DB calls, HTTP), operations with shared mutable state, and ordered streams where merge overhead outweighs gain
       - My inventory collections of 10,000 records were too small to benefit
       - Parallel overhead includes splitting the data via Spliterator, coordinating threads through ForkJoinPool, and merging results — for small datasets, this overhead exceeds the processing time saved
       - The data must be large (100k+ elements) to benefit — below this threshold, the thread pool coordination cost dominates
       - Measure with JMH (Java Microbenchmark Harness) to compare sequential vs parallel performance — `@Benchmark` annotations on both variants reveal actual throughput and latency differences
       - Speedup depends on `n/parallelism` where `n` is the number of elements and `parallelism` is the thread count — with 4 threads and 8 elements, each thread gets only 2 elements, making splitting overhead wasteful
14. What is `Optional`?
    - **Answer:**
       - Optional is a container that may or may not hold a value, used to avoid null pointer exceptions
       - In my CDMS APIs, I used Optional when fetching partner data by ID from JpaRepository, then handled present/empty cases with orElseThrow
       - Optional encourages explicit null handling — by returning `Optional<T>`, the method signature forces the caller to deal with the empty case
       - Never use Optional for fields or method parameters — it is designed only for return types, because serializing Optional in JSON/DB produces awkward results
       - Common methods: `map()` to transform the value, `flatMap()` to chain Optional-returning operations, `filter()` to narrow, `ifPresent()` to consume with a side effect, and `orElse()`/`orElseThrow()` for extraction
       - There is a minor performance overhead compared to a direct null check (object allocation, method call), but the safety and readability benefits far outweigh it for API design
15. Why should we not use `Optional.get()` directly?
    - **Answer:**
       - Optional.get() throws NoSuchElementException if the Optional is empty, defeating the purpose of Optional
       - In my code, I use orElse(), orElseThrow(), or orElseGet() to provide safe default values or custom exceptions
       - `orElseThrow()` is preferred for "must exist" cases — it throws a meaningful exception (e.g., `new EntityNotFoundException("Partner not found: " + id)`) instead of a generic `NoSuchElementException`
       - `orElseGet()` takes a `Supplier<T>` that executes only when the Optional is empty, avoiding unnecessary object creation — e.g., `opt.orElseGet(() -> expensiveDatabaseLookup())`
       - Using `isPresent()` followed by `get()` is an anti-pattern because it duplicates what `ifPresent()`, `map()`, and `orElseThrow()` do more safely and concisely
       - IntelliJ flags direct `get()` calls with a warning — always replace `opt.get()` with `opt.orElseThrow()` or handle it within `ifPresent()`
16. Difference between `orElse()` and `orElseGet()`.
    - **Answer:**
       - orElse always evaluates the default value even if the Optional is present
       - orElseGet takes a Supplier that runs only when the Optional is empty
       - In performance-sensitive code, I use orElseGet to avoid unnecessary object creation
       - The critical difference: `orElse(computeDefault())` executes `computeDefault()` every time, even when the Optional holds a value — the default is evaluated eagerly
       - A real bug I encountered: calling `orElse(new ExpensiveReport())` on every query result created an unnecessary object each time, wasting memory and CPU; switching to `orElseGet(ExpensiveReport::new)` fixed it
       - `orElseThrow()` bridges Optional with custom exceptions — for cases where the value must exist, use `orElseThrow(() -> new EntityNotFoundException("Not found"))` instead of returning a null or default
17. What are method references?
    - **Answer:**
       - Method references provide shorter syntax for lambdas that call an existing method (ClassName::methodName)
       - I use them in stream pipelines — e.g., `.map(PartnerDTO::new)` for constructor references or `.forEach(logger::info)` for method calls
       - Four types: static method reference (`Integer::parseInt`), bound instance method reference (`String::toLowerCase` on a specific instance), unbound instance method reference (`String::length` taking the instance as parameter), and constructor reference (`ArrayList::new`)
       - Choose method references over lambdas when the lambda body directly calls a single method — `list.stream().map(String::toLowerCase)` is cleaner than `list.stream().map(s -> s.toLowerCase())`
       - Method references are not always more readable — when the lambda body includes additional logic (conditionals, multiple statements), a full lambda is clearer
18. What are default methods?
    - **Answer:**
       - Default methods allow interfaces to have method implementations without breaking implementing classes
       - I use them when extending functional interfaces with convenience methods, like adding a default `orElseThrow` method to a custom validation interface
       - They were introduced because Java 8 needed to add `forEach()` to `Collection` without breaking thousands of existing implementations — adding a default method meant no class was forced to implement it
       - When a class implements two interfaces with conflicting default methods, the class must override the method to resolve the conflict — "class wins over interface" is the primary resolution rule
       - If two sibling interfaces have the same default method and the class does not override it, a compilation error occurs — the diamond problem. You resolve it by explicitly overriding the method in the implementing class
19. What is `CompletableFuture`?
    - **Answer:**
       - CompletableFuture is a Future that can be manually completed and supports chaining async operations
       - In my cold-chain project, I used CompletableFuture to fetch sensor data from multiple IoT sources asynchronously and combine results without blocking
       - CompletableFuture extends `Future` with callback chaining — `thenApply()` for sync transforms, `thenCompose()` for async chaining, `thenAccept()` for consuming results
       - `supplyAsync(() -> ...)` creates a task returning a value; `runAsync(() -> ...)` creates a void task — both execute on the common ForkJoinPool by default or on a custom executor passed as the second argument
       - `allOf(future1, future2, future3)` combines multiple futures — I use it to fetch data from 3 IoT gateways in parallel and join the results only when all complete
       - Custom thread pool configuration: pass a `ThreadPoolExecutor` to `supplyAsync()` to avoid blocking the common ForkJoinPool — critical when mixing CPU-bound and I/O-bound async tasks
20. Difference between `thenApply()` and `thenCompose()`.
    - **Answer:**
       - thenApply transforms the result of a CompletableFuture synchronously (returns a nested CompletableFuture if the function returns one)
       - thenCompose flattens nested futures
       - I use thenApply for simple transformations and thenCompose to chain dependent async calls
       - `thenApply` is analogous to `Stream.map()` — it applies a function to the result; if the function itself returns a `CompletableFuture<T>`, you get `CompletableFuture<CompletableFuture<T>>`, which is nested and inconvenient
       - `thenCompose` is analogous to `Stream.flatMap()` — it flattens the nested future, returning `CompletableFuture<T>` directly
       - `thenCompose` avoids callback hell by allowing sequential async chains: `future.thenCompose(data -> fetchDetails(data)).thenApply(details -> transform(details))`
21. Difference between `thenApply()` and `thenAccept()`.
    - **Answer:**
       - thenApply transforms a value and returns a result
       - thenAccept consumes the value and returns Void
       - I use thenAccept when I need to perform a side effect (like logging or caching) after a future completes without returning a new value
       - `thenApply` takes a `Function<T, R>` and returns `CompletableFuture<R>`; `thenAccept` takes a `Consumer<T>` and returns `CompletableFuture<Void>`
       - `thenRun` takes a `Runnable` and performs an action after the future completes, without access to the result — useful for cleanup tasks
       - Use cases: `thenAccept` for sending notifications or updating cache after async processing; `thenApply` when the result feeds into the next stage of the pipeline
       - Exception propagation is identical for both — exceptions in either stage are captured and forwarded to downstream exception handlers
22. How do you handle exceptions in `CompletableFuture`?
    - **Answer:**
       - I use exceptionally() to recover from errors with a fallback value, handle() to process both success and failure, and whenComplete for side-effect cleanup
       - In my async IoT data pipeline, I logged errors via exceptionally and used fallback readings
       - Exceptions in one stage propagate to all downstream stages — a failed future carries the exception forward until a handler catches it
       - `handle()` differs from `exceptionally()` because it always runs regardless of success or failure — it receives both the result and the exception as parameters, making it ideal for cleanup or logging both paths
       - `completeExceptionally()` manually completes a future with an exception — useful when integrating with callback-based APIs or when a timeout occurs
       - Unchecked exceptions (`RuntimeException`) need careful handling because they silently propagate through the chain — always attach `exceptionally()` or `handle()` at the end of any async pipeline
23. What is Java Date and Time API?
    - **Answer:**
       - The java.time package provides immutable, thread-safe date/time classes like LocalDate, LocalDateTime, ZonedDateTime, and Instant
       - I use LocalDate for inventory dates and Instant for sensor timestamps in cold-chain, replacing the flawed java.util.Date
       - Legacy `Date` problems: mutable (thread-unsafe), poor design (months are zero-indexed, years start at 1900), and inconsistent — `Date` mixes date and time with timezone
       - `Duration` measures time between two instants (hours, minutes, seconds); `Period` measures date-based amounts (years, months, days)
       - `DateTimeFormatter` replaces `SimpleDateFormat` — it is immutable, thread-safe, and uses `format()`/`parse()` methods instead of the thread-unsafe `SimpleDateFormat.format()`
       - Timezone handling: `ZoneId` represents a timezone (e.g., `ZoneId.of("Asia/Kolkata")`); use `ZonedDateTime.of(localDateTime, zoneId)` to attach a timezone, or `Instant.atZone(zoneId)` to convert a UTC timestamp
24. Difference between `LocalDateTime`, `ZonedDateTime`, and `Instant`.
    - **Answer:**
       - LocalDateTime has no timezone
       - ZonedDateTime includes full timezone rules
       - Instant is a UTC timestamp
       - I use Instant for storing IoT sensor timestamps in InfluxDB, ZonedDateTime for user-facing displays, and LocalDate for inventory date-only fields
       - Convert between them: `LocalDateTime.atZone(ZoneId)` produces `ZonedDateTime`; `zonedDateTime.toInstant()` extracts the UTC instant; `Instant.atZone(ZoneId)` converts back to `ZonedDateTime`
       - Never store `LocalDateTime` without knowing the zone — it loses the timezone context, making DST transitions and time comparisons ambiguous
       - `ZonedDateTime` handles daylight saving transitions automatically — it uses `ZoneRules` to determine whether an offset change occurred at a given point in time
       - `OffsetDateTime` is a lighter alternative to `ZonedDateTime` — it stores the UTC offset (e.g., `+05:30`) without full timezone rules, useful for REST API parameters and database storage
25. What are records in Java?
    - **Answer:**
       - Records are immutable data carriers introduced in Java 14 that generate constructor, getters, equals, hashCode, and toString automatically
       - In my DTO layers, I use records for API response objects to reduce boilerplate and ensure immutability
       - The canonical constructor is generated from the state components declared in the record header — `record Employee(String name, int id)` auto-generates the constructor, `name()`, `id()`, `equals()`, `hashCode()`, and `toString()`
       - A compact constructor validates arguments before assignment — `record Employee(String name, int id) { Employee { if (id < 0) throw new IllegalArgumentException("Negative ID"); } }`
       - Records are implicitly `final` and extend `java.lang.Record` — you cannot extend another class but can implement interfaces
       - Records support standard Java serialization via `readResolve()` but have JPA limitations — they lack a no-arg constructor, which JPA requires, so they need workarounds or adapters for entity mapping
       - Pattern matching with records (Java 16+) enables deconstruction — `if (obj instanceof Employee(String name, int id))` extracts the components directly in a single expression
26. What are sealed classes?
    - **Answer:**
       - Sealed classes restrict which classes can extend them, providing controlled inheritance
       - I would use sealed classes for domain event types in my projects — like `SealedEvent permits SensorEvent, AlertEvent, SystemEvent` — for exhaustive pattern matching
       - The `permits` clause lists every allowed subclass — all permitted classes must be in the same module (or same package if not in a module), and each must be `final`, `sealed`, or `non-sealed`
       - Sealed interfaces work the same way — define a sealed interface and list permitted implementing classes
       - Exhaustive switch (Java 17+) with sealed types guarantees all cases are handled — the compiler enforces that every permitted type has a case, so you can omit the `default` clause safely
       - The `non-sealed` modifier on a permitted subclass reopens inheritance for that subclass — it allows any class to extend it, useful when one branch of the hierarchy should remain open
       - Sealed classes improve domain modeling compared to final or unrestricted classes — they express the complete set of subtypes, enabling the compiler to verify exhaustive pattern matching and preventing unexpected extensions
27. What is pattern matching?
    - **Answer:**
       - Pattern matching allows type checking and deconstruction in a single construct
       - In Java 16+, I can use pattern matching for instanceof: `if (obj instanceof String s)` which eliminates separate casting, making my validation code cleaner
       - It evolved across versions: `instanceof` pattern matching in Java 16, `switch` patterns in Java 17 (preview) and 19 (final), and record deconstruction in Java 19+
       - Guard patterns use `&&` to add conditions: `if (obj instanceof String s && s.length() > 5)` — the variable `s` is in scope for both the type check and the guard
       - Exhaustive matching with sealed classes ensures the compiler verifies all permitted subtypes are handled in switch expressions — no `default` clause needed when all cases are covered
       - Pattern matching reduces boilerplate in if-else chains — instead of `instanceof` + cast + field access in separate steps, a single pattern extracts everything: `if (obj instanceof Employee(String name, int id, double salary))`
28. What are switch expressions?
    - **Answer:**
       - Switch expressions return a value and use arrow syntax for concise cases
       - I use switch expressions when mapping enum values to strings or numeric thresholds, like converting `AlertSeverity.LOW` to a color code without break statements
       - Arrow syntax (`case X -> expression`) eliminates fall-through — each case is independent, making the code safer and more readable than traditional `case X: statement; break;`
       - Colon syntax (`case X: statement;`) still works in switch expressions but requires explicit `break` or `yield` — prefer arrow syntax for simplicity
       - The `yield` keyword returns a value from a block case: `case LARGE -> { int v = compute(); yield v; }`
       - Switch expressions are exhaustive — every possible value must be handled, either by listing all cases or providing a `default` clause; enums with all values listed need no default
       - Pattern matching in switch (Java 17+) allows richer conditions: `case String s && s.length() > 10 -> "long string"` — combining type checks with guards in a single expression
29. What are text blocks?
    - **Answer:**
       - Text blocks (""") provide multi-line string literals with clean formatting
       - I use them for SQL queries in my inventory project — embedding multi-line validation queries without concatenation or escape sequences for newlines
       - Leading whitespace is stripped using the minimum indentation line as the alignment reference — the closing `"""` position determines the left margin, and all content is de-indented accordingly
       - Escape sequences work inside text blocks — `\t` for tabs, `\n` for newlines, `\"` for quotes, and `\` at the end of a line trims the trailing whitespace
       - Text blocks dramatically improve readability of JSON, HTML, and SQL strings compared to escaped single-line strings — e.g., embedding a multi-line SQL query without `\"` and `\n`
       - Text blocks were a preview feature in Java 13 (text blocks preview 1) and Java 14 (preview 2), and became final in Java 15
30. What are virtual threads?
    - **Answer:**
       - Virtual threads are lightweight threads from Project Loom (Java 21+) that are managed by the JVM, not the OS, allowing millions of concurrent tasks
       - I am excited to use them for handling high-volume Kafka message processing where each sensor reading can be a virtual thread
       - Virtual threads differ from platform threads in that the JVM schedules them onto carrier threads (backed by OS threads) — millions of virtual threads multiplex onto a small pool of carrier threads
       - Use virtual threads for I/O-heavy workloads with many concurrent tasks (HTTP calls, database queries, file I/O); avoid them for CPU-bound computations because they don't add parallelism — they just reduce context-switching overhead
       - Virtual threads can pin carrier threads when inside `synchronized` blocks — this defeats the purpose because the carrier thread is blocked; use `ReentrantLock` instead of `synchronized` for I/O-bound code
       - Create them with `Executors.newVirtualThreadPerTaskExecutor()` — each submitted task runs on its own virtual thread, with no thread pool sizing needed
       - Spring Boot 3.2+ supports virtual threads natively — set `spring.threads.virtual.enabled=true` and the framework uses virtual threads for all request handling automatically
31. Given a `List<Employee>` where each employee has Name, ID, Salary, and Experience, filter employees based on experience and salary criteria, then print the results in ascending or descending order. Write the Java code.
   - **Answer:**
      - Using the Stream API I filter with a `Predicate` and sort with `Comparator`
      - For example, to keep only employees with at least 5 years experience and a salary greater than 50,000, sorted by salary ascending:
   ```java
   List<Employee> result = employees.stream()
       .filter(e -> e.getExperience() >= 5)
       .filter(e -> e.getSalary() > 50000)
       .sorted(Comparator.comparing(Employee::getSalary))   // ascending
       .collect(Collectors.toList());
   result.forEach(System.out::println);
   ```
   - For descending order, use `.sorted(Comparator.comparing(Employee::getSalary).reversed())` or `.sorted(Comparator.comparingDouble(Employee::getSalary).reversed())`
   - If I also want to tie-break by experience, I chain comparators with `Comparator.comparing(Employee::getSalary).thenComparing(Employee::getExperience)`
   - `filter` is an intermediate operation and `collect` is terminal — the pipeline is lazy and elements are not processed until `collect` executes
   - Both filters can be combined into one lambda `e -> e.getExperience() >= 5 && e.getSalary() > 50000` for conciseness, though two separate filters improve readability
   - For null-safe sorting, wrap the comparator: `Comparator.nullsLast(Comparator.comparing(Employee::getSalary))` — this pushes null values to the end of the sorted result
   - The alternative without streams is a plain `for` loop plus `Collections.sort()` with a custom `Comparator` — clearer for simple cases, but the stream approach is more composable for complex pipelines
   - `Comparator.comparing` uses a type-aware comparator; prefer `Comparator.comparingInt` or `comparingDouble` for primitive fields to avoid autoboxing overhead