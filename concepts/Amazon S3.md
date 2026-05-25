---
tags: [concept, storage, object-storage, serverless, durability, managed-service]
aliases: [S3, Amazon S3, AWS S3, Simple Storage Service]
date: 2026-05-08
---

# Amazon S3

**Amazon Simple Storage Service (S3)** is an **infinitely scalable object storage service** that offers industry-leading durability (99.999999999% — 11 nines), availability, security, and performance. It is one of the most fundamental and heavily tested AWS services across all certification exams.

S3 stores data as **objects** inside **buckets** and is designed for virtually unlimited storage capacity.

> [!IMPORTANT]
> **Core exam concept:** S3 is **object storage**, not block storage ([[Elastic Block Store (EBS) Volumes|EBS]]) or file storage ([[Elastic File System (EFS)|EFS]]). Objects are stored in a flat namespace within buckets, accessed via HTTP/HTTPS. You cannot mount S3 as a filesystem or install an OS on it.

---

## Core Concepts

### Buckets

- Buckets are **globally unique containers** for objects.
- Defined at the **Region level** (despite the global namespace).
- Naming: 3–63 characters, lowercase, no underscores, not an IP, must start with letter/number.
- No limit on the number of objects in a bucket.

### Objects

- Objects are **files** stored in buckets.
- Each object has a **Key** — the full path (e.g., `s3://my-bucket/folder/file.txt`).
- The key is composed of a **prefix** (`folder/`) + **object name** (`file.txt`).
- Max object size: **5 TB** (5,000 GB).
- For uploads > **100 MB**, use **Multi-Part Upload** (required for > **5 GB**).

```
Bucket: my-company-data
├── images/logo.png          ← Key: images/logo.png
├── images/banner.jpg        ← Key: images/banner.jpg
├── reports/2026/q1.pdf      ← Key: reports/2026/q1.pdf
└── config.json              ← Key: config.json

• There are NO real "folders" — just key prefixes
• The console shows a folder-like UI, but it's flat
```

> [!WARNING]
> **Exam trap:** S3 has no concept of directories. What looks like `folder/subfolder/file.txt` is a single flat key. The console displays a folder hierarchy as a convenience, but S3 is a flat key-value store.

### Metadata & Tags

| Feature | Description |
|---|---|
| **Metadata** | Key-value pairs attached to objects (system + user-defined). Set at upload time. |
| **Tags** | Unicode key-value pairs (up to 10). Useful for lifecycle policies, access control, analytics. |
| **Version ID** | Assigned when versioning is enabled. |

---

## S3 Security

Security is one of the **most exam-heavy** S3 topics. There are multiple layers:

### Access Control Mechanisms

```
                    ┌──────────────────────────────────────────────────────┐
                    │                   S3 Access Decision                 │
                    │                                                      │
                    │   DENY wins.  Then: Is there an explicit ALLOW?     │
                    │                                                      │
                    │   ┌────────────────┐  ┌────────────────┐            │
                    │   │  IAM Policies   │  │ Bucket Policies │            │
                    │   │  (User/Role)   │  │ (Resource-based)│            │
                    │   └───────┬────────┘  └───────┬────────┘            │
                    │           │                    │                      │
                    │           └──────┬─────────────┘                      │
                    │                  ▼                                    │
                    │         UNION of ALLOW                               │
                    │         (both evaluated)                             │
                    │                  │                                    │
                    │                  ▼                                    │
                    │         MINUS any DENY                               │
                    │         (explicit deny wins)                         │
                    │                  │                                    │
                    │                  ▼                                    │
                    │            FINAL: ALLOW or DENY                      │
                    └──────────────────────────────────────────────────────┘
```

### Bucket Policies (Resource-Based)

