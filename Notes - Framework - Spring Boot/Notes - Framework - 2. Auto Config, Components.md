# Spring Boot Auto-Configuration, Components & Enterprise Web Architecture

## Spring Boot Startup & Auto-Configuration Mechanics

### Spring Boot Startup Sequence (SpringApplication.run())
- **Startup Sequence (4-Phase Lifecycle):**
  1. **Bootstrap Phase:** Checks the classpath environment to deduce application type (`SERVLET`, `REACTIVE`, or `NONE`), instantiates bootstrap listeners, and loads initializers from `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports` (or `spring.factories`).
  2. **Environment Preparation:** Builds and merges `application.properties`, YAML files, OS environment variables, system properties, and CLI arguments into a unified `ConfigurableEnvironment` and prints the startup banner.
  3. **Context Creation:** Instantiates the `ApplicationContext` (e.g., `AnnotationConfigServletWebServerApplicationContext`), scans classes starting from `@SpringBootApplication`, and registers bean definitions.
  4. **Context Refresh (`context.refresh()`):** The core phase—instantiates singleton beans, resolves and wires `@Autowired` dependencies, registers `BeanPostProcessors`, and starts the embedded web container (Tomcat/Jetty/Netty).
  5. **Final Lifecycle Hooks:** Executes all `CommandLineRunner` and `ApplicationRunner` beans, and publishes the `ApplicationReadyEvent` indicating traffic readiness.

### Auto-Configuration Discovery & Conditionals
- **Core Concept:** Eliminates boilerplate XML/Java configuration by automatically configuring beans based on classpath JARs and environment settings.
- **Discovery Mechanism:** Triggered by `@SpringBootApplication` (wrapping `@EnableAutoConfiguration`). Spring scans `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports` to evaluate candidate configuration classes.
- **The Back-Off Principle (`@ConditionalOnMissingBean`):**
  - Auto-configuration uses conditional annotations (`@ConditionalOnClass`, `@ConditionalOnMissingBean`, `@ConditionalOnProperty`).
  - User-defined beans always take precedence. If a custom `DataSource` or `ObjectMapper` `@Bean` is defined, Spring Boot backs off its default auto-configuration.

### @ConditionalOnProperty & Feature Toggles
- **Purpose:** Conditionally registers a `@Bean` or `@Configuration` class based on the presence, absence, or value of a configuration property.
- **Key Attributes:**
  - `name` / `value`: The target configuration property key.
  - `havingValue`: The expected value required to match (supports relaxed boolean binding).
  - `matchIfMissing`: If `true`, the bean is registered when the property is absent in properties/YAML.
- **Feature Toggles:** Senior engineers prefer `@ConditionalOnProperty` over `@Profile` for runtime feature flags that can be toggled via environment variables without rebuilding or altering profiles. Pair with `@ConfigurationProperties` validation for type safety.

### Externalized Configuration & Spring Cloud Config
- **Environment Management:** Manage environments via `application-{profile}.properties` or multi-document YAML (`spring.config.activate.on-profile`). Activate using `spring.profiles.active=prod` or OS variable `SPRING_PROFILES_ACTIVE=prod`.
- **Spring Cloud Config Server Architecture:**
  - **Config Server:** Centralized service annotated with `@EnableConfigServer` connected to a versioned Git/Vault repository.
  - **Config Client:** Connects at startup before context creation.
- **Modern Cloud-Native Standards:**
  - **`spring.config.import`:** In Spring Boot 2.4+, `bootstrap.yml` is obsolete; configure the server URL directly in `application.yml` using `spring.config.import=optional:configserver:http://config-server:8888`.
  - **Live Reloads (`@RefreshScope`):** Singletons do not auto-refresh on Git changes; annotate beans with `@RefreshScope` and broadcast refresh events using Spring Cloud Bus (Kafka/RabbitMQ).
  - **Kubernetes / Cloud Alternative:** In containerized clouds, replace Config Server JVMs with native Kubernetes ConfigMaps/Secrets or AWS Systems Manager Parameter Store/Secrets Manager.

