---
tags: [postgresql, mvcc, locking, concurrency, deadlock, skip-locked, upsert, on-conflict, listen-notify, npgsql]
level: Senior
date: 2026-06-28
---

# MVCC, блокировки и конкурентность в PostgreSQL

> MVCC (читатели не блокируют писателей), табличные и строчные lock modes, `SKIP LOCKED` для очередей без брокера, профилактика deadlock, идемпотентные вставки через `ON CONFLICT` и `LISTEN`/`NOTIFY` — с вызовом из Npgsql.

## Кратко
Как Postgres даёт читателям не блокировать писателей (MVCC), что реально происходит при `UPDATE`/`DELETE` на уровне строк, как устроены блокировки (от lock modes до `SKIP LOCKED` для очередей), как не словить deadlock, как делать идемпотентные вставки (`ON CONFLICT`) и слать события (`LISTEN`/`NOTIFY`). Это фундамент, который связывает изоляцию транзакций, VACUUM и производительность под нагрузкой в одну картину.

---

# Часть 1. MVCC изнутри

## Что такое MVCC и зачем
**MVCC = Multi-Version Concurrency Control.** Вместо того чтобы блокировать строку на чтение, Postgres хранит **несколько версий** одной логической строки. Каждая транзакция видит ту версию, которая была актуальна на момент её снимка (snapshot).

**Главное следствие:** читатели не блокируют писателей, писатели не блокируют читателей. `SELECT` никогда не ждёт `UPDATE` той же строки — он просто видит старую версию. Это то, чем Postgres/Oracle отличаются от СУБД с блокировочным чтением.

**Аналогия:** Git. Каждое изменение строки — новый commit версии. Транзакция работает на «своём» снимке истории и не видит чужих незакоммиченных коммитов. VACUUM — это `git gc`, который вычищает версии, которые уже никому не видны.

## Системные колонки: xmin / xmax
У каждой физической версии строки (tuple) есть скрытые системные колонки:

| Колонка | Смысл |
|---------|-------|
| `xmin` | XID (transaction id) транзакции, которая **создала** эту версию |
| `xmax` | XID транзакции, которая эту версию **удалила/заменила** (0 — версия живая) |
| `ctid` | физический адрес версии: `(page, offset)` |

Посмотреть руками:
```sql
SELECT xmin, xmax, ctid, * FROM users WHERE id = 42;
```

## Что реально происходит при DML
- **INSERT** → создаётся одна tuple с `xmin = текущий_xid`, `xmax = 0`.
- **DELETE** → строка не стирается физически; ей проставляется `xmax = текущий_xid`. Версия становится «мёртвой» (dead tuple) для будущих снимков.
- **UPDATE** = **DELETE + INSERT** на уровне версий: старой tuple ставится `xmax`, создаётся новая tuple с новым `xmin`. Старая остаётся как dead tuple.

Отсюда два важных вывода для производительности:
1. **`UPDATE` — это запись новой строки.** Частые апдейты раздувают таблицу мёртвыми версиями (bloat) → нужен VACUUM.
2. **Even DELETE — это запись** (проставление xmax), а не освобождение места. Место возвращает VACUUM.

## Snapshots и видимость
Снимок (snapshot) — это «список XID, которые видны/невидны на момент начала». Версия видна транзакции, если упрощённо:
- её `xmin` принадлежит **закоммиченной** транзакции, которая видна в снимке, **и**
- её `xmax` равен 0 / принадлежит транзакции, которая **не** закоммичена или не видна в снимке.

То есть «видна последняя закоммиченная версия, не удалённая видимыми транзакциями».

## Связь с уровнями изоляции
Уровни изоляции в Postgres различаются **тем, когда берётся снимок**:

| Уровень | Снимок | Поведение |
|---------|--------|-----------|
| `READ COMMITTED` (default) | **новый снимок на каждый statement** | видит чужие коммиты между запросами в одной транзакции (non-repeatable read возможен) |
| `REPEATABLE READ` | **один снимок на всю транзакцию** | стабильная картина; при конфликте записи — `could not serialize` (40001) |
| `SERIALIZABLE` | RR + SSI | предсказывает аномалии сериализации, откатывает с 40001 |

