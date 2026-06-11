---
tags: [testing, xunit, tunit, testcontainers, mutation-testing, property-based, playwright, aspire, nbomber]
level: Senior
---

# Тестирование — стратегия и инструменты

## Что это, зачем и когда

### Что такое тестирование?
**Автоматическая проверка, что код работает правильно.** Написал тест → запустил → зелёный = всё ок, красный = баг.

**Аналогия:** Чек-лист перед вылетом самолёта. Пилот не «верит», что шасси работает — он проверяет по списку. Тесты — чек-лист для кода, который запускается на каждое изменение.

### Зачем?

| Без тестов | С тестами |
|-----------|-----------|
| «Работает на моей машине» | Работает везде — CI проверяет на каждый PR |
| Поменял одно — сломалось в другом месте | Тесты сразу покажут что сломалось |
| Боишься рефакторить | Рефакторишь смело — тесты страхуют |
| Баг в продакшене, клиент нашёл | Баг найден в тесте, до production не дошёл |
| "Что делает этот код?" — никто не помнит | Тесты — живая документация |

### Виды тестов — пирамида и алмаз

| Тип | Что | Скорость | Когда |
|-----|-----|----------|-------|
| **Unit** | Один метод/класс в изоляции | мс | Бизнес-логика, Value Objects, чистые функции |
| **Integration** | Связка компонентов (API → БД) | сек | Endpoints, репозитории, EF queries |
| **Contract** | API ↔ consumer контракт | сек | Microservices, public APIs |
| **E2E** | Весь путь пользователя (UI → API → DB) | мин | Критические сценарии (оплата, регистрация) |
| **Load** | Поведение под нагрузкой | мин | Pre-launch, perf regression |
| **Mutation** | Качество тестов (а не кода) | долго | Periodic, для critical bizlogic |
| **Property-based** | Generative invariants | сек | Парсеры, алгоритмы, math, переключатели состояний |

### Пирамида vs Алмаз

| Подход | Распределение | Когда |
|--------|--------------|-------|
| **Пирамида** | Много unit → меньше integration → мало E2E | Сложная бизнес-логика (DDD, domain rich) |
| **Алмаз** | Мало unit → много integration → мало E2E | Backend (80% — интеграция с БД и API) |
| **Trophy** | Static (TS/Roslyn) → unit → integration → E2E | Frontend, but tools matter |

> [!question]- **Интервью: Пирамида vs Алмаз тестирования?**
> **Пирамида** — много unit, меньше integration, мало E2E. Хороша когда у тебя много чистой логики (DDD aggregates, calculation engines).
> **Алмаз (diamond)** — для типичного backend: больше integration тестов (WebApplicationFactory + Testcontainers), меньше unit. Причина: backend-код = интеграция (DB, HTTP, message bus). Integration тесты ловят реальные баги, mock-based unit'ы могут пройти при сломанной интеграции.

---

## Stack 2026

| Tool | Назначение | NuGet | Статус |
|------|-----------|-------|--------|
| **xUnit** | Тестовый фреймворк (default) | `xunit` | ✅ OSS, де-факто стандарт |
| **TUnit** | Modern alternative (.NET 8+, source generators) | `TUnit` | ✅ OSS, для новых проектов worth checking |
| **NSubstitute** | Mocking | `NSubstitute` | ✅ OSS |
| **Shouldly** | Assertions (рекомендуется) | `Shouldly` | ✅ OSS — **замена FluentAssertions** |
| **FluentAssertions** | Assertions (legacy) | `FluentAssertions` | ⚠️ С 2025 — commercial для prod use |
| **Testcontainers** | Docker для integration tests | `Testcontainers.PostgreSql` etc. | ✅ OSS |
| **WebApplicationFactory** | In-memory ASP.NET Core | `Microsoft.AspNetCore.Mvc.Testing` | ✅ Built-in |
| **Aspire Testing** | Distributed-app в тестах | `Aspire.Hosting.Testing` | ✅ Microsoft |
| **Bogus** | Fake data generation | `Bogus` | ✅ OSS |
| **AutoFixture** | Auto-create test objects | `AutoFixture` | ✅ OSS |
| **Respawn** | DB reset между тестами | `Respawn` | ✅ OSS |
| **Verify** | Snapshot testing | `Verify.Xunit` | ✅ OSS |
| **WireMock.NET** | HTTP service stub | `WireMock.Net` | ✅ OSS |
| **Stryker.NET** | Mutation testing | `dotnet-stryker` (CLI) | ✅ OSS |
| **FsCheck** | Property-based tests | `FsCheck.Xunit` | ✅ OSS |
| **Playwright** | E2E browser tests | `Microsoft.Playwright` | ✅ Microsoft |
| **NBomber** | Load testing на C# | `NBomber` | ✅ OSS |

