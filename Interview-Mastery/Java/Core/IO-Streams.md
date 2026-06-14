# Java I/O Streams

---

## Overview

- **Purpose**
  - Java I/O Streams are a fundamental API for reading from and writing to data sources — files, network sockets, memory buffers, and the system console — using the abstraction of a continuous flow of data.

- **Three Generations**
  - Java provides the original blocking `java.io` stream API (Java 1.0), the `java.nio` buffer-and-channel API with selectors for non-blocking I/O (Java 1.4), and the NIO.2 file system API in `java.nio.file` with asynchronous channels (Java 7).
  - NIO.2 `Path` and `Files` should be your default for all new file I/O code.

  **Why three generations?**
    - Java 1.0's I/O was designed for the thread-per-connection model of the 90s — blocking reads, simple streams, adequate for applets and desktop apps.
    - By Java 1.4, web servers needed to handle 10K+ concurrent connections without 10K threads (each consuming ~1MB of native stack). NIO introduced `Selector` and `Channel` to scale with non-blocking multiplexed I/O.
    - By Java 7, the legacy `java.io.File` class was beyond repair — its boolean return values, symbolic link blindness, and filesystem-ignorant design drove the creation of NIO.2 `Path`, `Files`, and asynchronous channel APIs.
    - Each generation layers on top of the previous: you can still use blocking I/O with NIO channels, and NIO.2 files can be accessed through NIO channels.

- **Memory-Mapped Files**
  - `MappedByteBuffer` maps file regions directly into virtual memory, allowing the OS to handle paging transparently and providing 10-100x speed improvements for large-file random access.

- **Cardinal Rule**
  - Always close resources using try-with-resources (Java 7+), which guarantees that `close()` is called even when an exception is thrown.

```java
try (InputStream in = new FileInputStream("file")) { ... }
```

| API | Package | I/O Model | Since |
|-----|---------|-----------|-------|
| **I/O Streams** | `java.io` | Blocking, byte/char | Java 1.0 |
| **NIO** | `java.nio` | Buffers, channels, selectors | Java 1.4 |
| **NIO.2** | `java.nio.file` | File system API, async channels | Java 7 |
| **Memory-Mapped** | `java.nio.MappedByteBuffer` | File → Direct memory | Java 1.4 |

---

## Byte Streams (8-bit)

- **Purpose**
  - Byte streams handle I/O of raw binary data with `InputStream` as the abstract root for reading and `OutputStream` for writing.
  - `FileInputStream` and `FileOutputStream` read from and write to files, but they should always be wrapped with `BufferedInputStream` / `BufferedOutputStream` to avoid per-byte system calls.

- **ByteArray Streams**
  - `ByteArrayInputStream` and `ByteArrayOutputStream` operate on in-memory byte arrays, useful for testing and data transformation within the same JVM.

- **Data Streams**
  - `DataInputStream` and `DataOutputStream` allow reading and writing Java primitive types (`int`, `long`, `double`) in a portable binary format with `readInt()`, `readLong()`, `readFully()`, and `readUTF()`.
  - The `readFully()` method guarantees the requested number of bytes is read or an `EOFException` is thrown.

- **Object Streams**
  - `ObjectInputStream` and `ObjectOutputStream` enable Java object serialization, converting entire object graphs to and from byte streams.

- **PushbackInputStream**
  - Allows a single byte to be "unread" after reading, useful for lookahead parsing.

| InputStream | OutputStream | Purpose |
|-------------|-------------|---------|
| `FileInputStream` | `FileOutputStream` | File I/O |
| `ByteArrayInputStream` | `ByteArrayOutputStream` | In-memory buffer |
| `BufferedInputStream` | `BufferedOutputStream` | Buffering (always wrap!) |
| `DataInputStream` | `DataOutputStream` | Read/write primitives |
| `ObjectInputStream` | `ObjectOutputStream` | Object serialization |
| `PushbackInputStream` | — | Unread last byte |

```java
// Reading bytes from a file (with buffering)
try (InputStream in = new BufferedInputStream(new FileInputStream("data.bin"))) {
    byte[] buffer = new byte[4096];
    int bytesRead = in.read(buffer);
}
```

---

## Character Streams (16-bit Unicode)

- **Purpose**
  - Character streams handle I/O of character data with `Reader` and `Writer` as the abstract roots, managing character encoding transparently through an internal `Charset` decoder or encoder.

- **FileReader/FileWriter**
  - Convenient for simple text file operations but use the platform default charset, which causes data corruption when files are moved between systems with different default encodings.
  - Always specify the charset explicitly.

- **BufferedReader/BufferedWriter**
  - Add buffering and line-oriented operations — `BufferedReader.readLine()` is the standard idiom for processing text files line by line.

- **Bridge Classes**
  - `InputStreamReader` and `OutputStreamWriter` convert between byte streams and character streams.
  - The charset must always be specified explicitly: `new InputStreamReader(in, StandardCharsets.UTF_8)`.

- **StringReader/StringWriter**
  - Treat a `String` as a character source or sink, useful for testing.
  - `PrintWriter` provides formatted text output with `print()`, `printf()`, and `println()` methods and optional auto-flushing.

```java
// Reading text with proper encoding
try (BufferedReader reader = new BufferedReader(
        new InputStreamReader(new FileInputStream("file.txt"), StandardCharsets.UTF_8))) {
    String line;
    while ((line = reader.readLine()) != null) {
        System.out.println(line);
    }
}
```

---

## NIO.2 File API (Preferred for File I/O)

- **Path and Files**
  - The modern file API in `java.nio.file` (Java 7+) provides `Path` (an immutable, cross-platform file path) and `Files` (a utility class with static methods for all file operations).
  - `Path.of()` creates paths without hardcoded separators, making code platform-independent.

- **Reading Files**
  - `Files.readString()` (Java 11+) loads small files into a `String` in one line, while `Files.lines()` returns a lazy `Stream<String>` for processing large files without loading them entirely into memory.

- **Writing Files**
  - `Files.write()` and `Files.writeString()` handle writing with charset control.
  - Copying, moving, and deleting files use `Files.copy()`, `Files.move()`, and `Files.delete()` with options like `StandardCopyOption.REPLACE_EXISTING` and `ATOMIC_MOVE`.

- **Directory Traversal**
  - `Files.walk()` and `Files.find()` return `Stream<Path>` for efficient filtering and processing of directory trees.

