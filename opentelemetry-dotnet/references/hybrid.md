# Hybrid Instrumentation: Auto Agent + Manual Custom Spans

Hybrid means running the **OpenTelemetry .NET Auto-Instrumentation agent** (CLR Profiler + StartupHook) to get baseline coverage of all supported libraries, while adding **manual `ActivitySource` spans** for domain/business logic — without needing the OTel SDK in the target project for the manual spans.

Auto-instrumentation agent version: **1.14.1** (supports .NET 10).

## Why Hybrid?

| Approach | Coverage | Custom spans | Code changes |
|---|---|---|---|
| Auto only | All supported libs | None | Zero |
| Manual only | What you wire up | Full control | Significant |
| **Hybrid** | **All supported libs** | **Full control** | **Minimal** |

The auto agent handles the boring infra spans (HTTP, DB, queue receive). You add the spans that matter to your domain: `ProcessOrder`, `RunPricingEngine`, `ValidateTenant`.

## The One Key Variable

```bash
# Tell the auto agent to collect your custom ActivitySource names
OTEL_DOTNET_AUTO_TRACES_ADDITIONAL_SOURCES=MyApp.Orders,MyApp.Payments,MyApp.*
```

Your domain code needs only `System.Diagnostics` — no OTel SDK NuGet packages required:

```csharp
// No OTel NuGet needed — System.Diagnostics is in the runtime
private static readonly ActivitySource _source = new("MyApp.Orders");

public async Task ProcessOrderAsync(Order order)
{
    using var activity = _source.StartActivity("ProcessOrder", ActivityKind.Internal);
    activity?.SetTag("order.id",   order.Id);
    activity?.SetTag("order.tier", order.Tier);
    // auto agent spans (HttpClient, DB) are automatically children of this span
}
```

## Plugin API — Enrichment Without Code Changes

The Plugin API lets you configure the auto agent from code (a compiled DLL) without modifying application source. Use this for enrichment, filtering, or adding custom SDK configuration.

```bash
OTEL_DOTNET_AUTO_PLUGINS=MyApp.OTelPlugin, MyApp.OTelPlugin.Assembly
```

Duck-typed interface — no base class needed, just matching method names:

```csharp
public class MyOTelPlugin
{
    // Called before the TracerProvider is built — add sources, samplers, processors
    public void BeforeConfigureTracerProvider(TracerProviderBuilder builder)
    {
        builder.AddSource("MyApp.*");
        builder.SetSampler(new ParentBasedSampler(new TraceIdRatioBasedSampler(0.1)));
    }

    // Called after the TracerProvider is built — access final provider
    public void AfterConfigureTracerProvider(TracerProvider provider) { }

    // Typed enrichment for specific instrumentation options
    public void ConfigureTracesOptions(HttpClientTraceInstrumentationOptions options)
    {
        options.FilterHttpRequestMessage = req => req.RequestUri?.Host != "telemetry.internal";
    }

    public void ConfigureTracesOptions(EntityFrameworkInstrumentationOptions options)
    {
        // options.SetDbStatementForText = true;  // dev only
    }

    // Resource enrichment
    public void ConfigureResource(ResourceBuilder builder)
    {
        builder.AddAttributes(new Dictionary<string, object>
        {
            ["deployment.environment"] = Environment.GetEnvironmentVariable("ASPNETCORE_ENVIRONMENT") ?? "production",
            ["team.name"] = "platform",
        });
    }
}
```

> Plugin DLL must target the same .NET TFM as the application and be placed in a path accessible to the agent.

## Per-Library Hybrid Patterns

### RabbitMQ

**Auto agent coverage:**
- **RabbitMQ.Client v5/v6** → bytecode patching (spans created without source changes)
- **RabbitMQ.Client v7+** → native `ActivitySource` support built into the library

```bash
# v7+ — tell the agent to collect the native source
OTEL_DOTNET_AUTO_TRACES_ADDITIONAL_SOURCES=RabbitMQ.Client
```

**Manual spans for processing logic (all versions):**

