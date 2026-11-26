# System Design - Chapter 1: Distributed Scale & High-Concurrency Systems

## Distributed Transactions: Two-Phase Commit vs Saga Pattern

### Two-Phase Commit Protocol Mechanics

In distributed architectures, business workflows frequently span multiple independent service databases—for example, deducting stock in an Inventory Service database while simultaneously charging a credit card and persisting ledger entries in an Order Service database. The Two-Phase Commit (2PC) protocol is an atomic commitment protocol coordinated by a centralized Transaction Manager (Coordinator) to achieve strict ACID (Atomicity, Consistency, Isolation, Durability) guarantees across heterogeneous, networked databases.

```
+-------------------------------------------------------------------+
|                     Transaction Coordinator                       |
+-------------------------------------------------------------------+
       |                                     |
  (1) Prepare?                          (1) Prepare?
       |                                     |
       v                                     v
+------------------+                   +------------------+
| Resource Mgr A   |                   | Resource Mgr B   |
| (Database A)     |                   | (Database B)     |
| [Locks Rows]     |                   | [Locks Rows]     |
+------------------+                   +------------------+
       |                                     |
  (2) Voted YES                         (2) Voted YES
       |                                     |
       +------------------+------------------+
                          |
                   All Voted YES?
                          |
             +------------+------------+
             |                         |
       (3) COMMIT                (3) COMMIT
             |                         |
             v                         v
       [Write & Release]         [Write & Release]
```

2PC divides distributed execution into two discrete phases:

1. **The Prepare Phase (Voting Phase):**
   * The centralized coordinator issues a `PREPARE` directive to every participating database node (Resource Manager).
   * Each participating database executes the transaction locally up to the point of commit: writing changes to its write-ahead redo log (WAL), validating constraints, and acquiring exclusive physical locks on all targeted rows and resources.
   * If a node successfully prepares and guarantees durability of its pending state, it replies with a `VOTE_COMMIT` (Yes). If any node encounters a constraint violation, deadlock, or internal failure, it replies with a `VOTE_ABORT` (No).

2. **The Commit Phase (Completion Phase):**
   * **Global Commit:** If and only if **100% of participants vote Yes**, the coordinator persists a commit decision in its own transaction log and broadcasts a `GLOBAL_COMMIT` command to all nodes. Each database writes the final commit record to disk, releases its row locks, and acknowledges success.
   * **Global Rollback:** If **even a single participant votes No** (or if the coordinator encounters a network timeout before receiving all votes), the coordinator broadcasts a `GLOBAL_ABORT` command. Every database uses its undo logs to revert the local changes and immediately releases all acquired locks.

---

### Limitations of Two-Phase Commit at Microservice Scale

While 2PC provides mathematical, strict transactional consistency, principal systems architects actively classify 2PC as an anti-pattern in modern cloud-native, high-throughput microservice architectures due to fundamental operational failure modes:

| 2PC Limitation | Architectural Root Cause | System Consequence |
| :--- | :--- | :--- |
| **The Blocking Bottleneck** | Synchronous, pessimistic locking spanning network boundaries. | Participating databases must hold exclusive physical locks on records for the entire duration of all remote network round trips and coordinator processing. A network hiccup or high latency on a single node forces all other nodes to queue waiting threads, causing thread pool exhaustion, latency spikes, and system-wide throughput collapse. |
| **Single Point of Failure (SPOF)** | Centralized coordinator architecture and in-doubt state. | If the coordinator crashes or experiences a network partition *after* participants have voted Yes but *before* the Commit directive is acknowledged, participants enter an **In-Doubt State**. Nodes cannot unilaterally commit or abort because they do not know the global consensus. They must hold row locks indefinitely until the coordinator recovers, freezing critical business tables. |
| **Cloud & Distributed Incompatibility** | Incompatible scaling and storage models. | Modern distributed datastores (Apache Cassandra, ScyllaDB, Amazon DynamoDB, MongoDB) and event streaming backends (Apache Kafka, RabbitMQ) do not natively implement XA/JTA 2PC protocols. Enforcing 2PC across such heterogeneous systems contradicts their partition-tolerant, append-only, and distributed consensus scaling paradigms. |

