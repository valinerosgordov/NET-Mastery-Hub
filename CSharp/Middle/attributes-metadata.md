---
tags: [csharp, attributes, metadata, middle, reflection, source-generators, custom-attributes]
level: Middle
date: 2026-05-07
---

# Attributes и Metadata — декларативные метаданные

> **Декларации, прикрепляемые к коду и читаемые в runtime / compile-time.** Built-in атрибуты, custom attributes, reflection для чтения, source generators (.NET 5+), CallerMemberName и compile-time helpers. Закрывает пробел: «знаю про `[Obsolete]`, не понимаю как написать свой и зачем `[AttributeUsage]`».

---

## 0. Как читать

Если впервые работаешь с атрибутами — раздел 1→3. Если уже используешь BCL атрибуты, но непонятно как создать свой — раздел 4. Если интересна compile-time magic — раздел 7 (source generators), 8 (Caller* атрибуты). Production guidance — раздел 10 (best practices), 12 (pitfalls).

Cross-language якоря свёрнуты. Interview-вопросы встроены.

---

## 1. Что это, зачем и когда

### 1.1. Что такое атрибут

**Attribute** — class, наследник `System.Attribute`, который **прикрепляется** к code element (class, method, property, parameter и т.д.) для добавления **metadata**:

```csharp
[Obsolete("Use NewMethod instead")]
public void OldMethod() { /* ... */ }

[Required]
[StringLength(100)]
public string Name { get; set; } = "";

[HttpGet("/users/{id}")]
public IActionResult GetUser(int id) { /* ... */ }
```

Атрибуты **не выполняют** код напрямую — они хранят данные, которые читают другие компоненты (compiler, runtime, frameworks).

### 1.2. Зачем атрибуты

| Use case | Пример |
|----------|--------|
| **Compiler hints** | `[Obsolete]`, `[Conditional]`, `[CallerMemberName]` |
| **Validation** | `[Required]`, `[Range(0, 100)]`, `[EmailAddress]` |
| **Serialization** | `[JsonIgnore]`, `[JsonPropertyName]`, `[XmlElement]` |
| **Routing** | `[Route]`, `[HttpGet]`, `[HttpPost]` |
| **DI / IoC** | `[Inject]`, `[Service]` (некоторые containers) |
| **Testing** | `[Fact]`, `[Theory]`, `[InlineData]` (xUnit) |
| **Source generators** | `[GeneratedRegex]`, `[StringSyntax]` |
| **Code analysis** | `[Pure]`, `[NotNull]`, `[MaybeNull]` |

### 1.3. Главное правило

```
Используй built-in attributes когда возможно.
Custom attribute создавай когда:
  - Существует tool / framework который их читает
  - Reflection позволяет извлекать metadata в runtime
  - Source generator может процессить compile-time

НЕ создавай custom attribute если:
  - Нужна простая business logic (используй interface / inheritance)
  - Никто не будет читать metadata
  - Решение можно через regular code
```

### 1.4. Эволюция

| Версия | Что |
|--------|-----|
| **.NET 1.0** | Custom attributes, reflection |
| **.NET 2.0** | Generic types — атрибуты могут быть generic (.NET 7+) |
| **C# 4.5+** | `CallerMemberName`, `CallerLineNumber`, `CallerFilePath` |
| **C# 8+** | Nullable annotation attributes (`[NotNull]`, `[NotNullWhen]`) |
| **C# 10+** | `[CallerArgumentExpression]` |
| **.NET 7+** | Generic attributes (`[Attr<T>]`) |
| **.NET 8+** | `[StringSyntax]` для IDE syntax highlighting |

> [!info]- Если ты знаешь Java / Python / TypeScript / Rust
> **Java:** annotations (`@Override`, `@Deprecated`, custom через `@interface`). Reflection API похож. Compile-time annotations через annotation processors ↔ source generators.
>
> **Python:** decorators (`@staticmethod`, `@property`). Не ровно то же — decorators wrap functions, attributes — pure metadata. Custom decorators могут replicate attribute pattern.
>
> **TypeScript:** decorators (experimental до недавнего, теперь stable) — близко к Java/C# attributes по syntax.
>
> **Rust:** `#[derive(...)]`, `#[attribute_name]` — compile-time только. Очень похожий syntax. Procedural macros ↔ source generators.

> [!question]- Интервью: чем атрибут отличается от обычного класса?
> Атрибут — **метаданные, прикреплённые к code element**, не behavior. Класс наследник `System.Attribute`, instance создаётся **lazy** при reflection. Атрибут не выполняется напрямую — другой код (frameworks, compiler, runtime) читает его через `MemberInfo.GetCustomAttributes(...)`. Built-in patterns: validation (`[Required]`), serialization (`[JsonIgnore]`), routing (`[HttpGet]`). Custom attributes требуют tool/framework который их интерпретирует.

---

## 2. Built-in attributes — самые важные

### 2.1. [Obsolete] — deprecation

