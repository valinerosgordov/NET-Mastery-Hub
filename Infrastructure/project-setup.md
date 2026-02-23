---
tags: [project-setup, ci-cd, github-actions, dotnet]
level: Senior
---

# How to Start a New .NET Project in 2026

> По материалам: [How to Start a New .NET Project in 2026](https://antondevtips.com/blog/how-to-start-a-new-dotnet-project-in-2026/)

## 7 шагов для нового проекта

### 1. Directory.Build.props — настройки решения

Файл рядом с `.sln`. Централизует настройки для всех проектов.

```xml
<Nullable>enable</Nullable>
<ImplicitUsings>enable</ImplicitUsings>
<AnalysisLevel>latest</AnalysisLevel>
<AnalysisMode>All</AnalysisMode>
<TreatWarningsAsErrors>true</TreatWarningsAsErrors>
<CodeAnalysisTreatWarningsAsErrors>true</CodeAnalysisTreatWarningsAsErrors>
<EnforceCodeStyleInBuild>true</EnforceCodeStyleInBuild>
```

### 2. Static Code Analysis

Пакеты в Directory.Build.props:
- **SonarAnalyzer.CSharp** — качество, безопасность, code smells
- **Meziantou.Analyzer** — async/await, LINQ, производительность
- **Roslynator.Analyzers** — рефакторинг, идиоматичный C#
- **xunit.analyzers** — проверка тестов

### 3. .editorconfig

Рядом с `.sln`. Правила кодирования, severity для analyzer rules. Nullable, naming conventions, async suffix.

### 4. Central Package Management (Directory.Packages.props)

`ManagePackageVersionsCentrally = true`. Все версии пакетов в одном месте. В `.csproj` — без версий.

### 5. Aspire

- **ServiceDefaults** — общая конфигурация, observability
- **AppHost** — оркестратор, зависимости (PostgreSQL, Redis)
- `aspire publish` — Docker Compose
- Connection strings через environment variables

### 6. OpenTelemetry

В ServiceDefaults: логи, метрики, трейсы. Aspire Dashboard. Для prod — Jaeger, Seq, Grafana.

### 7. GitHub Actions

CI: restore, build, test, Docker build. На каждый push/PR.

## Шаблон

[.NET Backend Project Template](https://antondevtips.com/templates/modular-monolith) — Modular Monolith, Vertical Slices, Clean Architecture.

---

## Best Practices (дополнительно)

- **TreatWarningsAsErrors** — включать с первого дня. На legacy-проекте вводить постепенно, фиксируя по модулям.
- **Directory.Packages.props** — при добавлении пакета сразу указывать версию в CPM. Не оставлять `PackageReference` с версией в `.csproj`.
- **Aspire** — для нового проекта. Для существующего — оценить миграцию: AppHost может потребовать рефакторинга конфигурации.
- **CI cache** — кэшировать `~/.nuget/packages` и `dotnet restore` в GitHub Actions. Сокращает время сборки на 30–50%.
- **Fail fast в pipeline** — сначала `dotnet build`, потом тесты. Не собирать Docker, если build упал.
- **Секреты** — никогда в коде. GitHub Secrets, Azure Key Vault, переменные окружения в CI.

---

## См. также

- [[Topics/CodeQuality/code-quality-best-practices|Code Quality]]
- [[Topics/ProjectSetup/top-10-things-dotnet-2026|Top 10 .NET 2026]]
- [[dotnet-knowledge-base|.NET Knowledge Base]]
