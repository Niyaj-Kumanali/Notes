# Docker & Azure DevOps Interview Guide for C# Developers

---

## Docker Basics

### What is Docker?

- Docker is an open-source **containerization platform** that packages applications and their dependencies into standardized units called **containers**
- It provides a lightweight, portable way to run applications across different environments — local machine, staging, production, cloud
- Docker uses OS-level virtualization to deliver software in packages called containers
- Containers are isolated from each other and bundle their own software, libraries, and configuration files
- Docker engine runs on the host OS kernel — there is no guest operating system inside each container

```plaintext
+----------------------------------+
|          Host Operating System   |
|   (Windows/Linux/macOS Kernel)  |
+----------------------------------+
| Docker Engine (daemon + CLI)     |
+----------------------------------+
| Container A  | Container B      |
| (App + deps) | (App + deps)     |
+----------------------------------+
```

### Container vs Virtual Machine

| Feature              | Container                        | Virtual Machine                    |
|----------------------|----------------------------------|------------------------------------|
| OS                   | Shares host kernel               | Each has its own OS                |
| Size                 | Megabytes (MB)                   | Gigabytes (GB)                     |
| Startup time         | Seconds                          | Minutes                            |
| Isolation            | Process-level                    | Full hardware-level                |
| Performance          | Near-native                      | Overhead from hypervisor           |
| Density              | Hundreds per host                | Dozens per host                    |
| Portability          | Highly portable                  | Less portable                      |

```plaintext
Virtual Machine Stack:
+---------+ +---------+
|   VM 1  | |   VM 2  |
| App     | | App     |
| Bins/   | | Bins/   |
| Libs    | | Libs    |
| Guest   | | Guest   |
| OS      | | OS      |
+---------+ +---------+
|     Hypervisor      |
+---------------------+
|   Host OS / HW      |
+---------------------+

Container Stack:
+---------+ +---------+
|  Ctr A  | |  Ctr B  |
| App     | | App     |
| Bins/   | | Bins/   |
| Libs    | | Libs    |
+---------+ +---------+
|  Docker Engine      |
+---------------------+
|   Host OS / HW      |
+---------------------+
```

- Containers share the host kernel, which makes them significantly smaller and faster
- A virtual machine includes a full operating system, making it heavier and slower to start
- Containers provide process-level isolation, while VMs provide hardware-level isolation
- For microservices, containers are almost always the better choice due to density and speed

### Why Docker Matters

- **Consistent environments** — "works on my machine" is eliminated; the container is the same everywhere
- **Easy deployment** — push a single image and run it anywhere Docker is installed
- **Scaling** — spin up dozens of identical containers in seconds for load balancing
- **Microservices friendly** — each service gets its own container with its own dependencies
- **Version control for infrastructure** — Dockerfiles are code, stored in source control
- **Isolation** — one application's dependencies don't conflict with another's
- **Reproducibility** — build once, run anywhere, every time the same way

### Image vs Container

- A **Docker image** is a read-only template containing application code, runtime, libraries, and dependencies
- A **Docker container** is a running instance of an image — it has state, can be started, stopped, and deleted
- One image can create many containers, each running independently
- Images are built in layers, each layer being a set of filesystem changes
- Containers add a writable layer on top of the image layers

```bash
# Build an image from a Dockerfile
docker build -t myapp:1.0 .

# Run a container from an image
docker run -d -p 8080:80 --name myapp myapp:1.0

# List running containers
docker ps

# List all images on the system
docker images
```

### Image Layers and Caching

- Every instruction in a Dockerfile creates a new layer on top of the previous one
- Docker caches layers — if a layer hasn't changed, Docker reuses the cached version instead of rebuilding
- Layer order matters: put instructions that change infrequently at the top, and frequently changing instructions at the bottom
- This dramatically speeds up builds when only small parts of the code change
- Each layer is immutable and content-addressable (identified by a SHA256 hash)

```plaintext
Dockerfile Instruction -> Layer Created -> Cached?
-----------------------------------------------
FROM mcr.microsoft.com/dotnet/sdk:8.0  -> Layer 1 (cached on 2nd build)
WORKDIR /app                           -> Layer 2 (cached)
COPY *.csproj .                        -> Layer 3 (cached if csproj unchanged)
RUN dotnet restore                     -> Layer 4 (cached if restore unchanged)
COPY . .                               -> Layer 5 (rebuilt if ANY file changed)
RUN dotnet publish -c Release          -> Layer 6 (rebuilt)
```

```bash
# View the layers of an image
docker history myapp:1.0

# Inspect image layers
docker inspect myapp:1.0 --format='{{.RootFS.Layers}}'
```

### Dockerfile Instructions

- **FROM** — sets the base image for subsequent instructions; every Dockerfile must start with FROM
- **WORKDIR** — sets the working directory inside the container; creates it if it doesn't exist
- **COPY** — copies files from the host machine into the container's filesystem
- **ADD** — similar to COPY but also supports URLs and automatic tar extraction (prefer COPY over ADD)
- **RUN** — executes a command during image build; each RUN creates a new layer
- **EXPOSE** — documents which port the container listens on at runtime (does NOT publish the port)
- **CMD** — provides the default command to run when a container starts; only one CMD per Dockerfile
- **ENTRYPOINT** — configures the container to run as an executable; more rigid than CMD
- **ENV** — sets environment variables in the image
- **ARG** — defines build-time variables that users can pass to the builder
- **USER** — sets the user or UID for subsequent instructions and for container runtime
- **HEALTHCHECK** — tells Docker how to test if the container is still working

```dockerfile
FROM mcr.microsoft.com/dotnet/aspnet:8.0
WORKDIR /app
COPY . .
EXPOSE 80
ENV ASPNETCORE_URLS=http://+:80
ENTRYPOINT ["dotnet", "MyApp.dll"]
```

