---
tags: [testing, fundamentals, junior, types-of-tests, tdd, pyramid]
level: Junior
date: 2026-04-30
---

# Testing Fundamentals — что такое тестирование

> **Что такое тест, какие виды бывают, зачем каждый нужен, где применяется**. Базовые концепты для джуна, систематизация для middle/senior. Перед тем как писать тесты — понять зачем и какие.

---

## Что это, зачем и когда

### Что такое тест?

**Код, который проверяет другой код** — даёт ему вход, проверяет выход.

```csharp
// Production code
public int Add(int a, int b) => a + b;

// Test
[Fact]
public void Add_returns_sum()
{
    var result = Add(2, 3);
    Assert.Equal(5, result);
}
```

**Аналогия:** Контролёр на фабрике. Каждый виджет проходит проверку — соответствует спецификации? OK — на следующий конвейер. Не соответствует — bracket. Без контролёра — брак уезжает к клиенту.

### Зачем тестировать?

| Без тестов | С тестами |
|------------|-----------|
| Изменил код → "вроде работает" | Изменил → запустил тесты → точно знаешь |
| Refactoring страшно (что-то сломаешь) | Refactor свободно — тесты ловят regressions |
| Bug в production обнаруживается юзерами | До deploy ловится |
| Code review только глазами | Тесты — automated check |
| Каждый release — бессонная ночь "что упадёт?" | Confidence at deploy |
| 1 час фиксят bug → 1 день debug сессии в production | 5 минут локально |

### Cost of bug

```
Bug в production:                       $$$ (downtime, lost users, support time)
Bug перед production (QA нашёл):        $$
Bug перед PR merge (CI поймал):         $
Bug при разработке (test поймал сразу): $
```

Каждый шаг ближе к prod — стоимость **в 10x больше**.

### Когда (не) писать тесты

✅ **Всегда:**
- Domain logic / business rules
- Critical paths (payment, auth, security)
- Bug fixes (regression test)
- Public API (контракт)
- Сложные алгоритмы
- Edge cases — null, boundary, overflow

⚠️ **Возможно нет:**
- Throwaway scripts / experiments
- Простой UI binding (если есть E2E test потом)
- Generated code

❌ **Не нужно:**
- Properties (geters/setters)
- Framework code (это уже Microsoft протестировал)
- Trivial mappings (DTO → Entity)
- Тесты только "ради coverage"

---

## 1. Виды тестов

### Test Pyramid (Mike Cohn, 2009)

```
              /\
             /UI\          ← E2E / UI tests   ~5%
            /----\
           /Integ.\         ← Integration     ~25%
          /--------\
         /  Unit    \       ← Unit tests      ~70%
        /------------\
```

**Снизу вверх:**
- Объём убывает (меньше тестов)
- Скорость убывает (медленнее)
- Стабильность убывает (более flaky)
- Уверенность растёт (ближе к real prod)

### Unit Tests

**Тестируют один "юнит"** — обычно один класс / метод **в изоляции** (mocks для зависимостей).

```csharp
public class CalculatorTests
{
    [Fact]
    public void Add_two_positive_numbers()
    {
        var calc = new Calculator();
        var result = calc.Add(2, 3);
        result.Should().Be(5);
    }
    
    [Fact]
    public void Divide_by_zero_throws()
    {
        var calc = new Calculator();
        Action act = () => calc.Divide(10, 0);
        act.Should().Throw<DivideByZeroException>();
    }
}
```

**Характеристики:**
- Очень быстрые (<10ms каждый)
- Изолированы от внешнего мира (нет DB, HTTP, файловой системы)
- Стабильны (не flaky)
- Много тестов (70-80% от всех)

**Что тестировать:**
- Domain logic (расчёты, валидация, бизнес правила)
- Algorithms
- Pure functions
- State transitions

**Где compromise:**
- Если нужно много mocks — может быть **integration** test надо
- Если тестируешь "что код вызывает X" — это **implementation**, не **behavior** test

