---
tags: [csharp, attributes, metadata, reflection, middle]
level: Middle
date: 2026-04-30
---

# Attributes и Metadata — атрибуты в C#

> **Атрибуты — метаданные на коде**. Используются везде: ASP.NET (`[Authorize]`, `[ApiController]`), validation (`[Required]`), serialization (`[JsonPropertyName]`), tests (`[Fact]`), DI marker interfaces. Closes пробел "знаю что такие штуки в квадратных скобках, но не понимаю как они работают и как создавать свои".

---

## Что это, зачем и когда

### Что такое атрибут

**Метаданные**, прикреплённые к элементам кода (классам, методам, properties, параметрам). Не выполняются сами по себе — **frameworks и runtime их читают** через reflection.

```csharp
[Obsolete("Use NewMethod instead")]
public void OldMethod() { }

// "Obsolete" — атрибут. Компилятор читает его и выдаёт warning.
```

**Аналогия:** Стикеры на коробках на складе. Сама коробка — это код. Стикер ("fragile", "this side up") — атрибут. Коробка не "хрупкая" сама по себе — но рабочие читают стикер и обращаются с ней соответственно.

### Зачем

| Без атрибутов | С атрибутами |
|---------------|--------------|
| Marker interfaces (empty `IXxx`) | `[Xxx]` атрибут |
| Configuration через код / XML | Декларативно: `[Required]`, `[MaxLength(100)]` |
| Switch-case "если тип такой — действие" | Reflection ищет атрибуты автоматически |
| Verbose API definitions | `[ApiController]` — один декоратор |

### Где встречаются

```csharp
// ASP.NET Core
[ApiController]
[Route("api/[controller]")]
public class UsersController : ControllerBase
{
    [HttpGet("{id}")]
    [Authorize(Roles = "Admin")]
    public IActionResult Get([FromRoute] int id) { }
}

// Validation
public class CreateUserRequest
{
    [Required]
    [MaxLength(100)]
    public string Name { get; set; }

    [EmailAddress]
    public string Email { get; set; }
}

// JSON serialization
public class User
{
    [JsonPropertyName("user_id")]
    public int Id { get; set; }

    [JsonIgnore]
    public string Password { get; set; }
}

// Tests
[Fact]
public void Test() { }

[Theory]
[InlineData(1)]
[InlineData(2)]
public void TestWithData(int x) { }

// EF Core
public class User
{
    [Key]
    public Guid Id { get; set; }

    [Column("user_name")]
    [Required]
    public string Name { get; set; }
}

// Marker / behavior
[Obsolete]
[Serializable]
[Flags]
[DebuggerDisplay("{Name}")]
```

---

## 1. Базовый syntax

### Применение атрибута

```csharp
// На класс
[Serializable]
public class User { }

// На метод
[Obsolete]
public void OldMethod() { }

// На property
[Required]
public string Name { get; set; }

// На параметр
public void Method([NotNull] string param) { }

// На assembly
[assembly: AssemblyVersion("1.0.0")]

// Несколько на один элемент
[Required]
[MaxLength(100)]
public string Name { get; set; }

// Combined syntax (одна строка)
[Required, MaxLength(100)]
public string Name { get; set; }
```

### Параметры в атрибуте

```csharp
// Positional parameters (в constructor)
[Obsolete("Use NewMethod")]
public void Old() { }

// Named parameters
[Obsolete("Use NewMethod", error: true)]
public void Old() { }

// Combined
[Authorize(Roles = "Admin", Policy = "MinAge")]
public IActionResult AdminAction() { }
```

---

## 2. Часто используемые built-in атрибуты

### Compiler / Runtime

```csharp
[Obsolete("Use NewMethod")]                    // warning при использовании
[Obsolete("Removed", true)]                    // error при использовании

[Serializable]                                  // legacy serialization
[NonSerialized]                                 // exclude field

[Flags]                                         // enum как bitmask
[DebuggerDisplay("{Name}")]                     // debugger formatting
[CompilerGenerated]                             // generated code marker
[Conditional("DEBUG")]                          // вызов только при #define
[CallerMemberName]                              // auto fill caller's name
```

#### CallerInfo атрибуты — useful для logging

```csharp
public void Log(
    string message,
    [CallerMemberName] string caller = "",
    [CallerFilePath] string file = "",
    [CallerLineNumber] int line = 0)
{
    Console.WriteLine($"{file}:{line} ({caller}): {message}");
}

// Caller — без указания этих параметров
public void DoWork()
{
    Log("Started");
    // Output: /path/MyClass.cs:42 (DoWork): Started
}
```

