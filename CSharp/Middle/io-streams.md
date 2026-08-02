---
tags: [csharp, io, streams, middle, file, async, encoding, pipelines]
level: Middle
date: 2026-08-02
---

# IO и Streams — работа с файлами и потоками

> **Универсальная абстракция для byte sequences: файлы, network, memory, pipes.** `Stream` hierarchy, `FileStream`, `MemoryStream`, `StreamReader`/`StreamWriter`, async I/O, `System.IO.Pipelines` для high-perf. Закрывает пробел: «знаю про `File.ReadAllText`, не понимаю когда `FileStream` и почему `await using` важен для files».

---

## 0. Как читать

Если впервые работаешь с files — раздел 1→3. Если уже пишешь streams, но непонятно про buffering — раздел 5. Async I/O — раздел 7. Hot path — раздел 9 (Pipelines). Production — раздел 11 (best practices), 13 (pitfalls).

Cross-language якоря свёрнуты. Interview-вопросы встроены.

---

## 1. Что это, зачем и когда

### 1.1. Stream — abstract base

`Stream` — abstract класс, представляет **последовательность bytes**. Источник может быть file, network, memory, pipe, encrypted, compressed:

```csharp
public abstract class Stream : IDisposable, IAsyncDisposable
{
    public abstract bool CanRead { get; }
    public abstract bool CanWrite { get; }
    public abstract bool CanSeek { get; }
    public abstract long Length { get; }
    public abstract long Position { get; set; }
    
    public abstract int Read(byte[] buffer, int offset, int count);
    public abstract void Write(byte[] buffer, int offset, int count);
    public abstract void Flush();
    public abstract long Seek(long offset, SeekOrigin origin);
    public abstract void SetLength(long value);
}
```

### 1.2. Stream hierarchy

```
Stream (abstract)
├── FileStream         — file system
├── MemoryStream       — in-memory byte[]
├── NetworkStream      — TCP socket
├── BufferedStream     — buffer wrapper
├── CryptoStream       — encryption layer
├── GZipStream         — compression
├── DeflateStream
├── PipeStream         — named pipes
├── UnmanagedMemoryStream
└── ...
```

Composable — `BufferedStream(GZipStream(FileStream))` — buffered gzip над файлом.

### 1.3. Stream vs higher-level APIs

| Уровень | API | Когда |
|---------|-----|-------|
| **Highest** | `File.ReadAllText`, `File.WriteAllBytes` | Простой случай, файл целиком |
| **High** | `StreamReader.ReadLine`, `StreamWriter.WriteLine` | Текст по строкам |
| **Mid** | `BinaryReader`, `BinaryWriter` | Структурированные bytes |
| **Low** | `Stream.Read/Write` | Custom processing |
| **Lowest** | `System.IO.Pipelines` | High-perf, zero-allocation |

Выбирай **самый высокий** который покрывает потребность. Lower — больше control + сложнее.

### 1.4. Главное правило

```
File ≤ 100 MB, нужен целиком в памяти → File.ReadAllText / ReadAllBytes
Текст по строкам → StreamReader.ReadLine
Большой файл — streaming → Stream.Read в loop
Network / async / cancellation → ReadAsync / WriteAsync
Performance hot path → System.IO.Pipelines
Compression → GZipStream / DeflateStream / BrotliStream
Crypto → CryptoStream

Всегда async для I/O — UI threads, scalability backend.
Всегда await using для streams.
```

### 1.5. Эволюция

| Версия | Что |
|--------|-----|
| **.NET 1.0** | `Stream`, `FileStream`, `MemoryStream`, sync I/O |
| **.NET 4.5** | `ReadAsync`/`WriteAsync` (TAP) |
| **.NET Core 2.1** | `Stream.Read(Span<byte>)` — Span overloads |
| **.NET Core 2.1** | `System.IO.Pipelines` — zero-allocation (2018, вместе с Kestrel rewrite) |
| **.NET Core 3.0** | `IAsyncDisposable`, `await using` |
| **.NET 5+** | `RandomAccess` API для random reads |
| **.NET 6+** | `File.ReadLinesAsync`, `IAsyncEnumerable<string>` |
| **.NET 8+** | `Memory<T>` improvements, perf gains |

> [!info]- Если ты знаешь Java / Python / Go / Rust
> **Java:** `InputStream`/`OutputStream` ↔ `Stream`. `BufferedReader` ↔ `StreamReader`. `Files.readAllLines` ↔ `File.ReadAllLines`. NIO Channels ↔ Pipelines.
>
> **Python:** `open()` returns file object — combines Stream + Reader. `with open(f) as fp:` ↔ `await using`. `asyncio.open_file` ↔ async I/O.
>
> **Go:** `io.Reader` / `io.Writer` interfaces — light-weight, composable. Похожий design philosophy.
>
> **Rust:** `Read`/`Write` traits. `tokio::fs::File` ↔ async file I/O. Очень похожая abstraction.