```csharp
private static readonly ActivitySource _source = new("MyApp.Messaging");

public async Task HandleOrderCreatedAsync(BasicDeliverEventArgs ea)
{
    // Extract W3C context from message headers
    var headers = ea.BasicProperties.Headers;
    var extractedCtx = Propagators.DefaultTextMapPropagator.Extract(
        default,
        headers,
        (h, key) => h.TryGetValue(key, out var v)
            ? new[] { Encoding.UTF8.GetString((byte[])v) }
            : Array.Empty<string>());

    using var activity = _source.StartActivity(
        "order.process",
        ActivityKind.Consumer,
        extractedCtx.ActivityContext);

    activity?.SetTag("messaging.system",      "rabbitmq");
    activity?.SetTag("messaging.destination", ea.RoutingKey);
    activity?.SetTag("order.id",              ReadOrderId(ea.Body));

    await ProcessOrderAsync(activity);
}
```

**Alternative:** `RabbitMQ.Client.OpenTelemetry` NuGet package provides `AddRabbitMQClientInstrumentation()` for manual SDK setups.

### Kafka / Confluent

**Auto agent coverage:** Kafka support is **alpha** and version-gated — verify your `Confluent.Kafka` version is supported before relying on the agent.

**Recommended for Kafka: use the manual instrumentation package:**

```xml
<!-- Alpha package — pin the version -->
<PackageReference Include="OpenTelemetry.Instrumentation.ConfluentKafka" Version="0.1.0-alpha.5" />
```

```csharp
// Producer
var producerBuilder = new ProducerBuilder<string, string>(config);
producerBuilder.SetOTelInstrumentation();  // extension from the package

// Consumer
var consumerBuilder = new ConsumerBuilder<string, string>(config);
consumerBuilder.SetOTelInstrumentation();
```

**Community alternative** (more stable): `Confluent.Kafka.Extensions.OpenTelemetry` v0.4.0:

```csharp
.WithTracing(t => t.AddConfluentKafkaInstrumentation(producer, consumer))
```

**Hybrid custom spans for message processing:**

```csharp
private static readonly ActivitySource _source = new("MyApp.Kafka");

public async Task ConsumeAsync(ConsumeResult<string, string> result)
{
    // Extract W3C context from Kafka headers
    var extractedCtx = Propagators.DefaultTextMapPropagator.Extract(
        default,
        result.Message.Headers,
        (headers, key) =>
        {
            var header = headers.FirstOrDefault(h => h.Key == key);
            return header != null
                ? new[] { Encoding.UTF8.GetString(header.GetValueBytes()) }
                : Array.Empty<string>();
        });

    using var activity = _source.StartActivity(
        $"{result.Topic} process",
        ActivityKind.Consumer,
        extractedCtx.ActivityContext);

    activity?.SetTag("messaging.system",           "kafka");
    activity?.SetTag("messaging.destination",      result.Topic);
    activity?.SetTag("messaging.kafka.partition",  result.Partition.Value);
    activity?.SetTag("messaging.kafka.offset",     result.Offset.Value);
}
```

### PostgreSQL / Npgsql

**Auto agent coverage:** Full — the agent detects `Npgsql.OpenTelemetry` and activates it automatically.

```bash
# Enable query text capture (dev only — may expose sensitive data)
OTEL_DOTNET_EXPERIMENTAL_NPGSQL_SET_DB_STATEMENT=true
```

**Hybrid enrichment via `ConfigureTracing()` (Npgsql 9+):**

```csharp
var dataSource = new NpgsqlDataSourceBuilder(connectionString)
    .ConfigureTracing(options =>
    {
        // Filter out noisy SET commands
        options.Filter = cmd => !cmd.CommandText.StartsWith("SET ");
    })
    .Build();
```

**Or via Plugin API (zero source changes):**

```csharp
public void ConfigureTracesOptions(NpgsqlTraceInstrumentationOptions options)
{
    options.Filter = cmd => !cmd.CommandText.StartsWith("SET ");
}
```

**Custom spans for business logic wrapping DB calls:**

