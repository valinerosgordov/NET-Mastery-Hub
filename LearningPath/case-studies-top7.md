---
tags: [csharp, case-studies, real-world, examples, async, oop, linq, ef, error-handling, auth, top-files]
level: Middle to Senior
date: 2026-04-30
---

# Case Studies — глубокие реальные сценарии для топ-7 файлов

> **Дополнение к топ-7 наиболее важным файлам vault'а**: реальные production кейсы, какие проблемы возникают, как решаются. Читать после или параллельно с основными файлами.

Этот файл дополняет:
- [[../CSharp/async-threading|async-threading.md]]
- [[../CSharp/oop|oop.md]]
- [[../CSharp/collections-linq|collections-linq.md]]
- [[../CSharp/error-handling|error-handling.md]]
- [[../EFCore/basics-tracking|EFCore/basics-tracking.md]]
- [[../EFCore/queries-performance|EFCore/queries-performance.md]]
- [[../AspNetCore/auth-security|AspNetCore/auth-security.md]]

---

## Часть 1: Async/Await — production cases

### Case Study #1: ThreadPool starvation в production

**Сценарий.** ASP.NET Core API. Под нагрузкой 1000 RPS — внезапно начинаются timeout'ы, p99 latency 30 секунд. CPU не загружен, RAM не растёт. Что происходит?

**Расследование.**
```csharp
public class OrderController : ControllerBase
{
    public async Task<Order> Get(int id)
    {
        var order = _orderService.GetById(id);  // ⚠️ sync method
        return order;
    }
}

public class OrderService
{
    public Order GetById(int id)
    {
        // Внутри — sync HTTP вызов
        var resp = _httpClient.GetAsync($"...").Result;  // ⚠️ .Result!
        return ParseOrder(resp.Content.ReadAsStringAsync().Result);
    }
}
```

**Проблема.** Каждый request:
1. Пришёл → ThreadPool thread берёт его
2. Внутри `.Result` — thread **блокирован** ждёт I/O
3. Ещё 1000 requests тоже блокированы
4. ThreadPool обычно ~32 threads (после auto-scaling — больше, но медленно растёт)
5. **Deadlock-like state** — все threads ждут I/O, новые requests некому handle

**Решение.**
```csharp
public class OrderController : ControllerBase
{
    public async Task<Order> Get(int id)
    {
        return await _orderService.GetByIdAsync(id);  // async all way
    }
}

public class OrderService
{
    public async Task<Order> GetByIdAsync(int id)
    {
        var resp = await _httpClient.GetAsync($"...");
        var content = await resp.Content.ReadAsStringAsync();
        return ParseOrder(content);
    }
}
```

**Результат.** Те же 1000 RPS — без timeout'ов. Threads не блокированы — освобождаются пока I/O ждёт.

**Lesson.** **`async all the way`**. Один `.Result` или `.Wait()` в production = potential disaster.

См. [[../CSharp/async-threading|async-threading.md]] и [[../Runtime/threading-basics|threading-basics.md]].

---

### Case Study #2: ConfigureAwait в библиотеке

**Сценарий.** Library опубликована в NuGet. Работала на ASP.NET Core (нет SyncContext). Кто-то использует её в WPF — deadlock в UI.

**Код library.**
```csharp
public class HttpService
{
    public async Task<string> Get(string url)
    {
        var resp = await _client.GetAsync(url);
        return await resp.Content.ReadAsStringAsync();
    }
}
```

**WPF caller — deadlock.**
```csharp
private void Button_Click(object sender, RoutedEventArgs e)
{
    var result = _service.Get("https://...").Result;  // ⚠️ блокирует UI thread
    // .Result ждёт continuation, continuation ждёт UI thread → deadlock
}
```

**Решение в library.**
```csharp
public async Task<string> Get(string url)
{
    var resp = await _client.GetAsync(url).ConfigureAwait(false);
    // continuation НЕ возвращается на captured context (UI thread)
    return await resp.Content.ReadAsStringAsync().ConfigureAwait(false);
}
```

**Lesson.** В **library code** — всегда `.ConfigureAwait(false)`. В **app code** (ASP.NET Core нет SyncContext) — не нужно.

---

### Case Study #3: Parallel.ForEach + async = wrong

**Сценарий.** Нужно скачать 100 файлов параллельно.

