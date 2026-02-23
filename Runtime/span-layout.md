---
tags:
  - span
  - memory
  - performance
  - zero-allocation
  - struct-layout
  - deepdive
complexity: Senior
date: 2026-02-23
---

# Zero-allocation: Span, Memory и Layout

## 1. Span&lt;T&gt; — окно в память без аллокаций

### Что это

`Span<T>` — **ссылка на непрерывный участок памяти** любого происхождения: массив, стек, native. Без копирования, без аллокаций.

### Внутреннее устройство

```csharp
// Упрощённая структура Span<T>
public readonly ref struct Span<T>
{
    internal readonly ref T _reference;  // ByReference<T> — managed pointer
    private readonly int _length;

    // Всего 2 поля: указатель + длина = 16 bytes на x64
}
```

```
Массив в памяти:
[Header][Length][ 0 ][ 1 ][ 2 ][ 3 ][ 4 ][ 5 ][ 6 ][ 7 ]

Span<int> span = array.AsSpan(2, 4):
                        ↓ _reference          ↓ _length = 4
                  [ 0 ][ 1 ][ 2 ][ 3 ][ 4 ][ 5 ][ 6 ][ 7 ]
                              ^^^^^^^^^^^^^^^^^^^
                              span видит только это
```

### Почему ref struct

`Span<T>` содержит **managed pointer** (`ref T`) — ссылку, которая может указывать на стек. Если Span попадёт на heap (поле класса, boxing), GC не сможет отследить этот указатель → memory corruption.

| Ограничение ref struct | Причина |
|------------------------|---------|
| Не может быть полем класса | Класс на heap, ref на stack → dangling pointer |
| Не может быть в async/await | State machine — класс на heap |
| Не может быть в замыкании | Замыкание — объект на heap |
| Не может быть boxed | Boxing → heap |
| Не может быть в `IEnumerable` | yield → state machine на heap |

### Span vs ReadOnlySpan

```csharp
// Span<T> — чтение и запись
Span<byte> writable = stackalloc byte[256];
writable[0] = 0xFF;

// ReadOnlySpan<T> — только чтение (безопасность)
ReadOnlySpan<char> text = "Hello, World!".AsSpan();
ReadOnlySpan<char> hello = text[..5];  // слайс без аллокации строки

// string → ReadOnlySpan<char> неявно
void Process(ReadOnlySpan<char> data) { }
Process("some text");  // без аллокации
```

### Парсинг без аллокаций

```csharp
// Кейс: парсинг бинарного протокола
// Формат: [4 bytes length][N bytes payload][2 bytes checksum]
public static (ReadOnlySpan<byte> payload, ushort checksum) ParsePacket(
    ReadOnlySpan<byte> raw)
{
    int length = BinaryPrimitives.ReadInt32LittleEndian(raw);
    var payload = raw.Slice(4, length);
    ushort checksum = BinaryPrimitives.ReadUInt16LittleEndian(raw[(4 + length)..]);
    return (payload, checksum);
}

// Парсинг CSV строки без string.Split (zero allocation)
public static void ParseCsvLine(ReadOnlySpan<char> line)
{
    while (!line.IsEmpty)
    {
        int comma = line.IndexOf(',');
        ReadOnlySpan<char> field = comma >= 0 ? line[..comma] : line;

        // Обработка field — без аллокации строк
        if (int.TryParse(field, out int value))
            ProcessValue(value);

        line = comma >= 0 ? line[(comma + 1)..] : ReadOnlySpan<char>.Empty;
    }
}
```

> [!warning] Span и async — несовместимы
> `Span<T>` нельзя использовать в `async` методах. Решение: `Memory<T>` (см. ниже) или выделить синхронную часть работы с Span в отдельный метод.

---

## 2. Memory&lt;T&gt; — Span для async мира

### Когда Span недостаточно

`Memory<T>` — **обёртка**, которая может жить на heap. Можно передавать в async методы, хранить в полях.

```csharp
// Memory<T> внутри — OwnedMemory или ArraySegment
public readonly struct Memory<T>
{
    private readonly object? _object;  // массив или MemoryManager<T>
    private readonly int _index;
    private readonly int _length;

    public Span<T> Span => /* создаёт Span из _object + _index + _length */;
}
```

### Использование в async

