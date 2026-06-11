---
tags: [csharp, exceptions, error-handling, middle, try-catch, throw, result-pattern]
level: Middle
date: 2026-05-07
---

# Error Handling — обработка ошибок

> **Exception hierarchy, try/catch/finally, exception filters, custom exceptions, Result pattern, ASP.NET Core middleware.** Закрывает пробел: «знаю про try/catch, не понимаю когда throw vs throw ex, и почему `catch (Exception)` — anti-pattern».

---

## 0. Как читать

Если впервые — раздел 1→3. Custom exceptions — раздел 5. Result pattern (alternative) — раздел 8. Production guidance — раздел 11 (best practices), 13 (pitfalls).

Cross-language якоря свёрнуты. Interview-вопросы встроены.

---

## 1. Что это, зачем и когда

### 1.1. Exception model

Exception — **mechanism** для signaling **abnormal conditions**. Когда метод не может complete его contract — throws exception, caller (или ancestor) catches.

```csharp
public int Divide(int a, int b)
{
    if (b == 0) throw new DivideByZeroException();
    return a / b;
}

try
{
    var x = Divide(10, 0);
}
catch (DivideByZeroException ex)
{
    Console.WriteLine($"Cannot divide: {ex.Message}");
}
```

### 1.2. Exception vs error code

Old-school: error code returns. C# использует **exceptions** для unexpected conditions:

```csharp
// ❌ Old-school error code
public int Divide(int a, int b, out int result)
{
    if (b == 0) { result = 0; return -1; }   // -1 = error
    result = a / b;
    return 0;   // success
}

// ✅ Exception-based
public int Divide(int a, int b) => b == 0 ? throw new DivideByZeroException() : a / b;
```

Преимущества exceptions: clean control flow, propagation up stack, hard to ignore, structured handling.

### 1.3. Главное правило

```
Throw exception для:
  - Невозможность выполнить method contract
  - Invalid arguments (ArgumentException)
  - Не найденный resource (FileNotFoundException, KeyNotFoundException)
  - I/O failure, network error
  - Programming bugs (NullReferenceException, IndexOutOfRangeException)

НЕ throw для:
  - Expected conditions ("user not found" — TryGet pattern или Result<T>)
  - Control flow (никогда не throw для return)
  - Validation user input (выдай structured result)

Validation rule: exceptions для exceptional, не для expected business outcomes.
```

### 1.4. Exception cost

Throwing exception — **expensive**:
- Stack walk для unwinding.
- Capture stack trace.
- Allocation exception object.

~10,000-100,000ns per throw vs ~1ns simple bool check. **Не использовать для control flow в hot path**.

### 1.5. Эволюция

| Версия | Что |
|--------|-----|
| **.NET 1.0** | try/catch/finally, Exception hierarchy |
| **C# 6** | Exception filters (`when`), `nameof` |
| **C# 7+** | throw expression (`?? throw`) |
| **.NET 5+** | `StackTrace.ToString()` improvements |
| **.NET 6+** | `ArgumentNullException.ThrowIfNull` |
| **.NET 7+** | More `ThrowIf` helpers |

> [!info]- Если ты знаешь Java / Python / Rust / Go
> **Java:** очень similar — try/catch/finally, exception hierarchy. Java имеет **checked exceptions** (forced throws declaration). C# — все unchecked.
>
> **Python:** try/except/finally, raise. Pattern matching exceptions через type. EAFP принцип ("easier to ask forgiveness than permission") — exceptions в обычном flow обычно OK.
>
> **Rust:** `Result<T, E>` enum для recoverable errors, `panic!` для unrecoverable. C# `Result<T>` pattern имитирует Rust для critical paths.
>
> **Go:** error values возвращаются как обычные return + tuple. `defer` + `panic`/`recover` для exceptional. Очень different model.

> [!question]- Интервью: чем exception отличается от error code?
> **Exception** — control flow mechanism, propagates up stack, structured handling через try/catch. **Error code** — return value, requires manual check на каждом call site. C# использует exceptions для unexpected conditions (I/O failure, invalid args, not found). Преимущества exceptions: clean code (нет if-error на каждой line), can't ignore (если не catch — propagates), structured (catch by type). Недостатки: cost (~10,000+ ns per throw), не для control flow / hot path. Pattern для recoverable: `TryParse`/`TryGetValue` (return bool + out), `Result<T>` pattern для composable error handling.

---

## 2. Exception hierarchy

### 2.1. System.Exception base

```
Exception
├── SystemException
│   ├── ArgumentException
│   │   ├── ArgumentNullException
│   │   └── ArgumentOutOfRangeException
│   ├── InvalidOperationException
│   ├── NotImplementedException
│   ├── NotSupportedException
│   ├── NullReferenceException
│   ├── IndexOutOfRangeException
│   ├── DivideByZeroException
│   ├── OverflowException
│   ├── FormatException
│   ├── IOException
│   │   ├── FileNotFoundException
│   │   └── DirectoryNotFoundException
│   ├── UnauthorizedAccessException
│   ├── TaskCanceledException
│   └── OperationCanceledException
├── ApplicationException (deprecated, не используй)
└── (custom exceptions)
```

### 2.2. Exception properties

