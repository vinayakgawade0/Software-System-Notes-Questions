# Database Systems - Chapter 2: Transactions, ACID & Concurrency Control

## Database Transactions and the ACID Paradigm

### Definition and Anatomy of a Database Transaction

A database transaction is a logical unit of work comprising one or more SQL operations executed as an atomic operation against a database state. Transactions are designed to satisfy the ACID properties (Atomicity, Consistency, Isolation, Durability) to guarantee data validity despite software crashes, hardware failures, concurrent client requests, or power outages.

```
       +-----------------------------------------------------------+
       |                  ACID Transaction Bounds                  |
       +-----------------------------------------------------------+
       |  BEGIN TRANSACTION;                                       |
       |    UPDATE accounts SET balance = balance - 500 WHERE id=1;|
       |    UPDATE accounts SET balance = balance + 500 WHERE id=2;|
       |  COMMIT; (or ROLLBACK on failure)                         |
       +-----------------------------------------------------------+
```

### Atomicity: All-or-Nothing Guarantees and Write-Ahead Logging

Atomicity dictates that a transaction is completely indivisible: either every single statement inside the transaction boundary completes successfully and commits, or the entire transaction is aborted and rolled back to its initial state, leaving no intermediate or partial updates persisted in the database.

* **Internal Mechanics:**
  * **Write-Ahead Logging (WAL / Redo Log):** Before modifying any data pages in memory or writing them to table files on disk, the database engine synchronously flushes an append-only log record of the pending change to persistent disk.
  * **Undo Logging / Rollback Segments:** The database records the original, before-image state of the rows in an undo log. If an unhandled exception, syntax error, deadlock, or client disconnect occurs before `COMMIT`, the engine uses the undo log to rewind modified pages back to their pre-transaction values.

### Consistency: Invariant Preservation and Schema Constraints

Consistency guarantees that a transaction moves the database from one valid state to another valid state, satisfying all declared schema invariants, business rules, and relational integrity constraints.

* **Internal Mechanics:**
  * **Explicit Declarative Invariants:** Foreign key cascades, `PRIMARY KEY` uniqueness, `NOT NULL` constraints, and domain-level `CHECK` constraints (e.g., `CHECK (balance >= 0)`) are strictly validated prior to commit.
  * **Application-Level Invariants:** Complex business logic invariants (e.g., double-entry bookkeeping where the sum of credits must equal debits across accounts) that span multiple tables are safeguarded within the transaction scope. If any constraint is violated, the transaction aborts automatically.

### Isolation: Concurrent Execution and Isolation Levels

Isolation ensures that concurrent transactions execute without interfering with one another. The intermediate states of an in-flight transaction are invisible to other concurrently running transactions until that transaction successfully commits.

* **The Challenge:** Complete isolation requires serial execution, which eliminates concurrency and severely bottlenecks database throughput. Databases provide configurable Isolation Levels that trade mathematical isolation for concurrency throughput.

### Durability: Crash Recovery, Redo Logs, and fsync Mechanics

Durability guarantees that once a transaction commits, its modifications are permanently recorded in non-volatile storage and will survive any subsequent operating system crash, power failure, or hardware fault.

* **Internal Mechanics:**
  * **The `COMMIT` Contract:** A transaction `COMMIT` command does not return success to the client application until the database engine issues a physical `fsync` call to flush the transaction's commit record from operating system page buffers onto persistent disk (SSD/NVMe).
  * **Crash Recovery Engine (ARIES Protocol):** Upon restarting after an unexpected crash, the engine scans the WAL:
    * **Redo Phase:** Replays all committed transactions from the last checkpoint forward to reconstruct the memory state.
    * **Undo Phase:** Reverses all uncommitted in-flight transactions that were active when the system crashed.

---

## Concurrency Phenomena and Isolation Levels

### ANSI SQL Isolation Levels and Read Anomalies

When multiple transactions read and write to the same rows concurrently, various race conditions—known as concurrency read anomalies—can occur if isolation is relaxed.

#### 1. Concurrency Anomalies Defined

