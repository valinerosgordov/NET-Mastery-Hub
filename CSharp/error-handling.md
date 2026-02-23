---
tags: [exceptions, error-handling, strings, json, regex]
level: Senior
---

# Обработка ошибок, строки и I/O

> Справочник по exception handling, строкам, I/O, JSON и Regex. C# 13 / .NET 9.
> Теория → практика → senior-level код → вопросы интервью.

---

## Exception Handling

### try / catch / finally — поток выполнения

Базовый механизм обработки ошибок. `finally` выполняется **всегда** — даже при `return` из `try`.

```csharp
try
{
    int result = int.Parse("not_a_number");
    Console.WriteLine(result);
}
catch (FormatException ex)
{
    // Ловим конкретный тип — ПРАВИЛЬНО
    Console.WriteLine($"Ошибка формата: {ex.Message}");
}
catch (Exception ex)
{
    // Общий catch — только в конце цепочки
    Console.WriteLine($"Неизвестная ошибка: {ex.Message}");
}
finally
{
    // Выполняется ВСЕГДА: и при успехе, и при ошибке, и при return
    Console.WriteLine("Очистка ресурсов");
}
```

Порядок выполнения при ошибке:
```
1. try → выполняется до строки с ошибкой
2. catch → первый подходящий по типу (сверху вниз)
3. finally → всегда
```

### Exception Hierarchy

```
                        Exception
                           |
              +------------+------------+
              |                         |
      SystemException           ApplicationException (устарел)
              |
  +-----------+-----------+-----------+
  |           |           |           |
ArgumentException  InvalidOp...  IOException
  |                                   |
ArgumentNullException          FileNotFoundException
ArgumentOutOfRangeException    DirectoryNotFoundException
```

### Common Exceptions — когда какой бросать

```csharp
// ArgumentNullException — аргумент null, а не должен быть
public decimal CalculateDiscount(Order order)
{
    ArgumentNullException.ThrowIfNull(order);           // .NET 6+
    ArgumentException.ThrowIfNullOrEmpty(order.Id);     // .NET 7+ (ArgumentException, не ArgumentNullException!)
    return order.Total * 0.1m;
}

// ArgumentException — аргумент невалидный (не null, но неправильный)
public void SetAge(int age)
{
    ArgumentOutOfRangeException.ThrowIfNegativeOrZero(age); // .NET 8+
    ArgumentOutOfRangeException.ThrowIfGreaterThan(age, 150);
    _age = age;
}

// InvalidOperationException — объект в неправильном состоянии
public void Complete()
{
    if (_status == OrderStatus.Cancelled)
        throw new InvalidOperationException(
            $"Нельзя завершить отменённый заказ. Текущий статус: {_status}");
    _status = OrderStatus.Completed;
}

// NotSupportedException — операция не поддерживается типом
public override void Write(byte[] buffer, int offset, int count)
    => throw new NotSupportedException("Поток только для чтения");

// KeyNotFoundException — ключ не найден в коллекции
public User GetUser(string id)
    => _users.TryGetValue(id, out var user)
        ? user
        : throw new KeyNotFoundException($"Пользователь '{id}' не найден");

// NotImplementedException — заглушка (TODO в коде)
public decimal CalculateTax()
    => throw new NotImplementedException("TODO: реализовать расчёт налога");
```

### throw vs throw ex — сохранение Stack Trace

**КРИТИЧЕСКИ ВАЖНО:** `throw ex` теряет оригинальный stack trace.

```csharp
try
{
    SomeRiskyOperation();
}
catch (Exception ex)
{
    LogError(ex);
    throw;      // ПРАВИЛЬНО: сохраняет оригинальный stack trace
    // throw ex; // НЕПРАВИЛЬНО: stack trace начнётся с этой строки
}
```

**ExceptionDispatchInfo** — повторный throw с полным сохранением стека:

```csharp
using System.Runtime.ExceptionServices;

ExceptionDispatchInfo? capturedException = null;

try
{
    await ProcessPaymentAsync();
}
catch (Exception ex)
{
    capturedException = ExceptionDispatchInfo.Capture(ex);
}

// ... позже, может быть даже в другом методе
if (capturedException is not null)
{
    capturedException.Throw(); // Полный оригинальный stack trace сохранён
}
```