> [!question]- Интервью: что такое `Stream` в C#?
> Abstract base class для **последовательностей bytes**. Источник: file, network socket, memory, pipe, encrypted, compressed. API: `Read(buffer)`, `Write(buffer)`, `Seek`, `Flush`, `Length`, `Position`. Все streams реализуют `IDisposable` (sync) и `IAsyncDisposable` (async, .NET Core 3+). Composable — `GZipStream(FileStream)` — wrappers создают processing pipeline. Higher-level wrappers: `StreamReader/Writer` для text, `BinaryReader/Writer` для structured. Best practice: всегда `await using` + async ReadAsync/WriteAsync для I/O.

---

## 2. File operations — простые случаи

### 2.1. File.ReadAllText / WriteAllText

```csharp
// Читать весь файл как string
string content = File.ReadAllText("data.txt");
string content2 = File.ReadAllText("data.txt", Encoding.UTF8);

// Async (.NET Core 2.1+)
string content3 = await File.ReadAllTextAsync("data.txt");

// Запись
File.WriteAllText("out.txt", "Hello");
await File.WriteAllTextAsync("out.txt", "Hello", Encoding.UTF8);
```

`Read/WriteAllText` — entire file в memory. **Только для small files** (< 100 MB обычно).

### 2.2. File.ReadAllLines / ReadLines

```csharp
// Все строки в массиве (в памяти)
string[] lines = File.ReadAllLines("data.txt");

// Lazy iterator — line by line
foreach (var line in File.ReadLines("data.txt"))
{
    Process(line);
}

// Async streaming (.NET 6+)
await foreach (var line in File.ReadLinesAsync("data.txt"))
{
    Process(line);
}
```

`ReadLines` — lazy, не загружает весь файл. Для большие файлы.

### 2.3. File.ReadAllBytes / WriteAllBytes

```csharp
byte[] data = File.ReadAllBytes("image.png");
File.WriteAllBytes("copy.png", data);

await File.ReadAllBytesAsync("image.png");
```

Бинарные файлы, малые (image, doc).

### 2.4. File.Copy / Move / Delete

```csharp
File.Copy("source.txt", "dest.txt", overwrite: false);
File.Move("old.txt", "new.txt");
File.Delete("temp.txt");
File.Exists("data.txt");   // bool
```

### 2.5. File.AppendAllText

```csharp
File.AppendAllText("log.txt", $"{DateTime.UtcNow:O}: Event\n");
await File.AppendAllTextAsync("log.txt", "...");
```

Atomic append — открывает, пишет, закрывает.

### 2.6. Directory operations

```csharp
Directory.CreateDirectory("/path/to/dir");
Directory.Exists("/path");
Directory.Delete("/path", recursive: true);

string[] files = Directory.GetFiles("/path", "*.txt", SearchOption.AllDirectories);
foreach (var f in Directory.EnumerateFiles("/path"))
    Console.WriteLine(f);
```

`EnumerateFiles` — lazy, для больших directories.

### 2.7. Path operations

```csharp
string combined = Path.Combine("/path", "to", "file.txt");
string fileName = Path.GetFileName("/path/to/file.txt");        // "file.txt"
string ext = Path.GetExtension("/path/file.txt");                // ".txt"
string dir = Path.GetDirectoryName("/path/file.txt");            // "/path"
string nameNoExt = Path.GetFileNameWithoutExtension("file.txt");  // "file"

string temp = Path.GetTempFileName();   // OS temp file
string tempDir = Path.GetTempPath();
```

`Path.Combine` — cross-platform separator. Не используй ручную concat с `\` или `/`.

> [!question]- Интервью: когда `File.ReadAllText` vs `File.ReadLines`?
> **`ReadAllText`** — entire file → `string` в memory. OK для small files (< 100 MB). Простой одной операцией. **`ReadLines`** — `IEnumerable<string>` lazy iterator, читает line-by-line с buffering. Для large files (> 100 MB) — экономия memory. `ReadAllLines` — eager array (читает всё). `.NET 6+` добавил `ReadLinesAsync` для `await foreach`. Best practice: small файл — `ReadAllText`/`ReadAllLines`, large/streaming — `ReadLines`/`ReadLinesAsync`.

---

## 3. FileStream

### 3.1. Открытие

```csharp
// Read
using var stream = File.OpenRead("data.txt");
using var stream2 = new FileStream("data.txt", FileMode.Open, FileAccess.Read);

// Write (создаёт или overwrites)
using var stream3 = File.OpenWrite("out.txt");
using var stream4 = new FileStream("out.txt", FileMode.Create, FileAccess.Write);

// Append
using var stream5 = new FileStream("log.txt", FileMode.Append, FileAccess.Write);

// Read+Write
using var stream6 = new FileStream("data.txt", FileMode.OpenOrCreate, FileAccess.ReadWrite);

// С options (FileShare, FileOptions, buffer size)
using var stream7 = new FileStream(
    "data.txt",
    FileMode.Open,
    FileAccess.Read,
    FileShare.Read,
    bufferSize: 4096,
    options: FileOptions.SequentialScan | FileOptions.Asynchronous);
