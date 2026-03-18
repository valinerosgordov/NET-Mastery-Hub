---
tags: [security, tokens, hashing, timing-safe, path-traversal, owasp]
level: Senior
---

# Security Practices

## Что это, зачем и когда

### Что такое Security Practices?
**Набор конкретных приёмов защиты кода** от атак. Не «поставь фаерволл», а «как сравнивать токены», «как хранить секреты», «как защитить загрузку файлов».

**Аналогия:** Ты можешь поставить замок на дверь (HTTPS, JWT). Но если ключ лежит под ковриком (секрет в appsettings.json в git) или дверь открывается от удара (SQL injection) — замок бесполезен. Security practices — это «не клади ключ под коврик».

### Зачем?

| Без практик | С практиками |
|------------|-------------|
| Токен сравнивается через `==` → timing attack раскрывает символы | `FixedTimeEquals` — время одинаковое для любого ввода |
| Refresh token в БД как plain text → утечка БД = доступ ко всем аккаунтам | SHA256 хэш → утечка БД бесполезна для атакующего |
| Путь к файлу от пользователя: `../../etc/passwd` | Reject `..` и `\` → path traversal невозможен |
| Секреты в `appsettings.json` в git | Environment variables / User Secrets / Key Vault |

---

## Timing-Safe сравнение токенов

### Проблема: обычное `==`

```csharp
// ✗ ОПАСНО — timing attack
public bool ValidateToken(string provided, string stored)
{
    return provided == stored;
    // "aaaa..." vs "baaa..." → быстрый false (первый символ не совпал)
    // "abcd..." vs "abce..." → медленнее (совпали 3 символа, упали на 4-м)
    // Атакующий измеряет время → подбирает посимвольно
}
```

### Решение: `CryptographicOperations.FixedTimeEquals`

```csharp
using System.Security.Cryptography;
using System.Text;

// ✓ Время сравнения НЕ зависит от количества совпавших символов
public static bool ValidateToken(string provided, string stored)
{
    var providedBytes = Encoding.UTF8.GetBytes(provided);
    var storedBytes = Encoding.UTF8.GetBytes(stored);

    // Если длины разные — всё равно проверяем за одинаковое время
    if (providedBytes.Length != storedBytes.Length)
    {
        // Сравниваем stored сам с собой чтобы потратить то же время
        CryptographicOperations.FixedTimeEquals(storedBytes, storedBytes);
        return false;
    }

    return CryptographicOperations.FixedTimeEquals(providedBytes, storedBytes);
}
```

**Когда нужно:** API keys, refresh tokens, webhook signatures, HMAC — любое сравнение секретов. Для паролей — `BCrypt`/`Argon2` (у них timing-safe встроен).

---

## Хранение токенов — SHA256 хэш

### Проблема: plain text в БД

```csharp
// ✗ Plain text — утечка БД = полный доступ
public class RefreshToken
{
    public string Token { get; set; } = "abc123xyz..."; // plain text!
}
```

### Решение: хранить только хэш

```csharp
public static class TokenHasher
{
    // Генерация токена
    public static (string RawToken, string HashedToken) GenerateRefreshToken()
    {
        var rawBytes = RandomNumberGenerator.GetBytes(32); // 256 бит энтропии
        var rawToken = Convert.ToBase64String(rawBytes);
        var hashedToken = HashToken(rawToken);
        return (rawToken, hashedToken);
    }

    // Хэширование для хранения
    public static string HashToken(string token)
    {
        var bytes = SHA256.HashData(Encoding.UTF8.GetBytes(token));
        return Convert.ToBase64String(bytes);
    }
}

// Использование при логине
public sealed class AuthService(AppDbContext context, TimeProvider timeProvider)
{
    public async Task<Result<AuthResponse>> LoginAsync(LoginRequest request, CancellationToken ct)
    {
        // ... валидация логина/пароля ...

        // Генерируем refresh token
        var (rawToken, hashedToken) = TokenHasher.GenerateRefreshToken();

        // Удаляем старые refresh tokens этого пользователя
        await context.RefreshTokens
            .Where(t => t.UserId == user.Id)
            .ExecuteDeleteAsync(ct);

        // Сохраняем ТОЛЬКО хэш
        context.RefreshTokens.Add(new RefreshToken
        {
            HashedToken = hashedToken,  // хэш в БД
            UserId = user.Id,
            ExpiresAt = timeProvider.GetUtcNow().AddDays(30)
        });

        await context.SaveChangesAsync(ct);

        // Отдаём сырой токен клиенту (один раз!)
        return Result<AuthResponse>.Ok(new AuthResponse(jwtToken, rawToken));
    }

