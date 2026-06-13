# Java Memory Model

## Overview

- **Definition** — The Java Memory Model (JMM) is a formal specification (JLS Chapter 17) defining how threads interact through shared memory and what values a read can see given certain writes.
- **Why It Exists** — Without a formal memory model, different hardware architectures and JIT compilers could reorder or cache memory operations in ways that break multi-threaded programs. The JMM establishes a contract guaranteeing visibility and ordering when programmers use proper synchronization.
- **Historical Context** — The original JMM (Java 1.0–1.4) had ambiguities that made it hard to reason about thread safety. The current JMM was specified under JSR 133 and released in Java 5, introducing happens-before, volatile semantics, and final field guarantees. It was influenced by the work of Bill Pugh, Jeremy Manson, and others.
- **Key Concepts** — **Happens-before** is a partial ordering ensuring action visibility. **Volatile** guarantees visibility and prevents reordering. **Synchronized** provides mutual exclusion and visibility via monitor locks. **Atomic classes** offer lock-free thread safety via CAS. **Memory barriers** are CPU instructions preventing reordering. **Reordering** is CPU/compiler optimization restricted by happens-before. **Visibility guarantees** ensure values written by one thread are seen by another. **Final field semantics** guarantee safe initialization without synchronization. **Heap vs Stack** separates shared object storage from per-thread call frames. **Method area** stores class metadata. **Young/Old gen** divides heap for generational GC. **Metaspace** replaces PermGen for class metadata in Java 8+.

## Core Concepts

- **JMM overview** — The JMM specifies that a program executes under a set of well-formed executions. An execution is valid if it respects happens-before ordering and is consistent with intra-thread semantics. The model allows out-of-thin-air values to be produced only when data races exist.

- **Heap vs Stack** — The heap is a shared runtime data area where all objects, arrays, and static fields live. It is managed by the garbage collector and accessible from all threads. Each thread has a private stack storing frames — each frame contains local variables, partial results, and method call state. Stack memory is freed automatically when a method exits.

- **Method area** — A logical part of the heap (or native memory with Metaspace) storing per-class structures: runtime constant pool, field and method bytecode, static variables, and symbolic references. Shared across threads.

- **Young and old generations** — The heap is divided into young generation (Eden + Survivor spaces S0, S1) and old generation (Tenured). New objects are allocated in Eden. After surviving several minor GC cycles, objects are promoted to old gen. This generational design exploits the weak generational hypothesis: most objects die young.

- **Metaspace** — Introduced in Java 8, replaces PermGen. Stores class metadata in native memory (outside the heap). Grows automatically and can be limited via `-XX:MaxMetaspaceSize`. Eliminates the common PermGen OutOfMemoryError caused by class loading in application servers.

- **Happens-before** — A partial ordering between actions defined in JLS 17.4.5. If action X happens-before action Y, then X's results are visible to Y and X is ordered before Y. Rules: volatile write happens-before subsequent volatile read of same variable. Unlock of a monitor happens-before every subsequent lock of that monitor. `Thread.start()` happens-before first action in started thread. `Thread.join()` returns only after all actions in the joined thread. Transitive closure of these rules.

- **Volatile** — A field declared `volatile` guarantees that a write to the field happens-before every subsequent read of that field. The JVM inserts memory barriers around volatile operations: a StoreStore barrier before the write, a StoreLoad barrier after the write, and LoadLoad/LoadStore barriers after the read. Volatile prevents reordering of the volatile access with respect to other memory operations.

- **Synchronized** — An intrinsic lock (monitor) providing both mutual exclusion and visibility. When a thread enters a `synchronized` block, it reads fresh values from main memory (flushes local cache). When exiting, it writes back all modified values. The unlock action happens-before the subsequent lock on the same monitor. Synchronized in Java 6+ uses biased locking, lightweight locking, and heavy monitor inflation.