```csharp
[Obsolete("Use NewMethod instead")]
public void OldMethod() { }

[Obsolete("Removed in v3", error: true)]   // error, не warning
public void RemovedMethod() { }
```

Compiler выдаёт warning (или error) при использовании. Стандарт для API evolution.

### 2.2. [Conditional] — compile-time inclusion

```csharp
[Conditional("DEBUG")]
public void LogDebug(string msg) => Console.WriteLine(msg);

// В Release build — calls к LogDebug elided полностью
```

`Conditional` symbol определён → method calls компилируются normally. Не определён → calls **исчезают** (включая argument evaluation!).

### 2.3. [Flags] — bitfield enum

```csharp
[Flags]
public enum Permissions
{
    None = 0,
    Read = 1,
    Write = 2,
    Delete = 4,
    All = Read | Write | Delete
}

var p = Permissions.Read | Permissions.Write;   // 3
p.ToString();   // "Read, Write" (с Flags)
```

Без `[Flags]` — `ToString()` возвращает "3", не "Read, Write". См. [[enums-flags]].

### 2.4. [Serializable] / serialization attributes

```csharp
[Serializable]
public class User
{
    [NonSerialized]
    private int _cache;
}

// JSON
public class UserDto
{
    [JsonPropertyName("user_id")]
    public int Id { get; set; }
    
    [JsonIgnore]
    public string Password { get; set; } = "";
}
```

### 2.5. [Required] / Validation

```csharp
public class UserRequest
{
    [Required]
    [StringLength(100, MinimumLength = 2)]
    public string Name { get; set; } = "";
    
    [EmailAddress]
    public string Email { get; set; } = "";
    
    [Range(18, 120)]
    public int Age { get; set; }
    
    [RegularExpression(@"^\+?\d{10,15}$")]
    public string Phone { get; set; } = "";
}

// ASP.NET Core / DataAnnotations используют для validation
```

### 2.6. [DebuggerDisplay] / [DebuggerStepThrough]

```csharp
[DebuggerDisplay("User {Id}: {Name}")]
public class User
{
    public int Id { get; set; }
    public string Name { get; set; } = "";
}

[DebuggerStepThrough]   // debugger пропускает этот метод
public T Validate<T>(T input) where T : notnull => input;
```

См. [[debugging-basics]].

### 2.7. [DllImport] / interop

```csharp
[DllImport("user32.dll")]
public static extern int MessageBox(IntPtr hWnd, string text, string caption, int type);
```

P/Invoke — вызов native APIs.

### 2.8. ASP.NET Core attributes

```csharp
[ApiController]
[Route("api/[controller]")]
public class UsersController : ControllerBase
{
    [HttpGet("{id}")]
    public IActionResult Get([FromRoute] int id) { }
    
    [HttpPost]
    public IActionResult Create([FromBody] CreateUserRequest req) { }
    
    [Authorize(Roles = "Admin")]
    [HttpDelete("{id}")]
    public IActionResult Delete(int id) { }
}
```

Routing, model binding, authorization — всё через атрибуты.

> [!question]- Интервью: что делает `[Conditional("DEBUG")]`?
> Атрибут на method: **call to method** (включая argument evaluation!) **eliminated полностью** при компиляции, если symbol не определён. Например, `LogDebug(ExpensiveCompute())` в Release build — не вызывает ни LogDebug, ни ExpensiveCompute. Ограничения: метод должен быть `void` (нельзя на returning method). Полезно для debug-only logging без runtime overhead. Альтернативы: `Debug.WriteLine` (тоже Conditional), `#if DEBUG` блок (более explicit). Conditional не работает на pre-existing call sites из other assemblies (только compile-time when site compiles).

---

## 3. Применение и synthax

### 3.1. К чему можно применять

```csharp
[Attribute]
public class MyClass { }                   // class

[Attribute]
public struct MyStruct { }                 // struct

[Attribute]
public interface IFoo { }                  // interface

[Attribute]
public enum Color { }                       // enum

[Attribute]
public delegate void Handler();            // delegate

public class Container
{
    [Attribute]
    private int _field;                     // field
    
    [Attribute]
    public string Name { get; set; }        // property
    
    [Attribute]
    public event EventHandler? Changed;     // event
    
    [Attribute]
    public void Method([Attribute] int param)   // method, parameter
    {
        [Attribute] int local = 0;          // local (rarely)
    }
    
    [Attribute]
    public int this[int i] => 0;            // indexer
    
    [return: Attribute]                      // return value
    public int Compute() => 42;
    
    [field: Attribute]                       // backing field of auto-property
    public string Name2 { get; set; }
}

[assembly: Attribute]                        // assembly-level
[module: Attribute]                          // module-level
```

### 3.2. Multiple attributes

```csharp
[Required]
[StringLength(100)]
[EmailAddress]
public string Email { get; set; } = "";

// Эквивалент
[Required, StringLength(100), EmailAddress]
public string Email2 { get; set; } = "";
```

### 3.3. Targets — `[return:]`, `[param:]`, `[field:]`, `[method:]`

