# Java Flight Recorder

## Overview

- **Definition** — Java Flight Recorder (JFR) is a built-in, low-overhead event-recording framework for profiling and diagnostics embedded in the HotSpot JVM.
- **Why It Exists** — Traditional profilers (YourKit, JProfiler) use JVMTI agents that cause significant overhead (5–20%) and require JVM restart to attach. JFR runs natively inside the JVM with <1% overhead, can be started/stopped at runtime via `jcmd`, and records a continuous circular buffer of events that is dumped on demand — making it safe for always-on production use.
- **Historical Context** — JFR was introduced in Oracle JDK 7u4 (2012) as a commercial feature requiring a license key. Microbenchmarking with JMH required `-XX:+UnlockCommercialFeatures -XX:+FlightRecorder`. It was open-sourced in OpenJDK 11 (2018) and is now free and available in any JDK build. JDK 14 added event streaming (JDK 14), enabling real-time consumption without dump files.
- **Key Concepts** — **Events** are the core data unit (instant events like GC pause, duration events like socket read). **Recordings** are named configurations of events to collect. **`jcmd`** is the primary CLI tool to start/stop/dump recordings. **JMC** (Java Mission Control) is the GUI for viewing recordings. **Event streaming** (JDK 14+) allows consuming events in real time via `RecordingStream`. **`jfr`** CLI tool converts/flights recording files (`.jfr` to JSON, FLAC, etc.). **Overhead** is typically <1% with default settings. **Template files** (`.jfc`) define which events to collect and their thresholds.

## Core Concepts

- **Event types** — JFR defines three event shapes: **instant** (occurs at a point in time, e.g., thread start), **duration** (has a start and end time, e.g., GC pause), and **timed** (a duration event that is only recorded if it exceeds a threshold, e.g., socket read >10ms). Duration events with thresholds prevent recording every trivial operation.

- **Recording modes** — **Continuous recording** writes events to a fixed-size circular buffer in memory (default 1–10 minutes), keeping the most recent data. Older events are overwritten when the buffer wraps. **Dump** writes the buffer to a `.jfr` file on demand or on JVM exit. **Profiling recording** adds high-overhead events (allocation stack traces, native method sampling) that are not suitable for always-on use but are invaluable during targeted investigations.

- **Event categories** — JFR events are grouped into categories: **Java Application** (thread sleep, socket I/O, file I/O, method profiling via `-XX:StartFlightRecording`), **JVM Internals** (GC events, JIT compilation, class loading, safepoint, biased locking revocation), **OS Events** (CPU load, context switches), **JDK Libraries** (HTTP client events, TLS handshake, ZGC phases), and **Custom Events** (user-defined `@Event` classes).

- **`jcmd` commands** — The primary control interface:
  ```
  # Start a continuous 60-minute recording named "production-recording"
  jcmd <pid> JFR.start name=production-recording duration=60m settings=profile
  
  # Dump the recording to a file
  jcmd <pid> JFR.dump name=production-recording filename=recording.jfr
  
  # Stop and discard
  jcmd <pid> JFR.stop name=production-recording
  ```
  - Also supports `filename=` on start for automatic file writing, `maxage=` and `maxsize=` for buffer management.

- **Template files (`.jfc`)** — XML files that select event types and set thresholds. JDK ships two templates: **`default.jfc`** (safe for always-on, <1% overhead, ~100 events) and **`profile.jfc`** (adds allocation stack traces, method sampling, and method profiling; ~2-5% overhead). Custom templates can enable specific events (e.g., track all HTTP client calls below 100ms).

- **Event streaming (JDK 14+)** — Instead of dumping files, consume events programmatically:
  ```java
  try (var rs = new RecordingStream()) {
      rs.enable("jdk.GCPause").withThreshold(Duration.ofMillis(10));
      rs.onEvent("jdk.GCPause", event -> {
          System.out.println("GC pause: " + event.getDuration());
      });
      rs.start(); // blocks
  }
  ```
  - Enables real-time alerting, auto-tuning, and metrics pipelines without disk I/O or file management.

