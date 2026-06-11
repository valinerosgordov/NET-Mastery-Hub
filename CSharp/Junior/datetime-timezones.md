---
tags: [csharp, datetime, timezones, junior, datetimeoffset, dateonly, timeonly, nodatime]
level: Junior
date: 2026-05-04
---

# DateTime и Timezones — даты, время, часовые пояса

> **Самая ошибочная тема в backend-коде.** `DateTime`, `DateTimeOffset`, `DateOnly`, `TimeOnly`, IANA vs Windows zones, DST, `DateTimeKind`, NodaTime, EF Core mapping. Закрывает пробел: «знаю, что есть `DateTime.Now`, не понимаю, почему оно ломает прод».

---

## 0. Как читать этот файл

Если ты впервые работаешь с датами в .NET — читай разделы 1→4 подряд: получишь работающую модель и поймёшь, **почему `DateTime.Now` опасен**. Если уже пишешь с timezone-ами — раздел 5 (DateTimeKind), 7 (TimeZoneInfo), 9 (DST). Если строишь production систему — раздел 11 (NodaTime), 12 (EF Core), 13 (JSON), 14 (best practices). Если в проекте уже миграция в timezone-aware — раздел 17 (workflow перехода).

Все примеры самостоятельные. `// expected: ...` показывает ожидаемый вывод (для time-зависимых операций — приблизительный). Cross-language якоря (`> [!info]-`) свёрнуты — раскрывай, если переходишь из Python / JS / Java / Go / Rust. Interview-вопросы (`> [!question]-`) встроены рядом с теорией.

---

## 1. Что это, зачем и когда

### 1.1. Почему даты — главная боль backend

Самые частые баги production-систем за последние 20 лет:

- Знаменитый **Y2K bug** — `1999` хранилось как `99`, при +1 год → `00`, а это **1900**.
- **DST transitions** — два часа ночи в день перехода случается дважды (или не случается вообще).
- **Server timezone mismatch** — приложение деплоится на Linux UTC, разработчики на Windows UTC+3, тесты падают «случайно» в зависимости от времени запуска.
- **Mobile app timezone** — пользователь летит из Токио в Нью-Йорк, но "Last seen" показывает время в Токио.
- **Database storage** — DateTime сохранён без timezone, через год меняется TZ сервера, все данные «сдвигаются».
- **JSON serialization** — клиент отправляет `"2024-05-15T10:00:00"`, сервер интерпретирует как UTC, реально это локальное время.

Все эти баги — следствие одной проблемы: **"datetime" — недостаточно, чтобы однозначно представить момент времени**. Нужны дополнительные данные: timezone, kind, offset.

### 1.2. Что такое момент времени

В реальности есть **три разных концепции**, которые джуниоры путают:

1. **Instant (момент во вселенной)** — конкретная точка на оси времени, одна для всех людей на Земле. Лучше всего представляется как **Unix timestamp** (секунды от 1970-01-01 UTC) или **UTC datetime**.
2. **Wall time (время на стене)** — то, что показывают часы в комнате. Зависит от timezone. «10:00 утра в Москве» — это конкретный wall time.
3. **Date / Time без момента** — «1 января 2024 года» (без времени) или «10:00» (без даты) — концепции, которые не привязаны к моменту во вселенной.

`DateTime` в C# мутирует между всеми тремя — поэтому такая боль.

`DateTimeOffset` чище — это всегда **instant** + **offset для отображения**.

`DateOnly` / `TimeOnly` (.NET 6+) — это явно **только date** или **только time**, без претензий на момент.

### 1.3. Главные типы — сводная таблица

| Тип | Что хранит | Имеет TZ? | Размер | Когда использовать |
|-----|-----------|-----------|--------|-------------------|
| `DateTime` | Date + Time | `Kind` (Utc/Local/Unspecified) | 8 байт | **Избегать**, если возможно. Legacy. |
| `DateTimeOffset` | Date + Time + Offset | Offset (`+03:00`) | 16 байт | Instant с контекстом отображения |
| `DateOnly` | Только date (year, month, day) | Нет | 4 байта | День рождения, дата отчёта |
| `TimeOnly` | Только time (hour, min, sec) | Нет | 8 байт | Время начала рабочего дня |
| `TimeSpan` | Длительность | Нет | 8 байт | Интервал, не момент |
| `TimeZoneInfo` | Информация о timezone | — | — | Конверсия между зонами |
| **NodaTime** `Instant` | UTC момент | Подразумевается UTC | 12 байт | Точный момент во вселенной |
| **NodaTime** `LocalDateTime` | Date + Time без TZ | Нет | 12 байт | Wall time без контекста |
| **NodaTime** `ZonedDateTime` | Instant + DateTimeZone | Да (named TZ) | 32 байта | Полная информация о моменте |

### 1.4. Главное правило: store UTC, display local

```
БД / wire format / API contracts:    UTC всегда
Бизнес-логика:                       UTC, либо ZonedDateTime
UI / Reports / User interaction:     local (с известным TZ пользователя)
```

Это базовая аксиома. Любые отклонения — потенциальные баги.

### 1.5. Эволюция: .NET 1.0 → .NET 10

| Версия | Год | Что появилось |
|--------|-----|---------------|
| **.NET 1.0** | 2002 | `DateTime`, `TimeSpan` |
| **.NET 2.0** | 2005 | `DateTimeKind` enum, `TimeZoneInfo` |
| **.NET 3.5 SP1** | 2008 | `DateTimeOffset` |
| **.NET 6** | 2021 | `DateOnly`, `TimeOnly` |
| **.NET 6** | 2021 | `TimeProvider` abstract type, IANA support на Windows |
| **.NET 8** | 2023 | Better Intl support, performance improvements |
| **.NET 10** | 2025 | Уточнения formatting, `DateOnly.ToDateTime` overloads |

### 1.6. Когда что использовать — quick decision

```
Хранить точный момент во вселенной?     → DateTimeOffset (UTC) или NodaTime Instant
Только дата (без времени)?                → DateOnly
Только время дня (без даты)?              → TimeOnly
Длительность / интервал?                  → TimeSpan
Конверсия между timezone?                 → TimeZoneInfo
Сложная calendar-логика?                  → NodaTime
Legacy код / interop с старыми API?       → DateTime (с осторожностью)
```

> [!info]- Если ты знаешь Python / JavaScript / Java / Go / Rust
> **Python:** `datetime` имеет наивные (без TZ) и aware варианты. `pytz` или `zoneinfo` (3.9+) для timezone. Близко по идее к C# `DateTime` (наивный) vs `DateTimeOffset` (aware). Главное — **explicit awareness**.
>
> **JavaScript:** `Date` объект — frustrating, всегда хранит как Unix timestamp + интерпретирует через системную TZ. `new Intl.DateTimeFormat()` для locale-aware форматирования. Библиотека `date-fns-tz` или `Luxon` — close к NodaTime.
>
> **Java:** до Java 8 — `java.util.Date` (как .NET DateTime, плохо). С Java 8 — `java.time` (`Instant`, `LocalDateTime`, `ZonedDateTime`, `LocalDate`, `LocalTime`) — точно копирует Joda Time / NodaTime. Самый чистый design в современных языках.
>
> **Go:** `time.Time` — единый тип, всегда содержит timezone. `time.UTC()` для конверсии. Минимально, но хорошо продумано — нет наивных datetime.
>
> **Rust:** стандартная библиотека skinny — `std::time::SystemTime`. Для серьёзной работы — `chrono` crate с `DateTime<Utc>`, `DateTime<Local>`, `NaiveDateTime`, `NaiveDate`. Очень похоже на NodaTime подходом.

> [!question]- Интервью: чем DateTime отличается от DateTimeOffset?
> `DateTime` хранит date + time + flag (`Kind`: Utc/Local/Unspecified), без явного offset. `DateTimeOffset` хранит date + time + явный offset (`+03:00`). Главная разница: `DateTimeOffset` однозначно идентифицирует момент во вселенной, `DateTime` — только если `Kind == Utc`. С `DateTime.Local` или `Unspecified` нельзя однозначно сказать, какой это instant. Best practice: для wire format / DB / instant-tracking — `DateTimeOffset` (или UTC `DateTime`), для wall time без контекста — отдельные типы.

---

## 2. DateTime — основы и ловушки

### 2.1. Создание DateTime

```csharp
// Конкретная дата и время
var d1 = new DateTime(2024, 5, 15, 10, 30, 0);   // 15 May 2024, 10:30:00 — Unspecified Kind
var d2 = new DateTime(2024, 5, 15);               // 15 May 2024, 00:00:00

// Текущее время — три варианта
var now1 = DateTime.Now;        // local time, Kind = Local
var now2 = DateTime.UtcNow;     // UTC, Kind = Utc
var today = DateTime.Today;     // сегодня в 00:00:00 local

// С Kind
var utc = new DateTime(2024, 5, 15, 10, 30, 0, DateTimeKind.Utc);
var local = new DateTime(2024, 5, 15, 10, 30, 0, DateTimeKind.Local);
var unspec = new DateTime(2024, 5, 15, 10, 30, 0, DateTimeKind.Unspecified);
```

### 2.2. Свойства

```csharp
var dt = new DateTime(2024, 5, 15, 10, 30, 45);

dt.Year;           // 2024
dt.Month;          // 5
dt.Day;            // 15
dt.Hour;           // 10
dt.Minute;         // 30
dt.Second;         // 45
dt.Millisecond;    // 0
dt.Microsecond;    // 0 (.NET 7+)
dt.Nanosecond;     // 0 (.NET 7+)
dt.DayOfWeek;      // Wednesday
dt.DayOfYear;      // 136
dt.Ticks;          // 638 514 414 450 000 000 (100-нс интервалов от 0001-01-01)
dt.Date;           // 15.05.2024 00:00:00
dt.TimeOfDay;      // TimeSpan 10:30:45
dt.Kind;           // Unspecified (мы не указали)
```

### 2.3. Арифметика

```csharp
var dt = new DateTime(2024, 5, 15);

// Add операции возвращают новый DateTime (immutable)
dt.AddDays(7);          // 22.05.2024
dt.AddHours(-5);        // 14.05.2024 19:00
dt.AddMonths(3);        // 15.08.2024
dt.AddYears(1);         // 15.05.2025
dt.AddMinutes(30);      // 15.05.2024 00:30
dt.Add(TimeSpan.FromMinutes(30));   // эквивалентно

// Difference — TimeSpan
var diff = new DateTime(2024, 12, 31) - new DateTime(2024, 1, 1);
diff.TotalDays;   // 365
diff.TotalHours;  // 8760
```

### 2.4. **Главная ловушка**: DateTime.Now vs DateTime.UtcNow

```csharp
DateTime.Now;       // зависит от TZ на машине
DateTime.UtcNow;    // всегда UTC, не зависит от TZ
```

Проблема `DateTime.Now`:

```csharp
// Тест на машине разработчика (Москва, UTC+3)
public void TestSomething()
{
    var deadline = DateTime.Now.AddDays(1);
    SaveToDatabase(deadline);   // 16.05.2024 12:00:00 (Москва) — Kind=Local
}

// Тот же код на сервере (UTC)
// deadline становится 16.05.2024 09:00:00 (UTC) — Kind=Local
// При чтении из БД информация о TZ потеряна!
```

`DateTime.Now` — это **антипаттерн в backend-коде**. Используй `DateTime.UtcNow`. Local time — только при отображении пользователю.

### 2.5. **Вторая ловушка**: DateTimeKind не сохраняется в БД

Когда ты сохраняешь `DateTime` в БД, информация о `Kind` **теряется** (для большинства провайдеров и колонок). При чтении ты получишь `DateTime` с `Kind = Unspecified`:

```csharp
var utcDt = DateTime.UtcNow;          // Kind = Utc
db.SaveChanges();                      // SQL: '2024-05-15 10:30:45' — без TZ информации
// ...
var fromDb = db.Records.First().Date;  // Kind = Unspecified! Не Utc!
```

Как итог: `fromDb.ToLocalTime()` может работать неправильно, потому что система не знает, что это UTC.

**Решения** см. в разделе 12 (EF Core mapping).

### 2.6. **Третья ловушка**: формат ISO 8601 vs русский

```csharp
var dt = new DateTime(2024, 5, 15, 10, 30, 0);

dt.ToString();                       // "15.05.2024 10:30:00" на Windows ru-RU
dt.ToString(CultureInfo.InvariantCulture);   // "05/15/2024 10:30:00"
dt.ToString("yyyy-MM-dd HH:mm:ss");          // "2024-05-15 10:30:00"
dt.ToString("o");                            // "2024-05-15T10:30:00.0000000" (round-trip)
dt.ToString("u");                            // "2024-05-15 10:30:00Z" (universal sortable)
```

**Никогда не парси/форматируй даты без указания CultureInfo**. На разных серверах настройка по умолчанию разная — будут странные баги.

