---
tags: [aspnet, api, controllers, minimal-api, versioning]
level: Senior
date: 2026-08-02
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
| Когда | Новые проекты (любой сложности) | Legacy, MVC views, custom model binders |
| DI | Явный (в параметрах) | Через конструктор |
| Группировка | MapGroup() | [Route] атрибут |
| AOT/Trimming | Поддерживает | Ограниченно |

**Рекомендация:** Для новых проектов — **Minimal API**. Для сложной организации — Minimal API + Handler-классы + MapGroup.

---

> [!question]- **Интервью: Minimal API vs Controllers — trade-offs?**
> С .NET 8-10 функциональный паритет почти полный: у Minimal API есть endpoint-фильтры, per-endpoint auth (`RequireAuthorization`), встроенный OpenAPI — «сложный API → Controllers» больше не аргумент. **Minimal API** — меньше boilerplate, быстрее, дружит с Native AOT; организация через MapGroup + IEndpoint pattern. **Controllers** оправданы в legacy-кодовой базе, при MVC views и при кастомном model binding (`IModelBinder`), которого в Minimal API нет.

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

Одна фича = одна папка. Удаление фичи = удаление папки. См. [[architecture-patterns|VSA]].

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
| **Бизнес-правила с DI** | Собственный `IValidator<T>` + MediatR pipeline, или FluentValidation | Валидация с обращением к БД, сервисам, конфигу. Атрибуты этого не умеют. |
| **Domain-инварианты** | Конструкторы Value Objects, guards в entity | Инвариант агрегата (Clean Arch / DDD) — невалидное состояние недопустимо на уровне домена. |

### Почему DataAnnotations упираются в потолок

Атрибут — статическая метадата. В него **нельзя прокинуть зависимости через DI**. Как только нужно проверить уникальность email по базе, валидность купона через сервис или ограничения из `appsettings.json` — атрибутов не хватает.

### Собственный `IValidator<T>` + MediatR behavior

**Работает без внешних библиотек**:

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

### FluentValidation — статус лицензии (проверено 2026-07)

> [!info] **FluentValidation остаётся бесплатной** — Apache 2.0
> В отличие от `FluentAssertions`, `MediatR` и `AutoMapper` v15+ (перешли на коммерческие лицензии), FluentValidation — полностью open-source, без платных тиров; автор лишь просит sponsorship для коммерческих проектов.
>
> **Выбор инструмента:**
>
> | Вариант | Плюсы | Минусы |
> |---------|-------|--------|
> | **Inline через Result pattern / свой IValidator\<T\>** (см. выше) — дефолт этого vault'а | Ноль зависимостей, полный контроль, 100 строк кода | Базовые правила пишутся руками |
> | **FluentValidation** | Декларативные composable-правила, богатые built-in validators, локализация | Лишняя зависимость; business-логика легко «утекает» в validators |
> | **MiniValidation** + DataAnnotations | `MiniValidation.TryValidate(obj)` — атрибуты + `IValidatableObject` одной строкой | Нет DI — только простые сценарии |

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
> Атрибут — статическая метадата, в него нельзя через DI прокинуть сервисы. Проверить «email уникален в базе» или «купон валиден через PricingService» — не получится. Для простых правил (`[Required]`, `[MaxLength]`) — норм, для бизнес-логики — отдельный `IValidator<T>` с DI, регистрируется в MediatR pipeline behavior. FluentValidation делает то же самое декларативно (и остаётся бесплатной, Apache 2.0), но свой 100-строчный валидатор закрывает задачу без внешней зависимости.

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

### OpenAPI в .NET 10

- **OpenAPI 3.1 по умолчанию** (откат: `options.OpenApiVersion = OpenApiSpecVersion.OpenApi3_0`).
- **YAML-вывод** — суффикс маршрута: `app.MapOpenApi("/openapi/{documentName}.yaml")`.
- **XML doc comments** попадают в документ через source generator: `<summary>` / `<param>` / `<response>` из кода становятся summaries/descriptions в OpenAPI без атрибутов. Достаточно `<GenerateDocumentationFile>true</GenerateDocumentationFile>` в .csproj — генератор подключается автоматически с пакетом `Microsoft.AspNetCore.OpenApi`.

### Scalar — популярная замена Swagger UI

```csharp
// NuGet: Scalar.AspNetCore
app.MapOpenApi();
if (app.Environment.IsDevelopment())
{
    app.MapScalarApiReference(); // → /scalar/v1
}
```

Современный UI поверх встроенного OpenAPI-документа: тёмная тема, поиск, генерация примеров запросов (curl, HttpClient, fetch). Частый дефолт для новых .NET 9/10 проектов вместо Swagger UI.

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

### Swashbuckle — проект возрождён

После паузы 2023-2024 у Swashbuckle новая core-команда: v8 вышла в марте 2025, актуальная v10.x поддерживает ASP.NET Core 10 и OpenAPI 3.1 (opt-in). Валидный выбор, если нужен полный Swagger UI-стек из коробки или проект ещё на .NET 8.

```csharp
// NuGet: Swashbuckle.AspNetCore
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
| **Встроенный OpenAPI** | .NET 9+ | Нет сторонних зависимостей, Document Transformers; в .NET 10 — OpenAPI 3.1 + YAML + XML comments | UI нужен отдельно (Scalar / Swagger UI) |
| **Swashbuckle** | .NET 6+ | Swagger UI из коробки, зрелый; активно поддерживается (v10.x, OpenAPI 3.1) | Сторонняя зависимость |
| **NSwag** | Любая | Генерация клиентов (C#, TS) | Сложнее в настройке |

---

## Server-Sent Events (.NET 10)

.NET 10 добавил нативный SSE: `TypedResults.ServerSentEvents` принимает `IAsyncEnumerable<SseItem<T>>` и берёт на себя wire-формат `text/event-stream` (data-строки, event id, retry, cancellation). Раньше SSE собирали руками через `Response.WriteAsync` с ручным контролем buffering.

```csharp
app.MapGet("/stocks", (StockService service, CancellationToken ct) =>
    TypedResults.ServerSentEvents(
        service.StreamPricesAsync(ct),   // IAsyncEnumerable<SseItem<StockPrice>>
        eventType: "priceUpdate"));
```

Это меняет decision tree «SignalR vs SSE vs WebSocket»: для **one-way** стриминга (AI-токены из LLM, live-цены, прогресс long-running задач) SSE теперь first-class в фреймворке — обычный HTTP, проходит через прокси/CDN без апгрейда соединения, авто-reconnect у браузерного `EventSource`. SignalR оправдан, когда нужен **двусторонний** обмен, группы/broadcast или fallback-транспорты. См. [[signalr|SignalR]].

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
