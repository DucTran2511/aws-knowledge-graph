---
tags: [concept, messaging, decoupling, serverless, queue, integration]
aliases: [SQS, Amazon SQS, Simple Queue Service, Message Queue, FIFO Queue, Standard Queue]
date: 2026-05-25
---

# Amazon SQS (Simple Queue Service)

**Amazon SQS** is a fully managed **message queuing service** that enables you to **decouple** and scale microservices, distributed systems, and serverless applications. It acts as a buffer between message **producers** and **consumers**.

SQS is one of the **oldest AWS services** (launched in 2004) and is a core building block for event-driven and decoupled architectures.

> [!IMPORTANT]
> **Core exam concept:** SQS is the answer when the question involves **decoupling components**, **buffering requests**, or **handling variable workloads** asynchronously. Any time you see "decouple" in an exam question → think **SQS**.

---

## Why SQS? — The Decoupling Problem

```
  Tightly Coupled (Bad):

  ┌──────────┐   direct call   ┌──────────┐
  │ Service A│────────────────►│ Service B│
  │          │◄────────────────│          │
  └──────────┘   synchronous   └──────────┘

  Problems:
  • If Service B is down → Service A fails too
  • If Service B is slow → Service A blocks
  • Can't scale independently
  • No buffering if B can't keep up

  ─────────────────────────────────────────────

  Decoupled with SQS (Good):

  ┌──────────┐              ┌──────────┐              ┌──────────┐
  │ Service A│   send msg   │   SQS    │   poll msg   │ Service B│
  │ (Producer│─────────────►│  Queue   │◄─────────────│(Consumer)│
  │          │              │          │              │          │
  └──────────┘              └──────────┘              └──────────┘

  Benefits:
  ✅ Service A doesn't know or care about Service B
  ✅ Service B can be down — messages wait in the queue
  ✅ Scale producers and consumers independently
  ✅ Built-in buffering for traffic spikes
```

---

## SQS Queue Types

SQS offers **two queue types** with fundamentally different guarantees:

### Standard Queue

```
  ┌─────────────────────────────────────────────────┐
  │                 Standard Queue                   │
  │                                                  │
  │  Producer ──► [msg3] [msg1] [msg2] ──► Consumer │
  │                                                  │
  │  • Unlimited throughput                          │
  │  • Best-effort ordering                          │
  │  • At-least-once delivery                       │
  │    (messages may be delivered MORE than once)    │
  └─────────────────────────────────────────────────┘
```

### FIFO Queue

```
  ┌─────────────────────────────────────────────────┐
  │                   FIFO Queue                     │
  │                                                  │
  │  Producer ──► [msg1] [msg2] [msg3] ──► Consumer │
  │                                                  │
  │  • 300 msg/s (3,000 with batching)              │
  │  • Guaranteed ordering (First-In-First-Out)     │
  │  • Exactly-once delivery                        │
  │    (duplicates are removed)                     │
  │  • Queue name must end in .fifo                 │
  └─────────────────────────────────────────────────┘
```

| Feature | Standard Queue | FIFO Queue |
|---|---|---|
| **Throughput** | **Unlimited** (nearly) | **300 msg/s** (3,000 with batching) |
| **Ordering** | Best-effort (not guaranteed) | **Guaranteed** (First-In-First-Out) |
| **Delivery** | **At-least-once** (possible duplicates) | **Exactly-once** (deduplication) |
| **Deduplication** | ❌ Not built-in | ✅ Content-based or deduplication ID |
| **Queue name** | Any valid name | Must end in **`.fifo`** |
| **Use case** | High throughput, order not critical | Order matters, no duplicates allowed |

> [!WARNING]
> **Exam trap:** If the question says "messages must be processed in order" or "no duplicate processing" → **FIFO queue**. If it says "highest possible throughput" or "millions of messages per second" → **Standard queue**.

---

## Producing Messages

```
  ┌──────────┐    SendMessage API     ┌──────────┐
  │ Producer │ ──────────────────────► │   SQS    │
  │          │                         │  Queue   │
  └──────────┘                         └──────────┘

  • Message body: up to 256 KB (text: XML, JSON, plain text)
  • Metadata: message attributes (key-value pairs)
  • Optional: delay delivery (Delay Seconds)
  • SDK: SendMessage API
  
  After sending, message is PERSISTED in SQS until
  consumed or retention period expires.
```

