---
tags: [concept, storage, s3, security, encryption, kms]
aliases: [S3 Encryption, S3 SSE, SSE-S3, SSE-KMS, SSE-C, S3 Server-Side Encryption]
date: 2026-05-08
---

# S3 Encryption

[[Amazon S3]] supports multiple encryption mechanisms to protect data at rest and in transit. Encryption is a **heavily tested topic** across all AWS certification exams.

> [!IMPORTANT]
> **As of January 2023**, all new objects uploaded to S3 are **encrypted by default** with **SSE-S3** (AES-256). You can override this with SSE-KMS, SSE-C, or client-side encryption.

---

## Server-Side Encryption (SSE)

S3 encrypts objects **after receiving them** and decrypts them **before sending them back**.

### SSE-S3 (S3-Managed Keys)

```
  ┌───────┐   Object    ┌─────────────────────────────────┐
  │ Client │────────────►│            Amazon S3             │
  └───────┘              │                                 │
                         │   1. Receives object            │
                         │   2. Generates unique AES-256   │
                         │      key per object             │
                         │   3. Encrypts object            │
                         │   4. Encrypts key with a        │
                         │      root key (managed by S3)   │
                         │   5. Stores encrypted object    │
                         │      + encrypted key            │
                         └─────────────────────────────────┘

  Header: "x-amz-server-side-encryption": "AES256"
```

| Aspect | Detail |
|---|---|
| **Key management** | Fully managed by AWS — you never see the key |
| **Header** | `x-amz-server-side-encryption: AES256` |
| **Default** | ✅ This is the default encryption for all new objects |
| **Audit** | ❌ Cannot audit individual key usage |
| **Cost** | Free |

---

### SSE-KMS (KMS-Managed Keys)

```
  ┌───────┐   Object    ┌──────────┐        ┌──────────────┐
  │ Client │────────────►│    S3    │───────►│   AWS KMS    │
  └───────┘              │          │◄───────│              │
                         │ Encrypts │ Data   │ Manages      │
                         │ with DEK │ Key    │ CMK + DEK    │
                         └──────────┘        └──────────────┘

  Header: "x-amz-server-side-encryption": "aws:kms"

  Envelope Encryption:
  ┌─────────────────────────────────────────────┐
  │  KMS CMK encrypts ──► Data Encryption Key   │
  │  Data Encryption Key encrypts ──► Object    │
  │                                             │
  │  Object stored with encrypted DEK           │
  │  CMK never leaves KMS                       │
  └─────────────────────────────────────────────┘
```

| Aspect | Detail |
|---|---|
| **Key management** | You control via KMS — use AWS-managed or customer-managed CMK |
| **Header** | `x-amz-server-side-encryption: aws:kms` |
| **Audit** | ✅ Every key usage logged in **CloudTrail** |
| **Key rotation** | ✅ Automatic annual rotation (for AWS-managed CMK) or on-demand |
| **Cost** | KMS API calls charged (GenerateDataKey, Decrypt) |

> [!WARNING]
> **KMS Quota Limitation:** Every S3 GET/PUT with SSE-KMS calls KMS APIs. KMS has a **per-second quota** (5,500 to 30,000 depending on region). For high-throughput workloads, this can become a **bottleneck**.
>
> Solutions:
> - Request a **KMS quota increase** via Service Quotas.
> - Use **S3 Bucket Keys** to reduce KMS API calls by up to 99%.

### S3 Bucket Keys

```
  Without Bucket Key:                     With Bucket Key:
  
  Each object ──► KMS API call            S3 generates a bucket-level
  Each object ──► KMS API call            key from KMS, then uses it
  Each object ──► KMS API call            to encrypt objects locally
  (N objects = N KMS calls)               
                                          Bucket Key ──► KMS (1 call)
                                          Object 1 ──► local encrypt
                                          Object 2 ──► local encrypt
                                          Object 3 ──► local encrypt
                                          (N objects ≈ 1 KMS call)
```

- Reduces KMS API calls by up to **99%** → reduces cost and avoids throttling.
- The bucket-level key is cached and rotated periodically.

> [!TIP]
> **Exam Pattern:** "SSE-KMS is being throttled" or "reduce KMS costs for S3" → Enable **S3 Bucket Keys**.

---

### SSE-C (Customer-Provided Keys)

```
  ┌───────┐   Object + Key (HTTPS only!)    ┌──────────┐
  │ Client │────────────────────────────────►│    S3    │
  └───────┘                                  │          │
                                             │ 1. Uses YOUR key to  │
                                             │    encrypt object    │
                                             │ 2. Discards key     │
                                             │    immediately       │
                                             │ 3. Stores encrypted  │
                                             │    object + key hash │
                                             └──────────┘

  On retrieval: Client must provide the SAME key again.
  S3 verifies the hash, decrypts, and returns the object.
```

