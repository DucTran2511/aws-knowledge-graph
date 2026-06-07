---
tags: [concept, serverless, compute, lambda, event-driven, function]
aliases: [Lambda, AWS Lambda, Lambda Function, Serverless Function, Lambda@Edge]
date: 2026-05-29
---

# AWS Lambda

**AWS Lambda** is a **serverless compute service** that runs your code in response to events. You upload your code, define a trigger, and Lambda handles everything — provisioning, scaling, patching, and monitoring. You pay only for the compute time you consume — **per request and per GB-second**.

> [!IMPORTANT]
> **Core exam concept:** Lambda is the **default answer** for serverless compute. If the question mentions "run code without managing servers," "event-driven processing," or "serverless function" → **Lambda**. If the task runs longer than **15 minutes** → Lambda is NOT the answer (use [[Containers on AWS|ECS/Fargate]], [[AWS Step Functions]], or EC2).

---

## Lambda Execution Model

```
  Event Source                    Lambda                        Destination
  ───────────                    ──────                        ───────────

  ┌──────────┐    trigger     ┌──────────────────┐    result    ┌──────────┐
  │API Gateway│──────────────►│                  │────────────►│ DynamoDB │
  │S3 Event   │               │  Lambda Function │              │ S3       │
  │SQS Message│               │                  │              │ SQS      │
  │DDB Stream │               │  Your Code       │              │ SNS      │
  │EventBridge│               │  + Dependencies  │              │ Kinesis  │
  │CloudWatch │               │                  │              │          │
  └──────────┘               └──────────────────┘              └──────────┘

  1. Event triggers Lambda
  2. Lambda provisions a container (if cold) or reuses one (if warm)
  3. Your code executes (up to 15 min)
  4. Lambda scales automatically (up to 1,000 concurrent by default)
  5. You pay per request + per GB-second of compute
```

### Supported Runtimes

| Runtime | Languages |
|---|---|
| **Native** | Node.js, Python, Java, C#/.NET, Go, Ruby, PowerShell |
| **Custom Runtime** | Any language via **Custom Runtime API** (provide a bootstrap file) |
| **Container Image** | Package as Docker container (must implement Lambda Runtime API, up to 10 GB image) |

> [!NOTE]
> Lambda Container Image support does NOT mean Lambda = ECS. The container must implement the **Lambda Runtime API**. You can't just take any Docker image and run it on Lambda.

---

## Lambda Execution Lifecycle — Cold Start vs Warm Start

```
  COLD START (first invocation or scale-up):
  ──────────────────────────────────────────

  ┌─────────────┐   ┌─────────────┐   ┌──────────────────┐
  │ Download    │──►│ Start       │──►│ Execute          │
  │ code        │   │ runtime +   │   │ handler function │
  │ package     │   │ init code   │   │                  │
  └─────────────┘   └─────────────┘   └──────────────────┘
  ├──── INIT phase (billed) ────┤     ├── INVOKE phase ──┤
       ~100ms–10s                          your code

  WARM START (subsequent invocations — reuse existing container):
  ───────────────────────────────────────────────────────────────

                                      ┌──────────────────┐
                            ────────►│ Execute          │
                                      │ handler function │
                                      │                  │
                                      └──────────────────┘
                                      ├── INVOKE phase ──┤
                                       ✅ FAST — no init
```

### Reducing Cold Starts

| Strategy | Description |
|---|---|
| **Provisioned Concurrency** | Pre-initializes N execution environments. **Eliminates cold starts entirely.** Costs extra money. |
| **Smaller deployment package** | Fewer dependencies = faster download + init |
| **Lighter runtimes** | Python/Node.js init faster than Java/.NET |
| **Init code outside handler** | DB connections, SDK clients initialized once, reused across warm invocations |
| **SnapStart (Java)** | Snapshots initialized state for near-instant cold starts (Java only) |

> [!TIP]
> **Exam Pattern:** "Latency-sensitive API" or "eliminate cold starts" → **Provisioned Concurrency**. "Java Lambda cold start too slow" → **SnapStart**. Remember: Provisioned Concurrency must be set on a **Version or Alias**, NOT `$LATEST`.

---

## Invocation Types

### Synchronous Invocation (caller waits)

