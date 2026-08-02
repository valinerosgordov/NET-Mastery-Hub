---
tags: [csharp, strings, regex, junior, encoding, stringbuilder, span, source-generators]
level: Junior
date: 2026-08-02
---

# Strings и Regex — строки и регулярные выражения

> **Самый частый тип в любом коде — и самый недооценённый.** Immutability, encoding (UTF-8/UTF-16), comparison ordinals vs culture, StringBuilder, `Span<char>` для perf, Regex с source generators (.NET 7+). Закрывает пробел: «вижу `string`, не понимаю, почему `+=` в loop опасно и зачем `StringComparison`».

---

## 0. Как читать этот файл

Если ты впервые работаешь со strings в C# — читай разделы 1→4 подряд: получишь рабочую модель и поймёшь, **почему string immutable**. Если уже пишешь, но непонятно про comparison/culture — раздел 4. Если интересны perf оптимизации — раздел 9 (`Span<char>`), 10 (StringBuilder). Regex с нуля — раздел 11→13. Если строишь production систему — раздел 17 (best practices), 20 (pitfalls).

Все примеры самостоятельные. `// expected: ...` показывает ожидаемый вывод. Cross-language якоря (`> [!info]-`) свёрнуты — раскрывай если переходишь из Python / Java / JavaScript / Rust / Go. Interview-вопросы (`> [!question]-`) встроены.

---

## 1. Что это, зачем и когда

### 1.1. Что такое string в C#

`string` (System.String) — это **immutable reference type**, представляющий последовательность UTF-16 code units (`char` = 16 бит):

```csharp
string s = "hello";
s.Length;       // 5 — количество char (UTF-16 code units)
s[0];           // 'h' — char
s[4];           // 'o'
```

Под капотом — массив `char[]` + length, упакованный в один heap-объект для эффективности.

### 1.2. Главные свойства

1. **Immutable** — после создания строка не меняется. Все методы (`Replace`, `ToUpper`, `Trim`) возвращают **новую** строку.
2. **Reference type** — но с value semantics для equality (`==` сравнивает содержимое, не ссылки).
3. **UTF-16 internally** — один `char` = 16 бит. Большинство Unicode помещается в один char, но эмодзи и редкие иероглифы — surrogate pairs (2 char'а на символ).
4. **Indexable** — `s[0]` доступ по индексу (но это `char`, а не «символ»! — surrogate pairs).
5. **Hashable** — `Equals` + `GetHashCode` работают по содержимому.

### 1.3. Зачем immutability

```csharp
string s1 = "hello";
string s2 = s1;
s1 = s1.ToUpper();   // НЕ мутирует s2

Console.WriteLine(s1);   // HELLO
Console.WriteLine(s2);   // hello — без изменений
```

Преимущества:
- **Thread-safe by default** — нельзя случайно измените из другого потока.
- **Hash code stable** — можно использовать как Dictionary key.
- **Sharing safe** — можно передавать в методы без defensive copy.
- **String interning** — одинаковые literals shared в memory (раздел 16).

Недостаток:
- **Builder pattern для модификаций** — каждое изменение = новая аллокация. Решение: `StringBuilder` для частых мутаций.

### 1.4. Главное правило

```
Используй string для:
  - Хранения текста (имена, ID, messages)
  - Сравнения, поиска
  - Read-only API contracts

Используй StringBuilder для:
  - Построения строк в loop (concat в loop = O(n²))
  - Условных операций (нужно несколько append'ов)
  - Большое количество мутаций в hot path

Используй Span<char>/ReadOnlySpan<char> для:
  - Парсинг без allocations
  - Substring views без копирования
  - Performance-critical hot path
```

### 1.5. Эволюция: .NET 1.0 → .NET 10

| Версия | Год | Что появилось |
|--------|-----|---------------|
| **.NET 1.0** | 2002 | `string`, `StringBuilder`, `String.Format` |
| **.NET 2.0** | 2005 | `String.Compare` с culture |
| **.NET 4.0** | 2010 | `String.Join` overloads |
| **C# 6.0** | 2015 | String interpolation `$"..."` |
| **.NET Core 2.1** | 2018 | `Span<T>`, `ReadOnlySpan<char>` |
| **C# 8.0** | 2019 | Nullable reference types — `string?` |
| **C# 11** | 2022 | Raw string literals `"""..."""`, UTF-8 string literals `"..."u8` |
| **.NET 7** | 2022 | `[GeneratedRegex]` source generator |
| **.NET 8-9** | 2023-2024 | `SearchValues<char>` / `SearchValues<string>` (см. 9.9), `CompositeFormat` |
| **C# 12-13** | 2023-2024 | `params ReadOnlySpan<char>`, performance improvements |

### 1.6. string vs char vs StringBuilder

| | string | char | StringBuilder |
|---|--------|------|---------------|
| Mutable | ❌ | ❌ | ✅ |
| Memory | heap (с char[] inside) | 2 байта (UTF-16 unit) | heap, growable buffer |
| Allocation per modification | новая string | — | grows in place (with capacity) |
| Use for | хранение, comparison | individual character | building text |

> [!info]- Если ты знаешь Python / Java / JavaScript / Rust / Go
> **Python:** `str` immutable, encoded as UTF-8 internally (CPython 3.3+). Очень близко к C#, но iteration по «символам» (Unicode code points), а не code units. C# `char` = UTF-16 code unit; Python `str[i]` = Unicode character.
>
> **Java:** `String` immutable, UTF-16 internally (как C#). `StringBuilder`/`StringBuffer` для мутации (Buffer thread-safe). API очень близкий: `String.equals()`, `length()`, `charAt(i)`, `substring(i, j)`. C# `+=` для String — то же anti-pattern.
>
> **JavaScript:** `String` primitive (immutable), без явной encoding модели (UCS-2 / UTF-16). `String.prototype.length` — code units. Template literals `` `...${var}...` `` ↔ C# `$"...{var}..."`. Без Builder — `Array.join('')` для perf.
>
> **Rust:** `String` (heap, mutable, owned, UTF-8 internally) vs `&str` (read-only slice). Это ближе к C# `string` vs `ReadOnlySpan<char>`. Один `char` в Rust = Unicode scalar value (4 bytes), не UTF-16 unit.
>
> **Go:** `string` immutable, UTF-8 internally. `[]byte` для мутации, `strings.Builder` для perf concat. Iteration по байтам или по rune (Unicode code point).

> [!question]- Интервью: почему string в C# immutable?
> Дизайн-решение Microsoft, скопированное из Java. Преимущества: 1) **Thread-safe** by default — нельзя случайно изменить из другого потока, нет race conditions. 2) **String interning** — одинаковые literals (`"hello"`) shared в одном memory, экономия. 3) **Hash code stable** — можно использовать как Dictionary key, hash не меняется. 4) **Reference passing safe** — можно передавать в методы без defensive copy. 5) **Equality semantics простая** — equals по содержимому. Недостаток: каждая модификация = новая string allocation. Решение для частых мутаций — StringBuilder.

---

## 2. Создание строк

### 2.1. String literal

```csharp
string s = "hello";
string empty = "";
string nullStr = null;          // только если NRT отключены или string?
string explicit = string.Empty;  // эквивалент ""
```

`string.Empty` ≡ `""`. Раньше предпочитали `string.Empty` (избегало allocation), сейчас — стилистический выбор. Литералы интернированы (раздел 16).

### 2.2. Verbatim string `@"..."`

Без escape sequences:

```csharp
string path = @"C:\Users\Alice\Documents";   // вместо "C:\\Users\\Alice\\Documents"
string regex = @"^\d+$";                       // вместо "^\\d+$"
string multiline = @"line 1
line 2
line 3";
```

`@` префикс — никаких escape. Backslash, quote, newline — буквально.

Чтобы вставить кавычку — двойная:

```csharp
string s = @"He said ""hello""";   // He said "hello"
```

### 2.3. Raw string literals `"""..."""` (C# 11+)

Для multiline без escape:

```csharp
string json = """
    {
        "name": "Alice",
        "age": 30
    }
    """;

string sql = """
    SELECT *
    FROM Users
    WHERE Id = @id
    """;
```

**Правила:**
- Минимум 3 кавычки (`"""`). Можно больше для содержимого с тремя кавычками.
- Indentation определяется closing `"""` — leading whitespace отрезается до его уровня.
- Никаких escapes — backslash, кавычка, newline буквально.

Это **лучший** способ для multi-line и embedded JSON / SQL / HTML.

### 2.4. Interpolated string `$"..."` (C# 6+)

```csharp
string name = "Alice";
int age = 30;
string greeting = $"Hello, {name}! You are {age} years old.";
// "Hello, Alice! You are 30 years old."

// С форматированием
decimal price = 1234.5m;
string s = $"Price: {price:F2}";       // "Price: 1234.50"
string d = $"Date: {DateTime.Now:yyyy-MM-dd}";   // ISO date

// С выражениями
int a = 5, b = 7;
string sum = $"{a} + {b} = {a + b}";   // "5 + 7 = 12"
```

Под капотом C# generates `String.Format` (или `DefaultInterpolatedStringHandler` в .NET 6+ для perf).

### 2.5. `$@"..."` или `@$"..."` — interpolation + verbatim

```csharp
string name = "Alice";
string path = $@"C:\Users\{name}\Documents";   // verbatim + interpolation
```

Порядок `$@` или `@$` не важен. C# 8+ — `@$` тоже работает.

### 2.6. Raw interpolated `$"""..."""` (C# 11+)

```csharp
string user = "Alice";
string json = $$"""
    {
        "user": "{{user}}",
        "literal": "{not interpolated}"
    }
    """;
```

`$$` — два знака `$` означают, что для interpolation нужно `{{...}}` (двойные скобки). Полезно для JSON / curly-brace-heavy content.

### 2.7. UTF-8 string literals `"..."u8` (C# 11+)

```csharp
ReadOnlySpan<byte> utf8 = "hello"u8;   // ReadOnlySpan<byte>, без runtime encoding
ReadOnlySpan<byte> emoji = "😀"u8;     // 4 bytes UTF-8
```

`u8` суффикс создаёт `ReadOnlySpan<byte>` непосредственно из literal — без runtime encoding. Полезно для networking, file I/O, perf-critical парсинга.

### 2.8. char to string

```csharp
char c = 'A';
string s1 = c.ToString();        // "A"
string s2 = new string(c, 5);    // "AAAAA"

// Из массива char
char[] chars = ['h', 'e', 'l', 'l', 'o'];
string s3 = new string(chars);   // "hello"
```

