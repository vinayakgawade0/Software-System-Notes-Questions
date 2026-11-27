# Database Systems - Chapter 4: Query Optimization & Advanced SQL

## Query Optimization Principles and Execution Mechanics

### Full Table Scans vs Index Lookups and Cardinality Considerations

In relational systems, database performance hinges on minimizing physical disk input/output (I/O). When a query executes, the storage engine fetches data in fixed pages (typically 8 KB in PostgreSQL, 16 KB in MySQL InnoDB).

* **Full Table Scan (Sequential Scan):** The engine reads every single page from disk from the beginning of the table file to the end, resulting in $O(N)$ computational complexity. On a table with 10 million rows, this saturates storage I/O channels, sweeps valuable cached pages out of the buffer pool, and spikes CPU utilization.
* **B-Tree Index Scan:** The engine traverses a balanced tree with $O(\log N)$ complexity, requiring only 3–4 page reads from disk even on a multi-million row table, and fetches only the targeted leaf entries and matching heap pages.
* **Cardinality and Selectivity:** Indexes are only effective on **high-cardinality** columns (columns with many unique values, like `user_id`, `email`, or `uuid`). On low-cardinality columns (e.g., `gender`, `is_active BOOLEAN`), the query optimizer will intentionally ignore the index and favor a sequential scan because jumping back and forth between secondary index pages and heap pages (random disk I/O) is physically slower than a contiguous sequential table read.

### SARGable Queries and Function Blindness on Indexed Columns

**SARGable** stands for **Search Argument Able**. A query is SARGable if the database optimizer can use an existing index to accelerate the `WHERE` or `JOIN` evaluation.

```
+-----------------------------------+-----------------------------------+
|     Non-SARGable (Index Blinded)  |      SARGable (Index Accelerated) |
+-----------------------------------+-----------------------------------+
| WHERE YEAR(created_at) = 2023     | WHERE created_at >= '2023-01-01'  |
|                                   |   AND created_at <  '2024-01-01'  |
+-----------------------------------+-----------------------------------+
| WHERE LOWER(email) = 'a@b.com'    | WHERE email = 'a@b.com'           |
|                                   | (or use functional index)         |
+-----------------------------------+-----------------------------------+
| WHERE balance * 1.1 > 1000        | WHERE balance > 1000 / 1.1        |
+-----------------------------------+-----------------------------------+
| WHERE phone LIKE '%5551234'       | WHERE phone LIKE '5551234%'       |
+-----------------------------------+-----------------------------------+
```

* **Why Functions Blind Indexes:** A standard B-Tree index stores the raw column values (`created_at`), not the computed output of an arbitrary SQL function (`YEAR(created_at)`). When a column is wrapped in a function, the database cannot perform a binary tree seek; it must execute a Full Table Scan, invoking the function on every single row in the table before evaluating the equality.

### Composite Indexes and the Leftmost Prefix Rule

When queries frequently filter by multiple columns together (e.g., `WHERE last_name = 'Smith' AND first_name = 'John'`), architects create a **Composite Index (Multi-Column Index)**:

```sql
CREATE INDEX idx_users_name ON users(last_name, first_name);
```

#### The Leftmost Prefix Rule
A composite index sorts data by the first column first, and then within duplicate values of the first column, it sorts by the second column, and so forth.

```
       [Composite Index: (last_name, first_name)]
+--------------------+---------------------+--------------+
| last_name (Sorted) | first_name (Sorted) | row_pointer  |
+--------------------+---------------------+--------------+
| Adams              | Abigail             | ctid (1, 4)  |
| Adams              | John                | ctid (2, 8)  |
| Smith              | Alice               | ctid (3, 12) |
| Smith              | John                | ctid (1, 19) |
+--------------------+---------------------+--------------+
```

* `WHERE last_name = 'Smith' AND first_name = 'John'` $\rightarrow$ **INDEX USAGE: YES** (both columns used).
* `WHERE last_name = 'Smith'` $\rightarrow$ **INDEX USAGE: YES** (leftmost prefix matched).
* `WHERE first_name = 'John'` $\rightarrow$ **INDEX USAGE: NO / FULL SCAN** (without `last_name`, `first_name` is completely unsorted across the tree).

### Query Anti-Patterns: Eliminating SELECT * and Unnecessary Joins

1. **The `SELECT *` Anti-Pattern:**
   * **RAM and Bandwidth Waste:** Pulls unbounded columns, massive text fields, and unneeded attributes across the network wire into application heap memory.
   * **Prevents Index-Only Scans:** If a query only needs `id` and `email`, an index on `(id, email)` allows PostgreSQL to return results straight from the B-Tree leaf without visiting the table heap. `SELECT *` forces the engine to read every underlying heap page, doubling disk I/O.
