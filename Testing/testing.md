---
tags: [testing, xunit, testcontainers, integration-tests, mocking]
level: Senior
---

# Тестирование: стратегия и инструменты

> [!question]- **Интервью: Пирамида vs Алмаз тестирования?**
> **Пирамида** — много unit, меньше integration, мало E2E. **Алмаз (diamond)** — для backend: больше integration тестов (WebApplicationFactory + Testcontainers), меньше unit. Причина: backend — это интеграция (DB, HTTP, message bus). Integration тесты ловят реальные баги.

> [!question]- **Интервью: Тестирование с БД и внешними API?**
> **БД:** Testcontainers (real PostgreSQL/SQL Server в Docker). Respawn — очистка между тестами. **Внешние API:** WireMock, HttpMessageHandler mock. **Не мокай БД** — мок не ловит SQL-ошибки.

## Стек

| Инструмент | Назначение | NuGet |
|------------|------------|-------|
| **xUnit** | Тестовый фреймворк | `xunit` |
| **NSubstitute** | Моки зависимостей | `NSubstitute` |
| **FluentAssertions** | Читаемые assertions | `FluentAssertions` |
| **Testcontainers** | Docker-контейнеры для интеграционных тестов | `Testcontainers.PostgreSql` |
| **WebApplicationFactory** | In-memory ASP.NET pipeline | `Microsoft.AspNetCore.Mvc.Testing` |
| **Bogus** | Генерация фейковых данных | `Bogus` |
| **Respawn** | Сброс состояния БД между тестами | `Respawn` |

---

## Unit тесты

Тестируют один класс в изоляции. Зависимости — моки. Быстрые, без I/O. Arrange — Act — Assert.

### Базовый пример

```csharp
public class OrderServiceTests
{
    private readonly IOrderRepository _repo = Substitute.For<IOrderRepository>();
    private readonly ILogger<OrderService> _logger = Substitute.For<ILogger<OrderService>>();
    private readonly OrderService _sut; // System Under Test

    public OrderServiceTests() => _sut = new OrderService(_repo, _logger);

    [Fact]
    public async Task GetOrder_WhenExists_ReturnsOrder()
    {
        // Arrange
        var expected = new Order { Id = Guid.NewGuid(), Total = 100m };
        _repo.GetByIdAsync(expected.Id, Arg.Any<CancellationToken>())
            .Returns(expected);

        // Act
        var result = await _sut.GetAsync(expected.Id);

        // Assert
        result.Should().NotBeNull();
        result!.Total.Should().Be(100m);
    }

    [Fact]
    public async Task GetOrder_WhenNotFound_ReturnsNull()
    {
        _repo.GetByIdAsync(Arg.Any<Guid>(), Arg.Any<CancellationToken>())
            .Returns((Order?)null);

        var result = await _sut.GetAsync(Guid.NewGuid());

        result.Should().BeNull();
    }
}
```

### Theory — параметризованные тесты

```csharp
[Theory]
[InlineData(0, false)]
[InlineData(-1, false)]
[InlineData(100, true)]
[InlineData(10000.01, false)] // превышает лимит
public async Task CreateOrder_ValidatesTotal(decimal total, bool expectedSuccess)
{
    var result = await _sut.CreateAsync(new CreateOrderDto("Customer", total));
    result.IsSuccess.Should().Be(expectedSuccess);
}

// ClassData — для сложных данных
[Theory]
[ClassData(typeof(InvalidOrderTestData))]
public async Task CreateOrder_InvalidData_ReturnsError(CreateOrderDto dto, string expectedError)
{
    var result = await _sut.CreateAsync(dto);
    result.Error.Should().Contain(expectedError);
}
```

### NSubstitute — нюансы

```csharp
// Проверка вызова с конкретными аргументами
await _repo.Received(1).AddAsync(
    Arg.Is<Order>(o => o.CustomerId == customerId && o.Total == 100m),
    Arg.Any<CancellationToken>());

// Проверка что метод НЕ вызывался
await _repo.DidNotReceive().DeleteAsync(Arg.Any<Guid>(), Arg.Any<CancellationToken>());

// Бросить исключение
_repo.GetByIdAsync(Arg.Any<Guid>(), Arg.Any<CancellationToken>())
    .ThrowsAsync(new TimeoutException("DB timeout"));

// Последовательные вызовы
_repo.GetByIdAsync(Arg.Any<Guid>(), Arg.Any<CancellationToken>())
    .Returns(
        x => null,          // первый вызов
        x => new Order());  // второй вызов
```

**Нюанс:** NSubstitute vs Moq — NSubstitute проще синтаксис, но может скрывать ошибки (silent auto-mocking). Moq более явный. Для новых проектов — NSubstitute.

---

## Integration тесты

Тестируют несколько компонентов вместе. Реальная БД, HTTP pipeline. Ловят проблемы, невидимые в unit-тестах.

### WebApplicationFactory — полный HTTP pipeline