JSON-based policies attached directly to the bucket. Most common method for:
- Granting **public access** to a bucket.
- Granting **cross-account access**.
- Forcing **encryption** at upload.

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "PublicReadGetObject",
      "Effect": "Allow",
      "Principal": "*",
      "Action": "s3:GetObject",
      "Resource": "arn:aws:s3:::my-bucket/*"
    }
  ]
}
```

### S3 Block Public Access

An **account-level or bucket-level override** that blocks all public access regardless of bucket policies or ACLs.

```
  ┌─────────────────────────────────────────────────────┐
  │           S3 Block Public Access Settings           │
  │                                                     │
  │  ☑ Block public access to buckets and objects       │
  │    granted through NEW ACLs                         │
  │  ☑ Block public access to buckets and objects       │
  │    granted through ANY ACLs                         │
  │  ☑ Block public access to buckets and objects       │
  │    granted through new public bucket policies       │
  │  ☑ Block public and cross-account access to         │
  │    buckets and objects through ANY public policy    │
  │                                                     │
  │  Even if a bucket policy says "Allow *",            │
  │  Block Public Access OVERRIDES it!                  │
  └─────────────────────────────────────────────────────┘
```

> [!CAUTION]
> **Exam critical:** Block Public Access is enabled **by default** on new buckets. You must explicitly disable it AND set a bucket policy to make objects public. A bucket policy alone is not enough.

### Encryption

See: [[S3 Encryption]]

| Method | Key | Description |
|---|---|---|
| **SSE-S3** | AWS-managed (AES-256) | Default. Header: `x-amz-server-side-encryption: AES256` |
| **SSE-KMS** | AWS KMS key | You control key + audit via CloudTrail. Header: `aws:kms` |
| **SSE-C** | Customer-provided | You send key with every request. HTTPS required. |
| **Client-Side** | Client manages | Encrypt before upload. AWS never sees plaintext. |

> [!TIP]
> **Exam Pattern:** "Audit who accessed the encryption key" or "control key rotation" → **SSE-KMS**. "Company manages its own keys outside AWS" → **SSE-C**. Default / simplest → **SSE-S3**.

---

## S3 Versioning

Versioning keeps **multiple variants** of an object in the same bucket. Enabled at the **bucket level**.

```
  Bucket: my-docs (Versioning ENABLED)

  Key: report.pdf
  ┌──────────────────────────────────────────┐
  │  Version 3 (latest)  ← GET returns this │
  │  Version 2                               │
  │  Version 1                               │
  └──────────────────────────────────────────┘

  DELETE report.pdf → adds a "Delete Marker" (soft delete)
  ┌──────────────────────────────────────────┐
  │  Delete Marker      ← GET returns 404    │
  │  Version 3                               │
  │  Version 2                               │
  │  Version 1          ← All still exist!   │
  └──────────────────────────────────────────┘

  To restore: delete the Delete Marker
  To permanently delete: specify the Version ID
```

| Aspect | Detail |
|---|---|
| **Protect against** | Unintended deletes and overwrites |
| **Delete without version ID** | Adds a **Delete Marker** (recoverable) |
| **Delete with version ID** | **Permanently deletes** that version |
| **Suspend versioning** | Stops creating new versions; existing versions remain |
| **Pre-versioning objects** | Have version ID = `null` |

> [!WARNING]
> **Exam trap:** Suspending versioning does NOT delete existing versions. All previous versions remain and continue to consume storage costs.

---

## S3 Replication

S3 supports **asynchronous** object replication between buckets:

| Feature | CRR (Cross-Region) | SRR (Same-Region) |
|---|---|---|
| **Full name** | Cross-Region Replication | Same-Region Replication |
| **Use cases** | Compliance, low-latency access, DR | Log aggregation, live replication between prod/test |
| **Regions** | Different regions | Same region |
| **Versioning** | Required on both source and destination | Required on both |
| **Replication** | Asynchronous | Asynchronous |

### Prerequisites
- **Versioning** must be enabled on source and destination buckets.
- Must grant proper **IAM permissions** to S3 for replication.
- Can be cross-account (with proper bucket policies).

### Key Behaviors

```
  Source Bucket (us-east-1)              Destination Bucket (eu-west-1)
  ┌─────────────────────────┐           ┌─────────────────────────┐
  │  file.txt (v3) ──────────────────►  │  file.txt (v3)          │
  │  file.txt (v2)          │  ASYNC    │  file.txt (v2)          │
  │  image.png (v1) ────────────────►   │  image.png (v1)         │
  │                         │           │                         │
  │  ⚠ Existing objects     │           │  Only NEW objects are   │
  │    NOT replicated       │           │  replicated by default  │
  │    automatically        │           │                         │
  └─────────────────────────┘           └─────────────────────────┘