> [!warning] **FluentAssertions с 2025 — платная для commercial use**
> Автор перешёл на коммерческую лицензию (V8+). OSS и личные проекты — условно бесплатно. Production commercial — нужна лицензия.
>
> Миграция на Shouldly:
> ```csharp
> // FluentAssertions          →  Shouldly
> result.Should().Be(42);          result.ShouldBe(42);
> result.Should().NotBeNull();     result.ShouldNotBeNull();
> result.Should().BeEmpty();       result.ShouldBeEmpty();
> list.Should().Contain(x);        list.ShouldContain(x);
> action.Should().Throw<Ex>();     Should.Throw<Ex>(action);
> ```
> То же касается **FluentValidation** (тот же автор, тот же ход).

---

## xUnit basics

```csharp
public class OrderServiceTests
{
    private readonly IOrderRepository _repo = Substitute.For<IOrderRepository>();
    private readonly OrderService _sut;

    public OrderServiceTests() => _sut = new OrderService(_repo);

    [Fact]
    public async Task GetOrder_WhenExists_ReturnsOrder()
    {
        var expected = new Order { Id = Guid.NewGuid(), Total = 100m };
        _repo.GetByIdAsync(expected.Id, Arg.Any<CancellationToken>()).Returns(expected);

        var result = await _sut.GetAsync(expected.Id);

        result.ShouldNotBeNull();
        result.Total.ShouldBe(100m);
    }

    [Theory]
    [InlineData(0, false)]
    [InlineData(-1, false)]
    [InlineData(100, true)]
    public async Task CreateOrder_ValidatesTotal(decimal total, bool expectedSuccess)
    {
        var result = await _sut.CreateAsync(new CreateOrderDto("Customer", total));
        result.IsSuccess.ShouldBe(expectedSuccess);
    }
}
```

### Naming convention

```
Method_Scenario_ExpectedResult

GetOrder_WhenExists_ReturnsOrder
GetOrder_WhenNotFound_ReturnsNull
CreateOrder_InvalidTotal_ReturnsValidationError
CreateOrder_DuplicateId_ThrowsConflictException
```

### Fixtures для shared state

```csharp
public class DatabaseFixture : IAsyncLifetime
{
    public string ConnectionString { get; private set; } = null!;

    public async Task InitializeAsync()
    {
        // setup once for all tests using this fixture
    }
    public Task DisposeAsync() => Task.CompletedTask;
}

public class OrderTests : IClassFixture<DatabaseFixture>
{
    public OrderTests(DatabaseFixture fixture) { /* ... */ }
}

// Cross-class shared fixture
[CollectionDefinition("Database")]
public class DatabaseCollection : ICollectionFixture<DatabaseFixture> { }

[Collection("Database")]
public class OrderTests { /* ... */ }

[Collection("Database")]
public class CustomerTests { /* ... */ }
```

---

## TUnit — modern alternative

`TUnit` использует Source Generators вместо reflection — быстрее, AOT-friendly, modern API.

```csharp
[Test]
[Arguments(0, false)]
[Arguments(-1, false)]
[Arguments(100, true)]
public async Task CreateOrder(decimal total, bool expected)
{
    var result = await _sut.CreateAsync(new CreateOrderDto("Customer", total));
    await Assert.That(result.IsSuccess).IsEqualTo(expected);
}

[Test]
[Repeat(3)]   // запустить 3 раза
[ParallelLimiter<MyParallelLimiter>]  // own concurrency limit
public async Task RaceConditionTest() { /* ... */ }
```

| | xUnit | TUnit |
|--|-------|-------|
| Maturity | Production stable | Newer, под активным развитием |
| AOT support | Limited | Native AOT support |
| Speed | Reflection | Source-gen (faster) |
| Hooks | `IClassFixture`, attributes | Lifecycle hooks через attributes |
| Когда | По умолчанию для большинства проектов | New projects, AOT requirements |

xUnit для existing — нет смысла мигрировать. Но для нового проекта — стоит попробовать TUnit.

---

## NSubstitute — mocking

```csharp
// Setup
_repo.GetByIdAsync(Arg.Any<Guid>(), Arg.Any<CancellationToken>())
    .Returns(new Order());

// Conditional setup
_repo.GetByIdAsync(Arg.Is<Guid>(id => id != Guid.Empty), Arg.Any<CancellationToken>())
    .Returns(new Order());

// Sequence — разные ответы при последовательных вызовах
_repo.GetByIdAsync(Arg.Any<Guid>(), Arg.Any<CancellationToken>())
    .Returns(_ => null, _ => new Order());

// Throw
_repo.GetByIdAsync(Arg.Any<Guid>(), Arg.Any<CancellationToken>())
    .ThrowsAsync(new TimeoutException("DB timeout"));

// Verify call
await _repo.Received(1).AddAsync(
    Arg.Is<Order>(o => o.Total == 100m),
    Arg.Any<CancellationToken>());

// Verify NOT called
await _repo.DidNotReceive().DeleteAsync(Arg.Any<Guid>(), Arg.Any<CancellationToken>());
```

