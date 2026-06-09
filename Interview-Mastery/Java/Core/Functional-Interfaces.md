# Java Functional Interfaces

---

## Overview

- **Definition** — A functional interface is an interface with exactly one abstract method, known as the Single Abstract Method (SAM). They are the foundation of lambda expressions and method references in Java — every lambda implements a functional interface, and the compiler infers which interface a lambda targets based on context.
- **@FunctionalInterface Annotation** — The annotation is optional but strongly recommended — it causes a compile error if a second abstract method is accidentally added, preserving the SAM contract. A functional interface can contain any number of `default` and `static` methods.
- **Before Java 8** — Anonymous inner classes were the only way to pass behavior, requiring verbose boilerplate. Functional interfaces enable the lambda syntax, reducing ceremony and making behavior parameterization practical at scale.
- **Standard Package** — The `java.util.function` package provides 43 standard functional interfaces covering most common use cases, so custom functional interfaces are rarely needed.

  **Why 43?** The 43 interfaces come from a combinatorial design across four dimensions: function shape (Predicate, Consumer, Function, Supplier, UnaryOperator, BinaryOperator, BiPredicate, BiConsumer, BiFunction, plus ToXxx/XxxToXxx bridges), primitive specializations (int, long, double), arity (unary, binary), and source/target type (e.g. `ToIntFunction<T>`, `IntToLongFunction`). The number isn't arbitrary — it covers the intersection of all commonly needed conversion patterns in numeric processing. If you find yourself reaching for a custom interface, check the package first; there's a good chance it already exists under a different name.

```java
@FunctionalInterface
public interface Predicate<T> {
    boolean test(T t);
}
```

---

## Core Functional Interfaces

- **Predicate<T>** — Accepts an argument and returns a `boolean`, used for testing conditions and filtering (`test(T)`).
- **Consumer<T>** — Accepts an argument and returns `void`, used for side effects like logging, printing, or saving (`accept(T)`).
- **Function<T,R>** — Accepts an argument and returns a result of a potentially different type, used for value transformations (`apply(T)`).
- **Supplier<T>** — Takes no arguments and returns a value, used for lazy initialization and factory patterns (`get()`).
- **UnaryOperator<T>** — A specialization of `Function` where the input and output types are the same, commonly used for iterative transformations (`apply(T)`).
- **BinaryOperator<T>** — A specialization of `BiFunction` where both arguments and the return type are the same, used for reduction operations like summing (`apply(T,T)`).

| Interface | Signature | Purpose | Method |
|-----------|-----------|---------|--------|
| `Predicate<T>` | `T → boolean` | Test a condition | `test(T)` |
| `Consumer<T>` | `T → void` | Consume a value (side effect) | `accept(T)` |
| `Function<T,R>` | `T → R` | Transform a value | `apply(T)` |
| `Supplier<T>` | `() → T` | Supply a value (factory) | `get()` |
| `UnaryOperator<T>` | `T → T` | Transform, same type | `apply(T)` |
| `BinaryOperator<T>` | `(T,T) → T` | Combine two values, same type | `apply(T,T)` |

```java
Predicate<String> isEmpty = s -> s.isEmpty();
Consumer<String> logger = s -> System.out.println(s);
Function<String, Integer> parser = s -> Integer.parseInt(s);
Supplier<LocalDate> today = () -> LocalDate.now();
UnaryOperator<String> toUpper = String::toUpperCase;
BinaryOperator<Integer> sum = (a, b) -> a + b;
```

### Bi-variants

- **BiPredicate<T,U>** — `(T,U) → boolean`, tests a condition on two arguments.
- **BiConsumer<T,U>** — `(T,U) → void`, performs a side effect with two arguments.
- **BiFunction<T,U,R>** — `(T,U) → R`, transforms two arguments into a result.
- **Usage** — The bi-variants are essential for operations on map entries, combining two data sources, and callback patterns that need multiple parameters. There is no `BiSupplier` because a supplier by definition takes no arguments, and no `BiUnaryOperator` because a unary operator by definition operates on a single type.

---

## Primitive Specializations

- **Purpose** — Primitive-specialized functional interfaces operate directly on `int`, `long`, and `double` values, completely avoiding the boxing and unboxing overhead inherent in using generic interfaces like `Predicate<Integer>`. A `Predicate<Integer>` autoboxes each primitive `int` to an `Integer` object consuming 16-28 bytes of heap per value.
- **Performance** — The equivalent `IntPredicate` operates on raw `int` values with zero allocation, making it 5-10x faster in performance-critical numeric processing. Each primitive type has specializations for predicates, consumers, functions, suppliers, and operators.
- **Naming Conventions** — Specializations follow naming like `IntPredicate`, `LongConsumer`, `DoubleFunction`, `ToIntFunction`, and `IntToDoubleFunction`. In hot paths processing millions of elements, the difference between `Predicate<Integer>` and `IntPredicate` can be 50ms versus noticeable GC pauses.

  **Why only int, long, double?** These three types cover nearly all numeric processing use cases in server-side applications (counters, timestamps, floating-point calculations). Boolean, byte, short, char, and float are trivially autoboxed with negligible overhead — `Byte` carries the same 16-28 bytes as `Integer` in the heap, but byte-by-byte processing at scale is rare. The JDK designers chose the 80% case: `int` for general counters and indices, `long` for timestamps and large numbers, `double` for decimal math (and `float` is almost never preferred over `double` in business logic).

