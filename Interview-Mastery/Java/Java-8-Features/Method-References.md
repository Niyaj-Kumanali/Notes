# Java Method References

---

## Overview

- **Definition**
  - Method references are shorthand syntax for lambda expressions that delegate directly to an existing method, using the `::` operator.
  - They are not a new feature — they compile to the same `invokedynamic` bytecode as lambdas — but provide more readable, more concise code when a lambda's body is a single method call.
  - The compiler infers the functional interface's target type and maps the method reference's signature to the interface's single abstract method.

- **Why It Exists**
  - Before Java 8, passing behavior required anonymous inner classes or named implementations, both of which introduced boilerplate even for trivial delegation.
  - Lambdas solved the boilerplate problem but introduced a readability concern: `s -> s.length()` reads as "take s and return s.length()", while `String::length` reads as "the length method of String" — a direct statement of intent.
  - Method references also fail faster at compile time: if the referenced method does not exist or its signature is incompatible, the compiler reports the error immediately, whereas a lambda with the same body would compile and only fail if the method is actually unreachable.

- **Key Concepts**
  - The `::` operator binds a method name to a target (class, instance, or constructor) without invoking it, producing a functional interface instance.
  - The compiler uses type inference to match the method signature against the functional interface's abstract method — overloading resolution follows the same rules as lambda target-type inference.
  - Method references are always non-capturing when the target is static, and capturing (with one object allocation per evaluation) when the target is an instance.

---

## Core Concepts

### The Four Kinds of Method References

- **Static Method Reference** `Class::staticMethod`
  - Maps to a lambda where all parameters are forwarded to a static method: `Math::max` → `(a, b) -> Math.max(a, b)`.
  - The functional interface method and the static method must have compatible parameter lists — the interface's parameters become the static method's arguments.
  - Always non-capturing — the `LambdaMetafactory` generates a single cached instance on first use with zero allocation per invocation.

```java
Function<String, Integer> parseInt = Integer::parseInt;
// Equivalent: (String s) -> Integer.parseInt(s)

BiFunction<Integer, Integer, Integer> max = Math::max;
// Equivalent: (Integer a, Integer b) -> Math.max(a, b)

// Common pattern: Comparator factory
List<String> names = Arrays.asList("Alice", "Bob", "Charlie");
names.sort(Comparator.comparingInt(String::length));
```

- **Instance Method Reference of a Specific Object** `instance::method`
  - The instance is captured when the method reference is created, and the functional interface's parameters become arguments to the instance method.
  - The captured instance reference prevents GC of that object until the method reference itself becomes unreachable — a memory leak vector if the reference is long-lived.
  - Allocates one object per evaluation (the lambda implementation class instance that holds the captured instance reference).

```java
String prefix = "Mr. ";
Function<String, String> greeter = prefix::concat;
// Equivalent: (String name) -> prefix.concat(name)
// Captures: prefix (a local variable)

Consumer<String> printer = System.out::println;
// Equivalent: (String s) -> System.out.println(s)
// Captures: System.out (the PrintStream instance)

// Real-world: logging with captured logger
Logger log = LoggerFactory.getLogger(OrderService.class);
Consumer<String> infoLog = log::info;
// The log reference is captured — log must not be GC'd while infoLog is reachable
```

- **Instance Method Reference of an Arbitrary Object of a Particular Type** `Class::instanceMethod`
  - The first parameter of the functional interface becomes the receiver (the object on which the method is called), and remaining parameters become the method's arguments.
  - `String::length` → `s -> s.length()` — the `String` parameter becomes the receiver.
  - `String::isEmpty` → `s -> s.isEmpty()` — a `Predicate<String>` that checks if a string is empty.
  - `String::compareToIgnoreCase` → `(a, b) -> a.compareToIgnoreCase(b)` — a `Comparator<String>`.
  - The receiver object is passed as an argument at call time, so the method reference itself captures nothing and is cached as a static singleton — the same allocation profile as a static method reference.

```java
// Predicate: first param becomes receiver
Predicate<String> isEmpty = String::isEmpty;
// Equivalent: (String s) -> s.isEmpty()

// Comparator: first param is receiver, second is argument
Comparator<String> caseInsensitive = String::compareToIgnoreCase;
// Equivalent: (String a, String b) -> a.compareToIgnoreCase(b)

// Function: first param is receiver, return value is method result
Function<String, Integer> length = String::length;
// Equivalent: (String s) -> s.length()

// BiFunction: first param is receiver, second is argument, return is result
BiFunction<String, Integer, Character> charAt = String::charAt;
// Equivalent: (String s, Integer i) -> s.charAt(i)

// Map sort with Comparator via method reference
Map<String, Integer> scores = new HashMap<>();
scores.entrySet().stream()
      .sorted(Map.Entry.comparingByKey(String::compareToIgnoreCase))
      .forEach(e -> System.out.println(e.getKey() + ": " + e.getValue()));
```

