# API: Model Binding, Controllers, Versioning

## Model Binding

Извлечение данных из route, query, form, body. Источники по приоритету: Route → Query → Form → Body. Сложные типы — из body по умолчанию.

**[FromBody], [FromQuery], [FromRoute], [FromForm], [FromHeader]** — явное указание источника.

**Custom model binder**: IModelBinder, регистрация через [ModelBinder] или IModelBinderProvider.

---

## Controllers

**ControllerBase** — API, нет View. **Controller** — MVC с View(), ViewData. Lifecycle: создание при выборе action (Scoped), Dispose после запроса. Return types: IActionResult, ActionResult<T>, конкретный тип (200 + JSON).

**Method injection**: [FromServices] в параметре action — для опциональных зависимостей.

**Validation**: ModelState.IsValid, BadRequest(ModelState). ProblemDetails (RFC 7807) для стандартного формата ошибок.

---

## Versioning, Swagger, Files

**API versioning**: Asp.Versioning.Mvc. Стратегии: URL path, query, header. Без изменения URL — HeaderApiVersionReader или QueryStringApiVersionReader.

**Swagger**: Swashbuckle.AspNetCore. XML-комментарии + GenerateDocumentationFile. ProducesResponseType, SwaggerOperation для документации.

**Файлы**: File(stream, contentType, fileName), PhysicalFile. Приём: IFormFile, IFormFileCollection. Ограничение размера — MultipartBodyLengthLimit.

---

## Deploy

Self-contained / framework-dependent publish. Docker — `FROM mcr.microsoft.com/dotnet/aspnet:8.0` + output. IIS — ASP.NET Core Module. Azure App Service, AKS, AWS.
