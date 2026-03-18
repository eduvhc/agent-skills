# Instrumentation Libraries Reference

## ASP.NET Core (Stable)

Package: `OpenTelemetry.Instrumentation.AspNetCore`

```csharp
.WithTracing(t => t
    .AddAspNetCoreInstrumentation(options =>
    {
        // Filter health check noise
        options.Filter = ctx =>
            !ctx.Request.Path.StartsWithSegments("/health") &&
            !ctx.Request.Path.StartsWithSegments("/alive");

        // Enrich from request/response
        options.EnrichWithHttpRequest  = (activity, request) =>
            activity.SetTag("app.tenant", request.Headers["X-Tenant-Id"].ToString());
        options.EnrichWithHttpResponse = (activity, response) =>
            activity.SetTag("app.response_size", response.ContentLength);

        // Record unhandled exceptions as span events
        options.RecordException = true;
    }))
.WithMetrics(m => m
    .AddAspNetCoreInstrumentation())   // http.server.request.duration, http.server.active_requests
```

Produces spans for every incoming request following HTTP server semantic conventions.
Metrics produced: `http.server.request.duration`, `http.server.active_requests`.

## HttpClient (Stable)

Package: `OpenTelemetry.Instrumentation.Http`

```csharp
.WithTracing(t => t
    .AddHttpClientInstrumentation(options =>
    {
        // Exclude telemetry self-calls
        options.FilterHttpRequestMessage = req =>
            req.RequestUri?.Host != "telemetry.internal";

        options.EnrichWithHttpRequestMessage  = (activity, req) =>
            activity.SetTag("app.service_target", req.RequestUri?.Host);
        options.EnrichWithHttpResponseMessage = (activity, res) =>
            activity.SetTag("app.response_size",  res.Content.Headers.ContentLength);

        options.RecordException = true;
    }))
```

> On .NET 9+, native HTTP instrumentation exists in the runtime, but `AddHttpClientInstrumentation()` is still required for: context propagation injection, enrichment/filtering, and full attribute coverage.

## Runtime Metrics (Stable)

Package: `OpenTelemetry.Instrumentation.Runtime`

```csharp
.WithMetrics(m => m.AddRuntimeInstrumentation())
```

Captures: GC collections, GC heap size, thread pool queue depth, thread pool workers, JIT compilations, exceptions per second, lock contention. On .NET 9+, bridges to `System.Runtime` built-in meters.

## Entity Framework Core (Beta)

Package: `OpenTelemetry.Instrumentation.EntityFrameworkCore` (prerelease)

```csharp
.WithTracing(t => t
    .AddEntityFrameworkCoreInstrumentation(options =>
    {
        // Filter to specific DB providers
        options.Filter = (providerName, command) =>
            providerName?.Contains("Npgsql", StringComparison.OrdinalIgnoreCase) == true;

        // Enrich with custom data
        options.EnrichWithIDbCommand = (activity, cmd) =>
            activity.SetTag("db.connection_string_hash",
                cmd.Connection?.ConnectionString.GetHashCode().ToString());

        // Enable query text (dev only — may contain sensitive data)
        // Set via: OTEL_DOTNET_EXPERIMENTAL_EFCORE_ENABLE_TRACE_DB_QUERY_PARAMETERS=true
    }))
```

**Limitations:** relational DBs only — does NOT support Azure Cosmos DB. Status: beta.

## Npgsql Native (Preferred for Postgres)

Package: `Npgsql.OpenTelemetry` v10.0.1

```csharp
// Simple
.WithTracing(t => t.AddNpgsql())

// With filtering
var dataSource = new NpgsqlDataSourceBuilder(connectionString)
    .ConfigureTracing(options =>
    {
        options.Filter = cmd => !cmd.CommandText.StartsWith("SET ");
    })
    .Build();
```

**Prefer over EF Core instrumentation for Postgres** — more precise, stable, Npgsql-version aligned.

## StackExchange.Redis (Beta)

Package: `OpenTelemetry.Instrumentation.StackExchangeRedis` (prerelease)

