# Database Systems - Chapter 1: Architecture, SQL vs NoSQL & Schema Design

## Database Types and Architecture: Relational vs NoSQL

### Architectural Overview of Relational and Document Databases

Modern data architectures generally divide persistent storage into relational (SQL) and non-relational (NoSQL) paradigms. Relational Database Management Systems (RDBMS) like MySQL and PostgreSQL enforce structured schemas, mathematical set-theoretic relations, and strict ACID guarantees. Document-oriented NoSQL databases like MongoDB store semi-structured, self-describing BSON (Binary JSON) documents organized into collections rather than rows organized into tables.

| Database | Primary Architecture & Engine | Data Model | Schema Enforcement | Best Suited Workload |
| :--- | :--- | :--- | :--- | :--- |
| **MySQL** | Relational (InnoDB, MyISAM) | Structured tables, rows, columns | Strict schema defined by DDL | Read-heavy web applications, e-commerce stores, standard transactional CRUD systems |
| **PostgreSQL** | Advanced Object-Relational (ORDBMS) | Tables, custom types, JSONB, arrays | Strict schema with rich extensible typing | Complex relational models, analytical queries, financial systems, spatial data (PostGIS) |
| **MongoDB** | NoSQL Document Store (WiredTiger) | Hierarchical BSON documents in collections | Dynamic / Flexible schema (optional JSON Schema validation) | High-velocity streaming, real-time analytics, polymorphic catalogs, rapidly iterating data models |

### Workload Use-Cases and Database Selection Criteria

The selection between MySQL, PostgreSQL, and MongoDB depends strictly on the application's data access patterns, consistency requirements, query complexity, and scaling profile.

| Use-Case Area | Preferred Database | Architectural Rationale |
| :--- | :--- | :--- |
| **Strictly Structured Data & Strong Relationships** | MySQL, PostgreSQL | Native foreign key constraints, complex multi-table `JOIN` operations, declarative referential integrity, and complete ACID transaction support. |
| **Complex Queries, Advanced Analytics & Custom Data Types** | PostgreSQL | Built-in window functions, Common Table Expressions (CTEs), rich indexing extensions (GIN, GiST, BRIN), native `JSONB` with indexing, and extensibility. |
| **High Write Throughput, Unstructured / Semi-Structured Ingestion** | MongoDB | Dynamic schema avoids DDL migration locks, native horizontal sharding out of the box, and high write throughput via append-friendly WiredTiger storage engine. |
| **Rapidly Evolving and Polymorphic Schemas** | MongoDB | Document structure easily stores variable attributes per record without requiring sparse columns or extensive `ALTER TABLE` migrations. |
| **Standard Web Applications & Read-Heavy Portals** | MySQL | Extremely mature ecosystem, lightweight footprint, excellent read replica replication topologies, and widespread hosting support. |

### Indexing Mechanisms Across SQL and NoSQL Engines

* **MySQL (InnoDB Engine):**
  * Uses clustered B+Trees for primary keys where table leaf pages store the actual row data.
  * Secondary indexes store the primary key column as their row pointer, requiring a secondary index lookup followed by a clustered index seek (bookmark lookup).
* **PostgreSQL:**
  * Uses heap files for row storage where tables store unordered rows; all indexes (including the primary key index) are secondary B-trees pointing to table heap tuples via physical tuple IDs (`ctid`).
  * Supports extensible index algorithms: B-tree (default, equality/range), GIN (Generalized Inverted Index for JSONB/full-text/arrays), GiST (Generalized Search Tree for geometric/range types), and BRIN (Block Range Index for multi-gigabyte append-only timeseries data).
* **MongoDB:**
  * Uses B-Trees under the WiredTiger storage engine to index arbitrary fields and nested paths within documents (`address.zipCode`).
  * Supports compound indexes, partial indexes, TTL (Time-To-Live) indexes for automated document expiration, geospatial indexes (2dsphere), and text search indexes.

### Transaction and Concurrency Guarantees

