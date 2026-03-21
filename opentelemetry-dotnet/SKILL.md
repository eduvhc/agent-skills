---
name: opentelemetry-dotnet
description: OpenTelemetry for .NET 10 skill covering tracing (ActivitySource, spans, semantic conventions), metrics (IMeterFactory, instruments, exemplars), logging (ILogger bridge), exporters (OTLP, Prometheus, Console), sampling, context propagation, and all instrumentation libraries. Use for any OpenTelemetry .NET implementation task, OTel setup, observability questions, or telemetry debugging.
version: 1.0.0
references:
  - packages
  - setup
  - tracing
  - metrics
  - logging
  - exporters
  - sampling
  - propagation
  - semantic-conventions
  - instrumentation
  - production
  - pitfalls
---

# OpenTelemetry for .NET 10 Skill

**Core SDK version: 1.15.0 | Current as of March 2026**

Authoritative skill for implementing OpenTelemetry in .NET 10 applications. Covers all three signals (traces, metrics, logs), all exporters, semantic conventions, and production best practices.

## Retrieval Sources

Always fetch latest before citing package versions or API signatures.

| Source | URL | Use for |
|--------|-----|---------|
| OTel .NET docs | `https://opentelemetry.io/docs/languages/dotnet/` | API reference, getting started |
| OTel .NET GitHub | `https://github.com/open-telemetry/opentelemetry-dotnet` | Source, changelogs, issues |
| OTel Contrib GitHub | `https://github.com/open-telemetry/opentelemetry-dotnet-contrib` | Instrumentation library source |
| Semantic conventions | `https://opentelemetry.io/docs/specs/semconv/` | Attribute names, required/recommended |
| NuGet | `https://www.nuget.org/packages/OpenTelemetry/` | Latest stable versions |
| MS Docs | `https://learn.microsoft.com/en-us/dotnet/core/diagnostics/observability-with-otel` | .NET-specific guidance |

## Quick Decision Trees

### "What do I need to instrument?"

```
Signal?
├─ Incoming/outgoing HTTP, DB queries, queue messages → Tracing (ActivitySource)
├─ Counts, durations, queue depths, error rates → Metrics (IMeterFactory)
└─ Application log messages → Logging (ILogger bridge — zero code change)
```

### "Which exporter?"

```
Export target?
├─ Aspire Dashboard / Grafana / Jaeger / any OTLP backend → UseOtlpExporter() or AddOtlpExporter()
├─ Prometheus scrape → AddPrometheusExporter() (beta)
├─ Local debugging → AddConsoleExporter() (dev only, NEVER production)
└─ Unit tests → AddInMemoryExporter()
```

### "What instrumentation package?"

```
Can you modify the source code?
├─ YES → Manual instrumentation (NuGet packages, AddXxx() calls)
│   ├─ ASP.NET Core (incoming requests) → AddAspNetCoreInstrumentation()
│   ├─ HttpClient (outgoing requests) → AddHttpClientInstrumentation()
│   ├─ EF Core → AddEntityFrameworkCoreInstrumentation() (beta)
│   ├─ Npgsql (direct) → AddNpgsql() from Npgsql.OpenTelemetry — preferred
│   ├─ Redis → AddRedisInstrumentation()
│   ├─ gRPC client → AddGrpcClientInstrumentation()
│   ├─ SQL Server (ADO.NET) → AddSqlClientInstrumentation()
│   └─ .NET runtime metrics → AddRuntimeInstrumentation()
├─ NO → Auto-Instrumentation (CLR profiler + startup hook, zero source changes)
│   ├─ Local/VM → install script + env vars
│   ├─ Docker → COPY from otel/dotnet-auto-instrumentation base image
│   └─ Kubernetes → OTel Operator + pod annotation inject-dotnet: "true"
└─ BOTH → Hybrid (recommended for greenfield) — see references/hybrid.md
    ├─ Auto agent handles: HTTP, DB, queues, gRPC baseline spans
    ├─ OTEL_DOTNET_AUTO_TRACES_ADDITIONAL_SOURCES=MyApp.* → your ActivitySource
    ├─ Plugin API (OTEL_DOTNET_AUTO_PLUGINS) → enrichment/filtering without SDK deps
    └─ MassTransit / Azure Service Bus → NOT in agent, register as ADDITIONAL_SOURCES
```

### "Something is missing / not showing up"

```
Problem?
├─ No spans in backend → ActivitySource name mismatch (case-sensitive exact match required)
├─ No metrics in backend → Meter name not registered with AddMeter()
├─ No export at all → OTEL_EXPORTER_OTLP_ENDPOINT not set; defaults to localhost:4317
├─ HTTP/protobuf 404 → Missing /v1/traces path on endpoint URL
├─ UseOtlpExporter throws NotSupportedException → Called more than once or mixed with AddOtlpExporter()
├─ Last spans dropped on shutdown → Provider not disposed; add ForceFlush()
└─ High memory usage → High-cardinality tag values (user IDs, raw URLs)
```

