# Spring Core, Dependency Injection & Bean Lifecycle

## Spring Framework Evolution & Embedded Servers

### Spring Boot 2.x vs. 3.x Ecosystem Modernization
- **Baseline Requirements:** Spring Boot 3 requires Java 17 as a strict minimum baseline (supporting Java 21 LTS) and Spring Framework 6.
- **Jakarta EE Migration:** The single biggest breaking change is the complete migration from Java EE (`javax.*`) to Jakarta EE (`jakarta.*`) namespaces across Servlets, Persistence, and Validation APIs.
- **GraalVM Native Image Support:** First-class Ahead-Of-Time (AOT) compilation producing native machine binaries with sub-second startup times and near-zero memory footprint for serverless and containerized deployments.
- **Virtual Threads (Project Loom):** Introduced in Spring Boot 3.2+, enabling high-throughput lightweight threading with `spring.threads.virtual.enabled=true`.
- **Security Modernization:** Removal of legacy security adapters (e.g., `WebSecurityConfigurerAdapter`) in favor of component-based `SecurityFilterChain` bean declarations.

### Embedded Servers: Tomcat vs. Netty
- **Apache Tomcat (Default in `spring-boot-starter-web`):**
  - **Model:** Blocking, imperative Thread-Per-Request model.
  - **Thread Pool:** Maintains a worker thread pool (default 200 threads). When an incoming request awaits a slow database query or external REST call, the thread is suspended, risking thread exhaustion under high concurrent load.
  - **Best For:** Standard relational database architectures (JPA/JDBC), CPU-bound processing, and traditional Spring MVC applications.
- **Eclipse Netty (Default in `spring-boot-starter-webflux`):**
  - **Model:** Asynchronous, non-blocking Event Loop architecture.
  - **Thread Pool:** Operates a minimal fixed pool of threads (typically 2x CPU cores). Instead of waiting for I/O, threads register OS interrupt callbacks (e.g., Linux `epoll`) and handle other concurrent requests.
  - **The Strict Rule:** Netty event loop threads must never be blocked. Calling blocking libraries (standard JDBC, `RestTemplate`, `Thread.sleep()`) inside Netty threads freezes the event loop and destroys application throughput.

### Spring Framework vs. Spring Boot Architecture
- **Spring Framework:** The foundational, unopinionated framework providing core inversion of control (IoC), dependency injection (DI), transaction management, and messaging abstractions without imposing configuration or deployment opinions.
- **Spring Boot:** An opinionated, convention-over-configuration layer built on top of Spring Framework designed to eliminate boilerplate and accelerate development.
- **The Three Core Superpowers of Spring Boot:**
  1. **Auto-Configuration:** Scans dependencies on the classpath to automatically configure Spring beans with sensible defaults.
  2. **Starter POMs:** Curated dependency descriptors (e.g., `spring-boot-starter-web`) that aggregate compatible transitive libraries and eliminate version-mismatch conflicts.
  3. **Embedded Web Servers:** Embeds Tomcat, Jetty, or Undertow directly into a self-contained, executable "fat JAR" runnable via `java -jar`.

### Observability: Logging Architecture & Actuator Security
- **Logging Best Practices:**
  - Standardizes on SLF4J (Simple Logging Facade for Java) backed by Logback.
  - Never use `System.out.println()`: it blocks the thread during I/O and cannot be formatted, filtered, or routed asynchronously.
  - Always use parameterized logging with `{}` placeholders (e.g., `log.info("Processing order ID: {}", orderId)`). String concatenation (`"Order " + id`) forces heap allocation and evaluation even when the log level is disabled.
- **Spring Boot Actuator:**
  - Provides built-in production operational endpoints for monitoring JVM metrics, health checks, environment properties, and thread dumps.
  - **Critical Security Pitfall:** Exposing all endpoints via `management.endpoints.web.exposure.include=*` on public internet ports exposes sensitive environment secrets (`/actuator/env`) and heap dumps (`/actuator/heapdump`).
  - **Production Hardening:**
    - Segregate Actuator to an internal, non-public port (e.g., `management.server.port=8081`).
    - Enforce role-based access in `SecurityFilterChain` for management paths.
- **Production Benefits:**
  - **Kubernetes Probes:** Exposes Liveness (`/actuator/health/liveness`) and Readiness (`/actuator/health/readiness`) checks. If a database fails, Readiness signals Kubernetes to stop routing traffic without terminating the container.
  - **Prometheus Metrics:** Exposes `/actuator/prometheus` for scraping metrics into Prometheus and Grafana.
  - **Dynamic Log Level Switching:** Adjust package logging levels at runtime via HTTP POST to `/actuator/loggers/<package>` without restarting the JVM.