```java
// Avoid — autoboxing overhead in hot paths
Predicate<Integer> p = x -> x > 5;

// Prefer — no boxing
IntPredicate p = x -> x > 5;
```

| For int | For long | For double |
|---------|----------|------------|
| `IntPredicate` | `LongPredicate` | `DoublePredicate` |
| `IntConsumer` | `LongConsumer` | `DoubleConsumer` |
| `IntFunction<R>` | `LongFunction<R>` | `DoubleFunction<R>` |
| `IntSupplier` | `LongSupplier` | `DoubleSupplier` |
| `IntUnaryOperator` | `LongUnaryOperator` | `DoubleUnaryOperator` |
| `IntBinaryOperator` | `LongBinaryOperator` | `DoubleBinaryOperator` |
| `ToIntFunction<T>` | `ToLongFunction<T>` | `ToDoubleFunction<T>` |

---

## Composition

- **Predicate Composition** — `Predicate` supports `and()`, `or()`, and `negate()` — logical operations that return new `Predicate` instances, enabling concise construction of complex conditions from simple building blocks.
- **Function Composition** — `Function` supports `andThen()` (apply this function first, then the other) and `compose()` (apply the other first, then this one), allowing transformation pipelines that read left-to-right or right-to-left.
- **Consumer Composition** — `Consumer` supports `andThen()` for chaining multiple side-effect operations in sequence, with each consumer receiving the same input value.
- **Lightweight Wrappers** — These composition methods return new functional interface instances that delegate to the originals, meaning the composition is a lightweight wrapper — the original functions are not copied or modified.

```java
// Predicate composition
Predicate<String> isLong = s -> s.length() > 10;
Predicate<String> containsDigit = s -> s.matches(".*\\d.*");
Predicate<String> complex = isLong.and(containsDigit).negate();

// Function composition
Function<String, Integer> parse = Integer::parseInt;
Function<Integer, String> format = Object::toString;
Function<String, String> combined = parse.andThen(format);  // parse then format
// or: format.compose(parse)  // same thing

// Consumer composition
Consumer<String> log = s -> System.out.println("Log: " + s);
Consumer<String> save = s -> db.save(s);
Consumer<String> pipeline = log.andThen(save);
```

---

## Under the Hood: Lambda Compilation

- **invokedynamic Instruction** — Lambdas and method references are compiled using the `invokedynamic` JVM instruction (introduced in Java 7), not as anonymous inner classes. The Java compiler emits an `invokedynamic` call site referencing `LambdaMetafactory` as the bootstrap method.
- **Runtime Generation** — At class load time or first invocation, the bootstrap method executes, generating the lambda's implementation class using `MethodHandle` APIs — not reflection — and linking it into a `CallSite` for permanent caching.
- **Performance Trade-offs** — Non-capturing lambdas produce a single static instance reused forever with zero allocation per invocation. Anonymous inner classes allocate a new object on the heap every time they are instantiated.
- **Memory Benefits** — No separate `.class` file per lambda, no classloader pressure, and captured values are handled through generated bytecode rather than reflection.

```
Source:  list.filter(s -> s.length() > 3)
         ↓
Bytecode: invokedynamic #bootstrapMethod
         ↓
Runtime: LambdaMetafactory.metafactory()
         ↓
         Generates CallSite → Predicate<String> at runtime
```

| Mechanism | Memory | Speed | Notes |
|-----------|--------|-------|-------|
| Anonymous class | ~100 bytes per instance | Fast | Creates .class file |
| Lambda (non-capturing) | ~1 object (static) | Fastest | Single instance, cached |
| Lambda (capturing) | ~1 object per call | Fast | New instance each time |
| Method reference | ~1 object (static) | Fastest | Same as non-capturing lambda |

---

## Common Mistakes

