---
tags: [csharp, oop, junior, classes, inheritance, polymorphism, encapsulation, abstraction, records, solid]
level: Junior
date: 2026-05-04
---

# OOP и классы — четыре столпа объектно-ориентированного программирования

> **Способ организации кода через объекты, объединяющие данные и поведение.** Encapsulation, Inheritance, Polymorphism, Abstraction. Classes, structs, records, interfaces, abstract classes, virtual/override механика, SOLID для junior, composition over inheritance. Закрывает пробел: «знаю про классы, не понимаю когда interface vs abstract class и почему `sealed` по умолчанию».

---

## 0. Как читать этот файл

Если ты впервые работаешь с OOP в C# — читай разделы 1→6 подряд: получишь рабочую модель и поймёшь четыре столпа. Если уже пишешь classes, но непонятно про абстракции — раздел 7 (abstract classes), 8 (interfaces), 9 (когда что). Если строишь архитектуру — раздел 14 (DI), 15 (SOLID), 16 (composition over inheritance). Production-ready наставления — раздел 18 (best practices), 21 (pitfalls).

Все примеры самостоятельные. Cross-language якоря (`> [!info]-`) свёрнуты — раскрывай если переходишь из Java / Python / TypeScript / Kotlin / Rust. Interview-вопросы (`> [!question]-`) встроены рядом с теорией.

---

## 1. Что это, зачем и когда

### 1.1. Что такое OOP

**Объектно-ориентированное программирование** — парадигма, где код организован вокруг **объектов** — сущностей, объединяющих **данные** (поля, свойства) и **поведение** (методы).

```csharp
public class BankAccount
{
    // Данные
    private decimal _balance;
    private string _owner;
    
    public BankAccount(string owner, decimal initialBalance)
    {
        _owner = owner;
        _balance = initialBalance;
    }
    
    // Поведение
    public void Deposit(decimal amount)
    {
        if (amount <= 0) throw new ArgumentException("Amount must be positive");
        _balance += amount;
    }
    
    public bool Withdraw(decimal amount)
    {
        if (amount > _balance) return false;
        _balance -= amount;
        return true;
    }
    
    public decimal GetBalance() => _balance;
}

// Использование
var account = new BankAccount("Alice", 1000m);
account.Deposit(500m);
account.Withdraw(200m);
Console.WriteLine(account.GetBalance());   // 1300
```

Объект `account` — самостоятельная единица, которая знает свой баланс и умеет с ним работать. Внешний код не видит `_balance` напрямую — только через методы.

### 1.2. Четыре столпа OOP

1. **Encapsulation (инкапсуляция)** — скрытие деталей реализации за публичным API. Внешний код не зависит от внутреннего устройства.
2. **Inheritance (наследование)** — один класс получает поля и методы другого. `Dog` наследует от `Animal`.
3. **Polymorphism (полиморфизм)** — один интерфейс, разные реализации. `IPayment.Process()` работает для карты, перевода, криптовалюты.
4. **Abstraction (абстракция)** — работа с понятиями более высокого уровня. `ILogger.Log()` — не важно, пишет ли в файл, в консоль или в БД.

| Принцип | Что даёт | Пример |
|---------|----------|--------|
| **Encapsulation** | Защита invariant'ов, изменение реализации без breaking change | `_balance` private + `Deposit/Withdraw` валидируют |
| **Inheritance** | Reuse общей логики | `Animal` → `Dog`, `Cat` |
| **Polymorphism** | Гибкость через единый интерфейс | `IPayment.Process()` — много implementations |
| **Abstraction** | Decoupling от деталей | `ILogger` вместо `FileLogger` |

### 1.3. Главное правило

```
Используй OOP когда:
  - Есть несколько связанных операций над общим состоянием (BankAccount)
  - Состояние имеет invariants, которые нужно защищать (баланс не отрицательный)
  - Несколько вариантов поведения за общим интерфейсом (IPayment)
  - Сложная domain model с понятиями реального мира

НЕ используй OOP когда:
  - Простые stateless функции (math, formatting) — static methods
  - Pure data structures без логики — record / struct
  - Pipeline трансформации — LINQ / functional style
```

OOP — не серебряная пуля. C# поддерживает и procedural (static methods), и functional (LINQ, records, pattern matching) подходы. Хороший C# код **сочетает** все три парадигмы.

### 1.4. Эволюция: .NET 1.0 → C# 13

| Версия | Год | Что появилось |
|--------|-----|---------------|
| **.NET 1.0** | 2002 | classes, inheritance, interfaces, virtual/override |
| **.NET 2.0** | 2005 | generics |
| **.NET 3.5** | 2008 | LINQ, extension methods (расширения через статические методы) |
| **.NET 4.0** | 2010 | dynamic, contravariance/covariance |
| **C# 6.0** | 2015 | property initializers, expression-bodied members |
| **C# 7.0** | 2017 | tuples, pattern matching, deconstruction |
| **C# 8.0** | 2019 | default interface methods, nullable reference types |
| **C# 9.0** | 2020 | **records (record class)** |
| **C# 10** | 2021 | record struct, primary constructors для records |
| **C# 11** | 2022 | required properties, generic math interfaces |
| **C# 12** | 2023 | **primary constructors для classes** |
| **C# 13** | 2024 | params для коллекций, partial properties |

### 1.5. Class vs Struct vs Record — quick decision

| | class | struct | record class | record struct |
|---|-------|--------|--------------|---------------|
| Reference / value | Reference | Value | Reference | Value |
| Default mutability | Mutable | Mutable | Immutable (with `init`) | Mutable (с `readonly` immutable) |
| Equality | Reference | Value (по полям) | Value (по полям) | Value |
| `with`-expression | ❌ | ❌ | ✅ | ✅ |
| Inheritance | ✅ | ❌ (sealed) | ✅ (только records) | ❌ |
| Use case | Domain entities, services | Performance-critical small data | DTO, value objects | Lightweight value objects |

> [!info]- Если ты знаешь Java / Python / TypeScript / Kotlin / Rust
> **Java:** очень близко — class, interface, abstract class, inheritance с `extends`/`implements`. Главные отличия: C# имеет structs (Java только classes), records (Java records появились в 14), sealed по умолчанию (Java — open). Patterns одинаковые.
>
> **Python:** classes есть, но multiple inheritance (C# только single + multiple interfaces), нет access modifiers (только convention `_protected`, `__private`), duck typing (нет необходимости в interfaces). Concept похож, реализация свободнее.
>
> **TypeScript:** structural typing (если объект имеет поля, он совместим с интерфейсом, без явного `implements`). C# nominal — нужен явный `implements`. Otherwise — близко: classes, abstract, interfaces.
>
> **Kotlin:** очень близко к C#. data class ↔ record class, sealed class в Kotlin = sum type (как F# records, скоро в C# discriminated unions). open/final в Kotlin ↔ sealed/virtual в C# (Kotlin: classes final by default, C#: sealable).
>
> **Rust:** нет classes. Есть structs + traits (как struct + interface). Нет наследования — composition only. C# OOP более гибкий, Rust строго functional+procedural.

> [!question]- Интервью: что такое OOP и какие 4 столпа?
> OOP — парадигма, где код организован вокруг **объектов** (data + behavior). 4 столпа: 1) **Encapsulation** — скрытие реализации, доступ через public API, защита invariants. 2) **Inheritance** — производный класс получает поля/методы базового, reuse code. 3) **Polymorphism** — один интерфейс, разные реализации (virtual dispatch). 4) **Abstraction** — работа с понятиями более высокого уровня через interfaces / abstract classes. C# поддерживает OOP полностью, но не requires — можно писать procedural (static methods) или functional (LINQ, records). Хороший C# код смешивает три парадигмы.

---

## 2. Класс — основы

### 2.1. Минимальный класс

```csharp
public class Person
{
    public string Name;        // public field — anti-pattern, см. раздел 3
    public int Age;
}

var p = new Person();
p.Name = "Alice";
p.Age = 30;
```

Класс — **template** для создания объектов. `new Person()` создаёт **instance** (объект) на heap. `p` — reference на объект.

### 2.2. Поля, свойства, методы, конструкторы

```csharp
public class Product
{
    // Поле (field) — хранит state, обычно private
    private decimal _price;
    private int _stock;
    
    // Свойство (property) — public access с getter/setter
    public string Name { get; set; } = "";
    public string Sku { get; init; } = "";   // init — set только в constructor / object initializer
    
    // Свойство с backing field
    public decimal Price
    {
        get => _price;
        set
        {
            if (value < 0) throw new ArgumentException("Price cannot be negative");
            _price = value;
        }
    }
    
    // Конструктор
    public Product(string name, string sku, decimal price, int stock)
    {
        Name = name;
        Sku = sku;
        Price = price;   // через setter — валидация работает
        _stock = stock;
    }
    
    // Метод
    public void RestockBy(int amount)
    {
        if (amount <= 0) throw new ArgumentException("Amount must be positive");
        _stock += amount;
    }
    
    public int CurrentStock => _stock;   // expression-bodied property
}

// Использование
var product = new Product("Widget", "SKU-001", 19.99m, 100);
product.RestockBy(50);
Console.WriteLine(product.CurrentStock);   // 150
```

### 2.3. Auto-property (auto-implemented property)

```csharp
public class Person
{
    public string Name { get; set; } = "";       // auto-property
    public int Age { get; set; }
    public DateTime BirthDate { get; init; }      // init-only — set в constructor
    public string FullName { get; private set; } = "";   // private setter
    public bool IsAdult => Age >= 18;             // computed
}
```

Compiler автоматически генерирует backing field. Чаще всего достаточно auto-property — без `_field` boilerplate.