- **Custom Health Checks (`HealthIndicator`):**
  - Implement `HealthIndicator` to verify critical third-party dependencies:
    ```java
    @Component
    public class ThirdPartyApiHealthIndicator implements HealthIndicator {
        @Override
        public Health health() {
            boolean isUp = checkThirdPartyStatus();
            if (isUp) {
                return Health.up().withDetail("service", "Payment Gateway Available").build();
            }
            return Health.down().withDetail("error", "Payment Gateway Unreachable").build();
        }
    }
    ```

---

## Inversion of Control (IoC) & Dependency Injection (DI)

### Inversion of Control (IoC) Architectural Philosophy
- **Core Principle:** A software design principle where the custom code relinquishes control over execution flow and object creation to an external framework or container.
- **Historical Context:** Before Spring, Java applications either instantiated dependencies explicitly via `new` or used heavy JNDI lookups inside application servers (e.g., WebLogic, WebSphere). Code actively hunted for dependencies, resulting in rigid coupling and long server startup times during unit tests.
- **The Spring Paradigm:** Classes passively declare required dependencies (e.g., constructor parameters), while the Spring IoC Container (`ApplicationContext`) instantiates, configures, and assembles them.

### Dependency Injection (DI) Mechanics & Benefits
- **The Relationship:** IoC is the broad architectural philosophy ("Don't call us, we'll call you"); Dependency Injection is the concrete mechanical technique used to implement IoC.
- **The Surgeon Analogy:** A surgeon (business service) does not leave the operating table to manufacture a scalpel (calling `new Scalpel()`). Instead, a nurse (the IoC container) places the scalpel into the surgeon's hand (Dependency Injection).
- **Architectural Benefits:**
  - **Loose Coupling:** Services interact with abstract interfaces (e.g., `PaymentProcessor`), allowing implementations (Stripe, PayPal) to be swapped globally without altering business code.
  - **Deterministic Testability:** Dependencies are easily substituted with mock instances (e.g., `Mockito.mock(PaymentProcessor.class)`) in isolated unit tests without spinning up databases or external APIs.

### Dependency Injection Types (Constructor, Setter, Field)
1. **Constructor Injection (Enterprise Standard):**
   - Dependencies are passed through the class constructor.
   - Allows fields to be declared `final`, guaranteeing immutability.
   - Prevents `NullPointerException` by ensuring an object cannot be instantiated in an incomplete state.
   - Facilitates pure unit testing without requiring Spring or reflection frameworks.
2. **Setter Injection:**
   - Dependencies are injected via `@Autowired` setter methods.
   - Allows re-injection or optional dependencies, but sacrifices field immutability and exposes objects to mutation.
3. **Field Injection (Anti-Pattern):**
   - Placing `@Autowired` directly on private fields.
   - Uses Java reflection, breaks encapsulation, prevents `final` declarations, and makes unit testing outside of Spring containers difficult.

### Circular Dependencies: Causes, Pitfalls & Resolution (@Lazy, Redesign)
- **The Cause:** Occurs when two or more beans depend on each other (Bean A requires Bean B, and Bean B requires Bean A).
- **The Failure:** The IoC container enters an infinite instantiation loop attempting to satisfy constructors, throwing a `BeanCurrentlyInCreationException`.
- **Resolution Strategies:**
  1. **Architectural Redesign (Best Practice):** Circular dependencies indicate a violation of the Single Responsibility Principle. Extract shared logic into a distinct, third service (e.g., `OrderValidationService`) so both dependent services point downward in the dependency tree.
  2. **`@Lazy` Annotation:** Placing `@Lazy` on one constructor parameter causes Spring to inject a CGLIB dynamic proxy. The real bean is initialized only when a method is first called.
  3. **Setter Injection:** Allows both beans to be created in heap memory before setter wiring occurs. (Note: As of Spring Boot 2.6+, circular dependencies are disabled by default and require `spring.main.allow-circular-references=true`).

---

## Spring Beans, Scopes & Lifecycle Management

### Normal Java Beans vs. Spring Beans
- **Normal Java Bean:** An object instantiated, initialized, and garbage-collected manually by developer code using `new`.
- **Spring Bean:** An object instantiated, configured, assembled, and managed entirely by the Spring IoC Container (`ApplicationContext`), participating in lifecycle callbacks and AOP proxying.

