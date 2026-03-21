# opentelemetry-dotnet

OpenTelemetry for .NET 10 skill covering all three signals, exporters, instrumentation libraries, and production best practices.

**Core SDK version: 1.15.0**

## What this skill covers

- **Tracing** — `ActivitySource`, spans, events, links, parent-child relationships, status codes
- **Metrics** — `IMeterFactory`, Counter, Histogram, Gauge, ObservableGauge, `TagList`, exemplars, cardinality management
- **Logging** — `ILogger` bridge, log level filtering, trace/log correlation
- **Exporters** — OTLP (gRPC + HTTP/protobuf), Prometheus, Console, In-Memory, batch tuning
- **Sampling** — ParentBased, TraceIdRatioBased, AlwaysOn/Off, tail sampling, error capture
- **Context propagation** — W3C TraceContext, Baggage, B3, manual inject/extract
- **Semantic conventions** — HTTP, DB, messaging, resource attributes
- **Instrumentation libraries** — ASP.NET Core, HttpClient, EF Core, Npgsql, Redis, gRPC, SQL Server, .NET runtime
- **Hybrid auto+manual** — CLR profiler agent + manual SDK, Plugin API, Docker, Kubernetes, MassTransit, Azure Service Bus
- **Production** — resource attributes, self-diagnostics, retry exporters, graceful shutdown
- **Pitfalls** — 12 documented pitfalls with fixes (name mismatches, UseOtlpExporter conflicts, null Activity, cardinality explosions)

## Activation

This skill activates automatically when you ask about:
- Instrumenting ASP.NET Core, HttpClient, EF Core, gRPC, or queue consumers
- Setting up OTLP exporters for Aspire Dashboard, Jaeger, Grafana, or any OTLP backend
- Configuring metrics with proper cardinality and exemplars
- Implementing distributed tracing with context propagation
- Hybrid auto-instrumentation + manual spans in Docker or Kubernetes
- Troubleshooting missing telemetry, high memory, or export failures
- OpenTelemetry package versions or API signatures for .NET
