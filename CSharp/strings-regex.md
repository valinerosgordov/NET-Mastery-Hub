---
tags: [csharp, strings, regex, encoding, culture, performance]
level: Junior
date: 2026-04-30
---

# Strings и Regex — работа со строками

> **Полный гайд по строкам в C#**: от basics до performance. Закрывает: string immutability, intern pool, StringBuilder, Span\<char\>, encoding, culture-aware operations, Regex deep, source-generated regex (.NET 7+), типичные ошибки и performance traps.

---

## Что это, зачем и когда

### Почему важно

Строки — **самый частый тип** в коде. И **самый недопонимаемый**:

- Строки **immutable** — каждая операция создаёт новую (allocation!)
- Equality сравнение **зависит от culture** — "İ" и "I" в Turkish equals
- UTF-16 в .NET — **не байты**, не символы, а code units
- Regex — мощно, но дорого если каждый раз компилировать
- StringBuilder помогает, но **не всегда** нужен

**Аналогия:** Думать о строках как о массивах символов — упрощённо. Реально — это immutable UTF-16 sequences с culture, encoding и memory layout особенностями.

---

## 1. Строки — fundamentals

### Immutability

```csharp
string s = "Hello";
s += " World";  // ❌ "Hello" не изменилась — создан новый объект
// Старый "Hello" останется в памяти пока GC не соберёт
```

Каждая операция = новая аллокация:
```csharp
string result = "";
for (int i = 0; i < 10; i++)
    result += i;  // 10 allocations! Каждый "+=" = new string
```

### String literal interning

Одинаковые строковые литералы — **один объект** в memory (intern pool):

```csharp
string a = "hello";
string b = "hello";
ReferenceEquals(a, b);  // true! Один и тот же объект

// Runtime-created — нет
string c = new string("hello".ToCharArray());
ReferenceEquals(a, c);  // false

// Force intern
string d = string.Intern(c);
ReferenceEquals(a, d);  // true
```

> [!warning] Intern pool — long-lived
> Объекты в intern pool **не собираются GC**. Don't intern user input — memory leak.

### Equality vs reference

```csharp
string a = "hello";
string b = "h" + "ello";  // compile-time concatenation
string c = "h" + new string('e',1) + "llo";  // runtime

a == b;                   // true (== for string compares value)
a == c;                   // true
ReferenceEquals(a, b);    // true (compiler interned)
ReferenceEquals(a, c);    // false (runtime-created)
a.Equals(c);              // true
```

`==` для string — **value comparison** (overloaded). Для других объектов — reference.

### Char и codepoint

C# `char` — это **UTF-16 code unit**, не "символ".

```csharp
string s = "🎉";
s.Length;  // 2! Эмодзи занимает 2 char в UTF-16 (surrogate pair)
s[0];      // первый surrogate
s[1];      // второй surrogate

// Правильный count символов
s.EnumerateRunes().Count();  // 1 (Unicode codepoint)
new System.Globalization.StringInfo(s).LengthInTextElements;  // 1
```

Когда Length важна — будь осторожен с emoji, китайский, accented characters.

---

## 2. Concatenation — performance

### Способы concat

```csharp
// 1. String concatenation (+) — fine для маленьких операций
string s = "Hello" + " " + "World";

// 2. String.Concat
string s = string.Concat("Hello", " ", "World");

// 3. String interpolation
string s = $"{firstName} {lastName}";

// 4. String.Format
string s = string.Format("{0} {1}", firstName, lastName);

// 5. StringBuilder — для много операций
var sb = new StringBuilder();
sb.Append("Hello");
sb.Append(' ');
sb.Append("World");
string s = sb.ToString();

// 6. String.Join — для collections
string csv = string.Join(", ", items);
```

### Когда что использовать

```csharp
// ✅ Concat / interpolation — известное число операций
string s = $"{user.Name}, age {user.Age}";

// ✅ StringBuilder — loop с >5-10 iterations
var sb = new StringBuilder();
foreach (var item in millionItems)
    sb.AppendLine(item.ToString());

// ❌ String concatenation в loop — kill perf
string result = "";
foreach (var item in millionItems)
    result += item;  // 1M allocations!

// ✅ String.Join — лучший для collections
string result = string.Join(Environment.NewLine, items);
```

### Benchmark numbers (типичные)

