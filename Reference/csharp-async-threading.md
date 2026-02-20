# C# — Async, Threading и Concurrency

> Полный справочник по асинхронности, многопоточности и concurrency в C# 13 / .NET 9.
> Каждый концепт с рабочим примером кода.

---

## Основы потоков

### Thread class — создание, запуск, Join

`Thread` — низкоуровневый примитив ОС. Создание потока дорогое (~1 MB стека).

```csharp
// Создание и запуск потока
var thread = new Thread(() =>
{
    Console.WriteLine($"Worker thread: {Environment.CurrentManagedThreadId}");
    Thread.Sleep(1000);
    Console.WriteLine("Worker done");
});

thread.IsBackground = true; // не блокирует завершение процесса
thread.Name = "MyWorker";
thread.Start();

// Ожидание завершения
thread.Join(); // блокирует текущий поток до завершения worker
Console.WriteLine("Main continues after Join");
```

```csharp
// Передача данных в поток
var thread = new Thread((object? state) =>
{
    var data = (string)state!;
    Console.WriteLine($"Received: {data}");
});

thread.Start("Hello from main thread");
thread.Join();
```

### ThreadPool — зачем, как работает

`ThreadPool` — пул переиспользуемых потоков. Избегает overhead создания/уничтожения потоков.

```csharp
// Постановка работы в ThreadPool
ThreadPool.QueueUserWorkItem(state =>
{
    Console.WriteLine($"ThreadPool thread: {Environment.CurrentManagedThreadId}");
});

// Информация о пуле
ThreadPool.GetMinThreads(out int workerMin, out int ioMin);
ThreadPool.GetMaxThreads(out int workerMax, out int ioMax);
Console.WriteLine($"Workers: {workerMin}-{workerMax}, IO: {ioMin}-{ioMax}");

// Настройка (осторожно — влияет на весь процесс)
ThreadPool.SetMinThreads(workerThreads: 8, completionPortThreads: 8);
```

### Thread vs Task — почему Task лучше

| Аспект | Thread | Task |
|---|---|---|
| Создание | Дорогое (~1 MB) | Дешёвое (ThreadPool) |
| Результат | Нет встроенного | `Task<T>` возвращает значение |
| Composition | Сложно | `WhenAll`, `WhenAny`, `ContinueWith` |
| Exceptions | Теряются | Propagate через `await` |
| Cancellation | Ручная (`Abort` deprecated) | `CancellationToken` |
| async/await | Несовместим | Нативная поддержка |

```csharp
// Thread — неудобно получить результат
string? threadResult = null;
var thread = new Thread(() => threadResult = "done");
thread.Start();
thread.Join();
Console.WriteLine(threadResult); // "done"

// Task — элегантно
string taskResult = await Task.Run(() => "done");
Console.WriteLine(taskResult); // "done"
```

**Правило:** используй `Thread` только если нужен dedicated long-running поток (и то — через `Task.Factory.StartNew` с `TaskCreationOptions.LongRunning`).

---

## Task и Task\<T\>

### Создание: Task.Run, Task.Factory.StartNew

```csharp
// Task.Run — основной способ запуска CPU-bound работы
Task task = Task.Run(() => Console.WriteLine("Fire!"));
Task<int> taskWithResult = Task.Run(() =>
{
    Thread.Sleep(100);
    return 42;
});

int result = await taskWithResult;
Console.WriteLine(result); // 42
```

```csharp
// Task.Factory.StartNew — когда нужен контроль
// LongRunning — создаёт отдельный поток (не из пула)
Task longTask = Task.Factory.StartNew(
    () =>
    {
        // Долгая блокирующая операция
        while (true)
        {
            Thread.Sleep(1000);
            Console.WriteLine("Heartbeat");
        }
    },
    CancellationToken.None,
    TaskCreationOptions.LongRunning,
    TaskScheduler.Default);
```

> **Важно:** `Task.Factory.StartNew` не разворачивает вложенные Task. Если lambda возвращает `Task`, получишь `Task<Task>`. Используй `Task.Run` для async lambda.

```csharp
// ОПАСНО — Task<Task<int>>, внешний Task завершается сразу
var bad = Task.Factory.StartNew(async () =>
{
    await Task.Delay(1000);
    return 42;
});
// bad.Result — это Task<int>, а не int!

// ПРАВИЛЬНО
var good = Task.Run(async () =>
{
    await Task.Delay(1000);
    return 42;
});
int value = await good; // 42
```

### Ожидание: await, Wait(), Result (ОПАСНО — deadlock)

```csharp
// await — ЕДИНСТВЕННЫЙ правильный способ
var data = await LoadDataAsync();

// Wait() — БЛОКИРУЕТ поток. Deadlock в UI/ASP.NET с SynchronizationContext
// task.Wait(); // НЕ ДЕЛАЙ ТАК

// .Result — тоже БЛОКИРУЕТ поток
// var x = task.Result; // НЕ ДЕЛАЙ ТАК

// Если ВЫНУЖДЕН вызвать async из sync (legacy код):
// Вариант 1: GetAwaiter().GetResult() — чуть лучше (не оборачивает в AggregateException)
var result = LoadDataAsync().GetAwaiter().GetResult();

// Вариант 2: в отдельном потоке (безопаснее)
var result2 = Task.Run(() => LoadDataAsync()).GetAwaiter().GetResult();
```

### Task.WhenAll — параллельное выполнение

```csharp
// Запуск нескольких задач параллельно
async Task<UserDashboardDto> GetDashboardAsync(int userId, CancellationToken ct)
{
    // Все три запроса идут одновременно
    Task<User> userTask = GetUserAsync(userId, ct);
    Task<List<Order>> ordersTask = GetOrdersAsync(userId, ct);
    Task<decimal> balanceTask = GetBalanceAsync(userId, ct);

    await Task.WhenAll(userTask, ordersTask, balanceTask);

    return new UserDashboardDto
    {
        User = userTask.Result,       // безопасно — задача уже завершена
        Orders = ordersTask.Result,
        Balance = balanceTask.Result
    };
}
```

```csharp
// WhenAll с коллекцией задач
async Task<int[]> ProcessBatchAsync(List<string> urls, CancellationToken ct)
{
    var tasks = urls.Select(url => DownloadSizeAsync(url, ct));
    int[] results = await Task.WhenAll(tasks);
    return results;
}
```

