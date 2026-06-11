---
tags: [aspnetcore, rate-limiting, throttling, middle, middleware]
level: Middle
date: 2026-05-10
---

# ASP.NET Core Rate Limiting (.NET 7+) — defense, throttling, fairness

> **Built-in rate limiting middleware (.NET 7+), 4 algorithms (fixed, sliding, token bucket, concurrency), per-user/per-IP policies, queue rejected requests.** Production-grade DDoS защита и API fairness.

---

## 0. Как читать

Rate limiting — критическая часть production API. Без него: DDoS атаки, abusive clients, exhausted DB connections, downstream service collapse. .NET 7+ имеет встроенное решение — раньше нужны были третьи-party libraries (AspNetCoreRateLimit).

---

## 1. Зачем rate limiting

### 1.1. Проблема

```
Без rate limiting:
- Bot отправляет 10K req/sec → ваш API падает
- Один клиент monopolizes resources → others страдают
- Ошибка в client side (loop bug) → DoS себе
- Brute force на login endpoint → password cracking
- Cost explosion если pay-per-call (Azure / AWS)

С rate limiting:
- Лимит per IP / per user / per API key
- Honest clients не affected
- Bursts handled gracefully
- Cost predictable
```

### 1.2. Layers protection

```
1. Network/Edge (CDN / WAF):
   - Cloudflare DDoS protection
   - Per-IP global rate limit
   
2. API Gateway (Azure APIM / AWS API Gateway):
   - Per subscription / per tier
   
3. Application (ASP.NET Core):
   - Per user / per endpoint
   - Business logic limits
   
4. Database / Service:
   - Connection pool limits
   - Query timeout
```

ASP.NET Core rate limiting — для **application layer**.

---

## 2. .NET 7+ Built-in Rate Limiting

### 2.1. Setup

```csharp
using System.Threading.RateLimiting;

var builder = WebApplication.CreateBuilder(args);

builder.Services.AddRateLimiter(options =>
{
    options.GlobalLimiter = PartitionedRateLimiter.Create<HttpContext, string>(context =>
        RateLimitPartition.GetFixedWindowLimiter(
            partitionKey: context.User.Identity?.Name ?? context.Connection.RemoteIpAddress?.ToString() ?? "anonymous",
            factory: _ => new FixedWindowRateLimiterOptions
            {
                AutoReplenishment = true,
                PermitLimit = 100,
                Window = TimeSpan.FromMinutes(1)
            }));
});

var app = builder.Build();

app.UseRateLimiter();   // before endpoints

app.MapGet("/api/users", () => /* ... */);

app.Run();
```

100 requests per minute **per user/IP**.

### 2.2. Apply to endpoints

```csharp
// Apply globally
app.UseRateLimiter();

// Per endpoint
app.MapGet("/api/users", () => /* ... */)
   .RequireRateLimiting("UsersPolicy");

// Skip rate limiting
app.MapGet("/health", () => /* ... */)
   .DisableRateLimiting();
```

### 2.3. Controllers

```csharp
[EnableRateLimiting("ApiPolicy")]
public class UsersController : ControllerBase
{
    [HttpGet]
    [DisableRateLimiting]   // skip for one action
    public IActionResult Health() { }
    
    [HttpPost("login")]
    [EnableRateLimiting("LoginPolicy")]   // strict для login
    public IActionResult Login() { }
}
```

---

## 3. Алгоритмы — 4 варианта

### 3.1. Fixed Window

```
Определи окно (e.g. 1 minute) и лимит (100 requests).
Reset счётчика в начале каждого окна.

|---window 1---|---window 2---|---window 3---|
   100 reqs OK   100 reqs OK   100 reqs OK

Проблема: burst на границе окна.
В 12:00:59 100 requests + 12:01:01 ещё 100 = 200 за 2 секунды.
```

```csharp
options.AddFixedWindowLimiter("Fixed", opt =>
{
    opt.PermitLimit = 100;
    opt.Window = TimeSpan.FromMinutes(1);
    opt.QueueLimit = 0;   // reject excess
});
```

**Use case**: simple, predictable, дешёвый. Не критично для precise enforcement.

