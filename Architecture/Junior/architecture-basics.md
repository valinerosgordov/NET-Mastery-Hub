---
tags: [architecture, basics, junior, layers, separation-of-concerns]
level: Junior
date: 2026-08-02
---

# Architecture Basics — основы архитектуры приложений

> **Зачем структурировать код, что такое слои, separation of concerns, dependency direction.** Введение перед [[patterns-decision-guide|Middle/patterns-decision-guide]] и [[architecture-patterns|Senior/architecture-patterns]].

---

## 0. Как читать

Junior уровень — основы. Если уже знаешь "что такое слой" и "почему DI" — иди в Middle/Senior. Здесь — fundamentals для тех кто пишет в один файл и хочет понять зачем разделять.

Cross-language якоря свёрнуты. Interview-вопросы встроены.

---

## 1. Что это, зачем и когда

### 1.1. Что такое архитектура

Структура приложения — **как именно разложен код** на файлы, классы, проекты, и **как они общаются**. Хорошая архитектура отвечает на вопросы:
- Где найти код для X?
- Если изменить Y — что сломается?
- Как добавить новую feature?
- Как протестировать Z?

```
Плохая архитектура:
- Один Program.cs на 5000 строк
- Логика DB / UI / business в одном методе
- Изменить email → нужно править 12 мест
- Тесты невозможны (всё связано со всем)

Хорошая архитектура:
- Понятная структура папок
- Каждый класс — одна ответственность
- Изменить email → 1 файл
- Тесты пишутся легко (зависимости injected)
```

### 1.2. Зачем — что архитектура решает

```
Без архитектуры:
- Onboarding нового разработчика — недели
- Багов больше (changes неожиданно ломают)
- Refactoring страшно (всё связано)
- Тесты не пишутся (нельзя isolate)
- Code review бесполезен (никто не понимает context)
- Технический долг растёт экспоненциально

С архитектурой:
- Onboarding — дни
- Изменения локальны
- Refactoring безопасный
- Unit-тесты быстрые и надёжные
- Code review продуктивный
- Технический долг под контролем
```

### 1.3. Когда задумываться о архитектуре

```
Маленький pet project (< 500 строк):
- Не парься, делай "как удобно"
- Один Program.cs OK

Учебный проект:
- Минимальная структура (controller / service / data)
- Цель: понять concepts

Production app:
- Архитектура — обязательно
- С первого дня

Team project:
- Архитектура критична
- Conventions важнее личных preferences
```

> [!info]- Если ты пишешь скрипты на Python / автоматизацию
> Скрипты могут жить без архитектуры — 200 строк, выполняется и забывается. Production app другой — живёт годами, его меняют десятки людей. Архитектура — про долгосрочную поддержку.

> [!question]- Интервью: что такое good architecture в одном предложении?
> **Good architecture минимизирует cost изменений** — добавить feature, исправить bug, заменить компонент должно быть **локальным** изменением, не cascade through всего приложения. Конкретно: separation of concerns + dependency injection + clear boundaries между слоями.

---

## 2. Separation of Concerns (SoC)

### 2.1. Главный принцип

**Каждый класс / модуль / слой делает одну вещь.** Не смешивай:
- Чтение из БД + бизнес-логика
- Бизнес-логика + UI rendering
- Валидация + отправка email

```csharp
// ❌ Один класс делает всё — нет SoC
public class OrderManager
{
    public void ProcessOrder(int orderId)
    {
        // 1. Читаем из БД (SQL запрос)
        var conn = new SqlConnection("...");
        conn.Open();
        var cmd = new SqlCommand("SELECT * FROM Orders WHERE Id = @id", conn);
        cmd.Parameters.AddWithValue("@id", orderId);
        var reader = cmd.ExecuteReader();
        reader.Read();
        var total = reader.GetDecimal(2);
        
        // 2. Бизнес-логика (расчёт скидки)
        if (total > 1000) total *= 0.9m;
        
        // 3. Отправляем email (SMTP)
        var smtp = new SmtpClient("smtp.example.com");
        var msg = new MailMessage("from@x.com", "to@y.com");
        msg.Subject = "Order processed";
        smtp.Send(msg);
        
        // 4. Логируем (file IO)
        File.AppendAllText("log.txt", $"Order {orderId} processed\n");
    }
}
```

Проблемы:
- Хочешь сменить БД с SQL Server на Postgres — переписывать ProcessOrder.
- Хочешь добавить SMS вместо email — править ProcessOrder.
- Хочешь unit-тест ProcessOrder — нужны реальные SQL/SMTP/file system.
- Один класс — много причин изменения.

> [!warning] `AddWithValue()` — type inference ломает индексы
> `cmd.Parameters.AddWithValue("@id", orderId)` выводит `SqlDbType` из CLR-типа значения, и вывод нередко промахивается (`string` → `nvarchar`, `decimal` с чужими precision/scale). Несовпадение с типом колонки заставляет SQL Server делать implicit conversion и **игнорировать индекс** → seek превращается в scan. Предпочитай явный тип: `cmd.Parameters.Add("@id", SqlDbType.Int).Value = orderId;`. Подробнее — [[security-practices]].