```

### 3.2. FileMode

| Mode | Что |
|------|-----|
| `CreateNew` | Создать (throws если exists) |
| `Create` | Создать или overwrite |
| `Open` | Открыть (throws если no file) |
| `OpenOrCreate` | Открыть или создать |
| `Truncate` | Открыть, обнулить content |
| `Append` | Открыть для append, position = end |

### 3.3. Reading bytes

```csharp
using var fs = File.OpenRead("data.bin");

byte[] buffer = new byte[4096];
int bytesRead;
while ((bytesRead = fs.Read(buffer, 0, buffer.Length)) > 0)
{
    Process(buffer, bytesRead);
}

// .NET Core 2.1+ Span overload
Span<byte> span = stackalloc byte[4096];
while ((bytesRead = fs.Read(span)) > 0)
{
    Process(span[..bytesRead]);
}
```

### 3.4. Writing bytes

```csharp
using var fs = File.Create("out.bin");
byte[] data = [1, 2, 3, 4, 5];
fs.Write(data, 0, data.Length);

// Span overload
ReadOnlySpan<byte> span = data;
fs.Write(span);

await fs.FlushAsync();   // ensure flushed to disk before close
```

### 3.5. Seeking

```csharp
using var fs = File.OpenRead("data.bin");

fs.Seek(100, SeekOrigin.Begin);    // absolute position
fs.Seek(50, SeekOrigin.Current);    // relative forward
fs.Seek(-10, SeekOrigin.End);       // от конца назад

fs.Position = 0;   // эквивалент Seek(0, Begin)
```

`CanSeek` — не все streams поддерживают (NetworkStream — no).

### 3.6. FileShare — concurrent access

```csharp
// Other processes могут читать одновременно
using var fs = new FileStream(
    "data.txt", FileMode.Open, FileAccess.Read, FileShare.Read);

// Никто не может open while открыт
using var fs2 = new FileStream(
    "data.txt", FileMode.Open, FileAccess.ReadWrite, FileShare.None);
```

### 3.7. FileOptions

```csharp
FileOptions.Asynchronous       // optimize для async I/O
FileOptions.SequentialScan      // hint OS — sequential reads
FileOptions.RandomAccess        // hint OS — random reads
FileOptions.WriteThrough        // writes go straight to disk
FileOptions.DeleteOnClose       // delete file при close
FileOptions.Encrypted            // EFS encryption
```

> [!question]- Интервью: что такое `FileShare`?
> Параметр в FileStream constructor — определяет, как **другие processes/threads** могут одновременно открыть тот же файл. `FileShare.None` — exclusive lock, никто не может открыть. `FileShare.Read` — другие могут читать одновременно. `FileShare.ReadWrite` — другие могут читать и писать. `FileShare.Delete` — позволяет delete пока открыт. Default `FileShare.Read`. Для logging — `FileShare.ReadWrite` нужен (multi-process write). Pitfall: file locked другим process → IOException.

---

## 4. StreamReader и StreamWriter

### 4.1. StreamReader — text reading

```csharp
using var reader = new StreamReader("data.txt");
// Default Encoding.UTF8

// Чтение полностью
string content = reader.ReadToEnd();

// Line-by-line
string? line;
while ((line = reader.ReadLine()) != null)
{
    Process(line);
}

// Char-by-char (редко)
int ch;
while ((ch = reader.Read()) != -1)
{
    Process((char)ch);
}
```

### 4.2. StreamWriter — text writing

```csharp
using var writer = new StreamWriter("out.txt");
// Overwrite по default — параметр append для add

writer.WriteLine("First line");
writer.WriteLine($"Number: {42}");
writer.Write("No newline");

await writer.FlushAsync();
```

### 4.3. Encoding

```csharp
// Default — UTF-8 без BOM (.NET 4.5+)
using var r = new StreamReader("data.txt", Encoding.UTF8);
using var r2 = new StreamReader("data.txt", new UTF8Encoding(encoderShouldEmitUTF8Identifier: false));

// Auto-detect (от BOM)
using var r3 = new StreamReader("data.txt", detectEncodingFromByteOrderMarks: true);
```

См. [[strings-regex]] раздел 7.

### 4.4. Wrap existing Stream

```csharp
using var fs = File.OpenRead("data.txt");
using var reader = new StreamReader(fs);
// reader.Dispose() закрывает fs тоже

// Чтобы НЕ закрывать base stream
using var reader2 = new StreamReader(fs, Encoding.UTF8, detectEncodingFromByteOrderMarks: false, bufferSize: 1024, leaveOpen: true);
```

`leaveOpen: true` — caller сохраняет ownership base stream.

### 4.5. Async API

```csharp
using var reader = new StreamReader("data.txt");

string content = await reader.ReadToEndAsync();

string? line;
while ((line = await reader.ReadLineAsync()) != null)
{
    Process(line);
}

// IAsyncEnumerable<string> (через extension methods libraries)
```

### 4.6. StringReader / StringWriter — in-memory

```csharp
// Read string как stream
using var reader = new StringReader("line 1\nline 2\nline 3");
while ((line = reader.ReadLine()) != null)
    Console.WriteLine(line);