2. **Workload-Specific Index Balancing:**
   * **OLAP (Analytical / Data Warehouses):** Read-heavy with minimal real-time writes. Tables should maintain extensive indexes, covering indexes, and bitmap indexes.
   * **OLTP (High-Speed Transactional Ledgers):** Write-heavy. Tables must maintain only the minimum necessary indexes to avoid write latency spikes and WAL bloat.

---

## Analytical Query Building and Aggregate Pattern Matching

### Identifying Duplicate Records Using GROUP BY and HAVING Clauses

A classic production challenge is detecting duplicate entity attributes (such as duplicate emails or phone numbers) within a table.

```sql
SELECT email, COUNT(email) AS duplicate_count
FROM users
GROUP BY email
HAVING COUNT(email) > 1;
```

#### Execution Mechanics
1. **`GROUP BY email`:** The database hashes or sorts the table based on the `email` column, gathering identical emails into distinct logical partitions.
2. **`COUNT(email)`:** The aggregate engine iterates through each grouped partition and tallies the number of rows within that partition.
3. **`HAVING COUNT(email) > 1`:** While the `WHERE` clause filters rows *before* grouping occurs, the `HAVING` clause filters aggregated group results *after* grouping is executed, returning only groups that contain duplicate entries.

### Multi-Table Aggregations: LEFT JOIN, ON Filtering, and COALESCE Handling

Consider an enterprise data model:
* `users` (`id`, `name`)
* `enrollments` (`id`, `user_id`, `policy_id`, `status`)
* `policies` (`id`, `premium`)

**Business Requirement:** Fetch each user and calculate the sum of all premiums considering only `ACTIVE` enrollments, while ensuring users with zero active policies are still included in the result set.

```sql
SELECT
    u.id,
    u.name,
    COALESCE(SUM(p.premium), 0) AS total_premium
FROM users u
LEFT JOIN enrollments e
    ON e.user_id = u.id
    AND e.status = 'ACTIVE'
LEFT JOIN policies p
    ON p.id = e.policy_id
GROUP BY u.id, u.name
ORDER BY u.id;
```

#### Architectural and SQL Nuances
1. **Preserving Users via `LEFT JOIN`:** An `INNER JOIN` would drop any user who does not have an enrollment. A `LEFT JOIN` guarantees that every user appears in the final output.
2. **Filtering Condition in `ON` vs `WHERE`:**
   * **Correct (`ON e.status = 'ACTIVE'`):** Evaluates the status *during* the join. If a user has only inactive enrollments or zero enrollments, the right side returns `NULL`, preserving the user row.
   * **The Fatal Trap (`WHERE e.status = 'ACTIVE'`):** Placing this condition in the `WHERE` clause filters rows *after* the join. Because `NULL = 'ACTIVE'` evaluates to `UNKNOWN`, it drops all users without active enrollments, accidentally converting the query into an `INNER JOIN`.
3. **`COALESCE(SUM(p.premium), 0)`:** For users with no active enrollments, `SUM(p.premium)` yields SQL `NULL`. `COALESCE` intercepts `NULL` and replaces it with `0`, returning clean numeric data to the application layer.
4. **Variant (Active Users Only):** If the requirement explicitly specifies that only users with active enrollments must be returned, use `INNER JOIN`:

```sql
SELECT
    u.id,
    u.name,
    SUM(p.premium) AS total_premium
FROM users u
JOIN enrollments e
    ON e.user_id = u.id
    AND e.status = 'ACTIVE'
JOIN policies p
    ON p.id = e.policy_id
GROUP BY u.id, u.name
ORDER BY u.id;
```
---

## Finding the Second-Highest and Nth-Highest Salary in SQL

### Edge Cases That Break Naive SQL Queries
When ranking salaries (or any numeric metric) in SQL, three edge cases determine whether a query is production-safe:
1. **Duplicate Top Salaries (Ties):** If two employees share the highest salary (`100000`, `100000`, `85000`), `ORDER BY salary DESC LIMIT 1 OFFSET 1` falsely returns `100000` instead of `85000` unless `DISTINCT` or `DENSE_RANK()` is used.
2. **Fewer Than $N$ Distinct Salaries:** If all employees earn the same salary or the table has only one row, the query must return `NULL` (single scalar result) or an empty set rather than failing or returning the top salary.
3. **NULL Salaries:** In SQL `ORDER BY salary DESC`, some engines (PostgreSQL, Oracle) place `NULL` values **first** by default (`NULLS FIRST`), which can pollute top-$N$ results unless filtered (`WHERE salary IS NOT NULL`) or ordered with `NULLS LAST`.

