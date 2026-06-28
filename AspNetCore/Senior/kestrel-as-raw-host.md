---
tags: [aspnet, kestrel, hosting, http, transport, minimal-api, framework-author, senior]
level: Senior
date: 2026-06-19
---

# Kestrel как raw HTTP host

> Как использовать Kestrel напрямую — без MVC, без endpoint routing — когда **ты сам** строишь HTTP-framework поверх транспорта. Один catch-all handler, своя маршрутизация, shared-server manager на `(host, port)`. Закрывает пробел "знаю Minimal API, но не понимаю что под ним и когда стоит спуститься на уровень ниже".

---

## Что это, зачем и когда

### Что значит "raw host"

ASP.NET Core — это слоёный пирог. Снизу вверх:

```
Kestrel (transport: sockets, HTTP/1.1, HTTP/2, HTTP/3, TLS)
   ↓
HttpContext (Request/Response абстракция)
   ↓
Middleware pipeline
   ↓
Endpoint Routing (match URL → endpoint)
   ↓
MVC / Minimal API (model binding, filters, result execution)
```

"Raw host" — это когда ты берёшь Kestrel плюс `HttpContext`, а всё выше (routing, MVC, model binding) **заменяешь своим**. Один универсальный handler ловит каждый запрос, а дальше твой код решает что делать.

### Почему вообще обходить MVC — transport coupling

Главная причина не техническая, а архитектурная. Если ты пишешь **переиспользуемый framework** (API gateway, proxy, протокол-сервер, embeddable HTTP-движок, тестовый mock-сервер), то завязка на endpoint routing и MVC означает завязку на **конкретный способ хостинга**. Твой framework начинает диктовать пользователю:

- как регистрировать маршруты (`MapGet`/`MapControllerRoute`),
- как выглядит DI-граф (controllers, filters, `IActionResult`),
- как устроена model binding.

Это и есть **transport coupling** — доменная логика framework'а намертво срастается с транспортным слоем ASP.NET Core. Заменить Kestrel на что-то другое, запустить два независимых сервера в одном процессе, или дать пользователю свою модель маршрутов — становится невозможно без переписывания.

Спускаясь к raw host, ты разрываешь эту связь: ASP.NET Core отвечает **только** за транспорт (сокеты, HTTP-парсинг, TLS, HTTP/2/3), а семантику запроса (маршрутизация, диспетчеризация, формирование ответа) владеешь ты.

### Когда это оправдано vs обычный Minimal API

> [!warning]
> Это техника для **авторов framework'ов и инфраструктуры**, не для прикладных API. Для 99% сервисов правильный ответ — Minimal API. Спускаться к raw host без причины — это reinvent endpoint routing хуже, чем его уже написали в Microsoft.

| Сценарий | Решение |
|----------|---------|
| Обычный REST/JSON API приложения | Minimal API |
| Микросервис с бизнес-логикой | Minimal API + `MapGroup` |
| Reverse proxy с конфиг-driven маршрутами | YARP (см. [[api-gateway]]) |
| **Свой routing DSL** (не URL-паттерны ASP.NET) | raw host |
| **Свой wire-протокол** поверх HTTP (RPC, streaming) | raw host |
| **Embeddable HTTP-движок** в чужом процессе | raw host |
| **Несколько независимых серверов** на разных `(host, port)` в одном процессе | raw host + server manager |
| Mock/fake сервер для интеграционных тестов с динамической логикой | raw host |

Критерий простой: **если ты можешь выразить задачу через `Map*` + middleware — используй их.** Raw host оправдан, только когда твоя модель маршрутизации/диспетчеризации фундаментально не совпадает с моделью ASP.NET Core, или когда ты строишь инфраструктуру, которая не должна знать про неё.

---

> [!question]- **Интервью: Зачем обходить MVC/endpoint routing, если есть Minimal API?**
> Не для прикладного кода — для framework-авторства. Endpoint routing завязывает framework на транспорт (transport coupling): пользователь обязан регистрировать маршруты через `Map*`, жить с DI-графом MVC, model binding. Если ты строишь gateway/proxy/protocol-server/embeddable-движок со своей моделью маршрутов, raw host (Kestrel + один catch-all handler) даёт владение семантикой запроса при том, что ASP.NET Core отвечает только за транспорт. Цена — ты сам реализуешь routing, и легко сделать его хуже встроенного.

> [!question]- **Интервью: В чём разница CreateBuilder vs CreateSlimBuilder?**
> `CreateSlimBuilder` (.NET 8+) — облегчённый хост: не подтягивает reflection-зависимые фичи (полную конфигурацию, дефолтную JSON-reflection, IIS integration, hosting startup assemblies), регистрирует минимум сервисов. Стартует быстрее, меньше памяти, AOT-friendly. Для raw host это идеально — нам не нужно ничего из MVC-обвязки, только Kestrel и DI. См. [[native-aot]].

---

## 1. CreateSlimBuilder и прямая конфигурация Kestrel

### Почему именно Slim

Для raw host мы не используем ни controllers, ни дефолтную model binding, ни значительную часть авто-регистраций обычного `CreateBuilder`. `CreateSlimBuilder` даёт ровно то, что нужно: DI, configuration, logging, Kestrel — без reflection-тяжёлого обвеса. Это также делает host AOT-совместимым (подробно — [[native-aot]], раздел про `CreateSlimBuilder`).

