# Pipeline, Middleware и Routing

## Pipeline и Middleware

Pipeline (конвейер) — цепочка компонентов (middleware), через которую проходит каждый HTTP-запрос. Каждый middleware получает `HttpContext` и решает: вызвать `next()` для передачи дальше или завершить обработку (short-circuit). Запрос проходит «вниз» до endpoint, ответ — «вверх» обратно через ту же цепочку.

```
Request → M1 → M2 → M3 → Endpoint → M3 → M2 → M1 → Response
```

### Порядок middleware

Порядок регистрации в `Program.cs` **критически важен** — это порядок выполнения. Типичный порядок:

```
Exception handling → HTTPS redirect → HSTS → CORS → Static files →
Routing → Authentication → Authorization → Endpoint
```

Почему порядок важен:
- `UseAuthentication()` **до** `UseAuthorization()` — иначе авторизация не увидит identity пользователя
- `UseExceptionHandler()` **первым** — чтобы перехватывать ошибки из всех последующих middleware
- `UseStaticFiles()` **до** Routing — статические файлы отдаются без прохождения аутентификации (обычно это нормально, но для защищённых файлов нужен другой подход)
- `UseCors()` **до** Routing — CORS-заголовки должны добавляться даже к preflight-запросам

### Short-circuit

Short-circuit — прерывание цепочки, когда middleware не вызывает `next()`. Запрос не доходит до endpoint, ответ возвращается сразу.

Примеры:
- Health check endpoint — мгновенный ответ `200 OK`
- Статические файлы — отдача из файловой системы
- Rate limiting — ответ `429 Too Many Requests`
- Кастомная проверка API key — `401 Unauthorized`

### Создание middleware

**1. Inline (lambda)** — для простой логики:

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

**2. Класс с `InvokeAsync`** — для сложной логики с DI:

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
        _logger.LogInformation("Request {Path} took {Ms}ms", context.Request.Path, sw.ElapsedMilliseconds);
    }
}
```

**3. Extension method** — удобная регистрация:

```csharp
public static class MiddlewareExtensions
{
    public static IApplicationBuilder UseRequestTiming(this IApplicationBuilder app)
        => app.UseMiddleware<RequestTimingMiddleware>();
}

// Использование: app.UseRequestTiming();
```

### Тонкости и нюансы

- Middleware-класс имеет **Singleton lifetime** — конструктор вызывается один раз. Не храните request-specific данные в полях класса
- Scoped-сервисы инжектируются **через параметры `InvokeAsync`**, не через конструктор
- `app.Run(...)` — терминальный middleware, всегда выполняет short-circuit (нет `next`)
- `app.Map("/path", ...)` — branch pipeline на отдельный путь (fork)
- `app.UseWhen(predicate, ...)` — условный branch, но запрос возвращается в основную цепочку
- `app.MapWhen(predicate, ...)` — условный branch с полным отделением

---

## Routing

Routing сопоставляет входящий HTTP-запрос с конкретным endpoint'ом. С .NET Core 3.0 используется Endpoint Routing — двухфазный процесс.

### Endpoint Routing

```csharp
app.UseRouting();          // Фаза 1: Matching — выбор endpoint по URL и HTTP-методу
// Middleware между Routing и Endpoints имеют доступ к выбранному endpoint
app.UseAuthentication();   // Знает, какой endpoint будет вызван
app.UseAuthorization();    // Может проверить [Authorize] до вызова endpoint
// app.UseEndpoints(...)   // Фаза 2: Dispatch — вызов endpoint (неявно в .NET 6+)
```

Между `UseRouting()` и dispatch можно читать metadata endpoint'а — например, `[Authorize]` атрибуты.

### Конвенции маршрутизации

| Тип | Пример | Применение |
|-----|--------|------------|
| Conventional | `{controller}/{action}/{id?}` | MVC с View |
| Attribute | `[Route("api/[controller]")]` | Web API |
| Minimal API | `app.MapGet("/users/{id}", ...)` | Minimal API |

### Параметры маршрутов

```csharp
// Обязательный параметр
app.MapGet("/users/{id}", (int id) => ...);

// Опциональный параметр
app.MapGet("/users/{id?}", (int? id) => ...);

// Constraint — проверка типа/формата
app.MapGet("/users/{id:int}", (int id) => ...);
app.MapGet("/orders/{date:datetime}", (DateTime date) => ...);
app.MapGet("/files/{name:regex(^[a-z]+$)}", (string name) => ...);

// Catch-all — всё после префикса
app.MapGet("/files/{*path}", (string path) => ...);

// Составные constraints
app.MapGet("/page/{num:int:min(1):max(100)}", (int num) => ...);
```

### Тонкости и нюансы Routing

- **Route precedence**: literal > parameter > catch-all. Более конкретные маршруты имеют приоритет
- **Ambient values**: в MVC значения текущего маршрута повторно используются при генерации URL
- **LinkGenerator** — сервис для генерации URL вне контроллеров
- Constraint `{id:int}` **не заменяет** валидацию — это только matching. Значение `0` или `-1` пройдёт constraint
- Custom constraints: реализация `IRouteConstraint` для бизнес-логики маршрутизации

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

Легковесные эндпоинты без контроллеров. Быстрее за счёт меньших абстракций (нет MVC filter pipeline по умолчанию).

```csharp
var app = builder.Build();

// Группировка
var users = app.MapGroup("/api/users")
    .RequireAuthorization()
    .WithTags("Users");

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

**Endpoint filters** (.NET 7+) — аналог Action Filters для Minimal API:

```csharp
app.MapGet("/orders", GetOrders)
    .AddEndpointFilter(async (context, next) =>
    {
        // Логика до handler'а
        var result = await next(context);
        // Логика после handler'а
        return result;
    });
```

### Тонкости Minimal API

- DI через параметры handler'а — автоматическое разрешение зарегистрированных сервисов
- `TypedResults` (.NET 7+) вместо `Results` — сильная типизация для OpenAPI
- Группы (`MapGroup`) поддерживают вложенность и общие фильтры/метаданные
- Для большого API — выносить handler'ы в статические методы или отдельные классы
- Не поддерживают `ModelState` — используйте Endpoint Filters + FluentValidation
