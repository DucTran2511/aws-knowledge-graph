---
tags: [concept, streaming, real-time, analytics, big-data, integration]
aliases: [Kinesis, Amazon Kinesis, Kinesis Data Streams, Kinesis Data Firehose, Kinesis Data Analytics, Real-time Streaming]
date: 2026-05-25
---

# Amazon Kinesis

**Amazon Kinesis** is a family of services for **real-time streaming data** — collecting, processing, and analyzing data streams at any scale. Think of it as the AWS answer to Apache Kafka.

> [!IMPORTANT]
> **Core exam concept:** Kinesis is the answer when the question involves **real-time data streaming**, **log ingestion**, **IoT telemetry**, or **clickstream analytics**. Any time you see "real-time" + "streaming" → think **Kinesis**.

---

## The Kinesis Family

```
  ┌───────────────────────────────────────────────────────────────────┐
  │                       Amazon Kinesis Family                       │
  │                                                                   │
  │  ┌─────────────────────┐   ┌─────────────────────────────────┐    │
  │  │  Kinesis Data       │   │  Kinesis Data Firehose          │    │
  │  │  Streams            │   │  (Delivery → S3/Redshift/etc.)  │    │
  │  │  (Real-time ingest) │   │  (Near-real-time, fully managed)│    │
  │  └─────────────────────┘   └─────────────────────────────────┘    │
  │                                                                   │
  │  ┌─────────────────────┐   ┌─────────────────────────────────┐    │
  │  │  Kinesis Data       │   │  Kinesis Video Streams          │    │
  │  │  Analytics          │   │  (Video ingestion & processing) │    │
  │  │  (SQL on streams)   │   │                                 │    │
  │  └─────────────────────┘   └─────────────────────────────────┘    │
  |                                                                   |
  └───────────────────────────────────────────────────────────────────┘
```

---

## Kinesis Data Streams

The **core streaming service** — ingest and store real-time data for processing by consumers.

```
  ┌─────────────┐
  │ Producers   │         ┌────────────────────────────────────┐
  │             │         │     Kinesis Data Stream             │
  │ • Apps/SDK  │         │                                    │
  │ • KPL       │  Put    │  Shard 1: [rec1] [rec2] [rec3] →  │
  │ • Kinesis   │  Record │  Shard 2: [rec1] [rec2] →         │     ┌───────────┐
  │   Agent     │────────►│  Shard 3: [rec1] [rec2] [rec3] →  │────►│ Consumers │
  │ • CloudWatch│         │  Shard 4: [rec1] →                │     │           │
  │ • IoT       │         │                                    │     │ • Apps/SDK│
  └─────────────┘         │  Retention: 1 day → 365 days      │     │ • KCL     │
                          │  Data is IMMUTABLE (append-only)   │     │ • Lambda  │
                          └────────────────────────────────────┘     │ • Firehose│
                                                                     │ • Analytics│
                                                                     └───────────┘
```

### Shards — The Unit of Capacity

| Attribute | Detail |
|---|---|
| **Shard** | Base throughput unit of a Kinesis stream |
| **Write capacity** | **1 MB/s** or **1,000 records/s** per shard |
| **Read capacity** | **2 MB/s** per shard (shared across consumers) |
| **Enhanced fan-out** | **2 MB/s per shard per consumer** (dedicated, push-based) |
| **Scaling** | Add shards (split) or remove shards (merge) — manual or auto |

```
  Capacity Planning:

  Need 5 MB/s write throughput?
  → 5 MB/s ÷ 1 MB/s per shard = 5 shards

  Need 10 MB/s read throughput (shared)?
  → 10 MB/s ÷ 2 MB/s per shard = 5 shards
```

### Provisioned vs On-Demand Mode

| Mode | Description |
|---|---|
| **Provisioned** | You specify the number of shards. Pay per shard-hour. Manual scaling. |
| **On-Demand** | AWS auto-scales shards based on throughput. No capacity planning. Pay per stream-hour + per GB. |

### Key Properties

| Property | Detail |
|---|---|
| **Retention** | Default **24 hours**, up to **365 days** |
| **Immutability** | Data **cannot be deleted** — it's append-only and expires after retention |
| **Replay** | ✅ Consumers can **re-read/replay** data from any point in the retention window |
| **Ordering** | Messages ordered **per shard** (use Partition Key for consistent routing) |
| **Partition Key** | Determines which shard receives the record (hashed). Same key → same shard → ordered. |

