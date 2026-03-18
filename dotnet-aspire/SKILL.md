---
name: dotnet-aspire
description: .NET Aspire (Aspire 13 / .NET 10) skill covering AppHost orchestration, ServiceDefaults, OpenTelemetry integration, integration testing with DistributedApplicationTestingBuilder, health checks, service discovery, and best practices. Use for any .NET Aspire development task.
references:
  - apphost
  - service-defaults
  - testing
  - opentelemetry
  - health-checks
---

# .NET Aspire Skill (Aspire 13 / .NET 10)

Skill for building cloud-native distributed applications with .NET Aspire. Covers the full stack: orchestration, observability, and integration testing.

## Retrieval Sources

Fetch the **latest** information before citing specific API signatures, package versions, or configuration options. Do not rely on pre-training alone.

| Source | URL | Use for |
|--------|-----|---------|
| Aspire docs | `https://aspire.dev/docs/` | All official docs |
| Aspire API reference | `https://aspire.dev/` | API shapes, package versions |
| Aspire GitHub | `https://github.com/dotnet/aspire` | Issues, releases, breaking changes |
| Breaking changes | `https://learn.microsoft.com/en-us/dotnet/aspire/compatibility/` | Version-specific breaking changes |
| .NET Blog | `https://devblogs.microsoft.com/dotnet/` | New features, announcements |

When pre-trained knowledge and the docs disagree, **trust the docs** — especially for package versions, API signatures, and breaking changes.

## Quick Decision Trees

### "What project do I need?"

```
Setting up .NET Aspire?
├─ Orchestration / startup coordination → AppHost project
├─ Shared OTel + health + resilience config → ServiceDefaults project
├─ Long-lived services / APIs / workers → reference ServiceDefaults
├─ Integration tests against the real stack → Tests project + Aspire.Hosting.Testing
└─ Short-lived migration / job → Worker SDK + BackgroundService pattern
```

### "I need to add a resource"

```
Resource type?
├─ PostgreSQL → builder.AddPostgres("name").AddDatabase("db")
├─ Redis → builder.AddRedis("name")
├─ RabbitMQ → builder.AddRabbitMQ("name")
├─ SQL Server → builder.AddSqlServer("name")
├─ External DB (not managed by Aspire) → builder.AddConnectionString("name")
├─ Secret / parameter → builder.AddParameter("name", secret: true)
└─ Container → builder.AddContainer("name", "image:tag")
```

### "I need to test my Aspire app"

```
Test scope?
├─ Full stack (real containers, real HTTP) → DistributedApplicationTestingBuilder
├─ Single service in isolation → WebApplicationFactory<T> (no Aspire needed)
├─ AppHost config / env var wiring → DistributedApplicationTestingBuilder + GetEffectiveEnvironmentAsync
├─ Need config injected before AppHost builds → DistributedApplicationFactory (OnBuilderCreating)
├─ Worker service (non-HTTP) → assert via KnownResourceStates, ResourceLoggerService, or DB side effects
└─ Resilience / failure testing → ResourceCommands.ExecuteCommandAsync(StopCommand/StartCommand)
```

### "Something is not working"

```
Problem?
├─ "relation already exists" migration error → see references/testing.md § DB Pitfalls
├─ WaitForResourceHealthyAsync hangs → bounded CancellationToken + health check logging
├─ Port conflicts in tests → use CreateHttpClient("name"), never hardcode ports
├─ Container volume permissions on Windows → pass UseVolumes=false to test builder
├─ AppHost reference causes recursion → IsAspireProjectResource="false" on ProjectReference
└─ OTel not exported in production → verify OTEL_EXPORTER_OTLP_ENDPOINT env var is set
```

## Architecture at a Glance

```
┌──────────────────────────────────────────────────────────────────┐
│  Solution                                                        │
│  ├─ MyApp.AppHost          ← orchestration (dev/test only)       │
│  ├─ MyApp.ServiceDefaults  ← OTel + health + resilience (shared) │
│  ├─ MyApp.ApiService       ← service, references ServiceDefaults │
│  ├─ MyApp.Migrator         ← worker, references ServiceDefaults  │
│  └─ MyApp.Tests            ← integration tests                   │
└──────────────────────────────────────────────────────────────────┘

Developer Control Plane (DCP) — engine that starts/monitors processes
Aspire Dashboard — built-in OTLP collector + UI for local dev
```

## Core OTel Environment Variables (Injected by DCP)

| Variable | Example | Purpose |
|---|---|---|
| `OTEL_SERVICE_NAME` | `api` | Service identifier in all telemetry |
| `OTEL_RESOURCE_ATTRIBUTES` | `service.instance.id=abc` | Resource metadata |
| `OTEL_EXPORTER_OTLP_ENDPOINT` | `http://localhost:18889` | Dashboard OTLP gRPC port |

In production these must be set by your deployment environment — Aspire does not set them outside local dev.

## Key APIs Quick Reference

| API | Package | Purpose |
|---|---|---|
| `DistributedApplicationTestingBuilder.CreateAsync<T>()` | `Aspire.Hosting.Testing` | Bootstrap AppHost in tests |
| `app.CreateHttpClient("name")` | `Aspire.Hosting.Testing` | Service-discovery-aware HTTP client |
| `app.GetConnectionStringAsync("name")` | `Aspire.Hosting.Testing` | Get resource connection string in tests |
| `app.ResourceNotifications.WaitForResourceHealthyAsync("name", ct)` | `Aspire.Hosting` | Block until resource healthy |
| `builder.AddServiceDefaults()` | ServiceDefaults project | OTel + health + resilience |
| `app.MapDefaultEndpoints()` | ServiceDefaults project | `/health` + `/alive` endpoints |
| `builder.AddParameter("name", secret: true)` | `Aspire.Hosting` | Secrets / parameters |
| `builder.AddConnectionString("name")` | `Aspire.Hosting` | External connection strings |
| `.WaitFor(resource)` | `Aspire.Hosting` | Dependency startup ordering |
| `.WaitForCompletion(resource)` | `Aspire.Hosting` | Wait for job/migration exit |
| `.WithHttpHealthCheck("/path")` | `Aspire.Hosting` | HTTP health probe (replaces WithHttpsHealthCheck in 9.3+) |
| `.WithDataVolume()` | `Aspire.Hosting.*` | Persistent container volume |
| `.WithLifetime(ContainerLifetime.Persistent)` | `Aspire.Hosting` | Persist container across restarts |
| `.WithReference(resource)` | `Aspire.Hosting` | Inject dependency config/URLs |

## Detailed References

- `references/apphost.md` — AppHost patterns, resources, parameters, lifecycle
- `references/service-defaults.md` — ServiceDefaults setup, worker variants
- `references/opentelemetry.md` — OTel tracing, metrics, logging, production config
- `references/testing.md` — Integration testing, all framework fixtures (MSTest/xUnit/NUnit), WaitForResourceAsync overloads, WaitBehavior, KnownResourceStates, worker testing, seeding, resilience, 10 pitfalls
- `references/health-checks.md` — AppHost health checks vs app health checks
