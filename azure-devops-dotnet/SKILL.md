---
name: azure-devops-dotnet
description: This skill should be used when the user asks about Azure DevOps pipelines for .NET, CI/CD for .NET Core, .NET 8, .NET 9, .NET 10, building or publishing NuGet packages, Azure Artifacts feeds, DotNetCoreCLI@2, NuGetAuthenticate, pipeline YAML for C# projects, multi-stage pipelines for .NET, dotnet restore/build/test/pack/push in pipelines, NuGet caching, versioning strategies for packages (GitVersion, semantic versioning), or deploying .NET applications via Azure DevOps.
version: 1.0.0
---

# Azure DevOps Pipelines for .NET & NuGet

## Overview

This skill covers building, testing, packaging, and deploying .NET 8/9/10 projects using Azure DevOps YAML pipelines and Azure Artifacts. All examples use current task versions as of 2026.

**Quick rule of thumb:**
- Use `DotNetCoreCLI@2` for all `dotnet` CLI operations (restore/build/test/pack/push/publish)
- Use `NuGetAuthenticate@1` before any NuGet operation on Ubuntu 24.04 or when crossing org boundaries
- Avoid `NuGetCommand@2 restore` on `ubuntu-latest` (Ubuntu 24.04) — it is broken there

---

## 1. SDK Installation — UseDotNet@2

Always install the SDK explicitly unless the agent image already has the version you need (ubuntu-latest has .NET 8 and 9 preinstalled as of 2026):

```yaml
- task: UseDotNet@2
  displayName: Install .NET 8 SDK
  inputs:
    packageType: sdk
    version: '8.x'           # or '9.x', '10.x'
    includePreviewVersions: false
    performMultiLevelLookup: true   # required on Windows agents
```

Agent pool .NET preinstall matrix (early 2026):
| Pool | .NET versions preinstalled |
|------|---------------------------|
| ubuntu-latest (24.04) | .NET 8, .NET 9 |
| windows-latest (Server 2022) | .NET 8 |
| macos-latest (macOS 14) | .NET 8 |

---

## 2. NuGet Authentication — NuGetAuthenticate@1

Run **before** any restore/push/pack step. Sets credentials for the entire job.

```yaml
# Same-org Azure Artifacts feed — no inputs needed
- task: NuGetAuthenticate@1

# Cross-org or third-party feed — provide service connection names
- task: NuGetAuthenticate@1
  inputs:
    nuGetServiceConnections: 'OtherOrgConn, ThirdPartyConn'
    forceReinstallCredentialProvider: false
```

**Why NuGetAuthenticate instead of inline credentials in DotNetCoreCLI@2?**
- `NuGetAuthenticate@1` sets `VSS_NUGET_URI_PREFIXES` / `VSS_NUGET_ACCESSTOKEN` env vars — valid for the whole job.
- Required on `ubuntu-latest` when using `dotnet` CLI scripts directly.
- `DotNetCoreCLI@2` with `vstsFeed` only authenticates within that single task.

---

## 3. DotNetCoreCLI@2 — All Commands

### Restore

```yaml
# From Azure Artifacts feed (dropdown selection)
- task: DotNetCoreCLI@2
  displayName: Restore
  inputs:
    command: restore
    projects: '**/*.csproj'
    feedsToUse: select
    vstsFeed: 'MyProject/MyFeed'   # [project/]feedName, or just feedName for org-scoped
    includeNuGetOrg: true
    verbosityRestore: Normal
    requestTimeout: '300000'       # ms; increase for large repos (default = 5 min)

# From nuget.config (external feeds / multi-feed)
- task: DotNetCoreCLI@2
  displayName: Restore
  inputs:
    command: restore
    projects: '**/*.csproj'
    feedsToUse: config
    nugetConfigPath: NuGet.config
    externalFeedCredentials: 'MyNuGetServiceConnection'
```

### Build

```yaml
- task: DotNetCoreCLI@2
  displayName: Build
  inputs:
    command: build
    projects: '**/*.csproj'
    arguments: '--configuration $(buildConfiguration) --no-restore'
```

### Test