```csharp
// ❌ Может работать на dev, ломаться на prod
DateTime.Parse("05/15/2024");

// ✅ Всегда явно
DateTime.ParseExact("05/15/2024", "MM/dd/yyyy", CultureInfo.InvariantCulture);
DateTime.Parse("2024-05-15T10:30:00Z", CultureInfo.InvariantCulture, DateTimeStyles.RoundtripKind);
```

### 2.7. Tick — низкоуровневое представление

```csharp
DateTime.MinValue.Ticks;   // 0 — 0001-01-01 00:00:00.0000000
DateTime.MaxValue.Ticks;   // 3 155 378 975 999 999 999 — 9999-12-31 23:59:59.9999999
```

Один `Tick` = 100 наносекунд. `DateTime` хранится как `long` внутри. Точность — 100ns.

`Ticks` полезен для:
- Точного вычитания и сравнения.
- Сериализации (хранить `long` компактно).
- Hash key для коротких диапазонов.

Не путай с `Environment.TickCount` (миллисекунды от старта системы) и `Stopwatch.GetTimestamp()` (high-precision counter).

> [!question]- Интервью: почему `DateTime.Now` опасен в backend?
> 1) Возвращает local time зависимо от TZ сервера. На dev (Москва) и prod (UTC) код ведёт себя по-разному. 2) `Kind = Local` теряется при сохранении в БД — после чтения ты не знаешь, какой это TZ. 3) При DST переходах два часа ночи случается дважды — local DateTime неоднозначен. 4) При сериализации в JSON может потерять offset. Best practice: в backend всегда `DateTime.UtcNow`. Local time — только в UI слое для конкретного пользователя с известным TZ.

---

## 3. DateTimeOffset — лучшая альтернатива

### 3.1. Что хранит

`DateTimeOffset` = `DateTime` + явный `TimeSpan` offset:

```csharp
var dto = new DateTimeOffset(2024, 5, 15, 10, 30, 0, TimeSpan.FromHours(3));
// 15.05.2024 10:30:00 +03:00

dto.UtcDateTime;       // 15.05.2024 07:30:00 (Utc)
dto.LocalDateTime;     // 15.05.2024 локально (Local)
dto.DateTime;          // 15.05.2024 10:30:00 (Unspecified)
dto.Offset;            // 03:00:00
dto.UtcTicks;          // тики UTC момента
```

Ключевое: **`DateTimeOffset` всегда однозначно идентифицирует момент во вселенной**. Два `DateTimeOffset` равны если и только если они представляют один и тот же UTC момент.

### 3.2. Создание

```csharp
DateTimeOffset.Now;       // local + системный TZ offset
DateTimeOffset.UtcNow;    // UTC, offset = +00:00

// Из DateTime
var local = new DateTime(2024, 5, 15, 10, 30, 0, DateTimeKind.Local);
new DateTimeOffset(local);   // offset берётся из системного TZ

// С явным offset
new DateTimeOffset(2024, 5, 15, 10, 30, 0, TimeSpan.FromHours(3));

// Из строки
DateTimeOffset.Parse("2024-05-15T10:30:00+03:00");
DateTimeOffset.ParseExact("2024-05-15T10:30:00+03:00", "yyyy-MM-ddTHH:mm:sszzz", CultureInfo.InvariantCulture);
```

### 3.3. Equality по UTC моменту

```csharp
var a = new DateTimeOffset(2024, 5, 15, 13, 30, 0, TimeSpan.FromHours(3));   // 13:30 +03:00
var b = new DateTimeOffset(2024, 5, 15, 10, 30, 0, TimeSpan.Zero);            // 10:30 +00:00

a == b;          // true! Тот же UTC момент: 10:30 UTC
a.Equals(b);     // true
a.UtcTicks == b.UtcTicks;   // true
```

В отличие от `DateTime`:

```csharp
var a = new DateTime(2024, 5, 15, 13, 30, 0, DateTimeKind.Local);
var b = new DateTime(2024, 5, 15, 10, 30, 0, DateTimeKind.Utc);

a == b;          // false — разное представление, хотя возможно тот же момент
```

### 3.4. ToOffset — конверсия в другой TZ для отображения

```csharp
var dto = DateTimeOffset.UtcNow;   // 10:30 +00:00

dto.ToOffset(TimeSpan.FromHours(3));    // 13:30 +03:00 (Москва)
dto.ToOffset(TimeSpan.FromHours(-5));   // 05:30 -05:00 (NYC EST)
dto.ToOffset(TimeSpan.FromHours(9));    // 19:30 +09:00 (Токио)
```

Это **не меняет** UTC момент — только то, как он отображается.

### 3.5. Сравнение operations

```csharp
var a = DateTimeOffset.UtcNow;
var b = a.AddHours(1);

a < b;           // true
b - a;           // 01:00:00 (TimeSpan)
a.AddDays(7);    // через неделю
```

Все операции работают по UTC момент, не по wall time.

### 3.6. **Ловушка**: DateTimeOffset не сохраняет timezone, только offset

```csharp
var d = new DateTimeOffset(2024, 5, 15, 10, 30, 0, TimeSpan.FromHours(3));
// "Москва" или "Сирия" или "Стамбул"? Все +03:00.
```

`DateTimeOffset` знает только **смещение**, не **timezone**. Для DST-aware операций это критично:

```csharp
var winter = new DateTimeOffset(2024, 1, 15, 10, 30, 0, TimeSpan.FromHours(3));   // Европа: +03:00 в зимнее время? Зависит от страны
var summer = new DateTimeOffset(2024, 7, 15, 10, 30, 0, TimeSpan.FromHours(3));   // Европа: +02:00 в летнее время для большинства стран

// DateTimeOffset не различит "+03:00 in summer Europe" vs "Moscow" (всегда +03:00 без DST)
```

Если нужен реальный timezone, используй `TimeZoneInfo` (раздел 7) или NodaTime `ZonedDateTime` (раздел 11).

### 3.7. Когда DateTimeOffset

Используй `DateTimeOffset` когда:

- Хочешь сохранить момент во вселенной + контекст offset (например, лог события «случилось в 13:30 +03:00»).
- Передаёшь wire format (JSON/API) — клиент видит и UTC момент, и locale.
- Сравниваешь моменты — equality работает корректно.
- БД-колонка типа `datetimeoffset` (SQL Server) или `timestamptz` (PostgreSQL).

Не используй когда:

- Нужно учитывать DST правила — `DateTimeOffset` хранит offset на момент создания, не правило.
- Нужна именованная TZ — используй `TimeZoneInfo` или NodaTime.
- Только дата без времени — `DateOnly`.

> [!question]- Интервью: чем offset отличается от timezone?
> Offset — это статическая величина смещения от UTC (`+03:00`, `-05:00`). Timezone — это **правило**, которое определяет offset в каждый момент времени с учётом DST переходов и исторических изменений. Например, "America/New_York" — это timezone, который зимой = `-05:00`, летом = `-04:00`. `DateTimeOffset` хранит offset, не timezone — для DST-aware операций недостаточно. Для именованных TZ нужен `TimeZoneInfo` (Windows IDs или IANA с .NET 6+) или NodaTime `DateTimeZone`.

---

## 4. DateOnly и TimeOnly (.NET 6+)

### 4.1. Зачем эти типы

До .NET 6 для «только даты» использовали `DateTime` с временем 00:00:00:

```csharp
// Старый стиль
var birthday = new DateTime(1990, 5, 15);   // 00:00:00 — но это «время» бессмысленно!

// Проблемы:
birthday.AddHours(5);   // becomes 05:00:00 — неинтуитивно для даты
birthday.Hour;          // 0 — но почему оно даже есть?
birthday.ToString();    // "15.05.1990 00:00:00" — лишняя часть
```

`DateOnly` и `TimeOnly` решают это явно:

```csharp
var birthday = new DateOnly(1990, 5, 15);
birthday.Year;       // 1990
birthday.AddDays(7); // 22.05.1990
birthday.AddYears(1);// 15.05.1991
// birthday.Hour    // ❌ нет такого свойства

var workStart = new TimeOnly(9, 0);
workStart.Hour;      // 9
workStart.AddHours(8); // 17:00:00
```

### 4.2. DateOnly API

```csharp
var d = new DateOnly(2024, 5, 15);

d.Year;           // 2024
d.Month;          // 5
d.Day;            // 15
d.DayOfWeek;      // Wednesday
d.DayOfYear;      // 136
d.DayNumber;      // 738996 (количество дней от 0001-01-01)

// Арифметика
d.AddDays(7);     // 22.05.2024
d.AddMonths(3);   // 15.08.2024
d.AddYears(1);    // 15.05.2025

// Парсинг / форматирование
DateOnly.Parse("2024-05-15");
DateOnly.ParseExact("15/05/2024", "dd/MM/yyyy", CultureInfo.InvariantCulture);
d.ToString("yyyy-MM-dd");   // "2024-05-15"

// Конверсия
d.ToDateTime(TimeOnly.MinValue);   // DateTime 15.05.2024 00:00:00 (Kind = Unspecified)
d.ToDateTime(new TimeOnly(10, 30)); // DateTime 15.05.2024 10:30:00
```

### 4.3. TimeOnly API

```csharp
var t = new TimeOnly(10, 30, 45);

t.Hour;          // 10
t.Minute;        // 30
t.Second;        // 45
t.Millisecond;   // 0
t.Ticks;         // тики от полуночи

// Арифметика
t.AddHours(2);     // 12:30:45
t.AddMinutes(30);  // 11:00:45
t.Add(TimeSpan.FromMinutes(15));   // эквивалентно

// Парсинг / форматирование
TimeOnly.Parse("10:30");
TimeOnly.ParseExact("10:30:45", "HH:mm:ss");
t.ToString("HH:mm:ss");   // "10:30:45"

// Сравнение
t.IsBetween(new TimeOnly(9, 0), new TimeOnly(18, 0));   // true (между 9:00 и 18:00)
```

### 4.4. Когда DateOnly

```csharp
public class User
{
    public DateOnly Birthday { get; set; }       // ✅ день рождения — только дата
    public DateOnly RegistrationDate { get; set; }   // ✅ дата регистрации
    public DateTimeOffset LastLogin { get; set; }    // ✅ момент во вселенной
    public DateTimeOffset CreatedAt { get; set; }    // ✅
}

public class Holiday
{
    public string Name { get; set; } = "";
    public DateOnly Date { get; set; }   // ✅ праздник — это дата, не момент
}
```

Используй `DateOnly` когда:

- День рождения, дата свадьбы, дата основания компании.
- Дата праздника, выходного.
- Дата отчёта, billing period.
- Любая дата без привязки к моменту во вселенной (без timezone).

### 4.5. Когда TimeOnly

```csharp
public class StoreSchedule
{
    public DayOfWeek Day { get; set; }
    public TimeOnly Open { get; set; }    // ✅ "9:00" — время открытия
    public TimeOnly Close { get; set; }   // ✅ "21:00"
}

public class Alarm
{
    public string Name { get; set; } = "";
    public TimeOnly Time { get; set; }    // ✅ "07:00" — будильник
    public DayOfWeek[] Days { get; set; } = [];
}
```

Используй `TimeOnly` когда:

- Время открытия/закрытия магазина (одинаковое каждый день).
- Время будильника.
- Расписание занятий (без даты).
- Час начала рабочего дня.

### 4.6. **Ловушки** DateOnly / TimeOnly

**JSON serialization до .NET 7:**

```csharp
// .NET 6 — System.Text.Json НЕ умел DateOnly/TimeOnly из коробки
var dto = new MyClass { Date = new DateOnly(2024, 5, 15) };
JsonSerializer.Serialize(dto);
// ❌ Throws — нет конвертера

// Решение: AddJsonOptions кастомный converter, или использовать в .NET 7+
```

В **.NET 7+** работает встроенно: сериализуется как `"2024-05-15"`.

**EF Core маппинг:**

С EF Core 8+ — встроенная поддержка `DateOnly`/`TimeOnly` для большинства провайдеров. Маппится в SQL Server `date` / `time` соответственно.

**Subtraction между DateOnly:**

```csharp
var a = new DateOnly(2024, 1, 1);
var b = new DateOnly(2024, 5, 15);

// b - a;   // ❌ Compile error до .NET 9 — нет минус-оператора
// .NET 9+: возвращает int (количество дней)

// Универсально:
int days = b.DayNumber - a.DayNumber;   // 135
```

> [!question]- Интервью: когда использовать DateOnly вместо DateTime?
> Когда у понятия нет attached time of day и timezone: день рождения, дата праздника, регистрация, billing period. Семантически это «день в году», а не «момент во вселенной». `DateTime` для этого технически работает, но `DateTime` имеет лишнее (`Hour`, `Minute`, `Kind`) и провоцирует ошибки (например, `birthday.AddHours(5)` некорректно). `DateOnly` явно говорит «только дата», IDE и компилятор могут помочь избежать misuse.

---

## 5. DateTimeKind — три состояния

### 5.1. Что это

`DateTime.Kind` — это enum с тремя значениями:

```csharp
public enum DateTimeKind
{
    Unspecified = 0,
    Utc = 1,
    Local = 2
}
```

