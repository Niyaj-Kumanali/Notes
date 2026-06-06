# Java Multithreading

---

## Overview

Multithreading is the ability of a CPU to execute multiple threads concurrently, where a thread is the smallest unit of execution within a process. Java provides built-in support for creating threads, managing their lifecycle, and coordinating their access to shared resources. Multithreading improves CPU utilization by keeping processors busy while threads wait for I/O operations, maintains application responsiveness by keeping the UI thread available while background work completes, increases throughput by processing multiple requests concurrently, and leverages shared memory within a process (cheaper than inter-process communication). A process has its own separate address space with heavy creation overhead via `fork/exec`, while threads within a process share memory, have lightweight creation, and cheaper context switching because they share memory mappings, file descriptors, and other OS resources. Java threads map to OS-level kernel threads in a 1:1 model on modern JVMs (HotSpot), meaning each `Thread` object corresponds to a native thread scheduled by the operating system.

---

## Creating Threads

Java provides four ways to create and execute threads. Extending the `Thread` class by overriding `run()` is simple but not recommended because Java's single-inheritance model prevents the class from extending any other class. Implementing `Runnable` is preferred as it separates the task (the `run()` method) from the execution mechanism (the `Thread`), and the `Runnable` can be passed to a thread pool or a `Thread` constructor. Lambda expressions (Java 8+) provide the most concise form for simple tasks, letting you write `new Thread(() -> System.out.println("Hello")).start()` without defining any separate class. For tasks that return a result or throw checked exceptions, use `Callable<T>` with an `ExecutorService` — the `submit()` method returns a `Future<T>` that provides the result via a blocking `get()` call.

```java
// Extend Thread class: Simple but not recommended (can't extend other classes)
class Worker extends Thread {
    @Override
    public void run() {
        System.out.println("Working in thread: " + getName());
    }
}
new Worker().start();

// Implement Runnable (preferred): Separates task from execution mechanism
class Task implements Runnable {
    @Override
    public void run() {
        System.out.println("Task in thread: " + Thread.currentThread().getName());
    }
}
new Thread(new Task()).start();

// Lambda (Java 8+): Most concise for simple tasks
new Thread(() -> System.out.println("Lambda thread")).start();

// Callable + ExecutorService: Returns a result
ExecutorService executor = Executors.newFixedThreadPool(4);
Future<Integer> future = executor.submit(() -> {
    Thread.sleep(1000);
    return 42;
});
Integer result = future.get();  // blocks until result ready
```

---

## Thread Lifecycle

```
          ┌──────────┐
          │   NEW    │  (after new Thread())
          └────┬─────┘
               │ start()
               ▼
          ┌──────────┐
          │ RUNNABLE │  (scheduled by OS)
          └────┬─────┘
               │
     ┌─────────┼─────────┐
     │         │         │
     ▼         ▼         ▼
┌────────┐ ┌────────┐ ┌──────────┐
│RUNNING │ │BLOCKED │ │ WAITING  │
└───┬────┘ │(sync)  │ │(wait/join)│
    │      └────────┘ └──────────┘
    │           │         │
    └───────────┴─────────┘
               │
               ▼
          ┌──────────┐
          │TERMINATED│
          └──────────┘
```

A Java thread transitions through six well-defined states in its lifecycle. `NEW` is the state immediately after `new Thread()` but before `start()` is called — the thread has been created but is not yet eligible for execution. `RUNNABLE` means the thread is eligible for CPU scheduling; it may be currently executing (`RUNNING`) or waiting for the OS scheduler to allocate a processor. `BLOCKED` means the thread is waiting to acquire a monitor lock to enter a `synchronized` block or method that another thread currently holds. `WAITING` means the thread is waiting indefinitely for another thread to perform a specific action, entered via `Object.wait()`, `Thread.join()`, or `LockSupport.park()`. `TIMED_WAITING` is waiting with a timeout via `Thread.sleep(time)`, `Object.wait(timeout)`, `Thread.join(timeout)`, or `LockSupport.parkNanos()`. `TERMINATED` means the thread's `run()` method has completed normally or thrown an uncaught exception.

