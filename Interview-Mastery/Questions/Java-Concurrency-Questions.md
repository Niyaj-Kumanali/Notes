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
   - **Answer:**
      - A thread is the smallest unit of execution within a process
      - Each thread has its own stack but shares heap memory
      - For example, Kafka consumer listener threads process messages concurrently, sharing access to caches while maintaining independent execution
      - Thread lifecycle: NEW → RUNNABLE → (BLOCKED/WAITING/TIMED_WAITING) → TERMINATED
      - Platform threads map 1:1 to OS threads; virtual threads (Java 21+) use multiplexing over fewer carrier threads, enabling millions of concurrent threads
      - Thread stack size can be tuned via -Xss flag or new Thread(group, target, name, stackSize); smaller stacks reduce memory per thread but risk StackOverflowError
2. What is process vs thread?
   - **Answer:**
      - A process is an independent program with its own memory space, while threads share memory within a process
      - Communication between processes requires IPC (pipes, sockets), but threads can communicate via shared objects, which is why threads are preferred for concurrent data processing
      - Context switching between threads within the same process is cheaper than between processes because threads share memory space and page tables
      - The JVM itself is a single OS process; GC, JIT compilation, finalizers, and application code all run as separate threads within that process
      - Multithreading is preferred because shared memory eliminates IPC overhead, threads consume fewer resources than processes, and thread switching is faster
3. How do you create a thread in Java?
   - **Answer:**
      - One can extend Thread, implement Runnable, or use Callable with ExecutorService
      - In practice, extending Thread directly is discouraged — Runnable with thread pools or CompletableFuture.supplyAsync provides cleaner async execution
      - Approaches include extending Thread, implementing Runnable and passing to Thread or ExecutorService, implementing Callable for return values, using lambda expressions, or CompletableFuture.supplyAsync
      - Implementing Runnable is better because Java has single inheritance, Runnable tasks are decoupled from the thread executing them, and Runnable can be submitted to thread pools for reuse
      - Lambda syntax simplifies thread creation: new Thread(() -> doWork()).start(), or executor.submit(() -> doWork()) for pooled execution
4. Difference between extending `Thread` and implementing `Runnable`.
   - **Answer:**
      - Extending Thread locks you into inheritance; implementing Runnable leaves the class free to extend other classes
      - In Spring services, implementing Runnable or using lambdas keeps the class available for Spring bean proxying
      - Thread class implements Runnable; when you call Thread.start(), it runs the run() method which by default calls the Runnable target's run() if one was provided
      - Extending Thread couples the task logic to the thread lifecycle, prevents extending other classes, and makes the task non-reusable with thread pools
      - Runnable follows the strategy pattern: the task (Runnable) is decoupled from the execution strategy (Thread, ExecutorService, CompletableFuture), enabling flexible execution
      - ExecutorService.execute(Runnable) runs the task asynchronously; submit(Runnable) wraps it in a Future that returns null on completion
5. What is `Callable`?
   - **Answer:**
      - Callable is like Runnable but can return a result and throw checked exceptions
      - For example, Callable can be used with ExecutorService to submit validation tasks that return counts of processed records
      - Callable.call() returns a result and can throw checked exceptions; Runnable.run() returns void and cannot throw checked exceptions
      - ExecutorService.submit(Callable) returns a Future<V>; calling future.get() blocks until the result is available or throws ExecutionException if the task failed
      - Callable is preferred when the task produces a result that the caller needs; Runnable only signals completion without returning data
      - CompletableFuture.supplyAsync wraps a Supplier and internally uses ForkJoinPool; it returns a CompletableFuture that supports chaining, unlike basic Future
6. Difference between `Runnable` and `Callable`.
   - **Answer:**
      - Runnable returns void and cannot throw checked exceptions; Callable returns a value and can throw Exception
      - Runnable is suitable for fire-and-forget tasks like logging, while Callable is used when a result is needed from async computation
      - submit(Runnable) returns Future<Void> where get() returns null on success; submit(Callable) returns Future<T> where get() returns the computed value
      - When submitting a Runnable, the returned Future.get() always returns null because Runnable has no return value
      - Runnable lambda has no return: () -> { doWork(); }; Callable lambda returns a value: () -> computeResult()
