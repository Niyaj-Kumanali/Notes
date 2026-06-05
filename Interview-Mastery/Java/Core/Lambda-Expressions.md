# Java Lambda Expressions

---

## Overview

- **Definition:** Lambda expressions introduce functional programming constructs to Java — anonymous functions that can be treated as values (passed as arguments, returned from methods, stored in variables).

- **Why It Exists:** Before lambdas, Java required verbose anonymous inner classes for passing behavior. Lambdas enable concise code, functional programming with Stream API, behavior parameterization, and easier parallel processing.

- **Syntax:**

```java
// Full syntax
(parameters) -> { body; return value; }

// Single parameter, no parentheses
param -> expression

// No parameters
() -> expression

// Single expression (returns automatically)
(a, b) -> a + b

// Multiple statements need braces and return
(a, b) -> {
    int sum = a + b;
    log.debug("Sum: {}", sum);
    return sum;
}
```

- **Where Lambdas Are Used:**
  - Collections: `list.sort((a, b) -> a.compareTo(b))`
  - Streams: `list.stream().filter(x -> x > 5).map(x -> x * 2)`
  - Optionals: `optional.orElseGet(() -> expensiveDefault())`
  - CompletableFuture: `future.thenApply(result -> transform(result))`
  - Threads: `new Thread(() -> doWork()).start()`

---

## Lambda vs Anonymous Inner Class

```java
// Anonymous inner class — verbose
button.addActionListener(new ActionListener() {
    @Override
    public void actionPerformed(ActionEvent e) {
        System.out.println("Clicked at " + e.getWhen());
    }
});

// Lambda — concise
button.addActionListener(e -> System.out.println("Clicked at " + e.getWhen()));
```

---

## Type Inference

- **Definition:** The compiler infers the target type of a lambda from the context (assignment, method parameter, return type).

```java
// Explicit types
Function<String, Integer> f = (String s) -> Integer.parseInt(s);

// Inferred types (most common)
Function<String, Integer> f = s -> Integer.parseInt(s);
```

The compiler determines the parameter types, return type, and the functional interface from the context. Type inference reduces boilerplate but can sometimes lead to ambiguity with overloaded methods.

---

## Variable Capture

- **Definition:** Lambdas can access variables from the enclosing scope. The captured variables must be effectively final (not reassigned after initialization).

- **Rules:**
  - **Effectively final local variables** — captured by value (copied into the lambda)
  - **Instance fields** — captured via `this` reference
  - **Static fields** — accessible anytime

```java
public class Example {
    private int instanceField = 42;

    public Supplier<Integer> createSupplier() {
        int localVar = 10;
        // localVar = 20;  // Would cause compile error — not effectively final

        return () -> instanceField + localVar;
        // Captures: this (for instanceField), localVar (copy)
    }
}
```

- **Important:** Captured local variables are copied into the lambda object (stored on heap). Instance fields capture the `this` reference, which can cause memory leaks if the lambda outlives the enclosing object.

---

## Scope

- **Definition:** Lambdas don't introduce a new scope — `this` refers to the enclosing class, not the lambda itself.

```java
public class ScopeExample {
    private String value = "class";

    public void test() {
        Runnable r = () -> {
            System.out.println(this.value);  // "class" — refers to instance field
            // String value = "";  // Compile error — already defined in scope
        };
    }
}
```

This is different from anonymous inner classes where `this` refers to the inner class instance. In lambdas, `this` is the same as in the enclosing method.

---

## Under the Hood: invokedynamic

- **Definition:** Lambdas are compiled using the `invokedynamic` JVM instruction (added in Java 7), not as anonymous inner classes.

```
Source: Supplier<String> s = () -> "hello";

javac generates:
invokedynamic #bootstrap, args:[]

Bootstrap method (LambdaMetafactory):
- Creates CallSite at link time
- Generates implementation class on-the-fly
- Links method handle to the lambda body
- Subsequent calls invoke linked handle directly (no reflection)
```

- **Benefits:**
  - No separate .class file per lambda
  - Lambda is generated once and cached (for non-capturing lambdas)
  - Bootstrap method only runs once
  - Lower memory footprint than anonymous classes

---

## Memory Characteristics