```csharp
public class Exception
{
    public string Message { get; }
    public Exception? InnerException { get; }
    public string? StackTrace { get; }
    public string? Source { get; }
    public IDictionary Data { get; }
    public string? HelpLink { get; set; }
    public int HResult { get; protected set; }
    public MethodBase TargetSite { get; }
}
```

### 2.3. Самые частые built-in

```csharp
throw new ArgumentException("Invalid format", nameof(arg));
throw new ArgumentNullException(nameof(arg));
throw new ArgumentOutOfRangeException(nameof(index), index, "Must be non-negative");
throw new InvalidOperationException("Order is already paid");
throw new NotImplementedException();
throw new NotSupportedException("Read-only collection");
throw new FileNotFoundException("Config not found", path);
throw new UnauthorizedAccessException();
throw new OperationCanceledException();
```

### 2.4. ApplicationException — **не используй**

```csharp
// ❌ Microsoft изначально рекомендовал base для custom, потом отказался
public class MyException : ApplicationException { }
```

Best practice: наследуй от **`Exception`** напрямую.

> [!question]- Интервью: какие самые частые built-in exceptions?
> 1) **`ArgumentException`** — invalid argument (с подклассами `ArgumentNullException`, `ArgumentOutOfRangeException`). 2) **`InvalidOperationException`** — object в неподходящем state ("can't pay an already-paid order"). 3) **`NullReferenceException`** — null dereference (обычно bug, не throw manually). 4) **`IndexOutOfRangeException`** — массив index. 5) **`KeyNotFoundException`** — Dictionary missing key. 6) **`IOException`** + подклассы — file/network. 7) **`OperationCanceledException`** — cancellation. 8) **`OverflowException`** — checked math. **`ApplicationException`** — deprecated, не используй для custom.

---

## 3. try / catch / finally

### 3.1. Базовый

```csharp
try
{
    DoSomething();
}
catch (SpecificException ex)
{
    HandleSpecific(ex);
}
catch (Exception ex)
{
    LogAndRethrow(ex);
}
finally
{
    Cleanup();
}
```

### 3.2. Catch ordering — specific to general

```csharp
// ❌ Order wrong — Exception catches all, IOException unreachable
try { }
catch (Exception ex) { }
catch (IOException ex) { /* unreachable */ }   // ❌ compile warning

// ✅ Specific first
try { }
catch (FileNotFoundException ex) { /* file-specific */ }
catch (IOException ex) { /* general I/O */ }
catch (Exception ex) { /* fallback */ }
```

### 3.3. Catch без variable

```csharp
try { }
catch (NotSupportedException) { /* don't need ex */ }
catch { /* C# 1+ catch any */ }   // catches non-CLS exceptions too
```

### 3.4. Exception filter — `when` (C# 6+)

```csharp
try
{
    response = await client.GetAsync(url);
}
catch (HttpRequestException ex) when (ex.StatusCode == HttpStatusCode.NotFound)
{
    // Только 404
    return null;
}
catch (HttpRequestException ex) when (ex.StatusCode == HttpStatusCode.TooManyRequests)
{
    await Task.Delay(1000);
    return await Retry();
}
catch (HttpRequestException ex)
{
    // Other HTTP errors
    Log.Error(ex);
    throw;
}
```

`when (condition)` — runtime filter. Если false — exception **продолжает propagate** (filter не consumes).

### 3.5. Filter не throws

```csharp
catch (Exception ex) when (LogAndContinue(ex)) { }   // pattern для logging
catch (Exception ex) when (false) { }   // never matches — для debugging

bool LogAndContinue(Exception ex) { Log.Error(ex); return false; }
```

`LogAndContinue` returns false — filter никогда не matches, но logged. Преимущество vs `catch + log + throw`: stack trace **не unwind** — cleaner debugging.

### 3.6. Exception filter + StackTrace preservation

Filter runs **before** stack unwind. Если filter false — original stack trace preserved. С `catch + throw` — stack может unwound и rewound.

```csharp
// Filter — clean stack
catch (Exception ex) when (Log(ex)) { }

// Catch + throw — stack может unwind first
catch (Exception ex) { Log(ex); throw; }
```

### 3.7. throw vs throw ex

```csharp
catch (Exception ex)
{
    throw;       // ✅ rethrows preserving stack
    throw ex;    // ❌ resets stack trace — теряем где originally happened
}
```

`throw;` — preserves. `throw ex;` — resets stack trace **с current line**. Anti-pattern.

### 3.8. Wrap exception

```csharp
catch (IOException ex)
{
    throw new ConfigLoadException("Failed to load config", ex);   // wrap, preserve inner
}

// Caller получает доступ к inner
catch (ConfigLoadException ex)
{
    Console.WriteLine(ex.InnerException?.Message);
}
```

> [!question]- Интервью: чем `throw` отличается от `throw ex`?
> **`throw`** (без exception variable) — **rethrow preserving stack trace**. Continues propagation с original throw site. **`throw ex`** — throws as **new** location, resets stack trace **до current line**. Original throw site lost — debugging hell. Always `throw;` для rethrow в catch. Если хочешь wrap exception: `throw new MyException("...", ex)` — original в InnerException, current stack для wrapper. Microsoft Style + Roslyn analyzer (CA2200) предупреждает `throw ex;`.

---

## 4. Throwing exceptions