- **Constructor Reference** `Class::new`
  - Resolves the constructor that matches the functional interface's parameter list — `Supplier<ArrayList>` → `() -> new ArrayList()`, `Function<String, Integer>` → `value -> new Integer(value)` (deprecated but illustrates the pattern).
  - The compiler selects the correct overloaded constructor based on the functional interface signature — a mismatch in parameter count or type produces a compile-time error.
  - Constructor references are non-capturing (no external variables to capture) and cached by the `LambdaMetafactory` as singletons.

```java
// No-arg constructor
Supplier<List<String>> listFactory = ArrayList::new;
List<String> list = listFactory.get();
// Equivalent: () -> new ArrayList<>()

// Single-arg constructor
Function<String, File> fileFactory = File::new;
File f = fileFactory.apply("/tmp/data.txt");
// Equivalent: (String path) -> new File(path)

// Multi-arg constructor — use array syntax for arrays
IntFunction<int[]> arrayFactory = int[]::new;
int[] arr = arrayFactory.apply(10);
// Equivalent: (int size) -> new int[size]

// Real-world: DTO mapping with constructor references
record OrderDTO(String id, BigDecimal amount) {}

Function<String, OrderDTO> orderFactory = OrderDTO::new;
OrderDTO dto = orderFactory.apply("ORD-001");

// Constructor reference with stream
List<String> nameStrings = Arrays.asList("Alice", "Bob");
Stream<Person> people = nameStrings.stream().map(Person::new);
// Equivalent to: nameStrings.stream().map(name -> new Person(name))
```

### Method Reference vs Lambda: When to Use Which

- **Prefer method references when:**
  - The lambda body is a single method call with the same parameters as the lambda's parameter list.
  - The intent is better expressed by naming the method being called: `Integer::parseInt` vs `s -> Integer.parseInt(s)`.
  - The code is in a hot path — static and `Class::instanceMethod` references are non-capturing and cached, while a lambda that calls the same method may be capturing if it references external variables.
  - The method name communicates domain intent: `orders.stream().map(Order::getTotal)` is self-documenting; `orders.stream().map(o -> o.getTotal())` is slightly more verbose with no additional clarity.

- **Prefer lambdas when:**
  - The logic involves more than a single method call (multiple statements, conditionals, or method chaining).
  - You need to transform or combine arguments before passing them to the method.
  - The method is overloaded and the compiler cannot resolve the correct overload from the functional interface signature alone — a lambda gives the compiler more type information through target-type inference.
  - You need to call methods on different objects for different parameters.

```java
// Method reference — clear and concise
List<BigDecimal> totals = orders.stream()
    .map(Order::getTotal)
    .collect(Collectors.toList());

// Lambda — necessary when combining method calls
List<BigDecimal> taxedTotals = orders.stream()
    .map(o -> o.getTotal().multiply(BigDecimal.valueOf(1.08)))
    .collect(Collectors.toList());

// Lambda — necessary when calling multiple methods
List<String> summaries = orders.stream()
    .map(o -> o.getId() + ": " + o.getTotal())
    .collect(Collectors.toList());

// Method reference fails for overloaded methods
// If Order has getTotal() and getTotal(Currency), the reference Order::getTotal is ambiguous
// Use lambda: o -> o.getTotal(Currency.USD)
```

### Working with Function and BiFunction

- **Function<T, R> with Method References**
  - `Function<T, R>` takes one argument of type `T` and returns `R`. A static or constructor method reference with one parameter matches directly.
  - The `andThen()` and `compose()` default methods compose method references into pipelines.

```java
// Chaining method references through composition
Function<String, Integer> parse = Integer::parseInt;
Function<Integer, String> toHex = Integer::toHexString;
Function<String, String> parseThenHex = parse.andThen(toHex);

String result = parseThenHex.apply("255"); // "ff"

// Real-world: value transformation pipeline
record Config(String key, String value) {}

Function<Config, String> extractValue = Config::value;
Function<String, Integer> parsePort = Integer::parseInt;
Function<Config, Integer> extractPort = extractValue.andThen(parsePort);

Config cfg = new Config("port", "8080");
int port = extractPort.apply(cfg); // 8080
```

- **BiFunction<T, U, R> with Method References**
  - `BiFunction<T, U, R>` takes two arguments. Static method references with two parameters map directly, as do instance method references where the first parameter is the receiver.
  - The `Class::instanceMethod` pattern naturally maps two-parameter functional interfaces when the instance method takes one argument.