| Aspect | Detail |
|---|---|
| **Key management** | Fully managed by the customer |
| **HTTPS** | **Required** — keys are sent in the request |
| **Key stored by AWS?** | ❌ No — S3 discards the key after use |
| **Console support** | ❌ Cannot use AWS Console (CLI/SDK only) |
| **Use case** | Regulatory requirement to manage own keys outside AWS |

> [!CAUTION]
> If you lose the encryption key, the data is **permanently unrecoverable**. AWS does not store your key.

---

## Client-Side Encryption

The client encrypts data **before** uploading to S3. AWS never sees the plaintext.

```
  ┌───────┐  1. Encrypt locally    ┌──────────────┐    2. Upload    ┌──────┐
  │ Client │──────────────────────►│ Encrypted    │───────────────►│  S3  │
  │        │                       │ Object       │                │      │
  └───────┘                        └──────────────┘                └──────┘
                                   
  Client manages entire encryption lifecycle:
  • Key generation
  • Encryption algorithm
  • Key storage & rotation
  
  AWS S3 stores opaque encrypted bytes.
```

| Aspect | Detail |
|---|---|
| **Key management** | 100% client responsibility |
| **AWS involvement** | None — S3 stores encrypted bytes |
| **Use case** | Maximum control, compliance requirements |
| **Libraries** | AWS Encryption SDK, S3 Encryption Client |

---

## Encryption in Transit (TLS/SSL)

| Endpoint | Protocol | Encryption |
|---|---|---|
| `http://s3.amazonaws.com` | HTTP | ❌ Not encrypted |
| `https://s3.amazonaws.com` | HTTPS | ✅ TLS encrypted |

You can **force HTTPS** by adding a bucket policy with a condition:

```json
{
  "Effect": "Deny",
  "Principal": "*",
  "Action": "s3:*",
  "Resource": "arn:aws:s3:::my-bucket/*",
  "Condition": {
    "Bool": {
      "aws:SecureTransport": "false"
    }
  }
}
```

---

## Default Encryption vs Bucket Policy

| Method | Behavior |
|---|---|
| **Default encryption** | Applied to objects that don't specify encryption at upload |
| **Bucket policy** | Can **deny** uploads that don't include a specific encryption header |

```
  Priority:
  
  1. Bucket Policy (evaluated FIRST)
     └── Deny if no encryption header? → Request rejected
  
  2. Default Encryption (evaluated SECOND)
     └── If request has no encryption header and policy allows it,
         default encryption is applied
```

> [!NOTE]
> Bucket policies are evaluated **before** default encryption. A policy that denies unencrypted uploads will reject the request before default encryption can apply.

---

## Comparison Table

| Feature | SSE-S3 | SSE-KMS | SSE-C | Client-Side |
|---|---|---|---|---|
| **Key managed by** | AWS S3 | AWS KMS (you control) | Customer | Customer |
| **Key visible to you** | ❌ | Via KMS console | ✅ | ✅ |
| **CloudTrail audit** | ❌ | ✅ | ❌ | ❌ |
| **Key rotation** | Automatic | Configurable | Manual | Manual |
| **HTTPS required** | No (recommended) | No (recommended) | **Yes** | No |
| **Console upload** | ✅ | ✅ | ❌ | ❌ |
| **Header** | `AES256` | `aws:kms` | Customer key in header | N/A |
| **Default** | ✅ | Can be set as default | ❌ | ❌ |
| **Cost** | Free | KMS API charges | Free (you manage key) | Free |
| **KMS throttling risk** | No | Yes (use Bucket Keys) | No | No |

---

## Exam Cheat Sheet

> [!TIP]
> **Trigger phrases → Encryption type:**
> - "Default encryption" or "simplest encryption" → **SSE-S3**
> - "Audit encryption key usage" or "CloudTrail for keys" → **SSE-KMS**
> - "Control key rotation schedule" → **SSE-KMS** (customer-managed CMK)
> - "Company must manage keys on-premises" → **SSE-C**
> - "Encrypt before upload" or "client controls everything" → **Client-Side Encryption**
> - "KMS API throttling" or "reduce KMS cost" → **S3 Bucket Keys**
> - "Force HTTPS" → Bucket policy with `aws:SecureTransport` condition
>
> **Key facts:**
> - SSE-S3 is the **default** since Jan 2023
> - SSE-C requires **HTTPS** (keys travel in the request)
> - SSE-KMS has **quota limits** (5,500–30,000 req/sec)
> - S3 Bucket Keys reduce KMS calls by **up to 99%**
> - Bucket policy is evaluated **before** default encryption