```csharp
public class OrdersApiTests : IClassFixture<CustomWebApplicationFactory>
{
    private readonly HttpClient _client;

    public OrdersApiTests(CustomWebApplicationFactory factory)
    {
        _client = factory.CreateClient();
    }

    [Fact]
    public async Task CreateOrder_ValidRequest_Returns201()
    {
        var dto = new CreateOrderRequest("Customer", [new("Product1", 2, 50m)]);

        var response = await _client.PostAsJsonAsync("/api/orders", dto);

        response.StatusCode.Should().Be(HttpStatusCode.Created);
        var created = await response.Content.ReadFromJsonAsync<OrderResponse>();
        created!.Id.Should().NotBeEmpty();
    }

    [Fact]
    public async Task CreateOrder_InvalidRequest_Returns422()
    {
        var dto = new CreateOrderRequest("", []); // пустой customer, нет items

        var response = await _client.PostAsJsonAsync("/api/orders", dto);

        response.StatusCode.Should().Be(HttpStatusCode.UnprocessableEntity);
    }
}
```

### Custom WebApplicationFactory

```csharp
public class CustomWebApplicationFactory : WebApplicationFactory<Program>, IAsyncLifetime
{
    private readonly PostgreSqlContainer _db = new PostgreSqlBuilder()
        .WithImage("postgres:16")
        .Build();

    protected override void ConfigureWebHost(IWebHostBuilder builder)
    {
        builder.ConfigureServices(services =>
        {
            // Заменяем БД на Testcontainers
            services.RemoveAll<DbContextOptions<AppDbContext>>();
            services.AddDbContext<AppDbContext>(opts =>
                opts.UseNpgsql(_db.GetConnectionString()));
        });

        builder.UseEnvironment("Testing");
    }

    public async Task InitializeAsync()
    {
        await _db.StartAsync();
    }

    public new async Task DisposeAsync()
    {
        await _db.DisposeAsync();
        await base.DisposeAsync();
    }
}
```

### Testcontainers — реальная БД в Docker

```csharp
// PostgreSQL
private readonly PostgreSqlContainer _postgres = new PostgreSqlBuilder()
    .WithImage("postgres:16")
    .WithDatabase("testdb")
    .WithUsername("test")
    .WithPassword("test")
    .Build();

// Redis
private readonly RedisContainer _redis = new RedisBuilder()
    .WithImage("redis:7")
    .Build();

// RabbitMQ
private readonly RabbitMqContainer _rabbit = new RabbitMqBuilder()
    .WithImage("rabbitmq:3-management")
    .Build();
```

**Нюанс:** Testcontainers запускает Docker-контейнер для каждого тестового класса (IClassFixture) или для каждого теста (IAsyncLifetime). Для скорости — IClassFixture + Respawn для сброса данных.

### Respawn — сброс БД между тестами

```csharp
public class DatabaseFixture : IAsyncLifetime
{
    private Respawner _respawner = null!;
    public string ConnectionString { get; private set; } = null!;

    public async Task InitializeAsync()
    {
        // ... запуск Testcontainers, миграции
        _respawner = await Respawner.CreateAsync(ConnectionString, new RespawnerOptions
        {
            TablesToIgnore = ["__EFMigrationsHistory"],
            DbAdapter = DbAdapter.Postgres
        });
    }

    public async Task ResetDatabaseAsync()
        => await _respawner.ResetAsync(ConnectionString);

    public Task DisposeAsync() => Task.CompletedTask;
}
```

---

## Bogus — генерация тестовых данных

```csharp
var faker = new Faker<CreateOrderDto>()
    .RuleFor(o => o.CustomerName, f => f.Person.FullName)
    .RuleFor(o => o.Email, f => f.Internet.Email())
    .RuleFor(o => o.Total, f => f.Finance.Amount(10, 1000))
    .RuleFor(o => o.Items, f => f.Make(3, () =>
        new OrderItemDto(f.Commerce.ProductName(), f.Random.Int(1, 5), f.Finance.Amount())));

var testOrders = faker.Generate(100);
```

---

## Naming Convention

```
Method_Scenario_ExpectedResult

GetOrder_WhenExists_ReturnsOrder
GetOrder_WhenNotFound_ReturnsNull
CreateOrder_InvalidTotal_ReturnsValidationError
CreateOrder_DuplicateId_ThrowsConflictException
```

---

## Что тестировать / не тестировать

| Тестировать | Не тестировать |
|-------------|----------------|
| Бизнес-логику, валидацию | Framework internals |
| Граничные случаи (null, empty, max) | Простые CRUD без логики |
| Error handling | Private методы напрямую |
| API endpoints (интеграционно) | Конфигурацию DI |
| Mapping логику | Get/Set свойства |

---

## Best Practices

- **Один assert на тест** (или логически связанные). Имя: `Method_Scenario_Expected`.
- **AAA** — Arrange, Act, Assert. Пустая строка между секциями.
- **Не мокать то, что не контролируешь** — HttpClient, DbContext мокать через абстракции (интерфейсы), а не напрямую.
- **Testcontainers > InMemory** — InMemory БД не ведёт себя как реальная. Testcontainers даёт уверенность.
- **CancellationToken** — передавать в async-тесты. `_cts.Token` или `TestContext.Current.CancellationToken`.
- **Не тестировать фреймворк** — не проверять что DI работает, что EF сохраняет данные. Тестировать бизнес-логику.
- **Параллелизм** — xUnit запускает тесты параллельно по умолчанию. Для shared state — `[Collection("Database")]`.

---

## См. также

- [Architecture Tests (NetArchTest)](../../Architecture/architecture-tests-netarchtest.md)
- [Docker и CI/CD](../Docker/docker-deploy.md)
- [Interview: Testing](../../Interview/8-testing.md)
