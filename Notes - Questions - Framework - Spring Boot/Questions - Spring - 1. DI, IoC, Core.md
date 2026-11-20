# Spring Core, Dependency Injection & Bean Lifecycle - Interview Q&A

### Explain Spring Boot 2.x.x / 3.x.x ecosystem difference

> **Direct Answer:**
> - **Java 17 Baseline:** Spring Boot 3 raised the minimum Java runtime baseline to Java 17 (supporting Java 21 LTS), leveraging modern language features and JVM optimizations.
> - **Jakarta EE Namespace Migration:** The most critical breaking change is the complete transition from Java EE (`javax.*`) to Jakarta EE (`jakarta.*`) packages across Servlets, Persistence, and Validation.
> - **Native Images & Lightweight Threads:** Introduced production-ready GraalVM Ahead-Of-Time (AOT) compilation for near-instant native startup and low memory usage, alongside native support for Virtual Threads (Project Loom) in Spring Boot 3.2+.
>
> **Key Nuance / Trade-off:**
> - Migrating from Boot 2 to Boot 3 breaks third-party libraries still relying on `javax.*` packages, requiring simultaneous upgrades of dependencies (e.g., Hibernate 6, Jackson 2.14+, and Security 6).
>
> **Reference Concept:** `Spring Boot 2.x vs. 3.x Ecosystem Modernization`

---

### Compare Tomcat and Netty as embedded servers in Spring Boot. What are their architectural differences, and when would you choose one over the other?

> **Direct Answer:**
> - **Architectural Concurrency:** Apache Tomcat (`starter-web`) uses a blocking Thread-Per-Request model with a pool of ~200 worker threads. Netty (`starter-webflux`) uses an asynchronous, non-blocking Event Loop architecture driven by a small, fixed thread pool (~2x CPU cores).
> - **Execution Trade-offs:** Tomcat suspends threads during I/O waits, which can exhaust thread pools under high concurrent traffic. Netty threads never wait; they register OS interrupt callbacks (e.g., `epoll`) and immediately switch to process other requests.
>
> **Key Nuance / Trade-off:**
> - **The Netty Blocking Trap:** Netty requires a 100% non-blocking ecosystem. Calling any blocking library (traditional JDBC, `RestTemplate`, blocking SDKs) on a Netty thread halts the entire event loop, catastrophically degrading application throughput.
>
> **Reference Concept:** `Embedded Servers: Tomcat vs. Netty`

---

### What is the difference between Spring and Spring Boot, and when would you choose one over the other?

> **Direct Answer:**
> - **Core Engine vs. Opinionated Platform:** Spring is the foundational, unopinionated framework providing core IoC, DI, and declarative transactions. Spring Boot is an opinionated framework built on top of Spring that eliminates boilerplate setup.
> - **The Three Superpowers of Boot:** Boot eliminates manual XML/Java bean wiring via Auto-Configuration, harmonizes dependency versions via Starter POMs, and packages embedded web servers (Tomcat) into standalone executable JARs.
> - **Selection Strategy:** Choose plain Spring only when maintaining legacy deployments running inside external application servers (WebLogic/WebSphere). Choose Spring Boot for all modern microservices and web APIs.
>
> **Key Nuance / Trade-off:**
> - Spring Boot does not replace Spring Framework; it wraps and auto-configures it, accelerating development velocity without sacrificing underlying Spring extensibility.
>
> **Reference Concept:** `Spring Framework vs. Spring Boot Architecture`

---

### You need to log events in your application instead of using System.out.println(). How do you do it?

> **Direct Answer:**
> - **SLF4J Facade & Logback Engine:** Use SLF4J as the logging abstraction backed by Logback, the production standard bundled with Spring Boot.
> - **Lombok `@Slf4j`:** Annotate service classes with Lombok's `@Slf4j` to automatically inject a static `log` instance without manual factory boilerplate.
> - **Parameterized Logging:** Always use `{}` parameter placeholders (e.g., `log.info("Processing order: {}", orderId)`) rather than string concatenation.
>
> **Key Nuance / Trade-off:**
> - `System.out.println()` synchronously blocks the thread during console I/O, cannot be filtered by level, and wastes memory. String concatenation forces the JVM to allocate heap strings even when the target log level is disabled, whereas `{}` placeholders evaluate arguments only if the level is active.
>
> **Reference Concept:** `Observability: Logging Architecture & Actuator Security`

