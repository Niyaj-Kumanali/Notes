# Virtual Threads

## Overview

- **Definition** — Virtual threads are lightweight, JVM-managed threads that enable high-concurrency server applications by multiplexing many virtual threads onto a small number of platform (OS) threads.
- **Why It Exists** — Platform threads are expensive (1:1 mapping to kernel threads) and limit concurrency to the order of thousands. Virtual threads make the "one thread per request" model viable for hundreds of thousands of concurrent requests without the complexity of reactive programming.
- **Historical Context** — Part of Project Loom, previewed in JDK 19 and JDK 20, finalized in JDK 21.
- **Key Concepts** — **Platform thread** is an OS-managed thread with 1:1 kernel mapping, expensive and limited; **virtual thread** is JVM-managed with many-to-one mapping to carrier threads, cheap and plentiful; **mounting** attaches a virtual thread to a carrier platform thread during execution; **unmounting** detaches it transparently on blocking I/O; **one-thread-per-request** becomes scalable; **avoid synchronized** because it causes pinning (prevents unmounting); **avoid ThreadLocal** with large pools as it defeats memory advantages; **CPU-bound** workloads are not helped; **structured concurrency** via `StructuredTaskScope` groups related tasks with cancellation propagation.

## Core Concepts

- Creating a virtual thread:

  ```java
  Thread vThread = Thread.ofVirtual().start(() -> {
    System.out.println("Hello from virtual thread");
  });
  ```

- Using `Executors.newVirtualThreadPerTaskExecutor()` for a per-task executor:

  ```java
  try (var executor = Executors.newVirtualThreadPerTaskExecutor()) {
    executor.submit(() -> processRequest());
  }
  ```

- Mounting and unmounting: when a virtual thread executes, it is mounted on a carrier platform thread. When it blocks on I/O or a blocking operation, the JVM unmounts it, allowing the carrier thread to run other virtual threads. The mounting/unmounting is transparent.
- Platform threads are expensive (stack size ~1MB, context-switch overhead). Virtual threads have a small stack (initially ~few KB), can be created in the millions, and have much lower creation and context-switch cost.
- Use cases: IO-bound workloads — web servers, REST calls, database queries, file reads. The one-thread-per-request model that was previously limited by OS thread count now scales.
- Limitations:
  - **Synchronized blocks/methods** cause pinning: when a virtual thread enters `synchronized` and blocks, the carrier thread cannot unmount it, reducing throughput. Use `ReentrantLock` instead.
  - **ThreadLocal** with many virtual threads can retain large memory per thread. Avoid pooling of virtual threads (they are cheap enough to create per-task) and avoid ThreadLocal-heavy patterns.
  - **CPU-bound workloads** are not helped — virtual threads cannot run in parallel on multiple cores beyond the carrier pool size. Use platform threads or parallel streams for CPU-intensive work.
- Structured concurrency via `StructuredTaskScope`:

  ```java
  try (var scope = new StructuredTaskScope.ShutdownOnFailure()) {
    Future<String> user = scope.fork(() -> fetchUser(id));
    Future<String> order = scope.fork(() -> fetchOrder(id));
    scope.join().throwIfFailed();
    return new Response(user.resultNow(), order.resultNow());
  }
  ```

- `StructuredTaskScope.ShutdownOnFailure` cancels all subtasks if any one fails (fail-fast). `StructuredTaskScope.ShutdownOnSuccess` cancels remaining subtasks once any succeeds.
- Structured concurrency enforces that subtasks complete before the scope is closed, preventing task leaks and simplifying error handling.
- Comparison with reactive programming: virtual threads allow writing synchronous, blocking-style code that is easy to read and debug, while achieving similar concurrency to reactive frameworks (WebFlux, RxJava). Reactive programming still has advantages for backpressure and streaming scenarios.

## Common Mistakes

