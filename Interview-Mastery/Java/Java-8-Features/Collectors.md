# Collectors

## Overview
- **Definition**: `Collectors` is a utility class in `java.util.stream` providing implementations of `Collector` for common reduction operations like accumulating elements into collections, grouping, partitioning, summarizing, and joining.
- **Why It Exists**: Without `Collectors`, terminal operations on streams would require manual mutable reduction with verbose `collect()` calls. `Collectors` provides reusable, composable, and optimized collectors.
- **Key Concepts**: Mutable reduction, downstream collectors, finisher functions, characteristics (CONCURRENT, UNORDERED, IDENTITY_FINISH).
- Every `Collector` implements four functions: supplier, accumulator, combiner, and finisher.
- Collectors can be composed: a downstream collector receives results of an upstream collector.
- `Collector.Characteristics` hints at optimization opportunities for the runtime.

## Core Concepts

### toList()

- Accumulates stream elements into a `List`:
```java
List<String> list = Stream.of("a", "b", "c").collect(Collectors.toList());
```

- `toList()` returns a mutable `ArrayList` (Java 8-15; Java 16+ may vary).
- Collecting with `toUnmodifiableList()` (Java 10+):
```java
List<String> unmodifiable = Stream.of("x", "y")
    .collect(Collectors.toUnmodifiableList());
```

- Collecting to a specific list implementation:
```java
ArrayList<String> arrayList = Stream.of("a", "b")
    .collect(Collectors.toCollection(ArrayList::new));
LinkedList<String> linkedList = Stream.of("a", "b")
    .collect(Collectors.toCollection(LinkedList::new));
```

### toSet()

- Accumulates elements into a `Set` (typically `HashSet`):
```java
Set<String> set = Stream.of("a", "b", "a").collect(Collectors.toSet());
```

- Duplicates are removed based on `equals()` / `hashCode()`.
- Collecting to a specific Set:
```java
TreeSet<String> sorted = Stream.of("c", "a", "b")
    .collect(Collectors.toCollection(TreeSet::new));
```

### toMap()

- Converts stream to a `Map` with key and value mappers:
```java
Map<Integer, String> map = Stream.of("a", "b", "c")
    .collect(Collectors.toMap(String::length, Function.identity()));
```

- Handling duplicate keys with merge function:
```java
Map<Integer, String> map = Stream.of("a", "bb", "c", "dd")
    .collect(Collectors.toMap(
        String::length,
        Function.identity(),
        (existing, replacement) -> existing + "," + replacement
    ));
// {1="a,c", 2="bb,dd"}
```

- `toMap` throws `IllegalStateException` on duplicate keys without a merge function.
- Specifying map implementation:
```java
TreeMap<Integer, String> treeMap = Stream.of("a", "b")
    .collect(Collectors.toMap(
        String::length, Function.identity(),
        (a, b) -> a, TreeMap::new
    ));
```

### groupingBy

- Groups elements by a classifier function:
```java
Map<Integer, List<String>> byLength = Stream.of("a", "bb", "cc", "d")
    .collect(Collectors.groupingBy(String::length));
// {1=["a", "d"], 2=["bb", "cc"]}
```

- `groupingBy` with a downstream collector:
```java
Map<Integer, Long> countByLength = Stream.of("a", "bb", "cc", "d")
    .collect(Collectors.groupingBy(String::length, Collectors.counting()));
// {1=2, 2=2}
```

- `groupingBy` with map supplier and downstream:
```java
Map<Integer, Set<String>> setByLength = Stream.of("a", "bb", "a", "cc")
    .collect(Collectors.groupingBy(
        String::length,
        TreeMap::new,
        Collectors.toSet()
    ));
```

- `groupingByConcurrent` for parallel streams with concurrent maps.

### partitioningBy

- Partitions elements into two groups (true/false) based on a predicate:
```java
Map<Boolean, List<Integer>> evenOdd = Stream.of(1, 2, 3, 4, 5)
    .collect(Collectors.partitioningBy(n -> n % 2 == 0));
// {false=[1, 3, 5], true=[2, 4]}
```

- `partitioningBy` with a downstream collector:
```java
Map<Boolean, Long> countEvenOdd = Stream.of(1, 2, 3, 4, 5)
    .collect(Collectors.partitioningBy(n -> n % 2 == 0, Collectors.counting()));
// {false=3, true=2}
```

