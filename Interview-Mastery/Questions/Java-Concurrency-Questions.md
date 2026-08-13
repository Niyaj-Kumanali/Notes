# Java Concurrency Questions

## Questions

1. What is a thread?
2. What is process vs thread?
3. How do you create a thread in Java?
4. Difference between extending `Thread` and implementing `Runnable`.
5. What is `Callable`?
6. Difference between `Runnable` and `Callable`.
7. What is `Future`?
8. What is `ExecutorService`?
9. What is thread pool?
10. Why use thread pools?
11. Difference between fixed thread pool and cached thread pool.
12. What is scheduled executor service?
13. What is synchronization?
14. What is race condition?
15. What is deadlock?
16. How do you prevent deadlock?
17. What is livelock?
18. What is starvation?
19. What is `volatile`?
20. Difference between `volatile` and `synchronized`.
21. What is atomic variable?
22. What is `AtomicInteger`?
23. What is lock?
24. Difference between intrinsic lock and `ReentrantLock`.
25. What is read-write lock?
26. What is semaphore?
27. What is countdown latch?
28. What is cyclic barrier?
29. What is concurrent collection?
30. What is `ThreadLocal`?
31. How does Spring Security use `ThreadLocal`?
32. What happens to `ThreadLocal` in async execution?
33. What is context propagation?
34. What is thread safety?
35. How do you make a class thread-safe?

---

## Answers

1. What is a thread?
   - **Answer:** A thread is the smallest schedulable unit of execution within a process. Each thread has its own program counter and call stack (holding local variables and method invocations), but all threads in a process share the same heap and method area, so they can communicate directly through shared objects. In the JVM, every application starts with a `main` thread, and additional threads are created to run tasks concurrently. Java threads follow a well-defined lifecycle: `NEW` (created but not started), `RUNNABLE` (ready to run or running), `BLOCKED` (waiting to acquire a monitor), `WAITING` (waiting indefinitely for another thread), `TIMED_WAITING` (waiting for a bounded period), and `TERMINATED` (execution finished). Since Java 21, threads come in two flavors: platform threads (each mapped 1:1 to an OS thread) and virtual threads (lightweight threads multiplexed onto a small pool of carrier OS threads). Because threads share heap memory, unsynchronized access to shared mutable state can lead to race conditions, so coordination mechanisms such as locks, `volatile`, and atomic classes are required. The key mental model is that threads provide concurrency through interleaved execution, and the OS scheduler decides when each thread runs.

2. What is process vs thread?
   - **Answer:** A process is an independent program that owns its own memory space (address space), file descriptors, and other OS resources; processes are isolated from one another. A thread is a lightweight execution unit inside a process, and multiple threads within the same process share the process's heap and global data. Inter-process communication (IPC) requires explicit mechanisms such as pipes, sockets, shared memory, or message queues, whereas threads communicate simply by reading and writing shared objects. Threads are much cheaper to create and context-switch than processes because they do not require duplicating the entire memory space. The JVM itself is a single OS process that hosts many threads; the GC runs on background threads while application code runs on others. Context switching between threads is faster because much of the state (heap, open files) is already shared, while switching processes requires the OS to swap memory mappings and caches. This is why multithreading is preferred over multiprocessing for backend services that need high concurrency within one runtime, although processes provide better fault isolation since a crash in one process does not take down others.

3. How do you create a thread in Java?
   - **Answer:** Java provides three primary ways to create concurrent work. The first is extending the `Thread` class and overriding `run()`; the second is implementing `Runnable` and passing an instance to a `Thread` or `Executor`; the third is using `Callable` with an `ExecutorService` to get a return value. With lambdas, a simple thread can be started as `new Thread(() -> System.out.println("work")).start();`. The recommended modern approach is to define the task separately from the execution mechanism: implement `Runnable` or `Callable` and submit it to an `ExecutorService`, which manages thread lifecycle and pool sizing. Directly creating `new Thread(...)` per task is discouraged because it couples the task to raw thread creation, bypasses pooling, and makes resource control and shutdown harder. For fire-and-forget or result-producing async work, `CompletableFuture.supplyAsync(...)` is a clean abstraction that internally uses the common ForkJoinPool. Whichever approach is chosen, the code inside `run()`/`call()` must handle exceptions itself, because exceptions thrown from `run()` cannot be caught by the caller. Example:

   ```java
   ExecutorService executor = Executors.newFixedThreadPool(4);
   Future<Integer> future = executor.submit(() -> 42);
   int result = future.get(); // blocks until the task completes
   executor.shutdown();
   ```

4. Difference between extending `Thread` and implementing `Runnable`.
   - **Answer:** Extending `Thread` makes the class itself a thread and locks it into the single-inheritance hierarchy, so it cannot extend any other class, which is a serious design constraint. Implementing `Runnable` keeps the task and the execution mechanism decoupled: the `Runnable` describes *what* to do, while a `Thread` or an `ExecutorService` decides *how and when* to run it. Since `Thread` itself implements `Runnable`, both approaches ultimately execute code in a `run()` method, so there is no functional difference in thread creation itself. Implementing `Runnable` is considered best practice because it follows composition over inheritance and the Strategy pattern, allowing the same task object to be reused across different pools or passed to constructors. Extending `Thread` also invites accidental override of methods like `start()`, and it exposes the full thread API on what should be a plain task class. With `Runnable`, a lambda such as `Runnable r = () -> doWork();` can be submitted directly to an executor. The clean separation is especially valuable in frameworks such as Spring, where beans are proxied; a bean that extends `Thread` cannot be proxied or injected cleanly. In short, extend `Thread` only in trivial examples; in real code, implement `Runnable` or use `Callable`.

