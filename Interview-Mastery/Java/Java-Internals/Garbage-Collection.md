# Garbage Collection

## Overview

- **Definition** — Garbage Collection (GC) is automatic memory management that reclaims heap memory occupied by objects no longer reachable from GC roots.
- **Why It Exists** — Manual memory management (malloc/free, new/delete) leads to memory leaks, dangling pointers, and double-free bugs. GC eliminates these classes of errors by automatically identifying and freeing unreachable objects, at the cost of occasional pause times and CPU overhead.
- **Historical Context** — GC originated in Lisp (1959, by John McCarthy). Java has used GC since JDK 1.0 (1996). Early collectors were simple mark-sweep; the HotSpot VM introduced generational GC in 1.2. CMS (2004) targeted low latency. G1 became the default in Java 9 (2017). ZGC and Shenandoah (Java 11–15) pushed pause times below 1ms.
- **Key Concepts** — **GC roots** are starting points for reachability (thread stacks, static fields, JNI). **Mark-sweep**, **mark-compact**, and **copying** are fundamental algorithms. **Generational GC** divides heap into young/old generations. **Minor**, **major**, and **full GC** refer to collection scope. **Stop-the-world** pauses all application threads. **Serial**, **Parallel**, **CMS**, **G1**, **ZGC**, **Shenandoah** are collector implementations. **TLAB** and **PLAB** are thread-local allocation buffers. **Finalization** (deprecated), **Cleaner**, and **PhantomReference** provide pre-death cleanup hooks.

## Core Concepts

- **GC roots** — Objects that are always reachable and serve as entry points for reachability traversal. Roots include: active thread stack frames (local variables and parameters), static fields of loaded classes, JNI global references (GlobalRef), objects waiting in synchronized blocks, and system class objects. Any object not reachable from a root is garbage.

- **Mark-sweep** — A two-phase algorithm. Phase 1 (mark): traverse the object graph from roots, marking every reachable object. Phase 2 (sweep): scan the heap linearly, adding unmarked objects to the free list. Disadvantage: creates fragmentation, which can cause allocation failures even if total free space is sufficient.

- **Mark-compact** — Extends mark-sweep with a third phase (compact). Live objects are moved to one end of the heap region, eliminating fragmentation. Compaction is expensive but eliminates allocation stalls caused by fragmentation. Used by Parallel Old GC and G1 (during full GC).

- **Copying collectors** — Divide the available space into two equal semi-spaces (from-space and to-space). Live objects are copied from from-space to to-space; the entire from-space is then reclaimed as free. Allocation is fast (bump pointer). Used by all young gen collectors (Eden to Survivor). Disadvantage: requires twice the space and copies all survivors even if many are long-lived.

- **Generational GC** — Based on the weak generational hypothesis: most objects die young. The heap is divided into young generation (Eden + two Survivor spaces) and old generation (Tenured). New objects are allocated in Eden. After surviving a configurable number of minor GCs (tenuring threshold), objects are promoted to old gen. This design minimizes the cost of full-heap scans.

- **Minor, Major, Full GC** — Minor GC collects only the young generation (fast, pauses short). Major GC collects the old generation (slower). Full GC collects the entire heap including metaspace (longest pause). Full GC occurs when young gen promotion fails, concurrent collection fails, or `System.gc()` is called with `-XX:+ExplicitGCInvokesConcurrent`.

- **Stop-the-world (STW)** — All application threads are paused (safepoint) while the collector performs root scanning, marking, and compaction. The duration depends on heap size, live object count, and collector. ZGC and Shenandoah aim to eliminate STW phases by making almost all operations concurrent.

- **Serial GC** (`-XX:+UseSerialGC`) — Single-threaded collector. Performs all GC work on one thread. Suitable for small heaps (<100 MB), single-core machines, or client-class environments. Has the lowest overhead per cycle but longer pause times.

- **Parallel GC** (`-XX:+UseParallelGC`) — Multi-threaded young and old collection. Uses multiple threads for mark-sweep-compact. Default in Java 8. Configurable via `-XX:ParallelGCThreads`. Designed for throughput — it maximizes total application time over GC time.