**❌ Wrong.**
```csharp
Parallel.ForEach(urls, async url =>
{
    var data = await DownloadAsync(url);  // не работает корректно!
    File.WriteAllBytes(GetPath(url), data);
});
// Parallel не ждёт async lambdas — некоторые tasks теряются
```

**✅ Correct.**
```csharp
// .NET 6+
await Parallel.ForEachAsync(urls,
    new ParallelOptions { MaxDegreeOfParallelism = 10 },
    async (url, ct) =>
    {
        var data = await DownloadAsync(url, ct);
        await File.WriteAllBytesAsync(GetPath(url), data, ct);
    });

// Pre-.NET 6
var sem = new SemaphoreSlim(10);
var tasks = urls.Select(async url =>
{
    await sem.WaitAsync();
    try
    {
        var data = await DownloadAsync(url);
        await File.WriteAllBytesAsync(GetPath(url), data);
    }
    finally { sem.Release(); }
});
await Task.WhenAll(tasks);
```

**Lesson.** `Parallel.ForEach` — для CPU-bound. Для I/O — `Task.WhenAll` или `Parallel.ForEachAsync` (.NET 6+).

---

## Часть 2: OOP — design decisions

### Case Study #4: Inheritance vs Composition — Plugin system

**Сценарий.** Editor с plugins (формат файлов). Нужно поддерживать .docx, .pdf, .markdown.

**❌ Inheritance.**
```csharp
public abstract class FileEditor
{
    public abstract void Open(string path);
    public abstract void Save(string path);
    public abstract void Render();
}

public class WordEditor : FileEditor { /* docx logic */ }
public class PdfEditor : FileEditor { /* pdf logic */ }
public class MarkdownEditor : FileEditor { /* md logic */ }

// Что если нужно поддержать оба rendering AND validation?
public abstract class FileEditorWithValidation : FileEditor { /* ... */ }
public class WordEditorWithValidation : FileEditorWithValidation { /* ... */ }
// ⚠️ Combinatorial explosion!
```

**✅ Composition.**
```csharp
public interface IFileLoader
{
    Document Load(string path);
}

public interface IFileSaver
{
    void Save(Document doc, string path);
}

public interface IRenderer
{
    void Render(Document doc);
}

public interface IValidator
{
    ValidationResult Validate(Document doc);
}

public class FileEditor
{
    private readonly IFileLoader _loader;
    private readonly IFileSaver _saver;
    private readonly IRenderer _renderer;
    private readonly IValidator? _validator;
    
    public FileEditor(IFileLoader l, IFileSaver s, IRenderer r, IValidator? v = null)
    {
        _loader = l; _saver = s; _renderer = r; _validator = v;
    }
    
    public Document Open(string path) => _loader.Load(path);
    public void Save(Document d, string p) => _saver.Save(d, p);
    public void Render(Document d) => _renderer.Render(d);
    public bool Validate(Document d) => _validator?.Validate(d).IsValid ?? true;
}

// Plug different combinations
var wordEditor = new FileEditor(new DocxLoader(), new DocxSaver(), new WordRenderer(), new WordValidator());
var simpleMd = new FileEditor(new MarkdownLoader(), new MarkdownSaver(), new MarkdownRenderer());
```

**Lesson.** **"Favor composition over inheritance"** (GoF). Inheritance — `is-a` (Manager `is-a` Employee). Composition — `has-a` (Editor `has-a` Loader, Saver, Renderer).

См. [[../CSharp/oop|oop.md]].

---

### Case Study #5: Interface segregation (ISP)

**Сценарий.** Есть `IRepository` который используется везде в коде.

**❌ Fat interface.**
```csharp
public interface IRepository<T>
{
    Task<T?> GetByIdAsync(int id);
    Task<List<T>> GetAllAsync();
    Task<List<T>> SearchAsync(string query);
    Task AddAsync(T entity);
    Task UpdateAsync(T entity);
    Task DeleteAsync(int id);
    Task<int> CountAsync();
    Task<bool> ExistsAsync(int id);
    Task<List<T>> GetPagedAsync(int page, int size);
}

// Что если read-only repository (audit log)?
public class AuditLogRepository : IRepository<AuditLog>
{
    public Task AddAsync(AuditLog e) => /* ... */;
    public Task UpdateAsync(AuditLog e) => throw new NotSupportedException();  // ⚠️
    public Task DeleteAsync(int id) => throw new NotSupportedException();      // ⚠️
}
```

