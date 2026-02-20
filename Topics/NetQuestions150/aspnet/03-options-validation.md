# Options Pattern и Validation

## Options Pattern

Типизированный доступ к конфигурации. `Configure<T>(configuration.GetSection("Section"))`, инжект `IOptions<T>`.

| Тип | Lifetime | Когда |
|-----|----------|-------|
| IOptions<T> | Singleton, кэш при первом обращении | Статичная конфигурация |
| IOptionsSnapshot<T> | Scoped, перечитывает при каждом запросе | Конфиг с reload |
| IOptionsMonitor<T> | Singleton, OnChange | Фоновые сервисы, Feature Flags |

**Валидация**: IValidateOptions<T>, ValidateOnStart(), ValidateDataAnnotations(). Ошибка при старте → приложение не запустится.

---

## Validation

**DataAnnotations** — атрибуты на модели, быстро для простых правил. **FluentValidation** — отдельный Validator, сложная логика, MustAsync, кросс-полевые проверки.

**Action Filters** — валидация до action, логирование, измерение времени. `IActionFilter` / `IAsyncActionFilter`. ModelState проверка: `if (!ModelState.IsValid) return BadRequest(ModelState)`.