---

### What is Spring Boot Actuator and what is the critical security flaw developers make when deploying it to production?

> **Direct Answer:**
> - **Operational Observability:** Actuator provides built-in production endpoints for inspecting application health, metrics, environment variables, thread dumps, and HTTP traces.
> - **The Critical Security Flaw:** Setting `management.endpoints.web.exposure.include=*` on public internet endpoints allows unauthorized users to access `/actuator/env` (exposing plaintext database passwords/tokens) and `/actuator/heapdump` (allowing memory dumps to extract active JWTs).
> - **Hardening Architecture:** Segregate Actuator to an internal, firewalled port (`management.server.port=8081`) or protect management paths with strict role-based access in `SecurityFilterChain`.
>
> **Key Nuance / Trade-off:**
> - Exposing only `/health` publicly is safe, but sensitive operational endpoints must never be reachable outside private cluster networks.
>
> **Reference Concept:** `Observability: Logging Architecture & Actuator Security`

---

### How does actuator help in production?

> **Direct Answer:**
> - **Infrastructure Orchestration:** Acts as the communication bridge between the JVM application and deployment infrastructure (Kubernetes, AWS ECS).
> - **Automated Healing (Liveness & Readiness):** Probes inform Kubernetes whether the pod is healthy or should stop receiving traffic (Readiness) without killing the container.
> - **Metric Scraping:** Exposes `/actuator/prometheus` to export JVM memory allocations, GC pause durations, and HTTP latency metrics directly to Prometheus and Grafana.
> - **Live Runtime Debugging:** The `/actuator/loggers` endpoint allows dynamically changing a package's logging level (e.g., `INFO` to `DEBUG`) via HTTP POST without restarting the server.
>
> **Key Nuance / Trade-off:**
> - Failing a Readiness probe temporarily removes the pod from the load balancer pool, preventing user-facing errors while allowing dependencies or caches to recover.
>
> **Reference Concept:** `Observability: Logging Architecture & Actuator Security`

---

### How do you configure Spring Boot Actuator, which endpoints are the most critical, and how do you write custom health checks?

> **Direct Answer:**
> - **Configuration:** Add `spring-boot-starter-actuator` and explicitly specify exposed endpoints in YAML (`management.endpoints.web.exposure.include=health,metrics,prometheus`).
> - **Critical Endpoints:** The primary operational endpoints are `/health`, `/metrics`, `/prometheus`, and `/env` (internal only).
> - **Custom Health Indicators:** Implement the `HealthIndicator` interface, override `health()`, ping external downstream dependencies, and return `Health.up()` or `Health.down().withDetail(...)`.
>
> **Key Nuance / Trade-off:**
> - If a custom `HealthIndicator` returns `Health.down()`, Spring dynamically flips the aggregate `/actuator/health` status to DOWN, signalling Kubernetes to pause routing traffic to that instance.
>
> **Reference Concept:** `Observability: Logging Architecture & Actuator Security`

---

### What is Dependency Injection (DI), and how does it relate to IoC?

> **Direct Answer:**
> - **Tactic vs. Strategy:** Inversion of Control (IoC) is the overarching architectural design principle, whereas Dependency Injection (DI) is the concrete physical mechanism used to achieve it.
> - **The Physical Process:** The IoC container creates bean instances and physically injects required dependencies into the dependent bean at runtime.
> - **The Analogy:** IoC dictates that a surgeon should not manufacture their own scalpel; Dependency Injection is the physical act of the nurse placing the scalpel into the surgeon's hand.
>
> **Key Nuance / Trade-off:**
> - DI shifts object lifecycle control away from individual business classes to the container, enabling true loose coupling and simplified mocking during testing.
>
> **Reference Concept:** `Dependency Injection (DI) Mechanics & Benefits`

---

### Explain dependency injection. Why is it needed?

