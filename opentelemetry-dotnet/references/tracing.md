# Tracing Reference

## ActivitySource — Create Once, Use Everywhere

```csharp
// Static readonly — NEVER create per-request (very expensive)
// Name: dot-separated UpperCamelCase — "Company.Product.Component"
// Must EXACTLY match what you pass to .AddSource() — case-sensitive
private static readonly ActivitySource Source =
    new("MyApp.OrderService", "1.0.0");
```

For DI-friendly code, wrap in a singleton:

```csharp
public sealed class OrderInstrumentation : IDisposable
{
    public static readonly ActivitySource Source =
        new("MyApp.OrderService", "1.0.0");

    public void Dispose() => Source.Dispose();
}

builder.Services.AddSingleton<OrderInstrumentation>();
```

Register it on the TracerProvider:

```csharp
.WithTracing(t => t
    .AddSource("MyApp.OrderService")   // exact match
    .AddSource("MyApp.*"))             // prefix wildcard also supported
```

## ActivityKind

| Kind | When to use |
|---|---|
| `Internal` | Default. Internal operation, no network call |
| `Server` | Incoming RPC/HTTP request handler |
| `Client` | Outgoing RPC/HTTP/DB call — use for any external resource |
| `Producer` | Message sent to a queue/broker |
| `Consumer` | Message received from a queue/broker |

## Full Span Pattern

```csharp
public async Task<Order> PlaceOrderAsync(PlaceOrderRequest req, CancellationToken ct)
{
    // StartActivity() returns null when no TracerProvider is registered — always null-check
    using var activity = Source.StartActivity("PlaceOrder", ActivityKind.Internal);

    // IsAllDataRequested: only true when someone is sampling this span
    // Use it to gate expensive tag-building code
    if (activity is { IsAllDataRequested: true })
    {
        activity.SetTag("order.customer_id", req.CustomerId);
        activity.SetTag("order.item_count",  req.Items.Count);
    }

    try
    {
        var order = await _repo.SaveAsync(req, ct);

        activity?.SetTag("order.id", order.Id);
        activity?.SetStatus(ActivityStatusCode.Ok);
        return order;
    }
    catch (Exception ex)
    {
        activity?.AddException(ex);                            // records stack trace as span event
        activity?.SetStatus(ActivityStatusCode.Error, ex.Message);
        throw;
    }
}
```

## Span Status

```csharp
activity?.SetStatus(ActivityStatusCode.Ok);
activity?.SetStatus(ActivityStatusCode.Error, "Descriptive error message");
// Unset is the default — do not set Ok unless you confirm success
```

`ActivityStatusCode` values: `Unset` (default), `Ok`, `Error`.

## Span Events

Timestamped annotations within a span. Use for discrete milestones. For high-volume logging, use `ILogger` instead (correlates to current trace automatically).

```csharp
activity?.AddEvent(new ActivityEvent("validation.started"));

var tags = new ActivityTagsCollection
{
    { "validation.rule_count", 5 },
    { "validation.mode", "strict" }
};
activity?.AddEvent(new ActivityEvent("validation.completed", DateTimeOffset.UtcNow, tags));
```

## Database Span Pattern (Semantic Conventions)

```csharp
using var activity = Source.StartActivity("db.migrate", ActivityKind.Client);

// OTel semantic conventions for DB spans
activity?.SetTag("db.system.name",    "postgresql");   // required
activity?.SetTag("db.namespace",      "orders");        // conditional (database name)
activity?.SetTag("db.operation.name", "migrate");       // conditional
activity?.SetTag("server.address",    "db.internal");  // recommended
activity?.SetTag("server.port",       5432);           // recommended

// Only include query text in dev — may expose sensitive data
// activity?.SetTag("db.query.text", sql);
```

## Span Links (Batch Processing)

Links associate a span with multiple parent contexts — used when a single operation processes items from different traces.

```csharp
void ProcessBatch(ActivityContext[] sourceContexts)
{
    using var activity = Source.StartActivity(
        name:          "ProcessBatch",
        kind:          ActivityKind.Consumer,
        parentContext: default,   // no single parent
        links:         sourceContexts.Select(ctx => new ActivityLink(ctx)));
}

// Capture context from an incoming message header:
var ctx = ActivityContext.Parse(traceParentHeader, traceStateHeader);
```

Links must be provided at `StartActivity()` — immutable afterward. Spec recommends max 128 links.

## Parent-Child (Explicit Parent)

```csharp
// Implicit: Activity.Current is automatically the parent
using var child = Source.StartActivity("ChildWork");

// Explicit parent from incoming header (e.g., message queue consumer):
var parentCtx = ActivityContext.Parse(traceparentHeader, tracestateHeader);
using var span = Source.StartActivity(
    name:          "ProcessMessage",
    kind:          ActivityKind.Consumer,
    parentContext: parentCtx);
```

## Accessing the Current Span

```csharp
// Enrich from middleware or a handler that doesn't own the source
var current = Activity.Current;
current?.SetTag("tenant.id", tenantId);
current?.SetTag("request.correlation_id", correlationId);
```

## AddException vs RecordException

In .NET's `System.Diagnostics`, the method is `AddException` (not `RecordException` as in other OTel SDKs):

```csharp
activity?.AddException(ex);   // .NET name — same as OTel "RecordException"
```

This adds a span event with:
- `exception.type` — fully qualified exception type name
- `exception.message` — exception message
- `exception.stacktrace` — full stack trace
