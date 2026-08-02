---
tags: [security, tokens, hashing, timing-safe, path-traversal, owasp]
level: Senior
date: 2026-06-28
---

# Security Practices

> Прикладные приёмы защиты кода: timing-safe сравнение токенов (`FixedTimeEquals`), SHA256-хэширование секретов, защита от path traversal и безопасное хранение секретов.

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

### Password Hashing Migration — смена алгоритма без поломки логина

**Когда нужно:** переходишь с PBKDF2 на BCrypt/Argon2id, либо увеличиваешь work factor. Простой swap реализации сломает логин всем существующим пользователям — старые хэши новый алгоритм не верифицирует.

**Правильный подход — multi-algorithm support + прозрачная миграция при логине:**

```csharp
// 1. Формат хэша с metadata (PHC string style)
// $pbkdf2-sha256$i=100000$<salt_base64>$<hash_base64>
// $bcrypt$v=2b$c=12$<bcrypt_hash>
// $argon2id$v=19$m=65536,t=3,p=4$<salt>$<hash>

public interface IPasswordHasher
{
    bool CanVerify(string hashedPassword);
    bool Verify(string password, string hashedPassword);
    string Hash(string password);
}

public sealed class Pbkdf2Hasher : IPasswordHasher
{
    public bool CanVerify(string h) => h.StartsWith("$pbkdf2-");
    public bool Verify(string pw, string hash) { /* parse params, verify */ }
    public string Hash(string pw) { /* для legacy — не используем для новых */ }
}

public sealed class BcryptHasher : IPasswordHasher
{
    public bool CanVerify(string h) => h.StartsWith("$bcrypt$");
    public bool Verify(string pw, string hash) => BCrypt.Net.BCrypt.Verify(pw, hash);
    public string Hash(string pw) => "$bcrypt$" + BCrypt.Net.BCrypt.HashPassword(pw, 12);
}

public sealed class PasswordService
{
    private readonly IReadOnlyList<IPasswordHasher> _hashers;
    private readonly IPasswordHasher _primary; // текущий (новый)

    public bool Verify(string password, string storedHash)
        => _hashers.First(h => h.CanVerify(storedHash)).Verify(password, storedHash);

    public bool NeedsRehash(string storedHash) => !_primary.CanVerify(storedHash);

    public string Hash(string password) => _primary.Hash(password);
}
```

**Прозрачная миграция в LoginHandler:**

```csharp
public async Task<Result<AuthResponse>> Handle(LoginCommand cmd, CancellationToken ct)
{
    var user = await _users.FindByEmailAsync(cmd.Email, ct);
    if (user is null) return Result<AuthResponse>.Fail(Error.Unauthorized(...));

    if (!_passwords.Verify(cmd.Password, user.PasswordHash))
        return Result<AuthResponse>.Fail(Error.Unauthorized(...));

    // Прозрачная миграция — пересохраняем новым алгоритмом
    if (_passwords.NeedsRehash(user.PasswordHash))
    {
        user.PasswordHash = _passwords.Hash(cmd.Password);
        await _users.UpdateAsync(user, ct);
    }

    return Result<AuthResponse>.Ok(IssueTokens(user));
}
```

**Свойства решения:**
- Без сброса паролей, без downtime
- Пользователи мигрируют естественно при первом логине
- Можно параллельно поддерживать старый алгоритм и мониторить через метрику «сколько legacy-хэшей осталось»
- Старый hasher **нельзя** удалять, пока есть хоть один такой хэш в БД → неактивные аккаунты (полгода-год без логина) форсировать через email password reset

> [!question]- **Интервью: Как сменить алгоритм хэширования паролей в проде без ломания логина?**
> Ошибка — просто заменить реализацию: старые хэши перестанут верифицироваться, массовые 401 на логине. Правильно: хранить в хэше metadata (алгоритм + параметры), поддерживать несколько hasher-ов одновременно, определять алгоритм по префиксу хэша, при успешном логине с legacy-хэшем пересохранять пароль новым алгоритмом. Для аккаунтов без активности — форсированный password reset через email. Без downtime, без сброса паролей.

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

## ADO.NET параметры — `AddWithValue` это анти-паттерн

Параметризация защищает от инъекций (см. выше), но **как** добавлять параметр — отдельный вопрос. `SqlCommand.Parameters.AddWithValue` массово встречается в туториалах и старом коде, и у него две тихие проблемы.

