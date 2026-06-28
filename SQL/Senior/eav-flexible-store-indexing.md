---
tags: [postgresql, sql, eav, indexes, multi-tenant, partial-index, covering-index, performance]
level: Senior
date: 2026-06-19
---

# EAV / Flexible Store Indexing — гибкое хранилище атрибутов без деградации

> Как держать EAV-таблицу (`_values` с типизированными колонками `_Long`/`_String`/`_DateTimeOffset`/`_Boolean`) быстрой при сотнях схем и высокой cardinality: ОДИН фиксированный набор индексов из DDL обслуживает ЛЮБУЮ схему, лидирующий `_id_structure` отсекает ~99.75% строк, covering-индексы дают index-only scan для facet-запросов, type-segregated partial-индексы обходят btree-лимит ~2700 байт, а три UNIQUE partial вместо одного монолита разделяют scalar / nested / array.

## Что это, зачем и когда

### Проблема EAV

**EAV (Entity-Attribute-Value)** — паттерн, в котором атрибуты сущности хранятся не как колонки таблицы, а как строки в общей таблице значений. Нужен, когда схема **не фиксирована**: пользователь/тенант определяет свои классы объектов и поля в рантайме, без `ALTER TABLE` и миграций.

**Аналогия:** обычная таблица — типография с фиксированным бланком (колонки заранее напечатаны). EAV — стопка карточек: каждая карточка = одно `(объект, атрибут, значение)`, и какие атрибуты есть у объекта решается на лету.

Классический EAV печально известен медлительностью: каждый объект — это N строк, а сборка одного объекта — N lookup'ов. Но при правильной индексной стратегии EAV-`_values` остаётся быстрым даже на миллиардах строк. Ключевая идея — **не индексировать "по классам", а построить один универсальный набор индексов, который физически отсекает чужие схемы до сравнения значений.**

### Целевая модель таблицы

```sql
-- Одна таблица значений на весь store. _id_structure = "что это за атрибут"
-- (FK на метаданные класса+поля). _id_object = конкретный объект.
-- _array_index = позиция в массиве (NULL для скаляра).
CREATE TABLE _values (
    _id_structure    bigint  NOT NULL,
    _id_object       bigint  NOT NULL,
    _array_index     integer NULL,        -- NULL = scalar/nested; >=0 = array element
    _id_parent       bigint  NULL,        -- для nested: ссылка на родительский объект
    _Long            bigint  NULL,
    _String          text    NULL,
    _DateTimeOffset  timestamptz NULL,
    _Boolean         boolean NULL
);
```

Ровно одна из типизированных колонок (`_Long` / `_String` / `_DateTimeOffset` / `_Boolean`) заполнена на строку — остальные `NULL`. Тип определяется метаданными `_id_structure`. Это и есть рычаг: индексы можно сегрегировать по типу и физически не трогать NULL-строки.

### Когда EAV оправдан, когда нет

| Сигнал | Решение |
|--------|---------|
| Тенант/пользователь создаёт классы и поля в рантайме | EAV `_values` (этот документ) |
| Схема полу-структурирована, читается/пишется документом целиком | `jsonb` + GIN (см. [[postgresql-deep]]) |
| Схема фиксирована, известна на этапе дизайна | Обычные колонки + индексы (см. [[indexes-deep]]) |
| Разреженные атрибуты, но фиксированный union типов | `jsonb` обычно проще EAV |

> [!warning] EAV — это бюджет на индексацию, а не на хранение
> Дёшево хранить `(structure, object, value)` легко. Дорого — отвечать на `WHERE Salary > 80000` по таблице, где Salary перемешан с тысячей других атрибутов всех тенантов. Вся стратегия ниже — про то, чтобы планировщик добрался до нужных значений, не сканируя чужие.

---

## Техника 1 — один фиксированный набор индексов на любую схему

Наивный подход к EAV: для каждого нового класса (или горячего атрибута) создавать свой индекс. Это не масштабируется — тысячи схем дают тысячи индексов, каждый INSERT обновляет их все, а DDL в рантайме под нагрузкой блокирует таблицу.

