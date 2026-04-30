---
tags: [runtime, threading, thread, threadpool, tpl, parallel, plinq, middle]
level: Middle
date: 2026-04-30
---

# Threading Basics — Thread, ThreadPool, TPL

> **Фундамент многопоточности перед async/await**. Что такое Thread, ThreadPool, Task Parallel Library, Parallel.For, PLINQ. Closes пробел "знаю про async/await но не понимаю threading basics". Частая Middle интервью тема.

---

## Что это, зачем и когда

### Зачем многопоточность

```
Один thread:
  task1 → task2 → task3   (последовательно)
  Total: 3 секунды

Multi-thread (3 cores):
  task1 ┐
  task2 ├ параллельно
  task3 ┘
  Total: 1 секунда
```

**Use cases:**
- **CPU-bound work** — вычисления, image processing, ML inference (`Parallel.For`)
- **I/O-bound work** — HTTP calls, БД, file reads (`async/await`)
- **Background processing** — фоновая работа без блокирования UI
- **Server scalability** — обработка многих clients одновременно

### Эволюция в C# / .NET

```
.NET 1.0 (2002): Thread class — manual, дорого
.NET 2.0 (2005): ThreadPool — переиспользование threads
.NET 4.0 (2010): TPL — Task, Parallel.For, PLINQ
.NET 4.5 (2012): async/await — syntactic sugar над TPL
.NET 5+ (2020+): Channels, Pipelines, IAsyncEnumerable
```

> [!info] Что использовать в 2026
> **99% случаев** — `async/await` + `Task` + `Parallel.For`. Thread / ThreadPool — для глубокого понимания, редко напрямую.

См. [[../CSharp/async-threading|Async и Threading]] для async deep.

---

## 1. Thread vs Process

### Process

**Изолированный** запущенный экземпляр программы. Свой адресное пространство, файловые handles.

```
Chrome.exe — process #1
Visual Studio.exe — process #2
SQL Server.exe — process #3
```

### Thread

**Поток выполнения внутри процесса**. Имеет свой stack, но **shared heap** с другими threads процесса.

```
Chrome.exe (process):
  ├── Main thread (UI)
  ├── Network thread
  ├── Renderer thread (для каждой tab)
  └── ...
```

| | Process | Thread |
|--|---------|--------|
| Изоляция памяти | Полная | Shared heap |
| Cost создания | High (~ms) | Medium (~10μs) |
| Communication | Pipes, sockets | Shared memory |
| Crash | Один не убивает другой | Один убивает process |

---

## 2. Thread class — низкий уровень

### Создание thread

```csharp
using System.Threading;

// Простой Thread
Thread t = new Thread(() =>
{
    Console.WriteLine("Running on a new thread");
    Thread.Sleep(1000);
    Console.WriteLine("Done");
});

t.Start();
t.Join();  // wait for thread to finish
```

### Параметры

```csharp
// Через delegate с параметром
Thread t = new Thread((object data) =>
{
    Console.WriteLine($"Got: {data}");
});

t.Start("Hello");

// Better — через closure (typed)
string message = "Hello";
Thread t = new Thread(() =>
{
    Console.WriteLine($"Got: {message}");
});
t.Start();
```

### Свойства Thread

```csharp
Thread t = new Thread(Work);
t.Name = "WorkerThread";
t.IsBackground = true;       // background thread — не держит process alive
t.Priority = ThreadPriority.AboveNormal;  // приоритет (редко работает)

t.Start();

// State
t.ThreadState;               // Unstarted, Running, WaitSleepJoin, Stopped, ...
t.IsAlive;                   // bool
t.ManagedThreadId;           // unique ID
```

### Background vs Foreground threads

```csharp
// Foreground (default) — process НЕ завершится пока thread жив
Thread t = new Thread(LongWork);
t.Start();
// Если Main завершится — process будет ждать t

// Background — process завершится, thread будет abrubtly остановлен
Thread t = new Thread(LongWork) { IsBackground = true };
t.Start();
```

### Thread.Sleep