### Проблема 1: type inference угадывает тип неверно

`AddWithValue` выводит `SqlDbType` из CLR-типа значения в рантайме. Угадывает он не всегда так, как колонка в БД:

- `string` → `nvarchar` (Unicode), даже если колонка `varchar` (ASCII).
- `decimal` → precision/scale берутся из значения, а не из колонки → усечение или `OverflowException`.
- `DateTime` → `datetime`, хотя колонка может быть `date` или `datetime2`.

### Проблема 2: implicit conversion убивает индекс

Самое дорогое следствие. Если колонка `varchar`, а параметр уехал как `nvarchar`, SQL Server вынужден повысить тип **колонки** до `nvarchar` для сравнения (правила data type precedence). Это происходит для **каждой строки** → index seek деградирует в index scan. Запрос, который должен брать одну строку по индексу, читает всю таблицу. На больших таблицах это разница между 1 ms и секундами, и она невидима в коде — план показывает `CONVERT_IMPLICIT` в predicate.

```csharp
// ✗ Тип угадывается из значения — string всегда nvarchar
var cmd = new SqlCommand(
    "SELECT Id, Name FROM Users WHERE Email = @email", conn);
cmd.Parameters.AddWithValue("@email", email);
// колонка Email — varchar(256) с индексом → implicit nvarchar → index scan
```

### Решение: `.Add()` с явным `SqlDbType`

```csharp
// ✓ Тип и размер заданы явно — совпадают с колонкой
var cmd = new SqlCommand(
    "SELECT Id, Name FROM Users WHERE Email = @email", conn);
cmd.Parameters.Add("@email", SqlDbType.VarChar, 256).Value = email;
// для decimal — задавай precision и scale:
cmd.Parameters.Add("@amount", SqlDbType.Decimal).Value = amount;
cmd.Parameters["@amount"].Precision = 18;
cmd.Parameters["@amount"].Scale = 2;
```

Явный тип фиксирует контракт с колонкой: правильная Unicode/ASCII семантика, отсутствие неявных конверсий, рабочие индексы. Чуть многословнее — но предсказуемо.

> [!info] Npgsql и EF Core
> Для Npgsql эффект тот же, но мягче: PostgreSQL чаще приводит литералы сам. Всё равно предпочитай `NpgsqlParameter` с явным `NpgsqlDbType` для типов, где угадывание дорого (`timestamptz` vs `timestamp`, `jsonb` vs `text`, `numeric`). В EF Core этой проблемы нет — провайдер маппит CLR-тип на тип колонки из модели; см. ниже.

---

## Шифрование данных at-rest — `AesGcm`

Хэш (SHA256/BCrypt) — **односторонний**: проверить совпадение можно, восстановить значение нельзя. Когда значение надо именно **расшифровать обратно** (PII, токены доступа к внешним API, номера документов), нужно симметричное шифрование. Современный выбор — **AES-GCM**: authenticated encryption, который шифрует и одновременно проверяет целостность (tag), защищая от подмены шифротекста.

### Ключевые правила

- **Nonce уникален на каждое шифрование.** Повтор nonce с тем же ключом полностью ломает GCM (раскрывается XOR открытых текстов и подделывается tag). Генерировать через `RandomNumberGenerator`, **никогда** не переиспользовать.
- Nonce не секретный — хранить рядом с шифротекстом (он нужен для расшифровки).
- Ключ — из секрет-менеджера/Key Vault, не из кода. 32 байта = AES-256.
- Tag (16 байт) проверяет целостность: при подмене данных `Decrypt` бросит `AuthenticationTagMismatchException` — это инфраструктурный сбой, не flow control.

