---
tags: [aspnet, graphql, hotchocolate, dataloader, federation, schema-first, code-first]
level: Senior
date: 2026-04-30
---

# GraphQL с HotChocolate

> Полный гайд по GraphQL в .NET. Закрывает: HotChocolate v15 (актуальная версия 2026), code-first vs schema-first, queries/mutations/subscriptions, DataLoader (N+1 решение), authentication/authorization, paging, filtering/sorting, federation для микросервисов, performance оптимизации, security, testing.

---

## Что это, зачем и когда

### Что такое GraphQL?
**Query language для API** — клиент сам описывает что ему нужно, сервер возвращает только запрошенные поля. Один endpoint вместо REST с десятками routes.

**Аналогия:** Ресторан. REST — фиксированное меню (получаешь ВСЕ что в `/orders/123`). GraphQL — конструктор пиццы (выбираешь ингредиенты — `{ order(id: 123) { items { name } } }`).

### REST vs GraphQL

| | REST | GraphQL |
|--|------|---------|
| Endpoints | `/orders/{id}`, `/orders/{id}/items` | Один `/graphql` |
| Over-fetch | `GET /orders/123` возвращает 50 полей | Клиент берёт только нужное |
| Under-fetch | Нужно 3 round-trips для order + items + customer | Один запрос с nested |
| Schema | OpenAPI/Swagger | Strongly typed schema встроена |
| Caching | HTTP caching из коробки | Сложнее (POST + dynamic queries) |
| Versioning | `/v1/`, `/v2/` | Schema evolution через deprecation |
| Mobile-friendly | Tons of round-trips | Один query — оптимально для мобилы |
| Аналитика | По endpoint | По type/field — точнее |

### Когда GraphQL

✅ **Хорошо для:**
- Mobile / SPA — клиент знает что нужно, разные клиенты разные данные
- BFF (Backend for Frontend) — агрегация данных из множества микросервисов
- Сложные иерархические данные (graphs)
- Много типов клиентов с разными требованиями

❌ **Не подходит:**
- Простой CRUD — REST проще
- File upload, streaming — GraphQL не для этого
- Highly cacheable read-heavy APIs — REST с CDN лучше
- Public API — REST + OpenAPI понятнее

### Библиотеки для .NET

| | HotChocolate | GraphQL.NET |
|--|-------------|-------------|
| Maintainer | ChilliCream | community |
| Version 2026 | v15 | v8 |
| Recommended | ✅ Default choice | Legacy projects |
| Performance | Лучше | Ниже |
| Code-first DX | Отличный | OK |
| Federation | Apollo Federation v2 | Limited |
| Tooling | Banana Cake Pop IDE, CLI | Без IDE |

**Выбор: HotChocolate v15** для всех новых проектов.

---

## 1. Установка и базовый setup

```xml
<!-- Directory.Packages.props -->
<PackageVersion Include="HotChocolate.AspNetCore" Version="15.0.0" />
<PackageVersion Include="HotChocolate.Data" Version="15.0.0" />
<PackageVersion Include="HotChocolate.Data.EntityFramework" Version="15.0.0" />
<PackageVersion Include="HotChocolate.Subscriptions.Redis" Version="15.0.0" />
```

```xml
<!-- MyApp.Api.csproj -->
<ItemGroup>
  <PackageReference Include="HotChocolate.AspNetCore" />
  <PackageReference Include="HotChocolate.Data" />
  <PackageReference Include="HotChocolate.Data.EntityFramework" />
</ItemGroup>
```

```csharp
// Program.cs
var builder = WebApplication.CreateBuilder(args);

builder.Services
    .AddGraphQLServer()
    .AddQueryType<Query>()
    .AddMutationType<Mutation>()
    .AddSubscriptionType<Subscription>();

var app = builder.Build();

app.MapGraphQL("/graphql");  // GraphQL endpoint
app.RunWithGraphQLCommands(args);  // Banana Cake Pop в browser
```

```bash
dotnet run
# http://localhost:5000/graphql — Banana Cake Pop IDE

```

