---
tags: [senior, tips, patterns, cheatsheet]
level: Senior
date: 2026-08-02
---

# C# Senior-Level Tips, Patterns & Anti-Patterns

> Концентрат Senior-практик: allocation-free паттерны (`Span<T>`, `ArrayPool<T>`, `FrozenDictionary`), async-дисциплина, DI, source generators, pattern matching, тестирование и security-чеклист — с готовыми сниппетами и явными anti-patterns.

Собрано из: опыт, каналы (@csharp_ci, .NET Blog), Microsoft Learn, community best practices.

---

## 1. Performance & Allocation-Free Patterns

### `Span<T>` / `Memory<T>` -- горячие пути

```csharp
// Parsing без аллокаций
ReadOnlySpan<char> input = "key=value";
int idx = input.IndexOf('=');
var key = input[..idx];            // ReadOnlySpan<char>, zero alloc
var value = input[(idx + 1)..];

// String operations
ReadOnlySpan<char> trimmed = line.AsSpan().Trim();
if (trimmed.StartsWith("//")) return; // no string allocation
```

### `ArrayPool<T>` -- переиспользование буферов

```csharp
var buffer = ArrayPool<byte>.Shared.Rent(4096);
try
{
    int bytesRead = await stream.ReadAsync(buffer.AsMemory(0, 4096), ct);
    ProcessData(buffer.AsSpan(0, bytesRead));
}
finally
{
    ArrayPool<byte>.Shared.Return(buffer, clearArray: true);
}
```

### FrozenDictionary / FrozenSet (read-heavy, write-once)

```csharp
// Создается один раз, читается миллионы раз
private static readonly FrozenDictionary<string, Handler> Handlers =
    new Dictionary<string, Handler>
    {
        ["create"] = new CreateHandler(),
        ["update"] = new UpdateHandler(),
        ["delete"] = new DeleteHandler(),
    }.ToFrozenDictionary(StringComparer.OrdinalIgnoreCase);
```

### ValueTask -- когда результат часто синхронный

```csharp
// Кеш попадает чаще, чем промахивается -> ValueTask
public ValueTask<User?> GetByIdAsync(Guid id, CancellationToken ct)
{
    if (_cache.TryGetValue(id, out var user))
        return ValueTask.FromResult<User?>(user);

    return LoadFromDatabaseAsync(id, ct);
}
```

### params `ReadOnlySpan<T>` -- zero-alloc variadic

```csharp
// C# 14: params с Span — нет скрытой аллокации массива
public static void Log(params ReadOnlySpan<string> messages)
{
    foreach (var msg in messages)
        Console.WriteLine(msg);
}
```

### SearchValues<T> (.NET 8+) -- быстрый поиск символов

```csharp
private static readonly SearchValues<char> Vowels =
    SearchValues.Create("aeiouAEIOU");

public static int CountVowels(ReadOnlySpan<char> text)
{
    int count = 0;
    foreach (var ch in text)
        if (Vowels.Contains(ch)) count++;
    return count;
}
```

### CompositeFormat -- предкомпилированные строки

```csharp
private static readonly CompositeFormat LogFormat =
    CompositeFormat.Parse("Order {0} processed in {1}ms");

// Использование (zero-alloc с string.Create)
string msg = string.Format(CultureInfo.InvariantCulture, LogFormat, orderId, elapsed);
```

---

## 2. Async / Await Best Practices

### Правила
1. **async all the way** -- не блокировать `.Result`, `.Wait()`, `.GetAwaiter().GetResult()`
2. **CancellationToken** -- всегда передавать, всегда проверять
3. **ConfigureAwait(false)** -- в библиотечном коде (не в UI)
4. **ValueTask** -- для горячих путей, где результат часто синхронный
5. **async void** -- ТОЛЬКО для event handlers, везде `async Task`

### IAsyncEnumerable -- стриминг данных

```csharp
public async IAsyncEnumerable<OrderDto> StreamOrdersAsync(
    [EnumeratorCancellation] CancellationToken ct)
{
    await foreach (var order in context.Orders.AsAsyncEnumerable().WithCancellation(ct))
    {
        yield return order.ToDto();
    }
}

// Клиент: обработка по мере получения
await foreach (var order in client.StreamOrdersAsync(ct))
{
    ProcessOrder(order);
}
```

### Parallel async (ограниченный параллелизм)

```csharp
await Parallel.ForEachAsync(items, new ParallelOptions
{
    MaxDegreeOfParallelism = 4,
    CancellationToken = ct
}, async (item, token) =>
{
    await ProcessAsync(item, token);
});
```

### Task.WhenAll vs Task.WhenEach (.NET 10)

