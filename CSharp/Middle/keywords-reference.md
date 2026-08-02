---
tags: [csharp, keywords, reference, middle, language-features]
level: Middle
date: 2026-08-02
---

# C# Keywords Reference — справочник по ключевым словам

> **Все 100+ keywords и contextual keywords языка с группировкой по назначению.** От `abstract` до `yield`, modifiers, types, control flow, modern (`record`, `init`, `with`). Закрывает пробел: «слышал про `volatile` и `unsafe`, не помню что они делают».

---

## 0. Как читать

Этот файл — справочник, **не** туториал. Используй как reference: ищи keyword по группе или alphabetically. Каждое слово — описание + минимальный пример + ссылка на deep notes если есть.

Группы: types и declarations (1-2), modifiers (3), control flow (4), exception handling (5), namespaces / using (6), values / literals (7), pattern matching (8), async / threading (9), unsafe / interop (10), modern features (11), contextual keywords (12).

---

## 1. Что это, зачем и когда

### 1.1. Reserved keywords vs contextual

C# имеет **~80 reserved keywords** (нельзя использовать как identifier) и **~40 contextual keywords** (special meaning только в context, иначе normal identifier).

```csharp
// Reserved — нельзя
int int = 5;          // ❌ compile error
string class = "x";   // ❌

// Contextual — можно (но плохой стиль)
int async = 5;        // OK — async тут identifier
var (yield, value) = (1, 2);   // yield — identifier (старый код)
```

`@` префикс позволяет использовать reserved как identifier:
```csharp
int @int = 5;   // OK — escape
```

### 1.2. Эволюция

Каждая C# version добавляет keywords. До 2026:
- C# 1.0 — base set (~50)
- C# 2.0 — `partial`, `yield`, generics
- C# 3.0 — `var`, `from`/`select`/`where` (LINQ)
- C# 4.0 — `dynamic`
- C# 5.0 — `async`/`await`
- C# 6.0 — `nameof`, `when`
- C# 7-8 — `is` patterns, `record` (preview), `notnull`
- C# 9 — `record`, `init`, `with`
- C# 11 — `required`, `file`, `scoped`
- C# 12 — primary constructors syntax (no new keyword)
- C# 13 — `params` collections, `allows ref struct`, `field` (preview)
- C# 14 — `extension`, `field` (stable)

> [!question]- Интервью: чем reserved keyword отличается от contextual?
> **Reserved** — слово, которое нельзя использовать как identifier (`int`, `class`, `void`). Compiler error если попытаться. **Contextual** — special meaning **только в специфическом context** (`async`, `var`, `yield`, `where`, `record`, `init`). Можно использовать как identifier вне context, но плохой стиль. `@` префикс позволяет escape: `int @int = 5;` OK. Новые keywords обычно contextual (для backward compat). Список в C# language reference (~80 reserved + ~40 contextual в 2026).

---

## 2. Type declarations

### 2.1. class

```csharp
public class Person { }
public class Container<T> { }
```

Reference type. Heap allocation, value/reference semantics — reference. См. [[oop|OOP]].

### 2.2. struct

```csharp
public struct Point { public int X, Y; }
public readonly struct Money { /* ... */ }
public ref struct Span<T> { /* ... */ }
```

Value type. Stack allocation для locals, inline в classes. `readonly struct` — immutable.

### 2.3. interface

```csharp
public interface IRepository<T>
{
    T? GetById(int id);
}
```

