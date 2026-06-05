# Java I/O Streams

---

## Overview

- **Definition:** Java I/O Streams are a fundamental API for reading from and writing to data sources — files, network sockets, memory buffers, system console — using a stream abstraction: a continuous flow of data.

- **Why Multiple APIs?:**

| API | Package | I/O Model | Since |
|-----|---------|-----------|-------|
| **I/O Streams** | `java.io` | Blocking, byte/char | Java 1.0 |
| **NIO** | `java.nio` | Buffers, channels, selectors | Java 1.4 |
| **NIO.2** | `java.nio.file` | File system API, async channels | Java 7 |
| **Memory-Mapped** | `java.nio.MappedByteBuffer` | File → Direct memory | Java 1.4 |

- **Key Principle:** Always close resources using try-with-resources (Java 7+):
  ```java
  try (InputStream in = new FileInputStream("file")) { ... }
  ```

---

## Byte Streams (8-bit)

- **Definition:** Handle I/O of raw binary data. The root classes are `InputStream` (reading) and `OutputStream` (writing).

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

- **Definition:** Handle I/O of character data. Root classes are `Reader` and `Writer`. They handle character encoding transparently.

| Reader | Writer | Purpose |
|--------|--------|---------|
| `FileReader` | `FileWriter` | File I/O (use with caution — charset issues) |
| `CharArrayReader` | `CharArrayWriter` | In-memory buffer |
| `BufferedReader` | `BufferedWriter` | Buffering |
| `InputStreamReader` | `OutputStreamWriter` | Bridge between byte and char streams |
| `StringReader` | `StringWriter` | String as source/sink |
| `PrintWriter` | — | Formatted text output |

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

- **Bridge Streams:** Use `InputStreamReader` to convert byte stream to character stream, and `OutputStreamWriter` for the reverse. Always specify the charset:
  ```java
  new InputStreamReader(in, StandardCharsets.UTF_8)
  ```

---

## NIO.2 File API (Preferred for File I/O)

- **Definition:** The modern file API in `java.nio.file` package (since Java 7) providing simpler, more powerful file operations.

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

- **Always prefer NIO.2 `Files` + `Path` over legacy `java.io.File`** for file operations. The NIO.2 API is more consistent, supports symbolic links, better error handling, and is generally faster.

---

## Buffering

- **Definition:** Buffering improves I/O performance by reducing the number of system calls. Each system call has overhead, so reading/writing in larger chunks is more efficient.

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

- **Always wrap streams with buffering** — the default buffer size is 8192 bytes, which is sufficient for most use cases.

---

## Under the Hood: Blocking vs Non-Blocking I/O

- **Blocking I/O (java.io):** The calling thread blocks until the I/O operation completes. Each connection needs its own thread. Simple but doesn't scale well for many concurrent connections.

- **NIO Non-Blocking I/O:** A `Selector` enables single-thread handling of multiple channels. The thread can process any channel that's ready, rather than being blocked on one.

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

- **Memory-Mapped Files:** Maps a file region directly into memory — the OS handles paging between disk and memory. Performance can be 10-100x faster than traditional `read()` for large files.

```java
FileChannel channel = FileChannel.open(path, StandardOpenOption.READ);
MappedByteBuffer buffer = channel.map(
    FileChannel.MapMode.READ_ONLY, 0, channel.size());
```

---

## Common Mistakes

- **Not closing streams** — resource leak. Always use try-with-resources.
- **No charset specified** — platform-dependent encoding. Always specify `StandardCharsets.UTF_8`.
- **Not buffering** — reading one byte at a time causes excessive system calls. Always wrap with `BufferedInputStream`/`BufferedReader`.
- **Forgetting flush()** — buffered data lost on crash without flush. BufferedWriter/OutputStream auto-flush may not cover all cases.
- **Partial reads** — `read(byte[])` may read fewer bytes than the array size. Use `readFully()` (DataInputStream) or loop.
- **Large file into memory** — `readAllBytes()`/`readAllLines()` on huge files causes OOM. Use streaming with `Files.lines()`.
- **File.exists() before access** — TOCTOU race condition (file deleted between check and access). Just open and handle `FileNotFoundException`.
- **File.separator hardcoding** — use `Path.of()` for cross-platform paths. Don't hardcode `/` or `\`.
- **Calling flush() too frequently** — defeats the purpose of buffering.

