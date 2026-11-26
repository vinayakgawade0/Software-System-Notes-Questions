# Index: System Design — Architecture & Design

This index organizes the foundational architecture and design engineering notes and parallel interview question-and-answer sets for enterprise distributed systems, microservices, and object-oriented design.

---

## Architecture & Design Chapters

| Chapter | Topic Focus | Notes Document | Question & Answer Document |
| :---: | :--- | :--- | :--- |
| **01** | **Microservices Fundamentals & Inter-Service Communication** | [Notes - Chapter 1](./Notes%20-%20Architecture%20-%201.%20Microservices%20Fundamentals%20%26%20Inter-Service%20Communication.md) | [Questions - Chapter 1](./Questions%20-%20Architecture%20-%201.%20Microservices%20Fundamentals%20%26%20Inter-Service%20Communication.md) |
| **02** | **Security, Resilience & Traffic Management** | [Notes - Chapter 2](./Notes%20-%20Architecture%20-%202.%20Security%2C%20Resilience%20%26%20Traffic%20Management.md) | [Questions - Chapter 2](./Questions%20-%20Architecture%20-%202.%20Security%2C%20Resilience%20%26%20Traffic%20Management.md) |
| **03** | **Distributed Architecture Patterns** | [Notes - Chapter 3](./Notes%20-%20Architecture%20-%203.%20Distributed%20Architecture%20Patterns.md) | [Questions - Chapter 3](./Questions%20-%20Architecture%20-%203.%20Distributed%20Architecture%20Patterns.md) |
| **04** | **SOLID Principles & Enterprise Patterns** | [Notes - Chapter 4](./Notes%20-%20Architecture%20-%204.%20SOLID%20Principles%20%26%20Enterprise%20Patterns.md) | [Questions - Chapter 4](./Questions%20-%20Architecture%20-%204.%20SOLID%20Principles%20%26%20Enterprise%20Patterns.md) |
| **05** | **Creational & Structural Design Patterns** | [Notes - Chapter 5](./Notes%20-%20Architecture%20-%205.%20Creational%20%26%20Structural%20Design%20Patterns.md) | [Questions - Chapter 5](./Questions%20-%20Architecture%20-%205.%20Creational%20%26%20Structural%20Design%20Patterns.md) |

---

## Detailed Topic Breakdown

### [Chapter 1: Microservices Fundamentals & Inter-Service Communication](./Notes%20-%20Architecture%20-%201.%20Microservices%20Fundamentals%20%26%20Inter-Service%20Communication.md)
- **Microservices Architecture Fundamentals:** Core microservices definition, Bounded Contexts, Database-per-Service, Spring Boot support (embedded Tomcat, auto-configuration, Actuator, Spring Cloud), Strangler Fig migration pattern, banking architecture example.
- **System Scalability and State Management:** Scale Cube dimensions (X, Y, Z axes), Horizontal Pod Autoscaling (HPA), Stateless vs. Stateful services comparison, offloading state to external caches/JWTs.
- **Caching Strategies in High-Traffic Systems:** L1 JVM cache (Caffeine), L2 distributed cache (Redis Cluster), L3 CDN/Edge cache; Cache-Aside, Write-Through, Write-Behind; Cache Avalanche, Breakdown (stampede), and Penetration defenses.
- **Advantages, Disadvantages and Trade-offs:** Elasticity, fault isolation, Conway's Law alignment vs. network latency, eventual consistency, distributed tracing complexity.
- **Communication Protocols and Paradigms:** Synchronous (REST over HTTP/1.1 JSON, gRPC over HTTP/2 Protobuf) vs. Asynchronous (RabbitMQ command queues vs. Apache Kafka distributed event commit logs).
- **Thread Utilization and Modern Concurrency:** Blocking I/O vs. Non-blocking I/O event loops (Netty); Thread pool exhaustion in servlet containers; Java 21+ Virtual Threads (Project Loom); Distributed tracing (OpenTelemetry, Trace ID, Span ID).
- **Resiliency and Distributed Data Patterns:** The Temporal Coupling trap (Distributed Monolith); Transactional Outbox Pattern with CDC (Debezium); Workflow coordination: Choreography vs. Orchestration.
- **Service Evolution and Contract Management:** API versioning (URI, Header, Content Negotiation); Postel's Law / Tolerant Reader pattern; Event Schema Registry (Avro/Protobuf); Expand-Contract (Parallel Change) pattern; Consumer-Driven Contract (CDC) testing with Pact.
- **Shared Libraries and the Share Nothing Principle:** The `commons.jar` anti-pattern; Misapplication of DRY across distributed systems; Share Nothing principle; Permissible technical code vs. forbidden domain code; Standalone microservices.

