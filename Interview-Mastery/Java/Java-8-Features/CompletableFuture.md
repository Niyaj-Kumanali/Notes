# CompletableFuture

## Overview
- **Definition**: `CompletableFuture<T>` is a `Future` and `CompletionStage` introduced in Java 8 for asynchronous, non-blocking computation with manual completion and declarative composition.
- **Why It Exists**: Traditional `Future` from Java 5 lacks callback mechanisms, composition, error handling, and manual completion. `CompletableFuture` fixes all of these.
- **Key Concepts**: Asynchronous execution, callback chaining, composing independent/dependent futures, combining results, failure recovery, and timed waiting.
- `CompletableFuture` implements both `Future` and `CompletionStage`.
- `CompletionStage` is a contract for composing asynchronous steps.
- A `CompletableFuture` can be completed explicitly via `complete(T)` or `completeExceptionally(Throwable)`.
- ForkJoinPool.commonPool() is the default executor unless overridden.
- All non-async methods run in the calling thread unless otherwise noted.
- Methods ending in `Async` accept an optional `Executor` parameter.

## Core Concepts

### Creating CompletableFuture

- Create a completed future:
```java
CompletableFuture<String> completed = CompletableFuture.completedFuture("done");
```

- Create an incomplete future (manual completion):
```java
CompletableFuture<String> manual = new CompletableFuture<>();
manual.complete("hello");
```

- Run a task asynchronously with `runAsync` (Runnable, no result):
```java
CompletableFuture<Void> future = CompletableFuture.runAsync(() -> {
    System.out.println("Running in thread: " + Thread.currentThread().getName());
});
```

- Run a task asynchronously with `supplyAsync` (Supplier, returns result):
```java
CompletableFuture<String> future = CompletableFuture.supplyAsync(() -> {
    return "Result from " + Thread.currentThread().getName();
});
```

- Supply async with custom Executor:
```java
ExecutorService executor = Executors.newFixedThreadPool(4);
CompletableFuture<String> future = CompletableFuture.supplyAsync(() -> {
    return "Custom executor result";
}, executor);
```

### Callback Chaining

- `thenApply` — transform result (Function, like `Stream.map`):
```java
CompletableFuture<Integer> future = CompletableFuture.supplyAsync(() -> "42")
    .thenApply(Integer::parseInt);
System.out.println(future.get()); // 42
```

- `thenAccept` — consume result (Consumer, no return):
```java
CompletableFuture.supplyAsync(() -> "Hello")
    .thenAccept(System.out::println);
```

- `thenRun` — run after completion (Runnable, ignores result):
```java
CompletableFuture.supplyAsync(() -> "Data")
    .thenRun(() -> System.out.println("Done!"));
```

- Chaining multiple callbacks:
```java
CompletableFuture.supplyAsync(() -> "Order-123")
    .thenApply(order -> order + "-processed")
    .thenApply(String::toUpperCase)
    .thenAccept(System.out::println);
```

- Async versions of callbacks:
```java
CompletableFuture.supplyAsync(() -> "Task")
    .thenApplyAsync(result -> result + " async")
    .thenAcceptAsync(System.out::println);
```

### Composition

- `thenCompose` — flatten nested futures (like `flatMap`):
```java
CompletableFuture<String> future = CompletableFuture.supplyAsync(() -> "User")
    .thenCompose(user -> fetchProfile(user));

private CompletableFuture<String> fetchProfile(String user) {
    return CompletableFuture.supplyAsync(() -> user + "'s profile");
}
```

- `thenCombine` — combine results of two independent futures:
```java
CompletableFuture<String> f1 = CompletableFuture.supplyAsync(() -> "Hello");
CompletableFuture<String> f2 = CompletableFuture.supplyAsync(() -> "World");
CompletableFuture<String> combined = f1.thenCombine(f2, (a, b) -> a + " " + b);
System.out.println(combined.get()); // Hello World
```

