# Database Systems - Questions - Chapter 5: Scalability, Stored Procedures & Connection Pooling

## Stored Procedures and Database Programmability

### Do you have experience with Stored Procedures? What are they, and when should we use them?

> **Direct Answer:** A Stored Procedure is a precompiled batch of SQL statements combined with procedural programming logic (loops, conditionals, variable assignments, and exception handling) stored and executed directly inside the database engine. In modern microservice architecture, standard business logic is maintained in the application layer (Java/Spring Boot) to ensure horizontal scalability, testability, and avoid vendor lock-in. Stored procedures are primarily reserved for massive, data-local batch processing (such as calculating nightly interest across 50 million rows), where streaming millions of records across the network would cause network saturation and JVM Out-Of-Memory errors.

> **Key Nuance / Trade-off:** While stored procedures reduce network round-trips by running logic directly next to the storage engine, they shift heavy compute burdens onto the database CPU, which is notoriously difficult and expensive to scale compared to horizontally scaling stateless application containers. Furthermore, stored procedures create severe database vendor lock-in and complicate modern CI/CD deployment pipelines.

> **Reference Concept:** `Stored Procedure Architecture, Execution, and Implementation`

---

## Database Scalability and High-Availability Strategies

### How do you ensure database performance and scalability in large systems using strategies such as replication, sharding, caching, and connection pooling?

> **Direct Answer:** Database performance and scalability are achieved through a layered architecture: optimizing queries with B-Tree indexes and execution plans, leveraging connection pooling (HikariCP) to avoid connection churn, deploying in-memory caching layers (Redis/Memcached) via Cache-Aside to intercept 80–90% of reads, and scaling reads via Primary-Replica replication. When dataset size and write throughput exceed single-node hardware limits, horizontal sharding is introduced to partition rows across independent database clusters based on a deterministic shard key.

> **Key Nuance / Trade-off:** Read replication introduces "Replication Lag," creating consistency hazards where users may not immediately see their own writes if reading from a lagging replica. Sharding provides near-infinite write scalability but breaks cross-shard relational `JOIN` operations and forces multi-node transactions into complex, high-latency Distributed Two-Phase Commit (2PC) or Saga patterns.

> **Reference Concept:** `Vertical Scaling vs Horizontal Scaling Architectures`

---

## Enterprise Connection Pooling in Java and Spring Boot

### What is Connection Pooling in Java / Spring Boot, why is it necessary, and how is it configured?

> **Direct Answer:** Connection pooling is a performance optimization that maintains a warm cache of pre-allocated physical database connections, eliminating the high CPU, network handshake, authentication, and process-spawning overhead of opening new connections for every client request. In Spring Boot, it is managed automatically by HikariCP and configured in `application.yml` via properties such as `maximum-pool-size`, `minimum-idle`, `connection-timeout`, and `max-lifetime`. Threads borrow connections in microseconds and return them to the pool without closing the underlying TCP socket.

> **Key Nuance / Trade-off:** A common misconception is that larger connection pools increase throughput; in reality, setting an excessively large pool (e.g., 500 connections) degrades performance due to severe CPU thread context switching and disk I/O queue contention, especially in engines like PostgreSQL that fork a physical OS process per connection. Following the canonical sizing formula $\text{Pool Size} = (\text{CPU Cores} \times 2) + 1$, a pool size of 10 to 20 connections is optimal for most enterprise microservices.

> **Reference Concept:** `The Physical Cost of Connection Creation and Pooling Mechanics`
