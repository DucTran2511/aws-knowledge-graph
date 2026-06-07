---
tags: [concept, containers, docker, images, microservices, devops, compute]
aliases: [Docker, Docker Engine, Dockerfile, Docker Image, Docker Container, Container Runtime]
date: 2026-05-28
---

# Docker — Deep Dive

**Docker** is a platform for building, shipping, and running applications in **containers** — lightweight, portable, isolated environments that package code with all its dependencies. Docker is the foundation that powers AWS container services (ECS, EKS, Fargate).

> [!IMPORTANT]
> **Why this matters for SAA-C03:** While Docker itself isn't tested in depth, understanding how images, containers, and registries work is essential for answering ECS/ECR/EKS questions correctly. Task Definitions reference Docker images, ECR stores Docker images, and ECS runs Docker containers.

---

## Containers vs Virtual Machines

```
  Virtual Machines:                    Containers:

  ┌──────┐ ┌──────┐ ┌──────┐         ┌──────┐ ┌──────┐ ┌──────┐
  │App A │ │App B │ │App C │         │App A │ │App B │ │App C │
  ├──────┤ ├──────┤ ├──────┤         │+Libs │ │+Libs │ │+Libs │
  │Guest │ │Guest │ │Guest │         └──┬───┘ └──┬───┘ └──┬───┘
  │ OS   │ │ OS   │ │ OS   │            │        │        │
  └──┬───┘ └──┬───┘ └──┬───┘         ┌──▼────────▼────────▼──┐
     │        │        │              │   Docker Engine        │
  ┌──▼────────▼────────▼──┐          ├───────────────────────┤
  │     Hypervisor        │          │   Host OS (shared)    │
  ├───────────────────────┤          ├───────────────────────┤
  │     Host OS           │          │   Hardware            │
  ├───────────────────────┤          └───────────────────────┘
  │     Hardware          │
  └───────────────────────┘           ✅ Seconds to start
                                      ✅ MBs in size
  ❌ Minutes to start                 ✅ Shared OS kernel
  ❌ GBs in size                      ✅ Near-native performance
  ❌ Full OS per VM                   ✅ Higher density (more per host)
```

| Feature | Virtual Machine | Container |
|---|---|---|
| **Isolation** | Full hardware-level (hypervisor) | Process-level (OS kernel) |
| **Startup** | Minutes | Seconds |
| **Size** | GBs (includes full OS) | MBs (shares host kernel) |
| **Performance** | Slight overhead (hypervisor) | Near-native |
| **Density** | 10–20 VMs per host | 100s of containers per host |
| **OS** | Each VM has its own OS | All containers share host OS kernel |
| **Portability** | Less portable (heavy images) | Highly portable (lightweight images) |

> [!TIP]
> **Key distinction:** VMs virtualize **hardware** (each VM has its own OS kernel). Containers virtualize the **OS** (all containers share the host kernel). This is why containers are faster and lighter, but VMs provide stronger isolation.

---

## Docker Architecture

```
  ┌─────────────────────────────────────────────────────────────────┐
  │                        Docker Host                               │
  │                                                                  │
  │  ┌────────────┐     ┌──────────────────────────────────────┐    │
  │  │  Docker    │     │         Docker Daemon (dockerd)       │    │
  │  │  Client    │────►│                                      │    │
  │  │  (CLI)     │REST │  ┌──────────┐  ┌──────────┐         │    │
  │  │            │API  │  │Container │  │Container │         │    │
  │  │ docker run │     │  │    A     │  │    B     │         │    │
  │  │ docker build│    │  └──────────┘  └──────────┘         │    │
  │  │ docker pull│     │                                      │    │
  │  └────────────┘     │  ┌──────────┐  ┌──────────┐         │    │
  │                      │  │ Image A  │  │ Image B  │         │    │
  │                      │  └──────────┘  └──────────┘         │    │
  │                      └──────────────────────────────────────┘    │
  │                                      │                           │
  └──────────────────────────────────────│───────────────────────────┘
                                         │ docker pull / push
                                         ▼
                              ┌──────────────────────┐
                              │   Registry           │
                              │   (Docker Hub / ECR) │
                              │                      │
                              │  ┌──────┐ ┌──────┐   │
                              │  │img:v1│ │img:v2│   │
                              │  └──────┘ └──────┘   │
                              └──────────────────────┘
```

