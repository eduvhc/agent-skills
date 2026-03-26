---
name: dotnet-native-aot
description: Native AOT compilation for .NET 10 covering project setup, JSON source generators, ASP.NET Core Minimal APIs, SignalR, data access (raw ADO.NET, Dapper.AOT), Docker multi-stage builds, trimming, and performance optimization. Use for any .NET AOT development task.
references:
  - setup
  - json-serialization
  - aspnet-minimal-api
  - signalr
  - data-access
  - docker
  - trimming
  - pitfalls
---

# Native AOT Skill (.NET 10)

Skill for building and deploying Native AOT applications with .NET 10. Covers project configuration, AOT-compatible patterns, serialization, data access, Docker, and common pitfalls.

## Retrieval Sources

Fetch the **latest** information before citing specific API signatures, package versions, or configuration options.

| Source | URL | Use for |
|--------|-----|---------|
| AOT deployment docs | `https://learn.microsoft.com/en-us/dotnet/core/deploying/native-aot/` | Official AOT reference |
| ASP.NET Core AOT | `https://learn.microsoft.com/en-us/aspnet/core/fundamentals/native-aot` | ASP.NET-specific AOT guidance |
| JSON source generators | `https://learn.microsoft.com/en-us/dotnet/standard/serialization/system-text-json/source-generation` | System.Text.Json AOT patterns |
| Trimming options | `https://learn.microsoft.com/en-us/dotnet/core/deploying/trimming/trimming-options` | Feature switches, trim settings |
| AOT warnings | `https://learn.microsoft.com/en-us/dotnet/core/deploying/native-aot/fixing-warnings` | IL2xxx/IL3xxx warning resolution |
| .NET Blog | `https://devblogs.microsoft.com/dotnet/` | New features, announcements |
| Docker images | `https://github.com/dotnet/dotnet-docker` | Container base images for AOT |

When pre-trained knowledge and the docs disagree, **trust the docs**.

## Quick Decision Trees

### "Should I use Native AOT?"

```
Use case?
+-- CLI tool / short-lived process --> YES (big startup win)
+-- Serverless / Functions --> YES (cold start matters)
+-- Microservice with many replicas --> YES (image size matters)
+-- Container-heavy deployment --> YES (smaller images)
+-- JIT disallowed by policy --> YES (only option)
+-- Heavy EF Core usage --> NO (EF Core not AOT-compatible)
+-- MVC / Blazor Server / Razor Pages --> NO (not supported)
+-- Dynamic plugin loading --> NO (no runtime assembly loading)
+-- Peak steady-state throughput priority --> MAYBE (JIT can be faster after warmup)
```

### "Is my dependency AOT-compatible?"

```
Check dependency?
+-- Has [assembly: IsAotCompatible] --> likely safe
+-- Uses reflection-based serialization --> NOT safe
+-- Uses System.Reflection.Emit --> NOT safe
+-- Uses Newtonsoft.Json --> NOT safe (use System.Text.Json)
+-- Uses Dapper --> NOT safe (use Dapper.AOT or raw ADO.NET)
+-- Uses EF Core --> NOT safe
+-- Uses raw ADO.NET (SqliteCommand etc) --> safe
+-- Uses IHttpClientFactory --> safe
+-- Uses System.Text.Json with source generators --> safe
```

### "How do I fix this AOT warning?"

```
Warning code?
+-- IL2026 (RequiresUnreferencedCode) --> add source-generated alternative or suppress with justification
+-- IL2057 (Type.GetType with variable) --> use compile-time type reference
+-- IL2067/IL2070 (missing DynamicallyAccessedMembers) --> add [DynamicallyAccessedMembers] attribute
+-- IL3050 (RequiresDynamicCode) --> replace with non-dynamic pattern
+-- IL3058 (assembly not IsAotCompatible) --> update package or find alternative
```

## Core Patterns

### Minimal AOT csproj

```xml
<PropertyGroup>
    <PublishAot>true</PublishAot>
    <JsonSerializerIsReflectionEnabledByDefault>false</JsonSerializerIsReflectionEnabledByDefault>
</PropertyGroup>
```

### JSON Source Generator

```csharp
[JsonSerializable(typeof(MyResponse))]
[JsonSerializable(typeof(List<MyItem>))]
[JsonSourceGenerationOptions(PropertyNamingPolicy = JsonKnownNamingPolicy.CamelCase)]
internal partial class AppJsonContext : JsonSerializerContext;

// Register in DI
builder.Services.ConfigureHttpJsonOptions(o =>
    o.SerializerOptions.TypeInfoResolverChain.Insert(0, AppJsonContext.Default));
```

### Named Types Required

```csharp
// BAD: anonymous types can't be serialized with AOT
app.MapGet("/api/data", () => new { name = "test", count = 42 });

// GOOD: named record with [JsonSerializable]
record DataResponse(string Name, int Count);
app.MapGet("/api/data", () => new DataResponse("test", 42));
```

### AOT-Safe Data Access

```csharp
using var cmd = connection.CreateCommand();
cmd.CommandText = "SELECT id, name FROM items WHERE id = $id";
cmd.Parameters.AddWithValue("$id", id);
using var reader = cmd.ExecuteReader();
if (reader.Read())
    return new Item(reader.GetInt32(0), reader.GetString(1));
```

### Docker (Alpine, smallest image)

```dockerfile
FROM mcr.microsoft.com/dotnet/sdk:10.0-alpine-aot AS build
WORKDIR /src
COPY . .
RUN dotnet publish -c Release -r linux-musl-x64 -o /app/publish

FROM mcr.microsoft.com/dotnet/runtime-deps:10.0-alpine
WORKDIR /app
COPY --from=build /app/publish .
ENTRYPOINT ["./MyApp"]
```

## Rules

1. **Every type** passed through HTTP endpoints or SignalR must have a `[JsonSerializable]` entry in the context.
2. **No anonymous types** in API responses -- use named records.
3. **No `Newtonsoft.Json`** -- use `System.Text.Json` with source generators.
4. **No EF Core** -- use raw ADO.NET or Dapper.AOT.
5. **No strongly-typed SignalR hubs** (`Hub<T>`) -- use `Hub` with `SendAsync`.
6. **Use `JsonStringEnumConverter<TEnum>`** (generic) not `JsonStringEnumConverter` (non-generic).
7. **Test with `dotnet publish`** -- `dotnet run` uses JIT and won't catch AOT issues.
8. **Zero IL warnings = likely AOT-safe**, but publish is the definitive test.