```

- After enabling, only **new objects** are replicated (use **S3 Batch Replication** for existing).
- **Delete markers** are NOT replicated by default (can be enabled).
- **Permanent deletes** (with version ID) are NEVER replicated (to prevent malicious deletes).
- **No chaining**: Bucket A → B → C does NOT work. A's objects don't replicate to C.

> [!TIP]
> **Exam Pattern:** "Replicate objects to another region for compliance" → **CRR**. "Aggregate logs from multiple buckets" → **SRR**. "Replicate existing objects" → **S3 Batch Replication**.

---

## S3 Storage Classes

S3 offers multiple storage classes optimized for different access patterns and cost requirements.

See: [[S3 Storage Classes]]

| Storage Class | Availability | Min Duration | Retrieval Fee | Use Case |
|---|---|---|---|---|
| **S3 Standard** | 99.99% | None | None | Frequently accessed data |
| **S3 Intelligent-Tiering** | 99.9% | None | None | Unknown/changing access patterns |
| **S3 Standard-IA** | 99.9% | 30 days | Per-GB | Infrequent but rapid access |
| **S3 One Zone-IA** | 99.5% | 30 days | Per-GB | Infrequent, re-creatable data |
| **S3 Glacier Instant** | 99.9% | 90 days | Per-GB | Archive with ms retrieval |
| **S3 Glacier Flexible** | 99.99% | 90 days | Per-GB | Archive, mins–hours retrieval |
| **S3 Glacier Deep Archive** | 99.99% | 180 days | Per-GB | Long-term archive, 12–48 hrs |

> All classes share **11 nines (99.999999999%) durability**.

### Storage Class Transition Flow

```
  S3 Standard
       │
       ▼
  S3 Standard-IA  ←──  S3 Intelligent-Tiering
       │
       ▼
  S3 One Zone-IA
       │
       ▼
  S3 Glacier Instant Retrieval
       │
       ▼
  S3 Glacier Flexible Retrieval
       │
       ▼
  S3 Glacier Deep Archive
```

> [!IMPORTANT]
> **Lifecycle Rules** automate transitions between storage classes. You can define rules to transition objects (e.g., move to IA after 30 days, Glacier after 90 days) or expire/delete objects after a period.

---

## S3 Lifecycle Rules

Lifecycle rules automate object management — transitioning between classes or deleting objects.

| Action | Description |
|---|---|
| **Transition actions** | Move objects to another storage class after N days |
| **Expiration actions** | Delete objects after N days, delete old versions, delete incomplete multi-part uploads |

```
  Object Created
       │
       ├── Day 0:   S3 Standard
       │
       ├── Day 30:  ──► S3 Standard-IA    (Transition Rule)
       │
       ├── Day 90:  ──► S3 Glacier Flexible (Transition Rule)
       │
       └── Day 365: ──► DELETE             (Expiration Rule)
```

### Common Exam Scenarios

| Scenario | Solution |
|---|---|
| "Recover deleted objects for 30 days" | Enable versioning + lifecycle rule to expire non-current versions after 30 days |
| "Images accessed frequently for 45 days, rarely after" | Transition to Standard-IA after 45 days |
| "Delete incomplete multi-part uploads" | Lifecycle rule to abort incomplete uploads after N days |
| "Move non-current versions to cheap storage" | Transition non-current versions to Glacier |

---

## S3 Event Notifications

S3 can send notifications when events occur on objects:

```
  S3 Bucket
  ┌─────────────────────────┐
  │  Events:                │
  │  • s3:ObjectCreated:*   │────►  ┌────────────┐
  │  • s3:ObjectRemoved:*   │────►  │  SNS Topic │
  │  • s3:ObjectRestore:*   │────►  │  SQS Queue │
  │  • s3:Replication:*     │────►  │  Lambda    │
  │                         │────►  │ EventBridge│
  └─────────────────────────┘       └────────────┘
