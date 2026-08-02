---
tags: [architecture, testing, netarchtest, archunit, fitness-functions]
level: Senior
date: 2026-08-02
---

# Архитектурные тесты

> Правила архитектуры (Domain не зависит от Infrastructure, naming conventions, module boundaries) как исполняемые тесты в CI через NetArchTest.eNhancedEdition / ArchUnitNET — fitness functions, ловящие нарушения слоёв до merge.

## Что это, зачем и когда

### Что такое architecture tests?
**Тесты, проверяющие архитектурные правила кодом.** Domain не зависит от Infrastructure, controllers не используют DbContext напрямую, все entities в namespace Domain — таких правил много, забыть легко. Architecture tests — code review автоматически.

**Аналогия:** Если style rules проверяет linter, то "правильность взаимодействий между слоями" должны проверять architecture tests. Иначе через 2 года cooler dev сорвёт твою Clean Architecture.

### Зачем

| Без arch tests | С arch tests |
|----------------|-------------|
| "Никто не вызывает infrastructure из domain" — на честное слово | Проверяется в CI |
| Через 6 месяцев новые devs не знают границ | Тесты — живая документация |
| Код-ревью пропускает violations | Tests падают сразу при PR |
| Refactor сломал слои → нашли когда уже всё лежит | Падение в CI прежде чем merge |
| Onboarding "вот наша архитектура" в Confluence | Onboarding "вот тесты, читай" |

### Когда применять

| Применять | Не применять |
|-----------|--------------|
| Clean / Hexagonal / VSA архитектура | Простой CRUD без слоёв |
| Команда > 3 человек | Solo project |
| Codebase > 10K LOC | Прототип / MVP |
| Strict layering rules | Простой 1-tier |

---

## ⚠️ NetArchTest — оригинал unmaintained

**NetArchTest** (от ben.dechrai/BenMorris) был стандартом 2018-2023. С 2023 — **минимальная активность**, последние коммиты годовалые. Issues висят.

### Альтернативы

| Tool | Статус | Особенности |
|------|--------|-------------|
| **NetArchTest.eNhancedEdition** | Active fork (2024+) | Drop-in replacement, активная разработка, fluent API того же стиля |
| **ArchUnitNET** | Active | Port от Java ArchUnit, более мощный API, поддерживает SOLID checks из коробки |
| **NsDepCop** | Active | Namespace dependency checks через config files |
| **Roslyn analyzers** | Active | Свой analyzer — но писать сложнее |

В новых проектах:
- **`NetArchTest.eNhancedEdition`** — если знаком с NetArchTest API
- **`ArchUnitNET`** — если хочется большую мощь и SOLID checks built-in

В существующих с NetArchTest — мигрируй на eNhancedEdition (drop-in).

```bash
dotnet add package NetArchTest.eNhancedEdition
# или
dotnet add package ArchUnitNET.xUnit
```

> [!question]- **Интервью: что делать если используем NetArchTest, а он unmaintained?**
> 1. **eNhancedEdition fork** — 100% совместимый API, активная разработка
> 2. **ArchUnitNET** — более мощный, требует переписать тесты
> 3. **Свой Roslyn analyzer** — для команд с capacity и сложными правилами
>
> Главное — мигрируй с original NetArchTest, особенно если security CVE появятся в его dependencies.

---

## NetArchTest.eNhancedEdition — базовые правила

```csharp
using NetArchTest.Rules;

public class ArchitectureTests
{
    private const string DomainNamespace = "MyApp.Domain";
    private const string InfrastructureNamespace = "MyApp.Infrastructure";
    private const string ApiNamespace = "MyApp.Api";

    private static readonly Assembly DomainAssembly = typeof(Order).Assembly;

    [Fact]
    public void Domain_ShouldNotDependOn_Infrastructure()
    {
        var result = Types.InAssembly(DomainAssembly)
            .That()
            .ResideInNamespace(DomainNamespace)
            .ShouldNot()
            .HaveDependencyOn(InfrastructureNamespace)
            .GetResult();

        result.IsSuccessful.ShouldBeTrue(
            $"Domain has dependency on Infrastructure: {string.Join(", ", result.FailingTypes ?? [])}");
    }

    [Fact]
    public void Domain_ShouldNotDependOn_Api()
    {
        var result = Types.InAssembly(DomainAssembly)
            .That()
            .ResideInNamespace(DomainNamespace)
            .ShouldNot()
            .HaveDependencyOn(ApiNamespace)
            .GetResult();

        result.IsSuccessful.ShouldBeTrue();
    }
}
```

