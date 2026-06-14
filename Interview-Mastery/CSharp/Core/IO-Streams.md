# IO Streams

---

## Overview

- **Definition:** The `System.IO.Stream` abstract class provides a foundation for reading and writing bytes across diverse sources: files, memory, networks, pipes, and compression.
- **Why It Exists:** Abstracts data sources into a uniform byte-oriented read/write/seek interface, enabling composable pipelines via the decorator pattern and supporting both synchronous and asynchronous I/O.
- **Key Concepts:** **`FileStream`** (file I/O), **`MemoryStream`** (in-memory byte array), **`NetworkStream`** (socket I/O), **`GZipStream`** / **`CryptoStream`** (transform decorators), **`StreamReader`** / **`StreamWriter`** (encoding-aware text wrappers), **`BufferedStream`** (adds buffering), **`System.IO.Pipelines`** (high-performance zero-copy I/O), and **`MemoryMappedFile`** (shared memory / random access).

---

## Core Concepts

- **Stream Hierarchy:** `Stream` (abstract) → `FileStream`, `MemoryStream`, `NetworkStream`, `GZipStream`, `CryptoStream`, `BufferedStream`, `SslStream`, `PipeStream`. Decorator pattern allows chaining: `new StreamReader(new GZipStream(new FileStream(...)))`.
- **FileStream Internals:** On Windows uses `CreateFile`/`ReadFile`/`WriteFile` (Win32 API); on Linux uses `open`/`read`/`write` syscalls. Async I/O uses `OVERLAPPED` on Windows, `epoll` on Linux. Default buffer is 4096 bytes. `FileOptions.Asynchronous` enables true async I/O (without it, async operations block thread pool threads).

```csharp
// FileStream with async and sequential scan hint
using var fs = new FileStream(path, FileMode.Open, FileAccess.Read,
    FileShare.Read, 65536, FileOptions.Asynchronous | FileOptions.SequentialScan);
```

- **MemoryStream Internals:** Wraps a `byte[]` buffer that can be externally provided (not resizable) or internally allocated (resizable like `List<byte>`). `GetBuffer()` returns the internal array (dangerous — includes unused bytes). `ToArray()` returns a copy. `TryGetBuffer()` returns `ArraySegment<byte>`.

```csharp
// Capacity grows: max(buffer.Length * 2, _length + count)
// Buffer is NOT zero-initialized beyond _length
```

- **StreamReader Encoding Detection:** Detects encoding via BOM: UTF-8 (`EF BB BF`), UTF-16 LE (`FF FE`), UTF-16 BE (`FE FF`), UTF-32 LE (`FF FE 00 00`). Internal buffer is 1024 bytes by default for decoding.
- **Async I/O Requirements:** `FileOptions.Asynchronous` must be specified for true async file I/O. Without it, `ReadAsync` still blocks a thread pool thread. `CopyToAsync` uses a default buffer of 81920 bytes.
- **Memory-Mapped Files:** `MemoryMappedFile` + `MemoryMappedViewAccessor` for random access to large files, shared memory between processes, and files larger than 2GB. Zero-copy reads via OS paging.

```csharp
// Chaining decorators: compress then encrypt
await using var source = File.OpenRead(sourcePath);
await using var dest = File.Create(destPath);
await using var cryptoStream = new CryptoStream(dest, aes.CreateEncryptor(), CryptoStreamMode.Write);
await using var gzipStream = new GZipStream(cryptoStream, CompressionLevel.Optimal);
await source.CopyToAsync(gzipStream);
```

---

## Common Mistakes

- **Not disposing streams** — File handles leak if exceptions occur before `Dispose`. Always use `using` statements. Dispose order matters: inner streams are disposed first (outer wrapper flush → inner close).
  - **Why it looks correct:** the `Stream` variable goes out of scope and the finalizer eventually runs, but without explicit `using`, an exception before the manual `Close()` call leaks the OS handle, and finalization is non-deterministic — under load this exhausts the process handle quota.
- **Forgetting `FileOptions.Asynchronous`** — Without this flag, async reads/writes block a thread pool thread, defeating the purpose of async I/O.
  - **Why it looks correct:** `ReadAsync` compiles and returns a `Task` without error, but internally it queues the operation on a thread pool thread that blocks on the synchronous `ReadFile` call, achieving zero scalability benefit despite the async syntax.
