# IO Streams

## 1. Executive Summary

IO streams in C# form the foundation for reading and writing data across diverse sources: files, memory, networks, pipes, and compression. The `System.IO` namespace provides abstract `Stream`, concrete implementations (`FileStream`, `MemoryStream`, `NetworkStream`), and decorators (`BufferedStream`, `GZipStream`, `CryptoStream`). Understanding stream architecture, buffer management, async IO, and disposal patterns is critical for building robust, high-performance applications.

## 2. Core Theory

### Stream Hierarchy

```
Stream (abstract)
+-- FileStream
+-- MemoryStream
+-- NetworkStream
+-- GZipStream / DeflateStream / BrotliStream
+-- CryptoStream
+-- BufferedStream
+-- UnmanagedMemoryStream
+-- PipeStream (anonymous/named pipes)
+-- SslStream
+-- (Ionic.Zlib, etc. third party)
```

### Reader/Writer Wrappers

```
Stream (binary bytes)
+-- TextReader / TextWriter (abstract)
|   +-- StreamReader / StreamWriter (encoding-aware)
|   +-- StringReader / StringWriter
+-- BinaryReader / BinaryWriter (primitive types)
```

### Core Stream Members

```csharp
public abstract class Stream : IDisposable, IAsyncDisposable
{
    // Read/Write primitives
    public abstract int Read(byte[] buffer, int offset, int count);
    public abstract void Write(byte[] buffer, int offset, int count);
    public abstract Task<int> ReadAsync(byte[] buffer, int offset, int count,
                                        CancellationToken cancellationToken);
    public abstract ValueTask WriteAsync(ReadOnlyMemory<byte> buffer,
                                         CancellationToken cancellationToken = default);

    // Position and length
    public abstract long Position { get; set; }
    public abstract long Length { get; }
    public abstract long Seek(long offset, SeekOrigin origin);
    public abstract void SetLength(long value);

    // Capabilities
    public abstract bool CanRead { get; }
    public abstract bool CanWrite { get; }
    public abstract bool CanSeek { get; }
    public virtual bool CanTimeout => false;

    // Flush
    public abstract void Flush();
    public abstract Task FlushAsync(CancellationToken cancellationToken = default);

    // Timeouts
    public virtual int ReadTimeout { get; set; }
    public virtual int WriteTimeout { get; set; }

    // Copying
    public void CopyTo(Stream destination, int bufferSize = 81920);
    public Task CopyToAsync(Stream destination, int bufferSize = 81920,
                            CancellationToken cancellationToken = default);
}
```

## 3. Under-the-Hood Deep Dive

### FileStream Internals

```csharp
// FileStream core mechanisms:
// - On Windows: uses CreateFile/ReadFile/WriteFile (Win32 API)
// - On Linux: uses open/read/write (syscalls)
// - Asynchronous IO: uses OVERLAPPED on Windows, epoll on Linux

// Internal buffer (can be bypassed with FileOptions.WriteThrough)
// Default buffer = 4096 bytes (page size)
// FileStream caches a byte[] _buffer internally for buffered operations

// Key flags:
// FileOptions.Asynchronous -> enables async IO without blocking thread pool
// FileOptions.SequentialScan -> hints OS for read-ahead
// FileOptions.RandomAccess -> hints OS to avoid read-ahead
// FileOptions.WriteThrough -> bypass OS cache, write directly to disk

// FileShare flags:
// FileShare.Read   -> other processes can read
// FileShare.Write  -> other processes can write
// FileShare.Delete -> other processes can delete
// FileShare.None   -> exclusive access
```

### MemoryStream Internals

