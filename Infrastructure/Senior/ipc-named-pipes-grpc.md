---
tags: [ipc, named-pipes, memory-mapped-files, grpc, shared-memory, performance]
level: Senior
date: 2026-06-28
---

# Inter-Process Communication (IPC) в .NET

> Передача данных между процессами в .NET: Named Pipes, Memory-Mapped Files (zero-copy ring buffer), Anonymous Pipes и gRPC over HTTP/2 — выбор транспорта по латентности, размеру payload и cross-language требованиям.

## Что это, зачем и когда

### Что такое IPC?
**Способ передавать данные между разными процессами** — на одной машине или по сети. В отличие от потоков внутри процесса, процессы не делят адресное пространство, поэтому им нужен явный канал связи: pipe, shared memory, socket, очередь.

**Аналогия:** Потоки внутри процесса — соседи по квартире, могут передавать вещи из руки в руки. Процессы — это разные квартиры, нужен или почтовый ящик (pipe), или общий склад на лестнице (shared memory), или курьер (RPC по сети).

### Зачем процессы вместо потоков?

| Один процесс с потоками | Несколько процессов |
|------------------------|---------------------|
| Краш одного потока валит весь процесс | Один процесс упал — другие живут |
| Утечка памяти копится в одном heap | Каждый процесс изолирован, рестарт = очистка |
| Все потоки обязаны быть на одном языке/runtime | C# процесс может говорить с Python/C++/Rust |
| Один CPU-bound поток блокирует GC всех остальных | Каждый процесс — свой GC, свой ThreadPool |
| Простая отладка | Сложнее — нужно дебажить несколько процессов |

### Когда нужен IPC

| Сценарий | Почему IPC |
|----------|-----------|
| Hot-reload UI без рестарта движка | UI-процесс живёт, движок перезапускается |
| HFT-бот: connector в отдельном процессе для изоляции | Краш парсера фида не валит торговую логику |
| Sandbox для пользовательских скриптов | Untrusted код запускается с ограниченными правами в child-процессе |
| Worker pool для CPU-bound задач | Обходим GIL/GC contention, масштабируемся по физическим ядрам |
| Интеграция с legacy C++ библиотекой | Native dll → отдельный helper-процесс → IPC |
| Microservices на одной машине | gRPC-сервисы в Docker-compose |

---

## Обзор технологий

```
                    Локально на одной машине       │      По сети
                    ────────────────────────────────│─────────────────────
Низкая латентность  Memory-Mapped Files (zero-copy) │
       ▲             ↑                              │
       │            Shared Memory + Named Mutex     │   gRPC over HTTP/2
       │            Named Pipes (Windows)           │   (binary)
       │            Unix Domain Sockets (Linux)     │
       │            Anonymous Pipes (parent-child)  │   WebSocket / SignalR
       │                                            │
       ▼            Loopback TCP / NamedPipes       │   REST / JSON
Низкая  сложность                                   │
```

| Технология | Скорость | Размер payload | Cross-platform | Cross-language |
|-----------|----------|---------------|---------------|----------------|
| **Memory-Mapped Files** | Самая быстрая (~1µs) | До GB | Да | Да (через C-API) |
| **Named Pipes** | Очень быстро (~10µs) | До MB | Да в .NET (Linux через Unix-domain) | Да |
| **Anonymous Pipes** | Быстро | Stream | Да | Только parent-child |
| **Unix Domain Sockets** | Очень быстро | Stream | Linux/macOS only | Да |
| **gRPC** | Хорошо (~100µs локально) | До GB (streaming) | Да | Отлично |
| **Localhost TCP** | Хорошо (~100µs) | Stream | Да | Да |
| **MSMQ / RabbitMQ** | Медленно (~ms) | До MB | Да | Да |

---

## Named Pipes

**Канал между процессами в стиле "файла"** — клиент открывает по имени, сервер слушает на том же имени. На Windows реализовано через ядерный объект, на Linux .NET транслирует в Unix Domain Socket автоматически.

### Когда применять
- Локальный IPC внутри одной машины
- Двусторонняя связь request/response или streaming
- Когда ты пишешь и сервер, и клиент в .NET (или хотя бы знаешь обе стороны)

### Server (одна линия за раз)

```csharp
using System.IO.Pipes;
using System.Text;

const string PipeName = "tradingbot.commands";

// Сервер слушает в цикле
while (!ct.IsCancellationRequested)
{
    using var server = new NamedPipeServerStream(
        pipeName: PipeName,
        direction: PipeDirection.InOut,
        maxNumberOfServerInstances: 1,
        transmissionMode: PipeTransmissionMode.Byte,
        options: PipeOptions.Asynchronous);

    await server.WaitForConnectionAsync(ct);
    logger.LogInformation("Client connected");

    using var reader = new StreamReader(server, Encoding.UTF8, leaveOpen: true);
    using var writer = new StreamWriter(server, Encoding.UTF8, leaveOpen: true) { AutoFlush = true };

    while (server.IsConnected && !ct.IsCancellationRequested)
    {
        var line = await reader.ReadLineAsync(ct);
        if (line is null) break;

        var response = await ProcessCommandAsync(line, ct);
        await writer.WriteLineAsync(response.AsMemory(), ct);
    }
}
```