- **Custom events** — Define your own events by extending `jdk.jfr.Event`:
  ```java
  @Label("Credit Card Charge")
  @Category({"Business", "Payment"})
  public class ChargeEvent extends Event {
      @Label("Amount")
      private double amount;
      
      @Label("Merchant ID")
      private String merchantId;
  }
  
  // In business logic:
  var event = new ChargeEvent();
  event.amount = 49.99;
  event.merchantId = "merchant-123";
  event.commit();
  ```
  - Custom events appear in JMC alongside JVM events. They support `@Period` (auto-committing at a fixed rate), `@StackTrace` (control whether stack trace is captured), and `@Threshold` (only record if duration exceeds).

- **`jfr` CLI tool** — Post-processes `.jfr` files:
  ```
  jfr print --events jdk.GCPause recording.jfr
  jfr summary recording.jfr
  jfr metadata recording.jfr
  ```
  - Can convert to JSON, XML, or CSV for pipeline integration.

- **Overhead characteristics** — JFR uses a lock-free, wait-free event queue per thread. Events are written to thread-local buffers that flush to the global buffer on contention or threshold. This design ensures that JFR never blocks application threads. The overhead profile: `default.jfc` → <1% CPU, negligible allocation; `profile.jfc` → 2-5% CPU (allocation stack traces are expensive on allocation-heavy workloads).

- **JMC integration** — Java Mission Control opens `.jfr` files with a rich GUI: event browser, flame graph, heap histogram, thread analysis (locks, blocked times, I/O), and automated "problem" detection (high GC pause, excessive safepoint time, biased locking contention). JMC can also connect to a running JVM via JMX to trigger recordings.

## Common Mistakes

- **Using `profile.jfc` as the always-on template**
  - Enabling allocation-heavy profiling in production permanently, causing 5% CPU overhead on allocation-hot paths.
  - **Why it looks correct:** The profile template is named "profile" and seems like the right choice for understanding performance. The overhead seems acceptable initially.
  - Use `default.jfc` for continuous monitoring. Switch to `profile.jfc` only for targeted investigations (15-30 minutes), then revert.

- **Not setting `maxage` or `maxsize` on continuous recordings**
  - Starting `JFR.start` without specifying retention bounds, letting the circular buffer grow until it exhausts memory or uses excessive disk.
  - **Why it looks correct:** The recording starts and runs; there is no immediate error. The memory consumption of the JFR buffer is invisible in heap profiling.
  - Always set `maxage=1h` and `maxsize=500MB` on continuous recordings. The buffer wraps, keeping the most recent data within those bounds.

- **Dumping recordings too late after an incident**
  - Waiting until a support ticket is filed to dump the JFR recording, by which time the circular buffer has wrapped past the relevant event window.
  - **Why it looks correct:** The recording is always running. Engineers assume they can dump "after the fact" without realizing the buffer only holds the last N minutes.
  - Set a monitoring trigger that dumps the recording on anomaly detection (latency spike, error rate increase). Use event streaming for real-time alerting so the buffer is preserved for post-mortem.

- **Ignoring event threshold settings**
  - Enabling `jdk.SocketRead` without setting a threshold, recording every TCP read (millions per second on a busy server), overwhelming the buffer with trivial events and losing rare high-latency reads.
  - **Why it looks correct:** The event is enabled and data appears in the recording. It seems comprehensive.
  - Set threshold on all duration events: `withThreshold(Duration.ofMillis(10))` captures only reads that took meaningful time. Without threshold, JFR becomes unusably noisy.

- **Not defining custom events for business logic**
  - Using log statements with timestamps to diagnose business-level latency (checkout time, payment processing, order fulfillment), then manually correlating log lines.
  - **Why it looks correct:** Logging is the universal tool, and JFR seems like a JVM-level tool, not a business-level tool.
  - Define custom `@Event` classes for every business operation that has a latency or throughput concern. They appear directly in JMC alongside GC pauses and socket reads, making root cause analysis immediate.

## Real-World Scenarios

### Diagnosing latency spikes with continuous recording

