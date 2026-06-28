# Runtime — CLR internals

> 9 файлов / ~310 KB. Что под капотом .NET: GC, JIT, memory model, threading, P/Invoke, diagnostics.

[← Главный README](../readme.md) · [Полный INDEX](../INDEX.md)

---

## 🎯 Где начать

| Кто ты | С чего начать |
|--------|---------------|
| Junior, что вообще такое CLR | [`Junior/runtime-basics.md`](Junior/runtime-basics.md) → [`Junior/memory-stack-heap.md`](Junior/memory-stack-heap.md) |
| Middle, не понимаю threads | [`Middle/threading-basics.md`](Middle/threading-basics.md) |
| Memory leak в production | [`Senior/gc-memory.md`](Senior/gc-memory.md) → [`Senior/diagnostics-tools.md`](Senior/diagnostics-tools.md) |
| Senior interview prep | [`Senior/gc-memory.md`](Senior/gc-memory.md), [`Senior/compilation-jit.md`](Senior/compilation-jit.md), [`Senior/span-layout.md`](Senior/span-layout.md) |
| Performance hot path | [`Senior/span-layout.md`](Senior/span-layout.md), [`Senior/concurrency-atomics.md`](Senior/concurrency-atomics.md) |

---

## 📚 Все 9 файлов

### 🌱 Junior

| Файл | Описание |
|------|----------|
| [`runtime-basics.md`](Junior/runtime-basics.md) | CLR, IL, JIT — картина мира для старта |
| [`memory-stack-heap.md`](Junior/memory-stack-heap.md) | Stack vs heap, кто где живёт |

### 🌿 Middle

| Файл | Описание |
|------|----------|
| [`threading-basics.md`](Middle/threading-basics.md) | Thread, ThreadPool, TPL, Parallel |

### 🏆 Senior

| Файл | Описание |
|------|----------|
| [`gc-memory.md`](Senior/gc-memory.md) | GC generations, regions, DATAS, leaks ⭐ |
| [`span-layout.md`](Senior/span-layout.md) | Span\<T\>, Memory\<T\>, ref struct, layout |
| [`compilation-jit.md`](Senior/compilation-jit.md) | JIT, AOT, ReadyToRun, tiered compilation |
| [`concurrency-atomics.md`](Senior/concurrency-atomics.md) | Memory model, Interlocked, lock-free |
| [`interop-pinvoke.md`](Senior/interop-pinvoke.md) | P/Invoke, marshalling, native interop |
| [`diagnostics-tools.md`](Senior/diagnostics-tools.md) | dotnet-counters, trace, dump, PerfView |

---

## 🔗 Связанные папки

- [`CSharp/Senior/types-and-memory.md`](../CSharp/Senior/types-and-memory.md) — value vs reference (extends GC topic)
- [`CSharp/Senior/async-threading.md`](../CSharp/Senior/async-threading.md) — async поверх threading
- [`Performance/`](../Performance/) — практика performance