- **Atomic classes** — `java.util.concurrent.atomic` package provides `AtomicInteger`, `AtomicLong`, `AtomicReference`, `AtomicStampedReference`, etc. These use Compare-And-Swap (CAS) instructions at the hardware level (e.g., `Unsafe.compareAndSwapInt`). CAS is lock-free but can suffer from ABA problem (solved by `AtomicStampedReference`). On x86, CAS maps to the `CMPXCHG` instruction.

- **Memory barriers** — Low-level CPU instructions that fence memory operations. Types: LoadLoad (all loads before barrier complete before loads after), StoreStore (stores before complete before stores after), LoadStore, StoreLoad (most expensive — drains store buffer). The JVM inserts barriers based on a platform-specific memory model mapping; the Sun/Oracle JVM uses `OrderAccess` primitives.

- **Reordering** — Compilers (JIT) and CPUs reorder instructions for optimization. The JMM permits reordering as long as single-thread semantics and happens-before are preserved. Reordering can cause another thread to observe operations in a different order. Volatile, synchronized, and `VarHandle` acquire/release semantics insert barriers that prohibit specific reorderings.

- **Visibility guarantees** — Without synchronization, a thread may never see another thread's updates because of CPU caches (store buffers, invalidate queues), compiler hoisting of loads, or instruction reordering. The JMM guarantees visibility only when happens-before is established. This is why `volatile` or `synchronized` are required for shared mutable state.

- **Final field semantics** — The JMM guarantees that a properly constructed object's final fields are visible to all threads without synchronization, provided the constructor does not leak `this`. The compiler must not reorder a final field write with the constructor's return. This enables immutable objects to be safely published without synchronization.

## Common Mistakes

- **Using volatile for compound actions**
  - Declaring a `volatile int counter` and using `counter++` expecting thread safety. The increment is read-modify-write — volatile does not make it atomic.
  - **Why it looks correct:** Volatile guarantees visibility, so the updated value is visible to other threads. Developers assume visibility alone prevents races.
  - Use `AtomicInteger` or `synchronized` for compound actions. Volatile is only suitable for single atomic reads and writes of 32/64-bit values.

- **Double-checked locking without volatile**
  - Implementing a lazy singleton with `if (instance == null) { synchronized (Singleton.class) { if (instance == null) { instance = new Singleton(); } } }` but without volatile on the field.
  - **Why it looks correct:** The pattern is widely published in older resources. Without volatile, the `new Singleton()` write can be reordered such that another thread sees a partially constructed object.
  - Declare the instance field `volatile` (Java 5+). For even simpler code, use `Enum` singleton or `Holder` pattern.

- **Assuming synchronized(this) protects all fields**
  - Synchronizing on an object but expecting it to prevent concurrent access to fields that are touched outside synchronized blocks (e.g., in getters).
  - **Why it looks correct:** Synchronization is expensive, so developers leave getters unsynchronized assuming only writes need protection. But reads without synchronization have no happens-before guarantee.
  - Synchronize both read and write access to shared mutable state, or use volatile/atomic classes. Alternatively, use immutable objects to eliminate the need entirely.

- **Ignoring data races in non-volatile long/double**
  - Reading a `long` or `double` field not declared volatile without synchronization; the JVM may split 64-bit writes into two 32-bit operations, causing torn reads.
  - **Why it looks correct:** On most 64-bit hardware, writes are naturally atomic. Developers test only on x64 where the issue rarely manifests.
  - Mark all `long` and `double` fields accessed from multiple threads as `volatile`, or use `AtomicLong`/`AtomicDouble`.

- **Assuming Thread.start() establishes happens-before for constructor writes**
  - Passing an object to another thread via constructor argument of the `Thread` subclass, then assuming the other thread sees all writes made in the constructor of that object.
  - **Why it looks correct:** The object is passed before `start()` is called, so it seems safe. But the JMM guarantees happens-before from `start()` to the first action of the thread, not from earlier operations in the current thread.
  - Ensure the object being passed is either immutable, safely published via volatile/AtomicReference, or its construction is synchronized with the consumer.

## Real-World Scenarios

### Production data race in a metrics pipeline