### Core Components

| Component | Description |
|---|---|
| **Docker Client** | CLI tool (`docker` commands). Sends REST API requests to the Docker Daemon. |
| **Docker Daemon** | Background process (`dockerd`) that manages images, containers, networks, and volumes. |
| **Docker Registry** | Storage for Docker images. **Docker Hub** (public default), **Amazon ECR** (AWS private/public). |
| **Docker Image** | Read-only template used to create containers. Built from a Dockerfile. |
| **Docker Container** | Running instance of an image. Isolated process with its own filesystem, network, and PID space. |

---

## Docker Images — Layered Architecture

Docker images are built in **layers**. Each instruction in a Dockerfile creates a new read-only layer. Layers are **cached and shared** between images, saving storage and build time.

```
  Dockerfile Instruction              Layer Stack
  ─────────────────────              ──────────────

  FROM ubuntu:22.04          ──►  ┌────────────────────────┐
                                   │ Layer 1: Ubuntu base   │ 200 MB
                                   │ (OS, apt, bash)        │
  RUN apt-get install python ──►  ├────────────────────────┤
                                   │ Layer 2: Python        │  50 MB
                                   │ (python3, pip)         │
  COPY requirements.txt .   ──►  ├────────────────────────┤
                                   │ Layer 3: requirements  │  1 KB
                                   │ (text file)            │
  RUN pip install -r req..  ──►  ├────────────────────────┤
                                   │ Layer 4: Dependencies  │ 100 MB
                                   │ (Flask, requests, etc) │
  COPY . /app               ──►  ├────────────────────────┤
                                   │ Layer 5: App code      │  5 MB
                                   │ (your source code)     │
  CMD ["python", "app.py"]  ──►  ├────────────────────────┤
                                   │ Layer 6: Metadata      │  0 KB
                                   │ (startup command)      │
                                   └────────────────────────┘

                                   Total Image: ~356 MB
                                   All layers are READ-ONLY

  When container runs:
                                   ┌────────────────────────┐
                                   │ Writable Layer (thin)  │ ← Container writes here
                                   ├────────────────────────┤
                                   │ Layer 6: CMD           │ ↑ Read-only
                                   │ Layer 5: App code      │ │
                                   │ Layer 4: Dependencies  │ │ Shared between
                                   │ Layer 3: requirements  │ │ all containers
                                   │ Layer 2: Python        │ │ from this image
                                   │ Layer 1: Ubuntu        │ │
                                   └────────────────────────┘
```

### Why Layers Matter

| Benefit | Description |
|---|---|
| **Caching** | Unchanged layers are cached — only modified layers are rebuilt. Put frequently changing content (app code) in later layers. |
| **Sharing** | Multiple images sharing the same base layer (e.g., `ubuntu:22.04`) store it only once on disk. |
| **Transfer** | When pushing/pulling, only new/changed layers are transferred — not the entire image. |
| **Copy-on-Write** | Running containers get a thin writable layer on top. The image layers remain read-only and shared. |

> [!TIP]
> **Build optimization:** Order Dockerfile instructions from **least frequently changed** (OS, dependencies) to **most frequently changed** (app code). This maximizes layer cache hits and dramatically speeds up builds.

---

## Dockerfile — Anatomy

A **Dockerfile** is a text file with instructions to build a Docker image.

