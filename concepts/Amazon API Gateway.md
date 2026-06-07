---
tags: [concept, serverless, api, gateway, rest, http, websocket]
aliases: [API Gateway, Amazon API Gateway, APIGW, REST API, HTTP API, WebSocket API]
date: 2026-05-29
---

# Amazon API Gateway

**Amazon API Gateway** is a fully managed service to create, publish, maintain, monitor, and secure APIs at any scale. It acts as the **"front door"** for applications to access backend services like [[AWS Lambda]], EC2, or any HTTP endpoint. API Gateway is a core component of the [[Serverless on AWS]] ecosystem.

> [!IMPORTANT]
> **Core exam concept:** API Gateway is the standard way to **expose Lambda functions as REST/HTTP APIs**. "Create a serverless API" → **API Gateway + Lambda**. "Throttle API requests" or "cache API responses" → **API Gateway features**.

---

## API Types

```
  ┌──────────────────────────────────────────────────────────────┐
  │                    API Gateway — 3 API Types                  │
  │                                                              │
  │  ┌─────────────┐   ┌─────────────┐   ┌──────────────────┐  │
  │  │  REST API   │   │  HTTP API   │   │  WebSocket API   │  │
  │  │             │   │             │   │                  │  │
  │  │ Full-feature│   │ Simpler,    │   │ Real-time,       │  │
  │  │ Caching     │   │ faster,     │   │ two-way          │  │
  │  │ WAF         │   │ cheaper     │   │ communication    │  │
  │  │ Usage Plans │   │ JWT auth    │   │                  │  │
  │  │ API Keys    │   │ Lambda/HTTP │   │ Chat, gaming,    │  │
  │  │ Resource    │   │ proxy only  │   │ streaming,       │  │
  │  │ Policies    │   │             │   │ notifications    │  │
  │  └─────────────┘   └─────────────┘   └──────────────────┘  │
  │  $$$                $  (70% cheaper)  Per-message pricing   │
  └──────────────────────────────────────────────────────────────┘
```

| Feature | REST API | HTTP API | WebSocket API |
|---|---|---|---|
| **Protocol** | REST (HTTP) | REST (HTTP) | WebSocket |
| **Caching** | ✅ | ❌ | N/A |
| **WAF** | ✅ | ❌ | ❌ |
| **Usage Plans / API Keys** | ✅ | ❌ | ❌ |
| **Resource Policies** | ✅ | ❌ | ❌ |
| **Request/Response Transformation** | ✅ (Mapping Templates) | ❌ | ❌ |
| **Auth** | IAM, Cognito, Lambda Authorizer | IAM, JWT (native), Lambda Authorizer | IAM, Lambda Authorizer |
| **Private Integrations (VPC Link)** | ✅ (NLB) | ✅ (ALB, NLB, Cloud Map) | ❌ |
| **Cost** | $$$ | $ (70% cheaper) | Per-message + connection |
| **Use case** | Full-featured APIs | Simple proxy, cost-sensitive | Real-time bidirectional |

> [!TIP]
> **Exam Pattern:** "Cheapest API Gateway option" or "simple Lambda proxy" → **HTTP API**. "Need caching, WAF, or usage plans" → **REST API**. "Real-time bidirectional communication" (chat, gaming) → **WebSocket API**.

---

## Endpoint Types

```
  EDGE-OPTIMIZED (default for REST API):
  ──────────────────────────────────────
  ┌────────┐     ┌────────────┐     ┌───────────┐     ┌──────────┐
  │ Client │────►│ CloudFront │────►│ API GW    │────►│ Backend  │
  │(global)│     │    POP     │     │(one region)│     │ (Lambda) │
  └────────┘     └────────────┘     └───────────┘     └──────────┘

  • Requests routed through CloudFront edge locations
  • Reduces latency for geographically distributed clients
  • CloudFront distribution managed by API Gateway (not yours)

  REGIONAL:
  ─────────
  ┌────────┐                        ┌───────────┐     ┌──────────┐
  │ Client │───────────────────────►│ API GW    │────►│ Backend  │
  │(same   │                        │(same      │     │          │
  │ region)│                        │ region)   │     │          │
  └────────┘                        └───────────┘     └──────────┘

  • No CloudFront distribution
  • Best for clients in same region
  • Combine with your OWN CloudFront for custom caching/WAF

  PRIVATE:
  ────────
  ┌────────┐     ┌──────────────┐   ┌───────────┐     ┌──────────┐
  │ Client │────►│ VPC Endpoint │──►│ API GW    │────►│ Backend  │
  │(in VPC)│     │ (PrivateLink)│   │(private)  │     │          │
  └────────┘     └──────────────┘   └───────────┘     └──────────┘

  • Accessible ONLY from within your VPC
  • Uses interface VPC endpoint (AWS PrivateLink)
  • Secured by VPC endpoint policies + resource policies
  • No public internet exposure
```