// Build string через StreamWriter API
using var writer = new StringWriter();
writer.WriteLine("Hello");
string result = writer.ToString();   // "Hello\n"
```

Для unit tests, in-memory transformation.

> [!question]- Интервью: чем `StreamReader` отличается от `BinaryReader`?
> **`StreamReader`** — для **text**: использует Encoding (UTF-8 default), parses chars/lines, методы `Read`, `ReadLine`, `ReadToEnd`. **`BinaryReader`** — для **structured binary**: read primitives (`ReadInt32`, `ReadDouble`, `ReadString` с length-prefix), no encoding magic для primitives. StreamReader для text files (CSV, log, config). BinaryReader для custom binary protocols, file formats. Не путай: оба wrap `Stream`.

---

## 5. MemoryStream

### 5.1. In-memory byte[] как stream

```csharp
// Empty MemoryStream
using var ms = new MemoryStream();
ms.Write(data, 0, data.Length);
ms.Position = 0;   // reset для reading
byte[] buffer = new byte[100];
int read = ms.Read(buffer, 0, buffer.Length);

// Wrap existing byte[]
using var ms2 = new MemoryStream(existingBytes, writable: false);

// С capacity (initial)
using var ms3 = new MemoryStream(capacity: 1024);
```

### 5.2. Use cases

```csharp
// Convert string ↔ stream
string s = "hello";
byte[] bytes = Encoding.UTF8.GetBytes(s);
using var ms = new MemoryStream(bytes);
// ... pass to API expecting Stream

// Build response in memory
using var output = new MemoryStream();
JsonSerializer.Serialize(output, data);
output.Position = 0;
return output.ToArray();   // или output.ToArray() для byte[]
```

### 5.3. ToArray vs GetBuffer

```csharp
using var ms = new MemoryStream();
// ... write data

byte[] copy = ms.ToArray();          // copy size = Length (вычистает unused space)
byte[] buffer = ms.GetBuffer();       // reference на internal buffer (length = Capacity!)
// buffer.Length может быть > ms.Length
```

`ToArray()` — copy. `GetBuffer()` — internal access, faster но careful.

### 5.4. Pre-allocated MemoryStream

```csharp
// Reuse pattern для perf
using var ms = new MemoryStream(capacity: 4096);

for (int i = 0; i < 1000; i++)
{
    ms.SetLength(0);   // clear
    ms.Position = 0;
    
    JsonSerializer.Serialize(ms, items[i]);
    Process(ms.GetBuffer().AsSpan(0, (int)ms.Length));
}
```

> [!question]- Интервью: чем `MemoryStream.ToArray()` отличается от `GetBuffer()`?
> **`ToArray()`** — создаёт **новый** byte[] длиной `Length`, копирует data. Safe — buffer независимый. **`GetBuffer()`** — возвращает **internal buffer** (length = `Capacity`, не `Length`!). Faster — no copy. Caveat: buffer может содержать unused space после `Length`. Лимит: `GetBuffer()` available только если `MemoryStream` создан без external byte[]. Для perf — `GetBuffer()` + Span с `[..Length]`. Для safety / API contracts — `ToArray()`.

---

## 6. BinaryReader / BinaryWriter

### 6.1. Структурированные данные

```csharp
// Запись
using var fs = File.Create("data.bin");
using var writer = new BinaryWriter(fs);

writer.Write(42);                    // int (4 bytes)
writer.Write(3.14);                   // double (8 bytes)
writer.Write("hello");                // length-prefixed string (LEB128 length + UTF-8 bytes)
writer.Write(true);                   // bool (1 byte)
writer.Write(new byte[] { 1, 2, 3 }); // raw bytes

// Чтение
using var fs2 = File.OpenRead("data.bin");
using var reader = new BinaryReader(fs2);

int i = reader.ReadInt32();
double d = reader.ReadDouble();
string s = reader.ReadString();
bool b = reader.ReadBoolean();
byte[] bytes = reader.ReadBytes(3);
```

### 6.2. Endianness

`BinaryReader/Writer` — **little-endian** по default (x86/x64). Для cross-platform / network protocol — `BinaryPrimitives` с explicit endianness:

```csharp
using System.Buffers.Binary;

byte[] buffer = new byte[4];
BinaryPrimitives.WriteInt32BigEndian(buffer, 42);
BinaryPrimitives.WriteInt32LittleEndian(buffer, 42);

int v1 = BinaryPrimitives.ReadInt32BigEndian(buffer);
int v2 = BinaryPrimitives.ReadInt32LittleEndian(buffer);
```

### 6.3. Custom protocol

```csharp
public record Message(int Id, string Text, double Timestamp);

public static void WriteMessage(BinaryWriter w, Message m)
{
    w.Write(m.Id);
    w.Write(m.Text);
    w.Write(m.Timestamp);
}

public static Message ReadMessage(BinaryReader r) =>
    new(r.ReadInt32(), r.ReadString(), r.ReadDouble());