```csharp
[method: Conditional("DEBUG")]   // обычно implicit
[return: NotNull]
public string Process(string input)
{
    [field: NonSerialized]   // на backing field auto-property
    /* ... */
    return input;
}
```

Specifier нужен когда compiler ambiguous (например, на auto-property — `[Attr]` идёт на property, нужно `[field: Attr]` для backing field).

### 3.4. Named и positional arguments

```csharp
[Obsolete("message", error: true)]   // positional + named

public class CustomAttr : Attribute
{
    public CustomAttr(string positional1, int positional2) { /* ... */ }
    public string NamedProperty { get; set; } = "";
}

[CustomAttr("hello", 42, NamedProperty = "value")]
public class Foo { }
```

### 3.5. Attribute suffix — optional

```csharp
public class RequiredAttribute : Attribute { }

[Required]            // compiler ищет RequiredAttribute
[RequiredAttribute]   // тоже OK, но verbose
```

Convention: пишешь без `Attribute` suffix в use site.

> [!question]- Интервью: куда можно применять атрибуты?
> Почти везде: classes, structs, interfaces, enums, delegates, methods, properties, fields, events, parameters, return values, type parameters, locals (редко), assembly, module. Specifier помогает когда compiler ambiguous: `[return: Attr]` для return value, `[field: Attr]` для backing field auto-property, `[method: Attr]` для get/set accessor. `[AttributeUsage(AttributeTargets.X)]` на custom attribute класс ограничивает где можно применять.

---

## 4. Custom attributes

### 4.1. Минимальный custom attribute

```csharp
[AttributeUsage(AttributeTargets.Class)]
public class TrackedAttribute : Attribute
{
    public string Owner { get; }
    public DateOnly Since { get; }
    
    public TrackedAttribute(string owner, string since)
    {
        Owner = owner;
        Since = DateOnly.Parse(since);
    }
}

[Tracked("Alice", "2024-01-15")]
public class OrderService { }
```

### 4.2. AttributeUsage — controls применение

```csharp
[AttributeUsage(
    AttributeTargets.Method | AttributeTargets.Property,
    AllowMultiple = true,
    Inherited = false)]
public class TagAttribute : Attribute
{
    public string Name { get; }
    public TagAttribute(string name) => Name = name;
}

public class Foo
{
    [Tag("important")]
    [Tag("expensive")]   // multiple OK
    public void Method() { }
}
```

| Param | Default | Что |
|-------|---------|-----|
| `AttributeTargets` | All | Где можно применять |
| `AllowMultiple` | false | Несколько копий на одном element |
| `Inherited` | true | Унаследуется derived классами |

### 4.3. Constructor positional + properties named

```csharp
[AttributeUsage(AttributeTargets.Class)]
public class ApiVersionAttribute : Attribute
{
    public int Major { get; }
    public int Minor { get; }
    public bool Deprecated { get; set; }   // named — optional
    
    public ApiVersionAttribute(int major, int minor)
    {
        Major = major;
        Minor = minor;
    }
}

[ApiVersion(1, 0, Deprecated = true)]
public class OldController { }
```

Constructor params — обязательные positional. Properties с public setter — optional named.

### 4.4. Constraints на типы атрибутов

```csharp
// Constructor parameters - только compile-time constants
public class BadAttr : Attribute
{
    // public BadAttr(DateTime dt) { }   // ❌ DateTime не const-able
    public BadAttr(string isoDate) { }   // ✅ string OK, parse в run-time
}
```

Allowed types в constructor: primitives, string, `Type`, enum, 1D arrays of these.

### 4.5. Generic attributes (.NET 7+)

```csharp
[AttributeUsage(AttributeTargets.Class)]
public class HandlerAttribute<T> : Attribute where T : IRequest
{
    public Type RequestType => typeof(T);
}

[Handler<CreateUserRequest>]
public class CreateUserHandler { }
```

До .NET 7 — workaround через `Type` parameter:
```csharp
public HandlerAttribute(Type requestType) { /* ... */ }
[Handler(typeof(CreateUserRequest))]
```

### 4.6. Inheritance в attributes

```csharp
public class BaseValidationAttribute : Attribute { /* ... */ }
public class EmailValidationAttribute : BaseValidationAttribute { /* ... */ }
```

Custom attributes могут наследоваться. `[AttributeUsage(Inherited = true)]` (default) — производный class инherits attribute от base.

> [!question]- Интервью: как создать custom attribute?
> 1) Класс наследник `System.Attribute`. 2) Имя суффикс `Attribute` (convention). 3) `[AttributeUsage(AttributeTargets...)]` для controls применения. 4) Constructor — positional required parameters. 5) Public properties с setter — named optional. 6) Constructor параметры могут быть только compile-time constants (primitives, string, Type, enum, arrays of these). 7) `AllowMultiple = true` если нужно несколько копий. 8) Generic attributes — .NET 7+, до этого через `Type` parameter. Frameworks читают через reflection: `MemberInfo.GetCustomAttributes(typeof(MyAttr))`.

