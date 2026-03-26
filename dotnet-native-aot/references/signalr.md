# SignalR with Native AOT (.NET 10)

## Status

SignalR gained AOT + trimming support in .NET 9. Listed as "Partially Supported" in .NET 10.

## Configuration

```csharp
// Register JSON source generator for SignalR payloads
builder.Services.Configure<JsonHubProtocolOptions>(o =>
    o.PayloadSerializerOptions.TypeInfoResolverChain.Insert(0, AppJsonContext.Default));

// All types sent/received through SignalR must be registered
[JsonSerializable(typeof(string))]
[JsonSerializable(typeof(ProgressUpdate))]
[JsonSerializable(typeof(CompletedEvent))]
internal partial class AppJsonContext : JsonSerializerContext;
```

## AOT-Compatible Hub Pattern

```csharp
// CORRECT: use Hub (not Hub<T>) with SendAsync
class MyHub : Hub
{
    public async Task SendProgress(string url, double progress)
    {
        await Clients.All.SendAsync("ReceiveProgress", url, progress);
    }

    public async Task BroadcastMessage(ChatMessage message)
    {
        await Clients.All.SendAsync("NewMessage", message);
    }
}

app.MapHub<MyHub>("/hub");
```

## What Does NOT Work

```csharp
// WRONG: strongly-typed hubs are NOT supported with AOT
public interface IMyClient
{
    Task ReceiveProgress(string url, double progress);
}

class MyHub : Hub<IMyClient>  // <-- will produce warnings + runtime crash
{
    public async Task SendProgress(string url, double progress)
    {
        await Clients.All.ReceiveProgress(url, progress);  // uses reflection
    }
}
```

## Limitations

- **JSON protocol only** -- MessagePack is NOT supported
- **No `Hub<T>`** (strongly-typed hubs) -- runtime exceptions
- **No `IAsyncEnumerable<T>` or `ChannelReader<T>` with value types** as hub method parameters
- Only `Task`, `Task<T>`, `ValueTask`, `ValueTask<T>` for async return types

## Using IHubContext (AOT-safe)

```csharp
class MyService
{
    private readonly IHubContext<MyHub> _hub;

    public MyService(IHubContext<MyHub> hub) => _hub = hub;

    public async Task NotifyAll(CompletedEvent data)
    {
        // SendAsync with string method name -- AOT-safe
        await _hub.Clients.All.SendAsync("Completed", data);
    }
}
```
