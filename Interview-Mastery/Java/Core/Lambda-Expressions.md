# Java Lambda Expressions

---

## Overview

- **Purpose** — Lambda expressions, introduced in Java 8, are anonymous functions that can be treated as first-class values — passed as arguments to methods, returned from methods, and stored in variables. They enable declarative, functional programming styles and are the foundation for the Stream API, `Optional`, and `CompletableFuture`.
- **Before Lambdas** — Java required verbose anonymous inner classes to achieve behavior parameterization, making functional patterns like callbacks, event handlers, and sorting unnecessarily verbose with boilerplate class declarations.

  **Why not just sugar-coat anonymous classes?** Early prototypes compiled lambdas to anonymous inner classes, but this approach had three fatal flaws: (1) each lambda produced a separate `.class` file, increasing deployment size and classloader pressure; (2) the compiler could not optimize across anonymous class boundaries; (3) every evaluation allocated a new instance on the heap. The `invokedynamic` approach (introduced in Java 7 for dynamic languages) solved all three — it generates the implementation class once at runtime, caches it permanently, and allocates zero objects for non-capturing lambdas. **Why capture-by-value for local variables?** Local variables live on the stack. If a lambda escapes to another thread, the creating method's stack frame is gone. Copying the value into the lambda's heap object is the only safe approach — capture-by-reference would leave a dangling pointer. This is why captured local variables must be effectively final: if they could change, the lambda's copy and the original would diverge, violating the programmer's expectation.
- **Syntax Forms** — Full block syntax `(parameters) -> { body; return value; }`, single-expression syntax `param -> expression` where the expression is automatically returned, and no-parameter syntax `() -> expression`. They are used throughout the JDK for collection operations, stream pipelines, optional fallbacks, async callbacks, and thread creation.

### Syntax Examples

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

- **Compilation Difference** — Lambdas are compiled using the `invokedynamic` JVM instruction and do not produce a separate `.class` file, while anonymous inner classes compile into individual class files that must be loaded by the classloader.
- **this Reference** — In a lambda, `this` refers to the enclosing class instance, whereas in an anonymous inner class, `this` refers to the anonymous class instance itself — a distinction that causes subtle bugs during migration.
- **Memory Characteristics** — Non-capturing lambdas are cached as static singletons by the JVM and never allocate after the first invocation, while every use of an anonymous inner class creates a new instance on the heap.
- **Capabilities** — Anonymous inner classes can declare fields, methods, and constructors, and they can implement multiple methods of the interface — capabilities that lambdas fundamentally lack.

---

## Type Inference

- **Compiler Inference** — The Java compiler infers the target type of a lambda from the context in which it appears — the assignment target, the method parameter type, or the return type. In `Function<String, Integer> f = s -> Integer.parseInt(s)`, the compiler knows `s` is `String` and the return type is `Integer`.
- **Reduced Boilerplate** — Type inference makes lambda expressions significantly more concise than anonymous inner class equivalents, but it can cause ambiguity when a method is overloaded with different functional interface parameters.
- **Disambiguation** — When inference fails, provide explicit parameter types: `(String s) -> Integer.parseInt(s)`, or cast the lambda to the desired functional interface.

```java
// Explicit types
Function<String, Integer> f = (String s) -> Integer.parseInt(s);

// Inferred types (most common)
Function<String, Integer> f = s -> Integer.parseInt(s);
```

---

## Variable Capture