```dockerfile
# CMD vs ENTRYPOINT
# CMD can be overridden by command-line arguments
CMD ["dotnet", "MyApp.dll"]

# ENTRYPOINT is harder to override; use it for the main executable
ENTRYPOINT ["dotnet"]
CMD ["MyApp.dll"]
```

---

## Dockerfile for ASP.NET Core

### Multi-Stage Build

- Multi-stage builds use **multiple FROM statements** in a single Dockerfile
- Each FROM begins a new stage, and you can selectively copy artifacts from earlier stages
- The final image only contains what you explicitly COPY from the last stage
- This keeps the production image small (runtime-only, no SDK, no build tools)
- The build stage uses the full SDK image; the runtime stage uses the slim aspnet image

```dockerfile
# Stage 1: Build
FROM mcr.microsoft.com/dotnet/sdk:8.0 AS build
WORKDIR /src

# Copy csproj and restore as distinct layers (caching)
COPY ["MyApp.csproj", "."]
RUN dotnet restore

# Copy everything and build
COPY . .
RUN dotnet publish -c Release -o /app/publish

# Stage 2: Runtime
FROM mcr.microsoft.com/dotnet/aspnet:8.0 AS runtime
WORKDIR /app
COPY --from=build /app/publish .
EXPOSE 8080
ENV ASPNETCORE_URLS=http://+:8080
ENTRYPOINT ["dotnet", "MyApp.dll"]
```

### Why Multi-Stage Matters

- A single-stage build with the SDK image produces an image of 800MB+ (includes compiler, NuGet, etc.)
- A multi-stage build with only the aspnet runtime produces an image of 200MB or less
- Smaller images mean faster pulls, faster deployments, and smaller attack surface
- The SDK is only needed during the build stage, never at runtime
- This is the **recommended pattern** for all production ASP.NET Core Docker images

```bash
# Compare image sizes
# Single-stage (SDK only): ~800MB
# Multi-stage (aspnet runtime): ~200MB
docker images
```

### Copy and Run Pattern

- The standard pattern is: copy csproj first, restore, then copy everything else
- This maximizes cache efficiency — the restore layer only rebuilds when csproj changes
- If you copy everything first, any source code change invalidates the restore cache

```dockerfile
# GOOD: copy csproj first for cache efficiency
COPY ["MyApp.csproj", "."]
RUN dotnet restore
COPY . .

# BAD: copies everything, no caching benefit
COPY . .
RUN dotnet restore
```

### Exposing Ports

- Use EXPOSE to document which port your application listens on
- EXPOSE does NOT actually publish the port — you must use `-p` at runtime
- In .NET 8, the default port is 8080 (changed from 80 in .NET 7 and earlier)

```dockerfile
# Document the port
EXPOSE 8080

# At runtime, publish the port
# docker run -p 8080:8080 myapp:1.0
```

### Health Checks

- HEALTHCHECK tells Docker how to verify that the container is functioning correctly
- Docker runs the health check command periodically and marks the container as unhealthy if it fails
- This is critical for orchestrators (Docker Swarm, Kubernetes) to know when to restart or replace a container

```dockerfile
HEALTHCHECK --interval=30s --timeout=5s --start-period=10s --retries=3 \
  CMD curl -f http://localhost:8080/health || exit 1
```

```csharp
// In your ASP.NET Core Program.cs, expose a health endpoint:
app.MapHealthChecks("/health");
```

### Environment Variables

- ENV sets default environment variables in the image
- These can be overridden at runtime with `-e` flag
- Common ASP.NET Core environment variables: ASPNETCORE_ENVIRONMENT, ASPNETCORE_URLS, ASPNETCORE_HTTP_PORTS

```dockerfile
ENV ASPNETCORE_ENVIRONMENT=Production
ENV ASPNETCORE_URLS=http://+:8080
ENV DOTNET_EnableDiagnostics=0
```

```bash
# Override at runtime
docker run -e ASPNETCORE_ENVIRONMENT=Staging -p 8080:8080 myapp:1.0
```

---

## Docker Compose

### What is Docker Compose?

- Docker Compose is a tool for defining and running **multi-container Docker applications**
- You define all services, networks, and volumes in a single YAML file
- One command starts everything: `docker compose up`
- Ideal for development environments where you need app + database + cache + other services
- Docker Compose v2 is the current version (replaces docker-compose with a hyphen)

### docker-compose.yml Structure

```yaml
version: '3.8'

services:
  webapp:
    build:
      context: .
      dockerfile: Dockerfile
    ports:
      - "8080:8080"
    environment:
      - ASPNETCORE_ENVIRONMENT=Development
      - ConnectionStrings__DefaultConnection=Server=db;Database=MyApp;User=sa;Password=YourPassword123!
    depends_on:
      - db
      - redis
    networks:
      - app-network

  db:
    image: mcr.microsoft.com/mssql/server:2022-latest
    environment:
      - ACCEPT_EULA=Y
      - SA_PASSWORD=YourPassword123!
    volumes:
      - db-data:/var/opt/mssql
    ports:
      - "1433:1433"
    networks:
      - app-network

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
    networks:
      - app-network

volumes:
  db-data:

networks:
  app-network:
    driver: bridge
```

### Services

- Each service is a container configuration
- A service can be built from a Dockerfile or pulled from a registry as a pre-built image
- Services are isolated but can communicate using the service name as a hostname

### Networks

- Compose creates a default bridge network for all services
- Services can reach each other by service name (e.g., `db`, `redis`, `webapp`)
- Custom networks provide additional isolation between groups of services
- Bridge is the default driver; overlay is used for Swarm mode

### Volumes

- Volumes persist data beyond the lifecycle of a container
- Named volumes (like `db-data`) are managed by Docker and survive `docker compose down`
- Bind mounts map a host directory to a container directory (useful for development)
- Volumes are essential for databases — without them, data is lost when the container stops

```bash
# Start all services
docker compose up -d

# Stop all services
docker compose down

# Stop and remove volumes (deletes data!)
docker compose down -v

# View logs
docker compose logs -f webapp

# Rebuild after Dockerfile changes
docker compose up -d --build
```

