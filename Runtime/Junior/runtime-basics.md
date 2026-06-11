---
tags: [runtime, basics, junior, clr, jit, gc, managed-code]
level: Junior
date: 2026-05-10
---

# Runtime Basics — CLR, JIT, GC, managed code overview

> **Что происходит когда запускается C# программа: CLR, JIT compilation, GC, managed memory.** Bird's-eye view перед `Middle/threading-basics.md`, `gc-memory.md`, `compilation-jit.md`.

---

## 0. Как читать

Если только начинаешь .NET — раздел 1 (что такое managed runtime) → 2 (CLR) → 3 (compilation). После — 4 (GC overview), 5 (memory model basic). Глубокий dive в GC generations, JIT tiers, memory layouts — Senior файлы.

Cross-language якоря свёрнуты. Interview-вопросы встроены.

---

## 1. Managed vs Unmanaged code

### 1.1. Что такое managed code

```
Managed code — выполняется внутри runtime (CLR), который:
- Управляет памятью автоматически (Garbage Collector)
- Проверяет типы во время выполнения (type safety)
- Перехватывает exceptions
- Обеспечивает security (sandbox, permissions)

Examples managed:
- C# / VB.NET / F# (.NET)
- Java (JVM)
- Python (CPython)
- JavaScript (V8 / Node.js)

Unmanaged code — выполняется напрямую CPU:
- Manual memory management (malloc / free)
- No type safety
- Crash on memory bugs (segfaults)

Examples unmanaged:
- C / C++
- Rust (memory-safe но без runtime)
- Assembly
```

### 1.2. Trade-offs managed

```
✅ Pros:
- Memory leaks reduced (GC)
- No buffer overflows (bounds checked)
- Exceptions instead of crashes
- Cross-platform (одинаковый IL код запускается везде)
- Reflection и метаданные доступны

❌ Cons:
- Runtime overhead (memory + CPU для GC)
- GC pauses (миллисекунды)
- Slower startup (JIT compilation)
- Harder для real-time (hard latency requirements)
- Менее предсказуемое поведение performance
```

### 1.3. Когда managed — best choice

```
✅ Web applications (ASP.NET Core)
✅ Business apps (CRUD, workflows)
✅ Desktop apps (WPF, WinUI, MAUI)
✅ Mobile apps (.NET MAUI)
✅ Cloud services (microservices)
✅ Games scripting (Unity uses C#)

❌ Low-level system programming → C/C++/Rust
❌ Real-time control systems → C/Rust
❌ Bare-metal embedded → C/Assembly
❌ Hard latency requirements (HFT-критичные) → C++/Rust + careful tuning
```

> [!info]- Если ты приходил из Java
> Концепция идентична: JVM ↔ CLR. Bytecode (Java) ↔ IL (CIL). HotSpot JIT ↔ RyuJIT. G1 GC ↔ .NET GC. Generations такие же. Java garbage collector чуть отличается в типах (G1, ZGC, Shenandoah), но главные принципы одинаковые. Class loading в JVM ≈ assembly loading в CLR.

> [!info]- Если ты приходил из C/C++
> Главное отличие — **GC и нет указателей** (по default). Можно использовать `unsafe` блоки для pointers, но это редкая необходимость. **Memory** управляется автоматически — нет `malloc/free` / `new/delete` (на самом деле `new` есть, но `delete` нет — GC решает когда освобождать). **Trade-off**: меньше memory bugs (no leaks/UAF), но GC pauses + memory overhead.

> [!question]- Интервью: что такое managed code?
> Код, который выполняется внутри **runtime environment** (CLR для .NET), который управляет: 1) **Memory** (GC автоматически освобождает unused memory). 2) **Type safety** (runtime проверяет casts, bounds, etc.). 3) **Security** (sandbox, code access security). 4) **Exception handling** (structured exceptions). **Альтернатива**: unmanaged (C/C++) — runs directly на CPU, manual memory, faster но опаснее. **.NET specific**: C# code → IL (intermediate language) → JIT compiles в machine code at runtime → executes под управлением CLR.