```
Concat 10 strings:
  "+"          : 1 allocation, ~20 ns
  Concat       : 1 allocation, ~20 ns (compiler optimizes)
  Interpolation: 1 allocation, ~25 ns
  StringBuilder: 2-3 allocations, ~100 ns ← overhead не оправдан

Concat 1000 strings:
  "+" в loop          : 500K allocations, 100ms
  StringBuilder       : 5 allocations, 1ms ← 100x faster
  String.Join         : 1 allocation, 0.5ms ← если есть collection
```

### String interpolation (C# 10+) — efficient

```csharp
// До C# 10 — interpolation создавал object[] через string.Format
string s = $"{x} = {y}";

// C# 10+ — InterpolatedStringHandler — оптимизированный
// Если все part — strings, может быть faster чем StringBuilder!
```

### `string.Create` (.NET Core 2.1+)

Для maximum performance — direct write в Span\<char\>:

```csharp
public static string FormatCoord(double x, double y)
{
    return string.Create(20, (x, y), (span, state) =>
    {
        var written = state.x.TryFormat(span, out int charsWritten1);
        span[charsWritten1] = ',';
        state.y.TryFormat(span[(charsWritten1 + 1)..], out int charsWritten2);
    });
}
// Zero intermediate allocations!
```

---

## 3. Comparison — culture matters

### `==`, `Equals`, `Compare`

```csharp
"hello" == "Hello";                    // false (case-sensitive)
"hello".Equals("Hello");               // false
"hello".Equals("Hello", StringComparison.OrdinalIgnoreCase);  // true

string.Compare("a", "b");              // -1
string.Compare("a", "B");              // depends on culture!
string.Compare("a", "B", StringComparison.Ordinal);  // 32 (a > B in ASCII)
```

### StringComparison enum

| Value | Что |
|-------|-----|
| `Ordinal` | Бинарное сравнение (fastest, most predictable) |
| `OrdinalIgnoreCase` | Бинарное, ignore case |
| `CurrentCulture` | По текущей culture (default!) — **dangerous** |
| `CurrentCultureIgnoreCase` | + ignore case |
| `InvariantCulture` | Culture-neutral (но не binary) |
| `InvariantCultureIgnoreCase` | + ignore case |

### Turkish I problem

```csharp
// Default — CurrentCulture
"file".ToUpper();  // "FILE" в US, "FİLE" в Turkey!

// Turkish "i" → "İ" (с точкой)
// "I" (без точки) — отдельная буква

// File names, security checks — must use Ordinal
"FILE.txt".Equals("file.txt", StringComparison.OrdinalIgnoreCase);  // ✅ predictable
"FILE.txt".ToUpper();  // ❌ может выдать "FİLE" в TR locale
"FILE.txt".ToUpperInvariant();  // ✅ всегда "FILE"
```

> [!warning] Default StringComparison
> `string.Equals(a, b)` — `Ordinal`. Но `string.Compare`, sorting, `Contains` — `CurrentCulture`! **Всегда указывай явно**.

```csharp
// ❌ Culture-dependent, может удивить
list.Sort();

// ✅ Ordinal
list.Sort(StringComparer.Ordinal);

// ❌ Dangerous
"hello".StartsWith("h");

// ✅ Explicit
"hello".StartsWith("h", StringComparison.Ordinal);
```

### Best practice — выбор

```
File paths, identifiers, GUIDs:    Ordinal
Searching internal data:           Ordinal / OrdinalIgnoreCase
User-facing display:               CurrentCulture
Sorting for display:               CurrentCulture
URLs, email addresses:             Ordinal (RFC requires ASCII)
Security-sensitive:                Ordinal (no Turkish I attack)
```

---

## 4. Searching и slicing

### IndexOf / Contains / StartsWith / EndsWith

```csharp
string s = "Hello World";

s.IndexOf('o');                                              // 4
s.IndexOf('o', 5);                                           // 7
s.IndexOf("World");                                          // 6
s.LastIndexOf('o');                                          // 7

s.Contains('o');                                             // true
s.Contains("World", StringComparison.OrdinalIgnoreCase);     // true

s.StartsWith("Hello");                                       // true
s.EndsWith("World");                                         // true

// .NET 5+ — char overload (faster, no allocation)
s.Contains('o');  // быстрее чем s.Contains("o")
```

### Substring

