---
tags: [aspnet, auth, jwt, oauth2, oidc, cors, security, identityserver, keycloak, mtls]
level: Senior
date: 2026-06-28
---

# Authentication, Authorization и безопасность

## Что это, зачем и когда

### Что такое Authentication и Authorization?
- **Authentication (аутентификация)** — **кто ты?** Проверка личности (логин/пароль, JWT-токен, сертификат).
- **Authorization (авторизация)** — **что тебе можно?** Проверка прав (роли, политики, claims).

**Аналогия — клуб:** Охранник проверяет паспорт (authentication) — «Да, ты Иван». Потом проверяет VIP-список (authorization) — «Иван, тебе можно в VIP-зону».

### Зачем?
- Без authentication — любой может представиться кем угодно
- Без authorization — любой авторизованный пользователь имеет доступ ко всему (в т.ч. к админке)

### Когда что?

| Механизм | Когда |
|----------|-------|
| **JWT (Bearer Token)** | REST API, SPA, мобильные приложения. Stateless |
| **Cookie** | MVC/Razor Pages/Blazor Server, SSR. Server-side state |
| **OAuth2 + OIDC** | Single Sign-On, federation, third-party identity (Google/GitHub) |
| **API Key** | M2M, internal services с простой моделью |
| **mTLS (Mutual TLS)** | M2M в zero-trust сетях, финансовые системы |
| **Role-based** | Простые случаи: Admin, Manager, User |
| **Policy-based** | Сложные правила: «старше 18 И из РФ И верифицировал email» |
| **Resource-based** | «Пользователь редактирует ТОЛЬКО СВОИ заказы» |

---

> [!question]- **Интервью: Authentication vs Authorization?**
> **Authentication** — кто ты? (identity, JWT, Cookie). **Authorization** — что тебе можно? (roles, policies, claims).
> Authentication выполняется первой — пока не знаем кто, не можем проверить права.

> [!question]- **Интервью: Cookie vs JWT — когда что?**
> **Cookie** — server-side state, CSRF-защита нужна, подходит для MVC/SSR/Blazor Server. Auto-attached браузером, легко revoke (удалили session).
> **JWT** — stateless, подходит для API/SPA/mobile, **храним в httpOnly cookie** (не localStorage — XSS!). Нельзя revoke до истечения exp без blacklist'а.
> **Refresh tokens** — для ротации короткоживущих access tokens.

> [!question]- **Интервью: CORS — зачем и как?**
> Cross-Origin Resource Sharing. Браузер блокирует AJAX-запросы к другому origin. Сервер должен ответить `Access-Control-Allow-Origin`. CORS middleware — **до Auth**, чтобы preflight (OPTIONS без Authorization header) работал.

---

## Pipeline setup

```csharp
builder.Services.AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
    .AddJwtBearer(opts =>
    {
        opts.TokenValidationParameters = new TokenValidationParameters
        {
            ValidateIssuer = true,
            ValidateAudience = true,
            ValidateLifetime = true,
            ValidateIssuerSigningKey = true,
            ValidIssuer = builder.Configuration["Jwt:Issuer"],
            ValidAudience = builder.Configuration["Jwt:Audience"],
            IssuerSigningKey = new SymmetricSecurityKey(
                Encoding.UTF8.GetBytes(builder.Configuration["Jwt:Key"]!)),
            ClockSkew = TimeSpan.Zero  // По умолчанию 5 минут — выкл!
        };
    });

builder.Services.AddAuthorization();

var app = builder.Build();

app.UseCors();           // Перед auth — для preflight
app.UseAuthentication(); // Заполняет HttpContext.User
app.UseAuthorization();  // Проверяет права доступа
```

**Порядок middleware критичен:** Routing → CORS → Authentication → Authorization → Endpoints.

### Применение атрибутов

```csharp
[Authorize]                              // Требует аутентификации
[Authorize(Roles = "Admin")]             // Требует роль Admin
[Authorize(Policy = "MinAge18")]         // Требует policy
[Authorize(AuthenticationSchemes = "Bearer,Cookie")]  // Любая из схем
[AllowAnonymous]                         // Открытый endpoint

// Глобальная авторизация — всё закрыто, открываем явно
builder.Services.AddAuthorization(opts =>
{
    opts.FallbackPolicy = new AuthorizationPolicyBuilder()
        .RequireAuthenticatedUser()
        .Build();
});
```

---

## JWT — структура и тонкости

```
Header:    { "alg": "HS256", "typ": "JWT", "kid": "key-2026" }
Payload:   { "sub": "user-123", "email": "...", "exp": 1700000000, "iat": ..., "jti": ... }
Signature: HMACSHA256(base64(header) + "." + base64(payload), secret)
```

### Алгоритмы подписи — что выбрать

| Алгоритм | Тип | Когда |
|----------|-----|-------|
| **HS256** | HMAC-SHA256 (symmetric) | Один сервис генерирует и валидирует. Простой setup |
| **RS256** | RSA-SHA256 (asymmetric) | Identity Provider подписывает private key, resource servers валидируют public key. SSO/OIDC |
| **ES256** | ECDSA-SHA256 (asymmetric) | Как RS256, но короче подпись (быстрее, экономит трафик) |
| **EdDSA** | Ed25519 | Новый стандарт, лучшая производительность чем ECDSA |

**Правило:** для multi-service architecture — RS256/ES256. Для single-service — HS256 (но с **сильным** secret, минимум 256 bit / 32 байта).

### Генерация JWT

```csharp
public class TokenService(IOptions<JwtOptions> options)
{
    private readonly JwtOptions _options = options.Value;

    public string GenerateAccessToken(User user)
    {
        var claims = new List<Claim>
        {
            new(JwtRegisteredClaimNames.Sub, user.Id.ToString()),
            new(JwtRegisteredClaimNames.Email, user.Email),
            new(JwtRegisteredClaimNames.Jti, Guid.NewGuid().ToString()),  // Unique token ID для revoke
            new("tenant_id", user.TenantId.ToString()),                    // Custom claim
        };

        foreach (var role in user.Roles)
            claims.Add(new Claim(ClaimTypes.Role, role));

        var key = new SymmetricSecurityKey(Encoding.UTF8.GetBytes(_options.Key));
        var creds = new SigningCredentials(key, SecurityAlgorithms.HmacSha256);

        var token = new JwtSecurityToken(
            issuer: _options.Issuer,
            audience: _options.Audience,
            claims: claims,
            notBefore: DateTime.UtcNow,
            expires: DateTime.UtcNow.AddMinutes(15),  // КОРОТКИЙ TTL
            signingCredentials: creds);

        return new JwtSecurityTokenHandler().WriteToken(token);
    }
}
```

