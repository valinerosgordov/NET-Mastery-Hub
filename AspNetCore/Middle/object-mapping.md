---
tags: [aspnetcore, mapping, automapper, mapperly, dto, source-generators, middle]
level: Middle
date: 2026-08-02
---

# Object Mapping — AutoMapper, Mapperly, Manual

> **Daily work каждого ASP.NET проекта**: DTO ↔ Domain ↔ ViewModel mapping. Closes пробел "копирую properties вручную или AutoMapper тормозит — что лучше использовать в 2026".

---

## Что это, зачем и когда

### Зачем mapping вообще

В Clean Architecture / Layered architecture **разные слои используют разные типы**:

```
Web Layer (Controllers)
   ↓ MapsToDomain
Application Layer (Commands/Queries)
   ↓ MapsToEntity
Domain Layer (Entities, Aggregates)
   ↓ MapsToReadModel
Database Layer (EF entities)
   ↓ MapsToDto
Web Layer (Response DTOs)
```

Зачем:
- **Domain encapsulation** — внутренние свойства не leakают в API
- **Versioning** — DTO v1/v2 без changes Domain
- **Security** — не возвращать sensitive fields (PasswordHash, InternalNotes)
- **Decoupling** — Domain независим от ASP.NET, frontend, DB

### Главное решение

Three approaches:
1. **Manual mapping** — handwritten code
2. **AutoMapper** — runtime reflection-based
3. **Mapperly** — source generator (.NET 6+)

Performance benchmark (typical):
```
Manual:    1.0x  (baseline, ~50 ns per object)
Mapperly:  1.0x  (compile-time generated — same as manual!)
AutoMapper: 5-15x slower (~250-750 ns)
```

---

## 1. Manual mapping

### Простой пример

```csharp
public record User(int Id, string Name, string Email, string PasswordHash);

public record UserDto(int Id, string Name, string Email);  // без PasswordHash!

// Manual extension method
public static class UserMapping
{
    public static UserDto ToDto(this User user) =>
        new(user.Id, user.Name, user.Email);

    public static User ToEntity(this CreateUserDto dto) =>
        new(0, dto.Name, dto.Email, HashPassword(dto.Password));
}

// Использование
var dto = user.ToDto();
var newUser = createDto.ToEntity();
```

### Когда manual лучше всего

✅ **Используй manual когда:**
- Маленькие entities (< 10 properties)
- Mapping содержит **business logic** (не просто copy)
- Performance-critical hot path
- Нужна полная control

❌ **Manual плохо когда:**
- 30+ properties (boilerplate hell)
- Часто меняется schema
- Many DTO variants (User, UserPublic, UserAdmin, UserExport...)

### Проблема

```csharp
// Со временем — длинный, error-prone:
public static UserExportDto ToExportDto(this User user) =>
    new(
        user.Id,
        user.Name,
        user.Email,
        user.CreatedAt,
        user.LastLoginAt,
        user.Address?.Street,
        user.Address?.City,
        user.Address?.Country,
        user.Phone,
        user.IsVerified,
        user.Subscription?.Plan,
        user.Subscription?.ExpiresAt,
        user.Roles.Select(r => r.Name).ToList(),
        user.Preferences?.Language,
        user.Preferences?.Timezone,
        // ... 25 more properties
    );
// Forgot одно поле → bug в production
```

---

## 2. AutoMapper — runtime reflection

### Установка и setup

```bash
dotnet add package AutoMapper
# AddAutoMapper входит в core-пакет начиная с AutoMapper 13.
# Отдельный AutoMapper.Extensions.Microsoft.DependencyInjection — deprecated (репозиторий в архиве), не добавлять.
```

```csharp
// Profile
public class UserProfile : Profile
{
    public UserProfile()
    {
        CreateMap<User, UserDto>();
        CreateMap<CreateUserDto, User>()
            .ForMember(dest => dest.PasswordHash, 
                opt => opt.MapFrom(src => HashPassword(src.Password)))
            .ForMember(dest => dest.CreatedAt,
                opt => opt.MapFrom(src => DateTime.UtcNow));
    }
}

// DI
builder.Services.AddAutoMapper(typeof(UserProfile));

// Use
public class UserController(IMapper mapper, IUserService service) : ControllerBase
{
    [HttpPost]
    public async Task<IActionResult> Create(CreateUserDto dto)
    {
        var user = mapper.Map<User>(dto);
        var created = await service.CreateAsync(user);
        return Ok(mapper.Map<UserDto>(created));
    }
}
```

