# Architecture — паттерны и системы

> 16 файлов / ~548 KB. SOLID, DDD, CQRS, microservices, EIP, agent-safe + decision guides и 18 real-world сценариев.

[[README|← Главный README]] · [[INDEX|Полный INDEX]]

---

## 🎯 Где начать

| Кто ты | С чего начать |
|--------|---------------|
| Вообще не понимаю «архитектуру» | [[architecture-basics|`Junior/architecture-basics.md`]] |
| Не понимаю SOLID | [[solid|`Senior/solid.md`]] |
| Проектирую новое приложение | [[real-world-scenarios|`Middle/real-world-scenarios.md`]] — 18 case studies |
| Какой паттерн выбрать? | [[patterns-decision-guide|`Middle/patterns-decision-guide.md`]] |
| Изучаю архитектурные стили | [[architecture-patterns|`Senior/architecture-patterns.md`]] — N-Layer / Clean / VSA |
| DDD первый раз | [[ddd|`Senior/ddd.md`]] |
| Кодит AI-агент, размывает границы | [[agent-safe-architecture|`Senior/agent-safe-architecture.md`]] |
| Готовлюсь к system design интервью | [[system-design|`Senior/system-design.md`]] |
| Microservices vs monolith | [[microservices-vs-monolith|`Middle/microservices-vs-monolith.md`]] |

---

## 📚 Все 16 файлов

### 🌱 Junior

| Файл | Описание |
|------|----------|
| [[architecture-basics|`architecture-basics.md`]] | Что такое архитектура, слои, зависимости — вход в тему |

### 🌿 Middle — decision-making

| Файл | Описание |
|------|----------|
| [[patterns-decision-guide|`patterns-decision-guide.md`]] ⭐ | Какой паттерн под какую задачу |
| [[real-world-scenarios|`real-world-scenarios.md`]] ⭐ | 18 case studies: меню → e-commerce → HFT |
| [[microservices-vs-monolith|`microservices-vs-monolith.md`]] | Когда микросервисы оправданы |

### 🏆 Senior — стили, принципы, процесс

| Файл | Описание |
|------|----------|
| [[architecture-patterns|`architecture-patterns.md`]] | N-Layer / Clean / VSA / Hybrid ⭐ |
| [[solid|`solid.md`]] | SOLID + DRY/KISS/YAGNI |
| [[ddd|`ddd.md`]] | Bounded Contexts, Aggregates, Value Objects |
| [[cqrs-mediatr|`cqrs-mediatr.md`]] | CQRS, MediatR, pipeline behaviors |
| [[distributed-systems|`distributed-systems.md`]] | Saga, outbox, CAP, consistency |
| [[system-design|`system-design.md`]] | System design process & interview |
| [[architecture-decisions|`architecture-decisions.md`]] | ADRs — документирование решений |
| [[arch-tests|`arch-tests.md`]] | NetArchTest, ArchUnit — fitness functions |
| [[twelve-factor-app|`twelve-factor-app.md`]] | 12-factor методология для cloud apps |
| [[webai-csharp-architecture|`webai-csharp-architecture.md`]] | WebAI / [anonymized] project architecture (case study) |
| [[agent-safe-architecture|`agent-safe-architecture.md`]] ⭐ NEW | Архитектура, устойчивая к AI-агентам: физические границы + CI-гейты + human-owned contract tests |
| [[eip-content-based-router|`eip-content-based-router.md`]] ⭐ NEW | EIP Content-Based Router: IProcessor, Choice/When DSL, InOnly vs InOut |

---

## 🔗 Связанные папки

- [[design-patterns|`CSharp/Senior/design-patterns.md`]] — 13 GoF в коде
- [[ef-patterns|`EFCore/Senior/ef-patterns.md`]] — Repository, UoW, Specification
- [`Quality/`](../Quality/) — clean code, refactoring
- [`Infrastructure/`](../Infrastructure/) — messaging для distributed
