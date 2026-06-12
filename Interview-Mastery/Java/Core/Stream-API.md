# Java Stream API

---

## Overview

- **Purpose** — The Stream API, introduced in Java 8, provides a functional and declarative approach to processing sequences of data through a pipeline of operations without requiring explicit iteration. A stream is not a data structure; it does not store elements but instead conveys them from a source through a chain of operations.
- **Lazy Evaluation** — Streams are lazy by nature — intermediate operations execute only when a terminal operation triggers them. They are consumable exactly once, throwing `IllegalStateException` on any attempt to reuse a consumed pipeline.
- **When to Use** — Use streams when processing collections with multiple chained operations, when readability matters more than raw iteration speed, and when you want effortless parallelism. Avoid streams for simple loops of three lines or fewer, performance-critical hot paths where allocation overhead matters, checked-exception-throwing lambdas, and very large datasets that do not fit in memory.
- **Before Streams** — Data processing meant writing imperative loops with mutable accumulators, boilerplate variables, and error-prone manual parallelism. Streams let you specify what computation to perform declaratively, while the runtime handles iteration, short-circuiting, laziness, and parallelism.

  **Why internal iteration?** External iteration (the `for` loop) forces the caller to manage the loop variable, collection state, and termination condition — the caller owns the iteration and the caller must also own any optimizations. Internal iteration (streams) transfers control to the library, which can then fuse operations into a single pass, skip operations for short-circuiting, parallelize by splitting the spliterator, and eliminate intermediate collections. This inversion of control is the fundamental architectural shift that makes laziness, fusion, and parallelism possible without burdening the caller.

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

- **Pipeline Stages** — A stream pipeline consists of three stages: a source that produces elements, zero or more intermediate operations that transform the stream, and a single terminal operation that produces a result or side effect.
- **Sources** — The source can be a collection via `.stream()`, an array via `Arrays.stream()`, static factory methods like `Stream.of()`, an I/O resource like `Files.lines()`, or an infinite generator via `Stream.iterate()` or `Stream.generate()`.
- **Operation Types** — Intermediate operations are divided into stateless operations (like `filter()` and `map()`) that process each element independently, and stateful operations (like `sorted()` and `distinct()`) that must buffer elements and examine the entire stream before producing output.
- **Terminal Trigger** — Terminal operations trigger the entire computation — without one, no work is done and the pipeline is a mere specification of intent.

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

- **Collection Source** — The most common source is a `Collection` via `.stream()`, which returns a sequential stream backed by the collection's `Spliterator`.
- **Array Conversion** — Arrays are converted through `Arrays.stream()`, which handles both object arrays and primitive arrays (returning `IntStream`, `LongStream`, or `DoubleStream`).
- **Convenience Methods** — `Stream.of()` varargs is convenient for small fixed sets, and `Stream.builder()` is useful for programmatic construction with a fluent API.
- **Infinite Streams** — `Stream.iterate()` and `Stream.generate()` produce infinite streams that must be paired with `limit()` to avoid unbounded execution.
- **File I/O** — `Files.lines()` returns a stream of file lines backed by an underlying `FileChannel` that must be closed properly with try-with-resources.

---

## Intermediate Operations

- **Lazy Nature** — Intermediate operations return a new `Stream` and are always lazy — they do not process any elements until a terminal operation is invoked on the pipeline.
- **filter()** — Selects elements that match a given condition using a `Predicate`. It is a stateless operation with O(n) time and O(1) memory.
- **map()** — Transforms each element via a one-to-one mapping function. It is stateless with O(n) time and O(1) memory.
- **flatMap()** — Transforms each element into zero or more elements (the function returns a `Stream`) and flattens all resulting streams into a single output stream. It is useful for handling optional results and expanding records.
- **distinct()** — Removes duplicates using `equals()` and buffers seen elements in a `Set` internally, making it O(n) in memory and a stateful operation.
- **sorted()** — Sorts elements according to natural order or a provided `Comparator`. It is a stateful operation that must collect all elements before producing any output — O(n log n) time and O(n) space.
- **peek()** — Primarily a debugging aid for inspecting elements as they flow through the pipeline. It is often misused for side effects in production.
- **limit() and skip()** — Truncate or bypass a prefix of the stream. `takeWhile()` and `dropWhile()` (Java 9+) provide conditional prefix-based operations that work well on sorted streams.

```java
List<String> result = list.stream()
    .filter(s -> s.length() > 3)
    .map(String::toUpperCase)
    .sorted()
    .collect(Collectors.toList());
```

---

## Terminal Operations

