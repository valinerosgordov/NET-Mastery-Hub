---
tags: [code-quality, analyzers, editorconfig, sonarcloud, roslyn, meziantou, global-usings]
level: Senior
---

# Code Quality — analyzers, EditorConfig, SonarCloud

> Автоматический enforcement стиля и качества кода: `.editorconfig` + Roslyn/Meziantou/SonarAnalyzer analyzers в IDE и CI плюс SonarCloud для PR-gating, чтобы ловить баги и code smells до merge в main.

## Что это, зачем и когда

### Что такое code quality enforcement?
**Автоматическая проверка кода на стиль, баги, antipatterns** до того как код попадёт в main branch. Linters, analyzers, formatters запускаются в IDE и CI.

**Аналогия:** Корректор в издательстве — проверяет грамматику, стиль, единообразие до публикации. Разработчик может ошибиться — analyzer замечает, не даёт смержить.

### Зачем

| Без analyzers | С analyzers |
|---------------|-------------|
| Code review проверяет синтаксис | Review только архитектура / логика |
| "У вас табы, у меня spaces" — войны | EditorConfig зафиксировал |
| Опечатка → bug в production | Analyzer ловит в IDE |
| Разные паттерны в разных частях codebase | Единые правила enforced |
| Senior сам пишет хорошо, junior забывает | Junior учится через подсказки analyzer |

### Стек

```
┌─────────────────────────────────────────────────────────┐
│  IDE / CI checks                                         │
└─────────────────────────────────────────────────────────┘
       │           │            │              │
   EditorConfig  Roslyn      Custom          SonarCloud
   (стиль,      analyzers   analyzers       (sonarqube)
    форматy)    (.NET +     (свои rules)    (cloud метрики)
                third-party)
```

---

## EditorConfig

Стандартный файл `.editorconfig` в корне репо. IDE и dotnet формат подхватывают автоматически.

```ini
# .editorconfig — production-ready
root = true

[*]
charset = utf-8
end_of_line = lf
indent_style = space
indent_size = 4
insert_final_newline = true
trim_trailing_whitespace = true

[*.{json,yml,yaml,md}]
indent_size = 2

[*.cs]
# Базовый стиль
indent_size = 4
tab_width = 4
end_of_line = lf

# Using directives
dotnet_separate_import_directive_groups = false
dotnet_sort_system_directives_first = true

# Modern C# features
csharp_style_namespace_declarations = file_scoped:warning
csharp_style_var_for_built_in_types = false:warning
csharp_style_var_when_type_is_apparent = true:suggestion
csharp_style_var_elsewhere = false:warning

csharp_style_expression_bodied_methods = when_on_single_line:silent
csharp_style_expression_bodied_constructors = when_on_single_line:silent
csharp_style_expression_bodied_operators = when_on_single_line:silent
csharp_style_expression_bodied_properties = true:silent
csharp_style_expression_bodied_indexers = true:silent
csharp_style_expression_bodied_accessors = true:silent
csharp_style_expression_bodied_lambdas = true:silent

# Pattern matching
csharp_style_pattern_matching_over_is_with_cast_check = true:warning
csharp_style_pattern_matching_over_as_with_null_check = true:warning
csharp_style_prefer_pattern_matching = true:warning
csharp_style_prefer_switch_expression = true:warning

# Nullable references
csharp_style_throw_expression = true:suggestion
dotnet_style_coalesce_expression = true:warning
dotnet_style_null_propagation = true:warning

# Code blocks
csharp_prefer_braces = true:warning
csharp_prefer_simple_using_statement = true:suggestion
csharp_prefer_simple_default_expression = true:suggestion

# Modifier preferences
dotnet_style_readonly_field = true:warning
csharp_preferred_modifier_order = public,private,protected,internal,static,extern,new,virtual,abstract,sealed,override,readonly,unsafe,volatile,async:warning

# Field/property naming
dotnet_naming_rule.private_fields_should_be_camel_case_with_underscore.symbols = private_fields
dotnet_naming_rule.private_fields_should_be_camel_case_with_underscore.style = camel_case_underscore_style
dotnet_naming_rule.private_fields_should_be_camel_case_with_underscore.severity = warning

dotnet_naming_symbols.private_fields.applicable_kinds = field
dotnet_naming_symbols.private_fields.applicable_accessibilities = private
dotnet_naming_symbols.private_fields.required_modifiers = readonly

dotnet_naming_style.camel_case_underscore_style.required_prefix = _
dotnet_naming_style.camel_case_underscore_style.capitalization = camel_case

# Suppressing
dotnet_diagnostic.CA1062.severity = none      # Validate arguments — false positives с nullable enabled
dotnet_diagnostic.CA1303.severity = none      # Localize strings — не нужно для большинства проектов
dotnet_diagnostic.CA1707.severity = none      # Underscores в names — иногда нужно

# Errors as compile errors (опционально)
dotnet_diagnostic.IDE0005.severity = warning  # Unused using
```