- **Recommendation**
  - Always prefer NIO.2 `Files` + `Path` over legacy `java.io.File` for all new code.
  - The NIO.2 API is more consistent, supports symbolic links, throws meaningful exceptions instead of returning boolean status codes, and provides lazy streaming operations for large files.

```java
// Reading
Path path = Path.of("file.txt");
List<String> lines = Files.readAllLines(path, StandardCharsets.UTF_8);  // Small files
String content = Files.readString(path);  // Java 11+

// Streaming (large files — lazy)
try (Stream<String> lines = Files.lines(path)) {
    lines.filter(l -> l.contains("ERROR")).forEach(System.out::println);
}

// Writing
Files.writeString(path, "content");  // Java 11+
Files.write(path, lines);

// Copy, move, delete
Files.copy(source, target, StandardCopyOption.REPLACE_EXISTING);
Files.move(source, target);
Files.delete(path);

// Walk file tree
try (Stream<Path> walk = Files.walk(rootDir)) {
    walk.filter(Files::isRegularFile)
        .filter(p -> p.toString().endsWith(".java"))
        .forEach(System.out::println);
}
```

---

## Buffering

- **Critical Optimization**
  - Buffering is the single most important optimization for I/O performance. Each system call has overhead from privilege level switching, context switching, and cache effects.

- **Problem**
  - Reading one byte at a time with `InputStream.read()` causes one system call per byte, translating to 1 billion system calls to read a 1GB file.

- **Solution**
  - `BufferedInputStream` wraps an input stream with an internal 8192-byte buffer, reading from the source in large chunks and serving individual bytes from the in-memory buffer.
  - This reduces system calls from 1 billion to approximately 125K for the same 1GB file — a 10,000x reduction.

- **Default Buffer Size**
  - 8192 bytes is sufficient for most use cases. Larger buffers (64KB) can help on high-latency storage like network file systems.
  - Always wrap streams with `BufferedInputStream` for binary data and `BufferedReader` for text data.

  **Why 8192?**
    - The default 8192 bytes is exactly 2 × 4096 (the standard OS page size).
    - This alignment means most reads naturally span two page boundaries, allowing the OS to prefetch the next page while processing the current one.
    - Smaller buffers (512 bytes) increase system call frequency. Larger buffers (64KB) provide diminishing returns for sequential access because modern storage subsystems already batch at the kernel level.
    - The 8192 default is a sweet spot that works well across HDDs, SSDs, and network file systems with no tuning required.

```java
// BAD: One syscall per byte — extremely slow
FileInputStream in = new FileInputStream("file");
int b;
while ((b = in.read()) != -1) { /* process b */ }

// GOOD: Buffered — reads in large chunks
try (BufferedInputStream in = new BufferedInputStream(new FileInputStream("file"))) {
    byte[] buffer = new byte[8192];
    int bytesRead;
    while ((bytesRead = in.read(buffer)) != -1) { /* process buffer */ }
}
```

---

## Under the Hood: Blocking vs Non-Blocking I/O

- **Blocking I/O**
  - The calling thread blocks until the I/O operation completes — the thread is parked in the kernel and cannot do any other work.
  - Each concurrent connection requires its own thread, and with the thread-per-connection model, the system cannot scale beyond a few thousand connections because each thread consumes approximately 1MB of native stack memory.

- **NIO Non-Blocking I/O**
  - Uses a `Selector` that monitors multiple `Channel` instances, allowing a single thread to manage thousands of concurrent connections by processing only those channels that are ready for read or write operations.
  - The selector thread calls `select()` which blocks until at least one channel is ready, then iterates through the ready `SelectionKey` instances.

- **Memory-Mapped Files**
  - A file region is mapped into the process's virtual memory address space, and the OS handles paging between disk and memory transparently.
  - File access appears as simple memory read and write operations with performance that can be 10-100x faster than traditional `read()` and `write()` for large-file random access patterns.

```java
Selector selector = Selector.open();
SocketChannel channel = SocketChannel.open();
channel.configureBlocking(false);
channel.register(selector, SelectionKey.OP_READ);

while (true) {
    selector.select();  // Blocks until at least one channel is ready
    for (SelectionKey key : selector.selectedKeys()) {
        if (key.isReadable()) { /* read from channel */ }
    }
}
```

```java
FileChannel channel = FileChannel.open(path, StandardOpenOption.READ);
MappedByteBuffer buffer = channel.map(
    FileChannel.MapMode.READ_ONLY, 0, channel.size());
```

---

## Common Mistakes

- **Not Closing Resources**
  - Unclosed file handles accumulate until the process reaches the OS file descriptor limit (typically 1024 on Linux), causing `IOException: Too many open files`.
  - Always use try-with-resources for any `InputStream`, `OutputStream`, `Reader`, `Writer`, `Channel`, or `Stream<Path>` returned by `Files.lines()` or `Files.walk()`.

- **Omitting the Charset**
  - `FileReader` and `FileWriter` use the platform default charset — on US Windows this is `windows-1252`, on Linux it is typically `UTF-8`.
  - A file written on one platform may be unreadable on another. Always specify `StandardCharsets.UTF_8` explicitly.

- **Reading Without Buffering**
  - Each unbuffered `read()` or `write()` call translates to a system call, and for files processed byte by byte, the overhead of millions of kernel context switches dominates the I/O time.
  - **Why it looks correct:** The code reads data and processes it — the API works, returns correct values, and shows no error. The performance problem only reveals itself under load or with large files. A 500MB file read one byte at a time takes approximately 45 minutes; wrapping with `BufferedInputStream` reduces this to under one minute — a 45x improvement that comes from changing one line of code.

- **Ignoring Partial Reads**
  - `InputStream.read(buffer)` is not guaranteed to fill the buffer — it returns the number of bytes actually read, which may be less than the array length.
  - Only `DataInputStream.readFully()` guarantees the requested number of bytes.
  - Processing partial buffers without checking the return value leads to processing stale data from previous reads.

- **TOCTOU Race Condition**
  - Using `File.exists()` before accessing a file introduces a Time-of-Check-Time-of-Use race condition — the file could be deleted between the check and the open call.
  - **Why it looks correct:** Defensive precondition checks are a standard practice for validating inputs before processing. But file system state can change between any two operations, unlike in-memory state which is single-threaded and predictable. Instead, attempt the operation directly and handle the `FileNotFoundException` or `NoSuchFileException`.

