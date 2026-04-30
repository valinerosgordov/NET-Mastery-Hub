---
tags: [performance, optimization, patterns, hot-path]
level: Middle to Senior
date: 2026-04-30
---

# Optimization Patterns — общие приёмы

> **Practical optimization patterns** для типичных задач. Когда применять, что измерять, какие trade-offs. Не "magic tricks" — паттерны с обоснованием.

---

## Что это, зачем и когда

### Зачем эти паттерны

Performance work — повторяющиеся проблемы. Опытные разработчики **узнают** ситуацию и применяют **proven** pattern, не изобретают велосипед.

### Когда оптимизировать

См. [[performance-fundamentals|Performance Fundamentals]] — **только после measurement**.

### Главные категории

```
1. Reduce work     — делать меньше операций
2. Reuse           — не создавать заново то что есть
3. Batch           — группируй операции
4. Async / parallel — пока ждём, делай другое
5. Lazy            — откладывай работу до нужности
6. Eager           — наоборот, делай заранее
7. Approximate     — точность не нужна — sample / probabilistic
```

---

## 1. Reduce work patterns

### Pattern: Early return / fail fast

```csharp
// ❌ Делает работу, потом проверяет result
public Result Process(Request req)
{
    var heavyResult = ExpensiveCalculation(req);
    if (req.IsInvalid)
        return Result.Failure();
    
    return Result.Success(heavyResult);
}

// ✅ Validate first, then work
public Result Process(Request req)
{
    if (req.IsInvalid)
        return Result.Failure();
    
    return Result.Success(ExpensiveCalculation(req));
}
```

### Pattern: Short-circuit boolean evaluation

```csharp
// ❌ Both checked даже если первое уже даёт answer
if (HasAccess(user) && IsAdmin(user)) { ... }

// ✅ Если HasAccess false — IsAdmin не вызывается
// (это default behavior C# с && / ||)
```

### Pattern: Filter early in LINQ

```csharp
// ❌ Materializes 1000 items, потом filter — wasted work
var data = await _db.Orders.ToListAsync();
var filtered = data.Where(o => o.IsActive).ToList();

// ✅ Filter в DB, materialize только нужное
var filtered = await _db.Orders.Where(o => o.IsActive).ToListAsync();
```

### Pattern: Project only needed columns

```csharp
// ❌ SELECT *
var users = await _db.Users.ToListAsync();

// ✅ SELECT id, name
var users = await _db.Users
    .Select(u => new { u.Id, u.Name })
    .ToListAsync();
```

См. [[../EFCore/queries-performance|EF Queries Performance]].

### Pattern: Skip unchanged work

```csharp
// ❌ Обновляет всегда
public async Task SetStatus(int id, string status)
{
    var entity = await _db.Find(id);
    entity.Status = status;
    await _db.SaveChangesAsync();
}

// ✅ Skip if unchanged
public async Task SetStatus(int id, string status)
{
    var entity = await _db.Find(id);
    if (entity.Status == status) return;  // No-op
    
    entity.Status = status;
    await _db.SaveChangesAsync();
}
```

---

## 2. Reuse patterns

### Pattern: Object Pool

Не создавать одноразовые объекты — переиспользовать.

```csharp
// ❌ Каждый request — new StringBuilder (100 KB allocation)
public string Render(Items items)
{
    var sb = new StringBuilder();
    foreach (var item in items) sb.Append(item.Name);
    return sb.ToString();
}

// ✅ Pool reuse
private static readonly ObjectPool<StringBuilder> _pool = 
    new DefaultObjectPool<StringBuilder>(new StringBuilderPooledObjectPolicy());

public string Render(Items items)
{
    var sb = _pool.Get();
    try
    {
        foreach (var item in items) sb.Append(item.Name);
        return sb.ToString();
    }
    finally
    {
        sb.Clear();
        _pool.Return(sb);
    }
}
```

**ObjectPool готов в .NET для:**
- StringBuilder
- ArrayPool\<T\>
- Custom через `ObjectPool<T>` (Microsoft.Extensions.ObjectPool)

### Pattern: ArrayPool\<T\>

```csharp
// ❌ Allocate 1 MB array каждый request
public byte[] Process()
{
    var buffer = new byte[1024 * 1024];
    // ... use buffer ...
    return result;
}

// ✅ Rent from pool
public byte[] Process()
{
    var buffer = ArrayPool<byte>.Shared.Rent(1024 * 1024);
    try
    {
        // ... use buffer ...
        return result;
    }
    finally
    {
        ArrayPool<byte>.Shared.Return(buffer, clearArray: true);
    }
}
```

