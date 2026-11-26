# Database Systems - Questions - Chapter 1: Architecture, SQL vs NoSQL & Schema Design

## Database Types and Architecture: Relational vs NoSQL

### What are the key differences between relational databases such as MySQL and PostgreSQL and NoSQL databases such as MongoDB, and when should each be used?

> **Direct Answer:** Relational databases (MySQL, PostgreSQL) enforce structured schemas, mathematical relations, multi-table `JOIN` operations, and strict ACID guarantees, making them ideal for systems requiring high data integrity and structured relationships. MongoDB is a document-oriented NoSQL database that stores flexible, hierarchical BSON documents, optimized for high-throughput unstructured or polymorphic data and native horizontal sharding. Choose relational databases for complex transactional and relational domains (e-commerce, banking) and MongoDB for dynamic schemas, rapid feature iteration, and write-heavy telemetry or content management.

> **Key Nuance / Trade-off:** While MongoDB introduced multi-document ACID transactions in version 4.0, they incur substantial performance and latency overhead compared to native RDBMS transactions. Conversely, modern RDBMS engines like PostgreSQL support native indexed `JSONB` columns, closing the flexibility gap and allowing hybrid relational/document workloads within a single ACID-compliant system.

> **Reference Concept:** `Architectural Overview of Relational and Document Databases`

---

## Object-Relational Mapping: Translating UML to Relational Architecture

### Explain how IS-A and HAS-A relationships are modeled visually in UML, and how they are physically translated into a Relational Database (RDBMS) architecture.

> **Direct Answer:** In UML, an IS-A inheritance relationship is represented by a solid line with a hollow triangle pointing to the parent, which RDBMS translates using either a Single Table Strategy (a single wide table with nullable columns and a discriminator column) or a Joined Table Strategy (separate subclass tables sharing the primary key as a foreign key). A HAS-A relationship represents object association and maps directly to SQL Foreign Keys. Loose HAS-A (Aggregation, hollow diamond) translates to a nullable foreign key with `ON DELETE SET NULL`, while strict HAS-A (Composition, filled diamond) translates to a non-nullable foreign key with `ON DELETE CASCADE`.

> **Key Nuance / Trade-off:** The Single Table strategy achieves maximum query performance because subclass queries require zero `JOIN` operations, but it produces sparse tables and prevents enforcing database-level `NOT NULL` constraints on subclass fields. The Joined Table strategy enforces strict normalization and data integrity, but incurs severe query latency on deep class hierarchies due to required multi-table joins.

> **Reference Concept:** `Modeling IS-A Inheritance in UML and RDBMS Table Strategies`

---

## Relational Schema Design Principles

### What are the key principles of database schema design and how do you design schemas for scalable and maintainable applications?

> **Direct Answer:** Scalable schema design is governed by domain integrity, strict data typing, surrogate primary keys, normalization up to 3NF, and forward-looking sharding/partitioning readiness. Maintainable applications isolate business identity from physical storage using immutable surrogate identifiers (`BIGINT` identity or ordered `UUIDv7`), establish clear naming conventions, and embed audit trails (`created_at`, `version`) to track record evolution. For scale, schemas isolate high-concurrency write tables from heavy reporting queries, leveraging controlled denormalization only where join overhead empirically degrades read throughput.

> **Key Nuance / Trade-off:** Over-normalizing can cripple performance in high-throughput read workloads due to excessive `JOIN` costs across dozens of tables. Conversely, premature denormalization creates data redundancy and risks severe update anomalies; architects must keep OLTP schemas normalized at 3NF and delegate analytical aggregations to read replicas, materialized views, or dedicated caching layers.

> **Reference Concept:** `Core Principles of Scalable and Maintainable Schema Design`

---

## Critical Factors in Database Schema Architecture

### What factors should be considered while designing database schemas such as normalization, indexing strategy, and relationships between entities?

> **Direct Answer:** Designing robust schemas requires balancing normalization level (eliminating update/insert anomalies while evaluating read join costs), indexing strategy (indexing high-cardinality search columns while minimizing the write-time rebalancing penalty), and relationship cardinality (1:1, 1:N, or M:N junction tables). Architects must also carefully evaluate storage data types to ensure minimal RAM footprint in buffer pools, define explicit referential integrity rules (`ON DELETE CASCADE` vs `SET NULL`), and implement concurrency mechanisms such as optimistic locking version columns.

> **Key Nuance / Trade-off:** Every foreign key and index created has an immediate write amplification penalty: every `INSERT`, `UPDATE`, and `DELETE` must maintain internal B-Trees and verify foreign key constraint locks across multiple tables. Schema designers must optimize strictly for the primary workload profile (OLTP vs OLAP) rather than indexing every searchable field.

> **Reference Concept:** `Architectural Factors in Schema Design`