### 4.1. Базовый throw

```csharp
throw new ArgumentException("Invalid format", nameof(arg));
```

### 4.2. throw expression (C# 7+)

```csharp
public string GetName() =>
    _name ?? throw new InvalidOperationException("Name not set");

public int FirstPositive(IEnumerable<int> source) =>
    source.FirstOrDefault(x => x > 0) is int n and not 0
        ? n
        : throw new InvalidOperationException("No positive");

var result = condition ? 42 : throw new ArgumentException();
```

`throw` теперь expression, не только statement.

### 4.3. ArgumentNullException.ThrowIfNull (.NET 6+)

```csharp
public void Process(string input, User user)
{
    ArgumentNullException.ThrowIfNull(input);    // throws если null
    ArgumentNullException.ThrowIfNull(user);
    
    // C# 10+ автоматически использует CallerArgumentExpression — paramName captured
}
```

`ThrowIfNull` — convenience helper. Использует `[CallerArgumentExpression]` для paramName.

### 4.4. ArgumentException.ThrowIfNullOrEmpty / WhiteSpace (.NET 7+)

```csharp
public void Process(string name)
{
    ArgumentException.ThrowIfNullOrEmpty(name);
    ArgumentException.ThrowIfNullOrWhiteSpace(name);   // .NET 8+
}
```

### 4.5. ObjectDisposedException.ThrowIf (.NET 7+)

```csharp
public class Service : IDisposable
{
    private bool _disposed;
    
    public void DoWork()
    {
        ObjectDisposedException.ThrowIf(_disposed, this);
        // ...
    }
}
```

### 4.6. ArgumentOutOfRangeException.ThrowIf* (.NET 8+)

```csharp
public void SetCount(int count)
{
    ArgumentOutOfRangeException.ThrowIfNegative(count);
    ArgumentOutOfRangeException.ThrowIfNegativeOrZero(count);
    ArgumentOutOfRangeException.ThrowIfGreaterThan(count, 100);
    ArgumentOutOfRangeException.ThrowIfLessThan(count, 1);
}
```

### 4.7. nameof для paramName

```csharp
public void Set(string value)
{
    if (value == null) throw new ArgumentNullException(nameof(value));   // refactor-safe
}
```

`nameof` — refactor-safe. Если переименуешь parameter, name автоматически updated.

### 4.8. Inner exception

```csharp
try
{
    await ConnectAsync();
}
catch (SocketException ex)
{
    throw new ConnectionFailedException(
        "Could not connect to server",
        innerException: ex);
}

// Custom exception class
public class ConnectionFailedException : Exception
{
    public ConnectionFailedException(string message, Exception innerException)
        : base(message, innerException) { }
}
```

> [!question]- Интервью: чем `ArgumentNullException.ThrowIfNull` лучше manual?
> 1) **Concise** — одна строка вместо `if/throw`. 2) **`[CallerArgumentExpression]`** автоматически captures paramName — refactor-safe. 3) **Inlined fast path** — JIT optimizes simple null check. 4) **Consistent message** — "Value cannot be null. (Parameter 'arg')". 5) **Roslyn analyzer hints** для apply pattern. .NET 6+ имеет `ArgumentNullException.ThrowIfNull`, .NET 7+ — `ArgumentException.ThrowIfNullOrEmpty/WhiteSpace`, `ObjectDisposedException.ThrowIf`, `ArgumentOutOfRangeException.ThrowIf*`. Best practice 2024+: используй helpers вместо manual.

---

## 5. Custom exceptions

### 5.1. Минимальный custom exception

```csharp
public class OrderNotFoundException : Exception
{
    public int OrderId { get; }
    
    public OrderNotFoundException(int orderId)
        : base($"Order with ID {orderId} not found")
    {
        OrderId = orderId;
    }
    
    public OrderNotFoundException(int orderId, Exception inner)
        : base($"Order with ID {orderId} not found", inner)
    {
        OrderId = orderId;
    }
}
```

### 5.2. Когда custom

✅ **Создавай custom когда:**
- Domain-specific error с дополнительным context (OrderId, UserId).
- Caller хочет distinguish typed catches.
- Built-in не fits semantically.

❌ **НЕ создавай когда:**
- Просто rename built-in (`MyArgumentException`).
- Один use site — используй existing.
- Adding behavior через Data dictionary достаточно.

### 5.3. Hierarchy domain exceptions

```csharp
public abstract class DomainException : Exception
{
    protected DomainException(string message) : base(message) { }
    protected DomainException(string message, Exception inner) : base(message, inner) { }
}

public class OrderException : DomainException
{
    protected OrderException(string message) : base(message) { }
}

public class OrderNotFoundException : OrderException
{
    public int OrderId { get; }
    public OrderNotFoundException(int orderId) : base($"Order {orderId} not found")
    {
        OrderId = orderId;
    }
}

public class OrderAlreadyPaidException : OrderException
{
    public int OrderId { get; }
    public OrderAlreadyPaidException(int orderId) : base($"Order {orderId} already paid")
    {
        OrderId = orderId;
    }
}
```

Caller может catch `OrderException` для general, или specific.

### 5.4. Constructors — best practice

