# 🤖 Agent Skills by eduvhc

A curated collection of AI agent skills for .NET development, published to [skills.sh](https://skills.sh).

[![skills.sh](https://img.shields.io/badge/skills.sh-published-blue?style=flat-square)](https://skills.sh)
[![.NET](https://img.shields.io/badge/.NET-10-purple?style=flat-square&logo=dotnet)](https://dotnet.microsoft.com/)
[![Bun](https://img.shields.io/badge/Bun-supported-f9f1e1?style=flat-square&logo=bun)](https://bun.sh/)

---

## 📦 Available Skills

> **✅ Current Status**: All skills are up to date with **.NET 10** (latest stable) and **OpenTelemetry 1.15.0** patterns as of March 2026.

### 🔷 dotnet-aspire
**Cloud-native .NET development with Aspire**

Complete guidance for building distributed applications with .NET Aspire 13 / .NET 10:

- **AppHost orchestration** - Resource coordination, container lifecycle, startup ordering
- **ServiceDefaults** - OpenTelemetry, health checks, resilience patterns
- **Integration testing** - `DistributedApplicationTestingBuilder`, test fixtures, worker testing
- **Observability** - OTel configuration, dashboard integration, production monitoring

```bash
# Using npm/npx
npx skills add eduvhc/agent-skills --skill dotnet-aspire

# Using Bun
bunx skills add eduvhc/agent-skills --skill dotnet-aspire
```

---

### 📊 opentelemetry-dotnet
**OpenTelemetry for .NET 10**

Comprehensive observability implementation with **OpenTelemetry 1.15.0** covering all three signals:

- **Tracing** - `ActivitySource`, spans, semantic conventions, context propagation
- **Metrics** - `IMeterFactory`, instruments, exemplars, cardinality management
- **Logging** - `ILogger` bridge, log correlation, level filtering
- **Exporters** - OTLP, Prometheus, Console, In-Memory with batch tuning
- **Auto-instrumentation** - CLR profiler, Docker, Kubernetes patterns

```bash
# Using npm/npx
npx skills add eduvhc/agent-skills --skill opentelemetry-dotnet

# Using Bun
bunx skills add eduvhc/agent-skills --skill opentelemetry-dotnet
```

---

### 🔵 azure-devops-dotnet
**Azure DevOps YAML pipelines for .NET & NuGet**

Complete pipeline authoring for .NET 8/9/10 projects and NuGet packages:

- **DotNetCoreCLI@2** - restore, build, test, publish, pack, push
- **NuGetAuthenticate@1** - same-org and cross-org feed authentication
- **NuGet caching** - `Cache@2` with lock files, typical 5× speedup
- **Versioning strategies** - GitVersion, byBuildNumber, byEnvVar, byPrereleaseNumber
- **Multi-stage pipelines** - build → publish packages → staging → production
- **Deployment strategies** - `runOnce`, `rolling`, `canary`
- **2025/2026 gotchas** - ubuntu-24.04 NuGetCommand@2 breakage, deployment job variables

```bash
# Using npm/npx
npx skills add eduvhc/agent-skills --skill azure-devops-dotnet

# Using Bun
bunx skills add eduvhc/agent-skills --skill azure-devops-dotnet
```

---

## 🚀 Quick Start

### Install All Skills

```bash
# npm/npx
npx skills add eduvhc/agent-skills

# Bun
bunx skills add eduvhc/agent-skills
```

### Install Specific Skills

```bash
# dotnet-aspire
npx skills add eduvhc/agent-skills --skill dotnet-aspire

# opentelemetry-dotnet
npx skills add eduvhc/agent-skills --skill opentelemetry-dotnet

# azure-devops-dotnet
npx skills add eduvhc/agent-skills --skill azure-devops-dotnet
```

### Install to Specific Agents

```bash
# npm/npx
npx skills add eduvhc/agent-skills --agent claude-code --agent opencode

# Bun
bunx skills add eduvhc/agent-skills --agent claude-code --agent opencode
```

---

## 📚 Documentation

Each skill includes comprehensive reference documentation:

| Skill | References |
|-------|------------|
| `dotnet-aspire` | `apphost.md`, `service-defaults.md`, `testing.md`, `opentelemetry.md`, `health-checks.md` |
| `opentelemetry-dotnet` | `tracing.md`, `metrics.md`, `logging.md`, `exporters.md`, `sampling.md`, `propagation.md`, `instrumentation.md`, `hybrid.md`, `semantic-conventions.md`, `production.md`, `pitfalls.md`, `packages.md`, `setup.md` |
| `azure-devops-dotnet` | Inline — all content in `SKILL.md` |

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

### When to use `azure-devops-dotnet`:
- Building or publishing .NET projects in Azure DevOps pipelines
- Authenticating to Azure Artifacts feeds (same-org or cross-org)
- Setting up NuGet package caching and versioning strategies
- Authoring multi-stage pipelines with deployment jobs and approval gates
- Troubleshooting broken restore on ubuntu-latest (24.04)

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
