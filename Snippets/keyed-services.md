# Keyed Services in .NET 8+

> Register and resolve multiple implementations of the same interface by key.

## Registration

```csharp
builder.Services.AddKeyedScoped<INotificationSender, EmailSender>("email");
builder.Services.AddKeyedScoped<INotificationSender, SmsSender>("sms");
builder.Services.AddKeyedScoped<INotificationSender, PushSender>("push");
```

## Injection

```csharp
public sealed class OrderEndpoints
{
    public static void Map(IEndpointRouteBuilder app)
    {
        app.MapPost("/orders/{id}/notify", async (
            int id,
            [FromQuery] string channel,
            [FromKeyedServices("email")] INotificationSender fallback,
            IServiceProvider sp,
            CancellationToken ct) =>
        {
            var sender = sp.GetKeyedService<INotificationSender>(channel) ?? fallback;
            await sender.SendAsync(id, ct);
            return TypedResults.NoContent();
        });
    }
}
```

## When to Use

| Before (Factory) | After (Keyed) |
|---|---|
| `INotificationSenderFactory.Create("email")` | `[FromKeyedServices("email")] INotificationSender` |
| Manual `switch`/`if` chains | DI container resolves by key |
| Extra factory interface + impl | Zero boilerplate |

Keyed Services replace the factory pattern for most cases in .NET 8+.