### NSubstitute vs Moq

| | NSubstitute | Moq |
|--|-------------|-----|
| Syntax | Простой, "natural" | Verbose lambdas |
| Strictness | Auto-mocking (silent) | Strict by default |
| When | Default для новых | Legacy code или strict mode |

### Когда **не** мокать

```csharp
// ❌ Не мокай DbContext напрямую — мок не ловит SQL ошибки
var mockDb = Substitute.For<AppDbContext>();

// ✅ Используй Testcontainers с реальной БД
```

```csharp
// ❌ Не мокай HttpClient напрямую (sealed class)
var mockHttp = Substitute.For<HttpClient>();  // throws

// ✅ Mock через DelegatingHandler
public class MockHandler(Func<HttpRequestMessage, Task<HttpResponseMessage>> handler) : DelegatingHandler
{
    protected override Task<HttpResponseMessage> SendAsync(HttpRequestMessage req, CancellationToken ct) =>
        handler(req);
}

// или WireMock.NET
```

---

## Integration tests — `WebApplicationFactory`

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
            services.RemoveAll<DbContextOptions<AppDbContext>>();
            services.AddDbContext<AppDbContext>(opts =>
                opts.UseNpgsql(_db.GetConnectionString()));

            // Замена внешнего HTTP-клиента на mock
            services.RemoveAll<IPaymentClient>();
            services.AddSingleton<IPaymentClient>(new FakePaymentClient());
        });
        builder.UseEnvironment("Testing");
    }

    public async Task InitializeAsync()
    {
        await _db.StartAsync();
        // Apply migrations
        using var scope = Services.CreateScope();
        await scope.ServiceProvider.GetRequiredService<AppDbContext>().Database.MigrateAsync();
    }

    public new async Task DisposeAsync()
    {
        await _db.DisposeAsync();
        await base.DisposeAsync();
    }
}

public class OrdersApiTests(CustomWebApplicationFactory factory) : IClassFixture<CustomWebApplicationFactory>
{
    private readonly HttpClient _client = factory.CreateClient();

    [Fact]
    public async Task CreateOrder_ValidRequest_Returns201()
    {
        var dto = new CreateOrderRequest("Customer", [new("Product1", 2, 50m)]);

        var response = await _client.PostAsJsonAsync("/api/orders", dto);

        response.StatusCode.ShouldBe(HttpStatusCode.Created);
        var created = await response.Content.ReadFromJsonAsync<OrderResponse>();
        created!.Id.ShouldNotBe(Guid.Empty);
    }
}
```

### Authentication для integration tests

```csharp
// Test scheme — фейковый JWT
builder.ConfigureServices(services =>
{
    services.AddAuthentication("TestScheme")
        .AddScheme<AuthenticationSchemeOptions, TestAuthHandler>("TestScheme", _ => { });
});

public class TestAuthHandler(...) : AuthenticationHandler<AuthenticationSchemeOptions>(...)
{
    protected override Task<AuthenticateResult> HandleAuthenticateAsync()
    {
        var claims = new[] {
            new Claim(ClaimTypes.NameIdentifier, "test-user"),
            new Claim(ClaimTypes.Role, "Admin"),
        };
        var identity = new ClaimsIdentity(claims, "TestScheme");
        return Task.FromResult(AuthenticateResult.Success(
            new AuthenticationTicket(new ClaimsPrincipal(identity), Scheme.Name)));
    }
}

// Use в тесте
client.DefaultRequestHeaders.Authorization = new("TestScheme", "test-token");
```

---

## Testcontainers — реальные сервисы в Docker

```csharp
// PostgreSQL
private readonly PostgreSqlContainer _postgres = new PostgreSqlBuilder()
    .WithImage("postgres:16")
    .WithDatabase("testdb")
    .WithUsername("test").WithPassword("test")
    .Build();

// Redis
private readonly RedisContainer _redis = new RedisBuilder()
    .WithImage("redis:7")
    .Build();

// RabbitMQ
private readonly RabbitMqContainer _rabbit = new RabbitMqBuilder()
    .WithImage("rabbitmq:3-management")
    .Build();

// Kafka
private readonly KafkaContainer _kafka = new KafkaBuilder()
    .WithImage("confluentinc/cp-kafka:latest")
    .Build();

// LocalStack — AWS in container
private readonly LocalStackContainer _aws = new LocalStackBuilder()
    .WithService(LocalStackContainer.AwsService.S3)
    .Build();

// Ollama — для LLM тестов
private readonly OllamaContainer _ollama = new OllamaBuilder()
    .WithImage("ollama/ollama:latest")
    .Build();