> [!info] Когда ArrayPool
> Hot path с large temporary arrays. Если массив < 1 KB или редко — обычный `new` OK.

См. [[../Runtime/span-layout|Span и Layout]].

### Pattern: Cache instances (singleton)

```csharp
// ❌ Регулярно создаёт одинаковые
public bool IsValid(string email) =>
    new Regex(@"^[\w.]+@[\w.]+$").IsMatch(email);

// ✅ Compile once
private static readonly Regex _emailRegex = 
    new(@"^[\w.]+@[\w.]+$", RegexOptions.Compiled);

public bool IsValid(string email) => _emailRegex.IsMatch(email);

// ✅✅ С Source Generator (.NET 7+) — zero startup cost
[GeneratedRegex(@"^[\w.]+@[\w.]+$")]
private static partial Regex EmailRegex();

public bool IsValid(string email) => EmailRegex().IsMatch(email);
```

### Pattern: Singleton expensive resources

```csharp
// ❌ HttpClient per request — exhausts ports
public async Task<string> Fetch(string url)
{
    using var client = new HttpClient();
    return await client.GetStringAsync(url);
}

// ✅ Reuse через IHttpClientFactory
public class MyService(IHttpClientFactory factory)
{
    public async Task<string> Fetch(string url)
    {
        using var client = factory.CreateClient();
        return await client.GetStringAsync(url);
    }
}
```

См. [[../AspNetCore/resilience|Resilience]].

---

## 3. Batch patterns

### Pattern: Bulk operations

```csharp
// ❌ N queries
foreach (var user in newUsers)
{
    _db.Users.Add(user);
    await _db.SaveChangesAsync();  // ⚠️ Save в loop!
}

// ✅ One query
_db.Users.AddRange(newUsers);
await _db.SaveChangesAsync();

// ✅✅ EF Core 7+ ExecuteUpdate / ExecuteDelete
await _db.Users.Where(u => u.IsInactive).ExecuteDeleteAsync();
```

### Pattern: Batch HTTP requests

```csharp
// ❌ Sequential — 100 × 50ms = 5 sec
var results = new List<Data>();
foreach (var id in ids)
{
    results.Add(await client.GetAsync($"/api/{id}"));
}

// ✅ Parallel — 50ms total
var tasks = ids.Select(id => client.GetAsync($"/api/{id}"));
var results = await Task.WhenAll(tasks);

// ✅✅ Bounded parallelism (если 1000 ids — не бомбить server)
var semaphore = new SemaphoreSlim(initialCount: 10);
var tasks = ids.Select(async id =>
{
    await semaphore.WaitAsync();
    try { return await client.GetAsync($"/api/{id}"); }
    finally { semaphore.Release(); }
});
var results = await Task.WhenAll(tasks);
```

### Pattern: Batch DB writes

```csharp
// Stack updates and flush periodically
public class BatchedWriter
{
    private readonly List<LogEntry> _buffer = new();
    private readonly Timer _timer;
    
    public BatchedWriter()
    {
        _timer = new Timer(async _ => await Flush(), null, 1000, 1000);
    }
    
    public void Write(LogEntry entry)
    {
        lock (_buffer) _buffer.Add(entry);
        if (_buffer.Count >= 100) _ = Flush();
    }
    
    private async Task Flush()
    {
        List<LogEntry> snapshot;
        lock (_buffer)
        {
            if (_buffer.Count == 0) return;
            snapshot = _buffer.ToList();
            _buffer.Clear();
        }
        
        await _db.Logs.AddRangeAsync(snapshot);
        await _db.SaveChangesAsync();
    }
}
```

---

## 4. Async / Parallel patterns

### Pattern: Parallel.ForEachAsync (.NET 6+)

```csharp
// ❌ Sequential
foreach (var item in items)
    await ProcessAsync(item);

// ✅ Parallel (CPU-bound)
await Parallel.ForEachAsync(items, async (item, ct) =>
{
    await ProcessAsync(item);
});

// ✅ Bounded
await Parallel.ForEachAsync(
    items,
    new ParallelOptions { MaxDegreeOfParallelism = 4 },
    async (item, ct) => await ProcessAsync(item));
```

### Pattern: Task.WhenAll for independent work

