---
tags: [quality, static-analysis, analyzers, roslyn, sonarqube, linting, code-review]
level: Senior
date: 2026-04-30
---

# Static Analysis — анализаторы кода

> Полный гайд по static analysis в .NET. Закрывает: встроенные analyzers, SonarAnalyzer, Meziantou, Roslynator, SonarQube для команды, написание custom analyzer, code metrics, third-party tools, integration в CI/CD.

---

## Что это, зачем и когда

### Что такое static analysis?

**Анализ кода без его выполнения** — компилятор / специальные инструменты ищут потенциальные баги, code smells, security issues, performance problems.

**Аналогия:** Корректор у писателя. Не пишет за тебя, но указывает на ошибки до публикации (production).

### Зачем

| Без static analysis | С static analysis |
|---------------------|-------------------|
| Bugs в production обнаруживаются юзерами | Compile-time / pre-merge |
| Inconsistent coding style | Auto-format + analyzers — единый стиль |
| Memory leaks в hot path незаметны | `CA2000`, MA-rules ловят |
| Security vulnerabilities ловятся аудитом | CodeQL / SonarQube alert'ит |
| Code review — нудный nitpicking | Analyzers ловят nits, review — про design |
| Tech debt накапливается невидимо | Quality gates в CI |

### Уровни

```
1. Compiler warnings        — встроены в C#
2. Roslyn analyzers          — runtime analysis в IDE / build
3. Standalone tools          — SonarQube, NDepend
4. Security scanners         — CodeQL, Snyk
5. Architecture validation   — NetArchTest
6. Mutation testing          — Stryker.NET (код или твои тесты надёжен?)
```

---

## 1. Встроенные C# warnings

### TreatWarningsAsErrors

```xml
<!-- csproj -->
<PropertyGroup>
  <TreatWarningsAsErrors>true</TreatWarningsAsErrors>
  <WarningsNotAsErrors>NU1900;NU1901</WarningsNotAsErrors>  <!-- кроме этих -->
  <NoWarn>CS1591</NoWarn>  <!-- отключить вообще -->
</PropertyGroup>
```

После этого warning ломает build. Принудительно следишь за качеством.

### Nullable reference types

```xml
<Nullable>enable</Nullable>
```

```csharp
string s = null;        // CS8600: warning
string? maybe = null;   // OK
```