```csharp
string s = "Hello World";
s.Substring(6);          // "World"
s.Substring(0, 5);       // "Hello"

// .NET Core 3+ — range syntax (быстрее, читабельнее)
s[6..];                  // "World"
s[..5];                  // "Hello"
s[6..^1];                // "Worl" (^1 — index from end)
s[^5..];                 // "World"
```

### Split / Join

```csharp
"a,b,c".Split(',');              // ["a", "b", "c"]
"a,b,,c".Split(',', StringSplitOptions.RemoveEmptyEntries);  // ["a","b","c"]
"a, b, c".Split(", ");           // ["a", "b", "c"]
"a, b ,c".Split(',', StringSplitOptions.TrimEntries);  // ["a","b","c"] (.NET 5+)

string.Join(", ", new[] { 1, 2, 3 });  // "1, 2, 3"
string.Join(",", items.Select(i => i.Name));  // "a,b,c"
```

### `string.Replace`

```csharp
"hello world".Replace("world", "C#");       // "hello C#"
"hello".Replace('l', 'L');                  // "heLLo"

// Multiple replacements — chain
"a-b-c".Replace("-", "_").Replace("a", "A");

// ⚠️ Каждый Replace = новая string!
// Для multiple — Regex или StringBuilder.Replace
```

### Trim / Pad

```csharp
"  hello  ".Trim();              // "hello"
"  hello  ".TrimStart();          // "hello  "
"  hello  ".TrimEnd();            // "  hello"
"---hello---".Trim('-');          // "hello"

"5".PadLeft(3, '0');              // "005"
"5".PadRight(3);                  // "5  "
```

---

## 5. StringBuilder — детально

```csharp
var sb = new StringBuilder(capacity: 1024);  // initial capacity!

sb.Append("Hello");
sb.Append(' ');
sb.Append("World");
sb.AppendLine();
sb.AppendFormat("Year: {0}", DateTime.Now.Year);
sb.AppendJoin(", ", new[] { 1, 2, 3 });

// Insert / Remove / Replace
sb.Insert(0, "Greetings: ");
sb.Remove(0, 11);
sb.Replace("World", "C#");

string result = sb.ToString();
```

### Capacity matters

```csharp
// ❌ Без capacity — много realloc'ов
var sb = new StringBuilder();
for (int i = 0; i < 10000; i++) sb.Append("x");

// ✅ Pre-allocate
var sb = new StringBuilder(10_000);
```

### Когда НЕ использовать

```csharp
// ❌ Overkill для 2-3 операций
var sb = new StringBuilder();
sb.Append("Hello ");
sb.Append(name);
return sb.ToString();

// ✅ Interpolation проще и не медленнее
return $"Hello {name}";
```

### `DefaultInterpolatedStringHandler` — under the hood

C# 10+ `$""` использует optimized handler — не аллоцирует object[].

---

## 6. Encoding — bytes vs chars

### UTF-16 в .NET

`string` хранится как **UTF-16** (2 bytes per code unit). Для I/O — нужна конвертация.

### Encoding в bytes

```csharp
string s = "Hello мир";

byte[] utf8 = Encoding.UTF8.GetBytes(s);       // 12 bytes
byte[] utf16 = Encoding.Unicode.GetBytes(s);   // 18 bytes (2 per char)
byte[] ascii = Encoding.ASCII.GetBytes(s);     // 9 bytes (мир → ???)
```

### Bytes в string

```csharp
byte[] data = ...;
string s = Encoding.UTF8.GetString(data);
string s2 = Encoding.UTF8.GetString(data, 0, length);  // часть
```

### File I/O

```csharp
// .NET defaults — UTF-8 без BOM (.NET 5+)
File.WriteAllText("out.txt", "content");
string content = File.ReadAllText("in.txt");

// Explicit encoding
File.WriteAllText("out.txt", "content", Encoding.UTF8);
File.WriteAllText("out.txt", "content", new UTF8Encoding(encoderShouldEmitUTF8Identifier: true));  // BOM
```

### Common encodings

| Encoding | Bytes per char | Использование |
|----------|---------------|----------------|
| ASCII | 1 (только 0-127) | Legacy, простые tех. файлы |
| UTF-8 | 1-4 (variable) | **Default** для веб, файлы, JSON |
| UTF-16 (Unicode) | 2-4 | .NET internal, Windows API W |
| UTF-32 | 4 | Редко |
| Windows-1252 / 1251 | 1 | Legacy Windows files |