7. What is `Future`?
   - **Answer:**
      - Future represents the result of an async computation with methods like get(), isDone(), and cancel()
      - CompletableFuture is generally preferred for chaining, but Future remains useful for getting results from thread pool tasks
      - future.get() blocks indefinitely; future.get(timeout, TimeUnit) throws TimeoutException if the result is not available within the specified time
      - cancel(true) interrupts the executing thread; cancel(false) only prevents the task from starting if it hasn't begun yet
      - Future lacks chaining (no thenApply), manual completion (no complete method), and callbacks (no whenComplete), requiring blocking get() for sequential workflows
      - CompletableFuture supports chaining with thenApply/thenCompose, composition with allOf/anyOf, exception handling with exceptionally, and non-blocking callbacks
8. What is `ExecutorService`?
   - **Answer:**
      - ExecutorService manages a pool of threads and decouples task submission from execution
      - For example, a custom ExecutorService can be configured for parallel report generation, submitting tasks and collecting results via invokeAll
      - corePoolSize is the minimum kept alive; maxPoolSize is the upper bound; keepAliveTime is idle timeout for excess threads; workQueue holds pending tasks when all core threads are busy
      - newFixedThreadPool(n) has fixed size; newCachedThreadPool creates threads on demand with 60s idle timeout; newSingleThreadExecutor runs tasks sequentially; newScheduledThreadPool(n) supports delayed/periodic tasks
      - shutdown() stops accepting new tasks and waits for running tasks to finish; shutdownNow() attempts to stop all tasks immediately and returns a list of pending tasks
9. What is thread pool?
   - **Answer:**
      - A thread pool reuses a fixed number of threads to execute tasks, avoiding the overhead of creating new threads per request
      - For example, Spring Boot async configuration can define a thread pool for @Async methods to handle concurrent processing
      - Thread pools reuse existing threads avoiding creation overhead, limit concurrent threads to control resource usage, and centralize thread lifecycle management
      - Worker threads loop waiting on a blocking queue for tasks; each worker pulls a task, executes it, then returns to waiting for the next task
      - Fixed has a set thread count; cached creates threads as needed; scheduled supports delayed/periodic execution; singleThread executes tasks one at a time
      - When the pool and queue are full, RejectedExecutionHandler decides the policy: AbortPolicy throws exception, CallerRunsPolicy runs on caller thread, DiscardPolicy silently drops, DiscardOldestPolicy removes oldest queued task
10. Why use thread pools?
    - **Answer:**
      - Thread pools reduce overhead from thread creation, control concurrency limits, and improve system stability
      - For example, using a bounded thread pool prevents runaway threads during peak processing from overwhelming the database connection pool
      - Creating a thread allocates stack memory (default 512KB-1MB), involves OS kernel calls, and requires JIT compilation of hot paths, making thread creation expensive
      - Thread pools queue excess tasks when all threads are busy, preventing sudden resource spikes and providing backpressure when the system is saturated
      - Thread starvation occurs when no threads are available for new tasks; resource exhaustion happens when too many threads consume all memory/CPU, degrading the entire system
      - ThreadPoolExecutor exposes metrics via JMX: activeCount, completedTaskCount, queueSize, poolSize, and largestPoolSize for capacity planning
11. Difference between fixed thread pool and cached thread pool.
    - **Answer:**
      - Fixed thread pool keeps a constant number of threads; cached thread pool creates new threads as needed and reuses idle ones
      - Fixed pools are preferred for controlled database access, while cached pools should be avoided in production due to unbounded thread creation risk
      - FixedThreadPool uses LinkedBlockingQueue (unbounded) so maxPoolSize is never reached; CachedThreadPool uses SynchronousQueue (zero capacity) so each new task creates a new thread if all existing threads are busy
      - CachedThreadPool suits short-lived tasks with low arrival rate where thread reuse would add latency, like handling sporadic RPC calls
      - CachedThreadPool creates unlimited threads under sustained load, exhausting memory and OS handles, potentially crashing the JVM or entire system