---

## 5. Reading attributes via reflection

### 5.1. Базовый пример

```csharp
[Tracked("Alice", "2024-01-15")]
public class OrderService { }

var type = typeof(OrderService);
var attr = type.GetCustomAttribute<TrackedAttribute>();
if (attr != null)
{
    Console.WriteLine($"Owner: {attr.Owner}, Since: {attr.Since}");
}
```

### 5.2. GetCustomAttributes для multiple

```csharp
[Tag("a")]
[Tag("b")]
public void Method() { }

var method = typeof(Foo).GetMethod("Method")!;
var tags = method.GetCustomAttributes<TagAttribute>();
foreach (var t in tags) Console.WriteLine(t.Name);
```

### 5.3. На members класса

```csharp
var type = typeof(User);

foreach (var prop in type.GetProperties())
{
    var required = prop.GetCustomAttribute<RequiredAttribute>();
    var stringLen = prop.GetCustomAttribute<StringLengthAttribute>();
    
    if (required != null) Console.WriteLine($"{prop.Name} is required");
    if (stringLen != null) Console.WriteLine($"{prop.Name} max length: {stringLen.MaximumLength}");
}
```

### 5.4. С наследованием

```csharp
// Inherited = true (default)
var attr = type.GetCustomAttribute<MyAttribute>(inherit: true);

// Inherited = false
var attr2 = type.GetCustomAttribute<MyAttribute>(inherit: false);
```

`inherit: true` — search в base classes тоже.

### 5.5. Performance

```csharp
// Кэшируй reflection results — медленно
private static readonly Dictionary<Type, MyAttribute?> _cache = new();

public MyAttribute? GetAttr(Type type)
{
    if (!_cache.TryGetValue(type, out var attr))
    {
        attr = type.GetCustomAttribute<MyAttribute>();
        _cache[type] = attr;
    }
    return attr;
}
```

Reflection — slow. Для hot path кэш или source generators (compile-time).

### 5.6. CustomAttributeData — без instantiation

```csharp
var data = type.GetCustomAttributesData();
foreach (var attrData in data)
{
    Console.WriteLine($"Type: {attrData.AttributeType.Name}");
    foreach (var arg in attrData.ConstructorArguments)
        Console.WriteLine($"  Arg: {arg.Value}");
}
```

`CustomAttributeData` — read metadata без создания attribute instance. Для analysis tools (Roslyn analyzers).

> [!question]- Интервью: как прочитать атрибут в runtime?
> Через **reflection** — `MemberInfo.GetCustomAttribute<T>()` или `GetCustomAttributes<T>()` (multiple). Доступно на `Type`, `MethodInfo`, `PropertyInfo`, `FieldInfo`, `ParameterInfo`, `EventInfo`. Параметр `inherit: true` — search в base classes. Performance: reflection slow, **кэшируй** результаты в `Dictionary<Type, T?>` или используй source generators для compile-time. `CustomAttributeData.GetCustomAttributesData()` — read metadata без instantiation (для analysis tools).

---

## 6. Validation pattern

### 6.1. DataAnnotations — full ecosystem

```csharp
public class CreateUserRequest
{
    [Required(ErrorMessage = "Name required")]
    [StringLength(100, MinimumLength = 2)]
    public string Name { get; set; } = "";
    
    [EmailAddress]
    public string Email { get; set; } = "";
    
    [Range(0, 120)]
    public int Age { get; set; }
    
    [Phone]
    public string? Phone { get; set; }
    
    [Url]
    public string? Website { get; set; }
}
```

ASP.NET Core, EF Core, DataAnnotations библиотека — все используют эти атрибуты.

### 6.2. Custom validation attribute

```csharp
public class FutureDateAttribute : ValidationAttribute
{
    public override bool IsValid(object? value)
    {
        if (value is DateTime dt) return dt > DateTime.UtcNow;
        if (value is DateOnly d) return d > DateOnly.FromDateTime(DateTime.UtcNow);
        return false;
    }
}

public class Event
{
    [FutureDate(ErrorMessage = "Event must be in future")]
    public DateTime Date { get; set; }
}
```

### 6.3. Validator helper

```csharp
public static List<ValidationResult> Validate<T>(T obj) where T : notnull
{
    var results = new List<ValidationResult>();
    var ctx = new ValidationContext(obj);
    Validator.TryValidateObject(obj, ctx, results, validateAllProperties: true);
    return results;
}

var errors = Validate(new CreateUserRequest { Name = "" });
foreach (var e in errors) Console.WriteLine(e.ErrorMessage);
```

### 6.4. ASP.NET Core integration

```csharp
[HttpPost]
public IActionResult Create([FromBody] CreateUserRequest request)
{
    if (!ModelState.IsValid)
        return BadRequest(ModelState);
    // ... validated, can proceed
}

// С [ApiController] — automatic validation
[ApiController]
public class UsersController : ControllerBase
{
    [HttpPost]
    public IActionResult Create([FromBody] CreateUserRequest request)
    {
        // ModelState already checked, BadRequest returned автоматически
    }
}
```

