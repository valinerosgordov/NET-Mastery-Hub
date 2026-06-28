---
tags: [quality, clean-code, naming, principles, junior, fundamentals]
level: Junior
date: 2026-04-30
---

# Clean Code — чистый код

> **Что такое читаемый код, как его писать, и почему это самая недооценённая навык программиста**. Базовые принципы плюс практические примеры. Применимо к любому языку, фокус на C#.

---

## Что это, зачем и когда

### Что такое "чистый код"?

**Код, который легко читать, понимать и менять**. Не самый "умный" код, не самый короткий — самый понятный для **человека, который его читает через полгода** (этим человеком чаще всего будешь ты сам).

**Аналогия:** Книга. Можешь написать роман предложениями по 200 слов с непонятными метафорами — формально это книга. Но никто не дочитает. Хорошая книга — простые предложения, ясные мысли. Так же с кодом.

### Зачем чистый код?

| "Грязный" код | Чистый код |
|----------------|-----------|
| Один баг в месяц правится 3 дня | Правится за час |
| Новый разработчик 2 недели входит в проект | 2 дня |
| "Не трогай этот класс — никто не знает как работает" | Любой может изменить |
| Каждое изменение ломает что-то ещё | Изолированные изменения |
| Ты сам через 6 месяцев не помнишь что писал | Понятно через 5 лет |

### Главное правило

> **Код пишется один раз, а читается сотни раз.**
> Оптимизируй для **чтения**, не для написания.

---

## 1. Имена — половина чистого кода

### Хорошие имена показывают **намерение**

```csharp
// ❌ Что это? Что считает? Зачем?
int d;        // elapsed time in days
List<int[]> l;
bool flag;

// ✅ Намерение очевидно
int elapsedDays;
List<int[]> activeUsersByMonth;
bool isUserActive;
```

### Не используй misleading имена

```csharp
// ❌ "list" предполагает List<T>
HashSet<User> userList;       // на самом деле — HashSet
Account[] accountList;          // на самом деле — массив

// ❌ "Customer" но содержит User?
class Customer
{
    public User UserData { get; set; }  // Confusing
}

// ✅
HashSet<User> activeUsers;
Account[] accounts;
```

### Используй произносимые имена

```csharp
// ❌ Не произнесёшь даже сам
int genymdhms;   // generation date, year, month, day, hour, minute, second
string XYZ_TXT;

// ✅
DateTime generatedAt;
string licenseText;
```

### Различимые имена

```csharp
// ❌ Что отличает их? a1, a2 что значит?
public void Copy(char a1[], char a2[])
{
    for (int i = 0; i < a1.Length; i++)
        a2[i] = a1[i];
}

// ✅
public void Copy(char[] source, char[] destination)
{
    for (int i = 0; i < source.Length; i++)
        destination[i] = source[i];
}
```

### Размер имени должен соответствовать scope

```csharp
// ✅ Короткое для маленького scope (loop variable)
for (int i = 0; i < items.Count; i++) { ... }

// ✅ Длинное для большого scope (класс, public method)
public class CustomerOrderProcessingService { ... }

// ❌ Огромное имя для loop variable
for (int currentIterationIndex = 0; ...) { ... }

// ❌ Короткое для класса
public class C { ... }
```

### Соглашения по типу

```csharp
// Class — существительное (что это?)
class User
class OrderProcessor
class HttpClient

// Method — глагол (что делает?)
void SendEmail()
bool IsValid()
User GetById(int id)

// Boolean — вопрос (Is/Has/Can)
bool isActive
bool hasPermission
bool canRead

// Constant — UPPER_CASE или PascalCase в C#
const int MAX_USERS = 100;
const string DefaultEmail = "noreply@example.com";

// Interface — I prefix в C# (convention)
interface IUserRepository { }
interface ILogger { }

// Async method — Async suffix
Task<User> GetUserAsync(int id);
```

### Не используй Hungarian notation

```csharp
// ❌ Hungarian — устарело, IDE показывает тип
string strName;
int iCount;
bool bIsActive;

// ✅ В современном C#
string name;
int count;
bool isActive;
```

---

## 2. Функции — маленькие и однозначные

### Правило 1: Маленькие

> **Функция должна быть маленькой. Потом ещё меньше.** — Robert C. Martin

Идеал — 5-15 строк. Метод из 100 строк почти всегда можно разбить.

