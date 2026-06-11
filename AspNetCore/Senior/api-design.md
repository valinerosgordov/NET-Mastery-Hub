---
tags: [aspnet, api, controllers, minimal-api, versioning]
level: Senior
---

# API: Model Binding, Controllers, Versioning

## Что это, зачем и когда

### Что такое REST API?
Способ общения между приложениями по HTTP. Клиент (браузер, мобилка) отправляет HTTP-запрос → сервер обрабатывает → возвращает JSON-ответ.

**Аналогия — официант:** Клиент (браузер) говорит «дай меню» (GET /menu). Официант (API) идёт на кухню (бизнес-логика), приносит меню (JSON-ответ).

### Minimal API vs Controllers — когда что?

| Критерий | Minimal API | Controllers |
|----------|------------|-------------|
| Объём кода | Мало boilerplate | Больше boilerplate |
| Производительность | Быстрее | Чуть медленнее |
| Когда | Новые проекты, простые API | Legacy, сложные фильтры |
| DI | Явный (в параметрах) | Через конструктор |
| Группировка | MapGroup() | [Route] атрибут |
| AOT/Trimming | Поддерживает | Ограниченно |

**Рекомендация:** Для новых проектов — **Minimal API**. Для сложной организации — Minimal API + Handler-классы + MapGroup.

---

> [!question]- **Интервью: Minimal API vs Controllers — trade-offs?**
> **Minimal API** — легковесные, меньше boilerplate, хорошо для простых CRUD и прототипов. **Controllers** — полная модель MVC, фильтры, model binding, Swagger из коробки. Для сложных API с валидацией, авторизацией на уровне action — Controllers.

## Model Binding

Механизм извлечения данных из HTTP-запроса и маппинга на параметры action-метода или свойства модели.

### Источники данных (по приоритету)

1. **Route values** — `/api/users/{id}` → `id`
2. **Query string** — `?page=1&size=20`
3. **Form data** — `application/x-www-form-urlencoded` или `multipart/form-data`
4. **Body** — JSON/XML (для сложных типов по умолчанию при `[ApiController]`)
5. **Headers** — кастомные заголовки

### Явное указание источника

```csharp
[HttpGet("{id}")]
public IActionResult Get(
    [FromRoute] int id,                    // Из маршрута
    [FromQuery] string? filter,            // Из query string
    [FromHeader(Name = "X-Correlation-Id")] string? correlationId, // Из заголовка
    [FromServices] ILogger<MyController> logger) // Из DI
{
    // ...
}

[HttpPost]
public IActionResult Create(
    [FromBody] CreateOrderDto dto,         // Из тела запроса (JSON)
    [FromForm] IFormFile? attachment)       // Из form data
{
    // ...
}
```

### Custom Model Binder

Для нестандартного маппинга (например, comma-separated values в query string):

```csharp
public class CommaSeparatedModelBinder : IModelBinder
{
    public Task BindModelAsync(ModelBindingContext bindingContext)
    {
        var value = bindingContext.ValueProvider.GetValue(bindingContext.FieldName).FirstValue;
        if (string.IsNullOrEmpty(value))
        {
            bindingContext.Result = ModelBindingResult.Success(Array.Empty<int>());
            return Task.CompletedTask;
        }

        var ids = value.Split(',').Select(int.Parse).ToArray();
        bindingContext.Result = ModelBindingResult.Success(ids);
        return Task.CompletedTask;
    }
}

// Использование
[HttpGet]
public IActionResult Get([ModelBinder(typeof(CommaSeparatedModelBinder))] int[] ids)
{
    // GET /api/items?ids=1,2,3
}
```

### Тонкости Model Binding

- `[ApiController]` автоматически применяет `[FromBody]` к сложным типам и `[FromQuery]` к простым
- `[BindNever]` — исключить свойство из binding (защита от overposting)
- `[BindRequired]` — требует наличие значения (ошибка ModelState, если отсутствует)
- Nullable reference types влияют на required-поведение в .NET 7+
- Body можно прочитать **только один раз** — для повторного чтения нужен `EnableBuffering()`

---

## Controllers

### ControllerBase vs Controller

| Класс | Назначение | View-поддержка |
|-------|-----------|----------------|
| `ControllerBase` | Web API (JSON/XML) | Нет |
| `Controller` | MVC с View | `View()`, `ViewData`, `TempData` |

### Lifecycle контроллера

Контроллер создаётся **на каждый запрос** (Scoped lifetime), dispose после завершения. Все зависимости инжектируются через конструктор.

### Return Types

