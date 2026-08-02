---
tags: [aspnetcore, controllers, minimal-api, routing, middle, model-binding]
level: Middle
date: 2026-08-02
---

# ASP.NET Core Controllers vs Minimal API — routing, binding, return types

> **Когда Controllers, когда Minimal API. Routing patterns, model binding, return types, action filters, MapGroup.** Closes пробел между Junior http-fundamentals и Senior api-design.

---

## 0. Как читать

После `Junior/http-fundamentals.md`. Перед `Senior/api-design.md`. Здесь — practical decisions: какой подход выбрать, как организовать routing, return types, модель binding.

---

## 1. Controllers vs Minimal API — выбор

### 1.1. Эволюция

```
ASP.NET Core 1.0 (2016):  Controllers (наследие MVC)
ASP.NET Core 6.0 (2021):  Minimal APIs (новый подход)
ASP.NET Core 7-9 (2022+): Minimal APIs паритет с Controllers
```

### 1.2. Сравнение

| Критерий | Controllers | Minimal API |
|----------|-------------|-------------|
| Boilerplate | Больше | Меньше |
| Performance | Чуть медленнее | Быстрее (~10-15%) |
| Convention-based | Да (DI через constructor) | Нет (DI через parameters) |
| Filter support | ✅ Action filters | ✅ Endpoint filters (.NET 7+) |
| Model binding | Полный | Полный |
| OpenAPI/Swagger | Из коробки | Из коробки |
| AOT compatibility | Limited | ✅ Полная |
| Versioning | Через атрибуты | Через MapGroup |
| Когда лучше | Сложные API, MVC views | Микросервисы, простые API |

### 1.3. Decision tree

```
Какой подход?
│
├── Новый проект, .NET 8+
│   ├── Простой CRUD API → Minimal API
│   ├── Микросервис → Minimal API
│   ├── AOT compilation нужна → Minimal API
│   └── Сложная organization, many filters → Controllers
│
├── Existing project с Controllers
│   ├── Не мигрируй ради миграции
│   └── Новые фичи можно через Minimal API в том же app
│
└── MVC views (server-side rendering)
    └── Controllers (Minimal API не для views)
```

> [!question]- **Интервью: Minimal API vs Controllers — trade-offs?**
> **Minimal API**: меньше boilerplate, faster (no controller activation), AOT-ready, modern (.NET 6+). **Controllers**: больше features (custom filters, conventions, view results), Familiar для MVC developers, легче organize большие APIs. **Best practice 2024+**: 1) Микросервисы / простые API → Minimal. 2) Сложные APIs с many filters / conventions → Controllers. 3) AOT critical → Minimal. **Можно mix** в одном app — Minimal для одних endpoints, Controllers для других. **Performance**: Minimal ~10-15% faster для simple CRUD из-за отсутствия controller activation overhead.

---

## 2. Controllers — fundamentals

### 2.1. Базовая структура

```csharp
[ApiController]
[Route("api/[controller]")]
public class UsersController : ControllerBase
{
    private readonly IUserService _service;
    private readonly ILogger<UsersController> _logger;
    
    public UsersController(IUserService service, ILogger<UsersController> logger)
    {
        _service = service;
        _logger = logger;
    }
    
    [HttpGet("{id}")]
    public async Task<ActionResult<UserDto>> Get(int id)
    {
        var user = await _service.GetByIdAsync(id);
        return user == null ? NotFound() : Ok(user);
    }
    
    [HttpPost]
    public async Task<ActionResult<UserDto>> Create([FromBody] CreateUserDto dto)
    {
        var id = await _service.CreateAsync(dto);
        return CreatedAtAction(nameof(Get), new { id }, dto);
    }
}
```

### 2.2. Routing attributes

