# Async Profiler

## Overview

- **Definition** — async-profiler is a low-overhead, production-safe sampling profiler for Java that combines CPU, allocation, wall-clock, and context-switch profiling using `perf_events` (Linux) and the AsyncGetCallTrace (AGCT) JVM interface.
- **Why It Exists** — Traditional Java profilers (YourKit, JProfiler) use JVMTI agents with `SetBreakpoint` or bytecode instrumentation, causing 5-20% overhead and requiring JVM restarts. `perf` and `perf-map-agent` natively profile C++ code but cannot decode Java stack frames. async-profiler bridges this gap: it uses Linux `perf_events` for hardware-level CPU sampling (cycle accurate, zero Java overhead) and AGCT to unwind Java stack frames from the perf samples, producing accurate flame graphs with both Java and native frames.
- **Historical Context** — Created by Andrey Pangin (2016) to solve the problem of profiling Java applications without the overhead of JVMTI agents. Open-sourced as `async-profiler` (Apache 2.0). Adopted by IntelliJ IDEA as its built-in profiler (2020+). Became the de-facto standard for production Java profiling, replacing Oracle's `hprof` and commercial JVMTI agents. Integrated into JDK Flight Recorder via the `jdk.JFR` event streaming bridge (JDK 16+).
- **Key Concepts** — **`perf_events`** provides hardware-based sampling (CPU cycles, instructions, cache misses). **AGCT** (AsyncGetCallTrace) is an undocumented JVM function that safely walks the call stack from any thread at any point in execution. **Flame graphs** are the primary visualization (icicle graph for CPU, flame graph for allocations). **Wall-clock profiling** captures threads in any state (not just running). **Allocation profiling** hooks into TLAB and GCLAB to capture allocation sites. **Differential profiling** compares two profiles to find regressions. **FD-transfer** sends profiling data to the JMC/flamegraph reader without file I/O. **`jattach`** attaches the profiler to a running JVM without restart.

## Core Concepts

- **CPU profiling** — async-profiler uses `perf_events` to sample the CPU's performance counter at a configurable interval (default 99 Hz or 100 Hz on macOS). At each sample, it reads the current instruction pointer and uses AGCT to walk the Java stack. The result is a statistical profile of where the JVM spends CPU cycles. The default interval of 99 Hz avoids aliasing with common timer frequencies (100 Hz is also common).
  ```bash
  # CPU profile for 30 seconds
  profiler.sh -d 30 -o flamegraph <pid>
  ```

- **Wall-clock profiling** — Unlike CPU profiling, wall-clock profiling captures threads regardless of their state (runnable, sleeping, blocked on I/O, waiting on locks). Use this when investigating latency rather than CPU usage: a slow response may be caused by a thread blocked on a database query, not by CPU saturation.
  ```bash
  # Wall-clock profile (includes idle/blocked threads)
  profiler.sh -d 30 -e wall -o flamegraph <pid>
  ```

- **Allocation profiling** — async-profiler hooks into TLAB (Thread-Local Allocation Buffer) and GCLAB (GC-Local Allocation Buffer) to capture allocation sites. Instead of instrumenting every `new` instruction (JVMTI overhead), it records the stack trace when a TLAB fills up, providing a statistical view of allocation hot spots with very low overhead (<2%).
  ```bash
  # Allocation profile (objects allocated by site)
  profiler.sh -d 30 -e alloc -o flamegraph <pid>
  ```

- **Lock profiling** — async-profiler can record contended lock acquisitions by sampling threads blocked on `AbstractOwnableSynchronizer` (AQS-based locks like `ReentrantLock`, `CountDownLatch`, `Semaphore`) and `synchronized` blocks.
  ```bash
  # Lock contention profile
  profiler.sh -d 30 -e lock -o flamegraph <pid>
  ```

- **Context-switch profiling** — Records the stack trace of threads that are context-switched out (involuntarily preempted). Useful for diagnosing thread starvation, scheduler latency, and excessive context switching under CPU contention.
  ```bash
  # Context-switch profiling (requires root on Linux)
  profiler.sh -d 30 -e context-switches -o flamegraph <pid>
  ```

