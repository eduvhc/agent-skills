# 🤖 Agent Skills by eduvhc

A curated collection of AI agent skills for .NET development, published to [skills.sh](https://skills.sh).

[![skills.sh](https://img.shields.io/badge/skills.sh-published-blue?style=flat-square)](https://skills.sh)
[![.NET](https://img.shields.io/badge/.NET-10-purple?style=flat-square&logo=dotnet)](https://dotnet.microsoft.com/)

---

## 📦 Available Skills

### 🔷 dotnet-aspire
**Cloud-native .NET development with Aspire**

Complete guidance for building distributed applications with .NET Aspire 13 / .NET 10:

- **AppHost orchestration** - Resource coordination, container lifecycle, startup ordering
- **ServiceDefaults** - OpenTelemetry, health checks, resilience patterns
- **Integration testing** - `DistributedApplicationTestingBuilder`, test fixtures, worker testing
- **Observability** - OTel configuration, dashboard integration, production monitoring

```bash
npx skills add eduvhc/agent-skills --skill dotnet-aspire
```

---

### 📊 opentelemetry-dotnet
**OpenTelemetry for .NET 10**

Comprehensive observability implementation covering all three signals:

- **Tracing** - `ActivitySource`, spans, semantic conventions, context propagation
- **Metrics** - `IMeterFactory`, instruments, exemplars, cardinality management
- **Logging** - `ILogger` bridge, log correlation, level filtering
- **Exporters** - OTLP, Prometheus, Console, In-Memory with batch tuning
- **Auto-instrumentation** - CLR profiler, Docker, Kubernetes patterns

```bash
npx skills add eduvhc/agent-skills --skill opentelemetry-dotnet
```

---

## 🚀 Quick Start

### Install All Skills

```bash
npx skills add eduvhc/agent-skills
```

### Install Specific Skills

```bash
# Just the Aspire skill
npx skills add eduvhc/agent-skills --skill dotnet-aspire

# Just the OpenTelemetry skill
npx skills add eduvhc/agent-skills --skill opentelemetry-dotnet
```

### Install to Specific Agents

```bash
npx skills add eduvhc/agent-skills --agent claude-code --agent opencode
```

---

## 📚 Documentation

Each skill includes comprehensive reference documentation:

| Skill | References |
|-------|------------|
| `dotnet-aspire` | `apphost.md`, `service-defaults.md`, `testing.md`, `opentelemetry.md`, `health-checks.md` |
| `opentelemetry-dotnet` | `tracing.md`, `metrics.md`, `logging.md`, `exporters.md`, `sampling.md`, `instrumentation.md`, `production.md`, `pitfalls.md` |

---

## 🎯 Use Cases

### When to use `dotnet-aspire`:
- Setting up a new .NET Aspire solution with AppHost and ServiceDefaults
- Configuring PostgreSQL, Redis, RabbitMQ, or SQL Server resources
- Writing integration tests against real containers
- Debugging orchestration issues (port conflicts, health check hangs)

### When to use `opentelemetry-dotnet`:
- Instrumenting ASP.NET Core APIs and HTTP clients
- Setting up metrics collection with proper cardinality
- Configuring OTLP exporters for Aspire Dashboard, Jaeger, or Grafana
- Implementing hybrid auto-instrumentation + manual spans
- Troubleshooting missing telemetry or high memory usage

---

## 🛠️ Requirements

- [.NET 10 SDK](https://dotnet.microsoft.com/download/dotnet/10.0)
- AI agent that supports skills (Claude Code, OpenCode, Cursor, etc.)

---

## 📄 License

MIT - Feel free to use these skills in your projects!

---

## 🤝 Contributing

These skills are maintained by [eduvhc](https://github.com/eduvhc). For issues or suggestions, please open an issue on this repository.

---

<p align="center">
  <sub>Built with ❤️ for the .NET community</sub>
</p>