- **Large synchronous reads on UI thread** — `File.ReadAllBytes(hugePath)` blocks the UI thread for seconds. Use async variants.
  - **Why it looks correct:** `ReadAllBytes` is a simple one-liner that returns a byte array, but it issues a synchronous OS read that blocks the calling thread entirely — on the UI thread this freezes the application and triggers the "not responding" state in Windows.
- **Not handling partial reads** — `Stream.Read` may return fewer bytes than requested. Must loop until all bytes are read or end of stream.
  - **Why it looks correct:** `Read` accepts a `count` parameter and most code assumes it fills the buffer, but the contract only guarantees 1 to `count` bytes — network streams especially return whatever is available in the buffer, not what was requested.
- **Writing to a closed stream** — Disposing a `StreamWriter` that wraps a `MemoryStream` flushes to the already-disposed inner stream. Nest disposals correctly.
  - **Why it looks correct:** the `StreamWriter` is disposed in a `using` block, but dispose order is outer-first — if the inner `MemoryStream` is disposed before the wrapper, the wrapper's `Flush` tries to write to a closed stream and throws `ObjectDisposedException`.
- **Encoding BOM issues** — `new StreamWriter(path)` writes UTF-8 BOM by default. Use `new UTF8Encoding(false)` for no BOM.
  - **Why it looks correct:** the file opens correctly in Notepad and most editors, but the UTF-8 BOM (`EF BB BF`) causes issues in Unix pipelines, JSON parsers, and HTTP response bodies where the preamble is misinterpreted as data.
- **FileShare violation** — Opening a file with `FileShare.None` while another process has it open throws `IOException`. Use appropriate `FileShare` flags.
  - **Why it looks correct:** `FileShare.None` appears to be the safest option (exclusive access), but it prevents any concurrent reader including antivirus scanners and log tailers, causing spurious `IOException` failures in production that are hard to reproduce in development.

```csharp
// Handling partial reads
int totalRead = 0;
while (totalRead < buffer.Length)
{
    int bytesRead = stream.Read(buffer, totalRead, buffer.Length - totalRead);
    if (bytesRead == 0) break;
    totalRead += bytesRead;
}
```

---

## Key Design Considerations

- **Async IO is not always faster** — For small operations (< 4KB), sync may be faster. Async prevents thread starvation under high concurrency, not per-request speed.
- **Pool buffers to reduce GC pressure** — Use `ArrayPool<byte>.Shared.Rent(n)` for temporary buffers instead of allocating new `byte[]` each time. Return with `ArrayPool<byte>.Shared.Return(buffer)`.
- **Never assume `Stream.Length` is available** — Network streams, `CryptoStream`, and `GZipStream` don't support seeking. Check `CanSeek` before using `Length` or `Position`.
- **`System.IO.Pipelines` for high-throughput** — `PipeReader`/`PipeWriter` provide zero-copy I/O with user-managed buffer lifecycles, backpressure via `Advance`/`Complete`, and integration with `MemoryPool<T>`. Preferred over raw `Stream` for server scenarios.
- **Memory-mapped files for random access** — Ideal for large files with non-sequential access patterns (databases, caches). Not beneficial for small files or purely sequential reads.

```csharp
// Pool buffers to reduce GC
byte[] buffer = ArrayPool<byte>.Shared.Rent(81920);
try { int read = await stream.ReadAsync(buffer, 0, buffer.Length); }
finally { ArrayPool<byte>.Shared.Return(buffer); }
```

---

## Real-World Scenarios

### Scenario 1: Large File Upload with Virus Scanning Pipeline
**Context:** A web API accepts file uploads up to 2GB. Files must be scanned for malware before being stored permanently. Cannot buffer the entire file in memory.

