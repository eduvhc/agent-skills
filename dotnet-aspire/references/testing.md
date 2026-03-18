# Integration Testing Reference

## Overview

Aspire testing is **closed-box integration testing** — it launches the full AppHost (all services, all containers) and tests make real calls against running resources.

- Services run in separate processes — you cannot mock/substitute them
- For isolated service testing, use `WebApplicationFactory<T>` instead
- Container startup is slow (30–120 s) — **always use shared fixtures**
- Ports are randomised by default — never hardcode them; use `app.CreateHttpClient("name")`

## Package Setup

```xml
<ItemGroup>
    <!-- Test framework -->
    <PackageReference Include="Aspire.Hosting.Testing" />   <!-- version = AppHost SDK version -->
    <PackageReference Include="Microsoft.NET.Test.Sdk" />
    <PackageReference Include="MSTest" />                    <!-- or xunit / nunit -->
</ItemGroup>

<ItemGroup>
    <!-- CRITICAL: prevents AppHost treating test project as a child resource -->
    <ProjectReference Include="../MyApp.AppHost/MyApp.AppHost.csproj"
                      IsAspireProjectResource="false" />
</ItemGroup>
```

Scaffold an empty test project:
```bash
dotnet new aspire-mstest -o MyApp.Tests
dotnet new aspire-xunit  -o MyApp.Tests
dotnet new aspire-nunit  -o MyApp.Tests
```

## Core API Reference

### `WaitForResourceAsync` — Three Overloads

All are extension/instance methods on `ResourceNotificationService` (accessed via `app.ResourceNotifications`).

```csharp
// Overload 1 — single target state
Task WaitForResourceAsync(
    string resourceName,
    string? targetState = default,
    CancellationToken cancellationToken = default);

// Overload 2 — wait for ANY of several states; returns the state that was reached
Task<string> WaitForResourceAsync(
    string resourceName,
    IEnumerable<string> targetStates,
    CancellationToken cancellationToken = default);

// Overload 3 — arbitrary predicate on the ResourceEvent (Aspire 9+)
Task<ResourceEvent> WaitForResourceAsync(
    string resourceName,
    Func<ResourceEvent, bool> predicate,
    CancellationToken cancellationToken = default);
```

All overloads throw `OperationCanceledException` on timeout and `InvalidOperationException` if the resource does not exist.

### `WaitForResourceHealthyAsync`

```csharp
// Wait until a resource is healthy (or Running, if it has no health check annotation)
Task<ResourceEvent> WaitForResourceHealthyAsync(
    string resourceName,
    CancellationToken cancellationToken = default);

// Fail fast when resource enters a terminal/unavailable state
Task<ResourceEvent> WaitForResourceHealthyAsync(
    string resourceName,
    WaitBehavior waitBehavior,
    CancellationToken cancellationToken = default);
```

**Default behaviour (`WaitOnResourceUnavailable`):** keeps waiting even if the resource enters `FailedToStart` (allows restart). Use `StopOnResourceUnavailable` in tests for fail-fast behaviour.

```csharp
// Recommended for tests — throws DistributedApplicationException immediately on failure
await app.ResourceNotifications.WaitForResourceHealthyAsync(
    "postgres",
    WaitBehavior.StopOnResourceUnavailable,
    cts.Token);
```

### `KnownResourceStates` — Complete List

```csharp
KnownResourceStates.NotStarted        // created, not yet started
KnownResourceStates.Waiting           // waiting on a dependency (WaitFor/WaitForCompletion)
KnownResourceStates.Starting          // starting up
KnownResourceStates.Running           // process is running (not necessarily healthy)
KnownResourceStates.Active            // resources without an explicit lifetime
KnownResourceStates.Stopping          // graceful shutdown in progress
KnownResourceStates.Exited            // process exited (any exit code)
KnownResourceStates.Finished          // process exited cleanly — use for migrators/init jobs
KnownResourceStates.FailedToStart     // could not start
KnownResourceStates.RuntimeUnhealthy  // Docker / DCP itself is unhealthy
KnownResourceStates.TerminalStates    // IReadOnlyList<string> of Exited, Finished, FailedToStart
```

## Shared Fixture Patterns

Starting the AppHost per-test multiplies startup cost (30–120 s per test run). Always share a single `DistributedApplication` across all tests in a class or assembly.

### MSTest — Class-level (`ClassInitialize`)

