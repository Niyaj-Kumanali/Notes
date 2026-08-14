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
   - **Answer:** Java 8 introduced lambdas, Stream API, Optional, default methods, CompletableFuture, and the new Date/Time API. In my projects, I use lambdas and streams daily for collection processing — filtering partners, mapping DTOs, and collecting reports without loops.
   - **If asked more:** I can explain the motivation behind each feature, how lambdas enabled functional-style programming, why the Date/Time API replaced Date/Calendar, and which features I use most (Stream API and Optional).
2. What is lambda expression?
   - **Answer:** Lambda expressions let me pass behavior as a method argument concisely. In my cold-chain project, I used lambdas in Stream API to filter sensor readings above a threshold and map them to alert objects without writing verbose anonymous classes.
   - **If asked more:** I can explain lambda syntax (parameters -> body), type inference by the compiler, variable capture (effectively final), how lambdas are compiled to invokedynamic, and common pitfalls like modifying captured variables.
3. What is functional interface?
   - **Answer:** A functional interface has exactly one abstract method and can be implemented by a lambda. Built-in ones like Predicate, Function, Consumer, and Supplier are used heavily in Stream API operations in my data processing pipelines.
   - **If asked more:** I can explain how to create custom functional interfaces, how @FunctionalInterface enforces the contract, how default methods do not break functional interface status, and examples like Comparator with lambda.
4. What is `@FunctionalInterface`?
   - **Answer:** @FunctionalInterface is an annotation that marks an interface as intended for lambda use, and the compiler enforces exactly one abstract method. I use it when defining custom functional interfaces for specific transformation logic in my validation framework.
   - **If asked more:** I can explain how overriding equals from Object does not count as abstract, what happens if multiple abstract methods exist (compilation error), and how Runnable and Callable are functional interfaces.
5. What is Stream API?
   - **Answer:** Stream API processes sequences of data declaratively using functional operations. In my inventory project, I used streams to filter, sort, and collect serial records by partner, replacing complex for-loops with readable one-liners.
   - **If asked more:** I can explain pipelines, intermediate vs terminal operations, stream sources (collections, arrays, I/O, generate/iterate), how streams do not modify the source, and Collectors.toMap/groupingBy for aggregation.
6. Difference between collection and stream.
   - **Answer:** Collections store all elements in memory, while streams compute elements on demand and cannot be reused. I use collections as data storage and streams for transformation pipelines — the data stays in the collection, streams just process it once.
   - **If asked more:** I can explain how collections are about data, streams are about computation, internal vs external iteration, why streams can be parallelized easily, and how a stream after terminal operation is consumed.
7. Difference between intermediate and terminal operations.
   - **Answer:** Intermediate operations (filter, map, sorted) return a new stream and are lazy, terminal operations (collect, forEach, count) trigger processing. In my filter-and-collect pattern, filter and map just build a pipeline until collect executes everything.
   - **If asked more:** I can explain how intermediate operations are fused, why order matters (filter first, then map), example of short-circuiting terminal operations (findFirst, anyMatch), and how peek differs from forEach.
8. What is lazy evaluation in streams?
   - **Answer:** Lazy evaluation means intermediate operations are not executed until a terminal operation is invoked. In my code, chaining multiple filters on a large collection does not process elements until collect() is called, enabling optimization like operation fusion.
   - **If asked more:** I can explain how laziness enables infinite streams (Stream.generate, Stream.iterate), how the JVM optimizes the pipeline, debugging with peek, and why lazy evaluation improves performance for early-terminating operations.
9. Difference between `map()` and `flatMap()`.
   - **Answer:** map transforms each element 1-to-1, flatMap transforms 1-to-many and flattens the result. I use map for simple DTO conversion (Entity->ResponseDTO) and flatMap when each sensor reading produces multiple alert objects in my cold-chain project.
   - **If asked more:** I can explain how flatMap works with nested collections, Optional.flatMap for chaining optional operations, flatMapping with Collectors.groupingBy for multi-level aggregation, and how flatMap differs in stream vs Optional.
