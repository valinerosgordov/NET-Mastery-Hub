---
tags: [delegates, events, lambdas, func, action]
level: Senior
---

# Delegates, Events и Lambdas

> Справочник по делегатам, событиям и лямбда-выражениям. C# 13 / .NET 9.
> Теория → практика → senior-level код → вопросы интервью.

---

## Delegates

### Что такое delegate

Delegate — **type-safe указатель на метод**. Это ссылочный тип, который хранит ссылку на один или несколько методов с определённой сигнатурой. Компилятор генерирует класс, наследующий `System.MulticastDelegate`.

```csharp
// Объявление delegate — определяем сигнатуру
delegate int MathOperation(int a, int b);

// Методы, совместимые с сигнатурой
static int Add(int a, int b) => a + b;
static int Multiply(int a, int b) => a * b;

// Использование
MathOperation op = Add;
int result = op(3, 4); // 7

op = Multiply;
result = op(3, 4);     // 12
```

### Объявление delegate

```csharp
// Без параметров, без возврата
delegate void NotifyHandler();

// С параметрами и возвратом
delegate TResult Converter<in TInput, out TResult>(TInput input);

// С ref/out параметрами
delegate bool TryParser<T>(string input, out T result);

// Generic delegate с constraints
delegate T Factory<T>() where T : new();
```

Под капотом компилятор генерирует:

```csharp
// Примерно такой класс (упрощённо)
sealed class MathOperation : MulticastDelegate
{
    public int Invoke(int a, int b);
    // ВНИМАНИЕ: BeginInvoke/EndInvoke НЕ поддерживаются в .NET Core / .NET 5+
    // Вызов бросает PlatformNotSupportedException. Используй Task/async вместо этого.
    public IAsyncResult BeginInvoke(int a, int b, AsyncCallback callback, object state);
    public int EndInvoke(IAsyncResult result);
}
```

### Multicast delegates (цепочка вызовов)

Delegate может хранить **несколько** методов. При вызове они выполняются последовательно. Возвращается результат **последнего** метода в цепочке.

```csharp
delegate void Logger(string message);

static void ConsoleLog(string msg) => Console.WriteLine($"[Console] {msg}");
static void FileLog(string msg) => Console.WriteLine($"[File] {msg}");
static void DbLog(string msg) => Console.WriteLine($"[DB] {msg}");

// Комбинирование
Logger log = ConsoleLog;
log += FileLog;
log += DbLog;

log("Приложение запущено");
// [Console] Приложение запущено
// [File] Приложение запущено
// [DB] Приложение запущено

// Удаление из цепочки
log -= FileLog;
log("Только Console и DB");
```

**Получение результатов каждого метода в multicast delegate:**

```csharp
delegate int Calculator(int x);

static int Double(int x) => x * 2;
static int Square(int x) => x * x;
static int AddTen(int x) => x + 10;

Calculator calc = Double;
calc += Square;
calc += AddTen;

// Обычный вызов — вернёт только результат последнего (AddTen)
int last = calc(5); // 15

// Получить результат каждого метода
foreach (Calculator d in calc.GetInvocationList().Cast<Calculator>())
{
    int result = d(5);
    Console.WriteLine(result); // 10, 25, 15
}
```

### Встроенные делегаты: Action, Func, Predicate

Для 99% случаев **не нужно** объявлять свой delegate — используй встроенные.

```csharp
// Action — void-метод, до 16 параметров
Action greet = () => Console.WriteLine("Привет");
Action<string> log = msg => Console.WriteLine(msg);
Action<string, int> repeat = (msg, count) =>
{
    for (int i = 0; i < count; i++)
        Console.WriteLine(msg);
};

// Func — метод с возвращаемым значением, до 16 параметров
// Последний generic-аргумент — тип возврата
Func<int> getRandom = () => Random.Shared.Next();
Func<int, int, int> add = (a, b) => a + b;
Func<string, bool> isValid = s => !string.IsNullOrWhiteSpace(s);

// Predicate — частный случай Func<T, bool>
Predicate<int> isEven = x => x % 2 == 0;

List<int> numbers = [1, 2, 3, 4, 5, 6, 7, 8];
List<int> evens = numbers.FindAll(isEven); // [2, 4, 6, 8]
```

