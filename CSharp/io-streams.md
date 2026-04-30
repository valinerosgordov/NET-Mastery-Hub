---
tags: [csharp, io, streams, files, async, middle]
level: Middle
date: 2026-04-30
---

# I/O и Streams — работа с файлами и потоками

> **Практический гайд по работе с файлами, потоками, async I/O в C#**. Каждый разработчик читает/пишет файлы. Закрывает: File API, Stream иерархия, StreamReader/Writer, BinaryReader/Writer, async I/O, encoding, paths, System.IO.Pipelines для performance.

---

## Что это, зачем и когда

### Что такое I/O

**Input/Output** — операции с внешним миром:
- Чтение / запись файлов
- Network communications (HTTP, sockets)
- Pipes между процессами
- Console input
- БД connections
- Memory streams

**Все эти операции** в .NET абстрагированы через **`Stream`** — единый интерфейс для последовательного чтения / записи bytes.

### Главное правило

> **I/O — медленно. Async I/O — must для production.**

```csharp
// ❌ Sync — блокирует thread на 100ms-1s
File.ReadAllText("big.txt");

// ✅ Async — освобождает thread пока читается
await File.ReadAllTextAsync("big.txt");
```

В web app с sync I/O — thread pool starvation → server падает под нагрузкой.

---

## 1. File API — высокий уровень

Простые операции с файлами через `static class System.IO.File`.

### Read

```csharp
// Read all text
string content = File.ReadAllText("file.txt");
string content = await File.ReadAllTextAsync("file.txt");

// With encoding
string content = File.ReadAllText("file.txt", Encoding.UTF8);

// Read all lines
string[] lines = File.ReadAllLines("file.txt");

// Read line by line (lazy — для больших файлов)
foreach (var line in File.ReadLines("big.txt"))
{
    // обработка по одной строке, не загружает весь файл в память
}

// Async streaming (.NET 6+)
await foreach (var line in File.ReadLinesAsync("big.txt"))
{
    // ...
}

// Read bytes
byte[] data = File.ReadAllBytes("image.jpg");
byte[] data = await File.ReadAllBytesAsync("image.jpg");
```

### Write

```csharp
// Write all text (overwrites!)
File.WriteAllText("output.txt", "Hello");
await File.WriteAllTextAsync("output.txt", "Hello");

// Write lines
string[] lines = { "Line 1", "Line 2", "Line 3" };
File.WriteAllLines("output.txt", lines);
await File.WriteAllLinesAsync("output.txt", lines);

// Append
File.AppendAllText("log.txt", "New log entry\n");
await File.AppendAllTextAsync("log.txt", "New log entry\n");
File.AppendAllLines("log.txt", new[] { "line 1", "line 2" });

// Write bytes
File.WriteAllBytes("data.bin", bytes);
```

### File operations

```csharp
// Existence
File.Exists("file.txt");           // true / false
Directory.Exists("folder");

// Delete
File.Delete("file.txt");           // throws если не exist
if (File.Exists("file.txt")) File.Delete("file.txt");

// Copy
File.Copy("source.txt", "dest.txt");
File.Copy("source.txt", "dest.txt", overwrite: true);

// Move / rename
File.Move("old.txt", "new.txt");
File.Move("old.txt", "new.txt", overwrite: true);  // .NET Core 3+

// Info
FileInfo fi = new FileInfo("file.txt");
fi.Length;                  // size in bytes
fi.CreationTime;
fi.LastWriteTime;
fi.IsReadOnly;
fi.Extension;               // ".txt"
fi.DirectoryName;           // папка
fi.FullName;                // абсолютный путь
```

### Directories

```csharp
// Create
Directory.CreateDirectory("path/to/folder");  // создаёт всю цепочку

// List files
string[] files = Directory.GetFiles("folder");
string[] files = Directory.GetFiles("folder", "*.txt");
string[] files = Directory.GetFiles("folder", "*.txt", SearchOption.AllDirectories);

// Lazy enumeration (для больших папок)
foreach (var file in Directory.EnumerateFiles("folder", "*.txt", SearchOption.AllDirectories))
{
    // ...
}

// List subdirectories
string[] dirs = Directory.GetDirectories("folder");

// Delete
Directory.Delete("folder");                    // throws если не пустая
Directory.Delete("folder", recursive: true);   // delete contents too
```

---