```csharp
// ✅ Разделили по ответственностям
public class OrderRepository   // только БД
{
    public Order GetById(int id) { /* SQL */ }
}

public class DiscountCalculator   // только бизнес-логика
{
    public decimal Apply(decimal total) =>
        total > 1000 ? total * 0.9m : total;
}

public class EmailService   // только отправка email
{
    public void Send(string to, string subject, string body) { /* SMTP */ }
}

public class Logger   // только логирование
{
    public void Log(string message) { /* file */ }
}

public class OrderService   // оркестрация — связывает компоненты
{
    private readonly OrderRepository _repo;
    private readonly DiscountCalculator _discount;
    private readonly EmailService _email;
    private readonly Logger _logger;
    
    public OrderService(OrderRepository repo, DiscountCalculator discount,
                        EmailService email, Logger logger)
    {
        _repo = repo;
        _discount = discount;
        _email = email;
        _logger = logger;
    }
    
    public void ProcessOrder(int orderId)
    {
        var order = _repo.GetById(orderId);
        order.Total = _discount.Apply(order.Total);
        _email.Send(order.CustomerEmail, "Order processed", $"Total: {order.Total}");
        _logger.Log($"Order {orderId} processed");
    }
}
```

Каждый класс — одна причина изменения. Тесты пишутся через mocks.

### 2.2. Single Responsibility Principle (SRP)

SoC на уровне классов = SRP. Robert Martin: "класс должен иметь одну причину изменяться".

```
Признаки нарушения SRP:
- Класс называется ManagerHelperUtility
- В классе > 10 публичных методов
- Методы работают с разными типами данных (User + Order + Email)
- Имя класса с "And" (UserAndOrderProcessor)
- Класс > 500 строк
- В классе много `private` хелперов разной природы
```

### 2.3. Cohesion vs Coupling

```
Cohesion (когезия) — насколько связаны элементы внутри класса/модуля
HIGH cohesion = good (всё в классе про одну тему)

Coupling (связность) — насколько связаны разные классы между собой
LOW coupling = good (классы независимы, легко менять)

Цель: HIGH cohesion + LOW coupling
```

```csharp
// HIGH cohesion: всё в классе UserService про user
public class UserService
{
    public User Create(...) { }
    public void Update(...) { }
    public void Delete(...) { }
    public User GetById(...) { }
}

// LOW cohesion: всё про разное
public class Helper
{
    public User GetUser(...) { }      // про user
    public void SendEmail(...) { }    // про email
    public string FormatDate(...) { } // про даты
    public byte[] CompressFile(...) { } // про файлы
}
```

> [!question]- Интервью: что такое SoC и SRP?
> **Separation of Concerns** — разделять разные ответственности (DB / business logic / UI / I/O) на разные классы/слои. **Single Responsibility Principle** — каждый класс имеет ОДНУ причину изменяться. SRP — application of SoC at class level. Цель: изменения локальны, тесты простые, рефакторинг безопасный.

---

## 3. Layers — слои приложения

### 3.1. Классическая 3-tier architecture

Большинство Web/API приложений следует этой структуре:

```
┌─────────────────────────────────────┐
│  Presentation Layer                  │  ← HTTP, UI, controllers
│  (Controllers, Views, ViewModels)    │
├─────────────────────────────────────┤
│  Business Logic Layer (BLL)          │  ← бизнес-правила
│  (Services, Domain Models)           │
├─────────────────────────────────────┤
│  Data Access Layer (DAL)             │  ← БД, file I/O
│  (Repositories, EF Core, ADO.NET)    │
└─────────────────────────────────────┘
              ↓
        ┌──────────┐
        │ Database │
        └──────────┘
```

### 3.2. Что в каждом слое

**Presentation Layer:**
- HTTP endpoints (`Controllers`, Minimal APIs)
- Request/Response DTOs
- Input validation
- Authentication/Authorization checks
- Mapping HTTP → domain models
- НЕ содержит business logic

```csharp
// UsersController.cs — Presentation
[ApiController]
[Route("api/users")]
public class UsersController : ControllerBase
{
    private readonly IUserService _service;
    
    public UsersController(IUserService service) => _service = service;
    
    [HttpGet("{id}")]
    public async Task<IActionResult> Get(int id)
    {
        var user = await _service.GetByIdAsync(id);
        if (user == null) return NotFound();
        return Ok(new UserDto(user.Id, user.Name, user.Email));
    }
}
```

**Business Logic Layer:**
- Domain models (User, Order, Product)
- Business rules (validation, calculations)
- Workflows (PlaceOrder, CalculateTax)
- Orchestration между repositories
- НЕ знает про HTTP / SQL