> [!question]- Интервью: чем `@"..."` отличается от `"""..."""`?
> `@"..."` (verbatim, .NET 1.0) — отключает escape sequences, позволяет multiline, для quote — двойная (`""`). `"""..."""` (raw string, C# 11+) — тоже без escape, но более выразительный для multi-line: indentation отрезается по closing `"""`, можно вкладывать любые quotes без удвоения. Raw strings лучше для embedded JSON/SQL/HTML, verbatim — historical compatibility и paths/regex. С C# 11+ raw strings — рекомендуемый подход.

---

## 3. Концатенация

### 3.1. `+` оператор

```csharp
string a = "hello";
string b = "world";
string result = a + " " + b;   // "hello world"
```

Каждое `+` создаёт **новую** string. Для двух строк OK. Для 10+ — медленно.

### 3.2. `String.Concat`

```csharp
string s = string.Concat("hello", " ", "world");
string s2 = string.Concat("a", "b", "c", "d");
```

Эквивалентно `+`, но может быть оптимизировано в одну allocation для известного количества частей.

### 3.3. `String.Join` — соединение коллекции

```csharp
string[] parts = ["a", "b", "c"];
string s = string.Join(", ", parts);   // "a, b, c"

// С IEnumerable<T>
var users = new[] { "Alice", "Bob", "Charlie" };
string list = string.Join(" | ", users);   // "Alice | Bob | Charlie"

// С разделителем-char
string s2 = string.Join('/', parts);   // "a/b/c"
```

`Join` оптимизирован — single allocation. Лучший выбор для коллекций.

### 3.4. **Anti-pattern**: `+=` в loop

```csharp
// ❌ O(n²) — каждая итерация копирует всё
string result = "";
for (int i = 0; i < 10000; i++)
    result += i.ToString();   // каждый += создаёт новую строку!

// На 10000 итераций — миллионы аллокаций
```

**Механизм:** `+=` создаёт новую строку с суммой длин предыдущей и новой части. Каждый шаг копирует growing string. Quadratic complexity.

### 3.5. `StringBuilder` — для loops

```csharp
// ✅ O(n) — buffer growable
var sb = new StringBuilder();
for (int i = 0; i < 10000; i++)
    sb.Append(i);

string result = sb.ToString();
```

`StringBuilder` имеет growable buffer — append амортизированно O(1). Финальный `ToString()` — одна allocation.

### 3.6. Когда `+` приемлем

```csharp
// ✅ Концат в одной expression — компилятор оптимизирует
string greeting = "Hello, " + name + "!";   // single string.Concat call

// ✅ Известное малое число
string s = a + b + c;   // OK, 3 части

// ❌ В loop / условный
for (...) result += ...;
```

C# compiler оптимизирует chain `+` в одной expression в один `Concat` call. В loop оптимизация не работает — каждая итерация отдельный statement.

### 3.7. String interpolation для concat

```csharp
// Альтернатива concat
string s = $"{a} {b} {c}";   // эквивалент string.Format

// vs concat
string s2 = a + " " + b + " " + c;
```

Interpolation в .NET 6+ использует `DefaultInterpolatedStringHandler` — близко к Concat по perf. Outside hot path — выбирай по readability.

### 3.8. Performance compare

```
| Method                           | 10000 iterations | Allocated |
|--------------------------------- |-----------------:|----------:|
| StringBuilder                    |          0.5 ms |      80 KB |
| string.Join (с pre-built array)  |          1.0 ms |      40 KB |
| string.Concat (chain)             |        100.0 ms |    400 MB |  ← O(n²)
| += в loop                         |        100.0 ms |    400 MB |  ← O(n²)
```

В loop — всегда StringBuilder. Outside loop — взаимозаменяемо.

> [!question]- Интервью: почему `+=` в loop медленный?
> `string` immutable. Каждое `result += part` создаёт **новую** string длины `result.Length + part.Length`, копируя весь предыдущий контент. На N итераций — sum of lengths растёт квадратично, total work O(N²). Для 10000 итераций — миллионы аллокаций и копирований. Решение: `StringBuilder` имеет growable buffer (амортизированно O(1) на append), final `ToString()` — одна allocation. Total O(N).

---

## 4. Comparison и culture

### 4.1. == vs Equals

```csharp
string a = "hello";
string b = "hello";

a == b;             // true — ВСЕГДА by content для string
a.Equals(b);        // true — by content
ReferenceEquals(a, b);  // часто true (interning), но НЕ полагайся
```

`==` для `string` перегружен — сравнивает по content. Это особенность string vs обычный reference type, где `==` по reference.

### 4.2. **Главная ловушка**: `==` использует Ordinal

```csharp
"i".Equals("I");                           // false
"i".Equals("I", StringComparison.OrdinalIgnoreCase);   // true
"i".ToUpper() == "I";                      // true (после ToUpper)

// Турецкая локаль — i / İ / I / ı
"İSTANBUL".ToLower();                      // istanbul (invariant) или İstanbul (Turkish)
```

`==` сравнивает по **byte**, не по семантике. `İ` (Turkish с точкой) и `I` — разные code points. ToLower зависит от culture (Turkish: I → ı, не i).

### 4.3. StringComparison enum

```csharp
public enum StringComparison
{
    CurrentCulture,
    CurrentCultureIgnoreCase,
    InvariantCulture,
    InvariantCultureIgnoreCase,
    Ordinal,                  // быстрее всех, byte-by-byte
    OrdinalIgnoreCase
}
```

| Значение | Когда использовать |
|----------|--------------------|
| **`Ordinal`** | Internal IDs, file paths, hashes, machine-to-machine comparisons. **Default для backend**. |
| **`OrdinalIgnoreCase`** | Same as Ordinal, но case-insensitive (a == A). HTTP headers, file system on Windows. |
| **`InvariantCulture`** | Cross-culture stable sorting (например, для display where cultural rules apply but stable). Редко. |
| **`InvariantCultureIgnoreCase`** | Same with case-insensitive. |
| **`CurrentCulture`** | Display to user — sorting names в UI с user locale. |
| **`CurrentCultureIgnoreCase`** | Same with case-insensitive. |

### 4.4. Best practice: всегда явно указывай

```csharp
// ❌ Без Comparison — default Ordinal, но непонятно намерение
if (s1 == s2) { }
if (s1.StartsWith("foo")) { }

// ✅ Явно
if (string.Equals(s1, s2, StringComparison.Ordinal)) { }
if (s1.StartsWith("foo", StringComparison.OrdinalIgnoreCase)) { }
```

Это convention Microsoft + StyleCop правило (CA1307, CA1310). Code analysis обычно подсказывает.

### 4.5. Comparing ignore case

```csharp
// ❌ Создаёт две новые строки
if (s1.ToLower() == s2.ToLower()) { }

// ✅ Не аллоцирует
if (s1.Equals(s2, StringComparison.OrdinalIgnoreCase)) { }
if (string.Equals(s1, s2, StringComparison.OrdinalIgnoreCase)) { }
```

`ToLower()` создаёт новую строку — дорогое allocation. `Equals` с `OrdinalIgnoreCase` — сравнивает character-by-character, без allocations.

### 4.6. `Compare` для sorting

```csharp
int cmp = string.Compare("apple", "banana", StringComparison.Ordinal);
// < 0 — "apple" сортируется раньше
// = 0 — одинаковые
// > 0 — "apple" позже

// Sort массива
var fruits = new[] { "banana", "apple", "cherry" };
Array.Sort(fruits, StringComparer.Ordinal);
// ["apple", "banana", "cherry"]
```

`StringComparer.Ordinal` — для production. `StringComparer.CurrentCulture` — для UI sorting.

### 4.7. `StringComparer` для Dictionary / HashSet

```csharp
// ❌ Default Ordinal — case-sensitive
var dict = new Dictionary<string, int>();
dict["Alice"] = 1;
dict["alice"];   // KeyNotFoundException — не найден

// ✅ Case-insensitive
var dict2 = new Dictionary<string, int>(StringComparer.OrdinalIgnoreCase);
dict2["Alice"] = 1;
dict2["alice"];   // 1 — found!
```

Передавай `StringComparer` в конструктор collection. Hash function использует тот же сравнитель.

### 4.8. CurrentCulture surprises

```csharp
// На US machine
"co-op".CompareTo("coop");   // < 0 (co-op раньше coop)

// На German machine
"straße".CompareTo("strasse");   // равны (ß == ss в German)

// На Turkish machine
"INSTANBUL" with InvariantCulture.ToLower();   // "instanbul"
"INSTANBUL" with Turkish.ToLower();             // "ınstanbul" (без dot!)
```

`CurrentCulture` — нестабильно между машинами. Никогда не используй для protocol / persistence — всегда `Ordinal` или `InvariantCulture`.

### 4.9. CA1307 / CA1310 analyzers

Roslyn rules автоматически предупреждают:
- **CA1307**: `string.Equals(other)` без StringComparison — warning.
- **CA1310**: `string.Compare(s1, s2)` без StringComparison — warning.

Включается через `.editorconfig`:
```ini
dotnet_diagnostic.CA1307.severity = warning
dotnet_diagnostic.CA1310.severity = warning
```

> [!question]- Интервью: какой `StringComparison` использовать для проверки equal в backend?
> **`Ordinal`** или **`OrdinalIgnoreCase`** — для internal/protocol/database/file-path comparisons. Это **byte-by-byte** сравнение, fastest, deterministic между машинами и locales. `CurrentCulture` использовать только для display sorting в UI с user locale. `InvariantCulture` — для stable cross-locale sorting (редко). Default `==` для string использует Ordinal, но Microsoft style guide требует **явно указывать** StringComparison через CA1307/CA1310 analyzers — для clarity намерения.

---

## 5. String search и manipulation

### 5.1. IndexOf / LastIndexOf

```csharp
string s = "Hello, World!";

s.IndexOf('W');                              // 7
s.IndexOf("World");                          // 7
s.IndexOf("world", StringComparison.OrdinalIgnoreCase);   // 7
s.IndexOf('Z');                              // -1 (not found)
s.IndexOf('o', startIndex: 5);               // 8 (после позиции 5)
s.LastIndexOf('o');                          // 8

// Все occurrences
int idx = -1;
while ((idx = s.IndexOf('l', idx + 1)) != -1)
    Console.WriteLine(idx);   // 2, 3, 10
```

### 5.2. Contains / StartsWith / EndsWith

```csharp
string s = "Hello, World!";

s.Contains("World");                                   // true
s.Contains('!');                                        // true
s.Contains("WORLD", StringComparison.OrdinalIgnoreCase); // true

s.StartsWith("Hello", StringComparison.Ordinal);      // true
s.EndsWith("!", StringComparison.Ordinal);             // true
```

С .NET 5+ `Contains(char)` доступен — fast (без string allocation).

### 5.3. Substring

```csharp
string s = "Hello, World!";

s.Substring(7);          // "World!" — с index 7 до конца
s.Substring(0, 5);       // "Hello" — с index 0, длина 5
s.Substring(7, 5);       // "World"

// .NET 8+: Range
s[7..];                   // "World!"
s[..5];                   // "Hello"
s[7..12];                 // "World"
```

`s[7..]` — range syntax, эквивалентно `Substring(7)`, но более readable.

### 5.4. Replace

```csharp
string s = "Hello, World!";

s.Replace("World", "C#");                // "Hello, C#!"
s.Replace('l', 'L');                      // "HeLLo, WorLd!"
s.Replace("hello", "Hi", StringComparison.OrdinalIgnoreCase);   // "Hi, World!"
```

Возвращает **новую** строку — original не меняется.

### 5.5. Split

```csharp
string csv = "alice,bob,charlie";
string[] parts = csv.Split(',');   // ["alice", "bob", "charlie"]

// Несколько разделителей
string s = "alice;bob,charlie|david";
string[] all = s.Split(';', ',', '|');   // ["alice", "bob", "charlie", "david"]

// Пустые строки
string s2 = "a,,b,,c";
s2.Split(',');                            // ["a", "", "b", "", "c"]
s2.Split(',', StringSplitOptions.RemoveEmptyEntries);   // ["a", "b", "c"]

// С trim
"a, b, c".Split(',', StringSplitOptions.TrimEntries);   // ["a", "b", "c"]
"  a, b, c  ".Split(',', StringSplitOptions.TrimEntries | StringSplitOptions.RemoveEmptyEntries);

// Limit count
"a,b,c,d".Split(',', count: 2);   // ["a", "b,c,d"]
```

### 5.6. Trim / TrimStart / TrimEnd

```csharp
"  hello  ".Trim();                  // "hello"
"  hello  ".TrimStart();              // "hello  "
"  hello  ".TrimEnd();                // "  hello"

// Specific chars
"--hello--".Trim('-');                // "hello"
"  hello\n\t".Trim();                  // "hello" — все whitespace
```

### 5.7. PadLeft / PadRight

```csharp
"42".PadLeft(5);              // "   42"
"42".PadLeft(5, '0');         // "00042"
"42".PadRight(5, '*');        // "42***"
```

Для форматирования вывода или fixed-width текстов.

### 5.8. ToUpper / ToLower

```csharp
"Hello".ToUpper();                          // "HELLO" (current culture)
"Hello".ToUpperInvariant();                 // "HELLO" (invariant)
"İstanbul".ToLowerInvariant();               // "i̇stanbul" (invariant — preserves dot)
"İstanbul".ToLower(new CultureInfo("tr-TR"));  // "istanbul" (Turkish)
```

**Правило:** для backend / protocol — `ToUpperInvariant` / `ToLowerInvariant`. Никогда не вызывай `ToUpper()` без culture, если результат используется в comparison или persistence.

### 5.9. Insert / Remove

```csharp
"Hello, World!".Insert(7, "C# ");        // "Hello, C# World!"
"Hello, World!".Remove(5);                 // "Hello" — с index 5 до конца
"Hello, World!".Remove(5, 7);              // "Hello!" — 7 chars с index 5
```

Возвращают новую строку.

### 5.10. IsNullOrEmpty / IsNullOrWhiteSpace

```csharp
string.IsNullOrEmpty(null);        // true
string.IsNullOrEmpty("");          // true
string.IsNullOrEmpty("  ");        // false — whitespace counts as content!
string.IsNullOrEmpty("hello");     // false

string.IsNullOrWhiteSpace(null);   // true
string.IsNullOrWhiteSpace("");     // true
string.IsNullOrWhiteSpace("  ");   // true — whitespace
string.IsNullOrWhiteSpace("hello");  // false
```

`IsNullOrWhiteSpace` обычно лучше — обрабатывает «пустой пользовательский input» (пробелы).

> [!question]- Интервью: чем `IsNullOrEmpty` отличается от `IsNullOrWhiteSpace`?
> `IsNullOrEmpty(s)` возвращает true только если `s == null` или `s.Length == 0`. Whitespace (`"   "`) НЕ считается empty. `IsNullOrWhiteSpace(s)` дополнительно проверяет, что все характеры — whitespace (`Char.IsWhiteSpace`). Для validation user input обычно нужен `IsNullOrWhiteSpace` — пробелы воспринимаются как «нет ввода». `IsNullOrEmpty` — для строгой проверки «вообще нет content».

---

## 6. Format strings и interpolation

### 6.1. String.Format — composite formatting

```csharp
string s = string.Format("Hello, {0}! You are {1} years old.", "Alice", 30);
// "Hello, Alice! You are 30 years old."

// С форматом
string price = string.Format("{0:F2}", 1234.5m);   // "1234.50"
string date = string.Format("{0:yyyy-MM-dd}", DateTime.Now);
string num = string.Format("{0:N0}", 1000000);     // "1,000,000"

// С шириной
string padded = string.Format("[{0,10}]", "hi");   // "[        hi]"
string left = string.Format("[{0,-10}]", "hi");    // "[hi        ]"
```

### 6.2. Interpolation — современный подход

```csharp
string name = "Alice";
int age = 30;

// Эквивалент string.Format
string s = $"Hello, {name}! You are {age} years old.";

// Format specifiers тоже работают
string s2 = $"Price: {1234.5:F2}";              // "Price: 1234.50"
string s3 = $"Date: {DateTime.Now:yyyy-MM-dd}";
string s4 = $"[{name,10}]";                      // padding
```

В .NET 6+ interpolation использует `DefaultInterpolatedStringHandler` — без allocation для simple cases, builder-like для complex.

### 6.3. Standard format specifiers

| Специфер | Что | Пример |
|----------|-----|--------|
| `D` / `D5` | Decimal integer | `42.ToString("D5")` → `"00042"` |
| `F` / `F2` | Fixed-point | `1234.5m.ToString("F2")` → `"1234.50"` |
| `N` / `N0` | Number with separators | `1000000.ToString("N0")` → `"1,000,000"` |
| `C` | Currency | `1234.5m.ToString("C")` → `"$1,234.50"` (locale-dependent) |
| `P` / `P2` | Percent | `0.42.ToString("P0")` → `"42 %"` |
| `X` / `X4` | Hex | `255.ToString("X")` → `"FF"` |
| `E` | Exponential | `1234.5.ToString("E2")` → `"1.23E+003"` |
| `G` | General | автоматически выбирает |

### 6.4. DateTime format

| Специфер | Что |
|----------|-----|
| `d` | short date (locale) |
| `D` | long date |
| `t` | short time |
| `T` | long time |
| `g` | short datetime |
| `G` | long datetime |
| `o` | round-trip ISO 8601 |
| `s` | sortable ISO без ms |
| `u` | universal sortable UTC |
| `R` | RFC 1123 |

```csharp
var dt = DateTime.Now;
dt.ToString("o", CultureInfo.InvariantCulture);   // "2024-05-15T10:30:45.1234567"
dt.ToString("yyyy-MM-dd HH:mm:ss");                // custom
```

### 6.5. Custom format

```csharp
double d = 1234567.89;

d.ToString("0,000.00");          // "1,234,567.89"
d.ToString("#,###.##");           // "1,234,567.89"
d.ToString("0.00E+0");            // "1.23E+6"

// С условиями (positive;negative;zero)
d.ToString("0.00;(0.00);'zero'");   // "1234567.89", негативные с (), ноль как 'zero'
```

### 6.6. CultureInfo.InvariantCulture для wire format

```csharp
// ❌ Locale-dependent
decimal price = 1234.5m;
string s = price.ToString();   // "1234,5" в Russia, "1234.5" в US

// ✅ Stable
string s2 = price.ToString(CultureInfo.InvariantCulture);   // "1234.5" везде
```

Для JSON / БД / API contracts — всегда `InvariantCulture`. Для UI — `CurrentCulture`.

### 6.7. Verbatim interpolation `$@` или `@$`

```csharp
string user = "Alice";
string path = $@"C:\Users\{user}\Documents";   // \ как литерал, {user} interpolated
```

### 6.8. FormattableString

```csharp
FormattableString fs = $"Price: {1234.5:F2}";
fs.ToString(CultureInfo.InvariantCulture);   // "Price: 1234.50"

// Для SQL parameterized queries (avoid SQL injection)
FormattableString sql = $"SELECT * FROM Users WHERE Id = {userId}";
// Подходит для libraries (Dapper, EF Core), которые знают, как обрабатывать параметры.
```

`FormattableString` сохраняет parts отдельно от arguments — позволяет custom formatting / parameterization.

> [!question]- Интервью: чем `string.Format` отличается от interpolation?
> `string.Format("text {0} text {1}", arg1, arg2)` — runtime лookup placeholders по index, всегда string allocation. `$"text {arg1} text {arg2}"` (interpolation) — компилятор разворачивает в `String.Format` для C# < 6 / for compatibility, или в `DefaultInterpolatedStringHandler` (.NET 6+) который умнее: может использовать stack buffer для коротких строк, избежать boxing для value types. Performance: interpolation в .NET 6+ обычно быстрее. Readability: interpolation сильно лучше — нет `{0}`, `{1}` за которыми надо следить. Используй interpolation в новом коде.

---

## 7. Encoding

### 7.1. UTF-16 internally, не UTF-8

C# `string` хранится в **UTF-16** (один `char` = 16 бит = 2 байта). Это отличает от Python (UTF-8) и Go (UTF-8).

```csharp
string s = "hello";
s.Length;                                  // 5 — UTF-16 code units
Encoding.Unicode.GetByteCount(s);          // 10 — bytes (UTF-16 LE)
Encoding.UTF8.GetByteCount(s);             // 5  — bytes UTF-8
```

### 7.2. Encoding API

```csharp
using System.Text;

string s = "hello";

// String → bytes
byte[] utf8 = Encoding.UTF8.GetBytes(s);              // [104, 101, 108, 108, 111]
byte[] utf16 = Encoding.Unicode.GetBytes(s);          // UTF-16 LE
byte[] ascii = Encoding.ASCII.GetBytes(s);            // [104, 101, 108, 108, 111]

// Bytes → string
string s2 = Encoding.UTF8.GetString(utf8);
string s3 = Encoding.Unicode.GetString(utf16);

// Размеры
Encoding.UTF8.GetByteCount(s);
Encoding.UTF8.GetMaxByteCount(s.Length);   // upper bound для buffer allocation
```

### 7.3. UTF-8 — стандарт для wire format

```csharp
// HTTP, файлы, БД, JSON, XML — всё UTF-8
// .NET по default использует UTF-8 для большинства I/O

await File.WriteAllTextAsync("data.txt", text, Encoding.UTF8);
string text = await File.ReadAllTextAsync("data.txt", Encoding.UTF8);

// HTTP request
var content = new StringContent(json, Encoding.UTF8, "application/json");
```

### 7.4. UTF-8 без BOM

```csharp
// ❌ Encoding.UTF8 имеет BOM (0xEF 0xBB 0xBF в начале файла) на Windows
File.WriteAllText("data.txt", text, Encoding.UTF8);
// → файл начинается с EF BB BF

// ✅ Без BOM (для Linux compatibility)
var utf8NoBom = new UTF8Encoding(encoderShouldEmitUTF8Identifier: false);
File.WriteAllText("data.txt", text, utf8NoBom);
```

BOM создаёт проблемы: shell scripts, JSON parsers, JavaScript loaders могут не понимать. **Всегда UTF-8 без BOM** для cross-platform.

### 7.5. Surrogate pairs — emoji и rare chars

```csharp
string emoji = "😀";
emoji.Length;                              // 2! — surrogate pair
emoji[0];                                   // высокий surrogate
emoji[1];                                   // низкий surrogate

// Правильное iteration по «символам» (rune = Unicode code point)
foreach (Rune rune in emoji.EnumerateRunes())
{
    Console.WriteLine(rune);   // 😀 (один rune)
}

// .NET 5+
emoji.EnumerateRunes().Count();   // 1
```

**Ловушка:** `s.Length` ≠ количество видимых символов. Для UI display / counting нужен `Rune`-based iteration.

### 7.6. ASCII vs Latin1 vs UTF-8

```csharp
// ASCII — только 0-127, остальное → ?
Encoding.ASCII.GetBytes("café");  // c, a, f, ?  (é не ASCII)

// Latin1 (ISO-8859-1) — 0-255
Encoding.Latin1.GetBytes("café"); // c, a, f, é (é = 0xE9)

// UTF-8 — все Unicode
Encoding.UTF8.GetBytes("café");   // c, a, f, 0xC3 0xA9 (é в UTF-8 — 2 байта)
```

В новом коде — **всегда UTF-8**. ASCII / Latin1 — для legacy interop only.

### 7.7. `.GetBytes()` vs `EncodingExtensions.GetBytes(Span)`

```csharp
// Allocating
byte[] bytes = Encoding.UTF8.GetBytes(s);

// Без allocation — пишет в существующий buffer
Span<byte> buffer = stackalloc byte[256];
int written = Encoding.UTF8.GetBytes(s, buffer);
ReadOnlySpan<byte> result = buffer[..written];
```

Для perf-critical I/O — Span-based API.

> [!question]- Интервью: какой encoding использует C# string внутри?
> **UTF-16** (точнее, UTF-16 LE на little-endian platforms). Один `char` = 16 бит = 2 байта. Это отличает от Python (UTF-8 в str) и Go (UTF-8 в string). Большинство Unicode characters помещаются в один `char`, но эмодзи и редкие символы (CJK extensions) требуют **surrogate pairs** — 2 char на character. Для wire format / persistence — конвертировать в UTF-8 через `Encoding.UTF8.GetBytes()`. Для iteration по «настоящим» символам — `Rune.EnumerateRunes()`.

---

## 8. StringBuilder

### 8.1. Зачем StringBuilder

```csharp
// ❌ string concat в loop — O(n²)
string result = "";
for (int i = 0; i < 10000; i++)
    result += i + ",";

// ✅ StringBuilder — O(n)
var sb = new StringBuilder();
for (int i = 0; i < 10000; i++)
    sb.Append(i).Append(',');
string result = sb.ToString();
```

`StringBuilder` имеет growable buffer. Append амортизированно O(1). Final `ToString()` — одна allocation.

### 8.2. API

```csharp
var sb = new StringBuilder();

// Добавление
sb.Append("hello");
sb.Append(' ');
sb.Append(42);                   // int — автоматически конвертирует
sb.AppendLine("world");          // + Environment.NewLine
sb.AppendFormat("{0:F2}", 3.14);  // как string.Format

// Цепочки (fluent)
sb.Append("a").Append("b").Append("c");

// Insert / Remove / Replace
sb.Insert(0, "prefix: ");
sb.Remove(0, 8);
sb.Replace("hello", "Hi");

// Длина
sb.Length;        // 5
sb.Capacity;      // currentn buffer size

// Финал
string result = sb.ToString();
string sub = sb.ToString(0, 5);   // substring
```

### 8.3. Pre-allocate capacity

```csharp
// ❌ Default capacity 16, потом удваивается — несколько realloc
var sb1 = new StringBuilder();

// ✅ Известна примерная длина — pre-allocate
var sb2 = new StringBuilder(capacity: 1024);

// Pre-allocate с initial value
var sb3 = new StringBuilder("initial", capacity: 1024);
```

Pre-allocation избегает несколько resize operations. Полезно когда знаешь approximate size.

### 8.4. Clear для re-use

```csharp
var sb = new StringBuilder();

// Use 1
sb.Append("first");
string s1 = sb.ToString();
sb.Clear();   // освобождает контент, сохраняет capacity

// Use 2
sb.Append("second");
string s2 = sb.ToString();
```

`Clear()` обнуляет Length, но сохраняет buffer. Полезно для re-using StringBuilder в hot path.

### 8.5. StringBuilder vs Concat — когда что

```
Use StringBuilder когда:
  - Concat в loop (любой N > ~5)
  - Условный append (несколько if/else строящих текст)
  - Replace/Remove нужны на partial-built string

Use string concat / interpolation когда:
  - Известное малое количество частей (2-5)
  - В одной expression (компилятор оптимизирует)
  - Out of hot path (readability важнее)
```

### 8.6. Performance threshold

```
| Method                  | N=2 | N=10 | N=100 | N=10000 |
|------------------------ |----:|-----:|------:|--------:|
| string +                | 1x  |  3x  |  20x  | 1000x   |
| string.Concat           | 1x  |  1x  |   1x  |    1x   |
| string.Join             | 1x  |  1x  |   1x  |    1x   |
| StringBuilder           | 2x  |  1x  |   1x  |    1x   |
| Interpolation (.NET 6+) | 1x  |  1x  |   1x  |    1x   |
```

Для N=2-5 — interpolation / concat ничем не хуже. Для N > 10 в loop — StringBuilder обязательно.

### 8.7. ValueStringBuilder (advanced)

`ValueStringBuilder` — internal struct в BCL для perf-critical code. Не public API, но можно эмулировать:

```csharp
// Используется через Span<char> + stackalloc
Span<char> buffer = stackalloc char[256];
int written = 0;

// "Write" в buffer без heap
"hello".CopyTo(buffer);
written += 5;

// Convert to string — единственная allocation
string result = new string(buffer[..written]);
```

Для hot path с очень короткими строками — даёт zero-allocation.

### 8.8. CompositeFormat (.NET 8+)

```csharp
// Pre-parsed format string
private static readonly CompositeFormat OrderFormat =
    CompositeFormat.Parse("Order #{0} for ${1:F2} placed at {2:O}");

// Use многократно — без re-parsing
string s = string.Format(null, OrderFormat, orderId, amount, dateTime);
```

Если один format string используется тысячи раз — pre-parse через `CompositeFormat`. Микро-оптимизация для hot path.

> [!question]- Интервью: когда использовать StringBuilder vs string?
> Используй **StringBuilder** когда: 1) concat в loop с любым N > 5-10. 2) условные append (несколько if/else строящих текст). 3) много модификаций (Replace/Insert/Remove) на partial result. Используй **string** когда: 1) известное малое число частей (2-5) — interpolation или concat. 2) в одной expression — compiler оптимизирует chain `+` в `string.Concat`. 3) read-only API contracts. Главное правило: `+=` в loop — anti-pattern (O(N²)). StringBuilder с pre-allocated capacity — O(N).

