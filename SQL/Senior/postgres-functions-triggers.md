---
tags: [postgresql, plpgsql, stored-procedures, functions, triggers, npgsql, raw-sql]
level: Senior
date: 2026-08-02
---

# PostgreSQL: Функции, процедуры, триггеры и raw SQL из .NET

> PL/pgSQL функции vs процедуры, volatility-категории, триггеры (BEFORE/AFTER, аудит, `updated_at`), вызов raw SQL из Npgsql/EF Core с ловлей доменных ошибок по ERRCODE — и senior-граница «целостность данных в БД, бизнес-логика в приложении».

## Кратко
Хранимые функции/процедуры на PL/pgSQL, триггеры и вызов raw SQL из Npgsql/EF Core — то, что Postgres-вакансии любят спрашивать отдельно. Главный senior-вопрос здесь не «как написать триггер», а «когда логика уместна в БД, а когда это анти-паттерн».

## Функция vs процедура — не одно и то же

В Postgres это **два разных объекта** (с PG 11 появились настоящие процедуры).

| | `FUNCTION` | `PROCEDURE` |
|--|------------|-------------|
| Вызов | `SELECT f(...)` / внутри запроса | только `CALL p(...)` |
| Возвращает | значение / таблицу / `void` | ничего (только OUT/INOUT-параметры) |
| Транзакции | работает **внутри** транзакции вызывающего, не может `COMMIT` | может `COMMIT` / `ROLLBACK` внутри себя |
| Где использовать | вычисления, выборки, в `SELECT`/`WHERE` | батч-операции, миграции данных, ETL |

**Правило:** нужно вернуть данные или вызвать из запроса → функция. Нужно управлять транзакциями внутри (например, обработать миллион строк пачками с промежуточными коммитами) → процедура.

## Функции

### Простая SQL-функция (без PL/pgSQL)
`LANGUAGE sql` — быстрее, инлайнится планировщиком, нет процедурной логики.

```sql
CREATE OR REPLACE FUNCTION full_name(p_first text, p_last text)
RETURNS text
LANGUAGE sql
IMMUTABLE
AS $$
    SELECT p_first || ' ' || p_last;
$$;
```

### Volatility — критично для производительности
Категория говорит планировщику, можно ли кэшировать/инлайнить результат.

| Категория | Смысл | Пример |
|-----------|-------|--------|
| `IMMUTABLE` | один вход → всегда один выход, без обращения к БД | `lower()`, арифметика |
| `STABLE` | в рамках одного запроса стабильна, читает БД | `now()`, выборка по справочнику |
| `VOLATILE` (default) | может меняться при каждом вызове | `random()`, `INSERT ... RETURNING` |

Неправильная категория = либо неверные результаты, либо потеря оптимизаций (например, `IMMUTABLE`-функцию можно использовать в expression index, `VOLATILE` — нельзя).

### RETURNS TABLE / SETOF — табличный результат
```sql
CREATE OR REPLACE FUNCTION active_users_since(p_since timestamptz)
RETURNS TABLE (id bigint, email text, last_login timestamptz)
LANGUAGE plpgsql
STABLE
AS $$
BEGIN
    RETURN QUERY
    SELECT u.id, u.email, u.last_login
    FROM users u
    WHERE u.last_login >= p_since
    ORDER BY u.last_login DESC;
END;
$$;

-- вызов как обычная таблица:
SELECT * FROM active_users_since(now() - interval '7 days');
```

### PL/pgSQL — структура блока
```sql
CREATE OR REPLACE FUNCTION transfer_credits(p_from bigint, p_to bigint, p_amount numeric)
RETURNS void
LANGUAGE plpgsql
AS $$
DECLARE
    v_balance numeric;
BEGIN
    -- блокируем строку отправителя
    SELECT balance INTO v_balance
    FROM accounts WHERE id = p_from
    FOR UPDATE;

    IF v_balance IS NULL THEN
        RAISE EXCEPTION 'Account % not found', p_from
            USING ERRCODE = 'no_data_found';
    END IF;

    IF v_balance < p_amount THEN
        RAISE EXCEPTION 'Insufficient funds: have %, need %', v_balance, p_amount
            USING ERRCODE = 'check_violation';
    END IF;

    UPDATE accounts SET balance = balance - p_amount WHERE id = p_from;
    UPDATE accounts SET balance = balance + p_amount WHERE id = p_to;
END;
$$;
```

