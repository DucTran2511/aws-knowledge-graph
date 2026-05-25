---
tags: [concept, messaging, migration, open-source, managed-service, hybrid]
aliases: [Amazon MQ, ActiveMQ, RabbitMQ, Message Broker, AWS MQ]
date: 2026-05-25
---

# Amazon MQ

**Amazon MQ** is a **managed message broker service** for **Apache ActiveMQ** and **RabbitMQ**. It is designed for companies migrating from on-premises message brokers to AWS **without rewriting application code**.

> [!IMPORTANT]
> **Core exam concept:** Amazon MQ is the answer when the question involves **migrating an existing on-premises message broker** (ActiveMQ, RabbitMQ) to AWS with **no code changes**. If there's no legacy broker, use [[Amazon SQS|SQS]] + [[Amazon SNS|SNS]] instead (cloud-native, better scaling).

---

## When to Use Amazon MQ vs SQS/SNS

```
  Decision Tree:

  "I need messaging on AWS..."

  ┌───────────────────────────────────────────────────────┐
  │  Do you have an EXISTING on-premises message broker?  │
  │  (ActiveMQ, RabbitMQ, MQTT, AMQP, STOMP, OpenWire)  │
  └───────────────────┬──────────────────┬────────────────┘
                      │                  │
                     YES                 NO
                      │                  │
                      ▼                  ▼
              ┌──────────────┐   ┌──────────────────────┐
              │  Amazon MQ   │   │  SQS + SNS           │
              │              │   │                      │
              │ • No code    │   │ • Cloud-native       │
              │   changes    │   │ • Virtually unlimited│
              │ • Same       │   │   scaling            │
              │   protocols  │   │ • Serverless         │
              │ • MQTT, AMQP │   │ • Higher availability│
              │   STOMP,     │   │ • Lower ops overhead │
              │   OpenWire   │   │                      │
              └──────────────┘   └──────────────────────┘
```

| Aspect | Amazon MQ | SQS + SNS |
|---|---|---|
| **Best for** | Migrating existing brokers | New cloud-native applications |
| **Protocols** | MQTT, AMQP, STOMP, OpenWire, WSS | AWS proprietary API |
| **Scaling** | Limited (runs on EC2-like instances) | Virtually unlimited, serverless |
| **Management** | Managed but NOT serverless | Fully serverless |
| **Queue + Topic** | Both (built into the broker) | SQS = Queue, SNS = Topic |
| **Code changes** | None required | Must use AWS SDK |

> [!WARNING]
> **Exam trap:** Amazon MQ does NOT scale as elastically as SQS/SNS. It runs on **provisioned broker instances** (like EC2). If the question asks for "unlimited scaling" or "serverless messaging" → **SQS/SNS**, not Amazon MQ.

---

## Supported Protocols

Amazon MQ supports **industry-standard** messaging protocols — this is its key differentiator from SQS/SNS:

| Protocol | Description | Common Use |
|---|---|---|
| **MQTT** | Lightweight IoT messaging | IoT devices, mobile apps |
| **AMQP** | Advanced Message Queuing Protocol | Enterprise messaging |
| **STOMP** | Simple Text Oriented Messaging | Web-based messaging |
| **OpenWire** | Apache ActiveMQ native protocol | Java/JMS applications |
| **WSS** | WebSocket Secure | Real-time web communication |

> [!TIP]
> **Exam Pattern:** If the question mentions **MQTT**, **AMQP**, **STOMP**, **OpenWire**, or **JMS** → the answer is **Amazon MQ**. SQS/SNS do NOT support these protocols.

---

## Architecture

### Single-Instance Broker (Development)

```
  ┌────────────────────────────────────────────────┐
  │                    AZ-1                         │
  │                                                │
  │  Producers ──►  ┌──────────────┐  ──► Consumers│
  │                 │  ActiveMQ /  │               │
  │                 │  RabbitMQ    │               │
  │                 │  Broker      │               │
  │                 │              │               │
  │                 │  EBS Storage │               │
  │                 └──────────────┘               │
  └────────────────────────────────────────────────┘

  ⚠️ Single point of failure — NOT for production
```

### Active/Standby Broker (Production — ActiveMQ)

```
  ┌────────────────────┐        ┌────────────────────┐
  │       AZ-1          │        │       AZ-2          │
  │                    │        │                    │
  │  ┌──────────────┐  │        │  ┌──────────────┐  │
  │  │  ActiveMQ    │  │        │  │  ActiveMQ    │  │
  │  │  ACTIVE      │  │◄──────►│  │  STANDBY     │  │
  │  │  Broker      │  │ failover│  │  Broker      │  │
  │  └──────┬───────┘  │        │  └──────┬───────┘  │
  │         │          │        │         │          │
  │  ┌──────▼───────┐  │        │  ┌──────▼───────┐  │
  │  │  Amazon EFS  │──│────────│──│  Amazon EFS  │  │
  │  │  (shared     │  │ shared │  │  (shared     │  │
  │  │   storage)   │  │ storage│  │   storage)   │  │
  │  └──────────────┘  │        │  └──────────────┘  │
  └────────────────────┘        └────────────────────┘

  ✅ Multi-AZ with automatic failover
  ✅ Shared storage via Amazon EFS
```