#### CallerArgumentExpression (.NET 6+)

```csharp
public static void ThrowIf(
    bool condition,
    [CallerArgumentExpression(nameof(condition))] string? expr = null)
{
    if (condition)
        throw new InvalidOperationException($"Expression failed: {expr}");
}

// Использование
int x = -5;
ThrowIf(x < 0);
// Throws: "Expression failed: x < 0"
```

### ASP.NET Core

```csharp
[ApiController]                    // включает API conventions
[Route("api/[controller]")]        // routing
[HttpGet], [HttpPost], [HttpPut], [HttpDelete]

[FromBody]                          // bind from request body
[FromQuery]                         // bind from query string
[FromRoute]                         // bind from route
[FromHeader]                        // bind from header
[FromForm]                          // bind from form

[Authorize]                         // require auth
[Authorize(Roles = "Admin")]
[AllowAnonymous]                    // override [Authorize]

[ProducesResponseType(200)]
[Consumes("application/json")]
[Produces("application/json")]
```

### Validation (System.ComponentModel.DataAnnotations)

```csharp
public class CreateOrderRequest
{
    [Required]
    public int UserId { get; set; }

    [Required, MinLength(3), MaxLength(100)]
    public string Title { get; set; }

    [Range(0.01, 1_000_000)]
    public decimal Amount { get; set; }

    [EmailAddress]
    public string Email { get; set; }

    [RegularExpression(@"^\d{4}-\d{2}-\d{2}$")]
    public string Date { get; set; }

    [Url]
    public string Link { get; set; }

    [StringLength(50, MinimumLength = 5)]
    public string Username { get; set; }

    [Compare(nameof(Password))]
    public string ConfirmPassword { get; set; }
}
```

### JSON serialization (System.Text.Json)

```csharp
public class User
{
    [JsonPropertyName("user_id")]
    public int Id { get; set; }

    [JsonIgnore]
    public string Password { get; set; }

    [JsonInclude]
    private string _internalField;

    [JsonPropertyOrder(1)]
    public string FirstName { get; set; }

    [JsonNumberHandling(JsonNumberHandling.AllowReadingFromString)]
    public int Age { get; set; }
}
```

### EF Core

```csharp
public class User
{
    [Key]                              // primary key
    public Guid Id { get; set; }

    [Required]                         // NOT NULL
    [MaxLength(100)]                   // VARCHAR(100)
    public string Name { get; set; }

    [Column("email_address")]          // column name override
    [Index(IsUnique = true)]
    public string Email { get; set; }

    [NotMapped]                        // не в БД
    public string Computed => $"{Name} <{Email}>";

    [Timestamp]                        // optimistic concurrency
    public byte[] RowVersion { get; set; }
}

[Table("users")]                       // table name override
public class User { }
```

### Testing (xUnit)

```csharp
[Fact]
public void Test_simple() { }

[Theory]
[InlineData(1, 2, 3)]
[InlineData(0, 0, 0)]
public void Add_returns_sum(int a, int b, int expected) { }

[Trait("Category", "Integration")]
public void IntegrationTest() { }

[Skip("Reason")]
public void NotImplementedYet() { }

[Collection("Database")]
public class DatabaseTest { }
```

### Diagnostics

```csharp
[StackTraceHidden]                                  // не показывать в stack trace
[DebuggerStepThrough]                               // F11 не зайдёт
[DebuggerHidden]                                    // не break inside
[Conditional("DEBUG")]                              // вызов compile-time conditional
```

---

## 3. Custom attributes — пишем свои

### Минимум

```csharp
public class MyAttribute : Attribute
{
}

// Использование
[My]
public class Foo { }
```

> [!info] Convention
> Имя класса заканчивается `Attribute`. При использовании можно опустить — компилятор добавит автоматически. `[My]` ↔ `MyAttribute`.

### С параметрами

```csharp
public class CategoryAttribute : Attribute
{
    public string Name { get; }
    public int Priority { get; set; } = 0;

    public CategoryAttribute(string name)
    {
        Name = name;
    }
}

// Использование
[Category("Important")]
public class Foo { }

[Category("Critical", Priority = 10)]
public class Bar { }
```

### `AttributeUsage` — control где можно применять