---

## Thread Properties

Every thread should be given a meaningful name with `thread.setName("worker-pool-1")` for debugging — thread dumps show thread names, and `pool-1-thread-1` is useless for identifying which thread is stuck. Daemon threads are background threads that do not prevent the JVM from exiting — when only daemon threads remain alive, the JVM shuts down. Calling `thread.setDaemon(true)` must be done before `start()`. Thread priority (`Thread.setPriority(Thread.NORM_PRIORITY)` with values 1-10) is platform-dependent and most operating systems ignore it entirely, so relying on priority for correctness or performance is a mistake. Thread stacks default to approximately 1MB on 64-bit JVMs (configurable with `-Xss1m`), storing local variables and method call frames that are not shared between threads.

```java
thread.setName("worker-pool-1");
```

---

## Under the Hood

Modern JVMs (HotSpot) use a 1:1 threading model where each Java thread maps directly to a native kernel thread managed by the operating system. The OS thread scheduler decides which thread runs on which core, and Java does not guarantee any scheduling order or fairness. Each thread gets its own stack of approximately 1MB by default (configurable with `-Xss`), which stores local variables and method call frames and is not shared between threads. Context switching between threads costs approximately 1-10 microseconds per switch, including saving and restoring CPU registers, flushing the TLB (Translation Lookaside Buffer), and incurring cache misses when the thread resumes on a different core. Thread creation costs approximately 1-2 microseconds for the JVM call plus the OS thread creation, and with 1MB default stack size, 1024 threads consume 1GB of virtual memory just for thread stacks — this is the fundamental scalability limit of the thread-per-connection model.

---

## Thread Pools (ExecutorService)

Managing threads manually with `new Thread()` and `thread.start()` is error-prone and does not scale. `ExecutorService` provides a higher-level API that decouples task submission from thread management, reusing a fixed number of threads to avoid creation overhead and limit resource consumption. `Executors.newFixedThreadPool(n)` creates a pool with a fixed number of threads that never grows — ideal for stable workloads. `Executors.newCachedThreadPool()` creates threads on demand and recycles idle threads (60-second timeout), suitable for bursty short-lived tasks but dangerous for high load because it creates unbounded threads. `Executors.newSingleThreadExecutor()` guarantees serial execution of all submitted tasks on a single worker thread. `Executors.newScheduledThreadPool(n)` supports delayed and periodic task execution with `schedule()`, `scheduleAtFixedRate()`, and `scheduleWithFixedDelay()`. All executors support graceful shutdown via `shutdown()` (prevent new tasks, complete existing) followed by `awaitTermination(timeout)` with `shutdownNow()` as a forced fallback.

```java
// Types of thread pools
ExecutorService fixed = Executors.newFixedThreadPool(10);       // Fixed size
ExecutorService cached = Executors.newCachedThreadPool();       // Dynamic
ExecutorService single = Executors.newSingleThreadExecutor();   // Single worker
ScheduledExecutorService scheduled = Executors.newScheduledThreadPool(5);  // Scheduled

// Submit tasks
Future<String> future = executor.submit(() -> doWork());
executor.execute(() -> doWork());  // Fire-and-forget

// Graceful shutdown
executor.shutdown();
if (!executor.awaitTermination(5, TimeUnit.SECONDS)) {
    executor.shutdownNow();
}
```

---

## Common Mistakes

Creating threads in unbounded loops or with `newCachedThreadPool()` for every request causes thread leaks that lead to `OutOfMemoryError: unable to create new native thread` when the OS process hits its thread limit. Always use bounded thread pools sized to the available resources.

Not naming threads makes thread dump analysis nearly impossible — `pool-1-thread-1` gives no clue which component owns the stuck thread. Always set meaningful names via a custom `ThreadFactory` or `Executors.defaultThreadFactory()` combined with `setName()`.

