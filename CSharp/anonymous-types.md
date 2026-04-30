---
tags: [csharp, anonymous-types, linq, projections, junior, middle]
level: Junior
date: 2026-04-30
---

# Anonymous Types — анонимные типы

> **`new { Name = "John", Age = 30 }`** — типы создаваемые на лету без явного объявления. Used heavily в LINQ projections и temp data shaping. Closes пробел "знаю что есть, не понимаю отличий от tuples и records".

---

## Что это, зачем и когда

### Что такое анонимный тип

**Тип созданный compiler'ом на основе object initializer**. У него нет имени в коде, но есть в IL.

```csharp
var person = new { Name = "John", Age = 30 };

// Compiler генерит:
// public class <>f__AnonymousType0<TName, TAge>
// {
//     public TName Name { get; }
//     public TAge Age { get; }
//     ...
// }
```

### Зачем

| Сценарий | Без anonymous types | С anonymous types |
|----------|---------------------|-------------------|
| LINQ projection | Создать класс для каждой проекции | `select new { x, y }` inline |
| Temp data shape | DTO для одного метода | `new { ... }` локально |
| Multi-key Group | Class с composite key | `g => new { g.Year, g.Month }` |
| Quick pass-through | Создать tuple | Anonymous — typed properties |

### Главные свойства

```csharp
var p = new { Name = "John", Age = 30 };

// 1. Properties read-only (immutable)
// p.Name = "Jane";  // ❌ Compile error

// 2. Имена properties — required
// var p = new { "John", 30 };  // ❌ — нужны Name = / Age =

// 3. Value equality (как records)
var p1 = new { Name = "John", Age = 30 };
var p2 = new { Name = "John", Age = 30 };
p1.Equals(p2);  // true — same shape, same values

// 4. Reasonable ToString
Console.WriteLine(p);  // { Name = John, Age = 30 }

// 5. Только в текущем method scope (нельзя возвращать typed)
```

### Когда применять

✅ **Используй когда:**
- LINQ projections (`select new { ... }`)
- Composite keys в `GroupBy` / `Join`
- Temp data shape внутри method
- Logging structured data
- Quick experiments / prototypes

❌ **НЕ используй когда:**
- Нужно вернуть из метода — используй `record` / class
- Нужно передать в API — определи type
- Cross-method scope — определи class
- Нужна mutability — определи class

См. [[modern-features|Records]] и [[tuples-deconstruction|Tuples]].

---

## 1. Базовое использование

### Создание

```csharp
// Inline assignment
var person = new { Name = "John", Age = 30 };
Console.WriteLine($"{person.Name}, {person.Age}");

// Из существующих переменных (capture)
string name = "John";
int age = 30;
var person = new { name, age };  // Имена = переменных!
// person.name, person.age (lowercase!)

// Mixed
var p = new { Name = name, Age = age + 5 };
```

### Property promotion

```csharp
// Если property name = source name — можно опустить
var user = new User { Name = "John", Email = "j@x.com" };

var dto = new { user.Name, user.Email };
// Эквивалентно
var dto = new { Name = user.Name, Email = user.Email };
```

### Nested

```csharp
var employee = new
{
    Name = "John",
    Address = new
    {
        City = "Moscow",
        Country = "RU"
    },
    Skills = new[] { "C#", "SQL" }
};

employee.Address.City;       // "Moscow"
employee.Skills[0];           // "C#"
```

---

## 2. Case Study #1 — LINQ projections

### Задача

Vault имеет orders. Хочется получить summary без загрузки всех колонок.

### Без anonymous types

```csharp
// Создать DTO class
public class OrderSummaryDto
{
    public int OrderId { get; set; }
    public string CustomerName { get; set; }
    public decimal Total { get; set; }
}

// Use
var summaries = await db.Orders
    .Select(o => new OrderSummaryDto
    {
        OrderId = o.Id,
        CustomerName = o.Customer.Name,
        Total = o.Total
    })
    .ToListAsync();
```