### Depends_on

- Controls startup order — the dependent service starts before the service that depends on it
- `depends_on` only waits for the container to start, NOT for the application inside to be ready
- Use health checks with depends_on to wait for actual readiness:

```yaml
services:
  webapp:
    depends_on:
      db:
        condition: service_healthy
      redis:
        condition: service_started

  db:
    image: mcr.microsoft.com/mssql/server:2022-latest
    healthcheck:
      test: /opt/mssql-tools/bin/sqlcmd -S localhost -U sa -P "YourPassword123!" -Q "SELECT 1"
      interval: 10s
      timeout: 5s
      retries: 5
```

### Development vs Production Compose Files

```yaml
# docker-compose.yml (base)
services:
  webapp:
    build: .
    ports:
      - "8080:8080"

# docker-compose.override.yml (development - auto-loaded)
services:
  webapp:
    volumes:
      - .:/app              # Hot reload
    environment:
      - ASPNETCORE_ENVIRONMENT=Development
      - ASPNETCORE_DETAILEDERRORS=true
    command: ["dotnet", "watch", "run"]

# docker-compose.prod.yml (production)
services:
  webapp:
    image: myregistry.azurecr.io/myapp:1.0
    environment:
      - ASPNETCORE_ENVIRONMENT=Production
    restart: always
```

```bash
# Development (uses override automatically)
docker compose up

# Production (explicit file)
docker compose -f docker-compose.yml -f docker-compose.prod.yml up -d
```

---

## Docker in Development

### Hot Reload with Volume Mounts

- Volume mounts map your local source code into the container
- Combined with `dotnet watch`, changes to your code are reflected immediately without rebuilding
- This gives you the speed of local development with the consistency of containerized dependencies

```yaml
# docker-compose.override.yml
services:
  webapp:
    volumes:
      - ./src:/app              # Map source code
      - /app/bin                 # Exclude bin (anonymous volume)
      - /app/obj                 # Exclude obj (anonymous volume)
    command: dotnet watch run
```

```dockerfile
# In development, use the SDK image (needed for dotnet watch)
FROM mcr.microsoft.com/dotnet/sdk:8.0
WORKDIR /app
# ... restore and build steps
ENTRYPOINT ["dotnet", "watch", "run"]
```

### Database Containers

```yaml
# SQL Server
services:
  sqlserver:
    image: mcr.microsoft.com/mssql/server:2022-latest
    environment:
      - ACCEPT_EULA=Y
      - SA_PASSWORD=YourStrong!Password123
    ports:
      - "1433:1433"
    volumes:
      - sqlserver-data:/var/opt/mssql

# PostgreSQL
services:
  postgres:
    image: postgres:16-alpine
    environment:
      - POSTGRES_USER=postgres
      - POSTGRES_PASSWORD=YourPassword123
      - POSTGRES_DB=myapp
    ports:
      - "5432:5432"
    volumes:
      - postgres-data:/var/lib/postgresql/data

# SQLite (file-based, no server needed)
services:
  webapp:
    volumes:
      - ./data:/app/data        # SQLite database file
```

### Redis Container

```yaml
services:
  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
    command: redis-server --appendonly yes
    volumes:
      - redis-data:/data
```

```csharp
// Connect to Redis from your ASP.NET Core app
// Connection string in appsettings.json: "localhost:6379"
builder.Services.AddStackExchangeRedisCache(options =>
{
    options.Configuration = builder.Configuration.GetConnectionString("Redis");
    options.InstanceName = "MyApp_";
});
```

### Debugging Containers in Visual Studio / VS Code

```yaml
# VS Code launch.json for container debugging
{
    "version": "0.2.0",
    "configurations": [
        {
            "name": ".NET Core Docker",
            "type": "coreclr",
            "request": "launch",
            "preLaunchTask": "docker-build",
            "program": "${workspaceFolder}/bin/Debug/net8.0/MyApp.dll",
            "args": [],
            "cwd": "${workspaceFolder}",
            "env": {
                "ASPNETCORE_ENVIRONMENT": "Development"
            }
        }
    ]
}
```

```json
// VS Code tasks.json for Docker build
{
    "version": "2.0.0",
    "tasks": [
        {
            "label": "docker-build",
            "type": "docker-build",
            "runOptions": {
                "runAfter": []
            }
        }
    ]
}
```

```powershell
# Debug with environment variables
docker run -it --rm -p 5000:8080 \
  -e ASPNETCORE_ENVIRONMENT=Development \
  -e DOTNET_EnableDiagnostics=1 \
  myapp:dev
```

---

## Azure DevOps

### What is Azure DevOps?

- Azure DevOps is Microsoft's **complete DevOps platform** for planning, building, testing, and deploying software
- It provides integrated tools for the entire software development lifecycle (SDLC)
- Available as a cloud service (Azure DevOps Services) or on-premises (Azure DevOps Server, formerly TFS)
- Supports any language, platform, or cloud provider — not just .NET or Azure
- Offers both free tier (for small teams) and paid plans with advanced features

### Azure DevOps Services (Five Pillars)

- **Repos** — Git repositories for source control with pull requests, code reviews, and branch policies
- **Pipelines** — CI/CD automation for building, testing, and deploying applications
- **Boards** — Agile project management with work items, Kanban boards, sprint planning, and backlogs
- **Artifacts** — Package management for NuGet, npm, Maven, and universal packages; also hosts pipeline artifacts
- **Test Plans** — Manual and exploratory testing with test plans, test suites, and test execution tracking

### YAML Pipelines vs Classic Pipelines

- **YAML Pipelines** — defined as code in a YAML file stored in the repository; version-controlled, reviewable, and reproducible
- **Classic Pipelines** — configured through the web UI with a visual editor; easier to start but not version-controlled
- Microsoft recommends YAML pipelines as the default for new projects
- YAML pipelines support multi-stage, multi-job, multi-step configurations

