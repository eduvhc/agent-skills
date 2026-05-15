# Library Instrumentation Reference

How to instrument a **shared/reusable library** (NuGet, internal package) so it
emits OpenTelemetry **without forcing OTel on consumers**. Critical when most
consuming apps use only classic Application Insights, a different SDK, or no
telemetry SDK at all.

## Rule: a library takes NO SDK and NO `OpenTelemetry.Api` dependency

`packages.md` says "library authors reference `OpenTelemetry.Api` only". For a
**widely-shared** library that is too much: `OpenTelemetry.Api` (`ActivitySource`
extension semantics aside) drags `Baggage`, `Propagators`, version coupling.
Everything needed for tracing + W3C propagation is already in the BCL:

| Need | BCL type (in `System.Diagnostics.DiagnosticSource`) |
|---|---|
| Span source | `System.Diagnostics.ActivitySource` |
| Span | `System.Diagnostics.Activity` / `ActivityKind` / `ActivityStatusCode` |
| Parent from wire | `ActivityContext.TryParse(traceparent, tracestate, out ctx)` |
| Inject/extract W3C + baggage | `System.Diagnostics.DistributedContextPropagator.Current` |
| Exception on span | `ActivityEvent("exception", tags:{exception.type/message/stacktrace})` |

`System.Diagnostics.DiagnosticSource` is almost always already a transitive
dependency (ASP.NET Core, HttpClient, most clients). Reference it explicitly
and pin to what the consuming surface already resolves — no new version.

## Safe-by-default: no listener ⇒ no-op

A library must not change behavior for apps without OTel.

- `ActivitySource.StartActivity(...)` returns **`null`** when no `ActivityListener`
  samples the source (i.e. no OTel SDK / classic App Insights only). Every
  `activity?.X()` then no-ops. **Zero behavioral change.**
- Classic Application Insights worker SDK does **not** auto-listen to arbitrary
  `ActivitySource`s — so even AI apps get the null/no-op path until they opt in
  with `AddSource("Your.Library")`.
- New public params for trace context **must be optional** (`string? traceParent = null`)
  so existing callers compile unchanged.
- Instrumentation **must never throw into the business path**. Wrap context
  inject/extract in `try/catch`; a telemetry failure must not fail a publish or
  a message handler.
- Context injection adds at most an inert `traceparent`/`tracestate`/`baggage`
  header that non-OTel consumers ignore (and AI honors W3C anyway).

## Centralize: one diagnostics class, zero magic strings

Do **not** scatter `activity.SetTag("messaging.system","rabbitmq")` across call
sites. One `static` class owns the `ActivitySource`, the semantic-convention
attribute **keys and values** as `const`, the span **factories**, exception
recording, and propagation. Call sites never see a magic string and never
reference `System.Diagnostics`.

