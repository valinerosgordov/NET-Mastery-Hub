---
tags: [testing, integration, testcontainers, webapplicationfactory, e2e]
level: Senior
date: 2026-04-30
---

# Integration Testing

> Глубокий гайд по integration tests в .NET. Закрывает: WebApplicationFactory, Testcontainers, тестирование с реальной БД, мокинг external services через WireMock, snapshot tests, contract tests, parallel execution, CI integration.

---

## Что это, зачем и когда

### Test pyramid

```
         /\
        /E2E\        ← очень мало (~5%), дорогие, slow
       /------\
      /Integr. \     ← средне (~25%), тестируют integration
     /----------\
    /   Unit     \   ← много (~70%), быстрые, дешёвые
   /--------------\
```

| Тип | Объём | Скорость | Стабильность |
|-----|-------|----------|--------------|
| **Unit** | 70-80% | <10ms | Stable |
| **Integration** | 15-25% | 10-100ms | Mostly stable |
| **E2E** | 1-5% | sec-min | Flaky |

### Когда integration > unit

✅ **Integration лучше когда:**
- API endpoint testing end-to-end
- EF Core query поведение (сложно мочить)
- Authentication / Authorization flow
- Real DB-specific behavior (constraints, transactions)
- Several layers взаимодействуют сложно

✅ **Unit лучше когда:**
- Pure domain logic
- Algorithm correctness
- Edge cases
- Скорость важна (TDD)

### Stack 2026

```
- xUnit / NUnit               — runner
- WebApplicationFactory       — real ASP.NET host
- Testcontainers              — Docker контейнеры в тестах
- Shouldly / FluentAssertions — readable asserts
- Bogus                       — fake test data
- WireMock.NET                — mock external HTTP
- Verify                      — snapshot testing
- Respawn                     — DB cleanup между тестами
```

---

## 1. WebApplicationFactory

```bash
dotnet add package Microsoft.AspNetCore.Mvc.Testing
```

```csharp
public class OrdersApiTests : IClassFixture<WebApplicationFactory<Program>>
{
    private readonly WebApplicationFactory<Program> _factory;

    public OrdersApiTests(WebApplicationFactory<Program> factory) => _factory = factory;

    [Fact]
    public async Task GetOrders_returns_ok()
    {
        var client = _factory.CreateClient();
        var response = await client.GetAsync("/api/orders");

        response.EnsureSuccessStatusCode();
        var orders = await response.Content.ReadFromJsonAsync<List<OrderDto>>();
        orders.ShouldNotBeNull();
    }
}
```

`WebApplicationFactory` поднимает **полный ASP.NET host в памяти** — все middleware, DI, controllers — без HTTP/TCP. Запросы через `HttpClient` идут сразу в pipeline.

### Custom factory

```csharp
public class CustomWebFactory : WebApplicationFactory<Program>
{
    protected override void ConfigureWebHost(IWebHostBuilder builder)
    {
        builder.ConfigureServices(services =>
        {
            services.RemoveAll<DbContextOptions<AppDbContext>>();
            services.AddDbContext<AppDbContext>(options =>
                options.UseInMemoryDatabase("Testing"));

            services.RemoveAll<IPaymentService>();
            services.AddSingleton<IPaymentService, MockPaymentService>();
        });
        builder.UseEnvironment("Testing");
    }
}
```

> [!warning] InMemory DB для EF Core — не идеален
> InMemory provider не emulates SQL: constraints, transactions, RAW SQL. Лучше — Testcontainers.

---

## 2. Testcontainers

