# Production Best Practices

## Always Set These Resource Attributes

```csharp
.ConfigureResource(r => r
    .AddService(
        serviceName:    "order-api",
        serviceVersion: "1.2.3",
        serviceNamespace: "com.acme",
        autoGenerateServiceInstanceId: true)  // stable service.instance.id per process
    .AddAttributes(new Dictionary<string, object>
    {
        ["deployment.environment"] = environment,      // "production", "staging"
        ["host.name"]              = Environment.MachineName,
        ["cloud.region"]           = "eu-west-1",
        ["k8s.namespace.name"]     = k8sNamespace,
        ["k8s.pod.name"]           = k8sPodName,
    })
    .AddEnvironmentVariableDetector()   // reads OTEL_RESOURCE_ATTRIBUTES at runtime
    .AddTelemetrySdk())                 // telemetry.sdk.name, .language, .version
```

Or fully via environment (no code change — preferred for Kubernetes):

```bash
OTEL_SERVICE_NAME=order-api
OTEL_RESOURCE_ATTRIBUTES=service.version=1.2.3,deployment.environment=production,host.name=web-01
```

## Environment Variables Cheatsheet

```bash
# Service identity
OTEL_SERVICE_NAME=my-service
OTEL_RESOURCE_ATTRIBUTES=service.version=1.0,deployment.environment=production

# OTLP exporter
OTEL_EXPORTER_OTLP_ENDPOINT=http://otel-collector:4317
OTEL_EXPORTER_OTLP_PROTOCOL=grpc
OTEL_EXPORTER_OTLP_HEADERS=x-api-key=secret
OTEL_EXPORTER_OTLP_TIMEOUT=10000

# Sampling
OTEL_TRACES_SAMPLER=parentbased_traceidratio
OTEL_TRACES_SAMPLER_ARG=0.05      # 5%

# Traces batch processor
OTEL_BSP_SCHEDULE_DELAY=5000
OTEL_BSP_MAX_QUEUE_SIZE=2048
OTEL_BSP_MAX_EXPORT_BATCH_SIZE=512
OTEL_BSP_EXPORT_TIMEOUT=30000

# Metrics export
OTEL_METRIC_EXPORT_INTERVAL=30000
OTEL_METRIC_EXPORT_TIMEOUT=10000

# Logs batch processor
OTEL_BLRP_SCHEDULE_DELAY=5000
OTEL_BLRP_MAX_QUEUE_SIZE=2048
OTEL_BLRP_MAX_EXPORT_BATCH_SIZE=512
OTEL_BLRP_EXPORT_TIMEOUT=30000

# Attribute limits
OTEL_ATTRIBUTE_COUNT_LIMIT=128
OTEL_SPAN_ATTRIBUTE_COUNT_LIMIT=128
OTEL_SPAN_EVENT_COUNT_LIMIT=128
OTEL_SPAN_LINK_COUNT_LIMIT=128

# Experimental features
OTEL_DOTNET_EXPERIMENTAL_OTLP_RETRY=in_memory
OTEL_DOTNET_EXPERIMENTAL_EFCORE_ENABLE_TRACE_DB_QUERY_PARAMETERS=true
OTEL_SEMCONV_STABILITY_OPT_IN=database
```

## SDK Self-Diagnostics

When telemetry is missing and you can't tell why, enable the SDK's own log file:

```json
// OTEL_DIAGNOSTICS.json — place in the app working directory
{
  "LogDirectory": ".",
  "FileSize": 32768,
  "LogLevel": "Warning"
}
```

## Provider Lifecycle (Non-DI Apps)

```csharp
// CRITICAL: Dispose() flushes the BatchProcessor
// Without disposal, the last ~5 seconds of spans/metrics are silently dropped
using var tracerProvider = Sdk.CreateTracerProviderBuilder()
    .AddSource("MyApp")
    .AddOtlpExporter()
    .Build();

// Explicit flush for graceful shutdown:
tracerProvider.ForceFlush(timeoutMilliseconds: 5000);
```

In DI apps (`AddOpenTelemetry()`), the Generic Host disposes providers automatically on shutdown.

## Multiple AddOpenTelemetry() Calls — Safe

```csharp
// Safe: multiple calls return the same builder, same provider
builder.Services.AddOpenTelemetry().WithTracing(/* ... */);
builder.Services.AddOpenTelemetry().WithMetrics(/* ... */);

// Common in Aspire: ServiceDefaults registers OTel, then each service adds its own sources
builder.AddServiceDefaults();
builder.Services.AddOpenTelemetry()
    .WithTracing(t => t.AddSource("MyApp.Custom"));
```

## .NET 10 Notable Changes

1. **W3C TraceContext is now the sole default** — remove any explicit `TraceContextPropagator` setup; it's redundant
2. **net10.0 TFM target** — all packages include a dedicated .NET 10 build
3. **New ASP.NET Core metrics** — authentication, Blazor, memory pool metrics available in `AddAspNetCoreInstrumentation()`
4. **Schema URL support** — metrics/traces carry semantic convention schema URLs for version tracking
5. **Npgsql.OpenTelemetry 10.0.1** — major version release aligned with .NET 10
6. **.NET Framework binding redirects** — if using .NET Framework, expand binding redirect max version from `9.9.9.9` to `10.0.0.0`
