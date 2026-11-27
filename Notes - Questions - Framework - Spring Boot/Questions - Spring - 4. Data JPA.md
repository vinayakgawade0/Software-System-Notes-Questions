# Spring Data JPA & Database Architecture - Interview Q&A

### Explain how to configure a datasource in Spring Boot.

> **Direct Answer:**
> - **Dependency & Properties:** Add `spring-boot-starter-data-jpa` and the database driver (e.g., `mysql-connector-java`), then configure connection properties (`spring.datasource.url`, `username`, `password`, `driver-class-name`) in `application.properties` or `application.yml`.
> - **Connection Pool Auto-Configuration:** Spring Boot automatically creates a `DataSource` bean powered by HikariCP as the high-performance default connection pool.
> - **Multi-DataSource Setup:** Declare custom `@Configuration` classes with separate `DataSource` beans using `DataSourceBuilder` and `@ConfigurationProperties`, marking the primary with `@Primary` and secondary with `@Qualifier`.
>
> **Key Nuance / Trade-off:**
> - In multi-datasource setups, auto-configuration backs off; you must explicitly configure a dedicated `LocalContainerEntityManagerFactoryBean` and `PlatformTransactionManager` for each database.
>
> **Reference Concept:** `DataSource Configuration & Connection Pooling`

---

### What are the architectural benefits of using the DAO pattern?

> **Direct Answer:**
> - **Strict Separation of Concerns:** Acts as an architectural firewall between business rules and database vendor logic, keeping the Service layer pure.
> - **Loose Coupling via Abstraction:** Exposes data operations through interface contracts (e.g., `UserDao`), allowing seamless switching between database engines (e.g., Oracle to PostgreSQL) or persistence technologies (JDBC to Hibernate) without modifying business code.
> - **Deterministic Testability:** Interfaces allow trivial injection of mock DAOs to simulate database behavior, timeouts, and errors in isolated, fast unit tests.
> - **Centralized Query Optimization:** Consolidates SQL queries and data access logic into single classes, ensuring schema changes and performance indexing require updates in only one place.
>
> **Key Nuance / Trade-off:**
> - **Exception Translation:** The DAO layer encapsulates low-level checked persistence exceptions (e.g., `java.sql.SQLException`) and translates them into Spring's unchecked `DataAccessException` hierarchy, keeping the Service layer decoupled from database error codes.
>
> **Reference Concept:** `Architectural Benefits of the DAO Pattern`

---

### How do you implement custom queries/methods in Hibernate or Spring Data JPA?

> **Direct Answer:**
> - **Derived Query Methods:** Declare query method signatures directly in your `JpaRepository` interface using naming conventions (e.g., `findByDepartment(String dept)`), and Spring Data automatically derives the query.
> - **`@Query` Annotation:** Define explicit queries directly on interface methods using JPQL (e.g., `@Query("SELECT e FROM Employee e WHERE e.department = :dept")`) or raw SQL via `nativeQuery = true`.
> - **Custom Repository Implementation:** For complex or dynamic logic, create a custom interface, implement it in a class suffixed with `Impl` (e.g., `EmployeeRepositoryImpl`) using `@PersistenceContext EntityManager`, and extend both interfaces in the main repository.
> - **Hibernate Session API:** In pure Hibernate without Spring Data, autowire `SessionFactory` and execute queries directly via `sessionFactory.openSession().createQuery(...)`.
>
> **Key Nuance / Trade-off:**
> - Spring Data strictly relies on the `Impl` naming convention (e.g., `<RepositoryName>Impl`) to automatically discover and composite the custom implementation with the runtime proxy.
>
> **Reference Concept:** `Custom Queries & Method Implementations`

---

### Explain difference between JPQL query and Native query in JPA

> **Direct Answer:**
> - **JPQL (Java Persistence Query Language):** An object-oriented query language targeting JPA Entities and their attributes rather than physical database tables (e.g., `SELECT u FROM User u WHERE u.email = :email`).
> - **Native Query:** Standard raw SQL targeting physical database tables and columns (e.g., `SELECT * FROM users_table WHERE user_email = ?`).
> - **Portability & Caching:** JPQL is database-agnostic, portable across SQL dialects, and integrates with Hibernate's First- and Second-Level Caches with automatic entity mapping.
> - **Performance & Vendor Features:** Native queries provide low-level access to vendor-specific SQL functions, indexing hints, and optimized joins, but tie the application to a specific database vendor.
>
> **Key Nuance / Trade-off:**
> - JPQL queries are validated at application startup and compile time, catching syntax and type errors early. Native queries are validated only at runtime when executed and often require explicit `@SqlResultSetMapping` for complex projections.
>
> **Reference Concept:** `JPQL vs. Native Queries`