```csharp
// IUserService.cs — Business
public interface IUserService
{
    Task<User?> GetByIdAsync(int id);
    Task<int> CreateAsync(string name, string email);
}

public class UserService : IUserService
{
    private readonly IUserRepository _repo;
    
    public UserService(IUserRepository repo) => _repo = repo;
    
    public async Task<User?> GetByIdAsync(int id) => await _repo.GetByIdAsync(id);
    
    public async Task<int> CreateAsync(string name, string email)
    {
        if (string.IsNullOrEmpty(name)) throw new ArgumentException("Name required");
        if (!email.Contains("@")) throw new ArgumentException("Invalid email");
        
        var user = new User { Name = name, Email = email, CreatedAt = DateTime.UtcNow };
        return await _repo.AddAsync(user);
    }
}
```

**Data Access Layer:**
- Database queries (EF Core, Dapper, raw SQL)
- File I/O
- External API calls (HttpClient)
- Caching (Redis, MemoryCache)
- НЕ содержит business logic

```csharp
// IUserRepository.cs — Data Access
public interface IUserRepository
{
    Task<User?> GetByIdAsync(int id);
    Task<int> AddAsync(User user);
}

public class UserRepository : IUserRepository
{
    private readonly AppDbContext _db;
    
    public UserRepository(AppDbContext db) => _db = db;
    
    public Task<User?> GetByIdAsync(int id) => _db.Users.FindAsync(id).AsTask();
    
    public async Task<int> AddAsync(User user)
    {
        _db.Users.Add(user);
        await _db.SaveChangesAsync();
        return user.Id;
    }
}
```

### 3.3. Dependency Direction — стрелки сверху вниз

Критическое правило: **upper layers depend on lower, NOT vice versa.**

```
Presentation
     ↓ depends on
Business
     ↓ depends on
Data Access
     ↓ depends on
Database
```

**Что НЕЛЬЗЯ:**
- DataAccess вызывает Controller — нарушение
- Repository знает про HTTP — нарушение
- Database model в API response — нарушение (нужен DTO)

```csharp
// ❌ Repository знает про HTTP — нарушение dependency direction
public class UserRepository
{
    public IActionResult GetById(int id)   // ❌ возвращает HTTP type!
    {
        var user = _db.Users.Find(id);
        if (user == null) return new NotFoundResult();
        return new OkObjectResult(user);
    }
}

// ✅ Repository ничего не знает про HTTP
public class UserRepository
{
    public User? GetById(int id) => _db.Users.Find(id);
}

// Controller преобразует в HTTP
public IActionResult Get(int id)
{
    var user = _repo.GetById(id);
    return user == null ? NotFound() : Ok(user);
}
```

### 3.4. Папочная структура — как реализовать

```
MyApp/
├── MyApp.Web/                    ← Presentation
│   ├── Controllers/
│   │   ├── UsersController.cs
│   │   └── OrdersController.cs
│   ├── Dtos/
│   │   ├── UserDto.cs
│   │   └── OrderDto.cs
│   └── Program.cs
│
├── MyApp.Application/            ← Business Logic
│   ├── Services/
│   │   ├── IUserService.cs
│   │   ├── UserService.cs
│   │   └── OrderService.cs
│   └── Validators/
│       └── CreateUserValidator.cs
│
├── MyApp.Domain/                 ← Domain Models
│   ├── Entities/
│   │   ├── User.cs
│   │   └── Order.cs
│   └── Interfaces/
│       ├── IUserRepository.cs
│       └── IOrderRepository.cs
│
└── MyApp.Infrastructure/         ← Data Access
    ├── Repositories/
    │   ├── UserRepository.cs
    │   └── OrderRepository.cs
    └── Data/
        └── AppDbContext.cs
```

Это **Clean Architecture** в простой форме. Подробно — `Senior/architecture-decisions.md`.

> [!question]- Интервью: почему DAL не должен знать про Controllers?
> **Reverse coupling** ломает архитектуру. Если Repository возвращает `IActionResult` — он становится зависим от ASP.NET Core. Хочешь использовать тот же Repository в Console app — нельзя, тянет HTTP. Хочешь unit-тест — нужно mock'ать HTTP. **Правило**: lower layers должны быть **frameworkless** или depend на минимальные libraries. Web framework only в Presentation.

---

## 4. Dependency Injection — основа

### 4.1. Зачем DI

DI = constructor-injection через ASP.NET Core / другой контейнер. Решает проблемы:
- **Testability** — mock dependencies в тестах
- **Lifecycle management** — кто создаёт/освобождает service
- **Coupling reduction** — class зависит от interface, не concrete

```csharp
// ❌ Hard-coded dependency — нельзя протестировать
public class UserService
{
    private SqlServerUserRepository _repo = new();   // создаём сами
    private SmtpEmailService _email = new();
    
    public void Register(string name, string email)
    {
        _repo.Add(new User { Name = name });
        _email.Send(email, "Welcome");
    }
}

// Тест:
var service = new UserService();
service.Register("Alice", "a@x.com");
// ❌ Реально пишет в SQL Server и шлёт email!
```

