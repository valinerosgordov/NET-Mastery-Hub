---
tags: [aspnet, signalr, websocket, sse, real-time, hubs, scale-out, redis-backplane, azure-signalr]
level: Senior
date: 2026-08-02
---

# SignalR — real-time коммуникация

> Полный гайд по standalone SignalR в .NET 10. Закрывает: Hub методы, transport selection (WebSocket/SSE/Long Polling), groups & users, scale-out (Redis backplane / Azure SignalR), authentication, JS / .NET / MAUI клиенты, streaming, MessagePack, security, observability.

---

## Что это, зачем и когда

### Что такое SignalR?
**Библиотека для real-time коммуникации server↔client** в ASP.NET Core. Сервер может **push** данные клиенту без request — клиент получает обновления как только они появляются.

**Аналогия:** REST — это письма по почте (запросил → ждёшь ответ). SignalR — это живой телефонный звонок (всегда соединены, можно говорить с обеих сторон).

### Зачем?

| Без SignalR | С SignalR |
|-------------|-----------|
| Polling — клиент каждые 5 сек спрашивает "есть новое?" | Server push мгновенно |
| Long polling — соединение висит до изменения | WebSocket — efficient bidirectional |
| Пишешь свой WebSocket boilerplate | Hub abstraction — методы вместо frame parsing |
| Reconnection руками | Auto-reconnect встроен |
| Один transport — не работает за proxy | Auto-fallback WebSocket → SSE → Long Polling |
| Scale-out руками через Redis | `AddStackExchangeRedis()` — встроено |

### Когда SignalR

✅ **Хорошо для:**
- Chat / messaging
- Live dashboards (метрики, IoT)
- Notifications (push events)
- Collaborative editing
- Live sports / stocks / trading
- Multiplayer games (low-latency turn-based)
- Live progress updates (long-running operations)

❌ **Не подходит:**
- Pure request-response (используй REST)
- Streaming большого объёма данных (gRPC streaming)
- Inter-service communication (используй RabbitMQ / Kafka / gRPC)
- Pub/sub между micro-services (используй MQ)

### SignalR vs альтернативы

| | SignalR | Raw WebSocket | gRPC streaming | Server-Sent Events |
|--|---------|---------------|----------------|---------------------|
| Bidirectional | ✅ | ✅ | ✅ | ❌ (только server→client) |
| Auto-reconnect | ✅ | ❌ | ❌ | ✅ |
| Multiple transports | ✅ | ❌ | ❌ | ❌ |
| Scale-out built-in | ✅ Redis/Azure | ❌ | ❌ | ❌ |
| RPC abstraction | ✅ Hubs | ❌ | ✅ | ❌ |
| Browser support | Excellent | Excellent | Limited | Excellent |
| Mobile / desktop | ✅ .NET / MAUI | ✅ | ✅ | ✅ |
| Когда | Default real-time | Custom protocol | Microservice streaming | Server push only |

### Нативный SSE в .NET 10 — когда SignalR не нужен

.NET 10 добавил first-class SSE в Minimal API: `TypedResults.ServerSentEvents` принимает `IAsyncEnumerable<SseItem<T>>` и сам пишет `text/event-stream` (data-строки, event id, retry, cancellation). Это сдвигает decision tree: раньше SignalR брали «по умолчанию» даже для one-way пуша, потому что руками SSE собирать было муторно — теперь для **однонаправленного** стриминга (AI-токены из LLM, live-цены, прогресс long-running задач) достаточно фреймворка.

```csharp
app.MapGet("/notifications", (NotificationService svc, CancellationToken ct) =>
    TypedResults.ServerSentEvents(
        svc.StreamAsync(ct),          // IAsyncEnumerable<SseItem<Notification>>
        eventType: "notification"));
```

Почему это часто лучше SignalR для server push: обычный HTTP (без upgrade — проходит через любые прокси/CDN), авто-reconnect с `Last-Event-Id` у браузерного `EventSource`, ноль клиентских библиотек. SignalR остаётся выбором, когда нужны **двусторонний** обмен, RPC-абстракция Hub'ов, groups/user-targeting или scale-out через backplane.

---

## 1. Setup

```xml
<!-- MyApp.Api.csproj -->
<ItemGroup>
  <PackageReference Include="Microsoft.AspNetCore.SignalR" />
  <PackageReference Include="Microsoft.AspNetCore.SignalR.Protocols.MessagePack" />
  <PackageReference Include="Microsoft.AspNetCore.SignalR.StackExchangeRedis" />
</ItemGroup>
```