> [!question]- **Интервью: throw vs throw ex — что теряется?**
> `throw;` — перебрасывает исключение с оригинальным stack trace. `throw ex;` — сбрасывает stack trace на текущую позицию, теряя информацию о месте возникновения.
>
> **Правило:** всегда `throw;` в catch. `throw new WrapperException("msg", ex);` для обёртки с сохранением inner exception. `ExceptionDispatchInfo.Capture(ex).Throw()` — для отложенного перебрасывания.

### Exception Filters — `when`

Фильтры проверяются **до** раскрутки стека. Если `when` вернул `false`, catch пропускается.

```csharp
try
{
    await httpClient.GetAsync(url, cancellationToken);
}
catch (HttpRequestException ex) when (ex.StatusCode == HttpStatusCode.NotFound)
{
    return Result.Failure<Order>("Заказ не найден");
}
catch (HttpRequestException ex) when (ex.StatusCode == HttpStatusCode.TooManyRequests)
{
    await Task.Delay(TimeSpan.FromSeconds(5));
    return await RetryAsync();
}
catch (Exception ex) when (ex is not OperationCanceledException)
{
    // Ловим всё, КРОМЕ отмены — пусть OperationCanceledException пролетит выше
    _logger.LogError(ex, "Ошибка HTTP-запроса");
    return Result.Failure<Order>(ex.Message);
}

// Фильтр для логирования без перехвата (when всегда false)
catch (Exception ex) when (LogException(ex))
{
    // Сюда никогда не попадём
}

static bool LogException(Exception ex)
{
    Console.Error.WriteLine(ex);
    return false; // Всегда false → catch не срабатывает, exception летит дальше
}
```

### Custom Exceptions — правильное создание

```csharp
/// <summary>
/// Доменное исключение: недостаточно средств на счёте.
/// </summary>
public sealed class InsufficientFundsException : Exception
{
    public decimal CurrentBalance { get; }
    public decimal RequestedAmount { get; }

    public InsufficientFundsException(decimal currentBalance, decimal requestedAmount)
        : base($"Недостаточно средств. Баланс: {currentBalance:C}, запрошено: {requestedAmount:C}")
    {
        CurrentBalance = currentBalance;
        RequestedAmount = requestedAmount;
    }

    public InsufficientFundsException(string message) : base(message) { }

    public InsufficientFundsException(string message, Exception innerException)
        : base(message, innerException) { }
}

// Использование
public void Withdraw(decimal amount)
{
    if (amount > Balance)
        throw new InsufficientFundsException(Balance, amount);

    Balance -= amount;
}
```

### Best Practices — итоговая таблица

| Правило | Пример |
|---------|--------|
| Ловить конкретные типы | `catch (FormatException)`, не `catch (Exception)` |
| Не использовать exceptions для control flow | Использовать `TryParse`, `TryGetValue`, `Result<T>` |
| Всегда включать inner exception | `throw new AppException("msg", ex)` |
| `throw`, не `throw ex` | Сохраняет stack trace |
| `finally` для очистки | Или лучше — `using` / `await using` |
| Exception filters вместо catch + rethrow | `catch (Exception ex) when (ShouldHandle(ex))` |
| `ArgumentNullException.ThrowIfNull()` | Вместо ручных `if (x is null) throw` |

> [!question]- **Интервью: Когда создавать кастомные исключения?**
> Когда стандартные не передают бизнес-смысл: `OrderNotFoundException`, `InsufficientBalanceException`. Наследуй от `Exception` (не `ApplicationException`). Конструкторы с `message` и `innerException`. Не используй исключения для flow control — stack unwinding дорог.

---

## Strings (углублённо)

### Immutability и String Pool

Строки в C# **неизменяемы** (immutable). Любая «модификация» создаёт **новый** объект.

```csharp
string greeting = "Hello";
string modified = greeting.Replace("Hello", "Hi"); // Новый объект в heap
// greeting по-прежнему "Hello"

// String interning — .NET автоматически интернирует литералы
string a = "hello";
string b = "hello";
Console.WriteLine(ReferenceEquals(a, b)); // True — один объект в памяти

// Ручное интернирование
string dynamic1 = new string(['h', 'e', 'l', 'l', 'o']);
string dynamic2 = new string(['h', 'e', 'l', 'l', 'o']);
Console.WriteLine(ReferenceEquals(dynamic1, dynamic2));               // False
Console.WriteLine(ReferenceEquals(string.Intern(dynamic1),
                                  string.Intern(dynamic2)));          // True

// Проверка, интернирована ли строка
bool isInterned = string.IsInterned(dynamic1) is not null;
```

