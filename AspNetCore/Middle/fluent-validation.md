---
tags: [aspnetcore, validation, fluent-validation, data-annotations, mediatr, middle]
level: Middle
date: 2026-08-02
---

# FluentValidation — валидация запросов в ASP.NET Core

> **Стандарт для validation в .NET 2026.** vs DataAnnotations: гибче, тестируемо, async-friendly. Closes пробел "пишу валидацию через `[Required]` атрибуты — но complex rules превращаются в bullshit".

---

## Что это, зачем и когда

### Проблема DataAnnotations

```csharp
public class CreateUserRequest
{
    [Required]
    [EmailAddress]
    [StringLength(100)]
    public string Email { get; set; }

    [Required]
    [StringLength(100, MinimumLength = 8)]
    [RegularExpression(@"^(?=.*[A-Z])(?=.*[0-9]).+$")]  // нечитаемо
    public string Password { get; set; }

    [Required]
    [Compare(nameof(Password))]
    public string ConfirmPassword { get; set; }
}
```

Минусы:
- ❌ Нельзя async (DB lookup для уникальности email)
- ❌ Cross-field rules неудобны (`[Compare]` ограничен)
- ❌ Атрибуты загромождают DTO
- ❌ Сложно testить отдельно от controller
- ❌ Conditional rules невозможны

### Решение — FluentValidation

```csharp
public class CreateUserRequestValidator : AbstractValidator<CreateUserRequest>
{
    public CreateUserRequestValidator(IUserRepository repo)
    {
        RuleFor(x => x.Email)
            .NotEmpty()
            .EmailAddress()
            .MaximumLength(100)
            .MustAsync(async (email, ct) => !await repo.EmailExistsAsync(email, ct))
                .WithMessage("Email already registered");

        RuleFor(x => x.Password)
            .NotEmpty()
            .MinimumLength(8)
            .Matches("[A-Z]").WithMessage("Password must contain uppercase letter")
            .Matches("[0-9]").WithMessage("Password must contain digit");

        RuleFor(x => x.ConfirmPassword)
            .Equal(x => x.Password).WithMessage("Passwords do not match");
    }
}
```

✅ DTO — pure data. Validation отдельно. Tests возможны без HTTP context.

> [!info] Лицензия (проверено 2026-07)
> FluentValidation — **по-прежнему бесплатна**: Apache 2.0, без платных тиров; автор лишь просит sponsorship для коммерческих проектов. Не путать с `FluentAssertions`, `MediatR` и `AutoMapper` v15+ — те перешли на коммерческие лицензии. Альтернатива без внешней зависимости — inline-валидация через Result pattern / свой `IValidator<T>` (см. [[api-design|API Design]]).

---

## 1. Установка и setup

```bash
dotnet add package FluentValidation
dotnet add package FluentValidation.DependencyInjectionExtensions
```

> [!warning] FluentValidation.AspNetCore — deprecated
> Пакет `FluentValidation.AspNetCore` официально deprecated, репозиторий в архиве. Авто-валидация (`AddFluentValidationAutoValidation`) не рекомендована самим автором библиотеки: она **синхронная** (async-правила кидают исключение в runtime), привязана к MVC model binding и **не работает с Minimal API**. Современный подход — явный вызов валидатора или endpoint-фильтр.

```csharp
// Program.cs — регистрация всех validators из assembly
builder.Services.AddValidatorsFromAssemblyContaining<CreateUserRequestValidator>();
```

### Явный вызов в endpoint

```csharp
app.MapPost("/users", async (
    CreateUserRequest request,
    IValidator<CreateUserRequest> validator,
    IUserService users,
    CancellationToken ct) =>
{
    var validation = await validator.ValidateAsync(request, ct);
    if (!validation.IsValid)
        return Results.ValidationProblem(validation.ToDictionary());

    var user = await users.CreateAsync(request, ct);
    return Results.Created($"/users/{user.Id}", user);
});
```

### Endpoint-фильтр — validation как cross-cutting concern

