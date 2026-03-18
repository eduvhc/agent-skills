# Sampling Reference

## Built-in Samplers

| Sampler | Description |
|---|---|
| `AlwaysOnSampler` | Records every span — default, overwhelming in production |
| `AlwaysOffSampler` | Drops every span — useful to disable tracing in specific environments |
| `TraceIdRatioBasedSampler(ratio)` | Samples `ratio` fraction by trace ID hash (consistent per trace ID) |
| `ParentBasedSampler(rootSampler)` | Follows parent's sampling decision; uses `rootSampler` for root spans |

## Configuration

```csharp
// Code
.WithTracing(t => t
    .SetSampler(new ParentBasedSampler(
        new TraceIdRatioBasedSampler(0.1)))  // 10% of new traces
    .AddSource("MyApp"))
```

```bash
# Environment variables
OTEL_TRACES_SAMPLER=parentbased_traceidratio
OTEL_TRACES_SAMPLER_ARG=0.1

# All available values:
# always_on
# always_off
# traceidratio
# parentbased_always_on       (default)
# parentbased_always_off
# parentbased_traceidratio
```

## Why ParentBased Matters

Without `ParentBasedSampler`, each service independently flips its own sampling coin:

```
Service A (1% sampler): samples trace → sends to Service B
Service B (1% sampler): independently decides → drops this trace

Result: broken partial traces that start in A but have no spans from B
```

`ParentBasedSampler` ensures all downstream spans follow the head decision made at the entry point. Always wrap your root sampler in `ParentBasedSampler`.

## Production Guidance

```bash
# Low traffic (< 10 RPS): higher rate is fine
OTEL_TRACES_SAMPLER=parentbased_traceidratio
OTEL_TRACES_SAMPLER_ARG=0.25    # 25%

# Medium traffic (10-100 RPS)
OTEL_TRACES_SAMPLER_ARG=0.05    # 5%

# High traffic (> 100 RPS): keep backend costs manageable
OTEL_TRACES_SAMPLER_ARG=0.01    # 1%

# Critical financial/security paths: force 100% regardless of global rate
# Use a custom sampler that checks operation name
```

## IsAllDataRequested — Performance Gate

When sampling rate is < 100%, most spans are dropped before they're exported. `IsAllDataRequested` tells you if this span is being sampled:

```csharp
using var activity = _source.StartActivity("ExpensiveOperation");

// Only build expensive tag values when someone will actually see them
if (activity?.IsAllDataRequested == true)
{
    activity.SetTag("db.query.text", BuildExpensiveQueryText());
    activity.SetTag("request.body",  SerializeRequestBody(req));
}
```

## Tail-Based Sampling (Always Capture Errors)

Head sampling misses errors that occur in the 99% dropped fraction. To ensure errors always get recorded:

```csharp
public class ErrorForceSamplingProcessor : BaseProcessor<Activity>
{
    public override void OnEnd(Activity activity)
    {
        if (activity.Status == ActivityStatusCode.Error)
        {
            // Force this span (and by extension the trace) to be recorded
            activity.ActivityTraceFlags |= ActivityTraceFlags.Recorded;
        }
        base.OnEnd(activity);
    }
}

.WithTracing(t => t
    .SetSampler(new ParentBasedSampler(new TraceIdRatioBasedSampler(0.05)))
    .AddProcessor(new ErrorForceSamplingProcessor())
    .AddOtlpExporter())
```

For more sophisticated tail sampling (by latency p99, error rate, specific operations), deploy an **OTel Collector** with the `tailsampling` processor — the .NET SDK itself only supports head sampling.

> `TraceIdRatioBasedSampler` is deprecated in the spec in favor of a new `ProbabilitySampler`, but remains supported in the SDK until at least January 2027.