```csharp
[TestClass]
public class AppHostTests
{
    private static DistributedApplication _app = null!;

    [ClassInitialize]
    public static async Task InitializeAsync(TestContext _)
    {
        var appHost = await DistributedApplicationTestingBuilder
            .CreateAsync<Projects.MyApp_AppHost>(["UseVolumes=false"]);

        appHost.Services.ConfigureHttpClientDefaults(b => b.AddStandardResilienceHandler());

        _app = await appHost.BuildAsync();
        await _app.StartAsync();

        using var cts = new CancellationTokenSource(TimeSpan.FromMinutes(5));
        await _app.ResourceNotifications
            .WaitForResourceHealthyAsync("api", WaitBehavior.StopOnResourceUnavailable, cts.Token);
    }

    [ClassCleanup(ClassCleanupBehavior.EndOfClass)]
    public static async Task CleanupAsync() => await _app.DisposeAsync();

    [TestMethod]
    public async Task GetOrders_ReturnsOk()
    {
        using var client = _app.CreateHttpClient("api");
        var response = await client.GetAsync("/orders");
        Assert.AreEqual(HttpStatusCode.OK, response.StatusCode);
    }
}
```

### MSTest — Assembly-level (`AssemblyInitialize`)

Share one `DistributedApplication` across **all test classes** in the assembly:

```csharp
[TestClass]
public static class AssemblySetup
{
    public static DistributedApplication App { get; private set; } = null!;

    [AssemblyInitialize]
    public static async Task InitializeAsync(TestContext _)
    {
        var appHost = await DistributedApplicationTestingBuilder
            .CreateAsync<Projects.MyApp_AppHost>(["UseVolumes=false"]);

        appHost.Services.ConfigureHttpClientDefaults(b => b.AddStandardResilienceHandler());

        App = await appHost.BuildAsync();
        await App.StartAsync();

        using var cts = new CancellationTokenSource(TimeSpan.FromMinutes(5));
        await App.ResourceNotifications
            .WaitForResourceHealthyAsync("api", WaitBehavior.StopOnResourceUnavailable, cts.Token);
    }

    [AssemblyCleanup]
    public static async Task CleanupAsync() => await App.DisposeAsync();
}

// Any test class references the shared app via AssemblySetup.App
[TestClass]
public class OrdersTests
{
    [TestMethod]
    public async Task GetOrders_ReturnsOk()
    {
        using var client = AssemblySetup.App.CreateHttpClient("api");
        var response = await client.GetAsync("/orders");
        Assert.AreEqual(HttpStatusCode.OK, response.StatusCode);
    }
}
```

### xUnit — Collection Fixture

```csharp
// AspireFixture.cs
public sealed class AspireFixture : IAsyncLifetime
{
    private static readonly TimeSpan StartupTimeout = TimeSpan.FromMinutes(10);
    private static readonly TimeSpan HealthTimeout  = TimeSpan.FromMinutes(5);

    public DistributedApplication App { get; private set; } = null!;

    public async Task InitializeAsync()
    {
        var appHost = await DistributedApplicationTestingBuilder
            .CreateAsync<Projects.MyApp_AppHost>(["UseVolumes=false"]);

        appHost.Services.ConfigureHttpClientDefaults(b => b.AddStandardResilienceHandler());
        appHost.Services.AddLogging(l =>
        {
            l.AddFilter("Aspire.Hosting.Dcp",              LogLevel.Warning);
            l.AddFilter("Microsoft.Hosting.Lifetime",       LogLevel.Warning);
        });

        App = await appHost.BuildAsync();

        using var startCts = new CancellationTokenSource(StartupTimeout);
        await App.StartAsync(startCts.Token);

        using var healthCts = new CancellationTokenSource(HealthTimeout);
        await App.ResourceNotifications
            .WaitForResourceHealthyAsync("api", WaitBehavior.StopOnResourceUnavailable, healthCts.Token);
    }

    public async Task DisposeAsync() => await App.DisposeAsync();
}

[CollectionDefinition("Aspire", DisableParallelization = true)]
public class AspireCollection : ICollectionFixture<AspireFixture> { }

[Collection("Aspire")]
public class ApiTests(AspireFixture fixture)
{
    [Fact]
    public async Task GetOrders_ReturnsOk()
    {
        using var client = fixture.App.CreateHttpClient("api");
        var response = await client.GetAsync("/orders");
        Assert.Equal(HttpStatusCode.OK, response.StatusCode);
    }
}
```

### NUnit — `OneTimeSetUp`

