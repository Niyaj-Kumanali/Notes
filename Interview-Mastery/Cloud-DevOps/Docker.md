# Docker

---

## Overview

- **Definition:** Docker is a containerization platform that packages applications and dependencies into lightweight, portable containers sharing the host OS kernel.
- **Why It Exists:** Docker solves the "it works on my machine" problem by ensuring consistency across environments. Containers are faster and more resource-efficient than VMs — startup in milliseconds, size in MBs versus GBs.
- **Key Concepts:** **Image** (read-only template), **Container** (runnable image instance), **Dockerfile** (build instructions), **Volume** (persistent data), **Network** (container communication), **Compose** (multi-container apps), **Registry** (image storage — Docker Hub, ECR, ACR)

---

## Core Concepts

- **Container vs VM:** Containers share the host OS kernel (process isolation via namespaces), VMs have dedicated guest OS (hardware isolation). Containers start in milliseconds, VMs in minutes.
- **Namespaces:** Linux namespaces provide isolation — PID (processes), Network (interfaces), Mount (filesystem), UTS (hostname), IPC (inter-process communication), User (user/group IDs).
- **Cgroups:** Control Groups limit resource usage (CPU, memory, disk I/O). Use `--cpus`, `--memory` flags or Compose `deploy.resources` limits.
- **Union Filesystem:** Docker uses OverlayFS to layer images. Each Dockerfile instruction creates a read-only layer. The container adds a thin read-write layer on top. Layers are cached and reused across builds.
- **Copy-on-Write (CoW):** When a container modifies a file from an image layer, Docker copies it to the writable layer first, then modifies it. Original layers remain unchanged.

```dockerfile
# Multi-stage build: separate build and runtime
FROM golang:1.21-alpine AS builder
WORKDIR /app
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 go build -o /app/server .

FROM alpine:3.19
RUN apk --no-cache add ca-certificates tzdata
COPY --from=builder /app/server .
USER 1001:1001
HEALTHCHECK --interval=30s --timeout=3s --retries=3 \
  CMD wget --spider http://localhost:8080/health || exit 1
ENTRYPOINT ["./server"]
```

```yaml
# Docker Compose with resource limits
services:
  app:
    image: myapp:latest
    ports:
      - "8080:8080"
    deploy:
      resources:
        limits:
          cpus: '0.5'
          memory: 512M
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost/health"]
    volumes:
      - app-data:/data
volumes:
  app-data:
```

---

## Common Mistakes

- **Using `latest` tag** — Unpredictable; pin specific versions
- **Running as root** — Security risk; create and use non-root user
- **Hardcoding secrets** — Leaks; use Docker secrets or external vault
- **No .dockerignore** — Large build context; exclude node_modules, .git, build artifacts
- **Single-stage builds** — Bloated images (1GB+); use multi-stage builds
- **No healthcheck** — Orchestrator can't detect failures; always define HEALTHCHECK
- **Storing data in containers** — Data loss on restart; use named volumes
- **No resource limits** — Noisy neighbor; set CPU/memory limits in production
- **Exposing Docker socket** — Security risk; mount only when absolutely necessary

---

## Key Design Considerations

- **Dockerfile Best Practices** — Use specific base images (not `latest`), order FROM first then system deps then app deps then code (leverage layer caching), multi-stage builds, non-root user, HEALTHCHECK, .dockerignore
- **Image Size Optimization** — Prefer Alpine or distroless base images. Multi-stage builds separate build tools from runtime binaries. Example: Go app from 1.2GB to 20MB. Minimize layers and clean up package managers.
- **Networking** — Bridge (default, isolated per host), Host (shared with host, no isolation), Overlay (multi-host, Docker Swarm/K8s). Use custom networks for service isolation. Internal networks for backend services.
- **Data Persistence** — Volumes (managed by Docker, `/var/lib/docker/volumes`), Bind mounts (host directory into container), tmpfs (in-memory for temp data). Use volumes for databases, bind mounts for config files.
- **Security** — Non-root user (`USER 1001`), drop all capabilities (`--cap-drop=ALL`) and add only needed, read-only root filesystem (`read_only: true`), resource limits, image scanning with Trivy/Docker Scout, signed images with Docker Content Trust
- **Production Readiness** — Resource limits on all containers, health checks, centralized logging (fluentd, ELK), metrics (Prometheus endpoints), graceful shutdown (SIGTERM handler), rolling update strategy

---

## Real-World Scenarios

**Scenario 1: Reducing Image Size from 1.2GB to 35MB for a Go Microservice**
A team deploys a Go microservice but the Docker image is 1.2GB because it uses `golang:latest` as the runtime base. Fix: Use multi-stage builds — build in `golang:1.21-alpine`, copy only the binary to `alpine:3.19` or `gcr.io/distroless/base`. Result: 35MB image, 97% reduction. Startup time drops from 8s to 1s. Deployment time drops from 45s to 5s. Smaller attack surface (no shell, no package manager, no compiler).

