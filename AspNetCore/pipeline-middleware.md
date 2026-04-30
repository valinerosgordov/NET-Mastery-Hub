---
tags: [aspnet, pipeline, middleware, routing, exception-handler, rate-limiting, output-caching, problem-details]
level: Senior
date: 2026-04-30
---

# Pipeline, Middleware и Routing

> Полный гайд по ASP.NET Core pipeline. Закрывает: middleware patterns deep, IExceptionHandler (.NET 8+), Output Caching (.NET 7+), Rate Limiting (.NET 7+), ProblemDetails RFC 7807, request decompression, response.OnStarting/OnCompleted, WebSocket middleware, branching (Map/MapWhen/UseWhen/UsePathBase), TestServer для тестирования.

---

## Что это, зачем и когда

### Что такое Middleware?
Middleware — «слой» через который проходит **каждый** HTTP-запрос. Цепочка middleware — конвейер: запрос проходит через слои один за другим, потом ответ возвращается обратно.

**Аналогия — аэропорт:**
1. Проверка билета (Authentication) — кто ты?
2. Проверка визы (Authorization) — имеешь ли право?
3. Досмотр багажа (Validation) — всё ли в порядке с запросом?
4. Посадка (Endpoint) — обработка запроса
5. Выход (Response) — ответ проходит обратно через те же слои

### Зачем нужно?
- **Разделение ответственности** — логирование, аутентификация, rate limiting не в каждом endpoint, а В ОДНОМ МЕСТЕ
- **Порядок** — сначала проверяем «кто ты», потом «имеешь ли право», потом обрабатываем запрос
- **Short-circuit** — если пользователь не авторизован — middleware ОСТАНАВЛИВАЕТ запрос, endpoint даже не вызывается

### Когда пишешь свой middleware?
- Логирование всех запросов/ответов
- Глобальная обработка ошибок
- Замер времени выполнения запроса
- Добавление correlation ID к каждому запросу

---

> [!question]- **Интервью: Как устроен request pipeline? Порядок middleware?**
> Цепочка middleware: каждый решает — вызвать `next()` или short-circuit. Порядок критичен: Exception Handler первым, CORS до Auth, Auth до Endpoint. Middleware — глобальная логика (логи, ошибки). Filters — логика привязанная к MVC action.

> [!question]- **Интервью: Middleware vs Filters — когда что?**
> **Middleware** — уровень всего pipeline, не знает о MVC. Для: логирования, ошибок, CORS, compression.
> **Filters** — привязаны к MVC action. Для: валидации, авторизации на уровне action, форматирования ответа.

## Pipeline и Middleware

Pipeline (конвейер) — цепочка компонентов (middleware), через которую проходит каждый HTTP-запрос. Каждый middleware получает `HttpContext` и решает: вызвать `next()` для передачи дальше или завершить обработку (short-circuit). Запрос проходит «вниз» до endpoint, ответ — «вверх» обратно через ту же цепочку.

```
Request → M1 → M2 → M3 → Endpoint → M3 → M2 → M1 → Response
```

### Порядок middleware

Порядок регистрации в `Program.cs` **критически важен** — это порядок выполнения. Типичный порядок:

```csharp
// Recommended order (.NET 9+)
app.UseExceptionHandler();              // 1. Exception handler — first
app.UseHsts();                          // 2. HSTS (production)
app.UseHttpsRedirection();              // 3. HTTPS redirect
app.UseStaticFiles();                   // 4. Static files (без auth)
app.UseRequestDecompression();          // 5. Request decompression (.NET 7+)
app.UseResponseCompression();           // 6. Response compression
app.UseRouting();                       // 7. Routing (выбор endpoint)
app.UseRateLimiter();                   // 8. Rate limiting (.NET 7+)
app.UseRequestLocalization();           // 9. Localization
app.UseCors();                          // 10. CORS (после Routing!)
app.UseAuthentication();                // 11. Auth
app.UseAuthorization();                 // 12. Authz
app.UseSession();                       // 13. Session
app.UseOutputCache();                   // 14. Output caching (.NET 7+)
app.MapEndpoints();                     // 15. Endpoint dispatch
```

**Почему такой порядок:**
- `UseAuthentication()` **до** `UseAuthorization()` — иначе авторизация не увидит identity пользователя
- `UseExceptionHandler()` **первым** — чтобы перехватывать ошибки из всех последующих middleware
- `UseStaticFiles()` **до** Routing — статические файлы отдаются без auth
- `UseCors()` **после** Routing (но до Auth) — CORS-заголовки должны добавляться даже к preflight-запросам
- `UseRateLimiter()` **после** Routing — чтобы знать какой endpoint matched
- `UseOutputCache()` **после** Auth — чтобы cache key учитывал user

