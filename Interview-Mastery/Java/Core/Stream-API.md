# Java Stream API

---

## Overview

- **Definition:** The Stream API (Java 8+) provides a functional, declarative way to process sequences of data using a pipeline of operations. A stream represents a sequence of elements and supports aggregate operations.

- **Why It Exists:** Before streams, data processing meant imperative loops — verbose, error-prone, and hard to parallelize. Streams provide:
  - **Declarative code** — you specify what to do, not how to do it
  - **Composability** — chain operations together
  - **Lazy evaluation** — compute only what's needed
  - **Easy parallelism** — single `.parallel()` call

- **Key Characteristics:**
  - Not a data structure — doesn't store data
  - Lazy — intermediate operations run only when a terminal operation is invoked
  - Consumable — can be used only once (throws `IllegalStateException` on reuse)
  - Parallelizable — `.parallel()` enables multi-threaded processing

- **When to Use:** Processing collections, chaining multiple operations, parallel processing, when code readability matters more than raw performance.

- **When NOT to Use:** Simple loops (3 lines or fewer), performance-critical hot paths, primitive arrays in hot paths, checked exceptions in lambdas, very large datasets that don't fit in memory.

---

## Stream Pipeline Structure

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

---

## Creating Streams

```java
// From collections
List<String> list = List.of("a", "b", "c");
Stream<String> stream = list.stream();
Stream<String> parallelStream = list.parallelStream();

// From arrays
String[] array = {"a", "b", "c"};
Stream<String> arrayStream = Arrays.stream(array);

// From values
Stream<String> of = Stream.of("a", "b", "c");

// From functions (infinite streams — use with limit())
Stream<Integer> iterate = Stream.iterate(0, n -> n + 1);

// From builder
Stream<String> built = Stream.<String>builder()
    .add("a").add("b").add("c").build();

// From file
Stream<String> lines = Files.lines(Paths.get("file.txt"));
```

---

## Intermediate Operations

- **Definition:** Operations that return a new Stream. They are lazy — not executed until a terminal operation is called.

- **filter(Predicate):** Keep elements matching the predicate
- **map(Function):** Transform each element
- **flatMap(Function):** Flatten nested streams into a single stream
- **distinct():** Remove duplicates (uses equals)
- **sorted():** Sort elements (natural order or with Comparator)
- **peek(Consumer):** Debug — view each element (use sparingly)
- **limit(long):** Truncate to max size
- **skip(long):** Discard first N elements
- **takeWhile(Predicate):** Take elements while predicate is true (Java 9+)
- **dropWhile(Predicate):** Drop elements while predicate is true (Java 9+)

```java
List<String> result = list.stream()
    .filter(s -> s.length() > 3)
    .map(String::toUpperCase)
    .sorted()
    .collect(Collectors.toList());
```

---

## Terminal Operations

- **Definition:** Operations that produce a result or side effect. They trigger the entire pipeline execution.

- **forEach(Consumer):** Perform action for each element (side-effect)
- **toList():** Collect to immutable list (Java 16+)
- **collect(Collector):** Collect using a Collector
- **reduce(BinaryOperator):** Combine elements using associative reduction
- **count():** Return element count
- **anyMatch(Predicate):** Return true if any element matches
- **allMatch(Predicate):** Return true if all elements match
- **noneMatch(Predicate):** Return true if no elements match
- **findFirst():** Return first element (respects encounter order)
- **findAny():** Return any element (parallel-friendly)
- **min(Comparator):** Return minimum element
- **max(Comparator):** Return maximum element

```java
// Collect to list
List<String> collected = stream.collect(Collectors.toList());

// Group by key
Map<String, List<Order>> byStatus = orders.stream()
    .collect(Collectors.groupingBy(Order::getStatus));

// Check if any match
boolean hasAdults = users.stream()
    .anyMatch(u -> u.getAge() >= 18);

// Sum values
int total = numbers.stream()
    .reduce(0, Integer::sum);
```

---

## Collectors

- **Definition:** Utility class providing implementations of `Collector` for common reduction operations.

```java
// To collections
.collect(Collectors.toList())
.collect(Collectors.toSet())
.collect(Collectors.toCollection(ArrayList::new))

// To map
.collect(Collectors.toMap(Function<K>, Function<V>))

// Grouping
.collect(Collectors.groupingBy(Function))              // Map<K, List<V>>
.collect(Collectors.groupingBy(Function, Collector))   // with downstream

// Partitioning
.collect(Collectors.partitioningBy(Predicate))         // Map<Boolean, List<V>>

// Joining
.collect(Collectors.joining(", "))                     // "a, b, c"

// Summarizing
.collect(Collectors.summarizingInt(ToIntFunction))
.collect(Collectors.averagingInt(ToIntFunction))
.collect(Collectors.counting())
```

