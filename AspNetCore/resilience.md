---
tags: [resilience, polly, httpclient, retry, circuit-breaker]
level: Senior
---

# Resilience и HttpClient

## Что это, зачем и когда

### Что такое Resilience?
**Устойчивость приложения к сбоям.** Внешний сервис упал? Не падай вместе с ним — повтори запрос, подожди, верни fallback. Resilience — набор стратегий для graceful degradation.

**Аналогия:** Ты звонишь другу, а он не берёт трубку. **Без resilience** — паникуешь и падаешь. **С resilience** — перезваниваешь через 5 секунд (retry), если не ответил 3 раза — шлёшь SMS (fallback), если вообще недоступен — не звонишь час (circuit breaker).

### Зачем?

| Без Resilience | С Resilience |
|---------------|-------------|
| Внешний API вернул 500 → твой API упал | Retry через 1с → 2с → 4с → успех |
| API медленно отвечает → все потоки заняты, приложение зависло | Timeout 5с → отменяем → возвращаем ошибку |
| Сервис лёг → тысячи retry добивают его | Circuit Breaker: «сервис мёртв, не шли запросы 30 сек» |
| `new HttpClient()` в каждом методе → утечка сокетов | `IHttpClientFactory` → пул соединений, DNS refresh |

### Когда что?

| Стратегия | Когда | Параметры |
|-----------|-------|-----------|
| **Retry** | Transient ошибки (500, timeout, сеть) | 3 попытки, exponential backoff |
| **Circuit Breaker** | Сервис стабильно недоступен | 5 ошибок → открыть на 30 сек |
| **Timeout** | Защита от зависших запросов | 5-30 сек в зависимости от сервиса |
| **Fallback** | Есть запасной вариант (кеш, default) | Кешированный ответ, пустой список |
| **Rate Limiter** | Не перегружать внешний сервис | Concurrency limit, rate per second |

---

## HttpClient — правильное использование

### Проблема: `new HttpClient()`

```csharp
// ✗ НИКОГДА — утечка сокетов (socket exhaustion)
public class BadService
{
    public async Task<string> GetDataAsync()
    {
        using var client = new HttpClient(); // каждый вызов = новый сокет!
        return await client.GetStringAsync("https://api.example.com/data");
    }
    // Dispose не сразу закрывает соединение → TIME_WAIT 240 сек
    // 1000 вызовов = 1000 сокетов в TIME_WAIT → SocketException
}
```

### Typed HttpClient — правильный подход

```csharp
// 1. Интерфейс (Domain/Application layer)
public interface IPaymentGateway
{
    Task<Result<PaymentResponse>> ChargeAsync(ChargeRequest request, CancellationToken ct);
}

// 2. Реализация (Infrastructure layer)
public sealed class StripePaymentGateway(
    HttpClient httpClient,
    ILogger<StripePaymentGateway> logger) : IPaymentGateway
{
    public async Task<Result<PaymentResponse>> ChargeAsync(
        ChargeRequest request, CancellationToken ct)
    {
        try
        {
            var response = await httpClient.PostAsJsonAsync("/v1/charges", request, ct);

            if (!response.IsSuccessStatusCode)
            {
                var problem = await response.Content.ReadFromJsonAsync<ProblemDetails>(ct);
                return Result<PaymentResponse>.Fail(
                    Error.Internal("Payment.Failed", problem?.Detail ?? "Payment failed"));
            }

            var result = await response.Content.ReadFromJsonAsync<PaymentResponse>(ct);
            return Result<PaymentResponse>.Ok(result!);
        }
        catch (TaskCanceledException) when (!ct.IsCancellationRequested)
        {
            // Timeout (не user cancellation)
            logger.LogWarning("Payment gateway timeout for {Amount}", request.Amount);
            return Result<PaymentResponse>.Fail(
                Error.Internal("Payment.Timeout", "Payment gateway timed out"));
        }
    }
}

// 3. Регистрация с Resilience
builder.Services
    .AddHttpClient<IPaymentGateway, StripePaymentGateway>(client =>
    {
        client.BaseAddress = new Uri("https://api.stripe.com");
        client.DefaultRequestHeaders.Add("Authorization", $"Bearer {apiKey}");
        client.Timeout = TimeSpan.FromSeconds(30); // общий timeout
    })
    .AddResilienceHandler("stripe", pipeline =>
    {
        // Retry: 3 попытки с exponential backoff
        pipeline.AddRetry(new HttpRetryStrategyOptions
        {
            MaxRetryAttempts = 3,
            Delay = TimeSpan.FromSeconds(1),
            BackoffType = DelayBackoffType.Exponential, // 1s → 2s → 4s
            ShouldHandle = new PredicateBuilder<HttpResponseMessage>()
                .HandleResult(r => r.StatusCode == HttpStatusCode.TooManyRequests
                                || r.StatusCode >= HttpStatusCode.InternalServerError)
                .Handle<HttpRequestException>()
                .Handle<TimeoutRejectedException>()
        });

        // Circuit Breaker: 5 ошибок за 30 сек → открыть на 15 сек
        pipeline.AddCircuitBreaker(new HttpCircuitBreakerStrategyOptions
        {
            FailureRatio = 0.5,          // 50% ошибок
            SamplingDuration = TimeSpan.FromSeconds(30),
            MinimumThroughput = 5,        // минимум 5 запросов для оценки
            BreakDuration = TimeSpan.FromSeconds(15)
        });

        // Timeout per-attempt (внутри retry)
        pipeline.AddTimeout(TimeSpan.FromSeconds(10));
    });
```