```dockerfile
# Before: 1.2GB
FROM golang:latest
WORKDIR /app
COPY . .
RUN go build -o server .
CMD ["./server"]

# After: 35MB
FROM golang:1.21-alpine AS builder
WORKDIR /app
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 go build -o server .

FROM alpine:3.19
RUN apk --no-cache add ca-certificates tzdata
COPY --from=builder /app/server .
USER 1001:1001
CMD ["./server"]
```

**Scenario 2: Containerized CI/CD Pipeline with Layer Caching**
A development team's Docker builds take 8 minutes because every build rebuilds all layers. Fix: Optimize Dockerfile order to maximize layer caching. Use BuildKit with inline cache. Use Docker layer caching in CI. Use a dedicated builder instance with persistent cache. Result: builds drop to 90 seconds for code changes (npm install layer cached).

```dockerfile
# Optimized for layer caching
FROM node:20-alpine AS deps
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm ci --only=production  # Cached unless deps change

FROM node:20-alpine AS builder
WORKDIR /app
COPY --from=deps /app/node_modules ./node_modules
COPY . .
RUN npm run build

FROM node:20-alpine AS runner
WORKDIR /app
COPY --from=builder /app/dist ./dist
COPY --from=deps /app/node_modules ./node_modules
USER node
CMD ["node", "dist/server.js"]
```

**Scenario 3: Docker-in-Docker for Secure CI/CD**
A CI/CD pipeline needs to build Docker images but giving the agent full Docker socket access is a security risk. Fix: Use Docker-outside-of-Docker (DooD) — mount `/var/run/docker.sock` read-only, or better, use rootless Docker with `--privileged` in a dedicated VM agent. For Kubernetes runners, use kaniko or buildah for rootless container builds with no Docker daemon dependency.

---

## Scenario-Based Questions

1. **Q: Explain how Docker layers work and why they matter.**
   A: Each Dockerfile instruction creates a layer. Layers are cached — a rebuild only reruns instructions after the first change. Order FROM first, then system deps, then app deps, then code. This minimizes rebuild time. Layers are shared across images, saving disk space.

2. **Q: Design a zero-downtime deployment strategy using Docker.**
   A: Use blue-green deployment — two identical environments (blue/green). Deploy new version to inactive environment. Health check passes. Switch nginx/load balancer traffic. Keep old environment running for immediate rollback. Automate with Compose or Swarm.

3. **Q: How would you secure a Dockerized application in production?**
   A: Non-root user, read-only filesystem, drop all capabilities and add only needed, resource limits, image scanning in CI/CD, no Docker socket exposure, secrets via Docker secrets/Vault, network segmentation, signed images.

4. **Q: How does Docker's overlay network work?**
   A: Overlay networks enable multi-host communication. Uses VXLAN to encapsulate traffic. Each container gets a virtual IP. Built-in DNS for service discovery. Encrypted option available. Used by Docker Swarm for cross-host container communication.

5. **Q: What are the trade-offs between fat and slim containers?**
   A: Fat (1GB+): includes build tools, debugging utilities — larger attack surface, slower deploy, but easier debugging. Slim (alpine, distroless, multi-stage): smaller (5-50MB), faster deploy, smaller attack surface, but harder to debug. Prefer slim for production.

6. **Q: How do you handle database migrations in containerized environments?**
   A: Run migrations as a separate one-off container before app deployment. Use init containers in K8s or `depends_on` with health checks in Compose. Always have backward-compatible schema changes. Rollback migrations should exist.

7. **Q: Design a logging strategy for 100+ containers.**
   A: Use Docker's logging drivers (fluentd, awslogs, gelf). Ship logs to centralized system (ELK, Loki, CloudWatch). Parse structured logs (JSON format). Add container metadata (name, service, env) as log fields. Avoid `docker logs` for production.

8. **Q: Explain Docker Compose vs Docker Swarm vs Kubernetes.**
   A: Compose — single-host, dev/test. Swarm — multi-host, Docker-native, simpler than K8s. Kubernetes — complex, feature-rich, auto-scaling, self-healing, service mesh, suited for production and large deployments.

9. **Q: How do you implement CI/CD for Docker images?**
   A: Build image with commit SHA tag, scan for vulnerabilities, push to registry, then deploy. Use multi-stage builds for smaller images. Cache Docker layers in CI. Use kaniko or buildkit for rootless builds. Sign images with Cosign.

10. **Q: How do you debug a container that exits immediately?**
    A: Check logs (`docker logs <container>`), inspect exit code (`docker inspect`), run with interactive shell instead of command (`docker run -it --entrypoint sh`), check resource limits, verify config/environment variables, test locally.


---

## Interview Questions

1. **What is the difference between an Image and a Container?**
   A: An image is a read-only template with instructions for creating a container (like a class in OOP). A container is a runnable instance of an image (like an object). Multiple containers can run from the same image. Images are built, containers are started.

