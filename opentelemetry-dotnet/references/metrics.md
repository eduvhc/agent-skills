# Metrics Reference

## Instrument Types

| OTel Spec | .NET Type | Use Case |
|---|---|---|
| Counter | `Counter<T>` | Monotonically increasing count (requests, errors, bytes sent) |
| Histogram | `Histogram<T>` | Value distribution (latency, payload size) |
| Gauge | `Gauge<T>` | Instantaneous non-additive value (temperature, voltage) |
| UpDownCounter | `UpDownCounter<T>` | Additive value that goes up and down (active connections, queue depth) |
| Observable Counter | `ObservableCounter<T>` | Polled monotonic count (CPU time, total bytes) |
| Observable Gauge | `ObservableGauge<T>` | Polled instantaneous value (memory usage, thread count) |
| Observable UpDownCounter | `ObservableUpDownCounter<T>` | Polled additive gauge |

`T` is typically `long` (for counts) or `double` (for sizes/durations). Prefer `long` for integers.

## IMeterFactory Pattern (Recommended for DI Apps)

```csharp
// IMeterFactory was introduced in .NET 8
// Preferred over static Meter — enables proper DI lifecycle and test isolation
public sealed class OrderMetrics
{
    private readonly Counter<long>     _ordersPlaced;
    private readonly Histogram<double> _orderDuration;
    private readonly UpDownCounter<long> _pendingOrders;

    public OrderMetrics(IMeterFactory meterFactory)
    {
        var meter = meterFactory.Create("MyApp.Orders", "1.0.0");

        _ordersPlaced = meter.CreateCounter<long>(
            "orders.placed",
            unit: "{order}",
            description: "Number of orders placed");

        _orderDuration = meter.CreateHistogram<double>(
            "orders.processing_duration",
            unit: "ms",
            description: "Order processing time");

        _pendingOrders = meter.CreateUpDownCounter<long>(
            "orders.pending",
            unit: "{order}",
            description: "Orders currently being processed");
    }

    public void RecordOrderPlaced(string tier) =>
        _ordersPlaced.Add(1, new("customer.tier", tier));

    public void RecordDuration(double ms, bool success) =>
        _orderDuration.Record(ms, new("order.status", success ? "success" : "error"));

    public void TrackPendingOrders(int delta) =>
        _pendingOrders.Add(delta);
}

// Registration
builder.Services.AddSingleton<OrderMetrics>();
// IMeterFactory is automatically available when using AddOpenTelemetry()

// Register the meter with the pipeline
.WithMetrics(m => m.AddMeter("MyApp.Orders"))
```

## Static Meter Pattern (Library Code / Simple Apps)

```csharp
using System.Diagnostics.Metrics;

// NEVER create per-request — static readonly
private static readonly Meter _meter = new("MyApp.Orders", "1.0.0");

public static readonly Counter<long>    OrdersPlaced  = _meter.CreateCounter<long>("orders.placed", "{order}");
public static readonly Histogram<double> OrderDuration = _meter.CreateHistogram<double>("orders.duration", "ms");
```

## TagList — Performance-Optimized Tags

```csharp
// 1-3 tags: pass KeyValuePair directly (no heap allocation)
_counter.Add(1,
    new("http.method",      "GET"),
    new("http.status_code", 200));

// 4-8 tags: use TagList (stack-allocated struct, avoids heap)
var tags = new TagList
{
    { "http.method",        "POST" },
    { "http.status_code",   201 },
    { "service.region",     "eu-west-1" },
    { "order.type",         "express" }
};
_counter.Add(1, tags);
```

## Observable (Polled) Instruments

```csharp
// Polled by the SDK on each export cycle — ideal for external state
var queueDepth = meter.CreateObservableGauge<long>(
    "orders.queue.depth",
    () => _orderQueue.Count,
    unit: "{order}");

// Multiple measurements from one callback
var memoryGauge = meter.CreateObservableGauge<long>(
    "runtime.memory",
    () => new[]
    {
        new Measurement<long>(GC.GetTotalMemory(false),         new("gc.heap", "current")),
        new Measurement<long>(GC.GetGCMemoryInfo().HeapSizeBytes, new("gc.heap", "committed")),
    });
```

## Exemplars — Linking Metrics to Traces

Exemplars attach the current trace context (`traceId`, `spanId`) to a metric measurement, enabling backends like Grafana to "jump from this histogram bucket to the trace that caused it":

```csharp
builder.Services.AddOpenTelemetry()
    .WithMetrics(m => m
        .AddMeter("MyApp.Metrics")
        .SetExemplarFilter(ExemplarFilterType.TraceBased)   // only when inside a span
        .AddOtlpExporter());
```

No code change at the measurement site — SDK captures `Activity.Current` automatically.

Default reservoirs:
- **Histograms**: `AlignedHistogramBucketExemplarReservoir` — one exemplar per bucket (last-wins)
- **Other instruments**: `SimpleFixedSizeExemplarReservoir`

## Cardinality Limits

The SDK caps unique attribute combinations at **2,000 per metric** by default. Measurements exceeding this are merged into an overflow bucket tagged `otel.metric.overflow=true`.

**Never use unbounded values as tags:**

```csharp
// BAD — millions of unique series
counter.Add(1, new("user.id",      userId.ToString()));
counter.Add(1, new("request.url",  rawUrl));

// GOOD — bounded, low-cardinality values
counter.Add(1, new("user.tier",    "premium"));
counter.Add(1, new("http.route",   "/api/orders/{id}"));
```

## Histogram Explicit Boundaries

Custom bucket boundaries for latency histograms (instead of default [0, 5, 10, 25, 50, 75, 100, 250, 500, 750, 1000, 2500, 5000, 7500, 10000]):

```csharp
.WithMetrics(m => m
    .AddView(
        instrumentName: "orders.processing_duration",
        new ExplicitBucketHistogramConfiguration
        {
            Boundaries = [1, 5, 10, 50, 100, 500, 1000, 5000]
        }))
```
