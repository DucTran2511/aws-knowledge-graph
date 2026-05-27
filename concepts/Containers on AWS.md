---
tags: [concept, containers, docker, microservices, orchestration, serverless, compute]
aliases: [ECS, Fargate, ECR, EKS, Amazon ECS, AWS Fargate, Amazon ECR, Amazon EKS, Elastic Container Service, Elastic Kubernetes Service, Elastic Container Registry]
date: 2026-05-27
---

# Containers on AWS — ECS, Fargate, ECR & EKS

**Containers** package application code, dependencies, and configuration into a single portable unit that runs consistently across environments. AWS provides a full container ecosystem: **ECR** (store images), **ECS** (orchestrate with AWS-native tooling), **EKS** (orchestrate with Kubernetes), and **Fargate** (serverless compute for containers).

> [!IMPORTANT]
> **Core exam concept:** If the question mentions **Docker**, **containers**, or **microservices** → think **ECS or EKS**. If it adds "**no infrastructure to manage**" → think **Fargate**. If it says "**store container images**" → think **ECR**.

---

## Docker — The Foundation

```
  Traditional Deployment:              Container Deployment:

  ┌─────────────────────┐              ┌──────┐ ┌──────┐ ┌──────┐
  │   App A  │  App B   │              │App A │ │App B │ │App C │
  │   Libs   │  Libs    │              │+Libs │ │+Libs │ │+Libs │
  │ (conflict possible) │              └──┬───┘ └──┬───┘ └──┬───┘
  ├─────────────────────┤                 │        │        │
  │   Operating System  │              ┌──▼────────▼────────▼──┐
  ├─────────────────────┤              │    Container Runtime   │
  │     Hardware        │              │    (Docker Engine)     │
  └─────────────────────┘              ├───────────────────────┤
                                       │    Operating System   │
  ❌ Dependency conflicts              ├───────────────────────┤
  ❌ "Works on my machine"             │      Hardware         │
                                       └───────────────────────┘

                                       ✅ Isolated, portable
                                       ✅ Consistent everywhere
```

### Key Docker Concepts

| Concept | Description |
|---|---|
| **Docker Image** | Read-only template with app code + dependencies. Built from a **Dockerfile**. |
| **Container** | Running instance of an image. Lightweight, isolated, ephemeral. |
| **Dockerfile** | Instructions to build an image (base OS, copy code, install deps, set CMD). |
| **Registry** | Repository to store and distribute images (Docker Hub, **Amazon ECR**). |

---

## Amazon ECR (Elastic Container Registry)

**ECR** is a fully managed **Docker container image registry** — AWS's alternative to Docker Hub.

```
  ┌──────────────────────────────────────────────────────┐
  │                   Amazon ECR                          │
  │                                                      │
  │  ┌────────────┐  ┌────────────┐  ┌────────────┐     │
  │  │ my-app:v1  │  │ my-app:v2  │  │ api:latest │     │
  │  │ (image)    │  │ (image)    │  │ (image)    │     │
  │  └────────────┘  └────────────┘  └────────────┘     │
  │                                                      │
  │  ✅ Private repositories (default)                   │
  │  ✅ Public repositories (ECR Public Gallery)         │
  │  ✅ Image vulnerability scanning                     │
  │  ✅ Image lifecycle policies (auto-cleanup old tags) │
  │  ✅ Cross-region / cross-account replication         │
  └──────────────────────────────────────────────────────┘

  docker push → ECR        (store images)
  docker pull ← ECR        (retrieve images for ECS/EKS)
```

| Feature | Detail |
|---|---|
| **Private repos** | Default. IAM-based access control. |
| **Public repos** | ECR Public Gallery (gallery.ecr.aws) — publicly accessible images. |
| **Vulnerability scanning** | Automatic or on-push scanning for CVEs. |
| **Lifecycle policies** | Auto-delete untagged or old images to save storage costs. |
| **Replication** | Cross-region and cross-account image replication. |
| **Backed by S3** | Images stored durably in S3 (managed by AWS). |

> [!TIP]
> **Exam Pattern:** "Store Docker images on AWS" or "private container registry" → **Amazon ECR**. ECR integrates natively with ECS and EKS — no extra authentication configuration needed when using IAM roles.

