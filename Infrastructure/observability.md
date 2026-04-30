---
tags: [observability, opentelemetry, prometheus, grafana, loki, tempo, sli, slo, alerting]
level: Senior
---

# Observability — OpenTelemetry, метрики, traces, alerting

## Что это, зачем и когда

### Что такое observability?
Способность **по внешним сигналам понимать что происходит внутри системы**: где тормозит, что падает, какая очередь забита, какой эндпоинт хитят. Не путай с monitoring (это только часть).

**Аналогия:** Машина едет странно. Без observability ты только видишь что она тормозит. С observability у тебя bordcomputer — можешь посмотреть температуру каждого датчика, обороты, давление масла, какой цилиндр пропускает зажигание.

### Три pillars

| | Logs | Metrics | Traces |
|--|------|---------|--------|
| Что | Дискретные события с контекстом | Числа во времени | Цепочка вызовов через системы |
| Размер | Большой (текст) | Маленький (числа) | Средний |
| Хранение | Долго (compliance) | Долго (retention 30-90 дней) | Кратко (sampling) |
| Когда | Отладка конкретного случая | Мониторинг trends, alerting | "Где медленно? Где упало?" |
| Пример | "User 123 logged in" | `requests_total{endpoint="/login"} 5234` | `POST /order → ValidateAsync → SaveAsync → publish` |

**Все три нужны.** Logs без traces — не понять flow между сервисами. Metrics без logs — не понять причину проблемы. Traces без metrics — не понять "это нормально или нет".

### Зачем

| Без observability | С observability |
|-------------------|-----------------|
| "Что-то медленно" — сидишь у logs, ищешь | Видишь dashboard: p99 latency на /order вырос с 200ms до 800ms 10 минут назад |
| Падает по ночам — никто не видит до утра | Alerting — Telegram/PagerDuty будит on-call инженера |
| Производительность деградирует — открываешь когда уже поздно | SLO отслеживается, error budget burn rate alerts |
| Не понимаешь bottleneck в распределённой системе | Distributed trace показывает: 90% времени в одном downstream-вызове |

---

## OpenTelemetry — стандарт

CNCF-стандарт, объединяет logs/metrics/traces под единым API. Реализация для .NET через **OpenTelemetry .NET**.

```bash
dotnet add package OpenTelemetry.Extensions.Hosting
dotnet add package OpenTelemetry.Instrumentation.AspNetCore
dotnet add package OpenTelemetry.Instrumentation.Http
dotnet add package OpenTelemetry.Instrumentation.Runtime
dotnet add package OpenTelemetry.Exporter.OpenTelemetryProtocol
```

### Полная конфигурация (рекомендуемая)

```csharp
const string ServiceName = "MyApp";
const string ServiceVersion = "1.0.0";

var resource = ResourceBuilder.CreateDefault()
    .AddService(ServiceName, serviceVersion: ServiceVersion)
    .AddAttributes(new Dictionary<string, object>
    {
        ["deployment.environment"] = builder.Environment.EnvironmentName,
        ["host.name"] = Environment.MachineName,
    });

builder.Services.AddOpenTelemetry()
    .ConfigureResource(r => r.AddService(ServiceName, serviceVersion: ServiceVersion))
    .WithTracing(tracing =>
    {
        tracing
            .AddAspNetCoreInstrumentation(opts =>
            {
                opts.Filter = ctx => !ctx.Request.Path.StartsWithSegments("/health");
                opts.RecordException = true;
                opts.EnrichWithHttpRequest = (activity, request) =>
                {
                    activity.SetTag("http.client_ip", request.HttpContext.Connection.RemoteIpAddress?.ToString());
                };
            })
            .AddHttpClientInstrumentation()
            .AddSource("MassTransit")
            .AddSource(ServiceName)  // Custom ActivitySource
            .AddNpgsql()              // EF Core / Npgsql
            .SetSampler(new ParentBasedSampler(new TraceIdRatioBasedSampler(0.1)))  // 10% sampling
            .AddOtlpExporter(opts =>
            {
                opts.Endpoint = new Uri(builder.Configuration["Otel:Endpoint"]!);
                opts.Protocol = OtlpExportProtocol.Grpc;
            });
    })
    .WithMetrics(metrics =>
    {
        metrics
            .AddAspNetCoreInstrumentation()
            .AddHttpClientInstrumentation()
            .AddRuntimeInstrumentation()  // GC, ThreadPool, Memory
            .AddProcessInstrumentation()   // CPU, Memory из ОС
            .AddMeter("MassTransit")
            .AddMeter(ServiceName)
            .AddPrometheusExporter();       // Prometheus scrape endpoint
    });

// Logs через ILogger автоматически экспортируются
builder.Logging.AddOpenTelemetry(opts =>
{
    opts.SetResourceBuilder(resource);
    opts.IncludeScopes = true;
    opts.IncludeFormattedMessage = true;
    opts.AddOtlpExporter();
});

var app = builder.Build();
app.MapPrometheusScrapingEndpoint();  // /metrics для Prometheus
app.Run();
```

