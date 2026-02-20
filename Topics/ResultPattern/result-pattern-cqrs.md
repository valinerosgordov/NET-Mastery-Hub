# Result/Option и MediatR

## Result&lt;T&gt;

Ошибки как значения, не исключения. Railway Oriented Programming — цепочка операций, каждая возвращает Result.

```csharp
public record Result<T>
{
    public bool IsSuccess { get; }
    public T? Value { get; }
    public Error? Error { get; }
    public static Result<T> Ok(T value) => new(true, value, null);
    public static Result<T> Fail(Error error) => new(false, default, error);
}
```

**Библиотеки**: FluentResults, OneOf, Result.Net. Или свой минимальный тип.

---

## Option&lt;T&gt;

Значение может отсутствовать. Альтернатива null. Match, Map, Bind.

---

## MediatR

Mediator pattern. IRequest&lt;TResponse&gt;, IRequestHandler. Декомпозиция: один handler на use case. CQRS: Command vs Query.

```csharp
public record GetUserQuery(int Id) : IRequest<Result<User?>>;
public class GetUserHandler : IRequestHandler<GetUserQuery, Result<User?>>
{
    public async Task<Result<User?>> Handle(GetUserQuery request, CancellationToken ct)
        => await _repo.GetAsync(request.Id, ct) is { } u ? Result.Ok(u) : Result.Fail(Error.NotFound());
}
```

---

## Vertical Slice Architecture

Один срез = одна фича. Handler + Validator + DTO в одной папке. Минимум shared кода между срезами.

---

## См. также

- [[Architecture/architecture-tutorial|Архитектуры]]
- [[Topics/Snippets/snippet-mediatr-handlers|MediatR Handlers]]
- [[Topics/Snippets/snippet-result-pattern|Result Usage]]
- [[dotnet-knowledge-base|.NET Knowledge Base]]
