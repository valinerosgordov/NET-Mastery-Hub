---
tags: [hft, low-latency, performance, lock-free, channels, span, allocation-free, mt5]
level: Senior
---

# HFT и Low-Latency .NET — production guide

## Что это, зачем и когда

### Что такое low-latency код?
**Код, в котором время отклика измеряется в микросекундах**, а не миллисекундах. Hot-path вызывается миллионы раз в секунду, и каждая аллокация / lock / context-switch — это потерянная сделка, дроп в игре, заикание звука.

**Аналогия:** Обычный backend — это городской автобус с расписанием раз в 15 минут. Low-latency — это спортивная машина, где задержка на старте 0.5 сек = проигрыш гонки.

### Зачем

| Не-HFT задача | HFT-задача |
|---------------|------------|
| API отвечает за 100ms — пользователь не заметит | Сделку взяли на 50µs позже — потеря 50 пунктов = -$5000 |
| GC пауза 200ms на запросе | GC пауза 200ms = пропущено 200 тиков рынка = ликвидация |
| Аллокация на каждый запрос — норма | Аллокация в hot-loop = 10x slowdown |
| Async/await везде | Async overhead — лишняя state machine на пути |

### Где встречается

| Домен | Latency budget | Жёсткость |
|-------|---------------|-----------|
| HFT trading | 10-100µs end-to-end | Жёсткая (deal или smerk) |
| Game server tick | 16ms (60 FPS) | Soft real-time |
| Audio processing | 5-10ms | Hard (гарантия буфера) |
| Network proxy / packet shaper | 50-500µs per packet | Жёсткая |
| Real-time bidding (RTB) | 100ms (включая сеть) | Soft, штрафы |
| Industrial control | 1-10ms | Safety-critical |

### Когда **не** надо

99% backend-кода **не** требует low-latency. Внутри REST API на 100 RPS — оптимизация Span/lock-free ничего не даст. Преждевременная оптимизация — потеря читаемости, time-to-market и надёжности. Сначала измеряй (BenchmarkDotNet), потом оптимизируй.

---

## Latency budget — куда уходит время

```
Operation                       Approximate cost
─────────────────────────────────────────────────
L1 cache hit                    ~1 ns
L2 cache hit                    ~4 ns
L3 cache hit                    ~12 ns
DRAM access (cache miss)        ~100 ns
Branch misprediction            ~5-15 ns
Context switch                  ~1-5 µs
ThreadPool task scheduling      ~1-10 µs
Heap allocation (small)         ~50-500 ns
GC Gen0 collection              ~1-5 ms
GC Gen2 (LOH) collection        ~10-100 ms
Network packet (loopback)       ~10-50 µs
Network packet (LAN)            ~100 µs - 1 ms
SQL query (local Postgres)      ~1-10 ms
Disk SSD read                   ~50-150 µs
Disk HDD seek                   ~5-15 ms
```

**Главные враги latency-кода в .NET:**

1. **Аллокация в hot-path** → давление на GC → паузы
2. **Lock contention** → сериализация потоков → context switches
3. **Async/await overhead** → state machine + heap allocation для Task
4. **Boxing** → struct → object → heap
5. **Cache misses** → плохая memory layout структур
6. **JIT при первом вызове** → замедление первых N итераций
7. **Tiered compilation** → первая фаза компиляции медленнее

---

## Allocation-free patterns

### Span\<T\> и stackalloc

`Span<T>` — окно в непрерывную память (массив, stackalloc, unmanaged). `ref struct` — живёт только на стеке, не может попасть в heap, async, lambda.

```csharp
// ❌ BAD — выделение нового массива на каждый вызов в hot-path
public bool ParseTick(string line)
{
    var parts = line.Split(';');  // string[] на heap каждый раз
    return decimal.TryParse(parts[1], out _);
}

// ✅ GOOD — zero-alloc парсинг через Span
public bool ParseTick(ReadOnlySpan<char> line)
{
    var first = line.IndexOf(';');
    if (first < 0) return false;

    var rest = line[(first + 1)..];
    var second = rest.IndexOf(';');
    if (second < 0) return false;

    return decimal.TryParse(rest[..second], CultureInfo.InvariantCulture, out _);
}
```

### stackalloc для временного буфера