## Core Concepts

### The Three Signals

| Signal | .NET API | OTel Pipeline | SDK Type |
|---|---|---|---|
| Traces | `ActivitySource` / `Activity` | `TracerProvider` | `System.Diagnostics` |
| Metrics | `Meter` / instruments | `MeterProvider` | `System.Diagnostics.Metrics` |
| Logs | `ILogger` | Logging bridge | `Microsoft.Extensions.Logging` |

### Signal Flow

```
Application Code                  OTel SDK Pipeline              Backend
  ActivitySource.StartActivity() → TracerProvider
    → Sampler → Processor → BatchExporter → OTLP → Aspire/Jaeger/Grafana

  Meter.CreateCounter().Add()    → MeterProvider
    → Reader → BatchExporter    → OTLP → Prometheus/Grafana

  ILogger.LogInformation()       → OTel Logging Bridge
    → BatchLogRecordExporter     → OTLP → Loki/Grafana
```

## Package Version Cheatsheet

| Package | Version | Status |
|---|---|---|
| `OpenTelemetry` | 1.15.0 | Stable |
| `OpenTelemetry.Extensions.Hosting` | 1.15.0 | Stable |
| `OpenTelemetry.Exporter.OpenTelemetryProtocol` | 1.15.0 | Stable |
| `OpenTelemetry.Instrumentation.AspNetCore` | 1.15.1 | Stable |
| `OpenTelemetry.Instrumentation.Http` | 1.15.0 | Stable |
| `OpenTelemetry.Instrumentation.Runtime` | 1.15.0 | Stable |
| `OpenTelemetry.Instrumentation.EntityFrameworkCore` | 1.15.0-beta.1 | **Beta** |
| `OpenTelemetry.Instrumentation.StackExchangeRedis` | 1.15.0-beta.1 | **Beta** |
| `Npgsql.OpenTelemetry` | 10.0.1 | Stable |

## Critical Rules

1. **ActivitySource name must exactly match AddSource()** — case-sensitive, no wildcards unless using `"MyApp.*"` prefix pattern
2. **Meter name must exactly match AddMeter()** — same rule
3. **UseOtlpExporter() called once only** — cannot mix with AddOtlpExporter()
4. **Always null-check Activity** — `StartActivity()` returns `null` when no listener is active
5. **Static readonly ActivitySource/Meter** — never create per-request; extremely expensive
6. **IMeterFactory for DI apps** — prefer over static Meter for proper lifecycle and test isolation
7. **Dispose providers** — in non-DI apps, dispose TracerProvider/MeterProvider or the last batch is dropped

## Key APIs Quick Reference

| API | Purpose |
|---|---|
| `builder.Services.AddOpenTelemetry()` | Entry point for all OTel setup in DI apps |
| `.ConfigureResource(r => r.AddService(...))` | Set service.name, service.version, deployment.environment |
| `.WithTracing(t => t.AddSource("name"))` | Register ActivitySource for tracing |
| `.WithMetrics(m => m.AddMeter("name"))` | Register Meter for metrics |
| `.UseOtlpExporter()` | Export all signals to OTLP (one call covers traces+metrics+logs) |
| `builder.Logging.AddOpenTelemetry(...)` | Bridge ILogger to OTel pipeline |
| `activity?.AddException(ex)` | Record exception as span event with stack trace |
| `activity?.SetStatus(ActivityStatusCode.Error, msg)` | Mark span as failed |
| `activity?.IsAllDataRequested` | Check before running expensive tag-building code |
| `ExemplarFilterType.TraceBased` | Link metric measurements to active trace spans |

## Detailed References

- `references/packages.md` — full package inventory with versions and status
- `references/setup.md` — setup patterns (ASP.NET Core, Worker, minimal, Aspire)
- `references/tracing.md` — ActivitySource, spans, events, links, parent-child, status
- `references/metrics.md` — instruments, IMeterFactory, TagList, exemplars, cardinality
- `references/logging.md` — ILogger bridge, log level filtering, trace correlation
- `references/exporters.md` — OTLP (gRPC/HTTP), Console, Prometheus, In-Memory, batch tuning
- `references/sampling.md` — samplers, production guidance, tail sampling, error capture
- `references/propagation.md` — W3C TraceContext, Baggage, B3, manual inject/extract
- `references/semantic-conventions.md` — HTTP, DB, messaging, resource attributes
- `references/instrumentation.md` — all instrumentation libraries with config examples
- `references/hybrid.md` — hybrid auto+manual approach: Plugin API, RabbitMQ/Kafka/Postgres/Redis/EF Core/MassTransit/Azure Service Bus/gRPC patterns, Docker, support matrix
- `references/production.md` — resource attributes, batch config, self-diagnostics, retry
- `references/pitfalls.md` — 12 documented pitfalls with fixes and explanations