Правильный подход: **индексы определяются один раз в DDL и обслуживают ЛЮБУЮ схему**, потому что лидируют по `_id_structure`, а не по конкретному значению атрибута. Новый класс = новые значения `_id_structure`, не новые индексы.

```sql
-- НЕ так: per-class индекс — комбинаторный взрыв, рантайм-DDL под локом
-- CREATE INDEX idx_employee_salary ON _values (_Long) WHERE _id_structure = 1042;
-- CREATE INDEX idx_employee_name   ON _values (_String) WHERE _id_structure = 1043;
-- ... ×тысячи классов

-- А так: фиксированный набор, лидирует _id_structure, покрывает все будущие классы
CREATE INDEX ix_values_long
    ON _values (_id_structure, _Long)
    WHERE _Long IS NOT NULL;
```

Этот один индекс отвечает на `WHERE _id_structure = <любой> AND _Long <op> ?` для **всех** числовых атрибутов всех тенантов сразу. Добавление класса не требует DDL.

> [!info] Почему это работает именно для EAV
> В обычной таблице каждый атрибут — отдельная колонка, поэтому индекс физически привязан к колонке. В EAV "имя атрибута" — это **данные** (`_id_structure`), а не имя колонки. Значит индекс по `_id_structure` параметризуется значением, а не структурой — один индекс = бесконечно много логических индексов.

---

## Техника 2 — лидирующий `_id_structure` отсекает ~99.75% строк

`_id_structure` ставится **первой** колонкой каждого composite-индекса. Это применение leftmost-prefix + правила селективности из [[indexes-deep]], но с экстремальным выигрышем: в EAV cardinality `_id_structure` запредельно высока относительно одного запроса.

Прикидка: store с ~400 различными атрибутами (`_id_structure`), значения распределены примерно равномерно. Запрос про один атрибут (`Salary`) касается `1/400 = 0.25%` строк. Лидирующий `_id_structure` в индексе **отсекает остальные ~99.75% ещё до сравнения значения** — планировщик спускается по btree прямо в диапазон нужного атрибута и только там применяет `_Long > 80000`.

```sql
-- Запрос: "все объекты, у которых атрибут Salary (_id_structure=1042) > 80000"
SELECT _id_object
FROM _values
WHERE _id_structure = 1042   -- btree спускается в узкий диапазон: ~0.25% строк
  AND _Long > 80000;         -- сравнение значения только внутри этого диапазона
```

```
Логика indexa ix_values_long (_id_structure, _Long):

  ... _id_structure=1041 ...   ← не читается
  _id_structure=1042 → _Long: [..., 80001, 80500, 91000, ...]  ← только этот диапазон
  ... _id_structure=1043 ...   ← не читается
```

Если бы `_id_structure` стоял вторым (`(_Long, _id_structure)`), планировщик сравнивал бы `_Long > 80000` по значениям **всех** атрибутов (зарплаты, года, ID, счётчики — всё, что хранится в `_Long`), а уже потом фильтровал по структуре. Это разница между seek в 0.25% таблицы и scan большей её части.

> [!tip] Equality перед range — здесь это критично, а не косметика
> `_id_structure = ?` — equality, `_Long > ?` — range. Порядок `(equality, range)` (см. [[indexes-deep]]) обязателен: equality на лидере превращает индекс в "под-индекс конкретного атрибута", внутри которого range уже эффективен.

---

## Техника 3 — covering-индексы с INCLUDE → index-only scan для facet-запросов

Facet-запросы ("дай объекты с Salary > 80000", "посчитай распределение по City") читают по сути только `_id_object` плюс отбираемое значение. Если эти колонки уже в индексе, Postgres делает **index-only scan** — heap (таблицу) не трогает вовсе.