```csharp
// 1. IActionResult — гибкий, разные типы ответов
[HttpGet("{id}")]
public async Task<IActionResult> GetById(int id)
{
    var product = await _repo.GetByIdAsync(id);
    if (product is null) return NotFound();
    return Ok(product); // 200 + JSON
}

// 2. ActionResult<T> — типизированный, лучше для Swagger
[HttpGet("{id}")]
[ProducesResponseType(typeof(ProductDto), StatusCodes.Status200OK)]
[ProducesResponseType(StatusCodes.Status404NotFound)]
public async Task<ActionResult<ProductDto>> GetById(int id)
{
    var product = await _repo.GetByIdAsync(id);
    if (product is null) return NotFound();
    return product; // Неявное Ok(product)
}

// 3. Конкретный тип — всегда 200 OK
[HttpGet]
public async Task<List<ProductDto>> GetAll() => await _repo.GetAllAsync();
```

### Method Injection

```csharp
// [FromServices] — инъекция только в конкретный action, а не в конструктор
[HttpPost]
public async Task<IActionResult> Create(
    CreateProductDto dto,
    [FromServices] IValidator<CreateProductDto> validator)
{
    var result = await validator.ValidateAsync(dto);
    if (!result.IsValid) return BadRequest(result.Errors);
    // ...
}
```

### ProblemDetails (RFC 7807)

Стандартный формат ошибок API:

```csharp
builder.Services.AddProblemDetails(opts =>
{
    opts.CustomizeProblemDetails = ctx =>
    {
        ctx.ProblemDetails.Extensions["traceId"] = ctx.HttpContext.TraceIdentifier;
    };
});

// Возвращается автоматически при ошибках с [ApiController]:
// {
//   "type": "https://tools.ietf.org/html/rfc7231#section-6.5.1",
//   "title": "Bad Request",
//   "status": 400,
//   "errors": { "Name": ["The Name field is required."] },
//   "traceId": "00-abc..."
// }
```

---

## Minimal API — структура и паттерны

### Минимальный setup

```csharp
var builder = WebApplication.CreateBuilder(args);
builder.Services.AddEndpointsApiExplorer();
builder.Services.AddOpenApi();

var app = builder.Build();
app.MapOpenApi();

app.MapGet("/products/{id:guid}", async (Guid id, IProductRepository repo)
    => await repo.GetByIdAsync(id) is { } p ? Results.Ok(p) : Results.NotFound());

app.Run();
```

Быстрее запускается (меньше ceremony), работает отлично с Native AOT.

### IEndpoint pattern — структурирование Minimal API под Vertical Slice

**Проблема:** через месяц в `Program.cs` валятся 50+ endpoint-ов — никакого vertical slice. Решение — вынести каждый endpoint в свой класс:

```csharp
public interface IEndpoint
{
    void MapEndpoint(IEndpointRouteBuilder app);
}

public sealed class GetProductEndpoint : IEndpoint
{
    public void MapEndpoint(IEndpointRouteBuilder app) =>
        app.MapGet("products/{id:guid}", async (
                Guid id,
                ISender sender,
                CancellationToken ct) =>
            {
                var result = await sender.Send(new GetProductQuery(id), ct);
                return result.Match(Results.Ok, CustomResults.Problem);
            })
            .WithTags("Products")
            .WithName("GetProduct")
            .Produces<ProductResponse>();
}

// Extension — регистрация всех IEndpoint в сборке
public static IEndpointRouteBuilder MapEndpoints(
    this IEndpointRouteBuilder app,
    Assembly? assembly = null)
{
    var endpoints = (assembly ?? Assembly.GetExecutingAssembly())
        .GetTypes()
        .Where(t => typeof(IEndpoint).IsAssignableFrom(t)
                    && t is { IsAbstract: false, IsInterface: false })
        .Select(t => (IEndpoint)Activator.CreateInstance(t)!);

    foreach (var endpoint in endpoints)
        endpoint.MapEndpoint(app);

    return app;
}

// Program.cs
app.MapEndpoints();
```

**Структура папок (Vertical Slice):**

```
Features/
  Products/
    GetProduct/
      GetProductEndpoint.cs
      GetProductQuery.cs
      GetProductHandler.cs
      ProductResponse.cs
    CreateProduct/
      ...
  Orders/
    ...
```

Одна фича = одна папка. Удаление фичи = удаление папки. См. [VSA]().

### Альтернативы IEndpoint

| Пакет | Когда | Примечание |
|-------|-------|------------|
| **Carter** (`ICarterModule`) | Хочешь готовое решение | Добавляет Carter mediator, минимум boilerplate |
| **FastEndpoints** | REPR-паттерн (Request-Endpoint-Processor-Response) | Тяжелее, но больше даёт из коробки |
| **Source generator** | NativeAOT, компайл-тайм регистрация | Убирает reflection при старте |