### 2.4. Primary constructors (C# 12+)

```csharp
// Старый стиль
public class Customer
{
    private readonly string _email;
    private readonly DateTime _registeredAt;
    
    public Customer(string email, DateTime registeredAt)
    {
        _email = email;
        _registeredAt = registeredAt;
    }
}

// C# 12+ — primary constructor
public class Customer(string email, DateTime registeredAt)
{
    public string Email => email;
    public DateTime RegisteredAt => registeredAt;
}
```

`(string email, DateTime registeredAt)` — primary constructor параметры. Доступны во всех members класса (как captured variables в local function).

### 2.5. `this` keyword

```csharp
public class Order
{
    private int _id;
    
    public Order(int id)
    {
        _id = id;          // ссылка на поле
        this._id = id;      // эквивалент с явным this
    }
    
    public Order WithDoubledId()
    {
        return new Order(this._id * 2);   // явный this для clarity
    }
    
    public bool Equals(Order other) => other._id == this._id;
}
```

`this` — reference на текущий instance. Обычно implicit, явный `this.` нужен только для disambiguation (parameter same name as field) или readability.

### 2.6. Static members

```csharp
public class MathHelper
{
    public static double Pi = 3.14159;       // shared между всеми instances
    public static readonly DateTime Epoch = new(1970, 1, 1);
    
    public static double CircleArea(double radius)   // static method
        => Pi * radius * radius;
    
    private MathHelper() { }   // private constructor — не инстанцируется
}

// Использование без instance
double area = MathHelper.CircleArea(5);
```

Static members принадлежат **классу**, не instance. Не имеют доступа к instance state (`this`).

### 2.7. const vs static readonly

```csharp
public class Config
{
    public const int MaxRetries = 3;                   // compile-time, embedded in caller
    public static readonly TimeSpan DefaultTimeout = TimeSpan.FromSeconds(30);   // runtime
    public static readonly DateTime AppStartTime = DateTime.UtcNow;   // тоже runtime
}
```

`const` — compile-time constant, hardcoded в caller assemblies (опасно при breaking change в библиотеке). `static readonly` — initialized один раз в runtime, безопаснее для public API.

### 2.8. Объект на heap, reference на stack

```csharp
var p1 = new Person { Name = "Alice" };   // объект на heap, p1 — reference на stack
var p2 = p1;                                 // p2 указывает на тот же объект
p2.Name = "Bob";
Console.WriteLine(p1.Name);                  // "Bob" — same object!

p2 = null;
Console.WriteLine(p1.Name);                  // "Bob" — p1 still valid
```

Class — reference type. Несколько переменных могут ссылаться на один и тот же объект. Изменения через one ref видны через others.

> [!question]- Интервью: чем отличаются field, property и method?
> **Field** — переменная класса, хранит state. Обычно private (`_field`), доступ через property. **Property** — синтаксический сахар над getter/setter методами, который выглядит как поле. Может иметь validation в setter. Auto-property `{ get; set; }` — compiler генерирует backing field автоматически. **Method** — функция класса, выполняет behavior. Принимает параметры, возвращает результат, может изменять state. Best practice: fields private, properties public для access, methods для behavior.

---

## 3. Encapsulation deep

### 3.1. Access modifiers

| Modifier | Видимость |
|----------|-----------|
| **public** | Везде |
| **internal** | В том же assembly (обычно проект) |
| **protected** | В классе и derived classes |
| **protected internal** | Combination — assembly OR derived |
| **private protected** | Derived class в том же assembly (intersection) |
| **private** | Только в этом классе |
| **file** (C# 11+) | Только в этом файле |

```csharp
public class BankAccount
{
    private decimal _balance;              // только этот класс
    protected string _accountId = "";       // этот + наследники
    internal int InternalId;                // в этом assembly
    public string Owner { get; set; } = ""; // везде
}
```

### 3.2. Default modifiers

```csharp
class Foo { }              // default — internal
public class Bar { }       // явно public

class Container
{
    int _field;             // default — private (для членов класса)
    public int Public;
}
```

- **Top-level types** (class, struct, interface, enum) — default `internal`.
- **Class members** — default `private`.

### 3.3. Properties vs Public Fields

```csharp
// ❌ Public field — anti-pattern
public class User
{
    public string Name;   // изменяется без validation, нет интерфейса
    public int Age;
}

// ✅ Properties — encapsulation
public class User
{
    public string Name { get; set; } = "";
    public int Age
    {
        get;
        set
        {
            if (value < 0) throw new ArgumentException("Age cannot be negative");
            field = value;   // C# 13: contextual keyword 'field' instead of backing
        }
    }
}
```

Properties дают:
1. **Validation** в setter.
2. **Лазейку** для refactoring (переходить с auto-property на computed без breaking change).
3. **Interface compatibility** — interfaces могут декларировать только properties, не fields.
4. **Data binding** — UI frameworks ожидают properties.

### 3.4. Read-only properties

```csharp
public class Order
{
    public int Id { get; init; }                   // set только в constructor / object initializer
    public DateTime CreatedAt { get; }              // get-only — set только в constructor через `=`
    public List<OrderItem> Items { get; } = [];    // mutable list, immutable reference
    public decimal TotalPrice => Items.Sum(i => i.Price);   // computed, без backing field
    
    public Order(int id)
    {
        Id = id;
        CreatedAt = DateTime.UtcNow;   // set в constructor — OK
    }
}

// Использование
var order = new Order(42)
{
    // Id = 99   // ❌ init — нельзя после constructor
};
order.Items.Add(new OrderItem(...));   // ✅ list mutable
// order.CreatedAt = DateTime.UtcNow;   // ❌ get-only
```

`{ get; init; }` (C# 9+) — set только в constructor / object initializer. Иммутабельность после создания.

### 3.5. Required properties (C# 11+)

```csharp
public class User
{
    public required string Email { get; init; }
    public required string Name { get; init; }
    public DateTime CreatedAt { get; init; } = DateTime.UtcNow;
}

// ❌ Compile error — required property not set
// var u = new User();
// var u = new User { Name = "Alice" };

// ✅
var u = new User
{
    Email = "alice@example.com",
    Name = "Alice"
};
```

`required` — caller обязан проинициализировать в object initializer. Альтернатива constructor с обязательными параметрами.

### 3.6. Backing field через `field` keyword (C# 13+)

```csharp
public string Name
{
    get;
    set => field = value?.Trim() ?? "";   // 'field' — backing field
}
```

C# 13 contextual keyword `field` даёт доступ к auto-generated backing field без явного `_name`. Уменьшает boilerplate.

### 3.7. Private setter — controlled mutation

```csharp
public class User
{
    public string Name { get; private set; } = "";
    public string Email { get; private set; } = "";
    
    public void Rename(string newName)
    {
        // Только через метод можно изменить
        if (string.IsNullOrWhiteSpace(newName))
            throw new ArgumentException("Name required");
        Name = newName;
    }
}

var user = new User();
user.Name = "Alice";   // ❌ — private setter
user.Rename("Alice");  // ✅
```

Private setter — public read, private write. Изменение только через методы класса.

### 3.8. Why encapsulation matters

```csharp
// Без encapsulation — invariants могут сломаться
public class Account
{
    public decimal Balance;
}

var acc = new Account { Balance = 100 };
acc.Balance = -1000;   // отрицательный баланс — invalid state!

// С encapsulation — invariants защищены
public class Account
{
    private decimal _balance;
    public decimal Balance => _balance;
    
    public void Deposit(decimal amount)
    {
        if (amount <= 0) throw new ArgumentException();
        _balance += amount;
    }
    
    public void Withdraw(decimal amount)
    {
        if (amount <= 0 || amount > _balance) throw new InvalidOperationException();
        _balance -= amount;
    }
}
```

Класс гарантирует invariants. Внешний код не может пройти за границы дозволенного.

> [!question]- Интервью: чем properties лучше public fields?
> 1) **Validation** — setter может проверить value перед assignment. 2) **Refactoring без breaking change** — auto-property можно превратить в computed без перекомпиляции callers. 3) **Interface compatibility** — interfaces декларируют properties, не fields. 4) **Data binding** — UI frameworks (WPF, Blazor) подписываются на property changes. 5) **Debugger / serialization** работают с properties по convention. Public fields — anti-pattern в C# (см. naming conventions). Используй `{ get; set; }` минимум, validation в setter / private setter / `init` / `required` для строгих контрактов.

---

## 4. Constructors deep

### 4.1. Default constructor

```csharp
public class Foo
{
    // Если нет explicit constructor — compiler генерирует public Foo() { }
}

var foo = new Foo();
```

Compiler генерирует parameterless public constructor если нет других. **Если есть** хоть один — default не генерируется автоматически.

### 4.2. Parameterized constructor

```csharp
public class Person
{
    public string Name { get; }
    public int Age { get; }
    
    public Person(string name, int age)
    {
        Name = name;
        Age = age;
    }
}

var p = new Person("Alice", 30);
// var p2 = new Person();   // ❌ Compile error — нет parameterless constructor
```

### 4.3. Constructor overloading

```csharp
public class Order
{
    public Order() { }                                        // 0 args
    public Order(int customerId) : this(customerId, []) { }   // 1 arg → delegate to 2-arg
    public Order(int customerId, List<OrderItem> items)        // 2 args
    {
        CustomerId = customerId;
        Items = items;
        CreatedAt = DateTime.UtcNow;
    }
    
    public int CustomerId { get; }
    public List<OrderItem> Items { get; } = [];
    public DateTime CreatedAt { get; }
}
```

`: this(args)` — call other constructor этого класса. Избегает дубликации.

