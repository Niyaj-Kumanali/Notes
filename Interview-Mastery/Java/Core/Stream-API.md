# Java Stream API

---

## 1. Executive Summary

### What Is It?
The Stream API (Java 8+) provides a functional, declarative way to process sequences of data (collections, arrays, I/O channels) using a pipeline of operations: **source → intermediate operations → terminal operation**.

### Why Does It Exist?
Before streams, data processing meant imperative loops — verbose, error-prone, hard to parallelize. Streams provide:
- **Declarative code** — what, not how
- **Composability** — chain operations
- **Parallelism** — single `.parallel()` call
- **Lazy evaluation** — compute only what's needed

### Real-World Use Cases
- **Data transformation** — filter → map → collect
- **Reporting** — group by, aggregate, summarize
- **Batch processing** — read, transform, write
- **Paginated API responses** — filter, sort, paginate
- **Real-time analytics** — windowed operations on streams
- **ETL pipelines** — extract, transform, load

### When to Use
- Processing collections (lists, sets, maps)
- Need to chain multiple operations (filter → map → sort → collect)
- Want parallel processing for performance
- Code readability matters more than raw performance
- Data fits in memory (for in-memory streams)

### When NOT to Use
- Simple loops (3 lines or fewer) — a for-each is clearer
- Performance-critical hot paths — streams have overhead
- Primitive arrays in hot paths — use plain loops
- Checked exceptions in lambdas — cumbersome to handle
- Mutable state accumulation — use traditional loops
- Very large datasets (don't fit in memory) — use external iteration or database

---

## 2. Core Theory

### Stream Pipeline Structure

```
Source               Intermediate Ops         Terminal Op
┌────────┐        ┌───────────────┐        ┌──────────┐
│ List   │ ──────→│ .filter()     │ ──────→│ .collect()│
│ Array  │        │ .map()        │        │ .forEach()│
│ Stream │        │ .sorted()     │        │ .reduce() │
└────────┘        │ .distinct()   │        │ .count()  │
                  │ .limit()      │        │ .findFirst│
                  └───────────────┘        └──────────┘
```

### Stream Characteristics
- **Not a data structure** — doesn't store data
- **Lazy** — intermediate operations are evaluated only when a terminal operation is invoked
- **Consumable** — can be used only once (throws IllegalStateException on reuse)
- **Parallelizable** — `.parallel()` enables multi-threaded processing
- **Stateless** preferred — avoid stateful lambdas (exceptions for `distinct()`, `sorted()`, `limit()`)

### Creating Streams

```java
// From collections
List<String> list = List.of("a", "b", "c");
Stream<String> stream = list.stream();
Stream<String> parallelStream = list.parallelStream();

// From arrays
String[] array = {"a", "b", "c"};
Stream<String> arrayStream = Arrays.stream(array);
Stream<String> fullArrayStream = Arrays.stream(array, 0, 2);

// From values
Stream<String> of = Stream.of("a", "b", "c");
Stream<Integer> ofNullable = Stream.ofNullable(null); // empty stream

// From functions (infinite)
Stream<Integer> iterate = Stream.iterate(0, n -> n + 1);
Stream<Integer> iterateWithLimit = Stream.iterate(0, n -> n < 100, n -> n + 1);
Stream<Double> generate = Stream.generate(Math::random);
Stream.iterate(0, n -> n + 1).limit(10)
     .collect(Collectors.toList()); // [0,1,2,3,4,5,6,7,8,9]

// From builder
Stream<String> built = Stream.<String>builder()
    .add("a").add("b").add("c").build();

// From file
Stream<String> lines = Files.lines(Paths.get("file.txt"));
Stream<Path> paths = Files.list(Paths.get("dir"));
```

### Intermediate Operations (lazy — return a Stream)

| Operation | Type | Description |
|-----------|------|-------------|
| `filter(Predicate)` | Stateless | Keep elements matching predicate |
| `map(Function)` | Stateless | Transform each element |
| `flatMap(Function)` | Stateless | Flatten nested streams |
| `distinct()` | Stateful | Remove duplicates (uses equals) |
| `sorted()` | Stateful | Sort (natural order) |
| `sorted(Comparator)` | Stateful | Sort with comparator |
| `peek(Consumer)` | Stateless | Debug — view each element (use sparingly) |
| `limit(long)` | Stateful | Truncate to max size |
| `skip(long)` | Stateful | Discard first N elements |
| `takeWhile(Predicate)` | Stateless | Take elements while true (Java 9+) |
| `dropWhile(Predicate)` | Stateless | Drop elements while true (Java 9+) |

### Terminal Operations (eager — produce result or side effect)

| Operation | Returns | Description |
|-----------|---------|-------------|
| `forEach(Consumer)` | void | Side-effect per element |
| `forEachOrdered(Consumer)` | void | Maintain encounter order in parallel |
| `toList()` | `List<T>` | Collect to list (Java 16+) |
| `collect(Collector)` | `R` | Collect using collector |
| `collect(Supplier, BiConsumer, BiConsumer)` | `R` | Custom mutable reduction |
| `toArray()` | `Object[]` | Collect to array |
| `toArray(IntFunction)` | `T[]` | Collect to typed array |
| `reduce(BinaryOperator)` | `Optional<T>` | Associative reduction |
| `reduce(T, BinaryOperator)` | `T` | Reduction with identity |
| `reduce(U, BiFunction, BinaryOperator)` | `U` | Typed reduction |
| `min(Comparator)` | `Optional<T>` | Minimum element |
| `max(Comparator)` | `Optional<T>` | Maximum element |
| `count()` | `long` | Element count |
| `anyMatch(Predicate)` | `boolean` | Any element matches? |
| `allMatch(Predicate)` | `boolean` | All elements match? |
| `noneMatch(Predicate)` | `boolean` | No elements match? |
| `findFirst()` | `Optional<T>` | First element (respects order) |
| `findAny()` | `Optional<T>` | Any element (parallel-friendly) |

### Collectors (java.util.stream.Collectors)

```java
// To collections
.collect(Collectors.toList())
.collect(Collectors.toSet())
.collect(Collectors.toCollection(ArrayList::new))
.collect(Collectors.toMap(Function<K>, Function<V>))
.collect(Collectors.toMap(Function<K>, Function<V>, BinaryOperator<V>)) // merge
.collect(Collectors.toMap(Function<K>, Function<V>, BinaryOperator<V>, Supplier<Map>))

// Grouping
.collect(Collectors.groupingBy(Function))                    // Map<K, List<V>>
.collect(Collectors.groupingBy(Function, Collector))         // Map<K, V> (downstream)
.collect(Collectors.groupingBy(Function, Supplier, Collector)) // Map with specific impl
.collect(Collectors.partitioningBy(Predicate))               // Map<Boolean, List<V>>

// Partitioning
.collect(Collectors.partitioningBy(Predicate))
.collect(Collectors.partitioningBy(Predicate, Collector))

// Joining
.collect(Collectors.joining())                          // "abc"
.collect(Collectors.joining(", "))                       // "a, b, c"
.collect(Collectors.joining(", ", "[", "]"))            // "[a, b, c]"

// Summarizing
.collect(Collectors.summarizingInt(ToIntFunction))
.collect(Collectors.averagingInt(ToIntFunction))
.collect(Collectors.summingInt(ToIntFunction))
.collect(Collectors.counting())

// Reducing
.collect(Collectors.reducing(BinaryOperator))
.collect(Collectors.mapping(Function, Collector))
.collect(Collectors.flatMapping(Function, Collector)) // Java 9+
.collect(Collectors.filtering(Predicate, Collector))  // Java 9+

// Custom
.collect(Collector.of(
    () -> new ArrayList<>(),           // supplier
    (list, item) -> list.add(item),    // accumulator
    (left, right) -> { left.addAll(right); return left; }, // combiner
    Collector.Characteristics.IDENTITY_FINISH               // characteristics
));
```

### Primitive Streams
```java
IntStream, LongStream, DoubleStream

IntStream.range(1, 10)                    // [1, 2, ..., 9]
IntStream.rangeClosed(1, 10)              // [1, 2, ..., 10]
IntStream.of(1, 2, 3)
IntStream.iterate(0, n -> n + 2).limit(5) // [0, 2, 4, 6, 8]

// Conversion
Stream<Integer> boxed = intStream.boxed();
IntStream unboxed = streamOfIntegers.mapToInt(Integer::intValue);
```

---

## 3. Under-the-Hood Deep Dive

### Lazy Evaluation
Streams are lazy — intermediate operations don't execute until a terminal operation is called.

```java
// Nothing happens here:
Stream<String> stream = list.stream()
    .filter(s -> {
        System.out.println("filtering: " + s);
        return s.length() > 3;
    })
    .map(s -> {
        System.out.println("mapping: " + s);
        return s.toUpperCase();
    });

// Only when terminal op is called:
List<String> result = stream.toList();
```

**Fusion:** The JVM can fuse adjacent intermediate operations (e.g., `filter().map()`) into a single pass — elements go through the entire pipeline one at a time, not in batches.

### Spliterator — the Engine Behind Streams

```java
public interface Spliterator<T> {
    boolean tryAdvance(Consumer<? super T> action);
    Spliterator<T> trySplit();
    long estimateSize();
    int characteristics();
}
```

- `tryAdvance()` — process one element, return false if none left
- `trySplit()` — split for parallel processing, return null if can't split
- `characteristics()` — ORDERED, DISTINCT, SORTED, SIZED, NONNULL, IMMUTABLE, CONCURRENT, SUBSIZED

**Parallelism:** `ForkJoinPool.commonPool()` splits the Spliterator, processes sub-streams in parallel, then combines results.

### Stream Pipeline Execution

```java
list.stream()
    .filter(x -> x > 5)           // 1. Create StatelessOp
    .map(x -> x * 2)              // 2. Create StatelessOp
    .sorted()                     // 3. Create StatefulOp (buffer all)
    .limit(10)                    // 4. Create StatefulOp
    .collect(toList());           // 5. Terminal — triggers pipeline
```

**Execution:**
1. Terminal op calls `evaluate()` on the pipeline
2. Pipeline wraps from bottom up: `collect` → `limit` → `sorted` → `map` → `filter` → source
3. Each operation pushes/pulls elements through the `Sink` chain
4. `sorted()` must buffer ALL elements before emitting (stateful)
5. `limit()` short-circuits when limit reached

### Performance Characteristics

| Operation | Type | Time Complexity | Memory |
|-----------|------|----------------|--------|
| `filter()` | Stateless | O(n) | O(1) |
| `map()` | Stateless | O(n) | O(1) |
| `flatMap()` | Stateless | O(n) | O(1) |
| `distinct()` | Stateful | O(n)* | O(n) |
| `sorted()` | Stateful | O(n log n) | O(n) |
| `limit()` | Stateful | O(n) | O(limit) |
| `skip()` | Stateful | O(n) | O(1) |
| `reduce()` | Terminal | O(n) | O(1) |
| `collect()` | Terminal | O(n) | O(result) |

\* assuming good hash distribution

### Stream Reuse (or Lack Thereof)

```java
Stream<String> stream = list.stream();
stream.forEach(System.out::println);
stream.forEach(System.out::println); // IllegalStateException: stream has already been operated upon or closed
```

Each stream can be consumed only once. If you need to reuse, create a Supplier:
```java
Supplier<Stream<String>> supplier = () -> list.stream();
supplier.get().forEach(...);
supplier.get().forEach(...);
```

---

## 4. Production Code Examples

### 4.1 Basic — Filter, Map, Collect

```java
// Get names of active users older than 18
List<String> activeAdultNames = users.stream()
    .filter(u -> u.isActive())
    .filter(u -> u.getAge() >= 18)
    .map(User::getName)
    .sorted()
    .collect(Collectors.toList());
```

### 4.2 Intermediate — Grouping and Aggregation

```java
// Group orders by status, count per status, sum per status
Map<OrderStatus, OrderSummary> summary = orders.stream()
    .collect(Collectors.groupingBy(
        Order::getStatus,
        Collectors.teeing(
            Collectors.counting(),
            Collectors.summingDouble(Order::getTotal),
            (count, sum) -> new OrderSummary(count, sum)
        )
    ));
```

### 4.3 Advanced — Paginated, Sorted, Filtered Query

```java
public PageResult<Product> search(ProductSearchRequest request) {
    Stream<Product> stream = productRepository.findAll().stream();

    // Apply dynamic filters
    if (request.getCategory() != null) {
        stream = stream.filter(p -> p.getCategory().equals(request.getCategory()));
    }
    if (request.getMinPrice() != null) {
        stream = stream.filter(p -> p.getPrice() >= request.getMinPrice());
    }
    if (request.getMaxPrice() != null) {
        stream = stream.filter(p -> p.getPrice() <= request.getMaxPrice());
    }
    if (request.getSearchTerm() != null) {
        stream = stream.filter(p -> p.getName().toLowerCase()
            .contains(request.getSearchTerm().toLowerCase()));
    }

    // Count before pagination
    List<Product> all = stream.toList();
    long total = all.size();

    // Apply sort
    Comparator<Product> comparator = switch (request.getSortBy()) {
        case "price" -> Comparator.comparing(Product::getPrice);
        case "name" -> Comparator.comparing(Product::getName);
        default -> Comparator.comparing(Product::getId);
    };
    if (request.isDescending()) {
        comparator = comparator.reversed();
    }

    // Paginate
    List<Product> page = all.stream()
        .sorted(comparator)
        .skip((long) request.getPage() * request.getPageSize())
        .limit(request.getPageSize())
        .toList();

    return new PageResult<>(page, request.getPage(), request.getPageSize(), total);
}
```

### 4.4 Production Bad vs Good

```java
// BAD: Side-effects in forEach
List<String> names = new ArrayList<>();
users.stream()
    .filter(User::isActive)
    .forEach(u -> names.add(u.getName())); // Side-effect — not thread-safe for parallel

// GOOD: Collect
List<String> names = users.stream()
    .filter(User::isActive)
    .map(User::getName)
    .collect(Collectors.toList());

// BAD: Multiple streams from same source (inefficient)
long count = orders.stream().filter(o -> o.getTotal() > 100).count();
double sum = orders.stream().filter(o -> o.getTotal() > 100)
    .mapToDouble(Order::getTotal).sum();

// GOOD: Single stream, collect once
Map<Boolean, List<Order>> partitioned = orders.stream()
    .filter(o -> o.getTotal() > 100)
    .collect(Collectors.partitioningBy(o -> o.getTotal() > 100));
double sum = partitioned.get(true).stream()
    .mapToDouble(Order::getTotal).sum();
long count = partitioned.get(true).size();
```

### 4.5 Parallel Stream — Batch Processing

```java
// Parallel processing with custom thread pool
public void processBatch(List<Transaction> transactions) {
    ForkJoinPool customPool = new ForkJoinPool(8); // 8 threads
    try {
        customPool.submit(() ->
            transactions.parallelStream().forEach(this::processTransaction)
        ).get();
    } catch (Exception e) {
        log.error("Batch processing failed", e);
        throw new BatchProcessingException("Failed to process batch", e);
    } finally {
        customPool.shutdown();
    }
}

private void processTransaction(Transaction tx) {
    try {
        // CPU-intensive work
        tx.validate();
        tx.applyRules();
        tx.calculateFees();
        transactionRepository.save(tx);
    } catch (Exception e) {
        log.error("Failed to process transaction: {}", tx.getId(), e);
    }
}
```

### 4.6 Real Stream — File Processing

```java
// Filter and transform large file without loading everything
public Map<String, Long> analyzeLogFile(Path logPath) throws IOException {
    try (Stream<String> lines = Files.lines(logPath)) {
        return lines
            .filter(line -> line.contains("ERROR"))
            .map(this::extractServiceName)
            .collect(Collectors.groupingBy(
                Function.identity(),
                Collectors.counting()
            ));
    } // Auto-closed by try-with-resources
}
```

### 4.7 Custom Collector

```java
// Collect into immutable list (Java 16+ already has toList())
public static <T> Collector<T, ?, List<T>> toImmutableList() {
    return Collector.of(
        ArrayList::new,           // supplier
        List::add,                // accumulator
        (left, right) -> {        // combiner
            left.addAll(right);
            return left;
        },
        Collections::unmodifiableList, // finisher
        Collector.Characteristics.CONCURRENT // if input is concurrent
    );
}

// Usage
List<Order> immutableOrders = orders.stream()
    .filter(o -> o.isActive())
    .collect(toImmutableList());
```

---

## 5–16. Due to space constraints, remaining sections are summarized with key points:

### Real-World Scenarios
1. **N+1 with streams** — Don't make DB calls in `.map()` (one query per element). Use JOIN or batch.
2. **Parallel stream causing thread starvation** — Parallel on shared ForkJoinPool blocks other tasks. Use custom pool.
3. **Lazy evaluation hiding exceptions** — Exception thrown during `.map()` isn't caught by try-catch around terminal op.
4. **Memory from sorted() on large stream** — `sorted()` buffers all elements. Use database ORDER BY instead.
5. **Stream reuse error** — Can't use stream after terminal op. Use Supplier pattern.
6. **Null elements in stream** — Stream.of(null) returns [null]. Stream.ofNullable(null) returns [].
7. **Checked exceptions in lambdas** — Stream doesn't handle checked exceptions. Wrap in RuntimeException.
8. **Stateful lambda in parallel** — `map()` with mutable shared state causes race conditions.
9. **Limit + skips = wrong results in parallel** — `limit()` + `skip()` + parallel = non-deterministic order.
10. **Collectors.toMap with duplicate keys** — Throws IllegalStateException. Provide merge function.

### Performance
- Stream vs loop: ~10-20% overhead for simple operations, valuable for complex pipelines
- Parallel stream: only beneficial for CPU-bound, large datasets (10K+ elements), non-trivial per-element work
- Primitive streams: use `IntStream` not `Stream<Integer>` to avoid boxing
- Avoid: `sorted()` on large datasets, multiple stream creations from same source, `.parallel()` on I/O-bound ops

### Common Mistakes (10 Key)
1. **Modifying source during stream operation** → ConcurrentModificationException
2. **Forgetting try-with-resources for file streams** → resource leak
3. **Stateful lambda in parallel stream** → data race
4. **Not using primitive streams** → autoboxing overhead
5. **Null returns from collect/toList** → careful: toList() returns null for empty only if you collect to a collection that returns null
6. **Assuming stream order in parallel** → use forEachOrdered
7. **Suppliers with side effects** → `Stream.generate(() -> counter++)` — broken in parallel
8. **Infinite stream without limit** → never terminates
9. **peek() for production logic** → peek is for debugging, not guaranteed to execute
10. **flatMap with null inner stream** → NPE. Wrap: `flatMap(x -> x.getList() != null ? x.getList().stream() : Stream.empty())`

### Comparison: Stream vs Loop vs SQL

| Aspect | Stream | Loop | SQL |
|--------|--------|------|-----|
| Readability | High (for pipelines) | Low (verbose) | High |
| Performance | Moderate | High | Best (DB optimizations) |
| Memory | In-process | In-process | Server-side |
| Parallelism | Built-in | Manual | Built-in |
| Debugging | Hard (lambda stack traces) | Easy | Moderate |
| Lazy | Yes | No | Yes |

### Cheat Sheet

```
═══ STREAM API ═══════════════════════════════════════════════

┌─ PIPELINE ─────────────────────────────────────────────────┐
│ source .filter() .map() .sorted() .collect()                │
│ ↑                   ↑                       ↑               │
│ source              intermediate             terminal       │
└─────────────────────────────────────────────────────────────┘

┌─ CREATION ─────────────────────────────────────────────────┐
│ coll.stream()       Arrays.stream(arr)     Stream.of(a,b)  │
│ IntStream.range()   Files.lines(path)      Stream.iterate()│
└─────────────────────────────────────────────────────────────┘

┌─ INTERMEDIATE ─────────────────────────────────────────────┐
│ filter(Predicate)    map(Function)        flatMap(F)       │
│ distinct()           sorted()             peek(Consumer)   │
│ limit(n)             skip(n)              takeWhile(Pred)  │
└─────────────────────────────────────────────────────────────┘

┌─ TERMINAL ─────────────────────────────────────────────────┐
│ toList()   collect(Collector)  forEach(Consumer)           │
│ reduce(op) count()             anyMatch/allMatch/noneMatch │
│ findFirst() findAny()          min()/max()                 │
│ forEachOrdered()  toArray()                                │
└─────────────────────────────────────────────────────────────┘

┌─ COMMON COLLECTORS ────────────────────────────────────────┐
│ toList()      toSet()     toCollection(ArrayList::new)     │
│ toMap(k,v)    groupingBy(f)      partitioningBy(pred)      │
│ joining()     counting()         summingInt(f)             │
│ averagingInt() summarizingInt()  mapping(f, downstream)    │
│ filtering(p, downstream)  reducing()                      │
└─────────────────────────────────────────────────────────────┘

┌─ RULES ────────────────────────────────────────────────────┐
│ • Use primitive streams (IntStream) to avoid boxing        │
│ • Prefer collect/forEach over side-effects in lambdas      │
│ • Parallel: only on CPU-bound, 10K+ elements               │
│ • File streams need try-with-resources                      │
│ • Streams are one-use — Supplier pattern for reuse         │
│ • sorted() buffers all — use DB sort when possible         │
└─────────────────────────────────────────────────────────────┘
```
