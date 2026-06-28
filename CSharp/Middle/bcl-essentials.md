---
tags: [bcl, guid, random, timespan, stopwatch, timeprovider, middle]
level: Middle
date: 2026-06-12
---

# BCL essentials — Guid, Random, TimeSpan, Stopwatch, TimeProvider

> Canonical-сборник по «мелким» типам BCL, которые используются в каждом проекте, но обычно известны фрагментарно: `Guid` (v4 vs v7, ключи БД и порядок байтов), `Random` (thread safety, `Random.Shared`, граница с криптографией), `TimeSpan`, `Stopwatch` (замеры без аллокаций) и `TimeProvider` (.NET 8) — тестируемое время.

---

## 1. Что это, зачем и когда

Каждый из этих типов выглядит тривиально — и у каждого есть классический способ выстрелить в проде:

| Тип | Типовой выстрел |
|---|---|
| `Guid` | v4 как clustered PK → page splits и фрагментация индекса |
| `Random` | Один инстанс из нескольких потоков → битое внутреннее состояние |
| `TimeSpan` | `Seconds` вместо `TotalSeconds` → потерянные минуты |
| `Stopwatch` | `DateTime.Now` для замеров → отрицательные интервалы после NTP-синка |
| Время вообще | `DateTime.UtcNow` намертво в коде → нетестируемая логика |

На собеседовании эти темы всплывают не напрямую, а через «как вы генерируете ключи», «как тестируете код с таймаутами», «почему бенчмарк врёт».

---

## 2. Guid

### 2.1. Что внутри — 128 бит и версии

`Guid` — 128-битное значение. В битах зашиты **версия** (алгоритм генерации) и **вариант** (layout). С .NET 9 их можно прочитать напрямую: свойства `Version` и `Variant`.

```csharp
Guid id = Guid.NewGuid();
Console.WriteLine(id.ToString("D")); // 8-4-4-4-12 с дефисами — формат по умолчанию
Console.WriteLine(id.ToString("N")); // 32 hex-символа без дефисов
Console.WriteLine(id.ToString("B")); // в фигурных скобках
```

| Версия | Источник | В .NET |
|---|---|---|
| v4 | Криптослучайные биты | `Guid.NewGuid()` — всегда был и есть |
| v7 | 48 бит Unix-времени (мс) + случайные биты | `Guid.CreateVersion7()` — .NET 9+, RFC 9562 |

### 2.2. Guid.NewGuid() — v4, криптослучайный

Источник — CSPRNG операционной системы: значения непредсказуемы, вероятность коллизии пренебрежима (122 случайных бита). Thread-safe, можно звать из любого потока без синхронизации.

Проблема не в уникальности, а в **полной случайности**: для упорядоченных структур (clustered index) каждый новый ключ попадает в случайное место.

### 2.3. Guid.CreateVersion7() — сортируемый по времени

```csharp
Guid orderId = Guid.CreateVersion7();
// первые 48 бит — Unix timestamp в миллисекундах,
// остальное — случайность

Guid backdated = Guid.CreateVersion7(DateTimeOffset.UtcNow.AddDays(-1)); // явный timestamp
```

Свойства v7:

- лексикографическая сортировка по байтам ≈ сортировка по времени создания;
- внутри одной миллисекунды порядок **не** гарантирован (случайная часть);
- timestamp читается из значения → ID слегка «течёт» информацией о времени создания. Для большинства систем это плюс (отладка), для anonymity-чувствительных — минус.

### 2.4. Guid как ключ БД — главный практический вопрос

Механизм проблемы v4: clustered index хранит строки физически в порядке ключа. Случайные ключи → вставки в случайные страницы → page splits, полупустые страницы, раздутый индекс, холодный кеш. На таблицах в десятки миллионов строк это видимая деградация вставок.

| СУБД | Как сортирует uuid | Вывод |
|---|---|---|
| PostgreSQL | memcmp по 16 байтам | **v7 идеален**: время в старших байтах → вставки в конец |
| MySQL (`BINARY(16)`) | memcmp | v7 идеален |
| SQL Server (`uniqueidentifier`) | **Свой порядок**: сравнение начинается с последних 6 байт | v7 для неё — почти random! Нужен sequential-в-хвосте |

