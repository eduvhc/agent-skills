# ASP.NET Core Minimal APIs with Native AOT

## Compatibility (.NET 10)

| Feature | AOT Status |
|---------|------------|
| Minimal APIs | Partially Supported |
| gRPC | Fully Supported |
| SignalR | Partially Supported |
| JWT Authentication | Fully Supported |
| CORS, HealthChecks, StaticFiles, WebSockets | Fully Supported |
| OutputCaching, RateLimiting, HttpLogging | Fully Supported |
| MVC / Controllers | NOT Supported |
| Blazor Server | NOT Supported |
| Razor Pages | NOT Supported |
| Session, SPA | NOT Supported |

## Builder Options

### CreateSlimBuilder (recommended for AOT)

```csharp
var builder = WebApplication.CreateSlimBuilder(args);
```

Includes: JSON config (appsettings.json), user secrets, console logging.

Excludes: Hosting startup assemblies, IIS integration, UseStaticWebAssets, EventLog/Debug/EventSource logging, HTTP/3 (Quic), Regex routing constraints.

To add back HTTPS/HTTP3:
```csharp
builder.WebHost.UseKestrelHttpsConfiguration();  // HTTPS
builder.WebHost.UseQuic();                        // HTTP/3
```

### CreateBuilder (standard, also works with AOT)

```csharp
var builder = WebApplication.CreateBuilder(args);
```

Includes everything. Larger binary but fully featured.

### CreateEmptyBuilder (absolute minimum)

Omits JSON configuration, user secrets, console logging. For extreme optimization.

## Complete AOT Example

```csharp
using System.Text.Json.Serialization;

var builder = WebApplication.CreateSlimBuilder(args);
builder.Services.ConfigureHttpJsonOptions(o =>
    o.SerializerOptions.TypeInfoResolverChain.Insert(0, AppJsonContext.Default));

var app = builder.Build();

app.MapGet("/todos", () => new Todo[] { new(1, "Walk dog", false) });
app.MapGet("/todos/{id}", (int id) => new Todo(id, "Sample", false));
app.MapPost("/todos", (Todo todo) => Results.Created($"/todos/{todo.Id}", todo));

app.Run();

record Todo(int Id, string Title, bool IsComplete);

[JsonSerializable(typeof(Todo))]
[JsonSerializable(typeof(Todo[]))]
[JsonSourceGenerationOptions(PropertyNamingPolicy = JsonKnownNamingPolicy.CamelCase)]
internal partial class AppJsonContext : JsonSerializerContext;
```

## Key Rules

1. **Named types only** -- no anonymous types in responses
2. **All response types** must be in `JsonSerializerContext`
3. **No `[FromBody] dynamic`** -- use typed models
4. **No `[FromServices]`** -- use parameter injection (works in .NET 10)
5. **Static files work** -- `UseStaticFiles()` is AOT-compatible
6. **`Results.Ok(obj)` works** -- as long as `obj` type is registered

## .NET 10: webapiaot Template

```bash
dotnet new webapiaot
dotnet new webapiaot --no-openapi  # without OpenAPI
```

Generates a ready-to-go AOT project with JSON source generators configured.
