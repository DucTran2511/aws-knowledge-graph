---
tags: [concept, caching, in-memory, performance, high-availability, managed-service]
aliases: [ElastiCache, Amazon ElastiCache, AWS ElastiCache, Redis, Memcached]
date: 2026-04-30
---

# Amazon ElastiCache

**Amazon ElastiCache** is a fully managed **in-memory caching service** that makes it easy to deploy, operate, and scale an in-memory data store in the cloud. It dramatically improves application performance by retrieving data from fast, in-memory caches instead of slower disk-based databases.

ElastiCache supports two open-source engines:
- **Redis** (or Valkey) — feature-rich, persistent, highly available
- **Memcached** — simple, multi-threaded, volatile

> [!IMPORTANT]
> **Core exam concept:** ElastiCache sits **between your application and your database**. It reduces read load on the database, delivers sub-millisecond latency, and makes your application architecture more scalable. Think of it as the "[[Amazon RDS|RDS]] companion" for caching.

---

## Why Use ElastiCache?

The problem: Databases like [[Amazon RDS]] and [[Amazon Aurora]] are optimized for durable storage, not for ultra-fast reads of frequently accessed data. Every read hits the database, increasing latency and cost.

The solution: Put a cache layer in front of the database.

```
Without Cache:                          With ElastiCache:

┌──────┐       ┌──────────┐            ┌──────┐       ┌─────────────┐       ┌──────────┐
│ App  │──────►│   RDS    │            │ App  │──────►│ ElastiCache │       │   RDS    │
│      │◄──────│          │            │      │◄──────│  (in-mem)   │       │          │
└──────┘       └──────────┘            └──────┘       └──────┬──────┘       └──────────┘
  Every read hits the DB                                     │ Cache miss?       ▲
  (slow, expensive)                                          └──────────────────┘
                                                              Fetch from DB, store in cache
```

### Common Use Cases

| Use Case | Description |
|---|---|
| **Database query caching** | Cache frequently queried data to offload [[Amazon RDS]] / [[Amazon Aurora]] |
| **Session store** | Store user sessions in ElastiCache so the app tier becomes **stateless** |
| **Leaderboards / Sorted Sets** | Redis Sorted Sets enable real-time ranking (gaming, social) |
| **Real-time analytics** | Sub-millisecond reads for dashboards and counters |
| **Pub/Sub messaging** | Redis supports publish/subscribe patterns |
| **Geospatial indexing** | Redis native geospatial commands for location-based queries |

> [!TIP]
> **Exam Pattern:** "Make web application stateless" + "remove sticky sessions from [[Elastic Load Balancer (ELB)|ELB]]" → **ElastiCache as a session store**. User session data is stored in ElastiCache so any [[EC2]] instance can serve any user request.

---

## Redis vs Memcached

This is the **#1 exam topic** for ElastiCache. You must know when to choose each engine.

| Feature | Redis | Memcached |
|---|---|---|
| **Data Structures** | Strings, Hashes, Lists, Sets, Sorted Sets, Bitmaps, Streams | Strings only (simple key-value) |
| **Persistence** | ✅ Yes (Snapshots + AOF) | ❌ No (pure volatile cache) |
| **High Availability** | ✅ Multi-AZ with auto-failover | ❌ No HA |
| **Read Replicas** | ✅ Up to 5 replicas per shard | ❌ No replication |
| **Backup / Restore** | ✅ Yes | ❌ No |
| **Multi-threading** | ❌ Single-threaded (event loop) | ✅ Multi-threaded |
| **Cluster Mode** | ✅ Sharding across nodes | ✅ Auto Discovery |
| **Pub/Sub** | ✅ Yes | ❌ No |
| **Geospatial** | ✅ Yes | ❌ No |
| **Lua Scripting** | ✅ Yes | ❌ No |
| **Encryption** | ✅ In-transit + At-rest | ✅ In-transit + At-rest |
| **IAM Auth** | ✅ Yes | ❌ No |
| **Global Datastore** | ✅ Cross-region replication | ❌ No |

### Decision Framework

```
                    ┌──────────────────────────────┐
                    │  Do you need ANY of these?   │
                    │                              │
                    │  • Persistence               │
                    │  • Multi-AZ / HA             │
                    │  • Complex data structures   │
                    │  • Pub/Sub                   │
                    │  • Backup/Restore            │
                    │  • Read Replicas             │
                    │  • Global Datastore          │
                    └───────────┬──────────────────┘
                          YES   │   NO
                    ┌───────────┘   └───────────┐
                    ▼                           ▼
             ┌────────────┐              ┌────────────┐
             │   REDIS    │              │ MEMCACHED  │
             │            │              │            │
             │ Feature-   │              │ Simple,    │
             │ rich, HA,  │              │ volatile,  │
             │ persistent │              │ multi-core │
             └────────────┘              └────────────┘
```

