---
tags: [native-aot, trimming, performance, dotnet-10, deployment, source-generators]
level: Senior
date: 2026-08-02
---

# Native AOT в .NET — production guide

> AOT-компиляция в single executable без JIT и CLR ради мгновенного startup и малого footprint; цена — trimming, ограничения reflection и опора на source generators.

## Что это, зачем и когда

### Что такое Native AOT?
**Ahead-of-Time компиляция .NET-приложения в нативный machine code на этапе сборки**, без JIT, без CLR runtime. Получается single executable, который запускается на target-машине без установленного .NET.

**Аналогия:** Обычный .NET — это JavaScript в браузере: код едет вместе с runtime'ом, runtime интерпретирует/JIT'ит. Native AOT — это C++: ты компилируешь в `.exe`, кладёшь на машину, запускается напрямую как любой нативный бинарь.

### Эволюция компиляции в .NET

| | Когда | Что |
|--|-------|-----|
| **JIT** | Default | IL → native в runtime, при первом вызове метода |
| **R2R (ReadyToRun)** | .NET Core 3+ | Pre-compile часть в native, JIT доделывает горячие методы |
| **NativeAOT** | .NET 7+ | Полностью native бинарь, без JIT и runtime |
| **WASM AOT** | .NET 7+ | AOT для Blazor WebAssembly |

### Зачем

| | Standard .NET | Native AOT |
|--|---------------|------------|
| Размер бинаря | 60-100 MB (с runtime) | 5-15 MB |
| Startup time | 200-1000 ms | 5-50 ms (10-100x быстрее) |
| Memory footprint | 50-100 MB baseline | 10-30 MB baseline |
| Throughput peak | Лучше (Tier-2 JIT хорошо оптимизирует) | Чуть хуже (нет dynamic profile-guided optimization) |
| Reflection-heavy code | Работает | Сломается / нужны hints |
| Build time | Быстро | Медленнее (full LLVM-style оптимизации) |
| Debugging | Полная | Ограниченная |

### Когда применять

✅ **Идеально:**
- CLI-утилиты — startup критичен
- Lambda / FaaS / Serverless — cold start от 1 сек до 50 мс
- Container'ы — distroless `gcr.io/distroless/cc-debian12` без .NET
- Embedded — IoT, edge devices
- Native libraries — .NET-код вызывается из C/C++/Rust как `.dll`/`.so`

❌ **Не применяй:**
- Reflection-heavy frameworks (старые ORM, AutoMapper, NHibernate)
- Dynamic plugin-loading (`Assembly.LoadFrom`)
- Code generation в runtime (`Expression.Compile`)
- Большая часть legacy WinForms / WPF (не поддерживается AOT)
- Когда startup и size не критичны (большой web-сервис, который живёт неделями)

---

## Setup

### .csproj

```xml
<Project Sdk="Microsoft.NET.Sdk.Web">
  <PropertyGroup>
    <TargetFramework>net10.0</TargetFramework>
    <PublishAot>true</PublishAot>

    <!-- Опционально, дополнительные оптимизации -->
    <InvariantGlobalization>true</InvariantGlobalization>
    <StripSymbols>true</StripSymbols>
    <IlcOptimizationPreference>Size</IlcOptimizationPreference>  <!-- или Speed -->
    <IlcDisableReflection>true</IlcDisableReflection>             <!-- максимально жёстко -->

    <!-- Включить полные warning'и -->
    <IsTrimmable>true</IsTrimmable>
    <TreatWarningsAsErrors>true</TreatWarningsAsErrors>
    <EnableTrimAnalyzer>true</EnableTrimAnalyzer>
    <EnableAotAnalyzer>true</EnableAotAnalyzer>
  </PropertyGroup>
</Project>
```

### Publish

```bash
dotnet publish -c Release -r linux-x64 -o ./publish
# Или для Windows
dotnet publish -c Release -r win-x64 -o ./publish
```

Результат — single binary в `./publish/MyApp` (Linux) или `MyApp.exe` (Windows). Его можно запустить на любой target-машине **без** установленного .NET.

### Cross-compilation

```bash
# С Linux на Linux ARM64 (например, для Raspberry Pi)
dotnet publish -c Release -r linux-arm64

# Для Mac M-series
dotnet publish -c Release -r osx-arm64
```

Cross-compile с одного OS на другой работает, но требует target-toolchain (clang, ld).

---

## Что ломается в AOT

### Reflection — основной враг