### StringBuilder — когда использовать

**Правило:** если конкатенация строк в цикле или > 3–4 операций — используй `StringBuilder`.

```csharp
// ПЛОХО: каждая += создаёт новый объект (O(n²) по памяти)
string result = "";
for (int i = 0; i < 10_000; i++)
    result += i.ToString();

// ХОРОШО: StringBuilder — мутабельный буфер (O(n))
var sb = new StringBuilder(capacity: 64_000); // Указываем capacity заранее
for (int i = 0; i < 10_000; i++)
    sb.Append(i);

string output = sb.ToString();

// Fluent API
string html = new StringBuilder(256)
    .Append("<div class=\"card\">")
    .Append("<h2>").Append(title).Append("</h2>")
    .Append("<p>").Append(description).Append("</p>")
    .Append("</div>")
    .ToString();
```

### ReadOnlySpan<char> — парсинг без аллокаций

```csharp
// Парсинг строки "2024-01-15" без создания подстрок
ReadOnlySpan<char> date = "2024-01-15".AsSpan();

int year  = int.Parse(date[..4]);       // "2024" — без аллокации
int month = int.Parse(date[5..7]);      // "01"
int day   = int.Parse(date[8..10]);     // "15"

// Разделение по токенам без аллокаций
ReadOnlySpan<char> csv = "Alice,30,Developer".AsSpan();
int firstComma = csv.IndexOf(',');
ReadOnlySpan<char> name = csv[..firstComma]; // "Alice" — zero-alloc

// Сравнение без аллокаций
bool startsWith = csv.StartsWith("Alice");
bool contains = csv.Contains("Developer", StringComparison.Ordinal);
```

### StringComparison — производительность и корректность

```csharp
string a = "hello";
string b = "HELLO";

// Ordinal — побайтовое сравнение, САМОЕ БЫСТРОЕ
a.Equals(b, StringComparison.Ordinal);                // false
a.Equals(b, StringComparison.OrdinalIgnoreCase);      // true — для case-insensitive

// CurrentCulture — учитывает локаль (медленнее, для UI)
a.Equals(b, StringComparison.CurrentCultureIgnoreCase); // true

// ПРАВИЛО: для внутренней логики — Ordinal, для отображения пользователю — CurrentCulture
// Для Dictionary ключей:
var dict = new Dictionary<string, int>(StringComparer.OrdinalIgnoreCase);
dict["Hello"] = 1;
Console.WriteLine(dict["hello"]); // 1 — регистронезависимый поиск
```

### Raw String Literals (C# 11) и UTF-8 Literals

```csharp
// Raw string literals — без экранирования
string json = """
    {
        "name": "Alice",
        "age": 30,
        "tags": ["dev", "lead"]
    }
    """;

// Интерполяция в raw strings (количество $ = количество { для интерполяции)
string name = "Alice";
string template = $$"""
    {
        "name": "{{name}}",
        "query": "SELECT * FROM users WHERE name = '{name}'"
    }
    """;

// UTF-8 string literals (C# 11) — возвращает ReadOnlySpan<byte>
ReadOnlySpan<byte> utf8 = "Content-Type: application/json"u8;
// Без аллокации строки — идеально для HTTP headers, протоколов
```

### String.Create — оптимизация создания строк

```csharp
// Создаёт строку фиксированной длины, заполняя через Span (одна аллокация)
string hex = string.Create(8, 0xDEADBEEF, static (span, value) =>
{
    for (int i = span.Length - 1; i >= 0; i--)
    {
        int nibble = (int)(value & 0xF);
        span[i] = (char)(nibble < 10 ? '0' + nibble : 'A' + nibble - 10);
        value >>= 4;
    }
});
// hex == "DEADBEEF"

// Генерация случайной строки без промежуточных аллокаций
string randomId = string.Create(16, Random.Shared, static (span, rng) =>
{
    const string chars = "abcdefghijklmnopqrstuvwxyz0123456789";
    for (int i = 0; i < span.Length; i++)
        span[i] = chars[rng.Next(chars.Length)];
});
```

