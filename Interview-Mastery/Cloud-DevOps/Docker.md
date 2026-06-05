# Docker Study Guide

## 1. Executive Summary

Docker is a containerization platform that enables developers to package applications and their dependencies into lightweight, portable containers. Containers share the host OS kernel but run in isolated user spaces, providing consistency across development, staging, and production environments. Docker revolutionized software delivery by solving the "it works on my machine" problem and is foundational to modern DevOps practices, microservices architectures, and cloud-native development.

## 2. Core Theory

### 2.1 Container vs Virtual Machine

Containers virtualize the OS, while VMs virtualize hardware. Containers share the host kernel, making them far more lightweight than VMs, which each include a full guest OS.

| Feature | Container | Virtual Machine |
|---------|-----------|-----------------|
| Startup | Milliseconds | Minutes |
| Size | MBs | GBs |
| Kernel | Shared with host | Dedicated per VM |
| Isolation | Process-level | Hardware-level |

### 2.2 Docker Architecture

Docker uses a client-server architecture:

- **Docker Client**: CLI tool that communicates with the Docker daemon
- **Docker Daemon (dockerd)**: Background service that manages containers, images, networks, and storage volumes
- **Docker Registry**: Stores Docker images (public: Docker Hub, private: custom registry)
- **Docker Objects**: Images, containers, networks, volumes, plugins

```
+-------------------+        REST API        +--------------------+
|  Docker Client    | ---------------------> |  Docker Daemon     |
|  (docker CLI)     | <--------------------- |  (dockerd)         |
+-------------------+                        +--------------------+
                                                     |
                                          +----------+----------+
                                          |                     |
                                   +-----------+        +-----------+
                                   | Images    |        | Containers|
                                   +-----------+        +-----------+
                                   | Registry  |        | Volumes   |
                                   +-----------+        +-----------+
```

### 2.3 Key Components

- **Dockerfile**: A text file with instructions to build an image
- **Image**: A read-only template with instructions for creating a container
- **Container**: A runnable instance of an image
- **Volume**: Persistent data storage outside the container's filesystem
- **Network**: Communication pathway between containers
- **Compose**: Tool for defining multi-container applications

### 2.4 Container Lifecycle

```
Created -> Running -> Paused -> Unpaused -> Running -> Stopped -> Removed
                     \-> Stopped --------/
                     \-> Removed (if --rm)
```

## 3. Under-the-Hood Deep Dive

### 3.1 Namespaces

Linux namespaces provide isolation for containers. Docker uses several:

- **PID namespace**: Process isolation
- **Network namespace**: Network interfaces and routing tables
- **Mount namespace**: Filesystem mount points
- **UTS namespace**: Hostname and domain name
- **IPC namespace**: Inter-process communication resources
- **User namespace**: User and group IDs

### 3.2 Cgroups (Control Groups)

Cgroups limit and account for resource usage (CPU, memory, disk I/O, network). Docker uses cgroups to enforce resource constraints specified via `--cpus`, `--memory`, etc.

```yaml
# Docker Compose resource limits
services:
  app:
    image: myapp:latest
    deploy:
      resources:
        limits:
          cpus: '0.5'
          memory: 512M
        reservations:
          cpus: '0.25'
          memory: 256M
```

### 3.3 Union Filesystems

Docker uses union filesystems (OverlayFS, AUFS) to layer images. Each Dockerfile instruction creates a new layer. Layers are cached and reused across builds, reducing storage and speeding up builds.

```
Container Layer (read-write)
  |
  +-- Image Layer 3 (read-only) - CMD instruction
  |
  +-- Image Layer 2 (read-only) - RUN apt-get install
  |
  +-- Image Layer 1 (read-only) - FROM ubuntu:22.04
```

### 3.4 Copy-on-Write (CoW)

When a container modifies a file from an image layer, the file is copied to the container's writable layer first, then modified. The original image layer remains unchanged.

## 4. Production Code Examples

### 4.1 Multi-stage Dockerfile

```dockerfile
# Build stage
FROM golang:1.21-alpine AS builder
WORKDIR /app
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 GOOS=linux go build -o /app/server .

# Runtime stage
FROM alpine:3.19
RUN apk --no-cache add ca-certificates tzdata
WORKDIR /app
COPY --from=builder /app/server .
COPY --from=builder /app/static ./static
EXPOSE 8080
USER 1001:1001
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
  CMD wget --no-verbose --tries=1 --spider http://localhost:8080/health || exit 1
ENTRYPOINT ["./server"]
```

