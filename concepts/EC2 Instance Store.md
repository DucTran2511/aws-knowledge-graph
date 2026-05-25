---
tags: [concept, storage, compute]
aliases: [Instance Store, EC2 Ephemeral Storage, Ephemeral Storage]
date: 2026-04-16
---

# EC2 Instance Store

We just talked about EBS volumes, which are virtual hard drives connected to your server over the network. EC2 Instance Store is the opposite: it is storage physically attached to the host that runs your EC2 instance, usually very fast local NVMe SSD storage.

Because there is no network hop between the CPU and the disk, Instance Store gives you the absolute highest performance, lowest latency, and highest IOPS available for EC2 storage.

![[assests/ec2/ec2_instance_storage.png]]

## The Ephemeral Trap
AWS calls Instance Store **ephemeral storage**, which means the data is temporary and tied to the physical host.

### When Data Survives
- **Reboot:** Data survives because the instance stays on the same physical host.

### When Data Is Lost
- **Stop:** Data is destroyed because AWS can later start the instance on different hardware.
- **Underlying hardware failure:** Data is destroyed because the local disk fails with the host.
- **Terminate:** Data is destroyed.

This is one of the most tested storage distinctions in the SAA-C03 exam.

## Instance Store Is Not RAM
Instance Store is not RAM. This is a very common point of confusion because both are tied to the physical host, but they are completely different hardware components with different jobs.

### 1. The Hardware Difference
- **RAM:** System memory used by the CPU to run active applications and the operating system in real time.
- **Instance Store:** A physical disk, usually local NVMe SSD storage, exposed as block storage. You format it with a file system such as `ext4`, create directories, and save files to it like a normal hard drive.

### 2. The Reboot Test
This is the distinction to memorize for the SAA-C03 exam:

- **If you reboot the EC2 instance:**
  - **RAM is wiped.**
  - **Instance Store survives**, because a reboot does not remove the instance from that physical host.

- **If you stop the EC2 instance:**
  - **RAM is wiped.**
  - **Instance Store is lost**, because AWS can restart the instance on different hardware later.

### 3. The Desk Analogy
- **RAM:** The top of your desk, where active work happens right now.
- **EBS:** A filing cabinet in the basement. It is persistent, but you reach it over the network.
- **Instance Store:** A filing cabinet bolted to the side of your desk. It is much faster to reach than the basement cabinet, but you lose it if you are moved to a different desk.

## Summary
- Put data in **RAM** when the CPU needs it immediately for active computation.
- Put data in **Instance Store** when the application needs extremely fast local disk access for temporary files, caches, or scratch data.

## Why Use It If Data Can Disappear?
You do not use Instance Store for primary databases, uploaded files, or anything that must survive a stop or host failure.

You use it for data that is temporary, reproducible, or disposable, where raw performance matters more than persistence.

## Real-World Use Cases
- **Caching layers:** Data can be rebuilt from a source of truth.
- **Buffer or scratch data:** Temporary working space for batch processing, media transcoding, or large intermediate files.
- **Temporary session data:** Possible, although Redis or ElastiCache is usually the better design.

## Identifying Instance Store Instances
Not all EC2 instance types include Instance Store volumes.

You generally look for:
- Instance types with the letter `d`, such as `m5d.large`
- Storage-optimized families, such as `i` and `d` families

For example, `m5d.large` includes Instance Store, while `m5.large` relies only on EBS.

## SAA-C03 Exam Cheat Sheet: EBS vs. Instance Store

| Feature | EBS | EC2 Instance Store |
| --- | --- | --- |
| Connection Type | Network-attached | Physically attached to the host |
| Data Persistence | Persistent | Ephemeral |
| Survives Stop | Yes | No |
| Survives Host Failure | Yes | No |
| Performance | High, but network-backed | Ultra-high, local disk performance |
| Common Keywords | Persistent storage, database, survive stop | Scratch data, cache, highest IOPS, temporary buffer |

## When to Choose It
- Choose **EBS** when storage must persist.
- Choose **EC2 Instance Store** when you need the fastest possible temporary local storage.