### Integration Tests

**Тестируют взаимодействие компонентов** — обычно несколько классов работают вместе, реальная DB, реальный HTTP.

```csharp
public class OrdersApiTests : IClassFixture<WebApplicationFactory<Program>>
{
    [Fact]
    public async Task POST_orders_creates_order_in_database()
    {
        var client = _factory.CreateClient();
        var response = await client.PostAsJsonAsync("/api/orders", new 
        {
            CustomerId = 1,
            Items = new[] { new { ProductId = 1, Quantity = 2 } }
        });
        
        response.EnsureSuccessStatusCode();
        var order = await response.Content.ReadFromJsonAsync<OrderDto>();
        order.Id.Should().BeGreaterThan(0);
        
        // Verify in DB
        var dbOrder = await _ctx.Orders.FindAsync(order.Id);
        dbOrder.Should().NotBeNull();
    }
}
```

**Характеристики:**
- Medium speed (10-100ms)
- Реальные зависимости (DB через Testcontainers)
- Больше confidence
- 20-25% от всех тестов

**Что тестировать:**
- API endpoints end-to-end
- DB interactions (EF Core queries)
- External service integration
- Auth/auth flow
- Cross-component logic

См. [[integration-testing|Integration Testing]].

### E2E (End-to-End) Tests

**Тестируют всю систему как пользователь** — UI + backend + DB.

```csharp
// Playwright / Selenium
[Fact]
public async Task User_can_create_order()
{
    var page = await browser.NewPageAsync();
    await page.GotoAsync("https://localhost:5001/login");
    
    await page.FillAsync("input[name=email]", "test@example.com");
    await page.FillAsync("input[name=password]", "password");
    await page.ClickAsync("button[type=submit]");
    
    await page.GotoAsync("/products");
    await page.ClickAsync(".add-to-cart");
    await page.ClickAsync(".checkout");
    
    await page.WaitForSelectorAsync(".order-success");
    var orderText = await page.TextContentAsync(".order-id");
    orderText.Should().Contain("Order #");
}
```

**Характеристики:**
- Slow (1-30 sec each)
- Flaky (timing, network)
- Высокая уверенность (как у реального юзера)
- 5% от тестов — небольшая доля

**Что тестировать:**
- Critical user journeys (signup, login, checkout)
- Cross-cutting concerns (auth → permissions → UI)
- Smoke tests перед deploy

**Tools:**
- **Playwright** — modern (Microsoft)
- Selenium — classic
- Cypress — JS-only

### Acceptance Tests / BDD

**Specification by example** — тесты пишут продакт менеджер / QA в "почти английском".

```gherkin
Feature: Order placement
  Scenario: Customer places order with valid credit card
    Given I am logged in as customer "John"
    And I have product "Widget" in cart with quantity 2
    When I checkout with credit card "4111-1111-1111-1111"
    Then the order should be created
    And I should receive email confirmation
```

```csharp
// SpecFlow связывает Gherkin с C# steps
[Given(@"I am logged in as customer ""(.*)""")]
public async Task GivenLoggedIn(string name) { /* ... */ }

[When(@"I checkout with credit card ""(.*)""")]
public async Task WhenCheckout(string card) { /* ... */ }

[Then(@"the order should be created")]
public async Task ThenOrderCreated() { /* ... */ }
```

**Когда использовать:**
- Сложная бизнес логика, которую важно зафиксировать с бизнесом
- Требования меняются часто — спецификации в коде
- QA / PM хотят писать тесты сами

> [!info] BDD — сложно правильно
> Без правильной культуры — превращается в "Gherkin по приколу", всё равно тесты пишут разработчики, бизнес не читает. Используй когда правда нужно.

### Performance / Load Tests

См. [[mutation-load-testing|Mutation & Load Testing]].