Это **подсказка** для системы: «как интерпретировать этот datetime». Сам datetime — просто числа (year/month/.../second/ticks). `Kind` говорит, **что они означают**.

### 5.2. Откуда берётся Kind

```csharp
DateTime.UtcNow.Kind;        // Utc
DateTime.Now.Kind;            // Local
DateTime.Today.Kind;          // Local
new DateTime(2024, 5, 15).Kind;   // Unspecified (не указали)
new DateTime(2024, 5, 15, 0, 0, 0, DateTimeKind.Utc).Kind;   // Utc
```

`Unspecified` — самая частая ловушка. Это «я не знаю, какой это TZ». При операциях типа `ToLocalTime()` система **предполагает**, что Unspecified = UTC, что часто неверно.

### 5.3. ToUniversalTime() и ToLocalTime()

```csharp
DateTime utc = new DateTime(2024, 5, 15, 10, 30, 0, DateTimeKind.Utc);
DateTime local = utc.ToLocalTime();      // конверсия в local TZ системы
local.Kind;                              // Local

DateTime localToUtc = local.ToUniversalTime();   // обратно в UTC
localToUtc.Kind;                                  // Utc
```

**Ловушка:** что произойдёт с `Unspecified`?

```csharp
DateTime unspec = new DateTime(2024, 5, 15, 10, 30, 0);   // Unspecified

unspec.ToLocalTime();        // система: «считаю Unspecified как Utc, конвертирую в Local»
unspec.ToUniversalTime();    // система: «считаю Unspecified как Local, конвертирую в Utc»
```

В первом случае получишь смещение `system TZ`, во втором — `-system TZ`. Оба варианта ошибочны если ты на самом деле имел в виду другое.

**Правило:** не используй `ToUniversalTime`/`ToLocalTime` на `Unspecified` без явной интерпретации.

### 5.4. SpecifyKind — задать Kind без конверсии

```csharp
DateTime unspec = new DateTime(2024, 5, 15, 10, 30, 0);   // Unspecified

// Просто пометить как UTC — без конверсии
DateTime asUtc = DateTime.SpecifyKind(unspec, DateTimeKind.Utc);
// Те же числа, но теперь Kind = Utc
```

Используй когда **знаешь**, что значение хранилось как UTC, но Kind потерялся (например, после чтения из БД).

### 5.5. Workflow с БД

Самый частый сценарий:

```csharp
// Запись
var dt = DateTime.UtcNow;          // Kind = Utc
db.Records.Add(new Record { Date = dt });
db.SaveChanges();
// SQL: INSERT ... '2024-05-15 10:30:00'  (Kind не сохранился)

// Чтение
var record = db.Records.First();
var fromDb = record.Date;          // Kind = Unspecified (для большинства провайдеров)

// Если попытаться конвертировать без assertions:
fromDb.ToLocalTime();              // ⚠️ Wrong! Считает Unspecified как UTC, может work, но неявно

// Правильно — явно указать, что мы знаем Kind
DateTime asUtc = DateTime.SpecifyKind(fromDb, DateTimeKind.Utc);
asUtc.ToLocalTime();               // ✅ Корректно
```

См. раздел 12 для EF Core конвертеров, которые делают это автоматически.

### 5.6. Альтернатива — использовать DateTimeOffset

```csharp
public class Record
{
    public int Id { get; set; }
    public DateTimeOffset Date { get; set; }   // ✅ хранит offset, не теряется
}

// Запись
db.Records.Add(new Record { Date = DateTimeOffset.UtcNow });
db.SaveChanges();
// SQL Server: datetimeoffset колонка хранит offset
// PostgreSQL: timestamptz хранит UTC момент
```

Это самое чистое решение. `DateTime` + Kind — boilerplate.

> [!question]- Интервью: что произойдёт, если `DateTime.SpecifyKind(unspec, DateTimeKind.Utc).ToLocalTime()` на сервере с TZ Europe/Moscow?
> `SpecifyKind` помечает datetime как UTC без изменения чисел. Затем `ToLocalTime()` интерпретирует как UTC и конвертирует в локальный TZ сервера (Europe/Moscow = +03:00 без DST). Результат: к исходным числам прибавляется 3 часа. Если изначальные числа на самом деле были local time — получишь сдвиг на двойную TZ. Используй `SpecifyKind` только когда **уверен** в источнике данных.

---

## 6. Парсинг и форматирование

### 6.1. Стандартные форматы

```csharp
var dt = new DateTime(2024, 5, 15, 10, 30, 45);

dt.ToString("d");    // "15.05.2024" (short date, locale)
dt.ToString("D");    // "15 мая 2024 г." (long date, locale)
dt.ToString("t");    // "10:30" (short time)
dt.ToString("T");    // "10:30:45" (long time)
dt.ToString("g");    // "15.05.2024 10:30" (short datetime)
dt.ToString("G");    // "15.05.2024 10:30:45" (long datetime)
dt.ToString("o");    // "2024-05-15T10:30:45.0000000" (ISO 8601 round-trip)
dt.ToString("s");    // "2024-05-15T10:30:45" (sortable ISO без ms)
dt.ToString("u");    // "2024-05-15 10:30:45Z" (universal sortable, всегда UTC)
dt.ToString("R");    // "Wed, 15 May 2024 10:30:45 GMT" (RFC 1123)
dt.ToString("f");    // "15 мая 2024 г. 10:30"
dt.ToString("F");    // "15 мая 2024 г. 10:30:45"
dt.ToString("M");    // "15 мая" (month/day)
dt.ToString("Y");    // "май 2024 г." (year/month)
```

### 6.2. Custom форматы — placeholders

```csharp
dt.ToString("yyyy-MM-dd HH:mm:ss");    // "2024-05-15 10:30:45"
dt.ToString("dd/MM/yyyy");             // "15/05/2024"
dt.ToString("MMMM dd, yyyy");          // "май 15, 2024"
dt.ToString("HH:mm:ss.fff");           // "10:30:45.000"
dt.ToString("yyyy-MM-ddTHH:mm:ss.fffZ"); // "2024-05-15T10:30:45.000Z"
```

Главные placeholders:

| Token | Что | Пример |
|-------|-----|--------|
| `yyyy` | Год 4 цифры | 2024 |
| `yy` | Год 2 цифры | 24 |
| `MM` | Месяц 2 цифры | 05 |
| `MMM` | Сокр. имя месяца | май |
| `MMMM` | Полное имя месяца | мая |
| `dd` | День месяца 2 цифры | 15 |
| `ddd` | Сокр. имя дня недели | срд |
| `dddd` | Полное имя | среда |
| `HH` | Час 24-часовой формат | 10 |
| `hh` | Час 12-часовой формат | 10 |
| `mm` | Минуты | 30 |
| `ss` | Секунды | 45 |
| `fff` | Миллисекунды | 123 |
| `tt` | AM/PM | AM |
| `zzz` | Offset в формате `+03:00` | +03:00 |

### 6.3. Invariant culture — обязательно для wire format

```csharp
// ❌ Может работать на ru-RU, ломаться на en-US
dt.ToString("dd/MM/yyyy");   // "15/05/2024" на ru-RU, "15/05/2024" на en-US — везде одинаково
dt.ToString("d");            // "15.05.2024" на ru-RU, "5/15/2024" на en-US — разные!

// ✅ Invariant — стабильно
dt.ToString("dd/MM/yyyy", CultureInfo.InvariantCulture);
```

**Правило:** для wire format (JSON, log, БД, файл) **всегда** `CultureInfo.InvariantCulture`. Для UI — current culture или explicitly specified.

### 6.4. ISO 8601 — стандарт для wire format

```csharp
var utc = DateTime.UtcNow;

// Round-trip — best для serialization
utc.ToString("o", CultureInfo.InvariantCulture);
// "2024-05-15T10:30:45.1234567Z" — Z указывает на UTC

var local = DateTime.Now;
local.ToString("o", CultureInfo.InvariantCulture);
// "2024-05-15T13:30:45.1234567+03:00" — offset указан

// Парсинг round-trip
DateTime.Parse("2024-05-15T10:30:45.1234567Z", CultureInfo.InvariantCulture, DateTimeStyles.RoundtripKind);
```

Для `DateTimeOffset`:

```csharp
DateTimeOffset.UtcNow.ToString("o", CultureInfo.InvariantCulture);
// "2024-05-15T10:30:45.1234567+00:00"
```

ISO 8601 — стандарт всех серьёзных API. Используй везде, кроме UI.

### 6.5. ParseExact — самый строгий

```csharp
// Гарантированно парсится точно по формату — не зависит от culture defaults
var dt = DateTime.ParseExact(
    "15/05/2024 10:30",
    "dd/MM/yyyy HH:mm",
    CultureInfo.InvariantCulture);

// Несколько форматов — попробует все
var dt2 = DateTime.ParseExact(
    "15/05/2024",
    new[] { "dd/MM/yyyy", "yyyy-MM-dd", "MM/dd/yyyy" },
    CultureInfo.InvariantCulture,
    DateTimeStyles.None);
```

### 6.6. TryParse — для пользовательского input

```csharp
if (DateTime.TryParse(userInput, CultureInfo.InvariantCulture, DateTimeStyles.None, out var parsed))
{
    // OK
}
else
{
    // Валидация: показать пользователю
}
```

Используй `TryParse` всегда для пользовательского input — он не throw, возвращает bool.

### 6.7. DateTimeStyles — флаги поведения

```csharp
DateTime.Parse("2024-05-15T10:30:45Z", CultureInfo.InvariantCulture, DateTimeStyles.RoundtripKind);
// Сохраняет Kind из строки (Utc, потому что Z)

DateTime.Parse("2024-05-15T10:30:45Z", CultureInfo.InvariantCulture, DateTimeStyles.AdjustToUniversal);
// Сразу возвращает в UTC

DateTime.Parse("2024-05-15T10:30:45+03:00", CultureInfo.InvariantCulture, DateTimeStyles.AssumeUniversal);
// Если нет offset в строке — считает как UTC

DateTime.Parse("2024-05-15", CultureInfo.InvariantCulture, DateTimeStyles.AssumeLocal);
// Если нет offset — local
```

**Правило:** для round-trip JSON/wire — `RoundtripKind`. Для UI input — `AssumeLocal` (если ввод от пользователя в его TZ).

> [!question]- Интервью: чем `DateTime.Parse` отличается от `ParseExact`?
> `Parse` — пробует много форматов, использует culture defaults, может вести себя по-разному на разных машинах. `ParseExact` — строго по указанным форматам, fail если не подошёл. Для wire format (JSON, лог) **всегда** `ParseExact` или `Parse(... InvariantCulture, RoundtripKind)`. `Parse` без `InvariantCulture` опасен — на сервере с другой locale ломается. Для пользовательского ввода — `TryParse` с `CurrentCulture`, чтобы не throw.

---

## 7. TimeZoneInfo — работа с timezone

### 7.1. Что это

`TimeZoneInfo` — представление timezone как именованной сущности с правилами DST:

```csharp
TimeZoneInfo.Local;   // системный TZ
TimeZoneInfo.Utc;     // UTC

TimeZoneInfo.FindSystemTimeZoneById("Europe/Moscow");        // IANA (.NET 6+ на Windows)
TimeZoneInfo.FindSystemTimeZoneById("Russian Standard Time"); // Windows ID
TimeZoneInfo.FindSystemTimeZoneById("America/New_York");      // IANA с DST
```

С .NET 6 на Windows работает и Windows ID и IANA ID. На Linux/macOS — всегда IANA.

### 7.2. Список всех TZ

```csharp
foreach (var tz in TimeZoneInfo.GetSystemTimeZones())
{
    Console.WriteLine($"{tz.Id} — {tz.DisplayName}");
}

// На Linux/macOS:
// Africa/Abidjan — (UTC+00:00) Africa/Abidjan
// America/New_York — (UTC-05:00) America/New_York
// ...

// На Windows:
// Russian Standard Time — (UTC+03:00) Москва, Санкт-Петербург
// Eastern Standard Time — (UTC-05:00) Eastern Time (US & Canada)
// ...
```

### 7.3. Конверсия между TZ

```csharp
var moscow = TimeZoneInfo.FindSystemTimeZoneById("Europe/Moscow");
var ny = TimeZoneInfo.FindSystemTimeZoneById("America/New_York");

// UTC → конкретный TZ
DateTime utc = DateTime.UtcNow;
DateTime moscowTime = TimeZoneInfo.ConvertTimeFromUtc(utc, moscow);
DateTime nyTime = TimeZoneInfo.ConvertTimeFromUtc(utc, ny);

// TZ → UTC
DateTime moscowLocal = new DateTime(2024, 5, 15, 10, 30, 0, DateTimeKind.Unspecified);
DateTime utc2 = TimeZoneInfo.ConvertTimeToUtc(moscowLocal, moscow);

// TZ → TZ
DateTime nyLocal = TimeZoneInfo.ConvertTime(moscowLocal, moscow, ny);
```

### 7.4. Проверка DST