```csharp
// .NET 10: Task.WhenEach — обработка по мере завершения
await foreach (var task in Task.WhenEach(tasks))
{
    var result = await task;
    // обрабатываем первый завершившийся
}
```

### Anti-patterns

```csharp
// BAD: deadlock risk
var result = GetDataAsync().Result;

// BAD: fire-and-forget без обработки ошибок
_ = DoSomethingAsync();

// GOOD: fire-and-forget с обработкой
_ = DoSomethingAsync().ContinueWith(
    t => logger.LogError(t.Exception, "Background task failed"),
    TaskContinuationOptions.OnlyOnFaulted);
```

### ThreadPool Starvation -- sync-over-async обманывает hill-climbing

Блокирующее ожидание (`.Result`, `.Wait()`) внутри work-item паркует поток пула на всё время I/O. Hill-climbing меряет throughput по числу завершённых work-item, не по полезной работе CPU, поэтому добавляет потоки темпом ~4/сек, а не мгновенно. Сигнатура аварии: throughput падает, p99 уходит в секунды, `ThreadCount` растёт «лестницей», а CPU при этом ~8%. `SetMinThreads` лечит симптом, не причину — убирай блокировку (`await` all the way) и изолируй опасные пути bulkhead'ом. Подробно: [[threadpool-starvation-hill-climbing\|threadpool-starvation-hill-climbing]].

---

## 3. Dependency Injection -- Senior Patterns

### Keyed Services (.NET 8+)

```csharp
// Registration
builder.Services.AddKeyedSingleton<INotifier, EmailNotifier>("email");
builder.Services.AddKeyedSingleton<INotifier, SmsNotifier>("sms");
builder.Services.AddKeyedSingleton<INotifier, PushNotifier>("push");

// Injection
public sealed class NotificationService(
    [FromKeyedServices("email")] INotifier emailNotifier,
    [FromKeyedServices("sms")] INotifier smsNotifier)
{
    // ...
}
```

### Decorator pattern через DI

```csharp
builder.Services.AddScoped<IOrderRepository, OrderRepository>();
builder.Services.Decorate<IOrderRepository, CachedOrderRepository>();
// Requires Scrutor NuGet package
```

### Anti-patterns DI
- **Service Locator** -- не инжектить `IServiceProvider` в бизнес-логику
- **Captive Dependency** -- Scoped сервис в Singleton = утечка
- **Overuse of keyed services** -- если `IServiceProvider` для resolve keyed -> design smell

---

## 4. Source Generators & Interceptors

### Source Generator (compile-time codegen)

```csharp
// Генерация маппинга DTO -> Entity в compile-time
// Используй: Mapperly, AutoMapper.Extensions.EnumMapping
[Mapper]
public static partial class OrderMapper
{
    public static partial OrderDto ToDto(this Order order);
    public static partial Order ToEntity(this CreateOrderCommand command);
}
```

### Interceptors (C# 13+, experimental)
- Перехватывают вызовы методов в compile-time
- Используются фреймворками (ASP.NET, EF Core) для AOT-совместимости
- Не для прикладного кода напрямую

### JSON Source Generation (.NET 8+)

```csharp
[JsonSerializable(typeof(OrderDto))]
[JsonSerializable(typeof(List<OrderDto>))]
[JsonSourceGenerationOptions(PropertyNamingPolicy = JsonKnownNamingPolicy.CamelCase)]
public partial class AppJsonContext : JsonSerializerContext;

// Usage: zero-reflection, AOT-compatible
var json = JsonSerializer.Serialize(order, AppJsonContext.Default.OrderDto);
```

---

## 5. Pattern Matching -- Advanced

```csharp
// Relational + property patterns
var discount = (customer, order) switch
{
    ({ Tier: CustomerTier.Gold }, { Total: > 1000m })     => 0.15m,
    ({ Tier: CustomerTier.Gold }, _)                       => 0.10m,
    ({ Tier: CustomerTier.Silver }, { Total: > 500m })     => 0.07m,
    (_, { Items.Count: > 10 })                             => 0.05m,
    _                                                       => 0m,
};

// List patterns (C# 11+)
int[] numbers = [1, 2, 3, 4, 5];
var result = numbers switch
{
    [1, .., 5]     => "starts with 1, ends with 5",
    [_, _, 3, ..]  => "third element is 3",
    []             => "empty",
    _              => "other",
};
```

---

## 6. Architecture Anti-Patterns

### ЗАПРЕЩЕНО
1. **Anemic Domain Model** -- Entity без поведения, вся логика в сервисах
2. **God Service** -- один сервис на 2000+ строк
3. **Repository over Repository** -- обертка над EF Core без добавленной ценности (для CRUD)
4. **Exception-driven flow** -- `throw new NotFoundException()` вместо `Result.Fail()`
5. **Mixing Commands and Queries** -- запись в Query handler
6. **Leaking Domain** -- Entity в API response напрямую