**✅ Segregated interfaces.**
```csharp
public interface IReadRepository<T>
{
    Task<T?> GetByIdAsync(int id);
    Task<List<T>> GetAllAsync();
    Task<bool> ExistsAsync(int id);
    Task<int> CountAsync();
}

public interface IWriteRepository<T>
{
    Task AddAsync(T entity);
    Task UpdateAsync(T entity);
    Task DeleteAsync(int id);
}

public interface ISearchableRepository<T>
{
    Task<List<T>> SearchAsync(string query);
    Task<List<T>> GetPagedAsync(int page, int size);
}

// Audit — только Read + Add
public class AuditLogRepository : IReadRepository<AuditLog>, IWriteRepository<AuditLog>
{
    // Только то, что реально нужно — нет throw NotSupportedException
}

public class UserRepository : IReadRepository<User>, IWriteRepository<User>, ISearchableRepository<User>
{
    // Полный набор
}
```

**Lesson.** Интерфейс должен быть **минимальным для consumer'а**. Лучше много маленьких чем один fat.

---

### Case Study #6: Template Method — Document export pipeline

**Сценарий.** Нужно exporta documents в PDF, HTML, DOCX. Шаги одинаковые: load template → fill data → render → save. Различия — только в render.

```csharp
public abstract class DocumentExporter
{
    // Template method — invariant algorithm
    public async Task ExportAsync(Order order, string outputPath)
    {
        var template = await LoadTemplateAsync();
        ValidateTemplate(template);
        
        var filled = FillData(template, order);
        ValidateData(filled);
        
        var rendered = await RenderAsync(filled);  // ← варьируется
        ValidateOutput(rendered);
        
        await SaveAsync(rendered, outputPath);
        await LogExportAsync(order, outputPath);
    }
    
    protected virtual Task<Template> LoadTemplateAsync() { /* default impl */ }
    protected virtual void ValidateTemplate(Template t) { /* basic check */ }
    protected virtual Document FillData(Template t, Order o) { /* default fill */ }
    protected virtual void ValidateData(Document d) { /* check */ }
    
    protected abstract Task<byte[]> RenderAsync(Document d);  // ← children override
    
    protected virtual void ValidateOutput(byte[] data) { /* size check */ }
    protected virtual Task SaveAsync(byte[] data, string path) => File.WriteAllBytesAsync(path, data);
    protected virtual Task LogExportAsync(Order o, string p) { /* logging */ }
}

public class PdfExporter : DocumentExporter
{
    protected override async Task<byte[]> RenderAsync(Document d)
    {
        // PDF-specific render
        using var pdf = new PdfDocument();
        pdf.Render(d);
        return pdf.ToBytes();
    }
}

public class HtmlExporter : DocumentExporter
{
    protected override async Task<byte[]> RenderAsync(Document d)
    {
        var html = HtmlBuilder.Build(d);
        return Encoding.UTF8.GetBytes(html);
    }
}

public class DocxExporter : DocumentExporter
{
    protected override async Task<byte[]> RenderAsync(Document d) { /* DOCX */ }
    
    // Override default save — DOCX needs special encoding
    protected override async Task SaveAsync(byte[] data, string path)
    {
        // Custom DOCX saving
    }
}
```

**Lesson.** Template Method — когда **алгоритм фиксирован**, но **некоторые шаги варьируются**. Children override только specific steps.

---

## Часть 3: LINQ — performance gotchas

### Case Study #7: Multiple enumeration

**Сценарий.** Filter и потом count и потом iterate.

**❌ Bug — query выполняется 3 раза!**
```csharp
public async Task<Result> Process(int userId)
{
    var orders = _db.Orders.Where(o => o.UserId == userId);  // IQueryable — deferred
    
    var count = orders.Count();           // SQL query #1
    var total = orders.Sum(o => o.Total); // SQL query #2
    
    foreach (var o in orders)              // SQL query #3
    {
        await ProcessAsync(o);
    }
    
    return new Result(count, total);
}
```

**✅ Materialize once.**
```csharp
public async Task<Result> Process(int userId)
{
    var orders = await _db.Orders
        .Where(o => o.UserId == userId)
        .ToListAsync();  // SQL раз
    
    var count = orders.Count;            // memory access
    var total = orders.Sum(o => o.Total); // memory iteration
    
    foreach (var o in orders) { /* ... */ }  // memory iteration
    
    return new Result(count, total);
}
```

