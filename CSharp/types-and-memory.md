---
tags: [types, memory, value-types, boxing, span]
level: Senior
---

# Типы, память и основы C#

> Справочник по фундаментальным концепциям C# 13 / .NET 9.
> Формат: теория → практика → senior-level код → вопросы интервью.

---

## Система типов

### Value types vs Reference types

C# — строго типизированный язык. Все типы делятся на две категории:

```
                    System.Object
                         |
            +-----------+-----------+
            |                       |
      Value Types            Reference Types
            |                       |
   +--------+--------+      +------+------+
   |        |        |      |      |      |
  int    double   struct   class string  array
  bool   decimal  enum     interface
  char   long     tuple    delegate
```

**Value types** — хранят данные напрямую. При присваивании создаётся **копия**.
**Reference types** — хранят ссылку на объект в heap. При присваивании копируется **ссылка**.

```csharp
// Value type — каждая переменная хранит своё значение
int a = 42;
int b = a;     // копия значения
b = 100;       // a по-прежнему 42

// Reference type — обе переменные указывают на один объект
int[] arr1 = [1, 2, 3];
int[] arr2 = arr1;   // копия ссылки
arr2[0] = 999;       // arr1[0] тоже стал 999
```

### Stack vs Heap — где что хранится

| Характеристика | Stack                          | Heap                            |
|----------------|--------------------------------|---------------------------------|
| Что хранится   | Value types, ссылки            | Объекты reference types         |
| Скорость       | Очень быстрый (LIFO)          | Медленнее (GC управляет)       |
| Время жизни    | Автоматически при выходе из scope | До сборки мусора (GC)        |
| Размер         | Ограничен (~1 MB по умолчанию)| Ограничен RAM                  |

```csharp
void ProcessOrder()
{
    int quantity = 5;                  // Stack: value type
    decimal price = 99.99m;            // Stack: value type
    Order order = new("ORD-001");      // Stack: ссылка (8 байт)
                                       // Heap: сам объект Order

    Span<byte> buffer = stackalloc byte[256]; // Stack: весь буфер на стеке
}
// quantity, price — освобождены при выходе из метода
// order (ссылка) — освобождена, объект в heap ждёт GC
```

### Boxing / Unboxing

**Boxing** — упаковка value type в `object` (аллокация в heap).
**Unboxing** — извлечение value type из `object`.

```csharp
int number = 42;

// Boxing: int → object (аллокация в heap, ~16 байт overhead на x64)
object boxed = number;

// Unboxing: object → int (проверка типа + копирование)
int unboxed = (int)boxed;

// ПРОБЛЕМА: boxing в горячих путях убивает производительность
// Плохо — boxing на каждой итерации:
ArrayList oldList = new();
for (int i = 0; i < 1000; i++)
    oldList.Add(i);  // boxing каждого int

// Хорошо — generic коллекция, boxing нет:
List<int> genericList = new(1000);
for (int i = 0; i < 1000; i++)
    genericList.Add(i);  // нет boxing

// Прямой вызов ToString() на struct переменной НЕ боксит (constrained call).
// Boxing происходит при приведении struct к object/интерфейсу.
struct Point
{
    public int X, Y;
    // Override предотвращает вызов через рефлексию (default ValueType.Equals)
    public override string ToString() => $"({X}, {Y})";
}
```

**Правило:** в performance-critical коде избегай boxing. Используй generics, `Span<T>`, перегружай `ToString()`/`GetHashCode()`/`Equals()` в struct.