- **Flame graphs** — The primary output format. Each rectangle represents a stack frame; width is proportional to how often that frame was sampled. Flame graphs (bottom-up) show the call stack hierarchy — the wider a frame at the bottom, the more time is spent in that function or its descendants. Icicle graphs (top-down) invert the view. async-profiler generates interactive SVG flame graphs directly.
  ```bash
  profiler.sh -d 30 -o flamegraph -f profile.svg <pid>
  ```

- **`jattach`** — The profiler ships `jattach`, a lightweight utility that sends commands to a JVM via the Attach API without requiring `jcmd` or `tools.jar`. It does not require the JDK — a JRE is sufficient:
  ```bash
  jattach <pid> load /path/to/libasyncProfiler.so start=event=cpu,file=profile.svg
  ```

- **FD-transfer** — async-profiler can transfer profiling output directly to a file descriptor instead of writing to a file. This enables integration with tools like JMC: `profiler.sh -d 30 -o jfr -f /dev/fd/3 3>&1` pipes the JFR output to stdout.

- **Differential profiling** — Compare two profiles to find what changed. Typically used to identify regressions between deployments:
  ```bash
  profiler.sh -d 60 -o collapsed -f before.collapsed <pid>
  # ... after deployment ...
  profiler.sh -d 60 -o collapsed -f after.collapsed <pid>
  FlameGraph/difffolded.pl before.collapsed after.collapsed > diff.svg
  ```

- **JFR output** — async-profiler can output in JFR format, which can be opened in JMC:
  ```bash
  profiler.sh -d 30 -o jfr -f profile.jfr <pid>
  ```
  This combines async-profiler's low overhead with JMC's rich visualization.

- **SafePoints and profiling accuracy** — async-profiler uses AGCT which is signal-safe — it can be called from a signal handler (SIGPROF) to walk Java stacks at any point, including during safepoints. This means it accurately sees application time and GC time, unlike JVMTI `GetStackTrace` which blocks during safepoints.

## Common Mistakes

- **Using CPU profiling to diagnose application latency**
  - Running CPU profiling on a service whose bottleneck is I/O (database queries, HTTP calls, message queues). The CPU profile shows 95% idle, providing no actionable information.
  - **Why it looks correct:** CPU profiling is the default event type and the most well-known profiling mode. Developers reach for it first.
  - Use wall-clock profiling (`-e wall`) when the service is I/O-bound or when investigating response-time issues. Wall-clock captures blocked, sleeping, and waiting threads — revealing the actual source of latency.

- **Profiling at default frequencies without understanding alias risks**
  - Using the default 99 Hz or 100 Hz sampling rate on a system where periodic work happens at similar frequencies (e.g., a scheduled task running every 10ms).
  - **Why it looks correct:** The sampling rates are the recommended defaults and seem precise enough.
  - If periodic work aligns with the sampling interval, the profile may over- or under-sample certain code paths. Use prime-numbered frequencies (97 Hz, 199 Hz) to avoid aliasing, or vary the interval: `-i 997us` (approximately 1003 Hz).

- **Reading flame graphs bottom-up without considering Icicle view**
  - Looking at a CPU flame graph and concluding that "method X is hot" without checking whether X's siblings or children dominate the profile.
  - **Why it looks correct:** Flame graphs are traditionally displayed bottom-up (root at bottom, leaf at top). The widest frame at the bottom appears to be the hottest.
  - Always check both flame and icicle (reverse) views. A frame that is wide in the flame graph but narrow in the icicle view may be a callee of many different callers, not a hot method itself. The icicle view shows which calls initiate the workload.

- **Profiling allocation without understanding TLAB limits**
  - Using `-e alloc` on a JVM with very large TLABs (> 10MB), causing very few allocation samples and a sparse profile even under heavy allocation.
  - **Why it looks correct:** Allocation profiling works — events are recorded. The developer does not realize the sample count is far too low for statistical significance.
  - TLAB sizing affects allocation profile resolution. Smaller TLABs yield more samples (better accuracy) but higher overhead. For allocation profiling, set `-XX:TLABSize=512k` or lower during the profiling window. Monitor the total event count — a valid profile should have thousands of samples.