- A trading application collected latency metrics using a `long` field updated from multiple threads without synchronization. Under load, some metrics read values of zero or corrupted numbers. The `long` field was 64-bit — on a 32-bit JVM, writes were split into two 32-bit halves, causing torn reads. A thread reading midpoint could see the high 32 bits of one write and low 32 bits of another.

  ```java
  // Broken
  long totalLatency;
  void record(long latency) { totalLatency += latency; }
  long avg() { return totalLatency / count; }

  // Fixed
  private final AtomicLong totalLatency = new AtomicLong();
  void record(long latency) { totalLatency.addAndGet(latency); }
  long avg() { return totalLatency.get() / count; }
  ```

### Happens-before failure in a shutdown hook

- A service used a `boolean running` flag (not volatile) to signal worker threads to stop. The main thread set `running = false` and called `Thread.join()`. Workers occasionally failed to see the flag change, running indefinitely. Because the flag was not volatile and `join()` on workerB does not establish happens-before with workerA, the write to `running` was never flushed to workerA's cache.

  ```java
  // Broken
  boolean running = true;
  void shutdown() { running = false; }

  // Fixed
  volatile boolean running = true;
  ```

### Final field visibility in a configuration object

- An application built an immutable `Config` object in one thread and passed it to multiple worker threads. Workers sporadically saw default values (0, null) for fields not declared `final`. The runtime allowed the constructor's writes to be reordered after the reference was published, because only `final` fields have guaranteed visibility without synchronization. Adding `final` to all fields eliminated the issue with zero overhead.

## Scenario-Based Questions

**Q: Two threads increment a shared `volatile int` one million times each. The final value is less than 2,000,000. Why?**

- Volatile guarantees visibility of the read/write but does not provide atomicity for read-modify-write. The `++` operation is three steps: read, increment, write. Two threads can interleave: both read the same value (say 42), both increment to 43, both write 43. One increment is lost. Use `AtomicInteger.incrementAndGet()` which performs the operation atomically using CAS.
- **Interview follow-up:** How would you design a counter that maintains the same semantics but scales better than `AtomicInteger` under extreme contention?

**Q: A developer writes `StringBuilder` is not thread-safe, so they wrap every usage in `synchronized`. Their multi-threaded application is slow. How can the JMM help them choose a better approach?**

- The developer can use `StringBuffer` (which is synchronized internally) but the overhead is still significant. Better: use thread-local `StringBuilder` instances. The JMM's visibility rules are satisfied because each thread has its own copy — no shared state means no synchronization needed. If sharing is unavoidable, `synchronized` is correct but contention can be reduced by using `ReentrantLock` with fairness settings, or by batching operations.
- **Interview follow-up:** When would you use a `ReentrantLock` over `synchronized` from a memory model perspective?

**Q: A test repeatedly starts threads that read a map while the main thread writes to it without synchronization. The test passes locally but fails in production. Explain why the JMM is responsible.**

- The test runs on a single-socket x86 machine where hardware cache coherence is strong and the JIT may not reorder aggressively. Production might use multi-socket NUMA architectures, different JVM versions, or higher optimization levels. Without happens-before, the JMM permits any behavior: writes may not be visible to reader threads, and the map's internal invariants (linked list pointers, resize flags) can appear corrupted. The test has a data race; its outcome is undefined even if it always passes on one platform.
- **Interview follow-up:** What tools can you use to detect data races in production, and how do they work at the JVM level?

## Interview Questions

- **What is happens-before and why does it matter?**
  - Happens-before is a partial ordering defined in JLS 17.4.5. If action X happens-before action Y, then X's results are guaranteed to be visible to Y, and X is ordered before Y in the execution. It matters because without it, JIT and CPU reordering or caching can cause writes to be invisible or appear in different orders to different threads. Volatile, synchronized, thread start/join, and `ArrayBlockingQueue` operations all establish happens-before edges.

- **Explain the difference between volatile and synchronized.**
  - Volatile guarantees visibility and prevents reordering around the volatile field but provides no mutual exclusion. Synchronized guarantees both visibility and mutual exclusion. Volatile is cheaper (no locking) but cannot be used for compound actions. Synchronized can protect a block of multiple operations atomically. In terms of JMM: volatile write happens-before volatile read; synchronized unlock happens-before subsequent synchronized lock on the same monitor.