| Attribute | Detail |
|---|---|
| **Max message size** | **256 KB** |
| **Larger messages** | Use **SQS Extended Client Library** → store payload in S3, send pointer in SQS |
| **Retention** | Default: **4 days**, configurable: **1 minute to 14 days** |
| **Producers** | Any AWS service, SDK, CLI, Lambda, EC2, etc. |

> [!TIP]
> **Exam Pattern:** "Message larger than 256 KB" → Use **SQS Extended Client Library** with **S3** to store the large payload. SQS message contains a reference (pointer) to the S3 object.

---

## Consuming Messages

```
  ┌──────────┐    ReceiveMessage API    ┌──────────┐
  │ Consumer │ ◄──────────────────────  │   SQS    │
  │          │                          │  Queue   │
  │          │    (up to 10 msgs)       │          │
  │          │                          │          │
  │ Process  │                          │ Message  │
  │ message  │    DeleteMessage API     │ becomes  │
  │          │ ──────────────────────►  │ invisible│
  └──────────┘    (remove from queue)   └──────────┘

  1. Consumer POLLS the queue (pull model — not push)
  2. Consumer receives up to 10 messages at a time
  3. Consumer processes the message
  4. Consumer DELETES the message from the queue
  5. If not deleted → message reappears after visibility timeout
```

### Key Consumer Behaviors

- SQS is a **pull-based** system — consumers poll the queue. SQS does NOT push messages.
- Multiple consumers can poll the same queue → messages are distributed among consumers.
- After processing, consumers **must delete** the message. If not deleted, it becomes visible again.

> [!CAUTION]
> **Exam critical:** If a consumer receives a message but fails to delete it before the visibility timeout expires, the message becomes visible again and **another consumer will receive it** — leading to duplicate processing in Standard queues.

---

## Visibility Timeout

The **visibility timeout** is the period during which a message, after being received by a consumer, is **invisible** to other consumers. This prevents multiple consumers from processing the same message simultaneously.

```
  Timeline:
  ─────────────────────────────────────────────────────────────►
  │                                                            
  │  Consumer A polls message                                  
  │  ▼                                                         
  │  ├──── Visibility Timeout (default 30s) ────┤              
  │  │                                          │              
  │  │  Message INVISIBLE to other consumers   │              
  │  │  Consumer A is processing...             │              
  │  │                                          │              
  │  │  ✅ If Consumer A deletes message → DONE │              
  │  │  ❌ If not deleted → message reappears ──┤              
  │  │                                          │              
  │                                              ▼              
  │                                  Other consumers can        
  │                                  now receive the message    
```

| Attribute | Detail |
|---|---|
| **Default** | **30 seconds** |
| **Range** | 0 seconds to **12 hours** |
| **Too short** | Message reappears before consumer finishes → duplicate processing |
| **Too long** | If consumer crashes, message stays invisible too long → processing delay |
| **Change mid-flight** | Consumer can call **ChangeMessageVisibility** API to extend timeout |

> [!TIP]
> **Exam Pattern:** "Messages are being processed twice" or "duplicate processing" → **Increase the visibility timeout**. "Processing takes longer than expected" → Consumer should call **ChangeMessageVisibility** to extend.

---

## Long Polling vs Short Polling

```
  Short Polling (default):                Long Polling (recommended):
  ┌──────────┐     ┌─────┐               ┌──────────┐     ┌─────┐
  │ Consumer │────►│ SQS │               │ Consumer │────►│ SQS │
  │          │◄────│     │ Empty!         │          │     │     │
  │          │────►│     │               │          │     │     │ Waits...
  │          │◄────│     │ Empty!         │          │     │     │
  │          │────►│     │               │          │     │     │ Message arrives!
  │          │◄────│     │ Got 1 msg!    │          │◄────│     │ Got 1 msg!
  └──────────┘     └─────┘               └──────────┘     └─────┘
  
  ❌ Many empty responses                 ✅ Fewer API calls
  ❌ Higher cost (charged per request)    ✅ Lower cost
  ❌ Higher latency for detection         ✅ Messages received faster
```

| Polling Type | Description |
|---|---|
| **Short Polling** | Returns immediately, even if no messages. May return empty. Default behavior. |
| **Long Polling** | Waits up to **WaitTimeSeconds** (1–20 sec) for messages to arrive. Reduces empty responses and cost. |

**How to enable Long Polling:**
- Set **WaitTimeSeconds** > 0 (at queue level or per API call)
- Preferred value: **20 seconds** (maximum wait)