### 4.4. Constructor chaining `: base(...)`

```csharp
public class Animal
{
    public string Name { get; }
    
    public Animal(string name)
    {
        Name = name;
    }
}

public class Dog : Animal
{
    public string Breed { get; }
    
    public Dog(string name, string breed) : base(name)
    {
        Breed = breed;
    }
}
```

`: base(...)` — call base class constructor. Required если base не имеет parameterless constructor.

### 4.5. Static constructor

```csharp
public class Config
{
    public static readonly Dictionary<string, string> Settings;
    
    static Config()
    {
        // Вызывается один раз при первом access к классу
        Settings = LoadFromFile("config.json");
    }
}
```

Static constructor — для initialization static members, requires no parameters, никогда не вызывается напрямую. Thread-safe — runtime гарантирует один call.

### 4.6. Object initializer

```csharp
public class User
{
    public string Name { get; set; } = "";
    public int Age { get; set; }
    public string Email { get; set; } = "";
}

// Object initializer — после constructor, но в одном expression
var user = new User
{
    Name = "Alice",
    Age = 30,
    Email = "alice@example.com"
};

// Эквивалент
var user2 = new User();
user2.Name = "Alice";
user2.Age = 30;
user2.Email = "alice@example.com";
```

Object initializer работает с `set` или `init` accessors. Удобно когда constructor не принимает все нужные fields.

### 4.7. Collection initializer

```csharp
var list = new List<int> { 1, 2, 3 };       // Add(1), Add(2), Add(3)
var dict = new Dictionary<string, int>
{
    { "a", 1 },
    { "b", 2 }
};
// или с C# 6+
var dict2 = new Dictionary<string, int>
{
    ["a"] = 1,
    ["b"] = 2
};

// C# 12+ — collection expressions
List<int> nums = [1, 2, 3];
int[] arr = [1, 2, 3];
HashSet<string> set = ["a", "b"];
```

### 4.8. Primary constructors (C# 12+)

```csharp
// Без primary
public class Customer
{
    private readonly string _email;
    private readonly DateTime _registeredAt;
    
    public Customer(string email, DateTime registeredAt)
    {
        _email = email;
        _registeredAt = registeredAt;
    }
    
    public string GetDescription() => $"{_email} since {_registeredAt:d}";
}

// С primary (C# 12+)
public class Customer(string email, DateTime registeredAt)
{
    public string Email { get; } = email;       // expose как property
    public DateTime RegisteredAt { get; } = registeredAt;
    
    public string GetDescription() => $"{email} since {registeredAt:d}";
    // email и registeredAt доступны во всех методах
}
```

Primary constructor параметры доступны во всех members. Если хочешь expose как public — assign в property: `public string Email { get; } = email;`.

### 4.9. Required + primary constructor combo

```csharp
public class Order(int id, DateTime createdAt)
{
    public int Id => id;
    public DateTime CreatedAt => createdAt;
    
    public required string Description { get; init; }
    public List<string> Tags { get; init; } = [];
}

var order = new Order(42, DateTime.UtcNow)
{
    Description = "Important order",
    Tags = ["urgent", "vip"]
};
```

Constructor для core identity, `init`/`required` для optional configuration.

### 4.10. Pitfall: throwing в constructor

```csharp
public class Service
{
    public Service(IRepository repo)
    {
        if (repo == null) throw new ArgumentNullException(nameof(repo));
        // ... initialization
    }
}
```

Throwing в constructor OK — но осторожно. Если throw происходит **после** того как объект частично initialized, finalizer (если есть) может вызваться с инвалидным state.

> [!question]- Интервью: что такое constructor chaining?
> Способ вызвать другой constructor из текущего. Два варианта: 1) `: this(...)` — call **другого** constructor этого класса. Используется для избежания duplication между overloads. 2) `: base(...)` — call constructor **базового** класса. Required если базовый не имеет parameterless constructor. Chain выполняется до тела current constructor — initialization идёт от базового вверх. Static constructor нельзя вызвать напрямую — runtime сам вызывает один раз перед первым access к классу.

---

## 5. Inheritance basics

### 5.1. Базовый syntax

```csharp
public class Animal
{
    public string Name { get; set; } = "";
    
    public void Eat() => Console.WriteLine($"{Name} is eating");
    public void Sleep() => Console.WriteLine($"{Name} is sleeping");
}

public class Dog : Animal   // : означает "наследует от"
{
    public string Breed { get; set; } = "";
    
    public void Bark() => Console.WriteLine($"{Name} says Woof!");
}

// Использование
var dog = new Dog { Name = "Rex", Breed = "Labrador" };
dog.Eat();    // унаследовано от Animal
dog.Sleep();   // унаследовано
dog.Bark();    // собственный метод
```

`Dog` имеет всё, что `Animal` (поля, properties, methods), плюс свои добавления.

### 5.2. Single inheritance

C# поддерживает только **single inheritance** для classes — один base class:

```csharp
public class A { }
public class B { }
public class C : A, B { }   // ❌ Compile error — multiple inheritance forbidden

public class D : A { }      // ✅ один base class
```

Multiple **interface** inheritance OK (раздел 8). Это решение C# (и Java) — избежать "diamond problem" multiple inheritance.

### 5.3. base — доступ к базовому

```csharp
public class Animal
{
    public string Name { get; }
    
    public Animal(string name) { Name = name; }
    
    public virtual string Describe() => $"Animal named {Name}";
}

public class Dog : Animal
{
    public string Breed { get; }
    
    public Dog(string name, string breed) : base(name)   // base constructor
    {
        Breed = breed;
    }
    
    public override string Describe()
    {
        var animalDesc = base.Describe();   // call base implementation
        return $"{animalDesc}, breed {Breed}";
    }
}
```

`base.Method()` — вызывает implementation базового класса. Используется в `override` методах для расширения базовой логики.

### 5.4. virtual / override

```csharp
public class Shape
{
    public virtual double Area() => 0;   // virtual — может быть overridden
    public virtual string Describe() => $"Shape with area {Area()}";
}

public class Circle : Shape
{
    public double Radius { get; init; }
    
    public override double Area() => Math.PI * Radius * Radius;   // override базовый
}

public class Square : Shape
{
    public double Side { get; init; }
    
    public override double Area() => Side * Side;
}

// Polymorphism
Shape[] shapes = [new Circle { Radius = 5 }, new Square { Side = 4 }];
foreach (var s in shapes)
{
    Console.WriteLine(s.Area());   // virtual dispatch — actual type used
}
// 78.539... (Circle), 16 (Square)
```

`virtual` в base + `override` в derived = polymorphism. Runtime выбирает правильный метод по actual type объекта.

### 5.5. new keyword — hiding (anti-pattern)

```csharp
public class Animal
{
    public string Speak() => "Some sound";
}

public class Dog : Animal
{
    public new string Speak() => "Woof!";   // hide, не override
}

Animal a = new Dog();
Console.WriteLine(a.Speak());        // "Some sound" — base method!

Dog d = new Dog();
Console.WriteLine(d.Speak());        // "Woof!" — derived method
```

`new` keyword — **hide** базовый метод, не overriding. Compiler выбирает по compile-time типу, не runtime.

**Anti-pattern** в большинстве случаев — поведение confusing. Используй `virtual`/`override` для polymorphism.

### 5.6. abstract methods

```csharp
public abstract class Shape
{
    public abstract double Area();   // нет implementation — derived MUST override
    
    public void Print() => Console.WriteLine($"Shape area: {Area()}");   // virtual via abstract
}

public class Circle : Shape
{
    public double Radius { get; init; }
    public override double Area() => Math.PI * Radius * Radius;
}

// var s = new Shape();   // ❌ Compile error — abstract class
var c = new Circle { Radius = 5 };
c.Print();   // OK
```

`abstract` метод — declared, но без body. Класс с abstract методом тоже должен быть `abstract` — нельзя инстанцировать. Derived классы обязаны override.

### 5.7. sealed — запрет наследования

```csharp
public sealed class Money   // нельзя наследовать
{
    public decimal Amount { get; }
    public string Currency { get; }
    public Money(decimal amount, string currency) { Amount = amount; Currency = currency; }
}

// public class FrozenMoney : Money { }   // ❌ Compile error

// Sealed для метода — нельзя override дальше
public class Animal
{
    public virtual void Speak() => Console.WriteLine("...");
}

public class Dog : Animal
{
    public sealed override void Speak() => Console.WriteLine("Woof!");
}

public class GoldenRetriever : Dog
{
    // public override void Speak() { }   // ❌ — sealed в Dog
}
```

`sealed` — запрет дальнейшего наследования. **Best practice:** classes по умолчанию sealed, virtual methods только когда extension point осознанный.

### 5.8. Inheritance vs Composition

```csharp
// Inheritance ("is-a")
public class Animal { public void Eat() { } }
public class Dog : Animal { public void Bark() { } }   // Dog IS A Animal

// Composition ("has-a")
public class Engine { public void Start() { } }
public class Car
{
    private readonly Engine _engine;
    public Car(Engine engine) { _engine = engine; }   // Car HAS A Engine
    public void Drive() { _engine.Start(); /* ... */ }
}
```

**Composition over inheritance** — design principle (раздел 16): предпочитай composition (объект имеет другой объект как поле) inheritance.

> [!question]- Интервью: чем `virtual`/`override` отличается от `new` keyword?
> `virtual` метод в base + `override` в derived — true polymorphism. Runtime выбирает implementation по **actual type** объекта (через vtable lookup). Если переменная typed как base, но содержит derived instance — вызывается derived method. `new` keyword — **hiding**: derived class объявляет свой метод с тем же именем, но это **другой** метод. Compiler выбирает по **compile-time** типу переменной. Anti-pattern в большинстве случаев. Использование `new` без явного нужды — bug. `virtual`/`override` — стандарт для extension points в OOP.