- A payment processing service has intermittent p99 latency spikes from 50ms to 2s. GC logging shows no long pauses. Thread dumps during the spike are inconclusive. A JFR continuous recording (`default.jfc`, 30-minute buffer) is deployed. When the spike recurs, the recording is dumped. The event browser reveals that `jdk.SocketRead` events in the database connection pool have a p99 of 1.8s, with a threshold of 200ms — the database is occasionally slow, and the connection pool has no timeout. The fix: configure HikariCP with `connectionTimeout=500ms`.

### Using profile.jfc to diagnose allocation pressure

- A search service's GC frequency increased from 1 minor GC/sec to 10 minor GC/sec after a deployment. JMC opens the profile recording and the allocation profile view shows that the new query parser creates a `StringBuilder` per character of the input query instead of using a reusable buffer. The allocation rate went from 50 MB/s to 800 MB/s. Fix: rewrite the parser to reuse a `StringBuilder` pooled in a `ThreadLocal`. GC frequency drops back to 1/sec.

### Event streaming for real-time autoscaling

- A team uses JFR event streaming to emit custom `CacheHitEvent`/`CacheMissEvent` from their caching layer. A `RecordingStream` subscribes to these events and maintains a hit-ratio sliding window. When the hit ratio drops below 70% for 10 seconds, a metric is incremented and fed into the HPA (Horizontal Pod Autoscaler) to scale the service. JFR's sub-millisecond overhead makes this cheaper than logging every cache access.

### Custom event tracking for payment SLA

- A payment gateway defines a `PaymentProcessingEvent` with fields for amount, merchant, payment method, and duration. The event is committed after every payment. JMC's event browser shows that payments via a specific payment processor have a p99 of 4s compared to 200ms for others. The business analyzes this data and negotiates a new contract with the slow processor. Without custom events, this correlation would require log parsing and cross-referencing with payment provider IDs.

### Always-on production profiling catching a JIT regression

- After a JDK version upgrade, the application's throughput drops 15% with no obvious code change. A continuous JFR recording with default settings shows that `jdk.Compilation` events for the hot method `OrderService.calculateTax()` increased from 50ms to 800ms compilation time. The JIT compiler generates a different optimization strategy for the newer JDK. The team compares the assembly output, identifies a missing inlining optimization, and works around it.

## Use Cases

Reach for JFR when you need production-safe, always-on profiling that can be activated or dumped on demand without restarting the JVM.

- **Always-on continuous monitoring** — Run `default.jfc` in production on every JVM. The 1% overhead is negligible, and the circular buffer captures the last hour of JVM events — GC, safepoints, JIT compilation, thread contention, socket I/O.
  - When an incident occurs, dump the buffer. No need to reproduce — the data is already there.
  - **Avoid when:** The application runs fewer than 1,000 requests/second with no latency requirements — the operational complexity of JFR may not justify the insight.

- **Targeted deep-dive profiling** — Switch to `profile.jfc` for 15–30 minutes when investigating allocation pressure, hot methods, or lock contention. The additional overhead (2-5%) is acceptable for a short window.
  - Use `jcmd <pid> JFR.start name=investigation duration=30m settings=profile` and dump at the end.
  - **Avoid when:** The investigation targets native code or kernel behavior — async-profiler or perf may provide deeper stack traces.

- **Real-time alerting with event streaming** — Use JDK 14+ `RecordingStream` to subscribe to specific events (GC pause > 200ms, socket read > 1s) and trigger alerts or metrics in real time.
  - No dump file needed — the pipeline consumes events as they fire.
  - **Avoid when:** The alert volume is higher than once per second — the Java-based `RecordingStream` adds overhead for high-frequency events.

- **Custom business event tracking** — Define `@Event` classes for business operations (order checkout, payment charge, recommendation computation) and commit them as part of the operation. View them in JMC alongside JVM events.
  - **Avoid when:** The event firing rate exceeds 10,000/second per thread — at high rates, the thread-local buffer flush contention may become measurable.

- **Capacity planning and trend analysis** — Periodically dump JFR recordings (every hour) and archive them. Compare GC frequency, allocation rates, and safepoint times across releases and load levels.
  - **Avoid when:** Storage is limited — `.jfr` files can be tens of MB per dump.

## Scenario-Based Questions