```csharp
Thread.Sleep(1000);                    // 1000 ms
Thread.Sleep(TimeSpan.FromMinutes(5));

// Sleep — БЛОКИРУЕТ thread (не отдаёт CPU)
// В async коде используй Task.Delay вместо!
```

### Thread.Yield

```csharp
Thread.Yield();
// Намекает scheduler'у — "если есть другой ready thread, дай ему cycle"
// Редко нужно
```

### Thread cancellation — нет!

```csharp
// ❌ Thread.Abort — устаревший, опасный, не работает в .NET Core
t.Abort();  // throws PlatformNotSupportedException на .NET Core+

// ✅ Cooperative cancellation через CancellationToken
var cts = new CancellationTokenSource();
Thread t = new Thread(() =>
{
    while (!cts.Token.IsCancellationRequested)
    {
        // ... work ...
    }
});

t.Start();
cts.Cancel();  // ask thread to stop
t.Join();
```

См. [[../CSharp/async-threading|Async и Threading]] — CancellationToken deep.

### Когда использовать Thread напрямую

❌ **Почти никогда** в современном C#:
- Thread тяжёлый (~1MB stack по default)
- Создание дорогое
- Нет result value
- Нет async поддержки

✅ **Только если:**
- Long-running dedicated thread (UI, hardware monitoring)
- Specific thread affinity (привязка к core)
- Apartment state (STA для COM)

---

## 3. ThreadPool — переиспользование threads

### Зачем

Создание Thread дорогое. **ThreadPool** — pool готовых threads, переиспользуемых.

```
Без ThreadPool:                   С ThreadPool:
  Task → Create Thread → Run        Task → Get from pool → Run → Return to pool
  → Destroy                          (no destroy)
  
  10ms overhead каждый task        ~1μs overhead
```

### Использование

```csharp
// ThreadPool.QueueUserWorkItem
ThreadPool.QueueUserWorkItem(state =>
{
    Console.WriteLine("Running on pool thread");
});

// С данными
ThreadPool.QueueUserWorkItem(state =>
{
    string s = (string)state;
    Console.WriteLine(s);
}, "Hello");
```

### Свойства ThreadPool

```csharp
// Min / max threads
ThreadPool.GetMinThreads(out int workerThreads, out int ioThreads);
ThreadPool.GetMaxThreads(out int max, out int maxIo);

ThreadPool.SetMinThreads(50, 50);   // Reserve 50 ready threads
```

### Available threads

```csharp
ThreadPool.GetAvailableThreads(out int worker, out int io);
Console.WriteLine($"Available: worker={worker}, io={io}");
```

### Когда напрямую ThreadPool

❌ **Не используй напрямую** — `Task.Run` лучше во всём:
- Имеет return value
- Поддерживает async
- Cancellation, exceptions, continuations

```csharp
// ❌ Старый стиль
ThreadPool.QueueUserWorkItem(_ => DoWork());

// ✅ Modern
Task.Run(DoWork);
```

### Pitfall: thread pool starvation

```csharp
// ❌ Sync I/O в async коде блокирует pool threads
public async Task<string> Get()
{
    return File.ReadAllText("file.txt");  // sync!
    // Thread занят 100ms ожиданием
}

// При 100 concurrent requests — все pool threads заняты ожиданием I/O
// → новые requests timeout
```

См. [[../CSharp/async-threading|Async deep]].

---

## 4. Task — TPL

### Что такое Task

**Абстракция асинхронной операции** — может вернуть значение, может быть cancelled, может иметь continuations.

```csharp
// Task без результата
Task t1 = Task.Run(() => Console.WriteLine("Hello"));
await t1;

// Task с результатом
Task<int> t2 = Task.Run(() => 42);
int result = await t2;

// Task.FromResult — already-completed task
Task<int> t3 = Task.FromResult(42);

// Task.CompletedTask — completed без результата
Task t4 = Task.CompletedTask;
```

### Task.Run vs new Thread

```csharp
// new Thread — heavyweight, специальный thread
Thread t = new Thread(Work);
t.Start();
t.Join();

// Task.Run — lightweight, использует ThreadPool
Task t = Task.Run(Work);
await t;
```

`Task.Run` использует ThreadPool под капотом — namного эффективнее.