```

> [!question]- Интервью: какой endianness использует `BinaryWriter`?
> **Little-endian** по default (matches x86/x64 architecture). Для cross-platform / network protocols / file formats где требуется specific endianness — используй `System.Buffers.Binary.BinaryPrimitives` (`WriteInt32BigEndian`, `ReadUInt64LittleEndian` etc.). Эти methods explicit, no platform dependence. `BinaryWriter` хорош для **own** binary format где обе стороны .NET.

---

## 7. Async I/O

### 7.1. ReadAsync / WriteAsync

```csharp
using var fs = File.OpenRead("data.bin");
byte[] buffer = new byte[4096];
int bytesRead = await fs.ReadAsync(buffer, 0, buffer.Length);

// Span overload (.NET Core 2.1+)
Memory<byte> mem = buffer.AsMemory();
int bytesRead2 = await fs.ReadAsync(mem);

// CopyToAsync
using var output = File.Create("copy.bin");
await fs.CopyToAsync(output);
```

### 7.2. Зачем async I/O

- **Не блокирует thread** — другие requests процессятся.
- **Scalability** — server handles 1000s of concurrent connections.
- **UI responsiveness** — main thread не frozen.

### 7.3. Cancellation

```csharp
public async Task ProcessAsync(string path, CancellationToken ct)
{
    using var fs = File.OpenRead(path);
    byte[] buffer = new byte[4096];
    int bytesRead;
    
    while ((bytesRead = await fs.ReadAsync(buffer, ct)) > 0)
    {
        ct.ThrowIfCancellationRequested();
        await ProcessChunk(buffer, bytesRead, ct);
    }
}

// Caller
using var cts = new CancellationTokenSource(TimeSpan.FromSeconds(30));
await ProcessAsync("data.bin", cts.Token);
```

### 7.4. await using для async dispose

```csharp
public async Task WriteFile(string path, string content)
{
    await using var fs = File.Create(path);
    await using var writer = new StreamWriter(fs);
    
    await writer.WriteAsync(content);
    // await writer.DisposeAsync() в finally
    // await fs.DisposeAsync() в finally
}
```

См. [[dispose-pattern]] раздел 9.

### 7.5. CopyToAsync с cancellation

```csharp
using var input = File.OpenRead("source");
using var output = File.Create("dest");

await input.CopyToAsync(output, bufferSize: 81920, ct);
```

### 7.6. ConfigureAwait(false) в library

```csharp
public async Task<string> ReadConfigAsync(string path)
{
    using var reader = new StreamReader(path);
    return await reader.ReadToEndAsync().ConfigureAwait(false);
}
```

В library — `ConfigureAwait(false)` чтобы avoid sync context. В app code обычно не нужно.

> [!question]- Интервью: зачем async I/O если есть sync?
> **Не блокирует thread**. Sync I/O — thread waits на disk/network. В web server — каждый request занимает thread. С 1000 concurrent requests — thread pool exhausted, server unable to accept new connections. Async I/O — thread returns в pool во время wait, OS notifies completion → callback runs. Один thread handles 1000s of operations. Также: UI apps — main thread не freezes. Best practice: всегда `ReadAsync`/`WriteAsync` с `await using` + CancellationToken. Sync I/O оправдана только в console scripts с одной operation.

---

## 8. Compression и Crypto

### 8.1. GZipStream — gzip compression

```csharp
// Compress
using var input = File.OpenRead("data.txt");
using var output = File.Create("data.txt.gz");
using var gzip = new GZipStream(output, CompressionLevel.Optimal);
await input.CopyToAsync(gzip);

// Decompress
using var compressed = File.OpenRead("data.txt.gz");
using var decompressed = File.Create("data.txt");
using var gzip2 = new GZipStream(compressed, CompressionMode.Decompress);
await gzip2.CopyToAsync(decompressed);
```

### 8.2. DeflateStream / BrotliStream

```csharp
// DeflateStream — без gzip header (sometimes used in protocols)
using var deflate = new DeflateStream(output, CompressionLevel.Optimal);

// BrotliStream — better compression, used в HTTP (br encoding)
using var br = new BrotliStream(output, CompressionLevel.Optimal);
```

### 8.3. CryptoStream — encryption

```csharp
using var aes = Aes.Create();
aes.Key = key;
aes.IV = iv;

// Encrypt
using var output = File.Create("encrypted.bin");
using var crypto = new CryptoStream(output, aes.CreateEncryptor(), CryptoStreamMode.Write);
crypto.Write(plainData);

// Decrypt
using var input = File.OpenRead("encrypted.bin");
using var crypto2 = new CryptoStream(input, aes.CreateDecryptor(), CryptoStreamMode.Read);
crypto2.CopyTo(output);
```

### 8.4. Composing streams

```csharp
// Encrypt + compress + write to file
using var fs = File.Create("output.enc.gz");
using var crypto = new CryptoStream(fs, aes.CreateEncryptor(), CryptoStreamMode.Write);
using var gzip = new GZipStream(crypto, CompressionLevel.Optimal);
using var writer = new StreamWriter(gzip);