12. What is scheduled executor service?
    - **Answer:**
      - ScheduledExecutorService can run tasks with delay or periodically
      - For example, it is commonly used for scheduled tasks like refreshing a cache every 5 minutes, with scheduleAtFixedRate for consistent interval execution
      - schedule() runs once after delay; scheduleAtFixedRate() runs at fixed intervals regardless of task duration (if task takes longer, next run starts immediately); scheduleWithFixedDelay() waits the specified delay after each task completes
      - If a task throws an exception, the thread pool is not notified and future executions are silently cancelled; uncaught exceptions terminate scheduled tasks
      - ScheduledThreadPoolExecutor uses a DelayedWorkQueue (a heap-based priority queue) sorted by trigger time to efficiently select the next task to execute
      - Spring's @Scheduled annotation is backed by ScheduledThreadPoolExecutor; @Scheduled(fixedRate=5000) maps to scheduleAtFixedRate, and @Scheduled(cron='...') maps to cron-based scheduling
13. What is synchronization?
    - **Answer:**
      - Synchronization prevents multiple threads from executing critical sections simultaneously, ensuring thread safety via mutual exclusion
      - For example, synchronized methods on shared maps prevent race conditions during concurrent updates
      - Every Java object has an intrinsic lock (monitor); entering a synchronized block acquires the lock, exiting releases it, ensuring only one thread holds it at a time
      - A synchronized instance method locks on 'this'; a static method locks on the Class object; a synchronized block allows locking on any object for finer-grained control
      - Java intrinsic locks are reentrant, meaning a thread that already holds the lock can reacquire it without blocking, enabling recursive or nested synchronized blocks
      - An unlock on a monitor happens-before every subsequent lock on that same monitor, ensuring all memory writes made before the unlock are visible to threads that acquire the lock
      - Excessive synchronization causes contention, context switching overhead, and reduced parallelism; smaller critical sections and lock-free alternatives reduce these costs
14. What is race condition?
    - **Answer:**
      - A race condition occurs when multiple threads access shared data simultaneously and the outcome depends on thread scheduling order
      - For example, a race condition on a shared counter can cause incorrect counts before synchronization is added
      - Check-then-act (if !map.containsKey(k) map.put(k,v)) and read-modify-write (count++) are non-atomic multi-step operations where another thread can intervene between steps
      - Look for shared mutable state accessed by multiple threads without synchronization, compound operations without atomic guarantees, and missing visibility guarantees
      - Atomic classes use CAS for lock-free atomicity; synchronized and ReentrantLock provide mutual exclusion so the entire compound operation executes atomically
      - ConcurrentHashMap uses fine-grained bucket-level locking (synchronized on individual nodes) and CAS operations, allowing concurrent writes to different buckets without global locking
15. What is deadlock?
    - **Answer:**
      - Deadlock occurs when two or more threads each hold a lock and wait for the other to release, causing indefinite blocking
      - Consistent locking order (same order for all threads) is the primary prevention strategy
      - Mutual exclusion (resource is non-sharable), hold-and-wait (thread holds one lock while waiting for another), no preemption (locks cannot be forcibly taken), and circular wait (thread A waits for B who waits for A)
      - jstack prints thread dumps showing lock ownership and wait chains; VisualVM and jconsole provide GUI-based thread monitoring with deadlock detection
      - Consistent lock ordering prevents circular wait; tryLock with timeouts avoids indefinite blocking; lock.striped or single lock eliminates multi-lock scenarios
16. How do you prevent deadlock?
    - **Answer:**
      - Deadlock is prevented by acquiring locks in a consistent global order, using tryLock with timeouts instead of intrinsic locks, and keeping critical sections as small as possible
      - Using a single ConcurrentHashMap can eliminate the need for multiple locks entirely
      - Assign a global ordering to all locks (e.g., by object hashcode) and always acquire them in that order, ensuring no circular wait can form
      - tryLock(timeout, unit) returns false if the lock is not acquired within the time, allowing the thread to release held locks and retry, breaking the hold-and-wait condition
      - Use jstack <pid> or kill -3 <pid> to generate thread dumps; the 'Found one Java-level deadlock' section shows which threads hold and wait for which locks
      - ConcurrentHashMap, AtomicInteger, and other lock-free structures use CAS instead of locks, eliminating the possibility of deadlock from lock acquisition