- **CMS GC** (`-XX:+UseConcMarkSweepGC`) — Concurrent Mark-Sweep. Performs most marking work concurrently with application threads. Reduces pause times but suffers from fragmentation, floating garbage, and "Concurrent Mode Failure" (fallback to Serial Old). Deprecated since Java 9, removed in Java 14.

- **G1 GC** (`-XX:+UseG1GC`) — Default since Java 9. Divides the heap into equal-sized regions (typically 1–32 MB). Marks concurrently, prioritizes regions with the most garbage (garbage-first). Achieves predictable pause targets via `-XX:MaxGCPauseMillis`. Uses remembered sets (RSet) for tracking cross-region references. Performs evacuation (copying) of live objects from selected regions.

- **ZGC** (`-XX:+UseZGC`) — Ultra-low-latency collector. Pause times typically below 1ms regardless of heap size (up to 16 TB). Uses colored pointers (bits in object reference address) for marking state. Performs marking, reference processing, relocation, and remapping concurrently. Relocation is per-region, not per-page. Available since Java 11 (experimental), production in Java 15+.

- **Shenandoah** (`-XX:+UseShenandoahGC`) — Low-pause concurrent GC. Similar goals to ZGC but uses Brooks forwarding pointers instead of colored pointers. Compacts concurrently using a snapshot-at-beginning (SATB) algorithm. Available since Java 12 (experimental), production in Java 15+ (OpenJDK builds).

- **GC tuning flags** — `-Xms<size>` sets initial heap size. `-Xmx<size>` sets maximum heap size. `-Xmn<size>` sets young generation size. `-XX:NewRatio` sets ratio of old:young. `-XX:SurvivorRatio` sets Eden:Survivor ratio. `-XX:MaxTenuringThreshold` controls promotion age. `-XX:MaxGCPauseMillis` (G1) sets pause target. `-XX:G1HeapRegionSize` configures region size. `-XX:ParallelGCThreads` and `-XX:ConcGCThreads` control thread counts.

- **GC logging** — Java 8: `-XX:+PrintGCDetails -XX:+PrintGCDateStamps -Xloggc:<file>`. Java 9+: `-Xlog:gc*:file=<file>`. Logs pause times, heap usage before/after, reason, and time. Useful flags: `-Xlog:gc+ergo*` (ergonomic decisions), `-Xlog:gc+heap*` (region details), `-Xlog:gc+age*` (tenuring). Tools like GCeasy and GCViewer parse these logs.

- **TLAB (Thread-Local Allocation Buffer)** — Each thread gets a small region in Eden (TLAB) for object allocation without synchronization. When the TLAB is exhausted, the thread requests a new one from JVM. TLAB sizing is adaptive; the JVM adjusts based on allocation rate. Enabled by default.

- **PLAB (Promotion-Local Allocation Buffer)** — During GC, each thread gets a PLAB in old gen for copying promoted objects, reducing contention on old gen allocation. PLAB sizing is also adaptive; the JVM monitors promotion rates and adjusts buffer sizes.

- **Finalization vs Cleaner vs PhantomReference** — `finalize()` (deprecated since Java 9, removed in Java 18) was called by GC before reclaiming an object, but execution timing was unpredictable, finalizers could run concurrently, and they could resurrect objects. `java.lang.ref.Cleaner` (Java 9+) registers `Runnable` cleanup actions, invoked by a dedicated thread after the referent becomes phantom-reachable. `PhantomReference` is the weakest reference type — `get()` always returns null. Used with a `ReferenceQueue` for precise post-mortem cleanup, commonly in direct buffer deallocation (`Cleaner` in NIO).

## Common Mistakes

- **Calling System.gc()**
  - Invoking `System.gc()` explicitly in application code to force garbage collection. This may trigger a full GC immediately, causing unnecessary pause times.
  - **Why it looks correct:** It intuitively says "please clean up memory now." Developers use it as a band-aid for memory issues without understanding the root cause.
  - Never call `System.gc()` in production. It is a hint, not a command, but most JVMs respect it by triggering a full GC. If the GC module needs to be notified of memory pressure, use `jcmd GC.run` or `-XX:+DisableExplicitGC` to disable it entirely.

