---
tags: [aspire, orchestration, distributed, devex, deployment]
level: Senior
date: 2026-08-02
---

# Aspire — оркестрация распределённых приложений

> Canonical deep-dive по Aspire. Закрывает: AppHost-композиция, ServiceDefaults, Dashboard, aspire CLI (init/run/publish/deploy), integrations (hosting vs client), testing, decision tree против docker-compose и k8s.

---

## Что это, зачем и когда

### Что такое Aspire в 2026

**Aspire** — opinionated-платформа для разработки, запуска и деплоя распределённых приложений: composition модель в коде + service discovery + observability из коробки + генерация деплой-артефактов.

Ключевой сдвиг ноября 2025 (**Aspire 13**, .NET Conf): бренд «.NET Aspire» умер — теперь просто **Aspire** (сайт — aspire.dev). Версия прыгнула с 9.x сразу на 13, чтобы отвязаться от нумерации .NET. Причина ребрендинга не маркетинг: платформа стала **polyglot** — Python и JavaScript/TypeScript получили first-class статус (свои hosting-интеграции, автогенерация Dockerfile, отладка), а с 13.4 AppHost можно писать на TypeScript (GA). Инструментарий при этом построен на .NET: CLI и AppHost требуют .NET 10 SDK.

Второй сдвиг: Aspire — уже **не «только dev experience»**. Модель приложения из AppHost прогоняется через publish/deploy pipeline: `aspire publish` генерирует Docker Compose / Kubernetes-манифесты (Helm) / Bicep, `aspire deploy` доводит до Azure Container Apps или k8s-кластера. Runtime'ом в проде Aspire по-прежнему **не является** — там работают compose/k8s/ACA; Aspire — это модель и конвейер к ним.

### Хронология версий

| Версия | Когда | Что |
|--------|-------|-----|
| 8.0 | Май 2024 | GA «.NET Aspire» — dev-time оркестрация для .NET |
| 9.x | 2024–2025 | Итерации, `azd`-деплой в ACA |
| **13.0** | Ноябрь 2025 | Ребрендинг в Aspire, polyglot (Python/JS), single-file AppHost, `aspire deploy` |
| 13.2 | Март 2026 | CLI как интерфейс для AI-агентов, TypeScript AppHost (preview) |
| 13.3 | Май 2026 | First-class Kubernetes/AKS деплой с Helm |
| 13.4 | Июнь 2026 | TypeScript AppHost GA, typed resource commands |

Актуальная на август 2026 — **13.4.x**. Всё, что написано про «.NET Aspire 8/9» (в т.ч. деплой только через `azd`), — устаревшая картина.

### Зачем

| Без Aspire | С Aspire |
|------------|----------|
| docker-compose.yml + руками прописанные connection strings | Композиция в C#: typed, refactor-safe, `WithReference` |
| OTel/health checks/resilience копипастятся в каждый сервис | `AddServiceDefaults()` — одна строка на сервис |
| «Подними Postgres, Redis, RabbitMQ, потом F5» | `aspire run` — весь граф + dashboard |
| Отдельно писать compose для dev и манифесты для прод | Одна модель → `aspire publish` генерирует артефакты |
| Логи/трейсы размазаны по терминалам | Dashboard: traces + logs + metrics всех сервисов |

### Когда применять

- Solution с 2+ сервисами и backing services (БД, кэш, брокер) — основной кейс
- Polyglot-команда: .NET API + Python ML-сервис + React/Vite фронт в одном графе
- Как локальный OTel-viewer — даже без оркестрации (standalone dashboard)
- **Не** для одиночного монолита без зависимостей и не как замена k8s в проде

---

## 1. AppHost — композиция приложения

AppHost — отдельный проект (или один файл), описывающий **граф ресурсов**: проекты, контейнеры, executables и связи между ними.

