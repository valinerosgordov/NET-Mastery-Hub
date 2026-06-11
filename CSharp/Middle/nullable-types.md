---
tags: [csharp, nullable, middle, nrt, nullable-reference-types, null-safety]
level: Middle
date: 2026-05-07
---

# Nullable Types — `Nullable<T>` и Nullable Reference Types

> **Два разных механизма с одним именем.** `Nullable<T>` (value types, .NET 2.0+) — runtime feature; Nullable Reference Types (NRT, C# 8+) — **static analysis**, no runtime cost. Закрывает пробел: «знаю про `int?`, не понимаю что значит warning CS8600 и почему `string?` не reference equal `string`».

---

## 0. Как читать

Если впервые видишь `?` после типа — раздел 1→3 подряд. Если уже работаешь с NRT, но непонятно null-forgiving `!` — раздел 5. Production guidance — раздел 8 (best practices), 10 (pitfalls).

Cross-language якоря свёрнуты. Interview-вопросы встроены.

---

## 1. Что это, зачем и когда

### 1.1. Две разные фичи с одним синтаксисом

```csharp
int? a = null;          // Nullable<int> — value type wrapper (runtime)
string? b = null;       // string с nullable annotation (compile-time)
```

`int?` и `string?` оба используют `?`, но это **разные** механизмы:

| | `int?` (`Nullable<T>`) | `string?` (NRT) |
|---|----------|-----|
| Появилось | .NET 2.0 (2005) | C# 8.0 (2019) |
| Вид feature | Runtime feature | Compile-time analysis |
| Тип | `Nullable<int>` (struct) | `string` (то же class) |
| Размер | 8 bytes (bool + int) | как `string` |
| Runtime check | `HasValue` | nope — все references могут быть null в runtime |
| Покрывает | Value types | Reference types |

### 1.2. Зачем `Nullable<T>`

Value types **не могут быть null** by design — `int x = null;` compile error. Но иногда нужно "значение неизвестно":

```csharp
int dbColumnValue;        // ❌ как сказать "NULL в БД"?
int? dbColumnValue;        // ✅ may be null
```

`Nullable<T>` — wrapper struct с `HasValue` flag.

### 1.3. Зачем NRT (Nullable Reference Types)

Reference types **могут быть null** в runtime. Это NullReferenceException — самая частая bug в C#:

```csharp
string name = GetUser().Name;   // что если GetUser() returns null?
// → NullReferenceException at runtime
```

NRT — **compile-time annotations** + flow analysis: `string` = "non-null" promise, `string?` = "may be null". Compiler warning при unsafe access.

### 1.4. Главное правило

```
int?, double?, bool? — Nullable<T> wrapper для value types
  - HasValue, Value, GetValueOrDefault
  - ?? и ?. для access

string?, User? — NRT annotations (если #nullable enable)
  - все references могут быть null в runtime, но compiler помогает
  - ! null-forgiving operator
  - = null! для escape hatch

Включай <Nullable>enable</Nullable> в проекте — стандарт с .NET 6+.
```

### 1.5. Эволюция

| Версия | Что |
|--------|-----|
| **.NET 2.0** | `Nullable<T>` для value types |
| **C# 2.0** | `?` syntax (`int?` ≡ `Nullable<int>`), `??` operator |
| **C# 6.0** | `?.` null-conditional operator |
| **C# 7.1** | `default` literal |
| **C# 8.0** | **NRT** (Nullable Reference Types), `!` null-forgiving |
| **.NET 6+** | NRT enabled by default в новых templates |
| **C# 11** | required properties — non-null guarantees |

> [!info]- Если ты знаешь Java / Kotlin / TypeScript / Rust
> **Java:** есть только `Optional<T>` wrapper. Нет compile-time NRT — все references могут быть null. Project Lombok / annotations like @NotNull / @Nullable — partial.
>
> **Kotlin:** ровно то же что C# NRT — `String?` vs `String`, compile-time enforced. Inspired NRT design.
>
> **TypeScript:** `string | null`, `string | undefined`, `--strictNullChecks`. Очень похож на C# NRT.
>
> **Rust:** `Option<T>` enum (Some/None) — runtime, нет null reference в принципе. Compile-time guarantee через type system.

> [!question]- Интервью: чем `int?` отличается от `string?`?
> `int?` — `Nullable<int>` struct, runtime feature (.NET 2.0+). 8 bytes (bool flag + int value), `HasValue` / `Value` properties. Без него value type не может быть null. `string?` — **только compile-time annotation** (C# 8+, NRT). Это всё ещё `string` (reference type). В runtime все references могут быть null. NRT даёт compiler warnings при unsafe access. Включается через `#nullable enable` или `<Nullable>enable</Nullable>` в csproj. Standard в .NET 6+ templates.

---

## 2. Nullable&lt;T&gt; для value types

### 2.1. Объявление

```csharp
int? a = null;
double? b = 3.14;
DateTime? c = DateTime.Now;
bool? d = null;

// Эквивалентно
Nullable<int> a2 = null;
Nullable<double> b2 = 3.14;
```

`int?` — синтаксический сахар для `Nullable<int>`.

### 2.2. HasValue, Value, GetValueOrDefault

```csharp
int? x = 42;
x.HasValue;                   // true
x.Value;                       // 42
x.GetValueOrDefault();         // 42
x.GetValueOrDefault(99);       // 42 (использует value)

int? y = null;
y.HasValue;                   // false
y.Value;                       // ❌ InvalidOperationException
y.GetValueOrDefault();         // 0 (default(int))
y.GetValueOrDefault(99);       // 99 (использует default)
```

### 2.3. Implicit conversion

```csharp
int x = 5;
int? y = x;          // implicit: int → int?
int z = y.Value;     // explicit через .Value (or unbox)
int z2 = (int)y;     // ❌ throws если null
int z3 = y ?? 0;     // safe: 5
```

### 2.4. Operator lifting

Operators автоматически lifted для nullable:

```csharp
int? a = 5, b = 10;
int? sum = a + b;    // 15

int? c = null;
int? sum2 = a + c;   // null (любая операция с null = null)

bool? p = true;
bool? q = null;
bool? r = p && q;    // null (3-valued logic)
bool? s = p || q;    // true (true || ? = true)
```

Arithmetic, comparison, logical — все lifted. Null propagates.

### 2.5. Comparison

```csharp
int? a = 5, b = null;

a == b;          // false
a == 5;          // true
a > b;           // false (всегда false с null)
a < b;           // false
b == null;       // true
b == default;    // true
```

`>`, `<`, `>=`, `<=` с null operand — всегда false (NaN-like semantics).

### 2.6. Boxing nullable

```csharp
int? x = 42;
object o = x;          // boxes как int (42), не как Nullable<int>!
o.GetType();           // typeof(int), not typeof(int?)

int? y = null;
object o2 = y;         // o2 == null (нет boxing!)
o2 == null;            // true
```

Special-cased: nullable boxes как underlying type. `null` nullable boxes to `null` reference.

> [!question]- Интервью: что хранится в `Nullable<T>` struct?
> `Nullable<T> { bool hasValue; T value; }` — bool flag + value. Размер: bool padded к alignment + sizeof(T). Для `int?` — 8 bytes (на 32-bit), 16 bytes (на 64-bit). `HasValue` true → value valid; false → value default(T) (но не используется). Operator lifting автоматически — `a + b` returns null если any operand null. Boxing special-cased: boxes как underlying T (или null если HasValue=false). `int?.Value` throws InvalidOperationException если null — лучше `??` или `GetValueOrDefault`.

---

## 3. Nullable Reference Types (NRT)

### 3.1. Включение

В `.csproj`:
```xml
<PropertyGroup>
  <Nullable>enable</Nullable>
</PropertyGroup>
```

Или per-file:
```csharp
#nullable enable

public class Service { /* NRT active */ }

#nullable disable

public class LegacyService { /* NRT off */ }
```

`.NET 6+` templates имеют `<Nullable>enable</Nullable>` by default.

### 3.2. Annotations

```csharp
#nullable enable

string a = "hello";       // non-nullable
string? b = null;          // nullable
string c = null;           // ⚠ warning CS8600

int? n = null;             // OK — value type nullable (отдельная фича)
List<string?> list;        // list of nullable strings
List<string>? maybe;       // nullable list of non-null strings
```

`?` после type — **may be null**. Без `?` — **promised non-null**.

### 3.3. Flow analysis

```csharp
public string Process(string? input)
{
    // input может быть null
    
    if (input == null) return "default";
    
    // Здесь compiler знает: input не null
    return input.ToUpper();   // OK без warning
}

public int Length(string? input)
{
    return input.Length;   // ⚠ CS8602 — possibly null
}

public int LengthSafe(string? input) =>
    input?.Length ?? 0;
```

Compiler анализирует control flow и снимает warning после null-checks.

### 3.4. Null-forgiving operator `!`

```csharp
public string GetName(int id)
{
    string? name = LookupName(id);
    return name!;   // I PROMISE это не null — suppress warning
}
```

`!` суффикс — "trust me, я знаю что не null". Без runtime check — если ошибся, NRE all the same.

### 3.5. ! на declarations

```csharp
public class Service
{
    public string Name { get; set; } = null!;   // initialized позже (DI, factory)
    public List<User> Users { get; set; } = null!;
}
```

`= null!` — suppress warning при declaration. Используется для DI-injected или late-initialized полей.

### 3.6. required (C# 11+)

```csharp
public class User
{
    public required string Email { get; init; }   // caller обязан задать
    public required string Name { get; init; }
}

var u = new User { Email = "a@x.com", Name = "Alice" };   // OK
// var u2 = new User();                                     // ❌ compile error
```

`required` — гарантия non-null без `= null!` workaround. Compile-time enforced.

### 3.7. Constructor injection guarantees

```csharp
public class Service
{
    private readonly IRepository _repo;   // non-null
    
    public Service(IRepository repo)
    {
        ArgumentNullException.ThrowIfNull(repo);
        _repo = repo;
    }
}
```

Constructor подтверждает non-null. Compiler знает после assignment.

### 3.8. Generic nullability

```csharp
public T? FirstOrDefault<T>(IEnumerable<T> source)
{
    // ...
}

// Для T = string (reference) → string?
// Для T = int (value)        → int?
// Для T = struct DateTime    → DateTime?

string? s = list.FirstOrDefault();   // string?
int? n = list.FirstOrDefault();      // int?
```

`T?` в generic — compiler смотрит constraint:
- `where T : class` → `string?`
- `where T : struct` → `int?` (Nullable<int>)
- Без constraint → MaybeNull annotation (advanced)

> [!question]- Интервью: NRT влияет ли на runtime?
> **Нет** — NRT это **compile-time only**. Bytecode не меняется, runtime characteristics те же. `string?` и `string` в IL identical (есть только attribute `NullableAttribute` для metadata). Главные effects: 1) Compiler **warnings** при unsafe access. 2) Runtime может всё равно бросить NRE — NRT не runtime guard. 3) `!` operator suppresses warning без runtime check. NRT — instrument для catching null bugs **на compile**, не replacement runtime null checks.