- **Not Flushing After Writes**
  - `BufferedOutputStream`, `BufferedWriter`, and `PrintWriter` buffer data in memory. If the JVM crashes before the buffer is flushed to disk, data is silently lost.
  - Always call `flush()` before critical checkpoints or close the stream properly (try-with-resources calls `close()` which flushes).
  - For transactional writes, use `Files.write()` which opens, writes, flushes, and closes atomically.

- **Assuming available() Returns File Size**
  - `InputStream.available()` returns the number of bytes that can be read *without blocking*, not the total file size.
  - For a `SocketInputStream`, `available()` may return 0 even though data is arriving.
  - For a `FileInputStream`, it returns the remaining bytes in the file — but only for local files, not for network or pipe streams.
  - Always use `Files.size()` for file length or read in a loop until `read()` returns -1.

---

## Real-World Scenarios

### Scenario 1: High-Throughput Log Ingestion Pipeline

A logging system ingests 50GB of application logs per day from 200 microservices. Logs arrive as gzipped files over HTTP. The system must parse, filter, and index each line with minimal memory footprint because multiple files may be processed concurrently on a server with limited RAM.

```java
public class LogIngestor {
    public void ingest(Path logFile) throws IOException {
        try (InputStream gzip = new GZIPInputStream(Files.newInputStream(logFile));
             BufferedReader reader = new BufferedReader(new InputStreamReader(gzip, StandardCharsets.UTF_8))) {

            String line;
            while ((line = reader.readLine()) != null) {
                LogEntry entry = parse(line);
                if (entry.severity() >= Severity.WARN) {
                    indexer.index(entry);
                }
            }
        }
    }
}
```

- `GZIPInputStream` wraps the underlying `FileInputStream` to decompress gzip data on the fly, avoiding the need to decompress the entire file to disk first.
- `BufferedReader` wraps the `InputStreamReader` for line-based reading with internal 8KB buffering, ensuring that system call overhead is amortized across many lines.
- The entire pipeline reads one line at a time — memory consumption stays at approximately 8KB for the buffer plus the size of one line, regardless of whether the file is 10MB or 10GB.
- Without buffering, each `readLine()` call would cause a system call, and without streaming decompression, the entire decompressed file (potentially 500MB) would need to fit in memory.

**Why this approach?**
  - A naive alternative would decompress the gzip file to a temporary file on disk (`GZIPInputStream` in → `FileOutputStream` out), then re-read the temp file for parsing.
  - This doubles disk I/O (write decompressed data, then read it back) and requires free disk space equal to the decompressed file size (up to 10GB).
  - The streaming pipeline avoids this entirely — data moves from disk → kernel buffer → decompression → character decoding → line parsing in one pass with no intermediate storage.
  - The decorator pattern (`GZIPInputStream` wraps `FileInputStream`, `BufferedReader` wraps `InputStreamReader`) makes each concern independently testable: you can test log parsing with a `StringReader`, decompression with a `ByteArrayInputStream`, and the full pipeline by providing a test gzip file.

### Scenario 2: Multipart File Upload with Progress

A web application allows users to upload large video files up to 2GB. The server must stream the file directly to disk without loading it entirely into memory, and it must track upload progress to display a progress bar to the user.

```java
@PostMapping("/upload")
public ResponseEntity<String> handleUpload(HttpServletRequest request) throws IOException {
    try (InputStream in = request.getInputStream();
         FileOutputStream out = new FileOutputStream(uploadDir.resolve(filename).toFile())) {

        byte[] buffer = new byte[8192];
        int bytesRead;
        long totalBytes = 0;
        while ((bytesRead = in.read(buffer)) != -1) {
            out.write(buffer, 0, bytesRead);
            totalBytes += bytesRead;
            progressTracker.update(filename, totalBytes);
        }
    }
}
```

- The `ServletInputStream` provides bytes as they arrive over the network, and the `FileOutputStream` writes them directly to disk.
- The 8KB reusable buffer keeps memory constant regardless of the 2GB file size — no part of the file is ever held in the Java heap.
- The progress tracker is updated after every buffer write, giving the UI real-time feedback.
- For further optimization, `FileChannel.transferFrom()` could use zero-copy to write directly from the network socket to the file system without passing through user space.

**Why this approach?**
  - The alternative — reading the entire request body into a `byte[]` or `ByteArrayOutputStream` — would require 2GB of contiguous heap memory, almost certainly triggering an `OutOfMemoryError`.
  - The streaming approach uses fixed memory regardless of file size.
  - The 8KB buffer is small enough to stay in L1 CPU cache, making the read-write loop CPU-efficient as well as memory-efficient.
  - `FileChannel.transferFrom()` with zero-copy would be even faster (no kernel→user→kernel data movement) but requires NIO channels, not the standard `ServletInputStream`.

### Scenario 3: Configurable Data Export with Character Encoding

A reporting system exports data to CSV files for clients worldwide. European clients require ISO-8859-1 encoding for compatibility with legacy spreadsheet software, while Asian clients need UTF-8 to represent non-Latin characters. The export must handle fields containing commas, line breaks, and double quotes.

```java
public class CsvExporter {
    public void export(List<Record> records, Path output, Charset charset, char delimiter) throws IOException {
        try (Writer writer = new OutputStreamWriter(
                new BufferedOutputStream(Files.newOutputStream(output)), charset)) {

            writer.write("ID" + delimiter + "NAME" + delimiter + "DESCRIPTION\n");
            for (Record record : records) {
                String escapedDesc = record.description()
                    .replace("\"", "\"\"")
                    .replace("\n", " ");
                writer.write(record.id() + delimiter + record.name() + delimiter + "\"" + escapedDesc + "\"\n");
            }
        }
    }
}
```

- The `OutputStreamWriter` bridges the byte stream to a character stream using the caller-specified charset, and `BufferedOutputStream` ensures that writes are batched into 8KB chunks before hitting the disk.
- Without explicit charset control via `OutputStreamWriter`, using `FileWriter` would silently use the platform default encoding, corrupting non-Latin text.
- The `delimiter` parameter allows switching between comma (for standard CSV) and semicolon (for European locales where comma is the decimal separator), and each field is properly quoted and escaped to handle embedded commas, quotes, and newlines.

