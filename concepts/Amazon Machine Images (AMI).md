---
tags: [concept, compute, images]
aliases: [AMI, Amazon Machine Image, Golden AMI]
date: 2026-04-16
---

# Amazon Machine Images (AMI)

Now that EBS snapshots make sense, AMIs become much easier to understand.

An AMI is essentially an EBS snapshot packaged with the metadata AWS needs in order to launch an EC2 instance from it.

If an EBS snapshot is just a frozen copy of a disk, an AMI adds the instruction manual: what operating system is on the image, how it should boot, what storage layout it expects, and who is allowed to use it.

![[assests/ami/image.png]]

## The 3 Ingredients of an AMI

### 1. Root Volume Template
This is the underlying EBS snapshot that contains the operating system and any preinstalled application code.

### 2. Launch Permissions
These define which AWS accounts are allowed to use the AMI.

### 3. Block Device Mapping
These are the instructions for which volumes should exist when the instance boots, including the root volume and any additional attached volumes.

## SAA-C03 Exam Traps

### 1. The Region Lock
AMIs are regional resources.

If you create a custom AMI in one Region, you cannot directly use the same AMI ID in another Region. You must copy the AMI so AWS can copy the underlying snapshot data and create a new AMI in the target Region.

### 2. Golden AMI vs. Bootstrapping
AWS often tests whether you know when to pre-bake software into an image versus install it at launch time.

- **Use a Golden AMI:** When instances must launch very quickly, such as in an Auto Scaling Group reacting to traffic spikes.
- **Use bootstrapping with EC2 User Data:** When you want more flexibility and are changing application code frequently, so it is easier to install or fetch what you need during instance startup.

## Relationship to Snapshots
Custom AMIs for EBS-backed EC2 instances are built on top of EBS snapshots plus launch metadata.

That is why AMIs and EBS snapshots are closely related, but they are not the same thing:

- An **EBS snapshot** is a backup of a volume.
- An **AMI** is a launch template for creating EC2 instances.

## SAA-C03 Cheat Sheet
- AMIs are regional.
- AMIs use snapshots underneath.
- AMIs include launch permissions and block device mapping.
- Golden AMIs optimize for fast launch time.
- User Data bootstrapping optimizes for flexibility.