```csharp
public class SecureFileUploadService
{
    public async Task<UploadResult> UploadAsync(Stream uploadStream, string fileName, CancellationToken ct)
    {
        // Phase 1: Stream to temp file while scanning
        var tempPath = Path.Combine(Path.GetTempPath(), Guid.NewGuid().ToString());
        try
        {
            await using var tempFile = new FileStream(tempPath, FileMode.Create, FileAccess.Write,
                FileShare.None, 81920, FileOptions.Asynchronous | FileOptions.SequentialScan);

            // Scan while writing: chain decorators
            await using var scanStream = new AntivirusScanStream(tempFile, _scanner);
            await uploadStream.CopyToAsync(scanStream, ct);

            if (scanStream.IsInfected)
                return UploadResult.Rejected("File contains malware");

            // Phase 2: Encrypt and move to permanent storage
            var permPath = GetPermanentPath(fileName);
            await using var source = new FileStream(tempPath, FileMode.Open, FileAccess.Read,
                FileShare.Read, 81920, FileOptions.Asynchronous | FileOptions.SequentialScan);
            await using var dest = new FileStream(permPath, FileMode.Create, FileAccess.Write,
                FileShare.None, 81920, FileOptions.Asynchronous);
            await using var aes = Aes.Create();
            await using var cryptoStream = new CryptoStream(dest, aes.CreateEncryptor(), CryptoStreamMode.Write);
            await source.CopyToAsync(cryptoStream, ct);

            return UploadResult.Success(permPath);
        }
        finally
        {
            if (File.Exists(tempPath)) File.Delete(tempPath);
        }
    }
}
```

### Scenario 2: High-Throughput Log File Tailer with Zero-Copy
**Context:** A monitoring service must tail multiple large log files (100MB+ each), filter lines in real-time, and forward matches over a network stream. Must minimize allocations.

```csharp
public class LogTailerService
{
    private readonly Pipe _pipe = new(new PipeOptions(useSynchronizationContext: false));
    private readonly Stream _networkStream;

    public async Task TailAndForwardAsync(string logPath, string filterPattern, CancellationToken ct)
    {
        // Reader: File → Pipe
        var fileReader = Task.Run(() => ReadFileToPipeAsync(logPath, ct), ct);
        // Writer: Pipe → Network (with line filtering)
        var networkWriter = Task.Run(() => WriteFilteredToNetworkAsync(filterPattern, ct), ct);

        await Task.WhenAll(fileReader, networkWriter);
    }

    private async Task ReadFileToPipeAsync(string path, CancellationToken ct)
    {
        await using var fs = new FileStream(path, FileMode.Open, FileAccess.Read,
            FileShare.ReadWrite, 65536, FileOptions.Asynchronous | FileOptions.SequentialScan);

        // Use Pipe directly for zero-copy from file to processing
        await fs.CopyToAsync(_pipe.Writer.AsStream(), ct);
        await _pipe.Writer.CompleteAsync();
    }

    private async Task WriteFilteredToNetworkAsync(string pattern, CancellationToken ct)
    {
        var reader = _pipe.Reader;
        while (true)
        {
            var result = await reader.ReadAsync(ct);
            var buffer = result.Buffer;

            // Process lines without copying (ReadOnlySequence<byte>)
            foreach (var line in buffer.SplitOnNewlines())
            {
                if (line.Span.Contains(pattern, StringComparison.OrdinalIgnoreCase))
                {
                    // Write filtered line directly to network
                    await _networkStream.WriteAsync(line.ToArray(), ct);
                }
            }

            reader.AdvanceTo(buffer.End);
            if (result.IsCompleted) break;
        }
    }
}
```

### Scenario 3: CSV File Processing with Memory-Mapped Files
**Context:** A data pipeline imports 5GB CSV files from financial exchanges. Random access to specific rows is needed for validation, then sequential processing.