```csharp
[AttributeUsage(
    AttributeTargets.Class | AttributeTargets.Struct,  // только на типы
    AllowMultiple = false,                               // один раз на элемент
    Inherited = true)]                                   // наследуется
public class CategoryAttribute : Attribute { }

// Что можно targets:
// Assembly | Module | Class | Struct | Enum | Constructor |
// Method | Property | Field | Event | Interface | Parameter |
// Delegate | ReturnValue | GenericParameter | All
```

### AllowMultiple — несколько раз

```csharp
[AttributeUsage(AttributeTargets.Class, AllowMultiple = true)]
public class TagAttribute : Attribute
{
    public string Name { get; }
    public TagAttribute(string name) => Name = name;
}

// Использование
[Tag("user")]
[Tag("api")]
[Tag("public")]
public class UserService { }
```

### Inherited — наследование

```csharp
[AttributeUsage(AttributeTargets.Class, Inherited = true)]
public class BaseClassAttribute : Attribute { }

[BaseClass]
public class Parent { }

public class Child : Parent { }  // тоже имеет [BaseClass] (inherited)
```

---

## 4. Чтение атрибутов через reflection

### Получить атрибуты класса

```csharp
[Category("Important", Priority = 5)]
public class Foo { }

// Read
Type type = typeof(Foo);

// All attributes
var attrs = type.GetCustomAttributes(true);

// Specific type
var category = type.GetCustomAttribute<CategoryAttribute>();
if (category != null)
{
    Console.WriteLine($"Name={category.Name}, Priority={category.Priority}");
}

// Все атрибуты типа CategoryAttribute
var categories = type.GetCustomAttributes<CategoryAttribute>();
```

### Атрибуты методов / properties

```csharp
public class Service
{
    [Obsolete]
    public void Old() { }

    [Category("Important")]
    public string Name { get; set; }
}

// Метод
var method = typeof(Service).GetMethod(nameof(Service.Old));
var obsolete = method.GetCustomAttribute<ObsoleteAttribute>();

// Property
var property = typeof(Service).GetProperty(nameof(Service.Name));
var category = property.GetCustomAttribute<CategoryAttribute>();

// Параметры метода
var param = method.GetParameters()[0];
var attrs = param.GetCustomAttributes();
```

### Has attribute — проверка

```csharp
// Defined check (без получения instance)
bool isObsolete = method.IsDefined(typeof(ObsoleteAttribute), inherit: true);

// Эквивалентно
bool isObsolete = method.GetCustomAttribute<ObsoleteAttribute>() != null;
```

### Iterate properties с атрибутом

```csharp
public class Validator
{
    public List<string> Validate(object obj)
    {
        var errors = new List<string>();
        var properties = obj.GetType().GetProperties();

        foreach (var prop in properties)
        {
            var required = prop.GetCustomAttribute<RequiredAttribute>();
            if (required != null && prop.GetValue(obj) == null)
                errors.Add($"{prop.Name} is required");

            var maxLength = prop.GetCustomAttribute<MaxLengthAttribute>();
            if (maxLength != null && prop.GetValue(obj) is string s && s.Length > maxLength.Length)
                errors.Add($"{prop.Name} too long");
        }

        return errors;
    }
}
```

---

## 5. Real-world patterns

### Pattern 1: Custom validator

```csharp
[AttributeUsage(AttributeTargets.Property)]
public class EvenNumberAttribute : ValidationAttribute
{
    protected override ValidationResult IsValid(object value, ValidationContext context)
    {
        if (value is int i && i % 2 == 0)
            return ValidationResult.Success;

        return new ValidationResult($"{context.MemberName} must be even");
    }
}

// Использование
public class Settings
{
    [EvenNumber]
    public int BatchSize { get; set; }
}
```

### Pattern 2: Categorization для tests

```csharp
[AttributeUsage(AttributeTargets.Method | AttributeTargets.Class)]
public class TestCategoryAttribute : Attribute
{
    public string[] Categories { get; }
    public TestCategoryAttribute(params string[] categories) => Categories = categories;
}

// Использование
[TestCategory("Slow", "Integration")]
public void IntegrationTest() { }

// Filtering
var slowTests = GetTestMethods()
    .Where(m => m.GetCustomAttribute<TestCategoryAttribute>()?
        .Categories.Contains("Slow") ?? false);
```

### Pattern 3: Auto-registration в DI

