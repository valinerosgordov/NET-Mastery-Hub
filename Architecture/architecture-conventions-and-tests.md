# Архитектурные соглашения и тесты

> Имена классов, слоёв, обработчиков — это часть архитектуры. Без формальных правил структура разваливается. Архитектурные тесты защищают её автоматически.

---

## Проблема: договорённости в голове

Типичная ситуация:

- «Мы же решили, что Handler заканчивается на CommandHandler»
- «Сервисы называем по домену: OrderService, не OrderManager»
- «Репозитории — только в Infrastructure»

Потом появляются `UserCmdHandler`, `DoStuffManager`, `NewService2`, `DataHelper` — и структура превращается в хаос. Новый разработчик не знает правил. Code review ловит не всё. Wiki устаревает.

**Решение:** архитектурные тесты. Правила формализованы в коде. Нарушение = падающий тест в CI.

---

## Соглашения по именованию

### 1. Handlers (CQRS / MediatR)

| Тип | Суффикс | Пример | Назначение |
|-----|---------|-------|------------|
| Command handler | `CommandHandler` | `CreateOrderCommandHandler` | Обработка команд (изменение состояния) |
| Query handler | `QueryHandler` | `GetOrderByIdQueryHandler` | Обработка запросов (чтение) |
| Notification handler | `NotificationHandler` | `OrderCreatedNotificationHandler` | Обработка уведомлений |
| Request handler | `RequestHandler` | `GetOrderRequestHandler` | Универсальный handler (MediatR IRequest) |

**Правило:** Handler всегда привязан к одному Command/Query/Notification. Имя = `{CommandName}Handler`.

**Антипаттерны:**
- `UserHandler` — неясно, что обрабатывает
- `OrderCmdHandler` — сокращения
- `HandleOrderStuff` — не класс, но плохое имя метода

---

### 2. Commands и Queries

| Тип | Суффикс | Пример |
|-----|---------|-------|
| Command | `Command` | `CreateOrderCommand`, `CancelOrderCommand` |
| Query | `Query` | `GetOrderByIdQuery`, `GetOrdersByCustomerQuery` |
| Result (Query) | `Result` или `Response` | `OrderDetailsResult`, `OrderListResponse` |

**Правило:** Глагол в начале (Create, Update, Delete, Get, List). Command — императив, Query — вопрос.

**Антипаттерны:**
- `OrderCommand` — неясное действие
- `OrderQuery` — неясно, что именно запрашивается
- `OrderData` — не Command и не Query

---

### 3. Сервисы

| Тип | Суффикс | Пример | Назначение |
|-----|---------|-------|------------|
| Domain service | `Service` | `OrderPricingService` | Доменная логика, оркестрация |
| Application service | `Service` | `OrderApplicationService` | Use case, координация |
| External integration | `Client` или `Service` | `PaymentGatewayClient`, `EmailService` | Внешние системы |

**Правило:** Имя отражает домен и ответственность. Не `Manager`, не `Helper`, не `Processor` (если это не pipeline).

**Антипаттерны:**
- `OrderManager` — размытая ответственность
- `DoStuffService` — неинформативно
- `Service1`, `NewService` — отсутствие смысла
- `UserHelper` — Helper для всего подряд

---

### 4. Репозитории

| Суффикс | Пример |
|---------|-------|
| `Repository` | `IOrderRepository`, `OrderRepository` |
| Альтернатива (специфично) | `IOrderStore`, `IOrderReadRepository` |

**Правило:** Интерфейс в Domain/Application, реализация в Infrastructure. Имя = сущность + Repository.

**Антипаттерны:**
- `OrderRepo` — сокращения
- `OrderDataAccess` — не репозиторий по смыслу
- `OrderManager` с методами Get/Add — это репозиторий, назвать соответственно

---

### 5. Controllers / Endpoints

| Стиль | Пример |
|-------|-------|
| REST resource | `OrdersController`, `UsersController` |
| Minimal API | `MapGet("/orders", ...)` — группа по ресурсу |
| Vertical Slice | `CreateOrderEndpoint`, `GetOrderEndpoint` |