- **Using synchronized with virtual threads**
  - When a virtual thread blocks inside a `synchronized` block, it is pinned to its carrier thread — the carrier thread cannot be reused.
  - **Why it looks correct:** Synchronized is the standard Java locking mechanism and works fine with platform threads.
  - The fix: replace `synchronized` with `ReentrantLock` or other `java.util.concurrent` locks that do not cause pinning.

- **Creating a ThreadLocal-heavy application with virtual thread pools**
  - ThreadLocal values are retained for the lifetime of the thread. With a per-task executor creating millions of virtual threads, each ThreadLocal allocation multiplies memory.
  - **Why it looks correct:** ThreadLocal is widely used with platform thread pools where the thread count is bounded.
  - The fix: avoid ThreadLocal in virtual-thread-based code. Pass context explicitly as parameters or use `ScopedValue` (incubator in JDK 21).

- **Using virtual threads for CPU-bound computation**
  - Virtual threads do not add parallelism beyond the available carrier threads (typically the number of platform threads).
  - **Why it looks correct:** Virtual threads are threads, and threads are used for parallelism.
  - The fix: use platform threads, parallel streams, or `ForkJoinPool` for CPU-intensive work.

- **Ignoring the pinning issue with native methods and blocking calls**
  - If a virtual thread calls a native method or a blocking operation that the JVM cannot intercept, it may be pinned.
  - **Why it looks correct:** Most blocking I/O operations are intercepted, but not all (e.g., JNI calls, some file I/O).
  - The fix: profile for pinning using JDK Flight Recorder or thread dumps, and refactor pinned operations.

- **Mixing virtual threads with thread-pool-based blocking patterns**
  - Using `Future.get()` inside a virtual thread is fine, but using a `CompletableFuture` chain that blocks a common ForkJoinPool can cause starvation.
  - **Why it looks correct:** Both use `Future` and look similar.
  - The fix: prefer `StructuredTaskScope` for coordinating virtual thread subtasks.

## Real-World Scenarios

### One-Thread-Per-Request Web Server

- A REST endpoint that calls three downstream services:

  ```java
  public Response handleRequest(Request req) throws Exception {
    try (var scope = new StructuredTaskScope.ShutdownOnFailure()) {
      Future<Data> a = scope.fork(() -> serviceA.call(req));
      Future<Data> b = scope.fork(() -> serviceB.call(req));
      Future<Data> c = scope.fork(() -> serviceC.call(req));
      scope.join().throwIfFailed();
      return new Response(a.resultNow(), b.resultNow(), c.resultNow());
    }
  }
  ```

- Each downstream call blocks inside a virtual thread, but the carrier thread is reused for other virtual threads.

### Batch File Processing

- Process thousands of independent files concurrently:

  ```java
  try (var executor = Executors.newVirtualThreadPerTaskExecutor()) {
    List<Path> files = listFiles();
    List<Future<Result>> futures = files.stream()
      .map(file -> executor.submit(() -> processFile(file)))
      .toList();
    // collect results
  }
  ```

### Database Query Fan-Out

- Execute multiple database queries that block on I/O:

  ```java
  public Dashboard loadDashboard(String userId) throws Exception {
    try (var scope = new StructuredTaskScope.ShutdownOnFailure()) {
      Future<UserProfile> profile = scope.fork(() -> userRepo.findById(userId));
      Future<List<Order>> orders = scope.fork(() -> orderRepo.findByUserId(userId));
      Future<Stats> stats = scope.fork(() -> statsRepo.compute(userId));
      scope.join().throwIfFailed();
      return new Dashboard(profile.resultNow(), orders.resultNow(), stats.resultNow());
    }
  }
  ```

## Scenario-Based Questions

**Q: Your web application currently uses a thread pool of 200 platform threads to handle HTTP requests. You want to migrate to virtual threads. What changes are required in the web server configuration?**

- Replace the platform thread executor with `Executors.newVirtualThreadPerTaskExecutor()`. The web server (Tomcat, Jetty, Undertow) must be configured to use this executor for request handling. Tomcat 10.1+ and Jetty 12+ support virtual threads natively. No code changes are needed inside request handlers if they already use `ReentrantLock` instead of `synchronized`.
- **Interview follow-up:** If a library your application depends on uses synchronized internally, what options do you have?

