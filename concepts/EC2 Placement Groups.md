---
tags: [concept, compute, high-availability]
aliases: [Placement Groups, Cluster Placement, Spread Placement, Partition Placement]
date: 2026-04-14
---

# EC2 Placement Groups

When you launch new [[EC2]] instances, AWS automatically places them across underlying hardware to minimize correlated failures. However, you can use **Placement Groups** to strictly influence how your instances are placed to meet the specific networking or high-availability needs of your workload.

There are three types of Placement Groups:

### 1. Cluster Placement Group
- **What it is:** Packs instances close together inside a single Availability Zone (same rack/network switch).
- **Use case:** Workloads that need incredibly low latency, high network throughput, or both.
- **Example:** High Performance Compute (HPC) applications, machine learning training, tightly coupled node-to-node communication.
- **Pros:** Maximum network performance (up to 10 Gbps single flow or 100 Gbps aggregate).
- **Cons:** If the underlying hardware rack fails, all instances might fail together (low availability).

### 2. Spread Placement Group
- **What it is:** Strictly places a small group of instances across distinct underlying hardware (distinct racks, each with its own network and power source).
- **Use case:** A small number of critical instances that must be kept strictly separate from each other to prevent simultaneous failures.
- **Example:** A small but critical database cluster with primary and replica instances.
- **Pros:** Maximizes high availability.
- **Cons:** Limited to a maximum of 7 instances per Availability Zone per group.

### 3. Partition Placement Group
- **What it is:** Divides each group into logical segments called *partitions*. Instances in one partition do not share underlying hardware with instances in another partition.
- **Use case:** Large, distributed, and scalable workloads across distinct hardware.
- **Example:** Big data tools like Hadoop, Cassandra, Kafka clusters.
- **Pros:** Protects massive workloads from single hardware failures. Scales to many instances (unlike Spread, which is limited).
- **Visibility:** Instances can "see" which partition they are in (via metadata API) so tools like Hadoop know data is stored robustly.
