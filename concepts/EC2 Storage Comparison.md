---
tags: [concept, storage, compute, comparison]
aliases: [EBS vs EFS vs Instance Store, AWS Storage Comparison]
date: 2026-04-22
---

# EC2 Storage Comparison

This is the core AWS compute storage showdown: [[EC2 Instance Store]] vs [[Elastic Block Store (EBS) Volumes|Amazon EBS]] vs [[Elastic File System (EFS)|Amazon EFS]].

If you want to design cost-effective cloud architectures and answer SAA-C03 storage questions quickly, you need to look at a workload and immediately know which of these three fits.

## The Ultimate Storage Comparison Table

| Feature | [[EC2 Instance Store]] | [[Elastic Block Store (EBS) Volumes|Amazon EBS]] | [[Elastic File System (EFS)|Amazon EFS]] |
| --- | --- | --- | --- |
| **Physical Location** | Physically attached to the host | Network-attached in one AZ | Network-attached shared file system across a Region |
| **Persistence** | **Ephemeral.** Lost on stop, terminate, or host failure | **Persistent.** Survives instance stops and supports snapshots | **Persistent.** Highly durable shared file storage |
| **Scope / Attachment** | **1-to-1** with the host | Usually **1-to-1** with one instance, except Multi-Attach cases | **1-to-many** across many EC2 instances |
| **Boot Volume?** | Rarely used this way today | **Yes.** Standard boot-volume choice | **No** |
| **Pricing Model** | Included in the cost of supported instance types | Pay for what you provision | Pay for what you store |
| **OS Support** | Linux and Windows | Linux and Windows | Linux-oriented shared NFS storage |

## SAA-C03 Decision Tree

### 1. The Temporary / Ultra-High IOPS Rule
Scenario:
The workload needs extremely high local performance with no network hop, and the data is temporary, reproducible, or disposable.

Answer:
- [[EC2 Instance Store]]

### 2. The Boot Drive / Single Database Rule
Scenario:
You need a boot drive, or a standard relational database needs dedicated persistent block storage in one AZ.

Answer:
- [[Elastic Block Store (EBS) Volumes|Amazon EBS]]

### 3. The Shared Linux Rule
Scenario:
A fleet of Linux instances across multiple AZs must read and write the same files at the same time.

Answer:
- [[Elastic File System (EFS)|Amazon EFS]]

### 4. The Windows Shared Trap
Scenario:
Multiple Windows instances need a shared file system.

Answer:
- **Amazon FSx for Windows File Server**

## Quick Memory Rules
- If the data is **temporary and speed matters most**, choose [[EC2 Instance Store]].
- If the data is **persistent block storage for one instance**, choose [[Elastic Block Store (EBS) Volumes|EBS]].
- If the data is **shared file storage for many Linux instances**, choose [[Elastic File System (EFS)|EFS]].