### Minimal API vs Controllers — когда что

| Сценарий | Выбор |
|----------|-------|
| Простой CRUD, CQRS через MediatR | Minimal API + IEndpoint |
| Native AOT / маленький image | Minimal API |
| Сложный model binding, filters, MVC views | Controllers |
| Большая legacy кодовая база с [Authorize], [ApiController] | Controllers |

> [!question]- **Интервью: Minimal API в больших проектах — не превращается в кашу?**
> Превращается, если всё писать в `Program.cs`. Решение — `IEndpoint` pattern: один класс на endpoint, регистрация через reflection-scan сборки или source generator. Плюс Vertical Slice-структура папок (`Features/Products/GetProduct/*`). На выходе — чище чем контроллеры, потому что фича целиком живёт в одной папке, и нет раздутых `ProductsController` с 15-20 action-ами.

---

## Input Validation

### Три уровня

| Уровень | Инструменты | Когда |
|---------|-------------|-------|
| **Простой (атрибуты)** | `[Required]`, `[MaxLength]`, `[Range]`, `[EmailAddress]`, `[RegularExpression]` | CRUD с базовыми правилами. Видны в Swagger → клиент получает схему. |
| **Бизнес-правила с DI** | Собственный `IValidator<T>` + MediatR pipeline, или OrionGuard | Валидация с обращением к БД, сервисам, конфигу. Атрибуты этого не умеют. |
| **Domain-инварианты** | Конструкторы Value Objects, guards в entity | Инвариант агрегата (Clean Arch / DDD) — невалидное состояние недопустимо на уровне домена. |

### Почему DataAnnotations упираются в потолок

Атрибут — статическая метадата. В него **нельзя прокинуть зависимости через DI**. Как только нужно проверить уникальность email по базе, валидность купона через сервис или ограничения из `appsettings.json` — атрибутов не хватает.

### Собственный IValidator<T> + MediatR behavior

**Работает без платных библиотек**:

```csharp
public interface IValidator<in T>
{
    Task<ValidationResult> ValidateAsync(T instance, CancellationToken ct);
}

public sealed class CreateUserValidator : IValidator<CreateUserCommand>
{
    private readonly IUserRepository _users;
    private readonly IOptions<PasswordPolicy> _policy;

    public CreateUserValidator(IUserRepository users, IOptions<PasswordPolicy> policy)
    {
        _users = users;
        _policy = policy;
    }

    public async Task<ValidationResult> ValidateAsync(
        CreateUserCommand cmd, CancellationToken ct)
    {
        var errors = new List<ValidationError>();

        if (string.IsNullOrWhiteSpace(cmd.Email))
            errors.Add(new("Email", "Required"));
        else if (await _users.ExistsByEmailAsync(cmd.Email, ct))
            errors.Add(new("Email", "Already exists"));

        if (cmd.Password.Length < _policy.Value.MinLength)
            errors.Add(new("Password", $"Min length {_policy.Value.MinLength}"));

        return errors.Count == 0
            ? ValidationResult.Valid
            : ValidationResult.Invalid(errors);
    }
}

// MediatR Pipeline Behavior
public sealed class ValidationBehavior<TRequest, TResponse>(
    IEnumerable<IValidator<TRequest>> validators)
    : IPipelineBehavior<TRequest, TResponse>
    where TResponse : IResult
{
    public async Task<TResponse> Handle(
        TRequest request,
        RequestHandlerDelegate<TResponse> next,
        CancellationToken ct)
    {
        foreach (var v in validators)
        {
            var result = await v.ValidateAsync(request, ct);
            if (result.IsInvalid)
                return (TResponse)Result.Fail(result.ToError());
        }

        return await next(ct);
    }
}
```

### FluentValidation — статус в 2026

> [!warning] **FluentValidation с 2026 — платная для коммерческого использования**
> Автор перешёл на коммерческую лицензионную модель (так же как `FluentAssertions`). Для OSS и pet-проектов — условно-бесплатно, для production — лицензия.
>
> **Альтернативы:**
>
> | Вариант | Плюсы | Минусы |
> |---------|-------|--------|
> | **Свой IValidator\<T\>** (см. выше) | Полный контроль, OSS, 100 строк кода | Нужно писать базовые правила руками |
> | **OrionGuard** | Совместим с FluentValidation API → миграция regex-ом, source-generator для NativeAOT | Молодая, маленькое комьюнити — для institutional-проектов риск |
> | **MiniValidation** + DataAnnotations | `MiniValidation.TryValidate(obj)` — атрибуты + `IValidatableObject` одной строкой | Нет DI — только простые сценарии |
> | **Фикс версии FluentValidation до смены лицензии** | Работает как раньше | Не получаешь security-патчи, compliance-серая зона |