---

## 6. Polymorphism deep

### 6.1. Что такое polymorphism

**Polymorphism** — один интерфейс, разные implementations. Каллер не знает конкретный тип, работает через base / interface, runtime выбирает correct method.

```csharp
public abstract class Shape
{
    public abstract double Area();
}

public class Circle : Shape { public override double Area() => Math.PI * R * R; public double R; }
public class Square : Shape { public override double Area() => S * S; public double S; }

void PrintArea(Shape s) => Console.WriteLine(s.Area());

PrintArea(new Circle { R = 5 });    // 78.5...
PrintArea(new Square { S = 4 });     // 16
```

### 6.2. Virtual dispatch — как работает

Каждый объект на heap имеет указатель на **method table** (vtable) — таблицу указателей на actual methods. JIT генерирует код, который читает vtable в runtime для virtual call:

```
Memory layout:
+------------------+
| MethodTablePtr   | → vtable: { Area: Circle.Area, ToString: ... }
| field _radius    |
+------------------+
```

`Shape s = new Circle()`. При `s.Area()` JIT генерирует: `[s].MethodTable->Area()`. Это **late binding** — метод выбирается в runtime.

### 6.3. override vs new в действии

```csharp
public class Base
{
    public virtual void M() => Console.WriteLine("Base.M virtual");
    public void N() => Console.WriteLine("Base.N");
}

public class Derived : Base
{
    public override void M() => Console.WriteLine("Derived.M override");
    public new void N() => Console.WriteLine("Derived.N new");
}

Base b = new Derived();
b.M();   // "Derived.M override" — virtual dispatch
b.N();   // "Base.N" — static dispatch (compile-time type)
```

`override` использует vtable (runtime type). `new` keyword — static dispatch (compile-time type).

### 6.4. Polymorphism в коллекциях

```csharp
List<Shape> shapes = [
    new Circle { R = 5 },
    new Square { S = 4 }
];

double total = shapes.Sum(s => s.Area());   // правильный Area для каждого
```

Добавить `Triangle : Shape` — никто из callers не меняется. Это **Open-Closed Principle**.

### 6.5. Pattern matching для type checks

```csharp
Shape s = GetShape();

// is + cast (старый стиль)
if (s is Circle) { var c = (Circle)s; /* ... */ }

// Pattern matching (C# 7+)
if (s is Circle circle) Console.WriteLine(circle.R);

// Switch expression (C# 8+)
double area = s switch
{
    Circle c => Math.PI * c.R * c.R,
    Square sq => sq.S * sq.S,
    _ => 0
};
```

> [!question]- Интервью: как работает virtual dispatch?
> Каждый объект имеет указатель на method table (vtable) — таблицу указателей на actual methods. При вызове virtual метода runtime читает vtable, находит правильный method для actual type объекта. Это **late binding**. Non-virtual = **early binding** (compiler выбирает по compile-time типу). `override` использует vtable, `new` keyword — static dispatch.

---

## 7. Abstract classes

### 7.1. Что такое abstract class

```csharp
public abstract class Animal
{
    public string Name { get; }
    
    protected Animal(string name) { Name = name; }
    
    public void Sleep() => Console.WriteLine($"{Name} is sleeping");   // shared
    public abstract string Speak();   // derived MUST implement
    public virtual string Describe() => $"Animal named {Name}";        // override optional
}

public class Dog : Animal
{
    public Dog(string name) : base(name) { }
    public override string Speak() => "Woof!";
}

// var a = new Animal("X");   // ❌ — abstract
var d = new Dog("Rex");      // ✅
```

Abstract class — не instantiable, **base** для derived. Может содержать abstract methods (без implementation) и concrete methods (shared).

### 7.2. Когда abstract class

✅ **Используй когда:**
- Несколько подклассов разделяют общую implementation.
- Нужен constructor / shared state.
- Domain hierarchy с естественной "is-a" связью.

❌ **Не используй когда:**
- Только декларация контракта без shared logic → interface.
- Multiple inheritance нужно (только interfaces).

### 7.3. Template Method pattern

```csharp
public abstract class ReportGenerator
{
    // Template — fixed flow, derived изменяет шаги
    public string Generate()
    {
        var data = LoadData();
        var formatted = Format(data);
        return AddHeader(formatted);
    }
    
    private string AddHeader(string content) => $"=== Report ===\n{content}";
    
    protected abstract object LoadData();
    protected abstract string Format(object data);
}

public class SalesReport : ReportGenerator
{
    protected override object LoadData() => /* fetch sales */;
    protected override string Format(object data) => /* format as table */;
}
```

Classic OOP pattern для extensible workflows.

### 7.4. Constructor в abstract class

```csharp
public abstract class Animal
{
    public string Name { get; }
    
    protected Animal(string name)   // protected — только derived
    {
        if (string.IsNullOrWhiteSpace(name)) throw new ArgumentException();
        Name = name;
    }
}
```

`protected` — доступ derived только. Abstract класс не instantiated напрямую, только через derived.

> [!question]- Интервью: чем abstract class отличается от обычного?
> 1) **Не instantiable** — `new AbstractClass()` compile error. 2) Может содержать **abstract methods** без implementation, derived обязан override. 3) Constructor обычно `protected`. 4) Хранит shared state и provides shared methods. Используй для domain hierarchy с "is-a" + shared logic. Альтернатива — interface (только contract, или с default implementations C# 8+).

---

## 8. Interfaces

### 8.1. Что такое interface

```csharp
public interface IRepository<T>
{
    Task<T?> GetByIdAsync(int id);
    Task<List<T>> GetAllAsync();
    Task AddAsync(T item);
}

public class UserRepository : IRepository<User>
{
    private readonly DbContext _db;
    public UserRepository(DbContext db) { _db = db; }
    
    public async Task<User?> GetByIdAsync(int id) => await _db.Users.FindAsync(id);
    public async Task<List<User>> GetAllAsync() => await _db.Users.ToListAsync();
    public async Task AddAsync(User item) { _db.Users.Add(item); await _db.SaveChangesAsync(); }
}

IRepository<User> repo = new UserRepository(db);   // dependency inversion
```

Interface — контракт без implementation. Класс implements interface — обязан предоставить все методы.

### 8.2. Multiple interface inheritance

```csharp
public interface IReader { string Read(); }
public interface IWriter { void Write(string s); }

public class FileSystem : IReader, IWriter   // multiple interfaces OK
{
    public string Read() => File.ReadAllText("data.txt");
    public void Write(string s) => File.WriteAllText("data.txt", s);
}
```

Class может implement множество interfaces. Multiple class inheritance — нет.

### 8.3. Default interface methods (C# 8+)

```csharp
public interface ILogger
{
    void Log(string message);
    
    // Default implementation — bonus для callers
    void LogError(string error) => Log($"[ERROR] {error}");
    void LogInfo(string info) => Log($"[INFO] {info}");
}

public class ConsoleLogger : ILogger
{
    public void Log(string message) => Console.WriteLine(message);
    // LogError, LogInfo — наследуются default
}
```

C# 8+ — interfaces могут иметь implementation. Используй для **non-breaking extension** existing interfaces.

### 8.4. Explicit interface implementation

```csharp
public interface IReader { void Read(); }
public interface IWriter { void Read(); }   // тот же name

public class FileSystem : IReader, IWriter
{
    void IReader.Read() => Console.WriteLine("Reader.Read");   // explicit
    void IWriter.Read() => Console.WriteLine("Writer.Read");
}

((IReader)fs).Read();   // "Reader.Read"
((IWriter)fs).Read();   // "Writer.Read"
```

Explicit — для разрешения name collision или скрытия от direct access.

### 8.5. Interface properties

```csharp
public interface IIdentifiable
{
    int Id { get; }                          // get-only
    string Name { get; set; }                  // read/write
    DateTime CreatedAt { get; init; }          // init only (C# 9+)
}
```

Interface декларирует contract. Implementation choice — у класса.

> [!question]- Интервью: чем interface отличается от abstract class в C# 8+?
> Раньше interface — только декларация. C# 8+ добавил **default interface methods**. Различия: 1) Class может **implement множество** interfaces, но **наследовать только один** abstract class. 2) Interface не имеет instance fields, constructor. 3) Abstract class — для domain hierarchy с shared state. Interface — для multiple capabilities, decoupling, DI. Default implementations используются для non-breaking extension existing interfaces.

---

## 9. Interface vs abstract class — when which

### 9.1. Decision matrix

```
Multiple inheritance нужно?      → Interface
Shared state (fields)?            → Abstract class
Shared implementation?            → Abstract class (или default interface methods если minimal)
Domain "is-a" hierarchy?          → Abstract class (Animal → Dog)
Capability / "can-do"?            → Interface (IDisposable, IComparable)
Decoupling (DI, mocking)?         → Interface
Несколько unrelated implementations? → Interface
```

### 9.2. Interface — для capability

```csharp
public interface IDisposable { void Dispose(); }
public interface ISerializable { string Serialize(); }
public interface IComparable<T> { int CompareTo(T other); }

public class Resource : IDisposable, ISerializable, IComparable<Resource>
{
    public void Dispose() { /* ... */ }
    public string Serialize() => /* ... */;
    public int CompareTo(Resource? other) => /* ... */;
}
```

Один класс combine разные capabilities — independent contracts.

### 9.3. Abstract class — для hierarchy с shared state

```csharp
public abstract class Animal
{
    public string Name { get; }
    public DateTime BornAt { get; }
    
    protected Animal(string name) { Name = name; BornAt = DateTime.UtcNow; }
    
    public abstract string Speak();
    public virtual string Describe() => $"{GetType().Name} named {Name}";
}

public class Dog : Animal { /* ... */ }
public class Cat : Animal { /* ... */ }
```

Animal hierarchy share `Name`, `BornAt`, `Describe()`. Abstract — natural choice.

### 9.4. Combining — interface + abstract class

```csharp
public interface ITrackable
{
    DateTime CreatedAt { get; }
    DateTime? UpdatedAt { get; }
}

public abstract class Entity : ITrackable
{
    public DateTime CreatedAt { get; protected set; } = DateTime.UtcNow;
    public DateTime? UpdatedAt { get; protected set; }
    
    public void MarkUpdated() => UpdatedAt = DateTime.UtcNow;
}

public class Order : Entity { /* ... */ }
```

Abstract class реализует interface, derived наследуют оба.

### 9.5. Best practice — start with interface

Microsoft рекомендация: для **public API** в библиотеках — start with interface. Можно добавить abstract base later как helper.

```csharp
public interface IUserRepository
{
    Task<User?> GetByIdAsync(int id);
}

public abstract class UserRepositoryBase : IUserRepository
{
    public abstract Task<User?> GetByIdAsync(int id);
    
    // Helper logic не require loading data
    public virtual async Task<User> RequireByIdAsync(int id)
        => await GetByIdAsync(id) ?? throw new NotFoundException();
}
```

> [!question]- Интервью: когда interface, когда abstract class?
> **Interface** — когда: 1) Контракт без shared state. 2) Multiple inheritance нужен. 3) Decoupling для DI / тестов. 4) Capability ("can-do" — IComparable, IDisposable). **Abstract class** — когда: 1) Shared state (instance fields). 2) Domain hierarchy "is-a". 3) Template method pattern. 4) Constructor нужен. Best practice для library API: start with interface, add abstract helper class later.

