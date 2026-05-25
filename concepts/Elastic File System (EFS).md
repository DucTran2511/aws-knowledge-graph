---
tags: [concept, storage, shared-files]
aliases: [EFS, Amazon EFS, Elastic File System]
date: 2026-04-17
---

# Elastic File System (EFS)

If Amazon EBS is a private virtual hard drive attached to one machine, Amazon EFS is the shared network drive that many machines can use at the same time.

In technical terms, EFS is a fully managed NFS file system. It solves the problem that EBS does not: letting many EC2 instances read and write the same files safely at the same time.

## Core Concepts

### 1. The Superpower: Multi-AZ and Multi-Instance
The biggest limitation of EBS is that a volume is locked to one Availability Zone.

EFS breaks that rule. A standard EFS file system is built for regional availability and can be mounted by EC2 instances across multiple AZs in the same Region.

That means:

- An EC2 instance in `us-east-1a` can mount the same EFS file system as an EC2 instance in `us-east-1b`.
- If one instance writes a file, the other instance can access that same file through the shared file system.

This is why EFS is commonly used for shared uploads, shared home directories, and fleets of web servers that need common file access.

### 2. The Elastic in EFS
With EBS, you provision a fixed volume size up front.

EFS works differently. You do not preallocate storage size in the same way. The file system grows as you add files and shrinks as you remove them, and billing is based on the amount of data stored.

This lets EFS scale automatically to very large datasets without manual resizing.

### 3. Storage Classes and Cost Control
EFS is usually much more expensive per gigabyte than EBS, so cost optimization matters.

The main storage choices are:

- **EFS Standard:** Multi-AZ, highly available storage for actively accessed files.
- **EFS Standard-IA:** Lower-cost storage for infrequently accessed files, usually paired with lifecycle policies.
- **EFS One Zone:** Lower-cost EFS stored in only one AZ.

Lifecycle policies can automatically move older files from Standard to Standard-IA after a chosen period of inactivity.

> [!WARNING]
> EFS One Zone is cheaper, but it is not Multi-AZ. If that AZ fails, the file system is unavailable and data resilience is lower. Use it only when that tradeoff is acceptable.

## The Big SAA-C03 Exam Traps

### Trap 1: The Linux Rule
EFS uses NFS and is the normal shared file-system answer for Linux-based EC2 fleets.

If the scenario asks for a shared file system for Windows servers, the answer is not EFS. The expected answer is typically **Amazon FSx for Windows File Server**.

### Trap 2: EFS vs. EBS Multi-Attach
AWS frequently tries to confuse these two.

- **Choose EFS:** When standard Linux instances need normal shared file access such as uploads, config files, or shared directories.
- **Choose EBS Multi-Attach:** When a specialized clustered application needs shared block storage with extremely high IOPS in a single AZ.

### Trap 3: EFS vs. S3
Both EFS and S3 can hold very large amounts of data, but they solve different problems.

- **Choose S3:** When the application is built around object APIs such as `PutObject` and `GetObject`, and cost matters.
- **Choose EFS:** When the application expects a traditional mounted file system with directories, file paths, and normal operating-system file operations.

## Common Use Cases
- Shared uploads for a fleet of web servers
- Shared content directories across multiple EC2 instances
- Shared home directories for Linux users
- Lift-and-shift applications that require a POSIX-style shared file system

## SAA-C03 Cheat Sheet
- EFS is shared file storage for many EC2 instances.
- EFS works across multiple AZs in a Region.
- EFS is the standard shared file-system answer for Linux fleets.
- EFS is not the answer for Windows shared file systems.
- EFS is more expensive than EBS, so lifecycle and storage class choices matter.