### Тонкости JWT

| Pitfall | Решение |
|---------|---------|
| Long-lived JWT нельзя revoke | TTL 5-15 минут + Refresh Token rotation + jti blacklist в Redis |
| Чувствительные данные в payload | Payload **только Base64**, не шифрование. Никогда не клади пароли/PII |
| `ClockSkew` 5 минут default | Установи `TimeSpan.Zero` — иначе токен живёт 5 мин после exp |
| Алгоритм `none` | `ValidateIssuerSigningKey = true` обязателен — иначе атакующий подсунет `alg: none` |
| Размер токена с 50 ролями | Кладите только `sub`, queryите permissions из БД через cache |
| Symmetric key утёк | Rotation через `kid` (key id) header — несколько ключей одновременно работают |

> [!question]- **Интервью: что такое алгоритм `none` атака?**
> Старые JWT-библиотеки принимали токены с `"alg": "none"` (без подписи). Атакующий генерирует JWT с любым payload и `alg: none`, и сервер его принимает. Защита — явный whitelist разрешённых алгоритмов в TokenValidationParameters; современные SDK блокируют `none` по умолчанию.

> [!question]- **Интервью: как сделать JWT revocation?**
> JWT по дизайну non-revocable до exp. Способы:
> 1. **Короткий TTL (5-15 мин)** — atypically revocation не нужна
> 2. **Blacklist по `jti`** — Redis hash, проверка в middleware. TTL = exp - now
> 3. **Whitelist по `jti`** — все валидные jti в Redis, на logout удаляем. Дороже по памяти
> 4. **Token versioning** — claim `token_version` в JWT, в БД у user'а тоже. Меняем version → все старые токены invalid

---

## Login flow + Refresh Token Rotation

### Полный flow

```
1. POST /auth/login { email, password }
2. Сервер: BCrypt.Verify / Argon2.Verify на стороне password
3. Генерирует:
   - Access Token (JWT, 15 мин)
   - Refresh Token (opaque random, 7-30 дней)
4. Refresh Token хешируется (SHA256) и сохраняется в БД/Redis
5. Response:
   - Access Token → в body { "accessToken": "..." }
   - Refresh Token → httpOnly cookie

6. Клиент: Authorization: Bearer <access>
7. На 401 → POST /auth/refresh (Cookie с refresh)
8. Сервер: валидирует refresh, генерирует НОВУЮ пару (rotation)
9. Старый refresh инвалидируется (помечается ReplacedBy)
```

### Schema

```csharp
public class RefreshToken
{
    public Guid Id { get; set; }
    public string TokenHash { get; set; } = "";  // Не сам токен — только хэш!
    public Guid UserId { get; set; }
    public DateTime ExpiresAt { get; set; }
    public DateTime CreatedAt { get; set; }
    public DateTime? RevokedAt { get; set; }
    public Guid? ReplacedByTokenId { get; set; }  // Для detection кражи
    public string? RevokedReason { get; set; }
    public string CreatedByIp { get; set; } = "";
}
```

### Rotation + theft detection

При каждом refresh:
1. Validate refresh token
2. Если уже revoked — **revoke ALL family** (от первого корня) — это признак кражи
3. Сгенерировать новую пару, записать ReplacedByTokenId

```csharp
public async Task<TokenPair> RefreshAsync(string refreshToken, string ip, CancellationToken ct)
{
    var hash = HashToken(refreshToken);
    var token = await _db.RefreshTokens.FirstOrDefaultAsync(t => t.TokenHash == hash, ct)
        ?? throw new SecurityException("Invalid token");

    if (token.RevokedAt is not null)
    {
        // Кто-то использует уже revoked токен — это могло быть кражей!
        await RevokeAllUserTokensAsync(token.UserId, "Token reuse detected", ct);
        throw new SecurityException("Token reuse detected — all sessions revoked");
    }

    if (token.ExpiresAt < DateTime.UtcNow)
        throw new SecurityException("Refresh token expired");

    // Rotate
    var (newAccess, newRefresh) = await IssueTokensAsync(token.UserId, ip, ct);
    token.RevokedAt = DateTime.UtcNow;
    token.ReplacedByTokenId = newRefresh.Id;
    await _db.SaveChangesAsync(ct);

    return new TokenPair(newAccess, newRefresh.Token);
}

private static string HashToken(string token) =>
    Convert.ToHexString(SHA256.HashData(Encoding.UTF8.GetBytes(token)));
```

> [!question]- **Интервью: где безопаснее хранить токены — localStorage, sessionStorage, cookie?**
> 1. **localStorage / sessionStorage** — доступны через JS → **уязвимы к XSS**. Любой injected script читает токен.
> 2. **httpOnly cookie** — JS не имеет доступа, защищено от XSS. **Уязвимо к CSRF**, но решается через `SameSite=Strict/Lax` + CSRF token для критичных операций.
>
> Production выбор: **Refresh Token в httpOnly cookie**, **Access Token в memory** (JS-переменная, не localStorage). Доступен только на время сессии страницы, при F5 — refresh через cookie.

---

## OAuth2 — flows и когда какой

OAuth2 — фреймворк для **авторизации**. Стандартизирует как третий сервис получает доступ к ресурсам пользователя без пароля. **Не путать** с authentication — для аутентификации поверх OAuth2 есть OIDC.

### Authorization Code + PKCE — стандарт для SPA / Mobile

```
1. Client → /authorize?response_type=code&client_id=...&code_challenge=SHA256(verifier)
2. User logs in, разрешает доступ
3. Auth Server → redirect_uri?code=AUTH_CODE
4. Client → /token POST { code, code_verifier, client_id }
5. Auth Server проверяет SHA256(code_verifier) == code_challenge
6. → { access_token, refresh_token, id_token }
```

PKCE (Proof Key for Code Exchange) — защита от перехвата `code` в native приложениях / SPA. Раньше был для public clients, **теперь обязательно для всех** (RFC 9126).

```csharp
// Генерация на клиенте
var verifier = GenerateRandomString(64);  // Cryptographically random
var challenge = Base64Url.Encode(SHA256.HashData(Encoding.UTF8.GetBytes(verifier)));

// Шаг 1: Authorization request
var url = $"{authServer}/authorize?" +
    $"response_type=code&" +
    $"client_id={clientId}&" +
    $"redirect_uri={redirectUri}&" +
    $"code_challenge={challenge}&" +
    $"code_challenge_method=S256&" +
    $"state={antiCsrfToken}&" +
    $"scope=openid profile email";

// Шаг 5: Token exchange
var response = await http.PostAsync($"{authServer}/token", new FormUrlEncodedContent(new Dictionary<string, string>
{
    ["grant_type"] = "authorization_code",
    ["code"] = code,
    ["redirect_uri"] = redirectUri,
    ["client_id"] = clientId,
    ["code_verifier"] = verifier,
}));
```

