---
tags: [aspnet, hosting, background-service, hosted-service]
level: Senior
date: 2026-06-28
---

# Hosting и фоновые задачи

> Фоновая обработка через `BackgroundService` с `Channel<T>` и `PeriodicTimer`, управление жизненным циклом приложения (`IHostApplicationLifetime`) и graceful shutdown.

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


---

## Hangfire — production-grade job scheduling

### Зачем Hangfire

Built-in `BackgroundService` хорош для simple scenarios. Production апликейшны часто требуют:
- **Persistent jobs** — пережили restart
- **Retries** автоматические с exponential backoff
- **Scheduled jobs** в будущем (через 30 минут, завтра в 9:00)
- **Recurring jobs** (cron syntax)
- **Dashboard** для мониторинга
- **Distributed** — несколько workers обрабатывают очередь

**Hangfire** — самая популярная library в .NET ecosystem.

### Setup

```bash
dotnet add package Hangfire
dotnet add package Hangfire.SqlServer    # или Hangfire.PostgreSql
dotnet add package Hangfire.AspNetCore
```

```csharp
// Program.cs
builder.Services.AddHangfire(config => config
    .SetDataCompatibilityLevel(CompatibilityLevel.Version_180)
    .UseSimpleAssemblyNameTypeSerializer()
    .UseRecommendedSerializerSettings()
    .UsePostgreSqlStorage(connStr));

builder.Services.AddHangfireServer();   // worker process

var app = builder.Build();

app.UseHangfireDashboard("/hangfire", new DashboardOptions
{
    Authorization = new[] { new MyAuthFilter() }
});

app.Run();
```

### Job types

```csharp
public class EmailService(IEmailClient client)
{
    // 1. Fire-and-forget — execute once, ASAP
    BackgroundJob.Enqueue<EmailService>(s => s.SendEmail("user@x.com", "Hello"));
    
    // 2. Delayed — execute через N времени
    BackgroundJob.Schedule<EmailService>(
        s => s.SendReminder(orderId),
        TimeSpan.FromHours(24));
    
    // 3. Recurring — по cron
    RecurringJob.AddOrUpdate<EmailService>(
        "daily-digest",
        s => s.SendDailyDigest(),
        Cron.Daily(8));   // каждый день в 8:00 UTC
    
    // 4. Continuations — после parent job
    var parentId = BackgroundJob.Enqueue<OrderService>(s => s.Process(orderId));
    BackgroundJob.ContinueJobWith<EmailService>(
        parentId,
        s => s.SendConfirmation(orderId));
    
    // 5. Batch jobs (Hangfire Pro — paid)
    var batchId = BatchJob.StartNew(b =>
    {
        for (int i = 0; i < 100; i++)
            b.Enqueue<MyService>(s => s.Process(i));
    });
}
```

### Retry behavior

```csharp
public class EmailService
{
    [AutomaticRetry(Attempts = 5, DelaysInSeconds = new[] { 60, 300, 900, 3600, 14400 })]
    public async Task SendEmail(string to, string subject)
    {
        // Если throws — retry per attempts, exponential backoff
        await _client.SendAsync(to, subject);
    }
    
    // Перманентный fail после исчерпания retries
    [AutomaticRetry(Attempts = 0)]   // не retry
    public void DeleteOldLogs() { }
}
```

### Cron syntax примеры

```csharp
RecurringJob.AddOrUpdate("daily", () => Job(), Cron.Daily(8));     // 8:00 каждый день
RecurringJob.AddOrUpdate("hourly", () => Job(), Cron.Hourly());     // каждый час
RecurringJob.AddOrUpdate("weekly", () => Job(), Cron.Weekly(DayOfWeek.Monday, 9));  // понедельник 9:00
RecurringJob.AddOrUpdate("custom", () => Job(), "*/15 * * * *");    // каждые 15 минут
RecurringJob.AddOrUpdate("monthly", () => Job(), "0 0 1 * *");      // 1-го числа в полночь
```

### Trade-offs

```
✅ Hangfire:
- Persistent storage — survive crashes
- Built-in retry с exponential backoff
- Dashboard для monitoring
- Production-tested
- Free для core features

❌ Cons:
- Lock-in на Hangfire scheduler
- Storage overhead (DB tables)
- Lock contention под high load
- Hangfire.Pro features paid
```