```csharp
// ❌ BAD — небольшой массив на heap
public string FormatTick(double bid, double ask)
{
    var buffer = new char[64];
    var written = ((bid + ask) / 2).TryFormat(buffer, out var len, "F5");
    return new string(buffer, 0, len);
}

// ✅ GOOD — stackalloc + ReadOnlySpan
public string FormatTick(double bid, double ask)
{
    Span<char> buffer = stackalloc char[64];
    if (((bid + ask) / 2).TryFormat(buffer, out var len, "F5", CultureInfo.InvariantCulture))
        return new string(buffer[..len]);

    return string.Empty;
}
```

**Правило:** stackalloc только для **известно маленьких** буферов (< 1KB). Большие — через `ArrayPool<T>`.

### ArrayPool\<T\> для переменного размера

```csharp
// Hot-path обработка тика с динамическим буфером
public void ProcessBatch(int tickCount)
{
    var pool = ArrayPool<MarketTick>.Shared;
    var buffer = pool.Rent(tickCount);

    try
    {
        for (var i = 0; i < tickCount; i++)
            buffer[i] = _feed.ReadTick();

        Process(buffer.AsSpan(0, tickCount));
    }
    finally
    {
        pool.Return(buffer, clearArray: false); // clearArray для типов с GC-references
    }
}
```

**Важно:** `Return(buffer)` обязательно, даже на exception path — `try/finally`. Иначе пул "теряет" массив, и следующий `Rent` выделяет новый.

### struct vs class на hot-path

```csharp
// Если структура < 16 байт и часто копируется — struct
public readonly struct Tick(long timestamp, double bid, double ask)
{
    public long Timestamp { get; } = timestamp;
    public double Bid { get; } = bid;
    public double Ask { get; } = ask;
    public double Mid => (Bid + Ask) / 2;
}

// Передача ref — без копирования
public void Process(ref readonly Tick tick) { /* ... */ }
public void Process(in Tick tick) { /* same as ref readonly */ }

// ❌ Boxing убьёт перформанс
List<object> ticks = new();
ticks.Add(tick); // BOXING — копия struct в heap

// ✅ Generic коллекция без boxing
List<Tick> ticks = new();
ticks.Add(tick);
```

> [!question]- **Интервью: когда использовать struct, когда class?**
> **Struct:**
> - Маленький размер (< 16-24 байта по эмпирике, до 32 — терпимо)
> - Immutable / readonly
> - Часто копируется без логики lifetime
> - Hot path где аллокации недопустимы
>
> **Class:**
> - Большой размер
> - Identity matters (две сущности с одинаковыми полями ≠ одна)
> - Lifecycle важен (нужно явное управление, finalizer)
> - Полиморфизм
>
> Большой struct (> 32 байт), который часто передаётся, на самом деле может быть **медленнее** class — потому что копируется целиком. Меряй BenchmarkDotNet'ом.

### `ref struct` для Span-like типов

```csharp
public ref struct ParsingState
{
    public ReadOnlySpan<char> Remaining;
    public int Pos;

    public bool TryReadField(char delimiter, out ReadOnlySpan<char> field)
    {
        var idx = Remaining.IndexOf(delimiter);
        if (idx < 0) { field = default; return false; }

        field = Remaining[..idx];
        Remaining = Remaining[(idx + 1)..];
        Pos += idx + 1;
        return true;
    }
}

// Использование (синхронно — ref struct нельзя в async)
var state = new ParsingState { Remaining = line, Pos = 0 };
if (state.TryReadField(';', out var date)) { /* ... */ }
```

---

## Lock-free структуры

### Interlocked — atomic-операции

```csharp
private long _totalProcessed;

public void OnTick()
{
    Interlocked.Increment(ref _totalProcessed);
}

public long GetCount() => Interlocked.Read(ref _totalProcessed);

// CAS — Compare and Swap
public bool TryUpdateBest(double newPrice)
{
    var current = Volatile.Read(ref _bestPrice);
    while (newPrice > current)
    {
        var original = Interlocked.CompareExchange(ref _bestPrice, newPrice, current);
        if (original == current) return true;  // успех
        current = original;                     // ретрай с новым значением
    }
    return false;
}
```

### ConcurrentQueue\<T\> для MPSC

`ConcurrentQueue` — lock-free для multi-producer single-consumer (или MPMC, но с большими накладными расходами).

```csharp
private readonly ConcurrentQueue<MarketTick> _queue = new();

// Producer (любой поток)
_queue.Enqueue(tick);

// Single consumer
while (_queue.TryDequeue(out var tick))
{
    Process(tick);
}
```

**Для MPSC лучше — `Channel<T>`** (об этом ниже).

### Single-writer principle

Самая дешёвая «синхронизация» — её отсутствие. Спроектируй систему так, чтобы каждое поле имело **ровно одного писателя**:

```csharp
// Pattern: один поток парсит фид, пишет в ring → другой поток только читает
public sealed class FeedProcessor
{
    private long _writeIndex;  // только feed-thread пишет
    private long _readIndex;   // только consumer-thread пишет
    private readonly Tick[] _buffer = new Tick[1024];

    // Вызывается только из feed-thread
    public void Push(in Tick t)
    {
        _buffer[_writeIndex & 1023] = t;
        Volatile.Write(ref _writeIndex, _writeIndex + 1);  // release fence
    }

    // Вызывается только из consumer-thread
    public bool TryPop(out Tick t)
    {
        if (_readIndex >= Volatile.Read(ref _writeIndex))
        {
            t = default; return false;
        }
        t = _buffer[_readIndex & 1023];
        _readIndex++;
        return true;
    }
}
```

Это база **LMAX Disruptor pattern** — высокопроизводительные ring buffers с одним писателем на каждый sequence.

> [!question]- **Интервью: что такое LMAX Disruptor?**
> Паттерн lock-free очереди, где каждый поток имеет отдельный sequence-counter, и они координируются через memory barriers вместо локов. Изначально из LMAX Exchange (трейдинг-биржа Лондона), теперь применяется в Aeron, Kafka, и многих low-latency .NET приложениях.
> Ключевые идеи: pre-allocated ring buffer (нет аллокаций после старта), single-writer-principle (каждое поле — один писатель), batch processing (consumer обрабатывает несколько событий сразу), memory padding (избегаем false sharing между sequence-counters).

### Memory barriers — Volatile, Interlocked, lock

| Construct | Что делает | Cost |
|-----------|-----------|------|
| `Volatile.Read/Write` | Acquire/release fence на одном поле | Очень дёшево |
| `Interlocked.*` | Atomic + full fence | Дёшево (но дороже Volatile) |
| `lock` (Monitor.Enter/Exit) | Mutex + fence | Дороже всего |
| `Thread.MemoryBarrier()` | Explicit full fence | Редко нужно |

Правило: для single-field publication используй `Volatile`. Для счётчиков и CAS — `Interlocked`. Для блока кода с несколькими полями — `lock`. **Никогда не используй `volatile` keyword** — он семантически слабее `Volatile.Read/Write`.

---

## Channels — современный MPSC/MPMC в .NET

`System.Threading.Channels` — это **рекомендуемый** producer-consumer паттерн в современном .NET. Замена `BlockingCollection` и более эффективная альтернатива `ConcurrentQueue` для MPSC.

### Bounded vs Unbounded

```csharp
// Unbounded — без лимита (опасно для потоковой обработки)
var channel = Channel.CreateUnbounded<MarketTick>(new UnboundedChannelOptions
{
    SingleReader = true,
    SingleWriter = false,
    AllowSynchronousContinuations = false,
});

// Bounded — back-pressure через blocking writer
var bounded = Channel.CreateBounded<MarketTick>(new BoundedChannelOptions(capacity: 10_000)
{
    SingleReader = true,
    SingleWriter = false,
    FullMode = BoundedChannelFullMode.Wait,  // или DropOldest, DropNewest, DropWrite
});
```

### Producer-consumer через Channels

```csharp
// Producer
public async Task ProduceAsync(CancellationToken ct)
{
    while (!ct.IsCancellationRequested)
    {
        var tick = await ReadFeedAsync(ct);
        await _channel.Writer.WriteAsync(tick, ct);
    }
    _channel.Writer.Complete();
}

// Consumer
public async Task ConsumeAsync(CancellationToken ct)
{
    await foreach (var tick in _channel.Reader.ReadAllAsync(ct))
    {
        Process(tick);
    }
}
```

### Channel.WaitToReadAsync для batch processing

```csharp
public async Task ConsumeBatchAsync(CancellationToken ct)
{
    var buffer = new List<Tick>(capacity: 1024);

    while (await _channel.Reader.WaitToReadAsync(ct))
    {
        buffer.Clear();
        while (buffer.Count < 1024 && _channel.Reader.TryRead(out var tick))
            buffer.Add(tick);

        ProcessBatch(buffer);
    }
}
```

**Batch processing драматически снижает накладные расходы** в high-throughput системах — обрабатывая 1000 элементов одновременно, амортизируешь cost вызова async-метода и переключения потока.

