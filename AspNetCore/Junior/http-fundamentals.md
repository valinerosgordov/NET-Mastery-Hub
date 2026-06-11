---
tags: [aspnetcore, http, https, cors, headers, status-codes, web-fundamentals, junior]
level: Junior
date: 2026-05-01
---

# HTTP / HTTPS Fundamentals — основа web разработки

> **Без понимания HTTP — невозможно делать качественный backend.** Status codes, methods, headers, HTTPS, CORS — то что Junior должен знать ДО изучения ASP.NET Core. Closes пробел "пишу API, но не знаю когда возвращать 422 vs 400".

---

## Что это, зачем и когда

### Зачем Junior знать HTTP детально

Большинство багов в API — от незнания HTTP:
- **400 vs 422 vs 409** — путаница в status codes
- **CORS** не настроен → frontend "Access-Control-Allow-Origin" errors
- **PUT vs PATCH** — реализуют одинаково (вместо разных семантик)
- **Cache headers** не используются → лишний трафик
- **Cookie SameSite** не настроен → CSRF атаки

### Что покроем

1. HTTP методы и их семантика (idempotency, safety)
2. Status codes детально (1xx-5xx)
3. Headers (request + response)
4. HTTPS, TLS handshake, certificates
5. CORS — как настроить в ASP.NET Core
6. Cookies vs Sessions vs JWT
7. HTTP/2 и HTTP/3 (QUIC) — что меняется
8. Caching через HTTP (Cache-Control / ETag)

---

## 1. HTTP методы

### Семантика каждого

| Method | Idempotent | Safe | Body | Назначение |
|--------|------------|------|------|------------|
| **GET** | ✅ | ✅ | ❌ | Read resource |
| **HEAD** | ✅ | ✅ | ❌ | Headers only (no body) |
| **OPTIONS** | ✅ | ✅ | ❌ | CORS preflight, allowed methods |
| **POST** | ❌ | ❌ | ✅ | Create / non-idempotent action |
| **PUT** | ✅ | ❌ | ✅ | Replace resource entirely |
| **PATCH** | ❌* | ❌ | ✅ | Partial update |
| **DELETE** | ✅ | ❌ | optional | Remove resource |

**Idempotent** = N вызовов = 1 вызов (по эффекту).  
**Safe** = не меняет state на сервере.

*PATCH может быть idempotent если operation такая (replace vs increment).

### Когда POST vs PUT vs PATCH

```
POST /users           → create new user (server picks ID)
POST /orders/123/pay  → action (charge payment)

PUT /users/42         → replace ENTIRE user object
PATCH /users/42       → partial update (только email field)

DELETE /users/42      → remove user
```

### ASP.NET Core примеры

```csharp
[ApiController]
[Route("api/users")]
public class UsersController : ControllerBase
{
    [HttpGet]                          // GET /api/users
    public IActionResult List() { }

    [HttpGet("{id}")]                  // GET /api/users/42
    public IActionResult Get(int id) { }

    [HttpPost]                          // POST /api/users
    public IActionResult Create(CreateUserDto dto) { }

    [HttpPut("{id}")]                  // PUT /api/users/42
    public IActionResult Replace(int id, UserDto dto) { }

    [HttpPatch("{id}")]                // PATCH /api/users/42
    public IActionResult Update(int id, JsonPatchDocument<User> patch) { }

    [HttpDelete("{id}")]               // DELETE /api/users/42
    public IActionResult Remove(int id) { }
}
```

См. [[api-design|API Design]].

---

## 2. Status codes

### Категории

```
1xx — Informational    (редко используется в API)
2xx — Success
3xx — Redirection
4xx — Client error
5xx — Server error
```

### Самые важные для API