```yaml
# With XPlat Code Coverage (cross-platform, Coverlet)
- task: DotNetCoreCLI@2
  displayName: Test
  inputs:
    command: test
    projects: '**/*Tests/*.csproj'
    arguments: >-
      --configuration $(buildConfiguration)
      --no-build
      --collect:"XPlat Code Coverage"
      --results-directory $(Agent.TempDirectory)
    publishTestResults: true        # auto-uploads .trx files — no separate task needed
    testRunTitle: 'Unit Tests - $(Build.BuildNumber)'

# Windows built-in collector (produces .coverage files)
- task: DotNetCoreCLI@2
  displayName: Test
  inputs:
    command: test
    projects: '**/*Tests/*.csproj'
    arguments: '--configuration $(buildConfiguration) --no-build --collect "Code Coverage"'
    publishTestResults: true
```

### Publish

```yaml
# Web project (auto-detected)
- task: DotNetCoreCLI@2
  displayName: Publish
  inputs:
    command: publish
    publishWebProjects: true
    arguments: '--configuration $(buildConfiguration) --output $(Build.ArtifactStagingDirectory)'
    zipAfterPublish: true
    modifyOutputPath: false     # false = flat output; true = project-named subdirectory

# Non-web project (explicit path)
- task: DotNetCoreCLI@2
  displayName: Publish API
  inputs:
    command: publish
    publishWebProjects: false
    projects: 'src/MyApi/MyApi.csproj'
    arguments: '--configuration $(buildConfiguration) --no-build --output $(Build.ArtifactStagingDirectory)'
    zipAfterPublish: true
```

### Pack

```yaml
- task: DotNetCoreCLI@2
  displayName: Pack
  inputs:
    command: pack
    packagesToPack: 'src/**/*.csproj'
    configuration: '$(buildConfiguration)'
    packDirectory: '$(Build.ArtifactStagingDirectory)'
    nobuild: true               # skip build if already built — avoids double build
    includesymbols: true        # generates .snupkg
    includesource: false
    versioningScheme: byEnvVar  # off | byPrereleaseNumber | byEnvVar | byBuildNumber | bySemVerBuildNumber
    versionEnvVar: PACKAGE_VERSION
    buildProperties: 'RepositoryUrl=$(Build.Repository.Uri)'
    verbosityPack: Normal
```

### Push

```yaml
# Internal Azure Artifacts feed
- task: DotNetCoreCLI@2
  displayName: Push to Azure Artifacts
  inputs:
    command: push
    packagesToPush: '$(Build.ArtifactStagingDirectory)/*.nupkg'
    nuGetFeedType: internal
    publishVstsFeed: 'MyProject/MyFeed'
    publishPackageMetadata: true    # links package to pipeline run in feed UI

# External feed (e.g., NuGet.org)
- task: DotNetCoreCLI@2
  displayName: Push to NuGet.org
  inputs:
    command: push
    packagesToPush: '$(Build.ArtifactStagingDirectory)/*.nupkg'
    nuGetFeedType: external
    publishFeedCredentials: 'NuGetOrgServiceConnection'

# Alternative: dotnet CLI push (after NuGetAuthenticate)
- task: NuGetAuthenticate@1
- script: |
    dotnet nuget push \
      $(Build.ArtifactStagingDirectory)/*.nupkg \
      --source "https://pkgs.dev.azure.com/$(System.TeamFoundationCollectionUri)/_packaging/$(feedName)/nuget/v3/index.json" \
      --api-key AzureArtifacts \
      --skip-duplicate
  displayName: Push packages
```

---

## 4. NuGet Package Caching

Requires lock files committed to source control.

**Step 1 — Add to each `.csproj`:**
```xml
<PropertyGroup>
  <RestorePackagesWithLockFile>true</RestorePackagesWithLockFile>
</PropertyGroup>
```

**Step 2 — Generate locally:**
```
dotnet restore --use-lock-file
git add packages.lock.json
```

**Step 3 — Pipeline:**
```yaml
variables:
  NUGET_PACKAGES: $(Pipeline.Workspace)/.nuget/packages

steps:
- task: Cache@2
  displayName: Cache NuGet packages
  inputs:
    key: 'nuget | "$(Agent.OS)" | **/packages.lock.json,!**/bin/**,!**/obj/**'
    restoreKeys: |
      nuget | "$(Agent.OS)"
      nuget
    path: $(NUGET_PACKAGES)
    cacheHitVar: CACHE_RESTORED

- task: DotNetCoreCLI@2
  displayName: Restore
  condition: ne(variables.CACHE_RESTORED, true)   # skip on cache hit
  inputs:
    command: restore
    projects: '**/*.csproj'
    feedsToUse: select
    vstsFeed: 'MyFeed'
```

Cache is immutable once written. `restoreKeys` enables partial hits across branches. Typical speedup: ~41s → ~8s.