```csharp
var redis = ConnectionMultiplexer.Connect("localhost:6379");

.WithTracing(t => t
    .AddRedisInstrumentation(redis, options =>
    {
        options.EnrichActivityWithTimingEvents = true;
        options.SetVerboseDatabaseStatements   = true;   // include command text
    }))
```

## gRPC Client (Beta)

Package: `OpenTelemetry.Instrumentation.GrpcNetClient` (prerelease)

```csharp
.WithTracing(t => t
    .AddGrpcClientInstrumentation(options =>
    {
        // Prevents double-counting when HttpClient instrumentation is also active
        options.SuppressDownstreamInstrumentation = true;
    }))
```

## SQL Client / ADO.NET (Beta)

Package: `OpenTelemetry.Instrumentation.SqlClient` (prerelease)

```csharp
.WithTracing(t => t
    .AddSqlClientInstrumentation(options =>
    {
        options.RecordException                    = true;
        options.EnableConnectionLevelAttributes    = true;
        options.SetDbStatementForText              = true;   // include query text (dev only)
    }))
```

## Auto-Instrumentation (Zero-Code)

GitHub: https://github.com/open-telemetry/opentelemetry-dotnet-instrumentation

Use when source changes are impossible (third-party apps, legacy code, no-code policy). Works via a CLR profiler + startup hook — no NuGet packages needed in the target app.

### When to Use Auto vs Manual

| Scenario | Recommendation |
|---|---|
| Greenfield .NET 10 app | Manual — more control, better performance, custom spans |
| Legacy app you don't own | Auto — zero source changes |
| Third-party commercial app | Auto — only option |
| Kubernetes + OTel Operator | Auto via operator injection |
| Sidecar / fleet rollout | Auto — consistent baseline across all services |
| Need custom business spans | Manual (or auto + manual hybrid) |

### Installation

```bash
# Linux/macOS — install script
curl -sSfL https://github.com/open-telemetry/opentelemetry-dotnet-instrumentation/releases/latest/download/otel-dotnet-auto-install.sh | bash

# Windows PowerShell
$module_url = "https://github.com/open-telemetry/opentelemetry-dotnet-instrumentation/releases/latest/download/OpenTelemetry.DotNet.Auto.psm1"
Invoke-WebRequest -Uri $module_url -OutFile OpenTelemetry.DotNet.Auto.psm1
Import-Module .\OpenTelemetry.DotNet.Auto.psm1
Install-OpenTelemetryCore

# Docker — use base image with agent pre-installed
FROM otel/dotnet-auto-instrumentation:latest AS otel-auto
FROM mcr.microsoft.com/dotnet/aspnet:10.0
COPY --from=otel-auto /autoinstrumentation /autoinstrumentation
```

### Environment Variables

```bash
# Required — tell CLR where the profiler lives
OTEL_DOTNET_AUTO_HOME=/autoinstrumentation
CORECLR_ENABLE_PROFILING=1
CORECLR_PROFILER={918728DD-259F-4A6A-AC2B-B85E1B658318}
CORECLR_PROFILER_PATH=/autoinstrumentation/linux-x64/OpenTelemetry.AutoInstrumentation.Native.so
DOTNET_ADDITIONAL_DEPS=/autoinstrumentation/AdditionalDeps
DOTNET_SHARED_STORE=/autoinstrumentation/store
DOTNET_STARTUP_HOOKS=/autoinstrumentation/net/OpenTelemetry.AutoInstrumentation.StartupHook.dll

# Standard OTel config
OTEL_SERVICE_NAME=my-service
OTEL_EXPORTER_OTLP_ENDPOINT=http://collector:4317
OTEL_RESOURCE_ATTRIBUTES=deployment.environment=production

# Profiler paths by OS/arch
# Linux x64:   OpenTelemetry.AutoInstrumentation.Native.so
# Linux arm64: OpenTelemetry.AutoInstrumentation.Native.so (arm64 subfolder)
# Windows x64: OpenTelemetry.AutoInstrumentation.Native.dll
# macOS:       OpenTelemetry.AutoInstrumentation.Native.dylib
```