17. What is livelock?
    - **Answer:**
      - Livelock is when threads keep changing state in response to each other without making progress, like two people stepping aside in the same direction repeatedly
      - Random delays or exponential backoff in retry logic help avoid livelock
      - In deadlock, threads are blocked and make no progress; in livelock, threads are actively running and changing state but still make no progress because they keep responding to each other
      - Thread A and B both try to acquire two locks, detect contention, release their lock, and retry simultaneously, repeatedly failing to make progress despite being active
      - Exponential backoff adds increasing random delays between retries, breaking the synchronized retry pattern so one thread acquires the lock before the other retries
18. What is starvation?
    - **Answer:**
      - Starvation occurs when a thread is perpetually denied access to resources because other threads keep getting priority
      - Fair locks (new ReentrantLock(true)) ensure fairness in thread pools when multiple threads contend for shared resources
      - Low-priority threads may never get CPU time if high-priority threads continuously occupy the scheduler, especially when high-priority threads perform I/O and yield briefly before reacquiring
      - Intrinsic locks are unfair: when the lock is released, any waiting thread can acquire it, not necessarily the one that waited longest, leading to potential starvation
      - new ReentrantLock(true) creates a fair lock using a FIFO queue; the longest-waiting thread gets the lock next, at the cost of lower throughput
      - Undersized thread pools queue tasks that may never execute if the pool is saturated; matching pool size to workload prevents tasks from starving in the queue
19. What is `volatile`?
    - **Answer:**
      - volatile ensures that reads and writes to a variable are directly from main memory, not thread-local cache, providing visibility guarantees
      - For example, a volatile boolean flag can serve as a graceful shutdown signal across threads
      - A write to a volatile variable happens-before every subsequent read of that same variable, ensuring visibility of all memory writes that occurred before the volatile write
      - volatile ensures visibility but not atomicity; count++ involves read, increment, and write as three separate operations, so another thread can interleave between them
      - volatile only ensures visibility and ordering; synchronized provides mutual exclusion (atomicity), visibility, and ordering, making synchronized suitable for compound operations
      - Common volatile use cases include status flags (running, shutdownRequested), double-checked locking (with volatile Singleton instance), and low-frequency configuration values
20. Difference between `volatile` and `synchronized`.
    - **Answer:**
      - volatile guarantees visibility only, synchronized guarantees both visibility and atomicity
      - volatile is suitable for simple flag variables, while synchronized (or locks) is needed for compound operations like check-then-act sequences
      - synchronized ensures only one thread executes the critical section at a time; volatile allows concurrent reads and writes but guarantees all threads see the latest value
      - volatile inserts load-load and store-store barriers preventing reordering; synchronized inserts full barriers (acquire on enter, release on exit) ensuring all preceding operations complete before the critical section
      - AtomicBoolean provides volatile-like visibility plus atomic operations (compareAndSet, getAndSet) without requiring synchronized blocks
      - Double-checked locking uses volatile to prevent instruction reordering: the instance reference is published to other threads only after the constructor completes, preventing use of partially constructed objects
21. What is atomic variable?
    - **Answer:**
      - Atomic variables (AtomicInteger, AtomicLong, AtomicReference) support lock-free, thread-safe operations using CAS (Compare-And-Swap)
      - For example, AtomicLong can track total processed records across consumer threads without locks
      - CAS (Compare-And-Swap) atomically compares the current value with an expected value and, only if they match, swaps it with a new value; the hardware ensures this is a single uninterruptible operation
      - Atomic variables use lock-free CAS operations without thread blocking, avoiding context switching and monitor overhead, making them faster for single-variable updates under moderate contention
      - ABA occurs when a value changes from A to B and back to A; CAS sees A and succeeds even though the value was modified in between; AtomicStampedReference solves this by adding a version stamp
      - LongAdder maintains separate counters per thread (cells array) and sums them on read, reducing CAS contention compared to AtomicLong's single variable under high write contention