```

### Respawn — reset между тестами

```csharp
public class DatabaseFixture : IAsyncLifetime
{
    private Respawner _respawner = null!;
    public string ConnectionString { get; private set; } = null!;

    public async Task InitializeAsync()
    {
        // ... start container, migrate
        _respawner = await Respawner.CreateAsync(ConnectionString, new RespawnerOptions
        {
            TablesToIgnore = ["__EFMigrationsHistory"],
            DbAdapter = DbAdapter.Postgres
        });
    }

    public Task ResetDatabaseAsync() => _respawner.ResetAsync(ConnectionString);
    public Task DisposeAsync() => Task.CompletedTask;
}

[Collection("Database")]
public class OrderTests
{
    public OrderTests(DatabaseFixture fixture)
    {
        _fixture = fixture;
    }

    [Fact]
    public async Task FreshTest()
    {
        await _fixture.ResetDatabaseAsync();  // Чистая БД
        // ...
    }
}
```

Без Respawn — тесты влияют друг на друга. С — каждый стартует с чистой БД (но без перезапуска контейнера, что быстрее).

---

## Bogus + AutoFixture — fake data

```csharp
// Bogus — domain-aware fake data
var faker = new Faker<CreateOrderDto>()
    .RuleFor(o => o.CustomerName, f => f.Person.FullName)
    .RuleFor(o => o.Email, f => f.Internet.Email())
    .RuleFor(o => o.Total, f => f.Finance.Amount(10, 1000))
    .RuleFor(o => o.Items, f => f.Make(3, () =>
        new OrderItemDto(f.Commerce.ProductName(), f.Random.Int(1, 5), f.Finance.Amount())));

var orders = faker.Generate(100);
```

```csharp
// AutoFixture — анонимные данные без явного setup
var fixture = new Fixture();
var order = fixture.Create<Order>();  // Заполняет все поля random'ом

// С customizations
fixture.Customize<Order>(c => c
    .With(o => o.Total, 100m)
    .Without(o => o.CustomerId));  // оставит default
```

| | Bogus | AutoFixture |
|--|-------|-------------|
| Когда | Когда хочется realistic данных (имена, emails) | Когда не важны конкретные значения |
| Customization | Через Faker rules | Через Fixture customizations |
| Combine | Часто используют вместе |

---

## WireMock.NET — mock external HTTP

```csharp
public class PaymentApiTests : IAsyncLifetime
{
    private WireMockServer _wireMock = null!;
    private HttpClient _client = null!;

    public Task InitializeAsync()
    {
        _wireMock = WireMockServer.Start();

        _wireMock
            .Given(Request.Create().WithPath("/payments").UsingPost())
            .RespondWith(Response.Create()
                .WithStatusCode(200)
                .WithHeader("Content-Type", "application/json")
                .WithBody("""{ "id": "pay_123", "status": "succeeded" }"""));

        _client = new HttpClient { BaseAddress = new Uri(_wireMock.Url!) };
        return Task.CompletedTask;
    }

    public Task DisposeAsync()
    {
        _wireMock?.Stop();
        return Task.CompletedTask;
    }

    [Fact]
    public async Task PayWithStripe_VerifiesCorrectRequest()
    {
        await _client.PostAsJsonAsync("/payments", new { amount = 100 });

        _wireMock.LogEntries.Count.ShouldBe(1);
        var request = _wireMock.LogEntries[0].RequestMessage;
        request.Path.ShouldBe("/payments");
        request.Body.ShouldContain("100");
    }
}
```

**Когда WireMock vs `DelegatingHandler` mock:**
- WireMock — когда нужно полное HTTP behavior (headers, status codes, delays, fault injection)
- `DelegatingHandler` — простой mock на уровне C#, без сетевого стека

---

## Verify — snapshot testing

Проверяет что output **не изменился** vs предыдущий запуск.

```csharp
[Fact]
public Task GetOrder_ReturnsExpectedJson()
{
    var order = new Order { Id = Guid.Parse("..."), Total = 100m };
    return Verifier.Verify(order);
}
```

Первый запуск создаёт `OrderTests.GetOrder_ReturnsExpectedJson.received.txt`. Подтверждаешь — он становится `.verified.txt`. Следующие запуски сравниваются.

```csharp
// Verify HTTP responses
return Verifier.Verify(httpResponse);

// Verify через JSON formatter
return Verifier.Verify(complex)
    .ScrubMember("CreatedAt")     // игнорировать timestamp
    .ScrubInlineGuids();          // игнорировать GUIDs
```

**Когда Verify:**
- Сложные DTOs с многими полями (легче чем 50 assert'ов)
- Generated code (Roslyn analyzers)
- Complex JSON output API endpoints
- Migrations — что Сгенерилось в БД

---

## Mutation Testing — Stryker.NET

**Mutation testing проверяет качество твоих тестов**, не код. Stryker модифицирует твой код (заменяет `+` на `-`, `>` на `<`, `true` на `false`) и запускает тесты. Если тесты **не падают** на mutation → твоя test suite не покрывает этот case.

```bash
dotnet tool install -g dotnet-stryker

