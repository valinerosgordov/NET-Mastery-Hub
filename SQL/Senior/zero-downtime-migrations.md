---
tags: [postgresql, migrations, zero-downtime, ddl, locking, ef-core, expand-contract, dba]
level: Senior
---

# Zero-downtime миграции схемы в PostgreSQL

## Кратко
Как менять схему большой таблицы под нагрузкой, не роняя прод: какие DDL-операции дёшевы, а какие переписывают/сканируют таблицу под жёсткой блокировкой; приёмы `lock_timeout` + retry, `CREATE INDEX CONCURRENTLY`, `NOT VALID` + `VALIDATE`, `SET NOT NULL` без full scan; и паттерн **expand-contract**, который нужен потому, что во время деплоя старый и новый код живут одновременно. Раздел прямо под требование «тысячи пользователей, быстрый отклик».

---

## Почему миграция роняет прод

Две независимые причины — обе надо понимать.

### 1. DDL берёт `ACCESS EXCLUSIVE`, а он конфликтует даже с SELECT
Большинство `ALTER TABLE` берут самый сильный табличный lock — `ACCESS EXCLUSIVE` (см. матрицу в [[mvcc-and-locking]]). Он несовместим **со всем**, включая обычные `SELECT`. Пока операция держит lock, любой запрос к таблице ждёт.

### 2. Head-of-line blocking в очереди блокировок ⭐
Это то, что отличает «прочитал про CONCURRENTLY» от реального понимания. Если `ALTER` не может сразу взять `ACCESS EXCLUSIVE` (таблицу читает долгий запрос), он **встаёт в очередь** — и **все последующие запросы встают в очередь ЗА ним**, даже безобидные `SELECT`, которые с текущим держателем совместимы.

Итог: один `ALTER`, ждущий за одним медленным `SELECT`, замораживает **всю** нагрузку на таблицу, пока медленный запрос не закончится. Поэтому **DDL без `lock_timeout` на нагруженной таблице — это мина**.

### 3. Деплой не атомарен
Во время выкатки старые и новые поды работают **одновременно**. Миграция должна быть совместима с **обеими** версиями кода → отсюда expand-contract.

---

## Карта стоимости операций

«Дёшево» = меняется только каталог (метаданные), доли секунды. «Дорого» = full table rewrite или full scan под `ACCESS EXCLUSIVE`.

| Операция | Стоимость | Нюанс |
|----------|-----------|-------|
| `ADD COLUMN` nullable, без дефолта | дёшево | метаданные |
| `ADD COLUMN ... DEFAULT <const>` | дёшево (PG 11+) | non-volatile дефолт хранится в каталоге, таблица **не** переписывается |
| `ADD COLUMN ... DEFAULT now()/random()` | **дорого** | volatile дефолт → full rewrite. Избегать, backfill вручную |
| `ADD COLUMN ... NOT NULL` без дефолта | **нельзя** при наличии строк | нужен дефолт или отдельный backfill + поздний `SET NOT NULL` |
| `DROP COLUMN` | дёшево | логическое удаление; место вернёт VACUUM/rewrite позже |
| `RENAME COLUMN` / `RENAME TABLE` | дёшево, **но ломает старый код** | только через expand-contract |
| `SET NOT NULL` | **дорого** (full scan) | PG 12+: scan можно избежать через валидный CHECK (см. ниже) |
| `ADD CHECK` / `ADD FOREIGN KEY` | **дорого** (scan) | спасает `NOT VALID` + `VALIDATE` |
| `ALTER COLUMN TYPE` | обычно **rewrite** | исключения без rewrite: `varchar(n)→varchar(m>n)`, `→text` (бинарно совместимы) |
| `CREATE INDEX` | блокирует запись (`SHARE`) | использовать `CONCURRENTLY` |
| `TRUNCATE`, `VACUUM FULL`, `DROP TABLE` | `ACCESS EXCLUSIVE` | на проде осторожно |

---

## Приём №1: `lock_timeout` + retry (обязателен всегда)

Никогда не пускай DDL без ограничения ожидания блокировки — иначе словишь head-of-line blocking.

```sql
SET lock_timeout = '3s';
ALTER TABLE orders ADD COLUMN note text;  -- если за 3с lock не взят → ошибка 55P03
```
Если получаешь `55P03 (lock_not_available)` — значит таблица занята; **подожди и повтори**, а не вешай прод. Это лучше, чем «успешный» ALTER, заморозивший всех.

