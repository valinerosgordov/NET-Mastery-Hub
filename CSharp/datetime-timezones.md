---
tags: [csharp, datetime, timezone, dateonly, timeonly, nodatime]
level: Junior
date: 2026-04-30
---

# DateTime, Timezones и время

> **Полный гайд по работе со временем в C#**. Самая baga-prone тема в любом проекте. Закрывает: DateTime vs DateTimeOffset, DateOnly/TimeOnly (.NET 6+), TimeProvider (.NET 8+), timezone обработка, DST, NodaTime для serious приложений, типичные ошибки и как их избегать.

---

## Что это, зачем и когда

### Почему время — сложно

```
Звучит просто:    "сохранить когда event произошёл"
В реальности:
  - В какой timezone?
  - Server timezone? User timezone? UTC?
  - Daylight Saving Time переход?
  - User путешествует — timezone изменилась?
  - DB timezone?
  - JSON serialization formats?
  - Display для разных culture?
  - Calendar (Gregorian / Hebrew / Japanese era)?
```

Bagи со временем — **самые дорогие**:
- "Order showed yesterday хотя placed today" — клиент злой
- "Notification at 3am instead of 3pm" — pager wakes engineer
- "DST переход — duplicated calendar events"
- "1 hour offset на каждом event 2 раза в год"

### Главное правило

> **Сохраняй UTC. Конвертируй только для display.**

```
DB:     UTC всегда
API:    UTC в JSON
Logs:   UTC
Memory: UTC
Display: → user's timezone
```

---

## 1. Типы для времени в .NET

### DateTime

Точка времени **без TZ информации**. Имеет `Kind`: Local / Utc / Unspecified.

```csharp
DateTime now = DateTime.Now;             // Local
DateTime utc = DateTime.UtcNow;          // UTC
DateTime parsed = DateTime.Parse("2024-01-15 12:00");  // Unspecified!

now.Kind;     // Local
utc.Kind;     // Utc
parsed.Kind;  // Unspecified
```

**Pитфolls:**
- `DateTime.Now` — local time текущей машины (server, не user!)
- `Kind = Unspecified` после parse — "?". Может ломать логику
- Сравнение Local и UTC — никаких автоматических conversions

### DateTimeOffset ⭐ (preferred)

DateTime + offset от UTC. **Явное указание TZ** (как offset).

```csharp
DateTimeOffset now = DateTimeOffset.Now;       // 2024-01-15T12:00:00+05:00
DateTimeOffset utc = DateTimeOffset.UtcNow;    // 2024-01-15T07:00:00+00:00

now.Offset;      // 05:00:00
now.UtcDateTime; // UTC version
now.LocalDateTime; // Local version
```

**Преимущество:** offset включён → нет ambiguity.

**Недостаток:** хранит offset, не **timezone identity** (важно для DST!).

### TimeSpan

Длительность (не точка во времени).

```csharp
TimeSpan duration = TimeSpan.FromMinutes(30);
TimeSpan diff = end - start;
duration.TotalSeconds;
duration.Days;
```

### DateOnly и TimeOnly (.NET 6+)

Только дата или только время — без time / date components.

```csharp
DateOnly date = new(2024, 1, 15);
DateOnly today = DateOnly.FromDateTime(DateTime.Today);

TimeOnly time = new(12, 30, 0);  // 12:30:00
TimeOnly noon = TimeOnly.Parse("12:00");

// Combine
DateTime full = date.ToDateTime(time);
```

**Зачем:**
- Birthday — только date, не время
- Office hours — только time
- Schedule в day — DateOnly + TimeOnly

### TimeProvider (.NET 8+) ⭐

Абстракция для time — для testability.

```csharp
public class OrderService(TimeProvider time)
{
    public bool IsExpired(Order order) =>
        time.GetUtcNow() > order.ExpiresAt;
}

// Production
services.AddSingleton(TimeProvider.System);

// Testing
var fake = new FakeTimeProvider();
fake.SetUtcNow(DateTimeOffset.Parse("2024-06-01"));
var service = new OrderService(fake);

// Можно "ускорить время"
fake.Advance(TimeSpan.FromMinutes(5));
```

> [!info] TimeProvider — modern way
> До .NET 8 — custom `IClock` interface. Сейчас — `TimeProvider` встроенный, поддерживается xUnit, Microsoft.Extensions.Testing.