| Lambda Type | Instance | Cached | Captures |
|-------------|----------|--------|----------|
| Non-capturing | Single static instance | Yes (forever) | Nothing |
| Capturing (local vars) | New per call | No | Copies of variables |
| Capturing (this) | New per call | No | `this` reference |
| Method reference (static) | Single static | Yes | Nothing |

Non-capturing lambdas are essentially free — a single static instance is reused forever. Capturing lambdas allocate a new instance each time they're created in a method call.

---

## Method References

- **Definition:** A shorthand for lambdas that call an existing method.

```java
// Lambda
Function<String, Integer> f = s -> Integer.parseInt(s);

// Method reference
Function<String, Integer> f = Integer::parseInt;

// Types:
Class::staticMethod         // Math::max          (a, b) -> Math.max(a, b)
instance::instanceMethod    // System.out::println  x -> System.out.println(x)
Class::instanceMethod       // String::length      s -> s.length()
Class::new                  // ArrayList::new      () -> new ArrayList()
```

Method references are more concise and often more readable than lambdas for simple method delegation.

---

## Common Mistakes

- **Too complex lambda bodies** — extract multi-line lambdas into named methods
- **Mutable captures** — variable must be effectively final or compile error
- **Checked exceptions** — Stream functional interfaces don't declare checked exceptions, need try-catch wrapper
- **Ambiguous overloads** — if method is overloaded with different functional interfaces, cast to disambiguate
- **this reference confusion** — in lambdas, `this` refers to enclosing class, not the lambda
- **Performance assumptions** — capturing lambdas allocate each invocation; non-capturing are cached
- **Debugging difficulty** — lambda stack traces are less readable than named method traces
- **Overusing** — sometimes a simple for-each loop is clearer than a stream pipeline

---

## Real-World Scenarios

### Scenario 1: Dynamic Pricing Engine

An e-commerce platform adjusts prices in real-time based on demand, competitor pricing, and inventory. The pricing logic changes frequently and is configured via business rules. Different strategies (percentage markup, competitor-match, clearance) must be pluggable.

```java
public class PricingEngine {
    private final Map<String, Function<Double, Double>> strategies = Map.of(
        "PERCENTAGE_MARKUP", price -> price * 1.15,
        "COMPETITOR_MATCH", price -> Math.min(price, competitorService.getLowestPrice(productId)),
        "CLEARANCE", price -> price * 0.5
    );

    public double calculatePrice(Product product, String strategy) {
        Function<Double, Double> pricingFn = strategies.getOrDefault(strategy,
            p -> { throw new IllegalArgumentException("Unknown strategy: " + strategy); });
        return pricingFn.apply(product.getBasePrice());
    }
}
```

Each pricing strategy is a `Function<Double, Double>` that can be tested independently. New strategies are added by inserting into the map — no switch statements or if-else chains. The lambda captures configuration (like `competitorService`) at creation time, and the JVM caches non-capturing strategies (like `CLEARANCE`) as a single instance.

### Scenario 2: Event Bus with Listener Filtering

A GUI framework needs to dispatch mouse events to registered listeners. Listeners can filter by event type, region, or modifier keys. Using lambdas avoids creating anonymous inner classes for each listener, reducing memory pressure.

```java
public class EventBus {
    private final List<Consumer<MouseEvent>> listeners = new CopyOnWriteArrayList<>();

    public void onEvent(Predicate<MouseEvent> filter, Consumer<MouseEvent> handler) {
        listeners.add(event -> {
            if (filter.test(event)) handler.accept(event);
        });
    }

    public void dispatch(MouseEvent event) {
        listeners.forEach(listener -> listener.accept(event));
    }
}

// Usage
eventBus.onEvent(
    e -> e.getType() == MouseEvent.CLICK && e.getButton() == 1,
    e -> statusBar.setText("Clicked at " + e.getPoint())
);
```

Each `onEvent()` call creates a capturing lambda that holds references to `filter` and `handler`. The `CopyOnWriteArrayList` ensures thread-safe iteration. The real cost: if `onEvent()` is called in a hot loop, each call allocates a new lambda object — a price worth paying for the expressive API.

### Scenario 3: Microservice Request Circuit Breaker

