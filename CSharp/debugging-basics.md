---
tags: [csharp, debugging, junior, visual-studio, breakpoints, debugger, tools]
level: Junior
date: 2026-04-30
---

# Отладка — debugging для Junior

> **Как находить баги в C# коде**: breakpoints, watch, call stack, immediate window, exception handling в debug, логирование. Closes пробел "знаю что есть отладчик, но реально использую только Console.WriteLine".

---

## Что это, зачем и когда

### Зачем уметь отлаживать

Самый частый ответ Junior на bug: `Console.WriteLine` везде. Это работает, но **очень медленно**. Senior находит bug за 5 минут вместо 50 потому что **умеет debugger**.

```
Console.WriteLine отладка:           Debugger:
─────────────────                     ─────────────────
1. Добавил Console.WriteLine          1. Поставил breakpoint
2. Запустил                           2. Запустил
3. Не туда — изменил                  3. Hover на переменной — видишь value
4. Запустил снова                     4. F10 — следующая строка
5. Не помогло — добавил ещё            5. Bug ясен
6. Forgot remove перед commit          
```

### Что должен уметь Junior

После этого файла:
- Ставить breakpoint и понимать его
- Использовать **F5 / F10 / F11** (Continue / Step Over / Step Into)
- **Hover** на переменной чтобы увидеть значение
- **Watch window** для отслеживания expressions
- **Immediate window** для quick eval
- **Conditional breakpoints**
- **Call stack** — откуда я попал сюда?
- **Exception breakpoints**
- Базовое **logging** через `ILogger`

---

## 1. Breakpoints — основа

### Как поставить

В Visual Studio / VS Code / Rider:

```csharp
public int Calculate(int a, int b)
{
    int sum = a + b;        // ← клик слева от строки = красный кружок (breakpoint)
    int product = a * b;
    return sum + product;
}
```

Когда выполнение дойдёт до этой строки — **остановится**, IDE покажет state.

### Hotkey

| IDE | Toggle Breakpoint |
|-----|-------------------|
| Visual Studio | `F9` |
| VS Code | `F9` |
| Rider | `Ctrl+F8` |

### Что делать когда остановился

```
F5     → Continue (продолжить до следующего breakpoint)
F10    → Step Over (выполнить строку, не заходя внутрь методов)
F11    → Step Into (зайти внутрь метода)
Shift+F11 → Step Out (выполнить остаток метода и выйти)
```

### Hover — вижу значение

```csharp
int x = CalculateSomething();   // ← breakpoint
//   ↑ навёл мышью на x → показывает текущее значение
```

В Visual Studio — целое **DataTip** окно с возможностью раскрыть property у объектов.

---

## 2. Case Study #1 — Off-by-one bug

### Сценарий

Метод считает сумму первых N элементов. Возвращает неправильное значение.

```csharp
public int SumFirstN(int[] arr, int n)
{
    int sum = 0;
    for (int i = 0; i <= n; i++)  // bug здесь
    {
        sum += arr[i];
    }
    return sum;
}

// Test
var nums = new[] { 1, 2, 3, 4, 5 };
SumFirstN(nums, 3);  // Ожидаем 1+2+3 = 6, но получаем 1+2+3+4 = 10!
```

### ❌ Junior подход — print debugging

```csharp
public int SumFirstN(int[] arr, int n)
{
    int sum = 0;
    for (int i = 0; i <= n; i++)
    {
        Console.WriteLine($"i={i}, arr[i]={arr[i]}, sum={sum}");  // print
        sum += arr[i];
    }
    Console.WriteLine($"Final sum={sum}");
    return sum;
}
```

Работает, но: засоряет код, нужно убирать перед commit, не покажешь struct'ы детально.

### ✅ Debugger подход

1. **Breakpoint** на `for (int i = 0; i <= n; i++)`
2. **F5** — запуск
3. Hover на `n` — видишь `n=3`
4. **F10** — войти в loop
5. Hover на `i` — `i=0`. Hover на `arr[i]` — `1`
6. **F10** — `i=1`, `arr[i]=2`, `sum=1`
7. ... продолжай F10 → видишь что i доходит до 3 и **дополнительная итерация** (i=4 ещё проходит). Bug найден — должно быть `i < n`, не `i <= n`!

**5 секунд** vs print debugging.

---

## 3. Watch window — отслеживать expressions

### Как использовать