    public async Task<Result<AuthResponse>> RefreshAsync(string rawRefreshToken, CancellationToken ct)
    {
        var hashedToken = TokenHasher.HashToken(rawRefreshToken);

        // Ищем по хэшу
        var stored = await context.RefreshTokens
            .FirstOrDefaultAsync(t => t.HashedToken == hashedToken, ct);

        if (stored is null || stored.ExpiresAt < timeProvider.GetUtcNow())
            return Result<AuthResponse>.Fail(Error.Unauthorized("Auth.InvalidToken", "Invalid refresh token"));

        // ... ротация: удалить старый, создать новый ...
    }
}
```

### Почему SHA256, а не BCrypt?

| Алгоритм | Для чего | Скорость | Почему |
|----------|---------|----------|--------|
| **BCrypt/Argon2** | Пароли | Намеренно медленный | Пароли угадываемые → brute-force должен быть дорогим |
| **SHA256** | Токены | Быстрый | Токены 256 бит энтропии → brute-force невозможен, скорость не важна |

---

## Path Traversal Protection

### Проблема

```csharp
// ✗ ОПАСНО — пользователь контролирует путь
app.MapGet("/files/{fileName}", (string fileName) =>
{
    var path = Path.Combine("/uploads", fileName);
    return Results.File(path);
    // fileName = "../../etc/passwd" → читает системный файл!
    // fileName = "..\..\..\appsettings.json" → секреты приложения!
});
```

### Решение

```csharp
public static class PathSecurity
{
    public static Result<string> ValidateFileName(string fileName)
    {
        // 1. Reject путевые разделители и parent directory
        if (fileName.Contains("..") || fileName.Contains('\\') || fileName.Contains('/'))
            return Result<string>.Fail(
                Error.Validation("File.InvalidPath", "Invalid file name"));

        // 2. Reject невалидные символы
        if (fileName.IndexOfAny(Path.GetInvalidFileNameChars()) >= 0)
            return Result<string>.Fail(
                Error.Validation("File.InvalidChars", "File name contains invalid characters"));

        // 3. Собираем полный путь и проверяем что он внутри разрешённой директории
        var basePath = Path.GetFullPath("/uploads");
        var fullPath = Path.GetFullPath(Path.Combine(basePath, fileName));

        if (!fullPath.StartsWith(basePath, StringComparison.OrdinalIgnoreCase))
            return Result<string>.Fail(
                Error.Validation("File.OutsideRoot", "Access denied"));

        return Result<string>.Ok(fullPath);
    }
}

// Использование
app.MapGet("/files/{fileName}", (string fileName) =>
{
    var pathResult = PathSecurity.ValidateFileName(fileName);
    return pathResult.Match(
        path => Results.File(path),
        error => Results.Problem(error.Message, statusCode: 400));
});
```

---

## Секреты — где хранить

```csharp
// ✗ НИКОГДА — секреты в коде или appsettings.json в git
"ConnectionStrings": {
    "Default": "Server=prod;Password=SuperSecret123"  // УТЕЧКА!
}

// ✓ Уровень 1: User Secrets (только dev)
dotnet user-secrets set "Stripe:ApiKey" "sk_test_..."
// Хранится в %APPDATA%/Microsoft/UserSecrets/{id}/secrets.json

// ✓ Уровень 2: Environment Variables (production)
// Docker / CI:
// CONNECTIONSTRINGS__DEFAULT=Server=...;Password=...
builder.Configuration.AddEnvironmentVariables();

// ✓ Уровень 3: Key Vault (enterprise)
builder.Configuration.AddAzureKeyVault(
    new Uri("https://myvault.vault.azure.net"),
    new DefaultAzureCredential());
```

### Иерархия конфигурации (.NET)

```
appsettings.json (base, в git)
  ↓ override
appsettings.{Environment}.json
  ↓ override
User Secrets (dev only)
  ↓ override
Environment Variables (production)
  ↓ override
Command-line args
  ↓ override
Key Vault / Secret Manager
```

---

## CORS — правильная настройка

```csharp
// ✗ ОПАСНО — всё разрешено
builder.Services.AddCors(o => o.AddDefaultPolicy(p =>
    p.AllowAnyOrigin().AllowAnyMethod().AllowAnyHeader()));
// Любой сайт может вызывать твой API!

// ✓ Правильно — только конкретные origins
builder.Services.AddCors(o => o.AddDefaultPolicy(p =>
    p.WithOrigins("https://myapp.com", "https://admin.myapp.com")
     .WithMethods("GET", "POST", "PUT", "DELETE")
     .WithHeaders("Content-Type", "Authorization")
     .AllowCredentials()));