```csharp
// AppHost/Program.cs
var builder = DistributedApplication.CreateBuilder(args);

var postgres = builder.AddPostgres("postgres")
    .WithDataVolume();                  // данные переживают перезапуск
var db = postgres.AddDatabase("appdb");

var cache = builder.AddRedis("cache");
var rabbit = builder.AddRabbitMQ("rabbit").WithManagementPlugin();

var api = builder.AddProject<Projects.MyApp_Api>("api")
    .WithReference(db)                  // инжектит connection string
    .WithReference(cache)
    .WithReference(rabbit)
    .WaitFor(db)                        // старт после готовности БД
    .WithExternalHttpEndpoints();

builder.AddProject<Projects.MyApp_Web>("web")
    .WithReference(api);                // service discovery по имени "api"

// Polyglot: Python и JS — first-class с Aspire 13
builder.AddUvicornApp("ml", "../ml-service", "main:app")
    .WithReference(db);
builder.AddViteApp("frontend", "../frontend")
    .WithReference(api);

builder.Build().Run();
```

Основные фабрики ресурсов:

| API | Что добавляет |
|-----|---------------|
| `AddProject<T>("name")` | .NET-проект (ссылка на csproj через generated `Projects.*`) |
| `AddContainer("name", "image")` | Произвольный контейнер |
| `AddExecutable(...)` | Произвольный процесс |
| `AddPythonApp` / `AddPythonModule` / `AddUvicornApp` | Python (uv/pip автодетект, `Aspire.Hosting.Python`) |
| `AddJavaScriptApp` / `AddViteApp` / `AddNodeApp` | JS/TS (`Aspire.Hosting.JavaScript`; старый `AddNpmApp` — deprecated в 13) |
| `AddPostgres` / `AddRedis` / `AddRabbitMQ` / ... | Hosting-интеграции (см. §5) |

### WithReference — сердце модели

`WithReference(resource)` делает две вещи в зависимости от типа ресурса:

1. **Backing service** (БД, кэш, брокер) → инжектит **connection string** в environment: `ConnectionStrings__appdb`. С Aspire 13 connection выдаётся и в polyglot-форматах — URI, JDBC, отдельные `HOST`/`PORT`/`USERNAME`/`PASSWORD`-переменные, чтобы Python/JS-сервисы читали без .NET-конвенций.
2. **Сервис с endpoint'ами** → включает **service discovery**: клиент ходит на `http://api`, а резолвинг в конкретный host:port делает `Microsoft.Extensions.ServiceDiscovery` через env vars (`services__api__https__0`; с 13 дополнительно упрощённые `API_HTTPS`).

Никаких hardcoded портов и строк подключения в конфиге сервисов — всё выводится из графа. Именованные ссылки: `WithReference(resource, "customName")`.

### Single-file AppHost (13+)

Для прототипов csproj не нужен — file-based C# с директивами:

```csharp
// apphost.cs
#:sdk Aspire.AppHost.Sdk@13.4.0
#:package Aspire.Hosting.Redis@13.4.0

var builder = DistributedApplication.CreateBuilder(args);

var cache = builder.AddRedis("cache");
builder.AddProject("api", "../MyApi").WithReference(cache);

builder.Build().Run();
```

> [!question]- Интервью: Aspire заменяет Kubernetes?
> Нет. Aspire — **модель приложения + dev-оркестратор + генератор артефактов**, а не production runtime. Локально он запускает контейнеры через Docker/Podman, в проде — генерирует compose/k8s/Bicep (`aspire publish`) и деплоит (`aspire deploy`), но дальше работу выполняют сами Compose/k8s/ACA. Правильный ответ: Aspire стоит **перед** k8s в pipeline, а не вместо него.

---

## 2. ServiceDefaults — production-defaults одной строкой

Проект `MyApp.ServiceDefaults` (генерируется шаблоном) подключается в каждом сервисе:

```csharp
var builder = WebApplication.CreateBuilder(args);
builder.AddServiceDefaults();   // всё нижеперечисленное
// ...
var app = builder.Build();
app.MapDefaultEndpoints();      // /health + /alive (только в Development)
```

Что именно включает `AddServiceDefaults()`:

| Компонент | Конкретно |
|-----------|-----------|
| **OpenTelemetry** | `ConfigureOpenTelemetry()`: metrics + tracing — ASP.NET Core, HttpClient, runtime instrumentation; OTLP-экспорт при заданном `OTEL_EXPORTER_OTLP_ENDPOINT` |
| **Resilience** | `ConfigureHttpClientDefaults` + `AddStandardResilienceHandler()` — retry, circuit breaker, timeout на **каждом** HttpClient |
| **Service discovery** | `AddServiceDiscovery()` — резолвинг `http://api` из env vars |
| **Health checks** | `AddDefaultHealthChecks()` — liveness-проверка `self` |

`MapDefaultEndpoints()` маппит `/health` (readiness — все проверки) и `/alive` (liveness — только tag `live`) — **по умолчанию только в Development**: в проде health-endpoint'ы без защиты — surface для DoS и утечки топологии. Хочешь их в проде — открывай осознанно (auth, отдельный порт). Трейсы на сами `/health`/`/alive` исключены из instrumentation, чтобы не шуметь.

Это не магический пакет, а **обычный код в твоём проекте** — шаблон генерирует `Extensions.cs`, который можно читать и править. Фактически — материализованный чеклист production-readiness.

---

## 3. Dashboard — локальный OTel-viewer

При `aspire run` поднимается dashboard: **Resources** (статус/endpoint'ы/env каждого ресурса), **Console logs**, **Structured logs**, **Traces** (distributed, с межсервисными span'ами), **Metrics**. Это полноценный OTLP-приёмник — то, ради чего локально обычно поднимали Jaeger + Prometheus + Grafana.

Dashboard работает и **standalone** — как OTel-viewer для любого приложения (не обязательно Aspire):

```bash
docker run --rm -it -p 18888:18888 -p 4317:18889 -p 4318:18890 \
    -d --name aspire-dashboard \
    mcr.microsoft.com/dotnet/aspire-dashboard:latest
```

UI — `:18888`, OTLP gRPC — `4317`, OTLP HTTP — `4318`. Направь `OTEL_EXPORTER_OTLP_ENDPOINT` любого сервиса на него — получишь трейсы без всякого AppHost. Телеметрия **in-memory** (лимиты настраиваются env-переменными `DASHBOARD__TELEMETRYLIMITS__*`) — это viewer, не хранилище; долговременный backend — [[observability|Observability]].

С 13.x dashboard включает **MCP server** для AI-агентов: `list_resources`, `list_console_logs`, `list_structured_logs`, `list_traces`, `execute_resource_command` — агент дебажит приложение через тот же интерфейс, что и человек. Исключить ресурс из MCP — `ExcludeFromMcp()`.

---

## 4. aspire CLI

Ставится скриптом (это не dotnet tool):

```bash
# Linux/macOS
curl -sSL https://aspire.dev/install.sh | bash
# Windows
irm https://aspire.dev/install.ps1 | iex
```

Ключевые команды (полный список — aspire.dev/reference/cli):

| Команда | Что делает |
|---------|-----------|
| `aspire new` | Новый проект из шаблона (Blazor+Minimal API, React+FastAPI, Empty) |
| `aspire init` | Добавить Aspire в **существующий** solution: анализирует репо, создаёт AppHost (можно single-file) + ServiceDefaults |
| `aspire run` | Запуск AppHost в dev-режиме с dashboard |
| `aspire add <integration>` | Добавить hosting-интеграцию в AppHost (`aspire add redis`) |
| `aspire update` * | Обновить Aspire-пакеты; `--self` — сам CLI |
| `aspire publish` * | Сгенерировать деплой-артефакты в `aspire-output/` (compose/k8s/Bicep) |
| `aspire deploy` * | publish + выполнение деплоя в target (ACA, k8s) |
| `aspire destroy` * | Снести задеплоенное окружение |
| `aspire do` * | Выполнить отдельные шаги pipeline (с параллелизмом) |
| `aspire doctor` | Диагностика окружения (SDK, Docker, сертификаты) |
| `aspire certs` | Dev-сертификаты HTTPS (trust across languages) |
| `aspire secret` | User secrets для AppHost |
| `aspire mcp` / `aspire agent mcp` | MCP-инструменты ресурсов / MCP-сервер для агентов |

`*` — на 13.4 официально в preview. Для CI — флаг `--non-interactive` или `ASPIRE_NON_INTERACTIVE=1`.