```csharp
[TestFixture]
public class ApiTests
{
    private DistributedApplication _app = null!;

    [OneTimeSetUp]
    public async Task SetUp()
    {
        var appHost = await DistributedApplicationTestingBuilder
            .CreateAsync<Projects.MyApp_AppHost>(["UseVolumes=false"]);

        _app = await appHost.BuildAsync();
        await _app.StartAsync();

        using var cts = new CancellationTokenSource(TimeSpan.FromMinutes(5));
        await _app.ResourceNotifications
            .WaitForResourceHealthyAsync("api", WaitBehavior.StopOnResourceUnavailable, cts.Token);
    }

    [OneTimeTearDown]
    public async Task TearDown() => await _app.DisposeAsync();

    [Test]
    public async Task GetRoot_ReturnsOk()
    {
        using var client = _app.CreateHttpClient("api");
        var response = await client.GetAsync("/");
        Assert.That(response.StatusCode, Is.EqualTo(HttpStatusCode.OK));
    }
}
```

## Overriding AppHost Configuration

### Method A — Args array (config key=value)

```csharp
var appHost = await DistributedApplicationTestingBuilder
    .CreateAsync<Projects.MyApp_AppHost>(["UseVolumes=false", "FeatureFlags:NewCheckout=true"]);
```

The AppHost reads these via `builder.Configuration`:
```csharp
// AppHost Program.cs
if (builder.Configuration.GetValue("UseVolumes", defaultValue: true))
    postgres.WithDataVolume();
```

### Method B — `configureBuilder` action (before host is built)

```csharp
var appHost = await DistributedApplicationTestingBuilder.CreateAsync<Projects.MyApp_AppHost>(
    args: [],
    configureBuilder: (appOptions, hostSettings) =>
    {
        appOptions.DisableDashboard = false;   // re-enable dashboard for debugging
        hostSettings.Configuration ??= new ConfigurationManager();
        hostSettings.Configuration["AZURE_SUBSCRIPTION_ID"] = "test-sub";
    });
```

### Method C — Configure services after builder creation

```csharp
appHost.Services.ConfigureHttpClientDefaults(b => b.AddStandardResilienceHandler());
appHost.Services.AddLogging(l => l.SetMinimumLevel(LogLevel.Debug));
```

## `DistributedApplicationFactory` — Advanced Lifecycle

Use when you need code to run **before the AppHost builder is even instantiated** (not possible with `DistributedApplicationTestingBuilder`), or to share a base class across multiple test projects.

```csharp
public class TestAppHostFactory()
    : DistributedApplicationFactory(typeof(Projects.MyApp_AppHost))
{
    // Called first — inject config before the builder exists
    protected override void OnBuilderCreating(
        DistributedApplicationOptions appOptions,
        HostApplicationBuilderSettings hostOptions)
    {
        hostOptions.Configuration ??= new ConfigurationManager();
        hostOptions.Configuration["Environment"] = "Testing";
        hostOptions.Configuration["UseVolumes"]  = "false";
    }

    // Called after builder exists but before Build()
    protected override void OnBuilderCreated(DistributedApplicationBuilder builder)
    {
        builder.Services.ConfigureHttpClientDefaults(b => b.AddStandardResilienceHandler());
    }

    // Called after BuildAsync()
    protected override void OnBuilt(DistributedApplication app) { }
}

// Usage
await using var factory = new TestAppHostFactory();
await using var app    = await factory.BuildAsync();
await app.StartAsync();
```

**`DistributedApplicationTestingBuilder` vs `DistributedApplicationFactory`:**

| Scenario | Use |
|---|---|
| Standard integration tests | `DistributedApplicationTestingBuilder` |
| Need config before AppHost builder is created | `DistributedApplicationFactory` |
| Base class shared across multiple test projects | `DistributedApplicationFactory` |

## Accessing Resources in Tests

```csharp
// HTTP client (service-discovery-aware, uses randomised port)
using var client = app.CreateHttpClient("api");

// Connection string (after app.StartAsync)
string? connStr = await app.GetConnectionStringAsync("postgres");

// Connection string from builder (before start — for AppHost config inspection)
var db = appHost.Resources.OfType<PostgresDatabaseResource>().Single(r => r.Name == "db");
string? connStr = await db.ConnectionStringExpression.GetValueAsync(CancellationToken.None);

// Endpoint URL
var endpoint = app.GetEndpoint("api", "http");
// endpoint.Url → "http://localhost:52345"

// Verify environment variable wiring
var project = appHost.Resources.OfType<ProjectResource>().Single(r => r.Name == "api");
var envVars = await project.GetEffectiveEnvironmentAsync();
Assert.Contains("services__database__https__0", envVars.Keys);
```

