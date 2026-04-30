---
tags: [csharp, basics, junior, fundamentals, variables, control-flow, methods]
level: Junior
date: 2026-04-30
---

# C# Basics — основы языка

> **Стартовая точка для Junior**. Всё что нужно знать чтобы начать писать C#: переменные, типы, операторы, циклы, условия, методы, классы. Каждая концепция объяснена с **примерами** и **зачем это нужно**.

---

## Что это, зачем и когда

### Что такое C#

**Язык программирования** от Microsoft. На нём пишут:
- Web сайты / API (ASP.NET Core)
- Desktop программы (WPF, Avalonia)
- Игры (Unity)
- Mobile apps (.NET MAUI)
- Microservices, cloud-native приложения
- ML модели (ML.NET)

C# работает на платформе **.NET** — runtime, который выполняет код. Кроссплатформенный (Windows / Linux / macOS).

### Структура программы

```csharp
// File: Program.cs
Console.WriteLine("Hello, World!");
```

Это **полная** программа в современном C# (top-level statements, .NET 6+). До .NET 6 нужно было:

```csharp
namespace MyApp
{
    class Program
    {
        static void Main(string[] args)
        {
            System.Console.WriteLine("Hello, World!");
        }
    }
}
```

Сейчас компилятор сам это генерирует под капотом из top-level кода.

### Запуск

```bash
dotnet new console -n MyApp    # создать новый проект
cd MyApp
dotnet run                      # запустить
```

См. [[../Infrastructure/project-setup|Project Setup]] для подробностей про проекты.

---

## 1. Переменные и типы

### Что такое переменная

**Именованная коробочка для значения.** Помнит что в неё положили.

```csharp
int age = 25;        // в коробочке "age" — число 25
string name = "John"; // в коробочке "name" — текст "John"
bool isActive = true; // в коробочке "isActive" — да/нет (true/false)
```

C# **типизированный** — переменная имеет тип, его не поменять.

```csharp
int age = 25;
age = "John";  // ❌ Compile error — int не может содержать string
```

### Объявление variables

```csharp
// Полная форма
int x = 5;
string greeting = "Hello";

// var — компилятор выводит тип сам
var x = 5;            // int (из 5)
var greeting = "Hello"; // string (из "Hello")
var date = DateTime.Now; // DateTime

// var нельзя без initial value
var x;  // ❌ error

// Без значения — нужно указать тип
int x;  // ✅ default value (0 для int)
x = 5;  // присваиваем позже
```

### Когда `var` vs explicit type

```csharp
// ✅ Используй var когда тип очевиден
var users = new List<User>();
var name = "John";
var count = 42;

// ✅ Explicit когда тип НЕ очевиден
int count = GetCount();
decimal price = CalculatePrice();
```

### Основные типы

#### Integer types — целые числа

| Type | Размер | Range |
|------|--------|-------|
| `byte` | 1 байт | 0 ... 255 |
| `short` | 2 байта | -32,768 ... 32,767 |
| **`int`** ⭐ | 4 байта | ±2.1 billion |
| **`long`** | 8 байт | ±9.2 quintillion |

**Default — `int`**. Используй `long` для timestamp в миллисекундах, IDs в больших системах.

```csharp
int count = 42;
long bigNumber = 9_000_000_000L;  // L suffix — long literal
                                  // подчеркивания — для readability
```

#### Floating-point — дробные

| Type | Размер | Точность |
|------|--------|----------|
| `float` | 4 байта | ~7 digits |
| **`double`** ⭐ | 8 байт | ~15-17 digits |
| **`decimal`** ⭐ | 16 байт | ~28-29 digits |

```csharp
float pi = 3.14f;        // f suffix
double e = 2.71828;       // double (default для дробных)
decimal price = 19.99m;   // m suffix (от Money)
```

> [!warning] Деньги — всегда `decimal`!
> `float` и `double` — **неточные** для денег. `0.1 + 0.2 != 0.3` в double!
> `decimal` — точные для финансов, медленнее в 10x.

```csharp
double a = 0.1 + 0.2;      // 0.30000000000000004 ⚠️
decimal b = 0.1m + 0.2m;   // 0.3 ✅
```