### Task continuation — ContinueWith

```csharp
// Старый стиль
Task<int> task = Task.Run(() => 42);
task.ContinueWith(t =>
{
    Console.WriteLine($"Result: {t.Result}");
});

// ✅ Modern — async/await
var result = await Task.Run(() => 42);
Console.WriteLine($"Result: {result}");
```

### WaitAll / WaitAny

```csharp
Task t1 = Task.Run(Work1);
Task t2 = Task.Run(Work2);
Task t3 = Task.Run(Work3);

// Wait all (sync — blocks!)
Task.WaitAll(t1, t2, t3);

// ✅ Async version
await Task.WhenAll(t1, t2, t3);

// Wait any (first to complete)
int firstIndex = Task.WaitAny(t1, t2, t3);  // sync
Task firstCompleted = await Task.WhenAny(t1, t2, t3);  // async
```

См. [[../CSharp/async-threading|Async]] для деталей.

### Cancellation

```csharp
var cts = new CancellationTokenSource();

Task t = Task.Run(() =>
{
    while (!cts.Token.IsCancellationRequested)
    {
        // work
    }
}, cts.Token);

cts.CancelAfter(TimeSpan.FromSeconds(5));

try
{
    await t;
}
catch (OperationCanceledException)
{
    Console.WriteLine("Cancelled");
}
```

---

## 5. Parallel — параллельные циклы

### Parallel.For

```csharp
using System.Threading.Tasks;

// Sequential
for (int i = 0; i < 1_000_000; i++)
{
    array[i] = ProcessExpensive(i);
}

// Parallel
Parallel.For(0, 1_000_000, i =>
{
    array[i] = ProcessExpensive(i);
});
// Использует все доступные cores
```

### Parallel.ForEach

```csharp
List<string> urls = new() { "url1", "url2", "url3", ... };

Parallel.ForEach(urls, url =>
{
    DownloadFile(url);
});

// С опциями
Parallel.ForEach(urls, new ParallelOptions
{
    MaxDegreeOfParallelism = 4  // максимум 4 одновременно
}, url =>
{
    DownloadFile(url);
});
```

### Parallel.Invoke

```csharp
Parallel.Invoke(
    () => Task1(),
    () => Task2(),
    () => Task3()
);
// Все 3 параллельно, ждёт всех
```

### Когда Parallel

✅ **Хорошо:**
- CPU-bound работа (вычисления, обработка изображений)
- Много элементов в коллекции
- Каждый элемент независимый
- Работа > 1ms на элемент

❌ **Плохо:**
- I/O-bound (используй `async/await` + `Task.WhenAll`)
- Маленькие массивы (<1000 elements обычно)
- Зависимости между итерациями
- Очень short work — overhead больше profit

### Pitfalls Parallel.For

#### 1. Race conditions

```csharp
// ❌ Race на shared variable!
int sum = 0;
Parallel.For(0, 1_000_000, i =>
{
    sum += i;  // ⚠️ Race condition — corrupted result
});

// ✅ Parallel.For с aggregation
int sum = 0;
Parallel.For(0, 1_000_000,
    () => 0,                          // local init
    (i, state, localSum) => localSum + i,  // local update
    localSum => Interlocked.Add(ref sum, localSum));  // local finalize

// ✅ Или Sum через PLINQ
int sum = Enumerable.Range(0, 1_000_000).AsParallel().Sum();
```

См. [[concurrency-atomics|Concurrency и Atomics]].

#### 2. Не подходит для I/O

```csharp
// ❌ Parallel.ForEach с async — БЛОКИРУЕТ pool threads
Parallel.ForEach(urls, async url =>
{
    await DownloadAsync(url);  // не правильно работает!
});

// ✅ Task.WhenAll для I/O
var tasks = urls.Select(DownloadAsync);
await Task.WhenAll(tasks);

// ✅ С ограничением concurrency
var sem = new SemaphoreSlim(10);
var tasks = urls.Select(async url =>
{
    await sem.WaitAsync();
    try { return await DownloadAsync(url); }
    finally { sem.Release(); }
});
await Task.WhenAll(tasks);

// ✅ Parallel.ForEachAsync (.NET 6+)
await Parallel.ForEachAsync(urls, new ParallelOptions
{
    MaxDegreeOfParallelism = 10
}, async (url, ct) =>
{
    await DownloadAsync(url, ct);
});
```

