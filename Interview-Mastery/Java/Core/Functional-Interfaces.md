# Java Functional Interfaces

---

## 1. Executive Summary

### What Is It?
A functional interface is an interface with exactly **one abstract method** (SAM — Single Abstract Method). They are the foundation of lambda expressions and method references in Java.

### Why Does It Exist?
Before Java 8, anonymous inner classes were the only way to pass behavior:
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

### Core Functional Interfaces (java.util.function)

| Interface | Signature | Purpose |
|-----------|-----------|---------|
| `Predicate<T>` | `T → boolean` | Test a condition |
| `Consumer<T>` | `T → void` | Consume a value (side effect) |
| `Function<T,R>` | `T → R` | Transform a value |
| `Supplier<T>` | `() → T` | Supply a value (factory) |
| `UnaryOperator<T>` | `T → T` | Transform, same type |
| `BinaryOperator<T>` | `(T,T) → T` | Combine two values, same type |
| `BiPredicate<T,U>` | `(T,U) → boolean` | Test two values |
| `BiConsumer<T,U>` | `(T,U) → void` | Consume two values |
| `BiFunction<T,U,R>` | `(T,U) → R` | Transform two values |

### Primitive Specializations
To avoid autoboxing overhead, there are specialized variants:

| For `int` | For `long` | For `double` |
|-----------|------------|--------------|
| `IntPredicate` | `LongPredicate` | `DoublePredicate` |
| `IntConsumer` | `LongConsumer` | `DoubleConsumer` |
| `IntFunction<R>` | `LongFunction<R>` | `DoubleFunction<R>` |
| `IntSupplier` | `LongSupplier` | `DoubleSupplier` |
| `IntUnaryOperator` | `LongUnaryOperator` | `DoubleUnaryOperator` |
| `IntBinaryOperator` | `LongBinaryOperator` | `DoubleBinaryOperator` |
| `ToIntFunction<T>` | `ToLongFunction<T>` | `ToDoubleFunction<T>` |
| `IntToLongFunction` | `LongToIntFunction` | etc. |

---

## 2. Core Theory

### @FunctionalInterface Annotation
```java
@FunctionalInterface
public interface Predicate<T> {
    boolean test(T t);
}
```

The annotation is optional but recommended — it makes intent clear and causes compilation failure if a second abstract method is added.

### Relationship to Lambdas
Every lambda expression is a syntactic shorthand for implementing a functional interface:

```java
// Lambda
Function<String, Integer> f = s -> s.length();

// Equivalent anonymous class
Function<String, Integer> f = new Function<>() {
    @Override
    public Integer apply(String s) {
        return s.length();
    }
};
```

### Default and Static Methods in Functional Interfaces
A functional interface can have `default` and `static` methods — only the single abstract method matters:

```java
@FunctionalInterface
public interface Comparator<T> {
    int compare(T o1, T o2);                    // Single abstract method
    
    default Comparator<T> reversed() { ... }    // Default method
    static <T> Comparator<T> nullsFirst(Comparator<T> c) { ... }  // Static method
}
```

### Method References as Functional Interface Implementations
```java
// Lambda
Function<String, Integer> f = s -> Integer.parseInt(s);

// Method reference (shorthand)
Function<String, Integer> f = Integer::parseInt;

// Types:
// 1. Static method:         Class::staticMethod
// 2. Instance method:       instance::method
// 3. Instance method on class: Class::method (first arg becomes receiver)
// 4. Constructor:           Class::new
```

---

## 3. Under-the-Hood Deep Dive

### Lambda Compilation

Lambdas are NOT compiled as anonymous inner classes. Instead, Java uses `invokedynamic` (JVM instruction added in Java 7):

```
Source:  list.filter(s -> s.length() > 3)
         ↓
Bytecode: invokedynamic #bootstrapMethod
         ↓
Runtime: LambdaMetafactory.metafactory()
         ↓
         Generates CallSite → Function<String, Boolean> at runtime
```

**Benefits of invokedynamic:**
- No separate .class file per lambda
- Lambda is generated once and cached
- Bootstrap method only runs once
- Lower memory footprint than anonymous classes
- Captured variables use `MethodHandle` not reflection

### Variable Capture
Lambdas can capture:
- `final` local variables (effectively final since Java 8)
- Static fields
- Instance fields

