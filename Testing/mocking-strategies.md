---
tags: [testing, mocking, nsubstitute, moq, fakes, stubs, test-doubles]
level: Middle to Senior
date: 2026-04-30
---

# Mocking Strategies — стратегии моков

> **Когда мокать, как мокать, а главное — когда НЕ мокать**. Test doubles разных видов, NSubstitute vs Moq, anti-patterns "over-mocking", patterns для DI / EF Core / HTTP.

---

## Что это, зачем и когда

### Что такое mock?

**Замена настоящей зависимости** в тесте на упрощённую версию.

```csharp
// Без мока — реальный API call в тесте
var realApi = new RealPaymentApi();
var service = new OrderService(realApi);
service.PlaceOrder(order);  // ⚠️ Реально оплатит!

// С моком — replace API
var mockApi = Substitute.For<IPaymentApi>();
mockApi.Charge(Arg.Any<decimal>()).Returns(true);

var service = new OrderService(mockApi);
service.PlaceOrder(order);  // ✅ Безопасно
```

### Зачем мокать

| Без mocks | С mocks |
|-----------|---------|
| Тест требует реальный API / DB / file | Изолированный |
| Slow (network calls) | Fast |
| Flaky (network down → false fail) | Deterministic |
| Coupled — meняется external API → тест fail | Stable |
| Side effects (отправка email!) | No effects |

### Когда **НЕ** мокать

❌ Не мокай:
- **Value objects** — просто создай реальный
- **Pure functions** — нет зависимостей
- **Что владеешь** + **просто** — реальный
- **Sealed types** — не получится
- **Concrete implementations 3rd-party** — мокай интерфейс

---

## 1. Виды test doubles

### Stub — возвращает hardcoded

```csharp
public class StubUserRepo : IUserRepo
{
    public User? GetById(int id) =>
        id == 1 ? new User { Name = "Test" } : null;
}

// Использование
var repo = new StubUserRepo();
var service = new UserService(repo);
service.GetUser(1);  // вернёт User
service.GetUser(2);  // null
```

**Когда:** простые случаи, "just enough".

### Mock — verify взаимодействия

```csharp
var mockRepo = Substitute.For<IUserRepo>();
mockRepo.GetById(1).Returns(new User { Name = "Test" });

var service = new UserService(mockRepo);
service.DoSomething(1);

// Проверяем что метод был вызван
mockRepo.Received(1).GetById(1);

// Проверяем что НЕ было вызвано
mockRepo.DidNotReceive().Save(Arg.Any<User>());
```

**Когда:** verify side effect (был отправлен email, был сохранён audit log).

### Fake — рабочая альтернатива

```csharp
// In-memory implementation интерфейса
public class InMemoryUserRepo : IUserRepo
{
    private readonly Dictionary<int, User> _users = new();
    
    public User? GetById(int id) => 
        _users.TryGetValue(id, out var u) ? u : null;
    
    public void Save(User user) => _users[user.Id] = user;
    
    public IEnumerable<User> GetAll() => _users.Values;
}

// Использование
var repo = new InMemoryUserRepo();
repo.Save(new User { Id = 1, Name = "Test" });

var service = new UserService(repo);
var user = service.GetUser(1);
```

**Когда:** сложная заглушка нужна (in-memory DB, in-memory cache).

### Spy — record вызовов

```csharp
public class SpyEmailService : IEmailService
{
    public List<(string To, string Body)> SentEmails { get; } = new();
    
    public Task SendAsync(string to, string body)
    {
        SentEmails.Add((to, body));
        return Task.CompletedTask;
    }
}

[Fact]
public void Test()
{
    var spy = new SpyEmailService();
    var service = new MyService(spy);
    
    service.NotifyUser(1);
    
    spy.SentEmails.Should().ContainSingle()
        .Which.To.Should().Be("user1@example.com");
}
```

**Когда:** middle ground между mock (verify) и fake (alternative impl).