### Short-circuit

Short-circuit — прерывание цепочки, когда middleware не вызывает `next()`. Запрос не доходит до endpoint, ответ возвращается сразу.

Примеры:
- Health check endpoint — мгновенный ответ `200 OK`
- Статические файлы — отдача из файловой системы
- Rate limiting — ответ `429 Too Many Requests`
- Кастомная проверка API key — `401 Unauthorized`

### Создание middleware — 4 паттерна

#### 1. Inline (lambda) — для простой логики

```csharp
app.Use(async (context, next) =>
{
    // Логика ДО следующего middleware
    var sw = Stopwatch.StartNew();

    await next(context); // передаём дальше

    // Логика ПОСЛЕ (на обратном пути)
    sw.Stop();
    context.Response.Headers["X-Elapsed-Ms"] = sw.ElapsedMilliseconds.ToString();
});
```

#### 2. Convention-based class — Singleton lifetime по умолчанию

```csharp
public class RequestTimingMiddleware
{
    private readonly RequestDelegate _next;
    private readonly ILogger<RequestTimingMiddleware> _logger;

    // Конструктор — вызывается ОДИН раз (Singleton lifetime)
    public RequestTimingMiddleware(RequestDelegate next, ILogger<RequestTimingMiddleware> logger)
    {
        _next = next;
        _logger = logger;
    }

    // InvokeAsync — вызывается на КАЖДЫЙ запрос
    // Scoped-зависимости можно получать через параметры метода
    public async Task InvokeAsync(HttpContext context, IMyService myService)
    {
        var sw = Stopwatch.StartNew();
        await _next(context);
        sw.Stop();
        _logger.LogInformation("Request {Path} took {Ms}ms", 
            context.Request.Path, sw.ElapsedMilliseconds);
    }
}

// Регистрация
app.UseMiddleware<RequestTimingMiddleware>();
```

#### 3. IMiddleware — Scoped/Transient lifetime

```csharp
public class TimingMiddleware(ILogger<TimingMiddleware> logger) : IMiddleware
{
    public async Task InvokeAsync(HttpContext context, RequestDelegate next)
    {
        var sw = Stopwatch.StartNew();
        await next(context);
        sw.Stop();
        logger.LogInformation("Request took {Ms}ms", sw.ElapsedMilliseconds);
    }
}

// Регистрация — нужно зарегистрировать в DI
builder.Services.AddScoped<TimingMiddleware>();
app.UseMiddleware<TimingMiddleware>();
```

| Convention-based | IMiddleware |
|------------------|-------------|
| Singleton (instance кэшируется) | Per-request (scoped/transient) |
| Scoped через method params | Scoped через constructor |
| Производительнее (меньше аллокаций) | Чище DI integration |
| Default | Когда нужны Scoped зависимости |

#### 4. Extension method — удобная регистрация

```csharp
public static class MiddlewareExtensions
{
    public static IApplicationBuilder UseRequestTiming(this IApplicationBuilder app)
        => app.UseMiddleware<RequestTimingMiddleware>();
}

// Использование: app.UseRequestTiming();
```

### Тонкости

- Middleware-класс convention-based имеет **Singleton lifetime** — конструктор вызывается один раз. Не храните request-specific данные в полях класса
- Scoped-сервисы инжектируются **через параметры `InvokeAsync`** в convention-based middleware
- `app.Run(...)` — терминальный middleware, всегда выполняет short-circuit (нет `next`)
- `app.Map("/path", ...)` — branch pipeline на отдельный путь (fork)
- `app.UseWhen(predicate, ...)` — условный branch, но запрос возвращается в основную цепочку
- `app.MapWhen(predicate, ...)` — условный branch с полным отделением

---

## IExceptionHandler — современная обработка ошибок (.NET 8+)

Раньше — `app.UseExceptionHandler(...)` с lambda или `IExceptionFilter`. Сейчас — рекомендуется `IExceptionHandler` interface.

### Старый подход (lambda)

```csharp
// Legacy
app.UseExceptionHandler(builder =>
{
    builder.Run(async context =>
    {
        var feature = context.Features.Get<IExceptionHandlerFeature>();
        var exception = feature?.Error;
        
        context.Response.StatusCode = StatusCodes.Status500InternalServerError;
        await context.Response.WriteAsJsonAsync(new { error = exception?.Message });
    });
});
```