---

## 9. `Span<char>` и `ReadOnlySpan<char>` для perf

### 9.1. Зачем Span

`Substring` создаёт новую string. Для парсинга / стриминг это аллокации каждый раз. `Span<char>` / `ReadOnlySpan<char>` — view над существующей памятью, без копирования.

```csharp
string url = "https://api.example.com/users/42";

// ❌ Substring аллоцирует
string scheme = url.Substring(0, 5);          // "https" — heap allocation
string host = url.Substring(8, 15);            // "api.example.com" — heap

// ✅ Span — view над тем же memory
ReadOnlySpan<char> urlSpan = url.AsSpan();
ReadOnlySpan<char> scheme2 = urlSpan[..5];     // "https" — no allocation
ReadOnlySpan<char> host2 = urlSpan.Slice(8, 15);
```

Span — `ref struct`, существует только на stack, не может быть полем класса.

### 9.2. AsSpan / Slice

```csharp
string s = "Hello, World!";
ReadOnlySpan<char> span = s.AsSpan();              // вся строка
ReadOnlySpan<char> hello = s.AsSpan(0, 5);         // "Hello"
ReadOnlySpan<char> world = s.AsSpan(7, 5);         // "World"
ReadOnlySpan<char> world2 = span.Slice(7, 5);      // эквивалент
```

