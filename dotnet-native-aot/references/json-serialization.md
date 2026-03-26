# JSON Serialization with Source Generators (AOT)

## Basic Setup

```csharp
using System.Text.Json.Serialization;

record Todo(int Id, string Title, bool IsComplete);

[JsonSourceGenerationOptions(
    PropertyNamingPolicy = JsonKnownNamingPolicy.CamelCase,
    WriteIndented = false)]
[JsonSerializable(typeof(Todo))]
[JsonSerializable(typeof(List<Todo>))]
internal partial class AppJsonContext : JsonSerializerContext;
```

## Register in ASP.NET Core

```csharp
builder.Services.ConfigureHttpJsonOptions(options =>
    options.SerializerOptions.TypeInfoResolverChain.Insert(0, AppJsonContext.Default));
```

## Direct Usage

```csharp
// Serialize with TypeInfo
string json = JsonSerializer.Serialize(todo, AppJsonContext.Default.Todo);

// Deserialize with TypeInfo
Todo? todo = JsonSerializer.Deserialize(json, AppJsonContext.Default.Todo);
```

## Combining Multiple Contexts

```csharp
var options = new JsonSerializerOptions
{
    TypeInfoResolver = JsonTypeInfoResolver.Combine(ContextA.Default, ContextB.Default)
};
// Or append:
options.TypeInfoResolverChain.Add(ContextC.Default);
```

## Generation Modes

```csharp
// Metadata only (deserialization support)
[JsonSourceGenerationOptions(GenerationMode = JsonSourceGenerationMode.Metadata)]

// Serialization only (fast path, best performance)
[JsonSourceGenerationOptions(GenerationMode = JsonSourceGenerationMode.Serialization)]

// Both (default)
[JsonSourceGenerationOptions(GenerationMode = JsonSourceGenerationMode.Default)]
```

## Enum Serialization

```csharp
// WRONG: non-generic is NOT AOT-compatible
[JsonConverter(typeof(JsonStringEnumConverter))]

// CORRECT: generic version is AOT-safe
[JsonConverter(typeof(JsonStringEnumConverter<MyEnum>))]
public enum MyEnum { Foo, Bar }

// Or blanket policy:
[JsonSourceGenerationOptions(UseStringEnumConverter = true)]
[JsonSerializable(typeof(MyType))]
internal partial class AppJsonContext : JsonSerializerContext;
```

## Object-Typed Members

When properties are `object`, register all concrete runtime types:

```csharp
public record Wrapper(object Data);

[JsonSerializable(typeof(Wrapper))]
[JsonSerializable(typeof(bool))]    // if Data could be bool
[JsonSerializable(typeof(int))]     // if Data could be int
[JsonSerializable(typeof(string))]  // if Data could be string
internal partial class AppJsonContext : JsonSerializerContext;
```

## Disable Reflection Fallback

```xml
<JsonSerializerIsReflectionEnabledByDefault>false</JsonSerializerIsReflectionEnabledByDefault>
```

Makes accidental reflection-based serialization throw `InvalidOperationException` at runtime instead of silently working (and then failing in published AOT binary).

## HttpClient JSON (AOT-safe)

```csharp
// Instead of:
var result = await http.GetFromJsonAsync<MyType>(url);

// Use TypeInfo overload:
var result = await http.GetFromJsonAsync(url, AppJsonContext.Default.MyType);

// POST:
await http.PostAsJsonAsync(url, data, AppJsonContext.Default.MyType);
```

## Common Mistakes

1. **Forgetting `List<T>`** -- registering `[JsonSerializable(typeof(MyType))]` but not `[JsonSerializable(typeof(List<MyType>))]`
2. **Anonymous types** -- `new { x = 1 }` can't be source-generated, use named records
3. **Missing nullable** -- `[JsonSerializable(typeof(MyType?))]` needed for nullable value types
4. **Derived types** -- polymorphism requires `[JsonDerivedType]` attributes on the base type