* **MySQL (InnoDB):** Full ACID compliance supporting ANSI SQL isolation levels (`READ UNCOMMITTED`, `READ COMMITTED`, `REPEATABLE READ` [default], `SERIALIZABLE`). Implements Multi-Version Concurrency Control (MVCC) utilizing undo logs and redo logs.
* **PostgreSQL:** Complete ACID compliance with sophisticated MVCC implementation. Does not overwrite dead tuples in-place; instead, it creates new row versions tagged with transaction identifiers (`xmin`, `xmax`). Default isolation level is `READ COMMITTED`, with true Serializable Snapshot Isolation (SSI) available.
* **MongoDB:** Supported multi-document ACID transactions across replica sets since version 4.0 and across sharded clusters since version 4.2. Transactions operate at the collection and document level with snapshot isolation. However, multi-document transactions incur noticeable latency overhead compared to native RDBMS transactions, making embedded document modeling preferred for atomic operations.

---

## Object-Relational Mapping: Translating UML to Relational Architecture

### Modeling IS-A Inheritance in UML and RDBMS Table Strategies

Relational databases operate on relational algebra and flat two-dimensional tables; they have no native construct for Object-Oriented inheritance (`extends`), abstract classes, or polymorphism. Software architects map OOP inheritance hierarchies into relational schemas using standard physical modeling strategies.

```
       [UML Generalization: IS-A]
             +---------------+
             |   Employee    |
             | (id, name, ..)|
             +---------------+
                     ^
                     | (Hollow Triangle)
             +---------------+
             |    Manager    |
             | (budget, ..)  |
             +---------------+
```

#### 1. Single Table Strategy (Table-per-Hierarchy)
All classes in an inheritance hierarchy are mapped into a single, comprehensive database table.
* **Structure:** A discriminator column (e.g., `employee_type VARCHAR(20)`) specifies the concrete type of the record (`'EMPLOYEE'`, `'MANAGER'`, `'DIRECTOR'`). Columns specific to subclasses are made nullable.
* **Advantages:** Blazing fast performance; querying any subclass requires zero `JOIN` operations.
* **Disadvantages:** Violates normalization; columns specific to subclasses cannot enforce database-level `NOT NULL` constraints, resulting in sparse tables with many `NULL` values.

```sql
CREATE TABLE employees (
    id BIGINT PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    base_salary DECIMAL(10, 2) NOT NULL,
    employee_type VARCHAR(20) NOT NULL, -- Discriminator ('EMPLOYEE', 'MANAGER')
    manager_budget DECIMAL(12, 2) NULL  -- Only populated when employee_type = 'MANAGER'
);
```

#### 2. Joined Table Strategy (Table-per-Subclass)
Each class in the hierarchy maps to its own separate database table.
* **Structure:** The base table contains common columns (`id`, `name`). The subclass table contains only columns specific to that subclass, and its primary key acts simultaneously as a Foreign Key referencing the base table's primary key.
* **Advantages:** Completely normalized; eliminates sparse `NULL` columns; enables strict `NOT NULL` constraints on subclass attributes.
* **Disadvantages:** Reading subclass entities requires SQL `JOIN` operations between base and child tables, introducing query latency as hierarchies deepen.

```sql
CREATE TABLE employees (
    id BIGINT PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    base_salary DECIMAL(10, 2) NOT NULL
);

CREATE TABLE managers (
    employee_id BIGINT PRIMARY KEY,
    manager_budget DECIMAL(12, 2) NOT NULL,
    CONSTRAINT fk_managers_employee FOREIGN KEY (employee_id) REFERENCES employees(id) ON DELETE CASCADE
);
```

### Modeling HAS-A Relationships: Aggregation vs Composition in SQL Constraints

Object-oriented design uses HAS-A relationships to represent associations between independent entities. Relational databases physically represent all HAS-A associations through Foreign Key (`FK`) constraints. The distinction between Aggregation and Composition is enforced strictly through nullability and referential integrity delete actions.

```
 [Aggregation: Loose HAS-A]             [Composition: Strict HAS-A]
+------------+       +------------+     +------------+       +------------+
| Department |<>-----| Professor  |     |    Car     |*------|   Engine   |
+------------+ (Hollow+------------+    +------------+(Filled+------------+
                Diamond)                               Diamond)
```

