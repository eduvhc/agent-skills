# OpenTelemetry Reference

## How OTel Flows in Aspire

```
Service Process                     Aspire Dashboard / OTLP Collector
  ├─ ILogger    → OTel LogRecord  →  OTEL_EXPORTER_OTLP_ENDPOINT (set by DCP)
  ├─ Activity   → OTel Span       →  same endpoint
  └─ Meter      → OTel Metric     →  same endpoint
```

DCP injects three env vars into every managed project automatically:

| Variable | Example | Purpose |
|---|---|---|
| `OTEL_SERVICE_NAME` | `api` | Service identifier |
| `OTEL_RESOURCE_ATTRIBUTES` | `service.instance.id=abc123` | Resource metadata |
| `OTEL_EXPORTER_OTLP_ENDPOINT` | `http://localhost:18889` | Dashboard gRPC OTLP port |

## Custom Activity Source (Distributed Tracing)

```csharp
// 1. Define ActivitySource — one per logical component
public static class Telemetry
{
    public static readonly ActivitySource Source =
        new("MyApp.Orders", "1.0.0");
}

// 2. Register in DI and OTel pipeline
builder.Services.AddSingleton(Telemetry.Source);
builder.Services.AddOpenTelemetry()
    .WithTracing(tracing => tracing.AddSource("MyApp.Orders"));

// 3. Use in application code
public class OrderService(ILogger<OrderService> logger)
{
    public async Task<Order> CreateOrderAsync(CreateOrderRequest request, CancellationToken ct)
    {
        using var activity = Telemetry.Source.StartActivity("CreateOrder");
        activity?.SetTag("order.customer_id", request.CustomerId);
        activity?.SetTag("order.item_count", request.Items.Count);

        try
        {
            var order = await ProcessAsync(request, ct);
            activity?.SetStatus(ActivityStatusCode.Ok);
            return order;
        }
        catch (Exception ex)
        {
            activity?.RecordException(ex);
            activity?.SetStatus(ActivityStatusCode.Error, ex.Message);
            logger.LogError(ex, "Failed to create order for {CustomerId}", request.CustomerId);
            throw;
        }
    }
}
```

## Custom Metrics (IMeterFactory Pattern)

Prefer `IMeterFactory` over a static `Meter` — it participates in DI lifecycle.

```csharp
public sealed class OrderMetrics
{
    private readonly Counter<long> _created;
    private readonly Histogram<double> _duration;

    public OrderMetrics(IMeterFactory meterFactory)
    {
        var meter = meterFactory.Create("MyApp.Orders", "1.0.0");
        _created = meter.CreateCounter<long>(
            "orders.created",
            description: "Number of orders created");
        _duration = meter.CreateHistogram<double>(
            "orders.processing_duration",
            unit: "ms",
            description: "Time taken to process an order");
    }

    public void RecordCreated(string tier) =>
        _created.Add(1, new TagList { { "customer.tier", tier } });

    public void RecordDuration(double ms, string status) =>
        _duration.Record(ms, new TagList { { "order.status", status } });
}

// Register
builder.Services.AddSingleton<OrderMetrics>();

// Hook into OTel pipeline
.WithMetrics(metrics => metrics.AddMeter("MyApp.Orders"));
```

## Structured Logging Best Practices

```csharp
// Use source-generated log messages for performance (no boxing)
public static partial class Log
{
    [LoggerMessage(Level = LogLevel.Information,
        Message = "Applying {Count} pending migration(s): {Migrations}")]
    public static partial void ApplyingMigrations(
        ILogger logger, int count, IReadOnlyList<string> migrations);

    [LoggerMessage(Level = LogLevel.Error,
        Message = "Migration failed after {ElapsedMs}ms")]
    public static partial void MigrationFailed(
        ILogger logger, double elapsedMs, Exception ex);
}

// Usage
Log.ApplyingMigrations(logger, pending.Count, pending);
```

## Standalone Dashboard (Local Dev Without Full Aspire)

```bash
docker run --rm -it \
  -p 18888:18888 \
  -p 4317:18889 \
  --name aspire-dashboard \
  mcr.microsoft.com/dotnet/aspire-dashboard:latest
```

```json
// appsettings.Development.json
{
  "OTEL_EXPORTER_OTLP_ENDPOINT": "http://localhost:4317",
  "OTEL_SERVICE_NAME": "my-app"
}
```

## Production Configuration

```bash
# Point to your collector, Grafana Agent, or OTLP-compatible backend
OTEL_EXPORTER_OTLP_ENDPOINT=http://otel-collector:4317
OTEL_SERVICE_NAME=my-api
OTEL_RESOURCE_ATTRIBUTES=deployment.environment=production,service.version=1.2.3

# Sampling — default AlwaysOn is overwhelming at scale
OTEL_TRACES_SAMPLER=parentbased_traceidratio
OTEL_TRACES_SAMPLER_ARG=0.1   # 10%
```

**Critical:** If `OTEL_EXPORTER_OTLP_ENDPOINT` is not set, `UseOtlpExporter()` is not called and telemetry is silently dropped. Always verify this variable in your deployment manifests.

## Manual OTel Setup (Without ServiceDefaults)

```xml
<PackageReference Include="OpenTelemetry.Exporter.OpenTelemetryProtocol" />
<PackageReference Include="OpenTelemetry.Extensions.Hosting" />
<PackageReference Include="OpenTelemetry.Instrumentation.AspNetCore" />
<PackageReference Include="OpenTelemetry.Instrumentation.Http" />
<PackageReference Include="OpenTelemetry.Instrumentation.Runtime" />
<!-- Optional -->
<PackageReference Include="OpenTelemetry.Instrumentation.EntityFrameworkCore" />
<PackageReference Include="OpenTelemetry.Instrumentation.StackExchangeRedis" />
```

```csharp
builder.Logging.AddOpenTelemetry(logging =>
{
    logging.IncludeFormattedMessage = true;
    logging.IncludeScopes = true;
});

builder.Services.AddOpenTelemetry()
    .WithMetrics(metrics =>
    {
        metrics.AddAspNetCoreInstrumentation()
               .AddHttpClientInstrumentation()
               .AddRuntimeInstrumentation()
               .AddMeter("MyApp.Orders");
    })
    .WithTracing(tracing =>
    {
        tracing.AddSource(builder.Environment.ApplicationName)
               .AddSource("MyApp.Orders")
               .AddAspNetCoreInstrumentation()
               .AddHttpClientInstrumentation()
               .AddEntityFrameworkCoreInstrumentation();
    });

if (!string.IsNullOrWhiteSpace(builder.Configuration["OTEL_EXPORTER_OTLP_ENDPOINT"]))
    builder.Services.AddOpenTelemetry().UseOtlpExporter();
```

## Npgsql / EF Core OTel Integration

When using `Aspire.Npgsql.EntityFrameworkCore.PostgreSQL`, tracing and metrics are registered automatically when OTel is configured. No extra packages needed.

For manual setup:
```csharp
.WithTracing(tracing => tracing
    .AddNpgsql()   // from Npgsql.OpenTelemetry package
    .AddEntityFrameworkCoreInstrumentation(o =>
    {
        o.SetDbStatementForText = true;  // include SQL in spans (dev only)
    }));
```