```csharp
// WhenAll с обработкой ошибок — ловит ВСЕ исключения
try
{
    await Task.WhenAll(task1, task2, task3);
}
catch (Exception)
{
    // await бросает только ПЕРВОЕ исключение
    // Чтобы получить все:
    var allTask = Task.WhenAll(task1, task2, task3);
    try
    {
        await allTask;
    }
    catch
    {
        AggregateException all = allTask.Exception!;
        foreach (var ex in all.InnerExceptions)
        {
            Console.WriteLine($"Error: {ex.Message}");
        }
    }
}
```

### Task.WhenAny — гонка задач

```csharp
// Первый ответивший сервер выигрывает
async Task<string> GetFastestResponseAsync(string[] urls, CancellationToken ct)
{
    using var cts = CancellationTokenSource.CreateLinkedTokenSource(ct);
    var tasks = urls.Select(url => FetchAsync(url, cts.Token)).ToList();

    Task<string> winner = await Task.WhenAny(tasks);
    await cts.CancelAsync(); // отменяем остальные

    return await winner;
}
```

```csharp
// Таймаут через WhenAny
async Task<T> WithTimeoutAsync<T>(Task<T> task, TimeSpan timeout)
{
    using var cts = new CancellationTokenSource();
    var delayTask = Task.Delay(timeout, cts.Token);

    Task completedTask = await Task.WhenAny(task, delayTask);

    if (completedTask == delayTask)
    {
        throw new TimeoutException($"Operation timed out after {timeout}");
    }

    await cts.CancelAsync(); // отменяем delay
    return await task;
}

// Использование
var result = await WithTimeoutAsync(
    FetchDataAsync(),
    TimeSpan.FromSeconds(5));
```

### ValueTask\<T\> — когда использовать (hot path, sync completion)

`ValueTask<T>` — struct, избегает аллокации когда результат уже готов (cache hit, buffered read).

```csharp
// Кэширование — часто результат возвращается синхронно
public class CachedRepository(IDbConnection db)
{
    private readonly ConcurrentDictionary<int, Product> _cache = new();

    // ValueTask — не аллоцирует Task при cache hit
    public ValueTask<Product> GetProductAsync(int id, CancellationToken ct)
    {
        if (_cache.TryGetValue(id, out var cached))
        {
            return ValueTask.FromResult(cached); // SYNC — zero allocation
        }

        return LoadFromDbAsync(id, ct); // ASYNC path
    }

    private async ValueTask<Product> LoadFromDbAsync(int id, CancellationToken ct)
    {
        var product = await db.QuerySingleAsync<Product>(
            "SELECT * FROM products WHERE id = @Id", new { Id = id });
        _cache.TryAdd(id, product);
        return product;
    }
}
```

**Правила для ValueTask:**
1. Можно `await` только ОДИН раз
2. Нельзя одновременно `await` из нескольких потоков
3. Нельзя вызвать `.Result` до завершения
4. Используй на hot path с частым sync completion
5. В остальных случаях — `Task<T>` безопаснее

```csharp
// НЕЛЬЗЯ — multiple await
ValueTask<int> vt = GetValueAsync();
// var a = await vt;
// var b = await vt; // UNDEFINED BEHAVIOR

// Если нужно несколько раз — конвертируй в Task
var task = GetValueAsync().AsTask();
var a = await task;
var b = await task; // OK
```

### Task.CompletedTask, Task.FromResult

```csharp
// Когда метод sync, но интерфейс требует Task
public Task HandleAsync(Notification notification, CancellationToken ct)
{
    Console.WriteLine(notification.Message);
    return Task.CompletedTask; // no allocation (.NET кэширует)
}

public Task<bool> IsValidAsync(string input, CancellationToken ct)
{
    bool valid = input.Length > 0;
    return Task.FromResult(valid); // кэшируется для true/false
}

// .NET 9 — Task.FromResult кэширует:
// - bool (true / false)
// - int от -1 до 9
// - string.Empty, null
// Для остальных — аллокация. Используй ValueTask если критично.
```

---

## async / await

### Как работает state machine

Компилятор превращает `async` метод в state machine (struct в .NET 9, release mode).

```csharp
// Что пишешь:
async Task<string> FetchAsync(string url, CancellationToken ct)
{
    using var client = new HttpClient();
    var response = await client.GetAsync(url, ct);    // Suspension point #1
    var content = await response.Content.ReadAsStringAsync(ct); // Suspension point #2
    return content.ToUpperInvariant();
}

// Что генерирует компилятор (упрощённо):
// struct FetchAsyncStateMachine : IAsyncStateMachine
// {
//     public int state; // -1 = start, 0 = after GetAsync, 1 = after ReadAsString
//     public AsyncTaskMethodBuilder<string> builder;
//     public string url;
//     // ... все локальные переменные как поля
//
//     void MoveNext()
//     {
//         switch (state)
//         {
//             case -1: /* start GetAsync, state = 0, return */ break;
//             case 0:  /* resume, start ReadAsString, state = 1, return */ break;
//             case 1:  /* resume, set result, complete */ break;
//         }
//     }
// }
```

**Ключевые моменты:**
- Каждый `await` — потенциальная точка приостановки (suspension point)
- Если Task уже завершён — выполнение продолжается синхронно (нет переключения)
- State machine — struct в release (не аллоцируется на heap если нет suspension)

### ConfigureAwait(false) — когда и зачем

```csharp
// В LIBRARY коде — ВСЕГДА ConfigureAwait(false)
// Не захватывает SynchronizationContext → нет deadlock
public async Task<byte[]> ReadFileAsync(string path, CancellationToken ct)
{
    var bytes = await File.ReadAllBytesAsync(path, ct).ConfigureAwait(false);
    // continuation может выполниться на ЛЮБОМ потоке ThreadPool
    return bytes;
}

// В APPLICATION коде (ASP.NET Core) — НЕ НУЖЕН
// ASP.NET Core не имеет SynchronizationContext
public async Task<IActionResult> GetData()
{
    var data = await _service.GetDataAsync(); // без ConfigureAwait — ОК
    return Ok(data);
}

// В WPF/WinForms — НУЖЕН в не-UI слоях
// UI layer:
private async void Button_Click(object sender, RoutedEventArgs e)
{
    var data = await _service.GetDataAsync(); // вернётся на UI поток
    TextBlock.Text = data.Name; // OK — мы на UI потоке
}

// Service layer (library):
public async Task<Data> GetDataAsync()
{
    // ConfigureAwait(false) — continuation не вернётся на UI поток
    var raw = await _httpClient.GetStringAsync(url).ConfigureAwait(false);
    return JsonSerializer.Deserialize<Data>(raw)!;
}
```