- **Local Variable Rules** — Lambdas can access variables from the enclosing scope. Local variables and method parameters must be effectively final — their value must not change after initialization — because they are copied into the lambda object when it is created.
- **Instance Fields** — Instance fields are captured via the `this` reference, meaning the lambda holds a reference to the enclosing object and can access its mutable state directly. Static fields have no capture cost.
- **Memory Implications** — Capturing local variables copies their values into the lambda's heap-allocated implementation object, while capturing instance fields prevents the enclosing object from being garbage collected as long as the lambda is reachable — a common source of memory leaks.

  **Why effectively-final?** The rule exists because local variables live on the stack. If a lambda escapes to another thread, the creating method's stack frame is gone — a captured reference to a stack-allocated variable would dangle. Java's designers chose capture-by-value (copy the bits into the lambda's heap object) over capture-by-reference. The effectively-final constraint is the logical consequence: if the local variable could change, the lambda's copy and the original would diverge, creating a correctness hole that cannot be patched without runtime checks. Instance fields avoid this because they live on the heap — `this` is always accessible through the lambda's reference to the enclosing object.

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

---

## Scope

- **Lexical Scoping** — Lambdas do not introduce a new lexical scope — the `this` keyword inside a lambda refers to the enclosing class instance, not to the lambda itself. This is fundamentally different from anonymous inner classes where `this` refers to the anonymous class instance.
- **Variable Shadowing** — Because lambdas share their scope with the enclosing method, they cannot declare local variables with the same name as variables in the enclosing scope — attempting to do so causes a compile error about duplicate definitions.
- **Consistency** — This scoping behavior means lambdas are more consistent with surrounding code (no unexpected `this` changes) but cannot replace anonymous inner classes when the handler needs to refer to itself by `this`.

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

---

## Under the Hood: invokedynamic

- **Compilation** — Lambda expressions are compiled using the `invokedynamic` JVM instruction (introduced in Java 7), not as anonymous inner classes. The Java compiler generates an `invokedynamic` call site in the bytecode that points to a bootstrap method in `LambdaMetafactory`.
- **Runtime Behavior** — At runtime, the first invocation of the lambda triggers the bootstrap method, which uses `MethodHandle` (not reflection) to generate the lambda's implementation class on the fly, creating a `CallSite` that is permanently linked to the generated implementation.
- **Performance** — Subsequent invocations of the same lambda call point call the linked method handle directly without any bootstrap overhead, reflection, or intermediate allocation. Non-capturing lambdas are generated once and cached forever.
- **Benefits** — No separate `.class` files per lambda (reducing deployment size and classloader pressure), the implementation is generated once and cached, and the memory footprint is significantly lower than anonymous inner classes.

  **Why `invokedynamic` and not just compile to anonymous classes?** The Java 8 team evaluated three alternatives:
  1. **Anonymous class compilation** — Each lambda produces a `.class` file, loaded and verified by the classloader. Cost: one class file, class-loading overhead, and one new heap instance per evaluation (even for stateless lambdas). Plus, class definition is forever — no JVM can unload a class once loaded.
  2. **Reflection-based proxies** — `Proxy.newProxyInstance` generates proxy classes dynamically but uses reflection for dispatch (slower than direct invocation) and cannot be inlined by the JIT.
  3. **`invokedynamic` with `LambdaMetafactory`** — The call site is linked once at runtime using `MethodHandle`, then the JIT can inline it as aggressively as any regular method call. Non-capturing lambdas allocate zero bytes. The strategic advantage: the `LambdaMetafactory` itself is an implementation detail that can be replaced in future JDK releases without recompiling source code — exactly what happened in JDK 15-17 when the inner `InnerClassLambdaMetafactory` was significantly optimized.

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

---

## Memory Characteristics

| Lambda Type | Instance | Cached | Captures |
|-------------|----------|--------|----------|
| Non-capturing | Single static instance | Yes (forever) | Nothing |
| Capturing (local vars) | New per call | No | Copies of variables |
| Capturing (this) | New per call | No | `this` reference |
| Method reference (static) | Single static | Yes | Nothing |

- **Non-Capturing Lambdas** — Essentially free in terms of allocation cost — the JVM creates a single static instance on first invocation and reuses it forever, making the allocation cost zero after the first use.
- **Capturing Lambdas** — Allocate a new instance on the heap every time the lambda expression is evaluated, which occurs each time the enclosing method is called.
- **Method References** — A method reference to a static method like `Integer::parseInt` is always non-capturing and cached, while an instance method reference like `this::process` captures `this` and allocates per call.

---

## Method References

- **Shorthand Notation** — Method references use the `::` operator with four distinct forms as a shorthand for lambdas that simply call an existing method. They are more concise and often more readable than equivalent lambdas.
- **Class::staticMethod** — `Math::max` translates to `(a, b) -> Math.max(a, b)` and is always non-capturing and cached.
- **instance::instanceMethod** — `System.out::println` translates to `x -> System.out.println(x)` and captures the instance reference.
- **Class::instanceMethod** — `String::length` translates to `s -> s.length()` where the first argument becomes the receiver of the method call.
- **Class::new** — `ArrayList::new` translates to `() -> new ArrayList()` and invokes the constructor corresponding to the functional interface's parameter list.
- **Compile-Time Safety** — Method references fail at compile time if the method signature does not match the functional interface, providing earlier error detection than lambdas.

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

---

## Common Mistakes

- **Overly Complex Lambda Bodies** — A lambda with 10+ lines, multiple conditionals, and try-catch blocks is harder to debug (stack traces show `lambda$methodName$N` instead of a meaningful method name) and cannot be unit tested. Extract such logic into a named private method and use a method reference instead.
- **Mutating Captured Local Variables** — Attempting to mutate a captured local variable causes a compile-time error because the variable must be effectively final. If you need mutable local state inside a lambda, use a mutable container like an array with one element or an `AtomicReference`.
- **Checked Exceptions in Lambdas** — Calling a method that throws a checked exception inside a standard functional interface lambda produces a compile error because interfaces like `Function<T,R>` do not declare checked exceptions. Workarounds include wrapping in a try-catch that rethrows as `RuntimeException`, creating a custom `@FunctionalInterface` that declares the checked exception, or using a utility method that adapts a throwing function.
- **Overloaded Method Ambiguity** — Using lambdas with overloaded methods that take different functional interfaces causes ambiguity that the compiler cannot resolve. The fix is to rename one method, cast the lambda, or assign the lambda to a typed variable before passing it.
- **Capturing `this` in Hot Path** — A lambda like `items.forEach(item -> process(item))` inside an instance method captures `this` implicitly because `process()` is an instance method. Each invocation allocates a new lambda object holding a reference to the enclosing object, preventing GC of that object until the lambda is unreachable. In a hot loop called millions of times, this means millions of allocations holding references to the same object. The fix: extract to a static method and pass state explicitly: `items.forEach(item -> processHelper(config, item))`.
- **Shared Mutable State in Parallel Streams** — `list.parallelStream().forEach(x -> counter++)` has a data race on `counter` and the lambda itself is not the problem — it's the shared mutable state. The concrete mistake is assuming the lambda's isolation implies thread safety. The fix is to use `map()` and `collect()` for stateless accumulation: `list.parallelStream().collect(Collectors.summingInt(x -> 1))`.

---

## Real-World Scenarios

### Scenario 1: Dynamic Pricing Engine

An e-commerce platform adjusts product prices in real-time based on demand levels, competitor pricing feeds, and current inventory positions. The pricing logic changes frequently as the business team deploys new rules, and different product categories use different strategies. The system must support pluggable strategies without requiring code changes for each new rule.

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

Each pricing strategy is a `Function<Double, Double>` that can be tested independently, and new strategies are added by inserting a new entry into the map with no switch statements or if-else chains required. The `CLEARANCE` strategy is a non-capturing lambda that the JVM caches as a single static instance with zero allocation per invocation, while `COMPETITOR_MATCH` captures the `competitorService` instance (via `this`) and the `productId` parameter. This pattern demonstrates the power of treating functions as values — the pricing logic becomes data that can be configured, tested, and composed.

**Why this approach?** A traditional strategy pattern using an interface with implementations (PercentageMarkupStrategy, CompetitorMatchStrategy, etc.) would require a class per strategy, a switch statement to select the right one, and class-loading overhead for every strategy. The `Map<String, Function>` approach eliminates all boilerplate around the strategy pattern — adding a new strategy is a single-line map insertion instead of a new class file. The trade-off is that complex strategies with mutable state or multi-step calculations need a named class; but for pure mathematical transformations, lambdas are both more concise and more performant (the JVM caches non-capturing strategies permanently).

### Scenario 2: Event Bus with Listener Filtering

A GUI framework dispatches mouse events to registered listeners, where each listener specifies a filter predicate (e.g., only left-click events in a specific region) and a handler that processes matching events. Using lambdas avoids creating anonymous inner classes for each listener registration, reducing memory pressure in UI-heavy applications.

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
```

Each `onEvent()` call creates a capturing lambda that holds references to `filter` and `handler` — the lambda captures both parameters because they are used in its body and they are effectively final. The `CopyOnWriteArrayList` ensures thread-safe iteration when dispatching events to listeners, providing lock-free reads at the cost of copying the array on each listener registration. If `onEvent()` is called in a hot loop (e.g., registering hundreds of listeners per frame), each call allocates a new lambda object — a cost that is acceptable for the readability and expressiveness it provides over anonymous inner classes.

**Why this approach?** The alternative — a dedicated `FilteredListener` class implementing `Consumer<MouseEvent>` with filter and handler as constructor parameters — would produce the same memory footprint (one instance per registration) without the lambda's caching advantage for non-capturing cases. The lambda version is preferred because it reads as a declarative rule: "on left-click in this region, do this action" — the filter and handler appear inline where the listener is registered, not in a separate file. The `CopyOnWriteArrayList` is chosen over synchronized blocks because event dispatch is read-heavy (many dispatches per registration); copy-on-write makes reads lock-free at the cost of making registration O(n) — a correct trade-off when registration is infrequent.

### Scenario 3: Microservice Request Circuit Breaker

A circuit breaker wraps calls to downstream services, tracking failure counts and opening the circuit when failures exceed a configurable threshold. Once the circuit is open, subsequent calls immediately route to a fallback function without attempting the primary operation, preventing cascading failures in a microservice topology.

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

The lambdas capture the `order` variable (effectively final) and the `paymentClient` reference (an instance field captured via `this`). This capturing behavior is acceptable because the circuit breaker is created once per request, not in a hot loop — the allocation cost is negligible compared to the downstream HTTP call. The key design insight is knowing when capturing is acceptable (per-request framework objects) versus problematic (hot-loop iteration where millions of lambdas would be allocated per second).

**Why this approach?** A circuit breaker implemented as an abstract class with template methods (abstract `doPrimary()` and `doFallback()`) would require a subclass per call site, each with its own `.class` file, or an anonymous inner class that allocates regardless of capture semantics. The `Supplier<Response>` interface is the minimal contract — the circuit breaker does not need to know how the primary or fallback works, only that they produce responses. Lambdas let the caller supply behavior inline at construction, keeping the circuit breaker generic and reusable. The allocation cost of the lambdas (a few objects per request) is dwarfed by the HTTP connection overhead, so optimizing the lambda allocation here would be premature.

---

## Scenario-Based Questions

**Q: You are building a batch processing framework where the user defines transformation steps as lambdas. Each step processes 10M records. The user writes `records.stream().map(s -> process(s)).collect(toList())` but the lambda captures a large configuration object that's created for each batch. The batch runs once per minute and memory grows unbounded. What's happening?**

A: The lambda captures the configuration object via `this` because `process()` is an instance method of the configuration class. Each batch creates a new `Stream` object, a new lambda instance that holds a reference to the configuration, and the captured configuration keeps references to per-batch data that prevents garbage collection. The fix is to make the transformation logic static: extract `process` to a static method and pass the configuration as an explicit parameter: `records.stream().map(record -> ConfigProcessor.process(config, record))`. Now only `config` and `record` are captured, and `config` can be reused across batches and properly GC'd. The deeper issue is that capturing `this` in a lambda prevents the entire enclosing object from being reclaimed — an easy mistake when migrating instance methods to lambdas without considering the capture semantics.

**Q: You have a high-frequency trading system that processes 1M order book updates per second. Each update triggers a lambda that checks price thresholds and sends alerts. The system creates millions of short-lived lambda instances. JVM GC pauses spike to 500ms every few seconds. How do you reduce allocation pressure?**

A: Non-capturing lambdas are cached as static singletons by the `LambdaMetafactory` and allocate exactly zero heap objects per invocation after the first bootstrap. Capturing lambdas allocate a new instance on every evaluation of the lambda expression, which in a hot loop means millions of allocations per second. Audit every lambda in the hot path: extract instance methods to static where possible, use method references like `orderBook::update` which are non-capturing and cached, and create `Function` constants as `private static final` fields for frequently used transformations. For the extreme hot path, replace lambdas with direct method calls or pre-allocated `Consumer` instances stored in fields. The 500ms GC pauses are a direct symptom of the allocation rate exceeding the young generation collection budget — making lambdas non-capturing eliminates 90% of the garbage and restores predictable latency.

**Q: A lambda passed to `CompletableFuture.supplyAsync()` captures a JDBC connection. The async operation runs 30 seconds later, by which time the connection is closed. The lambda throws `SQLException`. How do you prevent this?**

A: The lambda captured a reference to the `Connection` object, but that connection was opened in the calling thread and likely closed in a `try-with-resources` block or connection pool return before the async task executed. Never capture resources that are time-bound — always capture the factory or datasource and create the resource inside the lambda body:
```java
// Good — capture connection parameters, create connection inside
ResultSet process(DataSource ds, String query) {
    return CompletableFuture.supplyAsync(() -> {
        try (Connection conn = ds.getConnection()) {
            return conn.executeQuery(query);
        }
    }).join();
}
```
The rule is to capture the factory (DataSource), not the resource (Connection). Resources are time-bound and may be closed before the lambda executes — especially in async contexts where the lambda runs on a different thread and potentially after the calling method has returned.

**Q: A team is migrating from anonymous inner classes to lambdas. They find that some `this` references behave differently. In an anonymous class, `this.getStatus()` returns the listener's status; with a lambda, it returns the enclosing class's status. How do you document and mitigate this?**

A: The scoping difference is the most common migration pitfall. In anonymous inner classes, `this` refers to the anonymous class instance itself — calling `this.getStatus()` calls the listener's own `getStatus()` method if it exists. In lambdas, `this` refers to the enclosing class instance, so `this.getStatus()` calls the enclosing object's method. Before migration, audit all `this` references in the anonymous inner class: replace `this.method()` with `EnclosingClass.this.method()` in the anonymous class to make them explicit, then convert to a lambda. After conversion, verify that any `this` references that were intended to refer to the inner class itself (for example, passing `this` to another method) are replaced with the appropriate enclosing reference. If the anonymous class overrides methods beyond the single abstract method of the functional interface, it cannot be replaced with a lambda at all.

**Q: A service processes webhook callbacks. Each callback has a unique signature (event type + source). The handler for each signature is a lambda stored in a `Map<String, Consumer<WebhookEvent>>`. Over time, memory usage grows as handlers accumulate. What's the issue?**

A: Each handler lambda captures the context in which it was registered — typically the `this` reference of the registering object or some other scope containing resources. As long as these lambdas are held in the map, they prevent the captured objects from being garbage collected, creating a retention chain. The fix involves making handlers non-capturing where possible: register static method references like `register("payment.success", PaymentHandler::onSuccess)` which capture nothing and are cached forever. If handlers require instance state, store the state in the handler object itself rather than capturing it from the registration context, or use a `WeakHashMap<HandlerKey, Consumer<WebhookEvent>>` so that GC can reclaim unused handler contexts when the registering object is no longer strongly reachable.

**Q: A sorting API accepts `Comparator<Person>` as a lambda. Users write `list.sort((a, b) -> a.age() - b.age())`. This fails for large age values due to integer overflow. How do you prevent this anti-pattern across the codebase?**

A: The subtraction-based comparator `a.age() - b.age()` has an integer overflow bug: if `a.age()` is `Integer.MAX_VALUE` and `b.age()` is `-1`, the result wraps around to a negative value, incorrectly reporting that the maximum age is less than -1. Enforce the use of `Comparator.comparingInt(Person::age)` through static analysis tools (ErrorProne, SpotBugs, or Checkstyle rules). The `Comparator` utility methods are not only safe from integer overflow but also more readable and potentially faster because the JVM can intrinsify primitive comparison operations. For multi-field sorting, chain comparators: `Comparator.comparingInt(Person::age).thenComparing(Person::name)`. Add a lint rule that rejects subtraction-based comparators with an error message explaining the overflow risk.

**Q: A logging framework uses `Supplier<String>` for lazy message evaluation: `log.debug(() -> expensiveToString())`. A developer accidentally calls `log.debug(expensiveToString())` (eager evaluation), causing performance degradation. How do you make the API foolproof?**

A: The distinction between `log.debug(() -> expensiveToString())` (lazy, lambda) and `log.debug(expensiveToString())` (eager, method call) is subtle and easily missed in code review. Several approaches help: overload the `debug()` method so one version takes `String` (eager) and another takes `Supplier<String>` (lazy), relying on overload resolution to pick the right one. For added safety, give the lazy version a distinct method name like `debugLazy()`. In the overloaded version, the eager call `log.debug(expensiveToString())` evaluates the string eagerly before passing it, which the overload resolution correctly routes to the `String`-taking overload — the developer doesn't get lazy evaluation but also doesn't accidentally degrade performance. The `Supplier` pattern itself is correct; the challenge is making the API hard to misuse through naming and overloading.

**Q: You have a function `public void process(Consumer<String> handler)` that is called thousands of times per second. Each call creates a new lambda `s -> doSomething(s)`. The allocation rate causes GC pressure. How do you optimize without changing the API?**

A: Extract the lambda to a static final field so that it is created once and cached forever by the `LambdaMetafactory`. If `doSomething` is a static method, the lambda is non-capturing: `private static final Consumer<String> HANDLER = s -> doSomething(s);`. Now every call to `process(HANDLER)` passes the same cached instance with zero allocation. If `doSomething` is an instance method, use an instance method reference `this::doSomething` — it captures `this` but may still be cached more effectively than a capturing lambda, or extract the instance state into the method parameter so the lambda becomes non-capturing. The optimization principle is: if the same lambda behavior is used repeatedly, store it in a field rather than recreating it at every call site.

**Q: A lambda `x -> compute(x)` is used in a `Stream.flatMap()`. `compute()` returns `Stream.empty()` for some inputs, which is correct behavior. However, the stream is sometimes `null` instead of empty, causing NPE. How do you enforce the contract?**

A: The `flatMap()` method throws `NullPointerException` if the mapping function returns `null` because the `Stream` contract prohibits null elements. The lambda must never return `null` from a `flatMap()` mapping function — use `Stream.empty()` for the no-result case. The safer approach is to separate the mapping and flattening: `.map(this::compute).filter(Objects::nonNull).flatMap(Function.identity())`, where `compute()` is allowed to return `null` (meaning no result) and the filter removes nulls before flattening. This pattern makes the null possibility explicit in the code and prevents the NPE from propagating through the stream pipeline. If `compute()` is controlled by a different team, document the contract clearly: the function must return non-null `Stream` instances, using `Stream.empty()` for empty results.

**Q: A team writes lambdas that access mutable instance fields without synchronization. Under load, the fields have wrong values. The team blames lambdas for not being thread-safe. Is the lambda the problem?**

A: Lambdas themselves are not the problem — they are just syntax for anonymous functions and introduce no thread-safety guarantees beyond what the underlying code provides. The bug is shared mutable state accessed from multiple threads without synchronization, regardless of whether the access happens in a lambda, an anonymous inner class, or a regular method. In the common anti-pattern `items.parallelStream().forEach(item -> results.add(process(item)))`, the lambda captures `results` which is a shared `ArrayList` with no thread-safe semantics — multiple threads call `add()` concurrently, corrupting the list's internal structure. The fix is to use a stateless approach: `items.parallelStream().map(item -> process(item)).collect(toList())`, which lets the stream framework handle thread-safe accumulation. The lambda is not the source of the thread-safety problem; it is merely the vehicle that exposes the existing unsynchronized access.

---

## Interview Questions

**What is a lambda expression in Java?** A lambda expression is an anonymous function that can be treated as a first-class value — passed as an argument, returned from a method, or stored in a variable. It provides concise syntax for implementing the single abstract method of a functional interface using the form `(parameters) -> { body }`. Lambdas enable functional programming patterns like behavior parameterization and are the foundation for the Stream API, `Optional`, and `CompletableFuture`.

**What is the difference between a lambda and an anonymous inner class?** Lambdas use `invokedynamic` compilation (no separate `.class` file, lower memory footprint) while anonymous inner classes compile to individual class files. Lambdas cannot declare instance fields or additional methods; anonymous classes can. The `this` keyword in a lambda refers to the enclosing class, whereas in an anonymous inner class it refers to the anonymous instance. Non-capturing lambdas are cached as singletons with zero allocation per use; anonymous inner classes allocate a new instance every time.

**What does "effectively final" mean in the context of lambdas?** A variable is effectively final if its value is never changed after initialization, even without the `final` keyword. Lambdas can only capture local variables that are effectively final because the captured value is copied into the lambda's heap object when created — if the local variable could change, the lambda and the enclosing method would have inconsistent copies. Instance fields and static fields do not have this restriction because they are always accessed through their declaring class rather than being copied.

**What is variable capture and how does it work under the hood?** Variable capture occurs when a lambda accesses variables from the enclosing lexical scope. Local variables are copied into the lambda's generated implementation class fields when the lambda is created. Instance fields are captured via the enclosing `this` reference, meaning the generated class holds a reference to the enclosing object. Under the hood, the `LambdaMetafactory` generates an implementation class at runtime that holds captured values as instance fields and implements the functional interface by calling the lambda body with those captured values available.

**How are lambdas compiled and executed?** The Java compiler translates each lambda expression into an `invokedynamic` instruction in the bytecode with a reference to `LambdaMetafactory` as the bootstrap method. At runtime, the first invocation triggers the bootstrap method which uses `MethodHandle` APIs to spin the lambda implementation class, link it into a `CallSite`, and cache it permanently. Subsequent invocations jump directly to the linked method handle without any bootstrap overhead, reflection, or allocation for non-capturing lambdas.

**What is a method reference and how does it differ from a lambda?** A method reference (`String::length`, `Integer::parseInt`) is a shorthand for a lambda that delegates directly to an existing method. It is more concise than an equivalent lambda and fails at compile time if the method signature does not match the functional interface. The four types are `Class::staticMethod`, `instance::instanceMethod`, `Class::instanceMethod` (where the first parameter becomes the receiver), and `Class::new` (constructor reference). Method references are typically non-capturing when static and share the same `invokedynamic` compilation mechanism as lambdas.

**Can lambdas have checked exceptions? How do you handle them?** No standard functional interface in `java.util.function` declares checked exceptions in its abstract method signature, so lambdas that throw checked exceptions produce compile errors. Common solutions include wrapping the checked-exception-throwing call in a try-catch block that rethrows as an unchecked `RuntimeException`, creating a custom `@FunctionalInterface` that declares the checked exception, or using a utility adapter method that converts a throwing function into a standard lambda-friendly one.

**What is the difference between a capturing and non-capturing lambda, and why does it matter for performance?** A non-capturing lambda does not reference any variables from the enclosing scope beyond its parameters and is cached by the JVM as a single static instance with zero allocation per invocation. A capturing lambda references local variables, parameters, or `this`, and creates a new instance on the heap each time the lambda expression is evaluated. In performance-critical hot paths, capturing lambdas create allocation pressure that can cause GC pauses, while non-capturing lambdas have zero allocation cost.

**How do you make a lambda serializable?** Lambdas are not serializable by default because the generated implementation class does not implement `Serializable`. To make a lambda serializable, cast it to the intersection type: `(Function<String, Integer> & Serializable) s -> s.length()`. For distributed systems like Apache Spark or Hazelcast, extract the logic to a class that implements `Serializable` and use method references: `(Function<String, Integer> & Serializable) String::length`. Serializable lambdas capture their enclosing scope, which must also be fully serializable.

**What are the limitations of lambdas compared to anonymous inner classes?** Lambdas cannot declare instance fields or additional methods, cannot shadow enclosing variables (no new scope), can only implement single-method interfaces (functional interfaces), cannot refer to `this` as the lambda instance itself, and have less flexibility with generic type parameters. Anonymous inner classes are still the correct choice when the implementation requires additional state, multiple interface methods, or `this`-referential behavior.

---

## Developer Recommendations

- **Prefer non-capturing lambdas in hot paths** — The JVM caches them as static singletons with zero allocation after the first invocation. Capturing lambdas allocate a new object every time the lambda expression is evaluated, which in a loop processing 1 million items creates 1 million garbage objects. Transform capturing lambdas by extracting instance state into method parameters and using method references: `items.forEach(this::process)` instead of `items.forEach(item -> process(item, config))`.
- **Use method references over lambdas for single-method delegation** — They are more readable, fail earlier at compile time with clearer error messages, and often have better allocation characteristics. `list.sort(Comparator.comparingInt(Order::getTotal))` communicates intent more directly than `list.sort((a, b) -> Integer.compare(a.getTotal(), b.getTotal()))`. Method references also survive refactoring better — renaming the referenced method produces a compile error.
- **Extract multi-line lambdas into named methods** — Debug stack traces show `lambda$methodName$N` for inline lambdas, making it difficult to identify which lambda threw the exception. A lambda with five or more lines, multiple conditionals, or try-catch blocks should be a named method referenced as `stream.map(this::processOrder)`. This improves stack trace readability, allows unit testing, and documents the transformation with a meaningful method name.
- **Keep lambdas stateless with parallel streams** — The stream framework partitions data across multiple threads, and a lambda that increments a shared counter or adds to a shared list introduces data races. Use `map()` for stateless transformations and `collect()` for thread-safe accumulation: `stream.map(this::transform).collect(toList())` is correct and performant.
- **Avoid ambiguous method overloads with different functional interfaces** — Lambdas rely on target-type inference. If a class overloads `filter(Predicate<T>)` and `filter(Function<T,R>)`, calling `filter(x -> doSomething(x))` fails to compile due to ambiguity. Use distinct method names: `filterByCondition(Predicate)` and `transform(Function)`.
- **Profile lambda allocation before investing in optimization** — Many developers assume lambdas are expensive when they are actually negligible compared to I/O, database, or network costs. Use a profiler (Async Profiler, Java Flight Recorder) to confirm that lambda allocation appears as a significant fraction of CPU time or GC pressure before refactoring.
- **Use IntStream.range() to avoid loop variable capture** — Instead of `for (int i = 0; i < n; i++) { int copy = i; executor.submit(() -> process(copy)); }`, use `IntStream.range(0, n).forEach(i -> executor.submit(() -> process(items.get(i))))` where `i` is unique per iteration by virtue of being a lambda parameter.