Ключевое: `DECLARE` → `BEGIN ... END`, `SELECT ... INTO`, `RAISE EXCEPTION ... USING ERRCODE`, `FOR UPDATE` для блокировки. `ERRCODE` важен — по нему ловим конкретную ошибку в C# (см. ниже).

## Процедуры с управлением транзакциями
```sql
CREATE OR REPLACE PROCEDURE purge_old_logs(p_batch int DEFAULT 10000)
LANGUAGE plpgsql
AS $$
DECLARE
    v_deleted int;
BEGIN
    LOOP
        DELETE FROM logs
        WHERE id IN (
            SELECT id FROM logs
            WHERE created_at < now() - interval '90 days'
            LIMIT p_batch
        );
        GET DIAGNOSTICS v_deleted = ROW_COUNT;
        EXIT WHEN v_deleted = 0;
        COMMIT;  -- ← в функции так нельзя, в процедуре можно
    END LOOP;
END;
$$;

CALL purge_old_logs(5000);
```
Батчевое удаление с промежуточными коммитами — чтобы не держать гигантскую транзакцию и не раздувать WAL. Классический пример «зачем вообще процедура, а не функция».

## Триггеры

Триггер = **trigger function** (возвращает `trigger`) + объект `CREATE TRIGGER`, который её привязывает.

### Матрица момента и уровня
| | `FOR EACH ROW` | `FOR EACH STATEMENT` |
|--|----------------|----------------------|
| `BEFORE` | валидация/модификация `NEW` до записи | проверка перед всей операцией |
| `AFTER` | аудит, каскады, события (`NEW`/`OLD` доступны) | агрегаты после всей операции |
| `INSTEAD OF` | только на VIEW (делает view writable) | — |

В row-level триггере доступны `NEW` (новая строка, для INSERT/UPDATE) и `OLD` (старая, для UPDATE/DELETE). `TG_OP` = операция (`'INSERT'`/`'UPDATE'`/`'DELETE'`).

### Авто-обновление updated_at (рабочая лошадка)
```sql
CREATE OR REPLACE FUNCTION set_updated_at()
RETURNS trigger
LANGUAGE plpgsql
AS $$
BEGIN
    NEW.updated_at := now();
    RETURN NEW;   -- BEFORE-триггер обязан вернуть NEW, иначе строка не запишется
END;
$$;

CREATE TRIGGER trg_users_updated_at
    BEFORE UPDATE ON users
    FOR EACH ROW
    EXECUTE FUNCTION set_updated_at();
```

### Аудит-триггер (частый пример на собесе)
```sql
CREATE OR REPLACE FUNCTION audit_changes()
RETURNS trigger
LANGUAGE plpgsql
AS $$
BEGIN
    INSERT INTO audit_log(table_name, op, row_id, old_data, new_data, changed_at)
    VALUES (
        TG_TABLE_NAME,
        TG_OP,
        COALESCE(NEW.id, OLD.id),
        CASE WHEN TG_OP <> 'INSERT' THEN to_jsonb(OLD) END,
        CASE WHEN TG_OP <> 'DELETE' THEN to_jsonb(NEW) END,
        now()
    );
    RETURN COALESCE(NEW, OLD);  -- AFTER: возврат игнорируется, но пишем для единообразия
END;
$$;

CREATE TRIGGER trg_orders_audit
    AFTER INSERT OR UPDATE OR DELETE ON orders
    FOR EACH ROW
    EXECUTE FUNCTION audit_changes();
```