### Server (множество клиентов параллельно)

```csharp
public async Task RunMultiClientServerAsync(CancellationToken ct)
{
    var clientId = 0;
    while (!ct.IsCancellationRequested)
    {
        var server = new NamedPipeServerStream(
            PipeName,
            PipeDirection.InOut,
            maxNumberOfServerInstances: NamedPipeServerStream.MaxAllowedServerInstances,
            PipeTransmissionMode.Byte,
            PipeOptions.Asynchronous);

        await server.WaitForConnectionAsync(ct);

        var id = Interlocked.Increment(ref clientId);

        // Каждый клиент — отдельная задача, не блокируем accept loop
        _ = Task.Run(() => HandleClientAsync(server, id, ct), ct);
    }
}

private async Task HandleClientAsync(NamedPipeServerStream pipe, int clientId, CancellationToken ct)
{
    try
    {
        // ... обработка клиента
    }
    catch (Exception ex)
    {
        logger.LogError(ex, "Client {Id} failed", clientId);
    }
    finally
    {
        await pipe.DisposeAsync();
    }
}
```

### Client

```csharp
using var client = new NamedPipeClientStream(
    serverName: ".",                       // "." = localhost
    pipeName: PipeName,
    direction: PipeDirection.InOut,
    options: PipeOptions.Asynchronous);

await client.ConnectAsync(timeout: 5000, ct);

using var writer = new StreamWriter(client, Encoding.UTF8, leaveOpen: true) { AutoFlush = true };
using var reader = new StreamReader(client, Encoding.UTF8, leaveOpen: true);

await writer.WriteLineAsync("PING");
var response = await reader.ReadLineAsync(ct); // "PONG"
```

### Security (Windows)

По умолчанию named pipe доступен от пользователя, под которым запущен процесс. Чтобы разрешить другим — настрой `PipeSecurity`:

```csharp
var security = new PipeSecurity();
security.AddAccessRule(new PipeAccessRule(
    new SecurityIdentifier(WellKnownSidType.AuthenticatedUserSid, null),
    PipeAccessRights.ReadWrite,
    AccessControlType.Allow));

var server = NamedPipeServerStreamAcl.Create(
    PipeName, PipeDirection.InOut, 1,
    PipeTransmissionMode.Byte, PipeOptions.Asynchronous, 0, 0, security);
```

> [!question]- **Интервью: какие подводные камни Named Pipes?**
> 1. **`.` vs `localhost`** — для NamedPipeClientStream сервер указывается как `"."` (текущая машина), не `"localhost"`. Это специфика Windows-API.
> 2. **MaxNumberOfServerInstances** — если выставить 1, второй клиент не подключится пока первый не отключится. По умолчанию советую `NamedPipeServerStream.MaxAllowedServerInstances` и обрабатывать каждого клиента в отдельной задаче.
> 3. **Кириллица** — без явной `Encoding.UTF8` ловишь "?????" на не-латинице (default — `Encoding.Default` зависит от locale).
> 4. **Linux** — на Linux пайп создаётся как Unix Domain Socket в `/tmp/CoreFxPipe_{name}`. Это transparent для .NET, но если на Linux другой процесс (не .NET) пытается подключиться по тому же имени — он не найдёт его как named pipe, нужно искать по пути.

---

## Message framing над stream-транспортом (TCP / NetworkStream)

Pitfall #2 выше говорит: для cross-platform всегда `Byte`-режим + framing руками. То же верно для **любого** byte-stream транспорта — TCP-сокета, `NetworkStream`, Named Pipe в `Byte`-режиме. Здесь — почему это обязательно и как именно выглядит цикл, которого нет в большинстве туториалов.

### Почему «один ReadAsync = одно сообщение» — ложь

TCP — это **поток байт без границ сообщений**. Транспорт волен порезать твои 1000 байт на TCP-сегменты как угодно: их склеит Nagle, разрежет MTU, перемешает буферизация ядра. `Stream.ReadAsync` гарантирует только одно: вернёт **от 1 до N** байт (либо 0 на EOF). Он НЕ обязан вернуть ровно столько, сколько ты просил, даже если данные «вроде бы уже пришли».

Отсюда два бага, которые в проде всплывают только под нагрузкой:

- **Short read** — попросил 28 байт заголовка, получил 11. Если распарсить буфер как есть — мусор в полях длины → следующий кадр уедет на произвольный offset, и поток рассинхронизируется навсегда.
- **Coalescing** — два логических сообщения пришли одним `ReadAsync`. Без префикса длины ты не знаешь, где кончается первое.

Вывод: нужен **явный протокол кадрирования**. Самый ходовой — *length-prefixed framing*: фиксированный заголовок (включающий длину тела) + тело объявленной длины.

> [!info] Почему фиксированный заголовок, а не разделитель
> Delimiter-based framing (`\n`, как `ReadLineAsync`) ломается на бинарных payload'ах, где байт-разделитель легально встречается внутри данных, и требует escape'инга. Length-prefix работает с любыми байтами и позволяет аллоцировать буфер тела ровно один раз, зная размер заранее.

