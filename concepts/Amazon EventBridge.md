---
tags: [concept, serverless, integration, eventbridge, events, event-driven, automation]
aliases: [EventBridge, Amazon EventBridge, CloudWatch Events, Event Bus, Event-Driven Architecture]
date: 2026-06-07
---

# Amazon EventBridge

**Amazon EventBridge** is AWS's **serverless event bus** service for building event-driven architectures. It answers the question: **"When something happens, what should react?"** — routing events from AWS services, custom applications, and SaaS partners to targets like [[AWS Lambda]], [[AWS Step Functions]], [[Amazon SQS]], and more.

> [!IMPORTANT]
> **Core exam concept:** EventBridge is the **evolution of CloudWatch Events** (same underlying service, new name, more features). If the exam says "CloudWatch Events," think EventBridge. EventBridge = **event routing and automation**. It does NOT monitor metrics (that's [[Amazon CloudWatch]]) or track API calls (that's [[AWS CloudTrail]]).

---

## Architecture Overview

```
  ┌─────────────────────────────────────────────────────────────────────────┐
  │                        Amazon EventBridge                               │
  │                                                                         │
  │  EVENT SOURCES              EVENT BUS              TARGETS              │
  │  ┌──────────────┐          ┌──────────┐          ┌──────────────┐      │
  │  │ AWS Services │──────────►          │          │ Lambda       │      │
  │  │ (100+ svc)   │          │          │          │ Step Funct.  │      │
  │  └──────────────┘          │          │──Rules──►│ SQS / SNS    │      │
  │  ┌──────────────┐          │  Event   │          │ Kinesis      │      │
  │  │ Custom Apps  │──────────►  Bus     │          │ API Gateway  │      │
  │  │ (PutEvents)  │          │          │          │ API Dest.    │      │
  │  └──────────────┘          │          │          │ CloudWatch   │      │
  │  ┌──────────────┐          │          │          │ SSM / EC2    │      │
  │  │ SaaS Partners│──────────►          │          │ ECS Task     │      │
  │  │ (Zendesk,    │          └──────────┘          │ CodePipeline │      │
  │  │  Datadog...) │                                └──────────────┘      │
  │  └──────────────┘                                                       │
  │  ┌──────────────┐          ┌──────────┐                                │
  │  │  Scheduled   │──────────► Schedule │  (cron / rate expressions)     │
  │  │  Events      │          └──────────┘                                │
  │  └──────────────┘                                                       │
  └─────────────────────────────────────────────────────────────────────────┘
```

---

## Core Concepts

### Event

An **event** is a JSON object describing a state change. Every event has a standard **envelope** plus a service-specific **detail** section.

```json
{
  "version": "0",
  "id": "12345-abcde-67890",
  "source": "aws.ec2",
  "detail-type": "EC2 Instance State-change Notification",
  "account": "123456789012",
  "region": "us-east-1",
  "time": "2026-06-07T12:00:00Z",
  "resources": ["arn:aws:ec2:us-east-1:123456789012:instance/i-1234"],
  "detail": {
    "instance-id": "i-1234",
    "state": "stopped"
  }
}
```

| Field | Description |
|---|---|
| **source** | Identifies the service or application (e.g., `aws.ec2`, `myapp.orders`) |
| **detail-type** | Describes the event type (e.g., `EC2 Instance State-change Notification`) |
| **detail** | Service-specific payload (the actual event data) |
| **account / region** | Where the event originated |

### Event Bus

An **event bus** receives events and routes them via rules. There are **three types**:

| Type | Description |
|---|---|
| **Default event bus** | Automatically exists in every account. Receives all AWS service events. |
| **Custom event bus** | Created by you for your own application events (`PutEvents` API) |
| **Partner event bus** | Receives events from SaaS partners (Zendesk, Datadog, Auth0, Shopify, etc.) |

> [!NOTE]
> A single account can have **multiple event buses**. Use custom event buses to isolate application domains (e.g., `orders-bus`, `payments-bus`). Each bus has its own rules and permissions.

### Rules

A **rule** matches incoming events and routes them to one or more **targets**.

```
  Event arrives ──► Rule evaluates ──► Pattern match? ──► YES ──► Send to Target(s)
                                                         NO  ──► Event discarded
```

| Rule Feature | Description |
|---|---|
| **Event pattern** | JSON-based filter matching on source, detail-type, detail fields |
| **Schedule** | Cron expression (`cron(0 12 * * ? *)`) or rate (`rate(5 minutes)`) |
| **Targets** | Up to **5 targets per rule** |
| **Input transformation** | Reshape event JSON before sending to target |
| **Retry policy** | Configurable retry attempts (0–185) and max event age (1 min – 24 hours) |
| **Dead-letter queue** | [[Amazon SQS]] queue for events that fail delivery after retries |