---

## 2. CLR — Common Language Runtime

### 2.1. Что такое CLR

```
CLR = Common Language Runtime — engine, который выполняет .NET код.

Содержит:
- Class Loader (загружает assemblies)
- JIT Compiler (IL → machine code)
- Garbage Collector
- Type System (Common Type System, CTS)
- Security Manager
- Exception Handling
- Thread Manager
- Native interop layer
```

### 2.2. .NET Framework vs .NET Core vs .NET 5+

```
.NET Framework (legacy, до 4.8.1):
- Только Windows
- Поддержка до 2031 (LTS)
- НЕ для новых проектов

.NET Core 1.0-3.1:
- Cross-platform (Windows/Linux/macOS)
- Open-source
- Прародитель .NET 5+

.NET 5 / 6 / 7 / 8 / 9:
- Cross-platform, единая платформа
- LTS releases (6, 8, 10) — 3 года поддержки
- STS releases (5, 7, 9) — 18 месяцев

В 2024-2026: .NET 8 (LTS) и .NET 9 (STS) — current.
```

### 2.3. Что выбрать для нового проекта

```
✅ Новый проект → .NET 8 (LTS до Nov 2026) или .NET 9
✅ Library → multi-target (net8.0;net9.0)
❌ .NET Framework только для legacy maintenance

Migration .NET Framework → .NET 8:
- ~Большинство кода работает
- WCF: переходи на gRPC / ASP.NET Core
- WebForms: переписывать (нет direct замены)
- WPF / WinForms: работают на .NET 8 (Windows only)
```

### 2.4. Versions runtime в системе

```bash
# Список installed runtimes
dotnet --list-runtimes

# Output (примерно):
# Microsoft.AspNetCore.App 8.0.0 [...]
# Microsoft.AspNetCore.App 9.0.0 [...]
# Microsoft.NETCore.App 8.0.0 [...]
# Microsoft.NETCore.App 9.0.0 [...]
# Microsoft.WindowsDesktop.App 8.0.0 [...]   (только Windows)
```

### 2.5. Side-by-side runtimes

Можно иметь несколько versions installed одновременно. App выбирает требуемую через `<TargetFramework>` в csproj.

```xml
<TargetFramework>net8.0</TargetFramework>
```

> [!question]- Интервью: что такое CLR?
> **Common Language Runtime** — execution engine для .NET. Когда запускаешь C# программу, CLR: 1) **Loads assembly** (DLL/EXE с IL код). 2) **JIT compiles** IL в machine code (just-in-time, при первом вызове метода). 3) **Manages memory** через Garbage Collector. 4) **Enforces type safety**. 5) **Handles exceptions**. 6) **Manages threads**. **Cross-language**: одна CLR работает с C#, F#, VB.NET — все компилируются в IL. **.NET 5+**: CoreCLR — open-source cross-platform реализация.

---

## 3. Compilation — как C# становится executable

### 3.1. Two-step compilation

```
Step 1: C# compiler (Roslyn, csc.exe / dotnet build)
- Source code (.cs files) → Intermediate Language (IL)
- IL — platform-independent bytecode
- Saved в Assembly (.dll или .exe)
- Метаданные (типы, methods) embedded

Step 2: JIT compiler (at runtime)
- IL → native machine code (x64 / ARM64)
- Происходит когда method вызывается first time
- Cached в memory (subsequent calls — direct machine code)
- Optimized для current CPU
```

```
.cs source → C# compiler → .dll (IL) → JIT → machine code
                                              (выполняется CPU)
```

### 3.2. Что в .dll файле

```bash
# Inspect using ildasm (Visual Studio) или Reflector / ILSpy
# Содержит:
- IL code (instructions)
- Metadata (types, methods, fields, attributes)
- References (другие assemblies)
- Resources (embedded strings, images)
- Entry point (для .exe)
```

### 3.3. Decompile .dll → C#