**Q: You deploy JFR with `default.jfc` on all production JVMs. A team reports that the application's throughput dropped 3% after enabling JFR. Is this expected, and what do you investigate?**

- A 3% drop with `default.jfc` is higher than typical (~1%). Investigate: (a) Is the JVM version older than JDK 11 — early JFR implementations had higher overhead. (b) Is the application allocation-heavy — the default template captures allocation stack traces at a low rate but the impact multiplies with very high allocation rates. (c) Are there buggy third-party tools injecting JFR events — check for custom `@Event` classes in the classpath.
- **Interview follow-up:** You find that JFR overhead is 5% on one specific JVM. `jfr summary` shows an unusually high rate of `jdk.ThreadSleep` events. Why would JFR have higher overhead when the application sleeps frequently?

**Q: You dump a JFR recording after a 5-minute latency spike, but the relevant events are not in the file. The recording was started 1 hour before the spike. What happened?**

- The continuous recording uses a circular buffer with a default retention of 1-10 minutes (depending on JVM version and configuration). The spike occurred at 3:00 PM; the dump at 3:05 PM only contains events from 3:00–3:05 — events from before 3:00 PM have been overwritten because the 30-minute buffer wrapped. The fix: set `maxage=1h` to ensure at least 1 hour of events are retained, or use event streaming for real-time alerts that trigger an immediate dump.
- **Interview follow-up:** You set `maxage=1h` but the buffer still contains only 10 minutes of data on a high-traffic server because events are generated at a high rate. What other setting must you adjust?

**Q: A custom `@Event` class is defined in a library JAR. The event is committed in business logic, but it never appears in JFR dumps. What is the most likely cause?**

- JFR custom events must be explicitly enabled — they are not enabled by default in `default.jfc` or `profile.jfc`. Enable them via `jcmd <pid> JFR.start name=myrecording settings=profile` or create a custom `.jfc` template that includes the event. Alternatively, start the recording programmatically with `Recording.enable("com.example.MyEvent")`.
- **Interview follow-up:** You add the custom event to the template, restart the recording, but the event still does not appear. You confirm the event class is on the classpath and the commit() code path is executed. What JVM flag might be preventing the event from being registered?

**Q: A JFR recording file from production cannot be opened in JMC — it shows "corrupted recording" or truncation. What are the possible causes?**

- The file was not properly closed (JVM crashed before `JFR.stop` or `JFR.dump` completed). Use `jfr metadata <file>` to check integrity. For crash-safe dumps, enable automatic dump on JVM crash: `-XX:-OmitStackTraceInFastThrow` is unrelated, but `-XX:CrashDump` can trigger JFR dump. Alternatively, use event streaming to consume events in real time instead of relying on dump files for crash recovery.
- **Interview follow-up:** You enable automatic JFR dump on OOM using `-XX:StartFlightRecording=filename=crash.jfr,dumponexit=true`, but on OOM the recording file is truncated. The JVM is killed by the OOM killer (SIGKILL), not Java's OOM error. What happens to the JFR dump in this scenario?

**Q: JFR event streaming is consuming events for `jdk.GCPause`. The stream runs in a separate thread and processes 10,000 events/second. After 1 hour, the application's GC behavior changes — minor GC pauses increase from 10ms to 30ms. Is the event stream the cause?**

- JFR event streaming with `RecordingStream` runs in the same JVM and consumes events from the same lock-free queue. At 10K events/second, the overhead is non-trivial but typically <1%. The GC pause increase from 10ms to 30ms is likely caused by something else — check if the stream thread is triggering GC by allocating during event processing, or if the application's allocation rate increased. Profile both with and without the stream to isolate the impact.
- **Interview follow-up:** You profile with the stream disabled and the GC pause is still 30ms. What JFR event type would you enable to determine what changed in the GC behavior between the two time windows?

**Q: You need to record a custom business event every time an expense report is submitted. The event includes the employee ID, expense amount, and processing time. How do you implement this with JFR?**