---

## 5. NuGet Versioning Strategies

### GitVersion (recommended for libraries)

```yaml
- task: DotNetCoreCLI@2
  displayName: Install GitVersion
  inputs:
    command: custom
    custom: tool
    arguments: 'install --global GitVersion.Tool --version 5.*'

- script: dotnet-gitversion /output buildserver
  displayName: Run GitVersion
  # Sets pipeline variables: GitVersion.NuGetVersionV2, GitVersion.SemVer, etc.

- task: DotNetCoreCLI@2
  displayName: Pack
  inputs:
    command: pack
    versioningScheme: byEnvVar
    versionEnvVar: GitVersion.NuGetVersionV2
```

### Build number versioning

```yaml
name: $(Major).$(Minor).$(Rev:r)   # pipeline-level build number format

variables:
  Major: '1'
  Minor: '0'

steps:
- task: DotNetCoreCLI@2
  inputs:
    command: pack
    versioningScheme: byBuildNumber   # uses $(Build.BuildNumber)
```

### Pre-release versioning

```yaml
- task: DotNetCoreCLI@2
  inputs:
    command: pack
    versioningScheme: byPrereleaseNumber
    majorVersion: '2'
    minorVersion: '1'
    patchVersion: '0'
    # Produces: 2.1.0-CI-YYYYMMDD-HHMMSS
```

### SemVer build number (cloud only — not Server 2022)

```yaml
- task: DotNetCoreCLI@2
  inputs:
    command: pack
    versioningScheme: bySemVerBuildNumber
```

### MSBuild properties (full control)

```yaml
- task: DotNetCoreCLI@2
  inputs:
    command: pack
    versioningScheme: off
    buildProperties: 'Version=$(PACKAGE_VERSION);InformationalVersion=$(Build.SourceVersion)'
```

---

## 6. Code Coverage

```yaml
# Coverlet (XPlat) + ReportGenerator
- task: DotNetCoreCLI@2
  displayName: Test with coverage
  inputs:
    command: test
    projects: '**/*Tests/*.csproj'
    arguments: >-
      --configuration $(buildConfiguration)
      --no-build
      --collect:"XPlat Code Coverage"
      --results-directory $(Agent.TempDirectory)
    publishTestResults: true

- script: dotnet tool install --global dotnet-reportgenerator-globaltool
  displayName: Install ReportGenerator

- script: |
    reportgenerator \
      "-reports:$(Agent.TempDirectory)/**/coverage.cobertura.xml" \
      "-targetdir:$(Build.ArtifactStagingDirectory)/coverage" \
      "-reporttypes:HtmlInline_AzurePipelines;Cobertura"
  displayName: Generate coverage report

- task: PublishCodeCoverageResults@2
  displayName: Publish coverage
  inputs:
    codeCoverageTool: Cobertura
    summaryFileLocation: '$(Build.ArtifactStagingDirectory)/coverage/Cobertura.xml'
    reportDirectory: '$(Build.ArtifactStagingDirectory)/coverage'
```

---

## 7. Multi-Stage Pipeline Skeleton