> В Postgres `REPEATABLE READ` уже даёт snapshot isolation (нет фантомов на чтении), что сильнее, чем в стандарте SQL. `SERIALIZABLE` добавляет Serializable Snapshot Isolation (SSI) — ловит write-skew.

**Практика:** под `REPEATABLE READ`/`SERIALIZABLE` приложение **обязано** уметь ретраить транзакцию на ошибке `40001` — это не баг, это контракт.

## Dead tuples → зачем VACUUM (мост к bloat)
Мёртвые версии нельзя удалить, пока их может видеть хоть одна «старая» транзакция. `VACUUM` помечает место от dead tuples переиспользуемым; `autovacuum` делает это в фоне. **Долгая открытая транзакция (или забытый replication slot) держит «горизонт» и не даёт чистить dead tuples всей базы** — частая причина внезапного bloat. Деталь VACUUM — в [[postgresql-deep]].

## Transaction ID wraparound — почему это «тихая катастрофа»
XID — 32-битный счётчик (~4.2 млрд значений), он **зацикливается**. Postgres определяет «прошлое/будущее» транзакции по разнице XID по модулю; если транзакция старше ~2 млрд, она вдруг покажется «из будущего» → её строки станут невидимы → **потеря данных**.

Защита — **freezing**: VACUUM помечает достаточно старые tuple как «замороженные» (frozen), и они считаются видимыми всем независимо от XID. `autovacuum` запускает агрессивный «anti-wraparound» проход по `autovacuum_freeze_max_age`.

Что должен знать senior:
- Мониторить `age(datfrozenxid)` по базам.
- При приближении к лимиту Postgres сыплет варнингами, а на пороге **переходит в read-only / отказывает в новых командах**, чтобы не потерять данные. Лечится ручным `VACUUM (FREEZE)`.
- Anti-wraparound autovacuum **нельзя «просто отменить»** — он критичен.

## HOT updates (Heap-Only Tuples) — важная оптимизация
Если `UPDATE` **не меняет ни одной проиндексированной колонки** и на той же странице есть место — Postgres делает **HOT update**: новая версия кладётся на ту же страницу и связывается цепочкой, **без обновления индексов**. Это резко снижает write amplification и нагрузку на индексы.

Практический вывод: на «горячих» часто обновляемых таблицах **не индексируй колонки, которые постоянно меняются**, и оставляй `fillfactor` < 100 (например 80–90), чтобы на странице было место для HOT-версий.

## Проверить самому
```sql
-- сколько мёртвых версий и когда последний autovacuum
SELECT relname, n_live_tup, n_dead_tup, last_autovacuum
FROM pg_stat_user_tables
ORDER BY n_dead_tup DESC;

-- возраст «заморозки» (риск wraparound)
SELECT datname, age(datfrozenxid) AS xid_age
FROM pg_database ORDER BY xid_age DESC;
```

---

# Часть 2. Блокировки

MVCC снимает блокировки **на чтение**, но запись и DDL по-прежнему берут блокировки. Их два уровня: **табличные** и **строчные**.

## Табличные lock modes (8 режимов)
По возрастанию «силы». Главное — знать, **кто что берёт** и **что с чем конфликтует** (это ключ к zero-downtime миграциям, см. [[zero-downtime-migrations]]).

| Lock mode | Кто берёт (типично) |
|-----------|---------------------|
| `ACCESS SHARE` | `SELECT` |
| `ROW SHARE` | `SELECT ... FOR UPDATE/SHARE` |
| `ROW EXCLUSIVE` | `INSERT`, `UPDATE`, `DELETE` |
| `SHARE UPDATE EXCLUSIVE` | `VACUUM` (не FULL), `ANALYZE`, `CREATE INDEX CONCURRENTLY`, некоторые `ALTER TABLE` |
| `SHARE` | `CREATE INDEX` (без CONCURRENTLY) |
| `SHARE ROW EXCLUSIVE` | `CREATE TRIGGER`, часть `ALTER TABLE` |
| `EXCLUSIVE` | `REFRESH MATERIALIZED VIEW CONCURRENTLY` |
| `ACCESS EXCLUSIVE` | `DROP`, `TRUNCATE`, `VACUUM FULL`, многие `ALTER TABLE`, `LOCK TABLE` (default) |