```
  ┌──────────┐    invoke     ┌──────────┐    response   ┌──────────┐
  │  Caller  │──────────────►│  Lambda  │──────────────►│  Caller  │
  │          │               │          │               │          │
  │API Gateway│              │          │               │          │
  │ALB        │              │          │               │          │
  │CLI/SDK    │              │          │               │          │
  └──────────┘               └──────────┘               └──────────┘

  ❌ No automatic retries — caller must handle errors
  ❌ Caller blocks until Lambda responds
```

### Asynchronous Invocation (fire-and-forget)

```
  ┌──────────┐    invoke     ┌──────────────┐    ┌──────────┐
  │  Caller  │──────────────►│ Event Queue  │───►│  Lambda  │
  │          │  (returns 202)│ (internal)   │    │          │
  │S3 Event  │               └──────────────┘    └────┬─────┘
  │SNS       │                                        │
  │EventBridg│                              ┌─────────┴──────────┐
  └──────────┘                              │                    │
                                     ┌──────▼─────┐    ┌────────▼────────┐
                                     │  Success   │    │  Failure        │
                                     │Destination │    │  (2 retries)    │
                                     │(SQS/SNS/  │    │  then → DLQ or  │
                                     │ Lambda/   │    │  Failure         │
                                     │ EventBridg)│    │  Destination    │
                                     └────────────┘    └─────────────────┘
```

> [!WARNING]
> **Exam trap:** Async invocation automatically retries **twice** on failure (total 3 attempts). After all retries fail → message goes to a **Dead-Letter Queue (DLQ)** or a **failure destination**. Lambda Destinations are preferred over DLQs (more flexible — support SQS, SNS, Lambda, or EventBridge).

### Event Source Mappings (Lambda polls the source)

```
  ┌──────────┐         ┌─────────────────────┐         ┌──────────┐
  │   SQS    │◄────────│  Event Source        │────────►│  Lambda  │
  │  Kinesis │  poll   │  Mapping             │ invoke  │ Function │
  │DDB Stream│         │  (managed by Lambda) │         │          │
  └──────────┘         └─────────────────────┘         └──────────┘

  SQS:          Batch size 1–10, Lambda deletes msgs after success
  Kinesis:      Batch size 1–10,000, per-shard parallelism
  DDB Streams:  Batch size 1–10,000, per-shard processing
```

| Source | Batch Size | Error Behavior |
|---|---|---|
| **[[Amazon SQS]]** | 1–10 (Standard), 1–10 (FIFO) | Failed msgs return to queue → DLQ on **SQS queue** |
| **[[Amazon Kinesis]]** | 1–10,000 | Retries entire batch. Bisect on error. Shard blocked until success. |
| **DynamoDB Streams** | 1–10,000 | Same as Kinesis (retry + bisect) |
| **Amazon MQ / Kafka** | Varies | Varies |

> [!CAUTION]
> **Exam critical:** With SQS event source mapping, set the queue's **visibility timeout to 6x the Lambda timeout** to prevent duplicate processing. Configure the **DLQ on the SQS queue** (not on Lambda) for event source mapping failures.

---

## Lambda Concurrency

```
  Account Concurrency Pool (default: 1,000 per region)
  ═══════════════════════════════════════════════════════

  ┌───────────────────────────────────────────────────┐
  │  Function A: Unreserved    (shares the pool)      │
  │  Function B: Reserved = 100 (guaranteed, capped)  │
  │  Function C: Provisioned = 50 (pre-warmed, no     │
  │              cold starts, costs extra)             │
  └───────────────────────────────────────────────────┘

  Unreserved:   Shares remaining pool. Risk: other functions can
                starve it. No cost.
  Reserved:     Guarantees N concurrent executions. Also CAPS at N.
                No cost. Protects other functions too.
  Provisioned:  Pre-initializes N execution environments.
                Eliminates cold starts. Costs money.
                Must be set on a VERSION or ALIAS (not $LATEST).
```

| Concurrency Type | Guarantees Capacity | Eliminates Cold Start | Cost | Set On |
|---|---|---|---|---|
| **Unreserved** | ❌ | ❌ | Free | Default |
| **Reserved** | ✅ (also caps) | ❌ | Free | Function |
| **Provisioned** | ✅ | ✅ | 💰 Extra | Version/Alias |