```java
@Label("Expense Report")
@Category({"Business", "Expense"})
public class ExpenseReportEvent extends Event {
    @Label("Employee ID")
    private String employeeId;
    
    @Label("Amount")
    private double amount;
}

// In submission code:
var event = new ExpenseReportEvent();
event.employeeId = user.getId();
event.amount = report.getTotal();
event.begin();
// ... processing ...
event.commit();
```
- **Interview follow-up:** The expense report processing involves three internal microservice calls. How would you design the JFR events to capture the full processing chain so JMC can show a waterfall view of the total processing time?

**Q: A team uses `jcmd <pid> JFR.start settings=profile`. After 30 minutes, the JVM is unresponsive for 2 seconds. The team blames JFR. Is JFR the likely cause?**

- JFR with `profile.jfc` adds 2-5% CPU overhead but never causes multi-second application pauses. The 2-second pause is likely a GC Full STW pause (old gen collection or concurrent mode failure) that happened to coincide with the JFR recording. Use the JFR recording itself to investigate: check `jdk.GCPause` events for duration, `jdk.Safepoint` for time spent in safepoint, and `jdk.JavaMonitorWait` for lock contention. JFR is the diagnostic tool, not the cause.
- **Interview follow-up:** You open the JFR recording and see a `jdk.VMOperation` event with duration 2.1s. What JVM operation takes that long and is it JFR-related?

**Q: You want to monitor HTTP client call latencies in production with JFR. The JDK HttpClient emits JFR events (`jdk.HttpClientRequest`, `jdk.HttpClientResponse`). What is the minimum JFR configuration to capture calls that take longer than 500ms?**

- Enable the events with a threshold: create a custom `.jfc` template that includes `jdk.HttpClientRequest` and `jdk.HttpClientResponse` with `threshold="500 ms"`. Or use event streaming:
  ```java
  rs.enable("jdk.HttpClientRequest").withThreshold(Duration.ofMillis(500));
  rs.onEvent("jdk.HttpClientRequest", e -> alertIfSlow(e));
  ```
- **Interview follow-up:** You enable the events but they never fire. After investigation, the application uses Apache HttpClient, not JDK's built-in HttpClient. What are your options to instrument the Apache HttpClient with JFR?

**Q: A JFR continuous recording shows 10% safepoint time (`jdk.Safepoint`). The application's throughput is 20% lower than expected. What do you investigate?**

- High safepoint time (>1-2%) indicates the JVM is spending excessive time stopping all threads. Common causes: (a) Biased locking revocation — `-XX:+UseBiasedLocking` (enabled by default in JDK 8-15, deprecated in JDK 15, disabled in JDK 17+) can cause safepoints when locks are revoked. (b) System.gc() calls. (c) JVMTI agents requesting safepoints. (d) Thread dump requests. Check `jdk.Safepoint` events for the reason field. Use `-XX:+UnlockDiagnosticVMOptions -XX:+PrintSafepointStatistics` for detailed safepoint analysis.
- **Interview follow-up:** The safepoint reason is "BulkRevokeBias". You are on JDK 17 where biased locking is disabled by default. What other JVM configuration or library dependency could still cause bulk revocations?

**Q: You configure JFR with `-XX:StartFlightRecording=maxage=1h` on a JVM with 8 GB heap. After 4 hours of continuous operation, the JVM crashes with `OutOfMemoryError: Java heap space`. The heap dump shows no application-level leak. Could JFR have caused the OOM?**

- JFR's circular buffer is allocated outside the Java heap (native memory). The `maxage=1h` setting only controls event retention time, not the buffer size. If the event generation rate is very high (e.g., allocation profiling enabled, high-throughput socket I/O), the native buffer can grow to consume significant native memory. Check `-XX:FlightRecorderBufferSize` (default 64 KB per thread). With many application threads, the total native memory for JFR buffers can reach hundreds of MB. The OOM is likely not JFR itself but the combination of JFR buffer size + other native memory usage (direct buffers, thread stacks, code cache) exhausting the process address space. Fix: set explicit `maxsize` (e.g., `maxsize=500MB`) to cap the buffer, or reduce the number of enabled events.
- **Interview follow-up:** You cap the JFR buffer at 500 MB and the OOM disappears. However, during traffic spikes, the recording now only contains 2 minutes of events instead of 1 hour. The team needs 1 hour of retention for compliance. How do you satisfy both constraints?

