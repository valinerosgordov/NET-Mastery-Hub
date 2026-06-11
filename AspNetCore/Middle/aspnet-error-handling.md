---
tags: [aspnetcore, error-handling, exceptions, problemdetails, middle, iexceptionhandler]
level: Middle
date: 2026-05-10
---

# ASP.NET Core Error Handling — exceptions, ProblemDetails, IExceptionHandler

> **Exception handling middleware, IExceptionHandler (.NET 8+), ProblemDetails RFC 7807, error pages, validation errors, custom exception types.** Production-grade error handling.

---

## 0. Как читать

После Junior знакомства с try/catch. Здесь — production patterns: middleware, IExceptionHandler, ProblemDetails, structured exceptions, secure error responses.

---

## 1. Default behavior

### 1.1. Без custom handling

```csharp
[HttpGet("{id}")]
public async Task<UserDto> Get(int id)
{
    var user = await _service.GetByIdAsync(id);
    if (user == null) throw new NotFoundException();
    return user;
}
```

Что происходит:
- **Development**: `app.UseDeveloperExceptionPage()` — детальный stack trace HTML page
- **Production**: 500 Internal Server Error без подробностей (security)

### 1.2. Стандартные responses

```csharp
// ASP.NET Core auto-generated 500 response:
{
  "type": "https://tools.ietf.org/html/rfc7231#section-6.6.1",
  "title": "An error occurred while processing your request.",
  "status": 500,
  "traceId": "00-abc..."
}
```

Это **ProblemDetails** (RFC 7807) format.

---

## 2. Exception Handler Middleware (legacy approach)

### 2.1. UseExceptionHandler

```csharp
var app = builder.Build();

if (!app.Environment.IsDevelopment())
{
    app.UseExceptionHandler("/error");   // redirects на /error при exception
}

app.MapGet("/error", (HttpContext context) =>
{
    var exception = context.Features.Get<IExceptionHandlerFeature>()?.Error;
    
    return Results.Problem(
        title: "Server error",
        statusCode: 500);
});
```

### 2.2. Inline handler

```csharp
app.UseExceptionHandler(errorApp =>
{
    errorApp.Run(async context =>
    {
        var exception = context.Features.Get<IExceptionHandlerFeature>()?.Error;
        
        context.Response.StatusCode = exception switch
        {
            ValidationException => 400,
            NotFoundException => 404,
            UnauthorizedException => 401,
            _ => 500
        };
        
        context.Response.ContentType = "application/problem+json";
        await context.Response.WriteAsJsonAsync(new ProblemDetails
        {
            Status = context.Response.StatusCode,
            Title = "Error",
            Detail = exception?.Message
        });
    });
});
```

Работает, но monolithic — все handling в одном месте.

---

## 3. IExceptionHandler (.NET 8+) — modern

### 3.1. Интерфейс

```csharp
public interface IExceptionHandler
{
    ValueTask<bool> TryHandleAsync(
        HttpContext httpContext,
        Exception exception,
        CancellationToken cancellationToken);
}
```

Возвращает `true` если обработал — pipeline останавливается. `false` → следующий handler.

### 3.2. Implementation

```csharp
public class GlobalExceptionHandler : IExceptionHandler
{
    private readonly ILogger<GlobalExceptionHandler> _logger;
    private readonly IHostEnvironment _env;
    
    public GlobalExceptionHandler(
        ILogger<GlobalExceptionHandler> logger,
        IHostEnvironment env)
    {
        _logger = logger;
        _env = env;
    }
    
    public async ValueTask<bool> TryHandleAsync(
        HttpContext httpContext,
        Exception exception,
        CancellationToken cancellationToken)
    {
        _logger.LogError(exception, "Unhandled exception: {Message}", exception.Message);
        
        var (statusCode, title, type) = exception switch
        {
            ValidationException => (400, "Validation Error", "https://tools.ietf.org/html/rfc7231#section-6.5.1"),
            NotFoundException => (404, "Resource Not Found", "https://tools.ietf.org/html/rfc7231#section-6.5.4"),
            UnauthorizedAccessException => (403, "Forbidden", "https://tools.ietf.org/html/rfc7231#section-6.5.3"),
            TimeoutException => (504, "Gateway Timeout", "https://tools.ietf.org/html/rfc7231#section-6.6.5"),
            _ => (500, "Internal Server Error", "https://tools.ietf.org/html/rfc7231#section-6.6.1")
        };
        
        var problem = new ProblemDetails
        {
            Type = type,
            Title = title,
            Status = statusCode,
            Instance = httpContext.Request.Path,
            Detail = _env.IsDevelopment() ? exception.Message : "An error occurred"
        };
        
        if (_env.IsDevelopment())
        {
            problem.Extensions["stackTrace"] = exception.StackTrace;
        }
        
        problem.Extensions["traceId"] = httpContext.TraceIdentifier;
        
        httpContext.Response.StatusCode = statusCode;
        await httpContext.Response.WriteAsJsonAsync(problem, cancellationToken);
        
        return true;   // обработали
    }
}
```