**Q: You are building a microservice that calls five external APIs in parallel. How would you implement this with virtual threads and structured concurrency?**

- Use `StructuredTaskScope.ShutdownOnFailure` to fork all five calls. If any call fails, the scope cancels the remaining ones. This provides clean error handling and ensures no background tasks leak.

  ```java
  try (var scope = new StructuredTaskScope.ShutdownOnFailure()) {
    Future<A> a = scope.fork(() -> apiA.call());
    Future<B> b = scope.fork(() -> apiB.call());
    Future<C> c = scope.fork(() -> apiC.call());
    Future<D> d = scope.fork(() -> apiD.call());
    Future<E> e = scope.fork(() -> apiE.call());
    scope.join().throwIfFailed();
    return combine(a.resultNow(), b.resultNow(), c.resultNow(), d.resultNow(), e.resultNow());
  }
  ```

- **Interview follow-up:** How would you change this if you wanted to return the first successful result and ignore the others?

**Q: Your application has a synchronized block inside a request handler that uses virtual threads. What performance issue can you expect?**

- The virtual thread will be pinned to its carrier thread while inside the `synchronized` block. This reduces concurrency because the carrier thread cannot be reused for other virtual threads during the blocking operation. Replace `synchronized` with `ReentrantLock` to avoid pinning.
- **Interview follow-up:** How would you detect pinning in a running application?

**Q: You want to use virtual threads for file I/O operations. Are virtual threads beneficial for disk I/O compared to network I/O?**

- Virtual threads help with any blocking I/O where the thread spends most of its time waiting. File I/O on local disks typically has lower latency than network I/O, but for many concurrent file operations (e.g., batch processing thousands of files), virtual threads still improve throughput by reducing OS thread overhead.
- **Interview follow-up:** Does file I/O on Windows cause pinning issues with virtual threads?

**Q: You are migrating a legacy application from a fixed thread pool to virtual threads. The application uses ThreadLocal for request-scoped data. What issues might arise?**

- With virtual threads, each request creates a new virtual thread, and ThreadLocal memory scales with the number of threads. Millions of virtual threads each holding a ThreadLocal can exhaust heap memory. Use `ScopedValue` or pass context as method parameters instead.
- **Interview follow-up:** How does `ScopedValue` differ from `ThreadLocal` in the context of virtual threads?

**Q: How would you handle timeout for a virtual thread that is blocked on a network call?**

- Use `StructuredTaskScope` with `scope.joinUntil(Instant)` to set a deadline for all subtasks. Alternatively, use `Future.get(timeout, unit)` inside the forked task. Virtual threads can be interrupted during blocking operations just like platform threads.
- **Interview follow-up:** What happens to a virtual thread if the `Future.get()` timeout expires?

**Q: You have a CPU-intensive computation that you currently run in a thread pool. Would virtual threads improve performance?**

- No. Virtual threads do not add parallelism. They are multiplexed onto a limited number of carrier threads (typically the number of CPU cores). CPU-bound workloads benefit from parallelism via platform threads or `ForkJoinPool`, not from virtual threads.
- **Interview follow-up:** What is the recommended approach for mixing CPU-bound and I/O-bound tasks in the same application?

**Q: A team is using `Executors.newVirtualThreadPerTaskExecutor()` inside a loop that submits 100,000 tasks. How does this affect resource usage compared to a fixed thread pool?**

- Each task runs on a new virtual thread, which is cheap (~few KB stack). The executor does not require a pool — virtual threads are created and disposed per task. A fixed platform thread pool would limit concurrency to the pool size but consume more memory per thread. Virtual threads scale to 100,000 concurrent tasks easily.
- **Interview follow-up:** Should you ever pool virtual threads?

**Q: You need to call a third-party library that uses `Object.wait()` and `notify()` internally. Does this work with virtual threads?**