- Combine with BiFunction:
```java
CompletableFuture<Integer> prices = CompletableFuture.supplyAsync(() -> 100);
CompletableFuture<Integer> taxes = CompletableFuture.supplyAsync(() -> 20);
prices.thenCombine(taxes, (price, tax) -> price + tax)
      .thenAccept(total -> System.out.println("Total: " + total));
```

### Error Handling

- `exceptionally` — recover with a fallback value:
```java
CompletableFuture<Integer> future = CompletableFuture.supplyAsync(() -> {
    if (true) throw new RuntimeException("Failed");
    return 1;
}).exceptionally(ex -> {
    System.out.println("Error: " + ex.getMessage());
    return 0;
});
System.out.println(future.get()); // 0
```

- `handle` — handle both result and error (BiFunction):
```java
CompletableFuture<Integer> future = CompletableFuture.supplyAsync(() -> {
    return Integer.parseInt("not-a-number");
}).handle((result, ex) -> {
    if (ex != null) return -1;
    return result;
});
```

- `whenComplete` — side-effect on completion (BiConsumer):
```java
CompletableFuture.supplyAsync(() -> "Data")
    .whenComplete((result, ex) -> {
        if (ex == null) System.out.println("Success: " + result);
        else System.out.println("Failed: " + ex.getMessage());
    });
```

- Exception propagation in chains:
```java
CompletableFuture.supplyAsync(() -> "42")
    .thenApply(Integer::parseInt)
    .thenApply(n -> n / 0) // ArithmeticException
    .exceptionally(ex -> {
        System.out.println("Caught: " + ex.getClass().getSimpleName());
        return 0;
    });
```

### Combining Multiple Futures

- `allOf` — wait for all futures to complete:
```java
CompletableFuture<String> f1 = CompletableFuture.supplyAsync(() -> "A");
CompletableFuture<String> f2 = CompletableFuture.supplyAsync(() -> "B");
CompletableFuture<String> f3 = CompletableFuture.supplyAsync(() -> "C");

CompletableFuture<Void> all = CompletableFuture.allOf(f1, f2, f3);
all.thenRun(() -> System.out.println("All completed"));

// Collect results:
CompletableFuture<List<String>> allResults = all.thenApply(v ->
    Stream.of(f1, f2, f3).map(CompletableFuture::join).collect(Collectors.toList())
);
```

- `anyOf` — wait for any one future to complete:
```java
CompletableFuture<String> f1 = CompletableFuture.supplyAsync(() -> {
    sleep(100); return "Slow";
});
CompletableFuture<String> f2 = CompletableFuture.supplyAsync(() -> "Fast");

CompletableFuture<Object> any = CompletableFuture.anyOf(f1, f2);
System.out.println(any.get()); // Fast (whichever finishes first)
```

### Timeouts and Completion Overrides

- `orTimeout` — complete exceptionally if not done within duration (Java 9):
```java
CompletableFuture<String> future = CompletableFuture.supplyAsync(() -> {
    sleep(5000);
    return "Late";
}).orTimeout(1, TimeUnit.SECONDS);
// Throws TimeoutException after 1 second
future.get();
```

- `completeOnTimeout` — supply default value on timeout (Java 9):
```java
CompletableFuture<String> future = CompletableFuture.supplyAsync(() -> {
    sleep(5000);
    return "Late";
}).completeOnTimeout("Default", 1, TimeUnit.SECONDS);
System.out.println(future.get()); // Default
```

### Custom Executor

- Always prefer a custom executor for production:
```java
ExecutorService executor = Executors.newFixedThreadPool(10,
    new ThreadFactoryBuilder().setNameFormat("async-pool-%d").build());

CompletableFuture.supplyAsync(() -> fetchData(), executor)
    .thenApplyAsync(data -> transform(data), executor)
    .thenAcceptAsync(result -> save(result), executor);
```

- Using ForkJoinPool with specific parallelism:
```java
ForkJoinPool pool = new ForkJoinPool(4);
CompletableFuture<String> future = CompletableFuture.supplyAsync(() -> {
    return "Custom pool";
}, pool);
```

### CompletableFuture vs Future