```bash
# Установить ilspycmd
dotnet tool install -g ilspycmd

# Decompile assembly обратно в C# (approximation)
ilspycmd MyApp.dll
```

Полезно для debugging third-party packages, понимания framework internals.

### 3.4. AOT — Ahead-Of-Time compilation

```
Default JIT:
- IL ships в DLL
- Compile в machine code at runtime
- Pros: smaller binary, optimized for current CPU
- Cons: slower startup (cold JIT)

AOT (Native AOT, .NET 7+):
- IL compiled в machine code заранее (build time)
- Ships native executable (без runtime!)
- Pros: instant startup, smaller deployments, no runtime needed
- Cons: bigger binary file, no reflection, no runtime code generation
- Use cases: serverless (cold start), CLI tools, embedded
```

```bash
# Build Native AOT
dotnet publish -c Release -r linux-x64 \
    -p:PublishAot=true \
    -p:StripSymbols=true
```

### 3.5. Tiered compilation

`.NET 6+` использует **tiered JIT**:

```
Tier 0: Quick JIT (no optimization)
- Method called first time
- Quick compile, slow execution
- ~100ms first call

Tier 1: Optimized JIT
- Method called frequently (~30 times typically)
- Heavy optimization
- Subsequent calls fast

Result:
- Cold start fast (Tier 0 quick compile)
- Steady state fast (Tier 1 optimized)
- Best of both worlds
```

> [!question]- Интервью: что такое JIT compilation?
> **Just-In-Time** compilation — преобразование IL (intermediate language) в native machine code **во время выполнения**, а не во время build. **Workflow**: 1) C# → IL при build (saved в DLL). 2) При запуске CLR loads DLL. 3) Когда method called first time — JIT compiles в machine code. 4) Результат cached. 5) Subsequent calls — direct execution. **Pros**: optimized для current CPU (SSE/AVX availability), platform-independent IL. **Cons**: slow cold start (compilation). **Tiered JIT** (.NET 6+): Tier 0 (quick) + Tier 1 (optimized) — балансирует startup speed и runtime performance. **Альтернатива**: Native AOT — compiles ahead of time, no JIT, instant startup.

---

## 4. Garbage Collector — overview

### 4.1. Что делает GC

```
GC = автоматический memory management.

Без GC (C/C++):
- Programmer выделяет memory: malloc / new
- Programmer освобождает: free / delete
- Забыл free → memory leak
- Free несколько раз → crash
- Use after free → undefined behavior

С GC (.NET / Java):
- Programmer выделяет: new
- GC отслеживает использование
- GC автоматически освобождает unused
- No leaks (большинство случаев), no UAF, no double-free
```

### 4.2. Когда GC запускается

```
Триггеры GC:
1. Allocation pressure — generation 0 заполнилось
2. Manual: GC.Collect() (редко нужен)
3. Memory pressure из OS
4. После значительного количества allocations

GC решает САМ когда — programmer не контролирует.
```

### 4.3. Generations — bird's eye view

```
.NET GC использует generational hypothesis:
"Большинство объектов умирают молодыми"

Gen 0: новые объекты
- Маленькая (~MB)
- Часто collected (быстро)
- Большинство объектов умирают здесь

Gen 1: survived Gen 0
- Mid-size
- Less frequent collection

Gen 2: long-lived objects
- Большая
- Редкие collections (slow)
- Tenured objects (cached data, singletons)

LOH (Large Object Heap):
- Объекты > 85,000 bytes
- Collected с Gen 2
- НЕ компактится по default (fragmentation risk)
```

См. `Runtime/gc-memory.md` для deep treatment.

### 4.4. GC pause

```
Workstation GC (default для desktop apps):
- Concurrent (parallel с user code)
- Short pauses (~ms)
- Low latency приоритет

Server GC (default для ASP.NET Core):
- Multi-threaded GC
- Higher throughput
- Slightly longer pauses

Real-time apps (HFT, gaming):
- Server GC + tuning
- Object pooling
- Avoid allocations в hot path
```

### 4.5. Что делать чтобы помочь GC

