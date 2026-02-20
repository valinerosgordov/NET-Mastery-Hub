# 8) Тестирование, качество, релизы

## Пирамида/алмаз тестирования

### Классическая пирамида

Много unit-тестов (быстрые, изолированные, дешёвые). Меньше интеграционных (медленнее, реальные зависимости). Мало e2e (медленные, хрупкие, дорогие в поддержке).

### Алмаз (Diamond)

Акцент на интеграционных тестах критичных путей. Меньше unit (только сложная логика), больше интеграционных (реальные сценарии), мало e2e. Подход для backend, где ценность интеграционных тестов выше — проверка реального взаимодействия с БД, API.

---

## Unit vs интеграционные

### Unit

Тестируют изолированную логику. Зависимости — моки (NSubstitute, Moq). Быстро, детерминированно. Покрывают ветвления, граничные случаи. Не проверяют интеграцию с БД, сетью, файлами.

```csharp
public class OrderServiceTests
{
    private readonly IOrderRepository _repo = Substitute.For<IOrderRepository>();
    private readonly OrderService _sut;

    public OrderServiceTests() => _sut = new OrderService(_repo);

    [Fact]
    public async Task CreateOrder_ValidInput_ReturnsSuccess()
    {
        // Arrange
        _repo.AddAsync(Arg.Any<Order>()).Returns(Task.CompletedTask);

        // Act
        var result = await _sut.CreateAsync(new CreateOrderDto("Customer", 100m));

        // Assert
        result.IsSuccess.Should().BeTrue();
        await _repo.Received(1).AddAsync(Arg.Is<Order>(o => o.Total == 100m));
    }

    [Theory]
    [InlineData(0)]
    [InlineData(-1)]
    public async Task CreateOrder_InvalidTotal_ReturnsError(decimal total)
    {
        var result = await _sut.CreateAsync(new CreateOrderDto("Customer", total));

        result.IsSuccess.Should().BeFalse();
        result.Error.Should().Contain("Total");
    }
}
```

**Нюанс:** `Substitute.For<T>()` (NSubstitute) — проще синтаксис, чем Moq. `Arg.Is<T>(predicate)` — проверка аргументов. `Received(n)` — вызов произошёл n раз.

### Интеграционные

Тестируют взаимодействие компонентов. Реальная БД (Testcontainers), HTTP (WebApplicationFactory). Медленнее, могут быть нестабильны (внешние факторы). Покрывают критичные сценарии: «запрос пришёл → запрос к БД → ответ».

```csharp
public class OrdersApiTests : IClassFixture<WebApplicationFactory<Program>>
{
    private readonly HttpClient _client;

    public OrdersApiTests(WebApplicationFactory<Program> factory)
    {
        _client = factory.WithWebHostBuilder(builder =>
        {
            builder.ConfigureServices(services =>
            {
                // Заменяем реальную БД на Testcontainers PostgreSQL
                services.RemoveAll<DbContextOptions<AppDbContext>>();
                services.AddDbContext<AppDbContext>(opts =>
                    opts.UseNpgsql(TestDatabase.ConnectionString));
            });
        }).CreateClient();
    }

    [Fact]
    public async Task GetOrders_ReturnsOk()
    {
        var response = await _client.GetAsync("/api/orders");

        response.StatusCode.Should().Be(HttpStatusCode.OK);
        var orders = await response.Content.ReadFromJsonAsync<List<OrderDto>>();
        orders.Should().NotBeNull();
    }
}
```

**Нюанс:** `WebApplicationFactory<Program>` поднимает реальный ASP.NET pipeline в памяти. Тестируется полный путь: routing → middleware → controller → DB.

### Выбор

Unit — бизнес-правила, валидация, маппинг. Интеграционные — API endpoints, репозитории, оркестрация.

---

## Тестирование с БД и внешними API

### БД

**Testcontainers** — поднимает реальную БД (PostgreSQL, SQL Server) в Docker. Тесты выполняются против реального движка. Миграции применяются. Близко к prod, но медленнее и требует Docker.

**In-memory** (EF InMemory) — быстро, но не настоящая БД. Поведение может отличаться (нет constraints, другой SQL). Подходит для быстрых проверок, не для полного доверия.

### Внешние API

**Моки/заглушки** — полный контроль, быстро. Риск расхождения с реальным API.

**Контрактные тесты (Pact)** — проверка, что consumer и provider совместимы. Consumer тесты генерируют контракт; provider тесты проверяют соответствие.

**Sandbox** — реальное тестовое окружение провайдера. Ближе к prod, но зависимость от внешнего сервиса, может быть нестабильно.

### Ложная уверенность

Моки не отражают реальное поведение. Интеграционные тесты для критичных путей снижают риск. Баланс: unit для логики, интеграционные для границ.

---

## Backward compatibility контрактов

- **Pact** — контракты между consumer и provider. Изменение контракта ломает тесты.
- **OpenAPI diff** — сравнение схем. Детекция breaking changes.
- **Регрессионные тесты API** — старые клиенты против нового API. Набор запросов, ожидаемые ответы.
- **Версионирование** — старые версии API поддерживаются до deprecation.

---

## CI перед merge

- Сборка (restore, build).
- Unit-тесты.
- Интеграционные тесты (если есть, могут быть в отдельном job).
- Линтеры (StyleCop, analyzers).
- Анализ уязвимостей (NuGet audit, Snyk).
- Проверка контрактов (если используется).
- Code coverage (опционально, с осторожностью к гонке за процентами).

---

## Feature flags

Конфигурация (appsettings, база) или специализированные сервисы (LaunchDarkly, Unleash). Включение/выключение фич без деплоя. Постепенный rollout: сначала внутренние пользователи, потом процент трафика, потом все. Возможность отката без отката деплоя.

---

## Canary / blue-green для .NET

### Blue-green

Два идентичных окружения. Одно активно (blue), другое — для деплоя (green). Деплой в green, проверка, переключение трафика. Мгновенный откат — переключение обратно на blue.

### Canary

Часть трафика (например, 5%) направляется на новую версию. Метрики сравниваются. При проблемах — откат. Постепенное увеличение процента. Требует поддержки в load balancer или service mesh.

### .NET

Стандартный деплой — приложение останавливается, обновляется, запускается. Blue-green и canary — через инфраструктуру (Kubernetes, Azure, AWS), не специфичны для .NET.

---

## Definition of Done для backend

- Код написан, соответствует стандартам.
- Unit-тесты для новой логики.
- Интеграционные тесты для новых endpoints (если применимо).
- Документация API обновлена (OpenAPI, комментарии).
- Логирование и обработка ошибок.
- Code review пройден.
- CI зелёный.
- Нет известных критичных багов.

---

## Best Practices

- **Testcontainers** — для интеграционных тестов с БД. Реальная БД, не InMemory.
- **Моки** — только для внешних зависимостей. Не мокать то, что контролируете.
- **FluentAssertions** — читаемые assertion. `result.Should().BeEquivalentTo(expected)`.