### 3.2. Sliding Window

```
Окно "скользит" непрерывно.
Лимит = N requests в any 60-second window.

|---last 60 sec---|
        ↑
    Текущий момент: 87 requests за last 60s, OK
    
Через 1s: |---last 60 sec---|
                              ↑
                          88 requests, OK
```

Решает проблему burst на границе fixed window.

```csharp
options.AddSlidingWindowLimiter("Sliding", opt =>
{
    opt.PermitLimit = 100;
    opt.Window = TimeSpan.FromMinutes(1);
    opt.SegmentsPerWindow = 6;   // window разделён на 6 segments по 10s
    opt.QueueLimit = 0;
});
```

**Use case**: production APIs где precise enforcement нужно. Чуть больше memory чем fixed.

### 3.3. Token Bucket

```
Bucket держит N tokens. Каждый request забирает 1 token.
Каждый период (e.g. 1s) добавляется M tokens (replenishment).

Bucket capacity: 100 tokens
Replenishment: 10 tokens / second

→ Average rate: 10 req/s
→ Burst tolerance: до 100 в один момент

Подходит для: bursty traffic, accommodating spikes.
```

```csharp
options.AddTokenBucketLimiter("TokenBucket", opt =>
{
    opt.TokenLimit = 100;
    opt.QueueLimit = 0;
    opt.ReplenishmentPeriod = TimeSpan.FromSeconds(1);
    opt.TokensPerPeriod = 10;
    opt.AutoReplenishment = true;
});
```

**Use case**: API endpoints с occasional spikes (login bursts, batch operations).

### 3.4. Concurrency Limiter

```
Лимит одновременных concurrent requests (не rate, а concurrency).
N = 10 → max 10 in-flight requests.
```

```csharp
options.AddConcurrencyLimiter("Concurrency", opt =>
{
    opt.PermitLimit = 10;
    opt.QueueLimit = 5;
    opt.QueueProcessingOrder = QueueProcessingOrder.OldestFirst;
});
```

**Use case**: heavy operations (file processing, AI inference) где limit by capacity, not rate.

### 3.5. Decision tree

```
Что выбрать?
│
├── Simple, predictable rate → Fixed Window
├── Strict per-window enforcement → Sliding Window
├── Bursty traffic (login spikes) → Token Bucket
└── Limit by capacity (heavy ops) → Concurrency Limiter
```

> [!question]- **Интервью: какие rate limiting алгоритмы?**
> 4 встроенных в .NET 7+: 1) **Fixed Window** — N requests per fixed period (1 min). Simple, но burst на границе. 2) **Sliding Window** — rolling N requests за last period. Решает burst issue. 3) **Token Bucket** — bucket с tokens, replenishment rate. Tolerates bursts. 4) **Concurrency Limiter** — N concurrent requests max (не rate). **Use cases**: Fixed для simple, Sliding для precise, Token Bucket для bursty, Concurrency для heavy ops. **Production**: combine — Sliding для общего API + Concurrency для specific endpoints.

---

## 4. Partitioning — per user / per IP / per tenant

### 4.1. Per IP

```csharp
options.GlobalLimiter = PartitionedRateLimiter.Create<HttpContext, string>(context =>
    RateLimitPartition.GetFixedWindowLimiter(
        partitionKey: context.Connection.RemoteIpAddress?.ToString() ?? "unknown",
        factory: _ => new FixedWindowRateLimiterOptions
        {
            PermitLimit = 100,
            Window = TimeSpan.FromMinutes(1)
        }));
```

⚠️ Behind reverse proxy (nginx, Cloudflare) — `RemoteIpAddress` будет proxy IP. Use `X-Forwarded-For`:

```csharp
builder.Services.Configure<ForwardedHeadersOptions>(options =>
{
    options.ForwardedHeaders = ForwardedHeaders.XForwardedFor;
    options.KnownNetworks.Clear();
    options.KnownProxies.Clear();
});

app.UseForwardedHeaders();   // ПЕРЕД UseRateLimiter
app.UseRateLimiter();
```

### 4.2. Per authenticated user