> [!info] Always UTF-8 для new code
> JSON, HTTP, файлы, БД — UTF-8 standard. UTF-16 — internal .NET only.

---

## 7. Span\<char\> и ReadOnlySpan\<char\> — zero-allocation

### Базовое

```csharp
string s = "key=value";
ReadOnlySpan<char> span = s.AsSpan();
ReadOnlySpan<char> key = span[..3];      // "key" — без allocation!
ReadOnlySpan<char> value = span[4..];    // "value"
```

### Parsing без allocation

```csharp
// ❌ Allocations: 2 substring + parse
public int Parse(string s)
{
    var parts = s.Split('=');  // 2 substring allocations
    return int.Parse(parts[1]);
}

// ✅ Span — zero allocation
public int Parse(string s)
{
    int idx = s.IndexOf('=');
    return int.Parse(s.AsSpan(idx + 1));  // int.Parse имеет Span overload
}
```

### Comparison через Span

```csharp
ReadOnlySpan<char> path = "C:\\Users\\file.txt".AsSpan();

// Без allocation
if (path.EndsWith(".txt".AsSpan(), StringComparison.OrdinalIgnoreCase))
{
    // ...
}
```

### `MemoryExtensions` (.NET Core 2.1+)

```csharp
ReadOnlySpan<char> s = "hello world".AsSpan();
s.IndexOf('o');                  // 4
s.IndexOf("world".AsSpan());     // 6
s.Contains("world".AsSpan(), StringComparison.OrdinalIgnoreCase);  // true
s.Trim();
s.Split(' ');                    // .NET 8+ Spans of indices
```

### Pitfall: `string` создаётся при `ToString`

```csharp
ReadOnlySpan<char> span = ...;
string s = span.ToString();  // ⚠️ Allocation!
```

В hot path — оставайся в Span до самого конца.

См. [[../Runtime/span-layout|Span и Layout]] для deep dive.

---

## 8. Regex — pattern matching

### Базовый

```csharp
using System.Text.RegularExpressions;

var regex = new Regex(@"^\d{4}-\d{2}-\d{2}$");

regex.IsMatch("2024-01-15");      // true
regex.IsMatch("01-15-2024");       // false

// Match
var match = regex.Match("Date: 2024-01-15");
match.Success;          // false (^...$ обязывает start/end)

// Без anchors
var r2 = new Regex(@"\d{4}-\d{2}-\d{2}");
var m = r2.Match("Date: 2024-01-15");
m.Success;              // true
m.Value;                // "2024-01-15"
m.Index;                // 6
```

### Группы

```csharp
var regex = new Regex(@"(\d{4})-(\d{2})-(\d{2})");
var match = regex.Match("2024-01-15");

match.Groups[0].Value;  // "2024-01-15" (whole match)
match.Groups[1].Value;  // "2024"
match.Groups[2].Value;  // "01"
match.Groups[3].Value;  // "15"

// Named groups
var r2 = new Regex(@"(?<year>\d{4})-(?<month>\d{2})-(?<day>\d{2})");
var m2 = r2.Match("2024-01-15");
m2.Groups["year"].Value;   // "2024"
m2.Groups["month"].Value;  // "01"
```

### Replace

```csharp
// Простая замена
Regex.Replace("hello world", @"\w+", "X");  // "X X"

// С функцией
var result = Regex.Replace("123 abc 456", @"\d+", m =>
    (int.Parse(m.Value) * 2).ToString());
// "246 abc 912"

// Backreferences
Regex.Replace("hello world", @"(\w+) (\w+)", "$2 $1");  // "world hello"
```

### Все matches

```csharp
var matches = Regex.Matches("phone: 123-4567, fax: 987-6543", @"\d{3}-\d{4}");
foreach (Match m in matches)
    Console.WriteLine(m.Value);
// 123-4567
// 987-6543
```

### Options

```csharp
var regex = new Regex(@"hello", RegexOptions.IgnoreCase | RegexOptions.Compiled);

// Common options:
// IgnoreCase            — case-insensitive
// Multiline             — ^ и $ matches per line, не whole string
// Singleline            — . matches newline тоже
// Compiled              — JIT compile pattern (faster matching, slower init)
// IgnorePatternWhitespace — ignore whitespace and # comments в pattern
// CultureInvariant      — culture-invariant matching
```