### async void — почему ЗАПРЕЩЕНО (кроме event handlers)

```csharp
// ЗАПРЕЩЕНО — исключение убьёт процесс
async void FireAndForgetBad()
{
    await Task.Delay(100);
    throw new InvalidOperationException("Boom!"); // UNOBSERVED → crash
}

// ПРАВИЛЬНО — event handler (единственное исключение)
async void Button_Click(object sender, RoutedEventArgs e)
{
    try
    {
        await DoWorkAsync();
    }
    catch (Exception ex)
    {
        // Обработай ошибку!
        MessageBox.Show(ex.Message);
    }
}

// ПРАВИЛЬНО — async Task вместо async void
async Task FireAndForgetSafe()
{
    await Task.Delay(100);
    throw new InvalidOperationException("Caught!"); // можно поймать
}

// Если НУЖЕН fire-and-forget:
static void SafeFireAndForget(Task task, Action<Exception>? onError = null)
{
    _ = task.ContinueWith(t =>
    {
        if (t.Exception is not null)
        {
            onError?.Invoke(t.Exception);
        }
    }, TaskContinuationOptions.OnlyOnFaulted);
}

// Использование:
SafeFireAndForget(
    SendNotificationAsync(userId),
    ex => logger.LogError(ex, "Notification failed"));
```

### Cancellation: CancellationToken, CancellationTokenSource

```csharp
// Создание и отмена
using var cts = new CancellationTokenSource();

// Передача токена
Task<string> task = FetchDataAsync(cts.Token);

// Отмена
await cts.CancelAsync(); // .NET 8+, потокобезопасная
// cts.Cancel(); // старый способ (синхронный)

// Проверка отмены в коде
async Task ProcessItemsAsync(IEnumerable<Item> items, CancellationToken ct)
{
    foreach (var item in items)
    {
        ct.ThrowIfCancellationRequested(); // бросает OperationCanceledException

        await ProcessAsync(item, ct);
    }
}
```

```csharp
// Linked token — отмена из нескольких источников
async Task HandleRequestAsync(CancellationToken requestCt)
{
    using var timeoutCts = new CancellationTokenSource(TimeSpan.FromSeconds(30));
    using var linkedCts = CancellationTokenSource.CreateLinkedTokenSource(
        requestCt, timeoutCts.Token);

    // Отменится если: request отменён ИЛИ таймаут 30с
    await DoWorkAsync(linkedCts.Token);
}
```

```csharp
// Регистрация callback при отмене
using var cts = new CancellationTokenSource();
CancellationTokenRegistration reg = cts.Token.Register(() =>
{
    Console.WriteLine("Cancellation requested! Cleaning up...");
});

// Не забудь dispose registration когда не нужна
reg.Dispose();
```

### Таймауты

```csharp
// Способ 1: CancellationTokenSource с таймаутом
using var cts = new CancellationTokenSource(TimeSpan.FromSeconds(5));
try
{
    await httpClient.GetAsync("https://slow-api.com", cts.Token);
}
catch (OperationCanceledException) when (!cts.Token.IsCancellationRequested)
{
    // Таймаут, а не ручная отмена
    throw new TimeoutException("Request timed out");
}

// Способ 2: CancelAfter
using var cts = new CancellationTokenSource();
cts.CancelAfter(TimeSpan.FromSeconds(10));
await LongRunningAsync(cts.Token);

// Способ 3: Task.WhenAny + Task.Delay
async Task<T> WithTimeoutAsync<T>(
    Func<CancellationToken, Task<T>> operation,
    TimeSpan timeout,
    CancellationToken ct = default)
{
    using var cts = CancellationTokenSource.CreateLinkedTokenSource(ct);
    var task = operation(cts.Token);
    var delayTask = Task.Delay(timeout, cts.Token);

    if (await Task.WhenAny(task, delayTask) == delayTask)
    {
        await cts.CancelAsync();
        throw new TimeoutException($"Timeout after {timeout}");
    }

    return await task;
}

// Способ 4: .NET 8+ WaitAsync
var result = await SomeTaskAsync()
    .WaitAsync(TimeSpan.FromSeconds(5)); // бросает TimeoutException
```

### IAsyncEnumerable\<T\> — async streams

```csharp
// Генерация async stream
async IAsyncEnumerable<int> GenerateNumbersAsync(
    int count,
    [EnumeratorCancellation] CancellationToken ct = default)
{
    for (int i = 0; i < count; i++)
    {
        ct.ThrowIfCancellationRequested();
        await Task.Delay(100, ct); // имитация async работы
        yield return i;
    }
}

// Потребление
await foreach (var number in GenerateNumbersAsync(10, cancellationToken))
{
    Console.WriteLine(number);
}
```

```csharp
// Реальный пример — streaming из БД
async IAsyncEnumerable<Order> GetOrdersStreamAsync(
    int userId,
    [EnumeratorCancellation] CancellationToken ct = default)
{
    await using var connection = new NpgsqlConnection(_connectionString);
    await connection.OpenAsync(ct);

    await using var command = new NpgsqlCommand(
        "SELECT id, total, created_at FROM orders WHERE user_id = @userId",
        connection);
    command.Parameters.AddWithValue("userId", userId);

    await using var reader = await command.ExecuteReaderAsync(ct);
    while (await reader.ReadAsync(ct))
    {
        yield return new Order
        {
            Id = reader.GetInt32(0),
            Total = reader.GetDecimal(1),
            CreatedAt = reader.GetDateTime(2)
        };
    }
}
```

```csharp
// LINQ-подобные расширения для IAsyncEnumerable (System.Linq.Async NuGet)
var expensiveOrders = await GetOrdersStreamAsync(userId, ct)
    .Where(o => o.Total > 1000)
    .Take(10)
    .ToListAsync(ct);
```

---

## Синхронизация

### lock statement / Monitor

```csharp
// lock — syntactic sugar для Monitor.Enter/Exit
public class ThreadSafeCounter
{
    private readonly Lock _lock = new(); // .NET 9: System.Threading.Lock
    private int _count;

    public int Increment()
    {
        lock (_lock) // .NET 9 Lock — оптимизированный, не object
        {
            return ++_count;
        }
    }

    public int Value
    {
        get
        {
            lock (_lock)
            {
                return _count;
            }
        }
    }
}
```