---

## 4. ?? и ?. operators

### 4.1. ?? null-coalescing

```csharp
string? input = GetInput();
string output = input ?? "default";   // если input null → "default"

int? n = null;
int x = n ?? 0;                        // 0

User u = repo.Find(id) ?? new User();
```

`a ?? b` — return a если non-null, иначе b.

### 4.2. ??= null-coalescing assignment

```csharp
string? name = null;
name ??= "Unknown";   // assigns если null

// Эквивалент
if (name == null) name = "Unknown";
```

`a ??= b` — assigns b если a null.

### 4.3. ?. null-conditional

```csharp
string? name = user?.Name;             // null если user null
int? len = user?.Name?.Length;          // null propagates через chain

User? user = repo.Find(id);
user?.Update();   // вызов только если non-null

list?.Add(item);   // safe call
```

### 4.4. ?[] null-conditional indexer

```csharp
int[]? arr = GetArray();
int? first = arr?[0];   // null если arr null
```

### 4.5. Combining

```csharp
string firstWord = sentence?.Split(' ')?[0] ?? "(empty)";
//      ↑ chain через ?.        ↑  fallback через ??
```

### 4.6. Null-conditional + invocation

```csharp
public event EventHandler<EventArgs>? OnChanged;

OnChanged?.Invoke(this, EventArgs.Empty);   // safe — не throw если no subscribers
```

