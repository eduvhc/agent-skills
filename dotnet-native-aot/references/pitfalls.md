# Native AOT Common Pitfalls

## Reflection

- `Assembly.LoadFile` -- NOT supported, no dynamic assembly loading
- `System.Reflection.Emit` -- NOT supported
- `Type.MakeGenericType` -- produces IL3050 warning, may fail at runtime
- Walking Type graphs (reflection-based serializers) -- trimmer removes types you expect to reflect on
- Code works in `dotnet run` (JIT) but crashes after `dotnet publish` (AOT)

## Serialization

| Library | AOT Status |
|---------|------------|
| System.Text.Json (source generators) | Compatible |
| System.Text.Json (reflection) | NOT Compatible |
| Newtonsoft.Json | NOT Compatible |
| XmlSerializer | NOT Compatible |
| BinaryFormatter | Removed in .NET 9+ |

Key mistakes:
- Non-generic `JsonStringEnumConverter` -- use `JsonStringEnumConverter<TEnum>`
- Anonymous types in responses -- use named records
- Missing `List<T>` registration -- registering `T` but not `List<T>`

## Dynamic Code

- `System.Linq.Expressions` uses interpreted form (slower but works)
- Generic parameters with struct types -- every instantiation generates specialized code (disk size impact)
- Generic virtual methods -- create instantiations for every implementing type
- LINQ to objects works, but some optimized paths are removed with `UseSizeOptimizedLinq`

## Data Access

- **Dapper** -- uses `Reflection.Emit`, NOT compatible (use Dapper.AOT or raw ADO.NET)
- **EF Core** -- NOT compatible (reflection-based change tracking, runtime code gen)
- **Raw ADO.NET** -- always works
- **Npgsql, Microsoft.Data.Sqlite** -- compatible

## SignalR

- **`Hub<T>` (strongly-typed)** -- NOT supported, produces warnings + runtime crash
- **MessagePack protocol** -- NOT supported
- Use `Hub` with `SendAsync` string method names

## DI Container

- Microsoft.Extensions.DependencyInjection works with AOT
- Custom DI containers using reflection may not work
- Avoid `Activator.CreateInstance` in custom factories

## Assembly & Runtime

- No dynamic assembly loading at runtime
- Startup hooks (`DOTNET_STARTUP_HOOKS`) disabled
- C++/CLI not supported
- No built-in COM on Windows

## The "Works in Debug" Trap

The most dangerous pitfall: code runs fine with `dotnet run` (JIT mode) but crashes after `dotnet publish` (AOT). Always verify by publishing and testing the actual binary.

## Performance Tradeoffs

| Aspect | JIT | AOT |
|--------|-----|-----|
| Cold start | 250-500ms | 5-50ms |
| Memory | 100-150 MB | 30-60 MB |
| Peak throughput | Higher (after warmup) | Lower but deterministic |
| Binary size | Needs runtime (~200 MB container) | Self-contained (~18 MB Alpine) |
| Debugging | Full support | Limited |

## Real-World Adoption Numbers (.NET 10)

- **Aspire CLI**: ~15 MB native executable
- **Windows App SDK 1.6**: 50% startup reduction, ~8x package size reduction
- **PowerToys Command Palette**: ~15% memory reduction, ~40% load time improvement
