# Spring Data JPA & Database Architecture

## DataSource Configuration & Connection Pooling

### Single DataSource Configuration
- **Dependencies (Maven `pom.xml`):**
  ```xml
  <dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-jpa</artifactId>
  </dependency>
  <dependency>
    <groupId>mysql</groupId>
    <artifactId>mysql-connector-java</artifactId>
  </dependency>
  ```
- **Configuration Properties (`application.properties` / `application.yml`):**
  ```properties
  spring.datasource.url=jdbc:mysql://localhost:3306/mydatabase
  spring.datasource.username=root
  spring.datasource.password=secret
  spring.datasource.driver-class-name=com.mysql.cj.jdbc.Driver

  spring.jpa.hibernate.ddl-auto=update
  spring.jpa.show-sql=true
  spring.jpa.database-platform=org.hibernate.dialect.MySQLDialect
  ```
- **Auto-Configuration:** Spring Boot auto-configures a `DataSource` bean using these properties.

### Connection Pooling with HikariCP
- **Default Connection Pool:** `HikariCP` is the default connection pool in Spring Boot. Pool size and behavior are configured directly:
  ```properties
  spring.datasource.hikari.maximum-pool-size=10
  spring.datasource.hikari.minimum-idle=5
  spring.datasource.hikari.idle-timeout=30000
  spring.datasource.hikari.max-lifetime=1800000
  ```
- **Performance:** These settings manage efficient database connection usage for concurrent requests and optimal performance.

### Multi-Database (Multi-DataSource) Configuration
- **Multi-Database Properties (`application.properties`):**
  ```properties
  # Primary DB
  spring.datasource.url=jdbc:mysql://localhost:3306/db1
  spring.datasource.username=root
  spring.datasource.password=pwd

  # Secondary DB
  second.datasource.url=jdbc:mysql://localhost:3306/db2
  second.datasource.username=root
  second.datasource.password=pwd
  ```
- **Configuration Classes:**
  ```java
  @Configuration
  public class PrimaryDataSourceConfig {
      @Bean
      @Primary
      @ConfigurationProperties("spring.datasource")
      public DataSource primaryDataSource() {
          return DataSourceBuilder.create().build();
      }
  }

  @Configuration
  public class SecondaryDataSourceConfig {
      @Bean
      @ConfigurationProperties("second.datasource")
      public DataSource secondaryDataSource() {
          return DataSourceBuilder.create().build();
      }
  }
  ```
- **Repository Wiring:** Annotate repositories with `@Primary` or `@Qualifier` as needed to target the appropriate datasource.

---

## Data Access Architecture & DAO Pattern

### Architectural Benefits of the DAO Pattern
1. **Strict Separation of Concerns (The Firewall):**
   - The DAO acts as an architectural firewall between business rules and the database vendor.
   - *The Threat:* If SQL queries are placed directly inside a `TransactionService`, business logic becomes polluted with database-specific syntax.
   - *The Benefit:* Persistence logic is forced out of the Service layer. The Service layer focuses entirely on what to do with data (business rules), while the DAO focuses entirely on how to store and retrieve it.
2. **Loose Coupling via Abstraction (The Interface Contract):**
   - Senior architecture relies on interfaces rather than concrete implementations.
   - *The Threat:* Tightly coupling an application to Oracle makes migrating to PostgreSQL technically and financially paralyzing.
   - *The Benefit:* The Service layer only references a `UserDao` interface, completely decoupled from whether the implementation is `UserDaoJdbcImpl`, `UserDaoHibernateImpl`, or `UserDaoMongoImpl`. The underlying database technology can be swapped without changing business logic.
3. **Deterministic Testability (Mocking):**
   - *The Threat:* Testing business logic hardcoded to a database requires spinning up a real database, populating test data, and cleaning up afterward (slow, flaky CI/CD pipelines).
   - *The Benefit:* Because the DAO is an interface, a "Mock DAO" can be injected into the Service layer during Unit Testing. Database timeouts or empty result sets are simulated in milliseconds in total isolation.
