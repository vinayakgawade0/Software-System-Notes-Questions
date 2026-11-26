# Caching - Chapter 3: Distributed Caching Strategies & Redis Architecture

## Distributed Caching Strategies and Architectural Trade-offs

### Core Fundamentals of Distributed Caching

In distributed software systems, persistent relational and document databases rely on physical block storage (SSDs, NVMe drives, or network-attached SAN/EBS volumes). Even with aggressive operating system page buffering, disk-bound I/O, relational indexing traversals, and query parsing incur millisecond-range latency.

Caching is the architectural practice of storing frequently requested data in volatile, high-speed Random Access Memory (RAM). By serving reads directly from RAM in microseconds, caching:
1. Drastically reduces end-to-end request latency for client applications.
2. Shields backend databases from connection starvation, lock contention, and CPU exhaustion.
3. Provides high cost efficiency by substituting expensive database scaling with lightweight in-memory nodes.

---

### Cache-Aside (Lazy Loading) Architecture

Cache-Aside (also known as Lazy Loading) is the most ubiquitous caching topology in distributed architectures. In this pattern, the application code is entirely responsible for coordinating read and write workflows between the cache and the primary datastore.

```
Read Workflow (Cache-Aside):
[ Client ] ---> [ Application ] ---> (1) GET key ---> [ Cache Tier ]
                     |                                       |
                     +---------------------------------------+
                     |
        +------------+------------+
        | (Cache Hit)             | (Cache Miss)
        v                         v
   Return Data              (2) SELECT from DB ---> [ Primary Database ]
                                  |
                            (3) SET key (populate)
                                  v
                            [ Cache Tier ]
                                  |
                            Return Data
```

#### Execution Lifecycle

1. **Read Request (Cache Hit):** The application queries the distributed cache for the given key. If found, the data is returned immediately without contacting the database.
2. **Read Request (Cache Miss):**
   * The cache returns `null`.
   * The application queries the primary database (`SELECT * FROM users WHERE id = ?`).
   * The application persists a copy of the retrieved record into the cache with an optional Time-To-Live (TTL).
   * The data is returned to the client.
3. **Write / Mutation Request:**
   * The application executes an update against the primary database (`UPDATE users SET ...`).
   * The application explicitly evicts or updates the corresponding key in the cache (`DEL user:101`). Evicting the key (invalidation) is preferred over updating to avoid race conditions.

#### Architectural Trade-offs

* **Resilience:** If the cache tier crashes, becomes partitioned, or runs out of memory, the system experiences a graceful performance degradation. The application continues functioning by falling back to direct database queries.
* **Staleness Risk:** If updates to the primary database fail to evict the corresponding cache key (e.g., due to an application crash between DB commit and cache deletion), subsequent read requests will serve stale data until the key's TTL expires.

---

### Read-Through Caching Mechanics

In a Read-Through architecture, the application treats the caching tier as its primary data interface. The application code never directly queries the database during read operations; instead, the cache library or middleware transparently loads data from the database upon a cache miss.

```
Read-Through Flow:
[ Application ] ---> (1) GET key ---> [ Cache Provider ]
                                             |
                                  Is data present?
                                             |
                                  +----------+----------+
                                  | (Hit)               | (Miss)
                                  v                     v
                             Return Value        (2) Transparently fetch from DB
                                                        |
                                                 [ Primary DB ]
                                                        |
                                                 (3) Store in Cache & Return
```

#### Key Characteristics

* **Simplified Application Logic:** The application code is cleaner because it does not maintain fallback boilerplate or manual cache population logic.
* **Framework Coupling:** Requires caching infrastructure or middleware that natively understands the underlying database schema and connection drivers.

---

### Write-Through Caching Architecture

Write-Through caching coordinates writes such that the application writes directly to the caching tier. The cache immediately and **synchronously** writes the same data down to the primary database before acknowledging success back to the caller.

