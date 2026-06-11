# Architecture — паттерны и системы

> 12 файлов / 385 KB. SOLID, DDD, CQRS, microservices + decision guides и 18 real-world сценариев.

[← Главный README]() · [Полный INDEX]()

---

## 🎯 Где начать

| Кто ты                             | С чего начать                                                                    |
| ---------------------------------- | -------------------------------------------------------------------------------- |
| Не понимаю SOLID                   | [`solid.md`]()                                                           |
| Проектирую новое приложение        | [`real-world-scenarios.md`]() — 18 case studies           |
| Какой паттерн выбрать?             |  [`patterns-decision-guide.md`]()                      |
| Изучаю архитектурные стили         | [`patterns.md`]() — N-Layer / Clean / VSA |
| DDD первый раз                     | [`ddd.md`]()                                                               |
| Готовлюсь к system design интервью | [`system-design.md`]()                                           |
| Microservices vs monolith          | [`microservices-vs-monolith.md`]()                   |

---

## 📚 Все 12 файлов

### Принципы

| Файл | Описание |
|------|----------|
| [`solid.md`]() | SOLID + DRY/KISS/YAGNI |

### Архитектурные стили

| Файл | Описание |
|------|----------|
| [`patterns.md`]() | N-Layer / Clean / VSA / Hybrid (52 KB) |
| [`ddd.md`]() | Bounded Contexts, Aggregates, Value Objects |
| [`cqrs-mediatr.md`]() | CQRS, MediatR, pipeline behaviors |

### Distributed systems

| Файл | Описание |
|------|----------|
| [`distributed-systems.md`]() | Saga, outbox, CAP, consistency |
| [`microservices-vs-monolith.md`]() | Когда микросервисы оправданы |

### Decision-making (⭐ NEW)

| Файл | Описание |
|------|----------|
| ⭐ [`patterns-decision-guide.md`]() | Какой паттерн под какую задачу |
| ⭐ [`real-world-scenarios.md`]() | 18 case studies: меню → e-commerce → HFT |

### Process

| Файл | Описание |
|------|----------|
| [`system-design.md`]() | System design process & interview |
| [`architecture-decisions.md`]() | ADRs — документирование решений |
| [`arch-tests.md`]() | NetArchTest, ArchUnit — fitness functions |

### Project-specific

| Файл | Описание |
|------|----------|
| [`webai-csharp-architecture.md`](webai-csharp-architecture.md) | WebAI / [anonymized] project architecture |

---

## 🔗 Связанные папки

- [`CSharp/design-patterns`]() — 13 GoF в коде
- [`EFCore/patterns`]() — Repository, UoW, Specification
- [`Quality/`](../Quality/) — clean code, refactoring
- [`Infrastructure/messaging`]() — для distributed
