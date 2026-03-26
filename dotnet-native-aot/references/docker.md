# Docker with Native AOT (.NET 10)

## Key Concept

AOT-published apps are self-contained native binaries. They don't need the .NET runtime, only OS-level dependencies. Use `runtime-deps` images (not `aspnet` or `runtime`).

## Multi-Stage Dockerfile (Ubuntu)

```dockerfile
FROM mcr.microsoft.com/dotnet/sdk:10.0-noble-aot AS build
WORKDIR /src
COPY . .
RUN dotnet publish -c Release -r linux-x64 -o /app/publish

FROM mcr.microsoft.com/dotnet/runtime-deps:10.0-noble-chiseled
WORKDIR /app
COPY --from=build /app/publish .
ENTRYPOINT ["./MyApp"]
```

## Alpine (Smallest Images)

```dockerfile
FROM mcr.microsoft.com/dotnet/sdk:10.0-alpine-aot AS build
WORKDIR /src
COPY . .
RUN dotnet publish -c Release -r linux-musl-x64 -o /app/publish

FROM mcr.microsoft.com/dotnet/runtime-deps:10.0-alpine
WORKDIR /app
COPY --from=build /app/publish .
ENTRYPOINT ["./MyApp"]
```

**Important**: Alpine uses musl libc. Use `-r linux-musl-x64` or `-r linux-musl-arm64`.

## Available SDK AOT Images (.NET 10)

| Image | Base OS |
|-------|---------|
| `sdk:10.0-noble-aot` | Ubuntu 24.04 |
| `sdk:10.0-alpine-aot` | Alpine |
| `sdk:10.0-azurelinux3.0-aot` | Azure Linux 3.0 |
| `sdk:10.0-trixie-aot` | Debian Trixie |

The `-aot` SDK images include `clang`, `llvm`, and `zlib1g-dev` needed for AOT compilation. Regular SDK images lack these.

## Image Size Comparison

| Configuration | Approximate Size |
|--------------|-----------------|
| JIT + aspnet:10.0 runtime | ~200-216 MB |
| AOT + Ubuntu runtime-deps | 80-90 MB |
| AOT + Ubuntu chiseled | Smaller than standard |
| AOT + Alpine runtime-deps | ~18 MB (barebone ASP.NET Core) |

## Using Non-AOT SDK Image

If you can't use the `-aot` SDK image, install prerequisites manually:

```dockerfile
FROM mcr.microsoft.com/dotnet/sdk:10.0 AS build
RUN apt-get update && apt-get install -y clang zlib1g-dev
WORKDIR /src
COPY . .
RUN dotnet publish -c Release -r linux-x64 -o /app/publish
```

## Runtime-deps Note

Microsoft does NOT publish separate `-aot` runtime-deps tags for .NET 10. The standard `runtime-deps` images work for both JIT and AOT. The only difference is that CoreCLR needs `libstdc++` while Native AOT does not (~1MB).

## Multi-Architecture Builds

```dockerfile
# Build for current platform
FROM --platform=$BUILDPLATFORM mcr.microsoft.com/dotnet/sdk:10.0-noble-aot AS build
ARG TARGETARCH
WORKDIR /src
COPY . .
RUN dotnet publish -c Release -a $TARGETARCH -o /app/publish

FROM mcr.microsoft.com/dotnet/runtime-deps:10.0-noble-chiseled
WORKDIR /app
COPY --from=build /app/publish .
ENTRYPOINT ["./MyApp"]
```

```bash
docker buildx build --platform linux/amd64,linux/arm64 -t myapp:latest .
```

## Volumes and Paths

AOT doesn't change how volumes work. Same patterns:

```dockerfile
VOLUME /app/data
VOLUME /downloads
ENV DownloadPath=/downloads
```