```csharp
// Monitor с таймаутом
public bool TryUpdate(Action action, TimeSpan timeout)
{
    bool lockTaken = false;
    try
    {
        Monitor.TryEnter(_syncRoot, timeout, ref lockTaken);
        if (lockTaken)
        {
            action();
            return true;
        }
        return false;
    }
    finally
    {
        if (lockTaken)
            Monitor.Exit(_syncRoot);
    }
}
```

> **Важно:** `lock` НЕЛЬЗЯ использовать с `await` внутри. Для async — `SemaphoreSlim`.

### SemaphoreSlim — async-friendly, ограничение concurrency

```csharp
// Как async lock (initialCount: 1)
public class AsyncSafeService
{
    private readonly SemaphoreSlim _semaphore = new(1, 1);

    public async Task<string> GetDataAsync(CancellationToken ct)
    {
        await _semaphore.WaitAsync(ct);
        try
        {
            // Только один поток за раз
            return await LoadExpensiveDataAsync(ct);
        }
        finally
        {
            _semaphore.Release();
        }
    }
}
```

```csharp
// Ограничение concurrency — максимум N одновременных операций
public class ThrottledProcessor
{
    private readonly SemaphoreSlim _throttle = new(10); // max 10 параллельных

    public async Task ProcessBatchAsync(
        IReadOnlyList<WorkItem> items,
        CancellationToken ct)
    {
        var tasks = items.Select(async item =>
        {
            await _throttle.WaitAsync(ct);
            try
            {
                await ProcessItemAsync(item, ct);
            }
            finally
            {
                _throttle.Release();
            }
        });

        await Task.WhenAll(tasks);
    }
}
```

### ReaderWriterLockSlim

Оптимизирован для сценария "много читателей, мало писателей".

```csharp
public class ThreadSafeCache<TKey, TValue> where TKey : notnull
{
    private readonly ReaderWriterLockSlim _lock = new();
    private readonly Dictionary<TKey, TValue> _cache = [];

    public TValue? Get(TKey key)
    {
        _lock.EnterReadLock(); // множество потоков одновременно
        try
        {
            return _cache.GetValueOrDefault(key);
        }
        finally
        {
            _lock.ExitReadLock();
        }
    }

    public void Set(TKey key, TValue value)
    {
        _lock.EnterWriteLock(); // эксклюзивный доступ
        try
        {
            _cache[key] = value;
        }
        finally
        {
            _lock.ExitWriteLock();
        }
    }

    public TValue GetOrAdd(TKey key, Func<TKey, TValue> factory)
    {
        // Сначала читаем
        _lock.EnterUpgradeableReadLock();
        try
        {
            if (_cache.TryGetValue(key, out var existing))
                return existing;

            _lock.EnterWriteLock();
            try
            {
                // Double-check
                if (_cache.TryGetValue(key, out existing))
                    return existing;

                var value = factory(key);
                _cache[key] = value;
                return value;
            }
            finally
            {
                _lock.ExitWriteLock();
            }
        }
        finally
        {
            _lock.ExitUpgradeableReadLock();
        }
    }
}
```

> **Замечание:** `ReaderWriterLockSlim` не поддерживает `await`. Для async сценариев используй `SemaphoreSlim` или `Channel<T>`.

### Interlocked — атомарные операции

```csharp
public class AtomicCounter
{
    private long _count;

    public long Increment() => Interlocked.Increment(ref _count);
    public long Decrement() => Interlocked.Decrement(ref _count);
    public long Value => Interlocked.Read(ref _count);

    // Compare-and-swap (CAS) — основа lock-free алгоритмов
    public long AddIfLessThan(long value, long limit)
    {
        long current;
        do
        {
            current = Interlocked.Read(ref _count);
            if (current >= limit) return current;
        }
        while (Interlocked.CompareExchange(ref _count, current + value, current) != current);

        return current + value;
    }
}
```

```csharp
// Атомарная замена ссылки
private ImmutableList<string> _items = [];

public void Add(string item)
{
    ImmutableList<string> original, updated;
    do
    {
        original = _items;
        updated = original.Add(item);
    }
    while (Interlocked.CompareExchange(ref _items, updated, original) != original);
}
```

### volatile keyword

```csharp
// volatile — запрещает кэширование переменной в регистрах CPU
// Гарантирует видимость записи другим потокам
public class VolatileFlag
{
    private volatile bool _running = true;

    public void Stop() => _running = false;

    public void WorkerLoop()
    {
        while (_running) // без volatile компилятор может закэшировать true
        {
            DoWork();
        }
    }
}

// ВАЖНО: volatile НЕ гарантирует атомарность для long/double на 32-bit
// Для этого используй Interlocked
```

---

## Concurrent паттерны

### Channel\<T\> — producer/consumer

`Channel<T>` — современная замена `BlockingCollection<T>`. Полностью async, zero allocation при правильном использовании.

```csharp
// Bounded channel — с ограничением ёмкости (backpressure)
var channel = Channel.CreateBounded<WorkItem>(new BoundedChannelOptions(100)
{
    FullMode = BoundedChannelFullMode.Wait, // ждать при переполнении
    SingleReader = true,   // оптимизация если один consumer
    SingleWriter = false
});

// Producer
async Task ProduceAsync(ChannelWriter<WorkItem> writer, CancellationToken ct)
{
    try
    {
        for (int i = 0; i < 1000; i++)
        {
            var item = new WorkItem(i);
            await writer.WriteAsync(item, ct); // ждёт если канал полон
        }
    }
    finally
    {
        writer.Complete(); // сигнал: больше данных не будет
    }
}

// Consumer
async Task ConsumeAsync(ChannelReader<WorkItem> reader, CancellationToken ct)
{
    await foreach (var item in reader.ReadAllAsync(ct))
    {
        await ProcessAsync(item, ct);
    }
    // Выход когда writer.Complete() и канал пуст
}

// Запуск
var producer = ProduceAsync(channel.Writer, ct);
var consumer = ConsumeAsync(channel.Reader, ct);
await Task.WhenAll(producer, consumer);
```

```csharp
// Unbounded channel — без ограничений (осторожно с памятью)
var channel = Channel.CreateUnbounded<LogEntry>(new UnboundedChannelOptions
{
    SingleReader = true,
    SingleWriter = false
});

// Multiple consumers
var consumers = Enumerable.Range(0, 3)
    .Select(_ => ConsumeAsync(channel.Reader, ct))
    .ToArray();

await Task.WhenAll(consumers);
```