### Client Credentials — для M2M

Один сервис вызывает другой, нет пользователя.

```
1. Service A → /token POST { grant_type=client_credentials, client_id, client_secret, scope }
2. Auth Server → { access_token } (no refresh — просто запрашивай новый каждый раз)
3. Service A → Service B + Bearer access_token
```

```csharp
// MS Identity Web для упрощения
builder.Services.AddDistributedTokenCaches();
builder.Services.AddTokenAcquisition();

public class ServiceBClient(ITokenAcquisition tokenAcq, HttpClient http)
{
    public async Task<Result> CallAsync()
    {
        var token = await tokenAcq.GetAccessTokenForAppAsync("api://service-b/.default");
        http.DefaultRequestHeaders.Authorization = new("Bearer", token);
        return await http.GetFromJsonAsync<Result>("/api/data");
    }
}
```

### Device Code — для устройств без браузера

Smart TV, IoT, CLI:
```
1. Device → /device { client_id }
2. → { device_code, user_code: "ABCD-EFGH", verification_uri }
3. Device показывает: "Зайди на example.com/device, введи ABCD-EFGH"
4. User делает это в браузере
5. Device polling /token → когда user одобрил → получает access_token
```

### ROPC (Resource Owner Password Credentials) — DEPRECATED

Старый flow: client получает username/password напрямую и обменивает на token. **Не используй** — нарушает принцип "пользователь не должен отдавать пароль клиенту". Заменён на Code+PKCE везде.

### Implicit flow — DEPRECATED

Старый SPA-flow без backend, где access_token приходил в URL fragment. Заменён на Code+PKCE.

> [!question]- **Интервью: какой OAuth2 flow выбрать для SPA в 2026?**
> **Authorization Code + PKCE.** Implicit flow — устаревший (deprecated в OAuth 2.1 draft). PKCE требуется даже для public clients. Refresh token — в httpOnly cookie с `SameSite=Strict`.

---

## OpenID Connect (OIDC) — Authentication поверх OAuth2

OAuth2 — про авторизацию (доступ к API). OIDC добавляет **аутентификацию** — кто такой user. Главное отличие — `id_token` (JWT с claims о пользователе).

### Discovery endpoint

```
GET /.well-known/openid-configuration
{
  "issuer": "https://auth.example.com",
  "authorization_endpoint": "...",
  "token_endpoint": "...",
  "userinfo_endpoint": "...",
  "jwks_uri": "https://auth.example.com/.well-known/jwks.json",
  "response_types_supported": ["code", "token", "id_token"],
  "scopes_supported": ["openid", "profile", "email"]
}
```

### JWKS — public keys для валидации

```
GET /.well-known/jwks.json
{
  "keys": [
    { "kty": "RSA", "kid": "key-1", "n": "...", "e": "AQAB", "use": "sig" }
  ]
}
```

В .NET валидация автоматически подтягивает JWKS:
```csharp
builder.Services.AddAuthentication("Bearer")
    .AddJwtBearer(opts =>
    {
        opts.Authority = "https://auth.example.com";  // discovery
        opts.Audience = "api://my-api";
        // RS256 ключ автоматически загружается из jwks_uri
        // Refresh каждые 24h по умолчанию
    });
```

### id_token vs access_token

| | id_token | access_token |
|--|----------|--------------|
| Кому | Клиенту (читает кто пользователь) | API (использует для доступа) |
| Audience | Client ID | Resource Server (API) |
| Содержимое | Identity claims (name, email) | Permissions (scopes) |
| Жизнь | Короткая | Короткая |

---

## Identity Provider — что выбрать

| | Keycloak | IdentityServer | Auth0 | Clerk | Supabase Auth | Microsoft Entra ID |
|--|----------|----------------|-------|-------|----------------|---------------------|
| Тип | Self-hosted | Self-hosted (commercial с .NET 8) | SaaS | SaaS | SaaS | SaaS (Azure) |
| Цена | Free OSS | Платный | Per-user $$$ | Per-user $$ | Per-MAU $ | Per-user $$ |
| Setup | Сложно (Postgres + конфиг) | Сложно (свой код) | Очень просто | Очень просто | Просто | Сложно (admin portal) |
| Кастомизация | Высокая | Максимальная | Средняя | Средняя | Средняя | Низкая |
| Self-service signup | Да | Свой код | Да | Да (lit) | Да | Зависит от tier |
| Multi-tenant | Через realms | Свой код | Да | Да | Через RLS | Через tenants |
| .NET integration | OIDC | Native | OIDC | OIDC | OIDC | Native |
| Когда | Enterprise on-prem | Полный контроль над auth | Стартап быстро запустить | SaaS B2C, low-budget | Already on Supabase | Microsoft tenant |

**Решающие факторы:**
- Ты хочешь **владеть данными пользователей** → Keycloak / IdentityServer / Supabase
- Ты хочешь **минимум кода** → Clerk / Auth0
- Ты на **Microsoft stack уже** → Entra ID
- У тебя **Postgres** уже стоит → Supabase Auth (он рядом)

### IdentityServer — пример с Duende

```bash
dotnet add package Duende.IdentityServer
```

```csharp
// IdentityServer host
builder.Services.AddIdentityServer(opts =>
{
    opts.IssuerUri = "https://auth.example.com";
})
    .AddInMemoryIdentityResources(IdentityConfig.IdentityResources)
    .AddInMemoryApiScopes(IdentityConfig.ApiScopes)
    .AddInMemoryClients(IdentityConfig.Clients)
    .AddAspNetIdentity<User>()
    .AddDeveloperSigningCredential();  // В prod — реальный сертификат

app.UseIdentityServer();

// Конфиг
public static class IdentityConfig
{
    public static IEnumerable<Client> Clients =>
    [
        new Client
        {
            ClientId = "spa",
            ClientName = "SPA App",
            AllowedGrantTypes = GrantTypes.Code,
            RequirePkce = true,
            RequireClientSecret = false,  // Public client
            RedirectUris = ["https://app.example.com/callback"],
            PostLogoutRedirectUris = ["https://app.example.com"],
            AllowedCorsOrigins = ["https://app.example.com"],
            AllowedScopes = ["openid", "profile", "api"],
            AllowOfflineAccess = true,  // refresh tokens
            AccessTokenLifetime = 900,    // 15 мин
            RefreshTokenUsage = TokenUsage.OneTimeOnly,  // rotation
        },
    ];
}
```

