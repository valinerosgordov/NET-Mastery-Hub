# Типы и память

## readonly vs const

**const** — значение определяется при компиляции, подставляется inline (как литерал). Только примитивы, enum, string. При изменении значения нужна перекомпиляция всех зависимых сборок.

**readonly** — значение определяется в runtime (конструктор или инициализатор). Любой тип. Может быть instance или static.

```csharp
public class Config
{
    public const int MaxRetries = 3;            // inline при компиляции
    public static readonly TimeSpan Timeout =   // runtime, любой тип
        TimeSpan.FromSeconds(30);

    public readonly string Name;                // instance, задаётся в конструкторе
    public Config(string name) => Name = name;
}
```

**Нюанс:** `const` в публичном API — опасно. Если библиотека меняет значение, потребители не увидят изменение без перекомпиляции. Для публичных значений предпочитать `static readonly`.

**init** (C# 9) — свойство, которое можно задать только при инициализации объекта:

```csharp
public record Order
{
    public required string Name { get; init; }  // required + init
}
```

---

## class, record, struct

| Аспект | class | record | struct | record struct |
|--------|-------|--------|--------|---------------|
| Тип | Reference | Reference | Value | Value |
| Равенство | По ссылке | По значению | По значению | По значению |
| Наследование | Да | Да | Нет | Нет |
| Мутабельность | По умолчанию | Immutable (init) | Мутабельный | Immutable (init) |
| Назначение | Объекты с идентичностью | DTO, immutable данные | Маленькие значения (<16 bytes) | Immutable маленькие значения |

```csharp
// record — автогенерация Equals, GetHashCode, ToString, with-выражений
public record OrderDto(Guid Id, string Customer, decimal Total);

// with — создание копии с изменёнными полями
var updated = order with { Total = 200m };

// record struct — value type с семантикой record
public readonly record struct Point(double X, double Y);
```

**Нюанс:** `record class` (default) — reference type. `record struct` — value type. Primary constructor параметры в record — это свойства (init). В class primary constructor — это поля (captured parameters), не свойства.

---

## ref struct

Только на стеке. Не может быть полем класса, боксироваться, использоваться в async/await или замыканиях.

```csharp
// Span<T> — ref struct, работа с памятью без аллокаций
ReadOnlySpan<char> span = "Hello, World!".AsSpan();
ReadOnlySpan<char> hello = span[..5];  // без аллокации новой строки

// Парсинг без аллокаций
public static int ParseFast(ReadOnlySpan<char> input)
    => int.Parse(input);

// stackalloc + Span
Span<byte> buffer = stackalloc byte[256];
```

**Нюанс:** начиная с C# 13 ref struct может реализовывать интерфейсы (например, `IDisposable`). В .NET 9+ `allows ref struct` constraint для generic.

---

## Heap и Stack

**Stack** — LIFO, автоматическое управление. Хранит: локальные value types, параметры, адреса возврата. Быстрый, ограниченный размер (~1 MB).

**Heap** — managed heap, управляется GC. Хранит: объекты (reference types), boxed value types. Медленнее выделение, но неограниченный размер.

```
Метод вызван:
┌──────────────┐     ┌──────────────────┐
│   Stack      │     │     Heap         │
│ int x = 42   │     │ new Order() ──┐  │
│ ref → ───────┼─────┤              │  │
│ decimal d    │     │  {Id, Total}  │  │
└──────────────┘     └──────────────────┘
```

**Нюанс:** даже value type может оказаться на heap — если он поле класса, boxed, или captured в замыкании. Stack vs Heap — не строго по типу, а по контексту использования.

---

## Boxing, string, StringBuilder

### Boxing / Unboxing

Value type → object: создаётся копия на heap (аллокация + копирование). Обратно — unboxing (проверка типа + копирование).

```csharp
int x = 42;
object boxed = x;       // boxing — аллокация на heap
int y = (int)boxed;     // unboxing — проверка типа + копия

// Как избежать boxing:
// 1. Generic коллекции: List<int> вместо ArrayList
// 2. IEquatable<T> — struct не боксится при сравнении
// 3. Span<T> — обработка без аллокаций
```

### string

Immutable, reference type. Каждая модификация создаёт новый объект. String interning — одинаковые литералы указывают на один объект.

```csharp
string a = "hello";
string b = "hello";
Console.WriteLine(ReferenceEquals(a, b)); // true — interning

// Сравнение
"abc".Equals("ABC", StringComparison.OrdinalIgnoreCase); // true
string.Compare(a, b, StringComparison.Ordinal);           // без culture
```

### StringBuilder

Мутабельный буфер. Используть при 3+ конкатенациях в цикле.

```csharp
var sb = new StringBuilder(256); // указать capacity для избежания reallocations
for (int i = 0; i < 1000; i++)
    sb.Append($"Item {i}, ");
```

**Нюанс:** для 2-3 конкатенаций вне цикла обычный `+` или интерполяция быстрее (компилятор оптимизирует через `string.Concat` или `DefaultInterpolatedStringHandler` в C# 10+).

---

## GC (Garbage Collector)

Поколения: Gen0 (часто, быстро), Gen1 (промежуточный), Gen2 (редко, дорого). LOH (Large Object Heap) — объекты > 85 KB, собирается с Gen2.

```csharp
// Проверка давления GC
GC.CollectionCount(0); // сколько сборок Gen0
GC.GetTotalMemory(false); // приблизительный размер heap

// Подсказка GC (редко нужно)
GC.Collect(2, GCCollectionMode.Aggressive, blocking: true);
```

**Нюанс:** Workstation GC — минимальные паузы (UI). Server GC — максимальная пропускная способность (ASP.NET). Настройка в csproj: `<ServerGarbageCollection>true</ServerGarbageCollection>`.

---

## См. также

- [C# Reference: Типы и основы](../../../Reference/csharp-types-and-basics.md)
- [.NET Performance](../../Performance/dotnet-performance.md)