### Сложные mappings

```csharp
public class OrderProfile : Profile
{
    public OrderProfile()
    {
        // Custom value resolver
        CreateMap<Order, OrderDto>()
            .ForMember(dest => dest.Total, 
                opt => opt.MapFrom(src => src.Items.Sum(i => i.Price * i.Quantity)))
            .ForMember(dest => dest.ItemCount, 
                opt => opt.MapFrom(src => src.Items.Count))
            .ForMember(dest => dest.CustomerName, 
                opt => opt.MapFrom(src => src.Customer.FullName));

        // Conditional
        CreateMap<User, UserDto>()
            .ForMember(dest => dest.PhoneVisible, 
                opt => opt.Condition(src => src.IsVerified));
    }
}
```

### EF Core ProjectTo

Главное преимущество AutoMapper — `ProjectTo` для EF queries:

```csharp
// Без ProjectTo — selects ALL columns then maps
var users = await db.Users
    .Where(u => u.IsActive)
    .ToListAsync();
var dtos = mapper.Map<List<UserDto>>(users);

// ✅ ProjectTo — SQL SELECT только нужные columns
var dtos = await db.Users
    .Where(u => u.IsActive)
    .ProjectTo<UserDto>(mapper.ConfigurationProvider)
    .ToListAsync();

// Generated SQL:
// SELECT u.Id, u.Name, u.Email FROM Users u WHERE u.IsActive = 1
// (без других columns User entity)
```

См. [[queries-performance|EF Queries Performance]].

### Минусы AutoMapper

**1. Performance** — reflection-based, slow в hot path:

```
Map User → UserDto (10 properties): 250 ns
Manual:                                50 ns
Difference: 5x
```

При 10K RPS = 2 ms wasted per second per мапping = заметно в latency.

**2. Magic** — silent failures:

```csharp
public class UserDto
{
    public string EmailAddress { get; set; }  // в Source — Email!
}

CreateMap<User, UserDto>();
// Compiles, runs, EmailAddress = null. No warning.
```

**3. Testing burden** — `mapper.ConfigurationProvider.AssertConfigurationIsValid()` обязателен.

**4. AOT-incompatible** — reflection не работает в Native AOT.

### ⚠️ AutoMapper commercial (v15+, 2025)

В **июле 2025** автор AutoMapper (Jimmy Bogard) перевёл библиотеку на commercial license под Lucky Penny Software — та же история, что с MediatR (см. [[cqrs-mediatr|CQRS и MediatR]]).

| | До v15 | v15+ |
|--|--------|------|
| Лицензия | Apache 2.0 / MIT (free) | Dual: RPL-1.5 или commercial |
| OSS, личное, revenue < $5M | Free | Free tier |
| Production / commercial | Free | Платная (per-developer tiers) |
| Security updates | Free | Commercial only после 14.x |

Enforcement — только предупреждения в логах (нет license-сервера, фичи не отключаются), но compliance-риск для компаний реальный.

**Что делать:** остаться на ≤ 14.x (Apache 2.0 / MIT, без обновлений), заплатить — или мигрировать на Mapperly, что и так recommended default этого файла. Лицензия — ещё один аргумент в пользу миграции сверх performance и AOT.

---

## 3. Mapperly — source generator (recommended 2026)

### Что это

Source generator — генерирует mapping code **в compile time**. Performance = manual + zero boilerplate.

### Установка

```bash
dotnet add package Riok.Mapperly
```

### Базовое использование

```csharp
[Mapper]
public partial class UserMapper
{
    public partial UserDto ToDto(User user);
    public partial User ToEntity(CreateUserDto dto);
}

// Use
var mapper = new UserMapper();
var dto = mapper.ToDto(user);
```

**Что генерируется compile-time:**
```csharp
// Auto-generated by Mapperly:
public partial class UserMapper
{
    public partial UserDto ToDto(User user)
    {
        return new UserDto
        {
            Id = user.Id,
            Name = user.Name,
            Email = user.Email
        };
    }
}
```

**Performance = manual.** Никакой reflection.

### Custom logic

```csharp
[Mapper]
public partial class UserMapper
{
    [MapProperty(nameof(CreateUserDto.Password), nameof(User.PasswordHash))]
    public partial User ToEntity(CreateUserDto dto);

    // Custom converter
    private string HashPassword(string raw) => /* hash logic */;
}
```

### Compile-time errors!