> [!question]- **Интервью: Duende IdentityServer стал платным с 2022. Что делать?**
> Альтернативы:
> 1. **OpenIddict** — open-source, MIT, .NET-native, активно развивается
> 2. **Keycloak** — JVM-stack, но best-in-class OSS IDP
> 3. **Использовать managed (Auth0/Clerk/Supabase)**
> Проекты на старом IdentityServer4 — **deprecated** с 2024, security patches не выходят. Срочно мигрируй.

---

## API Key authentication

Простой вариант для M2M или внутренних сервисов. Клиент шлёт `X-API-Key: ...` header.

```csharp
public class ApiKeyAuthenticationHandler(
    IOptionsMonitor<ApiKeyAuthOptions> options,
    ILoggerFactory logger,
    UrlEncoder encoder,
    IApiKeyValidator validator)
    : AuthenticationHandler<ApiKeyAuthOptions>(options, logger, encoder)
{
    protected override async Task<AuthenticateResult> HandleAuthenticateAsync()
    {
        if (!Request.Headers.TryGetValue("X-API-Key", out var apiKey))
            return AuthenticateResult.NoResult();

        var result = await validator.ValidateAsync(apiKey!, Request.HttpContext.RequestAborted);
        if (result is null)
            return AuthenticateResult.Fail("Invalid API key");

        var claims = new[]
        {
            new Claim(ClaimTypes.NameIdentifier, result.ClientId),
            new Claim("api_scope", result.Scope),
        };
        var identity = new ClaimsIdentity(claims, Scheme.Name);
        var principal = new ClaimsPrincipal(identity);
        return AuthenticateResult.Success(new AuthenticationTicket(principal, Scheme.Name));
    }
}

// Регистрация
builder.Services.AddAuthentication()
    .AddScheme<ApiKeyAuthOptions, ApiKeyAuthenticationHandler>("ApiKey", _ => { });
```

### Best practices для API keys

```csharp
public sealed class ApiKey
{
    public Guid Id { get; set; }
    public string Prefix { get; set; } = "";       // "sk_live_" — видимая часть, по ней findable
    public string KeyHash { get; set; } = "";       // SHA-256(full_key)
    public string ClientId { get; set; } = "";
    public DateTime ExpiresAt { get; set; }
    public DateTime? RevokedAt { get; set; }
    public DateTime LastUsedAt { get; set; }
    public string[] Scopes { get; set; } = [];
    public string? IpWhitelist { get; set; }        // CIDR
}

// Genesis (показываем user 1 раз!)
public async Task<string> CreateAsync(string clientId, CancellationToken ct)
{
    var raw = "sk_live_" + Convert.ToHexString(RandomNumberGenerator.GetBytes(32)).ToLowerInvariant();
    var hash = Convert.ToHexString(SHA256.HashData(Encoding.UTF8.GetBytes(raw)));

    _db.ApiKeys.Add(new ApiKey { Prefix = raw[..12], KeyHash = hash, ClientId = clientId });
    await _db.SaveChangesAsync(ct);

    return raw;  // ТОЛЬКО ЗДЕСЬ — больше клиент его не увидит
}
```

Никогда не храни сырой API key — только хэш. Если БД утечёт, ключи невозможно восстановить.

---

## mTLS — Mutual TLS

Двусторонняя проверка через сертификаты. Клиент и сервер каждый шлёт свой cert. Используется в **zero-trust** сетях, **финансовых API**, M2M в Kubernetes (через service mesh).

### Setup в Kestrel

```csharp
builder.WebHost.ConfigureKestrel(options =>
{
    options.ConfigureHttpsDefaults(httpsOptions =>
    {
        httpsOptions.ClientCertificateMode = ClientCertificateMode.RequireCertificate;
        httpsOptions.ClientCertificateValidation = (cert, chain, errors) =>
        {
            // Custom validation
            return cert.Issuer == "CN=Trusted CA" && errors == SslPolicyErrors.None;
        };
    });
});

builder.Services.AddAuthentication(CertificateAuthenticationDefaults.AuthenticationScheme)
    .AddCertificate(options =>
    {
        options.AllowedCertificateTypes = CertificateTypes.Chained;
        options.ValidateCertificateUse = true;
        options.ValidateValidityPeriod = true;
        options.Events = new CertificateAuthenticationEvents
        {
            OnCertificateValidated = ctx =>
            {
                var claims = new[]
                {
                    new Claim(ClaimTypes.NameIdentifier, ctx.ClientCertificate.Subject),
                    new Claim(ClaimTypes.Thumbprint, ctx.ClientCertificate.Thumbprint),
                };
                ctx.Principal = new ClaimsPrincipal(new ClaimsIdentity(claims, ctx.Scheme.Name));
                ctx.Success();
                return Task.CompletedTask;
            },
        };
    });
```

### Когда mTLS

- Между сервисами в **service mesh** (Istio, Linkerd) — автоматический mTLS, ты ничего не настраиваешь
- B2B integrations (банки, payment processors)
- IoT устройства, идентифицирующие себя сертификатами
- Internal admin endpoints (доп. слой защиты сверх JWT)

mTLS дороже по setup и operations (rotation сертификатов!), но даёт **сильную идентификацию без secrets**.

---

## Policy-based Authorization

```csharp
builder.Services.AddAuthorization(opts =>
{
    opts.AddPolicy("AdminOnly", policy => policy.RequireRole("Admin"));

    opts.AddPolicy("MinAge18", policy =>
        policy.RequireAssertion(ctx =>
        {
            var birthDate = ctx.User.FindFirstValue("birth_date");
            return birthDate != null && DateTime.Parse(birthDate).AddYears(18) <= DateTime.Today;
        }));

    opts.AddPolicy("CanEditOrder", policy =>
        policy.Requirements.Add(new SameAuthorRequirement()));

    // Combine — несколько requirements
    opts.AddPolicy("PaidUsersOnly", policy => policy
        .RequireAuthenticatedUser()
        .RequireClaim("subscription", "active")
        .RequireRole("paid"));
});

public class SameAuthorRequirement : IAuthorizationRequirement { }

public class SameAuthorHandler : AuthorizationHandler<SameAuthorRequirement, Order>
{
    protected override Task HandleRequirementAsync(
        AuthorizationHandlerContext context,
        SameAuthorRequirement requirement,
        Order resource)
    {
        var userId = context.User.FindFirstValue(ClaimTypes.NameIdentifier);
        if (resource.CreatedByUserId == userId)
            context.Succeed(requirement);
        return Task.CompletedTask;
    }
}

builder.Services.AddSingleton<IAuthorizationHandler, SameAuthorHandler>();
```