---

## 2. Code-First — основной подход

### Простой Query

```csharp
public class Query
{
    public string Hello() => "World";
    
    public Author GetAuthor(int id) => 
        new Author { Id = id, Name = "John Doe" };
    
    public IEnumerable<Book> GetBooks() => 
        new[] {
            new Book { Title = "GraphQL in .NET", AuthorId = 1 },
            new Book { Title = "Clean Architecture", AuthorId = 2 }
        };
}

public class Author
{
    public int Id { get; set; }
    public string Name { get; set; } = "";
}

public class Book
{
    public string Title { get; set; } = "";
    public int AuthorId { get; set; }
}
```

GraphQL schema генерируется автоматически:

```graphql
type Query {
  hello: String!
  author(id: Int!): Author!
  books: [Book!]!
}

type Author {
  id: Int!
  name: String!
}

type Book {
  title: String!
  authorId: Int!
}
```

### Запрос из клиента

```graphql
query {
  author(id: 1) {
    id
    name
  }
  books {
    title
  }
}
```

```json
{
  "data": {
    "author": { "id": 1, "name": "John Doe" },
    "books": [
      { "title": "GraphQL in .NET" },
      { "title": "Clean Architecture" }
    ]
  }
}
```

---

## 3. Resolvers — как работают

В GraphQL — каждое поле имеет свой **resolver**. HotChocolate автоматически генерирует resolvers из properties и methods.

### Field resolver на классе типа

```csharp
public class Book
{
    public int Id { get; set; }
    public string Title { get; set; } = "";
    public int AuthorId { get; set; }
    
    // Resolver для поля author — отдельный SQL/API call
    public async Task<Author?> GetAuthorAsync([Service] AppDbContext db) =>
        await db.Authors.FindAsync(AuthorId);
}
```

### Resolver через extension class (preferred — separation)

```csharp
public class Book
{
    public int Id { get; set; }
    public string Title { get; set; } = "";
    public int AuthorId { get; set; }
}

[ExtendObjectType<Book>]
public class BookResolvers
{
    public async Task<Author?> GetAuthor(
        [Parent] Book book,
        [Service] AppDbContext db) =>
        await db.Authors.FindAsync(book.AuthorId);
    
    public async Task<int> GetReviewCount(
        [Parent] Book book,
        [Service] AppDbContext db) =>
        await db.Reviews.CountAsync(r => r.BookId == book.Id);
}

// Регистрация
builder.Services
    .AddGraphQLServer()
    .AddQueryType<Query>()
    .AddTypeExtension<BookResolvers>();
```

### Зависимости в resolver

```csharp
public class Query
{
    // [Service] — DI injection
    public async Task<List<Book>> GetBooks(
        [Service] AppDbContext db,
        [Service] ILogger<Query> log)
    {
        log.LogInformation("Fetching books");
        return await db.Books.ToListAsync();
    }
    
    // Параметры query
    public async Task<Book?> GetBook(
        int id,
        [Service] AppDbContext db) =>
        await db.Books.FindAsync(id);
}
```

```graphql
query {
  book(id: 1) { title }
}
```

---

## 4. Mutations — изменение данных

```csharp
public class Mutation
{
    public async Task<Book> AddBook(
        AddBookInput input,
        [Service] AppDbContext db,
        CancellationToken ct)
    {
        var book = new Book
        {
            Title = input.Title,
            AuthorId = input.AuthorId
        };
        
        db.Books.Add(book);
        await db.SaveChangesAsync(ct);
        return book;
    }
    
    public async Task<UpdateBookPayload> UpdateBook(
        UpdateBookInput input,
        [Service] AppDbContext db,
        CancellationToken ct)
    {
        var book = await db.Books.FindAsync(input.Id);
        if (book is null)
            return new UpdateBookPayload(null, "Book not found");
        
        book.Title = input.Title;
        await db.SaveChangesAsync(ct);
        return new UpdateBookPayload(book, null);
    }
}

public record AddBookInput(string Title, int AuthorId);
public record UpdateBookInput(int Id, string Title);
public record UpdateBookPayload(Book? Book, string? Error);
```