```csharp
public class UserDto
{
    public string EmailAddress { get; set; }  // Source has 'Email', not 'EmailAddress'
}

[Mapper]
public partial class UserMapper
{
    public partial UserDto ToDto(User user);
}

// COMPILE ERROR:
// RMG020: The member EmailAddress on the mapping target type UserDto cannot be mapped
```

**Не runtime fail — compile error!** Refactoring safe.

### EF Core projection

```csharp
[Mapper]
public partial class UserMapper
{
    public partial IQueryable<UserDto> ProjectToDto(IQueryable<User> users);
}

// Generated:
// public partial IQueryable<UserDto> ProjectToDto(IQueryable<User> users) =>
//     users.Select(u => new UserDto { Id = u.Id, Name = u.Name, Email = u.Email });

var dtos = await mapper.ProjectToDto(db.Users.Where(u => u.IsActive)).ToListAsync();
```

EF transates Select → SQL `SELECT Id, Name, Email FROM Users` — same как AutoMapper ProjectTo, но без reflection.

### AOT-friendly

```xml
<PropertyGroup>
  <PublishAot>true</PublishAot>
</PropertyGroup>
```

Mapperly работает в Native AOT (AutoMapper — нет).

См. [[source-generators|Source Generators]] и [[native-aot|Native AOT]].

---

## 4. Сравнение

| | Manual | AutoMapper | Mapperly |
|--|--------|------------|----------|
| **Performance** | 50 ns | 250 ns (5x) | 50 ns (=manual) |
| **Boilerplate** | High | Low | None |
| **Type safety** | Compile-time | Runtime | Compile-time |
| **Refactoring** | IDE supports | Magic strings | IDE supports |
| **AOT-compatible** | ✅ | ❌ | ✅ |
| **Testing burden** | Low | High (assert config) | Low |
| **Complex mappings** | Easy | Easy | Medium |
| **Learning curve** | None | Medium | Low |

---

## 5. Case Study #1 — API DTO chain

**Сценарий:** REST API. Request → Command → Domain → Entity → DTO → Response.

```csharp
// 1. Request (Web layer)
public record CreateOrderRequest(int CustomerId, List<OrderItemRequest> Items);
public record OrderItemRequest(int ProductId, int Quantity);

// 2. Command (Application layer)
public record CreateOrderCommand(int CustomerId, IEnumerable<(int ProductId, int Quantity)> Items) 
    : IRequest<Result<OrderDto>>;

// 3. Domain (Domain layer)
public class Order
{
    public int Id { get; private set; }
    public Customer Customer { get; private set; }
    public List<OrderItem> Items { get; private set; }
    public OrderStatus Status { get; private set; }
    public Money Total => new(Items.Sum(i => i.Subtotal.Amount), "USD");
    
    public static Order Create(Customer customer, IEnumerable<OrderItem> items) { /* ... */ }
}

// 4. EF Entity (Infrastructure)
public class OrderEntity
{
    public int Id { get; set; }
    public int CustomerId { get; set; }
    public List<OrderItemEntity> Items { get; set; }
    public string Status { get; set; }  // string в DB
    public DateTime CreatedAt { get; set; }
}

// 5. Response DTO (Web layer)
public record OrderDto(int Id, string CustomerName, decimal Total, string Status, List<OrderItemDto> Items);
public record OrderItemDto(int ProductId, string ProductName, int Quantity, decimal LineTotal);
```

### С Mapperly

```csharp
[Mapper]
public partial class OrderMapper
{
    public partial CreateOrderCommand ToCommand(CreateOrderRequest request);
    
    public partial OrderEntity ToEntity(Order domain);
    
    public partial Order ToDomain(OrderEntity entity);
    
    [MapProperty(nameof(Order.Customer) + "." + nameof(Customer.FullName), nameof(OrderDto.CustomerName))]
    [MapProperty(nameof(Order.Total) + "." + nameof(Money.Amount), nameof(OrderDto.Total))]
    public partial OrderDto ToDto(Order domain);
    
    public partial IQueryable<OrderDto> ProjectToDto(IQueryable<OrderEntity> orders);
}
```