```yaml
# Example YAML pipeline
trigger:
  - main

pool:
  vmImage: 'ubuntu-latest'

stages:
  - stage: Build
    jobs:
      - job: BuildJob
        steps:
          - task: DotNetCoreCLI@2
            displayName: 'Restore'
            inputs:
              command: 'restore'
              projects: '**/*.csproj'

          - task: DotNetCoreCLI@2
            displayName: 'Build'
            inputs:
              command: 'build'
              projects: '**/*.csproj'
              arguments: '--configuration Release --no-restore'
```

### Trigger Types

- **CI Trigger (Continuous Integration)** — fires automatically when code is pushed to a branch
- **PR Trigger (Pull Request)** — runs validation when a pull request is created or updated
- **Scheduled Trigger** — runs on a defined schedule (e.g., nightly at 2 AM)
- **Manual Trigger** — run pipeline on-demand from the UI or API
- **Batch CI** — batches multiple pushes into a single pipeline run for efficiency

```yaml
# CI trigger
trigger:
  branches:
    include:
      - main
      - develop
  paths:
    exclude:
      - '**/*.md'
      - 'docs/**'

# PR trigger
pr:
  branches:
    include:
      - main

# Scheduled trigger
schedules:
  - cron: "0 2 * * *"
    displayName: 'Nightly Build'
    branches:
      include:
        - main
    always: true
```

---

## CI/CD Pipeline

### What is CI (Continuous Integration)?

- CI is the practice of **integrating code changes frequently** into a shared repository
- Each integration is verified by an automated build and test suite
- Catches bugs early, before they reach production
- Every push or pull request triggers the pipeline to build and test the code
- The goal: always have a working codebase that can be deployed

### What is CD (Continuous Delivery/Deployment)?

- **Continuous Delivery** — every change is ready to deploy to production at the push of a button; requires manual approval for production releases
- **Continuous Deployment** — every change that passes all stages is automatically deployed to production with no human intervention
- CD ensures code is always in a deployable state
- The pipeline validates the code through multiple environments before reaching production

```plaintext
Developer pushes code
        |
        v
  +-----------+
  |   BUILD   |  Compile, restore packages
  +-----------+
        |
        v
  +-----------+
  |   TEST    |  Unit tests, integration tests, code coverage
  +-----------+
        |
        v
  +-----------+
  |  PUBLISH  |  Create artifacts, publish packages
  +-----------+
        |
        v
  +-----------+
  | DEPLOY    |  Deploy to Dev -> Staging -> Production
  +-----------+
```

### Pipeline Stages

```yaml
stages:
  - stage: Build
    displayName: 'Build Stage'
    jobs:
      - job: Build
        steps:
          - task: DotNetCoreCLI@2
            inputs:
              command: 'build'

  - stage: Test
    displayName: 'Test Stage'
    dependsOn: Build
    jobs:
      - job: UnitTests
        steps:
          - task: DotNetCoreCLI@2
            inputs:
              command: 'test'

  - stage: DeployDev
    displayName: 'Deploy to Dev'
    dependsOn: Test
    jobs:
      - deployment: DeployDev
        environment: 'development'

  - stage: DeployStaging
    displayName: 'Deploy to Staging'
    dependsOn: DeployDev
    jobs:
      - deployment: DeployStaging
        environment: 'staging'

  - stage: DeployProduction
    displayName: 'Deploy to Production'
    dependsOn: DeployStaging
    jobs:
      - deployment: DeployProduction
        environment: 'production'
```

### Build Agents

- **Microsoft-hosted agents** — maintained by Microsoft; available in Azure; different images for Windows, Linux, macOS; use them for most workloads
- **Self-hosted agents** — you install and manage the agent on your own infrastructure; useful for private networks, custom tooling, or cost control
- Microsoft-hosted agents are ephemeral — each job runs on a fresh virtual machine
- Self-hosted agents can be pooled and reused across pipelines

```yaml
# Microsoft-hosted agent
pool:
  vmImage: 'ubuntu-latest'

# Self-hosted agent
pool:
  name: 'MySelfHostedPool'
  demands:
    - npm
    - dotnet
```

### Pipeline Variables and Variable Groups

```yaml
# Inline variables
variables:
  buildConfiguration: 'Release'
  dotnetVersion: '8.0.x'

# Variable group from Azure DevOps Library
variables:
  - group: 'MyApp-Shared-Variables'

# Variable group with runtime values
variables:
  - group: 'MyApp-DevOps'
  - name: 'buildConfiguration'
    value: 'Release'
```

```yaml
# Reference variables in tasks
- task: DotNetCoreCLI@2
  inputs:
    command: 'build'
    arguments: '--configuration $(buildConfiguration)'
```

### Pipeline Templates and Reuse

```yaml
# templates/build-template.yml
parameters:
  - name: buildConfiguration
    type: string
    default: 'Release'

steps:
  - task: DotNetCoreCLI@2
    displayName: 'Restore'
    inputs:
      command: 'restore'
      projects: '**/*.csproj'

  - task: DotNetCoreCLI@2
    displayName: 'Build'
    inputs:
      command: 'build'
      projects: '**/*.csproj'
      arguments: '--configuration ${{ parameters.buildConfiguration }} --no-restore'

# azure-pipelines.yml (using template)
stages:
  - stage: Build
    jobs:
      - job: Build
        steps:
          - template: templates/build-template.yml
            parameters:
              buildConfiguration: 'Release'
```

### Artifacts and Publish Task

```yaml
# Build and publish artifact
- task: DotNetCoreCLI@2
  displayName: 'Publish'
  inputs:
    command: 'publish'
    publishWebProjects: true
    arguments: '--configuration Release --output $(Build.ArtifactStagingDirectory)'

# Publish artifact to pipeline
- task: PublishBuildArtifacts@1
  displayName: 'Publish Artifact'
  inputs:
    pathToPublish: '$(Build.ArtifactStagingDirectory)'
    artifactName: 'webapp-drop'
    publishLocation: 'Container'
```

### Environment-Specific Deployments