---

## 6. PLINQ — Parallel LINQ

LINQ запросы которые автоматически параллелятся.

```csharp
// Sequential
var result = numbers
    .Where(n => IsExpensive(n))
    .Select(n => Transform(n))
    .ToList();

// Parallel
var result = numbers
    .AsParallel()                 // включает PLINQ
    .Where(n => IsExpensive(n))
    .Select(n => Transform(n))
    .ToList();
```

### Опции

```csharp
var result = numbers
    .AsParallel()
    .WithDegreeOfParallelism(4)               // максимум 4 threads
    .WithCancellation(ct)                       // cancellation
    .WithExecutionMode(ParallelExecutionMode.ForceParallelism)
    .Where(n => IsExpensive(n))
    .ToList();
```

### Order preservation

```csharp
// Default — order может теряться (faster)
var unordered = numbers.AsParallel().Select(...).ToList();

// Preserve order (slower)
var ordered = numbers.AsParallel().AsOrdered().Select(...).ToList();
```

### Когда PLINQ

✅ **Хорошо для:**
- LINQ-style transformations
- CPU-bound операции в pipeline
- Большие коллекции (>10K elements)

❌ **Плохо:**
- I/O в pipeline (`Where(n => HttpCall(n))`)
- Маленькие коллекции
- Side effects во время iteration

### Sequential vs Parallel — measure!

```csharp
// Маленькая работа — overhead больше profit
var result = numbers
    .Where(n => n > 0)        // simple — sequential faster
    .ToList();

// Тяжёлая работа — parallel выгоден
var result = numbers
    .AsParallel()
    .Select(n => CalculateExpensiveStuff(n))
    .ToList();
```

---

## 7. Synchronization primitives

См. [[concurrency-atomics|Concurrency и Atomics]] для deep dive. Краткий overview:

### lock — простейший mutex

```csharp
private readonly object _lock = new();
private int _counter = 0;

public void Increment()
{
    lock (_lock)
    {
        _counter++;
    }
}
```

### Interlocked — atomic operations

```csharp
private int _counter = 0;

public void Increment()
{
    Interlocked.Increment(ref _counter);  // atomic, no lock
}
```

### SemaphoreSlim — concurrent limit

```csharp
SemaphoreSlim sem = new(maxCount: 4);

await sem.WaitAsync();
try { /* work */ }
finally { sem.Release(); }

// Максимум 4 concurrent работ
```

### ConcurrentDictionary, ConcurrentBag, ConcurrentQueue

```csharp
ConcurrentDictionary<string, int> dict = new();
dict.TryAdd("key", 1);
dict.AddOrUpdate("key", 1, (k, v) => v + 1);

ConcurrentQueue<int> queue = new();
queue.Enqueue(1);
queue.TryDequeue(out int x);

ConcurrentBag<int> bag = new();
bag.Add(1);
```

См. [[../CSharp/collections-linq|Collections и LINQ]].

---

## 8. Async/await vs Thread/Parallel

### Когда что

```
CPU-bound (calculations, image processing, ML)?
  → Parallel.For / PLINQ / Task.Run
  → Использует ThreadPool, parallel cores

I/O-bound (HTTP, DB, file)?
  → async/await + Task.WhenAll
  → НЕ занимает thread пока ждёт I/O
  → Server scales до тысяч concurrent requests

Long-running dedicated work?
  → Thread / Task.Factory.StartNew(LongRunning)
  → Не загружает ThreadPool

Short fire-and-forget?
  → Task.Run
  → Использует ThreadPool

UI thread + heavy work?
  → Task.Run для work + await для UI update
```

### Decision matrix