- **Purpose** — Terminal operations are what make the stream pipeline actually execute — without one, the pipeline is just a chain of lazy specifications that produces nothing.
- **forEach()** — Performs an action on each element, primarily used for side effects like logging. It sacrifices the functional purity that streams encourage.
- **toList() (Java 16+)** — Collects elements into an immutable list with a guarantee of no null elements. It is preferred over `collect(Collectors.toList())` for immutability guarantees.
- **collect()** — The general-purpose mutable reduction that can produce lists, sets, maps, strings, or custom accumulators using a `Collector`.
- **reduce()** — Performs an associative reduction operation — sum, product, concatenation, min, max — that works with both sequential and parallel execution.
- **Short-Circuiting Operations** — `anyMatch()`, `allMatch()`, `noneMatch()`, `findFirst()`, and `findAny()` can stop processing early once the result is determined, making them efficient for early-exit conditions on large streams.
- **Convenience Operations** — `count()`, `min()`, and `max()` are convenience terminal operations for common aggregation patterns.

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

- **Basic Collectors** — The `Collectors` utility class provides factory methods like `toList()`, `toSet()`, and `toCollection(Supplier)` for building collections, and `toMap()` for constructing maps with key and value extractors.
- **groupingBy()** — Classifies elements into a `Map<K, List<V>>`. With a downstream collector, it can perform nested aggregations like `groupingBy(Order::getStatus, counting())` or `groupingBy(Order::getStatus, mapping(Order::getAmount, toList()))`.
- **partitioningBy()** — Splits into exactly two groups based on a boolean predicate, returning `Map<Boolean, List<V>>`.
- **joining()** — Concatenates string representations with a delimiter, useful for building comma-separated strings.
- **Statistical Collectors** — `summarizingInt()`, `averagingInt()`, and `counting()` provide statistical reductions for numeric data.
- **Advanced Collectors** — `collectingAndThen()` wraps a collector to apply a finishing transformation, and `teeing()` (Java 12+) sends stream elements to two collectors simultaneously and merges their results.

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

- **Purpose** — `IntStream`, `LongStream`, and `DoubleStream` are specialized stream types for primitive numeric values that completely avoid the boxing and unboxing overhead inherent in `Stream<Integer>`. Each boxed `Integer` consumes 16-28 bytes of heap, so processing a million integers as `Stream<Integer>` creates a million short-lived objects.
- **Numeric Operations** — Primitive streams provide numeric-specific operations like `sum()`, `average()`, `min()`, `max()`, and `summaryStatistics()` directly without needing custom collectors.
- **Range Methods** — `range()` and `rangeClosed()` factory methods generate contiguous numeric sequences, commonly used for index-based iteration and pagination.
- **Conversion** — `mapToInt()`, `mapToLong()`, `mapToDouble()` convert from object streams, and `.boxed()` converts primitive streams back to object streams when you need to collect into generic collections like `List<Integer>`.

```java
IntStream.range(1, 10)                     // [1, 2, ..., 9]
IntStream.rangeClosed(1, 10)               // [1, 2, ..., 10]

// Conversion
IntStream intStream = list.stream().mapToInt(Integer::intValue);
Stream<Integer> boxed = intStream.boxed();
```

---

## Lazy Evaluation and Fusion

- **Lazy Execution** — Streams are lazy — intermediate operations registered in a pipeline are not executed until a terminal operation is invoked. Calling `.filter(predicate).map(function)` merely builds a description of the computation without touching a single element.
- **Stream Fusion** — The JVM merges multiple adjacent intermediate operations into a single pass over the data. Instead of iterating once for `filter()`, collecting survivors, and iterating again for `map()`, the fused pipeline processes each element through both operations before moving to the next element.
- **Performance Benefit** — Each element moves through the entire pipeline one at a time — `filter().map()` processes element 1 through filter, and if it passes, immediately through map, then element 2, and so on. This results in one pass instead of two, dramatically improving CPU cache locality by keeping each element in L1 cache across all operations.

  **How fusion works internally** — Each intermediate operation returns a new `Stream` wrapping a `Sink` (a callback that receives elements). When the terminal operation starts, the sinks are composed into a chain — `filter()`'s sink checks the predicate and passes matching elements downstream, `map()`'s sink transforms and forwards, and `collect()`'s sink accumulates. The JIT then inlines this sink chain into a tight loop with no virtual dispatch overhead. This is why writing pipelines as many small operations (`filter().map().sorted()`) is not slower than a hand-written loop — the JIT fuses them into roughly the same machine code, while the hand-written loop cannot expose parallelism opportunities.

```java
// Nothing happens here:
Stream<String> stream = list.stream()
    .filter(s -> s.length() > 3)
    .map(String::toUpperCase);

// Only when terminal op is called:
List<String> result = stream.toList();
```

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

