---
tags: [csharp, naming, conventions, junior, code-style, readability]
level: Junior
date: 2026-04-30
---

# Naming Conventions — именование в C#

> **Как называть классы, методы, переменные, файлы**. Стандартные .NET conventions + practical case studies. Closes пробел "пишу на C#, но имена выглядят как у Junior — `data`, `temp`, `myList`, `MyClass`". 

---

## Что это, зачем и когда

### Почему naming критичен

> **"Существует только две сложные вещи в Computer Science: cache invalidation и naming things"** — Phil Karlton

Plus over engineering, but anyway. Хорошее имя:
- **Передаёт намерение** — что делает / содержит
- **Снижает need для комментариев** — код документирует себя
- **Делает refactoring безопасным** — IDE понимает структуру

```csharp
// ❌ Плохие имена — читать невозможно
public class M
{
    public List<D> Get(int x)
    {
        var t = _r.Find(x);
        var r = new List<D>();
        foreach (var i in t)
        {
            if (i.S == 1) r.Add(i);
        }
        return r;
    }
}

// ✅ Хорошие имена — читается как английский
public class OrderService
{
    public List<Order> GetActiveOrdersByUser(int userId)
    {
        var allOrders = _repository.FindByUser(userId);
        var activeOrders = new List<Order>();
        foreach (var order in allOrders)
        {
            if (order.Status == OrderStatus.Active) activeOrders.Add(order);
        }
        return activeOrders;
    }
}
```

### Главный принцип

> **Имя должно сказать читателю**: что это, зачем, когда использовать. Без необходимости открыть implementation.

---

## 1. Casing — стили именования

### Стандарты .NET

| Тип | Стиль | Пример |
|-----|-------|--------|
| **Класс / struct / record** | PascalCase | `OrderService`, `UserDto` |
| **Interface** | I + PascalCase | `IOrderService`, `IRepository` |
| **Метод** | PascalCase | `GetUserById`, `CalculateTotal` |
| **Property** | PascalCase | `FirstName`, `OrderTotal` |
| **Public field** | PascalCase | (`public string Name`) |
| **Private field** | _camelCase | `_userService`, `_logger` |
| **Local variable** | camelCase | `var orderId = 5` |
| **Parameter** | camelCase | `void Process(int userId)` |
| **Constant** | PascalCase | `const int MaxRetries = 5` |
| **Enum value** | PascalCase | `OrderStatus.Pending` |
| **Type parameter (generic)** | T + PascalCase | `T`, `TKey`, `TValue`, `TResponse` |
| **Async method** | + Async suffix | `LoadDataAsync` |
| **File name** | PascalCase + `.cs` | `OrderService.cs` |
| **Namespace** | PascalCase | `MyApp.Services.Orders` |

### Примеры по комплексу

```csharp
namespace MyApp.Services.Orders
{
    public interface IOrderService
    {
        Task<Order> GetByIdAsync(int orderId);
        Task<List<Order>> SearchAsync(OrderSearchCriteria criteria);
    }

    public class OrderService : IOrderService
    {
        private const int MaxOrdersPerPage = 50;
        
        private readonly IOrderRepository _repository;
        private readonly ILogger<OrderService> _logger;
        
        public OrderService(IOrderRepository repository, ILogger<OrderService> logger)
        {
            _repository = repository;
            _logger = logger;
        }
        
        public async Task<Order> GetByIdAsync(int orderId)
        {
            _logger.LogInformation("Loading order {OrderId}", orderId);
            var order = await _repository.FindByIdAsync(orderId);
            return order ?? throw new OrderNotFoundException(orderId);
        }
        
        public async Task<List<Order>> SearchAsync(OrderSearchCriteria criteria)
        {
            // ...
        }
    }

    public enum OrderStatus
    {
        Draft,
        Pending,
        Confirmed,
        Shipped,
        Delivered,
        Cancelled
    }
}
```

---

## 2. Case Study #1 — Method naming

### ❌ Плохие имена

```csharp
public class UserService
{
    public User Get(int id) { }                      // Что get? Юзер? Email? 
    public bool Check(string s) { }                  // Что check?
    public List<User> GetData() { }                   // Какие data?
    public void Process(Order o) { }                  // Process что? Куда?
    public bool DoIt() { }                            // Сделать что?
    public string Format(User u, bool? f, int x) { } // Format что?
}
```

### ✅ Хорошие имена

```csharp
public class UserService
{
    public User GetById(int userId) { }                              // ясно что get + by what
    public bool IsEmailRegistered(string email) { }                  // ясное условие
    public List<User> GetActiveUsers() { }                            // что возвращает
    public void SubmitOrderForApproval(Order order) { }              // действие очевидно
    public Result CreateInvitationLink(int userId) { }                // verb + object
    public string FormatUserDisplayName(User user, bool includeEmail) // что format + параметры понятны
    {
    }
}
```