### Новый подход (.NET 8+) — IExceptionHandler

```csharp
public class GlobalExceptionHandler(
    ILogger<GlobalExceptionHandler> logger) : IExceptionHandler
{
    public async ValueTask<bool> TryHandleAsync(
        HttpContext context,
        Exception exception,
        CancellationToken ct)
    {
        logger.LogError(exception, "Unhandled exception");
        
        var (status, title) = exception switch
        {
            ValidationException => (StatusCodes.Status400BadRequest, "Validation failed"),
            NotFoundException => (StatusCodes.Status404NotFound, "Resource not found"),
            UnauthorizedException => (StatusCodes.Status401Unauthorized, "Unauthorized"),
            DomainException => (StatusCodes.Status422UnprocessableEntity, "Business rule violation"),
            _ => (StatusCodes.Status500InternalServerError, "Server error")
        };
        
        context.Response.StatusCode = status;
        
        var problemDetails = new ProblemDetails
        {
            Status = status,
            Title = title,
            Detail = exception.Message,
            Type = $"https://httpstatuses.com/{status}",
            Instance = context.Request.Path
        };
        
        // Correlation ID
        problemDetails.Extensions["traceId"] = context.TraceIdentifier;
        
        await context.Response.WriteAsJsonAsync(problemDetails, ct);
        
        return true;  // обработали, не передавать дальше
    }
}

// Несколько handlers — порядок важен
public class ValidationExceptionHandler : IExceptionHandler
{
    public async ValueTask<bool> TryHandleAsync(...)
    {
        if (exception is not ValidationException) 
            return false;  // не наш — пусть следующий handler пробует
        
        // ... обработка validation ...
        return true;
    }
}

// Регистрация — порядок имеет значение
builder.Services.AddExceptionHandler<ValidationExceptionHandler>();  // 1st
builder.Services.AddExceptionHandler<GlobalExceptionHandler>();      // fallback
builder.Services.AddProblemDetails();  // обязательно

var app = builder.Build();
app.UseExceptionHandler();  // подключает все registered handlers
```

### Преимущества IExceptionHandler

- **Composable** — несколько handlers, каждый отвечает за свой тип
- **Testable** — обычный класс, легко unit test
- **DI-friendly** — Scoped/Transient lifetime
- **Type-safe** — pattern matching на exception types

---

## ProblemDetails RFC 7807

Стандарт для error responses в HTTP API.

```json
{
  "type": "https://example.com/probs/out-of-credit",
  "title": "You do not have enough credit.",
  "status": 403,
  "detail": "Your current balance is 30, but that costs 50.",
  "instance": "/account/12345/msgs/abc"
}
```

### AddProblemDetails

```csharp
builder.Services.AddProblemDetails(options =>
{
    // Кастомизация
    options.CustomizeProblemDetails = ctx =>
    {
        ctx.ProblemDetails.Extensions["traceId"] = ctx.HttpContext.TraceIdentifier;
        ctx.ProblemDetails.Extensions["requestId"] = Activity.Current?.Id;
        
        if (ctx.HttpContext.Items.TryGetValue("ErrorCode", out var code))
            ctx.ProblemDetails.Extensions["errorCode"] = code;
    };
});

// Используется автоматически в:
// - ASP.NET Core встроенных обработчиках 4xx/5xx
// - app.UseStatusCodePages()
// - app.UseExceptionHandler()
// - Results.Problem() / TypedResults.Problem()
```

### Возврат ProblemDetails из endpoint

```csharp
app.MapPost("/orders", async (CreateOrderCommand cmd, IMediator mediator) =>
{
    var result = await mediator.Send(cmd);
    
    return result.Match<IResult>(
        success => TypedResults.Created($"/orders/{success.Id}", success),
        notFound => TypedResults.Problem(
            statusCode: StatusCodes.Status404NotFound,
            title: "Customer not found",
            detail: $"Customer {cmd.CustomerId} does not exist"),
        validationError => TypedResults.ValidationProblem(validationError.Errors));
});
```

---

## Output Caching (.NET 7+)

Замена `ResponseCaching`. Кэширует **ответы endpoint'ов**, поддерживает tag-based eviction.

### Setup