> [!question]- **Интервью: чем `Channel<T>` лучше `BlockingCollection<T>`?**
> 1. **Async-first** — `WriteAsync`/`ReadAsync` без блокировки потока (BC использует Wait, занимает поток ThreadPool)
> 2. **Backpressure** — bounded channel дёшево пушит back-pressure через async, без захвата thread'а
> 3. **Меньше аллокаций** — внутренние оптимизации под SingleReader/SingleWriter
> 4. **Cancellation native** — пробрасывается через `CancellationToken` в каждом методе
> 5. **`ReadAllAsync` + `await foreach`** — натуральный API
> BlockingCollection — legacy, сохраняем для совместимости.

---

## Channels vs Pipelines

`System.IO.Pipelines` — низкоуровневый byte-stream API для парсеров (HTTP, ProtoBuf, FAST, FIX). Решает проблему "у меня поток байт переменной длины, нужно распарсить в записи без копирования".

### Когда что

| | `Channels<T>` | `Pipelines` |
|--|---------------|-------------|
| Что передаёт | Готовые объекты `T` | Сырые байты `ReadOnlySequence<byte>` |
| Use case | Producer-consumer внутри сервиса | TCP/UDP/Socket parsers, frame extraction |
| Backpressure | Bounded channel + async wait | `PipeWriter.FlushAsync` блокирует если consumer отстаёт |
| Сложность | Низкая | Средняя |

### Pipelines — пример парсера длино-префиксного фрейма

```csharp
public async Task ProcessFeedAsync(Stream stream, CancellationToken ct)
{
    var reader = PipeReader.Create(stream);

    try
    {
        while (true)
        {
            var result = await reader.ReadAsync(ct);
            var buffer = result.Buffer;

            while (TryParseFrame(ref buffer, out var frame))
            {
                ProcessFrame(frame);
            }

            // Сообщаем PipeReader сколько мы уже обработали
            reader.AdvanceTo(consumed: buffer.Start, examined: buffer.End);

            if (result.IsCompleted) break;
        }
    }
    finally
    {
        await reader.CompleteAsync();
    }
}

private bool TryParseFrame(ref ReadOnlySequence<byte> buffer, out ReadOnlySequence<byte> frame)
{
    if (buffer.Length < 4) { frame = default; return false; }

    // Длина в первых 4 байтах
    Span<byte> lengthBytes = stackalloc byte[4];
    buffer.Slice(0, 4).CopyTo(lengthBytes);
    var length = BinaryPrimitives.ReadInt32LittleEndian(lengthBytes);

    if (buffer.Length < 4 + length) { frame = default; return false; }

    frame = buffer.Slice(4, length);
    buffer = buffer.Slice(4 + length);
    return true;
}
```

`ReadOnlySequence<byte>` — может быть **многосегментной** (когда payload пришёл по частям). API Pipelines справляется с этим без копирования и сборки в один массив. Это даёт zero-copy парсинг даже на TCP-потоках.

---

## Async / await — overhead и ловушки

### Async overhead

Каждый `async` метод — это сгенерированная компилятором state machine. На "happy path" (метод завершается синхронно) — overhead минимален, но всё равно есть. Если метод вызывается миллион раз в секунду — overhead заметен.

```csharp
// 1. Регулярный async — state machine, выделение Task на heap
public async Task<int> GetAsync()
{
    await Task.Delay(1);
    return 42;
}

// 2. ValueTask — экономит heap allocation на synchronous-completion path
public ValueTask<int> GetAsync()
{
    if (_cache.TryGet(out var value))
        return ValueTask.FromResult(value);  // synchronous, no alloc

    return GetSlowAsync();
}

private async ValueTask<int> GetSlowAsync()
{
    var v = await ComputeAsync();
    _cache.Set(v);
    return v;
}
```

> [!question]- **Интервью: когда использовать `ValueTask` вместо `Task`?**
> Когда метод **часто завершается синхронно**: cache hit, fast-path, optimistic check. ValueTask — это struct, нет heap alloc на synchronous completion.
> Ограничения:
> 1. Нельзя await дважды (`task1; task2;` от одного ValueTask) — undefined behavior
> 2. Нельзя `Task.WhenAll(valueTask1, valueTask2)` — нужно конвертить в `Task` через `.AsTask()`
> 3. Нельзя кэшировать в поле или передавать долго — должен быть await'нут или преобразован в Task
>
> Правило: возвращай `ValueTask` из методов с частым sync-completion path (cache, hot-path). Для "обычных" async методов — `Task`.

### `GetAwaiter().GetResult()` — антипаттерн