```csharp
options.GlobalLimiter = PartitionedRateLimiter.Create<HttpContext, string>(context =>
{
    var user = context.User;
    if (user.Identity?.IsAuthenticated == true)
    {
        var userId = user.FindFirst("sub")?.Value ?? user.Identity.Name!;
        return RateLimitPartition.GetFixedWindowLimiter(
            partitionKey: $"user:{userId}",
            factory: _ => new FixedWindowRateLimiterOptions
            {
                PermitLimit = 1000,    // авторизованные = больше
                Window = TimeSpan.FromMinutes(1)
            });
    }
    
    // Anonymous — limit by IP, более restrictive
    return RateLimitPartition.GetFixedWindowLimiter(
        partitionKey: $"ip:{context.Connection.RemoteIpAddress}",
        factory: _ => new FixedWindowRateLimiterOptions
        {
            PermitLimit = 50,
            Window = TimeSpan.FromMinutes(1)
        });
});
```

### 4.3. Per API key / tier

```csharp
options.GlobalLimiter = PartitionedRateLimiter.Create<HttpContext, string>(context =>
{
    var apiKey = context.Request.Headers["X-API-Key"].FirstOrDefault();
    if (string.IsNullOrEmpty(apiKey))
    {
        return RateLimitPartition.GetNoLimiter("no-key");   // no limiting (но 401 от auth)
    }
    
    var tier = GetTier(apiKey);   // resolve from cache
    
    return tier switch
    {
        "free" => RateLimitPartition.GetFixedWindowLimiter(
            partitionKey: $"key:{apiKey}",
            factory: _ => new FixedWindowRateLimiterOptions
            {
                PermitLimit = 100,
                Window = TimeSpan.FromMinutes(1)
            }),
        "pro" => RateLimitPartition.GetFixedWindowLimiter(
            partitionKey: $"key:{apiKey}",
            factory: _ => new FixedWindowRateLimiterOptions
            {
                PermitLimit = 1000,
                Window = TimeSpan.FromMinutes(1)
            }),
        "enterprise" => RateLimitPartition.GetFixedWindowLimiter(
            partitionKey: $"key:{apiKey}",
            factory: _ => new FixedWindowRateLimiterOptions
            {
                PermitLimit = 10000,
                Window = TimeSpan.FromMinutes(1)
            }),
        _ => RateLimitPartition.GetNoLimiter("unknown")
    };
});
```

### 4.4. Multiple policies — combinable

```csharp
builder.Services.AddRateLimiter(options =>
{
    // Global fallback
    options.GlobalLimiter = ...;
    
    // Named policies for specific endpoints
    options.AddPolicy("LoginPolicy", context =>
        RateLimitPartition.GetFixedWindowLimiter(
            partitionKey: context.Connection.RemoteIpAddress?.ToString() ?? "unknown",
            factory: _ => new FixedWindowRateLimiterOptions
            {
                PermitLimit = 5,    // strict для login
                Window = TimeSpan.FromMinutes(1),
                QueueLimit = 0
            }));
    
    options.AddPolicy("UploadPolicy", context =>
        RateLimitPartition.GetTokenBucketLimiter(
            partitionKey: context.User.Identity?.Name ?? "anon",
            factory: _ => new TokenBucketRateLimiterOptions
            {
                TokenLimit = 10,
                ReplenishmentPeriod = TimeSpan.FromMinutes(1),
                TokensPerPeriod = 5,
                AutoReplenishment = true
            }));
    
    options.AddPolicy("HeavyComputePolicy", context =>
        RateLimitPartition.GetConcurrencyLimiter(
            partitionKey: "global",
            factory: _ => new ConcurrencyLimiterOptions
            {
                PermitLimit = 5,
                QueueLimit = 10
            }));
});

// Apply
app.MapPost("/login", ...).RequireRateLimiting("LoginPolicy");
app.MapPost("/upload", ...).RequireRateLimiting("UploadPolicy");
app.MapPost("/ai-inference", ...).RequireRateLimiting("HeavyComputePolicy");
```

---

## 5. Rejected requests — что отвечать

### 5.1. Default — 503 Service Unavailable

Без custom configuration — 503 без body.

