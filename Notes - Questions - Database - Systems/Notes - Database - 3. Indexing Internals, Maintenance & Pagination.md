# Database Systems - Chapter 3: Indexing Internals, Maintenance & Pagination

## Relational Indexing Fundamentals and PostgreSQL Implementations

### Core Indexing Concepts, B-Tree Architecture, and PostgreSQL Index Types

An index is an auxiliary physical data structure created alongside a table to accelerate record retrieval without scanning every page in table storage. Without an index, finding a row requires a **Full Table Scan (Sequential Scan)** with $O(N)$ computational complexity. With an index, the database traverses a self-balancing search tree in $O(\log N)$ time.

```
                  +-------------------+
                  |    Root Node      |
                  |     [ 50 | 100 ]  |
                  +---------+---------+
                            |
           +----------------+----------------+
           |                                 |
+----------v----------+           +----------v----------+
|  Internal Node      |           |  Internal Node      |
|     [ 20 | 35 ]     |           |     [ 70 | 85 ]     |
+----------+----------+           +----------+----------+
           |                                 |
+----------v----------+           +----------v----------+
|   Leaf Node         |           |   Leaf Node         |
| [20->ctid|35->ctid] |           | [70->ctid|85->ctid] |
+---------------------+           +---------------------+
```

#### Index Types in PostgreSQL
While B-Tree is the universal default, PostgreSQL provides specialized index engines tailored for distinct data distributions:

1. **B-Tree (Balanced Tree):** The default engine. Optimized for equality (`=`), range comparisons (`<`, `<=`, `>`, `>=`), and `BETWEEN` operations on sorted scalar data. Primary keys automatically generate unique B-tree indexes.
2. **Hash Index:** Memory-efficient key-value mapping optimized exclusively for equality lookups (`=`). Does not support range scans or sorting.
3. **GIN (Generalized Inverted Index):** Multi-key index designed for data types containing multiple internal elements (arrays, full-text search tsvector, and `JSONB`). Maps each sub-element to a list of matching row pointers.
4. **GiST (Generalized Search Tree):** Extensible geometric and balanced tree structure used for geospatial data (PostGIS), polygon bounding boxes, and overlapping range types (`tsrange`).
5. **SP-GiST (Space-Partitioned GiST):** Tailored for partitioned, unbalanced spaces such as quad-trees, suffix trees, and phone prefixes.
6. **BRIN (Block Range Index):** Stores only the minimum and maximum value for physical page ranges on disk. Extremely compact (kilobytes instead of gigabytes), ideal for multi-billion row append-only tables physically clustered by timestamp or sequence.

```sql
-- Creating standard B-tree index
CREATE INDEX idx_users_email ON users(email);

-- Query utilizing the B-tree index
SELECT * FROM users WHERE email = 'alex@example.com';
```

### Text Column Indexing Internals and Expression Indexes on String Attributes