```csharp
public async Task ProcessAsync(Memory<byte> buffer, CancellationToken ct)
{
    // Memory<T> можно передавать в async — он на heap
    int bytesRead = await stream.ReadAsync(buffer, ct);

    // Для обработки — получаем Span (только в sync контексте)
    ProcessData(buffer.Span[..bytesRead]);
}

private void ProcessData(Span<byte> data)
{
    // Работаем со Span — zero allocation
    BinaryPrimitives.ReadInt32LittleEndian(data);
}
```

### Span vs Memory — когда что

| Критерий | `Span<T>` | `Memory<T>` |
|----------|-----------|-------------|
| Аллокации | Zero | Zero (обёртка) |
| async/await | Нельзя | Можно |
| Поле класса | Нельзя | Можно |
| Производительность | Максимум | Чуть медленнее (.Span) |
| stackalloc | Да | Нет |
| Источники | Array, stack, native | Array, MemoryManager |

> [!info] Правило
> Используй `Span<T>` в синхронном коде для максимальной производительности. `Memory<T>` — когда нужно хранить или передавать в async.

---

## 3. stackalloc — память на стеке

```csharp
// Аллокация на стеке — мгновенно, без GC, автоматически освобождается
Span<byte> buffer = stackalloc byte[256];

// Порог безопасности — не больше ~1KB на стеке
const int StackThreshold = 1024;

Span<byte> buf = size <= StackThreshold
    ? stackalloc byte[size]       // стек — быстро
    : new byte[size];             // heap — безопасно

// stackalloc + инициализация
Span<int> ids = stackalloc int[] { 1, 2, 3, 4, 5 };
```

| Характеристика | `stackalloc` | `new T[]` | `ArrayPool.Rent` |
|----------------|-------------|-----------|-------------------|
| Где | Stack | Heap (Gen 0) | Heap (pooled) |
| Скорость | Мгновенно (mov rsp) | Быстро | Мгновенно (reuse) |
| GC pressure | Нет | Да | Нет |
| Размер | ~1 KB безопасно | Неограничен | Неограничен |
| Освобождение | Автоматически | GC | Manual (Return) |

> [!warning] Stack Overflow
> Стек потока по умолчанию ~1 MB. `stackalloc byte[1_000_000]` → StackOverflowException. Всегда проверять размер и fallback на heap.

---

## 4. Data Alignment и Padding

### Как процессор читает память

Процессор читает память **словами** (8 bytes на x64). Если данные не выровнены по границе своего размера, процессор делает два чтения вместо одного.

**Правило выравнивания:**

$$Offset \% Size == 0$$

где $Offset$ — адрес поля, $Size$ — размер типа.

```
Пример: int (4 bytes) должен начинаться по адресу, кратному 4:
  Адрес 0x00 → ✓ (0 % 4 == 0)
  Адрес 0x04 → ✓ (4 % 4 == 0)
  Адрес 0x03 → ✗ (3 % 4 != 0) → неэффективно или невозможно
```

### Padding в структурах

```csharp
// Без оптимизации — CLR добавляет padding для выравнивания
struct BadLayout  // Ожидаем 13 bytes, реально 24 bytes!
{
    byte a;       // offset 0, size 1
    // [padding 7 bytes] — выравнивание long до 8
    long b;       // offset 8, size 8
    byte c;       // offset 16, size 1
    // [padding 3 bytes] — выравнивание до 4 (alignment struct)
    int d;        // offset 20, size 4
}
// Total: 24 bytes (11 bytes — padding!)

// Оптимизированный порядок полей
struct GoodLayout  // 16 bytes (вместо 24!)
{
    long b;       // offset 0, size 8 (самый большой первым)
    int d;        // offset 8, size 4
    byte a;       // offset 12, size 1
    byte c;       // offset 13, size 1
    // [padding 2 bytes]
}
// Total: 16 bytes
```

```
BadLayout (24 bytes):
| a | pad | pad | pad | pad | pad | pad | pad |  ← 8 bytes
| b  b  b  b  b  b  b  b |                      ← 8 bytes
| c | pad | pad | pad | d  d  d  d |              ← 8 bytes

GoodLayout (16 bytes):
| b  b  b  b  b  b  b  b |                      ← 8 bytes
| d  d  d  d | a | c | pad | pad |              ← 8 bytes
```