```csharp
public class MyException : Exception
{
    // 1) parameterless
    public MyException() { }
    
    // 2) message
    public MyException(string message) : base(message) { }
    
    // 3) message + inner
    public MyException(string message, Exception innerException) : base(message, innerException) { }
}
```

Microsoft FDG: добавь **все три** constructors. Хотя в практике первый часто не нужен.

### 5.5. Дополнительные данные

```csharp
public class ValidationException : Exception
{
    public IReadOnlyDictionary<string, string[]> Errors { get; }
    
    public ValidationException(IDictionary<string, string[]> errors)
        : base("Validation failed")
    {
        Errors = new ReadOnlyDictionary<string, string[]>(new Dictionary<string, string[]>(errors));
    }
}
```

### 5.6. Serialization (legacy)

```csharp
[Serializable]   // .NET Framework / older
public class MyException : Exception
{
    public MyException() { }
    public MyException(string message) : base(message) { }
    public MyException(string message, Exception inner) : base(message, inner) { }
    
    protected MyException(SerializationInfo info, StreamingContext context)
        : base(info, context) { }
}
```

В современном .NET 5+ — `[Serializable]` и `BinaryFormatter` deprecated. Опускай этот constructor.

> [!question]- Интервью: когда создавать custom exception?
> 1) **Domain-specific error** с дополнительным context (OrderId, UserId, validation errors). 2) **Caller хочет typed catch** — `catch (OrderNotFoundException)` cleaner чем checking message text. 3) **Hierarchy domain exceptions** — `OrderException` базовый, специфичные derived. **НЕ создавай** для simple rename built-in (`MyArgumentException`), one-off use, добавление behavior через `Data` dictionary достаточно. Microsoft FDG: все 3 constructors (parameterless, message, message + inner).

---

## 6. finally и using

### 6.1. finally — guaranteed cleanup

```csharp
Resource? r = null;
try
{
    r = AcquireResource();
    Process(r);
}
catch (Exception ex)
{
    Log.Error(ex);
    throw;
}
finally
{
    r?.Release();   // выполняется ВСЕГДА
}
```

`finally` block — выполняется при normal exit, exception, или через cancellation.

### 6.2. using — automatic finally

```csharp
using (var r = AcquireResource())   // эквивалент try/finally + Dispose
{
    Process(r);
}

// C# 8+
using var r2 = AcquireResource();   // dispose в конце scope
```

См. [[dispose-pattern]] раздел 2.

### 6.3. Exception в finally

```csharp
try
{
    // throws Exception A
}
finally
{
    // throws Exception B — A is LOST!
}
```

Exception в finally **перезаписывает** original. Anti-pattern.

**Фикс:** swallow или log в finally, не throw:

```csharp
finally
{
    try { Cleanup(); }
    catch (Exception ex) { Log.Error(ex); }
}
```

### 6.4. await using async cleanup

```csharp
await using var stream = new FileStream(path, FileMode.Open);
// async DisposeAsync — не блокирует thread
```

См. [[dispose-pattern]] раздел 9.

> [!question]- Интервью: что происходит при exception в finally?
> Original exception (из try) **перезаписывается** новым exception из finally. Original lost — debugging hell. Anti-pattern. Best practice: 1) **Не throw в finally**. 2) **Swallow** через nested try/catch + log. 3) **Cleanup operations** должны быть idempotent + safe. Pattern: `try { Cleanup(); } catch (Exception ex) { Log.Error(ex); }`. Дополнительно: с `IDisposable.Dispose()` — should swallow exceptions per design (см. [[dispose-pattern]]). `using` block — generated finally вызывает Dispose, exceptions проблема.

---

## 7. Cancellation

### 7.1. CancellationToken

```csharp
public async Task ProcessAsync(CancellationToken ct)
{
    foreach (var item in items)
    {
        ct.ThrowIfCancellationRequested();
        await ProcessItemAsync(item, ct);
    }
}

// Caller
using var cts = new CancellationTokenSource(TimeSpan.FromSeconds(30));
try
{
    await ProcessAsync(cts.Token);
}
catch (OperationCanceledException)
{
    Console.WriteLine("Cancelled by timeout");
}
```

### 7.2. OperationCanceledException

`OperationCanceledException` (или derived `TaskCanceledException`) — special. Не "ошибка" в business sense.

```csharp
try
{
    await DoAsync(ct);
}
catch (OperationCanceledException) when (ct.IsCancellationRequested)
{
    // Expected cancellation — handle gracefully
}
catch (Exception ex)
{
    // Real error
    Log.Error(ex);
}
```

### 7.3. ThrowIfCancellationRequested

```csharp
ct.ThrowIfCancellationRequested();   // throws OperationCanceledException если cancelled
```

Используй в loops / long operations periodically.

### 7.4. Linked tokens

```csharp
public async Task ProcessAsync(CancellationToken externalToken)
{
    using var timeoutCts = new CancellationTokenSource(TimeSpan.FromSeconds(30));
    using var linked = CancellationTokenSource.CreateLinkedTokenSource(
        externalToken, timeoutCts.Token);
    
    await DoAsync(linked.Token);
    // Cancels если external OR timeout
}
```