```csharp
// Program.cs
var builder = WebApplication.CreateBuilder(args);

builder.Services.AddSignalR()
    .AddMessagePackProtocol()       // более компактный binary protocol
    .AddStackExchangeRedis(redisConnString, options =>
    {
        options.Configuration.ChannelPrefix = RedisChannel.Literal("MyApp");
    });

var app = builder.Build();

app.MapHub<ChatHub>("/hubs/chat");
app.MapHub<NotificationHub>("/hubs/notifications");

app.Run();
```

---

## 2. Hub — основа

### Базовый Hub

```csharp
public class ChatHub : Hub
{
    private readonly ILogger<ChatHub> _logger;
    
    public ChatHub(ILogger<ChatHub> logger) => _logger = logger;
    
    // Метод вызывается клиентом
    public async Task SendMessage(string user, string message)
    {
        // Broadcast всем connected
        await Clients.All.SendAsync("ReceiveMessage", user, message);
    }
    
    // Override lifecycle hooks
    public override async Task OnConnectedAsync()
    {
        _logger.LogInformation("Connected: {ConnectionId}", Context.ConnectionId);
        await Clients.Caller.SendAsync("Welcome", "Hello!");
        await base.OnConnectedAsync();
    }
    
    public override async Task OnDisconnectedAsync(Exception? exception)
    {
        _logger.LogInformation("Disconnected: {ConnectionId}", Context.ConnectionId);
        await base.OnDisconnectedAsync(exception);
    }
}
```

### Targeting clients

```csharp
public class ChatHub : Hub
{
    public async Task SendMessage(string message)
    {
        // Все
        await Clients.All.SendAsync("Receive", message);
        
        // Все КРОМЕ caller
        await Clients.Others.SendAsync("Receive", message);
        
        // Только caller
        await Clients.Caller.SendAsync("Echo", message);
        
        // Конкретный connection
        await Clients.Client(connectionId).SendAsync("Receive", message);
        
        // Несколько connections
        await Clients.Clients(connId1, connId2).SendAsync("Receive", message);
        
        // Группа
        await Clients.Group("admins").SendAsync("Receive", message);
        
        // Все КРОМЕ группы
        await Clients.GroupExcept("admins", connectionId).SendAsync("Receive", message);
        
        // Группы (несколько)
        await Clients.Groups("admins", "moderators").SendAsync("Receive", message);
        
        // По User ID (одному пользователю на всех его devices/tabs)
        await Clients.User(userId).SendAsync("Receive", message);
        
        // Multiple users
        await Clients.Users(userId1, userId2).SendAsync("Receive", message);
    }
}
```

### Strongly-typed Hub (рекомендуется)

```csharp
// Interface для client methods
public interface IChatClient
{
    Task ReceiveMessage(string user, string message);
    Task UserJoined(string user);
    Task UserLeft(string user);
}

// Hub<T> — type-safe вызовы клиента
public class ChatHub : Hub<IChatClient>
{
    public async Task SendMessage(string user, string message)
    {
        await Clients.All.ReceiveMessage(user, message);  // ← type-safe!
        // если опечатался — compile error
    }
}
```

**Преимущества:**
- IntelliSense на client methods
- Compile-time check имени и параметров
- Refactor-safe (rename меняет везде)

---

## 3. Groups

```csharp
public class ChatHub : Hub<IChatClient>
{
    public async Task JoinRoom(string roomName)
    {
        await Groups.AddToGroupAsync(Context.ConnectionId, roomName);
        await Clients.Group(roomName).UserJoined($"User joined room {roomName}");
    }
    
    public async Task LeaveRoom(string roomName)
    {
        await Groups.RemoveFromGroupAsync(Context.ConnectionId, roomName);
        await Clients.Group(roomName).UserLeft($"User left room {roomName}");
    }
    
    public async Task SendToRoom(string roomName, string message)
    {
        await Clients.Group(roomName).ReceiveMessage(Context.UserIdentifier ?? "anon", message);
    }
}
```

> [!info] Group — ephemeral
> Группа существует пока в ней есть подписчики. Когда последний disconnect — группа исчезает. Это in-memory структура (или Redis при scale-out).

### Auto-cleanup при disconnect

SignalR автоматически удаляет connection из всех групп при disconnect — не нужно делать вручную.