### 3.3. Регистрация

```csharp
builder.Services.AddExceptionHandler<GlobalExceptionHandler>();
builder.Services.AddProblemDetails();   // обязательно

var app = builder.Build();

app.UseExceptionHandler();   // активируем pipeline
```

### 3.4. Несколько handlers — chain

```csharp
builder.Services.AddExceptionHandler<ValidationExceptionHandler>();
builder.Services.AddExceptionHandler<NotFoundExceptionHandler>();
builder.Services.AddExceptionHandler<GlobalExceptionHandler>();   // fallback
```

Order matters — handlers вызываются по очереди до первого `return true`.

```csharp
public class ValidationExceptionHandler : IExceptionHandler
{
    public async ValueTask<bool> TryHandleAsync(
        HttpContext context, Exception exception, CancellationToken ct)
    {
        if (exception is not ValidationException ex) return false;   // не наш тип
        
        var problem = new ValidationProblemDetails(ex.Errors)
        {
            Status = 400,
            Title = "Validation Failed"
        };
        
        context.Response.StatusCode = 400;
        await context.Response.WriteAsJsonAsync(problem, ct);
        return true;
    }
}
```

> [!question]- **Интервью: IExceptionHandler vs UseExceptionHandler middleware?**
> **`UseExceptionHandler`** (legacy) — middleware, **один** monolithic handler. **`IExceptionHandler`** (.NET 8+) — interface, **multiple** handlers chained, separation of concerns. **Plus IExceptionHandler**: 1) Type-specific handlers (ValidationException → ValidationHandler). 2) DI-friendly. 3) Testable отдельно. 4) Composable. **Best practice 2024+**: IExceptionHandler chain — specific handlers first, fallback last.

---

## 4. ProblemDetails — RFC 7807 format

### 4.1. Стандарт

RFC 7807 определяет JSON format для error responses:

```json
{
  "type": "https://example.com/problems/validation",
  "title": "Validation Failed",
  "status": 400,
  "detail": "Email is invalid",
  "instance": "/api/users/123"
}
```

### 4.2. AddProblemDetails

```csharp
builder.Services.AddProblemDetails();

// Дополнительные настройки
builder.Services.AddProblemDetails(options =>
{
    options.CustomizeProblemDetails = ctx =>
    {
        ctx.ProblemDetails.Extensions["traceId"] = ctx.HttpContext.TraceIdentifier;
        ctx.ProblemDetails.Extensions["timestamp"] = DateTimeOffset.UtcNow;
        ctx.ProblemDetails.Extensions["machineName"] = Environment.MachineName;
    };
});
```

Все automatic responses (404, 400 validation, etc.) теперь в ProblemDetails format.

### 4.3. Manual ProblemDetails

```csharp
[HttpGet("{id}")]
public IActionResult Get(int id)
{
    var user = _service.GetById(id);
    if (user == null)
    {
        return Problem(
            type: "https://example.com/problems/not-found",
            title: "User not found",
            detail: $"User {id} does not exist",
            statusCode: 404);
    }
    return Ok(user);
}
```

### 4.4. ValidationProblemDetails

```csharp
[HttpPost]
public IActionResult Create(CreateUserDto dto)
{
    if (!ModelState.IsValid)
    {
        return ValidationProblem(ModelState);
    }
    
    // ...
}
```

Generated:

```json
{
  "type": "https://tools.ietf.org/html/rfc7231#section-6.5.1",
  "title": "One or more validation errors occurred.",
  "status": 400,
  "errors": {
    "Email": ["The Email field is required."],
    "Password": ["Must be at least 8 characters"]
  }
}
```

### 4.5. Custom Extensions

```csharp
return Problem(
    title: "Insufficient funds",
    statusCode: 422,
    extensions: new Dictionary<string, object?>
    {
        ["userId"] = userId,
        ["requestedAmount"] = amount,
        ["availableBalance"] = balance
    });
```