Это синхронное ожидание async-метода. **Никогда** не делай этого в SyncContext (UI-thread, ASP.NET classic) — гарантированный deadlock. В ASP.NET Core (без SyncContext) — не deadlock, но ThreadPool starvation: захватывает поток на N секунд, пока async ждёт.

```csharp
// ❌ BAD — это шёл прямо в твой TradingBotForex CRIT-04b/c
public void Dispose()
{
    _client.DisposeAsync().GetAwaiter().GetResult();
}

// ✅ GOOD — IAsyncDisposable
public async ValueTask DisposeAsync()
{
    await _client.DisposeAsync();
}

// Использование:
await using var bot = new TradingBot(...);
```

> [!question]- **Интервью: почему `.Result` / `.Wait()` / `.GetAwaiter().GetResult()` — антипаттерн?**
> 1. **Deadlock на SyncContext** — UI/legacy ASP.NET захватывает SyncContext на основном потоке, async-continuation хочет вернуться туда же, основной поток ждёт continuation → deadlock
> 2. **ThreadPool starvation** — даже без SyncContext, синхронное ожидание занимает поток ThreadPool, который мог бы делать другую работу. Каскад → весь ThreadPool занят, новые запросы стоят в очереди
> 3. **AggregateException** — если внутри упало, ты получишь `AggregateException` вместо настоящего исключения, теряешь stack trace
>
> Решение: всё async вверх по стеку (`async all the way`), Dispose через `IAsyncDisposable`, в крайнем случае `Task.Run(...).GetAwaiter().GetResult()` (запуск на ThreadPool отдельно от текущего контекста).

### Async void — bug

```csharp
// ❌ ВСЕГДА BAD (кроме event handler'ов)
public async void DoBackgroundWork() { /* exception здесь убьёт процесс */ }

// ✅ Fire-and-forget с правильной обработкой
public Task DoBackgroundWorkAsync()  =>
    Task.Run(async () =>
    {
        try { await WorkAsync(); }
        catch (Exception ex) { _logger.LogError(ex, "Background work failed"); }
    });

// ✅ Event handler — единственный валидный async void
public async void Button_Click(object sender, EventArgs e)
{
    try { await SaveAsync(); }
    catch (Exception ex) { ShowError(ex); }
}
```

---

## ThreadPool tuning и DedicatedThread

### ThreadPool — для general-purpose work

```csharp
// Tune минимальное число потоков (на старте процесса)
ThreadPool.SetMinThreads(workerThreads: 100, completionPortThreads: 100);

// Бенчмарк перед прод: смотри dotnet-counters Threads.ActiveThreads / QueueLength
```

ThreadPool **наращивает потоки лениво** — это режет latency первых N запросов после старта. SetMinThreads(N) форсит N потоков сразу.

### DedicatedThread для critical loop

```csharp
public sealed class MarketDataProcessor
{
    private readonly Thread _thread;
    private readonly CancellationTokenSource _cts = new();

    public MarketDataProcessor()
    {
        _thread = new Thread(Run)
        {
            Name = "MarketData",
            IsBackground = true,
            Priority = ThreadPriority.Highest,
        };
        _thread.Start();
    }

    private void Run()
    {
        // Pin поток к конкретному CPU (Windows только)
        var thread = ProcessThread.GetCurrentThread();
        thread.IdealProcessor = 4;
        thread.ProcessorAffinity = (IntPtr)(1 << 4);

        var spin = new SpinWait();

        while (!_cts.IsCancellationRequested)
        {
            if (_ringBuffer.TryRead(out var tick))
            {
                spin.Reset();
                Process(tick);
            }
            else
            {
                // SpinWait: первые ~10 итераций — busy-spin, потом yield, потом sleep
                spin.SpinOnce();
            }
        }
    }
}
```

**Когда DedicatedThread, а не ThreadPool:**
- Hot loop, который не должен переключать контекст (один тик 24/7)
- Ниче не возвращать из этого thread'а через async (он сам управляет своим lifecycle)
- Latency-critical (пин на CPU, priority above normal)

`SpinWait.SpinOnce()` — умная стратегия busy-wait: первые ~10 итераций крутимся в CPU (latency низкая), дальше отдаём поток через `Thread.Yield`, в крайнем случае `Thread.Sleep(1)` (избегаем 100% CPU при пустой очереди).

---

## GC tuning для low-latency

### Server GC

```xml
<!-- .csproj -->
<PropertyGroup>
  <ServerGarbageCollection>true</ServerGarbageCollection>
  <ConcurrentGarbageCollection>true</ConcurrentGarbageCollection>
</PropertyGroup>
```