---

## 4. User-based targeting

```csharp
// Всем connections данного пользователя — независимо от device/browser
await Clients.User(userId).SendAsync("Notification", message);
```

### Default user identifier — ClaimTypes.NameIdentifier

```csharp
// JWT claim "sub" → User.Identity.NameIdentifier → SignalR userId
[Authorize]
public class NotificationHub : Hub<INotificationClient>
{
    // Context.UserIdentifier = JWT 'sub' claim
}
```

### Custom user identifier

```csharp
public class CustomUserIdProvider : IUserIdProvider
{
    public string? GetUserId(HubConnectionContext connection)
    {
        // Например, по email или tenantId+userId
        var email = connection.User?.FindFirst(ClaimTypes.Email)?.Value;
        return email;
    }
}

// Регистрация
builder.Services.AddSingleton<IUserIdProvider, CustomUserIdProvider>();
```

---

## 5. Send message FROM other parts of app

Из контроллера / background service нужно отправить SignalR клиенту:

```csharp
public class OrdersController(IHubContext<NotificationHub, INotificationClient> hubContext)
{
    [HttpPost("/orders/{id}/complete")]
    public async Task<IActionResult> Complete(Guid id)
    {
        // ... обработка ...
        
        // Push клиенту
        await hubContext.Clients.User(userId).OrderCompleted(id);
        
        return Ok();
    }
}
```

`IHubContext<THub, TClient>` — типизированный, можно использовать в любом сервисе.

### Из Background Service

```csharp
public class OrderProcessor(IHubContext<NotificationHub, INotificationClient> hubContext) : BackgroundService
{
    protected override async Task ExecuteAsync(CancellationToken ct)
    {
        await foreach (var event in queue.ReadAllAsync(ct))
        {
            await ProcessAsync(event);
            await hubContext.Clients.User(event.UserId).OrderCompleted(event.OrderId);
        }
    }
}
```

---

## 6. Authentication & Authorization

### JWT Authentication для SignalR

```csharp
builder.Services
    .AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
    .AddJwtBearer(options =>
    {
        options.TokenValidationParameters = /* ... */;
        
        // SignalR через WebSocket / SSE — не отправляет Authorization header
        // Token приходит как query string ?access_token=...
        options.Events = new JwtBearerEvents
        {
            OnMessageReceived = context =>
            {
                var accessToken = context.Request.Query["access_token"];
                var path = context.HttpContext.Request.Path;
                
                if (!string.IsNullOrEmpty(accessToken) && 
                    path.StartsWithSegments("/hubs"))
                {
                    context.Token = accessToken;
                }
                return Task.CompletedTask;
            }
        };
    });

builder.Services.AddAuthorization();

builder.Services.AddSignalR();

// Pipeline
app.UseAuthentication();
app.UseAuthorization();
app.MapHub<ChatHub>("/hubs/chat").RequireAuthorization();
```

### Authorization на Hub

```csharp
[Authorize]
public class AdminHub : Hub<IAdminClient> { ... }

[Authorize(Roles = "Admin")]
public class AdminMethodsHub : Hub<IAdminClient> { ... }

public class ChatHub : Hub<IChatClient>
{
    [Authorize(Policy = "PremiumUser")]
    public Task SendPremiumMessage(string msg) => /* ... */;
    
    [AllowAnonymous]  // override
    public Task GetPublicInfo() => /* ... */;
}
```

### Получение user info в Hub

```csharp
public class ChatHub : Hub<IChatClient>
{
    public async Task SendMessage(string message)
    {
        var user = Context.User;  // ClaimsPrincipal
        var userId = Context.UserIdentifier;
        var userName = user?.Identity?.Name;
        var roles = user?.FindAll(ClaimTypes.Role).Select(c => c.Value).ToList();
        
        await Clients.All.ReceiveMessage(userName ?? "anon", message);
    }
}
```

---

## 7. Streaming

### Server → Client streaming