- `Future` is read-only — you cannot complete it manually.
- `CompletableFuture` supports manual completion via `complete()` and `completeExceptionally()`.
- `Future.get()` blocks indefinitely; `CompletableFuture` supports non-blocking callbacks.
- `Future` has no composition — you cannot chain or combine multiple futures.
- `Future` has no error recovery — exceptions from `get()` are checked.
- `CompletableFuture` provides `orTimeout()` and `completeOnTimeout()` for bounded waits.
- `CompletableFuture` works well with modern async paradigms like reactive streams.

## Common Mistakes

- **Mistake: Calling `get()` or `join()` too early, blocking the calling thread**.
  - Why it looks correct: `get()` returns the result, and you need the result.
```java
// BAD: Blocks the calling thread immediately
CompletableFuture<String> future = CompletableFuture.supplyAsync(() -> fetch());
String result = future.get(); // blocked!
process(result);
```

- **Mistake: Not using a custom Executor**, causing all async tasks to share `ForkJoinPool.commonPool()`.
  - Why it looks correct: It works fine in development with low load.
```java
// BAD: Uses common pool shared across JVM
CompletableFuture.supplyAsync(this::heavyCompute);
// GOOD: Dedicated executor
CompletableFuture.supplyAsync(this::heavyCompute, executor);
```

- **Mistake: Swallowing exceptions by not attaching error handlers**.
  - Why it looks correct: The main chain "works" and exceptions are invisible.
```java
// BAD: Exception silently lost
CompletableFuture.supplyAsync(() -> { throw new RuntimeException("fail"); })
    .thenApply(x -> x.toString()); // never invoked, exception lost

// GOOD: Always add exceptionally or handle
CompletableFuture.supplyAsync(() -> { throw new RuntimeException("fail"); })
    .exceptionally(ex -> "recovered")
    .thenApply(x -> x.toString());
```

- **Mistake: Using `thenApply` for side-effects instead of `thenAccept`**.
  - Why it looks correct: `thenApply` accepts a Function and returns a value; returning null "works".
```java
// BAD: thenApply with side-effect, returns null
future.thenApply(result -> {
    saveToDb(result);
    return null;
});
// GOOD: thenAccept for consumers
future.thenAccept(result -> saveToDb(result));
```

- **Mistake: Forgetting that `thenApply`/`thenAccept` run on the completing thread**, not necessarily asynchronously.
  - Why it looks correct: The callback executes, so it seems async.
```java
// The callback may run in the supplier's thread
CompletableFuture.supplyAsync(() -> {
    // thread: pool-1-thread-1
    return "data";
}).thenApply(data -> {
    // thread: pool-1-thread-1 (same thread!)
    return data.toUpperCase();
});
```

- **Mistake: Ignoring cancellation and interruption**.
  - Why it looks correct: Cancellation "works" — the future stops being waited on.
```java
CompletableFuture<String> future = CompletableFuture.supplyAsync(() -> {
    while (!Thread.currentThread().isInterrupted()) {
        // heavy loop — never checks cancellation
    }
    return "done";
});
future.cancel(true); // may NOT actually stop the running task
```

## Real-World Scenarios

- **Scenario 1: Aggregating data from multiple microservices**.
```java
CompletableFuture<UserProfile> profileFuture =
    CompletableFuture.supplyAsync(() -> userService.getProfile(userId), executor);
CompletableFuture<OrderHistory> ordersFuture =
    CompletableFuture.supplyAsync(() -> orderService.getOrders(userId), executor);
CompletableFuture<Recommendations> recsFuture =
    CompletableFuture.supplyAsync(() -> recService.getRecommendations(userId), executor);

CompletableFuture.allOf(profileFuture, ordersFuture, recsFuture)
    .thenApply(v -> new Dashboard(
        profileFuture.join(),
        ordersFuture.join(),
        recsFuture.join()
    ))
    .thenAccept(dashboard -> render(dashboard));
```