---

## 5. Custom Exception Types

### 5.1. Domain exceptions

```csharp
public abstract class DomainException : Exception
{
    public string Code { get; }
    
    protected DomainException(string code, string message) : base(message)
    {
        Code = code;
    }
}

public class NotFoundException : DomainException
{
    public NotFoundException(string entity, object id) 
        : base("NotFound", $"{entity} with id '{id}' not found") { }
}

public class ValidationException : DomainException
{
    public IDictionary<string, string[]> Errors { get; }
    
    public ValidationException(IDictionary<string, string[]> errors) 
        : base("Validation", "Validation failed")
    {
        Errors = errors;
    }
}

public class ConflictException : DomainException
{
    public ConflictException(string message) : base("Conflict", message) { }
}

public class UnauthorizedException : DomainException
{
    public UnauthorizedException(string message) : base("Unauthorized", message) { }
}
```

### 5.2. Использование

```csharp
public async Task<User> GetByIdAsync(int id)
{
    var user = await _db.Users.FindAsync(id);
    if (user == null) throw new NotFoundException("User", id);
    return user;
}

public async Task<int> CreateAsync(CreateUserDto dto)
{
    if (await _db.Users.AnyAsync(u => u.Email == dto.Email))
        throw new ConflictException($"User with email {dto.Email} already exists");
    
    // ...
}
```

### 5.3. Handler для domain exceptions

```csharp
public class DomainExceptionHandler : IExceptionHandler
{
    public async ValueTask<bool> TryHandleAsync(
        HttpContext context, Exception exception, CancellationToken ct)
    {
        if (exception is not DomainException domain) return false;
        
        var (statusCode, title) = domain switch
        {
            NotFoundException => (404, "Not Found"),
            ValidationException => (400, "Validation Error"),
            ConflictException => (409, "Conflict"),
            UnauthorizedException => (401, "Unauthorized"),
            _ => (400, "Bad Request")
        };
        
        var problem = new ProblemDetails
        {
            Status = statusCode,
            Title = title,
            Detail = domain.Message,
            Extensions = { ["code"] = domain.Code }
        };
        
        if (domain is ValidationException valEx)
        {
            problem.Extensions["errors"] = valEx.Errors;
        }
        
        context.Response.StatusCode = statusCode;
        await context.Response.WriteAsJsonAsync(problem, ct);
        return true;
    }
}
```

---

## 6. Result pattern (alternative to exceptions)

### 6.1. Идея

Не throw exceptions для expected errors — return `Result<T>`:

```csharp
public sealed class Result<T>
{
    public T? Value { get; }
    public Error? Error { get; }
    public bool IsSuccess { get; }
    public bool IsFailure => !IsSuccess;
    
    private Result(T value) { Value = value; IsSuccess = true; }
    private Result(Error error) { Error = error; IsSuccess = false; }
    
    public static Result<T> Ok(T value) => new(value);
    public static Result<T> Fail(Error error) => new(error);
}

public sealed record Error(string Code, string Message, int StatusCode = 400);
```

### 6.2. Service возвращает Result

```csharp
public async Task<Result<UserDto>> CreateUserAsync(CreateUserDto dto)
{
    if (string.IsNullOrEmpty(dto.Email))
        return Result<UserDto>.Fail(new Error("Validation", "Email required", 400));
    
    if (await _db.Users.AnyAsync(u => u.Email == dto.Email))
        return Result<UserDto>.Fail(new Error("Conflict", "Email taken", 409));
    
    var user = new User { Email = dto.Email, Name = dto.Name };
    _db.Users.Add(user);
    await _db.SaveChangesAsync();
    
    return Result<UserDto>.Ok(new UserDto(user.Id, user.Name, user.Email));
}
```

### 6.3. Endpoint mapping

```csharp
app.MapPost("/users", async (CreateUserDto dto, IUserService service) =>
{
    var result = await service.CreateUserAsync(dto);
    
    return result.IsSuccess
        ? Results.Created($"/users/{result.Value!.Id}", result.Value)
        : Results.Problem(
            title: result.Error!.Code,
            detail: result.Error.Message,
            statusCode: result.Error.StatusCode);
});
```

### 6.4. Pros / Cons

```
✅ Result pattern:
- Explicit failure modes
- No exception throwing overhead
- Compiler-checked (нельзя случайно ignore)
- Clear control flow

❌ Cons:
- More verbose
- Need helpers (Match, Map, Bind)
- Mixes return types

✅ Exceptions:
- Simpler code
- Stack trace для debugging
- Standard для unexpected errors

❌ Cons:
- Hidden control flow
- Performance overhead (stack unwinding)
- Easy to forget catch
```

