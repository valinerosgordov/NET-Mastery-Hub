# Best Practices for Increasing Code Quality in .NET

> По материалам: [Best Practices for Code Quality in .NET](https://antondevtips.com/blog/best-practices-for-increasing-code-quality-in-dotnet-projects/)

## Инструменты

| Инструмент | Назначение |
|------------|------------|
| Code Review | Ловит то, что не видят автоматические инструменты |
| Static Code Analysis | Стандарты, code smells, баги при сборке |
| Code Analysis Software | SonarQube, Qodana, Codacy |
| IDE | Real-time feedback |

## Directory.Build.props

Рядом с `.sln`. Настройки для всего решения:

- `Nullable` — nullable reference types
- `ImplicitUsings` — автоматические using
- `AnalysisLevel` — latest
- `AnalysisMode` — All
- `TreatWarningsAsErrors` — нет предупреждений
- `CodeAnalysisTreatWarningsAsErrors`
- `EnforceCodeStyleInBuild`

Пакеты: Meziantou.Analyzer, SonarAnalyzer.CSharp, Roslynator.Analyzers.

## .editorconfig

Severity для правил. Indentation, naming, conventions. `csharp_prefer_braces = true:error`.

## IDE плагины

- **Rider:** Grazie (spell), Code Metrics, Cognitive Complexity
- **VS:** Spell Checker, Code Metrics

## Итог

Качество — с первого дня. TreatWarningsAsErrors не даёт накапливать техдолг.

---

## Best Practices (дополнительно)

- **Meziantou** — может быть шумным. Отключать правила по одному в .editorconfig, не `severity = none` для всего.
- **SonarAnalyzer** — в CI, не в каждом build. Локально — только при необходимости, иначе замедляет IDE.
- **Nullable** — включать в новых проектах. В legacy — `#nullable enable` по файлам, не глобально.
- **Pre-commit** — `dotnet build` в pre-commit hook. Ловит ошибки до push.
- **Code review checklist** — не дублировать то, что ловят analyzers. Фокус на архитектуре, бизнес-логике, безопасности.
- **Suppress** — `#pragma warning disable` с комментарием. Иначе через месяц никто не помнит, почему отключено.

---

## См. также

- [[Architecture/architecture-conventions-and-tests|Соглашения и тесты]]
- [[Topics/ProjectSetup/start-dotnet-project-2026|Start .NET Project 2026]]
- [[dotnet-knowledge-base|.NET Knowledge Base]]