- **Creating Custom Interfaces Unnecessarily** — Creating custom functional interfaces like `Transformer<T,R>` when `Function<T,R>` would suffice introduces incompatibility with standard APIs. A custom type cannot be used directly with `Stream.map()`, `Optional.map()`, or `CompletableFuture.thenApply()` without wrapping.
- **Neglecting Primitive Specializations** — Neglecting primitive specializations in hot paths causes invisible autoboxing overhead. `Predicate<Integer>` boxes every value, while `IntPredicate` avoids allocation entirely — in a stream processing 10 million integers, the difference is 280 MB of garbage versus zero.
- **Checked Exceptions in Standard Interfaces** — Handling checked exceptions inside lambdas using standard functional interfaces is impossible because no standard interface declares checked exceptions. The common workaround of wrapping in try-catch and rethrowing as `RuntimeException` is acceptable, but the original exception is preserved as the cause.
- **Overusing Composition** — Composing too many operations with `andThen()` or `compose()` reduces readability and makes debugging difficult because stack traces from composed functions do not identify which stage failed. Beyond 3-4 compositions, extract to named intermediate variables.
- **Supplier Without Memoization in Hot Paths** — `stream.generate(() -> expensiveLoad())` calls `expensiveLoad()` on every invocation. If the value is stable across the stream, wrap the supplier with memoization: `Supplier<Data> memoized = Suppliers.memoize(() -> expensiveLoad())` (Guava) or a simple lazy initialization holder pattern. Otherwise the same expensive computation runs N times with no caching.
- **Function<T, Boolean> Instead of Predicate<T>** — Both accept T and return a boolean-like value, but `Function<T, Boolean>` autoboxes the result (`boolean` → `Boolean`) and loses access to `and()`, `or()`, and `negate()` — the composition methods that make predicates composable. Prefer `Predicate<T>` whenever the semantics are testing a condition, not transforming a value.

---

## Real-World Scenarios

### Scenario 1: Configurable Validation Pipeline

A user registration system validates input against multiple rules that are loaded from a database and can change without code deployment. Each rule is a `Predicate<String>` that can be combined dynamically, and the rules must short-circuit — if the input is empty, the email regex and breach check should never execute. The system must also return a list of all failed rules for the user to see, not just the first failure.

```java
public class ValidationEngine {
    private final List<Predicate<String>> rules = new ArrayList<>();

    public void addRule(Predicate<String> rule) {
        rules.add(rule);
    }

    public boolean validate(String input) {
        return rules.stream().allMatch(rule -> rule.test(input));
    }

    public List<String> getFailures(String input) {
        return rules.stream()
            .filter(rule -> !rule.test(input))
            .map(rule -> "Validation failed: " + rule)
            .collect(Collectors.toList());
    }
}

// Usage: combine multiple rules
Predicate<String> notEmpty = s -> s != null && !s.trim().isEmpty();
Predicate<String> validEmail = s -> s.matches("^[A-Za-z0-9+_.-]+@(.+)$");
Predicate<String> notPwned = s -> !breachedPasswords.contains(s);

engine.addRule(notEmpty.and(validEmail).and(notPwned));
```

`Predicate.and()` short-circuits naturally — if the input is empty, `validEmail` and `notPwned` are never evaluated, preserving the fail-fast behavior. The `getFailures()` method uses a different terminal operation (`filter` + `collect`) to evaluate all rules and return every failure. Predicate composition allows building complex validation trees from simple, testable building blocks that are loaded dynamically from configuration.

**Why this approach?** A traditional validation framework (like Hibernate Validator or Apache Commons Validator) requires annotations or XML configuration files, creating a compile-time binding between rules and fields. The `Predicate`-based approach makes validation rules first-class objects that can be stored in a database, swapped at runtime, and composed with boolean logic without any code generation or annotation processing. The trade-off is loss of declarative metadata (you cannot inspect a composed predicate to discover which rules it contains) — for scenarios where you need per-rule error messages or rule ordering, a wrapper class holding a `Predicate` plus metadata string is better: `record ValidationRule(String name, Predicate<String> condition, String message) {}`.

### Scenario 2: Pluggable Cache Loading Strategy

A caching layer supports different loading strategies — load from a database, fetch from a remote API, or compute on the fly. Each strategy is a `Supplier<Data>` that the cache invokes when a key is missing. The strategy is chosen at configuration time, and the cache must guarantee that the supplier runs at most once per key even under concurrent access.

```java
public class CacheManager<K, V> {
    private final ConcurrentHashMap<K, V> cache = new ConcurrentHashMap<>();
    private final Supplier<V> defaultValue;

    public CacheManager(Supplier<V> defaultValue) {
        this.defaultValue = defaultValue;
    }

    public V get(K key) {
        return cache.computeIfAbsent(key, k -> defaultValue.get());
    }
}

// Different strategies as Suppliers
Supplier<Data> dbLoader = () -> database.load(key);
Supplier<Data> apiLoader = () -> restClient.fetch("/data/" + key);
Supplier<Data> computed = () -> expensiveComputation(key);

new CacheManager<>(dbLoader);
new CacheManager<>(apiLoader);
```

