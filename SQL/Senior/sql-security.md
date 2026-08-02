---
tags: [postgresql, security, sql-injection, roles, privileges, grant, least-privilege, rls, ef-core, npgsql]
level: Senior
date: 2026-06-28
---

# SQL Security: инъекции, роли и привилегии

## Кратко
Две темы, которые на собесе спрашивают вместе: как реально защититься от SQL-инъекций (параметризация, а не экранирование; second-order; идентификаторы нельзя параметризовать) и как устроить наименьшие привилегии в Postgres (роли, `GRANT`/`REVOKE`, отдельный owner для DDL, app-роль только на DML, `SECURITY DEFINER` пропасть). Плюс TLS, секреты и аудит.

---

# Часть 1. SQL Injection — глубоко

## Единственная настоящая защита — параметризация
Инъекция возможна, когда **данные** попадают в **текст** запроса через конкатенацию/интерполяцию. Параметризованный запрос отправляет текст и значения **раздельно** (extended/prepared protocol): сервер парсит запрос с плейсхолдерами, потом подставляет значения как данные — они физически не могут «стать кодом».

**Ручное экранирование — НЕ защита.** Блэклисты («вырезать `'` и `;`»), `string.Replace`, «проверка на DROP» обходятся (кодировки, юникод, комментарии, разные синтаксисы). Правильно — **всегда** параметры.

```csharp
// ОПАСНО — конкатенация
var sql = $"SELECT * FROM users WHERE email = '{email}'";   // email = ' OR 1=1 --

// БЕЗОПАСНО — параметр
await using var cmd = new NpgsqlCommand("SELECT * FROM users WHERE email = @e", conn);
cmd.Parameters.AddWithValue("e", email);
```

> [!warning] `AddWithValue` — анти-паттерн по типу параметра
> От инъекций защищает любой параметр, но `AddWithValue` **выводит тип из значения** (`string` → Unicode `nvarchar`/`text`). Если колонка другого типа, СУБД делает implicit conversion на каждую строку → **index scan вместо seek**. Предпочитай явный тип: `cmd.Parameters.Add("e", NpgsqlDbType.Varchar).Value = email`. Разбор и SQL Server-вариант — [[security-practices]].

## Виды инъекций
- **Classic / in-band** — результат виден прямо в ответе (`' OR 1=1 --`, `UNION SELECT ...`).
- **Blind (boolean / time-based)** — ответ не видно, но атакующий выводит данные по поведению: `... AND (SELECT ...) = 'x'` (страница меняется) или `... AND pg_sleep(5)` (отвечает дольше).
- **Second-order (stored) ⭐** — самый коварный. Вредоносная строка **сохраняется безопасно** (через параметр), а потом **читается из БД и конкатенируется** в другой запрос. Урок: параметризуй даже данные, пришедшие из собственной БД, а не только пользовательский ввод «с фронта».
- **Через идентификатор** — имя таблицы/колонки/`ORDER BY` **нельзя** параметризовать (см. ниже). Динамическая сортировка `ORDER BY {userInput}` — классическая дыра.

## Идентификаторы нельзя параметризовать
Плейсхолдер `@p` работает только для **значений**. Имя колонки/таблицы/направление сортировки параметром не передать. Для динамики — **whitelist в коде**:

```csharp
// динамический ORDER BY — только из разрешённого набора
static readonly FrozenDictionary<string, string> SortColumns =
    new Dictionary<string, string>
    {
        ["date"] = "created_at",
        ["name"] = "full_name",
    }.ToFrozenDictionary();

var column = SortColumns.GetValueOrDefault(sortKey, "created_at"); // не из ввода напрямую
var dir = ascending ? "ASC" : "DESC";
var sql = $"SELECT * FROM orders ORDER BY {column} {dir}";  // безопасно: значения из whitelist
```
Альтернатива в SQL — `quote_ident()` / `format('%I', ident)`, но whitelist надёжнее (ограничивает набор, а не только экранирует).

## EF Core: что безопасно, что нет
- **LINQ** — всегда параметризуется. Безопасно по определению.
- **`FromSql($"...")` / `ExecuteSql($"...")`** с C#-интерполяцией — EF **превращает интерполированные значения в параметры**. Безопасно.
- **`FromSqlRaw` / `ExecuteSqlRaw`** — безопасны **только** если сам передал `NpgsqlParameter`. Конкатенация/`string.Format` внутрь raw → дыра.

```csharp
// безопасно — интерполяция уходит в параметры
var users = await db.Users.FromSql($"SELECT * FROM users WHERE email = {email}").ToListAsync(ct);

// опасно — строковая сборка
var users = await db.Users.FromSqlRaw("SELECT * FROM users WHERE email = '" + email + "'").ToListAsync(ct);
```