```graphql
mutation {
  addBook(input: { title: "GraphQL", authorId: 1 }) {
    id
    title
  }
  
  updateBook(input: { id: 1, title: "GraphQL Updated" }) {
    book { id title }
    error
  }
}
```

---

## 5. DataLoader — решение N+1

### Проблема

```csharp
// Запрос
{
  books {
    title
    author { name }
  }
}

// 100 books → resolver author вызовется 100 раз → 100 SELECT'ов!
// Это и есть N+1 проблема.
```

### DataLoader — batch + cache

```csharp
public class AuthorByIdDataLoader : BatchDataLoader<int, Author>
{
    private readonly IDbContextFactory<AppDbContext> _dbFactory;
    
    public AuthorByIdDataLoader(
        IDbContextFactory<AppDbContext> dbFactory,
        IBatchScheduler batchScheduler,
        DataLoaderOptions? options = null)
        : base(batchScheduler, options)
    {
        _dbFactory = dbFactory;
    }
    
    protected override async Task<IReadOnlyDictionary<int, Author>> LoadBatchAsync(
        IReadOnlyList<int> keys,
        CancellationToken ct)
    {
        await using var db = await _dbFactory.CreateDbContextAsync(ct);
        return await db.Authors
            .Where(a => keys.Contains(a.Id))
            .ToDictionaryAsync(a => a.Id, ct);
    }
}

// В resolver
[ExtendObjectType<Book>]
public class BookResolvers
{
    public async Task<Author?> GetAuthor(
        [Parent] Book book,
        AuthorByIdDataLoader loader,
        CancellationToken ct) =>
        await loader.LoadAsync(book.AuthorId, ct);
}

// Регистрация
builder.Services.AddGraphQLServer()
    .AddDataLoader<AuthorByIdDataLoader>();
```

### Как работает

```
Request: { books { author { name } } }
  resolver Book.author called for each book
  ↓
  DataLoader.LoadAsync(1) → queue
  DataLoader.LoadAsync(2) → queue
  ...
  DataLoader.LoadAsync(100) → queue
  ↓
  HotChocolate batches → один SQL: WHERE Id IN (1, 2, ..., 100)
  ↓
  Возвращает Authors всем resolver'ам
```

**Результат:** 1 SQL вместо 100. Решение N+1 встроено в GraphQL.

### .NET 8+ — Source-generated DataLoader (HotChocolate 14+)

```csharp
// Теперь — просто атрибут, генерируется код
public class AuthorDataLoaders
{
    [DataLoader]
    public static async Task<IReadOnlyDictionary<int, Author>> AuthorByIdAsync(
        IReadOnlyList<int> keys,
        AppDbContext db,
        CancellationToken ct) =>
        await db.Authors
            .Where(a => keys.Contains(a.Id))
            .ToDictionaryAsync(a => a.Id, ct);
}

// Использование в resolver
public Task<Author?> GetAuthor(
    [Parent] Book book,
    IAuthorByIdDataLoader loader,
    CancellationToken ct) =>
    loader.LoadAsync(book.AuthorId, ct);
```

Boilerplate уменьшен в разы.

### GroupedDataLoader — для one-to-many

```csharp
[DataLoader]
public static async Task<ILookup<int, Book>> BooksByAuthorAsync(
    IReadOnlyList<int> authorIds,
    AppDbContext db,
    CancellationToken ct)
{
    var books = await db.Books
        .Where(b => authorIds.Contains(b.AuthorId))
        .ToListAsync(ct);
    
    return books.ToLookup(b => b.AuthorId);
}

// В resolver
public async Task<IEnumerable<Book>> GetBooks(
    [Parent] Author author,
    IBooksByAuthorDataLoader loader,
    CancellationToken ct) =>
    await loader.LoadAsync(author.Id, ct) ?? Enumerable.Empty<Book>();
```

---

## 6. Filtering, Sorting, Paging

### Cursor-based pagination