`AsSpan(start, length)` или `.Slice(start, length)`. C# range syntax `span[7..12]`.

### 9.3. Comparison через Span

```csharp
string s = "hello world";
ReadOnlySpan<char> first = s.AsSpan(0, 5);

first.Equals("hello", StringComparison.Ordinal);   // true — без allocation
first.SequenceEqual("hello");                       // true — char-by-char

first.StartsWith("hel");                           // true
first.EndsWith("llo");                              // true
first.Contains('l');                                // true
```

Все методы string работают на span — без создания substring.

### 9.4. Парсинг через Span

```csharp
// ❌ С allocations
string csv = "a,b,c,d";
foreach (var part in csv.Split(','))
    ProcessPart(part);   // каждый part — новая string

// ✅ Span-based, без allocations
ReadOnlySpan<char> csvSpan = csv.AsSpan();
while (!csvSpan.IsEmpty)
{
    int idx = csvSpan.IndexOf(',');
    ReadOnlySpan<char> part;
    if (idx == -1)
    {
        part = csvSpan;
        csvSpan = ReadOnlySpan<char>.Empty;
    }
    else
    {
        part = csvSpan[..idx];
        csvSpan = csvSpan[(idx + 1)..];
    }
    ProcessPart(part);
}

void ProcessPart(ReadOnlySpan<char> part) { /* без allocation */ }
```

В hot path парсера это ×10-100 perf boost.

### 9.5. int.Parse / TryParse с Span (.NET 8+)

```csharp
ReadOnlySpan<char> span = "42".AsSpan();

int n = int.Parse(span);
if (int.TryParse(span, out int value)) { }

// С CultureInfo
decimal d = decimal.Parse(span, CultureInfo.InvariantCulture);
```

Большинство BCL-парсеров поддерживают Span — без string allocation.

### 9.6. stackalloc для temp buffer

```csharp
// Ad-hoc string building без heap
Span<char> buffer = stackalloc char[64];
int written = 0;

"prefix: ".CopyTo(buffer);
written += 8;

if (number.TryFormat(buffer[written..], out int charsWritten))
    written += charsWritten;

// Final allocation — только при необходимости
string result = new string(buffer[..written]);
```

Stack memory дешевле heap. Для temp строк до ~1 KB — `stackalloc`.

### 9.7. Span limitations

```csharp
// ❌ Span не может быть полем
public class MyClass
{
    private ReadOnlySpan<char> _data;   // Compile error
}

// ❌ Span не может быть в async методе (прямо)
public async Task M()
{
    ReadOnlySpan<char> s = "hello".AsSpan();
    await Task.Delay(100);   // Compile error — span можно cross await
}

// ❌ Span не может быть generic параметром
List<ReadOnlySpan<char>> list = new();   // CS0306
```

Все ограничения из-за `ref struct` природы Span. Для long-lived storage — `ReadOnlyMemory<char>`.

### 9.8. `ReadOnlyMemory<char>`

```csharp
// Memory — heap-friendly альтернатива Span
ReadOnlyMemory<char> mem = "hello".AsMemory();

// Можно хранить в полях, async
public class MyClass
{
    public ReadOnlyMemory<char> Data { get; }
}

// Конвертация в span когда нужно
ReadOnlySpan<char> span = mem.Span;
```

`Memory` хранит в heap, `Span` — реальный view на каждом access. Используй Memory для long-lived, Span — для immediate processing.

### 9.9. `SearchValues<T>` — быстрый multi-value поиск (.NET 8+)

`IndexOfAny(char[])` при каждом вызове заново анализирует набор искомых символов. `SearchValues<T>` (namespace `System.Buffers`) — **предвычисленная** структура поиска: создаёшь один раз в `static readonly`, а BCL при создании выбирает оптимальную стратегию под конкретный набор (для ASCII — битовая карта на 128 бит, проверка символа за O(1), плюс SIMD-векторизация — сравнение сразу пачки символов за такт).

```csharp
using System.Buffers;

public static class Sanitizer
{
    // Создаётся один раз — вся оптимизация происходит здесь
    private static readonly SearchValues<char> Forbidden =
        SearchValues.Create("<>&\"'");

    public static bool NeedsEncoding(ReadOnlySpan<char> input) =>
        input.ContainsAny(Forbidden);          // работает поверх Span из 9.1-9.4

    public static int FirstForbidden(ReadOnlySpan<char> input) =>
        input.IndexOfAny(Forbidden);
}
```

.NET 9 добавил `SearchValues<string>` — поиск первого вхождения любой из **подстрок** (внутри — векторизованный multi-substring алгоритм, ручной цикл с `Contains` по списку так не сможет):

```csharp
private static readonly SearchValues<string> SuspiciousTags = SearchValues.Create(
    ["<script", "<iframe", "<object"],
    StringComparison.OrdinalIgnoreCase);

bool suspicious = html.AsSpan().IndexOfAny(SuspiciousTags) >= 0;
```

**Roslyn сам подскажет:** анализатор **CA1870** (Use a cached `SearchValues` instance) срабатывает, когда ты передаёшь в `IndexOfAny` / `ContainsAny` константный набор значений, и code-fix автоматически выносит его в `static readonly SearchValues<char>`. Если видишь это предупреждение — соглашайся: выигрыш на hot path кратный, аллокаций ноль.

Ключевое правило: `SearchValues` окупается **при переиспользовании** — кэшируй в `static readonly`, не создавай в цикле (создание дороже одного поиска).

> [!question]- Интервью: чем `Span<char>` отличается от `string.Substring`?
> `Substring` создаёт **новую** string на heap (O(n) копия). `Span<char>` / `ReadOnlySpan<char>` — это **view** над существующей памятью, **без allocation** (O(1)). Span хорош для парсеров и hot path. Ограничения Span: `ref struct`, нельзя в полях класса, нельзя в async методах (через await), нельзя в generic параметрах, живёт только в stack frame. Для long-lived storage — `ReadOnlyMemory<char>` (heap-friendly альтернатива). С .NET 6+ большинство BCL парсеров (int.Parse, decimal.Parse, DateTime.Parse) поддерживают Span — true zero-allocation парсинг.

---

## 10. String pooling и interning

### 10.1. Что такое intern pool