Смысловая пара publish/deploy: **publish** сериализует модель в параметризованные артефакты (секреты — плейсхолдерами), **deploy** резолвит параметры и применяет к среде. Состояние деплоя (subscription, resource group, значения параметров) кэшируется локально между запусками — сброс через `--clear-cache`.

---

## 5. Integrations — hosting vs client пакеты

Каждая интеграция — **два разных пакета для двух разных мест**, и их путают чаще всего:

| | Hosting integration | Client integration |
|--|--------------------|--------------------|
| Куда ставится | **AppHost** | Сервис (API/Worker) |
| Что делает | Поднимает ресурс (контейнер), health check, генерирует connection string | Регистрирует SDK-клиент в DI + health check + telemetry + retries |
| Naming | `Aspire.Hosting.*` | `Aspire.<SDK>` |

Примеры пар:

| Ресурс | AppHost (hosting) | Сервис (client) |
|--------|-------------------|-----------------|
| PostgreSQL | `Aspire.Hosting.PostgreSQL` → `AddPostgres("db")` | `Aspire.Npgsql` → `AddNpgsqlDataSource("db")`; EF: `Aspire.Npgsql.EntityFrameworkCore.PostgreSQL` → `AddNpgsqlDbContext<T>("db")` |
| Redis | `Aspire.Hosting.Redis` → `AddRedis("cache")` | `Aspire.StackExchange.Redis` → `AddRedisClient("cache")` |
| RabbitMQ | `Aspire.Hosting.RabbitMQ` → `AddRabbitMQ("rabbit")` | `Aspire.RabbitMQ.Client` → `AddRabbitMQClient("rabbit")` |
| Kafka | `Aspire.Hosting.Kafka` | `Aspire.Confluent.Kafka` |
| MongoDB | `Aspire.Hosting.MongoDB` | `Aspire.MongoDB.Driver` |

Связывает их **имя ресурса**: `AddPostgres("db")` в AppHost + `WithReference(db)` у проекта → client-интеграция `AddNpgsqlDataSource("db")` находит connection string по тому же имени из configuration. Клиент при этом сам вешает health check, метрики и трейсинг конкретного SDK.

Client-интеграции работают и **без AppHost** — обычный сервис с connection string в `ConnectionStrings:db` получает те же DI-клиент + телеметрию. Сотни интеграций сверх официальных — **CommunityToolkit.Aspire** (Ollama, Golang, Java, ...).

> [!question]- Интервью: чем client integration отличается от простой регистрации `NpgsqlDataSource` руками?
> Функционально клиентский пакет — это связка «DI-регистрация + health check + OTel-instrumentation + опции retry» для конкретного SDK, сконфигурированная по конвенции имени ресурса. Руками — 30–50 строк на каждый SDK в каждом сервисе, и всегда что-то забывают (обычно health check). Механизм тот же (`IServiceCollection`), ценность — в стандартизации.

---

## 6. Publish & deploy — от модели к артефактам

Target-среда объявляется прямо в AppHost как ресурс:

```csharp
var builder = DistributedApplication.CreateBuilder(args);

// один из вариантов (или несколько для разных сред):
builder.AddDockerComposeEnvironment("compose");          // Aspire.Hosting.Docker
builder.AddKubernetesEnvironment("k8s");                 // Aspire.Hosting.Kubernetes (Helm; AKS first-class с 13.3)
builder.AddAzureContainerAppEnvironment("production");   // Aspire.Hosting.Azure.AppContainers

// ... остальной граф без изменений
builder.Build().Run();
```

- `aspire publish` → в `aspire-output/`: docker-compose.yaml / k8s-манифесты + Helm chart / Bicep — **из той же модели**, что крутится локально. Секреты — параметры-плейсхолдеры, не значения.
- `aspire deploy` → publish + сборка/push образов (`WithContainerRegistry(env)`) + применение к среде: ACA, AKS/k8s.
- Сгенерированные артефакты можно коммитить и вести дальше руками/GitOps — Aspire не обязан оставаться в петле.

