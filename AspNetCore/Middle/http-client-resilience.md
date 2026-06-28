---
tags: [httpclient, ihttpclientfactory, resilience, networking, middle]
level: Middle
date: 2026-06-12
---

# HttpClient и IHttpClientFactory — pooling, DNS, resilience

> Canonical-файл по HTTP-клиенту в .NET: почему `new HttpClient()` на каждый запрос кладёт прод, почему singleton ловит stale DNS, что на самом деле решает `IHttpClientFactory` и как строить retry / circuit breaker на `Microsoft.Extensions.Http.Resilience`. Топ-5 вопрос на Middle/Senior собеседовании.

---

## 1. Что это, зачем и когда

### 1.1. Почему вокруг HttpClient столько боли

`HttpClient` выглядит как обычный сервис: создал, вызвал, задиспозил. Но это фасад над **connection pool'ом**, и оба «очевидных» способа им пользоваться ломаются под нагрузкой:

| Подход | Что ломается | Симптом в проде |
|---|---|---|
| `using var client = new HttpClient()` на каждый запрос | Socket exhaustion | `SocketException`, тысячи соединений в `TIME_WAIT` |
| Один singleton `HttpClient` на всё приложение | Stale DNS | После failover/blue-green трафик идёт на мёртвый IP |

Весь остальной файл — про механизм этих двух отказов и три рабочих решения.

### 1.2. Анатомия: клиент → handler → pool

`HttpClient` — тонкая обёртка (заголовки по умолчанию, `BaseAddress`, таймаут). Реальная работа — в цепочке `HttpMessageHandler`, на дне которой `SocketsHttpHandler` (с .NET Core 2.1 — единая кроссплатформенная реализация). **Connection pool живёт в handler'е**, не в клиенте.

```text
HttpClient
  └── DelegatingHandler (auth, logging, resilience...)   ← middleware-цепочка
        └── SocketsHttpHandler                            ← владелец connection pool
              └── TCP/TLS соединения (keep-alive)
```

Следствия:

- Dispose клиента → dispose handler'а → закрытие всех соединений pool'а.
- Два клиента поверх **одного** handler'а делят pool. Два handler'а — два независимых pool'а.
- Сам объект `HttpClient` дёшев. Дорог handler.

### 1.3. Когда что использовать (короткий ответ)

| Сценарий | Решение |
|---|---|
| ASP.NET Core / любое приложение с DI | `IHttpClientFactory` — named или typed clients |
| Console / worker без DI, простой случай | Один `static HttpClient` поверх `SocketsHttpHandler` с `PooledConnectionLifetime` |
| Запрос «раз в час» в скрипте | Что угодно — нагрузки нет, проблем нет |
| Никогда | `new HttpClient()` внутри метода на каждый вызов |

---

## 2. Failure mode #1 — socket exhaustion

### 2.1. Механизм

```csharp
// ❌ Классическая бомба замедленного действия
public async Task<string> GetOrderAsync(int id)
{
    using var client = new HttpClient(); // новый handler, новый pool
    return await client.GetStringAsync($"https://api.example.com/orders/{id}");
}
```

Что происходит по шагам:

1. Каждый вызов создаёт новый `SocketsHttpHandler` → новое TCP (+TLS) соединение. Handshake — лишние миллисекунды на каждый запрос, но это не главное.
2. `using` диспозит клиент → соединение **закрывается активной стороной** (нашей).
3. TCP-протокол требует, чтобы инициатор закрытия подержал сокет в состоянии `TIME_WAIT` (обычно 30–120 секунд) — защита от блуждающих пакетов старого соединения.
4. Под нагрузкой 100 RPS × 60 секунд = 6000 сокетов в `TIME_WAIT` одновременно. Эфемерные порты ОС заканчиваются.
5. Итог: `SocketException: Address already in use`, и падает не только этот код — **весь исходящий трафик процесса**.

Проверка на машине:

```bash
# Linux: посчитать сокеты в TIME_WAIT
ss -tan state time-wait | wc -l
```