### Format check в CI

```bash
# Проверка форматирования (без изменений)
dotnet format --verify-no-changes

# Применить (локально)
dotnet format
```

В CI fail если есть неформатированный код. Single source of truth — `.editorconfig`.

---

## Built-in .NET Analyzers

С .NET 8+ analyzers включены **по умолчанию**. Проверяют CA-rules (Code Analysis):

```xml
<!-- .csproj -->
<PropertyGroup>
  <AnalysisLevel>latest-recommended</AnalysisLevel>
  <AnalysisMode>All</AnalysisMode>
  <EnableNETAnalyzers>true</EnableNETAnalyzers>
  <TreatWarningsAsErrors>true</TreatWarningsAsErrors>
  <WarningsNotAsErrors>CA1062;CS1591</WarningsNotAsErrors>
  <Nullable>enable</Nullable>
  <ImplicitUsings>enable</ImplicitUsings>
</PropertyGroup>
```

`AnalysisLevel`:
- `latest-default` — defaults to current .NET
- `latest-recommended` — стандарт от Microsoft
- `latest-all` — максимум включено
- `9.0-recommended` — фиксированная версия

`TreatWarningsAsErrors` — каждое warning блокирует build. `WarningsNotAsErrors` — список исключений.

### Главные правила

| ID | О чём | Severity |
|----|-------|----------|
| **CA1062** | Validate non-null parameters | Disable если nullable enabled |
| **CA2007** | ConfigureAwait в library code | Suggestion в ASP.NET Core |
| **CA1707** | Underscores в имени | Disable для test method names |
| **CA1822** | Member can be static | Warning |
| **CA1819** | Properties не должны возвращать array | Warning |
| **CA1860** | Use Length/Count over Any() | Warning |
| **CA2227** | Collection properties readonly | Warning |
| **CA2208** | ArgumentException right ctor | Warning |
| **CA2245** | Property assigned to itself | Warning |
| **IDE0005** | Unused using | Warning + auto-fix |

### Per-file overrides

```csharp
#pragma warning disable CA1062
public void Method(string param)
{
    // ...
}
#pragma warning restore CA1062

// Or for whole file
[SuppressMessage("Performance", "CA1860:Avoid using Enumerable.Any")]
```

---

## Meziantou.Analyzer

Open-source расширенный набор от Gérald Barré. Покрывает что .NET-built-in пропускает.

```xml
<PackageReference Include="Meziantou.Analyzer" Version="2.0.x" PrivateAssets="all" />
```

### Top правила

| ID | Что |
|----|-----|
| **MA0004** | `ConfigureAwait(false)` в library |
| **MA0006** | Не использовать `Task.Run` без причины |
| **MA0011** | Использовать `IFormatProvider` (`CultureInfo.InvariantCulture`) |
| **MA0016** | Возвращать interface вместо concrete |
| **MA0048** | Не используй `string.Empty` — используй `""` |
| **MA0089** | `StringBuilder` для повторных concat |
| **MA0099** | `LINQ` без allocations (FirstOrDefault → правильно) |
| **MA0111** | Async-метод должен иметь имя с `Async` |
| **MA0157** | `try-catch` без re-throw — antipattern |