```
Write Workflow (Write-Through):
[ Application ] ---> (1) Write Data ---> [ Cache Tier ]
                                                |
                                      (2) Synchronous Write
                                                v
                                      [ Primary Database ]
                                                |
                                      (3) Confirm Write Success
                                                v
[ Application ] <--- (4) Write Complete --------+
```

#### Architectural Trade-offs

* **Absolute Consistency:** Guarantees that the cache and the primary database remain perfectly synchronized. Subsequent read requests immediately hit warm, consistent cache entries with zero risk of cache misses or stale reads.
* **Write Latency Penalty:** Every write operation incurs the cumulative latency of two sequential network hops:
  $$\text{Write Latency} = \text{Latency}_{\text{App} \to \text{Cache}} + \text{Latency}_{\text{Cache} \to \text{DB}}$$
* **Cache Churn:** Frequently writing transient or rarely read data pollutes the memory pool, evicting hotter keys.

---

### Write-Behind (Write-Back) Asynchronous Caching

Write-Behind caching is designed for ultra-high-throughput write workloads (e.g., IoT telemetry ingestion, real-time gaming state, live clickstream analytics).

In this pattern, the application writes data exclusively to the in-memory cache tier, which immediately acknowledges success to the client in sub-millisecond time. The cache tier subsequently flushes the accumulated mutations to the primary database **asynchronously** in batches or via background workers.

```
Write Workflow (Write-Behind):
[ Application ] ---> (1) Write Data ---> [ Cache Tier ] ---> (2) Immediate "Success!" (Sub-ms)
                                                |
                                     (Background Thread / Batch Queue)
                                                v
                                      (3) Async Batch Flush (e.g., every 5s)
                                                v
                                      [ Primary Database ]
```

#### Architectural Trade-offs

* **Extreme Write Throughput:** Decouples high-velocity write traffic from disk I/O bottlenecks. Disk writes are consolidated into sequential, bulk database operations, slashing write amplification.
* **Catastrophic Data Loss Vulnerability:** Because data resides exclusively in volatile RAM before the asynchronous batch flush occurs, an unexpected server crash, power loss, or fatal hardware event results in permanent data loss.

---

### Architectural Trade-off Matrix of Caching Patterns

| Caching Pattern | Read Latency | Write Latency | Data Consistency Guarantee | Failure Impact | Primary Industry Use Cases |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **Cache-Aside (Lazy Loading)** | Microseconds on hit; Milliseconds on miss. | Fast (writes hit DB directly; invalidates cache). | **Eventual Consistency.** Potential for stale reads if invalidation is missed. | **Graceful Degradation.** If cache crashes, DB serves all traffic (higher DB load). | General e-commerce, user profile management, news feeds, content catalogs. |
| **Read-Through** | Microseconds on hit; Milliseconds on miss. | Unchanged (delegated to separate write flow). | **Eventual / Tunable.** Transparently managed by cache framework. | Cache node failure halts reads unless fallback routing is configured. | Microservice data abstraction layers, standard enterprise ORMs with L2 cache. |
| **Write-Through** | Microseconds (data is always pre-warmed). | **High.** Bound by synchronous write to both cache and DB. | **Strong Consistency.** Cache and database are always identical. | Write failure in either tier aborts the entire transaction. | Financial ledgers, stock inventory reservation, authorization token stores. |
| **Write-Behind (Write-Back)** | Microseconds. | **Ultra-Low (Sub-ms).** Writes confirm instantly in RAM. | **Eventual Consistency.** Significant lag between RAM and disk. | **Critical Risk.** Power loss or node crash before flush causes unrecoverable data loss. | High-frequency trading logs, gaming leaderboards, IoT sensor telemetry. |

---

## Redis Architecture and In-Memory System Internals

### Redis vs In-Process Application Memory

When implementing caching, software engineers frequently evaluate storing data inside the application process memory (e.g., Java `ConcurrentHashMap`, Guava Cache, or Caffeine) versus deploying a dedicated, distributed in-memory datastore like **Redis (Remote Dictionary Server)**.