```yaml
# Deploy to different environments with different settings
- stage: DeployDev
  jobs:
    - deployment: DeployDev
      environment: 'development'
      strategy:
        runOnce:
          deploy:
            steps:
              - task: AzureWebApp@1
                inputs:
                  azureSubscription: 'Azure-Dev-Subscription'
                  appName: 'myapp-dev'
                  package: '$(Pipeline.Workspace)/webapp-drop/**/*.zip'

- stage: DeployProduction
  jobs:
    - deployment: DeployProd
      environment: 'production'
      strategy:
        runOnce:
          deploy:
            steps:
              - task: AzureWebApp@1
                inputs:
                  azureSubscription: 'Azure-Prod-Subscription'
                  appName: 'myapp-prod'
                  package: '$(Pipeline.Workspace)/webapp-drop/**/*.zip'
```

### Deployment Gates and Approvals

- **Approvals** — require explicit manual approval before deployment proceeds (configured in Environment settings)
- **Gates** — automated checks that must pass before deployment (e.g., Azure Monitor health, custom API checks)
- **Branch policies** — require PR reviews, work item linkage, and successful builds before merging
- These ensure quality and compliance without slowing down the pipeline for lower environments

```yaml
# In the Environment configuration (UI), set up:
# - Approvers: specific users who must approve
# - Gates: automated checks with timeout and polling interval

# Pipeline waits for the environment gate/approval
- stage: DeployProduction
  jobs:
    - deployment: DeployProd
      environment: 'production'  # This environment has approval gates configured
```

---

## Azure DevOps Pipeline for .NET

### DotNetCoreCLI@2 Tasks

```yaml
# Restore
- task: DotNetCoreCLI@2
  displayName: 'dotnet restore'
  inputs:
    command: 'restore'
    projects: '**/*.csproj'
    feedsToUse: 'select'
    vstsFeed: 'my-feed-id'

# Build
- task: DotNetCoreCLI@2
  displayName: 'dotnet build'
  inputs:
    command: 'build'
    projects: '**/*.csproj'
    arguments: '--configuration $(buildConfiguration) --no-restore'

# Test
- task: DotNetCoreCLI@2
  displayName: 'dotnet test'
  inputs:
    command: 'test'
    projects: '**/*Tests.csproj'
    arguments: '--configuration $(buildConfiguration) --collect:"XPlat Code Coverage" --logger trx --results-directory $(Agent.TempDirectory)/TestResults'

# Publish
- task: DotNetCoreCLI@2
  displayName: 'dotnet publish'
  inputs:
    command: 'publish'
    publishWebProjects: true
    arguments: '--configuration $(buildConfiguration) --no-build --output $(Build.ArtifactStagingDirectory)'
    zipAfterPublish: true
    modifyOutputPath: true
```

### NuGet Restore

```yaml
# Task-based NuGet restore
- task: NuGetToolInstaller@1
  displayName: 'Install NuGet'

- task: NuGetCommand@2
  displayName: 'NuGet Restore'
  inputs:
    command: 'restore'
    restoreSolution: '**/*.sln'
    feedsToUse: 'config'
    nugetConfigPath: 'nuget.config'

# Or use dotnet restore (recommended for SDK-style projects)
- task: DotNetCoreCLI@2
  displayName: 'dotnet restore'
  inputs:
    command: 'restore'
    projects: '**/*.sln'
    feedsToUse: 'select'
    vstsFeed: 'my-feed-id'
```

### Running Unit Tests in Pipeline

```yaml
- task: DotNetCoreCLI@2
  displayName: 'Run Unit Tests'
  inputs:
    command: 'test'
    projects: '**/*Tests.csproj'
    arguments: >
      --configuration $(buildConfiguration)
      --no-build
      --logger "trx;LogFileName=TestResults.trx"
      --collect:"XPlat Code Coverage"
      --results-directory $(Agent.TempDirectory)/TestResults

# Publish test results
- task: PublishTestResults@2
  displayName: 'Publish Test Results'
  inputs:
    testResultsFormat: 'VSTest'
    testResultsFiles: '**/*.trx'
    searchFolder: '$(Agent.TempDirectory)/TestResults'
    mergeTestResults: true
    testRunTitle: '.NET Tests'
```

### Code Coverage Reports

```yaml
# Collect code coverage during test run
- task: DotNetCoreCLI@2
  inputs:
    command: 'test'
    projects: '**/*Tests.csproj'
    arguments: '--collect:"XPlat Code Coverage" --results-directory $(Agent.TempDirectory)/Coverage'

# Publish code coverage
- task: PublishCodeCoverageResults@2
  displayName: 'Publish Code Coverage'
  inputs:
    summaryFileLocation: '$(Agent.TempDirectory)/Coverage/**/coverage.cobertura.xml'
```

### SonarQube Integration

```yaml
# Install SonarQube extension first (from Azure DevOps marketplace)

# Prepare analysis
- task: SonarQubePrepare@5
  displayName: 'Prepare SonarQube Analysis'
  inputs:
    SonarQube: 'SonarQube-Connection'
    scannerMode: 'MSBuild'
    projectKey: 'my-project'
    projectName: 'My Project'

# Build (triggers SonarQube scanner)
- task: DotNetCoreCLI@2
  displayName: 'Build'
  inputs:
    command: 'build'

# Analyze
- task: SonarQubeAnalyze@5
  displayName: 'Run SonarQube Analysis'

# Publish quality gate result
- task: SonarQubePublish@5
  displayName: 'Publish SonarQube Results'
  inputs:
    pollingTimeoutSec: '300'
```

---

## IIS Hosting

### What is IIS?

- IIS (Internet Information Services) is Microsoft's **web server** included with Windows
- It handles HTTP/HTTPS requests and hosts web applications
- IIS manages application pools, SSL termination, authentication, and request filtering
- ASP.NET Core applications run on IIS through the ASP.NET Core Module (ANCM)

### ASP.NET Core Module (ANCM)