```dockerfile
# ── Stage 1: Build ──────────────────────────────────
FROM node:18-alpine AS builder        # Base image (FROM is always first)
WORKDIR /app                          # Set working directory inside container
COPY package*.json ./                 # Copy dependency manifest first (cache!)
RUN npm ci --production               # Install dependencies (cached if pkg unchanged)
COPY . .                              # Copy application source code
RUN npm run build                     # Build the application

# ── Stage 2: Production ────────────────────────────
FROM node:18-alpine                   # Fresh, minimal base for production
WORKDIR /app
COPY --from=builder /app/dist ./dist  # Copy only build output from Stage 1
COPY --from=builder /app/node_modules ./node_modules
EXPOSE 3000                           # Document the port (metadata only)
ENV NODE_ENV=production               # Set environment variable
CMD ["node", "dist/server.js"]        # Default command when container starts
```

### Key Dockerfile Instructions

| Instruction | Purpose |
|---|---|
| `FROM` | Base image. **Must be the first instruction.** Every image starts from a parent. |
| `WORKDIR` | Sets the working directory for subsequent instructions. Creates it if needed. |
| `COPY` | Copies files from build context (host) into the image. |
| `ADD` | Like COPY but also handles URLs and auto-extracts tar archives. Prefer COPY. |
| `RUN` | Executes a command during **build time** — installs packages, compiles code. Creates a new layer. |
| `CMD` | Default command to run when a container starts. Only the **last CMD** takes effect. |
| `ENTRYPOINT` | Like CMD but **cannot be overridden** by `docker run` arguments (arguments are appended). |
| `EXPOSE` | Documents which port the container listens on. Does NOT actually publish the port. |
| `ENV` | Sets environment variables available at build time and runtime. |
| `ARG` | Build-time variables (not available at runtime). Used with `--build-arg`. |
| `VOLUME` | Creates a mount point for external storage. |
| `USER` | Sets the user for subsequent instructions and the container runtime. |

### CMD vs ENTRYPOINT

```
  CMD ["python", "app.py"]
  ──────────────────────────
  • Default command — can be FULLY OVERRIDDEN by docker run arguments
  • docker run myimage                    → runs "python app.py"
  • docker run myimage /bin/bash          → runs "/bin/bash" (CMD ignored)

  ENTRYPOINT ["python"]
  CMD ["app.py"]
  ──────────────────────────
  • ENTRYPOINT is the fixed executable — CMD provides default arguments
  • docker run myimage                    → runs "python app.py"
  • docker run myimage script.py          → runs "python script.py"
  • The ENTRYPOINT ("python") cannot be overridden (without --entrypoint flag)
```

> [!NOTE]
> **Best practice:** Use `ENTRYPOINT` for the main executable and `CMD` for default arguments. This allows users to override arguments without changing the executable. Example: `ENTRYPOINT ["nginx"]` + `CMD ["-g", "daemon off;"]`.

---

## Multi-Stage Builds

Multi-stage builds create **smaller, more secure production images** by separating the build environment from the runtime environment.

```
  Without Multi-Stage:               With Multi-Stage:

  ┌───────────────────────┐          ┌───────────────────────┐
  │  Final Image          │          │  Build Stage          │ (discarded)
  │                       │          │  • Compilers          │
  │  • Compilers (200 MB) │          │  • Source code        │
  │  • Build tools        │          │  • Build tools        │
  │  • Source code        │          │  • Test dependencies  │
  │  • Test dependencies  │          └───────────┬───────────┘
  │  • App binary (5 MB)  │                      │ COPY --from
  │                       │          ┌───────────▼───────────┐
  │  Total: ~800 MB       │          │  Production Image     │
  │  ❌ Attack surface    │          │  • App binary (5 MB)  │
  │  ❌ Bloated           │          │  • Runtime only       │
  └───────────────────────┘          │  Total: ~50 MB        │
                                     │  ✅ Minimal, secure   │
                                     └───────────────────────┘
```

| Benefit | Description |
|---|---|
| **Smaller images** | Final image contains only the runtime and compiled artifacts — no build tools, compilers, or source code. |
| **Security** | Smaller attack surface — fewer packages = fewer vulnerabilities. |
| **Faster deploys** | Smaller images push/pull faster to/from registries like ECR. |

---

## Docker Networking