## 2. Path — манипуляции с путями

```csharp
using System.IO;

// Combine — cross-platform safe
Path.Combine("folder", "subfolder", "file.txt");
// Linux: "folder/subfolder/file.txt"
// Windows: "folder\subfolder\file.txt"

// Decompose
Path.GetFileName("/folder/file.txt");           // "file.txt"
Path.GetFileNameWithoutExtension("file.txt");    // "file"
Path.GetExtension("file.txt");                    // ".txt"
Path.GetDirectoryName("/folder/file.txt");        // "/folder"

// Absolute path
Path.GetFullPath("file.txt");                     // current dir + file
Path.IsPathRooted("file.txt");                    // false
Path.IsPathRooted("C:\\file.txt");                // true

// Temp
Path.GetTempPath();                               // OS temp dir
Path.GetTempFileName();                           // unique temp file (создаёт!)
Path.GetRandomFileName();                          // random name (не создаёт)

// Change extension
Path.ChangeExtension("file.txt", ".md");          // "file.md"
```

### Cross-platform pitfalls

```csharp
// ❌ Hard-coded separator
var path = "folder\\subfolder\\file.txt";  // Windows-only

// ❌ Тоже плохо
var path = "folder/subfolder/file.txt";    // на Windows работает но конвенция unix

// ✅ Path.Combine
var path = Path.Combine("folder", "subfolder", "file.txt");

// ✅ Path.DirectorySeparatorChar если нужен char
'/' или '\\' — депенды от ОС
```

---

## 3. Stream — иерархия

`Stream` — abstract base class для последовательного чтения / записи bytes.

```
Stream (abstract)
  ├── FileStream         — файлы
  ├── MemoryStream       — in-memory bytes
  ├── NetworkStream      — TCP/UDP
  ├── BufferedStream     — wrap другой stream с buffer
  ├── GZipStream         — gzip compression
  ├── DeflateStream      — deflate compression
  ├── CryptoStream       — encryption / decryption
  └── (others)
```

Все streams имеют:
- `Read(buffer, offset, count)` / `ReadAsync(...)`
- `Write(buffer, offset, count)` / `WriteAsync(...)`
- `Seek`, `Position`, `Length` (не все streams seekable)
- `Flush` / `FlushAsync`
- `Dispose` / `DisposeAsync`

### FileStream

Низкоуровневый доступ к файлу.

```csharp
using FileStream fs = File.OpenRead("file.txt");

byte[] buffer = new byte[1024];
int bytesRead = fs.Read(buffer, 0, buffer.Length);

// Async
using FileStream fs = File.OpenRead("file.txt");
byte[] buffer = new byte[1024];
int bytesRead = await fs.ReadAsync(buffer);

// Modes
File.OpenRead(path);           // ReadOnly
File.OpenWrite(path);           // Write (creates or overwrites)
File.OpenWrite(path, FileMode.Append);  // Append

// Полный контроль
using var fs = new FileStream(
    "file.txt",
    FileMode.OpenOrCreate,
    FileAccess.ReadWrite,
    FileShare.Read,                 // другие могут читать
    bufferSize: 4096,
    useAsync: true);
```

### MemoryStream — in-memory

```csharp
// Write
using var ms = new MemoryStream();
byte[] data = Encoding.UTF8.GetBytes("Hello");
ms.Write(data, 0, data.Length);
byte[] result = ms.ToArray();  // копия

// Read existing bytes
byte[] input = File.ReadAllBytes("file.bin");
using var ms = new MemoryStream(input);
// ...

// Когда?
// - Имитация file API без реального файла (тесты)
// - Buffer для serialization
// - Когда library требует Stream но у тебя bytes
```

---

## 4. StreamReader / StreamWriter — text

Wrap Stream для удобной работы с **текстом** (handle encoding).

### StreamReader

```csharp
// Шорткат
using StreamReader reader = new("file.txt");
string content = await reader.ReadToEndAsync();

// Line by line
using StreamReader reader = new("file.txt");
string? line;
while ((line = await reader.ReadLineAsync()) != null)
{
    Console.WriteLine(line);
}

// Wrap другой stream
using FileStream fs = File.OpenRead("file.txt");
using StreamReader reader = new(fs, Encoding.UTF8);
string text = reader.ReadToEnd();
```

### StreamWriter