A circuit breaker wraps calls to downstream services. When failures exceed a threshold, the circuit opens and redirects to a fallback. The fallback logic varies per service and is passed as a lambda.

```java
public class CircuitBreaker {
    private final Supplier<Response> primary;
    private final Supplier<Response> fallback;
    private int failures = 0;
    private boolean open = false;

    public CircuitBreaker(Supplier<Response> primary, Supplier<Response> fallback) {
        this.primary = primary;
        this.fallback = fallback;
    }

    public Response execute() {
        if (open) return fallback.get();
        try {
            Response response = primary.get();
            failures = 0;
            return response;
        } catch (Exception e) {
            failures++;
            if (failures > 5) open = true;
            return fallback.get();
        }
    }
}

// Usage
CircuitBreaker cb = new CircuitBreaker(
    () -> paymentClient.charge(order),
    () -> Response.fallback("payment_deferred")
);
```

The lambdas capture the `order` variable (effectively final) and the `paymentClient` reference (instance field, captured via `this`). A capturing lambda in a framework like this is fine — the circuit breaker is created once per request, not in a hot loop. The key is knowing when capturing is acceptable (per-request) vs problematic (hot loop).

---

## Scenario-Based Questions

1. **Q: You are building a batch processing framework where the user defines transformation steps as lambdas. Each step processes 10M records. The user writes `records.stream().map(s -> process(s)).collect(toList())` but the lambda captures a large configuration object that's created for each batch. The batch runs once per minute and memory grows unbounded. What's happening?**
   A: The lambda captures the configuration object via `this` (if `process()` is an instance method of the configuration). Each batch creates a new `Stream` object, a new lambda instance, and the captured configuration keeps references to per-batch data that can't be GC'd. Fix: extract `process` to a static method or pass the configuration parameter explicitly: `records.stream().map(record -> ConfigProcessor.process(config, record))` — now only `config` and `record` are captured, and `config` can be reused across batches. Better: use method reference `ConfigProcessor::process` with a `BiFunction`.

2. **Q: You have a high-frequency trading system that processes 1M order book updates per second. Each update triggers a lambda that checks price thresholds and sends alerts. The system creates millions of short-lived lambda instances. JVM GC pauses spike to 500ms every few seconds. How do you reduce allocation pressure?**
   A: Non-capturing lambdas are cached as static singletons — they never allocate. Capturing lambdas allocate per invocation. Audit all lambdas in the hot path: (1) Extract instance method references — `orderBook::update` is non-capturing and cached. (2) For lambdas that need parameters, create `Function` constants: `private static final Function<Trade, Boolean> IS_LARGE_TRADE = t -> t.getQuantity() > 10000;`. (3) In hot loops, replace lambdas with direct calls or pre-allocated `Consumer` instances. The 500ms GC pauses are from allocating millions of lambda objects — making them non-capturing eliminates 90% of the garbage.

3. **Q: A lambda passed to `CompletableFuture.supplyAsync()` captures a JDBC connection. The async operation runs 30 seconds later, by which time the connection is closed. The lambda throws `SQLException`. How do you prevent this?**
   A: The lambda captured the connection reference, but the connection was closed by the time the lambda executed. Fix: open the connection inside the lambda or pass connection parameters (not the connection itself):
   ```java
   // Bad — captures connection that may close
   ResultSet process(Connection conn, String query) {
       return CompletableFuture.supplyAsync(() -> {
           return conn.executeQuery(query); // conn may be closed!
       }).join();
   }

   // Good — capture connection parameters, create connection inside
   ResultSet process(DataSource ds, String query) {
       return CompletableFuture.supplyAsync(() -> {
           try (Connection conn = ds.getConnection()) {
               return conn.executeQuery(query);
           }
       }).join();
   }
   ```
   The rule: always capture the factory (DataSource), not the resource (Connection). Resources are time-bound and may be closed before the lambda executes in a different thread.