---

## Tracing — distributed traces

### Custom ActivitySource

```csharp
public sealed class OrderService
{
    private static readonly ActivitySource ActivitySource = new("MyApp.Orders", "1.0.0");

    public async Task PlaceAsync(PlaceOrderCommand cmd, CancellationToken ct)
    {
        using var activity = ActivitySource.StartActivity("PlaceOrder");
        activity?.SetTag("order.user_id", cmd.UserId);
        activity?.SetTag("order.total", cmd.Total);

        try
        {
            await ValidateAsync(cmd, ct);
            await SaveAsync(cmd, ct);
            activity?.SetStatus(ActivityStatusCode.Ok);
        }
        catch (ValidationException ex)
        {
            activity?.SetStatus(ActivityStatusCode.Error, ex.Message);
            activity?.AddEvent(new ActivityEvent("validation.failed", tags: new ActivityTagsCollection
            {
                ["validation.errors"] = string.Join(",", ex.Errors),
            }));
            throw;
        }
    }
}
```

В Tempo/Jaeger увидишь trace, который пересекает: `POST /api/orders` → `OrderController.Place` → `OrderService.PlaceAsync` → `ValidateAsync` → `SaveAsync` → `INSERT INTO orders`.

### Sampling strategies

Trace всё подряд = тратишь storage и сеть на 99% мусора.

| Strategy | Когда |
|----------|-------|
| **Always-on** | Dev, low-traffic prod |
| **TraceIdRatioBased** (e.g. 10%) | Production high-traffic — простой sampling |
| **ParentBased** | Если parent span sampled → child sampled (consistency) |
| **AlwaysOnFor errors** | Ошибки трейсим всегда, остальное по 10% |
| **Tail-based sampling** | OTel Collector решает после получения всего trace (нужен collector) |

```csharp
// Production баланс
.SetSampler(new ParentBasedSampler(new TraceIdRatioBasedSampler(0.1)))

// Errors — всегда
.SetSampler(new AlwaysOnSampler())  // в OTel Collector — фильтр на errors
```