> [!WARNING]
> **Exam trap:** If the question mentions "multi-threaded" or "simplest caching solution" or "no persistence needed," the answer is **Memcached**. Everything else → **Redis**.

---

## Caching Strategies

Understanding caching strategies is critical for the exam. There are two primary patterns:

### Lazy Loading (Cache-Aside)

Data is loaded into the cache **only when requested**. This is the most common pattern.

```
         ┌──────────────────────────────────────────────────┐
         │                  LAZY LOADING                    │
         └──────────────────────────────────────────────────┘

  1. App requests data
  ┌───────┐    "Get user #42"    ┌─────────────┐
  │  App  │─────────────────────►│ ElastiCache │
  └───┬───┘                      └──────┬──────┘
      │                                 │
      │                          ┌──────┴──────┐
      │                     HIT? │             │ MISS?
      │                          ▼             ▼
      │                   2a. Return     2b. Return null
      │                   cached data
      │                                        │
      │                                        ▼
      │   3. App queries DB ◄──────────────────┘
      │──────────────────────►┌──────────┐
      │                       │   RDS    │
      │◄──────────────────────│          │
      │   4. DB returns data  └──────────┘
      │
      │   5. App writes data to cache
      │──────────────────────►┌─────────────┐
      │                       │ ElastiCache │
      └───────────────────────└─────────────┘
```

| Aspect | Detail |
|---|---|
| **Pros** | Only requested data is cached; node failure = cache rebuild on demand |
| **Cons** | Cache miss penalty (3 round trips); data can become **stale** |
| **Mitigation** | Set a **TTL (Time-to-Live)** to expire stale data |

### Write-Through

Data is written to the cache **at the same time** it's written to the database.

```
         ┌──────────────────────────────────────────────────┐
         │                 WRITE-THROUGH                    │
         └──────────────────────────────────────────────────┘

  ┌───────┐                                                 
  │  App  │   1. Write data                                 
  └───┬───┘                                                 
      │                                                     
      ├──────────────────────►┌──────────┐                  
      │   2a. Write to DB     │   RDS    │                  
      │                       └──────────┘                  
      │                                                     
      └──────────────────────►┌─────────────┐               
          2b. Write to Cache  │ ElastiCache │               
                              └─────────────┘               
```

| Aspect | Detail |
|---|---|
| **Pros** | Cache data is **never stale**; reads are always fast |
| **Cons** | Write penalty (2 writes per operation); cache fills with data that may never be read |
| **Mitigation** | Combine with **Lazy Loading** for reads + **TTL** to evict unused data |

### Cache Eviction

When the cache runs out of memory, items are evicted. Three approaches:

1. **TTL (Time-to-Live)** — Items expire after a set duration. Best for most use cases.
2. **LRU (Least Recently Used)** — Evict the item not accessed for the longest time.
3. **Explicit deletion** — Application manually removes stale items.

> [!TIP]
> **Exam Pattern:** "Data in cache is stale" → Add **TTL**. "Reduce database read load" → **Lazy Loading**. "Cache must always be in sync" → **Write-Through**.

---

## ElastiCache for Redis — Deep Dive

### Replication & High Availability

Redis supports **replication groups** (also called **clusters** in the console):

```
                        ┌────────────────────────┐
                        │    Replication Group    │
                        │                        │
                        │  ┌──────────────────┐  │
                        │  │  Primary Node     │  │
                        │  │  (Read + Write)   │  │
                        │  └────────┬─────────┘  │
                        │           │ ASYNC       │
                        │     ┌─────┴─────┐       │
                        │     ▼           ▼       │
                        │ ┌─────────┐ ┌─────────┐│
                        │ │Replica 1│ │Replica 2││
                        │ │(Read)   │ │(Read)   ││
                        │ └─────────┘ └─────────┘│
                        └────────────────────────┘
```

- Up to **5 read replicas** per shard.
- **Multi-AZ** with automatic failover — if the primary fails, a replica is promoted.
- Replication is **asynchronous** (same as [[RDS Read Replicas]]).

### Cluster Mode

| Feature | Cluster Mode Disabled | Cluster Mode Enabled |
|---|---|---|
| **Shards** | 1 shard only | Multiple shards (up to 500) |
| **Scaling** | Vertical only (bigger node) | Horizontal (add shards) |
| **Data capacity** | Limited by one node's memory | Distributed across shards |
| **Write scaling** | Single write endpoint | Write to any shard |
| **Use case** | Small datasets, simpler setup | Large datasets, high throughput |