---

### The Saga Pattern and Eventual Consistency

To survive at hyperscale throughput, distributed systems discard synchronous, blocking ACID transactions across service boundaries and embrace **BASE** (Basically Available, Soft-state, Eventual consistency). The Saga pattern decomposes a monolithic, distributed transaction into an ordered sequence of rapid, independent **local transactions**.

```
Standard 2PC (Global Blocking Lock):
[Lock DB-A]-----------------[Hold Lock Across Network]----------------->[Commit DB-A & DB-B]
[Lock DB-B]-----------------[Hold Lock Across Network]----------------->[Release Locks]

Saga Pattern (Local Instant Commits + Event Passing):
[Service A: Commit Local] -> (Publish Event)
                                   |
                                   v
                             [Service B: Commit Local] -> (Publish Event)
                                                                |
                                                                v
                                                          [Service C: Commit Local]
```

#### Core Saga Execution Flow

1. **Service A (e.g., Order Service)** executes its local database transaction, writes to its local schema, commits immediately (releasing all local database locks within milliseconds), and publishes a domain event (e.g., `OrderCreated`) to a distributed message broker (Apache Kafka or RabbitMQ).
2. **Service B (e.g., Payment Service)** consumes the `OrderCreated` event, executes its local payment processing transaction, commits to its local ledger, and publishes a `PaymentAuthorized` event.
3. **Service C (e.g., Inventory Service)** consumes the event and commits inventory reservations locally.

Because each microservice interacts strictly with its private database and releases locks immediately, database connection pools remain pristine and overall throughput scales horizontally.

#### Saga Implementation Topologies: Choreography vs Orchestration

| Dimension | Choreography-Based Saga | Orchestration-Based Saga |
| :--- | :--- | :--- |
| **Coordination Model** | Decentralized, event-driven; services react to pub/sub events. | Centralized Saga Orchestrator coordinates workflow state explicitly. |
| **Coupling** | Loose coupling; services only subscribe to specific topics. | Orchestrator holds explicit dependencies on participant command APIs. |
| **Complexity Profile** | Simple for small flows (2–3 steps); becomes difficult to trace and debug as steps increase (spaghetti event cycle). | Centralized visibility and state machine definition; ideal for complex workflows (5+ steps). |
| **Failure Handling** | Each participant listens for failure events and triggers local compensation. | Orchestrator explicitly invokes compensating command endpoints on participants. |

---

### Failure Recovery via Compensating Transactions

Because each local transaction in a Saga commits permanently to its database, standard RDBMS `ROLLBACK` commands cannot revert earlier stages if a downstream service fails (e.g., Service A and Service B succeed, but Service C fails due to zero inventory).

The Saga pattern resolves this using **Compensating Transactions**—explicit, business-level reversal logic executed in reverse chronological order:

```
Forward Path:
[Service A: Reserve Order] --> [Service B: Charge Card] --> [Service C: Reserve Stock (FAILS!)]
                                                                       |
Compensating Path (Backward Rollback):                                 v
[Service A: Cancel Order]  <-- [Service B: Refund Card] <---------- [Failure Event]
```

#### Compensating Transaction Design Rules

1. **Semantic Undo vs Physical Rollback:** A compensating transaction does not erase historical records. It applies an inverse semantic action (e.g., if the forward action was `Charge $50`, the compensation is `Credit $50`; if forward was `Deduct Stock`, compensation is `Restock Items`).
2. **Idempotency Guarantee:** Network retries, duplicate event deliveries, or out-of-order execution mean compensating handlers must be strictly idempotent (e.g., utilizing unique `saga_id` and idempotent consumer tables).
3. **Guaranteed Eventual Consistency:** Compensating operations cannot be allowed to fail permanently. If a compensating transaction encounters a transient network error, the orchestrator or message broker retry policy must replay it with exponential backoff until it completes, ensuring the system eventually returns to a sound business state.

