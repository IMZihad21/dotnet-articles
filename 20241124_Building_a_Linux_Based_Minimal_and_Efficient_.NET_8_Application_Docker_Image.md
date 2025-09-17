In this article, we will walk through a multi-stage Dockerfile optimized for building and running a .NET 8 application targeted for Linux. This setup emphasizes performance, size, and compatibility, particularly for applications running in Linux-based environments.

---

## Dockerfile Breakdown

### Stage 1: Build Stage

```dockerfile
FROM mcr.microsoft.com/dotnet/sdk:8.0-alpine AS build
ARG BUILD_CONFIGURATION=Release
ARG RUNTIME=linux-musl-x64
WORKDIR /src
```

We start with the official `.NET 8 SDK` image (`mcr.microsoft.com/dotnet/sdk:8.0-alpine`) for the build stage. The `alpine` variant is chosen for its small size and compatibility with the lightweight Linux `musl` runtime.

- `BUILD_CONFIGURATION`: Set to `Release` for optimized production builds.
- `RUNTIME=linux-musl-x64`: Targets the `linux-musl` runtime, specifically tailored for Alpine Linux. This ensures compatibility with musl-based systems commonly used in containerized environments like Kubernetes.

```dockerfile
COPY ["MyApplication.WebAPI/MyApplication.WebAPI.csproj", "MyApplication.WebAPI/"]
RUN dotnet restore "./MyApplication.WebAPI/MyApplication.WebAPI.csproj" -r "$RUNTIME"
```

We copy only the project file first to take advantage of Docker’s caching mechanism, which avoids re-downloading dependencies unless the `csproj` file changes. The runtime (`linux-musl-x64`) ensures the restored packages are compatible with the target Linux environment.

```dockerfile
COPY . .
```

After restoring dependencies, the full source code is copied. This order reduces unnecessary rebuilds during development.

```dockerfile
RUN dotnet publish "./MyApplication.WebAPI/MyApplication.WebAPI.csproj" \
    -c "$BUILD_CONFIGURATION" \
    -r "$RUNTIME" \
    --self-contained false \
    -o /app/publish \
    /p:UseAppHost=false \
    /p:PublishReadyToRun=true
```

Here’s a breakdown of the key options used in the dotnet publish command:

- `-r "$RUNTIME"`: Ensures the published app is optimized for the specified runtime (linux-musl-x64).
- `--self-contained false`: Keeps the application lightweight by relying on the shared .NET runtime instead of bundling it.
- `/p:PublishReadyToRun=true`: Precompiles assemblies to improve startup performance, which is crucial for production workloads.

---

### Stage 2: Final Stage

```dockerfile
FROM mcr.microsoft.com/dotnet/aspnet:8.0-alpine
```

The runtime stage uses `mcr.microsoft.com/dotnet/aspnet:8.0-alpine`, which is purpose-built for hosting ASP.NET Core applications and is significantly smaller than the SDK image.

```dockerfile
ENV DOTNET_SYSTEM_GLOBALIZATION_INVARIANT false
RUN apk add --no-cache icu-libs tzdata
```

- `DOTNET_SYSTEM_GLOBALIZATION_INVARIANT`: Set to `false` to enable full globalization support, such as culture-specific formatting.
- `icu-libs`: Installed to provide ICU libraries required for globalization.
- `tzdata`: Adds timezone data, ensuring the application handles time zones correctly.

```dockerfile
WORKDIR /app
USER app
EXPOSE 8080
```

- `USER app`: Enhances security by running the application as a non-root user.
- `EXPOSE 8080`: Exposes port `8080`, commonly used for ASP.NET Core applications.


```dockerfile
COPY --from=build /app/publish .
ENTRYPOINT ["dotnet", "MyApplication.WebAPI.dll"]
```

The published application is copied from the build stage, and the entry point specifies the main DLL file to start the application.

---

## Complete Dockerfile

```dockerfile
# Build Stage
FROM mcr.microsoft.com/dotnet/sdk:8.0-alpine AS build
ARG BUILD_CONFIGURATION=Release
ARG RUNTIME=linux-musl-x64
WORKDIR /src

# Copy only project files for caching
COPY ["MyApplication/MyApplication.csproj", "MyApplication/"]

# Restore dependencies
RUN dotnet restore \
    "./MyApplication/MyApplication.csproj" \
    -r "$RUNTIME"

# Copy the entire source after restore to prevent re-restoring
COPY . .

# Publish the application
RUN dotnet publish \
    "./MyApplication/MyApplication.csproj" \
    -c "$BUILD_CONFIGURATION" \
    -r "$RUNTIME" \
    --self-contained false \
    -o /app/publish \
    /p:UseAppHost=false \
    /p:PublishReadyToRun=true

# Final Stage
FROM mcr.microsoft.com/dotnet/aspnet:8.0-alpine

ENV DOTNET_SYSTEM_GLOBALIZATION_INVARIANT false
RUN apk add --no-cache icu-libs tzdata

WORKDIR /app
USER app
EXPOSE 8080

COPY --from=build /app/publish .
ENTRYPOINT ["dotnet", "MyApplication.dll"]
```

---

## Why Use the linux-musl Runtime?

### Targeting Linux Images

The linux-musl-x64 runtime is specifically designed for environments that use musl as the standard C library (such as Alpine Linux). By targeting this runtime:

- **Compatibility**: Ensures the application runs seamlessly in musl-based environments like Kubernetes or Docker containers using Alpine Linux.
- **Size Efficiency**: Produces smaller binaries compared to glibc-based Linux distributions.
- **Performance**: Musl is known for its lightweight and efficient design, making it ideal for containerized workloads.

### Why Alpine Linux?

Alpine Linux is widely used in containerized environments due to its:

- **Small Size**: A base image is typically less than 5 MB.
- **Security**: A minimal surface area reduces vulnerabilities.
- **Performance**: Optimized for resource-constrained environments.

---

## Conclusion

This `Dockerfile` is optimized for creating lightweight and efficient containers for .NET 8 applications targeted for Linux. By using the `linux-musl-x64` runtime and Alpine Linux, you ensure a small, compatible, and high-performing application that's ready for modern cloud-native deployments.