```csharp
// Реальный пример — async pipeline (ETL)
async Task RunPipelineAsync(CancellationToken ct)
{
    var rawChannel = Channel.CreateBounded<RawData>(100);
    var processedChannel = Channel.CreateBounded<ProcessedData>(50);

    // Stage 1: Extract
    var extractor = Task.Run(async () =>
    {
        try
        {
            await foreach (var data in ReadFromSourceAsync(ct))
            {
                await rawChannel.Writer.WriteAsync(data, ct);
            }
        }
        finally { rawChannel.Writer.Complete(); }
    }, ct);

    // Stage 2: Transform (3 параллельных worker'а)
    var transformers = Enumerable.Range(0, 3).Select(_ => Task.Run(async () =>
    {
        await foreach (var raw in rawChannel.Reader.ReadAllAsync(ct))
        {
            var processed = Transform(raw);
            await processedChannel.Writer.WriteAsync(processed, ct);
        }
    }, ct)).ToArray();

    _ = Task.WhenAll(transformers).ContinueWith(
        _ => processedChannel.Writer.Complete(), ct);

    // Stage 3: Load
    var loader = Task.Run(async () =>
    {
        await foreach (var item in processedChannel.Reader.ReadAllAsync(ct))
        {
            await SaveToDbAsync(item, ct);
        }
    }, ct);

    await Task.WhenAll(extractor, Task.WhenAll(transformers), loader);
}
```

### Parallel.ForEachAsync (.NET 6+)

```csharp
// Параллельная обработка с ограничением concurrency
var urls = new List<string> { "https://api1.com", "https://api2.com", /* ... */ };

await Parallel.ForEachAsync(
    urls,
    new ParallelOptions
    {
        MaxDegreeOfParallelism = 5, // максимум 5 параллельных
        CancellationToken = ct
    },
    async (url, token) =>
    {
        var response = await httpClient.GetStringAsync(url, token);
        await ProcessResponseAsync(url, response, token);
    });
```

```csharp
// Обработка файлов параллельно
var files = Directory.GetFiles("/data", "*.csv");

await Parallel.ForEachAsync(files, new ParallelOptions
{
    MaxDegreeOfParallelism = Environment.ProcessorCount
}, async (file, token) =>
{
    var lines = await File.ReadAllLinesAsync(file, token);
    var processed = lines.Select(ParseLine).ToArray();
    await SaveResultsAsync(processed, token);
});
```

### Task.WhenAll с ограничением concurrency (SemaphoreSlim)

```csharp
// Когда Parallel.ForEachAsync не подходит (нужен результат)
async Task<TResult[]> WhenAllThrottledAsync<TSource, TResult>(
    IEnumerable<TSource> source,
    Func<TSource, CancellationToken, Task<TResult>> operation,
    int maxConcurrency,
    CancellationToken ct = default)
{
    using var semaphore = new SemaphoreSlim(maxConcurrency);

    var tasks = source.Select(async item =>
    {
        await semaphore.WaitAsync(ct);
        try
        {
            return await operation(item, ct);
        }
        finally
        {
            semaphore.Release();
        }
    });

    return await Task.WhenAll(tasks);
}

// Использование
var results = await WhenAllThrottledAsync(
    userIds,
    async (id, ct) => await GetUserAsync(id, ct),
    maxConcurrency: 10,
    ct);
```

### PeriodicTimer (.NET 6+)

```csharp
// Замена Thread.Sleep в цикле / Timer callback
async Task RunPeriodicAsync(CancellationToken ct)
{
    using var timer = new PeriodicTimer(TimeSpan.FromSeconds(30));

    while (await timer.WaitForNextTickAsync(ct))
    {
        try
        {
            await PollForUpdatesAsync(ct);
        }
        catch (Exception ex) when (ex is not OperationCanceledException)
        {
            logger.LogError(ex, "Polling error");
            // Продолжаем — не ломаем цикл
        }
    }
}
```

```csharp
// Использование в BackgroundService
public class MetricsCollector(
    ILogger<MetricsCollector> logger,
    IMetricsService metrics) : BackgroundService
{
    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        using var timer = new PeriodicTimer(TimeSpan.FromMinutes(1));

        while (await timer.WaitForNextTickAsync(stoppingToken))
        {
            var snapshot = await metrics.CollectAsync(stoppingToken);
            logger.LogInformation("CPU: {Cpu}%, Mem: {Mem}MB",
                snapshot.CpuPercent, snapshot.MemoryMb);
        }
    }
}
```

---

## Распространённые ошибки

### Deadlock: .Result/.Wait() в sync context

```csharp
// DEADLOCK в WPF/WinForms/старый ASP.NET (есть SynchronizationContext)
public string GetData()
{
    // 1. GetDataAsync ставит continuation на UI поток (SynchronizationContext)
    // 2. .Result блокирует UI поток
    // 3. Continuation не может выполниться — UI поток занят
    // → DEADLOCK
    return GetDataAsync().Result; // НИКОГДА ТАК НЕ ДЕЛАЙ
}

// Фикс 1: async all the way
public async Task<string> GetDataAsync() => await _client.GetStringAsync(url);

// Фикс 2: ConfigureAwait(false) в library коде
public async Task<string> GetDataAsync()
{
    return await _client.GetStringAsync(url).ConfigureAwait(false);
}

// Фикс 3: Task.Run (обёртка — крайний случай)
public string GetData()
{
    return Task.Run(() => GetDataAsync()).GetAwaiter().GetResult();
}
```

### Fire-and-forget без обработки ошибок

```csharp
// ПЛОХО — исключение потеряется
_ = SendEmailAsync(user.Email);

// ПРАВИЛЬНО — логирование ошибок
_ = Task.Run(async () =>
{
    try
    {
        await SendEmailAsync(user.Email);
    }
    catch (Exception ex)
    {
        logger.LogError(ex, "Failed to send email to {Email}", user.Email);
    }
});

// Лучше — extension method
public static class TaskExtensions
{
    public static void FireAndForget(
        this Task task,
        ILogger? logger = null,
        [CallerMemberName] string? caller = null)
    {
        task.ContinueWith(t =>
        {
            if (t.Exception is { } ex)
            {
                logger?.LogError(ex, "Fire-and-forget error in {Caller}", caller);
            }
        }, TaskContinuationOptions.OnlyOnFaulted);
    }
}

// Использование
SendEmailAsync(user.Email).FireAndForget(logger);
```

### Shared mutable state без синхронизации

