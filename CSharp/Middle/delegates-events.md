---
tags: [csharp, delegates, events, middle, func, action, eventhandler, observer]
level: Middle
date: 2026-05-07
---

# Delegates и Events — функции как объекты

> **Type-safe function pointers + Observer pattern.** `Func`/`Action`/`Predicate`, multicast delegates, `event` keyword, `EventHandler<T>`, weak events. Закрывает пробел: «знаю про lambda, не понимаю чем `event` отличается от обычного `Action`, и почему raise через `?.Invoke`».

---

## 0. Как читать

Если впервые — раздел 1→3. Events deep — раздел 5. Production — раздел 9 (best practices), 11 (pitfalls).

Cross-language якоря свёрнуты. Interview-вопросы встроены.

---

## 1. Что это, зачем и когда

### 1.1. Delegate — type-safe function pointer

```csharp
public delegate int Operation(int a, int b);

Operation add = (a, b) => a + b;
Operation mul = (a, b) => a * b;

int result = add(2, 3);   // 5
result = mul(2, 3);        // 6
```

Delegate — type, который **хранит reference на method** (или lambda). Type-safe — signature checked compile-time.

### 1.2. Зачем delegates

1. **Callbacks** — pass behavior as data.
2. **Strategy pattern** — algorithm как parameter.
3. **Events** — Observer pattern.
4. **LINQ** — Where/Select принимают Func.
5. **Async patterns** — async continuations.

### 1.3. Главное правило

```
Delegate когда:
  - Pass behavior (callback, strategy)
  - Single-method interface — alternative
  - LINQ / functional style

Func<...> / Action<...> когда:
  - Generic delegates достаточны (~99% случаев)
  - Custom delegate type не нужен

Custom delegate когда:
  - Self-documenting name важно (event signature, clear intent)
  - Variance модификаторы нужны

Event когда:
  - Class publishes notifications (Observer pattern)
  - Subscribers add/remove dynamically
  - Только class сам raises event (encapsulation)
```

### 1.4. Эволюция

| Версия | Что |
|--------|-----|
| **.NET 1.0** | delegate keyword, events |
| **.NET 2.0** | Generic delegates, anonymous methods |
| **C# 3.0** | Lambda expressions |
| **C# 7+** | Local functions (вместо некоторых delegates) |
| **C# 8** | Static lambdas (no closure) |
| **C# 9** | Function pointers (`delegate*`) — unsafe |
| **C# 11** | Generic attributes, file-scoped namespaces |

> [!info]- Если ты знаешь Java / Python / JavaScript / Rust
> **Java:** functional interfaces (`Function<T,R>`, `Consumer<T>`, `Predicate<T>` от Java 8+) ↔ Func/Action/Predicate. Method references (`obj::method`) ↔ method group. Single-abstract-method interfaces.
>
> **Python:** functions are first-class — `def` returns callable, lambdas. Нет explicit delegate type — duck typing.
>
> **JavaScript:** functions first-class. `addEventListener` = events. Closures очень похожи на C# lambdas.
>
> **Rust:** `Fn`/`FnMut`/`FnOnce` traits ↔ Func variants. Closures с capture semantics. Strict ownership rules.

> [!question]- Интервью: что такое delegate в C#?
> Type-safe **function pointer** — type, хранящий reference на method (или lambda). Compile-time signature check. Делает functions first-class — pass as parameter, store, invoke. Built-in generic delegates: `Func<...>` (returns value), `Action<...>` (void), `Predicate<T>` (bool). Под капотом — class inheriting `MulticastDelegate` (множественные subscribers). Use cases: callbacks, Strategy pattern, LINQ (Where/Select), events. Method group conversion: `del = SomeMethod` (без parens — converts method to delegate).

---

## 2. Func / Action / Predicate

### 2.1. Func — returns value

```csharp
Func<int> f1 = () => 42;
Func<int, int> f2 = x => x * 2;
Func<int, int, int> f3 = (a, b) => a + b;
Func<string, int> len = s => s.Length;

// Last type parameter — return type
// Func<TResult>          — 0 params
// Func<T, TResult>       — 1 param
// Func<T1, T2, TResult>  — 2 params
// ... up to 16 params
```

### 2.2. Action — void