#### Boolean

```csharp
bool isActive = true;
bool isDeleted = false;
```

В C# **только** `true` / `false`. Нельзя `if (1)` как в C/JavaScript.

#### Char и string

```csharp
char letter = 'A';        // одинарные кавычки, один символ
string name = "Hello";    // двойные кавычки

// Escape sequences
string s = "Line 1\nLine 2";    // \n — новая строка
string path = "C:\\Users";       // \\ — backslash
string verbatim = @"C:\Users";   // @ — verbatim, без escape

// String interpolation
string greeting = $"Hello, {name}!";

// Multi-line raw string (C# 11+)
string json = """
    {
        "name": "John",
        "age": 30
    }
    """;
```

См. [[strings-regex|Strings и Regex]] для глубокого изучения.

#### DateTime

```csharp
DateTime now = DateTime.Now;        // local time
DateTime utc = DateTime.UtcNow;      // UTC (recommended!)
DateTime birthday = new(2000, 1, 15);
```

См. [[datetime-timezones|DateTime и Timezones]].

### Constants

```csharp
const int MaxRetries = 3;
const string ApiUrl = "https://api.example.com";

MaxRetries = 5;  // ❌ Cannot modify const
```

`const` известен **в compile time**. `readonly` — runtime constant (можно установить в constructor).

```csharp
public class Config
{
    public readonly int Port;  // can be set only in constructor

    public Config(int port) { Port = port; }
}
```

### Naming conventions

```csharp
// PascalCase — types, methods, properties, public fields
class UserService
public string FirstName { get; set; }
public void SaveUser() { }

// camelCase — local variables, parameters, private fields
int currentAge;
void Process(int userId) { }
private string _name;  // _ prefix для private fields

// UPPER_CASE — устарело
const int MaxRetries = 3;  // ✅ modern C#
```

---

## 2. Операторы

### Арифметические

```csharp
int sum = 5 + 3;         // 8
int diff = 5 - 3;         // 2
int product = 5 * 3;      // 15
int quotient = 10 / 3;    // 3 (integer division!)
int remainder = 10 % 3;   // 1 (modulo)
double precise = 10.0 / 3; // 3.333...

// Increment / decrement
int x = 5;
x++;   // x = 6
x--;   // x = 5

// Compound assignment
int a = 10;
a += 5;   // a = 15
a -= 3;   // a = 12
a *= 2;   // a = 24
a /= 4;   // a = 6
```

### Comparison

```csharp
5 == 5;    // true   (равно)
5 != 3;    // true   (не равно)
5 > 3;     // true   (больше)
5 < 3;     // false  (меньше)
5 >= 5;    // true   (больше или равно)
5 <= 4;    // false  (меньше или равно)
```

### Logical

```csharp
bool a = true;
bool b = false;

a && b;    // false (AND)
a || b;    // true  (OR)
!a;        // false (NOT)

// Short-circuit — если первое определяет результат, второе не проверяется
bool result = (user != null) && user.IsActive;  // safe — null check first
```

### Null operators

```csharp
string? name = null;

// ?? — null coalescing (если null, бери справа)
string display = name ?? "Anonymous";  // "Anonymous"

// ??= — null coalescing assignment
name ??= "Default";  // если name null, присвоить "Default"

// ?. — null conditional (safe member access)
int? length = name?.Length;  // null если name null

// !. — null forgiving (я знаю что не null!)
int length2 = name!.Length;  // throws если null
```

### Ternary (conditional)

```csharp
int age = 25;
string status = age >= 18 ? "Adult" : "Minor";  // "Adult"
```

### Type operators

```csharp
object obj = "hello";

// is — check тип
if (obj is string s)
{
    Console.WriteLine(s.Length);  // s — типа string внутри блока
}

// as — try cast (null если не подошёл)
string? s2 = obj as string;  // "hello" или null

// (T) — explicit cast (throws если не подошёл)
string s3 = (string)obj;  // "hello" или InvalidCastException
```

---

## 3. Control flow

### `if` / `else if` / `else`