```csharp
public sealed class ValidationFilter<T> : IEndpointFilter where T : class
{
    public async ValueTask<object?> InvokeAsync(
        EndpointFilterInvocationContext context,
        EndpointFilterDelegate next)
    {
        if (context.HttpContext.RequestServices.GetService<IValidator<T>>()
                is { } validator &&
            context.Arguments.OfType<T>().FirstOrDefault() is { } argument)
        {
            var result = await validator.ValidateAsync(
                argument, context.HttpContext.RequestAborted);
            if (!result.IsValid)
                return Results.ValidationProblem(result.ToDictionary());
        }

        return await next(context);
    }
}

// Use — фильтр вешается на endpoint или на всю группу
app.MapPost("/users", CreateUser)
    .AddEndpointFilter<ValidationFilter<CreateUserRequest>>();
```

### Альтернатива из коробки: встроенная валидация Minimal API (.NET 10)

.NET 10 добавил нативную валидацию для Minimal API — пакет `Microsoft.Extensions.Validation`. Работает через Roslyn source generator: на этапе компиляции обходит граф типов параметров endpoint'ов и генерирует валидационный код без runtime-reflection. Правила — стандартные DataAnnotations-атрибуты.

```csharp
// Program.cs
builder.Services.AddValidation();

public record CreateProductRequest(
    [property: Required, MaxLength(200)] string Name,
    [property: Range(0.01, 1_000_000)] decimal Price);

app.MapPost("/products", (CreateProductRequest request) => /* ... */);
// Невалидный запрос → автоматический 400 с ProblemDetails (errors по полям)
```

```xml
<!-- .csproj — включить source-generated интерцепторы -->
<PropertyGroup>
  <InterceptorsNamespaces>$(InterceptorsNamespaces);Microsoft.AspNetCore.Http.Validation.Generated</InterceptorsNamespaces>
</PropertyGroup>
```

Что умеет: DataAnnotations-атрибуты, `IValidatableObject`, кастомные `ValidationAttribute`, вложенные объекты и коллекции. Чего не умеет: async-правила (DB lookup), DI-зависимости в правилах, conditional-цепочки `When`/`Unless` — за этим по-прежнему к FluentValidation.

### Базовый Validator

```csharp
public class ProductValidator : AbstractValidator<Product>
{
    public ProductValidator()
    {
        RuleFor(p => p.Name).NotEmpty().MaximumLength(200);
        RuleFor(p => p.Price).GreaterThan(0).LessThan(1_000_000);
        RuleFor(p => p.Stock).GreaterThanOrEqualTo(0);
        RuleFor(p => p.Sku).Matches(@"^[A-Z0-9-]+$");
    }
}
```

---

## 2. Built-in validators

```csharp
// Базовые
.NotNull() / .NotEmpty()
.Equal(value) / .NotEqual(value)
.GreaterThan(x) / .LessThan(x) / .InclusiveBetween(min, max)
.Length(exact) / .Length(min, max)
.MaximumLength(n) / .MinimumLength(n)

// Strings
.EmailAddress()                   // RFC 5321
.Matches(regex)
.CreditCard()                      // Luhn algorithm

// Numbers
.GreaterThanOrEqualTo(x)
.PrecisionScale(precision, scale, ignoreTrailing)  // для decimal

// Collections
.NotEmpty()  // длина > 0
.Must(list => list.Count <= 10)

// Custom
.Must(predicate)
.MustAsync(async predicate)
```

---

## 3. Async validators

Главный аргумент против DataAnnotations.

```csharp
public class RegisterValidator : AbstractValidator<RegisterRequest>
{
    public RegisterValidator(IUserRepository repo)
    {
        RuleFor(x => x.Email)
            .NotEmpty()
            .EmailAddress()
            .MustAsync(async (email, ct) => 
                !await repo.EmailExistsAsync(email, ct))
            .WithMessage("Email уже зарегистрирован");

        RuleFor(x => x.PromoCode)
            .MustAsync(async (code, ct) => 
                await repo.IsPromoCodeValidAsync(code, ct))
            .When(x => !string.IsNullOrEmpty(x.PromoCode));
    }
}
```