```csharp
[Route("api/users")]                       // controller-level
public class UsersController : ControllerBase
{
    [HttpGet]                              // GET api/users
    public IActionResult List() { }
    
    [HttpGet("{id}")]                      // GET api/users/{id}
    public IActionResult Get(int id) { }
    
    [HttpGet("{id}/orders")]               // GET api/users/{id}/orders
    public IActionResult GetOrders(int id) { }
    
    [HttpPost]                             // POST api/users
    public IActionResult Create() { }
    
    [HttpPut("{id}")]
    public IActionResult Update(int id) { }
    
    [HttpDelete("{id}")]
    public IActionResult Delete(int id) { }
    
    [HttpGet("search")]                    // GET api/users/search
    public IActionResult Search() { }
}
```

### 2.3. Constraints в routes

```csharp
[HttpGet("{id:int}")]              // только integer
[HttpGet("{slug:alpha}")]           // только letters
[HttpGet("{date:datetime}")]        // datetime
[HttpGet("{guid:guid}")]            // GUID
[HttpGet("{name:length(2,50)}")]    // 2-50 chars
[HttpGet("{age:int:min(18)}")]      // int >= 18
[HttpGet("{regex:regex(^[a-z]+$)}")] // custom regex
```

### 2.4. Action results

```csharp
public IActionResult Method()
{
    return Ok();                          // 200
    return Ok(data);                      // 200 + body
    return Created("/api/users/1", user); // 201
    return CreatedAtAction(...);          // 201 + Location header
    return Accepted();                    // 202
    return NoContent();                   // 204
    return BadRequest("error");           // 400
    return Unauthorized();                // 401
    return Forbid();                      // 403
    return NotFound();                    // 404
    return Conflict();                    // 409
    return UnprocessableEntity();         // 422
    return StatusCode(500);               // any code
    
    // ProblemDetails (RFC 7807)
    return Problem("Detail", statusCode: 500);
    return ValidationProblem();
}
```

### 2.5. `ActionResult<T>` — typed return

```csharp
[HttpGet("{id}")]
public async Task<ActionResult<UserDto>> Get(int id)
{
    var user = await _service.GetByIdAsync(id);
    if (user == null) return NotFound();
    return user;   // implicit conversion to Ok(user)
}
// Better Swagger generation, type-safety
```

### 2.6. [ApiController] что даёт

```csharp
[ApiController]   // mandatory for modern API controllers
public class UsersController : ControllerBase
{
    // Автоматически:
    // 1. Model validation errors → 400 BadRequest без manual check
    // 2. [FromBody] inference для complex types
    // 3. [FromRoute] для route parameters
    // 4. ProblemDetails for 4xx/5xx
}
```

---

## 3. Minimal API — fundamentals

### 3.1. Базовая структура

```csharp
// Program.cs
var builder = WebApplication.CreateBuilder(args);

builder.Services.AddScoped<IUserService, UserService>();

var app = builder.Build();

// Endpoints inline
app.MapGet("/api/users/{id}", async (int id, IUserService service) =>
{
    var user = await service.GetByIdAsync(id);
    return user == null ? Results.NotFound() : Results.Ok(user);
});

app.MapPost("/api/users", async (CreateUserDto dto, IUserService service) =>
{
    var id = await service.CreateAsync(dto);
    return Results.Created($"/api/users/{id}", new { id });
});

app.Run();
```

### 3.2. MapGroup — organization

```csharp
var users = app.MapGroup("/api/users")
    .RequireAuthorization()                 // applied к всей группе
    .WithTags("Users")                      // OpenAPI tag
    .AddEndpointFilter<LoggingFilter>();    // .NET 7+

users.MapGet("/", GetAllUsers);
users.MapGet("/{id}", GetUser);
users.MapPost("/", CreateUser);
users.MapPut("/{id}", UpdateUser);
users.MapDelete("/{id}", DeleteUser);

// Handler functions
async Task<IResult> GetUser(int id, IUserService service)
{
    var user = await service.GetByIdAsync(id);
    return user == null ? Results.NotFound() : Results.Ok(user);
}
```

### 3.3. TypedResults — type-safe return

