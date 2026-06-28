---
tags: [polly, resilience, httpclient, retry, circuit-breaker, timeout, hedging, observability]
level: Senior
---

# Resilience и HttpClient

> Устойчивость к сбоям зависимостей через Polly v8 — Retry, Timeout, Circuit Breaker, Hedging, Bulkhead и Fallback поверх typed `HttpClient`.

## Что это, зачем и когда

### Что такое resilience?
**Способность системы продолжать работать при сбоях зависимостей.** Downstream сервис тормозит → твой сервис не валится. БД недоступна 5 секунд → пользователи не страдают.

**Аналогия:** Машина с запаской и резервным маслом. Прокол не означает «всё, машина в гараж» — едешь дальше.

### Главные сценарии

| Проблема | Pattern | Решение |
|----------|---------|---------|
| Transient error (5xx, network blip) | **Retry** | Повторить запрос с backoff |
| Запрос завис | **Timeout** | Не ждать вечно, отвалиться через N сек |
| Downstream дохлый | **Circuit Breaker** | Не насиловать его, fail fast |
| Один replica медленный | **Hedging** | Параллельно дёрнуть второй, взять первый ответ |
| Upstream бомбит RPS | **Rate Limiter** | Не пропускать больше N RPS |
| Большой объём данных | **Bulkhead** | Изолировать concurrent ops |
| Нет данных, downstream упал | **Fallback** | Вернуть default / cached |

### Когда применять

