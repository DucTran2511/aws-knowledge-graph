---
tags: [concept, serverless, compute, overview, hub]
aliases: [Serverless, AWS Serverless, Serverless Architecture]
date: 2026-05-29
---

# Serverless on AWS — Overview

**Serverless** is a cloud execution model where AWS manages the infrastructure entirely — provisioning, scaling, patching, and high availability. You write code and define configuration; AWS handles everything else. You **pay only for what you use** (per request, per second of compute, per GB of storage).

> [!IMPORTANT]
> **Core exam concept:** "Serverless" does NOT mean "no servers." It means **you don't manage servers**. AWS still runs servers behind the scenes.

---

## The AWS Serverless Ecosystem

```
  ┌──────────────────────────────────────────────────────────────────┐
  │                   AWS Serverless Ecosystem                       │
  │                                                                  │
  │  COMPUTE          DATABASE         API              INTEGRATION  │
  │  ┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐   │
  │  │  Lambda  │    │ DynamoDB │    │   API    │    │   SQS    │   │
  │  │          │    │          │    │ Gateway  │    │   SNS    │   │
  │  │  Fargate │    │  Aurora  │    │          │    │EventBridg│   │
  │  │          │    │Serverless│    │          │    │          │   │
  │  └──────────┘    └──────────┘    └──────────┘    └──────────┘   │
  │                                                                  │
  │  STORAGE          ORCHESTRATION   IDENTITY        DEPLOYMENT     │
  │  ┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐   │
  │  │    S3    │    │  Step    │    │ Cognito  │    │   SAM    │   │
  │  │          │    │Functions │    │          │    │          │   │
  │  │   EFS    │    │          │    │          │    │          │   │
  │  └──────────┘    └──────────┘    └──────────┘    └──────────┘   │
  └──────────────────────────────────────────────────────────────────┘

  ✅ No server management       ✅ Auto-scaling built-in
  ✅ Pay-per-use pricing         ✅ High availability by default
```

---

## Serverless Service Deep Dives

Each service below has its own dedicated deep-dive note:

| Service | Category | Note |
|---|---|---|
| **AWS Lambda** | Serverless Compute | [[AWS Lambda]] |
| **Amazon DynamoDB** | Serverless NoSQL DB | [[Amazon DynamoDB]] |
| **Amazon API Gateway** | API Management | [[Amazon API Gateway]] |
| **AWS Step Functions** | Workflow Orchestration | [[AWS Step Functions]] |
| **Amazon Cognito** | Identity & Access | [[Amazon Cognito]] |

Related services already documented:
- [[Amazon EventBridge]] — Serverless event bus (routing + scheduling)
- [[Amazon SQS]] — Message queuing (decoupling)
- [[Amazon SNS]] — Pub/sub notifications
- [[Amazon Kinesis]] — Real-time data streaming
- [[Amazon S3]] — Object storage
- [[Amazon CloudFront]] — CDN
- [[Containers on AWS]] — ECS, Fargate, ECR, EKS

---

## Serverless Architecture Patterns

### Pattern 1: REST API (Most Common)

```
  ┌────────┐     ┌───────────┐     ┌──────────┐     ┌──────────┐
  │ Client │────►│    API    │────►│  Lambda  │────►│ DynamoDB │
  │        │◄────│  Gateway  │◄────│          │◄────│          │
  └────────┘     └───────────┘     └──────────┘     └──────────┘

  ✅ Fully serverless, auto-scaling, pay-per-request
  ✅ The "classic" serverless architecture
```

### Pattern 2: Static Site + Serverless Backend

```
  ┌────────┐     ┌────────────┐     ┌──────────┐
  │ Client │────►│ CloudFront │────►│    S3    │  (static content)
  │        │     └────────────┘     └──────────┘
  │        │
  │        │     ┌───────────┐     ┌──────────┐     ┌──────────┐
  │        │────►│    API    │────►│  Lambda  │────►│ DynamoDB │
  │        │     │  Gateway  │     │          │     │          │
  └────────┘     └───────────┘     └──────────┘     └──────────┘

  ✅ S3 + CloudFront for static assets (HTML, CSS, JS)
  ✅ API Gateway + Lambda for dynamic API calls
  ✅ Cognito for user authentication
```

### Pattern 3: Event-Driven Processing

```
  ┌──────┐  upload   ┌──────┐  trigger  ┌──────────┐  store   ┌──────────┐
  │ User │──────────►│  S3  │─────────►│  Lambda  │────────►│ DynamoDB │
  └──────┘           └──────┘          │(thumbnail│         └──────────┘
                                        │ resize)  │
                                        └──────────┘
```

### Pattern 4: Async Processing with SQS