### @Component vs. @Configuration & CGLIB Proxying (Lite vs. Full Mode)
- **Difference:** `@Component` marks a class as a standard managed bean. `@Configuration` marks a class as a factory containing `@Bean` methods.
- **CGLIB Proxying (Full Mode):**
  - When `@Bean` method `userRepository()` directly invokes another `@Bean` method `dataSource()` inside the same `@Configuration` class, standard Java would invoke `new` twice, breaking the Singleton pattern.
  - Spring generates a CGLIB subclass proxy of the `@Configuration` class. The proxy intercepts the method call, checks the IoC container, and returns the cached singleton.
- **Lite Mode:** If `@Bean` methods are placed inside a `@Component`, no CGLIB proxy is generated ("Lite Mode"). Method calls execute standard Java invocations, resulting in duplicate object creation.
- **Performance Optimization:** If `@Bean` methods do not call each other, set `@Configuration(proxyBeanMethods = false)` to skip CGLIB proxy generation and reduce startup time.

### Background Job Scheduling (@Scheduled & ThreadPoolTaskScheduler)
- **Configuration:** Enable globally with `@EnableScheduling` and annotate methods with `@Scheduled`:
  - `fixedRate`: Executes start-to-start on a fixed duration (risks overlapping executions if tasks take longer than interval).
  - `fixedDelay`: Waits for the previous execution to finish before starting the delay timer.
  - `cron`: Schedules using cron expressions (e.g., `0 0 12 * * ?`).
- **The Threading Trap:** By default, Spring Boot executes scheduled tasks on a **single thread**. If one job blocks, all other scheduled tasks across the JVM are halted.
- **Production Fix:** Configure a dedicated `ThreadPoolTaskScheduler` bean with an adequate `poolSize`.

---

## Spring MVC Components & Architecture

### Core Stereotypes (@Component, @Controller, @Service, @Repository)
- **`@Component`:** The generic archetype annotation for Spring-managed beans.
- **`@Controller`:** Marks a class in the Presentation Layer, handling HTTP requests and returning views (HTML/JSP/Thymeleaf) or JSON when combined with `@ResponseBody`.
- **`@Service`:** Marks the Business Logic layer. Functionally equivalent to `@Component`, but communicates architectural intent for Domain-Driven Design and enables clear targeting for Aspect-Oriented Programming (AOP).
- **`@Repository`:** Marks the Data Access Layer (DAO). In addition to bean management, it automatically translates low-level database vendor exceptions into Spring's unified `DataAccessException` hierarchy.

### @Controller vs. @RestController
- **`@Controller` (Traditional MVC):** Intended for server-rendered UIs. When a method returns a String, Spring delegates to a `ViewResolver` to locate and render a physical HTML template.
- **`@RestController` (Modern REST APIs):** A meta-annotation combining `@Controller` and `@ResponseBody`. It bypasses the `ViewResolver` entirely; return values are converted directly into JSON/XML via `HttpMessageConverter` (e.g., Jackson) written to the HTTP response body.

### Routing (@RequestMapping vs. @GetMapping)
- **`@RequestMapping`:** The generic, multi-purpose routing annotation capable of mapping URLs across any HTTP method.
- **Composed Shortcuts:** `@GetMapping`, `@PostMapping`, `@PutMapping`, `@DeleteMapping`, and `@PatchMapping` are specialized variants introduced in Spring 4.3.
- **Architectural Best Practice:** Use `@RequestMapping("/api/v1/resource")` exclusively at the class level to establish the shared base URI, and use specific composed annotations (`@GetMapping`, `@PostMapping`) at the method level.

### Lombok @UtilityClass
- **Purpose:** Eliminates boilerplate for stateless utility/helper classes.
- **Generated Bytecode:**
  - Marks the class `final`.
  - Makes all methods, fields, and inner classes `static`.
  - Generates a private no-argument constructor that throws an `UnsupportedOperationException` to prevent instantiation via reflection.