```csharp
// Controller
[HttpPost]
public async Task<IActionResult> Create(CreateOrderRequest request)
{
    var command = _mapper.ToCommand(request);
    var result = await _mediator.Send(command);
    return result.IsSuccess ? Ok(result.Value) : BadRequest(result.Error);
}

// Handler
public async Task<Result<OrderDto>> Handle(CreateOrderCommand cmd, CancellationToken ct)
{
    var customer = await _customerRepo.GetAsync(cmd.CustomerId);
    var items = await BuildItems(cmd.Items);
    
    var order = Order.Create(customer, items);
    var entity = _mapper.ToEntity(order);
    
    _db.Orders.Add(entity);
    await _db.SaveChangesAsync();
    
    return Result.Ok(_mapper.ToDto(order));
}

// Query
public async Task<List<OrderDto>> GetByUser(int userId) =>
    await _mapper.ProjectToDto(_db.Orders.Where(o => o.CustomerId == userId)).ToListAsync();
```

См. [[cqrs-mediatr|CQRS & MediatR]].

---

## 6. Case Study #2 — AutoMapper performance issue в hot path

**Сценарий:** API endpoint вызывается 5K RPS, mapping User → UserDto (15 properties). Profiler показывает AutoMapper top in CPU.

### ❌ AutoMapper

```csharp
[HttpGet("{id}")]
public async Task<IActionResult> Get(int id)
{
    var user = await _service.GetByIdAsync(id);
    return Ok(_mapper.Map<UserDto>(user));  // 350 ns × 5K RPS = 1.75 ms/sec CPU только на мapping
}
```

### ✅ Migration на Mapperly

```csharp
[Mapper]
public partial class UserMapper
{
    public partial UserDto ToDto(User user);
}

// Controller
[HttpGet("{id}")]
public async Task<IActionResult> Get(int id)
{
    var user = await _service.GetByIdAsync(id);
    return Ok(_mapper.ToDto(user));  // 50 ns × 5K = 0.25 ms/sec — 7x reduction
}
```

**Результат:**
- p99 latency: -8 ms
- CPU usage: -12%
- Tests: все проходят (Mapperly compile-time validated)

---

## 7. Case Study #3 — Migration AutoMapper → Mapperly

**Сценарий:** Legacy app, 30+ AutoMapper profiles. Хотим migrate на Mapperly.

### Strategy — incremental

```csharp
// Step 1: Both работают параллельно
public class UserService
{
    private readonly IMapper _autoMapper;          // existing
    private readonly UserMapperly _mapperly;       // new

    public UserDto MapNew(User u) => _mapperly.ToDto(u);
    public UserDto MapOld(User u) => _autoMapper.Map<UserDto>(u);
}

// Step 2: Tests на equality
[Fact]
public void Mappers_ProduceSameResult()
{
    var user = new User { /* ... */ };
    
    var autoResult = _autoMapper.Map<UserDto>(user);
    var mapperResult = _mapperly.ToDto(user);
    
    autoResult.Should().BeEquivalentTo(mapperResult);
}

// Step 3: Replace в caller'ах постепенно
// Step 4: Удалить AutoMapper после full migration
```

### Real-world numbers

При migration большого проекта:
- 30 profiles × ~10 mappings = 300 mappings
- Mapperly accepted ~85% direct
- 15% потребовали custom MapProperty
- Performance gain: avg 4x в API endpoints

См. [[source-generators|Source Generators]].

---

## 8. Case Study #4 — Когда manual лучше всего

**Сценарий:** Простой DTO, 5 properties, mapping раз на 100 RPS.

```csharp
// ✅ Manual — простой и читаемый
public record UserSimpleDto(int Id, string Name, string Email);

public static class UserExtensions
{
    public static UserSimpleDto ToSimpleDto(this User user) =>
        new(user.Id, user.Name, user.Email);
}

// Один метод, одна строка. Не нужен framework.
```

**Argument:** Mapperly = 1 line. Manual = 1 line. Зачем dependency?

**Decision:** для < 10 mappings в проекте — manual проще. Для 30+ — Mapperly.

---

## 9. Case Study #5 — Mapping с domain logic

**Сценарий:** DTO имеет computed property — total с tax, status enum → string, и т.д.

### ❌ Mapping content в DTO

```csharp
public class OrderDto
{
    public decimal Total { get; set; }
    
    public OrderDto(Order order)
    {
        Total = order.Items.Sum(i => i.Price * i.Quantity) * 1.2m;  // tax!
        // ⚠️ Business logic в DTO — DTO должен быть data-only!
    }
}
```

### ✅ Logic в Domain, mapping flat