```sql
-- INCLUDE добавляет _id_object в листья индекса (без участия в сортировке)
CREATE INDEX ix_values_long_facet
    ON _values (_id_structure, _Long)
    INCLUDE (_id_object)
    WHERE _Long IS NOT NULL;

-- Index-only scan: всё нужное (_id_structure, _Long, _id_object) в индексе
EXPLAIN (ANALYZE)
SELECT _id_object
FROM _values
WHERE _id_structure = 1042 AND _Long > 80000;
-- → Index Only Scan using ix_values_long_facet
--   Index Cond: (_id_structure = 1042 AND _Long > 80000)
--   Heap Fetches: 0      ← таблица не читалась
```

Без `INCLUDE (_id_object)` план был бы Index Scan + heap fetch на каждую попавшую строку, чтобы достать `_id_object`. На широких facet-выборках (десятки тысяч строк) это разница в разы.

Те же covering-индексы для остальных типов:

```sql
CREATE INDEX ix_values_dto_facet
    ON _values (_id_structure, _DateTimeOffset)
    INCLUDE (_id_object)
    WHERE _DateTimeOffset IS NOT NULL;

CREATE INDEX ix_values_bool_facet
    ON _values (_id_structure, _Boolean)
    INCLUDE (_id_object)
    WHERE _Boolean IS NOT NULL;
```

> [!info] Index-only scan требует свежего visibility map
> Postgres делает index-only scan, только если VACUUM пометил страницы как all-visible (см. VACUUM/bloat в [[postgresql-deep]]). На высокозаписываемой `_values` следи за autovacuum — иначе `Heap Fetches` поползёт вверх и выигрыш растворится.

---

## Техника 4 — type-segregated PARTIAL индексы (обход btree-лимита ~2700 байт)

`_String` — особый случай. Btree в Postgres не индексирует строку, чья запись в индексе превышает ~2704 байта (примерно треть страницы 8 КБ); попытка вставить длинное значение в индексируемую колонку даёт ошибку `index row size ... exceeds btree version 4 maximum`. В EAV `_String` может содержать что угодно — от кода города до многокилобайтного текста.

Решение — **partial-индекс с двумя условиями**: индексировать `_String`, только если он не NULL **и** короче порога. Это одновременно:
1. обходит btree-лимит (длинные строки просто не попадают в индекс, INSERT не падает);
2. убирает NULL-строки из индекса (а их большинство — на строку заполнена одна типизированная колонка из четырёх).

```sql
-- _String индексируется только когда он есть и достаточно короткий для btree
CREATE INDEX ix_values_string
    ON _values (_id_structure, _String)
    INCLUDE (_id_object)
    WHERE _String IS NOT NULL AND length(_String) < 2000;
```

Эта же `WHERE`-сегрегация по типу нужна **каждому** типовому индексу — иначе индекс по `_Long` хранил бы NULL для всех строк, где заполнены `_String`/`_DateTimeOffset`/`_Boolean`. Partial `WHERE _Long IS NOT NULL` выкидывает этот балласт.

```sql
-- Партиал по каждому типу: индекс хранит только "свои" строки, без NULL-bloat
CREATE INDEX ix_values_long ON _values (_id_structure, _Long)
    INCLUDE (_id_object) WHERE _Long IS NOT NULL;

CREATE INDEX ix_values_dto ON _values (_id_structure, _DateTimeOffset)
    INCLUDE (_id_object) WHERE _DateTimeOffset IS NOT NULL;

CREATE INDEX ix_values_bool ON _values (_id_structure, _Boolean)
    INCLUDE (_id_object) WHERE _Boolean IS NOT NULL;
```

> [!warning] Длинные строки всё ещё надо уметь искать
> Partial-индекс по короткому `_String` означает, что поиск по длинному тексту индекс не использует. Если по длинным значениям нужен поиск — отдельный GIN full-text/trigram индекс (см. полнотекстовый поиск и `pg_trgm` в [[postgresql-deep]] и [[indexes-deep]]), не btree. Btree-индекс короткого `_String` — для equality/prefix/range на компактных значениях (коды, статусы, имена).

Чтобы планировщик использовал partial-индекс, предикат запроса должен **логически следовать** из `WHERE` индекса. Для короткого `_String` это обычно даётся бесплатно (equality на короткое значение), но для range стоит добавить границу длины явно или фильтровать на стороне приложения.

---