```csharp
// ❌ Слишком много делает
public void ProcessOrder(Order order)
{
    // Validate
    if (order == null) throw new ArgumentNullException();
    if (order.Items.Count == 0) throw new InvalidOperationException("Empty");
    foreach (var item in order.Items)
    {
        if (item.Quantity <= 0) throw new InvalidOperationException("Invalid quantity");
    }
    
    // Calculate total
    decimal total = 0;
    foreach (var item in order.Items)
    {
        total += item.Price * item.Quantity;
        if (item.HasDiscount)
            total -= item.DiscountAmount;
    }
    
    // Apply tax
    decimal taxRate = order.Country == "US" ? 0.08m : 0.20m;
    total *= (1 + taxRate);
    
    // Save
    using var conn = new SqlConnection(connStr);
    conn.Open();
    var cmd = new SqlCommand("INSERT INTO Orders ...", conn);
    cmd.ExecuteNonQuery();
    
    // Send email
    var smtp = new SmtpClient("smtp.example.com");
    smtp.Send(new MailMessage(...));
}

// ✅ Разбить на маленькие функции с понятными именами
public void ProcessOrder(Order order)
{
    ValidateOrder(order);
    var total = CalculateTotal(order);
    SaveOrder(order, total);
    NotifyCustomer(order);
}

private void ValidateOrder(Order order)
{
    if (order is null) throw new ArgumentNullException(nameof(order));
    if (order.Items.Count == 0) throw new InvalidOperationException("Empty order");
    
    foreach (var item in order.Items)
    {
        if (item.Quantity <= 0)
            throw new InvalidOperationException($"Invalid quantity for {item.ProductId}");
    }
}

private decimal CalculateTotal(Order order)
{
    var subtotal = order.Items.Sum(item => item.Price * item.Quantity - item.DiscountAmount);
    var taxRate = GetTaxRate(order.Country);
    return subtotal * (1 + taxRate);
}

private decimal GetTaxRate(string country) => country switch
{
    "US" => 0.08m,
    _ => 0.20m
};
```

Обрати внимание:
- Каждая функция читается **как абзац** на естественном языке
- Имена методов рассказывают что делает
- Главный метод — high-level steps, детали в helpers

### Правило 2: Делает одну вещь

> Функция должна делать **одну вещь**, делать её **хорошо** и делать только её.

Как понять что функция делает одно? Если можешь её описать без союза "и" / "затем":
- ❌ "Validate order **and** calculate total"
- ✅ "Validate order"
- ✅ "Calculate total"

### Правило 3: Один уровень абстракции

```csharp
// ❌ Mixing high-level (HTML) and low-level (string operations)
public void RenderPage()
{
    var html = "<html><body>";
    html += "<h1>" + GetTitle().Trim().ToUpper() + "</h1>";
    html += "<div class='content'>";
    foreach (var item in items)
    {
        html += "<p>" + item.Name.Replace("&", "&amp;") + "</p>";
    }
    html += "</div></body></html>";
    File.WriteAllText("page.html", html);
}

// ✅ Каждая функция — один уровень
public void RenderPage()
{
    var html = BuildHtml();
    SaveToFile(html, "page.html");
}

private string BuildHtml() =>
    $"<html><body>{BuildHeader()}{BuildContent()}</body></html>";

private string BuildHeader() => 
    $"<h1>{GetSafeTitle()}</h1>";

private string BuildContent() => 
    $"<div class='content'>{string.Concat(items.Select(BuildItem))}</div>";

private string BuildItem(Item item) => 
    $"<p>{Escape(item.Name)}</p>";
```

### Правило 4: Минимум аргументов

```csharp
// ❌ 6 аргументов — сложно использовать, легко перепутать порядок
public void CreateUser(string name, string email, int age, string country, string city, bool isActive) { }

// Вызов — easy to mess up:
CreateUser("John", "john@example.com", 30, "US", "NYC", true);
// А может email и name перепутали? Не видно.

// ✅ Object parameter (DTO / Record)
public record CreateUserRequest(
    string Name,
    string Email,
    int Age,
    string Country,
    string City,
    bool IsActive);

public void CreateUser(CreateUserRequest request) { }

// Вызов — named, понятно
CreateUser(new CreateUserRequest(
    Name: "John",
    Email: "john@example.com",
    Age: 30,
    Country: "US",
    City: "NYC",
    IsActive: true
));
```

| Аргументов | Оценка |
|-----------|--------|
| 0 | Идеально |
| 1 | Очень хорошо |
| 2 | Хорошо |
| 3 | OK |
| 4+ | Перебор — рассмотри parameter object |