> [!question]- Интервью: как работает validation через DataAnnotations?
> Атрибуты (`[Required]`, `[StringLength]`, `[Range]`) inherit from `ValidationAttribute` — abstract класс с `IsValid(object value)` методом. Validator (например, `Validator.TryValidateObject`) reflects properties объекта, читает атрибуты, вызывает `IsValid` для каждого. Errors — list of `ValidationResult`. ASP.NET Core integrate автоматически: `[ApiController]` validates input на model binding, returns 400 BadRequest при errors. Custom attribute — наследник `ValidationAttribute`, override `IsValid`. EF Core тоже использует для column constraints.

---

## 7. Source generators (.NET 5+)

### 7.1. Что такое

**Source generators** — analyzer-like компоненты, которые **генерируют code** на compile-time. Читают AST + metadata + attributes, выдают `.cs` файлы.

```csharp
// Atомарный пример — INotifyPropertyChanged
public partial class ViewModel : INotifyPropertyChanged
{
    [ObservableProperty]
    private string _name = "";
    
    [ObservableProperty]
    private int _count;
}

// Generator создаёт public Name { get; set; } с PropertyChanged invocation
```

### 7.2. Использование (consumer perspective)

Установка через NuGet:
```xml
<PackageReference Include="CommunityToolkit.Mvvm" Version="8.0.0" />
```

Атрибуты `[ObservableProperty]`, `[RelayCommand]` маркируют члены, generator пишет boilerplate.

### 7.3. Built-in source generators в BCL

```csharp
// .NET 7+
public partial class Validators
{
    [GeneratedRegex(@"\d+")]
    private static partial Regex DigitRegex();
}
// generator пишет implementation на compile-time, no runtime parsing

// JsonSerializer (.NET 6+)
[JsonSerializable(typeof(User))]
internal partial class MyJsonContext : JsonSerializerContext { }

JsonSerializer.Serialize(user, MyJsonContext.Default.User);
// generator avoids reflection, AOT-friendly
```

### 7.4. LoggerMessage source generator

```csharp
public partial class UserService
{
    private readonly ILogger<UserService> _logger;
    
    [LoggerMessage(
        EventId = 100,
        Level = LogLevel.Information,
        Message = "User {UserId} logged in from {IpAddress}")]
    static partial void LogUserLogin(ILogger logger, int userId, string ipAddress);
    
    public void OnLogin(int userId, string ip)
    {
        LogUserLogin(_logger, userId, ip);   // efficient, structured
    }
}
```

Source generator создаёт zero-allocation logging с structured fields.

### 7.5. Source generators vs reflection

| | Source generator | Reflection |
|---|------------------|------------|
| Когда работает | Compile-time | Runtime |
| Performance | Native speed | Slow |
| AOT-friendly | ✅ | ⚠ many limitations |
| Debugging | Generated `.cs` viewable | Black-box |
| Complexity | Higher для author | Easier для consumer |

**Best practice 2024+**: предпочитай source generators для metadata-heavy scenarios (JSON, regex, logging, MVVM).

> [!question]- Интервью: что такое source generators?
> Roslyn-based components, которые **генерируют C# code на compile-time** на основе атрибутов / metadata в исходном коде. Альтернатива reflection — performance lookup'ы becomes compile-time (no runtime cost), AOT-friendly (для Native AOT). Examples: `[GeneratedRegex]` (.NET 7+), `[JsonSerializable]` (.NET 6+), `[LoggerMessage]`, MVVM Toolkit `[ObservableProperty]`. Generator создаёт partial class implementations. Best practice 2024+: предпочитай source generators reflection для metadata-heavy scenarios.

---

## 8. Caller* атрибуты — compile-time helpers

### 8.1. CallerMemberName, CallerFilePath, CallerLineNumber

```csharp
public void Log(
    string message,
    [CallerMemberName] string member = "",
    [CallerFilePath] string file = "",
    [CallerLineNumber] int line = 0)
{
    Console.WriteLine($"[{Path.GetFileName(file)}:{line}] {member}: {message}");
}

public void DoWork()
{
    Log("Starting");   // member = "DoWork", file = "/path/to/file.cs", line = 42
}
```

Compiler **подставляет** значения на compile-time. Caller не знает, не пишет вручную.

### 8.2. INotifyPropertyChanged classic

```csharp
public class ViewModel : INotifyPropertyChanged
{
    public event PropertyChangedEventHandler? PropertyChanged;
    
    private string _name = "";
    public string Name
    {
        get => _name;
        set
        {
            if (_name != value)
            {
                _name = value;
                OnPropertyChanged();   // не нужно "Name" вручную
            }
        }
    }
    
    protected void OnPropertyChanged([CallerMemberName] string? property = null)
    {
        PropertyChanged?.Invoke(this, new PropertyChangedEventArgs(property));
    }
}
```

### 8.3. CallerArgumentExpression (C# 10+)