---

## 10. Sealed и protection

### 10.1. sealed class

```csharp
public sealed class Money
{
    public decimal Amount { get; }
    public string Currency { get; }
}

// public class FrozenMoney : Money { }   // ❌ Compile error
```

`sealed` — нельзя наследовать. **Best practice:** classes по умолчанию sealed, открывай только когда extension осознан.

### 10.2. Зачем sealed

1. **Performance** — virtual call optimizations (devirtualization, inlining).
2. **Safety** — нельзя breaking change через override.
3. **Maintenance** — class evolution без worry о derived breakage.
4. **Communication** — sealed signals "не для extension".

### 10.3. sealed override

```csharp
public class Dog : Animal
{
    public sealed override void Speak() => Console.WriteLine("Woof!");
}

public class GoldenRetriever : Dog
{
    // public override void Speak() { }   // ❌ — sealed
}
```

`sealed override` — final implementation, дальше override запрещён.

> [!question]- Интервью: зачем sealed классы?
> 1) **Performance** — JIT может devirtualize virtual calls. 2) **Safety** — нельзя breaking change через override. 3) **API stability**. 4) **Communication** — signals "finalized class". Best practice (Microsoft, Roslyn analyzers): classes sealed by default, opens only when extension explicitly designed.

---

## 11. Records

### 11.1. record class

```csharp
public record User(int Id, string Name, string Email);

var user = new User(42, "Alice", "alice@example.com");
Console.WriteLine(user);   // "User { Id = 42, Name = Alice, Email = alice@example.com }"

var user2 = new User(42, "Alice", "alice@example.com");
user == user2;   // true! — value equality
```

Record (C# 9+) — синтаксический сахар: auto constructor, init-only properties, value equality, ToString, Deconstruct.

### 11.2. with expression

```csharp
var user = new User(42, "Alice", "alice@example.com");
var renamed = user with { Name = "Bob" };
// new instance с измененным Name

Console.WriteLine(user.Name);     // "Alice"
Console.WriteLine(renamed.Name);   // "Bob"
```

`with` — non-destructive mutation. Original остаётся.

### 11.3. record struct

```csharp
public record struct Point(double X, double Y);

var p1 = new Point(1, 2);
var p2 = p1;        // КОПИЯ (struct semantics)
p2 = p2 with { X = 5 };
```

Record struct (C# 10+) — value type с record features. Для small immutable data.

### 11.4. Record vs class

```csharp
// Class — manual equality
public class UserClass
{
    public int Id { get; init; }
    public string Name { get; init; } = "";
    
    public override bool Equals(object? obj) =>
        obj is UserClass u && u.Id == Id && u.Name == Name;
    public override int GetHashCode() => HashCode.Combine(Id, Name);
}

// Record — то же автоматически
public record UserRecord(int Id, string Name);
```

### 11.5. Когда record vs class

```
record когда:
  - DTO / API request/response
  - Value Object (Money, Coordinate)
  - Immutable domain events
  - Equality по value

class когда:
  - Domain entity с behavior + identity
  - Mutable state с invariants
  - Service / Repository / Controller
```

> [!question]- Интервью: чем record отличается от class?
> Record — синтаксический сахар над class с автоматически generated: constructor, init-only properties, value equality (Equals/GetHashCode/== по полям), ToString с named properties, Deconstruct, `with`-expression. Используй для immutable DTO / value objects / events. Class — для mutable entities с behavior. record struct — value type с record features.

---

## 12. Nested types и static classes

### 12.1. Nested class

```csharp
public class Outer
{
    public class Inner   // nested class
    {
        public string Name { get; set; } = "";
    }
    
    private class PrivateInner { }   // только в Outer
}

var inner = new Outer.Inner();   // через outer namespace
```

Nested type — type внутри другого type. Полезно для tightly coupled types.

### 12.2. Когда nested

✅ **Используй когда:**
- Helper class который имеет смысл только в context outer (Builder, EventArgs).
- Implementation detail, не нужный извне.
- Tightly coupled types.

❌ **Не используй когда:**
- Type имеет independent meaning.
- Используется во многих местах.

### 12.3. Static class

```csharp
public static class StringExtensions
{
    public static bool IsEmail(this string s) => /* ... */;
    public static string Trim(this string s, char c) => /* ... */;
}

public static class Math   // BCL example
{
    public static double Pi { get; } = 3.14159;
    public static double Sqrt(double x) => /* ... */;
}
```

`static class`:
- Не instantiable — только static members.
- Implicit sealed.
- Конструктор недоступен.
- Идеально для extension methods, utility helpers, mathematical functions.

### 12.4. Когда static class

✅ **Используй когда:**
- Только static utility methods (no state).
- Extension methods.
- Math / formatting helpers.

❌ **Не используй когда:**
- Нужен state (используй обычный class или DI singleton).
- Нужен polymorphism (interfaces / abstract classes).

> [!question]- Интервью: когда использовать static class?
> Когда класс содержит **только static methods** без state, типично utility / helper / extension methods. Примеры BCL: `Math`, `Console`, `File`, `String`. Static class — implicit sealed, нельзя instantiate. Альтернативы: 1) Если нужен state — обычный class + DI singleton. 2) Если нужен polymorphism — interface. 3) Если методы расширяют existing types — extension methods (которые сами в static class).

---

## 13. Dependency Injection как OOP best practice

### 13.1. Что такое DI

**Dependency Injection** — technique, при которой объект получает свои dependencies из externally (constructor, property, method), вместо создания их сам.

```csharp
// ❌ Без DI — tight coupling
public class OrderService
{
    private readonly DbContext _db = new MyDbContext();   // hardcoded!
    private readonly EmailService _email = new EmailService();
    
    public async Task PlaceOrder(Order order)
    {
        _db.Orders.Add(order);
        await _db.SaveChangesAsync();
        _email.Send(order.Email, "Order placed");
    }
}

// ✅ С DI — loose coupling
public class OrderService
{
    private readonly IDbContext _db;
    private readonly IEmailService _email;
    
    public OrderService(IDbContext db, IEmailService email)
    {
        _db = db;
        _email = email;
    }
    
    public async Task PlaceOrder(Order order)
    {
        _db.Orders.Add(order);
        await _db.SaveChangesAsync();
        await _email.SendAsync(order.Email, "Order placed");
    }
}
```

### 13.2. Зачем DI

1. **Testability** — заменить real dependencies на mocks.
2. **Decoupling** — class не знает specific implementations.
3. **Configurability** — switch implementations через config.
4. **Single Responsibility** — class делает свою работу, не управляет lifecycle dependencies.

### 13.3. DI container в ASP.NET Core

```csharp
// Program.cs
var builder = WebApplication.CreateBuilder(args);

// Регистрация
builder.Services.AddDbContext<AppDbContext>(opts => opts.UseSqlServer(connStr));
builder.Services.AddScoped<IUserRepository, UserRepository>();
builder.Services.AddSingleton<IEmailService, SendGridEmailService>();
builder.Services.AddTransient<IOrderService, OrderService>();

var app = builder.Build();
```

DI container resolves dependency graph automatically — controller получает сервисы через constructor.

### 13.4. Lifetimes

| Lifetime | Когда создаётся |
|----------|-----------------|
| **Transient** | Каждый раз при запросе (новый instance always) |
| **Scoped** | Один раз на HTTP request (или scope) |
| **Singleton** | Один раз на application lifetime |

Choice:
- **Transient** — lightweight stateless services.
- **Scoped** — DbContext, services per-request state.
- **Singleton** — config, caches, expensive-to-create.

### 13.5. Constructor injection — основной style