- Unlike `groupingBy`, `partitioningBy` always yields exactly two keys.

### joining

- Concatenates strings from the stream:
```java
String joined = Stream.of("a", "b", "c").collect(Collectors.joining());
// "abc"
```

- `joining` with delimiter:
```java
String joined = Stream.of("a", "b", "c").collect(Collectors.joining(", "));
// "a, b, c"
```

- `joining` with delimiter, prefix, and suffix:
```java
String joined = Stream.of("a", "b", "c")
    .collect(Collectors.joining(", ", "[", "]"));
// "[a, b, c]"
```

- Performance: `joining` uses `StringBuilder` internally, more efficient than manual reduction.

### summarizingInt / Long / Double

- Computes count, sum, min, average, max in one pass:
```java
IntSummaryStatistics stats = Stream.of(1, 2, 3, 4, 5)
    .collect(Collectors.summarizingInt(Integer::intValue));

System.out.println(stats.getSum());      // 15
System.out.println(stats.getCount());    // 5
System.out.println(stats.getMin());      // 1
System.out.println(stats.getMax());      // 5
System.out.println(stats.getAverage());  // 3.0
```

- `LongSummaryStatistics` and `DoubleSummaryStatistics` work identically.

### averagingInt / Long / Double

- Computes the average of numeric values:
```java
Double avg = Stream.of(1, 2, 3, 4, 5)
    .collect(Collectors.averagingInt(Integer::intValue));
// 3.0
```

- Returns `Double.NaN` for empty stream (no `Optional`, unlike `Stream.average()` which returns `OptionalDouble`).

### counting

- Counts elements in the stream:
```java
Long count = Stream.of("a", "b", "c").collect(Collectors.counting());
```

- Equivalent to `Stream.count()` but useful as a downstream collector.

### mapping

- Applies a mapping function before collecting downstream:
```java
List<String> upper = Stream.of("a", "b", "c")
    .collect(Collectors.mapping(String::toUpperCase, Collectors.toList()));
// ["A", "B", "C"]
```

- `mapping` is primarily useful as a downstream collector with `groupingBy` or `partitioningBy`:
```java
Map<Integer, List<String>> mapped = Stream.of("Alice", "Bob", "Charlie")
    .collect(Collectors.groupingBy(
        String::length,
        Collectors.mapping(String::toUpperCase, Collectors.toList())
    ));
```

### filtering

- Filters elements before collecting downstream (Java 9+):
```java
List<Integer> even = Stream.of(1, 2, 3, 4)
    .collect(Collectors.filtering(n -> n % 2 == 0, Collectors.toList()));
// [2, 4]
```

- Useful as downstream in `groupingBy`:
```java
Map<Integer, List<String>> filtered = Stream.of("a", "bb", "ccc", "dd")
    .collect(Collectors.groupingBy(
        String::length,
        Collectors.filtering(s -> s.startsWith("b"), Collectors.toList())
    ));
```

### flatMapping

- Flattens nested stream elements before collecting (Java 9+):
```java
List<String> words = Stream.of("hello world", "foo bar")
    .collect(Collectors.flatMapping(
        s -> Stream.of(s.split(" ")),
        Collectors.toList()
    ));
// ["hello", "world", "foo", "bar"]
```

- Downstream usage with `groupingBy`:
```java
Map<String, List<String>> grouped = Stream.of("abc def", "ghi jkl")
    .collect(Collectors.groupingBy(
        s -> s.substring(0, 1),
        Collectors.flatMapping(
            s -> Stream.of(s.split(" ")),
            Collectors.toList()
        )
    ));
```

### collectingAndThen

- Adjusts the result of a collector with a finishing function:
```java
List<String> unmodifiable = Stream.of("a", "b")
    .collect(Collectors.collectingAndThen(
        Collectors.toList(),
        Collections::unmodifiableList
    ));
```

- Useful for converting collection types:
```java
Set<String> set = Stream.of("a", "b", "a")
    .collect(Collectors.collectingAndThen(
        Collectors.toSet(),
        Collections::unmodifiableSet
    ));
```

- Computing size after collection:
```java
Integer size = Stream.of("a", "b", "c")
    .collect(Collectors.collectingAndThen(
        Collectors.toList(),
        List::size
    ));
// 3
```