- **Stateless Operations** — Operations like `filter()` and `map()` process each element independently and require no additional memory beyond a single-element buffer.
- **Stateful Operations** — Operations like `sorted()` and `distinct()` must observe all or a prefix of elements before producing output — `sorted()` collects all elements into an array before sorting, consuming O(n) memory and potentially causing `OutOfMemoryError` on large datasets.
- **Parallel Streams** — Benefit CPU-bound operations on datasets of at least 10K elements where per-element work is significant. For smaller datasets, the overhead of splitting, distributing, and merging outweighs any parallelism gains and may actually be slower than sequential processing.

---

## Common Mistakes

- **Modifying Source During Stream** — Modifying the source collection while a stream is being processed throws `ConcurrentModificationException` if the stream's spliterator detects structural changes. It looks correct because the mutation happens in a separate lambda, and simple test cases with small collections often complete without throwing (the iterator may not have rechecked `modCount` yet). In production, this is non-deterministic — it only manifests under specific collection sizes and timing windows, making it a heisenbug that passes QA and surfaces in production under load.
- **Forgetting to Close I/O Streams** — Streams backed by I/O resources like `Files.lines()`, `Files.walk()`, or `Files.list()` implement `AutoCloseable`. Failing to close them causes file handle leaks that can exhaust the OS file descriptor limit. Always use `try (Stream<String> lines = Files.lines(path)) { ... }`.
- **Stateful Lambdas in Parallel** — Using stateful lambdas in parallel streams causes non-deterministic race conditions because elements from different partitions are processed by different threads. A lambda like `.map(x -> { counter++; return process(x); })` has a data race on `counter`. All lambdas in stream pipelines should be stateless.
- **Missing Primitive Streams** — Failing to use primitive streams for numeric data introduces significant performance overhead from autoboxing and object allocation. A `Stream<Integer>` processing 10 million values allocates 10 million `Integer` objects, causing GC pressure that can be 5-10x slower than the equivalent `IntStream`.
- **Order Assumptions with parallelStream()** — Assuming that `parallelStream()` preserves encounter order or that `forEach()` processes elements in order is a common source of non-deterministic bugs. Use `forEachOrdered()` instead of `forEach()` if processing order matters, but be aware that ordering guarantees reduce parallel performance.
- **sorted().findFirst() Instead of min()** — Calling `sorted().findFirst()` sorts the entire stream O(n log n) just to find the minimum element, which is O(n) with `min()` or `reduce()`. On a dataset of 10 million elements, the difference is 10 seconds of sorting versus 50ms of scanning. Similarly, `sorted().limit(k)` sorts all elements when a partial sort (O(n log k)) would suffice — use a priority queue or custom collector for top-k queries.
- **null in flatMap()** — The function passed to `flatMap()` must never return null; doing so throws `NullPointerException` because `Stream` does not allow null elements. Always return `Stream.empty()` for the no-result case. If the upstream `map()` might produce null, filter before flatMapping: `.filter(Objects::nonNull).flatMap(Function.identity())`.

---

## Real-World Scenarios

### Scenario 1: Real-Time Fraud Detection Pipeline