---

## Real-World Scenarios

### Scenario 1: High-Throughput Log Ingestion Pipeline

A logging system ingests 50GB of application logs per day from 200 microservices. Logs arrive as gzipped files over HTTP. The system must parse, filter, and index each line with minimal memory footprint.

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

`GZIPInputStream` wraps `FileInputStream` to decompress on the fly. `BufferedReader` wraps the `InputStreamReader` for line-based reading with internal buffering (8KB default). The entire pipeline reads one line at a time — memory stays at ~8KB + one line regardless of file size. Without buffering, each `readLine()` would cause a system call, and without streaming decompression, the entire gzip file (potentially 500MB decompressed) would need to fit in memory.

### Scenario 2: Multipart File Upload with Progress

A web application allows users to upload large video files (up to 2GB). The server must save the file to disk while streaming it — not loading the entire file into memory. It must also track upload progress.

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

The `ServletInputStream` (from `request.getInputStream()`) provides bytes as they arrive over the network. The `FileOutputStream` writes them directly to disk. The 8KB buffer keeps memory constant regardless of file size. `FileChannel.transferFrom()` could further optimize by using zero-copy if the servlet container supports it, but the buffered approach is simpler and works across all containers.

### Scenario 3: Configurable Data Export with Character Encoding

A reporting system exports data to CSV files for clients worldwide. European clients need ISO-8859-1 encoding; Asian clients need UTF-8. The export must handle line breaks within fields and use the correct column delimiter (comma vs semicolon for European locales).

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

The `OutputStreamWriter` bridges bytes to characters using the specified charset. `BufferedOutputStream` ensures writes are batched into 8KB chunks. Without explicit charset control, `FileWriter` would use the platform default (Windows-1252 on US Windows, causing data loss for Asian characters). The `delimiter` parameter allows comma/semicolon switching without code changes.

---

## Scenario-Based Questions

1. **Q: You are building a file watcher service that monitors a directory for new CSV files, processes them, and moves them to an archive. Files arrive at unpredictable times (from 1 to 1000 per minute). Each file is 100MB-2GB. How do you design the I/O pipeline to handle bursts without OOM or thread starvation?**
   A: Use a bounded thread pool (e.g., 4 threads) with a `BlockingQueue<Path>` for file discovery:
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
   The key design decisions: `Files.lines()` streams each file lazily (no OOM regardless of file size); bounded thread pool prevents thread starvation during bursts; `BlockingQueue` decouples discovery from processing with backpressure; `WatchService` uses OS-level file system events (no polling overhead).

2. **Q: A service must read a config file that is updated atomically (write to temp file, rename). The service should use the latest config within 5 seconds of a change without polling every few seconds. How do you design this with NIO.2?**
   A: Use `WatchService` for change notifications combined with atomic reads:
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
   `WatchService` uses OS-level inotify (Linux) or ReadDirectoryChanges (Windows) — no polling overhead. `volatile` config reference ensures visibility across threads. `Files.readString()` reads the entire config (assumed small, <1MB). For atomicity, the writer uses `Files.move(temp, target, ATOMIC_MOVE)` so the reader never sees a partially-written file.

3. **Q: A microservice communicates with a legacy system over a TCP socket using a custom binary protocol. Messages are length-prefixed (4 bytes big-endian length + payload). The connection is long-lived. How do you read messages without blocking the entire application?**
   A: Use NIO non-blocking channels with a `Selector`, or wrap with `DataInputStream` in a dedicated thread:
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
   `DataInputStream.readInt()` and `readFully()` handle the framing correctly — `readFully()` guarantees the entire payload is read (unlike raw `InputStream.read()` which may read partial). The dedicated thread blocks on I/O without affecting other parts of the system. For higher throughput with fewer threads, use NIO's `Selector` with a `ByteBuffer` to accumulate data.