```csharp
public class CsvMmFileProcessor
{
    public async Task ProcessLargeCsvAsync(string path, CancellationToken ct)
    {
        // Use memory-mapped file for random row access during validation
        using var mmf = MemoryMappedFile.CreateFromFile(path, FileMode.Open, null, 0, MemoryMappedFileAccess.Read);
        using var view = mmf.CreateViewAccessor(0, 0, MemoryMappedFileAccess.Read);

        long rowCount = GetRowCount(view); // Random access to find row boundaries
        var validationErrors = new ConcurrentBag<string>();

        // Validate rows in parallel (random access)
        Parallel.For(0, rowCount, new ParallelOptions { MaxDegreeOfParallelism = Environment.ProcessorCount }, i =>
        {
            var row = ReadRow(view, i);
            if (!ValidateRow(row)) validationErrors.Add($"Row {i}: {row}");
        });

        if (!validationErrors.IsEmpty)
            throw new ValidationException(string.Join("\n", validationErrors.Take(10)));

        // Sequential streaming for actual ETL (more efficient for bulk)
        await using var stream = File.OpenRead(path);
        using var reader = new StreamReader(stream);
        string? header = await reader.ReadLineAsync(ct);
        string? line;
        while ((line = await reader.ReadLineAsync(ct)) != null && !ct.IsCancellationRequested)
        {
            await ProcessRowAsync(line, ct);
        }
    }

    private unsafe string ReadRow(MemoryMappedViewAccessor view, long rowIndex)
    {
        byte* ptr = null;
        view.SafeMemoryMappedViewHandle.AcquirePointer(ref ptr);
        try { return ReadRowFromPointer(ptr, rowIndex); }
        finally { view.SafeMemoryMappedViewHandle.ReleasePointer(); }
    }
}
```

## Use Cases

- **File read/write for data processing** — reading configuration files, writing logs, or processing data files
  - `FileStream` with buffering for sequential access. `StreamReader`/`StreamWriter` for text. `BinaryReader`/`BinaryWriter` for structured binary data.
  - **Avoid when:** files are small (<1 KB) and read infrequently — `File.ReadAllText`/`File.WriteAllText` provide simpler one-shot access.

- **Network stream communication** — reading from HTTP responses, TCP sockets, or named pipes
  - `NetworkStream` provides a stream abstraction over sockets. `HttpClient.GetStreamAsync` streams responses without buffering the entire body.
  - **Avoid when:** you need request-response semantics — higher-level abstractions (HttpClient, SignalR) handle framing and error handling.

- **Memory-mapped files for large data** — processing multi-GB files without loading them entirely into memory
  - `MemoryMappedFile` maps a file region into virtual memory. OS handles paging. Enables random access to large files with low memory footprint.
  - **Avoid when:** access is strictly sequential — `FileStream` with buffering is simpler and may be faster.

- **Compression/decompression streams** — compressing log files or decompressing downloaded archives on-the-fly
  - `GZipStream`/`DeflateStream` wrap another stream and apply compression. Chain with file streams for transparent compression during read/write.
  - **Avoid when:** compression is CPU-bound and blocks the main thread — compress on a background thread or use async I/O.

- **Pipes and inter-process communication** — passing data between processes on the same machine
  - `NamedPipeServerStream`/`NamedPipeClientStream` for bidirectional IPC. Anonymous pipes for parent-child process communication.
  - **Avoid when:** processes are on different machines — use TCP/IP sockets or a message queue instead.

---

## Scenario-Based Questions

1. **Q: You are building a logging system that writes 50K log entries/second. `FileStream` writes are causing GC pressure from string encoding allocations. How do you optimize?**
   - **A:** Use `ArrayPool<byte>.Shared.Rent()` to reuse buffers for encoding. Pre-encode log format templates into `byte[]` at initialization, avoiding repeated `Encoding.UTF8.GetBytes()`. Use `FileStream` with `FileOptions.Asynchronous | FileOptions.WriteThrough` for durability. Batch writes with a `Channel<byte[]>` and flush every 100ms or 64KB. Consider `System.IO.Pipelines` for zero-copy buffer management. Measure: encoding allocation can be 50%+ of GC cost in log-heavy services.

2. **Q: You need to process a 50GB file on a server with 16GB RAM. The processing requires sorting records by timestamp. How do you approach this with streams?**
   - **A:** Implement an external merge sort: split the file into sorted chunks using `FileStream` (sequential read/write of each chunk), write each sorted chunk to temp files, then merge-sort the chunks by reading them simultaneously with separate `FileStream` instances. Use `StreamReader` for line-based processing. For the merge phase, maintain a priority queue of `(record, chunkStream)` pairs — read a line from each chunk, push to heap, pop smallest and write to output. This is the classic external sort pattern requiring only O(chunk_count) memory.