22. What is `AtomicInteger`?
    - **Answer:**
      - AtomicInteger provides atomic operations like incrementAndGet, addAndGet, and compareAndSet for int values
      - For example, AtomicInteger can count total events processed across multiple Kafka partitions without synchronized blocks
      - incrementAndGet loops calling CAS: it reads the current value, computes value+1, and attempts compareAndSet; if another thread modified the value concurrently, the CAS fails and the loop retries
      - All operations on AtomicInteger are implemented using volatile reads/writes and CAS at the hardware level, ensuring atomicity and visibility without explicit locks
      - updateAndGet(IntUnaryOperator) atomically applies a function: counter.updateAndGet(x -> x * 2) doubles the value atomically in a CAS loop
      - AtomicInteger outperforms synchronized int for simple operations because it avoids lock acquisition overhead; synchronized becomes competitive for complex compound operations
23. What is lock?
    - **Answer:**
      - A lock is a synchronization mechanism that provides more flexibility than synchronized blocks
      - In Java, ReentrantLock offers tryLock, timed lock, and interruptible lock acquisition
      - synchronized is preferred for simple cases and ReentrantLock for advanced scenarios
      - Lock interface defines: lock() acquires the lock, unlock() releases it, tryLock() attempts non-blocking acquisition, tryLock(timeout) blocks up to the timeout, lockInterruptibly() allows interruption during acquisition
      - Always call unlock() in a finally block to ensure the lock is released even if an exception occurs, preventing resource leaks and potential deadlocks
      - Condition (from lock.newCondition()) provides wait/notify-like coordination: await() releases the lock and blocks, signal() wakes one waiting thread, signalAll() wakes all, enabling fine-grained thread coordination
      - A fair ReentrantLock grants access to the longest-waiting thread (FIFO order); unfair locks (default) have higher throughput but can starve waiting threads
24. Difference between intrinsic lock and `ReentrantLock`.
    - **Answer:**
      - Intrinsic locks (synchronized) are simpler and automatically released; ReentrantLock offers tryLock, fairness, and condition support
      - synchronized is the default choice and ReentrantLock is preferred only when timed lock attempts or interruptible locking are needed
      - synchronized compiles to monitorenter and monitorexit bytecodes; the JVM ensures the lock is acquired on entry and released on exit, even if an exception occurs
      - ReentrantLock delegates to AQS which maintains a volatile state variable and a CLH queue of waiting threads, providing efficient lock management and condition variable support
      - Modern JVMs optimize synchronized with biased locking, thin locks, and lock elision; the performance gap with ReentrantLock is minimal for uncontended cases, but ReentrantLock wins under high contention
      - tryLock() returns false instead of blocking, allowing the thread to release held locks and retry with a different strategy, breaking circular wait conditions
25. What is read-write lock?
    - **Answer:**
      - ReadWriteLock allows multiple readers simultaneously but exclusive write access
      - For example, ReentrantReadWriteLock is useful for a configuration map where reads are frequent and writes are rare, improving throughput
      - ReadLock is shared: multiple threads can hold it simultaneously; WriteLock is exclusive: only one thread can hold it, and no read locks can be held concurrently
      - ReadLock uses reference counting; as long as no writer holds the lock, all readers acquire it without blocking, enabling parallel reads
      - If readers continuously acquire the read lock, a waiting writer may never get its turn; ReentrantReadWriteLock supports a fairness policy to prevent writer starvation at the cost of read throughput
      - ReadWriteLock suits read-heavy workloads with infrequent writes and when you need to guard complex invariants; ConcurrentHashMap is better when operations are independent per key