```csharp
Action a1 = () => Console.WriteLine("Hello");
Action<int> a2 = x => Console.WriteLine(x);
Action<string, int> a3 = (msg, code) => Log(msg, code);
```

### 2.3. `Predicate<T>` — bool result

```csharp
Predicate<int> isEven = x => x % 2 == 0;
bool result = isEven(4);   // true

// Equivalent: Func<int, bool>
Func<int, bool> isEvenFunc = x => x % 2 == 0;
```

В новом коде — `Func<T, bool>` чаще (LINQ uses Func). `Predicate<T>` — legacy (Array.Find, List.Find).

### 2.4. `EventHandler<T>`

```csharp
public class OrderService
{
    public event EventHandler<OrderCreatedEventArgs>? OrderCreated;
}

public class OrderCreatedEventArgs : EventArgs
{
    public Order Order { get; init; } = null!;
}

service.OrderCreated += (sender, args) => Console.WriteLine(args.Order.Id);
```

`EventHandler<TArgs>` — standard для events, signature `(object? sender, TArgs e)`.

### 2.5. Type inference

```csharp
var ints = Enumerable.Range(1, 10);
var evens = ints.Where(x => x % 2 == 0);   // x inferred as int
var doubled = ints.Select(x => x * 2);

// Lambda inference при assignment
var add = (int a, int b) => a + b;   // C# 10+ inferred Func<int,int,int>
```

### 2.6. Method group conversion

```csharp
public static int Square(int x) => x * x;

Func<int, int> sq = Square;   // method group → delegate
var squares = new[] { 1, 2, 3 }.Select(Square);   // works
```

> [!question]- Интервью: чем `Func<int, bool>` отличается от `Predicate<int>`?
> **Identical signature** — `int → bool`. Difference исключительно в **conventional usage**: `Predicate<T>` — older, в `Array.Find`, `List.Find`, `List.RemoveAll`. `Func<T, bool>` — newer, везде в LINQ. Compatible: можно cast между, но требует explicit. Best practice 2024+: `Func<T, bool>` для consistency с LINQ ecosystem. `Predicate<T>` оставить для legacy API compat.

---

## 3. Lambda expressions

### 3.1. Базовый syntax

```csharp
// Expression lambda
x => x * 2;

// Statement lambda
(x, y) =>
{
    var sum = x + y;
    return sum * 2;
};

// No parameters
() => Console.WriteLine("Hello");

// Discards (C# 9+)
(_, _) => 0;   // ignore both
```

### 3.2. Closure — captured variables

```csharp
int multiplier = 3;
Func<int, int> times = x => x * multiplier;   // captures multiplier

times(5);   // 15

multiplier = 10;
times(5);   // 50! — captures by reference (technically, через generated class)
```

### 3.3. Closure mechanics

Compiler генерирует hidden class:

```csharp
// Source
int factor = 3;
Func<int, int> mul = x => x * factor;

// Compiler-generated (~примерно)
class Closure
{
    public int factor;
    public int Lambda(int x) => x * factor;
}

var closure = new Closure { factor = 3 };
Func<int, int> mul = closure.Lambda;
```

Captures — **heap allocation**. Performance impact в hot path.

### 3.4. Static lambdas (C# 9+)

```csharp
Func<int, int> sq = static x => x * x;   // no closure capture allowed
// static lambda — гарантия nothing captured, compiler оптимизирует
```

`static` keyword — compile-time guarantee no captures. Performance optimization.

### 3.5. Closure pitfalls — loop variable

```csharp
var actions = new List<Action>();
for (int i = 0; i < 5; i++)
{
    actions.Add(() => Console.WriteLine(i));
}
foreach (var a in actions) a();
// Печатает 5 5 5 5 5 (captured i, mutates до 5)
```

**Фикс:** local copy.

```csharp
for (int i = 0; i < 5; i++)
{
    int copy = i;   // separate variable per iteration
    actions.Add(() => Console.WriteLine(copy));
}
// Печатает 0 1 2 3 4
```