---

## Production rules — реалистичные

### 1. Layered Architecture rules

```csharp
[Fact]
public void Domain_ShouldNotDependOnExternalLibraries()
{
    var result = Types.InAssembly(DomainAssembly)
        .That().ResideInNamespace("MyApp.Domain")
        .ShouldNot()
        .HaveDependencyOnAny(
            "Microsoft.AspNetCore",
            "Microsoft.EntityFrameworkCore",
            "Newtonsoft.Json",
            "AutoMapper",
            "MediatR")
        .GetResult();

    result.IsSuccessful.ShouldBeTrue();
}

[Fact]
public void Application_ShouldNotDependOn_Infrastructure()
{
    // Application слой может вызвать абстракции, но не реализации
    var result = Types.InAssembly(typeof(Application.IUseCase).Assembly)
        .That().ResideInNamespace("MyApp.Application")
        .ShouldNot()
        .HaveDependencyOn("MyApp.Infrastructure")
        .GetResult();

    result.IsSuccessful.ShouldBeTrue();
}

[Fact]
public void Controllers_ShouldNotDependOn_Domain_Directly()
{
    // Controllers идут через Application/MediatR, не лезут прямо в Domain
    var result = Types.InAssembly(typeof(Program).Assembly)
        .That()
        .ResideInNamespace("MyApp.Api.Controllers")
        .ShouldNot()
        .HaveDependencyOn("MyApp.Domain.Entities")
        .GetResult();

    result.IsSuccessful.ShouldBeTrue();
}
```

### 2. Naming conventions

```csharp
[Fact]
public void Repositories_ShouldHaveNameEndingWith_Repository()
{
    var result = Types.InAssembly(InfrastructureAssembly)
        .That()
        .ImplementInterface(typeof(IRepository<>))
        .Should()
        .HaveNameEndingWith("Repository")
        .GetResult();

    result.IsSuccessful.ShouldBeTrue();
}

[Fact]
public void RequestHandlers_ShouldHaveNameEndingWith_Handler()
{
    var result = Types.InAssembly(ApplicationAssembly)
        .That()
        .ImplementInterface(typeof(IRequestHandler<,>))
        .Should()
        .HaveNameEndingWith("Handler")
        .GetResult();

    result.IsSuccessful.ShouldBeTrue();
}

[Fact]
public void Validators_ShouldBeSealedAndHaveCorrectName()
{
    var result = Types.InAssembly(ApplicationAssembly)
        .That()
        .Inherit(typeof(AbstractValidator<>))
        .Should()
        .BeSealed()
        .And()
        .HaveNameEndingWith("Validator")
        .GetResult();

    result.IsSuccessful.ShouldBeTrue();
}
```

### 3. Public API hygiene

```csharp
[Fact]
public void DomainEntities_ShouldNotHavePublicSetters()
{
    // Domain entities не должны иметь setter — только через методы
    var result = Types.InAssembly(DomainAssembly)
        .That().ResideInNamespace("MyApp.Domain.Entities")
        .Should()
        .OnlyHavePublicProperties(p => !p.GetSetMethod()?.IsPublic == true)
        .GetResult();

    result.IsSuccessful.ShouldBeTrue();
}

[Fact]
public void Records_ShouldBeSealed()
{
    // Records должны быть sealed (нет inheritance)
    var result = Types.InAssembly(DomainAssembly)
        .That()
        .AreClasses()
        .And()
        .HaveNameEndingWith("Dto", "Request", "Response", "Command", "Query", "Event")
        .Should()
        .BeSealed()
        .GetResult();

    result.IsSuccessful.ShouldBeTrue();
}
```

### 4. Async / sync rules

```csharp
[Fact]
public void RepositoryMethods_ShouldEndWithAsync()
{
    var result = Types.InAssembly(InfrastructureAssembly)
        .That()
        .HaveNameEndingWith("Repository")
        .Should()
        .OnlyHavePublicMethods(m =>
            m.IsConstructor || m.Name.EndsWith("Async") || m.Name == "Dispose")
        .GetResult();

    result.IsSuccessful.ShouldBeTrue();
}

[Fact]
public void Application_ShouldNot_UseGetAwaiterGetResult()
{
    // Detect sync over async — anti-pattern
    // (Через assembly inspection — нужен IL анализ. Лучше Roslyn analyzer)
}
```

### 5. Module boundaries в modular monolith

