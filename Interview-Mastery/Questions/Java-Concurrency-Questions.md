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
   - **Answer:** A thread is the smallest unit of execution within a process. Each thread has its own stack but shares heap memory. In my Kafka consumers, each listener thread processes messages concurrently, sharing access to caches while maintaining independent execution.
   - **If asked more:** I can explain thread lifecycle (NEW, RUNNABLE, BLOCKED, WAITING, TIMED_WAITING, TERMINATED), how threads map to OS threads in platform threads vs virtual threads, and thread stack size tuning.
2. What is process vs thread?
   - **Answer:** A process is an independent program with its own memory space, while threads share memory within a process. Communication between processes requires IPC (pipes, sockets), but threads can communicate via shared objects, which is why I use threads for concurrent data processing.
   - **If asked more:** I can explain context switching costs (threads are lighter), how the JVM runs as a single process with multiple threads, and why multithreading is preferred over multiprocessing for backend services.
3. How do you create a thread in Java?
   - **Answer:** I can extend Thread, implement Runnable, or use Callable with ExecutorService. In my projects, I never extend Thread directly — I use Runnable with thread pools or CompletableFuture.supplyAsync for cleaner async execution.
   - **If asked more:** I can explain the different creation approaches, why implementing Runnable is better than extending Thread (flexibility, no single-inheritance limitation), and how lambda syntax simplifies new Thread(() -> work()).start().
4. Difference between extending `Thread` and implementing `Runnable`.
   - **Answer:** Extending Thread locks you into inheritance; implementing Runnable leaves the class free to extend other classes. In my Spring services, I always implement Runnable or use lambdas, keeping the class available for Spring bean proxying.
   - **If asked more:** I can explain how Thread itself implements Runnable, why extending Thread is considered an anti-pattern, the strategy pattern benefit of decoupling task from execution, and how ExecutorService accepts Runnable.
5. What is `Callable`?
   - **Answer:** Callable is like Runnable but can return a result and throw checked exceptions. In my inventory project, I used Callable with ExecutorService to submit validation tasks that returned counts of invalid serial records.
   - **If asked more:** I can explain the call() method vs run(), how Future wraps the result, why Callable is preferred for tasks needing return values, and how CompletableFuture.supplyAsync internally uses Callable.
6. Difference between `Runnable` and `Callable`.
   - **Answer:** Runnable returns void and cannot throw checked exceptions; Callable returns a value and can throw Exception. I use Runnable for fire-and-forget tasks like logging and Callable when I need a result from async computation.
   - **If asked more:** I can explain how ExecutorService.submit(Runnable) returns Future<?> vs submit(Callable) returns Future<T>, how to get null from Runnable futures, and lambda syntax (() -> result vs () -> { run(); }).
7. What is `Future`?
   - **Answer:** Future represents the result of an async computation with methods like get(), isDone(), and cancel(). In my projects, I moved to CompletableFuture for chaining, but Future was my earlier approach for getting results from thread pool tasks.
   - **If asked more:** I can explain blocking get() vs timeout version, how cancel works with mayInterruptIfRunning, limitations of Future (no chaining, no manual completion, no callbacks), and how CompletableFuture improves on it.
8. What is `ExecutorService`?
   - **Answer:** ExecutorService manages a pool of threads and decouples task submission from execution. In my CDMS project, I configured a custom ExecutorService for parallel report generation, submitting tasks and collecting results via invokeAll.
   - **If asked more:** I can explain ThreadPoolExecutor parameters (corePoolSize, maxPoolSize, keepAliveTime, workQueue), different factory methods (newFixedThreadPool, newCachedThreadPool), and shutdown vs shutdownNow.
9. What is thread pool?
   - **Answer:** A thread pool reuses a fixed number of threads to execute tasks, avoiding the overhead of creating new threads per request. In my Spring Boot async configuration, I defined a thread pool for @Async methods to handle concurrent inventory processing.
   - **If asked more:** I can explain how thread pools improve performance (reuse, control resource usage), the worker thread pattern, common pool types (fixed, cached, scheduled, single), and how rejecting tasks with RejectedExecutionHandler works.