### Event Patterns (Filtering)

Event patterns match on **content** using JSON structure. Only matching events are forwarded.

```json
// Match ALL EC2 instance state changes:
{
  "source": ["aws.ec2"],
  "detail-type": ["EC2 Instance State-change Notification"]
}

// Match ONLY when instance state = "terminated":
{
  "source": ["aws.ec2"],
  "detail-type": ["EC2 Instance State-change Notification"],
  "detail": {
    "state": ["terminated"]
  }
}

// Content-based filtering (advanced):
{
  "detail": {
    "price": [{ "numeric": [">", 100] }],       // numeric comparison
    "brand": [{ "prefix": "Ama" }],              // prefix match
    "status": [{ "anything-but": "cancelled" }], // negation
    "location": [{ "exists": true }]             // field existence
  }
}
```

> [!TIP]
> **Exam Pattern:** "Filter events by content before routing" → **EventBridge event pattern** with content-based filtering. This is more powerful than [[Amazon SNS]] message filtering (which only filters on message attributes, not body).

---

## Event Buses — Deep Dive

### Cross-Account Event Delivery

```
  Account A (Producer)                    Account B (Consumer)
  ┌──────────────────┐                   ┌──────────────────┐
  │                  │    Resource       │                  │
  │  Default Bus     │    Policy         │  Default Bus     │
  │                  │──────────────────►│                  │
  │  Rule:           │                   │  Rule:           │
  │  target = Bus B  │                   │  target = Lambda │
  └──────────────────┘                   └──────────────────┘

  • Account B's event bus must have a resource-based policy
    allowing Account A to PutEvents
  • Account A creates a rule with Account B's bus ARN as target
```

### Cross-Region Event Delivery

- EventBridge can send events to event buses in **other AWS regions**
- Use case: centralized event processing, multi-region event aggregation
- The target event bus in another region must allow the source via resource-based policy

> [!IMPORTANT]
> **Cross-account** and **cross-region** event routing is a key EventBridge differentiator vs [[Amazon SNS]] for building centralized event-driven architectures.

---

## Scheduling

EventBridge supports **two scheduling modes**:

### 1. Rule-Based Schedules (on Event Bus)

```
  cron(0 8 * * ? *)          ← Every day at 8:00 AM UTC
  rate(5 minutes)            ← Every 5 minutes
  rate(1 hour)               ← Every hour
```

### 2. EventBridge Scheduler (Dedicated Service)

| Feature | Rule-Based Schedule | EventBridge Scheduler |
|---|---|---|
| **Scale** | Limited by rules per bus | **Millions** of schedules |
| **One-time schedule** | ❌ | ✅ (`at(2026-06-07T15:00:00)`) |
| **Timezone** | UTC only | ✅ Any timezone |
| **Flexible time window** | ❌ | ✅ (spread invocations over a window) |
| **Dead-letter queue** | Via rule config | ✅ Built-in |
| **Use case** | Periodic rules | High-volume, one-time, or per-user scheduling |

> [!TIP]
> **Exam Pattern:** "Schedule Lambda to run every hour" → **EventBridge rule** with `rate(1 hour)`. "Schedule millions of one-time future events" → **EventBridge Scheduler**. "Replace cron on EC2" → EventBridge Schedule + Lambda.

---

## Targets

EventBridge can route events to **over 20 target types**:

| Category | Targets |
|---|---|
| **Compute** | [[AWS Lambda]], [[Containers on AWS\|ECS Task]], AWS Batch, EC2 (via SSM Run Command) |
| **Messaging** | [[Amazon SQS]], [[Amazon SNS]], [[Amazon Kinesis\|Kinesis Data Streams / Firehose]] |
| **Orchestration** | [[AWS Step Functions]], CodePipeline, CodeBuild |
| **API** | [[Amazon API Gateway]], API Destinations (any HTTP endpoint) |
| **Monitoring** | [[Amazon CloudWatch\|CloudWatch Logs]], CloudWatch Alarm actions |
| **Other** | Another EventBridge bus (cross-account/region), SSM Automation, Incident Manager |

### Input Transformation

Transform the event before it reaches the target:

```
  Original event:                  After InputTransformer:
  {                                {
    "detail": {                      "instanceId": "i-1234",
      "instance-id": "i-1234",       "message": "Instance i-1234 was stopped"
      "state": "stopped"            }
    }
  }
```