```csharp
private static readonly ActivitySource _source = new("MyApp.Orders");

public async Task<Order> CreateOrderAsync(OrderRequest request)
{
    using var activity = _source.StartActivity("orders.create", ActivityKind.Internal);
    activity?.SetTag("order.tenant", request.TenantId);

    // Npgsql spans (auto or via AddNpgsql()) are automatically children
    await _db.Orders.AddAsync(new Order(request));
    await _db.SaveChangesAsync();

    activity?.SetTag("order.id", order.Id);
    return order;
}
```

### Redis / StackExchange.Redis

**Auto agent coverage:** StackExchange.Redis v2.6–3.0 is covered by the agent.

**Hybrid enrichment via Plugin API:**

```csharp
public void ConfigureTracesOptions(StackExchangeRedisInstrumentationOptions options)
{
    options.SetVerboseDatabaseStatements = true;  // include command text (dev only)
    options.EnrichActivityWithTimingEvents = true;
}
```

**Custom spans for cache-aside patterns:**

```csharp
private static readonly ActivitySource _source = new("MyApp.Cache");

public async Task<UserProfile> GetProfileAsync(string userId)
{
    using var activity = _source.StartActivity("cache.get_profile", ActivityKind.Internal);
    activity?.SetTag("cache.key_prefix", "profile");
    activity?.SetTag("user.id.hash",     userId.GetHashCode().ToString()); // avoid PII

    var cached = await _cache.StringGetAsync($"profile:{userId}");
    if (cached.HasValue)
    {
        activity?.SetTag("cache.hit", true);
        return Deserialize<UserProfile>(cached);
    }

    activity?.SetTag("cache.hit", false);
    var profile = await _db.GetProfileAsync(userId);
    await _cache.StringSetAsync($"profile:{userId}", Serialize(profile), TimeSpan.FromMinutes(5));
    return profile;
}
```

### Entity Framework Core

**Auto agent coverage:** EF Core instrumentation is included in the agent.

```bash
# Enable query text (dev only — may contain sensitive data)
OTEL_DOTNET_AUTO_ENTITYFRAMEWORKCORE_SET_DB_STATEMENT_FOR_TEXT=true
```

**Hybrid enrichment via Plugin API:**

```csharp
public void ConfigureTracesOptions(EntityFrameworkInstrumentationOptions options)
{
    options.Filter = (providerName, command) =>
        providerName?.Contains("Npgsql", StringComparison.OrdinalIgnoreCase) == true;

    options.EnrichWithIDbCommand = (activity, cmd) =>
        activity.SetTag("db.connection_hash", cmd.Connection?.ConnectionString.GetHashCode().ToString());
}
```

> For Postgres, prefer `Npgsql.OpenTelemetry` over EF Core instrumentation — more precise, stable, Npgsql-version aligned.

### MassTransit v8+

**Auto agent coverage:** MassTransit is **NOT in the auto agent**. MassTransit v8+ has built-in native `ActivitySource` support — register its sources explicitly.

```bash
# Register MassTransit's built-in ActivitySource
OTEL_DOTNET_AUTO_TRACES_ADDITIONAL_SOURCES=MassTransit
```

**Or with the OTel SDK:**

```csharp
.WithTracing(t => t
    .AddSource("MassTransit")
    .AddMeter(InstrumentationOptions.MeterName))   // "MassTransit" metrics meter
```

**Custom spans for consumer business logic:**

```csharp
private static readonly ActivitySource _source = new("MyApp.Orders");

public class OrderCreatedConsumer : IConsumer<OrderCreated>
{
    public async Task Consume(ConsumeContext<OrderCreated> context)
    {
        // MassTransit span is the parent — our span is a child
        using var activity = _source.StartActivity("order.process_created");
        activity?.SetTag("order.id",     context.Message.OrderId);
        activity?.SetTag("order.tenant", context.Message.TenantId);

        await ProcessAsync(context.Message);
    }
}
```

**MassTransit header propagation (manual queues/transports):**