### Основные методы строк

```csharp
string text = "  Hello, World! Hello, C#!  ";

// Поиск
bool has      = text.Contains("World");                          // true
int idx       = text.IndexOf("Hello");                           // 2
int lastIdx   = text.LastIndexOf("Hello");                       // 17
bool starts   = text.TrimStart().StartsWith("Hello");            // true
bool ends     = text.TrimEnd().EndsWith("C#!");                  // true

// Модификация (каждый метод возвращает НОВУЮ строку)
string replaced = text.Replace("Hello", "Hi");                   // "  Hi, World! Hi, C#!  "
string trimmed  = text.Trim();                                   // "Hello, World! Hello, C#!"
string upper    = text.ToUpperInvariant();                        // "  HELLO, WORLD! HELLO, C#!  "
string lower    = text.ToLowerInvariant();                        // "  hello, world! hello, c#!  "
string sub      = "Hello, World!".Substring(7, 5);               // "World" (legacy)
string sub2     = "Hello, World!"[7..12];                        // "World" (modern — Range)

// Разделение и объединение
string[] parts  = "one,two,three".Split(',');                    // ["one", "two", "three"]
string[] parts2 = "one,,two".Split(',', StringSplitOptions.RemoveEmptyEntries); // ["one", "two"]
string joined   = string.Join(" | ", parts);                     // "one | two | three"

// Проверки
bool empty      = string.IsNullOrEmpty("");                      // true
bool blank      = string.IsNullOrWhiteSpace("   ");              // true
```

### String Formatting

```csharp
string name = "Alice";
decimal price = 1234.5m;
DateTime now = DateTime.Now;

// Интерполяция — предпочтительный способ
string msg1 = $"Привет, {name}! Цена: {price:C2}. Дата: {now:yyyy-MM-dd}";

// Выравнивание в интерполяции
string aligned = $"|{"Name",-20}|{"Price",10:C2}|";
// |Name                |  $1,234.50|

// String.Format (для шаблонов из ресурсов)
string msg2 = string.Format("Пользователь {0} заплатил {1:C2}", name, price);

// Числовые форматы
$"{1234567:N0}"    // "1,234,567"      — с разделителями
$"{0.856:P1}"      // "85.6%"          — проценты
$"{255:X2}"        // "FF"             — hex
$"{42:D5}"         // "00042"          — дополнение нулями
```

### Char — работа с символами

```csharp
char c = 'A';

char.IsDigit('5');           // true
char.IsLetter('A');          // true
char.IsWhiteSpace(' ');      // true
char.IsUpper('A');           // true
char.IsLower('a');           // true
char.IsLetterOrDigit('3');   // true
char.IsPunctuation('.');     // true

char upper = char.ToUpperInvariant('a');  // 'A'
char lower = char.ToLowerInvariant('Z');  // 'z'
int numeric = (int)char.GetNumericValue('7'); // 7
```

---

## File I/O

### File — статические методы (быстрый доступ)

```csharp
string path = @"C:\Data\config.txt";

// Запись
File.WriteAllText(path, "Hello, World!");
File.WriteAllLines(path, ["line1", "line2", "line3"]);
File.AppendAllText(path, "\nnew line");

// Чтение
string content      = File.ReadAllText(path);
string[] lines      = File.ReadAllLines(path);
byte[] bytes        = File.ReadAllBytes(path);

// Проверки
bool exists         = File.Exists(path);
DateTime created    = File.GetCreationTime(path);
DateTime modified   = File.GetLastWriteTime(path);

// Операции
File.Copy(path, @"C:\Data\config_backup.txt", overwrite: true);
File.Move(path, @"C:\Data\config_new.txt", overwrite: true);
File.Delete(path);
```

### Async File Operations

```csharp
// ВСЕГДА используй async для I/O в ASP.NET Core / GUI приложениях
string content = await File.ReadAllTextAsync("data.json", cancellationToken);
string[] lines = await File.ReadAllLinesAsync("log.txt", cancellationToken);

await File.WriteAllTextAsync("output.txt", result, cancellationToken);
await File.WriteAllLinesAsync("output.txt", lines, cancellationToken);
await File.WriteAllBytesAsync("image.png", imageBytes, cancellationToken);

// Append
await File.AppendAllTextAsync("log.txt", $"{DateTime.UtcNow}: событие\n");
```

