# Caching - Questions - Chapter 3: Distributed Caching Strategies & Redis Architecture

## Distributed Caching Strategies and Architectural Trade-offs

### Core Fundamentals of Distributed Caching

### Cache-Aside (Lazy Loading) Architecture

### Read-Through Caching Mechanics

### Write-Through Caching Architecture

### Write-Behind (Write-Back) Asynchronous Caching

### Architectural Trade-off Matrix of Caching Patterns

#### Can you explain the primary caching strategies used in distributed systems, and the architectural trade-offs of each?

> **Direct Answer:** The primary distributed caching strategies are Cache-Aside (Lazy Loading), Write-Through, and Write-Behind (Write-Back). In Cache-Aside, the application coordinates reads by querying the cache first and falling back to the primary database on a cache miss, providing high system resilience if the cache crashes at the expense of potential cache staleness. Write-Through guarantees absolute cache-database consistency by having the cache synchronously write updates to the database before acknowledging the caller, incurring write latency penalties. Write-Behind acknowledges writes immediately to the in-memory cache and flushes updates to disk asynchronously in batches, achieving maximum write throughput but risking catastrophic data loss during ungraceful server power outages.

> **Key Nuance / Trade-off:** Cache invalidation remains one of computer science's hardest problems; while Cache-Aside is the industry standard due to its fault tolerance, concurrent writes and reads can trigger race conditions where stale data is rewritten to the cache. Architects mitigate this by pairing Cache-Aside with short Time-To-Live (TTL) horizons and Change-Data-Capture (CDC) event streaming via tools like Debezium to invalidate cache keys asynchronously upon database commit.

> **Reference Concept:** [Architectural Trade-off Matrix of Caching Patterns](./Notes%20-%20Caching%20-%203.%20Distributed%20Caching%20Strategies%20%26%20Redis%20Architecture.md#architectural-trade-off-matrix-of-caching-patterns)

---

## Redis Architecture and In-Memory System Internals

### Redis vs In-Process Application Memory

#### What exactly is Redis, and why is it preferred over simply storing data in application memory?

> **Direct Answer:** Redis (Remote Dictionary Server) is an open-source, networked, in-memory key-value data structure store utilized as a distributed cache, message broker, and real-time database. It is overwhelmingly preferred over in-process application memory (such as Java `HashMap`, Guava, or Caffeine) because application memory is lost upon JVM restart and creates isolated, inconsistent data silos when applications scale horizontally across multiple instances behind a load balancer. Redis moves memory to an external, centralized tier that provides three foundational architectural pillars: advanced native data structures (Strings, Hashes, Lists, Sets, Sorted Sets), a single-threaded event loop guaranteeing atomic operations without locks or race conditions, and disk persistence options (RDB and AOF) to recover data after restarts.

> **Key Nuance / Trade-off:** While Redis eliminates data inconsistency across horizontally scaled microservices, it replaces local in-process memory access (nanoseconds, zero serialization) with networked TCP socket hops (typically 0.5 to 2 milliseconds). For ultra-low latency requirements, senior architects often implement a two-tier caching topology: a small in-process L1 cache (Caffeine) with aggressive TTLs backed by a shared distributed L2 cache (Redis Cluster).

> **Reference Concept:** [Redis vs In-Process Application Memory](./Notes%20-%20Caching%20-%203.%20Distributed%20Caching%20Strategies%20%26%20Redis%20Architecture.md#redis-vs-in-process-application-memory)

---

### Pillar 1: Advanced Native Data Structures and In-Memory Manipulation

### Pillar 2: Single-Threaded Event Loop and Atomic Operations

### Pillar 3: Persistence Mechanisms: RDB Snapshots vs Append-Only File (AOF)

### Redis Eviction Policies, TTL Mechanics, and High-Availability Clustering