> [!question]- **Интервью: BackgroundService vs Hangfire?**
> **BackgroundService** — built-in, simple, in-process. **Хорошо для**: continuous loops (queue consumers, periodic checks). **Плохо для**: scheduled jobs, retries with backoff, persistent across restarts. **Hangfire** — full job scheduler с persistence (DB), automatic retries, cron scheduling, dashboard. **Production**: combine — BackgroundService для real-time consumers, Hangfire для scheduled/retryable work. **Alternatives**: Quartz.NET (similar, более complex API), Coravel (lightweight), MassTransit Scheduling.

---

## Quartz.NET — alternative с rich cron

### Setup

```bash
dotnet add package Quartz
dotnet add package Quartz.Extensions.Hosting
```

```csharp
builder.Services.AddQuartz(q =>
{
    q.UseMicrosoftDependencyInjectionJobFactory();
    
    var jobKey = new JobKey("DataSyncJob");
    q.AddJob<DataSyncJob>(opts => opts.WithIdentity(jobKey));
    
    q.AddTrigger(opts => opts
        .ForJob(jobKey)
        .WithIdentity("DataSyncJob-trigger")
        .WithCronSchedule("0 */15 * * * ?"));   // каждые 15 минут
});

builder.Services.AddQuartzHostedService(q => q.WaitForJobsToComplete = true);
```

```csharp
public class DataSyncJob : IJob
{
    private readonly IServiceScopeFactory _scopeFactory;
    
    public DataSyncJob(IServiceScopeFactory scopeFactory) => _scopeFactory = scopeFactory;
    
    public async Task Execute(IJobExecutionContext context)
    {
        using var scope = _scopeFactory.CreateScope();
        var db = scope.ServiceProvider.GetRequiredService<AppDbContext>();
        
        await SyncDataAsync(db, context.CancellationToken);
    }
}
```

### Quartz vs Hangfire

| | Hangfire | Quartz.NET |
|--|----------|------------|
| Storage required | Yes (DB) | No (in-memory) или DB optional |
| Dashboard | Yes (built-in) | No (third-party) |
| Cron syntax | Standard 5-field | Quartz cron 7-field (more powerful) |
| API simplicity | Easy | Complex |
| Use case | Most apps | Complex scheduling needs |

---

## Distributed Locks — single-instance work в multi-pod

### Проблема

```csharp
public class CleanupJob : BackgroundService
{
    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        while (!stoppingToken.IsCancellationRequested)
        {
            await CleanupOldRecordsAsync();   // ⚠️ Каждый pod вызовет!
            await Task.Delay(TimeSpan.FromHours(1), stoppingToken);
        }
    }
}
```

В 4-pod deployment job выполнится **4 раза** одновременно. Может:
- Дублировать work
- Race conditions при concurrent updates
- Wasted resources

### Решение 1 — Distributed lock через Redis (RedLock)

```bash
dotnet add package RedLock.net
```

```csharp
public class CleanupJob : BackgroundService
{
    private readonly RedLockFactory _redLockFactory;
    private readonly IServiceScopeFactory _scopeFactory;
    
    public CleanupJob(IConnectionMultiplexer redis, IServiceScopeFactory scopeFactory)
    {
        _redLockFactory = RedLockFactory.Create(new[]
        {
            new RedLockMultiplexer(redis)
        });
        _scopeFactory = scopeFactory;
    }
    
    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        while (!stoppingToken.IsCancellationRequested)
        {
            await using var redLock = await _redLockFactory.CreateLockAsync(
                resource: "cleanup-job",
                expiryTime: TimeSpan.FromMinutes(10),
                waitTime: TimeSpan.Zero,           // не ждём — другой pod уже работает
                retryTime: TimeSpan.FromSeconds(1),
                cancellationToken: stoppingToken);
            
            if (redLock.IsAcquired)
            {
                using var scope = _scopeFactory.CreateScope();
                await CleanupOldRecordsAsync(scope.ServiceProvider, stoppingToken);
            }
            // else: other pod has lock — skip
            
            await Task.Delay(TimeSpan.FromHours(1), stoppingToken);
        }
    }
}
```