Idiom для events.

### 4.7. !? — null-forgiving + null-conditional

```csharp
string s = obj!.Property?.Name ?? "fallback";
//          ↑ I promise obj non-null
//                    ↑ but Property may be null
```

> [!question]- Интервью: что делает оператор `?.`?
> **Null-conditional access**: `a?.Member` returns `null` если `a` null, иначе access Member. Тип результата — nullable version. Chains: `a?.b?.c?.d` — null propagates через chain. Для invocation: `event?.Invoke(...)` — popular pattern для null-safe events. Для indexer: `arr?[i]`. Combine с `??` для fallback: `a?.b ?? "default"`. Не путать с `!.` (null-forgiving — promises non-null без access).

---

## 5. Null-forgiving `!` deep

### 5.1. Когда уместно

```csharp
public class Service
{
    private readonly IRepo _repo = null!;   // DI установит позже
    
    public string Name => _repo!.GetName();   // I trust _repo non-null здесь
}
```

`!` — escape hatch для cases когда compiler не может доказать non-null:
- DI fields (заполняются позже).
- Factories / late init.
- Test code где знаем точно.
- Roslyn analysis недостаточно умна для специфичной логики.

### 5.2. Anti-pattern — !\everywhere

```csharp
// ❌ Spam suppress — обходит весь смысл NRT
return user!.Name!.ToString()!;
```