### Валидация в Minimal API

```csharp
app.MapPost("/users", async (
    CreateUserCommand cmd,
    IValidator<CreateUserCommand> validator,
    ISender sender,
    CancellationToken ct) =>
{
    var validation = await validator.ValidateAsync(cmd, ct);
    if (validation.IsInvalid)
        return Results.ValidationProblem(validation.ToDictionary());

    var result = await sender.Send(cmd, ct);
    return result.Match(u => Results.Created($"/users/{u.Id}", u), CustomResults.Problem);
});
```

Или через pipeline behavior — тогда валидация не дублируется в endpoint-ах.

> [!question]- **Интервью: Чем DataAnnotations плохи для сложной валидации?**
> Атрибут — статическая метадата, в него нельзя через DI прокинуть сервисы. Проверить «email уникален в базе» или «купон валиден через PricingService» — не получится. Для простых правил (`[Required]`, `[MaxLength]`) — норм, для бизнес-логики — отдельный `IValidator<T>` с DI, регистрируется в MediatR pipeline behavior. FluentValidation делал именно это, но с 2026 стал платным для коммерческого использования → либо OrionGuard (совместимый API), либо свой 100-строчный валидатор.

---

## API Versioning

Управление эволюцией API без поломки существующих клиентов.

```csharp
// NuGet: Asp.Versioning.Mvc
builder.Services.AddApiVersioning(opts =>
{
    opts.DefaultApiVersion = new ApiVersion(1, 0);
    opts.AssumeDefaultVersionWhenUnspecified = true;
    opts.ReportApiVersions = true; // Заголовок api-supported-versions в ответе

    // Стратегия передачи версии
    opts.ApiVersionReader = ApiVersionReader.Combine(
        new UrlSegmentApiVersionReader(),          // /api/v1/products
        new QueryStringApiVersionReader("v"),       // ?v=1.0
        new HeaderApiVersionReader("X-Api-Version") // Header
    );
}).AddApiExplorer();
```

```csharp
[ApiController]
[ApiVersion("1.0")]
[ApiVersion("2.0")]
[Route("api/v{version:apiVersion}/products")]
public class ProductsController : ControllerBase
{
    [HttpGet]
    [MapToApiVersion("1.0")]
    public IActionResult GetV1() => Ok(new { format = "v1" });

    [HttpGet]
    [MapToApiVersion("2.0")]
    public IActionResult GetV2() => Ok(new { format = "v2", extra = "data" });
}
```

### Стратегии версионирования

| Стратегия | Пример | Плюсы | Минусы |
|-----------|--------|-------|--------|
| URL path | `/api/v2/products` | Наглядно, кэшируемо | Дублирование маршрутов |
| Query string | `?api-version=2.0` | Без изменения URL | Менее очевидно |
| Header | `X-Api-Version: 2.0` | Чистые URL | Сложнее тестировать в браузере |
| Media type | `Accept: application/json;v=2` | RESTful | Сложно реализовать |

---

## OpenAPI

### Встроенный OpenAPI (.NET 9+) — рекомендуемый подход

.NET 9 убрал зависимость от Swashbuckle. OpenAPI генерация встроена в ASP.NET Core.

```csharp
// Регистрация — без сторонних пакетов!
builder.Services.AddOpenApi();

var app = builder.Build();

// Endpoint для OpenAPI документа (JSON)
app.MapOpenApi(); // → /openapi/v1.json

// Для Swagger UI — отдельный пакет (только dev)
if (app.Environment.IsDevelopment())
{
    app.UseSwaggerUI(options =>
    {
        options.SwaggerEndpoint("/openapi/v1.json", "My API v1");
    });
    // NuGet: Swashbuckle.AspNetCore.SwaggerUI (только UI, без генератора)
}
```

### Кастомизация OpenAPI документа

```csharp
builder.Services.AddOpenApi(options =>
{
    options.AddDocumentTransformer((document, context, ct) =>
    {
        document.Info = new OpenApiInfo
        {
            Title = "My API",
            Version = "v1",
            Description = "Production API"
        };
        return Task.CompletedTask;
    });

    // JWT Bearer схема
    options.AddDocumentTransformer<BearerSecuritySchemeTransformer>();
});

// Security scheme transformer
public sealed class BearerSecuritySchemeTransformer : IOpenApiDocumentTransformer
{
    public Task TransformAsync(OpenApiDocument document,
        OpenApiDocumentTransformerContext context, CancellationToken ct)
    {
        document.Components ??= new OpenApiComponents();
        document.Components.SecuritySchemes["Bearer"] = new OpenApiSecurityScheme
        {
            Type = SecuritySchemeType.Http,
            Scheme = "bearer",
            BearerFormat = "JWT"
        };
        return Task.CompletedTask;
    }
}
```

