# Pipeline, Middleware и Routing

## Pipeline и Middleware

Pipeline — цепочка компонентов (middleware). Каждый получает `HttpContext` и вызывает `next()` или завершает (short-circuit). Запрос проходит «вниз» до endpoint, ответ — «вверх» обратно.

```
Request → M1 → M2 → M3 → Endpoint → M3 → M2 → M1 → Response
```

Порядок регистрации в `Program.cs` = порядок выполнения. Типичный порядок: Exception handling → HTTPS redirect → HSTS → CORS → Static files → Routing → Authentication → Authorization → Endpoint.

**Short-circuit** — не вызывать `next()`. Примеры: health check, статические файлы, rate limiting (429).

### Создание middleware

1. **Inline (lambda)**: `app.Use(async (context, next) => { await next(context); });`
2. **Класс с InvokeAsync**: `public async Task InvokeAsync(HttpContext context)` + `app.UseMiddleware<CustomMiddleware>()`
3. **Extension method**: `app.UseCustom()` → `app.UseMiddleware<CustomMiddleware>()`

---

## Routing

Routing сопоставляет HTTP-запрос с endpoint'ом. Endpoint Routing (с .NET Core 3.0): `UseRouting()` строит таблицу маршрутов, matching по URL и HTTP-методу, dispatch через `UseEndpoints()`.

**Конвенции**: Conventional (`{controller}/{action}/{id?}`), Attribute (`[Route("api/[controller]")]`), Minimal API (`app.MapGet("/users/{id}", ...)`).

**Параметры**: `{id}` — обязательный, `{id?}` — опциональный, `{id:int}` — constraint, `{*path}` — catch-all.

---

## MVC, Razor Pages, Minimal API

| Аспект | MVC | Razor Pages |
|--------|-----|-------------|
| Модель | Controller + View + Model | PageModel (Page + Model в одном) |
| Структура | Отдельные контроллеры | Одна страница = .cshtml + .cshtml.cs |
| Применение | API, SPA backend | Формы, CRUD |

**Minimal API** — легковесные эндпоинты без контроллеров. Быстрее за счёт меньших абстракций (нет ModelBinding, фильтров по умолчанию). Группировка: `MapGroup("/api/users")`, extension methods для изоляции. DI через параметры handler'а или класс-handler.