10. Why use thread pools?
   - **Answer:** Thread pools reduce overhead from thread creation, control concurrency limits, and improve system stability. In my inventory project, using a bounded thread pool prevented runaway threads during peak validation from overwhelming the database connection pool.
   - **If asked more:** I can explain the cost of creating threads (memory, OS handles), how thread pools smooth out burst loads, thread starvation vs resource exhaustion, and monitoring thread pool metrics via JMX.
11. Difference between fixed thread pool and cached thread pool.
   - **Answer:** Fixed thread pool keeps a constant number of threads; cached thread pool creates new threads as needed and reuses idle ones. I use fixed pools for controlled database access and avoid cached pools in production due to unbounded thread creation risk.
   - **If asked more:** I can explain the work queue difference (fixed uses LinkedBlockingQueue, cached uses SynchronousQueue), when cached is useful (many short-lived tasks), and why cached pools can crash a system under load.
12. What is scheduled executor service?
   - **Answer:** ScheduledExecutorService can run tasks with delay or periodically. In my projects, I used it for scheduled tasks like refreshing partner cache every 5 minutes, with scheduleAtFixedRate for consistent interval execution.
   - **If asked more:** I can explain schedule vs scheduleAtFixedRate vs scheduleWithFixedDelay, how missed executions are handled, ScheduledThreadPoolExecutor internals (DelayedWorkQueue), and Spring's @Scheduled abstraction over it.
13. What is synchronization?
   - **Answer:** Synchronization prevents multiple threads from executing critical sections simultaneously, ensuring thread safety via mutual exclusion. In my concurrent cache access, synchronized methods on shared maps prevented race conditions during inventory updates.
   - **If asked more:** I can explain intrinsic locks, the synchronized keyword on methods vs blocks, reentrancy, how synchronization creates happens-before guarantees, and performance costs of excessive synchronization.
14. What is race condition?
   - **Answer:** A race condition occurs when multiple threads access shared data simultaneously and the outcome depends on thread scheduling order. In my Kafka consumer, a race condition on shared counter caused incorrect record counts before I added synchronization.
   - **If asked more:** I can explain check-then-act and read-modify-write patterns, how to detect race conditions via code review, why atomic classes and locks prevent them, and how ConcurrentHashMap internal design avoids races.
15. What is deadlock?
   - **Answer:** Deadlock occurs when two or more threads each hold a lock and wait for the other to release, causing indefinite blocking. While I never encountered it in my projects, I design my locking order consistently (same order for all threads) to prevent it.
   - **If asked more:** I can explain the four necessary conditions (mutual exclusion, hold-and-wait, no preemption, circular wait), how to detect with jstack or visualvm, and prevention through lock ordering, timeouts, and tryLock.
16. How do you prevent deadlock?
   - **Answer:** I prevent deadlock by acquiring locks in a consistent global order, using tryLock with timeouts instead of intrinsic locks, and keeping critical sections as small as possible. In my cache design, a single ConcurrentHashMap eliminated the need for multiple locks.
   - **If asked more:** I can explain lock ordering strategy, how ReentrantLock.tryLock(time, unit) avoids indefinite blocking, deadlock detection via thread dumps, and how lock-free data structures bypass the problem entirely.
17. What is livelock?
   - **Answer:** Livelock is when threads keep changing state in response to each other without making progress, like two people stepping aside in the same direction repeatedly. I avoid it by adding random delays or backoff in retry logic.
   - **If asked more:** I can compare livelock with deadlock (blocked vs active-but-no-progress), real examples like two threads releasing and retrying locks in a loop, and how exponential backoff helps with coordination.
18. What is starvation?
   - **Answer:** Starvation occurs when a thread is perpetually denied access to resources because other threads keep getting priority. I ensure fairness in my thread pools by using a fair lock (new ReentrantLock(true)) when multiple threads contend for shared resources.
   - **If asked more:** I can explain how low-priority threads can starve, how synchronized blocks are unfair by default, how ReentrantLock allows fairness parameter, and how thread pool sizing affects starvation risk.
