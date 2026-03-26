# Native AOT Project Setup (.NET 10)

## csproj Configuration

### Minimal Setup

```xml
<PropertyGroup>
    <TargetFramework>net10.0</TargetFramework>
    <PublishAot>true</PublishAot>
    <JsonSerializerIsReflectionEnabledByDefault>false</JsonSerializerIsReflectionEnabledByDefault>
</PropertyGroup>
```

`PublishAot=true` implies:
- Enables static AOT analysis during `dotnet build`
- Enables trimming during publish
- Produces native binary during `dotnet publish`

### Library Projects

```xml
<PropertyGroup>
    <!-- Marks assembly as AOT-compatible (emits assembly metadata in .NET 10) -->
    <IsAotCompatible>true</IsAotCompatible>
</PropertyGroup>
```

`IsAotCompatible=true` implies:
- `IsTrimmable` = true
- `EnableTrimAnalyzer` = true
- `EnableSingleFileAnalyzer` = true
- `EnableAotAnalyzer` = true

### Verify Dependencies

```xml
<!-- Warns if referenced assemblies lack IsAotCompatible (.NET 10+) -->
<VerifyReferenceAotCompatibility>true</VerifyReferenceAotCompatibility>
```

## Size Optimization

### OptimizationPreference

```xml
<!-- Smaller binary (default for AOT) -->
<OptimizationPreference>Size</OptimizationPreference>

<!-- Faster execution -->
<OptimizationPreference>Speed</OptimizationPreference>
```

### Feature Switches (strip unused features)

```xml
<PropertyGroup>
    <InvariantGlobalization>true</InvariantGlobalization>
    <StackTraceSupport>false</StackTraceSupport>
    <UseSystemResourceKeys>true</UseSystemResourceKeys>
    <DebuggerSupport>false</DebuggerSupport>
    <EventSourceSupport>false</EventSourceSupport>
    <HttpActivityPropagationSupport>false</HttpActivityPropagationSupport>
    <MetadataUpdaterSupport>false</MetadataUpdaterSupport>
    <MetricsSupport>false</MetricsSupport>
    <!-- .NET 10 additions -->
    <Http3Support>false</Http3Support>
    <UseSizeOptimizedLinq>true</UseSizeOptimizedLinq>
</PropertyGroup>
```

`UseSizeOptimizedLinq` defaults to `true` with `PublishAot`. Trades LINQ throughput optimizations for smaller binary.

## Publishing

```bash
# Linux x64
dotnet publish -r linux-x64 -c Release

# Linux ARM64
dotnet publish -r linux-arm64 -c Release

# Alpine (musl)
dotnet publish -r linux-musl-x64 -c Release

# Windows
dotnet publish -r win-x64 -c Release
```

## Binary Size Progress (.NET 7 to .NET 10)

| Version | Console App Size |
|---------|-----------------|
| .NET 7  | 3.65 MB         |
| .NET 8  | 1.39 MB         |
| .NET 9  | 1.22 MB         |
| .NET 10 | 1.05 MB         |

## Testing AOT Readiness

### Without Publishing

```bash
# Analyzers run during build when PublishAot=true
dotnet build --no-incremental
```

Zero IL warnings = likely AOT-compatible. The only definitive test is `dotnet publish` + running the binary.

### View Generated Source

```xml
<EmitCompilerGeneratedFiles>true</EmitCompilerGeneratedFiles>
```

Generated files appear in `obj/Debug/net10.0/generated/`.