4. **Q: A batch job processes 1M records. For each record, it reads a file from disk, transforms it, and writes a new file. The job takes 6 hours. Profiling shows 40% CPU and 60% I/O wait. How do you overlap computation with I/O to improve throughput?**
   A: Use asynchronous I/O with `AsynchronousFileChannel` and `CompletableFuture`:
   ```java
   public CompletablePath<Void> processFile(Path input, Path output) {
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

   // Process 4 files concurrently — overlap I/O with I/O
   List<CompletableFuture<Void>> futures = files.stream()
       .map(f -> processFile(f.input(), f.output()))
       .toList();
   CompletableFuture.allOf(futures.toArray(new CompletableFuture[0])).join();
   ```
   `AsynchronousFileChannel` uses OS-level AIO (on Windows/IOCP) or thread-pool-backed AIO (on Linux/epoll). Processing 4 files concurrently allows the I/O subsystem to service reads from one file while another is transforming data. The result: I/O wait drops from 60% to 30%, and total time reduces from 6 hours to ~3.5 hours.

5. **Q: A Spring Boot application serves static assets (images, CSS, JS). Under load, file reads show high latency. The OS cache helps, but first requests are slow. How do you reduce file I/O latency for static assets?**
   A: Preload commonly accessed files into a `MappedByteBuffer` at startup and serve from memory:
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
   `FileChannel.map()` creates a memory-mapped file — the OS loads pages on demand but keeps them in the page cache. First access is faster than `FileInputStream` because the mapping is established at startup (not per-request). `.duplicate()` returns a new `ByteBuffer` sharing the same backing memory (zero-copy). For production, use a proper HTTP cache (ETag, Cache-Control) and a CDN — memory-mapped files optimize the server side when the CDN miss rate is high.

6. **Q: You need to tail a growing log file (like `tail -f`) and stream new lines to a WebSocket client. The log file is written by another process. How do you read only new data without re-reading the entire file?**
   A: Use `FileChannel.position()` to track read position and poll for changes:
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
   `FileChannel` allows seeking to any position — we track where we left off and only read new bytes. `RandomAccessFile` opens the file in read mode without locking. The 100ms poll is a reasonable trade-off between latency and CPU. For zero-latency updates, use `WatchService` (but it only reports directory changes, not file growth) or inotify directly. For production, use a library like Apache Commons IO `Tailer` which handles log rotation and encoding.

7. **Q: A file parser reads a binary format where records are variable-length but have a fixed-size header (32 bytes) containing the record length. The file is 50GB. How do you parse it efficiently using memory-mapped I/O?**
   A: Memory-map the file in large chunks and parse sequentially with a `ByteBuffer`:
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
   Memory mapping avoids copying data between kernel and user space. The 1GB `REGION_SIZE` maps a large chunk at a time — the OS handles paging. Sequential access within a mapped region is fast because the OS prefetches pages. The outer loop handles the case where a record spans region boundaries. This approach is 3-5x faster than `FileInputStream` for random-access binary formats.

8. **Q: A REST API aggregates data from 10 upstream services. Each upstream call returns JSON. The API currently calls each service sequentially (10 × 200ms = 2s total). How do you use NIO to parallelize the HTTP calls without creating 10 threads per request?**
   A: Use a non-blocking HTTP client (Java 11+ `HttpClient` with `sendAsync`) with a small connection pool:
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
   Java 11's `HttpClient` uses non-blocking I/O internally (NIO `Selector`). A single HTTP connection pool of 10-20 connections handles all 10 concurrent calls — no thread-per-connection overhead. The `sendAsync()` returns immediately with a `CompletableFuture`. The total response time drops from 2s to ~200ms (the slowest upstream). The thread pool is shared across all API requests, so 100 concurrent API requests don't create 1000 threads.

9. **Q: A service receives files via FTP, processes them, and archives them to S3. Files are 10MB-5GB. Occasionally a file is truncated (FTP transfer interrupted). How do you detect incomplete files before processing?**
   A: Write a marker file after the upload completes, or check file consistency:
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
   The marker file approach is simpler: the FTP process creates the `.done` file only after the upload is complete. The processing service only looks for files with a matching `.done` marker. For stronger guarantees, compute SHA-256 during upload and compare. `Files.newInputStream()` with `DigestUtils.sha256()` streams the file without loading it entirely into memory — safe for 5GB files.