## Interview Questions

- **What is Java Flight Recorder and how does it differ from traditional profilers?**
  - JFR is a built-in, low-overhead (sub-1%) event-recording framework that runs inside the JVM. Unlike JVMTI profilers (YourKit, JProfiler), JFR does not require agent attachment, can be started/stopped at runtime via `jcmd`, and is safe for always-on production use.

- **What is the overhead of JFR and how is it achieved?**
  - Default template: <1% CPU. Profile template: 2-5% CPU. JFR uses lock-free, wait-free per-thread event queues and thread-local buffers. Events are written without synchronization in the fast path; only buffer flushes require a CAS operation.

- **What is the difference between `default.jfc` and `profile.jfc`?**
  - `default.jfc` includes ~100 event types suitable for always-on use (GC, safepoint, JIT, thread, socket I/O with thresholds). `profile.jfc` adds allocation stack traces, method profiling, and native method sampling — enabling heap allocation hot-path analysis but with higher overhead.

- **How do you start and stop a JFR recording from the command line?**
  - `jcmd <pid> JFR.start name=recording duration=60s settings=default` to start. `jcmd <pid> JFR.dump name=recording filename=dump.jfr` to dump. `jcmd <pid> JFR.stop name=recording` to stop.

- **What is event streaming and when was it introduced?**
  - Event streaming (JDK 14+) allows consuming JFR events in real time via `jdk.jfr.consumer.RecordingStream` without dumping to a file. Enables real-time alerting, metrics, and auto-tuning pipelines.

- **How do you define a custom JFR event?**
  - Extend `jdk.jfr.Event`, annotate with `@Label`, `@Category`, `@Description`. Add fields with `@Label`. Call `event.commit()` to record. Custom events appear in JMC alongside JVM events.

- **What is a `.jfc` file?**
  - An XML file that defines which event types JFR collects and their thresholds. JDK ships `default.jfc` and `profile.jfc`. Custom templates enable targeted event selection (e.g., only HTTP client events above 500ms).

- **What types of JFR events exist?**
  - Instant (occur at a point in time — thread start, GC phase change), Duration (have a start and end — socket read, GC pause), Timed (duration events recorded only if they exceed a configurable threshold).

- **How does JFR handle thread-local buffering?**
  - Each thread writes events to a thread-local buffer. On exhaustion, the thread flushes to the global buffer. Flush uses a CAS-based linked-queue to avoid blocking. This design ensures JFR never blocks application threads.

- **How do you view a JFR recording?**
  - Java Mission Control (JMC) is the primary GUI. The `jfr` CLI tool prints events in text, JSON, or XML. Third-party tools (IntelliJ Profiler, VisualVM) also support `.jfr` format.

- **What is the `jfr` CLI tool?**
  - A post-processing tool for `.jfr` files. Commands include `jfr print` (print events), `jfr summary` (statistics), `jfr metadata` (event type definitions), `jfr view` (specific view like GC or thread).

- **How does JFR differ from GC logs?**
  - GC logs only cover GC events. JFR covers everything: GC, JIT compilation, thread contention, safepoints, socket I/O, file I/O, class loading, and custom business events. JFR events are structured (typed fields) while GC logs are text.

- **What JVM flags enable JFR?**
  - `-XX:StartFlightRecording=name=recording,duration=60s,settings=default,filename=recording.jfr,dumponexit=true`. This starts a recording on JVM start that dumps the buffer on exit.

- **What is the effect of `dumponexit=true`?**
  - The JFR recording is automatically dumped to the specified file when the JVM exits normally. It does NOT guarantee a dump on crash — a JVM crash (SIGSEGV, SIGKILL) may truncate or lose the recording.

- **How do you convert a `.jfr` file to human-readable text?**
  - `jfr print recording.jfr` prints all events. `jfr print --events jdk.GCPause recording.jfr` filters to specific event types. Output can be JSON or XML with `--json` or `--xml`.

- **What is the relationship between JFR and JMC?**
  - JFR generates the data; JMC consumes and visualizes it. JMC opens `.jfr` files, connects to running JVMs via JMX, and provides heap analysis, thread analysis, flame graphs, and automated problem detection.