---

## Primitive Streams

- **Definition:** Specialized streams for primitives (`IntStream`, `LongStream`, `DoubleStream`) to avoid autoboxing overhead.

```java
IntStream.range(1, 10)                     // [1, 2, ..., 9]
IntStream.rangeClosed(1, 10)               // [1, 2, ..., 10]

// Conversion
IntStream intStream = list.stream().mapToInt(Integer::intValue);
Stream<Integer> boxed = intStream.boxed();
```

---

## Lazy Evaluation and Fusion

- **Definition:** Streams are lazy — intermediate operations don't execute until a terminal operation is called. The JVM can fuse adjacent operations into a single pass.

```java
// Nothing happens here:
Stream<String> stream = list.stream()
    .filter(s -> s.length() > 3)
    .map(String::toUpperCase);

// Only when terminal op is called:
List<String> result = stream.toList();
```

- **Fusion:** Elements go through the entire pipeline one at a time. `filter().map()` processes each element through both operations, not all elements through filter then all through map.

---

## Performance Characteristics

| Operation | Type | Time | Memory |
|-----------|------|------|--------|
| filter() | Stateless | O(n) | O(1) |
| map() | Stateless | O(n) | O(1) |
| distinct() | Stateful | O(n) | O(n) |
| sorted() | Stateful | O(n log n) | O(n) |
| limit() | Stateful | O(n) | O(limit) |
| reduce() | Terminal | O(n) | O(1) |

- Stateful operations like `sorted()` and `distinct()` must buffer all elements — use cautiously with large datasets.
- Parallel streams benefit CPU-bound operations on 10K+ elements. Overhead outweighs benefits for small datasets.

---

## Common Mistakes

- **Modifying source during stream operation** — causes `ConcurrentModificationException`
- **Forgetting try-with-resources for file streams** — resource leak. Always use `try (Stream<String> lines = Files.lines(path))`
- **Stateful lambda in parallel stream** — shared mutable state causes race conditions
- **Not using primitive streams** — autoboxing overhead with `Stream<Integer>` instead of `IntStream`
- **Assuming stream order in parallel** — use `forEachOrdered()` if order matters
- **Infinite stream without limit** — never terminates
- **peek() for production logic** — peek is for debugging, not guaranteed to execute
- **Collectors.toMap with duplicate keys** — throws IllegalStateException. Provide merge function: `(v1, v2) -> v1`

---

## Real-World Scenarios

### Scenario 1: Real-Time Fraud Detection Pipeline

A payment processing system streams 10K transactions/second. Each transaction must be checked against 20 fraud rules (velocity checks, geographic anomalies, amount thresholds). Rules must be composable and the pipeline must handle late-arriving data.

```java
public class FraudDetectionPipeline {
    public FraudResult evaluate(Transaction tx) {
        return Stream.<Predicate<Transaction>>of(
            this::exceedsVelocityLimit,
            this::suspiciousGeography,
            this::unusualAmount,
            this::blacklistedMerchant
        )
        .filter(rule -> rule.test(tx))
        .findFirst()
        .map(rule -> FraudResult.FLAGGED)
        .orElse(FraudResult.CLEAR);
    }
}
```

The stream of predicates evaluates lazily — it stops at the first matching rule (short-circuit via `findFirst()`), avoiding unnecessary computation. Rules are individually testable and composable. For the full 20-rule evaluation, switching `filter(...).collect(toList())` gives all triggered rules for audit trails.

### Scenario 2: Data Warehouse ETL with Column Transformations

A nightly batch job reads 50M customer records, normalizes phone numbers, validates emails, geocodes addresses via an external API, and writes to a data warehouse. The job must be parallelizable and handle partial failures.

```java
public class CustomerETL {
    public List<CustomerRecord> process(List<RawRecord> records) {
        return records.parallelStream()
            .map(this::normalizePhone)
            .map(this::validateEmail)
            .flatMap(this::geocodeAddress) // may produce 0 or 1
            .filter(Objects::nonNull)
            .collect(Collectors.toList());
    }
}
```

`parallelStream()` distributes CPU-bound normalization across cores. `flatMap` handles the geocode step which may fail for some records (returning `Stream.empty()`). The pipeline is declarative — each transformation is independently testable. For checkpointing, wrap in a custom `Spliterator` that saves progress to a database.

