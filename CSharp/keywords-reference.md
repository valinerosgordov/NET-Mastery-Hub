---
tags: [csharp, keywords, reference, ref, in, out, scoped, required, init, file, checked, middle]
level: Middle
date: 2026-04-30
---

# C# Keywords Reference — справочник

> **Все ключевые слова которые часто встречаются в современном C#**: `ref/in/out/scoped/required/init/file/checked/unchecked/sealed/abstract/static/readonly` и т.д. С case studies где каждый применяется.

---

## Что это, зачем и когда

В C# — десятки keywords. Этот файл — **алфавитный справочник** с примерами и case studies. Используется как lookup при чтении/написании кода.

### Полный список

```
abstract, as, async, await, base, break, case, catch, checked, class, const,
continue, default, delegate, do, dynamic, else, enum, event, explicit, extern,
file, finally, fixed, for, foreach, get, global, goto, if, implicit, in, init,
interface, internal, is, lock, namespace, new, null, operator, out, override,
params, partial, private, protected, public, readonly, ref, required, return,
sealed, set, sizeof, stackalloc, static, struct, switch, this, throw, try, 
typeof, unchecked, unsafe, using, value, var, virtual, void, volatile, where,
while, yield
```

В этом файле — **группированы по теме** с case studies для самых нетривиальных.

---

## 1. Parameter modifiers — `ref`, `out`, `in`, `scoped`

### `ref` — pass by reference

```csharp
public void DoubleIt(ref int x)
{
    x = x * 2;
}

int n = 5;
DoubleIt(ref n);  // n = 10 (modified!)

// Caller обязан сначала assign value
int unset;
DoubleIt(ref unset);  // ❌ Compile error — must be assigned
```

**Когда использовать `ref`:**
- Изменить переменную из callee
- Передать большой struct без копирования
- Избежать boxing для interfaces

```csharp
// Pass big struct by reference (perf)
public struct LargeMatrix { public int[16] Data; /* ~64 bytes */ }

public void Process(ref LargeMatrix m)  // no copy!
{
    m.Data[0] = 42;
}
```

### `out` — output parameter

```csharp
public bool TryDivide(int a, int b, out int result)
{
    if (b == 0) { result = 0; return false; }
    result = a / b;
    return true;
}

// Caller — value не нужно assign
if (TryDivide(10, 2, out int r))
{
    Console.WriteLine(r);  // 5
}

// Method обязан assign перед exit
public void Bad(out int x) { }  // ❌ Compile error — x не assigned
```

**Когда использовать `out`:**
- Возвращать несколько значений (TryParse pattern)
- Method может fail — return bool + result
- В .NET API: `int.TryParse`, `Dictionary.TryGetValue`

### `in` — readonly reference (.NET 7.2+)

```csharp
public double DistanceFrom(in Vector3 origin)
{
    // origin доступен read-only — нельзя изменить
    // но передан без копирования
    return Math.Sqrt(origin.X * origin.X + origin.Y * origin.Y + origin.Z * origin.Z);
    
    // origin.X = 5;  // ❌ Compile error — readonly
}

Vector3 v = new(1, 2, 3);
double d = DistanceFrom(in v);  // no copy, no modification
```

**Когда использовать `in`:**
- Большие structs, read-only access
- Performance-critical code
- API contract — "я не буду изменять"

### `scoped` (.NET 7+) — escape protection

```csharp
public Span<int> GetBuffer()
{
    Span<int> stack = stackalloc int[10];
    return stack;  // ❌ Compile error — escapes scope
}

// scoped явно говорит "не выйдет из scope"
public void Process(scoped Span<int> buffer)
{
    // buffer не может escape — compile-time гарантия
    // Можно безопасно передавать stackalloc
    
    Span<int> stack = stackalloc int[10];
    Process(stack);  // OK — scoped enforces
}
```

**Когда использовать `scoped`:**
- Span/ref принимающие методы где явная гарантия non-escape
- Безопасная передача `stackalloc` Span
- API contract без heap escape

### Ref returns and ref locals (C# 7+)

```csharp
public ref int GetSlot(int[] arr, int idx)
{
    return ref arr[idx];  // return reference, not value
}

int[] data = { 1, 2, 3 };
ref int slot = ref GetSlot(data, 1);
slot = 99;  // мутирует data[1] напрямую!
Console.WriteLine(data[1]);  // 99
```

### Senior interview: `ref` vs `out` vs `in`