---

## Amazon ECS (Elastic Container Service)

**Amazon ECS** is AWS's **proprietary container orchestration** service. It manages the lifecycle of containers — starting, stopping, scaling, and placing them across a cluster of compute resources.

> [!IMPORTANT]
> **Core exam concept:** ECS is the **AWS-native** container orchestrator. If the question doesn't mention Kubernetes → **ECS**. If it mentions Kubernetes → **EKS**.

### ECS Architecture

```
  ┌──────────────────────────────────────────────────────────────┐
  │                        ECS Cluster                            │
  │                                                              │
  │  ┌────────────────────────────────────────────────────────┐  │
  │  │  Service: "web-app" (Desired count: 3)                 │  │
  │  │                                                        │  │
  │  │  ┌──────────┐  ┌──────────┐  ┌──────────┐             │  │
  │  │  │  Task 1  │  │  Task 2  │  │  Task 3  │             │  │
  │  │  │┌────────┐│  │┌────────┐│  │┌────────┐│             │  │
  │  │  ││Container││  ││Container││  ││Container││             │  │
  │  │  │└────────┘│  │└────────┘│  │└────────┘│             │  │
  │  │  └──────────┘  └──────────┘  └──────────┘             │  │
  │  └────────────────────────────────────────────────────────┘  │
  │                                                              │
  │  Compute: EC2 Launch Type  OR  Fargate Launch Type           │
  └──────────────────────────────────────────────────────────────┘
```

### Core ECS Concepts

| Concept | Description |
|---|---|
| **Cluster** | Logical grouping of tasks/services. Can span multiple AZs. |
| **Task Definition** | Blueprint (JSON) describing one or more containers: image, CPU, memory, ports, env vars, IAM role. Think of it as the **"Dockerfile for ECS"**. |
| **Task** | Running instance of a Task Definition. One or more containers running together. |
| **Service** | Maintains a **desired count** of tasks. Handles scaling, replacement, and load balancer integration. |
| **Container Agent** | Runs on each EC2 instance (EC2 launch type only). Communicates with ECS control plane. |

---

### ECS Launch Types

This is the **#1 most tested ECS concept** — understanding when to use EC2 vs Fargate.

#### EC2 Launch Type

```
  ┌──────────────────────────────────────────────────────────┐
  │                    ECS Cluster                            │
  │                                                          │
  │  ┌──────────────────┐  ┌──────────────────┐              │
  │  │  EC2 Instance 1  │  │  EC2 Instance 2  │              │
  │  │  ┌──────┐┌──────┐│  │  ┌──────┐        │              │
  │  │  │Task A││Task B││  │  │Task C│        │              │
  │  │  └──────┘└──────┘│  │  └──────┘        │              │
  │  │  ECS Agent ✓     │  │  ECS Agent ✓     │              │
  │  └──────────────────┘  └──────────────────┘              │
  │                                                          │
  │  YOU manage:                                             │
  │  • Provisioning EC2 instances                            │
  │  • Patching OS, Docker runtime                           │
  │  • Scaling the EC2 fleet (ASG)                           │
  │  • Capacity planning                                     │
  └──────────────────────────────────────────────────────────┘
```

- **You provision and maintain** the EC2 instances (the infrastructure).
- Each EC2 instance must run the **ECS Agent** to register with the cluster.
- **You are responsible** for scaling the underlying EC2 fleet (typically via an ASG).
- ECS places tasks on your EC2 instances based on CPU/memory availability.

#### Fargate Launch Type (Serverless)

```
  ┌──────────────────────────────────────────────────────────┐
  │                    ECS Cluster                            │
  │                                                          │
  │  ┌──────────┐  ┌──────────┐  ┌──────────┐               │
  │  │  Task A  │  │  Task B  │  │  Task C  │               │
  │  │  (auto)  │  │  (auto)  │  │  (auto)  │               │
  │  └──────────┘  └──────────┘  └──────────┘               │
  │                                                          │
  │  AWS manages ALL infrastructure:                         │
  │  • No EC2 instances to provision                         │
  │  • No capacity planning                                  │
  │  • No patching or OS management                          │
  │  • Auto-scales based on task count                       │
  │  • Pay per task (vCPU + memory per second)               │
  └──────────────────────────────────────────────────────────┘
```