Открыть **Watch window** (Visual Studio: `Ctrl+Alt+W, 1`).

Добавь expression в Watch:

```
arr.Length        → 5
arr[i]             → 4 (текущее значение)
sum + arr[i]       → 10 (что будет после операции)
i < arr.Length     → True
```

При каждом step — **значения автоматически обновляются**.

### Можно даже вычислять complex expressions

```
arr.Where(x => x > 2).Sum()    → 12 (LINQ работает!)
n.ToString("X")                 → "3" (формат)
DateTime.Now.Hour                → 14
```

---

## 4. Immediate Window — quick eval

В Visual Studio: `Ctrl+Alt+I`. Можно выполнять C# код прямо во время debug.

```
> arr.Sum()
15

> arr.Where(x => x > 2).ToArray()
{int[3]} { 3, 4, 5 }

> n = 4    // даже изменить переменную!
4
```

**Useful когда:**
- Quick test того что предполагаешь
- Не хочется добавлять Watch
- Изменить переменную чтобы проверить другую ветку

---

## 5. Case Study #2 — NullReferenceException

### Сценарий

Throws `NullReferenceException` где-то в коде.

```csharp
public string FormatUser(User user)
{
    return $"{user.Name} <{user.Email.Trim()}>";  // throws здесь
}

// Stack trace:
// System.NullReferenceException: Object reference not set to an instance of an object.
//    at FormatUser(User user) line 5
```

### Что не понятно из стека

`user.Name` или `user.Email`? Stack trace не говорит.

### Debugger подход

1. **Breakpoint** на проблемной строке
2. F5 → останавливается
3. Hover на `user` — видишь объект, expand
4. Видишь: `user.Name = "John"`, **`user.Email = null`**
5. Bug ясен — **где-то выше Email не установлен**

### Conditional breakpoint — для редких случаев

Bug проявляется только на user_id = 42. Не хочется F5 миллион раз.

1. Right-click на breakpoint → **Conditions**
2. Условие: `user.Id == 42`
3. Breakpoint сработает **только** когда условие true

```csharp
public string FormatUser(User user)  // ← conditional breakpoint
{
    return $"{user.Name} <{user.Email.Trim()}>";
}
```

---

## 6. Call Stack — откуда я попал сюда?

### Сценарий

Метод `Calculate()` вызывается из 5 разных мест. В одном — bug, но какое именно?

В Visual Studio: **Call Stack window** (`Ctrl+Alt+C`).

```
Calculate(int x)         ← here, breakpoint
ProcessOrder(Order o)
HandleRequest(Request r)
ApiController.Post()
[External Code]
```

**Click на любую строку** — IDE покажет state в той функции, можно проверить параметры.

### Frame variables

В каждом фрейме видны **локальные переменные**. Можно понять — какие данные передавались.

---

## 7. Exception Settings — break on exception

### Сценарий

Где-то в коде throws exception. Catch блок проглатывает её. Хочешь точно видеть **где** throws.

В Visual Studio: **Debug → Windows → Exception Settings** (`Ctrl+Alt+E`).

```
[ ] Common Language Runtime Exceptions
    [✓] System.NullReferenceException
    [✓] System.ArgumentException
    [ ] System.IO.FileNotFoundException
```

Включаешь нужный exception → debugger остановится в момент **throw**, до catch.

```csharp
try
{
    var data = LoadData();  // throws NRE здесь
}
catch (Exception)
{
    // Без exception settings — debugger пропустит throw, ты не узнаешь
}
```

### `Debug.Assert` для invariants

```csharp
public void Process(int[] data)
{
    Debug.Assert(data != null, "data must not be null");
    Debug.Assert(data.Length > 0, "data must have items");
    
    // ...
}

// В Debug build — assertion failure → debugger останавливается
// В Release — Assert удалён компилятором (zero cost)
```

---

## 8. Case Study #3 — Loop bug в LINQ

### Сценарий

```csharp
public List<string> GetActiveUserEmails(List<User> users)
{
    var result = users
        .Where(u => u.IsActive)
        .Select(u => u.Email)
        .ToList();
    return result;
}

// Возвращает empty list, хотя users не пустой
```

LINQ — одна строка. Console.WriteLine не помогает.

### Debugger подход

**LINQ Visualizer** (Visual Studio Premium / 2022 Community):

