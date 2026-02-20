# Clean Architecture и Vertical Slices: детальный разбор

> По материалам: [N-Layered vs Clean vs VSA](https://antondevtips.com/blog/n-layered-vs-clean-vs-vertical-slice-architecture), [Clean + VSA](https://antondevtips.com/blog/the-best-way-to-structure-your-dotnet-projects-with-clean-architecture-and-vertical-slices), [VSA структура](https://antondevtips.com/blog/vertical-slice-architecture-the-best-ways-to-structure-your-project).

---

## Содержание

1. [N-Layered Architecture](#1-n-layered-architecture)
2. [Clean Architecture](#2-clean-architecture)
3. [Vertical Slice Architecture](#3-vertical-slice-architecture)
4. [Гибрид: Clean Architecture + Vertical Slices](#4-гибрид-clean-architecture--vertical-slices)
5. [Варианты структуры Vertical Slices](#5-варианты-структуры-vertical-slices)
6. [Сводка: плюсы и минусы архитектур](#6-сводка-плюсы-и-минусы-архитектур)
7. [Когда что выбирать](#7-когда-что-выбирать)

---

## 1. N-Layered Architecture

### Суть

N-Layered (Controller-Service-Repository) — распространённый подход в .NET. Слои:

| Слой | Назначение |
|------|------------|
| **Data Access (Repository)** | Абстракция над персистентностью |
| **Business Logic (Service)** | Бизнес-правила, оркестрация |
| **Presentation** | Controllers, API endpoints, UI |

Типичная структура: `/Models`, `/Repositories`, `/Services`, `/Controllers`. Простая для понимания, привычная большинству разработчиков.

### Пример кода

```csharp
[ApiController]
[Route("api/shipments")]
public class ShipmentsController : ControllerBase
{
    private readonly IShipmentService _service;
    public ShipmentsController(IShipmentService service) => _service = service;

    [HttpGet("{id}")]
    public async Task<IActionResult> Get(int id)
    {
        var shipment = await _service.GetShipmentByIdAsync(id);
        return Ok(shipment);
    }
}

public class ShipmentService : IShipmentService
{
    private readonly IShipmentRepository _repository;
    public ShipmentService(IShipmentRepository repository) => _repository = repository;

    public async Task<ShipmentDto> GetShipmentByIdAsync(int id) =>
        await _repository.GetByIdAsync(id);
}

public class ShipmentRepository : IShipmentRepository
{
    private readonly ShipmentDbContext _dbContext;
    public async Task<ShipmentDto> GetByIdAsync(int id) =>
        await _dbContext.Shipments
            .Where(s => s.Id == id)
            .Select(s => new ShipmentDto { Number = s.Number, OrderId = s.OrderId })
            .FirstOrDefaultAsync();
}
```

### Проблема 1: Fat Controllers и Fat Services

С ростом требований Controller и Service быстро разрастаются. Вместо 4 методов — десятки:

```csharp
public class ShipmentsController : ControllerBase
{
    [HttpGet("{id}")] public async Task<IActionResult> GetShipment(int id);
    [HttpGet("user/{userId}")] public async Task<IActionResult> GetShipmentsByUser(int userId);
    [HttpGet("date-range")] public async Task<IActionResult> GetShipmentsByDateRange(DateTime from, DateTime to);
    [HttpPost] public async Task<IActionResult> CreateShipment(CreateShipmentRequest request);
    [HttpPut("{id}")] public async Task<IActionResult> UpdateShipment(int id, UpdateShipmentRequest request);
    [HttpPatch("{id}/status")] public async Task<IActionResult> UpdateShipmentStatus(int id, ShipmentStatus status);
    [HttpDelete("{id}")] public async Task<IActionResult> DeleteShipment(int id);
    [HttpPost("{id}/track")] public async Task<IActionResult> TrackShipment(int id);
    [HttpPost("{id}/cancel")] public async Task<IActionResult> CancelShipment(int id);
    [HttpPost("{id}/approve")] public async Task<IActionResult> ApproveShipment(int id);
}
```

Добавить метод проще, чем вынести новый Controller. Service и Repository растут аналогично.

### Проблема 2: Слишком много мелких Services и Repositories

При росте домена (Shipments, ShipmentItems, Orders) появляются отдельные репозитории на сущность. Но куда класть кросс-сущностные запросы?

- Загрузка shipment с историей (несколько сущностей)
- Order с связанными shipments
- Shipment со всеми items

Методы размазываются по репозиториям. При новой фиче непонятно, куда добавлять метод. Результат: либо много мелких репозиториев с 1–2 методами, либо «толстые» репозитории, знающие о других сущностях.

### Проблема 3: Слабая бизнес-логика, сложное тестирование

- Правила разбросаны по нескольким сервисам — трудно понять и изменить.
- Нет жёстких ограничений: можно обойти слой (например, вызвать Repository из Controller).
- Интерфейсы часто опускают — код сложнее тестировать.
- Дублирование логики между сервисами.
- N+1 при оркестрации нескольких репозиториев.

### Вывод по N-Layered

Подходит для простых CRUD и прототипов. Для сложных доменов, модульных монолитов и микросервисов часто становится ограничением. Многие команды переходят на Clean Architecture или VSA.

---

## 2. Clean Architecture

### Суть

Разделение приложения на слои с чёткой ответственностью. Цель — высокая связность внутри слоя и слабая связанность между слоями. Зависимости направлены внутрь: внешние слои зависят от внутренних, Domain не зависит ни от чего.

### Слои (изнутри наружу)

| Слой | Назначение | Зависит от |
|------|------------|------------|
| **Domain** | Сущности, value objects, доменные сервисы, интерфейсы репозиториев | Ничего |
| **Application** | Use cases, оркестрация, интерфейсы внешних сервисов | Domain |
| **Infrastructure** | БД, кэш, очереди, внешние API, реализация репозиториев | Application, Domain |
| **Presentation** | Web API, gRPC, GraphQL, MVC, консоль | Application, Infrastructure |

### Схема зависимостей

```
                    ┌─────────────────────────────────────┐
                    │           Presentation               │
                    │  (Controllers, Endpoints, gRPC)      │
                    └──────────────────┬──────────────────┘
                                       │
                    ┌──────────────────▼──────────────────┐
                    │          Infrastructure             │
                    │  (EF Core, Redis, RabbitMQ, etc.)   │
                    └──────────────────┬──────────────────┘
                                       │
                    ┌──────────────────▼──────────────────┐
                    │           Application                │
                    │  (Use cases, Commands, Queries)       │
                    └──────────────────┬──────────────────┘
                                       │
                    ┌──────────────────▼──────────────────┐
                    │             Domain                   │
                    │  (Entities, Value Objects, Rules)   │
                    └─────────────────────────────────────┘
```

### Плюсы

| Преимущество | Описание |
|--------------|----------|
| **Separation of Concerns** | Каждый слой решает свою задачу, код проще понимать и менять |
| **Testability** | Домен и use cases тестируются без БД, сети, UI |
| **Flexibility** | Смена БД, фреймворка, UI с минимальным влиянием на домен |
| **Code Reusability** | Домен и прикладная логика переиспользуются в разных UI/каналах |
| **Long-term Adaptability** | Бизнес-логика изолирована от технологий и фреймворков |

### Минусы

| Недостаток | Описание |
|------------|----------|
| **Complexity** | Много слоёв и абстракций, для простых проектов — overkill |
| **Overhead** | Интерфейсы, маппинг, DI — больше boilerplate |
| **Learning Curve** | Нужно время, чтобы правильно применять принципы |
| **Initial Setup** | Больше времени на старт по сравнению с простым CRUD |

### Типичная структура проектов

```
src/
├── MyApp.Domain/           # Entities, Value Objects, Domain Services
├── MyApp.Application/      # Use cases, CQRS, Interfaces
├── MyApp.Infrastructure/   # EF Core, Redis, HTTP clients
└── MyApp.Api/              # ASP.NET Core, Controllers/Endpoints
```

### Проблема Clean Architecture

Один use case размазан по нескольким проектам: Command в Application, Handler там же, Controller в Api, Repository в Infrastructure. Чтобы понять фичу целиком, приходится прыгать между проектами.

### Pragmatic Clean Architecture: EF Core в Handlers

Современный подход — использовать EF Core напрямую в Command/Query handlers, без репозиториев.

**Почему это допустимо:**

1. **DbContext = Repository + Unit of Work** — в документации EF Core DbContext уже реализует эти паттерны. Репозиторий поверх DbContext — абстракция над абстракцией.
2. **Смена БД в проде** — в большинстве проектов не происходит. При смене SQL-провайдера (Postgres → SQL Server) большая часть кода EF Core не меняется. Переход на MongoDB — по сути переписывание.
3. **Тестирование** — In-Memory DbContext для unit-тестов; для логики доступа к данным — интеграционные тесты.
4. **Дублирование запросов** — паттерн Specification или выборочный репозиторий для часто используемых запросов.

```csharp
// Без репозитория — DbContext напрямую
internal sealed class CreateShipmentCommandHandler(
    ShipmentsDbContext context,
    ILogger<CreateShipmentCommandHandler> logger)
    : IRequestHandler<CreateShipmentCommand, ErrorOr<CreateShipmentResponse>>
{
    public async Task<ErrorOr<CreateShipmentResponse>> Handle(
        CreateShipmentCommand request,
        CancellationToken cancellationToken)
    {
        var shipmentAlreadyExists = await context.Shipments
            .AnyAsync(x => x.OrderId == request.OrderId, cancellationToken);
        if (shipmentAlreadyExists)
            return Error.Conflict($"Shipment for order '{request.OrderId}' is already created");

        var shipment = request.MapToShipment();
        await context.Shipments.AddAsync(shipment, cancellationToken);
        await context.SaveChangesAsync(cancellationToken);
        return shipment.MapToResponse();
    }
}
```

### Rich Domain Model vs Anemic

**Anemic model** — сущность с публичными get/set, без логики:

```csharp
public class Shipment
{
    public Guid Id { get; set; }
    public string Number { get; set; }
    public ShipmentStatus Status { get; set; }
    // ... любой может изменить Status извне, обходя проверки
}
```

Правила размазываются по сервисам, легко обойти валидацию.

**Rich Domain Model** — логика в сущности, приватные сеттеры, фабричные методы:

```csharp
public class Shipment
{
    public Guid Id { get; private set; }
    public ShipmentStatus Status { get; private set; }
    // ...

    public static Shipment Create(string number, string orderId, ...) { }

    public ErrorOr<Success> Process()
    {
        if (Status is not ShipmentStatus.Created)
            return Error.Validation("Can only update to Processing from Created status");
        Status = ShipmentStatus.Processing;
        return Result.Success;
    }

    public ErrorOr<Success> Dispatch()
    {
        if (Status is not ShipmentStatus.Processing)
            return Error.Validation("Can only update to Dispatched from Processing status");
        Status = ShipmentStatus.Dispatched;
        return Result.Success;
    }
}
```

Правила в одном месте, use case только вызывает методы домена.

### Feature Folders в Clean Architecture

Классическая Clean Architecture — папки по техническим ролям: `/CommandHandlers`, `/Queries`, `/Controllers`. Связанный код разбросан.

**Feature Folders** — группировка по use case: `/Features/Shipments/CreateShipment` содержит всё для одной фичи. Навигация и изменения локализованы.

---

## 3. Vertical Slice Architecture

### Суть

Структура по фичам (вертикальные срезы), а не по техническим слоям. Один срез = одна фича: endpoint, валидация, бизнес-логика, доступ к данным. Всё, что нужно для фичи, собрано в одном месте.

### Сравнение с горизонтальными слоями

```
Горизонтально (N-Tier / Clean):     Вертикально (VSA):
┌─────────────────────────────┐     ┌──────┬──────┬──────┐
│      Controllers            │     │Slice1│Slice2│Slice3│
├─────────────────────────────┤     │  UI  │  UI  │  UI  │
│      Application            │     │ Logic│ Logic│ Logic│
├─────────────────────────────┤     │ Data │ Data │ Data │
│      Data Access            │     └──────┴──────┴──────┘
└─────────────────────────────┘
```

### Плюсы

| Преимущество | Описание |
|--------------|----------|
| **Reduced Coupling** | Слабая связь между срезами, изменения локализованы |
| **Maintainability** | Вся фича в одном месте, проще навигация и поддержка |
| **Flexibility** | В каждом срезе можно использовать свои подходы и технологии |
| **Scalability** | Команды могут работать над разными фичами параллельно |
| **Feature Focused** | Изменения в одной фиче не ломают другие |

### Минусы

| Недостаток | Описание |
|------------|----------|
| **Много классов и файлов** | Крупное приложение = много срезов и файлов |
| **Consistency** | Нужна дисциплина: общие concerns, стиль, конвенции |
| **Duplication** | Риск дублирования между срезами |

### Митигация недостатков

- **Много файлов** — выбор структуры (см. раздел 5): один файл на срез или разбиение по папкам.
- **Consistency** — MediatR pipelines для логирования, валидации, обработки ошибок; общие базовые классы.
- **Duplication** — shared-папки внутри bounded context, вынос общей логики в отдельные модули.

---

## 4. Гибрид: Clean Architecture + Vertical Slices

### Идея

Взять сильные стороны обоих подходов:

- **От Clean Architecture:** Domain-centric дизайн, изоляция домена, Infrastructure как отдельный слой.
- **От VSA:** Организация по фичам, быстрая навигация, локализация изменений.

### Как комбинировать

1. **Domain** — без изменений. Сущности, value objects, доменные сервисы.
2. **Infrastructure** — без изменений. БД, кэш, очереди, внешние API.
3. **Application + Presentation** — объединяются в вертикальные срезы. Каждый срез содержит endpoint, command/query, handler, валидацию, маппинг.

### Структура решения

```
src/
├── MyApp.Domain/              # Entities, Value Objects (DDD)
├── MyApp.Infrastructure/     # EF Core, Repositories, External APIs
└── MyApp.Api/                 # или MyApp.Features
    ├── Features/
    │   ├── Shipments/
    │   │   ├── CreateShipment/
    │   │   │   ├── CreateShipmentEndpoint.cs
    │   │   │   ├── CreateShipmentCommand.cs
    │   │   │   ├── CreateShipmentCommandHandler.cs
    │   │   │   ├── CreateShipmentRequest.cs
    │   │   │   ├── CreateShipmentValidator.cs
    │   │   │   └── CreateShipmentMapper.cs
    │   │   ├── GetShipment/
    │   │   └── DispatchShipment/
    │   └── Orders/
    └── ...
```

### Domain (DDD)

Домен остаётся центром. Сущности инкапсулируют бизнес-логику:

```csharp
public class Shipment
{
    public Guid Id { get; private set; }
    public string Number { get; private set; }
    public ShipmentStatus Status { get; private set; }
    // ...

    public ErrorOr<Success> Process()
    {
        if (Status is not ShipmentStatus.Created)
            return Error.Validation("Can only update to Processing from Created status");
        Status = ShipmentStatus.Processing;
        UpdatedAt = DateTime.UtcNow;
        return Result.Success;
    }

    public ErrorOr<Success> Dispatch()
    {
        if (Status is not ShipmentStatus.Processing)
            return Error.Validation("Can only update to Dispatched from Processing status");
        Status = ShipmentStatus.Dispatched;
        UpdatedAt = DateTime.UtcNow;
        return Result.Success;
    }
}
```

Use case в срезе вызывает методы домена, а не дублирует правила.

### Сложные vs простые срезы

**Сложный срез** — MediatR, Command/Query, Handler, валидация, маппинг:

```csharp
// CreateShipmentCommandHandler
var response = await mediator.Send(command, cancellationToken);
```

**Простой срез** — логика прямо в endpoint, репозиторий или DbContext:

```csharp
// DispatchShipmentEndpoint
var shipment = await repository.GetByNumberAsync(shipmentNumber, cancellationToken);
var response = shipment.Dispatch();  // Доменная логика в entity
await unitOfWork.SaveChangesAsync(cancellationToken);
```

Прагматизм: не везде нужен MediatR. Для простых операций достаточно endpoint + домен.

---

## 5. Варианты структуры Vertical Slices

### Вариант 1: Feature-Based Folders (много файлов)

Каждый срез — папка. Внутри — отдельные файлы для Request, Response, Command, Handler, Endpoint, Validator.

```
CreateShipment/
├── CreateShipmentRequest.cs
├── CreateShipmentResponse.cs
├── CreateShipmentCommand.cs
├── CreateShipmentCommandHandler.cs
├── CreateShipmentEndpoint.cs
└── CreateShipmentValidator.cs
```

| Плюсы | Минусы |
|-------|--------|
| Понятная структура, всё разложено по файлам | Много файлов, медленная навигация |
| Подходит для сложных handlers | |

---

### Вариант 2: Один файл на срез, вложенные классы

Всё в одном файле, внутри статического класса. Короткие имена: `Request`, `Response`, `Command`, `CommandHandler`.

```csharp
public static class CreateShipment
{
    public sealed record Request(...);
    public sealed record Response(...);
    internal sealed record Command(...) : IRequest<ErrorOr<Response>>;
    internal sealed class CommandHandler(...) : IRequestHandler<Command, ErrorOr<Response>> { }
    public class Validator : AbstractValidator<Request> { }
    public static void MapEndpoint(WebApplication app) { }
}
```

| Плюсы | Минусы |
|-------|--------|
| Вся фича в одном месте, быстрая навигация | Глубокое вложение, высокий файл при сложной логике |
| Короткие имена | |

---

### Вариант 3: Один основной файл + вынесенные concerns (рекомендуется)

Основная логика среза в одном файле. Validator, Mapper — в отдельных файлах.

```
CreateShipment/
├── CreateShipment.cs          # Request, Command, Handler, Endpoint
├── CreateShipmentValidator.cs
└── CreateShipmentMapper.cs
```

| Плюсы | Минусы |
|-------|--------|
| Баланс между количеством файлов и читаемостью | Нужны полные имена классов |
| Удобно выносить общую логику в shared | |
| Быстрая навигация по фиче | |

---

### Вариант 4: Pragmatic (всё в endpoint)

Без MediatR. Вся логика в endpoint, DbContext или репозиторий инжектируется напрямую.

```csharp
public class CreateShipmentEndpoint : IEndpoint
{
    public void MapEndpoint(WebApplication app) => app.MapPost("/api/shipments", Handle);

    private static async Task<IResult> Handle(
        [FromBody] CreateShipmentRequest request,
        IValidator<CreateShipmentRequest> validator,
        EfCoreDbContext context,
        CancellationToken ct)
    {
        var validationResult = await validator.ValidateAsync(request, ct);
        if (!validationResult.IsValid)
            return Results.ValidationProblem(validationResult.ToDictionary());

        // Вся логика здесь
        var shipment = request.MapToShipment();
        context.Shipments.Add(shipment);
        await context.SaveChangesAsync(ct);
        return Results.Ok(shipment.MapToResponse());
    }
}
```

| Плюсы | Минусы |
|-------|--------|
| Простота, мало кода | Сложнее unit-тесты |
| Подходит для небольших сервисов | Нет единого места для cross-cutting (логирование, валидация) |
| | Риск дублирования |

---

## 6. Сводка: плюсы и минусы архитектур

### N-Layered Architecture

#### Плюсы

| Плюс | Описание |
|------|----------|
| **Простота** | Понятная структура: Controllers → Services → Repositories. Легко объяснить новичку |
| **Быстрый онбординг** | Почти каждый .NET-разработчик уже работал с таким подходом |
| **Привычность** | Много примеров, туториалов, legacy-кода в таком стиле |
| **Быстрый старт** | Минимум настройки, можно сразу писать код |
| **Базовое разделение** | Есть хотя бы разделение на слои (в отличие от «всё в одном») |
| **Подходит для простого CRUD** | Для приложений без сложной логики — достаточный уровень абстракции |

#### Минусы

| Минус | Описание |
|-------|----------|
| **Fat Controllers / Fat Services** | С ростом требований классы раздуваются, ответственность размывается |
| **Много мелких репозиториев** | Неясно, куда класть кросс-сущностные запросы; репозитории либо слишком мелкие, либо слишком крупные |
| **Разбросанная бизнес-логика** | Правила распределены по сервисам, трудно найти и изменить |
| **Слабая изоляция** | Можно обойти слой (например, вызвать Repository из Controller) |
| **Сложное тестирование** | Часто опускают интерфейсы, используют конкретные реализации — моки затруднены |
| **Дублирование** | Похожая логика в разных сервисах |
| **N+1 и оркестрация** | Сервисы координируют несколько репозиториев — риск N+1 и неоптимальных запросов |
| **Плохая масштабируемость** | При росте домена структура быстро деградирует |

---

### Clean Architecture

#### Плюсы

| Плюс | Описание |
|------|----------|
| **Separation of Concerns** | Чёткое разделение: Domain, Application, Infrastructure, Presentation. Каждый слой — своя зона ответственности |
| **Testability** | Домен и use cases изолированы от БД, сети, UI. Unit-тесты без моков инфраструктуры |
| **Flexibility** | Смена БД, фреймворка, UI с минимальным влиянием на ядро. Зависимости направлены внутрь |
| **Code Reusability** | Домен и прикладная логика переиспользуются в разных каналах (API, gRPC, консоль, workers) |
| **Long-term Adaptability** | Бизнес-логика не привязана к технологиям, проще адаптироваться к изменениям |
| **Дисциплина** | Dependency Inversion и правила зависимостей задают жёсткие ограничения |
| **Rich Domain Model** | Удобно инкапсулировать логику в сущностях, избегать anemic model |
| **Защита домена** | Domain не знает про БД, HTTP, фреймворки — изменения снаружи не ломают ядро |

#### Минусы

| Минус | Описание |
|-------|----------|
| **Complexity** | Много слоёв и абстракций. Для простых проектов — overkill |
| **Overhead** | Интерфейсы, маппинг, DI, проекты — больше boilerplate и ceremony |
| **Learning Curve** | Нужно время, чтобы правильно применять принципы и не «перегибать» |
| **Initial Setup** | Больше времени на старт по сравнению с N-Layered |
| **Размазанный use case** | Один use case разбросан по нескольким проектам (Command в Application, Handler там же, Controller в Api, Repository в Infrastructure) |
| **Навигация** | Чтобы понять фичу целиком, приходится прыгать между проектами и папками |
| **Риск over-engineering** | Репозитории поверх DbContext, лишние абстракции «на будущее» |

---

### Vertical Slice Architecture

#### Плюсы

| Плюс | Описание |
|------|----------|
| **Reduced Coupling** | Слабая связь между срезами. Изменения в одной фиче не затрагивают другие |
| **Maintainability** | Вся фича в одном месте. Проще навигация, понимание и поддержка |
| **Flexibility** | В каждом срезе можно использовать свои подходы и технологии |
| **Scalability команды** | Разные команды и разработчики могут работать над разными фичами параллельно |
| **Feature Focused** | Изменения локализованы, меньше риска побочных эффектов |
| **Быстрая разработка** | Добавление новой фичи — один срез, без правок в нескольких слоях |
| **Быстрый онбординг** | Новый разработчик быстро понимает конкретную фичу |
| **Минимум «божественных» классов** | Каждый срез — небольшая, сфокусированная единица |

#### Минусы

| Минус | Описание |
|-------|----------|
| **Много классов и файлов** | Крупное приложение = много срезов. Каждый срез — несколько файлов (Request, Command, Handler, Endpoint, Validator) |
| **Consistency** | Нужна дисциплина: общие cross-cutting concerns, стиль, конвенции. Без этого — разнобой между срезами |
| **Duplication** | Риск дублирования логики между срезами. Нужны shared-папки и вынос общей логики |
| **Нет явного Domain-слоя** | В «чистом» VSA домен может раствориться в срезах. Для сложного домена лучше комбинировать с Clean |
| **Cross-cutting concerns** | Логирование, валидация, обработка ошибок — нужно продумывать (MediatR pipelines, middleware) |
| **Высокие файлы** | При варианте «один файл на срез» файлы могут стать очень большими |

---

### Clean Architecture + Vertical Slices (гибрид)

#### Плюсы

| Плюс | Описание |
|------|----------|
| **Domain-centric + Feature-centric** | Изоляция домена (Clean) и организация по фичам (VSA) одновременно |
| **Быстрая навигация** | Вся фича в одном срезе, при этом Domain и Infrastructure вынесены отдельно |
| **Масштабируемость** | Команды работают над разными фичами; Domain и Infrastructure общие, без дублирования |
| **Гибкость** | Проще добавлять и менять фичи; структура остаётся предсказуемой |
| **Тестируемость** | Домен изолирован, use cases тестируются без инфраструктуры |
| **Прагматизм** | Можно использовать DbContext в handlers (pragmatic Clean), простые срезы — без MediatR |
| **Лучшее из двух миров** | Строгость Clean + скорость и удобство VSA |

#### Минусы

| Минус | Описание |
|-------|----------|
| **Сложность** | Комбинация двух подходов. Нужно понимать оба и уметь их сочетать |
| **Онбординг** | Новым разработчикам нужно время, чтобы освоить и Clean, и VSA |
| **Overhead** | Domain и Infrastructure — отдельные проекты. Больше структуры, чем в «чистом» VSA |
| **Не для простых проектов** | Для простого CRUD или микросервиса без сложного домена — избыточно |
| **Баланс** | Нужно находить баланс: где MediatR, где прямой вызов; где репозиторий, где DbContext |

---

### Сравнительная таблица

| Критерий | N-Layered | Clean | VSA | Clean + VSA |
|----------|:--------:|:-----:|:---:|:-----------:|
| Простота | ✅✅✅ | ❌ | ✅✅ | ✅ |
| Тестируемость | ❌ | ✅✅✅ | ✅✅ | ✅✅✅ |
| Масштабируемость кода | ❌ | ✅✅ | ✅✅✅ | ✅✅✅ |
| Масштабируемость команды | ❌ | ✅✅ | ✅✅✅ | ✅✅✅ |
| Скорость разработки | ✅✅ | ✅ | ✅✅✅ | ✅✅ |
| Защита домена | ❌ | ✅✅✅ | ⚠️ | ✅✅✅ |
| Навигация по фиче | ❌ | ❌ | ✅✅✅ | ✅✅✅ |
| Онбординг | ✅✅✅ | ❌ | ✅✅ | ✅ |
| Подходит для простого CRUD | ✅✅✅ | ❌ | ✅✅ | ⚠️ |
| Подходит для сложного домена | ❌ | ✅✅✅ | ⚠️ | ✅✅✅ |

---

## 7. Когда что выбирать

### N-Layered Architecture

| Когда использовать | Сценарии |
|--------------------|----------|
| **Быстрый старт** | Нужно быстро что-то сделать, команда знакома с N-Layered |
| **Простая логика** | Прямолинейный CRUD, минимум бизнес-правил |
| **Маленькая команда** | 1–3 разработчика |
| **Прототип / PoC** | Область ограничена, больших изменений не ожидается |
| **Junior-команда** | Нужна простая, привычная структура |
| **Data-driven** | В основном работа с данными, мало доменной логики |

### Clean Architecture

| Когда использовать | Сценарии |
|--------------------|----------|
| **Высокая тестируемость** | Нужна изоляция логики от инфраструктуры |
| **Долгосрочный проект** | Поддержка и развитие годами |
| **Сложный домен** | Много правил, Rich Domain Model |
| **Средняя/большая команда** | 4–10 разработчиков |
| **Модульный монолит** | Чёткие границы модулей |
| **Интеграции** | Много внешних систем, смена технологий возможна |

### Vertical Slice Architecture

| Когда использовать | Сценарии |
|--------------------|----------|
| **Разные технологии по фичам** | Разные части системы с разной сложностью |
| **Много независимых фич** | Фичи слабо связаны |
| **Feature-focused разработка** | Спринты по фичам |
| **Быстрый онбординг** | Новые разработчики быстро входят в домен |
| **Скорость разработки** | Минимизация времени на новую фичу |

### Clean Architecture + Vertical Slices

| Когда использовать | Сценарии |
|--------------------|----------|
| **Масштабирование** | Рост и кода, и команды |
| **Сложный домен + много фич** | Rich domain и много use cases |
| **Средние и крупные приложения** | Модульный монолит, микросервисы |
| **Опыт с обоими подходами** | Команда знает Clean и VSA |
| **Скорость + структура** | Хочется и быстрой разработки, и чёткой архитектуры |

### Рекомендация на 2025

Для большинства новых проектов: **Clean Architecture + Vertical Slices**.

Даёт:

- **Масштабируемость команды** — разные команды работают над разными фичами.
- **Гибкость** — проще добавлять и менять фичи.
- **Структура** — Domain-centric дизайн, изоляция домена.
- **Скорость** — вся фича в одном месте, быстрая навигация.

Для простых CRUD и микросервисов без сложного домена — **только VSA** (без Clean), pragmatic-подход с DbContext в endpoint.

### Краткая таблица

| Сценарий | Рекомендация |
|----------|--------------|
| Прототип, PoC, простой CRUD | N-Layered |
| Сложный домен, долгосрочный проект | Clean + VSA |
| Много фич, feature-focused | VSA |
| Простой микросервис | VSA (pragmatic, без MediatR) |
| Модульный монолит | Clean + VSA |

---

## Best Practices (дополнительно)

- **Не смешивать стили** — если выбрали VSA, не добавлять «горизонтальные» папки Controllers/Services в корень.
- **Domain — священная корова** — никаких зависимостей наружу. Если нужен DateTime.UtcNow — IDateTimeProvider в Application.
- **Pragmatic over pure** — DbContext в handlers допустим. Репозиторий поверх EF — только если реально планируется смена БД.
- **Shared kernel осторожно** — в Modular Monolith общий код между модулями ведёт к связанности. Минимум shared.
- **Срезы не более 300 строк** — если файл среза раздулся, вынести в отдельные файлы (Validator, Mapper) или разбить use case.

- [N-Layered vs Clean vs Vertical Slice Architecture](https://antondevtips.com/blog/n-layered-vs-clean-vs-vertical-slice-architecture)
- [The Best Way To Structure Your .NET Projects with Clean Architecture and Vertical Slices](https://antondevtips.com/blog/the-best-way-to-structure-your-dotnet-projects-with-clean-architecture-and-vertical-slices)
- [Vertical Slice Architecture: The Best Ways to Structure Your Project](https://antondevtips.com/blog/vertical-slice-architecture-the-best-ways-to-structure-your-project)

---

## См. также

- [[Architecture/architecture-conventions-and-tests|Соглашения и тесты]]
- [[Architecture/architecture-tests-netarchtest|NetArchTest]]
- [[Topics/ResultPattern/result-pattern-cqrs|Result Pattern + CQRS]]
- [[dotnet-knowledge-base|.NET Knowledge Base]]