- **Relying on finalize() for resource cleanup**
  - Overriding `finalize()` to close file handles, sockets, or database connections. Finalization is unpredictable, may never run, and has severe performance overhead (100x slower than direct close).
  - **Why it looks correct:** It appears to be the "Java way" of cleanup, and many legacy examples still show it. Developers want a safety net for forgotten close() calls.
  - Implement `AutoCloseable` and use try-with-resources. For rare cases needing pre-death cleanup, use `Cleaner` or `PhantomReference` with `ReferenceQueue`.

- **Ignoring GC logs in production**
  - Running Java applications without enabling GC logging, or believing default GC settings are optimal for all workloads.
  - **Why it looks correct:** The application works and performance seems fine. GC is seen as an opaque subsystem that "just works."
  - Always enable GC logging (`-Xlog:gc*` in Java 9+). Monitor pause times, frequency, and heap usage. Set explicit heap sizes and collector selection based on measured behavior rather than defaults.

- **Setting -Xms too low for throughput-sensitive apps**
  - Setting `-Xms` (initial heap) much smaller than `-Xmx` (max heap), causing frequent GC cycles during startup and warmup as the heap grows incrementally.
  - **Why it looks correct:** Smaller initial heap saves memory, and the JVM will grow the heap as needed. Developers underestimate the cost of repeated GC-triggered heap expansion.
  - Set `-Xms` equal to `-Xmx` for server applications. This eliminates GC cycles caused by heap growth and ensures stable performance.

- **Using -Xmx values that exceed physical memory**
  - Setting max heap to 32 GB on a machine with 16 GB physical RAM, expecting swap to handle the overflow.
  - **Why it looks correct:** The application starts and runs, and developers do not monitor actual memory usage. OS swapping hides the problem until performance collapses.
  - Ensure total heap plus metaspace, thread stacks, off-heap buffers, and OS overhead stays within physical RAM. Monitor with `-Xlog:gc+heap*` and OS tools. For large heaps, use ZGC or Shenandoah to reduce GC overhead.

## Real-World Scenarios

### GC pause causing trading system latency spikes

- A high-frequency trading application using Parallel GC experienced 2–3 second full GC pauses every 5 minutes during peak hours. The full GC was triggered by old gen filling up because promotion rates increased during volatility. The 2-second pauses exceeded the firm's 100ms latency budget, causing missed trades.

  ```
  // Before
  -Xms4g -Xmx4g -XX:+UseParallelGC

  // After
  -Xms8g -Xmx8g -XX:+UseG1GC -XX:MaxGCPauseMillis=50 -XX:G1HeapRegionSize=4m
  ```

- Switching to G1 with a 50ms pause target eliminated full GC pauses. The larger heap reduced GC frequency. G1's concurrent marking and region-based collection kept pauses under 50ms even during promotion bursts.

### Metaspace OOM in a containerized microservice

- A Java 8 microservice with frequent redeploys and class reloading hit `OutOfMemoryError: Metaspace` after several hours. PermGen was replaced by Metaspace in Java 8, but no limit was set, causing unbounded growth.

  ```
  // Fix
  -XX:MaxMetaspaceSize=256m -XX:MetaspaceSize=128m
  ```

- Setting a MaxMetaspaceSize cap caused an immediate OOM if a class loader leak was present. The root cause was a framework that cached `ClassLoader` references indefinitely. Fixing the leak and adding `-XX:+TraceClassLoading -XX:+TraceClassUnloading` helped identify the culprit classes.

### TLAB waste in allocation-heavy application

- A data-processing application allocating millions of short-lived objects per second had high GC overhead despite small live sets. TLAB sizing was inflated because the JVM over-estimated allocation rates during a warm-up spike.

  ```
  // Fix: Use -XX:TLABSize=256k to cap TLAB size
  // or -XX:-ResizeTLAB to disable adaptive sizing
  ```