```
  ┌────────────────────────────────────────────────────────┐
  │                   Docker Host                           │
  │                                                        │
  │  Bridge Network (default):          Host Network:      │
  │  ┌──────────────────────┐          ┌───────────────┐   │
  │  │  docker0 bridge      │          │ Container     │   │
  │  │  (172.17.0.1)        │          │ shares host   │   │
  │  │                      │          │ network stack │   │
  │  │  ┌────┐    ┌────┐    │          │ (no isolation)│   │
  │  │  │ C1 │    │ C2 │    │          └───────────────┘   │
  │  │  │.2  │    │.3  │    │                              │
  │  │  └────┘    └────┘    │          None Network:       │
  │  │  Containers get      │          ┌───────────────┐   │
  │  │  private IPs, NAT    │          │ Container     │   │
  │  │  to reach outside    │          │ no networking │   │
  │  └──────────────────────┘          │ (fully isolated│  │
  │                                    └───────────────┘   │
  └────────────────────────────────────────────────────────┘
```

| Network Type | Description | Use Case |
|---|---|---|
| **bridge** | Default. Containers get private IPs on an isolated network. Use `-p` to publish ports. | Most workloads — isolated containers with port mapping |
| **host** | Container shares the host's network namespace. No port mapping needed. | Performance-sensitive apps (no NAT overhead) |
| **none** | No networking. Container is fully isolated. | Security-sensitive batch processing |
| **overlay** | Multi-host networking for Docker Swarm / orchestrators. | Distributed clusters (ECS uses its own networking) |
| **Custom bridge** | User-defined bridge with DNS-based service discovery between containers. | Multi-container apps (containers talk by name) |

### Port Mapping

```
  docker run -p 8080:3000 myapp

  Host (EC2 Instance)              Container
  ┌───────────────────┐           ┌─────────────────┐
  │                   │           │                 │
  │  Port 8080 ◄──────┼───────── │ Port 3000       │
  │  (published)      │  mapped   │ (EXPOSE'd)      │
  │                   │           │                 │
  └───────────────────┘           └─────────────────┘

  -p  HOST_PORT:CONTAINER_PORT
  -p  8080:3000    → Host:8080 maps to Container:3000
  -P  (auto)       → Randomly assigned host port (dynamic port mapping)
```

> [!TIP]
> **AWS relevance:** ECS with EC2 Launch Type uses **dynamic port mapping** (`-P`) with ALB. Each task gets a random host port, and the ALB Target Group tracks these dynamic ports. This allows multiple tasks of the same service to run on one EC2 instance.

---

## Docker Volumes & Data Persistence

Containers are **ephemeral** — when a container is deleted, its writable layer is lost. **Volumes** provide persistent data storage.

```
  ┌───────────────────────────────────────────────────────────┐
  │                       Docker Host                          │
  │                                                           │
  │  Named Volume:              Bind Mount:                   │
  │  ┌──────────┐               ┌──────────┐                  │
  │  │Container │               │Container │                  │
  │  │ /app/data│               │ /app/src │                  │
  │  └────┬─────┘               └────┬─────┘                  │
  │       │                          │                         │
  │       ▼                          ▼                         │
  │  /var/lib/docker/          /home/user/project/            │
  │  volumes/mydata/           src/                           │
  │  (Docker-managed)          (Host directory)               │
  │                                                           │
  │  ✅ Docker manages          ✅ Direct host access         │
  │  ✅ Portable                ❌ Host path dependent        │
  │  ✅ Backup-friendly         ✅ Real-time code sync        │
  └───────────────────────────────────────────────────────────┘
```

| Storage Type | Description | Use Case |
|---|---|---|
| **Named Volume** | Docker-managed storage. Persists after container deletion. Best for production data. | Databases, application state |
| **Bind Mount** | Maps a host directory into the container. Real-time file sync. | Development (live code reloading) |
| **tmpfs Mount** | In-memory storage. Fast but non-persistent. Linux only. | Sensitive data, temp caches |

> [!NOTE]
> **AWS connection:** In ECS, volumes are defined differently — Fargate tasks use **EFS** for persistent shared storage (no Docker volumes). EC2 Launch Type can use bind mounts and Docker volumes, but EFS is still recommended for cross-task/cross-AZ persistence.