The `Supplier<T>` abstracts the loading mechanism entirely — the cache manager has no knowledge of whether it is loading from a database, a remote API, or computing from scratch. Each supplier captures the specific dependencies it needs (database connection, REST client, computation parameters) and can be unit tested independently. The `computeIfAbsent` method guarantees the supplier runs at most once per key across all concurrent threads, making this pattern safe for high-concurrency environments.

**Why this approach?** A traditional cache using an abstract `load()` method (Template Method pattern) would require a subclass per loading strategy, creating a `.class` file per strategy and a classloader binding at compile time. The `Supplier<T>` approach makes the loading strategy a constructor parameter — the cache manager is closed for modification, open for extension. The `computeIfAbsent` guarantee is critical: without it, two concurrent threads calling `get(missingKey)` would both invoke the supplier, wasting resources and potentially duplicating work or creating race conditions on the backing store.

### Scenario 3: Event Processing with Consumer Chain

A monitoring system receives raw log events and processes them through a multistage pipeline: parse raw data, enrich with metadata, filter low-severity events, persist to the database, and send alerts for critical events. Each stage is a `Consumer<Event>` that can be composed into a single pipeline, and stages should be independently testable.

```java
public class EventPipeline {
    private Consumer<Event> pipeline;

    public EventPipeline() {
        Consumer<Event> parse = event -> event.setParsed(parseRaw(event.getRaw()));
        Consumer<Event> enrich = event -> event.setMetadata(lookupMetadata(event));
        Consumer<Event> filter = event -> { if (event.getSeverity() < 5) event.setSuppressed(true); };
        Consumer<Event> persist = event -> { if (!event.isSuppressed()) database.save(event); };
        Consumer<Event> alert = event -> { if (event.getSeverity() >= 8) alertService.send(event); };

        this.pipeline = parse.andThen(enrich).andThen(filter).andThen(persist).andThen(alert);
    }

    public void process(Event event) {
        pipeline.accept(event);
    }
}
```

`Consumer.andThen()` creates a composed consumer that executes all stages in order for each event. Each stage is a focused, independently testable consumer — you can test `parse` with a mock event and verify the parsed fields without involving the database. The `filter` stage uses `Consumer` as a side-effect operation by setting a `suppressed` flag on the event, which is appropriate here because we are mutating event state rather than transforming values. The pipeline introduces O(1) overhead regardless of the number of stages because `andThen()` creates a lightweight delegating wrapper, not a copy of the stages.

**Why this approach?** A traditional event pipeline using a `List<Consumer<Event>>` with a for-loop would achieve the same result with more code and no composition readability benefit — the loop obscures the ordering guarantee that `andThen()` makes explicit. The `Consumer` chain is the right abstraction because every stage mutates the event in place (setting parsed data, suppression flag, etc.). If stages needed to produce new event objects from old ones, `Function<Event, Event>` would be more appropriate, avoiding mutable state entirely. Choosing `Consumer` over `Function` here signals that the pipeline is a side-effect chain, not a transformation pipeline.

---

## Scenario-Based Questions

**Q: You are designing an API where users pass filtering logic as a `Predicate<T>`. Some users pass lambdas that throw checked exceptions (e.g., a database lookup in the predicate). The current `Predicate<T>` interface doesn't support checked exceptions. How do you design a filtering API that handles checked exceptions without forcing all callers to write try-catch blocks?**

A: Create a custom `ThrowingPredicate<T>` interface that declares `throws Exception` on its single abstract method, then provide an adapter method that wraps the throwing predicate into a standard `Predicate<T>`:
```java
@FunctionalInterface
public interface ThrowingPredicate<T> {
    boolean test(T t) throws Exception;
}

public <T> Stream<T> filter(Stream<T> stream, ThrowingPredicate<T> predicate) {
    return stream.filter(t -> {
        try { return predicate.test(t); }
        catch (Exception e) { throw new RuntimeException(e); }
    });
}
```
Callers who do not need checked exceptions use standard lambdas that are compatible with `ThrowingPredicate` via lambda type inference. Callers who need to call a checked-exception-throwing method write the same lambda without any try-catch boilerplate — the wrapping happens inside the API method. For callers who need fine-grained exception handling, provide an overload that accepts both a `ThrowingPredicate<T>` and a `Consumer<Exception>` error handler.

**Q: A method accepts `Consumer<String>` for logging. In production, the consumer writes to a file. In tests, the consumer captures messages for assertion. A developer accidentally passes a consumer that blocks indefinitely on the first message. How do you make this API safe?**

