# Architecture Tests в .NET (NetArchTest)

> По материалам: [Architecture Tests in .NET](https://antondevtips.com/blog/why-do-you-need-to-write-architecture-tests-in-dotnet/)

## Зачем

Архитектура дрейфует: разработчики добавляют зависимости между слоями, нарушают границы модулей. Architecture tests — unit-тесты для структуры. Build падает при нарушении.

## NetArchTest

```bash
dotnet add package NetArchTest.Rules
```

## Примеры правил

### Controllers с [ApiController]

```csharp
var result = Types.InAssembly(ApiAssembly)
    .That().HaveNameEndingWith("Controller")
    .Should().HaveCustomAttribute(typeof(ApiControllerAttribute))
    .GetResult();
```

### Domain entities наследуют Base Entity

```csharp
var result = Types.InAssembly(DomainAssembly)
    .That().ResideInNamespace(DomainNamespace).And().AreClasses()
    .Should().Inherit(typeof(Entity))
    .GetResult();
```

### Clean Architecture: Domain не зависит от других слоёв

```csharp
var result = Types.InAssembly(DomainAssembly)
    .That().ResideInNamespace(Domain)
    .ShouldNot().HaveDependencyOn(Application)
    .AndShouldNot().HaveDependencyOn(Infrastructure)
    .AndShouldNot().HaveDependencyOn(Api)
    .GetResult();
```

### Application не зависит от Infrastructure

```csharp
var result = Types.InAssembly(ApplicationAssembly)
    .That().ResideInNamespace(Application)
    .ShouldNot().HaveDependencyOn(Infrastructure)
    .AndShouldNot().HaveDependencyOn(Api)
    .GetResult();
```

## Modular Monolith

- Модули общаются только через Public API
- Модуль не ссылается на Domain/Infrastructure других модулей
- Тест: `NotHaveDependencyOnAny(CarriersNamespace, StocksNamespace)` для внутренних проектов

## AssemblyReference

В каждом проекте — marker class для загрузки assembly в тестах:

```csharp
public static class AssemblyReference
{
    public static readonly Assembly Assembly = typeof(AssemblyReference).Assembly;
}
```

## CI/CD

Architecture tests — обычные тесты. Запускать в pipeline при сборке.

---

## Best Practices (дополнительно)

- **Именование тестов** — `{Layer}_Should_{Rule}`. Например: `Domain_Should_Not_DependOn_Infrastructure`. Сразу понятно, что сломалось.
- **FormatFailure** — выводить `result.FailingTypeNames` в сообщение. Иначе непонятно, какой класс нарушил правило.
- **Не переусердствовать** — не писать тесты на каждый суффикс. Фокус на зависимостях и границах.
- **Документировать исключения** — если правило намеренно нарушено, `[ExcludeFromCodeCoverage]` или отдельный тест с `Skip`.
- **Скорость** — architecture tests быстрые. Держать в том же job, что и unit-тесты.

---

## См. также

- [[Architecture/architecture-tutorial|Архитектуры]]
- [[Architecture/architecture-conventions-and-tests|Соглашения и тесты]]
- [[Topics/Testing/testing-xunit-testcontainers|Testing]]