### Scenario 3: Real-Time Dashboard Aggregation

A monitoring system collects server metrics (CPU, memory, disk) from 1000 servers every 10 seconds. It computes per-service averages, percentiles (p50, p95, p99), and detects anomalies — all within a 5-second processing window.

```java
public class MetricsAggregator {
    public Map<String, DoubleSummaryStatistics> aggregate(List<Metric> metrics) {
        return metrics.stream()
            .collect(Collectors.groupingBy(
                Metric::getService,
                Collectors.summarizingDouble(Metric::getValue)
            ));
    }

    public Map<String, List<Metric>> detectAnomalies(List<Metric> metrics) {
        Map<String, Double> averages = metrics.stream()
            .collect(Collectors.groupingBy(
                Metric::getService,
                Collectors.averagingDouble(Metric::getValue)
            ));

        return metrics.stream()
            .filter(m -> Math.abs(m.getValue() - averages.get(m.getService())) > 3 * stddev)
            .collect(Collectors.groupingBy(Metric::getService));
    }
}
```

Two stream pipelines over the same data: the first computes averages, the second filters anomalies. Stream fusion ensures each pipeline processes elements one at a time, keeping memory O(1) per pipeline. For 1000 servers × 4 metrics × 6 readings = 24K elements, the entire aggregation completes in under 100ms.

---

## Scenario-Based Questions

1. **Q: You are building a search autocomplete feature. Users type a query, and you must return the top 10 suggestions from a dictionary of 500K phrases. The suggestions must match by prefix and be ordered by popularity. Users type every keystroke — response must be under 50ms. How do you use streams?**
   A: Don't use streams for the hot path — streams add indirection that misses the 50ms budget. Instead, use a `Trie` data structure for prefix lookup and a priority queue for ranking. Streams can be used for the offline index build: `dictionary.stream().sorted(byPopularity).collect(toList())` to pre-sort by popularity. For the online path, use raw loops. The lesson: streams prioritize readability over performance — don't use them in latency-critical hot paths.

2. **Q: You have a microservice that receives a list of order IDs from the API gateway. You need to fetch each order from a downstream service (HTTP call), enrich it with customer data from another service, and return the combined result. Each fetch takes 50-200ms. How do you run these calls concurrently with streams?**
   A: Use `CompletableFuture` with a stream pipeline for the fork-join pattern, but NOT `parallelStream()` (which uses the shared ForkJoinPool and can starve other tasks):
   ```java
   List<CompletableFuture<Order>> futures = orderIds.stream()
       .map(id -> CompletableFuture.supplyAsync(() -> fetchOrder(id), executor))
       .toList();
   List<Order> orders = futures.stream()
       .map(CompletableFuture::join)
       .toList();
   ```
   The first stream creates all futures (non-blocking). The second stream joins them. Using a dedicated `executor` with bounded thread pool prevents resource exhaustion. This pattern gives N-way concurrency limited only by the thread pool size.

3. **Q: You have a list of 10M transactions and need to compute: total revenue, average transaction value, number of fraudulent transactions, and revenue by merchant category. How do you avoid iterating 4 times?**
   A: Use a custom collector that accumulates all four statistics in a single pass:
   ```java
   record Stats(BigDecimal totalRevenue, double avgValue, long fraudCount, Map<String, BigDecimal> byCategory) {}

   Stats stats = transactions.stream().collect(Collector.of(
       () -> new BigDecimal[] { BigDecimal.ZERO, BigDecimal.ZERO, 0L },
       (acc, t) -> {
           acc[0] = acc[0].add(t.getAmount());
           acc[1] = acc[1].add(t.getAmount());
           if (t.isFraud()) acc[2]++;
       },
       (a, b) -> { a[0] = a[0].add(b[0]); a[1] = a[1].add(b[1]); a[2] += b[2]; return a; },
       acc -> new Stats(acc[0], acc[1].doubleValue() / count, acc[2], ...)
   ));
   ```
   For the category grouping in the same pass, use `Collectors.teeing()` (Java 12+) or a holder object. A single pass over 10M elements is ~10ms — four passes would be ~40ms and waste CPU cache.