**Why this approach?**
  - `PrintWriter` is the conventional CSV-writing tool (`println()`, `printf()`) but its auto-flush behavior flushes after every line, turning a single export of 1M records into 1M system calls.
  - The `BufferedOutputStream` + `OutputStreamWriter` + manual `for` loop design batches writes into 8KB chunks, reducing system calls from 1M to roughly 125 (at ~60 bytes per CSV line).
  - The manual loop also allows field-level escaping that would require a library like OpenCSV to achieve with `PrintWriter`.
  - For production CSV exports with millions of records, the difference between buffered and unbuffered writes is seconds versus hours.

---

## Scenario-Based Questions

**Q: A microservice receives CSV files from clients in the US, Europe, and Japan. US clients send UTF-8 files, European clients send ISO-8859-1 files, and Japanese clients send Shift-JIS files. The parser reads every file and produces garbled text for non-UTF-8 files. How do you fix this without asking every client to change their encoding?**

  - Never use `FileReader` or the no-arg `Files.readString()` — both use the platform default charset, which on the server is likely UTF-8 and will corrupt ISO-8859-1 and Shift-JIS files. The fix is to detect or negotiate the charset per client.
  - The simplest approach is to accept a `charset` query parameter or HTTP header: `Content-Type: text/csv; charset=Shift-JIS`.
  - For clients that cannot declare encoding, detect it heuristically by examining byte-order marks (BOM) or the byte distribution:
  ```java
  try (InputStream in = Files.newInputStream(path)) {
      Charset detected = detectCharset(in); // read BOM or analyze bytes
      try (Reader reader = new BufferedReader(
              new InputStreamReader(in, detected))) {
          // parse CSV with correct charset
      }
  }
  ```
  - Third-party libraries like Apache Tika or `juniversalchardet` implement charset detection using Mozilla's charset detection algorithm.
  - Once the charset is known, wrap the `InputStream` with an `InputStreamReader` specifying the detected charset explicitly.
  - The fundamental rule: `InputStreamReader` is the bridge between bytes and characters, and `Charset` is a mandatory parameter, not optional. Default charset is never correct for a multi-region deployment.

  > **Interview follow-up:** The candidate mentioned heuristic charset detection using byte distribution. For a high-throughput pipeline processing 10K files per day, how would you avoid running detection on every file and instead cache or negotiate the charset per client?

**Q: You are building a file watcher service that monitors a directory for new CSV files, processes them, and moves them to an archive. Files arrive at unpredictable times (from 1 to 1000 per minute). Each file is 100MB-2GB. How do you design the I/O pipeline to handle bursts without OOM or thread starvation?**

  - Use a bounded thread pool with a `BlockingQueue<Path>` to decouple file discovery from processing, and stream each file lazily via `Files.lines()`:
  ```java
  try (WatchService watcher = FileSystems.getDefault().newWatchService()) {
      dir.register(watcher, ENTRY_CREATE);
      for (int i = 0; i < 4; i++) {
          executor.submit(() -> {
              while (true) {
                  Path file = fileQueue.poll(10, SECONDS);
                  if (file == null) continue;
                  try (Stream<String> lines = Files.lines(file, UTF_8)) {
                      lines.skip(1).map(this::parse).forEach(this::process);
                  }
                  Files.move(file, archive.resolve(file.getFileName()));
              }
          });
      }
      while (true) {
          WatchKey key = watcher.take();
          key.pollEvents().stream()
              .filter(e -> e.kind() == ENTRY_CREATE)
              .map(e -> dir.resolve((Path) e.context()))
              .forEach(f -> fileQueue.offer(f));
          key.reset();
      }
  }
  ```
  - The four key design decisions are: `Files.lines()` streams each file lazily without loading it entirely into memory, preventing OOM regardless of file size; a bounded thread pool of 4 workers prevents thread starvation during bursts of 1000 files per minute; the `BlockingQueue` decouples high-speed file discovery from slower processing with natural backpressure when the queue fills; and `WatchService` uses OS-level file system events (inotify on Linux, ReadDirectoryChanges on Windows) so there is zero CPU cost when no files are arriving.

  > **Interview follow-up:** The candidate chose 4 worker threads for the pool. If files arrive at 1000 per minute and each takes 30 seconds to process, the queue grows by ~500 files per minute. How would you decide whether to add more workers or add backpressure by rejecting files when the queue exceeds a threshold?

**Q: A service must read a config file that is updated atomically (write to temp file, rename). The service should use the latest config within 5 seconds of a change without polling every few seconds. How do you design this with NIO.2?**

  - Use `WatchService` for OS-level change notifications combined with a `volatile` reference for thread-safe config access:
  ```java
  public class HotReloadConfig {
      private volatile Config config;
      private final Path configPath;

      public void startWatching() throws IOException {
          try (WatchService watcher = configPath.getParent().newWatchService()) {
              configPath.getParent().register(watcher, ENTRY_MODIFY, ENTRY_CREATE);
              reload(); // initial load
              while (true) {
                  WatchKey key = watcher.poll(5, SECONDS);
                  if (key != null) {
                      key.pollEvents().stream()
                          .filter(e -> e.context().equals(configPath.getFileName()))
                          .forEach(e -> reload());
                      key.reset();
                  }
              }
          }
      }

      private void reload() {
          try { this.config = parse(Files.readString(configPath, UTF_8)); }
          catch (IOException e) { log.error("Failed to reload config", e); }
      }
  }
  ```
  - `WatchService` uses OS-level file system event notifications with no polling overhead — the thread sleeps in the kernel until a file system event occurs.
  - The `volatile` keyword on the `config` reference provides the happens-before guarantee required for other threads to see the updated config immediately.
  - The atomic write pattern (`Files.move(temp, target, ATOMIC_MOVE)`) ensures that the reader never sees a partially written file, and `Files.readString()` reads the entire config in one operation — acceptable because config files are typically under 1MB.

  > **Interview follow-up:** The candidate used `WatchService.poll(5, SECONDS)` and a `volatile` reference. If the config file is updated twice within the same 5-second polling interval, would the second update be missed? How would you coalesce rapid consecutive updates?