### Сердце протокола — `ReadExactAsync`

`Stream.ReadExactlyAsync` существует с .NET 7 и делает именно это. Но понимать **руками написанный цикл** обязательно: это типовой вопрос на собесе, и он нужен, когда ты работаешь напрямую с `Socket.ReceiveAsync` (где аналога нет вовсе) либо хочешь нулевую зависимость от поведения конкретного `Stream`.

Цикл крутит `ReadAsync` в `buffer[totalRead..]`, пока не наберёт ровно `buffer.Length`, и **бросает на 0-байтном чтении** — это premature EOF (пир закрыл соединение посреди кадра), а не «данных пока нет»:

```csharp
using System.IO;
using System.Threading;
using System.Threading.Tasks;

/// <summary>
/// Reads exactly <paramref name="buffer"/>.Length bytes, looping until the
/// buffer is full. Throws on a 0-byte read (peer closed mid-frame = EOF).
/// Equivalent to Stream.ReadExactlyAsync, hand-rolled for zero dependency
/// on a specific Stream's read semantics (or for use over raw Socket).
/// </summary>
private static async ValueTask ReadExactAsync(
    Stream stream,
    Memory<byte> buffer,
    CancellationToken ct)
{
    var totalRead = 0;
    while (totalRead < buffer.Length)
    {
        var read = await stream
            .ReadAsync(buffer[totalRead..], ct)
            .ConfigureAwait(false);

        // 0 means orderly shutdown by the peer. Mid-frame it is a protocol
        // violation, not "no data yet" — never spin-loop here.
        if (read == 0)
        {
            throw new EndOfStreamException(
                $"Premature EOF: expected {buffer.Length} bytes, got {totalRead}.");
        }

        totalRead += read;
    }
}
```

> [!warning] Никогда не игнорируй `read == 0`
> Самая частая ошибка — `while (totalRead < len) totalRead += await ReadAsync(...)` без проверки нуля. Когда пир закрывает сокет, `ReadAsync` начнёт вечно возвращать 0 — получаешь busy-loop, жгущий ядро на 100% CPU, вместо честного исключения.

### Заголовок → тело: header-first, затем body

Логика приёма кадра в два шага: сперва `ReadExactAsync` фиксированного заголовка (узнаём объявленную длину тела), потом `ReadExactAsync` тела ровно этой длины. Парсим заголовок через `ReadOnlySpan<byte>` + `BinaryPrimitives` — **ноль промежуточных аллокаций** (никаких `BitConverter` + срезов массива):

```csharp
using System.Buffers;
using System.Buffers.Binary;
using System.IO;
using System.Threading;
using System.Threading.Tasks;

// Wire header: [4B magic][1B type][4B bodyLength], big-endian. Total 9 bytes.
private const int HeaderSize = 9;
private const uint Magic = 0x5446_5031; // "TFP1"
private const int MaxBodyLength = 16 * 1024 * 1024; // guard against hostile length

private static async ValueTask<(byte Type, byte[] Body)> ReadFrameAsync(
    Stream stream,
    CancellationToken ct)
{
    // 1) Header first — fixed size, so we know exactly how much to read.
    var header = new byte[HeaderSize];
    await ReadExactAsync(stream, header, ct).ConfigureAwait(false);

    // Parse with a Span over the stack-free buffer — zero allocations.
    var span = (ReadOnlySpan<byte>)header;
    var magic = BinaryPrimitives.ReadUInt32BigEndian(span[..4]);
    if (magic != Magic)
    {
        throw new InvalidDataException($"Bad magic 0x{magic:X8}; stream desynced.");
    }

    var type = span[4];
    var bodyLength = BinaryPrimitives.ReadInt32BigEndian(span[5..9]);

    // Always validate length from the wire BEFORE allocating — a malicious or
    // corrupt peer can declare 2 GB and OOM you.
    if ((uint)bodyLength > MaxBodyLength)
    {
        throw new InvalidDataException($"Body length {bodyLength} exceeds cap.");
    }

    // 2) Body — exactly bodyLength bytes, learned from the header.
    var body = new byte[bodyLength];
    await ReadExactAsync(stream, body, ct).ConfigureAwait(false);

    return (type, body);
}
```

> [!tip] Для горячего пути арендуй буфер тела
> Под нагрузкой `new byte[bodyLength]` на кадр — давление на GC. Бери `ArrayPool<byte>.Shared.Rent(bodyLength)`, читай в `buffer.AsMemory(0, bodyLength)` и возвращай в `finally`. См. [[memory-pooling|Memory Pooling]].

### Когда писать цикл руками, а когда — `System.IO.Pipelines`

Это **развилка по уровню контроля**, не «старое vs новое»:

| Берёшь | Когда | Почему |
|--------|-------|--------|
| Hand-rolled `ReadExactAsync` loop | Контролируешь `Socket` / `NetworkStream`, нужны нулевые framework-зависимости, простой кадр fixed-header + body | Минимум кода, полностью предсказуемо, легко портировать на `Socket.ReceiveAsync` где `ReadExactlyAsync` нет |
| `System.IO.Pipelines` + `SequenceReader` | Высокий throughput, много мелких кадров, backpressure, парсинг поверх фрагментированного `ReadOnlySequence\<byte\>` | Пул буферов и backpressure из коробки; нет ручного управления partial reads; `SequenceReader.TryReadExact` достаёт кадр без копирования через сегменты |

С Pipelines тот же header-first паттерн выглядит так — `TryRead` накапливает данные в `PipeReader`, а парсер просто сообщает, хватило ли байт на полный кадр:

```csharp
using System.Buffers;
using System.IO.Pipelines;
using System.Threading;
using System.Threading.Tasks;

private static async ValueTask ProcessFramesAsync(PipeReader reader, CancellationToken ct)
{
    while (true)
    {
        var result = await reader.ReadAsync(ct).ConfigureAwait(false);
        var buffer = result.Buffer;

        // Pull as many complete frames as the buffer currently holds.
        while (TryParseFrame(ref buffer, out var frame))
        {
            Handle(frame);
        }

        // Tell the pipe what we consumed vs. merely examined (drives backpressure
        // and tells the pipe to keep the not-yet-complete tail for next ReadAsync).
        reader.AdvanceTo(buffer.Start, buffer.End);

        if (result.IsCompleted)
        {
            break;
        }
    }

    await reader.CompleteAsync().ConfigureAwait(false);
}

private static bool TryParseFrame(ref ReadOnlySequence<byte> buffer, out ReadOnlySequence<byte> frame)
{
    var sr = new SequenceReader<byte>(buffer);

    // Need the full fixed header before we can learn the body length.
    if (!sr.TryReadExact(HeaderSize, out var headerSeq))
    {
        frame = default;
        return false; // not enough buffered yet — wait for the next ReadAsync
    }

    // Header may span segments; copy 9 bytes to the stack to parse.
    Span<byte> header = stackalloc byte[HeaderSize];
    headerSeq.CopyTo(header);
    var bodyLength = BinaryPrimitives.ReadInt32BigEndian(header[5..9]);

    if (!sr.TryReadExact(bodyLength, out frame))
    {
        frame = default;
        return false; // header arrived, body hasn't — keep waiting
    }

    buffer = buffer.Slice(sr.Position); // consume header + body
    return true;
}
```

> [!question]- **Интервью: TCP «потерял половину сообщения» — что не так в коде?**
> Почти наверняка короткое чтение: один `ReadAsync` приняли за один кадр и распарсили частичный буфер. Транспорт не обязан возвращать столько байт, сколько просили. Решение — `ReadExactAsync`-цикл (header-first, затем body по объявленной длине) либо `System.IO.Pipelines`. Контрольный вопрос «а что при `read == 0`?» отсекает тех, кто словит busy-loop на закрытии соединения.

---

## Memory-Mapped Files (Shared Memory)

**Файл (или анонимный регион памяти), отображённый в адресное пространство нескольких процессов.** Запись в этот файл от одного процесса мгновенно видна другому без копирования через ядро. Самый быстрый IPC — **порядка наносекунд**.

### Когда применять
- Передача больших объёмов данных (изображения, аудио-буферы, market data feed)
- Hot-path где IPC вызывается миллионы раз в секунду
- Producer-consumer pattern с ring-buffer
- Доступ к одной структуре из нескольких процессов с явной синхронизацией

### Простейший пример — общий счётчик

```csharp
using System.IO.MemoryMappedFiles;

// Producer
using var mmf = MemoryMappedFile.CreateOrOpen(
    mapName: "Global\\trading_counter",  // "Global\\" prefix — доступ из любой сессии Windows
    capacity: 1024);

using var accessor = mmf.CreateViewAccessor();
accessor.Write(0, 42L);   // long на offset 0
```

```csharp
// Consumer
using var mmf = MemoryMappedFile.OpenExisting("Global\\trading_counter");
using var accessor = mmf.CreateViewAccessor();
var value = accessor.ReadInt64(0); // 42
```

### Структуры через `MemoryMappedViewAccessor.Read<T>`

```csharp
[StructLayout(LayoutKind.Sequential, Pack = 1)]
public struct MarketTick
{
    public long Timestamp;     // 8 bytes
    public double Bid;          // 8
    public double Ask;          // 8
    public int Volume;          // 4
                                // total: 28 bytes
}

// Producer
using var accessor = mmf.CreateViewAccessor(0, 1024);
var tick = new MarketTick
{
    Timestamp = DateTimeOffset.UtcNow.ToUnixTimeMilliseconds(),
    Bid = 1.08501,
    Ask = 1.08503,
    Volume = 1000,
};
accessor.Write(0, ref tick);

// Consumer
accessor.Read(0, out MarketTick t);
```

### Ring-buffer через MMF — production pattern для market data

Один процесс (parser) пишет тики в кольцо, другой (стратегия) читает по своему индексу. Без блокировок при одном producer + одном consumer (SPSC).