A: Wrap the consumer with a decorator that enforces a timeout on each `accept()` call, so that a blocking consumer cannot hang the calling thread indefinitely:
```java
public static <T> Consumer<T> withTimeout(Consumer<T> delegate, long timeout, TimeUnit unit) {
    return t -> {
        var future = CompletableFuture.runAsync(() -> delegate.accept(t));
        try { future.get(timeout, unit); }
        catch (TimeoutException e) { future.cancel(true); throw new RuntimeException("Consumer timed out", e); }
    };
}
```
For production use, the API should never blindly trust externally provided consumers — always wrap them with error handling and timeout enforcement using `CompletableFuture.orTimeout()` or a dedicated `ScheduledExecutorService`. A blocking consumer can hang a thread in the thread pool indefinitely, eventually exhausting the pool and causing cascading failures across the application.

**Q: A configuration service returns `Optional<Config>`. Multiple callers use `.orElseGet(() -> loadDefaultConfig())` for fallback. The `loadDefaultConfig()` is expensive (reads a file). Some callers also need to differentiate between "config not found" and "error loading default". How do you design a `Supplier`-based fallback that handles both cases?**

A: Create a sealed `Fallback<T>` hierarchy that explicitly models the three possible outcomes — a value, a fallback produced from a supplier, or an error with the captured exception:
```java
public sealed interface Fallback<T> permits Value, Error, Lazy {
    record Value<T>(T value) implements Fallback<T> {}
    record Error<T>(Exception exception) implements Fallback<T> {}
    record Lazy<T>(Supplier<T> supplier) implements Fallback<T> {}
}

public Fallback<Config> getConfig(String key) {
    Optional<Config> config = configRepo.find(key);
    if (config.isPresent()) return new Fallback.Value<>(config.get());
    try {
        return new Fallback.Value<>(loadDefaultConfig());
    } catch (Exception e) {
        return new Fallback.Error<>(e);
    }
}
```
The pure `Supplier` approach loses error semantics — a supplier that throws is indistinguishable from a supplier that returns null or an empty optional. The sealed interface gives callers pattern-matching capability with `switch` expressions in Java 21+: `switch (result) { case Value v -> ...; case Error e -> log.warn("Fallback failed", e.exception()); }`. The `Lazy` variant wraps a `Supplier` for deferred evaluation, giving the caller control over when the fallback is computed.

**Q: You have a `Function<String, String>` that normalizes text (lowercase, trim, remove accents). This function is passed to 20 different stream pipelines. A developer modifies the function to also strip HTML tags, which breaks 15 of the 20 pipelines. How do you prevent this with functional interfaces?**

A: Use distinct functional interface types for semantically different operations, even when they share the same type signature. Create extended interfaces that carry semantic meaning:
```java
@FunctionalInterface interface Normalizer extends Function<String, String> {}
@FunctionalInterface interface Sanitizer extends Function<String, String> {}

Normalizer normalizer = s -> s.toLowerCase().trim();
Sanitizer sanitizer = s -> s.replaceAll("<[^>]*>", "");

// Now method signatures carry intent:
public String process(String input, Normalizer normalizer) { ... }
public String render(String input, Sanitizer sanitizer) { ... }
```
The distinct types prevent accidentally passing a sanitizer where a normalizer is expected, because the compiler enforces the type distinction. Extending `Function<String, String>` preserves full interoperability with standard stream APIs. This follows the "make illegal states unrepresentable" principle — the type system encodes the semantic distinction rather than relying on documentation or naming conventions.

**Q: A `BinaryOperator<BigDecimal>` is used in a `reduce()` operation. Parallel execution gives different results than sequential execution. The operation is `(a, b) -> a.multiply(b).setScale(2, RoundingMode.HALF_UP)`. What is wrong?**

A: The intermediate rounding makes this operation non-associative — `round(round(a × b) × c) ≠ round(a × round(b × c))` in general due to precision loss at each intermediate step. `reduce()` requires the combiner to be associative, and when the stream is parallelized, the JVM partitions the data and applies the reduction in an arbitrary tree structure, exposing the non-associativity:
```java
// Bad — non-associative due to intermediate rounding
BinaryOperator<BigDecimal> bad = (a, b) -> a.multiply(b).setScale(2, HALF_UP);

// Good — associative, round at the end
BinaryOperator<BigDecimal> good = BigDecimal::multiply;

BigDecimal total = amounts.stream()
    .reduce(BigDecimal.ONE, good)
    .setScale(2, HALF_UP);
```
The broader lesson: any `BinaryOperator` with state, rounding, truncation, or side effects is likely non-associative and will produce non-deterministic results in parallel streams. Test with `op(op(a, b), c) == op(a, op(b, c))` for all inputs to verify associativity.

**Q: A method returns `Supplier<Connection>` that lazily creates database connections. The supplier is stored in a static map and reused across requests. Over time, connections are never closed — the supplier creates a new one each time `get()` is called. How do you design a `Supplier` that manages resource lifecycle?**