```csharp
public static void ThrowIfNull(
    [NotNull] object? argument,
    [CallerArgumentExpression(nameof(argument))] string? paramName = null)
{
    if (argument is null)
        throw new ArgumentNullException(paramName);
}

ThrowIfNull(user);
// throws ArgumentNullException with paramName = "user" (capture expression)
```

`ArgumentNullException.ThrowIfNull` (BCL .NET 6+) использует это.

### 8.4. Logging guards

```csharp
public void LogIf<T>(
    bool condition,
    string message,
    [CallerArgumentExpression(nameof(condition))] string? expr = null)
{
    if (condition) Console.WriteLine($"{expr}: {message}");
}

var x = 42;
LogIf(x > 100, "exceeded");
// printed: "x > 100: exceeded"
```

> [!question]- Интервью: как работает `[CallerMemberName]`?
> Compiler **подставляет** имя caller'а в default parameter на compile-time. Объявление: `void Method([CallerMemberName] string caller = "")`. Caller `Method()` — `caller = "имя_метода_caller'а"`. Используется для INotifyPropertyChanged (не нужно "PropertyName" string), logging (file/line/member auto). Combine с `[CallerFilePath]` и `[CallerLineNumber]`. C# 10+ добавил `[CallerArgumentExpression]` — подставляет expression text (`x > 100` → string `"x > 100"`).

---

## 9. AssemblyInfo и assembly-level

### 9.1. Assembly metadata

```csharp
// AssemblyInfo.cs или Project.csproj generated
[assembly: AssemblyTitle("MyApp")]
[assembly: AssemblyDescription("My great app")]
[assembly: AssemblyVersion("1.0.0.0")]
[assembly: AssemblyFileVersion("1.0.0.0")]
[assembly: AssemblyInformationalVersion("1.0.0-beta")]
[assembly: AssemblyCompany("MyCompany")]
[assembly: AssemblyProduct("MyApp")]
[assembly: AssemblyCopyright("© 2024")]
```

### 9.2. InternalsVisibleTo

```csharp
// Friend assembly — internal члены видны другому assembly
[assembly: InternalsVisibleTo("MyApp.Tests")]
[assembly: InternalsVisibleTo("MyApp.Internal")]
```

Стандарт для tests видеть internal members. Также для split internal API между assemblies.

### 9.3. Module-level

```csharp
[module: SkipLocalsInit]   // perf optimization (no automatic zero-init for stackalloc)
```

### 9.4. Nullable context global

```csharp
[assembly: System.Runtime.CompilerServices.NullableContext(2)]
```

Реже встречается напрямую — обычно `<Nullable>enable</Nullable>` в csproj.

> [!question]- Интервью: зачем `[assembly: InternalsVisibleTo("...")]`?
> Объявляет **friend assemblies**, которым видны `internal` члены этого assembly. Стандартный use case: **unit tests** — test project видит internal classes / methods для testing implementation details. Также: split internal APIs между нескольких assemblies (например, MyApp.Core внутренние члены доступны MyApp.Infrastructure). Cargumentnt — assembly name (без extension). Strong-named assemblies требуют public key. Без InternalsVisibleTo testing internal только через `Reflection` (anti-pattern).

---

## 10. Best Practices

### 10.1. Используй built-in когда возможно

- ✅ `[Required]`, `[StringLength]`, `[Range]` для validation.
- ✅ `[JsonPropertyName]`, `[JsonIgnore]` для serialization.
- ✅ `[Obsolete]` для API evolution.
- ✅ `[Conditional("DEBUG")]` для debug-only methods.
- ❌ Custom attribute для simple bool flag — лучше interface.

### 10.2. Custom attributes

- ✅ **`[AttributeUsage]`** обязательно — define targets.
- ✅ **Suffix `Attribute`** — convention.
- ✅ **Constructor positional params** — required data.
- ✅ **Properties** — optional named arguments.
- ✅ **Inheritable когда нужно** (default true).
- ❌ **Mutable state** в attribute — instances immutable by design.

### 10.3. Performance

- ✅ **Кэшируй reflection** в `Dictionary<Type, T>`.
- ✅ **Source generators** для metadata-heavy scenarios.
- ✅ **`CallerMemberName`** для PropertyChanged без strings.
- ❌ **Reflection в hot path** без cache.
- ❌ **Per-call `GetCustomAttribute`** — slow.

### 10.4. Caller* атрибуты

- ✅ **`[CallerMemberName]`** для INotifyPropertyChanged.
- ✅ **`[CallerArgumentExpression]`** для validation helpers (ThrowIfNull).
- ✅ **`[CallerFilePath/LineNumber]`** для diagnostic logging.
- ❌ **Trust caller-supplied values без validation**.

### 10.5. Не делай

- ❌ Custom attribute где interface работает.
- ❌ Reflection без cache в loops.
- ❌ Side effects в attribute constructor (но constructor не выполняется до GetCustomAttribute).
- ❌ Изменение runtime поведения через attributes без read.

---

## 11. Decision tree