### TypedResults → точная OpenAPI схема

```csharp
// TypedResults автоматически генерируют корректную OpenAPI схему
app.MapGet("/orders/{id}", async (Guid id, GetOrderHandler handler, CancellationToken ct)
    : Results<Ok<OrderDto>, NotFound, ProblemHttpResult> =>
{
    var result = await handler.HandleAsync(id, ct);
    return result.Match<Results<Ok<OrderDto>, NotFound, ProblemHttpResult>>(
        order => TypedResults.Ok(order),
        error => error.Type == ErrorType.NotFound
            ? TypedResults.NotFound()
            : TypedResults.Problem(error.Message, statusCode: 400));
});
// OpenAPI: 200 → OrderDto, 404 → empty, 400 → ProblemDetails
```

### Swashbuckle (legacy, до .NET 9)

```csharp
// NuGet: Swashbuckle.AspNetCore (для .NET 8 и старше)
builder.Services.AddEndpointsApiExplorer();
builder.Services.AddSwaggerGen(opts =>
{
    opts.SwaggerDoc("v1", new OpenApiInfo { Title = "My API", Version = "v1" });

    // XML-комментарии
    var xmlFile = $"{Assembly.GetExecutingAssembly().GetName().Name}.xml";
    opts.IncludeXmlComments(Path.Combine(AppContext.BaseDirectory, xmlFile));

    // JWT Bearer в Swagger UI
    opts.AddSecurityDefinition("Bearer", new OpenApiSecurityScheme
    {
        Type = SecuritySchemeType.Http,
        Scheme = "bearer",
        BearerFormat = "JWT"
    });
});

// В .csproj: <GenerateDocumentationFile>true</GenerateDocumentationFile>
```

| Подход | Версия | Плюсы | Минусы |
|--------|--------|-------|--------|
| **Встроенный OpenAPI** | .NET 9+ | Нет сторонних зависимостей, Document Transformers | Swagger UI нужен отдельно |
| **Swashbuckle** | .NET 6-8 | Swagger UI из коробки, зрелый | Стороння зависимость, не обновляется |
| **NSwag** | Любая | Генерация клиентов (C#, TS) | Сложнее в настройке |

---

## Работа с файлами

```csharp
// Отдача файла
[HttpGet("{id}/download")]
public IActionResult Download(int id)
{
    var stream = _storage.OpenRead(id);
    return File(stream, "application/pdf", "report.pdf");
    // Или: PhysicalFile("/path/to/file.pdf", "application/pdf");
}

// Приём файла
[HttpPost("upload")]
[RequestSizeLimit(50 * 1024 * 1024)] // 50 MB
public async Task<IActionResult> Upload(IFormFile file)
{
    if (file.Length == 0) return BadRequest("Empty file");
    if (file.Length > 10_000_000) return BadRequest("Too large");

    // Проверка типа файла
    var allowedTypes = new[] { "image/jpeg", "image/png", "application/pdf" };
    if (!allowedTypes.Contains(file.ContentType))
        return BadRequest("Invalid file type");

    using var stream = file.OpenReadStream();
    await _storage.SaveAsync(stream, file.FileName);
    return Ok();
}
```

### Тонкости работы с файлами

- `IFormFile` буферизирует в memory до `MultipartBodyLengthLimit` (по умолчанию 128 MB)
- Для больших файлов — streaming через `Request.Body` без `IFormFile`
- **Никогда** не доверяйте `file.FileName` от клиента — генерируйте имя на сервере
- `ContentDisposition: inline` vs `attachment` — показать в браузере или скачать

---

## Deploy

| Способ | Описание |
|--------|----------|
| **Framework-dependent** | Требует установленного .NET runtime на сервере. Меньший размер |
| **Self-contained** | Включает runtime. Больший размер, но независимость от среды |
| **Single-file** | Один исполняемый файл (PublishSingleFile) |
| **Docker** | `FROM mcr.microsoft.com/dotnet/aspnet:8.0` |
| **IIS** | ASP.NET Core Module (in-process или out-of-process) |
| **Azure App Service** | PaaS, автоматическое масштабирование |
| **Kestrel + reverse proxy** | nginx/Caddy перед Kestrel для TLS termination, load balancing |