```csharp
builder.Services.AddOutputCache(options =>
{
    options.AddBasePolicy(builder => 
        builder.Expire(TimeSpan.FromSeconds(30)));
    
    options.AddPolicy("ProductsPolicy", builder =>
        builder.Expire(TimeSpan.FromMinutes(5))
            .Tag("products")
            .SetVaryByQuery("page", "size"));
    
    options.AddPolicy("AuthCachePolicy", builder =>
        builder.Expire(TimeSpan.FromMinutes(1))
            .SetVaryByHeader("Authorization"));
});

app.UseOutputCache();  // должно быть после Auth
```

### Применение

```csharp
// На endpoint
app.MapGet("/products", GetProducts).CacheOutput("ProductsPolicy");

app.MapGet("/health", () => "OK").CacheOutput(b => b.Expire(TimeSpan.FromSeconds(1)));

// Без policy — default
app.MapGet("/products/{id}", GetProductById).CacheOutput();

// На контроллере
[OutputCache(PolicyName = "ProductsPolicy")]
public class ProductsController : ControllerBase { }
```

### Tag-based eviction

```csharp
app.MapGet("/products/{id}", async (int id, IProductService svc) => 
    await svc.GetAsync(id))
    .CacheOutput(b => b.Tag($"product-{id}").Tag("products"));

app.MapPost("/products", async (Product p, IProductService svc, IOutputCacheStore cache) =>
{
    await svc.CreateAsync(p);
    await cache.EvictByTagAsync("products", default);  // инвалидируем все products
});

app.MapDelete("/products/{id}", async (int id, IProductService svc, IOutputCacheStore cache) =>
{
    await svc.DeleteAsync(id);
    await cache.EvictByTagAsync($"product-{id}", default);  // только этот продукт
    await cache.EvictByTagAsync("products", default);
});
```

### Distributed cache backing

```csharp
// In-memory (default)
builder.Services.AddOutputCache();

// Redis
builder.Services.AddStackExchangeRedisOutputCache(options =>
{
    options.Configuration = "localhost:6379";
});
```

---

## Rate Limiting (.NET 7+)

Native rate limiting в ASP.NET Core 7+.

### Algorithms

| Algorithm | Что делает |
|-----------|-----------|
| **Fixed Window** | N requests в окне 1 мин — счётчик сбрасывается |
| **Sliding Window** | N requests за rolling 1 мин — точнее |
| **Token Bucket** | Pool токенов, восстанавливается со временем |
| **Concurrency** | N параллельных запросов одновременно |

### Setup

```csharp
builder.Services.AddRateLimiter(options =>
{
    // Default policy — Fixed Window
    options.AddFixedWindowLimiter("FixedPolicy", config =>
    {
        config.PermitLimit = 100;
        config.Window = TimeSpan.FromMinutes(1);
        config.QueueLimit = 10;
        config.QueueProcessingOrder = QueueProcessingOrder.OldestFirst;
    });
    
    // Sliding Window
    options.AddSlidingWindowLimiter("SlidingPolicy", config =>
    {
        config.PermitLimit = 100;
        config.Window = TimeSpan.FromMinutes(1);
        config.SegmentsPerWindow = 6;  // более точное окно
    });
    
    // Token Bucket
    options.AddTokenBucketLimiter("TokenPolicy", config =>
    {
        config.TokenLimit = 100;
        config.TokensPerPeriod = 10;
        config.ReplenishmentPeriod = TimeSpan.FromSeconds(10);
        config.QueueLimit = 5;
    });
    
    // Concurrency limiter
    options.AddConcurrencyLimiter("ConcurrencyPolicy", config =>
    {
        config.PermitLimit = 50;
        config.QueueLimit = 100;
    });
    
    // Per-user / per-IP — partitioned
    options.AddPolicy("PerUser", context =>
    {
        var userId = context.User.Identity?.Name ?? context.Connection.RemoteIpAddress?.ToString();
        return RateLimitPartition.GetFixedWindowLimiter(userId!, _ => 
            new FixedWindowRateLimiterOptions
            {
                PermitLimit = 10,
                Window = TimeSpan.FromMinutes(1)
            });
    });
    
    // Custom rejection
    options.OnRejected = async (ctx, ct) =>
    {
        ctx.HttpContext.Response.StatusCode = StatusCodes.Status429TooManyRequests;
        if (ctx.Lease.TryGetMetadata(MetadataName.RetryAfter, out var retryAfter))
        {
            ctx.HttpContext.Response.Headers.RetryAfter = retryAfter.TotalSeconds.ToString();
        }
        await ctx.HttpContext.Response.WriteAsync("Too many requests", ct);
    };
});

app.UseRateLimiter();
```

### Применение