**Or aggregate в одном query (если orders large):**
```csharp
var stats = await _db.Orders
    .Where(o => o.UserId == userId)
    .GroupBy(_ => 1)
    .Select(g => new
    {
        Count = g.Count(),
        Total = g.Sum(o => o.Total)
    })
    .FirstAsync();
```

**Lesson.** `IQueryable` lazy — каждый terminal operator (`Count`, `Sum`, `ToList`, `foreach`) выполняет SQL. Materialize один раз!

См. [[../CSharp/collections-linq|collections-linq.md]] и [[../EFCore/queries-performance|EF Queries Performance]].

---

### Case Study #8: `Where` после `OrderBy`

**Сценарий.** Sort + filter.

**❌ Inefficient.**
```csharp
var top10Active = users
    .OrderByDescending(u => u.LastLoginAt)
    .Where(u => u.IsActive)
    .Take(10)
    .ToList();

// На in-memory: sort всех 1M users → filter → take 10
// Sort 1M items — slow!
```

**✅ Filter first.**
```csharp
var top10Active = users
    .Where(u => u.IsActive)        // Filter first — меньше items для sort
    .OrderByDescending(u => u.LastLoginAt)
    .Take(10)
    .ToList();

// Sort только active users (е.g., 100K) — 10x faster
```

В EF — query planner может оптимизировать, но **не всегда**. Пиши efficient порядок сразу.

---

### Case Study #9: `Include` + `Where` на child collections

**Сценарий.** Загрузить orders с их items, фильтровать items.

**❌ Не работает как ожидается.**
```csharp
var orders = await _db.Orders
    .Include(o => o.Items)
    .Where(o => o.Items.Any(i => i.Price > 100))  // фильтр по items на orders
    .ToListAsync();

// Bug: вернёт orders где есть item > 100, НО все items в Order.Items загрузятся!
// Не только те что > 100
```

**Решение — filtered include (EF Core 5+).**
```csharp
var orders = await _db.Orders
    .Where(o => o.Items.Any(i => i.Price > 100))
    .Include(o => o.Items.Where(i => i.Price > 100))  // только expensive items
    .ToListAsync();
```

См. [[../EFCore/queries-performance|EF Queries Performance]].

---

## Часть 4: Error handling — production strategies

### Case Study #10: Result<T> vs Exceptions

**Сценарий.** Validation в API. Failure — expected (user mistake), не error.

**❌ Exceptions для validation.**
```csharp
public async Task<User> Register(RegisterRequest req)
{
    if (string.IsNullOrEmpty(req.Email))
        throw new ValidationException("Email required");
    
    if (await _users.ExistsAsync(req.Email))
        throw new ConflictException("Email already registered");
    
    if (req.Password.Length < 8)
        throw new ValidationException("Password too short");
    
    return await _users.CreateAsync(req);
}

// Controller
[HttpPost]
public async Task<IActionResult> Register(RegisterRequest req)
{
    try
    {
        var user = await _service.Register(req);
        return Ok(user);
    }
    catch (ValidationException e) { return BadRequest(e.Message); }
    catch (ConflictException e) { return Conflict(e.Message); }
    // ⚠️ Exception flow для нормального case — slow, ugly
}
```

**✅ Result<T, E>.**
```csharp
public sealed record Result<T>
{
    public T? Value { get; init; }
    public string? Error { get; init; }
    public ErrorType ErrorType { get; init; }
    
    public bool IsSuccess => Error is null;
    
    public static Result<T> Ok(T value) => new() { Value = value };
    public static Result<T> Fail(string error, ErrorType type = ErrorType.BadRequest) 
        => new() { Error = error, ErrorType = type };
}

public enum ErrorType { BadRequest, Conflict, NotFound, Forbidden }

public async Task<Result<User>> Register(RegisterRequest req)
{
    if (string.IsNullOrEmpty(req.Email))
        return Result<User>.Fail("Email required");
    
    if (req.Password.Length < 8)
        return Result<User>.Fail("Password too short");
    
    if (await _users.ExistsAsync(req.Email))
        return Result<User>.Fail("Email already registered", ErrorType.Conflict);
    
    var user = await _users.CreateAsync(req);
    return Result<User>.Ok(user);
}

// Controller — clean mapping
[HttpPost]
public async Task<IActionResult> Register(RegisterRequest req)
{
    var result = await _service.Register(req);
    
    return result.IsSuccess
        ? Ok(result.Value)
        : result.ErrorType switch
        {
            ErrorType.Conflict => Conflict(result.Error),
            ErrorType.NotFound => NotFound(result.Error),
            ErrorType.Forbidden => Forbid(),
            _ => BadRequest(result.Error)
        };
}
```