```csharp
using StreamWriter writer = new("output.txt");
await writer.WriteLineAsync("Line 1");
await writer.WriteLineAsync("Line 2");
await writer.WriteAsync("No newline");

// Append
using StreamWriter writer = new("log.txt", append: true);
await writer.WriteLineAsync($"[{DateTime.UtcNow:o}] Event");

// С encoding
using StreamWriter writer = new("output.txt", append: false, Encoding.UTF8);
```

### Encoding — важно

```csharp
// .NET 5+ default — UTF-8 (без BOM)
using StreamWriter w = new("file.txt");

// Explicit
using StreamWriter w = new("file.txt", false, new UTF8Encoding(false));  // no BOM
using StreamWriter w = new("file.txt", false, new UTF8Encoding(true));    // with BOM

// UTF-16
using StreamWriter w = new("file.txt", false, Encoding.Unicode);

// ASCII
using StreamWriter w = new("file.txt", false, Encoding.ASCII);

// Legacy Windows-1251 (Russian) — нужно зарегистрировать provider
Encoding.RegisterProvider(CodePagesEncodingProvider.Instance);
using StreamWriter w = new("file.txt", false, Encoding.GetEncoding(1251));
```

---

## 5. BinaryReader / BinaryWriter — typed binary

Для бинарных файлов с типизированными значениями.

```csharp
// Write
using FileStream fs = File.OpenWrite("data.bin");
using BinaryWriter writer = new(fs);

writer.Write(42);                 // int — 4 bytes
writer.Write(3.14);                // double — 8 bytes
writer.Write("Hello");             // string — length-prefixed
writer.Write(true);                // bool — 1 byte
writer.Write(new byte[] { 1, 2, 3 }); // raw bytes

// Read
using FileStream fs = File.OpenRead("data.bin");
using BinaryReader reader = new(fs);

int i = reader.ReadInt32();
double d = reader.ReadDouble();
string s = reader.ReadString();
bool b = reader.ReadBoolean();
byte[] bytes = reader.ReadBytes(3);
```

> [!warning] BinaryWriter — endianness
> Записывает в little-endian. При обмене с другими системами проверяй.

---

## 6. Async I/O — best practice

### Sync vs Async

```csharp
// ❌ Sync — блокирует thread
public string LoadConfig()
{
    return File.ReadAllText("config.json");  // блокирует thread на N мс
}

// ✅ Async — thread свободен пока ждёт I/O
public async Task<string> LoadConfigAsync()
{
    return await File.ReadAllTextAsync("config.json");
}
```

### Почему async важен в web

```
Web сервер — thread pool с N threads (например, 32)
Если 32 запроса делают sync I/O — все 32 threads ждут
33-й запрос — ждёт пока thread освободится
→ thread pool starvation → запросы timeout

С async — те же 32 threads обрабатывают сотни одновременных I/O запросов
```

### CancellationToken — отмена

```csharp
public async Task<string> ReadAsync(string path, CancellationToken ct = default)
{
    using var stream = File.OpenRead(path);
    using var reader = new StreamReader(stream);

    return await reader.ReadToEndAsync(ct);
}

// Вызов с timeout
using var cts = new CancellationTokenSource(TimeSpan.FromSeconds(5));
try
{
    string content = await ReadAsync("file.txt", cts.Token);
}
catch (OperationCanceledException)
{
    // timeout
}
```

### Параллельное чтение

```csharp
// Read multiple files concurrently
var files = new[] { "a.txt", "b.txt", "c.txt" };

var tasks = files.Select(f => File.ReadAllTextAsync(f));
string[] contents = await Task.WhenAll(tasks);

// С limited concurrency
var sem = new SemaphoreSlim(maxParallel: 4);
var tasks = files.Select(async f =>
{
    await sem.WaitAsync();
    try { return await File.ReadAllTextAsync(f); }
    finally { sem.Release(); }
});
```

См. [[async-threading|Async и Threading]].

---

## 7. Streams composition — wrapping

Streams можно **обернуть** друг в друга:

```csharp
// File → Buffer → Decompression → Read text
using FileStream fs = File.OpenRead("file.gz");
using GZipStream gzip = new(fs, CompressionMode.Decompress);
using StreamReader reader = new(gzip);

string content = await reader.ReadToEndAsync();

// Reverse: text → encryption → compression → file
using FileStream fs = File.OpenWrite("output.bin.gz");
using GZipStream gzip = new(fs, CompressionLevel.Optimal);
using CryptoStream crypto = new(gzip, encryptor, CryptoStreamMode.Write);
using StreamWriter writer = new(crypto);

await writer.WriteAsync("secret data");
```