10. Difference between `filter()` and `map()`.
   - **Answer:** filter selects elements matching a predicate (narrowing), map transforms each element (changing). I chain filter before map to reduce processing — e.g., filter valid serial records then map to response DTOs in inventory APIs.
   - **If asked more:** I can explain how filter uses Predicate, map uses Function, how combining filter+map is idiomatic, why placing filter first reduces downstream work, and how distinct() and limit() are also filtering operations.
11. Difference between `findFirst()` and `findAny()`.
   - **Answer:** findFirst returns the first element in encounter order, findAny returns any element non-deterministically. In sequential streams they behave identically, but findAny is optimized for parallel streams because it does not enforce ordering.
   - **If asked more:** I can explain how encounter order depends on the source (List vs Set vs HashSet), why findFirst is slower in parallel, real use cases like "find any valid partner" vs "find the earliest alert", and orElseThrow usage.
12. Difference between sequential and parallel streams.
   - **Answer:** Sequential streams process elements in a single thread, parallel streams split work across multiple threads using ForkJoinPool. I avoid parallel streams in my projects because most of my collections are small or involve I/O where parallel adds overhead.
   - **If asked more:** I can explain the common ForkJoinPool, how parallelism works with spliterator, when parallel streams help (large datasets, CPU-intensive operations), and pitfalls like shared mutable state or blocking operations.
13. When should you avoid parallel streams?
   - **Answer:** I avoid parallel streams for small datasets, I/O-bound operations (DB calls, HTTP), operations with shared mutable state, and ordered streams where merge overhead outweighs gain. My inventory collections of 10,000 records were too small to benefit.
   - **If asked more:** I can explain how parallel overhead includes splitting, merging, thread coordination, why the data must be large (100k+ elements) to benefit, how to measure with JMH, and how n/parallelism determines speedup.
14. What is `Optional`?
   - **Answer:** Optional is a container that may or may not hold a value, used to avoid null pointer exceptions. In my CDMS APIs, I used Optional when fetching partner data by ID from JpaRepository, then handled present/empty cases with orElseThrow.
   - **If asked more:** I can explain how Optional encourages explicit null handling, why it is not for fields or method parameters (only return types), common methods (map, flatMap, filter, ifPresent), and performance overhead vs direct null check.
15. Why should we not use `Optional.get()` directly?
   - **Answer:** Optional.get() throws NoSuchElementException if the Optional is empty, defeating the purpose of Optional. In my code, I use orElse(), orElseThrow(), or orElseGet() to provide safe default values or custom exceptions.
   - **If asked more:** I can explain how orElseThrow is preferred for "must exist" cases, how orElseGet avoids eager evaluation, why isPresent()+get() is an anti-pattern (use ifPresent or map instead), and how IntelliJ warns about direct get().
16. Difference between `orElse()` and `orElseGet()`.
   - **Answer:** orElse always evaluates the default value even if the Optional is present, while orElseGet takes a Supplier that runs only when the Optional is empty. In performance-sensitive code, I use orElseGet to avoid unnecessary object creation.
   - **If asked more:** I can explain why orElse with method call (orElse(computeDefault())) executes computeDefault always, a real bug I encountered with expensive default computation, and how orElseThrow bridges Optional with custom exceptions.
17. What are method references?
   - **Answer:** Method references provide shorter syntax for lambdas that call an existing method (ClassName::methodName). I use them in stream pipelines — e.g., `.map(PartnerDTO::new)` for constructor references or `.forEach(logger::info)` for method calls.
   - **If asked more:** I can explain the four types: static (Integer::parseInt), instance (String::toLowerCase), constructor (ArrayList::new), and arbitrary object (String::length), and when to choose method references over lambdas for readability.