```

- Events delivered in **seconds** (can sometimes take a minute or longer).
- **Amazon EventBridge** integration enables advanced filtering, multiple destinations, archive, replay, and rule-based routing.

> [!TIP]
> **Exam Pattern:** "Process images upon upload" → S3 Event → [[Lambda]]. "Fan out notifications to multiple queues" → S3 Event → **EventBridge** or **SNS + SQS fan-out**.

---

## S3 Performance

### Baseline Performance

- **3,500 PUT/COPY/POST/DELETE** requests per second per prefix.
- **5,500 GET/HEAD** requests per second per prefix.
- No limit on the number of prefixes.

### Multi-Part Upload

- **Recommended** for files > 100 MB, **required** for > 5 GB.
- Parallelizes uploads, improves throughput, and enables retry of individual parts.

### S3 Transfer Acceleration

Uses **CloudFront edge locations** to accelerate long-distance uploads:

```
  User (Australia)                     S3 Bucket (us-east-1)
       │                                      ▲
       │  Upload to nearest                   │ AWS internal
       │  edge location                       │ backbone (fast)
       ▼                                      │
  ┌──────────────┐                    ┌───────┴───────┐
  │ Edge Location │──────────────────►│   S3 Bucket   │
  │ (Sydney)      │  Private AWS      │  (us-east-1)  │
  └──────────────┘  network           └───────────────┘
```

### S3 Byte-Range Fetches

- Parallelize GET requests by requesting specific **byte ranges**.
- Better resilience (retry smaller chunks) and faster downloads.
- Can retrieve only **partial data** (e.g., first 50 bytes = file header).

### S3 Select & Glacier Select

- Use **SQL expressions** to filter data **server-side** before transfer.
- Up to **400% faster** and **80% cheaper** than fetching full objects.
- Filters rows and columns from CSV, JSON, or Parquet files.

---

## S3 Static Website Hosting

S3 can host **static websites** (HTML, CSS, JS, images):

- Bucket must be configured for static website hosting.
- Must enable **public access** (disable Block Public Access + add bucket policy).
- URL format: `http://bucket-name.s3-website-region.amazonaws.com`

```
  ┌──────────┐     HTTP      ┌──────────────────────┐
  │  Browser  │─────────────►│  S3 Static Website   │
  │           │◄─────────────│  index.html           │
  └──────────┘               │  error.html           │
                             │  /css/style.css       │
                             │  /js/app.js           │
                             └──────────────────────┘

  For HTTPS + custom domain:
  ┌──────────┐  HTTPS   ┌────────────┐    HTTP    ┌──────────┐
  │  Browser  │────────►│ CloudFront  │──────────►│  S3 OAC  │
  └──────────┘          └────────────┘            └──────────┘
```

> [!WARNING]
> **Exam trap:** S3 static websites use **HTTP** only. For **HTTPS**, you must put [[CloudFront]] in front with an SSL certificate from ACM. If the question mentions "secure static website" or HTTPS → CloudFront + S3.

---

## Pre-Signed URLs

Generate temporary URLs to grant time-limited access to private objects:

| Aspect | Detail |
|---|---|
| **Expiration** | S3 Console: 1–720 min. CLI: up to 168 hours (7 days). |
| **Permissions** | Inherits the permissions of the IAM principal that generated it |
| **Use cases** | Download premium content, allow user uploads, temporary file sharing |

```
  ┌───────┐  1. Request pre-signed URL   ┌─────────┐
  │  App  │──────────────────────────────►│   S3    │
  │       │◄──────────────────────────────│         │
  └───┬───┘  2. Return signed URL         └─────────┘
      │                                        ▲
      │  3. Share URL with user               │
      ▼                                        │
  ┌───────┐  4. Access object directly        │
  │ User  │───────────────────────────────────┘
  │       │     (URL expires after set time)
  └───────┘
```

---

## S3 Access Points

Simplify managing access to shared datasets. Each Access Point has its own:
- DNS name (Internet or VPC origin)
- Access Point Policy (scoped to a prefix)