```java
// Static method with two params → BiFunction
BiFunction<Integer, Integer, Integer> max = Math::max;
int m = max.apply(10, 20); // 20

// Instance method with one param → BiFunction
// String.regionMatches(int, String, int, int) has 4 params
// But String.compareTo(String) has 1 param → maps to BiFunction
BiFunction<String, String, Integer> compare = String::compareTo;
int cmp = compare.apply("apple", "banana"); // negative

// Constructor with two params
record Pair<A, B>(A first, B second) {}
BiFunction<String, Integer, Pair<String, Integer>> pairFactory = Pair::new;
Pair<String, Integer> p = pairFactory.apply("age", 30);

// Three-arg scenario — Function inside Function (currying-like)
// Java has no TriFunction, but you can nest
Function<String, Function<String, Integer>> indexOf = s -> s::indexOf;
int idx = indexOf.apply("hello world").apply("world"); // 6
```

### Comparator.comparing with Method References

- **The Pattern**
  - `Comparator.comparing(KeyExtractor)` creates a `Comparator<T>` using a `Function<T, U>` that extracts a `Comparable` key — `Comparator.comparing(Person::getName)` sorts people by name.
  - The method reference is the `KeyExtractor` — it maps each element to its sort key.

```java
List<Person> people = getPeople();

// Single-field sort
people.sort(Comparator.comparing(Person::getName));

// Reverse
people.sort(Comparator.comparing(Person::getAge).reversed());

// Multi-field chaining
people.sort(Comparator.comparing(Person::getLastName)
                     .thenComparing(Person::getFirstName)
                     .thenComparing(Person::getAge));

// Null-safe: handle null keys
people.sort(Comparator.nullsLast(
    Comparator.comparing(Person::getName, Comparator.nullsLast(Comparator.naturalOrder()))
));

// Primitive specializations avoid boxing
people.sort(Comparator.comparingInt(Person::getAge));
people.sort(Comparator.comparingLong(Person::getId));
people.sort(Comparator.comparingDouble(Person::getSalary));
```

- **Custom Comparator with Method Reference**
  - When the extracted key is not `Comparable`, provide a second argument: the comparator.
  - `Comparator.comparing(Person::getName, String.CASE_INSENSITIVE_ORDER)`.

```java
// Custom key comparator
people.sort(Comparator.comparing(Person::getName, String.CASE_INSENSITIVE_ORDER));

// Comparator on a derived property
people.sort(Comparator.comparing(
    p -> p.getAddress().getCity(), 
    String.CASE_INSENSITIVE_ORDER
));

// Chaining with reversed and thenComparing
people.sort(Comparator.comparingInt(Person::getAge)
                     .reversed()
                     .thenComparing(Person::getName));
```

---

## Common Mistakes

- **Using `instance::method` in a Hot Loop (Memory Leak)**
  - Calling `list.forEach(log::info)` where `log` is a logger instance creates a capturing method reference that holds the logger instance.
  - **Why it looks correct:** The code is concise, reads naturally, and works perfectly in unit tests with small lists. The logger is a singleton that never needs GC, so the leak is invisible — but the capturing reference allocates a new object per evaluation even for a single-line delegation.
  - In production, if `forEach` iterates millions of items, each iteration evaluates the method reference expression, allocating a new capturing instance. Use a static method reference or extract the method reference to a `static final` field to ensure non-capturing behavior.

```java
// BAD — captures `this` or a service reference
items.forEach(this::processItem);

// GOOD — extract to a static final field
private static final Consumer<String> PROCESS = MethodRefs::processItem;
items.forEach(PROCESS);

// BETTER if you need instance state
private static void processItem(String item) {
    // static method — no capture
}
```

- **Constructor Reference with the Wrong Overload**
  - The compiler resolves constructor references by matching the functional interface signature against available constructors. If the wrong overload matches, the program compiles but produces unexpected results.
  - **Why it looks correct:** The code compiles without errors and the IDE offers it as a suggestion. The `Function<String, File>` maps to `File(String)`, but if `File(URI)` was intended, the compiler picks the wrong one based on parameter type matching.
  - Always verify which constructor is resolved by checking the parameter types against the functional interface.

```java
// Intended: File(URI)
Function<URI, File> fromUri = File::new; // Correct

// If you write expecting File(URI) but type as String:
Function<String, File> fromPath = File::new; // Matches File(String), not File(URI)

// Constructor reference with ambiguous primitives
// If class has Foo(int) and Foo(Integer), a Function<Integer, Foo> may be ambiguous
// Use lambda: i -> new Foo(i) to force correct overload
```