Server GC использует **отдельный heap на каждый CPU** + multi-threaded compaction → пауза в N раз короче чем Workstation GC.

### LatencyMode

Перед критичной операцией — переключи GC в latency-режим, который не делает Gen2 (LOH-сжатие) пока ты в этом блоке:

```csharp
var oldMode = GCSettings.LatencyMode;
GCSettings.LatencyMode = GCLatencyMode.LowLatency;

try
{
    // Critical section
    ProcessHotPath();
}
finally
{
    GCSettings.LatencyMode = oldMode;
}

// Альтернатива — `SustainedLowLatency`: длительный режим, годится для всего hot-path процесса
```

### Avoid LOH

Large Object Heap (≥ 85,000 байт) **не уплотняется** — фрагментация → out-of-memory или долгая Gen2 compaction. Стратегия:
- `ArrayPool<T>.Shared.Rent(N)` — возвращает массив **из пула**, может быть заранее в LOH, но переиспользуется
- Стрижка больших структур на маленькие чанки

```csharp
// ❌ BAD — каждый раз новая аллокация в LOH
var buffer = new byte[200_000];

// ✅ GOOD — переиспользование одного буфера из пула
var buffer = ArrayPool<byte>.Shared.Rent(200_000);
try { /* use buffer */ }
finally { ArrayPool<byte>.Shared.Return(buffer); }
```

### Pinned Object Heap (POH, .NET 5+)

Если объект надо пиновать (передать в native API), пинуй через `GC.AllocateArray<T>(length, pinned: true)` — тогда он попадёт в POH вместо обычного heap. POH не двигается GC, native код не сломается.

```csharp
// Buffer для native interop (например, передача в COM-компонент или unmanaged DLL)
var buffer = GC.AllocateArray<byte>(4096, pinned: true);
// Buffer не будет перемещён GC, можно безопасно отдать его native'у
```

### Profile перед оптимизацией

```bash
# Общие метрики
dotnet-counters monitor System.Runtime --process-id <pid>

# Снимок heap
dotnet-gcdump collect --process-id <pid>

# Trace (CPU + GC + Threading)
dotnet-trace collect --process-id <pid> --providers Microsoft-Windows-DotNETRuntime
```

---

## BenchmarkDotNet — production patterns

```csharp
[MemoryDiagnoser]                                  // показывает аллокации
[ThreadingDiagnoser]                              // contention, lock allocs
[SimpleJob(RuntimeMoniker.Net100)]                // явный фрейм-таргет
[RPlotExporter]                                   // графики через R
public class TickParserBench
{
    private string _line = "2026-04-28T12:34:56.789;EURUSD;1.08501;1.08503";
    private byte[] _bytes = Encoding.UTF8.GetBytes("...");

    [Benchmark(Baseline = true)]
    public bool ParseString_Naive()
    {
        var parts = _line.Split(';');
        return decimal.TryParse(parts[2], CultureInfo.InvariantCulture, out _);
    }

    [Benchmark]
    public bool ParseSpan()
    {
        var span = _line.AsSpan();
        var first = span.IndexOf(';');
        var second = span[(first + 1)..].IndexOf(';');
        var third = span[(first + 1 + second + 1)..].IndexOf(';');

        var bidStart = first + 1 + second + 1;
        var bidEnd = bidStart + third;
        return decimal.TryParse(span[bidStart..bidEnd], CultureInfo.InvariantCulture, out _);
    }

    [Benchmark]
    public bool ParseUtf8()
    {
        var span = _bytes.AsSpan();
        var first = span.IndexOf((byte)';');
        // ... аналогично, но без конверсии в string
        return true;
    }
}
```

Запускай в Release-конфиге, без attached debugger:
```bash
dotnet run -c Release --project Benchmarks
```

> [!question]- **Интервью: как BenchmarkDotNet получает точные замеры?**
> 1. **Прогрев (warmup)** — несколько итераций вне замера, чтобы JIT откомпилил, кэши прогрелись, tiered compilation сделала Tier1
> 2. **Множественные итерации** — десятки/сотни прогонов, статистика (mean, P95, StdDev)
> 3. **Изоляция** — каждый бенч в отдельном процессе, чтобы не было влияния от предыдущих
> 4. **Защита от dead code elimination** — `[Benchmark]` метод должен возвращать значение, иначе JIT может выбросить весь код
> 5. **Защита от inline'инга** — методы помечаются `[MethodImpl(NoInlining)]` если нужно явное измерение

---

## MetaTrader 5 COM Automation — практика