dotnet stryker --project MyApp.csproj --test-project MyApp.Tests.csproj
```

Output:
```
Mutation score: 78%

Mutants killed:    156 / 200 (78%)
Mutants survived:   42 / 200 (21%)  ← красный флаг
Timeout:             2 / 200
```

**Survived mutants** = тесты не отлавливают конкретные изменения. Investigate — почему. Часто это:
- Edge cases не покрыты
- Тест проверяет что метод вызвался, а не что результат корректный
- Ассерты слишком loose

### Когда mutation testing

- Periodic (раз в неделю/месяц) на CI
- Перед production launch — guarantee критичной business logic
- На вновь добавленный код (catch уровень покрытия для того что ты только что написал)
- Не на legacy с tonнами кода — слишком долго

### Pitfalls

- Mutation testing **долгий** — runs тестов × N mutants. На большом проекте — часы.
- "Equivalent mutants" — изменения которые семантически тот же код (`x * 1` → `x`, `x + 0` → `x`). Survived но не баг. Filter.
- Strikker config — exclude generated code, third-party libs.

> [!question]- **Интервью: что такое mutation testing и зачем?**
> Это **тестирование тестов**. Tool изменяет твой код в небольшой степени (mutation), запускает test suite. Если тесты не упали — они недостаточно строгие.
> **Coverage 100% != quality 100%.** Coverage показывает что строки execute'нулись. Mutation testing показывает что они actually проверены — что assertions проверяют правильные вещи.

---

## Property-based Testing — FsCheck

Вместо явных examples — описываешь **invariants**, FsCheck генерирует случайные inputs.

```csharp
[Property]
public bool ReverseTwiceIsIdentity(int[] input)
{
    return input.Reverse().Reverse().SequenceEqual(input);
}

[Property]
public bool SortedIsSorted(int[] input)
{
    var sorted = input.OrderBy(x => x).ToArray();
    for (int i = 1; i < sorted.Length; i++)
        if (sorted[i] < sorted[i - 1]) return false;
    return true;
}
```

FsCheck запускает 100 random inputs (разные размеры массивов, edge cases, и т.п.). Если найдёт failing input — **shrinks** его до минимального reproducer.

### Production examples

```csharp
[Property]
public bool DiscountNeverNegative(decimal price, decimal discountPercent)
{
    if (price < 0 || discountPercent < 0) return true;  // skip invalid
    var final = ApplyDiscount(price, discountPercent);
    return final >= 0;  // invariant: result never negative
}

[Property]
public bool PriceWithVatGreaterThanWithout(decimal price)
{
    if (price <= 0) return true;
    return ApplyVat(price) > price;
}
```

### Когда применять

| Применять | Не применять |
|----------|--------------|
| Парсеры (round-trip parse → serialize) | CRUD без логики |
| Алгоритмы (сортировка, поиск, графы) | UI компоненты |
| Math / финансовые формулы | Side-effect heavy code |
| State machines (transitions invariants) | Tests где трудно описать invariant |
| Compression / encoding (decode reverses encode) | Когда нет clear property |

Property-based **не заменяет** example-based unit tests, **дополняет**. Property checks general law, example tests конкретные scenarios.

---

## Aspire Testing — distributed system

`.NET Aspire` — фреймворк для multi-service apps. `Aspire.Hosting.Testing` — поднимает всю distributed app в test environment (контейнеры, services, dependencies).

```csharp
public class DistributedAppTests : IAsyncLifetime
{
    private DistributedApplication _app = null!;

    public async Task InitializeAsync()
    {
        var builder = await DistributedApplicationTestingBuilder
            .CreateAsync<Projects.MyApp_AppHost>();

        _app = await builder.BuildAsync();
        await _app.StartAsync();
    }

    public async Task DisposeAsync()
    {
        await _app.DisposeAsync();
    }