```csharp
public class StreamingHub : Hub<IStreamingClient>
{
    public ChannelReader<int> CountTo(int max, CancellationToken ct)
    {
        var channel = Channel.CreateUnbounded<int>();
        
        _ = WriteItemsAsync(channel.Writer, max, ct);
        
        return channel.Reader;
    }
    
    private static async Task WriteItemsAsync(ChannelWriter<int> writer, int max, CancellationToken ct)
    {
        try
        {
            for (var i = 0; i < max; i++)
            {
                await writer.WriteAsync(i, ct);
                await Task.Delay(100, ct);
            }
        }
        catch (Exception ex)
        {
            writer.TryComplete(ex);
        }
        
        writer.TryComplete();
    }
    
    // Альтернатива — IAsyncEnumerable (.NET Core 3+)
    public async IAsyncEnumerable<int> CountToAsync(
        int max,
        [EnumeratorCancellation] CancellationToken ct)
    {
        for (var i = 0; i < max; i++)
        {
            yield return i;
            await Task.Delay(100, ct);
        }
    }
}
```

JS client:
```javascript
connection.stream("CountTo", 10).subscribe({
    next: (item) => console.log(item),
    complete: () => console.log("done"),
    error: (err) => console.error(err)
});
```

### Client → Server streaming (.NET Core 3+)

```csharp
public async Task UploadStream(IAsyncEnumerable<string> stream)
{
    await foreach (var item in stream)
    {
        Console.WriteLine($"Received: {item}");
    }
}
```

---

## 8. Scale-out — multiple servers

### Проблема

Без backplane: 3 ASP.NET сервера за load balancer. User A на server-1 шлёт сообщение. Server-1 broadcast'ит — но получают только клиенты на **этом же** server. Клиенты на server-2 и server-3 ничего не видят.

### Redis backplane

```csharp
builder.Services.AddSignalR()
    .AddStackExchangeRedis(connString, options =>
    {
        options.Configuration.ChannelPrefix = RedisChannel.Literal("MyApp");
    });
```

Каждый сервер publish'ит сообщения в Redis pub/sub. Все servers subscribe → получают сообщение → forwardят своим клиентам.

### Azure SignalR Service

Managed service от Azure — handles scale-out автоматически. .NET app только проксирует к Azure.

```csharp
builder.Services.AddSignalR()
    .AddAzureSignalR(connString);
```

**Архитектура:**
- Клиенты подключаются напрямую к Azure SignalR
- Твои servers подключены к Azure через outbound connection
- Azure routes messages между clients и servers

**Плюсы:**
- Auto scale до миллионов connections
- Backplane не твоя проблема
- Connection limit твоего сервера снимается

**Минусы:**
- Платный
- Vendor lock-in на Azure

> [!info] Когда Redis vs Azure SignalR
> - **Redis** — self-hosted, до ~10k connections per pod, multiple pods
> - **Azure SignalR** — Cloud, для 100k+ connections, mobile apps

### Sticky sessions при WebSocket

WebSocket — постоянное соединение. Если LB направляет каждый request на разный pod — connection ломается.

**Решения:**
- Sticky sessions (Cookie-based) на LB
- Auto-reconnect клиент с восстановлением state
- Azure SignalR — без sticky (Azure всё разруливает)

```yaml
# Kubernetes Ingress — sticky sessions
metadata:
  annotations:
    nginx.ingress.kubernetes.io/affinity: "cookie"
    nginx.ingress.kubernetes.io/affinity-mode: "persistent"
    nginx.ingress.kubernetes.io/session-cookie-name: "route"
```

---

## 9. Transports

SignalR auto-negotiate transport в порядке:
1. **WebSocket** — bidirectional, lowest latency
2. **Server-Sent Events** — server→client only, fallback
3. **Long Polling** — last resort, slow but works through any proxy

```csharp
// Restrict transports
app.MapHub<ChatHub>("/hubs/chat", options =>
{
    options.Transports = HttpTransportType.WebSockets | HttpTransportType.ServerSentEvents;
});
```

> [!warning] Long Polling fallback в production
> Если корпоративный proxy не поддерживает WebSocket → fallback на Long Polling. Это **очень неэффективно** (HTTP polling каждые 1-5 сек). Для production — настрой proxy на WebSocket support.

---

## 10. Клиенты

### JavaScript / TypeScript

```javascript
import { HubConnectionBuilder, LogLevel } from "@microsoft/signalr";

const connection = new HubConnectionBuilder()
    .withUrl("/hubs/chat", {
        accessTokenFactory: () => localStorage.getItem("token")
    })
    .withAutomaticReconnect([0, 2000, 10000, 30000])  // retry intervals
    .configureLogging(LogLevel.Information)
    .build();

// Receive
connection.on("ReceiveMessage", (user, message) => {
    console.log(`${user}: ${message}`);
});

// Lifecycle
connection.onreconnecting(() => console.log("Reconnecting..."));
connection.onreconnected(() => console.log("Reconnected"));
connection.onclose(error => console.error("Disconnected", error));

// Connect
await connection.start();

// Send
await connection.invoke("SendMessage", "John", "Hello!");

// или fire-and-forget
await connection.send("SendMessage", "John", "Hello!");
```