> [!warning] Performance — async validators делают DB query
> Сделай их **последними**. Сначала cheap synchronous checks (email format), потом DB lookup.

---

## 4. Cross-field validation

```csharp
public class TransferValidator : AbstractValidator<MoneyTransfer>
{
    public TransferValidator()
    {
        // Поля по отдельности
        RuleFor(x => x.FromAccount).NotEmpty();
        RuleFor(x => x.ToAccount).NotEmpty();
        RuleFor(x => x.Amount).GreaterThan(0);

        // Между полями
        RuleFor(x => x.ToAccount)
            .NotEqual(x => x.FromAccount)
            .WithMessage("Cannot transfer to same account");

        // Валидация на уровне всего объекта
        RuleFor(x => x).Must(t => 
            t.Currency == "USD" || t.Amount <= 100_000)
            .WithMessage("Non-USD transfers limited to 100K");
    }
}
```

---

## 5. Conditional rules — When / Unless

```csharp
public class OrderValidator : AbstractValidator<Order>
{
    public OrderValidator()
    {
        RuleFor(x => x.ShippingAddress)
            .NotEmpty()
            .When(x => x.RequiresShipping);  // только если physical product

        RuleFor(x => x.GiftMessage)
            .MaximumLength(500)
            .Unless(x => string.IsNullOrEmpty(x.GiftMessage));

        // Group of rules
        When(x => x.PaymentMethod == "CreditCard", () =>
        {
            RuleFor(x => x.CardNumber).NotEmpty().CreditCard();
            RuleFor(x => x.CVV).Matches(@"^\d{3,4}$");
            RuleFor(x => x.ExpiryDate).Must(BeFutureDate);
        });
    }

    private bool BeFutureDate(string expiry) => /* ... */;
}
```

---

## 6. Custom validators

### Method-based

```csharp
public class UserValidator : AbstractValidator<User>
{
    public UserValidator()
    {
        RuleFor(x => x.Username).Must(BeValidUsername)
            .WithMessage("Username can only contain letters, digits, underscores");

        RuleFor(x => x.Birthday).Must(BeAdult)
            .WithMessage("Must be at least 18 years old");
    }

    private bool BeValidUsername(string s) => 
        Regex.IsMatch(s, @"^\w+$");

    private bool BeAdult(DateTime birthday) => 
        DateTime.UtcNow.Year - birthday.Year >= 18;
}
```

### Reusable — extension method

```csharp
public static class CustomValidators
{
    public static IRuleBuilderOptions<T, string> ValidPhoneNumber<T>(
        this IRuleBuilder<T, string> rule) =>
        rule.Matches(@"^\+?[1-9]\d{1,14}$")
            .WithMessage("Invalid phone format (E.164)");

    public static IRuleBuilderOptions<T, string> CyrillicOnly<T>(
        this IRuleBuilder<T, string> rule) =>
        rule.Matches(@"^[А-Яа-я\s-]+$")
            .WithMessage("Только кириллица");
}

// Use
RuleFor(x => x.Phone).ValidPhoneNumber();
RuleFor(x => x.NameRu).CyrillicOnly();
```

---

## 7. Integration с MediatR pipeline

Запускать validation **в pipeline** перед handler:

```csharp
public class ValidationBehavior<TRequest, TResponse> : IPipelineBehavior<TRequest, TResponse>
    where TRequest : IRequest<TResponse>
{
    private readonly IEnumerable<IValidator<TRequest>> _validators;

    public ValidationBehavior(IEnumerable<IValidator<TRequest>> validators) =>
        _validators = validators;

    public async Task<TResponse> Handle(
        TRequest request,
        RequestHandlerDelegate<TResponse> next,
        CancellationToken ct)
    {
        if (!_validators.Any()) return await next();

        var context = new ValidationContext<TRequest>(request);
        var failures = (await Task.WhenAll(_validators.Select(v => v.ValidateAsync(context, ct))))
            .SelectMany(r => r.Errors)
            .Where(e => e != null)
            .ToList();

        if (failures.Any())
            throw new ValidationException(failures);

        return await next();
    }
}

// Register
services.AddScoped(typeof(IPipelineBehavior<,>), typeof(ValidationBehavior<,>));
services.AddValidatorsFromAssemblyContaining<CreateUserHandler>();
```