```powershell
# Windows
(netstat -ano | Select-String "TIME_WAIT").Count
```

> [!question]- Интервью: почему `using var client = new HttpClient()` — баг, если HttpClient реализует IDisposable?
> Потому что Dispose здесь закрывает не «ресурс запроса», а **connection pool**. `IDisposable` у HttpClient — следствие владения handler'ом, а не приглашение диспозить на каждый вызов. Закрытие соединения нашей стороной оставляет сокет в `TIME_WAIT` на десятки секунд; при высоком RPS исчерпываются эфемерные порты. Правильно: переиспользовать handler (фабрика или static client), а время жизни соединений ограничивать `PooledConnectionLifetime`, а не Dispose'ом.

### 2.2. Почему «просто singleton» — тоже не ответ

Singleton решает exhaustion, но открывает второй failure mode — stale DNS. Поэтому ответ «сделай static и забудь» на собеседовании — это Middle-минус. Senior-ответ включает оба механизма.

---

## 3. Failure mode #2 — stale DNS

### 3.1. Механизм

Keep-alive соединения живут, пока живут. DNS-резолв происходит **один раз — при установке** соединения. Дальше TCP работает по IP, и TTL DNS-записи никого не волнует:

1. Singleton client установил 20 соединений к `api.example.com` → `10.0.0.5`.
2. Происходит failover / blue-green deploy / миграция — DNS теперь указывает на `10.0.0.9`.
3. Наши 20 соединений по-прежнему смотрят в `10.0.0.5`. Если старый узел жив, но «не тот» — мы тихо ходим в старую версию. Если мёртв — получаем таймауты, пока соединения не порвутся сами.

### 3.2. Решение — PooledConnectionLifetime

`SocketsHttpHandler` умеет принудительно рециркулировать соединения: pool продолжает работать, но каждое соединение живёт не дольше заданного срока, после чего закрывается и пересоздаётся — с новым DNS-резолвом.

```csharp
// ✅ Паттерн для приложений без DI: один клиент на процесс
public static class Http
{
    public static readonly HttpClient Client = new(new SocketsHttpHandler
    {
        PooledConnectionLifetime = TimeSpan.FromMinutes(2),   // ротация соединений → свежий DNS
        PooledConnectionIdleTimeout = TimeSpan.FromMinutes(1) // простаивающие закрывать раньше
    })
    {
        Timeout = TimeSpan.FromSeconds(15)
    };
}
```

2–5 минут — рабочий диапазон: достаточно редко, чтобы не терять выгоду pooling'а, достаточно часто, чтобы пережить failover за разумное время.

> [!info]- Если ты пришёл из мира .NET Framework
> Там той же цели служил `ServicePointManager.ConnectionLeaseTimeout`. В .NET Core+ `ServicePointManager` на `SocketsHttpHandler` **не действует** — настройки только через сам handler.

---

## 4. IHttpClientFactory — что решает на самом деле

### 4.1. Под капотом

```csharp
builder.Services.AddHttpClient();
```

Фабрика не «кеширует HttpClient». Она:

- держит **pool из `HttpMessageHandler`'ов** и ротирует их (default `HandlerLifetime` = 2 минуты) — это решает и exhaustion (handler переиспользуется), и stale DNS (handler периодически меняется);
- отдаёт лёгкий `HttpClient` поверх живого handler'а на каждый `CreateClient()` — создавать клиент на каждый запрос **можно и нужно**, он одноразовый по дизайну;
- интегрирует конфигурацию в DI: named/typed клиенты, цепочки `DelegatingHandler`;
- даёт логирование из коробки: категории `System.Net.Http.HttpClient.{Name}.LogicalHandler` / `.ClientHandler` — старт, статус, длительность каждого запроса.

Старые handler'ы после ротации не убиваются мгновенно — они уходят в expiry-pool и диспозятся, когда все их клиенты отпущены. Утечки нет.

### 4.2. Named clients