```yaml
trigger:
  branches:
    include: [main, 'release/*']
  paths:
    exclude: ['**/*.md']

pr:
  branches:
    include: [main]

name: '$(Build.SourceBranchName)_$(Rev:r)'

variables:
- group: my-variable-group
- name: buildConfiguration
  value: Release
- name: NUGET_PACKAGES
  value: $(Pipeline.Workspace)/.nuget/packages

stages:
# ── Build & Test ──────────────────────────────
- stage: CI
  displayName: Build & Test
  jobs:
  - job: Build
    pool:
      vmImage: ubuntu-latest
    steps:
    - task: UseDotNet@2
      inputs:
        version: '8.x'
    - task: NuGetAuthenticate@1
    - task: Cache@2
      inputs:
        key: 'nuget | "$(Agent.OS)" | **/packages.lock.json,!**/bin/**,!**/obj/**'
        path: $(NUGET_PACKAGES)
        cacheHitVar: CACHE_RESTORED
    - task: DotNetCoreCLI@2
      displayName: Restore
      condition: ne(variables.CACHE_RESTORED, true)
      inputs:
        command: restore
        projects: '**/*.csproj'
        feedsToUse: select
        vstsFeed: 'MyProject/MyFeed'
    - task: DotNetCoreCLI@2
      displayName: Build
      inputs:
        command: build
        projects: '**/*.csproj'
        arguments: '--configuration $(buildConfiguration) --no-restore'
    - task: DotNetCoreCLI@2
      displayName: Test
      inputs:
        command: test
        projects: '**/*Tests/*.csproj'
        arguments: '--configuration $(buildConfiguration) --no-build --collect:"XPlat Code Coverage"'
        publishTestResults: true
    - task: PublishCodeCoverageResults@2
      condition: succeededOrFailed()
      inputs:
        codeCoverageTool: Cobertura
        summaryFileLocation: '$(Agent.TempDirectory)/**/coverage.cobertura.xml'
    - task: DotNetCoreCLI@2
      displayName: Pack
      condition: and(succeeded(), ne(variables['Build.Reason'], 'PullRequest'))
      inputs:
        command: pack
        packagesToPack: 'src/MyLib/*.csproj'
        configuration: $(buildConfiguration)
        packDirectory: '$(Build.ArtifactStagingDirectory)/packages'
        nobuild: true
        includesymbols: true
        versioningScheme: byEnvVar
        versionEnvVar: PACKAGE_VERSION
    - task: DotNetCoreCLI@2
      displayName: Publish
      inputs:
        command: publish
        publishWebProjects: false
        projects: 'src/MyApi/MyApi.csproj'
        arguments: '--configuration $(buildConfiguration) --no-build --output $(Build.ArtifactStagingDirectory)/api'
        zipAfterPublish: true
    - task: PublishPipelineArtifact@1
      displayName: Upload artifacts
      inputs:
        targetPath: '$(Build.ArtifactStagingDirectory)'
        artifactName: drop

# ── Publish NuGet packages ─────────────────────
- stage: PublishPackages
  displayName: Publish NuGet
  dependsOn: CI
  condition: and(succeeded(), eq(variables['Build.SourceBranch'], 'refs/heads/main'))
  jobs:
  - job: Publish
    pool:
      vmImage: ubuntu-latest
    steps:
    - task: DownloadPipelineArtifact@2
      inputs:
        artifactName: drop
        targetPath: '$(Pipeline.Workspace)/drop'
    - task: NuGetAuthenticate@1
    - task: DotNetCoreCLI@2
      inputs:
        command: push
        packagesToPush: '$(Pipeline.Workspace)/drop/packages/*.nupkg'
        nuGetFeedType: internal
        publishVstsFeed: 'MyProject/MyFeed'
        publishPackageMetadata: true

# ── Deploy to staging ──────────────────────────
- stage: DeployStaging
  displayName: Deploy to Staging
  dependsOn: CI
  condition: and(succeeded(), eq(variables['Build.SourceBranch'], 'refs/heads/main'))
  jobs:
  - deployment: DeployStaging
    pool:
      vmImage: ubuntu-latest
    environment: staging     # approval gates set in DevOps UI on this environment
    strategy:
      runOnce:
        deploy:
          steps:
          - task: DownloadPipelineArtifact@2
            inputs:
              artifactName: drop
              targetPath: '$(Pipeline.Workspace)/drop'
          - script: echo deploy to staging

# ── Deploy to production ───────────────────────
- stage: DeployProduction
  displayName: Deploy to Production
  dependsOn: DeployStaging
  condition: succeeded()
  trigger: manual          # explicit manual trigger required
  jobs:
  - deployment: DeployProd
    pool:
      vmImage: ubuntu-latest
    environment: production
    strategy:
      runOnce:
        deploy:
          steps:
          - task: DownloadPipelineArtifact@2
            inputs:
              artifactName: drop
              targetPath: '$(Pipeline.Workspace)/drop'
          - script: echo deploy to production
```

---

## 8. Stage Dependency Patterns

```yaml
# Sequential (default)
stages:
- stage: Build
- stage: Test
  dependsOn: Build
- stage: Deploy
  dependsOn: Test

# Parallel stages
stages:
- stage: UnitTests
- stage: IntegrationTests
  dependsOn: []             # empty array = parallel

# Fan-out / fan-in
- stage: Build
- stage: DeployUS
  dependsOn: Build
- stage: DeployEU
  dependsOn: Build
- stage: SmokeTest
  dependsOn: [DeployUS, DeployEU]

# Branch-gated
- stage: Release
  condition: and(succeeded(), eq(variables['Build.SourceBranch'], 'refs/heads/main'))

# Manual stage
- stage: Production
  trigger: manual

# Non-skippable (enforce security/compliance)
- stage: SecurityScan
  isSkippable: false
```

