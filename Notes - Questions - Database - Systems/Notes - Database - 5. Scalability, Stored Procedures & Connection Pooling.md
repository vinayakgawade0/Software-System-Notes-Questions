# Database Systems - Chapter 5: Scalability, Stored Procedures & Connection Pooling

## Stored Procedures and Database Programmability

### Stored Procedure Architecture, Execution, and Implementation

A Stored Procedure is a compiled batch of SQL statements combined with procedural programming logic (control flow, variable declarations, loops, condition branching, and exception handling) stored directly inside the database catalog.

```
       [Client Application]
                 |
                 | 1. "CALL transfer_funds(10, 20, 500.00)" (Single Network Roundtrip)
                 v
   +-----------------------------------------------------------+
   |                  Database Engine                          |
   |                                                           |
   |  CREATE OR REPLACE PROCEDURE transfer_funds(...) AS $$   |
   |  BEGIN                                                    |
   |    -- Verify balance, lock rows, perform ACID debit/credit |
   |  END; $$ LANGUAGE plpgsql;                               |
   +-----------------------------------------------------------+
```

#### PostgreSQL PL/pgSQL Code Example
A classic financial fund transfer executing atomically within the database engine:

```sql
CREATE OR REPLACE PROCEDURE transfer_funds(
    sender_id BIGINT,
    receiver_id BIGINT,
    amount NUMERIC
)
LANGUAGE plpgsql
AS $$
DECLARE
    sender_balance NUMERIC;
BEGIN
    -- 1. Check sender's current balance with row-level lock
    SELECT balance INTO sender_balance 
    FROM accounts 
    WHERE id = sender_id 
    FOR UPDATE;

    IF sender_balance < amount THEN
        RAISE EXCEPTION 'Insufficient funds: available %, requested %', sender_balance, amount;
    END IF;

    -- 2. Deduct from sender
    UPDATE accounts 
    SET balance = balance - amount 
    WHERE id = sender_id;

    -- 3. Credit receiver
    UPDATE accounts 
    SET balance = balance + amount 
    WHERE id = receiver_id;

    -- Implicit transaction commit upon procedure completion
END;
$$;
```

### Architectural Evolution: Stored Procedures vs Application-Layer Business Logic

Enterprise software architecture has undergone a fundamental paradigm shift over the last two decades regarding where business logic should reside:

| Architectural Dimension | Historical Paradigm (1990s - 2000s): "Fat Database" | Modern Paradigm: "Smart Application, Dumb Database" |
| :--- | :--- | :--- |
| **Logic Location** | Stored Procedures inside Oracle PL/SQL, Microsoft T-SQL | Application microservices (Java, Spring Boot, Go, Node.js) |
| **Architectural Philosophy** | "Bring the code to the data." Database handles validation, calculation, and orchestration. | Database acts strictly as a persistent storage layer; application code orchestrates domain rules. |
| **Scalability Bottleneck** | **Database CPU Exhaustion:** Scaling database compute requires expensive vertical scaling or proprietary clustering. | **Horizontally Elastic:** Application stateless pods scale elastically on Kubernetes in seconds. |
| **Version Control & CI/CD** | Difficult to test, version-control, lint, and deploy zero-downtime schema changes. | Standard Git workflows, automated unit/integration testing, Docker containers, and CI/CD pipelines. |
| **Vendor Portability** | Extreme **Vendor Lock-in:** PL/SQL does not run on PostgreSQL or MySQL without complete rewrites. | High Portability: ORMs and ANSI SQL allow switching or polyglot persistence without rewriting core business logic. |

### Modern Batch Processing Use Cases and Trade-offs

While modern microservices avoid stored procedures for standard CRUD operations and transactional workflows, stored procedures remain valuable for specific heavy workloads:

* **Massive Batch Data Processing:** Performing interest calculations, tax recalculations, or billing aggregations over 50 million rows. Pulling 50 million rows over the network to the Java layer would saturate network bandwidth, cause Java Virtual Machine Out-Of-Memory (`OutOfMemoryError`) crashes, and incur massive GC pauses. In this scenario, sending procedural logic into the database engine is vastly superior.
* **Senior Architectural Takeaway:** Keep business rules and domain logic in the application tier (Java/Spring Boot) backed by `@Transactional` services, and reserve stored procedures strictly for heavy, data-local batch transformations.

---

## Database Scalability and High-Availability Strategies

### Vertical Scaling vs Horizontal Scaling Architectures

When database traffic or storage outgrows a single node, organizations evaluate two scaling vectors:

```
    [Vertical: Scale Up]                      [Horizontal: Scale Out]
       +-------------+                     +-------+   +-------+   +-------+
       |   Bigger    |                     | Node1 |   | Node2 |   | Node3 |
       | Server RAM/ |                     | ShardA|   | ShardB|   | ShardC|
       |  CPU/NVMe   |                     +-------+   +-------+   +-------+
       +-------------+
```

* **Vertical Scaling (Scale Up):** Adding more CPU cores, RAM, and faster NVMe drives to a single database server.
  * *Pros:* Simple; requires zero application code changes or distributed transaction complexity.
  * *Cons:* Hits physical hardware and cost ceilings; leaves a Single Point of Failure (SPOF).
* **Horizontal Scaling (Scale Out):** Distributing storage and queries across multiple physical nodes via Replication or Sharding.
  * *Pros:* Near-infinite scalability and high fault tolerance.
  * *Cons:* Requires complex operational tooling, distributed consensus, and careful data partitioning.

### Primary-Replica Replication, Read Scalability, and Replication Lag

In read-heavy applications (e.g., e-commerce, content platforms where reads exceed writes 10:1), architects deploy **Primary-Replica (Master-Slave) Replication**:

* **Write Path:** All `INSERT`, `UPDATE`, and `DELETE` statements target the single **Primary (Leader)** node. The Primary streams WAL / binlog changes to downstream nodes.
* **Read Path:** `SELECT` queries are load-balanced across multiple read-only **Replica (Follower)** nodes.
* **Asynchronous vs Synchronous Replication:**
  * *Asynchronous:* The Primary commits writes to local storage and confirms immediately to the client without waiting for replicas. High write throughput, but replicas experience **Replication Lag**.
  * *The "Read-Your-Own-Writes" Consistency Hazard:* If a user updates their profile and immediately refreshes the page, reading from a lagging replica will show stale, pre-update data.
  * *Solution:* Route critical read queries immediately following writes to the Primary node for a short grace window, or use synchronous replication.

### Database Sharding and Horizontal Partitioning Mechanics

When table sizes exceed terabytes and write throughput saturates the Primary node's disk I/O, vertical scaling and read replicas are insufficient. The database must be **Sharded**:

* **Sharding Defined:** Horizontally splitting rows across independent, isolated database instances based on a **Shard Key** (e.g., `user_id` or `tenant_id`).
* **Partitioning Strategies:**
  * **Hash-Based Sharding:** $\text{Target Node} = \text{hash}(\text{shard\_key}) \pmod N$. Guarantees uniform data distribution, but adding nodes requires re-sharding (mitigated by Consistent Hashing).
  * **Range-Based Sharding:** Partitions rows by ranges (e.g., IDs 1–10M on Shard 1, 10M–20M on Shard 2). Susceptible to hotspots if writes cluster on recent ranges.
* **The Sharding Trade-offs:**
  * Cross-shard `JOIN` operations become computationally prohibitive or impossible.
  * Multi-shard transactions require complex, high-latency Distributed Two-Phase Commit (2PC) or Saga patterns.

### Distributed Caching Layers and Query Offloading

Before implementing complex sharding, systems deploy an in-memory distributed cache (e.g., Redis or Memcached) to intercept read queries:

* **Cache-Aside (Lazy Loading):** The application checks Redis. On cache hit, it returns immediately; on cache miss, it reads from PostgreSQL, populates Redis with a Time-To-Live (TTL), and returns.
* **Impact:** Offloads up to 80–90% of repeated read traffic from the database engine, preserving database buffer pool RAM and CPU for writes.

## Table Partitioning vs Database Sharding

### Table Partitioning Single-Instance Scaling
Table partitioning splits a single large logical table into smaller physical segments (partitions) **within the same database server instance**. The database engine transparently routes queries to the relevant partition(s) via **partition pruning** while preserving a single schema and ACID transaction scope.

- **Common Partitioning Strategies:**
  - **Range Partitioning:** Splits rows by continuous ranges such as timestamps (`created_at` monthly/yearly partitions) or numeric IDs—ideal for time-series logs, orders, and audit tables.
  - **List Partitioning:** Splits rows by discrete categorical values such as `region` (`'IN'`, `'US'`, `'EU'`) or `tenant_tier`.
  - **Hash Partitioning:** Distributes rows evenly across a fixed number of partitions using `hash(partition_key) % N` when data lacks a natural range or list skew.
- **Scaling Problems Addressed:**
  - Shrinks B-Tree index depth and keeps active ("hot") partitions resident in RAM/buffer cache.
  - Speeds up maintenance operations: dropping or archiving old historical data becomes an instant `DROP TABLE` / `DETACH PARTITION` metadata operation instead of an expensive row-by-row `DELETE` that causes table bloat and vacuum overhead.
- **Limitation:** Remains bounded by the vertical hardware limits (CPU, RAM, disk IOPS, network bandwidth) of a single primary database server.

### Database Sharding Horizontal Multi-Node Scaling
Database sharding partitions data horizontally **across multiple independent database nodes (servers)**, where each shard holds a distinct subset of rows with an identical schema (shared-nothing architecture).

- **Shard Key & Routing:**
  - Every query ideally includes the **shard key** (e.g., `tenant_id`, `customer_id`, `account_id`) so a routing layer (application router, Vitess, Citus, or proxy) directs traffic to the exact target shard.
  - **Range-Based Sharding:** Maps contiguous key ranges to specific nodes (risks hotspotting on newest ranges).
  - **Hash / Consistent-Hash Sharding:** Applies a hash function to the shard key to distribute write load uniformly across nodes and minimize data movement when adding shards.
  - **Directory-Based Sharding:** Uses a lookup service mapping each tenant/entity ID to its physical shard (flexible rebalancing, but the lookup store becomes a critical dependency).
- **Scaling Problems Addressed:**
  - Scales **write throughput**, total storage capacity, and compute beyond what any single machine can vertically sustain.
  - Provides fault and noisy-neighbor isolation—degradation on one shard impacts only the subset of tenants/users mapped to it.
- **Operational Trade-Offs:**
  - Cross-shard queries (scatter-gather joins, global `ORDER BY` / aggregations) incur high network latency.
  - Distributed transactions across shards require complex coordination (Saga / 2PC), and resharding/rebalancing data requires careful dual-write or CDC migration pipelines.

### Architectural Comparison and Selection Criteria
| Dimension | Table Partitioning | Database Sharding |
| :--- | :--- | :--- |
| **Physical Topology** | Single database server/cluster | Multiple independent database nodes |
| **Primary Bottleneck Solved** | Large table/index bloat, slow scans, bulk data retention/archival | Single-node write IOPS ceiling, massive total storage, multi-tenant isolation |
| **Cross-Partition / Shard Joins** | Handled natively by the DB query planner | Expensive scatter-gather across network boundaries |
| **ACID Transactions** | Standard local ACID transactions | Local ACID within a single shard; eventual consistency across shards |
| **Operational Complexity** | Low (native DDL support in PostgreSQL, MySQL, Oracle) | High (shard routing, resharding, cross-shard schema migrations, backups) |