### Resource-based — проверка доступа к конкретному объекту

```csharp
[HttpPut("{id}")]
public async Task<IActionResult> Update(int id, UpdateOrderDto dto)
{
    var order = await _repo.GetByIdAsync(id);
    if (order is null) return NotFound();

    var authResult = await _authService.AuthorizeAsync(User, order, "CanEditOrder");
    if (!authResult.Succeeded) return Forbid();

    // ... update
}
```

### Permission-based вместо Roles

Roles слишком грубы для сложных приложений. **Permission-based**: явный список разрешений в claims.

```csharp
// Login: подгружаем permissions, кладём в claims
var permissions = await _permissionService.GetForUserAsync(user.Id);
foreach (var p in permissions)
    claims.Add(new Claim("permission", p));

// Policy
opts.AddPolicy("CanDeleteOrders", p => p.RequireClaim("permission", "orders.delete"));

// Handler
[Authorize(Policy = "CanDeleteOrders")]
public async Task<IActionResult> Delete(int id) { ... }
```

Преимущество: можешь менять permissions без редактирования кода. Один permission — один atomic action.

---

## Получение текущего пользователя

```csharp
// Controller / Minimal API
var userId = User.FindFirstValue(ClaimTypes.NameIdentifier);
var email = User.FindFirstValue(ClaimTypes.Email);
var roles = User.FindAll(ClaimTypes.Role).Select(c => c.Value);
var isAdmin = User.IsInRole("Admin");

app.MapGet("/me", (ClaimsPrincipal user) =>
{
    return Results.Ok(new { userId = user.FindFirstValue(ClaimTypes.NameIdentifier) });
});

// В сервисе — через IHttpContextAccessor
builder.Services.AddHttpContextAccessor();

public class CurrentUserService(IHttpContextAccessor accessor) : ICurrentUserService
{
    public string? UserId => accessor.HttpContext?.User.FindFirstValue(ClaimTypes.NameIdentifier);
    public bool IsAuthenticated => accessor.HttpContext?.User.Identity?.IsAuthenticated == true;
    public IEnumerable<string> Permissions =>
        accessor.HttpContext?.User.FindAll("permission").Select(c => c.Value) ?? [];
}
```

> [!question]- **Интервью: какие проблемы с `IHttpContextAccessor`?**
> 1. **Не потокобезопасен** — нельзя использовать в background-задачах. Сохраняй нужные данные (userId) до выхода из request scope.
> 2. **Singleton scope problem** — `IHttpContextAccessor` сам Singleton, но `HttpContext` per-request. Не забыть зарегистрировать через `AddHttpContextAccessor()`.
> 3. **Implicit dependency** — сервис, использующий `IHttpContextAccessor`, перестаёт быть testable без HTTP-контекста. Лучше прокинь userId через метод/конструктор как параметр сервиса.

### Claims transformation

```csharp
public class PermissionClaimsTransformation(IPermissionService permissions, IMemoryCache cache) : IClaimsTransformation
{
    public async Task<ClaimsPrincipal> TransformAsync(ClaimsPrincipal principal)
    {
        if (!principal.Identity?.IsAuthenticated ?? true)
            return principal;

        var userId = principal.FindFirstValue(ClaimTypes.NameIdentifier);
        if (userId is null) return principal;

        // Cache на 5 минут
        var perms = await cache.GetOrCreateAsync($"perms:{userId}", async entry =>
        {
            entry.AbsoluteExpirationRelativeToNow = TimeSpan.FromMinutes(5);
            return await permissions.GetForUserAsync(Guid.Parse(userId));
        });

        var identity = (ClaimsIdentity)principal.Identity!;
        foreach (var p in perms!)
            identity.AddClaim(new Claim("permission", p));

        return principal;
    }
}

builder.Services.AddTransient<IClaimsTransformation, PermissionClaimsTransformation>();
```

Вызывается на **каждом запросе** — кэшируй обязательно. Иначе DB-запрос на каждый эндпойнт.

---

## Cookie Authentication для Blazor Server / MVC

```csharp
builder.Services.AddAuthentication(CookieAuthenticationDefaults.AuthenticationScheme)
    .AddCookie(opts =>
    {
        opts.Cookie.Name = "MyApp.Auth";
        opts.Cookie.HttpOnly = true;
        opts.Cookie.SecurePolicy = CookieSecurePolicy.Always;
        opts.Cookie.SameSite = SameSiteMode.Strict;
        opts.ExpireTimeSpan = TimeSpan.FromHours(8);
        opts.SlidingExpiration = true;
        opts.LoginPath = "/login";
        opts.LogoutPath = "/logout";
        opts.AccessDeniedPath = "/forbidden";
        opts.Events = new CookieAuthenticationEvents
        {
            OnSigningIn = ctx =>
            {
                // Custom logic при логине
                return Task.CompletedTask;
            },
            OnValidatePrincipal = async ctx =>
            {
                // Каждые N минут проверяем что user не забанен
                if (ctx.Properties.IssuedUtc < DateTimeOffset.UtcNow.AddMinutes(-5))
                {
                    var userId = ctx.Principal!.FindFirstValue(ClaimTypes.NameIdentifier);
                    var stillActive = await CheckUserStillActiveAsync(userId!);
                    if (!stillActive)
                    {
                        ctx.RejectPrincipal();
                        await ctx.HttpContext.SignOutAsync();
                    }
                }
            },
        };
    });
```

### Login

```csharp
app.MapPost("/login", async (LoginDto dto, SignInManager<User> signIn) =>
{
    var result = await signIn.PasswordSignInAsync(dto.Email, dto.Password, isPersistent: dto.RememberMe, lockoutOnFailure: true);
    if (!result.Succeeded) return Results.Unauthorized();
    return Results.Ok();
});

app.MapPost("/logout", async (HttpContext ctx) =>
{
    await ctx.SignOutAsync();
    return Results.Ok();
});
```

---

## Secrets management

### Никогда

- ❌ Hardcode в коде / конфиге
- ❌ Commit в git
- ❌ Plaintext в `appsettings.json`

### В development

```bash
dotnet user-secrets init
dotnet user-secrets set "Jwt:Key" "..."
```

`secrets.json` хранится в `%APPDATA%/Microsoft/UserSecrets/<id>/` — вне репозитория.

### В production