await writer.WriteAsync(content);
```

LIFO disposal: writer → gzip → crypto → fs. Все flush в правильном order.

> [!question]- Интервью: как combine compression + encryption + file write?
> Wrapping streams в pipeline: `FileStream → CryptoStream → GZipStream → StreamWriter`. Writer пишет text в gzip stream → gzip compresses → crypto encrypts → file stores. Disposal LIFO order: писатель flush чтобы compress finalize, gzip flush для encrypt, crypto flush для file write, file flush to disk. `using var` automatic LIFO. Read pipeline reverse: `FileStream → CryptoStream(decrypt) → GZipStream(decompress) → StreamReader`. Important: правильный order или output corrupt.

---

## 9. System.IO.Pipelines (.NET Core 2.1+)

### 9.1. Что такое

`Pipelines` — high-perf, **zero-allocation** API для streaming data между producer и consumer. Замена `Stream.Read` + `Stream.Write` для performance-critical scenarios (HTTP servers, parsers).

```csharp
public async Task ParseAsync(PipeReader reader)
{
    while (true)
    {
        ReadResult result = await reader.ReadAsync();
        ReadOnlySequence<byte> buffer = result.Buffer;
        
        // Process buffer (find newlines, parse messages)
        SequencePosition? position;
        do
        {
            position = buffer.PositionOf((byte)'\n');
            if (position != null)
            {
                ProcessLine(buffer.Slice(0, position.Value));
                buffer = buffer.Slice(buffer.GetPosition(1, position.Value));
            }
        } while (position != null);
        
        reader.AdvanceTo(buffer.Start, buffer.End);
        
        if (result.IsCompleted) break;
    }
    
    await reader.CompleteAsync();
}
```

### 9.2. Преимущества

- **Zero allocation** для buffers — internal pool.
- **Backpressure** — automatic flow control.
- **Cancel-friendly** — built-in.
- **Multi-segment buffers** — `ReadOnlySequence<byte>`.

### 9.3. Когда использовать

✅ **Используй когда:**
- High-perf parser (HTTP, MQTT, custom protocols).
- Servers handling thousands of concurrent connections.
- Network streaming.

❌ **Не используй когда:**
- Простая file I/O — overkill.
- Single-shot read — `Stream.Read` enough.
- Learning curve не оправдывает.

### 9.4. Используется в Kestrel

ASP.NET Core HTTP server (Kestrel) использует Pipelines под капотом для request parsing. Для consumer code — обычно `Stream` API через abstractions.

> [!question]- Интервью: что такое `System.IO.Pipelines`?
> High-perf zero-allocation API для streaming bytes между producer и consumer. Замена `Stream.Read/Write` для performance-critical scenarios — HTTP parsers, network protocols, large file processing. Преимущества: 1) zero allocation через internal pool. 2) automatic backpressure. 3) `ReadOnlySequence<byte>` multi-segment buffers (без линейной copy). 4) cancellation. Используется в Kestrel (ASP.NET Core). Steep learning curve — не для простых case'ов. Pattern: `PipeReader.ReadAsync` → process buffer → `AdvanceTo(consumed, examined)` → repeat.

---

## 10. RandomAccess (.NET 6+)

### 10.1. Random reads без FileStream state

```csharp
using SafeFileHandle handle = File.OpenHandle("data.bin", FileMode.Open, FileAccess.Read);

byte[] buffer = new byte[1024];

// Read at specific position
int read = RandomAccess.Read(handle, buffer, fileOffset: 1000);

// Async
int read2 = await RandomAccess.ReadAsync(handle, buffer, fileOffset: 1000);