`foreach` (C# 5+) — каждая iteration имеет свой copy by default.

```csharp
foreach (var x in items)
{
    actions.Add(() => Console.WriteLine(x));   // OK по C# 5+
}
```

### 3.6. Local functions (C# 7+)

```csharp
public int Process(int input)
{
    return Helper(input);
    
    int Helper(int x)   // local function
    {
        return x * 2;
    }
}
```

Альтернатива lambda — нет heap allocation для captures (`static` local functions).

### 3.7. Expression-bodied vs statement-bodied

```csharp
Func<int, int> sq = x => x * x;                  // expression
Func<int, int> sq2 = x => { return x * x; };     // statement (verbose)
```

Expression — для simple. Statement — для multiple statements.

> [!question]- Интервью: что такое closure в C#?
> Lambda (или anonymous method), которая **captures variables** из enclosing scope. Под капотом compiler генерирует **hidden class** с captured variables как fields. Lambda становится метод этого класса. **Heap allocation** для closure object. Performance impact: per-invocation overhead если closure recreated. **`static` lambda** (C# 9+) — гарантия no captures, compiler оптимизирует. Pitfall: loop variable capture (до C# 5 — caught one shared, after — per-iteration).

---

## 4. Multicast delegates

### 4.1. Composition через `+=` / `-=`

```csharp
Action a = () => Console.WriteLine("first");
a += () => Console.WriteLine("second");
a += () => Console.WriteLine("third");

a();
// Output:
// first
// second
// third
```

Multiple methods invoked в order subscription.

### 4.2. Под капотом — invocation list

```csharp
Delegate[] invocations = a.GetInvocationList();
foreach (Delegate d in invocations)
{
    d.DynamicInvoke();   // alternative invocation
}
```

Multicast delegate stores **invocation list** — array of single delegates.

### 4.3. Func multicast — last result wins

```csharp
Func<int> f = () => 1;
f += () => 2;
f += () => 3;

int result = f();   // 3 — only last result returned!
```

For Func — multicast invokes all, but **only last return value returned**. Anti-pattern для Func multicast.

### 4.4. Exception в multicast

```csharp
Action a = () => throw new Exception("first");
a += () => Console.WriteLine("second");
a += () => Console.WriteLine("third");

a();   // First handler throws, остальные НЕ invoked!
```

Exception **прерывает** invocation chain. Subsequent handlers пропускаются.

### 4.5. Manual iteration для exception isolation

```csharp
var exceptions = new List<Exception>();
foreach (Delegate handler in a.GetInvocationList())
{
    try
    {
        ((Action)handler)();
    }
    catch (Exception ex)
    {
        exceptions.Add(ex);
    }
}

if (exceptions.Count > 0)
    throw new AggregateException(exceptions);
```

Manual iterate + try/catch per handler. Используется внутри events.

### 4.6. Removal с `-=`

```csharp
Action a = HandlerA;
a += HandlerB;
a += HandlerB;   // подписан 2 раза!

a -= HandlerB;   // удаляет первое вхождение, осталось одно
```

Important: `-=` удаляет **первое** matching. Для lambda — нужна **сохранённая reference**:

```csharp
// ❌ Не работает — два отдельных lambda objects
a += () => Log();
a -= () => Log();   // не tot же object

// ✅ Сохрани reference
Action handler = () => Log();
a += handler;
a -= handler;   // works
```

> [!question]- Интервью: что происходит когда multicast delegate имеет return value?
> Multicast invokes **все** handlers в order. Но returned value — только **последнего** handler (lost для всех остальных). Это anti-pattern для `Func<T>` multicast — typically не используется. Для `Action` (void) — все handlers run normally. Pitfall с exceptions: exception **прерывает chain** — handlers после throwing не invoked. Workaround: manual iterate `GetInvocationList()` + try/catch per handler.

---

## 5. Events deep

### 5.1. event keyword

```csharp
public class OrderService
{
    public event EventHandler<OrderCreatedEventArgs>? OrderCreated;
    
    public void PlaceOrder(Order order)
    {
        // ... save
        OrderCreated?.Invoke(this, new OrderCreatedEventArgs { Order = order });
    }
}

// Subscriber
service.OrderCreated += (sender, args) => Console.WriteLine($"Order {args.Order.Id}");
```

### 5.2. event vs обычный delegate field

```csharp
// Без event — public delegate field
public Action OnChanged;
// External code может: OnChanged() напрямую вызвать!
// Также: OnChanged = null — clear all subscribers!

// С event — protected
public event Action OnChanged;
// External: только += и -=. Invoke и assignment — только в classe.
```

`event` — protective layer над delegate field. Encapsulation:
- **Subscribers могут только +=/-=**.
- **Только publisher (declaring class)** может invoke.
- **Только publisher** может assign null (clear).

### 5.3. Custom event accessor

```csharp
private EventHandler<OrderCreatedEventArgs>? _orderCreated;

public event EventHandler<OrderCreatedEventArgs>? OrderCreated
{
    add { _orderCreated += value; Console.WriteLine("Subscribed"); }
    remove { _orderCreated -= value; Console.WriteLine("Unsubscribed"); }
}
```

Custom add/remove — для logging, validation, weak references.

### 5.4. Thread-safe raise

```csharp
public event EventHandler? Changed;

protected virtual void OnChanged()
{
    var handler = Changed;   // local copy для thread safety
    handler?.Invoke(this, EventArgs.Empty);
}

// Ещё короче (.NET 4+)
protected virtual void OnChanged() => Changed?.Invoke(this, EventArgs.Empty);
```

`?.Invoke` — null-safe + thread-safe (locally copies field).

### 5.5. Naming convention

```csharp
public event EventHandler<OrderCreatedEventArgs> OrderCreated;
public event EventHandler<OrderCancelledEventArgs> OrderCancelled;
public event PropertyChangedEventHandler? PropertyChanged;

// Past tense — event already happened
// "OnXxx" — protected method для raising (по convention)
protected virtual void OnOrderCreated(OrderCreatedEventArgs e) => OrderCreated?.Invoke(this, e);
```

### 5.6. EventArgs convention

```csharp
public class OrderCreatedEventArgs : EventArgs
{
    public Order Order { get; init; } = null!;
    public DateTime CreatedAt { get; init; }
}

// Empty args
service.OnDispose += (s, e) => { /* EventArgs.Empty */ };
```

Subclass `EventArgs`. Suffix `EventArgs`. Для no-data events — `EventArgs.Empty`.

### 5.7. Static events

```csharp
public class GlobalLogger
{
    public static event EventHandler<LogEventArgs>? OnLog;
    
    public static void Log(string message)
    {
        OnLog?.Invoke(null, new LogEventArgs { Message = message });
    }
}

GlobalLogger.OnLog += (s, e) => Console.WriteLine(e.Message);
```

Static events — global notifications. Watch out для **memory leaks** (см. раздел 6).

> [!question]- Интервью: чем `event` отличается от обычного delegate field?
> `event` — encapsulation layer над delegate. **Subscribers могут только `+=`/`-=`**, не invoke и не assign. **Только publisher (declaring class)** может invoke и clear (`null`-assign). Без `event` — public delegate field позволяет external code вызывать handler или wipe all subscribers — нарушение Observer pattern. `event` гарантирует controlled publish/subscribe. Compiler генерирует backing private field + public add/remove accessors. Custom add/remove — для logging, validation.

---

## 6. Memory leaks с events

### 6.1. Event source держит subscriber alive

```csharp
public class EventSource
{
    public event EventHandler? OnEvent;
}

public class Subscriber
{
    public Subscriber(EventSource source)
    {
        source.OnEvent += HandleEvent;   // strong reference
    }
    
    private void HandleEvent(object? sender, EventArgs e) { }
}

// Use case
var source = new EventSource();
{
    var sub = new Subscriber(source);
}   // sub goes out of scope BUT source.OnEvent держит reference
// → sub НЕ collected GC

source = null;
// Теперь и source может быть collected (если no другие references)
```

**Subscriber leaks** пока source alive. Static events / long-living publishers (Application, Singleton) — самые опасные.

### 6.2. Решение 1 — explicit unsubscribe

```csharp
public class Subscriber : IDisposable
{
    private readonly EventSource _source;
    
    public Subscriber(EventSource source)
    {
        _source = source;
        _source.OnEvent += HandleEvent;
    }
    
    public void Dispose()
    {
        _source.OnEvent -= HandleEvent;
    }
    
    private void HandleEvent(object? sender, EventArgs e) { }
}

using var sub = new Subscriber(source);
// При dispose — unsubscribe
```

### 6.3. Решение 2 — weak references

```csharp
public class WeakEventManager<TArgs> where TArgs : EventArgs
{
    private readonly List<WeakReference<EventHandler<TArgs>>> _handlers = new();
    
    public void Subscribe(EventHandler<TArgs> handler)
    {
        _handlers.Add(new WeakReference<EventHandler<TArgs>>(handler));
    }
    
    public void Raise(object sender, TArgs args)
    {
        for (int i = _handlers.Count - 1; i >= 0; i--)
        {
            if (_handlers[i].TryGetTarget(out var handler))
                handler(sender, args);
            else
                _handlers.RemoveAt(i);   // dead reference — clean up
        }
    }
}
```

Weak reference не препятствует GC. Subscriber может быть collected.

### 6.4. Решение 3 — short-lived subscribers

Best для UI: subscribe в OnLoaded, unsubscribe в OnUnloaded. View lifetime короткое — leaks bounded.

### 6.5. WPF / Reactive Extensions

WPF: `WeakEventManager<TEvent>` (Microsoft) для weak event subscriptions. Reactive (Rx): `Subject<T>` + `IObservable<T>` — explicit lifetime.

> [!question]- Интервью: как избежать memory leaks с events?
> Event source держит **strong references** на subscribers через invocation list. Если subscriber не unsubscribes — alive пока source alive. Long-living publishers (Application, Singleton, static events) — основная утечка. Решения: 1) **Explicit unsubscribe** (`-=`) при cleanup (IDisposable, OnUnloaded). 2) **Weak references** через `WeakEventManager` (WPF) или custom. 3) **Short-lived subscribers** — bounded lifetime. 4) **Reactive Extensions** (`IObservable<T>` + `IDisposable` subscription). Memory profiler — обнаруживает leaked subscribers (refs в delegate invocation list).

