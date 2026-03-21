# azure-devops-dotnet

Azure DevOps YAML pipeline skill for .NET 8/9/10 projects and NuGet packages.

## What this skill covers

- `DotNetCoreCLI@2` — all commands: restore, build, test, publish, pack, push
- `NuGetAuthenticate@1` — same-org and cross-org feed authentication
- `UseDotNet@2` — SDK installation
- NuGet package caching with `Cache@2` and lock files
- Multi-stage pipelines (build → publish packages → staging → production)
- Deployment job strategies: `runOnce`, `rolling`, `canary`
- NuGet versioning: GitVersion, byBuildNumber, byEnvVar, byPrereleaseNumber, bySemVerBuildNumber
- Azure Artifacts feed URLs, permissions, upstream sources
- Code coverage: Coverlet / XPlat + ReportGenerator
- Templates (step / job / stage / cross-repo / extends)
- Variable groups, secrets, Key Vault integration
- Docker multi-stage builds for .NET
- 2025/2026 gotchas (ubuntu-24.04 NuGetCommand@2 breakage, deployment job workspace variables, etc.)

## Activation

This skill activates automatically when you ask about:
- Building .NET projects in Azure DevOps
- NuGet package publishing or restore in pipelines
- DotNetCoreCLI, NuGetAuthenticate, Azure Artifacts
- CI/CD YAML for C#/.NET microservices
- dotnet pack, push, versioning strategies