Два правила, которые надо помнить наизусть:
- **`ACCESS EXCLUSIVE` конфликтует со ВСЕМ**, включая обычный `SELECT`. Любая операция, берущая его на большой таблице, останавливает прод.
- **`ACCESS SHARE` (обычный SELECT) конфликтует только с `ACCESS EXCLUSIVE`.** Поэтому читающая нагрузка спокойно сосуществует с записью и с `CREATE INDEX CONCURRENTLY`.

## Строчные блокировки
Берутся явно через `SELECT ... FOR ...`:

| Режим | Сила | Где |
|-------|------|-----|
| `FOR UPDATE` | сильнейший | «я буду менять/удалять строку» |
| `FOR NO KEY UPDATE` | слабее | `UPDATE`, не трогающий уникальный ключ |
| `FOR SHARE` | shared | «не дайте менять, но читать можно» |
| `FOR KEY SHARE` | слабейший | используется проверками FK |

`FOR UPDATE` и `FOR KEY SHARE` не конфликтуют между собой так, чтобы FK-проверки не блокировали обычные апдейты — поэтому в современных версиях FK редко создают неожиданные блокировки.

## Паттерн «возьми и обработай»
```sql
BEGIN;
SELECT * FROM accounts WHERE id = 42 FOR UPDATE;  -- блокируем строку
UPDATE accounts SET balance = balance - 100 WHERE id = 42;
COMMIT;  -- блокировка снимается на COMMIT/ROLLBACK
```
Строчные блокировки держатся **до конца транзакции**. Отсюда правило: транзакции с `FOR UPDATE` должны быть короткими.

## ⭐ SKIP LOCKED — очередь задач без брокера
Главный приём для конкурентной обработки. Несколько воркеров читают из одной таблицы, и каждый берёт **только не залоченные** строки, не мешая друг другу:

```sql
-- один воркер забирает пачку задач атомарно
WITH picked AS (
    SELECT id
    FROM jobs
    WHERE status = 'pending'
    ORDER BY created_at
    FOR UPDATE SKIP LOCKED   -- пропускаем строки, занятые другими воркерами
    LIMIT 20
)
UPDATE jobs j
SET status = 'processing', locked_at = now()
FROM picked
WHERE j.id = picked.id
RETURNING j.*;
```
`SKIP LOCKED` означает «не жди залоченные строки — пропусти их». Без него все воркеры выстроились бы в очередь на первую строку. Это даёт горизонтальное масштабирование обработки **на голом Postgres**, без RabbitMQ/Redis — для умеренных нагрузок этого часто достаточно.

`NOWAIT` — родственник: вместо пропуска **сразу падает с ошибкой**, если строка занята («не могу взять прямо сейчас — не хочу ждать»).

## Advisory locks
Блокировки «по числу», не привязанные к строкам — application-level mutex через БД (`pg_advisory_lock(key)`, `pg_try_advisory_lock(key)`). Полезно для «только один экземпляр сервиса делает X». Деталь — в [[postgresql-deep]] (раздел про advisory locks).

## Deadlock: как возникает и как не допускать
**Сценарий:** транзакция A блокирует строку 1, затем хочет строку 2. Транзакция B блокирует строку 2, затем хочет строку 1. Обе ждут друг друга → цикл.

**Как Postgres реагирует:** по таймеру `deadlock_timeout` (default 1s) строит wait-for граф, обнаруживает цикл и **убивает одну из транзакций** с ошибкой `40P01 (deadlock_detected)`. Вторая продолжается.

**Профилактика (по убыванию важности):**
1. **Единый порядок захвата ресурсов** во всём коде (например, всегда блокировать строки в порядке возрастания `id`: `ORDER BY id FOR UPDATE`).
2. **Короткие транзакции**, минимум удерживаемых блокировок.
3. `lock_timeout` — чтобы зависшая блокировка не висела вечно (`SET lock_timeout = '3s'`).
4. Приложение **ретраит** транзакцию на `40P01` (как и на `40001`).