Это ответ на главный исторический упрёк «Aspire — это только dev, а для прода всё переписывать в YAML»: YAML теперь **выход компиляции модели**, а не второй источник правды.

---

## 7. Testing — Aspire.Hosting.Testing

`DistributedApplicationTestingBuilder` поднимает **весь граф AppHost** в интеграционном тесте — реальные контейнеры, реальный service discovery:

```csharp
var appHost = await DistributedApplicationTestingBuilder
    .CreateAsync<Projects.MyApp_AppHost>(ct);
await using var app = await appHost.BuildAsync(ct);
await app.StartAsync(ct);

using var client = app.CreateHttpClient("api");
await app.ResourceNotifications.WaitForResourceHealthyAsync("api", ct);

var response = await client.GetAsync("/todos", ct);
Assert.Equal(HttpStatusCode.OK, response.StatusCode);
```

Ключевое: `WaitForResourceHealthyAsync` перед запросами (иначе flaky — контейнеры стартуют секунды) и `CreateHttpClient("name")` вместо hardcoded URL. Ниша — сквозные сценарии через несколько сервисов; для одного сервиса дешевле `WebApplicationFactory` + Testcontainers. Детально, включая соотношение с Testcontainers — [[testing|Testing]].

---

## 8. Aspire vs docker-compose vs k8s-манифесты руками

```
Что оркестрируем?
│
├── Один сервис + 1-2 backing services, команда не .NET-центрична?
│   → docker-compose: 20 строк YAML, нулевой порог входа
│
├── Solution 2+ сервисов, хочется F5-опыт + телеметрию локально?
│   → Aspire: typed-композиция, dashboard, ServiceDefaults
│   │
│   ├── Прод — ACA или AKS?
│   │   → aspire publish/deploy — родной путь (13.3+ Helm/AKS first-class)
│   │
│   ├── Прод — свой k8s с GitOps (Argo/Flux)?
│   │   → Aspire для dev + publish как генератор стартовых манифестов;
│   │     дальше манифесты живут в git и правятся руками — это норм
│   │
│   └── Прод — bare docker-compose на VPS?
│       → AddDockerComposeEnvironment + publish; либо compose руками,
│         Aspire оставить только для dev
│
├── Polyglot (Python/JS/Go) без .NET вообще?
│   → Aspire уже умеет (13+), но ценность ниже: без .NET-сервисов
│     теряется ServiceDefaults; честно сравнить с compose + Tilt
│
└── Нужен тонкий контроль k8s (operators, CRDs, service mesh, network policies)?
    → манифесты/Helm руками; Aspire-генерация — только заготовка
```

Ортогональное правило: **Aspire и compose — не взаимоисключающие**. Aspire для inner loop (F5, отладка, телеметрия), сгенерированный compose/k8s — для CI и прод. Один источник правды — AppHost.

---

## 9. Common Pitfalls

### 1. «Aspire — это прод-runtime»

Нет процесса «Aspire server» в проде. AppHost при `aspire run` — dev-оркестратор; при publish/deploy — компилятор модели в артефакты. Если задеплоил через `aspire deploy` в ACA — крутится ACA, Aspire там нет. Механизм важен: не надо «ставить Aspire на сервер» и не надо бояться, что «Aspire упадёт и уронит прод».

### 2. Hosting-пакет в сервисе (или client — в AppHost)

`Aspire.Hosting.Redis` в API-проекте — ошибка: тащит orchestration-модель в рантайм сервиса. Симптом — «зачем моему API `DistributedApplication`?». Правило: `Aspire.Hosting.*` живёт **только** в AppHost; сервисы ставят client-пакеты (`Aspire.StackExchange.Redis`).

### 3. Забытый `WaitFor` → гонки старта

`WithReference` передаёт connection string, но **не** ждёт готовности ресурса. API стартует раньше Postgres → connection refused на старте (или flaky-тесты). `WaitFor(db)` — ресурс станет healthy до запуска зависимого. В тестах то же самое — `WaitForResourceHealthyAsync`.

### 4. `/health` в проде «не работает» — by design

`MapDefaultEndpoints()` маппит health-endpoint'ы только в Development. k8s-проба в прод бьётся в 404 → рестарты подов. Решение — осознанно включить endpoint'ы для прод-среды (и ограничить доступ), а не копировать шаблон вслепую.