```csharp
var ny = TimeZoneInfo.FindSystemTimeZoneById("America/New_York");

ny.SupportsDaylightSavingTime;   // true

DateTime summer = new DateTime(2024, 7, 15, 10, 0, 0);
DateTime winter = new DateTime(2024, 1, 15, 10, 0, 0);

ny.IsDaylightSavingTime(summer);   // true
ny.IsDaylightSavingTime(winter);   // false

ny.GetUtcOffset(summer);   // -04:00 (EDT)
ny.GetUtcOffset(winter);   // -05:00 (EST)
```

### 7.5. IANA на Windows (.NET 6+)

До .NET 6 на Windows можно было использовать только Windows-style IDs (`"Russian Standard Time"`). На Linux/macOS — только IANA (`"Europe/Moscow"`).

С .NET 6 — **обе работают**:

```csharp
// Оба варианта работают на любой ОС (.NET 6+)
TimeZoneInfo.FindSystemTimeZoneById("Europe/Moscow");
TimeZoneInfo.FindSystemTimeZoneById("Russian Standard Time");

// Конверсия ID между форматами
var winId = TimeZoneInfo.TryConvertIanaIdToWindowsId("Europe/Moscow", out var w);   // "Russian Standard Time"
var ianaId = TimeZoneInfo.TryConvertWindowsIdToIanaId("Russian Standard Time", out var i);   // "Europe/Moscow"
```

**Правило:** в новом коде **всегда** IANA (`Europe/Moscow`, `America/New_York`, `Asia/Tokyo`). Это стандарт всего остального мира, кроссплатформенно, не deprecate-ится.

### 7.6. **Ловушка**: TZ database обновляется отдельно

Когда страны меняют TZ-правила (например, Россия отменила DST в 2014), .NET runtime читает из системы. Это значит:

- На Linux/macOS — TZ database в `/usr/share/zoneinfo/` обновляется через пакетный менеджер.
- На Windows — TZ через Registry, обновляется через Windows Update.
- Контейнеры — нужно регулярно обновлять base image.

Если приложение работает с историческими датами — обновления TZ database могут изменить интерпретацию старых datetime. В критичных случаях (страховые контракты, исторические данные) используй NodaTime с явной версией tzdata.

### 7.7. Список IANA TZ

| IANA ID | Описание |
|---------|----------|
| `UTC` | UTC |
| `Europe/Moscow` | Москва (+03:00, без DST с 2014) |
| `Europe/London` | Лондон (GMT/BST с DST) |
| `Europe/Berlin` | Берлин (CET/CEST с DST) |
| `Europe/Paris` | Париж (CET/CEST с DST) |
| `America/New_York` | Нью-Йорк (EST/EDT с DST) |
| `America/Los_Angeles` | LA (PST/PDT с DST) |
| `America/Chicago` | Чикаго (CST/CDT) |
| `Asia/Tokyo` | Токио (+09:00, без DST) |
| `Asia/Shanghai` | Шанхай (+08:00, без DST) |
| `Asia/Dubai` | Дубай (+04:00, без DST) |
| `Asia/Kolkata` | Индия (+05:30, без DST) |
| `Australia/Sydney` | Сидней (с DST) |

Полный список: https://en.wikipedia.org/wiki/List_of_tz_database_time_zones

> [!question]- Интервью: какой ID использовать — IANA или Windows?
> Всегда IANA (`Europe/Moscow`). Это стандарт worldwide, кроссплатформенно (Linux/macOS/Windows с .NET 6+), не deprecated. Windows IDs (`Russian Standard Time`) — legacy. С .NET 6 на Windows IANA работает встроенно. Если нужен interop с Windows-only legacy — `TimeZoneInfo.TryConvertIanaIdToWindowsId` для конверсии.

---

## 8. UTC vs Local — best practice

### 8.1. Server-side: всегда UTC

В backend-коде везде используй UTC:

```csharp
// ✅ Правильно
public class OrderService
{
    public Order Create(decimal amount)
    {
        return new Order
        {
            Amount = amount,
            CreatedAt = DateTime.UtcNow,    // или DateTimeOffset.UtcNow
        };
    }
}

// ❌ Неправильно
public class OrderService
{
    public Order Create(decimal amount)
    {
        return new Order
        {
            CreatedAt = DateTime.Now,    // local time! зависит от TZ сервера
        };
    }
}
```

### 8.2. Client-side: convert to user TZ при отображении

```csharp
public string FormatForUser(DateTimeOffset utcMoment, string userTimeZoneId)
{
    var userTz = TimeZoneInfo.FindSystemTimeZoneById(userTimeZoneId);
    var userLocal = TimeZoneInfo.ConvertTime(utcMoment, userTz);
    return userLocal.ToString("dd MMM yyyy HH:mm", CultureInfo.CurrentCulture);
}

// Для пользователя в Москве
string display = FormatForUser(order.CreatedAt, "Europe/Moscow");
// "15 май 2024 13:30"
```

### 8.3. Storing user TZ

```csharp
public class User
{
    public int Id { get; set; }
    public string Email { get; set; } = "";
    public string TimeZoneId { get; set; } = "UTC";   // IANA ID, по умолчанию UTC
}

// При регистрации — спросить у пользователя или determine от browser
public IActionResult Register(RegisterRequest req)
{
    var user = new User
    {
        Email = req.Email,
        TimeZoneId = req.TimeZoneId   // например "Europe/Moscow"
    };
    // ...
}
```

В UI-приложении (web) обычно:
- Frontend (browser) знает user TZ через `Intl.DateTimeFormat().resolvedOptions().timeZone`.
- Отправляет на server.
- Server хранит и использует для формирования отчётов / уведомлений.

### 8.4. Wall clock alarms — особый case

Что значит «напомни в 9 утра каждый день»? Это не UTC момент, это **wall time + recurrence**:

```csharp
public class Reminder
{
    public TimeOnly TimeOfDay { get; set; }    // 09:00
    public string TimeZoneId { get; set; } = ""; // в чьём TZ?
}

// На сервере scheduler пересчитывает следующий запуск:
public DateTimeOffset NextOccurrence(Reminder r)
{
    var tz = TimeZoneInfo.FindSystemTimeZoneById(r.TimeZoneId);
    var nowInUserTz = TimeZoneInfo.ConvertTime(DateTimeOffset.UtcNow, tz);
    var todayAt = new DateTimeOffset(
        nowInUserTz.Year, nowInUserTz.Month, nowInUserTz.Day,
        r.TimeOfDay.Hour, r.TimeOfDay.Minute, 0,
        tz.GetUtcOffset(nowInUserTz));

    if (todayAt > nowInUserTz) return todayAt.ToUniversalTime();
    // Сегодня уже прошло — завтра
    return todayAt.AddDays(1).ToUniversalTime();
}
```

В UTC это «следующий момент, когда у пользователя 9:00». При DST переходе UTC момент сдвинется на час, но wall time останется 9:00.

### 8.5. Aggregate / report queries — UTC range

Для отчётов «orders за май 2024»:

```csharp
// ❌ Неправильно — без TZ контекста, неясно чей май
var fromMay = db.Orders.Where(o => o.CreatedAt >= new DateTime(2024, 5, 1));

// ✅ Правильно — определить, чей "май"
var userTz = TimeZoneInfo.FindSystemTimeZoneById("Europe/Moscow");
var startOfMay = TimeZoneInfo.ConvertTimeToUtc(
    new DateTime(2024, 5, 1, 0, 0, 0, DateTimeKind.Unspecified), userTz);
var endOfMay = TimeZoneInfo.ConvertTimeToUtc(
    new DateTime(2024, 6, 1, 0, 0, 0, DateTimeKind.Unspecified), userTz);

var orders = db.Orders
    .Where(o => o.CreatedAt >= startOfMay && o.CreatedAt < endOfMay);
```

«Май в Москве» = с 30 апреля 21:00 UTC до 31 мая 21:00 UTC. Без явной конверсии запрос будет включать неправильные часы границ.

### 8.6. Логирование

```csharp
// ✅ В логах всегда UTC с явным offset
_logger.LogInformation("Order created at {CreatedAt:o}", DateTimeOffset.UtcNow);
// "Order created at 2024-05-15T10:30:45.123+00:00"

// При просмотре в Seq/ELK — все отметки в UTC, легко сортировать через сервера в разных TZ.
```

> [!question]- Интервью: почему DateTime.UtcNow лучше DateTime.Now в backend?
> 1) Не зависит от TZ сервера (deploy на Linux UTC vs Windows local — поведение одинаково). 2) Не имеет неоднозначностей при DST (DateTime.Now во время "fall back" может быть одним и тем же значением для двух разных моментов). 3) Конверсия в локальное время для отображения — отдельный шаг, легко тестируется. 4) При сохранении в БД и чтении обратно момент сохраняется (если правильно настроен Kind / DateTimeOffset). 5) Сравнение между serveрами в разных TZ работает без сюрпризов.

---

## 9. DST — Daylight Saving Time

### 9.1. Что это

Daylight Saving Time — практика сдвигать часы на 1 час весной и обратно осенью для экономии светлого времени. Применяется в большинстве стран Европы, Северной Америки, Австралии и др. Россия отменила DST в 2014.

DST создаёт **две аномалии в году**:

1. **Spring forward** — часы переводятся +1, час пропадает: 02:00 → 03:00. Время 02:30 в этот день **не существует**.
2. **Fall back** — часы переводятся −1, час повторяется: 03:00 → 02:00. Время 02:30 в этот день случается **дважды**.

### 9.2. Почему это критично для backend

```csharp
// Сценарий: alarm на 02:30 в США
var alarmTime = new DateTime(2024, 3, 10, 2, 30, 0);   // Spring forward в США 10 марта
var ny = TimeZoneInfo.FindSystemTimeZoneById("America/New_York");

// Это локальное время может не существовать!
ny.IsInvalidTime(alarmTime);   // true!

// Что делать?
TimeZoneInfo.ConvertTimeToUtc(alarmTime, ny);
// ❌ ArgumentException: время не существует в указанном TZ
```

```csharp
// Fall back — 03:00 → 02:00 в США 3 ноября 2024
var ambiguous = new DateTime(2024, 11, 3, 1, 30, 0);
ny.IsAmbiguousTime(ambiguous);   // true!

ny.GetAmbiguousTimeOffsets(ambiguous);
// [-04:00 (EDT), -05:00 (EST)] — два возможных offset для одного wall time
```

### 9.3. Обработка — `IsInvalidTime` / `IsAmbiguousTime`

```csharp
public DateTimeOffset SafeConvertToUtc(DateTime localTime, TimeZoneInfo tz)
{
    if (tz.IsInvalidTime(localTime))
    {
        // Время не существует — обычно сдвигаем на час вперёд
        localTime = localTime.AddHours(1);
    }

    if (tz.IsAmbiguousTime(localTime))
    {
        // Двусмысленно — выбираем первый offset (до перехода)
        var offsets = tz.GetAmbiguousTimeOffsets(localTime);
        return new DateTimeOffset(localTime, offsets[0]);
    }

    var offset = tz.GetUtcOffset(localTime);
    return new DateTimeOffset(localTime, offset);
}
```

Какой выбирать в ambiguous case — зависит от business rules. Часто берут первый (DST до перехода) или явно показывают пользователю выбор.

### 9.4. **Самая частая боль**: scheduler / cron

```csharp
// Cron job: «каждый день в 02:30»
public async Task RunDailyMaintenance()
{
    var now = DateTimeOffset.UtcNow;
    var nextRun = NextScheduledRun(now);   // 02:30 user-tz
    await Task.Delay(nextRun - now);
    DoMaintenance();
}
```

Что произойдёт 10 марта (spring forward) для пользователя в Нью-Йорке? 02:30 не существует — задача либо пропустится, либо запустится дважды (если scheduler не учёл).

Решения:
- **Не использовать time-of-day в DST-affected hours** (избегать 02:00-03:00 окна).
- **Использовать UTC для scheduling** — никаких сюрпризов.
- **Использовать NodaTime** — DST-aware из коробки.

### 9.5. Как проверить переход

```csharp
TimeZoneInfo ny = TimeZoneInfo.FindSystemTimeZoneById("America/New_York");

// Получить все DST правила для TZ
var rules = ny.GetAdjustmentRules();
foreach (var rule in rules)
{
    Console.WriteLine($"From {rule.DateStart:yyyy-MM-dd} to {rule.DateEnd:yyyy-MM-dd}");
    Console.WriteLine($"  DST start: {rule.DaylightTransitionStart}");
    Console.WriteLine($"  DST end: {rule.DaylightTransitionEnd}");
    Console.WriteLine($"  Delta: {rule.DaylightDelta}");   // обычно +01:00
}
```

### 9.6. **Россия — особый случай**

С октября 2014 года в России DST отменён. Все TZ в России — фиксированные:

```csharp
TimeZoneInfo.FindSystemTimeZoneById("Europe/Moscow").SupportsDaylightSavingTime;
// false — без DST

// Но некоторые TZ остались с DST правилами для исторических данных
TimeZoneInfo.FindSystemTimeZoneById("Europe/Moscow").GetAdjustmentRules();
// До 2014 были DST правила — для дат до 2014 они применяются
```

