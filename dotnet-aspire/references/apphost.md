# AppHost Reference

## Project Setup (.NET 10 / Aspire 13)

```xml
<Project Sdk="Aspire.AppHost.Sdk">
  <PropertyGroup>
    <TargetFramework>net10.0</TargetFramework>
  </PropertyGroup>
</Project>
```

No additional imports needed — the SDK provides `Aspire.Hosting` transitively.

## Minimal Program.cs

```csharp
var builder = DistributedApplication.CreateBuilder(args);

var postgres = builder.AddPostgres("postgres")
    .WithDataVolume()
    .AddDatabase("appdb");

var redis = builder.AddRedis("redis");

var api = builder.AddProject<Projects.MyApp_ApiService>("api")
    .WithReference(postgres)
    .WithReference(redis)
    .WaitFor(postgres)
    .WaitFor(redis);

builder.AddProject<Projects.MyApp_Web>("webfrontend")
    .WithReference(api)
    .WaitFor(api)
    .WithExternalHttpEndpoints();

builder.Build().Run();
```

## Resource Naming Rules

- Lowercase, alphanumeric + hyphens only: `my-api` not `MyApi`
- Names become DNS labels for service discovery: `http://my-api` resolves in services
- Names appear in traces, logs, and dashboard — choose descriptively

## WaitFor vs WaitForCompletion

```csharp
// WaitFor — long-running resource must reach Running/Healthy state
.WaitFor(postgres)

// WaitForCompletion — job/migration must exit with code 0
.WaitForCompletion(migrationJob)
```

## Dependency Pattern: Migrator Before API

```csharp
var postgres = builder.AddPostgres("postgres").AddDatabase("appdb");

var migrator = builder.AddProject<Projects.MyApp_Migrator>("migrator")
    .WithReference(postgres)
    .WaitFor(postgres);

builder.AddProject<Projects.MyApp_ApiService>("api")
    .WithReference(postgres)
    .WaitForCompletion(migrator);  // API only starts after migrator exits 0
```

## Parameters and Secrets

```csharp
// AppHost
var dbPassword = builder.AddParameter("db-password", secret: true);
var postgres = builder.AddPostgres("postgres", password: dbPassword);
```

Resolution order for `Parameters__db-password`:
1. Environment variable `Parameters__db-password`
2. `appsettings.json` → `"Parameters": { "db-password": "..." }`
3. User Secrets
4. Interactive dashboard prompt (fallback)

## External Connection Strings

For resources not managed by Aspire (cloud DB, external Redis, etc.):

```csharp
var externalDb = builder.AddConnectionString("external-db");
builder.AddProject<Projects.MyApp_ApiService>("api")
    .WithReference(externalDb);
```

Value read from config key `ConnectionStrings__external-db`.

## Conditional Volumes (Test-Friendly)

```csharp
var postgres = builder.AddPostgres("postgres").AddDatabase("appdb");

if (builder.Configuration.GetValue("UseVolumes", defaultValue: true))
    postgres.WithDataVolume();
```

Tests pass `["UseVolumes=false"]` to avoid Docker volume permission issues on Windows.

## Container Lifetime

```csharp
// Default — container recreated each run
builder.AddPostgres("postgres").WithDataVolume();

// Persistent — survives AppHost restarts (useful for large DBs in dev)
builder.AddPostgres("postgres")
    .WithDataVolume()
    .WithLifetime(ContainerLifetime.Persistent);
```

## Lifecycle Events

```csharp
builder.Eventing.Subscribe<BeforeStartEvent>((e, ct) =>
{
    Console.WriteLine("About to start all resources");
    return Task.CompletedTask;
});

builder.Eventing.Subscribe<AfterEndpointsAllocatedEvent>((e, ct) =>
{
    // All ports are now known
    return Task.CompletedTask;
});

builder.Eventing.Subscribe<AfterResourcesCreatedEvent>((e, ct) =>
{
    // All resource objects exist
    return Task.CompletedTask;
});
```

## .NET 10 / Aspire 13 New Features

```csharp
// AddCSharpApp — no project reference needed
var worker = builder.AddCSharpApp("worker", "../MyWorker/Program.cs")
    .WithReference(postgres);

// Dev Tunnels — expose local service externally
builder.AddProject<Projects.Api>("api").WithDevTunnel();

// OpenAI integration
var openai = builder.AddOpenAI("openai");
builder.AddProject<Projects.Api>("api").WithReference(openai);

// MCP server hosting
builder.AddMcpServer("my-mcp-server", "../McpServer")
    .WithReference(vectorDb);
```

## Common Hosting Packages

| Integration | Package |
|---|---|
| PostgreSQL | `Aspire.Hosting.PostgreSQL` |
| SQL Server | `Aspire.Hosting.SqlServer` |
| Redis | `Aspire.Hosting.Redis` |
| RabbitMQ | `Aspire.Hosting.RabbitMQ` |
| MongoDB | `Aspire.Hosting.MongoDB` |
| Kafka | `Aspire.Hosting.Kafka` |
| Azure Service Bus | `Aspire.Hosting.Azure.ServiceBus` |
| Azure Blob Storage | `Aspire.Hosting.Azure.Storage` |