### С anonymous types

```csharp
// Никаких extra classes
var summaries = await db.Orders
    .Select(o => new
    {
        OrderId = o.Id,
        CustomerName = o.Customer.Name,
        Total = o.Total
    })
    .ToListAsync();

// Use
foreach (var s in summaries)
{
    Console.WriteLine($"Order {s.OrderId}: {s.CustomerName} — {s.Total:C}");
}
```

### Когда DTO лучше

```csharp
// Если данные передаются:
// - В View / API response
// - В другой method
// - В public API

// → Создать DTO. Anonymous types НЕ для cross-boundary.

public async Task<List<OrderSummaryDto>> GetSummariesAsync()
{
    return await db.Orders
        .Select(o => new OrderSummaryDto  // ⚠️ Не anonymous!
        {
            OrderId = o.Id,
            CustomerName = o.Customer.Name,
            Total = o.Total
        })
        .ToListAsync();
}
```

См. [[../EFCore/queries-performance|EF Queries Performance]].

---

## 3. Case Study #2 — Composite key в GroupBy

### Задача

Сгруппировать orders по году+месяцу для отчёта.

### Без anonymous

```csharp
public class YearMonth
{
    public int Year { get; set; }
    public int Month { get; set; }
    
    // ⚠️ Нужно реализовать Equals/GetHashCode для GroupBy!
    public override bool Equals(object obj) =>
        obj is YearMonth other && other.Year == Year && other.Month == Month;
    
    public override int GetHashCode() => HashCode.Combine(Year, Month);
}

var byMonth = orders.GroupBy(o => new YearMonth 
{ 
    Year = o.Date.Year, 
    Month = o.Date.Month 
});
```

### С anonymous

```csharp
var byMonth = orders.GroupBy(o => new 
{ 
    o.Date.Year, 
    Month = o.Date.Month   // Year property promotion, Month — explicit
});

// Anonymous имеет встроенный Equals/GetHashCode по value!
// Никакого extra кода.

foreach (var group in byMonth)
{
    Console.WriteLine($"{group.Key.Year}-{group.Key.Month}: {group.Count()} orders");
}
```

### Sort + Group + Aggregate

```csharp
var report = orders
    .GroupBy(o => new { o.Date.Year, o.Date.Month, o.CustomerType })
    .Select(g => new 
    { 
        g.Key.Year,
        g.Key.Month,
        g.Key.CustomerType,
        Count = g.Count(),
        Total = g.Sum(o => o.Amount),
        Avg = g.Average(o => o.Amount)
    })
    .OrderBy(r => r.Year).ThenBy(r => r.Month)
    .ToList();
```

См. [[collections-linq|Collections & LINQ]].

---

## 4. Case Study #3 — Multi-table JOIN

### Задача

Get orders с customer info и shipping address — без extra DTO.

```csharp
var data = await db.Orders
    .Join(db.Customers, o => o.CustomerId, c => c.Id, (o, c) => new { Order = o, Customer = c })
    .Join(db.Addresses, oc => oc.Customer.AddressId, a => a.Id, (oc, a) => new
    {
        oc.Order.Id,
        oc.Order.Total,
        CustomerName = oc.Customer.Name,
        ShippingCity = a.City,
        ShippingZip = a.PostalCode
    })
    .ToListAsync();

// data — List of anonymous type, typed properties
foreach (var item in data)
{
    Console.WriteLine($"Order {item.Id} → {item.CustomerName} в {item.ShippingCity}");
}
```

См. [[../EFCore/queries-performance|EF Queries]].

---

## 5. Case Study #4 — Logging structured data

### Задача

Logging с ILogger — нужны structured properties.