// Length
long len = RandomAccess.GetLength(handle);
```

### 10.2. Multiple positions concurrent

```csharp
// Можно читать с разных positions одновременно (safer than Stream.Position)
var task1 = RandomAccess.ReadAsync(handle, buffer1, 0);
var task2 = RandomAccess.ReadAsync(handle, buffer2, 1024);
await Task.WhenAll(task1, task2);
```

`FileStream.Position` — shared state, не thread-safe. `RandomAccess` — atomic per-call offset.

### 10.3. Когда

Database engines, disk-backed caches, файлы с random-access patterns. Для linear streaming — обычный `FileStream` enough.

---

## 11. Best Practices

### 11.1. Файлы

- ✅ **`File.ReadAllText`** для маленьких файлов.
- ✅ **`File.ReadLines`** для построчного reading больших файлов.
- ✅ **`File.ReadLinesAsync`** (.NET 6+) для async streaming.
- ✅ **`Path.Combine`** для cross-platform paths.
- ❌ **Manual concatenation** path с `\` или `/`.
- ❌ **`ReadAllText` для big files** (> 100 MB).

### 11.2. Streams

- ✅ **`await using`** для async streams.
- ✅ **`Encoding.UTF8`** explicit при text I/O.
- ✅ **`leaveOpen: true`** для adapters/decorators.
- ✅ **`FileShare.Read`** для shared files.
- ✅ **`FlushAsync()`** перед close для async writers.
- ❌ **Sync I/O в async methods**.
- ❌ **`Read`/`Write` без cancellation token** в long operations.

### 11.3. Performance

- ✅ **Pre-allocated buffers** для reuse.
- ✅ **`Span<byte>` / `Memory<byte>`** для no-allocation.
- ✅ **`FileOptions.SequentialScan`** для sequential reads.
- ✅ **`FileOptions.Asynchronous`** для async optimization.
- ✅ **`System.IO.Pipelines`** для high-perf parsers.
- ❌ **`File.ReadAllBytes` для GB files** — OOM risk.

### 11.4. Encoding

- ✅ **UTF-8 без BOM** для cross-platform.
- ✅ **`UTF8Encoding(emitUtf8Identifier: false)`** для no BOM.
- ❌ **Default Encoding для text files** — может быть platform-dependent.

### 11.5. Не делай

- ❌ Открывать stream без `using`.
- ❌ Multiple iteration `IEnumerable<string>` от ReadLines (re-reads file).
- ❌ Wide `FileShare.None` для logs.
- ❌ Sync `Wait()` на async stream operations (deadlock).

---

## 12. Decision tree

```
Что нужно?
│
├── Small файл целиком в строку → File.ReadAllText
├── Small файл по строкам → File.ReadAllLines / ReadLines
├── Big файл streaming → File.ReadLines / ReadLinesAsync
├── Bytes целиком → File.ReadAllBytes
├── Custom processing → FileStream + buffer loop
├── Text wrap stream → StreamReader / StreamWriter
├── Binary structured → BinaryReader / BinaryWriter (own format)
├── Cross-platform binary → BinaryPrimitives explicit endianness
├── In-memory bytes как stream → MemoryStream
├── Compression → GZipStream / BrotliStream
├── Encryption → CryptoStream
├── Random reads → RandomAccess (.NET 6+)
├── High-perf parser → System.IO.Pipelines
└── Async + cancellation → ReadAsync + CancellationToken
```

---

## 13. Cheat sheet

```csharp
// === Simple reading ===
string text = await File.ReadAllTextAsync("data.txt");
string[] lines = await File.ReadAllLinesAsync("data.txt");
byte[] bytes = await File.ReadAllBytesAsync("image.png");

// === Streaming ===
await foreach (var line in File.ReadLinesAsync("big.log"))
    Process(line);

// === FileStream ===
await using var fs = File.OpenRead("data.bin");
byte[] buffer = new byte[4096];
int read;
while ((read = await fs.ReadAsync(buffer, ct)) > 0)
    Process(buffer, read);

// === StreamReader/Writer ===
await using var reader = new StreamReader("data.txt", Encoding.UTF8);
string content = await reader.ReadToEndAsync();

await using var writer = new StreamWriter("out.txt");
await writer.WriteLineAsync("line");

// === MemoryStream ===
await using var ms = new MemoryStream();
await JsonSerializer.SerializeAsync(ms, data);
byte[] result = ms.ToArray();

// === BinaryReader/Writer ===
await using var bw = new BinaryWriter(File.Create("data.bin"));
bw.Write(42);
bw.Write("hello");

// === Compression ===
await using var output = File.Create("out.gz");
await using var gzip = new GZipStream(output, CompressionLevel.Optimal);
await using var writer2 = new StreamWriter(gzip);
await writer2.WriteAsync(content);

// === Path ===
string p = Path.Combine("/path", "to", "file.txt");
string ext = Path.GetExtension(p);
string name = Path.GetFileNameWithoutExtension(p);

// === Directory ===
foreach (var file in Directory.EnumerateFiles("/path", "*.txt"))
    Console.WriteLine(file);
