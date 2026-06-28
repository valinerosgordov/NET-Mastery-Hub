# Testing — стратегии и инструменты

> 5 файлов / ~146 KB. Testing pyramid (Junior basics) + xUnit/TestContainers stack (Senior tools).

[← Главный README](../readme.md) · [Полный INDEX](../INDEX.md)

---

## 🎯 Где начать

| Кто ты | С чего начать |
|--------|---------------|
| Junior, что такое tests | [`testing-fundamentals.md`](Junior/testing-fundamentals.md) |
| Senior, нужны tools | [`testing.md`](Senior/testing.md) — xUnit, TUnit, TestContainers |
| Делаю integration tests | [`integration-testing.md`](Senior/integration-testing.md) |
| Mock'и vs fakes? | [`mocking-strategies.md`](Middle/mocking-strategies.md) |
| Mutation / load testing | [`mutation-load-testing.md`](Senior/mutation-load-testing.md) |

---

## 📚 Все 5 файлов

### Fundamentals

| Файл | Уровень | Описание |
|------|---------|----------|
| [`testing-fundamentals.md`](Junior/testing-fundamentals.md) | Junior/Senior | Test pyramid, FIRST, TDD ⭐ start here |
| [`testing.md`](Senior/testing.md) | Senior | xUnit, TUnit, TestContainers stack |

> ⚠️ **`testing.md` ≠ `testing-fundamentals.md`** — это разные уровни:  
> `testing-fundamentals` — теория и подходы (Junior)  
> `testing` — конкретные инструменты и stack 2026 (Senior)

### Specialized

| Файл | Описание |
|------|----------|
| [`integration-testing.md`](Senior/integration-testing.md) | TestContainers, WebApplicationFactory |
| [`mocking-strategies.md`](Middle/mocking-strategies.md) | Moq, NSubstitute, fakes vs mocks |
| [`mutation-load-testing.md`](Senior/mutation-load-testing.md) | Stryker.NET, NBomber |

---

## 🔗 Связанные папки

- [`Architecture/Senior/arch-tests.md`](../Architecture/Senior/arch-tests.md) — architecture-level tests
- [`Quality/`](../Quality/) — code quality в целом
- [`Performance/Middle/bottleneck-analysis.md`](../Performance/Middle/bottleneck-analysis.md) — perf testing related