```
Cluster Mode Enabled — Data is partitioned across shards:

  Shard 1                 Shard 2                 Shard 3
  ┌────────────────┐      ┌────────────────┐      ┌────────────────┐
  │ Primary        │      │ Primary        │      │ Primary        │
  │ Keys: A-H      │      │ Keys: I-P      │      │ Keys: Q-Z      │
  ├────────────────┤      ├────────────────┤      ├────────────────┤
  │ Replica 1      │      │ Replica 1      │      │ Replica 1      │
  │ Replica 2      │      │ Replica 2      │      │ Replica 2      │
  └────────────────┘      └────────────────┘      └────────────────┘
```

### Global Datastore (Cross-Region)

**Global Datastore** provides fully managed, **asynchronous cross-region replication** for Redis.

```
  Region us-east-1 (Primary)          Region eu-west-1 (Secondary)
  ┌──────────────────────────┐        ┌──────────────────────────┐
  │  Primary Cluster         │        │  Secondary Cluster       │
  │  (Read + Write)          │───────►│  (Read Only)             │
  │                          │ ASYNC  │                          │
  │  ┌────────┐ ┌────────┐  │  cross │  ┌────────┐ ┌────────┐  │
  │  │Primary │ │Replica │  │ region │  │Primary │ │Replica │  │
  │  └────────┘ └────────┘  │        │  └────────┘ └────────┘  │
  └──────────────────────────┘        └──────────────────────────┘
```

| Feature | Detail |
|---|---|
| **Replication** | Asynchronous cross-region |
| **Regions** | 1 primary + up to 2 secondary regions |
| **Failover** | Promote secondary to primary (manual, < 1 min) |
| **Use cases** | Low-latency geo-local reads, disaster recovery |
| **Latency** | Typically < 1 second cross-region |

> [!TIP]
> **Exam Pattern:** "Low-latency cache reads in multiple regions" or "cross-region disaster recovery for cache" → **ElastiCache Redis Global Datastore**.

### Redis Persistence

Redis offers two persistence mechanisms (unlike Memcached which has none):

| Mechanism | Description |
|---|---|
| **RDB Snapshots** | Point-in-time snapshots at configured intervals. Lower impact on performance. |
| **AOF (Append-Only File)** | Logs every write operation. More durable but higher performance impact. |

- Backups can be stored in [[S3]].
- Restoring creates a new cluster from the backup.

---

## ElastiCache for Memcached — Deep Dive

Memcached is the **simpler** engine, designed for pure caching with no frills.

```
  ┌─────────────────────────────────────────────────┐
  │           Memcached Cluster                     │
  │                                                 │
  │  ┌─────────┐  ┌─────────┐  ┌─────────┐        │
  │  │ Node 1  │  │ Node 2  │  │ Node 3  │  ...   │
  │  │         │  │         │  │         │        │
  │  └─────────┘  └─────────┘  └─────────┘        │
  │                                                 │
  │  • No replication between nodes                │
  │  • Data is partitioned (sharded) across nodes  │
  │  • If a node fails, data on it is LOST         │
  │  • Auto Discovery finds all nodes              │
  └─────────────────────────────────────────────────┘
```

### Key Characteristics

- **Multi-threaded** — can utilize multiple CPU cores on large nodes.
- **No persistence** — data is purely volatile. Node failure = data loss.
- **No replication** — each node is independent.
- **No failover** — if a node dies, you lose that partition's data.
- **Auto Discovery** — clients automatically detect all nodes in the cluster.
- **Horizontal scaling** — add/remove nodes dynamically.

> [!NOTE]
> Memcached is ideal when you need the absolute simplest, most lightweight caching solution and your cached data is easily re-creatable from the source database.

---

## ElastiCache Security

### Network Security

- Deploy in a **private subnet** within your [[VPC]].
- Control access with **Security Groups** (same pattern as [[Amazon RDS]]).
- **ElastiCache is never publicly accessible** — no public IP, no internet gateway access.

### Encryption

| Type | Redis | Memcached |
|---|---|---|
| **At-Rest** | ✅ KMS (AES-256) | ✅ KMS |
| **In-Transit** | ✅ TLS | ✅ TLS |

> [!WARNING]
> Encryption must be enabled **at cluster creation time**. You cannot enable encryption on an existing cluster — you must create a new encrypted cluster and migrate.

### Authentication

| Method | Redis | Memcached | Notes |
|---|---|---|---|
| **Redis AUTH** | ✅ | — | Password/token-based authentication |
| **IAM Authentication** | ✅ | ❌ | Use [[IAM]] identities to authenticate |
| **RBAC** | ✅ | — | Role-Based Access Control for fine-grained permissions |