Swallowing `InterruptedException` in an empty `catch` block loses the interrupt signal, making `shutdownNow()` and `future.cancel(true)` ineffective. Always restore the interrupt flag with `Thread.currentThread().interrupt()`.

Not shutting down executor services causes thread leaks that prevent the JVM from ever exiting. Always call `executor.shutdown()` in a `@PreDestroy` method or `finally` block.

---

## Real-World Scenarios

### Scenario 1: Async Image Processing Pipeline

A photo sharing application uploads images that need multiple processing steps: resizing to three sizes (thumbnail, medium, full), metadata extraction (EXIF data like camera model and GPS), and content moderation (NSFW detection via ML model). Each step is either CPU-intensive (resizing, moderation) or I/O-bound (metadata extraction from disk). The pipeline must process 100 uploads per second without blocking the HTTP request thread.

```java
public class ImageProcessingPipeline {
    private final ExecutorService cpuPool = Executors.newFixedThreadPool(
        Runtime.getRuntime().availableProcessors());
    private final ExecutorService ioPool = Executors.newCachedThreadPool();

    public CompletableFuture<ProcessedImage> process(UploadedImage image) {
        return CompletableFuture.supplyAsync(() -> extractMetadata(image), ioPool)
            .thenApplyAsync(this::resizeThumbnail, cpuPool)
            .thenApplyAsync(this::resizeMedium, cpuPool)
            .thenApplyAsync(this::resizeFull, cpuPool)
            .thenCombineAsync(
                CompletableFuture.supplyAsync(() -> moderateContent(image), cpuPool),
                (meta, isSafe) -> { if (!isSafe) throw new UnsafeContentException(); return meta; }
            );
    }
}
```

Separating CPU-bound operations (resizing, moderation) into their own pool sized to `availableProcessors()` and I/O-bound operations (metadata extraction) into a `cachedThreadPool()` prevents I/O threads from starving CPU work. The entire pipeline is non-blocking — the HTTP request thread submits the work via `CompletableFuture.supplyAsync()` and returns immediately, freeing the thread to handle the next request. Each processing step starts asynchronously when the previous step completes, without any thread blocking on `get()` or `join()`.

### Scenario 2: WebSocket Connection Manager

A collaborative editing application maintains 50,000 concurrent WebSocket connections. Each connection receives document edits from a user and must broadcast changes to all other collaborators viewing the same document. The system must handle burst edits without losing messages or creating unbounded thread counts.

```java
public class WebSocketManager {
    private final ConcurrentHashMap<String, CopyOnWriteArrayList<WebSocketSession>> rooms = new ConcurrentHashMap<>();
    private final ExecutorService broadcastPool = Executors.newFixedThreadPool(16);

    public void onEdit(String roomId, Edit edit, WebSocketSession sender) {
        CopyOnWriteArrayList<WebSocketSession> members = rooms.get(roomId);
        if (members == null) return;

        broadcastPool.submit(() -> {
            for (WebSocketSession member : members) {
                if (!member.getId().equals(sender.getId())) {
                    try {
                        synchronized (member) {
                            member.send(new TextMessage(edit.toJson()));
                        }
                    } catch (IOException e) {
                        log.error("Failed to send to {}", member.getId());
                    }
                }
            }
        });
    }
}
```

`CopyOnWriteArrayList` provides thread-safe iteration without locks — reads (the broadcast loop) are never blocked by concurrent modifications (users joining or leaving rooms). The `broadcastPool` with 16 bounded threads prevents a flood of edits from creating unlimited threads, while the `synchronized(member)` block ensures ordered delivery per connection — edits to the same user are sent one at a time in FIFO order without interleaving. This design maintains 50,000 connections using only 16 broadcast threads because threads are used for sending and then returned to the pool, not held waiting for each connection.

### Scenario 3: Graceful Shutdown in a Payment Gateway