### ПРАВИЛЬНО
1. **Rich Domain Model** -- Entity содержит бизнес-логику и инварианты
2. **Vertical Slices** -- feature folder вместо layer folder
3. **Result<T>** -- все ошибки как значения
4. **Domain Events** -- реакции через MediatR notifications
5. **Specification Pattern** -- сложные запросы инкапсулируются

### Agent-Safe Architecture -- границы, переживающие AI-codegen

Когда код пишет (и переписывает) AI-агент, на прозу полагаться нельзя: `AGENTS.md` гниёт под context rot, граница «в комментарии» для агента не существует. Держат только физические границы (отдельные проекты — нарушение не компилируется) и CI-gate'ы (NetArchTest fitness functions + human-owned contract tests, которые агенту нельзя переписывать под свою реализацию). Принцип: где границы нет — соединяется всё со всем, coupling растёт монотонно. Подробно: [[agent-safe-architecture\|agent-safe-architecture]].

---

## 7. Testing -- Senior Approach

### Integration > Unit (для backend)

```csharp
// WebApplicationFactory + Testcontainers
public sealed class OrderApiTests(WebAppFactory factory) : IClassFixture<WebAppFactory>
{
    [Fact]
    public async Task CreateOrder_ValidRequest_Returns201()
    {
        // Arrange
        var client = factory.CreateClient();
        var command = new { CustomerId = Guid.NewGuid(), Items = new[] { /* ... */ } };

        // Act
        var response = await client.PostAsJsonAsync("/api/orders", command);

        // Assert
        response.StatusCode.Should().Be(HttpStatusCode.Created);
        var body = await response.Content.ReadFromJsonAsync<OrderResponse>();
        body!.Id.Should().NotBeEmpty();
    }
}
```

### Architecture Tests (NetArchTest)

```csharp
[Fact]
public void Domain_ShouldNotReference_Infrastructure()
{
    Types.InAssembly(typeof(Order).Assembly)
        .ShouldNot()
        .HaveDependencyOn("MyApp.Infrastructure")
        .GetResult()
        .IsSuccessful
        .Should().BeTrue();
}
```

### Snapshot Testing (Verify)

```csharp
[Fact]
public Task OrderDto_Serialization_MatchesSnapshot()
{
    var dto = new OrderDto { Id = Guid.Empty, Status = "Active" };
    return Verify(dto);
}
```

---

## 8. Useful NuGet Packages (Senior toolkit)

| Package | Purpose |
|---------|---------|
| **Mapperly** | Compile-time mapping (source generator) |
| **Polly** | Resilience: retry, circuit breaker, timeout |
| **Scrutor** | DI decoration, assembly scanning |
| **FluentResults** | Production-ready Result<T> |
| **BenchmarkDotNet** | Micro-benchmarking |
| **Bogus** | Fake data generation for tests |
| **Verify** | Snapshot testing |
| **Refit** | Type-safe HTTP client from interfaces |
| **Serilog** | Structured logging (alternative to built-in) |
| **MassTransit** | Message bus abstraction |
| **Aspire** | Cloud-native orchestration |
| **Wolverine** | Alternative to MediatR (command bus + messaging) |

---

## 9. Structured Logging -- правила

```csharp
// ПРАВИЛЬНО: message template с именованными параметрами
logger.LogInformation("Order {OrderId} created for {CustomerId}", order.Id, customer.Id);

// НЕПРАВИЛЬНО: string interpolation (нет structured data)
logger.LogInformation($"Order {order.Id} created for {customer.Id}");

// НЕПРАВИЛЬНО: конкатенация
logger.LogInformation("Order " + order.Id + " created");

// High-performance logging (.NET 8+)
[LoggerMessage(Level = LogLevel.Information, Message = "Order {OrderId} created for {CustomerId}")]
public static partial void LogOrderCreated(this ILogger logger, Guid orderId, Guid customerId);
```

---

## 10. Security Checklist

- **SQL Injection**: параметризованные запросы всегда, `FromSqlInterpolated` в EF Core
- **XSS**: Razor автоматически экранирует, но `@Html.Raw()` — осторожно
- **CSRF**: `[ValidateAntiForgeryToken]` или SameSite cookies
- **Secrets**: User Secrets / Azure Key Vault, никогда в `appsettings.json`
- **Auth**: JWT + Refresh tokens, `[Authorize]` по умолчанию
- **CORS**: минимальные origins, никогда `AllowAnyOrigin()` в production
- **Rate Limiting**: `builder.Services.AddRateLimiter()` (.NET 7+)