```csharp
// MemoryStream wraps a byte[] _buffer
// The buffer can be:
// - Provided externally (not resizable, not expandable)
// - Internally allocated (resizable, grows like List<byte>)

// GetBuffer() returns the internal array (even unused bytes)
// ToArray() returns a copy (allocates)
// TryGetBuffer() returns ArraySegment<byte> (.NET 4.5+)

internal class MemoryStream : Stream
{
    private byte[] _buffer;
    private int _origin;       // For GetBuffer() offset
    private int _position;
    private int _length;
    private int _capacity;
    private bool _expandable;
    private bool _writable;
    private bool _exposable;   // Whether GetBuffer() can be called

    // Capacity grows: max(buffer.Length * 2, _length + count)
    // Buffer is NOT zero-initialized beyond _length
}
```

### StreamReader Encoding

```csharp
// StreamReader detects encoding via BOM (Byte Order Mark):
// - UTF-8:     EF BB BF
// - UTF-16 LE: FF FE
// - UTF-16 BE: FE FF
// - UTF-32 LE: FF FE 00 00
// - UTF-32 BE: 00 00 FE FF

// Internal buffer: 1024 bytes (by default) for decoding
// Decoder: converts bytes to chars using the encoding
// Preample detection: reads first bytes, checks for BOM
```

### Decorator Pattern in Streams

```csharp
// Streams use the decorator pattern:
// FileStream file = new("data.bin", FileMode.Open);
// GZipStream gzip = new(file, CompressionMode.Decompress);
// CryptoStream crypto = new(gzip, aes.CreateDecryptor(), CryptoStreamMode.Read);
// StreamReader reader = new(crypto);

// Each decorator wraps the inner stream and adds behavior.
// Dispose propagates (unless leaveOpen=true).
```

## 4. Production Code Examples

```csharp
// Safe file reading with async and cancellation
public async Task<string> ReadFileSafeAsync(string path, CancellationToken ct = default)
{
    var fileInfo = new FileInfo(path);
    if (!fileInfo.Exists)
        throw new FileNotFoundException("File not found", path);

    if (fileInfo.Length > 100 * 1024 * 1024) // 100 MB limit
        throw new IOException("File too large");

    using var stream = new FileStream(
        path,
        FileMode.Open,
        FileAccess.Read,
        FileShare.Read,
        4096,
        FileOptions.Asynchronous | FileOptions.SequentialScan);

    using var reader = new StreamReader(stream, Encoding.UTF8);
    return await reader.ReadToEndAsync(ct);
}
```

```csharp
// Streaming large file processing (no memory spike)
public async IAsyncEnumerable<string> ReadLinesAsync(string path,
    [EnumeratorCancellation] CancellationToken ct = default)
{
    using var stream = new FileStream(
        path, FileMode.Open, FileAccess.Read, FileShare.Read, 65536,
        FileOptions.Asynchronous | FileOptions.SequentialScan);
    using var reader = new StreamReader(stream, Encoding.UTF8);

    string line;
    while ((line = await reader.ReadLineAsync(ct)) is not null)
        yield return line;
}
```

```csharp
// Compress and encrypt a file
public async Task CompressAndEncryptAsync(string sourcePath, string destPath,
    byte[] key, byte[] iv)
{
    using var source = File.OpenRead(sourcePath);
    using var dest = File.Create(destPath);

    using var aes = Aes.Create();
    aes.Key = key;
    aes.IV = iv;

    await using var cryptoStream = new CryptoStream(
        dest, aes.CreateEncryptor(), CryptoStreamMode.Write);
    await using var gzipStream = new GZipStream(
        cryptoStream, CompressionLevel.Optimal);

    await source.CopyToAsync(gzipStream);
}
```

```csharp
// Memory-mapped file for high-performance random access
public class MemoryMappedReader : IDisposable
{
    private readonly MemoryMappedFile _mmf;
    private readonly MemoryMappedViewAccessor _accessor;

    public MemoryMappedReader(string path)
    {
        _mmf = MemoryMappedFile.CreateFromFile(
            path, FileMode.Open, null, 0, MemoryMappedFileAccess.Read);
        _accessor = _mmf.CreateViewAccessor(0, 0, MemoryMappedFileAccess.Read);
    }

    public int ReadInt32(long offset)
    {
        return _accessor.ReadInt32(offset);
    }

    public unsafe double ReadDouble(long offset)
    {
        byte* ptr = (byte*)_accessor.SafeMemoryMappedViewHandle.DangerousGetHandle();
        return *(double*)(ptr + offset);
    }

    public void Dispose()
    {
        _accessor?.Dispose();
        _mmf?.Dispose();
    }
}
```