### .NET / MAUI client

```csharp
// NuGet: Microsoft.AspNetCore.SignalR.Client

var connection = new HubConnectionBuilder()
    .WithUrl("https://api.example.com/hubs/chat", options =>
    {
        options.AccessTokenProvider = () => Task.FromResult(jwt);
    })
    .WithAutomaticReconnect()
    .AddMessagePackProtocol()
    .Build();

// Receive
connection.On<string, string>("ReceiveMessage", (user, message) =>
{
    Console.WriteLine($"{user}: {message}");
});

// Lifecycle
connection.Reconnecting += error =>
{
    Console.WriteLine("Reconnecting...");
    return Task.CompletedTask;
};

connection.Closed += async (error) =>
{
    Console.WriteLine($"Closed: {error?.Message}");
    await Task.Delay(Random.Shared.Next(0, 5) * 1000);
    await connection.StartAsync();  // manual reconnect
};

// Connect
await connection.StartAsync();

// Send
await connection.InvokeAsync("SendMessage", "John", "Hello!");

// Stream consume
await foreach (var item in connection.StreamAsync<int>("CountTo", 10))
{
    Console.WriteLine(item);
}
```

### Strongly-typed .NET client

```csharp
// Использование source generator для type-safe client
var hubProxy = connection.CreateHubProxy<IChatHub>();
await hubProxy.SendMessage("John", "Hello!");
```

