---
tags: [architecture, netarchtest, conventions, testing]
level: Senior
---

# Architecture Tests в .NET (NetArchTest)

## Что это, зачем и когда

### Что такое Architecture Tests?
**Unit-тесты для структуры кода**, а не для логики. Проверяют, что слои не нарушают границы, зависимости идут в правильном направлении, нейминг соблюдается.

**Аналогия:** Инспектор на стройке. Код может работать правильно, но если электрик протянул провод через несущую стену — это нарушение. Architecture tests ловят такие «нарушения строительных норм».

### Зачем?

| Без Architecture Tests | С Architecture Tests |
|------------------------|---------------------|
| Разработчик добавил `using Infrastructure` в Domain — никто не заметил | Build падает: «Domain не должен зависеть от Infrastructure» |
| Нейминг дрейфует: `OrderService`, `OrderManager`, `OrderHelper` | Тест: все сервисы заканчиваются на `Service` |
| Модуль A лезет в кишки модуля B | Тест: модули общаются только через публичные контракты |

### Когда нужны?
- Проект с Clean Architecture / любым layered подходом
- Команда > 1 человека (один обязательно нарушит границу)
- Долгоживущий проект (архитектура деградирует со временем)
- **НЕ нужны:** маленькие утилиты, скрипты, прототипы

---