26. What is semaphore?
    - **Answer:**
      - Semaphore controls access to a limited resource by maintaining a permit count
      - For example, a Semaphore can limit concurrent database connections during bulk data writes, ensuring the connection pool is not exhausted
      - acquire() decrements a permit (blocks if none available); release() increments a permit, unblocking a waiting thread; tryAcquire() returns immediately without blocking
      - A counting semaphore has N permits allowing N concurrent accesses; a binary semaphore (1 permit) acts like a mutex, though unlike locks it is not owner-restricted
      - A semaphore allows N concurrent accesses and has no ownership; any thread can release a permit regardless of which thread acquired it, unlike a lock which must be released by the holder
      - A fair semaphore grants permits in FIFO order; unfair (default) allows barging where a newly arriving thread can acquire a permit before a waiting thread
      - Common use cases include database connection pool limiting, rate limiting API calls, throttling concurrent file downloads, and limiting concurrent batch processing tasks
27. What is countdown latch?
    - **Answer:**
      - CountDownLatch lets one or more threads wait until a set of operations complete
      - For example, it can be used to wait for multiple parallel validation tasks to finish before proceeding to an aggregation step
      - await() blocks the calling thread; each countDown() decrements the internal count; when the count reaches zero, all blocked threads are released simultaneously
      - countDown() atomically decrements the count; it is thread-safe and can be called from any thread; the last thread to call countDown() triggers the release of waiting threads
      - Once the count reaches zero the latch cannot be reset; for repeated synchronization use CyclicBarrier or Semaphore instead
      - CountDownLatch lets N threads count down to zero while one or more wait; CyclicBarrier lets N threads wait for each other to arrive; CountDownLatch is one-shot, CyclicBarrier is reusable
28. What is cyclic barrier?
    - **Answer:**
      - CyclicBarrier lets multiple threads wait for each other at a common point before proceeding
      - Unlike CountDownLatch, it can be reused
      - It is useful in batch processing where N worker threads must sync after each batch of records
      - The barrier action is an optional Runnable that runs on the last arriving thread when the barrier trips, before releasing all waiting threads, useful for summary computation or logging
      - The party count is the number of threads that must call await() before the barrier trips; each call to await() decrements the count and blocks until all parties arrive
      - reset() forces the barrier back to its initial state, releasing any waiting threads with BrokenBarrierException, allowing the barrier to be reused for the next round
      - In CyclicBarrier all threads wait for each other to reach the barrier; in CountDownLatch the waiting thread blocks until the count reaches zero set by other threads
      - await(timeout, unit) throws TimeoutException if the barrier is not reached in time, preventing indefinite blocking when a thread fails to arrive
29. What is concurrent collection?
    - **Answer:**
      - Concurrent collections are thread-safe collections optimized for multi-threaded access, like ConcurrentHashMap, CopyOnWriteArrayList, and BlockingQueue
      - ConcurrentHashMap is commonly used in Kafka consumers for shared caches without external synchronization
      - ConcurrentHashMap uses CAS for simple updates (putIfAbsent) and synchronized blocks on individual bucket nodes for complex operations, allowing concurrent writes to different buckets
      - CopyOnWriteArrayList creates a new array copy on every write, so reads are lock-free and fast; writes are expensive due to array copying, making it ideal for read-heavy scenarios like listener lists
      - BlockingQueue provides put() which blocks when full and take() which blocks when empty, naturally implementing the producer-consumer pattern without manual synchronization
      - Concurrent collections offer better performance under contention by using fine-grained locking; synchronized wrappers (Collections.synchronizedMap) use a single global lock for all operations
30. What is `ThreadLocal`?
    - **Answer:**
      - ThreadLocal provides per-thread variable instances
      - In Spring Boot, each HTTP request runs on a thread and ThreadLocal stores user context
      - ThreadLocal is commonly used for request-scoped logging correlation IDs in APIs
      - Each thread holds a ThreadLocalMap (weak references to ThreadLocal keys, strong references to values); entries are lazily created and stored directly on the thread object
      - When threads are reused, ThreadLocal entries from previous tasks remain in the map; if ThreadLocal is not removed, the entries accumulate and can cause OutOfMemoryError over time
      - Calling remove() in finally blocks or at task completion prevents memory leaks in pooled threads and ensures stale data from previous requests is not visible to subsequent tasks