- ANCM is a native IIS module that handles incoming HTTP requests for ASP.NET Core applications
- It forwards requests to the Kestrel web server, which actually processes them
- ANCM manages process start, stop, and restart
- It supports in-process hosting (preferred) and out-of-process hosting

```plaintext
In-Process Hosting (Recommended):
Client -> IIS -> ANCM -> ASP.NET Core (in IIS worker process w3wp.exe)

Out-Process Hosting:
Client -> IIS -> ANCM -> Kestrel (separate process) -> ASP.NET Core
```

### Application Pool Settings

- Application pools in IIS isolate web applications from each other
- For ASP.NET Core, the application pool must be set to **"No Managed Code"**
- This tells IIS not to load the .NET CLR, since ASP.NET Core runs on .NET (not .NET Framework)
- Setting it to anything else causes errors or performance issues

```plaintext
IIS Application Pool Configuration:
  Name:             MyAppPool
  .NET CLR version: No Managed Code
  Managed pipeline mode: Integrated
  Identity:         ApplicationPoolIdentity
```

### Hosting Bundle Installation

- The ASP.NET Core Hosting Bundle installs ANCM, the .NET runtime, and IIS support
- Download from Microsoft's website (for each .NET version)
- Must be installed on the IIS server before deploying ASP.NET Core apps
- After installation, restart IIS: `iisreset`

```powershell
# Verify ANCM is installed
Get-WebGlobalModule | Where-Object { $_.Name -eq "AspNetCoreModuleV2" }

# Restart IIS after installing hosting bundle
iisreset
```

### web.config for ASP.NET Core

```xml
<?xml version="1.0" encoding="utf-8"?>
<configuration>
  <location path="." inheritInChildApplications="false">
    <system.webServer>
      <handlers>
        <add name="aspNetCore" path="*" verb="*"
             modules="AspNetCoreModuleV2"
             resourceType="Unspecified" />
      </handlers>
      <aspNetCore processPath="dotnet"
                  arguments=".\MyApp.dll"
                  stdoutLogEnabled="false"
                  stdoutLogFile=".\logs\stdout"
                  hostingModel="inprocess" />
    </system.webServer>
  </location>
</configuration>
```

```xml
<!-- For out-of-process hosting -->
<aspNetCore processPath="dotnet"
            arguments=".\MyApp.dll"
            stdoutLogEnabled="true"
            stdoutLogFile=".\logs\stdout"
            hostingModel="outofprocess" />
```

### HTTPS Bindings and SSL

```xml
<!-- IIS HTTPS binding configuration -->
<system.webServer>
  <security>
    <access sslFlags="Ssl" />
  </security>
</system.webServer>
```

```powershell
# Add HTTPS binding in IIS
New-WebBinding -Name "MyApp" -Protocol "https" -Port 443 -IPAddress "*"

# Assign SSL certificate
$cert = Get-ChildItem -Path Cert:\LocalMachine\My | Where-Object { $_.Subject -like "*myapp*" }
$binding = Get-WebBinding -Name "MyApp" -Protocol "https"
$binding.AddSslCertificate($cert.Thumbprint, "My")
```

### Deployment Modes

- **Framework-dependent deployment** — the target machine must have the .NET runtime installed; smaller deployment size; uses the shared runtime
- **Self-contained deployment** — includes the .NET runtime with the application; larger deployment size; no runtime installation required on target machine
- **Framework-dependent with trimmed** — removes unused assemblies to reduce size
- **Single file** — all files bundled into a single executable

```xml
<!-- In .csproj file -->
<PropertyGroup>
  <!-- Framework-dependent (default) -->
  <OutputType>Exe</OutputType>
  <TargetFramework>net8.0</TargetFramework>
</PropertyGroup>

<!-- Self-contained -->
<PropertyGroup>
  <RuntimeIdentifier>win-x64</RuntimeIdentifier>
  <SelfContained>true</SelfContained>
</PropertyGroup>
```

```bash
# Framework-dependent publish
dotnet publish -c Release -o ./publish

# Self-contained publish
dotnet publish -c Release -r win-x64 --self-contained -o ./publish-sc

# Single file publish
dotnet publish -c Release -r win-x64 --self-contained -p:PublishSingleFile=true -o ./publish-single
```

### Publish Profiles

```xml
<!-- Properties/PublishProfiles/FolderProfile.pubxml -->
<Project>
  <PropertyGroup>
    <WebPublishMethod>FileSystem</WebPublishMethod>
    <PublishProvider>FileSystem</PublishProvider>
    <Configuration>Release</Configuration>
    <TargetFramework>net8.0</TargetFramework>
    <PublishDir>\\server\deploy\MyApp</PublishDir>
  </PropertyGroup>
</Project>
```

```xml
<!-- Properties/PublishProfiles/IISProfile.pubxml -->
<Project>
  <PropertyGroup>
    <WebPublishMethod>MSDeploy</WebPublishMethod>
    <PublishProvider>IIS</PublishProvider>
    <SiteUrlToLaunchAfterPublish>https://myapp.example.com</SiteUrlToLaunchAfterPublish>
    <LaunchSiteAfterPublish>True</LaunchSiteAfterPublish>
    <ExcludeApp_Data>False</ExcludeApp_Data>
    <PublishIis>True</PublishIis>
    <SiteName>MyApp</SiteName>
    <AppendRuntimeToOutputPath>True</AppendRuntimeToOutputPath>
  </PropertyGroup>
</Project>
```

### Troubleshooting IIS Errors

- **HTTP 500.19** — web.config is malformed or ANCM is not installed; check XML syntax and install hosting bundle
- **HTTP 502.5** — process failed to start; check that the dotnet runtime is installed and the path in web.config is correct
- **HTTP 503** — application pool is stopped; check the event viewer and restart the app pool
- **HTTP 403** — identity doesn't have read permissions on the application folder

