---
tags: [crud, minimal-api, handlers, ef-core, dto, full-example]
level: Senior
---

# CRUD — полный пример от Endpoint до БД

## Что это, зачем и когда

### Что такое CRUD?
**Create, Read, Update, Delete** — четыре базовые операции с данными. Любое приложение с БД — это в основе CRUD.

**Аналогия:** Записная книжка. Добавить контакт (Create), найти контакт (Read), исправить номер (Update), удалить контакт (Delete).

### Зачем полный пример?
Архитектурные файлы объясняют «как правильно», но не показывают **весь путь** от HTTP-запроса до БД и обратно для всех 4 операций. Этот файл — template, который можно копировать.

### Структура примера

```
POST   /api/products        → CreateProductHandler → Domain → EF → 201 Created
GET    /api/products/{id}   → GetProductHandler    → EF → 200 OK
GET    /api/products        → ListProductsHandler  → EF → 200 OK (с пагинацией)
PUT    /api/products/{id}   → UpdateProductHandler → Domain → EF → 200 OK
DELETE /api/products/{id}   → DeleteProductHandler → Domain → EF → 204 NoContent
```

---

## 1. Domain — Entity

```csharp
public sealed class Product : Entity<Guid>
{
    public string Name { get; private set; } = null!;
    public string Description { get; private set; } = null!;
    public decimal Price { get; private set; }
    public bool IsActive { get; private set; }

    private Product() { } // EF Core

    public static Result<Product> Create(string name, string description, decimal price)
    {
        if (string.IsNullOrWhiteSpace(name))
            return Result<Product>.Fail(Error.Validation("Product.Name", "Name is required"));

        if (price < 0)
            return Result<Product>.Fail(Error.Validation("Product.Price", "Price cannot be negative"));

        return Result<Product>.Ok(new Product
        {
            Id = Guid.NewGuid(),
            Name = name.Trim(),
            Description = description?.Trim() ?? "",
            Price = price,
            IsActive = true
        });
    }

    public Result Update(string name, string description, decimal price)
    {
        if (string.IsNullOrWhiteSpace(name))
            return Result.Fail(Error.Validation("Product.Name", "Name is required"));

        if (price < 0)
            return Result.Fail(Error.Validation("Product.Price", "Price cannot be negative"));

        Name = name.Trim();
        Description = description?.Trim() ?? "";
        Price = price;
        return Result.Ok();
    }

    public Result Deactivate()
    {
        if (!IsActive)
            return Result.Fail(Error.Validation("Product.Inactive", "Product is already inactive"));

        IsActive = false;
        return Result.Ok();
    }
}
```

---

## 2. DTOs — Request и Response

```csharp
// Request DTOs — что приходит от клиента
public sealed record CreateProductRequest(
    required string Name,
    required string Description,
    required decimal Price);

public sealed record UpdateProductRequest(
    required string Name,
    required string Description,
    required decimal Price);

// Response DTO — что отдаём клиенту
public sealed record ProductDto(
    Guid Id,
    string Name,
    string Description,
    decimal Price,
    bool IsActive);

// Пагинированный ответ
public sealed record PagedResponse<T>(
    IReadOnlyList<T> Items,
    int TotalCount,
    int Page,
    int PageSize)
{
    public bool HasNextPage => Page * PageSize < TotalCount;
}

// Маппинг — extension method, без AutoMapper
public static class ProductMappings
{
    public static ProductDto ToDto(this Product product) => new(
        product.Id,
        product.Name,
        product.Description,
        product.Price,
        product.IsActive);
}
```

---

## 3. Repository — интерфейс + реализация