```csharp
[Fact]
public void OrdersModule_ShouldNotDependOn_OtherModules()
{
    // Modular monolith — модули не лезут в друг друга кроме через published events / API
    var result = Types.InAssembly(OrdersAssembly)
        .That()
        .ResideInNamespace("MyApp.Modules.Orders")
        .ShouldNot()
        .HaveDependencyOnAny(
            "MyApp.Modules.Users.Internal",
            "MyApp.Modules.Inventory.Internal",
            "MyApp.Modules.Notifications.Internal")
        .GetResult();

    result.IsSuccessful.ShouldBeTrue();
}
```

Module-boundary тест в CI ловит запрещённую зависимость кодом, но не отвечает на вопрос «кто должен ревьюить cross-context изменение». Дополни его авто-генерируемым `CODEOWNERS` (путь модуля → владеющая команда): тогда правка в чужом контексте требует approve от его команды на уровне PR, а не только зелёного теста. Генерируй файл из той же карты модулей, что использует arch-тест, чтобы две защиты не разъезжались.

```text
# CODEOWNERS — auto-generated from the module map
/src/Modules/Orders/         @org/orders-team
/src/Modules/Inventory/      @org/inventory-team
/src/Modules/Notifications/  @org/notifications-team
```

### 6. Forbidden namespaces

```csharp
[Fact]
public void ApplicationCode_ShouldNotUse_ConsoleWriteLine()
{
    var result = Types.InAssembly(ApplicationAssembly)
        .ShouldNot()
        .HaveDependencyOn("System.Console")
        .GetResult();

    result.IsSuccessful.ShouldBeTrue();
}

[Fact]
public void Production_ShouldNotUse_DebuggerBreak()
{
    var result = Types.InAssembly(ApplicationAssembly)
        .ShouldNot()
        .HaveDependencyOn("System.Diagnostics.Debugger")
        .GetResult();

    result.IsSuccessful.ShouldBeTrue();
}
```

---

## ArchUnitNET — alternative

```bash
dotnet add package ArchUnitNET.xUnit
```

```csharp
using ArchUnitNET.Domain;
using ArchUnitNET.Loader;
using ArchUnitNET.xUnit;
using static ArchUnitNET.Fluent.ArchRuleDefinition;

public class ArchitectureTests
{
    private static readonly Architecture Architecture =
        new ArchLoader().LoadAssemblies(
            typeof(Order).Assembly,
            typeof(OrderRepository).Assembly,
            typeof(Program).Assembly).Build();

    [Fact]
    public void Domain_ShouldNotDependOn_Infrastructure()
    {
        var domainLayer = Types().That().ResideInNamespace("MyApp.Domain.*", true).As("Domain Layer");
        var infrastructureLayer = Types().That().ResideInNamespace("MyApp.Infrastructure.*", true).As("Infrastructure Layer");

        IArchRule rule = Types()
            .That().Are(domainLayer)
            .Should().NotDependOnAny(infrastructureLayer)
            .Because("Domain должен быть independent");

        rule.Check(Architecture);
    }

    [Fact]
    public void Aggregates_ShouldHavePrivateSetters()
    {
        var aggregateRoots = Classes().That()
            .ImplementInterface("IAggregateRoot")
            .As("Aggregate Roots");

        IArchRule rule = MemberRules.MembersThat()
            .AreDeclaredIn(aggregateRoots)
            .And().AreProperties()
            .Should().NotBePublic()
            .Because("Aggregates manage state through methods");

        rule.Check(Architecture);
    }
}
```

ArchUnitNET преимущества:
- Полнее API (cycles detection, SOLID checks, member-level rules)
- Архитектурный stack описывается декларативно
- Cycles detection между слоями

NetArchTest преимущества:
- Проще API
- Меньше overhead
- Достаточно для большинства правил

---

## Fitness Functions

**Architecture fitness functions** — концепция от книги "Building Evolutionary Architectures" (Neal Ford). Это **measurable architectural goals** — automated check'и которые ensure architecture goals.

### Categories

| Category | Что проверяют | Tools |
|----------|--------------|-------|
| **Atomic** | Single architecture concern | NetArchTest, ArchUnit |
| **Holistic** | Несколько concerns вместе | Custom integration tests |
| **Triggered** | По событию (deploy, schedule) | CI/CD jobs |
| **Continual** | Постоянно (production monitoring) | OpenTelemetry, alerting |
| **Static** | Code analysis | Roslyn, static linter |
| **Dynamic** | Runtime behavior | Load tests, chaos engineering |

### Practical fitness functions