> [!TIP]
> **Exam Pattern:** "Protect critical Lambda from being throttled by other functions" → **Reserved Concurrency**. "Eliminate cold starts for latency-sensitive function" → **Provisioned Concurrency**. "Lambda is being throttled" → increase account concurrency limit or set reserved concurrency.

### Throttling Behavior

| Invocation Type | When Throttled |
|---|---|
| **Synchronous** | Returns **429 ThrottlingError** — caller must retry |
| **Asynchronous** | Event goes to internal queue, retried for up to 6 hours |
| **Event Source Mapping** | Depends on source (SQS: messages return to queue; Kinesis: shard blocked) |

---

## Lambda Versions, Aliases & Layers

### Versions & Aliases

```
  $LATEST (mutable — always latest code)
     │
     ├──► Version 1 (immutable snapshot)
     │        ▲
     │        │── Alias: "PROD" ──► points to Version 1
     │
     ├──► Version 2 (immutable snapshot)
     │        ▲
     │        │── Alias: "DEV" ──► points to Version 2
     │
     └──► Version 3 (immutable snapshot)

  Aliases support WEIGHTED TRAFFIC SHIFTING:
  Alias "PROD" → 90% Version 1, 10% Version 2 (canary deployment)
```

| Concept | Description |
|---|---|
| **$LATEST** | Mutable, always the latest code. NOT suitable for Provisioned Concurrency. |
| **Version** | Immutable snapshot of code + config. Once published, cannot be changed. |
| **Alias** | Named pointer to a version. Can split traffic between two versions (canary/blue-green). |

### Lambda Layers

```
  ┌──────────────────────────────┐
  │  Lambda Function             │
  │  ┌────────────────────────┐  │
  │  │  Your application code │  │  (small, changes often)
  │  └────────────────────────┘  │
  │  ┌────────────────────────┐  │
  │  │  Layer 1: numpy/pandas │  │  (shared, changes rarely)
  │  └────────────────────────┘  │
  │  ┌────────────────────────┐  │
  │  │  Layer 2: common utils │  │  (shared, changes rarely)
  │  └────────────────────────┘  │
  └──────────────────────────────┘

  ✅ Share libraries across functions (up to 5 layers)
  ✅ Smaller deployment packages
  ✅ Separate concerns: code vs dependencies
  📏 Total unzipped size (code + layers) ≤ 250 MB
```

---

## Lambda@Edge & CloudFront Functions

```
  ┌─────────────────────────────────────────────────────────────────┐
  │                     CloudFront Distribution                      │
  │                                                                  │
  │  Viewer Request    Origin Request     Origin Response   Viewer  │
  │  ┌──────────┐     ┌──────────┐       ┌──────────┐    Response  │
  │  │CF Function│     │Lambda@   │       │Lambda@   │    ┌──────┐  │
  │  │    OR     │     │Edge      │       │Edge      │    │CF Fn │  │
  │  │Lambda@   │     │          │       │          │    │  OR  │  │
  │  │Edge      │     │          │       │          │    │L@E   │  │
  │  └──────────┘     └──────────┘       └──────────┘    └──────┘  │
  │       ▼                ▼                   ▲              ▲     │
  │   ┌──────┐         ┌──────┐           ┌──────┐       ┌──────┐  │
  │   │Client│────────►│ Edge │──────────►│Origin│──────►│Client│  │
  │   └──────┘         └──────┘           └──────┘       └──────┘  │
  └─────────────────────────────────────────────────────────────────┘
```

| Feature | CloudFront Functions | Lambda@Edge |
|---|---|---|
| **Runtime** | JavaScript only | Node.js, Python |
| **Execution location** | 225+ CloudFront POPs | Regional Edge Caches |
| **Scale** | Millions of requests/sec | Thousands/sec |
| **Max execution time** | < 1 ms | 5–30 seconds |
| **Max memory** | 2 MB | 128–10,240 MB |
| **Network/File access** | ❌ | ✅ |
| **Request/Response body access** | ❌ | ✅ |
| **Trigger points** | Viewer Request/Response only | All 4 trigger points |
| **Pricing** | Very cheap (1/6 of Lambda@Edge) | More expensive |
| **Use case** | Cache key normalization, header manipulation, URL rewrites, redirects | A/B testing, user auth at edge, origin selection, image transformation |