### What Gets Auto-Instrumented

**Traces (stable):**
- ASP.NET Core (incoming HTTP requests)
- HttpClient / HttpWebRequest (outgoing HTTP)
- gRPC server and client (`Grpc.AspNetCore`, `Grpc.Net.Client`)
- SqlClient (ADO.NET for SQL Server)
- Entity Framework Core (SQL queries)
- StackExchange.Redis
- MongoDB.Driver
- Npgsql (via `Npgsql.OpenTelemetry` detection)
- RabbitMQ.Client 6.x (bytecode patching — 7.x has native support)
- MassTransit, NServiceBus

**Metrics (stable):**
- ASP.NET Core metrics
- HttpClient metrics
- .NET runtime metrics

**Logs:**
- NLog, Serilog, log4net (bridges to OTel logging pipeline)
- `Microsoft.Extensions.Logging` (ILogger)

### Enabling/Disabling Specific Instrumentations

```bash
# Disable a specific instrumentation by name
OTEL_DOTNET_AUTO_TRACES_INSTRUMENTATION_ENABLED=true
OTEL_DOTNET_AUTO_TRACES_SQLCLIENT_INSTRUMENTATION_ENABLED=false   # disable SqlClient only

# Enable experimental instrumentations (off by default)
OTEL_DOTNET_AUTO_TRACES_ADDITIONAL_SOURCES=MyApp.CustomSource

# Add your own ActivitySource names to be collected
OTEL_DOTNET_AUTO_TRACES_ADDITIONAL_SOURCES=MyApp.Orders,MyApp.Payments
```

### Kubernetes — OTel Operator Injection

With the [OpenTelemetry Operator](https://github.com/open-telemetry/opentelemetry-operator), auto-instrumentation is injected via pod annotations — no Dockerfile changes needed:

```yaml
# 1. Create an Instrumentation resource
apiVersion: opentelemetry.io/v1alpha1
kind: Instrumentation
metadata:
  name: my-instrumentation
spec:
  exporter:
    endpoint: http://otel-collector:4317
  propagators:
    - tracecontext
    - baggage
  sampler:
    type: parentbased_traceidratio
    argument: "0.1"
  dotnet:
    env:
      - name: OTEL_DOTNET_AUTO_TRACES_ADDITIONAL_SOURCES
        value: "MyApp.*"

---
# 2. Annotate pods to inject the agent
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-api
spec:
  template:
    metadata:
      annotations:
        instrumentation.opentelemetry.io/inject-dotnet: "true"
```

The operator mounts the auto-instrumentation volume and sets all required env vars automatically.

### Auto-Instrumentation Limitations vs Manual

| Limitation | Impact |
|---|---|
| Cannot create custom business spans | No domain-level tracing without adding code |
| No `IsAllDataRequested` checks | Slightly higher overhead for dropped spans |
| Tag enrichment requires env var config only | Less flexible than code-based enrichment |
| CLR profiler adds ~10-50ms startup overhead | Negligible for long-running services |
| Profiler path is OS/arch specific | Dockerfile must target correct platform |
| Does not instrument `System.Net.Sockets` or raw TCP | Only high-level library calls |

### Hybrid: Auto + Manual Custom Spans

Auto-instrumentation handles the baseline; you add domain spans on top:

```bash
# Tell auto-instrumentation to also collect your custom ActivitySource
OTEL_DOTNET_AUTO_TRACES_ADDITIONAL_SOURCES=MyApp.Orders
```

```csharp
// Your code — no SDK packages needed in the project; ActivitySource is in System.Diagnostics
private static readonly ActivitySource _source = new("MyApp.Orders");

public async Task ProcessOrderAsync(Order order)
{
    using var activity = _source.StartActivity("ProcessOrder", ActivityKind.Internal);
    activity?.SetTag("order.id", order.Id);
    // auto-instrumentation captures the parent HttpClient/ASP.NET Core span automatically
}
```