> [!WARNING]
> **Exam trap:** "API accessible only from within the VPC" → **Private endpoint** (uses VPC Interface Endpoint). "API for global users with low latency" → **Edge-Optimized** (default). "API in same region, want custom CloudFront" → **Regional**.

---

## Integration Types

```
  Client ──► API Gateway ──► Integration ──► Backend

  ┌─────────────────────────────────────────────────────────┐
  │  Lambda Proxy (MOST COMMON):                             │
  │  API GW passes entire request as-is → Lambda             │
  │  Lambda returns: { statusCode, headers, body }           │
  │  NO mapping templates needed                              │
  │                                                          │
  │  HTTP Proxy:                                              │
  │  API GW forwards request to HTTP endpoint as-is          │
  │                                                          │
  │  AWS Service (direct):                                    │
  │  API GW → SQS (SendMessage)                              │
  │  API GW → Step Functions (StartExecution)                │
  │  API GW → DynamoDB (PutItem)                             │
  │  → NO Lambda function needed! Reduces cost & latency.    │
  │                                                          │
  │  Mock:                                                    │
  │  API GW returns a hardcoded response (testing, CORS)     │
  │                                                          │
  │  VPC Link:                                                │
  │  API GW → NLB/ALB → private resources in VPC             │
  └─────────────────────────────────────────────────────────┘
```

| Integration | Description | When to Use |
|---|---|---|
| **Lambda Proxy** | Passes entire request to Lambda. Most common. | Standard serverless API |
| **Lambda Custom** | Uses mapping templates (VTL) to transform request/response. | Legacy or complex transformations |
| **HTTP Proxy** | Passes request to HTTP endpoint. | Proxy to existing HTTP services |
| **HTTP Custom** | Uses mapping templates for HTTP endpoint. | Transform requests to/from HTTP backend |
| **AWS Service** | Direct integration with AWS services. | SQS, Step Functions, DynamoDB — **skip Lambda** |
| **Mock** | Returns hardcoded response. | Testing, CORS preflight |
| **VPC Link** | Connect to private VPC resources. | Internal microservices behind NLB/ALB |

> [!TIP]
> **Exam Pattern:** "Reduce cost by removing Lambda" or "send messages to SQS via API without Lambda" → **AWS Service integration**. "Connect API Gateway to services inside a VPC" → **VPC Link** (REST API → NLB; HTTP API → ALB/NLB).

---

## Stages & Deployments

```
  API Gateway API
       │
       ├── Stage: dev    → https://api-id.execute-api.region.amazonaws.com/dev
       │     └── Stage Variables: { "table": "dev-orders", "alias": "DEV" }
       │
       ├── Stage: staging → https://api-id.execute-api.region.amazonaws.com/staging
       │
       └── Stage: prod   → https://api-id.execute-api.region.amazonaws.com/prod
             └── Stage Variables: { "table": "prod-orders", "alias": "PROD" }
             └── Canary: 10% traffic → new deployment

  Stages allow:
  • Different configurations per environment
  • Stage variables (passed to backend as context)
  • Canary deployments (shift % of traffic to new version)
  • Rollback to previous deployment
```

> [!NOTE]
> **Stage variables** can be used to reference different Lambda aliases, DynamoDB tables, or HTTP endpoints per stage. Example: `${stageVariables.alias}` in the Lambda ARN to invoke the correct alias (DEV/PROD).

---

## Throttling

```
  ┌──────────────────────────────────────────────────────────────┐
  │  Throttling Hierarchy (lowest limit wins):                    │
  │                                                              │
  │  1. Account-level:  10,000 rps across ALL APIs (soft limit) │
  │  2. Stage-level:    Per-method throttle limits               │
  │  3. Usage Plan:     Per-API-key limits (for 3rd parties)     │
  │                                                              │
  │  Token Bucket Algorithm:                                      │
  │  • Steady-state rate: requests per second                    │
  │  • Burst: max concurrent requests (bucket capacity)          │
  │                                                              │
  │  When limit exceeded → 429 Too Many Requests                 │
  └──────────────────────────────────────────────────────────────┘
```

| Level | Default | Configurable? |
|---|---|---|
| **Account-level** | 10,000 rps, 5,000 burst | Yes (service quota increase) |
| **Stage-level** | Inherits account-level | Yes (per-method override) |
| **Usage Plan** | No default | Yes (per-API-key rate + burst + quota) |