| Code | Имя | Когда |
|------|-----|-------|
| **200** | OK | Успешный GET / PUT / PATCH с возвратом тела |
| **201** | Created | POST — ресурс создан, в Location header — URL |
| **202** | Accepted | Async operation начата (не завершена) |
| **204** | No Content | Успех без тела (DELETE, PUT без return) |
| **301** | Moved Permanently | URL изменился навсегда (SEO) |
| **302** | Found | Temporary redirect |
| **304** | Not Modified | Conditional GET, ресурс не изменился (ETag) |
| **400** | Bad Request | Невалидный JSON, missing required field |
| **401** | Unauthorized | Нужна авторизация (нет/невалидный token) |
| **403** | Forbidden | Авторизован, но не имеет права |
| **404** | Not Found | Ресурс не существует |
| **405** | Method Not Allowed | Endpoint существует, но method нет (GET вместо POST) |
| **409** | Conflict | Конфликт состояния (уже существует, optimistic locking) |
| **410** | Gone | Ресурс был, но удалён навсегда |
| **422** | Unprocessable Entity | Синтаксис OK, но семантика невалидна (business rules) |
| **429** | Too Many Requests | Rate limit |
| **500** | Internal Server Error | Necaught exception |
| **502** | Bad Gateway | Upstream service вернул error |
| **503** | Service Unavailable | Перегрузка / maintenance |
| **504** | Gateway Timeout | Upstream timeout |

### 400 vs 422 — самая частая путаница

```csharp
// 400 — синтаксис JSON невалиден
POST /users
{ "name": "John" "email": "..." }   // missing comma — invalid JSON
→ 400 Bad Request

// 422 — JSON валиден, но business rules нарушены
POST /users
{ "name": "", "age": -5 }
→ 422 Unprocessable Entity  // FluentValidation failed
```

> [!info] Convention
> Многие API используют **400** для всего invalid input. Это OK. Но **422** — более семантично для validation errors.

### ASP.NET Core helpers

```csharp
[HttpGet("{id}")]
public IActionResult Get(int id)
{
    var user = _service.Find(id);
    return user is null
        ? NotFound()                          // 404
        : Ok(user);                            // 200
}

[HttpPost]
public IActionResult Create(CreateUserDto dto)
{
    var validation = _validator.Validate(dto);
    if (!validation.IsValid)
        return UnprocessableEntity(validation.Errors);  // 422

    if (_service.EmailExists(dto.Email))
        return Conflict(new { error = "Email already registered" });  // 409

    var user = _service.Create(dto);
    return CreatedAtAction(nameof(Get), new { id = user.Id }, user);  // 201
}

[HttpDelete("{id}")]
public IActionResult Delete(int id)
{
    _service.Delete(id);
    return NoContent();  // 204
}
```

---

## 3. Headers

### Request headers (что клиент отправляет)

| Header | Зачем |
|--------|-------|
| `Authorization: Bearer eyJ...` | JWT / OAuth token |
| `Content-Type: application/json` | Тип отправляемого body |
| `Accept: application/json` | Какой формат хочу получить |
| `Accept-Language: ru-RU, en;q=0.9` | Предпочтения языка |
| `User-Agent: Mozilla/5.0...` | Идентификация клиента |
| `Cookie: session=abc123` | Сессия |
| `X-Forwarded-For: 1.2.3.4` | Real IP за proxy |
| `If-None-Match: "abc123"` | Conditional GET (ETag) |
| `If-Modified-Since: Thu, 01 May 2026 12:00:00 GMT` | Conditional GET (date) |

### Response headers (что сервер возвращает)

| Header | Зачем |
|--------|-------|
| `Content-Type: application/json; charset=utf-8` | Тип body |
| `Content-Length: 1024` | Размер body в байтах |
| `Cache-Control: max-age=3600` | Caching directive |
| `ETag: "abc123"` | Версия ресурса |
| `Last-Modified: Thu, 01 May 2026 12:00:00 GMT` | Когда последний раз менялся |
| `Location: /users/42` | URL созданного/перемещённого ресурса |
| `Set-Cookie: session=...; HttpOnly; Secure` | Установить cookie |
| `Access-Control-Allow-Origin: *` | CORS |
| `X-RateLimit-Remaining: 99` | Сколько requests осталось |
| `WWW-Authenticate: Bearer` | Какой auth scheme |

