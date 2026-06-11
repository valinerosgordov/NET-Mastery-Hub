---
tags: [csharp, basics, junior, fundamentals, variables, control-flow, methods, mental-models]
level: Junior
date: 2026-05-04
---

# C# Basics — основы языка

> **Стартовая точка для Junior**. Не справочник синтаксиса — учебник с акцентом на **«почему C# устроен именно так»**: что компилятор делает за тебя, где живут переменные, какие проблемы решает каждая фича. После этого файла остальные файлы vault'а ложатся гораздо ровнее.

---

## 0. Как читать этот файл

Каждая тема построена по схеме: **зачем оно нужно → как это выглядит в коде → что под капотом → когда это ломается → ссылка на deep dive**. Если в каком-то месте появляется фраза «подробнее в [[oop|OOP]]» — это значит, я даю минимум для общей картины, а полное объяснение живёт в специализированном файле, чтобы здесь не утонуть.

Если ты пришёл из другого языка — обращай внимание на блоки **`> [!info] Если ты знаешь Python/JS/Java`**: я расставил якоря в ключевых местах.

---

## 1. Что это, зачем и когда

### 1.1. Что такое C#

**Язык программирования общего назначения**, разработанный в Microsoft (Anders Hejlsberg, 2000). Сегодня — open source, кроссплатформенный, на нём пишут backend (ASP.NET Core), desktop (WPF, Avalonia), игры (Unity), мобильные приложения (.NET MAUI), микросервисы, ML-модели (ML.NET).

### 1.2. Зачем C# существует — какие проблемы он решает

C# появился как ответ на боли, которые в начале 2000-х были у Java и C++:

| Боль | Решение C# |
|------|------------|
| C++ — ручное управление памятью, легко выстрелить себе в ногу | GC + safe-by-default (нет dangling pointers, нет use-after-free) |
| Java — многословный, нет value types, всё в heap | Структуры (`struct`), generics с reified типами, `var`, records, primary constructors |
| JavaScript / Python — нет проверки на этапе компиляции | Статическая типизация: половина багов ловится до запуска |
| C++ — зависимость от платформы, разные компиляторы | Один runtime (.NET), один формат бинарников (IL), JIT под целевую машину |

> [!info]- Если ты знаешь Python / JS
> Главное отличие — **статическая типизация**. В Python `x = 5; x = "hello"` — нормально. В C# — ошибка компиляции. Это не ограничение, а **гарантия**: компилятор и IDE знают про код больше, чем в Python, и ловят опечатки и несовместимости до того, как код запустится. Refactoring (`Rename` в IDE) работает безотказно. Автодополнение — точное.

> [!info]- Если ты знаешь Java
> C# изначально похож на Java синтаксически, но обогнал её: properties вместо геттеров/сеттеров, `var`, records, value types (`struct`), nullable reference types, pattern matching, primary constructors. Многое из того, что в Java добавляют сейчас (records, sealed, pattern matching) — в C# было давно.

### 1.3. Что такое .NET — платформа vs язык

**C# — это язык. .NET — это платформа.** Это разные вещи, и Junior часто путает.

```
        Твой код .cs
              │
              ▼
        ┌──────────┐
        │ Roslyn   │  ← компилятор C#
        │ (csc)    │
        └──────────┘
              │
              ▼
       Промежуточный язык (IL)        ← .dll / .exe файлы
              │
              ▼
        ┌──────────┐
        │ CLR      │  ← .NET runtime: загружает IL,
        │ + JIT    │     компилирует в машинный код,
        └──────────┘     управляет памятью, потоками
              │
              ▼
       Машинный код твоей CPU
```

- **C#** — синтаксис, который ты пишешь.
- **Roslyn (csc.exe)** — компилятор. Превращает `.cs` → `.dll`/`.exe` с IL-кодом внутри.
- **IL (Intermediate Language)** — байт-код, не зависящий от CPU.
- **CLR (Common Language Runtime)** — runtime: грузит сборки, управляет памятью (GC), запускает JIT.
- **JIT** — Just-In-Time компилятор. На лету превращает IL → машинный код под конкретную CPU.
- **BCL (Base Class Library)** — стандартная библиотека: `string`, `List<T>`, `File`, `Task` и т.д.

**Что это даёт:**
1. Один и тот же `.dll` запустится на Windows / Linux / macOS / ARM / x64 — JIT под капотом разный.
2. C#, F#, VB.NET компилируются в один и тот же IL — могут вызывать друг друга.
3. JIT может оптимизировать под актуальные данные на запуске (tiered compilation, PGO).

> **Когда тебе важна эта разница:** при чтении stack trace ("`at Module.Method() in file.cs:line 42`"), при использовании `dotnet publish --self-contained` (CLR упаковывается с приложением), при отладке производительности (cold start vs steady state).

### 1.4. Зачем C# именно сейчас

В 2026 году актуальная версия — C# 14 (.NET 10). Эволюция последних лет: nullable reference types (C# 8), records (C# 9), file-scoped namespaces и global usings (C# 10), raw string literals (C# 11), primary constructors и collection expressions (C# 12), `field` keyword (C# 13). Современный C# **значительно** короче и выразительнее старого. Если в туториалах ты видишь `class Program { static void Main(string[] args) { ... } }` — это устаревший стиль. Так больше не пишут.