Паттерн: короткий `lock_timeout`, ретраи с backoff, желательно в окно низкой нагрузки.

---

## Приём №2: `CREATE INDEX CONCURRENTLY`

```sql
CREATE INDEX CONCURRENTLY idx_orders_user ON orders (user_id);
```

| | `CREATE INDEX` | `CREATE INDEX CONCURRENTLY` |
|--|----------------|------------------------------|
| Lock | `SHARE` (блокирует запись) | `SHARE UPDATE EXCLUSIVE` (читать и писать можно) |
| Скорость | быстрее | медленнее (два прохода по таблице) |
| В транзакции | можно | **нельзя** |
| При сбое | чисто | может остаться **INVALID** индекс |

Если `CONCURRENTLY` упал — остаётся невалидный индекс: его надо `DROP INDEX CONCURRENTLY` и повторить. Проверка:
```sql
SELECT indexrelid::regclass FROM pg_index WHERE NOT indisvalid;
```
`DROP INDEX CONCURRENTLY` аналогично снимает индекс, не блокируя нагрузку.

---

## Приём №3: `NOT VALID` + `VALIDATE` (constraints без боли)

Добавление CHECK или FK обычно сканирует всю таблицу под `ACCESS EXCLUSIVE`. Разбиваем на два шага:

```sql
-- 1) мгновенно: не сканирует существующие строки, но enforce'ит для новых
ALTER TABLE orders
    ADD CONSTRAINT chk_amount_positive CHECK (amount > 0) NOT VALID;

-- 2) отдельно: сканирует под SHARE UPDATE EXCLUSIVE (чтение и запись не блокируются)
ALTER TABLE orders VALIDATE CONSTRAINT chk_amount_positive;
```
То же для FK:
```sql
ALTER TABLE orders
    ADD CONSTRAINT fk_orders_user FOREIGN KEY (user_id) REFERENCES users(id) NOT VALID;
ALTER TABLE orders VALIDATE CONSTRAINT fk_orders_user;
```
`NOT VALID` означает «старые строки не проверены, новые — проверяются». `VALIDATE` догоняет старые без тяжёлой блокировки.

---

## Приём №4: `SET NOT NULL` без full scan (PG 12+)

Прямой `SET NOT NULL` сканирует таблицу под `ACCESS EXCLUSIVE`. Обход:

```sql
-- 1) добавляем NOT VALID CHECK, что колонка не NULL (мгновенно)
ALTER TABLE orders ADD CONSTRAINT orders_note_nn CHECK (note IS NOT NULL) NOT VALID;
-- 2) валидируем без тяжёлого лока
ALTER TABLE orders VALIDATE CONSTRAINT orders_note_nn;
-- 3) теперь SET NOT NULL видит валидный CHECK и ПРОПУСКАЕТ полный скан
ALTER TABLE orders ALTER COLUMN note SET NOT NULL;
-- 4) CHECK больше не нужен
ALTER TABLE orders DROP CONSTRAINT orders_note_nn;
```

---

## Expand-Contract (parallel change) — концептуальное ядро

Любое **несовместимое** изменение (rename, смена типа, split/merge колонок) разбивается на фазы, разнесённые по релизам, потому что во время деплоя оба кода живут:

1. **EXPAND** — добавить новое рядом со старым. Backward-compatible: старый код продолжает работать со старым, новый код — пишет/читает новое. Деплой кода, умеющего dual-write.
2. **MIGRATE / backfill** — заполнить новое по старым данным (батчами, см. ниже). К этому моменту приложение пишет в оба места.
3. **CONTRACT** — когда весь старый код выведен из эксплуатации, переключить чтение на новое и **удалить старое** отдельной поздней миграцией.

### Пример: переименование колонки `name → full_name`
**Нельзя** `RENAME` — это мгновенно сломает работающий старый код, который шлёт `name`. Вместо:
1. EXPAND: `ADD COLUMN full_name text`. Приложение пишет в **обе** колонки (dual-write), читает из старой.
2. BACKFILL: `UPDATE ... SET full_name = name` батчами.
3. Переключить чтение на `full_name` (новый релиз).
4. CONTRACT: `DROP COLUMN name`, убрать dual-write.

### Пример: смена типа колонки
Add `amount_new numeric(18,4)` → dual-write → backfill из `amount` → switch reads → drop `amount` → (опц.) rename `amount_new → amount` поздним expand-contract.

---

## Backfill больших таблиц безопасно