A payment gateway processes credit card charges. During a deployment roll, the application receives a shutdown signal but must finish processing the 50 in-flight transactions before terminating. If a transaction does not complete within 30 seconds, it must be marked as failed so another instance can retry it. The shutdown must be graceful — no transactions should be lost in an indeterminate state.

```java
public class PaymentGateway {
    private final ExecutorService executor = Executors.newFixedThreadPool(10);
    private volatile CountDownLatch inFlight = new CountDownLatch(0);

    public Future<PaymentResult> process(PaymentRequest request) {
        inFlight = new CountDownLatch((int)inFlight.getCount() + 1);
        return executor.submit(() -> {
            try {
                return charge(request);
            } finally {
                inFlight.countDown();
            }
        });
    }

    public void shutdown() {
        executor.shutdown();
        try {
            if (!inFlight.await(30, TimeUnit.SECONDS)) {
                List<Runnable> cancelled = executor.shutdownNow();
                log.warn("Force shutdown {} pending tasks", cancelled.size());
            }
        } catch (InterruptedException e) {
            executor.shutdownNow();
            Thread.currentThread().interrupt();
        }
    }
}
```

The `CountDownLatch` tracks the exact number of in-flight requests, incrementing on each submission and decrementing in the `finally` block of each completed task. The three-phase shutdown — `shutdown()` (prevent new tasks), `awaitTermination()` (wait politely), `shutdownNow()` (force with interrupts) — is the standard pattern for graceful degradation. Each phase provides a fallback: first wait politely for up to 30 seconds, then force-interrupt remaining tasks, then log the count of tasks that were forcibly cancelled for operational visibility.

---

## Scenario-Based Questions

**Q: You are building a WebSocket-based live auction system. Each auction item has a countdown timer. When the timer reaches zero, no more bids are accepted. The timer must be accurate to within 100ms even under heavy load. How do you schedule and manage thousands of simultaneous auction timers?**

A: Use a single-threaded `ScheduledExecutorService` that manages all timers via a binary heap (not one thread per timer). Each item is tracked as a `ScheduledFuture` that closes bidding when it fires:
```java
public class AuctionTimer {
    private final ScheduledExecutorService scheduler = Executors.newScheduledThreadPool(1);
    private final ConcurrentHashMap<String, ScheduledFuture<?>> timers = new ConcurrentHashMap<>();

    public void startAuction(String itemId, long durationMs) {
        ScheduledFuture<?> existing = timers.put(itemId,
            scheduler.schedule(() -> closeBidding(itemId), durationMs, MILLISECONDS));
        if (existing != null) existing.cancel(false);
    }
}
```
A single scheduler thread is sufficient for thousands of timers because `ScheduledExecutorService` uses a `DelayedWorkQueue` (binary heap internally) that efficiently manages timer events without per-timer threads. When a timer fires, the scheduler submits the close action to a worker pool. The accuracy depends on the scheduler's tick precision — for 100ms accuracy, use `ScheduledExecutorService` rather than `Timer`, which blocks its single thread on long-running tasks and loses subsequent timer events.

**Q: A data processing job reads 10M records, transforms each record (CPU-intensive, 1ms each), and writes results to a database (I/O, 10ms each). On an 8-core machine with a fixed thread pool of 8, the job takes 60 minutes. How do you improve throughput?**

A: Separate CPU-bound and I/O-bound operations into different thread pools sized appropriately for each workload type:
```java
ExecutorService cpuPool = Executors.newFixedThreadPool(8);
ExecutorService ioPool = Executors.newFixedThreadPool(32);
CompletableFuture<?>[] futures = records.stream()
    .map(record -> CompletableFuture
        .supplyAsync(() -> transform(record), cpuPool)
        .thenComposeAsync(transformed ->
            CompletableFuture.runAsync(() -> db.write(transformed), ioPool))
    )
    .toArray(CompletableFuture[]::new);
CompletableFuture.allOf(futures).join();
```
With 8 threads on the CPU pool, transformation scales to fill all cores. With 32 threads on the I/O pool (computed as `cores * (1 + 10ms/1ms) = 8 * 11 ≈ 32`), the database writes can proceed in parallel so I/O wait time does not stall the CPU threads. The expected improvement is from 60 minutes to approximately 12 minutes — a 5x gain from simply separating the thread pools.