### teeing

- Combines results of two downstream collectors (Java 12+):
```java
Double average = Stream.of(1, 2, 3, 4, 5, 6)
    .collect(Collectors.teeing(
        Collectors.summingInt(Integer::intValue),
        Collectors.counting(),
        (sum, count) -> (double) sum / count
    ));
// 3.5
```

- Computing min and max simultaneously:
```java
Map<String, Integer> minMax = Stream.of(3, 1, 5, 2, 4)
    .collect(Collectors.teeing(
        Collectors.reducing(Integer::min),
        Collectors.reducing(Integer::max),
        (min, max) -> Map.of("min", min.get(), "max", max.get())
    ));
```

### Custom Collector Implementation

- Anatomy of a `Collector`:
```java
public interface Collector<T, A, R> {
    Supplier<A> supplier();
    BiConsumer<A, T> accumulator();
    BinaryOperator<A> combiner();
    Function<A, R> finisher();
    Set<Characteristics> characteristics();
}
```

- Custom collector collecting elements into a comma-separated string:
```java
Collector<String, StringBuilder, String> commaCollector = Collector.of(
    StringBuilder::new,
    (sb, s) -> {
        if (sb.length() > 0) sb.append(", ");
        sb.append(s);
    },
    (sb1, sb2) -> {
        if (sb2.length() > 0 && sb1.length() > 0) sb1.append(", ");
        sb1.append(sb2);
        return sb1;
    },
    StringBuilder::toString,
    Collector.Characteristics.UNORDERED
);

String result = Stream.of("a", "b", "c").collect(commaCollector);
// "a, b, c"
```

- Custom collector with `IDENTITY_FINISH`:
```java
Collector<Integer, List<Integer>, List<Integer>> topThree = Collector.of(
    ArrayList::new,
    (list, item) -> {
        list.add(item);
        list.sort(Comparator.reverseOrder());
        if (list.size() > 3) list.remove(3);
    },
    (l1, l2) -> {
        l1.addAll(l2);
        l1.sort(Comparator.reverseOrder());
        while (l1.size() > 3) l1.remove(3);
        return l1;
    },
    Collector.Characteristics.IDENTITY_FINISH,
    Collector.Characteristics.CONCURRENT
);
```

- Collector with `Collector.of` overload without finisher (uses identity):
```java
Collector<Integer, List<Integer>, List<Integer>> collector =
    Collector.of(ArrayList::new, List::add, (a, b) -> {
        a.addAll(b);
        return a;
    });
```

- Custom collector for immutable collection:
```java
Collector<String, ?, ImmutableList<String>> immutableCollector =
    Collector.<String, ImmutableList.Builder<String>, ImmutableList<String>>of(
        ImmutableList::builder,
        ImmutableList.Builder::add,
        (b1, b2) -> b1.addAll(b2.build()),
        ImmutableList.Builder::build
    );
```

- Using `Collectors.reducing` for custom reduction:
```java
Optional<Integer> max = Stream.of(1, 5, 2, 8, 3)
    .collect(Collectors.reducing(Integer::max));
// Optional[8]
```

## Common Mistakes

- **Mistake: Assuming `toList()` returns an immutable list**.
  - Why it looks correct: Some newer JDK versions return unmodifiable lists from `toList()`.
```java
// BAD: Might throw UnsupportedOperationException
List<String> list = someStream.collect(Collectors.toList());
list.add("x"); // OK in Java 8, fails in newer versions with List.of() streams

// Use toCollection(ArrayList::new) for guaranteed mutable list
List<String> safe = someStream.collect(Collectors.toCollection(ArrayList::new));
```

- **Mistake: Not providing a merge function for `toMap` when keys can duplicate**.
  - Why it looks correct: It works with unique keys during testing.
```java
// BAD: Throws IllegalStateException if duplicate keys appear
Map<Integer, String> map = stream.collect(Collectors.toMap(
    String::length, Function.identity()
));

// GOOD: Handle duplicates
Map<Integer, String> map = stream.collect(Collectors.toMap(
    String::length, Function.identity(),
    (a, b) -> a + "," + b
));
```

- **Mistake: Using `groupingBy` when you need a fixed two-way split** (use `partitioningBy`).
  - Why it looks correct: Both return a Map; `groupingBy` handles the job.