- `Object.wait()` causes pinning on virtual threads because it uses `synchronized` internally. The virtual thread will be pinned to its carrier thread during the wait. If possible, replace the library or use platform threads for that code.
- **Interview follow-up:** Does `LockSupport.park()` cause pinning?

**Q: How would you implement a retry mechanism for a virtual thread that calls an unreliable external service?**

- Use a `StructuredTaskScope` inside a loop with a counter. Each attempt is a separate fork. If the scope fails, retry up to N times. The structured scope ensures that failed attempts are properly cleaned up before retrying.

  ```java
  for (int i = 0; i < 3; i++) {
    try (var scope = new StructuredTaskScope.ShutdownOnFailure()) {
      Future<Result> f = scope.fork(() -> callService());
      scope.joinUntil(Instant.now().plusSeconds(5));
      return f.resultNow();
    } catch (Exception e) { /* retry */ }
  }
  throw new RuntimeException("Failed after 3 retries");
  ```

- **Interview follow-up:** How would you add exponential backoff between retries?

**Q: You are using virtual threads with a database connection pool. Should the pool size be adjusted?**

- Yes. With platform threads, the pool size typically matches the thread pool size (e.g., 200 connections for 200 threads). With virtual threads, you can have thousands of concurrent requests, so the connection pool may need to be larger. However, the database still has a finite connection limit, so consider connection pooling carefully.
- **Interview follow-up:** How does the `maxPoolSize` interact with virtual thread scalability?

## Interview Questions

- **What is the difference between a platform thread and a virtual thread?**
  - A platform thread is a wrapper around an OS thread with a 1:1 mapping, a large stack (~1 MB), and high creation/context-switch cost. A virtual thread is a JVM-managed thread with a many-to-one mapping to carrier platform threads, a tiny stack, and cheap creation/blocking.

- **What is pinning and why is it a problem for virtual threads?**
  - Pinning occurs when a virtual thread cannot be unmounted from its carrier thread, typically due to `synchronized` blocks or native method calls. This reduces concurrency because the carrier thread cannot be reused for other virtual threads while pinned.

- **How does structured concurrency differ from unstructured concurrency with ExecutorService?**
  - Structured concurrency ensures that the lifetime of subtasks is nested within the scope of the parent task. Subtasks are started, joined, and their resources cleaned up before the scope closes. This prevents task leaks, simplifies error propagation, and makes the code structure match the task structure. Unstructured concurrency with `ExecutorService` does not enforce this nesting.

- **When would you choose reactive programming over virtual threads?**
  - Reactive programming is still preferable when you need fine-grained backpressure, streaming of large datasets, or when integrating with reactive libraries that provide operators for complex event processing. Virtual threads are better for straightforward request-response services with blocking I/O.

- **Can a virtual thread be interrupted while unmounted?**
  - Yes. If a virtual thread is unmounted (waiting for I/O), it can still be interrupted via `Thread.interrupt()`. The interrupt will be delivered when the virtual thread resumes execution.

- **What is the default carrier thread pool size for virtual threads?**
  - The default carrier pool uses a `ForkJoinPool` with the number of threads equal to the number of available processors. This can be tuned via the `jdk.virtualThreadScheduler.parallelism` system property.

- **Can you create a daemon virtual thread?**
  - Yes. Virtual threads inherit the daemon status from the builder. `Thread.ofVirtual().daemon(true).start(runnable)` creates a daemon virtual thread that does not prevent JVM shutdown.

- **What happens to a virtual thread's stack when it is unmounted?**
  - When unmounted, the virtual thread's stack is moved from the carrier thread's stack to the heap as a small stack chunk. This allows the carrier thread to be reused and is the key to virtual thread scalability.

- **Can virtual threads use thread priorities?**
  - Virtual threads do not support thread priorities. The `setPriority()` method has no effect on virtual threads.