### Правило 5: Без побочных эффектов

```csharp
// ❌ Функция "проверки пароля" ещё и инициализирует session — сюрприз!
public bool CheckPassword(string username, string password)
{
    var user = FindUser(username);
    if (user is null) return false;
    
    if (Cryptographer.Verify(password, user.PasswordHash))
    {
        Session.Initialize(user);  // ← скрытый side effect!
        return true;
    }
    return false;
}

// ✅ Функция делает что говорит. Side effect отдельно.
public bool CheckPassword(string username, string password)
{
    var user = FindUser(username);
    return user is not null && Cryptographer.Verify(password, user.PasswordHash);
}

// Caller:
if (CheckPassword(username, password))
{
    Session.Initialize(user);
}
```

### Правило 6: Не возвращай null если можно

```csharp
// ❌ Caller должен проверять null постоянно
public List<Order> GetOrdersByUser(int userId)
{
    if (userId < 0) return null;  // ⚠️
    // ... 
}

var orders = GetOrdersByUser(id);
foreach (var order in orders) { ... }  // 💥 NullReferenceException

// ✅ Возвращай empty collection
public List<Order> GetOrdersByUser(int userId)
{
    if (userId < 0) return [];  // empty list
    // ...
}

// Caller безопасно:
var orders = GetOrdersByUser(id);
foreach (var order in orders) { ... }  // OK даже если empty
```

См.[[error-handling|Error Handling]] для Result\<T,E\> patterns.

---

## 3. Комментарии — последнее средство

### Хороший код **сам себя объясняет**

```csharp
// ❌ Комментарий объясняет что делает код — переименуй переменные!
// Check if employee is eligible for benefits
if (employee.flags & 0x01 == 0x01 && employee.age > 65) { ... }

// ✅ Код читается без комментария
if (employee.IsEligibleForBenefits) { ... }

// Внутри property:
public bool IsEligibleForBenefits =>
    HasBenefitsFlag && Age > RetirementAge;

private bool HasBenefitsFlag => (flags & BenefitsFlagMask) == BenefitsFlagMask;
private const int RetirementAge = 65;
private const int BenefitsFlagMask = 0x01;
```

### Когда комментарий **уместен**

#### 1. Объяснить **почему**, не **что**

```csharp
// ❌ Bad — что и так видно
i++;  // increment i

// ✅ Good — почему
// Skip the BOM at the start of UTF-8 files
if (firstByte == 0xEF) reader.Skip(3);

// ✅ Good — non-obvious requirement
// Tax authority requires us to round HALF_UP, not banker's rounding
var tax = Math.Round(amount, 2, MidpointRounding.AwayFromZero);

// ✅ Good — warning о consequence
// IMPORTANT: this method is called from message handler — don't make it async
public void ProcessMessage(...) { }
```

#### 2. Legal / copyright

```csharp
// Copyright (c) 2026 Company Inc. All rights reserved.
// Licensed under MIT License.
```

#### 3. TODO / FIXME (с context)

```csharp
// TODO(JIRA-1234): Replace polling with SignalR notifications when ready (target: Q3 2026)
// FIXME: Cache invalidation broken in concurrent scenario, see #4567
```

С номером тикета — иначе TODO будут гнить годами.

#### 4. XML doc comments на public API

```csharp
/// <summary>
/// Calculates the total price including taxes for a given order.
/// </summary>
/// <param name="order">The order to calculate price for. Must contain at least one item.</param>
/// <returns>The total amount in the order's currency.</returns>
/// <exception cref="ArgumentException">Thrown if order has no items.</exception>
public decimal CalculateTotal(Order order)
```

Только для **public** API библиотеки. На internal код — нагрузка.

### Когда комментарий **плохой**

#### 1. "Очевидное" но неправильное

```csharp
// ❌ Lies! 
// Returns true if user is admin
public bool IsAdmin(User u) => u.Role.Contains("Admin", StringComparison.OrdinalIgnoreCase);
// Что если "AdminAssistant"? Тоже true! Comment врёт.
```

#### 2. Закомментированный код

```csharp
// ❌
public void Process()
{
    // var oldLogic = ...
    // foreach (var item in oldList)
    // {
    //     // ... 50 lines of commented code
    // }
    
    NewLogic();
}
```

Удали! Git помнит. Комментированный код становится **lie** — никто не знает актуальный он или нет.

#### 3. Журнал изменений

```csharp
// ❌
// 2024-01-01: Added by Alice
// 2024-02-15: Modified by Bob — fixed bug
// 2024-03-20: Refactored by Carol
public void Process() { }
```