---

## Online Ticketing Architecture: High-Concurrency & Read-Heavy Scale

### Core Architectural Constraints and Domain Decoupling

Designing an enterprise online ticketing platform (e.g., BookMyShow, Ticketmaster) requires handling massive read volumes while preserving absolute consistency for financial reservations under extreme flash-sale pressure.

```
+---------------------------------------------------------------------------------------+
|                                    Client Tier                                        |
+---------------------------------------------------------------------------------------+
                                           |
                                           v
+---------------------------------------------------------------------------------------+
|                    API Gateway & Token Bucket Rate Limiter                            |
|             (Virtual Waiting Room for Surge Requests > System Capacity)               |
+---------------------------------------------------------------------------------------+
          |                                                 |
    (Read Catalog)                                    (Seat Booking)
          v                                                 v
+-----------------------------+           +-------------------------------------+
| Movie & Catalog Service     |           | Booking Service (Write-Heavy)       |
| (Read-Heavy Domain)         |           +-------------------------------------+
+-----------------------------+               |                             |
          |                                   | (1) Acquire Temporary Lock   | (2) Confirm Booking
          v                                   v                             v
+-----------------------------+   +-----------------------+     +-----------------------+
| Redis Cluster / CDN Caching |   | Redis Lock (TTL 5min) |     | PostgreSQL Master DB  |
| (Movie Metadata, Theaters)  |   | [SET lock:show:seat]  |     | [FOR UPDATE DB Lock]  |
+-----------------------------+   +-----------------------+     +-----------------------+
                                                                            |
                                                                   (3) Publish Event
                                                                            v
                                                                +-----------------------+
                                                                | Apache Kafka Topic    |
                                                                +-----------------------+
                                                                            |
                                                                            v
                                                                +-----------------------+
                                                                | Notification Service  |
                                                                +-----------------------+
```

#### Key System Constraints

* **Read-to-Write Ratio:** Heavily skewed between **100:1 and 1000:1**. Millions of concurrent users browse movie schedules, view cinema layouts, and inspect real-time seat availability, but only a tiny fraction proceed to checkout.
* **CAP Theorem Trade-off:**
  * **Browsing Domain (Catalog & Search):** Prioritizes **High Availability (AP)**. Serving slightly stale movie details or delayed seat map updates is acceptable to keep the browsing experience instantaneous.
  * **Booking Domain (Transactions & Checkout):** Strictly enforces **Strong Consistency (CP / ACID)**. Two users must never be allowed to book the exact same physical seat.

#### Decoupled Microservice Boundaries

1. **Movie & Catalog Service (Read-Heavy):** Manages cities, theaters, screening schedules, and movie metadata. Completely shielded from relational databases by sitting behind an edge CDN and a Redis distributed cache.
2. **Booking Service (Write-Heavy):** Coordinates seat reservations, inventory locking, and order status transitions.
3. **Payment Service:** Interfaces with external payment gateways (Stripe, Razorpay) via the Adapter pattern, supporting asynchronous webhook confirmations.
4. **Notification Service:** An asynchronous event consumer subscribed to Kafka topics (e.g., `TicketConfirmed`), dispatching SMS, WhatsApp, and email tickets without adding latency to the user HTTP request path.

---

### Relational Schema Design for Strict Booking Integrity

While catalog browsing can leverage document or key-value datastores, the core booking engine requires a Relational Database Management System (PostgreSQL or MySQL InnoDB). Relational engines provide mature ACID guarantees, row-level locking, foreign key integrity, and transactional isolation required for financial commerce.

#### Core Entity Relational Model