```
Нужно metadata?
│
├── Validation → DataAnnotations ([Required], [Range], custom : ValidationAttribute)
│
├── Serialization → JSON / XML attributes
│
├── Routing / DI → ASP.NET Core attributes
│
├── Compiler hints
│   ├── Deprecation → [Obsolete]
│   ├── Conditional compilation → [Conditional("SYMBOL")]
│   ├── Caller info → [CallerMemberName/FilePath/LineNumber]
│   └── Argument capture → [CallerArgumentExpression]
│
├── Custom metadata
│   ├── Создаёшь свой? Tool/framework для чтения существует?
│   │   ├── Да → custom attribute + reflection или source generator
│   │   └── Нет → interface / regular code
│   ├── [AttributeUsage(...)] обязательно
│   └── Constructor для required, properties для optional
│
└── Performance-critical metadata
    └── Source generators (compile-time, AOT-friendly)
```

---

## 12. Cheat sheet

```csharp
// === Built-in ===
[Obsolete("Use NewMethod")]
public void OldMethod() { }

[Conditional("DEBUG")]
public void Log(string msg) { }

[Flags]
public enum Permissions { None = 0, Read = 1, Write = 2 }

// === Validation ===
[Required, StringLength(100), EmailAddress]
public string Email { get; set; } = "";

// === Custom attribute ===
[AttributeUsage(AttributeTargets.Class | AttributeTargets.Method,
                AllowMultiple = true,
                Inherited = false)]
public class TaggedAttribute(string name) : Attribute
{
    public string Name { get; } = name;
    public string? Description { get; set; }   // optional named
}

[Tagged("important", Description = "Critical path")]
[Tagged("monitored")]
public void Process() { }

// === Reading via reflection ===
var tags = typeof(MyClass).GetCustomAttributes<TaggedAttribute>();
foreach (var t in tags) Console.WriteLine(t.Name);

// === Source generators (consumer) ===
[GeneratedRegex(@"\d+")]
private static partial Regex DigitRegex();

[JsonSerializable(typeof(User))]
internal partial class MyJsonContext : JsonSerializerContext { }

[LoggerMessage(EventId = 100, Level = LogLevel.Information,
               Message = "User {Id} logged in")]
static partial void LogUserLogin(ILogger logger, int id);

// === Caller* ===
public void Trace(
    string msg,
    [CallerMemberName] string m = "",
    [CallerLineNumber] int l = 0) { }

public static void NotNull<T>(
    [NotNull] T? arg,
    [CallerArgumentExpression(nameof(arg))] string? name = null)
{
    if (arg is null) throw new ArgumentNullException(name);
}

// === Assembly-level ===
[assembly: InternalsVisibleTo("MyApp.Tests")]
[assembly: AssemblyVersion("1.0.0.0")]
```

---

## 13. Common Pitfalls

### 13.1. Reflection в hot path без cache

```csharp
foreach (var item in millionItems)
{
    var attr = item.GetType().GetCustomAttribute<MyAttr>();   // ❌ slow!
}
```

**Фикс:** cache в `ConcurrentDictionary<Type, MyAttr?>`.

### 13.2. AttributeUsage забыт

```csharp
public class MyAttribute : Attribute { }
// no [AttributeUsage] — default AttributeTargets.All, AllowMultiple = false
```

**Фикс:** explicit `[AttributeUsage(AttributeTargets.X)]`.

### 13.3. Custom attribute с DateTime в constructor

```csharp
public class BadAttr : Attribute
{
    public BadAttr(DateTime dt) { }   // ❌ DateTime не const
}
```

**Фикс:** string ISO date + parse, или int ticks.

### 13.4. Conditional не работает на returning method

```csharp
[Conditional("DEBUG")]
public int Compute() { return 42; }   // ❌ должен быть void
```

**Фикс:** `void` метод или `#if DEBUG`.

### 13.5. Inherited = true unexpected

```csharp
[MyAttr]
public abstract class Base { }

public class Derived : Base { }
// Derived автоматически имеет MyAttr (inherited)!
```

**Фикс:** `Inherited = false` если не нужно.

### 13.6. AllowMultiple = false default

```csharp
[Tag("a")]
[Tag("b")]   // ❌ compile error если AllowMultiple = false
public class Foo { }
```

**Фикс:** `AllowMultiple = true`.

### 13.7. Attribute constructor с side effects

```csharp
public class BadAttr : Attribute
{
    public BadAttr() { Console.WriteLine("Created"); }   // ⚠ когда вызывается?
}
```

**Механизм:** instance создаётся **lazy** при `GetCustomAttribute<T>()`, не при declaration.

**Фикс:** только данные в constructor.

### 13.8. Source generator + non-partial class

```csharp
public class MyClass   // ❌ no partial — generator не может extend
{
    [ObservableProperty]
    private string _name = "";
}
```

**Фикс:** `public partial class MyClass`.

### 13.9. Caller* с non-default value