```csharp
// ✅ DI через constructor — testable
public class UserService
{
    private readonly IUserRepository _repo;
    private readonly IEmailService _email;
    
    public UserService(IUserRepository repo, IEmailService email)
    {
        _repo = repo;
        _email = email;
    }
    
    public void Register(string name, string email)
    {
        _repo.Add(new User { Name = name });
        _email.Send(email, "Welcome");
    }
}

// Тест:
var repoMock = new Mock<IUserRepository>();
var emailMock = new Mock<IEmailService>();
var service = new UserService(repoMock.Object, emailMock.Object);
service.Register("Alice", "a@x.com");
emailMock.Verify(e => e.Send("a@x.com", "Welcome"), Times.Once);
// ✅ Никаких реальных SQL/email
```

### 4.2. Регистрация в ASP.NET Core

```csharp
// Program.cs
var builder = WebApplication.CreateBuilder(args);

// Регистрируем зависимости
builder.Services.AddScoped<IUserRepository, UserRepository>();
builder.Services.AddScoped<IUserService, UserService>();
builder.Services.AddScoped<IEmailService, SmtpEmailService>();
builder.Services.AddDbContext<AppDbContext>(opt => opt.UseSqlServer(connStr));

var app = builder.Build();
// ...
```

DI контейнер автоматически создаёт UserService с правильными dependencies.

### 4.3. Service Lifetimes

```
Singleton — один instance на всё приложение
- Stateless services (formatters, calculators)
- Caches
- Configuration

Scoped — один instance на HTTP запрос
- DbContext (стандарт)
- Repositories
- Per-request state

Transient — новый instance каждый раз когда запрашивается
- Lightweight services без state
```

```csharp
builder.Services.AddSingleton<ICache, RedisCache>();
builder.Services.AddScoped<IUserRepository, UserRepository>();
builder.Services.AddTransient<IDateProvider, SystemDateProvider>();
```

### 4.4. Не делай

```csharp
// ❌ Service Locator — anti-pattern
public class UserService
{
    public void Register(string name)
    {
        var repo = ServiceLocator.GetService<IUserRepository>();   // hidden!
        repo.Add(new User { Name = name });
    }
}

// Hidden dependency, не видно из constructor.
```

```csharp
// ❌ new вместо DI
public class UserService
{
    public void Register(string name)
    {
        var repo = new UserRepository();   // hard dependency
    }
}
```

> [!question]- Интервью: что такое DI и зачем?
> **Dependency Injection** — pattern где class **получает** свои dependencies (через constructor) **вместо** того чтобы создавать их сам. **Зачем**: 1) Testability — mock зависимости. 2) Loose coupling — class зависит от interface, не concrete class. 3) Lifecycle management — DI container создаёт/освобождает. **Practice**: ASP.NET Core имеет встроенный DI container (`Microsoft.Extensions.DependencyInjection`). 3 lifetimes: Singleton, Scoped, Transient.

---

## 5. Interfaces vs Implementations

### 5.1. Зачем interfaces

```
Interface = contract: "вот что я делаю, без подробностей как"
Implementation = concrete: "вот как именно"

Зачем разделять:
- Multiple implementations (SQL / Mongo / In-Memory)
- Testability (mock interface)
- Compile-time decoupling
- Замена implementations без правки caller
```

```csharp
// Interface
public interface IUserRepository
{
    Task<User?> GetByIdAsync(int id);
    Task<int> AddAsync(User user);
}

// Implementation 1 — SQL
public class SqlUserRepository : IUserRepository
{
    private readonly AppDbContext _db;
    public SqlUserRepository(AppDbContext db) => _db = db;
    
    public Task<User?> GetByIdAsync(int id) => _db.Users.FindAsync(id).AsTask();
    public async Task<int> AddAsync(User user)
    {
        _db.Users.Add(user);
        await _db.SaveChangesAsync();
        return user.Id;
    }
}

// Implementation 2 — In-Memory (для тестов)
public class InMemoryUserRepository : IUserRepository
{
    private readonly Dictionary<int, User> _users = new();
    private int _nextId = 1;
    
    public Task<User?> GetByIdAsync(int id) =>
        Task.FromResult(_users.GetValueOrDefault(id));
    
    public Task<int> AddAsync(User user)
    {
        user.Id = _nextId++;
        _users[user.Id] = user;
        return Task.FromResult(user.Id);
    }
}

// Caller не знает какая implementation
public class UserService
{
    private readonly IUserRepository _repo;   // interface!
    public UserService(IUserRepository repo) => _repo = repo;
}
```

### 5.2. Когда interface лишний

```
✅ Создать interface когда:
- Нужно mock'ать в тестах (most common reason)
- Будут multiple implementations
- Слой представляет abstraction (Repository, Service)
- Cross-project boundary (Domain/Infrastructure split)

❌ НЕ создавать interface когда:
- Утилитные classes (Math, StringHelpers)
- Internal classes которые никто не mock'ает
- Simple DTOs / records
- "Просто чтобы было" — over-engineering
```