```sql
-- Core Schema DDL for Online Ticketing Engine

CREATE TABLE Theater (
    id BIGSERIAL PRIMARY KEY,
    city_id INT NOT NULL,
    name VARCHAR(255) NOT NULL,
    address TEXT NOT NULL
);

CREATE TABLE Screen (
    id BIGSERIAL PRIMARY KEY,
    theater_id BIGINT NOT NULL REFERENCES Theater(id),
    name VARCHAR(50) NOT NULL
);

CREATE TABLE Movie (
    id BIGSERIAL PRIMARY KEY,
    title VARCHAR(255) NOT NULL,
    duration_minutes INT NOT NULL,
    release_date DATE NOT NULL
);

CREATE TABLE Show (
    id BIGSERIAL PRIMARY KEY,
    movie_id BIGINT NOT NULL REFERENCES Movie(id),
    screen_id BIGINT NOT NULL REFERENCES Screen(id),
    start_time TIMESTAMP WITH TIME ZONE NOT NULL,
    end_time TIMESTAMP WITH TIME ZONE NOT NULL
);

CREATE TABLE Seat (
    id BIGSERIAL PRIMARY KEY,
    screen_id BIGINT NOT NULL REFERENCES Screen(id),
    row_label VARCHAR(5) NOT NULL,
    seat_number INT NOT NULL,
    seat_type VARCHAR(20) NOT NULL DEFAULT 'STANDARD',
    CONSTRAINT uq_screen_seat UNIQUE (screen_id, row_label, seat_number)
);

CREATE TABLE Booking (
    id UUID PRIMARY KEY,
    user_id BIGINT NOT NULL,
    show_id BIGINT NOT NULL REFERENCES Show(id),
    total_amount DECIMAL(10,2) NOT NULL,
    status VARCHAR(20) NOT NULL CHECK (status IN ('PENDING', 'BOOKED', 'FAILED', 'EXPIRED')),
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE Show_Seat_Mapping (
    id BIGSERIAL PRIMARY KEY,
    show_id BIGINT NOT NULL REFERENCES Show(id),
    seat_id BIGINT NOT NULL REFERENCES Seat(id),
    booking_id UUID REFERENCES Booking(id),
    status VARCHAR(20) NOT NULL CHECK (status IN ('AVAILABLE', 'LOCKED', 'BOOKED')),
    version INT NOT NULL DEFAULT 0,
    CONSTRAINT uq_show_seat UNIQUE (show_id, seat_id)
);

CREATE INDEX idx_show_seat_lookup ON Show_Seat_Mapping (show_id, status);
```

---

### Distributed Concurrency Control and Double-Booking Prevention

During high-demand flash sales (e.g., blockbuster premiere tickets), upwards of 50,000 users may concurrently attempt to select and book the exact same high-demand seats. Executing simultaneous relational transactions for every click would instantly exhaust database connection pools and induce deadlocks.

To solve this, the architecture utilizes a **Two-Tier Locking Strategy**:

```
User Click
    |
    v
+-------------------------------------------------------------+
| Phase 1: Temporary Lock (Redis Distributed Lock)            |
| Key: lock:show_{show_id}:seat_{seat_id}                     |
| TTL: 300 Seconds (5 Minutes)                                |
+-------------------------------------------------------------+
    |                                    |
(Lock Acquired)                   (Key Already Exists)
    |                                    |
    v                                    v
Proceed to Checkout               Fail Instantly: Return HTTP 409
    |                             (Seat Temporarily Reserved by Another User)
    |                             *Zero Database Load*
    v
+-------------------------------------------------------------+
| Phase 2: Database Hard Lock (PostgreSQL Row-Level Lock)     |
| SELECT status FROM Show_Seat_Mapping ... FOR UPDATE         |
| Status == AVAILABLE -> Set BOOKED -> COMMIT                 |
+-------------------------------------------------------------+
```

#### Phase 1: Temporary Distributed Lock via Redis TTL