```csharp
public async Task PlaceOrder(int customerId, List<int> productIds)
{
    var stopwatch = Stopwatch.StartNew();
    
    try
    {
        var order = await ProcessOrder(customerId, productIds);
        
        // Structured logging — anonymous как context
        _logger.LogInformation("Order placed: {@OrderInfo}", new
        {
            OrderId = order.Id,
            CustomerId = customerId,
            ItemCount = productIds.Count,
            Total = order.Total,
            Duration = stopwatch.ElapsedMilliseconds
        });
    }
    catch (Exception ex)
    {
        _logger.LogError(ex, "Order failed: {@Context}", new
        {
            CustomerId = customerId,
            ProductIds = productIds,
            Duration = stopwatch.ElapsedMilliseconds
        });
        throw;
    }
}
```

В Serilog / structured loggers — anonymous types serialize в JSON-like format.

См. [[../AspNetCore/logging-observability|Logging]].

---

## 6. Case Study #5 — Razor / Blazor view models

### Задача

Передать данные в Razor view — temp shape.

```csharp
// Controller (MVC)
public IActionResult Dashboard()
{
    var data = new
    {
        UserName = User.Identity.Name,
        ActiveOrders = db.Orders.Count(o => o.Status == "Active"),
        Revenue = db.Orders.Where(o => o.Date >= DateTime.Today).Sum(o => o.Total),
        TopProducts = db.Products.OrderByDescending(p => p.SalesCount).Take(5).ToList()
    };
    
    return View(data);
}
```

```razor
@* В Razor view *@
@model dynamic  @* anonymous type cannot be strongly typed in views *@

<h1>Hello @Model.UserName</h1>
<p>Active: @Model.ActiveOrders, Today's revenue: @Model.Revenue</p>
```

> [!warning] Razor + anonymous types → dynamic
> Anonymous types НЕ работают как strong-typed model в Razor (другая assembly). Лучше — define ViewModel class.

---

## 7. Anonymous Types vs Tuples vs Records

### Все три похожи но разные

```csharp
// 1. Anonymous type
var anon = new { Name = "John", Age = 30 };

// 2. ValueTuple
var tuple = (Name: "John", Age: 30);

// 3. Record
public record Person(string Name, int Age);
var rec = new Person("John", 30);
```

### Сравнение

| | Anonymous | ValueTuple | Record |
|--|-----------|------------|--------|
| Имена properties | ✅ Required | Optional (Item1/Item2) | ✅ Defined |
| Mutability | Read-only | Mutable | Init-only (default) |
| Equality | По value | По value | По value |
| Cross-method | ❌ (locally only) | ✅ | ✅ |
| Cross-assembly | ❌ | ✅ | ✅ |
| Можно вернуть | ❌ (typed) | ✅ | ✅ |
| Inheritance | ❌ | ❌ | ✅ |
| `with` expression | ❌ | ❌ | ✅ |
| Stack-allocated | Heap | Stack (struct) | Heap |
| Pattern matching | partial | ✅ | ✅ |

### Когда что

```
LINQ projection inside one method?
  → Anonymous type

Return multiple values from method?
  → ValueTuple

Domain entity with value semantics?
  → Record

Quick deconstruction?
  → Tuple

Cross-method shared shape?
  → Record (or class)
```

### Code examples

```csharp
// Anonymous — temp в методе
public void ProcessOrders()
{
    var summaries = orders.Select(o => new { o.Id, o.Total });
    // summaries usable только here
}

// Tuple — return multiple values
public (string Name, int Age) GetPerson() => ("John", 30);

var (name, age) = GetPerson();

// Record — domain
public record Customer(string Name, string Email)
{
    public DateTime CreatedAt { get; init; } = DateTime.UtcNow;
}

var c1 = new Customer("John", "j@x.com");
var c2 = c1 with { Email = "new@x.com" };  // copy + modify
```

См. [[tuples-deconstruction|Tuples]] и [[modern-features|Records]].

---

## 8. Internal mechanics

### Что генерит compiler

```csharp
var p = new { Name = "John", Age = 30 };
```