When an index is placed on a non-numeric text column (such as `email VARCHAR(255)`), the database does not convert the string into an integer hash. Instead, it sorts the string values lexicographically (using the table's collation rules) directly within the B-Tree node pages.

* **Physical Index Tuple Structure:**
  Each B-Tree leaf entry contains:
  1. The raw or truncated string value (e.g., `'alex@example.com'`).
  2. The physical row pointer (`ctid` in PostgreSQL: block number and tuple offset).
* **Index-Only Scans:** If a query selects only the indexed column (e.g., `SELECT email FROM users WHERE email = 'alex@example.com'`), the engine reads the answer directly from the B-Tree leaf without fetching the table heap page from disk.
* **The Expression Index Caveat:**
  Standard B-Tree indexes are strictly case-sensitive. Wrapping a column in a function blinds a standard index:

```sql
-- Bad: Standard index idx_users_email is ignored; triggers a Full Table Scan
SELECT * FROM users WHERE LOWER(email) = 'alex@example.com';

-- Solution: Functional / Expression Index
CREATE INDEX idx_users_lower_email ON users (LOWER(email));
```

* **Wildcard Limitations:** Normal B-Trees only support prefix wildcards (`WHERE email LIKE 'alex%'`). Suffix wildcards (`WHERE email LIKE '%@gmail.com'`) cannot use standard B-Trees and require the `pg_trgm` extension with a GIN index:

```sql
CREATE EXTENSION IF NOT EXISTS pg_trgm;
CREATE INDEX idx_users_email_trgm ON users USING gin (email gin_trgm_ops);
```

### Over-Indexing Pitfalls, Write Amplification, and Index Auditing Strategies

While indexes accelerate reads, every additional index imposes heavy costs on database resources:

1. **Write Amplification (The Write Penalty):** Relational engines cannot defer index maintenance. Every single `INSERT`, `UPDATE` (of indexed columns), or `DELETE` forces the database to open, lock, and rebalance every attached B-Tree on disk.
2. **Buffer Pool Cache Contention:** Unused indexes consume precious RAM within the database buffer cache (`shared_buffers`), crowding out hot table data and driving up physical disk I/O.
3. **Table Bloat & Storage Overhead:** In write-heavy workloads, index size frequently exceeds the size of the underlying table data.

#### Auditing and Eliminating Unused Indexes in PostgreSQL
Senior engineers continuously audit indexes using internal catalog statistics to locate indexes with zero scans:

```sql
-- Identify indexes that are never scanned by queries
SELECT
    schemaname,
    relname AS table_name,
    indexrelname AS index_name,
    idx_scan AS number_of_scans,
    pg_size_pretty(pg_relation_size(indexrelid)) AS index_size
FROM pg_stat_user_indexes
WHERE idx_scan = 0
  AND indexrelname NOT LIKE '%_pkey'
ORDER BY pg_relation_size(indexrelid) DESC;
```

#### Remediation Strategies
* **Drop Unused Indexes:** Purge indexes with zero scans after a production observation window.
* **Consolidate with Composite Indexes:** Replace separate indexes on `(last_name)` and `(first_name)` with a single composite index on `(last_name, first_name)`.
* **Partial Indexes:** Index only the subset of rows actually queried:
  ```sql
  -- Partial Index: Only indexes 2% of the table, saving 98% disk & write overhead
  CREATE INDEX idx_active_orders ON orders(created_at) WHERE status = 'PENDING';
  ```

---

## High-Performance Database Pagination Architectures

### Keyset and Cursor-Based Pagination vs the OFFSET Anti-Pattern

Pagination is fundamental to user interfaces and APIs. However, naive implementation causes database degradation at scale.

#### 1. The Offset-Based Pagination Anti-Pattern
Offset pagination uses SQL `LIMIT` and `OFFSET` keywords:

```sql
SELECT id, name, email
FROM users
ORDER BY id
LIMIT 50 OFFSET 100000;
```

* **The Internal Failure Mode:** Relational database tables are not indexed arrays. The database engine cannot jump directly to row 100,000. It must physically read **100,050 rows from disk**, sort them, discard the first 100,000 rows, and return the final 50 rows.
* **Complexity:** $O(N)$ execution time. Page 1 loads in 5ms; Page 2,000 takes 4 seconds; Page 50,000 causes database CPU exhaustion and query timeouts.
* **Data Drift (Inconsistency):** If rows are inserted or deleted while a user navigates between pages, rows shift position, resulting in duplicated or skipped records across page views.

#### 2. The Architectural Solution: Keyset (Cursor-Based) Pagination
Instead of telling the database how many rows to skip, Keyset Pagination specifies exactly where to resume scanning using a `WHERE` filter on a strictly monotonic, indexed column (such as the Primary Key or `created_at` timestamp):

```sql
-- Keyset / Cursor Pagination
SELECT id, name, email
FROM users
WHERE id > 100000
ORDER BY id ASC
LIMIT 50;
```

* **Internal Mechanics:** Because `id` is indexed via a B-Tree, the storage engine executes a binary tree seek in $O(\log N)$ time, lands directly on key `100001`, reads exactly 50 adjacent leaf entries, and immediately stops.
* **Performance:** Execution latency is constant $O(1)$ regardless of whether fetching page 1 or page 50,000.
* **Trade-off:** Prevents random page jumping (e.g., "Jump to Page 42"); requires sequential "Next / Previous" navigation tokens, making it the industry standard for infinite scroll and high-scale REST/GraphQL APIs.

---

## Query Diagnostics and Execution Plan Analysis

### Diagnosing Slow Queries: EXPLAIN vs EXPLAIN ANALYZE Mechanics

SQL is declarative: applications declare *what* data they require, leaving *how* to retrieve it to the Cost-Based Query Optimizer. Diagnosing slow queries requires inspecting the optimizer's execution plan.

```sql
EXPLAIN SELECT * FROM users WHERE email = 'user@example.com';
EXPLAIN ANALYZE SELECT * FROM users WHERE email = 'user@example.com';
```

| Dimension | `EXPLAIN` | `EXPLAIN ANALYZE` |
| :--- | :--- | :--- |
| **Execution** | **Does NOT execute** the query. Estimates execution path using table statistics. | **Physically runs** the query to completion on the database server. |
| **Output Metrics** | Estimated startup and total cost (arbitrary disk/CPU units), estimated rows, estimated width. | Real execution time (milliseconds), actual row counts, actual loops per node, buffer hits/reads. |
| **Production Safety** | Completely safe for all queries, including `DELETE` and `UPDATE`. | **Destructive for write queries.** Running `EXPLAIN ANALYZE DELETE ...` will delete live rows (wrap in `BEGIN ... ROLLBACK` if testing). |
| **Primary Use Case** | Fast structural analysis of execution plans without executing expensive queries. | Definitive performance profiling, discovering timing bottlenecks, and pinpointing stale statistics. |

### Execution Plan Nodes and the Statistics Mismatch Bottleneck

PostgreSQL renders execution plans as nested operational trees read from the inside out (deepest indentation executes first).

#### Critical Plan Nodes
* **`Seq Scan` (Sequential Scan):** Full table scan reading every page from disk. Catastrophic when executed on multi-million row tables.
* **`Index Scan`:** Traverses a B-Tree index to find matching row pointers, then visits table heap pages to retrieve non-indexed columns.
* **`Index Only Scan`:** Reads all requested data directly from the index tree without fetching heap pages (highest performance).
* **`Nested Loop`:** Iterates through outer table rows and searches inner table for each row. Fast for small sets ($< 100$ rows) with indexed inner lookups; disastrous on large unindexed tables ($O(N \times M)$).
* **`Hash Join`:** Builds an in-memory hash table from the smaller relation, then streams the larger relation to find matches ($O(N + M)$).

#### The Statistics Mismatch Bottleneck
In an `EXPLAIN ANALYZE` output, compare `rows=...` (estimated rows) with `actual rows=...`:

```
->  Seq Scan on orders  (cost=0.00..1800.00 rows=10 width=32) (actual time=0.02..850.12 rows=2500000 loops=1)
```

If the optimizer estimates **10 rows** but the database actually processes **2,500,000 rows**, the internal statistics stored in `pg_statistic` are completely stale. The optimizer chose a slow plan (such as a Nested Loop) because it believed the dataset was tiny. The resolution is not rewriting the SQL query, but updating database statistics via `VACUUM ANALYZE`.

---

## Storage Maintenance and Vacuuming in PostgreSQL

### MVCC Table Bloat, Dead Tuples, and the Role of VACUUM

Under PostgreSQL's MVCC architecture, modifying a row (`UPDATE`) or removing a row (`DELETE`) never overwrites or frees physical disk blocks in-place:
1. `UPDATE` marks the current tuple as dead (`xmax` populated) and inserts a new live tuple.
2. `DELETE` marks the tuple as dead.
3. The dead tuple remains on disk to allow concurrent in-flight transactions with older snapshots to read consistent historical data.

* **The Problem (Table Bloat):** Once all transactions that could view the dead tuples have terminated, those tuples become garbage. If left unmanaged, dead tuples accumulate, causing physical table files to balloon, fragmenting data pages, and forcing queries to read megabytes of dead bytes during scans.
* **`VACUUM` (The Garbage Collector):**
  * Scans table pages, locates dead tuples, and marks their space within the page as available for future `INSERT` and `UPDATE` operations within that same table.
  * **Crucial Architectural Detail:** Standard `VACUUM` **does not shrink the physical file size on the OS disk**. It merely reclaims internal free space to prevent future file growth.
  * To physically compact pages and return disk space to the OS, an administrator must run `VACUUM FULL`. However, `VACUUM FULL` acquires an exclusive `ACCESS EXCLUSIVE` table lock that blocks all reads and writes until completion.

### VACUUM vs VACUUM ANALYZE and the Production Autovacuum Daemon

PostgreSQL provides distinct maintenance operations depending on whether disk reclamation, statistical planning, or both are required:

| Command | Disk Space Action | Optimizer Statistics Action | Production Impact |
| :--- | :--- | :--- | :--- |
| **`VACUUM`** | Reclaims dead tuple space within table pages for reuse. | **None.** Leaves optimizer table statistics unchanged. | Low to moderate I/O; concurrent reads and writes continue uninterrupted. |
| **`VACUUM ANALYZE`** | Reclaims dead tuple space within table pages for reuse. | **Updates `pg_statistic`.** Collects random distribution samples, column histograms, and null fractions. | Reclaims space AND ensures query optimizer selects optimal index plans. |
| **`ANALYZE`** | **None.** Does not reclaim dead tuples. | Updates optimizer statistics only. | Fast; useful immediately after massive batch loads. |

#### The Autovacuum Background Daemon
In modern production environments, engineers rarely execute manual vacuuming. PostgreSQL runs a background daemon called **`autovacuum`** that monitors write activity via the statistics collector.

When dead tuples exceed configured thresholds, autovacuum automatically triggers `VACUUM ANALYZE` in the background:

$$\text{Threshold} = \text{autovacuum\_vacuum\_threshold} + (\text{autovacuum\_vacuum\_scale\_factor} \times \text{reltuples})$$

* *Example:* If threshold is 50 rows and scale factor is 0.2 (20%), a table with 1,000,000 rows triggers autovacuum when 200,050 rows are updated or deleted.
* Manual `VACUUM ANALYZE` is reserved for off-peak maintenance following bulk data migrations or large-scale batch purges.
