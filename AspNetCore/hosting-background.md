---
tags: [aspnet, hosting, background-service, hosted-service]
level: Senior
---

# Hosting и фоновые задачи

## Что это, зачем и когда

### Что такое Background Service?
Код, который работает **в фоне**, пока приложение обрабатывает HTTP-запросы. Не блокирует пользователя — работает параллельно.

**Аналогия — ресторан:** Официант (API) принимает заказы и отдаёт блюда. Повар (Background Service) **готовит** в фоне. Официант не ждёт пока блюдо приготовится — обслуживает другие столики.

### Зачем?
- **Отправка email/SMS** — не блокировать пользователя (ответ за 50мс вместо 2с)
- **Генерация отчётов** — долгая операция в фоне
- **Обработка файлов** — resize, конвертация загруженных изображений
- **Периодические задачи** — очистка кэша, синхронизация данных, проверка подписок

### Когда что?

| Инструмент | Когда |
|-----------|-------|
| `BackgroundService` + `Channel<T>` | Очередь задач внутри одного приложения |
| `BackgroundService` + `PeriodicTimer` | Задача по расписанию (каждые N минут) |
| **Hangfire / Quartz.NET** | Сложное расписание, retry, dashboard, персистентность |
| **RabbitMQ / MassTransit** | Задачи между РАЗНЫМИ сервисами |

---

## IHostApplicationLifetime

Интерфейс для отслеживания событий жизненного цикла приложения. Позволяет выполнять логику при запуске, остановке и после остановки.

### События

| Событие | Когда срабатывает | Типичное применение |
|---------|-------------------|---------------------|
| `ApplicationStarted` | Приложение полностью запущено, готово принимать запросы | Логирование, warm-up, уведомление |
| `ApplicationStopping` | Получен сигнал остановки (SIGTERM), начинается graceful shutdown | Завершение текущих запросов, flush буферов |
| `ApplicationStopped` | Приложение полностью остановлено | Очистка ресурсов, финальное логирование |

```csharp
public class LifetimeEventsService : IHostedService
{
    private readonly IHostApplicationLifetime _lifetime;
    private readonly ILogger<LifetimeEventsService> _logger;

    public LifetimeEventsService(IHostApplicationLifetime lifetime, ILogger<LifetimeEventsService> logger)
    {
        _lifetime = lifetime;
        _logger = logger;
    }

    public Task StartAsync(CancellationToken ct)
    {
        _lifetime.ApplicationStarted.Register(() =>
            _logger.LogInformation("Application started at {Time}", DateTime.UtcNow));

        _lifetime.ApplicationStopping.Register(() =>
            _logger.LogInformation("Application stopping..."));

        _lifetime.ApplicationStopped.Register(() =>
            _logger.LogInformation("Application stopped"));

        return Task.CompletedTask;
    }

    public Task StopAsync(CancellationToken ct) => Task.CompletedTask;
}
```

### Graceful Shutdown

При получении SIGTERM / Ctrl+C:
1. Срабатывает `ApplicationStopping`
2. Вызывается `StopAsync` на всех `IHostedService` (в обратном порядке регистрации)
3. Ожидание завершения текущих запросов (по умолчанию 30 секунд — `ShutdownTimeout`)
4. Срабатывает `ApplicationStopped`

```csharp
// Настройка таймаута graceful shutdown
builder.Host.ConfigureHostOptions(opts =>
{
    opts.ShutdownTimeout = TimeSpan.FromSeconds(60);
});
```

### Программная остановка

```csharp
// Остановить приложение из кода
app.MapPost("/admin/shutdown", (IHostApplicationLifetime lifetime) =>
{
    lifetime.StopApplication(); // Инициирует graceful shutdown
    return Results.Ok("Shutting down...");
});
```

---

## IHostedService и BackgroundService

### IHostedService

Базовый контракт для фоновых задач:

```csharp
public interface IHostedService
{
    Task StartAsync(CancellationToken cancellationToken);
    Task StopAsync(CancellationToken cancellationToken);
}
```