| Scenario | Right tool |
|----------|------------|
| Read 100 files параллельно | `await Task.WhenAll(files.Select(File.ReadAllTextAsync))` |
| Compute 1M values | `Parallel.For` |
| Single API call | `await httpClient.GetAsync()` |
| 100 API calls | `await Task.WhenAll(urls.Select(httpClient.GetAsync))` |
| Long-running daemon | `new Thread { IsBackground = true }` или `BackgroundService` |
| LINQ over big data | `data.AsParallel().Where(...)` |
| Don't await result | `_ = Task.Run(() => Work())` (но осторожно с exceptions!) |

---

## 9. Common patterns

### Pattern 1: Producer-Consumer

```csharp
using System.Collections.Concurrent;

ConcurrentQueue<WorkItem> queue = new();

// Producer
Task.Run(() =>
{
    foreach (var item in GenerateItems())
    {
        queue.Enqueue(item);
    }
});

// Consumer (multiple)
for (int i = 0; i < 4; i++)
{
    Task.Run(() =>
    {
        while (true)
        {
            if (queue.TryDequeue(out var item))
            {
                Process(item);
            }
            else
            {
                Thread.Sleep(10);  // ждём
            }
        }
    });
}

// ✅ Modern — Channels
Channel<WorkItem> channel = Channel.CreateUnbounded<WorkItem>();

// Producer
await channel.Writer.WriteAsync(item);

// Consumer
await foreach (var item in channel.Reader.ReadAllAsync())
{
    await ProcessAsync(item);
}
```

См. [[../CSharp/async-threading|Async]] — Channels deep.

### Pattern 2: Background service в ASP.NET Core

```csharp
public class CleanupService : BackgroundService
{
    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        while (!stoppingToken.IsCancellationRequested)
        {
            await CleanupOldData();
            await Task.Delay(TimeSpan.FromHours(1), stoppingToken);
        }
    }
}

// Регистрация
builder.Services.AddHostedService<CleanupService>();
```

См. [[../AspNetCore/hosting-background|Hosting & Background]].

### Pattern 3: Limited parallelism

```csharp
// Максимум 5 одновременных downloads
var sem = new SemaphoreSlim(5);
var tasks = urls.Select(async url =>
{
    await sem.WaitAsync();
    try
    {
        return await DownloadAsync(url);
    }
    finally
    {
        sem.Release();
    }
});

var results = await Task.WhenAll(tasks);

// .NET 6+ — встроенно
await Parallel.ForEachAsync(urls,
    new ParallelOptions { MaxDegreeOfParallelism = 5 },
    async (url, ct) => await DownloadAsync(url, ct));
```

### Pattern 4: Periodic timer (.NET 6+)

```csharp
using PeriodicTimer timer = new(TimeSpan.FromSeconds(10));

while (await timer.WaitForNextTickAsync(stoppingToken))
{
    await DoPeriodicWork();
}
```

### Pattern 5: Parallel pipeline

```csharp
// Stage 1: Read files
var files = Directory.EnumerateFiles("data");

// Stage 2: Parse parallel
var parsed = files.AsParallel().Select(ParseFile);

// Stage 3: Aggregate
var result = parsed.GroupBy(p => p.Category)
    .ToDictionary(g => g.Key, g => g.Sum(p => p.Value));
```

---

## 10. Common Pitfalls

### 1. Thread.Abort

```csharp
// ❌ Deprecated, throws on .NET Core
thread.Abort();

// ✅ Cooperative cancellation
var cts = new CancellationTokenSource();
// pass cts.Token, check IsCancellationRequested
```

### 2. lock на public objects

```csharp
// ❌ Anyone могут lock на этом — deadlock потенциальный
public class MyClass
{
    public object Lock = new();
    public void Method() { lock (Lock) { } }
}

// ✅ Private lock
public class MyClass
{
    private readonly object _lock = new();
    public void Method() { lock (_lock) { } }
}
```

### 3. lock на string

```csharp
// ❌ Strings interned — все instances same reference!
private string _lock = "lock";  // ⚠️ может lock с другими modules!

// ✅
private readonly object _lock = new();
```

### 4. Async + Thread.Sleep

```csharp
// ❌ Sleep блокирует thread (не отдаёт пul)
public async Task Method()
{
    Thread.Sleep(1000);  // ⚠️ blocks pool thread
}

// ✅ Task.Delay — releases thread
public async Task Method()
{
    await Task.Delay(1000);
}
```