См. [[cqrs-mediatr|CQRS & MediatR]].

---

## 8. Case Study #1 — Registration form

**Сценарий:** Multi-field validation: email уникальность, password strength, age check.

```csharp
public record RegisterRequest(
    string Email,
    string Password,
    string ConfirmPassword,
    DateTime BirthDate,
    bool AcceptTerms);

public class RegisterValidator : AbstractValidator<RegisterRequest>
{
    public RegisterValidator(IUserRepository repo)
    {
        RuleFor(x => x.Email)
            .NotEmpty().WithMessage("Email обязателен")
            .EmailAddress().WithMessage("Невалидный email")
            .MaximumLength(200)
            .MustAsync(async (email, ct) => !await repo.EmailExistsAsync(email, ct))
                .WithMessage("Email уже зарегистрирован");

        RuleFor(x => x.Password)
            .NotEmpty()
            .MinimumLength(8).WithMessage("Минимум 8 символов")
            .Matches("[A-Z]").WithMessage("Должна быть заглавная буква")
            .Matches("[a-z]").WithMessage("Должна быть строчная буква")
            .Matches("[0-9]").WithMessage("Должна быть цифра")
            .Matches(@"[!@#$%^&*]").WithMessage("Должен быть спецсимвол");

        RuleFor(x => x.ConfirmPassword)
            .Equal(x => x.Password).WithMessage("Пароли не совпадают");

        RuleFor(x => x.BirthDate)
            .NotEmpty()
            .Must(date => DateTime.UtcNow.Year - date.Year >= 18)
                .WithMessage("Должно быть 18+");

        RuleFor(x => x.AcceptTerms)
            .Equal(true).WithMessage("Необходимо принять условия");
    }
}
```

---

## 9. Case Study #2 — Multi-step wizard

**Сценарий:** 3-step checkout. Different fields обязательны на разных шагах.

```csharp
public class CheckoutRequest
{
    public int Step { get; set; }
    
    // Step 1
    public string? ShippingAddress { get; set; }
    public string? City { get; set; }
    
    // Step 2
    public string? PaymentMethod { get; set; }
    public string? CardNumber { get; set; }
    public string? CVV { get; set; }
    
    // Step 3
    public bool ConfirmOrder { get; set; }
}

public class CheckoutValidator : AbstractValidator<CheckoutRequest>
{
    public CheckoutValidator()
    {
        // Step 1
        When(x => x.Step >= 1, () =>
        {
            RuleFor(x => x.ShippingAddress).NotEmpty().MinimumLength(5);
            RuleFor(x => x.City).NotEmpty();
        });

        // Step 2
        When(x => x.Step >= 2, () =>
        {
            RuleFor(x => x.PaymentMethod).NotEmpty().Must(m => m is "Card" or "PayPal");

            When(x => x.PaymentMethod == "Card", () =>
            {
                RuleFor(x => x.CardNumber).NotEmpty().CreditCard();
                RuleFor(x => x.CVV).Matches(@"^\d{3,4}$");
            });
        });

        // Step 3 — final confirmation
        When(x => x.Step == 3, () =>
        {
            RuleFor(x => x.ConfirmOrder).Equal(true)
                .WithMessage("Подтвердите заказ");
        });
    }
}
```

---

## 10. Case Study #3 — Конфигурируемые правила

**Сценарий:** SaaS multi-tenant. Каждый tenant имеет свои rules (max length, allowed values).

```csharp
public class DynamicProductValidator : AbstractValidator<Product>
{
    public DynamicProductValidator(ITenantSettingsService settings)
    {
        var current = settings.GetCurrent();

        RuleFor(p => p.Name)
            .NotEmpty()
            .MaximumLength(current.MaxNameLength);  // tenant-specific

        RuleFor(p => p.Category)
            .Must(c => current.AllowedCategories.Contains(c))
            .WithMessage($"Допустимые категории: {string.Join(", ", current.AllowedCategories)}");

        RuleFor(p => p.Price)
            .GreaterThan(current.MinPrice)
            .LessThan(current.MaxPrice);

        if (current.RequireSku)
        {
            RuleFor(p => p.Sku).NotEmpty().Matches(current.SkuPattern);
        }
    }
}
```

