# Authentication и Authorization

## Настройка

AddAuthentication (JWT, Cookie) + AddAuthorization. Middleware: UseAuthentication → UseAuthorization (после Routing). `[Authorize]` на controller/action, `[AllowAnonymous]` для исключений. Глобально: AuthorizeFilter + AllowAnonymous на нужных методах.

---

## Login, JWT, Refresh tokens

**Login**: проверка credentials (BCrypt/Argon2), генерация Access (JWT, короткий) + Refresh (долгоживущий). Refresh — отдельный endpoint, проверка в БД/Redis, выдача новой пары.

**JWT**: header.payload.signature. Payload — claims (sub, exp, roles). Stateless, нельзя отозвать до истечения. Короткий TTL + Refresh — практика.

**Refresh**: хранение в БД или Redis. Ротация — каждый раз новый Refresh, старый инвалидируется. httpOnly cookie для защиты от XSS.

---

## Permissions

Policy-based authorization: AddPolicy с PermissionRequirement. IAuthorizationHandler проверяет claims или БД. Resource-based — проверка доступа к объекту (OwnerId == User.Id).

---

## Current User

`User` (ClaimsPrincipal) в Controller, `HttpContext.User`. Minimal API — параметр `ClaimsPrincipal user`. `User.FindFirstValue(ClaimTypes.NameIdentifier)`, `User.Identity?.Name`.