### 5.2. Customize rejection response

```csharp
builder.Services.AddRateLimiter(options =>
{
    options.RejectionStatusCode = StatusCodes.Status429TooManyRequests;
    
    options.OnRejected = async (context, ct) =>
    {
        context.HttpContext.Response.StatusCode = 429;
        
        // Retry-After header
        if (context.Lease.TryGetMetadata(MetadataName.RetryAfter, out var retryAfter))
        {
            context.HttpContext.Response.Headers.RetryAfter = 
                ((int)retryAfter.TotalSeconds).ToString();
        }
        
        await context.HttpContext.Response.WriteAsJsonAsync(new ProblemDetails
        {
            Status = 429,
            Title = "Too Many Requests",
            Detail = "You have exceeded the rate limit. Try again later.",
            Extensions = 
            {
                ["retryAfterSeconds"] = ((int?)retryAfter.TotalSeconds) ?? 60
            }
        }, ct);
    };
});
```

### 5.3. Logging rejections

```csharp
options.OnRejected = async (context, ct) =>
{
    var logger = context.HttpContext.RequestServices
        .GetRequiredService<ILogger<Program>>();
    
    logger.LogWarning("Rate limit exceeded for {Path} from {IP}, user {User}",
        context.HttpContext.Request.Path,
        context.HttpContext.Connection.RemoteIpAddress,
        context.HttpContext.User.Identity?.Name ?? "anonymous");
    
    // ... response
};
```

### 5.4. Queue requests instead of reject

```csharp
options.AddFixedWindowLimiter("Queued", opt =>
{
    opt.PermitLimit = 10;
    opt.Window = TimeSpan.FromSeconds(1);
    opt.QueueLimit = 50;   // up to 50 в очереди
    opt.QueueProcessingOrder = QueueProcessingOrder.OldestFirst;
});
```

Когда лимит достигнут — следующий request waits в очереди до освобождения slot. `QueueLimit` exceeded → 503/429.

```
✅ Queue use cases:
- API smoothing — accept bursts gracefully
- Batch processing — orderly execution

❌ Не queue для:
- Login attempts (хотим fail fast)
- User-facing fast responses (timeout risks)
```

---

## 6. Production examples

### 6.1. Login endpoint — strict

```csharp
options.AddPolicy("LoginPolicy", context =>
    RateLimitPartition.GetFixedWindowLimiter(
        partitionKey: context.Connection.RemoteIpAddress?.ToString() ?? "unknown",
        factory: _ => new FixedWindowRateLimiterOptions
        {
            PermitLimit = 5,                    // 5 попыток
            Window = TimeSpan.FromMinutes(15),  // в 15 минут
            QueueLimit = 0
        }));

app.MapPost("/auth/login", LoginHandler).RequireRateLimiting("LoginPolicy");
```

Brute force protection на login.

### 6.2. Public API — generous

```csharp
options.AddPolicy("PublicApi", context =>
    RateLimitPartition.GetSlidingWindowLimiter(
        partitionKey: context.Connection.RemoteIpAddress?.ToString() ?? "unknown",
        factory: _ => new SlidingWindowRateLimiterOptions
        {
            PermitLimit = 1000,
            Window = TimeSpan.FromMinutes(1),
            SegmentsPerWindow = 10
        }));

app.MapGet("/api/products", ...).RequireRateLimiting("PublicApi");
```

### 6.3. Premium API — based on tier

```csharp
options.AddPolicy("TierBased", context =>
{
    var tier = context.User.FindFirst("tier")?.Value ?? "free";
    var userId = context.User.Identity?.Name ?? "anonymous";
    
    return tier switch
    {
        "free" => RateLimitPartition.GetFixedWindowLimiter(
            $"free:{userId}",
            _ => new FixedWindowRateLimiterOptions { PermitLimit = 60, Window = TimeSpan.FromMinutes(1) }),
        "pro" => RateLimitPartition.GetFixedWindowLimiter(
            $"pro:{userId}",
            _ => new FixedWindowRateLimiterOptions { PermitLimit = 1000, Window = TimeSpan.FromMinutes(1) }),
        "enterprise" => RateLimitPartition.GetNoLimiter("enterprise"),
        _ => RateLimitPartition.GetFixedWindowLimiter(
            "anon",
            _ => new FixedWindowRateLimiterOptions { PermitLimit = 30, Window = TimeSpan.FromMinutes(1) })
    };
});
```