### 4.2 Docker Compose for Microservices

```yaml
version: '3.8'

services:
  api-gateway:
    build: ./gateway
    ports:
      - "8080:8080"
    environment:
      - REDIS_URL=redis://redis:6379
      - AUTH_SVC_URL=http://auth:5001
    depends_on:
      - redis
      - auth
    restart: unless-stopped
    logging:
      driver: json-file
      options:
        max-size: "10m"
        max-file: "3"

  auth:
    build: ./auth-service
    expose:
      - "5001"
    environment:
      - DB_URL=postgres://user:pass@db:5432/auth
    depends_on:
      db:
        condition: service_healthy

  redis:
    image: redis:7-alpine
    volumes:
      - redis-data:/data
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 5s

  db:
    image: postgres:16-alpine
    volumes:
      - postgres-data:/var/lib/postgresql/data
    environment:
      POSTGRES_PASSWORD_FILE: /run/secrets/db_password
    secrets:
      - db_password
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 10s

volumes:
  redis-data:
  postgres-data:

secrets:
  db_password:
    file: ./secrets/db_password.txt
```

### 4.3 Dockerfile Best Practices

```dockerfile
# Use specific tags, not latest
FROM python:3.12-slim-bookworm AS builder

# Set working directory
WORKDIR /app

# Install system dependencies
RUN apt-get update && apt-get install -y --no-install-recommends \
    gcc \
    libpq-dev \
    && rm -rf /var/lib/apt/lists/*

# Copy requirements first for layer caching
COPY requirements.txt .
RUN pip install --no-cache-dir --user -r requirements.txt

# Copy application code
COPY . .

# Multi-stage for production
FROM python:3.12-slim-bookworm
WORKDIR /app
COPY --from=builder /root/.local /root/.local
COPY --from=builder /app .

# Security best practices
RUN addgroup --system --gid 1001 appgroup && \
    adduser --system --uid 1001 appuser --ingroup appgroup
USER appuser

ENV PATH=/root/.local/bin:$PATH
EXPOSE 8000

HEALTHCHECK --interval=30s --timeout=10s --start-period=5s --retries=3 \
    CMD python -c "import urllib.request; urllib.request.urlopen('http://localhost:8000/health')" || exit 1

CMD ["gunicorn", "app:app", "--bind", "0.0.0.0:8000", "--workers", "4"]
```

### 4.4 Docker Network Setup

```yaml
version: '3.8'

networks:
  frontend:
    driver: overlay
    ipam:
      config:
        - subnet: 10.0.1.0/24
  backend:
    driver: overlay
    internal: true
    ipam:
      config:
        - subnet: 10.0.2.0/24

services:
  web:
    networks:
      - frontend
      - backend
  api:
    networks:
      - backend
  db:
    networks:
      backend:
        aliases:
          - database.internal
```

### 4.5 Docker Swarm Stack

```yaml
version: '3.8'

services:
  app:
    image: myapp:latest
    deploy:
      replicas: 3
      update_config:
        parallelism: 1
        delay: 10s
        order: start-first
      rollback_config:
        parallelism: 0
        order: stop-first
      restart_policy:
        condition: any
        delay: 5s
        max_attempts: 3
        window: 120s
      placement:
        constraints:
          - node.role == worker
          - node.labels.env == production
      resources:
        limits:
          cpus: '1'
          memory: 1G
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost/health"]
      interval: 30s
      timeout: 10s
      retries: 3
```

## 5. Real-World Scenarios

### 5.1 Blue-Green Deployment with Docker

Blue-green deployments use two identical environments (blue = current, green = new). Traffic is switched atomically.

```bash
#!/bin/bash
# deploy.sh - Blue-Green Deployment

BLUE_PORT=8081
GREEN_PORT=8082
NGINX_CONF=/etc/nginx/sites-available/app

deploy() {
  local TAG=$1
  local NEW_ENV

  # Check which environment is current
  if curl -sf http://localhost:$BLUE_PORT/health > /dev/null 2>&1; then
    NEW_ENV="green"
    NEW_PORT=$GREEN_PORT
    OLD_PORT=$BLUE_PORT
  else
    NEW_ENV="blue"
    NEW_PORT=$BLUE_PORT
    OLD_PORT=$GREEN_PORT
  fi

  echo "Deploying to $NEW_ENV (port $NEW_PORT)..."

  # Start new containers
  docker-compose -f docker-compose.$NEW_ENV.yml up -d --build

  # Wait for health check
  for i in {1..30}; do
    if curl -sf http://localhost:$NEW_PORT/health > /dev/null 2>&1; then
      echo "$NEW_ENV is healthy after ${i}s"
      break
    fi
    sleep 1
  done

  # Switch traffic
  sed -i "s/proxy_pass http:\/\/localhost:$OLD_PORT/proxy_pass http:\/\/localhost:$NEW_PORT/" $NGINX_CONF
  nginx -s reload

  # Stop old containers
  docker-compose -f docker-compose.$NEW_ENV.yml down

  echo "Deployment complete. Active: $NEW_ENV"
}

deploy $1
```