```csharp
// MassTransit injects W3C headers automatically for supported transports.
// For custom transports, use DiagnosticHeaders:
context.Headers.Set(DiagnosticHeaders.TraceParent, Activity.Current?.Id);
context.Headers.Set(DiagnosticHeaders.TraceState,  Activity.Current?.TraceStateString);
```

### Azure Service Bus

**Auto agent coverage:** Azure Service Bus is **NOT in the auto agent**. Register the Azure SDK's `ActivitySource` manually.

```bash
# Azure SDK emits spans under "Azure.Messaging.ServiceBus.*"
OTEL_DOTNET_AUTO_TRACES_ADDITIONAL_SOURCES=Azure.Messaging.ServiceBus.*
```

**Or with the OTel SDK:**

```csharp
.WithTracing(t => t.AddSource("Azure.Messaging.ServiceBus.*"))
```

**Manual W3C propagation for custom processors:**

```csharp
private static readonly ActivitySource _source = new("MyApp.ServiceBus");

// --- Sender ---
public async Task SendAsync<T>(T message)
{
    var carrier = new Dictionary<string, string>();
    Propagators.DefaultTextMapPropagator.Inject(
        new PropagationContext(Activity.Current?.Context ?? default, Baggage.Current),
        carrier,
        (dict, key, value) => dict[key] = value);

    var sbMessage = new ServiceBusMessage(BinaryData.FromObjectAsJson(message));
    foreach (var (key, value) in carrier)
        sbMessage.ApplicationProperties[key] = value;

    await _sender.SendMessageAsync(sbMessage);
}

// --- Processor ---
public async Task ProcessMessageAsync(ProcessMessageEventArgs args)
{
    var extractedCtx = Propagators.DefaultTextMapPropagator.Extract(
        default,
        args.Message.ApplicationProperties,
        (props, key) => props.TryGetValue(key, out var v) ? new[] { v?.ToString() ?? "" } : Array.Empty<string>());

    using var activity = _source.StartActivity(
        $"{args.Message.Subject} process",
        ActivityKind.Consumer,
        extractedCtx.ActivityContext);

    activity?.SetTag("messaging.system",      "servicebus");
    activity?.SetTag("messaging.destination", args.EntityPath);
    activity?.SetTag("messaging.message_id",  args.Message.MessageId);

    await HandleAsync(args.Message);
}
```

### gRPC

**Auto agent coverage:** `Grpc.Net.Client` v2.52–3.0 client calls are covered. Server-side is handled by ASP.NET Core instrumentation.

```bash
# No extra config needed — agent covers it automatically
# Server side is covered by ASP.NET Core instrumentation
```

**Hybrid — prevent double-counting when using both gRPC client and HttpClient instrumentation:**

```csharp
.WithTracing(t => t
    .AddGrpcClientInstrumentation(options =>
    {
        // Suppresses the underlying HttpClient span so you don't get duplicates
        options.SuppressDownstreamInstrumentation = true;
    }))
```

**Custom spans for gRPC service business logic:**

```csharp
private static readonly ActivitySource _source = new("MyApp.Grpc");

public override async Task<OrderResponse> CreateOrder(
    OrderRequest request, ServerCallContext context)
{
    using var activity = _source.StartActivity("grpc.order.create", ActivityKind.Internal);
    activity?.SetTag("order.tenant", request.TenantId);
    // ASP.NET Core instrumentation already created the server span as parent

    return await _service.CreateAsync(request);
}
```

## Docker / Container Patterns

### Shell Script (Linux)