```csharp
public void Method1(ref int x)  { x = 10; }    // bidirectional
public void Method2(out int x)  { x = 10; }    // only output (must assign)
public void Method3(in int x)   { /* read */ }  // only input (read-only ref)

// Вопрос: какой быстрее для большого struct?
// in (no copy + no allocation), ref (no copy)
// out — обычно не для structs, для multi-return
```

См. [[unsafe-pointers|Unsafe & Pointers]] и [[../Runtime/span-layout|Span layout]].

---

## 2. Inheritance / polymorphism — `abstract`, `virtual`, `override`, `sealed`, `new`

### `abstract` — must be overridden

```csharp
public abstract class Shape
{
    public abstract double Area();  // no body — must override
    
    public virtual void Print() => Console.WriteLine($"Shape: {Area()}");
}

public class Circle : Shape
{
    public double Radius;
    public override double Area() => Math.PI * Radius * Radius;  // обязательно
}

// Cannot instantiate
var s = new Shape();  // ❌ Compile error
```

### `virtual` / `override`

```csharp
public class Animal
{
    public virtual void Speak() => Console.WriteLine("Some sound");
}

public class Dog : Animal
{
    public override void Speak() => Console.WriteLine("Woof!");
}

Animal a = new Dog();
a.Speak();  // "Woof!" — virtual dispatch
```

### `sealed` — prevent further inheritance

```csharp
public sealed class Constants
{
    public const double Pi = 3.14;
}

// Нельзя наследовать
public class Derived : Constants  // ❌ Compile error
```

```csharp
// Sealed на override — для performance
public class Animal
{
    public virtual void Speak() { }
}

public class Cat : Animal
{
    public sealed override void Speak() { }  // нельзя override дальше
}
```

**Performance note:** sealed классы и методы — быстрее virtual call, JIT может devirtualize.

### `new` — hide inherited member

```csharp
public class Base { public void Method() => Console.WriteLine("Base"); }

public class Derived : Base
{
    public new void Method() => Console.WriteLine("Derived");  // hides base
}

Base b = new Derived();
b.Method();   // "Base" — НЕ override!

Derived d = new Derived();
d.Method();   // "Derived"
```

> [!warning] `new` ≠ `override`
> `new` хайдит метод. Через base reference — вызовется base. Это **обычно не то что хочешь**.

### Case Study: Template Method

```csharp
public abstract class DocumentExporter
{
    // Template method — algorithm shape
    public void Export(Document doc)
    {
        OpenStream();
        WriteHeader(doc);
        WriteBody(doc);  // делегируем подклассам
        WriteFooter(doc);
        CloseStream();
    }
    
    protected virtual void OpenStream() { }
    protected virtual void WriteHeader(Document doc) { }
    protected abstract void WriteBody(Document doc);  // обязательно override
    protected virtual void WriteFooter(Document doc) { }
    protected virtual void CloseStream() { }
}

public class PdfExporter : DocumentExporter
{
    protected override void WriteBody(Document doc) { /* PDF specific */ }
}

public class HtmlExporter : DocumentExporter
{
    protected override void WriteBody(Document doc) { /* HTML specific */ }
}
```

См. [[oop|OOP]] и [[design-patterns|Template Method]].

---

## 3. Modifiers — `static`, `readonly`, `const`, `volatile`

### `static` — class-level

```csharp
public static class MathUtils
{
    public static double Pi = 3.14;
    public static double Square(double x) => x * x;
}

// Use — без instance
MathUtils.Square(5);
```

### `readonly` — assigned only in declaration or constructor

```csharp
public class Config
{
    public readonly string Url;          // assigned в constructor
    public readonly Guid Id = Guid.NewGuid();  // или в declaration
    
    public Config(string url) => Url = url;
    
    public void Method()
    {
        Url = "...";  // ❌ Compile error — readonly
    }
}
```

### `const` — compile-time constant

```csharp
public class Constants
{
    public const double Pi = 3.14;       // только примитивные / string
    public const string Greeting = "Hello";
    public const int MaxRetries = 5;
    
    // ❌ public const DateTime Now = DateTime.Now;  // не const!
}

// Compile-time constant — встраивается в callers
// Если меняется — нужно перекомпилировать всех users!
```

### `static readonly` vs `const`

```csharp
// const — встраивается в IL → если меняется значение, callers нужно перекомпилировать
public const double Pi = 3.14;

// static readonly — runtime field → callers не нужно перекомпилировать
public static readonly double Pi = 3.14;

// Rule: для public API — static readonly. const — internal.
```