```csharp
// ПЛОХО — race condition
public class BadCounter
{
    private int _count;
    public void Increment() => _count++; // НЕ атомарно!
}

// ПРАВИЛЬНО
public class GoodCounter
{
    private int _count;
    public int Increment() => Interlocked.Increment(ref _count);
}

// ПЛОХО — Dictionary не потокобезопасен
private readonly Dictionary<string, int> _cache = [];
// _cache[key] = value; // Race condition из нескольких потоков

// ПРАВИЛЬНО
private readonly ConcurrentDictionary<string, int> _cache = new();
```

### Не передача CancellationToken

```csharp
// ПЛОХО — операция не отменяется при shutdown
public async Task<List<Order>> GetOrdersAsync()
{
    return await _dbContext.Orders.ToListAsync(); // нет CancellationToken!
}

// ПРАВИЛЬНО — token пробрасывается через всю цепочку
public async Task<List<Order>> GetOrdersAsync(CancellationToken ct)
{
    return await _dbContext.Orders.ToListAsync(ct);
}

// В ASP.NET Core — token приходит автоматически
app.MapGet("/orders", async (
    IOrderService service,
    CancellationToken ct) => // framework inject'ит
{
    return await service.GetOrdersAsync(ct);
});
```

### Task.Run в ASP.NET Core — не нужен

```csharp
// ПЛОХО — бессмысленный Task.Run в ASP.NET Core
app.MapGet("/data", async () =>
{
    // Зачем? Мы УЖЕ на ThreadPool!
    return await Task.Run(() => GetDataAsync());
});

// ПРАВИЛЬНО
app.MapGet("/data", async (CancellationToken ct) =>
{
    return await GetDataAsync(ct);
});

// Task.Run нужен ТОЛЬКО для CPU-bound работы в UI приложениях
// чтобы не блокировать UI поток
private async void Button_Click(object sender, RoutedEventArgs e)
{
    var result = await Task.Run(() => HeavyComputation());
    TextBlock.Text = result.ToString();
}
```

---

## BackgroundService

### Паттерн для long-running tasks в ASP.NET Core

```csharp
// IHostedService — базовый интерфейс
public interface IHostedService
{
    Task StartAsync(CancellationToken cancellationToken);
    Task StopAsync(CancellationToken cancellationToken);
}

// BackgroundService — абстракция поверх IHostedService
// Реализует StartAsync/StopAsync, предоставляет ExecuteAsync
```

```csharp
// Пример: обработка очереди сообщений
public sealed class OrderProcessingService(
    IServiceScopeFactory scopeFactory,
    ILogger<OrderProcessingService> logger,
    Channel<OrderCreatedEvent> channel) : BackgroundService
{
    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        logger.LogInformation("Order processing service started");

        await foreach (var order in channel.Reader.ReadAllAsync(stoppingToken))
        {
            try
            {
                // Создаём scope для scoped-зависимостей (DbContext и т.д.)
                await using var scope = scopeFactory.CreateAsyncScope();
                var handler = scope.ServiceProvider
                    .GetRequiredService<IOrderHandler>();

                await handler.HandleAsync(order, stoppingToken);
            }
            catch (Exception ex) when (ex is not OperationCanceledException)
            {
                logger.LogError(ex, "Failed to process order {OrderId}", order.Id);
                // Не бросаем — продолжаем обрабатывать следующие
            }
        }
    }
}

// Регистрация
builder.Services.AddSingleton(Channel.CreateBounded<OrderCreatedEvent>(1000));
builder.Services.AddHostedService<OrderProcessingService>();
```

### IHostedService vs BackgroundService

```csharp
// IHostedService — для инициализации/cleanup
// StartAsync вызывается ДО того, как приложение начнёт принимать запросы
public sealed class DatabaseMigrationService(
    IServiceScopeFactory scopeFactory,
    ILogger<DatabaseMigrationService> logger) : IHostedService
{
    public async Task StartAsync(CancellationToken ct)
    {
        logger.LogInformation("Running migrations...");
        await using var scope = scopeFactory.CreateAsyncScope();
        var dbContext = scope.ServiceProvider.GetRequiredService<AppDbContext>();
        await dbContext.Database.MigrateAsync(ct);
        logger.LogInformation("Migrations complete");
    }

    public Task StopAsync(CancellationToken ct) => Task.CompletedTask;
}
```

### Graceful shutdown

```csharp
public sealed class GracefulWorker(
    ILogger<GracefulWorker> logger) : BackgroundService
{
    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        // stoppingToken срабатывает при SIGTERM / app.StopAsync()
        logger.LogInformation("Worker starting");

        try
        {
            while (!stoppingToken.IsCancellationRequested)
            {
                await DoWorkAsync(stoppingToken);
                await Task.Delay(TimeSpan.FromSeconds(5), stoppingToken);
            }
        }
        catch (OperationCanceledException)
        {
            // Ожидаемо при shutdown — НЕ логируем как ошибку
        }
        finally
        {
            // Cleanup ресурсов
            logger.LogInformation("Worker stopping, cleaning up...");
            await CleanupAsync();
        }
    }
}

// Настройка таймаута graceful shutdown
builder.Services.Configure<HostOptions>(options =>
{
    options.ShutdownTimeout = TimeSpan.FromSeconds(30);
});
```

---

## Практические рецепты

### Retry с exponential backoff (Polly)

```csharp
// NuGet: Microsoft.Extensions.Http.Polly или Polly.Core

// Polly v8+ (новый API)
using Polly;
using Polly.Retry;

// Определение стратегии
var pipeline = new ResiliencePipelineBuilder<HttpResponseMessage>()
    .AddRetry(new RetryStrategyOptions<HttpResponseMessage>
    {
        MaxRetryAttempts = 3,
        Delay = TimeSpan.FromMilliseconds(500),
        BackoffType = DelayBackoffType.Exponential, // 500ms, 1s, 2s
        UseJitter = true, // ±random, предотвращает thundering herd
        ShouldHandle = new PredicateBuilder<HttpResponseMessage>()
            .Handle<HttpRequestException>()
            .HandleResult(r => r.StatusCode == System.Net.HttpStatusCode.TooManyRequests),
        OnRetry = args =>
        {
            Console.WriteLine($"Retry {args.AttemptNumber}, delay {args.RetryDelay}");
            return ValueTask.CompletedTask;
        }
    })
    .Build();

// Использование
var response = await pipeline.ExecuteAsync(
    async ct => await httpClient.GetAsync("https://api.example.com/data", ct),
    cancellationToken);
```