> [!IMPORTANT]
> **Long Polling is ALWAYS preferred** over Short Polling. It reduces the number of API calls (lower cost), decreases latency, and eliminates empty responses. Enable it by setting **WaitTimeSeconds** at the queue level.

---

## Dead-Letter Queue (DLQ)

A **Dead-Letter Queue** is a separate SQS queue where messages are sent when they **fail to be processed** after a configured number of attempts.

```
  ┌──────────┐     ┌──────────────┐     ┌──────────┐
  │ Producer │────►│  Main Queue  │────►│ Consumer │
  └──────────┘     └──────┬───────┘     └──────────┘
                          │                    │
                          │  Receive count     │ ❌ Processing fails
                          │  exceeds           │    (message returns)
                          │  maxReceiveCount   │
                          ▼                    │
                   ┌──────────────┐            │
                   │  Dead-Letter │◄───────────┘
                   │  Queue (DLQ) │  After N failures,
                   │              │  message moves here
                   └──────────────┘
                          │
                   Inspect & debug
                   failed messages
```

| Attribute | Detail |
|---|---|
| **maxReceiveCount** | Number of times a message can be received before moving to DLQ. Set via **Redrive Policy**. |
| **DLQ type** | Must match the source queue type: Standard DLQ for Standard queue, FIFO DLQ for FIFO queue |
| **Retention** | Set a **long retention** (e.g., 14 days) on the DLQ to give developers time to debug |
| **Redrive to source** | Can move messages from DLQ **back** to the original queue after fixing the bug |

> [!TIP]
> **Exam Pattern:** "Messages keep failing" or "analyze failed messages" → **Dead-Letter Queue**. "Replay failed messages after fix" → **Redrive to source** (move messages from DLQ back to original queue).

---

## Delay Queues

**Delay Queues** let you postpone the delivery of new messages for a specified period. Messages are invisible to consumers for the duration of the delay.

```
  Producer sends message
       │
       ▼
  ┌──────────────────────────────────────────────┐
  │   Delay Period (0 – 15 minutes)              │
  │   Message is INVISIBLE during this time      │
  └──────────────────────────────────────────────┘
       │
       ▼
  Message becomes visible → consumers can receive it
```

| Attribute | Detail |
|---|---|
| **Default delay** | 0 seconds (no delay) |
| **Maximum delay** | **15 minutes** |
| **Queue-level** | Set `DelaySeconds` on the queue → applies to ALL messages |
| **Message-level** | Override with `DelaySeconds` per message (Standard queue only) |

> [!NOTE]
> Delay queues are NOT the same as visibility timeout. **Delay** = time before a message is first visible. **Visibility timeout** = time a message is invisible after being received by a consumer.

---

## FIFO Queue — Deep Dive

### Message Group ID

FIFO queues use a **Message Group ID** to define ordered subgroups within the same queue. Messages within the **same group** are guaranteed to be ordered and processed one at a time. Different groups can be processed **in parallel**.

```
  ┌─────────────────────────────────────────────────────────┐
  │                     FIFO Queue                           │
  │                                                          │
  │  Group "order-123": [msg1] → [msg2] → [msg3]  (ordered)│
  │  Group "order-456": [msg1] → [msg2]            (ordered)│
  │  Group "order-789": [msg1]                     (ordered)│
  │                                                          │
  │  Each group has exactly ONE consumer at a time          │
  │  Different groups can be consumed IN PARALLEL           │
  └─────────────────────────────────────────────────────────┘
```

### Deduplication

FIFO queues prevent duplicate messages via two methods:

| Method | Description |
|---|---|
| **Content-based** | SQS generates a SHA-256 hash of the message body. Same body within dedup interval = rejected |
| **Message Deduplication ID** | Producer explicitly provides a unique ID. Same ID within **5-minute dedup interval** = rejected |

> [!WARNING]
> **Exam trap:** FIFO queues provide exactly-once processing BUT have limited throughput (300 msg/s, or 3,000 with batching). If the exam asks for "exactly-once + unlimited throughput" → that's a trick — you can't have both. Choose FIFO for correctness, Standard for throughput.

---

## SQS + Auto Scaling Group (ASG)

A classic exam architecture: use SQS queue depth to **auto-scale** consumers.

