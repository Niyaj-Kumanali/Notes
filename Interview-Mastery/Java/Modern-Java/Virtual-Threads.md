# Virtual Threads

- Purpose: lightweight threads enabling high-concurrency server applications
- Platform thread: OS-managed thread with a 1:1 mapping to kernel thread, expensive and limited
- Virtual thread: JVM-managed thread with many-to-one mapping to carrier threads, cheap and plentiful
- Virtual threads are mounted on carrier platform threads during execution and unmounted when blocking
- Mounting and unmounting happens transparently on blocking IO operations
- Use cases: IO-bound workloads like web servers, REST calls, database queries, file reads
- One-thread-per-request model becomes viable even with thousands of concurrent requests
- Avoid synchronized blocks and methods because pinning prevents unmounting
- Avoid ThreadLocal with many virtual threads as it defeats the lightweight memory advantage
- CPU-bound workloads are not helped; use platform threads or parallelism instead
- Structured concurrency via StructuredTaskScope groups related tasks and propagates cancellation
- StructuredTaskScope.ShutdownOnFailure for fail-fast patterns with subtasks
- More intuitive than reactive programming for most business logic
- JDK 21 finalized as production-ready