    [Fact]
    public async Task ApiCallsDownstream()
    {
        var httpClient = _app.CreateHttpClient("api");
        var response = await httpClient.GetAsync("/api/health");
        response.IsSuccessStatusCode.ShouldBeTrue();
    }
}
```

Полезно для:
- Тестов saga-flow через несколько сервисов
- Integration tests с full message bus + DB + cache
- Verifying что services правильно регистрируются и discoverable

---

## Playwright — E2E browser tests

```bash
dotnet add package Microsoft.Playwright
pwsh bin/Debug/net10.0/playwright.ps1 install
```

```csharp
public class CheckoutFlowTests : PageTest
{
    [Fact]
    public async Task CompleteCheckout()
    {
        await Page.GotoAsync("https://staging.myapp.com/products");

        await Page.GetByRole(AriaRole.Button, new() { Name = "Add to cart" }).ClickAsync();
        await Page.GetByRole(AriaRole.Link, new() { Name = "Cart" }).ClickAsync();

        await Page.GetByLabel("Email").FillAsync("test@example.com");
        await Page.GetByLabel("Card number").FillAsync("4242424242424242");

        await Page.GetByRole(AriaRole.Button, new() { Name = "Pay" }).ClickAsync();

        await Expect(Page.GetByText("Order confirmed")).ToBeVisibleAsync();
        await Expect(Page).ToHaveURLAsync(new Regex(".*\\/orders\\/.*"));
    }
}
```

### Tracing для debugging failed tests

```csharp
[Fact]
public async Task FlakyTest()
{
    await Context.Tracing.StartAsync(new() { Screenshots = true, Snapshots = true });
    try
    {
        await Page.GotoAsync(...);
        // ...
    }
    finally
    {
        await Context.Tracing.StopAsync(new() { Path = "trace.zip" });
    }
}
```

`trace.zip` → открыть в `playwright show-trace trace.zip` — timeline всего теста с screenshots на каждом step. Лучший debugging tool для E2E.

### Когда E2E

- Critical user flows (signup, checkout, payment)
- Regression catch для UI-heavy apps
- **Не** на каждом PR — медленно (минуты на тест)
- Flaky risk — пиши idempotent (`expect`, не `wait_for_timeout`), retry в CI

---

## NBomber — load testing

C#-native альтернатива k6/JMeter.

```csharp
var scenario = Scenario.Create("api_load", async ctx =>
{
    var response = await _httpClient.GetAsync("/api/products");
    return response.IsSuccessStatusCode ? Response.Ok() : Response.Fail();
})
.WithLoadSimulations(
    Simulation.RampingInject(rate: 100, interval: TimeSpan.FromSeconds(1), during: TimeSpan.FromMinutes(2)),
    Simulation.KeepConstant(copies: 100, during: TimeSpan.FromMinutes(5)));

NBomberRunner
    .RegisterScenarios(scenario)
    .WithReportFolder("reports")
    .WithReportFormats(ReportFormat.Html, ReportFormat.Md)
    .Run();
```

### Что включать в тест

```csharp
.WithLoadSimulations(
    // Warm-up — 30s на 10 RPS
    Simulation.RampingInject(rate: 10, interval: TimeSpan.FromSeconds(1), during: TimeSpan.FromSeconds(30)),
    // Normal — 5 минут на 100 RPS
    Simulation.KeepConstant(copies: 100, during: TimeSpan.FromMinutes(5)),
    // Spike — резкий up до 1000 RPS
    Simulation.RampingInject(rate: 1000, interval: TimeSpan.FromSeconds(1), during: TimeSpan.FromSeconds(30)),
    // Hold spike
    Simulation.KeepConstant(copies: 1000, during: TimeSpan.FromSeconds(60))
);
```

### Когда load testing

- Pre-launch capacity planning
- Verify SLO holds под expected traffic
- Regression check после major changes
- Before/after оптимизаций

См. подробно в[System Design]()) — capacity estimation.

---

## Architecture tests — NetArchTest

См. подробно в[arch-tests.md]()). Кратко — проверка архитектурных правил кодом.

```csharp
[Fact]
public void Domain_DoesNotDependOn_Infrastructure()
{
    var result = Types.InAssembly(typeof(Order).Assembly)
        .That()
        .ResideInNamespace("MyApp.Domain")
        .ShouldNot()
        .HaveDependencyOn("MyApp.Infrastructure")
        .GetResult();

    result.IsSuccessful.ShouldBeTrue();
}
```

---

## Best Practices

### 1. AAA pattern — Arrange, Act, Assert

```csharp
[Fact]
public async Task GetOrder_WhenExists_ReturnsOrder()
{
    // Arrange
    var orderId = Guid.NewGuid();
    _repo.GetByIdAsync(orderId, default).Returns(new Order { Id = orderId });

    // Act
    var result = await _sut.GetAsync(orderId);

    // Assert
    result.ShouldNotBeNull();
    result.Id.ShouldBe(orderId);
}
```

### 2. Один логический assert на тест
Не "проверяю всё что accidentally в моём mind". Один тест = одна гипотеза.

### 3. Test data builders для readability

```csharp
public class OrderBuilder
{
    private decimal _total = 100m;
    private string _customer = "Test";

    public OrderBuilder WithTotal(decimal t) { _total = t; return this; }
    public OrderBuilder WithCustomer(string c) { _customer = c; return this; }
    public Order Build() => new Order { Total = _total, Customer = _customer };
}

// Использование
var order = new OrderBuilder().WithTotal(0).Build();  // edge case
```

### 4. Avoid magic values

```csharp
// ❌
result.Total.ShouldBe(150);  // что 150?