```csharp
app.MapGet("/api/data", GetData).RequireRateLimiting("FixedPolicy");

// Группа endpoints
app.MapGroup("/api")
    .RequireRateLimiting("PerUser")
    .MapGet("/profile", GetProfile)
    .MapPost("/data", CreateData);

// Disable для конкретного endpoint
app.MapGet("/health", () => "OK").DisableRateLimiting();
```

### Distributed Rate Limiting

Native rate limiter работает **per-instance**. Для distributed (Redis) — нужны сторонние пакеты:

```csharp
// AspNetCoreRateLimit (legacy, поддерживает Redis)
// или RedisRateLimiting community package
// или собственная реализация через Redis sorted set + Lua script
```

См. [System Design — Rate Limiter](../Architecture/system-design.md).

---

## Request Decompression (.NET 7+)

```csharp
builder.Services.AddRequestDecompression();
app.UseRequestDecompression();

// Теперь endpoint автоматически декомпрессирует request body
// если Content-Encoding: gzip/deflate/brotli
```

Полезно для большого payload (file uploads, batch operations).

### Custom decompression provider

```csharp
builder.Services.AddRequestDecompression(options =>
{
    options.DecompressionProviders.Add("custom", new MyCustomDecompressionProvider());
});
```

---

## Response.OnStarting / OnCompleted

Callbacks на этапы response lifecycle.

```csharp
app.Use(async (context, next) =>
{
    // OnStarting — до отправки headers/body клиенту
    context.Response.OnStarting(() =>
    {
        context.Response.Headers["X-Server-Time"] = DateTime.UtcNow.ToString("O");
        return Task.CompletedTask;
    });
    
    // OnCompleted — после полной отправки response
    context.Response.OnCompleted(() =>
    {
        // Логирование, cleanup
        return Task.CompletedTask;
    });
    
    await next(context);
});
```

### Когда нужны

- **OnStarting** — добавить headers, которые зависят от response (после endpoint работы)
- **OnCompleted** — асинхронный cleanup (logging, audit) **не блокируя** response

> [!warning] OnStarting может НЕ вызваться
> Если throw exception до записи response — OnStarting может не успеть выполниться. Используй для не-критичных операций.

---

## Routing

Routing сопоставляет входящий HTTP-запрос с конкретным endpoint'ом.

### Endpoint Routing

```csharp
app.UseRouting();          // Фаза 1: Matching — выбор endpoint по URL и HTTP-методу
// Middleware между Routing и Endpoints имеют доступ к выбранному endpoint
app.UseAuthentication();   // Знает, какой endpoint будет вызван
app.UseAuthorization();    // Может проверить [Authorize] до вызова endpoint
```

Между `UseRouting()` и dispatch можно читать metadata endpoint'а — например, `[Authorize]` атрибуты.

### Параметры маршрутов

```csharp
// Обязательный параметр
app.MapGet("/users/{id}", (int id) => ...);

// Опциональный
app.MapGet("/users/{id?}", (int? id) => ...);

// Constraint
app.MapGet("/users/{id:int}", (int id) => ...);
app.MapGet("/orders/{date:datetime}", (DateTime date) => ...);
app.MapGet("/files/{name:regex(^[a-z]+$)}", (string name) => ...);
app.MapGet("/page/{num:int:min(1):max(100)}", (int num) => ...);

// Catch-all
app.MapGet("/files/{*path}", (string path) => ...);
```

### Custom constraints

```csharp
public class EvenIntConstraint : IRouteConstraint
{
    public bool Match(HttpContext? httpContext, IRouter? route, string routeKey,
        RouteValueDictionary values, RouteDirection routeDirection)
    {
        return values.TryGetValue(routeKey, out var value) 
            && value is int i 
            && i % 2 == 0;
    }
}

// Регистрация
builder.Services.Configure<RouteOptions>(opts =>
{
    opts.ConstraintMap.Add("even", typeof(EvenIntConstraint));
});

// Использование
app.MapGet("/evens/{id:even}", (int id) => $"Even: {id}");
```

### LinkGenerator — генерация URL

```csharp
app.MapGet("/orders/{id}/url", (int id, LinkGenerator linker, HttpContext ctx) =>
{
    var url = linker.GetUriByName(ctx, "GetOrder", new { id });
    return Results.Ok(url);
});

app.MapGet("/orders/{id}", (int id) => $"Order {id}").WithName("GetOrder");
```

---

## Branching: Map, MapWhen, UseWhen, UsePathBase

### Map — fork pipeline по path

