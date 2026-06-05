# Java I/O Streams

---

## 1. Executive Summary

### What Is It?
Java I/O (Input/Output) Streams are a fundamental API for reading from and writing to data sources — files, network sockets, memory buffers, system console — using a stream abstraction: a continuous flow of data.

### Architecture Types

| API | Package | I/O Model | Since |
|-----|---------|-----------|-------|
| **I/O Streams** | `java.io` | Blocking, byte/char | Java 1.0 |
| **NIO** | `java.nio` | Buffers, channels, selectors | Java 1.4 |
| **NIO.2** | `java.nio.file` | File system API, async channels | Java 7 |
| **Memory-Mapped** | `java.nio.MappedByteBuffer` | File → Direct memory | Java 1.4 |

### When to Use
- **File I/O**: NIO.2 `Files` + `Path` (preferred), legacy `File` + streams
- **Network I/O**: NIO `SocketChannel` + `Selector` for high concurrency; blocking `Socket` for simplicity
- **Memory I/O**: `ByteArrayInputStream`/`ByteArrayOutputStream`
- **Console I/O**: `System.in`, `System.out`, `Console`
- **Object Serialization**: `ObjectInputStream`/`ObjectOutputStream`
- **Buffered I/O**: `BufferedReader`, `BufferedInputStream` (always wrap streams with buffering)

### Key Principle: Always close resources
Use try-with-resources (Java 7+): `try (InputStream in = new FileInputStream("file")) { ... }`

---

## 2. Core Theory

### Stream Types

```
                  ┌──────────────┐
                  │   InputStream │
                  └──────┬───────┘
                         │
            ┌────────────┼────────────┐
            │            │            │
     ┌──────┴──────┐  ┌──┴─────┐  ┌──┴──────┐
     │FileInputStream│  │Filter │  │Object   │
     │ByteArray     │  │Stream │  │Stream   │
     └─────────────┘  └──┬─────┘  └─────────┘
                         │
              ┌──────────┼──────────┐
              │          │          │
        ┌─────┴────┐ ┌──┴──┐  ┌───┴────┐
        │Buffered  │ │Data │  │Pushback│
        │InputStream│ │Stream│  │Stream  │
        └──────────┘ └─────┘  └────────┘
```

### Byte Streams (8-bit)
```
InputStream                        OutputStream
├── FileInputStream                ├── FileOutputStream
├── ByteArrayInputStream           ├── ByteArrayOutputStream
├── BufferedInputStream            ├── BufferedOutputStream
├── DataInputStream                ├── DataOutputStream
├── ObjectInputStream              ├── ObjectOutputStream
├── PushbackInputStream            ├── PrintStream
├── SequenceInputStream            └── FilterOutputStream
└── FilterInputStream
```

### Character Streams (16-bit Unicode)
```
Reader                            Writer
├── FileReader                    ├── FileWriter
├── CharArrayReader               ├── CharArrayWriter
├── BufferedReader                ├── BufferedWriter
├── InputStreamReader             ├── OutputStreamWriter
├── StringReader                  ├── StringWriter
├── PushbackReader                ├── PrintWriter
└── FilterReader                  └── FilterWriter
```

### Bridge Streams (byte ↔ char)
- `InputStreamReader` — byte stream → character stream
- `OutputStreamWriter` — character stream → byte stream
- Always specify charset: `new InputStreamReader(in, StandardCharsets.UTF_8)`

### NIO.2 File API

```java
// Reading
Path path = Path.of("file.txt");

// All lines (small files)
List<String> lines = Files.readAllLines(path, StandardCharsets.UTF_8);
String content = Files.readString(path); // Java 11+

// Stream (large files — lazy)
try (Stream<String> lines = Files.lines(path)) {
    lines.filter(l -> l.contains("ERROR")).forEach(System.out::println);
}

// Writing
Files.writeString(path, "content"); // Java 11+
Files.write(path, lines);
Files.write(path, bytes);

// Copy/move/delete
Files.copy(source, target, StandardCopyOption.REPLACE_EXISTING);
Files.move(source, target);
Files.delete(path);

// Walk file tree (recursive)
try (Stream<Path> walk = Files.walk(rootDir)) {
    walk.filter(Files::isRegularFile)
        .filter(p -> p.toString().endsWith(".java"))
        .forEach(System.out::println);
}
```

---

## 3. Under-the-Hood Deep Dive

### Blocking I/O
Traditional I/O blocks the calling thread until the operation completes:
- `read()` blocks until data available
- `write()` blocks until data written
- Each connection needs its own thread

### NIO Non-Blocking I/O
`Selector` enables single-thread handling of multiple channels:
```java
Selector selector = Selector.open();
SocketChannel channel = SocketChannel.open();
channel.configureBlocking(false);
channel.register(selector, SelectionKey.OP_READ);

while (true) {
    selector.select(); // blocks until at least one channel is ready
    for (SelectionKey key : selector.selectedKeys()) {
        if (key.isReadable()) { /* read from channel */ }
        if (key.isWritable()) { /* write to channel */ }
    }
}
```

### Memory-Mapped Files
Maps file region directly into memory — OS handles paging:
```java
FileChannel channel = FileChannel.open(path, StandardOpenOption.READ);
MappedByteBuffer buffer = channel.map(
    FileChannel.MapMode.READ_ONLY, 0, channel.size()
);
// Read as if it's in memory
byte[] data = new byte[buffer.remaining()];
buffer.get(data);
```

Performance: 10-100x faster than traditional read() for large files.

### FileChannel vs Stream Performance