### 5.2 Zero-Downtime Database Migration

```bash
#!/bin/bash
# migrate-db.sh

OLD_TAG=$1
NEW_TAG=$2
MIGRATION_IMAGE="myapp/migrations"
DB_URL=$3

# Run migration in a temporary container
docker run --rm \
  --network=app_network \
  -e DB_URL=$DB_URL \
  $MIGRATION_IMAGE:$NEW_TAG \
  migrate up

# Verify migration
docker run --rm \
  --network=app_network \
  -e DB_URL=$DB_URL \
  $MIGRATION_IMAGE:$NEW_TAG \
  migrate verify

# Rolling update of application containers
docker service update \
  --image myapp/app:$NEW_TAG \
  --update-parallelism 2 \
  --update-delay 10s \
  --health-cmd "curl -f http://localhost/health" \
  --health-interval 5s \
  --health-retries 3 \
  app_service
```

### 5.3 Log Aggregation

```yaml
version: '3.8'

services:
  app:
    image: myapp:latest
    logging:
      driver: fluentd
      options:
        fluentd-address: "fluentd:24224"
        tag: "app.{{.Name}}"

  fluentd:
    image: fluent/fluentd:v1.16
    volumes:
      - ./fluentd/conf:/fluentd/etc
      - ./fluentd/buffer:/fluentd/log
    ports:
      - "24224:24224"
      - "24224:24224/udp"

  elasticsearch:
    image: elasticsearch:8.11
    environment:
      - discovery.type=single-node
      - xpack.security.enabled=false
    volumes:
      - es-data:/usr/share/elasticsearch/data

  kibana:
    image: kibana:8.11
    ports:
      - "5601:5601"
    depends_on:
      - elasticsearch
```

## 6. Performance

### 6.1 Optimization Techniques

- **Layer caching**: Order Dockerfile instructions from least to most frequently changing
- **Multi-stage builds**: Separate build and runtime dependencies
- **Minimal base images**: Use alpine or distroless images
- **.dockerignore**: Exclude unnecessary files from build context
- **Resource limits**: Set CPU/memory constraints to prevent noisy neighbors

### 6.2 Image Size Comparison

```dockerfile
# Bad: 1.2GB image
FROM ubuntu:22.04
RUN apt-get update && apt-get install -y python3 python3-pip nodejs npm
COPY . .
RUN pip install -r requirements.txt && npm install

# Good: 150MB image  
FROM node:20-alpine AS node-build
WORKDIR /app
COPY package*.json .
RUN npm ci --only=production

FROM python:3.12-alpine
WORKDIR /app
COPY --from=node-build /app/node_modules ./node_modules
COPY . .
RUN pip install --no-cache-dir -r requirements.txt

# Best: 50MB image
FROM python:3.12-alpine
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY . .
```

### 6.3 Docker Benchmark Commands

```bash
# Test container performance
docker run --rm alpine time seq 1000000 > /dev/null

# Benchmark disk I/O
docker run --rm alpine dd if=/dev/zero of=/tmp/test bs=1M count=100

# Monitor resource usage
docker stats --no-stream --format "table {{.Name}}\t{{.CPUPerc}}\t{{.MemUsage}}"

# Inspect container resource usage
docker inspect -f '{{.HostConfig.Memory}}' container_name
```

## 7. Security

### 7.1 Docker Security Best Practices

```dockerfile
# Do NOT run as root
RUN addgroup -S appgroup && adduser -S appuser -G appgroup
USER appuser

# Use read-only root filesystem
# In docker-compose: read_only: true
```

- **Don't expose the Docker socket** unless absolutely necessary
- **Use secrets management**, not environment variables for sensitive data
- **Scan images** for vulnerabilities: `docker scan` or `trivy image`
- **Sign and verify images** with Docker Content Trust (DCT)
- **Use non-root users** inside containers
- **Limit capabilities** with `--cap-drop=ALL --cap-add=NEEDED`