---

## 7. Async и delegates

### 7.1. `Func<Task<T>>`

```csharp
Func<int, Task<int>> compute = async x =>
{
    await Task.Delay(100);
    return x * 2;
};

int result = await compute(5);   // 10
```

### 7.2. async void — anti-pattern

```csharp
public event EventHandler? Click;

// ❌ async void event handler
service.Click += async (s, e) =>
{
    await DoSomethingAsync();   // exceptions становятся unobserved!
};
```

`async void` — exceptions terminate process (unobserved task). Best practice: `async Task` где возможно.

**Исключение для events** — UI frameworks ожидают `void` signatures. Wrap в try/catch:

```csharp
service.Click += async (s, e) =>
{
    try { await DoSomethingAsync(); }
    catch (Exception ex) { Log.Error(ex); }
};
```

### 7.3. `EventHandler<T>` async

C# events не имеют built-in async. Workaround patterns:

```csharp
// Func-based (custom)
public event Func<Order, Task>? OrderCreatedAsync;

public async Task PlaceOrderAsync(Order order)
{
    // ...
    if (OrderCreatedAsync != null)
    {
        foreach (Func<Order, Task> handler in OrderCreatedAsync.GetInvocationList())
            await handler(order);
    }
}
```

Or **MediatR** library — explicit async handlers.