### [Chapter 2: Security, Resilience & Traffic Management](./Notes%20-%20Architecture%20-%202.%20Security%2C%20Resilience%20%26%20Traffic%20Management.md)
- **Traffic Management and Denial-of-Service Defense:** Defense-in-depth model across Layer 1 (Edge Perimeter WAF/CDN - Cloudflare, AWS Shield), Layer 2 (API Gateway Traffic Control - Spring Cloud Gateway, Kong), and Layer 3 (Application Layer Self-Preservation - body size limits, pagination bounds, GraphQL depth limits, socket timeouts).
- **Distributed Rate Limiting and Throttling Algorithms:** Rate limiting vs. throttling; Algorithm comparison: Fixed Window Counter, Sliding Window Log, Sliding Window Counter, Token Bucket, Leaky Bucket; Client identification dimensions (avoiding the Corporate NAT IP trap; JWT User ID / API Key); Standard HTTP headers (`X-RateLimit-Limit`, `X-RateLimit-Remaining`, `X-RateLimit-Reset`, `Retry-After`); Distributed Redis + atomic Lua script implementation.
- **Microservices Security Architecture:** Castle-and-moat perimeter vs. Zero Trust Architecture; Mutual TLS (mTLS) with Envoy sidecar proxies and Service Mesh (Istio/Linkerd); JWT Bearer token propagation; Machine-to-Machine (M2M) OAuth2 Client Credentials grant; Kubernetes NetworkPolicies (Calico/Cilium).
- **Dynamic Service Discovery and Registration:** Elastic containers and ephemeral IP addresses; Client-side discovery (Netflix Eureka, Spring Cloud LoadBalancer) vs. Server-side discovery (Kubernetes Services, CoreDNS, kube-proxy); Liveness probes, readiness probes, heartbeat health monitoring.
- **Performance Tuning and Latency Reduction:** Connection pooling (HikariCP, persistent HTTP/2 pools); Binary gRPC / Protobuf migration; Gzip/Brotli payload compression; Non-blocking I/O and reactive pipelines.

### [Chapter 3: Distributed Architecture Patterns](./Notes%20-%20Architecture%20-%203.%20Distributed%20Architecture%20Patterns.md)
- **Gateway and Edge Routing Patterns:** The N+1 Client problem and cellular latency; API Composition; Backend-For-Frontend (BFF) pattern (Mobile BFF vs. Web BFF); Reactive aggregation with Spring WebClient `Mono.zip`; Avoiding the Gateway "God Object" anti-pattern.
- **Fault Tolerance and Systemic Resiliency Patterns:** Cascading failures and thread pool exhaustion; Circuit Breaker state machine (CLOSED, OPEN, HALF-OPEN); Spring Boot & Resilience4j implementation (`@CircuitBreaker`, configuration, fallbacks); Resiliency synergies: Retry with backoff/jitter vs. Circuit Breaker vs. Bulkhead resource isolation.
- **Distributed Transactions and Event-Driven Patterns:** Death of global 2PC/XA transactions; Database-per-Service constraints; Saga Pattern mechanics: Local ACID transactions and Compensating Transactions (business rollbacks); Choreography (Kafka event bus) vs. Orchestration (centralized state machine); Avoiding "Event Spaghetti" and orchestrator "God Services".
- **CQRS and Event Sourcing:** The CRUD bottleneck (normalized 3NF writes vs. high-throughput denormalized reads); Command Model (Write Path) vs. Query Model (Read Path); Asynchronous view projection via Kafka; Eventual Consistency trade-offs; Event Sourcing immutable commit logs.

### [Chapter 4: SOLID Principles & Enterprise Patterns](./Notes%20-%20Architecture%20-%204.%20SOLID%20Principles%20%26%20Enterprise%20Patterns.md)
- **Object-Oriented SOLID Principles:** Deep theoretical and practical analysis of Single Responsibility (SRP), Open/Closed (OCP), Liskov Substitution (LSP), Interface Segregation (ISP), and Dependency Inversion (DIP) applied to enterprise fintech systems.
- **Enterprise Java & Spring Framework Implementation:** SRP in Controller-Service-Repository layers; OCP via Spring-managed Strategy Pattern with `Map<PaymentType, PaymentStrategy>`; LSP through contract integrity and composition over inheritance; ISP through granular Spring Data repository contracts; DIP via Spring IoC and Constructor Dependency Injection for instant unit testability.
- **Enterprise Design Pattern Selection:** Practical justification matrix for Creational, Structural, Behavioral, and Distributed patterns (Strategy, Adapter, Builder, Observer, Template Method, Circuit Breaker, Outbox).

### [Chapter 5: Creational & Structural Design Patterns](./Notes%20-%20Architecture%20-%205.%20Creational%20%26%20Structural%20Design%20Patterns.md)
- **Creational Design Patterns (Singleton):** Purpose and mechanics; 6 Java implementation approaches (Eager, Lazy non-thread-safe, Synchronized method, Double-Checked Locking with mandatory `volatile`, Bill Pugh Holder idiom, Enum Singleton); 4 attack vectors (Reflection, Serialization, Cloning, ClassLoaders) and robust defenses.
- **Creational Design Patterns (Factory Method):** Interface abstraction, concrete products, parameterized factory class, decoupled client execution; centralizing complex configurations.
- **Structural Design Patterns (Adapter):** Resolving incompatible vendor APIs; Target Interface, Adaptee, Adapter; Java implementation (`PaymentProcessor` $\rightarrow$ `StripeAdapter` $\rightarrow$ `StripeService`); Anti-Corruption Layer (ACL) in Domain-Driven Design; `java.util.Arrays.asList()` and SLF4J examples.
- **Behavioral Design Patterns (Observer):** The "God Service" coupling trap; Push vs. Pull (YouTube subscription model); Spring Boot `@EventListener` with `@Async` execution; Distributed observers via Kafka topics; Reactive Streams (Project Reactor `Flux`/`Mono`).

---

## Question Linking Standards
All question documents mirror the exact `## Topic` and `### Sub-topic` section hierarchy of the parallel Notes documents. Each individual question entry is tagged with its verbatim question, direct answer, senior nuance/trade-off, and a direct Markdown anchor link pointing to the exact section in the parallel Notes document.