В крупных проектах Meziantou ловит десятки issues, которые ускользают от .NET analyzers.

---

## SonarAnalyzer.CSharp

```xml
<PackageReference Include="SonarAnalyzer.CSharp" Version="9.x" PrivateAssets="all" />
```

Часть SonarSource — те же rules что в SonarCloud (S-prefix). Бесплатно как analyzer:

| ID | Что |
|----|-----|
| **S1854** | Dead store (assignment never used) |
| **S2275** | Static methods invocation на instance |
| **S3267** | Loops can be replaced with LINQ |
| **S3343** | Caller info attributes order |
| **S3358** | Avoid nested ternaries |
| **S3878** | Array creation как parameter to params |
| **S4144** | Methods with same logic — extract |
| **S6603** | Override Equals/GetHashCode pair |

Combined с Meziantou + .NET built-in — почти все code smells покрыты.

---

## SonarCloud / SonarQube

**Cloud-based code quality platform.** Sonar анализирует репо после push, делает comment в PR с list issues.

### Setup для GitHub Actions

```yaml
# .github/workflows/sonar.yml
name: SonarCloud

on:
  push:
    branches: [main]
  pull_request:

jobs:
  sonarcloud:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - uses: actions/setup-java@v4
        with:
          distribution: 'temurin'
          java-version: 17

      - uses: actions/setup-dotnet@v4
        with:
          dotnet-version: '10.0.x'

      - name: Cache Sonar
        uses: actions/cache@v4
        with:
          path: ~/.sonar/cache
          key: ${{ runner.os }}-sonar

      - name: SonarCloud Scan
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}
        run: |
          dotnet tool install --global dotnet-sonarscanner

          dotnet sonarscanner begin \
            /k:"my-org_my-repo" \
            /o:"my-org" \
            /d:sonar.token="${{ secrets.SONAR_TOKEN }}" \
            /d:sonar.host.url="https://sonarcloud.io" \
            /d:sonar.cs.opencover.reportsPaths=coverage/coverage.opencover.xml

          dotnet build --configuration Release
          dotnet test --configuration Release \
            /p:CollectCoverage=true /p:CoverletOutputFormat=opencover \
            /p:CoverletOutput=../coverage/

          dotnet sonarscanner end /d:sonar.token="${{ secrets.SONAR_TOKEN }}"
```

### Quality Gate

Sonar по умолчанию имеет **Sonar way** quality gate:
- 0 new bugs
- 0 new vulnerabilities
- ≤ A maintainability rating
- Coverage ≥ 80% on new code
- Duplications < 3% on new code

Customize:
- ≥ 70% coverage on new code
- 0 hotspots reviewed → должны
- Cognitive complexity per method < 15

PR fail если quality gate red. Comment в PR показывает где issues.

### Когда Sonar vs analyzers

| | Built-in analyzers | SonarCloud |
|--|---------------------|------------|
| Where | IDE + CI | Cloud + PR comments |
| Скорость | Real-time | Run в CI |
| Coverage | Code quality | + security hotspots, duplications |
| History | Нет | Track over time, trends |
| Cost | Free | Free for OSS, $$$ для commercial |

Combined в production:
- **Built-in + Meziantou + SonarAnalyzer.CSharp** — для real-time IDE feedback
- **SonarCloud** — для PR-gating + history tracking

Бюджет тесный — ограничься analyzers. SonarCloud стоит для commercial.

---

## GlobalUsings (.NET 6+)

Вместо повторения `using System;` в каждом файле — глобально:

```csharp
// GlobalUsings.cs (или прямо в .csproj)
global using System;
global using System.Collections.Generic;
global using System.Linq;
global using System.Threading;
global using System.Threading.Tasks;
global using Microsoft.Extensions.Logging;
global using MyApp.Domain;
```

Или через MSBuild:
```xml
<ItemGroup>
  <Using Include="System" />
  <Using Include="System.Threading.Tasks" />
  <Using Include="MyApp.Domain" Static="true" Alias="Domain" />
  <Using Include="MyApp.Common.Result&lt;&gt;" />
</ItemGroup>
```