## Мониторинг блокировок
«Кто кого блокирует прямо сейчас»:
```sql
SELECT
    blocked.pid          AS blocked_pid,
    blocked.query        AS blocked_query,
    blocking.pid         AS blocking_pid,
    blocking.query       AS blocking_query
FROM pg_stat_activity blocked
JOIN pg_stat_activity blocking
    ON blocking.pid = ANY(pg_blocking_pids(blocked.pid))
WHERE blocked.wait_event_type = 'Lock';
```
`pg_locks` — низкоуровневый список всех блокировок; `pg_blocking_pids(pid)` — самый удобный способ найти «виновника».

---

# Часть 3. Конкурентные вставки и идемпотентность

## Проблема гонки
Два запроса параллельно вставляют строку с одним и тем же уникальным ключом. Один коммитит первым; второй получает `23505 (unique_violation)`. «Проверить SELECT-ом, потом INSERT» **не спасает** — между проверкой и вставкой влезает конкурент (классический TOCTOU). Решение — атомарный `INSERT ... ON CONFLICT`.

## ON CONFLICT DO NOTHING
```sql
INSERT INTO tags (name) VALUES ('postgres')
ON CONFLICT (name) DO NOTHING;
```
Если строка с таким `name` уже есть — вставка молча пропускается, ошибки нет. Требует **уникального индекса/ограничения** на конфликтную колонку.

## ON CONFLICT DO UPDATE (UPSERT)
```sql
INSERT INTO counters (key, hits) VALUES ('home', 1)
ON CONFLICT (key)
DO UPDATE SET hits = counters.hits + EXCLUDED.hits;
```
- `EXCLUDED` — **псевдострока с теми значениями, которые пытались вставить**. `EXCLUDED.hits` здесь = 1.
- `counters.hits` — текущее значение в таблице. Получается атомарный инкремент даже при гонке.

## Нюансы, которые отличают junior от senior
- **Конфликт-таргет обязателен для DO UPDATE.** Указываешь либо колонки `(key)`, либо `ON CONSTRAINT constraint_name`. Под капотом Postgres матчит по конкретному уникальному индексу.
- **Partial unique index** → нужен предикат: `ON CONFLICT (email) WHERE deleted_at IS NULL`.
- **Только один арбитр.** `ON CONFLICT` разруливает конфликт по одному уникальному индексу. Если у таблицы два разных уникальных ограничения и нарушиться может любое — UPSERT не покроет оба сразу.
- **`RETURNING` работает**, но с засадой: при `DO NOTHING` и срабатывании конфликта строка **не возвращается** (ничего не вставилось и не обновилось). Если нужен id и в случае конфликта — используй `DO UPDATE SET key = EXCLUDED.key` (no-op апдейт) ради `RETURNING`, либо отдельный `SELECT`.
- **Идемпотентность операций.** UPSERT по натуральному/идемпотентному ключу (`idempotency_key`) — основа безопасных ретраев платежей/вебхуков: повтор запроса не создаёт дубликат.

## MERGE (PG 15+) — не путать с UPSERT
`MERGE` (SQL-стандарт) умеет `WHEN MATCHED` / `WHEN NOT MATCHED` с произвольной логикой — мощно для батчевой синхронизации таблиц. **Но он не даёт той же атомарной защиты от конкурентной вставки, что `ON CONFLICT`:** при гонке `MERGE` может всё равно словить `unique_violation`. Правило: **для конкурентного upsert по ключу — `ON CONFLICT`; для сложной батч-синхронизации в одной транзакции — `MERGE`.**

---

# Часть 4. LISTEN / NOTIFY — pub/sub из коробки

Postgres умеет асинхронные уведомления между сессиями — лёгкий pub/sub без брокера.

```sql
-- слушатель
LISTEN order_events;

-- издатель (из триггера или приложения)
NOTIFY order_events, 'order:42:paid';
-- или с динамическим payload:
SELECT pg_notify('order_events', 'order:' || NEW.id || ':paid');
```

Свойства и ограничения:
- Payload ≤ 8000 байт.
- **Не durable.** Если в момент `NOTIFY` никто не слушает канал — сообщение теряется. Это не очередь и не event log.
- Уведомления доставляются **после COMMIT** транзакции-издателя.
- Хорошо для «инвалидируй кэш», «появилась новая задача — проснись и забери через SKIP LOCKED». Плохо как замена Kafka/durable-очереди.

