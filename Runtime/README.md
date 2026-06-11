# Runtime — CLR internals

> 7 файлов / 246 KB. Что под капотом .NET: GC, JIT, threading, memory layout, P/Invoke, diagnostics.

[← Главный README]() · [Полный INDEX]()

---

## 🎯 Где начать

| Кто ты | С чего начать |
|--------|---------------|
| Middle, не понимаю threads | [`threading-basics.md`]() |
| Memory leak в production | [`gc-memory.md`](gc-memory.md) → [`diagnostics-tools.md`]() |
| Senior interview prep | [`gc-memory.md`](gc-memory.md), [`compilation-jit.md`](compilation-jit.md), [`span-layout.md`](span-layout.md) |
| Performance hot path | [`span-layout.md`](span-layout.md), [`concurrency-atomics.md`](concurrency-atomics.md) |

---

## 📚 Все 7 файлов

### Memory

| Файл | Описание |
|------|----------|
| [`gc-memory.md`](gc-memory.md) | GC generations, regions, leaks (56 KB) ⭐ |
| [`span-layout.md`](span-layout.md) | Span\<T\>, Memory\<T\>, ref struct |

### Compilation

| Файл | Описание |
|------|----------|
| [`compilation-jit.md`](compilation-jit.md) | JIT, AOT, ReadyToRun, tiered compilation |

### Threading & concurrency

| Файл | Описание |
|------|----------|
| [`threading-basics.md`]() | Thread, ThreadPool, TPL, Parallel (Middle entry) |
| [`concurrency-atomics.md`](concurrency-atomics.md) | Memory model, Interlocked, lock-free |

### Native interop

| Файл | Описание |
|------|----------|
| [`interop-pinvoke.md`]() | P/Invoke, marshalling, native interop |

### Diagnostics

| Файл | Описание |
|------|----------|
| [`diagnostics-tools.md`]() | dotnet-counters, trace, dump, PerfView |

---

## 🔗 Связанные папки

- [`CSharp/types-and-memory`]() — value vs reference (extends GC topic)
- [`CSharp/async-threading`]() — async поверх threading
- [`Performance/`](../Performance/) — практика performance