---

### You are performing multiple database operations, and you want all of them to succeed or none at all (rollback on failure). How do you handle this?

> **Direct Answer:**
> - **Declarative Transaction Management:** Wrap the database operations inside a service method annotated with `@Transactional`.
> - **AOP Proxy Interception:** Spring creates an Aspect-Oriented Programming (AOP) proxy around the bean that intercepts the invocation, opens a transaction before method execution, and issues a `COMMIT` upon successful completion.
> - **Automatic Rollback:** If any operation in the method fails and throws an exception, the proxy catches it and issues an immediate `ROLLBACK`, restoring the database to its exact pre-transaction state.
>
> **Key Nuance / Trade-off:**
> - **The Checked Exception Trap:** By default, `@Transactional` only rolls back for unchecked exceptions (`RuntimeException` and `Error`). In production, always specify `@Transactional(rollbackFor = Exception.class)`.
> - **The Swallowed Exception Trap:** Catching exceptions in a `try-catch` block without re-throwing hides the failure from the proxy, causing Spring to commit partial state.
> - **The Self-Invocation Trap:** Calling a `@Transactional` method from within the same class (`this.method()`) bypasses the Spring AOP proxy, causing the transaction to be silently ignored.
>
> **Reference Concept:** `Transaction Management & Rollback Mechanics`

---

### What is a global transaction in Spring, and how is it different from a local transaction?

> **Direct Answer:**
> - **Local Transactions:** Scoped to a single database connection. Highly optimized, fast, and low-latency, but cannot coordinate atomic changes across disparate systems (e.g., updating a DB and publishing to Kafka).
> - **Global (Distributed) Transactions:** Spans multiple independent databases or resources (e.g., Payment DB, Warehouse DB, JMS broker) using the Java Transaction API (JTA) and the Two-Phase Commit (2PC) protocol.
> - **Two-Phase Commit (2PC):** Phase 1 (Prepare) queries all resources to lock rows and confirm readiness; Phase 2 (Commit) issues the global commit if all agreed, or global rollback if any failed.
> - **Spring Abstraction:** Spring provides a unified programming model where developers use `@Transactional` regardless of whether the underlying manager is a local `DataSourceTransactionManager` or a global `JtaTransactionManager`.
>
> **Key Nuance / Trade-off:**
> - **The 2PC Anti-Pattern:** Holding distributed row locks across networks introduces severe latency and thread exhaustion bottlenecks. Modern microservices favor **Eventual Consistency** via the **Saga Pattern**, replacing 2PC with a sequence of fast local transactions and compensating transactions.
>
> **Reference Concept:** `Local vs. Global (Distributed) Transactions`

---

### Q. How would you coordinate order, payment, and inventory changes across separate databases using a Saga? What happens when a later step fails, and when would you choose this over XA/2PC?