- **Method Reference to an Overloaded Method**
  - When a class has multiple methods with the same name but different parameter lists, the compiler picks the one whose signature matches the functional interface — and if none matches, it fails.
  - **Why it looks correct:** The method name is exactly right, and the IDE auto-completes `::` followed by the method name. But if the wrong overload is resolved, the behavior differs from the lambda equivalent.
  - When ambiguity arises, fall back to a lambda with explicit types.

```java
class Service {
    void process(String s) { }
    void process(Integer i) { }
    void process(String s, int flags) { }
}

// Which process() does this match?
Consumer<String> handler = service::process; // Matches process(String) ✓

// This fails — no process() with zero args
// Runnable r = service::process; // Compile error

// If process() were overloaded with (Object), the reference service::process 
// would match Consumer<Object> instead of Consumer<String>
```

- **Static Method Reference on an Instance**
  - Writing `instance::staticMethod` is syntactically valid but misleading — the instance is ignored by the compiler, which treats it as a static method reference.
  - **Why it looks correct:** The code compiles, the method is found, and the result works. But a reader sees `instance::staticMethod` and assumes the instance is relevant, when in fact it is discarded.
  - Always use `ClassName::staticMethod` to avoid misleading readers.

```java
// MISLEADING — instance is ignored
Service svc = new Service();
Consumer<String> c = svc::staticMethod; // Compiles, but svc is discarded

// CLEAR
Consumer<String> c = Service::staticMethod;
```

- **Method Reference to a Varargs Method**
  - Varargs methods may match unexpected functional interfaces because the compiler treats the varargs parameter as an array type, potentially producing a `ClassCastException` at runtime.
  - **Why it looks correct:** The method reference compiles, and basic tests pass with small argument counts. The varargs array wrapping happens transparently, but the functional interface's contract may pass a `String[]` where `String` was expected.
  - Test with the actual functional interface context to verify the parameter count and types align.

```java
class Util {
    static void log(String format, Object... args) { }
}

// Intended: log a single format string with variable args
// But this matches Consumer<String[]> — the (String, Object...) becomes (String[])
Consumer<String[]> consumer = Util::log; // Compiles, but not what you expect

// Correct approach: lambda
Consumer<String> logger = s -> Util.log(s);
```

---

## Real-World Scenarios

### Scenario 1: E-commerce Sorting with Multiple Criteria

An e-commerce product listing page allows users to sort by price, rating, popularity, or name, with ascending/descending toggles and secondary sort criteria. The sort keys are extracted from `Product` objects using method references, combined into chained comparators at runtime.

```java
public class ProductSortService {
    private static final Map<String, Comparator<Product>> SORTERS = Map.of(
        "price-asc", Comparator.comparingBigDecimal(Product::getPrice),
        "price-desc", Comparator.comparingBigDecimal(Product::getPrice).reversed(),
        "rating", Comparator.comparingDouble(Product::getRating).reversed(),
        "popularity", Comparator.comparingInt(Product::getSalesCount).reversed(),
        "name", Comparator.comparing(Product::getName, String.CASE_INSENSITIVE_ORDER)
    );

    public List<Product> sort(List<Product> products, String sortKey) {
        Comparator<Product> primary = SORTERS.get(sortKey);
        if (primary == null) throw new IllegalArgumentException("Unknown sort: " + sortKey);
        return products.stream()
            .sorted(primary.thenComparing(Product::getId))
            .collect(Collectors.toList());
    }

    // Custom helper: Comparator.comparingBigDecimal (not in JDK)
    private static Comparator<Product> comparingBigDecimal(
            Function<Product, BigDecimal> keyExtractor) {
        return Comparator.comparing(keyExtractor, 
            (a, b) -> a.compareTo(b));
    }
}
```

- Method references like `Product::getPrice`, `Product::getRating`, `Product::getName` are the key extractors — each is a non-capturing `Class::instanceMethod` reference, cached as a single static instance and allocating zero objects per sort.
- The `SORTERS` map is a `static final` constant, initialized once at class load time with no allocation cost per request.
- The `thenComparing(Product::getId)` tiebreaker ensures deterministic ordering when primary keys are equal — `Product::getId` is another non-capturing method reference.

### Scenario 2: Batch Report Generation with Constructor References

A reporting engine reads CSV rows, transforms each row into typed DTO objects, groups by category, and generates a summary report. Constructor references eliminate boilerplate factory methods for each DTO type.