> [!WARNING]
> **Exam trap:** API Gateway throttling applies **across all APIs in a region for your account**. One API with a traffic spike can throttle all other APIs. Solution: set **per-method or per-stage throttle limits** to protect critical APIs.

---

## Caching

```
  ┌────────┐     ┌──────────────────────┐     ┌──────────┐
  │ Client │────►│    API Gateway       │────►│ Backend  │
  │        │     │  ┌──────────────┐    │     │ (Lambda) │
  │        │     │  │    Cache     │    │     │          │
  │        │◄────│  │  HIT → ✅   │    │     │          │
  │        │     │  │  MISS → call │───►│────►│          │
  │        │     │  │  backend     │    │     │          │
  │        │     │  └──────────────┘    │     │          │
  └────────┘     └──────────────────────┘     └──────────┘
```

| Parameter | Detail |
|---|---|
| **Availability** | **REST API only** (not HTTP API) |
| **Capacity** | 0.5 GB – **237 GB** |
| **Default TTL** | **300 seconds** (5 min) |
| **TTL Range** | 0 (disabled) – 3,600 seconds (1 hour) |
| **Per-method override** | ✅ Enable/disable caching per method |
| **Cache invalidation** | Header `Cache-Control: max-age=0` (requires IAM authorization) |
| **Encryption** | Optional encryption of cached data |
| **Cost** | 💰 Additional charge based on cache size |

> [!TIP]
> **Exam Pattern:** "Reduce backend calls" or "cache API responses" → **API Gateway Caching** (REST API only). "Client needs to bypass cache" → `Cache-Control: max-age=0` header with proper IAM policy.

---

## Authentication & Authorization

```
  ┌──────────────────────────────────────────────────────────────┐
  │  3 Auth Methods for API Gateway:                              │
  │                                                              │
  │  1. IAM Authorization:                                       │
  │  ┌──────┐  SigV4 signed   ┌──────────┐                      │
  │  │ AWS  │────────────────►│ API GW   │   (IAM policy check) │
  │  │ User │                 │          │                       │
  │  └──────┘                 └──────────┘                       │
  │  Best for: AWS-to-AWS calls, cross-account                   │
  │                                                              │
  │  2. Cognito User Pool Authorizer:                            │
  │  ┌──────┐  1.Auth  ┌──────────┐  2.JWT  ┌──────────┐       │
  │  │ User │────────►│ Cognito  │───────►│ API GW   │       │
  │  │      │         │ User Pool│        │(validates │       │
  │  └──────┘         └──────────┘        │  JWT)     │       │
  │  Best for: end-user auth, least overhead                     │
  │                                                              │
  │  3. Lambda Authorizer (Custom):                              │
  │  ┌──────┐  token  ┌──────────┐  IAM    ┌──────────┐       │
  │  │ User │───────►│ Lambda   │ policy  │ API GW   │       │
  │  │      │        │Authorizer│───────►│          │       │
  │  └──────┘        └──────────┘        └──────────┘       │
  │  Best for: custom tokens (OAuth 2.0, SAML, 3rd party)       │
  └──────────────────────────────────────────────────────────────┘
```

| Method | Use Case | How It Works |
|---|---|---|
| **IAM** | AWS users/roles, cross-account | SigV4 signed requests. IAM policy evaluated. |
| **Cognito Authorizer** | End users (web/mobile). **Least overhead.** | Validates JWT from [[Amazon Cognito]] User Pool. |
| **Lambda Authorizer** | Custom/3rd-party auth (OAuth, SAML, custom header) | Lambda function evaluates token, returns IAM policy. Results cached (TTL). |
| **API Keys** | **NOT for auth** — usage tracking/throttling only | Identifies callers for Usage Plans. No security. |

> [!CAUTION]
> **Exam critical:** API Keys are NOT an authentication mechanism — they are for **usage metering and throttling**. For real auth: **IAM** (AWS callers), **Cognito** (end users, simplest/least overhead), or **Lambda Authorizer** (custom token validation).

### Lambda Authorizer Types

| Type | Input | Use Case |
|---|---|---|
| **Token-based** | Authorization header (Bearer token) | OAuth 2.0, JWT from 3rd party |
| **Request-based** | Headers, query strings, stage vars, context | Multi-parameter auth decisions |

---

## Usage Plans & API Keys