3. **Q: You are designing a microservice that receives multipart form uploads and streams them directly to Azure Blob Storage without touching disk. How?**
   - **A:** Use `HttpRequest.Body` as the source stream. In ASP.NET Core, the request body is a `FileBufferingReadStream` by default — disable buffering with `Request.EnableBuffering()` (actually, disable by not calling it). Use `BlobClient.UploadAsync(Stream, ...)` which accepts a `Stream` — it reads from the request stream directly. For progress reporting, wrap the request body in a `ProgressStream` decorator that updates a counter. For cancellation, pass `HttpContext.RequestAborted`. Challenge: without buffering, a slow client ties up the connection; set `KestrelLimits.KeepAliveTimeout` appropriately.
   - **Interview follow-up:** If the upstream connection is slower than the downstream blob write, what backpressure mechanism prevents the request stream from being read faster than the client sends data?

4. **Q: You are migrating from `BinaryFormatter` to `System.Text.Json` for serialization of a custom stream-based protocol. What stream patterns do you use?**
   - **A:** For JSON serialization, use `Utf8JsonWriter` which writes directly to an `IBufferWriter<byte>` — zero-copy, low allocation. For deserialization, use `Utf8JsonReader` over a `ReadOnlySequence<byte>` from `PipeReader`. Use `Stream` adapters: `TextReader` → `StreamReader` (bytes to chars), `TextWriter` → `StreamWriter`. For length-prefixed messages, use `BinaryReader`/`BinaryWriter` on a `NetworkStream`. Prefer protocol buffers (`Protobuf`) over custom binary for cross-platform scenarios.

5. **Q: You have a `NetworkStream` that sometimes reads partial messages. How do you handle message framing correctly?**
   - **A:** Never assume a single `ReadAsync` returns a complete message. Use length-prefixed framing: read 4 bytes (message length as `int` via `BinaryReader` or manually), then loop-read until `totalRead == length`. For text protocols (HTTP, WebSocket), look for delimiters (`\r\n\r\n`). Use `StreamReader.ReadLineAsync` for line-based protocols. For high performance, use `PipeReader` — it handles incomplete reads natively via `TryRead` and `AdvanceTo`. Benchmark: `PipeReader` can be 3-5x faster than manual buffering for partial reads.
   - **Interview follow-up:** How do you handle a malicious client that sends a length prefix indicating a 4GB message — does your framing code protect against memory exhaustion?

6. **Q: You are building a compression service that needs to compress 100 files simultaneously without exhausting memory. How do you control resource usage?**
   - **A:** Use `SemaphoreSlim` to limit concurrent compression operations (e.g., `new SemaphoreSlim(Environment.ProcessorCount)`). Each operation reads via `FileStream` and writes compressed output through `GZipStream` or `BrotliStream` (whichever offers better compression ratio for the data type). Use `ArrayPool<byte>` for copy buffers to avoid large allocations. For very large files, compress in chunks using `GZipStream` in a streaming fashion — never read the entire file into memory first. Set `CompressionLevel.Optimal` for archival, `Fastest` for real-time.

7. **Q: You need to implement a `Stream` wrapper that transparently encrypts data during write and decrypts during read. What's the correct dispose pattern?**
   - **A:** Create a decorator stream wrapping a `CryptoStream` inside a `FileStream`. Dispose order matters: outer stream first (flushes crypto), then inner stream (closes file). Use `await using` with the `CryptoStream` created with `leaveOpen: false` (default). For the reverse (read: decrypt → decompress), chain as `StreamReader(DecompressStream(CryptoStream(FileStream)))`. Always dispose the outer-most stream first — it triggers `FlushFinalBlock` on `CryptoStream`, which writes the final block. If dispose order is wrong, the file will be truncated.

8. **Q: You are processing a stream where the data source can produce data faster than the consumer can process. How do you implement backpressure?**
   - **A:** Use `System.IO.Pipelines` — the `PipeWriter.FlushAsync()` returns incomplete if the pipe buffer exceeds `PauseWriterThreshold`, naturally providing backpressure. The consumer controls the flow via `AdvanceTo`. For `Stream`-based pipelines, use a bounded `Channel<byte[]>`: the producer awaits `WriteAsync` (blocks when full), the consumer reads as fast as it can. Set channel capacity to limit memory usage. Trade-off: `Pipelines` is more efficient (zero-copy) but `Channel` is simpler.