[testcontainers.com/.NET](https://testcontainers.com/getting-started/dotnet/)

Запускает Docker контейнер с настоящей БД на время тестов.

```bash
dotnet add package Testcontainers.PostgreSql
```

### Combined WebApplicationFactory + Testcontainers

```csharp
public sealed class TestWebFactory : WebApplicationFactory<Program>, IAsyncLifetime
{
    private readonly PostgreSqlContainer _postgres = new PostgreSqlBuilder()
        .WithImage("postgres:16-alpine")
        .Build();

    private readonly RedisContainer _redis = new RedisBuilder()
        .WithImage("redis:7-alpine")
        .Build();

    public async Task InitializeAsync()
    {
        await Task.WhenAll(_postgres.StartAsync(), _redis.StartAsync());
    }

    protected override void ConfigureWebHost(IWebHostBuilder builder)
    {
        builder.ConfigureServices(services =>
        {
            services.RemoveAll<DbContextOptions<AppDbContext>>();
            services.AddDbContext<AppDbContext>(options =>
                options.UseNpgsql(_postgres.GetConnectionString()));

            services.AddStackExchangeRedisCache(opts =>
                opts.Configuration = _redis.GetConnectionString());
        });
    }

    async Task IAsyncLifetime.DisposeAsync()
    {
        await Task.WhenAll(_postgres.DisposeAsync().AsTask(), _redis.DisposeAsync().AsTask());
    }
}
```

### Available containers

```
PostgreSqlBuilder      — PostgreSQL
MsSqlBuilder           — SQL Server
MySqlBuilder           — MySQL
MongoDbBuilder         — MongoDB
RedisBuilder           — Redis
RabbitMqBuilder        — RabbitMQ
KafkaBuilder           — Kafka
ElasticsearchBuilder   — Elasticsearch
MinioBuilder           — MinIO (S3)
ContainerBuilder       — Custom Docker image
```

---

## 3. Test data isolation

### Respawn — clean DB между тестами

```bash
dotnet add package Respawn
```

```csharp
public class IntegrationTest : IAsyncLifetime
{
    private readonly Respawner _respawner;

    public async Task DisposeAsync()
    {
        await _respawner.ResetAsync(_connectionString);
    }
}
```

### Transaction rollback

```csharp
public class TransactionalTest : IAsyncDisposable
{
    private IDbContextTransaction _transaction;

    [Fact]
    public async Task Test()
    {
        _transaction = await _ctx.Database.BeginTransactionAsync();
        // ... arrange / act / assert ...
    }

    public async ValueTask DisposeAsync()
    {
        await _transaction.RollbackAsync();
    }
}
```

> [!warning] Transaction rollback не работает с parallelism

Параллелизм — это только половина проблемы. Главный аргумент против rollback-изоляции — **корректность, а не скорость**. Откатываемая транзакция **никогда не делает реальный `COMMIT`**, и это прячет целый класс багов:

- **Multi-`SaveChanges` сценарии** — несколько коммитов в рамках одной бизнес-операции (outbox, двухфазная запись) внутри обёрнутой транзакции схлопываются в один scope и ведут себя иначе, чем в проде.
- **Триггеры и deferred constraints** — то, что срабатывает на `COMMIT` (`DEFERRABLE INITIALLY DEFERRED`, after-commit триггеры), при rollback не выполняется вообще.
- **Post-commit hooks** — domain-events, отправляемые после успешного коммита, MassTransit `Publish` на `SaveChanges`, `TransactionScope` completion-callbacks — все они привязаны к реальному коммиту и в rollback-тесте молчат.

Итог: тест зелёный, потому что половина пути просто не исполнялась. Предпочтительнее **session-scoped контейнер + Respawn (TRUNCATE)**: каждый тест **по-настоящему коммитит**, а очистка между тестами делается truncate'ом. Так ты тестируешь реальный commit-path, сохраняя изоляцию.

> [!warning] `MigrateAsync` vs `EnsureCreated` — схему строй миграциями
> `EnsureCreated` / `EnsureCreatedAsync` создаёт схему **напрямую из модели**, минуя весь migration pipeline. Тесты будут зелёными, даже если реальная миграция сломана (забытый столбец, кривой `down`, ручной SQL в миграции, который не отражён в модели) — на проде накатывается совсем другая схема.
>
> `MigrateAsync` прогоняет **те же самые скрипты миграций**, что и продакшен. Тогда сам suite становится guard'ом против schema drift: сломанная миграция падает в CI, а не в проде. В integration-тестах с реальной БД всегда `MigrateAsync`, `EnsureCreated` — только для выкидных smoke-проверок.

### Unique data per test

```csharp
[Fact]
public async Task GetUser_returns_user()
{
    var unique = Guid.NewGuid();
    var email = $"test-{unique}@example.com";

    await _client.PostAsJsonAsync("/api/users", new { Email = email });
    var response = await _client.GetAsync($"/api/users/by-email?email={email}");
}
```

---

## 4. WireMock.NET — мокинг external HTTP

```bash
dotnet add package WireMock.Net
```

```csharp
public class PaymentApiTests
{
    private readonly WireMockServer _wireMock = WireMockServer.Start();

    [Fact]
    public async Task PaymentService_handles_success()
    {
        _wireMock
            .Given(Request.Create()
                .WithPath("/payments")
                .UsingPost()
                .WithBody("{\"amount\": 100}"))
            .RespondWith(Response.Create()
                .WithStatusCode(200)
                .WithBodyAsJson(new { transactionId = "tx_123" }));

        var paymentService = new PaymentService(
            new HttpClient { BaseAddress = new Uri(_wireMock.Url!) });
        var result = await paymentService.ProcessAsync(100);

        result.IsSuccess.ShouldBeTrue();
    }

    [Fact]
    public async Task PaymentService_handles_5xx()
    {
        _wireMock
            .Given(Request.Create().WithPath("/payments").UsingPost())
            .RespondWith(Response.Create().WithStatusCode(500));

        var result = await _paymentService.ProcessAsync(100);
        result.IsSuccess.ShouldBeFalse();
    }
}
```

### Verify external calls

```csharp
[Fact]
public async Task PaymentService_calls_correct_endpoint()
{
    _wireMock
        .Given(Request.Create().WithPath("/payments").UsingPost())
        .RespondWith(Response.Create().WithStatusCode(200));

    await _paymentService.ProcessAsync(100);

    var requests = _wireMock.LogEntries.ToList();
    requests.Count.ShouldBe(1);
    requests[0].RequestMessage.Body.ShouldContain("\"amount\":100");
}
```

---

## 5. Snapshot testing — Verify

```bash
dotnet add package Verify.Xunit
```

```csharp
[UsesVerify]
public class ApiSnapshotTests
{
    [Fact]
    public async Task GetOrder_response_matches_snapshot()
    {
        var response = await _client.GetAsync("/api/orders/123");
        var content = await response.Content.ReadAsStringAsync();

        await Verify(content);
    }
}
```

При первом запуске создаст `*.received.txt`. Подтверждаешь как `*.verified.txt`. После — каждый запуск сравнивает.

> [!info] Когда snapshot tests
> - API contract verification
> - Generated code (source generators)
> - Complex DTOs / serialization output

---

## 6. Authentication в integration tests

### Mock auth

```csharp
public class TestAuthHandler(
    IOptionsMonitor<AuthenticationSchemeOptions> options,
    ILoggerFactory logger,
    UrlEncoder encoder)
    : AuthenticationHandler<AuthenticationSchemeOptions>(options, logger, encoder)
{
    protected override Task<AuthenticateResult> HandleAuthenticateAsync()
    {
        var claims = new[] {
            new Claim(ClaimTypes.NameIdentifier, "test-user-id"),
            new Claim(ClaimTypes.Role, "Admin")
        };
        var identity = new ClaimsIdentity(claims, "Test");
        var ticket = new AuthenticationTicket(new ClaimsPrincipal(identity), "Test");
        return Task.FromResult(AuthenticateResult.Success(ticket));
    }
}

// В Factory
services.AddAuthentication("Test")
    .AddScheme<AuthenticationSchemeOptions, TestAuthHandler>("Test", _ => { });
```

### Real JWT generation

```csharp
public static string GenerateTestJwt(string userId, string[] roles)
{
    var claims = new List<Claim> { new(ClaimTypes.NameIdentifier, userId) };
    claims.AddRange(roles.Select(r => new Claim(ClaimTypes.Role, r)));

    var key = new SymmetricSecurityKey(Encoding.UTF8.GetBytes(JWT_TEST_SECRET));
    var creds = new SigningCredentials(key, SecurityAlgorithms.HmacSha256);

    var token = new JwtSecurityToken(
        issuer: "test", audience: "test",
        claims: claims, expires: DateTime.UtcNow.AddHours(1),
        signingCredentials: creds);

    return new JwtSecurityTokenHandler().WriteToken(token);
}

client.DefaultRequestHeaders.Authorization = new("Bearer", GenerateTestJwt("user-123", ["Admin"]));
```

---

## 7. Bogus — fake test data

```bash
dotnet add package Bogus
```

```csharp
var faker = new Faker<User>()
    .RuleFor(u => u.Id, _ => Guid.NewGuid())
    .RuleFor(u => u.Name, f => f.Name.FullName())
    .RuleFor(u => u.Email, f => f.Internet.Email())
    .RuleFor(u => u.Age, f => f.Random.Int(18, 65));

var user = faker.Generate();
var users = faker.Generate(100);

// Локализация
var ruFaker = new Faker<User>("ru")
    .RuleFor(u => u.Name, f => f.Name.FullName());
```

---

## 8. Parallel execution

xUnit параллелит assemblies, не tests внутри collection.

```csharp
[CollectionDefinition("Database")]
public class DatabaseCollection : ICollectionFixture<TestWebFactory> { }

[Collection("Database")]
public class OrdersTests { }

[Collection("Database")]
public class UsersTests { }
```

```json
// xunit.runner.json
{
  "parallelizeAssembly": true,
  "parallelizeTestCollections": true,
  "maxParallelThreads": 4
}
```

---

## 9. Performance в тестах

### Slow tests = pain

#### Reuse fixtures

```csharp
// ✅ Один контейнер на класс
public class UsersTests : IClassFixture<TestWebFactory> { }

// ❌ Новый container на каждый test (~3 sec startup!)
public class BadTests
{
    [Fact] public async Task Test1() { var f = new TestWebFactory(); ... }
}
```

#### Migrate один раз

```csharp
public async Task InitializeAsync()
{
    await _postgres.StartAsync();
    using var ctx = new AppDbContext(options);
    await ctx.Database.MigrateAsync();
}
```

#### Cleanup быстрее migrate

```csharp
// Между тестами — Respawn (TRUNCATE), не DROP+CREATE
await _respawner.ResetAsync(_connectionString);
```

---

## 10. Contract testing — Pact

Между microservices — verify что producer и consumer соглашаются на API contract.

```csharp
[Fact]
public async Task UserService_returns_user()
{
    var pact = Pact.V3("OrdersService", "UsersService", config)
        .WithHttpInteractions();

    pact.UponReceiving("a request for user 123")
        .Given("user 123 exists")
        .WithRequest(HttpMethod.Get, "/users/123")
        .WillRespond()
        .WithStatus(200)
        .WithJsonBody(new { id = 123, name = "Alice" });

    await pact.VerifyAsync(async ctx =>
    {
        var client = new UsersApiClient(ctx.MockServerUri);
        var user = await client.GetUserAsync(123);
        user.Name.ShouldBe("Alice");
    });
}
```

См. [docs.pact.io](https://docs.pact.io).

---

## 11. CI integration

```yaml
# .github/workflows/integration-tests.yml
name: Integration Tests
on:
  pull_request:
  push: { branches: [main] }

jobs:
  integration:
    runs-on: ubuntu-latest

    services:
      postgres:
        image: postgres:16
        env:
          POSTGRES_PASSWORD: postgres
        ports: ['5432:5432']
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5

    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-dotnet@v4
        with: { dotnet-version: '10.0.x' }

      - name: Build
        run: dotnet build --no-restore

      - name: Integration tests
        run: dotnet test tests/MyApp.IntegrationTests --no-build
        env:
          ConnectionStrings__Default: Host=localhost;Database=test;Username=postgres;Password=postgres
```

---

## 12. Common Pitfalls

### 1. Shared static state — flaky tests

```csharp
// ❌
public static int _counter = 0;

[Fact] public void Test1() { _counter++; }
[Fact] public void Test2() { /* counter может быть 0 или 1 */ }
```

### 2. Migrations каждый test run

```csharp
// ❌ Migrate каждый тест — 5 sec each
[Fact]
public async Task Test()
{
    await ctx.Database.MigrateAsync();  // slow!
}

// ✅ Migrate один раз в InitializeAsync, использовать Respawn
```

### 3. async void в test

```csharp
// ❌ Exceptions потеряются
[Fact]
public async void Test() { ... }

// ✅
[Fact]
public async Task Test() { ... }
```

### 4. Time-dependent тесты

```csharp
// ❌ Flaky
[Fact]
public void Test_at_midnight()
{
    if (DateTime.Now.Hour == 0) { ... }  // ⚠️
}

// ✅ TimeProvider
public class TestWithTime
{
    private readonly FakeTimeProvider _time = new();

    [Fact]
    public void Test()
    {
        _time.SetUtcNow(DateTimeOffset.Parse("2024-01-01T00:00:00Z"));
    }
}
```

### 5. WebApplicationFactory не disposed

```csharp
// ✅ IClassFixture автоматически disposes
public class Tests : IClassFixture<WebApplicationFactory<Program>> { }
```

### 6. Один `DbContext` на Act и Assert — зелёный тест поверх непроверенной БД

Самый коварный false-positive в integration-тестах. Если в Act и в Assert используется **один и тот же** экземпляр `DbContext`, проверочный запрос обслуживается не из БД, а из **identity map** (EF first-level cache): контекст возвращает тот же tracked-инстанс, который ты добавил в Act. Тест зелёный, хотя `SaveChanges` мог не выполниться (забыл `await`, фильтр в `SaveChanges` отбросил entity, триггер откатил вставку) — реального round-trip к БД не было.

```csharp
// ❌ Assert читает из identity-map кэша того же контекста — SQL мог не отработать
[Fact]
public async Task CreateOrder_persists()
{
    var order = new Order { Id = Guid.NewGuid(), Total = 100m };
    Db.Orders.Add(order);
    await Db.SaveChangesAsync();

    var loaded = await Db.Orders.FindAsync(order.Id);  // вернёт tracked-инстанс из памяти
    loaded.ShouldNotBeNull();                          // зелёный даже без реального SELECT
}
```

```csharp
// ✅ Очистить ChangeTracker между Act и Assert — форсируем реальный round-trip
[Fact]
public async Task CreateOrder_persists()
{
    var order = new Order { Id = Guid.NewGuid(), Total = 100m };
    Db.Orders.Add(order);
    await Db.SaveChangesAsync();

    Db.ChangeTracker.Clear();                           // сброс identity map

    var loaded = await Db.Orders.FindAsync(order.Id);  // настоящий SELECT в БД
    loaded.ShouldNotBeNull();
}
```

Надёжнее — отдельный `DbContext` (свежий scope) для Assert: Act-контекст для записи, Assert-контекст для чтения. Тогда identity map физически пуст и каждое чтение бьёт в БД. Механика identity map подробно — [[basics-tracking|EF Core Change Tracking]].

---

## 13. Best Practices

- **Test pyramid** — 70% unit / 25% integration / 5% E2E
- **WebApplicationFactory + Testcontainers** — modern integration stack
- **Respawn** для cleanup между tests
- **Bogus** для realistic fake data
- **WireMock.NET** для external HTTP services
- **Verify** для snapshot tests на API contracts
- **Per-test уникальные данные** для parallelism
- **One container per test class** или shared если данные изолированы
- **Migrate один раз**, cleanup многократно
- **Mock auth — TestAuthHandler** не реальный JWT для simplicity
- **Tests должны быть быстрыми** — <500ms each для integration
- **CI: docker layer caching** для Testcontainers images
- **Load tests — отдельный schedule**, не каждый PR

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
1. **AI первый pass** — Copilot Chat review кода
2. **Senior валидирует AI feedback** — отбрасывает false positives
3. **Senior фокусируется на architecture** — что AI не видит:
   - Domain logic correctness
   - Business rule violations
   - Performance implications
   - Security in context

**Time saved:** 30 min review → 10 min (AI на mechanical, senior на important).

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

- [[testing|Testing — общие principles]]
- [[mutation-load-testing|Mutation Testing — Stryker.NET]]
- [[arch-tests|Architecture Tests]]
- [[static-analysis|Static Analysis — quality gates]]
- [[basics-tracking|EF Core for repository tests]]

## Reading list

- **Microsoft Docs — Integration Tests** — learn.microsoft.com/aspnet/core/test/integration-tests
- **Testcontainers .NET docs** — testcontainers.com/getting-started/dotnet
- **WireMock.NET** — github.com/WireMock-Net/WireMock.Net
- **Verify by Simon Cropp** — github.com/VerifyTests/Verify
- **Andrew Lock — Testing in ASP.NET Core** — andrewlock.net
- **xUnit Test Patterns** — Gerard Meszaros (test design)
