---
tags: [aspnet, api, controllers, minimal-api, versioning]
level: Senior
---

# API: Model Binding, Controllers, Versioning

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

## Swagger / OpenAPI

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