- **Load test** — нагрузка симулирует production traffic
- **Stress test** — нагружай до failure
- **Soak test** — длительная нагрузка (memory leaks)
- **Spike test** — резкий bump

### Security Tests

- **OWASP ZAP / Burp Suite** — auto-scanning
- **CodeQL** — static security analysis (см. [[static-analysis|Static Analysis]])
- **Penetration testing** — нанятые security профи пытаются взломать

### Smoke Tests

**Минимальный набор** что система запускается. Обычно после deploy.

```csharp
[Fact]
public async Task Health_endpoint_returns_200()
{
    var response = await client.GetAsync("/health");
    response.StatusCode.Should().Be(HttpStatusCode.OK);
}

[Fact]
public async Task Database_is_reachable()
{
    var response = await client.GetAsync("/health/database");
    response.StatusCode.Should().Be(HttpStatusCode.OK);
}
```

### Regression Tests

**Тесты на найденные баги** — чтобы не вернулись.

```csharp
// Bug ticket #1234: разделение на ноль возвращало 0 вместо exception
[Fact]
public void Divide_by_zero_throws_exception_regression_1234()
{
    var calc = new Calculator();
    Action act = () => calc.Divide(10, 0);
    act.Should().Throw<DivideByZeroException>();
}
```

После любого bug fix — добавь test чтобы baгa не вернулся silently.

### Snapshot Tests