- Disabling adaptive TLAB resizing (`-XX:-ResizeTLAB`) and setting a fixed size stabilized allocation behavior. Combined with allocating in batches (reusing mutable objects), minor GC frequency dropped by 40%.

## Scenario-Based Questions

**Q: A team runs a Java 8 web service with default GC settings. Under load, response times spike every few seconds. What is likely happening and how do you fix it?**

- The default GC in Java 8 is Parallel GC. The response time spikes are likely stop-the-world minor or major GC pauses. The young generation is sized too small for the allocation rate, causing frequent minor GCs. Promotion failures can escalate to full GC. Fix: switch to G1 (`-XX:+UseG1GC`) with `-XX:MaxGCPauseMillis=100`, or tune Parallel GC by increasing heap (`-Xms4g -Xmx4g`), adjusting `-XX:NewRatio`, and monitoring GC logs to identify the pattern.
- **Interview follow-up:** How would you determine whether the spikes are caused by minor GC, major GC, or concurrent marking phases?

**Q: A developer uses `System.gc()` because their application has a "memory leak." What should they do instead?**

- `System.gc()` triggers a full GC, which may temporarily free memory but does not fix the leak. If the application relies on `System.gc()` to function, it has an undiscovered leak that will resurface. The correct approach: enable GC logging, capture a heap dump (`jmap -dump:live,format=b,file=heap.hprof <pid>`), and analyze with Eclipse MAT or JProfiler to find the unintentional object references. Common causes: unclosed stream objects, listener registration without deregistration, thread-local cache leaks, or classloader leaks in container environments.
- **Interview follow-up:** What are the differences between a "leak" in Java (unintentional object retention) and a "leak" in C++ (unfreed memory), and how does each manifest in GC behavior?

**Q: An application uses ZGC with a 128 GB heap but still sees occasional 10ms pauses. The team expects sub-1ms. What can they check?**

- ZGC typically achieves sub-1ms pauses, but 10ms pauses can arise from: the operating system's page fault handling (ZGC touches memory from multiple threads causing TLB shootdowns), concurrent thread root scanning if safepoint cleanup is slow, or `System.gc()` calls from libraries. Check GC logs for root scanning times, verify `-XX:+UseZGC` is correctly set, ensure `-XX:ConcGCThreads` is adequate, and check the OS for transparent huge pages or NUMA settings. Adding `-XX:+ZUncommit` may help if memory uncommit overhead is high.
- **Interview follow-up:** How does ZGC's colored-pointer approach differ from Shenandoah's forwarding-pointer approach in terms of CPU overhead and memory footprint?

## Interview Questions

- **What is the difference between minor, major, and full GC?**
  - Minor GC collects only the young generation (Eden + Survivor). It is fast and uses a copying collector. Major GC collects the old generation; the exact behavior depends on the collector (Parallel compacts, G1 evacuates regions). Full GC collects the entire heap, including metaspace, and always causes a stop-the-world pause. Full GC is often triggered by promotion failure, concurrent mode failure in CMS, or `System.gc()`.

- **Explain how G1 GC achieves predictable pause times.**
  - G1 divides the heap into equal-sized regions (typically 2048 regions). It tracks the amount of live data and garbage per region using remembered sets (RSets). G1 selects a set of regions with the most garbage (garbage-first) to collect in each pause. By limiting the number of regions collected per cycle and configuring `-XX:MaxGCPauseMillis`, G1 can bound pause duration. The concurrent marking phase runs in the background and identifies regions with high-garbage content for the next mixed collection.

- **What is the weak generational hypothesis and how does it drive GC design?**
  - The weak generational hypothesis states that most objects die young, and references from older objects to younger objects are rare. This observation drives generational GC: the young generation is optimized for rapid collection of short-lived objects (copying collector, small space), while the old generation uses a different algorithm suited for longer-lived objects. Because old-to-young references are rare, G1 and CMS can use remembered sets (card tables) to track cross-generational references without scanning the entire old gen.