### 6.4. Heavy operation — concurrency

```csharp
options.AddPolicy("AIInference", context =>
    RateLimitPartition.GetConcurrencyLimiter(
        partitionKey: "global",
        factory: _ => new ConcurrencyLimiterOptions
        {
            PermitLimit = 4,    // только 4 inference одновременно
            QueueLimit = 20,
            QueueProcessingOrder = QueueProcessingOrder.OldestFirst
        }));

app.MapPost("/ai/inference", AIHandler).RequireRateLimiting("AIInference");
```

---

## 7. Distributed rate limiting

### 7.1. Проблема single-instance

Built-in rate limiting — **per-process**. Если 4 instances приложения:
- Лимит 100/min на каждой → real лимит 400/min
- User может switch между instances → effectively unlimited

### 7.2. Distributed solution

Для multi-pod / kubernetes используй:

**Option 1: Redis-based**
- `RedisRateLimiting` (community NuGet)
- AspNetCoreRateLimit с Redis store

```bash
dotnet add package RedisRateLimiting.AspNetCore
```

```csharp
builder.Services.AddRateLimiter(options =>
{
    options.AddRedisFixedWindowLimiter("Distributed", opt =>
    {
        opt.PermitLimit = 100;
        opt.Window = TimeSpan.FromMinutes(1);
        opt.ConnectionMultiplexerFactory = () => 
            ConnectionMultiplexer.Connect("redis-host:6379");
    });
});
```

**Option 2: API Gateway**

Использовать Azure APIM / AWS API Gateway — они managed distributed rate limiting в edge.

**Option 3: Sticky sessions**

Load balancer routes пользователя всегда на одну instance (по cookie / IP). Per-process limit becomes effectively per-user.

### 7.3. Trade-off

```
Built-in (in-memory):
✅ Simple
✅ Fast
❌ Per-instance — multiplied with scaling

Redis-distributed:
✅ Accurate across instances
❌ Network roundtrip (~ms latency)
❌ Redis dependency
❌ Cost

API Gateway:
✅ No app changes
✅ Edge protection
❌ Vendor lock-in
❌ Cost
```

> [!question]- **Интервью: rate limiting в multi-pod scenario?**
> Built-in `AddRateLimiter` — **per-process**. С N подов лимит multiplies на N. **Solutions**: 1) **Redis-distributed** (RedisRateLimiting library) — shared store, accurate но network latency. 2) **API Gateway** (Azure APIM, AWS API Gateway, Kong) — edge protection, managed. 3) **Sticky sessions** в load balancer — user always одна instance, per-process limit становится per-user. **Best practice**: combine — API Gateway для global, application для business logic, sticky sessions для simple cases.

---

## 8. Common pitfalls

### 8.1. UseRateLimiter после endpoints

```csharp
// ❌ Order matters
app.MapGet("/api/users", ...);
app.UseRateLimiter();   // не работает!

// ✅
app.UseRateLimiter();
app.MapGet("/api/users", ...);
```

### 8.2. RemoteIpAddress без X-Forwarded-For

```csharp
// За nginx / Cloudflare — все clients имеют tot же proxy IP
partitionKey: context.Connection.RemoteIpAddress?.ToString();
// Все users считаются как один!
```

**Fix**: `UseForwardedHeaders` middleware ПЕРЕД UseRateLimiter.

### 8.3. Forgot DisableRateLimiting на health checks

```csharp
app.MapGet("/health", ...);   // ❌ rate-limited!
// Kubernetes health checks могут rejected → false unhealthy
```

**Fix**: `.DisableRateLimiting()` или separate endpoint без middleware.

### 8.4. Per-user без auth

```csharp
partitionKey: context.User.Identity?.Name   // ❌ anonymous = same partition!
```

Все anonymous users шлют в один bucket → low limit.

