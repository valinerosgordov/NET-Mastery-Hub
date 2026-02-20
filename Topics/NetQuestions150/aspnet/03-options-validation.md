# Options Pattern и Validation

## Options Pattern

Типизированный доступ к конфигурации — предпочтительный способ работы с настройками в ASP.NET Core. Конфигурационная секция маппится на POCO-класс.

### Регистрация

```csharp
// appsettings.json:
// "Smtp": { "Host": "smtp.example.com", "Port": 587, "UseSsl": true }

public class SmtpOptions
{
    public const string SectionName = "Smtp";
    public string Host { get; set; } = string.Empty;
    public int Port { get; set; } = 587;
    public bool UseSsl { get; set; } = true;
}

// Регистрация
builder.Services.Configure<SmtpOptions>(builder.Configuration.GetSection(SmtpOptions.SectionName));

// Альтернативно — BindConfiguration (.NET 8)
builder.Services.AddOptions<SmtpOptions>().BindConfiguration(SmtpOptions.SectionName);
```

### Три интерфейса для инъекции

| Тип | Lifetime | Поведение | Когда использовать |
|-----|----------|-----------|-------------------|
| `IOptions<T>` | Singleton | Кэширует при первом обращении, **не** обновляется | Статичная конфигурация, startup-настройки |
| `IOptionsSnapshot<T>` | Scoped | Перечитывает значения на каждый HTTP-запрос | Конфигурация с `reloadOnChange: true` |
| `IOptionsMonitor<T>` | Singleton | Обновляется при изменении, имеет `OnChange` callback | Background-сервисы, Feature Flags |

```csharp
// IOptions — самый простой и распространённый
public class EmailService
{
    private readonly SmtpOptions _options;
    public EmailService(IOptions<SmtpOptions> options)
    {
        _options = options.Value; // .Value возвращает T
    }
}

// IOptionsMonitor — для background-сервисов (Singleton), реагирует на изменения
public class HealthCheckService : BackgroundService
{
    private readonly IOptionsMonitor<HealthCheckOptions> _monitor;

    public HealthCheckService(IOptionsMonitor<HealthCheckOptions> monitor)
    {
        _monitor = monitor;
        _monitor.OnChange(opts => Console.WriteLine($"Config changed: {opts.Interval}"));
    }

    protected override async Task ExecuteAsync(CancellationToken ct)
    {
        while (!ct.IsCancellationRequested)
        {
            var current = _monitor.CurrentValue; // Всегда актуальное значение
            await Task.Delay(current.Interval, ct);
        }
    }
}

// IOptionsSnapshot — для Scoped-сервисов, обновляется на каждый запрос
public class OrderController : ControllerBase
{
    public OrderController(IOptionsSnapshot<PricingOptions> options)
    {
        var pricing = options.Value; // Свежее значение на момент запроса
    }
}
```

### Named Options

Несколько наборов настроек одного типа с разными именами:

```csharp
builder.Services.Configure<SmtpOptions>("primary", config.GetSection("Smtp:Primary"));
builder.Services.Configure<SmtpOptions>("backup", config.GetSection("Smtp:Backup"));

// Доступ по имени
public class EmailService
{
    public EmailService(IOptionsSnapshot<SmtpOptions> options)
    {
        var primary = options.Get("primary");
        var backup = options.Get("backup");
    }
}
```

### Валидация Options

```csharp
builder.Services.AddOptions<SmtpOptions>()
    .BindConfiguration("Smtp")
    .ValidateDataAnnotations()           // Проверка [Required], [Range] и т.д.
    .Validate(opts => opts.Port > 0, "Port must be positive") // Inline-валидация
    .ValidateOnStart();                  // Проверка при запуске приложения!
```

**ValidateOnStart** — ошибка конфигурации обнаруживается сразу при старте, а не при первом обращении к `IOptions<T>.Value`. Обязательно для production.

**IValidateOptions<T>** — для сложной валидации с DI:

```csharp
public class SmtpOptionsValidator : IValidateOptions<SmtpOptions>
{
    public ValidateOptionsResult Validate(string? name, SmtpOptions options)
    {
        if (string.IsNullOrEmpty(options.Host))
            return ValidateOptionsResult.Fail("SMTP Host is required");
        if (options.Port is < 1 or > 65535)
            return ValidateOptionsResult.Fail("Port must be between 1 and 65535");
        return ValidateOptionsResult.Success;
    }
}

builder.Services.AddSingleton<IValidateOptions<SmtpOptions>, SmtpOptionsValidator>();
```

### Тонкости Options Pattern

- **Не инжектируйте** `IOptionsSnapshot<T>` в Singleton-сервисы — получите исключение (Scoped в Singleton)
- `IOptions<T>` **не** поддерживает named options — вернёт значение без имени
- `PostConfigure<T>` — модификация значений после `Configure` (например, установка default-значений)
- `IOptionsFactory<T>` — создание экземпляров Options вручную, обход кэширования

---

## Validation

### DataAnnotations

Быстрый способ валидации через атрибуты на модели:

```csharp
public class CreateOrderDto
{
    [Required(ErrorMessage = "Имя обязательно")]
    [StringLength(100, MinimumLength = 2)]
    public string CustomerName { get; set; } = null!;

    [Range(1, 1000)]
    public int Quantity { get; set; }

    [EmailAddress]
    public string? Email { get; set; }

    [RegularExpression(@"^\d{10}$", ErrorMessage = "Телефон — 10 цифр")]
    public string? Phone { get; set; }
}
```

В контроллере `ModelState.IsValid` проверяется автоматически при наличии `[ApiController]`.

### FluentValidation

Отдельный класс-валидатор для сложной логики:

```csharp
public class CreateOrderValidator : AbstractValidator<CreateOrderDto>
{
    public CreateOrderValidator(IProductRepository repo)
    {
        RuleFor(x => x.CustomerName)
            .NotEmpty().WithMessage("Имя обязательно")
            .MaximumLength(100);

        RuleFor(x => x.Quantity)
            .InclusiveBetween(1, 1000);

        RuleFor(x => x.Email)
            .EmailAddress()
            .When(x => x.Email is not null);

        // Асинхронная валидация с обращением к БД
        RuleFor(x => x.ProductId)
            .MustAsync(async (id, ct) => await repo.ExistsAsync(id, ct))
            .WithMessage("Товар не найден");
    }
}
```

### Сравнение подходов

| Аспект | DataAnnotations | FluentValidation |
|--------|----------------|------------------|
| Сложность | Атрибуты на модели | Отдельный класс |
| Кросс-полевая валидация | Ограниченно (IValidatableObject) | Полноценная |
| Асинхронная валидация | Нет | MustAsync |
| DI | Нет | Полная поддержка |
| Тестирование | Через Validator | Простое юнит-тестирование |
| Reuse | Наследование модели | Включение правил (Include) |

### Action Filters и Endpoint Filters

Фильтры — способ внедрить логику до и после выполнения action/endpoint.

```csharp
// Action Filter для MVC
public class ValidationFilter : IAsyncActionFilter
{
    public async Task OnActionExecutionAsync(ActionExecutingContext context, ActionExecutionDelegate next)
    {
        if (!context.ModelState.IsValid)
        {
            context.Result = new BadRequestObjectResult(context.ModelState);
            return; // Short-circuit
        }
        await next();
    }
}

// Регистрация глобально
builder.Services.AddControllers(opts =>
{
    opts.Filters.Add<ValidationFilter>();
});
```

**Endpoint Filter** для Minimal API (.NET 7+):

```csharp
app.MapPost("/orders", CreateOrder)
    .AddEndpointFilter<ValidationFilter<CreateOrderDto>>();
```

### Тонкости Validation

- `[ApiController]` автоматически возвращает `400 BadRequest` при невалидной модели — не нужно проверять `ModelState.IsValid` вручную
- **ProblemDetails** (RFC 7807) — стандартный формат ошибок API, включается через `builder.Services.AddProblemDetails()`
- FluentValidation **не заменяет** бизнес-валидацию — она проверяет формат входных данных, бизнес-правила должны быть в domain layer
- Для Minimal API нет встроенного ModelState — используйте Endpoint Filters с ручным вызовом валидатора
- `[CustomValidation]` — можно создавать свои атрибуты валидации, но FluentValidation обычно удобнее
