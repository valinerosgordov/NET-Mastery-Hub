---
tags: [serialization, json, system-text-json, xml, messagepack, middle]
level: Middle
date: 2026-06-12
---

# Serialization deep — System.Text.Json, XML, binary

> Canonical-файл по сериализации в .NET: System.Text.Json как дефолт 2026 (options, конвертеры, полиморфизм, source generation, стриминг), миграция с Newtonsoft, что осталось от XML, почему BinaryFormatter мёртв и чем его заменили (MessagePack / protobuf / MemoryPack).

---

## 1. Что это, зачем и когда

### 1.1. Ландшафт 2026

| Технология | Статус | Когда |
|---|---|---|
| **System.Text.Json (STJ)** | Дефолт платформы | API, конфиги, кеши, всё новое |
| Newtonsoft.Json | Legacy, поддерживается | Старые кодобазы; фичи, которых нет в STJ |
| `XmlSerializer` / `DataContractSerializer` | Legacy | SOAP, старые интеграции, конфиги .NET Framework |
| `BinaryFormatter` | **Удалён** | Никогда. Из коробки выпилен в .NET 9 |
| MessagePack / protobuf / MemoryPack | Активны | Бинарная производительность: кеши, gRPC, межсервисные контракты |

### 1.2. Зачем знать глубже, чем `JsonSerializer.Serialize`

Сериализация — это граница системы: API-контракты, кеш, очереди, файлы. Ошибки здесь — это либо производительность (рекреация options, буферизация гигабайтов), либо безопасность (полиморфная десериализация — классический RCE-вектор), либо тихая порча данных (camelCase vs PascalCase, потерянные поля). На собеседовании тема всплывает через «как устроен ваш API-контракт» и «почему BinaryFormatter запрещён».

---

## 2. System.Text.Json — база

```csharp
public sealed record Order(int Id, string Customer, decimal Total);
```

```csharp
var order = new Order(42, "Vitali", 199.90m);

string json = JsonSerializer.Serialize(order);
Order? back = JsonSerializer.Deserialize<Order>(json);
```

Ключевые свойства STJ по сравнению с Newtonsoft:

- работает поверх `Utf8JsonReader` / `Utf8JsonWriter` — UTF-8 байты напрямую, без промежуточной UTF-16 строки;
- **strict по умолчанию**: case-sensitive имена, без комментариев, без trailing commas, даты — строго ISO 8601;
- меньше магии: приватные сеттеры, поля, циклы — всё opt-in.

### 2.1. Два пресета options

```csharp
// Дефолт: PascalCase, case-sensitive — «как в C#»
var strict = JsonSerializerOptions.Default;

// Web-пресет: camelCase, case-insensitive чтение, числа из строк —
// именно его использует ASP.NET Core для тел запросов/ответов
var web = new JsonSerializerOptions(JsonSerializerDefaults.Web);
```

Поэтому один и тот же тип в ручном `Serialize` и в ответе контроллера сериализуется **по-разному**, если не синхронизировать options.

---

## 3. JsonSerializerOptions — главный performance pitfall

При первом использовании options для типа STJ строит и кеширует метаданные (рефлексия, делегаты доступа). Кеш живёт **внутри экземпляра options**.

```csharp
// ❌ Антипаттерн: новый options на каждый вызов = пересборка метаданных каждый раз.
// На горячем пути это на порядок медленнее и аллоцирует как не в себя.
public static string SerializeBad<T>(T value) =>
    JsonSerializer.Serialize(value, new JsonSerializerOptions { WriteIndented = true });
```

```csharp
// ✅ Один статический экземпляр на конфигурацию
public static class Json
{
    public static readonly JsonSerializerOptions Pretty = new(JsonSerializerDefaults.Web)
    {
        WriteIndented = true
    };
}

public static class OrderJson
{
    public static string Serialize(object value) => JsonSerializer.Serialize(value, Json.Pretty);
}
```