> [!question]- Интервью: как handle cancellation gracefully?
> 1) **Принимай `CancellationToken` parameter** в async methods. 2) **Periodic `ct.ThrowIfCancellationRequested()`** в loops. 3) **Pass token down** в nested async calls. 4) **Catch `OperationCanceledException`** на entry point — это не error, expected. 5) **`when (ct.IsCancellationRequested)`** filter различает expected vs other. 6) **`CancellationTokenSource.CreateLinkedTokenSource`** для combining (external + timeout). Best practice: token — last parameter, default `default` (no cancellation if not provided).

---

## 8. Result pattern — alternative

### 8.1. Когда использовать

```
Exception для:
  - Truly unexpected failures
  - Programming bugs
  - I/O / network errors

Result<T> для:
  - Expected business outcomes ("user not found", "validation failed")
  - Hot path где exceptions expensive
  - Composable error handling (railway-oriented programming)
  - Library API где caller wants explicit handling
```

### 8.2. Минимальный Result

```csharp
public abstract record Result<T>
{
    public sealed record Success(T Value) : Result<T>;
    public sealed record Failure(string Error) : Result<T>;
    
    public bool IsSuccess => this is Success;
    public T? GetValueOrDefault() => this is Success s ? s.Value : default;
}

public Result<User> FindUser(int id) =>
    _users.TryGetValue(id, out var user)
        ? new Result<User>.Success(user)
        : new Result<User>.Failure($"User {id} not found");

var result = FindUser(42);
if (result is Result<User>.Success s) Console.WriteLine(s.Value.Name);
else if (result is Result<User>.Failure f) Console.WriteLine(f.Error);
```

### 8.3. С error type

```csharp
public abstract record Result<TValue, TError>
{
    public sealed record Success(TValue Value) : Result<TValue, TError>;
    public sealed record Failure(TError Error) : Result<TValue, TError>;
}

public enum FindUserError { NotFound, Banned, Locked }

public Result<User, FindUserError> FindUser(int id) { /* ... */ }
```

### 8.4. Map / Bind for chaining

```csharp
public abstract record Result<T>
{
    public sealed record Success(T Value) : Result<T>;
    public sealed record Failure(string Error) : Result<T>;
    
    public Result<U> Map<U>(Func<T, U> mapper) => this switch
    {
        Success s => new Result<U>.Success(mapper(s.Value)),
        Failure f => new Result<U>.Failure(f.Error),
        _ => throw new InvalidOperationException()
    };
    
    public Result<U> Bind<U>(Func<T, Result<U>> binder) => this switch
    {
        Success s => binder(s.Value),
        Failure f => new Result<U>.Failure(f.Error),
        _ => throw new InvalidOperationException()
    };
}

// Composing
var orderResult = FindUser(userId)
    .Map(u => new Order(u))
    .Bind(o => CalculateTotal(o));
```

### 8.5. OneOf library

Альтернатива custom Result — библиотека `OneOf<T1, T2, ...>` discriminated unions:

```csharp
using OneOf;

public OneOf<User, NotFoundError, BannedError> FindUser(int id) { /* ... */ }

var result = FindUser(42);
result.Switch(
    user => Console.WriteLine(user.Name),
    notFound => Console.WriteLine("Not found"),
    banned => Console.WriteLine("Banned")
);
```

### 8.6. Когда не Result

```csharp
public Result<int> Divide(int a, int b);   // ❌ overkill для simple math

// vs
public int Divide(int a, int b)
{
    if (b == 0) throw new DivideByZeroException();
    return a / b;
}
```

Result — когда **expected** failure case. Для programming bugs (divide by zero часто programming bug) — exception.

> [!question]- Интервью: когда `Result<T>` vs exception?
> **`Result<T>`** — для **expected** business failures: user not found, validation failed, business rule violation. Caller forced to handle (compile-time). Composable через Map/Bind. Без exception cost. **Exception** — для **truly exceptional** conditions: I/O failure, network error, programming bugs (NullReferenceException), unrecoverable. Performance: exception ~10,000+ ns vs Result ~1ns. Best practice: hot path business logic — Result. Public API library — both options. Mix: Result для "user-actionable" errors, exception для technical failures.

---

## 9. ASP.NET Core integration

### 9.1. Exception middleware

```csharp
public class ExceptionHandlingMiddleware
{
    private readonly RequestDelegate _next;
    private readonly ILogger<ExceptionHandlingMiddleware> _logger;
    
    public ExceptionHandlingMiddleware(RequestDelegate next, ILogger<ExceptionHandlingMiddleware> logger)
    {
        _next = next;
        _logger = logger;
    }
    
    public async Task InvokeAsync(HttpContext context)
    {
        try
        {
            await _next(context);
        }
        catch (NotFoundException ex)
        {
            context.Response.StatusCode = 404;
            await context.Response.WriteAsJsonAsync(new { error = ex.Message });
        }
        catch (ValidationException ex)
        {
            context.Response.StatusCode = 400;
            await context.Response.WriteAsJsonAsync(new { errors = ex.Errors });
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Unhandled exception");
            context.Response.StatusCode = 500;
            await context.Response.WriteAsJsonAsync(new { error = "Internal server error" });
        }
    }
}

// Program.cs
app.UseMiddleware<ExceptionHandlingMiddleware>();
```

### 9.2. UseExceptionHandler

