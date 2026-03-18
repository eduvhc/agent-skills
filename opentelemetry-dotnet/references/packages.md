# Package Inventory

All packages target .NET 8.0, 9.0, 10.0, .NET Standard 2.0/2.1, .NET Framework 4.6.2+.

## Core SDK

| Package | Version | Purpose |
|---|---|---|
| `OpenTelemetry` | **1.15.0** | Core SDK — TracerProvider, MeterProvider, logging bridge, sampling, processors |
| `OpenTelemetry.Api` | **1.15.0** | API-only for library authors — ActivitySource, Meter, Baggage. No SDK dependency |
| `OpenTelemetry.Extensions.Hosting` | **1.15.0** | `AddOpenTelemetry()` DI integration, manages provider lifecycle with Generic Host |

> **Library authors:** reference `OpenTelemetry.Api` only. Never take a hard dependency on the SDK in a library — the app decides which SDK version to use.

## Exporters

| Package | Version | Status | Purpose |
|---|---|---|---|
| `OpenTelemetry.Exporter.OpenTelemetryProtocol` | **1.15.0** | Stable | OTLP over gRPC (port 4317) or HTTP/protobuf (port 4318) — primary production exporter |
| `OpenTelemetry.Exporter.Console` | **1.15.0** | Stable | Stdout output — dev/debugging only, never production |
| `OpenTelemetry.Exporter.Prometheus.AspNetCore` | **1.15.0-beta.1** | Beta | Prometheus scrape endpoint via ASP.NET Core middleware |
| `OpenTelemetry.Exporter.Prometheus.HttpListener` | **1.15.0-beta.1** | Beta | Prometheus scrape endpoint for non-ASP.NET Core apps |
| `OpenTelemetry.Exporter.Zipkin` | **1.15.0** | Stable (legacy) | Zipkin format — prefer OTLP for new deployments |
| `OpenTelemetry.Exporter.InMemory` | **1.15.0** | Stable | In-memory collection for unit testing |

## Instrumentation — Core Repo

| Package | Version | Status | Purpose |
|---|---|---|---|
| `OpenTelemetry.Instrumentation.AspNetCore` | **1.15.1** | Stable | Incoming HTTP/gRPC spans, ASP.NET Core metrics |
| `OpenTelemetry.Instrumentation.Http` | **1.15.0** | Stable | Outgoing `HttpClient` traces and metrics |
| `OpenTelemetry.Instrumentation.Runtime` | **1.15.0** | Stable | .NET runtime metrics (GC, thread pool, JIT, memory) |

## Instrumentation — Contrib Repo

| Package | Version | Status | Purpose |
|---|---|---|---|
| `OpenTelemetry.Instrumentation.EntityFrameworkCore` | **1.15.0-beta.1** | **Beta** | EF Core query tracing (relational only, no Cosmos DB) |
| `OpenTelemetry.Instrumentation.StackExchangeRedis` | **1.15.0-beta.1** | **Beta** | StackExchange.Redis command tracing |
| `OpenTelemetry.Instrumentation.GrpcNetClient` | **1.15.0-beta.1** | **Beta** | Outgoing `Grpc.Net.Client` calls |
| `OpenTelemetry.Instrumentation.SqlClient` | **1.15.0-beta.1** | **Beta** | ADO.NET SqlClient database traces |
| `OpenTelemetry.Extensions.Propagators` | **1.15.0** | Stable | B3 single/multi-header propagators (Zipkin compat) |

## Native OTel Integrations

| Package | Version | Status | Purpose |
|---|---|---|---|
| `Npgsql.OpenTelemetry` | **10.0.1** | Stable | Npgsql-native tracing — preferred over EF Core instrumentation for Postgres |
| `RabbitMQ.Client.OpenTelemetry` | **1.0.0-rc.1** | RC | RabbitMQ.Client 7.0+ native OTel (6.x requires bytecode instrumentation) |

## Directory.Packages.props (Central Package Management)

```xml
<!-- Core OTel -->
<PackageVersion Include="OpenTelemetry" Version="1.15.0" />
<PackageVersion Include="OpenTelemetry.Api" Version="1.15.0" />
<PackageVersion Include="OpenTelemetry.Extensions.Hosting" Version="1.15.0" />

<!-- Exporter -->
<PackageVersion Include="OpenTelemetry.Exporter.OpenTelemetryProtocol" Version="1.15.0" />
<PackageVersion Include="OpenTelemetry.Exporter.Console" Version="1.15.0" />
<PackageVersion Include="OpenTelemetry.Exporter.InMemory" Version="1.15.0" />

<!-- Instrumentation (stable) -->
<PackageVersion Include="OpenTelemetry.Instrumentation.AspNetCore" Version="1.15.1" />
<PackageVersion Include="OpenTelemetry.Instrumentation.Http" Version="1.15.0" />
<PackageVersion Include="OpenTelemetry.Instrumentation.Runtime" Version="1.15.0" />

<!-- Instrumentation (beta) -->
<PackageVersion Include="OpenTelemetry.Instrumentation.EntityFrameworkCore" Version="1.15.0-beta.1" />
<PackageVersion Include="OpenTelemetry.Instrumentation.StackExchangeRedis" Version="1.15.0-beta.1" />
<PackageVersion Include="OpenTelemetry.Instrumentation.GrpcNetClient" Version="1.15.0-beta.1" />

<!-- Native -->
<PackageVersion Include="Npgsql.OpenTelemetry" Version="10.0.1" />
```
