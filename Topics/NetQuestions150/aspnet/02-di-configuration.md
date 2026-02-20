# DI и Configuration

## Lifetime сервисов

| Lifetime | Создание | Область |
|----------|----------|---------|
| Transient | Новый при каждом GetService | Один запрос к контейнеру |
| Scoped | Один на scope (HTTP-запрос) | В рамках scope |
| Singleton | Один на приложение | Глобально |

Transient — stateless. Scoped — DbContext, Unit of Work. Singleton — кэши, конфигурация. Singleton не должен зависеть от Scoped напрямую.

**Scoped в Singleton**: `IServiceScopeFactory` — создавать scope в методе, получать Scoped из него. Каждый вызов — новый scope.

**Keyed services** (.NET 8): `AddKeyedSingleton<T>("key", ...)`, `[FromKeyedServices("key")]` — несколько реализаций одного интерфейса, выбор по контексту.

---

## Configuration

**Layering** — последующие источники перезаписывают предыдущие: appsettings.json → appsettings.{Environment}.json → User secrets → Environment variables → Command-line args.

Переменные окружения: `Logging__LogLevel__Default` (двойное подчёркивание = вложенность).

**Чтение**: IConfiguration (`configuration["Key"]`), IOptions<T>, Bind, Get<T>(). Connection string: `GetConnectionString("DefaultConnection")`.