- **Scenario 2: Parallel API calls with timeout and fallback**.
```java
CompletableFuture<Price> priceFuture = CompletableFuture
    .supplyAsync(() -> pricingApi.getPrice(productId), executor)
    .completeOnTimeout(defaultPrice, 2, TimeUnit.SECONDS)
    .exceptionally(ex -> fallbackPrice);
```

- **Scenario 3: Chaining dependent async operations (e.g., order processing)**.
```java
CompletableFuture.supplyAsync(() -> paymentService.charge(order), executor)
    .thenCompose(charge -> inventoryService.reserve(order.items()))
    .thenCompose(inventory -> shippingService.schedule(order))
    .thenAccept(shipment -> notificationService.notifyUser(order.userId()))
    .exceptionally(ex -> {
        log.error("Order failed", ex);
        rollback(order);
        return null;
    });
```

- **Scenario 4: Caching with async refresh**.
```java
private volatile CompletableFuture<Data> cache;

public CompletableFuture<Data> getData() {
    CompletableFuture<Data> result = cache;
    if (result == null || result.isCompletedExceptionally()) {
        synchronized (this) {
            if (cache == null || cache.isCompletedExceptionally()) {
                cache = CompletableFuture.supplyAsync(this::loadData, executor);
            }
            result = cache;
        }
    }
    return result;
}
```

- **Scenario 5: First-success pattern with `anyOf`**.
```java
List<CompletableFuture<Location>> geoFutures = providers.stream()
    .map(provider -> CompletableFuture
        .supplyAsync(() -> provider.geocode(address), executor)
        .orTimeout(500, TimeUnit.MILLISECONDS)
        .exceptionally(ex -> null))
    .collect(Collectors.toList());

CompletableFuture.anyOf(geoFutures.toArray(new CompletableFuture[0]))
    .thenApply(result -> (Location) result)
    .thenAccept(location -> map.show(location));
```

## Scenario-Based Questions

- **Question: You need to call three external APIs in parallel, each taking 2–5 seconds, then merge results. How?**
```java
CompletableFuture<A> f1 = CompletableFuture.supplyAsync(() -> api1.call(), executor);
CompletableFuture<B> f2 = CompletableFuture.supplyAsync(() -> api2.call(), executor);
CompletableFuture<C> f3 = CompletableFuture.supplyAsync(() -> api3.call(), executor);

CompletableFuture.allOf(f1, f2, f3).thenApply(v ->
    new MergedResult(f1.join(), f2.join(), f3.join())
);
```

- **Question: One service is slow and you want a default value after 2 seconds.**
```java
CompletableFuture.supplyAsync(() -> slowService.call(), executor)
    .completeOnTimeout("default", 2, TimeUnit.SECONDS);
```

- **Question: You need to call ServiceB with the result of ServiceA, both async.**
```java
CompletableFuture.supplyAsync(() -> serviceA.call(), executor)
    .thenCompose(resultA -> CompletableFuture.supplyAsync(() -> serviceB.call(resultA), executor));
```

- **Question: The user cancels an operation. How do you stop the async task?**
```java
CompletableFuture<String> future = CompletableFuture.supplyAsync(() -> {
    while (!Thread.currentThread().isInterrupted()) {
        // check interruption cooperatively
    }
    throw new CancellationException();
}, executor);
future.cancel(true);
```

- **Question: You want to log failures but still propagate the exception.**
```java
future.whenComplete((result, ex) -> {
    if (ex != null) log.error("Operation failed", ex);
});
```

- **Question: You need to poll an external service every 100ms until a result is ready, with a maximum of 10 retries. How do you implement this with CompletableFuture?**
```java
CompletableFuture<Result> pollWithRetry(int maxRetries) {
    CompletableFuture<Result> future = CompletableFuture.supplyAsync(() -> poll(), executor);
    for (int i = 0; i < maxRetries; i++) {
        future = future.thenCompose(result -> {
            if (result.isReady()) return CompletableFuture.completedFuture(result);
            return CompletableFuture.supplyAsync(() -> {
                sleep(100);
                return poll();
            }, executor);
        });
    }
    return future;
}
```