**Fix**: fall-back to IP если не authenticated.

### 8.5. Memory exhaustion

```csharp
// Per-IP — если 1M unique IPs → 1M partitions in memory
options.GlobalLimiter = PartitionedRateLimiter.Create<HttpContext, string>(...);
```

Each partition имеет state → memory grows. Для public APIs нужен mechanism cleanup или distributed cache.

**Fix**: 
- Shorter windows (less state to keep)
- Redis-distributed
- Eviction policy

### 8.6. Concurrency limiter без queue

```csharp
options.AddConcurrencyLimiter("Strict", opt =>
{
    opt.PermitLimit = 1;
    opt.QueueLimit = 0;   // ❌ Любой 2й request → reject
});
```

Реальный traffic имеет concurrent requests. Use queue или higher PermitLimit.

### 8.7. Login policy слишком strict

```csharp
options.AddPolicy("Login", _ => /* 1 attempt per minute */);
// Real users часто mistype password — lockout legitimate users
```

**Fix**: balance — 5-10 попыток per 15 минут typically.

### 8.8. Не Retry-After header

```csharp
// Без Retry-After client не знает когда retry
// Smart clients используют exponential backoff vs random retry
```

**Fix**: include `Retry-After` header в rejection.

### 8.9. OnRejected не handling exceptions

```csharp
options.OnRejected = async (context, ct) =>
{
    var data = await SomeAsyncCall();   // ❌ если throws — server crash
    await context.HttpContext.Response.WriteAsJsonAsync(...);
};
```

**Fix**: try/catch в OnRejected.

### 8.10. Production без monitoring

Without metrics:
- Не видишь сколько rejected
- Не видишь distribution rejections
- Не tunes лимиты

**Fix**: emit metrics, alert на high rejection rate.

```csharp
options.OnRejected = async (context, ct) =>
{
    Metrics.Counter("rate_limit_rejections", 
        new[] { ("policy", context.Lease.PolicyName ?? "unknown") }).Increment();
    // ... response
};
```

> [!question]- **Интервью: топ-3 ошибки rate limiting?**
> 1) **RemoteIpAddress без ForwardedHeaders** — за reverse proxy все clients один IP. Fix: UseForwardedHeaders ПЕРЕД UseRateLimiter. 2) **Per-process в multi-pod** — 4 pods × 100/min = 400/min real limit. Fix: Redis-distributed или API Gateway. 3) **Login policy слишком strict** — locks out legitimate users который mistype password. Fix: balance, 5-10 attempts per 15 min. **Bonus**: forgot Retry-After header → bad client UX.

---

## 9. Cheat sheet

