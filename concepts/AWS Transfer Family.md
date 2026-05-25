---
tags: [concept, storage, data-migration, managed-service]
aliases: [AWS Transfer Family, SFTP, FTPS, FTP, Transfer for SFTP]
date: 2026-05-21
---

# AWS Transfer Family

**AWS Transfer Family** provides fully managed support for **file transfers** directly into and out of [[Amazon S3]] and [[Elastic File System (EFS)|Amazon EFS]] using standard file transfer protocols.

> [!IMPORTANT]
> **Core exam concept:** AWS Transfer Family is the answer when the question involves **SFTP/FTPS/FTP/AS2** file transfers into S3 or EFS. It replaces the need to run and manage your own file transfer servers.

---

## Supported Protocols

```
  ┌────────────────────────────────────────────────────────────┐
  │                  AWS Transfer Family                       │
  │                                                            │
  │  External Partners / Users                                 │
  │       │          │          │          │                    │
  │       │ SFTP     │ FTPS     │ FTP      │ AS2               │
  │       │          │          │          │                    │
  │       ▼          ▼          ▼          ▼                    │
  │  ┌───────────────────────────────────────────────┐         │
  │  │         AWS Transfer Family Endpoint          │         │
  │  │         (fully managed)                       │         │
  │  └──────────────────┬────────────────────────────┘         │
  │                     │                                      │
  │          ┌──────────┴──────────┐                           │
  │          ▼                     ▼                            │
  │    ┌───────────┐        ┌───────────┐                      │
  │    │    S3     │        │   EFS     │                      │
  │    │  Bucket   │        │   File    │                      │
  │    │           │        │   System  │                      │
  │    └───────────┘        └───────────┘                      │
  └────────────────────────────────────────────────────────────┘
```

| Protocol | Description | Encryption |
|---|---|---|
| **SFTP** | SSH File Transfer Protocol | Encrypted via SSH |
| **FTPS** | FTP over TLS/SSL | Encrypted via TLS |
| **FTP** | Plain File Transfer Protocol | ❌ No encryption (use within VPC only) |
| **AS2** | Applicability Statement 2 | Encrypted — B2B data exchange |

> [!WARNING]
> **FTP** (plain, unencrypted) is supported but should only be used **within a VPC** endpoint. For internet-facing transfers, use **SFTP** or **FTPS**.

---

## Architecture

```
  External Partners                        AWS Cloud
  ┌─────────────────────┐                  ┌──────────────────────┐
  │  Partner A (SFTP)    │                  │                      │
  │       │              │                  │  ┌────────────────┐  │
  │  Partner B (FTPS)    │                  │  │ Transfer Family│  │
  │       │              │                  │  │ Server         │  │
  │  Legacy System (FTP) │   Public or      │  │                │  │
  │       │              │   VPC endpoint   │  │ • Auth (IAM,   │  │
  │       │              │──────────────────│──│   AD, Lambda,  │  │
  │       │              │                  │  │   Custom IdP)  │  │
  │       │              │                  │  │                │  │
  │       │              │                  │  │ • IAM Role per │  │
  │       │              │                  │  │   user → S3/EFS│  │
  │       └──────────────│──────────────────│──│   permissions  │  │
  │                      │                  │  └───────┬────────┘  │
  │                      │                  │          │           │
  │                      │                  │  ┌───────▼────────┐  │
  │                      │                  │  │   S3 or EFS    │  │
  │                      │                  │  └────────────────┘  │
  └─────────────────────┘                  └──────────────────────┘
```

| Feature | Detail |
|---|---|
| **Storage backend** | [[Amazon S3]] or [[Elastic File System (EFS)\|Amazon EFS]] |
| **Endpoint types** | Public (internet-facing), VPC (internal), VPC with public IP |
| **Authentication** | Service-managed (SSH keys), AWS Directory Service (AD), Custom Identity Provider (Lambda / API Gateway) |
| **Scaling** | Fully managed — auto-scales |
| **DNS** | Assign a custom hostname via [[Amazon Route 53]] |
| **Pricing** | Per protocol enabled per hour + per GB transferred |

---

## Authentication Options

| Method | Description |
|---|---|
| **Service-managed** | Store SSH public keys in the Transfer Family service — simplest |
| **AWS Directory Service** | Authenticate against Microsoft Active Directory |
| **Custom IdP** | Use a Lambda function or API Gateway to authenticate against your own identity provider |

Each user is mapped to an **IAM Role** that defines their S3/EFS permissions (which bucket, which prefix, read/write).

---

## Common Use Cases

- **B2B file exchanges** — partners upload/download files via SFTP/FTPS
- **Data ingestion** — external systems push data into S3 for analytics pipelines
- **Legacy migration** — replace self-managed FTP servers with a managed service
- **Compliance workflows** — AS2 protocol for EDI (Electronic Data Interchange) in healthcare, retail, finance

---

## Exam Cheat Sheet

> [!TIP]
> **Trigger phrases → AWS Transfer Family:**
> - "SFTP into S3" or "FTP server for S3" → **AWS Transfer Family**
> - "Partners need to upload files via SFTP" → **AWS Transfer Family**
> - "Replace self-managed FTP server" → **AWS Transfer Family**
> - "AS2 protocol for B2B exchange" → **AWS Transfer Family**
>
> **Key facts:**
> - Supports SFTP, FTPS, FTP, and AS2 protocols
> - Backend storage: S3 or EFS
> - FTP (unencrypted) → VPC endpoint only
> - Custom identity provider via Lambda for flexible authentication
> - Fully managed — no servers to manage