```csharp
app.Map("/api", apiApp =>
{
    apiApp.UseAuthentication();
    apiApp.UseAuthorization();
    apiApp.MapGet("/users", () => Users);
});

app.Map("/admin", adminApp =>
{
    adminApp.UseAuthentication();
    adminApp.UseAuthorization();
    adminApp.MapGet("/dashboard", () => "Admin");
});
```

После Map — middleware изолированы между branches.

### MapWhen — conditional fork

```csharp
app.MapWhen(
    ctx => ctx.Request.Headers.ContainsKey("X-API-Key"),
    apiApp => apiApp.UseMiddleware<ApiKeyAuthMiddleware>()
              .Run(async ctx => await ctx.Response.WriteAsync("API")));
```

### UseWhen — conditional middleware, returns to main

```csharp
app.UseWhen(
    ctx => ctx.Request.Path.StartsWithSegments("/api"),
    apiApp => apiApp.UseMiddleware<RequestTimingMiddleware>());

// После UseWhen — обратно в основную цепочку
app.UseAuthentication();
```

### UsePathBase — для reverse proxy / sub-path deploy

```csharp
// App слушает /myapp/* как root
app.UsePathBase("/myapp");
// Теперь /myapp/users работает как /users
```

---

## WebSocket Middleware

```csharp
var app = builder.Build();

app.UseWebSockets(new WebSocketOptions
{
    KeepAliveInterval = TimeSpan.FromMinutes(2),
    AllowedOrigins = { "https://example.com" }
});

app.Use(async (context, next) =>
{
    if (context.Request.Path == "/ws" && context.WebSockets.IsWebSocketRequest)
    {
        using var ws = await context.WebSockets.AcceptWebSocketAsync();
        await HandleWebSocketAsync(ws, context.RequestAborted);
    }
    else
    {
        await next(context);
    }
});

static async Task HandleWebSocketAsync(WebSocket ws, CancellationToken ct)
{
    var buffer = new byte[1024 * 4];
    
    while (ws.State == WebSocketState.Open)
    {
        var result = await ws.ReceiveAsync(buffer, ct);
        
        if (result.MessageType == WebSocketMessageType.Close)
        {
            await ws.CloseAsync(WebSocketCloseStatus.NormalClosure, "Bye", ct);
            break;
        }
        
        var message = Encoding.UTF8.GetString(buffer, 0, result.Count);
        // ... обработка ...
        await ws.SendAsync(buffer.AsMemory(0, result.Count), 
            WebSocketMessageType.Text, true, ct);
    }
}
```

Для production — лучше использовать **SignalR** (см. [SignalR](signalr.md)).

---

## MVC, Razor Pages, Minimal API

| Аспект | MVC | Razor Pages | Minimal API |
|--------|-----|-------------|-------------|
| Модель | Controller + View + Model | PageModel (Page + Model) | Lambda / handler-метод |
| Структура | Отдельные контроллеры | .cshtml + .cshtml.cs | Один файл или extension method |
| Фильтры | Полный набор | Полный набор | Endpoint filters (.NET 7+) |
| Model Binding | Полный | Полный | Базовый (расширяемый) |
| Применение | API, SPA backend, сложная логика | Формы, CRUD-страницы | Микросервисы, простые API |

### Minimal API

```csharp
var app = builder.Build();

// Группировка
var users = app.MapGroup("/api/users")
    .RequireAuthorization()
    .WithTags("Users")
    .RequireRateLimiting("PerUser");

users.MapGet("/", async (IUserService svc) => Results.Ok(await svc.GetAllAsync()));
users.MapGet("/{id:int}", async (int id, IUserService svc) =>
    await svc.GetByIdAsync(id) is User user
        ? Results.Ok(user)
        : Results.NotFound());
users.MapPost("/", async (CreateUserDto dto, IUserService svc) =>
{
    var user = await svc.CreateAsync(dto);
    return Results.Created($"/api/users/{user.Id}", user);
});
```

### Endpoint Filters (.NET 7+)

Аналог Action Filters для Minimal API:

```csharp
app.MapGet("/orders", GetOrders)
    .AddEndpointFilter(async (context, next) =>
    {
        var sw = Stopwatch.StartNew();
        var result = await next(context);
        sw.Stop();
        Console.WriteLine($"Took {sw.ElapsedMilliseconds}ms");
        return result;
    });

// Reusable as class
public class ValidationFilter<T> : IEndpointFilter where T : class
{
    private readonly IValidator<T> _validator;
    
    public ValidationFilter(IValidator<T> validator) => _validator = validator;
    
    public async ValueTask<object?> InvokeAsync(
        EndpointFilterInvocationContext ctx, 
        EndpointFilterDelegate next)
    {
        var arg = ctx.GetArgument<T>(0);
        var result = await _validator.ValidateAsync(arg);
        
        if (!result.IsValid)
            return TypedResults.ValidationProblem(result.ToDictionary());
        
        return await next(ctx);
    }
}

app.MapPost("/orders", CreateOrder)
    .AddEndpointFilter<ValidationFilter<CreateOrderCommand>>();
```