```csharp
// Pipes for inter-process communication
public class PipeServer
{
    private readonly NamedPipeServerStream _pipe;

    public PipeServer(string pipeName)
    {
        _pipe = new NamedPipeServerStream(
            pipeName, PipeDirection.InOut, 1,
            PipeTransmissionMode.Byte, PipeOptions.Asynchronous);
    }

    public async Task RunAsync(CancellationToken ct)
    {
        await _pipe.WaitForConnectionAsync(ct);
        using var reader = new StreamReader(_pipe);
        using var writer = new StreamWriter(_pipe) { AutoFlush = true };

        string request = await reader.ReadLineAsync(ct);
        string response = ProcessRequest(request);
        await writer.WriteLineAsync(response);
    }
}
```

```csharp
// Custom stream that monitors throughput
public class MonitoringStream : Stream
{
    private readonly Stream _inner;
    private readonly Action<long, TimeSpan> _onData;

    private long _totalBytes;
    private readonly Stopwatch _stopwatch = Stopwatch.StartNew();

    public MonitoringStream(Stream inner, Action<long, TimeSpan> onData)
    {
        _inner = inner;
        _onData = onData;
    }

    public override async Task<int> ReadAsync(byte[] buffer, int offset, int count,
        CancellationToken ct)
    {
        int bytesRead = await _inner.ReadAsync(buffer, offset, count, ct);
        _totalBytes += bytesRead;
        _onData(_totalBytes, _stopwatch.Elapsed);
        return bytesRead;
    }

    // ... other overrides delegate to _inner
    public override bool CanRead => _inner.CanRead;
    public override bool CanSeek => _inner.CanSeek;
    public override bool CanWrite => _inner.CanWrite;
    public override long Length => _inner.Length;
    public override long Position { get => _inner.Position; set => _inner.Position = value; }
    public override void Flush() => _inner.Flush();
    public override int Read(byte[] buffer, int offset, int count) => _inner.Read(buffer, offset, count);
    public override long Seek(long offset, SeekOrigin origin) => _inner.Seek(offset, origin);
    public override void SetLength(long value) => _inner.SetLength(value);
    public override void Write(byte[] buffer, int offset, int count) => _inner.Write(buffer, offset, count);
}
```

## 5. Real-World Scenarios

**Scenario 1: Web Server Log Streaming**
- Use `FileStream` with `FileOptions.Asynchronous | FileOptions.WriteThrough` for append-only logs.
- Use `StreamWriter` with `AutoFlush = false` and periodic flush (batch writes).

**Scenario 2: Video Streaming Server**
- Use `FileStream` with sequential scan for large media files.
- Use `PipeStream` for transcoding pipeline.
- Implement range requests via `Seek` and `SetLength`.

**Scenario 3: Secure File Upload**
- Write to a temp file first, then `File.Move` (atomic on same volume).
- Use `CryptoStream` for encryption at rest.
- Use `BufferedStream` for performance.

**Scenario 4: Database Backup/Compression**
- Chain `FileStream` -> `GZipStream` -> write.
- Monitor progress via custom stream decorator.

## 6. Performance

```csharp
// Buffer size tuning
// For FileStream: default 4096; for large sequential IO, use 64K-1MB
using var fs = new FileStream(path, FileMode.Open, FileAccess.Read,
    FileShare.Read, 65536, FileOptions.SequentialScan);

// CopyTo uses 81920 default buffer; tune for your scenario
await source.CopyToAsync(dest, 131072); // 128KB buffer

// Disable intermediate buffering when using large buffers
using var fs = new FileStream(path, FileMode.Open, FileAccess.Read,
    FileShare.Read, 1, FileOptions.None);  // bufferSize=1 disables internal buffer

// Use MemoryMappedFile for:
// - Random access to large files
// - Shared memory between processes
// - Files larger than 2GB
```