`StartAsync` вызывается **до** того, как приложение начнёт принимать запросы. Если `StartAsync` долгий — приложение не запустится вовремя. Для долгих задач — запускайте `Task` и не ждите его.

### BackgroundService

Базовый класс, упрощающий создание фоновых задач:

```csharp
public class OrderProcessingService : BackgroundService
{
    private readonly IServiceScopeFactory _scopeFactory;
    private readonly ILogger<OrderProcessingService> _logger;

    public OrderProcessingService(IServiceScopeFactory scopeFactory, ILogger<OrderProcessingService> logger)
    {
        _scopeFactory = scopeFactory;
        _logger = logger;
    }

    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        _logger.LogInformation("Order processing started");

        while (!stoppingToken.IsCancellationRequested)
        {
            try
            {
                using var scope = _scopeFactory.CreateScope();
                var repo = scope.ServiceProvider.GetRequiredService<IOrderRepository>();
                await ProcessPendingOrdersAsync(repo, stoppingToken);
            }
            catch (Exception ex) when (ex is not OperationCanceledException)
            {
                _logger.LogError(ex, "Error processing orders");
            }

            await Task.Delay(TimeSpan.FromSeconds(30), stoppingToken);
        }
    }
}

// Регистрация
builder.Services.AddHostedService<OrderProcessingService>();
```

### PeriodicTimer (.NET 6+)

Предпочтительнее `Task.Delay` для периодических задач — точнее контролирует интервал и корректно работает с cancellation:

```csharp
protected override async Task ExecuteAsync(CancellationToken stoppingToken)
{
    using var timer = new PeriodicTimer(TimeSpan.FromMinutes(5));

    while (await timer.WaitForNextTickAsync(stoppingToken))
    {
        try
        {
            await DoWorkAsync(stoppingToken);
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Periodic task failed");
            // Не пробрасываем — продолжаем работу
        }
    }
}
```

**Разница**: `Task.Delay` ждёт фиксированное время после завершения работы. `PeriodicTimer` тикает с фиксированным интервалом от начала — если работа заняла 2 секунды из 5-секундного интервала, следующий тик через 3 секунды. Если работа заняла больше интервала — следующий тик сразу.

### Типичные применения

| Задача | Реализация |
|--------|------------|
| Периодическая синхронизация данных | `BackgroundService` + `PeriodicTimer` |
| Обработка очередей (RabbitMQ, Kafka) | `BackgroundService` + consumer loop |
| Warm-up кэша при запуске | `IHostedService.StartAsync` |
| Очистка временных файлов | `BackgroundService` + `PeriodicTimer` |
| Health check для внешних сервисов | `BackgroundService` + periodic polling |

### Тонкости и нюансы

- **BackgroundService — Singleton**: нельзя инжектить Scoped-сервисы в конструктор. Используйте `IServiceScopeFactory`
- **Необработанное исключение в `ExecuteAsync`** (.NET 6) — приложение **завершается**. В .NET 8 по умолчанию тоже. Всегда оборачивайте в try-catch
- `StartAsync` **блокирует запуск** приложения — не делайте долгих операций. `ExecuteAsync` вызывается из `StartAsync`, но обычно возвращает `Task`, который работает в фоне
- **Порядок остановки** — обратный порядку регистрации. Если сервис B зависит от A, регистрируйте A первым
- `StopAsync` получает `CancellationToken` с таймаутом (`ShutdownTimeout`) — если не уложились, принудительное завершение
- Для сложных сценариев с очередями — используйте `Channel<T>` как in-memory очередь между HTTP-запросами и background-обработкой

```csharp
// Паттерн: Channel как in-memory очередь
builder.Services.AddSingleton(Channel.CreateBounded<WorkItem>(new BoundedChannelOptions(100)
{
    FullMode = BoundedChannelFullMode.Wait
}));

// В контроллере — пишем в канал
app.MapPost("/tasks", async (WorkItem item, Channel<WorkItem> channel) =>
{
    await channel.Writer.WriteAsync(item);
    return Results.Accepted();
});

// В BackgroundService — читаем из канала
protected override async Task ExecuteAsync(CancellationToken ct)
{
    await foreach (var item in _channel.Reader.ReadAllAsync(ct))
    {
        await ProcessAsync(item);
    }
}
```