```csharp
// Built-in
app.UseExceptionHandler("/error");

app.MapGet("/error", (HttpContext ctx) =>
{
    var ex = ctx.Features.Get<IExceptionHandlerFeature>()?.Error;
    return Results.Problem(detail: ex?.Message);
});
```

### 9.3. ProblemDetails (RFC 7807)

```csharp
public class ApiException : Exception
{
    public int StatusCode { get; }
    public ApiException(int status, string message) : base(message) => StatusCode = status;
}

// Middleware returns ProblemDetails
catch (ApiException ex)
{
    context.Response.StatusCode = ex.StatusCode;
    await context.Response.WriteAsJsonAsync(new ProblemDetails
    {
        Status = ex.StatusCode,
        Title = ex.Message,
        Type = "https://example.com/errors/api"
    });
}
```

### 9.4. IExceptionHandler (.NET 8+)

```csharp
public class GlobalExceptionHandler : IExceptionHandler
{
    public async ValueTask<bool> TryHandleAsync(
        HttpContext context, Exception exception, CancellationToken ct)
    {
        // Handle, return true = handled
        return false;   // continue to next handler
    }
}

builder.Services.AddExceptionHandler<GlobalExceptionHandler>();
app.UseExceptionHandler();
```

> [!question]- Интервью: как handle exceptions в ASP.NET Core?
> 1) **Custom middleware** в pipeline — try/catch на entry, map exceptions to status codes. 2) **`UseExceptionHandler`** built-in для simple case. 3) **`IExceptionHandler`** (.NET 8+) — registered handler interface, composable. 4) **Domain exceptions hierarchy** — map `OrderNotFoundException` → 404, `ValidationException` → 400, etc. 5) **`ProblemDetails`** (RFC 7807) — standard error format. 6) **Logging** — все unhandled exceptions logged. 7) **Don't expose internals** — generic message в production, details в dev. Always handle на server level — никогда unhandled до user.

---

## 10. Best Practices

### 10.1. Throwing

- ✅ **Specific exceptions** — `ArgumentException` лучше `Exception`.
- ✅ **`nameof` для paramName**.
- ✅ **`ThrowIfNull` helpers** (.NET 6+).
- ✅ **Inner exception** при wrap.
- ✅ **Meaningful messages** — что happened + context.
- ❌ **`throw new Exception(...)`** — слишком general.
- ❌ **`ApplicationException`** — deprecated.
- ❌ **Throw для control flow** — slow.

### 10.2. Catching

- ✅ **Specific to general** order.
- ✅ **`when` filter** — preserve stack, conditional handling.
- ✅ **`throw;`** — никогда `throw ex;`.
- ✅ **Wrap** через `throw new MyException("...", ex)`.
- ✅ **Catch on boundaries** — controllers, middleware, top-level handlers.
- ❌ **Catch всё** без specifics — pokemon catching.
- ❌ **Empty catch** без logging — silent swallow.
- ❌ **Catch + don't handle** — лучше propagate.

### 10.3. Custom exceptions

- ✅ **Inherit Exception** напрямую.
- ✅ **3 constructors** (parameterless, message, message + inner).
- ✅ **Domain hierarchy** — abstract base + specific derived.
- ✅ **Дополнительные properties** для context (OrderId).
- ❌ **`MyArgumentException` rename** — используй built-in.

### 10.4. Result pattern

- ✅ **Для expected business outcomes**.
- ✅ **Map / Bind** для composability.
- ✅ **OneOf library** для discriminated unions.
- ❌ **Для programming bugs** — exception.
- ❌ **Для I/O failures** — exception.

### 10.5. Production

- ✅ **Logging** all unhandled.
- ✅ **Don't expose internals** в API responses.
- ✅ **Cancellation** через CancellationToken + OperationCanceledException.
- ✅ **`ProblemDetails`** RFC 7807 для HTTP errors.

---

## 11. Decision tree

```
Что нужно?
│
├── Throw exception
│   ├── Invalid args → ArgumentException family + ThrowIfNull
│   ├── Wrong state → InvalidOperationException
│   ├── Not found → KeyNotFoundException / FileNotFoundException / custom
│   ├── I/O → IOException family
│   ├── Cancellation → OperationCanceledException
│   ├── Domain rule → custom inherit Exception
│   └── Unexpected → bare Exception (rare)
│
├── Catch exception
│   ├── Specific handling → catch (SpecificException)
│   ├── Conditional → when filter
│   ├── Логирование без consume → when (Log(...) && false)
│   ├── Re-throw → throw; (никогда throw ex;)
│   ├── Wrap → throw new MyException("...", ex)
│   └── Boundary → middleware / top-level handler
│
├── Recoverable / expected error
│   ├── Single value → TryGet / TryParse pattern
│   ├── Composable → Result<T> / OneOf
│   └── Validation → ValidationException с errors dict
│
├── Cancellation
│   ├── Pass CancellationToken
│   ├── ct.ThrowIfCancellationRequested() periodically
│   ├── catch (OperationCanceledException) — expected
│   └── Linked tokens для combining
│
└── ASP.NET Core
    ├── Exception middleware — try/catch на entry
    ├── IExceptionHandler (.NET 8+)
    ├── UseExceptionHandler / Results.Problem
    └── ProblemDetails (RFC 7807)
```

---

## 12. Cheat sheet