**Tail-based** — single most powerful approach: OTel Collector видит весь trace целиком, потом решает trace всё или нет. Сэмплит: все ошибки + все медленные (> 1s) + 1% нормальных. Но требует [OTel Collector](https://opentelemetry.io/docs/collector/) в инфре.

### Context propagation

OpenTelemetry автоматически пробрасывает trace context через:
- HTTP headers (`traceparent`, `tracestate` — W3C Trace Context)
- gRPC metadata
- MassTransit message headers (если `AddSource("MassTransit")`)

В Jaeger/Tempo увидишь trace, который начинается с `POST /api/orders` (Service A), переходит в `OrderConsumer.Consume` (Service B через RabbitMQ), затем в `INSERT INTO orders` (Service B → Postgres).

### Baggage — propagated context

Trace ID — обязателен. Но можешь добавить **baggage** — пользовательские key-value пары, пробрасываемые через всю цепочку:

```csharp
// На входе (e.g. middleware)
Baggage.Current = Baggage.Current.SetBaggage("user.id", userId.ToString());
Baggage.Current = Baggage.Current.SetBaggage("tenant.id", tenantId.ToString());

// В downstream service эти значения автоматически приходят
var userId = Baggage.GetBaggage("user.id");
```

Полезно для cross-cutting context — tenant ID, user ID, feature flags, A/B test variant.

> [!question]- **Интервью: какая разница между span attributes и baggage?**
> **Span attributes** — характеризуют текущий span. Виден только в этом span'е, в отчётах и в child spans если ты их явно копируешь.
> **Baggage** — propagated через весь trace, на любом уровне можно достать. Кладётся в HTTP headers, есть **overhead** (передаётся с каждым запросом).
> Используй attributes для span-specific фактов (latency, status). Baggage — для cross-cutting context (user ID, tenant), который нужен всему trace.

---

## Metrics

### Custom Meter

```csharp
public sealed class OrderMetrics
{
    private static readonly Meter Meter = new("MyApp.Orders", "1.0.0");

    public static readonly Counter<long> OrdersPlaced =
        Meter.CreateCounter<long>("orders.placed", "{order}", "Total orders placed");

    public static readonly Histogram<double> OrderProcessingDuration =
        Meter.CreateHistogram<double>("orders.processing_duration", "ms", "Order processing time");

    public static readonly UpDownCounter<long> ActiveOrders =
        Meter.CreateUpDownCounter<long>("orders.active", "{order}", "Currently active orders");

    public static readonly ObservableGauge<int> QueueDepth =
        Meter.CreateObservableGauge<int>("orders.queue_depth", () =>
            new Measurement<int>(_currentQueueDepth, new KeyValuePair<string, object?>("queue", "orders")));
}

// Usage
public async Task PlaceAsync(...)
{
    var sw = Stopwatch.StartNew();
    OrderMetrics.ActiveOrders.Add(1);
    try
    {
        await DoWork();
        OrderMetrics.OrdersPlaced.Add(1, new("status", "success"));
    }
    catch
    {
        OrderMetrics.OrdersPlaced.Add(1, new("status", "failure"));
        throw;
    }
    finally
    {
        OrderMetrics.ActiveOrders.Add(-1);
        OrderMetrics.OrderProcessingDuration.Record(sw.ElapsedMilliseconds);
    }
}
```

### Instrument types

| Instrument | Когда |
|-----------|-------|
| **Counter** | Только растёт (requests_total, errors_total) |
| **UpDownCounter** | Может расти и убывать (active_connections, queue_depth) |
| **Histogram** | Distribution (latency, response size) |
| **ObservableGauge** | Текущее значение по callback'у (memory_usage, cpu_percent) |
| **ObservableCounter** | Cumulative value снимается по callback'у |
| **ObservableUpDownCounter** | Bidirectional, snapshot through callback |

### RED + USE methods

**RED — для services:**
- **R**ate — requests per second
- **E**rrors — error rate
- **D**uration — latency (p50, p95, p99)

**USE — для resources:**
- **U**tilization — % time used (CPU, disk)
- **S**aturation — backpressure (queue depth, lock contention)
- **E**rrors — error count

```csharp
// RED для HTTP endpoint
public static readonly Counter<long> Requests =
    Meter.CreateCounter<long>("http.server.requests", "{request}");
public static readonly Counter<long> Errors =
    Meter.CreateCounter<long>("http.server.errors", "{error}");
public static readonly Histogram<double> Duration =
    Meter.CreateHistogram<double>("http.server.duration", "ms");

// AddAspNetCoreInstrumentation() выдаёт это автоматически — стандартизированно по OTel semantic conventions
```

### Histogram exemplars

Exemplars — связь между metric и конкретным trace, который её произвёл. Видишь spike на latency dashboard → один клик → trace того конкретного запроса.

```csharp
// OpenTelemetry .NET 1.7+ автоматически добавляет exemplars если есть active Activity
metrics.AddPrometheusExporter(opts =>
{
    opts.WithExemplars = true;
});
```

В Grafana → click on histogram point → "Show exemplars" → получаешь link на Tempo trace.

### Cardinality — главный pitfall metrics

Каждая уникальная комбинация labels = отдельный time series. Если кладёшь user_id как label, и пользователей миллион — миллион time series. Prometheus падает.

```csharp
// ❌ BAD — high cardinality
OrderMetrics.OrdersPlaced.Add(1, new("user_id", userId));  // миллион пользователей

// ✅ GOOD — низкая cardinality
OrderMetrics.OrdersPlaced.Add(1, new("user_tier", "premium"));  // 3 значения
```

**Правила:**
- Не клади IDs (user, order, tenant) как metric labels
- Status code, error type, region, tier — OK
- Если нужно знать конкретного пользователя — он в logs/traces, не в metrics

---

## Logs

### Structured logging

Не строки, а **типизированные события**:

```csharp
// ❌ BAD
_logger.LogInformation($"User {userId} placed order {orderId} for ${total}");

// ✅ GOOD — structured
_logger.LogInformation("User {UserId} placed order {OrderId} for {Total:C}",
    userId, orderId, total);
```

В Loki/Elastic второй вариант индексируется по `UserId`, `OrderId`, `Total` отдельно. Запросы:
```
{service="orders"} | json | UserId = "abc-123"
```

### LoggerMessage source generator

Production-ready logging без аллокаций (см. [Source Generators](../CSharp/source-generators.md)):

```csharp
public partial class OrderService
{
    [LoggerMessage(
        EventId = 1001,
        Level = LogLevel.Information,
        Message = "Order {OrderId} placed for user {UserId}, total {Total:C}")]
    private partial void LogOrderPlaced(Guid orderId, Guid userId, decimal total);

    public async Task PlaceAsync(Order order, CancellationToken ct)
    {
        await _repo.SaveAsync(order, ct);
        LogOrderPlaced(order.Id, order.UserId, order.Total);
    }
}
```

EventId — **критично для filtering и alerting**. Уникальный ID каждого log event позволяет в Grafana / Seq / ELK фильтровать "только событие 1001" без regex'а на текст.

### Log scopes

```csharp
using (_logger.BeginScope(new Dictionary<string, object>
{
    ["UserId"] = userId,
    ["TenantId"] = tenantId,
    ["RequestId"] = requestId,
}))
{
    _logger.LogInformation("Started processing");  // эти fields добавляются ко всем логам в scope
    await DoWork();
    _logger.LogInformation("Finished");
}
```

### Что **не** логировать

- Passwords, tokens, secrets, API keys
- Credit cards, SSN, passport numbers (PII)
- Whole HTTP body — может содержать всё выше
- Stack traces от ожидаемых ошибок (validation failures)
- DEBUG-уровень в production (засоряет storage)

### Log levels — что когда

| Level | Когда |
|-------|-------|
| **Trace** | Hyper-detailed, dev only. В прод — никогда |
| **Debug** | Detailed flow. Dev. Прод — только при troubleshooting |
| **Information** | High-level бизнес-события (order placed, user logged in) |
| **Warning** | Recovered errors, degraded performance, retry happened |
| **Error** | Operation failed but app continues |
| **Critical** | App is in unusable state, terminating |

В прод default — **Warning**. Information выборочно для бизнес-событий.

---

## Production stack — Grafana + Loki + Tempo + Prometheus

Open-source observability stack от Grafana Labs. Полностью бесплатный (self-hosted).

```
                    ┌─────────────────┐
                    │   Application   │
                    │    (.NET app)   │
                    └────────┬────────┘
                             │ OTLP (gRPC :4317)
                             ▼
                    ┌─────────────────┐
                    │ OTel Collector  │
                    │  (агрегация,    │
                    │   sampling,     │
                    │   processing)   │
                    └─────────────────┘
                       │      │       │
                ┌──────┘      │       └───────┐
                ▼             ▼               ▼
         ┌───────────┐  ┌──────────┐   ┌──────────┐
         │   Loki    │  │  Tempo   │   │Prometheus│
         │  (logs)   │  │ (traces) │   │ (metrics)│
         └─────┬─────┘  └─────┬────┘   └──────┬───┘
               │              │               │
               └───── Grafana (visualization) ┘
```

### docker-compose snippet

```yaml
services:
  prometheus:
    image: prom/prometheus:latest
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
    ports: ["9090:9090"]

  loki:
    image: grafana/loki:latest
    ports: ["3100:3100"]

  tempo:
    image: grafana/tempo:latest
    command: ["-config.file=/etc/tempo.yaml"]
    volumes:
      - ./tempo.yaml:/etc/tempo.yaml
    ports: ["3200:3200", "4317:4317"]

  otel-collector:
    image: otel/opentelemetry-collector-contrib:latest
    command: ["--config=/etc/otelcol/config.yaml"]
    volumes:
      - ./otel-collector.yaml:/etc/otelcol/config.yaml
    ports: ["4317:4317", "4318:4318"]

  grafana:
    image: grafana/grafana:latest
    ports: ["3000:3000"]
    volumes:
      - ./grafana/provisioning:/etc/grafana/provisioning
```

### prometheus.yml

```yaml
scrape_configs:
  - job_name: 'myapp'
    metrics_path: /metrics
    static_configs:
      - targets: ['app1:8080', 'app2:8080']
```

### LogQL (Loki) queries

```
# Errors в orders service за последний час
{service="orders"} |= "ERROR" | json | __error__ = "" [1h]

# p99 latency aggregated из logs
{service="orders"} | json | line_format "{{.Duration}}" | unwrap Duration | quantile_over_time(0.99, [5m])

# Сравнение error rate per service
sum by (service) (rate({level="ERROR"} [5m]))
```

### TraceQL (Tempo) queries

```
# Traces со status error
{ status = error }

# Slow orders (> 1s) с конкретным user
{ name = "PlaceOrder" && duration > 1s && .order.user_id = "abc-123" }

# DB queries длиннее 500ms
{ db.system = "postgresql" && duration > 500ms }
```

### Deeplinks между Loki / Tempo / Prometheus

В Grafana при просмотре log с `traceID` field — клик "View trace" перебрасывает в Tempo. В Tempo при просмотре span — "Logs for this span" перебрасывает обратно в Loki с filter по `traceID`.

Главная фича — **correlation**. Видишь spike в Prometheus → click на exemplar → trace в Tempo → click на span → logs в Loki. Whole journey за 30 секунд.

---

## SLI / SLO / Error budgets

### Терминология

- **SLI** (Service Level Indicator) — измеряемая характеристика (latency, error rate, availability)
- **SLO** (Service Level Objective) — target для SLI (99.9% uptime, p99 < 500ms)
- **SLA** (Service Level Agreement) — контрактное обязательство с финансовыми последствиями
- **Error Budget** — допустимое количество "плохих" запросов в окне (1 - SLO)

### Пример

```
Service: API
SLO: 99.9% requests succeed (status 2xx) in 30 day window
     p95 latency < 500ms
Error budget: 0.1% × 30 days = 43.2 minutes/month "bad time"
```

### Burn rate alerting

Простой alert "error rate > 1%" — срабатывает на любой spike. Лучше — **burn rate**:
- "Слишком быстро тратим budget" (за 1 час съели то, что должно растягиваться на месяц)
- "Скоро нарушим SLO если так пойдёт"

```yaml
# Prometheus alert rule
- alert: HighErrorBudgetBurn
  expr: |
    (
      sum(rate(http_requests_total{job="api",status=~"5.."}[1h])) /
      sum(rate(http_requests_total{job="api"}[1h]))
    ) > 0.01  # 10x normal rate (SLO 99.9% = 0.1% threshold)
  for: 5m
  labels:
    severity: critical
  annotations:
    summary: "API burning error budget too fast"
```

### Multi-window multi-burn-rate alerts

Best practice (Google SRE Workbook):

```yaml
# Slow burn — сжигаем за 30 дней
- expr: rate(errors[1h]) > 1 * baseline_rate AND rate(errors[5m]) > 1 * baseline_rate
- expr: rate(errors[6h]) > 1 * baseline_rate AND rate(errors[30m]) > 1 * baseline_rate

# Fast burn — сжигаем за 2 часа = catastrophic
- expr: rate(errors[5m]) > 14.4 * baseline_rate AND rate(errors[1h]) > 14.4 * baseline_rate
```

---

## Health checks

### Setup

```csharp
builder.Services.AddHealthChecks()
    .AddNpgSql(builder.Configuration.GetConnectionString("Default")!, tags: ["db"])
    .AddRedis(builder.Configuration.GetConnectionString("Redis")!, tags: ["cache"])
    .AddRabbitMQ(rabbitConnectionString: ..., tags: ["broker"])
    .AddCheck<CustomHealthCheck>("custom", tags: ["custom"]);

app.MapHealthChecks("/health/live", new HealthCheckOptions
{
    Predicate = _ => false,  // Только app живёт
    ResponseWriter = UIResponseWriter.WriteHealthCheckUIResponse,
});

app.MapHealthChecks("/health/ready", new HealthCheckOptions
{
    Predicate = check => check.Tags.Contains("db") || check.Tags.Contains("cache"),
});
```

### Liveness vs Readiness (Kubernetes)

| Probe | Что значит | Что Kubernetes делает |
|-------|-----------|----------------------|
| **Liveness** | "Я жив?" — процесс работает | Если fail → restart pod |
| **Readiness** | "Я готов принимать traffic?" — DB/cache доступны | Если fail → убирает из service endpoints |
| **Startup** | "Я ещё стартую" — long-running init | Откладывает liveness/readiness |

**Не путай.** Liveness fail при недоступной БД → бесконечный restart loop. Readiness fail → pod просто исключается из LB пока БД не вернётся.

```csharp
public class DatabaseHealthCheck(AppDbContext db) : IHealthCheck
{
    public async Task<HealthCheckResult> CheckHealthAsync(
        HealthCheckContext context, CancellationToken ct = default)
    {
        try
        {
            await db.Database.ExecuteSqlRawAsync("SELECT 1", ct);
            return HealthCheckResult.Healthy();
        }
        catch (Exception ex)
        {
            return HealthCheckResult.Unhealthy("DB unavailable", ex);
        }
    }
}
```

---

## Profiling в production

### dotnet-monitor — continuous profiling

Sidecar-инструмент Microsoft для production .NET. Доступ к diagnostic events без stop-the-world.

```bash
docker pull mcr.microsoft.com/dotnet/monitor:8

# Запуск рядом с приложением (sidecar)
docker run --pid=container:myapp -p 52323:52323 mcr.microsoft.com/dotnet/monitor
```

```bash
# Снятие dump
curl -X POST http://localhost:52323/dump -o dump.dmp

# Trace на 30 секунд
curl http://localhost:52323/trace?durationSeconds=30 > trace.nettrace

# Live counters
curl http://localhost:52323/livemetrics
```

### dotnet-counters — real-time

```bash
dotnet tool install -g dotnet-counters

# Стандартные counters
dotnet-counters monitor --process-id <pid> System.Runtime

# Свои Meter
dotnet-counters monitor --process-id <pid> MyApp.Orders
```

### Profiling tools — какой когда

| Tool | Когда |
|------|-------|
| **dotnet-counters** | Real-time мониторинг RAM, GC, ThreadPool, custom Meters |
| **dotnet-trace** | CPU profiling, on-demand trace без рестарта |
| **dotnet-dump** | Memory dump для post-mortem analysis |
| **dotnet-gcdump** | Только heap, лёгкий dump |
| **PerfView** | Production crash dumps, glob analysis (только Windows) |
| **dotMemory / dotTrace** | Interactive UI, Windows + Mac |
| **VS Profiler** | Dev-time |

См. подробно в [Performance](../Performance/performance.md).

---

## Alerting strategy

### Что alert'ить

✅ **Делай alert'ом:**
- Bonking SLO (errors >= budget burn rate)
- p99 latency > target × 2 (sustained)
- Database connection pool exhausted
- Disk > 85%
- Memory > 85% sustained
- DLQ messages > 0 (5+ minutes)
- Cron job не выполнился вовремя (heartbeat)
- Certificate expires в N дней

❌ **НЕ делай alert'ом:**
- Каждая 4xx ошибка (это могут быть валидные client mistakes)
- Каждое 1% spike (false positive до alarm fatigue)
- Метрики, которые ты не понимаешь (page on confusion)
- Предупреждения, на которые не реагируют (уберают значение)

### Alert severity levels

| Level | Когда | Channel |
|-------|-------|---------|
| **Critical (P1)** | User-facing impact, требует немедленного вмешательства | PagerDuty + SMS + Slack |
| **High (P2)** | Risk of impact, требует ответа в час | PagerDuty + Slack |
| **Medium (P3)** | Issue, можно подождать рабочего дня | Slack only |
| **Low (P4)** | Informational, не требует action | Email digest |

### Runbooks

Каждый alert должен ссылаться на runbook — пошаговая инструкция что делать. Без runbook on-call инженер тратит 30 минут на диагностику в 3 утра.

```yaml
- alert: HighErrorRate
  annotations:
    summary: "API error rate > 1%"
    runbook: "https://wiki.example.com/runbooks/api-high-errors"
    dashboard: "https://grafana.example.com/d/api/api-overview"
```

---

## Distributed tracing across messaging

См. [Messaging](messaging.md) — `AddSource("MassTransit")` пробрасывает trace context через RabbitMQ messages.

```csharp
// HTTP endpoint
public async Task<IActionResult> Place([FromBody] PlaceOrderCommand cmd)
{
    // Span "POST /api/orders" автоматически
    await _publisher.Publish(new OrderPlaced(...));  // trace context в headers сообщения
    return Ok();
}

// Consumer в другом сервисе
public async Task Consume(ConsumeContext<OrderPlaced> ctx)
{
    // Span "OrderPlacedConsumer.Consume" — children of original HTTP span
    await SaveAsync(ctx.Message);  // INSERT в DB — child span
}
```

В Tempo видишь полную цепочку: HTTP → MassTransit publish → MassTransit consume → DB query.

---

## Common pitfalls

### 1. Logging в hot path
`_logger.LogTrace(...)` срабатывает миллион раз/сек → десятки MB логов/сек.
**Решение:** уровень Trace выкл в проде, LoggerMessage source-gen, sampling.

### 2. PII в logs
Утечка персональных данных через логи — GDPR нарушение.
**Решение:** redaction через `LogProperty(Redact)` (DataAnnotations), review logs sample на staging.

### 3. High cardinality metrics
`user_id` как label → миллионы time series → Prometheus падает.
**Решение:** только low-cardinality labels (status, region, tier).

### 4. Sampling всех traces
100% sampling в hi-traffic prod = TB трафика, $$$ за storage.
**Решение:** TraceIdRatioBased(0.1) + tail-sampling для errors.

### 5. Alerts на симптомы вместо причин
"CPU > 80%" — что делать? Alert должен быть на user-facing impact: "p99 latency > target".

### 6. Нет dashboards для критичных систем
Проблема на проде → начинаем строить dashboard в момент инцидента. Dashboards должны быть готовы заранее.

### 7. Структура логов меняется без warning
Раньше `{UserId}`, стало `{User.Id}` — все existing alerts ломаются.
**Решение:** semantic conventions OpenTelemetry, schema versioning.

### 8. Health check с тяжёлым query
`/health` делает `SELECT count(*) FROM big_table` → каждый probe нагружает DB.
**Решение:** `SELECT 1` или cached health state, обновляется в background.

### 9. Tracing медленный без причины
Synchronous OTLP exporter блокирует request thread.
**Решение:** batch export в background (default в .NET SDK).

### 10. Overlogging в success path
"Запрос принят", "валидация прошла", "обработано", "ответ отправлен" — 4 лога на один request × миллион requests = много.
**Решение:** один log на request с summary fields. Detail — только в trace.

---

## Production checklist

- [ ] OpenTelemetry SDK + Resource (service.name, version, environment)
- [ ] Все three pillars: logs, metrics, traces
- [ ] Auto-instrumentation: ASP.NET Core, HttpClient, Npgsql, MassTransit
- [ ] Custom ActivitySource + Meter для бизнес-логики
- [ ] Structured logging через `{Property}` syntax
- [ ] LoggerMessage source-gen для hot paths
- [ ] Sampling настроен (TraceIdRatioBased 10% + ParentBased)
- [ ] OTLP exporter в Tempo / Loki / Prometheus
- [ ] Grafana dashboards: RED для каждого service
- [ ] SLO документированы + burn rate alerts
- [ ] Health checks: liveness + readiness отдельно
- [ ] dotnet-monitor sidecar (или альтернатива)
- [ ] Runbooks для каждого alert
- [ ] PII redaction в logs
- [ ] Log retention policy (e.g. 30 дней)
- [ ] Trace deeplinks: metric → trace → log

---

## См. также

- [Logging и Observability (старая)](../AspNetCore/logging-observability.md) — базовые ILogger паттерны
- [Source Generators](../CSharp/source-generators.md) — LoggerMessage детально
- [Messaging](messaging.md) — distributed tracing через RabbitMQ
- [Distributed Systems](../Architecture/distributed-systems.md) — context propagation в saga
- [Performance](../Performance/performance.md) — profiling tools deep
- [Resilience](../AspNetCore/resilience.md) — Polly metrics, observability of failures
- [Auth и Security](../AspNetCore/auth-security.md) — что **не** логировать (PII, secrets)

## Reading list

- **OpenTelemetry .NET docs** — opentelemetry.io/docs/instrumentation/net/
- **OpenTelemetry Semantic Conventions** — opentelemetry.io/docs/specs/semconv/
- **Google SRE Workbook** — sre.google/workbook/table-of-contents/ (SLO, error budgets, alerting)
- **The Site Reliability Workbook (Ch. 5 Alerting)** — google's golden standard
- **Grafana docs** — grafana.com/docs (особенно Loki LogQL, Tempo TraceQL)
- **Honeycomb blog** — honeycomb.io/blog (deep observability стори, distributed tracing)
- **Liz Fong-Jones — Observability is a many-splendored thing** — все её talks
- **Charity Majors — Observability Engineering (O'Reilly)** — каноническая книга
- **Brendan Gregg — USE Method** — brendangregg.com/usemethod.html
- **Tom Wilkie — RED Method** — grafana.com/blog/2018/08/02/the-red-method-how-to-instrument-your-services/