С .NET 8 options можно «заморозить» явно (`MakeReadOnly()`), а ASP.NET Core свои options и так шарит. Правило простое: **options создаются один раз и переиспользуются**.

> [!question]- Интервью: почему new JsonSerializerOptions() в цикле — это плохо?
> Метаданные типа (резолв свойств, конвертеров, делегаты get/set) кешируются per-options-instance. Новый экземпляр = холодный кеш = повторная рефлексия на каждый вызов. Бенчмарки Microsoft показывают разницу в десятки раз на маленьких объектах. Фикс: static readonly options или `GetOptions` из DI.

---

## 4. Атрибуты и тюнинг контракта

```csharp
public sealed class UserDto
{
    [JsonPropertyName("user_id")]
    public required int Id { get; init; }

    [JsonIgnore(Condition = JsonIgnoreCondition.WhenWritingNull)]
    public string? MiddleName { get; init; }

    [JsonInclude]                       // приватный сеттер виден только с JsonInclude
    public string Status { get; private set; } = "active";

    [JsonNumberHandling(JsonNumberHandling.AllowReadingFromString)]
    public decimal Balance { get; init; }   // примет и 12.5, и "12.5"
}
```

Глобально через options:

```csharp
var apiOptions = new JsonSerializerOptions
{
    PropertyNamingPolicy = JsonNamingPolicy.SnakeCaseLower,  // .NET 8+: snake/kebab из коробки
    DefaultIgnoreCondition = JsonIgnoreCondition.WhenWritingNull,
    NumberHandling = JsonNumberHandling.AllowReadingFromString,
    Converters = { new JsonStringEnumConverter() }           // enum как строки, не числа
};
```

