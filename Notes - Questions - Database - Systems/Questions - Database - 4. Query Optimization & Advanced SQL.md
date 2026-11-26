# Database Systems - Questions - Chapter 4: Query Optimization & Advanced SQL

## Query Optimization Principles and Execution Mechanics

### How do you optimize database queries and indexing to improve performance in relational databases?

> **Direct Answer:** Relational query optimization focuses on eliminating Full Table Scans ($O(N)$) in favor of B-Tree Index Scans ($O(\log N)$) on high-cardinality columns, writing strictly SARGable queries that avoid wrapping indexed columns in functions, and eliminating `SELECT *` in favor of explicit column lists to allow index-only scans. For multi-column queries, engineers design composite indexes honoring the leftmost prefix rule. Optimization requires balancing read acceleration against the write penalty, maintaining dense indexing on read-heavy OLAP systems while keeping indexes lean on write-heavy OLTP transactional tables.

> **Key Nuance / Trade-off:** Wrapping indexed columns in SQL functions (e.g., `WHERE YEAR(created_at) = 2023`) blinds the index, forcing the database engine to perform an expensive full table scan to compute the function on every single row before evaluating the condition. Re-factoring the query into an open range condition (`WHERE created_at >= '2023-01-01' AND created_at < '2024-01-01'`) restores full B-Tree index traversal.

> **Reference Concept:** `Full Table Scans vs Index Lookups and Cardinality Considerations`

---

## Analytical Query Building and Aggregate Pattern Matching

### Write an SQL query to find duplicate email addresses in a users table and return each email along with the number of times it appears.

> **Direct Answer:** Duplicate email addresses and their corresponding occurrence counts are retrieved by grouping the table by email and filtering the grouped partitions using the `HAVING` clause:
> ```sql
> SELECT email, COUNT(email) AS duplicate_count
> FROM users
> GROUP BY email
> HAVING COUNT(email) > 1;
> ```
> `GROUP BY email` gathers matching email values into logical partitions, `COUNT(email)` calculates the size of each partition, and `HAVING COUNT(email) > 1` filters out unique emails, returning only those occurring two or more times.

> **Key Nuance / Trade-off:** In SQL execution order, `WHERE` evaluates row-by-row *before* aggregation, meaning `WHERE COUNT(email) > 1` is syntactically invalid. `HAVING` evaluates *after* the aggregation step, allowing post-group filtering. On large tables, performance is greatly enhanced by maintaining an index on `email`, allowing the engine to stream sorted index entries or perform hash aggregation directly.

> **Reference Concept:** `Identifying Duplicate Records Using GROUP BY and HAVING Clauses`

---

### Given users, enrollments, and policies, write an SQL query to fetch each user and calculate the sum of all premiums considering only ACTIVE enrollments.

> **Direct Answer:** To retrieve all users while calculating total active premiums (and returning zero for users without active policies), use `LEFT JOIN` operations with the status filter placed directly inside the `ON` clause:
> ```sql
> SELECT
>     u.id,
>     u.name,
>     COALESCE(SUM(p.premium), 0) AS total_premium
> FROM users u
> LEFT JOIN enrollments e
>     ON e.user_id = u.id
>     AND e.status = 'ACTIVE'
> LEFT JOIN policies p
>     ON p.id = e.policy_id
> GROUP BY u.id, u.name
> ORDER BY u.id;
> ```
> If the business requirement strictly demands returning only users who currently have active enrollments, replace `LEFT JOIN` with `INNER JOIN`.

> **Key Nuance / Trade-off:** A critical trap is placing `WHERE e.status = 'ACTIVE'` instead of putting it in the `JOIN ... ON` clause. Placing it in the `WHERE` clause filters rows after the outer join; because users without enrollments produce `NULL` for `e.status`, evaluating `NULL = 'ACTIVE'` yields `UNKNOWN`, discarding those users and accidentally converting the query into an `INNER JOIN`. Furthermore, `COALESCE(SUM(p.premium), 0)` is essential to prevent returning SQL `NULL` values to the application layer.

> **Reference Concept:** `Multi-Table Aggregations: LEFT JOIN, ON Filtering, and COALESCE Handling`