```java
// BAD: Grouping by boolean predicate — unnecessary Map overhead
Map<Boolean, List<Integer>> map = stream.collect(
    Collectors.groupingBy(n -> n % 2 == 0));

// GOOD: partitioningBy is more efficient and type-safe
Map<Boolean, List<Integer>> map = stream.collect(
    Collectors.partitioningBy(n -> n % 2 == 0));
```

- **Mistake: Forgetting that `averagingInt` returns 0 for empty streams, not `Optional`**.
  - Why it looks correct: It returns a `Double`, so you assume it reflects actual data.
```java
double avg = emptyStream.collect(Collectors.averagingInt(x -> x));
// Returns 0.0, not an error — silently misleading

// Use a custom collector if you need explicit empty handling
OptionalDouble safeAvg = emptyStream.mapToInt(x -> x).average();
```

- **Mistake: Using `collectingAndThen` with `IDENTITY_FINISH` collectors unnecessarily**.
  - Why it looks correct: The code compiles and runs.
```java
// BAD: Redundant — toList() is already IDENTITY_FINISH
List<String> list = stream.collect(Collectors.collectingAndThen(
    Collectors.toList(), Function.identity()));
// Just use:
List<String> list = stream.collect(Collectors.toList());
```

- **Mistake: Assuming `groupingBy` preserves insertion order**.
  - Why it looks correct: Small datasets appear ordered.
```java
// BAD: HashMap-based — order not guaranteed
Map<Integer, List<String>> map = stream.collect(
    Collectors.groupingBy(String::length));

// GOOD: Preserve order with LinkedHashMap
Map<Integer, List<String>> map = stream.collect(
    Collectors.groupingBy(String::length, LinkedHashMap::new, Collectors.toList()));
```

- **Mistake: Using `toMap` with identity function when values should be collected**.
  - Why it looks correct: It compiles and seems to "work" for unique keys.
```java
// Instead of this (loses all but last duplicate):
Map<String, String> map = names.stream()
    .collect(Collectors.toMap(Function.identity(), Function.identity(), (a, b) -> b));

// Consider groupingBy if you need all values:
Map<String, List<String>> grouped = names.stream()
    .collect(Collectors.groupingBy(Function.identity()));
```

## Real-World Scenarios

- **Scenario 1: Grouping sales by department and computing total revenue**.
```java
Map<String, Double> revenueByDept = sales.stream()
    .collect(Collectors.groupingBy(
        Sale::getDepartment,
        Collectors.summingDouble(Sale::getAmount)
    ));
```

- **Scenario 2: Building an index from words to line numbers**.
```java
Map<String, Set<Integer>> index = lines.stream()
    .flatMap(line -> Stream.of(line.split("\\s+")))
    .collect(Collectors.groupingBy(
        word -> word.toLowerCase(),
        TreeMap::new,
        Collectors.mapping(
            word -> lines.indexOf(word),
            Collectors.toCollection(TreeSet::new)
        )
    ));
```

- **Scenario 3: Partitioning transactions into valid and fraudulent**.
```java
Map<Boolean, List<Transaction>> partitioned = transactions.stream()
    .collect(Collectors.partitioningBy(
        tx -> tx.riskScore() > 0.8
    ));
List<Transaction> fraudulent = partitioned.get(true);
List<Transaction> valid = partitioned.get(false);
```

- **Scenario 4: Computing summary statistics for a data pipeline**.
```java
IntSummaryStatistics stats = orders.stream()
    .mapToInt(Order::getTotal)
    .summaryStatistics();
report.setStats(stats);
```

- **Scenario 5: Generating a CSV string from a list of objects**.
```java
String csv = employees.stream()
    .map(Employee::toCsvRow)
    .collect(Collectors.joining("\n", "name,age,dept\n", ""));
```

- **Scenario 6: Categorizing products with multiple tags via `flatMapping`**.
```java
Map<String, List<Product>> byTag = products.stream()
    .collect(Collectors.groupingBy(
        Product::getCategory,
        Collectors.flatMapping(
            p -> p.getTags().stream(),
            Collectors.toList()
        )
    ));
```