### Правила для методов

```
1. Глагол + noun — обязательно
   ❌ User(id)
   ✅ GetUser(id), CreateUser(...)

2. Verb prefix зависит от семантики:
   Get*           → возвращает, не изменяет, может throw если not found
   Find*          → возвращает или null (не throws)
   Try*           → bool + out parameter (TryParse pattern)
   Is/Has/Can*    → bool predicate
   Create*        → создаёт и возвращает new
   Build*         → конструирует составной объект
   Add/Remove*    → modify collection
   Save/Delete*   → persistence
   Calculate*     → computation
   Convert*       → transformation
   Validate*      → checks → throws ValidationException
   To*            → conversion (ToString, ToList, ToArray)
   On*            → event handler

3. Async → суффикс Async
   GetUserAsync, SaveOrderAsync
```

### Examples

```csharp
public User GetById(int id);             // throws if not found
public User? FindById(int id);            // returns null
public bool TryFindById(int id, out User user);  // bool + out

public bool IsActive { get; }
public bool HasPermission(string code);
public bool CanEdit(int userId);

public Task SaveChangesAsync();
public Task<List<User>> GetActiveUsersAsync();
```

---

## 3. Case Study #2 — Variable / parameter naming

### ❌ Плохое

```csharp
public List<Order> GetOrders(int x, DateTime d, bool f)
{
    var t = new List<Order>();
    var d2 = DateTime.Now;
    
    foreach (var i in _db.Orders)
    {
        if (i.U == x && i.D > d && (f || !i.A))
        {
            t.Add(i);
        }
    }
    
    return t;
}
```

### ✅ Хорошее

```csharp
public List<Order> GetOrders(int userId, DateTime fromDate, bool includeArchived)
{
    var matchingOrders = new List<Order>();
    
    foreach (var order in _db.Orders)
    {
        bool matchesUser = order.UserId == userId;
        bool matchesDate = order.CreatedAt > fromDate;
        bool isVisible = includeArchived || !order.IsArchived;
        
        if (matchesUser && matchesDate && isVisible)
        {
            matchingOrders.Add(order);
        }
    }
    
    return matchingOrders;
}
```

### Правила

```
1. Длина имени ∝ scope:
   - Loop variable (3-5 lines) → i, j, k OK
   - Local in function → meaningful
   - Class field / property → very meaningful
   - Public API → maximum clarity

2. Не аббревиатуры (кроме общеизвестных):
   ❌ usrSrv, mgr, hndlr
   ✅ userService, manager, handler
   
   Исключения OK: id, url, html, xml, css, http, db, ip
   ❌ HttpUrl  ✅ HttpUrl OK (сокращение известное)
   
3. Boolean — yes/no question:
   ❌ flag, status, mode
   ✅ isActive, hasPermission, canEdit, shouldRetry

4. Collection — plural:
   ❌ user (для list)
   ✅ users, orderList, activeOrders

5. Не упоминай тип в имени:
   ❌ orderList, userArray, idDictionary
   ✅ orders, users, idLookup
   (если refactoring меняет тип — имя не врёт)
```

---

## 4. Case Study #3 — Class / type naming

### ❌ Плохое

```csharp
public class Manager { }              // Manager чего?
public class Helper { }               // Helper для чего?
public class Util { }                 // Junk drawer
public class DataProcessor { }        // Какие data? Как process?
public class MyOrderClass { }          // "My", "Class" ничего не дают
public class Order2 { }                // 2 от чего?
public class OrderImpl { }             // "Impl" — это Java стиль
```

### ✅ Хорошее

```csharp
public class OrderRepository { }            // явно, что — repository для Order
public class OrderValidator { }             // validates Order
public class EmailNotificationService { }   // sends email notifications
public class JsonOrderSerializer { }        // serializes Order to JSON
public class CachingUserService : IUserService { }  // декоратор
public class OrderApiController { }          // ASP.NET controller
public class OrderEntity { }                 // EF entity (отделить от DTO)
public class OrderDto { }                    // Data Transfer Object
public class CreateOrderRequest { }          // Web API request
public class CreateOrderCommand { }          // CQRS command
```

### Patterns / suffixes