```csharp
// ❌ Падает в runtime AOT
var type = Type.GetType("MyApp.MyClass");           // ok если тип referenced
var instance = Activator.CreateInstance(type);       // RequiresUnreferencedCode warning
var prop = type.GetProperty("Value");                // может быть trim'нуто
prop.SetValue(instance, 42);                         // не работает в AOT

// ✅ Compile-time полиморфизм через generic / pattern matching / source generators
```

Trimmer/AOT compiler не знает что `Type.GetType("MyApp.MyClass")` нужен → удаляет неиспользуемый код. В runtime — `null` или `MissingMethodException`.

### Dynamic code generation

```csharp
// ❌ Не работает
Expression<Func<int, int>> expr = x => x * 2;
var compiled = expr.Compile();  // RequiresDynamicCode

// ❌ Также — IL Emit
var dyn = new DynamicMethod(...);
```

`System.Linq.Expressions.Expression.Compile()` использует runtime IL emission — невозможно в AOT.

### `Assembly.LoadFrom` — plugin-системы

```csharp
// ❌ Разрешает загружать произвольные .dll в runtime
var asm = Assembly.LoadFrom("plugin.dll");

// ✅ AOT — все assemblies должны быть статически известны
```

Если плагины критичны — выноси их в **отдельные процессы** (см. [[ipc-named-pipes-grpc|IPC patterns]]).

### JSON serialization — нужен JsonSerializerContext

```csharp
// ❌ Reflection-based — не работает в AOT
var json = JsonSerializer.Serialize(myObject);

// ✅ Source-generated context
[JsonSerializable(typeof(MyType))]
[JsonSerializable(typeof(List<MyType>))]
public partial class AppJsonContext : JsonSerializerContext { }

var json = JsonSerializer.Serialize(myObject, AppJsonContext.Default.MyType);
```

См. [[source-generators|Source Generators]] — AOT и SG взаимосвязаны, без SG большинство фреймворков сломается.

### Configuration binding

```csharp
// .csproj
<EnableConfigurationBindingGenerator>true</EnableConfigurationBindingGenerator>
```

Без этой опции `configuration.Get<MyOptions>()` использует reflection → not AOT-friendly.

### EF Core

EF Core 8+ имеет частичную AOT-совместимость, но **не полную** (LINQ → SQL translation использует Expression trees). Стратегии:

```csharp
// ✅ Compiled queries (заранее транслированные)
private static readonly Func<AppDbContext, Guid, Task<User?>> GetUserById =
    EF.CompileAsyncQuery((AppDbContext db, Guid id) =>
        db.Users.FirstOrDefault(u => u.Id == id));

await GetUserById(_db, userId);

// Или
// ✅ Dapper.AOT / raw ADO.NET — без LINQ Expression Trees (vanilla Dapper падает: Reflection.Emit)
await using var conn = await _dataSource.OpenConnectionAsync(ct);
var users = await conn.QueryAsync<User>("SELECT * FROM users WHERE active = true");
```

Полная AOT-совместимая alternative — `Microsoft.Data.Sqlite` / `Npgsql` напрямую.

---

## Trim warnings и как с ними работать

При publish AOT компилятор выдаёт warnings вроде:
```
warning IL2026: Using member 'X' which has 'RequiresUnreferencedCodeAttribute'
warning IL3050: Using member 'Y' which has 'RequiresDynamicCodeAttribute'
```

Это значит: Trimmer/AOT compiler не уверен что метод/тип будет работать в trimmed mode.

### Что делать

**1. Найти AOT-friendly альтернативу:**
- AutoMapper → Mapperly
- FluentValidation → частичная AOT-поддержка с 12.x, полной гарантии нет — либо ручная валидация
- AutoFixture → ручные test-builders
- NewtonSoft.Json → System.Text.Json + JsonSerializerContext

**2. Аннотировать API явно:**
```csharp
[RequiresUnreferencedCode("Calls reflection-based serializer. Not AOT-friendly.")]
public static T Deserialize<T>(string json)
{
    return JsonSerializer.Deserialize<T>(json)!;
}
```

Это документирует что метод не AOT-safe и signal'ит callers.

**3. `DynamicallyAccessedMembers` для метаданных:**
```csharp
public static T Create<[DynamicallyAccessedMembers(DynamicallyAccessedMemberTypes.PublicConstructors)] T>()
{
    return Activator.CreateInstance<T>();
}
```

Trimmer гарантирует что public ctors типа `T` сохранятся.

**4. Suppress warnings (только когда уверен):**
```csharp
[UnconditionalSuppressMessage("Trimming", "IL2026", Justification = "We checked: type is preserved")]
public void Foo() { /* reflection */ }
```

Suppress = ты берёшь ответственность что в runtime не упадёт.

---

## Производительность — реальные числа