**Q: A microservice communicates with a legacy system over a TCP socket using a custom binary protocol. Messages are length-prefixed (4 bytes big-endian length + payload). The connection is long-lived. How do you read messages without blocking the entire application?**

  - Use a dedicated single thread with `DataInputStream` for its framing guarantees, isolating the blocking I/O from the rest of the application:
  ```java
  // Blocking approach in a dedicated thread
  public class TcpClient {
      private final DataInputStream in;
      private final ExecutorService executor = Executors.newSingleThreadExecutor();

      public void start() throws IOException {
          SocketChannel channel = SocketChannel.open(new InetSocketAddress(host, port));
          this.in = new DataInputStream(Channels.newInputStream(channel));
          executor.submit(() -> {
              while (!Thread.currentThread().isInterrupted()) {
                  int length = in.readInt(); // blocks until 4 bytes available
                  byte[] payload = new byte[length];
                  in.readFully(payload); // blocks until all bytes received
                  process(payload);
              }
          });
      }
  }
  ```
  - `DataInputStream.readInt()` correctly assembles 4 bytes into a big-endian `int`, and `readFully()` guarantees that exactly `length` bytes are read — unlike raw `InputStream.read(byte[])` which may return fewer bytes.
  - The dedicated single-thread executor keeps the blocking I/O isolated so the rest of the application handles requests concurrently.
  - For higher throughput with fewer threads, use NIO's `Selector` with a `ByteBuffer` to accumulate partial reads, but for a single long-lived connection, the dedicated thread approach is simpler and equally efficient.

  > **Interview follow-up:** The candidate mentioned NIO Selector as an alternative. If the legacy system sends 1000 messages per second over the same connection, at what message rate does the Selector-based approach become meaningfully better than the dedicated thread with DataInputStream?

**Q: A batch job processes 1M records. For each record, it reads a file from disk, transforms it, and writes a new file. The job takes 6 hours. Profiling shows 40% CPU and 60% I/O wait. How do you overlap computation with I/O to improve throughput?**

  - Use `AsynchronousFileChannel` with `CompletableFuture` to overlap I/O operations from multiple files, allowing the I/O subsystem to service one file while another is being transformed:
  ```java
  public CompletableFuture<Void> processFile(Path input, Path output) {
      AsynchronousFileChannel inChannel = AsynchronousFileChannel.open(input, READ);
      AsynchronousFileChannel outChannel = AsynchronousFileChannel.open(output, WRITE, CREATE);
      ByteBuffer buffer = ByteBuffer.allocate(8192);

      return CompletableFuture.runAsync(() -> {
          while (inChannel.read(buffer, position).get() > 0) {
              buffer.flip();
              ByteBuffer transformed = transform(buffer);
              outChannel.write(transformed, writePos).get();
              writePos += transformed.position();
              buffer.clear();
          }
      }, ioExecutor);
  }

  // Process 4 files concurrently
  List<CompletableFuture<Void>> futures = files.stream()
      .map(f -> processFile(f.input(), f.output()))
      .toList();
  CompletableFuture.allOf(futures.toArray(new CompletableFuture[0])).join();
  ```
  - `AsynchronousFileChannel` uses OS-level asynchronous I/O where available (IOCP on Windows) or a thread-pool-backed implementation (on Linux).
  - Processing four files concurrently allows overlapping — while file A's I/O is waiting, file B's data is being transformed on the CPU. This converts the 60% I/O wait time into useful computation, reducing total processing time from 6 hours to approximately 3.5 hours.
  - The `ioExecutor` is a dedicated thread pool sized to the number of concurrent file operations, preventing the I/O tasks from competing with the application's main processing threads.

  > **Interview follow-up:** The candidate estimated a reduction from 6 hours to 3.5 hours with 4 concurrent files. The remaining 3.5 hours is still mostly I/O wait — at what concurrency level does adding more parallel files stop improving throughput and start increasing latency due to disk contention?

**Q: A Spring Boot application serves static assets (images, CSS, JS). Under load, file reads show high latency. The OS cache helps, but first requests are slow. How do you reduce file I/O latency for static assets?**

  - Preload commonly accessed files into `MappedByteBuffer` at application startup and serve from memory with zero-copy buffer duplication:
  ```java
  @Component
  public class AssetCache {
      private final ConcurrentHashMap<String, MappedByteBuffer> cache = new ConcurrentHashMap<>();

      @PostConstruct
      public void preload() throws IOException {
          List.of("styles.css", "app.js", "logo.png").forEach(name -> {
              Path path = Path.of("static", name);
              try (FileChannel channel = FileChannel.open(path, READ)) {
                  cache.put(name, channel.map(READ_ONLY, 0, channel.size()));
              }
          });
      }

      public ByteBuffer get(String name) {
          return cache.getOrDefault(name, empty).duplicate();
      }
  }
  ```
  - `FileChannel.map()` creates a memory-mapped region — the OS loads pages on demand when the data is first accessed, but the mapping itself is established at startup rather than per-request, eliminating per-request `open()` and `close()` overhead.
  - The `.duplicate()` method returns a new `ByteBuffer` that shares the same backing memory, providing zero-copy reads — no data is copied from the mapped buffer to the application heap.
  - For a production system, combine this with proper HTTP caching headers (ETag, Cache-Control) and a CDN; the memory-mapped cache optimizes the server-side path for requests that miss the CDN cache.

  > **Interview follow-up:** The candidate used `MappedByteBuffer` with `.duplicate()` for zero-copy reads. If an asset file is updated on disk (e.g., a new version of `app.js` deployed), do memory-mapped readers see the new content immediately, or would they need to remap?

**Q: You need to tail a growing log file (like `tail -f`) and stream new lines to a WebSocket client. The log file is written by another process. How do you read only new data without re-reading the entire file?**

  - Use `FileChannel` to track the read position and only read the new bytes appended since the last poll:
  ```java
  public class LogTailer {
      private final RandomAccessFile file = new RandomAccessFile(path, "r");
      private final FileChannel channel = file.getChannel();
      private long position = 0;

      public void streamToWebSocket(WebSocket socket) throws IOException {
          while (!closed) {
              long newSize = channel.size();
              if (newSize > position) {
                  channel.position(position);
                  ByteBuffer buffer = ByteBuffer.allocate((int)(newSize - position));
                  channel.read(buffer);
                  buffer.flip();
                  socket.send(StandardCharsets.UTF_8.decode(buffer).toString());
                  position = newSize;
              }
              Thread.sleep(100); // poll interval
          }
      }
  }
  ```
  - `FileChannel` allows seeking to any byte position with `position()`, so we track where we left off and only read the bytes beyond that point.
  - `RandomAccessFile` opens the file in read mode without locking, so the writer process is not blocked.
  - The 100ms polling interval is a reasonable trade-off between near-real-time latency (max 100ms delay) and CPU usage (10 polls per second).
  - For production use, libraries like Apache Commons IO `Tailer` handle log rotation detection, encoding, and configurable polling with less code.

  > **Interview follow-up:** The candidate used a 100ms polling interval. If the log file is rotated (deleted and replaced by the writer), the `RandomAccessFile` still references the deleted file's inode. How would you detect log rotation and reopen the file handle?