### Transactions (@Transactional Boundaries & Propagation)
- **ACID Guarantee:** Wraps service methods in an AOP proxy that opens a transaction before method execution, commits on successful completion, and rolls back upon uncaught unchecked exceptions (`RuntimeException`, `Error`).
- **Transaction Boundaries:** Senior engineers place boundaries strictly at the Service Layer to orchestrate multiple repository operations atomically.
- **Key Propagation Levels:**
  - `REQUIRED` (Default): Joins the active transaction if one exists; creates a new one if none exists.
  - `REQUIRES_NEW`: Suspends any active transaction and starts a completely independent, isolated transaction. Ideal for audit logging so log records commit even if the parent business transaction rolls back.

### CORS Architecture (@CrossOrigin vs. SecurityFilterChain)
- **The Problem:** Modern browsers enforce the Same-Origin Policy, blocking responses when frontend origins (e.g., `http://localhost:3000`) call APIs on a different origin (`http://localhost:8080`).
- **Pre-Flight Requests:** Browsers issue an HTTP `OPTIONS` pre-flight check before complex requests (like `POST` with `application/json`) to verify allowed headers and origins.
- **`@CrossOrigin`:** A localized annotation for quick controller-level fixes; creates maintenance overhead at scale.
- **Global Configuration:**
  - In pure Spring MVC: Configure `addCorsMappings()` via a `WebMvcConfigurer` bean.
  - **The Spring Security Trap:** If Spring Security is present, the MVC CORS configuration is bypassed because security filters run before MVC dispatch. CORS **must** be configured directly within the `SecurityFilterChain` bean (`http.cors(...)`).

---

## Data, Web Services & Resilience

### JPA Specification vs. Hibernate ORM
- **JPA (Java Persistence API / Jakarta Persistence):** A vendor-neutral specification (`jakarta.persistence`) defining interfaces (`EntityManager`), annotations (`@Entity`, `@Id`), and rules, with zero implementation logic.
- **Hibernate ORM:** A concrete implementation of the JPA specification, providing the execution engine plus vendor-specific extensions (caching, custom types, HQL).
- **Comparison Matrix:**
  | Aspect | JPA | Hibernate |
  | :--- | :--- | :--- |
  | **Nature** | Specification / API standard | Concrete ORM framework implementation |
  | **Package** | `jakarta.persistence` / `javax.persistence` | `org.hibernate` |
  | **Definitions** | Interfaces (`EntityManager`), annotations (`@Entity`, `@Table`) | Concrete classes (`Session`, `SessionFactory`), HQL |
  | **Portability** | High (Switch providers without changing business code) | Low (Ties application to Hibernate-specific APIs) |
  | **Feature Set** | Baseline standard ORM feature set | Superset of JPA (advanced caching, multi-tenancy) |

### Content Negotiation & Payload Conversion (XML vs. JSON)
- **Consuming XML:** Add `jackson-dataformat-xml`, annotate models with `@XmlRootElement`, and declare `consumes = MediaType.APPLICATION_XML_VALUE` on the `@PostMapping` endpoint.
- **Dynamic XML to JSON Conversion:**
  - Avoid brittle third-party XML converters.
  - Parse the raw XML into a generic `JsonNode` tree using Jackson's `XmlMapper`, then serialize that tree directly into a JSON string using Jackson's `ObjectMapper`:
    ```java
    XmlMapper xmlMapper = new XmlMapper();
    ObjectMapper jsonMapper = new ObjectMapper();
    JsonNode node = xmlMapper.readTree(xmlString);
    String jsonString = jsonMapper.writeValueAsString(node);
    ```