```
Scenario A: In-Process Application Memory (JVM Heap)
+-----------------------+     +-----------------------+     +-----------------------+
| Application Node 1    |     | Application Node 2    |     | Application Node 3    |
| [JVM Heap / HashMap]  |     | [JVM Heap / HashMap]  |     | [JVM Heap / HashMap]  |
+-----------------------+     +-----------------------+     +-----------------------+
           |                             |                             |
           +-----------------------------+-----------------------------+
                                         |
            * PROBLEM: Inconsistent state across nodes, memory lost on restart,
                       GC pauses trigger stop-the-world latency spikes.

Scenario B: Centralized Distributed Redis Cache
+-----------------------+     +-----------------------+     +-----------------------+
| Application Node 1    |     | Application Node 2    |     | Application Node 3    |
+-----------------------+     +-----------------------+     +-----------------------+
           \                             |                             /
            \                            |                            /
             v                           v                           v
                      +-------------------------------------+
                      |         Redis Cluster (RAM)         |
                      |  - Shared, consistent state         |
                      |  - Survives app redeployments       |
                      |  - Zero JVM Garbage Collection      |
                      +-------------------------------------+
```

#### Detailed Comparison

| Feature Dimension | In-Process Memory (JVM Heap / Caffeine) | Distributed Redis Cluster |
| :--- | :--- | :--- |
| **Data Consistency Across Nodes** | **Severely Inconsistent.** In a horizontally scaled cluster of 10 nodes, each node maintains its own isolated cache. Node A may serve updated state while Node B serves stale data. | **Universally Consistent.** A centralized distributed store guarantees all application instances read the exact same state over the network. |
| **Process Lifecycle & Durability** | **Volatile.** Redeploying, restarting, or crashing an application node instantly wipes the entire cache, triggering database request stampedes. | **Persistent & Decoupled.** Cache state survives application restarts, rolling deployments, and container restarts without interruption. |
| **Memory Footprint & GC Pressure** | Consumes application heap space. Massive object caches trigger JVM Garbage Collector thrashing and long "Stop-the-World" pauses. | Completely off-heap on dedicated infrastructure. Application JVM memory remains lean with zero GC overhead from cached objects. |
| **Access Latency** | Nanoseconds (in-memory pointer dereferencing within CPU cache and RAM). | Microseconds to 1–2 milliseconds (incurs network TCP serialization and round-trip). |

---

### Pillar 1: Advanced Native Data Structures and In-Memory Manipulation

Unlike primitive key-value stores (such as Memcached) which treat stored values strictly as opaque binary blobs or strings, Redis understands and manipulates rich, native data structures directly in memory.

```
                                +-----------------------------+
                                |    Redis Data Structures    |
                                +-----------------------------+
                                /      |       |       \                                     v       v       v        v      v
                           Strings   Hashes  Lists    Sets   Sorted Sets
                                                            (ZSets)
```

#### Core Data Structures and Operational Benefits

1. **Strings (`SET`, `GET`, `INCR`):**
   * Binary-safe sequences up to 512MB in size.
   * Supports atomic increments (`INCR`, `INCRBY`), making it the premier choice for distributed counters, page view metrics, and atomic token bucket rate limiting.
2. **Hashes (`HSET`, `HGET`, `HINCRBY`):**
   * Key-value field maps representing objects (e.g., `user:101 -> { name: "Alice", age: 30, score: 95 }`).
   * **In-Place Field Manipulation:** Enables updating or retrieving individual fields without downloading, deserializing, modifying, and uploading the entire JSON blob.
3. **Lists (`LPUSH`, `RPOP`, `BRPOP`):**
   * Linked lists / quicklists preserving insertion order.
   * Operates as ultra-fast message queues, background job queues, and recent timeline feeds (`LPUSH` + `LTRIM`).
