# Semantic Conventions Reference

Semantic conventions define standardized attribute names. Using them ensures interoperability with all OTel-compatible backends and dashboards.

## HTTP Server Spans (Stable — v1.23)

| Attribute | Required? | Example |
|---|---|---|
| `http.request.method` | Required | `"GET"`, `"POST"` |
| `url.path` | Required | `"/api/orders"` |
| `url.scheme` | Required | `"https"` |
| `http.response.status_code` | Conditional | `200`, `404` |
| `http.route` | Conditional | `"/api/orders/{id}"` |
| `server.address` | Recommended | `"api.example.com"` |
| `server.port` | Conditional | `443` |
| `client.address` | Recommended | `"192.168.1.1"` |
| `network.protocol.version` | Recommended | `"1.1"`, `"2"` |
| `user_agent.original` | Recommended | `"Mozilla/5.0..."` |
| `error.type` | Conditional | `"500"` or `"TimeoutException"` |

## HTTP Client Spans (Stable — v1.23)

| Attribute | Required? | Example |
|---|---|---|
| `http.request.method` | Required | `"GET"` |
| `server.address` | Required | `"api.example.com"` |
| `server.port` | Required | `443` |
| `url.full` | Required | `"https://api.example.com/v1/orders"` |
| `http.response.status_code` | Conditional | `200` |
| `error.type` | Conditional | `"System.Net.Http.HttpRequestException"` |
| `network.protocol.version` | Recommended | `"2"` |
| `http.request.resend_count` | Recommended | `1` (for retries) |

> **Query string redaction:** `AddHttpClientInstrumentation()` and `AddAspNetCoreInstrumentation()` redact query params by default (`?key=Redacted`). Disable with `OTEL_DOTNET_EXPERIMENTAL_HTTPCLIENT_DISABLE_URL_QUERY_REDACTION=true`.

## Database Spans (Experimental, stabilizing)

| Attribute | Required? | Example |
|---|---|---|
| `db.system.name` | Required | `"postgresql"`, `"redis"`, `"mssql"` |
| `db.namespace` | Conditional | `"orders_db"` (database name) |
| `db.operation.name` | Conditional | `"SELECT"`, `"INSERT"`, `"migrate"` |
| `db.query.text` | Recommended (dev only) | `"SELECT * FROM orders WHERE id = $1"` |
| `server.address` | Recommended | `"db.internal.example.com"` |
| `server.port` | Recommended | `5432` |
| `db.response.status_code` | Conditional | error code on failure |
| `error.type` | Conditional | exception type on error |

Opt into stable DB conventions:

```bash
OTEL_SEMCONV_STABILITY_OPT_IN=database        # new conventions only
OTEL_SEMCONV_STABILITY_OPT_IN=database/dup    # both old and new (migration period)
```

Common `db.system.name` values: `"postgresql"`, `"mysql"`, `"mssql"`, `"oracle"`, `"sqlite"`, `"redis"`, `"mongodb"`, `"cassandra"`, `"elasticsearch"`.

## Messaging Spans

| Attribute | Example |
|---|---|
| `messaging.system` | `"rabbitmq"`, `"kafka"`, `"servicebus"` |
| `messaging.destination.name` | `"orders.queue"` |
| `messaging.operation.type` | `"publish"`, `"receive"`, `"process"` |
| `messaging.message.id` | `"msg-abc123"` |

## Resource Attributes

| Attribute | Example | How to set |
|---|---|---|
| `service.name` | `"order-api"` | `AddService()` / `OTEL_SERVICE_NAME` |
| `service.version` | `"1.2.3"` | `AddService(serviceVersion: ...)` |
| `service.namespace` | `"com.acme.payments"` | `AddService(serviceNamespace: ...)` |
| `service.instance.id` | `"pod-abc123"` | `autoGenerateServiceInstanceId: true` |
| `deployment.environment` | `"production"` | `AddAttributes()` |
| `host.name` | `"web-01.prod"` | `AddAttributes()` / env var |
| `cloud.provider` | `"azure"` | `AddAttributes()` |
| `cloud.region` | `"eu-west-1"` | `AddAttributes()` |
| `k8s.pod.name` | `"order-api-7d4b9c-xyz"` | `AddAttributes()` |
| `k8s.namespace.name` | `"production"` | `AddAttributes()` |

## Setting Resource Attributes in Code

```csharp
.ConfigureResource(r => r
    .AddService(
        serviceName:    "order-api",
        serviceVersion: "1.2.3",
        serviceNamespace: "com.acme",
        autoGenerateServiceInstanceId: true)
    .AddAttributes(new Dictionary<string, object>
    {
        ["deployment.environment"] = environment,
        ["host.name"]              = Environment.MachineName,
        ["cloud.region"]           = "eu-west-1",
    })
    .AddEnvironmentVariableDetector()   // reads OTEL_RESOURCE_ATTRIBUTES
    .AddTelemetrySdk())
```

Or fully via environment variables (no code change):

```bash
OTEL_SERVICE_NAME=order-api
OTEL_RESOURCE_ATTRIBUTES=service.version=1.2.3,deployment.environment=production,host.name=web-01
```

## Exception Attributes (on Span Events)

When calling `activity?.AddException(ex)`, the SDK sets:

| Attribute | Value |
|---|---|
| `exception.type` | Fully qualified exception type name |
| `exception.message` | `ex.Message` |
| `exception.stacktrace` | Full stack trace string |