Если часто пишешь `!` — NRT design проекта не правильный. Лучше:
- Validate в constructor.
- Использовать pattern matching.
- Менять signatures.

### 5.3. !!! prevention guide

Reasons для legitimate `!`:
1. **DI fields** (`= null!` initialization).
2. **Roslyn limitation** — компилятор не выводит из вашей логики (например, Dictionary.TryGetValue в C# < 11).
3. **Throws guarantee** — метод throws если null, но compiler не знает.
4. **Test data** — всегда non-null fixtures.

Все остальные cases — refactor.

### 5.4. !? null-forgiving + null-conditional

```csharp
public string Process(User? user)
{
    var name = user!.Name;   // suppress warning, BUT runtime NRE still possible если user null!
    return name;
}

// Better — explicit check
public string Process(User? user)
{
    ArgumentNullException.ThrowIfNull(user);
    return user.Name;   // compiler знает non-null после throw
}
```

`!` does not add runtime safety. Только suppress warning.

### 5.5. ?? throw для assertion

```csharp
public string GetName(int id)
{
    return _names.TryGetValue(id, out var name)
        ? name
        : throw new KeyNotFoundException($"No name for {id}");
}

// или
public string Required(string? input) =>
    input ?? throw new ArgumentNullException(nameof(input));
```

`?? throw` — clear non-null guarantee + meaningful exception.

> [!question]- Интервью: когда уместен оператор `!`?
> Используй **редко** — это escape hatch когда compiler не может доказать non-null, но ты знаешь точно. Legitimate cases: 1) **DI fields** (`= null!` init, framework заполнит). 2) **Roslyn limitations** (TryGetValue в старых C#). 3) **Throws guarantees** в helper methods. 4) **Test fixtures**. Anti-pattern — `!.` everywhere для suppress warnings. Если часто `!` — проблема архитектуры. Лучше: `ArgumentNullException.ThrowIfNull`, `?? throw`, `required` properties, refactoring signatures. `!` без runtime check — не safety, только warning suppress.

---

## 6. Nullable annotations в API

### 6.1. Method signatures

```csharp
public User GetById(int id);          // never returns null
public User? FindById(int id);         // may return null
public void Save(User user);            // user must be non-null
public void Process(User? user);        // user may be null
```

API contracts через nullable annotations — самодокументирующие.

### 6.2. Output parameters

```csharp
public bool TryGet(int id, [NotNullWhen(true)] out User? user)
{
    if (id < 0) { user = null; return false; }
    user = LoadUser(id);
    return user != null;
}

// Использование
if (TryGet(42, out var u))
{
    // compiler знает u non-null здесь
    u.DoSomething();
}
```

`[NotNullWhen(true)]` — atomic: **если return true, then out parameter non-null**.

### 6.3. Постусловия

```csharp
[return: NotNullIfNotNull(nameof(input))]
public string? Process(string? input)
{
    return input?.ToUpper();   // null in → null out, non-null in → non-null out
}
```

`[NotNullIfNotNull]` — output non-null если input non-null.

### 6.4. Arguments

```csharp
public void Save([NotNull] User? user)
{
    ArgumentNullException.ThrowIfNull(user);
    // далее compiler знает user non-null
}

public void Throw([DoesNotReturn]) => throw new InvalidOperationException();
```

`[NotNull]`, `[DoesNotReturn]` — assert null state for compiler.

### 6.5. Property nullable patterns

```csharp
public class User
{
    public required string Email { get; init; }       // non-null guaranteed
    public string? MiddleName { get; init; }          // optional
    public List<string> Tags { get; init; } = [];    // never null, may be empty
    public List<string>? Aliases { get; init; }       // may be null OR empty
}
```

Phrasing: distinguish "non-null but empty" (Tags) от "may be null" (Aliases).

### 6.6. Lazy init

```csharp
public class Cache
{
    private Dictionary<string, User>? _data;
    
    public User Get(string key)
    {
        _data ??= LoadData();
        return _data[key];
    }
}
```

`?` для lazy-init field, `??=` для thread-unsafe lazy.

> [!question]- Интервью: что делает `[NotNullWhen(true)]`?
> Атрибут на out parameter в методе с bool return: **если метод returns true, то out parameter гарантированно non-null**. Compiler использует для flow analysis: после `if (TryGet(out var x))` он знает `x` non-null без warning. Pattern из BCL: `Dictionary.TryGetValue`, `int.TryParse`. Подобные атрибуты: `[NotNullWhenFalse]`, `[NotNullIfNotNull(...)]`, `[MaybeNullWhen(...)]`. В namespace `System.Diagnostics.CodeAnalysis`.

---

## 7. ReSharper / Roslyn warnings

### 7.1. Главные коды

| Код | Что значит |
|-----|------------|
| **CS8600** | Converting null literal to non-nullable |
| **CS8601** | Possible null reference assignment |
| **CS8602** | Dereference of a possibly null reference |
| **CS8603** | Possible null reference return |
| **CS8604** | Possible null reference argument |
| **CS8618** | Non-nullable property must contain non-null value when exiting constructor |
| **CS8619** | Nullability of reference types in value doesn't match target type |
| **CS8625** | Cannot convert null literal to non-nullable reference type |

### 7.2. Treat as errors

В `.csproj`:
```xml
<TreatWarningsAsErrors>true</TreatWarningsAsErrors>
<WarningsNotAsErrors>CS8618</WarningsNotAsErrors>   <!-- например, разрешить uninitialized properties -->
```

Best practice — `TreatWarningsAsErrors` в production. Принуждает обрабатывать null cases.

### 7.3. Per-warning suppression

```csharp
#pragma warning disable CS8602
var x = obj.Property;   // suppress только эту строку
#pragma warning restore CS8602
```

Used редко — обычно `!` или fix root cause.

> [!question]- Интервью: какие основные NRT warning коды?
> **CS8602** — dereference of possibly null reference (самый частый, `obj.Field` где `obj` может быть null). **CS8600** — converting null to non-nullable. **CS8603** — return null из метода с non-null return type. **CS8618** — non-nullable property без initialization в constructor. **CS8625** — `= null` для non-nullable. Best practice: `<TreatWarningsAsErrors>true</TreatWarningsAsErrors>` — null bugs caught при compile, не runtime. Все codes начинаются с CS86xx.

---

## 8. Best Practices

### 8.1. Включай NRT всегда

- ✅ `<Nullable>enable</Nullable>` в каждом проекте.
- ✅ `<TreatWarningsAsErrors>true</TreatWarningsAsErrors>` в CI.
- ✅ Migrate legacy gradually через `#nullable enable` per-file.

### 8.2. API design

- ✅ **`required`** для mandatory init properties.
- ✅ **Constructor validation** через `ArgumentNullException.ThrowIfNull`.
- ✅ **`?? throw`** для assertions.
- ✅ **`[NotNullWhen(true)]`** для Try-pattern out params.
- ✅ **Document intent** через annotations (`?` vs no `?`).

### 8.3. `Nullable<T>` для value types

- ✅ **`int?`, `DateTime?`** для optional value types.
- ✅ **`HasValue`** check вместо `!= null` (clear intent).
- ✅ **`GetValueOrDefault(fallback)`** вместо `?? fallback` для perf.
- ❌ **`Value` без `HasValue` check** — InvalidOperationException.

### 8.4. Operators

- ✅ **`??`** для default fallback.
- ✅ **`?.`** для safe member access.
- ✅ **`??=`** для lazy init.
- ✅ **`?? throw`** для required values.
- ❌ **`!` everywhere** — anti-pattern.

### 8.5. Не делай

- ❌ `string s = null;` — CS8600.
- ❌ Disable NRT в new code.
- ❌ `!.` для suppress без understanding.
- ❌ Mixing `null` и `string.Empty` semantics.
- ❌ Returning null когда `= ""` или `[]` лучше.

---

## 9. Decision tree

```
Что нужно?
│
├── Value type optional → Nullable<T> (int?, DateTime?)
│   ├── HasValue check
│   ├── GetValueOrDefault(fallback)
│   └── Value (только после HasValue check)
│
├── Reference type "may be null" → string?, User?
│   ├── ?. для access
│   ├── ?? для fallback
│   ├── ??= для lazy init
│   ├── ArgumentNullException.ThrowIfNull в methods
│   └── ?? throw для assertion
│
├── Reference "must be non-null" → string, User
│   ├── required (C# 11+) для init properties
│   ├── Constructor + ThrowIfNull
│   └── = "" / [] для defaults вместо null
│
├── Out parameter с null state
│   └── [NotNullWhen(true)] или [MaybeNullWhen(false)]
│
├── Generic with nullability
│   └── T? + where T : class или where T : struct
│
└── Late init / DI
    ├── = null! для escape
    ├── required для compile-time guarantee
    └── Lazy<T> для thread-safe
```

---

## 10. Cheat sheet

```csharp
// === Nullable<T> для value types ===
int? a = null;
double? b = 3.14;
DateTime? c = DateTime.Now;

a.HasValue;               // false
a.Value;                   // throws
a.GetValueOrDefault(0);    // 0
int x = a ?? 99;           // safe

// === NRT for reference types ===
#nullable enable
string a = "hello";        // non-null
string? b = null;          // nullable

string c = b ?? "default";
int? len = b?.Length;
b ??= "fallback";

// === Operators ===
var x = a?.b?.c ?? defaultValue;
var n = arr?[0];
event?.Invoke(...);
var p = repo.Find(id) ?? throw new NotFoundException();

// === Null-forgiving ===
public string Name { get; set; } = null!;   // DI init
return user!.Name;   // I promise non-null

// === Required ===
public required string Email { get; init; }
public required string Name { get; init; }

// === Annotations ===
public bool TryGet(int id, [NotNullWhen(true)] out User? user) { /* ... */ }

[return: NotNullIfNotNull(nameof(input))]
public string? Process(string? input);

public void Save([NotNull] User? user) { ArgumentNullException.ThrowIfNull(user); }

// === Validation ===
ArgumentNullException.ThrowIfNull(obj);
ArgumentException.ThrowIfNullOrEmpty(str);
ArgumentException.ThrowIfNullOrWhiteSpace(str);
```

---

## 11. Common Pitfalls

### 11.1. = null! без понимания

```csharp
public string Name { get; set; } = null!;   // Если DI не заполнит — NRE
```

**Механизм:** suppresses warning, не runtime check. Если что-то пошло не так — runtime NRE.
**Фикс:** validate в constructor или `required`.

### 11.2. Nullable.Value без check

```csharp
int? x = GetValue();
int y = x.Value;   // ❌ InvalidOperationException если null
```

**Фикс:** `x.GetValueOrDefault()` или `?? 0`.

### 11.3. CS8602 spam через !

```csharp
return user!.Name!.ToUpper()!;   // ❌ symptomatic — design issue
```

**Фикс:** validate раз в начале (`ThrowIfNull`), потом compiler знает.

### 11.4. == null vs is null

```csharp
if (obj == null) { }   // calls overload (если есть)
if (obj is null) { }   // ✅ всегда reference compare
```

`is null` — better для NRT context (no operator overload surprise).

### 11.5. Comparing nullable structs

```csharp
DateTime? a = null;
DateTime? b = null;
a == b;   // true (both null)

DateTime? c = DateTime.Now;
a < c;    // false (null < anything = false)
a > c;    // false (null > anything = false)
```

**Механизм:** `>`, `<` с null operand — всегда false. Не intuitive.

### 11.6. Boxing nullable как `Nullable<T>`

```csharp
int? x = null;
object o = x;
o is int? n;     // ⚠ pattern matching tricky
```

**Механизм:** boxes как underlying type, не `Nullable<T>`.

### 11.7. NRT не affect runtime

```csharp
public string GetName() => null!;   // suppress warning
var name = service.GetName();
name.Length;   // runtime NRE — NRT не помог!
```

**Механизм:** NRT compile-time only. Runtime НЕ checks.

### 11.8. Generic T? в .NET < 6

```csharp
public T? FindFirst<T>(IEnumerable<T> source)   // T? requires constraint
```

**Фикс:** добавь `where T : class` или `where T : struct`, или используй `[MaybeNull]`.

### 11.9. JsonSerializer + non-null

```csharp
public class Dto
{
    public string Name { get; set; } = "";   // non-null required
}

// JSON {} → deserialized с Name = null! (skipping default)
// → runtime NRE
```

**Фикс:** `required` (C# 11+) или validation после deserialize.

### 11.10. EF Core + NRT mismatch

```csharp
public class User
{
    public string Name { get; set; }   // ⚠ uninitialized — DB column allows null?
}
```

**Фикс:** explicit `string?` для nullable columns, `required` для NOT NULL columns.

> [!question]- Интервью: топ-3 ошибки с NRT?
> 1) **`= null!` без понимания** — suppress warning при declaration, но если field не заполнен в runtime — NRE. Используй constructor validation или `required`. 2) **`!` everywhere** для suppress warnings — symptomatic of design problem. Validate раз в entry point. 3) **NRT не affects runtime** — `!` не add safety. Все references могут быть null в runtime. NRT только compile-time hint, нужны runtime checks (`ArgumentNullException.ThrowIfNull`).