### Compression

```csharp
// Compress file
async Task CompressFileAsync(string source, string destination)
{
    using FileStream input = File.OpenRead(source);
    using FileStream output = File.Create(destination);
    using GZipStream gzip = new(output, CompressionLevel.Optimal);

    await input.CopyToAsync(gzip);
}

// Decompress
async Task DecompressFileAsync(string source, string destination)
{
    using FileStream input = File.OpenRead(source);
    using GZipStream gzip = new(input, CompressionMode.Decompress);
    using FileStream output = File.Create(destination);

    await gzip.CopyToAsync(output);
}
```

### CopyToAsync — efficient streaming

```csharp
// Copy stream to stream — без полного загрузки в память
using FileStream source = File.OpenRead("big.bin");
using FileStream dest = File.Create("copy.bin");
await source.CopyToAsync(dest);

// С buffer size
await source.CopyToAsync(dest, bufferSize: 81920);
```

---

## 8. JSON I/O

```csharp
using System.Text.Json;

// Write
var user = new { Name = "John", Age = 30 };
await using FileStream fs = File.Create("user.json");
await JsonSerializer.SerializeAsync(fs, user);

// Read
await using FileStream fs = File.OpenRead("user.json");
var user = await JsonSerializer.DeserializeAsync<User>(fs);

// String version
string json = JsonSerializer.Serialize(user);
User user = JsonSerializer.Deserialize<User>(json);

// Pretty
var options = new JsonSerializerOptions { WriteIndented = true };
string pretty = JsonSerializer.Serialize(user, options);
```

### JSON streaming (большие файлы)

```csharp
// Read массив больших данных не загружая всё в память
await using FileStream fs = File.OpenRead("huge.json");

await foreach (var item in JsonSerializer.DeserializeAsyncEnumerable<Item>(fs))
{
    ProcessItem(item);
}
```

---

## 9. CSV — парсинг

CSV в .NET — нет built-in, но есть libraries:

```bash
dotnet add package CsvHelper
```

```csharp
using CsvHelper;
using System.Globalization;

// Read
using var reader = new StreamReader("data.csv");
using var csv = new CsvReader(reader, CultureInfo.InvariantCulture);

var records = csv.GetRecords<Person>().ToList();

// Write
using var writer = new StreamWriter("output.csv");
using var csv = new CsvWriter(writer, CultureInfo.InvariantCulture);

csv.WriteRecords(records);
```

---

## 10. System.IO.Pipelines — high-perf

Для maximum performance — modern API (.NET Core 2.1+).

```csharp
using System.IO.Pipelines;

// Reader
PipeReader reader = PipeReader.Create(stream);

while (true)
{
    ReadResult result = await reader.ReadAsync();
    ReadOnlySequence<byte> buffer = result.Buffer;

    // Process buffer (multiple segments possible)
    foreach (ReadOnlyMemory<byte> segment in buffer)
    {
        // ... process bytes ...
    }

    // Mark consumed
    reader.AdvanceTo(buffer.End);

    if (result.IsCompleted) break;
}

await reader.CompleteAsync();
```

**Когда:** parsing protocols (HTTP, custom binary), streaming gigabytes of data, hot path performance.

**Когда не:** простые scenarios — Stream API проще.

См. [[../Runtime/span-layout|Span и Layout]] для контекста.

---

## 11. Memory-mapped files

Для **очень больших файлов** (>1 GB) — map в virtual memory без полного load.

```csharp
using System.IO.MemoryMappedFiles;

using MemoryMappedFile mmf = MemoryMappedFile.CreateFromFile("huge.dat", FileMode.Open);
using MemoryMappedViewAccessor accessor = mmf.CreateViewAccessor();

// Read at offset
int value = accessor.ReadInt32(offset: 1024);

// Write
accessor.Write(offset: 0, value: 42);

// Read structure
struct Record
{
    public int Id;
    public double Value;
}

accessor.Read<Record>(offset: 0, out Record record);
```

**Когда:**
- Random access к большим файлам
- Inter-process communication через shared memory
- Очень частые reads / writes к разным частям

---