**Q: A multi-threaded logging library writes log entries to a file. Under high load, log lines from different threads are interleaved and corrupted. How do you ensure atomic writes per log line without making every log call block on a global lock?**

A: Use per-thread buffering with a background writer thread that receives completed buffers from a queue:
```java
public class ThreadLocalLogger {
    private final ThreadLocal<StringBuilder> threadBuffer = ThreadLocal.withInitial(() -> new StringBuilder(4096));
    private final BlockingQueue<String> queue = new LinkedBlockingQueue<>(10000);
    private final Thread writer = new Thread(() -> {
        while (true) {
            String line = queue.take();
            fileWriter.write(line + "\n");
        }
    });
    public void log(String message) {
        StringBuilder buf = threadBuffer.get();
        buf.append(format(message)).append("\n");
        if (buf.length() > 2048) {
            queue.offer(buf.toString());
            buf.setLength(0);
        }
    }
}
```
Each producer thread writes to its own `StringBuilder` with zero contention — no locking, no CAS. When the buffer reaches 2048 bytes, it is offered to the `BlockingQueue` with a non-blocking `offer()`. The single consumer thread drains the queue and writes to the file sequentially, preventing interleaving without requiring every log call to synchronize. This is the same design used by Logback's `AsyncAppender` and Log4j2's asynchronous logger.

**Q: A Spring Boot application has `@Async` methods for sending emails. Under load, the `ThreadPoolTaskExecutor` queue grows unbounded and the application runs out of memory. How do you add backpressure so that the caller blocks when the queue is full?**

A: Configure `ThreadPoolTaskExecutor` with a bounded queue and `CallerRunsPolicy` rejection handler:
```java
@Bean
public ThreadPoolTaskExecutor emailExecutor() {
    ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
    executor.setCorePoolSize(5);
    executor.setMaxPoolSize(10);
    executor.setQueueCapacity(100);
    executor.setRejectedExecutionHandler(new ThreadPoolExecutor.CallerRunsPolicy());
    executor.setThreadNamePrefix("email-");
    return executor;
}
```
When the queue capacity (100) and the maximum pool size (10) are both exhausted, `CallerRunsPolicy` makes the submitting thread execute the email-sending task itself rather than throwing `RejectedExecutionException`. This creates natural backpressure — the HTTP request thread blocks while sending the email, slowing the submission rate and pushing the pressure upstream to the client. This prevents unbounded queue growth and the resulting OOM.

**Q: A service runs a daily batch job that processes 1M records using `ForkJoinPool` with `RecursiveAction`. On an 8-core machine, only 4 cores are utilized. What's wrong with the fork/join split?**

A: The `compute()` threshold for splitting is too high — the task splits into only 4 chunks (1M ÷ 4 = 250K each) and processes each chunk sequentially within a single thread, leaving 4 cores idle. The fork/join pool uses work-stealing where idle threads steal work from busy threads' queues, but with only 4 chunks, there is nothing to steal. The fix is to set the threshold such that the number of tasks is approximately 10 times the parallelism level: threshold = 1,000,000 ÷ (8 × 10) = 12,500. With 80 tasks and the fork-join pool's work-stealing algorithm, idle threads immediately steal tasks from busy threads, keeping all 8 cores fully utilized.

**Q: After migrating to Java 8, a `ConcurrentHashMap`-based cache starts returning null for keys that were just inserted. The code uses `put()` then `get()` in separate statements. What's happening and how do you fix it?**