```
✅ Избегай unnecessary allocations:
- Reuse объектов (pooling)
- Используй structs для small data
- StringBuilder для concatenation
- Span<T>/Memory<T> вместо arrays для slicing

✅ Избегай large object pinning
✅ Используй using/IDisposable для unmanaged resources
✅ Не вызывай GC.Collect() вручную (бесполезно почти всегда)

❌ Не оптимизируй GC преждевременно
❌ Не предполагай поведение GC — measure!
```

### 4.6. Disposable pattern

```csharp
// Для unmanaged resources (file handles, DB connections, sockets)
using var connection = new SqlConnection(connStr);
connection.Open();
// ... work ...
// Auto-dispose в конце block (вызывает Close internally)

// Без using — нужен manual dispose
var connection = new SqlConnection(connStr);
try
{
    connection.Open();
    // ... work ...
}
finally
{
    connection.Dispose();
}
```

`using` гарантирует что Dispose вызывается даже при exception. Освобождает unmanaged resources которые GC сам не освободит.

> [!question]- Интервью: что такое GC и зачем generations?
> **Garbage Collector** — компонент CLR, автоматически освобождающий память от неиспользуемых объектов. **Generational GC** основан на observation: **большинство объектов умирают молодыми** (locals в methods, temp objects). **3 поколения**: Gen 0 (новые, ~MB, частые collections), Gen 1 (выжившие в Gen 0), Gen 2 (long-lived, редкие collections, slow). **Преимущество**: вместо scanning всей heap каждый раз — scan только Gen 0 (small + fast). **LOH** — отдельная heap для объектов > 85K bytes (не компактится по default). **Trade-off**: GC pauses (ms-scale), memory overhead. **Помощь GC**: avoid allocations, pooling, structs для small data.

---

## 5. Memory model — basic

### 5.1. Stack vs Heap

```
Stack:
- Каждый thread имеет свой stack (~1 MB)
- LIFO — last in, first out
- Хранит:
  - Local variables (value types)
  - Method parameters
  - Return addresses
- Auto-cleanup при return из method
- Очень быстрый (microseconds)

Heap:
- Shared между threads
- Хранит:
  - Reference types (objects, arrays, strings)
  - Boxed value types
- Managed by GC
- Allocations медленнее stack (~10x)
- Cleanup автоматический (GC)
```

### 5.2. Value types vs Reference types

```csharp
// Value types — на stack (если локальная) или inline в containing object
struct Point
{
    public int X;
    public int Y;
}

int x = 42;          // value type — на stack
Point p = new(1, 2); // value type — на stack
DateTime now = DateTime.Now;   // value type

// Reference types — на heap, на stack reference (pointer)
class User
{
    public string Name;
    public int Age;
}

User user = new();   // user — pointer на stack, объект на heap
string name = "Alice";   // string — reference type, на heap
List<int> list = new();  // collections — reference, на heap
```

### 5.3. Boxing — value type → reference

```csharp
int x = 42;              // value type, on stack
object obj = x;          // boxing — копия в heap, obj указывает туда
int y = (int)obj;        // unboxing — копия из heap обратно в stack

// ❌ Boxing в hot path = performance killer
List<object> nums = new();
for (int i = 0; i < 1000000; i++)
{
    nums.Add(i);   // boxing каждый раз!
}

// ✅ Generic List<int>
List<int> nums = new();
for (int i = 0; i < 1000000; i++)
{
    nums.Add(i);   // no boxing
}
```

### 5.4. Strings — special

```csharp
string s = "Hello";        // string — reference type
string s2 = "Hello";       // s и s2 указывают на тот же object (string interning)
bool same = ReferenceEquals(s, s2);   // true!

// Strings IMMUTABLE
string a = "Hello";
string b = a + " World";   // создаёт NEW string, a не меняется
// Many string concatenations → many allocations

// Use StringBuilder для multiple concatenations
var sb = new StringBuilder();
for (int i = 0; i < 100; i++) sb.Append($"line {i}\n");
string result = sb.ToString();
```