4. **Centralized Query Optimization (Reusability):**
   - *The Threat:* If a specific SQL `JOIN` is duplicated in 15 places and a DBA adds a new index requiring query tuning, all 15 instances must be tracked down and updated.
   - *The Benefit:* The DAO centralizes database access. Schema changes or query optimizations require updating code in only one single class.
5. **Exception Translation (The Spring/Java Flex):**
   - *The Threat:* Raw databases throw vendor-specific checked exceptions (e.g., `java.sql.SQLException`). Forcing the Service layer to catch SQL exceptions breaks the Separation of Concerns.
   - *The Benefit:* A proper DAO catches vendor-specific low-level exceptions and translates them into generic, unchecked domain exceptions (e.g., Spring's `DataAccessException`). The Service layer only needs to know that data access failed, not database-specific error codes (e.g., MySQL Error Code 1062).

---

## Custom Queries & Method Implementations

### Derived Query Methods (Method Naming Convention)
- Spring Data JPA derives SQL/JPQL queries automatically from method signatures defined in interfaces extending `JpaRepository`.
- No explicit SQL or HQL is required:
  ```java
  @Repository
  public interface EmployeeRepository extends JpaRepository<Employee, Long> {
      List<Employee> findByDepartment(String dept);
      Employee findByEmail(String email);
      List<Employee> findBySalaryGreaterThan(double salary);
  }
  ```

### Explicit Queries with @Query (JPQL & Native SQL)
- Used when queries cannot be expressed cleanly via method names:
  ```java
  @Repository
  public interface EmployeeRepository extends JpaRepository<Employee, Long> {

      // JPQL query
      @Query("SELECT e FROM Employee e WHERE e.department = :dept")
      List<Employee> getByDepartment(@Param("dept") String dept);

      // Native query
      @Query(value = "SELECT * FROM employee WHERE salary > :salary", nativeQuery = true)
      List<Employee> getHighEarners(@Param("salary") double salary);
  }
  ```

### Custom Repository Implementation (@PersistenceContext)
- Used when queries are too complex for `@Query` or require dynamic construction with `EntityManager`:
  - **Step 1: Define Custom Interface:**
    ```java
    public interface EmployeeRepositoryCustom {
        List<Employee> searchByName(String name);
    }
    ```
  - **Step 2: Provide Implementation Class (named `<RepositoryName>Impl`):**
    ```java
    @Repository
    public class EmployeeRepositoryImpl implements EmployeeRepositoryCustom {

        @PersistenceContext
        private EntityManager entityManager;

        @Override
        public List<Employee> searchByName(String name) {
            String hql = "FROM Employee e WHERE e.name LIKE :name";
            return entityManager.createQuery(hql, Employee.class)
                                .setParameter("name", "%" + name + "%")
                                .getResultList();
        }
    }
    ```
  - **Step 3: Extend Both in the Main Repository Interface:**
    ```java
    @Repository
    public interface EmployeeRepository 
            extends JpaRepository<Employee, Long>, EmployeeRepositoryCustom {
    }
    ```

### Direct Hibernate Session API
- Using Hibernate directly without Spring Data JPA abstractions via `SessionFactory`:
  ```java
  public class EmployeeDAO {

      @Autowired
      private SessionFactory sessionFactory;

      public List<Employee> getEmployeesByDept(String dept) {
          try (Session session = sessionFactory.openSession()) {
              Query<Employee> query = session.createQuery(
                  "FROM Employee e WHERE e.department = :dept", Employee.class);
              query.setParameter("dept", dept);
              return query.list();
          }
      }
  }
  ```

---

## JPQL vs. Native Queries

### Query Mechanics & Portability
- **JPQL (Java Persistence Query Language):**
  - **Object-Oriented:** Queries persistent Entities (Java classes) and attributes, not database tables (e.g., `SELECT u FROM User u WHERE u.email = :email`).
  - **Database Agnostic:** Hibernate translates JPQL into the target dialect (MySQL, PostgreSQL, Oracle). Switching databases requires no query rewrites.
  - **Caching Integration:** Leverages First-Level Cache (Persistence Context) and Second-Level Cache for repeated entity retrieval.
  - **Automatic Mapping:** Result sets automatically map to Entity graphs.
- **Native Query:**
  - **Raw SQL:** Target actual database table and column names (e.g., `SELECT * FROM users_table WHERE user_email = ?`).
  - **Database Specific:** Leverages vendor-specific features (PostgreSQL JSONB, Oracle specialized functions), tying the query to a specific engine.
  - **Performance Optimization:** Ideal for complex reporting, multi-table joins, or database hints unsupported by JPQL.
  - **Manual Mapping:** While Spring Data JPA maps simple queries to Projections/Interfaces, complex native projections often require `@SqlResultSetMapping`.

### Comparison Matrix (JPQL vs. Native Query)
| Feature | JPQL | Native Query |
| :--- | :--- | :--- |
| **Target** | Persistent Entities (Java Classes) | Database Tables |
| **Portability** | High (Works across different DBs) | Low (Tied to a specific DB) |
| **Complexity** | Best for simple to moderate CRUD | Best for complex, optimized SQL |
| **Validation** | Verified at application startup/compile | Verified only at runtime |
| **Functions** | Limited to JPQL-supported functions | Supports all DB-specific functions |

---

## Transaction Management & Rollback Mechanics

### Declarative Transactions (@Transactional) & AOP Proxy
- **Database Transactions:** Group multiple operations into an atomic unit: `COMMIT` on success, `ROLLBACK` on failure to revert to the exact pre-transaction state.
- **Enterprise Spring Solution:** Declarative transaction management using `@Transactional`:
  ```java
  @Service
  public class OrderService {

      @Transactional
      public void processOrder(Order order) {
          inventoryRepo.deductStock(order.getItemId()); // Op 1
          paymentRepo.chargeUser(order.getUserId());    // Op 2
          orderRepo.save(order);                        // Op 3
          
          // If anything fails here, Spring intercepts the exception 
          // and automatically rolls back all 3 operations.
      }
  }
  ```
- **AOP Proxy Engine:** Spring wraps the target class in an Aspect-Oriented Programming (AOP) proxy. The proxy intercepts the method call, starts a transaction before execution, and executes commit or rollback based on exit status.

### Senior Rollback Traps (Checked Exceptions, Swallowed Errors, Self-Invocation)
1. **Trap A: The Checked Exception Flaw:**
   - *Problem:* By default, `@Transactional` **only rolls back for unchecked exceptions** (`RuntimeException`, `Error`). If code throws a checked exception (`IOException`, `SQLException`), Spring commits the partial transaction.
   - *The Fix:* Explicitly configure rollback behavior:
     ```java
     @Transactional(rollbackFor = Exception.class)
     ```
2. **Trap B: The "Swallowed" Exception:**
   - *Problem:* Catching exceptions inside the method with a `try-catch` block without re-throwing prevents the AOP proxy from seeing the failure. Spring treats the execution as successful and commits invalid state:
     ```java
     @Transactional
     public void badMethod() {
         try {
             db.updateSomething();
             db.crashHere(); 
         } catch (Exception e) {
             log.error("Error occurred"); 
             // TRAP! The exception is caught and dies here.
             // Spring sees a normal exit and COMMITS the broken state!
         }
     }
     ```
3. **Trap C: The Proxy Bypass (Self-Invocation):**
   - *Problem:* `@Transactional` requires method calls to enter via the outer AOP proxy. If Method A (non-transactional) calls Method B (`@Transactional`) within the **same class** via `this.methodB()`, the call bypasses the proxy entirely, and the transaction is ignored.

---

## Local vs. Global (Distributed) Transactions

### Local Transactions & Boundary Limits
- **Scope:** Confined to a single database connection.
- **Mechanics:** Fast, lightweight, and low-latency because all operations share one connection and commit boundary.
- **Limitation:** Cannot coordinate with external systems. If an application updates a PostgreSQL database and then publishes an event to Apache Kafka, a local transaction cannot rollback the Kafka message if the database commit fails, or vice versa.

### Global Transactions (JTA & Two-Phase Commit)
- **Scope:** Coordinates across multiple independent data sources (e.g., Payment DB, Warehouse DB, JMS/Kafka broker).
- **Protocol:** Java Transaction API (JTA) manages transactions across systems using **Two-Phase Commit (2PC)**:
  - **Phase 1 (Prepare):** The coordinator asks all participating resources over the network to lock records and confirm readiness to commit.
  - **Phase 2 (Commit):** If all resources vote "Yes", the coordinator issues `COMMIT`. If any resource fails or votes "No", the coordinator issues `ROLLBACK` to all participants.
- **Spring Abstraction:** `@Transactional` remains identical in application code. Spring delegates to a `JtaTransactionManager` under the hood.

### Distributed Systems & The Saga Pattern (Eventual Consistency)
- **The Architectural Anti-Pattern:** 2PC requires distributed row locks across networks. If one resource experiences high latency or network partition, locks remain held, causing thread starvation and cascading failures.
- **Modern Solution:** Modern microservice architectures replace 2PC with **Eventual Consistency** via the **Saga Pattern**:
  - Eliminates long-lived distributed locks.
  - Chains a series of fast local transactions across services.
  - If a step fails midway (e.g., inventory deduction fails after payment), a **Compensating Transaction** is triggered (e.g., refund payment) to revert earlier state.

---

## Batch Processing & Failure Retry Strategies

### Retry Policies (Fixed Delay, Exponential Backoff, Jitter)
- When a batch insert fails midway, the entire chunk must roll back inside a transaction to prevent partial state corruption.
- Retries should target strictly **transient errors** (deadlocks, connection timeouts, network blips) and fail fast on permanent errors (data validation, constraint violations).
- **Backoff Strategies:**
  - **Fixed Delay:** Retries on a constant interval (e.g., every 2s). Risky in distributed systems due to the "Thundering Herd" problem when multiple instances retry simultaneously.
  - **Exponential Backoff:** Wait time doubles after each failure (e.g., 1s, 2s, 4s, 8s), easing pressure on recovering infrastructure.
  - **Exponential Backoff with Jitter:** Adds random jitter to exponential intervals (e.g., 1.2s, 2.5s, 4.1s) to desynchronize concurrent worker retries.
- **Mandatory Safety Mechanisms:**
  - **Idempotency:** Operations must be idempotent (e.g., natural business key Upsert/Merge) so re-processing a failed chunk does not trigger `UniqueConstraintViolation` errors.
  - **Dead Letter Queue (DLQ) / Fallback:** Failed chunks exceeding max retry attempts are pushed to a DLQ (database table or Kafka topic) for offline inspection without stalling the entire batch.

### Fault-Tolerant Batching (Spring Batch & Dead Letter Queue)
- Chunk-based processing using Spring Batch `faultTolerant()` and `SkipPolicy`:
  ```java
  @Bean
  public Step insertStep(JobRepository jobRepository, PlatformTransactionManager txManager) {
      return new StepBuilder("insertStep", jobRepository)
              .<User, User>chunk(100, txManager)
              .reader(reader())
              .processor(processor())
              .writer(writer())
              .faultTolerant()
              .retryLimit(3)
              .retry(TransientDataAccessException.class)
              .retry(CannotAcquireLockException.class)
              // Skip and route to DLQ if retries are exhausted
              .skip(TransientDataAccessException.class) 
              .skipLimit(10)
              .listener(new CustomSkipListenerForDLQ()) 
              .build();
  }
  ```

### Method-Level Retries (Spring Retry & @Recover)
- Declarative method retry for lightweight batch operations:
  ```java
  @Retryable(
      value = { TransientDataAccessException.class }, 
      maxAttempts = 4, 
      backoff = @Backoff(delay = 1000, multiplier = 2.0) // 1s, 2s, 4s
  )
  @Transactional
  public void insertBatch(List<User> users) {
      userRepository.saveAll(users);
  }

  // Fallback method if all 4 attempts fail
  @Recover
  public void recover(TransientDataAccessException e, List<User> users) {
      dlqService.sendToDeadLetterQueue(users, e.getMessage());
  }
  ```

---

## Pagination Architectures & Large Dataset Scalability

### Standard Offset Pagination (Pageable & Page<T>)
- **Core Interfaces:**
  - `org.springframework.data.domain.Pageable`
  - `org.springframework.data.domain.PageRequest`
  - `org.springframework.data.domain.Page`
- **Offset Mechanism:** Spring MVC resolves HTTP query parameters (`?page=0&size=10`) into a `Pageable` instance, which repositories translate into SQL `LIMIT` and `OFFSET` queries.
- **Implementation:**
  ```java
  // 1. The Repository
  @Repository
  public interface EmployeeRepository extends JpaRepository<Employee, Long> {
      // Spring appends LIMIT and OFFSET based on Pageable
      Page<Employee> findByDepartment(String department, Pageable pageable);
  }

  // 2. The Controller
  @GetMapping("/employees")
  public Page<Employee> getEmployees(
          @RequestParam String department, 
          Pageable pageable) { // Automatically injected by Spring
      
      return employeeRepository.findByDepartment(department, pageable);
  }
  ```

### The Deep Paging Problem
- **Bottleneck at Scale:** When requesting high page numbers (e.g., page 100,000), standard SQL `OFFSET` forces the database engine to scan, count, and physically discard 999,999 records before returning the requested 10 rows.
- **Symptoms:** High CPU spikes, memory saturation, and severe latency degradation on large tables.

### Keyset (Cursor) Pagination
- **O(1) Constant Time Scalability:** Instead of page numbers, clients send the unique identifier of the last record seen (`lastSeenId`).
- **Index Seek:** The database leverages the primary key B-Tree index to jump directly to the target record via `WHERE id > :lastSeenId`, executing in $O(1)$ constant time regardless of table size.
- **Implementation:**
  ```java
  // 1. The Repository
  @Repository
  public interface EmployeeRepository extends JpaRepository<Employee, Long> {
      // Uses the B-Tree index to instantly jump to the ID, ignoring OFFSET entirely
      List<Employee> findByDepartmentAndIdGreaterThanOrderByIdAsc(
              String department, 
              Long lastSeenId, 
              Pageable pageable); // Pageable used ONLY for LIMIT (size), not offset
  }

  // 2. The Controller
  @GetMapping("/employees/cursor")
  public List<Employee> getEmployeesCursor(
          @RequestParam String department,
          @RequestParam(defaultValue = "0") Long lastSeenId, 
          @RequestParam(defaultValue = "10") int size) {
      
      // We only need the size (LIMIT), so we force page 0 (No OFFSET)
      Pageable limitOnly = PageRequest.of(0, size); 
      
      return employeeRepository.findByDepartmentAndIdGreaterThanOrderByIdAsc(
              department, lastSeenId, limitOnly);
  }
  ```

---

## Data Access Selection: ORM (JPA/Hibernate) vs. Bare Metal (JDBC)

### The Object-Relational Impedance Mismatch & Decision Matrix
- **The Core Problem:** Java models data through rich object graphs, references, and polymorphism. Relational databases store normalized rows and foreign keys. Bridging this mismatch is the primary goal of ORM and data mappers.
- **When to Choose JPA (Hibernate):**
  - Strongly Domain-Driven applications.
  - Complex nested object graphs (e.g., `Order` -> `LineItems` -> `Products`).
  - Standard OLTP CRUD workloads processing individual entities/aggregates.
  - Maximizes developer velocity and domain model encapsulation.
- **When to Choose JDBC / Spring JdbcTemplate:**
  - Predictable, bare-metal high-throughput execution.
  - Complex analytical reporting, aggregations, and heavy multi-table joins.
  - Massive bulk inserts/updates.
  - Situations where DBAs require exact index hints and specialized SQL.

### Hibernate Engine Mechanics (Persistence Context, Dirty Checking & Memory Traps)
- **Level 1 Cache & Dirty Checking:**
  - When entities are loaded, Hibernate stores snapshots in the Level 1 Cache (Persistence Context).
  - Mutating an entity in memory (`user.setName("Bob")`) triggers automatic update detection upon transaction commit; no explicit `.save()` invocation is needed.
- **Performance Pitfalls:**
  - **The N+1 Query Problem:** Fetching 100 entities with lazy collections can trigger 1 query for the parent records plus 100 individual queries for the children.
  - **Heap Exhaustion on Bulk Operations:** Updating 10,000 rows through JPA loads 10,000 managed objects into JVM heap memory, risking `OutOfMemoryError`.
- **JDBC Trade-off:** Zero hidden magic, full `EXPLAIN PLAN` control, and zero entity caching overhead, but requires verbose manual `ResultSet` mapping boilerplate.

### The CQRS Hybrid Architecture Pattern
- Combine the strengths of both abstractions in modern enterprise architectures:
  - **Command Path (Writes):** Use Spring Data JPA / Hibernate to execute business logic, entity state mutations, domain validation, and write persistence.
  - **Query Path (Reads):** Completely bypass JPA and use `JdbcTemplate` or JOOQ for complex reporting, dashboards, and read-heavy queries for zero-overhead performance.

---

## Composite Primary Keys (@EmbeddedId vs. @IdClass)

### @EmbeddedId Implementation Rules (Serializable, Equals/HashCode, No-Arg Constructor)
- Used when an entity is identified by a composite key (multiple columns) by encapsulating fields into a dedicated `@Embeddable` class.
- **3 Mandatory Specification Rules:**
  1. **Must implement `java.io.Serializable`:** Enables JPA providers to cache keys in second-level cache and serialize across network boundaries.
  2. **Must override `equals()` and `hashCode()`:** Required by Hibernate's Persistence Context (Level 1 Cache) to uniquely identify entity identity in memory.
  3. **Must provide a no-argument constructor:** Required for reflection-based instantiation by the persistence provider.
- **Code Implementation:**
  - **Step 1: The Embeddable Key Class:**
    ```java
    import jakarta.persistence.Embeddable;
    import java.io.Serializable;
    import java.util.Objects;

    @Embeddable
    public class EmployeeId implements Serializable {
        
        private String departmentId;
        private String employeeNumber;

        // 1. Mandatory no-arg constructor
        public EmployeeId() {}

        public EmployeeId(String departmentId, String employeeNumber) {
            this.departmentId = departmentId;
            this.employeeNumber = employeeNumber;
        }

        // Getters and Setters...

        // 2. Mandatory equals() and hashCode()
        @Override
        public boolean equals(Object o) {
            if (this == o) return true;
            if (o == null || getClass() != o.getClass()) return false;
            EmployeeId that = (EmployeeId) o;
            return Objects.equals(departmentId, that.departmentId) && 
                   Objects.equals(employeeNumber, that.employeeNumber);
        }

        @Override
        public int hashCode() {
            return Objects.hash(departmentId, employeeNumber);
        }
    }
    ```
  - **Step 2: The Entity Class:**
    ```java
    import jakarta.persistence.EmbeddedId;
    import jakarta.persistence.Entity;
    import jakarta.persistence.Table;

    @Entity
    @Table(name = "employees")
    public class Employee {

        // Inject the composite key here
        @EmbeddedId
        private EmployeeId id;

        private String name;
        private double salary;

        // Constructors, Getters, and Setters
    }
    ```

### Architectural Trade-off: @EmbeddedId vs. @IdClass
- **`@EmbeddedId` (Object-Oriented Encapsulation):**
  - *Pros:* Clean OOP design; the composite identifier is treated as a unified composite value object passed across service boundaries.
  - *Cons:* JPQL queries require accessing properties through the embedded object path (e.g., `SELECT e FROM Employee e WHERE e.id.departmentId = :deptId`).
- **`@IdClass` (Flattened Mapping):**
  - *Pros:* Key fields remain directly on the entity root, resulting in flatter JPQL queries (e.g., `SELECT e FROM Employee e WHERE e.departmentId = :deptId`).
  - *Cons:* Clutters the entity class by defining composite key fields directly alongside domain attributes.