---

## 9. Deployment Jobs — Strategies

### runOnce (most common)

```yaml
- deployment: DeployWeb
  environment: production
  strategy:
    runOnce:
      preDeploy:
        steps:
        - script: echo pre-deploy
      deploy:
        steps:
        - checkout: self          # deployment jobs do NOT auto-checkout — add explicitly
        - download: current
          artifact: drop
        - script: echo deploy
      on:
        failure:
          steps:
          - script: echo rollback
        success:
          steps:
          - script: echo notify
```

### Rolling (VMs)

```yaml
- deployment: VMDeploy
  environment:
    name: production
    resourceType: VirtualMachine
  strategy:
    rolling:
      maxParallel: 2
      deploy:
        steps:
        - script: echo deploy
```

### Canary (Kubernetes)

```yaml
- deployment: CanaryDeploy
  environment: 'production.bookings'
  strategy:
    canary:
      increments: [10, 25, 50]
      deploy:
        steps:
        - task: KubernetesManifest@1
          inputs:
            action: $(strategy.action)
            strategy: $(strategy.name)
            percentage: $(strategy.increment)
            manifests: manifests/*.yaml
```

---

## 10. Templates

### Step template

```yaml
# templates/dotnet-build.yml
parameters:
- name: buildConfiguration
  type: string
  default: Release
- name: projects
  type: string
  default: '**/*.csproj'

steps:
- task: UseDotNet@2
  inputs:
    version: '8.x'
- task: DotNetCoreCLI@2
  inputs:
    command: restore
    projects: ${{ parameters.projects }}
    feedsToUse: select
    vstsFeed: 'MyFeed'
- task: DotNetCoreCLI@2
  inputs:
    command: build
    projects: ${{ parameters.projects }}
    arguments: '--configuration ${{ parameters.buildConfiguration }} --no-restore'
- task: DotNetCoreCLI@2
  inputs:
    command: test
    projects: '**/*Tests/*.csproj'
    arguments: '--configuration ${{ parameters.buildConfiguration }} --no-build'
    publishTestResults: true
```

Consume:
```yaml
steps:
- template: templates/dotnet-build.yml
  parameters:
    buildConfiguration: Release
```

### Template from another repository

```yaml
resources:
  repositories:
  - repository: templates
    type: git
    name: MyProject/PipelineTemplates
    ref: refs/tags/v2.1.0       # pin to tag

stages:
- template: dotnet/build.yml@templates
  parameters:
    buildConfiguration: Release
```

### Extends (security enforcement)

```yaml
# Pipeline extends a mandatory template — required template check enforces this
extends:
  template: templates/secure-base.yml
  parameters:
    buildSteps:
    - task: DotNetCoreCLI@2
      inputs:
        command: build
```

---

## 11. Variables and Secrets

```yaml
# Inline + variable group
variables:
- group: my-variable-group
- name: buildConfiguration
  value: Release

# Stage/job scoped
stages:
- stage: Deploy
  variables:
  - group: prod-secrets
  jobs:
  - job: Deploy
    variables:
    - group: deploy-config
    steps:
    # Secret variables MUST be passed via env: — not accessible as $(var) in scripts
    - script: echo "Conn: $MY_CONN"
      env:
        MY_CONN: $(connectionString)
```

Azure Key Vault integration: link secrets to a variable group in the Library UI, then use `- group: my-keyvault-group`.

---

## 12. Artifact Publishing and Downloading

```yaml
# Upload artifact (build stage)
- task: PublishPipelineArtifact@1
  inputs:
    targetPath: '$(Build.ArtifactStagingDirectory)'
    artifactName: drop

# Download artifact (deploy stage)
- task: DownloadPipelineArtifact@2
  inputs:
    artifactName: drop
    targetPath: '$(Pipeline.Workspace)/drop'   # use Pipeline.Workspace in deployment jobs

# Cross-pipeline download
- task: DownloadPipelineArtifact@2
  inputs:
    buildType: specific
    project: 'MyProject'
    definition: 42
    buildVersionToDownload: latest
    artifactName: drop
    targetPath: '$(Pipeline.Workspace)/drop'
```

---

## 13. Azure Artifacts Feed URLs

| Feed type | URL |
|-----------|-----|
| Org-scoped | `https://pkgs.dev.azure.com/{org}/_packaging/{feed}/nuget/v3/index.json` |
| Project-scoped | `https://pkgs.dev.azure.com/{org}/{project}/_packaging/{feed}/nuget/v3/index.json` |
| Feed view (snapshot) | `https://pkgs.dev.azure.com/{org}/{project}/_packaging/{feed}@{view}/nuget/v3/index.json` |

