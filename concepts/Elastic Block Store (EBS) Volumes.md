---
tags: [concept, storage]
aliases: [EBS Volumes, Amazon EBS]
date: 2026-04-16
---

# Elastic Block Store (EBS) Volumes

An Amazon Elastic Block Store (EBS) volume is, in the simplest terms, a virtual hard drive in the cloud.

If an EC2 instance is the CPU and RAM of your virtual computer, the EBS volume is the hard drive where the operating system, your Spring Boot application code, and your database files are actually saved.

Because this is the cloud, EBS volumes have a few "superpowers" that normal physical hard drives do not have.

![[assests/ebs/image.png]]

## What a Hard Drive Actually Is
A hard drive is the primary permanent storage device inside a computer. It is the hardware where the operating system, software applications, and files actually live.

To understand why AWS offers so many different storage services and EBS volume types, it helps to compare RAM and hard drives directly.

### The Desk vs. The Filing Cabinet
Think of a computer like an office:

- **RAM is your desk:** It is fast and easy to access, but space is limited. If the power goes out, everything in RAM is lost. This is **volatile memory**.
- **The hard drive is your filing cabinet:** It stores data permanently. When you save a file, the data is written from memory to storage. If power is lost, the stored files remain. This is **non-volatile storage**.

## The Two Main Types of Hard Drives

### 1. HDD (Hard Disk Drive)
This is the older storage technology.

Inside an HDD are spinning magnetic platters and a mechanical read/write arm. The drive stores data magnetically and the arm physically moves to the correct location to read or write it.

- **Pros:** Very cheap for large amounts of storage.
- **Cons:** Slower because it depends on physical moving parts.
- **AWS equivalent:** `st1` and `sc1` EBS volumes.

### 2. SSD (Solid State Drive)
This is the modern storage technology.

An SSD has no moving parts. It stores data on flash memory chips, similar to the technology used in USB drives and smartphones.

- **Pros:** Much faster and more durable than HDDs.
- **Cons:** More expensive per gigabyte.
- **AWS equivalent:** `gp2`, `gp3`, `io1`, and `io2` EBS volumes.

## Tying It Back to AWS
When you launch an EC2 instance, AWS is using real physical storage hardware in its data centers.

When you create an EBS volume, AWS allocates a portion of its underlying storage infrastructure to your account and exposes it to your EC2 instance as a virtual disk over the network.

That is why EBS behaves like a hard drive from the operating system's perspective, even though the actual storage hardware is managed by AWS behind the scenes.

## Core Concepts

### 1. It Is Network-Attached
This is the most important idea to understand.

An EBS volume is not a physical disk plugged directly into the server running your EC2 instance. Instead, it sits on AWS-managed storage infrastructure somewhere else in the data center and connects to your EC2 instance over AWS's high-speed internal network.

- **Benefit:** If the physical host running your EC2 instance fails, the data on the EBS volume is still safe. You can detach the volume and attach it to another EC2 instance.
- **Tradeoff:** Because storage traffic goes over the network, EBS has slightly higher latency than local instance storage.

### 2. The Availability Zone Lock
An EBS volume is bound to a single Availability Zone.

If you create an EBS volume in `us-east-1a`, you can only attach it to an EC2 instance that also runs in `us-east-1a`. You cannot directly attach that same volume to an instance in `us-east-1b`.

The normal workaround is to create an [[EBS Snapshots|EBS snapshot]] of the volume. That snapshot is stored using S3 under the hood, and AWS can use it to create a new EBS volume in a different Availability Zone or Region.

### 3. Volume Types
Choosing the right EBS volume type is a balancing act between paying for performance and controlling cost.

![[assests/ebs/ebs volume types/image.png]]

The first split to understand is this:

- **SSDs:** For transactional workloads with lots of small, random reads and writes. Think databases and boot volumes. Performance is measured mainly in **IOPS**.
- **HDDs:** For streaming workloads with large, sequential reads and writes. Think analytics, log pipelines, and data warehousing. Performance is measured mainly in **throughput**.

#### Category 1: SSDs

##### 1. General Purpose SSD (`gp2`, `gp3`)
This is the default choice for most EBS workloads.

Best use cases:
- EC2 boot volumes
- Standard relational databases with moderate traffic
- Development and test environments
- General application servers and enterprise workloads

Architect note:
- Prefer `gp3` over `gp2` for new architectures. `gp3` is typically cheaper and lets you scale IOPS and throughput separately from storage size.

##### 2. Provisioned IOPS SSD (`io1`, `io2`, `io2 Block Express`)
This is the high-performance option for workloads where sustained disk performance is critical.

Best use cases:
- Mission-critical, high-traffic relational databases
- NoSQL databases needing consistently high performance
- Workloads that require more than 16,000 IOPS

Exam keyword:
- If the question explicitly requires **EBS Multi-Attach**, you are forced into `io1` or `io2`.

#### Category 2: HDDs
HDD-backed EBS volumes are designed for large streaming workloads, not random transactional access.

> [!IMPORTANT]
> You can never use an HDD-backed EBS volume as an EC2 boot volume.

##### 3. Throughput Optimized HDD (`st1`)
This is the low-cost option for heavy sequential workloads.

