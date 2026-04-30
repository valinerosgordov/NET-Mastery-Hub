---
tags: [source-generators, roslyn, codegen, performance, mapperly, logger-message, regex, json]
level: Senior
---

# Source Generators в .NET — production guide

## Что это, зачем и когда

### Что такое source generator?
**Roslyn-based компонент, который во время компиляции анализирует твой код и добавляет в проект сгенерированные .cs файлы.** Они компилируются вместе с твоим кодом — сначала генерация, потом основной компилятор всё это собирает.

**Аналогия:** Раньше был code-runner типа T4-templates — нужно было руками запускать generation и коммитить результат. Source generators — это автоматический T4 встроенный в build-pipeline. Пишешь partial-метод с атрибутом — генератор дописывает реализацию. Меняется код — реализация переписывается. Без коммитов в git, без отдельных шагов.

### Зачем

| Без source generators | С source generators |
|----------------------|---------------------|
| Reflection в runtime — медленно, alloc, AOT-несовместимо | Сгенерированный код — JIT'ится как обычный, AOT-friendly |
| Boilerplate INotifyPropertyChanged — 5 строк на каждое property | `[ObservableProperty]` — 1 атрибут |
| AutoMapper / FluentValidation — runtime-reflection, ломаются на rename | Mapperly — compile-time проверки, ошибка в IDE |
| `string.Format("{0} {1}", a, b)` в logging — boxing на value-types | `LoggerMessage` source-generator — zero-alloc |
| Ручные DTO ↔ Domain converters | Source-generated mapper |

### Эволюция

| | ISourceGenerator (.NET 5) | IIncrementalGenerator (.NET 6+) |
|--|---------------------------|--------------------------------|
| Когда запускается | Каждый раз заново на полный проект | Только когда меняется input — incremental |
| Performance в IDE | Тормозило при наборе кода | Быстро, не тормозит intellisense |
| API | Простой | Pipeline-based (декларативный) |
| Когда применять | **Не используй**, deprecated подход | Стандарт |

В современном .NET — только `IIncrementalGenerator`.

### Когда применять / не применять

✅ **Применяй когда:**
- Boilerplate, который повторяется много раз (`INotifyPropertyChanged` props, mapper-методы)
- Performance critical (logging, regex, JSON)
- Native AOT compatibility — без reflection
- Type-safe APIs (compile-time check вместо runtime exception)

❌ **Не применяй когда:**
- Простая задача решается одной abstract class / generic
- Нужно динамическое поведение (плагинная система)
- Отдельный preprocessing шаг (используй MSBuild target / Directory.Build.props)
- Решение нужно один раз — рефактор может занять полдня, источников и тестов больше чем самого кода

---

## Готовые source generators — что использовать в production

### CommunityToolkit.Mvvm — ObservableProperty / RelayCommand

```csharp
// Было (без generator):
public class TaskViewModel : INotifyPropertyChanged
{
    private string _title = "";
    public string Title
    {
        get => _title;
        set
        {
            if (_title != value)
            {
                _title = value;
                PropertyChanged?.Invoke(this, new PropertyChangedEventArgs(nameof(Title)));
                OnPropertyChanged(nameof(IsValid));  // зависимое свойство
                CompleteCommand.NotifyCanExecuteChanged();
            }
        }
    }
    public bool IsValid => !string.IsNullOrWhiteSpace(Title);
    public ICommand CompleteCommand => _completeCommand ??= new RelayCommand(...);
    public event PropertyChangedEventHandler? PropertyChanged;
    private void OnPropertyChanged(string p) => PropertyChanged?.Invoke(this, new(p));
}

// Стало (с generator):
public sealed partial class TaskViewModel : ObservableObject
{
    [ObservableProperty]
    [NotifyPropertyChangedFor(nameof(IsValid))]
    [NotifyCanExecuteChangedFor(nameof(CompleteCommand))]
    private string _title = "";

    public bool IsValid => !string.IsNullOrWhiteSpace(Title);

    [RelayCommand(CanExecute = nameof(IsValid))]
    private async Task CompleteAsync(CancellationToken ct) { /* ... */ }
}
```

Подробнее в [WPF Production](wpf-production.md).

### LoggerMessage — high-performance logging

```csharp
public partial class OrderService
{
    [LoggerMessage(
        Level = LogLevel.Information,
        Message = "Order {OrderId} placed for user {UserId}, total {Total:C}")]
    private partial void LogOrderPlaced(Guid orderId, Guid userId, decimal total);

    public async Task PlaceAsync(Order order, CancellationToken ct)
    {
        await _repo.SaveAsync(order, ct);
        LogOrderPlaced(order.Id, order.UserId, order.Total);
    }
}
```