**Q: A file parser reads a binary format where records are variable-length but have a fixed-size header (32 bytes) containing the record length. The file is 50GB. How do you parse it efficiently using memory-mapped I/O?**

  - Memory-map the file in 1GB regions and parse sequentially with `ByteBuffer`, handling the edge case where records span region boundaries:
  ```java
  public class BinaryParser {
      public void parse(Path path) throws IOException {
          try (FileChannel channel = FileChannel.open(path, READ)) {
              long fileSize = channel.size();
              long position = 0;
              while (position < fileSize) {
                  long mapSize = Math.min(REGION_SIZE, fileSize - position);
                  MappedByteBuffer region = channel.map(READ_ONLY, position, mapSize);
                  while (region.remaining() >= 32) {
                      int recordLength = region.getInt(region.position() + 28);
                      if (region.remaining() < recordLength) break;
                      parseRecord(region, recordLength);
                      region.position(region.position() + recordLength);
                      position += recordLength;
                  }
                  position += region.position();
              }
          }
      }
  }
  ```
  - Memory-mapping avoids copying data between kernel space and user space — the file data is directly accessible as a `ByteBuffer` in the process's virtual address space.
  - The 1GB `REGION_SIZE` maps a large chunk at a time, with the OS handling demand paging so only the accessed pages are loaded into physical memory.
  - Sequential access within a mapped region is fast because the OS prefetches subsequent pages.
  - The outer loop handles the edge case where a record header at the end of one region's data would exceed the mapped region — in that case, the inner loop breaks, the position is updated, and a new 1GB region is mapped starting from the record boundary.

  > **Interview follow-up:** The candidate chose 1GB as the mapping region size. On a 32-bit JVM, the virtual address space limits individual mappings. What is the maximum `MappedByteBuffer` size on a 32-bit JVM, and how does this change the parsing strategy for a 50GB file?

**Q: A REST API aggregates data from 10 upstream services. Each upstream call returns JSON. The API currently calls each service sequentially (10 x 200ms = 2s total). How do you use NIO to parallelize the HTTP calls without creating 10 threads per request?**

  - Use Java 11+ `HttpClient` with `sendAsync()` which uses NIO non-blocking I/O internally, allowing a single small thread pool to handle thousands of concurrent connections:
  ```java
  public CompletableFuture<AggregatedResponse> aggregate() {
      List<CompletableFuture<JsonNode>> futures = uris.stream()
          .map(uri -> httpClient.sendAsync(
              HttpRequest.newBuilder(uri).build(),
              BodyHandlers.ofByteArray())
              .thenApply(response -> parseJson(response.body())))
          .toList();

      return CompletableFuture.allOf(futures.toArray(new CompletableFuture[0]))
          .thenApply(v -> futures.stream()
              .map(CompletableFuture::join)
              .collect(collectingAndThen(toList(), AggregatedResponse::new)));
  }
  ```
  - Java 11's `HttpClient` uses NIO `Selector` internally, so a single HTTP connection pool of 10-20 connections handles all concurrent calls without a thread-per-connection model.
  - The `sendAsync()` method returns immediately with a `CompletableFuture`, and the underlying NIO selector processes the responses as they arrive.
  - The total response time drops from 2 seconds (serial 200ms x 10) to approximately 200ms (the slowest upstream service), with the connection pool shared across all API requests so that 100 concurrent API requests do not create 1000 threads.

  > **Interview follow-up:** The candidate mentioned a shared connection pool across requests. If one of the 10 upstream services takes 10 seconds to respond, does the NIO selector thread block, or can it continue processing other requests while waiting?

**Q: A service receives files via FTP, processes them, and archives them to S3. Files are 10MB-5GB. Occasionally a file is truncated (FTP transfer interrupted). How do you detect incomplete files before processing?**

  - Use a marker file approach combined with checksum verification to reliably detect incomplete transfers:
  ```java
  // Approach 1: Marker file
  Path marker = uploadDir.resolve(filename + ".done");
  if (Files.exists(marker)) {
      processFile(uploadDir.resolve(filename));
      Files.delete(marker);
  }

  // Approach 2: CRC check
  try (InputStream in = Files.newInputStream(path)) {
      byte[] actual = DigestUtils.sha256(in);
      String expected = readChecksumFile(path.resolveSibling(path.getFileName() + ".sha256"));
      if (!Arrays.equals(actual, Hex.decodeHex(expected))) {
          throw new IOException("Checksum mismatch — file truncated or corrupted");
      }
  }
  ```
  - The marker file approach is simpler: the FTP process creates a `.done` file atomically only after the upload is fully complete, and the processing service only looks for files with a matching `.done` marker.
  - For stronger guarantees against network corruption during transfer, compute a SHA-256 hash during upload and compare it against a distributed checksum file.
  - The `Files.newInputStream()` with `DigestUtils.sha256()` streams the file without loading it into memory, making it safe for 5GB files.

  > **Interview follow-up:** The candidate suggested the marker file approach. If the FTP process crashes after the data file is fully written but before the `.done` file is created, the file is never processed. How would you implement a periodic reconciliation that detects orphaned data files without a corresponding `.done` marker?

**Q: A legacy application writes logs using `System.out.println()`. You need to redirect all stdout to a rolling file without modifying the application code. How do you do this at the JVM level?**

  - Use `System.setOut()` at startup to replace the global `System.out` `PrintStream` with a custom implementation that rolls files hourly:
  ```java
  public class StdoutRedirector {
      public static void redirect(String logDir) {
          try {
              OutputStream rollingOut = new OutputStream() {
                  private PrintWriter current;
                  private long nextRotation = System.currentTimeMillis() + 3600_000;

                  @Override public void write(int b) throws IOException {
                      rotateIfNeeded();
                      current.write(b);
                  }

                  private void rotateIfNeeded() throws IOException {
                      if (current == null || System.currentTimeMillis() > nextRotation) {
                          if (current != null) current.close();
                          String filename = logDir + "/stdout-" + Instant.now().toString() + ".log";
                          current = new PrintWriter(new OutputStreamWriter(
                              new BufferedOutputStream(Files.newOutputStream(Path.of(filename))), UTF_8), true);
                          nextRotation = System.currentTimeMillis() + 3600_000;
                      }
                  }
              };
              System.setOut(new PrintStream(rollingOut, true, UTF_8));
          } catch (IOException e) { throw new RuntimeException(e); }
      }
  }
  ```
  - `System.setOut()` replaces the global stdout `PrintStream` with a custom implementation.
  - The custom `OutputStream` wraps a rolling file writer that creates a new log file every hour, with `BufferedOutputStream` batching the many small `write()` calls from `println()` into 8KB chunks.
  - The `autoFlush=true` parameter ensures each line is written to disk promptly — a trade-off between durability and write amplification.
  - For production use, a proper logging framework like Logback handles rotation, compression, retention, and cleanup with far less custom code.

  > **Interview follow-up:** The candidate mentioned `System.setOut()` wrapped in a custom `OutputStream`. If the legacy application also calls `System.err.println()`, those messages are lost to stderr. How would you capture both stdout and stderr into the same rolling file without modifying the application code?