```csharp
// IResult — generic
app.MapGet("/users/{id}", async (int id, IUserService service) =>
{
    var user = await service.GetByIdAsync(id);
    return user == null ? Results.NotFound() : Results.Ok(user);
});

// TypedResults — strongly typed (.NET 7+)
app.MapGet("/users/{id}", async Task<Results<Ok<UserDto>, NotFound>> (int id, IUserService service) =>
{
    var user = await service.GetByIdAsync(id);
    return user == null ? TypedResults.NotFound() : TypedResults.Ok(user);
});
// Better OpenAPI generation, AOT-friendly
```

### 3.4. Все Result helpers

```csharp
Results.Ok(data);                            // 200
Results.Created("/api/users/1", data);       // 201
Results.CreatedAtRoute("name", values, body);
Results.Accepted();                          // 202
Results.NoContent();                         // 204
Results.BadRequest();                        // 400
Results.Unauthorized();                      // 401
Results.Forbid();                            // 403
Results.NotFound();                          // 404
Results.Conflict();                          // 409
Results.UnprocessableEntity();               // 422
Results.StatusCode(500);
Results.Problem("Detail", statusCode: 500);  // RFC 7807
Results.ValidationProblem(errors);
Results.File("path/to/file.pdf");
Results.Stream(stream, "application/pdf");
Results.Redirect("/new-url");
```

### 3.5. Endpoint filters (.NET 7+)

```csharp
app.MapGet("/users/{id}", async (int id, IUserService service) =>
{
    var user = await service.GetByIdAsync(id);
    return user == null ? Results.NotFound() : Results.Ok(user);
})
.AddEndpointFilter(async (context, next) =>
{
    var sw = Stopwatch.StartNew();
    var result = await next(context);
    Console.WriteLine($"Took {sw.ElapsedMilliseconds}ms");
    return result;
});
```

> [!question]- **Интервью: что дает MapGroup?**
> Группировка endpoints под общим prefix + applying middleware/filters/conventions ко всей группе одной строкой. **Use cases**: 1) Версионирование API: `app.MapGroup("/api/v1")` и `app.MapGroup("/api/v2")`. 2) Auth: `.RequireAuthorization()` на группу. 3) Common filters / tags / conventions. 4) **Lazy registration** — endpoints в отдельных файлах через extension methods. **Аналог Controllers' [Route]/[Authorize]** на class-level, но более flexible.

---

## 4. Model Binding — откуда приходят данные

### 4.1. Источники

```
1. Route values        — URL path: /users/{id}
2. Query string        — ?page=1&size=20
3. Request body        — JSON / XML
4. Form data           — application/x-www-form-urlencoded
5. Headers             — Authorization, X-Custom
6. Services            — DI container (Minimal API)
```

### 4.2. Controllers binding

```csharp
[HttpGet("{id}")]
public IActionResult Get(
    [FromRoute] int id,                           // /users/{id}
    [FromQuery] string? search,                   // ?search=...
    [FromQuery] int page = 1,                     // ?page=1
    [FromHeader(Name = "X-Tenant")] string tenant, // header
    [FromBody] FilterDto filter,                  // JSON body
    [FromServices] IUserService service)          // DI
{
    // ...
}
```

С `[ApiController]` атрибутом — большинство binding inferred:

```csharp
[ApiController]
public class UsersController : ControllerBase
{
    [HttpPost]
    public IActionResult Create(CreateUserDto dto)
    {
        // dto inferred as [FromBody] (complex type)
    }
    
    [HttpGet("{id}")]
    public IActionResult Get(int id)
    {
        // id inferred as [FromRoute] (matches route template)
    }
}
```

### 4.3. Minimal API binding

Inference rules:
- **Route values** match parameter names → `[FromRoute]`
- **Query string** match parameter names → `[FromQuery]` (для simple types)
- **Complex types** (classes/records) → `[FromBody]`
- **Service в DI container** → `[FromServices]`

```csharp
app.MapGet("/users/{id}", async (
    int id,                          // route
    string? search,                  // query
    HttpContext context,             // built-in service
    IUserService service) =>          // DI service
{
    // ...
});

app.MapPost("/users", async (
    CreateUserDto dto,               // body (complex type)
    IUserService service) =>          // DI
{
    // ...
});
```