18. What are default methods?
   - **Answer:** Default methods allow interfaces to have method implementations without breaking implementing classes. I use them when extending functional interfaces with convenience methods, like adding a default `orElseThrow` method to a custom validation interface.
   - **If asked more:** I can explain why they were introduced (Java 8 streams needed Collection.forEach without breaking existing code), how conflicts are resolved (class wins over interface), and the diamond problem with multiple default methods.
19. What is `CompletableFuture`?
   - **Answer:** CompletableFuture is a Future that can be manually completed and supports chaining async operations. In my cold-chain project, I used CompletableFuture to fetch sensor data from multiple IoT sources asynchronously and combine results without blocking.
   - **If asked more:** I can explain how it extends Future, the async callback chaining (thenApply, thenCompose), how supplyAsync/runAsync create async tasks, how I combined multiple futures with allOf, and custom thread pool configuration.
20. Difference between `thenApply()` and `thenCompose()`.
   - **Answer:** thenApply transforms the result of a CompletableFuture synchronously (returns a nested CompletableFuture if the function returns one), while thenCompose flattens nested futures. I use thenApply for simple transformations and thenCompose to chain dependent async calls.
   - **If asked more:** I can explain how thenApply is like Stream.map and thenCompose is like Stream.flatMap, the issue of CompletableFuture<CompletableFuture<>> with thenApply, and how thenCompose avoids callback hell.
21. Difference between `thenApply()` and `thenAccept()`.
   - **Answer:** thenApply transforms a value and returns a result, thenAccept consumes the value and returns Void. I use thenAccept when I need to perform a side effect (like logging or caching) after a future completes without returning a new value.
   - **If asked more:** I can explain how each method relates to Function vs Consumer, how thenRun works for Runnable, use cases like thenAccept for sending notifications after async processing, and exception propagation.
22. How do you handle exceptions in `CompletableFuture`?
   - **Answer:** I use exceptionally() to recover from errors with a fallback value, handle() to process both success and failure, and whenComplete for side-effect cleanup. In my async IoT data pipeline, I logged errors via exceptionally and used fallback readings.
   - **If asked more:** I can explain how exceptions in one stage propagate to downstream stages, how handle() differs from exceptionally (handle always runs), how completeExceptionally is used, and why unchecked exceptions need careful handling.
23. What is Java Date and Time API?
   - **Answer:** The java.time package provides immutable, thread-safe date/time classes like LocalDate, LocalDateTime, ZonedDateTime, and Instant. I use LocalDate for inventory dates and Instant for sensor timestamps in cold-chain, replacing the flawed java.util.Date.
   - **If asked more:** I can explain the problems with legacy Date (mutable, poor design, month=0), how Duration/Period measure time, how DateTimeFormatter replaces SimpleDateFormat, and timezone handling with ZoneId.
24. Difference between `LocalDateTime`, `ZonedDateTime`, and `Instant`.
   - **Answer:** LocalDateTime has no timezone, ZonedDateTime includes full timezone rules, and Instant is a UTC timestamp. I use Instant for storing IoT sensor timestamps in InfluxDB, ZonedDateTime for user-facing displays, and LocalDate for inventory date-only fields.
   - **If asked more:** I can explain how to convert between them, why LocalDateTime should not be stored without knowing the zone, daylight saving handling in ZonedDateTime, and OffsetDateTime as a lighter alternative to ZonedDateTime.
25. What are records in Java?
   - **Answer:** Records are immutable data carriers introduced in Java 14 that generate constructor, getters, equals, hashCode, and toString automatically. In my DTO layers, I use records for API response objects to reduce boilerplate and ensure immutability.
   - **If asked more:** I can explain canonical constructor, compact constructor for validation, how records are final and extend java.lang.Record, serialization behavior, JPA limitations (no-arg constructor missing), and pattern matching with records in Java 16+.