## Техника 5 — три UNIQUE PARTIAL вместо одного монолита (scalar / nested / array)

Нужна гарантия уникальности: один объект не должен иметь два значения одного скалярного атрибута. Наивно — один UNIQUE на всё:

```sql
-- НЕ так: монолитный constraint ломается на nested/array
-- ALTER TABLE _values ADD CONSTRAINT uq_values UNIQUE (_id_structure, _id_object);
```

Этот constraint неверен сразу по двум причинам:
- **Array:** у массива несколько строк с одним `(_id_structure, _id_object)`, различаемых `_array_index` — монолитный UNIQUE их запретит.
- **Nested:** вложенные объекты различаются `_id_parent` — у одного `_id_object` могут быть значения под разными родителями.

Решение — **три UNIQUE PARTIAL индекса**, каждый для своей формы хранения, разделённые по `_array_index` / `_id_parent`:

```sql
-- 1. Scalar: одно значение на (структура, объект). Ни массив, ни вложенный.
CREATE UNIQUE INDEX uq_values_scalar
    ON _values (_id_structure, _id_object)
    WHERE _array_index IS NULL AND _id_parent IS NULL;

-- 2. Nested: уникальность в пределах родителя (вложенный объект).
CREATE UNIQUE INDEX uq_values_nested
    ON _values (_id_structure, _id_object, _id_parent)
    WHERE _array_index IS NULL AND _id_parent IS NOT NULL;

-- 3. Array: уникальность позиции элемента в массиве.
CREATE UNIQUE INDEX uq_values_array
    ON _values (_id_structure, _id_object, _array_index)
    WHERE _array_index IS NOT NULL;
```

Каждый индекс:
- покрывает **непересекающееся** подмножество строк (по `_array_index IS NULL/NOT NULL` и `_id_parent IS NULL/NOT NULL`) — вместе они полны и не конфликтуют;
- даёт **корректную** уникальность для своей формы (скаляр — один на объект; массив — один на позицию; nested — один на родителя);
- является **тоньше** монолита: каждый видит только свои строки, обслуживает upsert-путь (`INSERT ... ON CONFLICT`) для этой формы.

> [!tip] Раздельные UNIQUE = раздельные ON CONFLICT таргеты
> Upsert значения адресует конкретный partial UNIQUE как conflict target: `ON CONFLICT (_id_structure, _id_object) WHERE _array_index IS NULL AND _id_parent IS NULL DO UPDATE ...`. Монолитный constraint такой точечный upsert по форме не дал бы.

---

## Маппинг на SQL Server

Аналоги существуют, но называются иначе:

| PostgreSQL | SQL Server | Назначение |
|------------|------------|------------|
| Partial index (`... WHERE _Long IS NOT NULL`) | **Filtered index** (`CREATE INDEX ... WHERE _Long IS NOT NULL`) | Тип-сегрегация, отсев NULL, обход лимита |
| `INCLUDE (_id_object)` | `INCLUDE (_id_object)` | Covering / index-only scan (идентичный синтаксис) |
| UNIQUE partial | **Unique filtered index** | Scalar / nested / array уникальность |
| GIN full-text / `pg_trgm` для длинных строк | **Full-Text Search** (`CONTAINS` / `FREETEXT`) | Поиск по длинным `_String` вне btree |

```sql
-- SQL Server: filtered index — прямой аналог partial
CREATE NONCLUSTERED INDEX IX_values_Long
    ON _values (_id_structure, _Long)
    INCLUDE (_id_object)
    WHERE _Long IS NOT NULL;

-- Unique filtered для scalar-формы
CREATE UNIQUE NONCLUSTERED INDEX UQ_values_scalar
    ON _values (_id_structure, _id_object)
    WHERE _array_index IS NULL AND _id_parent IS NULL;
```

> [!warning] Отличия SQL Server, о которых легко споткнуться
> Btree-лимит ключа в SQL Server — 1700 байт (nonclustered), не ~2700; порог длины `_String` подбирай под него. Filtered index требует определённых `SET`-опций (`QUOTED_IDENTIFIER ON` и т.д.) при INSERT/UPDATE затронутой таблицы — иначе DML падает. Для длинных `_String` полноценный аналог trigram-поиска — именно Full-Text Search, а не filtered btree.

