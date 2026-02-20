# OpenTelemetry в .NET с Jaeger и Seq

> По материалам: [OpenTelemetry in .NET with Jaeger and Seq](https://antondevtips.com/blog/getting-started-with-open-telemetry-in-dotnet-with-jaeger-and-seq/)

## Что такое OpenTelemetry

Стандартизированный vendor-neutral фреймворк для сбора телеметрии: логи, метрики, трейсы. Единый API для экспорта данных в любую систему (Jaeger, Prometheus, Grafana, Application Insights).

## Три столпа observability

| Тип | Назначение | Пример | Инструмент |
|-----|------------|--------|------------|
| **Logs** | События с timestamp и контекстом | "Order 123 created" | Seq, ELK, Loki |
| **Traces** | Путь запроса через сервисы (spans) | HTTP → Service → DB | Jaeger, Tempo |
| **Metrics** | Числовые показатели во времени | RPS, latency p99, error rate | Prometheus, Grafana |

### Как связаны

```
Correlation ID: abc-123
├── Trace (путь запроса)
│   ├── Span: HTTP GET /api/orders (50ms)
│   ├── Span: OrderService.GetAsync (30ms)
│   └── Span: PostgreSQL SELECT (20ms)
├── Logs (события)
│   ├── [Info] Order 123 requested (CorrelationId=abc-123)
│   └── [Info] Order 123 found (CorrelationId=abc-123)
└── Metrics (счётчики)
    └── http_requests_total{method="GET",status="200"} += 1
```

---

## Пакеты

```xml
<!-- Tracing -->
<PackageReference Include="OpenTelemetry.Extensions.Hosting" />
<PackageReference Include="OpenTelemetry.Instrumentation.AspNetCore" />
<PackageReference Include="OpenTelemetry.Instrumentation.Http" />
<PackageReference Include="OpenTelemetry.Instrumentation.Runtime" />

<!-- Провайдеры БД -->
<PackageReference Include="Npgsql.OpenTelemetry" />
<PackageReference Include="OpenTelemetry.Instrumentation.StackExchangeRedis" />

<!-- Exporters -->
<PackageReference Include="OpenTelemetry.Exporter.OpenTelemetryProtocol" />  <!-- OTLP -->
<PackageReference Include="OpenTelemetry.Exporter.Prometheus.AspNetCore" /> <!-- Prometheus -->
```

---

## Конфигурация

### Tracing

```csharp
builder.Services.AddOpenTelemetry()
    .ConfigureResource(r => r
        .AddService(
            serviceName: "OrderService",
            serviceVersion: "1.0.0",
            serviceInstanceId: Environment.MachineName))
    .WithTracing(tracing => tracing
        .AddAspNetCoreInstrumentation(opts =>
        {
            // Фильтр — не трассировать health checks
            opts.Filter = ctx => !ctx.Request.Path.StartsWithSegments("/health");
        })
        .AddHttpClientInstrumentation()
        .AddNpgsql()
        .AddSource("MyApp")  // кастомные Activity sources
        .AddOtlpExporter(opts =>
        {
            opts.Endpoint = new Uri("http://localhost:4317"); // gRPC
        }));
```

### Metrics

```csharp
builder.Services.AddOpenTelemetry()
    .WithMetrics(metrics => metrics
        .AddAspNetCoreInstrumentation()
        .AddHttpClientInstrumentation()
        .AddRuntimeInstrumentation()    // GC, ThreadPool, Memory
        .AddMeter("MyApp")             // кастомные метрики
        .AddPrometheusExporter());      // /metrics endpoint

// Prometheus endpoint
app.MapPrometheusScrapingEndpoint(); // GET /metrics
```

### Logging (OTLP export)

```csharp
builder.Logging.AddOpenTelemetry(opts =>
{
    opts.IncludeFormattedMessage = true;
    opts.IncludeScopes = true;
    opts.AddOtlpExporter();
});
```

---

## Custom Traces (Activity)

```csharp
public class OrderService(IOrderRepository repo)
{
    // Определяем ActivitySource (аналог TracerProvider)
    private static readonly ActivitySource Activity = new("MyApp.OrderService");

    public async Task<Order?> GetByIdAsync(Guid id, CancellationToken ct)
    {
        // Создаём span
        using var activity = Activity.StartActivity("OrderService.GetById");
        activity?.SetTag("order.id", id.ToString());

        var order = await repo.GetByIdAsync(id, ct);

        if (order is null)
        {
            activity?.SetTag("order.found", false);
            activity?.SetStatus(ActivityStatusCode.Error, "Order not found");
        }
        else
        {
            activity?.SetTag("order.found", true);
            activity?.SetTag("order.total", order.Total);
        }

        return order;
    }
}
```

**Нюанс:** `Activity` — .NET-реализация OpenTelemetry Span. `using` — автоматическое завершение span. `?.` — если tracing отключён, Activity = null (no-op).

---

## Custom Metrics

```csharp
public class OrderMetrics
{
    private readonly Counter<long> _ordersCreated;
    private readonly Histogram<double> _orderProcessingDuration;

    public OrderMetrics(IMeterFactory meterFactory)
    {
        var meter = meterFactory.Create("MyApp");
        _ordersCreated = meter.CreateCounter<long>(
            "orders.created",
            description: "Number of orders created");
        _orderProcessingDuration = meter.CreateHistogram<double>(
            "orders.processing.duration",
            unit: "ms",
            description: "Order processing duration");
    }

    public void OrderCreated(string status)
        => _ordersCreated.Add(1, new KeyValuePair<string, object?>("status", status));

    public void RecordDuration(double ms)
        => _orderProcessingDuration.Record(ms);
}
```

---

## Jaeger

Распределённая трассировка. Визуализация spans, waterfall-диаграммы, сравнение трейсов.

```yaml
# docker-compose
jaeger:
  image: jaegertracing/all-in-one:1.54
  ports:
    - "16686:16686"  # UI
    - "4317:4317"    # OTLP gRPC
    - "4318:4318"    # OTLP HTTP
  environment:
    COLLECTOR_OTLP_ENABLED: true
```

Открыть UI: `http://localhost:16686`

---

## Seq

Structured logging + tracing. Полнотекстовый поиск по логам, SQL-like запросы, дашборды.

```yaml
seq:
  image: datalust/seq:2024
  ports:
    - "8081:80"     # UI
    - "5341:5341"   # Ingestion (OTLP, Serilog)
  environment:
    ACCEPT_EULA: Y
```

### Serilog → Seq

```csharp
builder.Host.UseSerilog((ctx, cfg) => cfg
    .ReadFrom.Configuration(ctx.Configuration)
    .WriteTo.Seq("http://localhost:5341")
    .Enrich.WithProperty("Application", "OrderService")
    .Enrich.WithCorrelationId());
```

---

## Grafana + Prometheus + Tempo (production stack)

```yaml
# docker-compose для полного observability стека
prometheus:
  image: prom/prometheus:v2.51.0
  volumes:
    - ./prometheus.yml:/etc/prometheus/prometheus.yml
  ports:
    - "9090:9090"

grafana:
  image: grafana/grafana:10.4.0
  ports:
    - "3000:3000"
  environment:
    GF_AUTH_ANONYMOUS_ENABLED: true

tempo:
  image: grafana/tempo:2.4.0
  ports:
    - "4317:4317"  # OTLP gRPC
```

**Нюанс:** Jaeger — для разработки и простых сценариев. Для production — Grafana Tempo (масштабируется, дешёвое хранение в S3) или managed решения (Azure Application Insights, Datadog).

---

## Correlation ID

```csharp
// Middleware — пробрасывает Correlation ID через все сервисы
public class CorrelationIdMiddleware(RequestDelegate next)
{
    public async Task InvokeAsync(HttpContext context)
    {
        var correlationId = context.Request.Headers["X-Correlation-Id"].FirstOrDefault()
            ?? Guid.NewGuid().ToString();

        context.Items["CorrelationId"] = correlationId;
        context.Response.Headers["X-Correlation-Id"] = correlationId;

        // Добавляем в лог-скоуп — все логи в рамках запроса получат CorrelationId
        using (context.RequestServices
            .GetRequiredService<ILogger<CorrelationIdMiddleware>>()
            .BeginScope(new Dictionary<string, object> { ["CorrelationId"] = correlationId }))
        {
            await next(context);
        }
    }
}
```

---

## Best Practices

- **Correlation ID** — один на запрос. Пробрасывать в заголовках, логах, spans. Без него трейсы не связать.
- **Sampling** — в prod не трассировать 100%. Head-based sampling для низкой нагрузки, tail-based для отладки.
- **Span names** — `{Entity}.{Action}`: `Order.Create`, `Order.GetById`. Не `Method1`, `DoStuff`.
- **Attributes** — `http.status_code`, `db.operation`, `messaging.operation`. Стандартные семантические конвенции.
- **Не трассировать health checks** — фильтр в инструментации. Иначе шум в трейсах.
- **Логи vs трейсы** — логи для событий и деталей, трейсы для latency и зависимостей. Не дублировать.
- **PII** — не добавлять персональные данные в spans и метрики. Только в логи (с маскированием).
- **Prod** — Jaeger/Seq без auth только для dev. Для prod — Grafana stack, managed APM, или ingress с auth.

---

## См. также

- [Docker и CI/CD](../Docker/docker-deploy.md)
- [.NET Performance](../Performance/dotnet-performance.md)
- [Interview: Logging и метрики](../../Interview/6-logging-metrics.md)