Замеры на ASP.NET Core minimal-API (.NET 10, тривиальный JSON-endpoint):

| | Standard JIT | NativeAOT |
|--|--------------|-----------|
| Cold start | 350 ms | 25 ms |
| Memory after startup | 75 MB | 22 MB |
| Single-file size | 95 MB | 12 MB |
| Throughput sustained | 480k RPS | 410k RPS |
| GC frequency | Baseline | Чуть чаще (нет Tier-2 escape analysis) |

**Ключевая выгода — startup и memory.** Throughput на длинной дистанции hours-running чуть проигрывает (нет PGO + Tier-2 JIT), но в большинстве задач разница 10-15% незначительна.

### Где AOT драматически выигрывает
- **Lambda cold start**: было 800ms — стало 30ms. Экономия $$$
- **CLI tools**: с AOT работает как `git`, без AOT — 200ms tax на каждый вызов
- **Sidecars / agents**: 20MB вместо 100MB на каждом pod'е → существенно для density

---

## ASP.NET Core с AOT

### Минимальный AOT API

```csharp
var builder = WebApplication.CreateSlimBuilder(args);  // Slim — без JSON-reflection и др.

builder.Services.ConfigureHttpJsonOptions(options =>
{
    options.SerializerOptions.TypeInfoResolverChain.Insert(0, AppJsonContext.Default);
});

var app = builder.Build();

app.MapGet("/health", () => new HealthResponse("healthy"));
app.MapPost("/echo", (EchoRequest req) => new EchoResponse(req.Message));

app.Run();

public sealed record HealthResponse(string Status);
public sealed record EchoRequest(string Message);
public sealed record EchoResponse(string Message);

[JsonSerializable(typeof(HealthResponse))]
[JsonSerializable(typeof(EchoRequest))]
[JsonSerializable(typeof(EchoResponse))]
public partial class AppJsonContext : JsonSerializerContext { }
```

`CreateSlimBuilder` — облегчённая версия: меньше регистраций по дефолту, без middleware которые требуют reflection.

### Что работает в ASP.NET Core AOT

| ✅ Работает | ❌ Не работает / частично |
|------------|---------------------------|
| Minimal API | Controllers (требуют reflection для discovery) |
| JSON через `JsonSerializerContext` | XmlSerializer |
| Authentication / Authorization | OData |
| Routing | SignalR (частично) |
| Middleware (большинство) | gRPC reflection / server reflection |
| OpenAPI generation | Razor pages (некоторые компоненты) |

### Controllers в AOT
Технически работают, но не рекомендуется. Используй Minimal API + `IEndpoint`-pattern.

```csharp
// IEndpoint-pattern (см. AspNetCore/api-design.md)
public interface IEndpoint
{
    void MapEndpoint(IEndpointRouteBuilder app);
}

public sealed class GetUsersEndpoint : IEndpoint
{
    public void MapEndpoint(IEndpointRouteBuilder app) =>
        app.MapGet("/users", async (UserService svc, CancellationToken ct) =>
            await svc.GetAllAsync(ct));
}

// Регистрация — каждый endpoint руками (без сканирования assembly через reflection)
app.MapEndpoint<GetUsersEndpoint>();
```

---

## Console / CLI приложение с AOT

```csharp
// Program.cs
public class Program
{
    public static async Task<int> Main(string[] args)
    {
        var rootCommand = new RootCommand("My CLI tool");

        var greetCommand = new Command("greet", "Greet someone");
        var nameOption = new Option<string>("--name", () => "World");
        greetCommand.AddOption(nameOption);

        greetCommand.SetHandler((name) =>
        {
            Console.WriteLine($"Hello, {name}!");
        }, nameOption);

        rootCommand.AddCommand(greetCommand);

        return await rootCommand.InvokeAsync(args);
    }
}
```

Размер итогового binary — 6-8MB на Linux. Startup — < 20ms.

```bash
dotnet publish -c Release -r linux-x64 -o ./out
ls -la ./out/MyApp  # 7.2 MB
time ./out/MyApp greet --name Vitaly
# real    0m0.014s  ← 14ms!

```

---

## Native libraries — .NET-код как `.so`/`.dll`

С AOT можно писать .NET-код, использовать его из C/C++/Rust:

```xml
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <TargetFramework>net10.0</TargetFramework>
    <PublishAot>true</PublishAot>
    <NativeLib>Shared</NativeLib>  <!-- или Static -->
    <RootNamespace>MyLib</RootNamespace>
    <AssemblyName>mylib</AssemblyName>
  </PropertyGroup>
</Project>
```