**Правило:** Контроллер = ресурс. Методы = HTTP-глаголы (Get, Post, Put, Delete). Не смешивать с бизнес-логикой.

**Антипаттерны:**
- `OrderController` с прямым вызовом `_orderRepository` — нарушение слоёв
- `ApiController`, `MainController` — не по ресурсу
- `OrderController` с методом `DoEverything` — размытая ответственность

---

### 6. DTO и модели

| Тип | Суффикс | Где живёт |
|-----|---------|-----------|
| Request DTO | `Request`, `Command` | API / Application |
| Response DTO | `Response`, `Result`, `Dto` | API / Application |
| Domain model | Без суффикса | Domain |
| Entity (EF) | Без суффикса или `Entity` | Infrastructure |

**Правило:** DTO не в Domain. Domain не знает про API-контракты. `CreateOrderRequest` — в API, `Order` — в Domain.

**Антипаттерны:**
- `OrderDto` в Domain — утечка контрактов
- `OrderViewModel` в Domain — View-слой в домене
- Суффикс `Model` для всего подряд — неинформативно

---

### 7. Validators

| Суффикс | Пример |
|---------|--------|
| `Validator` | `CreateOrderCommandValidator` |

**Правило:** Validator привязан к Command/Query. `{CommandName}Validator`.

---

### 8. Mappers

| Суффикс | Пример |
|---------|--------|
| `Mapper` | `OrderMapper` |
| Profile (AutoMapper) | `OrderMappingProfile` |

**Правило:** Один mapper на сущность или пару (Entity ↔ DTO).

---

### 9. Middleware и Filters

| Тип | Суффикс | Пример |
|-----|---------|--------|
| Middleware | `Middleware` | `CorrelationIdMiddleware` |
| Filter | `Filter` | `ValidationFilter`, `AuthorizationFilter` |

---

### 10. Исключения

| Суффикс | Пример |
|---------|--------|
| `Exception` | `OrderNotFoundException`, `ValidationException` |

**Правило:** Доменные исключения в Domain. Технические — в Infrastructure/Shared.

---

## Запрещённые суффиксы

| Суффикс | Почему плохо |
|---------|--------------|
| `Manager` | Размытая ответственность, «божественный объект» |
| `Helper` | Свалка статических методов без чёткой области |
| `Util`, `Utils` | То же, что Helper |
| `Processor` | Если не pipeline — неясно, что обрабатывает |
| `Handler` без контекста | `UserHandler` — что именно обрабатывает? |
| Цифры в имени | `Service2`, `HelperV2` — нет семантики |

---

## Зависимости между слоями

### Clean Architecture / Vertical Slice

```
Domain          — не зависит ни от чего
    ↑
Application     — зависит от Domain
    ↑
Infrastructure  — зависит от Application, Domain
    ↑
API             — зависит от Application, Infrastructure
```

**Правила:**
- Domain не ссылается на Application, Infrastructure, API
- Application не ссылается на Infrastructure, API
- Controllers не вызывают репозитории напрямую — только через Application (handlers, services)

---

## Архитектурные тесты

### Инструменты

- **NetArchTest.Rules** — правила для зависимостей и именования
- **ArchUnitNET** — порт ArchUnit (Java) для .NET
- Обычный xUnit + рефлексия — кастомные проверки

### Пример: NetArchTest