```csharp
[AttributeUsage(AttributeTargets.Class)]
public class ServiceAttribute : Attribute
{
    public ServiceLifetime Lifetime { get; }
    public ServiceAttribute(ServiceLifetime lifetime = ServiceLifetime.Scoped)
    {
        Lifetime = lifetime;
    }
}

[Service(ServiceLifetime.Singleton)]
public class CacheService { }

[Service]  // default Scoped
public class UserService { }

// В Startup / Program.cs:
public static IServiceCollection AddServicesFromAssembly(this IServiceCollection services, Assembly assembly)
{
    var typesWithAttribute = assembly.GetTypes()
        .Where(t => t.GetCustomAttribute<ServiceAttribute>() != null);

    foreach (var type in typesWithAttribute)
    {
        var attr = type.GetCustomAttribute<ServiceAttribute>();
        services.Add(new ServiceDescriptor(type, type, attr.Lifetime));
    }

    return services;
}
```

### Pattern 4: API documentation / Swagger

```csharp
[ApiController]
[Route("api/[controller]")]
public class OrdersController : ControllerBase
{
    [HttpGet("{id}")]
    [ProducesResponseType(typeof(OrderDto), StatusCodes.Status200OK)]
    [ProducesResponseType(StatusCodes.Status404NotFound)]
    [SwaggerOperation(Summary = "Get order by ID", Description = "Returns order if found")]
    public async Task<IActionResult> Get(int id) { }
}
```

### Pattern 5: Authorization

```csharp
[AttributeUsage(AttributeTargets.Method | AttributeTargets.Class)]
public class RequirePermissionAttribute : Attribute, IAuthorizationFilter
{
    public string Permission { get; }
    public RequirePermissionAttribute(string permission) => Permission = permission;

    public void OnAuthorization(AuthorizationFilterContext context)
    {
        var user = context.HttpContext.User;
        if (!user.HasClaim("permission", Permission))
        {
            context.Result = new ForbidResult();
        }
    }
}

// Использование
[RequirePermission("orders.write")]
public IActionResult CreateOrder() { }
```

### Pattern 6: Auto-mapping (упрощённый)

```csharp
[AttributeUsage(AttributeTargets.Property)]
public class MapsToAttribute : Attribute
{
    public string TargetProperty { get; }
    public MapsToAttribute(string targetProperty) => TargetProperty = targetProperty;
}

public class UserDto
{
    [MapsTo("FullName")]
    public string Name { get; set; }
}

public TDest Map<TSrc, TDest>(TSrc src) where TDest : new()
{
    var dest = new TDest();
    foreach (var prop in typeof(TSrc).GetProperties())
    {
        var mapsTo = prop.GetCustomAttribute<MapsToAttribute>();
        var targetName = mapsTo?.TargetProperty ?? prop.Name;
        var targetProp = typeof(TDest).GetProperty(targetName);
        targetProp?.SetValue(dest, prop.GetValue(src));
    }
    return dest;
}
```

---

## 6. Performance considerations

### Reflection — медленно

```csharp
// ❌ Reflection каждый call — slow
public bool IsValid(object obj)
{
    var props = obj.GetType().GetProperties();  // expensive
    foreach (var prop in props)
    {
        var attr = prop.GetCustomAttribute<RequiredAttribute>();  // slow
        // ...
    }
}

// ✅ Cache reflection results
private static readonly ConcurrentDictionary<Type, PropertyInfo[]> _propsCache = new();
private static readonly ConcurrentDictionary<PropertyInfo, RequiredAttribute?> _attrCache = new();

public bool IsValid(object obj)
{
    var props = _propsCache.GetOrAdd(obj.GetType(), t => t.GetProperties());
    foreach (var prop in props)
    {
        var attr = _attrCache.GetOrAdd(prop, p => p.GetCustomAttribute<RequiredAttribute>());
        // ...
    }
}
```

### Source Generators (.NET 5+) — лучшая альтернатива

Атрибуты можно использовать **в compile-time** через source generators — нулевой runtime cost.

```csharp
[GenerateValidator]
public partial class CreateUserRequest
{
    [Required]
    public string Name { get; set; }

    [EmailAddress]
    public string Email { get; set; }
}

// Source generator генерит:
public partial class CreateUserRequest
{
    public List<string> Validate()
    {
        var errors = new List<string>();
        if (string.IsNullOrEmpty(Name)) errors.Add("Name required");
        if (!IsValidEmail(Email)) errors.Add("Email invalid");
        return errors;
    }
}
```

См. [[source-generators|Source Generators]].

### Compiled expressions — middle ground

