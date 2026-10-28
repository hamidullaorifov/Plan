# 1. Containers: Dockerfiles, SDK Container Publishing and Compose

> Phase 4 · Week 1 · [← Phase index](README.md)

**Why this matters:** Almost every .NET service now ships as a container image (Docker is used by about 59% of developers). Image size, startup time, security posture (root or not, vulnerable OS packages) and configuration all start in the Dockerfile.

---

## 1.1 The mental model

| Concept | Meaning |
|---|---|
| **Image** | an immutable, layered file system plus metadata (entrypoint, environment, user) |
| **Layer** | the result of one instruction. Layers are cached and shared, so **order matters for caching** |
| **Container** | a running instance of an image, isolated with namespaces and limited with cgroups (CPU and memory) |
| **Registry** | stores images: Docker Hub, GHCR, Azure Container Registry (ACR), Amazon ECR |
| **Tag vs digest** | `api:1.4.2` is a movable label; `api@sha256:…` is immutable. **Deploy by digest** (or immutable tags) |

A container isn't a VM. It shares the host kernel. It should run **one process**, be **stateless** (state lives in databases, queues and blob storage), and be **disposable**.

---

## 1.2 A production-grade Dockerfile for ASP.NET Core

```dockerfile
# syntax=docker/dockerfile:1

# ---------- build stage ----------
FROM mcr.microsoft.com/dotnet/sdk:10.0 AS build
WORKDIR /src

# 1) Copy only the files that affect restore, so this layer stays cached until dependencies change
COPY global.json Directory.Build.props Directory.Packages.props ./
COPY src/Shop.Api/Shop.Api.csproj src/Shop.Api/
COPY src/Shop.Domain/Shop.Domain.csproj src/Shop.Domain/
RUN dotnet restore src/Shop.Api/Shop.Api.csproj

# 2) Copy the rest and publish
COPY . .
RUN dotnet publish src/Shop.Api/Shop.Api.csproj -c Release -o /app/publish --no-restore

# ---------- runtime stage ----------
FROM mcr.microsoft.com/dotnet/aspnet:10.0-noble-chiseled AS final
WORKDIR /app
COPY --from=build /app/publish .
USER $APP_UID
EXPOSE 8080
ENTRYPOINT ["dotnet", "Shop.Api.dll"]
```

| Choice | Why |
|---|---|
| **Multi-stage build** | the SDK (more than 800 MB) stays in the build stage; the final image contains only the runtime and your app |
| **Copy project files first, then restore** | code changes don't invalidate the cached restore layer, so builds go from minutes to seconds |
| **Chiseled runtime image** | an Ubuntu-based image with no shell, no package manager and only the minimal libraries: a smaller attack surface, fewer CVEs and a smaller size |
| **Non-root user** (`$APP_UID`) | .NET 8+ images define an `app` user. Running as root inside a container is a major risk if the app is compromised |
| **Port 8080** | the .NET 8+ images default to `ASPNETCORE_HTTP_PORTS=8080`, a non-privileged port |

> **Chiseled images have no shell**, so `docker exec -it … sh` won't work. Debug with `dotnet-monitor`, ephemeral debug containers (`kubectl debug`) or a temporary non-chiseled tag.

**`.dockerignore`**, so you don't send `bin/`, `obj/`, `.git` and secrets into the build context:

```
**/bin/
**/obj/
.git/
.vs/
.vscode/
**/*.user
**/appsettings.Development.json
**/secrets.json
```

---

## 1.3 Without a Dockerfile: SDK container publishing

The .NET SDK can build OCI images directly, with no Docker daemon needed to *build* them:

```powershell
dotnet publish src/Shop.Api -c Release /t:PublishContainer `
  -p ContainerRepository=shop-api `
  -p ContainerImageTag=1.4.2 `
  -p ContainerFamily=noble-chiseled
```

```xml
<!-- Or set it in the csproj -->
<PropertyGroup>
  <ContainerRepository>shop-api</ContainerRepository>
  <ContainerFamily>noble-chiseled</ContainerFamily>