```

---

## 14. Common Pitfalls

### 14.1. Не using

```csharp
var fs = File.OpenRead("data.txt");   // ❌ leak
ProcessFile(fs);
// fs не closed — file lock until GC
```

**Фикс:** `await using var fs = ...`.

### 14.2. ReadAllText для huge file

```csharp
string content = File.ReadAllText("10gb.log");   // ❌ OutOfMemoryException
```

**Фикс:** `ReadLines` или `FileStream` + buffer.

### 14.3. Multi-iter ReadLines

```csharp
var lines = File.ReadLines("data.txt");   // lazy IEnumerable
var count = lines.Count();   // полное чтение #1
foreach (var l in lines)     // полное чтение #2!
{ }
```

**Фикс:** `ReadAllLines` для array (in-memory), или iterate один раз.

### 14.4. Encoding mismatch

```csharp
File.WriteAllText("out.txt", text);                 // UTF-8 на новых .NET
var read = File.ReadAllText("out.txt", Encoding.ASCII);   // ❌ может быть corrupt non-ASCII
```

**Фикс:** explicit `Encoding.UTF8` всегда.

### 14.5. UTF-8 BOM в файлах

```csharp
File.WriteAllText("config.json", json, Encoding.UTF8);
// ⚠ BOM включен — JSON parser может error
```

**Фикс:** `new UTF8Encoding(emitUtf8Identifier: false)`.

### 14.6. FileShare.None для logs

```csharp
using var fs = new FileStream("log.txt", FileMode.Append, FileAccess.Write, FileShare.None);
// Other process не может open — log multi-process broken
```

**Фикс:** `FileShare.ReadWrite`.

### 14.7. Не Flush async writer

```csharp
await using var writer = new StreamWriter(fs);
await writer.WriteAsync(content);
// dispose writer → flushes, но если exception до dispose — data potentially lost
```

**Фикс:** `await writer.FlushAsync()` explicit.

### 14.8. Sync .Result на async

```csharp
var content = File.ReadAllTextAsync("data.txt").Result;   // ❌ deadlock risk
```

**Фикс:** `await File.ReadAllTextAsync(...)`.

### 14.9. Path concatenation manual

```csharp
string p = "/path" + "/" + filename;   // ❌ может быть double-slash, wrong separator
```

**Фикс:** `Path.Combine("/path", filename)`.

### 14.10. MemoryStream.GetBuffer() length confusion

```csharp
var buf = ms.GetBuffer();
ProcessAll(buf);   // ❌ buf.Length = Capacity, может быть > actual data!
```

**Фикс:** `buf.AsSpan(0, (int)ms.Length)` для actual data only.

> [!question]- Интервью: топ-3 ошибки с file I/O?
> 1) **ReadAllText для huge files** — OutOfMemoryException на > 100 MB. Используй `ReadLines` (lazy) или `FileStream` + buffer. 2) **Не using** — file lock until GC. Always `await using var`. 3) **UTF-8 BOM** в config files (json, yaml) — parsers могут error. `new UTF8Encoding(emitUtf8Identifier: false)`.

---

## 15. Practice exercises

### 15.1. Streaming line processor

```csharp
public async Task<int> CountWordsAsync(string path, CancellationToken ct = default)
{
    int total = 0;
    await foreach (var line in File.ReadLinesAsync(path, ct))
    {
        ct.ThrowIfCancellationRequested();
        total += line.Split(' ', StringSplitOptions.RemoveEmptyEntries).Length;
    }
    return total;
}
```

### 15.2. Compress + encrypt pipeline

```csharp
public static async Task EncryptAndCompressAsync(
    string inputPath,
    string outputPath,
    byte[] key,
    byte[] iv)
{
    using var aes = Aes.Create();
    aes.Key = key;
    aes.IV = iv;
    
    await using var input = File.OpenRead(inputPath);
    await using var output = File.Create(outputPath);
    await using var crypto = new CryptoStream(output, aes.CreateEncryptor(), CryptoStreamMode.Write);
    await using var gzip = new GZipStream(crypto, CompressionLevel.Optimal);
    
    await input.CopyToAsync(gzip);
}
```

### 15.3. Custom binary format

```csharp
public record User(int Id, string Name, double Score);

public static async Task WriteUsersAsync(string path, IList<User> users)
{
    await using var fs = File.Create(path);
    await using var bw = new BinaryWriter(fs);
    
    bw.Write(users.Count);
    foreach (var u in users)
    {
        bw.Write(u.Id);
        bw.Write(u.Name);
        bw.Write(u.Score);
    }
}

public static async Task<List<User>> ReadUsersAsync(string path)
{
    await using var fs = File.OpenRead(path);
    using var br = new BinaryReader(fs);
    
    int count = br.ReadInt32();
    var users = new List<User>(count);
    for (int i = 0; i < count; i++)
        users.Add(new User(br.ReadInt32(), br.ReadString(), br.ReadDouble()));
    
    return users;
}
```

---

## 16. Что читать дальше

1. **[[dispose-pattern|Dispose Pattern]]** — IAsyncDisposable, await using.
2. **[[strings-regex|Strings]]** — encoding deep.
3. **System.IO.Pipelines deep** — high-perf parsing.
4. **`HttpClient` streaming** — content as stream.
5. **EF Core file storage** — large content patterns.

---

## 17. См. также

- [[dispose-pattern|Dispose Pattern]] — `await using`
- [[strings-regex|Strings]] — encoding
- [[error-handling|Error Handling]] — IOException
- `System.IO.Pipelines`
- `Memory<T>` / `Span<T>` для buffers
- `RandomAccess` API

---

## 18. Reading list

- **Microsoft Docs — File and stream I/O** — learn.microsoft.com/dotnet/standard/io/
- **Microsoft Docs — Stream class** — learn.microsoft.com/dotnet/api/system.io.stream
- **Microsoft Docs — System.IO.Pipelines** — learn.microsoft.com/dotnet/standard/io/pipelines
- **Microsoft Docs — RandomAccess** — learn.microsoft.com/dotnet/api/system.io.randomaccess
- **David Fowler — Pipelines deep dive** — devblogs.microsoft.com
- **Stephen Cleary — Async I/O patterns** — blog.stephencleary.com
- **Adam Sitnik — File I/O performance** — adamsitnik.com
- **Andrew Lock — Streaming patterns** — andrewlock.net