26. What are sealed classes?
   - **Answer:** Sealed classes restrict which classes can extend them, providing controlled inheritance. I would use sealed classes for domain event types in my projects — like `SealedEvent permits SensorEvent, AlertEvent, SystemEvent` — for exhaustive pattern matching.
   - **If asked more:** I can explain permits clause, sealed interfaces as well, how exhaustive switch (Java 17+) works with sealed types, the `non-sealed` modifier, and how they improve domain modeling compared to final or unrestricted classes.
27. What is pattern matching?
   - **Answer:** Pattern matching allows type checking and deconstruction in a single construct. In Java 16+, I can use pattern matching for instanceof: `if (obj instanceof String s)` which eliminates separate casting, making my validation code cleaner.
   - **If asked more:** I can explain how it evolved across versions (instanceof in 16, switch in 17+, records in 19+), guard patterns with `&&`, exhaustive matching with sealed classes, and how it reduces boilerplate in if-else chains.
28. What are switch expressions?
   - **Answer:** Switch expressions return a value and use arrow syntax for concise cases. I use switch expressions when mapping enum values to strings or numeric thresholds, like converting `AlertSeverity.LOW` to a color code without break statements.
   - **If asked more:** I can explain arrow vs colon syntax, why no fall-through with arrows, yield keyword for blocks, exhaustive requirements (or default needed), and pattern matching in switch from Java 17+ for richer conditions.
29. What are text blocks?
   - **Answer:** Text blocks (""") provide multi-line string literals with clean formatting. I use them for SQL queries in my inventory project — embedding multi-line validation queries without concatenation or escape sequences for newlines.
   - **If asked more:** I can explain how leading whitespace is stripped using indentation alignment, escape sequences within text blocks, how they improve readability of JSON/HTML/SQL strings, and the equivalent Java 13 preview.
30. What are virtual threads?
   - **Answer:** Virtual threads are lightweight threads from Project Loom (Java 21+) that are managed by the JVM, not the OS, allowing millions of concurrent tasks. I am excited to use them for handling high-volume Kafka message processing where each sensor reading can be a virtual thread.
   - **If asked more:** I can explain how virtual threads differ from platform threads, when to use (I/O-heavy, many concurrent tasks) vs avoid (CPU-bound), how they work with synchronized, the Executors.newVirtualThreadPerTaskExecutor(), and Spring Boot 3.2+ virtual thread support.
31. Given a `List<Employee>` where each employee has Name, ID, Salary, and Experience, filter employees based on experience and salary criteria, then print the results in ascending or descending order. Write the Java code.
   - **Answer:** Using the Stream API I filter with a `Predicate` and sort with `Comparator`. For example, to keep only employees with at least 5 years experience and a salary greater than 50,000, sorted by salary ascending:
   ```java
   List<Employee> result = employees.stream()
       .filter(e -> e.getExperience() >= 5)
       .filter(e -> e.getSalary() > 50000)
       .sorted(Comparator.comparing(Employee::getSalary))   // ascending
       .collect(Collectors.toList());
   result.forEach(System.out::println);
   ```
   For descending order, use `.sorted(Comparator.comparing(Employee::getSalary).reversed())` or `.sorted(Comparator.comparingDouble(Employee::getSalary).reversed())`. If I also want to tie-break by experience, I chain comparators with `Comparator.comparing(Employee::getSalary).thenComparing(Employee::getExperience)`.
   - **If asked more:** I can explain that `filter` is an intermediate operation and `collect` is terminal, and that streams are lazy. I can combine both filters into one lambda `e -> e.getExperience() >= 5 && e.getSalary() > 50000`. For null-safe sorting I can use `Comparator.nullsLast()`. I'd also mention the alternative without streams — a plain loop plus `Collections.sort()` with a custom Comparator — and when that is clearer. I can show chaining `sorted()` with `reversed()` for descending and explain that `Comparator.comparing` uses a type-aware comparator, while `Comparator.comparingInt`/`comparingDouble` avoid boxing.