Одна гигантская `UPDATE` = долгая транзакция (держит горизонт MVCC → bloat), огромный WAL, и блокировки. Бьём на батчи по PK:

```sql
-- процедура: апдейтит пачками с промежуточными COMMIT (см. postgres-functions-triggers)
CREATE OR REPLACE PROCEDURE backfill_full_name(p_batch int DEFAULT 5000)
LANGUAGE plpgsql AS $$
DECLARE
    v_last bigint := 0;
    v_rows int;
BEGIN
    LOOP
        UPDATE users u SET full_name = name
        WHERE u.id IN (
            SELECT id FROM users
            WHERE id > v_last AND full_name IS NULL
            ORDER BY id LIMIT p_batch
        )
        RETURNING u.id INTO v_last;   -- упрощённо; на практике берём max(id) пачки

        GET DIAGNOSTICS v_rows = ROW_COUNT;
        EXIT WHEN v_rows = 0;
        COMMIT;                        -- отпускаем блокировки, даём autovacuum работать
        PERFORM pg_sleep(0.1);         -- троттлинг под нагрузкой
    END LOOP;
END;
$$;
CALL backfill_full_name(5000);
```
Принципы backfill: маленькие батчи, промежуточные коммиты, троттлинг, идемпотентность (`WHERE full_name IS NULL`), движение по индексу PK.

---

## Опасные операции → безопасные альтернативы

| Хочу | Наивно (роняет) | Безопасно |
|------|-----------------|-----------|
| Новая NOT NULL колонка | `ADD COLUMN x int NOT NULL` | `ADD COLUMN` nullable → backfill → `SET NOT NULL` через CHECK-трюк |
| Колонка с дефолтом-функцией | `ADD COLUMN ts timestamptz DEFAULT now()` | `ADD COLUMN` nullable → backfill батчами → дефолт для новых |
| Индекс на большой таблице | `CREATE INDEX` | `CREATE INDEX CONCURRENTLY` |
| Внешний ключ | `ADD FOREIGN KEY ...` | `... NOT VALID` → `VALIDATE` |
| CHECK-ограничение | `ADD CHECK ...` | `... NOT VALID` → `VALIDATE` |
| Переименовать колонку | `RENAME COLUMN` | expand-contract (add → dual-write → backfill → switch → drop) |
| Сменить тип | `ALTER COLUMN TYPE` (rewrite) | новая колонка + backfill + switch + drop |
| Удалить колонку, которую читает старый код | `DROP COLUMN` сразу | сначала вывести старый код, потом drop (contract-фаза) |

---

## Специфика EF Core (твой стек)

EF Core генерирует миграции, но **по умолчанию они не zero-downtime**. Что важно:

### Каждая миграция оборачивается в транзакцию
Поэтому `CREATE INDEX CONCURRENTLY` **нельзя** через обычный `migrationBuilder`. Нужно подавить транзакцию:
```csharp
protected override void Up(MigrationBuilder migrationBuilder)
{
    migrationBuilder.Sql(
        "CREATE INDEX CONCURRENTLY IF NOT EXISTS idx_orders_user ON orders (user_id);",
        suppressTransaction: true);   // ← обязательно для CONCURRENTLY
}
```

### EF не ставит `lock_timeout`
Добавляй сам в начало рискованной миграции:
```csharp
migrationBuilder.Sql("SET lock_timeout = '3s';");
```

### Ревью сгенерированного SQL — не накатывай вслепую
```bash
# посмотреть SQL миграции (между двумя миграциями)
dotnet ef migrations script PreviousMigration TargetMigration

# идемпотентный скрипт для CI/CD (безопасен к повторному прогону)
dotnet ef migrations script --idempotent --output migrate.sql
```
EF может сгенерировать `ALTER COLUMN`, который под капотом = rewrite. Всегда смотри итоговый SQL на больших таблицах.

### Разноси expand и contract по релизам
EXPAND-миграция едет с релизом N (добавляет колонку, dual-write в коде). CONTRACT-миграция (`DROP COLUMN`) — отдельной миграцией в релизе N+1, когда старый код уже не работает. **Не объединяй их в одну миграцию.**

### `NOT NULL` через raw SQL
EF-овский `SetColumnNullable(false)` = `SET NOT NULL` с full scan. На больших таблицах делай CHECK-трюк через `migrationBuilder.Sql(...)`.

---