```dockerfile
FROM mcr.microsoft.com/dotnet/aspnet:10.0 AS base

RUN curl -sSfL https://github.com/open-telemetry/opentelemetry-dotnet-instrumentation/releases/download/v1.14.1/otel-dotnet-auto-install.sh \
    | bash

ENV OTEL_DOTNET_AUTO_HOME=/root/.otel-dotnet-auto
ENV CORECLR_ENABLE_PROFILING=1
ENV CORECLR_PROFILER={918728DD-259F-4A6A-AC2B-B85E1B658318}
ENV CORECLR_PROFILER_PATH=/root/.otel-dotnet-auto/linux-x64/OpenTelemetry.AutoInstrumentation.Native.so
ENV DOTNET_ADDITIONAL_DEPS=/root/.otel-dotnet-auto/AdditionalDeps
ENV DOTNET_SHARED_STORE=/root/.otel-dotnet-auto/store
ENV DOTNET_STARTUP_HOOKS=/root/.otel-dotnet-auto/net/OpenTelemetry.AutoInstrumentation.StartupHook.dll
```

### NuGet Package (Cross-Platform, Preferred)

```xml
<PackageReference Include="OpenTelemetry.AutoInstrumentation" Version="1.14.1" />
```

```dockerfile
FROM mcr.microsoft.com/dotnet/aspnet:10.0

COPY --from=publish /app .

# OTEL_DOTNET_AUTO_HOME is set by the package build output
ENV CORECLR_ENABLE_PROFILING=1
ENV CORECLR_PROFILER={918728DD-259F-4A6A-AC2B-B85E1B658318}
ENV CORECLR_PROFILER_PATH=/app/otel-dotnet-auto/linux-x64/OpenTelemetry.AutoInstrumentation.Native.so
ENV DOTNET_ADDITIONAL_DEPS=/app/otel-dotnet-auto/AdditionalDeps
ENV DOTNET_SHARED_STORE=/app/otel-dotnet-auto/store
ENV DOTNET_STARTUP_HOOKS=/app/otel-dotnet-auto/net/OpenTelemetry.AutoInstrumentation.StartupHook.dll
```

> The NuGet approach copies agent files into the publish output — no download script needed at container build time.

### Pre-Built Base Image

```dockerfile
FROM otel/dotnet-auto-instrumentation:1.14.1 AS otel-auto
FROM mcr.microsoft.com/dotnet/aspnet:10.0

COPY --from=otel-auto /autoinstrumentation /autoinstrumentation
COPY --from=publish /app /app

ENV OTEL_DOTNET_AUTO_HOME=/autoinstrumentation
ENV CORECLR_ENABLE_PROFILING=1
ENV CORECLR_PROFILER={918728DD-259F-4A6A-AC2B-B85E1B658318}
ENV CORECLR_PROFILER_PATH=/autoinstrumentation/linux-x64/OpenTelemetry.AutoInstrumentation.Native.so
ENV DOTNET_ADDITIONAL_DEPS=/autoinstrumentation/AdditionalDeps
ENV DOTNET_SHARED_STORE=/autoinstrumentation/store
ENV DOTNET_STARTUP_HOOKS=/autoinstrumentation/net/OpenTelemetry.AutoInstrumentation.StartupHook.dll
```

## Hybrid Environment Variable Reference

```bash
# ── Core agent ──
OTEL_DOTNET_AUTO_HOME=/autoinstrumentation
CORECLR_ENABLE_PROFILING=1
CORECLR_PROFILER={918728DD-259F-4A6A-AC2B-B85E1B658318}
CORECLR_PROFILER_PATH=/autoinstrumentation/linux-x64/OpenTelemetry.AutoInstrumentation.Native.so
DOTNET_ADDITIONAL_DEPS=/autoinstrumentation/AdditionalDeps
DOTNET_SHARED_STORE=/autoinstrumentation/store
DOTNET_STARTUP_HOOKS=/autoinstrumentation/net/OpenTelemetry.AutoInstrumentation.StartupHook.dll

# ── Hybrid: your custom ActivitySource names ──
OTEL_DOTNET_AUTO_TRACES_ADDITIONAL_SOURCES=MyApp.Orders,MyApp.Payments

# ── Plugin API ──
OTEL_DOTNET_AUTO_PLUGINS=MyCompany.OTelPlugin, MyCompany.OTelPlugin.Assembly

# ── Disable specific auto-instrumented libraries ──
OTEL_DOTNET_AUTO_TRACES_SQLCLIENT_INSTRUMENTATION_ENABLED=false
OTEL_DOTNET_AUTO_TRACES_ENTITYFRAMEWORKCORE_INSTRUMENTATION_ENABLED=false

# ── Library-specific dev options ──
OTEL_DOTNET_EXPERIMENTAL_NPGSQL_SET_DB_STATEMENT=true
OTEL_DOTNET_AUTO_ENTITYFRAMEWORKCORE_SET_DB_STATEMENT_FOR_TEXT=true

# ── Service identity ──
OTEL_SERVICE_NAME=my-api
OTEL_RESOURCE_ATTRIBUTES=service.version=1.2.3,deployment.environment=production

# ── Export ──
OTEL_EXPORTER_OTLP_ENDPOINT=http://otel-collector:4317
OTEL_EXPORTER_OTLP_PROTOCOL=grpc
```