См. [[csharp-language-design#null-safety|C# Language Design — Null Safety]].

### Code style enforcement

```xml
<EnforceCodeStyleInBuild>true</EnforceCodeStyleInBuild>
```

В сочетании с `.editorconfig` — стиль становится частью build.

Два разных свойства закрывают две группы правил: `CodeAnalysisTreatWarningsAsErrors` поднимает до ошибок `CAxxxx` (quality/reliability/security analyzers), а `EnforceCodeStyleInBuild` включает в build `IDExxxx` (code-style). Это ортогональные тумблеры — типичная ошибка ждать, что один потянет оба.

```ini
# .editorconfig
[*.cs]
csharp_style_var_for_built_in_types = false:warning
csharp_style_namespace_declarations = file_scoped:error
csharp_style_prefer_primary_constructors = true:suggestion
```

См. [[project-setup#editorconfig|Project Setup — .editorconfig]].

---

## 2. .NET Built-in Analyzers (Microsoft.CodeAnalysis.NetAnalyzers)

В .NET 5+ — встроены автоматически.

```xml
<PropertyGroup>
  <AnalysisMode>All</AnalysisMode>            <!-- или Recommended, Default -->
  <AnalysisLevel>latest</AnalysisLevel>
  <EnableNETAnalyzers>true</EnableNETAnalyzers>
</PropertyGroup>
```

### Категории правил

| Prefix | Категория |
|--------|-----------|
| **CA1xxx** | Design — design issues (interfaces, naming) |
| **CA2xxx** | Reliability — exceptions, dispose |
| **CA3xxx** | Security — XSS, SQL injection |
| **CA5xxx** | Security cryptography |
| **IDE0xxx** | Code style |

### Топ важных rules

```ini
# .editorconfig

# Reliability
dotnet_diagnostic.CA2000.severity = error    # Dispose objects
dotnet_diagnostic.CA2007.severity = error    # ConfigureAwait (для library)
dotnet_diagnostic.CA2208.severity = error    # ArgumentException constructor

# Performance
dotnet_diagnostic.CA1825.severity = warning  # Avoid empty arrays — use Array.Empty<T>()
dotnet_diagnostic.CA1860.severity = warning  # Use Length/Count instead of .Any()

# Security
dotnet_diagnostic.CA2100.severity = error    # SQL injection
dotnet_diagnostic.CA3061.severity = error    # XML loading without resolver

# Design
dotnet_diagnostic.CA1051.severity = warning  # Public fields
```

---

## 3. Third-party analyzers

### SonarAnalyzer.CSharp

[github.com/SonarSource/sonar-dotnet](https://github.com/SonarSource/sonar-dotnet)

От SonarSource (создатели SonarQube). Очень богатый набор rules.

```xml
<PackageReference Include="SonarAnalyzer.CSharp" Version="11.0.0" PrivateAssets="all" />
```

**Хорошо ловит:**
- `S1144` — Unused private types / methods
- `S125` — Commented-out code
- `S2629` — Don't use string interpolation in logger calls (perf — interpolated string created всегда)
- `S2259` — Null pointers
- `S2696` — Static fields modified from instance methods
- `S3260` — Sealed for inheritance prevention
- `S3776` — Cognitive complexity (метрика читабельности)
- `S109` — Magic numbers — assign to constant
- `S5344` — Weak password hashing (PBKDF2/bcrypt с малым числом итераций, MD5/SHA1 для паролей) — security hotspot

```csharp
// ❌ S2629 — string interpolation всегда выполняется, даже если log level disabled
logger.LogInformation($"User {user.Id} logged in");

// ✅ Структурированный log — interpolation только если уровень enabled
logger.LogInformation("User {UserId} logged in", user.Id);
```

### Meziantou.Analyzer

[github.com/meziantou/Meziantou.Analyzer](https://github.com/meziantou/Meziantou.Analyzer)

От Gérald Barré (Microsoft MVP). Фокус на async/await + perf.

```xml
<PackageReference Include="Meziantou.Analyzer" Version="2.0.0" PrivateAssets="all" />
```

**Топ rules:**
- `MA0040` — Forward CancellationToken to methods accepting one
- `MA0042` — Don't use blocking calls in async method (`.Result`, `.Wait()`)
- `MA0045` — Don't use sync over async
- `MA0048` — File name should match type name
- `MA0004` — Use `Task.ConfigureAwait(false)` (для library кода)
- `MA0098` — Use `IsNullOrEmpty` and `IsNullOrWhiteSpace`
- `MA0099` — Use Explicit enum value (do not rely on default)

```csharp
// MA0042 caught:
public async Task DoWork()
{
    var data = SlowMethod().Result;  // ❌ blocking in async!
    var data2 = await SlowMethodAsync();  // ✅
}

// MA0040 caught:
public async Task ProcessAsync(CancellationToken ct)
{
    await Task.Delay(1000);            // ❌ ct не передан
    await Task.Delay(1000, ct);        // ✅
}
```

### Roslynator

[github.com/dotnet/roslynator](https://github.com/dotnet/roslynator)

От Josef Pihrt. Code style + refactoring suggestions.

```xml
<PackageReference Include="Roslynator.Analyzers" Version="4.13.0" PrivateAssets="all" />
<PackageReference Include="Roslynator.Formatting.Analyzers" Version="4.13.0" PrivateAssets="all" />
```

**Топ rules:**
- `RCS1090` — Add `ConfigureAwait(false)` (для library)
- `RCS1124` — Inline local variable
- `RCS1163` — Unused parameter
- `RCS1213` — Remove unused member declaration
- `RCS1235` — Optimize method call (e.g. `string.IsNullOrEmpty` vs `== ""`)

### Combined setup в Directory.Build.props

```xml
<Project>
  <ItemGroup>
    <PackageReference Include="SonarAnalyzer.CSharp" PrivateAssets="all" />
    <PackageReference Include="Meziantou.Analyzer" PrivateAssets="all" />
    <PackageReference Include="Roslynator.Analyzers" PrivateAssets="all" />
    <PackageReference Include="Roslynator.Formatting.Analyzers" PrivateAssets="all" />
  </ItemGroup>
</Project>
```

См. [[project-setup|Project Setup]] — full setup.

---

## 4. SonarQube — для команды

[sonarqube.org](https://www.sonarqube.org)

Centralized server для quality tracking всей organization.

### Setup

```bash
# Запустить SonarQube локально
docker run -d --name sonarqube -p 9000:9000 sonarqube:lts-community

# Открыть http://localhost:9000 (admin/admin)
```

### Анализ проекта

```bash
# Установить scanner
dotnet tool install --global dotnet-sonarscanner

# Анализ
dotnet sonarscanner begin /k:"my-project" /d:sonar.host.url="http://localhost:9000" /d:sonar.token="$SONAR_TOKEN"
dotnet build
dotnet test --collect:"XPlat Code Coverage"
dotnet sonarscanner end /d:sonar.token="$SONAR_TOKEN"
```

### Quality Gates

В SonarQube — настраиваешь "правила прохода":
- Coverage >= 80% on new code
- No new bugs
- No new vulnerabilities
- Cognitive complexity reasonable

В CI — если quality gate failed → block PR.

```yaml
# .github/workflows/sonarqube.yml
- name: SonarQube Scan
  uses: SonarSource/sonarqube-scan-action@v3
  env:
    SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}
    SONAR_HOST_URL: ${{ vars.SONAR_HOST_URL }}
```

---

## 5. CodeQL — security scanning

[codeql.github.com](https://codeql.github.com)

GitHub-native security analysis. Бесплатно для public repos.

```yaml
# .github/workflows/codeql.yml
name: "CodeQL"
on:
  push: { branches: [main] }
  pull_request: { branches: [main] }
  schedule: [ { cron: "0 6 * * 1" } ]  # weekly

jobs:
  analyze:
    runs-on: ubuntu-latest
    permissions:
      security-events: write
      contents: read
    steps:
      - uses: actions/checkout@v4
      - uses: github/codeql-action/init@v3
        with: { languages: csharp }
      - uses: github/codeql-action/autobuild@v3
      - uses: github/codeql-action/analyze@v3
```

CodeQL ловит:
- SQL injection
- XSS
- Path traversal
- Hardcoded credentials
- Insecure deserialization
- Cryptographic weaknesses

Result — в Security tab GitHub repo.

---

## 6. Architecture Tests — NetArchTest

См. [[arch-tests|Architecture Tests]].

```csharp
[Fact]
public void Domain_should_not_reference_Infrastructure()
{
    Types.InAssembly(typeof(Order).Assembly)
        .ShouldNot()
        .HaveDependencyOn("MyApp.Infrastructure")
        .GetResult()
        .IsSuccessful
        .Should().BeTrue();
}
```

Архитектурные правила enforced в test phase. Если кто-то добавит зависимость которую не должно — test fail.

---

## 7. Code Metrics

### Cyclomatic Complexity

Сколько различных путей через метод. >10 = сложный, >20 = плохо.

```csharp
// CC = 1 (один путь)
public int Add(int a, int b) => a + b;

// CC = 4 (3 if + happy path)
public string Categorize(int score)
{
    if (score < 0) return "invalid";
    if (score < 50) return "fail";
    if (score < 80) return "pass";
    return "excellent";
}
```

В Visual Studio: `Analyze → Calculate Code Metrics`.

### Cognitive Complexity (Sonar)

Лучше CC — учитывает nesting и control flow patterns.

```csharp
// CC=4, Cognitive=4
if (a) { if (b) { while (c) { ... } } }
```

### Maintainability Index

Microsoft formula с CC + lines of code + Halstead complexity. 0-100, >20 хорошо.

### Lines of Code

Метрика сама по себе — bad. Но **рост** LoC за месяц — индикатор tech debt accumulation.

---

## 8. Custom Roslyn Analyzer

Когда нужно свои правила — Roslyn API позволяет писать analyzers.

```bash
dotnet new analyzer -n MyAnalyzer
```

### Простой analyzer

```csharp
[DiagnosticAnalyzer(LanguageNames.CSharp)]
public class NoTodoAnalyzer : DiagnosticAnalyzer
{
    private static readonly DiagnosticDescriptor Rule = new(
        id: "MA0001",
        title: "Don't leave TODOs in production code",
        messageFormat: "TODO comment found: {0}",
        category: "Maintenance",
        defaultSeverity: DiagnosticSeverity.Warning,
        isEnabledByDefault: true);

    public override ImmutableArray<DiagnosticDescriptor> SupportedDiagnostics => 
        ImmutableArray.Create(Rule);

    public override void Initialize(AnalysisContext context)
    {
        context.ConfigureGeneratedCodeAnalysis(GeneratedCodeAnalysisFlags.None);
        context.EnableConcurrentExecution();
        
        context.RegisterSyntaxTreeAction(AnalyzeSyntaxTree);
    }

    private static void AnalyzeSyntaxTree(SyntaxTreeAnalysisContext context)
    {
        var trivia = context.Tree.GetRoot()
            .DescendantTrivia()
            .Where(t => t.IsKind(SyntaxKind.SingleLineCommentTrivia));

        foreach (var t in trivia)
        {
            var text = t.ToString();
            if (text.Contains("TODO", StringComparison.OrdinalIgnoreCase))
            {
                var diagnostic = Diagnostic.Create(Rule, t.GetLocation(), text);
                context.ReportDiagnostic(diagnostic);
            }
        }
    }
}
```

### Code fix provider

Если правило имеет auto-fix:

```csharp
[ExportCodeFixProvider(LanguageNames.CSharp, Name = nameof(NoTodoFixProvider))]
public class NoTodoFixProvider : CodeFixProvider
{
    public override ImmutableArray<string> FixableDiagnosticIds =>
        ImmutableArray.Create("MA0001");

    public override async Task RegisterCodeFixesAsync(CodeFixContext context)
    {
        // Зарегистрировать option "Remove TODO comment"
        context.RegisterCodeFix(
            CodeAction.Create(
                title: "Remove TODO comment",
                createChangedDocument: ct => RemoveCommentAsync(context.Document, context.Span, ct),
                equivalenceKey: "RemoveTODO"),
            context.Diagnostics);
    }
    
    private async Task<Document> RemoveCommentAsync(Document doc, TextSpan span, CancellationToken ct)
    {
        var root = await doc.GetSyntaxRootAsync(ct);
        var newRoot = root!.ReplaceText(span, "");  // удалить
        return doc.WithSyntaxRoot(newRoot);
    }
}
```

### Distribution

```xml
<!-- analyzer.csproj -->
<PropertyGroup>
  <TargetFramework>netstandard2.0</TargetFramework>  <!-- IMPORTANT -->
  <IsPackable>true</IsPackable>
  <PackAsAnalyzer>true</PackAsAnalyzer>
</PropertyGroup>

<ItemGroup>
  <PackageReference Include="Microsoft.CodeAnalysis.CSharp" Version="4.10.0" />
</ItemGroup>
```

```bash
dotnet pack
# Пакет можно опубликовать в NuGet — другие проекты подключают
```

> [!info] Когда писать свой analyzer
> - Internal company conventions (e.g. "all repositories must extend BaseRepository")
> - Domain-specific rules (e.g. "all DateTime in API must be UTC")
> - Preventing common mistakes specific to your codebase
> - Migration helpers (deprecated API → new API)

---

## 9. Mutation Testing — Stryker.NET

Тестирует **твои тесты**: меняет код (mutation) и проверяет, поймали ли тесты изменение.

```bash
dotnet tool install -g dotnet-stryker
cd MyProject.Tests
dotnet stryker
```

### Пример

```csharp
public bool IsAdult(int age) => age >= 18;
```

Stryker создаст mutations:
- `age >= 18` → `age > 18` (boundary mutation)
- `age >= 18` → `age <= 18`
- `age >= 18` → `false`

Если твои тесты не отлавливают эти изменения — score низкий, тесты слабые.

```
Mutation score: 87.50% (35 mutations killed, 5 survived)
```

> [!info] Когда mutation testing
> - Critical business logic (payment, auth)
> - Когда test coverage уже >80% но непонятно, реально ли тесты ловят баги
> - Code review — "докажи, что эти тесты не theatre"

---

## 10. NDepend — commercial

[ndepend.com](https://www.ndepend.com)

Платный, но самый мощный для:
- Architecture visualization
- Dependency analysis
- Code metrics dashboard
- Tech debt estimation
- Custom rules через CQLinq

Используется enterprise-командами.

---

## 11. Linting и formatting

### dotnet format

Built-in formatter (.NET 6+).

```bash
# Format всё
dotnet format

# Только code style
dotnet format style

# Только whitespace
dotnet format whitespace

# Verify (CI mode)
dotnet format --verify-no-changes
```

В CI:
```yaml
- name: Verify formatting
  run: dotnet format --verify-no-changes
```

### CSharpier — opinionated formatter

Подобен Prettier для JS — нет настроек, форматирует одинаково везде.

```bash
dotnet tool install -g CSharpier
dotnet csharpier .
```

### EditorConfig — основа

См. [[project-setup#editorconfig|Project Setup .editorconfig]].

---

## 12. Pre-commit hooks через Husky.NET

Запускает analyzers / format / tests **до commit**.

```bash
dotnet new tool-manifest
dotnet tool install Husky
dotnet husky install
```

`.husky/pre-commit`:
```bash
#!/bin/sh
dotnet husky run --name pre-commit
```

`task-runner.json`:
```json
{
    "tasks": [
        {
            "name": "pre-commit",
            "command": "dotnet",
            "args": ["format", "--include", "${staged}"],
            "include": ["**/*.cs"]
        },
        {
            "name": "pre-commit",
            "command": "dotnet",
            "args": ["build", "--no-restore", "/p:TreatWarningsAsErrors=true"]
        }
    ]
}
```

После — каждый commit auto-format'ится и build verify'ится.

---

## 13. CI integration — quality gates

### GitHub Actions full pipeline

```yaml
name: Quality Gate
on: [pull_request]

jobs:
  quality:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with: { fetch-depth: 0 }
      
      - uses: actions/setup-dotnet@v4
        with: { dotnet-version: '10.0.x' }
      
      # 1. Format check
      - name: Format check
        run: dotnet format --verify-no-changes
      
      # 2. Build with warnings as errors
      - name: Build
        run: dotnet build --configuration Release /p:TreatWarningsAsErrors=true
      
      # 3. Run all tests with coverage
      - name: Test
        run: dotnet test --no-build --collect:"XPlat Code Coverage"
      
      # 4. Architecture tests (NetArchTest)
      - name: Architecture
        run: dotnet test --filter Category=Architecture
      
      # 5. SonarQube
      - name: SonarQube
        uses: SonarSource/sonarqube-scan-action@v3
        env:
          SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}
      
      # 6. CodeQL (security)
      - name: CodeQL Init
        uses: github/codeql-action/init@v3
      - name: CodeQL Analyze
        uses: github/codeql-action/analyze@v3
      
      # 7. Mutation testing (только weekly)
      - name: Stryker
        if: github.event.schedule
        run: dotnet stryker --break-at 80
```

---

## 14. Code Review — что должны ловить тесты vs review

### Analyzers ловят (autоматически)

- Style / formatting nits
- Common bugs (null deref, missing dispose)
- Async/await mistakes
- Performance pitfalls
- Security обвcious problems
- Naming conventions

### Code Review (human)

- **Architecture / design** — правильно ли разделено?
- **Бизнес-логика правильна** — соответствует requirements?
- **Tests test the right thing** — не просто for coverage
- **Edge cases** — что если null / 0 / max int?
- **Maintainability** — читабельность через 6 месяцев
- **Error handling strategy** — где throw, где Result
- **Trade-offs** — почему выбрано это решение

> [!info] Цель analyzers
> Освободить code review от nitpicking, чтобы reviewer фокусировался на design.

---

## 15. Best Practices

- **Включи всё с 1-го дня** — `TreatWarningsAsErrors`, `Nullable enable`, analyzers
- **`.editorconfig` в repo root** — единый стиль
- **Combined analyzers** — Sonar + Meziantou + Roslynator + .NET built-in
- **CodeQL** для security (free на public repo)
- **SonarQube** для team (commercial для private repo > 5)
- **Pre-commit hooks** — auto-format + build verify
- **CI quality gates** — block merge если quality regress
- **Mutation testing** на critical paths (Stryker)
- **Architecture tests** (NetArchTest) — runtime enforcement
- **Custom analyzers** для company conventions
- **Severity tuning** — не все rules error, прагматично

### Постепенное внедрение в legacy

Если в legacy 5000 warnings — нельзя сразу `TreatWarningsAsErrors=true`:

```
Phase 1: Per-project включить
  <TreatWarningsAsErrors>false</TreatWarningsAsErrors> в legacy projects
  <TreatWarningsAsErrors>true</TreatWarningsAsErrors> в новых

Phase 2: Per-rule
  Постепенно поднимай severity для rules: Hidden → Suggestion → Warning → Error

Phase 3: Per-file (через .editorconfig sections)
  [src/NewModule/**.cs]
  dotnet_diagnostic.CA2007.severity = error  # только в новом коде
  
  [src/LegacyModule/**.cs]
  dotnet_diagnostic.CA2007.severity = none

Phase 4: Eventual
  Все warnings → errors
```

---

## 16. Сравнение tools

| Tool | Use case | Cost |
|------|----------|------|
| .NET Analyzers | Built-in, базовые правила | Free |
| SonarAnalyzer.CSharp | Quality + bugs | Free (NuGet) |
| Meziantou.Analyzer | Async + perf | Free |
| Roslynator | Style + refactoring | Free |
| SonarQube Server | Team quality tracking | Free (Community) / Paid |
| SonarCloud | Hosted SonarQube | Free для open-source |
| CodeQL | Security | Free (GitHub native) |
| Snyk | Security + dependencies | Free / Paid |
| NDepend | Architecture deep | Paid |
| Stryker.NET | Mutation testing | Free |
| NetArchTest | Architecture rules | Free |

---

## См. также

- [[project-setup|Project Setup]] — Directory.Build.props, .editorconfig
- [[source-generators|Source Generators]] — близкие технологии
- [[reflection-expression-trees|Reflection и Expression Trees]] — Roslyn API basics
- [[arch-tests|Architecture Tests]] — NetArchTest
- [[testing|Testing]] — общие best practices
- [[code-quality|Code Quality]] — общие best practices

## Reading list

- **Roslyn API documentation** — learn.microsoft.com/dotnet/csharp/roslyn-sdk
- **SonarSource Rules** — rules.sonarsource.com/csharp
- **Meziantou Rules** — github.com/meziantou/Meziantou.Analyzer/blob/main/docs/Rules.md
- **Roslynator Rules** — github.com/dotnet/roslynator/blob/main/src/Analyzers/README.md
- **Writing Roslyn Analyzers** — Tim Heuer blog posts
- **Stryker.NET docs** — stryker-mutator.io
- **SonarQube docs** — docs.sonarqube.org
- **Code Quality book** — Bertrand Meyer (теория OO design and quality)