Частый продакшн-паттерн: триггер `AFTER INSERT` на `jobs` шлёт `NOTIFY`, воркер по уведомлению просыпается и забирает задачи через `FOR UPDATE SKIP LOCKED` (LISTEN/NOTIFY как «будильник», SKIP LOCKED как «надёжная выборка»).

---

# Вызов из .NET (Npgsql)

## Очередь задач со SKIP LOCKED
```csharp
public async Task<IReadOnlyList<JobDto>> ClaimBatchAsync(int batchSize, CancellationToken ct)
{
    await using var conn = await dataSource.OpenConnectionAsync(ct).ConfigureAwait(false);
    await using var tx = await conn.BeginTransactionAsync(ct).ConfigureAwait(false);

    const string sql = """
        WITH picked AS (
            SELECT id FROM jobs
            WHERE status = 'pending'
            ORDER BY created_at
            FOR UPDATE SKIP LOCKED
            LIMIT @batch
        )
        UPDATE jobs j SET status = 'processing', locked_at = now()
        FROM picked WHERE j.id = picked.id
        RETURNING j.id, j.payload, j.created_at;
        """;

    await using var cmd = new NpgsqlCommand(sql, conn, tx);
    cmd.Parameters.AddWithValue("batch", batchSize);

    var jobs = new List<JobDto>();
    await using (var reader = await cmd.ExecuteReaderAsync(ct).ConfigureAwait(false))
    {
        while (await reader.ReadAsync(ct).ConfigureAwait(false))
        {
            jobs.Add(new JobDto
            {
                Id = reader.GetInt64(0),
                Payload = reader.GetString(1),
                CreatedAt = reader.GetFieldValue<DateTimeOffset>(2),
            });
        }
    }
    await tx.CommitAsync(ct).ConfigureAwait(false);
    return jobs;
}
```

> [!warning] `AddWithValue` здесь и ниже — упрощение
> `AddWithValue` выводит `NpgsqlDbType` из CLR-типа значения; для неоднозначных типов это даёт implicit conversion, который ломает использование индекса. В проде задавай тип явно через `.Add(new NpgsqlParameter(name, NpgsqlDbType.X) { Value = ... })`. Разбор — [[security-practices]].

## Ретрай на serialization/deadlock
```csharp
async Task<T> WithRetryAsync<T>(Func<Task<T>> action, int maxAttempts = 3)
{
    for (var attempt = 1; ; attempt++)
    {
        try { return await action().ConfigureAwait(false); }
        catch (PostgresException ex) when (
            ex.SqlState is PostgresErrorCodes.SerializationFailure   // 40001
                       or PostgresErrorCodes.DeadlockDetected         // 40P01
            && attempt < maxAttempts)
        {
            await Task.Delay(attempt * 50, ct).ConfigureAwait(false); // backoff
        }
    }
}
```

## UPSERT
```csharp
const string sql = """
    INSERT INTO counters (key, hits) VALUES (@key, 1)
    ON CONFLICT (key) DO UPDATE SET hits = counters.hits + EXCLUDED.hits
    RETURNING hits;
    """;
await using var cmd = new NpgsqlCommand(sql, conn);
cmd.Parameters.AddWithValue("key", key);
var total = (int)(await cmd.ExecuteScalarAsync(ct).ConfigureAwait(false))!;
```

## Слушатель LISTEN/NOTIFY
```csharp
await using var conn = await dataSource.OpenConnectionAsync(ct).ConfigureAwait(false);
conn.Notification += (_, e) => logger.LogInformation("Notify {Channel}: {Payload}", e.Channel, e.Payload);

await using (var cmd = new NpgsqlCommand("LISTEN order_events", conn))
    await cmd.ExecuteNonQueryAsync(ct).ConfigureAwait(false);

while (!ct.IsCancellationRequested)
    await conn.WaitAsync(ct).ConfigureAwait(false);  // блокируется до прихода уведомления
```

---

## Типичные вопросы на собесе

