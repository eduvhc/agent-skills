# Health Checks Reference

## Two Distinct Health Check Systems

| System | Purpose | Who polls it |
|---|---|---|
| **AppHost resource health checks** | Gates `WaitFor` dependency ordering | DCP (Developer Control Plane) |
| **Application `/health` endpoint** | Signals readiness for traffic | Load balancers, K8s probes, dashboard |

## AppHost Health Checks (WaitFor Gating)

### WithHttpHealthCheck (Aspire 9.3+)

```csharp
// Aspire polls this endpoint to determine when the resource is ready
builder.AddContainer("legacy-service", "my-registry/legacy:latest")
    .WithHttpEndpoint(targetPort: 8080)
    .WithHttpHealthCheck("/health");

builder.AddProject<Projects.Api>("api")
    .WaitFor(legacyService);   // waits until /health returns 200
```

> **Breaking change in 9.3:** `WithHttpsHealthCheck` is obsolete. `WithHttpHealthCheck` now prefers `https` endpoints when available.

```csharp
// OLD (deprecated in 9.3)
.WithHttpsHealthCheck("/health")

// NEW — prefers https if present
.WithHttpHealthCheck("/health")

// NEW — explicit scheme
.WithHttpHealthCheck("/health", endpointName: "http")
```

### Custom AppHost Health Check

```csharp
var startTime = DateTime.UtcNow;
builder.Services.AddHealthChecks()
    .AddCheck("warmup-delay", () =>
        DateTime.UtcNow - startTime > TimeSpan.FromSeconds(10)
            ? HealthCheckResult.Healthy()
            : HealthCheckResult.Unhealthy("Still warming up"));

var myResource = builder.AddPostgres("pg")
    .WithHealthCheck("warmup-delay");

builder.AddProject<Projects.Api>("api").WaitFor(myResource);
```

## Application Health Checks (MapDefaultEndpoints)

Exposed by `MapDefaultEndpoints()` in ServiceDefaults (development only):

| Endpoint | Tags | Purpose |
|---|---|---|
| `/health` | all | Overall readiness |
| `/alive` | `live` | Liveness (is the process alive?) |

```csharp
// ServiceDefaults Extension
public static WebApplication MapDefaultEndpoints(this WebApplication app)
{
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
```

## Disabling Integration Health Checks

Aspire client integrations (Npgsql, Redis, etc.) register health checks by default. Disable them per-integration:

```json
// appsettings.json in the service project
{
  "Aspire": {
    "Npgsql": {
      "DisableHealthChecks": true
    },
    "StackExchange.Redis": {
      "DisableHealthChecks": false
    }
  }
}
```

## Adding Custom Health Checks to a Service

```csharp
builder.Services.AddHealthChecks()
    .AddCheck("self", () => HealthCheckResult.Healthy(), tags: ["live"])
    .AddCheck<DatabaseHealthCheck>("database", tags: ["ready"])
    .AddUrlGroup(new Uri("http://external-dep/health"), "external", tags: ["ready"]);
```

## WaitFor vs WaitForCompletion

```csharp
// WaitFor — resource must reach Running/Healthy (for long-lived services)
builder.AddProject<Projects.Api>("api")
    .WaitFor(postgres);

// WaitForCompletion — resource must exit with code 0 (for jobs/migrations)
builder.AddProject<Projects.Api>("api")
    .WaitForCompletion(migrator);
```

## Kubernetes Probe Mapping

```yaml
# Map Aspire health endpoints to K8s probes
livenessProbe:
  httpGet:
    path: /alive
    port: 8080
  initialDelaySeconds: 10
  periodSeconds: 10

readinessProbe:
  httpGet:
    path: /health
    port: 8080
  initialDelaySeconds: 5
  periodSeconds: 5
```
