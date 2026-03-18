# Logging Reference

## ILogger Bridge Configuration

The OTel logging bridge captures `ILogger` output and routes it through the OTel pipeline. **No code changes required in existing logging calls.**

```csharp
builder.Logging.AddOpenTelemetry(options =>
{
    options.IncludeFormattedMessage = true;   // includes the fully rendered message string
    options.IncludeScopes           = true;   // captures ILogger.BeginScope() values as attributes
    options.ParseStateValues        = true;   // parses structured log state into individual attributes
    // export is handled by UseOtlpExporter() or AddOtlpExporter()
});
```

Or with explicit OTLP:

```csharp
builder.Logging.AddOpenTelemetry(options =>
{
    options.IncludeFormattedMessage = true;
    options.IncludeScopes           = true;
    options.ParseStateValues        = true;
    options.AddOtlpExporter();   // if not using UseOtlpExporter()
});
```

## Log Level Filtering

The OTel bridge respects standard `ILogger` category filtering:

```json
{
  "Logging": {
    "OpenTelemetry": {
      "Default":              "Information",
      "Microsoft.AspNetCore": "Warning",
      "Microsoft.EntityFrameworkCore": "Warning",
      "System":               "Warning",
      "Aspire.Hosting.Dcp":   "Warning"
    }
  }
}
```

## Trace-Log Correlation

Log records emitted while a span is active automatically include `TraceId` and `SpanId` — no extra configuration. Backends like Grafana Loki, Jaeger, and Seq use these to correlate logs with traces.

```csharp
// This log record automatically carries traceId + spanId
// when called inside an active Activity span
logger.LogInformation("Processing order {OrderId}", orderId);
```

## Structured Logging Best Practices

```csharp
// Named placeholders become structured attributes
logger.LogInformation("Order {OrderId} placed for customer {CustomerId}",
    order.Id, customer.Id);

// Use scopes for cross-cutting context (captured when IncludeScopes = true)
using var scope = logger.BeginScope(new Dictionary<string, object>
{
    ["RequestId"] = httpContext.TraceIdentifier,
    ["UserId"]    = userId,
    ["TenantId"]  = tenantId,
});
logger.LogWarning("Payment validation failed for order {OrderId}", orderId);
```

## Source-Generated Log Messages (Performance)

For hot paths, source-generated log methods avoid boxing and string allocation:

```csharp
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

    [LoggerMessage(Level = LogLevel.Information,
        Message = "All {Count} migration(s) applied in {ElapsedMs}ms")]
    public static partial void MigrationsApplied(
        ILogger logger, int count, long elapsedMs);
}

// Usage
Log.ApplyingMigrations(logger, pending.Count, pending);
Log.MigrationsApplied(logger, pending.Count, sw.ElapsedMilliseconds);
```

## What Gets Exported

Each `ILogger` call produces a `LogRecord` with:
- `Timestamp`
- `LogLevel` → OTel severity
- `CategoryName` → `logger.name` attribute
- Message template → `body`
- Named parameters → individual attributes (when `ParseStateValues = true`)
- Scopes → additional attributes (when `IncludeScopes = true`)
- `TraceId` / `SpanId` → from `Activity.Current` (automatic)
- Exception type, message, stack trace (when an exception is logged)