**Lesson.** **Exceptions для exceptional**. Validation failures — это normal flow. Result<T> делает их explicit.

См. [[../CSharp/error-handling|error-handling.md]] и [[../Snippets/result-pattern|Result Pattern Snippet]].

---

### Case Study #11: Retry + Circuit Breaker (Polly)

**Сценарий.** API зависит от external payment service. Бывает downtime.

**❌ Naive retry.**
```csharp
public async Task<PaymentResult> Charge(Order order)
{
    for (int i = 0; i < 3; i++)
    {
        try
        {
            return await _paymentApi.ChargeAsync(order);
        }
        catch (HttpRequestException)
        {
            await Task.Delay(1000);  // wait 1 sec
        }
    }
    throw new PaymentFailedException();
}

// Проблемы:
// 1. Retry на ANY exception — может делать дублирующие charges!
// 2. Нет exponential backoff — 3x загружает DOS-style при downtime
// 3. Нет circuit breaker — все requests retrying даже если payment service down 10 минут
```

**✅ Polly.**
```csharp
// Setup в Startup.cs
services.AddHttpClient<IPaymentApi, PaymentApi>()
    .AddTransientHttpErrorPolicy(p => p
        .WaitAndRetryAsync(
            retryCount: 3,
            sleepDurationProvider: attempt => TimeSpan.FromSeconds(Math.Pow(2, attempt))))  // 2, 4, 8 sec
    .AddTransientHttpErrorPolicy(p => p
        .CircuitBreakerAsync(
            handledEventsAllowedBeforeBreaking: 5,
            durationOfBreak: TimeSpan.FromMinutes(1)));

// Code просто использует
public async Task<PaymentResult> Charge(Order order)
{
    return await _paymentApi.ChargeAsync(order);
    // Polly handle retries + circuit breaker автоматически
}
```

**Что делает Polly.**
- **Retry** — exponential backoff (2, 4, 8 sec)
- **Circuit Breaker** — после 5 failures блокирует все вызовы на 1 минуту → не DDoS payment service
- **Idempotency** — конкретные scenarios (TransientHttpError) не дублируют side effects

См. [[../AspNetCore/resilience|Resilience]].

---

## Часть 5: EF Core — production gotchas

### Case Study #12: Tracking vs No-Tracking

**Сценарий.** Reading API endpoint — performance trouble.

**❌ С tracking (default).**
```csharp
[HttpGet("{id}")]
public async Task<User> Get(int id)
{
    return await _db.Users
        .Include(u => u.Orders)
        .Include(u => u.Address)
        .FirstAsync(u => u.Id == id);
}

// EF tracker создаёт snapshot для всех loaded entities
// На каждом property access — change detection
// Для read-only — overhead 30-50%
```

**✅ AsNoTracking для reads.**
```csharp
[HttpGet("{id}")]
public async Task<User> Get(int id)
{
    return await _db.Users
        .AsNoTracking()  // ← без tracking
        .Include(u => u.Orders)
        .Include(u => u.Address)
        .FirstAsync(u => u.Id == id);
}

// 30-50% faster — нет snapshots, нет change detection
```

**Когда tracking нужен:**
- Save changes после изменения
- Lazy loading

**Когда AsNoTracking:**
- Read-only API endpoints (90% cases в Web API)
- Reports / analytics
- Read replica scenarios

См. [[../EFCore/basics-tracking|EFCore basics-tracking.md]].

---

### Case Study #13: N+1 проблема

**Сценарий.** Order list page — slow.

**❌ N+1 query.**
```csharp
public async Task<List<OrderDto>> GetOrders()
{
    var orders = await _db.Orders.ToListAsync();  // 1 query
    
    var dtos = new List<OrderDto>();
    foreach (var order in orders)
    {
        // ⚠️ Lazy loading или explicit fetch — N queries!
        var customer = await _db.Customers.FindAsync(order.CustomerId);
        dtos.Add(new OrderDto { OrderNumber = order.Number, CustomerName = customer.Name });
    }
    
    return dtos;
}

// 1 + N queries. 100 orders → 101 queries. Slow!
```