5. What is `Callable`?
   - **Answer:** `Callable` is a functional interface in `java.util.concurrent` that represents a task that produces a result and can throw a checked exception. Its single abstract method is `V call()` (as opposed to `Runnable`'s `void run()`), so a `Callable` can return a value and propagate exceptions to the caller. It is normally used with an `ExecutorService`, which submits the task and returns a `Future` that holds the eventual result:

   ```java
   Callable<Integer> task = () -> expensiveComputation();
   ExecutorService executor = Executors.newFixedThreadPool(2);
   Future<Integer> future = executor.submit(task);
   Integer result = future.get(); // rethrows exceptions wrapped in ExecutionException
   ```

   Because `call()` may throw `Exception`, `Future.get()` wraps any failure in an `ExecutionException`, so error handling happens on the calling side rather than inside the task. `Callable` is the right choice whenever a task needs to return data, such as fetching a record, computing a value, or querying a service. `CompletableFuture.supplyAsync(Supplier)` and similar APIs internally wrap the work in `Callable` semantics, which shows how central the interface is to modern concurrency. Unlike `Runnable`, a `Callable` cannot be passed directly to a `Thread` constructor, so it is tied to the executor abstraction. The combination of `Callable`, `Future`, and `ExecutorService` forms the foundational building block for all result-oriented async computation in Java.

6. Difference between `Runnable` and `Callable`.
   - **Answer:** `Runnable` has `void run()`, returns nothing, and cannot throw checked exceptions; `Callable` has `V call()`, returns a result, and can throw checked exceptions. This one signature difference drives everything else: `Runnable` is for fire-and-forget work such as logging, cache warming, or notification, while `Callable` is for tasks whose result the caller needs. When submitting to an executor, `submit(Runnable)` returns `Future<?>` whose `get()` yields `null`, whereas `submit(Callable)` returns `Future<T>` whose `get()` yields the computed value. Lambda syntax makes the difference visible: `() -> work()` is a `Runnable`, while `() -> computeValue()` is a `Callable`. Checked exceptions also differ: a `Runnable` must catch them internally or wrap them, while a `Callable` can declare `throws Exception` and let the caller handle the failure through `Future.get()`. In practice, `CompletableFuture.runAsync` expects a `Runnable` and `CompletableFuture.supplyAsync` expects a `Supplier`, which maps to the same void-versus-value distinction. Choosing between them is really a question of whether the task produces a result and whether the caller needs failure details.

7. What is `Future`?
   - **Answer:** `Future` is an interface that represents the result of an asynchronous computation that may not be complete yet. It is returned by `ExecutorService.submit()` and provides methods such as `get()` (blocking until the result is ready), `get(timeout, unit)` (blocking with a bound), `isDone()`, `isCancelled()`, and `cancel()`. The blocking `get()` throws `InterruptedException` if the waiting thread is interrupted and `ExecutionException` if the task itself failed, so the caller must handle both. `Future` gives a way to launch a task, continue doing other work, and later collect the result, which is the core idea of asynchronous programming. However, plain `Future` has notable limitations: it does not support chaining dependent tasks, combining multiple futures, registering callbacks, or manually completing a computation. These gaps are exactly what `CompletableFuture` fills by adding composition (`thenApply`, `thenCompose`), combination (`thenCombine`, `allOf`), and callbacks (`thenAccept`, `whenComplete`). `cancel(true)` attempts to interrupt the running task, but success depends on the task checking its interruption status. In modern code, `CompletableFuture` or `CompletableFuture`-style APIs are generally preferred, with plain `Future` used for simple "submit and wait" scenarios.

8. What is `ExecutorService`?
   - **Answer:** `ExecutorService` is an interface in `java.util.concurrent` that decouples task submission from thread management. Instead of creating a `Thread` per task, code submits `Runnable` or `Callable` instances, and the executor decides which pooled thread runs them and when. It extends `Executor`, adding lifecycle management: `shutdown()` (stops accepting new tasks and finishes queued ones), `shutdownNow()` (attempts to interrupt running tasks and returns queued ones), and `awaitTermination()` for graceful wait. The default implementations are the factory methods of `Executors` and the configurable `ThreadPoolExecutor`, whose key parameters are `corePoolSize`, `maximumPoolSize`, `keepAliveTime`, and the `workQueue`. Common factory methods are `newFixedThreadPool(n)`, `newCachedThreadPool()`, `newSingleThreadExecutor()`, and `newScheduledThreadPool(n)`. `invokeAll(...)` and `invokeAny(...)` support batch submission, returning `Future`s or the first successfully completed result respectively. Because an executor holds shared threads, it must be shut down explicitly, otherwise the JVM may not exit cleanly. `ExecutorService` is the standard way to build thread pools in Java and is the foundation that `CompletableFuture`, `@Async`, and virtual-thread executors build upon.

9. What is thread pool?
   - **Answer:** A thread pool is a collection of pre-created, reusable worker threads that execute submitted tasks. Instead of creating a new thread for every task, the pool reuses idle threads from its set, and tasks wait in a queue when all workers are busy. The core implementation is `ThreadPoolExecutor`, which maintains a set of worker threads, a work queue, and a rejection policy for tasks submitted when the pool is saturated. When a task is submitted, the pool first tries a core worker; if all cores are busy, the task is queued; if the queue fills and the pool is below `maximumPoolSize`, new threads are spawned; only when everything is exhausted does the `RejectedExecutionHandler` decide what happens (e.g., `AbortPolicy` by default, or `CallerRunsPolicy`). Common pools include fixed pools (constant size), cached pools (unbounded dynamic size with short keep-alive), single-thread executors (serialize tasks), and scheduled pools (delayed/periodic execution). Thread pools give predictable resource usage, amortize thread-creation cost, and provide a natural throttling mechanism. The trade-off is complexity: pool sizing, queue sizing, and rejection policies must all be tuned to the workload, and misconfiguration can cause queued-task pileups or thread exhaustion.

10. Why use thread pools?
    - **Answer:** Thread pools exist because creating a new OS thread is expensive: each thread requires a stack (default around 512KB–1MB of memory) plus OS handles, and thread creation/teardown adds latency to every task. A pool reuses a fixed set of threads, so the per-task overhead drops to queue enqueue/dequeue, which dramatically improves throughput for frequent short-lived tasks. Pools also bound resource usage: with a fixed number of threads, the number of concurrent operations is capped, protecting downstream resources such as database connections, sockets, and CPU. Without a cap, a spike in requests can spawn thousands of threads, exhaust memory, and crash the process—a failure mode pools prevent. Pools smooth out burst loads by queuing excess work instead of failing or spawning unbounded threads. They also provide a single place to configure, monitor, and shut down concurrency, and metrics like active count and queue depth can be exposed via JMX. In short, thread pools trade a little scheduling complexity for control, stability, and dramatically reduced thread-creation overhead.

11. Difference between fixed thread pool and cached thread pool.
    - **Answer:** `newFixedThreadPool(n)` creates a pool with a constant number of threads, using an unbounded `LinkedBlockingQueue`; once the `n` core threads are all busy, additional tasks wait in the queue. `newCachedThreadPool()` starts with zero threads and creates new threads as needed, reusing idle ones, with an unbounded maximum and a 60-second keep-alive, backed by a `SynchronousQueue` that hands tasks directly to workers. A fixed pool gives predictable, bounded concurrency, which suits workloads that must not exceed a certain parallelism, such as database-backed processing or calling rate-limited APIs. A cached pool adapts to load and shines for many short-lived, I/O-heavy tasks where threads are rarely all busy at once. The danger of a cached pool is that the thread count is unbounded: a sustained burst can spawn thousands of threads and exhaust memory, so it is risky in production for uncontrolled workloads. A fixed pool is safer for controlled access to shared resources but can build a large backlog if tasks are long and the queue is unbounded. The choice is essentially bounded-latency-and-bounded-parallelism (fixed) versus adaptive-resource-usage (cached).

12. What is scheduled executor service?
    - **Answer:** `ScheduledExecutorService` extends `ExecutorService` and can execute tasks after a delay or on a periodic schedule. Its key methods are `schedule(callable, delay, unit)` for one-shot delayed execution, `scheduleAtFixedRate(runnable, initialDelay, period, unit)` for fixed-interval execution, and `scheduleWithFixedDelay(runnable, initialDelay, delay, unit)` for fixed-delay-after-completion execution. The difference between the two periodic forms is subtle but important: `scheduleAtFixedRate` starts the next execution at a fixed clock offset regardless of how long the previous run took (unless the run overruns the period, in which case the schedule effectively slips), while `scheduleWithFixedDelay` always waits the fixed delay after the previous run finishes. The underlying `ScheduledThreadPoolExecutor` uses a `DelayedWorkQueue` that orders tasks by next execution time, and the core pool size is the number of threads that can run scheduled tasks concurrently. If a periodic task throws an exception, its future executions are silently cancelled, so tasks should catch their own exceptions. Spring's `@Scheduled` annotation is built on top of this service, abstracting the scheduling mechanics. It is the standard tool for background jobs such as cache refresh, heartbeat checks, and cleanup tasks.

13. What is synchronization?
    - **Answer:** Synchronization is the mechanism that restricts access to a critical section to one thread at a time, enforcing mutual exclusion on shared mutable state. In Java it is expressed with the `synchronized` keyword on methods or blocks, which uses an intrinsic (monitor) lock: a thread must acquire the monitor before entering, and any other thread attempting to enter blocks until it is released. Every Java object has an intrinsic lock, so `synchronized(obj)` uses the lock of the specified object, while `synchronized` instance methods lock `this` and static methods lock the class object. Synchronization provides two guarantees: mutual exclusion (no two threads execute the section concurrently) and visibility (a release of a lock happens-before a subsequent acquire of the same lock, so writes inside are visible to the next locker). Locks are reentrant, meaning the same thread can re-acquire a lock it already holds without deadlocking itself. Excessive synchronization harms throughput by serializing execution and can cause contention, so the goal is to keep critical sections small and to use higher-level alternatives where possible. Example:

    ```java
    private int counter;
    public synchronized void increment() {
        counter++; // read-modify-write is now atomic w.r.t. other threads
    }
    ```

14. What is race condition?
    - **Answer:** A race condition is a bug where the program's output depends on the unpredictable interleaving of threads, typically because multiple threads access shared mutable state without proper synchronization. The classic forms are read-modify-write (e.g., `count++` reads, increments, and writes, and two threads can interleave between these steps, losing an update) and check-then-act (e.g., checking whether a key exists and then inserting, where another thread inserts in between). The outcome is data-dependent: sometimes the code works, sometimes it loses updates or reads inconsistent values, which makes races notoriously hard to reproduce and debug. Races cannot be reliably detected by testing because they require a specific thread scheduling; they are best found through code review, stress testing with many threads, and tools like thread sanitizers or `-XX:+PrintConcurrentLocks`. Prevention strategies are eliminating shared mutable state (immutability or thread confinement), using atomic classes for single-variable operations, and using locks or `synchronized` to make compound operations atomic. Even with synchronized individual operations, a compound sequence of separate synchronized calls can still race, so the atomicity boundary must cover the whole check-then-act. The fundamental rule is that any shared, mutable field accessed by more than one thread requires a memory-visibility and atomicity strategy.

15. What is deadlock?
    - **Answer:** Deadlock is a situation where two or more threads are blocked forever, each holding a resource that another thread needs, forming a cycle of waiting. It requires all four Coffman conditions to hold simultaneously: mutual exclusion (resources are not shareable), hold-and-wait (a thread holds one resource while waiting for another), no preemption (resources cannot be forcibly taken away), and circular wait (thread A waits on B's resource while B waits on A's). A simple example is thread 1 holding lock A and waiting for lock B while thread 2 holds lock B and waiting for lock A; neither can proceed. Deadlock manifests as hung application threads; it is detected by taking thread dumps (`jstack <pid>`, `jcmd Thread.print`, or VisualVM) and looking for threads in `BLOCKED` state waiting on each other's monitors. Because detection happens after the fact, prevention is the primary strategy. Deadlock can also occur with database locks, connection pools, and other non-JVM resources, not just monitors. The classic counterexample to keep in mind is that `synchronized` is reentrant, so the same thread re-locking the same monitor does not cause deadlock; true deadlock always involves multiple resources and multiple threads.

16. How do you prevent deadlock?
    - **Answer:** The most reliable prevention is to eliminate the circular-wait condition by acquiring locks in a consistent, globally ordered sequence, so every thread that needs multiple locks takes them in the same order and releases them in reverse order. Breaking any single Coffman condition suffices: use `tryLock` with timeouts so a thread that cannot acquire all locks backs off and releases what it holds (breaking hold-and-wait and no preemption), avoid nested locks by consolidating them into one lock or a single concurrent data structure, and keep critical sections small. `ReentrantLock.tryLock(long timeout, TimeUnit unit)` is the practical tool here because it returns `false` instead of blocking forever, allowing the thread to release its current lock and retry. Lock-free data structures such as `ConcurrentHashMap` and atomic classes avoid locks entirely and therefore cannot deadlock on internal state. A bounded number of resources also helps: acquiring all resources atomically or failing fast reduces the chance of hold-and-wait. Practically, deadlock prevention is about discipline—documenting lock ordering, minimizing the number of locks, and preferring higher-level concurrency utilities. When deadlock is suspected, thread dumps reveal the cycle, and the fix is usually to reorder lock acquisition or add a timeout-based backoff.

17. What is livelock?
    - **Answer:** Livelock is a concurrency failure where threads are not blocked but are perpetually active without making progress, each reacting to the other's actions in a way that never completes. A common analogy is two people meeting in a hallway who both step aside in the same direction, repeatedly mirroring each other and never passing. In code, livelock often appears as two threads that detect a conflict, release their locks, wait, and then retry, only to collide again each time. Unlike deadlock, where threads are stuck in a `BLOCKED`/`WAITING` state doing nothing, livelocked threads consume CPU continuously because they keep executing their retry logic. A typical mitigation is to introduce randomness or asymmetric backoff so the retries desynchronize: one thread delays longer than the other, breaking the symmetric reaction loop. Exponential backoff with jitter in retry loops is the standard cure, because it makes repeated collision exponentially unlikely. Livelock is rarer than deadlock but is harder to notice because the threads look busy, so detecting it requires observing that throughput is zero while CPU usage stays high. The key contrast to remember: deadlock is "stuck and idle," livelock is "stuck but busy."

18. What is starvation?
    - **Answer:** Starvation is a concurrency problem where a thread is ready to run and is repeatedly allowed to wait while other threads are continuously granted the resources it needs, so it never makes progress. It happens when the scheduler or locking scheme is unfair: a low-priority thread, or a thread that just lost a race for a lock, can be perpetually outcompeted by higher-priority or repeatedly successful threads. `synchronized` blocks are unfair by default, so under heavy contention the same thread can be passed over indefinitely. `ReentrantLock` offers a fairness knob: `new ReentrantLock(true)` creates a fair lock that hands the lock to the longest-waiting thread, preventing indefinite starvation at the cost of some throughput. Thread pools can also cause starvation, for instance when all pool threads are blocked waiting on the results of other tasks also submitted to the same pool—a self-inflicted denial of service. Unlike deadlock, the starved thread is runnable and would eventually progress if given a chance, but the chance never comes. Solutions include fair locks, prioritizing tasks with deadlines, bounding queue sizes with rejection policies, and avoiding placing dependent tasks in the same pool. The mental model is that deadlock is a cycle of holds, while starvation is a queue that never reaches the end.

19. What is `volatile`?
    - **Answer:** The `volatile` keyword tells the JVM that a field may be accessed by multiple threads, so reads and writes to it must go through main memory rather than being cached in thread-local CPU caches. This provides a visibility guarantee: a write to a `volatile` field by one thread is visible to subsequent reads by other threads, and the write establishes a happens-before relationship with reads of that same field. Critically, `volatile` provides visibility only—it does not provide atomicity, so `volatile int count; count++;` is still unsafe because the increment is a read-modify-write sequence that can interleave across threads. The canonical use cases are status flags and the double-checked-locking singleton, where `volatile` ensures the partially-constructed object is not published before its fields are visible:

    ```java
    private static volatile Singleton instance;
    public static Singleton getInstance() {
        if (instance == null) {                 // first check (unsynchronized)
            synchronized (Singleton.class) {
                if (instance == null) {         // second check (synchronized)
                    instance = new Singleton(); // volatile publish
                }
            }
        }
        return instance;
    }
    ```

    A `volatile` boolean is a common shutdown signal read by worker loops while set by a different thread. It is cheaper than `synchronized` for simple flag publication because it does not require lock acquisition, but it must never be used where compound operations are needed. The rule of thumb: use `volatile` when the variable is written by one thread and read by others, and the writes are independent single assignments.

20. Difference between `volatile` and `synchronized`.
    - **Answer:** `volatile` provides only visibility guarantees; `synchronized` provides both mutual exclusion and visibility. `volatile` does not block threads or serialize access, so two threads can still interleave a read-modify-write on a `volatile` variable and lose updates, whereas `synchronized` forces one thread at a time through the critical section and makes compound operations atomic. Both establish happens-before relationships: a `volatile` write happens-before a subsequent read of the same variable, and a lock release happens-before the next acquire of the same lock. `volatile` is appropriate for single-assignment flags and for publishing immutable state; `synchronized` is required for compound operations like `count++`, check-then-act, and any sequence of steps that must appear atomic. `synchronized` carries a performance and contention cost and can block threads, while `volatile` has negligible overhead but offers no exclusion. Atomic classes like `AtomicInteger` sit between them, providing lock-free atomicity for single variables by combining CAS (compare-and-swap) with volatile-style memory effects. The memory-barrier mechanics differ: `volatile` inserts store/load barriers around the access, while `synchronized` additionally acquires and releases a monitor. In practice, the choice is guided by the operation: single-variable flag → `volatile`; multi-step critical section → `synchronized`/lock; single counter → atomic class.

21. What is atomic variable?
    - **Answer:** Atomic variables are classes in `java.util.concurrent.atomic`—such as `AtomicInteger`, `AtomicLong`, `AtomicBoolean`, and `AtomicReference`—that provide thread-safe, lock-free operations on single values. They rely on CAS (compare-and-swap): the CPU instruction atomically compares a memory location to an expected value and, only if it matches, writes a new value, retrying in a loop until it succeeds. Because CAS is a hardware instruction, these classes avoid heavyweight lock acquisition and outperform `synchronized` for fine-grained counters under moderate contention. They also provide the same visibility guarantees as `volatile` (each update happens-before subsequent reads), so they are fully safe for cross-thread sharing. Beyond `incrementAndGet()`, they support `getAndSet()`, `compareAndSet(expected, newValue)`, `updateAndGet(...)`, and `accumulateAndGet(...)`. For very high write contention, `LongAdder` and `LongAccumulator` use cell striping to distribute updates across multiple counters, trading lower read accuracy for dramatically higher write throughput. Atomic classes are ideal for counters, sequence generators, and lock-free state transitions, but they only make a *single* operation atomic—they do not make a multi-step sequence safe unless the whole sequence is built from one CAS. The `ABA` problem can affect `AtomicReference` when a value changes A→B→A, which is why `AtomicStampedReference` and `AtomicMarkableReference` exist.

22. What is `AtomicInteger`?
    - **Answer:** `AtomicInteger` is an atomic variable that wraps an `int` and provides thread-safe, lock-free operations such as `incrementAndGet()`, `getAndIncrement()`, `addAndGet(int delta)`, `getAndSet(int)`, and `compareAndSet(int expect, int update)`. Under the hood, `incrementAndGet()` performs a CAS loop: it reads the current value, computes the new value, and tries `compareAndSet`; if another thread changed the value in between, it retries with the fresh value until it succeeds. This makes it safe for concurrent counters where `int i++` would lose updates. It is the right replacement for a shared `int` counter that many threads increment, such as counting processed records, requests, or failures. The visibility guarantee matches `volatile`: `incrementAndGet()` writes with release semantics and subsequent reads acquire the latest value. For custom transformations, `updateAndGet(IntUnaryOperator)` atomically applies a function to the current value, and `getAndUpdate` returns the previous value. Because it is lock-free, it avoids thread suspension and performs well under moderate contention; under extreme contention, `LongAdder` is a better fit. `compareAndSet` is also useful for implementing simple state machines, like idempotent single-winner transitions. In short, `AtomicInteger` is the standard tool for thread-safe integer counting without `synchronized`.

23. What is lock?
    - **Answer:** A lock is a synchronization construct that enforces exclusive access to a shared resource, and in Java the `Lock` interface (`java.util.concurrent.locks.Lock`) generalizes the intrinsic monitor with more flexible operations. The main implementation is `ReentrantLock`, whose methods are `lock()`, `unlock()`, `tryLock()`, `tryLock(timeout, unit)`, and `lockInterruptibly()`. Unlike `synchronized`, which releases the monitor automatically, a `Lock` must always be released manually, so `unlock()` is placed in a `finally` block to guarantee release even on exception. `tryLock` with a timeout is the killer feature for deadlock avoidance: a thread can attempt to acquire a lock for a bounded time and back off if it fails instead of blocking forever. `lockInterruptibly()` allows a waiting thread to be interrupted and escape the wait. `Lock` also supports fairness (`new ReentrantLock(true)` gives locks to the longest-waiting thread, preventing starvation) and condition variables via `newCondition()`, which provides `await()`/`signal()` analogous to `wait()`/`notify()` but with multiple, separately named condition queues per lock. The trade-off is manual discipline: forgetting to unlock leaks the lock and causes subtle deadlocks. Example:

    ```java
    Lock lock = new ReentrantLock();
    lock.lock();
    try {
        // critical section
    } finally {
        lock.unlock();
    }
    ```

24. Difference between intrinsic lock and `ReentrantLock`.
    - **Answer:** The intrinsic lock (the monitor behind `synchronized`) is simple, auto-released, and reentrant, but its acquisition is unconditional and non-interruptible, it is unfair by default, and it exposes only `wait()`/`notify()` for coordination. `ReentrantLock` offers explicit `lock()`/`unlock()`, plus `tryLock()` with timeouts, `lockInterruptibly()`, a fairness option, and multiple `Condition` objects. `synchronized` compiles to `monitorenter`/`monitorexit` bytecodes and requires no try/finally ceremony, while `ReentrantLock` requires the `finally { unlock(); }` pattern, making it easy to leak if misused. Internally, `ReentrantLock` is built on `AbstractQueuedSynchronizer` (AQS), a FIFO wait-queue framework. Historically `synchronized` was slower, but since Java 6 the JVM applies biased locking and adaptive spin tuning, so for most simple cases the performance difference is negligible and `synchronized` is preferred for readability. `ReentrantLock` should be chosen when timed lock acquisition, interruptible waiting, fairness, or multiple conditions are needed. Both are reentrant: the same thread may acquire them again without self-deadlocking. The general guidance is to prefer `synchronized` by default and reach for `ReentrantLock` when its specific capabilities justify the extra ceremony.

25. What is read-write lock?
    - **Answer:** A `ReadWriteLock` maintains two separate locks—a shared read lock and an exclusive write lock—so that multiple readers can hold the read lock concurrently, while a writer must hold the write lock exclusively. Reads do not block other reads (that is the point), but a write blocks all readers and other writers, and a writer blocks while any reader holds the lock. The canonical implementation is `ReentrantReadWriteLock`, used through `readLock()` and `writeLock()`:

    ```java
    ReadWriteLock rwl = new ReentrantReadWriteLock();
    Object get(String key) {
        rwl.readLock().lock();
        try { return cache.get(key); } finally { rwl.readLock().unlock(); }
    }
    void put(String key, Object v) {
        rwl.writeLock().lock();
        try { cache.put(key, v); } finally { rwl.writeLock().unlock(); }
    }
    ```

    It pays off when reads vastly outnumber writes, such as a configuration map that is loaded once and read hot. The danger is writer starvation: if readers continuously arrive, a waiting writer may never get the lock, so `ReentrantReadWriteLock` offers a `fair` mode that gives the longest-waiting thread priority. Read-write locks are more complex than plain locks and have higher lock-acquisition overhead, so they only win when the read:write ratio is high and critical sections are non-trivial. Often a `ConcurrentHashMap` is a better fit for read-heavy data because it needs no locking at all for reads. A read-write lock is the right answer when the data is mutable, reads dominate, and a copy-on-write structure would be too costly.

26. What is semaphore?
    - **Answer:** A `Semaphore` maintains a set of permits, and threads `acquire()` a permit before entering a section and `release()` it afterwards, effectively limiting how many threads can use a resource concurrently. It is a counting synchronization primitive: a semaphore with N permits allows at most N concurrent acquisitions, and a binary semaphore (permits = 1) behaves like a mutual-exclusion gate. Unlike locks, semaphores have no ownership—any thread can release a permit it did not acquire, so they are ideal for guarding shared external resources such as database connections, sockets, or a license-limited API. `acquire()` blocks when no permits remain; `tryAcquire(timeout)` gives a bounded-wait alternative, and `Semaphore` supports fair acquisition via a fairness flag. Because the permit count is a hard ceiling, semaphores are the classic tool for rate limiting and connection-pooling control. The asymmetry between locks and semaphores is important: a lock is owned by the thread that acquired it, while a semaphore is a pure counter and cannot be used for monitor-style `wait`/`notify` coordination. Example:

    ```java
    Semaphore sem = new Semaphore(10); // allow 10 concurrent operations
    sem.acquire();
    try { performLimitedWork(); } finally { sem.release(); }
    ```

    When used for pooling, the pattern is acquire the permit, take a resource, and always release the permit in `finally`.

27. What is countdown latch?
    - **Answer:** `CountDownLatch` is a synchronization aid that lets one or more threads wait until a set of operations completes. It is initialized with a count, threads decrement it via `countDown()` as they finish work, and waiting threads block in `await()` until the count reaches zero, at which point all waiters are released together. It is single-use: once the count hits zero, the latch cannot be reset, so it fits a one-time rendezvous rather than repeated barriers. The primary use case is "start all workers, wait for all to finish, then aggregate"—for example, launching N parallel validation tasks and waiting for them all before proceeding. `await(timeout, unit)` bounds the wait so callers are not stuck forever. The latch is a good tool for testing concurrency too, coordinating threads to start simultaneously and then completing. The key contrast with `CyclicBarrier`: a latch is a countdown others wait on, while a barrier is a point all participating threads arrive at together and then proceed as a group; and a latch cannot be reused. Example:

    ```java
    CountDownLatch latch = new CountDownLatch(3);
    IntStream.range(0, 3).forEach(i -> executor.submit(() -> {
        doWork(i);
        latch.countDown();
    }));
    latch.await(); // blocks until all three tasks have counted down
    ```

28. What is cyclic barrier?
    - **Answer:** `CyclicBarrier` makes a fixed number of threads wait for each other at a common rendezvous point before any of them proceeds; when all N parties have arrived, the barrier "trips" and all threads continue together. Its key advantage over `CountDownLatch` is reusability—after tripping, the barrier resets and can synchronize the next round, making it ideal for phases of batch processing where workers must realign after each batch. A `CyclicBarrier` can also take a barrier action, a `Runnable` executed by the last arriving thread when the barrier trips, commonly used for one-time aggregation or phase-complete notifications. The thread that trips the barrier runs the action, and other threads wait until it finishes, so the action must be quick. Arrival and waiting are explicit: `await()` declares arrival and blocks, `await(timeout, unit)` adds a deadline, and `reset()` forces a fresh cycle (causing waiters to throw `BrokenBarrierException`). If any waiting thread is interrupted or the barrier is reset, other waiters get `BrokenBarrierException`, so handlers must account for barrier breakage. Example:

    ```java
    int parties = 4;
    CyclicBarrier barrier = new CyclicBarrier(parties, () -> System.out.println("round complete"));
    for (int i = 0; i < parties; i++) executor.submit(() -> {
        processChunk();
        barrier.await(); // all parties must arrive before any continues
    });
    ```

    The mental distinction: a `CountDownLatch` waits for an event (count reaching zero); a `CyclicBarrier` waits for other threads to catch up, and it resets for the next round.

29. What is concurrent collection?
    - **Answer:** Concurrent collections are thread-safe data structures in `java.util.concurrent` designed for high-concurrency access without external synchronization, including `ConcurrentHashMap`, `CopyOnWriteArrayList`, `ConcurrentLinkedQueue`, and the `BlockingQueue` family. `ConcurrentHashMap` is the workhorse: reads are generally lock-free, and writes use fine-grained synchronization (CAS plus per-bin locks since Java 8), so it scales far better than a `Hashtable` or a `Collections.synchronizedMap` that serialize every access. `CopyOnWriteArrayList` maintains a snapshot array and copies it on every modification, so readers never block and see a consistent snapshot—it is ideal for read-dominated lists such as listener registries. `BlockingQueue` implementations (`ArrayBlockingQueue`, `LinkedBlockingQueue`) add `put`/`take` that block when the queue is full or empty, which directly supports the producer-consumer pattern. Iterators of concurrent collections are weakly consistent: they reflect the state at traversal time and do not throw `ConcurrentModificationException`, unlike fail-fast iterators of legacy collections. Concurrent collections also use weaker bounds: size is approximate, and individual compound operations like `putIfAbsent`, `compute`, and `merge` provide atomic per-entry operations. The choice is workload-driven: high read/write map access → `ConcurrentHashMap`; read-mostly lists → `CopyOnWriteArrayList`; stream coordination → `BlockingQueue`. These collections eliminate whole categories of locking bugs when they match the access pattern.

30. What is `ThreadLocal`?
    - **Answer:** `ThreadLocal` provides variables that each thread owns independently: each thread sees its own instance, initialized lazily by the `initialValue()` supplier, so threads never share or overwrite each other's values. Internally, each `Thread` holds a `ThreadLocalMap` keyed by weak references to the `ThreadLocal` objects, so values are per-thread storage attached to the thread itself. It is the standard mechanism for request-scoped data such as a correlation ID, the current user, or a transaction context, because each request thread naturally carries its own value without passing parameters through every method. Two hazards require care. First, memory leaks: in a thread pool, threads are long-lived and reused, and the `ThreadLocal` entry stays in the map until removed, so a value held by a pooled thread is never GC'd and leaks; the fix is always calling `remove()` (e.g., in a `finally` block or `afterCompletion`). Second, propagation: `ThreadLocal` values are invisible to other threads by design, so async execution on a different thread loses the context unless it is explicitly copied. Usage pattern:

    ```java
    ThreadLocal<String> traceId = new ThreadLocal<>();
    try {
        traceId.set(UUID.randomUUID().toString());
        doWork(); // reads traceId.get() within this thread
    } finally {
        traceId.remove(); // mandatory with pooled threads
    }
    ```

    `ThreadLocal` is for state that is logically shared across a call chain but must not be shared across threads.

31. How does Spring Security use `ThreadLocal`?
    - **Answer:** Spring Security stores the authenticated principal in `SecurityContextHolder`, which by default keeps the `SecurityContext` in a `ThreadLocal`. This means that anywhere in the same request thread—interceptors, filters, controllers, service methods—code can call `SecurityContextHolder.getContext().getAuthentication()` and retrieve the current user without threading the object through every method signature. The authentication filter populates the context once at the start of the request, and the chain cleans it up with `SecurityContextHolder.clearContext()` when the request completes, preventing leakage across pooled threads. The default strategy is `MODE_THREADLOCAL`, which gives one context per request thread. Because the context lives in a `ThreadLocal`, it is automatically lost when work is handed to another thread: an `@Async` method, a task submitted to an executor, or a reactive pipeline running on a different thread will see an empty context. For those cases Spring offers `MODE_INHERITABLETHREADLOCAL` (propagates to directly spawned child threads only, not pooled threads), or explicit propagation by passing the authentication as a parameter or copying the context in a `TaskDecorator`. `SecurityContextHolder` also supports `MODE_GLOBAL` (a single shared context), which is rarely used. The design principle is that security state rides along with the request thread, which is exactly the strength—and the async weakness—of `ThreadLocal`.

32. What happens to `ThreadLocal` in async execution?
    - **Answer:** `ThreadLocal` values do not automatically travel to another thread, so when a task is executed asynchronously—via `@Async`, `ExecutorService`, `CompletableFuture.supplyAsync`, or `ForkJoinPool`—the executing thread does not inherit the caller's `ThreadLocal` values. This breaks request-scoped data like security context, tracing/MDC IDs, or transaction context, because the async thread sees `null` or a default. `InheritableThreadLocal` propagates values to directly created child threads at the moment of creation, but it does not help with thread pools, where the worker thread already exists and is reused, so its inherited values are stale or absent. The standard solutions are manual propagation: capture the values before the async boundary and restore them in the worker, commonly implemented with a `TaskDecorator` for `ThreadPoolTaskExecutor` (Spring's hook that wraps every submitted `Runnable`) or with a wrapper `Runnable` that sets the `ThreadLocal` before running and removes it afterwards. Without restoration, downstream code silently behaves differently—no user, no trace ID, or wrong context—which makes these bugs subtle. Because pooled threads are reused, whatever is set in one task persists for the next unless explicitly removed, so the wrapper must always `remove()` in `finally`. The general rule is that `ThreadLocal` context is thread-bound by design, so every thread boundary needs an explicit copy-in/copy-out, and this is precisely what context propagation libraries automate.

