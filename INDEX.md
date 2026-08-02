# 📑 INDEX — оглавление vault'а

> Автогенерируется скриптом `Scripts/generate_index.ps1` из frontmatter (`level:`) и tagline'ов файлов. **Не редактировать руками** — перегенерировать после изменений.
>
> **169 файлов / 6,1 MB** · обновлено 2026-08-02

---

## По папкам

### LearningPath — 10 файлов / 137 KB

| Файл | Уровень | KB | Описание |
|---|---|---:|---|
| [00_overview.md](LearningPath/00_overview.md) | All | 5,6 | Главная навигация по vault. Гайд "с чего начать" для разных уровней. Все ссылки ведут на актуальные файлы в... |
| [01_language-map.md](LearningPath/01_language-map.md) | All | 13 | Карта самого языка C# (без ASP.NET / EF / архитектуры). Зеркалит таксономию «Complete C# 2026 Cheat Sheet» ... |
| [02_junior-to-middle.md](LearningPath/02_junior-to-middle.md) | Junior to Middle | 11,3 | Roadmap для перехода из junior в middle. Ориентир: 3-6 месяцев active learning. Все ссылки ведут на актуаль... |
| [03_middle-to-senior.md](LearningPath/03_middle-to-senior.md) | Middle to Senior | 13,3 | Roadmap для Senior-уровня. Senior — не "знает всё", а думает системно и принимает trade-offs. Этот путь идё... |
| [04_interview-prep.md](LearningPath/04_interview-prep.md) | All | 24,8 | Подготовка к техническому собеседованию на .NET Backend позиции. От Junior до Senior. Что повторить, какие ... |
| [05_topics-by-priority.md](LearningPath/05_topics-by-priority.md) | All | 10 | Рейтинг тем по value vs effort. Если хочешь точечно прокачать — здесь приоритеты. |
| [09_senior-tips-cheatsheet.md](LearningPath/09_senior-tips-cheatsheet.md) | Senior | 14,2 | Концентрат Senior-практик: allocation-free паттерны (`Span<T>`, `ArrayPool<T>`, `FrozenDictionary`), async-... |
| [10_interview-behavioral.md](LearningPath/10_interview-behavioral.md) | Senior | 6,6 | Senior behavioral-вопросы и каркас ответов (STAR): прод-инциденты, решения при неполных данных, осознанный ... |
| [99_reading-list.md](LearningPath/99_reading-list.md) | All | 8,8 | Курируемый список внешних .NET-ресурсов для Senior-level чтения: блоги (antondevtips, Milan Jovanović), Tel... |
| [case-studies-top7.md](LearningPath/case-studies-top7.md) | Middle to Senior | 29,1 | Дополнение к топ-7 наиболее важным файлам vault'а: реальные production кейсы, какие проблемы возникают, как... |

### CSharp — 42 файлов / 2 528 KB