### Решение 2 — IDistributedLockProvider

`Microsoft.Extensions.DistributedLock` или DistributedLock library:

```bash
dotnet add package DistributedLock.SqlServer    # или Redis, Azure
```

```csharp
public class CleanupJob : BackgroundService
{
    private readonly IDistributedLockProvider _lockProvider;
    
    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        while (!stoppingToken.IsCancellationRequested)
        {
            var @lock = _lockProvider.CreateLock("cleanup-job");
            await using var handle = await @lock.TryAcquireAsync(
                timeout: TimeSpan.Zero,
                cancellationToken: stoppingToken);
            
            if (handle != null)   // мы acquired
            {
                await CleanupOldRecordsAsync(stoppingToken);
            }
            
            await Task.Delay(TimeSpan.FromHours(1), stoppingToken);
        }
    }
}
```

### Решение 3 — Database advisory lock (PostgreSQL)

```csharp
public async Task RunWithLockAsync(string lockName, Func<Task> work)
{
    using var connection = new NpgsqlConnection(_connStr);
    await connection.OpenAsync();
    
    var lockId = HashLockName(lockName);   // bigint hash
    
    // Acquire advisory lock
    var acquired = await connection.QuerySingleAsync<bool>(
        "SELECT pg_try_advisory_lock(@id)",
        new { id = lockId });
    
    if (!acquired) return;   // другой process держит
    
    try
    {
        await work();
    }
    finally
    {
        await connection.ExecuteAsync("SELECT pg_advisory_unlock(@id)", new { id = lockId });
    }
}
```

### Trade-offs locks

```
RedLock (Redis):
✅ Fast (~ms)
✅ Battle-tested algorithm
❌ Redis dependency
❌ Сложности с partial failures (multi-Redis)

DistributedLock library:
✅ Multiple backends (Redis, SQL Server, Azure, ZooKeeper)
✅ Simple API
❌ Less control fine-grained

PostgreSQL advisory locks:
✅ No extra infrastructure
✅ ACID guarantees
❌ Bound к existing DB
❌ Connection-bound (tricky if connection drops)
```

> [!question]- **Интервью: distributed lock в multi-pod?**
> **Проблема**: один pod = один BackgroundService instance. 4 pods = 4 instances of same job → wasted work, conflicts. **Решение**: distributed lock across pods. **Options**: 1) **RedLock** (Redis) — fast, battle-tested. 2) **IDistributedLockProvider** (DistributedLock library) — multiple backends. 3) **PostgreSQL advisory locks** — no extra infrastructure. **Pattern**: `TryAcquire(timeout: zero)` → если acquired = run job, else skip (другой pod работает). **Critical**: lock duration > work duration; lock auto-expires если pod crashes (avoid permanent lockout).

---

## Leader Election — explicit single-instance

Альтернатива distributed lock — explicit leader election. Один pod становится leader, остальные — followers.

### Microsoft.Kubernetes.Controller (для Kubernetes)

```csharp
builder.Services.AddSingleton<ILeaseClient>(sp =>
{
    var k8sClient = sp.GetRequiredService<IKubernetes>();
    return new LeaseClient(k8sClient, namespaceName: "default");
});

public class LeaderJob : BackgroundService
{
    private readonly ILeaseClient _leaseClient;
    private readonly string _podName;
    
    public LeaderJob(ILeaseClient leaseClient)
    {
        _leaseClient = leaseClient;
        _podName = Environment.GetEnvironmentVariable("POD_NAME")!;
    }
    
    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        while (!stoppingToken.IsCancellationRequested)
        {
            var isLeader = await _leaseClient.TryAcquireLeaseAsync(
                "my-job-leader",
                _podName,
                duration: TimeSpan.FromSeconds(15),
                stoppingToken);
            
            if (isLeader)
            {
                await DoLeaderWorkAsync(stoppingToken);
                await _leaseClient.RenewLeaseAsync("my-job-leader", _podName, stoppingToken);
            }
            
            await Task.Delay(TimeSpan.FromSeconds(10), stoppingToken);
        }
    }
}
```

### Kubernetes Lease object

Kubernetes имеет встроенный `Lease` resource в `coordination.k8s.io/v1`. Используется для leader election в core controllers (kube-controller-manager).