---

## 12. Practice exercises

### 12.1. Migration legacy → NRT

```csharp
// Legacy
public class UserService
{
    private DbContext _db;
    public User GetById(int id) => _db.Users.Find(id);
    public void Save(User user) => _db.Users.Add(user);
}

// NRT
public class UserService
{
    private readonly DbContext _db;
    
    public UserService(DbContext db)
    {
        ArgumentNullException.ThrowIfNull(db);
        _db = db;
    }
    
    public User? GetById(int id) => _db.Users.Find(id);   // may not exist
    
    public void Save(User user)
    {
        ArgumentNullException.ThrowIfNull(user);
        _db.Users.Add(user);
    }
}
```

### 12.2. TryGet pattern

```csharp
public bool TryGetUser(int id, [NotNullWhen(true)] out User? user)
{
    user = _users.GetValueOrDefault(id);
    return user is not null;
}

if (TryGetUser(42, out var u))
{
    Console.WriteLine(u.Name);   // compiler знает non-null
}
```

### 12.3. Required + init

```csharp
public class CreateOrderRequest
{
    public required int CustomerId { get; init; }
    public required List<int> ProductIds { get; init; }
    public string? Notes { get; init; }
    public DateTimeOffset? ScheduledFor { get; init; }
}

var request = new CreateOrderRequest
{
    CustomerId = 42,
    ProductIds = [1, 2, 3]
    // Notes, ScheduledFor — optional
};
```