19. What is `volatile`?
   - **Answer:** volatile ensures that reads and writes to a variable are directly from main memory, not thread-local cache, providing visibility guarantees. In my Kafka consumer, I used a volatile boolean flag for graceful shutdown signal across threads.
   - **If asked more:** I can explain the happens-before relationship with volatile, why volatile does not provide atomicity (increment is still three operations), how it differs from synchronized, and common use cases (flags, double-checked locking).
20. Difference between `volatile` and `synchronized`.
   - **Answer:** volatile guarantees visibility only, synchronized guarantees both visibility and atomicity. I use volatile for simple flag variables and synchronized (or locks) for compound operations like check-then-act sequences.
   - **If asked more:** I can explain how synchronized provides mutual exclusion while volatile does not, memory barrier effects, how atomic classes like AtomicBoolean combine both, and the classic double-checked locking pattern with volatile.
21. What is atomic variable?
   - **Answer:** Atomic variables (AtomicInteger, AtomicLong, AtomicReference) support lock-free, thread-safe operations using CAS (Compare-And-Swap). In my inventory project, I used AtomicLong for tracking total processed records across consumer threads without locks.
   - **If asked more:** I can explain how CAS works (compareAndSet), why atomic variables are faster than locks for simple counters, the ABA problem with AtomicReference, and how LongAdder improves on AtomicLong for high-contention writes.
22. What is `AtomicInteger`?
   - **Answer:** AtomicInteger provides atomic operations like incrementAndGet, addAndGet, and compareAndSet for int values. In my cold-chain project, I used it to count total sensor events processed across multiple Kafka partitions without synchronized blocks.
   - **If asked more:** I can explain how incrementAndGet uses CAS in a loop, why it is thread-safe, how to use updateAndGet for custom transformations, and performance comparison with synchronized int.
23. What is lock?
   - **Answer:** A lock is a synchronization mechanism that provides more flexibility than synchronized blocks. In Java, ReentrantLock offers tryLock, timed lock, and interruptible lock acquisition. I prefer synchronized for simple cases and ReentrantLock for advanced scenarios.
   - **If asked more:** I can explain the Lock interface (lock, unlock, tryLock, lockInterruptibly), how lock() must be paired with unlock() in finally, condition variables with await/signal, and ReentrantLock fairness.
24. Difference between intrinsic lock and `ReentrantLock`.
   - **Answer:** Intrinsic locks (synchronized) are simpler and automatically released; ReentrantLock offers tryLock, fairness, and condition support. I use synchronized by default and ReentrantLock only when I need timed lock attempts or interruptible locking.
   - **If asked more:** I can explain how synchronized uses monitor enter/exit bytecodes, ReentrantLock uses AbstractQueuedSynchronizer, performance comparison (synchronized optimized in recent Java), and how tryLock helps prevent deadlocks.
25. What is read-write lock?
   - **Answer:** ReadWriteLock allows multiple readers simultaneously but exclusive write access. In my hot-cache scenario, I could use ReentrantReadWriteLock for a configuration map where reads are frequent and writes are rare, improving throughput.
   - **If asked more:** I can explain the readLock/writeLock split, how multiple readers do not block each other, write starvation with many readers, and when ReadWriteLock is beneficial vs when a simple ConcurrentHashMap suffices.
26. What is semaphore?
   - **Answer:** Semaphore controls access to a limited resource by maintaining a permit count. In my cold-chain project, I would use Semaphore to limit concurrent database connections during bulk sensor data writes, ensuring the connection pool is not exhausted.
   - **If asked more:** I can explain acquire() and release() methods, counting vs binary semaphore, how Semaphore differs from locks (no ownership), fair vs unfair acquisition, and practical use cases like rate limiting.
27. What is countdown latch?
   - **Answer:** CountDownLatch lets one or more threads wait until a set of operations complete. I could use it in my inventory system to wait for multiple parallel validation tasks to finish before proceeding to the aggregation step.
   - **If asked more:** I can explain how await() blocks until count reaches zero, how countDown() decrements, why CountDownLatch is single-use (cannot reset), and CyclicBarrier vs CountDownLatch differences.