When a customer selects a seat on the interactive map:
1. The Booking Service executes an atomic distributed lock in Redis:
   ```bash
   SET lock:show_101:seat_50 "user_89432" NX EX 300
   ```
   * `NX`: Sets the key only if it does not already exist.
   * `EX 300`: Sets an automatic Time-To-Live expiration of 300 seconds (5 minutes).
2. **Lock Success:** User A successfully claims the 5-minute checkout window.
3. **Lock Contention:** If User B clicks the same seat one millisecond later, Redis rejects the operation in $O(1)$ time. User B immediately receives a `"Seat Temporarily Reserved"` response without generating any read or write traffic against the PostgreSQL database.
4. **Auto-Recovery:** If User A abandons their shopping cart or their payment times out, the Redis key automatically expires after 300 seconds, releasing the seat back to the general pool with zero manual administrative overhead.

#### Phase 2: Database Hard Lock via Pessimistic Row Locking

Because in-memory distributed caches are susceptible to cluster failover, network blips, or rare split-brain states, final financial authorization must be guarded by relational engine locks.

When User A submits payment confirmation:
```sql
BEGIN TRANSACTION;

-- Pessimistic locking of the target show seat
SELECT status 
FROM Show_Seat_Mapping
WHERE show_id = 101 AND seat_id = 50
FOR UPDATE;

-- Application checks if status is still AVAILABLE / LOCKED by this user
-- If valid:
UPDATE Show_Seat_Mapping
SET status = 'BOOKED',
    booking_id = 'c4a8d09e-76e3-49b8-93a1-12f5e3d7a8b1',
    version = version + 1
WHERE show_id = 101 AND seat_id = 50;

UPDATE Booking
SET status = 'BOOKED',
    updated_at = CURRENT_TIMESTAMP
WHERE id = 'c4a8d09e-76e3-49b8-93a1-12f5e3d7a8b1';

COMMIT;
```

* `SELECT ... FOR UPDATE` acquires an exclusive row lock (`X-lock`).
* If concurrent transactions attempt to execute the same query on identical rows, the RDBMS engine serializes them. The winning transaction commits the `BOOKED` state; subsequent serialized transactions read the updated state and immediately abort, mathematically eliminating double-booking anomalies.

---

### High-Throughput Read Caching and Surge Traffic Management

During blockbuster ticket launches, rendering interactive seat maps via relational queries (`SELECT ... FROM Show_Seat_Mapping WHERE show_id = ?`) degrades throughput.

1. **Seat Map Caching via In-Memory Bitmasks / JSON:**
   * Seat maps for active screenings are pre-materialized directly in Redis using compact bitmasks or serialized JSON objects.
   * Client browsers poll the lightweight Redis cache every 2 seconds to refresh seat availability colors (green/gray). Relational databases are never touched during seat map rendering.
2. **Virtual Waiting Rooms (Traffic Shedding):**
   * If incoming request volume exceeds maximum downstream system capacity (e.g., > 1,000,000 concurrent sessions), the API Gateway activates a **Token Bucket Rate Limiter**.
   * Excess requests are redirected to a **Virtual Waiting Room** queue managed via Redis Sorted Sets or edge CDN workers (e.g., Cloudflare Waiting Room). Users are admitted into the checkout funnel at a strictly regulated drip rate, preventing cascading database collapse.

---

## URL Shortener Architecture: Algorithmic Design & Distributed Storage

### Core Requirements and Capacity Estimation

A distributed URL shortening system (e.g., Bitly, TinyURL) provides short aliases that transparently redirect HTTP clients to original, long URLs.

#### System Requirements

* **Traffic Profile:** Radically read-heavy. The read-to-write ratio is typically **100:1**.
* **Latency Guarantee:** Redirect operations must execute with sub-10 millisecond latency. Any redirect lag directly degrades user browsing experience.
* **Scale & Capacity Projections:**
  * **Write Ingestion:** 100 million new URLs generated per month.
  * **Retention Period:** 10 years minimum persistence.
  * **Total Record Capacity:**
    $$\text{Total Records} = 100{,}000{,}000 \times 12 \times 10 = 12{,}000{,}000{,}000 \text{ (12 Billion URLs)}$$
  * **Storage Footprint:** Assuming an average record size of 500 bytes (URL, hash, timestamps, user ID):
    $$\text{Total Storage} = 12 \times 10^9 \times 500 \text{ bytes} \approx 6 \text{ Terabytes}$$