9. **Q: You are reading from a `NetworkStream` in an async loop, but under heavy load you see increased latency and allocation. What's happening and how do you fix it?**
   - **A:** Each `ReadAsync` allocates a `Task` and possibly a `SocketAsyncEventArgs`. The allocation adds GC pressure. Fix: use `PipeReader` which reuses buffers from a `MemoryPool<T>`, reducing allocations. Also check for the Nagle algorithm — disable via `Socket.NoDelay = true` for low-latency scenarios. Use `SocketAsyncEventArgs` directly (or `NetworkStream` with `Pipe`) for the fastest path. If using `NetworkStream`, increase the buffer size to reduce ReadAsync calls.

10. **Q: You are implementing a file watcher that reads new lines appended to a growing log file. How do you handle this efficiently?**
     - **A:** Open the file with `FileShare.ReadWrite` to allow concurrent writes. Use `FileStream` with `FileOptions.Asynchronous` and `FileOptions.SequentialScan`. Track `_lastPosition` and poll `stream.Length` periodically. Use `StreamReader` to read from the last position. For low latency, use `ReadDirectoryChangesW` via `FileSystemWatcher` to detect changes, then read the new bytes. For the stream position, use `stream.Seek(0, SeekOrigin.End)` to skip to the current end on open. Edge case: log rotation — detect by comparing file handle's `FileAttributes` with the original.
    - **Interview follow-up:** If the log file is truncated (not rotated) by a log shipper, how does your polling loop detect that `Length` has decreased and recover without crashing or reading stale data?

---

## Interview Questions

1. **What is the difference between `FileStream` and `MemoryStream`?**
   - **A:** `FileStream` reads/writes bytes to a file on disk using OS file handles. `MemoryStream` reads/writes bytes to an in-memory `byte[]` buffer. `FileStream` supports async I/O that frees threads; `MemoryStream` operations are always synchronous (memory copy).

2. **What does `FileOptions.Asynchronous` do?**
   - **A:** It tells the OS to open the file for overlapped I/O (Windows) or `O_NONBLOCK` (Linux). Without it, `ReadAsync`/`WriteAsync` still block a thread pool thread internally. True async I/O frees the thread entirely during the I/O wait, improving scalability.

3. **How do Stream decorators work?**
   - **A:** Stream decorators wrap another `Stream` and add behavior. `GZipStream` adds compression/decompression, `CryptoStream` adds encryption/decryption, `BufferedStream` adds buffering. They can be chained: `new StreamReader(new GZipStream(new FileStream(...)))`. Dispose order matters — outer first, inner second.

4. **What is `System.IO.Pipelines`?**
   - **A:** A high-performance I/O API providing `PipeReader` and `PipeWriter` for zero-copy, user-managed buffer lifecycles. Unlike `Stream`, it gives the caller control over buffer allocation, consumption, and backpressure via `AdvanceTo`. Preferred for server workloads where allocation matters.

5. **How does `MemoryMappedFile` differ from `FileStream`?**
   - **A:** `MemoryMappedFile` maps file pages into virtual memory — reads are memory accesses (no syscalls), ~25x faster for random access. `FileStream` requires `Seek` + `Read` syscalls for each access. MMF is ideal for large files with random access patterns; `FileStream` is better for sequential reads.

6. **What causes `IOException: The process cannot access the file`?**
   - **A:** The file is locked by another process that opened it without `FileShare.Read` or `FileShare.Write`. Common causes: antivirus scanning, another instance of the application, or a previous `FileStream` that wasn't disposed. Fix: use appropriate `FileShare` flags and ensure proper disposal.

7. **How does `StreamReader` handle encoding detection?**
   - **A:** It reads the first few bytes and checks for a BOM: UTF-8 (`EF BB BF`), UTF-16 LE (`FF FE`), UTF-16 BE (`FE FF`), UTF-32 LE (`FF FE 00 00`). If no BOM is found, it defaults to UTF-8. You can override by specifying the encoding explicitly in the constructor.