### 4.4. Explicit attributes

```csharp
app.MapPost("/users", async (
    [FromBody] CreateUserDto dto,
    [FromHeader(Name = "X-Tenant")] string tenant,
    [FromQuery] string? referrer,
    IUserService service) => { });
```

### 4.5. Custom binding с TryParse / BindAsync

```csharp
public class DateRange
{
    public DateTime From { get; set; }
    public DateTime To { get; set; }
    
    // Static TryParse — для query/route values
    public static bool TryParse(string? value, IFormatProvider? provider, out DateRange? result)
    {
        result = null;
        var parts = value?.Split(',');
        if (parts?.Length != 2) return false;
        if (!DateTime.TryParse(parts[0], out var from)) return false;
        if (!DateTime.TryParse(parts[1], out var to)) return false;
        result = new DateRange { From = from, To = to };
        return true;
    }
}

// Usage
app.MapGet("/orders", (DateRange range) => /* ... */);
// GET /orders?range=2024-01-01,2024-12-31
```

### 4.6. AsParameters — bind в class

```csharp
public class GetOrdersRequest
{
    public int Page { get; set; } = 1;
    public int Size { get; set; } = 20;
    public string? Search { get; set; }
    public OrderStatus? Status { get; set; }
}

app.MapGet("/orders", ([AsParameters] GetOrdersRequest request) =>
{
    // Все properties auto-bound из query string
});
```

> [!question]- **Интервью: model binding в Minimal API без [FromBody]?**
> Minimal API использует **inference rules**: 1) Parameter name matches route → FromRoute. 2) Simple type matches query → FromQuery. 3) Complex type (class/record) → FromBody. 4) Type registered в DI → FromServices. 5) Built-in (HttpContext, ClaimsPrincipal) → auto-injected. **Override**: explicit attributes ([FromBody], [FromHeader]). **Custom binding**: implement static `TryParse` (для simple values) или `BindAsync` (для complex async). **Bonus**: `[AsParameters]` для bundling много query params в один class.

---

## 5. Validation

### 5.1. Data Annotations

```csharp
public record CreateUserDto
{
    [Required]
    [StringLength(100, MinimumLength = 2)]
    public string Name { get; init; } = "";
    
    [Required]
    [EmailAddress]
    public string Email { get; init; } = "";
    
    [Range(0, 150)]
    public int Age { get; init; }
}
```

### 5.2. Controllers — auto validation

```csharp
[ApiController]
public class UsersController : ControllerBase
{
    [HttpPost]
    public IActionResult Create(CreateUserDto dto)
    {
        // [ApiController] автоматически:
        // - Validate dto
        // - Return 400 Bad Request если invalid
        // - С detailed errors в ProblemDetails format
        
        // Если дошли сюда — validation passed
        return Ok();
    }
}
```

### 5.3. Minimal API — manual или endpoint filter

```csharp
// Manual в endpoint
app.MapPost("/users", (CreateUserDto dto) =>
{
    var validationContext = new ValidationContext(dto);
    var results = new List<ValidationResult>();
    
    if (!Validator.TryValidateObject(dto, validationContext, results, true))
    {
        return Results.ValidationProblem(
            results.ToDictionary(
                r => r.MemberNames.FirstOrDefault() ?? "",
                r => new[] { r.ErrorMessage ?? "" }));
    }
    
    // Process...
    return Results.Ok();
});

// Лучше через endpoint filter (.NET 7+)
public class ValidationFilter<T> : IEndpointFilter
{
    public async ValueTask<object?> InvokeAsync(EndpointFilterInvocationContext context, EndpointFilterDelegate next)
    {
        var arg = context.Arguments.OfType<T>().FirstOrDefault();
        if (arg == null) return await next(context);
        
        var validationContext = new ValidationContext(arg);
        var results = new List<ValidationResult>();
        
        if (!Validator.TryValidateObject(arg, validationContext, results, true))
        {
            return Results.ValidationProblem(
                results.ToDictionary(
                    r => r.MemberNames.FirstOrDefault() ?? "",
                    r => new[] { r.ErrorMessage ?? "" }));
        }
        
        return await next(context);
    }
}

app.MapPost("/users", (CreateUserDto dto) => /* ... */)
   .AddEndpointFilter<ValidationFilter<CreateUserDto>>();
```