// ✅
const decimal expectedTotal = 100m + 50m * 1m;  // base + tax
result.Total.ShouldBe(expectedTotal);
```

### 5. Cancellation tokens

```csharp
[Fact]
public async Task LongOperation_CancellationWorks()
{
    var cts = new CancellationTokenSource(TimeSpan.FromSeconds(5));
    await Should.ThrowAsync<OperationCanceledException>(() =>
        _sut.LongRunningAsync(cts.Token));
}
```

### 6. Avoid sleeps — wait for conditions

```csharp
// ❌ Flaky
await Task.Delay(1000);
result.Status.ShouldBe("Processed");

// ✅ Wait for condition
await TestHelper.WaitFor(() => result.Status == "Processed", TimeSpan.FromSeconds(5));
```

---

## Common pitfalls

### 1. Тестирование framework
Проверяешь что DI работает, что EF Core save'ит. Не твоя ответственность.

### 2. Tightly coupled tests
Один тест зависит от другого ("test1 created data, test2 uses"). Тесты должны быть независимы.

### 3. Mock'и того что не контролируешь
DateTime.Now, Guid.NewGuid, environment variables — wrap в abstraction (`IClock`, `IGuidProvider`), мокай abstraction.

### 4. InMemory provider EF

```csharp
// ❌ InMemory не ведёт себя как реальная БД (нет constraints, transactions)
opts.UseInMemoryDatabase("test");

// ✅ Testcontainers
```

### 5. Не очищать БД между тестами
Один тест портит state, второй падает по случайной причине.

### 6. Слишком много setup в каждом тесте
Много lines в Arrange — сложно понять что тестируется. Используй builders.

### 7. Глаголы в Assert messages

```csharp
// ❌
result.ShouldBe(42);  // если упадёт — сообщение неинформативно

// ✅
result.ShouldBe(42, "User balance should be 42 after credit");
```

### 8. Async tests без await

```csharp
// ❌ Тест "проходит" даже если внутри exception
[Fact]
public Task Test() => DoSomethingAsync();  // если вернёт failed Task — silently игнор

// ✅
[Fact]
public async Task Test() => await DoSomethingAsync();
```

### 9. Не запускать тесты в CI
Тесты которые не запускаются — это документация, не safety net.

### 10. Branch coverage as goal
"Хочу 100% coverage" — пишутся бессмысленные тесты ради метрики. Цель — **уверенность в коде**, не цифра.

---

## Testing checklist

- [ ] xUnit или TUnit как framework
- [ ] Shouldly как assertions (миграция с FluentAssertions если был)
- [ ] WebApplicationFactory для integration tests
- [ ] Testcontainers для real DB / Redis / RabbitMQ
- [ ] Respawn для DB reset
- [ ] Bogus или AutoFixture для test data
- [ ] WireMock.NET для external HTTP mocking
- [ ] Verify для snapshot тестов сложных DTOs
- [ ] Stryker.NET periodic для critical bizlogic
- [ ] Property-based (FsCheck) для парсеров / алгоритмов
- [ ] Playwright для critical E2E user flows
- [ ] NBomber для load tests перед launches
- [ ] CI запускает all tests на каждый PR
- [ ] Coverage report в PR (Codecov / SonarCloud)
- [ ] Architecture tests (NetArchTest) для слоёв
- [ ] Performance regression check (BenchmarkDotNet baseline в CI)

---

## Case Studies

### Case Study #1 — Refactoring legacy "god class"

**Сценарий:** `OrderManager` класс — 3000 строк, 50 методов, 20 dependencies. Невозможно testить.

**Strategy:**
1. Identify responsibilities — найти SRP violations
2. Extract interfaces — `IOrderRepository`, `IPaymentService`, `INotificationService`
3. Move methods → small focused classes
4. Tests перед refactoring (characterization tests)
5. Refactor пошагово, run tests после каждого step

**Result:**
- 3000 строк → 8 классов по 200-400 строк each
- Test coverage: 5% → 80%
- Onboarding new developer: 2 weeks → 3 days

---

### Case Study #2 — Code review где AI помогает

**Сценарий:** Senior просматривает PR от Junior'а. Стандартные issues: naming, missing nulls, dead code.

**Workflow:**
1. **AI первый pass** — Copilot Chat / Claude review кода
2. **Senior валидирует AI feedback** — отбрасывает false positives
3. **Senior фокусируется на architecture** — что AI не видит:
   - Domain logic correctness
   - Business rule violations
   - Performance implications
   - Security in context

**Time saved:** 30 min review → 10 min (AI на mechanical, senior на important).

См.[[ai-coding-tools|AI Coding Tools]].

---

### Case Study #3 — Static analysis для legacy

**Сценарий:** 200K LOC unmaintained code. Hidden bugs, технический долг.

**Tools setup:**
```xml
<!-- Directory.Build.props -->
<Project>
  <PropertyGroup>
    <AnalysisLevel>latest-recommended</AnalysisLevel>
    <TreatWarningsAsErrors>true</TreatWarningsAsErrors>
    <EnforceCodeStyleInBuild>true</EnforceCodeStyleInBuild>
  </PropertyGroup>
  <ItemGroup>
    <PackageReference Include="Microsoft.CodeAnalysis.NetAnalyzers" Version="*" />
    <PackageReference Include="StyleCop.Analyzers" Version="*" />
    <PackageReference Include="SonarAnalyzer.CSharp" Version="*" />
  </ItemGroup>