```csharp
public class Query
{
    [UsePaging]
    [UseFiltering]
    [UseSorting]
    public IQueryable<Book> GetBooks([Service] AppDbContext db) =>
        db.Books.AsQueryable();
}

builder.Services.AddGraphQLServer()
    .AddQueryType<Query>()
    .AddFiltering()
    .AddSorting()
    .AddProjections();
```

```graphql
query {
  books(
    first: 10
    where: { title: { contains: "GraphQL" } }
    order: { publishedAt: DESC }
  ) {
    edges {
      cursor
      node {
        id
        title
      }
    }
    pageInfo {
      hasNextPage
      endCursor
    }
  }
}
```

### Offset pagination

```csharp
[UseOffsetPaging]
public IQueryable<Book> GetBooks([Service] AppDbContext db) =>
    db.Books.AsQueryable();
```

```graphql
{
  books(skip: 20, take: 10) {
    items { title }
    totalCount
    pageInfo { hasNextPage }
  }
}
```

### Filtering DSL

HotChocolate генерирует filter input automatically:

```graphql
input BookFilterInput {
  and: [BookFilterInput!]
  or: [BookFilterInput!]
  id: IntOperationFilterInput
  title: StringOperationFilterInput
  publishedAt: DateTimeOperationFilterInput
}

input StringOperationFilterInput {
  eq: String
  neq: String
  contains: String
  startsWith: String
  endsWith: String
  in: [String!]
  nin: [String!]
}
```

```graphql
{
  books(where: {
    and: [
      { title: { contains: "GraphQL" } }
      { publishedAt: { gt: "2024-01-01" } }
    ]
  }) {
    nodes { title }
  }
}
```

### Custom filter

```csharp
public class BookFilterType : FilterInputType<Book>
{
    protected override void Configure(IFilterInputTypeDescriptor<Book> descriptor)
    {
        descriptor.BindFieldsExplicitly();
        descriptor.Field(b => b.Title);
        descriptor.Field(b => b.PublishedAt);
        // Не позволяем фильтровать по AuthorId напрямую — отдельное поле
    }
}

[UseFiltering<BookFilterType>]
public IQueryable<Book> GetBooks(...) => ...;
```

---

## 7. Subscriptions — real-time

### Setup

```csharp
builder.Services
    .AddGraphQLServer()
    .AddQueryType<Query>()
    .AddSubscriptionType<Subscription>()
    .AddInMemorySubscriptions();  // или Redis для distributed

// WebSocket support
app.UseWebSockets();
app.MapGraphQL();
```

### Subscription handler

```csharp
public class Subscription
{
    [Subscribe]
    [Topic("BookAdded")]
    public Book OnBookAdded([EventMessage] Book book) => book;
}

public class Mutation
{
    public async Task<Book> AddBook(
        AddBookInput input,
        [Service] AppDbContext db,
        [Service] ITopicEventSender sender,
        CancellationToken ct)
    {
        var book = new Book { Title = input.Title, AuthorId = input.AuthorId };
        db.Books.Add(book);
        await db.SaveChangesAsync(ct);
        
        // Push event subscribers
        await sender.SendAsync("BookAdded", book, ct);
        return book;
    }
}
```

```graphql
subscription {
  onBookAdded {
    id
    title
  }
}
```

### Redis backplane — для multiple replicas

```csharp
builder.Services
    .AddGraphQLServer()
    .AddQueryType<Query>()
    .AddSubscriptionType<Subscription>()
    .AddRedisSubscriptions((sp) => 
        ConnectionMultiplexer.Connect(redisConnString));
```

Каждый pod может publish events, все pods доставят клиентам.

---

## 8. Authentication & Authorization

### Authentication — через ASP.NET Core auth

```csharp
builder.Services
    .AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
    .AddJwtBearer(options => /* ... */);

builder.Services.AddAuthorization();

builder.Services.AddGraphQLServer()
    .AddAuthorization()
    .AddQueryType<Query>();

// В Program.cs
app.UseAuthentication();
app.UseAuthorization();
app.MapGraphQL();
```