**✅ Include или Projection.**
```csharp
// Option A — Include
public async Task<List<OrderDto>> GetOrders()
{
    var orders = await _db.Orders
        .Include(o => o.Customer)
        .ToListAsync();  // 1 query с JOIN
    
    return orders.Select(o => new OrderDto
    {
        OrderNumber = o.Number,
        CustomerName = o.Customer.Name
    }).ToList();
}

// Option B — Projection (быстрее, меньше data)
public async Task<List<OrderDto>> GetOrders()
{
    return await _db.Orders
        .Select(o => new OrderDto
        {
            OrderNumber = o.Number,
            CustomerName = o.Customer.Name  // EF делает JOIN автоматически
        })
        .ToListAsync();  // 1 query
}
```

**Lesson.** Включи **`AsSplitQuery`** или **`AsSingleQuery`** explicit для понятности. Используй EF logging чтобы видеть SQL.

См. [[../EFCore/queries-performance|EFCore queries-performance.md]].

---

### Case Study #14: Bulk update — проблема с EF до 7

**Сценарий.** Деактивировать 1M users by some criteria.

**❌ Pre EF Core 7.**
```csharp
var inactive = await _db.Users
    .Where(u => u.LastLoginAt < DateTime.UtcNow.AddYears(-1))
    .ToListAsync();  // Load 1M в memory!

foreach (var u in inactive)
{
    u.IsActive = false;
}

await _db.SaveChangesAsync();
// 1M UPDATE statements!
```

**✅ EF Core 7+ ExecuteUpdate.**
```csharp
await _db.Users
    .Where(u => u.LastLoginAt < DateTime.UtcNow.AddYears(-1))
    .ExecuteUpdateAsync(s => s.SetProperty(u => u.IsActive, false));

// Один SQL UPDATE statement, ноль данных в memory
```

**Old way (до EF 7) — RawSQL или Dapper.**
```csharp
await _db.Database.ExecuteSqlAsync($@"
    UPDATE Users 
    SET IsActive = 0 
    WHERE LastLoginAt < {DateTime.UtcNow.AddYears(-1)}");
```

См. [[../EFCore/dapper-comparison|Dapper vs EF]].

---

## Часть 6: Auth & Security — common mistakes

### Case Study #15: JWT — secret key management

**❌ Hardcoded secret.**
```csharp
// Program.cs
builder.Services.AddAuthentication("Bearer")
    .AddJwtBearer(options =>
    {
        options.TokenValidationParameters = new()
        {
            ValidIssuer = "myapp",
            IssuerSigningKey = new SymmetricSecurityKey(
                Encoding.UTF8.GetBytes("mySecretKey123")  // ⚠️ В коде!
            )
        };
    });

// Утечка в git → все JWT можно forge
```

**✅ Secure secrets.**
```csharp
// Development — User Secrets
// dotnet user-secrets init
// dotnet user-secrets set "Jwt:Secret" "actual-secret-here"

// Production — Azure Key Vault / AWS Secrets Manager / HashiCorp Vault
// Не commit'ить!

builder.Services.AddAuthentication("Bearer")
    .AddJwtBearer(options =>
    {
        var secret = builder.Configuration["Jwt:Secret"]
            ?? throw new InvalidOperationException("JWT secret not configured");
        
        options.TokenValidationParameters = new()
        {
            ValidIssuer = builder.Configuration["Jwt:Issuer"],
            ValidAudience = builder.Configuration["Jwt:Audience"],
            ValidateIssuerSigningKey = true,
            IssuerSigningKey = new SymmetricSecurityKey(Encoding.UTF8.GetBytes(secret)),
            ValidateLifetime = true,
            ClockSkew = TimeSpan.FromSeconds(30)  // tighter than default 5 min
        };
    });
```

**Critical mandatory.**
- Secret >= 256 bits (32 bytes)
- Different per environment (dev/staging/prod)
- Rotate periodically
- **NEVER** в git!

См. [[../AspNetCore/auth-security|auth-security.md]].

---

### Case Study #16: Password hashing — bcrypt vs PBKDF2

**❌ Plain SHA256.**
```csharp
public string HashPassword(string password)
{
    using var sha = SHA256.Create();
    var hash = sha.ComputeHash(Encoding.UTF8.GetBytes(password));
    return Convert.ToBase64String(hash);
}

// ⚠️ Без soli — rainbow tables работают
// ⚠️ SHA256 быстрый — 1B hashes/sec → bruteforce easy
```