Source generator создаёт implementation, который:
- **Не делает boxing** value-types (decimal, Guid)
- **Не форматирует строку** если уровень выключен (LogLevel.Trace в проде)
- Статическая `EventId` для каждого сообщения — удобно фильтровать в Application Insights / Seq

Замеры: source-generated logging **в 5-10x быстрее** обычного `_logger.LogInformation("...")` на горячем пути с числовыми параметрами.

> [!question]- **Интервью: почему `_logger.LogInformation("Value: {V}", value)` медленнее source-generated?**
> 1. **Аргументы боксятся** — `value` (например, `int`) превращается в `object` для `params object?[] args`
> 2. **Создаётся `params` array** — heap allocation на каждый вызов
> 3. **`Message` парсится каждый раз** — lookup placeholders {V}
> 4. **Нет early-exit** на `IsEnabled(level)` — передача аргументов уже произошла
>
> Source-generator решает всё это: typed параметры, `IsEnabled` check встроенный, no params array.

### Microsoft.Extensions.Configuration.Binder source generator

В .NET 8+ есть генератор для `Configuration.Bind<T>()` — без reflection. Включается через:

```csharp
// .csproj
<PropertyGroup>
    <EnableConfigurationBindingGenerator>true</EnableConfigurationBindingGenerator>
</PropertyGroup>
```

Это критично для **Native AOT** — без него `IConfiguration.Bind<T>()` не работает (использует reflection).

### System.Text.Json — JsonSerializerContext

```csharp
[JsonSerializable(typeof(Order))]
[JsonSerializable(typeof(List<OrderItem>))]
[JsonSerializable(typeof(ApiResponse<Order>))]
public partial class AppJsonContext : JsonSerializerContext { }

// Использование
var json = JsonSerializer.Serialize(order, AppJsonContext.Default.Order);
var order = JsonSerializer.Deserialize<Order>(json, AppJsonContext.Default.Order);

// В ASP.NET Core
builder.Services.ConfigureHttpJsonOptions(opt =>
    opt.SerializerOptions.TypeInfoResolverChain.Insert(0, AppJsonContext.Default));
```

Без `JsonSerializerContext` — runtime reflection (медленно + AOT-incompatible). С ним — pre-generated сериализация: 2-5x быстрее, AOT-ready.

### Regex source-generated

```csharp
public partial class EmailValidator
{
    [GeneratedRegex(@"^[\w\.\-]+@[\w\.\-]+\.\w+$", RegexOptions.Compiled | RegexOptions.IgnoreCase)]
    private static partial Regex EmailRegex();

    public bool IsValid(string email) => EmailRegex().IsMatch(email);
}
```

Раньше: `new Regex(...)` — компиляция в IL в runtime.
Теперь: исходный код regex генерируется во время сборки. Преимущества:
- Faster startup (нет runtime IL emission)
- AOT compatible
- Можно посмотреть сгенерированный код

### Riok.Mapperly — typed mappings

```csharp
[Mapper]
public partial class OrderMapper
{
    public partial OrderDto OrderToDto(Order source);
    public partial Order DtoToOrder(OrderDto source);
    public partial List<OrderDto> OrdersToDto(List<Order> source);

    // Кастомизация частных случаев
    [MapProperty(nameof(Order.Items), nameof(OrderDto.LineItems))]
    public partial OrderDto OrderToDto(Order source);
}
```

vs AutoMapper:
- AutoMapper — runtime reflection, ошибки конфигурации в runtime, медленно, **commercial** с 2024
- Mapperly — compile-time generation, ошибки в IDE, fast, MIT, AOT-friendly

В NexusAI — Mapperly зафиксирован в `Locked Choices`.

### CommunityToolkit.HighPerformance.GeneratedDeconstruct

Меньше известный, но удобный:
```csharp
[GeneratedDeconstruct]
public partial record Point(int X, int Y, int Z);
// Без GeneratedDeconstruct — Deconstruct(out int x, out int y, out int z) надо писать руками для record class
```

---

## Свой source generator — пошагово

### Setup проекта