A: `put()` and `get()` are individually atomic, but the compound operation — put then get — is not atomic as a unit. Between the `put()` and `get()`, another thread may remove the entry via eviction or explicit removal, or a table resize may temporarily affect visibility. The fix is to use `compute()`, `merge()`, or `computeIfAbsent()` which execute the entire operation atomically under the per-bucket lock:
```java
// Non-atomic compound operation
cache.put(key, value);
V result = cache.get(key);

// Atomic — use merge or compute
V result = cache.compute(key, (k, v) -> v == null ? value : v);
```
The `compute()` method applies the remapping function atomically — between the read of the old value and the write of the new value, no other thread can modify that key's entry.

**Q: A microservice calls 3 downstream services in parallel using `Executors.newFixedThreadPool(10)` and `Future.get()`. Occasionally, one downstream service is slow (5s timeout), blocking one thread in the pool. If enough slow calls accumulate, all 10 threads are blocked. How do you prevent one slow service from consuming all threads?**

A: Use the bulkhead pattern — separate thread pools per downstream service so a slow service can only exhaust its own pool, not the shared pool:
```java
private final ExecutorService userPool = Executors.newFixedThreadPool(5);
private final ExecutorService orderPool = Executors.newFixedThreadPool(5);
private final ExecutorService inventoryPool = Executors.newFixedThreadPool(5);

public Dashboard getDashboard(String userId) {
    CompletableFuture<User> user = fetchAsync(userPool, "/users/" + userId);
    CompletableFuture<List<Order>> orders = fetchAsync(orderPool, "/orders?user=" + userId);
    CompletableFuture<List<Product>> inventory = fetchAsync(inventoryPool, "/inventory");
    return CompletableFuture.allOf(user, orders, inventory)
        .thenApply(v -> new Dashboard(user.join(), orders.join(), inventory.join()))
        .orTimeout(3, TimeUnit.SECONDS)
        .exceptionally(ex -> Dashboard.fallback());
}
```
With isolated thread pools, a slow user service blocks only 5 threads in the user pool, leaving the order and inventory pools unaffected. The 3-second overall timeout via `orTimeout()` prevents the entire request from hanging, and `exceptionally()` provides a fallback response. This is the bulkhead pattern from resilience engineering, implemented with separate thread pools.

**Q: A system uses `synchronized(this)` in all instance methods for thread safety. Under load, throughput is very low. Profiling shows all threads contending on the same lock. The class has 3 independent fields. How do you reduce lock contention?**

A: Replace one coarse-grained lock with three fine-grained locks, one per independent field (lock striping):
```java
// BEFORE — one lock for everything
public synchronized void setA(int a) { this.a = a; }
public synchronized void setB(int b) { this.b = b; }
public synchronized void setC(int c) { this.c = c; }

// AFTER — separate locks for independent fields
private final Object lockA = new Object();
private final Object lockB = new Object();
private final Object lockC = new Object();
public void setA(int a) { synchronized(lockA) { this.a = a; } }
public void setB(int b) { synchronized(lockB) { this.b = b; } }
public void setC(int c) { synchronized(lockC) { this.c = c; } }
```
With three separate locks, three threads can update different fields concurrently — throughput triples for operations on independent fields. This is the same lock-striping technique that `ConcurrentHashMap` uses internally with 16 buckets by default. The caveat is that operations accessing multiple fields together must lock all relevant locks in a consistent order to avoid deadlock.

**Q: A service creates a new thread pool for every request using `Executors.newCachedThreadPool()` which creates threads on demand. Under load, the JVM creates thousands of threads and crashes with OOM (unable to create native thread). How do you enforce a hard limit on thread creation?**

A: Never use `newCachedThreadPool()` for user-facing requests because it has an unbounded thread creation policy. Instead, use `new ThreadPoolExecutor()` with explicit bounds:
```java
// DANGEROUS — unbounded thread creation
Executors.newCachedThreadPool();

// SAFE — hard limit with bounded queue
ExecutorService safe = new ThreadPoolExecutor(
    10, 20, 60, TimeUnit.SECONDS,
    new ArrayBlockingQueue<>(100),
    new ThreadPoolExecutor.CallerRunsPolicy()
);
```
The `ThreadPoolExecutor` with `corePoolSize=10`, `maxPoolSize=20`, and an `ArrayBlockingQueue` of 100 ensures at most 20 threads are ever created. If the queue fills and all 20 threads are busy, `CallerRunsPolicy` applies backpressure by running the task on the submitting thread. The deadly error "unable to create native thread" occurs when the OS process hits the per-process thread limit (typically 1024-4096 on Linux), and the only fix is to bound thread creation at the application level.