```csharp
// ❌ Sequential — total = sum of all
var user = await GetUserAsync(id);
var orders = await GetOrdersAsync(id);
var addr = await GetAddressAsync(id);

// ✅ Parallel — total = max
var userTask = GetUserAsync(id);
var ordersTask = GetOrdersAsync(id);
var addrTask = GetAddressAsync(id);
await Task.WhenAll(userTask, ordersTask, addrTask);

var user = await userTask;
var orders = await ordersTask;
var addr = await addrTask;
```

### Pattern: Task.WhenEach (.NET 9+)

Process tasks по мере завершения:

```csharp
await foreach (var task in Task.WhenEach(tasks))
{
    var result = await task;
    // Обрабатываем первый завершившийся, не ждём всех
    DisplayPartial(result);
}
```

См. [[../CSharp/async-threading|Async и Threading]].

---

## 5. Lazy patterns

### Pattern: Lazy\<T\>

```csharp
// ❌ Expensive init при создании class
public class Service
{
    private readonly Dictionary<string, Config> _configs = LoadAllConfigs();  // 5 sec!
    
    public Config Get(string key) => _configs[key];
}

// ✅ Lazy init при первом обращении
public class Service
{
    private readonly Lazy<Dictionary<string, Config>> _configs = 
        new(() => LoadAllConfigs(), LazyThreadSafetyMode.ExecutionAndPublication);
    
    public Config Get(string key) => _configs.Value[key];
}
```

### Pattern: Lazy loading в EF Core

```csharp
// Eager — загружает всё сразу
var orders = await _db.Orders.Include(o => o.Items).ToListAsync();

// Lazy — загружается при first access
public class Order
{
    public virtual ICollection<Item> Items { get; set; }  // virtual = lazy
}

// Explicit — manual control
var order = await _db.Orders.FindAsync(id);
await _db.Entry(order).Collection(o => o.Items).LoadAsync();
```

См. [[../EFCore/queries-performance|EF Queries]].

### Pattern: IEnumerable vs IList

```csharp
// ❌ Materializes даже если caller не нужен весь list
public List<Item> GetAll() =>
    _items.Where(x => x.IsActive).ToList();

// ✅ Lazy — caller выбирает
public IEnumerable<Item> GetAll() =>
    _items.Where(x => x.IsActive);

// Caller:
var first = service.GetAll().First();  // Stops on first match
```

---

## 6. Eager patterns

Иногда лучше **сделать сразу**, не lazy.

### Pattern: Pre-compute ahead of time

```csharp
// Если знаем что данные понадобятся — pre-compute
public class ReportService
{
    private static readonly Dictionary<int, Decimal> _taxRates;
    
    static ReportService()
    {
        // Computed at startup, не на каждый request
        _taxRates = LoadAndComputeTaxRates();
    }
    
    public decimal GetRate(int countryId) => _taxRates[countryId];
}
```

### Pattern: Eager hydration

```csharp
// ❌ Lazy load каждый field
public class User
{
    public string Name { get; set; }  // OK
    
    public Address Address { get; set; }  // Может быть Lazy<>
    public List<Order> Orders { get; set; }  // Lazy?
}

// ✅ Если всегда нужны — eager (one query)
var user = await _db.Users
    .Include(u => u.Address)
    .Include(u => u.Orders)
    .FirstOrDefaultAsync(u => u.Id == id);
```

---

## 7. Approximate patterns

### Pattern: Sampling

Вместо точного count — sample (если погрешность OK).

```csharp
// ❌ Count exact = полный scan
var totalUsers = await _db.Users.CountAsync();  // 100M rows = slow

// ✅ Approximate count из system tables
var approx = await _db.Database.ExecuteSqlRawAsync(@"
    SELECT reltuples::bigint FROM pg_class WHERE relname = 'users'");
```

### Pattern: HyperLogLog (probabilistic)

Cardinality estimation в Redis с погрешностью ~1%.

```csharp
await _redis.HyperLogLogAddAsync("unique_visitors", visitorId);
var count = await _redis.HyperLogLogLengthAsync("unique_visitors");
// 100M unique IDs занимают 12 KB в Redis вместо ~10 GB!
```

### Pattern: Bloom filter

"Возможно ли значение existует в set?" — без хранения всего set.

```csharp
// "Has user already registered?" — без БД hit
if (!_bloomFilter.MightContain(email))
{
    // Точно нет — не нужно идти в БД
    return false;
}
// Может быть — проверить в БД
return await _db.Users.AnyAsync(u => u.Email == email);
```