8. **What is `BufferedStream` and when would you use it?**
   - **A:** A decorator stream that adds buffering to an unbuffered stream (like `NetworkStream`). It reduces the number of OS calls by reading/writing in larger chunks. Default buffer is 4096 bytes. Typically unnecessary for `FileStream` (which has its own buffer) or `MemoryStream` (which is already in memory).

9. **Explain the dispose pattern for chained streams.**
   - **A:** Dispose the outermost stream first. The outer stream flushes its buffer (e.g., `GZipStream` writes pending compressed data, `CryptoStream` writes the final block), then disposes the inner stream. If you dispose the inner stream first, the outer stream can't flush and data is lost. Use `await using` and `leaveOpen: false` (default).

10. **How do you handle partial reads from a stream?**
    - **A:** `Stream.Read` returns the actual number of bytes read (0 for end of stream). Always loop until the desired count is reached or end of stream. In .NET 7+, `Stream.ReadExactly` handles this automatically. `BinaryReader.ReadBytes` also guarantees reading the specified count.

---

## Developer Recommendations

- **Use `FileOptions.Asynchronous` for all file I/O in server applications** — Without it, async file operations block a thread pool thread, defeating the purpose of async I/O. Always combine with `FileOptions.SequentialScan` for large sequential reads to optimize OS caching.
  - **Production story:** A high-throughput file processing service once omitted this flag, causing thread pool starvation under load — every "async" file read blocked a thread for 100ms+, and with 200 concurrent requests the thread pool ran out of available threads and request latency spiked to 30 seconds.

- **Prefer `System.IO.Pipelines` over `Stream` for high-throughput server scenarios** — `Pipelines` provides zero-copy buffer management with user-controlled allocation via `MemoryPool<T>`. The `AdvanceTo` API gives explicit backpressure. Throughput can be 3-5x higher than equivalent `Stream` code for network I/O.
  - **Production story:** A telemetry ingestion gateway migrating from `NetworkStream` + manual buffering to `PipeReader`/`PipeWriter` reduced per-message allocation by 80 % and eliminated GC pauses that had been causing periodic p99 latency spikes from 50ms to 2s.

- **Always use `ArrayPool<byte>.Shared.Rent()` instead of `new byte[n]` for temporary buffers** — The GC cost of allocating large byte arrays is significant. Pooled buffers are reused, reducing GC Gen 2 collections. Return the buffer in a `finally` block. For buffers holding sensitive data, pass `clearArray: true` to `Return`.

- **Never assume `Stream.Length` is available** — `NetworkStream`, `CryptoStream`, and `GZipStream` don't support seeking. Check `CanSeek` before using `Length` or `Position`. For non-seekable streams, use a counting wrapper stream to track bytes read/written.

- **Chain decorator streams in the correct order** — For reading: create the outermost (text) first, but dispose it last. For writing: `CryptoStream(GZipStream(FileStream))` means encrypt then compress — data is encrypted, not compressible. Order matters: compress first, then encrypt (`GZipStream(CryptoStream(FileStream))`).

- **Use memory-mapped files for random access patterns, not sequential reads** — `MemoryMappedFile` is ~25x faster for random access but offers no benefit for sequential scans. For databases and large configuration files with non-sequential lookups, MMF is ideal. For log processing and bulk data transfer, `FileStream` is simpler and equally efficient.

- **Prefer `Stream.ReadExactly` (.NET 7+) over manual read loops** — Manual read loops are error-prone and repetitive. `ReadExactly` handles partial reads, end-of-stream detection, and timeouts consistently. For earlier .NET versions, use `BinaryReader.ReadBytes` or a helper method.

---

## Stream Type Comparison

| Stream | Seekable | Buffered | Async | Best For |
|---|---|---|---|---|
| `FileStream` | Yes* | Yes | Yes | File I/O |
| `MemoryStream` | Yes | N/A | Yes | In-memory data |
| `NetworkStream` | No | No | Yes | Network I/O |
| `GZipStream` | No | Internal | Yes | Compression |
| `CryptoStream` | No | Internal | No | Encryption |
| `SslStream` | No | Internal | Yes | TLS |
| `PipeStream` | No | Internal | Yes | IPC |
| MMF Accessor | Yes | OS page | No | Random access |

\*Files on disk only; not pipes/console