## Чеклист безопасной миграции
- [ ] Операция дёшевая (метаданные) или дорогая (rewrite/scan)? Свериться с картой стоимости.
- [ ] Если дорогая — разбита на безопасные шаги (CONCURRENTLY / NOT VALID / CHECK-трюк)?
- [ ] Выставлен `lock_timeout` + предусмотрен retry?
- [ ] Изменение backward-compatible со старым кодом (expand-contract)?
- [ ] Backfill — батчами с коммитами и троттлингом, не одной транзакцией?
- [ ] `DROP`/`contract` отложен до вывода старого кода?
- [ ] Сгенерированный EF SQL просмотрен глазами?
- [ ] Откат (rollback) продуман — особенно после удаления данных?

---

## Типичные вопросы на собесе

- **Почему `ALTER TABLE ADD COLUMN NOT NULL DEFAULT 0` мог раньше ронять прод?** → volatile/historic поведение: full rewrite под `ACCESS EXCLUSIVE`. В PG 11+ non-volatile дефолт — метаданные, дёшево; `now()` — всё ещё rewrite.
- **Что такое head-of-line blocking при DDL?** → `ALTER`, ждущий `ACCESS EXCLUSIVE`, встаёт в очередь, и все запросы после него ждут тоже — даже совместимые SELECT. Поэтому `lock_timeout`.
- **Чем `CREATE INDEX CONCURRENTLY` отличается?** → берёт `SHARE UPDATE EXCLUSIVE` вместо `SHARE`, не блокирует запись; медленнее, два прохода, нельзя в транзакции, может оставить invalid индекс.
- **Как добавить FK на большую таблицу без даунтайма?** → `ADD CONSTRAINT ... NOT VALID`, затем `VALIDATE CONSTRAINT` отдельно.
- **Как сделать колонку NOT NULL без full scan?** → добавить `CHECK (col IS NOT NULL) NOT VALID`, `VALIDATE`, потом `SET NOT NULL` (PG 12+ пропустит скан), удалить CHECK.
- **Что такое expand-contract и зачем?** → разнести несовместимое изменение на фазы, потому что во время деплоя старый и новый код работают одновременно.
- **Как переименовать колонку без даунтайма?** → не `RENAME`; add new → dual-write → backfill → switch reads → drop old.
- **Как backfill миллион строк?** → батчами по PK с промежуточными коммитами и троттлингом, не одной `UPDATE`.
- **Что не так с EF-миграциями из коробки?** → нет `lock_timeout`, нет `CONCURRENTLY` (транзакция), `ALTER COLUMN` может быть rewrite. Ревьюить SQL, использовать `suppressTransaction`.

## Pitfalls
1. **DDL без `lock_timeout`** на нагруженной таблице → head-of-line blocking, прод встаёт за одним медленным SELECT.
2. **`ADD COLUMN ... DEFAULT now()`** — volatile дефолт = full rewrite. Использовать nullable + backfill.
3. **`CONCURRENTLY` внутри транзакции EF** → ошибка. `suppressTransaction: true`.
4. **Одна гигантская `UPDATE` для backfill** → долгая транзакция, bloat, раздутый WAL, блокировки.
5. **`RENAME`/`DROP` в одном релизе с кодом** → старый под падает на отсутствующей/переименованной колонке во время выкатки.
6. **Невалидный индекс после сбоя `CONCURRENTLY`** остаётся и не используется — надо найти (`indisvalid = false`) и пересоздать.
7. **`ALTER COLUMN TYPE` вслепую** — часто скрытый full rewrite под `ACCESS EXCLUSIVE`.
8. **Накат EF-миграции без просмотра SQL** — генератор не знает про размер таблицы и нагрузку.

## См. также
- [[mvcc-and-locking]] — lock modes, очередь блокировок, MVCC/bloat за backfill
- [[postgres-functions-triggers]] — процедуры для батч-backfill с COMMIT
- [[postgresql-deep]] — VACUUM, bulk-операции
- [[indexes-deep]] — `CREATE INDEX CONCURRENTLY`, invalid index
- [[../EFCore/...]] — миграции EF Core (общий контекст)

## Reading list
- PostgreSQL docs: `ALTER TABLE` (lock levels), `CREATE INDEX CONCURRENTLY`, constraints (`NOT VALID`/`VALIDATE`)
- EF Core docs: Migrations, custom SQL operations, `--idempotent` scripts
- «Postgres at scale» / zero-downtime migration практики (Braintree, GitLab DB migration helpers — как референс паттернов)

---
*Добавлено: 2026-05-29 — расширение SQL-раздела (zero-downtime DDL, expand-contract, EF Core миграции)*
