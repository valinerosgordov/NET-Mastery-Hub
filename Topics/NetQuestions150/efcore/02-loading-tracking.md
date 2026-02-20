# Loading и Tracking

## Eager, Explicit, Lazy

| Тип | Когда | Как |
|-----|-------|-----|
| Eager | В основном запросе | Include(), ThenInclude() |
| Explicit | По вызову | Entry(entity).Collection(...).Load() |
| Lazy | При обращении к навигации | Прокси, ILazyLoader (по умолчанию отключён) |

Eager — предсказуемо. Explicit — контроль. Lazy — риск N+1.

---

## Tracking и AsNoTracking

| Режим | Change Tracker | SaveChanges | Использование |
|-------|----------------|-------------|---------------|
| Tracked | Отслеживаются | UPDATE/DELETE | Редактирование |
| Untracked | Нет | Игнорируются | Read-only |

AsNoTracking() — для всех read-only сценариев. Меньше памяти, быстрее.

---

## Include и AsSplitQuery

Несколько Include с коллекциями → Cartesian explosion. AsSplitQuery() — несколько запросов, объединение в памяти. Trade-off: больше round-trips, меньше данных по сети.