> [!TIP]
> **Exam Pattern:** "Simple header manipulation at the edge" or "URL rewrite" → **CloudFront Functions**. "Complex logic needing network access" or "auth at the edge" or "dynamic origin selection" → **Lambda@Edge**.

---

## Lambda in a VPC

```
  ┌─────────────────────────────────────────────────┐
  │  Your VPC                                        │
  │                                                  │
  │  ┌───────────────────────────────────────────┐   │
  │  │  Private Subnet                           │   │
  │  │                                           │   │
  │  │  ┌──────────┐     ┌──────────┐            │   │
  │  │  │  Lambda  │────►│   RDS    │            │   │
  │  │  │  (ENI)   │     │          │            │   │
  │  │  └──────────┘     └──────────┘            │   │
  │  │       │                                    │   │
  │  │       │  To reach internet:                │   │
  │  │       ▼  NAT Gateway in public subnet     │   │
  │  │  ┌──────────┐     ┌──────────┐            │   │
  │  │  │  NAT GW  │────►│   IGW    │────► Internet  │
  │  │  └──────────┘     └──────────┘            │   │
  │  └───────────────────────────────────────────┘   │
  │                                                  │
  │  For AWS services without internet:              │
  │  ┌────────────────────────────────────────────┐  │
  │  │ VPC Endpoints (Gateway: S3, DynamoDB)      │  │
  │  │ VPC Endpoints (Interface: all other svc)   │  │
  │  └────────────────────────────────────────────┘  │
  └─────────────────────────────────────────────────┘
```

| Scenario | Lambda in VPC? | Internet Access |
|---|---|---|
| **Default** (no VPC) | ❌ | ✅ Has internet, ❌ no VPC resource access |
| **Lambda in VPC** | ✅ | ❌ Loses internet unless NAT Gateway configured |
| **Lambda in VPC + NAT GW** | ✅ | ✅ Internet via NAT + VPC resource access |
| **Lambda in VPC + VPC Endpoints** | ✅ | AWS services only (no internet, but free for S3/DynamoDB) |

> [!CAUTION]
> **Exam critical:** By default, Lambda is NOT in your VPC — it has internet access but cannot reach VPC resources. If you put Lambda in a VPC, it **loses internet access** unless you set up a **NAT Gateway**. For AWS service access, use **VPC Endpoints** (cheaper than NAT for S3 and DynamoDB — Gateway Endpoints are free).

---

## Lambda + RDS Proxy

```
  Without RDS Proxy:                    With RDS Proxy:
  ┌──────┐ ┌──────┐ ┌──────┐           ┌──────┐ ┌──────┐ ┌──────┐
  │ λ    │ │ λ    │ │ λ    │           │ λ    │ │ λ    │ │ λ    │
  └──┬───┘ └──┬───┘ └──┬───┘           └──┬───┘ └──┬───┘ └──┬───┘
     │        │        │                   │        │        │
     ▼        ▼        ▼                   └────────┼────────┘
  ┌─────────────────────────┐                       ▼
  │     RDS Database        │              ┌────────────────┐
  │  ❌ Connection overload │              │   RDS Proxy    │
  │  ❌ Too many connections│              │  (pool & share │
  └─────────────────────────┘              │   connections) │
                                           └───────┬────────┘
                                                   ▼
                                           ┌────────────────┐
                                           │  RDS Database  │
                                           │  ✅ Controlled │
                                           └────────────────┘

  ✅ Pools and shares DB connections
  ✅ Reduces failover time by 66%
  ✅ Supports IAM authentication to DB
  ✅ Must be in same VPC as RDS
  ✅ Fully managed by AWS
```

> [!TIP]
> **Exam Pattern:** "Lambda + RDS connection issues" or "too many database connections from Lambda" → **[[Amazon RDS|RDS Proxy]]**. Lambda must be in a VPC to use RDS Proxy. RDS Proxy also improves RDS failover time.

---

## Lambda + EFS