- **Question: You have a list of 1000 CompletableFuture tasks. Using `allOf` with 1000 futures creates a single callback that fires only when all complete. How do you process results incrementally as each completes instead of waiting for all?**

  - `allOf` waits for all futures — you cannot get partial results. Use a callback per future instead:
  ```java
  List<CompletableFuture<Result>> futures = tasks.stream()
      .map(task -> CompletableFuture.supplyAsync(() -> process(task), executor))
      .collect(Collectors.toList());

  futures.forEach(f -> f.thenAccept(result -> resultsQueue.add(result)));
  ```
  - Each future's completion triggers independent processing. The main thread can wait on `allOf` for overall completion, but individual results are consumed as they arrive.
  - For backpressure, use a blocking queue with bounded capacity so slow consumers force producers to wait.

- **Question: You call serviceA, serviceB, and serviceC in parallel. If serviceA fails, you want to cancel serviceB and serviceC immediately. How do you implement cancellation propagation?**

  - `CompletableFuture` does not have built-in cancellation propagation. You must implement it manually:
  ```java
  List<CompletableFuture<?>> all = List.of(f1, f2, f3);
  f1.exceptionally(ex -> {
      f2.cancel(true);
      f3.cancel(true);
      return null;
  });
  ```
  - However, `cancel(true)` only sets the `CancellationException` result — it does not stop the running thread unless the task checks `Thread.isInterrupted()`.
  - For true cancellation, use a shared `AtomicBoolean cancelled` flag that all tasks check periodically, or use the `CompletableFuture` timeout approach with `orTimeout()` and `completeOnTimeout()`.

- **Question: You chain `thenApply` -> `thenApply` -> `thenAccept`. The second `thenApply` throws an exception. Where does the exception propagate — to `thenAccept` or to a downstream `exceptionally`?**

  - The exception propagates through the chain, skipping all subsequent stages that depend on the result. `thenAccept` is never invoked because it requires a successful result from the previous stage.
  - An `exceptionally` attached after the chain catches the exception from any prior stage. An `exceptionally` placed between stages catches exceptions only from the stages above it.
  - `handle` captures both result and exception at its position in the chain, replacing the exception with a fallback value that downstream stages can process.

- **Question: You have a microservice that receives 1000 requests per second, each triggering a CompletableFuture chain. After running for 30 minutes, response times degrade from 50ms to 5 seconds. What is the most likely cause and how do you fix it?**

  - The most likely cause is thread pool starvation: all async tasks are running on `ForkJoinPool.commonPool()`, which has `Runtime.availableProcessors() - 1` threads. For a typical server with 8 cores, the common pool has 7 threads. 1000 requests/second with average task duration of 50ms means 50 concurrent tasks on average — and the 7-thread pool is saturated.
  - The fix: use a dedicated `ExecutorService` with a bounded work queue and a thread pool sized for the expected concurrency: `new ThreadPoolExecutor(50, 100, 60, TimeUnit.SECONDS, new LinkedBlockingQueue<>(1000))`.
  - Also add monitoring: queue depth, active thread count, rejected task count. When the queue fills, the executor strategy determines behavior (`CallerRunsPolicy` throttles the caller, `AbortPolicy` throws, `DiscardPolicy` silently drops).

---

## Interview Questions

- What is the difference between `Future` and `CompletableFuture`?
- What is `CompletionStage` and why does `CompletableFuture` implement it?
- Explain the difference between `thenApply`, `thenAccept`, and `thenRun`.
- What is the difference between `thenCompose` and `thenCombine`?
- How does `exceptionally` differ from `handle` and `whenComplete`?
- When would you use `allOf` vs `anyOf`?
- What happens if an exception occurs in the middle of a callback chain?
- How do you provide a default value when a future times out?
- Why should you avoid `ForkJoinPool.commonPool()` in production?
- What is the threading behavior of `thenApply` vs `thenApplyAsync`?
- How do you implement retry logic with `CompletableFuture`?
- What is the difference between `complete()` and `obtrudeValue()`?
- How do you cancel a `CompletableFuture`? Does it stop the running thread?
- Explain how `thenCompose` prevents nested `CompletableFuture<CompletableFuture<T>>`.
- How do you collect results from a list of `CompletableFuture`?

