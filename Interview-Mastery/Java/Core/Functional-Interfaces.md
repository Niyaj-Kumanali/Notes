# Java Functional Interfaces

---

## Overview

- **Definition:** A functional interface is an interface with exactly **one abstract method** (SAM — Single Abstract Method). They are the foundation of lambda expressions and method references in Java.

- **Why It Exists:** Before Java 8, anonymous inner classes were the only way to pass behavior:
  ```java
  button.addActionListener(new ActionListener() {
      @Override
      public void actionPerformed(ActionEvent e) {
          System.out.println("Clicked!");
      }
  });
  ```
  Functional interfaces enable lambda syntax:
  ```java
  button.addActionListener(e -> System.out.println("Clicked!"));
  ```

- **The @FunctionalInterface Annotation:**
  - Optional but recommended — makes intent clear
  - Compilation failure if a second abstract method is added
  - ```java
    @FunctionalInterface
    public interface Predicate<T> {
        boolean test(T t);
    }
    ```

- **Default and Static Methods:** A functional interface can have `default` and `static` methods — only the single abstract method matters for the SAM contract. `Comparator` is a good example with many default and static methods but one abstract method (`compare`).

---

## Core Functional Interfaces

- **Definition:** The `java.util.function` package provides 43 functional interfaces. The six core ones cover most use cases.

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

- **BiPredicate<T,U>:** `(T,U) → boolean`
- **BiConsumer<T,U>:** `(T,U) → void`
- **BiFunction<T,U,R>:** `(T,U) → R`

---

## Primitive Specializations

- **Definition:** Specialized functional interfaces that operate on primitives directly to avoid autoboxing overhead.

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

- **Definition:** Functional interfaces provide default methods for composing multiple operations together.

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

- **Definition:** Lambdas are NOT compiled as anonymous inner classes. Instead, Java uses `invokedynamic` (JVM instruction added in Java 7).

```
Source:  list.filter(s -> s.length() > 3)
         ↓
Bytecode: invokedynamic #bootstrapMethod
         ↓
Runtime: LambdaMetafactory.metafactory()
         ↓
         Generates CallSite → Predicate<String> at runtime
```

- **Benefits:**
  - No separate .class file per lambda
  - Lambda is generated once and cached
  - Lower memory footprint than anonymous classes
  - Captured variables use `MethodHandle` not reflection

| Mechanism | Memory | Speed | Notes |
|-----------|--------|-------|-------|
| Anonymous class | ~100 bytes per instance | Fast | Creates .class file |
| Lambda (non-capturing) | ~1 object (static) | Fastest | Single instance, cached |
| Lambda (capturing) | ~1 object per call | Fast | New instance each time |
| Method reference | ~1 object (static) | Fastest | Same as non-capturing lambda |

---

## Common Mistakes

- **Creating custom functional interfaces when standard ones suffice** — often `Consumer<T>` or `Function<T,R>` replaces a custom interface
- **Not using primitive specializations in hot paths** — autoboxing penalty with `Predicate<Integer>` vs `IntPredicate`
- **Checked exceptions in lambdas** — no standard functional interface declares checked exceptions; requires wrapper
- **Variable capture with mutable state** — captured variables must be effectively final
- **Lambda serialization** — lambdas are not serializable by default; requires cast to `Serializable`
- **Ambiguous method overloads** — overloaded methods accepting different functional interfaces cause compilation ambiguity
- **Overusing composition** — long chains of `.andThen()` can reduce readability

---

## Real-World Scenarios

### Scenario 1: Configurable Validation Pipeline

A user registration system must validate input against multiple rules. Each rule is a `Predicate<String>` that can be combined dynamically. Rules are loaded from a database and can change without code deployment.

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

`Predicate.and()` short-circuits — if input is empty, the email regex and breach check never run. `Predicate` composition allows building complex validation trees from simple, testable building blocks. The rules are stored in a `List<Predicate<String>>` so new rules can be added without modifying the engine.

### Scenario 2: Pluggable Cache Loading Strategy

A caching layer needs different loading strategies: load from database, load from remote API, or compute on the fly. Each strategy is a `Supplier<Data>` that the cache uses when a key is missing. The strategy is chosen at configuration time.

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

The `Supplier<T>` abstracts the loading mechanism. The cache manager doesn't know whether it's loading from DB, API, or computation. Each supplier captures the dependencies it needs (database connection, REST client) and can be tested independently. The `computeIfAbsent` method guarantees the supplier runs at most once per key.

### Scenario 3: Event Processing with Consumer Chain

