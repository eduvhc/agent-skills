# Exporters Reference

## OTLP — Primary Production Exporter

### gRPC (default, port 4317)

```csharp
.AddOtlpExporter(options =>
{
    options.Endpoint = new Uri("http://otel-collector:4317");
    options.Protocol = OtlpExportProtocol.Grpc;      // default
    options.Headers  = "x-api-key=mykey,x-tenant=acme";
    options.TimeoutMilliseconds = 10_000;
})
```

### HTTP/protobuf (port 4318)

```csharp
.AddOtlpExporter(options =>
{
    // HTTP/protobuf REQUIRES the signal path in the URL
    options.Endpoint = new Uri("http://otel-collector:4318/v1/traces");
    options.Protocol = OtlpExportProtocol.HttpProtobuf;
})
// Paths: /v1/traces  /v1/metrics  /v1/logs
```

**gRPC vs HTTP/protobuf:**
- gRPC: requires HTTP/2, more efficient at high throughput
- HTTP/protobuf: works through HTTP/1.1 proxies, easier in constrained environments

### Environment Variables (Preferred in Production)

```bash
OTEL_EXPORTER_OTLP_ENDPOINT=http://otel-collector:4317
OTEL_EXPORTER_OTLP_PROTOCOL=grpc                       # or http/protobuf
OTEL_EXPORTER_OTLP_HEADERS=x-api-key=secret,x-tenant=acme
OTEL_EXPORTER_OTLP_TIMEOUT=10000

# Signal-specific overrides (only works with AddOtlpExporter, not UseOtlpExporter)
OTEL_EXPORTER_OTLP_TRACES_ENDPOINT=http://jaeger:4317
OTEL_EXPORTER_OTLP_METRICS_ENDPOINT=http://prometheus:4317
OTEL_EXPORTER_OTLP_LOGS_ENDPOINT=http://loki:4317
```

### mTLS (.NET 8+)

```bash
OTEL_EXPORTER_OTLP_CERTIFICATE=/certs/ca.pem
OTEL_EXPORTER_OTLP_CLIENT_CERTIFICATE=/certs/client.pem
OTEL_EXPORTER_OTLP_CLIENT_KEY=/certs/client-key.pem
```

## UseOtlpExporter vs AddOtlpExporter

| | `UseOtlpExporter()` | `AddOtlpExporter()` |
|---|---|---|
| Call count | Once only (throws on second call) | Multiple calls OK |
| Signals covered | Traces + Metrics + Logs | Per-signal |
| Signal-specific endpoints | No | Yes |
| Can mix with other | No | Yes |
| Recommended when | Same endpoint for all signals | Different endpoints per signal |

## Console Exporter (Dev Only)

```csharp
.AddConsoleExporter()   // NEVER in production
```

Synchronously writes to stdout — blocks the export pipeline. Produces excessive output. Dev and debugging only.

## Prometheus (Pull/Scrape)

```csharp
// ASP.NET Core (beta)
builder.Services.AddOpenTelemetry()
    .WithMetrics(m => m
        .AddMeter("MyApp.Metrics")
        .AddPrometheusExporter());

app.UseOpenTelemetryPrometheusScrapingEndpoint();           // default: /metrics
app.UseOpenTelemetryPrometheusScrapingEndpoint("/prometheus"); // custom path
```

Or push via OTLP receiver:

```bash
OTEL_EXPORTER_OTLP_METRICS_ENDPOINT=http://prometheus:9090/api/v1/otlp/v1/metrics
```

## In-Memory Exporter (Testing)

```csharp
var exportedActivities = new List<Activity>();
var exportedMetrics    = new List<Metric>();

using var tracerProvider = Sdk.CreateTracerProviderBuilder()
    .AddSource("MyApp.Tests")
    .AddInMemoryExporter(exportedActivities)
    .Build();

using var meterProvider = Sdk.CreateMeterProviderBuilder()
    .AddMeter("MyApp.Tests")
    .AddInMemoryExporter(exportedMetrics)
    .Build();

// Run code under test
DoWork();

Assert.Single(exportedActivities);
Assert.Equal("PlaceOrder", exportedActivities[0].DisplayName);
Assert.Equal(ActivityStatusCode.Ok, exportedActivities[0].Status);
```

## Batch Processor Tuning

```csharp
// Per-signal batch tuning
builder.Logging.AddOpenTelemetry(o =>
    o.AddOtlpExporter((exporterOptions, processorOptions) =>
    {
        processorOptions.BatchExportProcessorOptions.ScheduledDelayMilliseconds  = 5_000;
        processorOptions.BatchExportProcessorOptions.MaxExportBatchSize          = 512;
        processorOptions.BatchExportProcessorOptions.MaxQueueSize                = 2_048;
        processorOptions.BatchExportProcessorOptions.ExporterTimeoutMilliseconds = 30_000;
    }));
```

Environment variable equivalents:

```bash
# Traces
OTEL_BSP_SCHEDULE_DELAY=5000
OTEL_BSP_MAX_QUEUE_SIZE=2048
OTEL_BSP_MAX_EXPORT_BATCH_SIZE=512
OTEL_BSP_EXPORT_TIMEOUT=30000

# Metrics
OTEL_METRIC_EXPORT_INTERVAL=30000
OTEL_METRIC_EXPORT_TIMEOUT=10000

# Logs
OTEL_BLRP_SCHEDULE_DELAY=5000
OTEL_BLRP_MAX_QUEUE_SIZE=2048
OTEL_BLRP_MAX_EXPORT_BATCH_SIZE=512
OTEL_BLRP_EXPORT_TIMEOUT=30000
```

## Metrics Temporality

```csharp
.WithMetrics(m => m
    .AddOtlpExporter((exporterOptions, readerOptions) =>
    {
        // Delta: most backends prefer this (Prometheus is cumulative)
        readerOptions.TemporalityPreference =
            MetricReaderTemporalityPreference.Delta;
        readerOptions.PeriodicExportingMetricReaderOptions.ExportIntervalMilliseconds = 30_000;
    }))
```

## Experimental Retry

```bash
OTEL_DOTNET_EXPERIMENTAL_OTLP_RETRY=in_memory    # retry in process memory
OTEL_DOTNET_EXPERIMENTAL_OTLP_RETRY=disk          # retry via disk (survives crash)
OTEL_DOTNET_EXPERIMENTAL_OTLP_DISK_RETRY_DIRECTORY_PATH=/var/otel/retry
```
