# Authentication и Authorization

## Основные концепции

**Authentication** (аутентификация) — КТО вы? Проверка identity пользователя.
**Authorization** (авторизация) — ЧТО вам разрешено? Проверка прав доступа.

### Настройка в Pipeline

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
            ClockSkew = TimeSpan.Zero // По умолчанию 5 минут допуска!
        };
    });

builder.Services.AddAuthorization();

var app = builder.Build();

app.UseAuthentication();  // Заполняет HttpContext.User
app.UseAuthorization();   // Проверяет права доступа
// ПОРЯДОК КРИТИЧЕН: Authentication ПЕРЕД Authorization, оба ПОСЛЕ Routing
```

### Применение атрибутов

```csharp
[Authorize]                              // Требует аутентификации
[Authorize(Roles = "Admin")]             // Требует роль Admin
[Authorize(Policy = "MinAge18")]         // Требует соответствие политике
[AllowAnonymous]                         // Разрешает анонимный доступ

// Глобальная авторизация — всё закрыто, открываем явно
builder.Services.AddAuthorization(opts =>
{
    opts.FallbackPolicy = new AuthorizationPolicyBuilder()
        .RequireAuthenticatedUser()
        .Build();
});
// Теперь [AllowAnonymous] нужен для открытых endpoint'ов
```

---

## JWT (JSON Web Token)

Структура: `header.payload.signature` (Base64-encoded).

```
Header:  { "alg": "HS256", "typ": "JWT" }
Payload: { "sub": "user123", "name": "John", "roles": ["Admin"], "exp": 1700000000 }
Signature: HMACSHA256(base64(header) + "." + base64(payload), secret)
```

### Генерация JWT

```csharp
public class TokenService
{
    private readonly JwtOptions _options;

    public TokenService(IOptions<JwtOptions> options) => _options = options.Value;

    public string GenerateAccessToken(User user)
    {
        var claims = new List<Claim>
        {
            new(JwtRegisteredClaimNames.Sub, user.Id.ToString()),
            new(JwtRegisteredClaimNames.Email, user.Email),
            new(JwtRegisteredClaimNames.Jti, Guid.NewGuid().ToString()), // Уникальный ID токена
        };

        // Добавляем роли как отдельные claims
        foreach (var role in user.Roles)
            claims.Add(new Claim(ClaimTypes.Role, role));

        var key = new SymmetricSecurityKey(Encoding.UTF8.GetBytes(_options.Key));
        var creds = new SigningCredentials(key, SecurityAlgorithms.HmacSha256);

        var token = new JwtSecurityToken(
            issuer: _options.Issuer,
            audience: _options.Audience,
            claims: claims,
            expires: DateTime.UtcNow.AddMinutes(15), // Короткий TTL!
            signingCredentials: creds);

        return new JwtSecurityTokenHandler().WriteToken(token);
    }
}
```

### Тонкости JWT

- JWT **stateless** — нельзя отозвать до истечения `exp`. Поэтому TTL должен быть коротким (5-15 минут)
- `ClockSkew` по умолчанию **5 минут** — токен продолжает работать 5 минут после `exp`. Установите `TimeSpan.Zero` для строгой проверки
- Не храните чувствительные данные в payload — он легко декодируется (Base64, не шифрование)
- Для отзыва токенов до истечения — blacklist в Redis по `jti` claim
- Размер JWT растёт с количеством claims — при большом количестве ролей/разрешений лучше хранить только `sub` и запрашивать права из БД

---

## Login Flow и Refresh Tokens

### Полный Flow

```
1. POST /auth/login { email, password }
2. Сервер проверяет credentials (BCrypt.Verify / Argon2)
3. Генерирует Access Token (JWT, 15 мин) + Refresh Token (opaque, 7-30 дней)
4. Refresh Token сохраняется в БД/Redis (hash + userId + expiry + isRevoked)
5. Access Token → в response body, Refresh Token → в httpOnly cookie