```csharp
// Ручной retry без Polly
async Task<T> RetryAsync<T>(
    Func<CancellationToken, Task<T>> operation,
    int maxRetries = 3,
    CancellationToken ct = default)
{
    int delay = 200;

    for (int attempt = 0; ; attempt++)
    {
        try
        {
            return await operation(ct);
        }
        catch (Exception ex) when (attempt < maxRetries && ex is not OperationCanceledException)
        {
            var jitter = Random.Shared.Next(0, 100);
            await Task.Delay(delay + jitter, ct);
            delay *= 2; // exponential backoff
        }
    }
}

// Использование
var data = await RetryAsync(
    ct => httpClient.GetStringAsync("https://api.example.com", ct),
    maxRetries: 3,
    ct);
```

### Parallel HTTP requests с ограничением

```csharp
// Загрузка множества URL с ограничением параллельности
async Task<Dictionary<string, string>> FetchAllAsync(
    IReadOnlyList<string> urls,
    int maxParallelism = 10,
    CancellationToken ct = default)
{
    var results = new ConcurrentDictionary<string, string>();

    await Parallel.ForEachAsync(
        urls,
        new ParallelOptions
        {
            MaxDegreeOfParallelism = maxParallelism,
            CancellationToken = ct
        },
        async (url, token) =>
        {
            // ⚠️ НЕ создавай new HttpClient() на каждый запрос — socket exhaustion!
            // Используй IHttpClientFactory или один shared HttpClient.
            var content = await httpClient.GetStringAsync(url, token);
            results.TryAdd(url, content);
        });

    return new Dictionary<string, string>(results);
}
```

```csharp
// С IHttpClientFactory (правильный подход в ASP.NET Core)
public sealed class BatchFetcher(IHttpClientFactory clientFactory)
{
    public async Task<IReadOnlyList<Result<string>>> FetchBatchAsync(
        IReadOnlyList<string> urls,
        CancellationToken ct)
    {
        using var semaphore = new SemaphoreSlim(10);
        var client = clientFactory.CreateClient("default");

        var tasks = urls.Select(async url =>
        {
            await semaphore.WaitAsync(ct);
            try
            {
                var response = await client.GetStringAsync(url, ct);
                return Result<string>.Success(response);
            }
            catch (Exception ex)
            {
                return Result<string>.Failure(ex.Message);
            }
            finally
            {
                semaphore.Release();
            }
        });

        return await Task.WhenAll(tasks);
    }
}
```

### Debounce / Throttle

```csharp
// Debounce — выполнить ТОЛЬКО после паузы (поиск при вводе)
public sealed class Debouncer : IDisposable
{
    private CancellationTokenSource? _cts;
    private readonly TimeSpan _delay;

    public Debouncer(TimeSpan delay) => _delay = delay;

    public async Task DebounceAsync(Func<CancellationToken, Task> action)
    {
        // Отменяем предыдущий вызов
        _cts?.Cancel();
        _cts?.Dispose();
        _cts = new CancellationTokenSource();
        var token = _cts.Token;

        try
        {
            await Task.Delay(_delay, token);
            await action(token);
        }
        catch (OperationCanceledException)
        {
            // Ожидаемо — новый вызов отменил предыдущий
        }
    }

    public void Dispose()
    {
        _cts?.Cancel();
        _cts?.Dispose();
    }
}

// Использование в WPF ViewModel
public partial class SearchViewModel : ObservableObject
{
    private readonly Debouncer _debouncer = new(TimeSpan.FromMilliseconds(300));

    [ObservableProperty]
    private string _searchText = string.Empty;

    partial void OnSearchTextChanged(string value)
    {
        _ = _debouncer.DebounceAsync(async ct =>
        {
            var results = await _searchService.SearchAsync(value, ct);
            SearchResults = new ObservableCollection<SearchResult>(results);
        });
    }
}
```

```csharp
// Throttle — не чаще чем раз в N (rate limiting)
public sealed class Throttle
{
    private readonly SemaphoreSlim _semaphore = new(1, 1);
    private readonly TimeSpan _interval;
    private DateTime _lastExecution = DateTime.MinValue;

    public Throttle(TimeSpan interval) => _interval = interval;

    public async Task<bool> ExecuteAsync(
        Func<CancellationToken, Task> action,
        CancellationToken ct = default)
    {
        await _semaphore.WaitAsync(ct);
        try
        {
            var elapsed = DateTime.UtcNow - _lastExecution;
            if (elapsed < _interval)
            {
                var remaining = _interval - elapsed;
                await Task.Delay(remaining, ct);
            }

            await action(ct);
            _lastExecution = DateTime.UtcNow;
            return true;
        }
        finally
        {
            _semaphore.Release();
        }
    }
}

// Использование — API rate limit
var throttle = new Throttle(TimeSpan.FromMilliseconds(100)); // max 10 req/sec

foreach (var item in items)
{
    await throttle.ExecuteAsync(
        ct => apiClient.SendAsync(item, ct),
        cancellationToken);
}
```

### Producer-Consumer через Channel

```csharp
// Полный рабочий пример: обработка логов
public sealed class LogProcessor : IAsyncDisposable
{
    private readonly Channel<LogEntry> _channel;
    private readonly Task _processingTask;
    private readonly CancellationTokenSource _cts = new();

    public LogProcessor(ILogWriter writer, int bufferSize = 10_000)
    {
        _channel = Channel.CreateBounded<LogEntry>(new BoundedChannelOptions(bufferSize)
        {
            FullMode = BoundedChannelFullMode.DropOldest, // при переполнении — теряем старые
            SingleReader = true,
            SingleWriter = false
        });

        // Запуск consumer
        _processingTask = ProcessLogsAsync(writer, _cts.Token);
    }

    // Producer — вызывается из разных потоков
    public ValueTask WriteAsync(LogEntry entry, CancellationToken ct = default)
    {
        return _channel.Writer.WriteAsync(entry, ct);
    }

    // Неблокирующая попытка записи
    public bool TryWrite(LogEntry entry) => _channel.Writer.TryWrite(entry);

    private async Task ProcessLogsAsync(ILogWriter writer, CancellationToken ct)
    {
        var batch = new List<LogEntry>(100);

        await foreach (var entry in _channel.Reader.ReadAllAsync(ct))
        {
            batch.Add(entry);

            // Собираем batch пока есть элементы в канале (без ожидания)
            while (batch.Count < 100 && _channel.Reader.TryRead(out var extra))
            {
                batch.Add(extra);
            }

            // Пишем batch
            await writer.WriteBatchAsync(batch, ct);
            batch.Clear();
        }
    }

    public async ValueTask DisposeAsync()
    {
        _channel.Writer.Complete();

        // Даём время дообработать
        try
        {
            await _processingTask.WaitAsync(TimeSpan.FromSeconds(10));
        }
        catch (TimeoutException)
        {
            await _cts.CancelAsync();
        }

        _cts.Dispose();
    }
}

// Регистрация в DI
builder.Services.AddSingleton<LogProcessor>();

// Использование
app.MapPost("/api/log", async (LogEntry entry, LogProcessor processor) =>
{
    await processor.WriteAsync(entry);
    return Results.Accepted();
});
```