### Custom headers (`X-*` prefix)

```csharp
// Добавить custom response header
[HttpGet]
public IActionResult Get()
{
    Response.Headers["X-Total-Count"] = "1234";
    Response.Headers["X-API-Version"] = "v2";
    return Ok(data);
}

// Прочитать request header
[HttpGet]
public IActionResult Get()
{
    var apiKey = Request.Headers["X-API-Key"].FirstOrDefault();
    var correlationId = Request.Headers["X-Correlation-ID"].FirstOrDefault() 
        ?? Guid.NewGuid().ToString();
    // ...
}
```

> [!info] X-prefix — устарел
> RFC 6648 (2012) рекомендует **не использовать `X-`** для новых headers. Просто давай descriptive имя: `Correlation-ID` вместо `X-Correlation-ID`.

---

## 4. HTTPS — что внутри

### Зачем HTTPS

- **Encryption** — никто между client и server не видит трафик
- **Authentication** — клиент уверен, что общается с правильным сервером (через certificate)
- **Integrity** — данные не изменены в пути

### TLS handshake (упрощённо)

```
Client                                   Server
  │                                        │
  │── ClientHello (supported ciphers) ────→│
  │                                        │
  │←──── ServerHello + Certificate ────────│
  │                                        │
  ├── Verify cert через trusted CA ────────│
  │                                        │
  │── Premaster secret (encrypted) ───────→│
  │                                        │
  │← Both derive session keys ────────────→│
  │                                        │
  │←══ Encrypted communication ═══════════→│
```

### Setup в ASP.NET Core

```csharp
// Program.cs
var builder = WebApplication.CreateBuilder(args);

var app = builder.Build();

// Forced HTTPS redirect
app.UseHttpsRedirection();

// HSTS — браузер запомнит "только HTTPS на 1 год"
app.UseHsts();

app.MapControllers();
app.Run();
```

### Development certificates

```bash
# .NET создаёт self-signed cert для dev
dotnet dev-certs https --trust

# Production — Let's Encrypt (бесплатно)
# Или commercial CA (DigiCert, GlobalSign)
```

См. [[security-practices|Security Practices]].

---

## 5. CORS — самый частый source of frustration

### Что такое CORS

**Cross-Origin Resource Sharing** — браузер блокирует запросы к другому origin (домен/порт/протокол) если сервер не разрешил явно.

```
Origin = https://myapp.com (scheme + host + port)

Same origin:
  https://myapp.com/api/users    ✓
  https://myapp.com/api/orders   ✓

Different origin:
  https://api.myapp.com/users    ✗ (другой host)
  http://myapp.com/api/users     ✗ (другой scheme)
  https://myapp.com:8080/api     ✗ (другой port)
```

### Setup в ASP.NET Core

```csharp
var builder = WebApplication.CreateBuilder(args);

builder.Services.AddCors(options =>
{
    options.AddPolicy("AllowFrontend", policy =>
    {
        policy.WithOrigins("https://myapp.com", "http://localhost:3000")
            .AllowAnyMethod()
            .AllowAnyHeader()
            .AllowCredentials();  // для cookies
    });
});

var app = builder.Build();

app.UseCors("AllowFrontend");

app.MapControllers();
app.Run();
```

### Preflight request (OPTIONS)

Для **non-simple** requests браузер сначала отправляет `OPTIONS`:

```
OPTIONS /api/users HTTP/1.1
Origin: https://myapp.com
Access-Control-Request-Method: POST
Access-Control-Request-Headers: Authorization

← 204 No Content
Access-Control-Allow-Origin: https://myapp.com
Access-Control-Allow-Methods: POST, GET
Access-Control-Allow-Headers: Authorization
Access-Control-Max-Age: 86400
```

> [!warning] CORS — это **БРАУЗЕРНАЯ** защита
> CORS НЕ защищает от curl, Postman, server-to-server запросов. Это просто ограничение JavaScript в браузере. Для real security — auth + rate limiting.