`<ImplicitUsings>enable</ImplicitUsings>` — автоматический набор для типа проекта (Web / Console / Library).

### Когда стоит и не стоит

| Использовать | Не использовать |
|--------------|----------------|
| Стандартные .NET типы (System, Linq, Tasks) | Domain types (читать `Order` без `using` мешает понимать откуда) |
| Внутренние shared abstractions (`Result<T>`, `IClock`) | Conflicting types между namespaces |
| Test infrastructure (`global using Xunit;`) | Каждый файл уникален |

---

## Custom Roslyn Analyzers

Для domain-specific rules. Например, "запретить `DateTime.Now`, использовать `IClock`".

```csharp
[DiagnosticAnalyzer(LanguageNames.CSharp)]
public class ForbidDateTimeNowAnalyzer : DiagnosticAnalyzer
{
    public static readonly DiagnosticDescriptor Rule = new(
        "MYAPP001",
        "Don't use DateTime.Now",
        "Use IClock.UtcNow instead of DateTime.Now",
        "Reliability",
        DiagnosticSeverity.Error,
        isEnabledByDefault: true);

    public override ImmutableArray<DiagnosticDescriptor> SupportedDiagnostics => [Rule];

    public override void Initialize(AnalysisContext context)
    {
        context.ConfigureGeneratedCodeAnalysis(GeneratedCodeAnalysisFlags.None);
        context.EnableConcurrentExecution();
        context.RegisterSyntaxNodeAction(Analyze, SyntaxKind.SimpleMemberAccessExpression);
    }

    private static void Analyze(SyntaxNodeAnalysisContext context)
    {
        var memberAccess = (MemberAccessExpressionSyntax)context.Node;
        if (memberAccess.Expression is IdentifierNameSyntax identifier &&
            identifier.Identifier.Text == "DateTime" &&
            memberAccess.Name.Identifier.Text is "Now" or "Today")
        {
            context.ReportDiagnostic(Diagnostic.Create(Rule, memberAccess.GetLocation()));
        }
    }
}
```

См. [Source Generators]() — Roslyn API similar (analyzer и generator используют одни базовые types).

### Когда custom

- Domain-specific rules ("в этом проекте не используй X")
- Forbidden APIs (legacy code, deprecated functions)
- Internal architecture rules не покрытые arch-tests
- Educational — junior получает "не делай так" message

---

## Forbidden Code Analyzer

Простой способ запретить APIs без custom analyzer.

```xml
<PackageReference Include="BannedApiAnalyzers" Version="3.x" PrivateAssets="all" />
```

```
# BannedSymbols.txt
M:System.DateTime.get_Now;Use IClock.UtcNow instead
M:System.DateTime.get_Today;Use IClock.UtcNow.Date instead
M:System.Console.WriteLine(System.String);Use ILogger
T:Newtonsoft.Json.JsonConvert;Use System.Text.Json
M:System.Threading.Thread.Sleep(System.Int32);Don't block — use await Task.Delay
```

Build падает если кто-то использует banned API. Документация каждого ban — почему запрещено.

---

## .editorconfig + analyzer rules как ADR

Каждое нестандартное правило документируй:

```ini
# .editorconfig

# CA1062 disabled because Nullable references handle parameter validation
dotnet_diagnostic.CA1062.severity = none

# MA0048 — string.Empty vs "" — silent (style preference, not enforced)
dotnet_diagnostic.MA0048.severity = silent
```

Большой `.editorconfig` без комментариев — никто не помнит почему так. Через год хочется убрать `severity = none` и лезть в git blame.

---

## Pre-commit hooks (опционально)

Husky.NET — запуск hooks перед commit:

```bash
dotnet tool install -g Husky
husky install
husky add pre-commit "dotnet format --verify-no-changes"
```

Pre-commit fail → commit не проходит. Защита от "забыл format".

```bash
# .husky/pre-commit
#!/bin/sh
dotnet format --verify-no-changes
dotnet test --filter Category=Architecture
```

Не для всех — pre-commit hooks tend быть аnnoying если slow. Лучше CI как primary gate, hooks как opt-in для local dev.