> [!CAUTION]
> **Exam critical:** Kinesis Data Streams supports **replay** — consumers can re-read old data. SQS does NOT (message deleted after consumption). This is a key differentiator.

### Producers

| Producer | Description |
|---|---|
| **AWS SDK** | PutRecord / PutRecords API |
| **Kinesis Producer Library (KPL)** | Batching, compression, retry — high-performance |
| **Kinesis Agent** | Pre-built agent for log files (monitors files and sends to stream) |

### Consumers

| Consumer | Description |
|---|---|
| **AWS SDK** | GetRecords API (shared throughput) |
| **Kinesis Client Library (KCL)** | Checkpointing, load balancing across shards, runs on EC2/ECS |
| **AWS Lambda** | Serverless consumer — triggered by stream records |
| **Kinesis Data Firehose** | Deliver to S3, Redshift, OpenSearch, etc. |
| **Kinesis Data Analytics** | Run SQL or Flink on stream data |

### Enhanced Fan-Out

```
  Standard Consumer (Shared):        Enhanced Fan-Out (Dedicated):

  Shard ──► 2 MB/s shared            Shard ──► 2 MB/s → Consumer A
             among ALL consumers              2 MB/s → Consumer B
                                               2 MB/s → Consumer C
  
  Consumer polls (pull)               Kinesis pushes (push via HTTP/2)
  200ms latency                       ~70ms latency
```

| Aspect | Standard (Shared) | Enhanced Fan-Out |
|---|---|---|
| **Throughput** | 2 MB/s per shard (shared) | 2 MB/s per shard **per consumer** |
| **Model** | Pull (GetRecords) | Push (SubscribeToShard, HTTP/2) |
| **Latency** | ~200ms | ~70ms |
| **Cost** | Lower | Higher (per consumer-shard-hour) |
| **Use when** | Few consumers | Many consumers, low latency critical |

> [!TIP]
> **Exam Pattern:** "Multiple applications reading from the same stream with low latency" → **Enhanced Fan-Out** (dedicated 2 MB/s per consumer).

---

## Kinesis Data Firehose

A fully managed **delivery service** — takes data and loads it into destinations. **No code to write.**

```
  ┌─────────────┐     ┌──────────────────┐     ┌─────────────────┐
  │  Sources    │     │  Kinesis Data     │     │  Destinations   │
  │             │     │  Firehose         │     │                 │
  │ • KDS       │     │                  │     │ • Amazon S3     │
  │ • SDK/KPL   │────►│ • Buffer (size/  │────►│ • Amazon Redshift│
  │ • CloudWatch│     │   time based)    │     │ • OpenSearch    │
  │ • IoT       │     │ • Transform      │     │ • 3rd party     │
  │ • Kinesis   │     │   (Lambda)       │     │   (Splunk,      │
  │   Agent     │     │ • Compress       │     │    Datadog...)  │
  └─────────────┘     │ • Convert format │     │ • HTTP endpoint │
                      └──────────────────┘     └─────────────────┘
                                                        │
                                                All failures → S3
                                                (backup bucket)
```

| Attribute | Detail |
|---|---|
| **Fully managed** | No shards, no capacity planning, auto-scales |
| **Near real-time** | Buffer interval: **60 seconds minimum** (or 1 MB buffer size) |
| **Data transformation** | Optional Lambda function to transform records |
| **Compression** | GZIP, ZIP, Snappy |
| **Format conversion** | JSON → Parquet / ORC (for analytics) |
| **Failed data** | ALL failed records go to a **backup S3 bucket** |
| **No replay** | Data is delivered and gone — no storage/retention |

> [!WARNING]
> **Exam trap:** Firehose is **near real-time** (minimum 60s buffer), NOT real-time. For true real-time (sub-second) → use **Kinesis Data Streams** with custom consumers. Firehose also does **not support replay**.

### Kinesis Data Streams vs Firehose

| Feature | Data Streams | Data Firehose |
|---|---|---|
| **Latency** | **Real-time** (~200ms or ~70ms with EFO) | **Near real-time** (60s minimum buffer) |
| **Management** | You manage shards, consumers | Fully managed, serverless |
| **Scaling** | Manual (shard split/merge) or On-Demand | Automatic |
| **Storage** | 1–365 days retention | No storage (pass-through) |
| **Replay** | ✅ Yes | ❌ No |
| **Consumers** | Custom (SDK, KCL, Lambda) | S3, Redshift, OpenSearch, HTTP, 3rd party |
| **Data transform** | Custom code in consumer | Lambda (optional) |
| **Use case** | Custom processing, real-time apps | Load data into data stores |