```csharp
// Domain
public class Order
{
    public Money Subtotal => new(Items.Sum(i => i.LineTotal.Amount), "USD");
    public Money Tax => Subtotal * 0.2m;
    public Money Total => Subtotal + Tax;
}

// DTO — pure data
public record OrderDto(int Id, decimal Subtotal, decimal Tax, decimal Total);

[Mapper]
public partial class OrderMapper
{
    [MapProperty(nameof(Order.Subtotal) + "." + nameof(Money.Amount), nameof(OrderDto.Subtotal))]
    [MapProperty(nameof(Order.Tax) + "." + nameof(Money.Amount), nameof(OrderDto.Tax))]
    [MapProperty(nameof(Order.Total) + "." + nameof(Money.Amount), nameof(OrderDto.Total))]
    public partial OrderDto ToDto(Order order);
}
```

**Lesson:** Domain — место для business logic. DTO — pure data. Mapping — transformation, не logic.

См. [[ddd|DDD]].

---

## 10. Case Study #6 — Mapping для разных audiences

**Сценарий:** User entity — один. Но API возвращает разные DTOs:
- Admin → полный DTO (PhoneNumber, LastLoginIP, IsBanned)
- Public → минимальный (Name, Avatar)
- Owner → personal info (Email, PhoneNumber, но не PasswordHash)

```csharp
[Mapper]
public partial class UserMapper
{
    public partial UserPublicDto ToPublicDto(User user);
    public partial UserOwnerDto ToOwnerDto(User user);
    public partial UserAdminDto ToAdminDto(User user);
}

public record UserPublicDto(int Id, string Name, string? AvatarUrl);
public record UserOwnerDto(int Id, string Name, string Email, string? PhoneNumber, string? AvatarUrl);
public record UserAdminDto(int Id, string Name, string Email, string? PhoneNumber, 
    string? LastLoginIP, bool IsBanned, DateTime CreatedAt);

// Controller
[HttpGet("{id}")]
public async Task<IActionResult> Get(int id)
{
    var user = await _service.GetAsync(id);
    
    return User.IsInRole("Admin") ? Ok(_mapper.ToAdminDto(user))
        : User.GetUserId() == id ? Ok(_mapper.ToOwnerDto(user))
        : Ok(_mapper.ToPublicDto(user));
}
```

**Преимущество:** компилятор проверяет — нельзя случайно вернуть `PasswordHash` через public DTO (он не существует в `UserPublicDto`).

См. [[auth-security|Auth & Security]].

---

## 11. Common Pitfalls

### 1. AutoMapper в hot path

5x slower vs manual / Mapperly. В hot path — заметно.

### 2. AutoMapper без `AssertConfigurationIsValid`

```csharp
// В app startup или в test:
mapper.ConfigurationProvider.AssertConfigurationIsValid();
// Проверяет что все members mapped (или explicit ignored)
```

Без этой проверки — silent failures с null fields.

### 3. Mapping vault'ом — sensitive data leak

```csharp
public class UserDto
{
    public string PasswordHash { get; set; }  // ⚠️ leaked!
}

mapper.Map<UserDto>(user);  // Включит PasswordHash
```

**Защита:** explicit DTO без sensitive fields. Records — immutable, легко audit.

### 4. Magic strings в AutoMapper

```csharp
.ForMember("EmailAddress", opt => opt.MapFrom(src => src.Email))
// ⚠️ String — refactor сломает

// ✅ nameof
.ForMember(dest => dest.EmailAddress, opt => opt.MapFrom(src => src.Email))
```

Mapperly — compile-time, ту же problem не имеет.

### 5. Mapping с side effects

```csharp
public static UserDto ToDto(this User user)
{
    _logger.LogInformation("Mapping user {Id}", user.Id);  // ⚠️ side effect!
    return new UserDto { /* ... */ };
}
```

Mapping должен быть **pure**. Side effects → отдельный method.

### 6. Reverse mapping без validation

```csharp
public static User FromDto(this UserDto dto) =>
    new() { Id = dto.Id, Email = dto.Email, Name = dto.Name };

// ⚠️ Что если dto.Email не валиден? Создали broken User.
```

**Solution:** validation **до** mapping (FluentValidation в pipeline).

### 7. Глубокий граф через AutoMapper

```csharp
// User → UserDto
//   includes Orders → OrderDto
//     includes Items → ItemDto
//       includes Product → ProductDto

// AutoMapper загрузит весь граф из EF (lazy loading triggered)
// → N+1 query problem
```

**Solution:** ProjectTo (для AutoMapper) или explicit Include + Mapperly query.

### 8. Mapperly с mutable types