1. Breakpoint на `.ToList()` строке
2. F10 — после её выполнения
3. Hover на `result` → клик на иконку **LINQ Results Visualizer**
4. Видишь intermediate steps: какие `users` входят, что фильтрует Where, что выбирает Select.

Without visualizer — разбиваешь chain:

```csharp
public List<string> GetActiveUserEmails(List<User> users)
{
    var active = users.Where(u => u.IsActive).ToList();  // ← breakpoint
    // Hover на active → проверь Count
    
    var emails = active.Select(u => u.Email).ToList();    // ← breakpoint
    // Hover на emails
    
    return emails;
}
// Найдёшь где данные теряются
```

---

## 9. Logging — для production debugging

В production debugger недоступен. Замена — **structured logging**.

### `ILogger<T>` — встроенно в .NET

```csharp
public class UserService
{
    private readonly ILogger<UserService> _logger;
    
    public UserService(ILogger<UserService> logger) => _logger = logger;
    
    public async Task<User?> GetById(int id)
    {
        _logger.LogInformation("Loading user {UserId}", id);
        
        try
        {
            var user = await _repo.FindAsync(id);
            
            if (user == null)
            {
                _logger.LogWarning("User {UserId} not found", id);
                return null;
            }
            
            _logger.LogDebug("User loaded: {UserName}", user.Name);
            return user;
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Failed to load user {UserId}", id);
            throw;
        }
    }
}
```

### Log levels

```
LogTrace        — самый детальный, все calls
LogDebug        — debug detail, для разработки
LogInformation  — нормальные events приложения
LogWarning      — подозрительно, но не error
LogError        — что-то сломалось
LogCritical     — приложение падает / critical failure
```

В production обычно: **Information** + выше. Debug/Trace — только в development.

### Structured arguments

```csharp
// ❌ String concatenation
_logger.LogInformation($"User {userId} loaded {count} items");
// В Seq/ELK — search by userId не работает!

// ✅ Structured
_logger.LogInformation("User {UserId} loaded {Count} items", userId, count);
// В Seq можно искать: "UserId = 42 AND Count > 100"
```

См. [[../AspNetCore/logging-observability|Logging & Observability]].

---

## 10. Case Study #4 — Production bug без debugger

### Сценарий

В production иногда `Order.Total` приходит как 0. У тебя 100 logs/сек — найти когда именно невозможно через тоновые grep.

### Решение — structured logging + correlation ID

```csharp
public class OrderProcessor
{
    public async Task<Order> Process(OrderRequest req)
    {
        var correlationId = Guid.NewGuid();
        
        using (_logger.BeginScope(new { CorrelationId = correlationId, OrderId = req.OrderId }))
        {
            _logger.LogInformation("Processing started, items: {ItemCount}", req.Items.Count);
            
            decimal total = 0;
            foreach (var item in req.Items)
            {
                total += item.Quantity * item.Price;
                _logger.LogDebug("Item {ItemId}: qty={Qty}, price={Price}, running={Total}",
                    item.Id, item.Quantity, item.Price, total);
            }
            
            if (total == 0)
            {
                _logger.LogWarning("Zero total! Items: {@Items}", req.Items);  // @ — serialize object
            }
            
            return new Order { Total = total, /* ... */ };
        }
    }
}
```

В **Seq** или **ELK**:
1. Search `Total: 0` → находишь все zero-total events
2. Берёшь `CorrelationId` → видишь весь scope
3. Видишь конкретные items — bug ясен

Это **production debugging** — без debugger, через logs.

См. [[../Infrastructure/observability|Observability]].

---

## 11. Performance debugging — `Stopwatch`

### Когда нужно

Метод медленный, не понятно где bottleneck.

```csharp
public async Task<Result> ComplexOperation(Request req)
{
    var sw = Stopwatch.StartNew();
    
    var data = await LoadData(req);
    _logger.LogDebug("LoadData: {Ms} ms", sw.ElapsedMilliseconds);
    sw.Restart();
    
    var processed = Transform(data);
    _logger.LogDebug("Transform: {Ms} ms", sw.ElapsedMilliseconds);
    sw.Restart();
    
    var result = await SaveResult(processed);
    _logger.LogDebug("SaveResult: {Ms} ms", sw.ElapsedMilliseconds);
    
    return result;
}

// Output (logs):
// LoadData: 50 ms
// Transform: 5 ms
// SaveResult: 3500 ms  ← bottleneck!
```