- **How does `Thread.onSpinWait()` behave in virtual threads?**
  - `Thread.onSpinWait()` hints that the thread is in a spin loop. In virtual threads, this prevents unmounting because the thread is expected to resume quickly. Avoid spin-waiting in virtual threads.

- **Can virtual threads be used with `CompletableFuture`?**
  - Yes. Virtual threads can create and complete `CompletableFuture` instances. However, avoid blocking on `CompletableFuture.get()` inside a virtual thread if the future is completed by another virtual thread in the same carrier pool, as this can cause starvation.

- **What is the memory footprint of an idle virtual thread?**
  - An idle (not running) virtual thread occupies approximately a few hundred bytes. Its stack starts very small (a few KB) and grows as needed. This is orders of magnitude less than a platform thread (~1 MB stack).

- **How do you get a thread dump of virtual threads?**
  - Use `jcmd <pid> Thread.dump_to_file -format=json <file>` which includes virtual threads. JDK Flight Recorder also records virtual thread events. Traditional `jstack` does not show virtual threads.

- **Can virtual threads be used with `ForkJoinPool`?**
  - Virtual threads can submit tasks to a `ForkJoinPool`, but the pool's worker threads are platform threads. The virtual threads will be unmounted while waiting for the fork-join task to complete.

- **What is the relationship between virtual threads and Java's `ThreadGroup`?**
  - Virtual threads belong to a `ThreadGroup` just like platform threads. By default, they are placed in a system thread group. You can specify a custom `ThreadGroup` using the thread builder.

- **Can virtual threads be renamed after creation?**
  - Yes. Virtual threads support `setName()` like platform threads. The name change is reflected in thread dumps and debugging tools.

- **How do you handle thread-local cleanup in a virtual-thread-per-task executor?**
  - Since virtual threads are created per task, thread-local cleanup is automatic when the virtual thread terminates. However, avoid relying on `ThreadLocal.remove()` in cleanup hooks because the thread may be reused in a pooled scenario.

- **What is the default stack size of a virtual thread?**
  - The default stack size is very small (a few KB), growing dynamically as needed. This contrasts with platform threads that have a fixed stack (typically 1 MB). The dynamic nature contributes to virtual thread scalability.

- **Can virtual threads be used with Java NIO selectors?**
  - Yes. Virtual threads can perform selector operations. The virtual thread will be unmounted while waiting for channel events, allowing the carrier thread to be reused.

- **How do you debug a virtual thread that is stuck or deadlocked?**
  - Use JDK Flight Recorder with virtual thread events enabled, or generate a thread dump via `jcmd <pid> Thread.dump_to_file -format=json`. The dump includes all virtual threads and their stack traces, showing which carrier thread they are mounted on.

## Developer Recommendations

- **Replace synchronized blocks with ReentrantLock in code that runs on virtual threads**
  - Synchronized causes pinning, reducing throughput. `ReentrantLock` does not pin and allows the virtual thread to unmount when blocked.
  - Audit your codebase for `synchronized` before migrating to virtual threads. Libraries that use synchronized internally may also need workarounds.

- **Use StructuredTaskScope for task coordination**
  - `StructuredTaskScope` enforces structured concurrency, ensuring subtasks complete before the scope closes and providing automatic cancellation on failure.
  - Use `ShutdownOnFailure` for fail-fast and `ShutdownOnSuccess` for first-result patterns. Avoid manual `CompletableFuture` orchestration for simple fan-out.

- **Avoid ThreadLocal in virtual-thread-based applications**
  - ThreadLocal memory scales with the number of threads, which can be huge with virtual threads.
  - Pass context explicitly as method parameters or use immutable context objects. For inherited context, consider `ScopedValue` (incubator in JDK 21).
  - **Production story:** A team migrated their REST API gateway from 50 platform threads per node to virtual threads, handling 50,000 concurrent requests per node with no code changes except replacing the executor and fixing two synchronized blocks. Latency at the 99th percentile dropped from 800ms to 120ms because blocking I/O no longer consumed OS threads.