### nuget.config for multiple feeds

```xml
<?xml version="1.0" encoding="utf-8"?>
<configuration>
  <packageSources>
    <clear />
    <add key="MyOrgFeed"
         value="https://pkgs.dev.azure.com/{org}/_packaging/{feed}/nuget/v3/index.json" />
    <add key="nuget.org"
         value="https://api.nuget.org/v3/index.json" />
  </packageSources>
</configuration>
```

### Feed permissions for pipelines

- **Same project**: build service gets read access automatically.
- **Different project**: grant `{Project} Build Service` the **Feed Reader** or **Contributor** role in Artifacts > Feed Settings > Permissions.
- `publishPackageMetadata: true` creates provenance links between package versions and pipeline runs.

---

## 14. Docker builds for .NET

### Dockerfile (multi-stage, production)

```dockerfile
FROM mcr.microsoft.com/dotnet/sdk:8.0 AS build
WORKDIR /src

# Copy .csproj first — enables Docker layer cache for restore
COPY ["src/MyApi/MyApi.csproj", "src/MyApi/"]
COPY ["src/MyLib/MyLib.csproj", "src/MyLib/"]
RUN dotnet restore "src/MyApi/MyApi.csproj"

COPY . .
WORKDIR "/src/src/MyApi"
RUN dotnet build "MyApi.csproj" -c Release -o /app/build

FROM build AS publish
RUN dotnet publish "MyApi.csproj" -c Release -o /app/publish \
    /p:UseAppHost=false \
    /p:PublishReadyToRun=true

FROM mcr.microsoft.com/dotnet/aspnet:8.0 AS final
WORKDIR /app
EXPOSE 8080
USER app                         # run as non-root
COPY --from=publish /app/publish .
ENTRYPOINT ["dotnet", "MyApi.dll"]
```

### Docker task in pipeline

```yaml
- task: Docker@2
  displayName: Build and push
  inputs:
    containerRegistry: 'MyACRServiceConnection'
    repository: 'myapi'
    command: buildAndPush
    dockerfile: 'src/MyApi/Dockerfile'
    buildContext: '$(Build.Repository.LocalPath)'
    tags: |
      $(Build.BuildNumber)
      latest
```

---

## 15. Critical Gotchas (2025/2026)

| # | Issue | Fix |
|---|-------|-----|
| 1 | `NuGetCommand@2 restore` broken on ubuntu-latest (24.04) | Use `NuGetAuthenticate@1` + `DotNetCoreCLI@2 restore` |
| 2 | `bySemVerBuildNumber` not available in DevOps Server 2022 | Use `byBuildNumber` or `byEnvVar` instead |
| 3 | Deployment jobs don't auto-checkout source | Add `- checkout: self` explicitly if needed |
| 4 | `$(Build.ArtifactStagingDirectory)` not set in deployment jobs | Use `$(Pipeline.Workspace)` |
| 5 | Fork PR builds fail to push to internal feeds (HTTP 500) | Expected — fork PRs have no internal feed credentials |
| 6 | Secret variables inaccessible directly in script steps | Pass via `env:` mapping |
| 7 | Cache task not supported in Classic release pipelines | YAML pipelines only |
| 8 | `publishTestResults: true` + `PublishTestResults@2` = duplicate results | Pick one approach |
| 9 | `nobuild: true` on pack skips redundant build | Always set when you've already run `dotnet build` |
| 10 | Large monorepo restore timeouts | Set `requestTimeout: '600000'` (10 min) on restore task |
| 11 | Template file limit | Max 100 included files, max 100 nesting levels |

---

## 16. NuGet.exe vs dotnet CLI Decision Table

| Scenario | Use |
|----------|-----|
| .NET 6/8/9/10 SDK-style projects | `DotNetCoreCLI@2` |
| Legacy .NET Framework + `.csproj` | `DotNetCoreCLI@2 restore` |
| Legacy .NET Framework + `packages.config` | `NuGetCommand@2 restore` |
| Ubuntu 24.04 restore | `NuGetAuthenticate@1` + `DotNetCoreCLI@2` |
| Push to NuGet.org with API key | `dotnet nuget push --api-key $(myKey)` (NuGetAuthenticate cannot handle API keys for NuGet.org) |
