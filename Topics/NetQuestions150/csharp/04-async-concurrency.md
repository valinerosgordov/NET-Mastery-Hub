# Async и Concurrency

## Task, Task.WhenAll, Task.Run

### Task.WhenAll — параллельное выполнение

```csharp
// ✓ Запуск задач ДО await — выполняются параллельно
var task1 = GetOrdersAsync(ct);
var task2 = GetCustomersAsync(ct);
var task3 = GetProductsAsync(ct);
var (orders, customers, products) = (
    await task1, await task2, await task3);

// Или через WhenAll
await Task.WhenAll(task1, task2, task3);

// ✗ Плохо — последовательное выполнение
var orders = await GetOrdersAsync(ct);      // ждём
var customers = await GetCustomersAsync(ct); // потом ждём
```

**Task.WhenAny** — первый завершённый. Полезно для timeout или racing:

```csharp
var completed = await Task.WhenAny(dataTask, Task.Delay(TimeSpan.FromSeconds(5), ct));
if (completed != dataTask) throw new TimeoutException();
var result = await dataTask;
```

### Task.Run vs Task.Factory.StartNew

```csharp
// Task.Run — обёртка ThreadPool.QueueUserWorkItem, для CPU-bound работы
await Task.Run(() => HeavyComputation(), ct);

// Task.Factory.StartNew — больше контроля
await Task.Factory.StartNew(
    () => VeryLongWork(),
    ct,
    TaskCreationOptions.LongRunning,  // выделенный поток, не из пула
    TaskScheduler.Default);
```

**Нюанс:** `Task.Run` в ASP.NET — антипаттерн для I/O-операций. ThreadPool поток блокируется впустую. Использовать нативный async (HttpClient, EF Core).

---

## Async Streams, lock для async

### IAsyncEnumerable&lt;T&gt;

Потоковое чтение без загрузки всего в память:

```csharp
public async IAsyncEnumerable<Order> GetOrdersStreamAsync(
    [EnumeratorCancellation] CancellationToken ct = default)
{
    await foreach (var batch in GetBatchesAsync(ct))
    {
        foreach (var order in batch)
            yield return order;
    }
}

// Потребление
await foreach (var order in GetOrdersStreamAsync(ct))
{
    Process(order); // обработка по одному, без загрузки всех в память
}
```

### Async lock — SemaphoreSlim

```csharp
private readonly SemaphoreSlim _semaphore = new(1, 1);

public async Task<T> GetOrCreateAsync(string key, CancellationToken ct)
{
    await _semaphore.WaitAsync(ct);
    try
    {
        // критическая секция
        return await LoadFromDbAsync(key, ct);
    }
    finally
    {
        _semaphore.Release();
    }
}
```

**Нюанс:** `lock` + `await` внутри — ошибка компиляции. `Monitor` — только синхронный. Для async — `SemaphoreSlim`. Для producer-consumer — `Channel<T>`.

---

## Thread, ThreadPool, BackgroundService

| Механизм | Когда использовать |
|----------|--------------------|
| `Thread` | Почти никогда. Legacy код |
| `ThreadPool.QueueUserWorkItem` | Простая CPU-работа без результата |
| `Task.Run` | CPU-bound с результатом |
| `BackgroundService` | Фоновые процессы в ASP.NET |

```csharp
public class OrderProcessingService(
    Channel<Order> channel,
    IServiceScopeFactory scopeFactory,
    ILogger<OrderProcessingService> logger) : BackgroundService
{
    protected override async Task ExecuteAsync(CancellationToken ct)
    {
        await foreach (var order in channel.Reader.ReadAllAsync(ct))
        {
            using var scope = scopeFactory.CreateScope();
            var handler = scope.ServiceProvider.GetRequiredService<IOrderHandler>();
            try
            {
                await handler.ProcessAsync(order, ct);
            }
            catch (Exception ex)
            {
                logger.LogError(ex, "Failed to process order {OrderId}", order.Id);
            }
        }
    }
}
```

**Нюанс:** `BackgroundService` + `IServiceScopeFactory` — единственный правильный способ работать со Scoped-сервисами (DbContext) в Singleton-фоновом сервисе.

---

## Interlocked, volatile, Synchronization

### Interlocked — атомарные операции

