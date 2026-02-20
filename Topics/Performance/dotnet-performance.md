# .NET Performance

## Span и Memory

**Span&lt;T&gt;**, **ReadOnlySpan&lt;T&gt;** — срезы без копирования. Stack-only (ref struct) или heap (Memory&lt;T&gt;). Работа с буферами, парсинг без аллокаций.

```csharp
ReadOnlySpan<char> span = "hello".AsSpan();
var slice = span[1..4];  // "ell", без копирования
```

---

## ArrayPool

Пул массивов. Аренда вместо new byte[]. Возврат в пул — меньше давления на GC.

```csharp
var pool = ArrayPool<byte>.Shared;
var buffer = pool.Rent(1024);
try
{
    // use buffer
}
finally
{
    pool.Return(buffer);
}
```

---

## ValueTask

Для методов, которые часто завершаются синхронно. Меньше аллокаций чем Task. Не использовать с блокирующими операциями.

---

## BenchmarkDotNet

Бенчмарки с тёплым запуском, статистикой. [MemoryDiagnoser], [DisassemblyDiagnoser].

```csharp
[MemoryDiagnoser]
public class MyBenchmark
{
    [Benchmark]
    public int Method() => DoWork();
}
```

---

## Избегать

Boxing в hot path. LINQ в циклах. Лишние аллокации строк (Span, stackalloc). Client-side evaluation в EF.

---

## См. также

- [[Topics/SQL/sql-query-optimization|SQL Optimization]]
- [[Topics/Observability/opentelemetry-jaeger-seq|OpenTelemetry]]
- [[dotnet-knowledge-base|.NET Knowledge Base]]