| Файл | Уровень | KB | Описание |
|---|---|---:|---|
| [Junior/anonymous-types.md](CSharp/Junior/anonymous-types.md) | Junior | 105,5 | `new { Name = "John", Age = 30 }` — типы, рождённые компилятором на лету, без явного `class` объявления. Вс... |
| [Junior/async-await-basics.md](CSharp/Junior/async-await-basics.md) | Junior | 45,3 | Вход в асинхронность для Junior. Здесь — рабочая ментальная модель: зачем async существует, что реально дел... |
| [Junior/collections-linq.md](CSharp/Junior/collections-linq.md) | Junior | 59,5 | Главные коллекции BCL и LINQ как универсальный язык pipeline'ов. `List<T>`, `Dictionary<TKey,TValue>`, `Has... |
| [Junior/csharp-basics.md](CSharp/Junior/csharp-basics.md) | Junior | 88,9 | Стартовая точка для Junior. Не справочник синтаксиса — учебник с акцентом на «почему C# устроен именно так»... |
| [Junior/datetime-timezones.md](CSharp/Junior/datetime-timezones.md) | Junior | 105,1 | Самая ошибочная тема в backend-коде. `DateTime`, `DateTimeOffset`, `DateOnly`, `TimeOnly`, IANA vs Windows ... |
| [Junior/debugging-basics.md](CSharp/Junior/debugging-basics.md) | Junior | 117,1 | Как находить баги в .NET-коде быстро. Breakpoints (все виды), stepping, inspect state, call stack, exceptio... |
| [Junior/dotnet-cli-getting-started.md](CSharp/Junior/dotnet-cli-getting-started.md) | Junior | 114,8 | Кросс-платформенный командный интерфейс к .NET. `dotnet` — единственный обязательный инструмент: создание п... |
| [Junior/enums-flags.md](CSharp/Junior/enums-flags.md) | Junior | 102,5 | Типобезопасные константы вместо magic numbers и строк. `enum`, `[Flags]`, bitwise операции, конверсии в стр... |
| [Junior/extension-methods.md](CSharp/Junior/extension-methods.md) | Junior | 106,7 | `obj.MyMethod()` для типов, которые ты не контролируешь. Под капотом — обычные статические методы. Сахарный... |
| [Junior/iterators-yield.md](CSharp/Junior/iterators-yield.md) | Junior | 101,6 | Метод, который возвращает по одному элементу за раз, а не материализует всю коллекцию. `yield return`, `IEn... |
| [Junior/naming-conventions.md](CSharp/Junior/naming-conventions.md) | Junior | 105,7 | PascalCase, camelCase, и почему `Id` лучше `ID`. Microsoft Framework Design Guidelines, naming patterns (As... |
| [Junior/oop.md](CSharp/Junior/oop.md) | Junior | 90,3 | Способ организации кода через объекты, объединяющие данные и поведение. Encapsulation, Inheritance, Polymor... |
| [Junior/strings-regex.md](CSharp/Junior/strings-regex.md) | Junior | 94,7 | Самый частый тип в любом коде — и самый недооценённый. Immutability, encoding (UTF-8/UTF-16), comparison or... |
| [Junior/tuples-deconstruction.md](CSharp/Junior/tuples-deconstruction.md) | Junior | 119,1 | Modern C# everyday feature. ValueTuple, named tuples, deconstruction, multiple return values. Появилось в C... |
| [Middle/attributes-metadata.md](CSharp/Middle/attributes-metadata.md) | Middle | 40 | Декларации, прикрепляемые к коду и читаемые в runtime / compile-time. Built-in атрибуты, custom attributes,... |
| [Middle/bcl-essentials.md](CSharp/Middle/bcl-essentials.md) | Middle | 23 | Canonical-сборник по «мелким» типам BCL, которые используются в каждом проекте, но обычно известны фрагмент... |
| [Middle/delegates-events.md](CSharp/Middle/delegates-events.md) | Middle | 37,8 | Type-safe function pointers + Observer pattern. `Func`/`Action`/`Predicate`, multicast delegates, `event` k... |
| [Middle/dispose-pattern.md](CSharp/Middle/dispose-pattern.md) | Middle | 46,1 | Детерминированное освобождение неуправляемых ресурсов в managed runtime. `IDisposable`, `using`, `IAsyncDis... |
| [Middle/equality-comparison.md](CSharp/Middle/equality-comparison.md) | Middle | 31,8 | Reference equality vs value equality, контракт `Equals`/`GetHashCode`, `IEquatable<T>`, `IComparable<T>`, c... |
| [Middle/error-handling.md](CSharp/Middle/error-handling.md) | Middle | 42,3 | Exception hierarchy, try/catch/finally, exception filters, custom exceptions, Result pattern, ASP.NET Core ... |
| [Middle/generics-deep.md](CSharp/Middle/generics-deep.md) | Middle | 34,2 | Type parameters, constraints (`where T : ...`), variance (`in`/`out`), generic methods, generic math (.NET ... |
| [Middle/indexers-operators.md](CSharp/Middle/indexers-operators.md) | Middle | 31 | Объект ведёт себя как массив или встроенный тип. `this[index]` indexer + operator overloading (`+`, `-`, `=... |
| [Middle/io-streams.md](CSharp/Middle/io-streams.md) | Middle | 37,8 | Универсальная абстракция для byte sequences: файлы, network, memory, pipes. `Stream` hierarchy, `FileStream... |
| [Middle/keywords-reference.md](CSharp/Middle/keywords-reference.md) | Middle | 33,3 | Все 100+ keywords и contextual keywords языка с группировкой по назначению. От `abstract` до `yield`, modif... |
| [Middle/modern-features.md](CSharp/Middle/modern-features.md) | Middle | 45,4 | Records, `init`, pattern matching, primary constructors, raw string literals, collection expressions, gener... |
| [Middle/nullable-types.md](CSharp/Middle/nullable-types.md) | Middle | 30,6 | Два разных механизма с одним именем. `Nullable<T>` (value types, .NET 2.0+) — runtime feature; Nullable Ref... |
| [Middle/numeric-types-math.md](CSharp/Middle/numeric-types-math.md) | Middle | 37,9 | Integer types, floating-point, decimal, BigInteger, generic math (.NET 7+). Когда `double`, когда `decimal`... |
| [Middle/serialization-deep.md](CSharp/Middle/serialization-deep.md) | Middle | 26,9 | Canonical-файл по сериализации в .NET: System.Text.Json как дефолт 2026 (options, конвертеры, полиморфизм, ... |
| [Senior/async-threading.md](CSharp/Senior/async-threading.md) | Senior | 72 | `async/await` глубоко, Task vs ValueTask, threading primitives, `lock`/Mutex/Semaphore, `Parallel`, `Channe... |
| [Senior/cli-tools-scripting.md](CSharp/Senior/cli-tools-scripting.md) | Senior | 40,8 | Top-level statements, `dotnet script`, `System.CommandLine`, Spectre.Console, Native AOT для CLI. Замена Ba... |
| [Senior/csharp-language-design.md](CSharp/Senior/csharp-language-design.md) | Senior | 43,6 | Принципы проектирования C#, design committee, эволюция через 14 версий, trade-offs. Зачем добавили (и не до... |
| [Senior/csharp-vs-other-langs.md](CSharp/Senior/csharp-vs-other-langs.md) | Senior | 48,4 | Конкретные сравнения C# с Java, Kotlin, TypeScript, Go, Rust, Python, F#. Где C# выигрывает, где проигрывае... |
| [Senior/design-patterns.md](CSharp/Senior/design-patterns.md) | Senior | 56,6 | SOLID principles, GoF patterns, modern .NET equivalents. Какие patterns актуальны до сих пор, какие устарел... |
| [Senior/desktop-frameworks.md](CSharp/Senior/desktop-frameworks.md) | Senior | 46,4 | WPF (mature Windows), WinUI 3 (modern Windows), WinForms (legacy), MAUI (cross-platform Microsoft), Avaloni... |
| [Senior/fenwick-bit.md](CSharp/Senior/fenwick-bit.md) | Senior | 13,3 | Binary Indexed Tree: prefix-sum + point-update за O(log n) на плоском `int[]`, без node objects и без аллок... |
| [Senior/functional-csharp.md](CSharp/Senior/functional-csharp.md) | Senior | 48,6 | Pure functions, immutability, higher-order functions, monads (`Option<T>`, `Result<T>`), function compositi... |
| [Senior/gof-patterns-extended.md](CSharp/Senior/gof-patterns-extended.md) | Senior | 61,4 | Полный справочник 23 GoF design patterns с C# implementation, modern .NET equivalents, real-world usage. Re... |
| [Senior/memory-pooling.md](CSharp/Senior/memory-pooling.md) | Senior | 38,7 | Сокращение allocation pressure через reuse: `ArrayPool<T>`, `MemoryPool<T>`, `ObjectPool<T>`, `Span<T>`/`Me... |
| [Senior/reflection-expression-trees.md](CSharp/Senior/reflection-expression-trees.md) | Senior | 53 | `System.Reflection` — runtime introspection, dynamic invocation. Expression trees — lambda как data, founda... |
| [Senior/source-generators.md](CSharp/Senior/source-generators.md) | Senior | 55,8 | Roslyn-based components, генерирующие C# code на compile-time на основе AST + metadata. Замена reflection д... |
| [Senior/types-and-memory.md](CSharp/Senior/types-and-memory.md) | Senior | 57,4 | Value types vs reference types, stack vs heap, boxing/unboxing, struct semantics, ref struct, escape analys... |
| [Senior/unsafe-pointers.md](CSharp/Senior/unsafe-pointers.md) | Senior | 37,6 | `unsafe` context, raw pointers, `fixed`, `stackalloc`, function pointers, P/Invoke marshaling. Когда manage... |

### Runtime — 9 файлов / 329 KB

| Файл | Уровень | KB | Описание |
|---|---|---:|---|
| [Junior/memory-stack-heap.md](Runtime/Junior/memory-stack-heap.md) | Junior | 30,8 | Где живут переменные, как работает копирование, что такое boxing, почему `string` immutable. Введение перед... |
| [Junior/runtime-basics.md](Runtime/Junior/runtime-basics.md) | Junior | 28,8 | Что происходит когда запускается C# программа: CLR, JIT compilation, GC, managed memory. Bird's-eye view пе... |
| [Middle/threading-basics.md](Runtime/Middle/threading-basics.md) | Middle | 29,5 | Фундамент многопоточности перед async/await. Что такое Thread, ThreadPool, Task Parallel Library, Parallel.... |
| [Senior/compilation-jit.md](Runtime/Senior/compilation-jit.md) | Senior | 43,1 | Полный pipeline исполнения: Roslyn компилирует C# в IL, CLR грузит assembly и MethodTable, RyuJIT через Tie... |
| [Senior/concurrency-atomics.md](Runtime/Senior/concurrency-atomics.md) | Senior | 40,6 | Самая глубокая заметка по lock-free и low-level concurrency. Цель — закрыть всё, что нужно знать Senior'у п... |
| [Senior/diagnostics-tools.md](Runtime/Senior/diagnostics-tools.md) | Senior | 33,2 | Полный гайд по production troubleshooting в .NET. Закрывает: dotnet-counters/trace/dump/gcdump/monitor, Eve... |
| [Senior/gc-memory.md](Runtime/Senior/gc-memory.md) | Senior | 55,1 | Это самая глубокая заметка по runtime в этом vault. Цель — закрыть все вопросы по памяти, которые задают на... |
| [Senior/interop-pinvoke.md](Runtime/Senior/interop-pinvoke.md) | Senior | 30,4 | Полный гайд по работе с native кодом из .NET. Закрывает: P/Invoke, `LibraryImport` source generator (.NET 7... |
| [Senior/span-layout.md](Runtime/Senior/span-layout.md) | Senior | 37,7 | Самая глубокая заметка по работе с памятью без аллокаций. Закрывает Span/Memory, struct layout, SIMD/Vector... |

### AspNetCore — 23 файлов / 713 KB

| Файл | Уровень | KB | Описание |
|---|---|---:|---|
| [Junior/http-fundamentals.md](AspNetCore/Junior/http-fundamentals.md) | Junior | 27,7 | Без понимания HTTP — невозможно делать качественный backend. Status codes, methods, headers, HTTPS, CORS — ... |
| [Middle/aspnet-controllers-routing.md](AspNetCore/Middle/aspnet-controllers-routing.md) | Middle | 27,2 | Когда Controllers, когда Minimal API. Routing patterns, model binding, return types, action filters, MapGro... |
| [Middle/aspnet-dependency-injection-deep.md](AspNetCore/Middle/aspnet-dependency-injection-deep.md) | Middle | 27 | DI lifetimes deep, IServiceScopeFactory, keyed services (.NET 8+), TimeProvider, captive dependencies, asyn... |
| [Middle/aspnet-error-handling.md](AspNetCore/Middle/aspnet-error-handling.md) | Middle | 26,8 | Exception handling middleware, IExceptionHandler (.NET 8+), ProblemDetails RFC 7807, error pages, validatio... |
| [Middle/aspnet-rate-limiting.md](AspNetCore/Middle/aspnet-rate-limiting.md) | Middle | 26,6 | Built-in rate limiting middleware (.NET 7+), 4 algorithms (fixed, sliding, token bucket, concurrency), per-... |
| [Middle/fluent-validation.md](AspNetCore/Middle/fluent-validation.md) | Middle | 28 | Стандарт для validation в .NET 2026. vs DataAnnotations: гибче, тестируемо, async-friendly. Closes пробел "... |
| [Middle/http-client-resilience.md](AspNetCore/Middle/http-client-resilience.md) | Middle | 27,4 | Canonical-файл по HTTP-клиенту в .NET: почему `new HttpClient()` на каждый запрос кладёт прод, почему singl... |
| [Middle/object-mapping.md](AspNetCore/Middle/object-mapping.md) | Middle | 25,3 | Daily work каждого ASP.NET проекта: DTO ↔ Domain ↔ ViewModel mapping. Closes пробел "копирую properties вру... |
| [Senior/api-design.md](AspNetCore/Senior/api-design.md) | Senior | 27,7 | С .NET 8-10 функциональный паритет почти полный: у Minimal API есть endpoint-фильтры, per-endpoint auth (`R... |
| [Senior/auth-security.md](AspNetCore/Senior/auth-security.md) | Senior | 59,1 | Authentication — кто ты? (identity, JWT, Cookie). Authorization — что тебе можно? (roles, policies, claims). |
| [Senior/blazor-server.md](AspNetCore/Senior/blazor-server.md) | Senior | 32 | Web UI на C#/Razor с рендером на сервере и доставкой diff'ов через SignalR-circuit; разбор render modes, уп... |
| [Senior/blazor-wasm.md](AspNetCore/Senior/blazor-wasm.md) | Senior | 34,8 | C# в браузере через WebAssembly. Альтернатива Blazor Server и React/Angular/Vue. Закрывает: WASM модель, ho... |
| [Senior/caching.md](AspNetCore/Senior/caching.md) | Senior | 30,1 | Многоуровневое кэширование (`IMemoryCache`, Redis, HybridCache, Output Cache, CDN) и защита API через Rate ... |
| [Senior/di-configuration.md](AspNetCore/Senior/di-configuration.md) | Senior | 24 | DI-контейнер ASP.NET Core, lifetimes, Keyed Services и типизированная конфигурация через `IOptions` с учёто... |
| [Senior/graphql.md](AspNetCore/Senior/graphql.md) | Senior | 31,5 | Полный гайд по GraphQL в .NET. Закрывает: HotChocolate v15 (актуальная версия 2026), code-first vs schema-f... |
| [Senior/hosting-background.md](AspNetCore/Senior/hosting-background.md) | Senior | 38,4 | Фоновая обработка через `BackgroundService` с `Channel<T>` и `PeriodicTimer`, управление жизненным циклом п... |
| [Senior/kestrel-as-raw-host.md](AspNetCore/Senior/kestrel-as-raw-host.md) | Senior | 51,4 | Как использовать Kestrel напрямую — без MVC, без endpoint routing — когда ты сам строишь HTTP-framework пов... |
| [Senior/logging-observability.md](AspNetCore/Senior/logging-observability.md) | Senior | 14,1 | Structured logging сохраняет данные как key-value, а не как строку. Позволяет фильтровать, искать, агрегиро... |
| [Senior/native-aot.md](AspNetCore/Senior/native-aot.md) | Senior | 23,2 | AOT-компиляция в single executable без JIT и CLR ради мгновенного startup и малого footprint; цена — trimmi... |
| [Senior/pipeline-middleware.md](AspNetCore/Senior/pipeline-middleware.md) | Senior | 42,2 | Полный гайд по ASP.NET Core pipeline. Закрывает: middleware patterns deep, IExceptionHandler (.NET 8+), Out... |
| [Senior/resilience.md](AspNetCore/Senior/resilience.md) | Senior | 24,2 | Устойчивость к сбоям зависимостей через Polly v8 — Retry, Timeout, Circuit Breaker, Hedging, Bulkhead и Fal... |
| [Senior/security-practices.md](AspNetCore/Senior/security-practices.md) | Senior | 28,6 | Прикладные приёмы защиты кода: timing-safe сравнение токенов (`FixedTimeEquals`), SHA256-хэширование секрет... |
| [Senior/signalr.md](AspNetCore/Senior/signalr.md) | Senior | 35,6 | Полный гайд по standalone SignalR в .NET 10. Закрывает: Hub методы, transport selection (WebSocket/SSE/Long... |

### EFCore — 13 файлов / 441 KB

| Файл | Уровень | KB | Описание |
|---|---|---:|---|
| [Junior/ef-basics.md](EFCore/Junior/ef-basics.md) | Junior | 37,6 | DbContext, DbSet, основные CRUD операции, миграции, конфигурация. Введение перед `Middle/dapper-comparison.... |
| [Junior/ef-crud-queries.md](EFCore/Junior/ef-crud-queries.md) | Junior | 31,7 | Where, Select, OrderBy, FirstOrDefault, joins basics, includes, aggregations. Practical querying для daily ... |
| [Middle/dapper-comparison.md](EFCore/Middle/dapper-comparison.md) | Middle | 23,8 | Dapper — micro-ORM: тонкий wrapper над ADO.NET, raw SQL + mapping. Часто требуют в вакансиях как альтернати... |
| [Middle/ef-bulk-operations.md](EFCore/Middle/ef-bulk-operations.md) | Middle | 31,9 | Bulk INSERT/UPDATE/DELETE без change tracking. ExecuteUpdate/ExecuteDelete (EF Core 7+), EFCore.BulkExtensi... |
| [Middle/ef-loading-strategies.md](EFCore/Middle/ef-loading-strategies.md) | Middle | 28,7 | Как загружать связанные данные правильно: Include vs Select vs SplitQuery vs explicit loading. Closes пробе... |
| [Middle/ef-transactions-concurrency.md](EFCore/Middle/ef-transactions-concurrency.md) | Middle | 29,1 | Транзакции в EF Core, isolation levels, optimistic concurrency tokens, deadlock handling, distributed trans... |
| [Middle/ef-value-converters.md](EFCore/Middle/ef-value-converters.md) | Middle | 18,2 | ValueConverter, ValueComparer, owned types, JSON columns (EF Core 7+), backing fields, computed columns. Вс... |
| [Senior/basics-tracking.md](EFCore/Senior/basics-tracking.md) | Senior | 36,9 | Глубокий гайд по фундаментальным механизмам EF Core. Закрывает: DbContext lifecycle, Change Tracker внутрен... |
| [Senior/concurrency.md](EFCore/Senior/concurrency.md) | Senior | 40,8 | Полный гайд по concurrent доступу к данным через EF Core. Закрывает: optimistic / pessimistic locking, isol... |
| [Senior/ef-patterns.md](EFCore/Senior/ef-patterns.md) | Senior | 44,9 | Production-grade паттерны на EF Core. Закрывает: Repository / Unit of Work, Soft Delete, Audit interceptors... |
| [Senior/migrations.md](EFCore/Senior/migrations.md) | Senior | 40,2 | Полный гайд по управлению схемой БД в .NET. Закрывает: EF Core migrations, idempotent scripts, zero-downtim... |
| [Senior/queries-performance.md](EFCore/Senior/queries-performance.md) | Senior | 34,7 | Performance-разбор read-side EF Core: устранение N+1 через `Include`/projection/split queries, `AsNoTrackin... |
| [Senior/relationships.md](EFCore/Senior/relationships.md) | Senior | 42,6 | Полный гайд по связям и типам в EF Core. Закрывает: все relationships (1:1/1:N/N:N), TPH/TPT/TPC inheritanc... |

### SQL — 9 файлов / 259 KB

| Файл | Уровень | KB | Описание |
|---|---|---:|---|
| [Junior/sql-basics.md](SQL/Junior/sql-basics.md) | Junior | 29,1 | Базовые понятия реляционных БД и SQL для разработчиков. Закрывает пробел "что такое SQL" перед optimization... |
| [Middle/indexes-deep.md](SQL/Middle/indexes-deep.md) | Middle to Senior | 25,8 | Глубокий гайд по индексам в реляционных БД. Что такое индекс изнутри, какие виды бывают, когда применять, к... |
| [Senior/eav-flexible-store-indexing.md](SQL/Senior/eav-flexible-store-indexing.md) | Senior | 24,4 | Как держать EAV-таблицу (`_values` с типизированными колонками `_Long`/`_String`/`_DateTimeOffset`/`_Boolea... |
| [Senior/mvcc-and-locking.md](SQL/Senior/mvcc-and-locking.md) | Senior | 29,1 | MVCC (читатели не блокируют писателей), табличные и строчные lock modes, `SKIP LOCKED` для очередей без бро... |
| [Senior/optimization.md](SQL/Senior/optimization.md) | Senior | 47,3 | Индексы (clustered/covering/filtered, leftmost prefix), чтение execution plan и join-алгоритмы, паттерны за... |
| [Senior/postgres-functions-triggers.md](SQL/Senior/postgres-functions-triggers.md) | Senior | 17,1 | PL/pgSQL функции vs процедуры, volatility-категории, триггеры (BEFORE/AFTER, аудит, `updated_at`), вызов ra... |
| [Senior/postgresql-deep.md](SQL/Senior/postgresql-deep.md) | Senior | 49 | Production-паттерны Npgsql (`NpgsqlDataSource`, multiplexing, bulk `COPY`, pipelining), JSONB и GIN-индексы... |
| [Senior/sql-security.md](SQL/Senior/sql-security.md) | Senior | 17,4 | От инъекций защищает любой параметр, но `AddWithValue` выводит тип из значения (`string` → Unicode `nvarcha... |
| [Senior/zero-downtime-migrations.md](SQL/Senior/zero-downtime-migrations.md) | Senior | 20,1 | Как менять схему большой таблицы под нагрузкой: карта стоимости DDL, `lock_timeout` + retry против head-of-... |

### Architecture — 17 файлов / 630 KB

| Файл | Уровень | KB | Описание |
|---|---|---:|---|
| [Junior/architecture-basics.md](Architecture/Junior/architecture-basics.md) | Junior | 44,9 | Зачем структурировать код, что такое слои, separation of concerns, dependency direction. Введение перед Mid... |
| [Middle/microservices-vs-monolith.md](Architecture/Middle/microservices-vs-monolith.md) | Middle to Senior | 25,3 | Когда монолит, когда микросервисы, когда modular monolith. Реальные trade-offs, не маркетинг. Закрывает: ти... |
| [Middle/patterns-decision-guide.md](Architecture/Middle/patterns-decision-guide.md) | Middle to Senior | 39,1 | Главный навигационный файл vault'а: под какую задачу что выбрать. Связывает GoF паттерны, архитектурные сти... |
| [Middle/real-world-scenarios.md](Architecture/Middle/real-world-scenarios.md) | Middle to Senior | 52 | Сценарий → решение. От меню навигации до микросервисного e-commerce. От чего зависит выбор архитектуры, как... |
| [Senior/agent-safe-architecture.md](Architecture/Senior/agent-safe-architecture.md) | Senior | 22,3 | Прозаические правила в `AGENTS.md` гниют под context rot: чем длиннее сессия, тем дальше инструкция уезжает... |
| [Senior/arch-tests.md](Architecture/Senior/arch-tests.md) | Senior | 23,3 | Правила архитектуры (Domain не зависит от Infrastructure, naming conventions, module boundaries) как исполн... |
| [Senior/architecture-decisions.md](Architecture/Senior/architecture-decisions.md) | Senior | 37 | Полный гайд по ADR: что это, зачем, когда писать, как структурировать, как поддерживать. Закрывает: lifecyc... |
| [Senior/architecture-patterns.md](Architecture/Senior/architecture-patterns.md) | Senior | 53,7 | По материалам: [N-Layered vs Clean vs VSA](https://antondevtips.com/blog/n-layered-vs-clean-vs-vertical-sli... |
| [Senior/choosing-dependencies.md](Architecture/Senior/choosing-dependencies.md) | Senior | 28,9 | С 2023 по 2026 коммерциализировалась половина «дефолтного» стека .NET-бэкендера: MediatR, AutoMapper, MassT... |
| [Senior/cqrs-mediatr.md](Architecture/Senior/cqrs-mediatr.md) | Senior | 27,8 | Разделение Command/Query и развязка sender'а от handler'а через mediator с pipeline behaviors (validation, ... |
| [Senior/ddd.md](Architecture/Senior/ddd.md) | Senior | 64,5 | Tactical DDD (Entity, VO, Aggregate) — это про код. |
| [Senior/distributed-systems.md](Architecture/Senior/distributed-systems.md) | Senior | 67 | Примеры в этом файле написаны на MassTransit v8 (Apache 2.0) — код валиден. Но v9 (компания Massient) — пла... |
| [Senior/eip-content-based-router.md](Architecture/Senior/eip-content-based-router.md) | Senior | 20,9 | CBR выбирает ровно один маршрут из многих (first-match-wins). Message Filter — частный случай с одной ветко... |
| [Senior/solid.md](Architecture/Senior/solid.md) | Senior | 24 | Пять принципов SOLID (SRP, OCP, LSP, ISP, DIP) плюс DRY/KISS/YAGNI с примерами на C#, маркерами нарушений и... |
| [Senior/system-design.md](Architecture/Senior/system-design.md) | Senior | 32,2 | Фреймворк system design интервью (requirements, capacity estimation, tradeoffs) плюс готовые шаблоны: rate ... |
| [Senior/twelve-factor-app.md](Architecture/Senior/twelve-factor-app.md) | Senior | 36,8 | Manifesto от Heroku (2011), стал industry standard. 12 принципов для apps which можно reliably deploy в clo... |
| [Senior/webai-csharp-architecture.md](Architecture/Senior/webai-csharp-architecture.md) | Senior | 30 | Landing page generator: user fills form -> AI generates texts + images -> site published instantly. |

### Quality — 5 файлов / 135 KB

| Файл | Уровень | KB | Описание |
|---|---|---:|---|
| [Junior/clean-code.md](Quality/Junior/clean-code.md) | Junior | 34,6 | Что такое читаемый код, как его писать, и почему это самая недооценённая навык программиста. Базовые принци... |
| [Middle/refactoring.md](Quality/Middle/refactoring.md) | Middle to Senior | 29,9 | Систематические техники для улучшения кода без изменения внешнего поведения. Каталог refactorings, code sme... |
| [Senior/code-quality.md](Quality/Senior/code-quality.md) | Senior | 21,5 | Автоматический enforcement стиля и качества кода: `.editorconfig` + Roslyn/Meziantou/SonarAnalyzer analyzer... |
| [Senior/static-analysis.md](Quality/Senior/static-analysis.md) | Senior | 23,5 | Полный гайд по static analysis в .NET. Закрывает: встроенные analyzers, SonarAnalyzer, Meziantou, Roslynato... |
| [code-review.md](Quality/code-review.md) | All | 25,5 | Полный гайд по code review: что искать, как давать feedback, как принимать критику, anti-patterns, growth-o... |

### Testing — 5 файлов / 159 KB

| Файл | Уровень | KB | Описание |
|---|---|---:|---|
| [Junior/testing-fundamentals.md](Testing/Junior/testing-fundamentals.md) | Junior | 41,4 | Что такое тест, какие виды бывают, зачем каждый нужен, где применяется. Базовые концепты для джуна, система... |
| [Middle/mocking-strategies.md](Testing/Middle/mocking-strategies.md) | Middle to Senior | 22,9 | Когда мокать, как мокать, а главное — когда НЕ мокать. Test doubles разных видов, NSubstitute vs Moq, anti-... |
| [Senior/integration-testing.md](Testing/Senior/integration-testing.md) | Senior | 26,4 | Глубокий гайд по integration tests в .NET. Закрывает: WebApplicationFactory, Testcontainers, тестирование с... |
| [Senior/mutation-load-testing.md](Testing/Senior/mutation-load-testing.md) | Senior | 24,4 | Mutation testing проверяет качество тестов. Load testing — performance под нагрузкой. Закрывает: Stryker.NE... |
| [Senior/testing.md](Testing/Senior/testing.md) | Senior | 44,1 | Стратегия и tooling-стек тестирования в .NET 2026: пирамида vs алмаз, xUnit/TUnit, NSubstitute, Testcontain... |

### Performance — 12 файлов / 253 KB

| Файл | Уровень | KB | Описание |
|---|---|---:|---|
| [Junior/performance-fundamentals.md](Performance/Junior/performance-fundamentals.md) | Junior | 24,6 | Базовое понимание performance. Что такое "медленно", как мерить, почему оптимизировать преждевременно вредн... |
| [Middle/async-performance.md](Performance/Middle/async-performance.md) | Middle to Senior | 13,4 | Async-specific performance: ValueTask, TaskCompletionSource, async pitfalls, ConfigureAwait. Когда async по... |
| [Middle/bottleneck-analysis.md](Performance/Middle/bottleneck-analysis.md) | Middle to Senior | 12,2 | Систематический подход к нахождению где система тормозит. CPU, memory, I/O, network, DB — как определить ка... |
| [Middle/caching-strategies.md](Performance/Middle/caching-strategies.md) | Middle to Senior | 20,9 | Какие виды кеша есть, когда какой применять, как избежать stale data, как не сделать хуже. Глубокий разбор ... |
| [Middle/lazy-eager-loading.md](Performance/Middle/lazy-eager-loading.md) | Middle | 13,2 | Когда загружать данные сразу, когда откладывать. Trade-offs для EF Core, lazy initialization, eager configu... |
| [Middle/optimization-patterns.md](Performance/Middle/optimization-patterns.md) | Middle to Senior | 27,7 | Practical optimization patterns для типичных задач. Когда применять, что измерять, какие trade-offs. Не "ma... |
| [Senior/capacity-planning.md](Performance/Senior/capacity-planning.md) | Senior | 10,6 | Сколько ресурсов нужно для текущей и будущей нагрузки. Расчёт CPU, memory, network, БД для заданного traffic. |
| [Senior/hft-low-latency.md](Performance/Senior/hft-low-latency.md) | Senior | 43,4 | Микросекундный hot-path в .NET: allocation-free через `Span<T>`/`stackalloc`/`ArrayPool<T>`, lock-free стру... |
| [Senior/memory-profiling.md](Performance/Senior/memory-profiling.md) | Senior | 17,7 | Как находить memory leaks, GC pressure, оптимизировать heap usage. Tools, workflow, decoding heap snapshots. |
| [Senior/performance-budgets.md](Performance/Senior/performance-budgets.md) | Senior | 13,1 | Performance как functional requirement — устанавливаем cost targets, мониторим, alert когда превышаем. |
| [Senior/performance.md](Performance/Senior/performance.md) | Senior | 35,7 | Workflow измерения и оптимизации .NET: точные замеры через BenchmarkDotNet (`[MemoryDiagnoser]`), профилиро... |
| [Senior/threadpool-starvation-hill-climbing.md](Performance/Senior/threadpool-starvation-hill-climbing.md) | Senior | 20,5 | Самый коварный production-инцидент в .NET: throughput падает в разы, p99 уходит в секунды, а CPU при этом 8... |

### Infrastructure — 17 файлов / 615 KB

| Файл | Уровень | KB | Описание |
|---|---|---:|---|
| [Junior/docker-for-dev.md](Infrastructure/Junior/docker-for-dev.md) | Junior | 29 | Что такое контейнеры, daily команды, Dockerfile основы, docker-compose для local dev. Введение перед `Senio... |
| [Junior/project-setup-basics.md](Infrastructure/Junior/project-setup-basics.md) | Junior | 28,6 | dotnet CLI, solution + projects organization, .gitignore, basic git workflow для C# проектов. Введение пере... |
| [Middle/ai-coding-tools.md](Infrastructure/Middle/ai-coding-tools.md) | Middle | 29,8 | Ландшафт AI-tooling глазами .NET-разработчика: от автодополнения до автономных агентов в CI. Кто есть кто (... |
| [Middle/cicd-github-actions.md](Infrastructure/Middle/cicd-github-actions.md) | Middle | 25,5 | Continuous Integration / Continuous Delivery для .NET: build, test, lint, security scan, deploy. GitHub Act... |
| [Middle/kubernetes.md](Infrastructure/Middle/kubernetes.md) | Middle to Senior | 23 | Что Senior .NET должен знать про Kubernetes: концепты, Pod/Deployment/Service, deploy .NET app, health chec... |
| [Senior/api-gateway.md](Infrastructure/Senior/api-gateway.md) | Senior | 32,2 | Single entry point для микросервисов. Routing, auth, rate limiting, transformation — в одном месте. Closes ... |
| [Senior/aspire.md](Infrastructure/Senior/aspire.md) | Senior | 27,8 | Canonical deep-dive по Aspire. Закрывает: AppHost-композиция, ServiceDefaults, Dashboard, aspire CLI (init/... |
| [Senior/docker.md](Infrastructure/Senior/docker.md) | Senior | 70 | Самая глубокая заметка по контейнеризации в этом vault. Цель — закрыть все вопросы по Docker, которые задаю... |
| [Senior/ipc-named-pipes-grpc.md](Infrastructure/Senior/ipc-named-pipes-grpc.md) | Senior | 44,5 | Передача данных между процессами в .NET: Named Pipes, Memory-Mapped Files (zero-copy ring buffer), Anonymou... |
| [Senior/llm-rag-patterns.md](Infrastructure/Senior/llm-rag-patterns.md) | Senior | 52,5 | Production-RAG на .NET: chunking, embeddings, hybrid search (BM25 + dense + RRF), reranking, tool use и str... |
| [Senior/mcp-csharp.md](Infrastructure/Senior/mcp-csharp.md) | Senior | 27,2 | MCP как стандартный разъём между LLM-хостами и твоими инструментами/данными: архитектура протокола (host/cl... |
| [Senior/messaging.md](Infrastructure/Senior/messaging.md) | Senior | 47,1 | Асинхронная коммуникация через message brokers: RabbitMQ vs Kafka, MassTransit, Outbox pattern и Dead Lette... |
| [Senior/nosql-databases.md](Infrastructure/Senior/nosql-databases.md) | Senior | 24,6 | Когда NoSQL > SQL и какой NoSQL для какой задачи. MongoDB / Redis / Cosmos DB / DynamoDB / Cassandra — реал... |
| [Senior/observability.md](Infrastructure/Senior/observability.md) | Senior | 32,8 | Три pillars наблюдаемости (logs, metrics, traces) на OpenTelemetry для .NET: экспорт в Prometheus / Grafana... |
| [Senior/project-setup.md](Infrastructure/Senior/project-setup.md) | Senior | 30,6 | Полный гайд по starting up .NET проекта с production-grade defaults. Закрывает: Directory.Build.props, .edi... |
| [Senior/semantic-kernel.md](Infrastructure/Senior/semantic-kernel.md) | Senior | 53,1 | Semantic Kernel — зрелый AI SDK Microsoft для .NET (LLM, embeddings, vector search, agents), с октября 2025... |
| [Senior/wpf-production.md](Infrastructure/Senior/wpf-production.md) | Senior | 36,3 | Современный production-стек WPF на .NET 10: CommunityToolkit.Mvvm (source generators), WPF-UI (Fluent Desig... |

### Snippets — 6 файлов / 44 KB

| Файл | Уровень | KB | Описание |
|---|---|---:|---|
| [crud-example.md](Snippets/crud-example.md) | Senior | 13,3 | Сквозной copy-paste template всех четырёх CRUD-операций: путь от Minimal API endpoint через explicit Handle... |
| [efcore-queries.md](Snippets/efcore-queries.md) | Middle to Senior | 5 | Производительные EF Core read-паттерны: `AsNoTracking` + проекция в DTO, offset- и keyset-пагинация, `AsSpl... |
| [mediatr-handlers.md](Snippets/mediatr-handlers.md) | Middle to Senior | 5,4 | CQRS-обвязка на MediatR: Command/Query handlers, возвращающие `Result<T>`, FluentValidation validator и `IP... |
| [result-pattern.md](Snippets/result-pattern.md) | Middle | 3,8 | Кастомный `Result<T>` + `Error` record как замена исключениям: фабрики Ok/Fail, `Match()` для маппинга в HT... |
| [vertical-slice-handler.md](Snippets/vertical-slice-handler.md) | All | 9 | Copy-paste слайс без mediator-библиотеки: feature-класс с Command/Response records и Handler (primary const... |
| [wpf-viewmodel.md](Snippets/wpf-viewmodel.md) | Middle | 7 | WPF на CommunityToolkit.Mvvm: `ObservableObject` с `[ObservableProperty]`/`[RelayCommand]`, async-safe загр... |

### (root) — 1 файлов / 2 KB

| Файл | Уровень | KB | Описание |
|---|---|---:|---|
| [LICENSE.md](LICENSE.md) | — | 2 |  |

---

## По уровню (компактно)

### Junior (26)

[[architecture-basics]] · [[http-fundamentals]] · [[anonymous-types]] · [[async-await-basics]] · [[collections-linq]] · [[csharp-basics]] · [[datetime-timezones]] · [[debugging-basics]] · [[dotnet-cli-getting-started]] · [[enums-flags]] · [[extension-methods]] · [[iterators-yield]] · [[naming-conventions]] · [[oop]] · [[strings-regex]] · [[tuples-deconstruction]] · [[ef-basics]] · [[ef-crud-queries]] · [[docker-for-dev]] · [[project-setup-basics]] · [[performance-fundamentals]] · [[clean-code]] · [[memory-stack-heap]] · [[runtime-basics]] · [[sql-basics]] · [[testing-fundamentals]]

### Junior to Middle (1)

[[02_junior-to-middle]]

### Middle (32)

[[aspnet-controllers-routing]] · [[aspnet-dependency-injection-deep]] · [[aspnet-error-handling]] · [[aspnet-rate-limiting]] · [[fluent-validation]] · [[http-client-resilience]] · [[object-mapping]] · [[attributes-metadata]] · [[bcl-essentials]] · [[delegates-events]] · [[dispose-pattern]] · [[equality-comparison]] · [[error-handling]] · [[generics-deep]] · [[indexers-operators]] · [[io-streams]] · [[keywords-reference]] · [[modern-features]] · [[nullable-types]] · [[numeric-types-math]] · [[serialization-deep]] · [[dapper-comparison]] · [[ef-bulk-operations]] · [[ef-loading-strategies]] · [[ef-transactions-concurrency]] · [[ef-value-converters]] · [[ai-coding-tools]] · [[cicd-github-actions]] · [[lazy-eager-loading]] · [[threading-basics]] · [[result-pattern]] · [[wpf-viewmodel]]

### Middle to Senior (15)

[[microservices-vs-monolith]] · [[patterns-decision-guide]] · [[real-world-scenarios]] · [[kubernetes]] · [[03_middle-to-senior]] · [[case-studies-top7]] · [[async-performance]] · [[bottleneck-analysis]] · [[caching-strategies]] · [[optimization-patterns]] · [[refactoring]] · [[efcore-queries]] · [[mediatr-handlers]] · [[indexes-deep]] · [[mocking-strategies]]

### Senior (87)

[[agent-safe-architecture]] · [[arch-tests]] · [[architecture-decisions]] · [[architecture-patterns]] · [[choosing-dependencies]] · [[cqrs-mediatr]] · [[ddd]] · [[distributed-systems]] · [[eip-content-based-router]] · [[solid]] · [[system-design]] · [[twelve-factor-app]] · [[webai-csharp-architecture]] · [[api-design]] · [[auth-security]] · [[blazor-server]] · [[blazor-wasm]] · [[caching]] · [[di-configuration]] · [[graphql]] · [[hosting-background]] · [[kestrel-as-raw-host]] · [[logging-observability]] · [[native-aot]] · [[pipeline-middleware]] · [[resilience]] · [[security-practices]] · [[signalr]] · [[async-threading]] · [[cli-tools-scripting]] · [[csharp-language-design]] · [[csharp-vs-other-langs]] · [[design-patterns]] · [[desktop-frameworks]] · [[fenwick-bit]] · [[functional-csharp]] · [[gof-patterns-extended]] · [[memory-pooling]] · [[reflection-expression-trees]] · [[source-generators]] · [[types-and-memory]] · [[unsafe-pointers]] · [[basics-tracking]] · [[concurrency]] · [[ef-patterns]] · [[migrations]] · [[queries-performance]] · [[relationships]] · [[api-gateway]] · [[aspire]] · [[docker]] · [[ipc-named-pipes-grpc]] · [[llm-rag-patterns]] · [[mcp-csharp]] · [[messaging]] · [[nosql-databases]] · [[observability]] · [[project-setup]] · [[semantic-kernel]] · [[wpf-production]] · [[09_senior-tips-cheatsheet]] · [[10_interview-behavioral]] · [[capacity-planning]] · [[hft-low-latency]] · [[memory-profiling]] · [[performance-budgets]] · [[performance]] · [[threadpool-starvation-hill-climbing]] · [[code-quality]] · [[static-analysis]] · [[compilation-jit]] · [[concurrency-atomics]] · [[diagnostics-tools]] · [[gc-memory]] · [[interop-pinvoke]] · [[span-layout]] · [[crud-example]] · [[eav-flexible-store-indexing]] · [[mvcc-and-locking]] · [[optimization]] · [[postgres-functions-triggers]] · [[postgresql-deep]] · [[sql-security]] · [[zero-downtime-migrations]] · [[integration-testing]] · [[mutation-load-testing]] · [[testing]]

### Без уровня / All (8)

[[LICENSE]] · [[00_overview]] · [[01_language-map]] · [[04_interview-prep]] · [[05_topics-by-priority]] · [[99_reading-list]] · [[code-review]] · [[vertical-slice-handler]]