## Testing Worker Services (Non-HTTP)

Worker services have no HTTP endpoints. Test them via state, logs, or side effects.

### Assert process is running
```csharp
using var cts = new CancellationTokenSource(TimeSpan.FromMinutes(2));
await app.ResourceNotifications
    .WaitForResourceAsync("myworker", KnownResourceStates.Running, cts.Token);
```

### Assert exit-style worker completes

```csharp
// Wait for any terminal state; returns the state that was reached
using var cts = new CancellationTokenSource(TimeSpan.FromMinutes(3));
string finalState = await app.ResourceNotifications
    .WaitForResourceAsync("scrapio-migrator", KnownResourceStates.TerminalStates, cts.Token);

Assert.AreEqual(KnownResourceStates.Finished, finalState);
```

### Assert on log output
```csharp
var logService = app.Services.GetRequiredService<ResourceLoggerService>();
using var cts = new CancellationTokenSource(TimeSpan.FromSeconds(30));

await foreach (var batch in logService.WatchAsync("myworker").WithCancellation(cts.Token))
{
    if (batch.Any(l => l.Content.Contains("Processing complete")))
        return; // success
}
Assert.Fail("Expected log line not found");
```

### Assert on database side effects
```csharp
await app.ResourceNotifications
    .WaitForResourceAsync("myworker", KnownResourceStates.Finished, cts.Token);

string? connStr = await app.GetConnectionStringAsync("postgres");
await using var conn = new NpgsqlConnection(connStr);
await conn.OpenAsync();
var count = await conn.ExecuteScalarAsync<int>("SELECT COUNT(*) FROM processed_items");
Assert.IsTrue(count > 0);
```

## Database Seeding

```csharp
// Wait for DB to be healthy, then seed before tests run
using var cts = new CancellationTokenSource(TimeSpan.FromMinutes(3));
await app.ResourceNotifications.WaitForResourceHealthyAsync("postgres", cts.Token);

string? connStr = await app.GetConnectionStringAsync("postgres");
await using var conn = new NpgsqlConnection(connStr);
await conn.OpenAsync();
await conn.ExecuteAsync("INSERT INTO products (name, price) VALUES ('Test', 9.99)");
```

## Resilience Testing — Start/Stop Resources

```csharp
// Stop a resource to test failure handling
await app.ResourceCommands.ExecuteCommandAsync(
    "redis", KnownResourceCommands.StopCommand, cts.Token);

// Assert your service handles the outage, then recover
await app.ResourceCommands.ExecuteCommandAsync(
    "redis", KnownResourceCommands.StartCommand, cts.Token);

await app.ResourceNotifications
    .WaitForResourceHealthyAsync("redis", WaitBehavior.StopOnResourceUnavailable, cts.Token);
```

## Pitfalls and Solutions

### P1: Hardcoded Ports

```csharp
// WRONG — port changes every run
var client = new HttpClient { BaseAddress = new Uri("http://localhost:5001") };

// CORRECT
using var client = app.CreateHttpClient("api");
```

### P2: No Timeout on `WaitForResourceHealthyAsync`

```csharp
// WRONG — hangs forever if the resource fails
await app.ResourceNotifications.WaitForResourceHealthyAsync("postgres");

// CORRECT — bounded token + fail-fast behaviour
using var cts = new CancellationTokenSource(TimeSpan.FromMinutes(3));
await app.ResourceNotifications
    .WaitForResourceHealthyAsync("postgres", WaitBehavior.StopOnResourceUnavailable, cts.Token);
```

### P3: Missing `IsAspireProjectResource="false"`

```xml
<!-- WRONG — AppHost tries to orchestrate the test project, causes recursion -->
<ProjectReference Include="../MyApp.AppHost/MyApp.AppHost.csproj" />

<!-- CORRECT -->
<ProjectReference Include="../MyApp.AppHost/MyApp.AppHost.csproj"
                  IsAspireProjectResource="false" />
```

### P4: Model Types Cause AppHost to Launch API Project

If you reference an API project only to use its model types, Aspire treats it as a resource.

**Fix:** Extract models into a `MyApp.Contracts` project — reference that freely without side effects.

### P5: Volume State Leaks Between Test Runs