### FileInfo — instance-based

```csharp
var fileInfo = new FileInfo(@"C:\Data\report.pdf");

if (fileInfo.Exists)
{
    Console.WriteLine($"Имя:       {fileInfo.Name}");           // report.pdf
    Console.WriteLine($"Размер:    {fileInfo.Length} байт");    // 1234567
    Console.WriteLine($"Каталог:   {fileInfo.DirectoryName}");  // C:\Data
    Console.WriteLine($"Расш.:     {fileInfo.Extension}");      // .pdf
    Console.WriteLine($"Создан:    {fileInfo.CreationTime}");
    Console.WriteLine($"Изменён:   {fileInfo.LastWriteTime}");
    Console.WriteLine($"ReadOnly:  {fileInfo.IsReadOnly}");

    fileInfo.CopyTo(@"C:\Backup\report.pdf", overwrite: true);
}
```

### Directory и DirectoryInfo

```csharp
string dir = @"C:\Projects\MyApp";

// Directory (static)
Directory.CreateDirectory(dir);                         // Создаёт вложенные каталоги
bool dirExists      = Directory.Exists(dir);
string[] files      = Directory.GetFiles(dir, "*.cs");
string[] subDirs    = Directory.GetDirectories(dir);
string[] allCsFiles = Directory.GetFiles(dir, "*.cs", SearchOption.AllDirectories);

// EnumerateFiles — lazy, не загружает всё в память
foreach (string file in Directory.EnumerateFiles(dir, "*.log", SearchOption.AllDirectories))
{
    Console.WriteLine(file);
}

Directory.Move(@"C:\Old", @"C:\New");
Directory.Delete(dir, recursive: true); // recursive удаляет содержимое

// DirectoryInfo (instance)
var dirInfo = new DirectoryInfo(dir);
FileInfo[] csFiles = dirInfo.GetFiles("*.cs", SearchOption.AllDirectories);
DirectoryInfo[] subDirectories = dirInfo.GetDirectories();
```

### Path — безопасная работа с путями

```csharp
// ВСЕГДА используй Path.Combine — он правильно обрабатывает разделители
string full = Path.Combine("C:", "Users", "mos26", "file.txt");
// C:\Users\mos26\file.txt

Path.GetFileName(@"C:\Data\report.pdf");       // "report.pdf"
Path.GetFileNameWithoutExtension(@"report.pdf"); // "report"
Path.GetExtension(@"report.pdf");               // ".pdf"
Path.GetDirectoryName(@"C:\Data\report.pdf");   // "C:\Data"
Path.GetFullPath("relative/path.txt");          // Абсолютный путь

Path.GetTempPath();                             // Путь к temp-каталогу
Path.GetTempFileName();                         // Создаёт временный файл
Path.GetRandomFileName();                       // "a3b2c1d4.tmp" (не создаёт файл)

// .NET 8+
Path.Exists(@"C:\Data\file.txt");               // Проверяет и файл, и каталог
```

### StreamReader / StreamWriter — для больших файлов

```csharp
// Чтение построчно — не загружает весь файл в память
await using var reader = new StreamReader("large_file.csv", Encoding.UTF8);
while (await reader.ReadLineAsync() is { } line)
{
    ProcessLine(line);
}

// Запись
await using var writer = new StreamWriter("output.txt", append: false, Encoding.UTF8);
await writer.WriteLineAsync("Первая строка");
await writer.WriteLineAsync("Вторая строка");
await writer.FlushAsync(); // Принудительный сброс буфера
```

### FileStream — binary данные

```csharp
// Запись binary
await using var writeStream = new FileStream(
    "data.bin",
    FileMode.Create,
    FileAccess.Write,
    FileShare.None,
    bufferSize: 4096,
    useAsync: true);

byte[] data = [0x48, 0x65, 0x6C, 0x6C, 0x6F];
await writeStream.WriteAsync(data, cancellationToken);

// Чтение binary
await using var readStream = new FileStream("data.bin", FileMode.Open, FileAccess.Read,
    FileShare.Read, bufferSize: 4096, useAsync: true);

byte[] buffer = new byte[(int)readStream.Length]; // Length — long, нужен каст
int bytesRead = await readStream.ReadAsync(buffer, cancellationToken);
```