```csharp
// Atomic: layer dependencies
[Fact]
public void Domain_DoesNotDependOn_Infrastructure() { /* NetArchTest */ }

// Holistic: cyclic dependencies
[Fact]
public void NoCyclicDependencies()
{
    var result = Types.InAssembly(typeof(Program).Assembly)
        .Should()
        .NotHaveDependencyOnAny("...")  // Проверка cycles
        .GetResult();
}

// Triggered: deploy time check
// CI step:
// dotnet test --filter Category=Architecture
// dotnet test --filter Category=Performance

// Continual: production monitoring
// OpenTelemetry alert: p99 latency > 500ms (architectural SLO)

// Static: complexity threshold
// SonarQube Cloud — fail PR if cyclomatic complexity > 15

// Dynamic: load test — система держит target RPS
// k6 / NBomber в CI на staging
```

### Документирование fitness functions

В ADR (Architecture Decision Record) фиксируй:
```markdown
## ADR-005: Layered Architecture Boundaries

### Decision
Domain layer must not depend on Infrastructure or Web layers.

### Fitness function
File: `tests/ArchitectureTests/LayeredTests.cs`
- `Domain_ShouldNotDependOn_Infrastructure()`
- `Domain_ShouldNotDependOn_Web()`

These run on every PR.

### Status
Accepted, 2026-04-28.
```

---

## Custom Roslyn analyzers

Когда NetArchTest/ArchUnit недостаточно (нужен check уровня IL / syntax):

```csharp
[DiagnosticAnalyzer(LanguageNames.CSharp)]
public class ForbidGetAwaiterGetResultAnalyzer : DiagnosticAnalyzer
{
    public static readonly DiagnosticDescriptor Rule = new(
        "MYARCH001",
        "Forbid sync-over-async",
        "Don't use GetAwaiter().GetResult() — use await instead",
        "Architecture",
        DiagnosticSeverity.Error,
        isEnabledByDefault: true);

    public override ImmutableArray<DiagnosticDescriptor> SupportedDiagnostics => [Rule];

    public override void Initialize(AnalysisContext context)
    {
        context.ConfigureGeneratedCodeAnalysis(GeneratedCodeAnalysisFlags.None);
        context.EnableConcurrentExecution();
        context.RegisterSyntaxNodeAction(AnalyzeInvocation, SyntaxKind.InvocationExpression);
    }

    private static void AnalyzeInvocation(SyntaxNodeAnalysisContext context)
    {
        var invocation = (InvocationExpressionSyntax)context.Node;
        if (invocation.Expression is MemberAccessExpressionSyntax member &&
            member.Name.Identifier.Text == "GetResult" &&
            member.Expression is InvocationExpressionSyntax inner &&
            inner.Expression is MemberAccessExpressionSyntax innerMember &&
            innerMember.Name.Identifier.Text == "GetAwaiter")
        {
            context.ReportDiagnostic(Diagnostic.Create(Rule, invocation.GetLocation()));
        }
    }
}
```

См. подробно [[source-generators|Source Generators]] — Roslyn API similar к analyzers.

---

## Когда что выбирать

| Need | Tool |
|------|------|
| Layer dependencies, naming, basic rules | NetArchTest.eNhancedEdition |
| SOLID checks, cycles, member-level | ArchUnitNET |
| Forbidden API usage (Console.WriteLine, etc.) | Roslyn analyzer + EditorConfig |
| Code complexity, cyclomatic | SonarQube Cloud / SonarQube for IDE |
| Forbidden namespaces / packages | NsDepCop |
| Forbidden imports / using | Roslyn analyzer |

В большинстве проектов — комбинация:
- **NetArchTest** для layer rules
- **SonarQube Cloud** для quality metrics
- **EditorConfig + .NET analyzers** для style/code quality
- **Custom Roslyn analyzers** для domain-specific patterns

---

## CI integration

```yaml
# .github/workflows/ci.yml
- name: Architecture Tests
  run: dotnet test --filter "Category=Architecture" --logger "console;verbosity=detailed"
```

Все architecture tests должны быть с категорией:
```csharp
[Fact, Trait("Category", "Architecture")]
public void Domain_DoesNotDependOn_Infrastructure() { /* ... */ }
```

CI fail → PR не merge'ится. Architecture violations не доходят до main branch.

---

## Performance considerations

Architecture tests **долгие** — load all assemblies, analyze IL. Single test может выполняться секунду-две.

### Оптимизации

```csharp
// 1. Cache loaded architecture (ArchUnitNET)
private static readonly Architecture Architecture = new ArchLoader()
    .LoadAssemblies(...)
    .Build();
// Singleton — переиспользуется между тестами

// 2. Группировать в один тест
[Fact]
public void AllLayerRules()
{
    Types.InAssembly(Domain).That()...
    Types.InAssembly(Domain).That()...
    Types.InAssembly(Domain).That()...
    // Все правила в одном Fact — быстрее чем 10 Facts с reload
}

// 3. Run только при PR на main
// .gitignore arch test results
```