---

## 6. Case Study #1 — Idempotent POST для платежей

**Сценарий:** Пользователь нажал "Оплатить" → плохая сеть → нажал снова. Без защиты — двойной charge.

### ❌ Naive POST

```csharp
[HttpPost("orders/{id}/pay")]
public IActionResult Pay(int id, PaymentDto payment)
{
    var result = _paymentService.Charge(id, payment);
    return Ok(result);
}

// User retry → второй charge!
```

### ✅ Idempotency-Key header

```csharp
[HttpPost("orders/{id}/pay")]
public async Task<IActionResult> Pay(int id, PaymentDto payment)
{
    var idempotencyKey = Request.Headers["Idempotency-Key"].FirstOrDefault();
    if (string.IsNullOrEmpty(idempotencyKey))
        return BadRequest(new { error = "Idempotency-Key header required" });

    // Проверяем кеш — уже обрабатывали этот key?
    var cached = await _cache.GetAsync<PaymentResult>($"payment:{idempotencyKey}");
    if (cached != null)
        return Ok(cached);  // вернуть тот же результат

    var result = await _paymentService.ChargeAsync(id, payment);

    // Сохранить на 24 часа
    await _cache.SetAsync($"payment:{idempotencyKey}", result, TimeSpan.FromHours(24));

    return Ok(result);
}
```

**Frontend генерирует UUID на каждую "Оплатить":**
```javascript
const idempotencyKey = crypto.randomUUID();

await fetch('/api/orders/42/pay', {
  method: 'POST',
  headers: { 'Idempotency-Key': idempotencyKey, 'Content-Type': 'application/json' },
  body: JSON.stringify(payment)
});
// Retry — same key → тот же результат
```

**Stripe / PayPal делают так же** — стандартный pattern.

См. [[api-design|API Design]].

---

## 7. Case Study #2 — ETag для conditional GET

**Сценарий:** Mobile app каждые 30 сек проверяет обновления newsfeed (1000+ постов). 99% времени — ничего не изменилось. Пустая трата трафика.

### ❌ Naive

```csharp
[HttpGet("newsfeed")]
public IActionResult Get()
{
    var posts = _service.GetPosts();
    return Ok(posts);  // всегда 200 + 500 KB body
}
```

### ✅ ETag pattern

```csharp
[HttpGet("newsfeed")]
public async Task<IActionResult> Get()
{
    var lastUpdated = await _service.GetLastUpdatedAsync();
    var etag = $"\"{lastUpdated.Ticks}\"";  // version из БД

    // Client прислал If-None-Match — проверяем
    var ifNoneMatch = Request.Headers["If-None-Match"].FirstOrDefault();
    if (ifNoneMatch == etag)
    {
        return StatusCode(304);  // Not Modified — пустой body!
    }

    var posts = await _service.GetPostsAsync();

    Response.Headers["ETag"] = etag;
    Response.Headers["Cache-Control"] = "private, max-age=30";

    return Ok(posts);
}
```

**Результат:**
- 99% requests → `304 Not Modified` (size <1 KB)
- Только 1% → реальный response (500 KB)
- **500x экономия трафика** для mobile users
- Battery savings (меньше radio activity)

См. [[caching|Caching]].

---

## 8. Case Study #3 — CORS misconfiguration в production

**Сценарий:** Frontend на `app.example.com`, API на `api.example.com`. В dev всё работает, в prod — ошибки CORS.

### ❌ Hardcoded localhost

```csharp
builder.Services.AddCors(options =>
{
    options.AddDefaultPolicy(policy =>
    {
        policy.WithOrigins("http://localhost:3000")  // ⚠️ только dev!
            .AllowAnyMethod()
            .AllowAnyHeader();
    });
});

// Prod: фронт на app.example.com получает CORS ошибку
```

### ✅ Из конфигурации