---

## 11. Case Study #4 — Тестирование validators

```csharp
public class CreateUserValidatorTests
{
    private readonly Mock<IUserRepository> _repo = new();
    private readonly CreateUserValidator _validator;

    public CreateUserValidatorTests()
    {
        _repo.Setup(r => r.EmailExistsAsync(It.IsAny<string>(), default))
            .ReturnsAsync(false);
        _validator = new CreateUserValidator(_repo.Object);
    }

    [Theory]
    [InlineData("", "Email обязателен")]
    [InlineData("not-email", "Невалидный email")]
    [InlineData("a@b.c", null)]  // valid
    public async Task Email_validation(string email, string? expectedError)
    {
        var result = await _validator.ValidateAsync(
            new CreateUserRequest { Email = email, Password = "ValidPass1!" });

        if (expectedError == null)
            result.IsValid.Should().BeTrue();
        else
            result.Errors.Should().Contain(e => e.ErrorMessage == expectedError);
    }

    [Fact]
    public async Task Email_already_exists()
    {
        _repo.Setup(r => r.EmailExistsAsync("taken@x.com", default))
            .ReturnsAsync(true);

        var result = await _validator.TestValidateAsync(  // FluentValidation.TestHelper
            new CreateUserRequest { Email = "taken@x.com", Password = "ValidPass1!" });

        result.ShouldHaveValidationErrorFor(x => x.Email)
            .WithErrorMessage("Email уже зарегистрирован");
    }
}
```

См. [[testing-fundamentals|Testing Fundamentals]].

---

## 12. Case Study #5 — Локализация

**Сценарий:** API поддерживает RU/EN, error messages должны быть на языке user'а.

```csharp
public class ProductValidator : AbstractValidator<Product>
{
    public ProductValidator(IStringLocalizer<ProductValidator> L)
    {
        RuleFor(p => p.Name)
            .NotEmpty().WithMessage(_ => L["Name_Required"])
            .MaximumLength(200).WithMessage(_ => L["Name_TooLong"]);

        RuleFor(p => p.Price)
            .GreaterThan(0).WithMessage(_ => L["Price_MustBePositive"]);
    }
}
```

```json
// Resources/ProductValidator.ru.json
{
  "Name_Required": "Имя обязательно",
  "Name_TooLong": "Имя слишком длинное",
  "Price_MustBePositive": "Цена должна быть больше 0"
}

// Resources/ProductValidator.en.json
{
  "Name_Required": "Name is required",
  "Name_TooLong": "Name is too long",
  "Price_MustBePositive": "Price must be positive"
}
```

```csharp
// Setup
builder.Services.AddLocalization(opts => opts.ResourcesPath = "Resources");
builder.Services.AddRequestLocalization(opts =>
{
    opts.SupportedCultures = new[] { new CultureInfo("en"), new CultureInfo("ru") };
    opts.DefaultRequestCulture = new RequestCulture("en");
});

app.UseRequestLocalization();
```

---

## 13. Case Study #6 — Validation в response API

```csharp
public class GlobalExceptionHandler : IExceptionHandler
{
    public async ValueTask<bool> TryHandleAsync(HttpContext ctx, Exception ex, CancellationToken ct)
    {
        if (ex is FluentValidation.ValidationException validationEx)
        {
            ctx.Response.StatusCode = 422;
            ctx.Response.ContentType = "application/problem+json";

            var problem = new
            {
                type = "https://example.com/errors/validation",
                title = "Validation failed",
                status = 422,
                errors = validationEx.Errors
                    .GroupBy(e => e.PropertyName)
                    .ToDictionary(
                        g => g.Key,
                        g => g.Select(e => e.ErrorMessage).ToArray())
            };

            await ctx.Response.WriteAsJsonAsync(problem, ct);
            return true;
        }
        return false;
    }
}

// Register
builder.Services.AddExceptionHandler<GlobalExceptionHandler>();
```