---

## Сводка стратегии

```
_values (_id_structure, _id_object, _array_index, _id_parent, _Long, _String, _DateTimeOffset, _Boolean)
│
├── Поиск/facet по значению (любой класс, любой тенант)
│   └── Фиксированный набор covering partial-индексов, лидирует _id_structure:
│       ix_values_long  (_id_structure, _Long)            INCLUDE (_id_object) WHERE _Long IS NOT NULL
│       ix_values_dto   (_id_structure, _DateTimeOffset)  INCLUDE (_id_object) WHERE _DateTimeOffset IS NOT NULL
│       ix_values_bool  (_id_structure, _Boolean)         INCLUDE (_id_object) WHERE _Boolean IS NOT NULL
│       ix_values_string(_id_structure, _String)          INCLUDE (_id_object) WHERE _String IS NOT NULL AND length(_String) < 2000
│
├── Поиск по длинным строкам
│   └── Отдельный GIN full-text / pg_trgm (НЕ btree)
│
└── Уникальность по форме хранения
    ├── scalar  → UNIQUE (_id_structure, _id_object)               WHERE _array_index IS NULL AND _id_parent IS NULL
    ├── nested  → UNIQUE (_id_structure, _id_object, _id_parent)   WHERE _array_index IS NULL AND _id_parent IS NOT NULL
    └── array   → UNIQUE (_id_structure, _id_object, _array_index) WHERE _array_index IS NOT NULL
```

Принцип, который связывает всё: в EAV "имя атрибута" — это данные (`_id_structure`), поэтому фиксированный набор индексов, лидирующий по `_id_structure`, ведёт себя как бесконечно много логических индексов, отсекает ~99.75% чужих строк до сравнения значения, через `INCLUDE` отдаёт результат без heap-fetch, через partial по типу/длине обходит btree-лимит и NULL-bloat, а три UNIQUE partial аккуратно разводят scalar/nested/array.

---

## Best Practices

- **Лидируй `_id_structure`** в каждом типовом индексе — это equality-фильтр, отсекающий ~99.75% строк до range по значению.
- **Один фиксированный набор индексов из DDL**, никаких per-class индексов и рантайм-DDL под нагрузкой.
- **`INCLUDE (_id_object)`** на каждом facet-индексе — index-only scan вместо heap-fetch.
- **Partial `WHERE <col> IS NOT NULL`** на каждом типе — индекс хранит только свои строки, без NULL-bloat.
- **Порог длины для `_String`** (`length(_String) < N`) — обход btree-лимита; длинные строки уходят в GIN/full-text.
- **Три UNIQUE partial** (scalar / nested / array), не монолит — корректная уникальность по форме + точечный `ON CONFLICT`.
- **Следи за autovacuum** — index-only scan жив только при свежем visibility map (см. [[postgresql-deep]]).
- **`CREATE INDEX CONCURRENTLY`** при добавлении индексов в прод (см. [[indexes-deep]]).
- **SQL Server**: filtered index + filtered unique + Full-Text Search; помни про лимит ключа 1700 байт и `SET`-опции.

---

## См. также

- [[indexes-deep|Indexes Deep]] — composite-порядок, partial/covering, leftmost-prefix, EXPLAIN
- [[postgresql-deep|PostgreSQL Deep]] — JSONB-альтернатива, full-text, VACUUM/visibility map, partial-индексы
- [[optimization|SQL Optimization]] — планы выполнения и join-алгоритмы

## Reading list

- **Use The Index, Luke!** — use-the-index-luke.com (partial и covering индексы)
- **PostgreSQL Documentation — Partial Indexes / Index-Only Scans** — postgresql.org/docs/current/indexes-partial.html
- **SQL Server — Filtered Indexes** — learn.microsoft.com (Create Filtered Indexes)
- **The Anti-Pattern of EAV (and how to make it fast)** — разборы EAV-индексации в Crunchy Data / pganalyze блогах