### API Destinations

Send events to **any HTTP endpoint** (external SaaS, webhooks, on-premises APIs):

```
  EventBridge ──► API Destination ──► https://api.example.com/webhook
                  │
                  ├── Connection (auth: API Key, OAuth, Basic)
                  ├── HTTP method (POST, PUT, GET, etc.)
                  ├── Rate limiting (invocations/sec)
                  └── Retry + DLQ
```

> [!NOTE]
> **API Destinations** are how EventBridge integrates with **any external service** — not just AWS. The Connection object stores authentication credentials securely in Secrets Manager.

---

## Event Replay & Archive

```
  Event Bus ──► Archive (store all or filtered events)
                  │
                  └──► Replay ──► Re-process events from archive
                                   (specify start/end time)

  Use cases:
  • Debug: replay past events to test new rule/target
  • Recovery: reprocess events after a downstream failure
  • Testing: replay production events in dev environment
```

| Feature | Description |
|---|---|
| **Archive** | Store events indefinitely or with a retention period. Can filter which events to archive. |
| **Replay** | Re-send archived events to the event bus. Specify time range. Events appear as replayed (not duplicate). |

> [!TIP]
> **Exam Pattern:** "Replay past events for debugging" or "reprocess events after failure" → **EventBridge Archive + Replay**.

---

## Schema Registry & Discovery

```
  ┌──────────────────────────────────────────────────┐
  │  Schema Registry                                  │
  │                                                    │
  │  • Auto-discovers event schemas from the bus       │
  │  • Generates code bindings (Java, Python, TS)     │
  │  • Version-controlled schemas                      │
  │  • OpenAPI 3.0 format                              │
  │                                                    │
  │  Schema Discovery ──► infers schema from events   │
  │  Schema Registry  ──► stores/versions schemas     │
  │  Code Bindings    ──► download typed event classes │
  └──────────────────────────────────────────────────┘
```

---

## EventBridge vs Other AWS Services

### EventBridge vs SNS

| Feature | EventBridge | [[Amazon SNS]] |
|---|---|---|
| **Model** | Event bus (many-to-many) | Pub/sub (topic-based) |
| **Filtering** | **Content-based** (any JSON field) | **Attribute-based** only |
| **Sources** | AWS + SaaS + Custom | Custom publishers |
| **Targets** | 20+ AWS services | SQS, Lambda, HTTP, Email, SMS |
| **Schema** | Schema registry + discovery | ❌ |
| **Archive/Replay** | ✅ | ❌ |
| **Cross-account** | ✅ Native | ✅ (topic policy) |
| **Throughput** | Soft limits, scales | Very high throughput |
| **Latency** | ~0.5s typical | Sub-second |
| **Use case** | Event routing, SaaS integration, automation | Fan-out notifications, high throughput |

> [!IMPORTANT]
> **Exam decision:** "Route events from AWS services or SaaS" → **EventBridge**. "Fan-out messages to many subscribers at high speed" → **SNS**. "Content-based filtering on event body" → **EventBridge**. "Filter on message attributes" → **SNS** is sufficient.

### EventBridge vs SQS

| Feature | EventBridge | [[Amazon SQS]] |
|---|---|---|
| **Model** | Event routing (push) | Message queue (pull) |
| **Delivery** | Push to targets | Consumer polls queue |
| **Retention** | Events are transient (archive optional) | Up to 14 days |
| **Ordering** | Best-effort | FIFO available |
| **Use case** | React to events | Decouple + buffer workloads |

---

## Common Architecture Patterns

### Pattern 1: Automated Remediation

```
  EC2 instance       EventBridge        Lambda            EC2
  state → stopped ──► Rule ───────────► (remediation) ──► restart
                      │                                    instance
                      ├── Pattern: source = aws.ec2
                      └── detail.state = "stopped"
```

### Pattern 2: Cross-Account Security Monitoring

```
  Account A                     Central Security Account
  ┌──────────┐                 ┌──────────────────────────┐
  │ GuardDuty│──► EventBridge  │  EventBridge (default)   │
  │ finding  │    default bus  │         │                 │
  └──────────┘       │        │    ┌────▼─────┐           │
                     └────────►    │   Rule   │           │
  Account B                   │    └────┬─────┘           │
  ┌──────────┐               │         ▼                  │
  │ Config   │──► EventBridge │    SNS → Security Team    │
  │ non-     │    default bus │    Lambda → Auto-remediate │
  │ compliant│       │        └──────────────────────────┘
  └──────────┘       │
                     └────────►
```

### Pattern 3: SaaS Integration