---

## Kinesis Data Analytics

Run **SQL queries** or **Apache Flink** applications on streaming data in real-time.

```
  ┌──────────────┐     ┌──────────────────────┐     ┌──────────────┐
  │ Kinesis Data │     │  Kinesis Data         │     │ Destinations │
  │ Streams      │────►│  Analytics            │────►│              │
  │              │     │                      │     │ • KDS        │
  │ Kinesis Data │────►│  • SQL queries        │────►│ • Firehose   │
  │ Firehose     │     │  • Apache Flink       │     │ • Lambda     │
  └──────────────┘     │  • Reference data     │     └──────────────┘
                       │    from S3            │
                       └──────────────────────┘
```

| Feature | SQL | Apache Flink |
|---|---|---|
| **Language** | SQL | Java, Scala, Python |
| **Complexity** | Simple transformations | Complex event processing |
| **Use case** | Time-series analytics, dashboards | Stateful processing, ML on streams |

---

## Kinesis vs SQS vs SNS

| Feature | Kinesis Data Streams | SQS | SNS |
|---|---|---|---|
| **Model** | Stream (pull/push) | Queue (pull) | Pub/Sub (push) |
| **Delivery** | All consumers read all data | One consumer per message | All subscribers get all messages |
| **Persistence** | 1–365 days | Up to 14 days | None (fire-and-forget) |
| **Ordering** | Per-shard ordering | FIFO queue only | FIFO topic only |
| **Replay** | ✅ Yes | ❌ No | ❌ No |
| **Throughput** | Provisioned (per shard) | Unlimited (Standard) | Unlimited |
| **Consumers** | KCL, Lambda, SDK | SDK, Lambda | SQS, Lambda, HTTP, email, SMS |
| **Use case** | Real-time streaming, analytics | Decouple, buffer | Notifications, fan-out |

> [!CAUTION]
> **Exam critical distinctions:**
> - Need **replay**? → **Kinesis** (only one that supports it)
> - Need **real-time analytics on stream**? → **Kinesis**
> - Need to **decouple + buffer**? → **SQS**
> - Need to **notify multiple subscribers**? → **SNS**
> - Need **fan-out with durability**? → **SNS + SQS**

---

## Data Ordering: Kinesis vs SQS FIFO

```
  Kinesis: Ordering by Partition Key

  Partition Key "truck-1" → always goes to Shard 1
  Partition Key "truck-2" → always goes to Shard 2

  Within each shard, records are ORDERED.

  ───────────────────────────────────────────

  SQS FIFO: Ordering by Message Group ID

  Group ID "order-123" → ordered within group
  Group ID "order-456" → ordered within group

  Different groups can be processed in parallel.
```

> [!TIP]
> **Exam Pattern:** "Order data by device/user/entity ID" → **Kinesis** with Partition Key = entity ID. "Order processing by order ID" → **SQS FIFO** with Message Group ID = order ID.

---

## Exam Cheat Sheet

> [!TIP]
> **Trigger phrases → Kinesis:**
> - "Real-time streaming" or "IoT data ingestion" → **Kinesis Data Streams**
> - "Clickstream analytics" or "log ingestion" → **Kinesis Data Streams**
> - "Replay data" or "re-process stream" → **Kinesis Data Streams** (only one that supports replay)
> - "Load streaming data into S3/Redshift" → **Kinesis Data Firehose**
> - "Near real-time delivery to data lake" → **Kinesis Data Firehose**
> - "SQL on streaming data" → **Kinesis Data Analytics**
> - "Multiple consumers, each gets all data" → **Kinesis** (not SQS)
>
> **Key facts:**
> - Data Streams: 1 MB/s write, 2 MB/s read per shard. Retention 1–365 days. Replay ✅
> - Firehose: Near real-time (60s buffer min). Fully managed. No replay. Delivers to S3/Redshift/OpenSearch
> - Data Streams is **real-time**. Firehose is **near real-time**. Don't confuse them.
> - Partition Key determines shard → ordering within shard
> - Enhanced Fan-Out: 2 MB/s per consumer per shard (push via HTTP/2)
> - On-Demand mode: no shard management, auto-scales