33. What is context propagation?
    - **Answer:** Context propagation is the technique of carrying thread-bound state—security context, tracing/correlation IDs, MDC (Mapped Diagnostic Context) for logging, transaction metadata—across asynchronous boundaries and across services, so that the logical request remains identifiable end to end. The core challenge is that `ThreadLocal` is the primary storage for such context in frameworks like Spring Security and SLF4J MDC, but it is inherently tied to the executing thread, and async execution moves work to other threads. Thread pools make this worse because worker threads are reused, so context must be explicitly captured at submission time, injected into the worker, and cleaned up afterwards. The mechanical solution is a wrapper that snapshots context before submitting and restores it inside the task:

    ```java
    public Runnable withContext(Runnable task) {
        Map<String, String> snapshot = MDC.getCopyOfContextMap();
        return () -> {
            MDC.setContextMap(snapshot);
            try { task.run(); } finally { MDC.clear(); }
        };
    }
    ```

    Frameworks standardize this: Spring's `TaskDecorator` wraps tasks for `ThreadPoolTaskExecutor`, and in reactive code context travels via a `Context` carried with the subscription rather than thread state. OpenTelemetry generalizes the idea with a `Context` object that is propagated across thread boundaries and across HTTP calls (W3C `traceparent` headers), while Micrometer's tracing integrates with executor instrumentation. The payoff is coherent logs and traces: the same correlation ID appears in every log line a request produces, regardless of which thread or process ran it.