```csharp
// === Throw ===
throw new ArgumentException("...", nameof(arg));
throw new ArgumentNullException(nameof(arg));
throw new InvalidOperationException("Order is paid");
throw new NotImplementedException();

// .NET 6+ helpers
ArgumentNullException.ThrowIfNull(arg);
ArgumentException.ThrowIfNullOrEmpty(name);
ArgumentException.ThrowIfNullOrWhiteSpace(name);   // .NET 8+
ObjectDisposedException.ThrowIf(_disposed, this);   // .NET 7+
ArgumentOutOfRangeException.ThrowIfNegative(count);   // .NET 8+
ArgumentOutOfRangeException.ThrowIfGreaterThan(count, max);

// throw expression (C# 7+)
var x = obj ?? throw new ArgumentNullException(nameof(obj));

// === Try / catch / finally ===
try { /* ... */ }
catch (SpecificException ex) when (ex.Code == 42) { /* filter */ }
catch (Exception ex) { Log.Error(ex); throw; }
finally { Cleanup(); }

// === Custom exception ===
public class OrderNotFoundException : Exception
{
    public int OrderId { get; }
    
    public OrderNotFoundException(int orderId)
        : base($"Order {orderId} not found") { OrderId = orderId; }
    
    public OrderNotFoundException(int orderId, Exception inner)
        : base($"Order {orderId} not found", inner) { OrderId = orderId; }
}

// === Cancellation ===
public async Task DoAsync(CancellationToken ct)
{
    foreach (var x in items)
    {
        ct.ThrowIfCancellationRequested();
        await ProcessAsync(x, ct);
    }
}

// === Result ===
public abstract record Result<T>
{
    public sealed record Success(T Value) : Result<T>;
    public sealed record Failure(string Error) : Result<T>;
}

// === ASP.NET Core middleware ===
public class ExceptionMiddleware
{
    public async Task InvokeAsync(HttpContext context, RequestDelegate next)
    {
        try { await next(context); }
        catch (NotFoundException ex)
        {
            context.Response.StatusCode = 404;
            await context.Response.WriteAsJsonAsync(new { ex.Message });
        }
        catch (Exception)
        {
            context.Response.StatusCode = 500;
            await context.Response.WriteAsJsonAsync(new { error = "Internal error" });
        }
    }
}
```

---

## 13. Common Pitfalls

### 13.1. throw ex resets stack

```csharp
catch (Exception ex)
{
    throw ex;   // ❌ stack from current line
}
```

**Фикс:** `throw;`.

### 13.2. Pokemon catching

```csharp
try { /* ... */ }
catch (Exception)   // catches ALL — обыч anti-pattern
{
    // silent
}
```

**Фикс:** specific exceptions, log если нужен fallback.

### 13.3. Empty catch

```csharp
try { }
catch { }   // ❌ silently swallow
```

**Фикс:** at least log.

### 13.4. Exception в finally

```csharp
finally
{
    Cleanup();   // throws — original exception lost
}
```

**Фикс:** `try { Cleanup(); } catch (Exception ex) { Log.Error(ex); }`.

### 13.5. Exception для control flow

```csharp
try { return int.Parse(input); }
catch (FormatException) { return 0; }   // ❌ slow
```

**Фикс:** `int.TryParse(input, out var x) ? x : 0`.

### 13.6. Throw в hot path

```csharp
public bool Validate(string input)
{
    try { CheckSomething(input); return true; }
    catch { return false; }   // ❌ exception cost в loop
}
```

**Фикс:** explicit checks.

### 13.7. Catch (Exception) на боковой level

```csharp
public void Handler() {
    try { _service.DoWork(); }
    catch (Exception) { }   // ❌ swallows everything
}
```

**Фикс:** catch только что можешь обработать.

### 13.8. ApplicationException base

```csharp
public class MyException : ApplicationException { }   // ❌ deprecated
```

**Фикс:** inherit `Exception`.

### 13.9. NullReferenceException в production

NRE — почти всегда **bug**, не expected. Если throw NRE — fix root cause (NRT, ArgumentNullException ранее), не catch.

### 13.10. catch (OperationCanceledException) с logging

```csharp
catch (OperationCanceledException ex)
{
    Log.Error(ex);   // ❌ это expected, не error
}
```

**Фикс:** `Log.Information("Cancelled")` или ничего.

> [!question]- Интервью: топ-3 ошибки с exceptions?
> 1) **`throw ex;`** — resets stack trace до current line, debugging hell. Always `throw;` для rethrow. 2) **Pokemon catching** (`catch (Exception)`) на любом уровне — swallows real bugs. Specific exceptions only, logging если fallback. 3) **Exception для control flow** — try/catch в loop вместо TryParse / explicit check. Throwing exception ~10,000+ ns vs ~1ns explicit check. Hot path — никогда exception для expected. Бонус: empty catch (`catch {}` без logging) — bugs vanish silently.

---

## 14. Practice exercises

### 14.1. Custom exception hierarchy