#### 1. Translating Aggregation (Loose HAS-A)
In UML Aggregation, the child entity can exist independently of the parent (e.g., A Professor belongs to a Department, but if the Department dissolves, the Professor still exists).
* **Physical RDBMS Translation:**
  * Foreign key column in the child table is defined as **`NULL`**.
  * Foreign key constraint specifies **`ON DELETE SET NULL`** or **`ON DELETE RESTRICT`**.
  * If the parent row is deleted, the child row is safely preserved in the database with an orphaned/null foreign key reference.

```sql
CREATE TABLE professors (
    id BIGINT PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    department_id BIGINT NULL,
    CONSTRAINT fk_prof_dept FOREIGN KEY (department_id) 
        REFERENCES departments(id) ON DELETE SET NULL
);
```

#### 2. Translating Composition (Strict HAS-A)
In UML Composition, the child entity's lifecycle is bound strictly to the parent entity (e.g., A Car owns an Engine; if the Car is destroyed, the Engine ceases to exist within that context).
* **Physical RDBMS Translation:**
  * Foreign key column in the child table is defined as **`NOT NULL`**.
  * Foreign key constraint specifies **`ON DELETE CASCADE`**.
  * If the parent row is deleted, the relational engine immediately and automatically purges all associated child rows, guaranteeing no orphaned records exist.

```sql
CREATE TABLE engines (
    id BIGINT PRIMARY KEY,
    serial_number VARCHAR(50) UNIQUE NOT NULL,
    car_id BIGINT NOT NULL,
    CONSTRAINT fk_engine_car FOREIGN KEY (car_id) 
        REFERENCES cars(id) ON DELETE CASCADE
);
```

---

## Relational Schema Design Principles

### Core Principles of Scalable and Maintainable Schema Design

Designing schemas for scalable, high-concurrency enterprise applications requires balancing mathematical purity with physical storage realities:

1. **Domain Integrity & Type Constancy:** Every attribute must represent a single, atomic concept backed by the most restrictive data type possible (e.g., `VARCHAR(50)` instead of unbounded text; `DECIMAL` instead of floating-point numbers for currency).
2. **Surrogate Keys vs. Natural Keys:**
   * Natural keys (e.g., SSN, email) frequently change due to user edits, regulatory mandates, or data privacy laws.
   * Prefer immutable surrogate keys for primary identifiers (e.g., `BIGINT GENERATED ALWAYS AS IDENTITY` or ordered `UUIDv7` for distributed systems).
3. **Immutability of Audit Trails:** Core entity state changes should be preserved through audit columns (`created_at`, `updated_at`, `created_by`, `version`), append-only event ledgers, or temporal tables.
4. **Partitioning and Sharding Readiness:** Schemas expected to scale to hundreds of millions of rows must include tenant or time-based partitioning keys (e.g., `tenant_id`, `created_at`) in primary and unique compound keys from day one.

### Normalization Stages and Anomaly Prevention

Database normalization is the formal process of structuring relational tables to minimize data redundancy and eliminate operational anomalies:

* **Update Anomalies:** Inconsistent state when modifying redundant data in one row while leaving other rows unchanged.
* **Insertion Anomalies:** Inability to record valid data about an entity without artificially creating a record for an unrelated entity.
* **Deletion Anomalies:** Unintended loss of crucial entity data as a side-effect of deleting an unrelated entity record.

#### Formal Normal Forms Summary

1. **First Normal Form (1NF):**
   * Each column must contain atomic (indivisible) values.
   * No repeating groups or arrays stored in single fields.
   * A primary key uniquely identifies each row.
2. **Second Normal Form (2NF):**
   * Must satisfy 1NF.
   * Eliminates **Partial Functional Dependencies**: All non-key attributes must depend on the *entire* primary key, not a subset of a composite primary key.
3. **Third Normal Form (3NF):**
   * Must satisfy 2NF.
   * Eliminates **Transitive Functional Dependencies**: Non-key attributes must depend *directly* on the primary key, rather than through another non-key attribute ($A \rightarrow B \rightarrow C$).
4. **Boyce-Codd Normal Form (BCNF):**
   * A stricter version of 3NF where for every functional dependency $X \rightarrow Y$, $X$ must be a superkey.

### Controlled Denormalization and Performance Trade-offs

While 3NF is the standard baseline for transactional OLTP systems, strict normalization requires multi-table `JOIN` operations that degrade read throughput under heavy load.