### Internationalization (i18n) & Localized Messages
- **Header Resolution:** Clients provide preferred languages via the `Accept-Language` header. Spring's `AcceptHeaderLocaleResolver` extracts the `Locale` and stores it in `LocaleContextHolder` (`ThreadLocal`).
- **Resource Bundles:** Messages are stored in `src/main/resources/messages.properties` (fallback) and `messages_{locale}.properties` (e.g., `messages_fr.properties`).
- **Approaches:**
  1. **Declarative Validation:** Use `{property.key}` in Jakarta Validation annotations (`@NotNull(message = "{user.name.required}")`).
  2. **Programmatic Exceptions:** Inject `MessageSource` to resolve messages with runtime parameters (`messageSource.getMessage("error.code", new Object[]{id}, locale)`).
- **Thread Boundary Gotcha:** Because `LocaleContextHolder` uses `ThreadLocal`, async worker threads (`@Async`) and reactive streams (`Mono`/`Flux`) lose the locale context unless passed explicitly from the parent request thread.

### REST Validation, PathVariables & 404 Error Handling
- **Path Variable Capture:** Extract resource IDs from URL paths using `@PathVariable` (`GET /api/users/{userId}`).
- **Custom Exceptions:** Throw domain-specific runtime exceptions (`UserNotFoundException`) from the service layer rather than embedding HTTP logic inside business services.
- **Global Error Handling:** Intercept exceptions with `@RestControllerAdvice` and `@ExceptionHandler(UserNotFoundException.class)`. Return structured DTOs or Spring Boot 3's built-in `ProblemDetail` (RFC 7807 / RFC 9457).

### Microservice Resilience & Circuit Breakers (Resilience4j)
- **The Rule:** Never hard-code static error responses or allow threads to block indefinitely waiting for dead downstream microservices.
- **Circuit Breaker Mechanism:** Monitors downstream failures. If failure thresholds are exceeded, the circuit "opens" and immediately halts outgoing requests, preserving thread pools.
- **Strategies:**
  - **Strategy A - Fail Fast:** For mandatory operations (e.g., payment checkout), return `503 Service Unavailable` with a structured `ProblemDetail` explaining the offline dependency.
  - **Strategy B - Graceful Degradation (Fallback):** For optional features (e.g., recommendations), return a `200 OK` fallback with cached or empty data.

### External HTTP Service Integration (RestClient & WebClient)
- **Modern Standards (Spring 6.1+):** `RestTemplate` is in maintenance mode.
  - **`RestClient` (Synchronous / Blocking):** Modern fluent builder for synchronous Spring MVC applications.
  - **`WebClient` (Asynchronous / Non-Blocking):** Functional reactive client for high-concurrency or WebFlux architectures.
  - **OpenFeign (Declarative):** Interface-driven declarative HTTP client auto-generated by Spring Cloud.

### Hiding JSON Fields & DTO Serialization
- **The DTO Pattern (Best Practice):** Map internal database entities to dedicated Response DTOs that physically lack sensitive fields (passwords, SSNs), preventing accidental data leakage.
- **Jackson Annotations:**
  - `@JsonIgnore`: Completely excludes a field from both serialization and deserialization.
  - `@JsonProperty(access = Access.WRITE_ONLY)`: Accepts the property during incoming `POST`/`PUT` requests, but never includes it in outgoing JSON responses.

### Large File Uploads (Streaming & Pre-Signed URLs)
- **The Memory Trap:** Calling `MultipartFile.getBytes()` directly loads the entire payload into JVM heap memory, causing `OutOfMemoryError` on large files.
- **Application Solution:** Configure Tomcat limits in `application.properties` (`spring.servlet.multipart.max-file-size=2GB`) and stream the `InputStream` directly to disk or cloud storage.
- **Architecture Solution (Pre-Signed URLs):** Proxying massive files through Spring microservices wastes CPU and network bandwidth. The API authenticates the user, generates a secure, temporary AWS S3 Pre-Signed URL, and returns it to the client. The client uploads the file directly to cloud storage, bypassing the backend JVM entirely.