```csharp
// === Setup ===
builder.Services.AddRateLimiter(options =>
{
    // Global limiter
    options.GlobalLimiter = PartitionedRateLimiter.Create<HttpContext, string>(context =>
        RateLimitPartition.GetFixedWindowLimiter(
            partitionKey: context.Connection.RemoteIpAddress?.ToString() ?? "anon",
            factory: _ => new FixedWindowRateLimiterOptions
            {
                PermitLimit = 100,
                Window = TimeSpan.FromMinutes(1)
            }));
    
    // Named policies
    options.AddFixedWindowLimiter("Login", opt =>
    {
        opt.PermitLimit = 5;
        opt.Window = TimeSpan.FromMinutes(15);
    });
    
    options.AddSlidingWindowLimiter("Api", opt =>
    {
        opt.PermitLimit = 1000;
        opt.Window = TimeSpan.FromMinutes(1);
        opt.SegmentsPerWindow = 10;
    });
    
    options.AddTokenBucketLimiter("Bursty", opt =>
    {
        opt.TokenLimit = 100;
        opt.ReplenishmentPeriod = TimeSpan.FromSeconds(1);
        opt.TokensPerPeriod = 10;
        opt.AutoReplenishment = true;
    });
    
    options.AddConcurrencyLimiter("Heavy", opt =>
    {
        opt.PermitLimit = 4;
        opt.QueueLimit = 20;
    });
    
    // Rejection response
    options.RejectionStatusCode = 429;
    options.OnRejected = async (context, ct) =>
    {
        if (context.Lease.TryGetMetadata(MetadataName.RetryAfter, out var retryAfter))
        {
            context.HttpContext.Response.Headers.RetryAfter = 
                ((int)retryAfter.TotalSeconds).ToString();
        }
        
        await context.HttpContext.Response.WriteAsJsonAsync(new
        {
            error = "TooManyRequests",
            message = "Rate limit exceeded"
        }, ct);
    };
});

// Middleware order (CRITICAL!)
app.UseForwardedHeaders();   // for real IP behind proxy
app.UseRateLimiter();
app.UseAuthentication();
app.UseAuthorization();
// ... endpoints

// === Apply ===
app.MapPost("/login", ...).RequireRateLimiting("Login");
app.MapGet("/api/data", ...).RequireRateLimiting("Api");
app.MapGet("/health", ...).DisableRateLimiting();

// Controller-level
[EnableRateLimiting("Api")]
public class UsersController : ControllerBase { }

// === Per-tier ===
options.AddPolicy("ByTier", context =>
{
    var tier = context.User.FindFirst("tier")?.Value ?? "free";
    return tier switch
    {
        "free" => RateLimitPartition.GetFixedWindowLimiter(
            $"free:{context.User.Identity?.Name}",
            _ => new FixedWindowRateLimiterOptions { PermitLimit = 60, Window = TimeSpan.FromMinutes(1) }),
        "pro" => RateLimitPartition.GetFixedWindowLimiter(
            $"pro:{context.User.Identity?.Name}",
            _ => new FixedWindowRateLimiterOptions { PermitLimit = 1000, Window = TimeSpan.FromMinutes(1) }),
        _ => RateLimitPartition.GetNoLimiter("unlimited")
    };
});
```

---

## 10. Practice exercises

### 10.1. Multi-tier API

Реализуй rate limiting:
- Anonymous: 30 req/min per IP
- Free user: 100 req/min
- Pro user: 1000 req/min  
- Enterprise: unlimited
- Login: strict 5 attempts / 15 minutes per IP

С proper Retry-After headers и ProblemDetails responses.

### 10.2. Per-endpoint policies

API имеет:
- `GET /products` — public, generous (1000/min)
- `POST /orders` — authenticated, moderate (100/min)
- `POST /uploads` — authenticated, low + concurrent limit (10 simultaneous)
- `POST /ai-generate` — authenticated, low + queue (5/min, queue 20)

Реализуй каждый со своей policy.

### 10.3. Distributed rate limiting

Текущий setup — in-memory. Migrate на Redis-distributed:
- Add RedisRateLimiting package
- Configure connection
- Test fairness между несколькими instance app

---

## 11. Что читать дальше

1. **`AspNetCore/Senior/api-design.md`** — API design
2. **`AspNetCore/Senior/security-practices.md`** — security
3. **`AspNetCore/Senior/resilience.md`** — Polly, circuit breakers
4. **`AspNetCore/Middle/aspnet-error-handling.md`** — error responses
5. **`Performance/Senior/capacity-planning.md`** — capacity context

---

## 12. См. также

- [[api-design|AspNetCore/Senior/api-design]] — design
- [[security-practices|AspNetCore/Senior/security-practices]] — security
- [[resilience|AspNetCore/Senior/resilience]] — circuit breakers / Polly
- [[aspnet-error-handling|AspNetCore/Middle/aspnet-error-handling]] — errors
- [[capacity-planning|Performance/Senior/capacity-planning]] — capacity

---

## 13. Reading list

- **Microsoft Docs — Rate Limiting** — learn.microsoft.com/aspnet/core/performance/rate-limit
- **Microsoft Docs — System.Threading.RateLimiting** — learn.microsoft.com/dotnet/api/system.threading.ratelimiting
- **RedisRateLimiting** — github.com/cristipufu/aspnetcore-redis-rate-limiting
- **AspNetCoreRateLimit** — github.com/stefanprodan/AspNetCoreRateLimit (legacy)
- **Andrew Lock — Rate Limiting articles** — andrewlock.net
- **OWASP — API Rate Limiting** — owasp.org/www-project-api-security