```xml
<!-- MyGenerator.csproj -->
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <TargetFramework>netstandard2.0</TargetFramework>  <!-- ОБЯЗАТЕЛЬНО netstandard2.0 -->
    <LangVersion>latest</LangVersion>
    <Nullable>enable</Nullable>
    <IsRoslynComponent>true</IsRoslynComponent>
    <EnforceExtendedAnalyzerRules>true</EnforceExtendedAnalyzerRules>
  </PropertyGroup>

  <ItemGroup>
    <PackageReference Include="Microsoft.CodeAnalysis.CSharp" Version="4.8.0" PrivateAssets="all" />
    <PackageReference Include="Microsoft.CodeAnalysis.Analyzers" Version="3.3.4" PrivateAssets="all" />
  </ItemGroup>
</Project>
```

`netstandard2.0` обязательно — Roslyn-генераторы хостятся в Visual Studio (которая до сих пор на .NET Framework / mixed runtime).

В консьюмере подключи как analyzer:
```xml
<ItemGroup>
  <ProjectReference Include="..\MyGenerator\MyGenerator.csproj"
                    OutputItemType="Analyzer"
                    ReferenceOutputAssembly="false" />
</ItemGroup>
```

### Простейший пример: `[AutoToString]`

Цель — генерируем красивый `ToString()` на класс с атрибутом:

```csharp
// Атрибут (этот код в общей библиотеке-атрибутов, ставится в проект пользователя)
[AttributeUsage(AttributeTargets.Class)]
public sealed class AutoToStringAttribute : Attribute { }

// Применение
[AutoToString]
public partial class Order
{
    public Guid Id { get; init; }
    public string Customer { get; init; } = "";
    public decimal Total { get; init; }
}

// Сгенерированный код (что должен сделать наш генератор)
partial class Order
{
    public override string ToString() =>
        $"Order(Id={Id}, Customer=\"{Customer}\", Total={Total})";
}
```

### Сам генератор

```csharp
using Microsoft.CodeAnalysis;
using Microsoft.CodeAnalysis.CSharp.Syntax;
using Microsoft.CodeAnalysis.Text;
using System.Text;

[Generator]
public sealed class AutoToStringGenerator : IIncrementalGenerator
{
    public void Initialize(IncrementalGeneratorInitializationContext context)
    {
        // Step 1: атрибут регистрируем в compilation
        context.RegisterPostInitializationOutput(static ctx =>
        {
            ctx.AddSource("AutoToStringAttribute.g.cs", """
                namespace MyApp;

                [System.AttributeUsage(System.AttributeTargets.Class)]
                internal sealed class AutoToStringAttribute : System.Attribute { }
                """);
        });

        // Step 2: pipeline — собираем все классы с нашим атрибутом
        var classes = context.SyntaxProvider
            .ForAttributeWithMetadataName(
                fullyQualifiedMetadataName: "MyApp.AutoToStringAttribute",
                predicate: static (node, _) => node is ClassDeclarationSyntax,
                transform: static (ctx, _) => GetClassInfo(ctx))
            .Where(static x => x is not null)
            .Select(static (x, _) => x!.Value);

        // Step 3: для каждого — генерируем файл
        context.RegisterSourceOutput(classes, static (ctx, cls) =>
        {
            var source = GenerateToString(cls);
            ctx.AddSource($"{cls.ClassName}.AutoToString.g.cs", SourceText.From(source, Encoding.UTF8));
        });
    }

    private static ClassInfo? GetClassInfo(GeneratorAttributeSyntaxContext ctx)
    {
        if (ctx.TargetSymbol is not INamedTypeSymbol classSymbol) return null;

        var properties = classSymbol.GetMembers()
            .OfType<IPropertySymbol>()
            .Where(p => p.DeclaredAccessibility == Accessibility.Public && !p.IsStatic)
            .Select(p => new PropertyInfo(p.Name, p.Type.ToDisplayString()))
            .ToImmutableArray();

        return new ClassInfo(
            Namespace: classSymbol.ContainingNamespace.ToDisplayString(),
            ClassName: classSymbol.Name,
            Properties: properties);
    }

    private static string GenerateToString(ClassInfo cls)
    {
        var sb = new StringBuilder();
        sb.AppendLine($"namespace {cls.Namespace};");
        sb.AppendLine();
        sb.AppendLine($"partial class {cls.ClassName}");
        sb.AppendLine("{");
        sb.AppendLine("    public override string ToString() =>");

        var args = string.Join(", ", cls.Properties.Select(p =>
        {
            var format = p.TypeName == "string" ? $"{p.Name}=\"{{{p.Name}}}\"" : $"{p.Name}={{{p.Name}}}";
            return format;
        }));

        sb.AppendLine($"        $\"{cls.ClassName}({args})\";");
        sb.AppendLine("}");
        return sb.ToString();
    }

    // Records — equatable structurally → incremental кэш работает корректно
    private sealed record ClassInfo(string Namespace, string ClassName, ImmutableArray<PropertyInfo> Properties);
    private sealed record PropertyInfo(string Name, string TypeName);
}
```