```yaml
apiVersion: coordination.k8s.io/v1
kind: Lease
metadata:
  name: my-app-leader
  namespace: default
spec:
  holderIdentity: "pod-abc-123"
  leaseDurationSeconds: 15
  renewTime: "2026-05-10T12:00:00Z"
```

### Lock vs Leader Election

| | Distributed Lock | Leader Election |
|--|------------------|-----------------|
| Granularity | Per-task | Per-instance |
| Use case | One task at a time | Long-running leader role |
| Failover | Lock expires → next acquires | Leader fails → re-election |
| Complexity | Simpler | More complex |
| Ideal for | Periodic jobs (every hour) | Continuous coordinator |

---

## Worker Service Templates

`dotnet new worker` — отдельное приложение без HTTP, только background work.

```bash
dotnet new worker -n MyWorker
```

```csharp
// Program.cs (Worker)
var builder = Host.CreateApplicationBuilder(args);

builder.Services.AddDbContext<AppDbContext>(opt =>
    opt.UseSqlServer(builder.Configuration.GetConnectionString("Default")));

builder.Services.AddHostedService<DataProcessingWorker>();

var host = builder.Build();
await host.RunAsync();
```

### Use cases for separate Worker app

- **Heavy batch processing** — изолировать от web app
- **Independent scaling** — workers отдельно от API
- **Different resources** — workers нужны больше memory / CPU
- **Long-running jobs** — не блокируют web requests

### Docker pattern

```yaml
# docker-compose.yml
services:
  api:
    image: myapp/api
    replicas: 4
    
  worker:
    image: myapp/worker
    replicas: 2   # меньше workers
    environment:
      - WORKER_TYPE=email
  
  scheduled-worker:
    image: myapp/worker
    replicas: 1   # один pod для cron jobs
    environment:
      - WORKER_TYPE=scheduler
```

---

## Testing Background Services

### Unit test

```csharp
public class OrderProcessorTests
{
    [Fact]
    public async Task ProcessOrders_HandlesCancellation()
    {
        // Arrange
        var scopeFactory = Substitute.For<IServiceScopeFactory>();
        var logger = Substitute.For<ILogger<OrderProcessor>>();
        var processor = new OrderProcessor(scopeFactory, logger);
        
        var cts = new CancellationTokenSource();
        cts.CancelAfter(TimeSpan.FromMilliseconds(100));
        
        // Act + Assert
        await processor.StartAsync(cts.Token);
        await Task.Delay(200);
        await processor.StopAsync(CancellationToken.None);
        
        // Verify
        Assert.True(cts.IsCancellationRequested);
    }
}
```

### Integration test

```csharp
public class BackgroundServiceIntegrationTests : IClassFixture<WebApplicationFactory<Program>>
{
    [Fact]
    public async Task BackgroundService_RunsOnStartup()
    {
        await using var factory = new WebApplicationFactory<Program>();
        await using var services = factory.Services.CreateAsyncScope();
        
        // Trigger через test server
        using var client = factory.CreateClient();
        var response = await client.PostAsync("/api/orders", JsonContent.Create(orderDto));
        
        // Wait for background service
        await Task.Delay(TimeSpan.FromSeconds(2));
        
        // Verify side effects
        var db = services.ServiceProvider.GetRequiredService<AppDbContext>();
        Assert.True(await db.Orders.AnyAsync(o => o.Status == OrderStatus.Processed));
    }
}
```

### TimeProvider для тестируемого timing

```csharp
// Use TimeProvider вместо Task.Delay для testability
public class PeriodicJob(TimeProvider timeProvider, ILogger<PeriodicJob> logger) : BackgroundService
{
    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        using var timer = timeProvider.CreateTimer(
            callback: _ => DoWork(),
            state: null,
            dueTime: TimeSpan.FromMinutes(1),
            period: TimeSpan.FromMinutes(5));
        
        await Task.Delay(Timeout.Infinite, stoppingToken);
    }
}

// Test:
var fakeTime = new FakeTimeProvider();
var job = new PeriodicJob(fakeTime, logger);

await job.StartAsync(default);
fakeTime.Advance(TimeSpan.FromMinutes(6));   // simulate 6 минут
// Job выполнен 1 раз (после первого периода)
```