- **Serverless** — you just define Task Definitions and AWS runs them.
- No EC2 instances to manage, no ECS Agent, no capacity planning.
- Pay per **task resource usage** (vCPU + memory, per second).

### EC2 vs Fargate Comparison

| Feature | EC2 Launch Type | Fargate Launch Type |
|---|---|---|
| **Infrastructure** | You manage EC2 instances | AWS manages everything |
| **Scaling** | Scale EC2 fleet (ASG) + ECS tasks | Scale tasks only |
| **Pricing** | Pay for EC2 instances (even if idle) | Pay per task (vCPU + memory/second) |
| **Control** | Full control (SSH, OS, GPU, custom AMI) | No SSH, no OS access |
| **Use case** | Need GPU, custom OS, compliance, cost optimization at scale | Serverless, simple, no ops overhead |
| **ECS Agent** | Required on each instance | Not needed |

> [!WARNING]
> **Exam trap:** "No servers to manage" or "serverless containers" → **Fargate**. "Need GPU instances" or "need SSH access to container host" or "custom AMI" → **EC2 Launch Type**. Both run on ECS — the difference is WHO manages the infrastructure.

---

### ECS IAM Roles

```
  EC2 Launch Type:

  ┌──────────────────────────────────────┐
  │  EC2 Instance                        │
  │  ┌─────────────────────────────────┐ │
  │  │ EC2 Instance Profile            │ │
  │  │ (shared by ALL containers)      │ │
  │  │ • Pull images from ECR          │ │
  │  │ • Send logs to CloudWatch       │ │
  │  │ • Reference secrets from SSM    │ │
  │  └─────────────────────────────────┘ │
  │                                      │
  │  ┌──────────┐  ┌──────────┐          │
  │  │  Task A  │  │  Task B  │          │
  │  │  ┌─────┐ │  │  ┌─────┐ │          │
  │  │  │Role │ │  │  │Role │ │          │
  │  │  │  A  │ │  │  │  B  │ │          │
  │  │  └─────┘ │  │  └─────┘ │          │
  │  └──────────┘  └──────────┘          │
  │  Task Roles (per-task permissions)   │
  │  • Task A → access S3 bucket X       │
  │  • Task B → access DynamoDB table Y  │
  └──────────────────────────────────────┘
```

| Role | Scope | Purpose |
|---|---|---|
| **EC2 Instance Profile** | Per EC2 instance (EC2 launch type only) | ECS Agent permissions: pull ECR images, send CloudWatch logs, access SSM/Secrets Manager |
| **ECS Task Role** | Per task definition | Application-level permissions: access S3, DynamoDB, etc. **Each task can have a different role.** |
| **ECS Task Execution Role** | Per task definition | Infrastructure permissions: pull ECR images, write logs. Used by **Fargate** and ECS Agent. |

> [!CAUTION]
> **Exam critical:** Don't confuse **Task Role** (what the application inside the container can do) with **Task Execution Role** (what ECS/Fargate needs to launch the task — pull image, write logs). Task Role = **app permissions**. Task Execution Role = **infrastructure permissions**.

---

### ECS + Load Balancer Integration

```
  Internet
     │
     ▼
  ┌──────────────────┐
  │  Application     │
  │  Load Balancer   │
  │  (ALB)           │
  └────────┬─────────┘
           │  Dynamic port mapping (EC2)
           │  or IP-based targeting (Fargate)
           ▼
  ┌──────────────────────────────────────┐
  │  ECS Service (desired count: 3)      │
  │                                      │
  │  ┌────────┐ ┌────────┐ ┌────────┐   │
  │  │ Task 1 │ │ Task 2 │ │ Task 3 │   │
  │  │ :8080  │ │ :8081  │ │ :8082  │   │
  │  └────────┘ └────────┘ └────────┘   │
  └──────────────────────────────────────┘
```

- **ALB** is the recommended load balancer for ECS (supports dynamic port mapping, path-based routing).
- **NLB** is recommended for high throughput, TCP/UDP, or AWS PrivateLink use cases.
- ECS Service automatically registers/deregisters tasks with the target group.