> [!warning] SQL Server сортирует Guid не так, как все
> `uniqueidentifier` сравнивается по группам байтов **с конца**: сначала байты 10–15, потом 8–9 и так далее. Поэтому `NEWSEQUENTIALID()` монотонен именно в последних байтах, и EF Core для SQL Server генерирует client-side ключи через `SequentialGuidValueGenerator`, который учитывает этот порядок. Перенос «v7 решает всё» с PostgreSQL на SQL Server без понимания механизма — классическая Senior-ловушка.

Рабочие стратегии ключей:

- PostgreSQL/MySQL: `Guid.CreateVersion7()` как PK — норм.
- SQL Server: identity `int`/`long` как clustered PK + `Guid` как alternate key для внешнего мира; либо sequential-генерация под её порядок байтов.
- Распределённая генерация без координации — ровно то, ради чего Guid вообще существует: клиент создаёт ID до INSERT, нет round-trip за identity.

### 2.5. Парсинг

```csharp
if (Guid.TryParse(input, out Guid parsed))
{
    // TryParse на горячем пути; Parse + catch — антипаттерн flow control
}

bool isEmpty = parsed == Guid.Empty; // default(Guid) — все нули, валидный «нет значения»
```

> [!question]- Интервью: чем v7 лучше v4 для primary key?
> Оба уникальны; разница — в локальности вставок. v4 полностью случаен → в clustered index новые строки попадают в случайные страницы → page splits и фрагментация. v7 несёт Unix-время (мс) в старших битах → лексикографический порядок ≈ хронологический → вставки идут в конец индекса, как с identity, но без координации с БД. Оговорка: SQL Server сравнивает `uniqueidentifier` с последних байтов, поэтому там v7 не помогает — нужен sequential-в-хвосте (NEWSEQUENTIALID-семантика).

---

## 3. Random

### 3.1. Что это — PRNG, seed, детерминизм

`Random` — псевдослучайный генератор: из одного seed всегда одна последовательность. С .NET 6 **без seed** внутри работает быстрый xoshiro256**; конструктор **с seed** сохраняет старый алгоритм ради воспроизводимости между версиями.

```csharp
var reproducible = new Random(42); // тот же seed → та же последовательность (тесты, генерация миров)
int roll = reproducible.Next(1, 7); // 1..6 — верхняя граница исключается
```

### 3.2. Thread safety — главная ловушка

Инстанс `Random` **не thread-safe**. Параллельные вызовы `Next()` гонкой портят внутреннее состояние: в .NET Framework это знаменито вырождалось в бесконечные нули, в современных рантаймах — «просто» некорректное распределение. Симптом коварный: код не падает.

```csharp
// ❌ Один инстанс на всех — гонка по состоянию
public class BadDice
{
    private readonly Random _rng = new();
    public int Roll() => _rng.Next(1, 7); // зовётся из ThreadPool — состояние бьётся
}
```

### 3.3. Random.Shared (.NET 6+) — правильный дефолт

```csharp
int damage = Random.Shared.Next(10, 21);      // thread-safe, без создания инстансов
double chance = Random.Shared.NextDouble();   // [0.0, 1.0)
```

Один статический потокобезопасный генератор на процесс. Если не нужен seed — это ответ по умолчанию; `new Random()` в полях и циклах больше не нужен.

### 3.4. Новые API (.NET 8+)

```csharp
char[] code = Random.Shared.GetItems<char>("ABCDEFGHJKMNPQRSTUVWXYZ23456789", 6); // промокод

int[] deck = Enumerable.Range(1, 52).ToArray();
Random.Shared.Shuffle(deck); // Fisher–Yates in-place
```

### 3.5. Граница с криптографией

`Random` предсказуем по построению — для всего, что связано с безопасностью, он запрещён. Токены, пароли, ключи, коды подтверждения → `System.Security.Cryptography.RandomNumberGenerator`:

```csharp
byte[] key = RandomNumberGenerator.GetBytes(32);            // ключ/секрет
int otp = RandomNumberGenerator.GetInt32(100_000, 1_000_000); // 6-значный код без modulo bias
string token = RandomNumberGenerator.GetString(
    "abcdefghijklmnopqrstuvwxyz0123456789", 32);            // .NET 8+
```

Правило выбора одной строкой: **геймплей/сэмплинг/джиттер → `Random.Shared`; что-то, что атакующий хочет угадать → `RandomNumberGenerator`.**

> [!question]- Интервью: почему Random нельзя для генерации токенов сброса пароля?
> `Random` — детерминированный PRNG: зная алгоритм и наблюдая несколько выходов, состояние восстанавливается, дальнейшие значения предсказываются. Seed тоже не спасает — у него ограниченная энтропия. `RandomNumberGenerator` берёт энтропию из CSPRNG ОС и спроектирован против предсказания. Бонус: `GetInt32(min, max)` корректно избегает modulo bias, который легко внести, делая `rng-байты % диапазон` руками.

---

## 4. TimeSpan

### 4.1. Создание и арифметика

```csharp
TimeSpan timeout = TimeSpan.FromSeconds(30);
TimeSpan total = TimeSpan.FromHours(1) + TimeSpan.FromMinutes(15);
TimeSpan negative = TimeSpan.FromMinutes(-5); // отрицательные — валидны

TimeSpan parsed = TimeSpan.Parse("01:30:00"); // hh:mm:ss; culture-чувствителен — для контрактов лучше ISO 8601 или секунды числом
```

### 4.2. Seconds vs TotalSeconds — ловушка №1

```csharp
TimeSpan span = TimeSpan.FromSeconds(125);

int component = span.Seconds;        // 5  — компонент «секунды» (минуты ушли в Minutes)
double whole = span.TotalSeconds;    // 125 — вся длительность в секундах
```

`Days/Hours/Minutes/Seconds` — **разложение** на компоненты; `Total*` — вся величина в выбранной единице. Перепутать — значит молча потерять минуты и часы. Та же пара существует для всех единиц.

### 4.3. TimeSpan ≠ инструмент измерения

`DateTime.Now - start` для замера длительности — баг: системные часы не монотонны (NTP-коррекция, ручной перевод, DST для `Now`) → интервал может выйти отрицательным или прыгнуть. Замеры — только `Stopwatch` (ниже). `TimeSpan` — это **представление** длительности, а не способ её получить.

---

## 5. Stopwatch

### 5.1. База

```csharp
var sw = Stopwatch.StartNew();
DoWork();
sw.Stop();
Console.WriteLine($"Took {sw.ElapsedMilliseconds} ms");
```

Под капотом — монотонный высокочастотный счётчик ОС (на Windows — QueryPerformanceCounter): не зависит от системных часов, не прыгает, тикает только вперёд.

### 5.2. ElapsedTicks ≠ TimeSpan.Ticks

```csharp
var sw2 = Stopwatch.StartNew();
DoWork();
sw2.Stop();

TimeSpan ok = sw2.Elapsed;                    // ✅ корректная конвертация
TimeSpan bug = new TimeSpan(sw2.ElapsedTicks); // ❌ тики Stopwatch — в единицах Frequency, не 100 ns!
```

Тик `TimeSpan` — всегда 100 наносекунд. Тик `Stopwatch` — `1 / Stopwatch.Frequency` секунды, и `Frequency` зависит от платформы. Совпадают они только случайно. Использовать `Elapsed`/`ElapsedMilliseconds` — они конвертируют правильно.

### 5.3. Замер без аллокаций — GetTimestamp / GetElapsedTime (.NET 7+)

`Stopwatch` — класс: `StartNew()` на каждый запрос в hot path — это аллокация и давление на GC. Статическая пара решает:

```csharp
long start = Stopwatch.GetTimestamp();
ProcessRequest();
TimeSpan elapsed = Stopwatch.GetElapsedTime(start); // только long на стеке
```

Именно так меряют middleware, логирование длительности запросов и метрики.

### 5.4. Ручной бенчмарк — почему врёт

Голый `Stopwatch` вокруг метода игнорирует: JIT-компиляцию первого вызова, tiered compilation, прогрев кешей, GC посреди замера, dead-code elimination. Для «насколько A быстрее B» — BenchmarkDotNet ([[performance|Performance]]); `Stopwatch` — для производственной телеметрии, не для сравнений.

---

## 6. TimeProvider (.NET 8+) — тестируемое время

### 6.1. Проблема

```csharp
// ❌ Нетестируемо: «через 24 часа скидка сгорает» проверяется только ожиданием 24 часов
public bool IsExpired(DateTime createdUtc) =>
    DateTime.UtcNow - createdUtc > TimeSpan.FromHours(24);
```

`DateTime.UtcNow` — статика, в тесте не подменишь. До .NET 8 каждый писал свой `IClock`; теперь абстракция встроена в BCL.

### 6.2. API

```csharp
public sealed class DiscountService(TimeProvider time)
{
    public bool IsExpired(DateTimeOffset createdUtc) =>
        time.GetUtcNow() - createdUtc > TimeSpan.FromHours(24);

    public TimeSpan Measure(Action work)
    {
        long start = time.GetTimestamp();         // монотонный, как Stopwatch
        work();
        return time.GetElapsedTime(start);
    }
}
```

```csharp
// DI: один системный провайдер на приложение
builder.Services.AddSingleton(TimeProvider.System);
```

`TimeProvider` объединяет оба вида времени: wall-clock (`GetUtcNow`, `GetLocalNow`, `LocalTimeZone`) и монотонное (`GetTimestamp`/`GetElapsedTime`), плюс фабрику таймеров `CreateTimer`. Интеграция в BCL: перегрузки `Task.Delay(delay, timeProvider)`, конструкторы `PeriodicTimer` и `CancellationTokenSource` принимают провайдер.

### 6.3. FakeTimeProvider — тест без ожидания

Пакет `Microsoft.Extensions.TimeProvider.Testing`:

```csharp
[Fact]
public void Discount_Expires_After24Hours()
{
    var fake = new FakeTimeProvider(DateTimeOffset.Parse("2026-06-12T10:00:00Z"));
    var sut = new DiscountService(fake);
    var created = fake.GetUtcNow();

    fake.Advance(TimeSpan.FromHours(23));
    sut.IsExpired(created).Should().BeFalse();

    fake.Advance(TimeSpan.FromHours(2));
    sut.IsExpired(created).Should().BeTrue();
}
```

`Advance` мгновенно двигает время и срабатывает таймеры/`Task.Delay`, ждавшие через этот провайдер. Тест таймаутов, ретраев, истечения кеша — за миллисекунды реального времени. Связка с [[integration-testing|Integration Testing]] и [[async-threading|Async и Threading]] (`PeriodicTimer` в background-сервисах).

> [!question]- Интервью: как протестировать «повторить запрос через 5 минут после неудачи»?
> Внедрить `TimeProvider` вместо обращений к `DateTime.UtcNow`/`Task.Delay(...)` напрямую: продакшен получает `TimeProvider.System` из DI, тест — `FakeTimeProvider`. В тесте: спровоцировать неудачу, `fake.Advance(TimeSpan.FromMinutes(5))` — и таймер/`Task.Delay`, ждущие на этом провайдере, срабатывают синхронно. Реального ожидания нет, флаки нет. До .NET 8 то же делали самописным `IClock`; теперь это стандартный тип BCL с интеграцией в `Task.Delay`, `PeriodicTimer`, `CancellationTokenSource`.

---

## 7. Common Pitfalls — с механизмами