### Authorization на resolver

```csharp
public class Query
{
    [Authorize]
    public IQueryable<Book> GetBooks([Service] AppDbContext db) => 
        db.Books;
    
    [Authorize(Roles = new[] { "Admin" })]
    public IQueryable<User> GetAllUsers([Service] AppDbContext db) =>
        db.Users;
    
    [Authorize(Policy = "PremiumUser")]
    public IQueryable<PremiumContent> GetPremium(...) => ...;
}
```

### Authorization на типе / поле

```csharp
[Authorize]  // весь тип требует auth
public class User
{
    public int Id { get; set; }
    public string Name { get; set; } = "";
    
    [Authorize(Roles = new[] { "Admin" })]
    public string Email { get; set; } = "";  // только Admin видит
}
```

### Current user в resolver

```csharp
public class Query
{
    public async Task<User?> GetMe(
        ClaimsPrincipal claimsPrincipal,
        [Service] AppDbContext db,
        CancellationToken ct)
    {
        var userId = claimsPrincipal.FindFirst(ClaimTypes.NameIdentifier)?.Value;
        if (userId is null) return null;
        return await db.Users.FindAsync(Guid.Parse(userId));
    }
}
```

---

## 9. Performance — критичное

### Projections — только нужные колонки

```csharp
[UseProjection]
public IQueryable<Book> GetBooks([Service] AppDbContext db) => db.Books;
```

Если клиент запрашивает `{ books { title } }` — EF делает `SELECT title FROM Books`. Не загружает всю entity.

```csharp
// Без UseProjection
SELECT Id, Title, AuthorId, PublishedAt, Description, ... FROM Books

// С UseProjection
SELECT Title FROM Books
```

### Combine: Paging + Filtering + Sorting + Projection

```csharp
[UsePaging]
[UseProjection]
[UseFiltering]
[UseSorting]
public IQueryable<Book> GetBooks([Service] AppDbContext db) => db.Books;
```

Order matters: Paging → Projection → Filtering → Sorting.

### Query complexity / depth limit

```csharp
builder.Services.AddGraphQLServer()
    .AddCostAnalyzer(opts =>
    {
        opts.EnableEnforcement = true;
        opts.MaximumFieldCost = 1000;
    })
    .AddMaxExecutionDepthRule(maxAllowedExecutionDepth: 10)
    .AddQueryType<Query>();
```

Защита от malicious queries (типа nested 1000 уровней).

### Persisted queries

Клиент сохраняет query на сервере → отправляет hash, не сам query (меньше трафика, бойцовская стандартная безопасность).

```csharp
builder.Services.AddGraphQLServer()
    .AddInMemoryQueryStorage()  // или Redis
    .UseAutomaticPersistedQueryPipeline();
```

### Response compression

```csharp
builder.Services.AddResponseCompression(opts =>
{
    opts.MimeTypes = new[] { "application/graphql-response+json", "application/json" };
});

app.UseResponseCompression();
```

---

## 10. Federation — для микросервисов

Apollo Federation v2 — спецификация для разделения GraphQL по сервисам.

### Subgraph 1 — Users service

```csharp
[Key("id")]
public class User
{
    public int Id { get; set; }
    public string Name { get; set; } = "";
}

builder.Services
    .AddGraphQLServer()
    .AddApolloFederation(FederationVersion.Federation20)
    .AddQueryType<UserQuery>();

public class UserQuery
{
    public Task<User?> GetUser(int id, ...) => ...;
}
```

### Subgraph 2 — Orders service

```csharp
[Key("id")]
public class Order
{
    public int Id { get; set; }
    public int UserId { get; set; }
    
    // Reference на User из другого subgraph
    public User User => new() { Id = UserId };
}

[ReferenceResolver]
public static class UserResolvers
{
    public static User? ResolveUserReference(int id) => 
        new User { Id = id };  // только Id, federation gateway соберёт остальное
}
```

### Apollo Router / Cosmo Router

Gateway собирает schemas всех subgraphs в единый supergraph:

```graphql
# User service знает: type User { id, name }
# Order service знает: type Order { id, user: User }

# Client query идёт через gateway:
{
  order(id: 1) {
    id
    user {
      name      # ← gateway понимает что это в users service
    }
  }
}
```

См. [Distributed Systems](../Architecture/distributed-systems.md).

---

## 11. Schema-first vs Code-first

### Schema-first

Пишем `.graphql` файл со schema → инструмент генерирует C# код.

```graphql
# schema.graphql
type Query {
  books: [Book!]!
  book(id: Int!): Book
}

type Book {
  id: Int!
  title: String!
  author: Author!
}

type Author {
  id: Int!
  name: String!
  books: [Book!]!
}
```

**Когда:**
- Schema создаётся отдельно (продукт-менеджер, frontend, дизайнер схемы)
- Команда уже использует GraphQL pattern
- Cross-language (один schema для .NET и Node)

### Code-first (HotChocolate default)

C# → schema generated. Default подход с .NET.

**Когда:**
- Tight coupling C# модели и schema
- Не нужна schema "first-class citizen"
- Менее boilerplate

> [!info] HotChocolate поддерживает оба
> Можно смешивать. Но новые проекты — code-first.

---

## 12. Testing

### Unit test resolver

```csharp
public class QueryTests
{
    [Fact]
    public async Task Get_book_returns_book()
    {
        // Arrange
        var dbContext = CreateInMemoryContext();
        dbContext.Books.Add(new Book { Id = 1, Title = "Test" });
        await dbContext.SaveChangesAsync();
        
        var query = new Query();
        
        // Act
        var book = await query.GetBookAsync(1, dbContext, CancellationToken.None);
        
        // Assert
        book.ShouldNotBeNull();
        book.Title.ShouldBe("Test");
    }
}
```

### Integration test через TestServer

```csharp
public class GraphQLIntegrationTests : IClassFixture<WebApplicationFactory<Program>>
{
    private readonly WebApplicationFactory<Program> _factory;
    
    public GraphQLIntegrationTests(WebApplicationFactory<Program> factory) => _factory = factory;
    
    [Fact]
    public async Task Query_books_returns_data()
    {
        var client = _factory.CreateClient();
        
        var request = new { query = "{ books { title } }" };
        var response = await client.PostAsJsonAsync("/graphql", request);
        
        response.EnsureSuccessStatusCode();
        var json = await response.Content.ReadAsStringAsync();
        json.ShouldContain("\"books\"");
    }
}
```

### Schema snapshot testing

```csharp
[Fact]
public async Task Schema_should_match_snapshot()
{
    var schema = await new ServiceCollection()
        .AddGraphQLServer()
        .AddQueryType<Query>()
        .BuildSchemaAsync();
    
    var sdl = schema.ToString();
    
    // Snapshot — сравнение с baseline
    Snapshot.Match(sdl);  // Verify NuGet
}
```

Если кто-то случайно добавил поле / тип в schema → тест падает → review.

---

## 13. Common Pitfalls

### 1. N+1 без DataLoader

Resolver `GetAuthor` вызывается для каждой book → 100 SQL.

**Лечение:** DataLoader.

### 2. Возврат IQueryable без `[UseProjection]`

```csharp
// ❌ Загружает все колонки даже если client просит только title
public IQueryable<Book> GetBooks(...) => db.Books;
```

**Лечение:** `[UseProjection]`.

### 3. Tracking enabled для read-only queries

EF tracking + GraphQL = большой overhead.

```csharp
// ✅ AsNoTracking
public IQueryable<Book> GetBooks([Service] AppDbContext db) => 
    db.Books.AsNoTracking();
```

### 4. Sub-resolvers без CancellationToken

Если client закрывает connection — полезно отменять in-flight resolvers.

```csharp
// ✅ CancellationToken пробрасывается
public async Task<Author?> GetAuthor(
    [Parent] Book book,
    AuthorByIdDataLoader loader,
    CancellationToken ct) =>
    await loader.LoadAsync(book.AuthorId, ct);
```