### Dummy — placeholder

Объект который **передаётся но не используется**.

```csharp
public void Method(IDep1 dep1, IDep2 dep2)
{
    dep1.Do();  // dep2 не используется но требуется по сигнатуре
}

// Тест
var realDep1 = new Dep1Impl();
var dummyDep2 = Substitute.For<IDep2>();  // никогда не вызывается

new Service(realDep1, dummyDep2).Method();
```

---

## 2. Mocking libraries — сравнение

### NSubstitute (recommended 2026)

```csharp
var repo = Substitute.For<IUserRepo>();

// Setup
repo.GetById(1).Returns(new User { Name = "Alice" });
repo.GetById(Arg.Any<int>()).Returns(new User { Name = "Default" });

// Throw
repo.GetById(0).Returns(_ => throw new ArgumentException());

// Async
repo.GetByIdAsync(1).Returns(Task.FromResult(new User()));

// Verify
repo.Received(1).GetById(1);
repo.Received().GetById(Arg.Is<int>(x => x > 0));
repo.DidNotReceive().Delete(Arg.Any<int>());

// Match by predicate
repo.GetByIdAsync(Arg.Is<int>(x => x > 0)).Returns(...);
```

✅ **Pros:**
- Чистый syntax (нет `.Setup(x => x...)` boilerplate)
- Легко читать
- Active development

### Moq (классика)

```csharp
var repo = new Mock<IUserRepo>();

repo.Setup(r => r.GetById(1)).Returns(new User { Name = "Alice" });
repo.Setup(r => r.GetById(It.IsAny<int>())).Returns(new User());

repo.Verify(r => r.GetById(1), Times.Once());
repo.Verify(r => r.Delete(It.IsAny<int>()), Times.Never());

var service = new UserService(repo.Object);  // .Object нужен!
```

⚠️ **Cons:**
- Verbose (`Setup`, `It.IsAny<>`)
- `Mock.Object` для access — boilerplate
- **Moq 4.20** controversy — analytics scandal

### FakeItEasy

```csharp
var repo = A.Fake<IUserRepo>();
A.CallTo(() => repo.GetById(1)).Returns(new User());
A.CallTo(() => repo.GetById(1)).MustHaveHappened();
```

Меньше популярна, но работает похоже на NSubstitute.

### Выбор

```
Новый проект 2026:    NSubstitute ✅
Existing с Moq:       Stay with Moq, не migrate без причины
Functional preference: FakeItEasy
```

---

## 3. Patterns для DI

### Constructor injection — лучший для testability

```csharp
public class OrderService
{
    private readonly IOrderRepo _repo;
    private readonly IPaymentApi _payment;
    private readonly ILogger<OrderService> _logger;
    
    public OrderService(IOrderRepo repo, IPaymentApi payment, ILogger<OrderService> logger)
    {
        _repo = repo;
        _payment = payment;
        _logger = logger;
    }
    
    public async Task PlaceOrder(Order order) { ... }
}

// Test
var repo = Substitute.For<IOrderRepo>();
var payment = Substitute.For<IPaymentApi>();
var logger = NullLogger<OrderService>.Instance;

var service = new OrderService(repo, payment, logger);
```

### Service locator — testability ад

```csharp
// ❌ Service locator — невозможно мокать
public class OrderService
{
    public async Task PlaceOrder(Order order)
    {
        var repo = ServiceLocator.GetService<IOrderRepo>();  // hidden dep!
        var payment = ServiceLocator.GetService<IPaymentApi>();
        // ...
    }
}
```

**Лечение:** только constructor injection.

### Static methods — нельзя мокать

```csharp
// ❌ DateTime.UtcNow в коде — нельзя мокать
public bool IsExpired(Order order) =>
    DateTime.UtcNow > order.ExpiresAt;

// ✅ Через TimeProvider (.NET 8+)
public class OrderService(TimeProvider time)
{
    public bool IsExpired(Order order) =>
        time.GetUtcNow() > order.ExpiresAt;
}

// Test
var time = new FakeTimeProvider();
time.SetUtcNow(DateTimeOffset.Parse("2024-06-01"));
var service = new OrderService(time);
```

