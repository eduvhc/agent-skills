# Data Access with Native AOT

## Compatibility Matrix

| Library | AOT Status |
|---------|------------|
| Raw ADO.NET (SqliteCommand, SqlConnection, etc.) | Fully Compatible |
| Microsoft.Data.Sqlite | Fully Compatible |
| Npgsql | Fully Compatible |
| Dapper.AOT | Compatible (source-generated) |
| Dapper (standard) | NOT Compatible (uses Reflection.Emit) |
| Entity Framework Core | NOT Compatible |
| Linq2Db | Compatible |

## Recommended: Raw ADO.NET

Zero reflection, fully AOT-safe, best performance.

### Repository Pattern

```csharp
class TodoRepository
{
    private readonly string _connStr;

    public TodoRepository(string connectionString) => _connStr = connectionString;

    private SqliteConnection Open()
    {
        var conn = new SqliteConnection(_connStr);
        conn.Open();
        return conn;
    }

    public List<Todo> GetAll()
    {
        using var db = Open();
        using var cmd = db.CreateCommand();
        cmd.CommandText = "SELECT id, title, is_complete FROM todos ORDER BY id";
        var items = new List<Todo>();
        using var r = cmd.ExecuteReader();
        while (r.Read())
            items.Add(new Todo(r.GetInt32(0), r.GetString(1), r.GetBoolean(2)));
        return items;
    }

    public Todo? GetById(int id)
    {
        using var db = Open();
        using var cmd = db.CreateCommand();
        cmd.CommandText = "SELECT id, title, is_complete FROM todos WHERE id = $id";
        cmd.Parameters.AddWithValue("$id", id);
        using var r = cmd.ExecuteReader();
        return r.Read() ? new Todo(r.GetInt32(0), r.GetString(1), r.GetBoolean(2)) : null;
    }

    public void Insert(string title)
    {
        using var db = Open();
        using var cmd = db.CreateCommand();
        cmd.CommandText = "INSERT INTO todos (title, is_complete) VALUES ($title, 0)";
        cmd.Parameters.AddWithValue("$title", title);
        cmd.ExecuteNonQuery();
    }

    public void Delete(int id)
    {
        using var db = Open();
        using var cmd = db.CreateCommand();
        cmd.CommandText = "DELETE FROM todos WHERE id = $id";
        cmd.Parameters.AddWithValue("$id", id);
        cmd.ExecuteNonQuery();
    }
}
```

### Transactions

```csharp
public void TransferItems(int fromId, int toId)
{
    using var db = Open();
    using var tx = db.BeginTransaction();
    try
    {
        ExecTx(db, tx, "UPDATE items SET owner = $to WHERE id = $id",
            ("$to", toId), ("$id", fromId));
        ExecTx(db, tx, "UPDATE counts SET total = total - 1 WHERE owner = $from",
            ("$from", fromId));
        tx.Commit();
    }
    catch
    {
        tx.Rollback();
        throw;
    }
}

private static void ExecTx(SqliteConnection db, SqliteTransaction tx, string sql,
    params (string Name, object Value)[] parameters)
{
    using var cmd = db.CreateCommand();
    cmd.Transaction = tx;
    cmd.CommandText = sql;
    foreach (var (name, value) in parameters)
        cmd.Parameters.AddWithValue(name, value);
    cmd.ExecuteNonQuery();
}
```

### Nullable Column Reading

```csharp
// Check for DBNull before reading
var title = r.IsDBNull(1) ? null : r.GetString(1);

// For parameters, use DBNull.Value for null
cmd.Parameters.AddWithValue("$notes", (object?)notes ?? DBNull.Value);
```

## Alternative: Dapper.AOT

Uses C# interceptors to replace Dapper calls at compile time:

```xml
<PropertyGroup>
    <InterceptorsPreviewNamespaces>$(InterceptorsPreviewNamespaces);Dapper.AOT</InterceptorsPreviewNamespaces>
</PropertyGroup>
<ItemGroup>
    <PackageReference Include="Dapper" Version="2.1.35" />
    <PackageReference Include="Dapper.AOT" Version="1.0.48" />
</ItemGroup>
```

```csharp
[module: DapperAot]  // Enable AOT interception globally

// Then use Dapper normally -- calls are intercepted at compile time
var items = connection.Query<Todo>("SELECT id, title FROM todos");
```

Note: Dapper.AOT intercepts specific Dapper method calls. Not all Dapper features are supported.

## Why Not EF Core?

EF Core relies on:
- Reflection for change tracking
- Runtime code generation in query pipeline
- Dynamic proxy creation

Not AOT-compatible even in .NET 10. Future roadmap item for the EF team.