```csharp
public class OrderService(
    IDbContext db,
    IEmailService email,
    ILogger<OrderService> logger)   // C# 12+ primary constructor
{
    public async Task PlaceOrder(Order order)
    {
        logger.LogInformation("Placing order {OrderId}", order.Id);
        // ...
    }
}
```

Constructor injection — самый распространённый. Dependencies явные, required.

> [!question]- Интервью: что такое Dependency Injection?
> Pattern, при котором объект получает свои dependencies **извне** (через constructor, property, method), вместо создания их сам. Преимущества: 1) **Testability** — заменить на mocks. 2) **Decoupling** — class зависит от interfaces, не concrete implementations. 3) **Configurability** — switch implementations. 4) **Single Responsibility** — class делает работу, не управляет lifecycle. В ASP.NET Core встроен DI container с lifetimes: Transient (new always), Scoped (per-request), Singleton (per-application).

---

## 14. SOLID принципы (junior level)

### 14.1. S — Single Responsibility Principle

Класс должен иметь **одну причину** для изменения.

```csharp
// ❌ Class делает слишком много
public class User
{
    public void Save() { /* save to DB */ }
    public void SendEmail() { /* send notification */ }
    public string ToJson() { /* serialize */ }
    public bool ValidatePassword(string p) { /* validate */ }
}

// ✅ Разделено по responsibility
public class User { /* data + behavior */ }
public class UserRepository { public void Save(User u) { } }
public class EmailService { public void Send(User u, string subj) { } }
public class UserSerializer { public string ToJson(User u) { } }
public class PasswordValidator { public bool Validate(string p) { } }
```

### 14.2. O — Open-Closed Principle

Классы открыты для extension, закрыты для modification.

```csharp
// ❌ Modification — добавить новый payment requires changing class
public class PaymentProcessor
{
    public void Process(string type)
    {
        if (type == "card") { /* ... */ }
        else if (type == "transfer") { /* ... */ }
        else if (type == "crypto") { /* ... */ }   // нужно add new — modify class
    }
}

// ✅ Extension через polymorphism
public interface IPayment { void Process(); }
public class CardPayment : IPayment { public void Process() { } }
public class TransferPayment : IPayment { public void Process() { } }
public class CryptoPayment : IPayment { public void Process() { } }

public class PaymentProcessor
{
    public void Process(IPayment payment) => payment.Process();
}
// Add NewPayment : IPayment — никто не меняется
```

### 14.3. L — Liskov Substitution Principle

Объект subclass должен быть substitutable for объект base class без поломки behavior.

```csharp
// ❌ Square : Rectangle нарушает LSP
public class Rectangle
{
    public virtual int Width { get; set; }
    public virtual int Height { get; set; }
}

public class Square : Rectangle
{
    public override int Width
    {
        get => base.Width;
        set { base.Width = value; base.Height = value; }   // mutates Height too!
    }
}

void Test(Rectangle r)
{
    r.Width = 5;
    r.Height = 4;
    Assert.Equal(20, r.Width * r.Height);   // ✅ для Rectangle, ❌ для Square (16)!
}

// ✅ Square отдельный класс, не subclass Rectangle
public class Rectangle { public int Width; public int Height; }
public class Square { public int Side; }
```

LSP — derived class не должен нарушать contract base.

### 14.4. I — Interface Segregation Principle

Лучше много specific interfaces, чем один большой.

```csharp
// ❌ Один большой interface
public interface IWorker
{
    void Work();
    void Eat();      // не все workers едят (роботы)
    void Sleep();
}

public class HumanWorker : IWorker { /* implement all */ }
public class RobotWorker : IWorker
{
    public void Work() { }
    public void Eat() => throw new NotSupportedException();   // ⚠️ LSP violation!
    public void Sleep() => throw new NotSupportedException();
}

// ✅ Разделено
public interface IWorker { void Work(); }
public interface IFeeder { void Eat(); }
public interface ISleeper { void Sleep(); }

public class HumanWorker : IWorker, IFeeder, ISleeper { }
public class RobotWorker : IWorker { }
```

Interface должен fit конкретного user, не всё подряд.

### 14.5. D — Dependency Inversion Principle

High-level modules не должны зависеть от low-level. Оба должны зависеть от abstractions.

```csharp
// ❌ OrderService directly depends on SqlDatabase
public class OrderService
{
    private readonly SqlDatabase _db;   // concrete dependency
    public OrderService() { _db = new SqlDatabase(); }
}

// ✅ Зависит от abstraction
public interface IDatabase { void Save<T>(T entity); }

public class SqlDatabase : IDatabase { public void Save<T>(T entity) { } }
public class MongoDatabase : IDatabase { public void Save<T>(T entity) { } }

public class OrderService
{
    private readonly IDatabase _db;
    public OrderService(IDatabase db) { _db = db; }   // injected
}
```

DI = practical application of DIP.

> [!question]- Интервью: что такое SOLID?
> Acronym из 5 принципов OOP design (Robert Martin): **S**ingle Responsibility (класс — одна причина для изменения), **O**pen-Closed (открыт для extension, закрыт для modification — через polymorphism), **L**iskov Substitution (subclass substitutable for base без поломки), **I**nterface Segregation (специфичные interfaces лучше большого), **D**ependency Inversion (depend on abstractions, not concretions). Не догма, но ориентир для design. DI — practical application of DIP.

---

## 15. Composition over inheritance

### 15.1. Принцип

> Prefer **composition** (HAS-A) over **inheritance** (IS-A).

```csharp
// ❌ Inheritance — Car IS-A Vehicle IS-A MotorizedThing
public class MotorizedThing { public int Horsepower; }
public class Vehicle : MotorizedThing { public int Wheels; }
public class Car : Vehicle { public int Seats; }
// Tightly coupled — изменение Vehicle ломает Car

// ✅ Composition — Car HAS-A Engine, HAS-A Wheels
public class Engine { public int Horsepower; public void Start() { } }
public class WheelSet { public int Count; }
public class Car
{
    private readonly Engine _engine;
    private readonly WheelSet _wheels;
    
    public Car(Engine engine, WheelSet wheels) { _engine = engine; _wheels = wheels; }
    public void Drive() => _engine.Start();
}
```

### 15.2. Зачем composition

1. **Loose coupling** — Engine можно заменить.
2. **Testability** — mock Engine в Car tests.
3. **Flexibility** — Car может иметь EV Engine, Hybrid Engine, Diesel Engine.
4. **No diamond problem** — нет multiple inheritance.

### 15.3. Inheritance — когда уместно

Inheritance — для **true is-a** отношений с **stable hierarchy**:

```csharp
// ✅ Реальная "is-a" — Dog IS-A Animal
public abstract class Animal { public abstract string Speak(); }
public class Dog : Animal { public override string Speak() => "Woof"; }
public class Cat : Animal { public override string Speak() => "Meow"; }
```

Когда:
- Hierarchy stable (не часто меняется).
- Substitutability работает (LSP).
- Shared behavior + state.

### 15.4. Composition с DI

```csharp
public class OrderService
{
    private readonly IPaymentProcessor _payment;
    private readonly IInventoryService _inventory;
    private readonly INotificationService _notification;
    
    public OrderService(
        IPaymentProcessor payment,
        IInventoryService inventory,
        INotificationService notification)
    {
        _payment = payment;
        _inventory = inventory;
        _notification = notification;
    }
}
```

OrderService composes из меньших services. Каждый — independent, testable, swappable.

### 15.5. Strategy pattern — composition

```csharp
public interface ISortStrategy<T>
{
    void Sort(List<T> list);
}

public class QuickSort<T> : ISortStrategy<T> where T : IComparable<T> { }
public class MergeSort<T> : ISortStrategy<T> where T : IComparable<T> { }

public class SortService<T>
{
    private readonly ISortStrategy<T> _strategy;
    public SortService(ISortStrategy<T> strategy) { _strategy = strategy; }
    public void Process(List<T> list) => _strategy.Sort(list);
}
```

Strategy — algorithm injected, не inheritance.

> [!question]- Интервью: что значит "composition over inheritance"?
> Design principle: предпочитай composition (объект имеет другой объект как поле, "has-a") inheritance (subclass is-a base). Преимущества composition: 1) **Loose coupling** — компоненты swappable. 2) **Testability** — mock dependencies. 3) **Flexibility** — runtime composition. 4) Нет diamond problem. Inheritance уместен когда: hierarchy stable, true "is-a", LSP holds, shared state + behavior. В современных C# проектах composition + interfaces + DI — основной подход. Inheritance — редкая, точечная техника.

---

## 16. Common patterns

### 16.1. Factory

```csharp
public interface ILogger
{
    void Log(string msg);
}

public class ConsoleLogger : ILogger { public void Log(string msg) => Console.WriteLine(msg); }
public class FileLogger : ILogger { public void Log(string msg) { /* write to file */ } }

public static class LoggerFactory
{
    public static ILogger Create(string type) => type switch
    {
        "console" => new ConsoleLogger(),
        "file" => new FileLogger(),
        _ => throw new ArgumentException(nameof(type))
    };
}

ILogger logger = LoggerFactory.Create("console");
```

Factory — создаёт объекты по запросу, инкапсулирует logic выбора implementation.

### 16.2. Builder

```csharp
public class HttpRequest
{
    public string Url { get; set; } = "";
    public string Method { get; set; } = "GET";
    public Dictionary<string, string> Headers { get; } = new();
    public string? Body { get; set; }
}

public class HttpRequestBuilder
{
    private readonly HttpRequest _request = new();
    
    public HttpRequestBuilder WithUrl(string url) { _request.Url = url; return this; }
    public HttpRequestBuilder WithMethod(string method) { _request.Method = method; return this; }
    public HttpRequestBuilder WithHeader(string key, string value) { _request.Headers[key] = value; return this; }
    public HttpRequestBuilder WithBody(string body) { _request.Body = body; return this; }
    
    public HttpRequest Build() => _request;
}

var request = new HttpRequestBuilder()
    .WithUrl("https://api.example.com")
    .WithMethod("POST")
    .WithHeader("Content-Type", "application/json")
    .WithBody("{}")
    .Build();
```

