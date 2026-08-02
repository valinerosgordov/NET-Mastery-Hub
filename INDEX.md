# 📑 INDEX — оглавление vault'а

> Автогенерируется скриптом `Scripts/generate_index.ps1` из frontmatter (`level:`) и tagline'ов файлов. **Не редактировать руками** — перегенерировать после изменений.
>
> **162 файлов / 5,8 MB** · обновлено 2026-07-02

---

## По папкам

### LearningPath — 10 файлов / 131 KB

| Файл | Уровень | KB | Описание |
|---|---|---:|---|
| [[00_overview|00_overview.md]] | All | 5,6 | Главная навигация по vault. Гайд "с чего начать" для разных уровней. Все ссылки ведут на актуальные файлы в... |
| [[01_language-map|01_language-map.md]] | All | 12,8 | Карта самого языка C# (без ASP.NET / EF / архитектуры). Зеркалит таксономию «Complete C# 2026 Cheat Sheet» ... |
| [[02_junior-to-middle|02_junior-to-middle.md]] | Junior to Middle | 10,9 | Roadmap для перехода из junior в middle. Ориентир: 3-6 месяцев active learning. Все ссылки ведут на актуаль... |
| [[03_middle-to-senior|03_middle-to-senior.md]] | Middle to Senior | 13,2 | Roadmap для Senior-уровня. Senior — не "знает всё", а думает системно и принимает trade-offs. Этот путь идё... |
| [[04_interview-prep|04_interview-prep.md]] | All | 21 | Подготовка к техническому собеседованию на .NET Backend позиции. От Junior до Senior. Что повторить, какие ... |
| [[05_topics-by-priority|05_topics-by-priority.md]] | All | 9,9 | Рейтинг тем по value vs effort. Если хочешь точечно прокачать — здесь приоритеты. |
| [[09_senior-tips-cheatsheet|09_senior-tips-cheatsheet.md]] | — | 13,8 | Концентрат Senior-практик: allocation-free паттерны (`Span<T>`, `ArrayPool<T>`, `FrozenDictionary`), async-... |
| [[10_interview-behavioral|10_interview-behavioral.md]] | — | 6,5 | Senior behavioral-вопросы и каркас ответов (STAR): прод-инциденты, решения при неполных данных, осознанный ... |
| [[99_reading-list|99_reading-list.md]] | All | 8,6 | Курируемый список внешних .NET-ресурсов для Senior-level чтения: блоги (antondevtips, Milan Jovanović), Tel... |
| [[case-studies-top7|case-studies-top7.md]] | Middle to Senior | 29,1 | Дополнение к топ-7 наиболее важным файлам vault'а: реальные production кейсы, какие проблемы возникают, как... |

### CSharp — 41 файлов / 2 442 KB