```csharp
// Pass UseVolumes=false in the test builder
var appHost = await DistributedApplicationTestingBuilder
    .CreateAsync<Projects.AppHost>(["UseVolumes=false"]);

// AppHost Program.cs reads the flag:
if (builder.Configuration.GetValue("UseVolumes", defaultValue: true))
    postgres.WithDataVolume();
```

### P6: `inotify` Exhaustion on Linux / CI

File-change watchers created by every hosted service can exhaust the OS `inotify` limit on CI runners.

```csharp
// Add to test project — runs before any test code
[assembly: ModuleInitializer]
internal static class TestModuleInitializer
{
    [ModuleInitializer]
    internal static void Initialize() =>
        Environment.SetEnvironmentVariable("DOTNET_HOSTBUILDER__RELOADCONFIGONCHANGE", "false");
}
```

### P7: Health Check Hangs Silently (Default `WaitOnResourceUnavailable`)

By default, `WaitForResourceHealthyAsync` keeps retrying even after `FailedToStart`. Switch to `StopOnResourceUnavailable` in tests so failures surface immediately with an exception rather than a timeout.

### P8: Dashboard Disabled in Tests (by design)

Dashboard is disabled by default under `DistributedApplicationTestingBuilder`. To re-enable for local debugging only:

```csharp
appHost.Services.Configure<DistributedApplicationOptions>(o => o.DisableDashboard = false);
```

### P9: `"relation already exists"` — Migration Applied to Existing DB

Tables exist but `__EFMigrationsHistory` has no record.

```sql
CREATE TABLE IF NOT EXISTS "__EFMigrationsHistory" (
    "MigrationId"    character varying(150) NOT NULL,
    "ProductVersion" character varying(32)  NOT NULL,
    CONSTRAINT "PK___EFMigrationsHistory" PRIMARY KEY ("MigrationId")
);

INSERT INTO "__EFMigrationsHistory" ("MigrationId", "ProductVersion")
VALUES ('20260317192249_InitialCreate', '9.0.6');
```

### P10: Randomised Ports Break Debugging

```csharp
// Disable port randomisation — use only for local debugging, never CI
var appHost = await DistributedApplicationTestingBuilder
    .CreateAsync<Projects.AppHost>(["DcpPublisher:RandomizePorts=false"]);
```

## AppHost Startup Ordering (Migrator Pattern)

```csharp
// AppHost/Program.cs
var postgres = builder.AddPostgres("postgres").AddDatabase("appdb");

var migrator = builder.AddProject<Projects.MyApp_Migrator>("migrator")
    .WithReference(postgres)
    .WaitFor(postgres);

builder.AddProject<Projects.MyApp_Api>("api")
    .WithReference(postgres)
    .WaitForCompletion(migrator);  // API won't start until migrator exits

// Test — verify the full chain
using var cts = new CancellationTokenSource(TimeSpan.FromMinutes(5));

await app.ResourceNotifications
    .WaitForResourceHealthyAsync("postgres", WaitBehavior.StopOnResourceUnavailable, cts.Token);

string migratorFinalState = await app.ResourceNotifications
    .WaitForResourceAsync("migrator", KnownResourceStates.TerminalStates, cts.Token);
Assert.AreEqual(KnownResourceStates.Finished, migratorFinalState);

await app.ResourceNotifications
    .WaitForResourceHealthyAsync("api", WaitBehavior.StopOnResourceUnavailable, cts.Token);
```

## New APIs by Aspire Version

| API | Added in | Notes |
|---|---|---|
| `WaitForResourceHealthyAsync` | 9.0 | Not available in 8.x |
| `WaitForResourceAsync(predicate)` | 9.0 | Predicate overload not in 8.x |
| `WaitBehavior.StopOnResourceUnavailable` | 9.0 | Fail-fast for tests |
| `KnownResourceStates.RuntimeUnhealthy` | 9.0 | Docker/DCP unhealthy |
| `KnownResourceStates.Active` | 9.0 | Resources without explicit lifetime |
| `KnownResourceStates.NotStarted` | 9.0 | — |
| `KnownResourceStates.TerminalStates` | 9.0 | List of Exited/Finished/FailedToStart |
| `ResourceCommands.ExecuteCommandAsync` | 9.0 | Start/stop resources in tests |
| `WaitForResourceAsync` throws on missing resource | 9.0 | — |
| `DistributedApplicationTestingBuilder.Create` (sync) | 13.0 | Sync variant added |
| `ResourceNotificationServiceOptions.DefaultWaitBehavior` | 9.0 | Global `WaitBehavior` default |