- **What is a safepoint and when does it occur?**
  - A safepoint is a point in execution where all application threads have stopped, and the JVM can examine their state. Safepoints occur during GC pauses, deoptimization, biased-lock revocation, and thread dump generation. The JVM uses polling — threads check a memory page at safe locations (method returns, loop back-edges). A thread blocked at a safepoint can delay the entire GC pause.

- **Compare ZGC and Shenandoah.**
  - Both are low-latency concurrent collectors targeting sub-1ms pauses. ZGC uses colored pointers (bits in the 64-bit reference) to encode marking state; Shenandoah uses a Brooks forwarding pointer (extra pointer field in object header). ZGC relies on load-barriers (executed on every object reference read), while Shenandoah uses a SATB (snapshot-at-beginning) algorithm with a similar barrier. ZGC currently supports more platforms (Linux x64, aarch64, Windows) while Shenandoah is in OpenJDK builds. ZGC handles larger heaps (up to 16 TB) more efficiently.

## Developer Recommendations

- **Select the right GC for the workload**
  - No single GC fits all use cases. Throughput-oriented applications benefit from Parallel GC. Latency-sensitive applications need G1, ZGC, or Shenandoah. Small heaps on single-core systems work well with Serial GC.
  - Measure latency percentiles, throughput, and footprint under realistic load. Start with G1 (the modern default), then switch to ZGC/Shenandoah if sub-10ms pauses are required. Profile before tuning — many "GC problems" are actually application allocation problems.
  - **Production story:** A real-time analytics platform using Parallel GC had 500ms+ pauses during daily batch processing. Switching to G1 with 200ms pause target eliminated latency alerts. Later, migrating to ZGC when the heap grew past 64 GB removed GC from their monitoring dashboard entirely — pauses dropped to under 2ms.

- **Always set -Xms equal to -Xmx for server JVMs**
  - Unequal initial and max heap sizes cause the JVM to spend time growing the heap, triggering additional GC cycles during the growth process. This increases variance in latency and throughput.
  - Set both flags to the same value at startup. Determine the value from production measurements — monitor heap usage after warmup, add 30–50% headroom, and set that as the fixed heap size.
  - **Production story:** A payment gateways team set `-Xms512m -Xmx4g` for a Java 8 service. During traffic spikes after lunch, the heap grew from 512 MB to 3 GB, causing a chain of minor GCs every 2 seconds. Fixing to `-Xms4g -Xmx4g` eliminated the growth-related GC cycles and reduced 99th percentile latency by 60%.

- **Enable and monitor GC logging from day one**
  - GC logs are the primary diagnostic tool for memory-related issues. Without them, identifying a promotion rate problem, metaspace leak, or excessive safepoint times is guesswork.
  - Use `-Xlog:gc*:file=gc-%t.log:time,level,tags:filesize=10m,filecount=10` (Java 9+) or the Java 8 equivalent. Integrate GC log analysis into the alerting pipeline using GCeasy or custom parsers.
  - **Production story:** A team spent two weeks debugging intermittent latency spikes, convinced it was a database issue. One day of GC log parsing revealed that a full GC ran every time the batch job kicked off, due to a `-XX:+ExplicitGCInvokesConcurrent` interaction. The fix: `-XX:+DisableExplicitGC` and removing `System.gc()` from the batch job.

- **Avoid finalization entirely; prefer Cleaner or try-with-resources**
  - finalize() is unpredictable, carries a huge performance penalty (the object requires two GC cycles to reclaim), and can resurrect objects. It is deprecated and removed in recent JDKs.
  - Implement `AutoCloseable` for all external resource wrappers. Use try-with-resources. For cases where resource cleanup must happen automatically when an object becomes unreachable (e.g., direct buffer deallocation), use `Cleaner` with a registered `Runnable`.
  - **Production story:** A networking library used `finalize()` to close socket channels. Under high throughput, the finalization queue grew to millions of entries, causing `OutOfMemoryError` because the finalizer thread could not keep up. The GC overhead was 40%. Replacing with `Cleaner` and explicit `close()` in try-with-resources eliminated the queuing and reduced GC CPU from 40% to 5%.
