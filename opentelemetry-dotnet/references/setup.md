# Setup Patterns

## ASP.NET Core — Full Setup (Recommended)

```csharp
using OpenTelemetry.Logs;
using OpenTelemetry.Metrics;
using OpenTelemetry.Resources;
using OpenTelemetry.Trace;

var builder = WebApplication.CreateBuilder(args);

Action<ResourceBuilder> configureResource = r => r
    .AddService(
        serviceName:    "my-api",
        serviceVersion: "1.2.0",
        autoGenerateServiceInstanceId: true)
    .AddAttributes(new Dictionary<string, object>
    {
        ["deployment.environment"] = builder.Environment.EnvironmentName,
        ["host.name"]              = Environment.MachineName,
    })
    .AddEnvironmentVariableDetector()   // reads OTEL_RESOURCE_ATTRIBUTES
    .AddTelemetrySdk();                 // adds telemetry.sdk.* attributes

builder.Services.AddOpenTelemetry()
    .ConfigureResource(configureResource)
    .WithTracing(tracing => tracing
        .AddSource("MyApp.ActivitySource")
        .AddAspNetCoreInstrumentation(o =>
        {
            o.Filter         = ctx => !ctx.Request.Path.StartsWithSegments("/health");
            o.RecordException = true;
        })
        .AddHttpClientInstrumentation()
        .AddNpgsql())                   // Npgsql-native; or AddEntityFrameworkCoreInstrumentation()
    .WithMetrics(metrics => metrics
        .AddMeter("MyApp.Metrics")
        .AddAspNetCoreInstrumentation()
        .AddHttpClientInstrumentation()
        .AddRuntimeInstrumentation())
    .UseOtlpExporter();                 // single call covers traces + metrics + logs

builder.Logging.AddOpenTelemetry(logging =>
{
    logging.IncludeFormattedMessage = true;
    logging.IncludeScopes           = true;
    logging.ParseStateValues        = true;
    // logging is exported by UseOtlpExporter() automatically
});

var app = builder.Build();
app.Run();
```

## Worker / Background Service

```csharp
var builder = Host.CreateApplicationBuilder(args);

builder.Services.AddOpenTelemetry()
    .ConfigureResource(r => r.AddService("my-worker", serviceVersion: "1.0.0"))
    .WithTracing(tracing => tracing
        .AddSource(MyWorker.ActivitySourceName)
        .AddHttpClientInstrumentation())
    .WithMetrics(metrics => metrics
        .AddMeter(MyWorker.MeterName)
        .AddRuntimeInstrumentation())
    .UseOtlpExporter();

builder.Logging.AddOpenTelemetry(logging =>
{
    logging.IncludeFormattedMessage = true;
    logging.IncludeScopes           = true;
});

builder.Services.AddHostedService<MyWorker>();

await builder.Build().RunAsync();
```

## Unified Exporter — UseOtlpExporter vs AddOtlpExporter

**Prefer `UseOtlpExporter()`** when all signals go to the same endpoint:

```csharp
builder.Services.AddOpenTelemetry()
    .WithTracing(t => t.AddSource("MyApp"))
    .WithMetrics(m => m.AddMeter("MyApp"))
    .UseOtlpExporter();   // one call, all signals

// CRITICAL: UseOtlpExporter() can be called ONCE only.
// Calling it twice throws NotSupportedException.
// Cannot be mixed with .AddOtlpExporter() calls.
```

Use **per-signal `AddOtlpExporter()`** only when signals go to different endpoints:

```csharp
builder.Services.AddOpenTelemetry()
    .WithTracing(t => t
        .AddSource("MyApp")
        .AddOtlpExporter(o => o.Endpoint = new Uri("http://jaeger:4317")))
    .WithMetrics(m => m
        .AddMeter("MyApp")
        .AddOtlpExporter(o => o.Endpoint = new Uri("http://prometheus:4317")));
```

## Aspire / ServiceDefaults Pattern

```csharp
// In each service's Program.cs
var builder = WebApplication.CreateBuilder(args);
builder.AddServiceDefaults();   // all OTel wiring from ServiceDefaults project

// Add your ActivitySource on top:
builder.Services.AddOpenTelemetry()
    .WithTracing(t => t.AddSource("MyApp.Orders"));
```

## Minimal — Console App (No DI)

```csharp
using OpenTelemetry;
using OpenTelemetry.Resources;
using OpenTelemetry.Trace;

// MUST be disposed — Dispose() flushes the batch processor
using var tracerProvider = Sdk.CreateTracerProviderBuilder()
    .SetResourceBuilder(ResourceBuilder.CreateDefault()
        .AddService("my-console-app", serviceVersion: "1.0.0"))
    .AddSource("MyApp")
    .AddOtlpExporter()
    .Build();

// Similarly for metrics
using var meterProvider = Sdk.CreateMeterProviderBuilder()
    .SetResourceBuilder(ResourceBuilder.CreateDefault()
        .AddService("my-console-app"))
    .AddMeter("MyApp")
    .AddOtlpExporter()
    .Build();

// If explicit flush needed before exit:
// tracerProvider.ForceFlush(timeoutMilliseconds: 5000);
```

## Adding Custom Source to Existing Aspire/ServiceDefaults Setup

```csharp
// ServiceDefaults already configures the OTel pipeline.
// Chain AddOpenTelemetry() to add your sources — multiple calls are safe,
// they return the same builder and register to the same provider.
builder.Services.AddOpenTelemetry()
    .WithTracing(t => t
        .AddSource(MigrationWorker.ActivitySourceName));   // safe to call again
```