См. [SignalR Client Source Generator](https://learn.microsoft.com/aspnet/core/signalr/client-source-generator).

---

## 11. MessagePack vs JSON

```csharp
builder.Services.AddSignalR()
    .AddMessagePackProtocol();
```

| | JSON | MessagePack |
|--|------|-------------|
| Размер сообщения | Текст, large | Binary, ~50-70% smaller |
| Скорость serialization | Medium | Fast |
| Browser debug | Read in DevTools | Binary, harder to inspect |
| Browser support | Native | Через библиотеку |

**Когда MessagePack:**
- Mobile apps — экономия трафика
- High-throughput (sports/trading)
- Большие сообщения

**Когда JSON:**
- Прототип / debugging easy
- Browser-only клиенты с малой нагрузкой

---

## 12. Performance optimizations

### Connection limits

```csharp
builder.Services.AddSignalR(options =>
{
    options.MaximumReceiveMessageSize = 32 * 1024;  // default 32 KB
    options.ClientTimeoutInterval = TimeSpan.FromSeconds(30);
    options.KeepAliveInterval = TimeSpan.FromSeconds(15);
    options.HandshakeTimeout = TimeSpan.FromSeconds(15);
    options.EnableDetailedErrors = false;  // production: false
});
```

### Backpressure — bounded channels

```csharp
public ChannelReader<int> StreamData(CancellationToken ct)
{
    var channel = Channel.CreateBounded<int>(new BoundedChannelOptions(100)
    {
        FullMode = BoundedChannelFullMode.DropOldest
    });
    
    // Если client медленнее producer'а — старые messages drop
    return channel.Reader;
}
```

### Hub method optimizations

```csharp
public class ChatHub : Hub<IChatClient>
{
    // ❌ N+1 — fetch DB на каждом сообщении
    public async Task SendMessage(string msg)
    {
        var user = await db.Users.FindAsync(Context.UserIdentifier);
        await Clients.All.ReceiveMessage(user.Name, msg);
    }
    
    // ✅ Cache user info на connect
    public override async Task OnConnectedAsync()
    {
        var user = await db.Users.FindAsync(Context.UserIdentifier);
        Context.Items["userName"] = user.Name;
        await base.OnConnectedAsync();
    }
    
    public async Task SendMessage(string msg)
    {
        var userName = (string)Context.Items["userName"]!;
        await Clients.All.ReceiveMessage(userName, msg);
    }
}
```

### Group cardinality

Большая группа (10000+ subscribers) — `SendAsync` к группе берёт время O(N). Распределяй по sub-groups.

---

## 13. Observability

### OpenTelemetry integration

```csharp
builder.Services.AddOpenTelemetry()
    .WithTracing(tracing =>
    {
        tracing.AddAspNetCoreInstrumentation();
        // SignalR события автоматически попадают в trace через AspNetCore
    });
```

### Logging

```csharp
builder.Services.AddSignalR(options =>
{
    options.EnableDetailedErrors = builder.Environment.IsDevelopment();
});

// SignalR использует ILogger — конфигурируется через Logging
"Logging": {
  "LogLevel": {
    "Microsoft.AspNetCore.SignalR": "Information",
    "Microsoft.AspNetCore.Http.Connections": "Warning"
  }
}
```

### Custom metrics

```csharp
public class ChatHub : Hub<IChatClient>
{
    private readonly Counter<int> _messagesCounter;
    
    public ChatHub(IMeterFactory meterFactory)
    {
        var meter = meterFactory.Create("ChatHub");
        _messagesCounter = meter.CreateCounter<int>("chat.messages_sent");
    }
    
    public async Task SendMessage(string msg)
    {
        _messagesCounter.Add(1, new KeyValuePair<string, object?>("user_type", "registered"));
        await Clients.All.ReceiveMessage(Context.UserIdentifier ?? "anon", msg);
    }
}
```

См. [[diagnostics-tools|Diagnostics Tools]].

---

## 14. Security

### Origin validation

```csharp
builder.Services.AddCors(options =>
{
    options.AddPolicy("SignalRPolicy", policy =>
    {
        policy
            .WithOrigins("https://myapp.com", "https://www.myapp.com")
            .AllowAnyMethod()
            .AllowAnyHeader()
            .AllowCredentials();  // ← обязательно для SignalR cookies
    });
});

app.UseCors("SignalRPolicy");
```

### Rate limiting на Hub методы

```csharp
public class ChatHub : Hub<IChatClient>
{
    private static readonly Dictionary<string, DateTime> _lastMessage = new();
    
    public async Task SendMessage(string msg)
    {
        var key = Context.ConnectionId;
        if (_lastMessage.TryGetValue(key, out var last) && 
            DateTime.UtcNow - last < TimeSpan.FromSeconds(1))
        {
            await Clients.Caller.Error("Too many messages");
            return;
        }
        
        _lastMessage[key] = DateTime.UtcNow;
        await Clients.All.ReceiveMessage(Context.UserIdentifier ?? "anon", msg);
    }
}
```

Для глобального rate limiting — middleware (.NET 7+ Rate Limiting).

### Input validation

```csharp
public async Task SendMessage(string msg)
{
    if (string.IsNullOrWhiteSpace(msg) || msg.Length > 1000)
    {
        throw new HubException("Invalid message");
    }
    
    // sanitize — не позволяй HTML/JS injection
    var sanitized = HttpUtility.HtmlEncode(msg);
    await Clients.All.ReceiveMessage(Context.UserIdentifier ?? "anon", sanitized);
}
```

### Не отправляй sensitive data в общие группы

```csharp
// ❌ Все в комнате видят чужие emails
await Clients.Group(roomId).ReceiveMessage(user.Email, msg);

// ✅ Только display name
await Clients.Group(roomId).ReceiveMessage(user.DisplayName, msg);
```

---

## 15. Testing

### Unit testing Hub method

```csharp
public class ChatHubTests
{
    [Fact]
    public async Task SendMessage_broadcasts_to_all()
    {
        // Arrange
        var clients = Substitute.For<IHubCallerClients<IChatClient>>();
        var allClient = Substitute.For<IChatClient>();
        clients.All.Returns(allClient);
        
        var hub = new ChatHub
        {
            Clients = clients,
            Context = MockHubContext("user1", "conn1")
        };
        
        // Act
        await hub.SendMessage("Hello");
        
        // Assert
        await allClient.Received(1).ReceiveMessage("user1", "Hello");
    }
    
    private static HubCallerContext MockHubContext(string userId, string connId)
    {
        var context = Substitute.For<HubCallerContext>();
        context.UserIdentifier.Returns(userId);
        context.ConnectionId.Returns(connId);
        return context;
    }
}
```

### Integration test through TestServer

```csharp
[Fact]
public async Task Client_can_send_and_receive_message()
{
    await using var factory = new WebApplicationFactory<Program>();
    var server = factory.Server;
    
    var connection = new HubConnectionBuilder()
        .WithUrl(new Uri(server.BaseAddress, "/hubs/chat"), options =>
        {
            options.HttpMessageHandlerFactory = _ => server.CreateHandler();
        })
        .Build();
    
    var receivedMessages = new List<string>();
    connection.On<string, string>("ReceiveMessage", (user, msg) =>
    {
        receivedMessages.Add($"{user}: {msg}");
    });
    
    await connection.StartAsync();
    await connection.InvokeAsync("SendMessage", "test", "hello");
    
    // Wait for message
    await Task.Delay(100);
    receivedMessages.ShouldContain("test: hello");
}
```

---

## 16. Common Pitfalls

### 1. Hub — Transient (новый instance на каждый method call!)

```csharp
public class ChatHub : Hub
{
    private List<string> _messages = new();  // ❌ Новый list на каждый вызов
    
    public Task Add(string msg)
    {
        _messages.Add(msg);  // ❌ Lost после вызова
        return Task.CompletedTask;
    }
}

// ✅ State в external service (Redis, DB)
public class ChatHub(IChatStore store) : Hub
{
    public Task Add(string msg) => store.AddAsync(msg);
}
```

### 2. Long-running operation в Hub method

```csharp
// ❌ Блокирует Hub instance
public async Task GenerateReport()
{
    await Task.Delay(60_000);  // 1 минута
    await Clients.Caller.SendAsync("Done");
}

// ✅ Fire-and-forget с notify через IHubContext
public Task StartReport()
{
    _ = Task.Run(async () =>
    {
        await Task.Delay(60_000);
        await hubContext.Clients.Client(connId).SendAsync("ReportDone");
    });
    return Task.CompletedTask;
}
```

### 3. Reconnect теряет state (groups, server-side data)

```csharp
public override async Task OnConnectedAsync()
{
    // При reconnect — нужно re-add в группы
    var rooms = await GetUserRoomsAsync(Context.UserIdentifier);
    foreach (var room in rooms)
    {
        await Groups.AddToGroupAsync(Context.ConnectionId, room);
    }
}
```

### 4. JWT token expires во время connection

WebSocket connection живёт долго. JWT истёк → next method call fails.

**Решения:**
- Refresh token mechanism в client
- Long-lived JWT для SignalR (с осторожностью)
- Переподписать перед expiry → reconnect with new token

### 5. SignalR за reverse proxy без WebSocket support

Nginx без правильной конфигурации:

```nginx
location /hubs/ {
    proxy_pass http://backend;
    proxy_http_version 1.1;
    proxy_set_header Upgrade $http_upgrade;     # ← важно
    proxy_set_header Connection "upgrade";       # ← важно
    proxy_set_header Host $host;
    proxy_read_timeout 86400s;                   # ← long-lived connection
}
```

### 6. Без backplane — в кластере не работает

3 pod'а, user A на pod-1 → broadcast получает только pod-1 клиенты.

**Лечение:** `AddStackExchangeRedis` или `AddAzureSignalR`.

### 7. Большие сообщения throw exception

`MaximumReceiveMessageSize` default 32KB. Файлы / большой JSON → exception.

**Лечение:**
- Увеличить limit (но осторожно — память)
- Streaming для больших данных
- Upload файла через REST, потом notify через SignalR

### 8. Group не работает с Azure SignalR ServerlessMode

Serverless mode имеет ограничения. Используй Default mode для full feature set.

### 9. WebSocket + AppService Always-On

В Azure App Service нужно включить:
- WebSockets
- Always On (free tier — нет, paid — есть)

### 10. Test без `EnableDetailedErrors`

```csharp
// dev — true для удобства, prod — false (security)
options.EnableDetailedErrors = builder.Environment.IsDevelopment();
```

В production exception details скрыты от клиента (good security), но в logs полные.

---

## 17. Best Practices

- **Strongly-typed Hubs** (Hub\<TClient\>) — type safety, refactor-safe
- **MessagePack protocol** для production (mobile / high-throughput)
- **Redis backplane** или **Azure SignalR** для multi-instance
- **AutomaticReconnect** в client с back-off
- **CancellationToken** в Hub method и client
- **AsNoTracking** + DbContextFactory в Hub (Hub — Transient)
- **State в external store** — Hub instance не persistent
- **Group cleanup** — auto при disconnect, custom logic в OnConnected
- **JWT через query string** для WebSocket / SSE
- **CORS с AllowCredentials** для cross-origin
- **Origin validation** + rate limiting на Hub methods
- **Detailed errors off** в production
- **Sticky sessions** на LB или Azure SignalR
- **OpenTelemetry tracing** для observability
- **Streaming для больших данных** (не один большой message)

---

## 18. Когда что использовать — резюме

| Сценарий | Решение |
|----------|---------|
| Chat в browser | SignalR + Redis backplane + JS client |
| Live dashboard для админов | SignalR + groups (one per dashboard) |
| Mobile push (только server→client) | SSE (Server-Sent Events) или web push |
| Multiplayer game | SignalR + UDP for lossy / WebSocket for reliable |
| Trading platform real-time prices | SignalR + Azure SignalR + MessagePack + streaming |
| Notification service для миллионов | Azure SignalR Service |
| Inter-microservice events | RabbitMQ / Kafka / gRPC streaming, **не SignalR** |

---

## Cheat sheet

| Need | Solution |
|------|----------|
| Real-time chat | SignalR Hub + Group |
| Live notifications (1:1) | SignalR + UserId-based send |
| Broadcast (1:N) | `Clients.All` или `Clients.Group` |
| Per-connection state | `Context.Items` |
| Disconnect handling | `OnDisconnectedAsync` |
| Backpressure | `BoundedChannel` или throttle на client |
| Multi-server scale | Redis backplane |
| Auth | JWT в `access_token` query parameter (WebSockets quirk) |
| Reconnection | Auto-reconnect (default in JS client) |
| Serialization | MessagePack (faster than JSON) |
| Streaming response | `IAsyncEnumerable<T>` from Hub method |
| Streaming upload | `ChannelReader<T>` parameter |
| Connection groups | `Groups.AddToGroupAsync` |
| Heartbeat / keepalive | Built-in (24s default) |

**Production checklist:**
- ✅ Redis backplane для multi-instance
- ✅ Sticky sessions если без backplane
- ✅ JWT auth для WebSockets
- ✅ MessagePack protocol
- ✅ Backpressure handling
- ✅ Reconnection logic в client


---

## Decision tree

```
Real-time нужен?
│
├── 1-to-1 push?
│   ├── Mobile push notifications → APNS/FCM (не SignalR)
│   └── Web → SignalR + UserId
│
├── 1-to-N broadcast?
│   ├── Live updates dashboard → SignalR Groups
│   └── Stock prices, sports score → SignalR + Server-Sent Events backup
│
├── Streaming данных?
│   ├── Server → Client → IAsyncEnumerable<T>
│   ├── Client → Server → ChannelReader<T>
│   └── Bi-directional → Hub methods + Streaming
│
├── Scale?
│   ├── 1 instance, < 5K connections → SignalR без backplane
│   ├── Multi-instance → Redis backplane
│   └── 100K+ connections → Azure SignalR Service
│
├── Альтернативы?
│   ├── Server-Sent Events (SSE) — one-way, проще; в .NET 10 нативно (TypedResults.ServerSentEvents)
│   ├── WebSockets raw — больше control, less abstraction
│   ├── Long polling — fallback для проблемных networks
│   └── gRPC streaming — service-to-service, не browser
│
└── Проще без real-time?
    └── Polling каждые 30 сек, ETag — для слабых updates
```


---

## См. также

- [[pipeline-middleware|Pipeline & Middleware]] — auth pipeline
- [[auth-security|Authentication & Security]] — JWT для SignalR
- [[resilience|Resilience]] — auto-reconnect patterns
- [[graphql|GraphQL]] — subscriptions для real-time alternative
- [[distributed-systems|Distributed Systems]] — pub/sub, MQ
- [[ipc-named-pipes-grpc|IPC: Named Pipes & gRPC]] — alternatives
- [[caching|Caching]] — Redis для backplane
- [[diagnostics-tools|Diagnostics Tools]] — мониторинг

## Reading list

- **Microsoft Docs — SignalR** — learn.microsoft.com/aspnet/core/signalr
- **Microsoft Docs — Azure SignalR Service** — learn.microsoft.com/azure/azure-signalr
- **Marc Gravell — SignalR performance posts** — blog.marcgravell.com
- **Damian Edwards — SignalR talks** — youtube
- **SignalR Client Source Generator** — learn.microsoft.com/aspnet/core/signalr/client-source-generator
- **Andrew Lock — SignalR series** — andrewlock.net
- **MessagePack-CSharp** — github.com/MessagePack-CSharp/MessagePack-CSharp
