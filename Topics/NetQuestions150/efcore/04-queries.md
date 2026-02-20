# Запросы

## Raw SQL

FromSqlRaw, FromSqlInterpolated (параметризация). ExecuteSqlRawAsync для команд без возврата. Параметры — защита от SQL injection.

---

## First vs Single, Client vs Server

**FirstOrDefault** — TOP 1, возвращает первый. **SingleOrDefault** — ожидается 0 или 1, иначе исключение.

**Server-side** — EF переводит в SQL. **Client-side** — загрузка в память, выполнение в приложении. Неподдерживаемые выражения → предупреждение в логах. Избегать.

---

## Expression Trees

LINQ компилируется в Expression<T>. EF обходит дерево и строит SQL. Неподдерживаемые операции (вызовы методов) → client-side evaluation.