| Способ | Когда |
|--------|-------|
| **Environment variables** | Минимум, для простых случаев |
| **Azure Key Vault / AWS Secrets Manager** | Cloud apps, ротация автоматизирована |
| **HashiCorp Vault** | On-prem или multi-cloud |
| **Kubernetes Secrets** | K8s native, но шифровать через encryption-at-rest или sealed-secrets |
| **SOPS / Age** | GitOps — зашифрованные secrets в git |

```csharp
// Azure Key Vault integration
builder.Configuration.AddAzureKeyVault(
    new Uri($"https://{keyVaultName}.vault.azure.net"),
    new DefaultAzureCredential());
```

### Secret rotation

JWT signing keys, DB passwords, API keys — все должны ротироваться периодически. Стратегии:
- **Multiple active secrets** — старый и новый одновременно валидны несколько часов/дней. JWT — через `kid` в header
- **Re-deploy after rotation** — простейший: смена secret + redeploy
- **Hot reload через `IOptionsMonitor`** — при rotation Configuration перечитывается, secret применяется без рестарта

---

## Security Headers

```csharp
app.Use(async (ctx, next) =>
{
    ctx.Response.Headers.Append("X-Content-Type-Options", "nosniff");
    ctx.Response.Headers.Append("X-Frame-Options", "DENY");
    ctx.Response.Headers.Append("Referrer-Policy", "strict-origin-when-cross-origin");
    ctx.Response.Headers.Append("Permissions-Policy", "geolocation=(), microphone=(), camera=()");
    ctx.Response.Headers.Append("Content-Security-Policy",
        "default-src 'self'; script-src 'self' 'unsafe-inline'; style-src 'self' 'unsafe-inline'; img-src 'self' data:");
    await next();
});

app.UseHsts();  // HTTP Strict Transport Security
app.UseHttpsRedirection();
```

| Header | Защита от |
|--------|-----------|
| `Strict-Transport-Security` | MITM, downgrade to HTTP |
| `X-Content-Type-Options: nosniff` | MIME sniffing attack |
| `X-Frame-Options: DENY` | Clickjacking |
| `Content-Security-Policy` | XSS, code injection |
| `Referrer-Policy` | Утечка URL в referer |
| `Permissions-Policy` | Доступ к camera/mic/geo |

Используй пакет **NWebsec** или **NetEscapades.AspNetCore.SecurityHeaders** для типизированного API.

---

## CSRF / XSRF Protection

При cookie-based auth атакующий может заставить браузер пользователя отправить запрос с автоматически прикреплённой cookie ("submit-form-on-load атака").

### `SameSite` cookie

`SameSite=Strict` — cookie не отправляется на cross-origin запросах. **Лучшая защита**.

`SameSite=Lax` — cookie шлётся на top-level navigation (клик ссылку), не на POST/AJAX. Default в современных браузерах.

### CSRF token (когда `SameSite` недостаточно)

```csharp
builder.Services.AddAntiforgery(options =>
{
    options.HeaderName = "X-CSRF-TOKEN";
    options.SuppressXFrameOptionsHeader = false;
});

app.UseAntiforgery();

// Endpoint выдаёт токен
app.MapGet("/csrf-token", (IAntiforgery antiforgery, HttpContext ctx) =>
{
    var tokens = antiforgery.GetAndStoreTokens(ctx);
    return Results.Json(new { token = tokens.RequestToken });
});

// Клиент шлёт токен в header на каждый POST
fetch('/api/transfer', {
    method: 'POST',
    headers: { 'X-CSRF-TOKEN': csrfToken },
    body: JSON.stringify({ amount: 100 }),
    credentials: 'include',
});
```

JWT auth (без cookie) — **не уязвим** к CSRF. Атакующий не может прикрепить ваш токен из своего домена.

---

## CORS — детали

```csharp
builder.Services.AddCors(options =>
{
    options.AddPolicy("ProductionApi", policy =>
    {
        policy.WithOrigins("https://app.example.com", "https://admin.example.com")
              .WithMethods("GET", "POST", "PUT", "DELETE")
              .WithHeaders("Authorization", "Content-Type", "X-CSRF-TOKEN")
              .AllowCredentials()  // ВАЖНО для cookies
              .SetPreflightMaxAge(TimeSpan.FromMinutes(10));
    });
});

app.UseCors("ProductionApi");
```

### Preflight

Браузер делает `OPTIONS` запрос перед "сложным" запросом (не GET/HEAD/POST с simple content-type). Сервер должен ответить корректными headers — иначе основной запрос даже не начнётся.

```
OPTIONS /api/data
Origin: https://app.example.com
Access-Control-Request-Method: PUT
Access-Control-Request-Headers: Authorization

→ 204 No Content
Access-Control-Allow-Origin: https://app.example.com
Access-Control-Allow-Methods: GET, POST, PUT, DELETE
Access-Control-Allow-Headers: Authorization, Content-Type
Access-Control-Allow-Credentials: true
Access-Control-Max-Age: 600
```

### Pitfall — `AllowAnyOrigin()` + `AllowCredentials()`

Браузер блокирует комбинацию `*` origin + credentials. Нужно явно перечислить origins. Это by design (защита от credential theft).

### Pitfall — `Vary: Origin` обязателен при reflected origin

Как только `Access-Control-Allow-Origin` **вычисляется из** заголовка `Origin` запроса (а не статичный `*`), ответ становится **origin-dependent**: для origin A сервер вернёт `Access-Control-Allow-Origin: https://a.example`, для origin B — `https://b.example`. `WithOrigins(...)` с whitelist именно так и работает — он отражает совпавший origin обратно.

Проблема — **shared / CDN cache**. Если кэш ключует ответ только по URL (метод + path + query), он не различает запросы от разных origin. Первый клиент (origin A) прогревает кэш, в кэш попадает ответ с `Access-Control-Allow-Origin: https://a.example`. Следующий запрос на тот же URL от origin B **получает закэшированный ответ с CORS-заголовками origin A**. Это cache poisoning: либо origin B видит чужие CORS-разрешения (и его браузер блокирует легитимный запрос), либо — хуже — `Allow-Credentials: true` с чужим origin утекает в браузер, который не должен был его получить.

Лекарство — добавить `Origin` в ключ кэширования через заголовок `Vary`. Тогда кэш хранит **отдельную запись на каждый origin**, и ответ origin A никогда не отдаётся origin B.

```csharp
app.Use(async (ctx, next) =>
{
    ctx.Response.OnStarting(() =>
    {
        // Если CORS-middleware отразил Origin в ACAO — кэш ОБЯЗАН варьировать по Origin
        if (ctx.Response.Headers.ContainsKey("Access-Control-Allow-Origin"))
            ctx.Response.Headers.Append("Vary", "Origin");
        return Task.CompletedTask;
    });
    await next(ctx);
});
```

