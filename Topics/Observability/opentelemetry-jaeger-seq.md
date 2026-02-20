# OpenTelemetry в .NET с Jaeger и Seq

> По материалам: [OpenTelemetry in .NET with Jaeger and Seq](https://antondevtips.com/blog/getting-started-with-open-telemetry-in-dotnet-with-jaeger-and-seq/)

## Что такое OpenTelemetry

Стандартизированный сбор телеметрии: логи, метрики, трейсы. Vendor-neutral, extensible, lightweight.

## Типы данных

| Тип | Назначение |
|-----|------------|
| **Logs** | События с timestamp |
| **Traces** | Путь запроса через сервисы, spans |
| **Metrics** | Counters, Gauges, Histograms, Summaries |

## Пакеты

```
OpenTelemetry.Extensions.Hosting
OpenTelemetry.Instrumentation.AspNetCore
OpenTelemetry.Instrumentation.Http
OpenTelemetry.Instrumentation.Runtime
Npgsql.OpenTelemetry
OpenTelemetry.Instrumentation.StackExchangeRedis
OpenTelemetry.Exporter.OpenTelemetryProtocol
```

## Конфигурация

```csharp
services.AddOpenTelemetry()
    .ConfigureResource(r => r.AddService("MyService"))
    .WithTracing(t => t
        .AddAspNetCoreInstrumentation()
        .AddHttpClientInstrumentation()
        .AddNpgsql()
        .AddOtlpExporter());
```

## Jaeger

Docker, порт 16686. OTLP endpoint: `http://localhost:4318`. Визуализация трейсов.

## Seq

Логи + трейсы в одном месте. Docker, порт 8081. OTLP: `http://localhost:5341/ingest/otlp/v1/traces`. API Key для prod.

## Custom traces

```csharp
var tracer = TracerProvider.Default.GetTracer("MyApp");
using var span = tracer.StartActiveSpan("OperationName");
span.SetAttribute("key", "value");
```

## Best practices

- Не добавлять PII в spans
- Иерархия parent-child
- Записывать исключения
- Осмысленные атрибуты
- Не трассировать всё подряд

---

## Best Practices (дополнительно)

- **Correlation ID** — один на запрос. Пробрасывать в заголовках, логи, spans. Без него трейсы не связать.
- **Sampling** — в prod не трассировать 100%. Head-based sampling для низкой нагрузки, tail-based для отладки.
- **Span names** — `{Entity}.{Action}`: `Order.Create`, `Order.GetById`. Не `Method1`, `DoStuff`.
- **Attributes** — `http.status_code`, `db.operation`, `messaging.operation`. Стандартные семантические конвенции.
- **Логи vs трейсы** — логи для событий, трейсы для latency. Не дублировать одно и то же в оба.
- **Prod** — Jaeger без auth. Для prod — Grafana Tempo, Jaeger с ingress, или managed (Application Insights).

---

## См. также

- [[Topics/Docker/docker-deploy|Docker и CI/CD]]
- [[Topics/Performance/dotnet-performance|.NET Performance]]
- [[dotnet-knowledge-base|.NET Knowledge Base]]