→ Generates:

```csharp
[CompilerGenerated]
internal sealed class <>f__AnonymousType0<<Name>j__TPar, <Age>j__TPar>
{
    private readonly <Name>j__TPar <Name>i__Field;
    private readonly <Age>j__TPar <Age>i__Field;
    
    public <Name>j__TPar Name => <Name>i__Field;
    public <Age>j__TPar Age => <Age>i__Field;
    
    public <>f__AnonymousType0(<Name>j__TPar Name, <Age>j__TPar Age)
    {
        <Name>i__Field = Name;
        <Age>i__Field = Age;
    }
    
    public override bool Equals(object value) { /* по полям */ }
    public override int GetHashCode() { /* по полям */ }
    public override string ToString() { /* "{ Name = X, Age = Y }" */ }
}
```

### Re-use anonymous type

```csharp
// Same shape → same compiler-generated type
var a = new { Name = "John", Age = 30 };
var b = new { Name = "Jane", Age = 25 };

a.GetType() == b.GetType();  // true — re-used!

// Different shape → different type
var c = new { Name = "Bob", Email = "b@x.com" };
a.GetType() == c.GetType();  // false
```

### Property order matters!

```csharp
var a = new { Name = "X", Age = 30 };
var b = new { Age = 30, Name = "X" };  // другой порядок!

a.GetType() == b.GetType();  // false — different types
a.Equals(b);                   // false
```

> [!warning] Property order
> Compiler generates type per **порядок** properties. `new { A, B }` ≠ `new { B, A }`.

---

## 9. Common Pitfalls

### 1. Невозможно вернуть typed

```csharp
public ??? GetData()  // что писать?
{
    return new { Name = "John", Age = 30 };
}

// Workarounds:
// A. Return object (теряем типизацию)
public object GetData() => new { Name = "John", Age = 30 };

// B. Use dynamic
public dynamic GetData() => new { Name = "John", Age = 30 };

// C. Best — define record/class
public record Person(string Name, int Age);
public Person GetData() => new("John", 30);
```

### 2. Pass через method boundary

```csharp
public void Process(/* type? */)
{
    // ... use anonymous type ...
}

// ❌ Нельзя specifically typed
// ✅ Generic helper
public void Process<T>(T item) { }  // works, но T inferred
```

### 3. Mutability

```csharp
var p = new { Name = "John" };
p.Name = "Jane";  // ❌ Compile error — properties read-only

// Если нужна мутация — использовать запись:
var p2 = new { p.Name };  // shallow copy, но nothing changes — это просто copy

// ✅ С record — `with` expression
public record Person(string Name);
var p = new Person("John");
var p2 = p with { Name = "Jane" };
```

### 4. Reflection-only access

```csharp
// Внутри method — typed
var p = new { Name = "John" };
p.Name;  // OK

// Передан как object — reflection
public void Print(object obj)
{
    var prop = obj.GetType().GetProperty("Name");
    var value = prop?.GetValue(obj);
    Console.WriteLine(value);
}
```

### 5. JSON serialization works

```csharp
var data = new { Name = "John", Age = 30 };
string json = JsonSerializer.Serialize(data);  
// {"Name":"John","Age":30} ✅

// Deserialize обратно — нужен type. Поэтому для return — class!
```

### 6. EF Core projections — best с anonymous

```csharp
// ✅ EF translates to SQL
var summaries = db.Orders
    .Select(o => new { o.Id, o.Total })
    .ToList();
// SELECT Id, Total FROM Orders

// Но ToList с DTO class — тоже OK, often clearer for production code
```

### 7. Nested anonymous + LINQ

```csharp
// Поддерживается
var data = orders
    .Select(o => new
    {
        Order = o,
        Items = o.Items.Select(i => new { i.Name, i.Price }).ToList()
    });

// EF Core translates это в один SQL c JOINами
```

### 8. Comparing different anon types