4. **Q: A team is migrating from anonymous inner classes to lambdas. They find that some `this` references behave differently. In an anonymous class, `this.getStatus()` returns the listener's status; with a lambda, it returns the enclosing class's status. How do you document and mitigate this?**
   A: Document the scoping rule: in lambdas, `this` refers to the enclosing instance; in anonymous classes, `this` refers to the anonymous class instance. For migration: (1) Replace `this.method()` with `EnclosingClass.this.method()` in anonymous inner classes before converting to lambdas — this disambiguates. (2) After conversion, verify that `this` references still work correctly. (3) For event handlers that need `this` to refer to the handler itself, keep anonymous inner classes or extract named inner classes. The rule: if the anonymous class overrides methods besides the SAM (rare), it can't be replaced with a lambda.

5. **Q: A service processes webhook callbacks. Each callback has a unique signature (event type + source). The handler for each signature is a lambda stored in a `Map<String, Consumer<WebhookEvent>>`. Over time, memory usage grows as handlers accumulate. What's the issue?**
   A: Each handler lambda captures the context in which it was created (often `this`). If the map holds references to these lambdas, it also holds references to the objects that created them, preventing GC. Fix: make handlers non-capturing where possible: (1) Register handlers as static method references: `register("payment.success", PaymentHandler::onSuccess)`. (2) If the handler needs instance state, store the state in the handler object itself, not captured from the registration context. (3) Use a weak-valued map (`new WeakHashMap<>)`) for handler registrations so that GC can reclaim unused handler contexts.

6. **Q: A sorting API accepts `Comparator<Person>` as a lambda. Users write `list.sort((a, b) -> a.age() - b.age())`. This fails for large age values due to integer overflow. How do you prevent this anti-pattern across the codebase?**
   A: Integer subtraction as comparator is an overflow bug — `Integer.MAX_VALUE - (-1)` overflows to negative. Fix: enforce `Comparator.comparingInt(Person::age)` instead:
   ```java
   // Anti-pattern — integer overflow bug
   list.sort((a, b) -> a.age() - b.age()); // Overflow for extreme values

   // Correct — no overflow
   list.sort(Comparator.comparingInt(Person::age));

   // For multi-field: chaining
   list.sort(Comparator.comparingInt(Person::age)
       .thenComparing(Person::name));
   ```
   Use static lint rules (ErrorProne, SpotBugs) to flag subtraction-based comparators. The `Comparator` utility methods are not only safer but also more readable and potentially faster (JVM intrinsics for comparing primitives).

7. **Q: A logging framework uses `Supplier<String>` for lazy message evaluation: `log.debug(() -> expensiveToString())`. A developer accidentally calls `log.debug(expensiveToString())` (eager evaluation), causing performance degradation. How do you make the API foolproof?**
   A: This is a lambda vs method reference confusion. The eager call `log.debug(expensiveToString())` evaluates the expression before passing it. Several approaches: (1) Use overloaded methods where the eager version takes `String` and the lazy takes `Supplier<String>` — overload resolution picks the eager version for `debug("literal")` and the lazy for `debug(someObject::toString)`. (2) Use a distinct method name: `log.debugLazy(() -> expensiveToString())`. (3) Add a parameter annotation processor (Checker Framework) that flags eager calls with `@Lazy` annotated parameters. The `Supplier` pattern is correct — the problem is the caller's mental model of "pass a lambda" vs "call a method and pass the result."

8. **Q: You have a function `public void process(Consumer<String> handler)` that is called thousands of times per second. Each call creates a new lambda `s -> doSomething(s)`. The allocation rate causes GC pressure. How do you optimize without changing the API?**
   A: Pre-allocate the lambda as a constant. If `doSomething` is stateless, the lambda is non-capturing and can be a static field:
   ```java
   // Before — allocates per call
   service.process(s -> doSomething(s));

   // After — single instance, cached forever
   private static final Consumer<String> HANDLER = s -> doSomething(s);
   service.process(HANDLER);
   ```
   If `doSomething` depends on instance state, extract state to a method parameter and use a `BiConsumer` applied via currying, or use an instance method reference: `service.process(this::doSomething)` captures `this` but is still cached (method references to instance methods are cached per method per class).

