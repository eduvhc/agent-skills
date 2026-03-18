# Context Propagation Reference

## Default Propagators (.NET 10)

The default composite propagator includes:
1. **W3C TraceContext** — `traceparent` and `tracestate` headers
2. **W3C Baggage** — `baggage` header

`AddAspNetCoreInstrumentation()` automatically **extracts** incoming headers.
`AddHttpClientInstrumentation()` automatically **injects** outgoing headers.

**No extra code needed for standard HTTP service-to-service propagation.**

## Baggage — Cross-Service Key-Value Propagation

```csharp
// Set baggage in Service A
Baggage.Current = Baggage.Current
    .SetBaggage("tenant.id", tenantId)
    .SetBaggage("user.tier", "premium");

// Read in Service B (propagated automatically via HTTP headers)
var tenantId = Baggage.Current.GetBaggage("tenant.id");
```

> Keep baggage small — every key-value pair is serialized into every outgoing request header. Never put large objects in baggage.

## B3 Propagation (Zipkin Compatibility)

```csharp
// dotnet add package OpenTelemetry.Extensions.Propagators
using OpenTelemetry.Context.Propagation;
using OpenTelemetry.Extensions.Propagators;

// Call BEFORE building the provider
Sdk.SetDefaultTextMapPropagator(new CompositeTextMapPropagator(new TextMapPropagator[]
{
    new TraceContextPropagator(),      // W3C (keep for modern services)
    new BaggagePropagator(),
    new B3Propagator(),                // single-header B3
    // new B3Propagator(singleHeader: false)  // multi-header B3
}));
```

## Manual Inject/Extract (Message Queues, Custom Protocols)

For non-HTTP transports where instrumentation libraries don't handle propagation automatically:

```csharp
// === PRODUCER (inject context into outgoing message) ===
var carrier = new Dictionary<string, string>();

Propagators.DefaultTextMapPropagator.Inject(
    new PropagationContext(Activity.Current?.Context ?? default, Baggage.Current),
    carrier,
    (dict, key, value) => dict[key] = value);

// Add carrier entries to message headers / envelope
message.Headers["traceparent"] = carrier["traceparent"];
message.Headers["baggage"]     = carrier.GetValueOrDefault("baggage");

// === CONSUMER (extract context from incoming message) ===
var extractedCtx = Propagators.DefaultTextMapPropagator.Extract(
    default,
    message.Headers,
    (headers, key) =>
        headers.TryGetValue(key, out var v) ? new[] { v } : Array.Empty<string>());

using var activity = _source.StartActivity(
    "ProcessMessage",
    ActivityKind.Consumer,
    extractedCtx.ActivityContext);

Baggage.Current = extractedCtx.Baggage;
```

## RabbitMQ Manual Propagation Example

```csharp
// Publisher
var props = channel.CreateBasicProperties();
props.Headers = new Dictionary<string, object>();

Propagators.DefaultTextMapPropagator.Inject(
    new PropagationContext(Activity.Current?.Context ?? default, Baggage.Current),
    props.Headers,
    (headers, key, value) => headers[key] = value);

channel.BasicPublish("exchange", "routing.key", props, body);

// Consumer
var headers    = ea.BasicProperties.Headers;
var extractCtx = Propagators.DefaultTextMapPropagator.Extract(
    default,
    headers,
    (h, key) => h.TryGetValue(key, out var v)
        ? new[] { Encoding.UTF8.GetString((byte[])v) }
        : Array.Empty<string>());

using var activity = _source.StartActivity(
    "rabbitmq.consume",
    ActivityKind.Consumer,
    extractCtx.ActivityContext);
```