Common literals shared в одном памяти CLR:

```csharp
string a = "hello";
string b = "hello";

ReferenceEquals(a, b);   // true! — interned
```

Compile-time literals автоматически interned. CLR хранит таблицу — при загрузке assembly literals регистрируются.

### 10.2. Manual interning

```csharp
string a = "hello";
string b = string.Intern("hello");   // ищет в pool, добавляет если нет
ReferenceEquals(a, b);                // true

// Создаёт через runtime
string c = new string(['h', 'e', 'l', 'l', 'o']);   // не interned
ReferenceEquals(a, c);                // false
string d = string.Intern(c);          // теперь есть в pool
ReferenceEquals(a, d);                // true
```

`string.Intern(s)` возвращает interned версию или регистрирует.

### 10.3. Когда interning полезен

```csharp
// Many duplicate strings — экономия памяти
public class Token
{
    public string Type { get; set; } = "";   // "keyword", "identifier", "literal"
}

var tokens = new List<Token>();
foreach (var token in ParseSourceCode())
{
    tokens.Add(new Token { Type = string.Intern(token.Type) });
}
// Все "keyword" tokens shared один string instance в memory
```

Полезно для: parsers, log messages с repeating strings, configuration keys.

### 10.4. Когда interning вреден

```csharp
// ❌ Interning user input
string s = string.Intern(userInput);   // НЕ делай!
```

Проблемы:
1. Intern pool **никогда не освобождается** в lifetime AppDomain.
2. Если intern many user inputs — **memory leak**.
3. Hash collision attacks возможны (если CryptoSafe hash не используется).

Только для **stable, known set** strings.

### 10.5. IsInterned

```csharp
string s = "hello";
string? interned = string.IsInterned(s);   // returns string if in pool, else null

if (interned != null)
    // s is interned
```

Полезно для diagnostics. В обычном коде не нужен.

### 10.6. Не путай с intern pool в Dictionary

```csharp
// String deduplication через ConditionalWeakTable / dictionary
var pool = new Dictionary<string, string>();

string Dedupe(string s)
{
    if (pool.TryGetValue(s, out var existing))
        return existing;
    pool[s] = s;
    return s;
}

// Custom dedup — managed lifetime, можно clear
var s1 = Dedupe("hello");
var s2 = Dedupe("hello");
ReferenceEquals(s1, s2);   // true
```

Свой pool можно очищать. CLR intern pool — нет.

> [!question]- Интервью: что такое string interning и когда полезен?
> **Intern pool** — таблица в CLR, где shared common strings (literals автоматически interned). `ReferenceEquals(a, b)` для двух одинаковых literals возвращает true. Полезно для: 1) экономии памяти при множественных дубликатах (parsers, tokenizers, log keys). 2) faster equality (`==` через reference compare если оба interned). Опасности: 1) intern pool **не освобождается** — memory leak при interning user input. 2) только для stable known strings. Custom dedup-pool через Dictionary — managed lifetime, можно clear. В большинстве кода interning не нужен — CLR делает достаточно автоматически.

---

## 11. Regex основы

### 11.1. Что такое regex

**Regular expression** — pattern для поиска/замены текста по правилам. В .NET через `System.Text.RegularExpressions.Regex`:

```csharp
using System.Text.RegularExpressions;

var regex = new Regex(@"\d+");          // pattern: одна или более digits
var match = regex.Match("hello 42 world 100");
match.Success;     // true
match.Value;       // "42"
match.Index;       // 6
```

Regex — мощный инструмент, но дорогой и сложный для отладки. Использовать **точечно**.

### 11.2. Базовый синтаксис

| Pattern | Что | Пример |
|---------|-----|--------|
| `.` | Любой char (кроме newline) | `a.c` matches `abc`, `axc` |
| `\d` | Digit (0-9) | `\d+` matches `42`, `100` |
| `\D` | Не-digit | `\D+` matches `abc` |
| `\w` | Word char (letter, digit, _) | `\w+` matches `foo_bar123` |
| `\W` | Не-word | `\W` matches `, !` |
| `\s` | Whitespace | `\s+` matches spaces, tabs |
| `\S` | Не-whitespace | `\S+` matches `hello` |
| `[abc]` | Один из a, b, c | `[aeiou]` гласная |
| `[^abc]` | Не один из | `[^aeiou]` согласная |
| `[a-z]` | Диапазон | `[a-zA-Z0-9]` |

### 11.3. Anchors

```
^        Начало строки (или строки в multiline)
$        Конец строки
\b       Word boundary
\A       Начало входа
\z       Конец входа
```

```csharp
new Regex(@"^hello$").IsMatch("hello");           // true
new Regex(@"^hello$").IsMatch("hello world");     // false — есть мусор после
new Regex(@"\bhello\b").IsMatch("say hello!");    // true — граница слова
```

### 11.4. Quantifiers

| Quantifier | Что | Пример |
|-----------|-----|--------|
| `*` | 0 или больше | `a*` matches `""`, `"a"`, `"aaa"` |
| `+` | 1 или больше | `a+` matches `"a"`, `"aaa"` |
| `?` | 0 или 1 | `a?` matches `""`, `"a"` |
| `{n}` | Точно n | `a{3}` matches `"aaa"` |
| `{n,}` | n или больше | `a{2,}` matches `"aa"`, `"aaa"` |
| `{n,m}` | от n до m | `a{2,4}` matches `"aa"`, `"aaa"`, `"aaaa"` |

### 11.5. Greedy vs lazy

```csharp
// ❌ Greedy — берёт максимум
new Regex(@"<.+>").Match("<a><b><c>").Value;
// "<a><b><c>" — match всё!

// ✅ Lazy — берёт минимум (`?` после quantifier)
new Regex(@"<.+?>").Match("<a><b><c>").Value;
// "<a>" — minimum match
```

По умолчанию quantifiers — greedy. `?` после делает lazy. Lazy обычно нужен для поиска вложенных tags / quotes.

### 11.6. Groups

```csharp
// Capturing group
var regex = new Regex(@"(\w+)@(\w+\.\w+)");
var match = regex.Match("Email: user@example.com");
match.Groups[0].Value;   // "user@example.com" — full match
match.Groups[1].Value;   // "user"
match.Groups[2].Value;   // "example.com"

// Named group
var named = new Regex(@"(?<user>\w+)@(?<domain>\w+\.\w+)");
var m = named.Match("user@example.com");
m.Groups["user"].Value;     // "user"
m.Groups["domain"].Value;    // "example.com"

// Non-capturing group (для grouping без extraction)
var nc = new Regex(@"(?:foo|bar)\d+");
nc.IsMatch("foo42");   // true
```

### 11.7. Alternation

```csharp
new Regex(@"cat|dog|bird").IsMatch("I love cats");   // true (cat)
new Regex(@"^(yes|no|maybe)$").IsMatch("yes");        // true
```

`|` — «или». Группируется через `()`.

### 11.8. Lookahead / lookbehind

```csharp
// Positive lookahead — за чем следует
new Regex(@"\d+(?=px)").Match("10px 20em").Value;   // "10"

// Negative lookahead — за чем НЕ следует
new Regex(@"\d+(?!px)").Match("10px 20em").Value;   // "20"

// Lookbehind — что было до
new Regex(@"(?<=\$)\d+").Match("price: $100").Value;   // "100"
new Regex(@"(?<!\$)\d+").Match("100 items, $50").Value; // "100"
```

Lookarounds — zero-width assertions, не consume characters.

### 11.9. Escape characters

```csharp
// Special chars нужно escape: . * + ? ( ) [ ] { } \ | ^ $
new Regex(@"\$\d+").Match("price: $100").Value;   // "$100"
new Regex(@"a\.b").Match("a.b").Success;           // true (литеральная точка)
```

В `@"..."` (verbatim) backslash — литерал. В обычной string — `\\` для regex backslash.

> [!question]- Интервью: чем greedy quantifier отличается от lazy?
> **Greedy** (default — `*`, `+`, `?`, `{n,}`) — берёт **максимум** characters, потом backtracks если общий pattern не matched. **Lazy** (`*?`, `+?`, `??`, `{n,}?`) — берёт **минимум**, добавляет characters если pattern не matched. Пример: `<.+>` на `<a><b>` greedy match всё `<a><b>`, lazy `<.+?>` match только `<a>`. Lazy критичен для поиска nested tags / quoted strings — иначе захватывает слишком много. Lazy медленнее greedy в worst case (больше backtracking) но даёт правильный результат для частых задач.

---

## 12. Regex API

### 12.1. Match — single match

```csharp
var regex = new Regex(@"\d+");
var match = regex.Match("hello 42 world 100");

if (match.Success)
{
    Console.WriteLine(match.Value);   // "42"
    Console.WriteLine(match.Index);   // 6
    Console.WriteLine(match.Length);   // 2
}
```

`Match` возвращает первое match. Если нужны все — `Matches`.

### 12.2. Matches — все matches

```csharp
var matches = regex.Matches("a1 b22 c333");
foreach (Match m in matches)
    Console.WriteLine(m.Value);   // 1, 22, 333

// LINQ
var values = matches.Select(m => m.Value).ToList();   // ["1", "22", "333"]
```

### 12.3. IsMatch — bool check

```csharp
bool hasNumber = Regex.IsMatch("hello 42", @"\d+");   // true
```

Если нужен только bool — `IsMatch` быстрее `Match` (не extracts).

### 12.4. Replace

```csharp
// Простой replace
string result = Regex.Replace("hello world", @"\w+", "X");
// "X X"

// Replace с capture group
string formatted = Regex.Replace(
    "John Smith",
    @"(\w+)\s+(\w+)",
    "$2 $1");   // "Smith John" — swap

// С delegate (computed replacement)
string upper = Regex.Replace(
    "hello world",
    @"\w+",
    m => m.Value.ToUpper());
// "HELLO WORLD"
```

### 12.5. Split

```csharp
// Split по pattern (не по char)
string[] parts = Regex.Split("a1 b22 c333", @"\d+");
// ["a", " b", " c", ""]

// С whitespace
string[] words = Regex.Split("hello   world\t\tfoo", @"\s+");
// ["hello", "world", "foo"]
```

`Regex.Split` мощнее `string.Split` — по pattern, не по фиксированным разделителям.

### 12.6. RegexOptions

```csharp
var regex = new Regex(@"hello", RegexOptions.IgnoreCase);
regex.IsMatch("HELLO");   // true

// Multiple options
var multi = new Regex(@"^foo", RegexOptions.IgnoreCase | RegexOptions.Multiline);

// Important options:
// IgnoreCase     — case-insensitive
// Multiline      — ^ и $ matches line-start/end (не string)
// Singleline     — . matches newline
// Compiled       — JIT-compile в IL (faster если pattern reused)
// CultureInvariant — culture-independent comparison
// IgnorePatternWhitespace — ignore whitespace в pattern + comments
```

### 12.7. Static vs instance

```csharp
// Instance — кэшируется и reused
private static readonly Regex EmailRegex = new(@"^[\w]+@[\w]+\.[\w]+$");
EmailRegex.IsMatch("user@example.com");

// Static — convenience, но parses pattern каждый раз (cache в Regex internally до 15)
Regex.IsMatch("user@example.com", @"^[\w]+@[\w]+\.[\w]+$");
```

Best practice: для часто используемых паттернов — `static readonly Regex` field.

### 12.8. Timeout — защита от ReDoS

```csharp
// Без timeout — может зависнуть на catastrophic backtracking
var regex = new Regex(@"^(\w+)*$");

// ✅ С timeout
var safe = new Regex(@"^(\w+)*$", RegexOptions.None, TimeSpan.FromSeconds(1));

try
{
    safe.IsMatch(maliciousInput);
}
catch (RegexMatchTimeoutException)
{
    // Defense triggered
}
```

Для user input — **обязательно timeout**. Раздел 14 (catastrophic backtracking) объясняет почему.