### Условный триггер через WHEN
```sql
CREATE TRIGGER trg_notify_status
    AFTER UPDATE ON orders
    FOR EACH ROW
    WHEN (OLD.status IS DISTINCT FROM NEW.status)  -- срабатывает только при смене статуса
    EXECUTE FUNCTION notify_status_change();
```
`IS DISTINCT FROM` вместо `<>` — корректно работает с NULL.

## Вызов из .NET

### Npgsql — функция, возвращающая таблицу
```csharp
await using var conn = new NpgsqlConnection(connString);
await conn.OpenAsync(ct).ConfigureAwait(false);

await using var cmd = new NpgsqlCommand(
    "SELECT id, email, last_login FROM active_users_since(@since)", conn);
cmd.Parameters.AddWithValue("since", DateTimeOffset.UtcNow.AddDays(-7));

await using var reader = await cmd.ExecuteReaderAsync(ct).ConfigureAwait(false);
var users = new List<ActiveUserDto>();
while (await reader.ReadAsync(ct).ConfigureAwait(false))
{
    users.Add(new ActiveUserDto
    {
        Id = reader.GetInt64(0),
        Email = reader.GetString(1),
        LastLogin = reader.GetFieldValue<DateTimeOffset>(2),
    });
}
```

> [!warning] `AddWithValue` здесь и ниже — упрощение
> `AddWithValue` выводит `NpgsqlDbType` из CLR-типа значения, и для неоднозначных типов (`timestamptz` vs `timestamp`, `jsonb` vs `text`, `numeric` precision/scale) это даёт неверный маппинг и implicit conversion, который ломает индексы. В проде задавай тип явно: `cmd.Parameters.Add(new NpgsqlParameter("since", NpgsqlDbType.TimestampTz) { Value = ... })`. Разбор — [[security-practices]].

### Процедура через CALL
```csharp
await using var cmd = new NpgsqlCommand("CALL purge_old_logs(@batch)", conn);
cmd.Parameters.AddWithValue("batch", 5000);
await cmd.ExecuteNonQueryAsync(ct).ConfigureAwait(false);
```
Процедуру нельзя вызвать через `CommandType.StoredProcedure` в Npgsql — это поведение в Npgsql 6+ маппится на `CALL`, но для функций используется `SELECT`. Надёжнее писать текст команды явно.

### Ловля доменной ошибки по ERRCODE
```csharp
try
{
    await using var cmd = new NpgsqlCommand("SELECT transfer_credits(@from, @to, @amt)", conn);
    cmd.Parameters.AddWithValue("from", fromId);
    cmd.Parameters.AddWithValue("to", toId);
    cmd.Parameters.AddWithValue("amt", amount);
    await cmd.ExecuteNonQueryAsync(ct).ConfigureAwait(false);
}
catch (PostgresException ex) when (ex.SqlState == PostgresErrorCodes.CheckViolation)
{
    // тот самый RAISE EXCEPTION ... USING ERRCODE = 'check_violation'
    throw new InsufficientFundsException(fromId);
}
```
`PostgresException.SqlState` = SQLSTATE из `ERRCODE`. Так доменные ошибки из БД превращаются в доменные исключения приложения.

### EF Core
```csharp
// raw-запрос с маппингом на entity/keyless-тип
var users = await db.Users
    .FromSql($"SELECT * FROM active_users_since({since})")
    .AsNoTracking()
    .ToListAsync(ct)
    .ConfigureAwait(false);

// процедура/команда без результата
await db.Database
    .ExecuteSqlAsync($"CALL purge_old_logs({batch})", ct)
    .ConfigureAwait(false);
```
`FromSql` (интерполированный, параметризованный) безопаснее `FromSqlRaw`. Скалярную функцию можно замапить через `modelBuilder.HasDbFunction(...)` и вызывать прямо в LINQ.

## Хранимки vs логика в приложении — senior trade-offs

Это и есть вопрос «на подумать». Сбалансированная позиция:

**Логика уместна в БД, когда:**
- Целостность, которую нельзя обойти ни одним путём записи (несколько сервисов/скриптов пишут в одну таблицу) → constraint/trigger надёжнее, чем дисциплина в коде.
- Аудит и `updated_at` — кросс-таблично, тривиально, не бизнес-правило.
- Операция data-heavy: дешевле обработать рядом с данными, чем тащить миллион строк в приложение (батч-очистка, агрегации).

**Логика должна быть в приложении, когда:**
- Это бизнес-правило, которое надо версионировать, тестировать и ревьюить как код. PL/pgSQL слабо покрывается unit-тестами, прячется от code review, размывает доменную модель.
- Нужна переносимость между СУБД.
- В команде DDD/Clean Architecture: бизнес-логика в триггерах — это «logic in two places», тяжело отлаживать каскады (триггер дёргает триггер).

**Формулировка для собеса:** «Триггеры и функции — отличный инструмент для инвариантов данных и аудита, то есть для того, что должно держаться независимо от того, какой код пишет в таблицу. Но я держу бизнес-правила в домене: их надо тестировать и ревьюить. Граница — "целостность данных в БД, бизнес-решения в приложении".» Это не противоречит культуре, где любят хранимки, и показывает суждение.

## Типичные вопросы на собесе

- **Функция или процедура для X?** → нужно вернуть данные/вызвать в запросе = функция; нужны коммиты внутри = процедура.
- **Зачем IMMUTABLE/STABLE/VOLATILE?** → планировщик кэширует/инлайнит; `IMMUTABLE` нужна для expression index.
- **BEFORE vs AFTER триггер?** → BEFORE может менять `NEW` и отменять запись (`RETURN NULL`); AFTER для аудита/каскадов, `NEW` уже зафиксирован.
- **Почему BEFORE-триггер должен возвращать NEW?** → возврат `NULL` отменяет операцию для этой строки; возврат `NEW` пропускает (с изменениями).
- **Как из C# отличить «нет средств» от «нет аккаунта»?** → `RAISE ... USING ERRCODE`, ловить `PostgresException.SqlState`.
- **Чем опасны триггеры?** → скрытая логика, каскады триггер→триггер, просадка на bulk-операциях (row-level срабатывает на каждую строку), сложность отладки.
- **`<>` или `IS DISTINCT FROM` в `WHEN`?** → `IS DISTINCT FROM`, иначе NULL ломает сравнение.

## Pitfalls
1. **Row-level триггер на bulk-INSERT** — срабатывает на каждую строку, INSERT на 1M строк превращается в 1M вызовов функции. Для массовых операций — statement-level или отключение триггера на время загрузки.
2. **Бизнес-логика в триггере** — невидима в code review, не покрыта тестами домена, каскады отлаживаются часами.
3. **VOLATILE по умолчанию** — забытая категория мешает оптимизатору и блокирует expression index.
4. **`SELECT ... INTO` без проверки `NOT FOUND`** — переменная молча остаётся NULL.
5. **Рекурсивные/циклические триггеры** — UPDATE в триггере на ту же таблицу → повторный вызов. Защита через `WHEN` или `pg_trigger_depth()`.
6. **Утечка транзакции при `CALL`** — процедура коммитит внутри; если приложение тоже открыло транзакцию, поведение неожиданное. Решать осознанно.

## См. также
- [[postgresql-deep]] — JSONB, RLS, партиционирование, EXPLAIN
- [[optimization]] — планы запросов, статистика
- [[indexes-deep]] — expression index (где нужен IMMUTABLE)
- [[dapper-comparison]] — raw SQL через Dapper

## Reading list
- PostgreSQL docs: PL/pgSQL, CREATE FUNCTION, CREATE TRIGGER
- Npgsql docs: Stored functions/procedures, error handling
- EF Core docs: Raw SQL queries, user-defined function mapping

---
*Добавлено: 2026-05-29 — закрывает gap по вакансии (хранимки/триггеры/raw SQL)*
