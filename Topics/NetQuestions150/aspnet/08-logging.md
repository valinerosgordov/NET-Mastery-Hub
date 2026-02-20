# Logging и безопасность

## Конфигурация

AddConsole, AddDebug, AddConfiguration(GetSection("Logging")). appsettings.json: LogLevel по namespace (Default, Microsoft, MyApp). Serilog, NLog — сторонние провайдеры.

---

## Structured Logging

Логи как пары ключ-значение: `logger.LogInformation("User {UserId} ordered {OrderId}", userId, orderId)`. Поиск по полям в ELK, Seq, Application Insights.

**Request/Response logging**: middleware с EnableBuffering для body, подмена Response.Body на MemoryStream. Не логировать чувствительные данные.

---

## Sensitive Data и Exceptions

Маскирование, redaction (Serilog DestructuringPolicy), scrubbing, политики. GDPR: шифрование, ограничение доступа, retention.

**Глобальная обработка исключений**: UseExceptionHandler("/error") или кастомный middleware с try-catch. Exception Filter (IExceptionFilter) для MVC. ProblemDetails в ответе.