A payment processing system streams 10K transactions per second through a pipeline of 20 fraud detection rules. The rules include velocity checks (has this user made more than 5 transactions in the last minute?), geographic anomalies (is the IP location consistent with the billing address?), amount thresholds (is this transaction 10x the user's average?), and blacklisted merchant checks. The pipeline must stop evaluation at the first triggered rule for performance (fail-fast) but also produce a complete list of all violations for audit trails.

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

The stream of predicates is evaluated lazily — `findFirst()` is a short-circuit terminal operation that stops at the first matching rule, so for a transaction that triggers the velocity check, the remaining 19 rules are never evaluated. Each rule is individually testable, and the rule list is composable — adding a new rule is a one-line change in the `Stream.of()` call. For the full audit trail, simply change the terminal operation to `.filter(rule -> rule.test(tx)).collect(Collectors.toList())`, which evaluates all rules and collects the triggered ones.

**Why this approach?** An imperative implementation would need a `for` loop over rules, a `break` statement for fail-fast, a `List<String>` to accumulate failures for auditing, and a `boolean` flag to track whether any rule triggered — four mutable state variables scattered across a single method. The stream version encodes these concerns in the choice of terminal operation: `findFirst()` for fail-fast, `collect(toList())` for full evaluation. No state, no branches, no break statements. This is the core value of the declarative model — the what (filter by condition) is separated from the how (short-circuit or full evaluation).

### Scenario 2: Data Warehouse ETL with Column Transformations

A nightly batch job reads 50 million customer records from a staging database, normalizes phone numbers, validates email addresses, geocodes addresses via an external API, and writes the cleaned records to a data warehouse. The job must parallelize across available CPU cores and handle partial failures gracefully without crashing the entire batch.

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

`parallelStream()` distributes CPU-bound normalization (phone format, email regex validation) across all available cores, processing batches in parallel via the common ForkJoinPool. `flatMap` is used for the geocode step because it may fail for some records (invalid address, API timeout) and returns `Stream.empty()` in those cases, gracefully excluding them from the output without crashing the pipeline. Each transformation is an independently testable method, and the pipeline is entirely declarative — no mutable accumulators, no explicit error handling, no looping constructs. For production robustness, consider wrapping the stream in a custom `Spliterator` that periodically persists processing progress to a database for checkpoint-based recovery.

**Why this approach?** An imperative ETL would have a `for` loop with try-catch around each transformation, conditional logic for skipping failed records, and manual thread management for parallelism. The stream version eliminates the try-catch boilerplate by using `flatMap` with `Stream.empty()` on failure — failures are treated as "no result" rather than exceptions. This shifts error handling from exceptions (control flow) to data (empty stream elements), which is more predictable in parallel execution because exceptions in one partition can affect other partitions. The `Objects::nonNull` filter is a safety net that catches any nulls that slip through, making the pipeline robust without exception-handling code scattered through every transformation.

### Scenario 3: Real-Time Dashboard Aggregation

A monitoring system collects server metrics (CPU, memory, disk I/O) from 1000 servers every 10 seconds. It must compute per-service averages, detect anomalies (values more than 3 standard deviations from the mean), and present a real-time dashboard — all within a 5-second processing window to keep the dashboard current.

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

Two stream pipelines process the same source data: the first computes per-service averages using `groupingBy` with a downstream `summarizingDouble` collector that captures count, sum, min, max, and average in a single pass. The second pipeline filters metrics that deviate more than 3 standard deviations from their service's mean and groups the anomalies by service for dashboard highlighting. Stream fusion ensures each pipeline processes elements one at a time, keeping memory O(1) per pipeline — for 1000 servers, 4 metrics, and 6 readings (24K elements), the entire aggregation completes in under 100ms on modern hardware.

**Why this approach?** The imperative alternative would require nested loops: outer loop over services, inner loop over metrics — three passes (compute averages, compute stddev, detect anomalies) with manual state accumulation. The stream version uses two separate pipelines for two separate concerns (aggregation vs. anomaly detection), each self-contained and independently testable. `DoubleSummaryStatistics` captures 5 metrics in one pass without custom collector code. The grouping is done by the framework rather than manual `Map.computeIfAbsent()` calls, reducing boilerplate. The cost is two passes over the same data — acceptable for 24K elements (sub-millisecond each), but for 24M elements, a single pass with a custom collector would be warranted.

---

## Scenario-Based Questions

**Q: A developer writes `Stream<String> stream = list.stream(); stream.forEach(System.out::println); long count = stream.count();` and gets `IllegalStateException: stream has already been operated upon or closed`. They ask: "Can't I reuse the stream? It still has the same filter and map stages defined." What do you tell them?**

A: Streams are single-use pipelines, not reusable query definitions. Think of a stream as a cursor traversing the source collection — once it reaches the end, it cannot be rewound. Even if the pipeline defines the same intermediate operations, the stream does not replay them from the source; the source's `Spliterator` is consumed irreversibly. The fix is to create a new stream for each terminal operation: `list.stream().forEach(System.out::println); long count = list.stream().count();`. If the same pipeline is reused frequently, extract the source + intermediate stages to a method: `Stream<String> pipeline() { return list.stream().filter(s -> s.length() > 3).map(String::toUpperCase); }`. Each call to `pipeline()` creates a fresh stream backed by a fresh `Spliterator` from the collection, allowing unlimited reuse of the pipeline definition.

> **Interview follow-up:** The candidate said streams are single-use but suggested a method that returns a new stream each time. If the stream's source is a `Collection`, each call creates a fresh `Spliterator`. But what if the source is `Files.lines()` — does the same pattern apply, and what about resource cleanup for the returned stream?

**Q: You are building a search autocomplete feature. Users type a query, and you must return the top 10 suggestions from a dictionary of 500K phrases. The suggestions must match by prefix and be ordered by popularity. Users type every keystroke — response must be under 50ms. How do you use streams?**

A: Do not use streams for the online hot path — stream pipeline overhead (spliterator creation, lambda dispatch, collector allocation) adds 1-5ms per evaluation, which is too expensive within a 50ms budget that also includes network latency and prefix matching. Instead, use a `Trie` data structure for the prefix lookup (O(k) time where k is the query length, with zero allocation) and a priority queue for ranking the top results. Streams are appropriate for the offline index build: `dictionary.stream().sorted(byPopularity).collect(Collectors.toList())` pre-sorts the dictionary by popularity before building the trie. The lesson: streams prioritize readability and declarativeness over raw performance — avoid them in latency-critical hot paths where every microsecond counts.

> **Interview follow-up:** The candidate ruled out streams for the hot path but used them for the offline sort. If the dictionary updates in real-time (new phrases added every minute), how would you incrementally update the sorted popularity index without re-sorting 500K phrases on every update?

**Q: You have a microservice that receives a list of order IDs from the API gateway. You need to fetch each order from a downstream service (HTTP call), enrich it with customer data from another service, and return the combined result. Each fetch takes 50-200ms. How do you run these calls concurrently with streams?**

A: Use `CompletableFuture` with a stream pipeline for the fork-join pattern, but do NOT use `parallelStream()` because it uses the shared common `ForkJoinPool` that is also used by parallel stream operations throughout the JVM. Blocking I/O calls in the common pool can starve other components of threads, causing system-wide latency spikes. Instead, create the futures with a dedicated executor:
```java
List<CompletableFuture<Order>> futures = orderIds.stream()
    .map(id -> CompletableFuture.supplyAsync(() -> fetchOrder(id), executor))
    .toList();
List<Order> orders = futures.stream()
    .map(CompletableFuture::join)
    .toList();
```
The first stream creates all HTTP call futures asynchronously (non-blocking, O(1) per submission), and the second stream joins them, blocking the current thread on each future. Using a dedicated `executor` with a bounded thread pool sized to `cores * (1 + waitTime / computeTime)` prevents resource exhaustion and isolates the I/O threads from the compute pool. This pattern provides N-way concurrency limited only by the thread pool size.

> **Interview follow-up:** The candidate used two streams — one to submit all futures, then another to join them. Why is this two-pass approach (submit all, then join all) better than submitting and joining each future in a single `map()` call within the same stream?

**Q: You have a list of 10M transactions and need to compute: total revenue, average transaction value, number of fraudulent transactions, and revenue by merchant category. How do you avoid iterating 4 times?**

A: Use a custom collector or `Collectors.teeing()` (Java 12+) to compute all four aggregations in a single pass over the data. Iterating the 10M list four times means processing 40M elements and thrashing the CPU cache, while a single pass processes 10M elements that likely stay in L2 or L3 cache:
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
For the category grouping in the same pass, `Collectors.teeing()` combines two collectors: one for the aggregate stats and another for the `groupingBy` on merchant category. A single pass over 10M elements is approximately 10ms on modern hardware — four separate passes would be 40ms with higher cache miss rates.

> **Interview follow-up:** The candidate used a custom `Collector` with mutable accumulator arrays. If the stream is parallelized, the combiner function merges partial results — does `teeing()` handle parallel accumulation correctly for both collectors, or does it require the combiner to be associative and thread-safe?

**Q: A stream pipeline processes a file with `Files.lines()`. Halfway through, a line has malformed data and throws a runtime exception. The file handle is never closed. How do you ensure robust resource cleanup?**

A: Always wrap `Files.lines()` in a try-with-resources statement so that `close()` is invoked on the stream even when an exception is thrown inside the pipeline:
```java
try (Stream<String> lines = Files.lines(path)) {
    lines.map(this::parseLine)
         .forEach(this::process);
} catch (IOException e) {
    log.error("Failed to process file", e);
}
```
Without try-with-resources, if a lambda in `map()` or `forEach()` throws an uncaught `RuntimeException`, the stream's `onClose()` handlers never execute and the underlying `FileChannel` is leaked. This leak is particularly dangerous in long-running applications because the operating system limits the number of open file descriptors per process, and once exhausted, all file I/O operations fail with `IOException: Too many open files`. For production robustness, also catch exceptions within individual `map()` calls, log the problematic line, filter out the null, and let the pipeline continue with the remaining records.

> **Interview follow-up:** The candidate mentioned catching exceptions within `map()` and filtering nulls to continue processing. If a line causes an exception, the stream pipeline catches it, logs, and returns null — does `forEach()` then attempt to process the null element? How would you design the pipeline to skip that element entirely?

**Q: You need to paginate through a large dataset (1M records) returned from a database cursor in batches of 100. The cursor is stateful and not thread-safe. How do you process all records using streams without loading them all into memory?**

A: Create a custom `Spliterator` that wraps the database cursor and provides a pull-based, on-demand stream that reads records lazily:
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
The custom spliterator reads one record at a time from the database cursor via `tryAdvance()`, which is called by the stream pipeline for each element. The pipeline chains operations (filter, map, collect) without ever holding all 1M records in memory because the spliterator pulls records on demand. For parallel processing, implement `trySplit()` to partition the cursor by key range, but note that the cursor itself is not thread-safe — each partition must use its own cursor or the data must be pre-partitioned in the database query.

> **Interview follow-up:** The candidate mentioned implementing `trySplit()` for parallel processing by key range. If the data is not evenly distributed (e.g., 90% of records fall in a single key range), how would the work imbalance affect parallel performance, and what alternative splitting strategy would you use?

**Q: A system streams sensor readings at 100K events per second. You need to compute the moving average over a 5-second sliding window. The stream is infinite. How do you maintain only the relevant data?**

A: Use a ring (circular) buffer backed by a fixed-size `double[]` array, combined with a running sum that avoids iterating the entire window on every event:
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
This is O(1) per event with zero heap allocation after the initial buffer allocation — no `ArrayList` resizing, no `LinkedList` node creation, no garbage collection pressure. The running sum avoids iterating the entire 500K-element window on each event, which would be prohibitive at 100K events per second. After the buffer fills, each new event evicts the oldest value, subtracts it from the sum, adds the new value, and computes the average — all in a few nanoseconds. In real-time stream processing, this allocation-free pattern is essential because GC pauses at 100K events per second would cause data loss while the application is stopped.

> **Interview follow-up:** The candidate used a running sum to avoid iterating the window. What happens if the window needs to support min, max, or percentile (p99) instead of just average — can the running-sum trick be adapted, or do those metrics require a different data structure like a segment tree or a deque for a monotonic queue?

**Q: You are building a data validation service. Each record must pass 15 validation rules (some are cheap string checks, others are expensive DB lookups). You want to fail fast — stop at the first violation — but also want to collect ALL violations for the audit log. How do you design this with streams?**

A: Use two separate stream pipelines with different terminal operations for the two different requirements. For the fail-fast path used in the request thread, use `rules.stream().filter(r -> !r.test(record)).findFirst()` — the `findFirst()` short-circuits immediately on the first rule violation and returns an `Optional` of the failing rule, allowing the API to return an error response to the client in milliseconds instead of waiting for all 15 rules to evaluate. For the full audit path, use `rules.stream().map(r -> r.validate(record)).filter(Objects::nonNull).collect(toList())`, which evaluates all rules, collects every violation message, and logs the complete results to the audit store. Submit the full validation to a background `ExecutorService` so the request thread returns immediately with either a pass or the first failure, while the audit trail is populated asynchronously without affecting response latency. The key architectural insight is that streams support both short-circuit and full-evaluation modes through different terminal operations — you do not need separate implementations.

> **Interview follow-up:** The candidate suggested submitting the full audit validation to a background `ExecutorService`. If the request thread's fail-fast check passes but the background audit pipeline finds violations, how would you handle the inconsistency — the client was told the record passed, but auditing later flags it as a violation?

**Q: You have a stream of events, each with a `LocalDateTime timestamp`. Events can arrive out of order (up to 30 seconds late). You need to group events into 1-minute windows and process each window in chronological order. How do you handle the watermark and late data?**

A: This requires a stateful collector or a data structure that manages windows with a watermark — the stream API alone is insufficient for event-time processing with out-of-order data:
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
The watermark is the point in event-time before which the system considers all events to have been received — events with timestamps older than `currentTime - watermark` are discarded or processed in a late-data handler. The `TreeMap` keeps windows sorted by their timestamp key, and `headMap()` efficiently retrieves and removes all completed windows (those whose end time is older than the watermark). This is exactly how Apache Flink's `TumblingEventTimeWindows` and Kafka Streams' windowed aggregations work internally.

> **Interview follow-up:** The candidate used a `TreeMap` with `headMap()` to evict completed windows. If late-arriving events arrive after `headMap().clear()` has already been called, they are lost. How would you implement a late-data handler that stores these events separately for reprocessing rather than silently dropping them?

**Q: You need to join two streams: a stream of Orders and a stream of Payments. Each order has multiple payments. You must enrich each order with its total paid amount. Both streams are large (millions). How do you join them with streams?**

A: Use a two-pass hash join approach: the first stream pass collects payments into a lookup map, and the second pass enriches orders using map lookups:
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
The first pass builds a `HashMap` in O(n) time and O(n) memory where n is the number of payments, using `groupingBy` with a downstream `reducing` collector that sums amounts per order ID. The second pass iterates orders and performs O(1) lookups into the map, producing the enriched result. For datasets too large to fit in memory, streams are not the right tool — use an external sort-merge join (sort both datasets by the join key and merge with a cursor) or push the join down to the database. Streams excel for in-memory hash joins but have no native support for streaming joins across unbounded sources, which is where Apache Flink, Kafka Streams, or Spark Structured Streaming are appropriate.

> **Interview follow-up:** The candidate mentioned an external sort-merge join for datasets too large for memory. If both the orders and payments streams are infinite and unbounded (a live event stream), how would you implement a streaming join that accounts for late payments arriving minutes after the corresponding order — does the hash-join approach still work, or does it require a state store with TTL?

**Q: A stream of transactions produces a side effect (logging) in `peek()` for debugging. The pipeline is later parallelized, and the log output is jumbled across threads. How do you safely add logging to a stream pipeline?**

A: `peek()` is documented as a debugging aid and is not guaranteed to execute in order when the stream is parallelized — elements from different partitions are logged by different threads, and the output interleaves non-deterministically. For safe, ordered logging with parallel streams, use `forEachOrdered()` instead of `forEach()` as the terminal operation:
```java
transactions.parallelStream()
    .filter(t -> t.getAmount() > 1000)
    .forEachOrdered(t -> log.info("Large tx: {}", t.getId()));
```
`forEachOrdered()` respects encounter order, so log output is deterministic and ordered even in parallel execution — though the ordering guarantee reduces parallelism because elements must be re-sequenced after processing. A better approach is to separate logging from the pipeline entirely: log in the terminal operation, not in `peek()`, or use `collect(Collectors.collectingAndThen(toList(), list -> { list.forEach(t -> log.info(...)); return list; }))` which logs the entire result after the pipeline completes, providing clean, ordered, and deterministic log output regardless of parallelism.

> **Interview follow-up:** The candidate suggested `collectingAndThen` which materializes the entire stream into a list before logging. For a stream of 10 million transactions, this list holds 10 million objects in memory at once. How would you add ordered logging to a parallel stream without materializing the entire result set first?

---

## Interview Questions

**What is the difference between intermediate and terminal operations?** Intermediate operations like `filter()` and `map()` return a new `Stream` and are evaluated lazily — they do no work until a terminal operation triggers the pipeline. Terminal operations like `collect()`, `forEach()`, and `reduce()` produce a concrete result or side effect and consume the stream, after which the stream cannot be reused. A pipeline must have exactly one terminal operation, and the stream is closed after it completes.

**What is the difference between `map()` and `flatMap()`?** `map()` performs a one-to-one transformation: for every input element, exactly one output element is produced. `flatMap()` performs a one-to-many transformation: the function returns a `Stream` of zero, one, or many elements, and all resulting streams are concatenated into a single output stream. `flatMap()` is used to flatten nested collections, to handle optional results by returning `Stream.empty()` for missing values, and to expand a single record into multiple output records.

**What is stream fusion and how does it improve performance?** Stream fusion is an optimization where multiple adjacent intermediate operations are compiled into a single pass over the data, eliminating intermediate storage and reducing memory traffic. Instead of creating an intermediate collection for `filter()` and then iterating again for `map()`, the JVM processes each element through both `filter()` and `map()` before advancing to the next element. This improves CPU cache locality because the element stays in L1 cache through all operations, and it eliminates the overhead of constructing intermediate stream objects between operations.

**What is the difference between `findFirst()` and `findAny()`?** `findFirst()` returns the first element of the stream respecting its encounter order, which requires coordination in parallel execution. `findAny()` returns any element without ordering guarantees, making it significantly faster in parallel streams because any thread can return its match without waiting for other partitions. In sequential streams, both typically return the same result. Use `findAny()` when order does not matter for maximum parallel performance.

**How does `Collectors.groupingBy()` work internally?** `groupingBy(Function)` returns a `Collector` that classifies elements into a `Map<K, List<V>>` by applying the classifier function to each element and accumulating results in a list per key. Internally, it uses a `HashMap` as the mutable accumulator and `ArrayList` for the value collections. The overloaded versions accept a downstream collector for nested aggregations (e.g., `counting()`, `mapping()`, `summingInt()`) and a map factory supplier for controlling the map implementation (e.g., `LinkedHashMap` for insertion-order or `ConcurrentHashMap` for thread-safe accumulation).

**What is the difference between `Collection.stream()` and `Stream.of(collection)`?** `collection.stream()` returns a stream of the elements contained in the collection. `Stream.of(collection)` treats the entire collection as a single element, returning `Stream<List<T>>` rather than `Stream<T>`. The `Stream.of()` method uses varargs, and when called with a single array argument (e.g., `String[]`), it treats the array elements as the stream elements — this works for arrays but not for collections. This distinction is a common source of bugs where the developer expects `Stream.of(list)` to produce a stream of list elements but instead gets a stream containing the list itself.

**When should you use `reduce()` vs `collect()`?** `reduce()` performs an immutable reduction using an associative combining function — it takes two values and produces one, without modifying any container. Common uses are sum, product, min, max, and string concatenation. `collect()` performs a mutable reduction by accumulating elements into a mutable container like a `List`, `Set`, `Map`, or `StringBuilder`. Use `collect()` for building collections or performing terminal operations that need mutable state, and use `reduce()` for arithmetic aggregations. Mixing them up — using `reduce()` with mutable accumulators or `collect()` with simple sums — leads to correctness or performance issues.

**What is the purpose of `IntStream`, `LongStream`, and `DoubleStream`?** These primitive stream specializations avoid the autoboxing overhead of `Stream<Integer>` where each `int` value is wrapped in an `Integer` object consuming 16-28 bytes of heap. For large numeric datasets, primitive streams are 5-10x faster and use a fraction of the memory. They also provide specialized terminal operations like `sum()`, `average()`, `summaryStatistics()`, and factory methods like `range()` and `rangeClosed()` that are not available on object streams.

**Why are streams consumable only once?** Streams are designed as single-use pipelines because most stream sources can be traversed only once — network streams, file lines, iterator wrappers, and generator functions all produce elements that are consumed as they are read. Even collection-backed streams are single-use because intermediate results are not cached by the stream implementation; reusing a stream would require re-executing the entire pipeline from the source. Attempting a second terminal operation throws `IllegalStateException: stream has already been operated upon or closed`.

**What is the difference between sequential and parallel streams in terms of thread safety?** Sequential streams execute entirely on the calling thread with no concurrency concerns. Parallel streams split the source into segments using the `Spliterator`'s `trySplit()` method and process each segment on a separate thread from the common `ForkJoinPool`. Parallel streams require that all intermediate operation lambdas be stateless and non-interfering — they must not read or write shared mutable state. Compound operations like `findAny()` and `unordered().skip()` are more efficient in parallel because they exploit non-determinism, while ordered operations like `findFirst()` and `forEachOrdered()` reduce parallel efficiency.

---

## Developer Recommendations

- **Prefer primitive streams over boxed streams** — The performance difference is substantial: `IntStream.range(0, 1_000_000).sum()` avoids boxing one million integers, while `Stream<Integer>` creates 28 bytes of garbage per element (16-byte object header + 4-byte value + padding). For hot paths in data processing, analytics, or scientific computing, the choice between `IntStream` and `Stream<Integer>` can be the difference between 50ms and 500ms with visible GC pauses.
- **Use stream.toList() over collect(Collectors.toList())** — `stream.toList()` (Java 16+) returns an immutable list guaranteed to contain no null elements, communicates intent clearly, and is shorter to write. `Collectors.toList()` returns a mutable `ArrayList` whose type is an implementation detail — callers can add or remove elements, potentially corrupting the result. When you genuinely need a mutable list, use `collect(Collectors.toCollection(ArrayList::new))`.
- **Avoid parallelStream() for I/O-bound operations** — The common `ForkJoinPool` is shared system-wide — blocking I/O operations in parallel streams consume threads that other parallel stream operations and `CompletableFuture` pipelines depend on. Reserve `parallelStream()` for CPU-intensive operations on datasets larger than 10K elements where each element requires significant computation. For I/O-bound workloads, use `CompletableFuture` with a dedicated `ExecutorService`.
- **Use method references over lambdas** — Method references improve both readability and performance. `orders.stream().map(Order::getTotal)` is more concise than `orders.stream().map(o -> o.getTotal())`, and non-capturing lambdas are cached by the JVM but method references have even lower invocation overhead. For complex multi-line logic, extract to a named method and reference it.
- **Avoid stateful lambdas in parallel streams** — Shared mutable state introduces race conditions that produce non-deterministic results. A lambda like `.map(x -> { counter++; return transform(x); })` has a data race on `counter`. All lambdas should be stateless, meaning their output depends only on their input and captured immutable variables. Use `collect()` for thread-safe accumulation.
- **Use try-with-resources for I/O-backed streams** — `Files.lines()`, `Files.walk()`, `Files.list()`, and `Files.find()` return streams backed by resources that must be explicitly closed. The try-with-resources construct ensures `close()` is called even when the pipeline throws an exception: `try (Stream<String> lines = Files.lines(path)) { lines.forEach(...); }`.
- **Use Collectors.teeing() for multiple aggregations** — When you need two different aggregations from the same stream (Java 12+), `teeing()` sends each element to two downstream collectors and merges their results with a bi-function. This avoids iterating the stream twice and halves the processing time for large datasets. Before Java 12, create a custom `Collector` with a mutable holder object.
