# Testing — стратегии и инструменты

> 5 файлов / ~146 KB. Testing pyramid (Junior basics) + xUnit/TestContainers stack (Senior tools).

[[README|← Главный README]] · [[INDEX|Полный INDEX]]

---

## 🎯 Где начать

| Кто ты | С чего начать |
|--------|---------------|
| Junior, что такое tests | [[testing-fundamentals|`testing-fundamentals.md`]] |
| Senior, нужны tools | [[testing|`testing.md`]] — xUnit, TUnit, TestContainers |
| Делаю integration tests | [[integration-testing|`integration-testing.md`]] |
| Mock'и vs fakes? | [[mocking-strategies|`mocking-strategies.md`]] |
| Mutation / load testing | [[mutation-load-testing|`mutation-load-testing.md`]] |

---

## 📚 Все 5 файлов

### Fundamentals

| Файл | Уровень | Описание |
|------|---------|----------|
| [[testing-fundamentals|`testing-fundamentals.md`]] | Junior/Senior | Test pyramid, FIRST, TDD ⭐ start here |
| [[testing|`testing.md`]] | Senior | xUnit, TUnit, TestContainers stack |

> ⚠️ **`testing.md` ≠ `testing-fundamentals.md`** — это разные уровни:  
> `testing-fundamentals` — теория и подходы (Junior)  
> `testing` — конкретные инструменты и stack 2026 (Senior)

### Specialized

| Файл | Описание |
|------|----------|
| [[integration-testing|`integration-testing.md`]] | TestContainers, WebApplicationFactory |
| [[mocking-strategies|`mocking-strategies.md`]] | Moq, NSubstitute, fakes vs mocks |
| [[mutation-load-testing|`mutation-load-testing.md`]] | Stryker.NET, NBomber |

---

## 🔗 Связанные папки

- [[arch-tests|`Architecture/Senior/arch-tests.md`]] — architecture-level tests
- [`Quality/`](../Quality/) — code quality в целом
- [[bottleneck-analysis|`Performance/Middle/bottleneck-analysis.md`]] — perf testing related