### `volatile` — memory ordering

```csharp
public class Counter
{
    private volatile int _value;
    
    public int Value => _value;  // memory barrier — гарантирует latest
    public void Increment() => Interlocked.Increment(ref _value);
}
```

> [!warning] `volatile` редко то что нужно
> Большинство concurrent scenarios решаются через `Interlocked`, `lock`, или `ConcurrentDictionary`. `volatile` — только для специфических lock-free patterns. См. [[../Runtime/concurrency-atomics|Concurrency & Atomics]].

---

## 4. Modern keywords — `init`, `required`, `file`, `record`

### `init` (C# 9+) — set only at object initialization

```csharp
public class User
{
    public string Name { get; init; }
    public string Email { get; init; }
}

// Можно set в object initializer
var u = new User { Name = "John", Email = "j@x.com" };

// После — нельзя
u.Name = "Jane";  // ❌ Compile error
```

### `required` (C# 11+) — caller обязан init

```csharp
public class User
{
    public required string Name { get; init; }
    public required string Email { get; init; }
}

// ❌ Compile error — required Name, Email
var u = new User();

// ✅
var u = new User { Name = "John", Email = "j@x.com" };
```

**Когда использовать:**
- Без default value
- Чтобы compiler enforced — пользователь не забыл

### `file` (C# 11+) — file-scoped types

```csharp
// MyHelpers.cs
file class InternalHelper  // visible только в этом файле
{
    public static void Method() { }
}

// AnotherFile.cs
InternalHelper.Method();  // ❌ Compile error — not visible
```

**Когда использовать:**
- Source generators (избегать конфликтов names)
- Helpers только для одного файла

### `record` (C# 9+)

```csharp
public record User(string Name, string Email);

// Auto-generated:
// - Constructor
// - Properties (init-only)
// - Equals / GetHashCode (по value)
// - ToString
// - Deconstruct
// - with expression

var u = new User("John", "j@x.com");
var u2 = u with { Name = "Jane" };  // copy + modification
u == new User("John", "j@x.com")    // true (value equality)
```