- **What are memory barriers and which ones does the JVM use?**
  - Memory barriers (fences) are CPU instructions that restrict reordering of memory operations. Types: LoadLoad (waits for all loads before to complete before any loads after), StoreStore (stores before complete before stores after), LoadStore, and StoreLoad (the strongest — ensures all stores before are visible before subsequent loads). The JVM inserts these based on the platform's memory model. On x86, only StoreLoad is explicitly needed because x86 already guarantees the other ordering properties. The JVM's `Unsafe.fullFence()` inserts a StoreLoad barrier.

- **How do final fields achieve thread safety without synchronization?**
  - The JMM guarantees that a thread observing a reference to a properly constructed object will see the correct values for all its final fields. The compiler must not reorder a final field store with the constructor's return. This is implemented via a StoreStore barrier after the final field writes but before the reference becomes visible. If the constructor leaks `this` (e.g., passing `this` to another thread before returning), the guarantee is void.

- **What is false sharing and how does it relate to the JMM?**
  - False sharing occurs when two threads modify independent variables that happen to reside on the same CPU cache line (typically 64 bytes). Even though the variables are unrelated, the cache coherence protocol invalidates the entire line, forcing repeated cache misses. The JMM does not directly address false sharing but affects it through the visibility semantics of volatile. Mitigation: pad objects to align fields to separate cache lines using `@Contended` (Java 8+) or manual padding fields.

## Developer Recommendations

- **Prefer final fields for immutable objects**
  - Final fields provide visibility guarantees at zero synchronization cost. Using them simplifies reasoning about thread safety.
  - Always declare fields `final` when they are set once and never changed. Use constructor-based initialization instead of setters. The JMM ensures that a reference to the constructed object can be safely published without volatile or synchronized.
  - **Production story:** A financial risk system constructed large `Trade` objects in a builder thread and published them to worker threads via a `BlockingQueue`. Non-final fields caused sporadic stale reads: workers saw `null` counterparty or `0` notional for trades that were fully constructed. Making all trade fields `final` eliminated the data race and improved throughput by removing synchronization from the read path.

- **Use volatile for simple status flags**
  - Volatile is the correct tool when a field is written by one thread and read by others with no compound logic.
  - Declare `volatile boolean shutdown`, `volatile State state`, etc. Ensure the field size is at most 64 bits. Do not use volatile for counters, accumulators, or any read-modify-write pattern.
  - **Production story:** A batch processor used a `boolean running` field without volatile to coordinate shutdown. On certain JVMs, the worker thread never observed the flag change and continued processing for hours after shutdown was requested. Changing to `volatile boolean running` fixed the issue with zero performance impact.

- **Synchronize all access to shared mutable state, not just writes**
  - The JMM visibility guarantee applies only when both reads and writes are synchronized on the same monitor. Unsynchronized reads can see stale values even if writes are synchronized.
  - Use `synchronized` on both getter and setter, or use `ReadWriteLock` if reads dominate. For collections, use `Collections.synchronized*` wrappers or `ConcurrentHashMap`. For simple state, prefer `Atomic*` classes.
  - **Production story:** A leaderboard service used `synchronized` on the update method but left the read method unsynchronized to improve latency. During traffic spikes, reads returned scores that were days old because the JIT hoisted the read out of a loop. Synchronizing both paths eliminated the issue, and `ReadWriteLock` restored the latency target.

- **Measure false sharing before optimizing**
  - False sharing is hardware-specific and rarely the bottleneck in well-structured code. Optimizing prematurely adds complexity.
  - Profile with perf (Linux) or Intel VTune to detect cache misses before applying `@Contended`. Use thread-local data or padding sparingly. Monitor the `cache-misses` hardware counter.
  - **Production story:** A team added `@Contended` to every field in their event dispatcher based on a blog post, but it bloated memory by 3x with no measurable throughput gain. Reverting the change and instead batching events per-thread improved performance more without memory overhead.