---

## Docker Compose

**Docker Compose** defines and runs **multi-container applications** using a single YAML file.

```yaml
# docker-compose.yml
version: "3.8"

services:
  web:
    build: ./web                  # Build from Dockerfile in ./web
    ports:
      - "80:3000"                 # Host:Container port mapping
    environment:
      - DB_HOST=database          # DNS name = service name
    depends_on:
      - database                  # Start database before web
    restart: always

  database:
    image: postgres:15            # Pull from registry
    volumes:
      - db-data:/var/lib/postgresql/data   # Named volume
    environment:
      - POSTGRES_PASSWORD=secret

  cache:
    image: redis:7-alpine
    ports:
      - "6379:6379"

volumes:
  db-data:                        # Declare named volume
```

```
  docker compose up -d     →  Start all services in background
  docker compose down      →  Stop and remove all containers
  docker compose logs -f   →  Stream logs from all services
  docker compose ps        →  List running services

  ┌─────────────────────────────────────────────────┐
  │  Docker Compose Network (auto-created)           │
  │                                                  │
  │  ┌──────┐     ┌──────────┐     ┌───────┐        │
  │  │ web  │────►│ database │     │ cache │        │
  │  │ :3000│ DNS │ :5432    │     │ :6379 │        │
  │  └──────┘     └──────────┘     └───────┘        │
  │                                                  │
  │  Containers discover each other by SERVICE NAME │
  │  (web connects to "database:5432")              │
  └─────────────────────────────────────────────────┘
```

> [!TIP]
> **AWS equivalent:** Docker Compose is for **local development**. In production on AWS, use **ECS Services** (one per microservice) with **ALB/Service Connect** for discovery, **EFS** for volumes, and **Secrets Manager** for credentials. ECS Task Definitions serve a similar role to Compose service definitions.

---

## Docker Security Best Practices

```
  Security Layers:

  ┌──────────────────────────────────────────────────────┐
  │  1. Image Security                                    │
  │     • Use minimal base images (alpine, distroless)   │
  │     • Scan for CVEs (ECR scanning, Trivy, Snyk)      │
  │     • Pin image versions (node:18.17.0, not :latest) │
  │     • Use multi-stage builds (no build tools in prod)│
  ├──────────────────────────────────────────────────────┤
  │  2. Runtime Security                                  │
  │     • Run as non-root user (USER directive)          │
  │     • Read-only filesystem (--read-only flag)        │
  │     • Drop Linux capabilities (--cap-drop ALL)       │
  │     • No --privileged flag in production             │
  ├──────────────────────────────────────────────────────┤
  │  3. Secret Management                                │
  │     • NEVER put secrets in images or Dockerfiles     │
  │     • Use runtime injection (env vars, mounted files)│
  │     • AWS: Secrets Manager / SSM Parameter Store     │
  ├──────────────────────────────────────────────────────┤
  │  4. Registry Security                                │
  │     • Use private registries (ECR private)           │
  │     • Enable image immutability (prevent tag reuse)  │
  │     • IAM-based access control                       │
  └──────────────────────────────────────────────────────┘
```

| Practice | Why |
|---|---|
| **Use `alpine` or `distroless` base** | Smaller image = fewer packages = fewer vulnerabilities |
| **Pin versions** | `FROM node:18.17.0-alpine` not `FROM node:latest` — prevents unexpected breaking changes |
| **Non-root user** | `USER appuser` — prevents container breakout privilege escalation |
| **Scan images** | ECR scanning, Trivy, Snyk detect known CVEs in image layers |
| **Multi-stage builds** | Production image has no compilers, source code, or dev dependencies |
| **.dockerignore** | Exclude `.git`, `node_modules`, secrets from build context |

---

## Essential Docker Commands