### 9.7. Best practice по DST

1. **Backend всегда UTC** — DST переходы прозрачны.
2. **Алармы и scheduling — UTC + recurrence rule**.
3. **Display только** — конверсия в user TZ при выводе, всё остальное UTC.
4. **Для сложных календарных расчётов** — NodaTime с явной семантикой `ZonedDateTime` (раздел 11).

> [!question]- Интервью: что произойдёт с DateTime в момент DST spring forward?
> Spring forward (например, 10 марта 2024 в США): часы прыгают с 02:00 сразу на 03:00. Местное время в диапазоне 02:00-03:00 **не существует** в этот день. Если в коде есть `new DateTime(2024, 3, 10, 2, 30, 0)` и попытка `TimeZoneInfo.ConvertTimeToUtc(..., ny)` — `ArgumentException`. Решение: проверить через `tz.IsInvalidTime(...)` и сдвинуть на час вперёд (или throw business error). Аналогично fall back создаёт ambiguous time — час повторяется. Защита: backend в UTC, scheduler-задачи на UTC, для wall clock алармов использовать NodaTime.

---

## 10. Stopwatch и измерение времени

### 10.1. DateTime.Now не для измерений

```csharp
// ❌ Неправильный способ замерить интервал
var start = DateTime.UtcNow;
DoWork();
var elapsed = DateTime.UtcNow - start;
```

Проблемы:
- **Низкая точность** — `DateTime.UtcNow` имеет точность около 15ms на Windows (ОС timer).
- **Не монотонно** — если NTP синхронизирует системные часы, `DateTime` может «прыгнуть» назад. Для длительных операций результат может быть отрицательным.
- **Изменение TZ** — теоретически (хотя и редко) может повлиять.

### 10.2. Stopwatch — правильный инструмент

```csharp
using System.Diagnostics;

var sw = Stopwatch.StartNew();
DoWork();
sw.Stop();

Console.WriteLine(sw.Elapsed);              // TimeSpan
Console.WriteLine(sw.ElapsedMilliseconds);  // long ms
Console.WriteLine(sw.ElapsedTicks);          // long, high-resolution ticks
```

`Stopwatch` использует high-resolution performance counter (на Windows `QueryPerformanceCounter`, на Linux `clock_gettime(CLOCK_MONOTONIC)`). Точность — наносекунды, монотонно (не скачет назад).

### 10.3. Stopwatch API

```csharp
// Создание и старт за одну строку
var sw = Stopwatch.StartNew();

// Pause / resume
sw.Stop();
sw.Start();   // продолжает с того же места — не сбрасывает

// Reset — обнулить
sw.Reset();
sw.Start();

// Restart — обнулить и запустить заново
sw.Restart();

// Прочитать без остановки
var current = sw.Elapsed;
// продолжаем...
```

### 10.4. Timestamp (статически, без объекта)

```csharp
long start = Stopwatch.GetTimestamp();
DoWork();
long end = Stopwatch.GetTimestamp();

// .NET 7+
var elapsed = Stopwatch.GetElapsedTime(start, end);

// До .NET 7
double seconds = (end - start) / (double)Stopwatch.Frequency;
TimeSpan ts = TimeSpan.FromSeconds(seconds);
```

`GetTimestamp()` без объекта — для микро-измерений в hot loop. Меньше allocations.

### 10.5. **Ловушка**: Stopwatch для сравнения с другим источником

```csharp
// ❌ Stopwatch и DateTime — независимые counter'ы
var sw = Stopwatch.StartNew();
// 10 секунд работы
var elapsed = sw.Elapsed;
var dtNow = DateTime.UtcNow;

// elapsed.Ticks ≠ (dtNow - dtStart).Ticks даже если измеряют одно и то же
```

Stopwatch измеряет **прошедшее время**, DateTime — **календарное время**. Они принципиально разные. Не пытайся конвертировать одно в другое для абсолютных вычислений.

### 10.6. .NET 6+ TimeProvider — abstraction для тестов

```csharp
public class OrderService
{
    private readonly TimeProvider _time;
    public OrderService(TimeProvider time) => _time = time;

    public Order Create(decimal amount)
    {
        return new Order
        {
            Amount = amount,
            CreatedAt = _time.GetUtcNow()    // вместо DateTimeOffset.UtcNow
        };
    }
}

// В production — TimeProvider.System (реальное время)
// В тестах — FakeTimeProvider (контролируемое)

// xunit + Microsoft.Extensions.Time.Testing
[Fact]
public void CreatesOrderWithCorrectDate()
{
    var fakeTime = new FakeTimeProvider(new DateTimeOffset(2024, 5, 15, 10, 0, 0, TimeSpan.Zero));
    var service = new OrderService(fakeTime);
    var order = service.Create(100m);
    Assert.Equal(new DateTimeOffset(2024, 5, 15, 10, 0, 0, TimeSpan.Zero), order.CreatedAt);

    // Шагнуть вперёд во времени
    fakeTime.Advance(TimeSpan.FromHours(1));
    var nextOrder = service.Create(50m);
    Assert.Equal(new DateTimeOffset(2024, 5, 15, 11, 0, 0, TimeSpan.Zero), nextOrder.CreatedAt);
}
```

`TimeProvider` — стандарт .NET 8+ для testable time. Заменяет ad-hoc обёртки (`IDateTimeProvider`).

> [!question]- Интервью: почему Stopwatch лучше DateTime для измерений?
> 1) Точность — Stopwatch использует high-resolution performance counter (наносекунды), DateTime — около 15ms на Windows. 2) Монотонность — Stopwatch не «прыгает» при NTP-синхронизации системных часов. DateTime может уйти назад, давая отрицательные интервалы. 3) Не зависит от TZ изменений. Для измерения скорости / latency / benchmarking — только Stopwatch (или `Stopwatch.GetTimestamp` для микро-оптимизации). DateTime.UtcNow — для записи момента события, не для измерения интервалов.

---

## 11. NodaTime — лучшая альтернатива

### 11.1. Зачем NodaTime

NodaTime — библиотека от Jon Skeet, портированная с Joda Time (Java). Адресует все недостатки `DateTime`:

- **Чёткое разделение типов** — `Instant`, `LocalDateTime`, `ZonedDateTime`, `LocalDate`, `LocalTime`, `OffsetDateTime`, `Duration`, `Period`.
- **Невозможны boilerplate-ошибки** — нельзя случайно сравнить wall time с UTC момент.
- **DST правильно** — `ZonedDateTime` обрабатывает spring forward / fall back корректно.
- **Иммутабельность** — все типы immutable.
- **Тестируемость** — `IClock` interface.

```bash
dotnet add package NodaTime
```

### 11.2. Главные типы

```csharp
using NodaTime;

// Instant — точка во вселенной (UTC)
Instant now = SystemClock.Instance.GetCurrentInstant();   // эквивалент DateTimeOffset.UtcNow
Instant.FromUtc(2024, 5, 15, 10, 30, 0);

// LocalDateTime — wall time БЕЗ TZ (только дата + время)
LocalDateTime ldt = new LocalDateTime(2024, 5, 15, 10, 30, 0);

// ZonedDateTime — wall time + TZ
DateTimeZone moscow = DateTimeZoneProviders.Tzdb["Europe/Moscow"];
ZonedDateTime zdt = ldt.InZoneStrictly(moscow);   // ARGUMENT exception если invalid time

// OffsetDateTime — wall time + offset (без TZ rules)
OffsetDateTime odt = new OffsetDateTime(ldt, Offset.FromHours(3));

// LocalDate / LocalTime
LocalDate date = new LocalDate(2024, 5, 15);
LocalTime time = new LocalTime(10, 30, 0);

// Duration — точная длительность (UTC-based, не DST-aware)
Duration d = Duration.FromHours(2);

// Period — календарная длительность (DST-aware)
Period p = Period.FromMonths(1).Plus(Period.FromDays(5));
```

### 11.3. Конверсии

```csharp
// Instant → ZonedDateTime
Instant instant = SystemClock.Instance.GetCurrentInstant();
ZonedDateTime moscowTime = instant.InZone(moscow);

// ZonedDateTime → Instant
Instant utc = zdt.ToInstant();

// LocalDateTime → ZonedDateTime (с обработкой DST)
LocalDateTime ldt = new LocalDateTime(2024, 5, 15, 10, 30, 0);

// Strictly — throw на invalid/ambiguous
ZonedDateTime zdt1 = ldt.InZoneStrictly(moscow);

// Leniently — лояльно к DST (выбирает разумно)
ZonedDateTime zdt2 = ldt.InZoneLeniently(moscow);
```

### 11.4. DST правильно из коробки

```csharp
// Spring forward — invalid time
LocalDateTime invalid = new LocalDateTime(2024, 3, 10, 2, 30, 0);   // 02:30 не существует в NY DST

DateTimeZone ny = DateTimeZoneProviders.Tzdb["America/New_York"];

invalid.InZoneStrictly(ny);   // throws SkippedTimeException
invalid.InZoneLeniently(ny);  // сдвигает на 03:30 (через переход)
```

```csharp
// Fall back — ambiguous time
LocalDateTime ambig = new LocalDateTime(2024, 11, 3, 1, 30, 0);

ambig.InZoneStrictly(ny);     // throws AmbiguousTimeException
ambig.InZoneLeniently(ny);    // выбирает earlier offset
```

Никакой неявной магии — ошибки явные, обработка осознанная.

### 11.5. Period vs Duration

```csharp
// Duration — фиксированный интервал (всегда столько же UTC секунд)
Duration day = Duration.FromHours(24);

// Period — календарный интервал (DST-aware)
Period oneDay = Period.FromDays(1);
```

```csharp
LocalDateTime spring = new LocalDateTime(2024, 3, 9, 22, 0, 0);
DateTimeZone ny = DateTimeZoneProviders.Tzdb["America/New_York"];
ZonedDateTime zdt = spring.InZoneLeniently(ny);

// Plus(Duration) — добавляет UTC секунды
ZonedDateTime afterDuration = zdt.Plus(Duration.FromHours(24));
// afterDuration.LocalDateTime = 23:00 (через DST переход прошло)

// Plus(Period) — добавляет календарные часы
ZonedDateTime afterPeriod = zdt.LocalDateTime.Plus(Period.FromDays(1)).InZoneLeniently(ny);
// afterPeriod.LocalDateTime = 22:00 (тот же wall time)
```

Это разница, которую `DateTime` не выражает. Для DST-aware операций — `Period`. Для UTC интервалов — `Duration`.

### 11.6. IClock — для тестов

```csharp
public class OrderService
{
    private readonly IClock _clock;
    public OrderService(IClock clock) => _clock = clock;

    public Order Create(decimal amount) =>
        new Order { Amount = amount, CreatedAt = _clock.GetCurrentInstant() };
}

// Production
var service = new OrderService(SystemClock.Instance);

// Тест
var fakeClock = new FakeClock(Instant.FromUtc(2024, 5, 15, 10, 30, 0));
var service = new OrderService(fakeClock);
```

### 11.7. NodaTime в JSON / EF Core

```bash
dotnet add package NodaTime.Serialization.SystemTextJson
dotnet add package NodaTime.Serialization.JsonNet
```

```csharp
// System.Text.Json
var options = new JsonSerializerOptions();
options.ConfigureForNodaTime(DateTimeZoneProviders.Tzdb);

// EF Core (через Npgsql.EntityFrameworkCore.PostgreSQL.NodaTime для Postgres)
services.AddDbContext<AppDb>(opts => opts.UseNpgsql(connStr, o => o.UseNodaTime()));
```

### 11.8. Когда NodaTime

✅ **Используй когда:**
- Серьёзная работа с timezone / DST.
- Финансовые / медицинские / юридические системы (точность критична).
- Календарная логика (subscriptions, billing periods).
- Расписания с recurring events.
- Тестируемое время (через IClock).

❌ **Не нужен когда:**
- Простой CRUD без timezone-логики.
- Только UTC моменты, нет конверсий.
- Маленький проект, дополнительная зависимость не оправдана.

> [!question]- Интервью: чем NodaTime лучше встроенного DateTime?
> 1) Чёткое разделение типов: `Instant` (UTC момент), `LocalDateTime` (wall time без TZ), `ZonedDateTime` (с TZ), `LocalDate`/`LocalTime` (только дата/время) — нельзя случайно перепутать. 2) DST обрабатывается правильно — `InZoneStrictly` throws на invalid/ambiguous, `InZoneLeniently` лояльно. 3) `Period` (календарный) vs `Duration` (UTC) — две концепции явно разделены. 4) `IClock` interface для тестируемости (с .NET 8 встроен `TimeProvider`, но NodaTime более полный). 5) Иммутабельность всех типов. NodaTime — стандарт серьёзного backend в .NET; для простых задач избыточен, для финансов/медицины/юриспруденции почти обязателен.

---

## 12. EF Core — mapping и storage

### 12.1. Колонки и типы — SQL Server