---

### Base62 Encoding Algorithmic Core

To convert numeric identifiers into human-readable, compact short links (e.g., `https://bit.ly/3x8aB9`), the system uses **Base62 Encoding**.

```
Base62 Character Alphabet:
Index  0 - 25:  a - z  (26 lowercase characters)
Index 26 - 51:  A - Z  (26 uppercase characters)
Index 52 - 61:  0 - 9  (10 numeric characters)
Total Radix  :  62 characters
```

#### Capacity of a 7-Character Token

If short tokens are fixed at exactly **7 characters**:
$$\text{Combinations} = 62^7 = 3{,}521{,}614{,}606{,}208 \approx 3.52 \text{ Trillion Unique URLs}$$

3.52 trillion available unique tokens easily accommodates 12 billion URLs over a 10-year span, utilizing less than 0.35% of the total address space.

#### Bijective Base62 Algorithm

```java
public class Base62Encoder {
    private static final String BASE62 = "abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789";
    private static final int BASE = BASE62.length();

    public static String encode(long id) {
        StringBuilder sb = new StringBuilder();
        if (id == 0) return String.valueOf(BASE62.charAt(0));
        while (id > 0) {
            sb.append(BASE62.charAt((int) (id % BASE)));
            id /= BASE;
        }
        return sb.reverse().toString();
    }

    public static long decode(String str) {
        long id = 0;
        for (char c : str.toCharArray()) {
            id = id * BASE + BASE62.indexOf(c);
        }
        return id;
    }
}
```

---

### Collision Prevention: Key Generation Service vs Hashing

A critical design challenge is ensuring that when long URLs are registered, generated short aliases are universally unique and collision-free.

```
+---------------------------------------------------------------------------------------+
| Approach 1: Hash Truncation (MD5 / SHA-256) [ANTI-PATTERN]                            |
| Long URL ---> [ MD5 ] ---> Truncate first 7 chars ---> Collision Guaranteed!          |
| * Requires constant DB lookups (SELECT WHERE short_url = ?) and retry loops.          |
+---------------------------------------------------------------------------------------+

+---------------------------------------------------------------------------------------+
| Approach 2: Key Generation Service (KGS) [ENTERPRISE STANDARD]                         |
|                                                                                       |
| +-------------------------+       Pre-generate 7-char Base62 keys                     |
| | Offline KGS Daemon      | ------------------------------------+                     |
| +-------------------------+                                     |                     |
|                                                                 v                     |
|                                                   +---------------------------+       |
|                                                   | Unused Keys Table (RAM/DB)|       |
|                                                   +---------------------------+       |
|                                                                 |                     |
| User Shorten Request ---> [ API Server ] <----------------------+                     |
|                           Assigns key instantly (Zero Latency, Zero Collisions)       |
+---------------------------------------------------------------------------------------+
```

#### The Hashing Approach and Why It Fails

* **Mechanism:** Hash the long URL using cryptographic algorithms (MD5, SHA-256) and truncate the output to the first 7 characters.
* **Why It Fails:** By Pigeonhole Principle, truncating a 128-bit hash to 7 alphanumeric characters ($62^7$) guarantees hash collisions at scale.
* **Operational Penalty:** To handle collisions, the ingestion worker must query the primary database (`SELECT * FROM urls WHERE short_url = ?`). If a collision occurs, it must append a counter, re-hash, and repeat. Under high write concurrency, this query loop produces severe read amplification and unacceptable write latency.

#### The Key Generation Service (KGS) Solution