```
  ┌───────────────────────────────────────────────────────┐
  │                    S3 Bucket                          │
  │                                                       │
  │  /finance/  ──► Finance Access Point ──► Finance Team │
  │  /analytics/──► Analytics Access Point──► Data Team   │
  │  /sales/    ──► Sales Access Point   ──► Sales Team   │
  │                                                       │
  │  Instead of one massive bucket policy,                │
  │  each team gets a scoped Access Point                │
  └───────────────────────────────────────────────────────┘
```

### S3 Object Lambda Access Point

Transform objects on the fly using [[Lambda]] functions before returning to the caller. Use cases: redact PII, convert formats, resize images, enrich data.

---

## S3 Object Lock & Glacier Vault Lock

### S3 Object Lock (WORM)

Write Once Read Many — prevents objects from being deleted or overwritten:

| Mode | Description |
|---|---|
| **Governance Mode** | Only users with special permissions can delete/overwrite |
| **Compliance Mode** | **Nobody** can delete/overwrite, not even the root user |
| **Legal Hold** | Indefinite protection until explicitly removed. No retention period. |

### Glacier Vault Lock

Lock the vault policy so it can **never be changed or deleted** — for compliance and data retention.

> [!IMPORTANT]
> **Exam Pattern:** "Regulatory requirement that data cannot be deleted for 7 years" → **S3 Object Lock (Compliance Mode)** or **Glacier Vault Lock**. "Prevent even root user from deleting" → **Compliance Mode**.

---

## CORS (Cross-Origin Resource Sharing)

When a web application in one S3 bucket needs to access resources in another bucket:

```
  Browser loads page from:              Browser requests asset from:
  bucket-a.s3-website.amazonaws.com     bucket-b.s3.amazonaws.com
         │                                       ▲
         │                                       │
         └──── Cross-origin request ─────────────┘
                                        
  bucket-b must have CORS headers configured
  to allow requests from bucket-a's origin
```

> [!TIP]
> **Exam Pattern:** "Website hosted on S3 can't load fonts/images from another S3 bucket" → Enable **CORS** on the target bucket.

---

## MFA Delete

Adds an extra layer of protection by requiring **MFA** to:
- **Permanently delete** an object version.
- **Suspend versioning** on a bucket.

MFA is NOT required for enabling versioning or listing deleted versions. Can only be enabled by the **bucket owner (root account)** via the **CLI** (not the console).

---

## S3 Access Logs

- Log all access requests to an S3 bucket into **another S3 bucket** (in the same region).
- For analysis with [[Amazon Athena]], [[Amazon Redshift]], etc.

> [!CAUTION]
> **Never set the logging bucket to the same bucket being monitored** — this creates an infinite logging loop that grows exponentially.

---

## Exam Cheat Sheet

> [!TIP]
> **Trigger phrases → S3:**
> - "Infinitely scalable storage" or "store unlimited data" → **S3**
> - "11 nines durability" → **S3** (all classes)
> - "Static website hosting" → **S3** (+ CloudFront for HTTPS)
> - "Archive data for compliance" → **S3 Glacier** (choose tier by retrieval need)
> - "Lifecycle management" → **S3 Lifecycle Rules**
> - "Cross-region data replication" → **S3 CRR**
> - "Encrypt with customer-managed keys + audit" → **SSE-KMS**
> - "Prevent deletion, even by root" → **S3 Object Lock Compliance Mode**
> - "Speed up long-distance uploads" → **S3 Transfer Acceleration**
> - "Process files on upload" → **S3 Event Notification → Lambda**
> - "Query data in-place without downloading" → **S3 Select** or **Athena**
> - "Temporary access to private object" → **Pre-Signed URL**
> - "Unknown access pattern" → **S3 Intelligent-Tiering**
>
> **Key facts:**
> - Max object size: **5 TB**. Multi-part upload required > 5 GB.
> - 3,500 PUT / 5,500 GET per second per prefix.
> - Bucket names are **globally unique**, but buckets are **regional**.
> - Versioning is at the **bucket level**. Once enabled, can only be suspended (not disabled).
> - Replication requires versioning on both sides. Only new objects replicated.
> - Delete markers NOT replicated by default. Permanent deletes NEVER replicated.
> - SSE-S3 (AES-256) is the **default** encryption.
> - Block Public Access is **ON by default**.