---

## 9. Regex performance

### Компиляция дорогая

```csharp
// ❌ Создаёт новый Regex каждый call
public bool IsValidEmail(string s) =>
    new Regex(@"^[\w.-]+@[\w.-]+$").IsMatch(s);

// ✅ Один раз — static field
private static readonly Regex EmailRegex = 
    new(@"^[\w.-]+@[\w.-]+$", RegexOptions.Compiled);

public bool IsValidEmail(string s) => EmailRegex.IsMatch(s);
```

### `RegexOptions.Compiled` — JIT компилирует pattern

- **+** Matching: ~30% faster
- **−** Initialization: 100x slower (compile pattern в IL)
- **+/−** Memory: больше (JIT'd code)

**Использовать когда:** regex используется много раз. Не использовать для one-shot.

### Source-generated regex (.NET 7+) — лучший вариант

```csharp
public partial class Validator
{
    [GeneratedRegex(@"^[\w.-]+@[\w.-]+$", RegexOptions.IgnoreCase)]
    private static partial Regex EmailRegex();
    
    public bool IsValidEmail(string s) => EmailRegex().IsMatch(s);
}
```

Source generator создаёт **specialized C# код** во время compile:
- Native speed (no runtime IL emit)
- AOT-friendly
- Inspectable (можешь посмотреть generated code)
- Best performance

### Performance сравнение

```
new Regex(...) каждый раз:           ~10 µs first + 1 µs match
Static Regex без Compiled:            ~50 ns match
Static Regex с Compiled:              ~30 ns match (1ms init)
[GeneratedRegex] (.NET 7+):           ~25 ns match (no init cost)
String.Contains/IndexOf:              ~10 ns (если pattern static)
```

### Когда regex НЕ нужен

```csharp
// ❌ Overkill
Regex.IsMatch(s, @"^hello$");

// ✅ Просто
s == "hello";

// ❌ Overkill
Regex.IsMatch(s, @"hello");

// ✅
s.Contains("hello", StringComparison.Ordinal);
```

---

## 10. Common Regex patterns

```csharp
// Email (упрощённо!)
@"^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$"

// Phone (US)
@"^\(\d{3}\) \d{3}-\d{4}$"

// URL
@"https?://[^\s/$.?#].[^\s]*"

// IPv4
@"^(?:[0-9]{1,3}\.){3}[0-9]{1,3}$"

// GUID
@"^[0-9a-fA-F]{8}-[0-9a-fA-F]{4}-[0-9a-fA-F]{4}-[0-9a-fA-F]{4}-[0-9a-fA-F]{12}$"

// Whitespace
@"\s+"  // one or more

// Word boundary
@"\bword\b"

// Lookahead
@"(?=\d)"   // followed by digit (no consume)
@"(?!\d)"   // NOT followed by digit

// Lookbehind
@"(?<=\$)\d+"   // digits preceded by $
```

> [!warning] Email validation regex
> "Правильный" email regex по RFC 5322 — **3000+ символов**. Используй упрощённый или встроенные validators (`MailAddress.TryCreate`).

---

## 11. Culture-aware operations

### Numbers

```csharp
double d = 1234.56;

d.ToString();                    // "1234.56" в US, "1234,56" в DE!
d.ToString(CultureInfo.InvariantCulture);  // "1234.56" всегда
d.ToString("F2", CultureInfo.GetCultureInfo("de-DE"));  // "1234,56"

// Parsing
double.Parse("1234.56");                            // OK in US
double.Parse("1234.56", CultureInfo.InvariantCulture);  // ✅ всегда
double.Parse("1.234,56", CultureInfo.GetCultureInfo("de-DE"));  // OK
```

### Dates

См. [[datetime-timezones|DateTime и Timezones]].

### Pitfall — JSON serialization

```csharp
// ⚠️ Default JSON может использовать current culture
var json = JsonSerializer.Serialize(new { Price = 1.5 });
// В US: {"Price":1.5}
// В DE: {"Price":1,5}  ← Invalid JSON!

// ✅ JSON всегда invariant — System.Text.Json делает это автоматически
// Newtonsoft.Json — по дефолту тоже Invariant
```

### Best practice

```
User-facing:           CurrentCulture
Internal data / API:   InvariantCulture
File names:            Ordinal (никакой culture)
```

---

## 12. Common Pitfalls

### 1. String concat в loop

```csharp
// ❌ 1M allocations
string result = "";
for (int i = 0; i < 1_000_000; i++)
    result += i;

// ✅ StringBuilder
var sb = new StringBuilder();
for (int i = 0; i < 1_000_000; i++)
    sb.Append(i);
string result = sb.ToString();
```

### 2. ToUpper/ToLower вместо OrdinalIgnoreCase

```csharp
// ❌ 2 allocations + culture-dependent
if (s1.ToLower() == s2.ToLower()) { ... }

// ✅ Zero allocation, predictable
if (s1.Equals(s2, StringComparison.OrdinalIgnoreCase)) { ... }
```

### 3. Default StringComparison

```csharp
// ❌ Default — CurrentCulture, может surprise
if (s.Contains("hello")) { ... }

// ✅ Ordinal
if (s.Contains("hello", StringComparison.Ordinal)) { ... }

// CA1310 — встроенный analyzer warning
```

### 4. New Regex каждый call

```csharp
// ❌ Compile pattern каждый call
public bool Validate(string s) => 
    new Regex(@"...").IsMatch(s);

// ✅ Static + Compiled, или GeneratedRegex
private static readonly Regex Re = new(@"...", RegexOptions.Compiled);
public bool Validate(string s) => Re.IsMatch(s);
```

### 5. ReDoS (Regex Denial of Service)

Некоторые patterns на адверсариальном вводе → exponential time.

```csharp
// ❌ Vulnerable — nested quantifiers
var regex = new Regex(@"^(a+)+$");
regex.IsMatch("aaaaaaaaaaaaaaaaaaaaaaaaaaaa!");  // часами!

// ✅ Лечение:
// 1. Timeout
var safe = new Regex(@"^(a+)+$", RegexOptions.None, TimeSpan.FromSeconds(1));

// 2. Atomic groups
var safe2 = new Regex(@"^(?>a+)$");

// 3. Source-generated — analyzer warns
```

### 6. Encoding mismatch

```csharp
// ❌ File saved as UTF-8, read as ANSI
string s = File.ReadAllText("file.txt");  // default UTF-8 на .NET 5+, ANSI на Windows older

// ✅ Explicit
string s = File.ReadAllText("file.txt", Encoding.UTF8);
```

### 7. Surrogate pairs

```csharp
string s = "Hello 🎉";
s[s.Length - 1];  // ⚠️ Один surrogate, не emoji!
s[s.Length - 2];  // другой surrogate

// ✅ Через Rune (Unicode codepoint)
foreach (Rune r in s.EnumerateRunes())
{
    Console.WriteLine(r);  // правильно итерирует по logical characters
}
```

### 8. String.Format vs interpolation

```csharp
// ❌ Old way
string s = string.Format("Hello {0}", name);

// ✅ Modern
string s = $"Hello {name}";
```

`$""` evaluates как `string.Format` если используется как `string`. Но при использовании с `FormattableString` или `IFormattable` — culture можно контролировать.

### 9. Trim() с string aргументом до .NET 5

```csharp
// ❌ До .NET 5 — only trim chars from set
"abcXYZ".Trim("XYZ");  // ⚠️ trim each X, Y, Z — не "XYZ"

// .NET 5+ — но всё равно char-based
"abcXYZ".TrimEnd("XYZ".ToCharArray());  // "abc"

// ✅ Для substring trim
if (s.EndsWith("XYZ")) s = s[..^3];
```

### 10. `char.IsDigit` vs ASCII

```csharp
// IsDigit включает все Unicode digits!
char.IsDigit('5');    // true
char.IsDigit('५');    // true (Devanagari 5)
char.IsDigit('०');    // true

// ✅ Только ASCII
c >= '0' && c <= '9'
// или (.NET 7+)
char.IsAsciiDigit('5');
```

---

## 13. Modern features (.NET 7+)

### `IsAscii*` checks

```csharp
char.IsAsciiDigit('5');       // .NET 7+
char.IsAsciiLetter('a');
char.IsAsciiLetterOrDigit('a');
char.IsAsciiHexDigit('F');
char.IsAsciiLetterUpper('A');
char.IsAsciiLetterLower('a');
```

### `ReplaceLineEndings()`

```csharp
// .NET 6+ — нормализация \r\n → \n
text.ReplaceLineEndings();         // \n
text.ReplaceLineEndings("\r\n");   // CRLF
```

### Raw string literals (C# 11)

```csharp
var json = """
{
    "name": "John",
    "city": "New York"
}
""";

var pattern = """\d+""";  // raw — не нужно escape

var sql = $"""
    SELECT * FROM users
    WHERE id = {userId}
    """;  // interpolation в raw
```

### `string.Create` overloads (.NET 6+)

```csharp
// Span-based formatting
string s = string.Create(CultureInfo.InvariantCulture, $"value: {x:F2}");
// More efficient than $"value: {x:F2}"
```

---

## 14. Best Practices

### Performance

- **`StringComparison.Ordinal`** для большинства internal сравнений
- **Static Regex** или `[GeneratedRegex]` (.NET 7+)
- **StringBuilder** для loop concatenation
- **Span\<char\>** для parsing hot paths
- **`string.Create`** для maximum performance string building
- **Initial capacity** в StringBuilder
- **Char overloads** где возможно (`Contains('a')` vs `Contains("a")`)

### Correctness

- **Always specify StringComparison** (avoid default!)
- **CultureInfo.InvariantCulture** для serialization, file names, IDs
- **Regex timeout** для user input
- **Encoding.UTF8** explicit для I/O
- **Rune** или `StringInfo` для logical character iteration (emoji)

### Style

- **Interpolation `$""`** вместо `string.Format`
- **Raw strings `"""`** для JSON, SQL, regex (C# 11+)
- **`nameof()`** вместо string literals для member names
- **Range syntax `s[..5]`** вместо `Substring(0, 5)` (.NET Core 3+)

### Modern code

- **`[GeneratedRegex]`** (.NET 7+) для regex
- **`string.Create`** для performance
- **Span/Memory** в hot paths
- **`IsAscii*`** методы для validation (.NET 7+)
- **`ReplaceLineEndings`** для нормализации (.NET 6+)

---

## 15. Cheatsheet

| Задача | Best practice |
|--------|---------------|
| Сравнить case-insensitive | `s1.Equals(s2, StringComparison.OrdinalIgnoreCase)` |
| Сравнить начало | `s.StartsWith(prefix, StringComparison.Ordinal)` |
| Concat в loop | `StringBuilder` |
| Concat known list | `string.Join(separator, items)` |
| Parse int from substring | `int.Parse(s.AsSpan(start, length))` |
| Substring | `s[start..end]` (range syntax) |
| Replace multiple | `Regex` или sequential `.Replace()` |
| Format number invariant | `n.ToString(CultureInfo.InvariantCulture)` |
| File I/O | `File.WriteAllText(path, text, Encoding.UTF8)` |
| Validate email | Built-in `MailAddress.TryCreate` или [GeneratedRegex] |
| ToUpper/ToLower | `ToUpperInvariant()` / `ToLowerInvariant()` |
| Iterate emoji-safe | `s.EnumerateRunes()` |
| Check ASCII | `char.IsAsciiDigit(c)` (.NET 7+) |

---

## См. также

- [[modern-features|Modern C# Features]] — raw strings, interpolation
- [[../Runtime/span-layout|Span и Layout]] — Span\<char\> deep
- [[collections-linq|Collections и LINQ]] — string как IEnumerable\<char\>
- [[datetime-timezones|DateTime и Timezones]] — date formatting
- [[functional-csharp|Functional C#]] — string processing pipelines
- [[../Runtime/gc-memory|GC и память]] — string allocations

## Reading list

- **Microsoft Docs — Strings** — learn.microsoft.com/dotnet/standard/base-types/best-practices-strings
- **Strings Best Practices** — learn.microsoft.com (детальный гайд)
- **Regex syntax reference** — learn.microsoft.com/dotnet/standard/base-types/regular-expression-language-quick-reference
- **Regex101** — regex101.com (regex tester)
- **Regex Source Generator** — devblogs.microsoft.com/dotnet/regular-expression-improvements-in-dotnet-7
- **Stephen Toub — string performance posts** — devblogs.microsoft.com/dotnet
- **Adam Sitnik — strings benchmarks** — adamsitnik.com
- **Effective C#** — Bill Wagner (главы про string)