> **Direct Answer:**
> - **Externalized Provisioning:** Dependency Injection provides an object with its collaborating dependencies from an external assembler (Spring IoC Container) rather than having the object construct them via `new`.
> - **Loose Coupling:** Eliminates hardcoded dependencies (e.g., `OrderService` depending directly on `StripePaymentProcessor`), allowing alternate implementations to be swapped globally without modifying business code.
> - **Testability:** Makes it trivial to pass mock implementations (e.g., `Mockito.mock(PaymentProcessor.class)`) into constructors during unit testing, isolating domain logic from databases and networks.
>
> **Key Nuance / Trade-off:**
> - Hardcoding dependencies with `new` permanently locks classes to specific concrete implementations, creating fragile architectures that cannot be unit-tested in isolation.
>
> **Reference Concept:** `Dependency Injection (DI) Mechanics & Benefits`

---

### What are the types of Dependency Injection in Spring?

> **Direct Answer:**
> - **Constructor Injection (Enterprise Standard):** Dependencies are supplied via class constructors. Allows fields to be declared `final` (ensuring immutability) and guarantees objects cannot be created in incomplete states.
> - **Setter Injection:** Dependencies are injected through `@Autowired` setter methods. Useful for optional dependencies, but leaves fields mutable.
> - **Field Injection (Anti-Pattern):** Placing `@Autowired` directly on private fields. Uses reflection, breaks encapsulation, prevents `final` modifiers, and hinders pure unit testing without a Spring container.
>
> **Key Nuance / Trade-off:**
> - Field injection hides dependencies from the class contract, whereas Constructor injection makes all mandatory dependencies explicit at compile time.
>
> **Reference Concept:** `Dependency Injection Types (Constructor, Setter, Field)`

---

### What is Circular Dependency? Why does it happen and how do you fix it? (@Lazy, setter injection, redesign)

> **Direct Answer:**
> - **The Cycle:** Occurs when Bean A requires Bean B, while Bean B simultaneously requires Bean A during constructor instantiation.
> - **The Crash:** The IoC container enters an infinite initialization loop and throws a `BeanCurrentlyInCreationException` at startup.
> - **Architectural Redesign (Best Practice):** Break the cycle by extracting the shared logic into a distinct, third service (e.g., `ValidationService`) so both services depend downward.
> - **Alternative Workarounds:** Use `@Lazy` on one constructor parameter to inject a proxy shell, or use setter injection (though Spring Boot 2.6+ disables circular references by default).
>
> **Key Nuance / Trade-off:**
> - Circular dependencies almost always indicate an architectural flaw violating the Single Responsibility Principle; fixing the domain model is always preferred over masking the issue with `@Lazy`.
>
> **Reference Concept:** `Circular Dependencies: Causes, Pitfalls & Resolution (@Lazy, Redesign)`

---

### What is the exact relationship between Inversion of Control (IoC) and Dependency Injection (DI), and how did Java developers build applications before Spring existed?

> **Direct Answer:**
> - **Philosophy vs. Mechanism:** IoC is the architectural pattern that inverts object creation control, while DI is the primary mechanism implementing IoC in modern software.
> - **Historical Pre-Spring Era:** Developers instantiated classes directly with `new` or performed complex, slow JNDI lookups inside heavy enterprise application servers (WebLogic/WebSphere).
> - **The Spring Revolution:** Spring inverted this model by introducing POJOs (Plain Old Java Objects) that passively declare dependencies, allowing lightning-fast unit testing via mock injection without booting heavyweight application servers.
>
> **Key Nuance / Trade-off:**
> - Pre-Spring applications actively hunted for dependencies, binding code tightly to server runtime environments; Spring POJOs remain completely decoupled from the container.
>
> **Reference Concept:** `Inversion of Control (IoC) Architectural Philosophy`

---

### What is Inversion of Control (IoC) and what is the IoC Container?

> **Direct Answer:**
> - **Inversion Principle:** A software design principle where execution control and object lifecycle management are transferred from custom application code to an external container.
> - **The IoC Container (`ApplicationContext`):** The software engine in Spring responsible for scanning configuration annotations, instantiating beans, injecting dependencies, managing lifecycles, and configuring proxies.
> - **The Analogy:** Cooking a meal yourself by sourcing every ingredient represents traditional programming; ordering at a restaurant where the kitchen handles preparation and delivers the dish represents IoC.
>
> **Key Nuance / Trade-off:**
> - The IoC container acts as both an object factory and an application lifecycle orchestrator, ensuring beans are created in the correct dependency order.
>
> **Reference Concept:** `Inversion of Control (IoC) Architectural Philosophy`