```csharp
public abstract class DomainException : Exception
{
    protected DomainException(string message) : base(message) { }
    protected DomainException(string message, Exception inner) : base(message, inner) { }
}

public class OrderException : DomainException
{
    public int OrderId { get; }
    protected OrderException(int orderId, string message) : base(message) => OrderId = orderId;
}

public class OrderNotFoundException : OrderException
{
    public OrderNotFoundException(int orderId)
        : base(orderId, $"Order {orderId} not found") { }
}

public class OrderAlreadyPaidException : OrderException
{
    public DateTime PaidAt { get; }
    public OrderAlreadyPaidException(int orderId, DateTime paidAt)
        : base(orderId, $"Order {orderId} already paid at {paidAt:O}")
    {
        PaidAt = paidAt;
    }
}

// Использование
try { _service.PayOrder(42); }
catch (OrderAlreadyPaidException ex)
{
    Console.WriteLine($"Already paid at {ex.PaidAt}");
}
catch (OrderException ex)   // fallback для других order errors
{
    Console.WriteLine($"Order error: {ex.Message}");
}
```

### 14.2. Result pattern

```csharp
public abstract record Result<T>
{
    public sealed record Success(T Value) : Result<T>;
    public sealed record Failure(string Error) : Result<T>;
    
    public Result<U> Map<U>(Func<T, U> f) => this switch
    {
        Success s => new Result<U>.Success(f(s.Value)),
        Failure fail => new Result<U>.Failure(fail.Error),
        _ => throw new InvalidOperationException()
    };
    
    public Result<U> Bind<U>(Func<T, Result<U>> f) => this switch
    {
        Success s => f(s.Value),
        Failure fail => new Result<U>.Failure(fail.Error),
        _ => throw new InvalidOperationException()
    };
}

public static class UserService
{
    public static Result<User> FindUser(int id) =>
        id <= 0 ? new Result<User>.Failure("Invalid id") :
        Repository.TryFind(id) is { } user ? new Result<User>.Success(user) :
        new Result<User>.Failure($"User {id} not found");
    
    public static Result<Order> CreateOrder(int userId) =>
        FindUser(userId)
            .Map(u => new Order(u))
            .Bind(o => o.User.IsBanned
                ? new Result<Order>.Failure("User banned")
                : new Result<Order>.Success(o));
}

var result = UserService.CreateOrder(42);
if (result is Result<Order>.Success s)
    Console.WriteLine($"Order created for {s.Value.User.Name}");
else if (result is Result<Order>.Failure f)
    Console.WriteLine($"Failed: {f.Error}");
```

### 14.3. ASP.NET Core exception middleware

```csharp
public class GlobalExceptionMiddleware
{
    private readonly RequestDelegate _next;
    private readonly ILogger<GlobalExceptionMiddleware> _logger;
    
    public GlobalExceptionMiddleware(RequestDelegate next, ILogger<GlobalExceptionMiddleware> logger)
    {
        _next = next;
        _logger = logger;
    }
    
    public async Task InvokeAsync(HttpContext context)
    {
        try { await _next(context); }
        catch (OrderNotFoundException ex)
        {
            await WriteError(context, 404, ex.Message);
        }
        catch (ValidationException ex)
        {
            context.Response.StatusCode = 400;
            await context.Response.WriteAsJsonAsync(new
            {
                errors = ex.Errors,
                title = "Validation failed"
            });
        }
        catch (UnauthorizedAccessException ex)
        {
            await WriteError(context, 403, ex.Message);
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Unhandled exception in {Path}", context.Request.Path);
            await WriteError(context, 500, "Internal server error");
        }
    }
    
    private static Task WriteError(HttpContext context, int statusCode, string message)
    {
        context.Response.StatusCode = statusCode;
        return context.Response.WriteAsJsonAsync(new ProblemDetails
        {
            Status = statusCode,
            Title = message
        });
    }
}

// Program.cs
app.UseMiddleware<GlobalExceptionMiddleware>();
```

---

## 15. Что читать дальше

1. **[[dispose-pattern|Dispose Pattern]]** — exceptions in Dispose.
2. **[[nullable-types|Nullable Types]]** — ArgumentNullException + NRT.
3. **CancellationToken patterns** — async cancellation.
4. **OneOf library** — discriminated unions.
5. **ProblemDetails RFC 7807** — HTTP error format.

---

## 16. См. также

- [[dispose-pattern|Dispose Pattern]] — try/finally
- [[nullable-types|Nullable Types]] — null exceptions
- [[debugging-basics|Debugging]] — exception breakpoints
- ASP.NET Core middleware
- OneOf library (github.com/mcintyre321/OneOf)
- ProblemDetails (RFC 7807)

---

## 17. Reading list

- **Microsoft Docs — Exception handling** — learn.microsoft.com/dotnet/csharp/fundamentals/exceptions/
- **Microsoft Docs — Best practices for exceptions** — learn.microsoft.com/dotnet/standard/exceptions/best-practices-for-exceptions
- **Microsoft Docs — Exception class** — learn.microsoft.com/dotnet/api/system.exception
- **Microsoft Docs — ArgumentNullException.ThrowIfNull** — learn.microsoft.com/dotnet/api/system.argumentnullexception.throwifnull
- **Microsoft FDG — Exceptions guidelines** — learn.microsoft.com/dotnet/standard/design-guidelines/exceptions
- **Stephen Cleary — Cancellation patterns** — blog.stephencleary.com
- **Vladimir Khorikov — Result pattern** — enterprisecraftsmanship.com
- **Scott Wlaschin — Railway-Oriented Programming** — fsharpforfunandprofit.com