4. **Sets (`SADD`, `SMEMBERS`, `SINTER`):**
   * Unordered collections of unique strings.
   * Enables $O(1)$ uniqueness checks, tag indexing, and instantaneous set mathematics (intersections for mutual friends: `SINTER user:1:friends user:2:friends`).
5. **Sorted Sets / ZSets (`ZADD`, `ZRANGEBYSCORE`, `ZREVRANK`):**
   * Elements mapped to floating-point scores, indexed internally via **SkipLists** and hash tables.
   * Delivers logarithmic $O(\log N)$ insertion, removal, and range queries. Powers real-time gaming leaderboards, sliding-window rate limiters, and delayed task scheduling.
6. **Bitmaps & HyperLogLog:**
   * **Bitmaps:** Space-efficient bit arrays for user activity tracking (100 million daily active users tracked in ~12MB of RAM).
   * **HyperLogLog (`PFADD`, `PFCOUNT`):** Probabilistic cardinality estimation tracking unique website visitors with less than 0.81% standard error using exactly 12KB of memory.

---

### Pillar 2: Single-Threaded Event Loop and Atomic Operations

The architectural cornerstone of Redis performance is its **single-threaded execution model** for processing client commands.

```
+-------------------------------------------------------------------------------+
|                             Client Connections Tier                           |
|       Client 1                     Client 2                     Client 3      |
+-------------------------------------------------------------------------------+
           \                            |                            /
            \                           |                           /
             v                          v                          v
+-------------------------------------------------------------------------------+
|                       I/O Multiplexer (epoll / kqueue)                        |
|  - Monitors thousands of non-blocking TCP sockets simultaneously              |
|  - Converts socket readiness events into an ordered Event Queue               |
+-------------------------------------------------------------------------------+
                                        |
                                        v
+-------------------------------------------------------------------------------+
|                   Single-Threaded Command Execution Engine                    |
|  - Executes one command at a time to completion                               |
|  - Zero thread context switches                                               |
|  - Zero mutexes, spinlocks, or read-write locks                               |
|  - Guaranteed atomic execution of commands & Lua scripts                      |
+-------------------------------------------------------------------------------+
```

#### Why Single-Threaded is Faster

1. **Elimination of Lock Contention:** In multi-threaded in-memory engines, threads spend the majority of their CPU cycles contending for locks (mutexes, spinlocks) to guard shared data structures. Redis has zero lock contention.
2. **Zero Context Switching:** CPU cores execute instructions contiguously without the overhead of saving and restoring thread registers or flushing CPU L1/L2 caches.
3. **Non-Blocking I/O Multiplexing:** Redis uses kernel-level event demultiplexers (`epoll` on Linux, `kqueue` on macOS) to handle tens of thousands of active socket connections over a single thread.
4. **Inherent Atomicity:** Because commands execute serially without interruption, multi-step primitives (e.g., `INCR`, `SETNX`, or complex Lua scripts executed via `EVAL`) are mathematically atomic. Distributed locks and rate limiters require zero external synchronization.

*(Note: Starting in Redis 6.0+, multi-threading was introduced exclusively for network I/O serialization and socket socket reads/writes; the core execution engine remains strictly single-threaded).*

---

### Pillar 3: Persistence Mechanisms: RDB Snapshots vs Append-Only File (AOF)

While Redis operates primarily in volatile RAM to deliver microsecond responses, it provides robust disk persistence options to survive unexpected crashes and host reboots.

```
+-------------------------------------------------------------------------------+
| RDB (Redis Database Snapshots)                                                |
| - Forks child process using OS Copy-On-Write (COW).                           |
| - Writes point-in-time binary snapshot (dump.rdb) to disk.                    |
| - Compact, fast startup; potential loss of last few minutes of data.          |
+-------------------------------------------------------------------------------+

+-------------------------------------------------------------------------------+
| AOF (Append-Only File)                                                        |
| - Logs every write command sequentially to append-only log (appendonly.aof).   |
| - Sync options: fsync everysec (standard) or always.                          |
| - Maximum durability; larger disk footprint and slower server recovery.      |
+-------------------------------------------------------------------------------+
```