4. **Q: A stream pipeline processes a file with `Files.lines()`. Halfway through, a line has malformed data and throws a runtime exception. The file handle is never closed. How do you ensure robust resource cleanup?**
   A: Always use try-with-resources with `Files.lines()`:
   ```java
   try (Stream<String> lines = Files.lines(path)) {
       lines.map(this::parseLine)
            .forEach(this::process);
   } catch (IOException e) {
       log.error("Failed to process file", e);
   }
   ```
   The try-with-resources calls `close()` on the stream even if an exception is thrown inside the pipeline. Without it, if a lambda throws `RuntimeException` (e.g., NPE), the underlying `FileChannel` is leaked. In production, also add `onClose()` handlers and wrap parsing in `map(m -> { try { return parse(m); } catch (Exception e) { log.warn("Skipping bad line", e); return null; } }).filter(Objects::nonNull)`.

5. **Q: You need to paginate through a large dataset (1M records) returned from a database cursor in batches of 100. The cursor is stateful and not thread-safe. How do you process all records using streams without loading them all into memory?**
   A: Create a custom `Spliterator` that wraps the database cursor:
   ```java
   public class CursorSpliterator<T> extends Spliterators.AbstractSpliterator<T> {
       private final Cursor<T> cursor;
       CursorSpliterator(Cursor<T> cursor) {
           this(Long.MAX_VALUE, Spliterator.ORDERED | Spliterator.NONNULL);
           this.cursor = cursor;
       }
       @Override public boolean tryAdvance(Consumer<? super T> action) {
           if (!cursor.hasNext()) return false;
           action.accept(cursor.next());
           return true;
       }
   }
   // Use: StreamSupport.stream(new CursorSpliterator<>(cursor), false)
   ```
   The custom spliterator reads one record at a time from the cursor. The stream pipeline chains operations (filter, map, collect) without ever holding all records in memory. For batch processing, add a `trySplit()` implementation that partitions the cursor by range.

6. **Q: A system streams sensor readings at 100K events/second. You need to compute the moving average over a 5-second sliding window. The stream is infinite. How do you maintain only the relevant data?**
   A: Use `Stream.iterate()` with a bounded buffer or a ring buffer backed by an array:
   ```java
   public class SlidingWindowAverage {
       private final double[] buffer = new double[500_000]; // 5s * 100K/s
       private int index = 0;
       private double sum = 0;
       private boolean filled = false;

       public double next(double value) {
           if (filled) sum -= buffer[index];
           buffer[index] = value;
           sum += value;
           index = (index + 1) % buffer.length;
           if (index == 0) filled = true;
           int count = filled ? buffer.length : index;
           return sum / count;
       }
   }
   ```
   This is O(1) per event with no allocations after initialization. A stream-based approach using `collect()` with a custom collector avoids GC pressure. The ring buffer pattern is essential for real-time stream processing where GC pauses cause data loss.

7. **Q: You are building a data validation service. Each record must pass 15 validation rules (some are cheap string checks, others are expensive DB lookups). You want to fail fast — stop at the first violation — but also want to collect ALL violations for the audit log. How do you design this with streams?**
   A: Two strategies: (1) Fail-fast path: `rules.stream().filter(r -> !r.test(record)).findFirst()` — stops at first failure. (2) Full audit path: `rules.stream().map(r -> r.validate(record)).filter(Objects::nonNull).collect(toList())` — collects all violations. Use the fail-fast path in the request thread (return error to client immediately) and submit the full validation to a background executor for the audit trail. The key: streams support both short-circuit and full-evaluation modes depending on the terminal operation.

8. **Q: You have a stream of events, each with a `LocalDateTime timestamp`. Events can arrive out of order (up to 30 seconds late). You need to group events into 1-minute windows and process each window in chronological order. How do you handle the watermark and late data?**
   A: This requires a custom `Spliterator` or a stateful collector with a watermark:
   ```java
   public class WindowedProcessor {
       private final long watermarkMs = 30_000;
       private final TreeMap<Long, List<Event>> windows = new TreeMap<>();

       public void process(Event event) {
           long windowKey = event.getTimestamp().truncatedTo(ChronoUnit.MINUTES)
               .toInstant(ZoneOffset.UTC).toEpochMilli();
           windows.computeIfAbsent(windowKey, k -> new ArrayList<>()).add(event);

           long now = System.currentTimeMillis();
           windows.headMap(now - watermarkMs - 60_000, false)
               .forEach((key, events) -> processWindow(key, events));
           windows.headMap(now - watermarkMs - 60_000).clear();
       }
   }
   ```
   This is exactly how Apache Flink's `TumblingEventTimeWindows` works. Streams alone are insufficient — you need stateful windowing with watermarks. The `TreeMap` provides sorted windows, and `headMap()` efficiently evicts completed windows.