**Best practice 2024+**: Result pattern для **expected business failures** (validation, not found, conflict). Exceptions для **unexpected** (DB down, network failure, programming errors).

> [!question]- **Интервью: exceptions vs Result pattern?**
> **Exceptions** для **unexpected** scenarios — programming errors, infrastructure failures, security violations. **`Result<T>`** для **expected business failures** — validation errors, not found, business rule violations. **Why mixed approach**: 1) Exceptions карают control flow и performance — overuse vs business cases. 2) Result pattern explicit, compile-time checked. 3) **DDD style**: domain operations return `Result<T>`, infrastructure errors throw. **Implementation**: custom Result type или library (FluentResults, ErrorOr). **Endpoint**: Result.Match → IActionResult/IResult.

---

## 7. Logging exceptions

### 7.1. Log levels по типу

```csharp
public class GlobalExceptionHandler : IExceptionHandler
{
    public async ValueTask<bool> TryHandleAsync(
        HttpContext context, Exception exception, CancellationToken ct)
    {
        var logLevel = exception switch
        {
            // Expected — Information
            NotFoundException => LogLevel.Information,
            ValidationException => LogLevel.Information,
            
            // Client errors — Warning
            UnauthorizedException => LogLevel.Warning,
            ConflictException => LogLevel.Warning,
            
            // Server errors — Error
            DbUpdateException => LogLevel.Error,
            
            // Unexpected — Critical
            _ => LogLevel.Error
        };
        
        _logger.Log(logLevel, exception, "Exception {Type}: {Message}",
            exception.GetType().Name, exception.Message);
        
        // ... return response
    }
}
```

### 7.2. Structured logging context

```csharp
using (_logger.BeginScope(new Dictionary<string, object>
{
    ["UserId"] = context.User.GetUserId(),
    ["Path"] = context.Request.Path,
    ["Method"] = context.Request.Method,
    ["TraceId"] = context.TraceIdentifier
}))
{
    _logger.LogError(exception, "Request failed");
}
```

### 7.3. Не log sensitive data

```csharp
// ❌ Может leak password, token
_logger.LogError("Failed request: {Body}", await context.Request.ReadAsStringAsync());

// ✅ Sanitize
_logger.LogError("Failed POST {Path} from {User}", context.Request.Path, userId);
```

---

## 8. Production patterns

### 8.1. Don't expose internals в production

```csharp
public class ProductionExceptionHandler : IExceptionHandler
{
    public async ValueTask<bool> TryHandleAsync(...)
    {
        var problem = new ProblemDetails
        {
            Status = 500,
            Title = "An error occurred",
            // ❌ Detail = exception.Message — может leak: "Cannot connect to db at 10.0.5.42"
            Detail = "An error occurred while processing your request"   // ✅ generic
        };
        
        // Trace ID для support — connect к logs
        problem.Extensions["traceId"] = context.TraceIdentifier;
        
        // Save full details в logs only
        _logger.LogError(exception, "Unhandled with trace {TraceId}", context.TraceIdentifier);
    }
}
```

### 8.2. Correlation ID

```csharp
app.Use(async (context, next) =>
{
    var correlationId = context.Request.Headers["X-Correlation-ID"].FirstOrDefault()
        ?? Guid.NewGuid().ToString();
    
    context.Items["CorrelationId"] = correlationId;
    context.Response.Headers["X-Correlation-ID"] = correlationId;
    
    using (_logger.BeginScope(new { CorrelationId = correlationId }))
    {
        await next();
    }
});
```

### 8.3. Rate limit error responses

```csharp
// Rate limiting middleware (.NET 7+)
builder.Services.AddRateLimiter(options =>
{
    options.OnRejected = async (context, ct) =>
    {
        context.HttpContext.Response.StatusCode = 429;
        await context.HttpContext.Response.WriteAsJsonAsync(new ProblemDetails
        {
            Status = 429,
            Title = "Too Many Requests",
            Detail = "You have exceeded the rate limit. Try again later."
        }, ct);
    };
});
```

### 8.4. Healthcheck failures