### Bean Declaration: @Component vs. @Bean
- **`@Component` (Auto-Discovery):**
  - Class-level stereotype annotation discovered automatically via classpath component scanning (`@ComponentScan`).
  - Used on classes that you author and own directly in application source code.
- **`@Bean` (Explicit Factory Method):**
  - Method-level annotation used within `@Configuration` classes.
  - Explicitly returns a bean instance managed by the container.
  - Required when integrating third-party libraries (where source code cannot be annotated) or when constructing instances with complex manual configuration logic.

### Bean Scopes & Concurrency (Singleton, Prototype, Web Scopes)
- **`singleton` (Default Scope):**
  - Spring creates exactly one instance per `ApplicationContext` and shares it across all injection points.
  - **Concurrency Rule:** Because singletons are accessed by multiple concurrent request threads, they must remain **strictly stateless**. Never maintain mutable instance state (e.g., `currentUser`) in singleton beans.
- **`prototype` Scope:**
  - A new instance is created every time the bean is requested from the container.
  - Note: Spring manages initialization of prototypes but does not manage their destruction (`@PreDestroy` is not called on prototypes).
- **Web Scopes:**
  - `request`: Exactly one instance per HTTP request lifecycle.
  - `session`: Exactly one instance per HTTP session.
  - `application`: Scoped to the lifecycle of the `ServletContext`.

### Spring Bean Lifecycle Sequence & BeanPostProcessor Proxies
- **Lifecycle Sequence:**
  1. **Instantiation:** Spring instantiates the bean class via reflection or constructor.
  2. **Populate Properties:** Resolves and injects dependencies via `@Autowired`.
  3. **Aware Interfaces:** Injects framework context (`BeanNameAware`, `ApplicationContextAware`).
  4. **Pre-Initialization (`BeanPostProcessor.postProcessBeforeInitialization`):** Executes `@PostConstruct` annotated methods.
  5. **Initialization:** Invokes `InitializingBean.afterPropertiesSet()` or custom `initMethod`.
  6. **Post-Initialization (`BeanPostProcessor.postProcessAfterInitialization`):** The critical phase where Spring checks for AOP annotations (`@Transactional`, `@Async`, `@Cacheable`) and wraps the raw bean in a **Dynamic Proxy** before storing it in the IoC container.
  7. **Destruction:** Invokes `@PreDestroy`, `DisposableBean.destroy()`, or custom `destroyMethod` upon container shutdown.

### Programmatic & Functional Bean Registration
- **Functional Registration (Spring 5+):**
  - Register beans dynamically at runtime using `GenericApplicationContext`:
    ```java
    GenericApplicationContext context = new GenericApplicationContext();
    context.registerBean(CustomService.class, () -> new CustomService());
    ```
- **Factory Registration (`DefaultListableBeanFactory`):**
  - Register an existing instance via `registerSingleton("beanName", instance)`.
  - **Trade-off:** Pre-instantiated instances registered via `registerSingleton()` bypass early Spring lifecycle hooks (`@PostConstruct` and `InitializingBean` are not invoked).

### Dependency Injection Best Practices & Stateless Singletons
- **Interface-Driven Design:** Inject service interfaces (e.g., `UserService`) rather than concrete implementations (`UserServiceImpl`) into controllers to maximize decoupling and allow dynamic mocking.
- **Lombok Cleanliness:** Use `@RequiredArgsConstructor` on classes with `final` fields to auto-generate constructor injection boilerplate.
- **Thread-Safety Warning:** Singletons are shared across all concurrent HTTP request threads; mutating class fields within a singleton causes severe data contamination and race conditions.

### Resolving Bean Ambiguity (@Primary vs. @Qualifier)
- **The Conflict:** If two beans implement the same interface (e.g., `StripePaymentProcessor` and `PaypalPaymentProcessor`), Spring throws a `NoUniqueBeanDefinitionException` at startup.
- **`@Primary` (Global Default):**
  - Placed on a bean class/method to designate it as the default choice when no specific qualifier is requested.
  - Best used when one implementation represents the primary production standard (90% use case).
- **`@Qualifier` (Explicit Specification):**
  - Placed at the injection point (`@Qualifier("paypalPaymentProcessor")`) to explicitly choose a specific bean by name.
  - Best used when multiple implementations are actively chosen and wired across different parts of the application.
- **Precedence Rule:** `@Qualifier` is specific and always takes precedence over `@Primary`.