См. [[csharp-language-design|C# Language Design]] для истории и [[modern-features|Modern Features]] для всех современных конструкций.

---

## 2. Первая программа — как код становится исполнением

### 2.1. Hello World

```csharp
// File: Program.cs
Console.WriteLine("Hello, World!");
```

Это **полная** программа в современном C# (top-level statements, .NET 6+). Компилируется, запускается. Под капотом компилятор разворачивает её в:

```csharp
namespace MyApp;

internal class Program
{
    private static void Main(string[] args)
    {
        System.Console.WriteLine("Hello, World!");
    }
}
```

Зачем оно есть: **снизить порог входа**. Раньше «Hello World» требовал 5 строк boilerplate, теперь — одну. Top-level statements удобны для скриптов, демо, маленьких утилит. В большом проекте всё равно появляются классы и `Program.cs` остаётся минимальным.

### 2.2. Создание и запуск проекта

```bash
dotnet new console -n MyApp     # создать новый console-проект
cd MyApp
dotnet run                       # скомпилировать и запустить
```

Что произошло:
1. `dotnet new console` создал папку `MyApp/` с `MyApp.csproj` (конфиг проекта) и `Program.cs`.
2. `dotnet run` запустил `dotnet restore` (скачал зависимости из NuGet), `dotnet build` (скомпилировал в `bin/Debug/net10.0/MyApp.dll`), и затем выполнил `.dll` через CLR.

Полный гайд по dotnet CLI и шаблонам — в [[dotnet-cli-getting-started|dotnet CLI: Getting Started]].

### 2.3. Как читать ошибки компилятора

Junior обычно паникует от красного текста. Но **ошибки компилятора C# — твой друг**: они почти всегда конкретны.

```csharp
int age = "John";
```

```
error CS0029: Cannot implicitly convert type 'string' to 'int'
```

Расшифровка: `CS0029` — код ошибки (можно загуглить, microsoft docs выдадут страницу с объяснением и фиксом). Текст — конкретно: тип `string` (`"John"`) нельзя положить в переменную типа `int`.

**Привычка Senior:** при ошибке `CSxxxx` сначала загугли код. У Microsoft есть полная документация на каждую ошибку. Не угадывай.

Самые частые на старте:
- `CS0103` — имя не существует в текущем контексте (опечатка, забыл `using`).
- `CS0029` / `CS0266` — несовместимые типы (попытка положить string в int и т.п.).
- `CS1002` — отсутствует `;` в конце строки.
- `CS0246` — не найден тип / namespace (забыл NuGet-пакет или `using`).
- `CS8600` / `CS8602` — nullable warnings (об этом отдельно в разделе про null).

---

## 3. Переменные и типы — фундамент

Этот раздел — **самый важный** в файле. Если ты усвоишь только модель из 3.2 и 3.3, остальное приложится.

### 3.1. Зачем нужны типы

В нетипизированном языке (Python, JS) переменная — это просто имя, к которому можно привязать что угодно:

```python
x = 5
x = "hello"     # OK
x = [1, 2, 3]   # OK
```

В типизированном (C#, Java, Rust, Go, TypeScript) — переменная объявляется с типом, и тип менять нельзя:

```csharp
int x = 5;
x = "hello";   // ❌ Compile error CS0029
```

**Что это даёт компилятору и тебе:**

1. **Ловля опечаток до запуска.** `user.Nmae` (опечатка) — ошибка компиляции, а не пустота в проде в три часа ночи.
2. **Точное автодополнение.** IDE знает что `user` — это `User`, и предлагает только методы и свойства `User`, ничего лишнего.
3. **Безопасный refactoring.** `Rename` переименовывает только реальные использования, а не все совпадения текста.
4. **Производительность.** Компилятор знает размер каждого типа, может разместить структуры эффективно, JIT генерит оптимальный машинный код. Для сравнения: Python вызывает `__add__` через словарь методов — медленно.
5. **Документация в коде.** Сигнатура `Order ProcessPayment(Customer customer, decimal amount)` сразу говорит что метод хочет и что возвращает. Без типов — иди читай тело, гадай.

«Типизация ограничивает» — миф. На практике она убирает целый класс багов. Цена — пара лишних слов в объявлении (часто компилятор сам выводит тип через `var`).

### 3.2. Stack vs Heap — первая ментальная модель

Это **базовая модель памяти**, которая нужна для понимания почти всего дальнейшего: value vs reference, передача в метод, GC, boxing, async, Span.

Когда программа исполняется, у неё две главные области памяти:

| Область | Что хранит | Скорость | Время жизни | Размер |
|---------|------------|----------|-------------|--------|
| **Stack** | Локальные переменные value-типов и **ссылки** | Очень быстро | До конца текущего метода | ~1 MB (по умолчанию) |
| **Heap** | Объекты reference-типов | Медленнее | До сборки мусора (GC) | Сколько RAM позволит |

```csharp
void DoWork()
{
    int count = 5;                 // на stack: 4 байта со значением 5
    string name = "Alice";          // на stack: ссылка (8 байт) указывает на heap
                                    // на heap: объект string "Alice"
    Order order = new Order();      // на stack: ссылка
                                    // на heap: объект Order
}
// При выходе из DoWork:
//   stack очищается автоматически (просто двигается указатель stack pointer)
//   объекты в heap остаются — пока на них есть ссылки
//   когда GC решает что на них никто не ссылается — освобождает
```

**Почему это важно:**

- **Stack очень быстрый**, потому что выделение = «двинуть указатель». Освобождение тоже. Никакого GC.
- **Heap медленнее**, потому что аллокатор ищет свободное место, а GC периодически останавливает программу для уборки.
- Поэтому **value types на стеке**, когда возможно — это hot path для производительности.

> [!info]- Эта картина — упрощение
> На самом деле value-типы могут жить **не на стеке**: если value-тип — поле reference-объекта, он лежит inline внутри объекта в heap. Если value-тип захватывается замыканием или async-методом — компилятор может создать класс-обёртку и положить его в heap. Полная картина в [[types-and-memory|Types and Memory]] и [[gc-memory|GC и память]].

> [!info]- Если ты знаешь Java
> В Java на стеке только примитивы (`int`, `double` и т.д.) и ссылки. **Все** объекты — в heap, всегда. В C# ты можешь объявить свой тип как `struct`, и он будет жить на стеке. Для маленьких immutable значений это огромный выигрыш по аллокациям.

> [!info]- Если ты знаешь Python / JS
> В обоих языках всё «по ссылке», за исключением immutable примитивов с copy-on-modify семантикой. В C# — два чёткие лагеря: value types копируются по значению, reference types — по ссылке. Это ты увидишь в каждом методе, передающем параметры.

### 3.3. Value types vs Reference types — два лагеря

**Value type** — содержит данные напрямую. При присваивании / передаче в метод **копируется значение**.

**Reference type** — содержит **ссылку на объект в heap**. При присваивании / передаче в метод **копируется ссылка** (сам объект — один).

```csharp
// Value type: int (struct под капотом)
int a = 42;
int b = a;        // создалась независимая копия со значением 42
b = 100;          // a по-прежнему 42

// Reference type: array (class)
int[] arr1 = [1, 2, 3];
int[] arr2 = arr1;     // arr1 и arr2 указывают на тот же массив в heap
arr2[0] = 999;          // arr1[0] тоже стал 999 — это один и тот же объект!
```

**Кто к какому лагерю относится:**

| Категория | Value types | Reference types |
|-----------|-------------|-----------------|
| Встроенные | `int`, `long`, `double`, `decimal`, `bool`, `char`, `byte`, ... | `string`, массивы (`int[]`), `object`, делегаты |
| Создаваемые | `struct`, `record struct`, `enum` | `class`, `record class`, `interface` |

> **`string` — особый случай:** это reference type, но ведёт себя «как value type» (immutable + value-based equality + перегруженный `==`). Из-за этого Junior часто думает что это value type. Технически — нет.

#### Передача в метод — где Junior спотыкается чаще всего

```csharp
void SetToZero(int x)        { x = 0; }
void ResetArray(int[] arr)   { arr[0] = 0; }
void Replace(int[] arr)      { arr = new int[] { 7, 7, 7 }; }

int n = 5;
SetToZero(n);
// n всё ещё 5 — value type, метод работал с КОПИЕЙ

int[] data = [1, 2, 3];
ResetArray(data);
// data[0] стал 0 — метод изменил САМ объект через скопированную ссылку

Replace(data);
// data всё ещё [0, 2, 3] — метод заменил СВОЮ копию ссылки,
// но снаружи это не видно. Сам объект не тронут.
```

Чтобы метод **переопределил** ссылку наружу — есть `ref`:

```csharp
void Replace(ref int[] arr) { arr = new int[] { 7, 7, 7 }; }

int[] data = [1, 2, 3];
Replace(ref data);
// data теперь [7, 7, 7]
```

Полный разбор `ref`/`out`/`in` — в [[keywords-reference|Keywords Reference]].

### 3.4. Базовые value-типы

#### Целочисленные

| Тип    | Размер  | Диапазон                               | Когда использовать |
|--------|---------|----------------------------------------|--------------------|
| `byte` | 1 байт  | 0 … 255                                | Байтовые буферы, цвета (RGB) |
| `short`| 2 байта | −32 768 … 32 767                       | Редко. Только если экономишь память в массивах |
| `int`  | 4 байта | ≈ ±2.1 миллиарда                       | **Дефолт.** 99% случаев |
| `long` | 8 байт  | ≈ ±9.2 квинтиллиона                    | ID в больших системах, timestamp в миллисекундах, файловые offset'ы |

```csharp
int count = 42;
long bigId = 9_000_000_000L;     // L — это long literal
                                  // _ — разделитель для читаемости (не часть числа)
```

**Почему столько целочисленных типов?** Исторически — экономия памяти и пропускной способности шины. Сегодня в обычном backend это редко важно, но в hot path (миллионы записей в массиве, протоколы по сети) разница `byte[]` vs `int[]` это 4× памяти и 4× cache pressure.

> [!warning] Overflow по умолчанию молчаливый
> ```csharp
> int max = int.MaxValue;
> int oops = max + 1;   // -2147483648, без ошибки!
> ```
> В release-сборке арифметика по умолчанию **не проверяет переполнение** (`unchecked`). Хочешь проверки — оборачивай в `checked { ... }` или включи в проекте `<CheckForOverflowUnderflow>true</CheckForOverflowUnderflow>` (только для Debug, перформанс падает). Подробно в [[keywords-reference|keywords-reference]].

#### Дробные

| Тип       | Размер  | Точность  | Скорость | Когда |
|-----------|---------|-----------|----------|-------|
| `float`   | 4 байта | ~7 цифр   | Очень быстро | Графика, ML, игры — где скорость важнее точности |
| `double`  | 8 байт  | ~15-17 цифр | Быстро | **Дефолт** для дробных. Научные расчёты, физика |
| `decimal` | 16 байт | ~28-29 цифр | В 5-10 раз медленнее | **Деньги**, бухгалтерия, всё где нужна точность до копейки |

```csharp
float pi = 3.14f;          // f-suffix
double e = 2.71828;        // дефолт для дробных
decimal price = 19.99m;    // m-suffix (от Money)
```

> [!warning] Деньги — всегда `decimal`. Никогда `double`/`float`.
> ```csharp
> double a = 0.1 + 0.2;       // 0.30000000000000004 — ошибка в 17-м знаке
> decimal b = 0.1m + 0.2m;    // 0.3 — точно
> ```
> **Почему так:** `double` хранит число в двоичной форме с фиксированным количеством бит. Десятичная дробь `0.1` не имеет конечного двоичного представления (как `1/3` не имеет конечного десятичного — `0.333...`). Округление при каждой операции даёт накопленную ошибку. `decimal` хранит число в десятичной форме (96-битная мантисса × степень 10) — `0.1` представим точно. Цена точности — скорость и память (в 5-10 раз медленнее, в 2 раза больше памяти).

#### Boolean

```csharp
bool isActive = true;
bool isDeleted = false;
```

Только `true` / `false`. Нельзя `if (1)` как в C/JavaScript — компилятор не позволит:

```csharp
int x = 5;
if (x) { ... }   // ❌ CS0029: Cannot implicitly convert type 'int' to 'bool'
if (x != 0) { ... }   // ✅
```

Это сознательное ограничение C# — убрать классический баг `if (x = 5)` (присваивание вместо сравнения), который в C/JS компилируется без вопросов.

#### char и string

```csharp
char letter = 'A';                  // одинарные кавычки, ровно 1 символ (UTF-16 unit)
string name = "Hello";              // двойные кавычки

// Escape sequences
string s = "Line 1\nLine 2";        // \n — перенос строки
string path = "C:\\Users";          // \\ — обратный слэш
string verbatim = @"C:\Users";      // verbatim string — escape отключен
string interp = $"Hello, {name}!";  // string interpolation — выражение в {}

// Raw string literal (C# 11+) — для JSON, SQL, HTML без экранирования
string json = """
    {
        "name": "John",
        "age": 30
    }
    """;
```

Глубокий разбор строк (StringBuilder, Span, performance, Regex) — в [[strings-regex|Strings и Regex]].

#### DateTime

```csharp
DateTime now = DateTime.Now;              // local time — почти всегда НЕ то что нужно
DateTime utc = DateTime.UtcNow;            // UTC — то что хранят в БД и логах
DateTime birthday = new(2000, 1, 15);
```

Подробнее, включая часовые пояса, NodaTime, частые баги — в [[datetime-timezones|DateTime и Timezones]].

### 3.5. Объявление переменных и `var`

Полная форма — пишешь тип явно:

```csharp
int count = 5;
List<User> users = new List<User>();
string name = "Alice";
```

С `var` компилятор выводит тип из правой части — короче, эквивалентный код:

```csharp
var count = 5;                   // → int
var users = new List<User>();    // → List<User>
var name = "Alice";              // → string
```

`var` **не делает переменную динамической** — тип всё равно статический и фиксирован, просто не написан вручную. Это сахар, в IL-коде нет никакой разницы.

Чего `var` не умеет:

```csharp
var x;            // ❌ CS0818: cannot use var without an initializer
var nothing = null;  // ❌ нельзя вывести тип из null
```

#### Когда `var`, когда явный тип — decision tree

> [!info] Decision: var или explicit?
> 1. **Тип очевиден из правой части?** → `var`
>    `var users = new List<User>();` — `List<User>` дважды писать незачем.
> 2. **Тип не очевиден (метод возвращает `object`, `T`, или сложное LINQ-выражение)?** → пиши явно
>    `decimal total = CalculateTotal(order);` — читателю важно сразу видеть что это `decimal`, а не `int`.
> 3. **Числовой литерал?** → пиши явно если важен конкретный тип
>    `var x = 5;` даёт `int`. Если ты хотел `long` или `decimal` — пиши `long x = 5;` или `decimal x = 5m;`.
> 4. **Анонимный тип, кортеж, длинное generic-имя?** → `var` обязателен или сильно желателен
>    `var pair = (Name: "Alice", Age: 30);`

Команды иногда требуют **always var** или **never var**. Аргументы у обеих сторон есть. Главное — единый стиль в проекте, настроенный через `.editorconfig`.

### 3.6. const, readonly, static readonly — три разных «не меняется»

Все три объявляют что значение не меняется. Но они **не взаимозаменяемы**.

| Конструкция | Значение известно | Где живёт | Что можно |
|-------------|-------------------|-----------|-----------|
| `const` | На этапе компиляции | Запекается в каждое использование | Только примитивы и `string` |
| `readonly` | На этапе создания экземпляра | Поле объекта | Любой тип. Присваивается в declaration или конструкторе |
| `static readonly` | На этапе первого обращения к классу | Один раз для всего процесса | Любой тип. Присваивается в static конструкторе или в declaration |

```csharp
public class Config
{
    public const int MaxRetries = 3;            // запеклось в каждый callsite
    public const string Version = "1.0.0";

    public readonly int Port;                    // присвоить в конструкторе
    public readonly DateTime CreatedAt;

    public static readonly DateTime AppStart = DateTime.UtcNow;  // вычислится 1 раз

    public Config(int port)
    {
        Port = port;
        CreatedAt = DateTime.UtcNow;
    }
}
```

> [!warning] `const` — невидимый contract
> Когда ты пишешь `public const int MaxRetries = 3;` в библиотеке, и потребитель пишет `if (count > Config.MaxRetries)`, компилятор **запекает 3 в код потребителя**. Если ты обновишь библиотеку до `MaxRetries = 5` и не пересоберёшь потребителя — у потребителя останется 3.
>
> **Правило:** для значений, которые никогда не изменятся (математические константы, magic strings) — `const`. Для всего остального — `static readonly`.

### 3.7. Naming conventions — на одной странице

| Что | Стиль | Пример |
|-----|-------|--------|
| Класс, struct, record, enum, interface (с `I`-префиксом) | `PascalCase` | `UserService`, `IRepository`, `OrderStatus` |
| Метод, property, public/protected поле, event | `PascalCase` | `GetById`, `IsActive`, `OnUpdated` |
| Локальная переменная, параметр | `camelCase` | `userId`, `currentAge` |
| Private поле | `_camelCase` | `_repository`, `_cache` |
| Constant (`const`) | `PascalCase` | `MaxRetries` (исторически было `UPPER_CASE`, но сейчас уже не пишут) |
| Generic type parameter | `T` или `TName` | `T`, `TKey`, `TValue`, `TUser` |

Полный гайд с case studies, как переименовать плохое имя в хорошее, как называть booleans/factories/handlers — в [[naming-conventions|Naming Conventions]].

---

## 4. Операторы

### 4.1. Арифметические — и почему `10 / 3 == 3`

```csharp
int sum = 5 + 3;          // 8
int diff = 5 - 3;          // 2
int product = 5 * 3;       // 15
int quotient = 10 / 3;     // 3 — НЕ 3.33!
int remainder = 10 % 3;    // 1 — modulo
double precise = 10.0 / 3; // 3.333...

x++;     // post-increment: использует, потом увеличивает
++x;     // pre-increment: увеличивает, потом использует
x += 5;  // x = x + 5

int a = 10;
a *= 2;  // 20
a /= 4;  // 5
```

> [!warning] Тип результата = тип операндов
> **Механизм:** в C# арифметическая операция возвращает тип операндов. Если оба `int` — результат `int`. Деление с отбрасыванием дробной части — это **математически нормальное** integer division (как в C, Java, Go).
> ```csharp
> int x = 10 / 3;          // 3 (int / int → int)
> double y = 10 / 3;       // 3.0 — деление было целочисленным, потом результат привели к double
> double z = 10.0 / 3;     // 3.333... — один из операндов double, оба приводятся к double
> double w = (double)10 / 3;  // 3.333...
> ```
> **Правило:** если хочешь дробный результат — хотя бы один операнд должен быть дробным.

### 4.2. Сравнение

```csharp
5 == 5;     // true
5 != 3;     // true
5 > 3;      // true
5 < 3;      // false
5 >= 5;     // true
5 <= 4;     // false
```

Для value-типов сравнивается значение. Для reference-типов `==` по умолчанию сравнивает **ссылки** (тот же объект или нет). `string` и `record` переопределяют `==` на сравнение по значению. Подробнее в [[equality-comparison|Equality and Comparison]].

### 4.3. Логические — и зачем short-circuit

```csharp
bool a = true;
bool b = false;

a && b;    // false — AND
a || b;    // true  — OR
!a;        // false — NOT
```

`&&` и `||` — **short-circuit**: если первого операнда хватает чтобы определить результат, второй **не вычисляется**.

```csharp
// Защита от NRE
if (user != null && user.IsActive) { ... }
//   ^^^^^^^^^^^^   ^^^^^^^^^^^^^
//   если null — второе НЕ выполнится, no NRE
```

Без short-circuit `user.IsActive` упало бы с `NullReferenceException`, потому что `user == null`. Это **самый частый идиом** для null-safety до того, как ты освоишь `?.`.

Есть еще `&` и `|` — bitwise операторы; для bool они работают как AND/OR, но **без** short-circuit. Они нужны только когда тебе **обязательно** нужно вычислить оба операнда (например, оба имеют side-effect). 99% случаев — пиши `&&` и `||`.

### 4.4. Null-операторы — без них в современном C# никак

```csharp
string? name = null;

// ?? — null coalescing: «если null, бери справа»
string display = name ?? "Anonymous";

// ??= — null coalescing assignment: «если null, присвоить»
name ??= "Default";

// ?. — null conditional: безопасный доступ к члену
int? length = name?.Length;
//      если name == null, length == null без NRE

// ?[i] — null conditional indexer
char? first = name?[0];

// ?. через цепочку
int? cityLength = order?.Customer?.Address?.City?.Length;

// ! — null forgiving: «я знаю что не null, не предупреждай»
int len = name!.Length;     // если name всё-таки null — кинется NRE в runtime
```

`!` — **последнее средство**, обещание компилятору которое ты не можешь объяснить через типы. Используется редко, в местах где компилятор не может вывести non-null (например, после `if (name is not null)` в lambda с захватом).

Глубже про null — в разделе 8 этого файла и в [[nullable-types|Nullable Types]].

### 4.5. Ternary

```csharp
int age = 25;
string status = age >= 18 ? "Adult" : "Minor";
```

Ternary имеет смысл когда обе ветки **короткие** и **возвращают значение**. Если ветки длинные — пиши `if/else`, не пихай тернарник в одну строку.

```csharp
// ❌ нечитаемо — вложенный тернарник
var sign = a > 0 ? (b > 0 ? "++" : "+-") : (b > 0 ? "-+" : "--");
```

```csharp
// ✅ так лучше — switch expression с tuple pattern
var sign = (a > 0, b > 0) switch
{
    (true,  true)  => "++",
    (true,  false) => "+-",
    (false, true)  => "-+",
    (false, false) => "--",
};
```

### 4.6. Type операторы: is, as, cast

```csharp
object obj = "hello";

// is — проверка типа, опционально с pattern (declaration)
if (obj is string s)
{
    Console.WriteLine(s.Length);   // s в этом блоке — типа string
}

// is с property pattern
if (order is { Status: OrderStatus.Paid, Total: > 100 })
{
    ApplyDiscount(order);
}

// as — попытка привести (null если не получилось)
string? maybe = obj as string;
if (maybe != null) { /* ... */ }

// (T) — явный cast (бросает InvalidCastException если не получилось)
string s2 = (string)obj;  // OK для obj == "hello"
int n = (int)obj;         // ❌ InvalidCastException
```

> [!info] Когда `is`, когда `as`, когда `(T)`
> - `is X x` — самый идиоматичный способ. Проверка + объявление в одном.
> - `as X` — когда дальше будет `if (x != null)` и тебе нужно cast без exception. В современном коде почти всегда заменяется на `is`.
> - `(T)` — когда ты **уверен** что cast пройдёт. Если не пройдёт — exception. Используется в horizontal hierarchy (например, парсинг JSON в типизированную модель), редко в обычном коде.

---

## 5. Управление потоком

### 5.1. if / else и почему скобки `{ }` всегда

```csharp
int age = 25;

if (age < 13)        Console.WriteLine("Child");
else if (age < 18)   Console.WriteLine("Teen");
else                 Console.WriteLine("Adult");
```

Можно опускать `{ }` для одного оператора. **Не делай так.** Это породило целую категорию багов уровня *Apple goto fail* (2014, утечка SSL). Скобки всегда:

```csharp
if (age >= 18) {
    Console.WriteLine("Adult");
}
```

В Microsoft style вообще ставят скобку на отдельной строке (`Allman`):

```csharp
if (age >= 18)
{
    Console.WriteLine("Adult");
}
```

Это спор уровня tabs vs spaces — главное настроить `.editorconfig` и не отвлекаться.

### 5.2. switch — старый и новый

#### Старый stateful switch

```csharp
switch (role)
{
    case "admin":
        AccessFull();
        break;
    case "user":
        AccessLimited();
        break;
    case "guest":
    case "visitor":         // несколько case, один обработчик
        AccessReadOnly();
        break;
    default:
        AccessDenied();
        break;
}
```

Проблемы старого switch: обязательный `break` (фоллтру по умолчанию — наследие C), много синтаксиса, не возвращает значение.

#### Switch expression (C# 8+) — современный

```csharp
string accessLevel = role switch
{
    "admin"            => "Full",
    "user"             => "Limited",
    "guest" or "visitor" => "ReadOnly",
    _                  => "Denied"   // _ — default
};
```

Возвращает значение, нет `break`, компактнее. **Используй везде где можешь.** Старый `switch` остаётся для случаев со side-effects в каждой ветке (хотя и тут можно жить с if/else).

Глубже про pattern matching (type, property, list, tuple, relational патерны) — в [[modern-features|Modern Features]].

### 5.3. Циклы — какой когда

```csharp
// for — известное число итераций, нужен индекс
for (int i = 0; i < 10; i++) { Console.WriteLine(i); }
for (int i = 10; i > 0; i--) { /* обратный ход */ }
for (int i = 0; i < 100; i += 2) { /* шаг 2 */ }

// foreach — итерация по любому IEnumerable<T>
string[] names = ["Alice", "Bob", "Charlie"];
foreach (var name in names) { Console.WriteLine(name); }

Dictionary<string, int> ages = new() { ["Alice"] = 30, ["Bob"] = 25 };
foreach (var (name, age) in ages) { /* ... */ }

// while — пока условие, может не выполниться ни разу
int countdown = 10;
while (countdown > 0) { Console.WriteLine(countdown--); }

// do-while — пока условие, выполняется минимум 1 раз
string? input;
do { input = Console.ReadLine(); }
while (input != "quit");
```

> [!info] Decision: какой цикл?
> - **Нужен индекс или нестандартный шаг?** → `for`
> - **Просто пройти по коллекции?** → `foreach` (предпочтительно — IDE/JIT оптимизируют его лучше всего)
> - **Условие зависит от состояния, не от счётчика?** → `while`
> - **Минимум один проход?** → `do-while`
> - **LINQ может это выразить чище?** → используй LINQ вместо ручного цикла (где не критична производительность)

### 5.4. break и continue

```csharp
// break — выйти из текущего цикла
for (int i = 0; i < 100; i++)
{
    if (i == 5) break;       // выходим
    Console.WriteLine(i);     // 0, 1, 2, 3, 4
}

// continue — следующая итерация
for (int i = 0; i < 10; i++)
{
    if (i % 2 == 0) continue; // пропускаем чётные
    Console.WriteLine(i);     // 1, 3, 5, 7, 9
}
```

**Внутри вложенных циклов** `break` выходит только из своего. Если нужно из обоих — выноси в отдельный метод и `return` (чище чем `goto`).

---

## 6. Методы

### 6.1. Базовая структура

```csharp
[modifiers] returnType Name(params)
{
    body
    return value;
}
```

```csharp
public int Add(int a, int b)
{
    return a + b;
}

int sum = Add(5, 3);   // 8
```

### 6.2. void и expression-bodied

```csharp
// void — метод не возвращает значение
public void PrintGreeting(string name)
{
    Console.WriteLine($"Hello, {name}!");
}

// Expression-bodied (=>) — для коротких методов в одну строку
public bool IsEven(int n) => n % 2 == 0;
public string FullName(User u) => $"{u.FirstName} {u.LastName}";
```

`=>` читается как «возвращает». Используется когда тело — одно выражение. Для нескольких строк — обычный `{ }`.

### 6.3. Параметры — required, optional, named

```csharp
// Required parameters
public void Greet(string name, int age) { }
Greet("John", 25);

// Optional parameters (default values)
public void Greet(string name, int age = 18) { }
Greet("John");          // age = 18
Greet("John", 25);      // age = 25

// Named arguments — порядок неважен, читаемость лучше
Greet(name: "John", age: 25);
Greet(age: 30, name: "Alice");
```

> [!warning] Default-параметры запекаются в callsite
> Тот же эффект что и у `const`. Если ты опубликовал библиотеку с `Greet(string name, int age = 18)`, и потребитель вызвал `Greet("John")`, в его сборке запеклось `Greet("John", 18)`. Меняешь default на 21 — у старого потребителя всё равно 18 пока он не пересобран.
>
> Для public API библиотек предпочтительнее **method overloading** (см. ниже) — он стабильнее при эволюции версий.

### 6.4. Method overloading — несколько методов с одним именем

Несколько перегрузок с разными параметрами:

```csharp
public int Add(int a, int b) => a + b;
public double Add(double a, double b) => a + b;
public int Add(int a, int b, int c) => a + b + c;

Add(1, 2);          // вызовет int-version
Add(1.5, 2.5);      // вызовет double-version
Add(1, 2, 3);       // вызовет 3-аргументную
```

Компилятор выбирает по типам и количеству параметров. Возвращаемый тип в выборе **не участвует** — нельзя сделать две перегрузки которые отличаются только им.

### 6.5. ref / out / in — параметры по ссылке

Минимально (deep dive в [[keywords-reference|Keywords Reference]]):

```csharp
// out — метод обязан установить значение
public bool TryParse(string s, out int result)
{
    return int.TryParse(s, out result);
}
TryParse("42", out int x);   // x = 42

// ref — pass by reference, и вход и выход
public void Swap(ref int a, ref int b)
{
    (a, b) = (b, a);   // tuple deconstruction
}
int x = 1, y = 2;
Swap(ref x, ref y);   // x=2, y=1

// in — readonly reference, оптимизация для больших struct
public double Distance(in Point a, in Point b)
{
    // a и b нельзя изменить, но и копия не создаётся
    return Math.Sqrt(...);
}
```

`out` чаще всего ты увидишь в **TryParse-паттерне** — стандартный идиом для парсинга без exception.

### 6.6. params — переменное число аргументов

```csharp
public int Sum(params int[] numbers)
{
    int total = 0;
    foreach (var n in numbers) total += n;
    return total;
}

Sum(1, 2);                  // 3
Sum(1, 2, 3, 4, 5);          // 15
Sum();                       // 0
Sum([1, 2, 3]);              // тоже работает — передали массив явно
```

В современном .NET 9+ `params` поддерживает не только массивы, но и `Span<T>`, `IEnumerable<T>` — без аллокации массива на каждый вызов:

```csharp
public int Sum(params ReadOnlySpan<int> numbers)
{
    int total = 0;
    foreach (var n in numbers) total += n;
    return total;
}
```

### 6.7. Local functions — методы внутри методов

```csharp
public int CalculateTotal(int[] prices)
{
    int total = 0;
    foreach (var p in prices)
        total += ApplyDiscount(p);
    return total;

    // Local function — видна только внутри CalculateTotal
    int ApplyDiscount(int price) => price > 100 ? price * 9 / 10 : price;
}
```

**Когда:** нужна вспомогательная функция, которую нельзя нормально назвать на уровне класса (слишком специфична). Также — для рекурсии в `IEnumerable`/`async` методах (помогает компилятору избежать накладных расходов на итераторе/state machine).

---

## 7. Классы — самые основы

Это **минимум**. Всё что глубже (наследование, интерфейсы, abstract, sealed, polymorphism, records vs classes vs structs) — в [[oop|OOP и классы]].

### 7.1. Класс vs объект — template vs instance

**Класс** — описание (template). **Объект (instance)** — конкретное «создание» класса.

```csharp
// Класс User — описание
public class User
{
    public string Name { get; set; }
    public int Age { get; set; }
}

// Объект user — конкретный экземпляр класса User
User user = new User { Name = "Alice", Age = 30 };
```

В памяти `User` (класс) — это метаданные (живут один раз на программу). `user` (объект) — это аллокация в heap (живёт пока на неё ссылаются). Один класс — много объектов.

### 7.2. Поля и свойства — почему свойства

```csharp
public class User
{
    public string Name;     // ← поле (field)
    public int Age;
}
```

Это работает, но Junior часто пишет именно так — и зря. **В C# принято использовать properties вместо публичных полей.**

```csharp
public class User
{
    public string Name { get; set; }    // ← свойство (property)
    public int Age { get; set; }
}
```

Снаружи это выглядит **идентично** (`user.Name = "..."`, `var n = user.Name;`), но даёт три критических преимущества:

#### 1. Возможность добавить логику без слома потребителей

Сегодня — простое свойство. Завтра возникает требование валидации:

```csharp
public class User
{
    private int _age;
    public int Age
    {
        get => _age;
        set
        {
            if (value < 0) throw new ArgumentException("Age cannot be negative");
            _age = value;
        }
    }
}
```

Потребители кода ничего не заметили — синтаксис вызова такой же. Если бы это было поле, тебе пришлось бы переименовать его или ломать API.

#### 2. Бинарная совместимость

Reflection / databinding / serializers / EF Core / ORM работают со свойствами по-разному, чем с полями. ASP.NET Core при привязке модели смотрит **только на свойства**. Если у тебя поле — оно не привяжется к JSON.

#### 3. Возможность read-only через init / get-only

```csharp
public class User
{
    public required string Email { get; init; }   // обязательно при создании
                                                   // потом не изменить
    public Guid Id { get; } = Guid.NewGuid();      // только в конструкторе/declaration
    public string Name { get; set; } = "";          // обычное r/w свойство
}
```

`init` (C# 9) — задать только при инициализации (`new User { Email = "..." }`), потом не изменить. `required` (C# 11) — компилятор требует задать.

### 7.3. Конструкторы — три подхода

```csharp
// 1. Параметризованный конструктор — классика
public class User
{
    public string Name { get; }
    public int Age { get; }

    public User(string name, int age)
    {
        Name = name;
        Age = age;
    }
}

var u1 = new User("Alice", 30);
```

```csharp
// 2. Object initializer + init/required — современный, без явного конструктора
public class User
{
    public required string Name { get; init; }
    public required int Age { get; init; }
}

var u2 = new User { Name = "Alice", Age = 30 };
```

```csharp
// 3. Primary constructor (C# 12+) — для DI и простых случаев
public class UserService(IUserRepository repo, ILogger<UserService> logger)
{
    public async Task<User?> GetById(Guid id)
    {
        logger.LogInformation("Getting user {Id}", id);
        return await repo.GetByIdAsync(id);
    }
}
```

> [!info] Decision: какой выбрать?
> - **DTO / value object / data class?** → object initializer + `required ... init;`
> - **Сервис с зависимостями (DI)?** → primary constructor
> - **Сложная логика инициализации (валидация, computed fields)?** → классический конструктор
> - **Несколько способов создать объект?** → классический конструктор + перегрузки или factory methods

### 7.4. static члены — принадлежат классу, не объекту

```csharp
public class MathUtils
{
    public static double Pi { get; } = 3.14159;
    public static int Add(int a, int b) => a + b;
}

// Доступ без создания экземпляра
double pi = MathUtils.Pi;
int sum = MathUtils.Add(2, 3);
```

`static` — общий для всех «использований». Часто это утилитные функции, константы, фабрики. Если **весь** класс состоит из `static` — пометь его `static class`, нельзя будет создать экземпляр случайно.

### 7.5. Records — для DTO / Value Objects

```csharp
public record User(string Name, int Age);

var u1 = new User("Alice", 30);
var u2 = new User("Alice", 30);

u1 == u2;      // true! сравнение по значению, не по ссылке

// non-destructive update — создаёт копию с изменением
var u3 = u1 with { Age = 31 };
```

Под капотом компилятор генерит конструктор, свойства, `Equals`, `GetHashCode`, `ToString`, `Deconstruct` и оператор `==`. **Когда твой класс — это пакет данных без поведения** (DTO, событие, value object) — пиши `record`, не `class`.

Полное сравнение `class` / `record class` / `struct` / `record struct`, когда что — в [[oop|OOP]] и [[modern-features|Modern Features]].

---

## 8. Null — отдельная боль

Null — изобретение Tony Hoare 1965 года, которое он называл «my billion-dollar mistake». В C# с этим живут, но современные инструменты сильно облегчают жизнь.

### 8.1. Что такое NullReferenceException

```csharp
string? name = null;
int len = name.Length;   // 💥 NullReferenceException at runtime
```

NRE — самый частый runtime-баг в .NET. Происходит когда обращаешься к члену объекта, который на самом деле `null`.

### 8.2. Nullable Reference Types (NRT) — компилятор тебе помогает

В современных проектах включён NRT (`<Nullable>enable</Nullable>` в `.csproj`). Это меняет правила:

Объявления:

```csharp
string a = null;        // ❌ CS8600: warning, nullable assigned to non-nullable
string? b = null;       // ✅ b явно может быть null
```

Использование nullable значения:

```csharp
string? maybe = GetMaybe();

string bad  = maybe;                   // ❌ CS8600: nullable → non-nullable
string ok1  = maybe ?? "default";      // ✅ null coalescing
string ok2  = maybe!;                  // ✅ null forgiving — на твою ответственность
```

`?` после типа — явно говорит «может быть null». Компилятор начинает следить:
- Не давать присваивать `null` в non-nullable.
- Не давать обращаться к членам без проверки или null-conditional.
- Делать flow analysis: после `if (x != null)` он считает что в этом блоке `x` гарантированно не null.

> [!warning] NRT — это warnings, не ошибки
> NRT по умолчанию выдаёт **warnings**, а не errors. Программа всё равно скомпилируется и запустится. Чтобы превратить в errors:
> ```xml
> <PropertyGroup>
>   <Nullable>enable</Nullable>
>   <TreatWarningsAsErrors>true</TreatWarningsAsErrors>
> </PropertyGroup>
> ```
> В качественных проектах так и делают. Игнорить NRT-warnings — это занимать долг.

### 8.3. Nullable value types — `int?` и его развёртка

Для value-типов работает иначе. `int` всегда имеет значение (по умолчанию 0). Чтобы дать ему возможность быть «отсутствующим», есть `Nullable<T>` (синтаксис `T?`):

```csharp
int x = 5;             // не может быть null
int? y = null;          // может
int? z = 42;

int w = y ?? 0;         // если null, бери 0
int w2 = y.Value;       // 💥 InvalidOperationException если null

if (y.HasValue) { /* y.Value доступно */ }
```

Под капотом `int?` — это `Nullable<int>`, struct с двумя полями: bool `HasValue` и `int Value`.

### 8.4. Идиомы null-safety

```csharp
// 1. Проверка перед использованием
if (user != null) { Console.WriteLine(user.Name); }

// 2. Null-conditional (?.)
Console.WriteLine(user?.Name);   // "Alice" или null, без NRE

// 3. Null-coalescing (??)
string name = user?.Name ?? "Anonymous";

// 4. Pattern matching с is not null
if (user is not null) { Console.WriteLine(user.Name); }

// 5. ArgumentNullException на входе метода (.NET 7+)
public void Process(User user)
{
    ArgumentNullException.ThrowIfNull(user);
    // дальше user гарантированно не null
}
```

Глубже про null, NRT, `Nullable<T>`, attributes (`[NotNull]`, `[MaybeNull]`) — в [[nullable-types|Nullable Types]].

---

## 9. Базовые идиомы — что встретишь в любом коде

### 9.1. Console I/O

```csharp
Console.Write("Enter name: ");          // без перевода строки
string? name = Console.ReadLine();       // читает строку или null (если EOF)

Console.Write("Enter age: ");
if (int.TryParse(Console.ReadLine(), out int age))
{
    Console.WriteLine($"Hello, {name}! You are {age}.");
}
else
{
    Console.WriteLine("Not a valid number.");
}
```

`Console.ReadLine()` возвращает `string?` — может вернуть `null` если поток закрыт. Игнорить это в production-коде нельзя.

### 9.2. Массивы и List\<T\>

```csharp
// Массив — фиксированный размер, выделяется один раз
int[] numbers = [1, 2, 3, 4, 5];           // collection expression (C# 12+)
int[] empty = new int[10];                  // 10 нулей
numbers[0];                                 // первый
numbers[^1];                                // последний (from-end index, C# 8+)
numbers[1..3];                              // slice [2, 3] (range, C# 8+)
numbers.Length;

// List<T> — динамический массив, рос по мере добавления
using System.Collections.Generic;

List<string> names = ["Alice", "Bob"];      // collection expression
names.Add("Charlie");
names.Remove("Bob");
names.Insert(0, "Zoe");
names.Count;
names.Contains("Alice");
```

В большинстве реального кода ты будешь работать с `List<T>`, не с массивом. Массивы — низкоуровневые, фиксированного размера, чаще всего встречаются в API/перформансе.

Подробно про collections (List, Dictionary, HashSet, Queue, Stack, FrozenDictionary, ImmutableArray, LINQ) — в [[collections-linq|Collections и LINQ]].

### 9.3. Dictionary\<K,V\> — словарь

```csharp
Dictionary<string, int> ages = new() { ["Alice"] = 30, ["Bob"] = 25 };

ages["Charlie"] = 35;        // добавить или обновить

int aliceAge = ages["Alice"];  // 30
                                // 💥 KeyNotFoundException если ключа нет!

// Безопасное чтение
if (ages.TryGetValue("Alice", out int age))
{
    Console.WriteLine(age);
}

ages.ContainsKey("Bob");       // true
ages.Remove("Bob");

foreach (var (name, value) in ages) { Console.WriteLine($"{name}: {value}"); }
```

Когда нужен быстрый поиск по ключу (O(1) в среднем) — `Dictionary`. Когда коллекция read-only и заполняется один раз — рассмотри `FrozenDictionary` (.NET 8+).

### 9.4. TryParse — парсинг без exception

```csharp
// ❌ Плохо: exception на каждый невалидный ввод
try { int n = int.Parse(input); }
catch (FormatException) { /* ... */ }

// ✅ Хорошо: TryParse возвращает bool, exception нет
if (int.TryParse(input, out int n))
{
    UseNumber(n);
}
else
{
    HandleInvalidInput();
}
```

**Зачем существует:** exceptions в C# — дорогие. Стектрейс собирается, происходит unwinding стека. Для невалидного пользовательского ввода (нормальная ситуация!) бросать exception — антипаттерн. `TryParse` есть у всех числовых типов, `DateTime`, `Guid`, `enum` и т.д.

### 9.5. try / catch — но только когда правда исключение

```csharp
try
{
    var data = await httpClient.GetAsync(url);  // может бросить HttpRequestException
    ProcessData(data);
}
catch (HttpRequestException ex) when (ex.StatusCode == HttpStatusCode.NotFound)
{
    // конкретный case с filter
    return null;
}
catch (HttpRequestException ex)
{
    logger.LogError(ex, "HTTP failed for {Url}", url);
    throw;          // re-throw, сохраняя стек
}
```

**Правила:**
- **Не лови `Exception` голым** — поймаешь всё, включая `OutOfMemoryException`.
- **Не лови ради «чтобы не падало»** — exception без обработки = баг скрытый, не починенный.
- **`when`-фильтры** — для уточнения condition без вложенных if.

Глубокий разбор exception handling (vs Result pattern, when использовать) — в [[error-handling|Error Handling]].

### 9.6. using — детерминированное освобождение ресурсов

**❌ Без `using` — стрим может не закрыться при exception:**

```csharp
StreamReader reader = new("file.txt");
string content = reader.ReadToEnd();
reader.Dispose();   // ← если ReadToEnd() бросит — этот Dispose не вызовется!
```

**✅ `using` statement — гарантия `Dispose()` даже при exception:**

```csharp
using (StreamReader reader = new("file.txt"))
{
    string content = reader.ReadToEnd();
}   // Dispose() вызывается здесь автоматически, даже при throw
```

**✅ `using` declaration (C# 8+) — короче, без вложенных скобок:**

```csharp
using StreamReader reader = new("file.txt");
string content = reader.ReadToEnd();
// Dispose() вызывается при выходе из текущего scope (метода/блока)
```

**✅ `await using` — для `IAsyncDisposable`:**

```csharp
await using var connection = new DbConnection(...);
await connection.OpenAsync();
// DisposeAsync() при выходе из scope
```

Если у типа есть `Dispose()` (`IDisposable`) — **обёрни в `using`**, иначе утекут handles, connections, file descriptors. GC однажды освободит, но «однажды» в production — это слишком поздно.

Глубже про `IDisposable` / `IAsyncDisposable` — в [[dispose-pattern|Dispose Pattern]].

---

## 10. Самые частые ошибки начинающих — с механизмами

Не «вот так нельзя», а **почему именно так получается**.

### 10.1. `==` для классов — сравнение ссылок, не значений

```csharp
class Person { public string Name; }
var p1 = new Person { Name = "Alice" };
var p2 = new Person { Name = "Alice" };
p1 == p2;   // false!
```

**Механизм:** `==` для reference type по умолчанию = `ReferenceEquals` = «один и тот же объект в памяти?». `p1` и `p2` — два разных объекта (две разные heap-аллокации). Содержимое идентично, но это разные объекты.

**Фикс:**
- Использовать `record` вместо `class` если нужна value-equality.
- Или переопределить `Equals` + `GetHashCode` + оператор `==`.

```csharp
record Person(string Name);
var p1 = new Person("Alice");
var p2 = new Person("Alice");
p1 == p2;   // true ✅
```

> **`string` — исключение:** для него оператор `==` переопределён на сравнение содержимого. Поэтому `"abc" == "abc"` это true. Многих сбивает с толку.

### 10.2. Integer division: `10 / 3 == 3`

**Механизм:** см. 4.1. Тип результата = тип операндов. `int / int` → `int`, дробь отбрасывается.

**Фикс:** один операнд должен быть дробным.

```csharp
double good1 = 10.0 / 3;          // ✅ 3.333...  один операнд double
double good2 = (double)10 / 3;     // ✅ 3.333...  явный cast одного операнда
double bad   = 10 / 3;             // ❌ 3.0       int/int = int, дробь отброшена,
                                   //              потом 3 → 3.0 при присваивании double
```

### 10.3. `0.1 + 0.2 != 0.3` для float/double

**Механизм:** см. 3.4. `double` хранит в двоичной системе с плавающей запятой; `0.1` десятичный = бесконечная двоичная дробь, неизбежное округление.

**Фикс:**
- Для денег — `decimal`.
- Для научного — сравнивай с epsilon: `Math.Abs(a - b) < 1e-9`.
- Для general purpose — переоценить нужна ли точность вообще или достаточно «приблизительно».

### 10.4. Null Reference Exception

**Механизм:** обращение к члену объекта, который равен null. JIT генерит проверку перед каждым `.` — если ссылка нулевая, бросает NRE.

**Фикс с NRT:**
1. Включи `<Nullable>enable</Nullable>` в `.csproj`.
2. Слушай warnings — обычно компилятор уже знает где null может быть.
3. Идиомы: `?.`, `??`, `is not null`, `ArgumentNullException.ThrowIfNull(x)`.

### 10.5. Изменение коллекции во время итерации

```csharp
List<int> numbers = [1, 2, 3, 4, 5];
foreach (var n in numbers)
{
    if (n % 2 == 0) numbers.Remove(n);   // 💥 InvalidOperationException
}
```

**Механизм:** `foreach` под капотом получает у коллекции `IEnumerator<T>`, который держит «версию» коллекции. При любом изменении версия инкрементится, и enumerator при следующем `MoveNext()` бросает «Collection was modified». Это защита от непредсказуемого поведения (что должно случиться если ты удалил элемент который ещё не дошёл до итерации?).

**Фикс — несколько вариантов:**

```csharp
// Вариант 1: создать новую коллекцию через LINQ
var filtered = numbers.Where(n => n % 2 != 0).ToList();

// Вариант 2: цикл в обратную сторону по индексу (без enumerator)
for (int i = numbers.Count - 1; i >= 0; i--)
{
    if (numbers[i] % 2 == 0) numbers.RemoveAt(i);
}

// Вариант 3: RemoveAll
numbers.RemoveAll(n => n % 2 == 0);
```

### 10.6. Off-by-one — выход за границы массива

```csharp
int[] arr = [1, 2, 3, 4, 5];
for (int i = 0; i <= arr.Length; i++)   // <= вместо <
{
    Console.WriteLine(arr[i]);            // 💥 IndexOutOfRangeException на arr[5]
}
```

**Механизм:** массив длины `n` имеет валидные индексы `0` … `n-1`. `<= arr.Length` означает `i == arr.Length`, что выходит за границу.

**Фикс:**
- `for (int i = 0; i < arr.Length; i++)` — стандартный шаблон.
- `foreach` — где можно, лучше: компилятор не ошибётся.
- `for-from-end` — `arr[^1]` для последнего, `arr[^2]` для предпоследнего.

### 10.7. Забытый `using` для disposable

```csharp
StreamReader reader = new("file.txt");
string content = reader.ReadToEnd();
// Забыли Dispose. Файл может остаться открытым до GC.
```

**Механизм:** native handles (file descriptors, sockets, mutexes) — это ограниченный ресурс ОС. GC освободит handle при сборке мусора, но GC может не запуститься часами в low-memory pressure. До этого — другие процессы не могут открыть файл, или у тебя кончатся handle'ы.

**Фикс:** `using` всегда. Современный синтаксис — `using` declaration (C# 8+):

```csharp
using StreamReader reader = new("file.txt");
string content = reader.ReadToEnd();
// Dispose() при выходе из текущего scope автоматически
```

### 10.8. Локаль ломает сериализацию

```csharp
double price = 1234.56;
string s = price.ToString();
// в US: "1234.56"
// в DE: "1234,56" — запятая вместо точки!
// в RU: "1234,56" тоже
```

**Механизм:** `ToString()` без аргументов использует `CultureInfo.CurrentCulture` — локаль ОС. Для UI это полезно (немец видит число в немецком формате). Для **сериализации** (CSV, JSON, log файлы, протоколы) — катастрофа: данные будут разные на разных машинах.

**Фикс:**

```csharp
using System.Globalization;

string s = price.ToString(CultureInfo.InvariantCulture);     // "1234.56" всегда
DateTime dt = DateTime.Parse(input, CultureInfo.InvariantCulture);

// Для display пользователю — CurrentCulture (default)
string display = price.ToString("C");   // "$1,234.56" или "1 234,56 ₽" по локали
```

**Правило:** для всего что **выходит за пределы текущей машины** (файлы, сеть, БД) — `InvariantCulture`. Для UI — `CurrentCulture`.

### 10.9. Magic numbers и magic strings

```csharp
if (status == 1) { /* ... */ }       // что значит 1?
if (elapsed > 86400) { /* ... */ }    // секунды? часы?
if (role == "admin") { /* ... */ }    // что если опечатался?
```

**Механизм:** компилятор не может проверить семантику числа или строки. Опечатка `"admn"` пройдёт компиляцию — упадёт в проде.

**Фикс:**

```csharp
// Enum для категориальных значений
enum OrderStatus { Pending = 1, Paid = 2, Shipped = 3 }
if (order.Status == OrderStatus.Paid) { /* ... */ }

// Const для magic чисел
private const int SecondsInDay = 86400;
if (elapsed > SecondsInDay) { /* ... */ }

// Лучше — TimeSpan
if (elapsed > TimeSpan.FromDays(1).TotalSeconds) { /* ... */ }

// Const для magic strings (или enum-конвертация)
private const string AdminRole = "admin";
```

### 10.10. Concatenation в цикле — медленно и memory-heavy

```csharp
string result = "";
for (int i = 0; i < 10_000; i++)
    result += i.ToString();      // 💥 каждая операция — новая строка
```

**Механизм:** `string` immutable. `result + "X"` создаёт **новый** string, копирует туда содержимое старого + "X". В цикле на 10к итераций — 10к аллокаций, 10к копирований всё больше и больше данных. O(n²) сложность.

**Фикс:**

```csharp
var sb = new StringBuilder();
for (int i = 0; i < 10_000; i++)
    sb.Append(i);
string result = sb.ToString();   // O(n)
```

`StringBuilder` хранит mutable буфер символов, удваивает его по мере необходимости. Для 2-5 склеек смысла нет (сам StringBuilder — отдельная аллокация). Для 10+ или цикла — обязательно.

---

## 11. Decision trees — куда смотреть когда сомневаешься

### 11.1. var или явный тип?

```
Начало
  │
  ▼
Тип очевиден из правой части?
  ├── Да → var
  │       (var users = new List<User>();)
  ▼
Тип НЕ очевиден / возвращает object/T?
  ├── Да → пиши явно
  │       (decimal total = CalculateTotal();)
  ▼
Длинное generic-имя или анонимный тип?
  └── Да → var обязателен или сильно желателен
          (var pair = (Name: "Alice", Age: 30);)
```

### 11.2. const, readonly, или static readonly?

```
Что я объявляю?
  │
  ├── Литерал, известный в момент компиляции, никогда не изменится → const
  │   (примитивы или string only)
  │
  ├── Значение, разное у разных экземпляров, не меняется после конструктора → readonly
  │   (public readonly int Port в каждом экземпляре свой)
  │
  └── Значение, общее для всех, вычисляется один раз → static readonly
      (public static readonly DateTime AppStart = DateTime.UtcNow;)
```

### 11.3. Поле или свойство?

```
Класс публичный или внутренний?
  │
  ├── Публичный (используется снаружи) → ВСЕГДА свойство
  │   (даже если сейчас просто хранение — завтра потребуется логика)
  │
  └── Internal/private, чисто внутреннее использование → можно поле
      (private int _counter; — не виден миру, проще)
```

Reflection / serializers / databinders / EF — почти все игнорят поля и работают только со свойствами. Если когда-нибудь твой класс попадёт в одну из этих систем — без свойств не обойдётся.

### 11.4. Primary constructor или классический?

```
Сколько работы в конструкторе?
  │
  ├── Только захват зависимостей (DI) или простое сохранение → primary
  │   (public class UserService(IRepo repo) { ... })
  │
  ├── Валидация, computed fields, сложная инициализация → классический
  │
  └── Несколько способов создать объект (overloading) → классический + factory
```

### 11.5. switch или if/else?

```
Сравнение с константами/типами/паттернами?
  │
  ├── Да, и каждая ветка возвращает значение → switch expression
  │   (var status = code switch { ... };)
  │
  ├── Да, но в каждой ветке side-effect (запись, вызов) → switch statement или if/else
  │
  └── Условия не «равно X», а сложные выражения (>5 && <10) → if/else
      (или switch с relational pattern, C# 9+)
```

---

## 12. Практика — упражнения с разбором

Решения здесь полные — читай после своей попытки.

### 12.1. FizzBuzz (классика собеседований)

**Задача:** напечатать числа 1-100. Если делится на 3 — `Fizz`. На 5 — `Buzz`. На оба — `FizzBuzz`.

```csharp
for (int i = 1; i <= 100; i++)
{
    string output = (i % 3 == 0, i % 5 == 0) switch
    {
        (true,  true)  => "FizzBuzz",
        (true,  false) => "Fizz",
        (false, true)  => "Buzz",
        (false, false) => i.ToString()
    };
    Console.WriteLine(output);
}
```

**Что демонстрируем:** switch expression с tuple pattern. На собеседовании это плюс — показывает что ты знаешь современный синтаксис.

### 12.2. Reverse string без LINQ

**Задача:** `"hello"` → `"olleh"`.

```csharp
public static string Reverse(string input)
{
    var chars = input.ToCharArray();
    Array.Reverse(chars);
    return new string(chars);
}
```

**С LINQ (короче, медленнее, чаще на интервью спросят без неё):**

```csharp
string reversed = new string(input.Reverse().ToArray());
```

**Под капотом:** `string` immutable, поэтому в любом случае создаётся новая строка. `Array.Reverse` модифицирует массив на месте.

### 12.3. Palindrome check

**Задача:** `"level"` → true, `"hello"` → false. Игнорируя регистр.

```csharp
public static bool IsPalindrome(string input)
{
    int left = 0;
    int right = input.Length - 1;

    while (left < right)
    {
        if (char.ToLowerInvariant(input[left]) !=
            char.ToLowerInvariant(input[right]))
        {
            return false;
        }
        left++;
        right--;
    }
    return true;
}
```

**Что важно:**
- `char.ToLowerInvariant` — не зависит от локали (`char.ToLower` использует CurrentCulture, на турецком `'I'.ToLower() == 'ı'`!).
- Two-pointer подход вместо `input == new string(input.Reverse().ToArray())` — O(n) без аллокации.

### 12.4. Word count в тексте

**Задача:** дан текст, вернуть Dictionary `слово → количество`.

```csharp
public static Dictionary<string, int> WordCount(string text)
{
    var counts = new Dictionary<string, int>(StringComparer.OrdinalIgnoreCase);
    var words = text.Split([' ', '\t', '\n', '.', ',', '!', '?'],
                           StringSplitOptions.RemoveEmptyEntries);

    foreach (var word in words)
    {
        if (counts.TryGetValue(word, out int count))
            counts[word] = count + 1;
        else
            counts[word] = 1;
    }

    return counts;
}
```

**Изящнее с LINQ:**

```csharp
public static Dictionary<string, int> WordCount(string text)
{
    return text
        .Split([' ', '\t', '\n', '.', ',', '!', '?'],
               StringSplitOptions.RemoveEmptyEntries)
        .GroupBy(w => w, StringComparer.OrdinalIgnoreCase)
        .ToDictionary(g => g.Key, g => g.Count(), StringComparer.OrdinalIgnoreCase);
}
```

**Что демонстрируем:** `StringComparer.OrdinalIgnoreCase` — case-insensitive ключи в словаре. `Split` с массивом разделителей. LINQ `GroupBy.ToDictionary`.

### 12.5. TODO list (мини-приложение)

```csharp
internal sealed class TodoList
{
    private readonly List<string> _items = [];

    public void Add(string item)
    {
        ArgumentException.ThrowIfNullOrWhiteSpace(item);
        _items.Add(item);
    }

    public bool Remove(int index)
    {
        if (index < 0 || index >= _items.Count) return false;
        _items.RemoveAt(index);
        return true;
    }

    public IReadOnlyList<string> Items => _items;
}

// Использование
var todo = new TodoList();
todo.Add("Read docs");
todo.Add("Write code");
todo.Add("Take a break");
todo.Remove(0);

foreach (var (item, idx) in todo.Items.Select((it, i) => (it, i)))
{
    Console.WriteLine($"{idx + 1}. {item}");
}
```

**Что важно:**
- `IReadOnlyList<string> Items => _items;` — наружу отдаём read-only view, мутировать снаружи нельзя.
- `ArgumentException.ThrowIfNullOrWhiteSpace` — современный guard clause (.NET 8+).
- `sealed class` — не предполагается наследование.

---

## 13. Cheat sheet

### Объявления

```csharp
int x = 5;                              // value type, на стеке
string s = "hello";                     // reference type, на heap
var users = new List<User>();           // var когда тип очевиден
const int Max = 100;                    // запекается в callsite
private static readonly DateTime Start = DateTime.UtcNow;
```

### Null

```csharp
string? maybe = null;                   // явно nullable
string sure = maybe ?? "default";       // null coalescing
int? len = maybe?.Length;               // null conditional
ArgumentNullException.ThrowIfNull(arg); // guard clause
```

### Управление потоком

```csharp
var status = code switch
{
    "A" => "Active",
    "I" => "Inactive",
    _   => "Unknown"
};

if (user is { Age: >= 18, IsActive: true } adult) { /* ... */ }

foreach (var (key, value) in dictionary) { /* ... */ }
```

### Классы

```csharp
public record User(string Name, int Age);              // DTO

public class Service(IRepo repo)                       // primary ctor + DI
{
    public async Task<User?> GetByIdAsync(Guid id)
        => await repo.GetByIdAsync(id);
}

public class Order
{
    public required Guid Id { get; init; }              // обязательно при создании
    public DateTime CreatedAt { get; } = DateTime.UtcNow; // только в declaration/ctor
}
```

### Идиомы

```csharp
// TryParse без exception
if (int.TryParse(input, out int n)) { Use(n); }

// using для disposable
using var stream = File.OpenRead(path);

// String interpolation
var greeting = $"Hello, {name.Trim()}!";

// Collection expression
int[] arr = [1, 2, 3];
List<string> list = ["a", "b"];

// Tuple deconstruction
var (name, age) = ("Alice", 30);
```

---

## 14. Что читать дальше — порядок и почему

После basics остальная Junior-программа в [[CSharp/README|CSharp/README]] выглядит так. Я даю не просто список, а **зачем** каждый файл идёт в этом порядке:

1. **[[dotnet-cli-getting-started|dotnet CLI]]** — чтобы наконец-то понять что делает каждая команда, которую ты копировал из туториалов. Открывает дверь к проектам, NuGet, тестам, publish.
2. **[[debugging-basics|Debugging]]** — учиться искать баги через debugger, а не через `Console.WriteLine`. Сразу же. Иначе привычка `printf-debug` намертво въестся.
3. **[[naming-conventions|Naming Conventions]]** — пока кода мало, легко переучиться. С привычкой называть `data`, `temp`, `MyClass` потом тяжелее.
4. **[[oop|OOP и классы]]** — сюда я отдал всё про классы кроме самых basics. Inheritance, interfaces, abstract, sealed, polymorphism, full records vs class разбор.
5. **[[collections-linq|Collections и LINQ]]** — без этого ты не напишешь ни одного реального метода. List, Dictionary, HashSet, и LINQ который заменяет половину циклов.
6. **[[error-handling|Error Handling]]** — exceptions, Result pattern, когда что. Plus строки и I/O в комплекте.
7. **[[iterators-yield|Iterators / yield]]** — `yield return` и lazy enumeration. Один из самых элегантных языковых механизмов.
8. **[[enums-flags|Enums and Flags]]** — `[Flags]`, parsing, сериализация. Часто встречается в API.
9. **[[strings-regex|Strings и Regex]]** — глубокий разбор строковых операций, performance, regex. Подкреплённый `Span<T>` для hot path.
10. **[[nullable-types|Nullable Types]]** — после basics nullable — глубже про NRT, attributes, паттерны.
11. **[[dispose-pattern|Dispose Pattern]]** — `IDisposable`, `IAsyncDisposable`, `SafeHandle`. Почему GC.SuppressFinalize. Без этого нельзя писать классы с unmanaged ресурсами.

После этих — middle уровень: [[async-threading|async/await]], [[types-and-memory|types and memory]] (теперь действительно глубоко), [[delegates-events|delegates/events]], [[modern-features|modern features]].

Полный roadmap — [[02_junior-to-middle|Junior → Middle Roadmap]].

---

## 15. Best practices даже как Junior

- **`var` для очевидного** (`new List<User>()`), **явный тип** для важного (`decimal price = ...`).
- **`decimal` для денег**, `double` для всего остального дробного. Никогда `float`/`double` для денег.
- **`int` для целых** по умолчанию. `long` — только если правда нужно (timestamps, big IDs).
- **String interpolation `$""`** > `String.Format` > конкатенация.
- **`StringBuilder`** в цикле от 10+ итераций.
- **`using`** для всего IDisposable. Современный синтаксис без `{ }`.
- **Свойства**, не публичные поля.
- **`record`** для DTO/value object, **`class`** для сервисов/entities.
- **`sealed`** для классов которые не предполагают наследование (большинство).
- **Switch expression** > старый switch, `if/else` тоже когда уместно.
- **`TryParse`** > `Parse` для пользовательского ввода.
- **`InvariantCulture`** для сериализации, `CurrentCulture` для UI.
- **camelCase** для локальных, **PascalCase** для публичного, **`_camelCase`** для private fields.
- **`{ }`** даже для одного оператора. Apple goto fail.
- **Nullable Reference Types** включи и слушай warnings.
- **Guard clauses** (`ArgumentNullException.ThrowIfNull`) в начале метода.
- **Не over-engineer.** Простое решение лучше «умного». Premature optimization is the root of all evil.

---

## 16. См. также

**Сразу после basics:**
- [[dotnet-cli-getting-started|dotnet CLI]]
- [[debugging-basics|Debugging Basics]]
- [[naming-conventions|Naming Conventions]]

**Углубление основных тем этого файла:**
- [[oop|OOP и классы]] — всё про классы и наследование
- [[types-and-memory|Types and Memory]] — value vs reference, boxing, struct internals
- [[modern-features|Modern Features]] — pattern matching, records, primary ctors, raw strings
- [[collections-linq|Collections и LINQ]]
- [[error-handling|Error Handling]]
- [[nullable-types|Nullable Types]]
- [[strings-regex|Strings и Regex]]
- [[datetime-timezones|DateTime и Timezones]]
- [[keywords-reference|Keywords Reference]] — ref/in/out/scoped/required/init/file/checked

**Смежные темы:**
- [[csharp-vs-other-langs|C# vs Java/Go/Rust/Python]] — для тех кто переходит из другого языка
- [[csharp-language-design|Language Design]] — почему C# именно такой

**Roadmap:**
- [[02_junior-to-middle|Junior → Middle Roadmap]]

---

## 17. Reading list

**Книги:**
- **C# in Depth** (Jon Skeet) — после basics. Глубоко в язык.
- **Pro C# 10 / .NET 6** (Andrew Troelsen) — большой справочник, особенно хорош как reference.
- **C# Yellow Book** (Rob Miles) — бесплатный PDF, классика для начинающих.

**Документация:**
- **Microsoft Docs — C# Programming Guide** — `learn.microsoft.com/dotnet/csharp` — Single source of truth.
- **Microsoft Learn — Take your first steps with C#** — бесплатный курс с практикой.

**Блоги и видео:**
- **Stephen Toub** — `devblogs.microsoft.com/dotnet/author/toub` — глубокие статьи о .NET internals.
- **Nick Chapsas** — YouTube, рассказывает современные практики.
- **Jeremy Sinclair** / **Jon Skeet on Stack Overflow** — отдельные жемчужины ответов на конкретные вопросы.

**Практика:**
- **leetcode.com** — алгоритмические задачи, фильтр Easy + C#.
- **codewars.com** — кат-прогрессия, можно увидеть чужие решения.
- **exercism.io** — задачи с менторингом, бесплатно.