```yaml
# Secure Docker Compose
services:
  app:
    image: myapp:latest
    user: 1001:1001
    read_only: true
    tmpfs:
      - /tmp:noexec,nosuid,size=64M
    cap_drop:
      - ALL
    cap_add:
      - NET_BIND_SERVICE
    security_opt:
      - no-new-privileges:true
    secrets:
      - api_key
```

### 7.2 Image Vulnerability Scanning

```bash
# Using Trivy
trivy image myapp:latest

# Using Docker Scout
docker scout quickview myapp:latest
docker scout recommendations myapp:latest

# Using Snyk
snyk container test myapp:latest --severity-threshold=high
```

## 8. Common Mistakes

| Mistake | Solution |
|---------|----------|
| Using `latest` tag | Pin specific versions |
| Running as root | Create and use non-root user |
| Hardcoding secrets | Use Docker secrets or external vault |
| Ignoring .dockerignore | Always exclude node_modules, .git, etc. |
| Single-stage builds | Use multi-stage builds |
| No healthcheck | Always define HEALTHCHECK |
| Storing data in containers | Use volumes |
| Default network driver (bridge) | Specify network in production |

```bash
# Bad: using latest and running as root
docker run -d -p 80:80 nginx:latest

# Good: specific tag, explicit user
docker run -d -p 8080:80 --user nginx nginx:1.25-alpine
```

## 9. Senior Engineer Perspective

### 9.1 Production Readiness Checklist

1. Image scanning integrated into CI/CD pipeline
2. Resource limits set on all containers
3. Health checks configured and tested
4. Logging shipped to centralized system
5. Metrics exported (Prometheus endpoints)
6. Graceful shutdown handling (SIGTERM)
7. Read-only filesystem where possible
8. Security scanning for base images
9. Rolling update strategy documented
10. Disaster recovery plan tested

### 9.2 Container Orchestration Decision

```yaml
# Docker Swarm (simpler, tighter Docker integration)
deploy:
  replicas: 5
  update_config:
    parallelism: 2
    delay: 30s

# Kubernetes (more features, complex)
# apiVersion: apps/v1
# kind: Deployment
# spec:
#   replicas: 5
#   strategy:
#     rollingUpdate:
#       maxUnavailable: 1
#       maxSurge: 1
```

### 9.3 Migration Strategy: Bare Metal to Containers

1. **Lift and shift**: Package existing apps into containers without modification
2. **Refactor**: Break monoliths into microservices
3. **Optimize**: Implement infrastructure as code, auto-scaling, service mesh
4. **Day-2 operations**: Monitoring, logging, alerting, backup/restore

## 10. Interview Questions (Easy)

1. What is Docker and what problem does it solve?
2. What is the difference between a Docker image and a container?
3. Explain the Dockerfile instructions: FROM, RUN, CMD, ENTRYPOINT.
4. What is Docker Compose used for?
5. How do you persist data in Docker containers?
6. What is the difference between CMD and ENTRYPOINT?
7. How do you expose ports in Docker?
8. What is the purpose of a .dockerignore file?
9. How do you view logs from a Docker container?
10. What is Docker Hub?

## 10. Interview Questions (Medium)

11. Explain the Docker container lifecycle.
12. How do Docker layers work and why are they important?
13. What is the difference between COPY and ADD in a Dockerfile?
14. How do you handle inter-container communication?
15. Explain Docker networking modes (bridge, host, overlay).
16. What are Docker volumes and how do they differ from bind mounts?
17. How do you limit resources (CPU, memory) for a container?
18. What is multi-stage build and when should you use it?
19. How do you debug a container that exits immediately?
20. Explain the concept of Docker health checks.

## 11. Advanced Interview Questions (Hard)

1. How do Linux namespaces and cgroups enable containerization?
2. Design a zero-downtime deployment strategy using Docker Swarm.
3. How would you secure a Dockerized application in production?
4. Explain the copy-on-write mechanism in Docker's union filesystem.
5. How do you implement a blue-green deployment with Docker?
6. Design a logging and monitoring strategy for 100+ containers.
7. How does Docker's overlay network driver work?
8. Explain the differences between Docker Swarm and Kubernetes.
9. How do you handle database migrations in containerized environments?
10. What are the trade-offs between fat containers and slim containers?

## 11. Advanced Interview Questions (System Design)