```csharp
builder.Services.AddHttpClient("github", client =>
{
    client.BaseAddress = new Uri("https://api.github.com/");
    client.DefaultRequestHeaders.UserAgent.ParseAdd("MyApp/1.0");
});
```

```csharp
public sealed class ReleaseChecker(IHttpClientFactory factory)
{
    public async Task<string> GetLatestAsync(CancellationToken ct)
    {
        var client = factory.CreateClient("github"); // дёшево, на каждый вызов — норм
        return await client.GetStringAsync("repos/dotnet/runtime/releases/latest", ct);
    }
}
```

Минус — строковое имя и ручной `CreateClient`. Для бизнес-кода идиоматичнее typed.

### 4.3. Typed clients

```csharp
public sealed class GitHubClient(HttpClient http)
{
    public async Task<string> GetLatestReleaseAsync(CancellationToken ct) =>
        await http.GetStringAsync("repos/dotnet/runtime/releases/latest", ct);
}
```

```csharp
builder.Services.AddHttpClient<GitHubClient>(client =>
{
    client.BaseAddress = new Uri("https://api.github.com/");
    client.DefaultRequestHeaders.UserAgent.ParseAdd("MyApp/1.0");
});
```

Typed client регистрируется как **transient**, и внутрь него фабрика инжектит правильно сконфигурированный `HttpClient`.

> [!warning] Captive dependency: typed client в singleton
> Если заинжектить typed client (transient) в singleton-сервис, он проживёт всё время жизни приложения вместе со своим `HttpClient` — ротация handler'ов для него фактически выключается, привет stale DNS. В singleton'ах инжекти `IHttpClientFactory` и зови `CreateClient()` per-операцию, либо используй паттерн из раздела 3.2. Подробнее про захват зависимостей — [[aspnet-dependency-injection-deep|DI deep]].

### 4.4. Чего фабрика НЕ решает

- **Retry/circuit breaker** — сама по себе нет, нужен resilience handler (раздел 6).
- **Аутентификация/токены** — нужен свой `DelegatingHandler` (раздел 5).
- DNS вне фабрики — если где-то остался ручной `new HttpClient()`, фабрика его не спасёт.

---

## 5. DelegatingHandler — middleware для исходящих запросов

Та же идея, что middleware-pipeline в ASP.NET Core (см. [[pipeline-middleware|Pipeline и Middleware]]), только для **outgoing** HTTP:

```csharp
public sealed class ApiKeyHandler(IApiKeyProvider keys) : DelegatingHandler
{
    protected override async Task<HttpResponseMessage> SendAsync(
        HttpRequestMessage request, CancellationToken cancellationToken)
    {
        request.Headers.Add("X-Api-Key", await keys.GetAsync(cancellationToken));
        return await base.SendAsync(request, cancellationToken);
    }
}
```

```csharp
builder.Services.AddTransient<ApiKeyHandler>();

builder.Services.AddHttpClient<GitHubClient>()
    .AddHttpMessageHandler<ApiKeyHandler>();   // порядок добавления = порядок выполнения
```

Нюансы:

- Handler'ы регистрируются **transient**, но фабрика кеширует собранную цепочку на время `HandlerLifetime` — не клади в handler scoped-зависимости напрямую, бери их через `IServiceScopeFactory` или прокидывай данные через `HttpRequestMessage.Options`.
- Типовые применения: auth-токены с рефрешем, корреляционные заголовки, подпись запросов, единая обработка 401.

---

## 6. Resilience — Microsoft.Extensions.Http.Resilience (.NET 8+)

### 6.1. Standard pipeline одной строкой

```bash
dotnet add package Microsoft.Extensions.Http.Resilience
```

```csharp
builder.Services.AddHttpClient<GitHubClient>()
    .AddStandardResilienceHandler();
```

Это Polly v8 под капотом. Конвейер стратегий (снаружи внутрь):