> [!question]- Интервью: проблема `async void` в event handlers?
> `async void` — exceptions становятся **unobserved** — terminate process в .NET Core 2.1+. **Cannot be awaited** — caller не знает completion. UI frameworks (WinForms, WPF) **требуют** void signature, поэтому unavoidable. Best practice: wrap body в try/catch, log all exceptions. Для new patterns — async events через `Func<TArgs, Task>` invocation list, или MediatR-style. **Никогда `async void`** в methods кроме event handlers + entry points.

---

## 8. Function pointers — `delegate*` (C# 9+)

### 8.1. Unsafe alternative

```csharp
unsafe
{
    delegate*<int, int, int> add = &Add;
    int result = add(2, 3);   // 5
}

static int Add(int a, int b) => a + b;
```

### 8.2. Когда использовать

- **Performance hot path** — meter than delegate (no allocation, no virtual call).
- **Native interop** — function pointers для P/Invoke.
- **Editor / parser** — dispatch tables.

### 8.3. Limitations

- `unsafe` context required.
- Только static methods (или unmanaged).
- No multicast.
- No generic type parameters.
- Без `[UnmanagedCallersOnly]` — managed only.

Используется редко. Обычный delegate enough.

---

## 9. Best Practices

### 9.1. Delegates

- ✅ **`Func<>`/`Action<>`** для most cases.
- ✅ **Custom delegate** для clear naming (event signatures).
- ✅ **`static` lambda** где no captures (.NET / C# 9+).
- ✅ **Method group** conversion `del = Method`.
- ✅ **Local functions** для encapsulated helpers.
- ❌ **`Predicate<T>` в новом коде** — `Func<T,bool>` для consistency.
- ❌ **Multicast `Func<T>`** — только last result.

### 9.2. Events

- ✅ **`EventHandler<TArgs>`** standard signature.
- ✅ **`EventArgs` subclass** для args.
- ✅ **`?.Invoke`** для null-safe + thread-safe raise.
- ✅ **Past-tense naming** (`OrderCreated`, не `CreateOrder`).
- ✅ **`On...` protected method** для raising в derived classes.
- ✅ **Unsubscribe** на cleanup (IDisposable, OnUnloaded).
- ❌ **`async void`** outside UI handlers.
- ❌ **Multicast Func events** — confusing semantics.

### 9.3. Performance

- ✅ **Static lambdas** для no-capture.
- ✅ **Cache delegates** в fields для reuse.
- ✅ **Local functions** vs lambdas в hot path.
- ❌ **New lambda per call** в loop.
- ❌ **DynamicInvoke** — slow, типа reflection.

### 9.4. Не делай

- ❌ Public delegate field вместо event.
- ❌ Forget unsubscribe — memory leak.
- ❌ Modify event handler во время invocation.
- ❌ async void outside event handlers / entry points.

---

## 10. Decision tree

```
Что нужно?
│
├── Pass behavior как parameter
│   ├── Built-in fits → Func<...> / Action<...> / Predicate<T>
│   ├── Self-doc name важен → custom delegate
│   └── Variance modifiers → custom delegate с in/out
│
├── Notification pattern (Observer)
│   ├── Public class API → event EventHandler<T>
│   ├── Internal callback → Action<...> field или property
│   └── Strong-typed many handlers → MediatR / Reactive
│
├── Async-aware events
│   ├── Простой → Func<T, Task> + manual invocation list
│   └── Production → MediatR
│
├── Performance
│   ├── No capture → static lambda
│   ├── Cached → field-level delegate
│   ├── Hot path interop → delegate* (function pointer, unsafe)
│   └── Local helper → local function (вместо lambda)
│
└── Memory
    ├── Event subscription → IDisposable + unsubscribe в Dispose
    ├── Long-living source + short-living subscribers → weak events
    └── UI bindings → OnLoaded/OnUnloaded pattern
```

---

## 11. Cheat sheet

```csharp
// === Delegates ===
public delegate int Operation(int a, int b);

Operation add = (a, b) => a + b;
Operation mul = (a, b) => a * b;

// Built-in generic
Func<int, int> sq = x => x * x;
Func<int, int, int> add2 = (a, b) => a + b;
Action<string> log = msg => Console.WriteLine(msg);
Predicate<int> isEven = x => x % 2 == 0;

// === Lambda ===
var f1 = (int x) => x * 2;          // expression
var f2 = (int x) => { return x * 2; };   // statement
Func<int> noArgs = () => 42;
Action<int, int> twoArgs = (a, b) => Console.WriteLine($"{a} {b}");

// Static lambda (no closure, C# 9+)
Func<int, int> sq2 = static x => x * x;

// === Multicast ===
Action chain = () => Console.WriteLine("A");
chain += () => Console.WriteLine("B");
chain();   // A, B

// === Events ===
public class Service
{
    public event EventHandler<OrderEventArgs>? OrderCreated;
    
    public void PlaceOrder(Order o)
    {
        // ... save
        OrderCreated?.Invoke(this, new OrderEventArgs(o));
    }
}

public class OrderEventArgs : EventArgs
{
    public Order Order { get; }
    public OrderEventArgs(Order o) => Order = o;
}

// Subscribe
service.OrderCreated += (s, e) => Console.WriteLine(e.Order.Id);

// Unsubscribe (с saved reference)
EventHandler<OrderEventArgs> handler = (s, e) => Console.WriteLine(e.Order.Id);
service.OrderCreated += handler;
service.OrderCreated -= handler;

// === Local function (C# 7+) ===
public int Process(int input)
{
    return Helper(input);
    int Helper(int x) => x * 2;
}

// === Custom add/remove ===
private EventHandler? _changed;
public event EventHandler? Changed
{
    add { _changed += value; Log("subscribed"); }
    remove { _changed -= value; Log("unsubscribed"); }
}

// === Function pointer (C# 9+) ===
unsafe
{
    delegate*<int, int, int> add = &AddStatic;
    int r = add(2, 3);
}
static int AddStatic(int a, int b) => a + b;
```

---

## 12. Common Pitfalls

### 12.1. Loop variable capture

```csharp
for (int i = 0; i < 5; i++)
    actions.Add(() => Console.WriteLine(i));
// Все печатают 5
```

**Фикс:** local copy `int copy = i;` или `foreach`.

### 12.2. Lambda subscribe vs unsubscribe

```csharp
service.OnEvent += () => Log();
service.OnEvent -= () => Log();   // ❌ другой lambda object!
```

**Фикс:** сохранить reference в variable.

### 12.3. async void

```csharp
service.Click += async (s, e) => await DoAsync();   // exceptions unobserved!
```

**Фикс:** try/catch + log внутри handler.

### 12.4. Multicast Func — last wins

```csharp
Func<int> f = () => 1;
f += () => 2;
int r = f();   // 2 — first lost
```

**Фикс:** не использовать Func multicast или manual iterate.

### 12.5. Exception в multicast прерывает

```csharp
Action a = HandlerA;
a += HandlerB;   // Если A throws — B не invoked
```

**Фикс:** manual iterate с try/catch если изоляция нужна.

### 12.6. Memory leak — event subscription

```csharp
class Sub
{
    public Sub(EventSource src) => src.OnEvent += HandleEvent;
    void HandleEvent(object? s, EventArgs e) { }
}
// Sub не collected пока src alive
```

**Фикс:** IDisposable + unsubscribe.

### 12.7. Public delegate field

```csharp
public Action OnChanged;   // ❌ external может clear или invoke
```

**Фикс:** `event Action OnChanged`.

### 12.8. Modification во время invocation

```csharp
public event EventHandler? OnEvent;
OnEvent?.Invoke(this, EventArgs.Empty);
// Если handler unsubscribes других в processing — race condition
```

**Фикс:** local copy `var handler = OnEvent; handler?.Invoke(...)`.

### 12.9. Closure heap allocation в hot path

```csharp
for (int i = 0; i < 1_000_000; i++)
    list.ForEach(x => x.Process(i));   // closure allocates каждый iteration
```

**Фикс:** static lambda + parameter, или regular for.

### 12.10. `Predicate<T>` vs `Func<T, bool>` mixing

```csharp
Predicate<int> p = x => x > 5;
list.Where(p);   // ❌ Where принимает Func<T,bool>, не Predicate
```

**Фикс:** `list.Where(x => p(x))` или один тип везде.

> [!question]- Интервью: топ-3 ошибки с delegates/events?
> 1) **Loop variable capture** — `for (int i...) { actions.Add(() => Console.WriteLine(i)); }` все печатают финальное значение `i`. До C# 5 та же problem с `foreach`. Используй local copy. 2) **`async void`** в event handlers — exceptions unobserved, terminate process. UI frameworks требуют void, обязательно try/catch внутри. 3) **Memory leaks через event subscription** — long-living source держит subscribers alive. Always unsubscribe в IDisposable / OnUnloaded.