---

### What is the difference between a normal Java Bean and a Spring Bean

> **Direct Answer:**
> - **Ownership & Lifecycle:** A normal Java Bean is created and managed directly by the developer using `new` and collected by the garbage collector when dereferenced.
> - **Container-Managed Spring Bean:** A Spring Bean is created, wired, configured, and managed by the Spring IoC Container (`ApplicationContext`).
> - **Enhanced Capabilities:** Because Spring manages the bean, it can participate in lifecycle hooks (`@PostConstruct`, `@PreDestroy`), property injection, and runtime AOP proxying (`@Transactional`).
>
> **Key Nuance / Trade-off:**
> - Objects instantiated via `new` inside a Spring Boot application bypass the IoC container and cannot utilize `@Autowired`, `@Transactional`, or other Spring annotations.
>
> **Reference Concept:** `Normal Java Beans vs. Spring Beans`

---

### What is the difference between @Component and @Bean?

> **Direct Answer:**
> - **Auto-Discovery vs. Explicit Factory:** `@Component` is a class-level annotation discovered automatically via classpath scanning (`@ComponentScan`). `@Bean` is a method-level annotation used within `@Configuration` classes to explicitly return a bean instance.
> - **When to Use `@Component`:** Use on classes that you author and directly control within application source code.
> - **When to Use `@Bean`:** Use when integrating third-party libraries (whose source code cannot be annotated) or when constructing objects requiring complex manual setup before registration.
>
> **Key Nuance / Trade-off:**
> - `@Bean` methods inside `@Configuration` classes are proxied with CGLIB to ensure Singleton scoping, whereas `@Component` classes rely on standard instantiation.
>
> **Reference Concept:** `Bean Declaration: @Component vs. @Bean`

---

### What is the default scope of a Spring bean?

> **Direct Answer:**
> - **Singleton Default:** The default scope for Spring beans is `singleton`.
> - **Single Instance:** Spring creates exactly one shared instance per `ApplicationContext` and injects that same memory reference across all dependents.
> - **Other Available Scopes:** `prototype` (creates a new instance per request), `request` (one per HTTP request), and `session` (one per HTTP session).
>
> **Key Nuance / Trade-off:**
> - **The Concurrency Rule:** Because a singleton bean is shared across multiple concurrent HTTP threads, it must remain strictly **stateless**; maintaining mutable class-level variables in a singleton causes severe race conditions and data corruption.
>
> **Reference Concept:** `Bean Scopes & Concurrency (Singleton, Prototype, Web Scopes)`

---

### How does the Spring IoC container manage Spring Beans?

> **Direct Answer:**
> - **Lifecycle Pipeline:** The container acts as a factory and lifecycle manager, executing: Instantiation, Dependency Injection, Aware Callbacks, Pre-Initialization, Initialization, Post-Initialization, and Destruction.
> - **Object Creation:** Uses Java reflection or constructors to instantiate the raw Java object.
> - **Dependency Wiring:** Resolves and populates `@Autowired` dependencies.
> - **Proxy Augmentation:** Uses `BeanPostProcessors` during post-initialization to wrap beans in dynamic proxies when AOP features (`@Transactional`, `@Async`) are detected.
>
> **Key Nuance / Trade-off:**
> - The container manages destruction callbacks (`@PreDestroy`) for singleton beans, but prototype beans are handed off to the caller after initialization without container destruction management.
>
> **Reference Concept:** `Spring Bean Lifecycle Sequence & BeanPostProcessor Proxies`

---

### Explain bean lifecycle in spring, How does Spring manage the bean lifecycle?