См. [[integration-testing#snapshot-testing|Integration Testing — Snapshot Testing]].

Сохраняет ожидаемый output как файл, сравнивает на следующих запусках.

### Property-Based Tests

Вместо конкретных examples — генерируем сотни random inputs:

```csharp
// FsCheck / Hedgehog
[Property]
public Property Reverse_twice_returns_original(string input)
{
    return (input.Reverse().Reverse().SequenceEqual(input)).ToProperty();
}

// FsCheck сгенерирует 100+ random strings, проверит invariant
```

Хорошо для:
- Algorithms (sorting, encoding)
- Math operations
- Serialization roundtrips

### Mutation Tests

См. [[mutation-load-testing|Mutation & Load Testing]].

**Тестируют качество тестов** — меняют код, проверяют поймали ли тесты.

### Contract Tests

См. [[integration-testing#contract-testing-pact|Integration Testing — Pact]].

Между микросервисами — verify producer и consumer agreeing on API.

---

## 2. Test pyramid в реальности

### Идеал

```
70% Unit (быстрые, много)
25% Integration (medium)
5% E2E (медленные, мало)
```

### Реальные anti-patterns

#### Inverted pyramid (ice cream cone)

```
70% E2E (медленные!)
20% Integration
10% Unit
```

Проблема: тесты медленные, flaky. Build занимает 30 минут. CI падает random.

**Лечение:** push tests **down** — больше unit, меньше E2E.

#### Hourglass

```
50% E2E
~5% Integration
50% Unit
```

Гэп в integration — самый ценный layer пустой.

**Лечение:** добавь integration tests для API endpoints.

### Где какой тест

```
1. Domain logic (calculate price, validate order)
   → Unit (изолированный, быстрый)

2. API endpoint (POST /orders → DB → response)
   → Integration (WebApplicationFactory + Testcontainers)

3. User signup flow (UI → API → DB → Email)
   → E2E (Playwright)

4. Critical algorithm (sorting, encoding)
   → Property-based + Unit

5. Deploy verification
   → Smoke (health checks)

6. Performance under load
   → Load (NBomber / k6)
```

---

## 3. Хороший тест — характеристики (FIRST)

### F — Fast

Тест должен быть быстрым. Slow tests = разработчики не запускают локально → bugs in CI only.

```
Goal:
  Unit:        <10ms each
  Integration: <500ms each
  E2E:         <30s each
  
Total suite:
  Unit:        <30 sec
  Integration: <2 min
  E2E:         <10 min
```

### I — Independent / Isolated

Тесты не зависят друг от друга. Можно запускать в любом порядке, по одиночке, parallel.

```csharp
// ❌ Тест 1 создаёт user, тест 2 ожидает что user уже есть
[Fact]
public void Test1() { CreateUser(id: 1); }

[Fact]  
public void Test2() { var user = GetUser(id: 1); user.Should().NotBeNull(); }
// Если Test2 запустится первым — fail!

// ✅ Каждый тест — самодостаточен
[Fact]
public void Test2()
{
    CreateUser(id: 1);  // setup в самом тесте
    var user = GetUser(id: 1);
    user.Should().NotBeNull();
}
```

### R — Repeatable

Каждый запуск даёт тот же результат.

```csharp
// ❌ Зависит от текущего времени
[Fact]
public void GetCurrentDay_returns_today()
{
    var day = service.GetDay();
    day.Should().Be(DateTime.Today.DayOfWeek.ToString());
    // Test fails в субботу из-за edge case в коде → нашёл bug в субботу
}

// ✅ Контролируемое время
[Fact]
public void GetDay_returns_correct_day()
{
    var time = new FakeTimeProvider();
    time.SetUtcNow(DateTimeOffset.Parse("2024-01-15"));  // Monday
    var service = new MyService(time);
    
    var day = service.GetDay();
    day.Should().Be("Monday");
}
```

### S — Self-Validating

Тест либо проходит либо падает — без manual проверки output.

```csharp
// ❌ Тест "проверяет"
[Fact]
public void GenerateReport()
{
    var report = service.Generate();
    Console.WriteLine(report);  // ⚠️ Приходится самому смотреть!
}

// ✅ Assert
[Fact]
public void GenerateReport_includes_total()
{
    var report = service.Generate();
    report.Should().Contain("Total: $100");
}
```

### T — Timely (or Thorough)

Пиши тесты **во время** написания кода, не "потом":
- TDD — тесты first
- After feature — тесты до merge

Как только feature merged — тесты на неё **не пишутся** (никто не возвращается).

---

## 4. AAA Pattern (Arrange-Act-Assert)

```csharp
[Fact]
public void Subtract_returns_correct_difference()
{
    // Arrange — подготовь сцену
    var calc = new Calculator();
    int a = 10;
    int b = 3;
    
    // Act — выполни действие
    int result = calc.Subtract(a, b);
    
    // Assert — проверь
    result.Should().Be(7);
}
```

Три ясные фазы. Если две слиты — рефактор:

```csharp
// ❌ Mixed
[Fact]
public void Test()
{
    var result = new Calculator().Subtract(10, 3);
    result.Should().Be(7);
}

// ✅ AAA
[Fact]
public void Test()
{
    var calc = new Calculator();        // Arrange
    var result = calc.Subtract(10, 3);  // Act
    result.Should().Be(7);              // Assert
}
```

Альтернативное название: **Given-When-Then** (BDD style).

---

## 5. Naming тестов

### Pattern 1: `MethodName_Scenario_ExpectedBehavior`

```csharp
[Fact]
public void Add_TwoPositiveNumbers_ReturnsSum() { }

[Fact]
public void Divide_ByZero_ThrowsException() { }

[Fact]
public void GetUser_NonexistentId_ReturnsNull() { }
```

### Pattern 2: `Should_Behavior_When_Condition`

```csharp
[Fact]
public void Should_ReturnSum_When_AddingTwoPositiveNumbers() { }

[Fact]
public void Should_ThrowException_When_DividingByZero() { }
```

### Pattern 3: BDD-style

```csharp
[Fact]
public void Calculator_can_add_two_numbers() { }

[Fact]
public void Calculator_throws_when_dividing_by_zero() { }
```

**Любой** consistent style — OK. Главное:
- Тест name отвечает на "что тестируется и при каких условиях"
- Не нужно читать тело чтобы понять что тест делает

### Anti-pattern naming

```csharp
[Fact]
public void Test1() { }                    // ❌ Что Test1?
[Fact]
public void TestAdd() { }                  // ❌ Что про Add?
[Fact]
public void TestAddPositive() { }          // ⚠️ Лучше но что ожидаешь?
[Fact]
public void TestAddPositiveReturnsSum() { } // ✅ Понятно
```

---

## 6. Test data builders

Когда test setup сложный — используй builder.

```csharp
// Без builder — много boilerplate в каждом тесте
[Fact]
public void Test1()
{
    var customer = new Customer
    {
        Id = 1, Name = "John", Email = "john@example.com",
        Address = new Address { Street = "Main", City = "NYC" },
        IsActive = true, RegistrationDate = DateTime.UtcNow,
        // ... 10 more fields
    };
    var order = new Order
    {
        Customer = customer,
        Items = new[] { new OrderItem { ProductId = 1, Quantity = 1 } },
        // ...
    };
    
    // ... actual test logic
}

// С builder
public class CustomerBuilder
{
    private string _name = "Test User";
    private string _email = "test@example.com";
    private bool _isActive = true;
    
    public CustomerBuilder WithName(string name) { _name = name; return this; }
    public CustomerBuilder WithEmail(string email) { _email = email; return this; }
    public CustomerBuilder Inactive() { _isActive = false; return this; }
    
    public Customer Build() => new()
    {
        Id = Random.Shared.Next(),
        Name = _name,
        Email = _email,
        IsActive = _isActive
        // ... defaults для всего остального
    };
}

// Использование
[Fact]
public void Test1()
{
    var customer = new CustomerBuilder()
        .WithName("John")
        .WithEmail("john@example.com")
        .Build();
    
    // ... actual test logic, ясно что важно
}
```

Альтернатива — **Bogus** (см. [[integration-testing#bogus|Integration Testing — Bogus]]).

---

## 7. Test doubles — mocks/stubs/fakes/spies

### Stub

Возвращает hardcoded значение. "Just enough to make it work".

```csharp
public class StubUserRepo : IUserRepo
{
    public User? GetById(int id) => 
        id == 1 ? new User { Name = "Test" } : null;
}
```

### Mock

Verify что были сделаны конкретные вызовы.

```csharp
var mockRepo = Substitute.For<IUserRepo>();
mockRepo.GetById(1).Returns(new User { Name = "Test" });

var service = new UserService(mockRepo);
service.DoSomething(1);

// Verify
mockRepo.Received(1).GetById(1);
```

### Fake

Полная альтернативная implementation для тестов.

```csharp
public class InMemoryUserRepo : IUserRepo
{
    private readonly Dictionary<int, User> _users = new();
    
    public User? GetById(int id) => 
        _users.TryGetValue(id, out var u) ? u : null;
    
    public void Save(User user) => _users[user.Id] = user;
}
```

### Spy

Record вызовы для inspection.

```csharp
public class SpyEmailService : IEmailService
{
    public List<string> SentEmails { get; } = new();
    
    public Task SendAsync(string to, string body)
    {
        SentEmails.Add($"{to}: {body}");
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
        .Which.Should().Contain("user1@example.com");
}
```

### Когда что

| Тип | Когда |
|-----|-------|
| **Stub** | Простой подходит — нужно только указать return value |
| **Mock** | Verify взаимодействие (e.g., "audit log called") |
| **Fake** | Сложная заглушка (in-memory DB, in-memory cache) |
| **Spy** | Record + assert — middle ground |

См. [[mocking-strategies|Mocking Strategies]] для детального гайда.

---

## 8. TDD — Test-Driven Development

### Red-Green-Refactor cycle

```
1. RED — пиши test, он падает (feature не написана)
2. GREEN — пиши минимум кода чтобы test прошёл
3. REFACTOR — улучши код, тесты проходят

Repeat.
```

### Пример

```csharp
// 1. RED — пиши test
[Fact]
public void Add_returns_sum()
{
    var result = Calculator.Add(2, 3);
    result.Should().Be(5);
}
// Compile error — Calculator.Add не существует

// 2. GREEN — минимум кода
public static class Calculator
{
    public static int Add(int a, int b) => 5;  // hardcoded чтобы test прошёл!
}
// ✅ Test passes

// Add second test
[Fact]
public void Add_different_numbers()
{
    Calculator.Add(10, 20).Should().Be(30);
}
// RED — current implementation fails

// GREEN — обобщить
public static int Add(int a, int b) => a + b;

// REFACTOR — улучшить если есть что (sometimes nothing to refactor)
```

### Преимущества TDD

- Code is **testable by design** (small, decoupled)
- 100% test coverage automatically
- Минимум кода (YAGNI принцип на практике)
- Документация-by-tests
- Refactoring fearlessly

### Минусы TDD

- **Learning curve** — навык, не сразу
- **Overhead** — каждый шаг medium
- Не подходит для **research** / experimental code
- Junior часто tests становятся implementation tests, не behavior

### Когда TDD

✅ **Хорошо:**
- Domain logic / pure functions
- Algorithms
- Bug fixes (regression test first)

❌ **Сложнее применить:**
- UI (Blazor / MAUI components)
- Infrastructure code (DB, HTTP)
- Spike / прототип

---

## 9. BDD — Behavior-Driven Development

Расширение TDD с фокусом на **поведение системы как видит пользователь**.

```gherkin
Feature: User registration
  
  As a new user
  I want to register an account
  So that I can use the application
  
  Scenario: Successful registration
    Given the registration page is open
    When I fill in valid email and password
    And I submit the form
    Then I should see a confirmation email message
    And my account should be created in database
```

См. секцию **Acceptance Tests / BDD** выше. Tools: SpecFlow (для C#).

---

## 10. Testing strategy — что куда

```
┌─────────────────────────────────────────────────────┐
│ Domain logic (pure)                                 │
│ → Unit tests (xUnit + Shouldly)                     │
│ → Maybe property-based (FsCheck) для алгоритмов     │
└─────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────┐
│ Application services (с зависимостями)              │
│ → Unit tests (mocks для repositories, external)     │
│ → Integration tests для critical paths              │
└─────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────┐
│ Repositories / EF Core                              │
│ → Integration tests (Testcontainers + real DB)     │
│ → Unit tests редко (мокать DbContext = boilerplate) │
└─────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────┐
│ API endpoints                                       │
│ → Integration tests (WebApplicationFactory)         │
└─────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────┐
│ UI / SPA                                            │
│ → Component tests (если фреймворк позволяет)        │
│ → E2E tests для критичных flows (Playwright)        │
└─────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────┐
│ External services                                   │
│ → WireMock в integration tests                      │
│ → Contract tests (Pact) если microservices         │
└─────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────┐
│ Performance critical                                │
│ → BenchmarkDotNet для micro                         │
│ → NBomber / k6 для load tests                       │
└─────────────────────────────────────────────────────┘
```

---

## 11. Common Pitfalls

### 1. Тесты тестируют implementation, не behavior

```csharp
// ❌ Test ломается при любом refactoring
[Fact]
public void Method_calls_Repository_GetById_then_Save()
{
    var repo = Substitute.For<IRepo>();
    var service = new Service(repo);
    
    service.Method();
    
    Received.InOrder(() =>
    {
        repo.GetById(Arg.Any<int>());
        repo.Save(Arg.Any<Entity>());
    });
}
// Если refactor — поменял implementation, behavior тот же — test падает зря

// ✅ Test поведения
[Fact]
public void Method_updates_entity_in_database()
{
    var entity = new Entity();
    var repo = new InMemoryRepo();
    repo.Add(entity);
    var service = new Service(repo);
    
    service.Method(entity.Id);
    
    var updated = repo.GetById(entity.Id);
    updated.Status.Should().Be("processed");
}
```

### 2. Test theatre

```csharp
// ❌ Test ничего реально не проверяет
[Fact]
public void Test()
{
    var service = new Service();
    var result = service.Do();
    Assert.True(true);  // ⚠️ всегда passes!
}

// ❌ Слабый assert
[Fact]
public void Test2()
{
    var result = service.Do();
    result.Should().NotBeNull();  // ⚠️ только non-null проверяет
    // А если возвращает {Status: "Error"}? Тест passes!
}

// ✅ Specific
[Fact]
public void Test3()
{
    var result = service.Do();
    result.Should().NotBeNull();
    result.Status.Should().Be("Success");
    result.Value.Should().Be(42);
}
```

### 3. Over-mocking

```csharp
// ❌ Каждая зависимость замочена — тест ничего не проверяет
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
    
    dep1.Received().Method1();
    dep2.Received().Method2();
    // Проверяешь что замочен dep1 был вызван — ну и что?
}
```

**Лечение:**
- Если 5+ моков нужно — это integration test возможно
- Tested class имеет **слишком много зависимостей** — refactor, разбей

### 4. Snapshot testing для всего

```csharp
// ❌ Snapshot test на DTO — каждое изменение DTO ломает test
[Fact]
public Task UserDto_serialization() => Verify(user.ToJson());
// Add new field → snapshot test fails. Approve каждый раз. Useless.
```

**Лечение:** snapshot для **stable** structures (e.g., generated SQL, generated code, API contracts where you want explicit approval).

### 5. Testing the framework

```csharp
// ❌ Test framework code, не свой
[Fact]
public void Property_setter_works()
{
    var u = new User();
    u.Name = "John";
    u.Name.Should().Be("John");
}
// Конечно works! Это framework.
```

### 6. Slow tests никто не запускает

```
Test suite runs 5 minutes locally
→ Developers don't run before push
→ Bugs caught only in CI
→ 30 min PR cycle
```

**Лечение:**
- Profile тесты (which slow?)
- Push slow → integration / E2E only
- Parallel execution
- Only run affected tests на pre-commit

### 7. Coverage chasing

```
"100% coverage" → не значит "ВСЕ кейсы покрыты"
Coverage только показывает что **строка выполнена**, не что **проверена**
```

```csharp
// 100% coverage, но baga ловит?
public int Divide(int a, int b) => a / b;

[Fact]
public void Divide_works() => Divide(10, 2).Should().Be(5);
// Coverage: 100%. Но `Divide(10, 0)` throw — тест не проверяет!
```

**Лечение:** mutation testing (см. [[mutation-load-testing|Mutation Testing]]) — реально проверяет качество тестов.

---

## 12. Tools для C# / .NET

| Tool | Purpose |
|------|---------|
| **xUnit** | Test framework (default 2026) |
| **NUnit** | Test framework (mature alternative) |
| **MSTest** | Microsoft framework (least popular) |
| **Shouldly** | Assertions library (clear messages) |
| **FluentAssertions** | Assertions library (fluent API) |
| **NSubstitute** | Mocking library (clean syntax) |
| **Moq** | Mocking library (mature, but Moq 4.20 controversy) |
| **Bogus** | Fake test data generator |
| **AutoFixture** | Auto-create complex test objects |
| **Testcontainers** | Real DB / services in tests |
| **WireMock.NET** | Mock HTTP services |
| **Verify** | Snapshot testing |
| **FsCheck** | Property-based testing |
| **Stryker.NET** | Mutation testing |
| **NBomber** | Load testing in C# |
| **Playwright** | E2E browser automation |
| **SpecFlow** | BDD / Gherkin |
| **Respawn** | DB cleanup between tests |
| **NetArchTest** | Architecture rules |

---

## 13. Recommended setup для нового проекта

```xml
<!-- Test project csproj -->
<ItemGroup>
  <PackageReference Include="Microsoft.NET.Test.Sdk" />
  <PackageReference Include="xunit" />
  <PackageReference Include="xunit.runner.visualstudio" />
  <PackageReference Include="Shouldly" />
  <PackageReference Include="NSubstitute" />
  <PackageReference Include="Bogus" />
  <PackageReference Include="Testcontainers.PostgreSql" />
  <PackageReference Include="Microsoft.AspNetCore.Mvc.Testing" />
  <PackageReference Include="Respawn" />
</ItemGroup>
```

```csharp
// Структура solution
MyApp/
├── src/
│   ├── MyApp.Domain/
│   ├── MyApp.Application/
│   ├── MyApp.Infrastructure/
│   └── MyApp.Api/
└── tests/
    ├── MyApp.UnitTests/         # фронт unit-тестов
    ├── MyApp.IntegrationTests/  # WebApplicationFactory + Testcontainers
    ├── MyApp.ArchitectureTests/ # NetArchTest правила
    └── MyApp.E2ETests/          # Playwright (отдельный assembly)
```

---

## 14. Best Practices summary

- **Test pyramid** — 70 unit / 25 integration / 5 E2E
- **FIRST** — Fast, Independent, Repeatable, Self-validating, Timely
- **AAA pattern** — Arrange-Act-Assert
- **Behavior over implementation** — test что а не как
- **One concept per test** — один тест проверяет одну вещь
- **Descriptive names** — название говорит что и при каких условиях
- **No shared state** — каждый тест self-contained
- **No production data в тестах** — генерируй fresh / Bogus
- **Real DB лучше InMemory** — Testcontainers
- **TestContainers for external services** — Postgres, Redis, Kafka
- **WireMock for HTTP** — не реальный HTTP в тестах
- **Bogus for fake data** — не hardcode "John Doe"
- **Mock external dependencies** — но не over-mock
- **Tests должны быть быстрыми** — иначе никто не запускает
- **Coverage не самоцель** — quality > quantity

---

## 15. Когда писать какой тест — flowchart

```
Что надо протестировать?
│
├── Pure function / algorithm?
│   → Unit test (+ property-based если algorithm complex)
│
├── Domain entity / aggregate?
│   → Unit test (in-memory, без DB)
│
├── Repository / EF Core query?
│   → Integration test (Testcontainers + real DB)
│
├── API endpoint?
│   → Integration test (WebApplicationFactory)
│
├── External HTTP service integration?
│   → Integration test с WireMock
│
├── Critical user journey?
│   → E2E test (Playwright)
│
├── Bug fix?
│   → Regression test (level соответствующий месту бага)
│
├── Performance critical code?
│   → BenchmarkDotNet (micro) + NBomber (load)
│
└── Architecture rules?
    → NetArchTest
```

---

## См. также

- [[testing|Testing — practical xUnit]]
- [[integration-testing|Integration Testing]]
- [[mocking-strategies|Mocking Strategies]]
- [[mutation-load-testing|Mutation & Load Testing]]
- [[../Architecture/arch-tests|Architecture Tests]]
- [[../Quality/clean-code|Clean Code]]
- [[../LearningPath/02_junior-to-middle|Junior → Middle]] — testing phase

## Reading list

- **Test Driven Development: By Example** — Kent Beck (классика TDD)
- **The Art of Unit Testing** — Roy Osherove (.NET focus)
- **xUnit Test Patterns** — Gerard Meszaros (catalog of patterns)
- **Working Effectively with Legacy Code** — Michael Feathers (testing legacy)
- **Unit Testing Principles, Practices, and Patterns** — Vladimir Khorikov (modern!)
- **Specification by Example** — Gojko Adzic (BDD)
- **Microsoft Docs — Testing** — learn.microsoft.com/dotnet/core/testing
- **Andrew Lock — Testing series** — andrewlock.net