```csharp
int age = 25;

if (age < 13)
{
    Console.WriteLine("Child");
}
else if (age < 18)
{
    Console.WriteLine("Teen");
}
else if (age < 65)
{
    Console.WriteLine("Adult");
}
else
{
    Console.WriteLine("Senior");
}
```

> [!info] Скобки `{ }` всегда
> Можно опускать для одного оператора, но **не делай этого**. С скобками — безопаснее.

### `switch`

```csharp
string role = "admin";

switch (role)
{
    case "admin":
        Console.WriteLine("Full access");
        break;
    case "user":
        Console.WriteLine("Limited access");
        break;
    case "guest":
    case "visitor":  // multiple cases
        Console.WriteLine("Read only");
        break;
    default:
        Console.WriteLine("Unknown");
        break;
}
```

### `switch expression` (modern, C# 8+)

```csharp
string accessLevel = role switch
{
    "admin" => "Full access",
    "user" => "Limited access",
    "guest" or "visitor" => "Read only",
    _ => "Unknown"  // _ — default
};
```

Гораздо короче. Используй везде где можешь.

### Pattern matching

```csharp
// Type pattern
object obj = 42;
string description = obj switch
{
    int i when i > 100 => "big int",
    int i => $"int: {i}",
    string s => $"string: {s}",
    null => "null",
    _ => "unknown"
};

// Property pattern
var person = new { Name = "John", Age = 30 };
string status = person switch
{
    { Age: < 18 } => "minor",
    { Age: >= 65 } => "senior",
    { Name: "John" } => "it's John",
    _ => "regular adult"
};
```