> [!info] Правило: располагай поля от большего к меньшему
> `long` → `int` → `short` → `byte`. Минимизирует padding. Для hot structs (в массивах, tight loops) — разница в производительности значительная из-за cache line utilization.

---

## 5. StructLayout и FieldOffset

### Explicit Layout

```csharp
// Union — несколько полей на одном адресе
[StructLayout(LayoutKind.Explicit)]
struct Packet
{
    [FieldOffset(0)] public int RawValue;     // 4 bytes
    [FieldOffset(0)] public byte Byte0;       // перекрывает RawValue
    [FieldOffset(1)] public byte Byte1;
    [FieldOffset(2)] public byte Byte2;
    [FieldOffset(3)] public byte Byte3;
}

// Использование — разбор int по байтам без BitConverter
var p = new Packet { RawValue = 0xAABBCCDD };
Console.WriteLine($"0x{p.Byte0:X2}"); // 0xDD (little-endian)
Console.WriteLine($"0x{p.Byte3:X2}"); // 0xAA
```

### Sequential Layout для interop

```csharp
// Точный контроль размера для P/Invoke
[StructLayout(LayoutKind.Sequential, Pack = 1)] // Pack=1 — без padding
struct NetworkHeader
{
    public byte Version;     // offset 0
    public byte Type;        // offset 1
    public ushort Length;    // offset 2
    public uint SequenceId;  // offset 4
}
// Sizeof = 8 bytes (без padding благодаря Pack = 1)

// Прямое чтение из буфера
ReadOnlySpan<byte> raw = socket.ReceiveBuffer;
var header = MemoryMarshal.Read<NetworkHeader>(raw);
```

### Парсинг бинарного протокола — полный пример

```csharp
// Zero-allocation парсинг сетевого пакета
[StructLayout(LayoutKind.Sequential, Pack = 1)]
readonly struct TradeMessage
{
    public readonly long Timestamp;    // 8 bytes
    public readonly int InstrumentId;  // 4 bytes
    public readonly decimal Price;     // 16 bytes
    public readonly int Quantity;      // 4 bytes
    // Total: 32 bytes
}

public static TradeMessage ParseTrade(ReadOnlySpan<byte> buffer)
{
    // Прямое reinterpret cast — zero copy, zero allocation
    return MemoryMarshal.Read<TradeMessage>(buffer);
}

// Парсинг массива сообщений
public static ReadOnlySpan<TradeMessage> ParseBatch(ReadOnlySpan<byte> buffer)
{
    return MemoryMarshal.Cast<byte, TradeMessage>(buffer);
    // Весь буфер интерпретируется как массив TradeMessage
    // Без единой аллокации!
}
```

> [!warning] Endianness
> `MemoryMarshal.Read` не конвертирует byte order. Если протокол big-endian, а CPU little-endian — нужен `BinaryPrimitives.ReverseEndianness()` или `ReadInt32BigEndian`.

---

## 6. Unsafe и fixed

```csharp
// fixed — пиннинг объекта для получения указателя
byte[] managed = new byte[1024];
unsafe
{
    fixed (byte* ptr = managed)
    {
        // ptr валиден внутри fixed блока
        // GC не двигает managed на время fixed
        *(int*)ptr = 42; // записать int в начало массива
    }
}

// Unsafe.As — reinterpret cast без копирования
float f = 3.14f;
int bits = Unsafe.As<float, int>(ref f);
// bits = 0x4048F5C3 (IEEE 754 representation)

// Unsafe.Add — pointer arithmetic для Span
ref byte start = ref MemoryMarshal.GetReference(span);
ref byte element = ref Unsafe.Add(ref start, index);
```

---

## Cheat Sheet: выбор подхода

```
Нужна ли память?
  │
  ├── < 1KB, синхронный код → stackalloc + Span<T>
  │
  ├── Большой буфер, переиспользуемый → ArrayPool<T>.Shared.Rent()
  │
  ├── Async контекст → Memory<T>
  │
  ├── I/O буфер, pinned для ОС → GC.AllocateArray<T>(pinned: true)
  │
  ├── Interop с native → NativeMemory.Alloc + Span<T>
  │
  └── Обычный объект → new T[] (пусть GC разбирается)
```

---

## См. также

- [GC, LOH и POH](gc-memory.md)
- [Concurrency и атомарность](concurrency-atomics.md)
- [Performance](../Performance/performance.md)
- [Типы и память](../CSharp/types-and-memory.md)