```csharp
using NetArchTest.Rules;
using FluentAssertions;

// Именование: все Command handler'ы заканчиваются на CommandHandler
[Fact]
public void CommandHandlers_Should_EndWith_CommandHandler()
{
    var result = Types
        .InAssembly(typeof(CreateOrderCommandHandler).Assembly)
        .That()
        .ResideInNamespace("Application.Handlers")
        .And()
        .HaveNameEndingWith("Handler")
        .Should()
        .HaveNameEndingWith("CommandHandler")
        .GetResult();

    result.IsSuccessful.Should().BeTrue(
        $"Violations: {string.Join(", ", result.FailingTypeNames)}");
}

// Зависимости: Application не тянет Infrastructure
[Fact]
public void Application_Should_Not_DependOn_Infrastructure()
{
    var result = Types
        .InAssembly(typeof(CreateOrderCommand).Assembly)
        .That()
        .ResideInNamespace("Application")
        .ShouldNot()
        .HaveDependencyOn("Infrastructure")
        .GetResult();

    result.IsSuccessful.Should().BeTrue(result.FailingTypeNames);
}

// Зависимости: Domain не знает про DTO и API-контракты
[Fact]
public void Domain_Should_Not_Contain_DTOs()
{
    var result = Types
        .InAssembly(typeof(Order).Assembly)
        .That()
        .ResideInNamespace("Domain")
        .ShouldNot()
        .HaveNameEndingWith("Dto")
        .And()
        .ShouldNot()
        .HaveNameEndingWith("Request")
        .And()
        .ShouldNot()
        .HaveNameEndingWith("Response")
        .GetResult();

    result.IsSuccessful.Should().BeTrue(result.FailingTypeNames);
}

// Запрет суффикса Manager (с возможным whitelist)
[Fact]
public void Should_Not_Use_Manager_Suffix()
{
    var result = Types
        .InAssembly(typeof(Order).Assembly)
        .That()
        .ArePublic()
        .ShouldNot()
        .HaveNameEndingWith("Manager")
        .GetResult();

    result.IsSuccessful.Should().BeTrue(result.FailingTypeNames);
}
```

> **Примечание:** Точные условия (`ResideInNamespace`, фильтры) подстраиваются под структуру проекта. Для проверки «Controller не использует Repository» может понадобиться анализ конструктора или кастомное правило.

### Установка NetArchTest

```bash
dotnet add package NetArchTest.Rules
```

---

## Чек-лист правил для тестов

| Правило | Тест |
|---------|------|
| Command handlers заканчиваются на `CommandHandler` | Именование |
| Query handlers заканчиваются на `QueryHandler` | Именование |
| Commands заканчиваются на `Command` | Именование |
| Queries заканчиваются на `Query` | Именование |
| Application не зависит от Infrastructure | Зависимости |
| Domain не зависит от Application/Infrastructure/API | Зависимости |
| Domain не содержит DTO/Request/Response | Зависимости |
| Controllers не ссылаются на Infrastructure | Зависимости |
| Нет классов с суффиксом `Manager` (или whitelist) | Именование |
| Нет классов с суффиксом `Helper` (или whitelist) | Именование |
| Репозитории заканчиваются на `Repository` | Именование |
| Validators заканчиваются на `Validator` | Именование |

---

## Интеграция в CI

Архитектурные тесты — обычные unit-тесты. Запускаются в pipeline вместе с остальными:

```yaml
# GitHub Actions / Azure DevOps
- name: Run tests
  run: dotnet test --filter "Category=Architecture"
```

Или без фильтра — все тесты, включая архитектурные.

---

## Best Practices (дополнительно)

- **Whitelist для исключений** — если `LegacyOrderManager` нельзя переименовать, добавить в whitelist в тесте, не отключать правило.
- **Один тест — одно правило** — не объединять 5 проверок в один тест. При падении должно быть понятно, что именно сломалось.
- **Конвенции в README** — кратко описать правила. Тесты — enforcement, README — onboarding.
- **Code review** — при добавлении нового типа проверить, не нарушает ли он существующие architecture tests.

1. **Именование** — часть архитектуры. Суффиксы задают роль класса.
2. **Договорённости в голове** ненадёжны. Формализуйте их в тестах.
3. **Архитектурные тесты** проверяют структуру, а не бизнес-логику.
4. **CI** не даёт замержить код с нарушением правил.
5. **Новые разработчики** получают обратную связь через падающие тесты, а не через устаревшую Wiki.

Структура, защищённая тестами, сохраняет проект поддерживаемым. Хаос начинается с мелочей: `UserCmdHandler`, `DoStuffManager`, `NewService2`.

---

## См. также

- [[Architecture/architecture-tutorial|Архитектуры]]
- [[Architecture/architecture-tests-netarchtest|NetArchTest]]
- [[Topics/CodeQuality/code-quality-best-practices|Code Quality]]