## 12. Common Pitfalls

### 1. Sync I/O в web app

```csharp
// ❌ Async controller с sync I/O
public IActionResult Get()
{
    var data = File.ReadAllText("config.json");  // sync!
    return Ok(data);
}

// ✅
public async Task<IActionResult> Get()
{
    var data = await File.ReadAllTextAsync("config.json");
    return Ok(data);
}
```

### 2. Не Dispose stream

```csharp
// ❌ Stream остаётся открыт — file lock!
StreamReader reader = new("file.txt");
string content = reader.ReadToEnd();
// reader не disposed — file locked

// ✅ using
using StreamReader reader = new("file.txt");
string content = await reader.ReadToEndAsync();
// auto-dispose
```

### 3. ReadAllText для больших файлов

```csharp
// ❌ 5GB файл — OutOfMemoryException
string content = File.ReadAllText("huge.log");

// ✅ Streaming
foreach (var line in File.ReadLines("huge.log"))
{
    Process(line);
}

// ✅ Async
await foreach (var line in File.ReadLinesAsync("huge.log"))
{
    await ProcessAsync(line);
}
```

### 4. Не передавать CancellationToken

```csharp
// ❌ Нельзя отменить
public async Task<string> Read(string path)
{
    return await File.ReadAllTextAsync(path);
}

// ✅ С токеном
public async Task<string> ReadAsync(string path, CancellationToken ct)
{
    return await File.ReadAllTextAsync(path, ct);
}
```

### 5. Encoding mismatch

```csharp
// File saved as UTF-8 with BOM на Windows
// Читаем без encoding:
string s = File.ReadAllText("file.txt");
// .NET 5+ — auto-detect UTF-8 BOM, OK
// .NET Framework — может прочитать неправильно

// ✅ Explicit
string s = File.ReadAllText("file.txt", Encoding.UTF8);
```

### 6. Path injection

```csharp
// ❌ User controls path — можно прочитать любой файл!
public string GetFile(string fileName)
{
    return File.ReadAllText($"data/{fileName}");
    // Если fileName = "../../etc/passwd" — exploit!
}

// ✅ Validate
public string GetFile(string fileName)
{
    if (fileName.Contains("..") || Path.IsPathRooted(fileName))
        throw new ArgumentException("Invalid path");

    var fullPath = Path.GetFullPath(Path.Combine("data", fileName));
    if (!fullPath.StartsWith(Path.GetFullPath("data")))
        throw new ArgumentException("Path escape");

    return File.ReadAllText(fullPath);
}
```

### 7. Filestream без `useAsync`

```csharp
// FileStream имеет flag для async — необходим для true async I/O
new FileStream(path, FileMode.Open, FileAccess.Read, FileShare.Read,
    bufferSize: 4096,
    useAsync: true);  // ✅ для async чтения

// Без useAsync — async методы могут реально работать sync под капотом
```

### 8. Race condition в file writes

```csharp
// ❌ Two threads пишут в один файл
File.AppendAllText("log.txt", "...");  // race!

// ✅ Lock
private static readonly object _lock = new();
lock (_lock)
{
    File.AppendAllText("log.txt", "...");
}

// ✅ Лучше — использовать ILogger (thread-safe)
```

### 9. StreamReader не Dispose'ит inner stream

```csharp
using FileStream fs = File.OpenRead("file.txt");
using StreamReader reader = new(fs);

// reader.Dispose() закроет ALSO fs!
// Если хочешь сохранить fs:
using StreamReader reader = new(fs, Encoding.UTF8, true, 1024, leaveOpen: true);
```

### 10. Long path на Windows

```csharp
// Windows historical limit — 260 chars
// .NET 4.6.2+ / .NET Core — long paths support
// Но: enable в OS / app.config:

// PathTooLongException
File.ReadAllText(@"C:\very\long\path\..."); // > 260 chars — throws

// Или префикс \\?\
File.ReadAllText(@"\\?\C:\very\long\path...");
```

---

## 13. Best Practices

### Performance

- **`File.ReadLines` / `ReadLinesAsync`** для больших файлов (lazy)
- **`CopyToAsync`** для копирования streams (не всё в память)
- **Buffer size 4-8KB** для streams (default OK обычно)
- **`System.IO.Pipelines`** для extreme performance
- **Memory-mapped files** для random access на gigabytes
- **Streaming JSON** через `DeserializeAsyncEnumerable`

