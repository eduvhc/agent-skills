# Common Pitfalls

## P1: ActivitySource Name Mismatch → Silent Drop

```csharp
// BUG: typo in registration
var source = new ActivitySource("MyApp.Orders");

.WithTracing(t => t.AddSource("MyApp.Order"));   // MISSING 's' — no match!
// All activities silently dropped, no error thrown

// FIX: exact match, case-sensitive
.WithTracing(t => t.AddSource("MyApp.Orders"))
// Or wildcard prefix:
.WithTracing(t => t.AddSource("MyApp.*"))
```

## P2: OTEL_EXPORTER_OTLP_ENDPOINT Not Set

```bash
# Symptom: no telemetry in backend, no error logged
# Cause: OTLP exporter defaults to http://localhost:4317 — no collector there

# FIX: always set in production
OTEL_EXPORTER_OTLP_ENDPOINT=http://otel-collector.monitoring.svc.cluster.local:4317
```

Or in code:

```csharp
.AddOtlpExporter(o => o.Endpoint = new Uri("http://otel-collector:4317"))
```

## P3: Meter Not Registered

```csharp
// BUG: meter created but not registered on MeterProvider
var meter = new Meter("MyApp.Orders");

.WithMetrics(m => m.AddMeter("MyApp.Payments"));  // "MyApp.Orders" never registered → silent drop

// FIX:
.WithMetrics(m => m
    .AddMeter("MyApp.Orders")
    .AddMeter("MyApp.Payments"))
```

## P4: Not Disposing Provider in Non-DI Apps

```csharp
// BUG: exits without disposing → last ~5s of spans never exported
static void Main()
{
    var provider = Sdk.CreateTracerProviderBuilder().Build();
    DoWork();
    // App exits — BatchProcessor never flushed
}

// FIX: using statement or explicit ForceFlush
using var provider = Sdk.CreateTracerProviderBuilder().Build();
// or:
provider.ForceFlush(timeoutMilliseconds: 5000);
```

In DI apps, `AddOpenTelemetry()` with Generic Host handles this automatically.

## P5: High-Cardinality Tags → Memory Explosion / Cardinality Limit

```csharp
// BUG: unbounded tag values
activity?.SetTag("user.id",     guid.ToString());         // millions of unique values
counter.Add(1, new("request.url", rawUrl));               // unbounded

// FIX: use low-cardinality values only
activity?.SetTag("user.tier",   "premium");               // bounded set
counter.Add(1, new("http.route", "/api/orders/{id}"));    // route pattern
```

SDK caps metric cardinality at 2,000 unique attribute combinations per metric — excess is merged into `otel.metric.overflow=true`.

## P6: Creating ActivitySource or Meter Per-Request

```csharp
// BUG: new object per call — extremely expensive
public void HandleRequest()
{
    var source = new ActivitySource("MyApp");   // DON'T
    using var activity = source.StartActivity("Handle");
}

// FIX: static readonly or DI singleton
private static readonly ActivitySource _source = new("MyApp");
```

## P7: Console Exporter in Production

```csharp
.AddConsoleExporter()   // NEVER in production
```

Synchronously writes to stdout — serializes the export pipeline. Use OTLP.

## P8: Wrong Endpoint Path for HTTP/protobuf

```csharp
// BUG: HTTP/protobuf requires signal path
options.Protocol = OtlpExportProtocol.HttpProtobuf;
options.Endpoint = new Uri("http://collector:4318");       // missing path → 404

// FIX: include signal path for HTTP/protobuf
options.Endpoint = new Uri("http://collector:4318/v1/traces");
// Other paths: /v1/metrics  /v1/logs

// gRPC does NOT need the path:
options.Protocol = OtlpExportProtocol.Grpc;
options.Endpoint = new Uri("http://collector:4317");       // correct
```

## P9: UseOtlpExporter Called Multiple Times

```csharp
// BUG: throws NotSupportedException
builder.Services.AddOpenTelemetry()
    .UseOtlpExporter()    // fine
    .UseOtlpExporter();   // THROWS

// Also BUG: cannot mix with AddOtlpExporter
builder.Services.AddOpenTelemetry()
    .WithTracing(t => t.AddOtlpExporter())   // conflicts
    .UseOtlpExporter();                       // THROWS
```

## P10: Missing Null-Check on Activity

```csharp
// BUG: NullReferenceException when no TracerProvider is registered
var activity = _source.StartActivity("work");
activity.SetTag("key", "value");   // NullReferenceException!

// FIX: always null-conditional
using var activity = _source.StartActivity("work");
activity?.SetTag("key", "value");
```

## P11: Missing RecordException / AddException

```csharp
// BUG: error span has no exception detail
catch (Exception ex)
{
    activity?.SetStatus(ActivityStatusCode.Error, ex.Message);
    throw;
}

// FIX: call AddException first
catch (Exception ex)
{
    activity?.AddException(ex);                               // records type, message, stack trace
    activity?.SetStatus(ActivityStatusCode.Error, ex.Message);
    throw;
}
```

## P12: Baggage Size Overhead

```csharp
// BUG: KB-sized value in every outgoing request header
Baggage.Current = Baggage.Current
    .SetBaggage("user.profile", JsonSerializer.Serialize(largeObject));

// FIX: baggage is for small, essential cross-cutting values only
Baggage.Current = Baggage.Current
    .SetBaggage("tenant.id", tenantId)    // short string
    .SetBaggage("user.tier", tier);       // bounded value
```