- **Attaching async-profiler as root on a containerized JVM without the correct privileges**
  - Running `profiler.sh` inside a Docker container without `--cap-add SYS_ADMIN` or `--privileged`, causing `perf_event_open` failures and empty CPU profiles.
  - **Why it looks correct:** The command runs, but silently produces no samples because AGCT returns "no stack trace available" for every sample.
  - For CPU profiling in containers, either: (a) run with `--cap-add SYS_ADMIN` or `--privileged`, (b) use `-e itimer` (async-profiler's safe-mode CPU profiling without `perf_events`), or (c) use wall-clock profiling (`-e wall`) which does not require `perf_events`.

## Real-World Scenarios

### CPU profile revealing unintended JSON serialization in hot path

- A search service's CPU usage increased from 20% to 80% after a deployment. CPU profiling with async-profiler shows 60% of samples in Jackson's `writeValueAsString()`. The flame graph shows the serialization happens inside a loop that processes 10,000 search results — the developer serializes per-result instead of building a list and serializing once. Fix: collect results into a list and serialize once. CPU drops back to 20%.

### Wall-clock profile identifying connection pool saturation

- A payment service has intermittent p99 spikes. CPU profiling shows low utilization. Wall-clock profiling (`-e wall`) reveals 70% of threads blocked on `HikariCP.getConnection()` with a stack trace showing pool timeout. The connection pool has 10 connections, and one payment processor endpoint takes 5 seconds per call, effectively exhausting the pool. Fix: increase pool size and add a circuit breaker to the slow endpoint.

### Allocation profiling finding string concatenation bottleneck

- A batch processing job runs 3x slower than expected. CPU profiling shows mostly GC time (30%). Allocation profiling (`-e alloc`) reveals that 80% of allocations come from a method that uses `String + String + String` formatting in a loop processing 1M records. Each iteration creates multiple intermediate `StringBuilder` objects and discarded strings, triggering constant GC. Fix: replace with a reusable `StringBuilder` outside the loop. GC time drops to 5%.

### Lock contention causing thread pool exhaustion

- A microservice's throughput drops 50% after enabling distributed tracing. CPU profiling is inconclusive. Lock profiling (`-e lock`) shows 40% of samples in `BraveSpan` (tracing library) synchronized block on a shared `Tracer` instance. The trace span is created per-request but synchronization on the tracer serializes all requests. Fix: use a thread-local span context. Throughput recovers.

### Differential profiling catching a JDK upgrade regression

- After upgrading from JDK 11 to JDK 17, request latency increased 15%. Differential profiling compares the JDK 11 flame graph with JDK 17: the delta shows 8% more time in `StringLatin1.indexOf()`. Investigation reveals that Java 17's `String.indexOf()` uses a different algorithm for certain patterns, and the application's search pattern is a worst-case input for the new algorithm. Fix: implement a custom search for the specific pattern.

## Use Cases

Reach for async-profiler when you need to understand what a JVM is actually doing at the CPU, allocation, or thread level — especially in production where traditional profilers are too heavy.

- **CPU hot-spot analysis** — Use CPU profiling when a service is CPU-bound and you need to find the hottest methods. The flame graph immediately shows where CPU cycles are spent.
  - **Avoid when:** The bottleneck is I/O — use wall-clock profiling instead.

- **Wall-clock latency investigation** — Use wall-clock profiling (`-e wall`) when response times are high but CPU utilization is low. This captures threads blocked on locks, database connections, network calls, or sleeping — revealing the true source of latency.
  - **Avoid when:** You are investigating CPU contention only — wall-clock includes idle time and dilutes the CPU signal.

- **Allocation hotspot detection** — Use allocation profiling (`-e alloc`) when GC frequency is high and you need to find which code paths allocate the most memory. Statistically samples allocation sites via TLAB flush events.
  - **Avoid when:** The application allocates trivially (< 100 MB/s) — allocation profiling noise may obscure meaningful patterns.

- **Lock contention diagnosis** — Use lock profiling (`-e lock`) when thread dumps show many threads in `BLOCKED` state, or when throughput drops under moderate concurrency.
  - **Avoid when:** Lock contention is already identified via thread dumps — use CPU profiling to understand what the lock-holding thread is doing.

- **Production incident response** — async-profiler attaches to a running JVM without restart. When an incident occurs, start a 30-second CPU or wall-clock profile. The SVG flame graph is immediately interpretable by the on-call engineer.
  - **Avoid when:** The JVM is in a crash loop with < 10 seconds of uptime — attach may not complete before the process exits.

- **CI/CD performance regression detection** — Run async-profiler in performance test pipelines. Output collapse files, store them per build, and use differential profiling to detect regressions between commits.
  - **Avoid when:** Performance tests are too short (< 10 seconds) — async-profiler needs sufficient samples for statistical significance.

## Scenario-Based Questions

**Q: You attach async-profiler to a production JVM for CPU profiling. The output flame graph shows 90% of samples in `[unknown]` frames. What went wrong?**

- The `[unknown]` frames indicate that AGCT could not decode the stack trace into Java frames. Common causes: (a) The JVM is running with `-XX:-PreserveFramePointer` and AGCT cannot walk the stack. (b) The profiler version is incompatible with the JDK version. (c) The JVM is in the middle of a JIT compilation, and the compiled method's entry has a stack walking issue. (d) The `perf_events` mapping for the JVM's code cache is not set up. Fix: ensure `-XX:+PreserveFramePointer` is set (or the JVM is modern enough to not need it), use the latest async-profiler version, and verify the `perf-map-agent` is configured.
- **Interview follow-up:** You verify `-XX:+PreserveFramePointer` is set and the profiler version matches the JDK. The flame graph still shows 90% `[unknown]` frames. What `perf`-related system configuration could be missing on a containerized deployment?

**Q: A wall-clock profile shows 60% of threads blocked on `java.util.concurrent.LinkedBlockingQueue.take()`. The application is a message consumer. Is this normal or a problem?**

- A consumer thread blocked on `take()` waiting for messages is normal — that is the expected behavior of a blocking queue consumer. The profile shows 60% of threads blocked, meaning 60% of the wall-clock time is spent idle waiting for work. If the application has 10 consumer threads and all 10 are blocked, there is no backlog. If 5 of 10 are blocked and 5 are processing, the system is balanced. The question is not whether threads are blocked, but whether the blocked time indicates under-utilization or a bottleneck. Check queue depth and processing rate.
- **Interview follow-up:** You find that only 2 of 10 consumer threads are processing while 8 are blocked. The message rate is 1000 msg/sec and each message takes 5ms to process (2 threads = 400 msg/sec capacity). The backlog is growing. The profile shows the processing threads spend 80% of their time in `String.format()`. What optimization reduces per-message processing time?

**Q: You profile a service with `-e alloc` for 60 seconds and get only 100 allocation samples. The application allocates at 500 MB/s. Why are there so few samples?**

- async-profiler's allocation profiling hooks into TLAB flush events. If the JVM's TLAB is very large (> 10 MB), each thread flushes its TLAB infrequently, generating few samples even under high allocation volume. The TLAB size is typically 1% of Eden space, so a large heap with large Eden produces large TLABs. Fix: reduce TLAB size temporarily during profiling with `-XX:TLABSize=512k`, or use CPU profiling with `-e cpu` and look for allocation-related methods (malloc, GC code) instead.
- **Interview follow-up:** You reduce TLAB size to 256k and restart profiling. Now you get 50,000 allocation samples. The allocation flame graph shows 70% of allocations in the same method, but GC time did not decrease when you optimized that method. What could explain this discrepancy?

**Q: You run CPU profiling on an I/O-intensive web server. The CPU flame graph shows 95% idle/blocked. What profiling mode should you use instead, and why?**

- Use wall-clock profiling (`-e wall`) instead of CPU profiling. CPU profiling only samples threads in RUNNABLE state. An I/O-bound thread spends most of its time blocked on network reads, database queries, or message queue receives — none of which are RUNNABLE. Wall-clock profiling samples threads regardless of state, showing where time is actually spent: waiting on database connections, waiting on HTTP responses, or blocked on locks.
- **Interview follow-up:** After switching to wall-clock profiling, you see that 40% of samples are in `java.net.SocketInputStream.read()`. The application is a microservice calling another service. How do you determine whether the 40% blocked time is due to network latency or the downstream service's processing time?

**Q: You have a method that is the widest frame in both the CPU flame graph and the allocation flame graph. The method uses a HashMap inside a hot loop. Should you optimize the HashMap usage or the allocation pattern?**

- The two profiles indicate different bottlenecks: CPU time and allocation rate. In the CPU flame graph, the width of the HashMap operations (put, get, resize) tells you whether the CPU cost is computational (hashing, equality checks) or structural (resizing/rehashing). In the allocation flame graph, check if the HashMap is being created repeatedly inside the loop (short-lived objects) or if the entries are being allocated frequently. If the CPU profile shows most time in `HashMap.put()` with allocation profile showing `HashMap` constructor, create the HashMap outside the loop to eliminate reallocation. If the CPU profile shows `HashMap.get()` and allocation shows `HashMap$Node` allocation, consider reducing lookup frequency.
- **Interview follow-up:** You move the HashMap outside the loop. CPU profile improves 20%. The allocation profile now shows a different method as the top allocation site — but GC time decreased by only 5%. What does this suggest about the relationship between allocation rate and GC pause time?

**Q: async-profiler's CPU profiling via `perf_events` requires `CAP_SYS_ADMIN` in Linux. You cannot get this permission in a shared Kubernetes cluster. How do you profile CPU usage?**

- Options: (a) Use async-profiler's `-e itimer` mode, which uses `setitimer(ITIMER_PROF)` instead of `perf_events`. This does not require special permissions and provides CPU profiling via an interval timer signal. It has slightly higher overhead but works in any container. (b) Use wall-clock profiling (`-e wall`) with a fine interval (500us) — though this includes idle time, it still shows where runnable threads spend time. (c) Use JFR's CPU profiling (`-XX:StartFlightRecording:settings=profile`) which also does not require kernel permissions.
- **Interview follow-up:** You switch to `-e itimer` and get a profile, but the total sample count is 5x lower than `perf_events` CPU profiling for the same duration. What explains the difference in sampling depth?

**Q: You profile a service and the flame graph shows a very wide `GC` frame taking 30% of CPU samples. The application has a 4 GB heap with 70% occupancy. What do you investigate?**

- 30% CPU in GC means the application spends nearly a third of its CPU time on memory management. Investigate: (a) What is the allocation rate? — high allocation forces frequent GC. (b) What collector is being used? — Parallel GC with 4 GB heap may cause long pauses; G1 with `MaxGCPauseMillis` may help. (c) Are there allocation hot spots? — use `-e alloc` to find the top allocation sites. (d) Is the heap sized correctly? — try increasing heap to reduce GC frequency (but larger heap may increase pause times with non-concurrent collectors).
- **Interview follow-up:** The allocation rate is 2 GB/s and the top allocation site is byte array allocation in a JSON deserialization library. The application parses large JSON payloads. You try to reduce allocations in the deserialization library but hit diminishing returns. What alternative approach reduces GC time without touching the allocation rate?

**Q: You run differential profiling comparing yesterday's profile with today's. The diff flame graph shows a new hot method `EncryptionUtil.encrypt()` taking 15% of CPU samples. No code change was deployed. What could have happened?**

- No code change does not mean no behavioral change. Possible causes: (a) A configuration change enabled encryption that was previously disabled. (b) A feature flag was toggled, enabling an encrypted code path. (c) An A/B test started serving a new variant that includes encryption. (d) A downstream service started returning encrypted data that must be decrypted before processing. (e) An external dependency was updated in a patch version that introduced encryption. Check deployment logs, feature flags, and dependency versions.
- **Interview follow-up:** You find that a third-party library was updated from version 1.0.0 to 1.0.1 (semver patch) which introduced transparent encryption. The library author considered it a patch because it was a security fix. How would you prevent this class of regression in your CI/CD pipeline?

**Q: You profile a Spring Boot application and the flame graph shows 40% of samples in `Catalina.exec()` and 30% in application code. The server handles 500 requests/second. What does this distribution tell you?**

- 40% in `Catalina.exec()` means the application is spending 40% of its time in the Tomcat request-dispatch loop (accepting, reading, writing). This is expected for I/O-bound web applications — Tomcat threads spend significant wall time waiting on socket I/O. The 30% in application code means the actual business logic is only 30% of wall time. This distribution is healthy. If application code were <10%, the bottleneck is likely I/O or framework overhead. If it were >70%, the application code itself is CPU-bound.
- **Interview follow-up:** You add more Tomcat threads (200 → 400) hoping to increase throughput. The application code share in the flame graph drops to 15%. What does this tell you about the bottleneck?

**Q: A developer attaches async-profiler with `-e cpu` on a laptop running macOS and gets zero samples. The same command works on Linux. What is the difference?**

- macOS does not have Linux `perf_events`. async-profiler on macOS uses DTrace (macOS's instrumentation framework) for CPU profiling, which requires different privileges and may not work on all macOS versions (especially on Apple Silicon where DTrace is limited). On macOS, use `-e wall` with `-i 1ms` for CPU profiling, or use the `-e cpu` mode (which falls back to `thread_cpu_time` API on macOS). For reliable macOS profiling, consider using JFR or the IntelliJ profiler integration, which handles macOS-specific profiling transparently.
- **Interview follow-up:** On Apple Silicon (M1/M2), even `-e wall` profiling shows different call stacks compared to Linux. What JVM implementation difference between macOS and Linux causes the same Java code to produce different async-profiler results?

## Interview Questions

- **What is async-profiler and how does it differ from JFR?**
  - async-profiler is a sampling profiler that uses Linux `perf_events` for CPU sampling and the JVM's AsyncGetCallTrace to walk Java stacks. JFR is an event-recording framework built into the JVM. async-profiler provides deeper stack traces (including native and kernel frames) and lower overhead for CPU profiling. JFR provides richer event types (GC phases, JIT compilation, safepoints) and does not require external tools.

- **How does AGCT work and why is it important?**
  - AsyncGetCallTrace is a JVM function that safely walks the call stack from any thread at any point, including during safepoints. It is signal-safe, meaning it can be called from a SIGPROF handler. This enables accurate statistical profiling without JVMTI overhead.

- **What profiling modes does async-profiler support?**
  - CPU (`-e cpu`), wall-clock (`-e wall`), allocation (`-e alloc`), lock contention (`-e lock`), context-switch (`-e context-switches`), and JFR output combined with any of these.

- **What output formats does async-profiler support?**
  - Flame graph SVG, collapse (summary text), tree, JFR, HTML. Collapse format is used for differential profiling.

- **How do you profile a Java application without restarting it?**
  - Use `profiler.sh -d 60 -o flamegraph -f profile.svg <pid>`. The profiler attaches via `jattach`, which uses the JVM Attach API. No restart is required.

- **What is `jattach` and why is it needed?**
  - `jattach` is a lightweight utility that sends commands to a JVM via the Attach API. It is needed because the Attach API is not available in a JRE (requires `tools.jar` in JDK 8). `jattach` works with JRE only.

- **What is the overhead of async-profiler?**
  - CPU profiling: <1% overhead (uses hardware counter sampling, no instrumentation). Allocation profiling: <2% overhead (hooks into TLAB flush). Wall-clock profiling: <1% overhead (timer-based sampling).

- **What is `perf_events` and how does async-profiler use it?**
  - `perf_events` is the Linux kernel's performance monitoring subsystem. async-profiler uses it to sample the CPU's performance counter (cycles or instructions) at a configurable frequency. When a sample fires (SIGPROF signal), async-profiler uses AGCT to walk the Java call stack and record it.

- **Why does async-profiler need `CAP_SYS_ADMIN` for CPU profiling on Linux?**
  - `perf_event_open()` system call requires `CAP_SYS_ADMIN` or `/proc/sys/kernel/perf_event_paranoid <= 1`. Without it, `perf_events` cannot open a counter. Containers (Docker, Kubernetes) typically deny this capability.

- **How do you profile CPU in a container without `CAP_SYS_ADMIN`?**
  - Use `-e itimer` (async-profiler's safe-mode CPU profiling using `setitimer(ITIMER_PROF)`) or `-e wall` (wall-clock profiling). Both work without `perf_events`.

- **What is a flame graph and how do you read one?**
  - A flame graph (SVG or HTML) visualizes stack traces from profiling. Each rectangle is a stack frame; width represents the proportion of samples. The bottom row is the root (topmost frame in the call stack). The wider a function at the bottom, the more CPU (or wall-clock) time is spent in that function or its descendants.

- **What is the difference between a flame graph and an icicle graph?**
  - Flame graphs arrange the call stack from bottom (root) to top (leaf). Icicle graphs invert this: the caller is at the top and callees hang below. Both show the same data; the icicle view makes it easier to see which top-level callers drive the workload.

- **How does allocation profiling work?**
  - async-profiler intercepts TLAB (Thread-Local Allocation Buffer) and GCLAB allocation events. When a TLAB fills up and a new one is allocated, the profiler captures the stack trace. This provides a statistical sample of allocation sites with very low overhead.

- **How does lock profiling work?**
  - async-profiler samples threads in `BLOCKED` state and records their stack traces. It also hooks into `AbstractQueuedSynchronizer` (AQS) to capture contended lock acquisitions. The resulting profile shows where threads are blocked waiting for locks.

- **How do you perform differential profiling?**
  - Run async-profiler twice (before and after), output collapse format each time. Use `FlameGraph/difffolded.pl` to generate a differential flame graph: red frames increased, blue frames decreased. This shows performance deltas between builds, deployments, or load levels.

- **What is the effect of `-XX:+PreserveFramePointer` on profiling?**
  - This JVM flag preserves the frame pointer register (RBP on x86, RFP on ARM), which is essential for AGCT to walk the call stack accurately. With JDK 11+, the JVM typically preserves frame pointers for interpreted methods, but JIT-compiled methods may omit them. `-XX:+PreserveFramePointer` forces all methods to maintain the frame pointer, at a small performance cost (<2%).

- **Can async-profiler profile native code?**
  - Yes. async-profiler captures native stack frames from `perf_events` samples and shows them in the flame graph alongside Java frames. This is critical for diagnosing JNI calls, native libraries, and kernel code.

- **What is the sampling frequency of async-profiler and how do you configure it?**
  - Default CPU sampling: 99 Hz (99 samples/second). Default wall-clock: 99 Hz. Allocation: per-TLAB-flush. Locks: per-contention-event. Configure with `-i N` (interval in nanoseconds) or `-f N` (frequency in Hz): `profiler.sh -d 30 -i 500us -o flamegraph <pid>`.

- **How does async-profiler integrate with IntelliJ IDEA?**
  - IntelliJ IDEA Ultimate bundles async-profiler as its built-in Java profiler. The "CPU Profiler" and "Allocation Profiler" run configurations use async-profiler under the hood. Results are displayed in the IntelliJ profiler UI with flame graph, call tree, and method lists.

- **What is the `-i` flag and how do you choose the sampling interval?**
  - `-i` sets the sampling interval in nanoseconds (or with suffix: `us`, `ms`, `s`). Default CPU interval is 10ms (99 Hz). For short-lived methods (<1ms), decrease the interval to 1ms (1000 Hz) for better resolution, at the cost of higher overhead. For long-running methods, the default interval is sufficient. The interval should be short enough to capture at least 1000 samples in the profiling duration for statistical significance: `-i 1ms` on a 30-second profile yields ~30,000 samples.

## Developer Recommendations

- **Start every investigation with wall-clock profiling, not CPU profiling** — Wall-clock captures all thread states (runnable, blocked, sleeping). It immediately shows whether the bottleneck is CPU (threads are runnable) or I/O (threads are blocked). CPU profiling only helps when you already know the bottleneck is CPU.
  - Implementation: `profiler.sh -d 30 -e wall -o flamegraph -f wall.svg <pid>`.
  - **Production story:** A team spent 2 weeks optimizing a method that appeared hot in the CPU profile. Switching to wall-clock showed the threads were actually blocked on a database connection pool timeout. The CPU profile was misleading because only the non-blocked threads showed up, making their CPU usage look disproportionately large.

- **Always run async-profiler in the "gap" between deployments** — When you suspect a regression between build N and build N+1, profile both in the same environment (production or staging). Use differential profiling (`difffolded.pl`) to identify exactly which methods changed.
  - Implementation: Build a CI step that profiles the performance test run, stores `.collapsed` files per build, and generates a diff SVG on demand.
  - **Production story:** A JDK minor update (11.0.12 → 11.0.13) caused a 15% throughput regression. Differential profiling revealed the JIT had changed its inlining strategy for a hot loop. The fix was adding a `@ForceInline` comment on a key method (enforced with `-XX:CompileCommand=inline,...`).

- **Profile allocation rate before tuning GC** — Many teams jump to GC tuning (heap size, collector selection, pause targets) without understanding the allocation rate. If your allocation rate is 2 GB/s, no GC tuning will save you — the JVM must allocate and free that 2 GB/s. async-profiler's `-e alloc` shows exactly which methods allocate.
  - Implementation: Profile allocation for 30 seconds: `profiler.sh -d 30 -e alloc -o flamegraph -f alloc.svg <pid>`. Target the top allocation sites.
  - **Production story:** A team doubled heap size to reduce GC frequency. The GC time barely decreased because the allocation rate was 3 GB/s and a larger heap just delayed the inevitable GC. Allocation profiling showed 80% of allocations came from a single `String.format()` call in a logging statement. Removing the format call reduced allocation to 500 MB/s and GC time dropped 80%.

- **Use JFR output mode for rich visualization in JMC** — `profiler.sh -d 30 -o jfr -f profile.jfr <pid>` produces a JFR file that opens in Java Mission Control. This gives you JMC's event navigation, thread analysis, and automated diagnostics alongside async-profiler's sampling data.
  - **Avoid when:** You need the raw flame graph SVG immediately — JFR files require JMC to open and may not be available on the investigation system.

- **For containerized environments, prefer `-e itimer` when you cannot get `perf_events` access** — The 2% overhead trade-off is acceptable for short profiling sessions. Do not rely on `-e wall` as a workaround for CPU profiling — wall-clock includes idle threads and dilutes the CPU signal.
  - Implementation: `profiler.sh -d 30 -e itimer -o flamegraph -f cpu-itimer.svg <pid>`.

- **Store collapsed profiles in a time-series database for trend analysis** — Convert flame graphs to collapsed format and store them per-deployment. A 10% increase in a specific method's sample share across builds signals a regression before it affects p99 latency.
  - Implementation: The collapsed format is one line per stack trace with a sample count. Index by method name in your observability pipeline and alert on per-method sample count changes > 15%.

- **Do not rely solely on sampling rate for accuracy — understand the "inverse" problem** — async-profiler samples at 99 Hz. If a method takes 1ms to execute and is called 100 times/second, it appears in ~10% of samples. But if it is called 1000 times/second, it is ~100% of samples. Sampling accuracy depends on both the method's execution time AND its frequency. A method that appears rarely may be slow but infrequent, not fast.
  - Implementation: Cross-reference flame graph width with method invocation counts from metrics (Micrometer, Prometheus). A 1% wide method called 1M times/second deserves more attention than a 50% wide method called 10 times/second.