```csharp
[StructLayout(LayoutKind.Sequential)]
public struct RingHeader
{
    public long WriteIndex;       // sequenced, monotonic
    public long ReadIndex;        // consumer-side
    public int Capacity;
}

public sealed unsafe class TickRingBuffer : IDisposable
{
    const string MapName = "Global\\market_ticks";
    const int Capacity = 65536;  // power of 2
    const int TickSize = 32;      // sizeof(MarketTick) padded to 32

    private readonly MemoryMappedFile _mmf;
    private readonly MemoryMappedViewAccessor _accessor;
    private readonly byte* _basePtr;

    public TickRingBuffer()
    {
        _mmf = MemoryMappedFile.CreateOrOpen(MapName, sizeof(RingHeader) + Capacity * TickSize);
        _accessor = _mmf.CreateViewAccessor();
        byte* ptr = null;
        _accessor.SafeMemoryMappedViewHandle.AcquirePointer(ref ptr);
        _basePtr = ptr;
    }

    private RingHeader* Header => (RingHeader*)_basePtr;
    private MarketTick* Slots => (MarketTick*)(_basePtr + sizeof(RingHeader));

    public bool TryWrite(in MarketTick tick)
    {
        var write = Volatile.Read(ref Header->WriteIndex);
        var read = Volatile.Read(ref Header->ReadIndex);

        if (write - read >= Capacity) return false;  // full

        Slots[write & (Capacity - 1)] = tick;

        // Release fence — consumer должен увидеть tick до увидения нового WriteIndex
        Volatile.Write(ref Header->WriteIndex, write + 1);
        return true;
    }

    public bool TryRead(out MarketTick tick)
    {
        var read = Volatile.Read(ref Header->ReadIndex);
        var write = Volatile.Read(ref Header->WriteIndex);

        if (read >= write) { tick = default; return false; }  // empty

        tick = Slots[read & (Capacity - 1)];

        Volatile.Write(ref Header->ReadIndex, read + 1);
        return true;
    }

    public void Dispose()
    {
        _accessor.SafeMemoryMappedViewHandle.ReleasePointer();
        _accessor.Dispose();
        _mmf.Dispose();
    }
}
```

### Синхронизация при множестве producer/consumer

Для many-producer/many-consumer SPSC паттерн не подходит — нужен либо MPMC ring (сложнее, atomic CAS), либо `Mutex`/`Semaphore`/`EventWaitHandle` поверх MMF:

```csharp
// Producer-Consumer wake-up через EventWaitHandle
var dataReady = new EventWaitHandle(false, EventResetMode.AutoReset, "Global\\market_data_event");

// Producer
WriteData();
dataReady.Set();   // wake one consumer

// Consumer
dataReady.WaitOne();
ReadData();
```

### Cleanup и persistence

- `MemoryMappedFile.CreateOrOpen` без указания файла — **анонимная память**, исчезает при выходе всех процессов
- `MemoryMappedFile.CreateFromFile` — реальный файл на диске, переживает рестарты
- На Windows названия с `Global\\` префиксом — доступны из любой сессии (Service vs User session). Без префикса — только в текущей сессии

> [!question]- **Интервью: чем MMF отличается от Named Pipes по производительности?**
> Named Pipe передаёт данные через ядро: один процесс пишет в буфер ядра → другой читает из буфера ядра, две копии. MMF — общий регион физической памяти, видимый обоим процессам через виртуальные таблицы страниц. **Нет копирования вообще** — один процесс пишет, другой видит мгновенно.
> Latency — pipe ~5-15µs, MMF ~50-200ns (в 100x быстрее). Но сложнее: нужна явная синхронизация, нет встроенного wake-up консьюмера, нет message framing'а.

---

## Anonymous Pipes (parent-child)

Когда родительский процесс запускает дочерний и хочет с ним общаться через `stdin`/`stdout`:

```csharp
var process = Process.Start(new ProcessStartInfo
{
    FileName = "child.exe",
    UseShellExecute = false,
    RedirectStandardInput = true,
    RedirectStandardOutput = true,
    RedirectStandardError = true,
    CreateNoWindow = true,
})!;

// Send command
await process.StandardInput.WriteLineAsync("PROCESS_FILE C:\\input.csv");
await process.StandardInput.FlushAsync();

// Read response (асинхронно построчно)
var line = await process.StandardOutput.ReadLineAsync(ct);
```

Хорош для:
- CLI-утилит, которые ты вызываешь из своего сервиса
- Изоляции untrusted кода (parent убивает child при подозрении)
- Простого pipeline'а из нескольких процессов

Минусы: только parent-child, не годится для постоянных IPC-каналов.

---

## gRPC — RPC между процессами и сервисами

**HTTP/2-based RPC с binary serialization (Protocol Buffers).** Можно использовать как для микросервисов по сети, так и для IPC внутри одной машины (через Unix Domain Sockets или localhost TCP).

### Когда применять
- Между микросервисами в Docker-compose / Kubernetes
- IPC между .NET-процессами разных языков
- Streaming большие потоки данных (server-streaming, client-streaming, bidi)
- Когда нужен формальный контракт через `.proto`

### Proto-файл