```
  Image Lifecycle:
  ────────────────
  docker build -t myapp:v1 .          # Build image from Dockerfile
  docker images                         # List local images
  docker tag myapp:v1 account.ecr/myapp:v1  # Tag for registry
  docker push account.ecr/myapp:v1     # Push to registry (ECR)
  docker pull nginx:latest             # Pull from registry
  docker rmi myapp:v1                  # Remove local image

  Container Lifecycle:
  ────────────────────
  docker run -d -p 8080:3000 myapp:v1  # Run in background, map ports
  docker ps                             # List running containers
  docker ps -a                          # List all (including stopped)
  docker logs -f <container>            # Stream container logs
  docker exec -it <container> /bin/sh   # Interactive shell inside container
  docker stop <container>               # Graceful stop (SIGTERM)
  docker rm <container>                 # Remove stopped container

  System:
  ───────
  docker system prune                   # Clean up unused resources
  docker inspect <container|image>      # Detailed JSON metadata
  docker stats                          # Live resource usage
```

---

## Docker → AWS Mapping

Understanding how Docker concepts translate to AWS services:

| Docker Concept | AWS Equivalent | Notes |
|---|---|---|
| **Docker Hub** | **Amazon ECR** | Private/public container image registry |
| **Dockerfile** | **ECS Task Definition** | Blueprint for running containers (image, CPU, memory, ports, env vars) |
| **docker run** | **ECS RunTask / CreateService** | Launch containers on a cluster |
| **Docker Compose** | **ECS Services + Task Definitions** | Multi-container orchestration (production-grade) |
| **docker network** | **VPC / Security Groups** | Network isolation and connectivity |
| **Docker volumes** | **EFS (Fargate) / EBS (EC2)** | Persistent container storage |
| **docker logs** | **CloudWatch Logs (awslogs driver)** | Centralized container logging |
| **Docker Swarm** | **ECS / EKS** | Container orchestration at scale |
| **Docker secrets** | **Secrets Manager / SSM** | Secure credential injection |

```
  Local Development → AWS Production Pipeline:

  ┌──────────────┐     ┌──────────────┐     ┌──────────────────────┐
  │  Developer   │     │   CI/CD      │     │   AWS Production     │
  │              │     │              │     │                      │
  │  Dockerfile  │────►│  Build Image │────►│  ECR (store image)   │
  │  Compose     │     │  Run Tests   │     │  ECS Task Definition │
  │  Local Test  │     │  Push to ECR │     │  ECS Service (run)   │
  │              │     │              │     │  ALB (route traffic)  │
  └──────────────┘     └──────────────┘     └──────────────────────┘
```

---

## Key Numbers & Limits

| Parameter | Value |
|---|---|
| **Max image layers** | 127 layers per image |
| **Default bridge subnet** | 172.17.0.0/16 |
| **Max ECR image size** | 10 GB per layer |
| **Container stop timeout** | 10 seconds (SIGTERM → SIGKILL) |
| **Dockerfile max size** | No hard limit (practical: keep under a few KB) |
| **Docker Hub pull rate limit** | 100 pulls/6 hours (anonymous), 200 (authenticated) |

---

## Exam Cheat Sheet

> [!TIP]
> **Docker concepts that appear in SAA-C03:**
> - **Docker image** = read-only template → stored in **ECR**
> - **Container** = running instance of image → managed by **ECS/EKS**
> - **Dockerfile** = build instructions → produces an image pushed to ECR
> - **Registry** = image storage → **ECR** (private default) or **Docker Hub** (public)
> - **Port mapping** = `-p HOST:CONTAINER` → ECS uses **dynamic port mapping** with ALB
> - **Volumes** = persistent storage → ECS uses **EFS** (Fargate) or bind mounts (EC2)
> - **Multi-stage builds** = smaller, secure images → best practice for ECR
>
> **Key connections to AWS:**
> - Docker CLI `push/pull` ↔ **ECR** (authenticate via `aws ecr get-login-password`)
> - Docker Compose (local) ↔ **ECS Task Definitions + Services** (production)
> - Container logs → **CloudWatch Logs** via `awslogs` log driver
> - Container secrets → **Secrets Manager / SSM Parameter Store** (never bake into image)
> - Image scanning → **ECR image scanning** (detect CVEs on push)