```json
// appsettings.json
{
  "AllowedOrigins": [ "http://localhost:3000" ]
}

// appsettings.Production.json
{
  "AllowedOrigins": [
    "https://app.example.com",
    "https://www.example.com"
  ]
}
```

```csharp
var allowedOrigins = builder.Configuration
    .GetSection("AllowedOrigins")
    .Get<string[]>() ?? Array.Empty<string>();

builder.Services.AddCors(options =>
{
    options.AddDefaultPolicy(policy =>
    {
        policy.WithOrigins(allowedOrigins)
            .AllowAnyMethod()
            .AllowAnyHeader()
            .AllowCredentials();
    });
});
```

> [!warning] НЕ используй `AllowAnyOrigin()` в production!
> Это разрешает **любому сайту** делать запросы к твоему API через браузер пользователя. Атаки CSRF становятся возможны.

---

## 9. Case Study #4 — Cookie security

**Сценарий:** Auth через cookie. Без правильных attributes — XSS / CSRF attacks.

### ❌ Без security attributes

```csharp
Response.Cookies.Append("session", token);  // ⚠️ не Secure, не HttpOnly!
```

Атаки:
- **XSS** — JavaScript на странице может прочитать cookie (если не HttpOnly)
- **MITM** — cookie передаётся по HTTP (если не Secure)
- **CSRF** — другой сайт может вызвать ваш API с cookie (если не SameSite)

### ✅ Все security attributes

```csharp
Response.Cookies.Append("session", token, new CookieOptions
{
    HttpOnly = true,                         // JS не может читать
    Secure = true,                            // только HTTPS
    SameSite = SameSiteMode.Strict,          // не отправляется на cross-site requests
    Expires = DateTimeOffset.UtcNow.AddDays(7),
    Path = "/api"
});
```

| Attribute | Защита от |
|-----------|-----------|
| `HttpOnly` | XSS (JS не читает) |
| `Secure` | MITM (только HTTPS) |
| `SameSite=Strict` | CSRF (cross-site requests) |
| `SameSite=Lax` | CSRF (но позволяет top-level navigation — практичнее) |

См. [[auth-security|Auth & Security]].

---

## 10. Case Study #5 — HTTP/2 multiplexing

**Сценарий:** API возвращает страницу с 30 ресурсами (images, JS, CSS). HTTP/1.1 — 6 параллельных connections, остальные ждут.

### HTTP/1.1 — head-of-line blocking

```
Connection 1: ──[req1]──[res1]──[req4]──[res4]──
Connection 2: ──[req2]──[res2]──[req5]──[res5]──
Connection 3: ──[req3]──[res3]──[req6]──[res6]──
... 6 connections max
```

Если res1 медленный — req4 ждёт.

### HTTP/2 — multiplexing

```
Connection 1: ──[req1][req2][req3][req4][req5]──
                ──[res2][res5][res1][res3][res4]──
                одновременно, parallel streams
```

### ASP.NET Core — HTTP/2 enabled by default

```csharp
builder.WebHost.ConfigureKestrel(options =>
{
    options.ListenAnyIP(5000, listenOptions =>
    {
        listenOptions.Protocols = HttpProtocols.Http1AndHttp2;
    });
});
```

**Когда нужен HTTP/2:**
- gRPC требует
- API с many small responses
- Single-page apps (SPA)

**HTTP/3 (QUIC):** UDP-based, ещё быстрее, но требует TLS 1.3 + cloud support.

---

## 11. Case Study #6 — Rate Limiting

**Сценарий:** API публичный, нужно защитить от abuse. .NET 7+ имеет встроенный rate limiter.