```csharp
public class Order
{
    public int Id { get; set; }
    public DateTime CreatedAt { get; set; }            // → datetime2 (без TZ)
    public DateTimeOffset UpdatedAt { get; set; }      // → datetimeoffset (с offset)
    public DateOnly OrderDate { get; set; }             // → date
    public TimeOnly DeliveryWindow { get; set; }        // → time
    public DateTime? ShippedAt { get; set; }            // → datetime2 NULL
}
```

| .NET тип | SQL Server | PostgreSQL | MySQL |
|----------|-----------|-----------|-------|
| `DateTime` (Utc/Local) | `datetime2` | `timestamp` | `datetime` |
| `DateTime` (Utc, явно UTC) | `datetime2` | `timestamp` | `datetime` |
| `DateTimeOffset` | `datetimeoffset` | `timestamptz` | (нет, эмулируется через datetime + offset колонку) |
| `DateOnly` | `date` | `date` | `date` |
| `TimeOnly` | `time` | `time` | `time` |
| `TimeSpan` | `time` (если < 24ч) или `bigint` | `interval` | `time` |

### 12.2. PostgreSQL — рекомендуемый

PostgreSQL имеет лучшую поддержку времени:

- `timestamp` — без TZ (только wall time).
- `timestamptz` — UTC момент, при чтении конвертируется в session TZ.
- Через NodaTime провайдер — `Instant`, `ZonedDateTime`, `LocalDateTime` маппятся отдельно.

```csharp
public class Order
{
    public int Id { get; set; }
    public DateTimeOffset CreatedAt { get; set; }   // → timestamptz
    public DateOnly OrderDate { get; set; }          // → date
}
```

### 12.3. **Главная проблема**: DateTime Kind не сохраняется

При сохранении `DateTime.UtcNow` в SQL Server колонку `datetime2` — Kind теряется. При чтении получаем `DateTime` с `Kind = Unspecified`:

```csharp
var order = new Order { CreatedAt = DateTime.UtcNow };   // Kind = Utc
db.SaveChanges();
// SQL: INSERT INTO Orders ('2024-05-15 10:30:45')

// Чтение позже
var loaded = db.Orders.First();
loaded.CreatedAt.Kind;   // Unspecified! Не Utc!
```

Это типичный bug: код «работал» потому что Kind всё равно правильно использовался в API, но при `.ToLocalTime()` получали неправильный результат.

### 12.4. Решение #1: ValueConverter для UTC

```csharp
public class UtcValueConverter : ValueConverter<DateTime, DateTime>
{
    public UtcValueConverter()
        : base(
            v => v.ToUniversalTime(),                                    // в БД
            v => DateTime.SpecifyKind(v, DateTimeKind.Utc))              // из БД
    { }
}

// В DbContext:
protected override void OnModelCreating(ModelBuilder mb)
{
    var utcConverter = new UtcValueConverter();

    foreach (var entityType in mb.Model.GetEntityTypes())
    {
        foreach (var prop in entityType.GetProperties())
        {
            if (prop.ClrType == typeof(DateTime) || prop.ClrType == typeof(DateTime?))
                prop.SetValueConverter(utcConverter);
        }
    }
}
```

Теперь все `DateTime` в проекте автоматически Kind=Utc. Никаких ручных `SpecifyKind`.

### 12.5. Решение #2: использовать DateTimeOffset везде

```csharp
public class Order
{
    public DateTimeOffset CreatedAt { get; set; }   // SQL Server: datetimeoffset
}
```

Offset сохраняется в БД, при чтении восстанавливается. Никаких ловушек с Kind.

Для wire format и JSON это тоже более явный формат:

```json
{ "createdAt": "2024-05-15T10:30:45.123+00:00" }
```

vs DateTime, где offset зависит от системы.

### 12.6. Решение #3: NodaTime + EF Core

Для PostgreSQL:

```bash
dotnet add package Npgsql.EntityFrameworkCore.PostgreSQL.NodaTime
```

```csharp
public class Order
{
    public int Id { get; set; }
    public Instant CreatedAt { get; set; }    // → timestamptz
    public LocalDate OrderDate { get; set; }  // → date
}

services.AddDbContext<AppDb>(opts =>
    opts.UseNpgsql(connStr, o => o.UseNodaTime()));
```

NodaTime типы напрямую маппятся в PostgreSQL native типы. Никаких ValueConverter, никаких Kind issues.

### 12.7. Запросы с date range

```csharp
// Получить orders за май 2024 в Москве
var moscowTz = TimeZoneInfo.FindSystemTimeZoneById("Europe/Moscow");
var monthStart = new DateTime(2024, 5, 1, 0, 0, 0, DateTimeKind.Unspecified);
var monthEnd = monthStart.AddMonths(1);

var startUtc = TimeZoneInfo.ConvertTimeToUtc(monthStart, moscowTz);
var endUtc = TimeZoneInfo.ConvertTimeToUtc(monthEnd, moscowTz);

var orders = await db.Orders
    .Where(o => o.CreatedAt >= startUtc && o.CreatedAt < endUtc)
    .ToListAsync();
```

EF Core транслирует это в SQL с UTC timestamps. Всё работает, если БД хранит UTC.

### 12.8. SQL `GETDATE()` vs C# `DateTime.UtcNow`

```csharp
// ❌ DB-side now (зависит от TZ сервера БД)
modelBuilder.Entity<Order>()
    .Property(o => o.CreatedAt)
    .HasDefaultValueSql("GETDATE()");   // SQL Server local TZ

// ✅ DB-side UTC
modelBuilder.Entity<Order>()
    .Property(o => o.CreatedAt)
    .HasDefaultValueSql("GETUTCDATE()");

// ✅ App-side UTC (явный)
modelBuilder.Entity<Order>()
    .Property(o => o.CreatedAt)
    .HasDefaultValueSql("SYSDATETIMEOFFSET()");   // SQL Server: с offset
```

**Правило:** для defaults в БД — UTC-функции, не local. Иначе сюрпризы при миграции БД на другой сервер.

> [!question]- Интервью: что произойдёт, если сохранить `DateTime.UtcNow` в SQL Server колонку `datetime2`?
> Сохранятся числа года/месяца/.../секунды/миллисекунды, но Kind не сохраняется. При чтении `Kind = Unspecified`. Это значит, что `loaded.ToLocalTime()` системно интерпретирует Unspecified как UTC и применяет local TZ — может работать, но небезопасно. Решения: 1) `ValueConverter<DateTime, DateTime>` который при чтении делает `SpecifyKind(value, Utc)`. 2) Использовать `DateTimeOffset` (колонка `datetimeoffset` сохраняет offset). 3) NodaTime `Instant` для чёткого UTC. Лучший подход — `DateTimeOffset`, чище всего.

---

## 13. JSON serialization

### 13.1. System.Text.Json — встроенно

```csharp
using System.Text.Json;

var dt = DateTime.UtcNow;                       // Kind = Utc
JsonSerializer.Serialize(dt);
// "2024-05-15T10:30:45.1234567Z"

var dto = DateTimeOffset.UtcNow;
JsonSerializer.Serialize(dto);
// "2024-05-15T10:30:45.1234567+00:00"

var local = DateTime.Now;                       // Kind = Local
JsonSerializer.Serialize(local);
// "2024-05-15T13:30:45.1234567+03:00" — offset из системного TZ
```

Десериализация:

```csharp
JsonSerializer.Deserialize<DateTime>("\"2024-05-15T10:30:45Z\"");
// Kind = Utc

JsonSerializer.Deserialize<DateTime>("\"2024-05-15T10:30:45\"");
// Kind = Unspecified! Без Z или offset.

JsonSerializer.Deserialize<DateTimeOffset>("\"2024-05-15T10:30:45+03:00\"");
// Offset = +03:00
```

### 13.2. **Самая частая боль**: timezone в JSON непонятен

Что значит JSON `"2024-05-15T10:30:45"`? Зависит от контекста. Без `Z` или offset — неоднозначно.

**Best practice для public API:**
- ВСЕГДА `Z` (UTC) или `+HH:MM` (offset).
- НИКОГДА naive datetime без TZ context.
- ISO 8601 формат (`yyyy-MM-ddTHH:mm:ss.fffZ`).

```json
{
  "createdAt": "2024-05-15T10:30:45.123Z",
  "scheduledAt": "2024-05-20T15:00:00+03:00",
  "birthday": "1990-05-15"
}
```

### 13.3. JsonSerializerOptions — настройка

```csharp
var options = new JsonSerializerOptions
{
    PropertyNamingPolicy = JsonNamingPolicy.CamelCase,
    WriteIndented = true,
    // DateTime / DateTimeOffset форматирование — встроено, ISO 8601 по умолчанию
};
```

С .NET 7+ `DateOnly` и `TimeOnly` сериализуются автоматически:

```json
"orderDate": "2024-05-15",
"deliveryTime": "15:00:00"
```

### 13.4. Кастомный converter для legacy формата

```csharp
public class CustomDateTimeConverter : JsonConverter<DateTime>
{
    private const string Format = "yyyy-MM-dd HH:mm:ss";

    public override DateTime Read(ref Utf8JsonReader reader, Type type, JsonSerializerOptions opts) =>
        DateTime.ParseExact(reader.GetString()!, Format, CultureInfo.InvariantCulture, DateTimeStyles.AssumeUniversal);

    public override void Write(Utf8JsonWriter writer, DateTime value, JsonSerializerOptions opts) =>
        writer.WriteStringValue(value.ToUniversalTime().ToString(Format, CultureInfo.InvariantCulture));
}
```

Используй когда внешняя система требует свой формат и поменять нельзя.

### 13.5. Newtonsoft.Json — для legacy проектов

```csharp
using Newtonsoft.Json;

JsonConvert.SerializeObject(dt);
// "2024-05-15T10:30:45.1234567Z"

JsonConvert.SerializeObject(dt, new JsonSerializerSettings
{
    DateFormatString = "yyyy-MM-ddTHH:mm:ss.fffZ",
    DateFormatHandling = DateFormatHandling.IsoDateFormat,
    DateTimeZoneHandling = DateTimeZoneHandling.Utc   // конвертирует в UTC при чтении
});
```

`DateTimeZoneHandling`:

| Значение | Что делает при чтении |
|----------|----------------------|
| `Local` | Конвертирует в local TZ |
| `Utc` | Конвертирует в UTC |
| `Unspecified` | Без конверсии, Kind = Unspecified |
| `RoundtripKind` | Сохраняет Kind из Z/offset в строке |

**Правило:** для public API — `Utc` (всегда работаем в UTC). Для legacy interop — `RoundtripKind`.

### 13.6. NodaTime + JSON

```bash
dotnet add package NodaTime.Serialization.SystemTextJson
```

```csharp
var options = new JsonSerializerOptions();
options.ConfigureForNodaTime(DateTimeZoneProviders.Tzdb);

JsonSerializer.Serialize(SystemClock.Instance.GetCurrentInstant(), options);
// "2024-05-15T10:30:45.1234567Z" — Instant как ISO 8601 UTC

JsonSerializer.Serialize(new LocalDate(2024, 5, 15), options);
// "2024-05-15"

JsonSerializer.Serialize(SystemClock.Instance.GetCurrentInstant().InZone(moscowZone), options);
// "2024-05-15T13:30:45.1234567+03:00 Europe/Moscow"  — ZonedDateTime включает TZ ID!
```

NodaTime для JSON более expressive — `ZonedDateTime` сохраняет именованную TZ.

### 13.7. OpenAPI / Swagger — schema documentation

```csharp
// Schema показывает формат
public class OrderResponse
{
    /// <summary>Created moment in UTC, ISO 8601</summary>
    /// <example>2024-05-15T10:30:45Z</example>
    public DateTimeOffset CreatedAt { get; set; }

    /// <summary>Order date (no time component)</summary>
    /// <example>2024-05-15</example>
    public DateOnly OrderDate { get; set; }
}
```

`DateOnly` в OpenAPI имеет format `date`, `DateTimeOffset` — `date-time`. Клиент-генераторы (NSwag, OpenAPI Generator) корректно мапят.

> [!question]- Интервью: что произойдёт, если десериализовать JSON `"2024-05-15T10:30:45"` без timezone в `DateTime`?
> `DateTime.Kind = Unspecified`. Без `Z` или offset формат неоднозначен — система не знает, UTC это или local. Поэтому при использовании этого значения (`ToLocalTime`, сравнение, сохранение в БД) могут быть тонкие баги. Best practice: API всегда возвращать `Z` или offset. Если получаешь legacy формат без TZ — обрабатывай явно через `DateTime.ParseExact(... DateTimeStyles.AssumeUniversal)` или `AssumeLocal` в зависимости от source.

---

## 14. Best Practices

### 14.1. Архитектурные правила