См. Bloom filter libraries: BloomFilter.NetCore.

---

## 8. Memory patterns

### Pattern: struct vs class

```csharp
// ❌ Class для small data — heap allocation
public class Point
{
    public int X, Y;
}

// 1M points = 1M heap allocations + GC pressure
var points = Enumerable.Range(0, 1_000_000).Select(i => new Point { X = i, Y = i }).ToList();

// ✅ Struct — stack allocation, contiguous memory
public struct Point
{
    public int X, Y;
}
// 1M points = single contiguous allocation
```

> [!warning] Struct — не всегда лучше
> - Большие structs (>16 bytes) — copy при passing expensive
> - Mutable struct в LINQ — surprises
> - Boxing если cast в interface — ещё хуже class

См. [[../CSharp/types-and-memory|Types & Memory]].

### Pattern: Span\<T\> для slicing без allocation

```csharp
// ❌ Substring создаёт новый string
public bool StartsWithPrefix(string s)
{
    var first10 = s.Substring(0, 10);  // allocation!
    return first10 == "PREFIX_VAL";
}

// ✅ Span — zero allocation
public bool StartsWithPrefix(string s)
{
    var first10 = s.AsSpan(0, 10);
    return first10.Equals("PREFIX_VAL", StringComparison.Ordinal);
}
```

См. [[../Runtime/span-layout|Span deep]].

### Pattern: stackalloc для small temp arrays

```csharp
// ❌ Allocate small array on heap
public int Sum(IEnumerable<int> items)
{
    var arr = items.ToArray();  // allocation
    return arr.Sum();
}

// ✅ Stack allocation если знаем upper bound
public int SumWithSquare(int[] input)
{
    Span<int> squares = stackalloc int[input.Length];
    for (int i = 0; i < input.Length; i++)
        squares[i] = input[i] * input[i];
    
    int sum = 0;
    foreach (var s in squares) sum += s;
    return sum;
}
```

> [!warning] stackalloc только для small (<1 KB)
> Stack overflow если большой. Threshold обычно ~1 KB.

---

## 9. I/O patterns

### Pattern: Sync to async I/O

```csharp
// ❌ Sync I/O блокирует thread
public string ReadConfig()
{
    return File.ReadAllText("config.json");  // Thread blocked
}

// ✅ Async — thread свободен пока ждёт диск
public async Task<string> ReadConfigAsync()
{
    return await File.ReadAllTextAsync("config.json");
}
```

### Pattern: Stream вместо load всё в memory

```csharp
// ❌ Loads весь file в memory
public async Task<List<string>> ReadAll()
{
    var content = await File.ReadAllTextAsync("huge.csv");  // 1 GB into RAM!
    return content.Split('\n').ToList();
}

// ✅ Stream — process line-by-line
public async IAsyncEnumerable<string> ReadAsync()
{
    using var reader = new StreamReader("huge.csv");
    string? line;
    while ((line = await reader.ReadLineAsync()) != null)
        yield return line;
}
```

---

## 10. Database patterns

### Pattern: Index strategy

См. [[../SQL/optimization|SQL Optimization]].

### Pattern: Read replicas

```csharp
// Write — primary
await _writeDbContext.Users.AddAsync(user);
await _writeDbContext.SaveChangesAsync();

// Read — replica (eventually consistent)
var users = await _readDbContext.Users.ToListAsync();
```

### Pattern: Pagination

```csharp
// ❌ Load all
var all = await _db.Orders.ToListAsync();
return all.Skip((page - 1) * size).Take(size).ToList();

// ✅ DB-level pagination
var page = await _db.Orders
    .OrderBy(o => o.CreatedAt)
    .Skip((pageNum - 1) * pageSize)
    .Take(pageSize)
    .ToListAsync();

// ✅✅ Cursor-based (lifeso для huge data)
var page = await _db.Orders
    .Where(o => o.Id > lastSeenId)
    .OrderBy(o => o.Id)
    .Take(pageSize)
    .ToListAsync();
```

См. [[../EFCore/queries-performance|EF Queries]].

---

## 11. Network patterns

### Pattern: Connection pooling

HttpClient, DB connections — reuse pool.

```csharp
// ✅ HttpClientFactory pools internally
builder.Services.AddHttpClient<IMyService, MyService>();

// ✅ EF Core uses connection pool автоматически
```

### Pattern: Compression