```
  Zendesk (new ticket)
        │
        ▼
  Partner Event Bus ──► Rule ──► Step Functions ──► Create Jira ticket
                                                ──► Notify Slack (API Dest)
                                                ──► Log to DynamoDB
```

### Pattern 4: Scheduled Data Pipeline

```
  EventBridge Schedule             Lambda              S3
  cron(0 2 * * ? *)  ──────────► (extract  ──────────► (data
  (daily at 2 AM)                  data)                 lake)
                                     │
                                     ▼
                                  DynamoDB
                                  (metadata)
```

### Pattern 5: Fan-Out with Event Bus

```
                               ┌──► Rule 1 ──► Lambda (process order)
  Order Service ──► Custom    │
  (PutEvents)       Event ────┼──► Rule 2 ──► SQS (inventory update)
                    Bus        │
                               └──► Rule 3 ──► SNS (customer notification)
```

---

## EventBridge Pipes

**EventBridge Pipes** provides point-to-point integrations between a **source** and a **target**, with optional filtering, enrichment, and transformation.

```
  Source ──► Filter ──► Enrichment ──► Target
    │                       │
    ├── SQS                 ├── Lambda
    ├── DynamoDB Streams    ├── Step Functions
    ├── Kinesis Streams     ├── API Gateway
    ├── Kafka / MSK         └── API Destination
    └── MQ (ActiveMQ/
        RabbitMQ)

  Key difference from Rules:
  • Pipes = point-to-point (1 source → 1 target)
  • Rules = event pattern matching (1 bus → many targets)
  • Pipes support POLLING sources (SQS, Streams, Kafka)
```

> [!NOTE]
> **EventBridge Pipes** simplifies the pattern of "poll from queue/stream → enrich → send to target" that previously required custom Lambda glue code. Think of it as a managed integration pipeline.

---

## Security

| Aspect | Details |
|---|---|
| **Resource-based policies** | Control which accounts/services can PutEvents to a bus |
| **IAM policies** | Control who can create rules, manage buses, etc. |
| **Encryption** | Events encrypted in transit (TLS). At rest with AWS-managed or CMK ([[Amazon S3\|KMS]]) |
| **VPC** | EventBridge is a regional public service — no VPC endpoint needed for AWS events |
| **Tag-based access** | Restrict rule management by tags |

---

## Key Limits

| Parameter | Value |
|---|---|
| **Rules per event bus** | 300 (soft limit) |
| **Targets per rule** | 5 |
| **Event size** | 256 KB max |
| **PutEvents batch** | Up to 10 entries per call |
| **Invocations per second** | Varies by region (soft limit, can be increased) |
| **Buses per account** | 100 |
| **Archive retention** | Indefinite or custom (days) |

---

## Exam Cheat Sheet

> [!TIP]
> **Trigger phrases → EventBridge:**
> - "React to AWS service events" (EC2 state change, S3 event, etc.) → **EventBridge rule** on default bus
> - "Schedule Lambda/task" or "replace cron job" → **EventBridge Schedule** (rule or Scheduler)
> - "Route events from SaaS partners" (Zendesk, Datadog) → **EventBridge partner event bus**
> - "Content-based event filtering" → **EventBridge event patterns** (more powerful than SNS)
> - "Cross-account event routing" → **EventBridge cross-account** (resource-based policy)
> - "Replay past events for debugging" → **EventBridge Archive + Replay**
> - "Send events to external HTTP API" → **EventBridge API Destinations**
> - "Automated remediation when resource changes" → EventBridge rule → Lambda / SSM
> - "Centralized event monitoring across accounts" → EventBridge cross-account bus aggregation
> - "Connect SQS/Kinesis/DynamoDB Stream to target" → **EventBridge Pipes**
> - "Millions of one-time future schedules" → **EventBridge Scheduler** (not rule-based)
>
> **Key facts:**
> - EventBridge = evolution of CloudWatch Events (same service, rebranded + enhanced).
> - Default event bus receives ALL AWS service events automatically.
> - Event pattern matching filters on JSON content (source, detail-type, detail fields).
> - Up to 5 targets per rule, 300 rules per bus (soft limits).
> - Max event size = 256 KB.
> - Archive + Replay = store events and re-send them later (debugging, recovery).
> - API Destinations = call any HTTP endpoint with auth (OAuth, API key, Basic).
> - EventBridge Pipes = point-to-point, Pipes poll; Rules = pattern-based, bus pushes.
> - EventBridge ≠ [[Amazon CloudWatch]] (metrics) ≠ [[AWS CloudTrail]] (API audit).