```csharp
// ❌ Лишний interface для DTO
public interface IUserDto { int Id { get; } string Name { get; } }
public class UserDto : IUserDto { ... }   // никто не mock'ает DTO

// ✅ Просто DTO
public record UserDto(int Id, string Name);
```

### 5.3. Naming convention

```
Interface: I prefix
public interface IUserRepository
public interface IEmailService

Implementation: descriptive name
public class SqlUserRepository : IUserRepository
public class SmtpEmailService : IEmailService

Не делай:
public interface UserRepository      // нет I prefix — не Microsoft style
public class UserRepositoryImpl       // Java style — не C#
```

---

## 6. DTO vs Domain Model vs Database Entity

### 6.1. Три типа моделей данных

В layered architecture важно различать:

```
Database Entity:
- Маппится на таблицу БД
- Содержит navigation properties
- Может иметь EF Core attributes
- НЕ выходит за DAL

Domain Model:
- Бизнес-сущность с поведением
- Может быть = Database Entity (простой случай)
- Содержит business rules / validation

DTO (Data Transfer Object):
- Передаётся через границы (HTTP, API)
- Без поведения, только properties
- Можно изменять без breaking changes БД
```

```csharp
// Database Entity (Infrastructure)
public class UserEntity
{
    public int Id { get; set; }
    public string Name { get; set; } = "";
    public string Email { get; set; } = "";
    public string PasswordHash { get; set; } = "";
    public DateTime CreatedAt { get; set; }
    public List<OrderEntity> Orders { get; set; } = new();   // navigation
    public bool IsDeleted { get; set; }
    public string InternalNotes { get; set; } = "";
}

// Domain Model (Domain)
public class User
{
    public int Id { get; private set; }
    public string Name { get; private set; }
    public string Email { get; private set; }
    public DateTime CreatedAt { get; private set; }
    
    public User(int id, string name, string email, DateTime createdAt)
    {
        if (string.IsNullOrEmpty(name)) throw new ArgumentException("Name required");
        if (!email.Contains("@")) throw new ArgumentException("Invalid email");
        Id = id;
        Name = name;
        Email = email;
        CreatedAt = createdAt;
    }
    
    public bool IsRecentlyCreated() => (DateTime.UtcNow - CreatedAt).TotalDays < 7;
}

// DTO (Presentation)
public record UserDto(int Id, string Name, string Email);
// без PasswordHash, без InternalNotes — это утечка
```

### 6.2. Mapping между ними

```csharp
// Database → Domain
public User ToDomain(UserEntity entity) =>
    new User(entity.Id, entity.Name, entity.Email, entity.CreatedAt);

// Domain → DTO
public UserDto ToDto(User user) =>
    new UserDto(user.Id, user.Name, user.Email);

// Используй AutoMapper / Mapster если много mappings (Senior уровень)
```

### 6.3. Зачем разделять

```
Database Entity ≠ DTO:
- Email change в API не должен ломать SQL schema
- PasswordHash не должен утекать в JSON response
- Internal flags не для frontend

Database Entity ≠ Domain Model:
- Domain имеет invariants (private setters, validation)
- Database — просто данные
- Domain не зависит от EF Core attributes
```

```csharp
// ❌ Возвращаем Entity напрямую — утечка PasswordHash!
[HttpGet("{id}")]
public IActionResult Get(int id)
{
    var entity = _db.Users.Find(id);
    return Ok(entity);   // PasswordHash в JSON!
}

// ✅ Mapping → DTO
[HttpGet("{id}")]
public IActionResult Get(int id)
{
    var entity = _db.Users.Find(id);
    if (entity == null) return NotFound();
    return Ok(new UserDto(entity.Id, entity.Name, entity.Email));
}
```

> [!question]- Интервью: зачем DTO если есть Entity?
> 1) **Security** — Entity может содержать sensitive поля (PasswordHash, InternalNotes), DTO выбирает только публичные. 2) **Versioning** — изменение БД не ломает API. 3) **Aggregation** — DTO может combine несколько Entity (UserDto с Address inline). 4) **Independent evolution** — API contract меняется отдельно от DB schema. 5) **Network optimization** — DTO содержит только нужное. **Cost**: дополнительный mapping. **Tools**: AutoMapper, Mapster для compile-time mapping.

---

## 7. Folder structure — как организовать

### 7.1. Маленькое API (1-2 разработчика)

```
MyApp/
├── MyApp.csproj
├── Program.cs
├── appsettings.json
├── Controllers/
│   ├── UsersController.cs
│   └── OrdersController.cs
├── Services/
│   ├── IUserService.cs
│   ├── UserService.cs
│   └── OrderService.cs
├── Repositories/
│   ├── IUserRepository.cs
│   ├── UserRepository.cs
│   └── OrderRepository.cs
├── Models/
│   ├── User.cs
│   └── Order.cs
├── Dtos/
│   ├── UserDto.cs
│   └── OrderDto.cs
└── Data/
    └── AppDbContext.cs
```

**Один проект** — простота. Подходит для MVP / прототипа / маленьких CRUD.