---

## 13. Practice exercises

### 13.1. Custom event publisher

```csharp
public class StockTicker
{
    public event EventHandler<PriceChangedEventArgs>? PriceChanged;
    
    private readonly Dictionary<string, decimal> _prices = new();
    
    public void UpdatePrice(string symbol, decimal price)
    {
        var oldPrice = _prices.GetValueOrDefault(symbol);
        _prices[symbol] = price;
        OnPriceChanged(new PriceChangedEventArgs(symbol, oldPrice, price));
    }
    
    protected virtual void OnPriceChanged(PriceChangedEventArgs e) =>
        PriceChanged?.Invoke(this, e);
}

public class PriceChangedEventArgs : EventArgs
{
    public string Symbol { get; }
    public decimal OldPrice { get; }
    public decimal NewPrice { get; }
    public decimal Change => NewPrice - OldPrice;
    public decimal ChangePercent => OldPrice == 0 ? 0 : (Change / OldPrice) * 100;
    
    public PriceChangedEventArgs(string symbol, decimal oldPrice, decimal newPrice)
    {
        Symbol = symbol;
        OldPrice = oldPrice;
        NewPrice = newPrice;
    }
}

// Use
var ticker = new StockTicker();
ticker.PriceChanged += (s, e) =>
    Console.WriteLine($"{e.Symbol}: {e.OldPrice} → {e.NewPrice} ({e.ChangePercent:F2}%)");
```