10. **Q: A legacy application writes logs using `System.out.println()`. You need to redirect all stdout to a rolling file without modifying the application code. How do you do this at the JVM level?**
    A: Use `System.setOut()` with a custom `PrintStream` that wraps a rolling file appender:
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
    `System.setOut()` replaces the global stdout `PrintStream`. The custom `OutputStream` wraps a rolling file writer that rotates hourly. `BufferedOutputStream` ensures writes are batched (reducing system calls from one per `println()` to one per 8KB). The `autoFlush=true` parameter on `PrintWriter` ensures data is written to disk promptly (but this is a trade-off between durability and performance). For production, use Logback's `SyslogAppender` or SLF4J's `Logback` which handles rotation, compression, and cleanup.

---

## Interview Questions

1. **What is the difference between `InputStream` and `Reader`?**
   A: `InputStream` reads raw bytes (8-bit). `Reader` reads characters (16-bit Unicode). `Reader` handles character encoding translation from bytes to chars using a `Charset`. Always use `InputStream` for binary data (images, ZIP files) and `Reader` for text data. Bridge between them with `InputStreamReader` which converts bytes to characters using a specified charset.

2. **What is try-with-resources and why is it important for I/O?**
   A: try-with-resources (Java 7+) automatically closes resources that implement `AutoCloseable`. For I/O, it ensures streams, channels, and readers are closed even if an exception occurs. Without it, unclosed streams cause file handle leaks, eventually throwing `TooManyOpenFilesException`. The syntax: `try (InputStream in = new FileInputStream("file")) { ... }`. Resources are closed in reverse order of declaration.

3. **What is the difference between `FileInputStream` and `FileChannel`?**
   A: `FileInputStream` is a blocking byte stream — each `read()` causes a system call. `FileChannel` provides more advanced operations: position-independent read/write, memory-mapped I/O (`map()`), zero-copy transfers (`transferTo()`, `transferFrom()`), and locking (`lock()`, `tryLock()`). `FileChannel` is typically faster for large files and random access. Use `FileInputStream` for simple sequential reads of small to medium files.

4. **What is buffering and why does it matter for I/O performance?**
   A: Buffering groups multiple small writes/reads into larger blocks, reducing the number of system calls. Each system call has overhead (kernel privilege switch, context switch). Reading one byte at a time causes N system calls for N bytes. `BufferedInputStream` with an 8KB buffer causes N/8192 system calls. For a 1GB file, that's 1 billion vs 125K system calls — a 10,000x reduction.

5. **What is the difference between `File` (java.io) and `Path` (java.nio.file)?**
   A: `File` is the legacy API (Java 1.0) with inconsistent methods, no symbolic link support, and poor error handling (returns boolean instead of throwing exceptions). `Path` (Java 7+) is immutable, supports `resolve()`, `relativize()`, symbolic links, and works with `Files` utility methods that throw meaningful exceptions. Always use `Path` and `Files` for new code.

6. **What is memory-mapped I/O and when should you use it?**
   A: `MappedByteBuffer` maps a file region into the process's virtual memory address space. The OS handles paging between disk and memory transparently. Reads and writes become memory operations — no explicit `read()`/`write()` calls. Best for: large files (100MB+), random access patterns, shared memory between processes. Avoid for: small files (overhead of mapping is not justified), very short-lived operations, files that change size frequently.

7. **How does `FileChannel.transferTo()` achieve zero-copy?**
   A: `transferTo()` uses OS-level `sendfile()` (Linux) or `TransmitFile()` (Windows). Data is copied directly from the file cache to the network socket (or output channel) without passing through the Java application's memory space (no user-kernel context switches for data). This is 10-50x faster than `read()` + `write()` for large data transfers. Common use: serving static files from a web server.

8. **What is the difference between `Files.readAllLines()` and `Files.lines()`?**
   A: `readAllLines()` loads all lines into a `List<String>` in memory — OOM risk for large files. `Files.lines()` returns a lazy `Stream<String>` that reads lines on demand — memory scales with the largest line, not the file size. Use `readAllLines()` for small files (<100MB) where you need random access to lines. Use `Files.lines()` for large files or streaming processing.