### 5. Blocking await

```csharp
// ❌ .Result / .Wait — deadlock potentially!
public string Get()
{
    return GetAsync().Result;  // ⚠️ может deadlock в UI / ASP.NET
}

// ✅ Async all the way
public async Task<string> Get()
{
    return await GetAsync();
}
```

См. [[../CSharp/async-threading|Async]] — sync over async pitfall.

### 6. Forgot ConfigureAwait в library

```csharp
// Library code
public async Task<string> GetData()
{
    return await httpClient.GetStringAsync(url);
    // По default — captures SyncContext (UI thread, ASP.NET context)
}

// ✅ В library
public async Task<string> GetData()
{
    return await httpClient.GetStringAsync(url).ConfigureAwait(false);
}
```

### 7. ConcurrentDictionary AddOrUpdate race

```csharp
ConcurrentDictionary<string, int> counts = new();

// ❌ TryGetValue + потом TryAdd — race condition!
if (counts.TryGetValue(key, out int v))
{
    counts[key] = v + 1;  // ⚠️ может перезаписать другую thread's update
}
else
{
    counts[key] = 1;
}

// ✅ AddOrUpdate — atomic
counts.AddOrUpdate(key, 1, (k, v) => v + 1);
```

### 8. Parallel.For + I/O

```csharp
// ❌ Parallel.For не подходит для async
Parallel.For(0, urls.Length, async i =>
{
    var data = await DownloadAsync(urls[i]);  // не работает корректно
});

// ✅ Task.WhenAll или Parallel.ForEachAsync
await Task.WhenAll(urls.Select(DownloadAsync));
// или
await Parallel.ForEachAsync(urls, async (url, ct) => await DownloadAsync(url, ct));
```

### 9. Fire-and-forget без exception handling

```csharp
// ❌ Exception тихо проглатывается
Task.Run(() =>
{
    throw new Exception("oops");
});

// Process продолжает но exception lost

// ✅ Catch внутри
Task.Run(() =>
{
    try { /* work */ }
    catch (Exception ex) { Log(ex); }
});

// ✅ Или await + catch
try { await Task.Run(Work); }
catch (Exception ex) { Log(ex); }
```

### 10. Race на коллекциях

```csharp
List<int> list = new();

// ❌ List<T> не thread-safe!
Parallel.For(0, 1000, i => list.Add(i));
// Может потерять элементы или throw

// ✅ ConcurrentBag / ConcurrentQueue
ConcurrentBag<int> bag = new();
Parallel.For(0, 1000, i => bag.Add(i));

// ✅ Или lock
List<int> list = new();
object lockObj = new();
Parallel.For(0, 1000, i =>
{
    lock (lockObj) { list.Add(i); }
});
// (но это снижает parallelism)
```

---

## 11. Performance considerations

### Cost of operations

```
Lock contention (uncontended):    ~25 ns
Lock contention (contended):      ~1 μs to ~ms
Thread context switch:            ~1-3 μs
Thread.Sleep wake-up:             ~1-15 ms
Task.Run overhead:                ~1 μs
Interlocked.Increment:            ~5 ns
ConcurrentDictionary lookup:      ~100 ns
```

### Amdahl's Law

> Если 50% кода нельзя параллелить, **maximum speedup** = 2x даже на бесконечном числе cores.

Profile перед параллелизацией!

### Striped locking

```csharp
// Вместо одного lock на всю коллекцию
private readonly Dictionary<int, T> _items = new();
private readonly object _lock = new();

// ✅ ConcurrentDictionary использует stripe locks внутри (16 locks по default)
private readonly ConcurrentDictionary<int, T> _items = new();
```

См. [[concurrency-atomics|Concurrency и Atomics]].

---

## 12. Best Practices

### General

- **`async/await` для I/O**, **Parallel для CPU**
- **Не используй Thread класс** напрямую — `Task.Run`
- **CancellationToken** в каждом long-running async method
- **`SemaphoreSlim`** для async, `lock` для sync
- **ThreadPool starvation** — не блокируй pool threads с sync I/O

### Locking