> [!TIP]
> **Exam Pattern:** "Expose ECS containers to the internet" or "distribute traffic across ECS tasks" → **ALB in front of ECS Service**. ALB supports dynamic port mapping, allowing multiple tasks on the same EC2 instance.

---

### ECS Data Persistence — EFS Integration

```
  ┌──────────────────────────────────────────────────────────┐
  │                     ECS Cluster                           │
  │                                                          │
  │  AZ-1                        AZ-2                        │
  │  ┌──────────┐               ┌──────────┐                │
  │  │  Task A  │               │  Task B  │                │
  │  │  (mount) │               │  (mount) │                │
  │  └────┬─────┘               └────┬─────┘                │
  │       │                          │                       │
  │       └──────────┐  ┌───────────┘                       │
  │                  ▼  ▼                                    │
  │           ┌──────────────┐                               │
  │           │  Amazon EFS  │                               │
  │           │  (shared     │                               │
  │           │   file       │                               │
  │           │   system)    │                               │
  │           └──────────────┘                               │
  │                                                          │
  │  ✅ Works with BOTH EC2 and Fargate launch types         │
  │  ✅ Multi-AZ shared storage for containers               │
  │  ✅ Persistent data survives task restarts               │
  └──────────────────────────────────────────────────────────┘
```

| Storage | Works with | Use case |
|---|---|---|
| **EFS** | EC2 + Fargate ✅ | Shared persistent storage across tasks and AZs |
| **EBS** | EC2 only ❌ Fargate | Single-task block storage (not shared) |
| **Bind Mounts** | EC2 only | Share data between containers in same task |

> [!WARNING]
> **Exam trap:** "Shared storage for Fargate tasks" → **EFS** (the only option). EBS volumes cannot be mounted to Fargate tasks. S3 is not a file system mount. Note: **S3 cannot be mounted as a file system** on ECS tasks.

---

### ECS Auto Scaling

ECS supports **two layers** of scaling:

```
  Layer 1: ECS Service Auto Scaling (Task-level)
  ───────────────────────────────────────────────
  Scale the NUMBER OF TASKS in a service.

  Metrics:
  • CPU Utilization (average across tasks)
  • Memory Utilization
  • ALB Request Count per Target

  Scaling Types:
  • Target Tracking (maintain CPU at 70%)
  • Step Scaling (CloudWatch alarm triggers)
  • Scheduled Scaling (time-based)

  ───────────────────────────────────────────────
  Layer 2: EC2 Auto Scaling (Infrastructure-level)
  ───────────────────────────────────────────────
  Scale the NUMBER OF EC2 INSTANCES (EC2 launch type only).

  Options:
  • ASG with CloudWatch alarms
  • ECS Cluster Capacity Provider (recommended)
    → Automatically adds/removes EC2 instances
       based on task placement needs
```

> [!TIP]
> **Exam Pattern:** "Auto-scale ECS tasks" → **ECS Service Auto Scaling**. "Auto-scale underlying EC2 instances for ECS" → **ECS Cluster Capacity Provider** or ASG. With **Fargate**, you ONLY scale tasks (no infrastructure to manage).

---

### ECS Rolling Updates

```
  Deployment Configuration:
  ──────────────────────────

  minimumHealthyPercent = 50%     maximumPercent = 200%
  (min tasks during update)       (max tasks during update)

  Example: Service with 4 tasks, deploying v2

  Step 1: Running 4 v1 tasks
          [v1] [v1] [v1] [v1]                    (100%)

  Step 2: Start 4 v2 tasks (max 200% = 8 total)
          [v1] [v1] [v1] [v1] [v2] [v2] [v2] [v2] (200%)

  Step 3: Drain 4 v1 tasks
          [v2] [v2] [v2] [v2]                    (100%)

  ✅ Zero downtime deployment
```

---

## AWS Fargate

**Fargate** is a **serverless compute engine** for containers. It works with **both ECS and EKS**.

```
  ┌─────────────────────────────────────────┐
  │             AWS Fargate                  │
  │                                         │
  │  "Serverless containers"                │
  │                                         │
  │  ┌───────────┐     ┌───────────┐        │
  │  │   ECS     │     │   EKS     │        │
  │  │  Tasks    │     │   Pods    │        │
  │  └───────────┘     └───────────┘        │
  │                                         │
  │  ✅ No EC2 instances to manage          │
  │  ✅ No capacity planning                │
  │  ✅ Pay per task (vCPU + memory)        │
  │  ✅ Scales automatically                │
  │  ❌ No SSH, no OS access                │
  │  ❌ No GPU support (use EC2 for GPU)    │
  └─────────────────────────────────────────┘
```