```protobuf
syntax = "proto3";
package trading;

option csharp_namespace = "TradingBot.Grpc";

service MarketData {
  rpc SubscribeQuotes (SubscribeRequest) returns (stream Quote);
  rpc PlaceOrder (PlaceOrderRequest) returns (PlaceOrderResponse);
  rpc Stream (stream OrderUpdate) returns (stream OrderResult);
}

message SubscribeRequest {
  repeated string symbols = 1;
}

message Quote {
  string symbol = 1;
  double bid = 2;
  double ask = 3;
  int64 timestamp = 4;
}

message PlaceOrderRequest {
  string symbol = 1;
  double quantity = 2;
  double price = 3;
}

message PlaceOrderResponse {
  string order_id = 1;
  OrderStatus status = 2;
}

enum OrderStatus {
  PENDING = 0;
  FILLED = 1;
  REJECTED = 2;
}
```

### Server (ASP.NET Core)

```csharp
// Program.cs
builder.Services.AddGrpc(options =>
{
    options.MaxReceiveMessageSize = 16 * 1024 * 1024; // 16 MB
    options.EnableDetailedErrors = builder.Environment.IsDevelopment();
});

var app = builder.Build();
app.MapGrpcService<MarketDataService>();
app.Run();

// Implementation
public class MarketDataService(IQuoteFeed feed, ILogger<MarketDataService> logger)
    : MarketData.MarketDataBase
{
    public override async Task SubscribeQuotes(
        SubscribeRequest request,
        IServerStreamWriter<Quote> responseStream,
        ServerCallContext context)
    {
        await foreach (var quote in feed.SubscribeAsync(request.Symbols, context.CancellationToken))
        {
            await responseStream.WriteAsync(new Quote
            {
                Symbol = quote.Symbol,
                Bid = quote.Bid,
                Ask = quote.Ask,
                Timestamp = quote.Timestamp.ToUnixTimeMilliseconds(),
            });
        }
    }

    public override async Task<PlaceOrderResponse> PlaceOrder(
        PlaceOrderRequest request,
        ServerCallContext context)
    {
        var deadline = context.Deadline; // от клиента — учитывай!
        var ct = context.CancellationToken;

        var result = await orderEngine.PlaceAsync(request, ct);
        return new PlaceOrderResponse
        {
            OrderId = result.Id,
            Status = result.Status switch
            {
                Engine.Status.Filled => OrderStatus.Filled,
                Engine.Status.Rejected => OrderStatus.Rejected,
                _ => OrderStatus.Pending,
            },
        };
    }
}
```

### Client

```csharp
using var channel = GrpcChannel.ForAddress("https://market-feed:5001");
var client = new MarketData.MarketDataClient(channel);

// Server-streaming
using var call = client.SubscribeQuotes(new SubscribeRequest
{
    Symbols = { "EURUSD", "USDCNH" },
});

await foreach (var quote in call.ResponseStream.ReadAllAsync(ct))
{
    Console.WriteLine($"{quote.Symbol}: {quote.Bid}/{quote.Ask}");
}

// Unary
var response = await client.PlaceOrderAsync(new PlaceOrderRequest
{
    Symbol = "EURUSD",
    Quantity = 100,
    Price = 1.0850,
}, deadline: DateTime.UtcNow.AddSeconds(5));
```

### gRPC через Unix Domain Sockets (локальный IPC)

```csharp
// Server
builder.WebHost.ConfigureKestrel(options =>
{
    options.ListenUnixSocket("/tmp/trading.sock", listenOptions =>
    {
        listenOptions.Protocols = HttpProtocols.Http2;
    });
});

// Client
var endpoint = new UnixDomainSocketEndPoint("/tmp/trading.sock");
var connectionFactory = new SocketsHttpHandler
{
    ConnectCallback = async (ctx, ct) =>
    {
        var socket = new Socket(AddressFamily.Unix, SocketType.Stream, ProtocolType.Unspecified);
        await socket.ConnectAsync(endpoint, ct);
        return new NetworkStream(socket, ownsSocket: true);
    },
};

var channel = GrpcChannel.ForAddress("http://localhost", new GrpcChannelOptions
{
    HttpHandler = connectionFactory,
});
```

Это даёт **производительность Unix-domain socket'а** (быстрее loopback TCP) при сохранении gRPC API.

### Bidi streaming

```csharp
// Client side
using var call = client.Stream();

var sendTask = Task.Run(async () =>
{
    foreach (var update in pendingUpdates)
    {
        await call.RequestStream.WriteAsync(update);
    }
    await call.RequestStream.CompleteAsync();
});

await foreach (var result in call.ResponseStream.ReadAllAsync(ct))
{
    HandleResult(result);
}

await sendTask;
```

### Deadlines, retry, error handling