### Benchmark Comparisons

| Operation                        | FileStream (sync) | FileStream (async) | MMF          |
|----------------------------------|-------------------|--------------------|--------------|
| Sequential read (100MB)          | ~300ms            | ~310ms             | ~280ms       |
| Random access (10K reads)        | ~50ms             | ~55ms              | ~2ms         |
| Write (100MB sequential)         | ~250ms            | ~260ms             | ~240ms       |
| Memory allocation                | Buffer            | Buffer + Task      | None (page)  |
| OS cache utilization             | Yes               | Yes                | Yes          |

## 7. Security

```csharp
// Path traversal prevention
public string SanitizePath(string userInput)
{
    // 1. Get full path and validate
    string fullPath = Path.GetFullPath(Path.Combine(_baseDirectory, userInput));

    // 2. Ensure it's within the base directory
    if (!fullPath.StartsWith(_baseDirectory, StringComparison.Ordinal))
        throw new UnauthorizedAccessException("Path traversal detected");

    // 3. Reject suspicious patterns
    if (fullPath.Contains(".."))
        throw new ArgumentException("Invalid path");

    return fullPath;
}

// Secure temp file creation
string tempFile = Path.GetTempFileName(); // Creates empty file with unique name
// OR for more control:
string tempDir = Path.Combine(Path.GetTempPath(), Guid.NewGuid().ToString());
Directory.CreateDirectory(tempDir);
string tempFile = Path.Combine(tempDir, "upload.tmp");

// File sharing security
using var fs = new FileStream(path, FileMode.Open, FileAccess.Read,
    FileShare.Read, 4096, FileOptions.Asynchronous);
// FileShare.None for exclusive access
```

```csharp
// Data integrity: compute hash while reading
public async Task<byte[]> ComputeHashAsync(string path, HashAlgorithm hasher)
{
    using var fs = File.OpenRead(path);
    // CryptoStream to hash on-the-fly
    using var cs = new CryptoStream(Stream.Null, hasher, CryptoStreamMode.Write);
    await fs.CopyToAsync(cs);
    return hasher.Hash;
}
```

## 8. Common Mistakes

```csharp
// MISTAKE 1: Not disposing streams
var fs = new FileStream(path, FileMode.Open);
// ... if exception occurs, handle is leaked
// FIX: using statement

// MISTAKE 2: Forgetting async I/O needs FileOptions.Asynchronous
using var fs = new FileStream(path, FileMode.Open, FileAccess.Read,
    FileShare.Read, 4096, FileOptions.None);
await fs.ReadAsync(buffer, 0, buffer.Length); // Thread pool thread blocked!
// FIX: add FileOptions.Asynchronous

// MISTAKE 3: Large synchronous reads on UI thread
byte[] data = File.ReadAllBytes(hugePath); // Blocks UI for seconds
// FIX: await File.ReadAllBytesAsync(hugePath)

// MISTAKE 4: Not handling partial reads
int totalRead = 0;
while (totalRead < buffer.Length)
{
    int bytesRead = stream.Read(buffer, totalRead, buffer.Length - totalRead);
    if (bytesRead == 0) break; // End of stream
    totalRead += bytesRead;
}

// MISTAKE 5: Assuming Position is updated after async read
long pos = stream.Position;
int read = await stream.ReadAsync(buffer, 0, count);
// Position is now pos + read (but best practice: use the return value)

// MISTAKE 6: Writing to a closed stream
using (var ms = new MemoryStream())
using (var sw = new StreamWriter(ms))
{
    sw.Write("data");
} // ms disposed before sw; sw.Dispose flushes to disposed ms
// FIX: nest disposals correctly (inner disposed first)

// MISTAKE 7: Encoding BOM issues
using var sw = new StreamWriter(path); // Writes UTF-8 BOM by default
// FIX: new StreamWriter(path, new UTF8Encoding(false))  // no BOM

// MISTAKE 8: FileShare violation
using var fs1 = File.Open(path, FileMode.Open, FileAccess.Read, FileShare.None);
using var fs2 = File.Open(path, FileMode.Open, FileAccess.Read, FileShare.None); // IOException
// FIX: Use appropriate FileShare

// MISTAKE 9: Not flushing GZipStream/CryptoStream before dispose
using var gz = new GZipStream(fs, CompressionLevel.Optimal);
gz.Write(data);
// Dispose called at end of using -> final flush happens automatically
// But if you reuse the underlying stream: gz.Close() (not dispose) then seek back
```