11. Design a CI/CD pipeline using Docker for a microservices architecture.
12. Design a multi-tenant container platform for SaaS.
13. Design a container-based disaster recovery solution.
14. Design a scalable logging pipeline for containerized applications.
15. Design a secrets management strategy for containers.
16. Design a container image promotion strategy across environments.
17. Design a container security scanning pipeline.
18. Design a hybrid cloud container strategy.
19. Design a container monitoring solution for 1000+ containers.
20. Design a cost-optimized container deployment on spot instances.

## 12. Expert-Level Interview Questions (Architect)

1. Design a global multi-region container platform serving millions of users. Consider service discovery, data locality, failover, and cost optimization.

2. Architect a platform that runs 10,000+ containerized workloads with varying security postures (PCI, HIPAA, SOC2) on shared infrastructure.

3. Design a container-native CI/CD system that handles monorepo with 200+ microservices, supporting canary deployments, feature flags, and automatic rollbacks.

4. How would you design a container networking solution that provides service mesh capabilities (mTLS, traffic shifting, circuit breaking) without sidecar proxies?

5. Architect a hybrid cloud container platform spanning on-premise and 3 public clouds with unified management, security policies, and disaster recovery.

6. Design a container-based data platform supporting streaming, batch processing, and ML training workloads with dynamic resource allocation.

7. How would you design a container runtime that achieves near-native performance for HPC workloads while maintaining Docker compatibility?

8. Architect a multi-cloud container image registry serving 10,000+ developers with geo-replication, vulnerability scanning, and supply chain attestation.

9. Design a container platform that supports both Windows and Linux containers across on-premise and cloud with unified orchestration.

10. How would you design a container billing and chargeback system for a platform running 50,000+ containers across 10 teams with different SLAs?

## 13. Debugging & Troubleshooting

### 13.1 Common Container Issues

```bash
# Container exits immediately
docker logs <container>  # Check logs
docker inspect <container>  # Check exit code and config

# Container runs but unhealthy
docker exec -it <container> /bin/sh  # Shell into container
docker stats <container>  # Check resource usage

# Network issues
docker network inspect <network>
docker exec <container> ping <other-container>

# Disk space issues
docker system df  # Show disk usage
docker container prune  # Remove stopped containers
docker image prune  # Remove unused images
docker system prune -a --volumes  # Full cleanup (careful)

# Performance issues
docker stats --no-stream  # CPU/Memory per container
docker top <container>  # Processes in container
```

### 13.2 Debugging Docker Builds

```bash
# Debug build failures
docker build --progress=plain .  # Show full build output
docker build --no-cache .  # Force rebuild

# Intermediate containers
docker build --target builder .  # Build to specific stage
docker run -it <intermediate-image> /bin/sh  # Debug stage

# Inspect layers
docker history <image>  # Show layer history
docker image inspect <image>  # Detailed image info
```

### 13.3 Recovery Scenarios

```bash
# Docker daemon not responding
sudo systemctl restart docker
# Check daemon logs: journalctl -u docker.service

# Corrupted image
docker rmi <image> && docker pull <image>

# Orphaned volumes
docker volume ls -qf dangling=true | xargs docker volume rm

# Docker socket permission denied
sudo usermod -aG docker $USER
newgrp docker  # or log out and back in
```

## 14. Comparison Section

### Docker vs Podman

| Feature | Docker | Podman |
|---------|--------|--------|
| Architecture | Client-Daemon | Daemonless (fork/exec) |
| Root required | Daemon runs as root | Rootless by default |
| Kubernetes | Mature (k8s, Swarm) | Pods compatible with k8s |
| Build | Dockerfile | Dockerfile + Buildah |
| Registry | Docker Hub | Any OCI-compliant |

### Docker vs Containerd

| Feature | Docker | Containerd |
|---------|--------|------------|
| Scope | Full platform | Core container runtime |
| Image management | Yes | Yes |
| Networking | Full stack | Basic (via CNI) |
| Volume management | Yes | Basic |
| CLI | docker CLI | ctr, nerdctl |

### Docker Swarm vs Kubernetes

| Feature | Swarm | Kubernetes |
|---------|-------|------------|
| Setup Complexity | Simple | Complex |
| Scaling | Automatic | Manual (HPA) |
| Service Discovery | Built-in | DNS + Services |
| Load Balancing | Built-in | Ingress + Services |
| Stateful Apps | Limited | StatefulSets |
| Community | Small | Large |
| Cloud Support | Limited | Everywhere |

## 15. Revision Notes

### Quick Recap: Docker Commands