> [!question]- Интервью: почему `static readonly Regex` лучше, чем static method?
> Создание Regex объекта парсит pattern и компилирует internal automaton — это медленно. `static readonly Regex` инициализируется один раз при class load, переиспользуется. `Regex.IsMatch(input, pattern)` (static) кэширует последние ~15 patterns — но для high-volume code это insufficient. Best practice: для каждого reused pattern — отдельный `static readonly Regex` field. С .NET 7+ `[GeneratedRegex]` source generator делает это ещё лучше — парсинг и compilation на compile-time, AOT-friendly.

---

## 13. Regex source generators (.NET 7+)

### 13.1. Что такое

`[GeneratedRegex]` атрибут — source generator, который генерирует optimized regex код **на compile-time**:

```csharp
public partial class Validators
{
    [GeneratedRegex(@"^[\w._%+-]+@[\w.-]+\.[a-zA-Z]{2,}$", RegexOptions.IgnoreCase)]
    private static partial Regex EmailRegex();
}

// Использование
bool isValid = Validators.EmailRegex().IsMatch("user@example.com");
```

**Преимущества:**
- **Faster** — нет runtime parsing/compilation.
- **AOT-friendly** — работает с Native AOT.
- **Strongly-typed** — IDE подсказывает named groups.
- **Compile-time errors** — invalid pattern не compile.

### 13.2. Как использовать

```csharp
// Класс должен быть partial
public partial class MyClass
{
    // Метод должен быть partial и static
    [GeneratedRegex(@"\d+")]
    private static partial Regex DigitRegex();
}

// IDE автоматически генерирует implementation
var match = MyClass.DigitRegex().Match("hello 42");
```

Generation работает через partial classes — generator пишет вторую часть partial.

### 13.3. С options

```csharp
[GeneratedRegex(
    @"^(?<scheme>https?)://(?<host>[\w.-]+)/(?<path>.*)$",
    RegexOptions.IgnoreCase | RegexOptions.Compiled)]
private static partial Regex UrlRegex();

// Использование с named groups
var m = UrlRegex().Match("https://example.com/users/42");
m.Groups["scheme"].Value;  // "https"
m.Groups["host"].Value;     // "example.com"
m.Groups["path"].Value;     // "users/42"
```

### 13.4. Performance

```
| Method                    |    Mean |    Allocated |
|-------------------------- |--------:|-------------:|
| Regex.IsMatch (static)    | 200 ns  |       0 B    |
| new Regex().IsMatch       | 5000 ns |    >1000 B   |  ← runtime parse
| static readonly Regex     | 200 ns  |       0 B    |
| [GeneratedRegex]          |  50 ns  |       0 B    |  ← 4x faster
```

Для high-volume или AOT — generated regex выигрывает.

### 13.5. Compile-time errors

```csharp
// ❌ Invalid pattern — compile error CS9102
[GeneratedRegex(@"[invalid")]
private static partial Regex BadRegex();
// Error: Cannot parse the regular expression
```

Это early feedback — bug detected до runtime.

### 13.6. Когда использовать

✅ **Используй `[GeneratedRegex]` когда:**
- .NET 7+ доступна.
- Pattern reused (не одноразовый).
- Native AOT compilation.
- Hot path где speed важен.

❌ **Не используй когда:**
- Dynamic pattern (приходит из конфига / runtime).
- One-off use.
- Старые .NET версии (< 7).

Для legacy / dynamic — обычный `static readonly Regex`.

### 13.7. Migration с обычного Regex

```csharp
// Before
public class Validators
{
    private static readonly Regex EmailRegex = new(
        @"^[\w._%+-]+@[\w.-]+\.[a-zA-Z]{2,}$",
        RegexOptions.IgnoreCase | RegexOptions.Compiled);

    public bool IsEmail(string s) => EmailRegex.IsMatch(s);
}

// After (.NET 7+)
public partial class Validators
{
    [GeneratedRegex(@"^[\w._%+-]+@[\w.-]+\.[a-zA-Z]{2,}$", RegexOptions.IgnoreCase)]
    private static partial Regex EmailRegex();

    public bool IsEmail(string s) => EmailRegex().IsMatch(s);
}
```

Migration механический — RegexOptions.Compiled неявный для Generated.

> [!question]- Интервью: что даёт `[GeneratedRegex]` в .NET 7+?
> Source generator превращает regex pattern в optimized C# код **на compile-time** вместо runtime parsing. Преимущества: 1) **Performance** ~4x быстрее (no runtime parse, no JIT compile). 2) **AOT-friendly** — работает с Native AOT publishing. 3) **Compile-time errors** — invalid regex pattern не compile. 4) **Strongly-typed** named groups в IntelliSense. Использование: `[GeneratedRegex(@"...")]` на partial static method. Класс тоже partial. Для production кода с reused regex — рекомендуемый подход. Не для dynamic patterns (которые из конфига).

---

## 14. Common patterns и pitfalls

### 14.1. Standard patterns

| Что | Pattern |
|-----|---------|
| **Email (basic)** | `^[\w._%+-]+@[\w.-]+\.[a-zA-Z]{2,}$` |
| **URL** | `^https?://[\w.-]+(/.*)?$` |
| **GUID** | `^[0-9a-fA-F]{8}-[0-9a-fA-F]{4}-[0-9a-fA-F]{4}-[0-9a-fA-F]{4}-[0-9a-fA-F]{12}$` |
| **Phone (US)** | `^\+?1?[-.\s]?\(?\d{3}\)?[-.\s]?\d{3}[-.\s]?\d{4}$` |
| **IPv4** | `^(25[0-5]\|2[0-4]\d\|[01]?\d?\d)(\.(25[0-5]\|2[0-4]\d\|[01]?\d?\d)){3}$` |
| **Date YYYY-MM-DD** | `^\d{4}-\d{2}-\d{2}$` |
| **Hex color** | `^#[0-9a-fA-F]{6}$` |
| **Username (alphanumeric + underscore, 3-20)** | `^[a-zA-Z0-9_]{3,20}$` |
| **Password (min 8, 1 upper, 1 lower, 1 digit)** | `^(?=.*[a-z])(?=.*[A-Z])(?=.*\d).{8,}$` |

**Важно:** для **email validation** — regex выше упрощённый. Real RFC 5321 email — practically невозможен в regex. Use `MailAddress` class:

```csharp
try
{
    var addr = new MailAddress(input);
    bool isValid = addr.Address == input;
}
catch
{
    // Invalid
}
```

### 14.2. Catastrophic backtracking — главная ловушка

```csharp
// ❌ Vulnerable pattern
var regex = new Regex(@"^(a+)+$");

// На "aaaaaaaaaaaaaaab" (нет b в конце) — regex может тратить часы!
regex.IsMatch("aaaaaaaaaaaaaaab");
```

**Механизм:** `(a+)+` создаёт exponentially много возможностей разбить string на группы. Когда match fails, regex engine пробует все комбинации.

**Признаки vulnerable patterns:**
- Nested quantifiers: `(a+)+`, `(a*)*`, `(a|a)*`.
- Overlapping alternations: `(a|a)+`.
- Repeating с overlap: `\w+\d`.

**Защита:**
1. **Timeout** — `new Regex(pattern, RegexOptions.None, TimeSpan.FromSeconds(1))`.
2. **Test untrusted input** — fuzz testing.
3. **Use atomic groups** — `(?>...)` — no backtracking.
4. **Generated regex** — .NET 7+ generators избегают многих backtracking issues.

### 14.3. ReDoS — security issue

ReDoS (Regular Expression Denial of Service) — атака через regex с user input:

```csharp
// ❌ Уязвимый код
public bool ValidateInput(string input)
{
    return Regex.IsMatch(input, @"^(a+)+$");
}

// Attacker отправляет "aaaaaaaaaaaaaaaaaaaaaab" — server hangs!
```

Defense:
- **Timeout** обязателен для user input.
- **Sanitize patterns** — не позволяй user-supplied regex.
- **Use safe patterns** — proven libraries / generated.

### 14.4. Verbatim strings для regex

```csharp
// ❌ С обычной string — двойные backslash
var regex1 = new Regex("\\d+\\s\\w+");

// ✅ Verbatim — read-able
var regex2 = new Regex(@"\d+\s\w+");

// ✅ Multi-line с IgnorePatternWhitespace
var regex3 = new Regex(@"
    \d+        # one or more digits
    \s         # whitespace
    \w+        # word chars
    ", RegexOptions.IgnorePatternWhitespace);
```

`@"..."` + `IgnorePatternWhitespace` — read-able regex для complex patterns.

### 14.5. Regex для парсинга — anti-pattern

```csharp
// ❌ Regex для HTML / JSON / SQL — anti-pattern
var hrefRegex = new Regex(@"<a\s+href=""([^""]+)""");
```

Real HTML / JSON / SQL имеют context (nesting, quotes, escapes), которые regex плохо обрабатывает. Используй:

- **HTML** — HtmlAgilityPack, AngleSharp.
- **JSON** — `System.Text.Json`.
- **SQL** — parameterized queries / ORM.
- **CSV** — CsvHelper.

Regex — только для simple pattern matching, не для structured parsing.

> [!question]- Интервью: что такое catastrophic backtracking?
> Worst-case behavior regex engine, когда pattern содержит nested quantifiers (`(a+)+`, `(a*)*`) и input «почти matches, но не quite» — engine пробует exponentially много комбинаций группировки characters. Может тратить часы на короткий input. Это **ReDoS** (Regular Expression DoS) attack vector. Защиты: 1) `TimeSpan` timeout в Regex constructor — обязательно для user input. 2) Atomic groups `(?>...)` — disable backtracking. 3) Avoid nested quantifiers. 4) `[GeneratedRegex]` (.NET 7+) с safer matching engine. 5) Test patterns на untrusted input.

---

## 15. Best Practices

### 15.1. Strings

- ✅ **`StringComparison`** в каждом сравнении — `Ordinal`/`OrdinalIgnoreCase` для backend.
- ✅ **`InvariantCulture`** для wire format / parsing / formatting.
- ✅ **`CurrentCulture`** только для UI display.
- ✅ **`IsNullOrWhiteSpace`** для validation user input (вместо IsNullOrEmpty).
- ✅ **Interpolation** в новом коде вместо `string.Format`.
- ✅ **Raw string literals** (`"""..."""`) для embedded JSON/SQL/HTML.
- ❌ **`+=` в loop** — anti-pattern (O(N²)).
- ❌ **`ToLower()` / `ToUpper()` для comparison** — используй `StringComparison`.
- ❌ **Equals без `StringComparison`** — CA1307 warning.
- ❌ **`Parse` без `InvariantCulture`** — locale-dependent baggage.

### 15.2. StringBuilder

- ✅ **Pre-allocate capacity** если знаешь approximate size.
- ✅ **Fluent chain** — `sb.Append(a).Append(b).Append(c)`.
- ✅ **Clear()** для re-use, не `new StringBuilder()`.
- ❌ **Не для коротких строк** — overhead больше benefit.
- ❌ **Не для одной expression** — interpolation проще.

### 15.3. Encoding

- ✅ **UTF-8** для wire format / files / БД.
- ✅ **UTF-8 без BOM** для cross-platform.
- ✅ **`Encoding.UTF8`** explicit при I/O.
- ✅ **`Rune.EnumerateRunes()`** для iteration по «настоящим» characters (с emoji).
- ❌ **ASCII / Latin1** — только для legacy interop.
- ❌ **Polагание на `s.Length`** для подсчёта symbols (surrogate pairs!).

### 15.4. Span

- ✅ **`AsSpan()`** для парсинга — без allocations.
- ✅ **`stackalloc`** для temp buffers (< 1 KB).
- ✅ **Span-based parse APIs** — `int.Parse(span)`, `decimal.Parse(span)`.
- ❌ **Span не в полях класса** — используй `Memory<T>`.
- ❌ **Span не в async** — может быть проблема с await crossing.