### 5.5. Thread stack visualization

```
Thread A stack (top → bottom):
┌──────────────────┐
│ Method3 frame    │  ← currently executing
│   local vars     │
├──────────────────┤
│ Method2 frame    │
│   local vars     │
├──────────────────┤
│ Method1 frame    │
│   local vars     │
└──────────────────┘
       ↑
   Stack base

Heap (shared):
┌──────────────────┐
│ User object      │  ← Method3 has reference to this
│ List<int>        │  ← Method1 has reference
│ string "Alice"   │
│ ...              │
└──────────────────┘
```

> [!question]- Интервью: разница value type и reference type?
> **Value types**: содержат данные **прямо** (struct, int, bool, DateTime, enum). На **stack** если локальные, **inline** если поле объекта. Copy by value (independent copies). **Reference types**: содержат **ссылку** на heap (class, string, array, delegate). Reference на stack, объект на heap. Copy by reference (несколько вариаций указывают на тот же объект). **Performance**: value types быстрее (no GC, no allocation в heap). **Boxing**: value type → reference type через `object` cast — копия в heap, slow. **Best practice**: structs для small immutable data (Point, Color), classes для everything else.

---

## 6. Threads — bird's eye view

### 6.1. Thread basics

```
Process — running instance of program
Thread — execution unit внутри process
- Один process может иметь много threads
- Threads share memory с process
- Каждый thread имеет свой stack (~1 MB)
- OS schedules threads на CPU cores
```

### 6.2. ThreadPool

```
.NET ThreadPool — pre-allocated threads, переиспользуются
- Создание thread медленно (~ms)
- Pool имеет ~ N (где N = CPU cores) workers
- Tasks (Task) выполняются в pool
- Async/await uses pool под капотом
```

### 6.3. Простой пример

```csharp
// Запустить work в ThreadPool
Task.Run(() =>
{
    Console.WriteLine($"Running on thread {Environment.CurrentManagedThreadId}");
});

// Несколько parallel
await Task.WhenAll(
    Task.Run(() => Compute1()),
    Task.Run(() => Compute2()),
    Task.Run(() => Compute3())
);
```

### 6.4. Async ≠ Threading

```
Async/await:
- НЕ обязательно создаёт thread
- Releases thread во время I/O wait
- Один thread обслуживает много async operations

Threading:
- Создаёт OS threads
- Блокирует thread (Thread.Sleep, etc.)
- Параллельное выполнение CPU-bound work

Common confusion:
- "Async код параллельный" — НЕ обязательно
- "Параллельный — значит async" — нет
```

См. deep treatment в `Middle/threading-basics.md` и `Senior/async-threading.md`.

> [!question]- Интервью: что такое ThreadPool и зачем?
> **ThreadPool** — pre-allocated коллекция OS threads, которые переиспользуются для tasks. **Зачем**: создание thread медленно (выделение stack, kernel allocation). Reuse threads — fast. **Default size**: основано на CPU cores. **Used by**: Task.Run, async/await continuations, ASP.NET Core request handling. **Min/Max threads**: configurable через `ThreadPool.SetMinThreads`. **Alternative**: создавать `new Thread()` вручную (rare, для long-running dedicated threads). **Best practice 2024+**: используй `Task.Run` / `async/await` (uses pool автоматически), manually `Thread` создаваешь только в специальных cases.

---

## 7. Common pitfalls

### 7.1. Manual GC.Collect()

```csharp
// ❌ Бесполезно почти всегда
GC.Collect();
GC.WaitForPendingFinalizers();
GC.Collect();
```

**Фикс**: GC sам решает когда. Manual collection обычно делает производительность ХУЖЕ. Use case только: после mass deletion в long-running app.

### 7.2. Boxing в hot path

```csharp
List<object> numbers = new();
for (int i = 0; i < 1_000_000; i++)
    numbers.Add(i);   // ❌ Million boxings = million allocations
```

**Фикс**: `List<int>`.

### 7.3. String concatenation в loop

