# Database Systems - Questions - Chapter 2: Transactions, ACID & Concurrency Control

## Database Transactions and the ACID Paradigm

### What are the ACID properties in databases and why are they important for maintaining reliable transactions?

> **Direct Answer:** ACID represents Atomicity (all operations succeed or the entire transaction is rolled back), Consistency (transactions transition the database between valid states while preserving all schema invariants and constraints), Isolation (concurrent transactions execute independently without observing intermediate uncommitted states), and Durability (committed changes are permanently flushed via Write-Ahead Logging and `fsync` to survive crashes). These properties are critical because they prevent data corruption, partial state writes, race conditions, and catastrophic data loss during concurrent operations and unexpected hardware failures.

> **Key Nuance / Trade-off:** While Atomicity and Durability are absolute engineering guarantees enforced by logs (WAL/Undo logs), Consistency and Isolation involve architectural trade-offs. Perfect isolation (Serializable) dramatically degrades concurrency throughput by requiring extensive locking or aborting concurrent transactions upon write skew; consequently, production systems typically balance isolation by default using Multi-Version Concurrency Control (MVCC) at `READ COMMITTED` or `REPEATABLE READ`.

> **Reference Concept:** `Definition and Anatomy of a Database Transaction`