Это для git, не для кода.

#### 4. Banner comments

```csharp
// ❌ Шум
//===========================================
// Setup phase
//===========================================
SetupDatabase();

//===========================================
// Processing phase
//===========================================
ProcessData();
```

Лучше — назови методы понятно или вынеси в private методы.

---

## 4. Форматирование

### Vertical formatting

Связанные понятия — **рядом**.

```csharp
// ❌ Логически связанные методы разбросаны
public class OrderService
{
    public void Process() { ... }
    
    private void Validate() { ... }
    
    public Order Get() { ... }
    
    private void NotifyCustomer() { ... }
    
    public void Cancel() { ... }
    
    private void SendEmail() { ... }
}

// ✅ Process и его helpers вместе
public class OrderService
{
    public void Process() { Validate(); /* ... */ NotifyCustomer(); }
    private void Validate() { ... }
    private void NotifyCustomer() { SendEmail(); }
    private void SendEmail() { ... }
    
    public Order Get() { ... }
    
    public void Cancel() { ... }
}
```

Public методы — наверху (entry points). Private helpers — рядом с теми кто их использует.

### Horizontal formatting

```csharp
// ❌ Слишком длинная строка, прокрутка нужна
var result = service.GetUserById(userId).Where(u => u.IsActive).Select(u => new UserDto { Name = u.Name, Email = u.Email }).ToList();

// ✅ Многострочная — читается сверху вниз
var result = service.GetUserById(userId)
    .Where(u => u.IsActive)
    .Select(u => new UserDto 
    { 
        Name = u.Name, 
        Email = u.Email 
    })
    .ToList();
```

Правило 80-120 chars per line — старое но всё ещё работает на side-by-side diff.

### Whitespace

```csharp
// ❌ Без воздуха
public class User{public string Name{get;set;}public int Age{get;set;}public bool IsActive(){return Age>=18;}}

// ✅ Воздух между concepts
public class User
{
    public string Name { get; set; }
    public int Age { get; set; }
    
    public bool IsActive() => Age >= 18;
}
```

Пустая строка — разделитель **смысловых блоков**. Не использовать как украшение.

---

## 5. Принципы (DRY, KISS, YAGNI)

### DRY — Don't Repeat Yourself

```csharp
// ❌ Дублирование validation
public void CreateUser(User u)
{
    if (string.IsNullOrEmpty(u.Email)) throw new();
    if (!u.Email.Contains("@")) throw new();
    if (u.Email.Length > 255) throw new();
    // ... save
}

public void UpdateUser(User u)
{
    if (string.IsNullOrEmpty(u.Email)) throw new();
    if (!u.Email.Contains("@")) throw new();
    if (u.Email.Length > 255) throw new();
    // ... update
}

// ✅ Один метод
private static void ValidateEmail(string email)
{
    if (string.IsNullOrEmpty(email)) throw new ArgumentException("Email empty");
    if (!email.Contains('@')) throw new ArgumentException("Email invalid");
    if (email.Length > 255) throw new ArgumentException("Email too long");
}

public void CreateUser(User u) { ValidateEmail(u.Email); /* save */ }
public void UpdateUser(User u) { ValidateEmail(u.Email); /* update */ }
```

> [!warning] DRY ≠ удалять любое дублирование
> Если две функции **похожи** но представляют **разные бизнес-концепции** — НЕ объединяй. Например:
> - `CalculateUsTax()` и `CalculateEuTax()` могут быть похожи но **меняются независимо**.
> - Объединишь их сейчас — потом разделять с разрастанием if/else.
>
> **DAMP > DRY for tests** — Don't Abstract Modules Prematurely.

### KISS — Keep It Simple, Stupid

```csharp
// ❌ Сложно для простой задачи
public bool IsAdult(int age) =>
    Enumerable.Range(18, 100).Contains(age);

// ✅ Просто
public bool IsAdult(int age) => age >= 18;
```

Если решение можно объяснить **одним предложением** — это правильное решение.

### YAGNI — You Aren't Gonna Need It

```csharp
// ❌ "На будущее" — никогда не пригодится
public class User
{
    public string Name { get; set; }
    
    // "Может пригодится"
    public string MiddleName { get; set; }
    public string SecondMiddleName { get; set; }
    public string Suffix { get; set; }
    public string Prefix { get; set; }
    public string PreferredName { get; set; }
    public string FormalName { get; set; }
}

// ✅ Минимум сейчас
public class User
{
    public string Name { get; set; }
    // Добавим когда будет requirement
}
```