### using и IDisposable

```csharp
// using statement — автоматический Dispose() в finally
using (var reader = new StreamReader("file.txt"))
{
    string content = await reader.ReadToEndAsync();
} // reader.Dispose() вызывается здесь

// using declaration (C# 8) — Dispose при выходе из scope
using var writer = new StreamWriter("log.txt", append: true);
await writer.WriteLineAsync("log entry");
// writer.Dispose() при выходе из метода

// await using — для IAsyncDisposable
await using var dbConnection = new NpgsqlConnection(connectionString);
await dbConnection.OpenAsync(cancellationToken);

// Свой IDisposable
public sealed class TempFileScope : IDisposable
{
    public string FilePath { get; } = Path.GetTempFileName();

    public void Dispose()
    {
        if (File.Exists(FilePath))
            File.Delete(FilePath);
    }
}

// Использование
using var temp = new TempFileScope();
await File.WriteAllTextAsync(temp.FilePath, "временные данные");
// Файл автоматически удалится
```

---

## JSON

### System.Text.Json — сериализация / десериализация

```csharp
using System.Text.Json;
using System.Text.Json.Serialization;

// Базовая сериализация
var order = new Order("ORD-001", "Alice", 299.99m);
string json = JsonSerializer.Serialize(order);

// Десериализация
Order? deserialized = JsonSerializer.Deserialize<Order>(json);

// С настройками
var options = new JsonSerializerOptions
{
    PropertyNamingPolicy   = JsonNamingPolicy.CamelCase,    // camelCase в JSON
    WriteIndented          = true,                           // Красивый вывод
    PropertyNameCaseInsensitive = true,                      // Нечувствителен к регистру при чтении
    DefaultIgnoreCondition = JsonIgnoreCondition.WhenWritingNull, // Пропускать null
    Converters             = { new JsonStringEnumConverter() },   // Enum как строки
};

string prettyJson = JsonSerializer.Serialize(order, options);
Order? parsed     = JsonSerializer.Deserialize<Order>(prettyJson, options);
```

### Атрибуты

```csharp
public sealed record OrderDto
{
    [JsonPropertyName("order_id")]           // Имя в JSON
    public required string Id { get; init; }

    [JsonPropertyName("customer_name")]
    public required string Customer { get; init; }

    [JsonIgnore]                              // Не сериализуется
    public string InternalCode { get; init; } = "";

    [JsonInclude]                             // Включить private/internal поле
    internal decimal discount = 0.1m;

    [JsonConverter(typeof(JsonStringEnumConverter))]
    public OrderStatus Status { get; init; }

    [JsonIgnore(Condition = JsonIgnoreCondition.WhenWritingDefault)]
    public int Priority { get; init; }        // Пропускается если 0
}
```

### Source Generators — [JsonSerializable] (AOT-friendly)

```csharp
// Генерация сериализатора на этапе компиляции — быстрее, без reflection
[JsonSerializable(typeof(OrderDto))]
[JsonSerializable(typeof(List<OrderDto>))]
[JsonSourceGenerationOptions(
    PropertyNamingPolicy = JsonKnownNamingPolicy.CamelCase,
    WriteIndented = true,
    DefaultIgnoreCondition = JsonIgnoreCondition.WhenWritingNull)]
public partial class AppJsonContext : JsonSerializerContext;

// Использование
string json = JsonSerializer.Serialize(order, AppJsonContext.Default.OrderDto);
OrderDto? result = JsonSerializer.Deserialize(json, AppJsonContext.Default.OrderDto);

// В ASP.NET Core Minimal API
builder.Services.ConfigureHttpJsonOptions(options =>
{
    options.SerializerOptions.TypeInfoResolverChain.Insert(0, AppJsonContext.Default);
});
```

> [!question]- **Интервью: System.Text.Json vs Newtonsoft — когда что?**
> **STJ** — встроен, быстрее, меньше аллокаций, Source Generators для AOT. По умолчанию для новых проектов.
>
> **Newtonsoft** — богаче фичами (JToken, JsonPath, сложный маппинг). Для legacy или когда STJ не покрывает сценарий.
>
> **Правило:** STJ для API (DTO). Newtonsoft — для сложных трансформаций JSON.