</Project>
```

**Phased approach:**
1. Week 1: Fix critical security issues (secrets, SQL injection)
2. Week 2-4: Fix top-10 most violated rules
3. Month 2: Enable analyzer warnings as errors на новый код
4. Quarter: Full compliance + maintain через CI gate

См. [[static-analysis|Static Analysis]] и[[cicd-github-actions|CI/CD]].


---

## Cheat sheet

| Quality concern | Tool / Practice |
|-----------------|-----------------|
| Code style enforcement | EditorConfig + dotnet format |
| Static analysis | Microsoft.CodeAnalysis.NetAnalyzers |
| Style rules | StyleCop.Analyzers |
| Security scanning | SonarAnalyzer.CSharp, GitHub CodeQL |
| Vulnerability scanning | `dotnet list package --vulnerable` |
| Outdated packages | `dotnet list package --outdated` |
| Dead code | ReSharper / dotnet-ide-cli `unused-code` |
| Cyclomatic complexity | NDepend, SonarQube |
| Test coverage | coverlet + ReportGenerator |
| Mutation testing | Stryker.NET |
| Architecture tests | NetArchTest, ArchUnitNET |
| Contract tests | PactNet, Pact.NET |
| Code review | PR + GitHub Copilot review |
| Pre-commit hooks | Husky.NET + lint-staged |
| CI quality gate | SonarCloud / Codacy |

| Refactoring smell | Action |
|-------------------|--------|
| Long method (50+ lines) | Extract method |
| Long parameter list (4+) | Parameter object |
| Duplicate code | Extract to function/class |
| Switch statement | Polymorphism (Strategy) |
| Feature envy | Move method к нужному classу |
| Data clumps | Wrap в class/record |
| Primitive obsession | Value objects (Money, Email) |
| God class | Split по SRP |
| Shotgun surgery | Cohesion problem — restructure |


---

## Decision tree

```
Quality issue?
│
├── Code style inconsistencies?
│   → EditorConfig + dotnet format в pre-commit
│
├── Hidden bugs / vulnerabilities?
│   ├── Logic bugs → Roslyn analyzers
│   ├── Security → SonarAnalyzer + CodeQL
│   └── Vulnerabilities → npm audit equivalent для NuGet
│
├── Test quality concerns?
│   ├── Coverage low → coverlet + minimum threshold в CI
│   ├── Tests pass but bugs ship → mutation testing (Stryker)
│   └── Flaky tests → identify + isolate (TestCategory)
│
├── Architectural drift?
│   ├── Boundaries violated → NetArchTest assertions в tests
│   ├── Dependencies wrong direction → dependency cruiser
│   └── Anti-patterns spreading → SonarQube + custom rules
│
├── Big tech debt?
│   ├── Identify → "boy scout rule" — лучше чем нашёл
│   ├── Plan → backlog с estimate
│   ├── Critical → 20% sprint capacity
│   └── Refactor → tests first, small steps
│
└── Code review bottleneck?
    ├── Junior уровень → AI first pass
    ├── Standards inconsistent → automated checks в CI
    └── Slow → smaller PRs, clear conventions
```


---

## См. также

-[Architecture Tests]()) — NetArchTest для архитектурных правил
-[Performance]()) — BenchmarkDotNet, NBomber, profiling
-[Distributed Systems]()) — testing distributed flows, saga
-[Blazor Server]()) — bUnit для component testing
-[Auth и Security]()) — TestAuthHandler для integration tests
-[Source Generators]()) — Verify для snapshot-тестирования генератора

## Reading list

- **xUnit docs** — xunit.net
- **Testcontainers .NET** — dotnet.testcontainers.org
- **Stryker.NET** — stryker-mutator.io/docs/stryker-net/introduction/
- **FsCheck book** — fscheck.github.io/FsCheck/
- **Microsoft Playwright docs** — playwright.dev/dotnet/
- **NBomber docs** — nbomber.com/docs
- **Verify** — github.com/VerifyTests/Verify (snapshot testing)
- **Modern Test-Driven Development in .NET** — Mark Seemann (про unit tests done right)
- **xUnit Test Patterns** — Gerard Meszaros (canonical reference)
- **Working Effectively with Legacy Code** — Michael Feathers (как добавить тесты в код без тестов)