См. [[modern-features|Modern C# Features]] для deep dive.

### Loops — циклы

#### `for` — известное количество итераций

```csharp
for (int i = 0; i < 10; i++)
{
    Console.WriteLine(i);  // 0, 1, 2, ..., 9
}

// Reverse
for (int i = 10; i > 0; i--)
{
    Console.WriteLine(i);
}

// Step 2
for (int i = 0; i < 100; i += 2)
{
    Console.WriteLine(i);  // 0, 2, 4, ...
}
```

#### `foreach` — итерация по коллекции

```csharp
string[] names = { "Alice", "Bob", "Charlie" };

foreach (var name in names)
{
    Console.WriteLine(name);
}

// Работает с любым IEnumerable<T>
List<int> numbers = new() { 1, 2, 3, 4, 5 };
foreach (var num in numbers)
{
    Console.WriteLine(num);
}

Dictionary<string, int> ages = new()
{
    ["Alice"] = 30,
    ["Bob"] = 25
};
foreach (var (name, age) in ages)  // tuple deconstruction
{
    Console.WriteLine($"{name}: {age}");
}
```

#### `while` — пока условие истинно

```csharp
int countdown = 10;
while (countdown > 0)
{
    Console.WriteLine(countdown);
    countdown--;
}
Console.WriteLine("Liftoff!");
```

#### `do-while` — выполнить минимум 1 раз

```csharp
string input;
do
{
    Console.Write("Enter 'quit' to exit: ");
    input = Console.ReadLine();
}
while (input != "quit");
```

### `break` и `continue`

```csharp
// break — выйти из цикла
for (int i = 0; i < 100; i++)
{
    if (i == 5) break;  // exit
    Console.WriteLine(i);  // 0, 1, 2, 3, 4
}

// continue — следующая итерация
for (int i = 0; i < 10; i++)
{
    if (i % 2 == 0) continue;  // skip even
    Console.WriteLine(i);  // 1, 3, 5, 7, 9
}
```

---

## 4. Methods (функции)

### Базовый method

```csharp
public int Add(int a, int b)
{
    return a + b;
}

// Использование
int sum = Add(5, 3);  // 8
```

Структура: `[modifiers] [returnType] [Name]([parameters]) { [body] }`

### Return типы

```csharp
public int Sum(int a, int b) { return a + b; }  // returns int

public void PrintGreeting(string name)            // void — ничего не возвращает
{
    Console.WriteLine($"Hello, {name}!");
}

public bool IsEven(int n) => n % 2 == 0;          // expression-bodied (для одной строки)
```

### Parameters

```csharp
// Required parameters
public void Greet(string name, int age) { }

Greet("John", 25);

// Optional parameters (default values)
public void Greet(string name, int age = 18) { }

Greet("John");        // age = 18
Greet("John", 25);    // age = 25

// Named arguments
Greet(name: "John", age: 25);   // explicit names
Greet(age: 30, name: "Alice");  // any order
```

### Method overloading

Разные методы с тем же именем, **разные параметры**:

```csharp
public int Add(int a, int b) => a + b;
public double Add(double a, double b) => a + b;
public int Add(int a, int b, int c) => a + b + c;

Add(1, 2);          // calls int version
Add(1.5, 2.5);      // calls double version
Add(1, 2, 3);       // calls 3-param version
```

### Out / Ref parameters

```csharp
// out — функция должна установить значение
public void TryParse(string s, out int result)
{
    result = int.Parse(s);
}

TryParse("42", out int x);
Console.WriteLine(x);  // 42

// ref — passed by reference, можно читать и писать
public void Swap(ref int a, ref int b)
{
    int temp = a;
    a = b;
    b = temp;
}

int x = 1, y = 2;
Swap(ref x, ref y);
// x = 2, y = 1
```

### Built-in pattern: TryParse

```csharp
// Возвращает bool, через out даёт результат
if (int.TryParse("42", out int number))
{
    Console.WriteLine($"Parsed: {number}");
}
else
{
    Console.WriteLine("Not a number");
}
```

### params — variable arguments

```csharp
public int Sum(params int[] numbers)
{
    int total = 0;
    foreach (var n in numbers) total += n;
    return total;
}

Sum(1, 2);          // 3
Sum(1, 2, 3, 4, 5); // 15
Sum();              // 0
Sum(new[] { 1, 2, 3 }); // тоже работает
```

### Local functions (C# 7+)

Метод внутри метода:

```csharp
public int CalculateTotal(int[] prices)
{
    int total = 0;
    foreach (var p in prices)
        total += ApplyDiscount(p);
    return total;

    // local function — доступна только внутри CalculateTotal
    int ApplyDiscount(int price) => price > 100 ? price * 9 / 10 : price;
}
```

---

## 5. Classes — основы

### Что такое класс

**Шаблон** для создания объектов. Описывает **что у объекта есть** (поля, properties) и **что он умеет** (методы).

```csharp
public class User
{
    // Поля (fields)
    public string Name;
    public int Age;

    // Метод
    public void Greet()
    {
        Console.WriteLine($"Hi, I'm {Name}");
    }
}

// Использование
User user = new User();
user.Name = "John";
user.Age = 25;
user.Greet();  // "Hi, I'm John"
```

### Properties — лучше чем fields

```csharp
public class User
{
    // Auto-property — компилятор сам генерит backing field
    public string Name { get; set; }
    public int Age { get; set; }

    // Read-only property
    public string FullInfo { get; }

    // Property с custom logic
    private int _age;
    public int AgeWithValidation
    {
        get => _age;
        set
        {
            if (value < 0) throw new ArgumentException("Age cannot be negative");
            _age = value;
        }
    }

    // Computed property
    public bool IsAdult => Age >= 18;
}

// Использование
var user = new User { Name = "John", Age = 25 };  // object initializer
Console.WriteLine(user.IsAdult);  // true
```

### Constructors

```csharp
public class User
{
    public string Name { get; set; }
    public int Age { get; set; }

    // Constructor — вызывается при создании объекта
    public User(string name, int age)
    {
        Name = name;
        Age = age;
    }

    // Параметрless constructor
    public User() : this("Unknown", 0)  // calls другой constructor
    {
    }
}

// Использование
var u1 = new User("John", 25);
var u2 = new User();  // "Unknown", 0
```

### Primary constructors (C# 12+, modern)

```csharp
// Старый стиль
public class UserService
{
    private readonly IRepository _repo;

    public UserService(IRepository repo)
    {
        _repo = repo;
    }

    public User GetUser(int id) => _repo.GetById(id);
}

// Modern (C# 12+) — primary constructor
public class UserService(IRepository repo)
{
    public User GetUser(int id) => repo.GetById(id);
}
```

Намного короче. Параметры доступны как captured fields.

### `static` members

```csharp
public class MathUtils
{
    public static double Pi = 3.14159;

    public static int Add(int a, int b) => a + b;
}

// Использование без создания объекта
double pi = MathUtils.Pi;
int sum = MathUtils.Add(2, 3);
```

`static` — **принадлежат классу**, не экземпляру.

### Records (C# 9+) — для DTO

```csharp
// Record — immutable data class
public record User(string Name, int Age);

var u1 = new User("John", 25);
var u2 = new User("John", 25);

u1 == u2;  // true!  (value equality)

// `with` expression — clone с изменением
var u3 = u1 with { Age = 26 };
```

См. [[oop|OOP]] и [[functional-csharp|Functional C#]].

---

## 6. Common patterns для Junior

### Pattern 1: Console input/output

```csharp
Console.Write("Enter your name: ");
string? name = Console.ReadLine();

Console.WriteLine($"Hello, {name}!");

// Read int
Console.Write("Enter age: ");
if (int.TryParse(Console.ReadLine(), out int age))
{
    Console.WriteLine($"Your age: {age}");
}
else
{
    Console.WriteLine("Not a valid number");
}
```

### Pattern 2: Working with arrays

```csharp
// Создание
int[] numbers = { 1, 2, 3, 4, 5 };
int[] empty = new int[10];        // 10 zeros
int[] withSize = new int[5];

// Доступ
numbers[0];           // 1 (первый элемент)
numbers[^1];          // 5 (последний — С# 8+ from end)
numbers[1..3];        // {2, 3} — slice (C# 8+)

// Длина
numbers.Length;       // 5

// Iteration
foreach (var n in numbers)
{
    Console.WriteLine(n);
}

for (int i = 0; i < numbers.Length; i++)
{
    Console.WriteLine($"Index {i}: {numbers[i]}");
}
```

### Pattern 3: Working with List\<T\>

`List<T>` — динамический массив. Обычно используется чаще чем array.

```csharp
using System.Collections.Generic;

List<string> names = new();  // empty
names.Add("Alice");
names.Add("Bob");
names.Add("Charlie");

// Initial values
List<int> numbers = new() { 1, 2, 3 };

// Доступ
names[0];                // "Alice"
names.Count;             // 3
names.Contains("Bob");   // true
names.Remove("Bob");
names.Insert(0, "Zoe");

// Foreach
foreach (var name in names)
{
    Console.WriteLine(name);
}
```

См. [[collections-linq|Collections и LINQ]].

### Pattern 4: Working with Dictionary

```csharp
Dictionary<string, int> ages = new()
{
    ["Alice"] = 30,
    ["Bob"] = 25
};

// Add
ages["Charlie"] = 35;

// Access
int aliceAge = ages["Alice"];

// Safe access
if (ages.TryGetValue("Alice", out int age))
{
    Console.WriteLine(age);
}

// Iteration
foreach (var (name, value) in ages)
{
    Console.WriteLine($"{name}: {value}");
}

// Check exists
ages.ContainsKey("Alice");  // true

// Remove
ages.Remove("Bob");
```

### Pattern 5: Try/catch — обработка ошибок

```csharp
try
{
    int number = int.Parse(input);  // может throw FormatException
    Console.WriteLine(number * 2);
}
catch (FormatException ex)
{
    Console.WriteLine($"Not a number: {ex.Message}");
}
catch (Exception ex)  // catch-all
{
    Console.WriteLine($"Error: {ex.Message}");
}
finally
{
    Console.WriteLine("This always runs");
}
```

См. [[error-handling|Error Handling]] для глубокого изучения.

### Pattern 6: String manipulation

```csharp
string name = "  John Doe  ";

name.Trim();                          // "John Doe" (без пробелов)
name.ToUpper();                        // "  JOHN DOE  "
name.ToLower();                        // "  john doe  "
name.Replace("John", "Jane");          // "  Jane Doe  "
name.Contains("John");                 // true
name.StartsWith(" ");                   // true
name.Split(' ');                       // ["John", "Doe"]
name.Length;                           // 12

// Interpolation
string greeting = $"Hello, {name.Trim()}!";

// String methods chaining
string clean = name.Trim().ToLower().Replace(" ", "_");  // "john_doe"
```

См. [[strings-regex|Strings и Regex]].

---

## 7. Common mistakes начинающих

### 1. `==` vs `Equals` для objects

```csharp
// ✅ Для строк — works (string overrides ==)
string a = "hello";
string b = "hello";
a == b;  // true

// ❌ Для custom classes — reference comparison!
class Person { public string Name; }
var p1 = new Person { Name = "John" };
var p2 = new Person { Name = "John" };
p1 == p2;  // false! Разные objects

// ✅ Используй record для value equality
record Person(string Name);
var p1 = new Person("John");
var p2 = new Person("John");
p1 == p2;  // true!
```

### 2. Integer division

```csharp
int a = 10;
int b = 3;
int c = a / b;        // 3, не 3.33!

// ✅ Для дробного результата
double d = (double)a / b;  // 3.333...
double e = 10.0 / 3;        // 3.333...
```

### 3. Modifying collection во время iteration

```csharp
List<int> numbers = new() { 1, 2, 3, 4, 5 };

// ❌ InvalidOperationException
foreach (var n in numbers)
{
    if (n % 2 == 0) numbers.Remove(n);
}

// ✅ Создать новую collection
var filtered = numbers.Where(n => n % 2 != 0).ToList();

// ✅ Или итерация в обратном порядке
for (int i = numbers.Count - 1; i >= 0; i--)
{
    if (numbers[i] % 2 == 0) numbers.RemoveAt(i);
}
```

### 4. Null reference exceptions

```csharp
string? name = GetName();  // может вернуть null

// ❌
int length = name.Length;  // NullReferenceException если null

// ✅ Проверить
if (name != null)
{
    int length = name.Length;
}

// ✅ Null conditional
int? length = name?.Length;

// ✅ Null coalescing
int length = name?.Length ?? 0;
```

### 5. Off-by-one errors

```csharp
int[] arr = { 1, 2, 3, 4, 5 };

// ❌ IndexOutOfRangeException
for (int i = 0; i <= arr.Length; i++)  // <= вместо <
{
    Console.WriteLine(arr[i]);  // crash на arr[5]
}

// ✅
for (int i = 0; i < arr.Length; i++)
{
    Console.WriteLine(arr[i]);
}
```

### 6. Comparing floats с `==`

```csharp
double a = 0.1 + 0.2;
double b = 0.3;

a == b;  // false! a = 0.30000000000000004

// ✅ Сравнивай с epsilon
const double Epsilon = 0.0001;
Math.Abs(a - b) < Epsilon;  // true

// ✅ Или используй decimal
decimal a2 = 0.1m + 0.2m;
decimal b2 = 0.3m;
a2 == b2;  // true
```

### 7. Использовать `string` для money

```csharp
// ❌ string не для математики
string price = "19.99";

// ❌ float / double не для money
double total = 0.1 + 0.2;  // 0.30000000000000004

// ✅ decimal для денег
decimal price = 19.99m;
decimal total = price * 3;  // exact
```

### 8. Забывать `using` для disposable resources

```csharp
// ❌ Stream не закроется если throw
StreamReader reader = new StreamReader("file.txt");
string content = reader.ReadToEnd();
reader.Close();  // забыли при exception!

// ✅ using
using (StreamReader reader = new StreamReader("file.txt"))
{
    string content = reader.ReadToEnd();
}  // auto-close

// ✅ using statement (C# 8+)
using StreamReader reader = new StreamReader("file.txt");
string content = reader.ReadToEnd();
// disposed at end of scope
```

### 9. Wrong String formatting culture

```csharp
double price = 1234.56;

// ❌ Зависит от локали
string s = price.ToString();  // "1234.56" в US, "1234,56" в DE!

// ✅ Invariant для serialization
string s = price.ToString(CultureInfo.InvariantCulture);
```

### 10. Магические числа

```csharp
// ❌ Что за 86400?
if (elapsed > 86400)  // ⚠️ день в секундах? часах?
{
    // ...
}

// ✅ Named constant
const int SecondsInDay = 86400;
if (elapsed > SecondsInDay) { }

// ✅ Лучше — TimeSpan
if (elapsed > TimeSpan.FromDays(1).TotalSeconds) { }
```

---

## 8. Practice экзерсайзы для Junior

### Beginner

1. **Hello World** — print "Hello" с твоим именем
2. **Calculator** — input два числа, оператор, вывод результата
3. **FizzBuzz** — print 1 to 100, "Fizz" для divisible by 3, "Buzz" by 5, "FizzBuzz" by both
4. **Number guessing game** — random 1-100, user угадывает, hints "higher/lower"
5. **Reverse string** — input "hello" → "olleh"
6. **Palindrome check** — "level" → true, "hello" → false
7. **Fibonacci** — first 20 Fibonacci numbers
8. **Prime check** — is number prime?

### Intermediate

9. **Word count** — text → count occurrences of each word (Dictionary)
10. **TODO list** — add/remove/list tasks (List)
11. **Bank account** — class with deposit/withdraw, balance
12. **Library system** — books, borrow/return, multiple users
13. **Temperature converter** — Celsius/Fahrenheit/Kelvin
14. **Roman numerals** — convert int ↔ Roman ("XIV" ↔ 14)
15. **Parse CSV file** — read data, calculate statistics
16. **Quiz app** — questions, score, save results to file

### Resources

- **leetcode.com** — algorithm exercises (filter Easy, C#)
- **codewars.com** — kata progression
- **exercism.io** — mentored exercises

---

## 9. Дальше — где учиться

### После basics — следующие темы

| Тема | Файл |
|------|------|
| **OOP deep** | [[oop|OOP]] |
| **Modern features** | [[modern-features|Modern Features]] |
| **Collections + LINQ** | [[collections-linq|Collections и LINQ]] |
| **Async/await** | [[async-threading|Async и Threading]] |
| **Error handling** | [[error-handling|Error Handling]] |
| **Strings deep** | [[strings-regex|Strings и Regex]] |
| **DateTime** | [[datetime-timezones|DateTime]] |

### Roadmap

См. [[../LearningPath/02_junior-to-middle|Junior → Middle Roadmap]] — детальный план на 3-6 месяцев.

---

## 10. Best Practices — даже как Junior

- **`var` для obvious types**, explicit для важного
- **`decimal` для денег**, `double` для всего остального дробного
- **`int` для целых чисел** по default
- **String interpolation `$""`** > `String.Format`
- **`using`** для всех disposable
- **Properties** > public fields
- **`record`** для DTO
- **Switch expressions** > old switch statements
- **TryParse** > Parse (если возможен invalid input)
- **CultureInfo.InvariantCulture** для serialization
- **camelCase для variables**, **PascalCase для public**
- **Скобки `{ }`** даже для одного оператора
- **Не over-engineer** — простое решение лучше "умного"

---

## См. также

- [[oop|OOP]] — после basics
- [[modern-features|Modern Features]] — record, pattern matching, primary ctors
- [[collections-linq|Collections и LINQ]] — работа с коллекциями
- [[async-threading|Async и Threading]] — async/await
- [[error-handling|Error Handling]] — exception handling
- [[strings-regex|Strings и Regex]]
- [[datetime-timezones|DateTime]]
- [[../LearningPath/02_junior-to-middle|Learning Path: Junior → Middle]]

## Reading list

- **C# in Depth** — Jon Skeet (after basics)
- **Microsoft Docs — C# Programming Guide** — learn.microsoft.com/dotnet/csharp
- **Microsoft Learn — Take your first steps with C#** — learn.microsoft.com (free course!)
- **C# Yellow Book** — Rob Miles (free PDF, classic for beginners)
- **freeCodeCamp YouTube — C# course** — youtube.com (free, comprehensive)