A monitoring system receives raw log events, processes them through a pipeline (parse, enrich, filter, persist), and sends alerts. Each stage is a `Consumer<Event>` that can be composed.

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

`Consumer.andThen()` creates a composed consumer that executes stages in order. Each stage is focused and testable. The `filter` stage uses `Consumer` as a side-effect operation (setting `suppressed` flag) — this is appropriate because we're mutating event state. The pipeline is O(1) overhead regardless of the number of stages.

---

## Scenario-Based Questions

1. **Q: You are designing an API where users pass filtering logic as a `Predicate<T>`. Some users pass lambdas that throw checked exceptions (e.g., a database lookup in the predicate). The current `Predicate<T>` interface doesn't support checked exceptions. How do you design a filtering API that handles checked exceptions without forcing all callers to write try-catch blocks?**
   A: Create a custom `ThrowingPredicate<T>` and provide a bridge method:
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
   Callers who don't need checked exceptions use standard lambdas (auto-boxed to `ThrowingPredicate` via lambda compatibility). Callers who need checked exceptions write the same lambda — the wrapping happens inside the API. For callers who need detailed exception handling, provide an overload that accepts `Predicate<T>` and `Consumer<Exception>`.

2. **Q: A method accepts `Consumer<String>` for logging. In production, the consumer writes to a file. In tests, the consumer captures messages for assertion. A developer accidentally passes a consumer that blocks indefinitely on the first message. How do you make this API safe?**
   A: Wrap the consumer with a timeout decorator:
   ```java
   public static <T> Consumer<T> withTimeout(Consumer<T> delegate, long timeout, TimeUnit unit) {
       return t -> {
           var future = CompletableFuture.runAsync(() -> delegate.accept(t));
           try { future.get(timeout, unit); }
           catch (TimeoutException e) { future.cancel(true); throw new RuntimeException("Consumer timed out", e); }
       };
   }
   ```
   Better: the API should accept `Consumer<String>` but execute it in a managed context with a timeout. For production, always wrap third-party consumers with error handling and timeouts. A blocking consumer can hang a thread in the pool — use `CompletableFuture.orTimeout()` or a `ScheduledExecutorService` to enforce deadlines.

3. **Q: A configuration service returns `Optional<Config>`. Multiple callers use `.orElseGet(() -> loadDefaultConfig())` for fallback. The `loadDefaultConfig()` is expensive (reads a file). Some callers also need to differentiate between "config not found" and "error loading default". How do you design a `Supplier`-based fallback that handles both cases?**
   A: Create a richer `Fallback<T>` abstraction:
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
   The pure `Supplier` approach loses error semantics. The `Fallback` sealed interface gives callers pattern-matching capability: `switch (result) { case Value v -> ...; case Error e -> ...; }`. The `Lazy` variant wraps `Supplier` for deferred evaluation. This is more explicit than `Optional.orElseGet()` and handles the 3-state outcome (value, fallback, error).

4. **Q: You have a `Function<String, String>` that normalizes text (lowercase, trim, remove accents). This function is passed to 20 different stream pipelines. A developer modifies the function to also strip HTML tags, which breaks 15 of the 20 pipelines. How do you prevent this with functional interfaces?**
   A: Use distinct functional interface types for distinct operations — don't reuse `Function<String, String>` for different semantic meanings:
   ```java
   // Instead of:
   Function<String, String> normalize = s -> s.toLowerCase().trim();
   Function<String, String> sanitize = s -> s.replaceAll("<[^>]*>", "");

   // Use:
   @FunctionalInterface interface Normalizer extends Function<String, String> {}
   @FunctionalInterface interface Sanitizer extends Function<String, String> {}

   Normalizer normalizer = s -> s.toLowerCase().trim();
   Sanitizer sanitizer = s -> s.replaceAll("<[^>]*>", "");

   // Now method signatures carry intent:
   public String process(String input, Normalizer normalizer) { ... }
   public String render(String input, Sanitizer sanitizer) { ... }
   ```
   The distinct types prevent accidentally passing a sanitizer where a normalizer is expected. The `extends Function<String, String>` ensures interoperability with stream APIs. This follows the "make illegal states unrepresentable" principle.