31. How does Spring Security use `ThreadLocal`?
    - **Answer:**
      - Spring Security stores the authenticated user in SecurityContextHolder using ThreadLocal by default, so every method in the same thread can access the current user's authentication
      - Controllers obtain the logged-in user via SecurityContextHolder.getContext()
      - MODE_THREADLOCAL is the default strategy: SecurityContextHolder stores authentication in a ThreadLocal, so each request thread has its own isolated security context
      - When an @Async or CompletableFuture task runs on a different thread, SecurityContextHolder.getContext() returns null or stale data because ThreadLocal values do not propagate automatically
      - MODE_INHERITABLETHREADLOCAL copies the parent's security context to child threads created by that thread, but this does not work with thread pools since pool threads are reused
      - Implement SecurityContextHolderStrategy to manually copy context, use TaskDecorator in ExecutorService configuration, or use Spring's SecurityContextPropagationFilter for reactive/WebFlux applications
32. What happens to `ThreadLocal` in async execution?
    - **Answer:**
      - ThreadLocal values are not automatically propagated to async threads because the async task runs on a different thread
      - @Async methods require manual context passing or Spring's TaskDecorator to copy ThreadLocal values to the executing thread
      - InheritableThreadLocal copies the parent's value when a child thread is created, but pool threads already exist so they retain their original ThreadLocal values, not the submitting thread's values
      - Create a TaskDecorator that captures ThreadLocal values before the task runs and restores them inside, then configure ThreadPoolTaskExecutor.setTaskDecorator(decorator) in your AsyncConfigurer
      - MDC (Mapped Diagnostic Context) uses ThreadLocal internally, so log correlation IDs and tracing information are lost in async threads unless explicitly propagated via TaskDecorator
33. What is context propagation?
    - **Answer:**
      - Context propagation transfers thread-local state (security, tracing, MDC) across thread boundaries
      - For example, in Kafka consumers, MDC context is propagated from the listener thread to async processing threads so correlation IDs remain consistent across logs
      - Thread pools reuse threads across tasks, so ThreadLocal values from one task leak into the next unless explicitly cleaned up, making context propagation essential for correctness
      - Micrometer's ContextPropagation API defines a ContextRegistry that registers key-value pairs for propagation; TaskDecorator or Reactor hooks copy context across thread boundaries
      - Spring Cloud Sleuth provides TraceRunnable and TraceExecutor that wrap tasks to propagate trace IDs and span IDs from parent to child threads for distributed tracing continuity
      - OpenTelemetry uses Context.current() with ThreadLocal and provides Context.current().wrap(Runnable) and wrapExecutor() to automatically propagate span context across thread boundaries
34. What is thread safety?
    - **Answer:**
      - A class is thread-safe when it behaves correctly under concurrent access
      - Thread safety is achieved through immutable objects, synchronized blocks, atomic variables, or thread-safe collections like ConcurrentHashMap
      - Thread confinement (each thread has its own copy), immutability (no state can change), synchronization (mutual exclusion), and thread-safe data structures (ConcurrentHashMap, CopyOnWriteArrayList) prevent concurrent access issues
      - Establish happens-before relationships between threads: volatile writes, lock releases, and Thread.start() create visibility edges; if every shared write has a corresponding happens-before edge to every read, the code is thread-safe
      - Use frameworks like jcstress or write JUnit tests with CountDownLatch/CyclicBarrier to run concurrent operations; assertions on shared state after synchronization reveals data races
35. How do you make a class thread-safe?
    - **Answer:**
      - Thread safety is achieved by using immutable fields (final), atomic classes for counters, synchronized blocks for critical sections, or delegating to ConcurrentHashMap/CopyOnWriteArrayList
      - For example, a cache service can use ConcurrentHashMap with AtomicLong counters
      - Audit all fields: static fields and non-final instance fields accessible from multiple threads are candidates; ensure each shared mutable field has a thread-safety strategy
      - Prefer immutable objects (zero contention), then atomic variables (lock-free), then locks (fine-grained), then synchronized (coarse-grained); the choice depends on contention level and operation complexity
      - Use @ThreadSafe, @GuardedBy annotations, and javadoc to document which synchronization strategy the class uses and how clients should use it correctly
      - Use ExecutorService with multiple threads performing concurrent operations, CountDownLatch for coordination, and assertions to verify invariants hold under concurrent modification