34. What is thread safety?
    - **Answer:** A class is thread-safe if it behaves correctly when accessed concurrently by multiple threads, meaning all its invariants hold and no data races or lost updates occur regardless of thread interleaving. The mechanisms to achieve it fall into four strategies. The first is confinement: don't share state at all, keeping objects within a single thread (e.g., `ThreadLocal` or method-local variables). The second is immutability: if the state cannot change after publication, sharing is free, because reads of an immutable, safely-published object need no synchronization. The third is synchronization: protecting critical sections with `synchronized`, locks, or atomic classes so compound operations are atomic. The fourth is using thread-safe data structures such as `ConcurrentHashMap` or `CopyOnWriteArrayList` that handle their own synchronization. Reasoning about thread safety requires the happens-before model: two accesses race unless one is ordered before the other by a synchronization action (volatile, lock, start/join, etc.). Thread safety is an API contract too—the class should document what guarantees it makes (immutable, thread-safe, conditionally thread-safe) so callers know whether extra synchronization is their responsibility. It is validated by stress tests and tools, but the real assurance comes from the design: identifiable shared mutable state must have a defined protection strategy.

35. How do you make a class thread-safe?
    - **Answer:** The first step is identifying shared mutable state: fields that are written by more than one thread. Then choose a protection strategy based on the nature of the state and the expected contention. For state that never changes after construction, declare fields `final`, initialize them in the constructor, and publish the object safely (e.g., via a `volatile` reference or a concurrent collection)—that alone makes it thread-safe with zero locking. For single counters or flags, use atomic classes like `AtomicInteger`/`AtomicBoolean`, which provide lock-free atomicity. For multi-step operations or invariants spanning several fields, guard them with `synchronized` blocks (preferably at the smallest scope, on a dedicated private lock object) or `ReentrantLock` when timed/interruptible acquisition is needed. When possible, delegate to concurrent collections—a `ConcurrentHashMap` for shared maps, `CopyOnWriteArrayList` for read-heavy lists—so the collection handles the synchronization. Collections of accessors is the key discipline: every path that reads or writes the shared state must go through the same protection, including iterators and compound read-modify-write sequences. Helper methods that return mutable state should return copies or unmodifiable views rather than internal references. Finally, document the guarantee and validate with concurrency tests that exercise interleavings under load. A rule of thumb for choosing: immutable > confined > concurrent collections > atomic classes > locks, in ascending cost and complexity.

