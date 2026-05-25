---
tags: [concept, messaging, pub-sub, notifications, serverless, integration]
aliases: [SNS, Amazon SNS, Simple Notification Service, Pub/Sub, Fan-Out]
date: 2026-05-25
---

# Amazon SNS (Simple Notification Service)

**Amazon SNS** is a fully managed **publish/subscribe (Pub/Sub) messaging service** that enables you to send messages to **multiple subscribers** simultaneously. Unlike [[Amazon SQS|SQS]] which is pull-based (queue), SNS is **push-based** — it actively delivers messages to all subscribers.

> [!IMPORTANT]
> **Core exam concept:** SNS is the answer when the question involves **sending one message to many recipients**, **notifications**, or **fan-out**. The "event producer" sends a message once, and SNS delivers it to all subscribers. Think: **one-to-many**.

---

## SNS vs SQS — The Fundamental Difference

```
  SQS (Queue — Pull Model):

  Producer ──► [Queue] ──► ONE Consumer polls and receives
                           (message deleted after processing)

  ─────────────────────────────────────────────────────

  SNS (Topic — Push Model):

                           ┌──► Subscriber A (email)
  Publisher ──► [Topic] ───┼──► Subscriber B (SQS queue)
                           ├──► Subscriber C (Lambda)
                           └──► Subscriber D (HTTP endpoint)

  ONE message → pushed to ALL subscribers simultaneously
```

| Aspect | SQS | SNS |
|---|---|---|
| **Model** | Queue (pull) | Pub/Sub (push) |
| **Delivery** | One consumer per message | All subscribers receive every message |
| **Persistence** | Up to 14 days | No persistence (fire-and-forget) |
| **Consumer** | Consumer polls | SNS pushes to subscriber |
| **Use case** | Decouple + buffer | Broadcast + notify |

---

## How SNS Works

```
  ┌───────────────┐        ┌──────────────┐        ┌───────────────────┐
  │  Publishers   │        │  SNS Topic   │        │   Subscribers     │
  │               │        │              │        │                   │
  │ • AWS SDK     │  Publish│             │  Push  │ • SQS queues      │
  │ • CloudWatch  │───────►│  "OrderTopic"│───────►│ • Lambda functions│
  │ • S3 Events   │        │              │        │ • HTTP/HTTPS      │
  │ • Any service │        │  Filtering   │        │ • Email / SMS     │
  │               │        │  (optional)  │        │ • Kinesis Firehose│
  └───────────────┘        └──────────────┘        │ • Mobile push    │
                                                    └───────────────────┘
```

### Subscribers (Who receives messages?)

| Subscriber Type | Description |
|---|---|
| **SQS** | Send to an SQS queue (fan-out pattern) |
| **Lambda** | Invoke a Lambda function |
| **HTTP/HTTPS** | POST to a web endpoint |
| **Email** | Send email (or Email-JSON for structured) |
| **SMS** | Send text messages |
| **Kinesis Data Firehose** | Stream to S3, Redshift, OpenSearch, etc. |
| **Mobile Push** | Push notifications (APNS, FCM, ADM) |

---

## SNS + SQS Fan-Out Pattern

This is the **#1 most tested SNS architecture** on the exam.

```
                                    ┌──────────┐     ┌──────────────────┐
                               ┌───►│ SQS Q1   │────►│ Image Processing │
                               │    └──────────┘     └──────────────────┘
  ┌──────────┐    ┌────────┐   │
  │ S3 Event │───►│  SNS   │───┤    ┌──────────┐     ┌──────────────────┐
  │ (upload) │    │ Topic  │   ├───►│ SQS Q2   │────►│ Metadata Extract │
  └──────────┘    └────────┘   │    └──────────┘     └──────────────────┘
                               │
                               │    ┌──────────┐     ┌──────────────────┐
                               └───►│ SQS Q3   │────►│ Audit Logging    │
                                    └──────────┘     └──────────────────┘
```

**Why fan-out?**
- Publisher sends **one message** to SNS (not N messages to N queues)
- Adding new consumers = subscribe a new SQS queue (no code changes)
- SQS provides **buffering** and **retry** on the consumer side

> [!IMPORTANT]
> **Exam Pattern:** "S3 can only send one event notification per prefix/event type — but I need multiple queues" → **S3 → SNS → multiple SQS queues (fan-out)**.

---

## SNS Message Filtering

Subscribers receive only a **subset** of messages based on a **filter policy** (JSON) attached to the subscription.

```
  Publisher sends: { "orderType": "electronics", "amount": 150 }

  ┌────────┐
  │  SNS   │
  │ Topic  │
  └───┬────┘
      ├──► SQS Queue A  (filter: {"orderType": ["electronics"]})  ✅ Receives
      ├──► SQS Queue B  (filter: {"orderType": ["clothing"]})     ❌ Filtered out
      └──► Lambda C     (no filter policy)                        ✅ Receives ALL
```

> [!TIP]
> **Exam Pattern:** "Different subscribers need different messages from the same topic" → **SNS Message Filtering**. No need for multiple topics.

---

## SNS FIFO Topics

SNS has **FIFO Topics** for ordered, deduplicated message delivery.

| Feature | SNS Standard Topic | SNS FIFO Topic |
|---|---|---|
| **Throughput** | Virtually unlimited | 300 publishes/s (3,000 with batching) |
| **Ordering** | No guarantee | Guaranteed (per Message Group ID) |
| **Deduplication** | ❌ | ✅ |
| **Subscribers** | All types | **SQS FIFO queues only** |
| **Topic name** | Any valid name | Must end in **`.fifo`** |

> [!WARNING]
> **Exam trap:** SNS FIFO topics can **ONLY** fan out to **SQS FIFO queues**. They cannot deliver to Lambda, email, SMS, or HTTP endpoints.

---

## SNS Security

| Type | Detail |
|---|---|
| **In-transit** | HTTPS (default) |
| **At-rest** | SSE using **KMS keys** |
| **IAM Policies** | Control who can publish/subscribe |
| **SNS Access Policies** | Resource-based (cross-account, allow S3/CloudWatch to publish) |

---

## Exam Cheat Sheet

> [!TIP]
> **Trigger phrases → SNS:**
> - "Notify multiple services" or "fan-out" → **SNS Topic**
> - "Pub/Sub" or "publish/subscribe" → **SNS**
> - "Push notifications" → **SNS** (mobile, email, SMS)
> - "CloudWatch alarm → notification" → **CloudWatch → SNS**
> - "S3 event to multiple consumers" → **S3 → SNS → SQS fan-out**
> - "Different subscribers need different messages" → **SNS Message Filtering**
> - "Ordered pub/sub" → **SNS FIFO Topic → SQS FIFO Queue**
>
> **Key facts:**
> - SNS is **push-based**. SQS is **pull-based**. Don't confuse them.
> - SNS has **no persistence** — fire and forget (back with SQS for durability)
> - Max message size: 256 KB (same as SQS)
> - SNS FIFO → can only deliver to **SQS FIFO queues**
> - Cross-account: use **SNS Access Policies** (resource-based)
> - Fan-out pattern (SNS + SQS) is the **gold standard** for one-to-many