> **Direct Answer:**
> - **Step-by-Step Sequence:** Instantiation -> Dependency Injection -> Aware interfaces -> `@PostConstruct` (Pre-Initialization) -> `InitializingBean.afterPropertiesSet()` -> Post-Initialization (`BeanPostProcessors`) -> Ready -> Destruction (`@PreDestroy`).
> - **The Critical Senior Detail:** The post-initialization phase is where `BeanPostProcessors` intercept the bean; if annotations like `@Transactional`, `@Async`, or `@Cacheable` are present, Spring wraps the bean in a CGLIB or JDK Dynamic Proxy before placing it in the container.
>
> **Key Nuance / Trade-off:**
> - Code inside `@PostConstruct` runs after dependencies are wired but before the bean is wrapped in its final AOP proxy, meaning self-invoked transactional calls inside `@PostConstruct` will not execute within a transaction.
>
> **Reference Concept:** `Spring Bean Lifecycle Sequence & BeanPostProcessor Proxies`

---

### You need to manually create and manage a Spring bean. How do you do it?

> **Direct Answer:**
> - **Functional Registration (Spring 5+):** Use `GenericApplicationContext.registerBean(Class<T>, Supplier<T>)` to register beans programmatically using lambda suppliers.
> - **Factory Registration:** Obtain `DefaultListableBeanFactory` and invoke `registerSingleton("beanName", instance)` with a pre-instantiated object.
> - **Runtime Dynamic Wiring:** Useful for registering dynamic plugins or runtime data sources without configuration classes.
>
> **Key Nuance / Trade-off:**
> - Manually registering pre-instantiated objects via `registerSingleton()` bypasses early lifecycle phases; `@PostConstruct` and `InitializingBean` callbacks will not be executed by Spring on these instances.
>
> **Reference Concept:** `Programmatic & Functional Bean Registration`

---

### You have a service class that needs to be injected into multiple controllers. What are the different ways to achieve this in Spring Boot?

> **Direct Answer:**
> - **Injection Mechanisms:** Inject via Constructor Injection (the production standard), Setter Injection, or Field Injection (anti-pattern).
> - **Lombok Cleanliness:** Annotate controllers with `@RequiredArgsConstructor` to auto-generate constructor wiring for `final` service fields.
> - **Decoupling via Interfaces:** Inject the service interface (`UserService`), not the concrete implementation (`UserServiceImpl`), to maintain loose coupling.
>
> **Key Nuance / Trade-off:**
> - Because the service is a singleton, one shared instance serves all controllers simultaneously; any mutable instance variables in the service will lead to data corruption across concurrent requests.
>
> **Reference Concept:** `Dependency Injection Best Practices & Stateless Singletons`

---

### What is the difference between @Primary and @Qualifier in Spring, and when should you use each?

> **Direct Answer:**
> - **Default vs. Explicit Specificity:** `@Primary` defines a global default implementation when multiple beans match an interface. `@Qualifier` explicitly targets a specific bean by name at the injection site.
> - **When to Use `@Primary`:** Use when 90% of the application expects one primary implementation (e.g., `StripePaymentProcessor`), leaving secondary beans for edge cases or testing.
> - **When to Use `@Qualifier`:** Use when the application actively selects and routes between different implementations concurrently.
>
> **Key Nuance / Trade-off:**
> - Specificity wins: `@Qualifier` is explicit and always overrides `@Primary` when both are present.
>
> **Reference Concept:** `Resolving Bean Ambiguity (@Primary vs. @Qualifier)`

---

### How to resolve NoUniqueBeanDefinitionException?

> **Direct Answer:**
> - **The Exception Cause:** Thrown at startup when the Spring IoC container attempts to autowire an interface implemented by multiple candidate beans without instructions on which to choose.
> - **Solution A (`@Primary`):** Annotate the primary bean with `@Primary` to establish it as the default choice across the application.
> - **Solution B (`@Qualifier`):** Annotate the injection point with `@Qualifier("beanName")` to specify the exact bean required.
> - **Solution C (Match Variable Name):** Match the parameter or field variable name to the exact bean name, which Spring uses as a secondary qualifier fallback.
>
> **Key Nuance / Trade-off:**
> - Relying on variable name matching is brittle and prone to breakage during code refactoring; using explicit `@Qualifier` or `@Primary` is the recommended enterprise practice.
>
> **Reference Concept:** `Resolving Bean Ambiguity (@Primary vs. @Qualifier)`