2. **What is a multi-stage build and why use it?**
   A: A Dockerfile with multiple FROM statements. Each FROM starts a new stage. Build tools and dependencies in early stages, only copy runtime artifacts to the final stage. Keeps the final image small (no build tools, compilers, or source code). Example: build Go binary in golang image, copy to scratch image.

3. **What is the difference between CMD and ENTRYPOINT?**
   A: CMD provides default arguments (can be overridden by running `docker run image command`). ENTRYPOINT sets the executable that runs (cannot be overridden without `--entrypoint`). Best practice: use ENTRYPOINT for the main command, CMD for default arguments.

4. **What is a Docker volume and when would you use it?**
   A: A Docker volume is persistent storage managed by Docker, stored in `/var/lib/docker/volumes/`. Used for: (1) Database data (survives container restart). (2) Sharing data between containers. (3) Config files managed by Docker. Unlike bind mounts, volumes are fully managed by Docker and can be backed up with Docker commands.

5. **What is the difference between Docker Compose and Docker Swarm?**
   A: Compose defines multi-container apps in YAML, runs on a single host (dev/test). Swarm is a container orchestration platform for multi-host deployments with built-in service discovery, load balancing, and rolling updates. Compose files can be adapted for Swarm stacks.

6. **How do you debug a container that exits immediately?**
   A: (1) `docker logs <container>` for error messages. (2) `docker inspect <container>` for exit code and state. (3) Run with different entrypoint: `docker run -it --entrypoint sh <image>`. (4) Check resource limits (OOM kill?). (5) Test locally with interactive mode.

7. **What is a health check in Docker?**
   A: A command that Docker runs periodically to check if the container is healthy. Defined in Dockerfile: `HEALTHCHECK --interval=30s --timeout=3s --retries=3 CMD curl -f http://localhost/health`. Container status shows "healthy" or "unhealthy". Orchestrators use this for rolling updates and auto-recovery.

8. **What is Docker layer caching and how does it work?**
   A: Each Dockerfile instruction creates a cached layer. On rebuild, Docker reuses cached layers if the instruction and context haven't changed. Order instructions from least to most frequently changing (FROM -> system deps -> app deps -> code). Layers are also shared between images built on the same host.

9. **How do you secure the Docker supply chain?**
   A: (1) Use specific base image tags (not latest). (2) Scan images with Trivy/Snyk/Docker Scout. (3) Sign images with Docker Content Trust or Cosign. (4) Use multi-stage builds. (5) Run as non-root user. (6) Verify base image provenance (Docker Official Images or distroless). (7) Use a private registry with vulnerability scanning enabled.

10. **What is the difference between `expose` and `ports` in Docker Compose?**
   A: `expose` documents that the container listens on a port (no actual publishing) — only accessible within the same network. `ports` publishes the port to the host (`host:container` mapping), making it accessible from outside the container network. Use expose for internal service communication, ports for external access.

---

## Developer Recommendations

- **Pin base image versions over using `latest`** — `latest` changes unpredictably and can break your build. Pinning to `alpine:3.19` ensures reproducible builds. Use Dependabot/Renovate to automate version bumps. For security patches, rebuild regularly rather than moving to unpredictable latest tags.

- **Use multi-stage builds for every production image** — Including build tools in runtime images is a security risk and wastes bandwidth. Build stage has everything needed (compilers, dev dependencies). Runtime stage copies only the compiled binary or built assets. Typical reduction: 1.2GB to 50MB. Trade-off: longer build time initially, but dramatically faster deployments and smaller attack surface.

- **Run as non-root, drop capabilities, read-only filesystem** — Containers running as root are a massive security risk — if compromised, the attacker has root on the host. Use `USER 1001`, create the user in the Dockerfile. Add `--cap-drop=ALL --cap-add=NET_BIND_SERVICE`. Set `--read-only` root filesystem with tmpfs for temp data. This three-layer defense stops most container breakout attempts.

- **Use Dockerignore like .gitignore** — Without `.dockerignore`, your build context includes node_modules, .git, test files, CI configs, etc. This slows builds, invalidates cache (checksum of large context), and can leak secrets. At minimum exclude: `node_modules`, `.git`, `.env`, `Dockerfile`, `*.md`, `test/`.

- **Prefer volumes over bind mounts for production** — Bind mounts tie container to host filesystem structure, break on different hosts, and have permission issues. Volumes are portable, backup-friendly, and managed by Docker. Use bind mounts only for development hot-reloading. For databases, always use named volumes.

- **Health checks are not optional — they enable self-healing** — Without HEALTHCHECK, the orchestrator doesn't know if your app is truly healthy (running != working). Define a real application health endpoint. Check actual dependencies (DB connectivity, cache reachability). Use appropriate intervals (30s) and retries (3). This enables auto-recovery and zero-downtime deployments.

- **Use BuildKit for faster, more secure builds** — BuildKit (`DOCKER_BUILDKIT=1`) provides concurrent builds, better cache management, secrets mounting (`--secret`), SSH agent forwarding, and garbage collection. It's the default in Docker 23.0+. Enable it explicitly in CI and locally. Build times improve 30-50% for complex Dockerfiles.