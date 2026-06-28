---
tags: [quality, refactoring, code-smells, technical-debt]
level: Middle to Senior
date: 2026-04-30
---

# Refactoring — улучшение существующего кода

> Систематические техники для улучшения кода **без изменения внешнего поведения**. Каталог refactorings, code smells, когда рефакторить, как делать безопасно. Closes Martin Fowler's classic with C#-specific examples.

---

## Что это, зачем и когда

### Что такое refactoring?

> **Изменение внутренней структуры кода без изменения наблюдаемого поведения.** — Martin Fowler

Ключевое: **поведение НЕ меняется**. Тесты, которые проходили до — должны проходить после.

**Аналогия:** Ремонт квартиры без сноса стен. Перекрашиваешь стены, переставляешь мебель, обновляешь проводку — но дом остаётся домом, в нём те же люди, те же комнаты.

### Когда рефакторить

✅ **Хорошие моменты:**
- Перед добавлением новой фичи (упрощаешь код, потом добавляешь)
- Когда баг найден — понять root cause, refactor для предотвращения класса багов
- Code review нашёл smell
- Перед meaningful изменением (don't refactor + add feature simultaneously)
- Если повторяется похожий код в 3+ местах

❌ **Плохие моменты:**
- "Большой rewrite" — никогда не работает
- За день до релиза
- Без тестов которые ловят regressions
- "Просто потому что мне не нравится" — без understanding

### Refactoring vs Rewriting

| Refactoring | Rewriting |
|-------------|-----------|
| Маленькие шаги, минуты-часы | Дни-месяцы |
| Тесты проходят на каждом шаге | Тесты могут лежать долго |
| Можно остановиться в любой момент | Полу-готово ≠ работающий код |
| Низкий риск | Очень высокий риск |
| **Always preferred** | Только в крайнем случае |

---

## 1. Тесты — фундамент refactoring

### Без тестов нельзя

```
Refactoring без тестов = играть в Jenga с закрытыми глазами
```

Если нет тестов:

1. **Сначала пиши тесты** на текущее поведение (characterization tests)
2. **Потом рефакторь**

```csharp
// 1. Что код делает сейчас? (черный ящик)
[Fact]
public void Calculate_returns_42_for_specific_input()
{
    var result = OldUglyMethod.Calculate(7, 3, "magic");
    result.Should().Be(42);  // Просто фиксируем что возвращает
}

// 2. Refactor
public class CleanCalculator
{
    public int Calculate(int a, int b, string mode) { ... }
}

// 3. Тест проходит → refactoring безопасный
```

### Test coverage перед refactoring

- Domain logic: 80%+ нужно
- Critical paths: 95%+ нужно
- UI: 50%+ OK (visual testing trickier)

См.[[integration-testing|Integration Testing]].

---

## 2. Базовые refactorings

### Rename (переименование)

```csharp
// До
public void Proc(int n)
{
    for (int i = 0; i < n; i++) { /* ... */ }
}

// После
public void ProcessOrders(int orderCount)
{
    for (int orderIndex = 0; orderIndex < orderCount; orderIndex++) { /* ... */ }
}
```

IDE делает это автоматически (F2 в VS / Rider). Безопасно когда нет string-based reflection.

### Extract Method (выделить метод)

Когда метод длинный — выделяй блоки в отдельные методы:

```csharp
// До — один длинный метод
public void PrintReport(Order order)
{
    // Header
    Console.WriteLine("=========");
    Console.WriteLine($"Order #{order.Id}");
    Console.WriteLine($"Date: {order.Date}");
    Console.WriteLine("=========");
    
    // Items
    foreach (var item in order.Items)
    {
        Console.WriteLine($"  {item.Name}: ${item.Price} x {item.Quantity}");
    }
    
    // Total
    var total = order.Items.Sum(i => i.Price * i.Quantity);
    var tax = total * 0.08m;
    Console.WriteLine($"Subtotal: ${total}");
    Console.WriteLine($"Tax: ${tax}");
    Console.WriteLine($"Total: ${total + tax}");
}

// После
public void PrintReport(Order order)
{
    PrintHeader(order);
    PrintItems(order.Items);
    PrintTotals(order);
}

private void PrintHeader(Order order)
{
    Console.WriteLine("=========");
    Console.WriteLine($"Order #{order.Id}");
    Console.WriteLine($"Date: {order.Date}");
    Console.WriteLine("=========");
}

private void PrintItems(IEnumerable<Item> items)
{
    foreach (var item in items)
        Console.WriteLine($"  {item.Name}: ${item.Price} x {item.Quantity}");
}

private void PrintTotals(Order order)
{
    var subtotal = order.Items.Sum(i => i.Price * i.Quantity);
    var tax = subtotal * TaxRate;
    Console.WriteLine($"Subtotal: ${subtotal}");
    Console.WriteLine($"Tax: ${tax}");
    Console.WriteLine($"Total: ${subtotal + tax}");
}

private const decimal TaxRate = 0.08m;
```

### Inline Method (встроить метод)

Если метод тривиальный и используется в одном месте:

```csharp
// До
public bool MoreThanFiveLateDeliveries(Driver driver) =>
    driver.NumberOfLateDeliveries > 5;

public int GetRating(Driver driver) =>
    MoreThanFiveLateDeliveries(driver) ? 2 : 1;

// После — inline
public int GetRating(Driver driver) =>
    driver.NumberOfLateDeliveries > 5 ? 2 : 1;
```

> [!info] Когда extract vs inline
> Extract — когда блок имеет смысл и переиспользуется или поясняет намерение.
> Inline — когда метод не помогает понять (тривиальный) и используется один раз.

### Replace Magic Number with Named Constant

```csharp
// До
public double PotentialEnergy(double mass, double height) =>
    mass * 9.81 * height;

// После
private const double GravityConstant = 9.81;

public double PotentialEnergy(double mass, double height) =>
    mass * GravityConstant * height;
```

### Replace Conditional with Polymorphism

```csharp
// До — switch / if-else цепи
public class Bird
{
    public string Type { get; set; }
    
    public double Speed() => Type switch
    {
        "European" => 35,
        "African" => 40 - 5 * NumberOfCoconuts,
        "NorwegianBlue" => IsNailed ? 0 : 10 + Voltage / 10,
        _ => 0
    };
    
    public int NumberOfCoconuts { get; set; }
    public bool IsNailed { get; set; }
    public double Voltage { get; set; }
}

// После — полиморфизм
public abstract record Bird
{
    public abstract double Speed { get; }
}

public record EuropeanBird : Bird
{
    public override double Speed => 35;
}

public record AfricanBird(int NumberOfCoconuts) : Bird
{
    public override double Speed => 40 - 5 * NumberOfCoconuts;
}

public record NorwegianBlueBird(bool IsNailed, double Voltage) : Bird
{
    public override double Speed => IsNailed ? 0 : 10 + Voltage / 10;
}

// Альтернатива в C# — pattern matching на sealed types (более functional)
public abstract record Bird;
public record European : Bird;
public record African(int Coconuts) : Bird;
public record NorwegianBlue(bool IsNailed, double Voltage) : Bird;

public static double Speed(Bird b) => b switch
{
    European => 35,
    African a => 40 - 5 * a.Coconuts,
    NorwegianBlue n => n.IsNailed ? 0 : 10 + n.Voltage / 10,
    _ => 0
};
```

### Replace Type Code with Subclasses

См. предыдущий пример.

### Move Method (переместить метод в подходящий класс)

```csharp
// До — Account "знает слишком много" о Customer
public class Account
{
    public Customer Customer { get; set; }
    public decimal Balance { get; set; }
    
    // Метод про customer'а лучше в Customer
    public bool CustomerEligibleForDiscount() =>
        Customer.Years > 5 && Customer.Tier == "Gold";
}

// После
public class Customer
{
    public int Years { get; set; }
    public string Tier { get; set; }
    
    public bool EligibleForDiscount() => Years > 5 && Tier == "Gold";
}

public class Account
{
    public Customer Customer { get; set; }
    public decimal Balance { get; set; }
}

// Caller
if (account.Customer.EligibleForDiscount()) { ... }
```

### Extract Class

Класс делает слишком много — раздели:

```csharp
// До
public class Person
{
    public string Name { get; set; }
    public string Email { get; set; }
    
    // Address fields — too much for Person
    public string Street { get; set; }
    public string City { get; set; }
    public string Country { get; set; }
    public string PostalCode { get; set; }
    
    public string GetFullAddress() => $"{Street}, {City}, {Country}";
}

// После
public class Person
{
    public string Name { get; set; }
    public string Email { get; set; }
    public Address Address { get; set; }
}

public class Address
{
    public string Street { get; set; }
    public string City { get; set; }
    public string Country { get; set; }
    public string PostalCode { get; set; }
    
    public string GetFullAddress() => $"{Street}, {City}, {Country}";
}
```

### Introduce Parameter Object

```csharp
// До — много параметров
public IEnumerable<Order> SearchOrders(
    DateTime? from,
    DateTime? to,
    decimal? minAmount,
    decimal? maxAmount,
    string? customerName,
    string? status)
{ ... }

// После
public record OrderSearchCriteria(
    DateTime? From = null,
    DateTime? To = null,
    decimal? MinAmount = null,
    decimal? MaxAmount = null,
    string? CustomerName = null,
    string? Status = null);

public IEnumerable<Order> Search(OrderSearchCriteria criteria) { ... }
```

### Replace Loop with LINQ

```csharp
// До
var activeUsers = new List<User>();
foreach (var user in users)
{
    if (user.IsActive)
        activeUsers.Add(user);
}

// После
var activeUsers = users.Where(u => u.IsActive).ToList();
```

### Replace Null with Null Object Pattern

```csharp
// До
public Customer? GetCustomer(int id) =>
    customers.FirstOrDefault(c => c.Id == id);

// Caller везде:
var customer = GetCustomer(id);
var name = customer?.Name ?? "Guest";
var discount = customer?.Discount ?? 0;
var canShip = customer?.CanShip() ?? false;

// После — Null Object
public class NullCustomer : Customer
{
    public override string Name => "Guest";
    public override decimal Discount => 0;
    public override bool CanShip() => false;
}

public Customer GetCustomer(int id) =>
    customers.FirstOrDefault(c => c.Id == id) ?? new NullCustomer();

// Caller проще:
var customer = GetCustomer(id);
var name = customer.Name;
var discount = customer.Discount;
var canShip = customer.CanShip();
```

> [!info] Trade-off
> Null Object скрывает что customer "не существует". Иногда хочешь явно — Result\<Customer, NotFound\> лучше.

---

## 3. Code Smells (запахи плохого кода)

Smell = подсказка что **возможно** нужен refactoring.

### Code Smells каталог

#### 1. Duplicate Code

Самый очевидный smell. Решение: Extract Method, Extract Class.

#### 2. Long Method (>30-50 строк)

Решение: Extract Method.

#### 3. Large Class (>500 строк)

Решение: Extract Class по responsibility.

#### 4. Long Parameter List (>3-4)

Решение: Introduce Parameter Object.

#### 5. Divergent Change

Один класс меняется по разным причинам — нарушение SRP.

```csharp
// Smell: класс меняется когда:
// - меняется бизнес правило для скидок
// - меняется формат email
// - меняется БД
public class CustomerService
{
    public decimal CalculateDiscount(Customer c) { ... }   // меняется когда меняется бизнес
    public string FormatEmail(Customer c) { ... }           // меняется когда меняется email формат
    public void Save(Customer c) { ... }                    // меняется когда меняется БД
}
```

Решение: Extract Class.

#### 6. Shotgun Surgery

Изменение одного бизнес правила требует менять 10 классов. Противоположность Divergent Change.

Решение: Move Method/Field — собрать связанные изменения в один класс.

#### 7. Feature Envy

Метод одного класса слишком много обращается к другому. См. секцию refactorings выше.

#### 8. Data Clumps

Группа полей всегда вместе появляются:

```csharp
public void CreateOrder(string street, string city, string country, string postal) { ... }
public void ShipTo(string street, string city, string country, string postal) { ... }
public Address GetAddress(string street, string city, string country, string postal) { ... }
```

Решение: Extract Class — `Address`.

#### 9. Primitive Obsession

```csharp
// Smell: string для всего
public void Send(string email, string phone, string country) { ... }

// Лучше
public void Send(Email email, PhoneNumber phone, Country country) { ... }
```

#### 10. Switch Statements (если повторяется тот же switch)

См. Replace Conditional with Polymorphism.

#### 11. Lazy Class

Класс который ничего не делает — удалить, inline.

#### 12. Speculative Generality

```csharp
// Smell: интерфейс с одним implementation, "на будущее"
public interface IUserService
{
    User GetUser(int id);
}

public class UserService : IUserService
{
    public User GetUser(int id) { ... }
}
```

YAGNI — удали интерфейс пока нет реальной нужды.

#### 13. Comments как deodorant

```csharp
// Smell: комментарий "пахнет" неясным кодом
// Check if user is admin and active and from US
if ((u.r & 0x01) > 0 && u.s == 1 && u.c == "US") { ... }
```

Решение: rename, extract method:

```csharp
if (user.IsAdmin && user.IsActive && user.IsFromUs) { ... }
```

#### 14. Inappropriate Intimacy

Два класса слишком много друг про друга знают.

#### 15. Message Chains (Law of Demeter violations)

```csharp
// Smell — chain через несколько объектов
var name = customer.GetAddress().GetCity().GetName();
```

Решение: Hide Delegate (помести метод в `customer`):

```csharp
public class Customer
{
    public string CityName => Address.City.Name;
}

var name = customer.CityName;
```

---

## 4. Refactoring catalog (быстрая справка)

### Composing Methods
- Extract Method
- Inline Method
- Extract Variable
- Inline Variable
- Replace Temp with Query

### Moving Features Between Objects
- Move Method
- Move Field
- Extract Class
- Inline Class
- Hide Delegate
- Remove Middle Man

### Organizing Data
- Replace Magic Number with Named Constant
- Encapsulate Variable
- Replace Primitive with Object
- Replace Type Code with Class

### Simplifying Conditional Expressions
- Decompose Conditional
- Consolidate Conditional Expression
- Replace Conditional with Polymorphism
- Introduce Null Object
- Introduce Assertion (guard clauses)

### Simplifying Method Calls
- Rename Method
- Add/Remove Parameter
- Separate Query from Modifier
- Parameterize Method
- Replace Parameter with Method
- Introduce Parameter Object

### Generalization
- Pull Up Method/Field
- Push Down Method/Field
- Extract Superclass
- Extract Interface
- Replace Inheritance with Delegation

Полный каталог: [refactoring.com/catalog](https://refactoring.com/catalog).

---

## 5. Process — как рефакторить безопасно

### Шаг 1: Identify smell

Заметил code smell — записал в TODO список. Не лезь сразу.

### Шаг 2: Cover with tests

Прежде чем менять — убедись что тесты ловят regression:

```csharp
// 1. Запускай тесты ДО refactoring — должны проходить
dotnet test  // ✅ All passing

// 2. Если coverage низкий — добавь characterization tests
// (тесты на текущее behavior, не идеальное)

// 3. Снова запусти — все passing
```

### Шаг 3: Маленькие шаги

Делай **минимальное** изменение, потом тесты, потом следующее.

```
Шаг 1: Renamed variable → tests run → ✅ commit
Шаг 2: Extracted method → tests run → ✅ commit  
Шаг 3: Moved method to other class → tests run → ✅ commit
Шаг 4: Removed dead code → tests run → ✅ commit
```

Если тесты упали — easy revert (один маленький commit).

### Шаг 4: Commit часто

> Commit early, commit often.

Каждый рабочий step — commit. Большие refactoring → много commit. PR может быть squash на merge если важна одна "feature unit".

### Шаг 5: Remote backup

PR / branch на remote — на случай если local lost.

### Шаг 6: Stop, если запутался

Если refactoring растянулся, тесты падают, не понимаешь что происходит:

1. **Stop**
2. **Revert to last working commit**
3. Подумай — может план неправильный
4. Начни заново с другим подходом

---

## 6. Refactoring и тесты — golden loop

```
1. Run tests           → all green
2. Make small change   → 
3. Run tests           → green? continue. red? revert.
4. Commit if green
5. Repeat
```

Это называется **TDD-like refactoring loop**. Маленькие итерации, всегда работающее состояние.

---

## 7. IDE refactoring tools

### Visual Studio / Rider

Большинство refactorings — встроены, **используй их**:

| Action | Shortcut (VS) | Shortcut (Rider) |
|--------|---------------|------------------|
| Rename | `F2` | `Shift+F6` |
| Extract Method | `Ctrl+R, M` | `Ctrl+Alt+M` |
| Extract Variable | `Ctrl+R, I` | `Ctrl+Alt+V` |
| Inline | `Ctrl+R, I` | `Ctrl+Alt+N` |
| Move Type to File | — | `F6` |
| Quick Actions | `Ctrl+.` | `Alt+Enter` |

IDE refactorings **гарантированно safe** для C# (понимают типы).

---

## 8. Technical Debt

### Что это

> Шорткат который ускоряет сейчас, но делает всё медленнее в будущем. — Ward Cunningham

Метафора: **долг**. Берёшь сейчас (быстро запустил), платишь проценты потом (баги, медленные изменения).

### Виды tech debt

| Тип | Что |
|-----|-----|
| **Deliberate, prudent** | "Знаем как правильно — но time-to-market важнее. Запишем что вернёмся." |
| **Deliberate, reckless** | "Не разбираемся, лепим как-нибудь" |
| **Inadvertent, prudent** | "Думали правильно, оказалось — не идеально. Узнали постфактум." |
| **Inadvertent, reckless** | "Не знали что делали" |

Только первый тип — **acceptable**. Остальные — проблемы.

### Tracking debt

```
1. Создай Tech Debt доску (Trello / Jira / GitHub Issues)
2. Когда замечаешь debt — пиши в неё
3. Раз в спринт — выделяй % времени (10-20%) на работу с debt
4. Приоритизируй: что ломает чаще / больно меняется
```

### Когда платить

✅ **Плати:**
- Перед major feature touching this code
- Когда debt стал блокером (нельзя добавить feature)
- Регулярно понемногу (boy scout rule)

❌ **Не плати:**
- Если код не трогается (пусть лежит)
- Прямо перед release
- "Big rewrite" — almost always fails

См.[[03_middle-to-senior|Middle → Senior]] — Senior softskill умение приоритизировать debt.

---

## 9. Common Pitfalls

### 1. "Большой refactoring" branch на 2 недели

```
Git history:
  feature/big-refactoring  ← 2 недели работы
  main                      ← merged 50 PRs пока ты рефакторил
```

При merge — конфликты везде. Тесты лежат. Ничего не работает.

**Лечение:** маленькими PRs в main, постоянно.

### 2. Refactoring + новая feature в одном PR

```
PR title: "Add notification feature + refactor user service"
```

Reviewer не понимает что review — что работает, что новое.

**Лечение:** один PR — одна цель. Refactor отдельно, feature отдельно.

### 3. Refactoring без тестов

```csharp
// Никаких тестов. Refactor. Что сломалось?
```

**Лечение:** characterization tests первыми.

### 4. Перфекционизм

"Я сделаю идеальный код, на это потребуется 6 месяцев." Никто не платит за refactoring 6 месяцев — release требуется.

**Лечение:** маленькие улучшения в ходе работы (boy scout). 80/20 rule — 80% улучшения за 20% времени.

### 5. "Это работает, не трогай"

```csharp
// "Magic" function — никто не понимает, но work
public string Process(string s)
{
    var x = s.ToCharArray().GroupBy(c => c).OrderBy(g => g.Count()).First().Key;
    // ... 50 lines of cryptic code ...
}
```

Если это hot path / часто меняется — рефакторь сейчас, до того как сломается под чужими руками.

### 6. Refactoring при unstable tests

Если тесты flaky — сначала зафиксируй их. Refactoring тогда покажет надёжный сигнал.

---

## 10. Best Practices

- **Тесты first** — characterization tests если нет других
- **Маленькие шаги** — один refactoring за раз
- **Тесты после каждого шага**
- **Commit часто** — easy revert
- **IDE refactorings** — safer than manual
- **Не смешивай refactoring с feature**
- **Boy scout rule** — улучшай по чуть-чуть постоянно
- **Не делай "большой rewrite"** — почти всегда проваливается
- **Отслеживай tech debt** — Trello / GitHub Issues
- **Pair programming** на сложный refactoring
- **Document why** — ADR для больших архитектурных refactoring

---

## 11. Когда NOT to refactor

- **Code не трогается** — пусть остаётся как есть
- **Перед deadline** — рискованно
- **Если не понимаешь код** — сначала разберись
- **Без тестов и нет времени их добавить** — отложи
- **"Just because"** — нужна reason

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

- [[clean-code|Clean Code]] — fundamentals
- [[code-quality|Code Quality]] — quality gates
- [[code-review|Code Review]] — как находить smells в чужом коде
- [[static-analysis|Static Analysis]] — автоматическое обнаружение
-[[solid|SOLID]] — принципы которые помогают rеfactor
-[[integration-testing|Integration Testing]] — characterization tests
-[[design-patterns|Design Patterns]] — целевая структура

## Reading list

- **Refactoring: Improving the Design of Existing Code** — Martin Fowler (классика, 2nd ed. в JS)
- **Working Effectively with Legacy Code** — Michael Feathers (как refactor без tests)
- **Refactoring to Patterns** — Joshua Kerievsky
- **Refactoring.com** — refactoring.com (Martin Fowler's catalog)
- **Code Complete** — Steve McConnell
- **Tidy First?** — Kent Beck (модерн взгляд на refactoring)
- **A Philosophy of Software Design** — John Ousterhout