Best use cases:
- Big Data analytics clusters
- Data warehousing
- Log processing and streaming pipelines

Exam keyword:
- Look for terms like **streaming**, **Big Data**, or **data warehouse**.

##### 4. Cold HDD (`sc1`)
This is the cheapest EBS volume type.

Best use cases:
- Rarely accessed data that still must remain attached to EC2
- Low-access archives or old server-side datasets

Exam keyword:
- Look for **lowest-cost EBS storage** when the data still needs to live on an attached volume.

#### SAA-C03 Decision Matrix
Use this quick filter when choosing an EBS volume type:

- **Does it need to be a boot drive?** Use `gp2`, `gp3`, `io1`, or `io2`. Eliminate HDDs immediately.
- **Does the workload need guaranteed sustained IOPS above 16,000?** Choose `io1` or `io2`.
- **Is it a Big Data or sequential streaming workload?** Choose `st1`.
- **Is the lowest possible EBS cost the top priority for rarely accessed attached data?** Choose `sc1`.

### 4. EBS Multi-Attach
Normally, an EBS volume can only be attached to one EC2 instance at a time.

EBS Multi-Attach is the exception to that rule. It allows one EBS volume to be attached to multiple EC2 instances at the same time, but only under strict conditions.

#### Rules of EBS Multi-Attach
- **Specific volume types only:** Only Provisioned IOPS SSD volumes (`io1` and `io2`) support Multi-Attach. General-purpose volumes such as `gp2` and `gp3` do not.
- **Same Availability Zone:** The volume and all attached EC2 instances must be in the exact same AZ.
- **Instance limit:** One Multi-Attach volume can be attached to up to 16 EC2 instances at the same time.
- **Nitro-based instances:** The EC2 instances must use the AWS Nitro System.

#### The Massive Catch: Data Corruption Risk
EBS Multi-Attach is not a normal shared file server.

If multiple instances attach the same volume and use a standard file system without coordination, concurrent writes can corrupt the data. Standard file systems such as `ext4` are not designed to let multiple separate machines safely write to the same raw block device at the same time.

To use Multi-Attach safely, you need a clustered file system or an application that explicitly manages distributed locking and coordinated writes.

#### SAA-C03 Exam Strategy: Multi-Attach vs. EFS
AWS often tests whether you can distinguish between shared block storage and shared file storage.

- **Choose [[Elastic File System (EFS)|Amazon EFS]]:** When multiple Linux instances need normal shared file access, especially across multiple AZs.
- **Choose EBS Multi-Attach:** Only when the workload explicitly needs shared block storage, extremely high IOPS, and coordinated access within a single AZ.

### 5. EBS Encryption
Data security is one of the most important parts of AWS architecture, and EBS encryption is designed to be simple to enable but still heavily tested on the SAA-C03 exam.

When you enable encryption on an EBS volume, AWS uses KMS-backed encryption with AES-256 under the hood.

#### What Gets Encrypted
Once an EBS volume is encrypted, AWS protects:

- **Data at rest:** The actual data stored on the underlying disks in AWS.
- **Data in transit:** Traffic moving between the EC2 instance and the EBS volume.
- **All snapshots:** Any [[EBS Snapshots|snapshot]] taken from the encrypted volume is also encrypted.
- **Derived volumes:** Any new volume created from those encrypted snapshots is also encrypted.

Architect note:
- This happens transparently on the EC2 side. Your application code does not need to change, and the performance overhead is generally negligible.

#### Default Encryption
In production, the best practice is to enable **default EBS encryption** at the account and Region level.

That way, every new EBS volume created in that Region is encrypted automatically, even if someone forgets to select the encryption checkbox manually.

#### The Ultimate Exam Trap: Encrypting an Existing Unencrypted Volume
You cannot directly flip an existing unencrypted volume into an encrypted one.

If AWS gives you a scenario where an unencrypted EBS volume must be encrypted without losing data, the standard workflow is:

1. Take a snapshot of the unencrypted volume.
2. Copy that snapshot and enable encryption during the copy process.
3. Create a new encrypted EBS volume from the encrypted snapshot.
4. Detach the old unencrypted volume and attach the new encrypted volume instead.

This is one of the most common EBS security questions on the exam.

#### Sharing Encrypted Snapshots Across Accounts
Sharing encrypted snapshots across AWS accounts has an extra requirement: the receiving account must be allowed to use the KMS key.

To do this safely:

- Use a **customer managed KMS key**, not just the default AWS-managed one.
- Update the KMS key policy to allow the other AWS account to use the key.
- Share the encrypted snapshot.
- The receiving account can then copy the snapshot for use in its own account.

## Common Use Cases
- Root volume for an EC2 instance
- Persistent storage for applications
- Database storage
- Data that must survive instance stop/start cycles or instance replacement

## SAA-C03 Storage Cheat Sheet
Use this logic when choosing between common AWS storage options:

- If storage must be attached to exactly one EC2 instance, choose **EBS**.
- If storage must be mounted by multiple Linux instances across multiple AZs, choose [[Elastic File System (EFS)|EFS]].
- If you need the absolute lowest latency local storage attached to the host itself, choose [[EC2 Instance Store]].
- If the data is a backup, snapshot, or object such as an image, video, or static file, choose **S3**.