```java
public class Example {
    private int instanceField = 1;
    
    public void method() {
        int localVar = 2; // must be effectively final
        
        Supplier<Integer> s = () -> instanceField + localVar;
        // Captures: this (for instanceField), localVar (value copy)
    }
}
```

**How it works:** Captured local variables are copied into the lambda object (stored in heap). Instance fields use captured `this` reference.

### Serialization
Functional interfaces can extend `Serializable`, but lambdas are not serializable by default:
```java
// Cast to Serializable to make it serializable
Function<String, Integer> f = (Function<String, Integer> & Serializable) s -> s.length();
```

### Performance

| Mechanism | Memory | Speed | Notes |
|-----------|--------|-------|-------|
| Anonymous class | ~100 bytes per instance | Fast | Creates .class file |
| Lambda (not capturing) | ~1 object (static) | Fastest | Single instance, cached |
| Lambda (capturing) | ~1 object per call | Fast | New instance each time |
| Method reference | ~1 object (static) | Fastest | Same as non-capturing lambda |

---

## 4. Production Code Examples

### 4.1 Predicate — Validation Pipeline

```java
// Combined validation rules
public class UserValidator {
    private final List<Predicate<User>> rules = List.of(
        u -> u.getEmail() != null && u.getEmail().contains("@"),
        u -> u.getAge() >= 18,
        u -> u.getName() != null && !u.getName().isBlank(),
        u -> !BlockedEmailDomains.contains(extractDomain(u.getEmail()))
    );

    public ValidationResult validate(User user) {
        List<String> failures = rules.stream()
            .filter(rule -> !rule.test(user))
            .map(rule -> "Validation failed: " + rule)
            .collect(Collectors.toList());
        
        return failures.isEmpty()
            ? ValidationResult.valid()
            : ValidationResult.invalid(failures);
    }
}
```

### 4.2 Function — Transformation Pipeline

```java
@Service
public class PriceCalculationPipeline {
    private final List<UnaryOperator<Price>> priceModifiers;

    public PriceCalculationPipeline(PricingConfig config) {
        this.priceModifiers = List.of(
            p -> p.applyMarkup(config.getMarkupPercent()),
            p -> p.applyTax(config.getTaxRate()),
            p -> p.applyDiscount(config.getCouponCode()),
            p -> p.roundToNearestCent()
        );
    }

    public Price calculate(Price basePrice) {
        return priceModifiers.stream()
            .reduce(
                UnaryOperator.identity(),
                (combined, next) -> combined.andThen(next)
            )
            .apply(basePrice);
    }
}
```

### 4.3 Supplier — Lazy Configuration

```java
@Component
public class ConfigProvider {
    private final Supplier<DatabaseConfig> databaseConfig = 
        Memoizer.memoize(this::loadDatabaseConfig);
    
    private final Supplier<List<RateLimitRule>> rateLimitRules =
        Memoizer.memoize(this::loadRateLimitRules);

    public DatabaseConfig getDatabaseConfig() {
        return databaseConfig.get();
    }
}

// Memoizer utility for thread-safe lazy init
public class Memoizer {
    public static <T> Supplier<T> memoize(Supplier<T> delegate) {
        return new Supplier<>() {
            private volatile T value;
            
            @Override
            public T get() {
                T result = value;
                if (result == null) {
                    synchronized (this) {
                        result = value;
                        if (result == null) {
                            value = result = delegate.get();
                        }
                    }
                }
                return result;
            }
        };
    }
}
```

### 4.4 Consumer — Event Bus

```java
@Component
public class EventBus {
    private final Map<Class<?>, List<Consumer<?>>> listeners = new ConcurrentHashMap<>();

    public <T> void register(Class<T> eventType, Consumer<T> listener) {
        listeners.computeIfAbsent(eventType, k -> new CopyOnWriteArrayList<>())
            .add(listener);
    }

    @SuppressWarnings("unchecked")
    public <T> void publish(T event) {
        List<Consumer<?>> consumers = listeners.get(event.getClass());
        if (consumers != null) {
            consumers.forEach(c -> ((Consumer<T>) c).accept(event));
        }
    }
}
```

### 4.5 Bad vs Good — Functional Interface Design