```java
record SalesRecord(String productId, LocalDate date, BigDecimal amount) {}
record CategorySummary(String category, BigDecimal total, long count) {}

public class ReportEngine {
    // Constructor reference as a factory
    private static final Function<String[], SalesRecord> RECORD_FACTORY = SalesRecord::new;

    public List<CategorySummary> generateReport(Path csvPath) throws IOException {
        return Files.lines(csvPath)
            .skip(1) // header
            .map(line -> line.split(","))
            .map(RECORD_FACTORY)
            .collect(Collectors.groupingBy(
                SalesRecord::productId,
                Collectors.teeing(
                    Collectors.mapping(SalesRecord::amount, 
                        Collectors.reducing(BigDecimal.ZERO, BigDecimal::add)),
                    Collectors.counting(),
                    (total, count) -> new CategorySummary(productIdFrom(total), total, count)
                )
            ))
            .values().stream()
            .sorted(Comparator.comparing(CategorySummary::total).reversed())
            .collect(Collectors.toList());
    }

    private static String productIdFrom(BigDecimal ignored) { return ""; }
}
```

- `SalesRecord::new` is a constructor reference that maps `String[]` to `SalesRecord` — the compiler resolves the constructor that takes `String[]` if a matching record constructor exists, or uses compact constructor parameter matching for records.
- `SalesRecord::productId`, `SalesRecord::amount` are accessor method references used as grouping and mapping extractors.
- `BigDecimal::add` is a `BinaryOperator<BigDecimal>` — a static method reference used as a reducer in the `Collectors.reducing()` collector.

### Scenario 3: Microservice Validation Chain

A request validation framework chains validation rules using method references as `Predicate` factories. Each rule is a method reference to a static validation function, composed with `and()` and `or()`.

```java
public class OrderValidator {
    // Method references as Predicates
    private static final Predicate<OrderRequest> HAS_CUSTOMER = OrderValidator::hasCustomerId;
    private static final Predicate<OrderRequest> VALID_AMOUNT = OrderValidator::isValidAmount;
    private static final Predicate<OrderRequest> NOT_BLOCKED = OrderValidator::isNotBlocked;

    private static final Predicate<OrderRequest> VALIDATION_RULES = HAS_CUSTOMER
        .and(VALID_AMOUNT)
        .and(NOT_BLOCKED);

    public ValidationResult validate(OrderRequest request) {
        if (VALIDATION_RULES.test(request)) {
            return ValidationResult.valid();
        }
        return ValidationResult.invalid(
            Stream.<Predicate<OrderRequest>>of(HAS_CUSTOMER, VALID_AMOUNT, NOT_BLOCKED)
                .filter(rule -> !rule.test(request))
                .map(rule -> rule.toString()) // simplified; real impl returns error codes
                .collect(Collectors.joining(", "))
        );
    }

    private static boolean hasCustomerId(OrderRequest r) {
        return r.customerId() != null && !r.customerId().isBlank();
    }

    private static boolean isValidAmount(OrderRequest r) {
        return r.amount() != null && r.amount().compareTo(BigDecimal.ZERO) > 0;
    }

    private static boolean isNotBlocked(OrderRequest r) {
        return !BLOCKED_CUSTOMERS.contains(r.customerId());
    }

    public static void main(String[] args) {
        // Demonstrating Predicate composition with method references
        Predicate<OrderRequest> specialRule = OrderValidator::hasCustomerId
            .and(r -> r.amount().compareTo(new BigDecimal("10000")) > 0)
            .or(OrderValidator::isNotBlocked);
    }
}
```

- Each validation method reference (`OrderValidator::hasCustomerId`) is a static method reference — non-capturing, cached forever, zero allocation per validation.
- The `VALIDATION_RULES` constant composes multiple predicates at initialization time, avoiding re-composition per request.
- The `Predicate::and`, `::or`, `::negate` default methods work seamlessly with method references because they operate on the predicates themselves, not on the method references.

---

## Scenario-Based Questions

**Q: You have a `List<Order>` where each `Order` has `getTotal()`, `getDate()`, `getStatus()`. The team writes `orders.sort((a, b) -> a.getTotal().compareTo(b.getTotal()))`. How would you improve this using method references?**

  - Replace the lambda with `Comparator.comparing(Order::getTotal)` — the method reference `Order::getTotal` is a `Function<Order, BigDecimal>` key extractor.
  - For multi-field sorts, chain with `Comparator.comparing(Order::getStatus).thenComparing(Order::getTotal)`.
  - If `getTotal()` returns `BigDecimal`, use `Comparator.comparing(Order::getTotal)` — `BigDecimal` implements `Comparable`.
  - If `getTotal()` returns a primitive, use `Comparator.comparingInt(Order::getTotalInt)`.
  - The method reference approach is safer (no integer overflow from subtraction-based comparators), more concise, and communicates intent directly.
  - For descending order, add `.reversed()`: `Comparator.comparing(Order::getTotal).reversed()`.

  > **Interview follow-up:** The candidate suggested `Comparator.comparing(Order::getTotal)`. If `getTotal()` can return `null` (e.g., unset orders), how would you handle null-safety — does `Comparator.comparing` throw `NullPointerException` for null keys, and which `Comparator.nullsFirst` / `Comparator.nullsLast` combination would you use?