### 5.4. FluentValidation

```bash
dotnet add package FluentValidation
dotnet add package FluentValidation.DependencyInjectionExtensions
```

```csharp
public class CreateUserValidator : AbstractValidator<CreateUserDto>
{
    public CreateUserValidator()
    {
        RuleFor(x => x.Name).NotEmpty().Length(2, 100);
        RuleFor(x => x.Email).NotEmpty().EmailAddress();
        RuleFor(x => x.Age).InclusiveBetween(0, 150);
        RuleFor(x => x.Password)
            .NotEmpty()
            .MinimumLength(8)
            .Matches("[A-Z]").WithMessage("Must contain uppercase")
            .Matches("[0-9]").WithMessage("Must contain digit");
    }
}

// Program.cs
builder.Services.AddValidatorsFromAssemblyContaining<CreateUserValidator>();
```

См. `AspNetCore/Middle/fluent-validation.md` для deep treatment.

---

## 6. Action Filters / Endpoint Filters

### 6.1. Action Filters (Controllers)

```csharp
public class LoggingFilter : ActionFilterAttribute
{
    public override void OnActionExecuting(ActionExecutingContext context)
    {
        var actionName = context.ActionDescriptor.DisplayName;
        Console.WriteLine($"Executing: {actionName}");
    }
    
    public override void OnActionExecuted(ActionExecutedContext context)
    {
        Console.WriteLine($"Executed: {context.ActionDescriptor.DisplayName}");
    }
}

// Apply
[LoggingFilter]
public class UsersController : ControllerBase
{
    [HttpGet]
    [LoggingFilter]   // на конкретном action
    public IActionResult Get() { }
}

// Globally
builder.Services.AddControllers(opt =>
{
    opt.Filters.Add<LoggingFilter>();
});
```

### 6.2. Filter types

```
- IAuthorizationFilter     — auth check (before everything)
- IResourceFilter           — caching, short-circuit
- IActionFilter            — before/after action
- IExceptionFilter         — handle exceptions
- IResultFilter            — before/after result execution
```

### 6.3. Endpoint Filters (Minimal API .NET 7+)

```csharp
public class TimingFilter : IEndpointFilter
{
    public async ValueTask<object?> InvokeAsync(
        EndpointFilterInvocationContext context, 
        EndpointFilterDelegate next)
    {
        var sw = Stopwatch.StartNew();
        var result = await next(context);
        Console.WriteLine($"Took {sw.ElapsedMilliseconds}ms");
        return result;
    }
}

app.MapGet("/users", () => /* ... */)
    .AddEndpointFilter<TimingFilter>();

// Или group
app.MapGroup("/api")
    .AddEndpointFilter<TimingFilter>();
```

---

## 7. Common pitfalls

### 7.1. ApiController забыли

```csharp
public class UsersController : ControllerBase
{
    [HttpPost]
    public IActionResult Create(CreateUserDto dto)
    {
        // ❌ Без [ApiController] — нужен manual ModelState check
        if (!ModelState.IsValid) return BadRequest(ModelState);
    }
}
```

**Фикс**: добавить `[ApiController]` атрибут.

### 7.2. Mixed routing styles

```csharp
[Route("api/[controller]")]
public class UsersController : ControllerBase
{
    [Route("get/{id}")]   // ❌ Mixed verbs/routes
    public IActionResult Get(int id) { }
    
    [HttpGet("{id}")]   // ✅ Лучше
    public IActionResult Get(int id) { }
}
```

### 7.3. Returning wrong status codes