```powershell
# Enable detailed errors in web.config
<aspNetCore processPath="dotnet"
            arguments=".\MyApp.dll"
            stdoutLogEnabled="true"
            stdoutLogFile=".\logs\stdout"
            hostingModel="inprocess" />

# Check Windows Event Viewer for ANCM errors
Get-EventLog -LogName Application -Source "IIS*" -Newest 20

# Check IIS logs
Get-Content "C:\inetpub\logs\LogFiles\W3SVC1\ex*.log" -Tail 50

# Test IIS configuration
appcmd list app
appcmd list apppool
```

---

## Common Mistakes

### Not Using Multi-Stage Docker Builds

```dockerfile
# BAD: Single stage — produces 800MB+ image
FROM mcr.microsoft.com/dotnet/sdk:8.0
WORKDIR /app
COPY . .
RUN dotnet publish -c Release -o /app
EXPOSE 8080
CMD ["dotnet", "/app/MyApp.dll"]

# GOOD: Multi-stage — produces ~200MB image
FROM mcr.microsoft.com/dotnet/sdk:8.0 AS build
WORKDIR /src
COPY *.csproj .
RUN dotnet restore
COPY . .
RUN dotnet publish -c Release -o /app/publish

FROM mcr.microsoft.com/dotnet/aspnet:8.0
WORKDIR /app
COPY --from=build /app/publish .
EXPOSE 8080
CMD ["dotnet", "MyApp.dll"]
```

### Hardcoding Secrets in Dockerfile

```dockerfile
# BAD: Secret in image layer (visible in docker history)
ENV ConnectionStrings__Default=Server=db;Password=SuperSecret123!
RUN dotnet user-secrets set "Password" "SuperSecret123"

# GOOD: Use environment variables at runtime or Docker secrets
# Pass at runtime: docker run -e ConnectionStrings__Default=... myapp
# Or use Docker secrets with Swarm/Kubernetes
```

### Not Cleaning Up Docker Images

```bash
# Remove dangling images
docker image prune

# Remove all unused images, containers, networks
docker system prune -a

# Remove images for a specific project
docker rmi $(docker images "myapp*" -q)

# Check disk usage
docker system df
```

### Pipeline Too Slow (Not Using Caching)

```yaml
# BAD: Copies all source before restore (no cache)
- task: DotNetCoreCLI@2
  inputs:
    command: 'restore'
    projects: '**/*.csproj'

# GOOD: Cache NuGet packages
- task: Cache@2
  displayName: 'Cache NuGet Packages'
  inputs:
    key: 'nuget | "$(Agent.OS)" | **/*.csproj'
    restoreKeys: 'nuget | "$(Agent.OS)"'
    path: '$(NUGET_PACKAGES)'

- task: DotNetCoreCLI@2
  inputs:
    command: 'restore'
```

### Not Using Artifacts Properly

```yaml
# BAD: Deploying directly from build output
# Build and deploy in the same step — no separation of concerns

# GOOD: Build once, publish artifact, deploy from artifact
- task: PublishBuildArtifacts@1
  inputs:
    pathToPublish: '$(Build.ArtifactStagingDirectory)'
    artifactName: 'drop'

# Then in deploy stage, download and deploy
- task: DownloadBuildArtifact@1
  inputs:
    artifactName: 'drop'
    downloadPath: '$(System.ArtifactsDirectory)'

- task: AzureWebApp@1
  inputs:
    package: '$(System.ArtifactsDirectory)/drop/**/*.zip'
```

### IIS Application Pool Misconfiguration

```plaintext
# BAD: Using .NET CLR instead of No Managed Code
App Pool: MyAppPool
.NET CLR version: v4.0.30319    ← WRONG for ASP.NET Core
Managed pipeline mode: Integrated

# GOOD: No Managed Code for ASP.NET Core
App Pool: MyAppPool
.NET CLR version: No Managed Code  ← CORRECT
Managed pipeline mode: Integrated
```

---

## Interview Questions

### 1. What is the difference between a Docker container and a virtual machine?

**Answer:**
A Docker container shares the host operating system kernel and is lightweight (MBs, seconds to start). A virtual machine includes a full guest operating system and is heavy (GBs, minutes to start). Containers provide process-level isolation while VMs provide hardware-level isolation. Containers are better for microservices and high-density deployments, while VMs are better when you need strong isolation or different OS kernels.

### 2. Why would you use a multi-stage Docker build?

**Answer:**
A multi-stage build separates the build environment from the runtime environment. The build stage uses the full SDK image (800MB+) to compile the application. The runtime stage uses only the slim ASP.NET Core runtime image (~200MB). This produces a much smaller final image with a reduced attack surface. Only the compiled output is copied from the build stage, keeping the production image clean.

### 3. What is the difference between CMD and ENTRYPOINT in a Dockerfile?

**Answer:**
`CMD` provides default arguments for the container and can be easily overridden by command-line arguments. `ENTRYPOINT` configures the container to run as an executable and is harder to override. In practice, `ENTRYPOINT` sets the main command (e.g., `dotnet`) and `CMD` provides default arguments (e.g., `MyApp.dll`). You can combine them: `ENTRYPOINT ["dotnet"]` with `CMD ["MyApp.dll"]`.

### 4. Explain the CI/CD pipeline stages and their purpose.

**Answer:**
- **Build** — compiles the code, restores packages, and produces binaries
- **Test** — runs unit tests, integration tests, and code coverage analysis
- **Publish** — creates deployment artifacts (ZIP files, Docker images, NuGet packages)
- **Deploy** — deploys artifacts to target environments (Dev, Staging, Production) with appropriate approvals

The pipeline ensures code quality through automated validation before reaching production.

### 5. What is the purpose of the EXPOSE instruction in a Dockerfile?

**Answer:**
EXPOSE documents which port the container listens on at runtime. It does NOT actually publish the port — you must use `-p` flag when running the container (e.g., `docker run -p 8080:8080`). It serves as documentation for developers and is used by some orchestration tools for automatic port mapping.

### 6. How does Docker caching improve build performance?