```csharp
private static readonly ConcurrentDictionary<Type, Func<object, bool>> _validators = new();

public bool IsValid(object obj)
{
    var validator = _validators.GetOrAdd(obj.GetType(), CompileValidator);
    return validator(obj);
}

private Func<object, bool> CompileValidator(Type type)
{
    // Build expression tree on first call, compile to delegate
    // Subsequent calls — direct delegate invoke (fast)
}
```

См. [[reflection-expression-trees|Reflection и Expression Trees]].

---

## 7. Аtrибуты vs альтернативы

### Атрибут vs marker interface

```csharp
// Marker interface (старый стиль)
public interface IAuditable { }

public class User : IAuditable { }

// Атрибут (modern)
[Auditable]
public class User { }
```

**Атрибут лучше когда:**
- Параметры нужны (`[Auditable("user_audit_log")]`)
- Behavior не required (just metadata)
- Multiple options без boolean flags
- Frameworks expect атрибуты (ASP.NET, EF)

**Interface лучше когда:**
- Behavior нужен (`I.Save()`)
- Polymorphism (`if (obj is IAuditable a) a.Save()`)
- Generic constraint (`where T : IAuditable`)

### Атрибут vs config file

```csharp
// Code-first (атрибут)
public class User
{
    [Required, MaxLength(100)]
    public string Name { get; set; }
}

// Config-first (Fluent API)
public class UserConfig : IEntityTypeConfiguration<User>
{
    public void Configure(EntityTypeBuilder<User> builder)
    {
        builder.Property(u => u.Name).IsRequired().HasMaxLength(100);
    }
}
```

EF Core best practice — Fluent API для сложных configs (атрибуты для simple).

См. [[../EFCore/basics-tracking|EF Core Basics]].

### Атрибут vs convention

```csharp
// Атрибут
[Repository]
public class UserRepository { }

// Convention (просто naming)
public class UserRepository { }  // class ending in "Repository" registered automatically

// DI auto-registration
services.Scan(scan => scan
    .FromAssemblyOf<UserRepository>()
    .AddClasses(c => c.Where(t => t.Name.EndsWith("Repository")))
    .AsImplementedInterfaces());
```

Conventions проще когда applicable. Атрибуты — для exceptions / explicit override.

---

## 8. Common Pitfalls

### 1. Reflection performance

См. секцию **6**. Cache!

### 2. Forgot to read атрибут

```csharp
[Category("Important")]
public class Foo { }

// Нигде не читается — атрибут декоративен!
// Бывает с custom attributes — забыл написать reflection код
```

### 3. AllowMultiple = false по default

```csharp
// ❌ Без [AttributeUsage(..., AllowMultiple = true)]
[Tag("a")]
[Tag("b")]   // ❌ Compile error!
public class Foo { }

// ✅
[AttributeUsage(AttributeTargets.Class, AllowMultiple = true)]
public class TagAttribute : Attribute { }
```

### 4. Attribute не наследуется

```csharp
// По default Inherited = true для большинства атрибутов
// Но для некоторых = false

[Category("X")]
public class Parent { }

public class Child : Parent { }

// Проверь поведение конкретного атрибута!
var attr = typeof(Child).GetCustomAttribute<CategoryAttribute>(inherit: true);
```

### 5. Атрибут изменяет state

```csharp
// ❌ Атрибут не должен иметь mutable state
public class CounterAttribute : Attribute
{
    public static int Count;  // ⚠️ shared mutable!
    public CounterAttribute() { Count++; }
}

// Атрибуты — immutable metadata. State — в class который их читает.
```

### 6. Constructor параметры — должны быть constants

```csharp
// ❌ Variable
[Authorize(Roles = someVariable)]  // Compile error

// ✅ Только constants / typeof / const arrays
[Authorize(Roles = "Admin")]
[Type(typeof(MyClass))]
[Items(new[] { 1, 2, 3 })]

// Атрибут baked at compile time — не runtime values
```

### 7. Performance в ASP.NET filters

```csharp
// Filter создаётся на каждый request если не Singleton-friendly
[ExpensiveFilter]  // ⚠️ если Filter делает heavy work — будет slow

// ✅ ServiceFilterAttribute / TypeFilterAttribute для DI
[ServiceFilter(typeof(MyExpensiveFilter))]
```

### 8. Validation атрибуты для enum

```csharp
public enum Status { Active, Pending, Closed }

public class Order
{
    [Required]
    public Status Status { get; set; }  // ⚠️ Status — value type, не может быть null
}

// ✅ Для optional — Status?
public Status? Status { get; set; }
```