---

## Production checklist

- [ ] `.editorconfig` в корне репо
- [ ] `<TreatWarningsAsErrors>true</TreatWarningsAsErrors>` в Directory.Build.props
- [ ] `<AnalysisLevel>latest-recommended</AnalysisLevel>`
- [ ] Meziantou.Analyzer установлен
- [ ] SonarAnalyzer.CSharp установлен
- [ ] SonarCloud (или SonarQube) integrated в CI с PR comments
- [ ] Quality Gate настроен (coverage threshold, complexity limits)
- [ ] BannedApiAnalyzers с domain-specific banned APIs
- [ ] Custom Roslyn analyzers если есть recurring violations
- [ ] `dotnet format --verify-no-changes` в CI
- [ ] GlobalUsings для повторяющихся imports
- [ ] EditorConfig правила имеют comments / ADR ссылки
- [ ] Все warnings → errors (не игнорировать)
- [ ] Warnings exclusion list (`WarningsNotAsErrors`) с обоснованием

---

## Common pitfalls

### 1. Bulk-suppress warnings

```xml
<!-- ❌ Слишком широко -->
<NoWarn>CA1062;CA1303;CA1707;CS1591;CS8600;CS8602;CS8603;CS8604;...</NoWarn>
```
Потом никто не помнит почему всё это disabled. Иногда там реальные баги.
**Решение:** explicit `severity = none` в editorconfig с comment.

### 2. Disabling вместо fixing
"CA1062 fails — добавлю null check" → правильно.
"CA1062 fails — disable" → накапливается тех долг.

### 3. WarningsAsErrors без guard
Зеркальная проблема — все warnings = errors → новый analyzer добавил правило → весь codebase падает на build.
**Решение:** `WarningsNotAsErrors` для известных warnings, fix gradually.

### 4. SonarCloud как замена code review
Sonar — assistant, не replacement. Code review проверяет архитектуру, intent, business logic. Sonar — automatable rules.

### 5. Локальный analyzer != CI
Dev забыл обновить analyzer version → его IDE не показывает warnings, но CI fail. Lock через `Directory.Packages.props`.

### 6. Custom analyzer без tests
Custom analyzer пишется несколько часов — без тестов превратится в шум через 6 месяцев.
**Решение:** snapshot tests с Verify (см. [Testing / Verify]()).

### 7. EditorConfig в проекте, но IDE не подхватывает
Проверь IDE settings — Rider, VS, VS Code должны иметь support включён. `.editorconfig` без support = декорация.

### 8. Не запускать `dotnet format` regularly
Стиль drift'ит. Один раз merge'нул код в неформатированном виде — потом всем mismatch'и в diff.
**Решение:** CI gate `dotnet format --verify-no-changes`.

### 9. Слишком мало правил включено
"Linter ничего не находит" — потому что disabled половина правил. Включай aggressive set, fix iteratively.

### 10. Игнорирование security analyzers
SonarCloud категории Security Hotspots — каждый требует ручного review. Игнорировать = serious risk.

---

## См. также

- [Architecture Tests]() — NetArchTest для arch rules + analyzers для code rules
- [Source Generators]() — Roslyn API для custom analyzers
- [Project Setup]() — Directory.Build.props глобальная конфигурация
- [Modern C# 8–14]() — что включено через `latest-recommended`
- [Testing]() — Snapshot tests для custom analyzers

## Reading list

- **Microsoft Learn — Code analysis** — learn.microsoft.com/dotnet/fundamentals/code-analysis/
- **Meziantou.Analyzer** — github.com/meziantou/Meziantou.Analyzer
- **SonarSource Rules — C#** — rules.sonarsource.com/csharp/
- **SonarCloud quality gates** — docs.sonarcloud.io
- **Andrew Lock — Code analysis series** — andrewlock.net
- **Gérald Barré (Meziantou) blog** — meziantou.net (canonical для C# analyzers)
- **Steve Smith — Clean Code analyzers** — ardalis.com
- **Roslyn Analyzers Wiki** — github.com/dotnet/roslyn-analyzers/wiki