---

### Four Standard SQL Approaches (with Trade-Offs)

#### Approach 1: Window Function with `DENSE_RANK()` (Recommended for Production & $N$-th Rank)
Assigns consecutive ranks without gaps across duplicate values, making it trivial to generalize to the $N$-th highest salary or partition per department (`PARTITION BY department_id`).
```sql
WITH ranked_salaries AS (
    SELECT
        salary,
        DENSE_RANK() OVER (ORDER BY salary DESC) AS salary_rank
    FROM employees
    WHERE salary IS NOT NULL
)
SELECT MAX(salary) AS second_highest_salary
FROM ranked_salaries
WHERE salary_rank = 2;
```
- **Why `MAX(salary)` in the outer query?** If multiple employees tie for 2nd place, `MAX(salary)` collapses duplicates into a single row; if no 2nd rank exists, `MAX(salary)` cleanly returns `NULL`.
- **Per-Department Extension:** Add `PARTITION BY department_id` inside `OVER (...)` to find the 2nd highest salary within every department in a single scan.

#### Approach 2: Nested `MAX()` Subquery (ANSI SQL Portable, Strictly 2nd Highest)
Finds the maximum salary strictly less than the global maximum salary.
```sql
SELECT MAX(salary) AS second_highest_salary
FROM employees
WHERE salary < (SELECT MAX(salary) FROM employees);
```
- **Strengths:** Works on every SQL dialect (MySQL, PostgreSQL, Oracle, SQL Server, SQLite), handles ties automatically, and returns `NULL` when no 2nd distinct salary exists.
- **Limitation:** Does not generalize cleanly beyond $N = 2$ (deeply nested subqueries become unreadable).

#### Approach 3: `DISTINCT` with `LIMIT` / `OFFSET` (or ANSI `FETCH FIRST`)
Deduplicates salaries first, sorts descending, skips the top row, and picks the next row.
```sql
-- PostgreSQL / MySQL
SELECT (
    SELECT DISTINCT salary
    FROM employees
    WHERE salary IS NOT NULL
    ORDER BY salary DESC
    LIMIT 1 OFFSET 1
) AS second_highest_salary;

-- ANSI SQL Standard (PostgreSQL, Oracle 12c+, SQL Server 2012+)
SELECT DISTINCT salary AS second_highest_salary
FROM employees
WHERE salary IS NOT NULL
ORDER BY salary DESC
OFFSET 1 ROWS FETCH NEXT 1 ROWS ONLY;
```
- **Note:** Wrapping the query as a scalar subquery `SELECT (...)` ensures `NULL` is returned when fewer than two distinct salaries exist.

#### Approach 4: Correlated Subquery (Classic $N$-th Highest Pattern)
Counts how many distinct higher salaries exist for each candidate row; for the 2nd highest salary, exactly $N - 1 = 1$ distinct salary is greater.
```sql
SELECT DISTINCT e1.salary AS second_highest_salary
FROM employees e1
WHERE 1 = (
    SELECT COUNT(DISTINCT e2.salary)
    FROM employees e2
    WHERE e2.salary > e1.salary
);
```
- **Complexity Warning:** Runs in $O(M^2)$ time without an index on `salary`; prefer `DENSE_RANK()` for large tables.

---

### Comparison Summary and Indexing Impact
| Approach | Handles Ties? | Generalizes to $N$-th / Per-Group? | Performance & Index Usage |
| :--- | :--- | :--- | :--- |
| **`DENSE_RANK()` CTE** | Yes | Yes (`salary_rank = N`, `PARTITION BY`) | Single sort/index scan ($O(M \log M)$, or $O(M)$ with a B-Tree index on `salary`) |
| **Nested `MAX()` Subquery** | Yes | No (best only for $N = 2$) | Two fast B-Tree index boundary lookups ($O(\log M)$ with index on `salary`) |
| **`DISTINCT` + `OFFSET 1`** | Yes (with `DISTINCT`) | Yes (`OFFSET N - 1`), not per-group | Efficient with index scan on `salary DESC`, degrades for large `OFFSET` |
| **Correlated Subquery** | Yes (with `DISTINCT`) | Yes (`N - 1 = ...`), not per-group | Poor on large datasets ($O(M^2)$) |
