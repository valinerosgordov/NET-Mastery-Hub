# Runtime — CLR internals

> 9 файлов / ~310 KB. Что под капотом .NET: GC, JIT, memory model, threading, P/Invoke, diagnostics.

[[README|← Главный README]] · [[INDEX|Полный INDEX]]

---

## 🎯 Где начать

| Кто ты | С чего начать |
|--------|---------------|
| Junior, что вообще такое CLR | [[runtime-basics|`Junior/runtime-basics.md`]] → [[memory-stack-heap|`Junior/memory-stack-heap.md`]] |
| Middle, не понимаю threads | [[threading-basics|`Middle/threading-basics.md`]] |
| Memory leak в production | [[gc-memory|`Senior/gc-memory.md`]] → [[diagnostics-tools|`Senior/diagnostics-tools.md`]] |
| Senior interview prep | [[gc-memory|`Senior/gc-memory.md`]], [[compilation-jit|`Senior/compilation-jit.md`]], [[span-layout|`Senior/span-layout.md`]] |
| Performance hot path | [[span-layout|`Senior/span-layout.md`]], [[concurrency-atomics|`Senior/concurrency-atomics.md`]] |

---

## 📚 Все 9 файлов

### 🌱 Junior

| Файл | Описание |
|------|----------|
| [[runtime-basics|`runtime-basics.md`]] | CLR, IL, JIT — картина мира для старта |
| [[memory-stack-heap|`memory-stack-heap.md`]] | Stack vs heap, кто где живёт |

### 🌿 Middle

| Файл | Описание |
|------|----------|
| [[threading-basics|`threading-basics.md`]] | Thread, ThreadPool, TPL, Parallel |

### 🏆 Senior

| Файл | Описание |
|------|----------|
| [[gc-memory|`gc-memory.md`]] | GC generations, regions, DATAS, leaks ⭐ |
| [[span-layout|`span-layout.md`]] | Span\<T\>, Memory\<T\>, ref struct, layout |
| [[compilation-jit|`compilation-jit.md`]] | JIT, AOT, ReadyToRun, tiered compilation |
| [[concurrency-atomics|`concurrency-atomics.md`]] | Memory model, Interlocked, lock-free |
| [[interop-pinvoke|`interop-pinvoke.md`]] | P/Invoke, marshalling, native interop |
| [[diagnostics-tools|`diagnostics-tools.md`]] | dotnet-counters, trace, dump, PerfView |

---

## 🔗 Связанные папки

- [[types-and-memory|`CSharp/Senior/types-and-memory.md`]] — value vs reference (extends GC topic)
- [[async-threading|`CSharp/Senior/async-threading.md`]] — async поверх threading
- [`Performance/`](../Performance/) — практика performance