```csharp
// Deadline через CallOptions
var response = await client.PlaceOrderAsync(request, deadline: DateTime.UtcNow.AddSeconds(2));

// Retry policy через config
var channel = GrpcChannel.ForAddress("https://...", new GrpcChannelOptions
{
    ServiceConfig = new ServiceConfig
    {
        MethodConfigs =
        {
            new MethodConfig
            {
                Names = { MethodName.Default },
                RetryPolicy = new RetryPolicy
                {
                    MaxAttempts = 3,
                    InitialBackoff = TimeSpan.FromMilliseconds(100),
                    MaxBackoff = TimeSpan.FromSeconds(2),
                    BackoffMultiplier = 2,
                    RetryableStatusCodes = { StatusCode.Unavailable, StatusCode.DeadlineExceeded },
                },
            },
        },
    },
});

// Catch
try { await client.PlaceOrderAsync(request); }
catch (RpcException ex) when (ex.StatusCode == StatusCode.Unavailable)
{
    // Сервис недоступен
}
```

> [!question]- **Интервью: когда выбрать gRPC, а когда REST?**
> **gRPC** — между сервисами (S2S), бинарный протокол, контракт через `.proto`, поддержка streaming, низкая латентность. Минусы — браузеры не поддерживают напрямую (нужен gRPC-Web), сложнее дебагить (binary, не curl).
>
> **REST/JSON** — внешние API для веба, мобильных, third-party интеграций; простота отладки и кэширования через стандартные HTTP-инструменты.
>
> Гибрид — внутри инфры всё gRPC, наружу — REST/GraphQL gateway.

> [!question]- **Интервью: как gRPC работает поверх HTTP/2?**
> HTTP/2 даёт мультиплексирование (несколько запросов параллельно по одному соединению без head-of-line blocking) и server push. gRPC использует HTTP/2 streams для каждого RPC: unary = 1 request frame + 1 response frame, server-streaming = 1 request + N response frames, и т.д. Сообщения сериализуются в Protobuf и заворачиваются в HTTP/2 DATA frames с префиксом длины.

---

## Decision matrix — когда что выбирать