```bash
# Images
docker build -t name:tag .
docker images
docker pull alpine:3.19
docker push repo/image:tag
docker rmi <image-id>
docker tag source:tag target:tag

# Containers
docker run -d --name web -p 80:8000 nginx:alpine
docker ps [-a]
docker stop|start|restart <container>
docker rm -f <container>
docker exec -it <container> /bin/sh
docker logs -f <container>
docker cp <container>:/path ./local-path

# System
docker system df
docker system prune -a
docker info

# Compose
docker-compose up -d
docker-compose down
docker-compose logs -f
docker-compose ps
docker-compose build
```

### Key Dockerfile Instructions

| Instruction | Purpose |
|-------------|---------|
| FROM | Base image |
| WORKDIR | Set working directory |
| COPY | Copy files from context |
| RUN | Execute commands during build |
| ENV | Set environment variables |
| EXPOSE | Document port |
| CMD | Default command |
| ENTRYPOINT | Executable entry point |
| HEALTHCHECK | Container health check |
| USER | Set runtime user |
| VOLUME | Create mount point |
| ARG | Build-time variable |
| LABEL | Metadata |
| ONBUILD | Trigger for downstream builds |
| STOPSIGNAL | Shutdown signal |

## 16. Cheat Sheet

```text
+======================================================================+
|                        DOCKER CHEAT SHEET                             |
+======================================================================+

  BUILD & PUSH
+----------------------------------------------------------------------+
| docker build -t name:tag .      Build image from Dockerfile          |
| docker build --no-cache .       Force rebuild without cache          |
| docker tag src:tag dest:tag     Tag an image                         |
| docker push repo/image:tag      Push image to registry               |
| docker pull repo/image:tag      Pull image from registry             |
+----------------------------------------------------------------------+

  RUN & MANAGE
+----------------------------------------------------------------------+
| docker run -d --name web -p 80:80 nginx     Run container detached   |
| docker ps                    List running containers                 |
| docker ps -a                 List all containers                     |
| docker stop/start/restart    Control container state                 |
| docker rm CONTAINER          Remove container                        |
| docker rm -f CONTAINER       Force remove running container          |
| docker exec -it CONTAINER sh Shell into running container            |
| docker logs -f CONTAINER     Follow container logs                   |
+----------------------------------------------------------------------+

  NETWORKING
+----------------------------------------------------------------------+
| docker network ls             List networks                          |
| docker network create mynet   Create network                         |
| docker network connect net cont  Connect container to network        |
| docker run --network=host     Use host networking                    |
| docker port CONTAINER         Show port mappings                     |
+----------------------------------------------------------------------+

  VOLUMES & DATA
+----------------------------------------------------------------------+
| docker volume create myvol    Create a volume                        |
| docker volume ls              List volumes                           |
| docker run -v myvol:/data     Mount volume                           |
| docker run -v /host:/cont     Bind mount                             |
| docker cp CONTAINER:/path .   Copy file from container               |
+----------------------------------------------------------------------+

  SYSTEM & CLEANUP
+----------------------------------------------------------------------+
| docker system df              Show disk usage                        |
| docker container prune        Remove stopped containers              |
| docker image prune            Remove dangling images                 |
| docker volume prune           Remove unused volumes                  |
| docker system prune -a        Remove everything unused               |
+----------------------------------------------------------------------+

  DOCKER COMPOSE
+----------------------------------------------------------------------+
| docker-compose up -d          Start all services                     |
| docker-compose down           Stop and remove containers             |
| docker-compose logs -f        Follow logs for all services           |
| docker-compose build          Rebuild images                         |
| docker-compose ps             List service status                    |
| docker-compose exec svc sh    Execute command in service             |
| docker-compose restart        Restart all services                   |
+----------------------------------------------------------------------+

  HEALTH & DEBUG
+----------------------------------------------------------------------+
| docker stats                  Live resource usage per container      |
| docker inspect CONTAINER      Detailed container metadata            |
| docker top CONTAINER          Running processes in container         |
| docker events                 Stream Docker events                   |
| docker image history IMAGE    Layer history of image                 |
+----------------------------------------------------------------------+

+======================================================================+
|  PRO TIPS:                                                           |
|  - Always pin image versions (never :latest)                         |
|  - Use multi-stage builds for smaller images                         |
|  - Use .dockerignore to exclude files from build context             |
|  - Set resource limits in production                                 |
|  - Never run containers as root                                      |
|  - Always define HEALTHCHECK                                         |
|  - Use COPY not ADD (unless extracting tar)                          |
+======================================================================+
```