**✅ ASP.NET Core Identity (PBKDF2).**
```csharp
public class PasswordHasher
{
    public string Hash(string password)
    {
        // ASP.NET Core Identity standard
        byte[] salt = RandomNumberGenerator.GetBytes(128 / 8);
        byte[] hash = KeyDerivation.Pbkdf2(
            password: password,
            salt: salt,
            prf: KeyDerivationPrf.HMACSHA256,
            iterationCount: 100_000,  // tunable — slower = stronger
            numBytesRequested: 256 / 8);
        
        // Format: {iterations}.{salt}.{hash}
        return $"100000.{Convert.ToBase64String(salt)}.{Convert.ToBase64String(hash)}";
    }
    
    public bool Verify(string password, string hashed)
    {
        var parts = hashed.Split('.');
        var iterations = int.Parse(parts[0]);
        var salt = Convert.FromBase64String(parts[1]);
        var expected = Convert.FromBase64String(parts[2]);
        
        var actual = KeyDerivation.Pbkdf2(password, salt, KeyDerivationPrf.HMACSHA256, iterations, 256 / 8);
        
        return CryptographicOperations.FixedTimeEquals(actual, expected);
    }
}
```

**Or use `Microsoft.AspNetCore.Identity.PasswordHasher<TUser>`** — built-in, battle-tested.

См. [[../AspNetCore/auth-security|auth-security.md]].

---

### Case Study #17: SQL injection — even с Dapper

**❌ String interpolation.**
```csharp
public async Task<User?> Find(string username)
{
    var sql = $"SELECT * FROM Users WHERE Username = '{username}'";  // ⚠️
    return await _conn.QueryFirstOrDefaultAsync<User>(sql);
}

// Attacker passes: ' OR '1'='1
// SQL becomes: SELECT * FROM Users WHERE Username = '' OR '1'='1'
// Returns ALL users!
```

**✅ Parameters.**
```csharp
public async Task<User?> Find(string username)
{
    return await _conn.QueryFirstOrDefaultAsync<User>(
        "SELECT * FROM Users WHERE Username = @Username",
        new { Username = username });
}

// Dapper passes username as parameter — sanitized
```

EF Core делает это автоматом, но в Dapper / `ExecuteSqlRaw` — easy mistake.

См. [[../AspNetCore/security-practices|Security Practices]] и [[../EFCore/dapper-comparison|Dapper vs EF]].

---

## Index — какой case изучать

### По теме

| Тема | Cases |
|------|-------|
| Async / threading | #1 ThreadPool starvation, #2 ConfigureAwait, #3 Parallel + async |
| OOP design | #4 Inheritance vs Composition, #5 ISP, #6 Template Method |
| LINQ / collections | #7 Multiple enumeration, #8 Where after OrderBy, #9 Include + Where |
| Error handling | #10 Result vs Exceptions, #11 Polly retry+CB |
| EF Core | #12 Tracking, #13 N+1, #14 Bulk update |
| Auth / Security | #15 JWT secrets, #16 Password hashing, #17 SQL injection |

### По boom-проблеме

| Симптом | Case |
|---------|------|
| Production timeouts | #1 ThreadPool starvation |
| WPF / desktop deadlock | #2 ConfigureAwait |
| Slow page (lazy loading) | #13 N+1 |
| API slow на reads | #12 Tracking |
| Bulk operations slow | #14 Bulk update |
| User input crashes | #17 SQL injection |

---

## См. также

- [[../CSharp/async-threading|async-threading.md]] — fundamentals async
- [[../CSharp/oop|oop.md]] — OOP basics
- [[../CSharp/collections-linq|collections-linq.md]] — LINQ deep
- [[../CSharp/error-handling|error-handling.md]] — strategies
- [[../EFCore/basics-tracking|EFCore basics-tracking.md]]
- [[../EFCore/queries-performance|EFCore queries-performance.md]]
- [[../AspNetCore/auth-security|AspNetCore auth-security.md]]
- [[../AspNetCore/resilience|Resilience]] — Polly deep
- [[../Architecture/real-world-scenarios|Real-World Scenarios]] — system-level cases
- [[../Architecture/patterns-decision-guide|Patterns Decision Guide]] — выбор pattern

## Reading list

- **Stephen Cleary — Concurrency in C#** (книга про async)
- **Mark Seemann — Dependency Injection in .NET** (composition vs inheritance)
- **Vladimir Khorikov — Domain Modeling Made Functional** (Result patterns)
- **Microsoft eShopOnContainers** (real-world reference architecture)