### 15.5. Regex

- ✅ **`static readonly Regex`** для reused patterns.
- ✅ **`[GeneratedRegex]`** (.NET 7+) для best perf и AOT.
- ✅ **`TimeSpan` timeout** для user input (защита от ReDoS).
- ✅ **Verbatim string `@"..."`** для regex patterns.
- ✅ **`IgnorePatternWhitespace`** для read-able complex patterns.
- ✅ **Lazy quantifiers `?`** для nested matching.
- ❌ **Не для HTML/JSON/SQL** — используй real parsers.
- ❌ **Не для security-critical validation** — use libraries.
- ❌ **Не nested quantifiers** `(a+)+` — catastrophic backtracking.

### 15.6. Performance

- ✅ **`StringComparer.Ordinal`** для Dictionary keys (faster).
- ✅ **`Equals(other, Ordinal)`** не аллоцирует, ToLower — аллоцирует.
- ✅ **`Span<char>`** для парсеров.
- ✅ **`stackalloc`** для temp buffer без heap.
- ✅ **String pool / dedup** для repeated values.
- ❌ **Не concat в loop** — StringBuilder.
- ❌ **Не CurrentCulture** в backend — Ordinal.

---

## 16. Decision tree

```
Что нужно?
│
├── Создание строки
│   ├── Известное содержимое → literal "..."
│   ├── Multi-line или embedded → raw """..."""
│   ├── Path, regex pattern → verbatim @"..."
│   ├── С variables → interpolation $"..."
│   └── С interpolation + verbatim → $@"..." или $"""..."""
│
├── Concat
│   ├── 2-5 частей в одной expression → +
│   ├── Из collection → string.Join
│   ├── В loop → StringBuilder
│   └── С форматированием → interpolation $"..."
│
├── Comparison
│   ├── Backend / IDs / paths → StringComparison.Ordinal
│   ├── Case-insensitive backend → OrdinalIgnoreCase
│   ├── Cross-locale stable sort → InvariantCulture
│   └── UI display sorting → CurrentCulture
│
├── Search / manipulation
│   ├── Простой → IndexOf, Contains, StartsWith, Replace, Split
│   ├── Complex pattern → Regex
│   └── No allocation → Span методы (AsSpan, Slice, Equals)
│
├── Format
│   ├── Backend / wire → CultureInfo.InvariantCulture
│   ├── UI → CultureInfo.CurrentCulture
│   ├── Static format → interpolation
│   └── Reused format string → CompositeFormat (.NET 8+)
│
├── Encoding
│   ├── Files / network / БД → UTF-8 (без BOM)
│   ├── Internal → string (UTF-16)
│   └── Legacy → Encoding.ASCII / Latin1 only if needed
│
├── Performance hot path
│   ├── Парсинг → Span<char>
│   ├── Building → StringBuilder с pre-allocate
│   ├── Comparison → Ordinal
│   ├── Regex → [GeneratedRegex] + timeout
│   └── Temp buffer → stackalloc
│
└── Validation
    ├── Email → MailAddress class
    ├── URL → Uri.TryCreate
    ├── Phone / structured → custom logic + regex
    ├── Free text → IsNullOrWhiteSpace + length checks
    └── Complex → FluentValidation + regex
```

---

## 17. Cheat sheet

```csharp
// === Создание ===
string s1 = "hello";                          // literal
string s2 = "";                                // empty
string s3 = string.Empty;                      // эквивалент
string s4 = @"C:\path\to\file";                // verbatim
string s5 = """multi-line""";                  // raw (C# 11+)
string s6 = $"Hello, {name}!";                 // interpolation
string s7 = $$"""json {{ val }} value""";       // raw + interp (C# 11+)
ReadOnlySpan<byte> bytes = "hello"u8;          // UTF-8 literal

// === Concat ===
string a = "hello";
string b = a + " " + "world";                   // 2-5 частей OK
string c = string.Concat("a", "b", "c");
string d = string.Join(", ", parts);            // collection
var sb = new StringBuilder(capacity: 1024);
sb.Append("a").Append("b");                     // в loop
string final = sb.ToString();

// === Comparison ===
"a".Equals("A", StringComparison.OrdinalIgnoreCase);
string.Compare(a, b, StringComparison.Ordinal);
var dict = new Dictionary<string, int>(StringComparer.OrdinalIgnoreCase);

// === Search ===
s.IndexOf('x');                                 // -1 if not found
s.Contains("foo", StringComparison.Ordinal);
s.StartsWith("pre");
s.Substring(7, 5);                              // или s[7..12]
s.Split(',', StringSplitOptions.TrimEntries);
s.Replace("old", "new");
s.Trim();

// === Format ===
$"Price: {1234.5:F2}"                           // 1234.50
$"{date:yyyy-MM-dd}"                            // ISO
1234.5m.ToString("F2", CultureInfo.InvariantCulture);

// === Encoding ===
byte[] utf8 = Encoding.UTF8.GetBytes(s);
string s = Encoding.UTF8.GetString(utf8);
File.WriteAllText(path, text, Encoding.UTF8);

// === Span ===
ReadOnlySpan<char> span = s.AsSpan();
span[7..];                                      // slice
span.Equals("hello", StringComparison.Ordinal);
int.Parse(span);                                // .NET 8+

// === StringBuilder ===
var sb = new StringBuilder(1024);
sb.Append("...").AppendLine().AppendFormat("{0:F2}", 3.14);
string result = sb.ToString();
sb.Clear();   // re-use

// === Regex ===
[GeneratedRegex(@"\d+")]
private static partial Regex DigitRegex();

DigitRegex().IsMatch("hello 42");
DigitRegex().Match("hello 42").Value;           // "42"
DigitRegex().Matches("a1 b22").Select(m => m.Value);
Regex.Replace("hello 42", @"\d+", "X");          // "hello X"

// Regex с timeout (для user input)
new Regex(pattern, RegexOptions.None, TimeSpan.FromSeconds(1));

// === Validation ===
string.IsNullOrWhiteSpace(input);
new MailAddress(s).Address == s;                 // email
Uri.TryCreate(s, UriKind.Absolute, out var uri); // URL
```

| Сценарий | Решение |
|----------|---------|
| Concat в loop | StringBuilder |
| Concat 2-5 в expression | `+` или interpolation |
| Compare in backend | `StringComparison.Ordinal` |
| Compare ignore case | `OrdinalIgnoreCase` (не ToLower) |
| Compare for UI sort | `CurrentCulture` |
| Format for wire | `CultureInfo.InvariantCulture` |
| Email validation | `MailAddress` class |
| URL validation | `Uri.TryCreate` |
| Multi-line raw text | `"""..."""` (C# 11+) |
| Path / regex pattern | `@"..."` |
| Reused regex | `[GeneratedRegex]` (.NET 7+) |
| Parsing hot path | `Span<char>` |
| User input regex | + `TimeSpan` timeout |
| Encoding for files | `Encoding.UTF8` без BOM |
| Iteration with emojis | `Rune.EnumerateRunes()` |

---

## 18. Common Pitfalls — с механизмами

### 18.1. `+=` в loop

```csharp
string result = "";
for (int i = 0; i < 10000; i++)
    result += i;   // ❌ O(N²)
```

**Механизм:** каждое `+=` создаёт новую string, копирует весь предыдущий контент.

**Фикс:** StringBuilder.

### 18.2. Compare через ToLower

```csharp
if (a.ToLower() == b.ToLower()) { }   // ❌ две лишних allocation
```

**Механизм:** `ToLower()` создаёт новые строки.

**Фикс:** `string.Equals(a, b, StringComparison.OrdinalIgnoreCase)`.

### 18.3. `==` без StringComparison

```csharp
if (s1 == s2) { }   // CA1307 warning — implicit Ordinal, но непонятно намерение
```

**Механизм:** default Ordinal, но не явно. На разных версиях BCL могут быть разные defaults для overload resolution.

**Фикс:** `string.Equals(s1, s2, StringComparison.Ordinal)`.

### 18.4. Parse без InvariantCulture

```csharp
decimal d = decimal.Parse("1234.50");   // ❌ зависит от current locale
// На ru-RU — throws (запятая ожидается)
```

**Механизм:** Parse использует `CurrentCulture`, decimal separator различается.

**Фикс:** `decimal.Parse("1234.50", CultureInfo.InvariantCulture)`.

### 18.5. ToString без culture

```csharp
decimal price = 1234.5m;
string s = price.ToString("F2");   // "1234,50" в RU, "1234.50" в US
```

**Механизм:** ToString с current culture — locale-dependent.

**Фикс:** `price.ToString("F2", CultureInfo.InvariantCulture)`.

### 18.6. Length для emoji

```csharp
"😀".Length;   // 2! (surrogate pair, не 1)
```

**Механизм:** UTF-16 surrogate pair = 2 code unit для одного character.

**Фикс:** `"😀".EnumerateRunes().Count()` → 1.

### 18.7. UTF-8 BOM в files

```csharp
File.WriteAllText("data.txt", text, Encoding.UTF8);   // adds BOM!
```

**Механизм:** `Encoding.UTF8` имеет default `encoderShouldEmitUTF8Identifier = true`.

**Фикс:** `new UTF8Encoding(encoderShouldEmitUTF8Identifier: false)`.

### 18.8. Catastrophic backtracking

```csharp
new Regex(@"^(a+)+$").IsMatch("aaaaaaaaaaaaab");   // hangs!
```

**Механизм:** nested quantifiers create exponential backtracking.

**Фикс:** simpler pattern `^a+$`, или timeout, или `[GeneratedRegex]`.

### 18.9. Regex на user input без timeout

```csharp
public bool Validate(string input) =>
    Regex.IsMatch(input, somePattern);   // ❌ может зависнуть
```

**Механизм:** ReDoS attack возможен на vulnerable patterns.

**Фикс:** `new Regex(pattern, RegexOptions.None, TimeSpan.FromSeconds(1))`.

### 18.10. Substring в hot path

```csharp
foreach (var line in millionLines)
{
    var parts = line.Split(',');   // N allocations per line
    var first = parts[0];           // Copy
    Process(first);
}
```

**Механизм:** Split аллоцирует array + N substrings. Million lines × N = много allocations.

**Фикс:** Span-based parsing.

```csharp
foreach (var line in millionLines)
{
    var lineSpan = line.AsSpan();
    int comma = lineSpan.IndexOf(',');
    var first = lineSpan[..comma];
    Process(first);   // accepts ReadOnlySpan<char>
}
```

> [!question]- Интервью: топ-3 ловушки strings в C# для junior?
> 1) **`+=` в loop** — O(N²) из-за immutability. Каждое присвоение создаёт новую строку, копирует все предыдущие. Фикс: StringBuilder. 2) **Comparison без StringComparison** — `s1 == s2` использует Ordinal (default), но `String.Compare(s1, s2)` без enum — CurrentCulture, что unstable между locales. Фикс: всегда явно указывай StringComparison. 3) **Parse / ToString без InvariantCulture** — на разных locales разные decimal/date separators, format ломается. Фикс: для wire format — `CultureInfo.InvariantCulture`, для UI — `CurrentCulture`.

---

## 19. Practice — упражнения с разбором

### 19.1. CSV parser без allocations

**Задача.** Парсить CSV строки используя `Span<char>` без allocations per line.