9. **Q: You need to join two streams: a stream of Orders and a stream of Payments. Each order has multiple payments. You must enrich each order with its total paid amount. Both streams are large (millions). How do you join them with streams?**
   A: Use a two-pass approach: collect payments into a `Map<OrderId, BigDecimal>` using a stream, then join with orders using another stream:
   ```java
   Map<String, BigDecimal> paymentTotals = payments.stream()
       .collect(Collectors.groupingBy(
           Payment::getOrderId,
           Collectors.mapping(Payment::getAmount,
               Collectors.reducing(BigDecimal.ZERO, BigDecimal::add))
       ));

   List<EnrichedOrder> enriched = orders.stream()
       .map(order -> new EnrichedOrder(order,
           paymentTotals.getOrDefault(order.getId(), BigDecimal.ZERO)))
       .collect(Collectors.toList());
   ```
   The first pass builds a hash map (O(n) memory), the second pass enriches orders with O(1) lookups. For streams too large for memory, use external sort-merge join or a database. Streams don't support streaming joins natively — this is where SQL or Flink/Spark is appropriate.

10. **Q: A stream of transactions produces a side effect (logging) in `peek()` for debugging. The pipeline is later parallelized, and the log output is jumbled across threads. How do you safely add logging to a stream pipeline?**
    A: `peek()` is not guaranteed to execute in order when parallelized, and it may execute more than once if the stream implementation retries. For safe logging:
    - With parallel streams, use `forEachOrdered()` instead of `forEach()` if order matters:
    ```java
    transactions.parallelStream()
        .filter(t -> t.getAmount() > 1000)
        .forEachOrdered(t -> log.info("Large tx: {}", t.getId()));
    ```
    - Use a dedicated logging collector: `collect(Collectors.collectingAndThen(toList(), list -> { list.forEach(t -> log.info(...)); return list; }))`.
    - Better: separate logging from the stream pipeline. Log in a terminal operation, not in `peek()`.

---

## Interview Questions

1. **What is the difference between intermediate and terminal operations?**
   A: Intermediate operations (e.g., `filter()`, `map()`) return a new `Stream` and are lazy — they don't execute until a terminal operation is called. Terminal operations (e.g., `collect()`, `forEach()`) trigger the pipeline execution and produce a result or side effect. A stream can have multiple intermediate operations but only one terminal operation. After a terminal operation, the stream is consumed and cannot be reused.

2. **What is the difference between `map()` and `flatMap()`?**
   A: `map()` transforms each element 1:1 — one input produces one output. `flatMap()` transforms each element 1:N — each input can produce zero, one, or many outputs, which are flattened into a single stream. `flatMap()` is used to flatten nested collections, handle null/optional results (return `Stream.empty()` instead of null), and expand one record into multiple.

3. **What is stream fusion and how does it improve performance?**
   A: Stream fusion is an optimization where multiple adjacent operations are fused into a single pass over the data. Instead of creating an intermediate collection for `filter()` then iterating again for `map()`, the JVM processes each element through the entire pipeline before moving to the next. This improves cache locality and reduces the overhead of creating intermediate streams. For example, `stream.filter(pred).map(fn).collect(toList())` processes each element through both filter and map in one iteration.

4. **What is the difference between `findFirst()` and `findAny()`?**
   A: `findFirst()` returns the first element from the stream respecting encounter order. `findAny()` returns any element and is non-deterministic. In sequential streams, both typically return the same result. In parallel streams, `findAny()` is significantly faster because it doesn't need to preserve order — any thread can return its match immediately. Use `findAny()` when you don't care which matching element is returned.

5. **How does `Collectors.groupingBy()` work internally?**
   A: `groupingBy(Function)` returns a `Collector` that classifies elements into a `Map<K, List<V>>`. Internally, it uses a `Map` accumulator — for each element, it applies the classifier function to get the key and adds the element to the corresponding list. The default `groupingBy()` uses `HashMap` and `ArrayList`. You can customize both the map type (`groupingBy(Function, Supplier<Map>, Collector)`) and the value collection (`groupingBy(Function, downstreamCollector)`).

6. **What is the difference between `Collection.stream()` and `Stream.of(collection)`?**
   A: `collection.stream()` returns a stream from the collection's elements. `Stream.of(collection)` treats the collection itself as a single element — it returns `Stream<List<T>>`, not `Stream<T>`. To stream elements from an array: `Stream.of(array)` works correctly because the varargs method treats the array as elements. This is a common source of bugs.