**Answer:**
Docker caches each layer created by Dockerfile instructions. If an instruction hasn't changed since the last build, Docker reuses the cached layer. This is why you should copy dependency files (like csproj) first, run restore, then copy source code. Source code changes won't invalidate the cached restore layer, saving significant build time.

### 7. What is the role of the ASP.NET Core Module (ANCM) in IIS?

**Answer:**
ANCM is a native IIS module that bridges IIS and ASP.NET Core. In in-process hosting, it runs ASP.NET Core directly inside the IIS worker process (w3wp.exe). In out-of-process hosting, it forwards requests to a separate Kestrel process. ANCM manages process lifecycle — start, stop, and restart of the ASP.NET Core application.

### 8. Why must IIS Application Pool be set to "No Managed Code" for ASP.NET Core?

**Answer:**
ASP.NET Core runs on its own .NET runtime, not the .NET Framework CLR. Setting the Application Pool to "No Managed Code" tells IIS not to load the CLR, since it's not needed. If you load the CLR unnecessarily, it wastes memory and can cause compatibility issues. The ASP.NET Core application runs its own runtime process, managed by ANCM.

### 9. What is the difference between Continuous Delivery and Continuous Deployment?

**Answer:**
Both ensure code is always in a deployable state. **Continuous Delivery** means every change passes through the pipeline and is ready to deploy, but a human must approve production releases. **Continuous Deployment** goes further — every change that passes all automated stages is deployed to production automatically with no manual intervention.

### 10. What are deployment gates in Azure DevOps?

**Answer:**
Deployment gates are automated checks that must pass before a deployment proceeds. They can include Azure Monitor alerts, custom APIs, and health checks. Gates poll at defined intervals and allow or block deployment based on configured conditions. This ensures production stability without manual approval for every deployment.

### 11. How do you handle secrets in a CI/CD pipeline?

**Answer:**
Never hardcode secrets in source code or pipeline YAML. Use Azure DevOps variable groups marked as secret, Azure Key Vault integration, or pipeline secret variables. Secrets are masked in logs. In Docker, pass secrets as environment variables at runtime or use Docker secrets with Swarm/Kubernetes. Never store secrets in Docker image layers.

### 12. Explain the difference between framework-dependent and self-contained deployments.

**Answer:**
**Framework-dependent** deployments require the .NET runtime to be installed on the target machine. They produce smaller output because the runtime is shared. **Self-contained** deployments include the .NET runtime with the application. They produce larger output but can run on machines without .NET installed. Self-contained is better for isolated environments; framework-dependent is better for controlled environments where you manage the runtime.

### 13. What is the purpose of docker-compose and when would you use it?

**Answer:**
Docker Compose defines and runs multi-container applications from a single YAML file. Use it when your application needs multiple services running together (web app + database + cache). It manages networking between containers, volume mounts, environment variables, and startup order. It's especially useful in development to quickly spin up a complete environment with one command.

### 14. How do you optimize a Docker image for production?

**Answer:**
- Use multi-stage builds to exclude build tools from the final image
- Use the slim runtime base image (aspnet:8.0 instead of sdk:8.0)
- Order Dockerfile instructions to maximize cache hits (dependencies before source)
- Combine RUN commands to reduce the number of layers
- Use .dockerignore to exclude unnecessary files
- Use distroless or Alpine-based images for smaller attack surface
- Run as a non-USER for security

### 15. What is the difference between Microsoft-hosted and self-hosted build agents?

**Answer:**
**Microsoft-hosted** agents are maintained by Microsoft, available on-demand, and ephemeral (fresh VM per job). They support Windows, Linux, and macOS. **Self-hosted** agents run on infrastructure you manage — your own VMs, containers, or physical machines. Self-hosted agents are useful for private networks, custom tooling requirements, access to internal resources, or cost optimization for large workloads.

### 16. How do you implement database migrations in a CI/CD pipeline?

**Answer:**
Use `dotnet ef database update` in the pipeline after deployment, or generate SQL scripts with `dotnet ef migrations script` and apply them with a SQL task. For zero-downtime deployments, use migration strategies that support backward compatibility. Store migration scripts as artifacts and apply them in a dedicated deployment stage before the application deployment.

### 17. What is the difference between InProcess and OutOfProcess hosting in IIS?

**Answer:**
**InProcess** hosting runs the ASP.NET Core application directly inside the IIS worker process (w3wp.exe). It has better performance because there's no inter-process communication. **OutOfProcess** hosting runs Kestrel as a separate process, with IIS acting as a reverse proxy. InProcess is the recommended hosting model for IIS. OutOfProcess is used when you need Kestrel features not available through IIS.

---

## Quick Reference

### Docker Commands

```bash
# Build
docker build -t myapp:1.0 .

# Run
docker run -d -p 8080:8080 --name myapp myapp:1.0

# Debug
docker exec -it myapp /bin/bash

# Logs
docker logs -f myapp

# Stop and remove
docker stop myapp && docker rm myapp

# Clean up
docker system prune -a
```

### Azure DevOps Pipeline Tasks

```yaml
# Common tasks for .NET CI/CD
DotNetCoreCLI@2        # dotnet CLI operations
PublishBuildArtifacts@1 # Publish pipeline artifacts
DownloadBuildArtifact@1 # Download pipeline artifacts
PublishTestResults@2   # Publish test results
PublishCodeCoverageResults@2 # Publish code coverage
AzureWebApp@1         # Deploy to Azure App Service
NuGetCommand@2        # NuGet operations
UseDotNet@2           # Install .NET SDK
Cache@2               # Cache pipeline dependencies
```

### IIS Checklist for ASP.NET Core

```plaintext
1. Install ASP.NET Core Hosting Bundle
2. Application Pool set to "No Managed Code"
3. web.config includes AspNetCoreModuleV2 handler
4. stdoutLogEnabled=true for debugging
5. App pool identity has read access to app folder
6. IIS restarted after hosting bundle install (iisreset)
7. HTTPS binding configured with valid SSL certificate
8. Firewall allows traffic on configured ports
```