```csharp
string result = "";
for (int i = 0; i < 1000; i++)
    result += $"Item {i}\n";   // ❌ 1000 new strings, GC pressure
```

**Фикс**: StringBuilder.

### 7.4. Не using для disposable

```csharp
var conn = new SqlConnection(connStr);
conn.Open();
// ... work ...
// ❌ Забыл Dispose — connection остаётся открытой
```

**Фикс**: `using var conn = new SqlConnection(connStr);`.

### 7.5. Pinning большие arrays

```csharp
var hugeArray = new byte[100 * 1024 * 1024];   // 100 MB
// ... передаёшь в native code with fixed { ... }
// ❌ Pinned array нельзя двигать → fragmentation в LOH
```

**Фикс**: short-lived pinning, или unmanaged buffers (`Marshal.AllocHGlobal`).

### 7.6. Allocations в Unity / game loop

```csharp
void Update()
{
    var list = new List<Enemy>();   // ❌ Allocation каждый frame!
    foreach (var e in enemies)
        if (e.IsActive) list.Add(e);
}
```

**Фикс**: cache list, reuse.

### 7.7. Latency expectations

```csharp
// Database call — sync
public void GetUser(int id)
{
    var user = _db.Users.Find(id);
    // ❌ Blocks thread на 50-200ms
    // В ASP.NET Core это thread pool starvation если много users
}
```

**Фикс**: async/await:

```csharp
public async Task<User> GetUserAsync(int id) =>
    await _db.Users.FindAsync(id);
```

### 7.8. Не понимать tiered compilation

```csharp
// Benchmark в Debug mode → unrealistic results
// JIT не optimized, GC otro behavior
```

**Фикс**: всегда benchmark в Release с warmup. Используй BenchmarkDotNet.

### 7.9. .NET Framework для нового проекта

```xml
<TargetFramework>net48</TargetFramework>   <!-- ❌ Legacy -->
```

**Фикс**: `<TargetFramework>net8.0</TargetFramework>` для нового кода.

### 7.10. Старые JIT assumptions

Старые посты ("StringBuilder быстрее string +") могут быть outdated в .NET 8+. JIT очень умный.

**Фикс**: measure (BenchmarkDotNet) — не предполагай.

> [!question]- Интервью: топ-3 ошибки начинающих с runtime?
> 1) **GC.Collect() вручную** — бесполезно почти всегда, делает только хуже. GC сам знает когда. 2) **Boxing в hot path** — `List<object>` для int values, миллион allocations. Fix: `List<int>` (generic). 3) **Sync I/O в ASP.NET Core** — blocks thread pool worker, starves сервер. Fix: async/await everywhere. **Bonus**: не using для IDisposable — connection leaks. **Bonus 2**: .NET Framework для нового проекта — выбирай .NET 8+.

---

## 8. Cheat sheet

```csharp
// === Runtime check ===
Console.WriteLine(Environment.Version);            // CLR version
Console.WriteLine(RuntimeInformation.FrameworkDescription);   // .NET 8.0.0
Console.WriteLine(RuntimeInformation.OSDescription);          // OS info
Console.WriteLine(Environment.ProcessorCount);     // CPU cores
Console.WriteLine(GC.GetTotalMemory(false));        // current heap usage
Console.WriteLine(GC.GetGCMemoryInfo().HeapSizeBytes);   // total heap

// === Disposable pattern ===
using var connection = new SqlConnection(connStr);
using var stream = File.Open("file.txt", FileMode.Open);

// === Async I/O (don't block) ===
var data = await client.GetStringAsync(url);
var user = await _db.Users.FindAsync(id);

// === Avoid boxing ===
List<int> nums = new();         // ✅ generic
List<object> nums = new();       // ❌ boxing

// === Avoid string concat в loop ===
var sb = new StringBuilder();
for (int i = 0; i < 1000; i++) sb.Append($"line {i}\n");

// === Force GC (rare) ===
GC.Collect();                    // ⚠️ only if real reason
GC.WaitForPendingFinalizers();
GC.Collect();

// === Performance ===
var sw = Stopwatch.StartNew();
// ... work ...
sw.Stop();
Console.WriteLine($"{sw.ElapsedMilliseconds}ms");

// === Thread info ===
Console.WriteLine(Environment.CurrentManagedThreadId);
Console.WriteLine(Thread.CurrentThread.IsThreadPoolThread);

// === Memory pressure ===
GC.AddMemoryPressure(1024 * 1024);    // tell GC about unmanaged 1MB
GC.RemoveMemoryPressure(1024 * 1024);
```