---

## Interview Questions

**What is the difference between `InputStream` and `Reader`?**
  - `InputStream` reads raw bytes (8-bit) for binary data like images, ZIP files, and serialized objects.
  - `Reader` reads characters (16-bit Unicode) for text data, with internal charset decoding from bytes to characters.
  - The bridge between them is `InputStreamReader`, which converts incoming bytes to characters using a specified `Charset`.
  - Always choose `InputStream` for binary data and `Reader` for text data.

**What is try-with-resources and why is it important for I/O?**
  - try-with-resources (Java 7+) automatically closes resources implementing `AutoCloseable` after the try block, regardless of exceptions.
  - For I/O, this prevents file handle leaks that would otherwise require explicit `finally` blocks and null checks.
  - Multiple resources are declared separated by semicolons and closed in reverse declaration order. The syntax is: `try (InputStream in = new FileInputStream("file")) { ... }`.

**What is the difference between `FileInputStream` and `FileChannel`?**
  - `FileInputStream` is a blocking byte stream where each `read()` causes a system call.
  - `FileChannel` provides position-independent read/write with zero-copy transfer methods (`transferTo()`, `transferFrom()`), memory-mapped I/O (`map()`), and file locking (`lock()`, `tryLock()`).
  - `FileChannel` is typically faster for large files and random access patterns. `FileInputStream` is simpler for small sequential reads.

**What is buffering and why does it matter for I/O performance?**
  - Buffering groups many small I/O operations into larger blocks to reduce the number of system calls.
  - Each system call incurs overhead from kernel privilege switching, context switching, and cache effects.
  - Reading one byte at a time from a 1GB file causes 1 billion system calls; `BufferedInputStream` with an 8KB buffer reduces this to 125K calls — a 10,000x reduction that can change processing time from 45 minutes to under one minute.

**What is the difference between `File` (java.io) and `Path` (java.nio.file)?**
  - `File` is the legacy API with inconsistent error handling (returns boolean instead of throwing exceptions), no symbolic link support, and unreliable `delete()` behavior.
  - `Path` is immutable, supports `resolve()` and `relativize()` for path manipulation, works with symbolic links, and integrates with `Files` utility methods that throw meaningful exceptions.
  - Always use `Path` and `Files` for new code.

**What is memory-mapped I/O and when should you use it?**
  - `MappedByteBuffer` maps a file region into the process's virtual memory, allowing the OS to handle paging between disk and memory transparently. Reads and writes become memory operations without explicit `read()`/`write()` calls.
  - Best for: large files (100MB+) with random access patterns, shared memory between processes, and high-performance database or indexing systems.
  - Avoid for small files, very short-lived operations, or files that change size frequently.

**How does `FileChannel.transferTo()` achieve zero-copy?**
  - `transferTo()` delegates to the OS-level `sendfile()` (Linux) or `TransmitFile()` (Windows).
  - Data is copied directly from the file system cache to the network socket (or output channel) within kernel space, never passing through the Java application's memory.
  - This eliminates the data copy from kernel to user space and back, reducing CPU usage and improving throughput by 10-50x for large data transfers.

**What is the difference between `Files.readAllLines()` and `Files.lines()`?**
  - `readAllLines()` loads the entire file into a `List<String>` in memory, risking `OutOfMemoryError` for large files.
  - `Files.lines()` returns a lazy `Stream<String>` that reads lines on demand from the underlying `FileChannel`, keeping memory proportional to the largest line rather than the file size.
  - Use `readAllLines()` only for small files under 100MB; use `Files.lines()` for all other text processing.

**What is the difference between blocking I/O (BIO) and non-blocking I/O (NIO)?**
  - In BIO, a thread blocks until the I/O operation completes — the thread-per-connection model cannot scale beyond a few thousand connections due to thread stack memory consumption.
  - In NIO, a `Selector` monitors many `Channel` instances and processes only those that are ready, allowing one thread to handle thousands of connections.
  - NIO is more complex but necessary for high-concurrency servers; BIO is simpler and sufficient for low-concurrency scenarios.

**How do you properly close resources when using multiple I/O streams?**
  - Use try-with-resources with separate variable declarations for each stream, because the outer stream's constructor may throw before `close()` is recorded, leaking the inner stream:
  ```java
  try (FileInputStream fis = new FileInputStream("file");
       BufferedInputStream bis = new BufferedInputStream(fis);
       DataInputStream dis = new DataInputStream(bis)) {
      // All three streams are closed, even if DataInputStream constructor throws
  }
  ```
  - In Java 9+, the `InputStreamReader(InputStream)` constructor is annotated with `@SuppressWarnings("try")` to handle this correctly even in a single-resource try-with-resources, but the multi-variable pattern is the most robust approach.

**What is the difference between `OutputStream` and `Writer`?**
  - `OutputStream` writes raw bytes for binary data; `Writer` writes characters with charset encoding. The bridge is `OutputStreamWriter`.
  - Always use `Writer` for text output to ensure proper charset conversion. Using `OutputStream.write(String.getBytes())` is error-prone because it depends on the platform default charset.

**What is `PipedInputStream` and `PipedOutputStream` used for?**
  - Pipe streams connect two threads within the same JVM — one thread writes to `PipedOutputStream`, another reads from the connected `PipedInputStream`.
  - Used for inter-thread communication without shared memory or files. The pipe has a fixed-size internal buffer (typically 1024 bytes), and writes block when the buffer is full.
  - In practice, `BlockingQueue` or `Exchanger` are preferred for thread communication because they handle synchronization more explicitly.

