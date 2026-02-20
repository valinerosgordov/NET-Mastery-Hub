# Performance

## N+1

Один запрос за список + N запросов при обращении к навигации. Решения: Include, проекция в DTO (Select с JOIN), explicit loading, batch loading. Проекция — часто лучший выбор.

---

## Оптимизация reads

AsNoTracking, проекция в DTO, AsSplitQuery при нескольких коллекциях, индексы, compiled queries, кэширование, read replicas (CQRS), пагинация.

---

## Compiled Queries

EF.CompileQuery — компиляция один раз, повторные вызовы используют план. Параметры — примитивы. Эффект при частом выполнении одного запроса.

---

## AutoInclude

Навигация включается автоматически. Проблемы: лишние данные, Cartesian explosion, конфликты с фильтрами. Явный Include предсказуемее.