Contract без implementation (или с default — C# 8+). Multiple inheritance.

### 2.4. enum

```csharp
public enum Color { Red, Green, Blue }
[Flags] public enum Permissions { Read = 1, Write = 2 }
```

Named integer values. См. [[enums-flags]].

### 2.5. delegate

```csharp
public delegate void EventHandler(object sender, EventArgs e);
public delegate TResult Func<in T, out TResult>(T arg);
```

Function pointer type. См. [[delegates-events]].

### 2.6. record (C# 9+)

```csharp
public record User(int Id, string Name);
public record class User2(int Id);
public record struct Point(int X, int Y);   // C# 10+
```

Auto-generated equality, ToString, Deconstruct, with-expression. См. [[oop|OOP]] раздел 11.

### 2.7. namespace

```csharp
namespace MyApp.Domain;   // file-scoped (C# 10+)

namespace MyApp.Domain
{
    public class Order { }
}
```

Logical grouping. File-scoped reduces indentation.

---

## 3. Access modifiers

### 3.1. public

Доступ везде.

```csharp
public class Foo { }
```

### 3.2. internal

Доступ только в текущем assembly. Default для top-level types.

```csharp
internal class Helper { }   // = только в этом проекте
```

### 3.3. protected

Доступ в классе и derived классах.

```csharp
public class Base
{
    protected int _value;
}
```

### 3.4. private

Доступ только в текущем классе. Default для members.

```csharp
public class Foo
{
    private int _state;
}
```

### 3.5. protected internal

Union — в этом assembly OR derived classes (даже из других assembly).

```csharp
public class Foo
{
    protected internal int X;   // in-assembly OR inherited
}
```

### 3.6. private protected (C# 7.2+)

Intersection — derived classes **в том же assembly**.

```csharp
public class Foo
{
    private protected int X;   // derived AND in-assembly
}
```

### 3.7. file (C# 11+)

Доступ только в этом файле.

```csharp
file class FileLocalHelper { }
```

Source generator-friendly — generated types не collide.

> [!question]- Интервью: чем `protected internal` отличается от `private protected`?
> **`protected internal`** — **OR** (union): доступно в том же assembly **или** в derived classes (даже из другого assembly). **`private protected`** (C# 7.2+) — **AND** (intersection): derived class **и** в том же assembly. `private protected` более restrictive. Примерные use cases: `protected internal` — расширяемые библиотеки (внешние derived могут access). `private protected` — internal hierarchy без external access.

---

## 4. Member modifiers

### 4.1. static

```csharp
public static int Count;
public static void Method() { }
public static class MathHelper { }
```

Принадлежит классу, не instance. Static class — все members static.

### 4.2. abstract

```csharp
public abstract class Shape
{
    public abstract double Area();   // no implementation
}
```

Не instantiable. Abstract method — derived обязан override.

### 4.3. virtual / override / new

```csharp
public class Base { public virtual void M() { } }
public class Derived : Base { public override void M() { } }
public class HiddenDerived : Base { public new void M() { } }   // hides, не overrides
```

`virtual` — может быть overridden. `override` — overrides. `new` — hides (anti-pattern). См. [[oop|OOP]] раздел 5-6.

### 4.4. sealed

```csharp
public sealed class Money { }            // нет inheritance
public sealed override void M() { }       // override + блокирует further override
```

См. [[oop|OOP]] раздел 10.

### 4.5. readonly

```csharp
public class Foo
{
    private readonly int _value;   // присваивается в constructor или declaration
    public readonly struct Money { }  // immutable struct
    
    public readonly int Compute() { return _value; }   // method — guarantees no mutation (struct only)
}
```

### 4.6. const

```csharp
public const int MaxRetries = 3;
public const string Greeting = "Hello";
```

Compile-time constant. Embedded в caller assemblies (опасно для public API breaking changes).

### 4.7. partial

```csharp
public partial class MyClass { /* part 1 */ }
public partial class MyClass { /* part 2 */ }   // в другом файле

public partial void MyMethod();   // declaration
private partial void MyMethod() { /* impl */ }   // implementation (C# 9+)
```

Split class / method между файлами. Used by source generators, designers.

### 4.8. async / await

```csharp
public async Task<int> GetCountAsync()
{
    var data = await LoadAsync();
    return data.Count;
}
```

Asynchronous method. См. async/await deep notes.

### 4.9. extern

```csharp
[DllImport("user32.dll")]
public static extern int MessageBox(IntPtr hWnd, string text, string caption, int type);
```

Method implementation extern (P/Invoke, COM interop).

### 4.10. unsafe

```csharp
public unsafe int* GetPointer() { /* ... */ }

unsafe
{
    int* p = stackalloc int[10];
}
```

Pointer arithmetic. Требует `<AllowUnsafeBlocks>true</AllowUnsafeBlocks>`.

### 4.11. volatile

```csharp
public class Counter
{
    private volatile int _value;   // memory barrier на reads/writes
}
```

Forces compiler/JIT не reorder reads/writes. **Используй `Interlocked` или `lock`** обычно — volatile mostly для micro-optimizations.

### 4.12. unsafe + fixed

```csharp
unsafe
{
    fixed (byte* p = &array[0])   // pin GC от moving
    {
        // direct memory access
    }
}
```

`fixed` — pin managed memory чтобы pointer оставался valid. Для interop / unsafe code.

> [!question]- Интервью: чем `readonly` отличается от `const`?
> **`const`** — compile-time constant. Value embedded в caller assembly. Тип: только primitives, string, null. Изменение в библиотеке = breaking change для callers (нужна recompilation). **`readonly`** — runtime constant. Initialized в constructor или declaration. Любой тип. Изменение реализации не breaks callers. Best practice: `static readonly` для public consts (более safe), `const` только для primitives которые точно не изменятся (`Math.PI`).

### 4.13. Parameter modifiers: ref / out / in — mental model

By default parameters передаются **by value**: метод получает копию аргумента (для value type — копию значения, для reference type — копию ссылки). `ref`, `out`, `in` меняют это: метод получает не копию, а **storage location** (адрес самой переменной caller'а). Сам тип при этом остаётся каким был — value type не превращается в reference type; ты просто передаёшь ссылку на ячейку, где он лежит.

```csharp
void ByValue(int x) => x = 99;       // менят копию, caller не видит
void ByRef(ref int x) => x = 99;     // менят саму переменную caller'а

int a = 1;
ByValue(a);   // a == 1
ByRef(ref a); // a == 99
```

Ключевая ментальная модель: `ref`/`out`/`in` передают **местоположение** (location), а не значение. Через эту location метод читает и пишет ту же ячейку памяти, что и caller. Категория типа (`value` vs `reference`) от этого не меняется — `int` остаётся value type, просто доступ к нему идёт по адресу.

### 4.14. ref — переменная должна быть инициализирована ДО вызова

`ref` — двусторонняя связь: метод и читает, и пишет переменную caller'а. Поэтому compiler требует, чтобы переменная была **инициализирована до вызова** (метод вправе её прочитать первым делом).

```csharp
void Increment(ref int x) => x++;   // читает x, затем пишет

int n = 5;        // обязательно инициализировать
Increment(ref n); // n == 6

int m;
Increment(ref m); // ❌ compile error: use of unassigned local variable 'm'
```

`ref` нужен в declaration **и** в call site (`Increment(ref n)`) — явность на стороне вызова показывает читателю, что переменная может измениться.

### 4.15. out — обязан быть присвоен ВНУТРИ метода

`out` — односторонняя связь наружу: метод **обязан присвоить** значение перед любым нормальным `return` (compiler это проверяет). Поэтому caller'у инициализировать переменную не нужно — её прежнее значение всё равно игнорируется.

```csharp
bool TryParse(string s, out int result)
{
    if (int.TryParse(s, out int parsed))
    {
        result = parsed;   // обязаны присвоить
        return true;
    }
    result = 0;            // и на этой ветке тоже — иначе compile error
    return false;
}

if (TryParse("42", out int value))   // out var — объявление прямо в вызове
    Console.WriteLine(value);        // 42
```

Классический паттерн — `Try*` методы: bool-результат + `out` значение. Внутри метода каждый путь выполнения обязан присвоить `out`-параметр.

### 4.16. in — readonly by-reference (передаём location, но менять нельзя)

`in` передаёт переменную **by reference**, но **read-only** — метод не может её изменить. Смысл — избежать копирования большого `struct` при передаче, сохранив immutability. Для маленьких типов (`int`, `long`) выигрыша нет, только лишний indirection.

```csharp
readonly struct BigValue   // readonly struct — ключевой момент
{
    public readonly long A, B, C, D;
    public long Sum() => A + B + C + D;
}

long Process(in BigValue v)   // передаётся по ссылке, без копии 32 байт
{
    // v.A = 1;   // ❌ compile error — in-параметр read-only
    return v.Sum();
}
```

> [!warning] Defensive copy на не-readonly struct
> Если struct **не** `readonly`, compiler не может гарантировать, что вызванный метод (например `v.Sum()`) не мутирует `v`. Чтобы защитить read-only контракт `in`, он создаёт **скрытую защитную копию** на каждом обращении к члену — и весь выигрыш `in` исчезает (становится даже хуже обычной передачи by value). Правило: `in` имеет смысл **только** для `readonly struct`, где defensive copy не нужна.

`in` на call site опционален (`Process(in v)` или `Process(v)`), но явный `in` документирует намерение. Senior-уровень (`ref returns`, `ref locals`, `ref readonly` для возврата ссылок и алиасов памяти) разобран в [[types-and-memory|Types и Memory]] раздел 4.5–4.7.

> [!question]- Интервью: в чём разница ref / out / in?
> Все три передают **storage location** (адрес переменной), а не копию — но сам тип остаётся value type, категория не меняется. **`ref`** — двусторонняя: переменная должна быть инициализирована **до** вызова (метод может прочитать), метод может читать и писать. **`out`** — наружу: caller инициализировать не обязан, метод **обязан присвоить** значение на каждом пути до `return` (паттерн `Try*`). **`in`** — внутрь read-only: передача by-reference без права мутации, нужна чтобы не копировать большой `readonly struct`. ⚠️ `in` на **не-`readonly`** struct провоцирует defensive copy при каждом обращении к члену — выигрыш теряется. `ref`/`out` требуют keyword и в declaration, и на call site; у `in` call-site keyword опционален.

---

## 5. Control flow

### 5.1. if / else

```csharp
if (condition) { }
else if (other) { }
else { }
```

### 5.2. switch (statement)

```csharp
switch (value)
{
    case 1: Console.WriteLine("one"); break;
    case 2:
    case 3: Console.WriteLine("two or three"); break;
    default: throw new ArgumentException();
}
```

### 5.3. switch expression (C# 8+)

```csharp
var result = value switch
{
    1 => "one",
    2 or 3 => "two or three",
    _ => throw new ArgumentException()
};
```

### 5.4. for / while / do-while

```csharp
for (int i = 0; i < 10; i++) { }

while (condition) { }

do { } while (condition);
```

### 5.5. foreach

```csharp
foreach (var item in collection)
{
    Process(item);
}

await foreach (var item in asyncStream)   // C# 8+
{
    Process(item);
}
```

### 5.6. break / continue

```csharp
for (int i = 0; i < 100; i++)
{
    if (i > 50) break;        // exit loop
    if (i % 2 == 0) continue;  // next iteration
    Process(i);
}
```

### 5.7. return / yield return

```csharp
public int Get() => 42;

public IEnumerable<int> Generate()
{
    yield return 1;
    yield return 2;
    yield break;   // end iteration
}
```

См. [[iterators-yield]].

### 5.8. goto

```csharp
for (int i = 0; i < 10; i++)
{
    if (condition) goto Done;
}
Done:
// ...
```

Используй редко. В switch — `goto case 1;` для fall-through.

> [!question]- Интервью: что делает `yield return`?
> Используется в methods, returning `IEnumerable<T>` или `IAsyncEnumerable<T>`. Compiler генерирует **state machine** — каждый `yield return X` pauses метод, returns X caller'у; следующий `MoveNext()` продолжает с того места. Lazy evaluation — values produced by demand. `yield break` — early termination. См. [[iterators-yield]] для deep dive. Used in LINQ implementations, custom iterators, async streams.

---

## 6. Exception handling

### 6.1. try / catch / finally

```csharp
try
{
    DoSomething();
}
catch (SpecificException ex) when (ex.Code == 42)   // filter (C# 6+)
{
    Handle(ex);
}
catch (Exception ex)
{
    LogAndRethrow(ex);
    throw;   // rethrow preserves stack
}
finally
{
    Cleanup();
}
```

См. [[error-handling]].

### 6.2. throw

```csharp
throw new ArgumentException(nameof(arg));
throw;   // rethrow в catch — preserves stack trace
throw new Exception("...", innerException);   // wrap

// throw expression (C# 7+)
var x = obj ?? throw new ArgumentNullException(nameof(obj));
```

### 6.3. checked / unchecked

```csharp
checked
{
    int x = int.MaxValue + 1;   // throws OverflowException
}

unchecked
{
    int y = int.MaxValue + 1;   // wraps to int.MinValue
}

// На уровне expression
int a = checked(b + c);
int d = unchecked(e + f);
```

By default — unchecked для performance. Compiler option `<CheckForOverflowUnderflow>` влияет.

---

## 7. Namespaces / using

### 7.1. using directive

```csharp
using System;
using System.Collections.Generic;
using static System.Math;   // import static members
using StringList = System.Collections.Generic.List<string>;   // alias
global using System;   // C# 10+ — applies to all files
```

### 7.2. using statement / declaration

```csharp
using (var resource = new Resource()) { }   // statement
using var resource2 = new Resource();         // declaration C# 8+

await using var asyncRes = new AsyncResource();   // C# 8+
```

См. [[dispose-pattern]] раздел 2.

---

## 8. Values / literals

### 8.1. true / false

Boolean literals.

### 8.2. null

Null reference.

### 8.3. default

```csharp
int x = default;        // 0
string s = default;     // null
T value = default(T);   // тип-specific default

int y = default(int);    // explicit
```

### 8.4. this / base

```csharp
public class Derived : Base
{
    public Derived(int x) : base(x) { }
    
    public void Method()
    {
        this.Field = 5;   // current instance
        base.Method();     // base implementation
    }
}
```

### 8.5. typeof

```csharp
Type t = typeof(string);          // compile-time
Type t2 = typeof(List<int>);
```

vs `obj.GetType()` (runtime).

### 8.6. nameof (C# 6+)

```csharp
throw new ArgumentNullException(nameof(arg));   // string "arg"
Console.WriteLine(nameof(MyClass.Property));    // "Property"
```

Compile-time evaluated. Refactoring-friendly (rename работает).

### 8.7. sizeof

```csharp
int size = sizeof(int);       // 4
int s = sizeof(MyStruct);     // unsafe для non-primitive
```

---

## 9. Pattern matching и type checks

### 9.1. is

```csharp
if (obj is string s) Console.WriteLine(s.Length);   // type pattern
if (obj is null) { }                                 // null pattern
if (obj is { Property: > 5 }) { }                    // property pattern (C# 8+)
if (obj is [1, 2, _]) { }                             // list pattern (C# 11+)
```

### 9.2. as

```csharp
string? s = obj as string;   // null если не string (no exception)
```

### 9.3. switch с patterns

```csharp
var description = shape switch
{
    Circle { Radius: > 100 } => "big circle",
    Circle c => $"circle r={c.Radius}",
    Square { Side: var s } => $"square {s}",
    null => "nothing",
    _ => "other"
};
```

### 9.4. when (filter)

```csharp
try { }
catch (HttpException ex) when (ex.StatusCode == 404) { }

var x = value switch
{
    int n when n > 0 => "positive",
    _ => "non-positive"
};
```

---

## 10. Async / threading

### 10.1. async / await

См. async deep notes (отдельный файл).

### 10.2. lock

```csharp
private readonly object _lock = new();

lock (_lock)
{
    // critical section
}
```

С .NET 9+ — `Lock` type optimized.

### 10.3. fixed

```csharp
unsafe
{
    fixed (byte* p = &array[0])
    {
        // GC won't move array while in fixed scope
    }
}
```

---

## 11. Modern features (C# 9+)

### 11.1. record (C# 9+)

См. раздел 2.6.

### 11.2. init

```csharp
public class User
{
    public string Name { get; init; } = "";   // set только в constructor / object init
}
```

### 11.3. with (C# 9+)

```csharp
var u1 = new User { Name = "Alice" };
var u2 = u1 with { Name = "Bob" };   // copy with change
```

Только для records и structs.

### 11.4. required (C# 11+)

```csharp
public class User
{
    public required string Email { get; init; }
}
```

Compile-time guarantee — caller обязан задать.

### 11.5. file (C# 11+)

```csharp
file class HiddenInThisFile { }
```

Доступ только в этом файле.

### 11.6. global using (C# 10+)

```csharp
// In any file
global using System;
global using System.Threading.Tasks;
```

Applies to all files в project.

### 11.7. raw string literals (C# 11+)

```csharp
string json = """
    {
        "name": "Alice"
    }
    """;
```

### 11.8. field (C# 14; preview в C# 13)

```csharp
public string Name
{
    get;
    set => field = value?.Trim() ?? "";   // 'field' contextual keyword
}
```

Без явного `_name` backing field. Стабилен с **C# 14** (.NET 10); в C# 13 был доступен только с `<LangVersion>preview</LangVersion>`.

> [!question]- Интервью: что делает `init` accessor?
> Variant `set`, доступный **только в constructor** или **object initializer**. После construction — read-only. Использует setter syntax: `public string Name { get; init; }`. Caller: `new User { Name = "Alice" }` OK; `user.Name = "Bob"` — compile error. С C# 9+. Удобный для immutable types без full constructor — flexible initialization + immutable after.

---

## 12. Contextual keywords (selected)

### 12.1. var

```csharp
var x = 42;        // int — type inferred
var s = "hello";   // string
```

### 12.2. dynamic

```csharp
dynamic d = GetDynamic();
d.AnyMethod();   // resolved at runtime, может throw RuntimeBinderException
```

DLR (Dynamic Language Runtime). Slow, type-checking off.

### 12.3. yield

См. раздел 5.7.

### 12.4. nameof, when, where, from, select, group, into, orderby, ascending, descending, on, equals, by, let, join

LINQ keywords (query syntax). Контекстные — only within LINQ.

### 12.5. async, await

See раздел 4.8.

### 12.6. notnull

```csharp
public class MyClass<T> where T : notnull   // generic constraint
{
}
```

### 12.7. partial

См. раздел 4.7.

### 12.8. record

См. раздел 2.6.

### 12.9. init

См. раздел 11.2.

### 12.10. with

См. раздел 11.3.

### 12.11. file

См. раздел 11.5.

### 12.12. required

См. раздел 11.4.

### 12.13. value

В property setter — implicit parameter:
```csharp
public string Name
{
    set { _name = value; }
}
```

### 12.14. add / remove

В event accessor:
```csharp
public event EventHandler MyEvent
{
    add { /* custom */ }
    remove { /* custom */ }
}
```

### 12.15. get / set

Property accessors.

### 12.16. scoped (C# 11+)

Ограничивает lifetime ref/`ref struct` параметра текущим методом — ссылка не может «убежать» наружу:

```csharp
public static void Fill(scoped Span<byte> buffer)
{
    buffer.Clear();
    // compiler гарантирует: buffer не сохранится в поле/не вернётся из метода
}
```

Позволяет compiler'у принимать код, который иначе отклонил бы ref-safety анализ (например, передачу `stackalloc`-буфера).

### 12.17. allows ref struct (C# 13+)

Anti-constraint для generics — разрешает `ref struct` (например, `Span<T>`) как type argument:

```csharp
public static void Process<T>(T value) where T : allows ref struct
{
    // T может быть Span<char>; boxing и поля класса по-прежнему запрещены
}
```

Детали ограничений — [[types-and-memory|Types и Memory]] раздел 4.

### 12.18. extension (C# 14+)

Открывает `extension`-блок в static class — extension members (properties, static members, operators), не только методы:

```csharp
public static class StringExtensions
{
    extension(string s)
    {
        public bool IsBlank => string.IsNullOrWhiteSpace(s);   // extension property
    }
}
```

Детали — [[modern-features|Modern C# Features]] раздел 12.

---

## 13. Best Practices

### 13.1. Используй modern features

- ✅ `record` для DTO / value objects.
- ✅ `init` для immutable properties.
- ✅ `required` для mandatory data.
- ✅ `var` когда тип очевиден.
- ✅ `nameof` для refactor-safe strings.
- ✅ `using var` для disposable.

### 13.2. Избегай

- ❌ `dynamic` — type safety lost, slow.
- ❌ `unsafe` — only для interop / micro-perf.
- ❌ `goto` — except switch fall-through.
- ❌ `volatile` — обычно `Interlocked` или `lock` лучше.
- ❌ `new` modifier — почти всегда anti-pattern.

### 13.3. Modifiers

- ✅ `sealed` для классов по default (если не extension intended).
- ✅ `readonly` для fields где возможно.
- ✅ `internal` для implementation details.
- ✅ `static readonly` для public constants (vs `const`).
- ❌ `public` field — used properties.

---

## 14. Decision tree — какой keyword

```
Тип?
├── Reference + behavior → class
├── Value type → struct (immutable → readonly struct)
├── Contract → interface
├── Constants set → enum (с [Flags] для bits)
├── Function pointer → delegate
└── Immutable data → record (record class / record struct)

Modifier для members?
├── Доступ → public/internal/protected/private
├── Constant → const (compile-time) или static readonly (runtime)
├── No mutation → readonly (field/method/struct)
├── Override → virtual + override
├── Hide → new (anti-pattern)
├── Final → sealed
└── Immutable property → init

Control flow?
├── Pattern check → is + switch
├── Lazy iteration → yield return
├── Async → async/await
└── Synchronization → lock
```

---

## 15. Cheat sheet

```csharp
// === Types ===
public class C { }
public struct S { }
public readonly struct RS { }
public ref struct Span<T> { /* ... */ }
public interface I { }
public enum E { A, B }
public delegate void D();
public record R(int X);
public record struct RecS(int X);

// === Modifiers ===
public, internal, protected, private,
protected internal, private protected, file
static, abstract, virtual, override, sealed, new
readonly, const, partial, async, extern, unsafe, volatile

// === Modern (C# 9+) ===
public record User(int Id, string Name)
{
    public required string Email { get; init; }
}

var u2 = u1 with { Name = "Bob" };

// raw strings (C# 11+)
string s = """text""";

// field keyword (C# 14; preview в C# 13)
public int X
{
    get;
    set => field = value > 0 ? value : 0;
}

// === Pattern matching ===
if (obj is User { Email: "x@y.com" } u) { }
var r = obj switch
{
    int n when n > 0 => "positive",
    null => "null",
    _ => "other"
};

// === Async ===
public async Task<int> M(CancellationToken ct)
{
    await Task.Delay(100, ct);
    return 42;
}

// === Resource ===
using var x = new Resource();
await using var y = new AsyncResource();

// === Const / readonly ===
public const int Max = 100;
public static readonly DateTime Epoch = DateTime.UtcNow;
private readonly IRepo _repo;
```

---

## 16. Common Pitfalls

### 16.1. const в public API

```csharp
// MyLibrary v1
public const int MaxRetries = 3;

// MyLibrary v2 — изменили на 5
public const int MaxRetries = 5;

// Caller компилирован против v1 — продолжает использовать 3 (embedded)
```

**Фикс:** `static readonly` для public constants.

### 16.2. virtual без override expectations

```csharp
public class Base { public virtual void M() { } }
// никто никогда не overrides — почему virtual?
```

**Фикс:** sealed по default.

### 16.3. new keyword

```csharp
public class Derived : Base
{
    public new void M() { }   // anti-pattern usually
}
```

**Фикс:** virtual + override.

### 16.4. dynamic в performance hot path

```csharp
dynamic d = GetData();
for (int i = 0; i < 1_000_000; i++)
    d.Method();   // ❌ DLR resolution каждый раз
```

**Фикс:** static typing.

### 16.5. lock на public object

```csharp
private readonly object _lock = this;   // ❌ external code может lock тоже
lock (_lock) { /* ... */ }
```

**Фикс:** `private readonly object _lock = new();`.

### 16.6. var для unclear types

```csharp
var x = service.Process();   // что возвращает? Type unclear
```

**Фикс:** explicit type или rename method.

### 16.7. checked overflow surprise

```csharp
int x = int.MaxValue;
int y = x + 1;   // -2147483648 (wraparound) если не checked
```

**Фикс:** `checked { }` в critical math, или enable globally.

### 16.8. unsafe без understanding

```csharp
unsafe int* GetPtr() => stackalloc int[10];   // ❌ stack memory escapes!
```

**Фикс:** `Span<int>` для safe stack alternative.

### 16.9. partial class spread across many files

```csharp
// MyClass.A.cs, MyClass.B.cs, MyClass.C.cs, ...
// 10 partial files — hard to navigate
```

**Фикс:** обычно один partial file достаточно (или generated). Multiple — symptom of god class.

### 16.10. global using over-applied

```csharp
// GlobalUsings.cs
global using System;
global using Newtonsoft.Json;   // applies to ALL files — namespace pollution
```

**Фикс:** только common usings (System, System.Collections.Generic).

---

## 17. Что читать дальше

1. **[[csharp-basics|C# Basics]]** — fundamentals.
2. **[[oop|OOP]]** — class/struct/record deep.
3. **[[modern-features|Modern Features]]** — records, init, with, primary constructors.
4. **[[generics-deep|Generics deep]]** — `where T : notnull`.
5. **C# Language Reference (Microsoft Docs)** — full keywords list.

---

## 18. См. также

- [[csharp-basics|C# Basics]]
- [[oop|OOP]]
- [[modern-features|Modern Features]]
- [[generics-deep|Generics deep]]
- [[iterators-yield|yield]]
- [[dispose-pattern|using/await using]]
- [[error-handling|try/catch/throw]]

---

## 19. Reading list

- **Microsoft Docs — C# Reference** — learn.microsoft.com/dotnet/csharp/language-reference/keywords/
- **Microsoft Docs — Contextual keywords** — learn.microsoft.com/dotnet/csharp/language-reference/keywords/contextual-keywords
- **Microsoft Docs — What's new in C#** — learn.microsoft.com/dotnet/csharp/whats-new/
- **Jon Skeet — C# in Depth** — comprehensive language reference
- **Bill Wagner — Effective C#** — keyword usage best practices
- **Roslyn source** — github.com/dotnet/roslyn (для compiler implementation details)