```csharp
// Множественные consumers с Channel (fan-out)
async Task FanOutProcessingAsync<T>(
    ChannelReader<T> reader,
    Func<T, CancellationToken, Task> handler,
    int consumerCount,
    CancellationToken ct)
{
    var consumers = Enumerable.Range(0, consumerCount)
        .Select(id => Task.Run(async () =>
        {
            await foreach (var item in reader.ReadAllAsync(ct))
            {
                try
                {
                    await handler(item, ct);
                }
                catch (Exception ex) when (ex is not OperationCanceledException)
                {
                    Console.WriteLine($"Consumer {id} error: {ex.Message}");
                }
            }
        }, ct))
        .ToArray();

    await Task.WhenAll(consumers);
}

// Использование
var channel = Channel.CreateUnbounded<WorkItem>();
await FanOutProcessingAsync(
    channel.Reader,
    async (item, ct) => await ProcessAsync(item, ct),
    consumerCount: Environment.ProcessorCount,
    ct);
```

---

## Бонус: Полезные утилиты

### AsyncLazy\<T\> — ленивая инициализация

```csharp
public sealed class AsyncLazy<T>(Func<Task<T>> factory)
{
    private readonly Lazy<Task<T>> _lazy = new(factory);

    public Task<T> Value => _lazy.Value;
    public bool IsValueCreated => _lazy.IsValueCreated;
}

// Использование — загрузка конфигурации один раз
public class AppConfig
{
    private readonly AsyncLazy<Settings> _settings = new(
        async () =>
        {
            var json = await File.ReadAllTextAsync("config.json");
            return JsonSerializer.Deserialize<Settings>(json)!;
        });

    public Task<Settings> GetSettingsAsync() => _settings.Value;
}
```

### Async initialization pattern

```csharp
// Паттерн для сервисов, требующих async инициализации
public interface IAsyncInitializable
{
    Task InitializeAsync(CancellationToken ct);
}

public sealed class CacheService : IAsyncInitializable
{
    private ImmutableDictionary<string, CacheItem> _cache =
        ImmutableDictionary<string, CacheItem>.Empty;
    private readonly SemaphoreSlim _initLock = new(1, 1);
    private bool _initialized;

    public async Task InitializeAsync(CancellationToken ct)
    {
        if (_initialized) return;

        await _initLock.WaitAsync(ct);
        try
        {
            if (_initialized) return; // double-check

            var data = await LoadCacheFromDbAsync(ct);
            _cache = data.ToImmutableDictionary(x => x.Key, x => x);
            _initialized = true;
        }
        finally
        {
            _initLock.Release();
        }
    }
}

// Автоматическая инициализация при старте
public sealed class AsyncInitializationHostedService(
    IEnumerable<IAsyncInitializable> initializables) : IHostedService
{
    public async Task StartAsync(CancellationToken ct)
    {
        foreach (var service in initializables)
        {
            await service.InitializeAsync(ct);
        }
    }

    public Task StopAsync(CancellationToken ct) => Task.CompletedTask;
}
```

### Timeout wrapper для любой операции

```csharp
public static class AsyncExtensions
{
    /// <summary>
    /// Добавляет таймаут к любой Task. Бросает TimeoutException.
    /// </summary>
    public static async Task<T> WithTimeoutAsync<T>(
        this Task<T> task,
        TimeSpan timeout,
        CancellationToken ct = default)
    {
        using var cts = CancellationTokenSource.CreateLinkedTokenSource(ct);

        var completed = await Task.WhenAny(
            task,
            Task.Delay(timeout, cts.Token));

        if (completed != task)
        {
            await cts.CancelAsync();
            throw new TimeoutException(
                $"Operation timed out after {timeout.TotalSeconds:F1}s");
        }

        await cts.CancelAsync(); // cancel the delay
        return await task; // propagate exceptions
    }

    /// <summary>
    /// Выполняет операцию и возвращает Result, не бросая исключений.
    /// </summary>
    public static async Task<Result<T>> TryAsync<T>(
        this Task<T> task)
    {
        try
        {
            var result = await task;
            return Result<T>.Success(result);
        }
        catch (Exception ex)
        {
            return Result<T>.Failure(ex.Message);
        }
    }
}

// Использование
var result = await httpClient.GetStringAsync(url, ct)
    .WithTimeoutAsync(TimeSpan.FromSeconds(5), ct);
```

---

## Шпаргалка — когда что использовать

| Задача | Инструмент |
|---|---|
| CPU-bound в UI | `Task.Run` |
| IO-bound | `async/await` (без Task.Run!) |
| Параллельная обработка коллекции | `Parallel.ForEachAsync` |
| Множество async операций | `Task.WhenAll` |
| Ограничение concurrency | `SemaphoreSlim` / `Parallel.ForEachAsync` |
| Producer-Consumer | `Channel<T>` |
| Периодическая задача | `PeriodicTimer` + `BackgroundService` |
| Фоновая задача в ASP.NET | `BackgroundService` |
| Инициализация при старте | `IHostedService` |
| Кэш с sync fast-path | `ValueTask<T>` |
| Retry/Circuit Breaker | Polly |
| Атомарные операции | `Interlocked` |
| Async lock | `SemaphoreSlim(1, 1)` |
| Много читателей, мало писателей | `ReaderWriterLockSlim` |
| Lock-free коллекции | `ConcurrentDictionary`, `ConcurrentQueue` |

---

## См. также

- [[Reference/csharp-delegates-events|Delegates и Events]]
- [[Reference/csharp-collections-linq|Collections и LINQ]]
- [[Reference/csharp-error-handling|Обработка ошибок]]
- [[Topics/Performance/dotnet-performance|.NET Performance]]
- [[dotnet-knowledge-base|.NET Knowledge Base]]