**Нюанс:** `IHttpClientFactory` автоматически:
- Пулит `HttpMessageHandler` (переиспользует соединения)
- Обновляет DNS каждые 2 минуты (не кеширует навсегда)
- Интегрируется с DI lifecycle

---

## Polly через Microsoft.Extensions.Resilience

### Установка

```bash
dotnet add package Microsoft.Extensions.Http.Resilience
```

### Стандартный Resilience Pipeline

```csharp
// Готовый набор: Rate Limiter → Timeout → Retry → Circuit Breaker → Timeout per attempt
builder.Services
    .AddHttpClient<IWeatherService, WeatherService>(client =>
    {
        client.BaseAddress = new Uri("https://api.weather.com");
    })
    .AddStandardResilienceHandler(); // включает всё из коробки
```

### Кастомизация стандартного pipeline

```csharp
builder.Services
    .AddHttpClient<IWeatherService, WeatherService>()
    .AddStandardResilienceHandler(options =>
    {
        // Переопределить retry
        options.Retry.MaxRetryAttempts = 5;
        options.Retry.Delay = TimeSpan.FromMilliseconds(500);

        // Переопределить circuit breaker
        options.CircuitBreaker.BreakDuration = TimeSpan.FromSeconds(30);

        // Переопределить timeout
        options.AttemptTimeout.Timeout = TimeSpan.FromSeconds(5);
        options.TotalRequestTimeout.Timeout = TimeSpan.FromSeconds(30);
    });
```

### Порядок стратегий

```
Запрос → Rate Limiter → Total Timeout → Retry ←→ Circuit Breaker → Attempt Timeout → HttpClient
                                           ↑                                              |
                                           └──────── retry при ошибке ─────────────────────┘
```

**Нюанс:** Порядок стратегий критичен:
- **Total Timeout** снаружи Retry — ограничивает ОБЩЕЕ время (все попытки)
- **Attempt Timeout** внутри Retry — ограничивает ОДНУ попытку
- **Circuit Breaker** внутри Retry — если circuit open, retry не тратит попытку

---

## Resilience для не-HTTP сценариев

```csharp
// Resilience pipeline для любой операции (не только HTTP)
builder.Services.AddResiliencePipeline("database", pipeline =>
{
    pipeline.AddRetry(new RetryStrategyOptions
    {
        MaxRetryAttempts = 3,
        Delay = TimeSpan.FromMilliseconds(200),
        BackoffType = DelayBackoffType.Exponential,
        ShouldHandle = new PredicateBuilder()
            .Handle<DbUpdateConcurrencyException>()
            .Handle<TimeoutException>()
    });
});

// Использование
public sealed class OrderRepository(
    AppDbContext context,
    [FromKeyedServices("database")] ResiliencePipeline pipeline)
{
    public async Task SaveWithRetryAsync(Order order, CancellationToken ct)
    {
        await pipeline.ExecuteAsync(async token =>
        {
            context.Orders.Add(order);
            await context.SaveChangesAsync(token);
        }, ct);
    }
}
```

---

## Retry с Idempotency Key

Retry безопасен только для идемпотентных операций. POST без idempotency key может создать дубликат.

```csharp
// Добавление Idempotency Key через DelegatingHandler
public sealed class IdempotencyKeyHandler : DelegatingHandler
{
    protected override Task<HttpResponseMessage> SendAsync(
        HttpRequestMessage request, CancellationToken ct)
    {
        if (request.Method == HttpMethod.Post || request.Method == HttpMethod.Put)
        {
            request.Headers.TryAddWithoutValidation(
                "Idempotency-Key", Guid.NewGuid().ToString());
        }
        return base.SendAsync(request, ct);
    }
}

// Регистрация
builder.Services.AddTransient<IdempotencyKeyHandler>();
builder.Services
    .AddHttpClient<IPaymentGateway, StripePaymentGateway>()
    .AddHttpMessageHandler<IdempotencyKeyHandler>() // перед resilience!
    .AddStandardResilienceHandler();
```

---

## Логирование retry

```csharp
pipeline.AddRetry(new HttpRetryStrategyOptions
{
    MaxRetryAttempts = 3,
    Delay = TimeSpan.FromSeconds(1),
    BackoffType = DelayBackoffType.Exponential,
    OnRetry = args =>
    {
        logger.LogWarning(
            "Retry {Attempt} after {Delay}ms. Outcome: {Outcome}",
            args.AttemptNumber,
            args.RetryDelay.TotalMilliseconds,
            args.Outcome.Result?.StatusCode ?? default);
        return ValueTask.CompletedTask;
    }
});
```

---

## См. также

- [Hosting и Background](hosting-background.md) — Health Checks
- [Messaging](../Infrastructure/messaging.md) — Resilience в message broker
- [Async и Threading](../CSharp/async-threading.md) — CancellationToken, timeout