- Naming policies в .NET 8+: `CamelCase`, `SnakeCaseLower/Upper`, `KebabCaseLower/Upper`.
- Enum по умолчанию пишется числом — контракт ломается при перестановке членов. `JsonStringEnumConverter` (для AOT — generic-версия `JsonStringEnumConverter<TEnum>`) почти всегда правильнее. Подробнее про enum — [[enums-flags|Enums и Flags]].
- `required` (C# 11) STJ понимает: отсутствие обязательного свойства в JSON → `JsonException`, а не молчаливый default.

### 4.1. Конструкторы

STJ умеет десериализовать через параметризованный конструктор — параметры матчатся по именам свойств (case-insensitive). Для records это работает из коробки. Если конструкторов несколько — пометь нужный `[JsonConstructor]`.

---

## 5. Custom converters

Когда формат поля не совпадает с типом: Unix-время, деньги строкой, доменные ID.

```csharp
public sealed class UnixDateTimeConverter : JsonConverter<DateTimeOffset>
{
    public override DateTimeOffset Read(
        ref Utf8JsonReader reader, Type typeToConvert, JsonSerializerOptions options) =>
        DateTimeOffset.FromUnixTimeSeconds(reader.GetInt64());

    public override void Write(
        Utf8JsonWriter writer, DateTimeOffset value, JsonSerializerOptions options) =>
        writer.WriteNumberValue(value.ToUnixTimeSeconds());
}
```

```csharp
var unixOptions = new JsonSerializerOptions
{
    Converters = { new UnixDateTimeConverter() }
};
```

Для generic-семейств типов (`Result<T>`, `Maybe<T>`) — `JsonConverterFactory`: фабрика по `Type` создаёт закрытый конвертер. Это тот же механизм, которым внутри STJ сделаны конвертеры коллекций.

Правила хорошего конвертера: не аллоцировать лишнего в `Read`/`Write` (это горячий путь), уважать `options` (naming policy и вложенная сериализация — через `JsonSerializer.Serialize(writer, value, options)`), бросать `JsonException` с понятным сообщением на мусорном входе.

---

## 6. Полиморфизм — и почему whitelist, а не TypeNameHandling

С .NET 7 полиморфная (де)сериализация — встроенная и **безопасная по построению**: только явно перечисленные наследники.

```csharp
[JsonPolymorphic(TypeDiscriminatorPropertyName = "$type")]
[JsonDerivedType(typeof(CardPayment), "card")]
[JsonDerivedType(typeof(SepaPayment), "sepa")]
public abstract record Payment(decimal Amount);

public sealed record CardPayment(decimal Amount, string Last4) : Payment(Amount);
public sealed record SepaPayment(decimal Amount, string Iban) : Payment(Amount);
```

```json
{ "$type": "card", "amount": 100, "last4": "4242" }
```

Десериализация в `Payment` вернёт правильный конкретный тип. Неизвестный дискриминатор → `JsonException` (поведение настраивается через `JsonUnknownDerivedTypeHandling`).

> [!warning] Почему у Newtonsoft TypeNameHandling.All — это RCE
> Там дискриминатор `$type` содержит **полное имя CLR-типа**, и десериализатор инстанцирует то, что пришло с проводов. Атакующий присылает имя гаджет-типа из BCL/популярных библиотек, чей конструктор/сеттер исполняет код (классика — `ObjectDataProvider`). Это не теоретическая дыра, а отработанный класс эксплойтов (ysoserial.net). STJ-подход — closed set наследников, имя типа с клиента не принимается в принципе. На собеседовании это canonical-ответ про «insecure deserialization».

---

## 7. Source generation — JsonSerializerContext

Reflection-based сериализация несовместима с trimming/Native AOT (типы срезаются, рефлексия слепнет) и платит за warm-up. Source generator переносит построение метаданных в compile-time:

```csharp
[JsonSourceGenerationOptions(
    PropertyNamingPolicy = JsonKnownNamingPolicy.CamelCase,
    GenerationMode = JsonSourceGenerationMode.Default)]   // Metadata + fast-path Serialization
[JsonSerializable(typeof(Order))]
[JsonSerializable(typeof(List<Order>))]
public sealed partial class AppJsonContext : JsonSerializerContext;
```

```csharp
// Вариант 1: явно через контекст
string json = JsonSerializer.Serialize(new Order(1, "A", 10m), AppJsonContext.Default.Order);

// Вариант 2: через options — удобно для ASP.NET Core
var sgOptions = new JsonSerializerOptions
{
    TypeInfoResolver = AppJsonContext.Default
};
```

```csharp
// Minimal API / AOT: регистрация контекста глобально
builder.Services.ConfigureHttpJsonOptions(o =>
    o.SerializerOptions.TypeInfoResolverChain.Insert(0, AppJsonContext.Default));
```

Режимы:

- **Metadata** — сгенерированные метаданные вместо рефлексии: AOT-safe, быстрый старт.
- **Serialization (fast-path)** — прямой сгенерированный код записи без слоя метаданных: максимальная скорость, но fast-path отключается, если в игре custom converters, `ReferenceHandler` или async-стриминг — STJ тихо падает обратно на metadata-режим.

Для Native AOT контекст **обязателен** — см. [[native-aot|Native AOT]]. Как вообще работают генераторы — [[source-generators|Source Generators]].

---

## 8. Стриминг и большие данные

### 8.1. Async по Stream

```csharp
await using var file = File.OpenRead("orders.json");
List<Order>? all = await JsonSerializer.DeserializeAsync<List<Order>>(file);
```

### 8.2. Элементы по одному — DeserializeAsyncEnumerable

Корневой JSON-массив можно читать, не материализуя целиком:

```csharp
await using var stream = File.OpenRead("orders.json"); // [ {...}, {...}, ... ]

await foreach (Order order in JsonSerializer.DeserializeAsyncEnumerable<Order>(stream))
{
    await ProcessAsync(order);
}
```

Память — O(один элемент), а не O(файл). Та же механика отдаёт `IAsyncEnumerable<T>` из ASP.NET Core endpoint'а — клиент начинает получать элементы до конца генерации. Про `IAsyncEnumerable` — [[iterators-yield|Iterators и yield]].

### 8.3. Utf8JsonReader — ручной разбор

Низкоуровневый ref struct reader для случаев «вытащить два поля из гигабайтного JSON»:

```csharp
public static long SumTotals(ReadOnlySpan<byte> utf8Json)
{
    var reader = new Utf8JsonReader(utf8Json);
    long sum = 0;
    while (reader.Read())
    {
        if (reader.TokenType == JsonTokenType.PropertyName && reader.ValueTextEquals("total"))
        {
            reader.Read();
            sum += reader.GetInt64();
        }
    }
    return sum;
}
```

Zero-allocation, forward-only. Это фундамент, на котором стоит весь STJ; руками нужен редко, но на собеседовании знание его существования отделяет «пользователя» от «понимающего».

---

## 9. DOM: JsonDocument и JsonNode

| | `JsonDocument` | `JsonNode` |
|---|---|---|
| Мутабельность | Read-only | Mutable |
| Память | Пулинг буферов, очень дёшево | Обычные объекты на куче |
| Dispose | **Обязателен** (вернуть буферы в pool) | Не нужен |
| Когда | Прочитать/проверить пару полей | Собрать/модифицировать JSON без типа |

```csharp
using var doc = JsonDocument.Parse(payload);          // не забыть using!
int id = doc.RootElement.GetProperty("id").GetInt32();
```

```csharp
var node = JsonNode.Parse("""{ "user": { "name": "V" } }""")!;
node["user"]!["name"] = "Vitali";
node["user"]!["active"] = true;
string mutated = node.ToJsonString();
```

Забытый `using` у `JsonDocument` — не утечка в классическом смысле, но буферы не возвращаются в `ArrayPool`, и под нагрузкой это лишние аллокации и GC pressure — [[memory-pooling|Memory Pooling]].

---

## 10. STJ vs Newtonsoft — миграционная шпаргалка

| Поведение | Newtonsoft | STJ |
|---|---|---|
| Регистр имён при чтении | Insensitive | Sensitive (Web-пресет — insensitive) |
| Комментарии / trailing commas | Терпит | `JsonException` (включается через options) |
| Приватные сеттеры | Пишет молча | Только с `[JsonInclude]` |
| Поля (fields) | Сериализует | Только с `IncludeFields = true` |
| Циклические ссылки | `ReferenceLoopHandling.Ignore` | `ReferenceHandler.IgnoreCycles` (.NET 6+) или `Preserve` (`$id`/`$ref`) |
| `DateTime` | Гибкий парсер | Строго ISO 8601 (иначе — custom converter) |
| Полиморфизм | `TypeNameHandling` (опасно) | `[JsonPolymorphic]` whitelist (безопасно) |
| Словарь с не-string ключом | Да | Да (.NET 5+, ключи — примитивы/enum/Guid) |
| `JObject`-стиль | `JObject` / `JToken` | `JsonNode` / `JsonElement` |

Стратегия миграции: новые контракты — STJ; legacy-эндпоинты с завязкой на тонкости Newtonsoft можно временно оставить через `AddNewtonsoftJson()`, но это глобальный свитч сериализатора MVC — осознанное решение, не дефолт.

---

## 11. XML — что осталось в 2026

`XmlSerializer` жив в интеграциях: SOAP, банковские/госформаты, старые конфиги.

```csharp
[XmlRoot("order")]
public sealed class OrderXml
{
    [XmlAttribute("id")]
    public int Id { get; set; }

    [XmlElement("customer")]
    public string Customer { get; set; } = "";

    [XmlArray("items"), XmlArrayItem("item")]
    public List<string> Items { get; set; } = [];
}
```

```csharp
public static class OrderXmlIo
{
    // Кеш обязателен — см. warning ниже
    private static readonly XmlSerializer Serializer = new(typeof(OrderXml));

    public static string Write(OrderXml order)
    {
        using var sw = new StringWriter();
        Serializer.Serialize(sw, order);
        return sw.ToString();
    }
}
```

> [!warning] Утечка памяти XmlSerializer
> Конструкторы `new XmlSerializer(Type)` и `(Type, defaultNamespace)` кешируют сгенерированную при первом вызове temp-сборку. **Все остальные перегрузки** (с `XmlRootAttribute`, overrides и т.п.) генерируют новую сборку на каждый вызов — сборки не выгружаются, память течёт необратимо. Правило: экземпляр `XmlSerializer` с «нестандартным» конструктором создаётся один раз и кешируется статически.

Требования `XmlSerializer`: публичный тип, публичный конструктор без параметров, сериализуются только публичные read/write свойства. `DataContractSerializer` (наследие WCF) — opt-in через `[DataContract]`/`[DataMember]`, умеет приватное, но для нового кода поводов нет.

---

## 12. Binary — смерть BinaryFormatter и наследники

### 12.1. Почему BinaryFormatter мёртв

`BinaryFormatter` десериализует **произвольный граф типов, описанный в payload'е** — то же фундаментальное свойство, что у `TypeNameHandling.All`, только хуже: формат бинарный, типы инстанцируются до всякой валидации. Любой источник данных, до которого дотянулся атакующий (файл, очередь, кука), = потенциальный RCE. Microsoft объявила его unfixable-by-design; с .NET 9 реализация **удалена из коробки** (методы бросают `PlatformNotSupportedException`), остался только отдельный compat-пакет для миграционного периода — использовать его в новом коде нельзя.

### 12.2. Чем заменяют

| Библиотека | Формат | Сильные стороны | Когда |
|---|---|---|---|
| **MessagePack-CSharp** | MessagePack | Очень быстрый, компактный, LZ4 из коробки | Кеши (Redis), внутренние очереди, SignalR-протокол |
| **protobuf** (Google.Protobuf / protobuf-net) | Protocol Buffers | Контракт `.proto`, эволюция схемы, кроссплатформенность | Межсервисные контракты, gRPC — [[ipc-named-pipes-grpc|gRPC и IPC]] |
| **MemoryPack** | Свой zero-encoding | Быстрее всех, source-gen, C#-only | Внутренний perf-critical обмен между .NET-узлами |

```csharp
[MessagePackObject]
public sealed class CacheEntry
{
    [Key(0)] public int Id { get; set; }
    [Key(1)] public string Payload { get; set; } = "";
}
```

```csharp
byte[] packed = MessagePackSerializer.Serialize(new CacheEntry { Id = 1, Payload = "x" });
CacheEntry restored = MessagePackSerializer.Deserialize<CacheEntry>(packed);
```

Выбор по границе: **наружу и между командами** — protobuf (контракт первичен), **внутри своей системы** — MessagePack/MemoryPack (скорость первична), **человекочитаемое** — JSON.

---

## 13. Common Pitfalls — с механизмами

1. **`new JsonSerializerOptions()` на вызов** → пересборка кеша метаданных, на порядок медленнее (раздел 3).
2. **PascalCase/camelCase рассинхрон.** Ручной `Serialize` использует дефолтные options, ASP.NET Core — Web-пресет. Поле «теряется» при чтении, потому что регистр не совпал и чтение case-sensitive. Фикс — единые options через DI.
3. **Enum числом в контракте.** Перестановка членов enum меняет значения на проводе. `JsonStringEnumConverter` + явные значения членов.
4. **Полиморфизм через объект с полем-типом руками** вместо `[JsonPolymorphic]` → или небезопасно (если резолвишь `Type.GetType` по строке клиента), или ломко. Встроенный whitelist-механизм закрывает оба риска (раздел 6).
5. **`JsonDocument` без `using`** → буферы не возвращаются в pool, GC pressure (раздел 9).
6. **Гигабайтный JSON через `Deserialize<List<T>>`** → весь файл в памяти + LOH. `DeserializeAsyncEnumerable` (раздел 8.2).
7. **Reflection-сериализация в Native AOT** → `NotSupportedException`/пустые объекты после trimming. Source generation обязателен (раздел 7).
8. **`XmlSerializer` с overrides в цикле** → утечка temp-сборок (раздел 11).
9. **`BinaryFormatter` для «быстрого» сохранения состояния** → RCE-вектор + код не соберётся на .NET 9+. MessagePack решает ту же задачу безопасно (раздел 12).
10. **`DateTime` без таймзонной дисциплины.** STJ пишет `DateTimeKind.Unspecified` без смещения — клиент интерпретирует как хочет. Использовать `DateTimeOffset` на границах — [[datetime-timezones|DateTime и Timezones]].

---

## 14. Decision tree

```text
Что сериализуем?
├─ HTTP API / конфиг / человекочитаемое → System.Text.Json
│   ├─ Native AOT или критичен cold start → + JsonSerializerContext (source-gen)
│   ├─ Иерархия наследников в контракте → [JsonPolymorphic] + [JsonDerivedType]
│   ├─ Гигантский массив → DeserializeAsyncEnumerable / Utf8JsonReader
│   └─ Нужны фичи Newtonsoft (legacy-тонкости) → точечно AddNewtonsoftJson, план миграции
├─ Межсервисный бинарный контракт / gRPC → protobuf
├─ Кеш / внутренняя очередь, скорость важна → MessagePack (или MemoryPack, если C#-only)
├─ SOAP / госформат / legacy → XmlSerializer (с кешированием экземпляра)
└─ «Просто сохранить объект в файл» → STJ. Не BinaryFormatter. Никогда.
```

---

## 15. Cheat sheet

```csharp
// Канонический setup для API-проекта
public static class Json
{
    public static readonly JsonSerializerOptions Default = new(JsonSerializerDefaults.Web)
    {
        DefaultIgnoreCondition = JsonIgnoreCondition.WhenWritingNull,
        Converters = { new JsonStringEnumConverter() }
    };
}
```

| Задача | Инструмент |
|---|---|
| Options | static readonly, один на конфигурацию |
| camelCase / snake_case | `PropertyNamingPolicy` (.NET 8: snake/kebab встроены) |
| Enum | `JsonStringEnumConverter` (`<TEnum>` для AOT) |
| Обязательное поле | `required` — STJ бросит, если нет в JSON |
| Наследование | `[JsonPolymorphic]` + `[JsonDerivedType]` |
| AOT / trimming | `JsonSerializerContext` |
| Поток элементов | `DeserializeAsyncEnumerable<T>` |
| Читать пару полей | `JsonDocument` (+ `using`) |
| Мутировать JSON | `JsonNode` |
| Бинарный кеш | MessagePack |
| Кросс-язычный контракт | protobuf |

---

## 16. См. также

- [[io-streams|IO и Streams]] — Stream-инфраструктура, поверх которой всё это работает
- [[attributes-metadata|Attributes и Metadata]] — как читаются атрибуты контракта
- [[source-generators|Source Generators]] — механика compile-time генерации
- [[native-aot|Native AOT]] — почему source-gen там обязателен
- [[datetime-timezones|DateTime и Timezones]] — даты на границах систем
- [[api-design|API design]] — версионирование контрактов
- [[ipc-named-pipes-grpc|gRPC и IPC]] — protobuf в деле
- [[memory-pooling|Memory Pooling]] — ArrayPool, который использует JsonDocument

## 17. Reading list

- Microsoft Learn: «JSON serialization and deserialization in .NET» — официальный hub по STJ
- Microsoft Learn: «How to use source generation in System.Text.Json»
- .NET Blog: «Try the new System.Text.Json source generator»
- Microsoft: «BinaryFormatter security guide» + «BinaryFormatter removal» (migration guide)
- neuecc: README MessagePack-CSharp — про дизайн и бенчмарки бинарных форматов