```java
// BAD: Too many methods in functional interface
@FunctionalInterface
public interface Processor {
    void process(Data data);
    default void preProcess() { /* ... */ }
    default void postProcess() { /* ... */ }
    static Processor chain(Processor a, Processor b) { /* ... */ }
}
// This is fine actually — only one abstract method

// BAD: Functional interface with checked exception
@FunctionalInterface
public interface FileProcessor {
    void process(Path path) throws IOException; // Lambda can't easily handle this
}
// Usage requires ugly try-catch in lambda

// BETTER: Wrap checked exception in runtime
@FunctionalInterface
public interface FileProcessor {
    void process(Path path); // unchecked
}

// BEST: Use standard functional interface where possible
// Don't create custom — use UnaryOperator<Path> or Consumer<Path>
```

---

## 5. Real-World Scenarios (5 Key)

### Scenario 1: Lambda Serialization in Distributed Systems
**Problem:** Lambdas used in Spark/Kafka Streams applications must be serializable, but default lambdas are not.

**Solution:** `(Function<String, Integer> & Serializable) s -> s.length()` or serialize the logic as data (Strategy pattern as a config).

### Scenario 2: Checked Exception Hell in Streams
**Problem:** Stream's functional interfaces don't declare checked exceptions. Calling `Files.lines()` in `.map()` requires try-catch.

**Solution:** Create a wrapper:
```java
@FunctionalInterface
public interface ThrowingFunction<T, R> {
    R apply(T t) throws Exception;
    
    static <T, R> Function<T, R> wrap(ThrowingFunction<T, R> f) {
        return t -> {
            try { return f.apply(t); }
            catch (Exception e) { throw new RuntimeException(e); }
        };
    }
}
// Usage: .map(ThrowingFunction.wrap(p -> Files.lines(p)))
```

### Scenario 3: Instance Field Capture in Lambdas
**Problem:** Lambda captures `this` when accessing instance fields, which can cause memory leaks in inner classes/listeners.

**Solution:** Assign instance field to a local variable before lambda:
```java
// Captures this → prevents GC of enclosing object
executor.submit(() -> doWork(field));

// Captures local var — no this reference
String localField = this.field;
executor.submit(() -> doWork(localField));
```

### Scenario 4: Lambda Overload Ambiguity
**Problem:** Method overloaded with different functional interfaces:
```java
void execute(Predicate<String> p);
void execute(Function<String, Integer> f);
execute(s -> s.length() > 3); // Ambiguous! Both match
```
**Solution:** Cast to disambiguate:
```java
execute((Predicate<String>) s -> s.length() > 3);
```

### Scenario 5: Caching Expensive Supplier Results
**Problem:** Expensive computation in `Supplier` called many times. Need lazy-cached result.

**Solution:** Memoizer pattern (see Section 4.3) — thread-safe, lazy, cached.

---

## 6. Comparison

| Aspect | Anonymous Class | Lambda | Method Reference |
|--------|----------------|--------|-----------------|
| Syntax | Verbose | Concise | Most concise |
| `this` | Refers to inner class | Refers to enclosing class | N/A |
| Captures | Full | Effectively final | Effectively final |
| Compilation | .class file | invokedynamic | invokedynamic |
| Memory | New instance per use | Cached (non-capturing) | Cached |
| Serialization | If implements | With cast | With cast |
| Readability | Low | High | Highest |

---

## Cheat Sheet

```
═══ FUNCTIONAL INTERFACES ══════════════════════════════════════

┌─ CORE 6 ───────────────────────────────────────────────────┐
│ Predicate<T>   T → boolean    .test(T)                     │
│ Consumer<T>    T → void       .accept(T)                   │
│ Function<T,R>  T → R          .apply(T)                    │
│ Supplier<T>    () → T         .get()                       │
│ UnaryOperator<T>  T → T       .apply(T)                    │
│ BinaryOperator<T>  (T,T)→T    .apply(T,T)                  │
└─────────────────────────────────────────────────────────────┘

┌─ COMPOSITION ──────────────────────────────────────────────┐
│ Predicate:   .and()  .or()  .negate()  .isEqual()          │
│ Function:    .andThen()  .compose()  .identity()            │
│ Consumer:    .andThen()                                     │
│ Comparator:  .thenComparing()  .reversed()                  │
└─────────────────────────────────────────────────────────────┘

┌─ RULES ────────────────────────────────────────────────────┐
│ • Lambda = SAM interface implementation                     │
│ • @FunctionalInterface enforces single abstract method      │
│ • Prefer standard interfaces over custom ones               │
│ • Use primitive variants (IntPredicate) in hot paths        │
│ • Capture only effectively final variables                  │
│ • Wrap checked exceptions for stream use                   │
└─────────────────────────────────────────────────────────────┘
```