## Library Support Matrix

| Library | Auto Agent | How | Hybrid Custom Spans | Notes |
|---|---|---|---|---|
| ASP.NET Core | ✅ Stable | Bytecode | Domain handlers | |
| HttpClient | ✅ Stable | Bytecode | Domain wrappers | |
| Npgsql / Postgres | ✅ Stable | Via Npgsql.OTel | Business logic spans | Use `ConfigureTracing()` for enrichment |
| EF Core | ✅ Stable | Bytecode | Repo/service layer | Prefer Npgsql for Postgres |
| StackExchange.Redis | ✅ Stable | Bytecode | Cache-aside wrappers | v2.6–3.0 supported |
| RabbitMQ v5/v6 | ✅ Stable | Bytecode | Consumer logic | |
| RabbitMQ v7+ | ✅ Stable | Native ActivitySource | Consumer logic | `ADDITIONAL_SOURCES=RabbitMQ.Client` |
| gRPC (client) | ✅ Stable | Bytecode | Service logic | `SuppressDownstreamInstrumentation=true` |
| gRPC (server) | ✅ via ASP.NET Core | Bytecode | — | |
| SQL Server (ADO.NET) | ✅ Stable | Bytecode | Repo layer | |
| MassTransit v8+ | ❌ NOT in agent | Manual | Consumer logic | `ADDITIONAL_SOURCES=MassTransit` |
| Azure Service Bus | ❌ NOT in agent | Manual | Processor logic | `ADDITIONAL_SOURCES=Azure.Messaging.ServiceBus.*` |
| Kafka/Confluent | ⚠️ Alpha | Bytecode | Consumer logic | Pin version; consider manual package |
| MongoDB | ✅ Stable | Bytecode | Repo layer | |

## Sources

- [OTel .NET Auto-Instrumentation GitHub](https://github.com/open-telemetry/opentelemetry-dotnet-instrumentation) — official source, env vars, Plugin API docs
- [Auto-Instrumentation releases v1.14.1](https://github.com/open-telemetry/opentelemetry-dotnet-instrumentation/releases/tag/v1.14.1)
- [Plugin API docs](https://github.com/open-telemetry/opentelemetry-dotnet-instrumentation/blob/main/docs/plugins.md)
- [Hybrid approach guide](https://github.com/open-telemetry/opentelemetry-dotnet-instrumentation/blob/main/docs/additional-manual-instrumentation.md)
- [RabbitMQ.Client.OpenTelemetry NuGet](https://www.nuget.org/packages/RabbitMQ.Client.OpenTelemetry)
- [MassTransit OTel docs](https://masstransit.io/documentation/configuration/observability)
- [Azure SDK OTel integration](https://learn.microsoft.com/en-us/dotnet/azure/sdk/observability)
- [OpenTelemetry.Instrumentation.ConfluentKafka NuGet](https://www.nuget.org/packages/OpenTelemetry.Instrumentation.ConfluentKafka)
- [Confluent.Kafka.Extensions.OpenTelemetry NuGet](https://www.nuget.org/packages/Confluent.Kafka.Extensions.OpenTelemetry)
- [Npgsql OTel tracing docs](https://www.npgsql.org/doc/diagnostics/tracing.html)