### Async

- **Async всегда** в web app / library
- **CancellationToken** в каждом async I/O method
- **`useAsync: true`** в FileStream для true async
- **`Task.WhenAll`** для параллельных reads
- **`SemaphoreSlim`** для ограничения concurrency

### Safety

- **`using`** для всех Streams / Readers / Writers
- **`Path.Combine`** не string concatenation
- **Validate user paths** (path injection prevention)
- **Specify encoding explicitly** для production
- **`FileShare.Read`** если другие читают (concurrent reads OK)

### Error handling

- **`File.Exists`** перед операцией если возможен miss
- **Catch IOException** для file errors
- **`UnauthorizedAccessException`** для permissions
- **`PathTooLongException`** на Windows legacy

### Cross-platform

- **`Path.Combine`** для путей
- **`Path.DirectorySeparatorChar`** если нужен char
- **`Environment.NewLine`** для line endings
- **`Encoding.UTF8`** explicit (default везде сейчас)

---

## 14. Cheat sheet

| Задача | API |
|--------|-----|
| Read text file | `File.ReadAllTextAsync(path)` |
| Read line by line | `File.ReadLinesAsync(path)` |
| Write text | `File.WriteAllTextAsync(path, content)` |
| Append text | `File.AppendAllTextAsync(path, content)` |
| Read bytes | `File.ReadAllBytesAsync(path)` |
| File exists | `File.Exists(path)` |
| Copy file | `File.Copy(src, dst, overwrite: true)` |
| Delete file | `File.Delete(path)` |
| List files | `Directory.EnumerateFiles(folder, "*.txt")` |
| Combine path | `Path.Combine(parts)` |
| Get extension | `Path.GetExtension(file)` |
| Temp file | `Path.GetTempFileName()` |
| Stream → string | `await reader.ReadToEndAsync()` |
| Stream → stream | `await source.CopyToAsync(dest)` |
| Compress | `GZipStream(stream, CompressionLevel.Optimal)` |
| Memory-mapped | `MemoryMappedFile.CreateFromFile(path, FileMode.Open)` |
| JSON read | `JsonSerializer.DeserializeAsync<T>(stream)` |
| Big files | `File.ReadLinesAsync` или streaming |

---

## 15. Decision tree

```
Что нужно сделать?
│
├── Read текстовый файл?
│   ├── Маленький (< 100 MB)?
│   │   → File.ReadAllTextAsync
│   └── Большой?
│       → File.ReadLinesAsync (line by line)
│
├── Write текст?
│   → File.WriteAllTextAsync (overwrite)
│   → File.AppendAllTextAsync (append)
│
├── Bytes?
│   → File.ReadAllBytesAsync / WriteAllBytesAsync
│
├── Streaming большой файл?
│   → FileStream + StreamReader/Writer
│   → CopyToAsync для копирования
│
├── JSON?
│   → JsonSerializer.SerializeAsync / DeserializeAsync
│   → DeserializeAsyncEnumerable для streaming
│
├── CSV?
│   → CsvHelper library
│
├── Compression?
│   → GZipStream / DeflateStream
│
├── Random access на гигабайты?
│   → MemoryMappedFile
│
├── Maximum performance?
│   → System.IO.Pipelines + Span<byte>
│
└── Process IPC?
    → Named Pipes (см. ipc-named-pipes-grpc)
```

---

## См. также

- [[async-threading|Async и Threading]] — почему async I/O
- [[strings-regex|Strings и Regex]] — encoding deep
- [[../Runtime/span-layout|Span и Layout]] — Pipelines, ReadOnlySequence
- [[../Infrastructure/ipc-named-pipes-grpc|IPC: Named Pipes / gRPC]]
- [[../AspNetCore/api-design|API Design]] — Stream return types

## Reading list

- **Microsoft Docs — File and Stream I/O** — learn.microsoft.com/dotnet/standard/io
- **Microsoft Docs — System.IO.Pipelines** — learn.microsoft.com/dotnet/standard/io/pipelines
- **Stephen Toub — System.IO.Pipelines deep dive** — devblogs.microsoft.com/dotnet
- **Andrew Lock — File I/O blogs** — andrewlock.net
- **Async file I/O** — David Fowler GitHub gists
- **CsvHelper docs** — joshclose.github.io/CsvHelper
