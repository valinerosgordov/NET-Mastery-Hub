---
tags: [aspnet, logging, observability, opentelemetry, structured-logging]
level: Senior
date: 2026-08-02
---

# Logging, Observability и диагностика

## Что это, зачем и когда

### Что такое логирование?
Запись событий, происходящих в приложении: кто запросил что, сколько это заняло, какая ошибка произошла.

**Аналогия:** Дневник капитана корабля. Записывает ВСЁ: курс, скорость, погоду, инциденты. Если корабль сядет на мель — дневник покажет почему.

### Зачем?
- **Debugging** — «почему заказ не создался?» → смотришь логи, видишь: «валидация упала, email невалидный»
- **Мониторинг** — «сколько ошибок за последний час?» → алерт если больше нормы
- **Аудит** — «кто удалил пользователя?» → лог с userId и timestamp

### Structured vs обычное логирование

| Обычное | Structured |
|---------|-----------|
| `$"Order {id} created for {name}"` | `"Order {OrderId} created for {CustomerName}", id, name` |
| Текстовая строка — искать невозможно | Поля OrderId, CustomerName — можно фильтровать в Seq/Kibana |
| `logger.LogInformation($"...")` — строка создаётся ВСЕГДА | `logger.LogInformation("...", args)` — строка создаётся ТОЛЬКО если уровень включён |

### Когда что?

| Инструмент | Когда |
|-----------|-------|
| `ILogger<T>` | Обычный код (99% случаев) |
| `[LoggerMessage]` source generator | Hot path (1000+ вызовов/сек) — нулевые аллокации |
| Serilog → Seq | Структурированный поиск по логам |
| OpenTelemetry → Jaeger | Трейсы — путь запроса через сервисы |
| Prometheus → Grafana | Метрики — дашборды, алерты |

---

> [!question]- **Интервью: Structured logging — зачем? Message Templates?**
> Structured logging сохраняет данные как key-value, а не как строку. Позволяет фильтровать, искать, агрегировать по полям (UserId, OrderId). Message Templates: `Log.Information("Order {OrderId} processed", orderId)` — `OrderId` становится свойством, не частью строки.

> [!question]- **Интервью: Логи vs метрики vs трейсы?**
> **Логи** — дискретные события с контекстом. **Метрики** — числовые агрегаты (counters, histograms). **Трейсы** — путь запроса через сервисы (spans). Логи — для debugging. Метрики — для alerting. Трейсы — для distributed performance analysis.

## Система логирования в ASP.NET Core

ASP.NET Core имеет встроенную абстракцию логирования через `ILogger<T>`. Поддерживает несколько провайдеров одновременно и конфигурацию уровней логирования.

### Конфигурация

```csharp
// Program.cs — провайдеры
builder.Logging.ClearProviders(); // Убрать defaults
builder.Logging.AddConsole();
builder.Logging.AddDebug();
builder.Logging.AddEventSourceLogger(); // ETW / dotnet-trace
```

```json
// appsettings.json — уровни по namespace
{
  "Logging": {
    "LogLevel": {
      "Default": "Information",
      "Microsoft.AspNetCore": "Warning",
      "Microsoft.EntityFrameworkCore.Database.Command": "Information",
      "MyApp.Services": "Debug"
    }
  }
}
```

### Уровни логирования

| Уровень | Когда использовать |
|---------|-------------------|
| `Trace` | Детальная отладка, трассировка потока выполнения |
| `Debug` | Отладочная информация, полезна при разработке |
| `Information` | Нормальный поток: запуск, завершение операций |
| `Warning` | Аномалии, которые не являются ошибками (retry, fallback) |
| `Error` | Ошибка, операция не выполнена, но приложение продолжает работу |
| `Critical` | Фатальная ошибка, приложение может упасть |
| `None` | Отключить логирование для категории |

### Использование `ILogger<T>`

```csharp
public class OrderService
{
    private readonly ILogger<OrderService> _logger;

    public OrderService(ILogger<OrderService> logger) => _logger = logger;

    public async Task<Order> CreateOrderAsync(CreateOrderDto dto)
    {
        _logger.LogInformation("Creating order for customer {CustomerId}", dto.CustomerId);

        try
        {
            var order = await _repo.CreateAsync(dto);
            _logger.LogInformation("Order {OrderId} created successfully", order.Id);
            return order;
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Failed to create order for customer {CustomerId}", dto.CustomerId);
            throw;
        }
    }
}
```

### Сторонние провайдеры

| Провайдер | Назначение |
|-----------|-----------|
| **Serilog** | Structured logging, множество sinks (File, Seq, Elasticsearch) |
| **NLog** | Гибкая конфигурация, targets, layouts |
| **Application Insights** | Azure-интеграция, distributed tracing |

```csharp
// Serilog — подключение
builder.Host.UseSerilog((context, config) =>
{
    config
        .ReadFrom.Configuration(context.Configuration)
        .WriteTo.Console()
        .WriteTo.Seq("http://localhost:5341")
        .Enrich.FromLogContext()
        .Enrich.WithMachineName()
        .Enrich.WithEnvironmentName();
});
```

---

## Structured Logging

Логи как структурированные данные (пары ключ-значение), а не просто строки. Это позволяет эффективно искать и фильтровать в ELK, Seq, Application Insights.

### Правильный подход

```csharp
// ПРАВИЛЬНО — structured logging, параметры как свойства
_logger.LogInformation("User {UserId} ordered {OrderId} with {ItemCount} items",
    userId, orderId, items.Count);
// В Seq/ELK: UserId=123, OrderId=456, ItemCount=3

// НЕПРАВИЛЬНО — строковая интерполяция, теряется структура
_logger.LogInformation($"User {userId} ordered {orderId}");
// Невозможно искать по UserId в системе логирования
```

