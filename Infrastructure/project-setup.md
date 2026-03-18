---
tags: [project-setup, ci-cd, github-actions, dotnet]
level: Senior
---

# How to Start a New .NET Project in 2026

> По материалам: [How to Start a New .NET Project in 2026](https://antondevtips.com/blog/how-to-start-a-new-dotnet-project-in-2026/)

## Что это, зачем и когда

### Что это?
Чеклист настроек, которые нужно сделать **в самом начале** проекта. Если не настроить сразу — потом будет больно: предупреждения копятся, стиль кода разный у всех, зависимости конфликтуют.

**Аналогия:** Фундамент дома. Закладываешь ОДИН раз в начале. Если криво — потом весь дом перекосит. Переделывать фундамент готового дома — дорого и больно.

### Зачем?
- **Directory.Build.props** — единые настройки для ВСЕХ проектов (nullable, warnings as errors). Без этого — каждый проект настраиваешь отдельно.
- **Analyzers** — автоматически находят баги, code smells, уязвимости. Без них — баги попадают в production.
- **EditorConfig** — единый стиль кода для всей команды. Без него — каждый пишет по-своему, PR невозможно ревьюить.
- **Central Package Management** — одна версия NuGet-пакета для всех проектов. Без этого — конфликты версий, «у меня работает».
- **CI/CD** — автоматическая проверка каждого коммита. Без этого — «забыл прогнать тесты».

### Когда применять?
- **Каждый новый проект** — с ПЕРВОГО дня
- **Существующий проект** — при первой возможности (чем раньше, тем дешевле)

---

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

### 5. .NET Aspire

**Что это:** Фреймворк для cloud-native .NET приложений. Оркестрация сервисов, автоматическая конфигурация, встроенный dashboard для мониторинга.

**Аналогия:** Docker Compose на стероидах. Aspire знает про .NET и автоматически настраивает connection strings, health checks, OpenTelemetry.

#### Структура проекта

```
MySolution/
├── MyApp.AppHost/          ← Оркестратор (точка входа для dev)
│   └── Program.cs
├── MyApp.ServiceDefaults/  ← Общая конфигурация для всех сервисов
│   └── Extensions.cs
├── MyApp.Api/              ← Web API
└── MyApp.Worker/           ← Background Worker
```

#### AppHost — оркестратор

```csharp
// MyApp.AppHost/Program.cs
var builder = DistributedApplication.CreateBuilder(args);

// Инфраструктура
var postgres = builder.AddPostgres("postgres")
    .WithDataVolume()              // persist данных
    .WithPgAdmin();                // UI для БД

var db = postgres.AddDatabase("appdb");

var redis = builder.AddRedis("cache")
    .WithRedisCommander();         // UI для Redis

// Сервисы
var api = builder.AddProject<Projects.MyApp_Api>("api")
    .WithReference(db)             // автоматический connection string
    .WithReference(redis)
    .WithExternalHttpEndpoints();  // публичный endpoint

builder.AddProject<Projects.MyApp_Worker>("worker")
    .WithReference(db)
    .WithReference(redis);

builder.Build().Run();
```

#### ServiceDefaults — общая конфигурация

```csharp
// MyApp.ServiceDefaults/Extensions.cs
public static class Extensions
{
    public static IHostApplicationBuilder AddServiceDefaults(
        this IHostApplicationBuilder builder)
    {
        // OpenTelemetry: логи, метрики, трейсы
        builder.ConfigureOpenTelemetry();

        // Health Checks
        builder.AddDefaultHealthChecks();

        // HttpClient resilience по умолчанию
        builder.Services.ConfigureHttpClientDefaults(http =>
        {
            http.AddStandardResilienceHandler();
        });

        // Service Discovery
        builder.Services.AddServiceDiscovery();

        return builder;
    }
}

// В каждом сервисе:
builder.AddServiceDefaults();
```

#### Aspire Dashboard

При запуске AppHost автоматически поднимается dashboard:
- **Traces** — distributed tracing всех сервисов
- **Metrics** — CPU, память, HTTP запросы
- **Logs** — structured logs всех сервисов
- **Resources** — статус каждого контейнера/проекта

#### Публикация

```bash
# Генерация Docker Compose / Kubernetes manifests
aspire publish --output-path ./deploy

# Или через CLI
dotnet run --project MyApp.AppHost -- publish
```

| Aspire vs Docker Compose | Aspire | Docker Compose |
|--------------------------|--------|---------------|
| Connection strings | Автоматически | Вручную в .env |
| Health Checks | Автоматически | Вручную в YAML |
| OpenTelemetry | Встроено | Настраивать самому |
| Service Discovery | Автоматически | Через DNS / nginx |
| Dashboard | Из коробки | Grafana / Portainer |
| Подходит для | .NET приложения | Любые контейнеры |

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