```csharp
app.MapHealthChecks("/health", new HealthCheckOptions
{
    ResponseWriter = async (context, report) =>
    {
        context.Response.ContentType = "application/json";
        var response = new
        {
            status = report.Status.ToString(),
            checks = report.Entries.Select(e => new
            {
                name = e.Key,
                status = e.Value.Status.ToString(),
                duration = e.Value.Duration.TotalMilliseconds,
                description = e.Value.Description
            })
        };
        await context.Response.WriteAsJsonAsync(response);
    }
});
```

---

## 9. Common pitfalls

### 9.1. Catch all без logging

```csharp
try
{
    await DoSomething();
}
catch (Exception)   // ❌ silent swallow
{
    // ничего
}
```

**Fix**: log + rethrow или handle properly.

### 9.2. Generic 500 для всего

```csharp
public IActionResult Get(int id)
{
    try
    {
        return Ok(_service.GetById(id));
    }
    catch (Exception ex)
    {
        return StatusCode(500);   // ❌ не различает NotFound, Validation
    }
}
```

**Fix**: централизованный IExceptionHandler с type-specific responses.

### 9.3. Exposing stack traces в production

```csharp
return Problem(
    title: "Error",
    detail: ex.ToString());   // ❌ stack trace в response!
```

### 9.4. Throwing exceptions для control flow

```csharp
public bool ValidateEmail(string email)
{
    try
    {
        new MailAddress(email);
        return true;
    }
    catch
    {
        return false;   // ❌ exception для control flow
    }
}
```

**Fix**: TryParse pattern.

### 9.5. Не handling cancellation

```csharp
public async Task<IActionResult> Get()
{
    try
    {
        var data = await _service.GetAsync();   // OperationCanceledException не handled
        return Ok(data);
    }
    catch (OperationCanceledException)   // ✅ обрабатывай separately
    {
        return StatusCode(499);   // Client Closed Request
    }
}
```

### 9.6. ProblemDetails несовместимый со старыми clients

```csharp
// Если клиенты ожидают custom format — ProblemDetails ломает
{
  "error": "Not Found",
  "code": 404
}

// Custom формат:
public class CustomError
{
    public string Error { get; set; } = "";
    public int Code { get; set; }
}
```

**Fix**: API versioning — v1 custom format, v2 ProblemDetails.

### 9.7. Не testable exception handling

```csharp
// ❌ Hard-coded в middleware
app.UseExceptionHandler(/* inline */);
```

**Fix**: IExceptionHandler — testable отдельно через unit tests.

### 9.8. Race conditions с error responses

```csharp
catch (Exception ex)
{
    context.Response.StatusCode = 500;   // если уже started → InvalidOperationException
    await context.Response.WriteAsJsonAsync(...);
}
```

**Fix**: проверь `context.Response.HasStarted`:

```csharp
if (context.Response.HasStarted)
{
    _logger.LogWarning("Response already started, can't modify");
    return;
}
```

### 9.9. Error logs missing context

```csharp
_logger.LogError(ex, "Something failed");   // ❌ что failed? для кого?
```

**Fix**: structured logging с context (UserId, Path, etc.).

### 9.10. ValidationProblemDetails забыли

```csharp
// Просто Problem не показывает field errors
return Problem(title: "Validation failed");

// ✅ ValidationProblem
return ValidationProblem(ModelState);   // или
return ValidationProblem(new ValidationProblemDetails(errors));
```

> [!question]- **Интервью: топ-3 ошибки error handling?**
> 1) **Catch всех Exception без logging** — silent failures, debugging impossible. Fix: log + rethrow или handle specifically. 2) **Stack traces в production responses** — security leak. Fix: `IsDevelopment()` check, generic message в prod, full details только в logs. 3) **Throwing для control flow** — performance penalty + readability. Fix: TryParse, Result pattern для expected failures. **Bonus**: response.HasStarted check перед modification.

---

## 10. Cheat sheet