См. [[../Testing/mocking-strategies|Mocking Strategies]].

### NodaTime (3rd-party) — для **serious** time work

```bash
dotnet add package NodaTime
```

Если работаешь с **timezone-heavy logic** — NodaTime > .NET BCL.

```csharp
using NodaTime;

Instant now = SystemClock.Instance.GetCurrentInstant();
DateTimeZone tz = DateTimeZoneProviders.Tzdb["America/New_York"];
ZonedDateTime zoned = now.InZone(tz);

LocalDate birthday = new LocalDate(1990, 5, 15);
LocalDateTime meeting = new LocalDateTime(2024, 1, 15, 14, 30);

Period age = Period.Between(birthday, LocalDate.FromDateTime(DateTime.Today));
Console.WriteLine($"{age.Years} years, {age.Months} months");
```

**Преимущества:**
- Different types для different concepts (Instant, ZonedDateTime, LocalDate)
- IANA timezone IDs (Windows + IANA tzdata)
- Calendar arithmetic правильно (months, years с DST)
- Strict typing — нельзя случайно перепутать LocalDate с ZonedDateTime

**Когда:**
- Расписания, scheduling
- Multi-timezone приложения
- Historical / future dates вычисления
- Banking, financial scenarios

---

## 2. UTC vs Local — main rule

### Default: всегда UTC

```csharp
// ✅ В коде — всегда UTC
DateTime utc = DateTime.UtcNow;
order.CreatedAt = DateTimeOffset.UtcNow;

// Database — UTC
public class Order
{
    public DateTime CreatedAt { get; set; }  // UTC всегда
}

// API JSON
return new { CreatedAt = order.CreatedAt };  // 2024-01-15T07:00:00Z
```

### Display — convert на boundary

```csharp
// ✅ Только для отображения user'у
public class OrderDto
{
    public DateTime CreatedAtLocal { get; set; }
    
    public OrderDto(Order order, TimeZoneInfo userTz)
    {
        CreatedAtLocal = TimeZoneInfo.ConvertTimeFromUtc(order.CreatedAt, userTz);
    }
}
```

### Проблемы Local time в коде

```csharp
// ❌ DateTime.Now — server's local time
DateTime now = DateTime.Now;
order.ExpiresAt = now.AddDays(30);

// Server в US, restart в EU — ExpiresAt совершенно другое!
// Если container deploy в other region — bugs!

// ✅
DateTime utcNow = DateTime.UtcNow;
order.ExpiresAt = utcNow.AddDays(30);
```

---

## 3. TimeZone

### TimeZoneInfo

```csharp
// Найти TZ
TimeZoneInfo nyTz = TimeZoneInfo.FindSystemTimeZoneById("America/New_York");
// или по Windows ID
TimeZoneInfo nyWin = TimeZoneInfo.FindSystemTimeZoneById("Eastern Standard Time");
// .NET 6+ Windows: оба IDs работают via ICU

// Все TZ
foreach (var tz in TimeZoneInfo.GetSystemTimeZones())
    Console.WriteLine(tz.Id);

// Local
TimeZoneInfo local = TimeZoneInfo.Local;
TimeZoneInfo utc = TimeZoneInfo.Utc;
```

### Conversion

```csharp
DateTime utcTime = DateTime.UtcNow;
TimeZoneInfo nyTz = TimeZoneInfo.FindSystemTimeZoneById("America/New_York");

// UTC → NY
DateTime ny = TimeZoneInfo.ConvertTimeFromUtc(utcTime, nyTz);

// Locale → UTC
DateTime fromLocal = TimeZoneInfo.ConvertTimeToUtc(localTime, sourceTimeZone);

// One TZ → another
DateTime targetTime = TimeZoneInfo.ConvertTime(sourceTime, sourceTz, targetTz);
```

### IANA vs Windows IDs

```
IANA:    "America/New_York", "Europe/Moscow"
Windows: "Eastern Standard Time", "Russian Standard Time"
```

В .NET 6+ на Windows — **оба формата работают** (через ICU library).

**Best practice:** используй **IANA IDs** — стандарт, кроссплатформенно, не меняются.

### DST — Daylight Saving Time