```
  ┌──────────────────────────────────────────────────────────────┐
  │  API Key: abc123xyz                                          │
  │  │                                                          │
  │  └──► Usage Plan: "Bronze"                                  │
  │        • Throttle: 100 rps, burst 200                       │
  │        • Quota: 10,000 requests/month                       │
  │        • Associated APIs: [OrdersAPI, InventoryAPI]         │
  │                                                              │
  │  API Key: def456uvw                                          │
  │  │                                                          │
  │  └──► Usage Plan: "Gold"                                    │
  │        • Throttle: 5,000 rps, burst 10,000                  │
  │        • Quota: unlimited                                    │
  │        • Associated APIs: [OrdersAPI, InventoryAPI]         │
  └──────────────────────────────────────────────────────────────┘

  Use case: SaaS API platform with different tiers
  API Key sent in header: x-api-key
```

---

## CORS (Cross-Origin Resource Sharing)

```
  Browser (origin: app.example.com) → API Gateway (api.example.com)
                                       │
                                       ▼
                              Must return CORS headers:
                              Access-Control-Allow-Origin
                              Access-Control-Allow-Methods
                              Access-Control-Allow-Headers

  Enable CORS on API Gateway:
  • For Lambda Proxy: Lambda function MUST return CORS headers in response
  • For non-proxy: Configure CORS in API Gateway console (adds OPTIONS method)
```

> [!NOTE]
> With **Lambda Proxy** integration, API Gateway passes everything through — so your **Lambda function** must return CORS headers in the response. With non-proxy integration, API Gateway can add them via the OPTIONS method mock integration.

---

## API Gateway + Other Services

| Integration | Pattern |
|---|---|
| **API GW + Lambda** | Classic serverless API (most common) |
| **API GW + [[Amazon SQS]]** | Async processing — API queues work, returns 200 immediately |
| **API GW + [[AWS Step Functions]]** | Start workflow execution via API call |
| **API GW + [[Amazon Kinesis]]** | Ingest streaming data via REST API |
| **API GW + [[Amazon DynamoDB]]** | Direct CRUD without Lambda (AWS Service integration) |
| **API GW + [[Amazon Cognito]]** | User auth with Cognito Authorizer |
| **API GW + WAF** | Protect API from SQL injection, XSS, rate limiting (REST API only) |

---

## Key API Gateway Limits

| Parameter | Value |
|---|---|
| **Account throttle** | **10,000 rps** (soft limit, per region) |
| **Burst limit** | **5,000** concurrent requests |
| **Max payload size** | **10 MB** (REST and HTTP API) |
| **Max timeout** | **29 seconds** (integration timeout) |
| **WebSocket message size** | **128 KB** |
| **WebSocket connection duration** | **2 hours** max idle, no max for active |
| **Cache capacity** | 0.5 GB – **237 GB** |
| **Cache TTL** | 0 – **3,600 seconds** |
| **Custom domain names** | ✅ (with ACM certificate) |

> [!WARNING]
> **Exam trap:** API Gateway has a **29-second timeout** — if your backend takes longer, the request will fail. For long operations, use an **async pattern**: API Gateway → SQS → Lambda (return 202 Accepted immediately).

---

## Exam Cheat Sheet

> [!TIP]
> **Trigger phrases → API Gateway:**
> - "Create/manage APIs" or "expose Lambda as REST API" → **API Gateway**
> - "Cheapest API Gateway" or "simple Lambda proxy" → **HTTP API** (70% cheaper)
> - "Need caching, WAF, usage plans" → **REST API**
> - "Real-time bidirectional" (chat, gaming) → **WebSocket API**
> - "API accessible only from VPC" → **Private endpoint + VPC Interface Endpoint**
> - "Cache API responses" → **API Gateway Caching** (REST API only, 0.5–237 GB)
> - "Throttle API requests" → **Usage Plans + API Keys** (throttle per client)
> - "Auth with least overhead" → **Cognito User Pool Authorizer**
> - "Custom auth logic / 3rd party tokens" → **Lambda Authorizer**
> - "Send to SQS without Lambda" → **AWS Service integration**
> - "Connect API to private VPC resources" → **VPC Link** (NLB for REST, ALB/NLB for HTTP)
> - "API Gateway timeout" → **29 seconds max** → use async pattern for longer ops
>
> **Key facts:**
> - REST API = full features (caching, WAF, usage plans). HTTP API = simpler, 70% cheaper.
> - Endpoint types: Edge-Optimized (default, CloudFront), Regional, Private (VPC only).
> - Default throttle: 10,000 rps account-wide. Exceeded → 429 error.
> - API Keys ≠ authentication. They're for usage tracking/throttling only.
> - Cognito Authorizer = simplest auth for end users. Lambda Authorizer = custom auth.
> - Max payload: 10 MB. Max integration timeout: 29 seconds.
> - CORS with Lambda Proxy: Lambda must return CORS headers (API GW doesn't add them).
> - Stage variables let you point to different backends per environment.
> - Canary deployments: shift % of stage traffic to new deployment.
