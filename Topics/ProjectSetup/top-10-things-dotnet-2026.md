# Top 10 Things Every .NET Developer Needs to Do in 2026



## 1. Миграция на .NET 10

LTS до ноября 2028. C# 14, File-Based Apps, OpenAPI 3.1, SSE, JSON Patch в Minimal APIs, Named Query Filters в EF 10.

## 2. Aspire

Оркестрация, service discovery, конфигурация, observability. AppHost + ServiceDefaults. Docker, Azure, AWS.

## 3. Code Quality с первого дня

- Directory.Build.props + TreatWarningsAsErrors
- .editorconfig
- Static analyzers (Meziantou, Sonar, Roslynator)
- AI code review (Code Rabbit)
- SonarQube / Qodana при необходимости

## 4. Observability везде

OpenTelemetry — логи, метрики, трейсы. Для монолитов и микросервисов. Jaeger, Seq.

## 5. DevOps: GitHub Actions / Azure DevOps

CI/CD на каждый PR. Build, test, deploy. Секреты в vault. Manual approval для prod.

## 6. Облако: Azure / AWS / GCP

Минимум один провайдер. Azure — естественный выбор для .NET. Compute, DB, Storage, Messaging, Monitoring.

## 7. Тесты: Integration, Load, Architecture

- **Integration** — Testcontainers, WebApplicationFactory, Respawn
- **Load** — NBomber, k6
- **Architecture** — NetArchTest, проверка слоёв и модулей

## 8. Modular Monolith вместо микросервисов

Martin Fowler: не начинать с микросервисов. Modular Monolith — границы модулей, один деплой, путь к микросервисам позже.

## 9. События между модулями

Слабая связанность, асинхронность. MassTransit, RabbitMQ.

## 10. Обучение

Новый фреймворк, книга. Clean Architecture, Pragmatic Clean Architecture (Milan Jovanovic).

---

## Best Practices (дополнительно)

- **Миграция на .NET 10** — сначала один пилотный проект. Использовать .NET Upgrade Assistant, проверить breaking changes.
- **Aspire** — начать с AppHost + один API. Не тянуть весь legacy в Aspire сразу.
- **Code Quality** — один analyzer за раз. Meziantou часто даёт много предупреждений — настраивать через .editorconfig, не отключать всё.
- **Observability** — correlation id с первого дня. Без него трейсы бесполезны при расследовании.
- **DevOps** — pipeline в YAML в репозитории. Не хранить логику в UI (Azure DevOps, GitHub).
- **Облако** — начать с managed-сервисов (App Service, Cosmos, Blob). IaaS (VM) — только при необходимости.
- **Тесты** — integration-тесты с Testcontainers в отдельном job. Не блокировать merge на медленных тестах.
- **Modular Monolith** — границы модулей фиксировать в architecture tests. Иначе они размываются.

---

## См. также

- [[Topics/ProjectSetup/start-dotnet-project-2026|Start .NET Project 2026]]
- [[dotnet-knowledge-base|.NET Knowledge Base]]