```csharp
TimeZoneInfo nyTz = TimeZoneInfo.FindSystemTimeZoneById("America/New_York");

DateTime january = new(2024, 1, 15, 12, 0, 0);  // EST (UTC-5)
DateTime july = new(2024, 7, 15, 12, 0, 0);     // EDT (UTC-4) — DST!

TimeZoneInfo.ConvertTimeFromUtc(january, nyTz);  // 7am EST
TimeZoneInfo.ConvertTimeFromUtc(july, nyTz);     // 8am EDT

// DST переход — gap или overlap
nyTz.IsAmbiguousTime(new DateTime(2024, 11, 3, 1, 30, 0));  // true (overlap)
nyTz.IsInvalidTime(new DateTime(2024, 3, 10, 2, 30, 0));    // true (gap)
```

> [!warning] DST gaps & overlaps
> - **Gap:** "spring forward" — 02:30 не существует
> - **Overlap:** "fall back" — 01:30 происходит дважды
>
> При scheduling — **сохраняй UTC**, не local. Иначе ambiguity.

---

## 4. Sereialization

### JSON

```csharp
// System.Text.Json (default 2026)
var dt = DateTimeOffset.UtcNow;
JsonSerializer.Serialize(dt);  // "2024-01-15T12:00:00+00:00"

DateTime localDt = DateTime.Now;
JsonSerializer.Serialize(localDt);  // "2024-01-15T15:00:00+05:00"
DateTime utcDt = DateTime.UtcNow;
JsonSerializer.Serialize(utcDt);    // "2024-01-15T10:00:00Z"
```

### Custom format

```csharp
public class DateTimeUtcConverter : JsonConverter<DateTime>
{
    public override DateTime Read(ref Utf8JsonReader reader, Type typeToConvert, JsonSerializerOptions options)
    {
        return DateTime.SpecifyKind(reader.GetDateTime(), DateTimeKind.Utc);
    }
    
    public override void Write(Utf8JsonWriter writer, DateTime value, JsonSerializerOptions options)
    {
        writer.WriteStringValue(value.ToUniversalTime().ToString("o"));  // ISO 8601
    }
}

// Регистрация
options.Converters.Add(new DateTimeUtcConverter());
```

### ISO 8601 — стандарт

```
2024-01-15T12:00:00.0000000Z          (UTC)
2024-01-15T12:00:00.0000000+05:00     (с offset)
2024-01-15T12:00:00                   (Unspecified — без TZ — bad!)
```

`"o"` format — round-trip ISO:
```csharp
DateTime dt = DateTime.UtcNow;
string s = dt.ToString("o");  // "2024-01-15T12:00:00.1234567Z"
DateTime back = DateTime.Parse(s);  // exact same value
```

---

## 5. Database

### EF Core + PostgreSQL

```csharp
public class Order
{
    public Guid Id { get; set; }
    
    // ✅ TIMESTAMPTZ в Postgres — UTC always
    public DateTime CreatedAt { get; set; }
    
    // ✅ DateTimeOffset тоже OK
    public DateTimeOffset UpdatedAt { get; set; }
}
```

В Npgsql 6+ — DateTime mapped к `timestamp with time zone` если `Kind = Utc`. Иначе — error.

```csharp
// ❌ Throws
order.CreatedAt = DateTime.Now;  // Local kind
db.SaveChanges();  // Error: Cannot write DateTime with Kind=Local

// ✅
order.CreatedAt = DateTime.UtcNow;
db.SaveChanges();  // OK
```

### EF Core + SQL Server

```csharp
// SQL Server datetime — без TZ info
public DateTime CreatedAt { get; set; }  // assume UTC

// SQL Server datetimeoffset — c TZ
public DateTimeOffset UpdatedAt { get; set; }
```

> [!info] PostgreSQL > SQL Server для time
> PG `timestamp with time zone` хранит **UTC внутри**, конвертирует на read. SQL Server `datetimeoffset` хранит **offset** — не identity.

См. [[../EFCore/basics-tracking|EF Core Basics]].

---

## 6. Common operations

### Arithmetic

```csharp
DateTime start = DateTime.UtcNow;
DateTime later = start.AddDays(7);
DateTime earlier = start.AddHours(-2);
DateTime exact = start.Add(TimeSpan.FromMinutes(30));

TimeSpan diff = later - start;  // 7 days
diff.TotalDays;     // 7.0
diff.TotalHours;    // 168.0
```

### Comparison