См. [[../CSharp/csharp-language-design|C# Language Design]] — TimeProvider.

---

## 4. Mocking EF Core — DON'T

### Anti-pattern: мокать DbContext

```csharp
// ❌ Не мокай DbContext — boilerplate, fragile, реально не тестирует ничего
var mockDbSet = Substitute.For<DbSet<User>>();
var mockContext = Substitute.For<AppDbContext>();
mockContext.Users.Returns(mockDbSet);

// 50+ lines сettings, и тест не проверяет реальное EF поведение
```

### ✅ Решения

#### 1. Repository pattern + интерфейс

```csharp
public interface IUserRepo
{
    Task<User?> GetByIdAsync(int id);
    Task SaveAsync(User user);
}

public class UserRepo(AppDbContext db) : IUserRepo
{
    public Task<User?> GetByIdAsync(int id) => db.Users.FindAsync(id).AsTask();
    public Task SaveAsync(User user) { db.Users.Add(user); return db.SaveChangesAsync(); }
}

// Test моcает интерфейс — ✅ просто
var repo = Substitute.For<IUserRepo>();
repo.GetByIdAsync(1).Returns(new User());
```

#### 2. Integration tests с real DB

См. [[integration-testing|Integration Testing]].

```csharp
// Через Testcontainers — реальный Postgres в test
var factory = new TestWebFactory();
var client = factory.CreateClient();
// real EF behavior tested
```

#### 3. InMemory provider (limited)

```csharp
var options = new DbContextOptionsBuilder<AppDbContext>()
    .UseInMemoryDatabase("test")
    .Options;
var db = new AppDbContext(options);

// Тестировать
db.Users.Add(new User());
await db.SaveChangesAsync();
```

⚠️ InMemory **не emulates** SQL constraints, transactions, raw SQL. Use только для simple cases.

---

## 5. Mocking HttpClient

```csharp
// HttpClient — concrete class, мокать через HttpMessageHandler
public class MockHttpHandler : HttpMessageHandler
{
    private readonly HttpResponseMessage _response;
    
    public MockHttpHandler(HttpStatusCode status, string body)
    {
        _response = new HttpResponseMessage(status)
        {
            Content = new StringContent(body)
        };
    }
    
    protected override Task<HttpResponseMessage> SendAsync(
        HttpRequestMessage request, CancellationToken ct) =>
        Task.FromResult(_response);
}

// Test
var handler = new MockHttpHandler(HttpStatusCode.OK, """{"id":1}""");
var httpClient = new HttpClient(handler);
var service = new MyApiClient(httpClient);
var result = await service.GetData();
```

### Better — WireMock.NET

См. [[integration-testing#wiremock|Integration Testing — WireMock]].

```csharp
var mock = WireMockServer.Start();
mock.Given(Request.Create().WithPath("/api/users").UsingGet())
    .RespondWith(Response.Create().WithBody("""[{"id":1}]"""));

var client = new HttpClient { BaseAddress = new Uri(mock.Url) };
var service = new MyApiClient(client);
```

---

## 6. Mocking ILogger

```csharp
// ✅ NullLogger для тестов — самое простое
var logger = NullLogger<MyService>.Instance;

var service = new MyService(logger);
```

### Если важно verify logging

```csharp
// NSubstitute с extension method (LoggerExtensions are static)
var logger = Substitute.For<ILogger<MyService>>();

// Verify через captured calls — boilerplate
logger.Received().Log(
    LogLevel.Error,
    Arg.Any<EventId>(),
    Arg.Any<object>(),
    Arg.Any<Exception>(),
    Arg.Any<Func<object, Exception?, string>>()
);
```

⚠️ Сложно — `ILogger.Log` принимает params неудобные. Лучше:

```csharp
// Используй MELT (Microsoft Extensions Logging Testing)
dotnet add package MELT
dotnet add package MELT.Xunit

[Fact]
public void Logs_warning()
{
    var loggerFactory = TestLoggerFactory.Create();
    var service = new MyService(loggerFactory.CreateLogger<MyService>());
    
    service.DoSomething();
    
    loggerFactory.Sink.LogEntries.Should()
        .Contain(e => e.LogLevel == LogLevel.Warning);
}
```

---

## 7. Mocking time

### TimeProvider (.NET 8+) — preferred

```csharp
public class OrderService(TimeProvider time)
{
    public bool IsExpired(Order order) => time.GetUtcNow() > order.ExpiresAt;
}

// Test
var fakeTime = new FakeTimeProvider();
fakeTime.SetUtcNow(DateTimeOffset.Parse("2024-06-01"));

var service = new OrderService(fakeTime);
service.IsExpired(order).Should().BeTrue();

// Advance
fakeTime.Advance(TimeSpan.FromMinutes(5));
```

### Custom interface (старый подход)

```csharp
public interface IClock
{
    DateTime UtcNow { get; }
}

public class SystemClock : IClock
{
    public DateTime UtcNow => DateTime.UtcNow;
}

public class FakeClock : IClock
{
    public DateTime UtcNow { get; set; } = DateTime.UtcNow;
}
```

---

## 8. Verify behavior, не implementation

### ❌ Тест проверяет implementation

```csharp
[Fact]
public void Test()
{
    var repo = Substitute.For<IRepo>();
    var service = new Service(repo);
    
    service.Do();
    
    // Тестируем что внутри сделал:
    Received.InOrder(() =>
    {
        repo.BeginTransaction();
        repo.Save(Arg.Any<Entity>());
        repo.Commit();
    });
}
```

При refactoring (другой порядок, async) — test ломается, behavior тот же.

### ✅ Тест проверяет behavior

```csharp
[Fact]
public async Task Service_saves_entity_to_database()
{
    // Real или fake repo
    var repo = new InMemoryRepo();
    var service = new Service(repo);
    
    await service.Do(entity);
    
    var saved = await repo.GetByIdAsync(entity.Id);
    saved.Should().NotBeNull();
    saved.Status.Should().Be("processed");
}
```

---

## 9. Common Pitfalls

### 1. Over-mocking

```csharp
// ❌ Каждая зависимость мокается, тест ничего не проверяет
[Fact]
public void Test()
{
    var dep1 = Substitute.For<IDep1>();
    var dep2 = Substitute.For<IDep2>();
    var dep3 = Substitute.For<IDep3>();
    var dep4 = Substitute.For<IDep4>();
    var dep5 = Substitute.For<IDep5>();
    
    var service = new Service(dep1, dep2, dep3, dep4, dep5);
    service.Do();
    
    // 5 моков verified — что мы реально протестировали?
}
```

**Признаки:**
- 5+ моков на один тест
- Моки моков (mock returns mock)
- Setups занимают 50% теста

**Лечение:**
- Разбей класс — слишком много responsibilities (SOLID)
- Integration test может быть лучше

### 2. Mock того что владеешь и просто

```csharp
// ❌ Зачем мокать Result?
var result = Substitute.For<Result<int>>();
result.IsSuccess.Returns(true);

// ✅ Создай реальный
var result = Result.Ok(42);
```

Правило: **мокай только то что не своё или сложное**.

### 3. Tightly coupled mocks

```csharp
// ❌ Тест "знает" implementation:
mock.Method().Returns(x);
mock.Method().Returns(y);  // вторая call возвращает y

service.Do();
mock.Received(2).Method();  // должен вызвать дважды!

// При refactoring — fragile.
```

### 4. Async без await

```csharp
// ❌ NSubstitute с async confusing
mock.GetAsync().Returns(new User());  // синхронно?

// ✅ Returns Task<T>
mock.GetAsync().Returns(Task.FromResult(new User()));
```

### 5. Forget reset mocks между tests

Если mocks shared (через xUnit fixture) — verify counts накапливаются.

```csharp
// ✅ Mock per test
[Fact]
public void Test1()
{
    var mock = Substitute.For<IRepo>();
    // ...
}

// Or reset
mock.ClearReceivedCalls();
```

### 6. Test theatre с моками

```csharp
[Fact]
public void Test()
{
    var mock = Substitute.For<IService>();
    mock.Process(1).Returns(42);
    
    var result = mock.Process(1);  // ⚠️ Тестируем мок, не реальный код!
    
    result.Should().Be(42);
}
```

Useless. Тест проверяет что мок настроен правильно.

---

## 10. When NOT to use mocks

### Pure functions

```csharp
// ❌ Зачем мокать pure function?
var calc = Substitute.For<ICalculator>();
calc.Add(2, 3).Returns(5);

// ✅ Реальный
var calc = new Calculator();
calc.Add(2, 3).Should().Be(5);
```

### Value objects / records

```csharp
// ❌
var address = Substitute.For<Address>();
address.City.Returns("NYC");

// ✅
var address = new Address("Main", "NYC", "US");
```

### Simple POCOs

```csharp
// ❌
var user = Substitute.For<User>();

// ✅
var user = new User { Name = "Test" };
```

### Что лучше protect через integration tests

API endpoints, EF queries, message handlers — лучше integration tests с реальной БД.

---

## 11. Best Practices

- **Constructor injection** — для testability
- **Mock interfaces, not classes** — concrete classes сложнее мокать
- **NSubstitute** для нового кода
- **NullLogger\<T\>.Instance** для логирования
- **TimeProvider** (.NET 8+) для time
- **Не мокай value objects / records / pure functions**
- **Не мокай DbContext** — repository pattern или integration tests
- **Не мокай HttpClient** напрямую — WireMock.NET
- **Verify behavior, не implementation** — тесты выживают при refactoring
- **Один thing tested per test** — не пытайся проверить 10 verify в одном test
- **Reset mocks между tests** или per-test instances
- **Если 5+ mocks** — рефактор production code (SRP violation)

---

## 12. Cheat sheet — что мокать

| Зависимость | Подход |
|-------------|--------|
| `IUserRepo` interface | NSubstitute mock |
| `DbContext` | НЕ мокать — repository pattern |
| `HttpClient` | WireMock.NET или HttpMessageHandler |
| `ILogger<T>` | NullLogger\<T\>.Instance |
| `TimeProvider` | FakeTimeProvider (.NET 8+) |
| `IConfiguration` | Real `ConfigurationBuilder` с in-memory |
| `IMediator` | NSubstitute mock |
| Pure function class | Реальный |
| Value object / record | Реальный |
| Static method | Wrap в interface, потом mock |
| External API | Mock interface или WireMock.NET |
| File system | `IFileSystem` (System.IO.Abstractions) |
| `IOptions<T>` | `Options.Create(new T())` |

---

## См. также

- [[testing-fundamentals|Testing Fundamentals]] — основы тестирования
- [[integration-testing|Integration Testing]] — когда integration > unit + mocks
- [[mutation-load-testing|Mutation Testing]] — проверка качества тестов
- [[testing|Testing — practical xUnit]]
- [[../Quality/clean-code|Clean Code]] — testable design

## Reading list

- **NSubstitute Documentation** — nsubstitute.github.io
- **Moq Documentation** — github.com/moq/moq
- **Unit Testing Principles, Practices, and Patterns** — Vladimir Khorikov (modern guide)
- **The Art of Unit Testing** — Roy Osherove
- **Microsoft Docs — Test isolation** — learn.microsoft.com
- **Mark Seemann blog** — blog.ploeh.dk (functional thinking on mocking)
- **TestContainers** docs — testcontainers.com