## Динамический SQL в PL/pgSQL
Если в функции собираешь запрос строкой — `format()` с правильными спецификаторами + `USING` для значений:
```sql
EXECUTE format('SELECT * FROM %I WHERE status = $1', p_table)  -- %I — безопасный identifier
USING p_status;                                                -- значение через USING, не в строку
```
`%I` квотирует **идентификатор**, `%L` — **литерал**. Значения лучше через `USING`, а не через `%L`.

## Хранимки сами по себе не защищают
Распространённый миф «перешли всё на stored procedures → инъекций нет». Если **внутри** процедуры динамический `EXECUTE` с конкатенацией — дыра ровно та же. Защищает параметризация (в т.ч. `USING`/`%I`), а не факт нахождения кода в БД.

## LIKE и спецсимволы
В `LIKE` символы `%` и `_` — wildcards. Пользовательский ввод в `LIKE` без экранирования = логическая дыра (не RCE, но обход фильтра/DoS):
```sql
WHERE name LIKE @pattern ESCAPE '\'   -- и экранировать % _ \ во вводе перед подстановкой в pattern
```

---

# Часть 2. Роли и привилегии (least privilege)

## Роль = и пользователь, и группа
В Postgres нет отдельных «users» и «groups» — есть **роли**. Роль с `LOGIN` ведёт себя как пользователь; роль без `LOGIN` (`NOLOGIN`) — как группа, в которую включают других (`GRANT group TO user`), и они наследуют её права.

## GRANT / REVOKE
```sql
GRANT SELECT, INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA app TO app_rw;
GRANT SELECT ON ALL TABLES IN SCHEMA app TO app_ro;
REVOKE DELETE ON sensitive_table FROM app_rw;
GRANT USAGE ON SCHEMA app TO app_rw, app_ro;   -- без USAGE на схему доступа к таблицам нет
```
Привилегии гранятся на объекты: таблицы, колонки, схемы, последовательности, функции. Бывает **column-level**: `GRANT SELECT (id, email) ON users TO app_ro` (остальные колонки невидимы).

## Принцип наименьших привилегий — схема ролей для приложения
**Приложение не должно ходить под суперюзером или владельцем таблиц.** Минимальный набор:

| Роль | Права | Кто использует |
|------|-------|----------------|
| `app_owner` | владелец объектов, DDL | **только миграции** (отдельный connection string) |
| `app_rw` | DML (`SELECT/INSERT/UPDATE/DELETE`), без DDL | runtime приложения |
| `app_ro` | `SELECT` | аналитика, дашборды, читающая реплика |

Так украденные runtime-креды не дают `DROP TABLE`, а аналитический доступ не может писать.

```csharp
// разные роли в разных connection strings
// миграции:  Username=app_owner;...
// runtime:   Username=app_rw;...
```

## Default privileges — чтобы будущие таблицы грантились сами
`GRANT ... ON ALL TABLES` раздаёт права только на **существующие** объекты. Для будущих:
```sql
ALTER DEFAULT PRIVILEGES IN SCHEMA app
    GRANT SELECT, INSERT, UPDATE, DELETE ON TABLES TO app_rw;
ALTER DEFAULT PRIVILEGES IN SCHEMA app
    GRANT USAGE ON SEQUENCES TO app_rw;
```
Иначе каждая новая таблица из миграции будет недоступна app-роли, пока вручную не грантнешь.

## PUBLIC schema (важное изменение PG 15)
До PG 15 псевдо-роль `PUBLIC` (= все роли) имела `CREATE` на схему `public` — любой мог создавать там объекты. **PG 15 это убрал** (`CREATE` на `public` больше не у `PUBLIC`). Если мигрируешь со старого кластера — учитывай, что часть «само работало» перестанет; и наоборот, не полагайся на открытый `public`.

## SECURITY DEFINER — поверхность эскалации
Функция с `SECURITY DEFINER` выполняется с правами **владельца**, а не вызывающего (аналог setuid). Полезно дать app-роли контролируемую операцию выше её прав. Но это **дыра эскалации**, если:
- не зафиксирован `search_path` → атакующий создаёт свой объект (функцию/таблицу) в схеме, которая ищется раньше, и подменяет вызов внутри функции.

Защита — всегда фиксировать `search_path` на такой функции:
```sql
CREATE FUNCTION app.do_privileged(...) RETURNS void
LANGUAGE plpgsql
SECURITY DEFINER
SET search_path = pg_catalog, app   -- ← фикс, чтобы нельзя было подменить объекты
AS $$ ... $$;
```
`SECURITY INVOKER` (default) безопаснее — выполняется с правами вызывающего.