```csharp
DateTime a = DateTime.UtcNow;
DateTime b = a.AddMinutes(5);

a < b;            // true
a.CompareTo(b);   // -1

// Между датами
bool isBetween = date >= start && date <= end;
```

### Relative time

```csharp
public static string TimeAgo(DateTime utcDate)
{
    var diff = DateTime.UtcNow - utcDate;
    return diff switch
    {
        { TotalSeconds: < 60 } => "just now",
        { TotalMinutes: < 60 } d => $"{(int)d.TotalMinutes}m ago",
        { TotalHours: < 24 } d => $"{(int)d.TotalHours}h ago",
        { TotalDays: < 7 } d => $"{(int)d.TotalDays}d ago",
        _ => utcDate.ToString("yyyy-MM-dd")
    };
}
```

### Start/End of period

```csharp
DateTime startOfDay = DateTime.UtcNow.Date;  // midnight
DateTime endOfDay = startOfDay.AddDays(1).AddTicks(-1);

DateTime startOfWeek = DateTime.UtcNow.Date.AddDays(-(int)DateTime.UtcNow.DayOfWeek);
DateTime startOfMonth = new(DateTime.UtcNow.Year, DateTime.UtcNow.Month, 1);
DateTime endOfMonth = startOfMonth.AddMonths(1).AddDays(-1);
DateTime startOfYear = new(DateTime.UtcNow.Year, 1, 1);
```

### Day of week / month info

```csharp
DateTime d = new(2024, 1, 15);
d.DayOfWeek;        // Monday
d.DayOfYear;        // 15

DateTime.DaysInMonth(2024, 2);  // 29 (2024 leap)
DateTime.IsLeapYear(2024);       // true
```

---

## 7. Formatting

### Standard formats

```csharp
DateTime d = DateTime.UtcNow;

d.ToString("d");    // "1/15/2024" — short date
d.ToString("D");    // "Monday, January 15, 2024" — long date
d.ToString("t");    // "12:30 PM" — short time
d.ToString("T");    // "12:30:45 PM" — long time
d.ToString("g");    // "1/15/2024 12:30 PM" — short
d.ToString("G");    // "1/15/2024 12:30:45 PM" — long
d.ToString("o");    // "2024-01-15T12:30:45.1234567Z" — round-trip
d.ToString("s");    // "2024-01-15T12:30:45" — sortable
d.ToString("u");    // "2024-01-15 12:30:45Z" — universal
d.ToString("R");    // RFC1123 "Mon, 15 Jan 2024 12:30:45 GMT"
```

### Custom formats

```csharp
d.ToString("yyyy-MM-dd");                       // 2024-01-15
d.ToString("yyyy-MM-dd HH:mm:ss");              // 2024-01-15 12:30:45
d.ToString("yyyy-MM-dd'T'HH:mm:ss");
d.ToString("HH:mm:ss.fff");                     // 12:30:45.123
d.ToString("MMMM yyyy");                        // January 2024
d.ToString("dddd, MMMM d, yyyy");               // Monday, January 15, 2024

// Custom format codes:
// yyyy   — year (4 digit)
// yy     — year (2 digit)
// MM     — month (01-12)
// MMM    — month (Jan)
// MMMM   — month (January)
// dd     — day (01-31)
// ddd    — day (Mon)
// dddd   — day (Monday)
// HH     — hour (00-23)
// hh     — hour (01-12)
// mm     — minute
// ss     — second
// fff    — millisecond
// tt     — AM/PM
// K      — TZ designator (Z or +HH:mm)
```

### Culture

```csharp
DateTime d = new(2024, 1, 15);

d.ToString();                                              // depends on locale!
d.ToString(CultureInfo.InvariantCulture);                  // "01/15/2024 00:00:00"
d.ToString("d", CultureInfo.GetCultureInfo("de-DE"));      // "15.01.2024"
d.ToString("D", CultureInfo.GetCultureInfo("ru-RU"));      // "15 января 2024 г."
```

---

## 8. Parsing

### `DateTime.Parse` — danger

```csharp
// ❌ Зависит от culture
DateTime.Parse("01/02/2024");  // 1 Feb in EU, 2 Jan in US!

// ✅ Invariant + format
DateTime.ParseExact("01/02/2024", "MM/dd/yyyy", CultureInfo.InvariantCulture);

// ✅ ISO 8601
DateTime.Parse("2024-01-15T12:00:00Z", CultureInfo.InvariantCulture);
```