### Когда использовать свой delegate vs встроенный

| Сценарий | Решение |
|---|---|
| Обычный callback / pipeline | `Func<T>` или `Action<T>` |
| Нужен `ref`, `out`, `in` параметр | Свой delegate |
| Нужна именованная семантика для API | Свой delegate |
| Event handler | `EventHandler<T>` |
| Больше 16 параметров (никогда) | Свой delegate |

```csharp
// Свой delegate нужен для ref/out
delegate bool TryParseHandler<T>(ReadOnlySpan<char> input, out T result);

TryParseHandler<int> tryParse = int.TryParse;
if (tryParse("42", out int value))
    Console.WriteLine(value); // 42
```

> [!question]- **Интервью: Func vs Action vs delegate — когда свой?**
> `Func<T, TResult>` — делегат с возвратом. `Action<T>` — без возврата. Покрывают 95% случаев.
> Свой delegate — когда нужны `ref`/`out`/`in` параметры, >16 параметров, или семантическое имя в API.

---

## Анонимные методы

### Синтаксис delegate { }

Анонимные методы — **устаревший** способ (C# 2.0). Лямбды почти полностью их заменили, но знать полезно для чтения legacy-кода.

```csharp
// Анонимный метод
Func<int, int, int> add = delegate (int a, int b)
{
    return a + b;
};

// То же самое лямбдой (предпочтительно)
Func<int, int, int> addLambda = (a, b) => a + b;

// Анонимный метод без параметров — может игнорировать параметры delegate
Action<string> ignore = delegate { Console.WriteLine("Параметр проигнорирован"); };
// Лямбда так не может — нужно явно указать параметр или discard
Action<string> ignoreLambda = _ => Console.WriteLine("Discard");
```

### Замыкания (closures) — захват переменных

Замыкание — лямбда или анонимный метод, который **захватывает переменные** из внешней области видимости. Компилятор генерирует скрытый класс для хранения захваченных переменных.

```csharp
int multiplier = 3;

Func<int, int> multiply = x => x * multiplier;

Console.WriteLine(multiply(5)); // 15

// Замыкание захватывает ПЕРЕМЕННУЮ, не значение!
multiplier = 10;
Console.WriteLine(multiply(5)); // 50 — значение изменилось
```

**Что генерирует компилятор (приблизительно):**

```csharp
// Компилятор создаёт DisplayClass
[CompilerGenerated]
sealed class <>c__DisplayClass
{
    public int multiplier;

    public int Method(int x) => x * multiplier;
}
```

### Опасности замыканий (captured variable в цикле)

Классическая ловушка — захват переменной цикла:

```csharp
// ОШИБКА: все лямбды захватывают ОДНУ переменную i
var actions = new List<Action>();
for (int i = 0; i < 5; i++)
{
    actions.Add(() => Console.WriteLine(i));
}
foreach (var action in actions)
    action(); // 5, 5, 5, 5, 5 — все печатают 5!

// ИСПРАВЛЕНИЕ: локальная копия
var actionsFixed = new List<Action>();
for (int i = 0; i < 5; i++)
{
    int local = i; // копия для каждой итерации
    actionsFixed.Add(() => Console.WriteLine(local));
}
foreach (var action in actionsFixed)
    action(); // 0, 1, 2, 3, 4

// foreach НЕ имеет этой проблемы (начиная с C# 5)
var items = new[] { "a", "b", "c" };
var prints = new List<Action>();
foreach (var item in items)
{
    prints.Add(() => Console.WriteLine(item)); // OK — каждая итерация свой item
}
```

**Ещё одна опасность — захват тяжёлых объектов:**

```csharp
// ПЛОХО: замыкание держит ссылку на весь массив, мешает GC
byte[] hugeBuffer = new byte[10_000_000];

Func<int> getLength = () => hugeBuffer.Length;

// hugeBuffer не будет собран GC, пока жив getLength
```

---

## Lambda-выражения

### Expression lambda vs Statement lambda

```csharp
// Expression lambda — одно выражение, return неявный
Func<int, int> square = x => x * x;
Func<string, string> toUpper = s => s.ToUpperInvariant();
Func<int, int, bool> isGreater = (a, b) => a > b;

// Statement lambda — блок кода, return явный
Func<int, string> classify = x =>
{
    if (x < 0) return "negative";
    if (x == 0) return "zero";
    return "positive";
};

// Expression lambda можно использовать как Expression Tree
Expression<Func<int, bool>> expr = x => x > 5;
// Компилируется в дерево выражений, а не в delegate
// Используется в EF Core, OData и т.д.
```

### Natural type inference (C# 10+)

Компилятор может вывести тип delegate из лямбды без явного указания:

```csharp
// C# 10+: natural type — компилятор выводит Func<int, int>
var square = (int x) => x * x;

// Тип параметров нужно указать для var
var greet = (string name) => $"Привет, {name}!";

// Для void-лямбд выводится Action
var log = (string msg) => Console.WriteLine(msg);

// Можно указать return type явно
var parse = int (string s) => int.Parse(s);

// Работает с method groups тоже
var writeLine = Console.WriteLine; // тип не выведется — перегрузки
// Нужно явно:
Action<string> write = Console.WriteLine; // OK
```

### Lambda attributes (C# 10+)

```csharp
// Атрибуты на лямбде — полезно для Minimal APIs
var handler = [Authorize] (HttpContext ctx) => Results.Ok("Секретные данные");

// Атрибуты на параметрах
var endpoint = ([FromQuery] string name, [FromServices] ILogger logger) =>
{
    logger.LogInformation("Запрос от {Name}", name);
    return Results.Ok($"Привет, {name}");
};

// Пример в Minimal API
app.MapGet("/api/users", [Authorize(Roles = "Admin")]
    async ([FromServices] IMediator mediator) =>
    {
        var result = await mediator.Send(new GetUsersQuery());
        return result.IsSuccess ? Results.Ok(result.Value) : Results.Problem();
    });
```

### Lambda parameter modifiers: ref, in, out

```csharp
// ref/out в лямбдах — C# 7.3+, in — C# 9+
// ref — чтение и запись по ссылке
var increment = (ref int x) => x++;

int value = 10;
increment(ref value);
Console.WriteLine(value); // 11

// in — только чтение по ссылке (без копирования)
var printLength = (in ReadOnlySpan<char> span) => Console.WriteLine(span.Length);

// out — запись значения через ссылку
var tryDivide = (int a, int b, out int result) =>
{
    if (b == 0) { result = 0; return false; }
    result = a / b;
    return true;
};

if (tryDivide(10, 3, out int res))
    Console.WriteLine(res); // 3
```

### Default parameter values в лямбдах (C# 12)

```csharp
// C# 12: значения по умолчанию в параметрах лямбд
var greet = (string name, string greeting = "Привет") => $"{greeting}, {name}!";

Console.WriteLine(greet("Мир"));           // Привет, Мир!
Console.WriteLine(greet("Мир", "Здравствуй")); // Здравствуй, Мир!

// Полезно в Minimal APIs
app.MapGet("/search", (string query, int page = 1, int pageSize = 20) =>
{
    // page и pageSize имеют значения по умолчанию
    return Results.Ok(new { query, page, pageSize });
});

// params тоже работает (C# 13)
var sum = (params int[] numbers) => numbers.Sum();
Console.WriteLine(sum(1, 2, 3, 4, 5)); // 15
```

### Static lambdas — избежание аллокаций

Static lambda **запрещает** захват переменных из внешней области. Это гарантирует отсутствие аллокации closure-объекта.

```csharp
int factor = 2;

// Обычная лямбда — захватывает factor, создаёт closure (аллокация)
Func<int, int> withClosure = x => x * factor;

// Static lambda — запрещает захват
Func<int, int> noCapture = static x => x * 2; // OK — литерал

// ОШИБКА КОМПИЛЯЦИИ: static lambda не может захватывать переменные
// Func<int, int> error = static x => x * factor;

// Полезно в hot path для избежания аллокаций
var numbers = new[] { 1, 2, 3, 4, 5 };

// Без аллокации closure
int[] doubled = Array.ConvertAll(numbers, static x => x * 2);

// Static local function — аналогичная оптимизация
static int Square(int x) => x * x;
int[] squares = numbers.Select(Square).ToArray();
```

**Сравнение аллокаций:**

```csharp
// Hot path — каждый вызов создаёт closure
void ProcessBad(int threshold)
{
    _items.Where(x => x.Value > threshold).ToList(); // аллокация DisplayClass
}

// Оптимизация — передаём threshold явно или используем другой подход
void ProcessGood(int threshold)
{
    // Вариант: цикл вместо LINQ в hot path
    foreach (var item in _items)
    {
        if (item.Value > threshold)
            ProcessItem(item);
    }
}
```

---

## Events

### event keyword — зачем нужен vs обычный delegate

Ключевое слово `event` добавляет **инкапсуляцию** поверх delegate. Снаружи класса можно только `+=` и `-=`, но нельзя вызвать или присвоить.

```csharp
// БЕЗ event — delegate как public поле (ОПАСНО)
class ButtonBad
{
    public Action? Clicked; // любой может вызвать и перезаписать
}

var badButton = new ButtonBad();
badButton.Clicked = null;      // ЗАТЁР все подписки — опасно!
badButton.Clicked();           // Может вызвать извне — опасно!

// С event — безопасная инкапсуляция
class Button
{
    public event Action? Clicked;

    public void SimulateClick() => Clicked?.Invoke();
}

var button = new Button();
button.Clicked += () => Console.WriteLine("Нажата!");
// button.Clicked = null;      // ОШИБКА КОМПИЛЯЦИИ
// button.Clicked();           // ОШИБКА КОМПИЛЯЦИИ
// button.Clicked?.Invoke();   // ОШИБКА КОМПИЛЯЦИИ
button.SimulateClick();        // OK — только изнутри класса
```

### EventHandler и EventHandler\<TEventArgs\>

Стандартный паттерн событий в .NET:

```csharp
// Стандартная сигнатура: (object? sender, EventArgs e)
class FileWatcher
{
    // Без данных
    public event EventHandler? FileDetected;

    // С данными
    public event EventHandler<FileChangedEventArgs>? FileChanged;

    public void Scan(string path)
    {
        // Нашли файл
        FileDetected?.Invoke(this, EventArgs.Empty);

        // Файл изменился
        FileChanged?.Invoke(this, new FileChangedEventArgs(path, DateTime.UtcNow));
    }
}

// Custom EventArgs
sealed class FileChangedEventArgs(string filePath, DateTime changedAt) : EventArgs
{
    public string FilePath { get; } = filePath;
    public DateTime ChangedAt { get; } = changedAt;
}

// Подписка
var watcher = new FileWatcher();
watcher.FileDetected += (sender, e) => Console.WriteLine("Файл обнаружен");
watcher.FileChanged += (sender, e) =>
    Console.WriteLine($"Изменён: {e.FilePath} в {e.ChangedAt:HH:mm:ss}");
```

> [!question]- **Интервью: Event vs delegate — зачем ключевое слово event?**
> `event` ограничивает доступ: подписка/отписка (`+=`/`-=`) — извне. Вызов (`Invoke`) — только изнутри класса. Без `event` любой код может перезаписать делегат (`handler = null`).
>
> **Утечки памяти:** подписчик удерживает ссылку на издателя. Если подписчик живёт дольше — утечка. Решение: отписка в `Dispose`, `WeakEventManager`, `static` handlers.

> [!question]- **Интервью: ref vs out vs in — различия?**
> | Модификатор | Чтение | Запись | Назначение |
> |-------------|--------|--------|------------|
> | `ref` | Да | Да | Двусторонняя передача |
> | `out` | До присвоения — нет | Обязательна | Возврат нескольких значений |
> | `in` | Да | Нет | Большие struct без копирования |
>
> `in` — readonly ref. Полезно для struct > 16 байт в hot path.

### Publish/Subscribe паттерн

```csharp
// Publisher — ничего не знает о подписчиках
sealed class OrderService
{
    public event EventHandler<OrderCreatedEventArgs>? OrderCreated;
    public event EventHandler<OrderCreatedEventArgs>? OrderShipped;

    public void CreateOrder(string product, decimal price)
    {
        // Бизнес-логика создания заказа...
        var args = new OrderCreatedEventArgs(product, price);
        OrderCreated?.Invoke(this, args);
    }

    public void ShipOrder(string product, decimal price)
    {
        var args = new OrderCreatedEventArgs(product, price);
        OrderShipped?.Invoke(this, args);
    }
}

sealed class OrderCreatedEventArgs(string product, decimal price) : EventArgs
{
    public string Product { get; } = product;
    public decimal Price { get; } = price;
}

// Subscribers — подписываются независимо
sealed class EmailNotifier
{
    public void Subscribe(OrderService service)
    {
        service.OrderCreated += OnOrderCreated;
        service.OrderShipped += OnOrderShipped;
    }

    private void OnOrderCreated(object? sender, OrderCreatedEventArgs e) =>
        Console.WriteLine($"[Email] Новый заказ: {e.Product} за {e.Price:C}");

    private void OnOrderShipped(object? sender, OrderCreatedEventArgs e) =>
        Console.WriteLine($"[Email] Заказ отправлен: {e.Product}");
}

sealed class AnalyticsTracker
{
    public void Subscribe(OrderService service)
    {
        service.OrderCreated += (_, e) =>
            Console.WriteLine($"[Analytics] Продажа: {e.Price:C}");
    }
}

// Связываем
var orderService = new OrderService();
var emailNotifier = new EmailNotifier();
var analytics = new AnalyticsTracker();

emailNotifier.Subscribe(orderService);
analytics.Subscribe(orderService);

orderService.CreateOrder("Ноутбук", 85_000m);
// [Email] Новый заказ: Ноутбук за 85 000,00 ₽
// [Analytics] Продажа: 85 000,00 ₽
```

### Custom EventArgs

```csharp
// Record-based EventArgs (C# 9+ — чисто и кратко)
sealed record ProgressEventArgs(int Current, int Total, string? Message = null) : EventArgs
{
    public double Percentage => Total > 0 ? (double)Current / Total * 100 : 0;
}

sealed class DataImporter
{
    public event EventHandler<ProgressEventArgs>? ProgressChanged;
    public event EventHandler<ImportCompletedEventArgs>? Completed;

    public async Task ImportAsync(IReadOnlyList<string> files, CancellationToken ct = default)
    {
        for (int i = 0; i < files.Count; i++)
        {
            ct.ThrowIfCancellationRequested();
            await ProcessFileAsync(files[i]);

            ProgressChanged?.Invoke(this, new(i + 1, files.Count, files[i]));
        }

        Completed?.Invoke(this, new(files.Count, true));
    }

    private Task ProcessFileAsync(string file) => Task.Delay(100); // имитация
}

sealed record ImportCompletedEventArgs(int TotalFiles, bool Success) : EventArgs;
```

### Event accessors: add/remove

Аналог get/set для свойств, но для событий. Позволяют контролировать подписку/отписку.

```csharp
sealed class ThreadSafePublisher
{
    private readonly object _lock = new();
    private EventHandler? _clicked;

    public event EventHandler Clicked
    {
        add
        {
            lock (_lock)
            {
                _clicked += value;
                Console.WriteLine($"Подписчик добавлен. Всего: {_clicked?.GetInvocationList().Length ?? 0}");
            }
        }
        remove
        {
            lock (_lock)
            {
                _clicked -= value;
                Console.WriteLine("Подписчик удалён");
            }
        }
    }

    public void RaiseClick()
    {
        EventHandler? handler;
        lock (_lock)
        {
            handler = _clicked;
        }
        handler?.Invoke(this, EventArgs.Empty);
    }
}
```

### Weak events — предотвращение утечек памяти

Обычный event держит **strong reference** на подписчика, мешая GC его собрать. Это типичная причина утечек памяти, особенно в WPF.

```csharp
// ПРОБЛЕМА: утечка памяти
class LongLivedPublisher
{
    public event EventHandler? DataReady; // Держит strong ref на подписчиков
}

class ShortLivedSubscriber
{
    public ShortLivedSubscriber(LongLivedPublisher publisher)
    {
        publisher.DataReady += OnDataReady; // Подписались — теперь GC не соберёт
    }

    private void OnDataReady(object? sender, EventArgs e) { }

    // Даже если ShortLivedSubscriber больше не используется,
    // publisher держит на него ссылку через event
}

// РЕШЕНИЕ 1: IDisposable + явная отписка
sealed class SafeSubscriber(LongLivedPublisher publisher) : IDisposable
{
    private bool _disposed;

    public void Subscribe() => publisher.DataReady += OnDataReady;

    private void OnDataReady(object? sender, EventArgs e)
    {
        if (_disposed) return;
        Console.WriteLine("Обработка данных");
    }

    public void Dispose()
    {
        if (_disposed) return;
        _disposed = true;
        publisher.DataReady -= OnDataReady;
    }
}

// РЕШЕНИЕ 2: WeakEventManager (WPF)
// using System.Windows;
// WeakEventManager<LongLivedPublisher, EventArgs>
//     .AddHandler(publisher, nameof(LongLivedPublisher.DataReady), OnDataReady);

// РЕШЕНИЕ 3: Собственная реализация через WeakReference
sealed class WeakEvent<TEventArgs> where TEventArgs : EventArgs
{
    private readonly List<WeakReference<EventHandler<TEventArgs>>> _handlers = [];

    public void Subscribe(EventHandler<TEventArgs> handler)
    {
        _handlers.Add(new WeakReference<EventHandler<TEventArgs>>(handler));
    }

    public void Raise(object? sender, TEventArgs args)
    {
        for (int i = _handlers.Count - 1; i >= 0; i--)
        {
            if (_handlers[i].TryGetTarget(out var handler))
            {
                handler(sender, args);
            }
            else
            {
                _handlers.RemoveAt(i); // Подписчик собран GC — убираем
            }
        }
    }
}
```

### Лучшие практики: проверка null, Volatile.Read

```csharp
sealed class SafeEventPublisher
{
    public event EventHandler<MessageEventArgs>? MessageReceived;

    // Способ 1: Null-conditional operator (рекомендуемый с C# 6+)
    // Thread-safe — компилятор копирует delegate перед вызовом
    public void OnMessage(string text) =>
        MessageReceived?.Invoke(this, new MessageEventArgs(text));

    // Способ 2: Явная копия (классический подход)
    public void OnMessageClassic(string text)
    {
        EventHandler<MessageEventArgs>? handler = MessageReceived;
        handler?.Invoke(this, new MessageEventArgs(text));
    }

    // Способ 3: Volatile.Read — максимальная корректность для multi-threaded
    // Требует отдельного backing field (нельзя ref на event напрямую)
    // private EventHandler<MessageEventArgs>? _messageReceived;
    // public event EventHandler<MessageEventArgs>? MessageReceived
    // {
    //     add => _messageReceived += value;
    //     remove => _messageReceived -= value;
    // }
    // public void OnMessageVolatile(string text)
    // {
    //     var handler = Volatile.Read(ref _messageReceived);
    //     handler?.Invoke(this, new MessageEventArgs(text));
    // }
    //
    // Упрощённый вариант (field-like event — компилятор генерирует backing field):
    public void OnMessageVolatile(string text)
    {
        // Для field-like event компилятор автоматически делает thread-safe доступ
        // Достаточно Способа 2 (handler?.Invoke) — он уже thread-safe
        var handler = MessageReceived;
        handler?.Invoke(this, new MessageEventArgs(text));
    }

    // НЕПРАВИЛЬНО: race condition
    // public void OnMessageBad(string text)
    // {
    //     if (MessageReceived != null)         // Может стать null тут
    //         MessageReceived(this, new(...)); // NullReferenceException!
    // }
}

sealed record MessageEventArgs(string Text) : EventArgs;
```

**Правила для событий:**

```csharp
// 1. Метод-обёртка OnXxx — protected virtual для наследования
class BaseControl
{
    public event EventHandler? Clicked;

    protected virtual void OnClicked() =>
        Clicked?.Invoke(this, EventArgs.Empty);
}

class CustomButton : BaseControl
{
    protected override void OnClicked()
    {
        Console.WriteLine("CustomButton pre-processing");
        base.OnClicked();
    }
}

// 2. Всегда отписывайся в Dispose
// 3. Не делай event async void (потеряешь исключения)
// 4. Используй EventHandler<T>, а не свой delegate
```

---

## Практические паттерны

### Strategy pattern через delegates

Delegates — самый лёгкий способ реализовать Strategy без интерфейсов и классов.

```csharp
// Стратегия ценообразования через Func<T>
sealed class PricingService
{
    public decimal CalculatePrice(
        decimal basePrice,
        Func<decimal, decimal> discountStrategy,
        Func<decimal, decimal> taxStrategy)
    {
        decimal discounted = discountStrategy(basePrice);
        return taxStrategy(discounted);
    }
}

// Стратегии — просто функции
static class DiscountStrategies
{
    public static Func<decimal, decimal> NoDiscount => price => price;
    public static Func<decimal, decimal> Percentage(decimal percent) =>
        price => price * (1 - percent / 100);
    public static Func<decimal, decimal> Fixed(decimal amount) =>
        price => Math.Max(0, price - amount);
    public static Func<decimal, decimal> BlackFriday =>
        price => price < 1000 ? price * 0.8m : price * 0.7m;
}

static class TaxStrategies
{
    public static Func<decimal, decimal> Russia => price => price * 1.20m;
    public static Func<decimal, decimal> NoTax => price => price;
}

// Использование — комбинируем стратегии как хотим
var pricing = new PricingService();

decimal finalPrice = pricing.CalculatePrice(
    basePrice: 5000m,
    discountStrategy: DiscountStrategies.Percentage(15),
    taxStrategy: TaxStrategies.Russia);

Console.WriteLine($"Итого: {finalPrice:C}"); // 5 100,00 ₽
```

### Callback pattern

```csharp
// Callback при завершении операции
sealed class FileDownloader
{
    public async Task DownloadAsync(
        string url,
        Action<long, long>? onProgress = null,
        Action<string>? onCompleted = null,
        Action<Exception>? onError = null)
    {
        try
        {
            long totalBytes = 10_000;
            for (long downloaded = 0; downloaded < totalBytes; downloaded += 1000)
            {
                await Task.Delay(50); // имитация загрузки
                onProgress?.Invoke(downloaded + 1000, totalBytes);
            }

            string savedPath = $"/downloads/{Guid.NewGuid()}.dat";
            onCompleted?.Invoke(savedPath);
        }
        catch (Exception ex)
        {
            onError?.Invoke(ex);
        }
    }
}

// Вызов с callback
var downloader = new FileDownloader();
await downloader.DownloadAsync(
    url: "https://example.com/file.zip",
    onProgress: (current, total) =>
        Console.Write($"\rЗагрузка: {current * 100 / total}%"),
    onCompleted: path =>
        Console.WriteLine($"\nСохранено: {path}"),
    onError: ex =>
        Console.WriteLine($"\nОшибка: {ex.Message}"));
```

### Observer pattern через events

```csharp
// Полноценный Observer через events (без интерфейсов IObservable/IObserver)
sealed class StockTicker
{
    public event EventHandler<StockPriceChangedEventArgs>? PriceChanged;

    private readonly Dictionary<string, decimal> _prices = [];

    public void UpdatePrice(string symbol, decimal newPrice)
    {
        decimal oldPrice = _prices.GetValueOrDefault(symbol);
        _prices[symbol] = newPrice;

        if (oldPrice != newPrice)
        {
            PriceChanged?.Invoke(this, new(symbol, oldPrice, newPrice));
        }
    }
}

sealed record StockPriceChangedEventArgs(
    string Symbol,
    decimal OldPrice,
    decimal NewPrice) : EventArgs
{
    public decimal Change => NewPrice - OldPrice;
    public decimal ChangePercent => OldPrice != 0 ? Change / OldPrice * 100 : 0;
}

// Observer 1: Логгер
sealed class PriceLogger : IDisposable
{
    private readonly StockTicker _ticker;

    public PriceLogger(StockTicker ticker)
    {
        _ticker = ticker;
        _ticker.PriceChanged += OnPriceChanged;
    }

    private void OnPriceChanged(object? sender, StockPriceChangedEventArgs e) =>
        Console.WriteLine(
            $"[LOG] {e.Symbol}: {e.OldPrice:F2} -> {e.NewPrice:F2} ({e.ChangePercent:+0.00;-0.00}%)");

    public void Dispose() => _ticker.PriceChanged -= OnPriceChanged;
}

// Observer 2: Алерт при большом изменении
sealed class PriceAlert(StockTicker ticker, decimal thresholdPercent) : IDisposable
{
    private bool _subscribed;

    public void Start()
    {
        if (_subscribed) return;
        ticker.PriceChanged += OnPriceChanged;
        _subscribed = true;
    }

    private void OnPriceChanged(object? sender, StockPriceChangedEventArgs e)
    {
        if (Math.Abs(e.ChangePercent) >= thresholdPercent)
        {
            Console.WriteLine(
                $"[ALERT] {e.Symbol} изменилась на {e.ChangePercent:F2}%!");
        }
    }

    public void Dispose()
    {
        ticker.PriceChanged -= OnPriceChanged;
        _subscribed = false;
    }
}

// Сборка
var ticker = new StockTicker();
using var logger = new PriceLogger(ticker);
using var alert = new PriceAlert(ticker, thresholdPercent: 5m);
alert.Start();

ticker.UpdatePrice("AAPL", 150.00m);
ticker.UpdatePrice("AAPL", 160.00m); // +6.67% — сработает алерт
ticker.UpdatePrice("GOOGL", 2800.00m);
ticker.UpdatePrice("GOOGL", 2810.00m); // +0.36% — без алерта
```

### Функциональные цепочки: Func\<T, T\> pipeline

```csharp
// Pipeline из функций — каждая трансформирует данные
static class Pipeline
{
    public static Func<T, T> Compose<T>(params Func<T, T>[] steps)
    {
        return input =>
        {
            T result = input;
            foreach (var step in steps)
                result = step(result);
            return result;
        };
    }
}

// Строковый pipeline
Func<string, string> processText = Pipeline.Compose<string>(
    s => s.Trim(),
    s => s.ToLowerInvariant(),
    s => System.Text.RegularExpressions.Regex.Replace(s, @"\s+", " "),
    s => System.Globalization.CultureInfo.CurrentCulture.TextInfo.ToTitleCase(s)
);

string result = processText("  hello   WORLD   from   C#  ");
Console.WriteLine(result); // "Hello World From C#"

// Числовой pipeline
Func<decimal, decimal> calculateTotal = Pipeline.Compose<decimal>(
    price => price * 0.9m,     // скидка 10%
    price => price + 500m,     // доставка
    price => price * 1.20m,    // НДС 20%
    price => Math.Round(price, 2)
);

Console.WriteLine(calculateTotal(10_000m)); // 11 400,00

// Extension method подход — fluent API
static class FuncExtensions
{
    public static Func<T, T> Then<T>(this Func<T, T> first, Func<T, T> next)
    {
        return input => next(first(input));
    }
}

// Fluent pipeline
Func<int, int> transform = ((Func<int, int>)(x => x * 2))
    .Then(x => x + 10)
    .Then(x => x * x);

Console.WriteLine(transform(3)); // (3*2 + 10)^2 = 256
```

**Async pipeline:**

```csharp
static class AsyncPipeline
{
    public static Func<T, Task<T>> Compose<T>(params Func<T, Task<T>>[] steps)
    {
        return async input =>
        {
            T result = input;
            foreach (var step in steps)
                result = await step(result).ConfigureAwait(false);
            return result;
        };
    }
}

// Пример: обработка заказа как async pipeline
var processOrder = AsyncPipeline.Compose<Order>(
    async order => { await ValidateAsync(order); return order; },
    async order => { order.Discount = await CalculateDiscountAsync(order); return order; },
    async order => { await SaveToDbAsync(order); return order; },
    async order => { await SendConfirmationEmailAsync(order); return order; }
);

// Order order = await processOrder(new Order { ... });
```

---

## Сводная таблица

| Концепт | Когда использовать |
|---|---|
| `Action<T>` | Void callback, логирование, side effects |
| `Func<T, TResult>` | Трансформация, стратегия, predicate |
| `Predicate<T>` | Фильтрация коллекций (`List<T>.FindAll`) |
| `event EventHandler<T>` | Pub/Sub, уведомления, UI events |
| Custom delegate | `ref`/`out` параметры, именованная семантика |
| Static lambda | Hot path без аллокаций closure |
| Expression lambda | Краткие однострочные трансформации |
| Statement lambda | Сложная логика с ветвлениями |

---

## См. также

- [Типы и память](types-and-memory.md)
- [ООП и классы](oop.md)
- [Error Handling](error-handling.md)
- [Async и потоки](async-threading.md)