### Ключевые моменты

**`ForAttributeWithMetadataName`** — современный (с .NET 8 / Roslyn 4.4) ускоренный API для поиска по атрибутам. Используй его вместо `CreateSyntaxProvider` где можно — в 100x быстрее.

**Records для cache keys** — Roslyn не пересоздаёт output если input не изменился. Чтобы сравнение работало, твои intermediate-структуры (`ClassInfo`, `PropertyInfo`) должны быть value-equatable. Records делают это автоматически.

**Не таскай через pipeline `INamedTypeSymbol`/`SyntaxNode`** — они тяжёлые, держат компиляцию. Извлекай нужное (имена, типы как строки) в transform-step и пробрасывай дальше только value-объекты.

---

## Debug source generator

### launchSettings.json

```json
{
  "profiles": {
    "MyGenerator (Debug)": {
      "commandName": "DebugRoslynComponent",
      "targetProject": "..\\ConsumerProject\\ConsumerProject.csproj"
    }
  }
}
```

В Visual Studio → выбираешь профиль → F5 → запускается компиляция consumer-проекта с подключённым debugger'ом к генератору. Можно ставить breakpoints в `Initialize`, `GetClassInfo`, etc.

### Снапшот-тестирование

```csharp
public class GeneratorTests
{
    [Fact]
    public Task ToString_Generated_For_Class_With_Attribute()
    {
        var source = """
            using MyApp;

            namespace TestNamespace;

            [AutoToString]
            public partial class TestClass
            {
                public string Name { get; init; } = "";
                public int Age { get; init; }
            }
            """;

        return TestHelper.Verify(source);
    }
}

public static class TestHelper
{
    public static Task Verify(string source)
    {
        var syntaxTree = CSharpSyntaxTree.ParseText(source);
        var compilation = CSharpCompilation.Create(
            "Test",
            new[] { syntaxTree },
            references: AppDomain.CurrentDomain.GetAssemblies()
                .Where(a => !a.IsDynamic)
                .Select(a => MetadataReference.CreateFromFile(a.Location)));

        var driver = CSharpGeneratorDriver
            .Create(new AutoToStringGenerator())
            .RunGenerators(compilation);

        return Verifier.Verify(driver);  // Verify (Verify.Xunit)
    }
}
```

`Verify` — записывает первый run в `*.received.txt`, последующие сверяет. Изменения в генераторе = изменения в snapshot, надо подтвердить через CLI.

---

## Pitfalls

### 1. `INamedTypeSymbol` равенства

```csharp
// ❌ Symbol equality из разных compilations не равны
.Where(s => s == otherSymbol)

// ✅ Используй SymbolEqualityComparer
.Where(s => SymbolEqualityComparer.Default.Equals(s, otherSymbol))
```

### 2. Generated файлы не обновляются в IDE
Кешируется в Visual Studio. Лечится перезапуском или:
```
Tools → Options → Text Editor → C# → Advanced → Show generated source files: True
```

### 3. Slow generator ломает intellisense
Генератор запускается на каждое изменение файла. Если он работает 500ms — IDE начинает тормозить. Профилируй через `EnableSourceGeneratorTracing` в build:
```xml
<PropertyGroup>
  <EmitCompilerGeneratedFiles>true</EmitCompilerGeneratedFiles>
  <CompilerGeneratedFilesOutputPath>$(BaseIntermediateOutputPath)Generated</CompilerGeneratedFilesOutputPath>
</PropertyGroup>
```

### 4. Roslyn API breaking changes
Microsoft.CodeAnalysis API меняется между мажорами. Зафиксируй версию (4.8.x) и **не** обновляй без необходимости — апдейт может сломать генератор.

### 5. Доступ к файлам / сети из генератора
**Запрещено.** Source generator должен быть deterministic: одинаковый input = одинаковый output. Запросы к БД, файловой системе, networking — всё это сделает build не воспроизводимым на CI.

### 6. Crash в генераторе ломает build
Любой uncaught exception в `Initialize` падает в виде `error CS8785: Generator 'X' failed to generate source`. Без stack trace в build log'е. Wrappни критичные участки в `try/catch` и репортируй через `Diagnostic`:

```csharp
context.RegisterSourceOutput(classes, static (ctx, cls) =>
{
    try { /* generation */ }
    catch (Exception ex)
    {
        ctx.ReportDiagnostic(Diagnostic.Create(
            new DiagnosticDescriptor(
                "MYGEN001", "Generator error", $"Failed: {ex.Message}",
                "MyGenerator", DiagnosticSeverity.Error, true),
            location: null));
    }
});
```

---

## Native AOT compatibility

Source generators **критичны** для Native AOT. Без них compiler не знает что нужно сохранить (всё что использует reflection — выкидывается trimmer'ом).

```xml
<!-- .csproj -->
<PropertyGroup>
  <PublishAot>true</PublishAot>
  <InvariantGlobalization>true</InvariantGlobalization>
</PropertyGroup>
```

Что **нужно**:
- ✅ `JsonSerializerContext` (system.text.json source-gen)
- ✅ `LoggerMessage` (logging source-gen)
- ✅ `[GeneratedRegex]`
- ✅ Mapperly (типизированный маппинг без reflection)
- ✅ Все custom-генераторы которые ты написал

Что **запрещено**:
- ❌ `Activator.CreateInstance(Type.GetType("..."))`
- ❌ `Assembly.Load`
- ❌ `Type.MakeGenericType` для типов которые компилятор не видел
- ❌ `Expression.Compile()` (большая часть LINQ-to-SQL под этим)

Когда компилируешь AOT — следи за warnings `IL3050` (RequiresDynamicCode), `IL2026` (RequiresUnreferencedCode). Это сигнал что reflection-based код не выдержит trim.

---

## Когда написать свой генератор vs использовать готовый

| Need | Pick |
|------|------|
| INPC + commands | CommunityToolkit.Mvvm |
| Logging | Microsoft.Extensions.Logging.Abstractions (`[LoggerMessage]`) |
| Mapping (DTO ↔ domain) | Riok.Mapperly |
| JSON serialization | System.Text.Json `JsonSerializerContext` |
| Regex | `[GeneratedRegex]` |
| Configuration binding (AOT) | `EnableConfigurationBindingGenerator` |
| Equality / hash code | C# 9+ records — нативно |
| Свой DSL / domain-specific codegen | Свой генератор |

90% задач закрываются готовыми. Свой пишешь когда есть domain-specific повторяющийся паттерн в проекте (на NexusAI — например, генератор endpoint registration по `[Endpoint]` атрибуту).

---

## Common pitfalls (повтор для checklist)

- [ ] `IIncrementalGenerator` (не legacy `ISourceGenerator`)
- [ ] `netstandard2.0` для генератора
- [ ] Records для intermediate types (cache equality)
- [ ] `ForAttributeWithMetadataName` вместо `CreateSyntaxProvider` для поиска по атрибутам
- [ ] Не таскай Symbol/SyntaxNode дальше первого transform-шага
- [ ] Symbol equality через `SymbolEqualityComparer.Default`
- [ ] Диагностики через `ctx.ReportDiagnostic(...)` для ошибок (не throw)
- [ ] Никаких I/O / сети из генератора (deterministic build!)
- [ ] Snapshot-tests для генератора (Verify.Xunit)
- [ ] launchSettings.json для отладки

---

## См. также

- [Modern C# 8–14](../CSharp/modern-features.md) — `partial properties`, `partial methods`, что нужно для генераторов
- [WPF Production](wpf-production.md) — реальное применение CommunityToolkit.Mvvm генераторов
- [Logging и Observability](../AspNetCore/logging-observability.md) — LoggerMessage в production
- [Performance](../Performance/performance.md) — почему source-gen быстрее reflection
- [HFT / Low-Latency](../Performance/hft-low-latency.md) — где зеро-allocation logging критичен
- [Code Quality](../Quality/code-quality.md) — analyzers и source-generators в одной экосистеме

## Reading list

- **Andrew Lock — Source Generators series** — andrewlock.net/series/creating-a-source-generator/ (15+ статей, фундаментальный материал)
- **Roslyn Cookbook** — github.com/dotnet/roslyn/blob/main/docs/features/source-generators.cookbook.md
- **IIncrementalGenerator design doc** — github.com/dotnet/roslyn/blob/main/docs/features/incremental-generators.md
- **Source Generator Samples** — github.com/dotnet/roslyn-sdk/tree/main/samples/CSharp/SourceGenerators
- **Stefan Pölz blog** — поглубже про edge cases, performance в IDE
- **Mapperly source** — github.com/riok/mapperly (отличный пример sophisticated генератора с богатой конфигурацией)