// Для dev — отдельная policy
if (builder.Environment.IsDevelopment())
{
    builder.Services.AddCors(o => o.AddPolicy("dev", p =>
        p.WithOrigins("http://localhost:3000")
         .AllowAnyMethod()
         .AllowAnyHeader()
         .AllowCredentials()));
}
```

---

## Input Validation — общие правила

```csharp
// 1. Всегда валидируй на границе системы (endpoint/handler)
// 2. Никогда не доверяй клиенту

// ✗ SQL Injection
var sql = $"SELECT * FROM Users WHERE Name = '{name}'";

// ✓ Параметризованный запрос
var users = await context.Users
    .Where(u => u.Name == name) // EF параметризует автоматически
    .ToListAsync(ct);

// ✗ XSS — вставка HTML от пользователя
var html = $"<div>{userInput}</div>";

// ✓ Санитизация
var html = $"<div>{WebUtility.HtmlEncode(userInput)}</div>";

// ✗ Redirect injection
return Redirect(returnUrl); // returnUrl = "https://evil.com"

// ✓ Проверка что URL локальный
if (!Url.IsLocalUrl(returnUrl))
    return Redirect("/");
return Redirect(returnUrl);
```

---

## Rate Limiting — защита от abuse

```csharp
builder.Services.AddRateLimiter(options =>
{
    // Глобальный лимит per-IP
    options.AddFixedWindowLimiter("fixed", o =>
    {
        o.PermitLimit = 100;
        o.Window = TimeSpan.FromMinutes(1);
        o.QueueLimit = 0; // не ставить в очередь — сразу 429
    });

    // Sliding window для API endpoints
    options.AddSlidingWindowLimiter("api", o =>
    {
        o.PermitLimit = 60;
        o.Window = TimeSpan.FromMinutes(1);
        o.SegmentsPerWindow = 6; // 10 запросов per 10 сек
    });

    // Строгий лимит для auth endpoints (защита от brute-force)
    options.AddFixedWindowLimiter("auth", o =>
    {
        o.PermitLimit = 5;
        o.Window = TimeSpan.FromMinutes(15);
    });

    // Ответ при превышении лимита
    options.RejectionStatusCode = StatusCodes.Status429TooManyRequests;
    options.OnRejected = async (context, ct) =>
    {
        context.HttpContext.Response.ContentType = "application/problem+json";
        await context.HttpContext.Response.WriteAsJsonAsync(new ProblemDetails
        {
            Title = "Too Many Requests",
            Status = 429,
            Detail = "Rate limit exceeded. Try again later."
        }, ct);
    };
});

app.UseRateLimiter();

// Применение к endpoint
app.MapPost("/auth/login", HandleLogin)
    .RequireRateLimiting("auth");

app.MapGroup("/api")
    .RequireRateLimiting("api");
```

### Отключение в тестах

```csharp
// WebApplicationFactory — отключаем rate limiting
public class ApiFixture : WebApplicationFactory<Program>
{
    protected override void ConfigureWebHost(IWebHostBuilder builder)
    {
        builder.ConfigureServices(services =>
        {
            // Убираем rate limiter для тестов
            services.AddRateLimiter(_ => { });
        });
    }
}
```

---

## Чеклист безопасности

| Категория | Проверка | Как |
|-----------|---------|-----|
| **Токены** | Сравнение timing-safe | `CryptographicOperations.FixedTimeEquals` |
| **Токены** | Хранение как хэш | `SHA256.HashData` |
| **Пароли** | Хэширование с солью | `BCrypt` / `Argon2` |
| **Файлы** | Path traversal protection | Reject `..`, `\`, проверка `StartsWith(basePath)` |
| **Ввод** | XSS protection | `WebUtility.HtmlEncode` |
| **SQL** | Injection protection | Параметризованные запросы (EF делает автоматически) |
| **CORS** | Конкретные origins | Никогда `AllowAnyOrigin` в production |
| **Секреты** | Не в git | User Secrets (dev) + Env Vars (prod) |
| **Rate Limiting** | Brute-force protection | `AddRateLimiter` на auth endpoints |
| **Headers** | Security headers | HSTS, X-Content-Type-Options, CSP |

---

## См. также

- [Auth и Security](auth-security.md) — JWT, Authentication, Authorization
- [Caching](caching.md) — Rate Limiting детали
- [API Design](api-design.md) — Problem Details для ошибок