---

## Common pitfalls

### Captive dependency в BackgroundService

См. `aspnet-dependency-injection-deep.md` раздел 2. Singleton (BackgroundService) → Scoped (DbContext) → ObjectDisposedException.

**Fix**: `IServiceScopeFactory.CreateScope()` каждую итерацию.

### Unhandled exceptions завершают app

```csharp
protected override async Task ExecuteAsync(CancellationToken stoppingToken)
{
    while (!stoppingToken.IsCancellationRequested)
    {
        await DoWorkAsync();   // ❌ если throws → app завершится в .NET 6+!
    }
}
```

С .NET 6+ unhandled exception в `ExecuteAsync` завершает host. Wrap в try-catch:

```csharp
while (!stoppingToken.IsCancellationRequested)
{
    try
    {
        await DoWorkAsync();
    }
    catch (Exception ex) when (ex is not OperationCanceledException)
    {
        _logger.LogError(ex, "Background service iteration failed");
    }
    
    await Task.Delay(TimeSpan.FromMinutes(5), stoppingToken);
}
```

### Long StartAsync блокирует приложение

```csharp
public async Task StartAsync(CancellationToken ct)
{
    await PreloadCacheAsync();   // ❌ занимает 30 секунд
    // App не accept HTTP requests пока не закончится
}
```

`StartAsync` должен возвращаться quickly. Heavy work запускай в `ExecuteAsync` в fire-and-forget:

```csharp
public Task StartAsync(CancellationToken ct)
{
    _ = Task.Run(async () => await PreloadCacheAsync(), ct);
    return Task.CompletedTask;
}
```

### Scheduled job в multi-pod без lock

См. раздел Distributed Locks выше. Без lock — job выполняется в каждом pod.

### Task.Delay без CancellationToken

```csharp
await Task.Delay(TimeSpan.FromMinutes(5));   // ❌ не cancellable
await Task.Delay(TimeSpan.FromMinutes(5), stoppingToken);   // ✅
```

При SIGTERM Task.Delay не cancellable → graceful shutdown зависает.

### PeriodicTimer vs Task.Delay

PeriodicTimer (.NET 6+) лучше для periodic jobs:

```csharp
using var timer = new PeriodicTimer(TimeSpan.FromMinutes(5));
while (await timer.WaitForNextTickAsync(stoppingToken))
{
    await DoWorkAsync();
}
```

vs Task.Delay — drift compounds:

```csharp
while (!stoppingToken.IsCancellationRequested)
{
    var sw = Stopwatch.StartNew();
    await DoWorkAsync();   // 30 sec
    await Task.Delay(TimeSpan.FromMinutes(5), stoppingToken);   // ещё 5 мин
    // Real period: 5:30 — drift!
}
```

### Hangfire jobs не idempotent

```csharp
public async Task ChargeCustomer(int orderId, decimal amount)
{
    await _payment.ChargeAsync(orderId, amount);
    // Если throws ПОСЛЕ payment, Hangfire retry → повторное charge!
}

// ✅ Idempotency через unique key
public async Task ChargeCustomer(int orderId, decimal amount, string idempotencyKey)
{
    await _payment.ChargeAsync(orderId, amount, idempotencyKey);
    // Payment provider rejects duplicates by key
}
```

### Graceful shutdown не respect

```csharp
public Task StopAsync(CancellationToken ct)
{
    // ❌ stops без waiting for in-flight work
    return Task.CompletedTask;
}

// ✅
public async Task StopAsync(CancellationToken ct)
{
    _stopRequested = true;
    
    // Wait for current iteration up to ct deadline
    await _currentTask.WaitAsync(ct);
}
```

> [!question]- **Интервью: топ-3 ошибки BackgroundService?**
> 1) **Captive dependency** — DbContext (scoped) injected в BackgroundService (singleton) → ObjectDisposedException. Fix: `IServiceScopeFactory.CreateScope()` per iteration. 2) **Unhandled exception завершает app** (.NET 6+ default). Fix: try-catch в loop. 3) **Multi-pod duplicate work** — каждый pod runs job. Fix: distributed lock (RedLock/PostgreSQL advisory) или leader election. **Bonus**: PeriodicTimer вместо Task.Delay (no drift).

---