```csharp
private long _counter;

public void Increment()
{
    Interlocked.Increment(ref _counter);       // атомарный i++
}

public bool TrySetOnce(ref int flag)
{
    // Атомарно: если flag == 0, установить 1, вернуть предыдущее
    return Interlocked.CompareExchange(ref flag, 1, 0) == 0;
}
```

### volatile

Запрещает кэширование и переупорядочивание чтений/записей компилятором и CPU. **Не даёт атомарности!**

```csharp
private volatile bool _running = true;

// Один поток:
_running = false; // гарантия: другие потоки увидят изменение

// Другой поток:
while (_running) { /* ... */ }
```

**Нюанс:** для `i++` volatile недостаточно (read-modify-write не атомарно). Использовать `Interlocked.Increment`.

### Events

```csharp
// AutoResetEvent — один поток проходит, автосброс
var auto = new AutoResetEvent(false);
auto.Set();      // один ожидающий поток пройдёт
auto.WaitOne();  // блокировка до Set()

// ManualResetEventSlim — все потоки проходят при Signal
var manual = new ManualResetEventSlim(false);
manual.Set();    // все ожидающие проходят
manual.Reset();  // снова блокирует
```

---

## Channel&lt;T&gt; и GC

### Channel&lt;T&gt; — async producer-consumer

```csharp
// Bounded — ограниченный буфер, backpressure
var channel = Channel.CreateBounded<Order>(new BoundedChannelOptions(100)
{
    FullMode = BoundedChannelFullMode.Wait // блокирует writer при полном канале
});

// Producer
await channel.Writer.WriteAsync(order, ct);
channel.Writer.Complete(); // сигнал: больше данных не будет

// Consumer
await foreach (var item in channel.Reader.ReadAllAsync(ct))
{
    await ProcessAsync(item, ct);
}
```

**Нюанс:** `Channel<T>` — предпочтительнее `BlockingCollection<T>` для async. Lock-free, поддерживает CancellationToken, await.

### GC (Garbage Collector)

| Поколение | Объекты | Частота | Стоимость |
|-----------|---------|---------|-----------|
| Gen0 | Новые, короткоживущие | Часто | Дёшево |
| Gen1 | Пережившие Gen0 | Средне | Средне |
| Gen2 | Долгоживущие | Редко | Дорого |
| LOH | > 85 KB | С Gen2 | Дорого, фрагментация |
| POH (.NET 5+) | Pinned | С Gen2 | Без фрагментации |

```csharp
// Server GC — для ASP.NET (параллельная сборка, больше throughput)
// Workstation GC — для desktop (меньше пауз)
// Настройка в .csproj:
// <ServerGarbageCollection>true</ServerGarbageCollection>

// Как уменьшить давление GC:
// 1. ArrayPool<T>.Shared.Rent/Return — повторное использование массивов
// 2. Span<T> / stackalloc — обработка на стеке
// 3. ObjectPool<T> — повторное использование объектов
// 4. ValueTask<T> — избежание аллокации Task для sync paths
```

**Нюанс:** `GC.Collect()` вызывать вручную — почти никогда не нужно. GC сам знает лучше. Исключение: перед benchmarking или при точном знании освобождения большого графа объектов.

---

## ConfigureAwait и SynchronizationContext

```csharp
// В ASP.NET Core — нет SynchronizationContext, ConfigureAwait(false) не нужен
// В библиотеках — ConfigureAwait(false) обязателен (могут использоваться в UI)
public async Task<int> LibraryMethodAsync()
{
    var data = await httpClient.GetAsync(url).ConfigureAwait(false);
    return Process(data);
}
```

**Нюанс:** `ValueTask<T>` — не аллоцирует Task если результат доступен синхронно. Нельзя await дважды. Нельзя использовать `.Result`. Для hot paths — значительная экономия.

---

## CancellationToken

```csharp
public async Task<Order> GetOrderAsync(Guid id, CancellationToken ct)
{
    // Передавать ct во ВСЕ async вызовы
    var order = await context.Orders.FirstOrDefaultAsync(o => o.Id == id, ct);
    ct.ThrowIfCancellationRequested(); // проверка в CPU-bound участках
    return order ?? throw new NotFoundException(id);
}
```

**Нюанс:** игнорирование `CancellationToken` — один из дорогих anti-pattern в enterprise .NET. Приводит к утечке ресурсов и невозможности отменить операцию при shutdown.

---

## См. также

- [C# Reference: Async и многопоточность](../../../Reference/csharp-async-threading.md)