---

## 13. Что читать дальше

1. **[[oop|OOP]]** — required properties, immutability.
2. **[[error-handling|Error Handling]]** — `ArgumentNullException.ThrowIfNull`.
3. **[[generics-deep|Generics deep]]** — generic nullability.
4. **[[modern-features|Modern Features]]** — primary constructors + nullable.
5. **JsonSerializer + NRT** — `required` для deserialization.

---

## 14. См. также

- [[oop|OOP]] — `required` properties
- [[error-handling|Error Handling]] — null validation
- [[generics-deep|Generics deep]] — generic nullability
- [[equality-comparison|Equality]] — null in equality
- System.Diagnostics.CodeAnalysis annotations
- C# language reference — Nullable

---

## 15. Reading list

- **Microsoft Docs — Nullable reference types** — learn.microsoft.com/dotnet/csharp/nullable-references
- **Microsoft Docs — `Nullable<T>`** — learn.microsoft.com/dotnet/api/system.nullable-1
- **Microsoft Docs — Null-conditional operators** — learn.microsoft.com/dotnet/csharp/language-reference/operators/member-access-operators
- **Microsoft Docs — Attributes for null-state analysis** — learn.microsoft.com/dotnet/csharp/language-reference/attributes/nullable-analysis
- **Mads Torgersen — NRT design rationale** (Microsoft blog)
- **Jon Skeet — C# in Depth (NRT chapter)**
- **Andrew Lock — NRT migration patterns** — andrewlock.net
- **Stephen Cleary — Async + nullable** — blog.stephencleary.com