```csharp
[HttpGet("{id}")]
public IActionResult Get(int id)
{
    var user = _service.GetById(id);
    if (user == null) return BadRequest();   // ❌ должен быть NotFound
    return Ok(user);
}
```

### 7.4. CreatedAtAction wrong route name

```csharp
return CreatedAtAction(nameof(Get), new { id = 1 }, dto);
// Если nameof(Get) не matches действительный route — runtime error
```

### 7.5. Body не binding

```csharp
[HttpPost]
public IActionResult Create([FromQuery] CreateUserDto dto)
{
    // ❌ Complex type не fits в query string
}

// ✅
public IActionResult Create([FromBody] CreateUserDto dto) { }
```

### 7.6. Minimal API DI wrong order

```csharp
app.MapPost("/users", async (
    IUserService service,        // ❌ Service ПОСЛЕ DTO в Minimal
    CreateUserDto dto) => { });

app.MapPost("/users", async (
    CreateUserDto dto,            // ✅ DTO first
    IUserService service) => { });
```

В Minimal API order matters для inference.

### 7.7. Filter применяется после auth

```csharp
// Auth filter должен быть ПЕРЕД business filters
public class CustomAuthFilter : IAuthorizationFilter   // правильный type!
{ }

// Не IActionFilter для auth — слишком поздно в pipeline
```

### 7.8. Не handling cancellation token

```csharp
[HttpGet]
public async Task<IActionResult> Get()
{
    var data = await _service.GetAsync();   // ❌ no CancellationToken
    return Ok(data);
}

// ✅
public async Task<IActionResult> Get(CancellationToken ct)
{
    var data = await _service.GetAsync(ct);
    return Ok(data);
}
```

### 7.9. Versioning без plan

```csharp
[Route("api/v1/users")]   // hardcoded version
[Route("api/v2/users")]   // separate controller
```

**Фикс**: используй `Asp.Versioning` package или MapGroup для proper versioning.

### 7.10. Minimal API без MapGroup

```csharp
// ❌ Все endpoints inline в Program.cs
app.MapGet("/api/users", ...);
app.MapGet("/api/users/{id}", ...);
app.MapPost("/api/users", ...);
// Program.cs становится 1000+ строк
```

**Фикс**: extract в extension methods + MapGroup.

```csharp
// Extensions/UserEndpoints.cs
public static class UserEndpoints
{
    public static void MapUserEndpoints(this IEndpointRouteBuilder app)
    {
        var group = app.MapGroup("/api/users").WithTags("Users");
        
        group.MapGet("/", GetAll);
        group.MapGet("/{id}", GetById);
        group.MapPost("/", Create);
    }
}

// Program.cs
app.MapUserEndpoints();
```

> [!question]- **Интервью: топ-3 ошибки с Controllers/Minimal API?**
> 1) **Forgot [ApiController]** — нужен manual ModelState validation, нет automatic ProblemDetails. Fix: всегда добавлять атрибут на API controllers. 2) **Wrong status codes** — return BadRequest вместо NotFound (404 для missing resource, 400 для validation). 3) **Не handling CancellationToken** — long-running operations не cancellable, blocking thread pool. **Bonus**: Minimal API parameter order — DTO ДО services для proper inference.

---

## 8. Cheat sheet