5. **Q: A `BinaryOperator<BigDecimal>` is used in a `reduce()` operation. Parallel execution gives different results than sequential execution. The operation is `(a, b) -> a.multiply(b).setScale(2, RoundingMode.HALF_UP)`. What is wrong?**
   A: The rounding mode makes this operation non-associative. `round(round(a × b) × c) ≠ round(a × round(b × c))` in general due to intermediate rounding. Fix: perform the reduction without rounding, then round at the end:
   ```java
   // Bad — non-associative due to intermediate rounding
   BinaryOperator<BigDecimal> bad = (a, b) -> a.multiply(b).setScale(2, HALF_UP);

   // Good — associative, round at the end
   BinaryOperator<BigDecimal> good = BigDecimal::multiply;

   BigDecimal total = amounts.stream()
       .reduce(BigDecimal.ONE, good)
       .setScale(2, HALF_UP);
   ```
   The broader lesson: any `BinaryOperator` or combiner function with state, rounding, or truncation is likely non-associative. Test with: `op(op(a, b), c) == op(a, op(b, c))` for all inputs. If not, the operation is unsuitable for parallel streams.

6. **Q: A method returns `Supplier<Connection>` that lazily creates database connections. The supplier is stored in a static map and reused across requests. Over time, connections are never closed — the supplier creates a new one each time `get()` is called. How do you design a `Supplier` that manages resource lifecycle?**
   A: Don't use `Supplier` for resources that need cleanup. Instead, use `AutoCloseable` with try-with-resources:
   ```java
   // Bad — Supplier creates but never closes
   Supplier<Connection> connSupplier = () -> createConnection();

   // Good — use try-with-resources pattern
   public <T> T withConnection(Function<Connection, T> callback) {
       try (Connection conn = createConnection()) {
           return callback.apply(conn);
       }
   }
   ```
   If you must use `Supplier`, wrap it in a managed supplier that tracks and closes resources:
   ```java
   public class ManagedSupplier<T extends AutoCloseable> implements Supplier<T>, AutoCloseable {
       private final List<T> resources = new ArrayList<>();
       private final Supplier<T> delegate;
       public ManagedSupplier(Supplier<T> delegate) { this.delegate = delegate; }
       @Override public T get() { T resource = delegate.get(); resources.add(resource); return resource; }
       @Override public void close() { resources.forEach(this::closeQuietly); }
   }
   ```
   The lesson: `Supplier<T>` is for value production, not resource management. If lifecycle matters, use `Function<Consumer<T>, R>` or the execute-around pattern.

7. **Q: A rate limiter accepts `Runnable` tasks and throttles execution to 10 QPS. A developer submits `() -> database.query("DELETE FROM users")` — the lambda captures a dangerous SQL string and executes it later. How do you design the API to make destructive operations more visible?**
   A: Use distinct functional interfaces for read vs write operations:
   ```java
   // Instead of:
   void submit(Runnable task); // Both reads and writes look the same

   // Use:
   @FunctionalInterface interface ReadOperation { void execute(); }
   @FunctionalInterface interface WriteOperation { void execute(); }

   void submitRead(ReadOperation op);
   void submitWrite(WriteOperation op);
   ```
   This forces callers to think about the operation type:
   ```java
   rateLimiter.submitWrite(() -> database.delete("users")); // Explicit: this is a write
   rateLimiter.submitRead(() -> database.query("SELECT *")); // Explicit: this is a read
   ```
   The two interface types have the same signature but different semantics. The rate limiter can apply different throttling, queuing, and error handling for reads vs writes. This pattern (phantom types / type-safe tagging) uses functional interfaces to encode semantic intent in the type system.

8. **Q: A `UnaryOperator<BigDecimal>` that applies tax is passed to a processing pipeline. The tax rate changes daily. The operator is created once at startup and cached — it always uses the old tax rate. How do you make it dynamic without recreating the operator?**
   A: Use a `Supplier<BigDecimal>` for the tax rate inside the `UnaryOperator`:
   ```java
   // Static — never updates
   UnaryOperator<BigDecimal> staticTax = price -> price.multiply(BigDecimal.valueOf(0.2));

   // Dynamic — reads current rate every time
   Supplier<BigDecimal> taxRateSupplier = () -> configService.getTaxRate();
   UnaryOperator<BigDecimal> dynamicTax = price -> price.multiply(taxRateSupplier.get());
   ```
   The `dynamicTax` operator captures a `Supplier`, not a value. Each invocation calls `taxRateSupplier.get()` to get the current rate. The `Supplier` abstraction decouples the "what" (apply tax) from the "when" (which rate). For caching with TTL, wrap the supplier: `Supplier<BigDecimal> cachedTax = Suppliers.memoizeWithDuration(taxRateSupplier, Duration.ofHours(1))`.