---

## Health Checks

### Что это и зачем?
**Endpoint `/health`, который отвечает «приложение живо и работает».** Kubernetes, load balancer, мониторинг — все проверяют health check. Если unhealthy — трафик перенаправляется на другой инстанс.

**Аналогия:** Пульс пациента. Врач (Kubernetes) регулярно проверяет: пульс есть (healthy) → всё ок. Нет пульса (unhealthy) → реанимация (restart pod).

### Типы проверок

| Тип | Что проверяет | Endpoint | Когда |
|-----|--------------|----------|-------|
| **Liveness** | Приложение не зависло | `/health/live` | Если unhealthy → restart контейнера |
| **Readiness** | Приложение готово принимать трафик | `/health/ready` | Если не ready → убрать из load balancer |
| **Startup** | Приложение запустилось | `/health/startup` | Не проверять liveness пока не стартовал |

### Базовая настройка

```csharp
// Регистрация health checks
builder.Services.AddHealthChecks()
    // БД — NuGet: AspNetCore.HealthChecks.NpgSql
    .AddNpgSql(
        connectionString,
        name: "postgresql",
        tags: ["ready"])
    // Redis — NuGet: AspNetCore.HealthChecks.Redis
    .AddRedis(
        redisConnectionString,
        name: "redis",
        tags: ["ready"])
    // Кастомная проверка
    .AddCheck<S3HealthCheck>("s3", tags: ["ready"])
    // Простая проверка (всегда healthy)
    .AddCheck("self", () => HealthCheckResult.Healthy(), tags: ["live"]);

// Маппинг endpoints
app.MapHealthChecks("/health/live", new HealthCheckOptions
{
    Predicate = check => check.Tags.Contains("live")
});

app.MapHealthChecks("/health/ready", new HealthCheckOptions
{
    Predicate = check => check.Tags.Contains("ready"),
    ResponseWriter = WriteDetailedResponse
});

// Детальный ответ в JSON
static async Task WriteDetailedResponse(HttpContext context, HealthReport report)
{
    context.Response.ContentType = "application/json";
    var result = new
    {
        status = report.Status.ToString(),
        duration = report.TotalDuration.TotalMilliseconds,
        checks = report.Entries.Select(e => new
        {
            name = e.Key,
            status = e.Value.Status.ToString(),
            duration = e.Value.Duration.TotalMilliseconds,
            description = e.Value.Description,
            error = e.Value.Exception?.Message
        })
    };
    await context.Response.WriteAsJsonAsync(result);
}
```

### Кастомный Health Check

```csharp
public sealed class S3HealthCheck(IAmazonS3 s3Client) : IHealthCheck
{
    public async Task<HealthCheckResult> CheckHealthAsync(
        HealthCheckContext context, CancellationToken ct)
    {
        try
        {
            await s3Client.ListBucketsAsync(ct);
            return HealthCheckResult.Healthy("S3 is accessible");
        }
        catch (Exception ex)
        {
            return HealthCheckResult.Unhealthy("S3 is not accessible", ex);
        }
    }
}

// Регистрация
builder.Services.AddHealthChecks()
    .AddCheck<S3HealthCheck>("s3", tags: ["ready"]);
```

### Kubernetes probes

```yaml
# deployment.yaml
livenessProbe:
  httpGet:
    path: /health/live
    port: 8080
  initialDelaySeconds: 5
  periodSeconds: 10

readinessProbe:
  httpGet:
    path: /health/ready
    port: 8080
  initialDelaySeconds: 10
  periodSeconds: 5
  failureThreshold: 3
```

**Нюанс:** Health check endpoint **не должен** требовать аутентификации. Добавьте `.AllowAnonymous()` или исключите из auth middleware.