28. What is cyclic barrier?
   - **Answer:** CyclicBarrier lets multiple threads wait for each other at a common point before proceeding. Unlike CountDownLatch, it can be reused. I would use it in batch processing where N worker threads must sync after each batch of records.
   - **If asked more:** I can explain barrier action (Runnable that runs when barrier trips), party count, how reset() makes it reusable, difference from CountDownLatch (threads wait for each other vs threads wait for countdown), and timeout usage.
29. What is concurrent collection?
   - **Answer:** Concurrent collections are thread-safe collections optimized for multi-threaded access, like ConcurrentHashMap, CopyOnWriteArrayList, and BlockingQueue. I use ConcurrentHashMap extensively in my Kafka consumers for shared caches without external synchronization.
   - **If asked more:** I can explain how ConcurrentHashMap uses CAS and synchronized internally, why CopyOnWriteArrayList is for read-heavy workloads, how BlockingQueue coordinates producers/consumers, and when to choose concurrent over synchronized wrappers.
30. What is `ThreadLocal`?
   - **Answer:** ThreadLocal provides per-thread variable instances. In Spring Boot, each HTTP request runs on a thread and ThreadLocal stores user context. I used ThreadLocal for request-scoped logging correlation IDs in my APIs.
   - **If asked more:** I can explain how ThreadLocalMap works internally (weak reference to thread), memory leak risks with thread pools (threads are reused, entries not cleaned up), and why remove() should always be called in async or pool scenarios.
31. How does Spring Security use `ThreadLocal`?
   - **Answer:** Spring Security stores the authenticated user in SecurityContextHolder using ThreadLocal by default, so every method in the same thread can access the current user's authentication. In my controllers, I get the logged-in user via SecurityContextHolder.getContext().
   - **If asked more:** I can explain MODE_THREADLOCAL (default per request thread), how it breaks in async execution, how to propagate context to child threads with MODE_INHERITABLETHREADLOCAL, and custom context propagation strategies.
32. What happens to `ThreadLocal` in async execution?
   - **Answer:** ThreadLocal values are not automatically propagated to async threads because the async task runs on a different thread. In my @Async methods, I had to manually pass context or use Spring's TaskDecorator to copy ThreadLocal values to the executing thread.
   - **If asked more:** I can explain how InheritableThreadLocal works for child threads but not thread pools, how to implement AsyncConfigurer with ThreadPoolTaskExecutor and TaskDecorator, and how MDC context is similarly lost in async.
33. What is context propagation?
   - **Answer:** Context propagation transfers thread-local state (security, tracing, MDC) across thread boundaries. In my Kafka consumers, I propagated MDC context from the listener thread to async processing threads so correlation IDs remained consistent across logs.
   - **If asked more:** I can explain challenges in thread pools, how libraries like Micrometer handle propagation, Spring Cloud Sleuth's approach with TraceRunnable, and OpenTelemetry's context propagation across services.
34. What is thread safety?
   - **Answer:** A class is thread-safe when it behaves correctly under concurrent access. In my design, I achieve thread safety through immutable objects, synchronized blocks, atomic variables, or thread-safe collections like ConcurrentHashMap.
   - **If asked more:** I can explain the four strategies: confinement (no sharing), immutability, synchronization, and thread-safe data structures, how to reason about thread safety with happens-before, and how to test with stress tests.
35. How do you make a class thread-safe?
   - **Answer:** I make a class thread-safe by using immutable fields (final), atomic classes for counters, synchronized blocks for critical sections, or delegating to ConcurrentHashMap/CopyOnWriteArrayList. In my cache service, I used ConcurrentHashMap with AtomicLong counters.
   - **If asked more:** I can explain identifying shared mutable state as the first step, choosing the right approach based on contention level (immutable > atomic > lock > synchronized), documenting thread-safety guarantees, and testing with multiple threads.