> [!question]- **Интервью: Где boxing «стреляет» по производительности?**
> 1. Не-generic коллекции (`ArrayList.Add(42)` — boxing каждого int)
> 2. Строковая интерполяция (в некоторых случаях до C# 10)
> 3. `Enum.GetValues()` — возвращает `object[]`, каждый элемент boxed
> 4. Рефлексия и dynamic — частый boxing value types
> 5. `Dictionary<object, T>` с int ключом — boxing при каждом lookup
>
> **Решения:** generic коллекции, `IEquatable<T>` для struct, `Span<T>`, `Enum.GetValues<T>()`.

---

## Встроенные типы

### Числовые типы

| Тип       | Размер  | Диапазон                                        | Использование                    |
|-----------|---------|-------------------------------------------------|----------------------------------|
| `byte`    | 1 байт  | 0 .. 255                                        | Бинарные данные, протоколы       |
| `short`   | 2 байта | -32 768 .. 32 767                               | Редко, legacy API                |
| `int`     | 4 байта | -2.1 млрд .. 2.1 млрд                          | По умолчанию для целых           |
| `long`    | 8 байт  | ±9.2 × 10^18                                   | ID, timestamps, большие счётчики |
| `float`   | 4 байта | ~6-9 значащих цифр                              | GPU, ML, 3D-графика              |
| `double`  | 8 байт  | ~15-17 значащих цифр                            | Научные расчёты                  |
| `decimal` | 16 байт | 28-29 значащих цифр                             | Финансы, деньги                  |
| `nint`    | 4/8 байт| Зависит от платформы (32/64 bit)                | Interop, pointer arithmetic      |

```csharp
// int — основной тип для целых чисел
int ordersCount = 1_000_000;                    // _ для читаемости
int hexColor = 0xFF_AA_33;                      // hex литерал
int binaryMask = 0b_1010_0101;                  // binary литерал

// long — для ID и больших значений
long telegramUserId = 7_234_567_890L;           // суффикс L
long unixTimestamp = DateTimeOffset.UtcNow.ToUnixTimeMilliseconds();

// float — ML / графика (суффикс f обязателен)
float learningRate = 0.001f;
float[] embeddings = new float[1536];            // OpenAI embedding dimension

// double — научные расчёты, по умолчанию для дробных литералов
double latitude = 55.753_215;
double longitude = 37.622_504;
double distance = Math.Sqrt(Math.Pow(x2 - x1, 2) + Math.Pow(y2 - y1, 2));

// decimal — деньги и финансы (суффикс m обязателен)
decimal price = 1_499.99m;
decimal taxRate = 0.20m;
decimal total = price * (1 + taxRate);           // точные вычисления без потери копеек

// ОПАСНОСТЬ: float/double теряют точность с деньгами
double bad = 0.1 + 0.2;                         // 0.30000000000000004
decimal good = 0.1m + 0.2m;                     // 0.3 — точно
```

### bool, char, string

```csharp
// bool — 1 байт, только true/false
bool isActive = true;
bool hasPermission = user.Roles.Contains("Admin");

// Нельзя неявно конвертировать int → bool (в отличие от C/C++)
// if (count) { }  // Ошибка компиляции!
if (count > 0) { }  // Правильно

// char — 2 байта (UTF-16), одиночный символ
char grade = 'A';
char newLine = '\n';
char unicodeHeart = '\u2764';                    // ❤
bool isDigit = char.IsDigit(grade);              // false
bool isLetter = char.IsLetterOrDigit(grade);     // true

// string — immutable последовательность char (reference type!)
string name = "Александр";
string empty = string.Empty;                      // предпочтительнее ""
bool isBlank = string.IsNullOrWhiteSpace(name);  // false
int length = name.Length;                         // 9
```

### object и dynamic

```csharp
// object — базовый тип ВСЕХ типов в C#
object anything = 42;          // boxing int
anything = "строка";           // теперь string
anything = new List<int>();    // теперь List<int>

// Для проверки типа используй pattern matching
if (anything is List<int> list)
{
    Console.WriteLine($"Список с {list.Count} элементами");
}

// dynamic — тип определяется в runtime (обходит compile-time проверки)
// Используй ТОЛЬКО для: COM interop, работа с JSON/NoSQL, reflection
dynamic config = GetDynamicConfig();
string value = config.Database.ConnectionString; // нет проверки компилятором

// Опасность: ошибки всплывают только в runtime
dynamic d = "hello";
// d.NonExistentMethod(); // RuntimeBinderException в runtime!

// Типичное использование — ExpandoObject
dynamic expando = new System.Dynamic.ExpandoObject();
expando.Name = "Order";
expando.Total = 500m;
```

### Nullable value types

```csharp
// int? — синтаксический сахар для Nullable<int>
int? quantity = null;           // может быть null
int? stock = 150;

// Проверка наличия значения
if (quantity.HasValue)
{
    int actual = quantity.Value; // доступ к значению
}

// Более идиоматичный вариант — pattern matching
if (quantity is int qty)
{
    Console.WriteLine($"Количество: {qty}");
}

// GetValueOrDefault — безопасный доступ
int safeQuantity = quantity.GetValueOrDefault(0);

// Null-coalescing для Nullable
int finalQuantity = quantity ?? 0;

// Nullable reference types (C# 8+, #nullable enable) — это ДРУГОЕ:
// Это подсказки компилятора, не меняют runtime поведение
string? nullableName = null;    // Компилятор знает, что может быть null
string nonNullName = "test";    // Компилятор предупредит при присвоении null

// Null-forgiving operator (подавление предупреждения)
string forced = nullableName!;  // "Я знаю что делаю" — используй с осторожностью
```

---

## Переменные и константы

### var, const, readonly

```csharp
// var — неявная типизация, тип определяется компилятором
var orders = new List<Order>();              // List<Order>
var total = CalculateTotal();                // тип = return type метода
var (name, age) = GetPerson();               // деконструкция с var

// var НЕЛЬЗЯ использовать:
// var x;              // ошибка: нет инициализатора
// var y = null;       // ошибка: тип не определим
// var z = (int a) => a * 2; // ошибка: лямбда без target type (до C# 10)

// const — compile-time константа, встраивается в IL
const int MaxRetries = 3;
const string ApiVersion = "v2";
const decimal TaxRate = 0.20m;

// const ОГРАНИЧЕНИЯ:
// - только примитивы, string и null
// - const DateTime нельзя (кроме default)
// - значение зашивается в вызывающую сборку (опасно при обновлении библиотек!)

// readonly — runtime константа, инициализируется в конструкторе или inline
public class AppSettings
{
    // readonly field — задаётся один раз
    private readonly string _connectionString;
    private readonly int _maxPoolSize = 100; // inline инициализация

    public AppSettings(IConfiguration config)
    {
        _connectionString = config.GetConnectionString("Default")
            ?? throw new InvalidOperationException("Connection string not configured");
    }
}

// static readonly — для "констант" сложных типов
public static class Defaults
{
    public static readonly TimeSpan Timeout = TimeSpan.FromSeconds(30);
    public static readonly IReadOnlyList<string> AllowedRoles = ["Admin", "Manager"];
}
```

> [!question]- **Интервью: readonly vs const — в чём разница?**
> **const** — compile-time, значение подставляется inline как литерал. Только примитивы, enum, string. Опасно в публичном API: при изменении значения все зависимые сборки нужно перекомпилировать.
>
> **readonly** — runtime, инициализируется в конструкторе или inline. Любой тип. `static readonly` — для «констант» сложных типов (TimeSpan, коллекции).
>
> **Правило:** для публичных значений предпочитать `static readonly` над `const`.

### Default values

```csharp
// Value types всегда имеют default
int num = default;           // 0
bool flag = default;         // false
decimal amount = default;    // 0.0m
DateTime date = default;     // 0001-01-01T00:00:00
Guid id = default;           // 00000000-0000-0000-0000-000000000000

// Reference types: default = null
string? text = default;      // null
int[]? array = default;      // null
Order? order = default;      // null

// default(T) в generic методах
T CreateDefault<T>() => default!;

// Полезно для проверки "пустых" значений
public static bool IsDefault<T>(T value) =>
    EqualityComparer<T>.Default.Equals(value, default!);

// Пример использования
if (IsDefault(orderId))    // orderId == Guid.Empty
    throw new ArgumentException("Order ID is required");
```

---

## Операторы

### Арифметические и логические

```csharp
// Арифметические
int sum = 10 + 3;          // 13
int diff = 10 - 3;         // 7
int product = 10 * 3;      // 30
int quotient = 10 / 3;     // 3 (целочисленное деление!)
double precise = 10.0 / 3; // 3.333...
int remainder = 10 % 3;    // 1 (остаток)

// Increment / Decrement
int counter = 0;
counter++;                  // post-increment: вернёт 0, потом станет 1
++counter;                  // pre-increment: станет 2, вернёт 2

// Составные операторы
decimal balance = 1000m;
balance += 500m;            // balance = 1500
balance -= 200m;            // balance = 1300
balance *= 1.1m;            // balance = 1430

// Логические операторы
bool isValid = age >= 18 && age <= 65;           // AND (short-circuit)
bool hasAccess = isAdmin || hasPermission;        // OR (short-circuit)
bool isDisabled = !isActive;                      // NOT

// & и | — НЕ short-circuit (оба операнда всегда вычисляются)
bool result = Validate(input) & LogAttempt(input); // LogAttempt ВСЕГДА вызовется
```

### Побитовые операторы

```csharp
// AND, OR, XOR, NOT, SHIFT
int a = 0b_1100;       // 12
int b = 0b_1010;       // 10

int and = a & b;       // 0b_1000 = 8
int or  = a | b;       // 0b_1110 = 14
int xor = a ^ b;       // 0b_0110 = 6
int not = ~a;           // инвертирует все биты

int left  = a << 2;    // 0b_110000 = 48 (умножение на 4)
int right = a >> 2;    // 0b_0011   = 3  (деление на 4)
int uright = a >>> 2;  // unsigned right shift (C# 11+)

// Практическое применение — флаги разрешений
[Flags]
enum Permission
{
    None   = 0b_0000,  // 0
    Read   = 0b_0001,  // 1
    Write  = 0b_0010,  // 2
    Delete = 0b_0100,  // 4
    Admin  = Read | Write | Delete
}

Permission userPerm = Permission.Read | Permission.Write;
bool canDelete = (userPerm & Permission.Delete) != 0;         // false
userPerm |= Permission.Delete;                                 // добавить Delete
userPerm &= ~Permission.Write;                                // убрать Write
bool hasAll = userPerm.HasFlag(Permission.Read);              // true
```

### Null-conditional и Null-coalescing

```csharp
// ?. — Null-conditional: вызывает только если не null
Order? order = GetOrder(id);
string? customerName = order?.Customer?.Name;      // null если любой элемент null
int? itemCount = order?.Items?.Count;              // null или число
decimal? firstPrice = order?.Items?[0]?.Price;     // ?[] — для индексаторов

// ?. с вызовом метода
order?.Customer?.SendNotification();                // не вызовется если null

// ?? — Null-coalescing: значение по умолчанию если null
string name = customerName ?? "Unknown";
int count = itemCount ?? 0;
List<Item> items = order?.Items ?? [];              // C# 12: collection expression

// ??= — присвоить только если null
private List<Order>? _cache;
public List<Order> Orders => _cache ??= LoadOrders(); // lazy initialization

// Цепочки
string displayName = user?.DisplayName
    ?? user?.Email
    ?? user?.Phone
    ?? "Anonymous";
```

### Тернарный оператор и другие

```csharp
// Тернарный оператор — для простых условий
string status = order.IsPaid ? "Оплачен" : "Ожидает оплаты";
decimal discount = quantity > 100 ? 0.15m : quantity > 50 ? 0.10m : 0m;

// is — проверка типа + pattern matching
if (shape is Circle { Radius: > 0 } circle)
{
    double area = Math.PI * circle.Radius * circle.Radius;
}

// is с логическими паттернами (C# 9+)
if (statusCode is >= 200 and < 300)
    Console.WriteLine("Success");

if (input is not null and not "")
    Process(input);

// as — безопасное приведение (null если не удалось)
IDisposable? disposable = connection as IDisposable;
disposable?.Dispose();

// typeof — получить Type в compile-time
Type intType = typeof(int);                       // System.Int32
Type listType = typeof(List<>);                   // открытый generic

// sizeof — размер value type (только в unsafe или для примитивов)
int intSize = sizeof(int);                        // 4
int decimalSize = sizeof(decimal);                // 16

// nameof — имя символа как строка (compile-time)
void SetName(string name)
{
    ArgumentNullException.ThrowIfNull(name);       // использует CallerArgumentExpression
    ArgumentException.ThrowIfNullOrWhiteSpace(name, nameof(name));
}

// nameof для INotifyPropertyChanged, логирования
_logger.LogInformation("Entering {Method}", nameof(ProcessPayment));
```

### Checked / Unchecked overflow

```csharp
// По умолчанию арифметическое переполнение НЕ бросает исключение
int max = int.MaxValue;    // 2_147_483_647
int overflow = max + 1;    // -2_147_483_648 (тихое переполнение!)

// checked — бросает OverflowException при переполнении
try
{
    int safe = checked(max + 1); // OverflowException!
}
catch (OverflowException ex)
{
    _logger.LogError(ex, "Arithmetic overflow detected");
}

// checked блок
checked
{
    int x = int.MaxValue;
    // int y = x + 1; // OverflowException
}

// unchecked — явно разрешить переполнение (полезно для хеш-функций)
unchecked
{
    int hash = 17;
    hash = hash * 31 + field1.GetHashCode();
    hash = hash * 31 + field2.GetHashCode(); // overflow допустим для хеша
}

// C# 11: checked операторы в пользовательских типах
public readonly struct Money
{
    public decimal Amount { get; init; }

    public static Money operator +(Money a, Money b) =>
        new() { Amount = a.Amount + b.Amount };

    public static Money operator checked +(Money a, Money b)
    {
        decimal result = checked(a.Amount + b.Amount);
        return new() { Amount = result };
    }
}
```

---

## Приведение типов

### Implicit vs Explicit conversion

```csharp
// Implicit — автоматическое, без потери данных (widening)
int intVal = 42;
long longVal = intVal;         // int → long: безопасно
float floatVal = intVal;       // int → float: implicit, но ТЕРЯЕТ precision для больших значений!
double doubleVal = floatVal;   // float → double: безопасно
decimal decVal = intVal;       // int → decimal: безопасно

// Explicit — требуется cast, возможна потеря данных (narrowing)
double pi = 3.14159;
int truncated = (int)pi;       // 3 (дробная часть отбрасывается)

long bigNum = 3_000_000_000L;
int overflow = (int)bigNum;    // непредсказуемый результат!

float precise = 1.7f;
int rounded = (int)precise;    // 1 (НЕ округляет, а обрезает)

// Безопасное преобразование с проверкой
if (longVal is >= int.MinValue and <= int.MaxValue)
{
    int safe = (int)longVal;
}
```

### Convert class

```csharp
// Convert — обрабатывает null и делает ОКРУГЛЕНИЕ (в отличие от cast)
double value = 1.7;
int casted = (int)value;                // 1 (truncation)
int converted = Convert.ToInt32(value); // 2 (banker's rounding!)

// Convert обрабатывает null → default для value types
object? nullObj = null;
int fromNull = Convert.ToInt32(nullObj); // 0 (не бросает исключение)

// Convert для различных типов
string numStr = "42";
int parsed = Convert.ToInt32(numStr);
bool fromInt = Convert.ToBoolean(1);     // true (любое не-0 = true)

// Convert для Base64
byte[] data = System.Text.Encoding.UTF8.GetBytes("Hello, World!");
string base64 = Convert.ToBase64String(data);
byte[] decoded = Convert.FromBase64String(base64);

// Convert для чисел в разных системах счисления
string binary = Convert.ToString(255, 2);   // "11111111"
string hex = Convert.ToString(255, 16);     // "ff"
int fromHex = Convert.ToInt32("ff", 16);    // 255
```

### Parse / TryParse

```csharp
// Parse — бросает исключение при неудаче
int count = int.Parse("42");
decimal price = decimal.Parse("1499.99", CultureInfo.InvariantCulture);
DateTime date = DateTime.Parse("2026-02-19");

// Parse с провайдером формата
decimal amount = decimal.Parse(
    "1,499.99",
    NumberStyles.AllowThousands | NumberStyles.AllowDecimalPoint,
    CultureInfo.InvariantCulture);

// TryParse — ВСЕГДА предпочтительнее Parse (нет исключений)
if (int.TryParse(userInput, out int result))
{
    ProcessQuantity(result);
}
else
{
    _logger.LogWarning("Invalid quantity input: {Input}", userInput);
}

// TryParse для разных типов
bool isValidGuid = Guid.TryParse(idString, out Guid orderId);
bool isValidDate = DateTimeOffset.TryParse(
    dateString,
    CultureInfo.InvariantCulture,
    DateTimeStyles.AssumeUniversal,
    out DateTimeOffset timestamp);

// C# 11: IParsable<T> — generic парсинг
T ParseOrDefault<T>(string? input, T defaultValue) where T : IParsable<T> =>
    T.TryParse(input, CultureInfo.InvariantCulture, out T? result)
        ? result
        : defaultValue;

int qty = ParseOrDefault("42", 0);
decimal price2 = ParseOrDefault("99.99", 0m);
```

### Пользовательские операторы преобразования

```csharp
public readonly struct Percentage
{
    public decimal Value { get; }

    private Percentage(decimal value) => Value = value;

    // implicit: decimal → Percentage (безопасно)
    public static implicit operator Percentage(decimal value) =>
        new(value is >= 0 and <= 100
            ? value
            : throw new ArgumentOutOfRangeException(nameof(value)));

    // explicit: Percentage → decimal (нужен cast)
    public static explicit operator decimal(Percentage p) => p.Value;

    // explicit: Percentage → int (потеря данных — explicit)
    public static explicit operator int(Percentage p) => (int)Math.Round(p.Value);

    public override string ToString() => $"{Value:F1}%";
}

// Использование
Percentage discount = 15.5m;             // implicit conversion
decimal raw = (decimal)discount;          // explicit conversion
int rounded = (int)discount;             // 16
Console.WriteLine(discount);              // "15.5%"
```

---

## Строки

### String immutability и interning

```csharp
// Строки НЕИЗМЕНЯЕМЫ — каждая "модификация" создаёт НОВЫЙ объект
string greeting = "Hello";
string modified = greeting.Replace("H", "J"); // "Jello" — новый объект
// greeting по-прежнему "Hello"

// String interning — одинаковые литералы ссылаются на один объект
string a = "hello";
string b = "hello";
bool sameRef = ReferenceEquals(a, b);  // true — interned

string c = new string("hello".ToCharArray());
bool sameRef2 = ReferenceEquals(a, c); // false — не interned

string d = string.Intern(c);
bool sameRef3 = ReferenceEquals(a, d); // true — вручную interned

// Interning полезен для часто повторяющихся строк (ключи словарей)
// Но осторожно: interned строки живут до конца AppDomain
```

### StringBuilder

```csharp
// ПРАВИЛО: используй StringBuilder при конкатенации в цикле
// Плохо — O(n^2) аллокаций:
string bad = "";
for (int i = 0; i < 10_000; i++)
    bad += i.ToString(); // каждый += создаёт новую строку!

// Хорошо — O(n) с одним буфером:
var sb = new StringBuilder(capacity: 50_000); // предварительный размер
for (int i = 0; i < 10_000; i++)
    sb.Append(i);
string good = sb.ToString();

// StringBuilder API
var builder = new StringBuilder();
builder.Append("SELECT ");
builder.AppendJoin(", ", ["id", "name", "email"]);
builder.AppendLine();
builder.Append("FROM users");
builder.AppendLine();
builder.Append("WHERE active = true");
builder.Insert(0, "-- User query\n");
builder.Replace("active", "is_active");

string query = builder.ToString();

// C# 6+: для НЕСКОЛЬКИХ конкатенаций (не в цикле) — интерполяция OK
string info = $"User {name} (ID: {id}) logged in at {DateTime.UtcNow:O}";
// Компилятор оптимизирует до DefaultInterpolatedStringHandler
```

### Интерполяция и специальные строки

```csharp
// Интерполяция $"" — основной способ форматирования
decimal price = 1_499.99m;
int quantity = 3;
string receipt = $"Товар: {quantity} шт. × {price:C2} = {price * quantity:C2}";

// Форматирование внутри интерполяции
DateTime now = DateTime.UtcNow;
string log = $"[{now:yyyy-MM-dd HH:mm:ss.fff}] Processing order #{orderId:D6}";
// D6 = дополнить нулями до 6 цифр

// Alignment
string table = $"{"Name",-20} {"Price",10:C2}";  // -20 = left-align, 10 = right-align

// Verbatim @"" — отключает escape-последовательности
string path = @"C:\Users\mos26\Documents\report.csv";
string regex = @"\d{3}-\d{2}-\d{4}";

// Комбинация: $@"" или @$""
string fullPath = $@"C:\Users\{username}\{folder}";

// Raw string literals (C# 11+) — многострочные без escape
string json = """
    {
        "name": "Order",
        "items": [
            { "id": 1, "price": 99.99 }
        ]
    }
    """;

// Raw string с интерполяцией (число $ = число { для escape)
int id = 42;
string jsonInterp = $$"""
    {
        "orderId": {{id}},
        "status": "pending"
    }
    """;
```

### Основные методы строк

```csharp
string text = "  Hello, World! Welcome to C#.  ";

// Поиск и проверка
bool contains = text.Contains("World");                  // true
bool starts = text.StartsWith("  Hello");                // true
bool ends = text.EndsWith("C#.  ");                      // true
int index = text.IndexOf("World");                       // 9
int lastIndex = text.LastIndexOf('!');                    // 15

// Модификация (возвращает новую строку)
string trimmed = text.Trim();                            // без пробелов по краям
string upper = text.ToUpperInvariant();                  // для сравнений
string replaced = text.Replace("World", "Developer");
string sub = text.Substring(9, 5);                       // "World" (legacy)
string sub2 = text[9..14];                               // "World" (Range — предпочтительно)

// Split и Join
string csv = "apple,banana,,cherry";
string[] parts = csv.Split(',', StringSplitOptions.RemoveEmptyEntries);
// ["apple", "banana", "cherry"]

string joined = string.Join(" | ", parts);  // "apple | banana | cherry"

// C# 13: новые методы
// Проверка пустоты
bool isEmpty = string.IsNullOrEmpty(text);
bool isBlank = string.IsNullOrWhiteSpace(text);
```

### Span\<char\> и StringComparison

```csharp
// Span<char> — работа с частью строки БЕЗ аллокации
string logLine = "2026-02-19T10:30:00 [INFO] Order processed successfully";

ReadOnlySpan<char> dateSpan = logLine.AsSpan(0, 19);     // без аллокации
ReadOnlySpan<char> levelSpan = logLine.AsSpan(20, 6);    // "[INFO]"
ReadOnlySpan<char> messageSpan = logLine.AsSpan(27);     // остаток

// Парсинг числа без создания подстроки
string data = "Price:1499.99:USD";
ReadOnlySpan<char> span = data.AsSpan();
int firstColon = span.IndexOf(':');
int lastColon = span.LastIndexOf(':');
ReadOnlySpan<char> priceSpan = span[(firstColon + 1)..lastColon];

if (decimal.TryParse(priceSpan, CultureInfo.InvariantCulture, out decimal parsedPrice))
{
    Console.WriteLine($"Parsed: {parsedPrice}");
}

// StringComparison — ВСЕГДА указывай при сравнении строк
string email1 = "Admin@Company.Com";
string email2 = "admin@company.com";

// Ordinal — побайтовое сравнение (самое быстрое)
bool exact = email1.Equals(email2, StringComparison.Ordinal);              // false

// OrdinalIgnoreCase — побайтовое без учёта регистра
bool ignoreCase = email1.Equals(email2, StringComparison.OrdinalIgnoreCase); // true

// CurrentCulture — для отображения пользователю (с учётом локали)
int order = string.Compare("straße", "strasse", StringComparison.CurrentCulture);

// ПРАВИЛО: для технических сравнений (ключи, пути, ID) — ВСЕГДА Ordinal/OrdinalIgnoreCase
// Для UI/пользовательского контента — CurrentCulture
Dictionary<string, object> headers = new(StringComparer.OrdinalIgnoreCase);
headers["Content-Type"] = "application/json";
bool hasContentType = headers.ContainsKey("content-type"); // true
```

---

## Массивы

### Одномерные, многомерные, jagged

```csharp
// Одномерные массивы
int[] numbers = [1, 2, 3, 4, 5];                // C# 12 collection expression
int[] zeros = new int[100];                       // заполнен нулями
string[] names = new string[3];                   // заполнен null

// Доступ по индексу
int first = numbers[0];                           // 1
int last = numbers[^1];                           // 5 (Index from end)
int[] slice = numbers[1..4];                      // [2, 3, 4] (Range)
int[] lastTwo = numbers[^2..];                    // [4, 5]

// Многомерные массивы (rectangular) — одна непрерывная область памяти
int[,] matrix = new int[3, 4];                    // 3 строки, 4 столбца
matrix[0, 0] = 1;
int rows = matrix.GetLength(0);                   // 3
int cols = matrix.GetLength(1);                   // 4

int[,] grid =
{
    { 1, 2, 3 },
    { 4, 5, 6 },
    { 7, 8, 9 }
};

// Jagged массивы (массив массивов) — каждый ряд может быть разной длины
int[][] jagged = new int[3][];
jagged[0] = [1, 2];
jagged[1] = [3, 4, 5, 6];
jagged[2] = [7];

// Jagged часто быстрее многомерных (JIT лучше оптимизирует)
```

### Array методы

```csharp
int[] data = [64, 25, 12, 22, 11, 90, 45];

// Сортировка
Array.Sort(data);                                   // [11, 12, 22, 25, 45, 64, 90]

// Поиск (BinarySearch требует массив, отсортированный по возрастанию!)
int index = Array.BinarySearch(data, 25);           // 3 (индекс элемента 25)

Array.Reverse(data);                                // [90, 64, 45, 25, 22, 12, 11]
// ⚠️ После Reverse BinarySearch вернёт некорректный результат!
int found = Array.Find(data, x => x > 50);         // первый элемент > 50
int[] allBig = Array.FindAll(data, x => x > 50);   // все элементы > 50
bool any = Array.Exists(data, x => x == 25);       // true

// Копирование
int[] copy = new int[data.Length];
Array.Copy(data, copy, data.Length);

// Fill
int[] buffer = new int[100];
Array.Fill(buffer, -1);                             // все элементы = -1
Array.Fill(buffer, 0, 10, 20);                     // элементы [10..30) = 0

// Resize (создаёт новый массив!)
int[] arr = [1, 2, 3];
Array.Resize(ref arr, 5);                           // [1, 2, 3, 0, 0]
```

### Span\<T\> и Memory\<T\>

```csharp
// Span<T> — окно в непрерывную память (stack only, ref struct)
int[] array = [10, 20, 30, 40, 50, 60, 70, 80];

Span<int> span = array.AsSpan();
Span<int> middle = span.Slice(2, 4);               // [30, 40, 50, 60] — без копирования!
middle[0] = 999;                                    // array[2] тоже стал 999

// Span из stackalloc — нулевые аллокации
Span<byte> stackBuffer = stackalloc byte[256];
stackBuffer.Fill(0xFF);

// Парсинг без аллокаций
ReadOnlySpan<char> csvLine = "100,John,Admin".AsSpan();
var enumerator = csvLine.Split(',');                 // нет аллокации строк

// Memory<T> — как Span, но можно хранить в heap (в полях класса, async методах)
Memory<byte> memory = new byte[1024];
Memory<byte> slice = memory.Slice(0, 512);

// Span нельзя в async методах, Memory — можно
async Task ProcessAsync(Memory<byte> data)
{
    // Создаём Span только в синхронных участках
    Span<byte> span = data.Span;
    ProcessSync(span);

    await Task.Delay(100);

    // Снова получаем Span
    span = data.Span;
    ProcessSync(span);
}
```

### ArrayPool\<T\>

```csharp
// ArrayPool — переиспользование массивов (нет нагрузки на GC)
using System.Buffers;

byte[] buffer = ArrayPool<byte>.Shared.Rent(4096);
// ВАЖНО: Rent может вернуть массив БОЛЬШЕ запрошенного размера
try
{
    int bytesRead = await stream.ReadAsync(buffer.AsMemory(0, 4096));
    ProcessData(buffer.AsSpan(0, bytesRead));
}
finally
{
    // ОБЯЗАТЕЛЬНО вернуть! Иначе — утечка пула
    ArrayPool<byte>.Shared.Return(buffer, clearArray: true);
}

// Практический пример: чтение файла чанками
async Task ProcessLargeFileAsync(string path)
{
    const int chunkSize = 81_920; // 80 KB
    byte[] buffer = ArrayPool<byte>.Shared.Rent(chunkSize);
    try
    {
        await using var stream = File.OpenRead(path);
        int bytesRead;
        while ((bytesRead = await stream.ReadAsync(buffer.AsMemory(0, chunkSize))) > 0)
        {
            ProcessChunk(buffer.AsSpan(0, bytesRead));
        }
    }
    finally
    {
        ArrayPool<byte>.Shared.Return(buffer);
    }
}
```

---

## Кортежи

### ValueTuple и деконструкция

```csharp
// ValueTuple — легковесный value type для группировки значений
(int x, int y) point = (10, 20);
Console.WriteLine($"X={point.x}, Y={point.y}");

// Без имён — доступ через Item1, Item2 (не рекомендуется)
(int, string) raw = (1, "test");
Console.WriteLine(raw.Item1);

// Return типы — кортежи вместо создания DTO для внутренних нужд
(decimal Total, int ItemCount, bool HasDiscount) CalculateCart(List<CartItem> items)
{
    decimal total = items.Sum(i => i.Price * i.Quantity);
    int count = items.Count;
    bool discount = total > 1000m;
    return (total, count, discount);
}

// Вызов и деконструкция
var (total, count, hasDiscount) = CalculateCart(items);
Console.WriteLine($"Итого: {total:C2}, товаров: {count}");

// Игнорирование ненужных значений с discard _
var (_, itemCount, _) = CalculateCart(items);

// Кортежи в pattern matching
string GetQuadrant((int x, int y) point) => point switch
{
    ( > 0, > 0) => "I",
    ( < 0, > 0) => "II",
    ( < 0, < 0) => "III",
    ( > 0, < 0) => "IV",
    _ => "На оси"
};

// Деконструкция пользовательских типов
public class Order
{
    public string Id { get; init; } = null!;
    public decimal Total { get; init; }
    public OrderStatus Status { get; init; }

    public void Deconstruct(out string id, out decimal total, out OrderStatus status)
    {
        id = Id;
        total = Total;
        status = Status;
    }
}

var (orderId, orderTotal, orderStatus) = GetOrder();
```

---

## Перечисления (enum)

### Объявление и underlying type

```csharp
// По умолчанию underlying type — int
enum OrderStatus
{
    Pending,          // 0
    Processing,       // 1
    Shipped,          // 2
    Delivered,        // 3
    Cancelled = 10    // явное значение
}

// Указание underlying type для экономии памяти
enum LogLevel : byte
{
    Trace = 0,
    Debug = 1,
    Information = 2,
    Warning = 3,
    Error = 4,
    Critical = 5
}

// Использование
OrderStatus status = OrderStatus.Processing;
int numValue = (int)status;                         // 1
OrderStatus fromInt = (OrderStatus)2;               // Shipped

// Pattern matching с enum
string message = status switch
{
    OrderStatus.Pending => "Заказ ожидает обработки",
    OrderStatus.Processing => "Заказ в обработке",
    OrderStatus.Shipped => "Заказ отправлен",
    OrderStatus.Delivered => "Заказ доставлен",
    OrderStatus.Cancelled => "Заказ отменён",
    _ => throw new ArgumentOutOfRangeException(nameof(status))
};
```

### Flags attribute

```csharp
[Flags]
enum FileAccess
{
    None      = 0,
    Read      = 1 << 0,   // 1
    Write     = 1 << 1,   // 2
    Execute   = 1 << 2,   // 4
    Delete    = 1 << 3,   // 8
    ReadWrite = Read | Write,
    All       = Read | Write | Execute | Delete
}

// Комбинирование флагов
FileAccess perms = FileAccess.Read | FileAccess.Write;

// Проверка наличия флага
bool canRead = perms.HasFlag(FileAccess.Read);          // true
bool canExecute = (perms & FileAccess.Execute) != 0;    // false (в .NET Core+ HasFlag инлайнится JIT — одинаковая производительность)

// Добавление и удаление флагов
perms |= FileAccess.Execute;                             // добавить
perms &= ~FileAccess.Write;                             // удалить

// ToString для [Flags]
Console.WriteLine(perms);                                // "Read, Execute"
```

### Enum.Parse и утилиты

```csharp
// Parse / TryParse
OrderStatus parsed = Enum.Parse<OrderStatus>("Shipped");
bool ok = Enum.TryParse<OrderStatus>("shipped", ignoreCase: true, out var result);

// Проверка что значение определено (важно при десериализации!)
int untrustedValue = 999;
bool isDefined = Enum.IsDefined(typeof(OrderStatus), untrustedValue); // false

// Получение всех значений
OrderStatus[] allStatuses = Enum.GetValues<OrderStatus>();
string[] allNames = Enum.GetNames<OrderStatus>();

// Практика: маппинг enum → string для API
static class OrderStatusExtensions
{
    public static string ToDisplayString(this OrderStatus status) => status switch
    {
        OrderStatus.Pending => "pending",
        OrderStatus.Processing => "processing",
        OrderStatus.Shipped => "shipped",
        OrderStatus.Delivered => "delivered",
        OrderStatus.Cancelled => "cancelled",
        _ => throw new ArgumentOutOfRangeException(nameof(status))
    };
}
```

---

## Структуры (struct)

### struct vs class

| Характеристика      | `struct` (value type)        | `class` (reference type)    |
|----------------------|------------------------------|-----------------------------|
| Хранение            | Stack (или inline в объекте) | Heap                        |
| Присваивание        | Копирование значения         | Копирование ссылки          |
| Default              | Все поля = default           | null                        |
| Наследование        | Нет (только интерфейсы)     | Есть                        |
| GC давление         | Нет (на стеке)               | Да                          |
| Размер рекомендация | До ~16 байт                  | Любой                       |
| Equality по умолчанию| По значениям (медленно через reflection) | По ссылке       |

> [!question]- **Интервью: Class vs Struct — когда что выбирать?**
> **Struct** — маленькие данные (<16-24 байт), семантика значения, неизменяемость, hot path без GC pressure. Примеры: координаты, Money, strongly-typed ID.
>
> **Class** — большие объекты (>32 байт), наследование, изменяемое состояние, identity. При сомнении — выбирай class.
>
> **Нюансы:** struct в `List<T>` копируется при каждом `Add`. Mutable struct — антипаттерн. `readonly struct` / `record struct` — предпочтительные формы.

```csharp
// struct — маленькие, часто копируемые данные
struct Coordinate
{
    public double Latitude { get; init; }
    public double Longitude { get; init; }
}

Coordinate a = new() { Latitude = 55.75, Longitude = 37.62 };
Coordinate b = a;           // полная копия
b = b with { Latitude = 59.93 }; // b изменён, a — нет (с record struct)
```

### readonly struct

```csharp
// readonly struct — гарантирует неизменяемость, позволяет оптимизации компилятора
public readonly struct Money : IEquatable<Money>
{
    public required decimal Amount { get; init; }
    public required string Currency { get; init; }

    // Все методы readonly неявно (компилятор не создаёт defensive copies)
    public Money Add(Money other)
    {
        if (Currency != other.Currency)
            throw new InvalidOperationException(
                $"Cannot add {Currency} and {other.Currency}");

        return this with { Amount = Amount + other.Amount };
    }

    public bool Equals(Money other) =>
        Amount == other.Amount &&
        string.Equals(Currency, other.Currency, StringComparison.Ordinal);

    public override bool Equals(object? obj) => obj is Money m && Equals(m);
    public override int GetHashCode() => HashCode.Combine(Amount, Currency);
    public override string ToString() => $"{Amount:F2} {Currency}";

    public static bool operator ==(Money left, Money right) => left.Equals(right);
    public static bool operator !=(Money left, Money right) => !left.Equals(right);
}

var price = new Money { Amount = 99.99m, Currency = "USD" };
var tax = new Money { Amount = 20.00m, Currency = "USD" };
var total = price.Add(tax); // 119.99 USD
```

### ref struct

```csharp
// ref struct — ТОЛЬКО на стеке, не может попасть в heap
// Используется для: Span<T>, ReadOnlySpan<T>, высокопроизводительные парсеры

public ref struct CsvLineParser
{
    private ReadOnlySpan<char> _remaining;

    public CsvLineParser(ReadOnlySpan<char> line)
    {
        _remaining = line;
    }

    public ReadOnlySpan<char> ReadNext()
    {
        if (_remaining.IsEmpty) return [];

        int commaIndex = _remaining.IndexOf(',');
        if (commaIndex == -1)
        {
            var result = _remaining;
            _remaining = [];
            return result;
        }

        var field = _remaining[..commaIndex];
        _remaining = _remaining[(commaIndex + 1)..];
        return field;
    }
}

// Использование
ReadOnlySpan<char> line = "100,John Doe,admin@test.com,true".AsSpan();
var parser = new CsvLineParser(line);

ReadOnlySpan<char> id = parser.ReadNext();      // "100"
ReadOnlySpan<char> name = parser.ReadNext();    // "John Doe"
ReadOnlySpan<char> email = parser.ReadNext();   // "admin@test.com"

// ref struct НЕЛЬЗЯ:
// - Хранить в полях класса
// - Использовать в async методах
// - Boxing (object boxed = spanValue) — ошибка компиляции
// - Использовать в лямбдах/замыканиях
```

> [!question]- **Интервью: Зачем ref struct? Какие ограничения?**
> `ref struct` может содержать managed pointer (`ref T`), указывающий на стек. Если бы попал на heap — GC не смог бы отследить → memory corruption.
>
> **Ограничения:** не поле класса, не boxing, не async/await, не замыкания, не yield. Примеры: `Span<T>`, `ReadOnlySpan<T>`.
>
> **C# 13:** ref struct может реализовывать интерфейсы (`IDisposable`). `allows ref struct` constraint для generic.

### record struct (C# 10+)

```csharp
// record struct — value type с автоматическими Equals, GetHashCode, ToString, Deconstruct
public readonly record struct GeoPoint(double Latitude, double Longitude)
{
    // Вычисляемое свойство
    public string AsString => $"{Latitude:F6},{Longitude:F6}";

    // Метод расчёта расстояния (формула Haversine)
    public double DistanceTo(GeoPoint other)
    {
        const double R = 6_371_000; // радиус Земли в метрах
        double dLat = ToRadians(other.Latitude - Latitude);
        double dLon = ToRadians(other.Longitude - Longitude);

        double a = Math.Sin(dLat / 2) * Math.Sin(dLat / 2) +
                   Math.Cos(ToRadians(Latitude)) * Math.Cos(ToRadians(other.Latitude)) *
                   Math.Sin(dLon / 2) * Math.Sin(dLon / 2);

        return R * 2 * Math.Atan2(Math.Sqrt(a), Math.Sqrt(1 - a));
    }

    private static double ToRadians(double deg) => deg * Math.PI / 180.0;
}

var moscow = new GeoPoint(55.7558, 37.6173);
var spb = new GeoPoint(59.9343, 30.3351);

double distance = moscow.DistanceTo(spb);        // ~634 км
Console.WriteLine(moscow);                         // GeoPoint { Latitude = 55.7558, Longitude = 37.6173 }

// with выражение — создаёт копию с изменённым полем
var shifted = moscow with { Latitude = 56.0 };

// Equality — по значению (автоматически)
var point1 = new GeoPoint(55.75, 37.62);
var point2 = new GeoPoint(55.75, 37.62);
bool equal = point1 == point2;                     // true

// Деконструкция (автоматически из positional parameters)
var (lat, lon) = moscow;
```

### Когда использовать struct

```csharp
// ИСПОЛЬЗУЙ struct когда:
// 1. Данные маленькие (до ~16 байт)
// 2. Семантика значения (два объекта с одинаковыми данными = равны)
// 3. Неизменяемый (readonly struct / record struct)
// 4. Часто создаётся/уничтожается (избегаем GC)

// Хорошие примеры struct:
public readonly record struct UserId(Guid Value);
public readonly record struct Money(decimal Amount, string Currency);
public readonly record struct DateRange(DateOnly Start, DateOnly End)
{
    public int Days => End.DayNumber - Start.DayNumber;
    public bool Contains(DateOnly date) => date >= Start && date <= End;
}

// Strongly-typed ID — отличный use case для struct
public readonly record struct OrderId(Guid Value)
{
    public static OrderId New() => new(Guid.NewGuid());
    public static OrderId Empty => new(Guid.Empty);
    public override string ToString() => Value.ToString("N");
}

// НЕ используй struct когда:
// 1. Нужно наследование
// 2. Размер > 16 байт и структура часто передаётся по значению
// 3. Нужна мутабельность (mutable struct — источник багов!)
// 4. Используется как элемент коллекции с частым доступом по индексу
//    (boxing в non-generic коллекциях)

// АНТИПАТТЕРН: мутабельный struct
struct MutableBad
{
    public int Value;
    public void Increment() => Value++;
}

MutableBad s = new() { Value = 10 };
MutableBad copy = s;
copy.Increment();       // s.Value по-прежнему 10 — неожиданно для новичков!
```

---

## См. также

- [ООП и классы](oop.md)
- [Collections и LINQ](collections-linq.md)
- [Modern C#](modern-features.md)
- [GC, LOH и POH](../Runtime/gc-memory.md)
- [Span Deep Dive](../Runtime/span-layout.md)