| Применять | Не применять |
|-----------|--------------|
| Любой external HTTP вызов | In-memory math operations |
| DB connections (через EF retry) | Pure compute |
| Message broker calls | Unit tests (mock'и не падают) |
| Cache (Redis) | Конфигурационные lookup'ы |
| File I/O в облаке (S3) | Local disk reads |

---

## Polly v8 — основной инструмент

`Polly` — стандарт для resilience patterns в .NET. v8+ полностью переработан — pipelines vs predicates вместо chained policies.

```bash
dotnet add package Polly
dotnet add package Microsoft.Extensions.Http.Resilience  # для HttpClient
```

### Базовый pipeline

```csharp
var pipeline = new ResiliencePipelineBuilder()
    .AddRetry(new RetryStrategyOptions
    {
        MaxRetryAttempts = 3,
        Delay = TimeSpan.FromSeconds(1),
        BackoffType = DelayBackoffType.Exponential,
        UseJitter = true,
        ShouldHandle = new PredicateBuilder()
            .Handle<HttpRequestException>()
            .Handle<TimeoutRejectedException>(),
    })
    .AddTimeout(TimeSpan.FromSeconds(30))
    .AddCircuitBreaker(new CircuitBreakerStrategyOptions
    {
        FailureRatio = 0.5,
        MinimumThroughput = 5,
        SamplingDuration = TimeSpan.FromSeconds(30),
        BreakDuration = TimeSpan.FromSeconds(15),
    })
    .Build();

// Execute
var result = await pipeline.ExecuteAsync(async ct =>
{
    return await ExternalApiCallAsync(ct);
});
```

### Polly v7 → v8 миграция

| | v7 | v8 |
|--|-----|-----|
| Базовый класс | `Policy` | `ResiliencePipelineBuilder` |
| Combine | `Policy.WrapAsync(retry, timeout)` | `.AddRetry(...).AddTimeout(...)` |
| Generic vs non-generic | `Policy<T>` отдельно | Унифицированный API |
| Predicates | Through callbacks | `PredicateBuilder` |
| Performance | Аллокации | Намного меньше |

Если есть код на v7 — мигрируй на v8. Перформанс лучше, API чище.

---

## Retry

### Стратегии backoff

```csharp
.AddRetry(new RetryStrategyOptions
{
    MaxRetryAttempts = 3,
    BackoffType = DelayBackoffType.Exponential,  // 1s, 2s, 4s, 8s...
    Delay = TimeSpan.FromSeconds(1),
    UseJitter = true,                              // случайность ±25% — anti thundering herd
    ShouldHandle = new PredicateBuilder()
        .Handle<HttpRequestException>()
        .Handle<TimeoutRejectedException>()
        .HandleResult(r => r is HttpResponseMessage h &&
            (h.StatusCode == HttpStatusCode.RequestTimeout ||
             h.StatusCode == HttpStatusCode.ServiceUnavailable ||
             h.StatusCode == HttpStatusCode.TooManyRequests)),
    OnRetry = args =>
    {
        _logger.LogWarning(
            "Retry #{Attempt} after {Delay}, status: {Status}",
            args.AttemptNumber, args.RetryDelay, args.Outcome.Result);
        return ValueTask.CompletedTask;
    },
});
```

| BackoffType | Когда |
|------------|-------|
| **Constant** | Predictable downstream — фиксированная задержка |
| **Linear** | Лёгкая нагрузка — равномерное увеличение |
| **Exponential** | Default — даёт downstream'у время восстановиться |

**Jitter обязателен.** Без jitter тысячи инстансов retry'ятся одновременно через 1s → 2s → 4s, бомбят downstream синхронными волнами. Jitter ±25% размывает их во времени.

### `RetryAfter` header

Если downstream шлёт `Retry-After: 30`, уважай:

```csharp
.AddRetry(new RetryStrategyOptions
{
    DelayGenerator = args =>
    {
        if (args.Outcome.Result is HttpResponseMessage response &&
            response.Headers.TryGetValues("Retry-After", out var values) &&
            int.TryParse(values.FirstOrDefault(), out var seconds))
        {
            return ValueTask.FromResult<TimeSpan?>(TimeSpan.FromSeconds(seconds));
        }
        return ValueTask.FromResult<TimeSpan?>(null);  // null → стандартный backoff
    },
});
```

### Когда **не** retry

- POST не-идемпотентные операции (если downstream не support'ит Idempotency-Key)
- 4xx ошибки (это твоя bag, retry не поможет)
- Аутентификация ошибки (проблема в credentials)
- Validation errors

> [!question]- **Интервью: какие 4xx стоит retry?**
> Только некоторые:
> - **408 Request Timeout** — downstream не успел, retry OK
> - **429 Too Many Requests** — обязательно с уважением `Retry-After`
> - **425 Too Early** — TLS handshake retry
>
> Все остальные 4xx (400 Bad Request, 401 Unauthorized, 403 Forbidden, 404 Not Found) — это **твоя** ошибка. Retry не поможет.

---

## Timeout

```csharp
.AddTimeout(TimeSpan.FromSeconds(10))

// или с custom logic
.AddTimeout(new TimeoutStrategyOptions
{
    Timeout = TimeSpan.FromSeconds(10),
    OnTimeout = args =>
    {
        _logger.LogWarning("Operation timed out after {Timeout}", args.Timeout);
        return ValueTask.CompletedTask;
    },
});
```

### Pessimistic vs Optimistic timeout

```csharp
// Pessimistic — для не-cooperative операций (не support CancellationToken)
.AddTimeout(new TimeoutStrategyOptions
{
    Timeout = TimeSpan.FromSeconds(10),
    TimeoutMode = TimeoutMode.Pessimistic,  // принудительно прерывает thread
});

// Optimistic — default, через CancellationToken
.AddTimeout(TimeSpan.FromSeconds(10));
```

**Default — optimistic.** Pessimistic прерывает thread что **опасно** (можно повредить state), используй только для legacy кода без CancellationToken.

### Total vs Per-Attempt timeout

```csharp
// Per-attempt timeout (для одной попытки)
var pipeline = new ResiliencePipelineBuilder()
    .AddTimeout(TimeSpan.FromSeconds(5))   // ← per-attempt
    .AddRetry(...)                          // 3 retries
    .Build();
// Каждая retry попытка имеет 5s таймаут. Total cap = 5 × 3 = 15s.

// Total timeout (на весь pipeline вместе с retries)
var pipeline = new ResiliencePipelineBuilder()
    .AddTimeout(TimeSpan.FromSeconds(20))  // ← total cap
    .AddRetry(...)
    .AddTimeout(TimeSpan.FromSeconds(5))   // ← per-attempt
    .Build();
// Total 20s, одна попытка — макс 5s
```

**Порядок важен!** Outer pipeline = total cap, inner = per-attempt.

---

## Circuit Breaker

```csharp
.AddCircuitBreaker(new CircuitBreakerStrategyOptions
{
    FailureRatio = 0.5,                              // 50% failures → open
    MinimumThroughput = 10,                          // Минимум 10 запросов за окно
    SamplingDuration = TimeSpan.FromSeconds(30),     // Окно — 30 секунд
    BreakDuration = TimeSpan.FromSeconds(15),        // Open на 15 секунд
    OnOpened = args =>
    {
        _logger.LogError("Circuit OPENED for {Duration}", args.BreakDuration);
        _metrics.CircuitBreakerOpened.Add(1);
        return ValueTask.CompletedTask;
    },
    OnHalfOpened = args =>
    {
        _logger.LogInformation("Circuit HALF-OPEN — probing");
        return ValueTask.CompletedTask;
    },
    OnClosed = args =>
    {
        _logger.LogInformation("Circuit CLOSED — recovered");
        return ValueTask.CompletedTask;
    },
});
```

### State machine

```
   CLOSED  ──────────────►  OPEN
     ▲     too many fails    │
     │                       │ break duration
     │                       ▼
     │                    HALF-OPEN
     │  success            │      │  fail
     └────────────────────┘      └──────────► OPEN
```

| State | Что происходит |
|-------|----------------|
| **Closed** | Нормальная работа, считаем failures |
| **Open** | Все запросы **сразу** падают с `BrokenCircuitException` (fast fail) |
| **Half-Open** | Один пробный запрос — успех = Closed, fail = Open снова |

### Зачем
Без CB: downstream упал → 1000 RPS превращаются в 1000 RPS таймаутов и retry'ев → нагружаем дохлого downstream'а ещё сильнее.
С CB: 50% requests fail → CB Open → следующие 15 секунд **0 запросов** к downstream → он рестартует — мы возобновляемся.

### Pitfalls

```csharp
// ❌ MinimumThroughput слишком низкий
options.MinimumThroughput = 1;
options.FailureRatio = 0.5;
// Один failure → ratio 1/1 = 100% → CB Open. Catastrophic false positive.

// ✅ Reasonable threshold
options.MinimumThroughput = 10;
options.FailureRatio = 0.5;  // CB Open только если 5+ из 10 fail
```

> [!question]- **Интервью: что такое Half-Open state?**
> После BreakDuration CB переходит в Half-Open. Пропускает **один** пробный запрос:
> - **Success** → возвращается в Closed (downstream выздоровел)
> - **Fail** → снова Open (ещё подождём BreakDuration)
>
> Без Half-Open мы бы либо вечно стояли в Open, либо открыли бы и сразу нагнали downstream при возврате нагрузки. Half-Open — gradual probing.

---

## Hedging — параллельное дублирование

Если у downstream есть N replicas — можешь параллельно дёрнуть несколько, взять первый response.

```csharp
.AddHedging(new HedgingStrategyOptions<HttpResponseMessage>
{
    MaxHedgedAttempts = 3,
    Delay = TimeSpan.FromMilliseconds(200),  // Через 200ms запускается второй
    ActionGenerator = args =>
    {
        return () => CallAlternateReplicaAsync(args.AttemptNumber, args.ActionContext);
    },
});
```

### Когда применять

| Применять | Не применять |
|-----------|--------------|
| Read-heavy operations с idempotency | Write-операции (двойной charge!) |
| Latency-sensitive (видеостриминг, search) | Cost-sensitive (платишь за два API call) |
| Несколько replicas/endpoints | Один downstream |
| p99 latency критичен | p50 OK |

Hedging **снижает p99 latency** на cost of дополнительного RPS на downstream. Используй для critical paths где latency важнее cost.

---

## Fallback

Когда всё упало — вернуть default / cached / degraded response.

```csharp
.AddFallback(new FallbackStrategyOptions<HttpResponseMessage>
{
    FallbackAction = args =>
    {
        return Outcome.FromResultAsValueTask(new HttpResponseMessage(HttpStatusCode.OK)
        {
            Content = new StringContent("[]"),  // Empty list as fallback
        });
    },
    OnFallback = args =>
    {
        _logger.LogWarning("Falling back due to {Exception}", args.Outcome.Exception?.Message);
        return ValueTask.CompletedTask;
    },
});
```

### Cache как fallback

Если downstream упал — отдай stale из cache:

```csharp
public async Task<Product?> GetResilientAsync(int id, CancellationToken ct)
{
    try
    {
        var fresh = await _pipeline.ExecuteAsync(async c =>
            await _api.GetAsync(id, c), ct);

        await _cache.SetAsync($"product:{id}", fresh, TimeSpan.FromHours(24), ct);
        return fresh;
    }
    catch (Exception ex) when (ex is HttpRequestException or TimeoutException or BrokenCircuitException)
    {
        _logger.LogWarning(ex, "Downstream failed, using cached");
        return await _cache.GetAsync<Product>($"product:{id}", ct);  // stale OK
    }
}
```

См. [Caching](caching.md) — детали cache resilience pattern.

---

## Bulkhead — изолированные thread pools

Старая концепция Polly v7. В v8 — через `ConcurrencyLimiter`:

```csharp
.AddConcurrencyLimiter(new RateLimiterStrategyOptions
{
    DefaultRateLimiterOptions = new ConcurrencyLimiterOptions
    {
        PermitLimit = 100,
        QueueLimit = 100,
    },
});
```

100 параллельных операций max + 100 в queue. 201-й запрос получит rejection.

### Зачем

Без bulkhead: 1000 параллельных HTTP calls к slow downstream → ThreadPool starvation → весь сервис тормозит.
С bulkhead: только 100 параллельных → быстрее fail fast для лишних → ThreadPool свободен для других операций.

См. также [HFT / Low-Latency]() — `Channel<T>` как natural bulkhead.

---

## Microsoft.Extensions.Http.Resilience — для HttpClient

Высокоуровневая обёртка над Polly специально для HttpClient.

### Standard Resilience Handler

```csharp
builder.Services.AddHttpClient<MyApiClient>(client =>
{
    client.BaseAddress = new Uri("https://api.example.com");
})
.AddStandardResilienceHandler();  // ВСЁ: retry + timeout + CB + total timeout + rate limiter
```

`AddStandardResilienceHandler` даёт **production-ready defaults**:
- Total timeout 30s
- Per-attempt timeout 10s
- Retry: 3 attempts, exponential backoff, jitter
- Circuit breaker: 10% failure ratio, 30s sampling
- Rate limiter: 1000 RPS

### Customization

```csharp
.AddStandardResilienceHandler(options =>
{
    options.Retry.MaxRetryAttempts = 5;
    options.Retry.Delay = TimeSpan.FromSeconds(2);
    options.AttemptTimeout.Timeout = TimeSpan.FromSeconds(5);
    options.CircuitBreaker.SamplingDuration = TimeSpan.FromMinutes(1);
    options.TotalRequestTimeout.Timeout = TimeSpan.FromSeconds(60);
});
```

### Hedging Handler

```csharp
.AddStandardHedgingHandler(handler =>
{
    handler.UrlGroupBuilder.SetUrls("https://api1.example.com", "https://api2.example.com");
    handler.HedgingOptions.MaxHedgedAttempts = 3;
    handler.HedgingOptions.Delay = TimeSpan.FromMilliseconds(200);
});
```

Используй когда:
- `Standard` — для большинства HTTP-клиентов
- `Hedging` — когда есть несколько endpoints (geo-replicas, load-balanced backends)

---

## Typed HttpClient pattern

```csharp
public sealed class WeatherClient(HttpClient http, ILogger<WeatherClient> logger)
{
    public async Task<WeatherResponse?> GetAsync(string city, CancellationToken ct)
    {
        try
        {
            return await http.GetFromJsonAsync<WeatherResponse>(
                $"weather?city={Uri.EscapeDataString(city)}", ct);
        }
        catch (HttpRequestException ex)
        {
            logger.LogWarning(ex, "Weather API failure for {City}", city);
            return null;
        }
    }
}

// Регистрация
builder.Services.AddHttpClient<WeatherClient>(client =>
{
    client.BaseAddress = new Uri(builder.Configuration["WeatherApi:BaseUrl"]!);
    client.DefaultRequestHeaders.Add("X-API-Key", builder.Configuration["WeatherApi:ApiKey"]!);
    client.Timeout = TimeSpan.FromSeconds(60);  // hard cap
})
.AddStandardResilienceHandler();
```

См. [auth-security.md / OpenAI typed HttpClient](llm-rag-patterns.md#openai-через-typed-httpclient--addresiliencehandler) — расширенный пример с custom config.

---

## Observability of resilience

Сама resilience политика — это **поведение системы**. Должны быть metrics:

```csharp
// Polly v8 has OpenTelemetry integration out of the box
builder.Services
    .ConfigureHttpClientDefaults(b => b.AddStandardResilienceHandler())
    .AddOpenTelemetry()
    .WithMetrics(metrics => metrics
        .AddMeter("Polly")            // Resilience metrics
        .AddPrometheusExporter());
```

### Что мониторить

| Metric | Almuni alert |
|--------|-------------|
| `polly.retry.count` | Резко растёт → downstream проблема |
| `polly.circuit_breaker.state` | Open state длительный → инцидент |
| `polly.timeout.count` | Растут → downstream slow |
| `polly.hedging.count` | Когда hedging активен — насколько часто |
| `polly.fallback.count` | Каждый fallback = degraded UX |

Smart alerts:
```yaml
- alert: CircuitBreakerStuckOpen
  expr: polly_circuit_breaker_state{state="open"} == 1
  for: 5m
  severity: critical
```

См. [Observability]() — full setup.

---

## Custom strategies

В Polly v8 можно писать свои strategies:

```csharp
public class JitteredDelayStrategy<T> : ResilienceStrategy<T>
{
    protected override async ValueTask<Outcome<T>> ExecuteCore<TState>(
        Func<ResilienceContext, TState, ValueTask<Outcome<T>>> callback,
        ResilienceContext context,
        TState state)
    {
        var jitter = TimeSpan.FromMilliseconds(Random.Shared.Next(0, 100));
        await Task.Delay(jitter, context.CancellationToken);
        return await callback(context, state);
    }
}

// Регистрация
.AddStrategy(new JitteredDelayStrategy<HttpResponseMessage>());
```

Полезно для domain-specific patterns (chaos engineering, custom backoff формулы, business rules).

---

## Chaos engineering — Polly Simmy

```csharp
.AddChaosFault(new ChaosFaultStrategyOptions
{
    InjectionRate = 0.1,  // 10% запросов
    Enabled = builder.Environment.IsDevelopment(),
    Fault = new InvalidOperationException("Chaos!"),
});

.AddChaosLatency(new ChaosLatencyStrategyOptions
{
    InjectionRate = 0.2,  // 20% запросов
    Latency = TimeSpan.FromSeconds(5),
    Enabled = builder.Environment.IsDevelopment(),
});
```

Включай в Dev/Staging для тестирования что система переживает random failures. **Никогда** в production.

См. подробно — Netflix Chaos Monkey, AWS Fault Injection Service. Same idea — proactively добавляем хаос чтобы найти baги до prod.

---

## Common pitfalls

### 1. Retry на 4xx

```csharp
// ❌ Retry POST который вернул 400 Bad Request — бесполезно, тело запроса плохое
.HandleResult(r => !r.IsSuccessStatusCode)

// ✅ Только transient
.HandleResult(r => r.StatusCode == HttpStatusCode.RequestTimeout
    || r.StatusCode == HttpStatusCode.ServiceUnavailable
    || r.StatusCode == HttpStatusCode.TooManyRequests
    || (int)r.StatusCode >= 500)
```

### 2. Retry без идемпотентности
POST без Idempotency-Key + retry = двойной charge / двойной заказ.
**Решение:** server поддерживает Idempotency-Key, или просто не retry POST.

### 3. Retry-storm
1000 instances retry'ятся одновременно через 1s → 2s → 4s — синхронные волны убивают downstream.
**Решение:** ВСЕГДА `UseJitter = true`.

### 4. Total timeout слишком маленький
`Timeout = 10s, MaxRetries = 3, Delay = 5s` — total > 10s, но timeout = 10s → не дождавшись retry, abort.
**Решение:** total timeout = (per-attempt × max_attempts) + buffer.

### 5. CB MinimumThroughput=1
Один failure → 100% ratio → CB Open. Catastrophic false positive.
**Решение:** `MinimumThroughput >= 10`.

### 6. Logging deeplinks с PII
Retry log содержит request body с password / credit card.
**Решение:** redaction layer перед logging, scrub sensitive fields.

### 7. Synchronous waiting блокирует ThreadPool
Polly thread-pool sleeping → ThreadPool starvation.
**Решение:** v8 fully async, но проверь что callback тоже async.

### 8. CB без observability
CB Open = твоя система degraded, но никто не знает.
**Решение:** OnOpened → metric + alert + Slack notification.

### 9. Не учитывается cumulative impact
Retry × 3 + per-attempt timeout 10s + circuit breaker delays = пользователь ждёт 60 секунд → UX terrible.
**Решение:** real-world UX testing — фиксируй p99 latency end-to-end.

### 10. Forget `AddStandardResilienceHandler`
Каждый new HttpClient без resilience = production риск.
**Решение:** `ConfigureHttpClientDefaults(b => b.AddStandardResilienceHandler())` глобально, opt-out для exceptions.

---

## Production checklist

- [ ] `AddStandardResilienceHandler` для всех HttpClient
- [ ] Per-attempt timeout < total timeout
- [ ] Retry only on transient errors (5xx, 408, 429, 425)
- [ ] `UseJitter = true` обязательно
- [ ] Circuit Breaker `MinimumThroughput >= 10`
- [ ] CB metrics + alerts (state Open за 5+ минут)
- [ ] Hedging для read-heavy idempotent ops (если есть replicas)
- [ ] Fallback strategy для критичных endpoints
- [ ] PII redaction в retry logs
- [ ] Polly OpenTelemetry meter active
- [ ] Chaos testing в staging environment
- [ ] EF Core `EnableRetryOnFailure` для DB transient errors
- [ ] Documented retry policies per service (в ADR)
- [ ] Idempotency-Key для retry-able POST endpoints

---

## См. также

- [Auth и Security](auth-security.md) — typed HttpClient + auth integration
- [Caching](caching.md) — cache как fallback
- [Observability]() — Polly metrics, alerting
- [Distributed Systems]() — idempotency для retry safety
- [HFT / Low-Latency]() — Channel<T> как bulkhead
- [System Design]() — resilience в архитектуре

## Reading list

- **Polly docs (v8)** — pollydocs.org
- **Microsoft.Extensions.Http.Resilience docs** — learn.microsoft.com/dotnet/core/resilience/http-resilience
- **Release It! (2nd ed.)** — Michael Nygard (canonical book on resilience patterns)
- **AWS Builders' Library — Timeouts, Retries, Backoff** — aws.amazon.com/builders-library/
- **Marc Brooker — Exponential Backoff and Jitter** — aws.amazon.com/blogs/architecture/exponential-backoff-and-jitter/
- **Google SRE Workbook (Ch. Cascading Failures)** — sre.google
- **Hystrix legacy** — Netflix' original CB pattern (concepts транслируются на Polly)
- **Polly Cancellation discussion** — github.com/App-vNext/Polly/issues — для глубоких deep-dives