```csharp
using System.Threading.RateLimiting;

builder.Services.AddRateLimiter(options =>
{
    // Per-user rate limit
    options.AddPolicy("per-user", context =>
    {
        var userId = context.User.FindFirst("sub")?.Value ?? "anonymous";
        return RateLimitPartition.GetFixedWindowLimiter(userId, _ =>
            new FixedWindowRateLimiterOptions
            {
                PermitLimit = 100,
                Window = TimeSpan.FromMinutes(1)
            });
    });

    // Global rate limit
    options.AddFixedWindowLimiter("global", limiter =>
    {
        limiter.PermitLimit = 1000;
        limiter.Window = TimeSpan.FromMinutes(1);
    });

    // Возврат headers с remaining quota
    options.OnRejected = async (context, ct) =>
    {
        context.HttpContext.Response.StatusCode = 429;
        context.HttpContext.Response.Headers["Retry-After"] = "60";
        await context.HttpContext.Response.WriteAsync("Rate limit exceeded", ct);
    };
});

app.UseRateLimiter();

[HttpGet("/api/data")]
[EnableRateLimiting("per-user")]
public IActionResult Get() { /* ... */ }
```

**Response при превышении:**
```
HTTP/1.1 429 Too Many Requests
Retry-After: 60
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 0
```

См. [[resilience|Resilience]].

---

## 12. Common Pitfalls

### 1. Возвращать 200 на ошибку

```csharp
// ❌
return Ok(new { success = false, error = "User not found" });
// Status 200, но в body — error. Frontend не понимает.

// ✅
return NotFound(new { error = "User not found" });
// Status 404 + body
```

### 2. POST для всего

```csharp
// ❌ Junior — POST везде
POST /api/getUserById
POST /api/deleteUser
POST /api/updateProfile

// ✅
GET /api/users/42
DELETE /api/users/42
PATCH /api/users/42
```

### 3. PUT для partial update

```csharp
// ❌ PUT с одним полем
PUT /api/users/42
{ "email": "new@example.com" }
// PUT должен заменить ВЕСЬ объект — что с другими полями?

// ✅ PATCH
PATCH /api/users/42
{ "email": "new@example.com" }
```

### 4. Cookie без HttpOnly

XSS на странице → атакующий читает session cookie → impersonation.

### 5. CORS `AllowAnyOrigin` + `AllowCredentials`

```csharp
// ❌ Compile-time invalid combination
policy.AllowAnyOrigin().AllowCredentials();
// CORS spec не разрешает credentials с *
```

Если нужны credentials — указывай конкретные origins.

### 6. `Authorization` в URL query string

```csharp
// ❌
GET /api/users?token=eyJ...

// Token попадёт в:
// - Server logs
// - Browser history  
// - Referrer header
```

Используй `Authorization: Bearer ...` header.

### 7. Не использовать HTTPS redirect

```csharp
// ❌ В production
app.UseHttpsRedirection();  // забыли!
// HTTP запросы не редиректятся → MITM возможен
```

### 8. Headers case-sensitivity confusion

HTTP headers — **case-insensitive**.
- `Content-Type`, `content-type`, `CONTENT-TYPE` — одно и то же.
- В .NET — `IHeaderDictionary` уже case-insensitive.

### 9. `Content-Length` врёт

Если body — chunked transfer encoding, `Content-Length` отсутствует. Используй `Transfer-Encoding: chunked`.

### 10. Игнорировать `Accept` header

```csharp
// Клиент: Accept: application/xml
// ❌ Сервер всегда возвращает JSON

// ✅ ASP.NET Core с content negotiation
builder.Services.AddControllers()
    .AddXmlSerializerFormatters();  // теперь поддерживается XML
```

---

## 13. Best Practices

### API design

- **Семантичные status codes** — 200/201/204/400/404/409/422/500
- **Идемпотентность** для POST через `Idempotency-Key`
- **PATCH** для partial updates, не PUT
- **HEAD** для существования без download
- **OPTIONS** для CORS preflight
- **Versioning** через URL (`/api/v2/users`) или header

### Security

- **HTTPS everywhere** — `UseHttpsRedirection` + `UseHsts`
- **HSTS** — `max-age=31536000; includeSubDomains; preload`
- **Cookies** — `HttpOnly + Secure + SameSite`
- **CORS** — конкретные origins, не `*`
- **Rate limiting** — per-user + global
- **Headers** — `X-Content-Type-Options: nosniff`, `X-Frame-Options: DENY`