```
  ┌──────────────────────────────────────────────────────────┐
  │  VPC                                                      │
  │                                                          │
  │  ┌──────────┐                ┌──────────┐                │
  │  │ Lambda A │───── mount ───►│          │                │
  │  └──────────┘                │  Amazon  │                │
  │                              │   EFS    │                │
  │  ┌──────────┐                │ (shared  │                │
  │  │ Lambda B │───── mount ───►│  file    │                │
  │  └──────────┘                │  system) │                │
  │                              └──────────┘                │
  │                                                          │
  │  ✅ Persistent storage across invocations                │
  │  ✅ Shared across multiple functions                     │
  │  ✅ Scales automatically                                 │
  │  ⚠️ Lambda must be in VPC with EFS mount target          │
  └──────────────────────────────────────────────────────────┘
```

> [!NOTE]
> Lambda's `/tmp` storage (up to 10 GB) is ephemeral — lost between invocations. For **persistent, shared** file storage across Lambda functions → **[[Elastic File System (EFS)|EFS]]**. Lambda must be VPC-connected to mount EFS.

---

## Lambda Pricing

```
  Cost = Request Charges + Duration Charges

  Request Charges:
  • First 1M requests/month: FREE
  • Then: $0.20 per 1M requests

  Duration Charges:
  • Measured in GB-seconds (memory allocated × execution time)
  • First 400,000 GB-seconds/month: FREE
  • Then: ~$0.0000166667 per GB-second

  Example: 1M requests/month, 512 MB memory, 200ms average
  = 1M × 0.5 GB × 0.2s = 100,000 GB-seconds
  = FREE (within free tier)
```

---

## Key Lambda Limits

| Parameter | Value |
|---|---|
| **Max execution time** | **15 minutes** (900 seconds) |
| **Memory** | 128 MB – **10,240 MB** (10 GB) — CPU scales with memory |
| **Ephemeral storage (/tmp)** | 512 MB – **10,240 MB** (10 GB) |
| **Deployment package (zip)** | **50 MB** compressed, **250 MB** unzipped (including layers) |
| **Container image** | Up to **10 GB** |
| **Environment variables** | 4 KB total |
| **Concurrency (default)** | **1,000** per region (can request increase) |
| **Layers** | Up to **5 layers** per function |
| **Invocation payload (sync)** | 6 MB |
| **Invocation payload (async)** | 256 KB |
| **Burst concurrency** | 500–3,000 (varies by region) |

---

## Exam Cheat Sheet

> [!TIP]
> **Trigger phrases → Lambda:**
> - "Run code without servers" or "event-driven compute" → **Lambda**
> - "Process longer than 15 minutes" → **NOT Lambda** (use ECS/Fargate or Step Functions)
> - "Eliminate cold starts" → **Provisioned Concurrency** (set on Version/Alias, not $LATEST)
> - "Java cold starts" → **SnapStart**
> - "Lambda + RDS connection exhaustion" → **RDS Proxy**
> - "Lambda needs to access VPC resources" → **Deploy Lambda in VPC**
> - "Lambda in VPC can't reach internet" → **NAT Gateway** or **VPC Endpoints**
> - "Modify CloudFront requests (simple)" → **CloudFront Functions**
> - "Modify CloudFront requests (complex)" → **Lambda@Edge**
> - "Share libraries across Lambda functions" → **Lambda Layers** (up to 5)
> - "Canary deployment for Lambda" → **Aliases with weighted traffic shifting**
> - "SQS + Lambda duplicate processing" → Set **visibility timeout = 6× Lambda timeout**
> - "Persistent shared storage for Lambda" → **EFS** (Lambda must be in VPC)
>
> **Key facts:**
> - Max timeout: 15 min. Memory: up to 10 GB (CPU scales with memory). Concurrency: 1,000/region.
> - Reserved concurrency: free, guarantees + caps. Provisioned: costs money, eliminates cold starts.
> - Async invocation retries 2 times. Lambda Destinations preferred over DLQs.
> - Event Source Mapping (SQS, Kinesis, DDB Streams): Lambda **polls** the source.
> - Lambda in VPC needs NAT GW for internet. Use VPC Endpoints for S3/DynamoDB (free).
> - RDS Proxy: pools connections, IAM auth, faster failover. Must be in same VPC.
> - $LATEST = mutable. Versions = immutable. Aliases = pointers (support traffic shifting).
