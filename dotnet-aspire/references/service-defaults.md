# ServiceDefaults Reference

## Project Setup

```xml
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <TargetFramework>net10.0</TargetFramework>
    <IsAspireSharedProject>true</IsAspireSharedProject>
  </PropertyGroup>

  <ItemGroup>
    <FrameworkReference Include="Microsoft.AspNetCore.App" />
    <PackageReference Include="Microsoft.Extensions.Http.Resilience" />
    <PackageReference Include="Microsoft.Extensions.ServiceDiscovery" />
    <PackageReference Include="OpenTelemetry.Exporter.OpenTelemetryProtocol" />
    <PackageReference Include="OpenTelemetry.Extensions.Hosting" />
    <PackageReference Include="OpenTelemetry.Instrumentation.AspNetCore" />
    <PackageReference Include="OpenTelemetry.Instrumentation.Http" />
    <PackageReference Include="OpenTelemetry.Instrumentation.Runtime" />
  </ItemGroup>
</Project>
```

## Canonical Extensions.cs

```csharp
using Microsoft.AspNetCore.Builder;
using Microsoft.AspNetCore.Diagnostics.HealthChecks;
using Microsoft.Extensions.DependencyInjection;
using Microsoft.Extensions.Diagnostics.HealthChecks;
using Microsoft.Extensions.Hosting;
using Microsoft.Extensions.Logging;
using OpenTelemetry;
using OpenTelemetry.Metrics;
using OpenTelemetry.Trace;

namespace Microsoft.Extensions.Hosting;

public static class ServiceDefaultsExtensions
{
    public static TBuilder AddServiceDefaults<TBuilder>(this TBuilder builder)
        where TBuilder : IHostApplicationBuilder
    {
        builder.ConfigureOpenTelemetry();
        builder.AddDefaultHealthChecks();

        builder.Services.AddServiceDiscovery();
        builder.Services.ConfigureHttpClientDefaults(http =>
        {
            http.AddStandardResilienceHandler();
            http.AddServiceDiscovery();
        });

        return builder;
    }

    public static TBuilder ConfigureOpenTelemetry<TBuilder>(this TBuilder builder)
        where TBuilder : IHostApplicationBuilder
    {
        builder.Logging.AddOpenTelemetry(logging =>
        {
            logging.IncludeFormattedMessage = true;
            logging.IncludeScopes = true;
        });

        builder.Services.AddOpenTelemetry()
            .WithMetrics(metrics =>
            {
                metrics.AddAspNetCoreInstrumentation()
                       .AddHttpClientInstrumentation()
                       .AddRuntimeInstrumentation();
            })
            .WithTracing(tracing =>
            {
                tracing.AddSource(builder.Environment.ApplicationName)
                       .AddAspNetCoreInstrumentation(o =>
                       {
                           // Filter health check noise from traces
                           o.Filter = ctx =>
                               !ctx.Request.Path.StartsWithSegments("/health") &&
                               !ctx.Request.Path.StartsWithSegments("/alive");
                       })
                       .AddHttpClientInstrumentation();
            });

        builder.AddOpenTelemetryExporters();
        return builder;
    }

    private static TBuilder AddOpenTelemetryExporters<TBuilder>(this TBuilder builder)
        where TBuilder : IHostApplicationBuilder
    {
        var useOtlpExporter = !string.IsNullOrWhiteSpace(
            builder.Configuration["OTEL_EXPORTER_OTLP_ENDPOINT"]);

        if (useOtlpExporter)
            builder.Services.AddOpenTelemetry().UseOtlpExporter();

        return builder;
    }

    public static TBuilder AddDefaultHealthChecks<TBuilder>(this TBuilder builder)
        where TBuilder : IHostApplicationBuilder
    {
        builder.Services.AddHealthChecks()
            .AddCheck("self", () => HealthCheckResult.Healthy(), tags: ["live"]);
        return builder;
    }

    public static WebApplication MapDefaultEndpoints(this WebApplication app)
    {
        // Only expose in development — avoids leaking internal state in prod
        if (app.Environment.IsDevelopment())
        {
            app.MapHealthChecks("/health");
            app.MapHealthChecks("/alive", new HealthCheckOptions
            {
                Predicate = r => r.Tags.Contains("live")
            });
        }
        return app;
    }
}
```

## Using ServiceDefaults in a Web Service

```csharp
var builder = WebApplication.CreateBuilder(args);

builder.AddServiceDefaults();

// Aspire client integrations hook into OTel automatically
builder.AddNpgsqlDbContext<AppDbContext>("appdb");
builder.AddRedisDistributedCache("redis");

var app = builder.Build();

app.MapDefaultEndpoints();   // /health, /alive
app.MapGet("/", () => "Hello World");
app.Run();
```

## Worker / Migrator Pattern (No ASP.NET Core)

Workers don't have an HTTP pipeline — do NOT call `MapDefaultEndpoints`.
Do NOT use `AddServiceDefaults()` from the web variant if the project lacks `Microsoft.AspNetCore.App`.
Instead, call `ConfigureOpenTelemetry()` directly and register your custom ActivitySource alongside it:

```csharp
// Worker Program.cs
var builder = Host.CreateApplicationBuilder(args);

builder.AddServiceDefaults();

// Register the worker's own activity source on top of service defaults
builder.Services.AddOpenTelemetry()
    .WithTracing(tracing => tracing
        .AddSource(MigrationWorker.ActivitySourceName));

builder.AddNpgsqlDbContext<AppDbContext>("scrapio");
builder.Services.AddHostedService<MigrationWorker>();

await builder.Build().RunAsync();
```

## Service Discovery

No hardcoded URLs. Use the AppHost resource name as the hostname:

```csharp
// Resolves to whatever port Aspire assigned to the "api" resource
var client = httpClientFactory.CreateClient("api");
```

In production, DNS or a service mesh handles the same name.

## Common Aspire Client Packages (Service Side)

| Resource | Package |
|---|---|
| PostgreSQL (raw) | `Aspire.Npgsql` |
| PostgreSQL + EF Core | `Aspire.Npgsql.EntityFrameworkCore.PostgreSQL` |
| Redis cache | `Aspire.StackExchange.Redis` |
| Redis distributed cache | `Aspire.StackExchange.Redis.DistributedCaching` |
| RabbitMQ | `Aspire.RabbitMQ.Client` |
| Azure Service Bus | `Aspire.Azure.Messaging.ServiceBus` |
| Azure Blob Storage | `Aspire.Azure.Storage.Blobs` |