| Файл | Уровень | KB | Описание |
|---|---|---:|---|
| [[anonymous-types|Junior/anonymous-types.md]] | Junior | 105,2 | `new { Name = "John", Age = 30 }` — типы, рождённые компилятором на лету, без явного `class` объявления. Вс... |
| [[collections-linq|Junior/collections-linq.md]] | Junior | 57,2 | Главные коллекции BCL и LINQ как универсальный язык pipeline'ов. `List<T>`, `Dictionary<TKey,TValue>`, `Has... |
| [[csharp-basics|Junior/csharp-basics.md]] | Junior | 88,6 | Стартовая точка для Junior. Не справочник синтаксиса — учебник с акцентом на «почему C# устроен именно так»... |
| [[datetime-timezones|Junior/datetime-timezones.md]] | Junior | 105,1 | Самая ошибочная тема в backend-коде. `DateTime`, `DateTimeOffset`, `DateOnly`, `TimeOnly`, IANA vs Windows ... |
| [[debugging-basics|Junior/debugging-basics.md]] | Junior | 116,8 | Как находить баги в .NET-коде быстро. Breakpoints (все виды), stepping, inspect state, call stack, exceptio... |
| [[dotnet-cli-getting-started|Junior/dotnet-cli-getting-started.md]] | Junior | 113,2 | Кросс-платформенный командный интерфейс к .NET. `dotnet` — единственный обязательный инструмент: создание п... |
| [[enums-flags|Junior/enums-flags.md]] | Junior | 101,1 | Типобезопасные константы вместо magic numbers и строк. `enum`, `[Flags]`, bitwise операции, конверсии в стр... |
| [[extension-methods|Junior/extension-methods.md]] | Junior | 103,6 | `obj.MyMethod()` для типов, которые ты не контролируешь. Под капотом — обычные статические методы. Сахарный... |
| [[iterators-yield|Junior/iterators-yield.md]] | Junior | 101,6 | Метод, который возвращает по одному элементу за раз, а не материализует всю коллекцию. `yield return`, `IEn... |
| [[naming-conventions|Junior/naming-conventions.md]] | Junior | 105,7 | PascalCase, camelCase, и почему `Id` лучше `ID`. Microsoft Framework Design Guidelines, naming patterns (As... |
| [[oop|Junior/oop.md]] | Junior | 90,2 | Способ организации кода через объекты, объединяющие данные и поведение. Encapsulation, Inheritance, Polymor... |
| [[strings-regex|Junior/strings-regex.md]] | Junior | 92,1 | Самый частый тип в любом коде — и самый недооценённый. Immutability, encoding (UTF-8/UTF-16), comparison or... |
| [[tuples-deconstruction|Junior/tuples-deconstruction.md]] | Junior | 119,1 | Modern C# everyday feature. ValueTuple, named tuples, deconstruction, multiple return values. Появилось в C... |
| [[attributes-metadata|Middle/attributes-metadata.md]] | Middle | 40 | Декларации, прикрепляемые к коду и читаемые в runtime / compile-time. Built-in атрибуты, custom attributes,... |
| [[bcl-essentials|Middle/bcl-essentials.md]] | Middle | 23 | Canonical-сборник по «мелким» типам BCL, которые используются в каждом проекте, но обычно известны фрагмент... |
| [[delegates-events|Middle/delegates-events.md]] | Middle | 37,2 | Type-safe function pointers + Observer pattern. `Func`/`Action`/`Predicate`, multicast delegates, `event` k... |
| [[dispose-pattern|Middle/dispose-pattern.md]] | Middle | 46,1 | Детерминированное освобождение неуправляемых ресурсов в managed runtime. `IDisposable`, `using`, `IAsyncDis... |
| [[equality-comparison|Middle/equality-comparison.md]] | Middle | 31,8 | Reference equality vs value equality, контракт `Equals`/`GetHashCode`, `IEquatable<T>`, `IComparable<T>`, c... |
| [[error-handling|Middle/error-handling.md]] | Middle | 42,1 | Exception hierarchy, try/catch/finally, exception filters, custom exceptions, Result pattern, ASP.NET Core ... |
| [[generics-deep|Middle/generics-deep.md]] | Middle | 34,2 | Type parameters, constraints (`where T : ...`), variance (`in`/`out`), generic methods, generic math (.NET ... |
| [[indexers-operators|Middle/indexers-operators.md]] | Middle | 31 | Объект ведёт себя как массив или встроенный тип. `this[index]` indexer + operator overloading (`+`, `-`, `=... |
| [[io-streams|Middle/io-streams.md]] | Middle | 37,8 | Универсальная абстракция для byte sequences: файлы, network, memory, pipes. `Stream` hierarchy, `FileStream... |
| [[keywords-reference|Middle/keywords-reference.md]] | Middle | 31,6 | Все 100+ keywords и contextual keywords языка с группировкой по назначению. От `abstract` до `yield`, modif... |
| [[modern-features|Middle/modern-features.md]] | Middle | 39,4 | Records, `init`, pattern matching, primary constructors, raw string literals, collection expressions, gener... |
| [[nullable-types|Middle/nullable-types.md]] | Middle | 30,6 | Два разных механизма с одним именем. `Nullable<T>` (value types, .NET 2.0+) — runtime feature; Nullable Ref... |
| [[numeric-types-math|Middle/numeric-types-math.md]] | Middle | 37,9 | Integer types, floating-point, decimal, BigInteger, generic math (.NET 7+). Когда `double`, когда `decimal`... |
| [[serialization-deep|Middle/serialization-deep.md]] | Middle | 26,9 | Canonical-файл по сериализации в .NET: System.Text.Json как дефолт 2026 (options, конвертеры, полиморфизм, ... |
| [[async-threading|Senior/async-threading.md]] | Senior | 67,6 | `async/await` глубоко, Task vs ValueTask, threading primitives, `lock`/Mutex/Semaphore, `Parallel`, `Channe... |
| [[cli-tools-scripting|Senior/cli-tools-scripting.md]] | Senior | 40,8 | Top-level statements, `dotnet script`, `System.CommandLine`, Spectre.Console, Native AOT для CLI. Замена Ba... |
| [[csharp-language-design|Senior/csharp-language-design.md]] | Senior | 41,6 | Принципы проектирования C#, design committee, эволюция через 13 версий, trade-offs. Зачем добавили (и не до... |
| [[csharp-vs-other-langs|Senior/csharp-vs-other-langs.md]] | Senior | 47,5 | Конкретные сравнения C# с Java, Kotlin, TypeScript, Go, Rust, Python, F#. Где C# выигрывает, где проигрывае... |
| [[design-patterns|Senior/design-patterns.md]] | Senior | 54,7 | SOLID principles, GoF patterns, modern .NET equivalents. Какие patterns актуальны до сих пор, какие устарел... |
| [[desktop-frameworks|Senior/desktop-frameworks.md]] | Senior | 44,8 | WPF (mature Windows), WinUI 3 (modern Windows), WinForms (legacy), MAUI (cross-platform Microsoft), Avaloni... |
| [[fenwick-bit|Senior/fenwick-bit.md]] | Senior | 13,3 | Binary Indexed Tree: prefix-sum + point-update за O(log n) на плоском `int[]`, без node objects и без аллок... |
| [[functional-csharp|Senior/functional-csharp.md]] | Senior | 48,4 | Pure functions, immutability, higher-order functions, monads (`Option<T>`, `Result<T>`), function compositi... |
| [[gof-patterns-extended|Senior/gof-patterns-extended.md]] | Senior | 60,5 | Полный справочник 23 GoF design patterns с C# implementation, modern .NET equivalents, real-world usage. Re... |
| [[memory-pooling|Senior/memory-pooling.md]] | Senior | 38,7 | Сокращение allocation pressure через reuse: `ArrayPool<T>`, `MemoryPool<T>`, `ObjectPool<T>`, `Span<T>`/`Me... |
| [[reflection-expression-trees|Senior/reflection-expression-trees.md]] | Senior | 51,9 | `System.Reflection` — runtime introspection, dynamic invocation. Expression trees — lambda как data, founda... |
| [[source-generators|Senior/source-generators.md]] | Senior | 50,1 | Roslyn-based components, генерирующие C# code на compile-time на основе AST + metadata. Замена reflection д... |
| [[types-and-memory|Senior/types-and-memory.md]] | Senior | 55,3 | Value types vs reference types, stack vs heap, boxing/unboxing, struct semantics, ref struct, escape analys... |
| [[unsafe-pointers|Senior/unsafe-pointers.md]] | Senior | 37,6 | `unsafe` context, raw pointers, `fixed`, `stackalloc`, function pointers, P/Invoke marshaling. Когда manage... |

### Runtime — 9 файлов / 323 KB

| Файл | Уровень | KB | Описание |
|---|---|---:|---|
| [[memory-stack-heap|Junior/memory-stack-heap.md]] | Junior | 30,8 | Где живут переменные, как работает копирование, что такое boxing, почему `string` immutable. Введение перед... |
| [[runtime-basics|Junior/runtime-basics.md]] | Junior | 28,4 | Что происходит когда запускается C# программа: CLR, JIT compilation, GC, managed memory. Bird's-eye view пе... |
| [[threading-basics|Middle/threading-basics.md]] | Middle | 28,8 | Фундамент многопоточности перед async/await. Что такое Thread, ThreadPool, Task Parallel Library, Parallel.... |
| [[compilation-jit|Senior/compilation-jit.md]] | Senior | 38,9 | Полный pipeline исполнения: Roslyn компилирует C# в IL, CLR грузит assembly и MethodTable, RyuJIT через Tie... |
| [[concurrency-atomics|Senior/concurrency-atomics.md]] | Senior | 40,6 | Самая глубокая заметка по lock-free и low-level concurrency. Цель — закрыть всё, что нужно знать Senior'у п... |
| [[diagnostics-tools|Senior/diagnostics-tools.md]] | Senior | 33 | Полный гайд по production troubleshooting в .NET. Закрывает: dotnet-counters/trace/dump/gcdump/monitor, Eve... |
| [[gc-memory|Senior/gc-memory.md]] | Senior | 54,9 | Это самая глубокая заметка по runtime в этом vault. Цель — закрыть все вопросы по памяти, которые задают на... |
| [[interop-pinvoke|Senior/interop-pinvoke.md]] | Senior | 30,2 | Полный гайд по работе с native кодом из .NET. Закрывает: P/Invoke, `LibraryImport` source generator (.NET 7... |
| [[span-layout|Senior/span-layout.md]] | Senior | 37,3 | Самая глубокая заметка по работе с памятью без аллокаций. Закрывает Span/Memory, struct layout, SIMD/Vector... |

### AspNetCore — 23 файлов / 703 KB

| Файл | Уровень | KB | Описание |
|---|---|---:|---|
| [[http-fundamentals|Junior/http-fundamentals.md]] | Junior | 27,7 | Без понимания HTTP — невозможно делать качественный backend. Status codes, methods, headers, HTTPS, CORS — ... |
| [[aspnet-controllers-routing|Middle/aspnet-controllers-routing.md]] | Middle | 27,2 | Когда Controllers, когда Minimal API. Routing patterns, model binding, return types, action filters, MapGro... |
| [[aspnet-dependency-injection-deep|Middle/aspnet-dependency-injection-deep.md]] | Middle | 27 | DI lifetimes deep, IServiceScopeFactory, keyed services (.NET 8+), TimeProvider, captive dependencies, asyn... |
| [[aspnet-error-handling|Middle/aspnet-error-handling.md]] | Middle | 26,8 | Exception handling middleware, IExceptionHandler (.NET 8+), ProblemDetails RFC 7807, error pages, validatio... |
| [[aspnet-rate-limiting|Middle/aspnet-rate-limiting.md]] | Middle | 26,6 | Built-in rate limiting middleware (.NET 7+), 4 algorithms (fixed, sliding, token bucket, concurrency), per-... |
| [[fluent-validation|Middle/fluent-validation.md]] | Middle | 24,2 | Стандарт для validation в .NET 2026. vs DataAnnotations: гибче, тестируемо, async-friendly. Closes пробел "... |
| [[http-client-resilience|Middle/http-client-resilience.md]] | Middle | 27,4 | Canonical-файл по HTTP-клиенту в .NET: почему `new HttpClient()` на каждый запрос кладёт прод, почему singl... |
| [[object-mapping|Middle/object-mapping.md]] | Middle | 25,1 | Daily work каждого ASP.NET проекта: DTO ↔ Domain ↔ ViewModel mapping. Closes пробел "копирую properties вру... |
| [[api-design|Senior/api-design.md]] | Senior | 24,5 | Minimal API — легковесные, меньше boilerplate, хорошо для простых CRUD и прототипов. Controllers — полная м... |
| [[auth-security|Senior/auth-security.md]] | Senior | 55,6 | Authentication — кто ты? (identity, JWT, Cookie). Authorization — что тебе можно? (roles, policies, claims). |
| [[blazor-server|Senior/blazor-server.md]] | Senior | 30,9 | Web UI на C#/Razor с рендером на сервере и доставкой diff'ов через SignalR-circuit; разбор render modes, уп... |
| [[blazor-wasm|Senior/blazor-wasm.md]] | Senior | 34,1 | C# в браузере через WebAssembly. Альтернатива Blazor Server и React/Angular/Vue. Закрывает: WASM модель, ho... |
| [[caching|Senior/caching.md]] | Senior | 31,5 | Многоуровневое кэширование (`IMemoryCache`, Redis, HybridCache, Output Cache, CDN) и защита API через Rate ... |
| [[di-configuration|Senior/di-configuration.md]] | Senior | 24,3 | DI-контейнер ASP.NET Core, lifetimes, Keyed Services и типизированная конфигурация через `IOptions` с учёто... |
| [[graphql|Senior/graphql.md]] | Senior | 31,5 | Полный гайд по GraphQL в .NET. Закрывает: HotChocolate v15 (актуальная версия 2026), code-first vs schema-f... |
| [[hosting-background|Senior/hosting-background.md]] | Senior | 38,2 | Фоновая обработка через `BackgroundService` с `Channel<T>` и `PeriodicTimer`, управление жизненным циклом п... |
| [[kestrel-as-raw-host|Senior/kestrel-as-raw-host.md]] | Senior | 51,4 | Как использовать Kestrel напрямую — без MVC, без endpoint routing — когда ты сам строишь HTTP-framework пов... |
| [[logging-observability|Senior/logging-observability.md]] | Senior | 17,1 | Structured logging сохраняет данные как key-value, а не как строку. Позволяет фильтровать, искать, агрегиро... |
| [[native-aot|Senior/native-aot.md]] | Senior | 22,4 | AOT-компиляция в single executable без JIT и CLR ради мгновенного startup и малого footprint; цена — trimmi... |
| [[pipeline-middleware|Senior/pipeline-middleware.md]] | Senior | 42,2 | Полный гайд по ASP.NET Core pipeline. Закрывает: middleware patterns deep, IExceptionHandler (.NET 8+), Out... |
| [[resilience|Senior/resilience.md]] | Senior | 24,2 | Устойчивость к сбоям зависимостей через Polly v8 — Retry, Timeout, Circuit Breaker, Hedging, Bulkhead и Fal... |
| [[security-practices|Senior/security-practices.md]] | Senior | 28,6 | Прикладные приёмы защиты кода: timing-safe сравнение токенов (`FixedTimeEquals`), SHA256-хэширование секрет... |
| [[signalr|Senior/signalr.md]] | Senior | 34,1 | Полный гайд по standalone SignalR в .NET 10. Закрывает: Hub методы, transport selection (WebSocket/SSE/Long... |

### EFCore — 13 файлов / 428 KB

| Файл | Уровень | KB | Описание |
|---|---|---:|---|
| [[ef-basics|Junior/ef-basics.md]] | Junior | 37,6 | DbContext, DbSet, основные CRUD операции, миграции, конфигурация. Введение перед `Middle/dapper-comparison.... |
| [[ef-crud-queries|Junior/ef-crud-queries.md]] | Junior | 31,7 | Where, Select, OrderBy, FirstOrDefault, joins basics, includes, aggregations. Practical querying для daily ... |
| [[dapper-comparison|Middle/dapper-comparison.md]] | Middle | 23,8 | Dapper — micro-ORM: тонкий wrapper над ADO.NET, raw SQL + mapping. Часто требуют в вакансиях как альтернати... |
| [[ef-bulk-operations|Middle/ef-bulk-operations.md]] | Middle | 28,6 | Bulk INSERT/UPDATE/DELETE без change tracking. ExecuteUpdate/ExecuteDelete .NET 7+, EFCore.BulkExtensions, ... |
| [[ef-loading-strategies|Middle/ef-loading-strategies.md]] | Middle | 27,4 | Как загружать связанные данные правильно: Include vs Select vs SplitQuery vs explicit loading. Closes пробе... |
| [[ef-transactions-concurrency|Middle/ef-transactions-concurrency.md]] | Middle | 29,1 | Транзакции в EF Core, isolation levels, optimistic concurrency tokens, deadlock handling, distributed trans... |
| [[ef-value-converters|Middle/ef-value-converters.md]] | Middle | 15,6 | ValueConverter, ValueComparer, owned types, JSON columns (.NET 7+), backing fields, computed columns. Всё ч... |
| [[basics-tracking|Senior/basics-tracking.md]] | Senior | 35,7 | Глубокий гайд по фундаментальным механизмам EF Core. Закрывает: DbContext lifecycle, Change Tracker внутрен... |
| [[concurrency|Senior/concurrency.md]] | Senior | 40,7 | Полный гайд по concurrent доступу к данным через EF Core. Закрывает: optimistic / pessimistic locking, isol... |
| [[ef-patterns|Senior/ef-patterns.md]] | Senior | 41,5 | Production-grade паттерны на EF Core. Закрывает: Repository / Unit of Work, Soft Delete, Audit interceptors... |
| [[migrations|Senior/migrations.md]] | Senior | 40,1 | Полный гайд по управлению схемой БД в .NET. Закрывает: EF Core migrations, idempotent scripts, zero-downtim... |
| [[queries-performance|Senior/queries-performance.md]] | Senior | 33,6 | Performance-разбор read-side EF Core: устранение N+1 через `Include`/projection/split queries, `AsNoTrackin... |
| [[relationships|Senior/relationships.md]] | Senior | 42,7 | Полный гайд по связям и типам в EF Core. Закрывает: все relationships (1:1/1:N/N:N), TPH/TPT/TPC inheritanc... |

### SQL — 9 файлов / 253 KB

| Файл | Уровень | KB | Описание |
|---|---|---:|---|
| [[sql-basics|Junior/sql-basics.md]] | Junior | 29,1 | Базовые понятия реляционных БД и SQL для разработчиков. Закрывает пробел "что такое SQL" перед optimization... |
| [[indexes-deep|Middle/indexes-deep.md]] | Middle to Senior | 25 | Глубокий гайд по индексам в реляционных БД. Что такое индекс изнутри, какие виды бывают, когда применять, к... |
| [[eav-flexible-store-indexing|Senior/eav-flexible-store-indexing.md]] | Senior | 24,4 | Как держать EAV-таблицу (`_values` с типизированными колонками `_Long`/`_String`/`_DateTimeOffset`/`_Boolea... |
| [[mvcc-and-locking|Senior/mvcc-and-locking.md]] | Senior | 29,1 | MVCC (читатели не блокируют писателей), табличные и строчные lock modes, `SKIP LOCKED` для очередей без бро... |
| [[optimization|Senior/optimization.md]] | Senior | 47,2 | Индексы (clustered/covering/filtered, leftmost prefix), чтение execution plan и join-алгоритмы, паттерны за... |
| [[postgres-functions-triggers|Senior/postgres-functions-triggers.md]] | Senior | 17,1 | PL/pgSQL функции vs процедуры, volatility-категории, триггеры (BEFORE/AFTER, аудит, `updated_at`), вызов ra... |
| [[postgresql-deep|Senior/postgresql-deep.md]] | Senior | 44 | Production-паттерны Npgsql (`NpgsqlDataSource`, multiplexing, bulk `COPY`, pipelining), JSONB и GIN-индексы... |
| [[sql-security|Senior/sql-security.md]] | Senior | 17,4 | От инъекций защищает любой параметр, но `AddWithValue` выводит тип из значения (`string` → Unicode `nvarcha... |
| [[zero-downtime-migrations|Senior/zero-downtime-migrations.md]] | Senior | 20 | Как менять схему большой таблицы под нагрузкой: карта стоимости DDL, `lock_timeout` + retry против head-of-... |

### Architecture — 16 файлов / 584 KB

| Файл | Уровень | KB | Описание |
|---|---|---:|---|
| [[architecture-basics|Junior/architecture-basics.md]] | Junior | 43,3 | Зачем структурировать код, что такое слои, separation of concerns, dependency direction. Введение перед `Mi... |
| [[microservices-vs-monolith|Middle/microservices-vs-monolith.md]] | Middle to Senior | 23,8 | Когда монолит, когда микросервисы, когда modular monolith. Реальные trade-offs, не маркетинг. Закрывает: ти... |
| [[patterns-decision-guide|Middle/patterns-decision-guide.md]] | Middle to Senior | 36,6 | Главный навигационный файл vault'а: под какую задачу что выбрать. Связывает GoF паттерны, архитектурные сти... |
| [[real-world-scenarios|Middle/real-world-scenarios.md]] | Middle to Senior | 49,6 | Сценарий → решение. От меню навигации до микросервисного e-commerce. От чего зависит выбор архитектуры, как... |
| [[agent-safe-architecture|Senior/agent-safe-architecture.md]] | Senior | 22,3 | Прозаические правила в `AGENTS.md` гниют под context rot: чем длиннее сессия, тем дальше инструкция уезжает... |
| [[arch-tests|Senior/arch-tests.md]] | Senior | 23,2 | Правила архитектуры (Domain не зависит от Infrastructure, naming conventions, module boundaries) как исполн... |
| [[architecture-decisions|Senior/architecture-decisions.md]] | Senior | 36,8 | Полный гайд по ADR: что это, зачем, когда писать, как структурировать, как поддерживать. Закрывает: lifecyc... |
| [[architecture-patterns|Senior/architecture-patterns.md]] | Senior | 52,7 | По материалам: [N-Layered vs Clean vs VSA](https://antondevtips.com/blog/n-layered-vs-clean-vs-vertical-sli... |
| [[cqrs-mediatr|Senior/cqrs-mediatr.md]] | Senior | 25,2 | Разделение Command/Query и развязка sender'а от handler'а через mediator с pipeline behaviors (validation, ... |
| [[ddd|Senior/ddd.md]] | Senior | 64,5 | Tactical DDD (Entity, VO, Aggregate) — это про код. |
| [[distributed-systems|Senior/distributed-systems.md]] | Senior | 64,9 | Один инстанс — N/A (нет distributed). Master-replica с synchronous replication — CP (writes ждут реплики, п... |
| [[eip-content-based-router|Senior/eip-content-based-router.md]] | Senior | 20,6 | CBR выбирает ровно один маршрут из многих (first-match-wins). Message Filter — частный случай с одной ветко... |
| [[solid|Senior/solid.md]] | Senior | 24 | Пять принципов SOLID (SRP, OCP, LSP, ISP, DIP) плюс DRY/KISS/YAGNI с примерами на C#, маркерами нарушений и... |
| [[system-design|Senior/system-design.md]] | Senior | 31,9 | Фреймворк system design интервью (requirements, capacity estimation, tradeoffs) плюс готовые шаблоны: rate ... |
| [[twelve-factor-app|Senior/twelve-factor-app.md]] | Senior | 35,3 | Manifesto от Heroku (2011), стал industry standard. 12 принципов для apps which можно reliably deploy в clo... |
| [[webai-csharp-architecture|Senior/webai-csharp-architecture.md]] | Senior | 29,3 | Landing page generator: user fills form -> AI generates texts + images -> site published instantly. |

### Quality — 5 файлов / 134 KB

| Файл | Уровень | KB | Описание |
|---|---|---:|---|
| [[clean-code|Junior/clean-code.md]] | Junior | 34,6 | Что такое читаемый код, как его писать, и почему это самая недооценённая навык программиста. Базовые принци... |
| [[refactoring|Middle/refactoring.md]] | Middle to Senior | 29,9 | Систематические техники для улучшения кода без изменения внешнего поведения. Каталог refactorings, code sme... |
| [[code-quality|Senior/code-quality.md]] | Senior | 21,3 | Автоматический enforcement стиля и качества кода: `.editorconfig` + Roslyn/Meziantou/SonarAnalyzer analyzer... |
| [[static-analysis|Senior/static-analysis.md]] | Senior | 23,1 | Полный гайд по static analysis в .NET. Закрывает: встроенные analyzers, SonarAnalyzer, Meziantou, Roslynato... |
| [[code-review|code-review.md]] | All | 25,5 | Полный гайд по code review: что искать, как давать feedback, как принимать критику, anti-patterns, growth-o... |

### Testing — 5 файлов / 152 KB

| Файл | Уровень | KB | Описание |
|---|---|---:|---|
| [[testing-fundamentals|Junior/testing-fundamentals.md]] | Junior | 40,8 | Что такое тест, какие виды бывают, зачем каждый нужен, где применяется. Базовые концепты для джуна, система... |
| [[mocking-strategies|Middle/mocking-strategies.md]] | Middle to Senior | 22,7 | Когда мокать, как мокать, а главное — когда НЕ мокать. Test doubles разных видов, NSubstitute vs Moq, anti-... |
| [[integration-testing|Senior/integration-testing.md]] | Senior | 26,3 | Глубокий гайд по integration tests в .NET. Закрывает: WebApplicationFactory, Testcontainers, тестирование с... |
| [[mutation-load-testing|Senior/mutation-load-testing.md]] | Senior | 23,1 | Mutation testing проверяет качество тестов. Load testing — performance под нагрузкой. Закрывает: Stryker.NE... |
| [[testing|Senior/testing.md]] | Senior | 39,2 | Стратегия и tooling-стек тестирования в .NET 2026: пирамида vs алмаз, xUnit/TUnit, NSubstitute, Testcontain... |

### Performance — 12 файлов / 246 KB

| Файл | Уровень | KB | Описание |
|---|---|---:|---|
| [[performance-fundamentals|Junior/performance-fundamentals.md]] | Junior | 24,5 | Базовое понимание perfomance. Что такое "медленно", как мерить, почему оптимизировать преждевременно вредно... |
| [[async-performance|Middle/async-performance.md]] | Middle to Senior | 13,4 | Async-specific performance: ValueTask, TaskCompletionSource, async pitfalls, ConfigureAwait. Когда async по... |
| [[bottleneck-analysis|Middle/bottleneck-analysis.md]] | Middle to Senior | 12,2 | Систематический подход к нахождению где система тормозит. CPU, memory, I/O, network, DB — как определить ка... |
| [[caching-strategies|Middle/caching-strategies.md]] | Middle to Senior | 20,5 | Какие виды кеша есть, когда какой применять, как избежать stale data, как не сделать хуже. Глубокий разбор ... |
| [[lazy-eager-loading|Middle/lazy-eager-loading.md]] | Middle | 13,2 | Когда загружать данные сразу, когда откладывать. Trade-offs для EF Core, lazy initialization, eager configu... |
| [[optimization-patterns|Middle/optimization-patterns.md]] | Middle to Senior | 25,5 | Practical optimization patterns для типичных задач. Когда применять, что измерять, какие trade-offs. Не "ma... |
| [[capacity-planning|Senior/capacity-planning.md]] | Senior | 10,6 | Сколько ресурсов нужно для текущей и будущей нагрузки. Расчёт CPU, memory, network, БД для заданного traffic. |
| [[hft-low-latency|Senior/hft-low-latency.md]] | Senior | 40,2 | Микросекундный hot-path в .NET: allocation-free через `Span<T>`/`stackalloc`/`ArrayPool<T>`, lock-free стру... |
| [[memory-profiling|Senior/memory-profiling.md]] | Senior | 17,6 | Как находить memory leaks, GC pressure, оптимизировать heap usage. Tools, workflow, decoding heap snapshots. |
| [[performance-budgets|Senior/performance-budgets.md]] | Senior | 12,9 | Performance как functional requirement — устанавливаем cost targets, мониторим, alert когда превышаем. |
| [[performance|Senior/performance.md]] | Senior | 35,3 | Workflow измерения и оптимизации .NET: точные замеры через BenchmarkDotNet (`[MemoryDiagnoser]`), профилиро... |
| [[threadpool-starvation-hill-climbing|Senior/threadpool-starvation-hill-climbing.md]] | Senior | 20,5 | Самый коварный production-инцидент в .NET: throughput падает в разы, p99 уходит в секунды, а CPU при этом 8... |

### Infrastructure — 14 файлов / 506 KB

| Файл | Уровень | KB | Описание |
|---|---|---:|---|
| [[docker-for-dev|Junior/docker-for-dev.md]] | Junior | 28,7 | Что такое контейнеры, daily команды, Dockerfile основы, docker-compose для local dev. Введение перед `Senio... |
| [[project-setup-basics|Junior/project-setup-basics.md]] | Junior | 27,9 | dotnet CLI, solution + projects organization, .gitignore, basic git workflow для C# проектов. Введение пере... |
| [[cicd-github-actions|Middle/cicd-github-actions.md]] | Middle | 21,9 | Continuous Integration / Continuous Delivery для .NET: build, test, lint, security scan, deploy. GitHub Act... |
| [[kubernetes|Middle/kubernetes.md]] | Middle to Senior | 23 | Что Senior .NET должен знать про Kubernetes: концепты, Pod/Deployment/Service, deploy .NET app, health chec... |
| [[api-gateway|Senior/api-gateway.md]] | Senior | 32,2 | Single entry point для микросервисов. Routing, auth, rate limiting, transformation — в одном месте. Closes ... |
| [[docker|Senior/docker.md]] | Senior | 66,5 | Самая глубокая заметка по контейнеризации в этом vault. Цель — закрыть все вопросы по Docker, которые задаю... |
| [[ipc-named-pipes-grpc|Senior/ipc-named-pipes-grpc.md]] | Senior | 44,2 | Передача данных между процессами в .NET: Named Pipes, Memory-Mapped Files (zero-copy ring buffer), Anonymou... |
| [[llm-rag-patterns|Senior/llm-rag-patterns.md]] | Senior | 48,6 | Production-RAG на .NET: chunking, embeddings, hybrid search (BM25 + dense + RRF), reranking, tool use и str... |
| [[messaging|Senior/messaging.md]] | Senior | 45,2 | Асинхронная коммуникация через message brokers: RabbitMQ vs Kafka, MassTransit, Outbox pattern и Dead Lette... |
| [[nosql-databases|Senior/nosql-databases.md]] | Senior | 24,6 | Когда NoSQL > SQL и какой NoSQL для какой задачи. MongoDB / Redis / Cosmos DB / DynamoDB / Cassandra — реал... |
| [[observability|Senior/observability.md]] | Senior | 32,6 | Три pillars наблюдаемости (logs, metrics, traces) на OpenTelemetry для .NET: экспорт в Prometheus / Grafana... |
| [[project-setup|Senior/project-setup.md]] | Senior | 31,7 | Полный гайд по starting up .NET проекта с production-grade defaults. Закрывает: Directory.Build.props, .edi... |
| [[semantic-kernel|Senior/semantic-kernel.md]] | Senior | 43 | Semantic Kernel как единый AI SDK от Microsoft для .NET (LLM, embeddings, agents) плюс vector search по смы... |
| [[wpf-production|Senior/wpf-production.md]] | Senior | 36,2 | Современный production-стек WPF на .NET 10: CommunityToolkit.Mvvm (source generators), WPF-UI (Fluent Desig... |

### Snippets — 5 файлов / 33 KB

| Файл | Уровень | KB | Описание |
|---|---|---:|---|
| [[crud-example|crud-example.md]] | Senior | 13,2 | Сквозной copy-paste template всех четырёх CRUD-операций: путь от Minimal API endpoint через explicit Handle... |
| [[efcore-queries|efcore-queries.md]] | Middle to Senior | 5 | Производительные EF Core read-паттерны: `AsNoTracking` + проекция в DTO, offset- и keyset-пагинация, `AsSpl... |
| [[mediatr-handlers|mediatr-handlers.md]] | Middle to Senior | 4,4 | CQRS-обвязка на MediatR: Command/Query handlers, возвращающие `Result<T>`, FluentValidation validator и `IP... |
| [[result-pattern|result-pattern.md]] | Middle | 3,7 | Кастомный `Result<T>` + `Error` record как замена исключениям: фабрики Ok/Fail, `Match()` для маппинга в HT... |
| [[wpf-viewmodel|wpf-viewmodel.md]] | Middle | 6,5 | WPF на CommunityToolkit.Mvvm: `ObservableObject` с `[ObservableProperty]`/`[RelayCommand]`, async-safe загр... |

---

## По уровню (компактно)

### Junior (25)

[[architecture-basics]] · [[http-fundamentals]] · [[anonymous-types]] · [[collections-linq]] · [[csharp-basics]] · [[datetime-timezones]] · [[debugging-basics]] · [[dotnet-cli-getting-started]] · [[enums-flags]] · [[extension-methods]] · [[iterators-yield]] · [[naming-conventions]] · [[oop]] · [[strings-regex]] · [[tuples-deconstruction]] · [[ef-basics]] · [[ef-crud-queries]] · [[docker-for-dev]] · [[project-setup-basics]] · [[performance-fundamentals]] · [[clean-code]] · [[memory-stack-heap]] · [[runtime-basics]] · [[sql-basics]] · [[testing-fundamentals]]

### Junior to Middle (1)

[[02_junior-to-middle]]

### Middle (31)

[[aspnet-controllers-routing]] · [[aspnet-dependency-injection-deep]] · [[aspnet-error-handling]] · [[aspnet-rate-limiting]] · [[fluent-validation]] · [[http-client-resilience]] · [[object-mapping]] · [[attributes-metadata]] · [[bcl-essentials]] · [[delegates-events]] · [[dispose-pattern]] · [[equality-comparison]] · [[error-handling]] · [[generics-deep]] · [[indexers-operators]] · [[io-streams]] · [[keywords-reference]] · [[modern-features]] · [[nullable-types]] · [[numeric-types-math]] · [[serialization-deep]] · [[dapper-comparison]] · [[ef-bulk-operations]] · [[ef-loading-strategies]] · [[ef-transactions-concurrency]] · [[ef-value-converters]] · [[cicd-github-actions]] · [[lazy-eager-loading]] · [[threading-basics]] · [[result-pattern]] · [[wpf-viewmodel]]

### Middle to Senior (15)

[[microservices-vs-monolith]] · [[patterns-decision-guide]] · [[real-world-scenarios]] · [[kubernetes]] · [[03_middle-to-senior]] · [[case-studies-top7]] · [[async-performance]] · [[bottleneck-analysis]] · [[caching-strategies]] · [[optimization-patterns]] · [[refactoring]] · [[efcore-queries]] · [[mediatr-handlers]] · [[indexes-deep]] · [[mocking-strategies]]

### Senior (82)

[[agent-safe-architecture]] · [[arch-tests]] · [[architecture-decisions]] · [[architecture-patterns]] · [[cqrs-mediatr]] · [[ddd]] · [[distributed-systems]] · [[eip-content-based-router]] · [[solid]] · [[system-design]] · [[twelve-factor-app]] · [[webai-csharp-architecture]] · [[api-design]] · [[auth-security]] · [[blazor-server]] · [[blazor-wasm]] · [[caching]] · [[di-configuration]] · [[graphql]] · [[hosting-background]] · [[kestrel-as-raw-host]] · [[logging-observability]] · [[native-aot]] · [[pipeline-middleware]] · [[resilience]] · [[security-practices]] · [[signalr]] · [[async-threading]] · [[cli-tools-scripting]] · [[csharp-language-design]] · [[csharp-vs-other-langs]] · [[design-patterns]] · [[desktop-frameworks]] · [[fenwick-bit]] · [[functional-csharp]] · [[gof-patterns-extended]] · [[memory-pooling]] · [[reflection-expression-trees]] · [[source-generators]] · [[types-and-memory]] · [[unsafe-pointers]] · [[basics-tracking]] · [[concurrency]] · [[ef-patterns]] · [[migrations]] · [[queries-performance]] · [[relationships]] · [[api-gateway]] · [[docker]] · [[ipc-named-pipes-grpc]] · [[llm-rag-patterns]] · [[messaging]] · [[nosql-databases]] · [[observability]] · [[project-setup]] · [[semantic-kernel]] · [[wpf-production]] · [[capacity-planning]] · [[hft-low-latency]] · [[memory-profiling]] · [[performance-budgets]] · [[performance]] · [[threadpool-starvation-hill-climbing]] · [[code-quality]] · [[static-analysis]] · [[compilation-jit]] · [[concurrency-atomics]] · [[diagnostics-tools]] · [[gc-memory]] · [[interop-pinvoke]] · [[span-layout]] · [[crud-example]] · [[eav-flexible-store-indexing]] · [[mvcc-and-locking]] · [[optimization]] · [[postgres-functions-triggers]] · [[postgresql-deep]] · [[sql-security]] · [[zero-downtime-migrations]] · [[integration-testing]] · [[mutation-load-testing]] · [[testing]]

### Без уровня / All (8)

[[00_overview]] · [[01_language-map]] · [[04_interview-prep]] · [[05_topics-by-priority]] · [[09_senior-tips-cheatsheet]] · [[10_interview-behavioral]] · [[99_reading-list]] · [[code-review]]