| Method | Small Files (<1MB) | Large Files (100MB+) |
|--------|-------------------|---------------------|
| `FileInputStream` + buffer | Moderate | Slow (many syscalls) |
| `BufferedInputStream` | Fast | Moderate |
| `FileChannel` + ByteBuffer | Fast | Fast |
| `MappedByteBuffer` | Overkill | Fastest |
| `Files.readAllBytes()` | Fastest | OOM risk |

---

## 4. Production Code Examples

### 4.1 Efficient File Copy

```java
// GOOD: NIO transferTo (zero-copy, OS-optimized)
public void copyFile(Path source, Path target) throws IOException {
    try (FileChannel in = FileChannel.open(source, StandardOpenOption.READ);
         FileChannel out = FileChannel.open(target, StandardOpenOption.CREATE_NEW, 
             StandardOpenOption.WRITE)) {
        in.transferTo(0, in.size(), out);
    }
}

// BAD: Manual byte-by-byte
public void badCopy(File source, File target) throws IOException {
    try (FileInputStream in = new FileInputStream(source);
         FileOutputStream out = new FileOutputStream(target)) {
        int b;
        while ((b = in.read()) != -1) { // One byte per syscall!
            out.write(b);
        }
    }
}
```

### 4.2 Large File Line Processing

```java
@Service
public class LogProcessor {
    public LogAnalysis analyzeLargeLog(Path logFile) throws IOException {
        LogAnalysis analysis = new LogAnalysis();
        
        try (Stream<String> lines = Files.lines(logFile, StandardCharsets.UTF_8)) {
            lines.forEach(line -> {
                LogEntry entry = LogEntry.parse(line);
                analysis.record(entry.getLevel());
                analysis.recordErrorSource(entry.getSource());
                if (entry.getLevel() == Level.ERROR) {
                    analysis.addErrorLine(line);
                }
            });
        }
        return analysis;
    }
}
```

### 4.3 Write with Buffering

```java
// GOOD: BufferedWriter with specific charset
public void writeReport(Path output, Report report) throws IOException {
    try (BufferedWriter writer = Files.newBufferedWriter(
            output, StandardCharsets.UTF_8, StandardOpenOption.CREATE,
            StandardOpenOption.TRUNCATE_EXISTING)) {
        writer.write("Report: " + report.getTitle());
        writer.newLine();
        for (ReportLine line : report.getLines()) {
            writer.write(String.format("| %-20s | %10.2f |", line.getName(), line.getValue()));
            writer.newLine();
        }
    }
}

// BAD: FileWriter without charset (uses default charset)
public void badWrite(File output, String content) throws IOException {
    try (FileWriter writer = new FileWriter(output)) {
        writer.write(content); // Platform-dependent encoding!
    }
}
```

### 4.4 Reading Binary Data (TCP Packet)

```java
public class PacketReader {
    public Packet readPacket(InputStream raw) throws IOException {
        // Use DataInputStream for primitive reading
        try (DataInputStream in = new DataInputStream(
                new BufferedInputStream(raw))) {
            int version = in.readInt();
            int type = in.readInt();
            long timestamp = in.readLong();
            int payloadLength = in.readInt();
            byte[] payload = new byte[payloadLength];
            in.readFully(payload); // Guarantees full read or throws
            return new Packet(version, type, timestamp, payload);
        }
    }
}
```

---

## 5-16 Key Points

### Common Mistakes
1. **Not closing streams** — resource leak. Use try-with-resources
2. **No charset specified** — platform-dependent. Always specify UTF-8
3. **Not buffering** — `read()` per byte = 1 syscall per byte. Always wrap
4. **Forgetting flush()** — buffered data lost on crash without flush
5. **Partial reads** — `read(byte[])` may read fewer bytes than array size. Use `readFully()` or loop
6. **Large file into memory** — `readAllBytes()` on huge file → OOM. Use streaming
7. **readLine() without encoding** — deprecated. Use BufferedReader with specific charset
8. **File.exists() before access** — TOCTOU race. Just open and handle FileNotFoundException
9. **Not closing file streams in finally** — use try-with-resources (Java 7+)
10. **File.separator hardcoding** — use `File.separator` or `Path.of()` for cross-platform
11. **Ignoring IOException in close()** — don't suppress in finally blocks
12. **Calling flush() too frequently** — defeats buffering purpose

### NIO vs IO Comparison

| Aspect | java.io | java.nio |
|--------|---------|----------|
| Model | Stream (byte/char) | Buffer + Channel |
| Blocking | Always blocking | Selectable: blocking/non-blocking |
| Direction | Unidirectional | Bidirectional (channels) |
| Buffering | Wrapped internally | Explicit ByteBuffer |
| Concurrency | Thread per stream | Selector, single thread |
| Performance | Moderate | Higher (especially NIO.2) |
| Complexity | Low | Higher |
| File operations | java.io.File | java.nio.file.Path/Files |

### Cheat Sheet

```
═══ JAVA I/O ═════════════════════════════════════════════════

┌─ STANDARD PATTERNS ────────────────────────────────────────┐
│ Read file:      Files.lines(path) / readString(path)        │
│ Write file:     Files.writeString(path, content)            │
│ Copy file:      Files.copy(source, target)                  │
│ Buffered read:  Files.newBufferedReader(path, UTF_8)        │
│ Buffered write: Files.newBufferedWriter(path, UTF_8)        │
└─────────────────────────────────────────────────────────────┘

┌─ RULES ────────────────────────────────────────────────────┐
│ • Always specify charset (UTF-8)                            │
│ • Always use try-with-resources                             │
│ • Always buffer: BufferedInputStream/Reader                 │
│ • Use NIO.2 Files API over java.io.File                    │
│ • For large files, use Stream<String> not readAllLines()    │
│ • For binary, use DataInputStream/DataOutputStream          │
│ • FileChannel.transferTo() for zero-copy file transfers     │
└─────────────────────────────────────────────────────────────┘
```