```csharp
public void Log(
    string msg,
    [CallerMemberName] string member = "")   // ✅ default — needed
{ }

public void Log(
    string msg,
    [CallerMemberName] string member)   // ❌ no default — caller обязан передать
{ }
```

**Фикс:** Caller* parameters MUST have defaults.

### 13.10. InternalsVisibleTo без strong name

```csharp
[assembly: InternalsVisibleTo("MyApp.Tests")]   // OK для unsigned

// Если assembly signed — нужен PublicKey:
[assembly: InternalsVisibleTo("MyApp.Tests, PublicKey=...")]
```

> [!question]- Интервью: топ-3 ошибки с attributes?
> 1) **Reflection в hot path без cache** — `GetCustomAttribute` slow, в loop становится bottleneck. Cache в `ConcurrentDictionary<Type, T>` или используй source generators. 2) **`[AttributeUsage]` забыт** — default `AttributeTargets.All` (можно везде) и `AllowMultiple = false`. Explicit обязательно. 3) **`[Conditional]` на non-void method** — compile error. Conditional работает только на `void` methods (или ones с `[Conditional]` parameter modifiers).

---

## 14. Practice exercises

### 14.1. Custom validation attribute

```csharp
[AttributeUsage(AttributeTargets.Property)]
public class StrongPasswordAttribute : ValidationAttribute
{
    public int MinLength { get; set; } = 8;
    
    public override bool IsValid(object? value)
    {
        if (value is not string s) return false;
        if (s.Length < MinLength) return false;
        if (!s.Any(char.IsUpper)) return false;
        if (!s.Any(char.IsLower)) return false;
        if (!s.Any(char.IsDigit)) return false;
        return true;
    }
    
    public override string FormatErrorMessage(string name)
        => $"{name} must be at least {MinLength} chars with upper/lower/digit";
}

public class RegisterRequest
{
    [Required, StrongPassword(MinLength = 10)]
    public string Password { get; set; } = "";
}
```

### 14.2. Reflection-based attribute reader с cache

```csharp
public static class AttributeCache<TAttribute> where TAttribute : Attribute
{
    private static readonly ConcurrentDictionary<Type, TAttribute?> _cache = new();
    
    public static TAttribute? Get(Type type) =>
        _cache.GetOrAdd(type, t => t.GetCustomAttribute<TAttribute>());
}

// Использование
var attr = AttributeCache<TrackedAttribute>.Get(typeof(OrderService));
```

### 14.3. Logger helper с CallerArgumentExpression

```csharp
public static class Guard
{
    public static void NotNull<T>(
        [NotNull] T? value,
        [CallerArgumentExpression(nameof(value))] string? name = null) where T : class
    {
        if (value is null) throw new ArgumentNullException(name);
    }
    
    public static void Positive(
        int value,
        [CallerArgumentExpression(nameof(value))] string? name = null)
    {
        if (value <= 0) throw new ArgumentException($"{name} must be positive");
    }
    
    public static void NotEmpty(
        string? value,
        [CallerArgumentExpression(nameof(value))] string? name = null)
    {
        if (string.IsNullOrWhiteSpace(value))
            throw new ArgumentException($"{name} cannot be empty");
    }
}

// Использование
public void Process(string name, int count, User user)
{
    Guard.NotEmpty(name);          // exception: "name cannot be empty"
    Guard.Positive(count);          // "count must be positive"
    Guard.NotNull(user);            // "Value cannot be null. (Parameter 'user')"
}
```

---

## 15. Что читать дальше

1. **Reflection deep** — `Type`, `MethodInfo`, dynamic invocation.
2. **[[debugging-basics|Debugging]]** — `[DebuggerDisplay]`, `[DebuggerStepThrough]`.
3. **Source generators** — pisanieе своих generators.
4. **Roslyn analyzers** — compile-time checks.
5. **DataAnnotations + ASP.NET Core validation**.

---

## 16. См. также

- [[debugging-basics|Debugging]] — debugger attributes
- [[strings-regex|Strings]] — `[GeneratedRegex]`
- [[oop|OOP]] — `[AttributeUsage]` design
- DataAnnotations namespace
- System.Diagnostics.CodeAnalysis attributes
- Roslyn analyzers и source generators

---

## 17. Reading list

- **Microsoft Docs — Attributes** — learn.microsoft.com/dotnet/csharp/programming-guide/concepts/attributes/
- **Microsoft Docs — Creating custom attributes** — learn.microsoft.com/dotnet/standard/attributes/writing-custom-attributes
- **Microsoft Docs — Caller information** — learn.microsoft.com/dotnet/csharp/language-reference/attributes/caller-information
- **Microsoft Docs — Source generators** — learn.microsoft.com/dotnet/csharp/roslyn-sdk/source-generators-overview
- **Andrew Lock — Source generators series** — andrewlock.net
- **Stephen Cleary — Reflection patterns** — blog.stephencleary.com
- **Jon Skeet — C# in Depth (attributes chapter)**
- **CommunityToolkit.Mvvm** — github.com/CommunityToolkit/dotnet