> [!info] Profiler — для serious cases
> Stopwatch для quick check. Если нужен deep performance analysis — **dotTrace, PerfView, dotnet-trace**. См. [[../Performance/performance|Performance Tools]].

---

## 12. Edit and Continue (Visual Studio)

В debug mode можно **изменить код** и **продолжить** без рестарта:

1. Breakpoint
2. F5 → остановился
3. Изменил код (например fix bug)
4. F5 → продолжается с **новым кодом**

**Ограничения:**
- Только в Debug build
- Не работает для async methods (некоторые случаи)
- Не работает после изменения method signature

Огромная экономия времени для long-running apps.

---

## 13. DebuggerDisplay — кастомный display в Watch

### Без атрибута

```csharp
public class User
{
    public int Id { get; set; }
    public string Name { get; set; }
}

// В Watch видишь: {Namespace.User} — нужно expand
```

### С атрибутом

```csharp
[DebuggerDisplay("User {Id}: {Name}")]
public class User
{
    public int Id { get; set; }
    public string Name { get; set; }
}

// В Watch: User 42: John  — сразу видно!
```

Полезно для собственных complex types.

### `DebuggerBrowsable` — скрыть детали

```csharp
public class Logger
{
    [DebuggerBrowsable(DebuggerBrowsableState.Never)]
    private readonly object _internalSyncRoot = new();
    
    [DebuggerBrowsable(DebuggerBrowsableState.RootHidden)]
    private List<LogEntry> _entries = new();
    
    public LogEntry[] Entries => _entries.ToArray();
}
```

---

## 14. Case Study #5 — Memory leak

### Сценарий

App работает 12 часов → memory растёт до 8 GB → OOM.

### Что делать Junior

1. **Open Diagnostic Tools** в Visual Studio (Debug → Windows → Diagnostic Tools)
2. Графики Memory + CPU в realtime
3. **Take Snapshot** периодически
4. **Compare snapshots** — видишь какие объекты растут

```
Snapshot 1 (start):     50 MB
Snapshot 2 (1h):        200 MB  ← +150 MB
Snapshot 3 (2h):        400 MB  ← +200 MB

Compare 2 vs 1:
+ 30,000 instances of CachedItem
+ 20,000 instances of byte[]
```

Bug ясен — где-то добавляешь в кеш и не убираешь.

> [!info] Production memory leak
> Используй `dotnet-dump` или `dotnet-counters` для production. См. [[../Runtime/diagnostics-tools|Diagnostics Tools]] и [[../Performance/memory-profiling|Memory Profiling]].

---

## 15. Common Pitfalls

### 1. Console.WriteLine навсегда

```csharp
public int Calculate(int x)
{
    Console.WriteLine($"x = {x}");  // ❌ забыл убрать
    return x * 2;
}
// Production logs засоряются
```

✅ Используй `_logger.LogDebug` — в production отключается через config.

### 2. Breakpoint в hot loop

```csharp
for (int i = 0; i < 1_000_000; i++)
{
    Process(i);  // ← breakpoint
    // F5 миллион раз? Нет, conditional breakpoint!
}
```

✅ Conditional breakpoint: `i == 999_999` или `i % 100_000 == 0`.

### 3. Не использовать Step Over

```csharp
public void Method()
{
    var data = ComplexOperation();  // ← здесь
    Use(data);
}
```

Junior жмёт **F11** — заходит вглубь ComplexOperation. Если она работает — это ВРЕМЯ. F10 шагает **через**, не **в**.

### 4. Forgot Edit and Continue limitation

```csharp
async Task DoWork()
{
    await Task.Delay(1000);
    // изменить код тут — может не сработать!
}
```

Restart — единственный надёжный способ для async.

### 5. Production vs Debug builds

```csharp
#if DEBUG
    Console.WriteLine("Started");
#endif

// Debug.Assert — также убирается в Release
Debug.Assert(x > 0);
```

Понимай в каком build выполняется код.

### 6. Logging в hot path

```csharp
public int Add(int a, int b)
{
    _logger.LogDebug("Add({A}, {B})", a, b);  // 1M calls/sec → log overhead!
    return a + b;
}

// ✅ Check level если log expensive
if (_logger.IsEnabled(LogLevel.Debug))
{
    _logger.LogDebug("Complex {Data}", expensive.GetDetails());
}
```

### 7. Catching Exception без logging

```csharp
catch (Exception)
{
    return null;  // ❌ swallowed silent
}

// ✅
catch (Exception ex)
{
    _logger.LogError(ex, "Failed to process");
    return null;
}
```