> [!TIP]
> **Exam Pattern:** "Authenticate to ElastiCache without passwords" or "centralized cache access management" → **IAM Authentication** (Redis only).

---

## ElastiCache as a Session Store

This is a **heavily tested pattern** on the exam. The architecture makes your application tier **stateless**:

```
                       ┌──────────────────────┐
                       │    ElastiCache       │
                       │   (Session Store)     │
                       └───────────┬──────────┘
                                   │
              ┌────────────────────┼────────────────────┐
              │                    │                    │
              ▼                    ▼                    ▼
       ┌────────────┐      ┌────────────┐      ┌────────────┐
       │   EC2 #1   │      │   EC2 #2   │      │   EC2 #3   │
       │   (App)    │      │   (App)    │      │   (App)    │
       └────────────┘      └────────────┘      └────────────┘
              ▲                    ▲                    ▲
              └────────────────────┼────────────────────┘
                                   │
                       ┌───────────┴──────────┐
                       │  [[Elastic Load      │
                       │   Balancer (ELB)|ELB]]│
                       └──────────────────────┘
                                   ▲
                               Users

  • User logs in via any EC2 instance
  • Session is stored in ElastiCache
  • Next request may go to ANY instance
  • Any instance retrieves session from ElastiCache
  • No sticky sessions needed! ✅
```

### Without ElastiCache (Sticky Sessions)

```
  User ──► ELB ──► EC2 #1 (session stored locally)
                   ▲
                   │ Must always go to the same instance!
                   │ If EC2 #1 dies, session is LOST.
```

### With ElastiCache (Stateless)

```
  User ──► ELB ──► ANY EC2 instance ──► ElastiCache (shared session)
                                         ↑ Session survives instance failure
```

> [!IMPORTANT]
> **Exam keywords:** "Stateless application," "remove sticky sessions," "session data shared across instances," "user sessions persist even if instance terminates" → **ElastiCache session store**.

---

## ElastiCache vs Other AWS Caching Solutions

| Solution | Type | Use Case |
|---|---|---|
| **ElastiCache** | In-memory key-value cache | DB query caching, session store, real-time data |
| **DAX (DynamoDB Accelerator)** | In-memory cache for [[DynamoDB]] | DynamoDB-specific caching only |
| **CloudFront** | CDN (edge cache) | Static content, media, API responses at edge locations |
| **API Gateway Cache** | API response cache | Cache REST API responses |

> [!WARNING]
> **Exam trap:** DAX is **only** for DynamoDB. If the question mentions caching for [[Amazon RDS]] or [[Amazon Aurora]], the answer is **ElastiCache**, never DAX.

---

## ElastiCache Serverless

AWS offers **ElastiCache Serverless** for both Redis and Memcached:

- **No capacity planning** — AWS automatically scales compute and memory.
- **Pay-per-use** — charged based on data stored and ElastiCache Processing Units (ECPUs).
- **High availability** — automatically replicates across multiple [[Availability Zones (AZ)|AZs]].
- Best for **unpredictable workloads** or when you don't want to manage node types and cluster sizing.

---

## Exam Cheat Sheet

> [!TIP]
> **Trigger phrases → ElastiCache:**
> - "In-memory caching" or "sub-millisecond latency" → **ElastiCache**
> - "Reduce read load on database" → **ElastiCache** (Lazy Loading)
> - "Stateless application" or "remove sticky sessions" → **ElastiCache session store**
> - "Real-time leaderboard" or "sorted rankings" → **ElastiCache Redis** (Sorted Sets)
> - "Multi-threaded cache" or "simplest cache" → **ElastiCache Memcached**
> - "Cache with persistence / HA / Multi-AZ" → **ElastiCache Redis**
> - "Cross-region cache replication" → **Redis Global Datastore**
> - "Stale data in cache" → Add **TTL** or use **Write-Through**
> - "Cache for DynamoDB" → **DAX** (not ElastiCache)
> - "Pub/Sub messaging from cache" → **Redis**
>
> **Key facts:**
> - Redis: up to **5 replicas** per shard, Multi-AZ, persistent, single-threaded
> - Memcached: no replication, no persistence, multi-threaded, auto discovery
> - Encryption must be set **at creation time** (cannot enable later)
> - ElastiCache is **never publicly accessible** — VPC only
> - Lazy Loading: 3 round trips on miss, risk of stale data
> - Write-Through: 2 writes per operation, never stale
> - Global Datastore: 1 primary + up to 2 secondary regions