## 9. Senior Engineer Perspective

**1. Async IO is not always faster.** For small operations (< 4KB), sync may be faster. Async prevents thread starvation under high concurrency.

**2. Pool buffers to reduce GC pressure.** Use `ArrayPool<byte>.Shared` for temporary buffers.

```csharp
byte[] buffer = ArrayPool<byte>.Shared.Rent(81920);
try
{
    int bytesRead = await stream.ReadAsync(buffer, 0, buffer.Length);
    // process buffer[0..bytesRead-1]
}
finally
{
    ArrayPool<byte>.Shared.Return(buffer);
}
```

**3. Use `Channel<T>` for producer-consumer stream processing.** Stream data in chunks through a channel.

**4. Never assume `Stream.Length` is available** (network streams, `CryptoStream` don't have it).

**5. For high-performance logging, use `System.IO.Pipelines`** (Pipe, PipeReader, PipeWriter) to avoid buffer copying.

**6. Memory-mapped files are ideal for large random-access workloads** but not for sequential reading of small files.

**7. Windows vs Linux differences:**
- `FileOptions.WriteThrough` has different semantics.
- File locking is advisory on Linux.
- Named pipes vs Unix domain sockets.

## 10. Interview Questions (Easy)

1. What is the abstract base class for all streams in .NET?
2. What is the difference between `Read()` and `ReadByte()`?
3. How does `FileStream` differ from `MemoryStream`?
4. What is `StreamReader` used for?
5. How do you correctly dispose a stream?
6. What is `BufferedStream`?
7. How do you check if a stream supports seeking?
8. What is the purpose of `Flush()`?
9. What does `CopyTo()` do?
10. How do you create a temp file in .NET?

## 11. Interview Questions (Medium)

1. Explain the decorator pattern used in .NET streams with examples.
2. How does async IO work under the hood (`ReadAsync` vs `BeginRead`)?
3. Compare `FileStream` with `MemoryMappedFile` for random access.
4. Explain BOM detection in `StreamReader`.
5. What is the difference between `FileShare.Read` and `FileShare.None`?
6. How does `GZipStream` handle compression internally?
7. Explain the internal buffer management of `BufferedStream`.
8. What is `FileOptions.Asynchronous` and why is it important?
9. How do you handle partial reads from a stream?
10. What is `UnmanagedMemoryStream` and when would you use it?

## 12. Advanced Interview Questions (Hard)

1. Implement a custom stream that transparently encrypts data using AES-GCM with authenticated encryption.
2. Design a `RecyclableMemoryStream` that avoids large object heap allocations.
3. Explain how `System.IO.Pipelines` works and compare it to traditional `Stream` API.
4. Implement a stream that supports concurrent reads and writes from multiple threads.
5. Design a `ChunkedStream` that reads from multiple underlying streams seamlessly.
6. Explain the kernel-level differences between sync and async file I/O on Windows.
7. Implement a `ThrottleStream` that limits bytes-per-second throughput.
8. Design a stream wrapper that tracks read/write statistics without allocating.
9. Explain how `Span<byte>` and `Memory<byte>` improved the Stream API in .NET Core.
10. Implement a lock-free ring buffer stream for inter-thread communication.

## 13. Interview Questions (System Design)

1. Design a distributed file system client using custom stream implementations.
2. Design a high-throughput log ingestion pipeline using `Channel<T>` and file streams.
3. Design a streaming ETL pipeline that transforms CSV to Parquet using decorator streams.
4. Design a secure file vault that encrypts/compresses all files transparently.
5. Design a multi-tier caching system (memory -> SSD -> network) using stream abstractions.
6. Design a real-time video transcoding pipeline using pipe streams.
7. Design a database backup system that streams compressed, encrypted snapshots.
8. Design a message queue broker using named pipe streams.
9. Design a cloud storage gateway that streams to S3/Azure Blob with retry logic.
10. Design a content-addressable storage system using streams and hashing.

## 14. Expert-Level Interview Questions (Architect)

1. Design a zero-allocation I/O framework using `System.IO.Pipelines`, `Span<T>`, and `MemoryPool<T>` that processes a million requests per second.
2. Architect a multi-tenant file storage system with per-tenant encryption, compression, and deduplication using custom stream chaining.
3. Design a streaming replication system (like Kafka) that guarantees exactly-once semantics using transactional file I/O and checkpointed streams.
4. Architect a high-performance RPC framework that uses `PipeWriter`/`PipeReader` for zero-copy serialization.
5. Design a distributed tracing system that correlates I/O operations across process boundaries using custom stream decorators.
6. Architect a cross-platform virtual file system that unifies local, network, and cloud storage behind a single Stream abstraction.
7. Design an I/O scheduler that reorders and coalesces write operations for SSD longevity and throughput.
8. Architect a streaming compression engine that works on arbitrarily large inputs using sliding windows and checkpoints.
9. Design a fault-tolerant stream abstraction that transparently retries on transient failures (network drop, disk full).
10. Architect a real-time collaborative editing system using Operational Transform over pipe streams.

## 15. Debugging & Troubleshooting

```csharp
// Detect file locks
public static string WhosUsingFile(string path)
{
    try
    {
        using var fs = File.Open(path, FileMode.Open, FileAccess.Read, FileShare.None);
        return "No one";
    }
    catch (IOException ex) when (ex.Message.Contains("being used"))
    {
        // On Windows: use Handle utility or Sysinternals Process Explorer
        return "File is locked by another process";
    }
}

// Common IO exceptions:
// - IOException: generic (disk full, handle invalid)
// - FileNotFoundException / DirectoryNotFoundException
// - UnauthorizedAccessException (permissions)
// - PathTooLongException (>260 chars on Windows, unless long paths enabled)
// - EndOfStreamException (reading past end)

// Windows troubleshooting:
// - Process Monitor (procmon) to trace file operations
// - Handle.exe to find who locked a file
// - fsutil to check disk health

// Linux troubleshooting:
// - strace to trace system calls
// - lsof to find open file handles
// - iostat for disk performance
```

## 16. Comparison Section

```
+------------------+------------+-----------+---------+----------+-----------+
| Stream Type      | Seekable   | Buffered  | Async   | Use Case | Overhead  |
+------------------+------------+-----------+---------+----------+-----------+
| FileStream       | Yes*       | Yes       | Yes     | Files    | Low       |
| MemoryStream     | Yes        | N/A       | Yes     | In-mem   | Very Low  |
| NetworkStream    | No         | No        | Yes     | Network  | Moderate  |
| GZipStream       | No         | Internal  | Yes     | Compress | High (CPU)|
| CryptoStream     | No         | Internal  | No      | Encrypt  | High (CPU)|
| BufferedStream   | Depends    | Yes       | Yes     | Wrapper  | Low       |
| SslStream        | No         | Internal  | Yes     | TLS      | High (CPU)|
| PipeStream       | No         | Internal  | Yes     | IPC      | Low       |
| MMF Accessor     | Yes        | OS page   | No      | Random   | Very Low  |
+------------------+------------+-----------+---------+----------+-----------+
*FileStream.CanSeek = true for files on disk, false for pipes/console

+--------------------+--------------------+----------------------+
| Aspect             | Stream             | PipeReader/Writer    |
+--------------------+--------------------+----------------------+
| Buffer management  | Internal           | User-managed         |
| Backpressure       | Not built-in       | Advance/Complete     |
| Allocation pattern | byte[] allocation  | MemoryPool<T>        |
| Cancellation       | CancellationToken  | CancellationToken    |
| Max throughput     | Good               | Excellent            |
| Complexity         | Low                | Higher               |
+--------------------+--------------------+----------------------+
```

## 17. Revision Notes

- `Stream` is abstract; always dispose with `using`.
- `FileStream` requires `FileOptions.Asynchronous` for true async I/O.
- `MemoryStream` wraps `byte[]`; use `ToArray()` for copy, `GetBuffer()` for direct access.
- Stream decorators wrap other streams; dispose order matters (inner first).
- `StreamReader` / `StreamWriter` handle encoding and BOM detection.
- `CopyTo`/`CopyToAsync` for efficient stream-to-stream transfer.
- `ArrayPool<byte>.Shared` for reusable buffers.
- `System.IO.Pipelines` for high-performance, zero-copy I/O.
- `MemoryMappedFile` for shared memory and large file random access.
- Always handle partial reads in loops.

## 18. Cheat Sheet

```
+------------------------------------------------------------------+
|                     IO STREAMS CHEAT SHEET                        |
+------------------------------------------------------------------+
| STREAM TYPES                                                      |
|  FileStream    - File I/O (sync/async)                           |
|  MemoryStream  - In-memory byte array                            |
|  NetworkStream - Network socket I/O                              |
|  GZipStream    - Compression/decompression                       |
|  CryptoStream  - Encryption/decryption                           |
|  SslStream     - TLS/SSL encrypted channel                       |
|  BufferedStream- Adds buffering to any stream                    |
+------------------------------------------------------------------+
| FILE I/O                                                         |
|  File.OpenRead(path)        - read-only stream                   |
|  File.OpenWrite(path)       - write-only stream                  |
|  File.Create(path)          - create/overwrite stream            |
|  File.Open(path, mode, access, share) - full control             |
|  File.ReadAllText(path)     - read all text (small files)        |
|  File.ReadAllBytes(path)    - read all bytes (small files)       |
|  File.WriteAllText(path, s) - write all text                     |
|  File.WriteAllBytes(path, b)- write all bytes                    |
+------------------------------------------------------------------+
| TEXT I/O                                                         |
|  StreamReader  - reads text from stream (encoding-aware)        |
|  StreamWriter  - writes text to stream                           |
|  StringReader  - reads from string                               |
|  StringWriter  - writes to StringBuilder                         |
+------------------------------------------------------------------+
| PATTERNS                                                         |
|  using var stream = new FileStream(...);  // auto dispose       |
|  await stream.ReadAsync(buf, 0, count);   // async read         |
|  await stream.WriteAsync(buf, 0, count);  // async write        |
|  await source.CopyToAsync(dest);           // pipe streams      |
+------------------------------------------------------------------+
| FLAGS & OPTIONS                                                   |
|  FileOptions.Asynchronous   - true async I/O                    |
|  FileOptions.SequentialScan - hint: read sequentially            |
|  FileOptions.RandomAccess   - hint: random access                |
|  FileOptions.WriteThrough   - bypass OS cache                   |
|  FileShare.Read/Write/Delete/None - sharing mode                 |
+------------------------------------------------------------------+
| PERFORMANCE TIPS                                                  |
|  Buffer size: 4096-131072; larger for sequential                 |
|  Use ArrayPool<byte> for temp buffers                            |
|  MMF for random access on large files                            |
|  Pipelines for high-throughput server I/O                        |
|  Async I/O requires FileOptions.Asynchronous                     |
|  Don't use streams on UI thread (use async variants)             |
+------------------------------------------------------------------+