A: `Supplier<T>` is designed for value production, not resource management, and has no concept of lifecycle or cleanup. Do not use `Supplier` for resources that must be explicitly closed. Instead, use the execute-around pattern with `Function<Connection, R>` that guarantees the resource is closed in a finally block:
```java
public <T> T withConnection(Function<Connection, T> callback) {
    try (Connection conn = createConnection()) {
        return callback.apply(conn);
    }
}
```
If you truly need a supplier-like API for closeable resources, create a `ManagedSupplier<T extends AutoCloseable>` that tracks all created resources and implements `AutoCloseable` itself. The broader lesson: choose the abstraction that matches the lifecycle. `Supplier<T>` is appropriate for stateless value production; `Function<T, R>` with execute-around is appropriate for resources that need cleanup.

**Q: A rate limiter accepts `Runnable` tasks and throttles execution to 10 QPS. A developer submits `() -> database.query("DELETE FROM users")` — the lambda captures a dangerous SQL string and executes it later. How do you design the API to make destructive operations more visible?**

A: Use distinct functional interface types for read versus write operations, even though both have the same method signature of `() → void`. The type distinction forces callers to explicitly choose which operation category they intend:
```java
@FunctionalInterface interface ReadOperation { void execute(); }
@FunctionalInterface interface WriteOperation { void execute(); }

void submitRead(ReadOperation op);
void submitWrite(WriteOperation op);

// Usage:
rateLimiter.submitWrite(() -> database.delete("users")); // Explicit: this is a write
rateLimiter.submitRead(() -> database.query("SELECT *")); // Explicit: this is a read
```
The two interface types have identical method signatures but encode different semantics in the type system. The rate limiter can apply different throttling policies, queuing strategies, and error handling for reads versus writes. This pattern — sometimes called phantom types or type-safe tagging — uses functional interfaces to encode semantic intent that the compiler enforces.

**Q: A `UnaryOperator<BigDecimal>` that applies tax is passed to a processing pipeline. The tax rate changes daily. The operator is created once at startup and cached — it always uses the old tax rate. How do you make it dynamic without recreating the operator?**

A: Rather than capturing the tax rate as a value at creation time, capture a `Supplier<BigDecimal>` that reads the current rate every time the operator is applied:
```java
// Static — never updates
UnaryOperator<BigDecimal> staticTax = price -> price.multiply(BigDecimal.valueOf(0.2));

// Dynamic — reads current rate every time
Supplier<BigDecimal> taxRateSupplier = () -> configService.getTaxRate();
UnaryOperator<BigDecimal> dynamicTax = price -> price.multiply(taxRateSupplier.get());
```
The `dynamicTax` operator captures a `Supplier`, not a fixed value, so each invocation calls `taxRateSupplier.get()` to fetch the latest rate from the configuration service. The `Supplier` abstraction decouples the "what to do" (apply tax) from the "what value to use" (the current rate). For performance-sensitive scenarios, wrap the supplier with time-based caching: `Supplier<BigDecimal> cachedTax = Suppliers.memoizeWithDuration(taxRateSupplier, Duration.ofHours(1))` from a library like Guava.

**Q: You have a chain `Function<A, B>.andThen(Function<B, C>).andThen(Function<C, D>)`. One of the functions throws NullPointerException intermittently. Stack traces only show the outer caller, not which function in the chain failed. How do you debug composition chains?**

A: Create a tracing wrapper decorator that captures the function's identity and the input value when an exception occurs:
```java
public static <T, R> Function<T, R> traced(String name, Function<T, R> fn) {
    return t -> {
        try {
            return fn.apply(t);
        } catch (Exception e) {
            throw new RuntimeException("Function '" + name + "' failed with input: " + t, e);
        }
    };
}

Function<A, B> step1 = traced("parse", this::parse);
Function<B, C> step2 = traced("validate", this::validate);
Function<C, D> step3 = traced("enrich", this::enrich);

Function<A, D> pipeline = step1.andThen(step2).andThen(step3);
```
The traced wrapper preserves the original exception as the cause and adds contextual information about which function name and input caused the failure. Without tracing, a composition chain of five or more functions becomes nearly impossible to debug because the stack trace from a composed `Function` does not include which stage threw the exception. For production, consider AOP-based instrumentation or structured logging that records the pipeline stage with each invocation.

**Q: A developer declares an interface `@FunctionalInterface interface Action { void execute(); }`. They add `boolean equals(Object obj)`, `int hashCode()`, and `String toString()` as default methods. Is it still a functional interface?**