```
  ┌──────────┐         ┌──────────┐         ┌─────────────────────┐
  │ Producers│────────►│   SQS    │◄────────│   Auto Scaling      │
  │          │         │  Queue   │         │   Group (EC2)       │
  └──────────┘         └──────────┘         │                     │
                            │                │  ┌───┐ ┌───┐ ┌───┐ │
                            │                │  │EC2│ │EC2│ │EC2│ │
                       CloudWatch            │  └───┘ └───┘ └───┘ │
                       Alarm                 └─────────────────────┘
                            │                         ▲
                            │  "Queue depth > 1000"   │
                            └─────────────────────────┘
                              Scale out (add EC2 instances)

  CloudWatch Metric: ApproximateNumberOfMessagesVisible
  Scale OUT when queue is deep → more consumers
  Scale IN when queue is empty → fewer consumers
```

The key CloudWatch metric is **ApproximateNumberOfMessagesVisible** — the number of messages waiting in the queue. Use this to trigger scaling policies.

> [!TIP]
> **Exam Pattern:** "Scale consumers based on workload" or "process messages faster when queue grows" → **SQS + ASG scaling based on queue depth (ApproximateNumberOfMessagesVisible metric)**.

---

## SQS + SNS: Fan-Out Pattern

Combine SQS with [[Amazon SNS]] for the **fan-out** pattern — send one message to multiple SQS queues simultaneously.

```
  ┌──────────┐         ┌──────────┐         ┌──────────┐
  │          │         │  SQS     │────────►│Consumer A│
  │          │    ┌───►│  Queue A │         │(Email)   │
  │          │    │    └──────────┘         └──────────┘
  │ Producer │    │
  │          │────┤    ┌──────────┐         ┌──────────┐
  │          │    │    │  SQS     │────────►│Consumer B│
  │  (sends  │    ├───►│  Queue B │         │(Analytics│
  │   to SNS)│    │    └──────────┘         └──────────┘
  │          │    │
  └──────────┘    │    ┌──────────┐         ┌──────────┐
       │          │    │  SQS     │────────►│Consumer C│
       │          └───►│  Queue C │         │(Archive) │
       ▼               └──────────┘         └──────────┘
  ┌──────────┐
  │   SNS    │   One message → published to SNS topic
  │  Topic   │   SNS fans out to ALL subscribed SQS queues
  └──────────┘
```

**Why not send directly to multiple SQS queues?**
- Sending to N queues requires N API calls from the producer.
- Adding a new queue requires modifying the producer.
- With SNS fan-out: producer sends once to SNS, SNS delivers to all subscribed queues. **Fully decoupled.**

> [!IMPORTANT]
> **Exam Pattern:** "Send the same message to multiple SQS queues" or "process one event in multiple ways" → **SNS + SQS Fan-Out pattern**. This is a very common exam architecture.

---

## SQS + S3 Event Notifications

S3 can send event notifications to SQS when objects are created, deleted, etc. Combined with the fan-out pattern:

```
  ┌──────┐    Event     ┌──────┐     Fan-out    ┌──────────┐
  │  S3  │────────────►│ SNS  │───────────────►│ SQS Q1   │→ Thumbnail generation
  │Bucket│   ObjectCreated│Topic│───────────────►│ SQS Q2   │→ Metadata extraction
  └──────┘              └──────┘───────────────►│ SQS Q3   │→ Audit logging
                                                 └──────────┘
```

> [!NOTE]
> For S3 events to go to multiple queues, you MUST use the SNS fan-out pattern. S3 event notification rules allow only **one destination per event type per prefix** — so SNS acts as the multiplexer.

---

## SQS Security

### Encryption

| Type | Detail |
|---|---|
| **In-transit** | HTTPS endpoints (encrypted by default) |
| **At-rest** | **SSE-SQS** (SQS managed keys, default) or **SSE-KMS** (customer-managed KMS keys) |

### Access Control

| Method | Description |
|---|---|
| **IAM Policies** | Control who can call SQS APIs (SendMessage, ReceiveMessage, etc.) |
| **SQS Access Policies** | Resource-based policies (like S3 bucket policies) — control cross-account access and allow other AWS services (S3, SNS) to write to the queue |

```
  Cross-Account Access:

  Account A                              Account B
  ┌────────────────────────┐            ┌────────────────────────┐
  │  SQS Queue             │            │  EC2 Consumer          │
  │                        │◄───────────│                        │
  │  SQS Access Policy:    │  poll      │  Uses IAM role with    │
  │  "Allow Account B      │            │  sqs:ReceiveMessage    │
  │   to receive messages" │            │                        │
  └────────────────────────┘            └────────────────────────┘
```