1. **Backend всегда UTC.** `DateTime.UtcNow` или `DateTimeOffset.UtcNow`, никогда `DateTime.Now`.
2. **Хранение в БД — UTC** (через `DateTimeOffset` / `Instant` / `datetime2 + ValueConverter`).
3. **Wire format — ISO 8601** с явным `Z` или offset (`2024-05-15T10:30:45Z`).
4. **Display — local TZ пользователя** (хранится в его профиле, IANA ID).
5. **Никогда `DateTime.Parse` без `InvariantCulture`** — на сервере с другой locale ломается.

### 14.2. Тип selection

- Точный момент во вселенной → `DateTimeOffset` (или NodaTime `Instant`).
- Только дата (день рождения, дата отчёта) → `DateOnly`.
- Только время (открытие магазина, будильник) → `TimeOnly`.
- Длительность (timeout, интервал) → `TimeSpan` (или NodaTime `Duration`).
- Календарный интервал (1 месяц, 3 года) → NodaTime `Period`.
- Сложная TZ-логика → NodaTime.

### 14.3. Не делай

- ❌ `DateTime.Now` в backend (только для UI)
- ❌ `DateTime.Parse` без `InvariantCulture`
- ❌ `dt.ToLocalTime()` на `Unspecified` без `SpecifyKind`
- ❌ Хранить в БД `DateTime.Now` без TZ awareness
- ❌ Возвращать в JSON datetime без `Z`/offset
- ❌ Использовать `DateTime` для измерения интервалов (используй `Stopwatch`)
- ❌ Хранить TZ как Windows ID (`Russian Standard Time`) — используй IANA (`Europe/Moscow`)
- ❌ Применять calendar-логику над `DateTime` — используй NodaTime для DST-aware
- ❌ `GETDATE()` для БД default (не UTC) — используй `GETUTCDATE()` или `SYSDATETIMEOFFSET()`
- ❌ Парсить дату по системной locale — invariant всегда

### 14.4. Делай

- ✅ `DateTimeOffset.UtcNow` для current moment
- ✅ `DateOnly` / `TimeOnly` для concept-aligned данных
- ✅ ISO 8601 во всех wire formats
- ✅ IANA TZ IDs (`Europe/Moscow`, `America/New_York`)
- ✅ `TimeProvider` (.NET 8+) или NodaTime `IClock` для тестируемости
- ✅ `Stopwatch` для измерения интервалов
- ✅ NodaTime для серьёзной TZ / calendar-логики
- ✅ EF Core `ValueConverter` для UTC enforcement или `DateTimeOffset` напрямую
- ✅ User TZ хранится в профиле, используется только при отображении
- ✅ Логирование с `{Time:o}` (round-trip ISO формат)

### 14.5. Тестирование

```csharp
public class OrderService
{
    private readonly TimeProvider _time;
    public OrderService(TimeProvider time) => _time = time;
    public DateTimeOffset GetCurrentTime() => _time.GetUtcNow();
}

[Fact]
public void TimeIsControlled()
{
    var fakeTime = new FakeTimeProvider(new DateTimeOffset(2024, 5, 15, 10, 0, 0, TimeSpan.Zero));
    var service = new OrderService(fakeTime);

    Assert.Equal(new DateTimeOffset(2024, 5, 15, 10, 0, 0, TimeSpan.Zero), service.GetCurrentTime());

    fakeTime.Advance(TimeSpan.FromHours(2));
    Assert.Equal(new DateTimeOffset(2024, 5, 15, 12, 0, 0, TimeSpan.Zero), service.GetCurrentTime());
}
```

Никогда не вызывай `DateTime.UtcNow` напрямую в логике — иначе тесты будут зависеть от текущего времени и непредсказуемы.

### 14.6. Performance

- `DateTime.UtcNow` дешёвый, но не точный (~15ms на Windows).
- `DateTimeOffset.UtcNow` чуть дороже, но даёт offset.
- `Stopwatch.GetTimestamp()` — для микро-измерений (наносекунды).
- `TimeProvider.GetUtcNow()` (.NET 8+) — testable wrapper над `DateTimeOffset.UtcNow`.

В hot path (миллион вызовов в секунду) — кэшируй `DateTimeOffset.UtcNow` если допустимо несколько ms задержка.

---

## 15. Decision tree

```
Что нужно?
│
├── Текущий момент во вселенной (для лога / события)
│   ├── Production code — DateTimeOffset.UtcNow или TimeProvider
│   └── Тесты — FakeTimeProvider или Mock IClock
│
├── Хранить точный момент
│   ├── DateTimeOffset (universal)
│   ├── DateTime + Kind=Utc + ValueConverter (legacy code)
│   └── NodaTime Instant (advanced TZ-aware)
│
├── День без времени
│   └── DateOnly (.NET 6+) или NodaTime LocalDate
│
├── Время дня без даты
│   └── TimeOnly (.NET 6+) или NodaTime LocalTime
│
├── Интервал
│   ├── Точное количество секунд → TimeSpan / Duration
│   └── Календарный (1 month) → NodaTime Period
│
├── Конверсия между TZ
│   ├── Простая → TimeZoneInfo.ConvertTime (IANA IDs)
│   └── Сложная (DST-aware) → NodaTime ZonedDateTime
│
├── Измерение длительности (бенчмарк, timeout)
│   ├── Stopwatch.StartNew() / .Elapsed
│   └── Stopwatch.GetTimestamp() (для hot path)
│
├── Display для пользователя
│   ├── Получить user.TimeZoneId (IANA)
│   ├── Конвертировать UTC → user TZ
│   └── ToString с CultureInfo.CurrentCulture
│
├── Wire format (JSON / API)
│   ├── ISO 8601 c Z или offset
│   └── Не naive datetime
│
└── Тестируемое время
    ├── .NET 8+ — TimeProvider
    ├── .NET 6/7 — обёртка IDateTimeProvider
    └── NodaTime — IClock + FakeClock
```

---

## 16. Cheat sheet

```csharp
// === Создание ===
DateTime.UtcNow                                    // ✅ backend
DateTimeOffset.UtcNow                              // ✅ предпочтительно
new DateOnly(2024, 5, 15)                          // только дата
new TimeOnly(9, 0)                                 // только время

// === Парсинг ===
DateTimeOffset.Parse("2024-05-15T10:30:00Z", CultureInfo.InvariantCulture, DateTimeStyles.RoundtripKind)
DateTime.ParseExact("15.05.2024", "dd.MM.yyyy", CultureInfo.InvariantCulture)
DateOnly.Parse("2024-05-15")

// === Форматирование ===
dt.ToString("o", CultureInfo.InvariantCulture)     // ISO 8601 round-trip
dt.ToString("yyyy-MM-dd HH:mm:ss", CultureInfo.InvariantCulture)

// === Конверсия TZ ===
var moscow = TimeZoneInfo.FindSystemTimeZoneById("Europe/Moscow");
var moscowTime = TimeZoneInfo.ConvertTimeFromUtc(utc, moscow);

// === DateTimeOffset ===
DateTimeOffset.UtcNow
dto.ToOffset(TimeSpan.FromHours(3))                // в другой TZ для отображения
dto.UtcDateTime                                     // как DateTime UTC

// === DateOnly / TimeOnly ===
new DateOnly(2024, 5, 15).AddDays(7)
new TimeOnly(9, 0).IsBetween(start, end)
date.ToDateTime(TimeOnly.MinValue)                  // в DateTime

// === Stopwatch ===
var sw = Stopwatch.StartNew();
DoWork();
Console.WriteLine(sw.ElapsedMilliseconds);

// === TimeProvider (.NET 8+) ===
public class Service(TimeProvider time)
{
    public void Log() => Console.WriteLine(time.GetUtcNow());
}

// === EF Core UTC enforcement ===
public class UtcConverter : ValueConverter<DateTime, DateTime>
{
    public UtcConverter() : base(
        v => v.ToUniversalTime(),
        v => DateTime.SpecifyKind(v, DateTimeKind.Utc)) { }
}

// === NodaTime ===
Instant now = SystemClock.Instance.GetCurrentInstant();
DateTimeZone moscow = DateTimeZoneProviders.Tzdb["Europe/Moscow"];
ZonedDateTime moscowTime = now.InZone(moscow);
```

| Что | Решение |
|-----|---------|
| Текущий момент в backend | `DateTimeOffset.UtcNow` |
| Текущее время для UI | `DateTimeOffset.UtcNow` → конвертировать в user TZ |
| День рождения | `DateOnly` |
| Время открытия магазина | `TimeOnly` |
| Длительность операции | `Stopwatch` |
| Хранение в БД | `DateTimeOffset` или `DateTime + ValueConverter` |
| TZ конверсия | `TimeZoneInfo.ConvertTime` с IANA ID |
| DST правильно | NodaTime `ZonedDateTime` |
| Wire format | ISO 8601 + `Z`/offset |
| Тесты времени | `TimeProvider` (.NET 8+) или NodaTime `IClock` |

---

## 17. Common Pitfalls — с механизмами

### 17.1. DateTime.Now на сервере

```csharp
public Order Create() => new Order { CreatedAt = DateTime.Now };
```

**Механизм:** `DateTime.Now` возвращает время в локальном TZ сервера. На dev-машине (Москва) и prod (UTC) поведение разное.

**Фикс:** `DateTime.UtcNow` или `DateTimeOffset.UtcNow`.

### 17.2. Kind теряется при чтении из БД

```csharp
var loaded = db.Orders.First();
var localTime = loaded.CreatedAt.ToLocalTime();   // некорректно — Kind=Unspecified
```

**Механизм:** SQL Server `datetime2` не хранит Kind. EF Core возвращает `DateTime` с `Kind = Unspecified`.

**Фикс:** `ValueConverter` с `SpecifyKind(value, Utc)` или использовать `DateTimeOffset`.

### 17.3. Парсинг без InvariantCulture

```csharp
DateTime.Parse("05/15/2024");   // на ru-RU — month/day, на en-US — month/day, формат интерпретируется по locale
```

**Механизм:** без явной culture результат зависит от системной locale. На dev-машине работает, на prod ломается.

**Фикс:** `DateTime.ParseExact("05/15/2024", "MM/dd/yyyy", CultureInfo.InvariantCulture)`.

### 17.4. ToLocalTime на Unspecified

```csharp
DateTime unspec = new DateTime(2024, 5, 15, 10, 30, 0);   // Unspecified
var local = unspec.ToLocalTime();                          // некорректно — система считает Unspecified как UTC
```

**Механизм:** при `ToLocalTime` система предполагает Kind=Utc для Unspecified. Если на самом деле это было local — двойной сдвиг.

**Фикс:** явно `SpecifyKind` перед конверсией.

### 17.5. DST и invalid time

```csharp
var localTime = new DateTime(2024, 3, 10, 2, 30, 0);   // 02:30 не существует в США 10 марта
TimeZoneInfo.ConvertTimeToUtc(localTime, ny);            // throws ArgumentException
```

**Механизм:** в день spring forward 02:00-03:00 не существует.

**Фикс:** проверить через `tz.IsInvalidTime(localTime)` и сдвинуть; или использовать NodaTime `InZoneLeniently`.

### 17.6. Windows TZ ID на Linux

```csharp
TimeZoneInfo.FindSystemTimeZoneById("Russian Standard Time");
// На Linux — TimeZoneNotFoundException
```

**Механизм:** Linux/macOS используют IANA, не Windows IDs (до .NET 6 на Windows IANA не работал).

**Фикс:** в новом коде всегда IANA (`Europe/Moscow`). С .NET 6+ работает на всех ОС.

### 17.7. JSON datetime без timezone

```json
{ "createdAt": "2024-05-15T10:30:45" }
```

**Механизм:** без `Z` или offset формат неоднозначен. Клиент и сервер могут интерпретировать по-разному.

**Фикс:** ISO 8601 с явным TZ — `"2024-05-15T10:30:45Z"` или `"2024-05-15T10:30:45+03:00"`.

### 17.8. DateTime для измерения интервалов

```csharp
var start = DateTime.UtcNow;
DoWork();
var elapsed = DateTime.UtcNow - start;   // может быть отрицательным после NTP-синхронизации
```

**Механизм:** `DateTime.UtcNow` не монотонный — может прыгнуть назад при синхронизации часов системы.

**Фикс:** `Stopwatch.StartNew()` для измерений.

### 17.9. EF Core date range без TZ

```csharp
var orders = db.Orders.Where(o => o.CreatedAt >= new DateTime(2024, 5, 1));
```

**Механизм:** `new DateTime(...)` с Kind=Unspecified, EF Core не знает, в каком TZ это интерпретировать.

**Фикс:** конвертировать в UTC явно перед запросом — `TimeZoneInfo.ConvertTimeToUtc(local, userTz)`.

### 17.10. Birthday как DateTime

```csharp
public DateTime Birthday { get; set; }   // 1990-05-15 00:00:00 — но это «время» бессмысленно
```

**Механизм:** `DateTime` тащит за собой ненужное время и Kind. При `birthday.AddHours(5)` получишь странный результат.

**Фикс:** `DateOnly Birthday { get; set; }` — явная семантика «только дата».