```csharp
// Domain layer — интерфейс
public interface IProductRepository
{
    Task<Product?> GetByIdAsync(Guid id, CancellationToken ct);
    Task<(IReadOnlyList<Product> Items, int TotalCount)> GetPagedAsync(
        int page, int pageSize, string? search, CancellationToken ct);
    void Add(Product product);
    void Remove(Product product);
}

// Infrastructure layer — реализация
public sealed class ProductRepository(AppDbContext context) : IProductRepository
{
    public async Task<Product?> GetByIdAsync(Guid id, CancellationToken ct)
        => await context.Products.FirstOrDefaultAsync(p => p.Id == id, ct);

    public async Task<(IReadOnlyList<Product> Items, int TotalCount)> GetPagedAsync(
        int page, int pageSize, string? search, CancellationToken ct)
    {
        var query = context.Products
            .AsNoTracking()
            .Where(p => p.IsActive);

        if (!string.IsNullOrWhiteSpace(search))
            query = query.Where(p => p.Name.Contains(search));

        var totalCount = await query.CountAsync(ct);

        var items = await query
            .OrderBy(p => p.Name)
            .Skip((page - 1) * pageSize)
            .Take(pageSize)
            .ToListAsync(ct);

        return (items, totalCount);
    }

    public void Add(Product product) => context.Products.Add(product);
    public void Remove(Product product) => context.Products.Remove(product);
}
```

---

## 4. Handlers — бизнес-логика

```csharp
// CREATE
public sealed class CreateProductHandler(
    IProductRepository repository,
    IUnitOfWork unitOfWork)
{
    public async Task<Result<ProductDto>> HandleAsync(
        CreateProductRequest request, CancellationToken ct)
    {
        var productResult = Product.Create(request.Name, request.Description, request.Price);
        if (productResult.IsFailure)
            return Result<ProductDto>.Fail(productResult.Error!);

        repository.Add(productResult.Value!);
        await unitOfWork.SaveChangesAsync(ct);

        return Result<ProductDto>.Ok(productResult.Value!.ToDto());
    }
}

// GET BY ID
public sealed class GetProductHandler(IProductRepository repository)
{
    public async Task<Result<ProductDto>> HandleAsync(Guid id, CancellationToken ct)
    {
        var product = await repository.GetByIdAsync(id, ct);
        return product is null
            ? Result<ProductDto>.Fail(Error.NotFound("Product.NotFound", $"Product {id} not found"))
            : Result<ProductDto>.Ok(product.ToDto());
    }
}

// LIST (с пагинацией и поиском)
public sealed class ListProductsHandler(IProductRepository repository)
{
    public async Task<PagedResponse<ProductDto>> HandleAsync(
        int page, int pageSize, string? search, CancellationToken ct)
    {
        var (items, totalCount) = await repository.GetPagedAsync(page, pageSize, search, ct);
        return new PagedResponse<ProductDto>(
            items.Select(p => p.ToDto()).ToList(),
            totalCount, page, pageSize);
    }
}

// UPDATE
public sealed class UpdateProductHandler(
    IProductRepository repository,
    IUnitOfWork unitOfWork)
{
    public async Task<Result<ProductDto>> HandleAsync(
        Guid id, UpdateProductRequest request, CancellationToken ct)
    {
        var product = await repository.GetByIdAsync(id, ct);
        if (product is null)
            return Result<ProductDto>.Fail(Error.NotFound("Product.NotFound", $"Product {id} not found"));

        var updateResult = product.Update(request.Name, request.Description, request.Price);
        if (updateResult.IsFailure)
            return Result<ProductDto>.Fail(updateResult.Error!);

        await unitOfWork.SaveChangesAsync(ct);
        return Result<ProductDto>.Ok(product.ToDto());
    }
}

// DELETE (soft delete через Deactivate)
public sealed class DeleteProductHandler(
    IProductRepository repository,
    IUnitOfWork unitOfWork)
{
    public async Task<Result> HandleAsync(Guid id, CancellationToken ct)
    {
        var product = await repository.GetByIdAsync(id, ct);
        if (product is null)
            return Result.Fail(Error.NotFound("Product.NotFound", $"Product {id} not found"));

        var deactivateResult = product.Deactivate();
        if (deactivateResult.IsFailure)
            return deactivateResult;

        await unitOfWork.SaveChangesAsync(ct);
        return Result.Ok();
    }
}
```

---

## 5. Endpoints — Minimal API