### TryParse — для validation

```csharp
if (DateTime.TryParseExact(input, "yyyy-MM-dd",
    CultureInfo.InvariantCulture, DateTimeStyles.None, out DateTime date))
{
    // valid
}
else
{
    // invalid format
}
```

### DateTimeStyles

```csharp
DateTimeStyles.AssumeUniversal;        // нет TZ → UTC
DateTimeStyles.AssumeLocal;            // нет TZ → Local
DateTimeStyles.AdjustToUniversal;      // convert к UTC

DateTime.Parse("2024-01-15 12:00",
    CultureInfo.InvariantCulture,
    DateTimeStyles.AssumeUniversal | DateTimeStyles.AdjustToUniversal);
// Result: UTC datetime
```

### Multiple formats

```csharp
string[] formats = { "yyyy-MM-dd", "MM/dd/yyyy", "dd.MM.yyyy" };
DateTime result;

if (DateTime.TryParseExact(input, formats, CultureInfo.InvariantCulture,
    DateTimeStyles.None, out result))
{
    // matched some format
}
```

---

## 9. Common Pitfalls

### 1. `DateTime.Now` в коде

```csharp
// ❌ Local TZ — server's, not user's!
order.CreatedAt = DateTime.Now;
order.ExpiresAt = DateTime.Now.AddDays(30);

// ✅ UTC всегда
order.CreatedAt = DateTime.UtcNow;
```

### 2. Hardcoded "Z" в string

```csharp
// ❌ Lying — фактически local time
string s = $"{DateTime.Now:yyyy-MM-dd}Z";  // ⚠️ "Z" но local time!

// ✅
string s = $"{DateTime.UtcNow:yyyy-MM-dd}Z";
```

### 3. Сравнение разных Kind

```csharp
DateTime localTime = DateTime.Now;
DateTime utcTime = DateTime.UtcNow;

// ❌ No conversion! Compares raw values
if (localTime > utcTime) { ... }  // могут быть unexpected results

// ✅ Convert to same kind
if (localTime.ToUniversalTime() > utcTime) { ... }
```

### 4. DST trap в schedules

```csharp
// User scheduled "every Monday at 9am local" — DST transition?
// Naive approach:
DateTime nextRun = DateTime.UtcNow.AddDays(7);

// ❌ В DST transition — fires at 8am or 10am local
// ✅ NodaTime или ручная обработка через TimeZoneInfo
```

### 5. SQL Server `datetime` precision

```csharp
DateTime now = DateTime.UtcNow;
// .NET DateTime: precision 100ns
// SQL Server datetime: precision 3.33ms

now.ToString("o");  // 2024-01-15T12:30:45.1234567Z
// После save в datetime: 2024-01-15T12:30:45.123Z
// Round-trip lost!

// ✅ datetime2 — precision 100ns (matches .NET)
[Column(TypeName = "datetime2")]
public DateTime CreatedAt { get; set; }
```

### 6. Caching DateTime values

```csharp
// ❌ Static field cached server start time
public static class Constants
{
    public static readonly DateTime AppStartTime = DateTime.UtcNow;
}

// При container restart — обновится. При scaling out — каждая instance имеет своё.
```

### 7. Birthday calculation

```csharp
// ❌ Unsafe для leap year birthdays
public int CalculateAge(DateTime birthday)
{
    return DateTime.Today.Year - birthday.Year;
}

// ✅ Properly check
public int CalculateAge(DateTime birthday)
{
    var today = DateTime.Today;
    var age = today.Year - birthday.Year;
    if (birthday.Date > today.AddYears(-age)) age--;
    return age;
}
```

### 8. `TotalMonths` not exists

```csharp
TimeSpan diff = end - start;
diff.TotalDays;     // ✅ exists
diff.TotalHours;    // ✅
diff.TotalMonths;   // ❌ doesn't exist!
// Months — variable length, не TimeSpan concept

// Нужно через DateTime arithmetic
int months = (end.Year - start.Year) * 12 + end.Month - start.Month;
```

### 9. Unix timestamps

```csharp
// To Unix
long unix = DateTimeOffset.UtcNow.ToUnixTimeSeconds();
long unixMs = DateTimeOffset.UtcNow.ToUnixTimeMilliseconds();

// From Unix
DateTimeOffset dt = DateTimeOffset.FromUnixTimeSeconds(unix);
```