### 5. Знания уровня «.NET Aspire 8/9»

Старые статьи говорят: деплой только `azd`, JS через `AddNpmApp`, «это чисто dev-tool». В 13: `AddNpmApp` — deprecated (→ `AddJavaScriptApp`), деплой — `aspire deploy`, версии 10–12 не существуют. При апгрейде с 9.x читать breaking changes (выпилен старый publishing API: `WithPublishingCallback` и др.).

### 6. Секреты в AppHost хардкодом

`.WithEnvironment("POSTGRES_PASSWORD", "secret123")` в AppHost → утечка в git. Правильно: `builder.AddParameter("db-password", secret: true)` — значение локально живёт в user secrets (управление — `aspire secret`), а в publish-артефакты попадает плейсхолдером и резолвится при deploy.

### 7. Dashboard как хранилище телеметрии

Телеметрия dashboard — in-memory с лимитами: перезапустил AppHost — история умерла. Для чего-либо длиннее одной dev-сессии — полноценный стек (см. [[observability|Observability]]).

---

## 10. Cheat sheet

```bash
irm https://aspire.dev/install.ps1 | iex   # установка CLI (Windows)
aspire new                                 # новый проект из шаблона
aspire init                                # добавить Aspire в существующий repo
aspire run                                 # запуск графа + dashboard
aspire add redis                           # hosting-интеграция в AppHost
aspire update                              # обновить пакеты (--self — CLI)
aspire publish                             # артефакты в aspire-output/ (preview)
aspire deploy                              # publish + деплой в ACA/k8s (preview)
aspire doctor                              # диагностика окружения
```

```csharp
// AppHost: минимальный полезный граф
var builder = DistributedApplication.CreateBuilder(args);
var db = builder.AddPostgres("postgres").WithDataVolume().AddDatabase("appdb");
var api = builder.AddProject<Projects.Api>("api")
    .WithReference(db).WaitFor(db)
    .WithExternalHttpEndpoints();
builder.AddDockerComposeEnvironment("compose");   // цель для publish
builder.Build().Run();

// Сервис: defaults + клиент
builder.AddServiceDefaults();                     // OTel + resilience + discovery + health
builder.AddNpgsqlDbContext<AppDbContext>("appdb");
app.MapDefaultEndpoints();                        // /health, /alive (Development)
```

| Запомнить | |
|-----------|--|
| Бренд | Aspire (не «.NET Aspire») с 13.0, ноябрь 2025; сайт aspire.dev |
| Версии | 9.x → 13.0 (скачок намеренный); сейчас 13.4.x; AppHost требует .NET 10 SDK |
| Пакеты | `Aspire.Hosting.*` → AppHost; `Aspire.<SDK>` → сервисы |
| Связка | имя ресурса + `WithReference` = connection string + service discovery |
| Прод | publish → compose/k8s(Helm)/Bicep; deploy → ACA/AKS; runtime — не Aspire |

---

## См. также

- [[docker|Docker]] — контейнеры, в которых всё это крутится; SDK container builds
- [[kubernetes|Kubernetes]] — прод-runtime, в который Aspire генерирует манифесты
- [[observability|Observability]] — постоянный OTel-стек вместо dashboard
- [[project-setup|Project Setup]] — место AppHost/ServiceDefaults в структуре решения
- [[testing|Testing]] — интеграционные тесты, Testcontainers vs Aspire.Hosting.Testing
- [[twelve-factor-app|12-Factor App]] — dev/prod parity, который Aspire закрывает

## Reading list

- **aspire.dev** — docs, integration catalog, CLI reference (canonical с ноября 2025)
- **What's new in Aspire 13 / 13.2 / 13.3 / 13.4** — aspire.dev/whats-new — хронология платформы
- **Aspire blog** — devblogs.microsoft.com/aspire — анонсы и roadmap
- **github.com/microsoft/aspire** — исходники + Discussions (roadmap-треды)
- **CommunityToolkit.Aspire** — community-интеграции (Ollama, Golang, Java, ...)