---

## 9. Practice exercises

### 9.1. Inspect runtime info

Создай console app, выведи:
- .NET version
- Operating System
- CPU cores
- Initial GC memory
- Allocate 100 MB array
- Print GC memory before/after
- Force GC, print again
- Что заметил? Освобождается?

### 9.2. Boxing benchmark

Сравни через BenchmarkDotNet:

```csharp
[Benchmark]
public int BoxedSum()
{
    List<object> list = new();
    for (int i = 0; i < 10000; i++) list.Add(i);
    int sum = 0;
    foreach (var item in list) sum += (int)item;
    return sum;
}

[Benchmark]
public int GenericSum()
{
    List<int> list = new();
    for (int i = 0; i < 10000; i++) list.Add(i);
    int sum = 0;
    foreach (var item in list) sum += item;
    return sum;
}
```

Сколько раз медленнее BoxedSum? (Spoiler: ~5-10x + memory overhead)

### 9.3. String vs StringBuilder

Сравни:

```csharp
[Benchmark]
public string ConcatString()
{
    string result = "";
    for (int i = 0; i < 1000; i++) result += $"Item {i}\n";
    return result;
}

[Benchmark]
public string StringBuilder()
{
    var sb = new StringBuilder();
    for (int i = 0; i < 1000; i++) sb.Append($"Item {i}\n");
    return sb.ToString();
}
```

Сравни: время + allocated bytes. (StringBuilder: ~10-50x faster, ~100x less memory).

---

## 10. Что читать дальше

1. **`Runtime/Middle/threading-basics.md`** — Thread, ThreadPool, TPL, Parallel
2. **`Runtime/gc-memory.md`** — GC generations deep
3. **`Runtime/concurrency-atomics.md`** — atomic operations, locks
4. **`Runtime/span-layout.md`** — `Span<T>`/`Memory<T>`, memory layouts
5. **`Runtime/compilation-jit.md`** — JIT tiers, optimization
6. **`CSharp/Senior/async-threading.md`** — async/await deep
7. **`CSharp/Senior/types-and-memory.md`** — value/reference, GC

---

## 11. См. также

- [[gc-memory|Runtime/gc-memory]] — GC deep
- [[threading-basics|Runtime/Middle/threading-basics]] — threading deep
- [[span-layout|Runtime/span-layout]] — memory layouts
- [[compilation-jit|Runtime/compilation-jit]] — JIT
- [[async-threading|CSharp/Senior/async-threading]] — async deep
- [[types-and-memory|CSharp/Senior/types-and-memory]] — types deep
- [[performance-fundamentals|Performance/Junior/performance-fundamentals]] — performance basics

---

## 12. Reading list

- **Microsoft Docs — .NET Architecture** — learn.microsoft.com/dotnet/standard/clr
- **Microsoft Docs — Garbage Collection** — learn.microsoft.com/dotnet/standard/garbage-collection/
- **Microsoft Docs — JIT Compilation** — learn.microsoft.com/dotnet/standard/managed-execution-process
- **"CLR via C#" — Jeffrey Richter** (book, deep but classic)
- **"Pro .NET Memory Management" — Konrad Kokosa** (book, deep)
- **.NET Performance blog** — devblogs.microsoft.com/dotnet/category/performance
- **Stephen Toub — Performance Improvements .NET** — devblogs.microsoft.com (annual posts)
- **dotnet/runtime GitHub** — github.com/dotnet/runtime (source code)