- **Scenario 7: Building a frequency map with `groupingBy` + `counting`**.
```java
Map<String, Long> wordFreq = words.stream()
    .collect(Collectors.groupingBy(
        word -> word.toLowerCase(),
        Collectors.counting()
    ));
```

- **Scenario 8: Collecting into an immutable collection for API response**.
```java
List<String> safeResult = data.stream()
    .map(this::sanitize)
    .collect(Collectors.collectingAndThen(
        Collectors.toCollection(ArrayList::new),
        Collections::unmodifiableList
    ));
```

## Scenario-Based Questions

- **Question: You have a stream of `Order` objects. Group them by customer and compute the total amount per customer.**
```java
Map<Customer, BigDecimal> totals = orders.stream()
    .collect(Collectors.groupingBy(
        Order::getCustomer,
        Collectors.reducing(
            BigDecimal.ZERO,
            Order::getAmount,
            BigDecimal::add
        )
    ));
```

- **Question: Split a list of numbers into positive/negative and find max in each partition.**
```java
Map<Boolean, Optional<Integer>> maxBySign = numbers.stream()
    .collect(Collectors.partitioningBy(
        n -> n >= 0,
        Collectors.maxBy(Integer::compare)
    ));
```

- **Question: Convert a stream of words into a comma-separated string sorted alphabetically.**
```java
String sortedCsv = words.stream()
    .sorted()
    .collect(Collectors.joining(", "));
```

- **Question: Group employees by department and collect only their names.**
```java
Map<String, List<String>> namesByDept = employees.stream()
    .collect(Collectors.groupingBy(
        Employee::getDepartment,
        Collectors.mapping(Employee::getName, Collectors.toList())
    ));
```

- **Question: Count occurrences of each word in a paragraph.**
```java
Map<String, Long> freq = Pattern.compile("\\W+")
    .splitAsStream(paragraph)
    .filter(word -> !word.isEmpty())
    .map(String::toLowerCase)
    .collect(Collectors.groupingBy(
        Function.identity(),
        Collectors.counting()
    ));
```

## Interview Questions

- What is a `Collector` and what are its four core functions?
- What are `Collector.Characteristics` and how do they affect parallel collect?
- What is the difference between `groupingBy` and `partitioningBy`?
- When does `toMap` throw `IllegalStateException` and how do you handle it?
- How would you create an unmodifiable list using `collectingAndThen`?
- What is `teeing` and when would you use it?
- How does `flatMapping` differ from `mapping` in the context of downstream collectors?
- What is the purpose of the finisher function in a custom `Collector`?
- How would you implement a custom collector that limits results to top N elements?
- What is the difference between `joining()` and manual `StringBuilder` reduction?
- How does `summarizingInt` differ from calling `mapToInt().summaryStatistics()`?
- When would you use `toCollection()` over `toList()` or `toSet()`?
- Can you combine two downstream collectors without `teeing`? How?
- What happens to the combiner in a sequential stream?
- How do you group by a composite key in `groupingBy`?

## Developer Recommendations

- Prefer `toCollection(ArrayList::new)` over `toList()` when you need mutability guarantees.
- Always provide a merge function for `toMap` — even if you "know" keys are unique, protect against production data surprises.
- Use `partitioningBy` for boolean predicates instead of `groupingBy` for better performance and clarity.
- Use `collectingAndThen` to wrap collections in unmodifiable wrappers before returning from APIs.
- Prefer `summarizingInt` family over multiple collectors when you need count, sum, min, max, and average — it does one pass.
- Use `groupingBy` with `LinkedHashMap::new` supplier to preserve encounter order.
- Use `flatMapping` downstream collector instead of first flattening the stream to avoid intermediate streams.
- For CSV or JSON array string building, use `joining` with delimiters rather than manual reduction.
- Avoid sharing mutable collectors across threads in parallel streams without proper understanding of combiner behavior.
- Write custom collectors when the built-in ones don't fit, but always measure performance impact.
- In production stories, replacing a `groupingBy` + manual merging loop with built-in `groupingBy` reduced a batch process from 15 minutes to 45 seconds by leveraging parallel stream support.
- The `teeing` collector is invaluable for dual-pass aggregate reporting — it halves the number of stream passes.
- Test custom collectors with both sequential and parallel streams to verify the combiner is correct and efficient.
- Document explicit type parameters for custom collectors to improve readability and catch type errors at compile time.
