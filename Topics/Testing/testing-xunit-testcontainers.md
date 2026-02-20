# Unit и Integration тесты

## Стек

**xUnit** — тестовый фреймворк. [Fact], [Theory], [InlineData]. IClassFixture для shared setup.

**NSubstitute** — моки. `var mock = Substitute.For<IService>()`, `mock.Get().Returns(value)`.

**FluentAssertions** — читаемые assertions. `result.Should().Be(expected)`, `list.Should().HaveCount(3)`.

---

## Unit тесты

Тестируют один класс в изоляции. Зависимости — моки. Быстрые, без I/O. Arrange — Act — Assert.

```csharp
[Fact]
public async Task GetUser_WhenExists_ReturnsUser()
{
    var repo = Substitute.For<IUserRepository>();
    repo.GetAsync(1, default).Returns(new User { Id = 1, Name = "Test" });
    var service = new UserService(repo);

    var result = await service.GetAsync(1);

    result.Should().NotBeNull();
    result!.Name.Should().Be("Test");
}
```

---

## Integration тесты

Тестируют несколько компонентов вместе. Реальная БД, HTTP. **Testcontainers** — PostgreSQL, Redis в Docker на время теста.

```csharp
public class ApiTests : IAsyncLifetime
{
    private readonly PostgreSqlContainer _db = new PostgreSqlBuilder().Build();

    public async Task InitializeAsync() => await _db.StartAsync();

    [Fact]
    public async Task GetUsers_ReturnsOk()
    {
        await using var app = await WebApplicationFactory
            .CreateFromFactory<Program>()
            .WithWebHostBuilder(b => b.UseSetting("ConnectionStrings:Default", _db.GetConnectionString()))
            .CreateAsync();
        var client = app.CreateClient();
        var response = await client.GetAsync("/api/users");
        response.StatusCode.Should().Be(HttpStatusCode.OK);
    }
}
```

---

## Best Practices

- Один assertion на тест (или логически связанные). Имя теста: Method_Scenario_Expected.
- Не тестировать фреймворк. Тестировать бизнес-логику и граничные случаи.
- ConfigureAwait(false) в библиотеках. CancellationToken в async-методах.

---

## См. также

- [[Architecture/architecture-tests-netarchtest|NetArchTest]]
- [[Topics/Docker/docker-deploy|Docker и CI/CD]]
- [[dotnet-knowledge-base|.NET Knowledge Base]]