```csharp
public class UserDto
{
    public List<string> Tags { get; set; }  // mutable!
}

[Mapper] public partial UserDto ToDto(User user);
// Generated: dto.Tags = user.Tags  ← ОДНА И ТА ЖЕ ссылка!
// Изменение dto.Tags меняет user.Tags
```

**Решение:** `[MapperConstructor]` для immutable mappings, или копировать collections explicitly.

### 9. Forgetting init-only

```csharp
public record UserDto
{
    public int Id { get; init; }
    public string Name { get; init; }
}

// AutoMapper / Mapperly — оба support init-only properties.
// Старый AutoMapper (до 10.0) — нет.
```

### 10. ConditionalMapping без default

```csharp
.ForMember(dest => dest.Phone, opt => opt.Condition(src => src.IsVerified));
// Если IsVerified = false → dest.Phone остаётся default (null или "")
// User думал "будет hidden" — нет, просто null
```

---

## 12. Best Practices

### Choosing strategy

```
Размер проекта?
├── Small (<10 mappings) → Manual
├── Medium (10-50) → Mapperly (recommended 2026)
└── Large (50+) → Mapperly + помощь от AI для генерации
```

### Mapping rules

- **DTOs — pure data** (records ideal)
- **Mapping — pure transformation** (no side effects)
- **Validation — отдельно** от mapping
- **Different DTOs для different audiences** (admin/public/owner)
- **`[MapperIgnoreSource]` / `[MapperIgnoreTarget]`** explicit когда нужно
- **Unit tests** для complex mappings (особенно с computed fields)

### EF Core integration

- **`ProjectTo` (AutoMapper) или `ProjectToDto` (Mapperly)** для queries
- **Не mapping после `ToList()`** — теряешь column projection
- **`AsNoTracking`** + projection — fastest reads

### Performance

- **Profile перед optimization** — может manual / Mapperly не нужен
- **AOT? → Mapperly** (или manual)
- **Hot path? → Manual** (или Mapperly с benchmark verify)
- **Bulk mapping? → ProjectTo SQL projection**

См. [[queries-performance|EF Queries Performance]].

---

## 13. Cheat sheet

| Need | Solution |
|------|----------|
| Few simple mappings | Manual extension methods |
| Many mappings + .NET 6+ | Mapperly |
| EF Core projection | Mapperly `ProjectToDto` |
| Existing project на AutoMapper | Migrate to Mapperly постепенно |
| Native AOT | Mapperly или manual |
| Custom logic в mapping | `[MapProperty]` + helper method |
| Different DTOs для audiences | Multiple methods в одном `[Mapper]` |
| Sensitive fields filtering | Explicit DTO без них |

---

## 14. Decision tree

```
Mapping нужен?
│
├── Сколько mappings в проекте?
│   ├── < 10 → Manual extension methods
│   ├── 10-50 → Mapperly (default 2026)
│   └── 50+ → Mapperly + auto-gen helpers
│
├── EF Core queries?
│   ├── Простые → Mapperly ProjectToDto
│   ├── Сложные projections → ProjectTo (AutoMapper) или manual Select
│
├── Performance critical?
│   ├── Yes → Manual или Mapperly (= same)
│   └── No → выбор по удобству
│
├── Native AOT?
│   ├── Yes → Manual / Mapperly (НЕ AutoMapper)
│   └── No → любой
│
└── Migrating с AutoMapper?
    └── Mapperly + dual-write tests + incremental
```

---

## 15. См. также

- [[source-generators|Source Generators]] — Mapperly использует SG
- [[queries-performance|EF Queries Performance]] — ProjectTo
- [[cqrs-mediatr|CQRS & MediatR]] — где mapping применяется
- [[ddd|DDD]] — Domain vs DTO separation
- [[native-aot|Native AOT]] — AutoMapper не работает
- [[api-design|API Design]] — DTO design
- [[auth-security|Auth & Security]] — sensitive data filtering

## 16. Reading list

- **Riok.Mapperly docs** — github.com/riok/mapperly
- **AutoMapper documentation** — automapper.org
- **MapsterMapper** — github.com/MapsterMapper/Mapster (alternative)
- **Andrew Lock — Mapperly migration** — andrewlock.net
- **Khalid Abuhakmeh — Source generators in .NET** — khalidabuhakmeh.com
- **Microsoft Docs — Source generators** — learn.microsoft.com/dotnet/csharp/roslyn-sdk/source-generators-overview