</PropertyGroup>
```

- Push to a registry with `-p ContainerRegistry=ghcr.io` (after `docker login` or a CI login step).
- Images run as non-root by default and have sensible defaults.
- In .NET 11, the SDK prefers platform-native local runtimes and produces reproducible image digests (`SOURCE_DATE_EPOCH`).
- **Use a Dockerfile instead** when you need OS packages (fonts for PDF rendering, ICU in globalization-invariant images, native libraries) or complex build steps.

---

## 1.4 Configuration and runtime behavior in containers

| Topic | Practice |
|---|---|
| **Configuration** | environment variables (`ConnectionStrings__Shop`, `Payments__BaseUrl`) override `appsettings.json`. Never bake environment-specific values into the image |
| **Secrets** | never in the image or `ENV` lines; inject them at runtime from a vault (guide 5 and guide 8) |
| **Logs** | write to **stdout/stderr** (the console logger or an OpenTelemetry exporter); the platform collects them. No log files inside containers |
| **Graceful shutdown** | the platform sends SIGTERM; .NET stops accepting requests and drains (`HostOptions.ShutdownTimeout`, 30 s by default). Keep it shorter than the orchestrator's grace period |
| **Memory limits** | the GC respects the container memory limit (the heap hard limit defaults to about 75% of the limit). Set limits, or the GC assumes the whole host's memory |
| **CPU limits** | the runtime sizes the thread pool and Server GC heaps to the CPU limit. DATAS (.NET 9+) adapts the heap count |
| **Globalization** | chiseled images have ICU variants (`-extra` tags); or set `InvariantGlobalization` if you don't need culture data |
| **Health** | expose `/health/live` and `/health/ready` for the orchestrator (Phase 2, guide 7) |

---

## 1.5 Docker Compose for local development

```yaml
# compose.yaml
services:
  api:
    build: .
    ports: ["8080:8080"]
    environment:
      ASPNETCORE_ENVIRONMENT: Development
      ConnectionStrings__Shop: Host=postgres;Database=shop;Username=shop;Password=shop
      ConnectionStrings__Redis: redis:6379
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_started
  postgres:
    image: postgres:17
    environment:
      POSTGRES_USER: shop
      POSTGRES_PASSWORD: shop           # local development only
      POSTGRES_DB: shop
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U shop"]
      interval: 5s
      retries: 10
    volumes: ["pgdata:/var/lib/postgresql/data"]
  redis:
    image: redis:7
volumes:
  pgdata:
```

Inside the Compose network, services reach each other **by service name** (`postgres`, `redis`), not via `localhost`.

> For .NET-centric local development, [Aspire](02-aspire.md) is often the better choice: it starts the same dependencies, wires the connection strings automatically and gives you a telemetry dashboard.

---

## 1.6 Image security basics

- **Scan** images in CI: `trivy image shop-api:1.4.2`. Fail the build on critical CVEs that have a fix.
- **Rebuild regularly**, even without code changes, to pick up patched base images. Monthly .NET patches come out on Patch Tuesday.
- **Pin base images** by tag (`10.0-noble-chiseled`) and optionally by digest. Automate updates with Dependabot or Renovate.
- Run with a **read-only root file system** where possible, and drop Linux capabilities (Kubernetes `securityContext`, guide 3).
- **Never** put secrets in build args or layers. They're visible in the image history (`docker history`).

---

## Interview questions

<details><summary><b>1. Why use a multi-stage Dockerfile?</b></summary>

The build tools (the SDK) stay in the build stage, and the final image contains only the runtime and the published app. That makes it smaller, faster to pull and less exposed to vulnerabilities.
</details>

<details><summary><b>2. How do you make Docker builds fast?</b></summary>

Order the layers from least to most frequently changing: copy the project files and restore first, then copy the source and publish. Use a `.dockerignore`, and use BuildKit cache mounts or the CI layer cache.
</details>

<details><summary><b>3. Why run containers as non-root? What are chiseled images?</b></summary>

If the app is compromised, root inside the container makes escaping or damaging the host easier. Chiseled images are minimal Ubuntu images with no shell or package manager and a non-root default user, which reduces the attack surface and the number of CVEs.
</details>

<details><summary><b>4. How should a containerized app handle configuration and logs?</b></summary>

Configuration comes from environment variables and mounted files, with secrets from a vault at runtime and nothing environment-specific baked into the image. Logs go to stdout/stderr (structured), and the platform ships them.
</details>

---

## Exercises

1. Containerize a lab API with the Dockerfile above. Compare its size against a single-stage build and against the SDK container-publish image ([Lab 1](10-labs.md#lab-1-production-grade-image)).
2. Change one line of C# and time the rebuild with good and bad layer ordering.
3. Scan the image with Trivy, and compare the CVE counts for the `aspnet:10.0` and `10.0-noble-chiseled` bases.
4. Set a 256 MB memory limit on the container and watch GC behavior under load with `dotnet-counters`.

## Self-check

- [ ] I can write a cached, multi-stage, non-root Dockerfile for ASP.NET Core.
- [ ] I can publish an image with the SDK and know when a Dockerfile is still needed.
- [ ] I configure containers via environment variables, log to stdout and handle SIGTERM gracefully.
- [ ] I scan images and keep base images patched.

## Further reading

- Microsoft Learn: *Containerize a .NET app*, *.NET container images* (chiseled images), *Publish .NET apps as containers with the SDK*.
- Docker docs: *Best practices for writing Dockerfiles*.