### 9. JsonIgnore не на private

```csharp
public class User
{
    public string Name { get; set; }
    
    [JsonIgnore]
    private string _secret;  // ⚠️ private не serialized по default!
}

// JsonIgnore полезен на public properties которые не должны serialize
```

### 10. Атрибут в namespace conflict

```csharp
using System.ComponentModel.DataAnnotations;
using System.Runtime.Serialization;

public class User
{
    [Required]      // оба namespace имеют Required!
    public string Name { get; set; }
}

// ✅ Fully qualify если ambiguous
[System.ComponentModel.DataAnnotations.Required]
public string Name { get; set; }
```

---

## 9. Best Practices

### Использование

- **Декларативно** — `[Required]` лучше runtime if-checks
- **Specific атрибуты** > generic
- **Naming conventions** — `XxxAttribute` суффикс
- **AttributeUsage** — limit где применять
- **Immutable** — атрибуты не должны иметь mutable state

### Custom attributes

- **Inherit from `Attribute`**
- **`AttributeUsage`** ALWAYS specify
- **AllowMultiple** explicit (true/false)
- **Constructor для required params**
- **Properties для optional** params

### Reading

- **Cache reflection results** — `ConcurrentDictionary<Type, ...>`
- **Source Generators** для compile-time validation (.NET 5+)
- **Compiled Expressions** для runtime + cache

### Анти-patterns

- ❌ Не используй атрибуты для **behavior** (используй interfaces)
- ❌ Не делай "чистые декоративные" атрибуты которые никто не читает
- ❌ Не читай reflection в hot path без cache
- ❌ Не клади бизнес-логику в атрибуты — только metadata

---

## 10. Cheat sheet

| Сценарий | Решение |
|----------|---------|
| Mark obsolete | `[Obsolete("reason")]` |
| Validation на API | `[Required]`, `[Range]`, `[MaxLength]` |
| JSON ignore field | `[JsonIgnore]` |
| JSON rename | `[JsonPropertyName("name")]` |
| EF column | `[Column("name")]`, `[Required]` |
| ASP.NET routing | `[Route]`, `[HttpGet]` |
| ASP.NET binding | `[FromBody]`, `[FromQuery]` |
| ASP.NET auth | `[Authorize]` |
| xUnit test | `[Fact]`, `[Theory]`, `[InlineData]` |
| Auto caller info | `[CallerMemberName]` |
| Custom attribute target | `[AttributeUsage(AttributeTargets.Class)]` |
| Multiple instances | `[AttributeUsage(..., AllowMultiple = true)]` |
| Read attribute | `member.GetCustomAttribute<T>()` |
| Check defined | `member.IsDefined(typeof(T))` |
| Cache reflection | `ConcurrentDictionary<Type, ...>` |

---

## 11. Decision tree

```
Нужны метаданные на коде?
│
├── Простой marker? → Custom attribute без params
├── С configuration? → Attribute с params
├── На класс / метод / property? → AttributeTargets.X
├── Несколько раз? → AllowMultiple = true
├── Наследуется? → Inherited = true (default)
└── Performance важен? → Cache reflection / Source Generator

Вместо attribute:
├── Behavior? → Interface
├── Type-level constraint? → Generic where T : ...
├── Convention? → Naming + reflection scan
└── Configuration? → Fluent API / config file
```

---

## См. также

- [[reflection-expression-trees|Reflection и Expression Trees]] — чтение атрибутов
- [[source-generators|Source Generators]] — compile-time альтернатива
- [[modern-features|Modern C# Features]] — CallerArgumentExpression
- [[oop|OOP]] — interfaces vs attributes
- [[../AspNetCore/auth-security|Auth & Security]] — \[Authorize\] deep
- [[../EFCore/basics-tracking|EF Core Basics]] — Data Annotations
- [[../Testing/testing-fundamentals|Testing]] — xUnit attributes

## Reading list

- **Microsoft Docs — Attributes** — learn.microsoft.com/dotnet/csharp/programming-guide/concepts/attributes
- **Microsoft Docs — Custom Attributes** — learn.microsoft.com/dotnet/standard/attributes/writing-custom-attributes
- **Jon Skeet — C# in Depth** (chapters про reflection)
- **CLR via C#** — Jeffrey Richter (внутренности reflection и метаданных)
- **Andrew Lock — Source Generators** — andrewlock.net
- **Stephen Toub — CallerArgumentExpression** — devblogs.microsoft.com/dotnet