* **When to Denormalize:** High-frequency read queries where the cost of repeated joins exceeds the cost of duplicate storage and write-time synchronization (e.g., storing `customer_name` directly in an `orders` summary record).
* **Architectural Trade-offs:**
  * **Read Latency:** Reduced dramatically due to single-table scans without joins.
  * **Write Penalty:** Increased write complexity; updates to the canonical entity require updating multiple denormalized locations (either via transactional triggers or asynchronous event listeners).
  * **Risk of Inconsistency:** If background synchronization fails, data divergence occurs.

---

## Critical Factors in Database Schema Architecture

### Architectural Factors in Schema Design

When evaluating schema design during system architecture, engineers must evaluate five core dimensions:

```
+-----------------------------------------------------------------+
|                   Schema Design Considerations                  |
+-----------------------------------------------------------------+
| 1. Read vs Write Workload Profile (OLTP vs OLAP)                |
| 2. Normalization vs Controlled Denormalization                  |
| 3. Indexing Strategy (B-Tree, Covering, Selective Filters)      |
| 4. Entity Relationship Cardinalities & Join Complexity          |
| 5. Storage Layout, Precision Data Typing & Concurrency Locking  |
+-----------------------------------------------------------------+
```

1. **Read vs. Write Workload Profile:**
   * Write-heavy systems (e.g., telemetry, ledgers) require minimal indexes, append-only tables, and lean column layouts to minimize disk I/O.
   * Read-heavy systems (e.g., product catalogs, reporting) favor pre-calculated fields, summary tables, and covering indexes.
2. **Indexing Strategy Integration:** Design column orders in composite primary keys and foreign keys to maximize B-Tree leftmost prefix traversal and index-only scans.
3. **Locking and Concurrency Strategy:** Include an integer `version` column in high-contention tables to enable non-blocking Optimistic Concurrency Control (OCC) rather than holding heavy pessimistic database row locks.

### Entity Relationship Cardinality and Join Architecture

Mapping real-world domain interactions requires translating business relationships into appropriate foreign key structures:

* **One-to-One (1:1):** Enforced by placing a foreign key on either table and adding a strict `UNIQUE` constraint on that foreign key column.
* **One-to-Many (1:N):** The child table holds a foreign key referencing the parent table's primary key. An index on this foreign key is mandatory for acceptable join and cascade performance.
* **Many-to-Many (M:N):** Relational engines cannot link multi-valued keys directly. Requires an intermediate **Junction Table** (Associative Entity) containing two foreign keys, whose combination forms the composite primary key.

```sql
-- M:N Junction Table Example
CREATE TABLE student_courses (
    student_id BIGINT NOT NULL,
    course_id BIGINT NOT NULL,
    enrolled_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    PRIMARY KEY (student_id, course_id),
    CONSTRAINT fk_sc_student FOREIGN KEY (student_id) REFERENCES students(id) ON DELETE CASCADE,
    CONSTRAINT fk_sc_course FOREIGN KEY (course_id) REFERENCES courses(id) ON DELETE CASCADE
);
```

### Data Typing, Storage Footprint, and Integrity Constraints

Database engines allocate fixed physical memory buffers (e.g., InnoDB Buffer Pool, PostgreSQL Shared Buffers). Compact schemas allow more rows to fit into RAM, reducing disk I/O.

* **Exact Numeric Precision:** Use `NUMERIC` / `DECIMAL` for financial and monetary calculations; never use IEEE 754 floating-point types (`FLOAT`, `DOUBLE`) due to rounding imprecision.
* **Text Fields:** Constrain text lengths (`VARCHAR(100)`) where business rules dictate fixed limits. Use `TEXT` only for unbounded narrative fields.
* **Timestamps:** Store timestamps in UTC using timezone-aware types (`TIMESTAMP WITH TIME ZONE` in PostgreSQL or `DATETIME` in MySQL).
* **Declarative Constraints:** Leverage `CHECK` constraints (e.g., `CHECK (status IN ('PENDING', 'ACTIVE', 'CANCELLED'))`) and `NOT NULL` constraints directly at the database engine level to safeguard data integrity against application-layer bugs.