### 5. Mutation возвращает только bool / int

GraphQL best practice — возвращать **изменённый объект** в response, чтобы client мог update cache.

```csharp
// ❌
public async Task<bool> AddBook(...) { ... }

// ✅
public async Task<Book> AddBook(...) { ... }
```

### 6. Subscription без backplane при multiple replicas

Pod-A subscribed к event, Mutation на Pod-B → Pod-A не получит.

**Лечение:** Redis subscriptions.

### 7. Запросы через POST + body не cacheable

REST GET cacheable HTTP-кэшем. GraphQL POST — нет.

**Лечение:**
- Persisted queries — клиент шлёт hash, server lookup
- HTTP GET для queries (без mutations) — поддерживается HotChocolate

### 8. Federation с разными версиями HotChocolate

Subgraphs должны использовать совместимые versions. Mismatch → schema composition errors на gateway.

### 9. Большой query depth — DoS

```graphql
# Malicious nested query — exponential complexity
{ user { friends { friends { friends { friends { name } } } } } }
```

**Лечение:** `AddMaxExecutionDepthRule(10)` + cost analyzer.

### 10. Returning errors через exceptions

```csharp
// ❌ Throw — клиент видит generic "Internal server error"
public Book GetBook(int id, ...) {
    var book = db.Books.Find(id);
    if (book == null) throw new Exception("Not found");
    return book;
}

// ✅ Union types или payloads с error info
public record GetBookResult(Book? Book, string? Error);

public GetBookResult GetBook(int id, ...) {
    var book = db.Books.Find(id);
    return book is null
        ? new GetBookResult(null, "Book not found")
        : new GetBookResult(book, null);
}
```

---

## 14. Best Practices

- **HotChocolate v15** для всех новых проектов
- **Code-first** (если нет причин для schema-first)
- **DataLoader для всего** что может быть N+1 (foreign keys, references)
- **`[UsePaging]` + `[UseProjection]` + `[UseFiltering]` + `[UseSorting]`** для list queries
- **`[Authorize]` явно** на каждом resolver — fail closed
- **CancellationToken** в каждый async resolver
- **DbContextFactory** для DataLoader (избегаем shared DbContext между concurrent resolvers)
- **Redis subscriptions** для multi-replica
- **Cost analyzer + max depth** — защита от DoS
- **Persisted queries** — для production mobile/web clients
- **Snapshot test schema** — alert на breaking changes
- **OpenTelemetry tracing** — видно какой field тормозит
- **AsNoTracking** для EF queries в resolver

---

## 15. GraphQL vs REST — выбор

```
Маленький monolith с CRUD?           → REST
Mobile app с разными screens?        → GraphQL
BFF для микросервисов?               → GraphQL
Public API для разработчиков?        → REST + OpenAPI
File upload интенсивно?              → REST
Real-time с complex schema?          → GraphQL subscriptions
```

Можно смешивать: REST для CRUD + GraphQL для complex queries.

---

## См. также

- [API Design](api-design.md) — REST patterns
- [SignalR](signalr.md) — alternative для real-time
- [Distributed Systems](../Architecture/distributed-systems.md) — federation
- [EF Core Basics & Tracking](../EFCore/basics-tracking.md) — projection, tracking
- [Authentication & Security](auth-security.md) — JWT для GraphQL
- [Caching](caching.md) — для persisted queries
- [Resilience](resilience.md) — retry, timeout

## Reading list

- **HotChocolate docs** — chillicream.com/docs/hotchocolate
- **Microsoft Docs — GraphQL in .NET** — learn.microsoft.com (раздел aspire/graphql)
- **GraphQL specification** — spec.graphql.org
- **Apollo Federation v2** — apollographql.com/docs/federation
- **Marc-André Giroux — Production Ready GraphQL** (книга)
- **Banana Cake Pop** — chillicream.com/products/bananacakepop (GraphQL IDE)
- **Cosmo Router** — wundergraph.com (alternative к Apollo Router)
- **The Guild — graphql tools** — the-guild.dev