```csharp
var a = new { Name = "X" };
var b = new { Name = "X" };

a == b           // false — reference comparison! ⚠️
a.Equals(b)      // true — value comparison

// Anonymous type не overrides == operator
```

### 9. ToString format

```csharp
var p = new { Name = "John", Age = 30 };
Console.WriteLine(p);  
// { Name = John, Age = 30 }  ← format не customizable
```

### 10. Cannot define methods

```csharp
// ❌ Anonymous types — только properties
var p = new
{
    Name = "John",
    SayHi = () => Console.WriteLine("Hi")  // это не method!
                                              // это property типа Action
};

p.SayHi();  // works — но это property с Action value

// Если нужны methods — class / record
```

---

## 10. Best Practices

### When to use

- ✅ **LINQ projections** в одном method
- ✅ **GroupBy / Join** composite keys
- ✅ **Structured logging** context
- ✅ **Razor views** (partial — но лучше ViewModel)
- ✅ **Quick prototyping**

### When NOT

- ❌ **Public API** — define class/record
- ❌ **Cross-method** — define class/record
- ❌ **Mutable state** — class
- ❌ **Inheritance/polymorphism** — class

### Modern alternative: tuple deconstruction

```csharp
// Anonymous
var summary = orders.Select(o => new { o.Id, o.Total });
foreach (var s in summary) { Console.WriteLine($"{s.Id}: {s.Total}"); }

// Tuple — позволяет deconstruct
var summary = orders.Select(o => (Id: o.Id, Total: o.Total));
foreach (var (id, total) in summary) { Console.WriteLine($"{id}: {total}"); }
```

### Records — modern best для cross-method

```csharp
// Если данные используются в нескольких местах
public record OrderSummary(int OrderId, string Customer, decimal Total);

public List<OrderSummary> GetSummaries() =>
    db.Orders.Select(o => new OrderSummary(o.Id, o.Customer.Name, o.Total)).ToList();
```

См. [[modern-features|Modern C# Features]].

---

## 11. Cheat sheet

| Сценарий | Решение |
|----------|---------|
| LINQ Select projection | `select new { x, y }` |
| LINQ GroupBy composite | `g => new { g.Year, g.Month }` |
| LINQ Join result | `(a, b) => new { ... }` |
| Logging context | `logger.LogXxx("...", new { ... })` |
| Quick test data | `new { Name = "...", Age = ... }` |
| Cross-method return | record / class |
| Need mutability | class |
| Need inheritance | class |
| Public API | define type |
| Multiple values from method | tuple |

---

## 12. Decision tree

```
Что нужно?
│
├── LINQ projection в одном method?
│   → new { ... } anonymous
│
├── Composite key для Group/Join?
│   → new { ... } anonymous
│
├── Structured logging context?
│   → new { ... } anonymous
│
├── Multiple values из method?
│   → ValueTuple (Name: ..., Age: ...)
│
├── Cross-method shape с value equality?
│   → record
│
├── Need inheritance / polymorphism?
│   → class (или record для DDD)
│
└── Mutable temporary state?
    → class
```

---

## См. также

- [[csharp-basics|C# Basics]] — object initializers
- [[tuples-deconstruction|Tuples]] — близкий concept
- [[modern-features|Modern Features — records]]
- [[collections-linq|Collections & LINQ]] — anonymous heavily used
- [[../EFCore/queries-performance|EF Queries]] — projections
- [[functional-csharp|Functional C#]] — records detailed

## Reading list

- **Microsoft Docs — Anonymous Types** — learn.microsoft.com/dotnet/csharp/fundamentals/types/anonymous-types
- **Microsoft Docs — When to use** — learn.microsoft.com/dotnet/csharp/programming-guide/classes-and-structs/anonymous-types
- **Eric Lippert — Anonymous types history** — ericlippert.com
- **C# in Depth** — Jon Skeet (chapter про LINQ + anonymous)