## Row-Level Security
Часть модели безопасности — фильтрация **строк** по политике (multi-tenant, «вижу только свои»). Подробно — в [[postgresql-deep]] (раздел RLS). Связка: app-роль + RLS-политика на `tenant_id` = арендатор не достанет чужие строки даже при ошибке в коде.

---

# Часть 3. Транспорт, секреты, аудит

## TLS
```text
sslmode=require       # шифрование канала
sslmode=verify-full   # + проверка сертификата И hostname (защита от MITM) — для прода
```
`require` шифрует, но не проверяет, с тем ли сервером говоришь; `verify-full` — проверяет. Для внешних соединений — `verify-full`.

## Секреты — не в коде
Connection string с паролем **не хардкодить и не коммитить**. Локально — `dotnet user-secrets` (твой привычный паттерн — `user-secrets set` в TradingBotForex), в проде — переменные окружения / секрет-менеджер. В логи строку подключения не писать.

## Аудит
- **pgAudit** — расширение для детального аудита (какая роль что выполнила).
- Лёгкий вариант — аудит-триггеры (см. [[postgres-functions-triggers]]), но это не security-grade против привилегированного пользователя.
- **Не логировать параметры с PII** — параметризация защищает от инъекций, но если включить логирование значений, чувствительные данные утекут в логи. Отдельно настраивать, что писать.

---

## Типичные вопросы на собесе
- **Как защититься от SQL-инъекции?** → Параметризация (раздельная передача текста и значений), не экранирование/блэклисты. Идентификаторы — whitelist.
- **Можно ли параметризовать имя таблицы или `ORDER BY`?** → Нет, только значения. Имена — whitelist или `quote_ident`/`%I`.
- **Что такое second-order injection?** → Вредонос сохранён безопасно, но позже прочитан и сконкатенирован в другой запрос. Лечится параметризацией всегда, даже данных из своей БД.
- **`FromSql` vs `FromSqlRaw` в EF Core?** → Интерполированный `FromSql` уходит в параметры (безопасно); `FromSqlRaw` безопасен только с явными параметрами, не с конкатенацией.
- **Защищают ли хранимки от инъекций?** → Сами по себе нет — динамический `EXECUTE` с конкатенацией внутри так же уязвим.
- **Как настроить least privilege для приложения?** → Отдельный owner для DDL/миграций, app-роль только на DML, readonly-роль на SELECT; `ALTER DEFAULT PRIVILEGES` для будущих таблиц.
- **Чем опасен `SECURITY DEFINER`?** → Эскалация прав; обязательно фиксировать `search_path`, иначе объекты можно подменить.
- **`sslmode=require` vs `verify-full`?** → require шифрует канал, verify-full ещё и проверяет сертификат/hostname (анти-MITM).
- **Что изменилось с `public` схемой в PG 15?** → У `PUBLIC` забрали `CREATE` на `public` — больше нельзя создавать там объекты по умолчанию.

## Pitfalls
1. **Ручное экранирование вместо параметров** — обходится; ложное чувство безопасности.
2. **`ORDER BY {userInput}`** — идентификатор нельзя параметризовать; без whitelist это дыра.
3. **Параметризовал ввод, но не данные из БД** — second-order injection.
4. **Приложение под суперюзером/владельцем** — украденные креды = `DROP TABLE`.
5. **Забытый `ALTER DEFAULT PRIVILEGES`** — новые таблицы из миграций недоступны app-роли.
6. **`SECURITY DEFINER` без `SET search_path`** — эскалация привилегий.
7. **`sslmode=require` для внешних соединений** — нет защиты от MITM, нужен `verify-full`.
8. **Логирование значений параметров** — PII утекает в логи несмотря на «защищённый» SQL.

## См. также
- [[postgresql-deep]] — Row-Level Security (multi-tenant), JSONB
- [[postgres-functions-triggers]] — `SECURITY DEFINER`, аудит-триггеры, динамический SQL
- [[sql-basics]] — параметризованные запросы (введение)
- [[mvcc-and-locking]] — конкурентность (отдельная тема)

## Reading list
- PostgreSQL docs: Database Roles, Privileges (`GRANT`), `ALTER DEFAULT PRIVILEGES`, `CREATE FUNCTION` (SECURITY DEFINER), SSL Support
- OWASP: SQL Injection Prevention Cheat Sheet
- EF Core docs: Raw SQL queries (parameterization), Npgsql security/SSL

---
*Добавлено: 2026-05-29 — расширение SQL-раздела (инъекции, роли/привилегии, TLS/секреты/аудит)*