```csharp
public static class ProductEndpoints
{
    public static IEndpointRouteBuilder MapProductEndpoints(this IEndpointRouteBuilder app)
    {
        var group = app.MapGroup("/api/products")
            .WithTags("Products");

        // POST /api/products
        group.MapPost("/", async (
            CreateProductRequest request,
            CreateProductHandler handler,
            CancellationToken ct) =>
        {
            var result = await handler.HandleAsync(request, ct);
            return result.ToResponse(p =>
                TypedResults.Created($"/api/products/{p.Id}", p));
        });

        // GET /api/products/{id}
        group.MapGet("/{id:guid}", async (
            Guid id,
            GetProductHandler handler,
            CancellationToken ct) =>
        {
            var result = await handler.HandleAsync(id, ct);
            return result.ToResponse(TypedResults.Ok);
        });

        // GET /api/products?page=1&pageSize=20&search=phone
        group.MapGet("/", async (
            [AsParameters] ProductListQuery query,
            ListProductsHandler handler,
            CancellationToken ct) =>
        {
            var result = await handler.HandleAsync(
                query.Page ?? 1, query.PageSize ?? 20, query.Search, ct);
            return TypedResults.Ok(result);
        });

        // PUT /api/products/{id}
        group.MapPut("/{id:guid}", async (
            Guid id,
            UpdateProductRequest request,
            UpdateProductHandler handler,
            CancellationToken ct) =>
        {
            var result = await handler.HandleAsync(id, request, ct);
            return result.ToResponse(TypedResults.Ok);
        });

        // DELETE /api/products/{id}
        group.MapDelete("/{id:guid}", async (
            Guid id,
            DeleteProductHandler handler,
            CancellationToken ct) =>
        {
            var result = await handler.HandleAsync(id, ct);
            return result.Match(
                () => Results.NoContent(),
                error => error.Type == ErrorType.NotFound
                    ? Results.NotFound()
                    : Results.Problem(error.Message, statusCode: 400));
        });

        return app;
    }
}

// Query parameters для GET /api/products
public sealed record ProductListQuery(int? Page, int? PageSize, string? Search);
```

### Регистрация

```csharp
// Program.cs
builder.Services.AddScoped<IProductRepository, ProductRepository>();
builder.Services.AddScoped<CreateProductHandler>();
builder.Services.AddScoped<GetProductHandler>();
builder.Services.AddScoped<ListProductsHandler>();
builder.Services.AddScoped<UpdateProductHandler>();
builder.Services.AddScoped<DeleteProductHandler>();

app.MapProductEndpoints();
```

---

## 6. EF Core Configuration

```csharp
public sealed class ProductConfiguration : IEntityTypeConfiguration<Product>
{
    public void Configure(EntityTypeBuilder<Product> builder)
    {
        builder.HasKey(p => p.Id);

        builder.Property(p => p.Name)
            .HasMaxLength(200)
            .IsRequired();

        builder.Property(p => p.Description)
            .HasMaxLength(2000);

        builder.Property(p => p.Price)
            .HasPrecision(18, 2);

        builder.HasIndex(p => p.Name);
        builder.HasQueryFilter(p => p.IsActive); // soft delete
    }
}
```

---

## HTTP методы — когда какой

| Метод | Семантика | Идемпотентный | Тело запроса | Ответ |
|-------|-----------|---------------|-------------|-------|
| **GET** | Получить данные | Да | Нет | 200 + данные |
| **POST** | Создать ресурс | Нет | Да | 201 + Location header |
| **PUT** | Полная замена ресурса | Да | Да | 200 + обновлённые данные |
| **PATCH** | Частичное обновление | Нет | Да (diff) | 200 + обновлённые данные |
| **DELETE** | Удалить ресурс | Да | Нет | 204 No Content |

**Нюанс:** PUT идемпотентный — вызови 10 раз с одними данными, результат одинаковый. POST — нет (10 вызовов = 10 записей).

---

## См. также

- [API Design]() — Minimal API vs Controllers, OpenAPI
- [DDD на практике]() — Domain logic, Value Objects
- [EF Core Queries](efcore-queries.md) — Оптимизация запросов, пагинация
- [Result Pattern](result-pattern.md) — Result\<T\> детально