MT5 на Windows экспонирует COM-объект `MetaTraderApi`. Это синхронный thread-affine API — все вызовы должны идти из одного STA-thread'а.

```csharp
[ComImport]
[Guid("...")]
[InterfaceType(ComInterfaceType.InterfaceIsDual)]
public interface IMT5
{
    bool Initialize();
    bool Login(int login, string password, string server);
    bool OrderSend(string symbol, double volume, double price, ...);
    // ...
}

public sealed class MetaTraderConnector : IDisposable
{
    private readonly Thread _staThread;
    private readonly BlockingCollection<Func<Task>> _commands = new();
    private dynamic _mt5;  // late-bound через COM

    public MetaTraderConnector()
    {
        _staThread = new Thread(RunSta);
        _staThread.SetApartmentState(ApartmentState.STA);  // обязательно для COM
        _staThread.IsBackground = true;
        _staThread.Start();
    }

    private void RunSta()
    {
        // Инициализация COM
        Type t = Type.GetTypeFromProgID("MetaTrader.API")!;
        _mt5 = Activator.CreateInstance(t)!;
        _mt5.Initialize();

        // Loop команд из других потоков
        foreach (var cmd in _commands.GetConsumingEnumerable())
        {
            try { cmd().GetAwaiter().GetResult(); }  // ОК тут, мы внутри dedicated STA thread
            catch (Exception ex) { _logger.LogError(ex, "MT5 command failed"); }
        }
    }

    public Task<TResult> ExecuteAsync<TResult>(Func<dynamic, TResult> action)
    {
        var tcs = new TaskCompletionSource<TResult>();
        _commands.Add(() =>
        {
            try { tcs.SetResult(action(_mt5)); }
            catch (Exception ex) { tcs.SetException(ex); }
            return Task.CompletedTask;
        });
        return tcs.Task;
    }

    public void Dispose()
    {
        _commands.CompleteAdding();
        _staThread.Join(TimeSpan.FromSeconds(5));
    }
}

// Использование (любой поток вызывает)
var balance = await connector.ExecuteAsync(mt5 => (double)mt5.AccountInfoDouble(BALANCE));
```

**Ключевые моменты:**
- COM-объект на STA-потоке. Любой другой поток дёргающий COM напрямую = COMException 0x8001010E
- Все команды сериализуются через BlockingCollection → один STA выполняет последовательно
- `GetAwaiter().GetResult()` тут **уместен**, потому что мы внутри dedicated thread, не в SyncContext, и нам нужно последовательное выполнение

---

## FAST/TWIME — краткий обзор

**MOEX TWIME** — двоичный протокол для подачи заявок (latency-critical, sponsored access).
**FAST** — двоичный сжатый протокол для market data feed.

Парсятся через `Span<byte>` + `BinaryPrimitives` без аллокаций:

```csharp
public bool TryParseTwimeHeader(ReadOnlySpan<byte> buffer, out TwimeHeader header)
{
    if (buffer.Length < 12) { header = default; return false; }

    header = new TwimeHeader
    {
        BlockLength = BinaryPrimitives.ReadUInt16LittleEndian(buffer[0..2]),
        TemplateId = BinaryPrimitives.ReadUInt16LittleEndian(buffer[2..4]),
        SchemaId = BinaryPrimitives.ReadUInt16LittleEndian(buffer[4..6]),
        Version = BinaryPrimitives.ReadUInt16LittleEndian(buffer[6..8]),
        SeqNum = BinaryPrimitives.ReadUInt32LittleEndian(buffer[8..12]),
    };
    return true;
}
```

В TradingBotForex уже есть `FastProtocol` (9 тестов) и `TwimeProtocol` (13 тестов) — это ваш хлеб.

---

## Production checklist для latency-critical .NET

- [ ] `<ServerGarbageCollection>true</ServerGarbageCollection>` в `.csproj`
- [ ] `<ConcurrentGarbageCollection>true</ConcurrentGarbageCollection>`
- [ ] `<TieredCompilation>true</TieredCompilation>` (default in .NET 6+)
- [ ] `ThreadPool.SetMinThreads(N, N)` на старте, где N = ожидаемая нагрузка
- [ ] `GCSettings.LatencyMode = SustainedLowLatency` для критичной фазы
- [ ] Hot-path методы помечены `[MethodImpl(MethodImplOptions.AggressiveInlining)]` где осмысленно
- [ ] Все коллекции на старте проинициализированы с `capacity` (избегаем resize)
- [ ] `ArrayPool<T>.Shared` для всех буферов > 1KB
- [ ] `Span<T>` / `stackalloc` для всех временных буферов на стеке
- [ ] Никаких `string.Format`, `+` для строк, `string.Split` в hot-path
- [ ] `Interlocked` / `Volatile` вместо `lock` для одиночных полей
- [ ] `Channel<T>` (или ring buffer) вместо `BlockingCollection`
- [ ] `IAsyncDisposable` вместо sync `Dispose` с `.GetAwaiter().GetResult()`
- [ ] `ConfigureAwait(false)` в библиотечном коде (хотя в new code не критично, ASP.NET Core без SyncContext)
- [ ] BenchmarkDotNet прогон на каждое изменение hot-path
- [ ] Continuous profiling в проде (dotnet-monitor, EventPipe)

