# Trimming Warnings and Resolution

## Common Warning Codes

| Code | Meaning | Fix |
|------|---------|-----|
| IL2026 | Calling `[RequiresUnreferencedCode]` method | Use source-generated alternative |
| IL2057 | `Type.GetType(string)` with variable | Use compile-time type reference |
| IL2067 | Passing unannotated Type to `DynamicallyAccessedMembers` method | Add attribute |
| IL2070 | Reflection on Type missing annotation | Add `[DynamicallyAccessedMembers]` |
| IL2072 | Return/extracted value missing annotation | Propagate annotation |
| IL3050 | Calling `[RequiresDynamicCode]` method | Replace with non-dynamic pattern |
| IL3058 | Referenced assembly not `IsAotCompatible` | Update package or find alternative |

## Resolution Strategies

### Strategy A: Add DynamicallyAccessedMembers

```csharp
// Before (IL2070):
void Process(Type t) { var method = t.GetMethod("Foo"); }

// After:
void Process(
    [DynamicallyAccessedMembers(DynamicallyAccessedMemberTypes.PublicMethods)] Type t)
{
    var method = t.GetMethod("Foo");
}
```

### Strategy B: Mark as incompatible (bubble up)

```csharp
[RequiresUnreferencedCode("Loads plugins by name")]
[RequiresDynamicCode("Uses Reflection.Emit")]
public void LoadPlugin(string name) { ... }
```

### Strategy C: Source-generated serialization

```csharp
// Before (IL2026):
JsonSerializer.Serialize(obj);

// After:
JsonSerializer.Serialize(obj, AppJsonContext.Default.MyType);
```

### Strategy D: Suppress (last resort)

```csharp
[UnconditionalSuppressMessage("AOT", "IL3050:RequiresDynamicCode",
    Justification = "Not reachable in AOT path")]
void SafeMethod() { ... }
```

**Never use** `#pragma warning disable` for IL warnings -- hides from Roslyn analyzer but the linker still sees issues.

## Build to Collect All Warnings

```bash
dotnet build --no-incremental
```

## MSBuild Properties

```xml
<!-- Show all warnings per-call-site, not per-assembly -->
<TrimmerSingleWarn>false</TrimmerSingleWarn>

<!-- Don't fail build on trimming warnings -->
<ILLinkTreatWarningsAsErrors>false</ILLinkTreatWarningsAsErrors>

<!-- Enable Roslyn trim analyzer (implied by PublishAot) -->
<EnableTrimAnalyzer>true</EnableTrimAnalyzer>
```