```csharp
// ASP.NET Core response compression
builder.Services.AddResponseCompression(options =>
{
    options.Providers.Add<GzipCompressionProvider>();
    options.Providers.Add<BrotliCompressionProvider>();
});

app.UseResponseCompression();
```

Большие JSON responses — 70% reduction.

### Pattern: HTTP/2 + HTTP/3

```csharp
// HTTP/3 (QUIC) — newer, faster в плохих сетях
builder.WebHost.ConfigureKestrel(options =>
{
    options.ListenAnyIP(443, listenOptions =>
    {
        listenOptions.Protocols = HttpProtocols.Http1AndHttp2AndHttp3;
        listenOptions.UseHttps();
    });
});
```

---

## 12. Common Pitfalls

### 1. Optimization без measurement

См. [[performance-fundamentals|Performance Fundamentals]].

### 2. Парallel когда не CPU-bound

```csharp
// ❌ Parallel.ForEach для I/O — workers заблокированы на await
Parallel.ForEach(urls, async url =>
{
    await client.GetAsync(url);
});

// ✅ Task.WhenAll для I/O
var tasks = urls.Select(u => client.GetAsync(u));
await Task.WhenAll(tasks);
```

### 3. Premature struct conversion

```csharp
// "Я слышал struct быстрее"
public struct LargeData  // 100 fields
{
    // ...
}
// Каждый pass = copy 100 fields = SLOWER чем reference!
```

### 4. Cache becomes memory leak

См. [[caching-strategies|Caching Strategies]].

### 5. Async overhead для simple methods

```csharp
// ❌ Async overhead для trivial
public async Task<int> AddAsync(int a, int b) => await Task.FromResult(a + b);

// ✅ Sync если no async work
public int Add(int a, int b) => a + b;
```

### 6. Incorrect parallelism level

```csharp
// ❌ MaxDegreeOfParallelism = unlimited
await Parallel.ForEachAsync(
    huge_collection,
    async (item, ct) => await CallExternalApi(item));
// Может ddos external API

// ✅ Bounded
await Parallel.ForEachAsync(
    items,
    new ParallelOptions { MaxDegreeOfParallelism = 4 },
    async (item, ct) => await CallExternalApi(item));
```

---

## 13. Best Practices

- **Measure first** — без profiling no optimization
- **Algorithm > micro-optimizations** — Big O matters
- **Reuse instead of allocate** — ObjectPool, ArrayPool
- **Batch I/O operations** — один request лучше чем 100
- **Project only needed columns** — не SELECT *
- **Async для I/O, parallel для CPU**
- **Bounded parallelism** — не unlimited
- **Cache wisely** — TTL + invalidation
- **Compress responses** — gzip / brotli
- **Connection pooling** для HTTP / DB
- **Stream large data** — не load всё в memory
- **Profile after** — verify changes улучшили

---

## 14. Декision tree — какой паттерн

```
Slow code?
├── DB queries slow?
│   ├── N+1 → Include/Select
│   ├── SELECT * → Project
│   ├── Missing index → Add index
│   └── Too many results → Pagination
├── External API slow?
│   ├── Sequential → Parallel (Task.WhenAll)
│   ├── Many calls → Batch endpoint
│   └── Rate limited → Caching
├── CPU bound?
│   ├── Algorithm O(n²) → Improve algorithm
│   ├── Single thread → Parallel
│   └── Allocations → ObjectPool / Span
├── Memory pressure?
│   ├── GC pauses → Reduce allocations
│   ├── Large objects → POH (.NET 5+)
│   └── Heap growth → Find leak (gcdump)
└── Startup slow?
    ├── DI graph → Lazy<> для expensive
    ├── Reflection → Source Generators
    └── Container init → Native AOT
```

---

## См. также

- [[performance-fundamentals|Performance Fundamentals]]
- [[caching-strategies|Caching Strategies]]
- [[performance|Performance Deep]]
- [[hft-low-latency|HFT / Low Latency]]
- [[../Runtime/gc-memory|GC и память]]
- [[../Runtime/span-layout|Span / Layout]]
- [[../EFCore/queries-performance|EF Queries Performance]]
- [[../CSharp/async-threading|Async и Threading]]

## Reading list

- **Stephen Toub blog** — devblogs.microsoft.com/dotnet
- **Adam Sitnik blog** — adamsitnik.com
- **Pro .NET Memory Management** — Konrad Kokosa
- **Writing High-Performance .NET Code** — Ben Watson