**Q: A testing framework spawns 100 threads, each performing operations on a shared data structure. The test runs fine in single-threaded mode but fails intermittently in multi-threaded mode. The test has no explicit synchronization. How do you write a deterministic multi-threaded test that consistently catches race conditions?**

A: Use `CountDownLatch` to orchestrate specific thread interleavings that expose the race:
```java
@Test
public void testRaceCondition() throws Exception {
    CountDownLatch t1Ready = new CountDownLatch(1);
    CountDownLatch t2Ready = new CountDownLatch(1);
    CountDownLatch allDone = new CountDownLatch(2);
    ExecutorService executor = Executors.newFixedThreadPool(2);
    executor.submit(() -> {
        shared.set(1); t1Ready.countDown();
        t2Ready.await();
        int val = shared.get(); // should be 2
        allDone.countDown();
    });
    executor.submit(() -> {
        t1Ready.await(); shared.set(2); t2Ready.countDown();
        allDone.countDown();
    });
    allDone.await(5, TimeUnit.SECONDS);
}
```
For stress testing that probabilistically catches races, use JCStress (the OpenJDK concurrency stress tool) which runs millions of iterations with varying thread interleavings. The key distinction: deterministic tests use latches to force specific interleavings that prove a race exists, while stress tests run many iterations to find races with low probability.

---

## Interview Questions

**What is the difference between a process and a thread in Java?** A process has its own separate address space with expensive creation (via `fork/exec`) and strong isolation. A thread shares memory and file descriptors within a process, has lightweight creation, and cheaper context switching because memory mappings are shared. Java threads map 1:1 to OS kernel threads on modern JVMs.

**What is the thread lifecycle in Java?** `NEW` (created, not started), `RUNNABLE` (eligible for CPU scheduling), `BLOCKED` (waiting for a monitor lock), `WAITING` (indefinitely waiting via `wait()`, `join()`, `park()`), `TIMED_WAITING` (waiting with timeout), and `TERMINATED` (completed or exception thrown). The JVM uses OS thread states under the hood.

**What happens when a thread throws an uncaught exception?** The thread terminates and transitions to `TERMINATED`. The exception is passed to the thread's `UncaughtExceptionHandler`. If none is set, the default handler prints the stack trace to `System.err`. The exception does not propagate to other threads. Always set an `UncaughtExceptionHandler` for background threads to log failures.

**What is the difference between `Runnable` and `Callable`?** `Runnable.run()` has a `void` return and cannot throw checked exceptions. `Callable.call()` returns a typed value and can throw checked exceptions. `Callable` is used with `ExecutorService.submit()` which returns `Future<T>` to retrieve the result or exception. Use `Runnable` for fire-and-forget; use `Callable` when you need a result.

**What is `ThreadLocal` and how does it work?** `ThreadLocal<T>` provides per-thread variable isolation. Each `Thread` object holds a `ThreadLocalMap` that maps `ThreadLocal` instances to their per-thread values. When a thread calls `get()`, the `ThreadLocal` looks up its own map entry. Values are garbage collected when the thread dies. Use for request context, transaction IDs, and thread-safe `SimpleDateFormat` — always clear in `finally` blocks.

**What is the difference between `wait()` and `sleep()`?** `wait()` must be called inside a `synchronized` block and releases the monitor lock, putting the thread in `WAITING` until `notify()`/`notifyAll()` is called. `sleep()` does not release any locks and can be called anywhere, putting the thread in `TIMED_WAITING`. Use `wait()`/`notify()` for coordination; use `sleep()` for pausing.