### RabbitMQ Cluster (Production)

```
  ┌────────────────────┐  ┌────────────────────┐  ┌────────────────────┐
  │       AZ-1          │  │       AZ-2          │  │       AZ-3          │
  │  ┌──────────────┐  │  │  ┌──────────────┐  │  │  ┌──────────────┐  │
  │  │  RabbitMQ    │  │  │  │  RabbitMQ    │  │  │  │  RabbitMQ    │  │
  │  │  Node 1      │◄─│──│─►│  Node 2      │◄─│──│─►│  Node 3      │  │
  │  └──────────────┘  │  │  └──────────────┘  │  │  └──────────────┘  │
  └────────────────────┘  └────────────────────┘  └────────────────────┘

  ✅ Cluster deployment across AZs
  ✅ Queues replicated across nodes
```

> [!NOTE]
> ActiveMQ HA = **Active/Standby** with shared EFS storage. RabbitMQ HA = **Cluster deployment** with queue replication across nodes.

---

## Amazon MQ Features

| Feature | Detail |
|---|---|
| **Managed** | AWS handles patching, maintenance, and broker provisioning |
| **Storage** | EBS (single-instance) or EFS (Active/Standby for ActiveMQ) |
| **Encryption** | At-rest (KMS) and in-transit (TLS) |
| **Access** | Runs in your VPC — private by default |
| **Monitoring** | CloudWatch metrics, CloudWatch Logs |
| **Broker types** | mq.m5.large, mq.m5.xlarge, etc. (instance-based pricing) |
| **Queue + Topic** | Both supported natively (SQS = queue only, SNS = topic only) |

---

## Amazon MQ vs SQS/SNS — Detailed Comparison

| Feature | Amazon MQ | SQS | SNS |
|---|---|---|---|
| **Type** | Managed broker | Managed queue | Managed pub/sub |
| **Protocols** | MQTT, AMQP, STOMP, OpenWire | AWS API | AWS API |
| **Scaling** | Vertical (instance size) | Virtually unlimited | Virtually unlimited |
| **Serverless** | ❌ (runs on instances) | ✅ | ✅ |
| **Multi-AZ** | ✅ (Active/Standby) | ✅ (built-in) | ✅ (built-in) |
| **Queue + Topic** | ✅ Both natively | Queue only | Topic only |
| **Migration** | Drop-in for on-prem brokers | Requires code changes | Requires code changes |
| **Cost model** | Per broker-hour + storage | Per request + data | Per request + data |
| **Max throughput** | Limited by instance | Unlimited | Unlimited |

---

## Common Use Cases

| Use Case | Why Amazon MQ |
|---|---|
| **Lift-and-shift migration** | Existing on-premises ActiveMQ/RabbitMQ apps move to AWS with zero code changes |
| **IoT with MQTT** | MQTT protocol support for IoT device messaging |
| **JMS applications** | Java apps using JMS API continue working unchanged |
| **Legacy enterprise** | Enterprise apps using AMQP or STOMP protocols |

---

## Exam Cheat Sheet

> [!TIP]
> **Trigger phrases → Amazon MQ:**
> - "Migrate on-premises message broker" → **Amazon MQ**
> - "MQTT" or "AMQP" or "STOMP" or "OpenWire" or "JMS" → **Amazon MQ**
> - "ActiveMQ" or "RabbitMQ" on AWS → **Amazon MQ**
> - "No code changes for messaging migration" → **Amazon MQ**
> - "Both queues AND topics in one service" → **Amazon MQ** (not SQS+SNS)
>
> **Key facts:**
> - Amazon MQ is for **migration**. SQS/SNS is for **new cloud-native apps**.
> - Amazon MQ does NOT scale like SQS/SNS — it runs on **provisioned instances**
> - ActiveMQ HA: Active/Standby with **EFS** shared storage
> - RabbitMQ HA: Cluster deployment across AZs
> - Supports MQTT, AMQP, STOMP, OpenWire, WSS
> - Runs in your **VPC** — not publicly accessible by default
> - If no legacy broker exists → always prefer **SQS + SNS** (cloud-native, serverless, unlimited scaling)

---

## The Big Picture: AWS Messaging Services

```
  "I need messaging on AWS..."

  New application, cloud-native?
  ├── Need queue (decouple, buffer)? ──────────► SQS
  ├── Need pub/sub (fan-out, notify)? ─────────► SNS
  ├── Need real-time streaming + replay? ──────► Kinesis Data Streams
  └── Need delivery to S3/Redshift? ───────────► Kinesis Data Firehose

  Migrating existing broker?
  └── ActiveMQ / RabbitMQ / MQTT / AMQP? ─────► Amazon MQ
```