Практические правила:

- Любой ответ с reflected origin → `Vary: Origin` (а при reflected `Access-Control-Request-Headers` ещё и `Vary: Access-Control-Request-Headers`).
- Статичный `Access-Control-Allow-Origin: *` без credentials — `Vary` не нужен (ответ одинаков для всех).
- На CDN (CloudFront/Fastly/nginx `proxy_cache`) проверь, что `Vary: Origin` **уважается** прокси, а не вырезается — иначе защита бесполезна. См. [[pipeline-middleware|Pipeline & Middleware]] (`Response.OnStarting` — добавление заголовков на этапе отправки).

> [!warning]
> `Vary: Origin` сам по себе **не** включает CORS — он только разделяет cache-записи. Без него отравление кэша возможно даже при идеально настроенном whitelist.

### Fail-closed per-route policy dispatch

Когда у разных маршрутов разные CORS-правила (публичный API с `*`, partner API с whitelist, внутренний — closed), нужен явный диспетчер политики **на каждый matched route**. Два требования делают его безопасным:

1. **Match по path, IGNORING HTTP-метод.** Preflight — это всегда `OPTIONS` **без** `Authorization` и без тела, и его метод не равен методу основного запроса (реальный метод лежит в `Access-Control-Request-Method`). Если резолвер политики матчит по паре (path + method), `OPTIONS`-preflight не найдёт политику `POST`-маршрута → CORS-заголовки не выставятся → браузер заблокирует основной запрос. Матчим **только по path**, чтобы preflight и реальный запрос разрешались в одну и ту же политику.
2. **Fail-closed порядок резолва origin** — при неоднозначности отвергаем, а не открываем:

```csharp
static string? ResolveAllowedOrigin(
    string requestOrigin,
    RoutePolicy policy)
{
    // 1. Явный резолвер (per-tenant из БД, динамический whitelist)
    if (policy.OriginResolver is { } resolver && resolver(requestOrigin) is { } resolved)
        return resolved;

    // 2. Явный публичный '*' — ТОЛЬКО если маршрут осознанно помечен Public
    if (policy.IsExplicitlyPublic)
        return "*";  // credentials на таком маршруте запрещены by design

    // 3. Статический whitelist — отражаем origin, если он в списке
    if (policy.Whitelist.Contains(requestOrigin))
        return requestOrigin;

    // 4. Ничего не совпало → reject (НЕ возвращаем '*' неявно)
    return null;
}
```

Ключевой инвариант: **`*` никогда не появляется неявно** как fallback. Origin отражается только если он прошёл явный резолвер или присутствует в whitelist; `*` — только для маршрута, который автор кода **явно** объявил публичным (и где credentials отключены — см. пинфол `AllowAnyOrigin()` + `AllowCredentials()` выше). Любая «дыра» в правилах схлопывается в `null` → запрос отвергается. Это и есть fail-closed: ошибка конфигурации приводит к отказу, а не к открытому для всех API.

Порядок middleware для такого диспетчера — тот же, что у встроенного CORS: **после Routing, до Auth**, чтобы preflight (`OPTIONS` без `Authorization`) проходил и чтобы политика читалась с уже выбранного endpoint. См. [[pipeline-middleware|Pipeline & Middleware]] (порядок: Routing → CORS → Authentication → Authorization).

---

## Multi-tenant authorization

```csharp
public class TenantRequirement : IAuthorizationRequirement
{
    public string TenantClaim { get; } = "tenant_id";
}

public class TenantHandler(IHttpContextAccessor http) : AuthorizationHandler<TenantRequirement>
{
    protected override Task HandleRequirementAsync(AuthorizationHandlerContext context, TenantRequirement requirement)
    {
        var userTenantId = context.User.FindFirstValue(requirement.TenantClaim);
        var routeTenantId = http.HttpContext?.Request.RouteValues["tenantId"]?.ToString();

        if (userTenantId is null || routeTenantId is null)
            return Task.CompletedTask;

        if (userTenantId == routeTenantId)
            context.Succeed(requirement);

        return Task.CompletedTask;
    }
}

opts.AddPolicy("SameTenant", p => p.AddRequirements(new TenantRequirement()));

// Endpoint
app.MapGet("/api/{tenantId}/orders", (...) => ...).RequireAuthorization("SameTenant");
```

См. [[postgresql-deep|PostgreSQL Deep]] — Row-Level Security как **нижний** слой защиты в многотенантной системе.

---

## Production checklist

- [ ] HTTPS only + HSTS включён
- [ ] JWT signing key в secrets manager (не в config)
- [ ] JWT TTL ≤ 15 минут, refresh tokens с rotation
- [ ] Refresh tokens хешируются перед сохранением
- [ ] `ClockSkew = TimeSpan.Zero`
- [ ] `ValidateIssuerSigningKey = true`, `ValidIssuer/Audience` явно проверяются
- [ ] Token revocation: blacklist для `jti` или token versioning
- [ ] Все cookies — `HttpOnly + Secure + SameSite=Strict`
- [ ] Antiforgery middleware при cookie-based auth
- [ ] Security headers (HSTS, CSP, X-Content-Type-Options, X-Frame-Options)
- [ ] CORS — explicit origins (не `*` с credentials)
- [ ] Rate limiting на login endpoint (anti-brute-force)
- [ ] Account lockout после N failed attempts
- [ ] Password hashing — Argon2 (или bcrypt с work factor ≥ 12)
- [ ] Multi-factor auth опция для админов
- [ ] Audit log всех auth-событий (login, logout, failed, role change)
- [ ] Logging без чувствительных данных (никаких passwords/tokens в логах!)
- [ ] DataProtection keys persisted (Redis / Azure / file system)
- [ ] DataProtection — обновление до версии без [[security-practices|CVE-2026-40372]]

---

## Security advisories

| CVE | Affected | Fix |
|-----|----------|-----|
| **CVE-2026-40372** | DataProtection 10.0.0–10.0.6 | Update to 10.0.7+ |
| Old IdentityServer4 | Все версии до 10/2024 | Migrate to OpenIddict |

См. подробнее в [[security-practices|security-practices.md]].

---

## Common pitfalls

### 1. JWT в localStorage
XSS = угон токена. Используй httpOnly cookie или memory + refresh из cookie.

### 2. JWT без `aud` валидации
Атакующий может использовать токен от другого resource server'а у тебя. `ValidateAudience = true` обязательно.

### 3. Не валидируешь signature key
Алгоритм `none` атака. `ValidateIssuerSigningKey = true` всегда.