```
Naming convention передаёт role:

*Service        → бизнес-логика
*Repository     → data access
*Controller     → ASP.NET MVC controller
*Handler        → MediatR / event handler
*Validator      → input validation
*Builder        → builder pattern
*Factory        → factory pattern
*Strategy       → strategy pattern
*Manager        → ⚠️ обычно слишком vague — лучше точное название
*Helper         → ⚠️ слишком generic — точное лучше
*Utils          → ⚠️ junk drawer alarm

*Dto            → data transfer
*Entity         → EF / domain entity
*Vm / *ViewModel → MVVM / web view model
*Request        → API request
*Response       → API response
*Command        → CQRS command (write)
*Query          → CQRS query (read)
*Event          → domain / integration event
*Exception      → custom exception
*Attribute      → custom attribute
*Configuration  → config class
*Options        → IOptions pattern (.NET)
```

### Domain-specific better than generic

```csharp
// ❌ Generic
public class DataManager { }
public class FileHelper { }
public class StringUtil { }

// ✅ Specific
public class OrderRepository { }
public class CsvFileReader { }
public class SqlIdentifierEscaper { }
```

---

## 5. Case Study #4 — Generic types

### Generic parameters

```csharp
// ❌ Плохо — что за T?
public class Cache<T> { }
public Result<T, U> Process<T, U>() { }

// ✅ Better — meaningful когда reasonable
public class Cache<TItem> { }
public Result<TValue, TError> Process<TValue, TError>() { }

public class Repository<TEntity, TKey>
    where TEntity : class
    where TKey : struct
{
    public Task<TEntity?> FindByIdAsync(TKey id) { /* ... */ }
}
```

**Conventions:**
```
T              — single generic, no further info
TItem, TValue  — содержание контейнера
TKey           — key dictionary
TResult        — return type
TRequest, TResponse  — request/response pair
TException     — exception type
TEntity        — domain entity
```

---

## 6. Case Study #5 — Boolean naming

### ❌ Vague

```csharp
public bool Status { get; set; }     // что Status?
public bool Mode { get; set; }       // mode чего?
public bool Flag { get; set; }       // flag для чего?
public bool Check { get; }            // check что?
```

### ✅ Yes/no question

```csharp
public bool IsActive { get; set; }
public bool IsEnabled { get; set; }
public bool HasItems { get; }
public bool HasPermission { get; }
public bool CanEdit { get; }
public bool CanRetry(int attempt);
public bool ShouldRetry { get; }
public bool RequiresApproval { get; }
public bool WasSuccessful { get; }
public bool IsValid(out string error);
```

### Negative naming — избегай

```csharp
// ❌ Double negative
public bool IsNotEmpty { get; }

if (!IsNotEmpty) { }   // двойное отрицание — confusing!

// ✅ Positive
public bool IsEmpty { get; }
if (IsEmpty) { }
if (!IsEmpty) { }   // single negation OK
```

---

## 7. Case Study #6 — Files и папки

### Convention

```
File name           → Тот же что main public type внутри
                       OrderService.cs содержит class OrderService

One type per file   → почти всегда
                       (исключения: tightly coupled types, partial classes)

Folder = namespace  → структура папок отражает namespaces

PascalCase          → имена файлов и папок
```

### Examples

```
src/
├── MyApp.Domain/
│   ├── Orders/
│   │   ├── Order.cs
│   │   ├── OrderItem.cs
│   │   ├── OrderStatus.cs
│   │   └── IOrderRepository.cs
│   ├── Users/
│   │   ├── User.cs
│   │   └── IUserRepository.cs
│   └── Common/
│       └── Money.cs
│
├── MyApp.Application/
│   ├── Orders/
│   │   ├── Commands/
│   │   │   ├── CreateOrderCommand.cs
│   │   │   ├── CreateOrderHandler.cs
│   │   │   └── CreateOrderValidator.cs
│   │   └── Queries/
│   │       └── GetOrderByIdQuery.cs
│   └── Common/
│       └── Result.cs
│
├── MyApp.Infrastructure/
│   ├── Persistence/
│   │   ├── AppDbContext.cs
│   │   └── Repositories/
│   │       ├── OrderRepository.cs
│   │       └── UserRepository.cs
│   └── Services/
│       └── EmailService.cs
│
└── MyApp.Api/
    ├── Controllers/
    │   ├── OrdersController.cs
    │   └── UsersController.cs
    ├── Filters/
    │   └── ExceptionFilter.cs
    └── Program.cs
```

### Test файлы

```
src/
└── MyApp.Domain/
    └── Orders/Order.cs

tests/
└── MyApp.Domain.Tests/
    └── Orders/
        └── OrderTests.cs   ← {ClassName}Tests.cs
```

---

## 8. Async naming

### Convention