### JsonDocument — DOM-based parsing

```csharp
// Для чтения JSON без десериализации в объект
string json = """{"name": "Alice", "scores": [95, 87, 92], "address": {"city": "Moscow"}}""";

using JsonDocument doc = JsonDocument.Parse(json);
JsonElement root = doc.RootElement;

string name = root.GetProperty("name").GetString()!;           // "Alice"
int firstScore = root.GetProperty("scores")[0].GetInt32();     // 95
string city = root.GetProperty("address").GetProperty("city").GetString()!; // "Moscow"

// Безопасный доступ
if (root.TryGetProperty("age", out JsonElement ageElement))
{
    int age = ageElement.GetInt32();
}

// Перечисление свойств
foreach (JsonProperty prop in root.EnumerateObject())
{
    Console.WriteLine($"{prop.Name}: {prop.Value.ValueKind}");
}
```

### Utf8JsonReader / Utf8JsonWriter — high-performance

```csharp
// Utf8JsonWriter — минимальные аллокации
using var stream = new MemoryStream();
using var writer = new Utf8JsonWriter(stream, new JsonWriterOptions { Indented = true }); // IDisposable, не IAsyncDisposable

writer.WriteStartObject();
writer.WriteString("name", "Alice");
writer.WriteNumber("age", 30);
writer.WriteStartArray("tags");
writer.WriteStringValue("developer");
writer.WriteStringValue("lead");
writer.WriteEndArray();
writer.WriteEndObject();
await writer.FlushAsync();

string result = Encoding.UTF8.GetString(stream.ToArray());

// Utf8JsonReader — потоковое чтение
byte[] jsonBytes = Encoding.UTF8.GetBytes("""{"name":"Alice","age":30}""");
var reader2 = new Utf8JsonReader(jsonBytes);

while (reader2.Read())
{
    if (reader2.TokenType == JsonTokenType.PropertyName && reader2.ValueTextEquals("name"u8))
    {
        reader2.Read();
        string value = reader2.GetString()!; // "Alice"
    }
}
```

### System.Text.Json vs Newtonsoft.Json

| Критерий | System.Text.Json | Newtonsoft.Json |
|----------|------------------|----------------|
| Производительность | Быстрее в 1.5–2x | Медленнее |
| Аллокации | Меньше (Utf8-native) | Больше |
| AOT / Trimming | Source generators | Не поддерживает |
| Гибкость | Ограниченная | Максимальная |
| `$type` polymorphism | `[JsonDerivedType]` (.NET 7+) | `TypeNameHandling` |
| Циклические ссылки | `ReferenceHandler.Preserve` | `PreserveReferencesHandling` |
| LINQ to JSON | `JsonNode` | `JObject` / `JArray` |
| Зрелость | Развивается | Стабильна, feature-complete |
| Рекомендация | **По умолчанию для новых проектов** | Legacy / сложные сценарии |

---

## Regex

### Базовое использование

```csharp
using System.Text.RegularExpressions;

string input = "Телефон: +7-999-123-4567, email: alice@example.com";

// Проверка совпадения
bool hasPhone = Regex.IsMatch(input, @"\+7-\d{3}-\d{3}-\d{4}");  // true

// Поиск первого совпадения
Match match = Regex.Match(input, @"[\w.+-]+@[\w-]+\.[\w.]+");
if (match.Success)
    Console.WriteLine(match.Value); // "alice@example.com"

// Все совпадения
MatchCollection matches = Regex.Matches(input, @"\d+");
foreach (Match m in matches)
    Console.WriteLine(m.Value); // "7", "999", "123", "4567"

// Замена
string masked = Regex.Replace(input, @"\d", "*");
// "Телефон: +*-***-***-****, email: alice@example.com"

// Разделение
string[] tokens = Regex.Split("one, two,  three", @",\s*");
// ["one", "two", "three"]
```

### Группы и именованные группы

```csharp
string logLine = "2024-01-15 14:30:45 [ERROR] Connection timeout";

var pattern = @"(?<date>\d{4}-\d{2}-\d{2})\s+(?<time>\d{2}:\d{2}:\d{2})\s+\[(?<level>\w+)\]\s+(?<msg>.+)";
Match m = Regex.Match(logLine, pattern);

if (m.Success)
{
    string date  = m.Groups["date"].Value;   // "2024-01-15"
    string time  = m.Groups["time"].Value;   // "14:30:45"
    string level = m.Groups["level"].Value;  // "ERROR"
    string msg   = m.Groups["msg"].Value;    // "Connection timeout"
}
```