### 7.2. Средний API (5+ файлов в папке)

```
MyApp/
├── MyApp.Web/                    ← API project
│   ├── Controllers/
│   ├── Dtos/
│   ├── Filters/
│   └── Program.cs
│
├── MyApp.Core/                   ← Business logic + Domain
│   ├── Models/
│   ├── Services/
│   └── Interfaces/
│
└── MyApp.Data/                   ← Data access
    ├── Repositories/
    ├── AppDbContext.cs
    └── Migrations/
```

3 проекта — clear boundaries. Web ссылается на Core. Data ссылается на Core.

### 7.3. Большой API / Clean Architecture

```
MyApp/
├── MyApp.Web/                    ← Presentation
├── MyApp.Application/            ← Use cases / Services
├── MyApp.Domain/                 ← Entities, business rules
└── MyApp.Infrastructure/         ← Data, External APIs, File I/O
```

**Domain** — независимый, не ссылается ни на кого.
**Application** ссылается на Domain.
**Infrastructure** ссылается на Application + Domain.
**Web** ссылается на Application.

См. `Senior/architecture-decisions.md`.

### 7.4. Феатура-folder (alternative)

```
MyApp/
├── Features/
│   ├── Users/
│   │   ├── UsersController.cs
│   │   ├── UserService.cs
│   │   ├── UserRepository.cs
│   │   ├── User.cs
│   │   └── UserDto.cs
│   ├── Orders/
│   │   └── ...
│   └── Products/
└── Shared/
    └── ...
```

Группировка **по фичам**, не по слоям. Хорошо когда фичи independent.

> [!question]- Интервью: Layer-folder vs Feature-folder?
> **Layer-folder** (по типу): группа Controllers, Services, Repositories отдельно. Plus: cleaner separation, conventional. Minus: добавить feature = править N folders. **Feature-folder** (по фиче): всё про Users в одном месте. Plus: easy add/remove feature, locality. Minus: harder to find общие patterns. **Best practice 2024+**: Feature-folder для серьёзных приложений, Layer-folder для small projects. **Hybrid**: Features/ + Shared/ (common DTOs, base classes).

---

## 8. Common pitfalls

### 8.1. God Class

```csharp
public class UserManager   // 50 методов!
{
    public User GetUser(int id);
    public void CreateUser(...);
    public void UpdateUser(...);
    public void DeleteUser(...);
    public void SendUserEmail(...);
    public void GenerateUserReport(...);
    public void ValidateUserInput(...);
    public void LogUserAction(...);
    public void EncryptUserData(...);
    public void ExportUserCsv(...);
    // ... ещё 40 методов
}
```

**Фикс**: split на UserService, EmailService, ReportGenerator, Validator, Logger, EncryptionService, CsvExporter — каждый со своей ответственностью.

### 8.2. Anaemic models

```csharp
// ❌ Только properties, никакой логики
public class Order
{
    public int Id { get; set; }
    public OrderStatus Status { get; set; }
    public decimal Total { get; set; }
}

public class OrderService   // ВСЯ логика здесь
{
    public void Pay(Order order, decimal amount)
    {
        if (order.Status != OrderStatus.Pending) throw ...;
        order.Status = OrderStatus.Paid;
        order.Total = amount;
    }
}
```

**Фикс**: rich domain — Order.Pay() сам enforces invariants.

```csharp
public class Order
{
    public int Id { get; private set; }
    public OrderStatus Status { get; private set; }
    public decimal Total { get; private set; }
    
    public void Pay(decimal amount)
    {
        if (Status != OrderStatus.Pending) throw new InvalidOperationException();
        Status = OrderStatus.Paid;
        Total = amount;
    }
}
```

### 8.3. Mixing layers

```csharp
// ❌ Repository выполняет business logic
public class UserRepository
{
    public async Task<int> AddAsync(User user)
    {
        if (string.IsNullOrEmpty(user.Email))   // ❌ business rule!
            throw new ArgumentException();
        if (!user.Email.Contains("@"))
            throw new ArgumentException();
        
        _db.Users.Add(user);
        await _db.SaveChangesAsync();
        return user.Id;
    }
}
```

**Фикс**: validation в Service layer, Repository — только сохранение.

### 8.4. Circular dependencies

```csharp
public class UserService
{
    private readonly OrderService _orders;
    public UserService(OrderService orders) => _orders = orders;
}

public class OrderService
{
    private readonly UserService _users;   // ❌ circular!
    public OrderService(UserService users) => _users = users;
}
```

**Фикс**: extract shared logic в третий service, или используй events.

### 8.5. Returning Entity from API

```csharp
[HttpGet("{id}")]
public IActionResult Get(int id)
{
    var user = _db.Users.Find(id);
    return Ok(user);   // ❌ leaks PasswordHash, InternalNotes
}
```

**Фикс**: mapping → DTO.

### 8.6. Using `Repository<T>` for everything