```csharp
public static IEnumerable<(int Id, string Name, decimal Price)> ParseCsvFile(string path)
{
    foreach (var line in File.ReadLines(path))
    {
        if (string.IsNullOrWhiteSpace(line)) continue;
        var parsed = ParseCsvLine(line);
        if (parsed.HasValue) yield return parsed.Value;
    }
}

public static (int Id, string Name, decimal Price)? ParseCsvLine(string line)
{
    ReadOnlySpan<char> span = line.AsSpan();

    // Field 1: Id (int)
    int comma1 = span.IndexOf(',');
    if (comma1 < 0) return null;
    if (!int.TryParse(span[..comma1], out int id)) return null;
    span = span[(comma1 + 1)..];

    // Field 2: Name (string — здесь придётся allocate, иначе lifetime issue)
    int comma2 = span.IndexOf(',');
    if (comma2 < 0) return null;
    string name = span[..comma2].Trim().ToString();
    span = span[(comma2 + 1)..];

    // Field 3: Price (decimal)
    if (!decimal.TryParse(span.Trim(), NumberStyles.Any, CultureInfo.InvariantCulture, out decimal price))
        return null;

    return (id, name, price);
}

// Использование
foreach (var (id, name, price) in ParseCsvFile("products.csv"))
    Console.WriteLine($"{id}: {name} = ${price:F2}");
```

**Разбор:** Span-based parsing — `IndexOf`, slice через `[start..end]`, `int.Parse(span)` без string allocation для numeric. Name требует allocation потому что мы возвращаем tuple через yield (Span не может cross await/yield). Если бы обрабатывали без возврата — Name тоже как Span.

### 19.2. Smart string builder для logs

**Задача.** Logger который accumulates messages с rich formatting, использует StringBuilder + interpolation correctly.

```csharp
public class StructuredLogger
{
    private readonly StringBuilder _buffer = new(capacity: 4096);
    private readonly object _lock = new();
    
    public void Log(LogLevel level, string message, params (string Key, object Value)[] context)
    {
        lock (_lock)
        {
            _buffer.Append('[')
                   .Append(DateTimeOffset.UtcNow.ToString("o", CultureInfo.InvariantCulture))
                   .Append("] [")
                   .Append(level)
                   .Append("] ")
                   .Append(message);
            
            if (context.Length > 0)
            {
                _buffer.Append(" {");
                bool first = true;
                foreach (var (key, value) in context)
                {
                    if (!first) _buffer.Append(", ");
                    _buffer.Append(key).Append('=').Append(value);
                    first = false;
                }
                _buffer.Append('}');
            }
            
            _buffer.AppendLine();
        }
    }
    
    public string Flush()
    {
        lock (_lock)
        {
            var result = _buffer.ToString();
            _buffer.Clear();
            return result;
        }
    }
}

// Использование
var logger = new StructuredLogger();
logger.Log(LogLevel.Info, "User logged in", ("userId", 42), ("ip", "192.168.1.1"));
logger.Log(LogLevel.Warning, "Slow query", ("duration_ms", 1500));

string allLogs = logger.Flush();
```

**Разбор:** StringBuilder с pre-allocate capacity. Reuse через `Clear()` после Flush. `InvariantCulture` для timestamp (wire format). Fluent chain через `.Append()`.

### 19.3. Email + URL validator

**Задача.** Валидаторы для email и URL с правильным error handling.

```csharp
using System.Net.Mail;

public static class Validators
{
    public static bool IsValidEmail(string? input)
    {
        if (string.IsNullOrWhiteSpace(input)) return false;
        
        try
        {
            var addr = new MailAddress(input);
            return addr.Address.Equals(input.Trim(), StringComparison.OrdinalIgnoreCase);
        }
        catch (FormatException)
        {
            return false;
        }
    }
    
    public static bool IsValidUrl(string? input, out Uri? url)
    {
        url = null;
        if (string.IsNullOrWhiteSpace(input)) return false;
        
        return Uri.TryCreate(input, UriKind.Absolute, out url) &&
               (url.Scheme == Uri.UriSchemeHttp || url.Scheme == Uri.UriSchemeHttps);
    }
    
    [GeneratedRegex(
        @"^\+?[1-9]\d{1,14}$",   // E.164 format
        RegexOptions.None,
        matchTimeoutMilliseconds: 1000)]
    private static partial Regex E164PhoneRegex();
    
    public static bool IsValidPhoneE164(string? input)
    {
        if (string.IsNullOrWhiteSpace(input)) return false;
        return E164PhoneRegex().IsMatch(input.Trim());
    }
}

// Использование
Validators.IsValidEmail("user@example.com");           // true
Validators.IsValidEmail("not-an-email");                // false
Validators.IsValidEmail(null);                          // false

if (Validators.IsValidUrl("https://api.example.com", out var url))
    Console.WriteLine(url.Host);   // "api.example.com"

Validators.IsValidPhoneE164("+15551234567");   // true
Validators.IsValidPhoneE164("555-1234");        // false (not E.164)
```

**Разбор:** для email — `MailAddress` class (real RFC validation). Для URL — `Uri.TryCreate` + scheme check. Phone — `[GeneratedRegex]` с `matchTimeoutMilliseconds` (защита от ReDoS). Всё в `null`/whitespace safe формате — `IsNullOrWhiteSpace`.

### 19.4. String comparison Dictionary

**Задача.** Cache user preferences с case-insensitive lookup и unicode-correct.

```csharp
public class UserPreferences
{
    private readonly Dictionary<string, string> _prefs;
    
    public UserPreferences()
    {
        // OrdinalIgnoreCase — backend, fast, не зависит от locale
        _prefs = new Dictionary<string, string>(StringComparer.OrdinalIgnoreCase);
    }
    
    public void Set(string key, string value)
    {
        if (string.IsNullOrWhiteSpace(key))
            throw new ArgumentException("Key required", nameof(key));
        
        _prefs[key.Trim()] = value;
    }
    
    public string? Get(string key) =>
        string.IsNullOrWhiteSpace(key) ? null :
        _prefs.TryGetValue(key.Trim(), out var value) ? value : null;
    
    public bool Has(string key) =>
        !string.IsNullOrWhiteSpace(key) && _prefs.ContainsKey(key.Trim());
    
    public IEnumerable<KeyValuePair<string, string>> All =>
        _prefs.OrderBy(kv => kv.Key, StringComparer.OrdinalIgnoreCase);
}

// Использование
var prefs = new UserPreferences();
prefs.Set("theme", "dark");
prefs.Set("Language", "en-US");

prefs.Get("THEME");        // "dark" — case-insensitive
prefs.Get("language");     // "en-US"
prefs.Has("missing");      // false
```

**Разбор:** `StringComparer.OrdinalIgnoreCase` в Dictionary constructor — все lookups case-insensitive, fast (Ordinal hash). `Trim()` для consistency. Validation через `IsNullOrWhiteSpace`. Sorting через `OrderBy` с тем же comparer.

### 19.5. Regex source generator для log parsing

**Задача.** Парсить лог-строки формата `[timestamp] [level] message {key=value, key2=value2}`.

```csharp
public partial class LogParser
{
    [GeneratedRegex(
        @"^\[(?<timestamp>[^\]]+)\]\s+\[(?<level>\w+)\]\s+(?<message>[^{]+?)(?:\s+\{(?<context>[^}]+)\})?$",
        RegexOptions.None,
        matchTimeoutMilliseconds: 1000)]
    private static partial Regex LineRegex();
    
    [GeneratedRegex(@"(?<key>\w+)=(?<value>[^,]+)", RegexOptions.None, matchTimeoutMilliseconds: 1000)]
    private static partial Regex ContextRegex();
    
    public static LogEntry? Parse(string line)
    {
        var match = LineRegex().Match(line);
        if (!match.Success) return null;
        
        var entry = new LogEntry(
            DateTimeOffset.Parse(match.Groups["timestamp"].Value, CultureInfo.InvariantCulture),
            Enum.Parse<LogLevel>(match.Groups["level"].Value, ignoreCase: true),
            match.Groups["message"].Value.Trim()
        );
        
        if (match.Groups["context"].Success)
        {
            var contextMatches = ContextRegex().Matches(match.Groups["context"].Value);
            foreach (Match m in contextMatches)
            {
                entry.Context[m.Groups["key"].Value] = m.Groups["value"].Value.Trim();
            }
        }
        
        return entry;
    }
}

public record LogEntry(DateTimeOffset Timestamp, LogLevel Level, string Message)
{
    public Dictionary<string, string> Context { get; } = new();
}

public enum LogLevel { Trace, Debug, Info, Warning, Error, Critical }

// Использование
var line = "[2024-05-15T10:30:45.123Z] [Info] User logged in {userId=42, ip=192.168.1.1}";
var entry = LogParser.Parse(line);

Console.WriteLine(entry?.Message);                // "User logged in"
Console.WriteLine(entry?.Context["userId"]);      // "42"
Console.WriteLine(entry?.Timestamp);              // 2024-05-15 10:30:45 +00:00
```

**Разбор:** `[GeneratedRegex]` для compile-time parsing. Named groups (`?<name>`) делают код read-able. `matchTimeoutMilliseconds` защита от ReDoS. Two passes: first regex для line structure, second для key=value pairs внутри context. `InvariantCulture` для timestamp parsing.

---

## 20. Что читать дальше — порядок и почему

1. **[[csharp-basics|C# Basics]]** — типы, переменные, value vs reference.
2. **[[datetime-timezones|DateTime и Timezones]]** — formatting и parsing с culture.
3. **[[anonymous-types|Anonymous Types]]** — для LINQ projections.
4. **[[collections-linq|Collections и LINQ]]** — обработка коллекций строк.
5. **System.IO Streams** — file I/O с UTF-8.
6. **JSON Serialization** — strings в API contracts.
7. **Regex deep dive** — advanced patterns, performance.
8. **Globalization / i18n** — culture-aware UI.

---

## 21. См. также

- [[csharp-basics|C# Basics]] — основа
- [[datetime-timezones|DateTime]] — formatting и parsing
- [[anonymous-types|Anonymous Types]] — LINQ projections
- [[collections-linq|Collections и LINQ]] — обработка коллекций
- [[debugging-basics|Debugging]] — как смотреть строки в debugger
- [[error-handling|Error Handling]] — exceptions для validation
- ICU library / globalization
- HtmlAgilityPack — парсинг HTML вместо regex
- AngleSharp — modern HTML/CSS parser
- CsvHelper — парсинг CSV с типобезопасностью

---

## 22. Reading list

- **Microsoft Docs — String class** — learn.microsoft.com/dotnet/api/system.string
- **Microsoft Docs — String operations** — learn.microsoft.com/dotnet/standard/base-types/best-practices-strings
- **Microsoft Docs — Comparing strings** — learn.microsoft.com/dotnet/standard/base-types/comparing
- **Microsoft Docs — String formatting** — learn.microsoft.com/dotnet/standard/base-types/composite-formatting
- **Microsoft Docs — StringBuilder** — learn.microsoft.com/dotnet/api/system.text.stringbuilder
- **Microsoft Docs — Regular expressions** — learn.microsoft.com/dotnet/standard/base-types/regular-expressions
- **Microsoft Docs — GeneratedRegex** — learn.microsoft.com/dotnet/standard/base-types/regular-expression-source-generators
- **Microsoft Docs — `Span<T>`** — learn.microsoft.com/dotnet/api/system.span-1
- **Microsoft Docs — Encoding** — learn.microsoft.com/dotnet/api/system.text.encoding
- **Stephen Toub — Performance improvements .NET 8 strings** — devblogs.microsoft.com/dotnet
- **Bart de Smet — Optimizing string performance** — bartdesmet.net
- **Andrew Lock — String interning patterns** — andrewlock.net
- **Jon Skeet — C# in Depth (chapter on strings)**
- **Mastering Regular Expressions (Jeffrey Friedl, 3rd ed.)** — стандартная книга по regex
- **regex101.com** — interactive regex tester
- **regexr.com** — alternative regex playground
- **Adam Sitnik — Performance benchmarks** — adamsitnik.com
- **ReDoS attack catalog** — owasp.org/www-community/attacks/Regular_expression_Denial_of_Service