7. **When should you use `reduce()` vs `collect()`?**
   A: `reduce()` is for immutable reduction — combining values using an associative operation (e.g., sum, max). `collect()` is for mutable reduction — accumulating into a mutable container (e.g., `List`, `StringBuilder`). Use `collect()` when building collections or string concatenation; use `reduce()` for arithmetic operations. `reduce()` with mutable objects requires the accumulator to be associative, which is error-prone for non-associative operations like addition to a list.

8. **What is the purpose of `IntStream`, `LongStream`, and `DoubleStream`?**
   A: These are primitive stream specializations that avoid autoboxing overhead. `Stream<Integer>` boxes each value to `Integer` (28 bytes per value). `IntStream` stores `int` values directly (4 bytes). For numeric operations on large datasets, primitive streams can be 5-10x faster and use significantly less memory. They also provide specialized operations like `sum()`, `average()`, `range()`, and `summaryStatistics()`.

9. **Why are streams consumable only once?**
   A: Streams are designed as one-shot data pipelines. Most streams are backed by a data source that can be traversed only once (e.g., network stream, file lines, iterator). Even collection-backed streams are one-shot because intermediate results aren't cached — reusing would require re-executing the entire pipeline. Attempting to reuse a stream throws `IllegalStateException: stream has already been operated upon or closed`.

10. **What is the difference between sequential and parallel streams in terms of thread safety?**
    A: Sequential streams execute on the calling thread. Parallel streams use the common `ForkJoinPool` to split the source into segments processed by multiple threads. Parallel streams require that the stream operations be stateless and non-interfering. Shared mutable state in lambdas causes race conditions. Operations like `findAny()`, `limit()`, and `unordered().skip()` are more efficient in parallel. Always measure — parallel streams add overhead for small datasets or trivial operations.

---

## Developer Recommendations

- **Prefer primitive streams over boxed streams for numeric data** — `IntStream.range(0, 1_000_000).sum()` avoids boxing 1M integers. `Stream<Integer>` creates 28 bytes of garbage per element. For hot paths, use `IntStream`, `LongStream`, or `DoubleStream`. The primitive `sum()`, `average()`, and `summaryStatistics()` are also more readable than manual reduction.

- **Use `toList()` (Java 16+) over `collect(Collectors.toList())`** — `stream.toList()` returns an immutable list, guarantees null-free elements, and is shorter. `collect(Collectors.toList())` returns a mutable `ArrayList`. Using immutable lists prevents accidental modification and communicates intent. If you need a mutable list, use `collect(Collectors.toCollection(ArrayList::new))`.

- **Avoid `parallelStream()` for I/O-bound operations** — Parallel streams use the common ForkJoinPool which is shared system-wide. I/O operations block threads, starving other parts of the system. For I/O, use `CompletableFuture` with a dedicated executor sized to `cores * (1 + waitTime / computeTime)`. Use `parallelStream()` only for CPU-intensive operations on datasets >10K elements where the per-element work justifies threading overhead.

- **Use method references over lambdas when possible** — `orders.stream().map(Order::getTotal)` is more readable and has less allocation overhead than `orders.stream().map(o -> o.getTotal())`. Non-capturing lambdas are cached, but method references are even more explicit. For complex logic, extract to a named method: `orders.stream().map(this::processOrder)`.

- **Avoid stateful lambdas in parallel streams** — `map()` lambdas should be stateless (no shared mutable state). Parallel streams partition data across threads, and stateful lambdas cause race conditions. If you need ordering, use `forEachOrdered()` — but this serializes the terminal operation. For thread-safe accumulation, use `collect()` with `ConcurrentHashMap` or thread-safe collectors.

- **Use `try-with-resources` for `Files.lines()` and `Stream<Path>` from `Files.walk()`** — These streams hold underlying file handles. Without try-with-resources, an exception in the pipeline leaks the file handle. Always wrap: `try (Stream<String> lines = Files.lines(path)) { lines.forEach(...); }`. This is a common cause of file handle leaks in production.

- **Use `Collectors.teeing()` (Java 12+) for dual aggregation in one pass** — When you need two different aggregations from the same stream (e.g., count and sum), `teeing()` lets you do both in one pass: `stream.collect(Collectors.teeing(Collectors.counting(), Collectors.summingInt(v -> v), (count, sum) -> new Stats(count, sum)))`. This avoids iterating twice and halves the processing time. Before Java 12, use a custom collector with a holder object.