9. **Q: You have a chain `Function<A, B>.andThen(Function<B, C>).andThen(Function<C, D>)`. One of the functions throws NullPointerException intermittently. Stack traces only show the outer caller, not which function in the chain failed. How do you debug composition chains?**
   A: Create a debugging wrapper that captures the function identity:
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

   // Usage
   Function<A, B> step1 = traced("parse", this::parse);
   Function<B, C> step2 = traced("validate", this::validate);
   Function<C, D> step3 = traced("enrich", this::enrich);

   Function<A, D> pipeline = step1.andThen(step2).andThen(step3);
   ```
   The traced wrapper preserves the original exception as the cause and adds context about which function in the chain failed and what the input was. For production, use a `java.util.function` decorator with structured logging. Without tracing, a composition chain of 5+ functions becomes impossible to debug when an intermediate step fails.

10. **Q: A `BiConsumer<HttpRequest, HttpResponse>` is used as a middleware handler. The first middleware modifies the request, passes it to the next, and modifies the response. The handler is: `(req, res) -> { audit.log(req); next.accept(req, res); encrypt(res); }`. What concurrency issues arise with this `BiConsumer` chain?**
    A: The `BiConsumer` captures `next` and `audit` via `this`. If multiple requests are processed concurrently, `encrypt(res)` may modify the response while the next handler is still reading it. Fix: ensure each middleware creates copies of mutable state:
    ```java
    // Instead of mutating in place:
    BiConsumer<HttpRequest, HttpResponse> handler = (req, res) -> {
        audit.log(req);
        next.accept(req, res); // res may still be modified by next
        encrypt(res); // Race: next may still be writing to res
    };

    // Use CompletableFuture chaining for pipeline isolation:
    Function<HttpRequest, CompletableFuture<HttpResponse>> handler = req ->
        auditAsync(req)
            .thenCompose(v -> next.apply(req))
            .thenApply(res -> encrypt(res));
    ```
    The `BiConsumer` side-effect pattern is inherently problematic for concurrent pipelines. Each stage shares mutable `req` and `res` references. Prefer `Function<T, CompletableFuture<R>>` where each stage produces a new value rather than mutating shared state.

---

## Interview Questions

1. **What is a functional interface in Java?**
   A: A functional interface is an interface with exactly one abstract method (SAM — Single Abstract Method). It may contain any number of `default` and `static` methods. The `@FunctionalInterface` annotation is optional but recommended — it causes a compile error if a second abstract method is added. Functional interfaces are the target type for lambda expressions and method references.

2. **What are the core functional interfaces in `java.util.function`?**
   A: The six core interfaces are: `Predicate<T>` (T → boolean, method: `test()`), `Consumer<T>` (T → void, method: `accept()`), `Function<T,R>` (T → R, method: `apply()`), `Supplier<T>` (() → T, method: `get()`), `UnaryOperator<T>` (T → T, method: `apply()`), and `BinaryOperator<T>` (T,T → T, method: `apply()`). Each also has bi-variants (`BiPredicate`, `BiConsumer`, `BiFunction`).

3. **What is the difference between `Consumer<T>` and `Function<T, R>`?**
   A: `Consumer<T>` takes an argument and returns no result — it's used for side effects (logging, printing, saving). `Function<T, R>` takes an argument and returns a result — it's used for transformations. Use `Consumer` when the purpose is an action; use `Function` when the purpose is a mapping. `Consumer` supports composition via `andThen()`; `Function` supports `andThen()` and `compose()`.

4. **What is `Supplier<T>` used for?**
   A: `Supplier<T>` represents a function that takes no arguments and returns a value. It's used for: lazy initialization (`Optional.orElseGet(() -> expensiveLoad())`), factory patterns (`Supplier<Connection> = () -> createConnection()`), generating sequences (`Stream.generate(supplier)`), and deferring computation to a later time or thread.

5. **What is the difference between `andThen()` and `compose()` in `Function`?**
   A: `f.andThen(g)` means apply `f` first, then `g` — equivalent to `g(f(x))`. `f.compose(g)` means apply `g` first, then `f` — equivalent to `f(g(x))`. `andThen()` is left-to-right; `compose()` is right-to-left. In most codebases, `andThen()` is more common because it reads naturally in method-chaining style.

6. **Why are primitive specializations like `IntPredicate` important?**
   A: Primitive specializations avoid autoboxing overhead. `Predicate<Integer>` boxes each `int` to `Integer` (28 bytes), while `IntPredicate` uses `int` directly (4 bytes). In hot paths with millions of operations, autoboxing causes allocation pressure and GC pauses. Always use `IntPredicate`, `IntFunction`, `ToIntFunction`, etc. when working with primitives in performance-sensitive code.

7. **Can a functional interface extend another functional interface?**
   A: Yes, but a functional interface with a SAM that matches the parent's SAM is still functional. If the child interface declares a new abstract method, it's no longer a functional interface. `Comparator<T>` is an example that extends `Serializable` with many default/static methods but only one abstract method (`compare`). This is also how to create type aliases: `@FunctionalInterface interface Transformer<T,R> extends Function<T,R> {}`.

8. **How do functional interfaces enable method references?**
   A: Method references (`String::length`, `System.out::println`) are compiled to the same `invokedynamic` instruction as lambdas. The method reference's signature must match the functional interface's SAM. For `Function<String, Integer>`, `String::length` matches because `length()` takes no args and returns `int`. Method references are resolved at compile time — if the method signature doesn't match, it's a compile error, not a runtime failure.

9. **What is the relationship between `Comparator<T>` and functional interfaces?**
   A: `Comparator<T>` is a functional interface with SAM `int compare(T o1, T o2)`. It has many `default` methods (`reversed()`, `thenComparing()`, `thenComparingInt()`) and `static` methods (`comparing()`, `naturalOrder()`, `nullsFirst()`). These composition methods return `Comparator` instances, enabling fluent comparator construction: `Comparator.comparing(Person::age).thenComparing(Person::name).reversed()`.

10. **How do you create a custom functional interface correctly?**
    A: Annotate with `@FunctionalInterface`, ensure exactly one abstract method, and consider extending a standard interface if possible. Example: `@FunctionalInterface interface ThrowingFunction<T,R> { R apply(T t) throws Exception; }`. If the standard `Function<T,R>` would work with a wrapper, prefer that over a custom interface. Custom interfaces should only be created when the standard ones lack necessary semantics (like checked exception support or multiple parameters with meaningful names).

---

## Developer Recommendations

- **Prefer standard functional interfaces over custom ones** — The `java.util.function` package has 43 interfaces covering most use cases. Creating custom interfaces (`Transformer`, `Converter`, `Mapper`) causes incompatibility with standard APIs (Streams, CompletableFuture). If you need semantic clarity, extend the standard interface: `interface Transformer<T,R> extends Function<T,R> {}` This gives type safety while preserving interoperability.

- **Use primitive specializations in performance-sensitive code** — `IntPredicate` avoids boxing 1M integers. `Predicate<Integer>` creates 28 bytes of garbage per element via autoboxing. In a loop processing 1M transactions, that's 28MB of unnecessary allocations. The primitive variants (`IntPredicate`, `LongConsumer`, `ToDoubleFunction`) are 5-10x faster for numeric operations.

- **Compose `Predicate` with `and()`/`or()`/`negate()` instead of writing custom logic** — `predicate.and(other).negate()` is more readable and testable than `x -> predicate.test(x) && !other.test(x)`. Composition methods short-circuit naturally — `and()` stops at the first `false`, `or()` stops at the first `true`. Extract complex combinations to named intermediate predicates for clarity.

- **Use `Consumer` for side effects, `Function` for transformations, `Supplier` for lazy values** — Each functional interface encodes intent. A method taking `Consumer<T>` should perform an action with side effect. A method taking `Function<T,R>` should transform without side effects. Mixing them (e.g., `Function` that prints to console) violates the principle of least surprise and makes the code harder to reason about.

- **Avoid checked exceptions in functional interfaces by wrapping, not by creating multiple custom interfaces** — Instead of creating `ThrowingFunction`, `ThrowingPredicate`, `ThrowingConsumer` (each for every checked exception type), create a single `Unchecked` utility: `Function<T,R> unchecked(ThrowingFunction<T,R> fn)`. This keeps the API surface small and callers can use standard `Function<T,R>` with the wrapper.

- **Use `BinaryOperator<T>` over `BiFunction<T,T,T>` when both args are the same type** — `BinaryOperator<T>` extends `BiFunction<T,T,T>` and adds `minBy()` and `maxBy()` static methods. It also better communicates intent: the operation combines two values of the same type into one. Use `BinaryOperator` for reduction operations, accumulators, and combiners in stream `reduce()` and `collect()`.

- **Prefer `UnaryOperator<T>` over `Function<T,T>` for same-type transformations** — `UnaryOperator<T>` extends `Function<T,T>` and communicates that the input and output types are the same. This matters for identity operations (`UnaryOperator.identity()`) and makes signatures more readable: `List<T> transform(List<T> input, UnaryOperator<T> op)` is clearer than using `Function<T,T>`.