```
  ┌───────────┐     ┌──────┐     ┌──────────┐     ┌──────────┐
  │    API    │────►│ SQS  │────►│  Lambda  │────►│ DynamoDB │
  │  Gateway  │     │Queue │     │(consumer)│     │          │
  └───────────┘     └──────┘     └──────────┘     └──────────┘

  ✅ Decoupled — API responds immediately
  ✅ SQS buffers during traffic spikes
  ✅ Lambda scales with queue depth
```

### Pattern 5: Orchestrated Workflow

```
  ┌──────────┐    trigger    ┌──────────────┐
  │EventBridg│──────────────►│ Step         │
  │ Rule     │               │ Functions    │
  └──────────┘               │              │
                              │ ┌──────────┐│
                              │ │ Validate ││
                              │ │ (Lambda) ││
                              │ └────┬─────┘│
                              │      ▼      │
                              │ ┌──────────┐│
                              │ │ Process  ││
                              │ │ (Lambda) ││
                              │ └────┬─────┘│
                              │      ▼      │
                              │ ┌──────────┐│
                              │ │ Notify   ││
                              │ │ (SNS)    ││
                              │ └──────────┘│
                              └──────────────┘
```

---

## Serverless vs Containers vs EC2

| Aspect | Lambda (Serverless) | [[Containers on AWS|ECS/Fargate]] (Containers) | EC2 (Traditional) |
|---|---|---|---|
| **Scaling** | Automatic (per request) | Automatic (per task/service) | Manual (ASG) |
| **Max duration** | 15 minutes | Unlimited | Unlimited |
| **Startup time** | Milliseconds (warm) / seconds (cold) | Seconds–minutes | Minutes |
| **State** | Stateless | Stateful possible | Stateful |
| **Pricing** | Per request + duration | Per task (vCPU + memory) | Per instance (hourly) |
| **Ops overhead** | None | Low (Fargate) / Medium (EC2) | High |
| **Use case** | Event-driven, short tasks, APIs | Long-running, microservices | Full control, legacy apps |

> [!TIP]
> **Exam decision tree:**
> - Short, event-driven task (< 15 min)? → **Lambda**
> - Long-running process, containerized? → **[[Containers on AWS|ECS/Fargate]]**
> - Need full OS control, GPU, custom networking? → **EC2**

---

## AWS SAM (Serverless Application Model)

**AWS SAM** is an open-source framework for building serverless applications. It extends CloudFormation with simplified syntax for [[AWS Lambda]], [[Amazon API Gateway]], [[Amazon DynamoDB]], and [[AWS Step Functions]].

```
  SAM Template (template.yaml):
  ─────────────────────────────
  Transform: AWS::Serverless-2016-10-31   ← SAM magic

  Resources:
    MyFunction:
      Type: AWS::Serverless::Function     ← simplified Lambda
      Properties:
        Handler: app.handler
        Runtime: python3.12
        Events:
          ApiEvent:
            Type: Api                     ← auto-creates API Gateway
            Properties:
              Path: /hello
              Method: get

    MyTable:
      Type: AWS::Serverless::SimpleTable  ← simplified DynamoDB

  CLI Commands:
  • sam init    → scaffold new project
  • sam build   → build deployment artifacts
  • sam deploy  → deploy via CloudFormation
  • sam local   → test Lambda locally (Docker)
```

> [!NOTE]
> SAM is lightly tested on SAA-C03. Know that it uses **CloudFormation Transform**, simplifies serverless resource definitions, and provides **local testing** with `sam local invoke`.

---

## Exam Cheat Sheet

> [!TIP]
> **Trigger phrases → Serverless Services:**
> - "Run code without servers" or "event-driven compute" → **[[AWS Lambda]]**
> - "Serverless NoSQL database" → **[[Amazon DynamoDB]]**
> - "Create/manage APIs" or "expose Lambda as REST API" → **[[Amazon API Gateway]]**
> - "Orchestrate multiple Lambda functions" or "visual workflow" → **[[AWS Step Functions]]**
> - "User sign-up/sign-in" or "authentication for web/mobile" → **[[Amazon Cognito|Cognito User Pools]]**
> - "Temporary AWS credentials for app users" → **[[Amazon Cognito|Cognito Identity Pools]]**
> - "Serverless containers" → **[[Containers on AWS|Fargate]]** (NOT Lambda)
> - "Long-running workflow (> 15 min)" → **[[AWS Step Functions]]** (not Lambda)
> - "Deploy serverless app with CloudFormation" → **AWS SAM**
>
> **Key facts:**
> - Serverless ≠ no servers. It means YOU don't manage them.
> - Lambda = functions. Fargate = containers. Both are serverless compute.
> - SAM uses CloudFormation Transform: `AWS::Serverless-2016-10-31`.
> - See each service's dedicated note for detailed limits and exam tips.