### Message Templates

- `{PropertyName}` — именованный placeholder, становится свойством в structured log
- `{@Object}` — destructure объект (сериализовать свойства)
- `{$Object}` — вызвать ToString()
- Порядок аргументов должен соответствовать порядку placeholders

### High-performance Logging

Для hot path — используйте `LoggerMessage.Define` или source generators, чтобы избежать boxing и аллокации строк:

```csharp
// Source generator (.NET 6+) — компилятор генерирует оптимальный код
public static partial class LogMessages
{
    [LoggerMessage(Level = LogLevel.Information, Message = "Order {OrderId} created for {CustomerId}")]
    public static partial void OrderCreated(ILogger logger, int orderId, string customerId);

    [LoggerMessage(Level = LogLevel.Error, Message = "Failed to process order {OrderId}")]
    public static partial void OrderProcessingFailed(ILogger logger, int orderId, Exception ex);
}

// Использование
LogMessages.OrderCreated(_logger, order.Id, dto.CustomerId);
```

Преимущества source generators:
- Нет boxing для value types
- Нет аллокации `params object[]`
- Проверка `IsEnabled` генерируется автоматически
- Compile-time проверка количества и типов аргументов

В .NET 9 / C# 13+ ручные `ILogger`-вызовы получили перегрузки с `params ReadOnlySpan<T>` — они полностью устраняют аллокацию массива `params object[]` и boxing аргументов даже без `[LoggerMessage]`.

---

## Request/Response Logging

Middleware для логирования входящих запросов и исходящих ответов:

```csharp
// Встроенный (.NET 6+)
app.UseHttpLogging(); // Логирует метод, путь, статус, заголовки

builder.Services.AddHttpLogging(opts =>
{
    opts.LoggingFields = HttpLoggingFields.RequestMethod
        | HttpLoggingFields.RequestPath
        | HttpLoggingFields.ResponseStatusCode
        | HttpLoggingFields.Duration;
    opts.RequestHeaders.Add("X-Correlation-Id");
    // НЕ логировать body по умолчанию — performance и безопасность
});
```

### Кастомный middleware для логирования body

```csharp
public class RequestResponseLoggingMiddleware
{
    private readonly RequestDelegate _next;
    private readonly ILogger _logger;

    public async Task InvokeAsync(HttpContext context)
    {
        // Request body
        context.Request.EnableBuffering(); // Позволяет прочитать body несколько раз
        var requestBody = await new StreamReader(context.Request.Body).ReadToEndAsync();
        context.Request.Body.Position = 0; // Сброс позиции для следующего middleware

        // Response body — подмена потока
        var originalBody = context.Response.Body;
        using var memoryStream = new MemoryStream();
        context.Response.Body = memoryStream;

        await _next(context);

        memoryStream.Position = 0;
        var responseBody = await new StreamReader(memoryStream).ReadToEndAsync();
        memoryStream.Position = 0;
        await memoryStream.CopyToAsync(originalBody);
        context.Response.Body = originalBody;

        _logger.LogInformation("Request: {Method} {Path} Body: {RequestBody} → Response: {StatusCode} Body: {ResponseBody}",
            context.Request.Method, context.Request.Path, requestBody,
            context.Response.StatusCode, responseBody);
    }
}
```

### Тонкости Request/Response Logging

- `EnableBuffering()` добавляет overhead — используйте только когда нужно логировать body
- **Не логируйте body** в production без фильтрации — performance, PII данные
- Для больших body — ограничивайте размер логируемого фрагмента
- Correlation ID — пробрасывайте через все сервисы для трассировки запроса

---

## Sensitive Data и защита данных

### Маскирование чувствительных данных

**Никогда не логируйте**: пароли, токены, номера карт, персональные данные (PII).

```csharp
// Serilog — DestructuringPolicy для автоматического маскирования
public class SensitiveDataDestructuringPolicy : IDestructuringPolicy
{
    public bool TryDestructure(object value, ILogEventPropertyValueFactory factory,
        out LogEventPropertyValue? result)
    {
        if (value is UserDto user)
        {
            result = factory.CreatePropertyValue(new
            {
                user.Id,
                user.Name,
                Email = MaskEmail(user.Email),
                Phone = "***"
            }, destructureObjects: true);
            return true;
        }
        result = null;
        return false;
    }
}

// Microsoft.Extensions.Compliance.Redaction (.NET 8)
builder.Services.AddRedaction(opts =>
{
    opts.SetRedactor<ErasingRedactor>(new DataClassificationSet(DataClassifications.Pii));
});
```

### GDPR-соображения

- Шифрование логов, содержащих PII
- Ограничение доступа к логам (RBAC)
- Retention policy — автоматическое удаление старых логов
- Право на удаление — возможность найти и удалить логи конкретного пользователя
- Минимизация данных — логировать только то, что необходимо

---

## Глобальная обработка исключений — кратко

Canonical-разбор — в [[aspnet-error-handling|ASP.NET Error Handling]]: `UseExceptionHandler` + `AddProblemDetails`, типизированные `IExceptionHandler` (.NET 8+), Result pattern vs исключения, `Response.HasStarted`. Для логирования важно одно правило: **логируй исключение в exception handler, а не в каждом catch-блоке** — иначе одно исключение размножается по логам, а `traceId` из ProblemDetails (`ctx.HttpContext.TraceIdentifier`) должен совпадать с полем трейса в логах, чтобы по ответу клиента находить полный контекст.

---

## См. также

- [[observability|Observability]] — canonical: OpenTelemetry (traces/metrics/logs), Jaeger/Prometheus/Grafana, sampling, alerting
- [[aspnet-error-handling|ASP.NET Error Handling]] — canonical: глобальная обработка исключений, ProblemDetails, `IExceptionHandler`
