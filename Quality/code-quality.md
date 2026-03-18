---
tags: [code-quality, analyzers, sonarqube, editorconfig]
level: Senior
---

# Best Practices for Increasing Code Quality in .NET

## Что это, зачем и когда

### Что такое Code Quality?
**Набор практик и инструментов**, которые автоматически следят за чистотой, консистентностью и безопасностью кода. Не полагаемся на «договорённости» — инструменты проверяют.

**Аналогия:** Автокорректор в Word. Ты можешь писать правильно, но автокорректор ловит опечатки, которые ты не заметил. Code quality tools — автокорректор для кода.

### Зачем?

| Без инструментов | С инструментами |
|-----------------|----------------|
| «У нас табы или пробелы?» — спор на каждом PR | EditorConfig: настроено один раз, форматирует автоматически |
| Забыл `await` — баг в продакшене | Roslyn Analyzer: предупреждение на этапе компиляции |
| Дублированный код в 5 местах | SonarQube: «обнаружен дубликат в файлах X, Y, Z» |
| Code review: «поправь стиль» x100 | Автоформатирование + анализаторы: review только про логику |

### Когда настраивать?

| Ситуация | Действие |
|----------|----------|
| Начало проекта | **Сразу:** EditorConfig + Directory.Build.props + анализаторы |
| Существующий проект без инструментов | Включить постепенно: сначала EditorConfig, потом анализаторы |
| CI/CD pipeline | Добавить `dotnet format --verify-no-changes` и анализаторы в build |
| Команда > 1 человека | **Обязательно** — без инструментов стиль разъедется за неделю |

---