9. **Q: A lambda `x -> compute(x)` is used in a `Stream.flatMap()`. `compute()` returns `Stream.empty()` for some inputs, which is correct behavior. However, the stream is sometimes `null` instead of empty, causing NPE. How do you enforce the contract?**
   A: `flatMap()` throws NPE if the lambda returns `null`. Fix: use `flatMap(x -> compute(x) != null ? compute(x) : Stream.empty())` but this calls `compute()` twice. Better:
   ```java
   // Correct — single evaluation with null guard
   items.stream()
       .flatMap(item -> {
           Stream<Result> stream = compute(item);
           return stream != null ? stream : Stream.empty();
       })
       .collect(toList());

   // Even better — use Optional
   items.stream()
       .map(this::compute)
       .filter(Objects::nonNull)
       .flatMap(Function.identity())
       .collect(toList());
   ```
   The broader lesson: lambdas that can return `null` violate the `flatMap()` contract. Always document in the method JavaDoc that non-null is required, and use the `Optional` pattern to make the contract explicit.

10. **Q: A team writes lambdas that access mutable instance fields without synchronization. Under load, the fields have wrong values. The team blames lambdas for not being thread-safe. Is the lambda the problem?**
    A: The lambda itself is not the problem — it's code that captures `this` and accesses mutable state. Lambdas don't introduce any thread-safety guarantees (or lack thereof) beyond anonymous inner classes. The issue is shared mutable state accessed from multiple threads:
    ```java
    // Problematic — lambda captures this, accesses mutable list
    List<String> results = new ArrayList<>();
    items.parallelStream().forEach(item -> results.add(process(item)));

    // Fix — use collection that belongs to the lambda
    List<String> results = items.parallelStream()
        .map(item -> process(item))
        .collect(toList());
    ```
    The lambda is a function, not a transaction. Thread safety depends on what the lambda does, not the lambda itself. Prefer stateless lambdas and let the stream framework handle accumulation.

---

## Interview Questions

1. **What is a lambda expression in Java?**
   A: A lambda expression is an anonymous function that can be treated as a value — passed as an argument, returned from a method, or stored in a variable. It provides a concise syntax for implementing single-method interfaces (functional interfaces). Syntax: `(parameters) -> { body }`.

2. **What is the difference between a lambda and an anonymous inner class?**
   A: Lambdas are compiled using `invokedynamic` (no separate `.class` file); anonymous inner classes are compiled to separate class files. Lambdas cannot have instance fields or methods; anonymous classes can. `this` in a lambda refers to the enclosing class; in an anonymous class, `this` refers to the anonymous class instance. Non-capturing lambdas are cached as singletons; anonymous inner classes allocate per use.

3. **What does "effectively final" mean in the context of lambdas?**
   A: A variable is effectively final if its value is never changed after initialization — even without the `final` keyword. Lambdas can only capture local variables that are effectively final. This prevents race conditions in concurrent lambda execution and makes the captured value predictable. Instance fields and static fields can be captured regardless of mutability because they are stored on the heap (visible to all threads).

4. **What is variable capture and how does it work under the hood?**
   A: Variable capture is when a lambda accesses variables from its enclosing scope. Local variables are copied into the lambda's object when it's created (they must be effectively final). Instance fields are captured via the `this` reference. Under the hood, the JVM uses `invokedynamic` with `LambdaMetafactory` — it generates a class at runtime that holds captured values as fields and implements the functional interface. Capturing local variables copies them into the generated class's fields.

5. **How are lambdas compiled and executed?**
   A: The Java compiler generates an `invokedynamic` instruction (Java 7+) pointing to `LambdaMetafactory`. At runtime, the first invocation triggers the bootstrap method, which generates the lambda implementation class using `MethodHandle` (not reflection). Subsequent calls invoke the cached implementation directly. Non-capturing lambdas generate a single static instance; capturing lambdas create a new instance on every call.

6. **What is a method reference and how does it differ from a lambda?**
   A: A method reference (`String::length`) is a shorthand for a lambda that calls an existing method. It's more concise and often more readable. Types: `Class::staticMethod`, `instance::instanceMethod`, `Class::instanceMethod`, `Class::new` (constructor). Method references are typically non-capturing (if static) or capture `this` (if instance method reference). They share the same invocation mechanism as lambdas.