### TypedResults — strongly-typed (.NET 7+)

```csharp
// Старый способ — Results
app.MapGet("/orders/{id}", (int id) => 
    id > 0 ? Results.Ok(new Order()) : Results.BadRequest());
// Type из Results.Ok — IResult, нет указания на возвращаемый тип

// Новый способ — TypedResults
app.MapGet("/orders/{id}", Results<Ok<Order>, BadRequest> (int id) =>
    id > 0 ? TypedResults.Ok(new Order()) : TypedResults.BadRequest());
// Type явно указан → OpenAPI знает что возвращается, тестируемо
```

### Тонкости Minimal API

- DI через параметры handler'а — автоматическое разрешение
- `TypedResults` (.NET 7+) для strongly-typed responses
- Группы (`MapGroup`) поддерживают вложенность и общие фильтры/метаданные
- `.WithName()` — для LinkGenerator, OpenAPI
- `.WithTags()` — для Swagger group
- `.WithSummary()` / `.WithDescription()` — OpenAPI
- Не поддерживают `ModelState` — используйте Endpoint Filters + FluentValidation

---

## Тестирование middleware с TestServer

```csharp
// NuGet: Microsoft.AspNetCore.TestHost
public class MiddlewareTests
{
    [Fact]
    public async Task TimingMiddleware_AddsHeader()
    {
        using var host = await new HostBuilder()
            .ConfigureWebHost(webBuilder =>
            {
                webBuilder.UseTestServer()
                    .ConfigureServices(services => 
                    {
                        services.AddLogging();
                    })
                    .Configure(app =>
                    {
                        app.UseMiddleware<RequestTimingMiddleware>();
                        app.Run(async ctx => await ctx.Response.WriteAsync("OK"));
                    });
            })
            .StartAsync();
        
        var client = host.GetTestClient();
        var response = await client.GetAsync("/");
        
        response.EnsureSuccessStatusCode();
        Assert.True(response.Headers.TryGetValues("X-Elapsed-Ms", out _));
    }
}
```

### WebApplicationFactory — для integration tests

```csharp
public class IntegrationTests : IClassFixture<WebApplicationFactory<Program>>
{
    private readonly HttpClient _client;
    
    public IntegrationTests(WebApplicationFactory<Program> factory)
    {
        _client = factory.WithWebHostBuilder(builder =>
        {
            builder.ConfigureServices(services =>
            {
                // Override services for testing
                services.AddSingleton<IEmailService, MockEmailService>();
            });
        }).CreateClient();
    }
    
    [Fact]
    public async Task GetOrders_Returns200() 
    {
        var response = await _client.GetAsync("/orders");
        response.EnsureSuccessStatusCode();
    }
}
```

---

## Common Pitfalls

### 1. Неправильный порядок middleware

```csharp
// ❌ Authorization до Authentication — User всегда null
app.UseAuthorization();
app.UseAuthentication();

// ❌ CORS после Routing-dispatch
app.MapEndpoints();
app.UseCors();  // никогда не выполнится

// ❌ ExceptionHandler в середине — не ловит ошибки из ранних middleware
app.UseStaticFiles();
app.UseExceptionHandler();  // не поймает ошибки из StaticFiles
```

### 2. Scoped service в convention-based middleware constructor

```csharp
public class MyMiddleware
{
    public MyMiddleware(RequestDelegate next, IDbContext db)  // ❌ Scoped в Singleton!
    {
        // db кэшируется в Singleton — переиспользуется между requests
    }
}

// ✅ Scoped через method params
public async Task InvokeAsync(HttpContext ctx, IDbContext db) { }
```

### 3. Утечка response stream