### 13.2. Subscriber with auto-unsubscribe

```csharp
public class Subscriber : IDisposable
{
    private readonly StockTicker _ticker;
    private readonly EventHandler<PriceChangedEventArgs> _handler;
    private bool _disposed;
    
    public Subscriber(StockTicker ticker)
    {
        _ticker = ticker;
        _handler = OnPriceChanged;
        _ticker.PriceChanged += _handler;
    }
    
    private void OnPriceChanged(object? sender, PriceChangedEventArgs e)
    {
        if (Math.Abs(e.ChangePercent) > 5)
            Console.WriteLine($"ALERT: {e.Symbol} changed {e.ChangePercent:F2}%");
    }
    
    public void Dispose()
    {
        if (_disposed) return;
        _ticker.PriceChanged -= _handler;
        _disposed = true;
    }
}

using var sub = new Subscriber(ticker);
ticker.UpdatePrice("AAPL", 150);
ticker.UpdatePrice("AAPL", 160);   // ALERT — 6.67%
// Dispose автоматически unsubscribe
```

### 13.3. Async event pattern

```csharp
public class AsyncOrderService
{
    public event Func<Order, Task>? OrderCreatedAsync;
    
    public async Task PlaceOrderAsync(Order order)
    {
        // ... save
        await RaiseOrderCreatedAsync(order);
    }
    
    private async Task RaiseOrderCreatedAsync(Order order)
    {
        var handlers = OrderCreatedAsync;
        if (handlers == null) return;
        
        var exceptions = new List<Exception>();
        foreach (Func<Order, Task> handler in handlers.GetInvocationList())
        {
            try { await handler(order); }
            catch (Exception ex) { exceptions.Add(ex); }
        }
        
        if (exceptions.Count > 0)
            throw new AggregateException(exceptions);
    }
}

// Use
var service = new AsyncOrderService();
service.OrderCreatedAsync += async order =>
{
    await SendEmailAsync(order);
};
service.OrderCreatedAsync += async order =>
{
    await UpdateInventoryAsync(order);
};

await service.PlaceOrderAsync(new Order { Id = 1 });
```