| Запрос | Pick |
|--------|------|
| 1 .NET процесс другому, локально, structured commands | **Named Pipes** |
| Один parent запускает короткоживущий child | **Anonymous Pipes** |
| Поток market data 100k сообщений/сек одному consumer | **MMF + ring buffer** |
| Тот же поток + N consumers с back-pressure | **Localhost gRPC server-streaming** |
| Между микросервисами в Docker | **gRPC (TCP)** |
| Cross-language (C# ↔ Python ↔ Rust) | **gRPC (proto)** |
| Из browser/JS клиента | **REST или SignalR (для streaming)** |
| Async messaging "fire and forget" между сервисами | **RabbitMQ / MassTransit** |
| Pub-Sub в реалтайме одной машины | **Channels + worker (внутри процесса)** или **Redis** (между процессами) |

---

## Performance benchmarks

Замеры на одной машине (Windows 11, .NET 10, Intel i7-12700H), latency one-way **примерные** значения:

| Транспорт | Median latency | P99 latency | Throughput (msg/s) |
|-----------|---------------|-------------|--------------------|
| Memory-Mapped File ring | 50ns | 200ns | 50M+ |
| Named Pipe (binary) | 8µs | 30µs | 200K |
| Unix Domain Socket | 6µs | 25µs | 250K |
| Localhost TCP | 15µs | 60µs | 150K |
| gRPC localhost (HTTP/2) | 100µs | 400µs | 30K |
| gRPC over UDS | 80µs | 300µs | 40K |

**Числа для контекста, не для копирования:** реальные значения зависят от размера сообщения, нагрузки CPU, GC paus. Для своей задачи всегда меряй BenchmarkDotNet'ом.

---

## Common pitfalls

### 1. Забытый Flush на StreamWriter

```csharp
// ❌ BAD — без AutoFlush клиент не получит сообщение пока буфер не заполнится
using var writer = new StreamWriter(pipe, Encoding.UTF8);
await writer.WriteLineAsync("GO");

// ✅ GOOD
using var writer = new StreamWriter(pipe, Encoding.UTF8) { AutoFlush = true };
```

### 2. Неправильный transmission mode
`PipeTransmissionMode.Message` — Windows-only, на Linux throw. Для cross-platform всегда `Byte` + framing руками (длина + payload или JSON line-delimited).

### 3. Утечка handles при exception

```csharp
// ❌ BAD — если ProcessAsync кинет, pipe не задиспоузится
var pipe = new NamedPipeServerStream(...);
await pipe.WaitForConnectionAsync();
await ProcessAsync(pipe);
pipe.Dispose();

// ✅ GOOD
await using var pipe = new NamedPipeServerStream(...);
await pipe.WaitForConnectionAsync();
await ProcessAsync(pipe);
```

### 4. MMF без явной синхронизации
Два процесса пишут в одну ячейку без CAS/Mutex — race condition, повреждённые данные. Всегда либо SPSC паттерн, либо явная синхронизация (`Mutex`, `Interlocked`).

### 5. gRPC без deadline

```csharp
// ❌ BAD — RPC может висеть бесконечно
var response = await client.PlaceOrderAsync(request);

// ✅ GOOD — deadline пробрасывается через всю цепочку RPC, защищает всю систему
var response = await client.PlaceOrderAsync(request, deadline: DateTime.UtcNow.AddSeconds(5));
```

### 6. Named Pipe security hole
Сервер на default security и слушает на хорошо известном имени → любой пользователь может подключиться. Всегда настраивай ACL или используй непредсказуемое имя (Guid).

### 7. Передача больших объектов через Named Pipe
Pipe имеет ограниченный буфер (~4-64 KB). Большие сериализованные объекты режутся на чанки и могут блокировать producer. Для больших payloads — MMF или gRPC streaming.

### 8. `GetAwaiter().GetResult()` в Dispose() pipe-сервера
Если в `Dispose` синхронно вызвать `await pipe.DisposeAsync().GetAwaiter().GetResult()` на Sync Context (UI/ASP.NET), — deadlock. Используй `IAsyncDisposable` и `await using`.

---

## Best Practices

### Best Practices for IPC

- **Same machine, low latency** — Named Pipes
- **Cross-platform same machine** — Unix domain sockets (.NET 7+) или TCP localhost
- **Cross-machine, structured** — gRPC (HTTP/2 + protobuf)
- **Cross-machine, simple** — REST/JSON
- **Async / decoupled** — Message broker (RabbitMQ, Kafka)
- **Streaming data** — gRPC bidirectional streams
- **Real-time browser** — SignalR / WebSockets
- **Service mesh integration** — gRPC + mTLS

**Performance hierarchy (faster → slower):**
1. Named Pipes / Unix sockets — < 1 ms
2. gRPC HTTP/2 — 1-5 ms
3. REST HTTP/1.1 — 5-20 ms
4. Message queue (durable) — 10-100 ms
5. WebSocket roundtrip — 5-50 ms

**Choosing protocol:**
- Internal service-to-service → gRPC
- Public API → REST + OpenAPI
- Real-time → WebSockets / SignalR
- Async / events → Kafka / RabbitMQ
- Same machine → Named pipes (Windows) / Unix sockets

См. [[messaging|Messaging]] и[[api-design|API Design]].


---

## Cheat sheet

| Protocol | Use case |
|----------|----------|
| Named Pipes | Same machine IPC, Windows |
| Unix Domain Sockets | Same machine, Unix |
| TCP localhost | Same machine, cross-platform |
| HTTP/REST | Public API, web friendly |
| gRPC | Service-to-service, structured, fast |
| WebSocket | Real-time bidirectional |
| Message Queue | Async decoupled |
| Shared Memory | Ultra-low latency, same process group |

| Need | Solution |
|------|----------|
| Synchronous request-reply | gRPC unary, REST |
| Async fire-and-forget | Message queue |
| Streaming | gRPC streaming, SignalR |
| Pub/sub | Kafka, Redis pub/sub |
| Request-reply async | Queue + correlation ID |
| Long-polling | HTTP long-polling |
| Service discovery | DNS-based (k8s) или Consul |


---

## Decision tree

```
Какой communication protocol?
│
├── Same machine / process boundary?
│   ├── Windows native → Named Pipes
│   ├── Cross-platform → Unix Domain Sockets (.NET 7+)
│   └── Простота → TCP localhost
│
├── Service-to-service network?
│   ├── Internal cluster → gRPC (HTTP/2, protobuf)
│   ├── Mixed languages → gRPC (universal)
│   └── REST если уже existing
│
├── Public API?
│   ├── Web / mobile → REST + OpenAPI
│   ├── Real-time → SignalR / WebSocket
│   └── Specific use → GraphQL
│
├── Async / decoupled?
│   ├── Reliable, durable → RabbitMQ / Azure Service Bus
│   ├── Stream processing → Kafka / Event Hubs
│   ├── Simple pub/sub → Redis pub/sub
│   └── Event sourcing → Event store
│
├── Streaming data?
│   ├── Server → client → gRPC server streaming, SignalR
│   ├── Client → server → gRPC client streaming
│   ├── Bidirectional → gRPC bidi, WebSockets
│   └── Big files → HTTP chunked transfer
│
└── Cross-domain frontend?
    └── REST + CORS (browser limitation)
```


---

## См. также

- [[async-threading|Async и Threading]] — `IAsyncEnumerable`, `CancellationToken`, ThreadPool, deadlocks
- [[hft-low-latency|HFT / Low-Latency]] — где IPC встречается с latency-critical паттернами
- [[concurrency-atomics|Concurrency и Atomics]] — Volatile, CAS, memory barriers (нужны для MMF ring)
- [[span-layout|Span, Memory, Layout]] — `StructLayout`, `unsafe`, fixed pointers (для MMF структур)
- [[messaging|Messaging]]) — RabbitMQ/MassTransit для async IPC между сервисами
- [[resilience|Resilience и HttpClient]] — паттерны retry/timeout также применимы к gRPC

## Reading list

- **Microsoft Docs — Pipes** — learn.microsoft.com/dotnet/standard/io/pipe-operations
- **gRPC for .NET docs** — learn.microsoft.com/aspnet/core/grpc
- **High Performance .NET** (Konrad Kokosa) — глава про межпроцессное взаимодействие и memory-mapped IO
- **Win32 API Named Pipes** — для глубокого понимания (mode bits, instance limits, ACL)
- **LMAX Disruptor whitepaper** — паттерны single-writer-principle, применимые к MMF ring buffers