```csharp
// ❌ Копируем response в memory stream без обратной записи
app.Use(async (ctx, next) =>
{
    var originalBody = ctx.Response.Body;
    using var ms = new MemoryStream();
    ctx.Response.Body = ms;
    
    await next(ctx);
    
    // Забыли скопировать ms обратно в originalBody!
    // → клиент получает пустой response
});

// ✅
app.Use(async (ctx, next) =>
{
    var originalBody = ctx.Response.Body;
    using var ms = new MemoryStream();
    ctx.Response.Body = ms;
    
    await next(ctx);
    
    ms.Seek(0, SeekOrigin.Begin);
    await ms.CopyToAsync(originalBody);
    ctx.Response.Body = originalBody;
});
```

### 4. Middleware не вызывает next() и не отвечает

```csharp
// ❌ Middleware молчит — клиент висит
app.Use(async (ctx, next) =>
{
    if (ctx.Request.Path == "/blocked")
    {
        return;  // не отвечаем!
    }
    await next(ctx);
});

// ✅ Short-circuit с явным response
app.Use(async (ctx, next) =>
{
    if (ctx.Request.Path == "/blocked")
    {
        ctx.Response.StatusCode = 403;
        await ctx.Response.WriteAsync("Forbidden");
        return;
    }
    await next(ctx);
});
```

### 5. Запись в response после next()

```csharp
app.Use(async (ctx, next) =>
{
    await next(ctx);
    
    // ❌ Если endpoint уже отправил response — ошибка
    ctx.Response.Headers["X-Foo"] = "bar";
});

// ✅ Использовать OnStarting
app.Use(async (ctx, next) =>
{
    ctx.Response.OnStarting(() =>
    {
        ctx.Response.Headers["X-Foo"] = "bar";
        return Task.CompletedTask;
    });
    await next(ctx);
});
```

### 6. ExceptionHandler глотает details в production

```csharp
// ❌ Production
app.UseExceptionHandler(b => b.Run(async ctx => 
    await ctx.Response.WriteAsync("Error")));
// Все ошибки → "Error", невозможно debugger в production

// ✅ Логировать, возвращать обобщённое
public class GlobalExceptionHandler(ILogger<GlobalExceptionHandler> logger) : IExceptionHandler
{
    public async ValueTask<bool> TryHandleAsync(...)
    {
        logger.LogError(exception, "Unhandled error: {TraceId}", ctx.TraceIdentifier);
        // Клиенту — generic message + traceId для поддержки
    }
}
```

### 7. Rate Limiter применён до Auth

```csharp
// ❌ Per-user rate limit без user
app.UseRateLimiter();   // partition by user
app.UseAuthentication(); // user не известен в RateLimiter!

// ✅
app.UseAuthentication();
app.UseAuthorization();
app.UseRateLimiter();
```

---

## Best Practices

- **Порядок middleware** — Exception → HSTS/HTTPS → StaticFiles → Routing → CORS → Auth → Authz → RateLimiter → OutputCache → Endpoint
- **IExceptionHandler** для глобальной обработки ошибок (.NET 8+)
- **ProblemDetails** для error responses (RFC 7807)
- **TypedResults** в Minimal API — для strongly-typed responses
- **Scoped service** через method params в convention middleware, через constructor в IMiddleware
- **MapGroup** для общих filters/auth/rate limit на группе endpoints
- **OnStarting** для headers зависящих от endpoint
- **TestServer / WebApplicationFactory** для integration tests middleware и API
- **OutputCache + Tags** — для granular invalidation
- **RateLimiter partitions** — per-user / per-IP, не глобально
- **RequestDecompression** для file uploads
- **TraceIdentifier** в каждом error response — для support

---

## См. также

- [DI и Configuration](di-configuration.md)
- [Auth & Security](auth-security.md)
- [API Design](api-design.md)
- [Caching](caching.md) — Output cache vs Response cache vs Distributed cache
- [Resilience](resilience.md) — Polly + middleware integration
- [Logging & Observability](logging-observability.md)
- [Testing](../Testing/testing.md) — TestServer, WebApplicationFactory

## Reading list

- **Microsoft Docs — Middleware** — learn.microsoft.com/aspnet/core/fundamentals/middleware/
- **Microsoft Docs — IExceptionHandler** — learn.microsoft.com/aspnet/core/fundamentals/error-handling
- **Microsoft Docs — Output Caching** — learn.microsoft.com/aspnet/core/performance/caching/output
- **Microsoft Docs — Rate Limiting** — learn.microsoft.com/aspnet/core/performance/rate-limit
- **Andrew Lock — ASP.NET Core in Action** (книга, главы про pipeline)
- **Stephen Cleary — Async middleware patterns** — blog.stephencleary.com
- **RFC 7807 — Problem Details for HTTP APIs** — datatracker.ietf.org/doc/html/rfc7807