```csharp
public interface IRepository<T>
{
    T GetById(int id);
    void Add(T entity);
    void Delete(int id);
}

public class UserRepository : IRepository<User> { }
public class OrderRepository : IRepository<Order> { }
public class ProductRepository : IRepository<Product> { }
// ...50 ещё
```

**Фикс**: EF Core IS the repository через DbContext. Generic Repository — over-abstraction. Specific repositories только для специальных queries.

### 8.7. Big controller methods

```csharp
[HttpPost]
public async Task<IActionResult> CreateOrder([FromBody] CreateOrderRequest request)
{
    // Validation
    if (request.Items.Count == 0) return BadRequest("Items required");
    
    // Business logic
    var total = 0m;
    foreach (var item in request.Items)
    {
        var product = await _db.Products.FindAsync(item.ProductId);
        if (product == null) return NotFound();
        total += product.Price * item.Quantity;
    }
    
    // Tax
    total *= 1.08m;
    
    // Save
    var order = new Order { Total = total, Status = OrderStatus.Pending };
    _db.Orders.Add(order);
    await _db.SaveChangesAsync();
    
    // Email
    await _emailService.SendAsync(...);
    
    // ... 50 more lines
    
    return Ok(order);
}
```

**Фикс**: Controller — thin, всё в Service:

```csharp
[HttpPost]
public async Task<IActionResult> CreateOrder([FromBody] CreateOrderRequest request)
{
    var orderId = await _orderService.PlaceOrderAsync(request);
    return CreatedAtAction(nameof(Get), new { id = orderId }, null);
}
```

> [!question]- Интервью: топ-3 architectural mistakes у джуниоров?
> 1) **God class** — один класс с 30+ методами разной природы. Fix: split по SoC. 2) **Mixing layers** — Repository выполняет business logic, Controller знает про SQL. Fix: каждый слой делает своё. 3) **Returning Entity from API** — утечка sensitive полей. Fix: DTO. **Bonus**: not using DI — hard-coded `new SqlConnection()` в каждом классе.

---

## 9. Decision tree

```
Что выбрать?
│
├── Размер проекта
│   ├── pet project < 500 строк → один файл OK
│   ├── small (1-2 dev) → один project, layer folders
│   ├── medium (5+ dev) → 3 projects (Web/Core/Data)
│   └── large (10+ dev) → Clean Architecture (4 projects)
│
├── Type разделения
│   ├── Layer folders → conventional, simple
│   ├── Feature folders → for independent features
│   └── Hybrid → Features/ + Shared/
│
├── Domain models
│   ├── Simple CRUD → Entity == Domain (записать в БД, отдать наружу)
│   ├── Business logic → Rich Domain (private setters + methods)
│   └── Public API → DTO (mapping от Domain)
│
├── Interface необходим?
│   ├── Mock в тестах → ДА
│   ├── Multiple implementations → ДА
│   ├── Cross-project boundary → ДА
│   ├── Internal utility class → НЕТ
│   └── Just because → НЕТ (over-engineering)
│
├── Repository pattern?
│   ├── Specific queries (ComplexProductSearch) → ДА, специфичный repo
│   ├── Just CRUD → EF Core DbContext IS the repo
│   └── Generic Repository<T> → НЕТ (over-abstraction)
│
└── Где business logic?
    ├── В Domain entities (Rich Domain) — preferred
    ├── В Services (orchestration) — OK
    ├── В Controllers — НЕТ (thin controllers)
    └── В Repository — НЕТ (mixing layers)
```

---

## 10. Cheat sheet

```csharp
// === Layered structure ===
// Presentation: Controllers, DTOs
// Business: Services, Domain Models
// Data: Repositories, DbContext

// === DI registration (ASP.NET Core) ===
builder.Services.AddSingleton<ICache, RedisCache>();          // 1 instance app
builder.Services.AddScoped<IUserRepository, UserRepository>(); // 1 per request
builder.Services.AddTransient<IDateProvider, SystemDateProvider>(); // new each time

// === Constructor injection ===
public class UserService
{
    private readonly IUserRepository _repo;
    private readonly ILogger<UserService> _logger;
    
    public UserService(IUserRepository repo, ILogger<UserService> logger)
    {
        _repo = repo;
        _logger = logger;
    }
}

// === Interface naming ===
public interface IUserRepository { }   // I prefix
public class SqlUserRepository : IUserRepository { }   // descriptive

// === DTO ===
public record UserDto(int Id, string Name, string Email);

// === Entity → Domain → DTO mapping ===
var entity = await _db.Users.FindAsync(id);
var domain = new User(entity.Id, entity.Name, entity.Email, entity.CreatedAt);
var dto = new UserDto(domain.Id, domain.Name, domain.Email);

// === Controller — thin ===
[HttpGet("{id}")]
public async Task<IActionResult> Get(int id)
{
    var user = await _userService.GetByIdAsync(id);
    return user == null ? NotFound() : Ok(new UserDto(user.Id, user.Name, user.Email));
}

// === Service — orchestration ===
public async Task<int> CreateAsync(string name, string email)
{
    if (string.IsNullOrEmpty(name)) throw new ArgumentException();
    var user = new User(0, name, email, DateTime.UtcNow);
    return await _repo.AddAsync(user);
}

// === Repository — data access ===
public Task<User?> GetByIdAsync(int id) => _db.Users.FindAsync(id).AsTask();
```