- **Блокирует ли `SELECT` строку, которую кто-то обновляет?** → Нет. MVCC отдаёт читателю предыдущую версию; читатели не ждут писателей.
- **Что физически делает `UPDATE`?** → Создаёт новую версию строки, старой проставляет `xmax`; старая становится dead tuple, место чистит VACUUM.
- **Зачем VACUUM, если есть autovacuum?** → autovacuum обычно достаточно, но долгая транзакция держит горизонт и копит bloat; плюс ручной `VACUUM (FREEZE)` против wraparound.
- **Что такое transaction ID wraparound и почему он опасен?** → 32-битный XID зацикливается; без freezing старые строки «уходят в будущее» и становятся невидимыми (потеря данных). VACUUM замораживает старые tuple.
- **Чем `READ COMMITTED` отличается от `REPEATABLE READ` в Postgres?** → Моментом снимка: на каждый statement vs один на транзакцию. RR в PG = snapshot isolation, требует ретраев на 40001.
- **Как сделать очередь задач на одном Postgres?** → `SELECT ... FOR UPDATE SKIP LOCKED LIMIT n` + `UPDATE status` в одной транзакции; опционально `LISTEN/NOTIFY` как будильник.
- **`SKIP LOCKED` vs `NOWAIT`?** → SKIP LOCKED пропускает занятые строки; NOWAIT сразу падает с ошибкой.
- **Как избежать deadlock?** → Единый порядок захвата строк (`ORDER BY id FOR UPDATE`), короткие транзакции, `lock_timeout`, ретрай на 40P01.
- **Как безопасно сделать «вставь или обнови» при гонке?** → `INSERT ... ON CONFLICT DO UPDATE`; `SELECT`-then-`INSERT` ломается из-за гонки.
- **`MERGE` заменяет `ON CONFLICT`?** → Нет, MERGE не атомарен против конкурентной вставки; для конкурентного upsert — `ON CONFLICT`.
- **Какой lock берёт `ALTER TABLE ... ADD COLUMN` и почему это опасно?** → `ACCESS EXCLUSIVE`, конфликтует даже с SELECT (подробнее — zero-downtime).

## Pitfalls
1. **`SELECT`-then-`INSERT` для уникальности** — гонка → дубликат/`23505`. Только `ON CONFLICT`.
2. **Долгая открытая транзакция** — держит горизонт MVCC, autovacuum не чистит dead tuples → bloat всей базы. Следить за `idle in transaction`.
3. **`RETURNING` при `ON CONFLICT DO NOTHING`** — не вернёт строку при конфликте. Использовать no-op `DO UPDATE` или отдельный SELECT.
4. **Нет ретрая на 40001/40P01** под `REPEATABLE READ`/`SERIALIZABLE` и при deadlock — приложение случайно «теряет» транзакции.
5. **`FOR UPDATE` без `ORDER BY`** в нескольких местах кода — разный порядок захвата → deadlock.
6. **Индекс на часто обновляемой колонке** — ломает HOT-update, усиливает write amplification и bloat индекса.
7. **LISTEN/NOTIFY как durable-очередь** — теряет сообщения без активного слушателя. Это будильник, не очередь.
8. **Игнор `age(datfrozenxid)`** — wraparound подкрадывается тихо и кладёт прод в read-only.

## См. также
- [[postgresql-deep]] — VACUUM/autovacuum/bloat, advisory locks, isolation, optimistic concurrency (xmin)
- [[zero-downtime-migrations]] — lock modes на практике, безопасный DDL
- [[optimization]] — deadlock, мониторинг в проде, изоляция
- [[postgres-functions-triggers]] — триггеры (часто шлют NOTIFY)
- [[indexes-deep]] — fillfactor, index-only scan

## Reading list
- PostgreSQL docs: Concurrency Control (MVCC), Explicit Locking, `INSERT ... ON CONFLICT`, `NOTIFY`/`LISTEN`, Routine Vacuuming (wraparound)
- Npgsql docs: Transactions, Notifications, error codes (`PostgresErrorCodes`)

---
*Добавлено: 2026-05-29 — расширение SQL-раздела (MVCC, блокировки, конкурентность, UPSERT, LISTEN/NOTIFY)*