### 4. Refresh token = JWT
Refresh должен быть **opaque** — random bytes, lookup по hash. JWT refresh = неотзываемый refresh.

### 5. Логирование ClaimsPrincipal в plaintext
В User claims часто есть PII. Не пиши `_logger.LogInformation("User: {User}", User)` — личные данные в логе.

### 6. Forgot HttpOnly на cookie
JS читает auth cookie → XSS = угон. Всегда `HttpOnly = true`.

### 7. CORS `AllowAnyOrigin()` в проде
Любой сайт может звать твой API с user credentials. Только explicit list. Смежные CORS-футганы в разделе «CORS — детали»: пропущенный `Vary: Origin` при reflected origin (cache poisoning через shared/CDN cache) и per-route dispatch, который матчит по методу и потому теряет `OPTIONS`-preflight.

### 8. `[Authorize]` забыли на endpoint
Каждый новый endpoint default authorize → используй `FallbackPolicy.RequireAuthenticatedUser()`. Открывай через `[AllowAnonymous]` явно.

### 9. Permissions проверяются только на UI
"Я скрыл кнопку Delete" — но API endpoint её не проверяет, atтакующий просто curl'ит. Проверка **обязательно** на backend.

### 10. Long-lived API keys без rotation
API ключи живут годами, ушёл сотрудник — все ключи действуют. Принудительная rotation каждые 90 дней + monitoring "lastUsedAt".

---

## Case Studies

### Case Study #1 — JWT secret rotation

**Сценарий:** Подозрение что JWT secret leaked. Нужно rotate без обнуления всех sessions.

**❌ Plain rotation:**
```csharp
// Меняем secret → все JWT становятся invalid → все users logged out
builder.Configuration["Jwt:Secret"] = newSecret;
```

**✅ Multi-key validation (graceful rotation):**
```csharp
builder.Services.AddAuthentication("Bearer")
    .AddJwtBearer(options =>
    {
        options.TokenValidationParameters = new()
        {
            IssuerSigningKeys = new[]
            {
                new SymmetricSecurityKey(Encoding.UTF8.GetBytes(newSecret)),  // primary
                new SymmetricSecurityKey(Encoding.UTF8.GetBytes(oldSecret))   // accept until expiry
            },
            ValidIssuer = "myapp",
            ValidAudience = "myapp"
        };
    });
```

После 24h (max JWT TTL) — удаляем oldSecret. Users не logout'ятся.

---

### Case Study #2 — Cookie-based auth + CSRF

**Сценарий:** SPA на same domain. Cookie auth простой, но CSRF.

**✅ Anti-forgery + SameSite:**
```csharp
builder.Services.AddAntiforgery(opts =>
{
    opts.HeaderName = "X-XSRF-TOKEN";
    opts.Cookie.SameSite = SameSiteMode.Strict;
    opts.Cookie.SecurePolicy = CookieSecurePolicy.Always;
});

[HttpPost("transfer")]
[ValidateAntiForgeryToken]
public IActionResult Transfer(TransferDto dto) { /* ... */ }
```

Frontend читает CSRF cookie → отправляет в header. Cross-site forge не сработает.

---

### Case Study #3 — Refresh tokens

**Сценарий:** Access tokens должны быть короткие (15 min). Но user не должен logout'иться каждые 15 минут.

**✅ Refresh token pattern:**
```csharp
public record TokenPair(string AccessToken, string RefreshToken);

[HttpPost("refresh")]
public async Task<IActionResult> Refresh(string refreshToken)
{
    var stored = await _refreshStore.GetAsync(refreshToken);
    if (stored is null || stored.ExpiresAt < DateTime.UtcNow)
        return Unauthorized();

    // Rotate refresh token (one-time use)
    await _refreshStore.RevokeAsync(refreshToken);
    var newPair = await GenerateTokenPair(stored.UserId);
    return Ok(newPair);
}
```

Access — 15 min, JWT, stateless.  
Refresh — 30 days, в DB, revocable, one-time use.

См. [[http-fundamentals|HTTP Fundamentals]] и[[nosql-databases|NoSQL]] (Redis для refresh tokens).


---

## Decision tree

```
Какой auth mechanism?
│
├── Public API (sharing с partners)?
│   ├── Stateless, scalable → JWT (Bearer)
│   ├── Need revocation → JWT + refresh tokens (DB)
│   └── 3rd party login → OAuth 2.0 / OIDC
│
├── SPA (single-page app, same domain)?
│   ├── Simple → Cookie auth + Anti-forgery
│   └── Multi-tenant → JWT в HttpOnly cookie
│
├── Mobile / native?
│   └── JWT + refresh tokens
│
├── Internal services (microservices)?
│   ├── User context propagation → JWT через gateway
│   └── Service-to-service → mTLS / API keys
│
├── Enterprise SSO?
│   ├── SAML (legacy) → AddSaml2
│   └── OIDC (modern) → Azure AD / Okta / Auth0
│
└── Permissions model?
    ├── Few static roles → Role-based (RBAC)
    ├── Granular permissions → Policy-based
    └── Resource-level → Attribute-based (ABAC)
```


---

## См. также

- [[security-practices|Security Practices]] — password hashing, timing attacks, CVE-2026-40372
- [[api-design|API Design]] — Minimal API + auth
- [[caching|Caching]] — кэш permissions через `IClaimsTransformation`
- [[postgresql-deep|PostgreSQL Deep]] — RLS как defense-in-depth для multi-tenant
- [[distributed-systems|Distributed Systems]] — propagation auth context между сервисами через OpenTelemetry baggage

## Reading list

- **OAuth 2.0 Simplified** — oauth.net/2/ (читаемое описание всех flows)
- **OAuth 2.1 draft** — datatracker.ietf.org/doc/html/draft-ietf-oauth-v2-1 (что меняется vs OAuth 2.0)
- **OWASP Top 10 2025** — owasp.org/Top10/ (Authentication, Access Control секции)
- **OWASP Authentication Cheat Sheet** — cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html
- **JWT Best Current Practices** — datatracker.ietf.org/doc/html/rfc8725
- **Microsoft — Identity Web** — learn.microsoft.com/entra/msal/dotnet/microsoft-identity-web
- **Auth0 Blog** — auth0.com/blog (отличные deep-dives несмотря на маркетинг)
- **Andrew Lock — Authentication series** — andrewlock.net/series/authentication-and-authorisation/
- **Duende IdentityServer docs** — docs.duendesoftware.com (даже если не используешь — концепции универсальны)
- **OWASP JWT Cheat Sheet** — cheatsheetseries.owasp.org/cheatsheets/JSON_Web_Token_for_Java_Cheat_Sheet.html