**Response:**
```json
{
  "type": "https://example.com/errors/validation",
  "title": "Validation failed",
  "status": 422,
  "errors": {
    "Email": ["Email уже зарегистрирован"],
    "Password": ["Минимум 8 символов", "Должна быть цифра"]
  }
}
```

См. [[http-fundamentals|HTTP Fundamentals]] и [[error-handling|Error Handling]].

---

## 14. Common Pitfalls

### 1. Async DB call в каждой rule

```csharp
// ❌ 5 async calls на одно validation
RuleFor(x => x.Email).MustAsync(EmailExists);
RuleFor(x => x.Phone).MustAsync(PhoneExists);
RuleFor(x => x.Username).MustAsync(UsernameExists);
// 5 DB roundtrips!

// ✅ One query вместо many
RuleFor(x => x).MustAsync(async (req, ct) =>
{
    var conflicts = await repo.GetConflictsAsync(req.Email, req.Phone, req.Username, ct);
    return conflicts.Count == 0;
});
```

### 2. Validator не зарегистрирован в DI

```csharp
// ❌ Endpoint резолвит IValidator<CreateUserRequest> — DI кидает
// InvalidOperationException или GetService возвращает null,
// и endpoint-фильтр молча пропускает запрос без валидации

// ✅
builder.Services.AddValidatorsFromAssemblyContaining<CreateUserValidator>();
```

### 3. Validation НЕ запускается на nested objects

```csharp
public class OrderValidator : AbstractValidator<Order>
{
    public OrderValidator()
    {
        RuleFor(x => x.Items).NotEmpty();
        // ❌ Items внутри не валидируются!
    }
}

// ✅
RuleForEach(x => x.Items).SetValidator(new OrderItemValidator());
```

### 4. CascadeMode по умолчанию — `Continue`

```csharp
RuleFor(x => x.Email)
    .NotEmpty()           // если empty —
    .EmailAddress();      // всё равно проверяется (DataAnnotations такого не делает)

// ✅ Stop при первой ошибке
RuleFor(x => x.Email)
    .Cascade(CascadeMode.Stop)
    .NotEmpty()
    .EmailAddress();
```

### 5. Validate результат игнорируется

```csharp
// ❌
var result = validator.Validate(request);
// Что дальше? Если не throw / return — bug.

// ✅
var result = validator.Validate(request);
if (!result.IsValid)
    throw new ValidationException(result.Errors);
```

### 6. Custom messages теряют context

```csharp
// ❌
RuleFor(x => x.Age).GreaterThan(18).WithMessage("Должно быть больше");
// "больше чего?"

// ✅
RuleFor(x => x.Age).GreaterThan(18)
    .WithMessage("Возраст должен быть больше 18");
```

### 7. Сложная business logic в validator

```csharp
// ❌ Validator делает business decisions
RuleFor(x => x.Order).Must(o => 
{
    var customer = LoadCustomer(o.CustomerId);
    var risk = CalculateRiskScore(customer, o);
    return risk < 0.8;
});

// ✅ Validation — checks. Business — в handler.
```

### 8. Не использовать `WithName`

```csharp
RuleFor(x => x.UsrEml).NotEmpty();
// Error: "UsrEml is required" — нечитаемо для frontend

// ✅
RuleFor(x => x.UsrEml).NotEmpty().WithName("Email");
// Error: "Email is required"
```

### 9. Регулярки внутри RuleFor

```csharp
// ❌ Regex компилируется каждый раз
RuleFor(x => x.Phone).Matches(@"^\+?[1-9]\d{1,14}$");

// ✅ Const или GeneratedRegex
private static readonly Regex PhoneRegex = new(@"^\+?[1-9]\d{1,14}$", RegexOptions.Compiled);
RuleFor(x => x.Phone).Matches(PhoneRegex);
```

### 10. Не cancellation token

```csharp
// ❌
.MustAsync((email, _) => repo.EmailExistsAsync(email))
// Если client cancel — DB query продолжается

// ✅
.MustAsync(async (email, ct) => await repo.EmailExistsAsync(email, ct))
```