```csharp
// ✅ Async suffix для async methods
public Task<User> GetByIdAsync(int id);
public Task SaveAsync();
public Task<bool> IsRegisteredAsync(string email);

// ❌ Без Async — неясно
public Task<User> GetById(int id);
```

### Исключения — interface controllers

```csharp
// ASP.NET Controllers — обычно без Async (новая convention)
public class OrdersController : ControllerBase
{
    public async Task<Order> Get(int id) { }   // OK без Async
}

// Но в service — с Async
public interface IOrderService
{
    Task<Order> GetByIdAsync(int id);   // Async обязательно
}
```

---

## 9. Common Pitfalls

### 1. Hungarian notation (старый стиль)

```csharp
// ❌ Microsoft в 90е (даже у MS на новых API не используется)
string strName;
int intCount;
bool bIsActive;
List<int> arrIds;

// ✅ Modern .NET
string name;
int count;
bool isActive;
List<int> ids;
```

### 2. Snake_case или kebab-case

```csharp
// ❌ Не C# convention
string user_name;
int max_retry_count;

// ✅ camelCase
string userName;
int maxRetryCount;
```

### 3. Сокращения без причины

```csharp
// ❌
public class UsrMgr { }
public bool ChkAuth(string usr, string pwd) { }

// ✅
public class UserManager { }
public bool CheckAuthentication(string username, string password) { }
```

**Допустимые сокращения:**
- `Id`, `Url`, `Html`, `Xml`, `Json`, `Csv`, `Sql`, `Db`, `Ip`, `Os`
- `Tcp`, `Udp`, `Http`, `Https`
- В loop iterators: `i`, `j`, `k`

### 4. Numbers в именах

```csharp
// ❌
public class Order2 { }
public string GetData2() { }

// ✅ — что отличает 2 от 1?
public class ImprovedOrder { }
public class OrderV2 { }   // если действительно version
public string GetDataPaged() { }
```

### 5. Обманчивые имена

```csharp
// ❌
public List<User> GetActiveUsers()
{
    return _db.Users.ToList();  // ⚠️ возвращает ВСЕХ, не только active!
}

// ✅
public List<User> GetActiveUsers()
{
    return _db.Users.Where(u => u.IsActive).ToList();
}

// или переименовать:
public List<User> GetAllUsers() { /* ... */ }
```

### 6. Несовпадение style между команды

```csharp
// ❌ Inconsistent
private string _userId;       // _camelCase ✓
private int M_count;          // M_PascalCase — Java-style!
private bool isActive_;       // trailing _ — C++ style

// ✅ Single style по convention
private string _userId;
private int _count;
private bool _isActive;
```

EditorConfig помогает enforce — см. [[../Quality/static-analysis|Static Analysis]].

### 7. Single-letter в широком scope

```csharp
public class OrderService
{
    private int n;             // ❌ что за n?
    private List<Order> l;     // ❌ list of what?
    
    public List<Order> Process(int x, DateTime d)  // ❌ x, d?
}

// Single letter OK ТОЛЬКО:
// - i, j, k в маленьких циклах
// - Generic params (T, K, V)
// - Lambda short ones: items.Where(x => x > 0)
```

### 8. Violations of conventions

```csharp
// ❌ От community:
public class SQLProvider { }   // или Sql или SQL — choose!

// ✅ Microsoft рекомендует:
public class SqlProvider { }   // multi-letter abbreviation = first cap, rest lowercase

// Исключения для 2-letter abbrev:
public class IDbConnection { } // ID — 2 letters, all caps OK
public class IPAddress { }     // IP — все caps OK
```

### 9. Static methods как class names

```csharp
// ❌
public static class Utils
{
    public static string FormatDate(DateTime d) { }
    public static int CalculateAge(DateTime birth) { }
    // junk drawer
}

// ✅ — split by domain
public static class DateFormatter
{
    public static string FormatShort(DateTime d) { }
}

public static class AgeCalculator
{
    public static int FromBirthDate(DateTime birth) { }
}
```

### 10. Inconsistent с .NET BCL

```csharp
// .NET BCL pattern: TryGet returns bool, value через out
public bool TryGetValue(int key, out string value);

// ✅ Твой код тоже — same pattern
public bool TryFindUser(int id, out User user);

// ❌ Disonance
public User TryGetUser(int id);  // Try но не bool? Confusing!
```

---

## 10. Best Practices

### General

- **Convention обязательна** — `_camelCase` для private fields, `PascalCase` для everything else
- **Имя = намерение** — "что делает", не "как"
- **Длиннее лучше vague short**
- **Избегай Manager, Helper, Util**
- **Refactor names при изменении semantics**

### Methods