---

## Common pitfalls

### 1. LINQ в hot loop

```csharp
// ❌ BAD — каждый Where/Select создаёт enumerator + closure
foreach (var t in ticks.Where(t => t.Volume > 100).Select(t => t.Bid))

// ✅ GOOD
foreach (var t in ticks)
{
    if (t.Volume <= 100) continue;
    var bid = t.Bid;
    // ...
}
```

### 2. Boxing в generic methods

```csharp
// ❌ BAD — boxing каждый вызов
public void Log<T>(T value) where T : struct
{
    _logger.LogInformation("Value: {V}", value); // value boxed to object
}

// ✅ GOOD — Source-generated logging без boxing
[LoggerMessage(Level = LogLevel.Information, Message = "Value: {V}")]
private partial void LogValue<T>(T value) where T : struct;
```

### 3. False sharing
Два часто-обновляемых поля разных потоков попадают в одну CPU cache line (64 байт) → каждое обновление инвалидирует cache у другого ядра.
```csharp
// ❌ Два потока пишут в _writePos и _readPos — оба в одной cache line
private long _writePos;
private long _readPos;

// ✅ Padding
[StructLayout(LayoutKind.Explicit, Size = 192)]  // 64 + 64 + 64 padding
private struct PaddedSequencer
{
    [FieldOffset(64)] public long Value;  // изолировано в свою cache line
}
```

### 4. Регулярки в hot-path
`Regex` компилируется JIT'ом в IL, но всё ещё дороже ручного парсинга через Span. В .NET 7+ — `[GeneratedRegex(...)]` — source generator делает быстрее, но всё равно медленнее Span-парсинга для простых случаев.

### 5. `DateTime.UtcNow` в hot-path
Системный вызов под капотом (~50-100ns). Если нужен timestamp на каждый тик — используй `Stopwatch.GetTimestamp()` (намного быстрее, monotonic).

```csharp
private static readonly long _baseTime = DateTime.UtcNow.Ticks;
private static readonly long _baseStopwatch = Stopwatch.GetTimestamp();

public static DateTime FastUtcNow()
{
    var elapsed = Stopwatch.GetTimestamp() - _baseStopwatch;
    var elapsedTicks = elapsed * 10_000_000 / Stopwatch.Frequency;
    return new DateTime(_baseTime + elapsedTicks);
}
```

### 6. Configure logger с `LogLevel.Trace` в проде
Каждая `_logger.LogTrace(...)` — это вызов с подсчётом аргументов даже если в итоге не пишется. Используй `LoggerMessage` source-generator или гард `if (logger.IsEnabled(LogLevel.Trace))`.

---

## См. также

- [Span, Memory, Layout]() — глубже про Span/Memory/StructLayout
- [Concurrency и Atomics]() — Volatile, CAS, memory barriers
- [GC, LOH и POH]() — поколения, фрагментация, finalization
- [Async и Threading]() — async overhead, ConfigureAwait, Channel
- [IPC]() — MMF ring-buffer для market data feed
- [Performance](performance.md) — BenchmarkDotNet, profiling
- [WPF Production]() — UI thread vs hot-path threads

## Reading list

- **Pro .NET Memory Management** — Konrad Kokosa (главы про GC tuning, struct layout, cache lines)
- **Concurrency in .NET** — Riccardo Terrell (lock-free, MailboxProcessor, Hopac)
- **LMAX Disruptor** — github.com/LMAX-Exchange/disruptor (whitepaper + .NET ports)
- **High Performance Browser Networking** — Ilya Grigorik (применимо к gRPC/HTTP-2)
- **Stephen Toub blogs** — devblogs.microsoft.com/dotnet/author/toub/ (всё про async и performance)
- **dotnet/runtime issues** — github.com/dotnet/runtime — там обсуждаются low-level perf-фичи .NET
