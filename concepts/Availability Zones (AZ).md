---
tags: [concept, infrastructure, regions]
aliases: [Availability Zone, AZ, AZs]
date: 2026-04-16
---

# Availability Zones (AZ)

An Availability Zone (AZ) is the fundamental building block of high availability in AWS.

When you deploy infrastructure in AWS, it does not live in an abstract cloud. It runs in real physical facilities. If you understand the difference between a Region and an Availability Zone, the rest of AWS high availability and disaster recovery starts to make much more sense.

## How AWS Organizes Physical Infrastructure

### 1. Region: The City
A Region is a physical geographic location in the world, such as `us-east-1` in Northern Virginia, `ap-southeast-1` in Singapore, or `eu-central-1` in Frankfurt.

You can think of a Region as a city.

Every Region is independent and contains multiple Availability Zones.

#### Why You Choose a Specific Region
- **Latency:** Put infrastructure close to your users so applications respond faster.
- **Data compliance:** Some workloads must stay inside specific countries or legal jurisdictions.
- **Pricing:** AWS service pricing varies by Region.

> [!IMPORTANT]
> Unless you explicitly configure cross-Region replication or copying, data stored in one Region stays in that Region.

### 2. Availability Zone: The Isolated Campus
Inside each Region, AWS operates multiple Availability Zones such as `us-east-1a`, `us-east-1b`, and `us-east-1c`.

You can think of an Availability Zone as an isolated campus inside that city.

An AZ is a cluster of one or more physical data centers with strict isolation boundaries:

- **Physical separation:** AZs in the same Region are separated by meaningful distance, often miles apart.
- **Independent infrastructure:** Each AZ has its own power, backup generators, cooling, and network connectivity.
- **Failure isolation:** AZs are designed so a localized disaster affects one AZ without taking down the others in the same Region.

### 3. Data Centers: The Buildings
One Availability Zone is not usually just one building.

An AZ is a logical grouping of one or more physical data centers that are close together and engineered to operate as a fault-isolated unit.

### 4. The Network Between AZs
Although AZs are physically separated, they are connected by very high-speed private fiber links.

This is why AWS can offer low-latency communication between AZs in the same Region while still keeping them far enough apart for fault isolation.

## Why Regions and AZs Matter
The point of AZs is to let you build highly available architectures.

If your application runs entirely inside one AZ and that AZ fails, your application can go offline even if the rest of the Region is healthy.

The standard AWS best practice is a **Multi-AZ deployment**:

- Put one web server in `us-east-1a`.
- Put another web server in `us-east-1b`.
- Put a load balancer in front of them.

If one AZ fails, the load balancer routes traffic to healthy instances in the surviving AZ.

That is the core idea behind high availability in AWS: do not depend on a single AZ.

## SAA-C03 Architecture Strategy
The exam constantly tests whether the right answer is **Multi-AZ** or **Multi-Region**.

### 1. High Availability
- **Strategy:** Multi-AZ
- **Why:** Protects against failure of a single data center or AZ while keeping the application online inside the same Region.
- **Common pattern:** Put instances in two AZs behind a load balancer.

Exam keywords:
- High availability
- Fault tolerance
- Protection from a single data center failure

### 2. Disaster Recovery
- **Strategy:** Multi-Region
- **Why:** Protects against loss of an entire Region and can also help serve globally distributed users with lower latency.
- **Common pattern:** Run a primary environment in one Region and replicate critical data to another Region.

Exam keywords:
- Entire Region failure
- Geographic disaster
- Global users with lowest latency

## The Account Mapping Trick
The AZ letters are not globally consistent between AWS accounts.

Your `us-east-1a` is often not the same physical AZ as another account's `us-east-1a`. AWS maps these labels differently per account so that customers do not all pile into the same physical facility.

That means the letters are account-specific logical labels, not universal physical identifiers.

## Architecture Implications
- A subnet belongs to exactly one AZ.
- An EBS volume belongs to exactly one AZ.
- Many high-availability AWS architectures spread resources across at least two AZs in the same Region.
- Multi-AZ improves fault tolerance, but it is not the same thing as multi-Region disaster recovery.
