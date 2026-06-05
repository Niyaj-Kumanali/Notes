# Java Multithreading

---

## Overview

- **Definition:** Multithreading is the ability of a CPU to execute multiple threads concurrently. A thread is the smallest unit of execution within a process. Java provides built-in support for creating and managing threads.

- **Why Multithreading?**
  - **CPU utilization** — keep CPU busy while one thread waits for I/O
  - **Responsiveness** — UI thread stays responsive while background threads work
  - **Throughput** — process multiple requests concurrently
  - **Resource sharing** — threads within a process share memory (cheaper than processes)

- **Thread vs Process:**
  - **Process:** Separate address space, heavy creation (fork/exec), expensive context switch, strong isolation
  - **Thread:** Shared memory within process, lightweight creation, cheaper context switch, weak isolation

---

## Creating Threads

- **Extend Thread class:** Simple but not recommended (can't extend other classes)
  ```java
  class Worker extends Thread {
      @Override
      public void run() {
          System.out.println("Working in thread: " + getName());
      }
  }
  new Worker().start();
  ```

- **Implement Runnable (preferred):** Separates task from execution mechanism
  ```java
  class Task implements Runnable {
      @Override
      public void run() {
          System.out.println("Task in thread: " + Thread.currentThread().getName());
      }
  }
  new Thread(new Task()).start();
  ```

- **Lambda (Java 8+):** Most concise for simple tasks
  ```java
  new Thread(() -> System.out.println("Lambda thread")).start();
  ```

- **Callable + ExecutorService:** Returns a result
  ```java
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

- **NEW:** Created but not started
- **RUNNABLE:** Eligible for execution (waiting for CPU)
- **BLOCKED:** Waiting for a monitor lock (synchronized)
- **WAITING:** Indefinitely waiting (wait(), join(), park())
- **TIMED_WAITING:** Waiting with timeout (sleep(), wait(timeout))
- **TERMINATED:** run() completed or exception thrown

---

## Thread Properties

- **Thread Name:** Always set meaningful names for debugging:
  ```java
  thread.setName("worker-pool-1");
  ```

- **Daemon Threads:** Background threads that don't keep JVM alive. JVM exits when only daemon threads remain:
  ```java
  thread.setDaemon(true);  // Must be called before start()
  ```

- **Priority:** `Thread.setPriority(Thread.NORM_PRIORITY)` (1-10). Platform-dependent — not guaranteed by OS.

---

## Under the Hood

- **Thread Stack:** Each thread gets its own stack (default ~1MB on 64-bit JVM, configurable with `-Xss1m`). Stores local variables and method call frames. Not shared between threads.

- **Thread Scheduling:** Modern JVMs use OS-level threads (1:1 mapping with kernel threads). The thread scheduler (OS-dependent) decides which thread runs. Java doesn't guarantee scheduling order or fairness.

- **Context Switch Cost:** ~1-10µs per context switch. Includes saving/restoring registers, TLB flush, and cache misses on resumption.

- **Thread Overhead:**
  - Thread creation: ~1-2µs (JVM call + OS)
  - Stack memory: ~1MB default (1024 threads = 1GB just for stacks)
  - Context switch: ~1-10µs

---

## Thread Pools (ExecutorService)

- **Definition:** Managing threads manually is error-prone. ExecutorService provides a higher-level API for managing thread pools.

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

- **Creating threads in unbounded loops** — thread leak, OOM. Use thread pools.
- **Not naming threads** — impossible to debug thread dumps. Always `setName()`.
- **Swallowing InterruptedException** — lost interrupt signal. Restore flag: `Thread.currentThread().interrupt()`
- **Not shutting down executor service** — thread leak, JVM never exits. `shutdown()` in `@PreDestroy`
- **Using Thread.stop()/suspend()/resume()** — deprecated, unsafe. Use volatile flag or interrupt.
- **Thread without exception handler** — silent failures. `setUncaughtExceptionHandler()`
- **Assuming thread priority matters** — OS typically ignores it. Don't rely on it.
- **Starting thread in constructor** — `this` escapes before full construction. Use factory method.
- **Not using daemon threads for background** — JVM can't exit. `setDaemon(true)` for background tasks.

---

## Real-World Scenarios

### Scenario 1: Async Image Processing Pipeline

A photo sharing app uploads images that need resizing (thumbnail, medium, full-size), metadata extraction (EXIF), and content moderation (NSFW detection). Each step is CPU or I/O intensive. The pipeline must process 100 uploads/second without blocking the HTTP request thread.

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

Separating CPU-bound (resizing, moderation) and I/O-bound (metadata extraction) operations into different thread pools prevents I/O threads from starving CPU work. The pipeline is completely non-blocking — the HTTP thread submits the work and returns immediately. `CompletableFuture` chains ensure each step starts when the previous completes, without blocking any thread. The two-pool design uses `availableProcessors()` for CPU work (optimal for compute) and `cachedThreadPool()` for I/O (threads created as needed, recycled when idle).

### Scenario 2: WebSocket Connection Manager

A collaborative editing application maintains 50K concurrent WebSocket connections. Each connection receives document edits from the user and broadcasts changes to other collaborators in the same document. The system must scale to handle burst edits without losing messages.

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

`CopyOnWriteArrayList` provides thread-safe iteration without locks — reads are never blocked by modifications (users joining/leaving). The `broadcastPool` with bounded threads prevents a flood of edits from creating unlimited threads. The `synchronized(member)` block ensures ordered delivery per connection — edits are sent one at a time in FIFO order. This design maintains 50K connections with only 16 broadcast threads because the threads are used for I/O (sending), not for waiting.

### Scenario 3: Graceful Shutdown in a Payment Gateway

A payment gateway processes credit card charges. During a deployment, the system receives a shutdown signal but must finish processing the 50 in-flight transactions before terminating. If a transaction doesn't complete within 30 seconds, it must be marked as failed and retried by another instance.

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

The `CountDownLatch` tracks in-flight requests. `shutdown()` first prevents new tasks, then waits for current tasks up to 30 seconds. If tasks don't complete, `shutdownNow()` interrupts them. The three-phase shutdown (`shutdown()` → `awaitTermination()` → `shutdownNow()`) is the standard pattern for graceful degradation. Each phase provides a fallback: first wait politely, then interrupt, then log.

---

## Scenario-Based Questions

1. **Q: You are building a WebSocket-based live auction system. Each auction item has a countdown timer. When the timer reaches zero, no more bids are accepted. The timer must be accurate to within 100ms even under heavy load. How do you schedule and manage thousands of simultaneous auction timers?**
   A: Use a `ScheduledExecutorService` with a single thread to manage all timers. Each item is a `ScheduledFuture` that cancels the bidding when it fires:
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
   A single scheduler thread is sufficient for thousands of timers because `ScheduledExecutorService` uses a `DelayedWorkQueue` (binary heap) — it doesn't create one thread per timer. When a timer fires, it submits the close action to a worker pool. The accuracy depends on the scheduler's tick precision and system load — for 100ms accuracy, use `ScheduledExecutorService` (not `Timer` which uses a single thread and blocks on long tasks).

2. **Q: A data processing job reads 10M records, transforms each record (CPU-intensive, 1ms each), and writes results to a database (I/O, 10ms each). On an 8-core machine with a fixed thread pool of 8, the job takes 60 minutes. How do you improve throughput?**
   A: Separate CPU-bound and I/O-bound operations into different thread pools. The bottleneck is I/O — threads spend 90% of time waiting for database writes:
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
   With 8 CPU threads for transformation and 32 I/O threads for DB writes, the pipeline overlaps computation with I/O. Expected improvement: from 60 minutes to ~12 minutes (5x).

3. **Q: A multi-threaded logging library writes log entries to a file. Under high load, log lines from different threads are interleaved and corrupted. How do you ensure atomic writes per log line without making every log call block on a global lock?**
   A: Use per-thread buffering with a background writer:
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
   Each thread writes to its own `StringBuilder` — no contention. When the buffer reaches 2048 bytes, it's offered to the queue (non-blocking). The single writer thread drains the queue and writes to file — writes are serialized, preventing interleaving. This is the design used by Logback's `AsyncAppender`.

4. **Q: A Spring Boot application has `@Async` methods for sending emails. Under load, the `ThreadPoolTaskExecutor` queue grows unbounded and the application runs out of memory. How do you add backpressure so that the caller blocks when the queue is full?**
   A: Configure `ThreadPoolTaskExecutor` with a bounded queue and `CallerRunsPolicy`:
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
   When the queue (100) and max pool (10) are both full, `CallerRunsPolicy` makes the submitting thread run the task, creating natural backpressure — the HTTP request blocks until the email is sent, slowing down the submission rate. This prevents OOM and pushes pressure upstream to the client.

5. **Q: A service runs a daily batch job that processes 1M records using `ForkJoinPool` with `RecursiveAction`. On an 8-core machine, only 4 cores are utilized. What's wrong with the fork/join split?**
   A: The threshold for splitting is too high — the task splits into 4 chunks (1M/4 = 250K each) and processes sequentially within each chunk. Fix: set the threshold so that the number of tasks is ~10x the parallelism level (e.g., threshold = 1M / 80 = 12,500). The fork/join pool uses work-stealing — idle threads steal tasks from busy threads' queues. With 80 tasks and 7 workers, work-stealing distributes work evenly and prevents one thread from being stuck with a large chunk while others are idle.

6. **Q: After upgrading to Java 8, a `ConcurrentHashMap`-based cache starts returning null for keys that were just inserted. The code uses `put()` then `get()` in separate statements. What's happening and how do you fix it?**
   A: `put()` and `get()` are individually atomic but the compound operation (put then get) is not atomic. Between `put()` and `get()`, another thread may remove the entry or trigger a resize that affects visibility. Fix by using `compute()` for atomic read-modify-write:
   ```java
   // Non-atomic compound operation
   cache.put(key, value);
   V result = cache.get(key);

   // Atomic — use merge or compute
   V result = cache.compute(key, (k, v) -> v == null ? value : v);
   ```
   Use `compute()`, `merge()`, or `computeIfAbsent()` for compound operations that must be atomic.

7. **Q: A microservice calls 3 downstream services in parallel using `Executors.newFixedThreadPool(10)` and `Future.get()`. Occasionally, one downstream service is slow (5s timeout), blocking one thread in the pool. If enough slow calls accumulate, all 10 threads are blocked. How do you prevent one slow service from consuming all threads?**
   A: Use separate thread pools per downstream service (bulkhead pattern):
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
   Isolating downstream services into separate pools prevents one slow service from starving others. If the user service is slow, only the 5 user-service threads are blocked. The 3-second overall timeout prevents the entire request from hanging.

8. **Q: A system uses `synchronized(this)` in all instance methods for thread safety. Under load, throughput is very low. Profiling shows all threads contending on the same lock. The class has 3 independent fields. How do you reduce lock contention?**
   A: Use separate locks for independent fields (lock striping):
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
   With three locks, three threads can update different fields concurrently. This is the same idea `ConcurrentHashMap` uses with per-bucket locks. If fields are frequently accessed together, keep them under the same lock to maintain consistency.

9. **Q: A service creates a new thread pool for every request using `Executors.newCachedThreadPool()` which creates threads on demand. Under load, the JVM creates thousands of threads and crashes with OOM (unable to create native thread). How do you enforce a hard limit on thread creation?**
   A: Never use `newCachedThreadPool()` for user-facing requests. It creates unbounded threads. Use a fixed pool or `ThreadPoolExecutor` with a hard limit:
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
   The `ThreadPoolExecutor` with `ArrayBlockingQueue` ensures at most 20 threads are ever created. If the queue is also full, `CallerRunsPolicy` applies backpressure. The error "unable to create native thread" occurs when the OS process hits its thread limit (typically 1024-4096 on Linux).

10. **Q: A testing framework spawns 100 threads, each performing operations on a shared data structure. The test runs fine in single-threaded mode but fails intermittently in multi-threaded mode. The test has no explicit synchronization. How do you write a deterministic multi-threaded test that consistently catches race conditions?**
    A: Use `CountDownLatch` to coordinate threads to execute specific interleavings:
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
    For stress testing, use a thread sanitizer (vmlens, ThreadSanitizer) or JCStress (OpenJDK's concurrency stress tester). The key: deterministic tests use latches to force specific interleavings; stress tests use many iterations with random timing.

---

## Interview Questions

1. **What is the difference between a process and a thread in Java?**
   A: A process has its own address space, heavy creation (fork/exec), and strong isolation. A thread shares memory within a process, has lightweight creation, and weaker isolation. Java threads are mapped to OS threads (1:1 model on modern JVMs). Context switching between threads is cheaper because threads share memory mappings and file descriptors.

2. **What is the thread lifecycle in Java?**
   A: `NEW` (created but not started), `RUNNABLE` (eligible for execution, waiting for CPU), `BLOCKED` (waiting for a monitor lock), `WAITING` (indefinitely waiting via `wait()`, `join()`, `park()`), `TIMED_WAITING` (waiting with timeout), `TERMINATED` (completed or threw exception).

3. **What happens when a thread throws an uncaught exception?**
   A: The thread terminates (moves to `TERMINATED`). The uncaught exception is handled by the thread's `UncaughtExceptionHandler`. If none is set, the default handler prints the stack trace to `System.err`. The exception does NOT propagate to other threads. Always set an `UncaughtExceptionHandler` for background threads to log failures.

4. **What is the difference between `Runnable` and `Callable`?**
   A: `Runnable` has `void run()` — no return value, cannot throw checked exceptions. `Callable<T>` has `T call()` — returns a value, can throw checked exceptions. `Callable` is used with `ExecutorService.submit()` which returns `Future<T>`. Use `Runnable` for fire-and-forget; use `Callable` when you need a result or checked exception propagation.

5. **What is `ThreadLocal` and how does it work?**
   A: `ThreadLocal<T>` provides per-thread variable isolation. Each `Thread` has a `ThreadLocalMap` mapping `ThreadLocal` instances to values. When a thread accesses `threadLocal.get()`, it looks up its own map. Values are GC'd when the thread dies. Use for: request context, transaction IDs, thread-safe `SimpleDateFormat`. Always clear in `finally` blocks when using thread pools.

6. **What is the difference between `wait()` and `sleep()`?**
   A: `wait()` releases the monitor lock and must be called inside `synchronized`. It puts the thread in `WAITING` until `notify()`/`notifyAll()`. `sleep()` does NOT release locks and can be called anywhere. It puts the thread in `TIMED_WAITING`. Use `wait()`/`notify()` for coordination; use `sleep()` for pausing. Prefer `TimeUnit.SECONDS.sleep()` over `Thread.sleep()`.

7. **What is a daemon thread?**
   A: A daemon thread is a background thread that doesn't prevent the JVM from exiting. When only daemon threads remain, the JVM shuts down. Use for: monitoring, periodic cache refresh, health checks. Set `thread.setDaemon(true)` before `start()`. Non-daemon threads keep the JVM alive — use for tasks that must complete.

8. **What is a thread pool and why use one?**
   A: A thread pool reuses a fixed number of threads, avoiding creation/destruction overhead per task. Benefits: bounded resource usage, reduced latency, task queuing, and graceful shutdown. Types: `FixedThreadPool` (bounded for stability), `CachedThreadPool` (unbounded, bursty short tasks), `SingleThreadExecutor` (serial execution), `ScheduledThreadPool` (delayed/periodic tasks).

9. **How do you handle `InterruptedException` correctly?**
   A: Two options: (1) Propagate — declare `throws InterruptedException`. (2) Restore the interrupt flag — catch, call `Thread.currentThread().interrupt()`, and either return fallback or rethrow as `RuntimeException`. Never swallow it — empty catch blocks break cancellation. Interruption is how `shutdownNow()` and `future.cancel()` work.

10. **What is the optimal thread pool size for CPU-bound vs I/O-bound tasks?**
    A: CPU-bound: `Runtime.getRuntime().availableProcessors()`. More threads than cores cause context switching overhead without parallelism. I/O-bound: `cores * (1 + waitTime / computeTime)` which is typically 2-4x core count. Too few threads underutilize the CPU during I/O waits; too many cause excessive context switching. Always measure and adjust.

---

## Developer Recommendations

- **Use thread pools instead of `new Thread()`** — Each `new Thread()` creates ~1MB of stack memory. 1000 concurrent requests = 1GB for stacks. Thread pools reuse threads, limit resource usage, and provide graceful shutdown. Use `Executors.newFixedThreadPool()` with a bounded size. For CPU-bound work: `availableProcessors()`. For I/O-bound: `cores * (1 + waitTime / computeTime)`.

- **Name your threads for debuggability** — `pool-1-thread-1` is useless in thread dumps. Use `new ThreadFactoryBuilder().setNameFormat("order-worker-%d").build()` (Guava) or a custom `ThreadFactory`. Name thread pools after their function ("db-pool", "kafka-consumer"). This turns hours of thread-dump analysis into minutes.

- **Use `CompletableFuture` over manual `Future` management** — Manual `ExecutorService.submit()` + `Future.get()` doesn't compose well. `CompletableFuture` chains async operations with `thenApply()`, `thenCompose()`, `thenCombine()`, provides declarative error handling (`exceptionally()`), and timeouts (`orTimeout()`). Always pass a dedicated `Executor` instead of using the common ForkJoinPool.

- **Always handle `InterruptedException` properly** — Never swallow it. Restore the flag: `Thread.currentThread().interrupt()`. If you can't propagate the checked exception, wrap in `RuntimeException` but preserve the interrupt flag. This is critical for `shutdownNow()` and `future.cancel()` to work correctly.

- **Prefer `BlockingQueue` over `wait()`/`notify()` for producer-consumer** — Manual `wait()`/`notify()` has missed notifications, spurious wakeups, and lost interrupts. `BlockingQueue` handles all correctly. Use `LinkedBlockingQueue` for unbounded, `ArrayBlockingQueue` for bounded (backpressure), and `SynchronousQueue` for zero-capacity handoff.

- **Use `volatile` for flags, `AtomicInteger` for counters, `synchronized` for compound actions** — `volatile` guarantees visibility for a single variable (flag, status). `AtomicInteger` provides atomic read-modify-write for counters. `synchronized` is needed for compound actions (check-then-act across multiple variables). Using `volatile` for `count++` is still a race condition.

- **Use `ScheduledExecutorService` over `Timer`** — `Timer` uses a single thread; if a task throws, all subsequent tasks are cancelled. `ScheduledExecutorService` uses a thread pool — one slow task doesn't affect others. It supports `scheduleAtFixedRate()` vs `scheduleWithFixedDelay()` and better error handling. Always use it for new code.

**Answer:** Creating a thread per request is unbounded — each thread consumes ~1MB of stack memory. Under load, 1000 concurrent requests = 1GB just for thread stacks. Fix: use a fixed thread pool that limits the number of concurrent threads:

```java
// BEFORE — thread per request (OOM risk)
new Thread(() -> process(request)).start();

// AFTER — bounded thread pool
private final ExecutorService executor = Executors.newFixedThreadPool(20);

public void handleRequest(Request request) {
    executor.submit(() -> process(request));
}

@PreDestroy
public void shutdown() {
    executor.shutdown();
}
```

---

**Scenario 2: An application hangs during shutdown. Thread dump shows several threads in WAITING state. The main thread is waiting on `Thread.join()`.**

**Answer:** Worker threads didn't terminate because they're blocking on an operation (e.g., queue.take(), socket.accept()). Fix: (1) Use `executor.shutdownNow()` which sends interrupts to worker threads. (2) Worker threads must handle `InterruptedException` properly. (3) For blocking operations, use timeouts.

```java
// Shutdown with timeout + force
executor.shutdown();
try {
    if (!executor.awaitTermination(30, TimeUnit.SECONDS)) {
        executor.shutdownNow();  // Interrupts running tasks
        if (!executor.awaitTermination(10, TimeUnit.SECONDS)) {
            log.error("Executor did not terminate");
        }
    }
} catch (InterruptedException e) {
    executor.shutdownNow();
    Thread.currentThread().interrupt();
}
```

---

**Scenario 3: A batch job uses 100 threads to process records. Processing is CPU-intensive but performance doesn't scale beyond 8 threads. What's wrong?**

**Answer:** CPU-intensive tasks don't benefit from more threads than available CPU cores. Excess threads cause context switching overhead without actual parallelism. Fix: set thread pool size to `Runtime.getRuntime().availableProcessors()` for CPU-bound tasks. For I/O-bound tasks, the formula is different (higher thread count).

```java
int cores = Runtime.getRuntime().availableProcessors();
ExecutorService executor = Executors.newFixedThreadPool(cores);
```

For I/O-bound tasks, use `cores * (1 + waitTime / computeTime)` which often results in 2-4x the core count.

---

**Scenario 4: A multithreaded service intermittently reports incorrect final results. Investigation shows `ConcurrentModificationException` in logs. The code uses `ArrayList` without synchronization.**

**Answer:** `ArrayList` is not thread-safe. When one thread modifies it (add/remove) while another iterates, the fail-fast iterator throws `ConcurrentModificationException`. Fix: use `CopyOnWriteArrayList` for read-heavy, write-rare lists, or `Collections.synchronizedList()` for general use, or better — use a thread-safe collection designed for the access pattern.

```java
// DANGEROUS — shared ArrayList
private List<String> list = new ArrayList<>();

// SAFE — thread-safe
private List<String> list = new CopyOnWriteArrayList<>();

// OR for different access patterns
private List<String> list = Collections.synchronizedList(new ArrayList<>());
```

---

**Scenario 5: A background thread reads sensor data and updates shared variables. The main thread reads these variables but always sees stale values. Both use `Thread.sleep()` for timing. What's wrong?**

**Answer:** Without proper happens-before relationships, the main thread may never see the background thread's updates due to CPU caching. The variables need synchronization or `volatile` keyword:

```java
// WRONG — no visibility guarantee
private int sensorValue;

// RIGHT — volatile guarantees visibility
private volatile int sensorValue;

// OR use AtomicInteger
private AtomicInteger sensorValue = new AtomicInteger();
```

Note: `sleep()` does NOT establish happens-before. Only synchronization, volatile, and atomic variables do.

---

**Scenario 6: A server crashes with 100% CPU usage. Thread dump shows a thread in an infinite loop spinning on `while (!ready)`. No synchronization is used.**

**Answer:** This is a busy-wait pattern that consumes 100% CPU. Additionally, without volatile, the loop may never see the updated `ready` flag. Fix: (1) Use proper synchronization (wait/notify, BlockingQueue, or CountDownLatch). (2) If busy-wait is unavoidable, add `Thread.onSpinWait()` (Java 9+) or `Thread.yield()`.

```java
// BAD — busy wait, 100% CPU
while (!ready) { }  // May never terminate without volatile

// BETTER — wait/notify
synchronized(lock) {
    while (!ready) {
        lock.wait();  // Releases CPU, waits for notification
    }
}

// GOOD for very short waits (Java 9+)
while (!ready) {
    Thread.onSpinWait();  // Hint to CPU: spin-loop, optimize power
}
```

---

**Scenario 7: A scheduled task monitoring system health occasionally freezes for several seconds. Thread dump shows the health check thread in TIMED_WAITING on `Thread.sleep(5000)`.**

**Answer:** `Thread.sleep()` only pauses the current thread — it doesn't affect other threads. The freeze is likely caused by another thread holding a lock that the health check thread needs, or by a GC pause. If `sleep()` is used for timing, switch to `ScheduledExecutorService` which doesn't block a thread between executions:

```java
// BAD — sleep-based scheduling (wastes thread)
while (true) {
    healthCheck();
    Thread.sleep(5000);
}

// GOOD — ScheduledExecutorService
ScheduledExecutorService scheduler = Executors.newScheduledThreadPool(1);
scheduler.scheduleAtFixedRate(this::healthCheck, 0, 5, TimeUnit.SECONDS);
```

---

**Scenario 8: A thread pool executor has `corePoolSize=10` and `maxPoolSize=100`. When 50 tasks are submitted, only 10 execute at a time. Why?**

**Answer:** `ThreadPoolExecutor` behavior: it creates new threads up to `corePoolSize` and queues remaining tasks. New threads are created beyond `corePoolSize` only when the queue is full. The default `LinkedBlockingQueue` is unbounded, so tasks queue up and no new threads are created. Fix: use a bounded queue:

```java
// Default — unbounded queue, only 10 threads created
new ThreadPoolExecutor(10, 100, 60, SECONDS, new LinkedBlockingQueue<>());

// Fixed — bounded queue, up to 100 threads created
new ThreadPoolExecutor(10, 100, 60, SECONDS, new ArrayBlockingQueue<>(200));
```

Or use `ThreadPoolExecutor.CallerRunsPolicy` as the rejection handler — the submitting thread runs the task when the pool and queue are full, providing natural backpressure.

---

**Scenario 9: A long-running background thread needs to be stopped gracefully. `Thread.stop()` is deprecated. What's the correct approach?**

**Answer:** Use a volatile flag for cooperative cancellation:

```java
public class BackgroundWorker {
    private volatile boolean running = true;

    public void start() {
        Thread worker = new Thread(() -> {
            while (running && !Thread.currentThread().isInterrupted()) {
                try {
                    doWork();
                } catch (InterruptedException e) {
                    Thread.currentThread().interrupt();
                    break;
                }
            }
        });
        worker.setName("background-worker");
        worker.setDaemon(true);
        worker.start();
    }

    public void stop() {
        running = false;  // Volatile write — visible to worker thread
    }
}
```

If the thread is blocked on I/O or wait(), call `worker.interrupt()` to wake it up.

---

**Scenario 10: After migrating from a single-threaded to a multi-threaded architecture, the application produces non-deterministic results. What's the root cause and how do you systematically fix it?**

**Answer:** Non-deterministic results indicate a race condition — threads are accessing shared mutable state without proper synchronization. Systematic approach: (1) Identify all shared mutable state. (2) Choose the right concurrency strategy for each: immutability (preferred), thread-safe collections, synchronization, or atomic variables. (3) Remove shared state where possible (thread-local storage). (4) Add proper visibility guarantees (volatile, synchronized, atomic). (5) Use higher-level abstractions (CompletableFuture, parallel streams) that manage thread safety internally. (6) Add stress tests with thread sanitizers to detect remaining races.