#### Comparison of Persistence Strategies

| Operational Feature | RDB (Redis Database Snapshots) | AOF (Append-Only File) |
| :--- | :--- | :--- |
| **Mechanics** | Creates compact, point-in-time binary dumps of entire memory space at scheduled intervals (e.g., every 5 minutes if > 100 keys changed). | Sequentially appends every single mutating write command (`SET`, `HSET`, `DEL`) to a log file on disk. |
| **Durability & Data Loss Window** | **Higher Data Loss Risk.** If Redis crashes between scheduled snapshots, all data written since the last snapshot is permanently lost. | **Near-Zero Data Loss.** With `appendfsync everysec`, at most 1 second of write operations can be lost during a sudden failure. |
| **Recovery / Restart Speed** | **Extremely Fast.** Redis reads the compact binary RDB image directly into memory in seconds. | **Slower.** Redis must replay every command in the AOF log sequentially to reconstruct state (mitigated by background AOF rewrite). |
| **Performance Overhead** | Executes via `fork()`. During fork, Linux Copy-On-Write (COW) memory duplication can cause transient latency spikes under heavy write workloads. | Continuous disk I/O write operations. If disk write queues stall, background fsync can block the main execution thread. |
| **Recommended Production Practice** | Ideal for disaster recovery backups and read replica initialization. | Ideal for high-durability transactional caching layers. **Best Practice:** Combine both (Hybrid RDB + AOF). |

---

### Redis Eviction Policies, TTL Mechanics, and High-Availability Clustering

#### Eviction Policies (When Memory is Full)

When Redis memory consumption hits the configured ceiling (`maxmemory`), it evicts keys based on the configured `maxmemory-policy`:

| Eviction Policy | Mechanics & Architectural Behavior |
| :--- | :--- |
| **`noeviction`** | Redis returns errors on write commands (`OOM command not allowed`) when memory limit is reached. Reads continue functioning. Standard for primary databases. |
| **`allkeys-lru`** | Evicts the **Least Recently Used** keys across the entire dataset. Premier choice for general-purpose caching workloads following power-law distributions. |
| **`volatile-lru`** | Evicts Least Recently Used keys strictly among keys with an explicit TTL expiration set. |
| **`allkeys-lfu`** | Evicts the **Least Frequently Used** keys across the dataset, preserving frequently accessed keys regardless of how recently they were read. |
| **`volatile-ttl`** | Evicts keys with the shortest remaining Time-To-Live. |

#### Distributed Clustering & Hash Slot Architecture

To scale beyond a single node's physical RAM capacity (e.g., scaling to terabytes of cache memory), **Redis Cluster** provides native horizontal sharding:

```
Total Address Space: Exactly 16,384 Hash Slots [0 - 16383]

+--------------------+      +--------------------+      +--------------------+
| Master Node 1      |      | Master Node 2      |      | Master Node 3      |
| Slots 0 - 5460     |      | Slots 5461 - 10922 |      | Slots 10923 - 16383|
+--------------------+      +--------------------+      +--------------------+
          |                           |                           |
    (Replication)               (Replication)               (Replication)
          v                           v                           v
+--------------------+      +--------------------+      +--------------------+
| Replica Node 1     |      | Replica Node 2     |      | Replica Node 3     |
+--------------------+      +--------------------+      +--------------------+
```

* **CRC16 Hash Slot Algorithm:**
  $$\text{Slot} = \text{CRC16}(\text{key}) \pmod{16384}$$
* **Deterministic Routing:** Client libraries (e.g., Jedis, Lettuce) cache the hash slot mappings. When issuing a command for `user:402`, the client hashes the key, calculates the exact hash slot, and dispatches the TCP request directly to the master node owning that slot with zero intermediate proxy hops.