```csharp
public static class MessagingDiagnostics
{
    public const string ActivitySourceName = "MyCompany.Messaging";
    public static readonly ActivitySource ActivitySource = new(ActivitySourceName);

    // semconv (stable messaging) — keys AND values centralized
    private const string AttrSystem          = "messaging.system";
    private const string AttrOperationType   = "messaging.operation.type";
    private const string AttrDestinationName = "messaging.destination.name";
    private const string AttrErrorType       = "error.type";
    private const string SystemValue         = "rabbitmq";
    private const string OpPublish           = "publish";
    private const string OpProcess           = "process";

    public static Activity StartPublish(string destination)
    {
        var a = ActivitySource.StartActivity($"{destination} {OpPublish}", ActivityKind.Producer);
        if (a != null)
        {
            a.SetTag(AttrSystem, SystemValue);
            a.SetTag(AttrOperationType, OpPublish);
            a.SetTag(AttrDestinationName, destination);
        }
        return a;
    }

    public static Activity StartConsume(string destination, string traceParent, string traceState)
    {
        Activity a;
        try
        {
            a = !string.IsNullOrEmpty(traceParent)
                && ActivityContext.TryParse(traceParent, traceState, out var parent)
                ? ActivitySource.StartActivity($"{destination} {OpProcess}", ActivityKind.Consumer, parent)
                : ActivitySource.StartActivity($"{destination} {OpProcess}", ActivityKind.Consumer);
        }
        catch { return null; }            // instrumentation never breaks consume

        if (a != null)
        {
            a.SetTag(AttrSystem, SystemValue);
            a.SetTag(AttrOperationType, OpProcess);
            a.SetTag(AttrDestinationName, destination);
        }
        return a;
    }

    // .NET 8 has no Activity.AddException (that is .NET 9+). Record the
    // spec-compliant exception event by hand — no OpenTelemetry.Api needed.
    public static void RecordException(Activity activity, Exception ex)
    {
        if (activity is null || ex is null) return;
        activity.SetStatus(ActivityStatusCode.Error, ex.Message);
        activity.SetTag(AttrErrorType, ex.GetType().FullName);
        activity.AddEvent(new ActivityEvent("exception", tags: new ActivityTagsCollection
        {
            { "exception.type", ex.GetType().FullName },
            { "exception.message", ex.Message },
            { "exception.stacktrace", ex.ToString() },
        }));
    }
}
```

Call sites stay clean:

```csharp
using var activity = MessagingDiagnostics.StartPublish(exchange);
MessagingDiagnostics.InjectTraceContext(headers, activity);
// ...
catch (Exception ex) { MessagingDiagnostics.RecordException(activity, ex); throw; }
```

> Span status: leave **Unset** on success (do not force `Ok`); set `Error`
> only on failure. Explicit `Ok` is discouraged for library instrumentation.

## BCL-only propagation (no `OpenTelemetry.Api`)

`propagation.md` shows `Propagators.DefaultTextMapPropagator` — that needs
`OpenTelemetry.Api`. The dependency-free equivalent is the BCL
`DistributedContextPropagator` (same W3C tracecontext + baggage, used under the
hood by ASP.NET Core / HttpClient):

```csharp
// PRODUCER — inject into message headers
public static void InjectTraceContext(IDictionary<string, object> headers, Activity activity)
{
    var source = activity ?? Activity.Current;
    if (source is null) return;
    DistributedContextPropagator.Current.Inject(source, headers, static (carrier, key, value) =>
    {
        if (carrier is IDictionary<string, object> h) h[key] = value;
    });
}

// CONSUMER — extract from a carrier, then feed into StartConsume(...)
DistributedContextPropagator.Current.ExtractTraceIdAndState(
    carrier,
    static (object c, string key, out string? v, out IEnumerable<string>? vs) =>
    {
        vs = null;
        v = c is IDictionary<string, object> h && h.TryGetValue(key, out var o) ? o?.ToString() : null;
    },
    out var traceParent, out var traceState);
```

When the consumer transport surfaces headers as an opaque blob (e.g. the Azure
Functions RabbitMQ trigger base64-encodes header values into `BindingData`),
add a small carrier-specific extractor in the diagnostics class and pass the
resulting `traceParent`/`traceState` strings to `StartConsume(...)`.

## Checklist

- [ ] No `OpenTelemetry`/`OpenTelemetry.Api` package reference in the library.
- [ ] Only `System.Diagnostics.DiagnosticSource` (explicit, pinned to the already-resolved version).
- [ ] One `static` diagnostics class; no `SetTag("…")` / `System.Diagnostics` at call sites.
- [ ] semconv keys **and** values as `const`; span name `{destination} {operation}`.
- [ ] New trace-context params optional; existing callers unchanged.
- [ ] Inject/extract + start wrapped so instrumentation never throws into business code.
- [ ] Verified no-op build path: with no `AddSource`, `StartActivity` is null and behavior is identical.
- [ ] Consumers opt in via `.AddSource("Your.Library")` (and `AddSource` for the app's own source).
