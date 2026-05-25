---
tags: [concept, storage, backup]
aliases: [EBS Snapshot, Snapshots]
date: 2026-04-16
---

# EBS Snapshots

An EBS snapshot is a point-in-time backup of an EBS volume.

If an EBS volume is a virtual hard drive, a snapshot is a frozen picture of exactly what data was on that volume at the moment the snapshot was created.

![[assests/ebs/image 2.png]]

## Core Mechanics

### 1. Where Snapshots Live
Although you manage EBS snapshots from the EC2 side of AWS, snapshots are stored behind the scenes using Amazon S3 infrastructure.

You do not see them as normal files in your own S3 buckets, but AWS uses S3's durability and scale to store them safely.

### 2. Incremental Backups
This is the most important concept to understand: EBS snapshots are incremental.

AWS does not copy the full volume every time you create a new snapshot. It only stores the blocks that have changed since the previous snapshot.

- **First snapshot:** AWS copies all blocks that contain data.
- **Later snapshots:** AWS copies only changed blocks.

This reduces storage costs and makes repeated backups faster. Even though later snapshots only store changed data, AWS can still reconstruct a full volume from any snapshot in the chain.

### 3. Deleting Snapshots
Deleting an older snapshot does not automatically break newer ones.

AWS tracks which blocks are still needed and preserves that data in the remaining snapshots. That means snapshots can generally be deleted without corrupting newer restore points.

### 4. Escaping the AZ Lock
An EBS volume belongs to a single Availability Zone, but a snapshot is a regional backup.

That is how you move EBS-backed data to another Availability Zone:

1. Create a snapshot of the EBS volume.
2. Create a new EBS volume from that snapshot in the target AZ.
3. Attach the new volume to an EC2 instance in that AZ.

Snapshots can also be copied to another Region, which is a common disaster recovery pattern.

### 5. AMIs Are Built on Snapshots
When you create a custom [[Amazon Machine Images (AMI)|AMI]] for an EC2 instance, AWS uses snapshots of the instance's EBS volumes and combines them with boot metadata and launch configuration.

That is why snapshots are foundational not just for backup, but also for image-based deployment workflows.

## SAA-C03 Exam Focus
- EBS snapshots are point-in-time backups of EBS volumes.
- Snapshots are incremental.
- Snapshots are the standard way to move EBS-backed data across AZs or Regions.
- Custom [[Amazon Machine Images (AMI)|AMIs]] for EBS-backed instances rely on snapshots.