* **Mechanism:** Key generation is completely decoupled from incoming user requests. A standalone background service (KGS) generates random or sequential 7-character Base62 keys offline and inserts them into an `Unused_Keys` datastore.
* **Concurrency Handling:** To avoid database write contention during runtime URL ingestion, KGS pre-loads batches of keys (e.g., 5,000 keys) into the local memory of each application server instance.
* **Zero-Collision Guarantee:** When a user requests a short URL, the server pops an unused key from its in-memory buffer, maps it to the long URL, and commits. The write latency is instantaneous, and collisions are mathematically eliminated.
* **Alternative (Distributed ID Generators):** Alternatively, systems can use an algorithmic 64-bit ID generator (Twitter Snowflake) and encode the resulting numeric ID directly into Base62.

---

### Distributed NoSQL Storage and Read Caching Topology

#### Datastore Selection: NoSQL Key-Value / Wide-Column

A URL shortener does not require relational `JOIN` operations, transactions across records, or complex foreign key graphs. The workload consists entirely of point lookups:
$$\text{GET } /3x8aB9 \implies \text{Long URL}$$

A distributed wide-column or key-value datastore (Apache Cassandra, Amazon DynamoDB, or ScyllaDB) is the ideal persistence layer:
* Unbounded horizontal scaling across sharded clusters.
* Predictable single-digit millisecond read/write latency.
* High partition tolerance without relational locking overhead.

#### Cassandra / DynamoDB Schema

```sql
CREATE TABLE short_urls (
    short_hash VARCHAR,
    original_url VARCHAR,
    created_at TIMESTAMP,
    user_id VARCHAR,
    click_count COUNTER,
    PRIMARY KEY (short_hash)
);
```

#### Multi-Tier Caching Topology

```
Client GET /3x8aB9
       |
       v
+---------------+      Cache Hit (Sub-millisecond)
| Redis Cluster | ---------------------------------> Return HTTP 302 Redirect
+---------------+
       |
  Cache Miss
       v
+---------------+
| Cassandra DB  |
+---------------+
       |
       +-----> Populate Redis (LRU Eviction) ---> Return HTTP 302 Redirect
```

* **Caching Strategy:** Implements **Cache-Aside**. Incoming short hash requests hit a distributed Redis cluster first.
* **Eviction Policy:** Configured with **Least Recently Used (LRU)** eviction.
* **Pareto Principle (80/20 Rule):** In URL shortening workloads, 80% of redirect requests hit only 20% of trending or active URLs (e.g., viral social media campaigns). Storing the top 20% of hot links in RAM guarantees a cache hit ratio exceeding 85%, shielding the database tier from billions of daily queries.

---

### HTTP Redirection Strategy: 301 vs 302 Trade-offs

When returning redirect responses to client web browsers, architects must carefully choose the HTTP status code based on infrastructure capacity and business monetization models:

| Dimension | HTTP 301 Moved Permanently | HTTP 302 Found (Temporary Redirect) |
| :--- | :--- | :--- |
| **Browser Caching Behavior** | The client browser caches the redirect mapping permanently in local disk cache. | The client browser does not cache the redirect mapping. |
| **Subsequent Request Path** | Future clicks navigate directly from the browser to the destination URL without contacting the shortener backend. | Every single click must send an HTTP request to the shortener backend to resolve the destination. |
| **Server Load Impact** | **Extremely Low.** Drastically reduces server, bandwidth, and database load. | **High.** The shortener infrastructure must process 100% of all redirect traffic. |
| **Analytics & Monetization** | **Destroyed.** Because requests bypass the server, the platform cannot track click analytics, referrers, geographic origins, or device telemetry. | **Preserved.** Complete, real-time telemetry on every click, enabling monetization, link expiration enforcement, and dynamic routing. |
| **Architectural Verdict** | Used only when infrastructure cost reduction overrides analytics needs. | **Standard industry choice for commercial shorteners** (e.g., Bitly) whose core business model depends on analytics. |