- **Decision Rule:**
  1. Start with **indexing, query tuning, connection pooling, read replicas, and caching**.
  2. Introduce **Table Partitioning** when a single table grows to tens/hundreds of millions of rows, index sizes exceed memory, or time-based archival is required, while single-node write capacity is still sufficient.
  3. Adopt **Database Sharding** only when write IOPS, working-set RAM, or total storage exceeds the vertical limit of a single primary server—or when strict tenant-level physical isolation is required.

---

## Enterprise Connection Pooling in Java and Spring Boot

### The Physical Cost of Connection Creation and Pooling Mechanics

Establishing a new physical connection from a Java application to an RDBMS is an extraordinarily expensive operation involving:
1. Multi-step TCP three-way handshake over the network.
2. TLS cryptographic negotiation and certificate exchange.
3. Database authentication, authorization, and session environment initialization.
4. **Physical Process Spawning:** In PostgreSQL, the server forks a heavyweight operating system process for *every single client connection*, allocating dedicated memory buffers.

```
       [Spring Boot Threads]                   [Database Engine]
         (Thread 1) (Thread 2)
              \       /
               v     v
         +-------------------+  Keep-Alive TCP  +-------------------+
         | HikariCP Pool     |===============>  | PostgreSQL Server |
         | [Conn 1] [Conn 2] |                  | (Pre-allocated    |
         | [Conn 3] [Conn 4] |                  |  Backend Processes|
         +-------------------+                  +-------------------+
```

#### How Connection Pooling Operates
A Connection Pool maintains a warm cache of pre-allocated physical connections:
* **Startup:** Opens and authenticates a configured number of connections during application startup.
* **Borrowing:** When a Spring `@Transactional` method or repository executes, a thread borrows an active connection from the pool in microseconds.
* **Returning:** When the transaction completes, calling `connection.close()` does not sever the physical TCP link; it resets connection state and returns the connection to the pool for reuse.

### HikariCP Configuration in Spring Boot and application.yml Architecture

Since Spring Boot 2.0, **HikariCP** is the default high-performance JDBC connection pool, engineered for zero-overhead lock-free concurrency.

```yaml
# application.yml
spring:
  datasource:
    url: jdbc:postgresql://localhost:5432/enterprise_db
    username: db_user
    password: secret_password
    driver-class-name: org.postgresql.Driver
    hikari:
      pool-name: EnterpriseHikariPool
      maximum-pool-size: 15          # Max physical connections allowed in pool
      minimum-idle: 5                # Min idle connections maintained during quiet periods
      connection-timeout: 30000      # Max wait time (30s) for a thread to acquire connection
      idle-timeout: 600000           # Max time (10 min) an idle connection is retained
      max-lifetime: 1800000          # Max lifespan (30 min) before connection is retired/recycled
      leak-detection-threshold: 2000 # Alerts if connection is held outside pool > 2s (leak detection)
```

### Optimal Pool Sizing Formulas and Context Switching Penalties

A common junior anti-pattern is configuring a massive connection pool (e.g., `maximum-pool-size: 500`) under the assumption that more connections equal higher throughput.

* **Why Massive Pools Destroy Performance:**
  * Database servers are bounded by physical CPU cores and storage I/O channels.
  * If 500 threads bombard a 16-core database server simultaneously, the operating system spends more CPU time performing **thread context switching** and disk head contention than executing SQL.
  * In PostgreSQL, 500 connections spawn 500 heavy OS processes, exhausting server RAM.
* **The Canonical HikariCP Sizing Formula:**
  Derived from disk spindle physics and queueing theory:

$$\text{Pool Size} = (\text{CPU Core Count} \times 2) + \text{Effective Spindle Count}$$

For modern NVMe/SSD storage (where effective spindle count is approximately 1):

$$\text{Pool Size} = (\text{CPU Cores} \times 2) + 1$$

* *Practical Benchmark:* A PostgreSQL server with 8 CPU cores and SSDs delivers peak throughput with a pool size of $(8 \times 2) + 1 = 17$ connections.
* *Enterprise Recommendation:* For standard enterprise Spring Boot microservices, a `maximum-pool-size` of **10 to 20** is optimal.