---

## 14. Что читать дальше

1. **[[oop|OOP]]** — Observer pattern, events.
2. **[[generics-deep|Generics deep]]** — Func/Action variance.
3. **[[error-handling|Error Handling]]** — async void exceptions.
4. **[[dispose-pattern|Dispose Pattern]]** — IDisposable + unsubscribe.
5. **MediatR** — async events / commands library.
6. **Reactive Extensions (Rx)** — `IObservable<T>`.

---

## 15. См. также

- [[oop|OOP]] — Observer pattern
- [[generics-deep|Generics deep]] — variance в Func/Action
- [[iterators-yield|yield]] — IEnumerable + delegates
- [[error-handling|Error Handling]] — async void
- [[dispose-pattern|Dispose Pattern]] — unsubscribe
- MediatR library
- ReactiveUI / Rx.NET

---

## 16. Reading list

- **Microsoft Docs — Delegates** — learn.microsoft.com/dotnet/csharp/programming-guide/delegates/
- **Microsoft Docs — Events** — learn.microsoft.com/dotnet/csharp/programming-guide/events/
- **Microsoft Docs — Lambda expressions** — learn.microsoft.com/dotnet/csharp/language-reference/operators/lambda-expressions
- **Microsoft Docs — Function pointers** — learn.microsoft.com/dotnet/csharp/language-reference/proposals/csharp-9.0/function-pointers
- **Jon Skeet — C# in Depth** — delegates and lambdas chapters
- **Stephen Cleary — async events** — blog.stephencleary.com
- **Jeffrey Richter — CLR via C#** — delegates internals
- **MediatR** — github.com/jbogard/MediatR
- **Reactive Extensions** — github.com/dotnet/reactive