- **Verb + noun** обязательно
- **Get** vs **Find** vs **Try** — semantics разные
- **Boolean** — yes/no question (Is, Has, Can, Should)
- **Async** suffix
- **One method = one purpose** (если имя сложное — может слишком много делает)

### Variables

- **Domain language** — `customer`, `orderId`, `paymentMethod` не `data`, `obj`, `temp`
- **Avoid encoding type** в имя
- **Short scope = short name OK**, long scope = descriptive
- **Plural for collections**

### Classes

- **Suffix указывает role** (`*Service`, `*Repository`)
- **Domain-specific** не generic
- **Interface — `I*`** prefix
- **One class per file** обычно

### Tools

- **EditorConfig** — enforce naming
- **Roslyn analyzers** — `CA1707` (no underscores), `CA1708` (case sensitivity), etc.
- **StyleCop** для extra strict
- **Code review** — naming feedback обязательно

См. [[../Quality/code-quality|Code Quality]] и [[../Quality/static-analysis|Static Analysis]].

---

## 11. Case Study #7 — Real refactor

### До

```csharp
public class CtrlMgr
{
    private List<X> data;
    private int n;
    
    public bool Process(string s, int x, bool f)
    {
        var t = new List<X>();
        foreach (var i in data)
        {
            if (i.S == s && (!f || i.N > x))
            {
                t.Add(i);
            }
        }
        n = t.Count;
        return n > 0;
    }
}
```

### Что не понятно

- `CtrlMgr` — что за controller? Manager?
- `X` — что это?
- `data`, `n` — какие именно?
- `Process(s, x, f)` — что делает, что означают параметры?

### После refactor

```csharp
public class OrderFilterService
{
    private readonly List<Order> _orders;
    private int _matchCount;
    
    public bool ApplyFilter(string status, int minAmount, bool requireMinAmount)
    {
        var matchedOrders = new List<Order>();
        
        foreach (var order in _orders)
        {
            bool matchesStatus = order.Status == status;
            bool meetsAmountRequirement = !requireMinAmount || order.Amount > minAmount;
            
            if (matchesStatus && meetsAmountRequirement)
            {
                matchedOrders.Add(order);
            }
        }
        
        _matchCount = matchedOrders.Count;
        return _matchCount > 0;
    }
}
```

**Что улучшилось:**
- Class name говорит **роль**
- Method name говорит **что делает**
- Параметры имеют smysl
- Locals читаются как business logic
- Booleans — yes/no questions (matchesStatus, meetsAmountRequirement)

---

## 12. Cheat sheet

| Convention | Пример |
|-----------|--------|
| Class | `OrderService` |
| Interface | `IOrderService` |
| Struct / Record | `Money`, `OrderCreated` |
| Method | `GetUserById`, `SaveAsync` |
| Property | `FirstName` |
| Private field | `_userService` |
| Public field | `Name` (rare — use property!) |
| Local var | `var orderId = 5` |
| Parameter | `int userId` |
| Constant | `MaxRetries` |
| Enum value | `OrderStatus.Pending` |
| Generic param | `T`, `TKey`, `TValue` |
| Async method | `LoadDataAsync` |
| File | `OrderService.cs` |
| Namespace | `MyApp.Services.Orders` |
| Test | `OrderServiceTests.cs` |
| Test method | `GetById_WhenNotFound_ThrowsException` |

| Verb prefix | Semantics |
|-------------|-----------|
| `Get*` | Returns, throws if not found |
| `Find*` | Returns or null |
| `Try*` | bool + out parameter |
| `Is/Has/Can/Should` | Boolean |
| `Create*` | New object |
| `Build*` | Compose complex |
| `Add/Remove*` | Modify collection |
| `Save/Delete*` | Persistence |
| `Calculate*` | Computation |
| `Convert/To*` | Transformation |
| `Validate*` | Throws on failure |
| `On*` | Event handler |

---

## 13. См. также

- [[csharp-basics|C# Basics]] — fundamentals
- [[../Quality/clean-code|Clean Code]] — naming как часть clean code
- [[../Quality/code-review|Code Review]] — naming в review
- [[../Quality/static-analysis|Static Analysis]] — automated naming checks
- [[oop|OOP]] — naming для inheritance
- [[design-patterns|Design Patterns]] — pattern names

## Reading list

- **Microsoft Naming Guidelines** — learn.microsoft.com/dotnet/standard/design-guidelines/naming-guidelines
- **Framework Design Guidelines** — Cwalina & Abrams (must-read для API design)
- **Clean Code** — Robert Martin (chapter "Meaningful Names")
- **Code Complete** — Steve McConnell (chapter про naming)
- **The Art of Readable Code** — Boswell & Foucher