9. **What is the difference between blocking I/O (BIO) and non-blocking I/O (NIO)?**
   A: In BIO, a thread blocks until the I/O operation completes — one thread per connection model that doesn't scale to thousands of connections. In NIO, a `Selector` monitors multiple channels and processes only those that are ready — one thread can handle thousands of connections. NIO is more complex but necessary for high-concurrency servers (10K+ connections). BIO is simpler and fine for low-concurrency scenarios.

10. **How do you properly close resources when using multiple I/O streams?**
    A: The outer stream's `close()` typically calls the inner stream's `close()`. But if the outer stream's constructor throws, the inner stream leaks. Best practice: use try-with-resources with separate variables for each resource:
    ```java
    try (FileInputStream fis = new FileInputStream("file");
         BufferedInputStream bis = new BufferedInputStream(fis);
         DataInputStream dis = new DataInputStream(bis)) {
        // Both fis and bis are closed even if DataInputStream constructor throws
    }
    ```
    Or if wrapping in a single try-with-resources, create the inner stream first and pass it to the outer. In Java 9+, the outer stream's constructor can take the inner as a parameter and both are closed properly.

---

## Developer Recommendations

- **Always specify charset explicitly** — `FileReader`, `FileWriter`, `String.getBytes()` use the platform default charset. On Windows this is `windows-1252`; on Linux it's `UTF-8`. A file written on one platform may be unreadable on another. Always use `StandardCharsets.UTF_8` or explicitly specify the charset. `Files.writeString(path, content, StandardCharsets.UTF_8)` is both explicit and concise.

- **Always buffer I/O streams** — Reading one byte at a time causes one system call per byte (millions of user-kernel context switches). Wrap with `BufferedInputStream` (8KB default buffer). For text, use `BufferedReader`/`BufferedWriter`. For a 500MB file, buffering reduces processing time from 45 minutes to under 1 minute — a 45x improvement from a single wrapper class.

- **Prefer NIO.2 `Files` + `Path` over legacy `java.io.File`** — `File` has inconsistent error handling (returns `boolean` instead of throwing), no symbolic link support, and unreliable `delete()` behavior. `Path` + `Files` throws meaningful exceptions, supports `walk()`, `find()`, `copy()`, `move()`, and works with `Stream<String>` for efficient processing. Migration is straightforward: replace `new File(path)` with `Path.of(path)`.

- **Use `Files.lines()` for large files, `Files.readString()` for small files** — `Files.readAllLines()` loads the entire file into a `List<String>` in memory. For a 2GB log file with 20M lines, that's 20M String objects — certain OOM. `Files.lines()` returns a lazy `Stream<String>` that reads lines on demand. For files under 100MB, `Files.readString()` (Java 11+) is simple and efficient.

- **Use `FileChannel.transferTo()` for zero-copy file transfers** — Copying a file via `read()` + `write()` loops through user space (copy from kernel → app → kernel). `transferTo()` uses OS-level `sendfile()` — data moves directly between file descriptors in kernel space. For a 1GB file, this is 10-50x faster. Use it for file copies, serving files over HTTP, and compressing files.

- **Use try-with-resources for ALL I/O resources** — Every `InputStream`, `OutputStream`, `Reader`, `Writer`, `Channel`, and `Stream<Path>` must be closed. Unclosed file handles accumulate until the process hits the OS limit (typically 1024-4096). try-with-resources guarantees cleanup even with exceptions. For `Files.lines()`, always wrap: `try (Stream<String> lines = Files.lines(path)) { ... }`.

- **Use `DataInputStream` for binary data with known structure** — Raw `InputStream.read()` returns partial data (may read fewer bytes than requested). `DataInputStream.readFully()` guarantees the requested bytes or throws `EOFException`. For multi-byte values, `readInt()`, `readLong()`, `readUTF()` handle endianness and framing correctly. This is essential for network protocols and binary file formats.

- **Use `Memory-mapped` files for random-access large files** — `FileChannel.map()` maps a file region into virtual memory. The OS manages paging, so random access patterns are fast (the OS keeps frequently accessed pages in memory). For a 10GB database file where you access random 4KB pages, memory-mapped I/O is 10-100x faster than `RandomAccessFile` because the OS optimizes page cache usage.