Builder — fluent construction объекта с many optional parameters.

### 16.3. Strategy

```csharp
public interface ITaxCalculator { decimal Calculate(decimal amount); }

public class USTaxCalculator : ITaxCalculator
{
    public decimal Calculate(decimal amount) => amount * 0.08m;
}

public class EUTaxCalculator : ITaxCalculator
{
    public decimal Calculate(decimal amount) => amount * 0.20m;
}

public class Order
{
    private readonly ITaxCalculator _taxCalc;
    public Order(ITaxCalculator taxCalc) { _taxCalc = taxCalc; }
    public decimal TotalWithTax(decimal amount) => amount + _taxCalc.Calculate(amount);
}
```

Strategy — algorithm как dependency, swappable runtime.

### 16.4. Observer (events)

```csharp
public class OrderCreatedEventArgs : EventArgs { public Order Order { get; init; } = null!; }

public class OrderService
{
    public event EventHandler<OrderCreatedEventArgs>? OrderCreated;
    
    public void PlaceOrder(Order order)
    {
        // ... save
        OrderCreated?.Invoke(this, new OrderCreatedEventArgs { Order = order });
    }
}

// Subscribers
service.OrderCreated += (s, e) => Console.WriteLine($"New order: {e.Order.Id}");
service.OrderCreated += (s, e) => SendEmail(e.Order);
```

Observer через C# events — встроенный pattern. Subscribers подписываются, publisher эмитит.

### 16.5. Singleton (с осторожностью)

```csharp
public sealed class Cache
{
    private static readonly Lazy<Cache> _instance = new(() => new Cache());
    public static Cache Instance => _instance.Value;
    
    private Cache() { }   // private constructor
    
    private readonly Dictionary<string, object> _store = new();
    public T? Get<T>(string key) => _store.TryGetValue(key, out var v) ? (T)v : default;
    public void Set<T>(string key, T value) => _store[key] = value!;
}

// Use
Cache.Instance.Set("key", "value");
```

Singleton — один global instance. **В современном C# обычно через DI**: `services.AddSingleton<ICache, Cache>()`. Это лучше — testable, configurable.

> [!question]- Интервью: какие OOP design patterns используются в .NET?
> 1) **Factory** — `LoggerFactory.CreateLogger`, `HttpClientFactory`. 2) **Builder** — `StringBuilder`, `HttpRequestBuilder`, `WebApplication.CreateBuilder`. 3) **Strategy** — `IComparer<T>`, `IEqualityComparer<T>`, sorting algorithms. 4) **Observer** — events, `INotifyPropertyChanged`, `IObservable<T>`. 5) **Decorator** — middleware pipeline в ASP.NET Core. 6) **Adapter** — `Stream` adapters. 7) **Repository** — data access abstraction. 8) **Singleton** — обычно через DI container, не manual. Понимание patterns помогает читать BCL код и работать с frameworks.

---

## 17. Best practices

### 17.1. Class design

- ✅ **Single Responsibility** — один класс, одна задача.
- ✅ **`sealed` by default** — open только когда extension осознан.
- ✅ **Encapsulation** — fields private, public access через properties.
- ✅ **Immutability when possible** — `init` properties, records.
- ✅ **Required + init** для compile-time guarantees.
- ✅ **Primary constructor** для DI dependencies (C# 12+).
- ❌ **Public mutable fields** — anti-pattern.
- ❌ **God classes** — > 500 строк, > 10 dependencies — split.

### 17.2. Inheritance

- ✅ **Composition over inheritance** для большинства cases.
- ✅ **`virtual` only when extension intended**.
- ✅ **`abstract` для template methods**.
- ✅ **`sealed override`** для финального override.
- ❌ **`new` keyword** — anti-pattern (hiding).
- ❌ **Deep hierarchies** — > 3 уровней — пересмотри.

### 17.3. Interfaces

- ✅ **Interface для DI / abstractions**.
- ✅ **Multiple specific** лучше одного большого (ISP).
- ✅ **Default methods** для non-breaking extension.
- ✅ **`I` prefix** — convention.
- ❌ **Marker interfaces без members** — используй attributes.

### 17.4. Records

- ✅ **`record`** для DTO, value objects, events.
- ✅ **`record struct`** для small immutable value types.
- ❌ **`record` для domain entities с behavior** — обычный class.

### 17.5. SOLID

- ✅ **DIP через DI** — depend on interfaces.
- ✅ **OCP через polymorphism** — расширение через new types, не editing existing.
- ✅ **SRP** — split big classes.
- ❌ **LSP violations** — Square : Rectangle, etc.
- ❌ **God interface** (`IRepository<T>` с 30 methods).

### 17.6. Не делай

- ❌ Public mutable state.
- ❌ Inheritance для code reuse без is-a.
- ❌ `new` keyword для method hiding.
- ❌ Static state mutation.
- ❌ Constructor с side effects (DB calls).
- ❌ Raise exceptions из getters.

---

## 18. Decision tree

```
Что нужно?
│
├── Хранилище данных без поведения
│   ├── Immutable → record
│   ├── Mutable → class с auto-properties
│   └── Small value type → record struct или struct
│
├── Domain entity с поведением + identity
│   └── class
│
├── Контракт для DI / mocking
│   └── interface
│
├── Hierarchy с shared state
│   └── abstract class
│
├── Capability (can-do)
│   └── interface (IDisposable, IComparable)
│
├── Polymorphism для variations
│   ├── Stable hierarchy → abstract class + override
│   └── Multiple unrelated → interface + DI
│
├── Utility / extension methods
│   └── static class (или extension methods)
│
├── Singleton state
│   └── DI container + AddSingleton (не manual singleton)
│
├── Reuse через composition vs inheritance
│   ├── True "is-a" → inheritance
│   └── "has-a" → composition (preferred)
│
└── Архитектурный выбор
    ├── Layered → DI + interfaces для слоёв
    ├── DDD → Aggregate roots, Value Objects, Domain Events
    └── CQRS → Commands/Queries как records, Handlers как classes
```

---

## 19. Cheat sheet

```csharp
// === Class basics ===
public class Person
{
    public string Name { get; init; } = "";              // init-only property
    public int Age { get; private set; }                  // private setter
    public required string Email { get; init; }           // required (C# 11+)
    
    private int _counter;                                  // private field
    public const int MaxAge = 150;                         // const PascalCase
    public static readonly DateTime Epoch = new(1970, 1, 1);
    
    public Person(string name, int age)                    // constructor
    {
        Name = name;
        Age = age;
    }
    
    public void GrowOlder()                                // method
    {
        Age++;
    }
}

// === Primary constructor (C# 12+) ===
public class Customer(string email, IRepository repo)
{
    public string Email { get; } = email;
    public async Task SaveAsync() => await repo.SaveAsync(email);
}

// === Inheritance ===
public abstract class Shape
{
    public abstract double Area();                         // must override
    public virtual string Describe() => $"Shape: {Area()}"; // can override
}

public class Circle : Shape
{
    public double R { get; init; }
    public override double Area() => Math.PI * R * R;
}

// === Interfaces ===
public interface IRepository<T>
{
    Task<T?> GetByIdAsync(int id);
    Task SaveAsync(T item);
    
    // Default implementation (C# 8+)
    Task<T> RequireByIdAsync(int id) => GetByIdAsync(id) ?? throw new NotFoundException();
}

// === Sealed ===
public sealed class Money(decimal amount, string currency)
{
    public decimal Amount => amount;
    public string Currency => currency;
}

// === Record ===
public record User(int Id, string Name, string Email);
public record struct Point(double X, double Y);

var user = new User(1, "Alice", "a@x.com");
var renamed = user with { Name = "Bob" };

// === Pattern matching ===
double area = shape switch
{
    Circle c => Math.PI * c.R * c.R,
    Square s => s.Side * s.Side,
    _ => 0
};

// === DI ===
builder.Services.AddScoped<IUserRepository, UserRepository>();
builder.Services.AddSingleton<ILogger, ConsoleLogger>();

public class OrderService(IUserRepository users, ILogger logger)
{
    public async Task ProcessAsync(int userId) { /* ... */ }
}
```

---

## 20. Common pitfalls

### 20.1. Public mutable fields

```csharp
public class User
{
    public string Name;   // ❌ — нет validation
}
```

**Фикс:** properties.

### 20.2. `new` keyword вместо override

```csharp
public class Dog : Animal
{
    public new string Speak() => "Woof";   // ❌ hiding
}
```

**Фикс:** `virtual`/`override`.

### 20.3. Naked inheritance для code reuse

```csharp
public class StringList : List<string>   // ❌ inheritance чтобы добавить методы
{
    public string ToCsv() => string.Join(",", this);
}
```

**Фикс:** extension method.

```csharp
public static class StringListExtensions
{
    public static string ToCsv(this List<string> list) => string.Join(",", list);
}
```

### 20.4. God class

```csharp
public class UserService   // ❌ 50 methods, 15 dependencies
{
    public void Save(...) { }
    public void Email(...) { }
    public void GenerateReport(...) { }
    // ... ещё 47 методов
}
```

**Фикс:** split по responsibilities (UserRepository, EmailService, ReportGenerator).

### 20.5. Abstract class когда interface достаточен

```csharp
// ❌ Abstract без shared state / logic
public abstract class IRepository
{
    public abstract Task SaveAsync(...);
}
```

**Фикс:** interface.

### 20.6. Constructor с side effects

```csharp
public class UserService
{
    public UserService()
    {
        _users = LoadFromDatabase();   // ❌ — DB call in constructor
    }
}
```

**Фикс:** DI + lazy initialization, или static factory.

### 20.7. Throwing в Property getter

```csharp
public string Name
{
    get
    {
        if (_name == null) throw new InvalidOperationException();   // ❌ unexpected
        return _name;
    }
}
```

**Фикс:** ensure invariant в constructor, или возвращай null/default.

### 20.8. Static mutable state

```csharp
public class Counter
{
    public static int Count;   // ❌ — global mutable, race conditions
}
```

**Фикс:** instance state + DI singleton.

### 20.9. Inheritance из библиотечных classes

```csharp
public class MyDataContext : DbContext   // ⚠️ только если действительно нужно
{
    // BCL может изменить DbContext в next version → breaking change
}
```

**Фикс:** composition + delegation, или sealed extension.

### 20.10. Tight coupling без interfaces

```csharp
public class OrderService
{
    private readonly SqlServerRepository _repo;   // ❌ concrete dependency
    public OrderService() { _repo = new SqlServerRepository(); }
}
```

**Фикс:** depend on `IRepository<T>` через DI.

> [!question]- Интервью: топ-3 OOP ошибки junior разработчика?
> 1) **Public mutable fields** вместо properties — нет validation, нет interface compatibility, breaking change при refactoring. 2) **Inheritance для code reuse** без true "is-a" — tight coupling, LSP violations, fragile base class. Use composition + DI. 3) **God classes** — один class делает всё (save, email, report, validate). Split по SRP. Дополнительно: `new` keyword вместо `virtual`/`override`, static mutable state, constructor с side effects.