6. Клиент отправляет Access Token в заголовке: Authorization: Bearer <token>
7. При 401 — клиент отправляет POST /auth/refresh с Refresh Token из cookie
8. Сервер валидирует Refresh Token, генерирует новую пару (Token Rotation)
9. Старый Refresh Token инвалидируется
```

### Refresh Token — реализация

```csharp
public class RefreshToken
{
    public string Token { get; set; } = null!;    // Криптографически случайный
    public string UserId { get; set; } = null!;
    public DateTime ExpiresAt { get; set; }
    public bool IsRevoked { get; set; }
    public string? ReplacedByToken { get; set; }   // Для обнаружения кражи
    public DateTime CreatedAt { get; set; }
}

// Генерация
public static string GenerateRefreshToken()
{
    var bytes = RandomNumberGenerator.GetBytes(64);
    return Convert.ToBase64String(bytes);
}
```

### Token Rotation и обнаружение кражи

При каждом refresh старый токен инвалидируется и заменяется новым. Если кто-то использует уже инвалидированный токен — это признак кражи. В этом случае **инвалидируются все** токены пользователя (revoke family).

---

## Policy-based Authorization

```csharp
builder.Services.AddAuthorization(opts =>
{
    // Простая policy через claims
    opts.AddPolicy("AdminOnly", policy => policy.RequireRole("Admin"));

    opts.AddPolicy("MinAge18", policy =>
        policy.RequireAssertion(ctx =>
        {
            var birthDate = ctx.User.FindFirstValue("birth_date");
            return birthDate != null && DateTime.Parse(birthDate).AddYears(18) <= DateTime.Today;
        }));

    // Policy с requirement + handler
    opts.AddPolicy("CanEditOrder", policy =>
        policy.Requirements.Add(new SameAuthorRequirement()));
});

// Requirement
public class SameAuthorRequirement : IAuthorizationRequirement { }

// Handler
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
        // Не вызываем Fail() — другой handler может Succeed
        return Task.CompletedTask;
    }
}

builder.Services.AddSingleton<IAuthorizationHandler, SameAuthorHandler>();
```

### Resource-based Authorization

Проверка доступа к конкретному объекту (не просто роль, а «это МОЙ заказ»):

```csharp
public class OrdersController : ControllerBase
{
    private readonly IAuthorizationService _authService;

    [HttpPut("{id}")]
    public async Task<IActionResult> Update(int id, UpdateOrderDto dto)
    {
        var order = await _repo.GetByIdAsync(id);
        var authResult = await _authService.AuthorizeAsync(User, order, "CanEditOrder");
        if (!authResult.Succeeded) return Forbid();
        // ... обновление
    }
}
```

---

## Получение текущего пользователя

```csharp
// В Controller
var userId = User.FindFirstValue(ClaimTypes.NameIdentifier);
var email = User.FindFirstValue(ClaimTypes.Email);
var roles = User.FindAll(ClaimTypes.Role).Select(c => c.Value);
var isAdmin = User.IsInRole("Admin");

// В Minimal API
app.MapGet("/me", (ClaimsPrincipal user) =>
{
    var userId = user.FindFirstValue(ClaimTypes.NameIdentifier);
    return Results.Ok(new { userId });
});

// В любом сервисе — через IHttpContextAccessor
builder.Services.AddHttpContextAccessor();

public class CurrentUserService
{
    private readonly IHttpContextAccessor _accessor;
    public CurrentUserService(IHttpContextAccessor accessor) => _accessor = accessor;
    public string? UserId => _accessor.HttpContext?.User.FindFirstValue(ClaimTypes.NameIdentifier);
}
```

### Тонкости Auth

- `IHttpContextAccessor` — **не** потокобезопасен, не используйте в background-задачах. Сохраняйте нужные данные (userId) до выхода из scope запроса
- Cookie authentication: `httpOnly` + `Secure` + `SameSite=Strict` — защита от XSS и CSRF
- Для multi-tenant приложений — кастомный `IAuthorizationHandler`, проверяющий `TenantId`
- `[Authorize]` на Minimal API: `.RequireAuthorization()` на endpoint
- Claims transformation: `IClaimsTransformation` для обогащения claims (например, добавить permissions из БД при каждом запросе)