### Generated Regex (C# 11 / .NET 7+) — compile-time

```csharp
public partial class LogParser
{
    // Компилируется в IL на этапе сборки — максимальная производительность
    [GeneratedRegex(
        @"(?<date>\d{4}-\d{2}-\d{2})\s+\[(?<level>\w+)\]\s+(?<msg>.+)",
        RegexOptions.ExplicitCapture)] // НЕ указывать RegexOptions.Compiled — в [GeneratedRegex] игнорируется
    private static partial Regex LogPattern();

    [GeneratedRegex(@"^[\w.+-]+@[\w-]+\.[\w.]+$", RegexOptions.IgnoreCase)]
    private static partial Regex EmailPattern();

    [GeneratedRegex(@"^\+7\d{10}$")]
    private static partial Regex PhonePattern();

    public static bool IsValidEmail(string email) => EmailPattern().IsMatch(email);
    public static bool IsValidPhone(string phone) => PhonePattern().IsMatch(phone);

    public static LogEntry? ParseLine(string line)
    {
        Match match = LogPattern().Match(line);
        if (!match.Success) return null;

        return new LogEntry(
            Date: DateOnly.Parse(match.Groups["date"].Value),
            Level: match.Groups["level"].Value,
            Message: match.Groups["msg"].Value);
    }
}

public record LogEntry(DateOnly Date, string Level, string Message);
```

### RegexOptions

```csharp
// Compiled — компиляция в IL при первом вызове (медленный старт, быстрое выполнение)
var compiled = new Regex(@"\d+", RegexOptions.Compiled);

// IgnoreCase — регистронезависимый поиск
Regex.IsMatch("Hello", @"hello", RegexOptions.IgnoreCase); // true

// Multiline — ^ и $ совпадают с началом/концом КАЖДОЙ строки
Regex.Matches("line1\nline2\nline3", @"^line\d", RegexOptions.Multiline); // 3 совпадения

// Singleline — точка (.) совпадает с \n
Regex.IsMatch("hello\nworld", @"hello.world", RegexOptions.Singleline); // true

// Комбинирование
var regex = new Regex(@"pattern",
    RegexOptions.Compiled | RegexOptions.IgnoreCase | RegexOptions.Multiline);

// Timeout — защита от ReDoS
var safe = new Regex(@"(a+)+$",
    RegexOptions.None,
    matchTimeout: TimeSpan.FromMilliseconds(100));

try
{
    safe.IsMatch("aaaaaaaaaaaaaaaaaaaaaaaaaaaa!");
}
catch (RegexMatchTimeoutException)
{
    Console.WriteLine("Regex timeout — возможная ReDoS-атака");
}
```

### Распространённые паттерны

```csharp
public static partial class CommonPatterns
{
    // Email (упрощённый, для базовой валидации)
    [GeneratedRegex(@"^[\w.+-]+@[\w-]+\.[\w.]+$", RegexOptions.IgnoreCase)]
    public static partial Regex Email();

    // Телефон РФ: +7XXXXXXXXXX
    [GeneratedRegex(@"^\+7\d{10}$")]
    public static partial Regex PhoneRu();

    // URL
    [GeneratedRegex(@"^https?://[\w\-.]+(:\d+)?(/[\w\-./?%&=]*)?$", RegexOptions.IgnoreCase)]
    public static partial Regex Url();

    // IPv4
    [GeneratedRegex(@"^((25[0-5]|2[0-4]\d|[01]?\d\d?)\.){3}(25[0-5]|2[0-4]\d|[01]?\d\d?)$")]
    public static partial Regex IPv4();

    // GUID
    [GeneratedRegex(@"^[0-9a-f]{8}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{12}$",
        RegexOptions.IgnoreCase)]
    public static partial Regex Guid();
}
```

---

## См. также

- [Типы и память](types-and-memory.md)
- [Async и потоки](async-threading.md)
- [Modern C#](modern-features.md)
- [Collections и LINQ](collections-linq.md)