> [!IMPORTANT]
> **Fargate is NOT a service** — it's a **launch type / compute engine**. You still use ECS or EKS to orchestrate. Fargate just replaces EC2 as the compute layer. Think: **ECS/EKS = orchestrator, Fargate = serverless compute**.

---

## Amazon EKS (Elastic Kubernetes Service)

**Amazon EKS** is a **managed Kubernetes** service. It runs the Kubernetes control plane for you.

```
  ┌──────────────────────────────────────────────────────────┐
  │                    Amazon EKS                             │
  │                                                          │
  │  ┌────────────────────────────────────────┐              │
  │  │  Kubernetes Control Plane (managed)    │              │
  │  │  • API Server                          │              │
  │  │  • etcd                                │              │
  │  │  • Scheduler                           │              │
  │  │  • Controller Manager                  │              │
  │  │  (runs across multiple AZs by AWS)     │              │
  │  └────────────────────────────────────────┘              │
  │                        │                                 │
  │           ┌────────────┼────────────┐                    │
  │           ▼            ▼            ▼                    │
  │  ┌──────────────┐ ┌──────────────┐ ┌──────────────┐     │
  │  │ Worker Node  │ │ Worker Node  │ │ Worker Node  │     │
  │  │ (EC2)        │ │ (EC2)        │ │ (Fargate)    │     │
  │  │ ┌────┐┌────┐ │ │ ┌────┐      │ │ ┌────┐       │     │
  │  │ │Pod ││Pod │ │ │ │Pod │      │ │ │Pod │       │     │
  │  │ └────┘└────┘ │ │ └────┘      │ │ └────┘       │     │
  │  └──────────────┘ └──────────────┘ └──────────────┘     │
  │                                                          │
  │  Worker Nodes: EC2 instances OR Fargate (serverless)     │
  └──────────────────────────────────────────────────────────┘
```

### EKS Key Concepts

| Concept | Description |
|---|---|
| **Pod** | Smallest deployable unit in Kubernetes. One or more containers. (Analogous to ECS Task) |
| **Node** | Worker machine (EC2 instance or Fargate) that runs Pods. |
| **Node Group** | Collection of EC2 instances managed as a group (Managed or Self-Managed). |
| **Control Plane** | Kubernetes master components. **Fully managed by AWS** in EKS. |

### EKS Node Types

| Type | Description |
|---|---|
| **Managed Node Groups** | AWS creates and manages EC2 instances (ASG). Supports On-Demand and Spot. |
| **Self-Managed Nodes** | You provision EC2 instances and register them. Full control. |
| **Fargate** | Serverless — no nodes to manage. One Fargate instance per Pod. |

> [!TIP]
> **Exam Pattern:** "Company already uses Kubernetes on-premises" or "multi-cloud Kubernetes" → **EKS**. "Cloud-native, no Kubernetes requirement" → **ECS**. EKS is Kubernetes-compatible — any K8s tool/plugin works.

---

## ECS vs EKS — When to Use Which

| Aspect | ECS | EKS |
|---|---|---|
| **Orchestrator** | AWS proprietary | Open-source Kubernetes |
| **Learning curve** | Simpler (AWS-native) | Steeper (Kubernetes ecosystem) |
| **Portability** | AWS-only | Multi-cloud, hybrid, on-premises |
| **Existing K8s** | Not compatible | Drop-in for existing Kubernetes |
| **Ecosystem** | AWS tools only | Huge K8s ecosystem (Helm, Istio, etc.) |
| **Compute** | EC2 or Fargate | EC2, Fargate, or EKS Anywhere |
| **Use case** | New AWS-native microservices | Existing K8s workloads, multi-cloud strategy |

> [!NOTE]
> Both ECS and EKS support Fargate as a compute option. The choice between ECS and EKS is about the **orchestration layer** (AWS-native vs Kubernetes), not the compute layer.

---

## EKS Anywhere & ECS Anywhere