```csharp
using System.Runtime.InteropServices;

public static class Exports
{
    [UnmanagedCallersOnly(EntryPoint = "add")]
    public static int Add(int a, int b) => a + b;

    [UnmanagedCallersOnly(EntryPoint = "compute_hash")]
    public static long ComputeHash(IntPtr data, int length)
    {
        unsafe
        {
            var span = new ReadOnlySpan<byte>(data.ToPointer(), length);
            return XxHash64.HashToUInt64(span);
        }
    }
}
```

```bash
dotnet publish -c Release -r linux-x64
# Получаем mylib.so (Linux) или mylib.dll (Windows)

```

Из C:
```c
#include <stdio.h>
#include <dlfcn.h>

int main() {
    void* handle = dlopen("./mylib.so", RTLD_LAZY);
    int (*add)(int, int) = dlsym(handle, "add");
    printf("%d\n", add(2, 3));  // 5
    return 0;
}
```

Используется в реальных проектах: AOT-скомпилированные .NET-библиотеки для интеграции в Python/Rust/Go без переписывания.

---

## Containers с AOT

### Distroless image

```dockerfile
# Build stage
FROM mcr.microsoft.com/dotnet/sdk:10.0 AS build
WORKDIR /src
COPY *.csproj .
RUN dotnet restore
COPY . .
RUN dotnet publish -c Release -r linux-x64 -o /app

# Final stage — distroless
FROM gcr.io/distroless/cc-debian12 AS final
WORKDIR /app
COPY --from=build /app/MyApp ./
ENTRYPOINT ["./MyApp"]
```

Финальный image — **15-25 MB** вместо 200-300 MB обычного `dotnet:10.0-aspnet`. Дистро contains только linker (libc, libgcc) — без shell, package manager, debug-tools. Гораздо меньше attack surface.

### Image size comparison

| | Standard | AOT |
|--|----------|-----|
| `dotnet:10.0-aspnet` | 220 MB | — |
| `dotnet:10.0-aspnet-chiseled` | 110 MB | — |
| Distroless `gcr.io/distroless/cc-debian12` + AOT binary | — | 18 MB |
| `scratch` + statically-linked AOT | — | 8 MB |

---

## Common pitfalls

### 1. NuGet пакет работает в JIT, ломается в AOT
Проверка: ищи в repo `RequiresUnreferencedCode` / `RequiresDynamicCode` атрибуты, или просто Google "X AOT compatibility".

Известно совместимы (.NET 10, 2026):
- ✅ ASP.NET Core (Minimal API)
- ✅ EF Core (с compiled queries)
- ✅ Dapper.AOT (build-time interceptors; vanilla Dapper — ❌, Reflection.Emit)
- ✅ Npgsql, Microsoft.Data.SqlClient
- ✅ Polly v8, Microsoft.Extensions.Http.Resilience
- ✅ Serilog, OpenTelemetry
- ✅ Mediator (martinothamar, source-generated: пакеты Mediator.SourceGenerator + Mediator.Abstractions) — AOT-дружественная замена MediatR
- ✅ Mapperly
- ✅ CommunityToolkit.Mvvm

Несовместимо или partial:
- ❌ MediatR — reflection + assembly scanning для discovery handler'ов, AOT-совместимости нет (use source-generated Mediator)
- ⚠️ Старый AutoMapper (use Mapperly)
- ⚠️ FluentValidation — частичная поддержка с 12.x (убрано assembly scanning), полной гарантии Native AOT нет: ядро на expression trees, под AOT они интерпретируются
- ❌ Newtonsoft.Json (use STJ)
- ❌ NHibernate (heavy reflection)

### 2. JSON deserialize в `dynamic`

```csharp
// ❌ AOT — runtime exception
var data = JsonSerializer.Deserialize<dynamic>(json);

// ✅ Известные типы
var data = JsonSerializer.Deserialize<MyDto>(json, AppJsonContext.Default.MyDto);

// ✅ JsonElement если структура динамическая
var element = JsonSerializer.Deserialize<JsonElement>(json);
```

### 3. Generic + reflection

```csharp
// ❌ Trimmer не знает какие T применятся
public static List<T> Convert<T>(string json) => JsonSerializer.Deserialize<List<T>>(json);

// ✅ Явная подсказка через DynamicallyAccessedMembers
public static List<T> Convert<[DynamicallyAccessedMembers(DynamicallyAccessedMemberTypes.PublicConstructors | ...)] T>(string json) => ...
```

### 4. Локализация / Culture
По умолчанию AOT включает `InvariantGlobalization=true` — нет support для других культур (datetime parsing, decimal separators, sorting). Если нужна локализация:
```xml
<InvariantGlobalization>false</InvariantGlobalization>
```