**Правило:** добавляй только когда **есть реальный use case сегодня**.

---

## 6. Smells — запахи плохого кода

### Long Method (>50 строк)

См. выше, разбивай на маленькие.

### Large Class (>500 строк или >20 методов)

```
class UserService
    + register
    + login
    + logout
    + resetPassword
    + sendNotification    ← это про email, не про user!
    + generateReport       ← это про reporting, не про user!
    + exportToCsv          ← это про export, не про user!
    + ...20 more methods
```

Разбей: `UserService` + `EmailService` + `ReportService` + `ExportService`.

### Long Parameter List (>3-4)

См. выше, parameter object.

### Duplicate Code

См. DRY.

### Feature Envy

Метод одного класса слишком много обращается к другому классу.

```csharp
// ❌ OrderProcessor "интересуется" больше Customer чем Order
public class OrderProcessor
{
    public decimal CalculateDiscount(Order order)
    {
        if (order.Customer.Tier == "Gold" && order.Customer.Years > 5) return 0.15m;
        if (order.Customer.Tier == "Silver" && order.Customer.Years > 2) return 0.10m;
        return 0;
    }
}

// ✅ Move method к Customer
public class Customer
{
    public decimal GetDiscount() => (Tier, Years) switch
    {
        ("Gold", > 5) => 0.15m,
        ("Silver", > 2) => 0.10m,
        _ => 0m
    };
}
```

### Primitive Obsession

```csharp
// ❌ string для всего
public void SendEmail(string email, string country)
{
    if (country == "US") { ... }  // magic strings
}

// ✅ Strongly-typed
public record Email(string Value)
{
    public Email
    {
        if (!Value.Contains('@')) throw new ArgumentException("Invalid email");
    }
}

public enum Country { Us, Ru, Eu, ... }

public void SendEmail(Email email, Country country) { ... }
```

### Magic Numbers

```csharp
// ❌
if (user.Age > 17) { ... }       // 17 it 18? 
var tax = price * 0.08m;          // что за 0.08?

// ✅ Const с именем
const int LegalAdultAge = 18;
const decimal UsTaxRate = 0.08m;

if (user.Age >= LegalAdultAge) { ... }
var tax = price * UsTaxRate;
```

### Comments как костыль

См. секцию "Комментарии".

### Dead Code

Удали неиспользуемый код. Git помнит, IDE показывает unused.

---

## 7. Boy Scout Rule

> **Always leave the code cleaner than you found it.** — Robert C. Martin

Каждый PR — улучши **что-то** в коде который трогаешь:
- Переименуй криво названную переменную
- Разбей длинный метод на 2
- Удали мёртвый код
- Добавь маленький тест

Маленькие улучшения **накапливаются**. Через год — codebase значительно чище.

---

## 8. Practical workflow

### Process написания

1. **Сделай работающим** — first draft, neполный, ugly
2. **Сделай правильным** — handle edge cases, валидация
3. **Сделай чистым** — refactor: имена, разбивка, удаление duplication
4. **Тесты** — покрой ключевые сценарии

Не пытайся написать идеальный код **с первой попытки**. Пиши, потом улучшай.

### Refactoring сессии

Раз в неделю / спринт — выделяй время **только на refactoring**:
- Какой класс самый болезненный?
- Где больше всего багов?
- Что чаще всего меняется?

Эти места — кандидаты на refactoring.

См. [[refactoring|Refactoring]] для детальной техники.

---

## 9. Чек-лист для PR

Перед submit PR — пройди:

- [ ] Имена переменных / методов / классов передают намерение?
- [ ] Методы маленькие (<30 строк), делают одну вещь?
- [ ] Параметров не больше 3-4?
- [ ] Нет дублирования логики?
- [ ] Нет magic numbers / strings?
- [ ] Нет закомментированного кода?
- [ ] TODO имеют ticket number?
- [ ] Public API имеет XML doc comments?
- [ ] Тесты добавлены / обновлены?
- [ ] Linter / analyzer warnings — 0?
- [ ] `dotnet format` прогнан?

---

## 10. Common Pitfalls (новичков)

### 1. Over-engineering "на будущее"

```csharp
// ❌ "Может понадобится поддержка нескольких типов БД"
public interface IUserRepository<TKey, TUser, TConnection> where TUser : IUser<TKey> { ... }

public class GenericRepository<TKey, TUser, TConnection> : IUserRepository<...> { ... }

// ✅ YAGNI — пиши простое
public class UserRepository
{
    public Task<User?> GetByIdAsync(int id) { ... }
}
// Усложни когда понадобится (не понадобится)
```