7. **Can lambdas have checked exceptions? How do you handle them?**
   A: No functional interface in `java.util.function` declares checked exceptions. To call a method that throws a checked exception inside a lambda, you must either: (1) wrap in try-catch and rethrow as unchecked (`RuntimeException`), (2) create a custom `@FunctionalInterface` that declares the exception, or (3) use a utility wrapper that converts checked to unchecked: `Function<T, R> unchecked(ThrowingFunction<T, R> f)`.

8. **What is the difference between a capturing and non-capturing lambda, and why does it matter for performance?**
   A: A non-capturing lambda doesn't access any variables from the enclosing scope and is cached as a singleton (zero allocation per call). A capturing lambda accesses local variables, parameters, or instance fields and creates a new instance each time it's evaluated. In hot loops, capturing lambdas cause allocation pressure and GC pauses. Method references to static methods are always non-capturing. Prefer non-capturing lambdas in performance-critical code.

9. **How do you make a lambda serializable?**
   A: Lambdas are not serializable by default. To make one serializable, cast it to the intersection type: `(Function<String, Integer> & Serializable) s -> s.length()`. For distributed systems (Spark, Hazelcast), extract the logic to a class that implements `Serializable` and use method references: `(Function<String, Integer> & Serializable) String::length`. Serializable lambdas capture their enclosing scope, which must also be serializable.

10. **What are the limitations of lambdas compared to anonymous inner classes?**
    A: Lambdas cannot: (1) have instance fields or methods (anonymous classes can), (2) define new variables in the same scope as enclosing variables (shadowing), (3) implement multiple methods (must be a functional interface), (4) refer to `this` as the lambda itself, (5) have the same flexibility with generic type parameters. Use anonymous inner classes when you need state or multiple methods; use lambdas for stateless behavior passing.

---

## Developer Recommendations

- **Prefer non-capturing lambdas in hot paths** — Non-capturing lambdas are cached as static singletons (zero allocation per invocation). Capturing lambdas allocate a new object every time. In a loop processing 1M items, a capturing lambda creates 1M garbage objects, causing GC pauses. Extract instance state to method parameters and use method references: `items.forEach(this::process)` instead of `items.forEach(item -> process(item, config))`.

- **Use method references over lambdas for single-method delegation** — `list.sort(Comparator.comparingInt(Order::getTotal))` is more readable and has better allocation characteristics than `list.sort((a, b) -> Integer.compare(a.getTotal(), b.getTotal()))`. Method references also fail at compile time if the method signature doesn't match (unlike lambdas which may compile but fail at runtime).

- **Extract multi-line lambdas into named methods** — A lambda with 5+ lines, multiple conditions, or try-catch blocks is harder to debug (stack traces show `lambda$methodName$N`) and harder to test. Extract to a private method and use method reference: `stream.map(this::processOrder)` instead of `stream.map(order -> { ... 10 lines ... })`. This improves stack trace readability and allows unit testing the method directly.

- **Keep lambdas stateless for parallel streams** — Parallel streams distribute elements across threads. Lambdas with shared mutable state (e.g., incrementing a counter, adding to a shared list) cause race conditions. Prefer `map()` with stateless transformations and let `collect()` handle accumulation: `stream.map(this::transform).collect(toList())` instead of `stream.forEach(item -> { synchronized(lock) { results.add(item); } })`.

- **Avoid ambiguous overloads with functional interfaces** — Overloading methods with `Predicate<T>` and `Function<T,R>` causes compilation errors when callers use lambdas. Use distinct method names: `filter(Predicate)` vs `transform(Function)`. If overloading is necessary, document which functional interface takes precedence and advise callers to use explicit casts.

- **Use `IntStream.range()` over loop-based lambda capture** — When creating lambdas in a loop (e.g., submitting tasks to an executor), copying the loop variable to an effectively-final local variable is error-prone. Use `IntStream.range(0, n).forEach(i -> executor.submit(() -> process(items.get(i))))` — the `i` variable in the lambda is unique per iteration, eliminating the need for manual copying.

- **Profile lambda allocation before optimizing** — Lambda allocation is often irrelevant compared to I/O, database, or network costs. Use a profiler (Async Profiler, JFR) to confirm lambdas are a bottleneck before optimizing. For capturing lambdas in non-hot paths, the readability benefit far outweighs the allocation cost. The rule: optimize only after measuring.