**Q: A team migrates all lambdas to method references and encounters a compile error on `service::process` because the `process` method is overloaded. The lambda `s -> service.process(s)` compiled fine. What happened and how do you fix it?**

  - The method reference `service::process` is ambiguous because `service` has multiple `process` overloads — the compiler cannot determine which one matches the target functional interface.
  - The lambda `s -> service.process(s)` compiles because target-type inference from the functional interface resolves the overload based on the parameter type `s`.
  - The fix is to keep the lambda for overloaded methods, or to extract the desired overload into a non-overloaded helper method and reference that.
  - As a workaround, assign the method reference to a typed variable: `Consumer<String> handler = service::process;` — this disambiguates by specifying the target type explicitly.

  > **Interview follow-up:** The candidate suggested explicit typing via a variable declaration. In a codebase with many overloaded methods, how would you write a lint rule (ErrorProne or Checkstyle) that flags method references to overloaded methods and suggests removing the overload or using a lambda instead?

**Q: A developer writes `list.stream().map(Foo::bar).collect(toList())` where `bar` is an instance method that takes no arguments and returns a value. The method reference is non-capturing. But when the same code is written as `list.stream().map(f -> f.bar()).collect(toList())`, the lambda captures `this`. Why the difference?**

  - `Foo::bar` is the `Class::instanceMethod` form — the instance is passed as the first parameter at call time, so the method reference captures nothing and is cached as a static singleton.
  - `f -> f.bar()` inside an instance method is a lambda that may capture `this` if `bar()` is an instance method called on the lambda parameter `f` — but in this case `f.bar()` does not capture `this` because `f` is the lambda parameter itself. The misconception is that instance method calls implicitly capture `this`.
  - Actually, `f -> f.bar()` is also non-capturing in this case — the lambda only references the parameter `f`, not any external variables.
  - Both forms are equivalent in allocation behavior, but the method reference is more concise.
  - The real capturing concern arises when you write `items.forEach(item -> process(item))` inside an instance method — here `this::process` captures `this`, while `f -> f.process()` where `process` is called on `f` does not.

  > **Interview follow-up:** The candidate correctly noted that `f -> f.bar()` is non-capturing. If `bar()` were a private instance method defined in the same class and the lambda was used inside a static method, would the behavior differ — does the `this` reference affect compilation when called from a static context?

**Q: A constructor reference `ArrayList::new` is used as a `Supplier<List<String>>`. The developer needs to pre-size the ArrayList to avoid reallocation. How do you express `() -> new ArrayList<>(initialCapacity)` with a method reference?**

  - Constructor references cannot pass constructor arguments that are not derived from the functional interface parameters — `Supplier<List<String>>` has zero parameters, so it always calls the no-arg constructor.
  - To pre-size the ArrayList, you cannot use a method reference directly — use a lambda: `() -> new ArrayList<>(initialCapacity)`.
  - If the initial capacity is configurable, wrap it in a factory method and reference that: `Supplier<List<String>> factory = () -> createArrayList(initialCapacity)`.
  - Constructor references are limited to matching the functional interface's parameter list exactly; any fixed constructor arguments require a lambda.

  > **Interview follow-up:** The candidate said constructor references cannot pass fixed arguments. What if you create a static factory method `static <T> ArrayList<T> sizedList(int capacity) { return new ArrayList<>(capacity); }` and use `Supplier<List<String>> factory = () -> sizedList(INITIAL_CAPACITY)` — does this allocation pattern differ from the direct lambda `() -> new ArrayList<>(INITIAL_CAPACITY)` in terms of capture semantics?