---

## 21. Practice — упражнения

### 21.1. Refactor из inheritance в composition

**Задача.** Refactor inheritance-heavy class в composition.

```csharp
// До
public class EmailUserService : UserService
{
    public override async Task RegisterAsync(User user)
    {
        await base.RegisterAsync(user);
        await SendWelcomeEmail(user);
    }
}

// После — composition
public class UserService
{
    private readonly IRepository<User> _repo;
    private readonly INotificationService? _notify;
    
    public UserService(IRepository<User> repo, INotificationService? notify = null)
    {
        _repo = repo;
        _notify = notify;
    }
    
    public async Task RegisterAsync(User user)
    {
        await _repo.AddAsync(user);
        if (_notify != null) await _notify.NotifyAsync(user, "Welcome!");
    }
}

// Или decorator pattern
public class NotifyingUserService : IUserService
{
    private readonly IUserService _inner;
    private readonly INotificationService _notify;
    
    public NotifyingUserService(IUserService inner, INotificationService notify)
    {
        _inner = inner;
        _notify = notify;
    }
    
    public async Task RegisterAsync(User user)
    {
        await _inner.RegisterAsync(user);
        await _notify.NotifyAsync(user, "Welcome!");
    }
}
```

**Разбор:** decorator оборачивает inner service, добавляет behavior без inheritance. Composition over inheritance в действии.

### 21.2. SOLID-ная архитектура

**Задача.** Спроектировать order processing с SOLID.

```csharp
// Interfaces (DIP)
public interface IOrderRepository { Task SaveAsync(Order order); }
public interface IInventoryService { Task<bool> ReserveAsync(int productId, int qty); }
public interface IPaymentProcessor { Task<PaymentResult> ProcessAsync(decimal amount, PaymentMethod method); }
public interface INotificationService { Task NotifyAsync(string email, string message); }

// Single Responsibility
public class OrderService(
    IOrderRepository orderRepo,
    IInventoryService inventory,
    IPaymentProcessor payment,
    INotificationService notify)
{
    public async Task<Order> PlaceAsync(PlaceOrderRequest request)
    {
        // 1. Reserve inventory
        var reserved = await inventory.ReserveAsync(request.ProductId, request.Quantity);
        if (!reserved) throw new InsufficientStockException();
        
        // 2. Process payment
        var paymentResult = await payment.ProcessAsync(request.Total, request.PaymentMethod);
        if (!paymentResult.IsSuccess) throw new PaymentFailedException();
        
        // 3. Save order
        var order = new Order(request.UserId, request.Total);
        await orderRepo.SaveAsync(order);
        
        // 4. Notify
        await notify.NotifyAsync(request.Email, "Order placed!");
        
        return order;
    }
}

// Open-Closed — новые payment methods через IPayment
public class StripePaymentProcessor : IPaymentProcessor { }
public class PayPalPaymentProcessor : IPaymentProcessor { }
public class CryptoPaymentProcessor : IPaymentProcessor { }
```

**Разбор:** все principles SOLID применены: SRP (OrderService только orchestrates), OCP (new payment via new IPaymentProcessor implementation), LSP (payment processors interchangeable), ISP (separate interfaces для repo/inventory/payment/notify), DIP (depends on interfaces, not concretions). DI инjects implementations.

### 21.3. Smart enum через sealed class

```csharp
public sealed class OrderStatus : IEquatable<OrderStatus>
{
    public static readonly OrderStatus Pending = new(1, "Pending", canTransitionTo: [2, 5]);
    public static readonly OrderStatus Paid = new(2, "Paid", canTransitionTo: [3, 5]);
    public static readonly OrderStatus Shipped = new(3, "Shipped", canTransitionTo: [4]);
    public static readonly OrderStatus Delivered = new(4, "Delivered", canTransitionTo: []);
    public static readonly OrderStatus Cancelled = new(5, "Cancelled", canTransitionTo: []);
    
    public int Id { get; }
    public string Name { get; }
    private readonly int[] _allowed;
    
    private OrderStatus(int id, string name, int[] canTransitionTo)
    {
        Id = id; Name = name; _allowed = canTransitionTo;
    }
    
    public bool CanTransitionTo(OrderStatus target) => _allowed.Contains(target.Id);
    public bool IsTerminal => _allowed.Length == 0;
    
    public bool Equals(OrderStatus? other) => other?.Id == Id;
    public override int GetHashCode() => Id;
    public override string ToString() => Name;
    
    public static IReadOnlyList<OrderStatus> All { get; } =
        [Pending, Paid, Shipped, Delivered, Cancelled];
}
```

**Разбор:** sealed class с static instances эмулирует enum + добавляет behavior. Encapsulation (state машина внутри), polymorphism (Equality), composition (CanTransitionTo возвращает behavior через data). См. подробно — [[enums-flags]].

---

## 22. Что читать дальше

1. **[[csharp-basics|C# Basics]]** — типы, переменные, value vs reference.
2. **[[naming-conventions|Naming]]** — как именовать classes / interfaces.
3. **[[modern-features|Modern Features]]** — records, pattern matching, primary constructors.
4. **[[generics-deep|Generics deep]]** — generic classes / methods / constraints.
5. **[[delegates-events|Delegates и Events]]** — Observer pattern.
6. **[[error-handling|Error Handling]]** — exception hierarchy.
7. **DDD — Domain-Driven Design** — Aggregate roots, Value Objects, Bounded Contexts.
8. **Design Patterns** — Gang of Four book.
9. **Clean Architecture** — слои + dependency rule.
10. **CQRS** — Commands/Queries separation.

---

## 23. См. также

- [[csharp-basics|C# Basics]] — основы
- [[naming-conventions|Naming Conventions]] — стиль для OOP
- [[modern-features|Modern Features]] — primary constructors, records
- [[generics-deep|Generics deep]] — generic classes
- [[delegates-events|Delegates и Events]] — Observer
- [[anonymous-types|Anonymous Types]] — projections
- [[error-handling|Error Handling]] — Exception hierarchy
- DDD — Eric Evans book
- Design Patterns — GoF
- Clean Architecture — Robert Martin
- DI containers — Microsoft.Extensions.DependencyInjection
- Refactoring — Martin Fowler

---

## 24. Reading list

- **Microsoft Docs — C# OOP** — learn.microsoft.com/dotnet/csharp/fundamentals/object-oriented
- **Microsoft Docs — Classes and objects** — learn.microsoft.com/dotnet/csharp/programming-guide/classes-and-structs/
- **Microsoft Docs — Interfaces** — learn.microsoft.com/dotnet/csharp/fundamentals/types/interfaces
- **Microsoft Docs — Records** — learn.microsoft.com/dotnet/csharp/whats-new/tutorials/records
- **Microsoft Docs — Inheritance** — learn.microsoft.com/dotnet/csharp/fundamentals/object-oriented/inheritance
- **Microsoft Docs — Polymorphism** — learn.microsoft.com/dotnet/csharp/fundamentals/object-oriented/polymorphism
- **Robert Martin — Clean Code (chapters on classes, objects)**
- **Robert Martin — Clean Architecture (SOLID, dependency rule)**
- **Eric Evans — Domain-Driven Design**
- **Gang of Four — Design Patterns**
- **Martin Fowler — Refactoring (2nd ed.)**
- **Steve Smith — Architecting Modern Web Applications** — learn.microsoft.com/dotnet/architecture/
- **Mark Seemann — Dependency Injection in .NET** — книга
- **Vaughn Vernon — Implementing DDD**
- **Steve Smith — Ardalis blog** — ardalis.com (DDD, SOLID, patterns)
- **Andrew Lock — Various OOP patterns** — andrewlock.net
- **Microsoft .NET Design Guidelines — Type Design** — learn.microsoft.com/dotnet/standard/design-guidelines/type