```csharp
// === IExceptionHandler (.NET 8+) ===
public class GlobalExceptionHandler : IExceptionHandler
{
    public async ValueTask<bool> TryHandleAsync(
        HttpContext context, Exception ex, CancellationToken ct)
    {
        var (status, title) = ex switch
        {
            ValidationException => (400, "Validation Failed"),
            NotFoundException => (404, "Not Found"),
            ConflictException => (409, "Conflict"),
            UnauthorizedException => (401, "Unauthorized"),
            _ => (500, "Internal Server Error")
        };
        
        context.Response.StatusCode = status;
        await context.Response.WriteAsJsonAsync(new ProblemDetails
        {
            Status = status,
            Title = title,
            Detail = ex.Message,
            Extensions = { ["traceId"] = context.TraceIdentifier }
        }, ct);
        
        return true;
    }
}

// === Registration ===
builder.Services.AddExceptionHandler<DomainExceptionHandler>();
builder.Services.AddExceptionHandler<GlobalExceptionHandler>();
builder.Services.AddProblemDetails();

var app = builder.Build();
app.UseExceptionHandler();

// === Domain exceptions ===
public abstract class DomainException(string code, string message) : Exception(message)
{
    public string Code { get; } = code;
}

public class NotFoundException(string entity, object id) 
    : DomainException("NotFound", $"{entity} {id} not found");

public class ValidationException(IDictionary<string, string[]> errors) 
    : DomainException("Validation", "Validation failed")
{
    public IDictionary<string, string[]> Errors { get; } = errors;
}

// === Result pattern alternative ===
public async Task<Result<UserDto>> CreateUserAsync(CreateUserDto dto)
{
    if (string.IsNullOrEmpty(dto.Email))
        return Result<UserDto>.Fail(new Error("Validation", "Email required"));
    
    // ...
}

// === Endpoint mapping ===
app.MapPost("/users", async (CreateUserDto dto, IUserService service) =>
{
    var result = await service.CreateUserAsync(dto);
    return result.IsSuccess
        ? Results.Created($"/users/{result.Value!.Id}", result.Value)
        : Results.Problem(
            title: result.Error!.Code,
            detail: result.Error.Message,
            statusCode: result.Error.StatusCode);
});

// === Manual ProblemDetails ===
return Results.Problem(
    title: "Insufficient funds",
    detail: "Account balance too low",
    statusCode: 422,
    extensions: new Dictionary<string, object?>
    {
        ["userId"] = userId,
        ["requestedAmount"] = amount
    });

// === ValidationProblem ===
return Results.ValidationProblem(new Dictionary<string, string[]>
{
    ["Email"] = new[] { "Invalid format" },
    ["Password"] = new[] { "Too short" }
});
```

---

## 11. Practice exercises

### 11.1. Implement domain exception chain

Создай domain exceptions hierarchy + handlers:
- `NotFoundException` → 404
- `ValidationException` → 400 с errors
- `ConflictException` → 409
- `UnauthorizedException` → 401
- Fallback handler → 500

### 11.2. Result pattern endpoint

Перепиши endpoint:
```csharp
[HttpPost]
public async Task<IActionResult> CreateUser(CreateUserDto dto)
{
    try
    {
        var user = await _service.CreateAsync(dto);
        return CreatedAtAction(...);
    }
    catch (ValidationException ex) { return BadRequest(...); }
    catch (ConflictException ex) { return Conflict(...); }
}
```

В `Result<UserDto>` pattern. Service не throws, returns Result.

### 11.3. Production-safe responses

Реализуй handler который:
- В Development: shows full stack trace, exception type, всё
- В Production: only generic message + traceId
- Logs full details в обоих случаях
- Includes correlation ID если provided in request

---

## 12. Что читать дальше

1. **`AspNetCore/Senior/api-design.md`** — API design
2. **`AspNetCore/Middle/aspnet-controllers-routing.md`** — controllers
3. **`AspNetCore/Middle/fluent-validation.md`** — validation
4. **`AspNetCore/Senior/logging-observability.md`** — logging deep
5. **`Architecture/Senior/cqrs-mediatr.md`** — Result pattern context

---

## 13. См. также

- [[api-design|AspNetCore/Senior/api-design]] — design
- [[aspnet-controllers-routing|AspNetCore/Middle/aspnet-controllers-routing]] — controllers
- [[aspnet-dependency-injection-deep|AspNetCore/Middle/aspnet-dependency-injection-deep]] — DI
- [[fluent-validation|AspNetCore/Middle/fluent-validation]] — validation
- [[logging-observability|AspNetCore/Senior/logging-observability]] — logging
- [[cqrs-mediatr|Architecture/Senior/cqrs-mediatr]] — Result pattern

---

## 14. Reading list

- **RFC 7807 — Problem Details** — tools.ietf.org/html/rfc7807
- **Microsoft Docs — Error Handling** — learn.microsoft.com/aspnet/core/fundamentals/error-handling
- **Microsoft Docs — IExceptionHandler** — learn.microsoft.com/aspnet/core/fundamentals/error-handling#iexceptionhandler
- **FluentResults** — github.com/altmann/FluentResults
- **ErrorOr** — github.com/amantinband/error-or
- **Andrew Lock — ProblemDetails articles** — andrewlock.net