**Structured Answer Roadmap (Follow in Sequence):**
1. **Contrast With XA/2PC First:** Refer to [Why XA Two-Phase Commit Fails at Scale](<./Notes - Spring - 4. Data JPA.md#why-xa-two-phase-commit-fails-at-scale>) — explain why holding locks across separate Order, Payment, and Inventory databases degrades throughput and availability.
2. **Walk Through the Forward Saga Flow (Order → Payment → Inventory):** Refer to [Saga Pattern Architecture and Coordination Models](<./Notes - Spring - 4. Data JPA.md#saga-pattern-architecture-and-coordination-models>) — choose **Orchestration** (or **Choreography**) and trace the exact execution sequence using local `@Transactional` boundaries + the **Transactional Outbox Pattern**:
   - **Step 1 (`Order Service`):** Locally persists `Order` with status `PENDING` and writes an `OrderCreated` outbox event.
   - **Step 2 (`Payment Service`):** Consumes the event/command, locally authorizes/captures payment (`PAYMENT_COMPLETED`), and emits `PaymentProcessed`.
   - **Step 3 (`Inventory Service`):** Consumes `PaymentProcessed`, reserves stock (`INVENTORY_RESERVED`), and emits `InventoryReserved` → `Order Service` transitions order status to `CONFIRMED`.
3. **Trace Failure & Reverse Compensation:** Refer to [Failure Handling and Compensating Transactions](<./Notes - Spring - 4. Data JPA.md#failure-handling-and-compensating-transactions>) — explain what happens when Step 3 (`Inventory Service`) fails due to out-of-stock:
   - `Inventory Service` emits `InventoryReservationFailed`.
   - **Reverse Compensation:** `Payment Service` executes an idempotent **refund/void** compensating transaction (`PaymentRefunded`), followed by `Order Service` marking the order `CANCELLED` (while `PENDING` status prevented premature fulfillment during the intermediate window).
4. **Conclude With Trade-Off Selection Criteria:** Refer to [When to Choose Saga vs XA Two-Phase Commit](<./Notes - Spring - 4. Data JPA.md#when-to-choose-saga-vs-xa-two-phase-commit>) — state when Saga wins (microservices, high throughput, polyglot/external APIs) vs. when XA/2PC is justified (low-volume, tightly coupled XA-compliant DBs requiring strict immediate isolation).

---

### How do you handle retry if a batch insert operation fails in between?

> **Direct Answer:**
> - **Transactional Rollback:** Wrap batch operations in a transaction so that midway failures roll back the entire chunk, preventing partial or inconsistent data persistence.
> - **Targeted Retry:** Apply retries strictly to transient exceptions (deadlocks, lock timeouts, network blips) while failing fast on permanent errors (validation failures, constraint violations).
> - **Exponential Backoff with Jitter:** Use exponential backoff with randomized jitter (e.g., 1.2s, 2.5s, 4.1s) to desynchronize concurrent threads and avoid the "Thundering Herd" problem against a struggling database.
> - **Dead Letter Queue (DLQ):** Route chunks that exhaust all retry attempts to a DLQ (database table or Kafka topic) via Spring Batch `SkipPolicy` or Spring Retry `@Recover` so the remaining batch jobs can complete.
>
> **Key Nuance / Trade-off:**
> - **Mandatory Idempotency:** Retrying is unsafe unless operations are idempotent; insert payloads must implement natural business key Upsert/Merge semantics to avoid throwing `UniqueConstraintViolation` on re-execution.
>
> **Reference Concept:** `Batch Processing & Failure Retry Strategies`

---

### How do you implement pagination in a Spring Boot backend, and what are the architectural trade-offs when dealing with massive datasets?

> **Direct Answer:**
> - **Offset Pagination Implementation:** Pass Spring Data's `Pageable` interface (`PageRequest.of(page, size)`) into repository query methods. Spring resolves query params (`?page=0&size=10`) into SQL `LIMIT` and `OFFSET` queries and returns a `Page<T>` object containing the data slice and total count metadata.
> - **The Deep Paging Bottleneck:** Standard offset pagination degrades severely on large tables ($O(N)$ scanning); requesting page 100,000 forces the database engine to scan, count, and discard 999,999 records before returning 10 rows, spiking CPU and I/O.
> - **Keyset (Cursor) Pagination Solution:** For massive datasets, replace offset pagination with Keyset pagination by passing the `lastSeenId`. The query uses `WHERE id > :lastSeenId` to leverage the primary key B-Tree index for constant $O(1)$ seek time.
>
> **Key Nuance / Trade-off:**
> - Keyset pagination provides constant-time performance across millions of rows, but eliminates the ability to jump directly to arbitrary page numbers (e.g., "Skip to Page 45") and requires ordering by a sequential, indexed column.
>
> **Reference Concept:** `Pagination Architectures & Large Dataset Scalability`

---

### When architecting a data access layer, how do you decide between using an ORM like JPA (Hibernate) versus raw JDBC (or Spring JdbcTemplate)?

> **Direct Answer:**
> - **Choose JPA (Hibernate):** When building domain-driven applications with complex object graphs and standard OLTP CRUD operations. Maximizes developer velocity through automated dirty checking, entity lifecycle management, and transparent caching.
> - **Choose JDBC / Spring `JdbcTemplate`:** When requiring bare-metal throughput, executing complex reporting aggregations, performing massive bulk inserts/updates, or utilizing vendor-specific SQL optimization and database hints.
> - **JPA Pitfalls:** Hiding SQL can trigger the N+1 query problem, and bulk operations can exhaust JVM heap memory by loading thousands of managed entities into the Persistence Context.
> - **JDBC Pitfalls:** Demands tedious, error-prone manual `ResultSet` mapping boilerplate and lacks entity dirty checking.
>
> **Key Nuance / Trade-off:**
> - **The CQRS Hybrid Pattern:** Enterprise systems frequently implement Command Query Responsibility Segregation (CQRS)—using Spring Data JPA for the Command/Write model to enforce domain business rules, and `JdbcTemplate`/JOOQ for the Query/Read model for zero-overhead performance without entity caching overhead.
>
> **Reference Concept:** `Data Access Selection: ORM (JPA/Hibernate) vs. Bare Metal (JDBC)`

---

### How do you use @EmbeddedId in JPA, and what are the strict rules for implementing it?

> **Direct Answer:**
> - **Core Concept:** Encapsulates composite primary key fields (multi-column keys) into a dedicated class annotated with `@Embeddable`, which is then embedded into the entity using the `@EmbeddedId` annotation.
> - **Rule 1 - Implement `java.io.Serializable`:** Mandatory so JPA providers can serialize keys for second-level caching and cross-tier transmission.
> - **Rule 2 - Override `equals()` and `hashCode()`:** Mandatory so Hibernate's Persistence Context (Level 1 Cache) can accurately evaluate entity identity and state.
> - **Rule 3 - No-Argument Constructor:** Mandatory for reflection-based instantiation by the persistence provider when constructing entities from result sets.
>
> **Key Nuance / Trade-off:**
> - **`@EmbeddedId` vs. `@IdClass`:** `@EmbeddedId` adheres to clean object-oriented design by treating composite keys as a single cohesive object, though it makes JPQL paths slightly longer (e.g., `e.id.departmentId`). `@IdClass` keeps key fields flat on the entity root for simpler JPQL, but clutters entity attributes.
>
> **Reference Concept:** `Composite Primary Keys (@EmbeddedId vs. @IdClass)`

---

### Q. How does lazy loading work in Hibernate? Given one query for orders followed by one query per order for its items, how would you detect the N+1 problem, choose a fetch strategy, and verify the fix?

**Structured Answer Roadmap (Follow in Sequence):**
1. **Explain Proxy & Collection Interception:** Refer to [How Lazy Loading Works Internally](<./Notes - Spring - 4. Data JPA.md#how-lazy-loading-works-internally>) — explain how Hibernate wraps `Order.items` in a `PersistentCollection` proxy during the initial `SELECT * FROM orders` query, deferring SQL until iteration, which triggers $N$ individual `SELECT * FROM order_items WHERE order_id = ?` queries.
2. **Walk Through Detection Mechanisms:** Refer to [Detecting the N+1 Query Problem](<./Notes - Spring - 4. Data JPA.md#detecting-the-n1-query-problem>) — cover SQL logging with bind parameters, Hibernate Statistics, automated query-count assertions in integration tests (Datasource-Proxy), and APM/database profiling.
3. **Compare and Select the Fetch Strategy:** Refer to [Choosing the Right Fetch Strategy](<./Notes - Spring - 4. Data JPA.md#choosing-the-right-fetch-strategy>) — frame the choice based on the access pattern:
   - Use **`JOIN FETCH`** or **`@EntityGraph(attributePaths = "items")`** when fetching unpaginated orders with their items in one round-trip.
   - Use **`@BatchSize`** (e.g., `size = 25`) when the orders endpoint is paginated (`Page<Order>`) to avoid `JOIN FETCH` in-memory pagination (`HHH90003004`).
   - Use **DTO / Interface Projections** if the endpoint is strictly read-only and does not need managed entity state.
4. **Describe Verification Steps:** Refer to [Verifying the Fix](<./Notes - Spring - 4. Data JPA.md#verifying-the-fix>) — confirm the SQL output collapses from $N+1$ statements to 1 `JOIN` (or $\lceil N/25 \rceil + 1$ batched queries), verify `EXPLAIN ANALYZE` uses the `order_id` foreign-key index, and assert the exact query count in tests.