| # | Стратегия | Default |
|---|---|---|
| 1 | Rate limiter (исходящий) | 1000 параллельных |
| 2 | **Total timeout** — на всю операцию со всеми ретраями | 30 c |
| 3 | Retry | 3 попытки, exponential backoff + jitter; реагирует на 5xx, 408, 429, сетевые ошибки; уважает `Retry-After` |
| 4 | Circuit breaker | открывается при ≥10% ошибок в окне 30 c (минимум 100 запросов), пауза 5 c |
| 5 | **Attempt timeout** — на одну попытку | 10 c |

### 6.2. Кастомизация

```csharp
builder.Services.AddHttpClient<GitHubClient>()
    .AddStandardResilienceHandler(options =>
    {
        options.Retry.MaxRetryAttempts = 5;
        options.Retry.Delay = TimeSpan.FromMilliseconds(200);
        options.AttemptTimeout.Timeout = TimeSpan.FromSeconds(3);
        options.TotalRequestTimeout.Timeout = TimeSpan.FromSeconds(20);
        options.CircuitBreaker.FailureRatio = 0.2;
    });
```

### 6.3. Свой конвейер, когда standard мало

```csharp
builder.Services.AddHttpClient<GitHubClient>()
    .AddResilienceHandler("github-pipeline", pipeline =>
    {
        pipeline.AddTimeout(TimeSpan.FromSeconds(20));
        pipeline.AddRetry(new HttpRetryStrategyOptions
        {
            MaxRetryAttempts = 4,
            BackoffType = DelayBackoffType.Exponential,
            UseJitter = true,
            ShouldHandle = args => ValueTask.FromResult(
                args.Outcome.Result?.StatusCode is HttpStatusCode.ServiceUnavailable
                    or HttpStatusCode.TooManyRequests)
        });
        pipeline.AddCircuitBreaker(new HttpCircuitBreakerStrategyOptions());
    });
```

Глубже про сами стратегии, их математику и Polly v8 в целом — [[resilience|Resilience и Polly]]. Здесь важна интеграционная точка: resilience живёт как `DelegatingHandler` в цепочке клиента.

### 6.4. Идемпотентность — ограничение, о котором забывают

Retry безопасен для GET/PUT/DELETE (идемпотентны по спецификации) и опасен для POST: повторённый «создать заказ» = два заказа. Варианты:

- не ретраить POST вовсе (фильтр в `ShouldHandle` по `request.Method`);
- идемпотентность на стороне API: заголовок `Idempotency-Key`, дедупликация на сервере — см. [[api-design|API design]];
- ретраить только ошибки до отправки тела (connect-level), не после 5xx.

> [!question]- Интервью: retry уже есть в standard handler — зачем circuit breaker?
> Retry лечит **редкие** сбои, но при лежащем downstream'е превращается в усилитель атаки: каждый клиентский запрос порождает N исходящих, забивает пул и добивает соседа (retry storm). Circuit breaker — предохранитель: после порога ошибок перестаём ходить вовсе и отдаём fail-fast, даём downstream'у время подняться. Они решают разные задачи и работают в паре: retry внутри окна нормальной работы, breaker — защита при деградации.

---

## 7. Таймауты — три слоя, которые путают

| Слой | Что ограничивает | Где задаётся |
|---|---|---|
| `HttpClient.Timeout` | Весь запрос целиком: соединение + заголовки + чтение тела. Default **100 секунд** | На клиенте; менять можно только до первого запроса |
| Attempt timeout | Одну попытку внутри retry-конвейера | Resilience pipeline |
| `CancellationToken` | Внешняя отмена (ушёл пользователь, выключается хост) | Параметр вызова |

Плюс отдельно: `SocketsHttpHandler.ConnectTimeout` — только фаза установки TCP/TLS (default — бесконечность, имеет смысл выставить 5–10 c).

```csharp
var handler = new SocketsHttpHandler
{
    ConnectTimeout = TimeSpan.FromSeconds(5),
    PooledConnectionLifetime = TimeSpan.FromMinutes(2)
};
```

Нюанс диагностики: до .NET 5 таймаут клиента летел как голый `TaskCanceledException` — неотличимо от отмены токеном. С .NET 5 у него внутри `TimeoutException`:

```csharp
try
{
    using var response = await Http.Client.GetAsync(url, ct);
}
catch (TaskCanceledException ex) when (ex.InnerException is TimeoutException)
{
    // именно таймаут клиента, не отмена вызывающим
}
```

---

## 8. Полезные ручки SocketsHttpHandler / HttpClient

```csharp
var tuned = new SocketsHttpHandler
{
    PooledConnectionLifetime = TimeSpan.FromMinutes(2),
    MaxConnectionsPerServer = 100,                       // default: int.MaxValue
    AutomaticDecompression = DecompressionMethods.All,   // gzip / brotli
    EnableMultipleHttp2Connections = true                // h2: ещё соединения при нехватке стримов
};

var client = new HttpClient(tuned)
{
    DefaultRequestVersion = HttpVersion.Version20,
    DefaultVersionPolicy = HttpVersionPolicy.RequestVersionOrLower
};
```

- HTTP/2 мультиплексирует запросы в одном соединении — для чатти-клиентов это меньше соединений и handshake'ов.
- HTTP/3 (QUIC) включается так же: `DefaultRequestVersion = HttpVersion.Version30` при поддержке ОС/сервера.
- Стриминг больших ответов: `HttpCompletionOption.ResponseHeadersRead` + чтение `Content.ReadAsStreamAsync()` — иначе всё тело буферизуется в память. Обязательно диспозить `HttpResponseMessage`, пока читаешь stream.

---

## 9. Тестирование кода с HttpClient

### 9.1. Unit: fake handler вместо мока клиента

`SendAsync` у handler'а — `protected`, обычные мок-библиотеки его не сетапят напрямую. Простейший рабочий приём — свой handler:

```csharp
public sealed class StubHandler(Func<HttpRequestMessage, HttpResponseMessage> respond)
    : HttpMessageHandler
{
    public List<HttpRequestMessage> Requests { get; } = [];

    protected override Task<HttpResponseMessage> SendAsync(
        HttpRequestMessage request, CancellationToken cancellationToken)
    {
        Requests.Add(request);
        return Task.FromResult(respond(request));
    }
}
```

```csharp
[Fact]
public async Task GetLatestRelease_ReturnsBody()
{
    var stub = new StubHandler(_ => new HttpResponseMessage(HttpStatusCode.OK)
    {
        Content = new StringContent("""{"tag_name":"v10.0.0"}""")
    });
    var http = new HttpClient(stub) { BaseAddress = new Uri("https://api.github.com/") };
    var sut = new GitHubClient(http);

    var result = await sut.GetLatestReleaseAsync(CancellationToken.None);

    result.Should().Contain("v10.0.0");
    stub.Requests.Should().ContainSingle()
        .Which.RequestUri!.AbsolutePath.Should().Be("/repos/dotnet/runtime/releases/latest");
}
```

### 9.2. Integration: WireMock.Net

Когда важно проверить ретраи, таймауты и сериализацию по-настоящему — поднимай локальный HTTP-стаб (`WireMock.Net`): он умеет сценарии «два раза 503, потом 200», задержки, fault injection. В связке с `WebApplicationFactory` — см. [[integration-testing|Integration Testing]].

---

## 10. Common Pitfalls — с механизмами