A: Yes — `Object` methods like `equals`, `hashCode`, and `toString` do not count when counting abstract methods for the SAM rule, even if declared explicitly in the interface. The JLS specifies that any public method declared in `Object` is excluded from the single-abstract-method count. This is why `Comparator<T>` is a functional interface despite declaring `boolean equals(Object obj)` alongside `int compare(T o1, T o2)` — `equals` is from Object and is ignored. However, if `Action` adds a non-Object abstract method like `void cleanup()`, it would no longer be a functional interface.

**Q: A class overloads `void process(Predicate<String>)` and `void process(Function<String, Boolean>)`. Calling `process(s -> s.isEmpty())` fails to compile with an ambiguous reference error. Why?**

A: Both `Predicate<String>` and `Function<String, Boolean>` have the same erased signature `(String) -> Object`, and the lambda body `s -> s.isEmpty()` matches both — it returns a `boolean` which can be autoboxed to `Boolean` for the `Function` variant. The compiler cannot decide which overload applies because both are equally specific. The fix is to avoid overloading methods with different functional interfaces that share the same shape (T → boolean-like). Use distinct method names: `void filter(Predicate<String>)` and `void transform(Function<String, Boolean>)`. This is a design-time constraint: functional interface overloading works only when the shapes are clearly different (e.g., `Consumer` vs `Supplier`).

**Q: A `BiConsumer<HttpRequest, HttpResponse>` is used as a middleware handler. The first middleware modifies the request, passes it to the next, and modifies the response. The handler is: `(req, res) -> { audit.log(req); next.accept(req, res); encrypt(res); }`. What concurrency issues arise with this `BiConsumer` chain?**

A: The `BiConsumer` captures `next` and `audit` via `this`, and when multiple requests are processed concurrently, `encrypt(res)` may modify the response while the next middleware is still reading or writing to it, creating a data race. The shared mutable `req` and `res` references violate the principle that each stage should produce new values rather than mutating shared state. Fix this by using `Function<HttpRequest, CompletableFuture<HttpResponse>>` where each stage produces a new, immutable response rather than mutating a shared reference:
```java
Function<HttpRequest, CompletableFuture<HttpResponse>> handler = req ->
    auditAsync(req)
        .thenCompose(v -> next.apply(req))
        .thenApply(res -> encrypt(res));
```
In this functional pipeline, `encrypt()` receives the response from the previous stage and produces a new encrypted response, leaving the original untouched. The `CompletableFuture` chaining provides thread isolation and backpressure naturally. The `BiConsumer` side-effect pattern is inherently problematic for concurrent middleware pipelines because it assumes single-threaded, sequential execution.

---

## Interview Questions

**What is a functional interface in Java?** A functional interface is an interface with exactly one abstract method (SAM — Single Abstract Method). It may contain any number of `default` and `static` methods without affecting its functional status. The `@FunctionalInterface` annotation is optional but recommended — it causes a compile error if a second abstract method is added. Functional interfaces are the compilation target for lambda expressions and method references.

**What are the core functional interfaces in `java.util.function`?** The six core interfaces are `Predicate<T>` (T → boolean), `Consumer<T>` (T → void), `Function<T,R>` (T → R), `Supplier<T>` (() → T), `UnaryOperator<T>` (T → T), and `BinaryOperator<T>` (T,T → T). Each has bi-variants: `BiPredicate<T,U>`, `BiConsumer<T,U>`, and `BiFunction<T,U,R>`. The package contains 43 total interfaces including primitive specializations.

**What is the difference between `Consumer<T>` and `Function<T, R>`?** `Consumer<T>` accepts a single argument and returns no result, making it suitable for side-effect operations like logging, printing, sending notifications, or mutating object state. `Function<T, R>` accepts an argument and returns a result, making it suitable for pure transformations. `Consumer` supports composition via `andThen()`, while `Function` supports both `andThen()` (left-to-right) and `compose()` (right-to-left).

**What is `Supplier<T>` used for?** `Supplier<T>` takes no arguments and returns a value. Common use cases include lazy evaluation in `Optional.orElseGet(() -> expensiveLoad())`, factory patterns where the caller controls when an object is created, stream generation via `Stream.generate(supplier)`, and deferring computation to a different thread or execution context. Suppliers should typically be idempotent or at least side-effect-free.

**What is the difference between `andThen()` and `compose()` in `Function`?** `f.andThen(g)` applies `f` first and then `g`, equivalent to `g(f(x))`. `f.compose(g)` applies `g` first and then `f`, equivalent to `f(g(x))`. `andThen()` reads left-to-right (natural for method chaining), while `compose()` reads right-to-left (natural for mathematical notation). In practice, `andThen()` is far more common in Java codebases.