### 2. Premature optimization

```csharp
// ❌ Оптимизация без замеров
// "Я слышал StringBuilder быстрее"
var sb = new StringBuilder();
foreach (var i in Enumerable.Range(0, 5))  // 5 раз!
    sb.Append(i);

// ✅ Простота для маленького количества
var s = string.Join("", Enumerable.Range(0, 5));
```

> "Premature optimization is the root of all evil." — Donald Knuth

### 3. Goto-paranoia

`break`, `continue`, early `return` — **не зло**. Используй:

```csharp
// ❌ Глубокая вложенность
public bool Process(Order order)
{
    bool result = false;
    if (order != null)
    {
        if (order.IsValid)
        {
            if (order.Total > 0)
            {
                result = true;
            }
        }
    }
    return result;
}

// ✅ Guard clauses + early return
public bool Process(Order order)
{
    if (order is null) return false;
    if (!order.IsValid) return false;
    if (order.Total <= 0) return false;
    
    return true;
}
```

### 4. Над-абстракция

```csharp
// ❌ Интерфейс на всё
public interface IUserNameProvider { string Name { get; } }
public interface IUserEmailProvider { string Email { get; } }
public interface IUserAgeProvider { int Age { get; } }

public class User : IUserNameProvider, IUserEmailProvider, IUserAgeProvider
{
    public string Name { get; set; }
    public string Email { get; set; }
    public int Age { get; set; }
}

// ✅ Just a class
public class User
{
    public string Name { get; set; }
    public string Email { get; set; }
    public int Age { get; set; }
}
```

Интерфейс нужен когда **есть несколько implementations** или **нужен mock в тестах** для DI.

### 5. Misleading тесты

```csharp
// ❌ Что проверяет?
[Fact]
public void Test1()
{
    var x = service.Do(1);
    Assert.NotNull(x);  // что NotNull? что было ожидание?
}

// ✅ Понятно из имени и тела
[Fact]
public void GetUserById_returns_user_when_id_exists()
{
    // Arrange
    var existingUserId = SeedTestUser();
    
    // Act
    var user = userService.GetById(existingUserId);
    
    // Assert
    user.Should().NotBeNull();
    user.Id.Should().Be(existingUserId);
}
```

---

## 11. Books и resources

См.[[99_reading-list|Reading List]].

Top-3 для clean code:

1. **Clean Code** — Robert C. Martin — классика, основа
2. **Code That Fits in Your Head** — Mark Seemann — современный взгляд
3. **The Pragmatic Programmer** — Hunt & Thomas — wider view

---

## 12. Best Practices summary

- **Имена раскрывают намерение** — не "что код делает", а "зачем"
- **Маленькие функции** — 5-15 строк, одну вещь делает
- **Один уровень абстракции** в методе
- **Минимум аргументов** (0-3, иначе DTO)
- **Без побочных эффектов** в методах валидации/проверки
- **Не возвращай null** — empty collections, Option\<T\>, или throw
- **Комментарии — почему**, не **что**
- **Удаляй мёртвый код** (включая закомментированный)
- **DRY — но не агрессивно** (бизнес concepts могут быть похожи но независимы)
- **KISS** — простое решение лучше "умного"
- **YAGNI** — не делай "на будущее"
- **Boy Scout Rule** — оставляй код чище чем нашёл
- **Refactor постоянно** — маленькими шагами, не "большой rewrite"

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

- [[code-quality|Code Quality]] — практические quality gates
- [[static-analysis|Static Analysis]] — автоматическая проверка
- [[refactoring|Refactoring]] — техники изменения кода
- [[code-review|Code Review]] — как ревьюить чужой код
-[[error-handling|Error Handling]] — паттерны для ошибок
-[[oop|OOP]] — объектно-ориентированные принципы
-[[solid|SOLID]] — высокоуровневые принципы

## Reading list

- **Clean Code** — Robert C. Martin
- **Code That Fits in Your Head** — Mark Seemann
- **The Pragmatic Programmer** — Hunt & Thomas
- **Code Complete** — Steve McConnell (детальный)
- **The Art of Readable Code** — Boswell & Foucher
- **Refactoring** — Martin Fowler (catalog of refactorings)
- **A Philosophy of Software Design** — John Ousterhout (другой взгляд на design)
