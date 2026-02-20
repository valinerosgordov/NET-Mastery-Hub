# Async и Concurrency

## Task, Task.WhenAll, Task.Run

Task.WhenAll — параллельное выполнение, ожидание всех. Запуск до await — задачи идут параллельно. Task.WhenAny — первый завершённый.

Task.Run — обёртка над ThreadPool. Task.Factory.StartNew — больше опций (LongRunning). Для async — Task.Run.

---

## Async Streams, lock для async

IAsyncEnumerable<T>, await foreach. Стриминг без загрузки всего в память.

lock — только sync. SemaphoreSlim.WaitAsync() для async. Channel — producer-consumer без блокировок.

---

## Thread, ThreadPool

Thread, ThreadPool.QueueUserWorkItem, Task.Run, BackgroundService. Task.Run — предпочтительно для фоновой работы.

---

## Interlocked, volatile, Events

Interlocked — атомарные операции (Increment, CompareExchange). volatile — visibility, не атомарность. Для i++ — Interlocked.Increment.

AutoResetEvent — один проходит, сброс. ManualResetEvent — все проходят при Signal.

---

## Channel, GC

Channel<T> — async producer-consumer. Bounded/Unbounded. Thread-safe.

GC: Gen0, Gen1, Gen2. Большинство объектов умирают в Gen0. LOH — объекты > 85 KB.