### 10. JSON datetime gotcha

```csharp
// JSON:
{ "createdAt": "2024-01-15T12:00:00" }   // без TZ

// Deserialize:
public DateTime CreatedAt { get; set; }
// → Kind = Local (depends on parser!) — bad!

// ✅ Use DateTimeOffset
public DateTimeOffset CreatedAt { get; set; }
// JSON without offset → throws or gets local — explicit error!
```

---

## 11. Best Practices

### Storage

- **UTC always** в DB
- **DateTimeOffset > DateTime** для events (включает offset)
- **DateOnly / TimeOnly** (.NET 6+) для дат / времени без другого
- **datetime2** в SQL Server (precision matches .NET)
- **timestamptz** в PostgreSQL

### Code

- **`DateTime.UtcNow`** не `DateTime.Now`
- **`TimeProvider`** (.NET 8+) для testability
- **NodaTime** для серьёзной timezone work
- **Invariant culture** для serialization
- **Explicit timezone** в API responses (offset или Z)

### API

- **ISO 8601 / "o" format** в JSON
- **DateTimeOffset** в DTO для events
- **DateOnly** для birthdays, deadlines
- **User's TZ как parameter** или header — не assume

### Testing

- **TimeProvider.System** в production
- **FakeTimeProvider** в tests
- **Test DST transitions** explicitly если schedule logic
- **Multiple TZ** test cases

---

## 12. Cheat sheet

| Задача | Решение |
|--------|---------|
| Текущее время | `DateTime.UtcNow` (или TimeProvider) |
| Сохранить event | `DateTime.UtcNow` или `DateTimeOffset.UtcNow` |
| Birthday | `DateOnly` |
| Office hours | `TimeOnly` |
| Show user | `TimeZoneInfo.ConvertTimeFromUtc(utc, userTz)` |
| Format ISO | `dt.ToString("o")` |
| Parse safely | `DateTime.TryParseExact` + format + invariant |
| Diff | `end - start` returns `TimeSpan` |
| Add 1 month | `dt.AddMonths(1)` |
| Days between | `(end - start).TotalDays` |
| Unix timestamp | `DateTimeOffset.UtcNow.ToUnixTimeSeconds()` |
| Schedule | NodaTime + TimeZoneInfo |
| Test | `TimeProvider` + `FakeTimeProvider` |

---

## 13. Decision tree

```
Сохранять "когда что-то произошло"?
  → DateTime.UtcNow или DateTimeOffset.UtcNow
  → DB как UTC

Schedule в future?
  → DateTime.UtcNow + AddDays / TimeProvider
  → Если timezone matters (user's local) — NodaTime или ZonedDateTime

Display user'у?
  → Convert UTC к user's TZ только в presentation layer
  → Format с culture user'а

Birthday / deadline без time?
  → DateOnly (.NET 6+)

Office hours, alarm time?
  → TimeOnly (.NET 6+)

Duration / interval?
  → TimeSpan (для < 1 month)
  → Period (NodaTime) для months / years

Multi-TZ business logic?
  → NodaTime
  → IANA timezone IDs

Testability time?
  → TimeProvider (.NET 8+)
  → FakeTimeProvider в tests
```

---

## См. также

- [[modern-features|Modern C# Features]] — DateOnly, TimeOnly, TimeProvider
- [[functional-csharp|Functional C#]] — immutable date types
- [[csharp-language-design|C# Language Design]] — DateTime evolution
- [[../EFCore/basics-tracking|EF Core Basics]] — DateTime + DB
- [[../Testing/mocking-strategies|Mocking Strategies]] — TimeProvider testing
- [[../SQL/postgresql-deep|PostgreSQL Deep]] — timestamptz

## Reading list

- **Microsoft Docs — DateTime** — learn.microsoft.com/dotnet/standard/datetime
- **Microsoft Docs — TimeProvider** — learn.microsoft.com/dotnet/api/system.timeprovider
- **NodaTime documentation** — nodatime.org
- **Jon Skeet — NodaTime articles** — codeblog.jonskeet.uk
- **The case for NodaTime** — codeblog.jonskeet.uk/2010/11/24/why-noda-time/
- **Falsehoods programmers believe about time** — infiniteundo.com (must-read humor)
- **IANA Timezone Database** — iana.org/time-zones
- **Stephen Toub — DateTime performance** — devblogs.microsoft.com