Не делай arch tests частью regular test run если они занимают минуты — отдельный CI job.

---

## Common pitfalls

### 1. Слишком строгие правила

```csharp
// ❌ "Domain не использует ничего кроме System"
.HaveDependencyOnly("System")
```
Domain может использовать basic .NET (System.Linq, System.Threading). Не ограничивай чрезмерно.

### 2. Tests без понятного error message

```csharp
// ❌
result.IsSuccessful.ShouldBeTrue();

// ✅ — what failed
result.IsSuccessful.ShouldBeTrue(
    $"Domain has dependency on Infrastructure: {string.Join(", ", result.FailingTypes ?? [])}");
```

### 3. Architecture test и unit test в одном проекте
Проблема: arch tests load main assembly + reflection. Случайно нарушают изоляцию unit tests.
**Решение:** отдельный test project `ArchitectureTests.csproj`.

### 4. Игнор failing tests
"Этот тест unstable, мы его пока скипаем" — арх правила сломаны, никто это не контролирует.
**Решение:** либо fix, либо удали правило явно.

### 5. Не запускать в CI
Arch tests существуют, но никто не запускает → правила не работают.
**Решение:** обязательный gate в PR pipeline.

### 6. Documenting в Confluence/wiki
"У нас Clean Architecture, см. doc.pdf" — не enforced.
**Решение:** правило existence == arch test. Wiki — только high-level overview.

### 7. Использовать original NetArchTest в 2026
Project unmaintained, security risk через transitive dependencies.
**Решение:** mig на eNhancedEdition.

### 8. Слишком много кастомных правил
50 architecture tests с кастомным синтаксисом → новый dev не понимает что нарушать опасно.
**Решение:** 5-10 ключевых правил, каждое с comment'ом и ADR-ссылкой.

### 9. Test проверяет имя класса вместо behavior

```csharp
// ❌ "Все классы заканчивающиеся на Service в Application"
.HaveNameEndingWith("Service")

// ✅ Behavior через interface
.ImplementInterface(typeof(IApplicationService))
```

### 10. Игнорировать разницу между assembly references и actual usage
Type из ref'нутой assembly не значит код её **использует**. NetArchTest различает — assembly reference vs runtime call. Проверь docs какой uses какой.

---

## Production checklist

- [ ] NetArchTest.eNhancedEdition или ArchUnitNET (не original NetArchTest)
- [ ] Layer dependencies проверяются (Domain не зависит от Infrastructure)
- [ ] Naming conventions tests (Repositories, Handlers, Validators)
- [ ] Public API hygiene (entities без public setters)
- [ ] Tests по категории "Architecture" — отдельный test run в CI
- [ ] Каждое правило связано с ADR (документация)
- [ ] Failing tests не "игнорируются", а fix'ятся или удаляются
- [ ] Architecture tests gate PRs to main
- [ ] Complexity / quality metrics через SonarQube Cloud / SonarQube for IDE
- [ ] Custom Roslyn analyzers для domain-specific (если нужно)

---

## См. также

- [[architecture-patterns|Architecture Patterns]] — Clean / Hexagonal / VSA — что проверяешь
- [[ddd|DDD на практике]] — domain rules для arch tests
- [[solid|SOLID + DRY/KISS/YAGNI]] — рулы которые arch tests могут проверить
- [[cqrs-mediatr|CQRS и Mediator]] — handler naming conventions
- [[code-quality|Code Quality]] — analyzers, EditorConfig, SonarQube Cloud
- [[architecture-decisions|Architecture Decisions]] — ADR formats
- [[testing|Testing]] — общие подходы к тестированию

## Reading list

- **Building Evolutionary Architectures** — Neal Ford, Rebecca Parsons (canonical book on fitness functions)
- **NetArchTest.eNhancedEdition** — github.com/BenMorris/NetArchTest (active fork)
- **ArchUnitNET docs** — archunitnet.readthedocs.io
- **ArchUnit (Java original)** — archunit.org (концепции универсальны)
- **Microsoft — Roslyn analyzers** — learn.microsoft.com/dotnet/csharp/roslyn-sdk/
- **Andrew Lock — Architecture tests** — andrewlock.net (несколько постов)
- **Steve Smith** — ardalis.com (часто пишет про arch tests как fitness functions)