| Read Anomaly | Formal Definition | Real-World Scenario |
| :--- | :--- | :--- |
| **Dirty Read** | Transaction A modifies a row without committing. Transaction B reads the uncommitted row. Transaction A rolls back. Transaction B acted on data that never existed. | User sees balance debited before transaction fails and rolls back. |
| **Non-Repeatable Read (Fuzzy Read)** | Transaction A reads a row. Transaction B updates or deletes that same row and commits. Transaction A re-reads the row and observes changed values or a missing row. | An invoice generator reads order total as \$100, re-reads midway through printing, and now sees \$120. |
| **Phantom Read** | Transaction A executes a range query (`WHERE age > 30`). Transaction B inserts new rows matching that range and commits. Transaction A re-executes the range query and sees new "phantom" rows. | A payroll report counts 50 senior engineers, but subsequent calculation finds 52. |
| **Serialization Anomaly (Write Skew)** | Two concurrent transactions read overlapping state, make independent disjoint updates based on that state, and violate an overall invariant that neither transaction violated individually. | Two doctors on call concurrently request leave when policy states at least one doctor must remain on call. Both requests succeed, leaving zero doctors. |

#### 2. ANSI SQL Isolation Levels Matrix

| Isolation Level | Dirty Read | Non-Repeatable Read | Phantom Read | Concurrency Mechanism |
| :--- | :--- | :--- | :--- | :--- |
| **READ UNCOMMITTED** | Permitted | Permitted | Permitted | Dirty read of uncommitted in-memory buffers; zero shared locks. |
| **READ COMMITTED** | Prevented | Permitted | Permitted | Default in PostgreSQL, Oracle, SQL Server. Reads always observe only committed data via short-lived shared locks or MVCC snapshots per statement. |
| **REPEATABLE READ** | Prevented | Prevented | Prevented (in MySQL/Postgres MVCC) | Default in MySQL InnoDB. Snapshot taken at transaction start; all queries within the transaction read from this consistent snapshot. |
| **SERIALIZABLE** | Prevented | Prevented | Prevented | Full serial execution guarantee via Strict Two-Phase Locking (S2PL) or Serializable Snapshot Isolation (SSI). |

### Lock-Based Concurrency Control vs Multi-Version Concurrency Control (MVCC)

Traditional database engines relied exclusively on two-phase locking (2PL) to enforce isolation: readers acquire Shared (`S`) locks, writers acquire Exclusive (`X`) locks. Under 2PL, **readers block writers, and writers block readers**, creating severe latency under high read-write contention.

Modern production databases (PostgreSQL, MySQL InnoDB, Oracle) overcome this bottleneck using **Multi-Version Concurrency Control (MVCC)**:
> **Core MVCC Axiom:** "Readers never block writers, and writers never block readers."

Instead of locking data rows, the database creates a new physical version of a row whenever an `UPDATE` or `DELETE` occurs. A reading query evaluates a historical snapshot of the database state consistent with the time the query (or transaction) started, completely ignoring in-flight uncommitted writes.

### PostgreSQL MVCC Mechanics: xmin, xmax, and Snapshot Isolation

PostgreSQL implements MVCC by attaching hidden system metadata columns to every physical row (tuple) in the heap:

```sql
SELECT ctid, xmin, xmax, * FROM users;
```

* **`xmin`:** The Transaction ID (`xid`) of the transaction that inserted the row.
* **`xmax`:** The Transaction ID of the transaction that updated or deleted the row (set to `0` if the row is live and active).
* **Row Updating Lifecycle:**
  1. When a transaction updates a row, PostgreSQL does not modify the existing tuple in-place.
  2. It writes an entirely new tuple into the table heap with its `xmin` set to the current transaction ID.
  3. It marks the existing tuple's `xmax` with the current transaction ID, transforming the old tuple into a **Dead Tuple** once the update commits.
* **Snapshot Evaluation:** When a query runs, PostgreSQL takes a snapshot consisting of the active transaction ID list. The query only reads tuples where `xmin` is committed and was committed *before* the snapshot began, and where `xmax` is either uncommitted, aborted, or committed *after* the snapshot began.

### MySQL InnoDB MVCC: Undo Logs, Redo Logs, and Purge Threads

Unlike PostgreSQL's append-only dead tuple approach, MySQL InnoDB handles row versions through in-memory undo log segments:

* **In-Place Updates:** InnoDB modifies table leaf pages in-place within the buffer pool.
* **Rollback Pointer (`DB_ROLL_PTR`):** Every row contains a hidden 7-byte pointer referencing an Undo Log record in the system tablespace.
* **Read Views:** When a transaction reads a row, InnoDB uses the rollback pointer to traverse backwards through the chain of undo logs to reconstruct the historical row version matching the transaction's Read View.
* **Purge Threads:** Background InnoDB purge threads continuously discard old undo logs once no active transaction's read view requires them, preventing undo log bloat.