## Cheat sheet

```csharp
// === BackgroundService с scoped deps ===
public class MyWorker(IServiceScopeFactory scopeFactory, ILogger<MyWorker> logger) : BackgroundService
{
    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        while (!stoppingToken.IsCancellationRequested)
        {
            try
            {
                using var scope = scopeFactory.CreateScope();
                var db = scope.ServiceProvider.GetRequiredService<AppDbContext>();
                await DoWorkAsync(db, stoppingToken);
            }
            catch (Exception ex) when (ex is not OperationCanceledException)
            {
                logger.LogError(ex, "Iteration failed");
            }
            
            await Task.Delay(TimeSpan.FromMinutes(5), stoppingToken);
        }
    }
}

// === PeriodicTimer (no drift) ===
using var timer = new PeriodicTimer(TimeSpan.FromMinutes(5));
while (await timer.WaitForNextTickAsync(stoppingToken))
{
    await DoWorkAsync();
}

// === Hangfire ===
builder.Services.AddHangfire(c => c.UsePostgreSqlStorage(connStr));
builder.Services.AddHangfireServer();

BackgroundJob.Enqueue<EmailService>(s => s.Send(orderId));               // fire-and-forget
BackgroundJob.Schedule<EmailService>(s => s.Remind(orderId), TimeSpan.FromHours(24));
RecurringJob.AddOrUpdate<EmailService>("digest", s => s.SendDigest(), Cron.Daily(8));

[AutomaticRetry(Attempts = 5)]
public async Task SendEmail() { }

// === Quartz.NET ===
builder.Services.AddQuartz(q =>
{
    var jobKey = new JobKey("DataSync");
    q.AddJob<DataSyncJob>(opts => opts.WithIdentity(jobKey));
    q.AddTrigger(opts => opts
        .ForJob(jobKey)
        .WithCronSchedule("0 */15 * * * ?"));
});
builder.Services.AddQuartzHostedService();

// === Distributed lock (RedLock) ===
await using var redLock = await _redLockFactory.CreateLockAsync(
    "cleanup-job",
    expiryTime: TimeSpan.FromMinutes(10),
    waitTime: TimeSpan.Zero,
    retryTime: TimeSpan.FromSeconds(1));

if (redLock.IsAcquired)
{
    await DoWorkAsync();
}

// === PostgreSQL advisory lock ===
var acquired = await connection.QuerySingleAsync<bool>(
    "SELECT pg_try_advisory_lock(@id)", new { id = HashLockName(lockName) });

if (acquired)
{
    try { await work(); }
    finally { await connection.ExecuteAsync("SELECT pg_advisory_unlock(@id)", new { id }); }
}

// === Channel as in-memory queue ===
builder.Services.AddSingleton(Channel.CreateBounded<WorkItem>(new BoundedChannelOptions(100)
{
    FullMode = BoundedChannelFullMode.Wait
}));

// Producer
await channel.Writer.WriteAsync(workItem);

// Consumer
await foreach (var item in channel.Reader.ReadAllAsync(stoppingToken))
    await ProcessAsync(item);
```

---

## Practice exercises

### 1. Multi-pod scheduled job

Реализуй background job:
- Runs каждый час
- Запускается ровно в одном pod (из 4)
- Уперший pod fails → другой берёт работу
- Logs кто currently leader

Используй RedLock или PostgreSQL advisory lock.

### 2. Hangfire с retry

Email service с automatic retries:
- 3 attempts с exponential backoff (1min, 5min, 30min)
- После всех retries — notify admin
- Idempotency key чтобы duplicate retries не отправили email дважды

### 3. Channel-based queue

API endpoint: POST /api/process — queues work item.
BackgroundService consumer: processes items, max 4 concurrent.

При shutdown — gracefully drain queue (process all queued items).

---

## Reading list

- **Hangfire docs** — docs.hangfire.io
- **Quartz.NET docs** — quartznet.sourceforge.io
- **Microsoft Docs — BackgroundService** — learn.microsoft.com/aspnet/core/fundamentals/host/hosted-services
- **DistributedLock library** — github.com/madelson/DistributedLock
- **RedLock.net** — github.com/samcook/RedLock.net
- **Andrew Lock — BackgroundService articles** — andrewlock.net
