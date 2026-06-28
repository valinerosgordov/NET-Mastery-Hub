# Architecture — паттерны и системы

> 16 файлов / ~548 KB. SOLID, DDD, CQRS, microservices, EIP, agent-safe + decision guides и 18 real-world сценариев.

[← Главный README](../readme.md) · [Полный INDEX](../INDEX.md)

---

## 🎯 Где начать

| Кто ты | С чего начать |
|--------|---------------|
| Вообще не понимаю «архитектуру» | [`Junior/architecture-basics.md`](Junior/architecture-basics.md) |
| Не понимаю SOLID | [`Senior/solid.md`](Senior/solid.md) |
| Проектирую новое приложение | [`Middle/real-world-scenarios.md`](Middle/real-world-scenarios.md) — 18 case studies |
| Какой паттерн выбрать? | [`Middle/patterns-decision-guide.md`](Middle/patterns-decision-guide.md) |
| Изучаю архитектурные стили | [`Senior/architecture-patterns.md`](Senior/architecture-patterns.md) — N-Layer / Clean / VSA |
| DDD первый раз | [`Senior/ddd.md`](Senior/ddd.md) |
| Кодит AI-агент, размывает границы | [`Senior/agent-safe-architecture.md`](Senior/agent-safe-architecture.md) |
| Готовлюсь к system design интервью | [`Senior/system-design.md`](Senior/system-design.md) |
| Microservices vs monolith | [`Middle/microservices-vs-monolith.md`](Middle/microservices-vs-monolith.md) |

---

## 📚 Все 16 файлов

### 🌱 Junior

| Файл | Описание |
|------|----------|
| [`architecture-basics.md`](Junior/architecture-basics.md) | Что такое архитектура, слои, зависимости — вход в тему |

### 🌿 Middle — decision-making

| Файл | Описание |
|------|----------|
| [`patterns-decision-guide.md`](Middle/patterns-decision-guide.md) ⭐ | Какой паттерн под какую задачу |
| [`real-world-scenarios.md`](Middle/real-world-scenarios.md) ⭐ | 18 case studies: меню → e-commerce → HFT |
| [`microservices-vs-monolith.md`](Middle/microservices-vs-monolith.md) | Когда микросервисы оправданы |

### 🏆 Senior — стили, принципы, процесс

| Файл | Описание |
|------|----------|
| [`architecture-patterns.md`](Senior/architecture-patterns.md) | N-Layer / Clean / VSA / Hybrid ⭐ |
| [`solid.md`](Senior/solid.md) | SOLID + DRY/KISS/YAGNI |
| [`ddd.md`](Senior/ddd.md) | Bounded Contexts, Aggregates, Value Objects |
| [`cqrs-mediatr.md`](Senior/cqrs-mediatr.md) | CQRS, MediatR, pipeline behaviors |
| [`distributed-systems.md`](Senior/distributed-systems.md) | Saga, outbox, CAP, consistency |
| [`system-design.md`](Senior/system-design.md) | System design process & interview |
| [`architecture-decisions.md`](Senior/architecture-decisions.md) | ADRs — документирование решений |
| [`arch-tests.md`](Senior/arch-tests.md) | NetArchTest, ArchUnit — fitness functions |
| [`twelve-factor-app.md`](Senior/twelve-factor-app.md) | 12-factor методология для cloud apps |
| [`webai-csharp-architecture.md`](Senior/webai-csharp-architecture.md) | WebAI / [anonymized] project architecture (case study) |
| [`agent-safe-architecture.md`](Senior/agent-safe-architecture.md) ⭐ NEW | Архитектура, устойчивая к AI-агентам: физические границы + CI-гейты + human-owned contract tests |
| [`eip-content-based-router.md`](Senior/eip-content-based-router.md) ⭐ NEW | EIP Content-Based Router: IProcessor, Choice/When DSL, InOnly vs InOut |

---

## 🔗 Связанные папки

- [`CSharp/Senior/design-patterns.md`](../CSharp/Senior/design-patterns.md) — 13 GoF в коде
- [`EFCore/Senior/ef-patterns.md`](../EFCore/Senior/ef-patterns.md) — Repository, UoW, Specification
- [`Quality/`](../Quality/) — clean code, refactoring
- [`Infrastructure/`](../Infrastructure/) — messaging для distributed