**What is a daemon thread?** A daemon thread is a background thread that does not prevent the JVM from exiting — when only daemon threads remain, the JVM shuts down. Set with `thread.setDaemon(true)` before `start()`. Use for monitoring, periodic cache refresh, and health checks. Non-daemon threads keep the JVM alive and should be used for tasks that must complete.

**What is a thread pool and why use one?** A thread pool reuses a fixed number of threads to avoid the creation and destruction overhead of `new Thread()`. Benefits include bounded resource usage (thread count limit), reduced latency (threads are pre-created), task queuing, and graceful shutdown capabilities. Types include `FixedThreadPool`, `CachedThreadPool`, `SingleThreadExecutor`, and `ScheduledThreadPool`.

**How do you handle `InterruptedException` correctly?** Two correct options: propagate by declaring `throws InterruptedException`, or restore the interrupt flag by calling `Thread.currentThread().interrupt()` in the catch block. Never swallow `InterruptedException` in an empty catch block — this breaks cancellation, `shutdownNow()`, and `future.cancel(true)`.

**What is the optimal thread pool size for CPU-bound vs I/O-bound tasks?** CPU-bound: `Runtime.getRuntime().availableProcessors()`. More threads than cores cause context switching overhead without actual parallelism. I/O-bound: `cores * (1 + waitTime / computeTime)`, which is typically 2-4x the core count. Always measure with your specific workload rather than relying on formulas alone.

---

## Developer Recommendations

Use thread pools instead of `new Thread()` because each `new Thread()` allocates approximately 1MB of stack memory, and 1,000 concurrent threads consume 1GB for stacks alone. Thread pools reuse threads, limit resource usage, provide task queuing, and support graceful shutdown. Use `Executors.newFixedThreadPool(n)` with a bounded size — for CPU-bound work, size to `availableProcessors()`; for I/O-bound work, use `cores * (1 + waitTime / computeTime)`.

Name all threads for debuggability because `pool-1-thread-1` in a thread dump gives no clue about which component owns the stuck thread. Use Guava's `ThreadFactoryBuilder().setNameFormat("order-worker-%d").build()` or a custom `ThreadFactory`. Name thread pools after their function ("db-pool", "kafka-consumer", "health-check").

Use `CompletableFuture` over manual `Future` management because `ExecutorService.submit()` + `Future.get()` does not compose well — you cannot chain operations or combine results. `CompletableFuture` supports non-blocking callbacks with `thenApply()`, composition with `thenCompose()` and `thenCombine()`, declarative error handling with `exceptionally()`, and timeouts with `orTimeout()`. Always pass a dedicated `Executor` instead of using the common ForkJoinPool.

Always handle `InterruptedException` properly — never swallow it in an empty catch block. Restore the interrupt flag with `Thread.currentThread().interrupt()`, then either propagate or wrap in `RuntimeException`. Correct interrupt handling is essential for `shutdownNow()` and `future.cancel(true)` to work.

Prefer `BlockingQueue` over manual `wait()`/`notify()` for producer-consumer patterns. Manual `wait()`/`notify()` is prone to missed notifications, spurious wakeups, and lost interrupts. `BlockingQueue` handles all of these correctly. Use `LinkedBlockingQueue` for unbounded scenarios, `ArrayBlockingQueue` for bounded (backpressure), and `SynchronousQueue` for zero-capacity handoffs.

Use `volatile` for flags, `AtomicInteger` for counters, and `synchronized` for compound actions. `volatile` guarantees visibility for single variable writes. `AtomicInteger` provides atomic read-modify-write for counters via CAS. `synchronized` is required for compound actions like check-then-act that span multiple variables.

Use `ScheduledExecutorService` over `Timer` because `Timer` uses a single thread — if any task throws an uncaught exception, all subsequent scheduled tasks are cancelled. `ScheduledExecutorService` uses a thread pool, so a failure in one task does not affect others. It also supports `scheduleAtFixedRate()` and `scheduleWithFixedDelay()` with configurable thread naming.