### 8. Не использовать `nameof` в logs

```csharp
// ❌
_logger.LogError("Argument 'userId' was null");
// Refactor → переименовали userId → log стал устаревший

// ✅
_logger.LogError("Argument {Param} was null", nameof(userId));
```

---

## 16. Best Practices

### Mindset

- **Debugger first** — Console.WriteLine только если debugger недоступен (production)
- **Hypothesize first** — что должно быть? Что вижу?
- **Binary search** — bug где-то в большом коде → найди середину, проверь invariant
- **Read errors внимательно** — stack trace говорит всё

### Daily workflow

```
1. Bug report пришёл
   ↓
2. Воспроизвести локально (минимальный repro)
   ↓
3. Поставить breakpoint в подозрительном месте
   ↓
4. Run debugger, проверить state
   ↓
5. Если непонятно — call stack, watch
   ↓
6. Fix
   ↓
7. Написать regression test
   ↓
8. Verify fix
```

### Logging discipline

- **`LogInformation`** для бизнес-events
- **`LogDebug`** для diagnostics — в production выключено
- **`LogWarning`** для подозрительного
- **`LogError`** — обязательно с **exception**
- **Structured arguments** всегда (никогда concat)
- **Correlation ID** для request tracing
- **`{@Object}`** для serialize complex types

См. [[../AspNetCore/logging-observability|Logging & Observability]].

---

## 17. Cheat sheet

| Hotkey (VS) | Действие |
|-------------|----------|
| `F9` | Toggle breakpoint |
| `F5` | Start / Continue |
| `F10` | Step Over |
| `F11` | Step Into |
| `Shift+F11` | Step Out |
| `Ctrl+Shift+F5` | Restart |
| `Ctrl+Alt+W, 1` | Watch window |
| `Ctrl+Alt+I` | Immediate window |
| `Ctrl+Alt+C` | Call Stack |
| `Ctrl+Alt+E` | Exception Settings |
| Right-click breakpoint | Conditional / Hit Count |

| Что нужно | Решение |
|-----------|---------|
| Variable value | Hover или Watch |
| Quick eval | Immediate window |
| Bug в determined условии | Conditional breakpoint |
| Bug глубоко в stack | Call Stack |
| Custom display | `[DebuggerDisplay]` |
| Production debugging | `ILogger` + correlation ID |
| Performance | `Stopwatch` или dotTrace |
| Memory leak | Diagnostic Tools snapshots |

---

## 18. См. также

- [[csharp-basics|C# Basics]] — без основ debugger не поможет
- [[error-handling|Error Handling]] — try/catch + log
- [[../AspNetCore/logging-observability|Logging & Observability]] — production logging
- [[../Infrastructure/observability|Observability]] — distributed systems
- [[../Runtime/diagnostics-tools|Diagnostics Tools]] — dotnet-counters, trace, dump
- [[../Performance/memory-profiling|Memory Profiling]]
- [[../Testing/testing-fundamentals|Testing Fundamentals]] — regression tests

## Reading list

- **Microsoft Docs — Debug in Visual Studio** — learn.microsoft.com/visualstudio/debugger
- **Microsoft Docs — Logging in .NET** — learn.microsoft.com/dotnet/core/extensions/logging
- **Tess Ferrandez blog** — debugging blog (классика по managed debugging)
- **Sasha Goldstein — Pro .NET Performance** (книга)
- **Konrad Kokosa — Pro .NET Memory Management** (deep memory debugging)

---

## Decision tree

```
Какой подход к отладке?
│
├── Lokal development?
│   ├── Reproducible bug → debugger (F9 breakpoint, F5 run, F10 step over)
│   ├── Сложно reproduce → conditional breakpoint
│   ├── Глубоко в stack → Call Stack window
│   └── Bug в LINQ chain → разбить на отдельные variables
│
├── Production?
│   ├── Crash / exception → structured logging + Exception filter
│   ├── Slow performance → Stopwatch logging или dotTrace
│   ├── Memory leak → Diagnostic Tools snapshots или dotnet-dump
│   └── Distributed bug → correlation ID + Loki/Seq queries
│
└── Bug непонятен?
    ├── Сначала reproduce — write failing test
    ├── Затем binary search — half code suspect, проверить middle
    └── Read errors carefully — stack trace говорит почти всё
```