- **Private lock objects** не public
- **Не lock на string / this / typeof()**
- **Минимизируй lock duration**
- **Один lock на ресурс** — не nested locks (deadlock risk)
- **Order locks consistently** — если nested неизбежны

### Collections

- **`ConcurrentDictionary` / Bag / Queue** — для concurrent access
- **`ConcurrentDictionary.AddOrUpdate`** — atomic update
- **`Channel<T>`** — для producer-consumer

### Async

- **`Task.WhenAll`** для параллельных async I/O
- **`SemaphoreSlim`** для ограничения concurrency
- **`Parallel.ForEachAsync`** (.NET 6+) для async batch
- **`ConfigureAwait(false)`** в libraries

### Modern (.NET 6+)

- **`PeriodicTimer`** для periodic work
- **`Parallel.ForEachAsync`** для async parallelism
- **`Channel<T>`** для pipelines
- **`IAsyncEnumerable<T>`** для streaming

---

## 13. Cheat sheet

| Сценарий | Решение |
|----------|---------|
| CPU-bound параллельный loop | `Parallel.For` |
| I/O-bound несколько operations | `await Task.WhenAll(...)` |
| LINQ pipeline parallel | `data.AsParallel().Select(...)` |
| Limited concurrency I/O | `SemaphoreSlim` или `Parallel.ForEachAsync` |
| Background service | `BackgroundService` (ASP.NET) |
| Periodic work | `PeriodicTimer` |
| Producer-consumer | `Channel<T>` |
| Atomic counter | `Interlocked.Increment(ref counter)` |
| Concurrent dictionary | `ConcurrentDictionary<K, V>` |
| Cancellation | `CancellationTokenSource` |
| Single mutex | `lock(obj)` |
| Async mutex | `SemaphoreSlim(1, 1)` |
| Sleep async | `await Task.Delay(...)` |
| Long-running thread | `BackgroundService` или `Task.Factory.StartNew(work, TaskCreationOptions.LongRunning)` |

---

## 14. Decision tree

```
Что параллелить?
│
├── CPU-bound (вычисления)?
│   ├── Loop → Parallel.For
│   ├── LINQ → AsParallel()
│   └── Single task → Task.Run
│
├── I/O-bound (HTTP, DB, file)?
│   ├── Несколько → Task.WhenAll
│   ├── Limited concurrency → SemaphoreSlim или Parallel.ForEachAsync
│   └── Streaming → IAsyncEnumerable + ConfigureAwait
│
├── Long-running background?
│   ├── ASP.NET → BackgroundService
│   └── Console → Task.Factory.StartNew(LongRunning)
│
└── UI + heavy work?
    └── await Task.Run(work) + await UI update

Synchronization?
│
├── Atomic int? → Interlocked
├── Single mutex? → lock (private object)
├── Async mutex? → SemaphoreSlim(1, 1)
├── Limit concurrency? → SemaphoreSlim(N)
├── Concurrent collection? → ConcurrentDictionary / Bag / Queue
└── Producer-consumer? → Channel<T>
```

---

## См. также

- [[../CSharp/async-threading|Async и Threading]] — async/await deep
- [[concurrency-atomics|Concurrency и Atomics]] — lock, Interlocked, memory model
- [[../CSharp/collections-linq|Collections и LINQ]] — concurrent collections
- [[../AspNetCore/hosting-background|Hosting & Background]] — BackgroundService
- [[gc-memory|GC и память]] — thread allocations
- [[../Performance/async-performance|Async Performance]]

## Reading list

- **Microsoft Docs — Threading** — learn.microsoft.com/dotnet/standard/threading
- **Microsoft Docs — TPL** — learn.microsoft.com/dotnet/standard/parallel-programming
- **Microsoft Docs — Parallel.ForEachAsync** — learn.microsoft.com
- **Concurrency in C#** — Stephen Cleary (must-read book)
- **Stephen Cleary blog** — blog.stephencleary.com
- **Stephen Toub — Threading internals** — devblogs.microsoft.com/dotnet
- **CLR via C#** — Jeffrey Richter (chapters про threading)
- **C# in Depth** — Jon Skeet (async/await)