**Why are primitive specializations like `IntPredicate` important?** Primitive specializations eliminate autoboxing overhead — `IntPredicate` operates on raw `int` values (4 bytes each), while `Predicate<Integer>` wraps each value in an `Integer` object (16-28 bytes on the heap). For pipelines processing millions of values, the boxed variant creates megabytes of garbage that trigger GC pauses. The difference between `sum()` on `IntStream` versus `reduce()` on `Stream<Integer>` can be 5-10x in throughput.

**Can a functional interface extend another functional interface?** Yes, as long as the child interface does not add a new abstract method. If the parent and child both declare the same single abstract method signature, the child remains a functional interface. This is how to create type-safe aliases: `@FunctionalInterface interface Transformer<T,R> extends Function<T,R> {}`. If the child declares any new abstract method beyond the parent's SAM, the child is no longer a functional interface and will be flagged by `@FunctionalInterface`.

**How do functional interfaces enable method references?** Method references (`String::length`, `Integer::parseInt`) are compiled to the same `invokedynamic` instruction as lambdas. The compiler resolves the method reference's signature against the target functional interface's SAM — for `Function<String, Integer>`, `String::length` matches because `length()` takes no arguments and returns `int`. If the method signature does not match, the error is caught at compile time, not at runtime.

**What is the relationship between `Comparator<T>` and functional interfaces?** `Comparator<T>` is a functional interface with SAM `int compare(T o1, T o2)`. It has many `default` methods like `reversed()`, `thenComparing()`, and `thenComparingInt()` for fluent comparator construction, and `static` methods like `comparing()`, `naturalOrder()`, and `nullsFirst()`. These composition methods return new `Comparator` instances, enabling declarative sorting: `Comparator.comparing(Person::age).thenComparing(Person::name).reversed()`.

**How do you create a custom functional interface correctly?** Annotate with `@FunctionalInterface`, ensure exactly one abstract method, and strongly consider extending a standard interface for interoperability. Standard functional interfaces should be preferred over custom ones in most cases — create a custom interface only when necessary for checked exceptions, multiple parameters with meaningful names, or type-safe semantic tagging. Example: `@FunctionalInterface interface ThrowingFunction<T,R> { R apply(T t) throws Exception; }`.

---

## Developer Recommendations

- **Prefer standard functional interfaces over custom ones** — The `java.util.function` package provides 43 interfaces covering the vast majority of lambda use cases. Creating custom interfaces like `Transformer` or `Mapper` introduces incompatibility with standard APIs — `Stream.map()`, `Optional.map()`, and `CompletableFuture.thenApply()` all accept `Function<T,R>`. If you need semantic clarity, extend the standard interface: `interface Transformer<T,R> extends Function<T,R> {}`.
- **Use primitive specializations in performance-sensitive numeric code** — `IntPredicate` avoids boxing one million integers, while `Predicate<Integer>` creates 28 bytes of garbage per element through autoboxing. For a pipeline processing 10 million transactions, that is 280 MB of unnecessary allocation pressure. The primitive variants are typically 5-10x faster for numeric operations.
- **Compose Predicate with and(), or(), and negate()** — Composition methods are more readable, naturally short-circuiting, and individually testable compared to manual logical expressions. `predicate.and(other).negate()` clearly communicates "not (this and that)" without the mental parsing needed for `x -> predicate.test(x) && !other.test(x)`. Extract complex composed predicates to named fields for documentation and reuse.
- **Use Consumer exclusively for side effects, Function for transformations, Supplier for lazy values** — Each functional interface encodes intent in its type signature. A method accepting `Consumer<T>` should perform actions with side effects; a method accepting `Function<T,R>` should transform without side effects. Violating this convention violates the principle of least surprise.
- **Avoid checked exceptions in functional interfaces** — Create a single `Unchecked` utility rather than a proliferation of custom interfaces for each exception type. One utility method `Function<T,R> unchecked(ThrowingFunction<T,R> fn)` covers all cases without requiring `ThrowingFunction`, `ThrowingPredicate`, `ThrowingConsumer`, and so on.
- **Use BinaryOperator<T> over BiFunction<T,T,T>** — `BinaryOperator` additionally provides `minBy()` and `maxBy()` static methods and more clearly communicates reduction semantics. Similarly, prefer `UnaryOperator<T>` over `Function<T,T>` for same-type transformations because it provides `identity()` for no-op transformations.
- **Test functional interfaces as you would any other behavior** — A lambda is an anonymous implementation; test it by exercising the functional interface through its SAM. For inline lambdas that are passed to methods, extract them to fields or factory methods so they can be unit tested: `private static final Predicate<String> IS_EMPTY = String::isEmpty;`. For composed predicates or functions, test each building block individually and then test the composition — a failure in a 5-way `andThen` chain is hard to attribute without isolated unit tests for each component. Mock libraries like Mockito can stub `Function.apply()` for integration tests, but prefer real lambda implementations in unit tests to catch logic errors.