- **How do you enable JFR for a Spring Boot application?**
  - Add `-XX:StartFlightRecording=name=spring-boot-prod,settings=default,maxage=1h,maxsize=500MB` to `JAVA_OPTS`. Or start it via `jcmd` on the running PID.

- **What events does JFR provide for HTTP client monitoring?**
  - JDK 11+ HttpClient emits `jdk.HttpClientRequest` (duration event with URI, method) and `jdk.HttpClientResponse` (response code, duration).

- **Can JFR be used with containerized environments like Docker and Kubernetes?**
  - Yes. JFR works inside containers. Use `jcmd` inside the container, or enable `-XX:StartFlightRecording` in the JVM args. For Kubernetes, set `JAVA_TOOL_OPTIONS` with the JFR flags, or use ephemeral containers to run `jcmd`.

- **What are the limitations of JFR in JDK 21?**
  - Method profiling is sampling-based, not exact — very short methods may be missed. No native memory tracking (NMT is separate). Event streaming is Java-only (not usable from Kotlin or Scala as elegantly). No kernel-level events (context switches, page faults) — that requires async-profiler or perf.

## Developer Recommendations

- **Run JFR with `default.jfc` on every production JVM** — The 1% overhead is negligible compared to the visibility it provides. When an incident occurs, you have the last hour of JVM events ready to dump. Without it, you are debugging blind.
  - Implementation: Add `-XX:StartFlightRecording=name=continuous,settings=default,maxage=1h,maxsize=500MB` to the JVM start command.
  - **Production story:** A team spent 3 weeks trying to reproduce a sporadic 2-second latency spike. They enabled JFR continuous recording and caught the event the same day — the spike was caused by a JIT compilation of a large method that was inlined incorrectly after a JDK update.

- **Write a custom `.jfc` template for your specific workload** — The default and profile templates are generic. Your application has unique performance characteristics that deserve tailored event selection.
  - Implementation: Copy `default.jfc` from the JDK, enable events relevant to your stack (HTTP client events, thread pool events, custom events), disable events you never use, and set thresholds aligned with your latency budget.

- **Use event streaming for real-time observability** — Dumping files after the fact is useful for post-mortems. Event streaming enables real-time alerting — trigger a page when GC pause exceeds 200ms or a socket read exceeds 1s.
  - Implementation: Deploy a lightweight Java agent or sidecar that subscribes to key events via `RecordingStream` and emits Prometheus metrics or alerts.
  - **Production story:** A fintech company used event streaming to detect when their database connection pool's socket read latency exceeded 500ms. They built an auto-remediation step that kills and restarts the connection pool, reducing p99 from 2s to 100ms.

- **Always set `maxage` and `maxsize` on continuous recordings** — Without bounds, the circular buffer grows indefinitely or wraps too quickly. Set both to ensure predictable memory and data retention.
  - Implementation: `maxage=1h` retains at least 1 hour of events. `maxsize=500MB` limits the total buffer size. The buffer wraps within these constraints.

- **Define custom events for every critical business operation** — If a business operation matters enough to monitor (payment, checkout, recommendation), it matters enough to instrument with a custom JFR event.
  - Implementation: Create a `@Event` class per operation. Commit the event after completion. The events appear in JMC alongside JVM events, making cross-correlation (high GC pause coinciding with slow checkout) immediate.

- **Integrate JFR with your CI/CD pipeline for regression detection** — Record JFR during performance tests. Compare the `.jfr` files across builds to detect regressions in allocation rate, GC frequency, JIT compilation time, or safepoint overhead.
  - Implementation: Use `jfr print --json` to extract numeric metrics, and compare them in your CI/CD dashboard. A 10% increase in GC time per request is a meaningful signal.

- **Do not rely on `dumponexit=true` for crash recovery** — A JVM crash (especially SIGKILL from OOM killer) can truncate the dump file. Use event streaming to flush events continuously, or accept that the last N seconds of events may be lost on hard crash.
  - Implementation: Set `dumponexit=true` as a best-effort baseline. For crash-critical events, add a separate event streaming consumer that writes to a network socket.