**How does `ObjectOutputStream` handle circular references during serialization?**
  - `ObjectOutputStream` maintains a reference table of all objects already written. When a previously serialized object is encountered again, it writes a back-reference to the table instead of serializing the object's data again.
  - This handles circular object graphs without infinite recursion and reduces the serialized stream size. The reference table is cleared when `reset()` is called on the stream.

**What is the purpose of `PushbackInputStream`?**
  - `PushbackInputStream` allows you to "unread" one or more bytes back into the stream so they will be returned by the next `read()` call.
  - Useful for parsing scenarios where you need to peek ahead to determine how to interpret the current token, then push back characters that were read speculatively. The pushback buffer size defaults to 1 byte.

**What is `SequenceInputStream` used for?**
  - `SequenceInputStream` concatenates multiple `InputStream` instances, reading from each in sequence until exhausted, then moving to the next.
  - Useful for merging multiple log files, configuration file fragments, or file segments into a single logical stream without creating a temporary concatenated file.

**What is the difference between `RandomAccessFile` and `FileChannel`?**
  - `RandomAccessFile` supports read/write at arbitrary positions via `seek()` but operates at the byte-stream level.
  - `FileChannel` provides the same random-access capability through `position()` and `read()`/`write()` at the channel level, with additional features like `transferTo()`, `map()` for memory-mapped I/O, and file locking. For new code, prefer `FileChannel` wrapped in `Channels.newInputStream()`/`newOutputStream()`.

**How does `Console` (System.console()) differ from `Scanner` for reading user input?**
  - `System.console()` provides password masking (`readPassword()` returns `char[]`, not `String`), formatted printing (`printf()`), and reader/writer access. It returns `null` if the application has no console (e.g., in an IDE or background process).
  - `Scanner` works anywhere but cannot mask passwords and has no direct console integration. Use `Console` for interactive CLI tools; use `Scanner` for testing or non-interactive scenarios.

**What is the difference between `FileSystem.getDefault()` and `Paths.get()`?**
  - `FileSystem.getDefault()` returns the JVM's default file system (typically the OS native file system). You can obtain separators, root directories, and file stores from it.
  - `Paths.get(String)` is a shortcut for `FileSystems.getDefault().getPath()`. Use `FileSystem.getDefault()` when you need file system operations beyond path creation, like creating `WatchService` instances or iterating file stores.

**What is `AsynchronousFileChannel` and how does it differ from `FileChannel`?**
  - `AsynchronousFileChannel` (Java 7 NIO.2) performs file I/O without blocking the calling thread. `read()` and `write()` return immediately and complete via `Future` or `CompletionHandler`.
  - Use it for high-concurrency file servers and event-driven architectures where thread-per-file is not scalable. Unlike `FileChannel`, operations can be submitted with a thread pool (`ExecutorService`), and multiple operations on the same channel can run concurrently.

---

## Developer Recommendations

- **Always specify the charset explicitly**
  - `FileReader`, `FileWriter`, and `String.getBytes()` use the platform default charset, which is platform-dependent.
  - A file written with `FileWriter` on Windows and read with `FileReader` on Linux may produce corrupted `?` characters.
  - Always use `StandardCharsets.UTF_8` or the appropriate charset in every I/O call.

- **Always buffer I/O streams**
  - Reading one byte at a time from a file causes a system call per byte, and with 1 billion system calls per GB, the user-kernel context switches dominate execution time.
  - Wrapping with `BufferedInputStream` (default 8KB buffer) reduces system calls by a factor of 8192 and can turn a 45-minute operation into a sub-minute one.

- **Prefer NIO.2 Path and Files over legacy java.io.File**
  - `File` has inconsistent error handling (returning `boolean` instead of throwing `IOException`), no support for symbolic links, and unreliable `delete()` behavior.
  - `Path` is immutable and thread-safe, supports `resolve()`, `relativize()`, symbolic links, and works with `Files` methods that throw specific exception subclasses.
  - At small scale (a few dozen files in a desktop app), the difference between `File` and `Path` rarely causes incidents. At microservice scale (thousands of files processed per minute), `File`'s silent boolean failures cause undetected data loss — a `File.delete()` that returns `false` instead of throwing means a temp file is never cleaned up, eventually exhausting disk space in production.

- **Use Files.lines() for large files and Files.readString() for small files**
  - `Files.readAllLines()` loads the entire file into a `List<String>` — for a 2GB log file with 20 million lines, that is 20 million `String` objects that will cause `OutOfMemoryError`.
  - `Files.lines()` returns a lazy `Stream<String>` that reads lines on demand with memory proportional to the largest line.

- **Use FileChannel.transferTo() for zero-copy file transfers**
  - Copying a file with `read()` and `write()` requires four kernel crossings per chunk.
  - `transferTo()` uses `sendfile()` — data moves directly between file descriptors within the kernel, eliminating all user-space copies and context switches.
  - For a 1GB file, this is 10-50x faster than the traditional read-write loop.

- **Use try-with-resources for every I/O resource**
  - Unclosed file handles accumulate until the process hits the OS's file descriptor limit (typically 1024 on Linux), causing all subsequent file operations to fail with `Too many open files`.
  - Every `Files.lines()` call must be wrapped: `try (Stream<String> lines = Files.lines(path)) { ... }`.

- **Use DataInputStream for binary data with known structure**
  - The `readFully()` method guarantees that the requested number of bytes is read or throws `EOFException`, unlike `InputStream.read(byte[])` which may return fewer bytes.
  - For network protocols and binary file formats, `readInt()`, `readLong()`, and `readUTF()` handle byte ordering and framing correctly.

- **Use memory-mapped files for random-access operations on large files**
  - `FileChannel.map()` maps a file region into virtual memory, and the OS manages the page cache, keeping frequently accessed pages in physical memory.
  - For a 10GB database file with random 4KB page accesses, memory-mapped I/O can be 10-100x faster than `RandomAccessFile`.

- **Know that Files.walk() traverses depth-first, not breadth-first**
  - `Files.walk()` performs a depth-first pre-order traversal (children before siblings).
  - For operations like deleting a directory tree, the depth-first order is correct — you must delete children before parents.
  - For operations that need breadth-first (e.g., limiting recursion depth), use `Files.walk()` with a max depth parameter `Files.walk(root, maxDepth)` or collect into levels with `Files.list()` in a loop.