- **What is the difference between `thenApply` and `thenCompose`?**
  - `thenApply` transforms the result of a `CompletableFuture<T>` using a `Function<T, R>` — the function returns a plain value, and the result is `CompletableFuture<R>`.
  - `thenCompose` transforms the result using a `Function<T, CompletionStage<R>>` — the function returns a `CompletionStage`, and the result is `CompletableFuture<R>` without nesting (`CompletableFuture<CompletableFuture<R>>`).
  - Use `thenApply` for simple transformations (parsing, mapping) and `thenCompose` for chaining another async operation (flatMap).

- **What is the difference between `complete()` and `obtrudeValue()`?**
  - `complete(T value)` sets the result only if the future is not already completed. Returns `true` if the value was set, `false` if the future was already completed.
  - `obtrudeValue(T value)` forcibly sets the result even if the future was already completed — it overwrites any existing result or exception.
  - `obtrudeValue` is intended for exceptional cases like caching systems where a stale result must be replaced. It should not be used in normal control flow.

- **How does `completeExceptionally` interact with callback chains?**
  - When a `CompletableFuture` is completed exceptionally, all downstream `thenApply`, `thenAccept`, and `thenCompose` callbacks are skipped — the exception propagates to the first `exceptionally`, `handle`, or `whenComplete` handler in the chain.
  - Multiple `exceptionally` handlers can be chained: each one can catch the exception and provide a recovery value, which then flows into the next normal callback.
  - `whenComplete` does not consume the exception — it observes the result or exception for side effects (logging, metrics) and then propagates the same result or exception downstream.

- **What is the `CompletableFuture` timeout behavior in Java 9+?**
  - `orTimeout(long timeout, TimeUnit unit)` completes the future exceptionally with a `TimeoutException` if not done before the timeout. Returns the same `CompletableFuture` for chaining.
  - `completeOnTimeout(T value, long timeout, TimeUnit unit)` supplies a default value on timeout instead of throwing. Returns the same `CompletableFuture`.
  - Both methods schedule a delayed task that competes with the normal completion — the first to complete wins. This is safe for one-shot operations but not for repeated timeouts.

- **What is the difference between `get()` and `join()` on CompletableFuture?**
  - `get()` throws `InterruptedException` (checked) and `ExecutionException`. `join()` throws `CompletionException` (unchecked) and `CancellationException`.
  - `join()` is preferred in lambda chains and stream pipelines because it does not require checked exception handling.
  - Both block the calling thread until completion. For non-blocking consumption, use `thenAccept` or `whenComplete`.

## Developer Recommendations

- Always provide a custom `Executor` with a meaningful thread pool name.
- Attach `exceptionally` or `handle` at the end of every chain even if you "know" it won't fail.
- Prefer `thenCompose` over nesting `future.get()` inside another `supplyAsync`.
- Use `allOf` + `join` pattern for fan-out/fan-in instead of collecting synchronously.
- Set timeouts on every async call to prevent thread starvation in the pool.
- Be mindful that `thenApply` runs on the thread that completes the stage, which may be unexpected.
- Use `thenApplyAsync`/`thenAcceptAsync` with the same executor if you want deterministic threading.
- Never call `get()` inside a callback — it blocks the common pool thread.
- Use `whenComplete` for logging/monitoring side-effects that should not change the result.
- Test your async chains with failure injection to verify error handling paths.
- Avoid shared mutable state across async stages; prefer immutable transformations.
- Consider resilience frameworks (Resilience4j, Hystrix) for circuit-breaking and retries at scale.
- Use `ThreadFactoryBuilder` from Guava or similar to name your threads for debugging.
- Monitor your async pool metrics (queue depth, active threads, rejected tasks).
- In production stories, teams have seen `ForkJoinPool.commonPool()` starvation cause cascading failures across the entire JVM because all modules unknowingly shared the same pool.