> По материалам: [Best Practices for Code Quality in .NET](https://antondevtips.com/blog/best-practices-for-increasing-code-quality-in-dotnet-projects/)

## Инструменты

| Инструмент | Назначение | Когда |
|------------|------------|-------|
| Code Review | Архитектура, бизнес-логика, безопасность | PR |
| Static Code Analysis | Стандарты, code smells, баги | Build |
| Roslyn Analyzers | Ошибки на этапе компиляции | IDE + Build |
| SonarQube / Qodana | Комплексный анализ, метрики, дубликаты | CI/CD |
| IDE | Real-time feedback | Development |

---

## Directory.Build.props

Один файл рядом с `.sln` — настройки для всех проектов решения:

```xml
<Project>
  <PropertyGroup>
    <!-- Базовые настройки -->
    <TargetFramework>net8.0</TargetFramework>
    <Nullable>enable</Nullable>
    <ImplicitUsings>enable</ImplicitUsings>

    <!-- Качество кода -->
    <AnalysisLevel>latest</AnalysisLevel>
    <AnalysisMode>All</AnalysisMode>
    <TreatWarningsAsErrors>true</TreatWarningsAsErrors>
    <CodeAnalysisTreatWarningsAsErrors>true</CodeAnalysisTreatWarningsAsErrors>
    <EnforceCodeStyleInBuild>true</EnforceCodeStyleInBuild>
    <EnableNETAnalyzers>true</EnableNETAnalyzers>
  </PropertyGroup>

  <ItemGroup>
    <!-- Analyzers для всех проектов -->
    <PackageReference Include="Meziantou.Analyzer" Version="2.*" PrivateAssets="all" />
    <PackageReference Include="SonarAnalyzer.CSharp" Version="9.*" PrivateAssets="all" />
    <PackageReference Include="Roslynator.Analyzers" Version="4.*" PrivateAssets="all" />
  </ItemGroup>
</Project>
```

**Нюанс:** `TreatWarningsAsErrors` с первого дня — не даёт накапливать техдолг. В legacy-проекте — включать постепенно, подавляя конкретные правила через `.editorconfig`.

### Directory.Packages.props (Central Package Management)

```xml
<Project>
  <PropertyGroup>
    <ManagePackageVersionsCentrally>true</ManagePackageVersionsCentrally>
  </PropertyGroup>
  <ItemGroup>
    <PackageVersion Include="MediatR" Version="12.*" />
    <PackageVersion Include="FluentValidation" Version="11.*" />
  </ItemGroup>
</Project>
```

Все версии NuGet-пакетов в одном месте. В `.csproj` только `<PackageReference Include="MediatR" />` без версии.

---

## .editorconfig

Контролирует стиль кода, severity правил, naming conventions:

```ini
# Корень решения
root = true

[*.cs]
# Indentation
indent_style = space
indent_size = 4

# Naming
dotnet_naming_rule.interfaces_must_start_with_i.severity = error
dotnet_naming_rule.interfaces_must_start_with_i.symbols = interfaces
dotnet_naming_rule.interfaces_must_start_with_i.style = prefix_i
dotnet_naming_symbols.interfaces.applicable_kinds = interface
dotnet_naming_style.prefix_i.required_prefix = I
dotnet_naming_style.prefix_i.capitalization = pascal_case

# Стиль
csharp_prefer_braces = true:error
csharp_using_directive_placement = outside_namespace:error
csharp_style_namespace_declarations = file_scoped:error
csharp_style_prefer_primary_constructors = true:suggestion

# Подавление шумных правил
dotnet_diagnostic.CA1062.severity = none  # null check для public methods (NRT решает)
dotnet_diagnostic.CA1848.severity = suggestion  # LoggerMessage delegates
dotnet_diagnostic.S3267.severity = none  # LINQ vs foreach (шум)
```

**Нюанс:** `.editorconfig` работает в IDE и в build. `severity = error` блокирует build. `severity = suggestion` — только подсветка в IDE.

---

## Roslyn Analyzers — обзор

### Meziantou.Analyzer

Строгие правила: конкатенация строк → интерполяция, missing ConfigureAwait, неоптимальные паттерны. Может быть шумным — отключать правила по одному, не всё сразу.

### SonarAnalyzer.CSharp

Code smells, баги, security hotspots. Хорошо интегрируется с SonarQube/SonarCloud в CI.

### Roslynator

Рефакторинг, упрощения, code fixes. Менее строгий, больше convenience.

### Microsoft.CodeAnalysis.NetAnalyzers

Встроенные (включены по умолчанию с `AnalysisLevel`). CA-правила: performance, security, design.

---

## Nullable Reference Types

```csharp
#nullable enable  // Включать в новых проектах глобально

// В legacy — включать пофайлово
#nullable enable
public class OrderService
{
    public async Task<Order?> GetAsync(Guid id, CancellationToken ct)
    {
        // Возвращаем nullable — вызывающий ДОЛЖЕН проверить
        return await _repo.GetByIdAsync(id, ct);
    }

    public async Task<Order> GetRequiredAsync(Guid id, CancellationToken ct)
    {
        // Non-nullable — гарантируем что не null
        return await _repo.GetByIdAsync(id, ct)
            ?? throw new NotFoundException("Order", id);
    }
}
```

**Нюанс:** NRT — статический анализ, не runtime проверка. `string?` компилируется в тот же `string`. Но предупреждения при `TreatWarningsAsErrors` = реальная защита.

---

## IDE плагины

### Rider
- **Cognitive Complexity** — сложность метода (цикломатическая + cognitive). Рефакторить если > 15.
- **Code Metrics** — количество строк, зависимостей, coupling.
- **Grazie** — проверка орфографии в комментариях и строках.

### Visual Studio
- **CodeMaid** — форматирование, удаление unused using, сортировка.
- **Spell Checker** — орфография.
- **Code Metrics** — встроенные метрики (Maintainability Index, Cyclomatic Complexity).

---

## Pre-commit hooks

```bash
# .husky/pre-commit (или .git/hooks/pre-commit)
#!/bin/sh
dotnet build --no-restore -warnaserror
if [ $? -ne 0 ]; then
    echo "Build failed. Fix errors before committing."
    exit 1
fi

dotnet test --no-build --filter "Category=Unit"
if [ $? -ne 0 ]; then
    echo "Unit tests failed."
    exit 1
fi
```

**Нюанс:** pre-commit hook ловит ошибки до push. Но не заменяет CI — hook можно обойти `--no-verify`. CI — обязательный gate.

---

## Code Review — чек-лист

При наличии analyzers — **не дублировать** то, что ловят автоматически. Фокус на:

| Категория | Что проверять |
|-----------|--------------|
| **Архитектура** | Границы слоёв, направление зависимостей |
| **Бизнес-логика** | Корректность, edge cases |
| **Безопасность** | Инъекции, утечка данных, авторизация |
| **Именование** | Понятные имена, единообразие |
| **Тестируемость** | Можно ли протестировать? Есть ли тесты? |
| **Performance** | N+1, лишние аллокации, blocking async |

---

## Suppress — когда и как

```csharp
// ✓ С комментарием — через месяц понятно почему
#pragma warning disable CA1822 // Mark members as static — used by DI
public string GetVersion() => "1.0";
#pragma warning restore CA1822

// ✓ SuppressMessage — для конкретного члена
[SuppressMessage("Design", "CA1062", Justification = "Validated by FluentValidation pipeline")]
public async Task<Result<Guid>> Handle(CreateOrderCommand command, CancellationToken ct)

// ✗ Плохо — глобальное подавление без причины
// dotnet_diagnostic.CA1062.severity = none  // в .editorconfig без комментария
```

---

## Метрики качества

| Метрика | Хорошо | Плохо |
|---------|--------|-------|
| Cyclomatic Complexity (метод) | < 10 | > 20 |
| Cognitive Complexity (метод) | < 15 | > 25 |
| Lines per method | < 30 | > 60 |
| Code coverage (unit) | > 70% | < 40% |
| Duplications | < 3% | > 10% |

**Нюанс:** 100% code coverage — антицель. Тестировать бизнес-логику и граничные случаи. Тривиальные геттеры и фреймворк-код — не тестировать ради процентов.

---

## Best Practices

- **TreatWarningsAsErrors** — с первого дня. Не накапливать техдолг.
- **Meziantou** — может быть шумным. Отключать правила по одному в .editorconfig.
- **SonarAnalyzer** — в CI, не в каждом build. Локально — только при необходимости.
- **Nullable** — включать в новых проектах. В legacy — `#nullable enable` по файлам.
- **Central Package Management** — единые версии NuGet для всего решения.
- **Pre-commit** — `dotnet build` в hook. Ловит ошибки до push.
- **Code review** — не дублировать analyzers. Фокус на архитектуре, безопасности, бизнес-логике.
- **Suppress с Justification** — без комментария через месяц никто не помнит причину.

---

## См. также

- [Architecture Conventions and Tests](../../Architecture/architecture-conventions-and-tests.md)
- [Start .NET Project 2026](../ProjectSetup/start-dotnet-project-2026.md)