```csharp
using System.Security.Cryptography;

public sealed class FieldEncryptor(byte[] key) // key: 32 bytes из Key Vault
{
    // Формат хранения: [nonce(12)][tag(16)][ciphertext] → одна строка/blob
    public byte[] Encrypt(ReadOnlySpan<byte> plaintext)
    {
        var nonce = new byte[AesGcm.NonceByteSizes.MaxSize];   // 12 байт
        RandomNumberGenerator.Fill(nonce);                     // уникальный nonce!

        var tag = new byte[AesGcm.TagByteSizes.MaxSize];       // 16 байт
        var ciphertext = new byte[plaintext.Length];

        using var aes = new AesGcm(key, tag.Length);
        aes.Encrypt(nonce, plaintext, ciphertext, tag);

        return [.. nonce, .. tag, .. ciphertext];              // collection expression
    }

    public byte[] Decrypt(ReadOnlySpan<byte> stored)
    {
        var nonce = stored[..12];
        var tag = stored[12..28];
        var ciphertext = stored[28..];
        var plaintext = new byte[ciphertext.Length];

        using var aes = new AesGcm(key, tag.Length);
        aes.Decrypt(nonce, ciphertext, tag, plaintext);        // бросит при подмене tag
        return plaintext;
    }
}
```

> [!warning] Когда НЕ писать своё шифрование
> Для cookie, antiforgery, Identity-токенов **не** изобретай это сам — используй `IDataProtector` (ASP.NET Core Data Protection): он управляет ключами, ротацией и форматом. `AesGcm` напрямую — для шифрования конкретных полей БД, где ты контролируешь хранение nonce+tag.

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

## Security Advisories

Отслеживать [dotnet/announcements](https://github.com/dotnet/announcements/issues) и `dotnet list package --vulnerable --include-transitive` в CI.

### CVE-2026-40372 — ASP.NET Core Data Protection (Апрель 2026)

**Затронуты:** `Microsoft.AspNetCore.DataProtection` версий **10.0.0 – 10.0.6** включительно.

**Суть:** в 10.0.6 попала регрессия — управляемый энкриптор вычислял HMAC-тег не над теми байтами полезной нагрузки, после чего результат выбрасывался. Параллельно открыта возможность повышения привилегий (elevation of privilege).

**Фикс:** обновиться до **10.0.7** (внеплановый патч).

```bash
dotnet add package Microsoft.AspNetCore.DataProtection --version 10.0.7
dotnet --info                        # проверить версию SDK
dotnet list package --vulnerable     # по всем проектам
```

**Что задето в приложении (всё, что использует Data Protection):**
- `IDataProtector` — явное шифрование
- Antiforgery tokens (CSRF)
- Authentication cookies и `TempData`
- Identity user tokens (reset password, confirm email)
- Protected query string parameters
- Blazor Server (использует Data Protection для SignalR connection tokens)

**После обновления:**
1. Пересборка и передеплой
2. Рестарт приложения — Data Protection инициализируется на старте
3. Формат совместим с 10.0.x — существующие cookie/сессии не инвалидируются

**Автоматизация в CI** (GitHub Actions):

```yaml
- name: Check vulnerable packages
  run: |
    dotnet restore
    dotnet list package --vulnerable --include-transitive
    # Fail job if any vulnerabilities found
    if dotnet list package --vulnerable --include-transitive | grep -q '>'; then
      echo "::error::Vulnerable packages detected"
      exit 1
    fi
```

> [!question]- **Интервью: Как отслеживать security advisories для .NET-зависимостей?**
> Три слоя: (1) подписка на [dotnet/announcements](https://github.com/dotnet/announcements/issues) и Microsoft Security Advisories — приходят в RSS/email; (2) `dotnet list package --vulnerable --include-transitive` в CI на каждом build — ловит публичные CVE через NuGet feed; (3) dependabot/renovate на репо — автоматические PR на обновление. Для критичных CVE (как CVE-2026-40372) — алерт в Slack/email из CI job, отдельный hotfix-пайплайн для срочных патчей в прод.

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
| **SQL** | Корректный тип параметра | `.Add(name, SqlDbType...)`, не `AddWithValue` |
| **Шифрование** | Обратимое шифрование PII | `AesGcm` с уникальным nonce; для cookie — `IDataProtector` |
| **CORS** | Конкретные origins | Никогда `AllowAnyOrigin` в production |
| **Секреты** | Не в git | User Secrets (dev) + Env Vars (prod) |
| **Rate Limiting** | Brute-force protection | `AddRateLimiter` на auth endpoints |
| **Headers** | Security headers | HSTS, X-Content-Type-Options, CSP |

---

## См. также

- [[auth-security|Auth и Security]] — JWT, Authentication, Authorization
- [[caching|Caching]] — Rate Limiting детали
- [[api-design|API Design]] — Problem Details для ошибок