**Q: A reporting system uses `map.getOrDefault(key, defaultValue::toString)` inside a hot loop. The `defaultValue` is a static constant. Does this method reference allocate per call?**

  - `defaultValue::toString` is an `instance::instanceMethod` reference on a static constant `defaultValue`. Because the instance is a static final field, the method reference expression is evaluated once per loop iteration and captures the static field reference.
  - However, the capturing object is created each time the method reference expression is evaluated — in a hot loop, that means one allocation per iteration.
  - Move the method reference to a `static final` field: `private static final Supplier<String> DEFAULT_STRING = defaultValue::toString;` — now the expression is evaluated once at class initialization, and the cached `Supplier` is reused with zero allocation per call.
  - Rule of thumb: any method reference expression inside a loop body allocates a new capturing instance per iteration unless it is a non-capturing form (static or `Class::instanceMethod`).

  > **Interview follow-up:** The candidate suggested extracting to a `static final` field. Does `defaultValue::toString` on a static final field allocate a new object every time the expression is evaluated in the loop, or does the JIT optimize it to a singleton after enough iterations — and what JIT flags would you use to verify the allocation profile?

**Q: A developer uses `Function.identity()` instead of a method reference like `String::valueOf` or `x -> x`. When should you use `Function.identity()` versus a method reference?**

  - `Function.identity()` returns a function that returns its input unchanged — equivalent to `t -> t`. It is a non-capturing singleton cached in the `Function` class.
  - `String::valueOf` converts any input to a `String` representation — it is not identity unless the input is already a `String`.
  - Use `Function.identity()` when you need to pass through the input unchanged (e.g., as a downstream function in `Collectors.mapping` or `Stream.map`).
  - Use a method reference like `String::toUpperCase` or `String::trim` when you need to transform the input.
  - `Function.identity()` is always exactly one static field access (singleton) while `x -> x` is a non-capturing lambda that the JVM also caches — they are equivalent in allocation and are both free after the first invocation.

  > **Interview follow-up:** The candidate explained the difference correctly. In a `.map(Function.identity()).filter(...)` pipeline, the identity map is redundant. How would you write a static analysis check that flags identity-mapped stream stages and suggests removing them?

---

## Interview Questions

**What are the four kinds of method references in Java 8?**

  - `Class::staticMethod` — delegates to a static method, parameters become arguments: `Math::max`.
  - `instance::instanceMethod` — delegates to an instance method on a captured instance: `System.out::println`.
  - `Class::instanceMethod` — first parameter becomes the receiver, remaining parameters become arguments: `String::length`.
  - `Class::new` — invokes the constructor matching the functional interface signature: `ArrayList::new`.

**When would you choose a method reference over a lambda?**

  - When the lambda body is a single method call with the same parameter list, a method reference is more concise and self-documenting.
  - When the method is static or a `Class::instanceMethod` form, the method reference is non-capturing and cached as a singleton — better allocation profile.
  - When the method is overloaded, prefer a lambda to avoid compiler ambiguity.
  - When additional logic (conditionals, chaining, argument transformation) is needed, use a lambda.

**How does the compiler resolve which constructor to use for a constructor reference?**

  - The compiler matches the functional interface's method signature (parameter count and types) against the class's constructors.
  - `Supplier<List<String>>` → no-arg constructor `ArrayList()`.
  - `Function<String, File>` → single-arg constructor `File(String)`.
  - If no matching constructor exists, the compiler emits an error. If multiple constructors match (e.g., due to type erasure or autoboxing), the most specific one is chosen following overload resolution rules.

**What is the difference between `String::length` and `s -> s.length()` in terms of capture semantics?**

  - `String::length` is a `Class::instanceMethod` reference — it is non-capturing because the receiver (the `String` instance) is provided as the first argument at call time.
  - `s -> s.length()` inside an instance method is also non-capturing if it only references the lambda parameter `s` and no external variables.
  - Both compile to the same `invokedynamic` bytecode and the JVM caches both as singletons — zero allocation per invocation.
  - The difference is readability: `String::length` directly names the method being called.

**How do you create a method reference that matches a `BiFunction<T, U, R>`?**

  - A static method with two parameters: `Math::max` → `BiFunction<Integer, Integer, Integer>`.
  - A `Class::instanceMethod` where the instance method takes one argument: `String::compareTo` → `BiFunction<String, String, Integer>`.
  - A two-argument constructor: `Pair::new` → `BiFunction<String, Integer, Pair<String, Integer>>`.
  - An `instance::instanceMethod` where the instance method takes one argument: `System.out::println` is `Consumer<String>`, not `BiFunction`.

**Can a method reference throw a checked exception? How does the compiler handle this?**

  - A method reference can throw a checked exception if the referenced method declares it, but the functional interface's abstract method must also declare it.
  - Most standard functional interfaces (`Function`, `Consumer`, `Predicate`) do not declare checked exceptions, so a method reference to a method that throws a checked exception will fail to compile.
  - The solution is the same as with lambdas: wrap in a try-catch, use a custom `@FunctionalInterface` that declares the exception, or use an adapter.