1. **`new HttpClient()` на запрос** → socket exhaustion. Механизм: Dispose закрывает соединение нашей стороной → `TIME_WAIT` 30–120 c → исчерпание эфемерных портов (раздел 2).
2. **Singleton без `PooledConnectionLifetime`** → stale DNS. Механизм: DNS резолвится только при установке соединения; keep-alive живёт мимо TTL (раздел 3).
3. **Typed client в singleton** → captive dependency, ротация handler'ов выключена (раздел 4.3).
4. **`BaseAddress` без trailing slash + относительный путь со слешем.** `new Uri(new Uri("https://host/api"), "/v1/x")` даст `https://host/v1/x` — сегмент `api` съеден. Правило: base заканчивается на `/`, relative **не** начинается с `/`.
5. **`.Result` / `.GetAwaiter().GetResult()` на запросах** → deadlock в окружениях с sync context и thread-pool starvation под нагрузкой. Async all the way — [[async-threading|Async и Threading]].
6. **Чтение огромного тела через `ReadAsStringAsync`** → LOH-аллокации и пики памяти. Стримить через `ResponseHeadersRead` (раздел 8).
7. **Менять `HttpClient.Timeout` после первого запроса** → `InvalidOperationException`. Клиент «замораживается» после первого использования.
8. **Игнорировать `Retry-After` на 429** → бан от внешнего API. Standard resilience handler уважает его из коробки; в самописных ретраях — читать заголовок руками.
9. **`EnsureSuccessStatusCode()` как flow control.** Бросает голый `HttpRequestException` без тела ответа — теряешь детали ошибки API. Для ожидаемых ошибок читай `IsSuccessStatusCode` + тело, и возвращай Result — [[error-handling|Error Handling]].
10. **Не диспозить `HttpResponseMessage` при стриминге** → соединение не возвращается в pool до GC, pool деградирует.

---

## 11. Decision tree

```text
Нужен исходящий HTTP?
├─ Приложение с DI (ASP.NET Core, Worker)?
│   ├─ Да → IHttpClientFactory
│   │   ├─ Один внешний API, инжектится в сервисы → typed client
│   │   ├─ Несколько эндпоинтов с разной конфигурацией → named clients
│   │   ├─ Потребитель — singleton → инжекти IHttpClientFactory, не typed client
│   │   └─ Внешний API ненадёжен / по сети → + AddStandardResilienceHandler
│   └─ Нет (CLI, скрипт, простой worker)
│       → static HttpClient + SocketsHttpHandler { PooledConnectionLifetime = 2–5 мин }
└─ Запросов меньше одного в минуту → любой вариант выше, но не new-на-запрос по привычке
```

---

## 12. Cheat sheet

```csharp
// DI: typed client + resilience — дефолт для 95% сервисов
builder.Services.AddHttpClient<GitHubClient>(c =>
        c.BaseAddress = new Uri("https://api.github.com/"))
    .AddHttpMessageHandler<ApiKeyHandler>()
    .AddStandardResilienceHandler();
```

```csharp
// Без DI: один клиент на процесс
public static class Http
{
    public static readonly HttpClient Client = new(new SocketsHttpHandler
    {
        PooledConnectionLifetime = TimeSpan.FromMinutes(2),
        ConnectTimeout = TimeSpan.FromSeconds(5)
    });
}
```

| Вопрос | Ответ |
|---|---|
| Клиент на запрос? | Из фабрики — да; `new` — никогда |
| Default `HandlerLifetime` фабрики | 2 минуты |
| Default `HttpClient.Timeout` | 100 секунд |
| Stale DNS без фабрики | `PooledConnectionLifetime` |
| Retry для POST | Только с idempotency key |
| Большое тело ответа | `ResponseHeadersRead` + stream |

---

## 13. См. также

- [[resilience|Resilience и Polly]] — стратегии retry/breaker/hedging глубоко
- [[aspnet-dependency-injection-deep|DI deep]] — lifetimes и captive dependencies
- [[pipeline-middleware|Pipeline и Middleware]] — та же модель для входящих запросов
- [[api-design|API design]] — идемпотентность, версионирование, контракты
- [[async-threading|Async и Threading]] — почему sync-over-async убивает пул
- [[integration-testing|Integration Testing]] — WebApplicationFactory и стабы
- [[observability|Observability]] — трассировка исходящих вызовов

## 14. Reading list

- Microsoft Learn: «Use IHttpClientFactory to implement resilient HTTP requests»
- Microsoft Learn: «Guidelines for using HttpClient» — pooled connections и DNS
- .NET Blog: «Building resilient cloud services with .NET 8» — Microsoft.Extensions.Http.Resilience
- Andrew Lock: серия про IHttpClientFactory под капотом
- Steve Gordon: «HttpClient internals» — SocketsHttpHandler и pool