См. [[modern-features|Modern C# Features]] и [[functional-csharp|Functional C#]].

---

## 5. Generics — `where`, `class`, `struct`, `new()`

```csharp
public T Process<T>(T item)
    where T : class                    // reference type
    where T : struct                   // value type
    where T : new()                    // has parameterless constructor
    where T : Animal                   // inherits Animal
    where T : IComparable<T>           // implements interface
    where T : notnull                  // не nullable
    where T : unmanaged                // unmanaged struct (no managed refs)
{
    return item;
}

// Multiple constraints
public T Method<T>() where T : class, IDisposable, new()
{
    var t = new T();
    t.Dispose();
    return t;
}
```

См. [[generics-deep|Generics Deep]].

---

## 6. Async / threading — `async`, `await`, `lock`

### `async` / `await`

```csharp
public async Task<int> GetDataAsync()
{
    var data = await httpClient.GetStringAsync(url);
    return data.Length;
}
```

### `lock`

```csharp
private readonly object _lock = new();
private int _counter = 0;

public void Increment()
{
    lock (_lock)
    {
        _counter++;
    }
}
```

См. [[async-threading|Async & Threading]] и [[../Runtime/concurrency-atomics|Concurrency]].

---

## 7. Pattern matching — `is`, `as`, `switch`

### `is` — type check + cast

```csharp
object obj = "hello";

// C# 6 — just check
if (obj is string)
{
    string s = (string)obj;
}

// C# 7+ — pattern matching
if (obj is string s)
{
    // s already typed!
}

// C# 9+ — patterns
if (obj is string { Length: > 5 } longStr)
{
    Console.WriteLine($"Long string: {longStr}");
}

// Negative
if (obj is not null)
{
    // ...
}
```

### `as` — safe cast or null

```csharp
object obj = "hello";

string? s = obj as string;  // null если не string

// Compare с (cast) — throws InvalidCastException
string s = (string)obj;
```

### `switch` expression (C# 8+)

```csharp
public string Describe(object o) => o switch
{
    int i when i < 0 => "Negative",
    int i => $"Number: {i}",
    string s => $"String: {s}",
    null => "Null",
    _ => "Unknown"
};
```

См. [[modern-features|Modern Features]].

---

## 8. Overflow control — `checked`, `unchecked`

### `checked` — throw on overflow

```csharp
int max = int.MaxValue;
int result = checked(max + 1);  // OverflowException

// Или для всего scope
checked
{
    int a = max + 1;  // throws
}
```

### `unchecked` — silent wrap (default)

```csharp
int max = int.MaxValue;
int result = unchecked(max + 1);  // = -2147483648, no exception

// Default behavior — unchecked
int result = max + 1;  // тоже -2147483648 (silent)
```

### Когда использовать

- **`checked`** — security critical (financial, sizes)
- **`unchecked`** — hash codes, bit operations (overflow expected)
- **csproj setting** — `<CheckForOverflowUnderflow>true</CheckForOverflowUnderflow>` для всего projecta

### Case Study: Financial calculation

```csharp
public decimal Calculate(decimal[] amounts)
{
    decimal total = 0;
    foreach (var a in amounts)
    {
        // Decimal не имеет overflow в обычной арифметике
        // (throws OverflowException автоматически)
        total += a;
    }
    return total;
}

// Для int — нужен checked
public int SumInt(int[] amounts)
{
    int total = 0;
    foreach (var a in amounts)
    {
        total = checked(total + a);  // safety!
    }
    return total;
}
```

См. [[numeric-types-math|Numeric Types & Math]].

---

## 9. Memory management — `unsafe`, `fixed`, `stackalloc`

```csharp
public unsafe void Method()
{
    int x = 5;
    int* p = &x;
    *p = 10;
    
    // stackalloc — на стеке
    int* arr = stackalloc int[10];
    
    // fixed — pin GC
    int[] data = { 1, 2, 3 };
    fixed (int* pData = data)
    {
        // pData stable, GC не двигает
    }
}
```

См. [[unsafe-pointers|Unsafe & Pointers]].

---

## 10. Resource management — `using`

### `using` statement (старый)

```csharp
using (var reader = new StreamReader("file.txt"))
{
    string content = reader.ReadToEnd();
}
// Auto Dispose
```

### `using` declaration (C# 8+) — until end of scope

```csharp
public void Method()
{
    using var reader = new StreamReader("file.txt");
    string content = reader.ReadToEnd();
}  // disposed here
```

### `await using` — для IAsyncDisposable

```csharp
await using var conn = new SqlConnection(...);
await conn.OpenAsync();
```

См. [[dispose-pattern|Dispose Pattern]].

---

## 11. Exception handling — `try`, `catch`, `finally`, `throw`

```csharp
try
{
    // risky code
}
catch (FileNotFoundException ex)
{
    // specific
}
catch (Exception ex) when (ex.Message.Contains("specific"))
{
    // filter
}
finally
{
    // cleanup, всегда выполняется
}

// Re-throw без потери stack
catch (Exception ex)
{
    Log(ex);
    throw;  // ✅ preserves stack
    // throw ex;  // ❌ resets stack — bad!
}

// Throw expression
public string GetName(string? name) => 
    name ?? throw new ArgumentNullException(nameof(name));
```

См. [[error-handling|Error Handling]].

---

## 12. Iteration — `yield`, `foreach`, `for`, `do/while`

### `yield return` — generator

```csharp
public IEnumerable<int> Range(int start, int end)
{
    for (int i = start; i < end; i++)
        yield return i;
}

// Use
foreach (var n in Range(1, 5))
    Console.WriteLine(n);
```

### `yield break` — exit generator

```csharp
public IEnumerable<int> TakeWhilePositive(IEnumerable<int> source)
{
    foreach (var x in source)
    {
        if (x <= 0) yield break;
        yield return x;
    }
}
```

См. [[iterators-yield|Iterators & yield]].

---

## 13. Access modifiers — `public`, `private`, `protected`, `internal`

```csharp
public      // visible everywhere
internal    // visible в same assembly
protected   // visible в this class + derived
protected internal  // protected OR internal (derived OR same assembly)
private protected   // protected AND internal (derived AND same assembly)
private     // visible только в this class
file        // C# 11+ — visible только в this file
```

### Case Study: API surface design

```csharp
public class UserService
{
    public async Task<User> GetById(int id) { /* ... */ }   // API — public
    
    private readonly IRepository _repo;                      // internal state — private
    
    internal void ResetCache() { /* ... */ }                 // for tests in same assembly
    
    protected virtual void OnUserLoaded(User u) { }         // for subclasses
}
```

---

## 14. Inheritance keywords — `interface`, `class`, `struct`, `partial`

### `partial` — split type across files

```csharp
// User.cs
public partial class User
{
    public string Name { get; set; }
}

// User.Generated.cs (e.g. от code generator)
public partial class User
{
    public Guid Id { get; set; }
}

// Все public/private modifiers must match
// Все atomic — compiler combines into single class
```

### Case Study: Source generators

```csharp
[GenerateValidator]
public partial class CreateUserRequest
{
    public string Email { get; set; }
}

// Source generator generates:
public partial class CreateUserRequest
{
    public ValidationResult Validate()
    {
        // ... auto-generated
    }
}
```

См. [[source-generators|Source Generators]].

---

## 15. Specialized — `params`, `default`, `nameof`, `typeof`, `sizeof`

### `params` — variadic arguments

```csharp
public int Sum(params int[] nums)
{
    return nums.Sum();
}

Sum(1, 2, 3);
Sum(1, 2, 3, 4, 5);
Sum();  // empty
Sum(new[] { 1, 2, 3 });  // pass array
```

### `default` — default value

```csharp
int x = default;       // 0
string s = default;    // null
DateTime d = default;  // 0001-01-01

// Generic
T value = default(T);
T value = default;     // C# 7.1+

// Default literal в method
public T Get<T>() => default;
```

### `nameof` — string from identifier

```csharp
public void Validate(string parameter)
{
    if (parameter == null)
        throw new ArgumentNullException(nameof(parameter));  // = "parameter"
}

// Refactoring-safe — переименование parameter обновит nameof
```

### `typeof` — Type object

```csharp
Type t = typeof(string);
Type t = typeof(List<int>);

// Compare
if (obj.GetType() == typeof(int)) { /* ... */ }
```

### `sizeof` — размер в bytes

```csharp
int size = sizeof(int);          // 4
int size = sizeof(double);        // 8
int size = sizeof(MyStruct);     // если blittable
```

---

## 16. Cheat sheet — частые

| Keyword | Когда |
|---------|-------|
| `ref` | Modify caller's variable, large struct perf |
| `out` | TryParse pattern, multi-return |
| `in` | Large struct read-only param |
| `scoped` | Stack-only Span, no escape |
| `init` | Set только in initializer, immutability |
| `required` | Compiler enforce caller инит |
| `file` | File-scoped types (rare) |
| `sealed` | Prevent inheritance, perf |
| `static readonly` | Public-API constant (vs const) |
| `volatile` | Lock-free patterns (rare) |
| `params` | Variadic args |
| `nameof` | Refactor-safe string |
| `default` | Default value, generic |
| `partial` | Source generators, split files |
| `record` | Value equality, immutable |
| `with` | Record cloning + modification |
| `checked` | Overflow guards |
| `using` (decl) | Auto-dispose до end of scope |

---

## 17. Decision tree

```
Что делаешь?
│
├── Modify caller variable → ref / out
├── Optimize struct param → in / ref
├── Set once after init → readonly или init
├── Compile-time constant → const
├── Runtime constant (public) → static readonly
├── Force caller initialization → required
├── Prevent inheritance → sealed
├── Multi-return → out parameters или tuple
├── Auto-dispose → using declaration
├── Variadic args → params
├── Type info → typeof / GetType
├── Identifier → string → nameof
├── Pin memory для interop → fixed
├── Heap-free buffer → stackalloc + Span
├── Pattern check + cast → is pattern
└── Source generated splitting → partial
```

---

## См. также

- [[csharp-basics|C# Basics]] — fundamental keywords
- [[modern-features|Modern Features]] — init, required, file, records
- [[oop|OOP]] — abstract/virtual/sealed
- [[generics-deep|Generics Deep]] — where constraints
- [[unsafe-pointers|Unsafe & Pointers]] — fixed, stackalloc, unsafe
- [[error-handling|Error Handling]] — try/catch/finally
- [[dispose-pattern|Dispose Pattern]] — using
- [[iterators-yield|Iterators]] — yield
- [[async-threading|Async]] — async/await/lock
- [[../Runtime/span-layout|Span layout]] — scoped, ref struct
- [[../Runtime/concurrency-atomics|Concurrency]] — volatile

## Reading list

- **Microsoft Docs — C# keywords** — learn.microsoft.com/dotnet/csharp/language-reference/keywords
- **Microsoft Docs — Modifiers** — learn.microsoft.com/dotnet/csharp/language-reference/keywords/modifiers
- **Eric Lippert blog** — ericlippert.com (history of language design)
- **Mads Torgersen — C# evolution** — devblogs.microsoft.com/dotnet
- **C# in Depth** — Jon Skeet (классика)