1. **v4 Guid как clustered PK** → случайные вставки, page splits, фрагментация (раздел 2.4). Фикс: v7 (PG/MySQL) или identity + alternate key (SQL Server).
2. **«v7 решает всё» на SQL Server** → она сравнивает `uniqueidentifier` с последних байтов; v7 там не sequential (warning в 2.4).
3. **Один `Random` из нескольких потоков** → гонка по внутреннему состоянию, испорченное распределение без исключений (3.2). Фикс: `Random.Shared`.
4. **`Random` для токенов/кодов** → предсказуем; `RandomNumberGenerator` (3.5).
5. **`span.Seconds` вместо `span.TotalSeconds`** → потерянные минуты и часы: компонент против всей величины (4.2).
6. **`DateTime.Now - start` для замеров** → системные часы не монотонны; `Stopwatch` (4.3).
7. **`new TimeSpan(sw.ElapsedTicks)`** → тики Stopwatch в единицах `Frequency`, не 100 ns (5.2).
8. **`Stopwatch.StartNew()` в hot path** → аллокация на каждый замер; `GetTimestamp`/`GetElapsedTime` (5.3).
9. **`DateTime.UtcNow` зашит в логику** → нетестируемые таймауты/истечения; `TimeProvider` через DI (6).
10. **`Guid.Parse` + `catch` как валидация ввода** → исключения как flow control на горячем пути; `TryParse` (2.5).

---

## 8. Decision tree

```text
Нужен идентификатор?
├─ Ключ БД, PostgreSQL/MySQL → Guid.CreateVersion7()
├─ Ключ БД, SQL Server → identity + Guid alternate key (или sequential-в-хвосте)
└─ Просто уникальный ID (корреляция, имена файлов) → Guid.NewGuid()

Нужна случайность?
├─ Безопасность (токен, код, ключ) → RandomNumberGenerator
├─ Всё остальное, без seed → Random.Shared
└─ Воспроизводимость (тест, генерация) → new Random(seed), один поток

Нужно время?
├─ Измерить длительность → Stopwatch.GetTimestamp / GetElapsedTime
├─ Wall-clock в бизнес-логике → TimeProvider.GetUtcNow() через DI
├─ Хранить/передавать момент → DateTimeOffset (см. datetime-timezones)
└─ Хранить/передавать длительность → TimeSpan
```

---

## 9. Cheat sheet

| Задача | Код |
|---|---|
| Сортируемый ID (.NET 9) | `Guid.CreateVersion7()` |
| Случайное число, любой поток | `Random.Shared.Next(min, maxExcl)` |
| Перемешать | `Random.Shared.Shuffle(array)` |
| Крипто-токен | `RandomNumberGenerator.GetString(alphabet, len)` |
| Вся длительность в секундах | `span.TotalSeconds` |
| Замер без аллокаций | `Stopwatch.GetTimestamp()` → `GetElapsedTime(start)` |
| Время в логике | `TimeProvider.GetUtcNow()` (DI) |
| Время в тесте | `FakeTimeProvider` + `Advance(...)` |

---

## 10. См. также

- [[datetime-timezones|DateTime и Timezones]] — wall-clock время глубоко: Kind, DST, NodaTime
- [[serialization-deep|Serialization deep]] — Guid/TimeSpan на проводе (JSON-контракты)
- [[queries-performance|EF Queries Performance]] — что фрагментация индекса делает с запросами
- [[performance|Performance]] — BenchmarkDotNet вместо ручных замеров
- [[async-threading|Async и Threading]] — PeriodicTimer, Task.Delay, отмена
- [[integration-testing|Integration Testing]] — FakeTimeProvider в тестовой инфраструктуре
- [[numeric-types-math|Numeric Types]] — соседний BCL-reference

## 11. Reading list

- RFC 9562 — UUID v7 спецификация
- Microsoft Learn: «Guid.CreateVersion7 Method», «Random.Shared Property»
- Microsoft Learn: «What's new in .NET 8 — TimeProvider» + «Unit test time-dependent code»
- .NET Blog: «.NET 6 — Random improvements» (xoshiro, Shared)
- Raymond Chen / SQL Server docs: порядок сравнения uniqueidentifier