Это добавит ICU-data — увеличит size на ~20MB.

### 5. Большой build time
Полный AOT publish может занимать минуты для большого проекта. На CI:
- Кэшируй `obj/` директорию между builds
- Параллельная сборка через `IlcMaxParallelism`
- Используй RyuJIT-only для PR builds, full AOT — только для releases

### 6. Hard-to-debug runtime errors
В AOT нет stack-trace symbols по умолчанию. Если что-то ломается:
```xml
<DebugSymbols>true</DebugSymbols>
<StripSymbols>false</StripSymbols>
```
Это добавит symbols → удобнее crash dumps анализировать. Но size возрастает.

### 7. `Activator.CreateInstance` в DI
Старые DI-контейнеры использовали reflection для создания instances → не AOT-safe. Microsoft.Extensions.DependencyInjection в .NET 8+ поддерживает source-generated DI:
```csharp
// Не нужно ничего особенного — встроенный DI .NET 8+ AOT-friendly
builder.Services.AddSingleton<IService, Service>();
builder.Services.AddTransient<IRepo, Repo>();
```

### 8. `IConfiguration.Bind<T>(...)` без generator
Включи `<EnableConfigurationBindingGenerator>true</EnableConfigurationBindingGenerator>` — тогда binding source-генерируется, не reflection.

---

## Production checklist

- [ ] `<PublishAot>true</PublishAot>` в .csproj
- [ ] `<EnableTrimAnalyzer>true</EnableTrimAnalyzer>` + `<EnableAotAnalyzer>true</EnableAotAnalyzer>`
- [ ] `<TreatWarningsAsErrors>true</TreatWarningsAsErrors>` — fail build на trim warnings
- [ ] `JsonSerializerContext` для всех serialization
- [ ] `EnableConfigurationBindingGenerator` если используешь `Get<T>()` / `Bind<T>()`
- [ ] EF Core compiled queries или прямой Npgsql/Dapper
- [ ] Все NuGet packages проверены на AOT compatibility
- [ ] Smoke-test AOT build на CI (не только JIT) для каждого PR
- [ ] Distroless / scratch base image для container
- [ ] Crash-handling symbols enabled на staging
- [ ] Performance baseline зафиксирован (cold start, throughput, memory)

---

## Troubleshooting workflow

### "Падает в runtime, в JIT работает"

1. Запусти с `<EnableTrimAnalyzer>true</EnableTrimAnalyzer>` и пересобери — увидишь warnings
2. Найди warnings IL2026/IL3050 в нужном code path
3. Замени reflection-based API на AOT-friendly (source-gen, generic, manual)
4. Если third-party — проверь updates / search GitHub Issues "AOT"
5. Last resort — wrap в `RequiresUnreferencedCode` / `UnconditionalSuppressMessage` (но тогда тестируй runtime!)

### "Slow build"

- `dotnet publish -c Release --verbosity normal` — посмотреть где тормозит
- Сократить assembly (split на subprojects)
- Параллелизация: `IlcMaxParallelism` MSBuild property

### "Binary слишком большой"

- `<IlcOptimizationPreference>Size</IlcOptimizationPreference>`
- Удалить unused features через features switches:
  ```xml
  <RuntimeHostConfigurationOption Include="System.Globalization.Invariant" Value="true" />
  <RuntimeHostConfigurationOption Include="System.Diagnostics.Tracing.Tracing" Value="false" />
  <RuntimeHostConfigurationOption Include="System.GC.Concurrent" Value="false" />
  ```
- Trim unused dependencies — `<TrimMode>full</TrimMode>` (default — link, оставляет всё)

---

## См. также

- [[source-generators|Source Generators]] — фундамент AOT (без них большинство frameworks ломается)
- [[compilation-jit|Compilation и JIT]] — как работает обычная JIT-компиляция
- [[modern-features|Modern C# 8–14]] — primary constructors, collection expressions (всё AOT-friendly)
- [[project-setup|Project Setup]] — современная структура с AOT-готовностью
- [[docker|Docker]] — distroless containers + AOT

## Reading list

- **Microsoft Learn — Native AOT** — learn.microsoft.com/dotnet/core/deploying/native-aot/
- **Trimming docs** — learn.microsoft.com/dotnet/core/deploying/trimming/
- **AOT compatibility checklist** — github.com/dotnet/runtime/blob/main/docs/design/coreclr/botr/aot-deployment.md
- **Andrew Lock — AOT Series** — andrewlock.net (несколько глубоких постов)
- **Egil Hansen — AOT для библиотек** — практические патерны для NuGet maintainers
- **dotnet/runtime issues с тегом `nativeaot`** — все edge cases и workarounds