---

## 15. Best Practices

### Architecture

- **Один validator на одну DTO/command/query**
- **Inject dependencies** через DI (repository, settings)
- **Validators в separate папке** (рядом с DTO)
- **Naming:** `{ClassName}Validator` (e.g., `CreateUserRequestValidator`)

### Performance

- **Cheap rules first** (sync checks)
- **Async rules последними**
- **Combine** async DB queries в одну
- **Compile regex** через `[GeneratedRegex]` (.NET 7+) или `RegexOptions.Compiled`

### Messages

- **Descriptive errors** — что именно не так
- **Используй `nameof`** для polev
- **Локализация** через `IStringLocalizer`
- **Format:** `{Property} is required` не "Required"

### Testing

- **`TestValidate`** helper для cleaner tests
- **Mock dependencies** (repository)
- **Theory + InlineData** для разных inputs
- **Property-based tests** для regex (FsCheck)

### Integration

- **MediatR pipeline behavior** для CQRS
- **Endpoint-фильтр или явный вызов** — НЕ deprecated auto-validation из `FluentValidation.AspNetCore`
- **Global exception handler** маппит → 422

См. [[clean-code|Clean Code]] и [[cqrs-mediatr|CQRS & MediatR]].

---

## 16. Cheat sheet

| Need | Use |
|------|-----|
| Required field | `.NotEmpty()` |
| Email | `.EmailAddress()` |
| Length | `.MinimumLength(n)` / `.MaximumLength(n)` |
| Range | `.InclusiveBetween(min, max)` |
| Regex | `.Matches(@"...")` |
| Custom check | `.Must(predicate)` |
| Async DB check | `.MustAsync(async (val, ct) => ...)` |
| Cross-field | `.Equal(x => x.OtherField)` |
| Conditional | `.When(predicate)` / `.Unless(predicate)` |
| Nested objects | `.SetValidator(new ChildValidator())` |
| List of objects | `RuleForEach(x => x.Items).SetValidator(...)` |
| Stop on first | `.Cascade(CascadeMode.Stop)` |
| Custom message | `.WithMessage("...")` |
| Override property name | `.WithName("Email")` |

---

## 17. Decision tree

```
Какой validation подход?
│
├── Minimal API + простые rules (required/format/range)?
│   → Встроенная валидация .NET 10 (Microsoft.Extensions.Validation)
│     zero dependencies, source-generated, 400 + ProblemDetails из коробки
│
├── Controllers + 2-3 simple rules?
│   → DataAnnotations (auto-validation через [ApiController])
│
├── Async checks (DB)?
│   → FluentValidation (ни DataAnnotations, ни встроенная не умеют)
│
├── Cross-field rules / DI-зависимости в правилах?
│   → FluentValidation
│
├── Complex business rules?
│   → FluentValidation
│
├── Multi-step / conditional?
│   → FluentValidation (When/Unless)
│
├── Тесты?
│   → FluentValidation (легко testить отдельно)
│
└── В CQRS / MediatR?
    → FluentValidation + ValidationBehavior pipeline
```

---

## 18. См. также

- [[api-design|API Design]] — request/response DTOs
- [[http-fundamentals|HTTP Fundamentals]] — 422 status code
- [[cqrs-mediatr|CQRS & MediatR]] — pipeline integration
- [[error-handling|Error Handling]] — `Result<T>` vs exceptions
- [[testing-fundamentals|Testing]] — validator tests
- [[basics-tracking|EF Basics]] — async repository
- [[object-mapping|Object Mapping]] — после validation

## 19. Reading list

- **FluentValidation docs** — docs.fluentvalidation.net
- **Jeremy Skinner blog** (FluentValidation creator) — jeremyskinner.co.uk
- **Domain-Driven Design** — Eric Evans (где validation располагать)
- **Microsoft Docs — Model Validation in ASP.NET Core** — learn.microsoft.com/aspnet/core/mvc/models/validation
- **MediatR + FluentValidation** — github.com/LuckyPennySoftware/MediatR/wiki