> По материалам: [Architecture Tests in .NET](https://antondevtips.com/blog/why-do-you-need-to-write-architecture-tests-in-dotnet/)

## Зачем

Архитектура дрейфует: разработчики добавляют зависимости между слоями, нарушают границы модулей. Architecture tests — unit-тесты для структуры кода. Build падает при нарушении правил.

**Что ловят:**
- Domain зависит от Infrastructure (прямая ссылка на DbContext, HttpClient)
- Controller содержит бизнес-логику вместо делегирования
- Модуль A напрямую обращается к внутренностям модуля B
- Нарушения naming conventions (сервис без суффикса Service, handler без Handler)

## Установка

```bash
dotnet add package NetArchTest.Rules
```

---

## Примеры правил

### Controllers с [ApiController]

```csharp
[Fact]
public void Controllers_Should_Have_ApiControllerAttribute()
{
    var result = Types.InAssembly(ApiAssembly)
        .That().HaveNameEndingWith("Controller")
        .Should().HaveCustomAttribute(typeof(ApiControllerAttribute))
        .GetResult();

    Assert.True(result.IsSuccessful,
        $"Controllers without [ApiController]: {FormatFailing(result)}");
}
```

### Domain entities наследуют BaseEntity

```csharp
[Fact]
public void DomainEntities_Should_Inherit_BaseEntity()
{
    var result = Types.InAssembly(DomainAssembly)
        .That().ResideInNamespace($"{DomainNamespace}.Entities")
        .And().AreClasses()
        .And().AreNotAbstract()
        .Should().Inherit(typeof(Entity))
        .GetResult();

    Assert.True(result.IsSuccessful,
        $"Entities not inheriting Entity: {FormatFailing(result)}");
}
```

### Clean Architecture: Domain не зависит от других слоёв

```csharp
[Fact]
public void Domain_Should_Not_DependOn_OtherLayers()
{
    var result = Types.InAssembly(DomainAssembly)
        .That().ResideInNamespace(DomainNamespace)
        .ShouldNot().HaveDependencyOnAny(
            ApplicationNamespace,
            InfrastructureNamespace,
            ApiNamespace)
        .GetResult();

    Assert.True(result.IsSuccessful,
        $"Domain depends on: {FormatFailing(result)}");
}
```

### Application не зависит от Infrastructure / API

```csharp
[Fact]
public void Application_Should_Not_DependOn_Infrastructure()
{
    var result = Types.InAssembly(ApplicationAssembly)
        .That().ResideInNamespace(ApplicationNamespace)
        .ShouldNot().HaveDependencyOnAny(
            InfrastructureNamespace,
            ApiNamespace)
        .GetResult();

    Assert.True(result.IsSuccessful,
        $"Application depends on: {FormatFailing(result)}");
}
```

### Interfaces живут в Domain/Application, не в Infrastructure

```csharp
[Fact]
public void Interfaces_Should_Not_Reside_In_Infrastructure()
{
    var result = Types.InAssembly(InfrastructureAssembly)
        .That().AreInterfaces()
        .ShouldNot().ResideInNamespace(InfrastructureNamespace)
        .GetResult();

    // Интерфейсы определяем в Domain/Application, реализации — в Infrastructure
    Assert.True(result.IsSuccessful);
}
```

---

## Naming Conventions

```csharp
[Fact]
public void CommandHandlers_Should_Be_Sealed()
{
    var result = Types.InAssembly(ApplicationAssembly)
        .That().ImplementInterface(typeof(IRequestHandler<,>))
        .Should().BeSealed()
        .GetResult();

    Assert.True(result.IsSuccessful,
        $"Unsealed handlers: {FormatFailing(result)}");
}

[Fact]
public void Repositories_Should_EndWith_Repository()
{
    var result = Types.InAssembly(InfrastructureAssembly)
        .That().ImplementInterface(typeof(IRepository<>))
        .Should().HaveNameEndingWith("Repository")
        .GetResult();

    Assert.True(result.IsSuccessful);
}
```

---

## Modular Monolith

В модульном монолите каждый модуль — изолированный bounded context. Модули общаются только через публичные контракты.

```csharp
[Fact]
public void OrdersModule_Should_Not_DependOn_OtherModules_Internals()
{
    var result = Types.InAssembly(OrdersAssembly)
        .That().ResideInNamespace("App.Modules.Orders")
        .ShouldNot().HaveDependencyOnAny(
            "App.Modules.Carriers.Domain",
            "App.Modules.Carriers.Infrastructure",
            "App.Modules.Stocks.Domain",
            "App.Modules.Stocks.Infrastructure")
        .GetResult();

    Assert.True(result.IsSuccessful,
        $"Orders module leaks into: {FormatFailing(result)}");
}

// Разрешена зависимость только на контракты (IntegrationEvents, Abstractions)
[Fact]
public void Module_Can_DependOn_SharedContracts()
{
    var result = Types.InAssembly(OrdersAssembly)
        .That().HaveDependencyOn("App.Modules.Carriers.Contracts")
        .Should().Exist()  // это допустимо
        .GetResult();
}
```

---

## AssemblyReference — маркер для тестов

В каждом проекте добавить маркерный класс для удобного доступа к Assembly:

```csharp
// В Domain проекте
namespace App.Domain;
public static class AssemblyReference
{
    public static readonly Assembly Assembly = typeof(AssemblyReference).Assembly;
}

// В Application проекте
namespace App.Application;
public static class AssemblyReference
{
    public static readonly Assembly Assembly = typeof(AssemblyReference).Assembly;
}

// В тестах
private static readonly Assembly DomainAssembly = App.Domain.AssemblyReference.Assembly;
private static readonly Assembly ApplicationAssembly = App.Application.AssemblyReference.Assembly;
```

---

## Helper для форматирования ошибок

```csharp
private static string FormatFailing(TestResult result)
    => result.FailingTypeNames is null
        ? "none"
        : string.Join(", ", result.FailingTypeNames);
```

**Нюанс:** без `FormatFailing` при падении теста вы увидите только «False should be True» — бесполезно. Всегда выводите `FailingTypeNames`.

---

## Полный пример тестового класса

```csharp
public class ArchitectureTests
{
    private static readonly Assembly DomainAssembly = Domain.AssemblyReference.Assembly;
    private static readonly Assembly ApplicationAssembly = Application.AssemblyReference.Assembly;
    private static readonly Assembly InfrastructureAssembly = Infrastructure.AssemblyReference.Assembly;
    private static readonly Assembly ApiAssembly = Api.AssemblyReference.Assembly;

    private const string DomainNamespace = "App.Domain";
    private const string ApplicationNamespace = "App.Application";
    private const string InfrastructureNamespace = "App.Infrastructure";

    [Fact]
    public void Domain_Should_Not_DependOn_OtherLayers()
    {
        var result = Types.InAssembly(DomainAssembly)
            .ShouldNot().HaveDependencyOnAny(
                ApplicationNamespace, InfrastructureNamespace, "App.Api")
            .GetResult();

        Assert.True(result.IsSuccessful, FormatFailing(result));
    }

    [Fact]
    public void Application_Should_Not_DependOn_Infrastructure()
    {
        var result = Types.InAssembly(ApplicationAssembly)
            .ShouldNot().HaveDependencyOnAny(InfrastructureNamespace, "App.Api")
            .GetResult();

        Assert.True(result.IsSuccessful, FormatFailing(result));
    }

    [Fact]
    public void Handlers_Should_Be_Internal_And_Sealed()
    {
        var result = Types.InAssembly(ApplicationAssembly)
            .That().HaveNameEndingWith("Handler")
            .Should().BeSealed()
            .And().NotBePublic()
            .GetResult();

        Assert.True(result.IsSuccessful, FormatFailing(result));
    }

    private static string FormatFailing(TestResult result)
        => result.FailingTypeNames is null
            ? string.Empty
            : string.Join(Environment.NewLine, result.FailingTypeNames);
}
```

---

## CI/CD

Architecture tests — обычные unit-тесты. Быстрые (миллисекунды). Запускать в том же job, что и unit-тесты.

```yaml
# GitHub Actions
- name: Run Architecture Tests
  run: dotnet test tests/ArchitectureTests --no-build
```

---

## Best Practices

- **Именование тестов** — `{Layer}_Should_{Rule}`. Например: `Domain_Should_Not_DependOn_Infrastructure`. Сразу понятно, что сломалось.
- **FormatFailure** — выводить `result.FailingTypeNames` в сообщение assert. Иначе непонятно, какой класс нарушил правило.
- **Не переусердствовать** — фокус на зависимостях и границах модулей. Не тестировать каждый суффикс.
- **Sealed handlers** — MediatR handlers как `internal sealed` — защита от случайного наследования и видимости.
- **Документировать исключения** — если правило намеренно нарушено, комментарий или отдельный тест с `Skip`.
- **Скорость** — architecture tests выполняются за миллисекунды. Держать вместе с unit-тестами.

---

## См. также

- [Архитектуры](patterns.md)
- [CQRS и MediatR](cqrs-mediatr.md)
- [Testing](../Topics/Testing/testing-xunit-testcontainers.md)