**How does the JVM optimize method references at runtime?**

  - Method references compile to the same `invokedynamic` instruction as lambdas, using `LambdaMetafactory` as the bootstrap method.
  - Static and `Class::instanceMethod` forms are non-capturing — the `LambdaMetafactory` generates a single implementation class, caches it permanently, and subsequent invocations jump directly to the linked method handle with zero allocation.
  - `instance::instanceMethod` forms capture the instance reference — they allocate a new implementation instance per evaluation of the expression, but this is a single object, not per-invocation.
  - The JIT can inline method references as aggressively as regular method calls because the `invokedynamic` call site is linked to a concrete method handle.

**What is the memory overhead of a method reference versus an anonymous inner class?**

  - A non-capturing method reference allocates one object (the cached `CallSite` target) once per call site, with zero allocation per invocation.
  - A capturing method reference allocates one object per evaluation of the expression.
  - An anonymous inner class allocates one `.class` file (loaded and stored in Metaspace permanently) and one heap instance per instantiation — the class definition cannot be unloaded, and every use creates a new heap object.
  - Method references are strictly better in both code size (no separate `.class` file) and runtime allocation (cached non-capturing forms).

---

## Developer Recommendations

- **Extract method reference expressions from hot loops**
  - A method reference like `log::info` evaluated inside a loop creates a new capturing object per iteration.
  - Store it in a `private static final` field: `private static final Consumer<String> LOG = log::info;`.
  - The field initialization evaluates the expression once, and the loop uses the cached reference with zero allocation.
  - A team discovered 500ms GC pauses every 30 seconds in a batch processor because `System.out::println` was evaluated inside a loop processing 2M records — the method reference was allocated 2M times. Moving it to a `static final` field eliminated the allocations entirely and GC pauses dropped to 20ms.

- **Prefer `Class::instanceMethod` over `instance::method` when possible**
  - `Class::instanceMethod` references are non-capturing and cached; `instance::method` references capture the instance and allocate per evaluation.
  - If you have a choice between `orders.sort(Comparator.comparing(Order::getTotal))` and `orders.sort(Comparator.comparing(orderService::getTotal))`, choose the former — it captures nothing.
  - The `Class::instanceMethod` form also makes the code more reusable: it works with any instance of that type, not just the captured one.

- **Use constructor references for dependency injection maps**
  - A map of `String` to `Supplier<?>` or `Function<String, ?>` where values are constructor references creates a pluggable factory system with zero boilerplate.
  - `Map.of("csv", CsvReport::new, "pdf", PdfReport::new)` — adding a new report type adds one map entry and one class, with no switch statement.
  - The constructor references are non-capturing and cached, so the factory map has zero allocation cost per report creation beyond the report object itself.

- **Combine method references with Comparator utilities for declarative sorting**
  - Write `Comparator.comparing(Person::getName).thenComparing(Person::getAge)` instead of implementing `compare()` manually.
  - This reads as a declarative statement: "sort by name, then by age" — the method references name the sort keys.
  - For primitive fields, use `comparingInt`, `comparingLong`, `comparingDouble` to avoid autoboxing overhead.
  - The `Comparator` utility methods are themselves implemented with method references internally, creating a chain of non-capturing references.

- **Avoid method references to overloaded methods**
  - If a method has multiple overloads, a method reference may be ambiguous or resolve to the wrong overload.
  - Fall back to a lambda with explicit parameter types when overloading is present: `s -> service.process(s)`.
  - Better yet, avoid overloading in APIs designed for method reference consumption — use distinct method names like `processString`, `processInteger`.

- **Use `MethodHandle` introspection for debugging method reference resolution**
  - When a method reference compiles but behaves unexpectedly, use `javap -v -p` to inspect the `invokedynamic` bootstrap method and see which method the compiler selected.
  - The `invokedynamic` call site's bootstrap arguments include a `MethodHandle` to the resolved method — if it's not the one you expected, the compiler resolved a different overload.
  - This technique was used by a team that found `service::process` was resolving to `process(Object)` instead of `process(String)` because the functional interface's type variable was erased to `Object`.

- **Profile before optimizing method reference allocation**
  - Non-capturing method references are effectively free. Capturing method references allocate one object per evaluation.
  - Before spending time extracting method references to `static final` fields, use a profiler (Java Flight Recorder, Async Profiler) to confirm that method reference allocation is a significant fraction of GC pressure.
  - A team optimized method references in a web request handler, reducing allocations by 40%, only to discover the bottleneck was JSON serialization (60% of CPU). The allocation reduction was wasted effort — the serialization dwarfed all other costs.
  - The right approach: profile first, identify the allocation hot spots, optimize those, and re-profile to confirm improvement.