```csharp
var builder = WebApplication.CreateSlimBuilder(args);

builder.WebHost.ConfigureKestrel(options =>
{
    // Конкретный интерфейс + порт
    options.Listen(IPAddress.Loopback, 5000, listen =>
    {
        listen.Protocols = HttpProtocols.Http1AndHttp2;
    });

    // Все интерфейсы (0.0.0.0) на отдельном порту с TLS + HTTP/3
    options.ListenAnyIP(5001, listen =>
    {
        listen.Protocols = HttpProtocols.Http1AndHttp2AndHttp3;
        listen.UseHttps(); // dev-cert; в проде — явный сертификат, см. ниже
    });

    // Лимиты транспорта — твоя ответственность, MVC defaults здесь не действуют
    options.Limits.MaxRequestBodySize = 10 * 1024 * 1024; // 10 MB
    options.Limits.MaxConcurrentConnections = 10_000;
});

var app = builder.Build();
```

### Listen vs ListenAnyIP vs ListenLocalhost

| Метод | Биндится на | Когда |
|-------|-------------|-------|
| `Listen(IPAddress.Loopback, port)` | только 127.0.0.1 | локальный/sidecar доступ |
| `ListenLocalhost(port)` | IPv4 + IPv6 loopback | dev |
| `ListenAnyIP(port)` | 0.0.0.0 (все интерфейсы) | публичный/контейнерный сервис |
| `Listen(IPAddress.Parse("10.0.0.5"), port)` | конкретный NIC | multi-homed машины |

> [!info]
> Программный `ConfigureKestrel` **полностью игнорирует** переменную `ASPNETCORE_URLS` и флаг `--urls`, если ты вызвал `Listen*` явно. Это плюс для framework'а: твоя bind-конфигурация детерминирована и не зависит от окружения, которое настроит пользователь. Но это же — частая ловушка ("почему мой `--urls` не работает").

### Протоколы: HTTP/1.1, HTTP/2, HTTP/3

`ListenOptions.Protocols` — это `[Flags]`-enum `HttpProtocols`:

```csharp
options.ListenAnyIP(5002, listen =>
{
    listen.Protocols = HttpProtocols.Http1;                 // только HTTP/1.1
});

options.ListenAnyIP(5003, listen =>
{
    listen.Protocols = HttpProtocols.Http2;                 // только HTTP/2 (gRPC, h2c)
});

options.ListenAnyIP(5004, listen =>
{
    listen.Protocols = HttpProtocols.Http1AndHttp2AndHttp3; // negotiation через ALPN
    listen.UseHttps();
});
```

Ключевые ограничения по механизму, а не по правилу:

- **HTTP/2 и HTTP/3 практически требуют TLS.** Браузеры не делают HTTP/2 без TLS, а ALPN (выбор протокола при handshake) живёт внутри TLS. Без `UseHttps()` HTTP/3 не поднимется вообще (он по спецификации только over QUIC+TLS 1.3).
- **HTTP/3 идёт по UDP (QUIC), а не TCP.** Это значит отдельный UDP-порт в firewall и заголовок `Alt-Svc`, которым сервер сообщает клиенту "я умею h3 — переключайся". Без `Alt-Svc` клиент останется на HTTP/2.
- **h2c** (HTTP/2 cleartext, без TLS) допустим только когда клиент заранее знает, что сервер — HTTP/2 (типично для внутреннего gRPC за mesh'ем), потому что ALPN-негоциации без TLS нет.

### UseHttps в проде

Dev-cert годится только для разработки. В проде — явный сертификат:

```csharp
options.ListenAnyIP(443, listen =>
{
    listen.Protocols = HttpProtocols.Http1AndHttp2;
    listen.UseHttps(httpsOptions =>
    {
        httpsOptions.ServerCertificateSelector = (context, dnsName) =>
            CertificateStore.Resolve(dnsName); // SNI: разные серты под разные хосты
    });
});
```

`ServerCertificateSelector` через SNI (Server Name Indication) — то, что делает один порт мульти-доменным: клиент в TLS handshake присылает имя хоста, ты выбираешь сертификат. Для framework'а, хостящего много логических серверов, это часто нужнее статического `ServerCertificate`.

---

## 2. Один catch-all handler — твой framework владеет маршрутизацией

Вместо десятков `Map*` мы регистрируем **ровно один** маршрут, который ловит всё, и отдаём управление своему диспетчеру.

```csharp
var app = builder.Build();

// Минимум встроенного pipeline. Никакого UseRouting/MapControllers.
var dispatcher = app.Services.GetRequiredService<IRequestDispatcher>();

app.Map("/{**path}", (HttpContext context, CancellationToken ct) =>
    dispatcher.DispatchAsync(context, ct));

app.Run();
```

`{**path}` — catch-all сегмент: матчит любой путь любой глубины, включая пустой. Всё, что приходит, попадает в твой `IRequestDispatcher`, который сам разбирает `context.Request.Path`, `Method`, заголовки и выбирает обработчик по **твоим** правилам.

Почему один handler, а не свой `MatcherPolicy`/`EndpointDataSource`: можно встроиться в endpoint routing на уровне ниже, но тогда ты всё равно живёшь внутри его модели (endpoints, metadata, `RouteEndpoint`). Catch-all + собственный dispatcher даёт чистый раздел: ASP.NET Core довёл до тебя `HttpContext`, дальше — твой мир.

```csharp
public interface IRequestDispatcher
{
    ValueTask DispatchAsync(HttpContext context, CancellationToken ct);
}

public sealed class RequestDispatcher(IRouteTable routes, ILogger<RequestDispatcher> logger)
    : IRequestDispatcher
{
    public async ValueTask DispatchAsync(HttpContext context, CancellationToken ct)
    {
        var path = context.Request.Path.Value ?? "/";
        var method = context.Request.Method;

        if (routes.TryMatch(method, path, out var handler, out var routeValues))
        {
            await handler.HandleAsync(context, routeValues, ct).ConfigureAwait(false);
            return;
        }

        Log.NoRouteMatched(logger, method, path);
        context.Response.StatusCode = StatusCodes.Status404NotFound;
    }
}

internal static partial class Log
{
    [LoggerMessage(Level = LogLevel.Debug, Message = "No route matched {Method} {Path}")]
    public static partial void NoRouteMatched(ILogger logger, string method, string path);
}
```

---

## 3. Shared-server manager: один Kestrel на `(host, port)`

Когда framework хостит много логических серверов в одном процессе (например, плагины, каждый со своим эндпоинтом, но часть из них хочет один и тот же порт), нельзя поднять два Kestrel на одном `(host, port)` — ОС не даст забиндить порт дважды. Нужен **менеджер**: на каждую уникальную пару `(host, port)` — ровно один физический сервер, который мультиплексирует запросы между логическими потребителями.

Регистрируем как DI singleton с **lazy-стартом по требованию** через double-checked locking — сервер на порту поднимается только когда его впервые попросили, и ровно один раз даже при гонке.

```csharp
public readonly record struct ServerKey(string Host, int Port);

public interface ISharedServerManager : IAsyncDisposable
{
    ValueTask<ISharedServer> GetOrStartAsync(ServerKey key, CancellationToken ct);
}

public sealed class SharedServerManager(
    IServiceProvider services,
    ILogger<SharedServerManager> logger) : ISharedServerManager
{
    // Готовые серверы — для lock-free fast path при повторных обращениях.
    private readonly ConcurrentDictionary<ServerKey, ISharedServer> _servers = new();

    // Per-key замок старта: не держим один глобальный lock на всё время bind'а.
    private readonly ConcurrentDictionary<ServerKey, Lock> _startLocks = new();

    public async ValueTask<ISharedServer> GetOrStartAsync(ServerKey key, CancellationToken ct)
    {
        // 1-я проверка: уже запущен — без блокировки.
        if (_servers.TryGetValue(key, out var existing))
            return existing;

        var gate = _startLocks.GetOrAdd(key, static _ => new Lock());

        // Синхронная часть (проверка + резервирование) под Lock.
        // Сам асинхронный StartAsync выносим за пределы критической секции.
        ISharedServer server;
        bool needsStart;

        lock (gate)
        {
            // 2-я проверка под замком: другой поток мог успеть создать.
            if (_servers.TryGetValue(key, out existing))
                return existing;

            server = new SharedServer(key, services, logger);
            _servers[key] = server;   // публикуем до старта — но в "not-started" состоянии
            needsStart = true;
        }

        if (needsStart)
        {
            try
            {
                await server.StartAsync(ct).ConfigureAwait(false);
            }
            catch
            {
                // Откат: не оставляем в словаре полу-поднятый сервер.
                _servers.TryRemove(key, out _);
                throw;
            }
        }

        return server;
    }

    public async ValueTask DisposeAsync()
    {
        foreach (var server in _servers.Values)
            await server.DisposeAsync().ConfigureAwait(false);
        _servers.Clear();
    }
}
```

```csharp
// DI-регистрация
builder.Services.AddSingleton<ISharedServerManager, SharedServerManager>();
```

> [!warning]
> Не оборачивай асинхронный `StartAsync` непосредственно внутри `lock (gate) { ... }` — `await` под `System.Threading.Lock` (как и под `Monitor`) запрещён: продолжение может уехать на другой поток, а замок reentrancy/ownership-зависим. Паттерн выше делает синхронную публикацию под замком, а сам `await server.StartAsync` — снаружи. Цена — нужен откат при ошибке старта (видно в `catch`).

Почему double-checked locking, а не просто `ConcurrentDictionary.GetOrAdd` с фабрикой: фабрика в `GetOrAdd` может вызваться **несколько раз** при гонке (контракт это допускает), а у нас побочный эффект — bind на порт. Дважды создать "сервер" и дважды биндить порт — гонка за дефицитным ресурсом ОС. Per-key `Lock` гарантирует ровно один старт на ключ, при этом не сериализует старты **разных** портов между собой.

`SharedServer` внутри держит свой `KestrelServer`/`WebApplication` на конкретном `(host, port)` и реестр логических потребителей, между которыми диспетчеризует входящие запросы (по host-header, по префиксу пути — по твоим правилам).

---

## 4. Порядок маршрутов: literal > parameter > catch-all

Раз ты владеешь маршрутизацией, ты владеешь и её корректностью. Классический инвариант: **более специфичный маршрут должен побеждать менее специфичный**, независимо от порядка регистрации. ASP.NET Core решает это через precedence-скоринг; в своём `IRouteTable` нужно воспроизвести ту же идею.

Специфичность сегмента, от высшей к низшей:

1. **Literal** — `/users/active` (точное совпадение).
2. **Parameter** — `/users/{id}` (захват одного сегмента).
3. **Catch-all** — `/users/{**rest}` (захват хвоста любой глубины).

```csharp
public enum SegmentKind { Literal = 0, Parameter = 1, CatchAll = 2 }
```

При матчинге, когда несколько шаблонов подходят к одному пути, выбираем тот, у которого **на первом различающемся сегменте** специфичность выше (меньше `SegmentKind`). То есть `/users/active` обыграет `/users/{id}`, а `/users/{id}` обыграет `/users/{**rest}` — для запроса `/users/active`.

```csharp
public sealed class RouteTable : IRouteTable
{
    private readonly List<CompiledRoute> _routes;

    public RouteTable(IEnumerable<CompiledRoute> routes)
    {
        // Сортируем один раз при построении: специфичные — раньше.
        // Сравниваем посегментно; на первом отличии решает SegmentKind.
        _routes = routes
            .OrderBy(r => r, RouteSpecificityComparer.Instance)
            .ToList();
    }

    public bool TryMatch(
        string method,
        string path,
        out IRouteHandler handler,
        out RouteValues values)
    {
        // Список уже в порядке убывания специфичности —
        // первый матч и есть самый специфичный.
        foreach (var route in _routes)
        {
            if (route.Method == method && route.TryMatch(path, out values))
            {
                handler = route.Handler;
                return true;
            }
        }

        handler = default!;
        values = default;
        return false;
    }
}
```

Почему сортировать заранее, а не сравнивать кандидатов на каждом запросе: матчинг — это hot path. Стабильный отсортированный список превращает выбор "самого специфичного" в "первый, который совпал" — линейный проход без аллокаций и без отбора-победителя на каждом запросе.

> [!info]
> Тонкость, которую легко пропустить: специфичность сравнивается **посегментно слева направо**, а не "число литералов всего". `/a/{id}/c` (literal-param-literal) и `/a/b/{rest}` для пути `/a/b/c` — на втором сегменте literal (`b`) бьёт parameter (`{id}`), поэтому побеждает `/a/b/{rest}`, хотя у первого больше литералов суммарно. Сравнивай покомпонентно — иначе получишь неинтуитивные матчи.

---

## 5. Reference-counting lifecycle: сервер живёт, пока есть маршруты

### Почему счётчик ссылок, а не просто Start/Stop

Раздел 3 поднимал сервер на `(host, port)` по требованию. Но framework, который добавляет и убирает маршруты **в рантайме** (плагин загрузился/выгрузился, tenant пришёл/ушёл), сталкивается с обратной задачей: когда **последний** маршрут на порту снят, сокет надо освободить. Иначе порт остаётся занят мёртвым сервером — а следующая попытка забиндить тот же `(host, port)` (другой плагин, перезапуск конфигурации) упадёт с `AddressAlreadyInUse`.

Наивное "стартуем в `GetOrStart`, стопаем в `Dispose` менеджера" не годится: жизнь сервера должна совпадать с жизнью **множества его потребителей**, а не с жизнью процесса. Это классический reference counting: сервер существует, пока счётчик регистраций > 0, и детерминированно гасится на переходе 1 → 0.

> [!info]
> Это **не** GC-проблема — сокет это unmanaged-ресурс ОС, финализатор его не освободит вовремя. Нужен явный детерминированный teardown ровно в момент последнего `unregister`, синхронизированный с возможным конкурентным `register` (кто-то регистрирует новый маршрут ровно тогда, когда последний снимается).

### Регистрация возвращает токен-disposable

Самый надёжный API — register отдаёт `IAsyncDisposable`-lease; снятие маршрута = `DisposeAsync` токена. Так потребитель не может «забыть» декремент, а двойной dispose идемпотентен.

```csharp
public readonly record struct ServerKey(string Host, int Port);

public interface IRouteLease : IAsyncDisposable;

public interface IDynamicRouteManager : IAsyncDisposable
{
    // Регистрирует маршрут; поднимает сервер на (host,port), если он первый.
    // Возврат токена, чей DisposeAsync снимает маршрут и гасит сервер на 1 → 0.
    ValueTask<IRouteLease> RegisterAsync(ServerKey key, CompiledRoute route, CancellationToken ct);
}
```

### Менеджер: refcount на каждый `(host, port)`

```csharp
public sealed class DynamicRouteManager(
    IServiceProvider services,
    ILogger<DynamicRouteManager> logger) : IDynamicRouteManager
{
    private sealed class Entry(ISharedServer server)
    {
        public ISharedServer Server { get; } = server;
        public int RefCount;          // защищён Gate
        public readonly Lock Gate = new();
    }

    private readonly ConcurrentDictionary<ServerKey, Entry> _entries = new();

    public async ValueTask<IRouteLease> RegisterAsync(
        ServerKey key, CompiledRoute route, CancellationToken ct)
    {
        // Цикл нужен на случай гонки с teardown: запись могли удалить между
        // GetOrAdd и инкрементом — тогда пробуем заново с новой записью.
        while (true)
        {
            var entry = _entries.GetOrAdd(
                key, static (k, s) => new Entry(new SharedServer(k, s.services, s.logger)),
                (services, logger));

            bool isFirst;
            lock (entry.Gate)
            {
                // Если запись уже помечена к сносу (RefCount ушёл в 0 и идёт teardown),
                // TryRemove ниже её выкинет — не цепляемся за обречённую запись.
                if (entry.RefCount < 0)
                    continue; // запись «умирает» — берём свежую на следующей итерации

                isFirst = entry.RefCount == 0;
                entry.RefCount++;
            }

            if (isFirst)
            {
                try
                {
                    // Async-старт ВНЕ Lock (см. раздел 3): await под Lock запрещён.
                    await entry.Server.StartAsync(ct).ConfigureAwait(false);
                }
                catch
                {
                    // Откат инкремента и удаление полу-поднятой записи.
                    await ReleaseAsync(key, entry).ConfigureAwait(false);
                    throw;
                }
            }

            entry.Server.AddRoute(route);
            Log.RouteRegistered(logger, route.Method, key.Host, key.Port);
            return new RouteLease(this, key, entry, route);
        }
    }

    private async ValueTask ReleaseAsync(ServerKey key, Entry entry)
    {
        bool isLast;
        lock (entry.Gate)
        {
            isLast = --entry.RefCount == 0;
            if (isLast)
                entry.RefCount = -1; // помечаем «умирает» — register начнёт заново
        }

        if (!isLast)
            return;

        // На 1 → 0: снимаем запись и детерминированно гасим сервер ВНЕ Lock.
        _entries.TryRemove(key, out _);
        await entry.Server.DisposeAsync().ConfigureAwait(false); // освобождает сокет
        Log.ServerStopped(logger, key.Host, key.Port);
    }

    private sealed class RouteLease(
        DynamicRouteManager owner, ServerKey key, Entry entry, CompiledRoute route)
        : IRouteLease
    {
        private int _disposed;

        public async ValueTask DisposeAsync()
        {
            // Идемпотентность: повторный DisposeAsync не декрементит дважды.
            if (Interlocked.Exchange(ref _disposed, 1) != 0)
                return;

            entry.Server.RemoveRoute(route);
            await owner.ReleaseAsync(key, entry).ConfigureAwait(false);
        }
    }

    public async ValueTask DisposeAsync()
    {
        foreach (var entry in _entries.Values)
            await entry.Server.DisposeAsync().ConfigureAwait(false);
        _entries.Clear();
    }
}
```

```csharp
internal static partial class Log
{
    [LoggerMessage(Level = LogLevel.Debug, Message = "Route registered {Method} on {Host}:{Port}")]
    public static partial void RouteRegistered(ILogger logger, string method, string host, int port);

    [LoggerMessage(Level = LogLevel.Information, Message = "Server stopped (last route removed) {Host}:{Port}")]
    public static partial void ServerStopped(ILogger logger, string host, int port);
}
```

> [!warning]
> Декремент и «последний ли это» — **одна** атомарная критическая секция под `Lock`. Если читать `RefCount`, потом отдельно декрементить, между ними успеет вклиниться `register` — и ты либо погасишь живой сервер, либо оставишь висеть мёртвый. `--entry.RefCount == 0` под замком + пометка `-1` («умирает») закрывают окно гонки register/teardown: опоздавший `register` видит `< 0` и берёт свежую запись.

Почему сам `DisposeAsync` сервера — вне `Lock`: остановка Kestrel асинхронна (дренаж соединений, освобождение сокета), а `await` под `Lock` запрещён по той же причине, что в разделе 3. Под замком — только переход счётчика и пометка к сносу; тяжёлый teardown — снаружи.

---

## 6. Hop-by-hop фильтрация: не отражай чужие заголовки

### Почему это проблема именно у raw-транспорта

Когда HTTP для тебя — **прозрачный транспорт** (proxy, gateway, туннель, RPC-over-HTTP), велик соблазн пробросить заголовки «как есть»: вход → апстрим, ответ апстрима → вниз. Это две разные ошибки:

1. **Hop-by-hop заголовки нельзя форвардить.** По RFC 9110 `Connection`, `Keep-Alive`, `Transfer-Encoding`, `TE`, `Trailer`, `Upgrade`, `Proxy-Authorization`, `Proxy-Authenticate` относятся к **одному hop'у** (соединению), а не к end-to-end сообщению. Пробросив `Transfer-Encoding: chunked` или `Connection: keep-alive` дальше, ты ломаешь фрейминг следующего соединения (двойной chunked, повисшие keep-alive). Плюс сам `Connection` перечисляет имена, которые тоже надо снять.
2. **Reflection/leak входных заголовков в ответ.** Если ты по неосторожности копируешь имена входных заголовков (`Host`, `User-Agent`, кастомные `X-Internal-*`) в ответ — раскрываешь внутреннюю топологию и даёшь reflection-вектор (атакующий шлёт заголовок, видит его в ответе).

### `SearchValues` + `FrozenSet` для hop-by-hop

Набор hop-by-hop имён фиксирован и мал — идеальный кейс для `FrozenSet<string>` с case-insensitive компаратором (имена заголовков ASCII-case-insensitive).

```csharp
public static class HopByHop
{
    // RFC 9110 §7.6.1 — заголовки уровня соединения, end-to-end проброс запрещён.
    private static readonly FrozenSet<string> Names = new[]
    {
        "Connection", "Keep-Alive", "Proxy-Connection", "Proxy-Authenticate",
        "Proxy-Authorization", "TE", "Trailer", "Transfer-Encoding", "Upgrade",
    }.ToFrozenSet(StringComparer.OrdinalIgnoreCase);

    public static bool IsHopByHop(string name) => Names.Contains(name);
}
```

### Динамический список: `Connection` диктует, что ещё снять

`Connection: X-Foo, X-Bar` означает «X-Foo и X-Bar — тоже hop-by-hop для этого соединения». Их надо вычислить из самого запроса и добавить к статическому набору.

```csharp
public static class HeaderForwarding
{
    // Возвращает множество имён, которые НЕЛЬЗЯ форвардить дальше:
    // статические hop-by-hop + перечисленные в Connection.
    public static FrozenSet<string> BuildExcluded(IHeaderDictionary inbound)
    {
        var excluded = new HashSet<string>(StringComparer.OrdinalIgnoreCase);

        // Статические hop-by-hop.
        foreach (var name in HopByHopStatic)
            excluded.Add(name);

        // Динамические — то, что перечислено в Connection.
        if (inbound.TryGetValue("Connection", out var connection))
        {
            foreach (var token in connection)
            {
                if (token is null)
                    continue;

                var span = token.AsSpan();
                foreach (var range in span.Split(','))
                {
                    var trimmed = span[range].Trim();
                    if (!trimmed.IsEmpty)
                        excluded.Add(trimmed.ToString());
                }
            }
        }

        return excluded.ToFrozenSet(StringComparer.OrdinalIgnoreCase);
    }

    private static readonly string[] HopByHopStatic =
    [
        "Connection", "Keep-Alive", "Proxy-Connection", "Proxy-Authenticate",
        "Proxy-Authorization", "TE", "Trailer", "Transfer-Encoding", "Upgrade",
    ];
}
```

### Трекинг входных имён, чтобы не отразить их в ответе

Чтобы гарантированно не «протечь» входными заголовками в ответ, снимай слепок имён входящего запроса и при формировании ответа исключай их (плюс hop-by-hop, плюс `Host`/`User-Agent`, которые в ответе бессмысленны).

```csharp
public static void CopyResponseHeaders(
    IHeaderDictionary upstream,
    IHeaderDictionary downstream,
    FrozenSet<string> inboundNames,        // имена входного запроса — не отражаем
    FrozenSet<string> hopByHop)            // hop-by-hop из BuildExcluded
{
    foreach (var (name, value) in upstream)
    {
        if (hopByHop.Contains(name))
            continue;                       // уровень соединения — не сквозной

        if (inboundNames.Contains(name))
            continue;                       // anti-reflection: не возвращаем чужой вход

        downstream[name] = value;
    }
}
```

> [!warning]
> `Host` — особый случай. Это не hop-by-hop, но при проксировании ты обязан **переписать** его на хост апстрима (иначе апстрим увидит чужой `Host` и может неверно сматчить virtual host / vhost-routing). И никогда не копируй входящий `Host`/`User-Agent` в **ответ** — в ответе их быть не должно, их присутствие там — сигнал бездумного «отзеркаливания» всего словаря.

Почему `FrozenSet`, а не `HashSet`: набор строится один раз (статическая часть — на старте, динамическая — на запрос) и потом только читается в hot path копирования заголовков. `FrozenSet` оптимизирует чтение ценой более дорогого построения — ровно профиль «built once, read many».

---

## 7. Streaming `IAsyncEnumerable<string>`: фрейминг по Content-Type

### Один источник данных — два проводных формата

Часто бизнес-слой отдаёт поток строк (`IAsyncEnumerable<string>`) — токены LLM, строки лога, события прогресса. Как это уедет по проводу, решает транспортный слой по `Content-Type`:

- `text/event-stream` → **SSE**: каждый элемент — кадр `data: ...\n\n`, заголовки уходят **до** первого элемента (late headers), браузер читает через `EventSource`.
- иначе → **HTTP chunked transfer**: тело пишется кусками без заранее известной длины (`Transfer-Encoding: chunked` Kestrel проставит сам, как только ты пишешь без `Content-Length`).

В обоих случаях критично **отключить буферизацию ответа** — иначе Kestrel/промежуточный слой накопит данные и отдаст разом, убив весь смысл стриминга (первый токен LLM должен уйти сразу, а не через 30 секунд).

### Диспетчер фрейминга

```csharp
public static class StreamingResponse
{
    public static async Task WriteAsync(
        HttpContext context,
        IAsyncEnumerable<string> source,
        CancellationToken ct)
    {
        // Отключаем буферизацию ДО первой записи: данные текут немедленно.
        var bodyFeature = context.Features.Get<IHttpResponseBodyFeature>();
        bodyFeature?.DisableBuffering();

        var contentType = context.Request.Headers.Accept.ToString();
        var wantsSse = contentType.Contains(
            "text/event-stream", StringComparison.OrdinalIgnoreCase);

        if (wantsSse)
            await WriteSseAsync(context, source, ct).ConfigureAwait(false);
        else
            await WriteChunkedAsync(context, source, ct).ConfigureAwait(false);
    }
}
```

### SSE: late headers + кадры `data:`

```csharp
private static async Task WriteSseAsync(
    HttpContext context,
    IAsyncEnumerable<string> source,
    CancellationToken ct)
{
    // Late headers: проставляем ДО первого тела, после первого WriteAsync
    // заголовки уже отправлены и менять их поздно (ResponseStarted == true).
    context.Response.StatusCode = StatusCodes.Status200OK;
    context.Response.ContentType = "text/event-stream";
    context.Response.Headers.CacheControl = "no-cache";
    context.Response.Headers["X-Accel-Buffering"] = "no"; // отключить буфер nginx

    await foreach (var item in source.WithCancellation(ct).ConfigureAwait(false))
    {
        // SSE-кадр: каждая строка значения — отдельный "data:", терминатор — пустая строка.
        // Многострочное значение бьём, иначе перенос внутри item сломает кадр.
        foreach (var line in item.Split('\n'))
        {
            await context.Response.WriteAsync($"data: {line}\n", ct).ConfigureAwait(false);
        }

        await context.Response.WriteAsync("\n", ct).ConfigureAwait(false); // конец кадра
        await context.Response.Body.FlushAsync(ct).ConfigureAwait(false);  // вытолкнуть сейчас
    }
}
```

### Chunked: тело без `Content-Length`

```csharp
private static async Task WriteChunkedAsync(
    HttpContext context,
    IAsyncEnumerable<string> source,
    CancellationToken ct)
{
    context.Response.StatusCode = StatusCodes.Status200OK;
    context.Response.ContentType = "text/plain; charset=utf-8";
    // НЕ ставим Content-Length → Kestrel применяет Transfer-Encoding: chunked.

    await foreach (var item in source.WithCancellation(ct).ConfigureAwait(false))
    {
        await context.Response.WriteAsync(item, ct).ConfigureAwait(false);
        await context.Response.Body.FlushAsync(ct).ConfigureAwait(false); // кадр уходит сразу
    }
}
```

> [!warning]
> `DisableBuffering()` и явный `FlushAsync` после каждого элемента — не «на всякий случай», а суть стриминга. Без них ASP.NET Core/Kestrel вправе накапливать write'ы во внутренний буфер и отдать их пачкой при завершении. Для SSE/LLM это означает «ничего → всё разом» вместо инкрементальной выдачи. Также: после первого `WriteAsync` заголовки уже на проводе (`Response.HasStarted == true`) — выставляй `StatusCode`/`ContentType`/headers строго ДО первой записи тела.

> [!info]
> `[EnumeratorCancellation]` на параметре `CancellationToken` источника обязателен, если поток генерируется через `yield return` в самом методе — иначе токен из `WithCancellation` не дойдёт до итератора, и отмена клиента (клиент закрыл вкладку → `RequestAborted`) не остановит дорогую генерацию на сервере. Прокидывай `context.RequestAborted` в `ct`.

---

> [!tip]- **Addendum: HttpListener / HTTP.sys — ступень НИЖЕ Kestrel**
> Для ~10 тривиальных внутренних admin-роутов (health, dump конфигурации, ручной trigger джобы) даже raw-Kestrel может быть избыточен. `System.Net.HttpListener` (кроссплатформенный, поверх HTTP.sys на Windows) или `UseHttpSys()` — это ещё более низкая ступень: голый `HttpListenerContext` без DI-хоста.
>
> Когда оправдано: отдельный loopback-порт для админки, которую не должно быть видно в основном pipeline; нулевые зависимости; на Windows HTTP.sys даёт kernel-mode port sharing (несколько процессов на одном порту по URL-префиксу) и kernel-режим TLS/Windows-auth.
>
> Цена осознанная и высокая: ты теряешь **всё**, что даёт ASP.NET Core — auth/authorization middleware, OpenAPI/Swagger, `IHttpClientFactory`/resilience, model binding, DI-граф, structured logging pipeline, OpenTelemetry-инструментацию. Для 10 строк health-эндпоинта это приемлемо; для чего-то, что вырастет в API — нет (мигрировать назад на Kestrel дороже, чем сразу взять `MapGroup("/admin")`). HTTP.sys также **только Windows** — кроссплатформенный код так не напишешь.

---

## 8. Trade-offs: raw host vs YARP vs обычный endpoint routing

| Критерий | Raw Kestrel host | YARP | Endpoint routing / Minimal API |
|----------|------------------|------|-------------------------------|
| Кто владеет маршрутизацией | ты | конфиг/код YARP | ASP.NET Core |
| Transport coupling | разорвана | средняя | полная (by design) |
| Объём кода для старта | высокий | низкий | минимальный |
| Подходит для proxy/gateway | да, но пишешь сам | да, из коробки | нет |
| Свой wire-протокол | да | нет | нет |
| Несколько серверов на `(host,port)` в процессе | да (server manager) | частично | неудобно |
| Риск сделать хуже Microsoft | высокий | низкий | нулевой |
| Maintenance burden | на тебе | на YARP-команде | на ASP.NET-команде |

Как читать таблицу по механизму, а не по галочкам:

- **YARP** — это уже написанный reverse proxy на тех же примитивах (Kestrel + `HttpContext`), но с готовой моделью маршрутов, кластеров, health-check'ов, трансформаций и балансировки. Если задача — "проксировать/балансировать трафик по конфигу", писать это руками поверх raw host бессмысленно: ты переизобретёшь YARP. См. [[api-gateway]] (YARP vs Ocelot vs NGINX, бенчмарки).
- **Endpoint routing** — оптимальный matcher (префиксное дерево, precedence-скоринг, генерация ссылок), отлаженный годами. Твой `RouteTable` из раздела 4 — это его упрощённая копия. Спускаться к raw host, чтобы получить *худшую* версию того же самого — анти-паттерн.
- **Raw host** выигрывает ровно в одном измерении: **владение семантикой**. Когда твоя модель маршрутизации/протокола фундаментально не совпадает с URL-паттернами ASP.NET, либо когда ты строишь инфраструктуру, которая принципиально не должна зависеть от endpoint routing (embeddable движок, multi-server менеджер). Во всех остальных случаях цена (свой matcher, свой dispatch, свой maintenance) не окупается.

Практическое правило принятия решения:

```
Нужно проксировать/балансировать по конфигу?      → YARP
Обычный API/сервис с бизнес-логикой?               → Minimal API
Своя модель маршрутов / свой протокол /
несколько серверов в процессе / embeddable движок? → raw Kestrel host
Иначе                                              → ты не там копаешь, вернись к Minimal API
```

---

## 9. Common pitfalls

### 1. Программный Listen* молча игнорирует `--urls`/`ASPNETCORE_URLS`

Если вызвал `options.Listen*`, переменные окружения на binding больше не влияют. Пользователь меняет `ASPNETCORE_URLS` — ничего не происходит. Документируй это явно или дай свой конфиг-источник.

### 2. HTTP/3 не поднимается без TLS и без `Alt-Svc`

HTTP/3 — только over QUIC+TLS 1.3, по UDP. Забыл `UseHttps()` — h3 не стартует. Забыл открыть UDP-порт или не отдал `Alt-Svc` — клиент тихо остаётся на HTTP/2 (выглядит как "h3 не работает", хотя сервер слушает).

### 3. `await` под `Lock`/`Monitor` в server manager

См. раздел 3. `await` внутри `lock` — компиляционная/логическая ошибка для `System.Threading.Lock` и опасный паттерн для `Monitor`. Держи под замком только синхронную публикацию, асинхронный старт — снаружи, с откатом при сбое.

### 4. `ConcurrentDictionary.GetOrAdd` с побочным эффектом в фабрике

Фабрика `GetOrAdd` может выполниться несколько раз при гонке. Если внутри неё bind на порт / старт сервера — получишь двойной bind. Для эффектов с дефицитным ресурсом нужен явный per-key `Lock` (double-checked), а не `GetOrAdd`.

### 5. Catch-all без явного 404

`app.Map("/{**path}", ...)` ловит **всё**. Если твой dispatcher не нашёл маршрут и забыл выставить `StatusCode`, клиент получит `200 OK` с пустым телом вместо `404`. Раз ты владеешь маршрутизацией — ты владеешь и фолбэком.

### 6. Сортировка маршрутов по "числу литералов" вместо посегментного сравнения

Суммарный счётчик литералов даёт неинтуитивные матчи (см. [!info] в разделе 4). Сравнивай специфичность покомпонентно, слева направо.

### 7. CreateSlimBuilder и потерянные дефолты

Slim не регистрирует часть того, что есть в `CreateBuilder` (полная конфигурация, JSON-reflection-дефолты, IIS integration). Если что-то "вдруг не работает" после перехода на Slim — это не баг, а отсутствующая регистрация. Добавляй нужное явно. См. [[native-aot]].

---

## 10. См. также

- [[native-aot]] — `CreateSlimBuilder`, AOT-готовность raw host, source-generated JSON
- [[api-gateway]] — YARP/Ocelot/NGINX: когда взять готовый proxy вместо raw host
- [[pipeline-middleware]] — что именно ты заменяешь, обходя routing/MVC
- [[di-configuration]] — регистрация singleton-менеджера, keyed services
- [[hosting-background]] — `BackgroundService`/`IHostedService` для жизненного цикла серверов

## 11. Reading list

- **Kestrel configuration** — learn.microsoft.com/aspnet/core/fundamentals/servers/kestrel/options
- **Endpoint Routing** — learn.microsoft.com/aspnet/core/fundamentals/routing (precedence-скоринг)
- **HTTP/3 in ASP.NET Core** — learn.microsoft.com/aspnet/core/fundamentals/servers/kestrel/http3
- **WebApplication.CreateSlimBuilder** — learn.microsoft.com/dotnet/api/microsoft.aspnetcore.builder.webapplication.createslimbuilder
- **YARP documentation** — microsoft.github.io/reverse-proxy