> [!TIP]
> **Exam Pattern:** "Allow another AWS account to read from my SQS queue" → **SQS Access Policy** (resource-based). "Allow S3 to send notifications to SQS" → **SQS Access Policy** allowing the S3 service principal.

---

## SQS Request-Response Pattern

For scenarios where you need a **response** from the consumer, use temporary response queues:

```
  ┌──────────┐   Request    ┌──────────┐   Process    ┌──────────┐
  │ Requester│─────────────►│  Request │─────────────►│ Responder│
  │          │              │  Queue   │              │          │
  │          │              └──────────┘              │          │
  │          │                                         │          │
  │          │   Response   ┌──────────┐              │          │
  │          │◄─────────────│ Response │◄─────────────│          │
  └──────────┘              │  Queue   │              └──────────┘
                            └──────────┘
  
  Use the SQS Temporary Queue Client to implement this efficiently.
  It uses virtual queues on top of a single physical queue.
```

---

## Key SQS Limits & Numbers

| Parameter | Value |
|---|---|
| **Max message size** | 256 KB |
| **Max retention** | 14 days |
| **Default retention** | 4 days |
| **Default visibility timeout** | 30 seconds |
| **Max visibility timeout** | 12 hours |
| **Max delay** | 15 minutes |
| **Long poll max wait** | 20 seconds |
| **FIFO throughput** | 300 msg/s (3,000 with batching) |
| **Standard throughput** | Virtually unlimited |
| **Max messages per ReceiveMessage** | 10 |
| **FIFO dedup interval** | 5 minutes |

---

## SQS vs SNS vs Kinesis

| Feature | SQS | SNS | Kinesis Data Streams |
|---|---|---|---|
| **Model** | Queue (pull) | Pub/Sub (push) | Stream (pull) |
| **Consumers** | Consumer pulls messages | SNS pushes to subscribers | Consumer pulls from shard |
| **Delivery** | Messages go to ONE consumer (per message) | Messages go to ALL subscribers | Messages go to ALL consumers (in consumer group) |
| **Persistence** | Up to 14 days | No persistence (fire-and-forget) | Up to 365 days |
| **Ordering** | FIFO queue only | FIFO topic only | Per-shard ordering |
| **Throughput** | Unlimited (Standard) | Unlimited | Provisioned (per shard) |
| **Replay** | ❌ (deleted after consumption) | ❌ | ✅ (replay from any point) |
| **Use case** | Decouple + buffer | Fan-out notifications | Real-time streaming + analytics |

> [!CAUTION]
> **Exam critical:** The SQS vs SNS vs Kinesis comparison is **heavily tested**. Key distinctions:
> - SQS = **pull**, one consumer per message, **no replay**
> - SNS = **push**, all subscribers, **no persistence**
> - Kinesis = **pull**, all consumers read all data, **replay capability**

---

## Exam Cheat Sheet

> [!TIP]
> **Trigger phrases → SQS:**
> - "Decouple" or "asynchronous processing" → **SQS**
> - "Buffer requests between services" → **SQS**
> - "Process messages in order" → **SQS FIFO**
> - "Exactly-once processing" → **SQS FIFO**
> - "Messages processed twice" → **Increase visibility timeout** or use **FIFO**
> - "Failed messages" or "retry analysis" → **Dead-Letter Queue**
> - "Scale consumers with queue depth" → **SQS + ASG + CloudWatch ApproximateNumberOfMessagesVisible**
> - "Send same message to multiple queues" → **SNS + SQS Fan-Out**
> - "Message larger than 256 KB" → **SQS Extended Client Library + S3**
> - "Reduce API calls when polling" → **Long Polling (WaitTimeSeconds)**
>
> **Key facts:**
> - Standard: unlimited throughput, at-least-once, best-effort ordering
> - FIFO: 300 msg/s (3,000 batched), exactly-once, guaranteed ordering, name ends in `.fifo`
> - Max message size: 256 KB | Max retention: 14 days | Default retention: 4 days
> - Visibility timeout: default 30s, max 12 hours — extend with ChangeMessageVisibility API
> - Long Polling (WaitTimeSeconds up to 20s) is ALWAYS preferred over Short Polling
> - DLQ must match queue type (Standard DLQ ↔ Standard queue, FIFO DLQ ↔ FIFO queue)
> - Cross-account access: use SQS Access Policies (resource-based)
> - SQS is **pull-based**. SNS is **push-based**. Don't confuse them.
