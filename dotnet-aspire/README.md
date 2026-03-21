# dotnet-aspire

.NET Aspire (Aspire 13 / .NET 10) skill for building cloud-native distributed applications.

## What this skill covers

- **AppHost orchestration** — resource coordination, container lifecycle, startup ordering, `.WaitFor()`, `.WaitForCompletion()`
- **ServiceDefaults** — shared OTel, health checks, resilience configuration
- **Resources** — PostgreSQL, Redis, RabbitMQ, SQL Server, containers, parameters, connection strings
- **Integration testing** — `DistributedApplicationTestingBuilder`, all test frameworks (xUnit / NUnit / MSTest), `WaitForResourceAsync` overloads, `WaitBehavior`, `KnownResourceStates`
- **Worker / migration testing** — non-HTTP service assertions via `ResourceLoggerService` and DB side effects
- **Resilience / failure testing** — `ResourceCommands.ExecuteCommandAsync(StopCommand/StartCommand)`
- **Health checks** — AppHost probes vs app-level health endpoints, `/health` + `/alive`
- **Observability** — OTel config injected by DCP, production environment variable requirements
- **Common pitfalls** — port conflicts, volume permissions, migration races, AppHost recursion

## Activation

This skill activates automatically when you ask about:
- Setting up or configuring a .NET Aspire solution
- AppHost, ServiceDefaults, or DCP (Developer Control Plane)
- Integration testing with `DistributedApplicationTestingBuilder`
- Adding resources (PostgreSQL, Redis, RabbitMQ, SQL Server, containers)
- Health check configuration in Aspire projects
- OTel / observability setup via Aspire ServiceDefaults