```
  Run AWS container orchestration OUTSIDE of AWS:

  ┌─────────────────────────────────────────────────┐
  │  EKS Anywhere                                    │
  │  • Run EKS on your own on-premises hardware     │
  │  • Full Kubernetes API compatibility             │
  │  • Connected or disconnected from AWS            │
  └─────────────────────────────────────────────────┘

  ┌─────────────────────────────────────────────────┐
  │  ECS Anywhere                                    │
  │  • Run ECS tasks on your on-premises servers    │
  │  • Install ECS Agent + SSM Agent on your infra  │
  │  • Managed from the AWS ECS console/API         │
  └─────────────────────────────────────────────────┘
```

> [!TIP]
> **Exam Pattern:** "Run containers on-premises managed by AWS" → **ECS Anywhere** or **EKS Anywhere**. Choice depends on whether they use Kubernetes or not.

---

## Amazon ECR + ECS + Fargate — End-to-End Flow

```
  Developer                    AWS Cloud
  ─────────                    ─────────

  1. Write Dockerfile
     │
  2. Build image
     │
  3. docker push ────────────► Amazon ECR
                                  │
  4. Create Task Definition       │ (pull image)
     (reference ECR image) ───────┤
     │                            │
  5. Create ECS Service           │
     (Fargate launch type)        │
     │                            ▼
  6. ECS places tasks ──────► Fargate provisions
                               compute, runs containers
                                  │
  7. ALB routes traffic ─────► Running tasks
                                  │
  8. CloudWatch ◄────────── Logs & Metrics
```

---

## Key Integrations

| Integration | Description |
|---|---|
| **ECS + ALB** | Expose services, dynamic port mapping, path-based routing |
| **ECS + EFS** | Persistent shared storage for tasks (works with Fargate) |
| **ECS + CloudWatch** | Container logs (awslogs driver), metrics, Container Insights |
| **ECS + EventBridge** | Trigger actions on task state changes (started, stopped) |
| **ECS + SSM Parameter Store / Secrets Manager** | Inject secrets and config into containers as env vars |
| **ECR + ECS/EKS** | Native image pull — no extra auth with proper IAM roles |

---

## Key Numbers & Limits

| Parameter | Value |
|---|---|
| **ECR max image size** | 10 GB per layer |
| **ECS max tasks per service** | 5,000 |
| **Fargate vCPU range** | 0.25 – 16 vCPU |
| **Fargate memory range** | 0.5 GB – 120 GB |
| **EKS max pods per node** | Depends on EC2 instance type (ENI-based) |
| **ECS Task Definition max containers** | 10 containers per task |

---

## Exam Cheat Sheet

> [!TIP]
> **Trigger phrases → Container Services:**
> - "Docker containers on AWS" → **ECS** (default) or **EKS** (if Kubernetes)
> - "No servers to manage" + "containers" → **Fargate** launch type
> - "Store container images" or "private registry" → **Amazon ECR**
> - "Already using Kubernetes" or "multi-cloud" → **EKS**
> - "Migrate on-prem containers to AWS, simplest" → **ECS** on Fargate
> - "Shared persistent storage for containers" → **EFS** mounted to ECS tasks
> - "Run containers on-premises" → **ECS Anywhere** or **EKS Anywhere**
> - "GPU workloads in containers" → **EC2 launch type** (Fargate doesn't support GPU)
> - "Serverless containers" → **Fargate** (NOT Lambda — Lambda is for functions, Fargate is for containers)
>
> **Key facts:**
> - ECS = AWS-native orchestrator. EKS = managed Kubernetes. Both support Fargate.
> - Fargate = serverless compute engine, NOT an orchestrator. Works with ECS AND EKS.
> - ECR = container image registry. Private by default. Supports vulnerability scanning.
> - Task Role ≠ Task Execution Role. Task Role = app permissions. Execution Role = infra permissions.
> - EFS is the ONLY persistent shared storage option for Fargate tasks.
> - ALB is the recommended LB for ECS (dynamic port mapping).
> - ECS Service Auto Scaling scales TASKS. Cluster Capacity Provider scales EC2 INSTANCES.
> - ECS Anywhere / EKS Anywhere → run containers on-premises, managed by AWS.