### Performance

- **Cache-Control** для статических ресурсов
- **ETag** для dynamic content
- **HTTP/2** для multiplexing
- **gzip / brotli compression** — `app.UseResponseCompression()`
- **Connection pooling** — `IHttpClientFactory` (не `new HttpClient()`)

### Logging / observability

- **Correlation-ID** для tracing
- **X-Request-ID** между сервисами
- **Logging request/response без sensitive data** (не token, password)

См. [[observability|Observability]].

---

## 14. Cheat sheet

| Сценарий | Решение |
|----------|---------|
| Read resource | `GET /resource/:id` → 200 / 404 |
| Create | `POST /resource` → 201 + Location |
| Replace fully | `PUT /resource/:id` → 200/204 |
| Partial update | `PATCH /resource/:id` → 200/204 |
| Delete | `DELETE /resource/:id` → 204 |
| Async operation | `POST` → 202 + Location to status |
| Validation error | 422 (или 400) + body с errors |
| Already exists | 409 Conflict |
| Rate limit | 429 + `Retry-After` |
| Не авторизован | 401 + `WWW-Authenticate` |
| Нет прав | 403 |
| HTTPS force | `app.UseHttpsRedirection()` + HSTS |
| CORS dev | конкретные origins из config |
| Cookie security | HttpOnly + Secure + SameSite=Strict |
| Idempotent POST | `Idempotency-Key` header + cache |
| Conditional GET | ETag + `If-None-Match` → 304 |
| Caching | `Cache-Control: max-age=...` |

---

## 15. Decision tree

```
Какой method использовать?
│
├── Read data, no side effects? → GET (or HEAD без body)
├── Create new, server picks ID? → POST
├── Replace entire object? → PUT
├── Update some fields? → PATCH
├── Remove? → DELETE
├── Action (charge, send, calc)? → POST /resource/:id/action
└── Auth check / CORS? → OPTIONS

Какой status code?
│
├── Operation succeed?
│   ├── Returns body → 200
│   ├── Created new (POST) → 201 + Location
│   ├── Async started → 202 + Location to status
│   └── No body → 204
│
├── Client error?
│   ├── Bad JSON / syntax → 400
│   ├── No / invalid auth → 401
│   ├── Not enough rights → 403
│   ├── Doesn't exist → 404
│   ├── Method not supported → 405
│   ├── Already exists / version mismatch → 409
│   ├── Validation failed (semantics) → 422
│   └── Rate limit → 429
│
└── Server error?
    ├── Uncaught exception → 500
    ├── Upstream service failed → 502
    ├── Maintenance / overload → 503
    └── Upstream timeout → 504
```

---

## 16. См. также

- [[api-design|API Design]] — REST principles, versioning, OpenAPI
- [[auth-security|Auth & Security]] — JWT, OAuth, Identity
- [[security-practices|Security Practices]] — OWASP, CSP, headers
- [[caching|Caching]] — IDistributedCache, OutputCaching
- [[pipeline-middleware|Pipeline & Middleware]] — где обрабатываются headers
- [[resilience|Resilience]] — retry, circuit breaker
- [[signalr|SignalR]] — real-time communication

## 17. Reading list

- **MDN — HTTP** — developer.mozilla.org/en-US/docs/Web/HTTP (best HTTP reference)
- **RFC 7231 — HTTP/1.1 Semantics** — tools.ietf.org/html/rfc7231
- **RFC 9110 — HTTP Semantics** (modern) — datatracker.ietf.org/doc/html/rfc9110
- **HTTP: The Definitive Guide** — David Gourley (классика)
- **High Performance Browser Networking** — Ilya Grigorik (free online: hpbn.co)
- **Microsoft Docs — HTTP/2 in ASP.NET Core** — learn.microsoft.com/aspnet/core/fundamentals/servers/kestrel/http2
- **Stripe API design** — stripe.com/docs/api (excellent example of REST design)