```csharp
// === Controllers ===
[ApiController]
[Route("api/[controller]")]
public class UsersController(IUserService service) : ControllerBase
{
    [HttpGet]
    public async Task<ActionResult<List<UserDto>>> GetAll(CancellationToken ct)
    {
        var users = await service.GetAllAsync(ct);
        return Ok(users);
    }
    
    [HttpGet("{id:int}")]
    public async Task<ActionResult<UserDto>> Get(int id, CancellationToken ct)
    {
        var user = await service.GetByIdAsync(id, ct);
        return user == null ? NotFound() : Ok(user);
    }
    
    [HttpPost]
    public async Task<ActionResult<UserDto>> Create(CreateUserDto dto, CancellationToken ct)
    {
        var id = await service.CreateAsync(dto, ct);
        return CreatedAtAction(nameof(Get), new { id }, dto);
    }
    
    [HttpPut("{id:int}")]
    public async Task<IActionResult> Update(int id, UpdateUserDto dto, CancellationToken ct)
    {
        await service.UpdateAsync(id, dto, ct);
        return NoContent();
    }
    
    [HttpDelete("{id:int}")]
    public async Task<IActionResult> Delete(int id, CancellationToken ct)
    {
        await service.DeleteAsync(id, ct);
        return NoContent();
    }
}

// === Minimal API ===
var users = app.MapGroup("/api/users")
    .WithTags("Users")
    .RequireAuthorization();

users.MapGet("/", async (IUserService service, CancellationToken ct) =>
    Results.Ok(await service.GetAllAsync(ct)));

users.MapGet("/{id:int}", async (int id, IUserService service, CancellationToken ct) =>
{
    var user = await service.GetByIdAsync(id, ct);
    return user == null ? Results.NotFound() : Results.Ok(user);
});

users.MapPost("/", async (CreateUserDto dto, IUserService service, CancellationToken ct) =>
{
    var id = await service.CreateAsync(dto, ct);
    return Results.Created($"/api/users/{id}", new { id });
});

// === TypedResults (.NET 7+) ===
users.MapGet("/{id:int}", async Task<Results<Ok<UserDto>, NotFound>> 
    (int id, IUserService service, CancellationToken ct) =>
{
    var user = await service.GetByIdAsync(id, ct);
    return user == null ? TypedResults.NotFound() : TypedResults.Ok(user);
});

// === Endpoint filter ===
app.MapPost("/users", ...)
    .AddEndpointFilter<ValidationFilter<CreateUserDto>>();
```

---

## 9. Practice exercises

### 9.1. Refactor controller to Minimal API

Возьми существующий `UsersController` с CRUD endpoints и:
1. Переписать как Minimal API с MapGroup
2. Extract endpoints в `UserEndpoints.cs` extension
3. Добавить TypedResults
4. Сравнить performance (BenchmarkDotNet)

### 9.2. Custom model binder

Реализуй binding для `Pagination` parameter:
```
GET /api/users?paging=10:50  → page=10, size=50
GET /api/users?paging=1:20   → page=1, size=20
```

Используй `TryParse` static method.

### 9.3. Versioning через MapGroup

Реализуй API versioning:
```
/api/v1/users — старая модель
/api/v2/users — новая модель с added fields
/api/users — alias на latest
```

Используй MapGroup и shared service interface.

---

## 10. Что читать дальше

1. **`AspNetCore/Senior/api-design.md`** — API design deep, versioning
2. **`AspNetCore/Senior/pipeline-middleware.md`** — middleware pipeline
3. **`AspNetCore/Middle/aspnet-dependency-injection-deep.md`** — DI deep
4. **`AspNetCore/Middle/aspnet-error-handling.md`** — error handling
5. **`AspNetCore/Middle/fluent-validation.md`** — FluentValidation

---

## 11. См. также

- [[http-fundamentals|AspNetCore/Junior/http-fundamentals]] — HTTP basics
- [[api-design|AspNetCore/Senior/api-design]] — design deep
- [[pipeline-middleware|AspNetCore/Senior/pipeline-middleware]] — middleware
- [[fluent-validation|AspNetCore/Middle/fluent-validation]] — validation
- [[object-mapping|AspNetCore/Middle/object-mapping]] — DTO mapping

---

## 12. Reading list

- **Microsoft Docs — Routing** — learn.microsoft.com/aspnet/core/fundamentals/routing
- **Microsoft Docs — Minimal APIs** — learn.microsoft.com/aspnet/core/fundamentals/minimal-apis
- **Microsoft Docs — Model Binding** — learn.microsoft.com/aspnet/core/mvc/models/model-binding
- **Andrew Lock — Minimal APIs** — andrewlock.net
- **David Fowler — Minimal APIs samples** — github.com/davidfowl