---

## 11. Practice exercises

### 11.1. Refactor God Class

Перепиши:

```csharp
public class UserManager
{
    public User GetUser(int id)
    {
        var conn = new SqlConnection("...");
        conn.Open();
        var cmd = new SqlCommand("SELECT * FROM Users WHERE Id = @id", conn);
        cmd.Parameters.AddWithValue("@id", id);
        var reader = cmd.ExecuteReader();
        if (!reader.Read()) return null;
        return new User { Id = reader.GetInt32(0), Name = reader.GetString(1) };
    }
    
    public void CreateUser(string name, string email)
    {
        if (string.IsNullOrEmpty(name)) throw new Exception();
        if (!email.Contains("@")) throw new Exception();
        
        var conn = new SqlConnection("...");
        conn.Open();
        var cmd = new SqlCommand("INSERT INTO Users (Name, Email) VALUES (@n, @e)", conn);
        cmd.Parameters.AddWithValue("@n", name);
        cmd.Parameters.AddWithValue("@e", email);
        cmd.ExecuteNonQuery();
        
        var smtp = new SmtpClient("smtp.example.com");
        smtp.Send(new MailMessage("from@x.com", email, "Welcome", "Hi!"));
    }
}
```

В:
- IUserRepository (data access)
- IEmailService (email)
- UserService (orchestration + validation)
- UsersController (HTTP)

### 11.2. Add new feature properly

В существующем 3-tier приложении нужна feature "Reset Password":
1. Где определишь интерфейс? (`IUserService.ResetPasswordAsync(int userId)`)
2. Где будет business logic? (`UserService.ResetPasswordAsync` — generate token, save, send email)
3. Где будет DB операция? (`IUserRepository.UpdatePasswordHashAsync`)
4. Где будет HTTP endpoint? (`UsersController` POST `/api/users/{id}/reset-password`)
5. Какой DTO? (`ResetPasswordRequestDto { string NewPassword }`)

Напиши код всех слоёв.

### 11.3. Identify violations

```csharp
public class OrdersController : ControllerBase
{
    private readonly AppDbContext _db;   // ❌ DbContext в Controller
    
    public OrdersController(AppDbContext db) => _db = db;
    
    [HttpPost]
    public async Task<IActionResult> CreateOrder(int userId, List<OrderItemRequest> items)
    {
        // ❌ Business logic в Controller
        if (items.Count == 0) return BadRequest();
        
        decimal total = 0;
        foreach (var item in items)
        {
            // ❌ EF Core query в Controller
            var product = await _db.Products.FindAsync(item.ProductId);
            total += product.Price * item.Quantity;
        }
        
        // ❌ Tax calculation в Controller
        total *= 1.08m;
        
        // ❌ Direct EF в Controller
        var order = new Order { UserId = userId, Total = total };
        _db.Orders.Add(order);
        await _db.SaveChangesAsync();
        
        // ❌ Возвращаем Entity (может содержать sensitive поля)
        return Ok(order);
    }
}
```

Найди 5 нарушений архитектуры. Перепиши правильно.

---

## 12. Что читать дальше

1. **`Architecture/Middle/patterns-decision-guide.md`** — patterns в depth
2. **`Architecture/Middle/microservices-vs-monolith.md`** — когда что
3. **`Architecture/Senior/solid.md`** — SOLID principles
4. **`Architecture/Senior/ddd.md`** — Domain-Driven Design
5. **`Architecture/Senior/architecture-decisions.md`** — Clean Architecture
6. **Robert Martin — "Clean Architecture"** (book) — фундамент

---

## 13. См. также

- [[architecture-patterns|Architecture Patterns]] — design patterns
- [[ddd|Architecture/Senior/ddd]] — DDD
- [[solid|Architecture/Senior/solid]] — SOLID principles
- [[design-patterns|CSharp/Senior/design-patterns]] — GoF patterns
- [[oop|CSharp/Junior/oop]] — OOP basics
- [[ef-basics|EFCore/Junior/ef-basics]] — EF Core entry

---

## 14. Reading list

- **Robert Martin — "Clean Architecture"** (2017)
- **Eric Evans — "Domain-Driven Design"** (2003)
- **Martin Fowler — "Patterns of Enterprise Application Architecture"** (2002)
- **"Dependency Injection in .NET"** by Mark Seemann
- **Microsoft Docs — Clean Architecture** — learn.microsoft.com/dotnet/architecture/modern-web-apps-azure/common-web-application-architectures
- **Microsoft Docs — Layered Architecture** — learn.microsoft.com/dotnet/architecture/microservices/microservice-ddd-cqrs-patterns/microservice-application-layer-implementation-web-api