> [!question]- Интервью: топ-3 ловушки datetime в .NET.
> 1) `DateTime.Now` в backend — зависит от TZ сервера. На разных машинах разный результат. Всегда `DateTime.UtcNow` или `DateTimeOffset.UtcNow`. 2) Kind теряется при сохранении в БД — после чтения `Unspecified` вместо `Utc`, последующие конверсии (`ToLocalTime`) могут быть неверными. Решение — `ValueConverter` или `DateTimeOffset`. 3) DST переходы — invalid time (02:30 в день spring forward не существует) и ambiguous time (02:30 в день fall back случается дважды). Защита: backend в UTC, для wall clock операций — NodaTime с `InZoneStrictly`/`InZoneLeniently`.

---

## 18. Practice — упражнения с разбором

### 18.1. Сервис с тестируемым временем

**Задача.** Реализовать `OrderService` с методом `CreateOrder`, в котором фиксируется `CreatedAt`. Должен быть тестируемым через `TimeProvider`.

```csharp
public class Order
{
    public int Id { get; set; }
    public DateTimeOffset CreatedAt { get; set; }
    public decimal Amount { get; set; }
}

public class OrderService
{
    private readonly TimeProvider _time;

    public OrderService(TimeProvider time) => _time = time;

    public Order CreateOrder(decimal amount) =>
        new Order
        {
            CreatedAt = _time.GetUtcNow(),
            Amount = amount
        };
}

// Тест
[Fact]
public void CreateOrder_uses_provided_time()
{
    var fakeTime = new FakeTimeProvider(new DateTimeOffset(2024, 5, 15, 10, 0, 0, TimeSpan.Zero));
    var service = new OrderService(fakeTime);

    var order = service.CreateOrder(100m);

    Assert.Equal(new DateTimeOffset(2024, 5, 15, 10, 0, 0, TimeSpan.Zero), order.CreatedAt);

    // Шагнуть вперёд во времени
    fakeTime.Advance(TimeSpan.FromHours(1));
    var laterOrder = service.CreateOrder(50m);
    Assert.Equal(new DateTimeOffset(2024, 5, 15, 11, 0, 0, TimeSpan.Zero), laterOrder.CreatedAt);
}
```

**Разбор:** `TimeProvider` (.NET 8+) — стандартная абстракция времени. Production использует `TimeProvider.System`, тесты — `FakeTimeProvider` из `Microsoft.Extensions.Time.Testing`. Никогда не вызывай `DateTimeOffset.UtcNow` напрямую в логике — иначе тесты непредсказуемы.

### 18.2. Конверсия UTC → user TZ для UI

**Задача.** Метод, принимающий UTC момент и IANA TZ ID, возвращающий форматированную строку «`15 май 2024 13:30`» в TZ пользователя.

```csharp
public static class DisplayHelper
{
    public static string FormatForUser(DateTimeOffset utcMoment, string userTimeZoneId, CultureInfo? culture = null)
    {
        culture ??= CultureInfo.CurrentCulture;
        var tz = TimeZoneInfo.FindSystemTimeZoneById(userTimeZoneId);
        var localMoment = TimeZoneInfo.ConvertTime(utcMoment, tz);
        return localMoment.ToString("dd MMM yyyy HH:mm", culture);
    }
}

// Использование
var utc = new DateTimeOffset(2024, 5, 15, 10, 30, 0, TimeSpan.Zero);

DisplayHelper.FormatForUser(utc, "Europe/Moscow", new CultureInfo("ru-RU"));
// "15 май 2024 13:30"

DisplayHelper.FormatForUser(utc, "America/New_York", new CultureInfo("en-US"));
// "15 May 2024 06:30"

DisplayHelper.FormatForUser(utc, "Asia/Tokyo", new CultureInfo("ja-JP"));
// "15 5月 2024 19:30"
```

**Разбор:** базовый pattern для multi-TZ UI. `TimeZoneInfo.ConvertTime` сохраняет offset в результирующем `DateTimeOffset`. Culture передаётся явно для предсказуемости (или берётся текущий поток).

### 18.3. Date range query за месяц в timezone пользователя

**Задача.** Получить все orders за «май 2024» в TZ пользователя (а не в UTC). Использовать UTC-storage в БД.

```csharp
public async Task<List<Order>> GetOrdersForMonth(int year, int month, string userTimeZoneId, CancellationToken ct = default)
{
    var tz = TimeZoneInfo.FindSystemTimeZoneById(userTimeZoneId);

    var monthStart = new DateTime(year, month, 1, 0, 0, 0, DateTimeKind.Unspecified);
    var monthEnd = monthStart.AddMonths(1);

    var startUtc = TimeZoneInfo.ConvertTimeToUtc(monthStart, tz);
    var endUtc = TimeZoneInfo.ConvertTimeToUtc(monthEnd, tz);

    return await _db.Orders
        .Where(o => o.CreatedAt >= startUtc && o.CreatedAt < endUtc)
        .OrderBy(o => o.CreatedAt)
        .ToListAsync(ct);
}

// Использование
var orders = await service.GetOrdersForMonth(2024, 5, "Europe/Moscow");
// Получит orders с 30 апреля 21:00 UTC до 31 мая 21:00 UTC
// (потому что 1 мая 00:00 в Москве = 30 апреля 21:00 UTC)
```

**Разбор:** «май в Москве» ≠ «май в UTC». Без TZ awareness query будет давать orders на пограничные часы из других месяцев. `Unspecified` для local boundaries — потому что мы предоставим TZ через `ConvertTimeToUtc`.

### 18.4. Recurring schedule с DST

**Задача.** Реализовать `NextRunTime` для cron-job «каждый день в 9:00» с учётом TZ пользователя и DST.

```csharp
public class DailyAlarm
{
    public TimeOnly TimeOfDay { get; set; }   // например 09:00
    public string TimeZoneId { get; set; } = "UTC";
}

public static class AlarmScheduler
{
    public static DateTimeOffset NextRun(DailyAlarm alarm, TimeProvider time)
    {
        var tz = TimeZoneInfo.FindSystemTimeZoneById(alarm.TimeZoneId);
        var nowUtc = time.GetUtcNow();
        var nowInUserTz = TimeZoneInfo.ConvertTime(nowUtc, tz);

        // Сегодня в указанное время
        var todayLocal = new DateTime(
            nowInUserTz.Year, nowInUserTz.Month, nowInUserTz.Day,
            alarm.TimeOfDay.Hour, alarm.TimeOfDay.Minute, 0,
            DateTimeKind.Unspecified);

        // Защита от DST invalid time
        if (tz.IsInvalidTime(todayLocal))
            todayLocal = todayLocal.AddHours(1);

        var todayOffset = tz.GetUtcOffset(todayLocal);
        var todayInUserTz = new DateTimeOffset(todayLocal, todayOffset);

        // Если сегодня уже прошло — завтра
        if (todayInUserTz <= nowInUserTz)
        {
            var tomorrowLocal = todayLocal.AddDays(1);
            if (tz.IsInvalidTime(tomorrowLocal))
                tomorrowLocal = tomorrowLocal.AddHours(1);
            var tomorrowOffset = tz.GetUtcOffset(tomorrowLocal);
            todayInUserTz = new DateTimeOffset(tomorrowLocal, tomorrowOffset);
        }

        return todayInUserTz.ToUniversalTime();   // в UTC для scheduler
    }
}

// Использование
var alarm = new DailyAlarm { TimeOfDay = new TimeOnly(9, 0), TimeZoneId = "America/New_York" };
var next = AlarmScheduler.NextRun(alarm, TimeProvider.System);
// Например: 2024-05-16T13:00:00+00:00 (9:00 в NY = 13:00 UTC летом)
```

**Разбор:** wall clock alarm — самый коварный случай. UTC момент алармa зависит от DST: летом 9:00 NY = 13:00 UTC, зимой = 14:00 UTC. Поэтому нужно вычислять «следующий 9:00 в TZ пользователя» каждый раз заново, не хранить UTC-расписание.

### 18.5. Migration к UTC в legacy-проекте

**Задача.** Дано: `Order { DateTime CreatedAt }` колонка `datetime2`, везде сохраняется `DateTime.Now`. Мигрировать на UTC без поломки существующих данных.

```csharp
// Шаг 1 — миграция кода
public class OrderService
{
    private readonly TimeProvider _time;
    public OrderService(TimeProvider time) => _time = time;

    public Order Create(decimal amount) =>
        new Order { CreatedAt = _time.GetUtcNow().UtcDateTime, Amount = amount };   // UTC явно
}

// Шаг 2 — ValueConverter для существующих данных
public class AppDbContext : DbContext
{
    protected override void OnModelCreating(ModelBuilder mb)
    {
        var utcConverter = new ValueConverter<DateTime, DateTime>(
            v => v.Kind == DateTimeKind.Utc ? v : v.ToUniversalTime(),
            v => DateTime.SpecifyKind(v, DateTimeKind.Utc));

        foreach (var entityType in mb.Model.GetEntityTypes())
            foreach (var prop in entityType.GetProperties())
                if (prop.ClrType == typeof(DateTime) || prop.ClrType == typeof(DateTime?))
                    prop.SetValueConverter(utcConverter);
    }
}

// Шаг 3 — SQL для миграции существующих данных (один раз)
// Если БД хранила local time с предположением "Europe/Moscow" (+03:00 без DST):
// UPDATE Orders SET CreatedAt = DATEADD(HOUR, -3, CreatedAt);
// Теперь все CreatedAt в UTC.

// Шаг 4 — для display
public string DisplayCreatedAt(Order order, string userTimeZoneId)
{
    var utc = new DateTimeOffset(order.CreatedAt, TimeSpan.Zero);
    var tz = TimeZoneInfo.FindSystemTimeZoneById(userTimeZoneId);
    return TimeZoneInfo.ConvertTime(utc, tz).ToString("g", CultureInfo.CurrentCulture);
}
```

**Разбор:** ключевые шаги migration: 1) код пишет в UTC через `TimeProvider`. 2) `ValueConverter` обеспечивает Kind=Utc при чтении (защита от legacy). 3) Однократная SQL-миграция для существующих данных (если они в local). 4) Display конвертирует в user TZ. После этого все будущие данные в UTC, старые — мигрированы.

---

## 19. Что читать дальше — порядок и почему

1. **[[csharp-basics|C# Basics]]** — типы и переменные, основа для DateTime.
2. **[[strings-regex|Strings и Regex]]** — формат-строки и парсинг.
3. **NodaTime documentation** — nodatime.org — для серьёзной TZ/calendar логики.
4. **[[testing-fundamentals|Testing Fundamentals]]** — `TimeProvider`, `FakeTimeProvider` для тестов.
5. **EF Core mapping** — ValueConverter, datetimeoffset, NodaTime провайдеры.
6. **Logging & Observability** — структурированные логи с timestamps.
7. **Async deep dive** — `Task.Delay`, scheduler, recurring jobs.
8. **Distributed systems** — synchronization clocks, vector clocks, logical time.

---

## 20. См. также

- [[csharp-basics|C# Basics]] — типы данных
- [[strings-regex|Strings]] — форматирование
- [[testing-fundamentals|Testing]] — TimeProvider для тестов
- [[error-handling|Error Handling]] — exceptions при invalid TZ
- EF Core Mapping — ValueConverter, datetimeoffset
- NodaTime documentation — серьёзная TZ-логика
- Quartz.NET / Hangfire — recurring jobs с DST awareness
- ISO 8601 standard — wire format даты
- IANA Time Zone Database — `tzdata` (часовые пояса)

---

## 21. Reading list

- **Microsoft Docs — DateTime** — learn.microsoft.com/dotnet/api/system.datetime
- **Microsoft Docs — DateTimeOffset** — learn.microsoft.com/dotnet/api/system.datetimeoffset
- **Microsoft Docs — DateOnly / TimeOnly** — learn.microsoft.com/dotnet/standard/datetime/dateonly
- **Microsoft Docs — TimeZoneInfo** — learn.microsoft.com/dotnet/api/system.timezoneinfo
- **Microsoft Docs — TimeProvider** — learn.microsoft.com/dotnet/api/system.timeprovider
- **NodaTime official site** — nodatime.org
- **NodaTime user guide** — nodatime.org/3.1.x/userguide/
- **Jon Skeet — Storing UTC is not a silver bullet** — codeblog.jonskeet.uk
- **Jon Skeet — Why NodaTime?** — codeblog.jonskeet.uk
- **Falsehoods Programmers Believe About Time** — infiniteundo.com/post/25326999628/falsehoods-programmers-believe-about-time
- **More Falsehoods Programmers Believe About Time** — infiniteundo.com/post/25509354022
- **IANA Time Zone Database** — iana.org/time-zones
- **List of tz database time zones** — wikipedia.org/wiki/List_of_tz_database_time_zones
- **Why is subtracting these two times in 1927 giving a strange result?** — stackoverflow.com/questions/6841333 (классика)
- **Andrew Lock — Working with dates in EF Core** — andrewlock.net
- **Stephen Cleary — DateTimeOffset and storing time** — blog.stephencleary.com
- **Microsoft Engineering Blog — TimeProvider design** — devblogs.microsoft.com/dotnet (поиск)
