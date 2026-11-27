# Spring Boot Auto-Configuration & Components - Interview Q&A

### What actually happens when we start a Spring Boot app?

> **Direct Answer:**
> - **Four-Phase Lifecycle:** Executing `SpringApplication.run()` initiates a four-phase sequence: Bootstrap, Environment Preparation, Context Creation, and Context Refresh, followed by startup runners.
> - **Bootstrap & Environment:** Deduce application type (`SERVLET` or `REACTIVE`), load initializers from `AutoConfiguration.imports`, and merge properties/YAML/environment variables into a unified `ConfigurableEnvironment`.
> - **Context Creation & Refresh:** Instantiate the `ApplicationContext`, register bean definitions, wire dependencies, configure `BeanPostProcessors`, and launch the embedded Tomcat or Netty server.
> - **Post-Startup Hooks:** Execute all `CommandLineRunner` and `ApplicationRunner` beans and publish the `ApplicationReadyEvent` to accept traffic.
>
> **Key Nuance / Trade-off:**
> - Beans are not instantiated during the Context Creation scanning phase; they are only registered as bean definitions. Full instantiation and dependency injection occur during the `context.refresh()` phase.
>
> **Reference Concept:** `Spring Boot Startup Sequence (SpringApplication.run())`

---

### Explain Spring Boot autoconfiguration and its advantages.

> **Direct Answer:**
> - **Convention Over Configuration:** Automatically configures beans and components based on classpath dependencies and active environment settings, eliminating manual XML and boilerplate Java configuration.
> - **Sensible Defaults & Seamless Integration:** Provides battle-tested default beans (e.g., HikariCP DataSource, Jackson ObjectMapper) while allowing effortless overrides.
> - **Conditional Activation:** Relies on `@Conditional` annotations (e.g., `@ConditionalOnClass`, `@ConditionalOnMissingBean`) triggered by `@SpringBootApplication`.
>
> **Key Nuance / Trade-off:**
> - Auto-configuration provides rapid development velocity, but can cause debugging challenges ("Spring magic") if developers do not understand evaluation order or how `@Conditional` annotations evaluate candidate beans.
>
> **Reference Concept:** `Auto-Configuration Discovery & Conditionals`

---

### How does Spring Boot auto-configuration work?

> **Direct Answer:**
> - **Discovery Phase:** Spring Boot reads `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports` (or `spring.factories` prior to Boot 2.7) to identify candidate auto-configuration classes.
> - **Evaluation Phase:** Each candidate class evaluates runtime conditions via `@ConditionalOnClass`, `@ConditionalOnProperty`, and `@ConditionalOnMissingBean`.
> - **The Back-Off Principle:** User-defined beans are registered first; when Spring Boot encounters `@ConditionalOnMissingBean`, it detects the custom bean and backs off its default configuration.
>
> **Key Nuance / Trade-off:**
> - Auto-configuration ordering matters; configurations specify `@AutoConfigureAfter` or `@AutoConfigureBefore` to ensure prerequisite beans are in the container before dependent conditionals evaluate.
>
> **Reference Concept:** `Auto-Configuration Discovery & Conditionals`

---

### What is the role of @ConditionalOnProperty?

> **Direct Answer:**
> - **Conditional Bean Registration:** Registers a `@Bean` or `@Configuration` class conditionally based on the presence, absence, or specific value of an environment property.
> - **Key Attributes:** Uses `name` (the property key), `havingValue` (the expected value), and `matchIfMissing` (whether the bean registers when the property is omitted).
> - **Feature Toggles:** Enables production feature flags that can be toggled across environments or CI/CD pipelines without altering code or active profiles.
>
> **Key Nuance / Trade-off:**
> - Property strings in `@ConditionalOnProperty` lack compile-time refactoring safety; pair with `@ConfigurationProperties` and metadata generators for validation.
>
> **Reference Concept:** `@ConditionalOnProperty & Feature Toggles`

---

### How do you manage environment-specific configuration in Spring Boot (dev, test, uat prod)?

> **Direct Answer:**
> - **Profile Properties:** Define `application-{profile}.properties` or multi-document YAML blocks partitioned by `spring.config.activate.on-profile`.
> - **Profile Activation:** Activate profiles via `spring.profiles.active=prod` or environment variable `SPRING_PROFILES_ACTIVE=prod`.
> - **Bean-Level Profiles:** Annotate specific `@Bean` or `@Configuration` classes with `@Profile("prod")` to wire environment-specific beans (e.g., real payment gateway in prod vs. mock gateway in dev).
> - **Externalized Storage:** Keep production secrets outside the application JAR using `/config/` directories, OS environment variables, or cloud secrets vaults.
>
> **Key Nuance / Trade-off:**
> - Hardcoding secrets or configurations inside profile files embedded in the JAR violates 12-factor application design; sensitive environment credentials should always be injected via externalized environment variables or secret stores.
>
> **Reference Concept:** `Externalized Configuration & Spring Cloud Config`

---

### How does Spring Boot integrate with cloud configuration services using Spring Cloud Config Server?

> **Direct Answer:**
> - **Centralized Configuration:** A dedicated Spring Boot server with `@EnableConfigServer` serves configuration files from a versioned Git repository, HashiCorp Vault, or AWS S3.
> - **Client Handshake:** Client microservices contact the Config Server during startup before initializing their ApplicationContext, loading profile-specific properties over HTTP.
> - **Modern `spring.config.import`:** Spring Boot 2.4+ obsoleted `bootstrap.yml`; clients configure the server URL directly in `application.yml` via `spring.config.import=optional:configserver:http://config-server:8888`.
>
> **Key Nuance / Trade-off:**
> - **Dynamic Reloads:** Singletons do not auto-refresh on Git commits; beans must be annotated with `@RefreshScope` and paired with Spring Cloud Bus (Kafka/RabbitMQ) to broadcast refresh webhooks across microservice instances.
> - **Cloud-Native Alternative:** Running a standalone Config Server JVM is often redundant in Kubernetes or AWS environments, where native ConfigMaps, Secrets, or AWS Parameter Store provide zero-JVM management.
>
> **Reference Concept:** `Externalized Configuration & Spring Cloud Config`

---

### What is the difference between @Component and @Configuration? Why does @Configuration use CGLIB proxying?

> **Direct Answer:**
> - **Functional Role:** `@Component` designates a class as a standard managed bean, whereas `@Configuration` identifies a class as a factory of `@Bean` definitions.
> - **CGLIB Proxying (Full Mode):** `@Configuration` classes are proxied using CGLIB at startup so that when one `@Bean` method directly invokes another `@Bean` method inside the same class, the call is intercepted to return the cached singleton rather than executing `new` twice.
> - **Lite Mode:** Putting `@Bean` methods inside a `@Component` runs in "Lite Mode" without CGLIB proxying; calling methods directly bypasses the IoC container and instantiates duplicate objects.
>
> **Key Nuance / Trade-off:**
> - If `@Bean` factory methods do not call each other, specifying `@Configuration(proxyBeanMethods = false)` skips CGLIB proxy generation, reducing JVM startup overhead and memory consumption.
>
> **Reference Concept:** `@Component vs. @Configuration & CGLIB Proxying (Lite vs. Full Mode)`

---

### How do you schedule recurring background jobs in Spring Boot?

> **Direct Answer:**
> - **Global Activation:** Enable scheduling across the application using `@EnableScheduling` on a configuration class.
> - **Method Declarations:** Annotate target methods with `@Scheduled` using `fixedRate` (start-to-start interval), `fixedDelay` (interval after completion), or `cron` (calendar-based expressions).
> - **Execution Control:** Ensure tasks are non-overlapping and maintain predictable intervals based on workload characteristics.
>
> **Key Nuance / Trade-off:**
> - **The Single-Thread Trap:** Spring Boot defaults to a single-threaded task scheduler; if one scheduled task hangs or runs long, all other background jobs across the application are blocked. Always declare a `ThreadPoolTaskScheduler` bean with a configured `poolSize`.
>
> **Reference Concept:** `Background Job Scheduling (@Scheduled & ThreadPoolTaskScheduler)`

---

### Explain Spring components such as @Controller, @Service and @Repository.

> **Direct Answer:**
> - **Layered Architecture:** Stereotype annotations specialize `@Component` to assign explicit architectural responsibilities to managed beans.
> - **`@Controller`:** Handles HTTP requests, manages presentation logic, and renders views or JSON responses in the Web/Presentation Layer.
> - **`@Service`:** Encapsulates business logic, domain rules, and transactional boundaries in the Service Layer.
> - **`@Repository`:** Encapsulates database CRUD operations and query logic in the Data Access Layer (DAO).
>
> **Key Nuance / Trade-off:**
> - `@Repository` provides active framework behavior: it automatically enables persistence exception translation, catching low-level database exceptions and wrapping them into Spring's unchecked `DataAccessException` hierarchy.
>
> **Reference Concept:** `Core Stereotypes (@Component, @Controller, @Service, @Repository)`

---

### What is the exact difference between @Controller and @RestController, and how does Spring handle the data returned by each?

> **Direct Answer:**
> - **Target Audience:** `@Controller` is designed for Server-Side Rendering (returning HTML/JSP templates), while `@RestController` is designed for RESTful Web APIs (returning raw data payloads like JSON/XML).
> - **ViewResolver vs. ResponseBody:** Returning a String from `@Controller` prompts the `ViewResolver` to locate and render a physical template. `@RestController` is a meta-annotation that applies `@ResponseBody` to every method.
> - **Data Handling:** In `@RestController`, the `ViewResolver` is bypassed entirely, and return objects are serialized directly into the HTTP response body by `HttpMessageConverters` (e.g., Jackson).
>
> **Key Nuance / Trade-off:**
> - Returning an object from a standard `@Controller` without `@ResponseBody` causes Spring to interpret the string/name as a view path, resulting in view resolution failures (e.g., 404 or `Circular view path`).
>
> **Reference Concept:** `@Controller vs. @RestController`

---

### What is the difference between @RequestMapping and specific request annotations like @GetMapping, and how should they be used together?

> **Direct Answer:**
> - **Generic vs. Composed:** `@RequestMapping` is the generic routing annotation requiring explicit HTTP method definitions (`method = RequestMethod.GET`), whereas `@GetMapping`, `@PostMapping`, etc., are composed shortcuts pre-configured with the HTTP verb.
> - **Boilerplate Reduction:** Composed annotations improve readability, reduce clutter, and clearly express REST semantics.
> - **Cohesive Architectural Pattern:** Place `@RequestMapping("/base-path")` at the class level to establish the shared root URI, and use specific composed annotations (`@GetMapping("/{id}")`) at the method level.
>
> **Key Nuance / Trade-off:**
> - Avoid using method-level `@RequestMapping` without specifying the HTTP method, as it matches all HTTP verbs (GET, POST, PUT, DELETE) on that endpoint, exposing security and API contract risks.
>
> **Reference Concept:** `Routing (@RequestMapping vs. @GetMapping)`

---

### What does the @Service annotation do? Difference between @Component and @Service

> **Direct Answer:**
> - **Mechanical Parity:** Mechanically, `@Service` is identical to `@Component` and registers the class as a Spring-managed bean in the IoC container.
> - **Architectural Intent:** Communicates Domain-Driven Design semantics, explicitly marking the bean as part of the business logic layer.
> - **AOP Pointcut Targeting:** Enables Aspect-Oriented Programming pointcuts to cleanly target the service layer (e.g., applying security, logging, or metric aspects specifically to `@within(org.springframework.stereotype.Service)`).
>
> **Key Nuance / Trade-off:**
> - While `@Service` has no unique mechanical bytecode enhancements compared to `@Repository`, replacing it with generic `@Component` degrades code maintainability and hinders layer-specific AOP aspects.
>
> **Reference Concept:** `Core Stereotypes (@Component, @Controller, @Service, @Repository)`

---

### What is the @UtilityClass annotation in Lombok, and what architectural boilerplate does it eliminate?

> **Direct Answer:**
> - **Stateless Helper Safety:** Automates the creation of secure, stateless utility classes.
> - **Generated Bytecode:** Makes the class `final`, converts all fields, methods, and inner classes to `static`, and creates a private constructor that throws `UnsupportedOperationException`.
> - **Boilerplate Elimination:** Eliminates verbose manual Java defensive boilerplate needed to prevent accidental class inheritance or reflection-based instantiation.
>
> **Key Nuance / Trade-off:**
> - Because all methods and fields become static, utility classes annotated with `@UtilityClass` cannot be managed as Spring beans or injected via dependency injection; they should strictly hold pure, stateless functions.
>
> **Reference Concept:** `Lombok @UtilityClass`

---

### Explain the @Transactional annotation. What are transaction boundaries and propagation levels?

> **Direct Answer:**
> - **Declarative ACID Boundaries:** Guarantees Atomicity by wrapping methods in an AOP proxy that starts a transaction, commits on success, and triggers a rollback upon encountering unchecked exceptions (`RuntimeException`).
> - **Transaction Boundaries:** Represent the start and end of a transactional unit of work, strictly established at the Service Layer to coordinate multiple repository calls.
> - **`REQUIRED` Propagation (Default):** Joins an ongoing transaction if one exists, or creates a new transaction if none is active.
> - **`REQUIRES_NEW` Propagation:** Suspends the current transaction and initiates a completely independent transaction (crucial for audit logging where logs must persist even if the parent business transaction fails).
>
> **Key Nuance / Trade-off:**
> - `@Transactional` does not roll back for checked exceptions by default; enterprise methods should specify `@Transactional(rollbackFor = Exception.class)` and avoid internal self-invocation which bypasses the AOP proxy.
>
> **Reference Concept:** `Transactions (@Transactional Boundaries & Propagation)`

---

### What is @CrossOrigin, why do we need it, and how do you architect CORS in a production environment?

> **Direct Answer:**
> - **Browser Same-Origin Policy:** Browsers block web pages on one origin (e.g., `localhost:3000`) from reading responses from a different origin (`localhost:8080`) unless CORS headers permit it.
> - **Pre-Flight `OPTIONS` Requests:** Browsers issue an HTTP `OPTIONS` check to verify whether the cross-origin server allows the method and headers before dispatching the real request.
> - **`@CrossOrigin` Limitation:** While useful for quick development, annotating individual controllers creates maintenance fragmentation in production.
> - **Production Architecture:** Configure CORS globally. In Spring MVC, define `addCorsMappings()` via a `WebMvcConfigurer` bean. In applications with Spring Security, CORS must be configured directly within `SecurityFilterChain`.
>
> **Key Nuance / Trade-off:**
> - If Spring Security is present, security filters intercept requests before Spring MVC executes; configuring CORS only in MVC will cause Spring Security to block pre-flight `OPTIONS` requests with 401/403 errors.
>
> **Reference Concept:** `CORS Architecture (@CrossOrigin vs. SecurityFilterChain)`

---

### What is the difference between JPA (Java Persistence API) and Hibernate ORM, and how do they relate to each other in a typical Spring/Spring Boot application?

> **Direct Answer:**
> - **Specification vs. Implementation:** JPA is a specification (`jakarta.persistence`) that defines interfaces (`EntityManager`) and annotations (`@Entity`) with no code implementation. Hibernate is the concrete ORM library implementing that specification.
> - **Vendor Neutrality:** Coding against JPA interfaces allows changing the underlying ORM provider (e.g., Hibernate to EclipseLink) without rewriting business entity mappings.
> - **Hibernate Superset:** Hibernate provides advanced features beyond standard JPA, including first- and second-level caching architectures, custom types, and Hibernate Query Language (HQL).
>
> **Key Nuance / Trade-off:**
> - Spring Boot applications code against JPA abstractions (`JpaRepository`, `jakarta.persistence.*`), while Spring Boot automatically configures Hibernate under the hood as the default persistence provider.
>
> **Reference Concept:** `JPA Specification vs. Hibernate ORM`

---

### How do you consume data in XML format if another application is sending it in JSON?

> **Direct Answer:**
> - **Content Negotiation:** Configure the endpoint to explicitly consume JSON by specifying `consumes = MediaType.APPLICATION_JSON_VALUE` on `@PostMapping`.
> - **Dependency Setup:** Ensure Jackson (`jackson-dataformat-xml` and standard Jackson JSON) dependencies are included in the project.
> - **Automatic Deserialization:** Spring MVC inspects the incoming `Content-Type: application/json` header and uses `MappingJackson2HttpMessageConverter` to deserialize the incoming JSON payload into the target Java DTO.
>
> **Key Nuance / Trade-off:**
> - If the client sends JSON but the controller specifies `consumes = MediaType.APPLICATION_XML_VALUE`, Spring will reject the request with an HTTP 415 Unsupported Media Type error.
>
> **Reference Concept:** `Content Negotiation & Payload Conversion (XML vs. JSON)`

---

### How do you efficiently convert an XML payload to JSON in a Java backend?

> **Direct Answer:**
> - **Jackson Tree Parsing:** Parse the XML string into an abstract, generic `JsonNode` tree using Jackson's `XmlMapper`.
> - **Direct JSON Serialization:** Immediately serialize that `JsonNode` tree into a formatted JSON string using Jackson's standard `ObjectMapper`.
> - **Dynamic Pipeline:** Eliminates the need to create intermediate Java POJO classes for transient payload translations.
>
> **Key Nuance / Trade-off:**
> - Converting XML to JSON dynamically can encounter type ambiguity (e.g., single XML elements vs. collections of one element); XML arrays without explicit container wrappers may serialize as single JSON objects unless configured.
>
> **Reference Concept:** `Content Negotiation & Payload Conversion (XML vs. JSON)`

---

### How do you handle a requirement where the client expects API error messages to be returned in their specific language?

> **Direct Answer:**
> - **Header Extraction:** Spring's `AcceptHeaderLocaleResolver` extracts the client's language preference from the HTTP `Accept-Language` header (e.g., `fr-FR`).
> - **ThreadLocal Context:** Spring automatically binds the extracted `Locale` to `LocaleContextHolder` in the executing thread.
> - **MessageSource Resolution:** Inject Spring's `MessageSource` to look up localized message strings from `messages_{locale}.properties` resource bundles.
> - **Centralized Advice:** Format the translated message into a standardized JSON response inside `@RestControllerAdvice`.
>
> **Key Nuance / Trade-off:**
> - `LocaleContextHolder` stores locale data in `ThreadLocal`; if processing is offloaded to `@Async` thread pools or reactive pipelines, the locale context is lost unless captured and passed explicitly to worker threads.
>
> **Reference Concept:** `Internationalization (i18n) & Localized Messages`

---

### How will you return the custom message per client in spring response?

> **Direct Answer:**
> - **Declarative Field Validation:** Enclose bundle keys inside `{}` in Jakarta Validation annotations (`@Size(message = "{user.size}")`), which `LocalValidatorFactoryBean` resolves automatically using the client's locale.
> - **Programmatic Service Exceptions:** Throw custom domain exceptions carrying translation keys and dynamic argument parameters, resolved via `MessageSource.getMessage(key, args, locale)`.
> - **Standardized JSON Formatting:** Intercept errors in `@RestControllerAdvice` and serialize localized messages into consistent response payloads (such as RFC 7807 `ProblemDetail`).
>
> **Key Nuance / Trade-off:**
> - Always supply a default message fallback in `messages.properties` or inside `messageSource.getMessage(...)` to prevent `NoSuchMessageException` when an unconfigured locale is requested.
>
> **Reference Concept:** `Internationalization (i18n) & Localized Messages`

---

### How do you capture a userId from an API request, and if it is invalid, how do you return a custom error message with a 404 Not Found status?

> **Direct Answer:**
> - **Capture via PathVariable:** Annotate the controller method argument with `@PathVariable String userId` on `@GetMapping("/{userId}")`.
> - **Throw Domain Exception:** If the service lookup returns empty (`userRepository.findById(userId).orElseThrow(...)`), throw a custom `UserNotFoundException`.
> - **Intercept with ControllerAdvice:** Intercept `UserNotFoundException` in a `@RestControllerAdvice` class annotated with `@ExceptionHandler(UserNotFoundException.class)`.
> - **Return 404 Structure:** Construct and return an `ApiError` record or `ProblemDetail` with an HTTP status of `HttpStatus.NOT_FOUND` (404).
>
> **Key Nuance / Trade-off:**
> - Throwing domain exceptions from the service layer keeps HTTP status codes out of business logic, while `@RestControllerAdvice` guarantees that clients receive structured JSON rather than default 500 error stack traces.
>
> **Reference Concept:** `REST Validation, PathVariables & 404 Error Handling`

---

### If one of your microservices is down, what type of response do you send? Are you hard coding it?

> **Direct Answer:**
> - **Never Hard-Code Failures:** Never hard-code static error responses or allow threads to block waiting for dead downstream microservices.
> - **Circuit Breaker Protection:** Implement a Circuit Breaker (using Resilience4j) that trips and opens when failure thresholds are exceeded, halting outgoing requests and saving thread pools.
> - **Strategy A - Fail Fast:** If downstream data is mandatory (e.g., payment checkout), return `503 Service Unavailable` or `504 Gateway Timeout` with a structured `ProblemDetail`.
> - **Strategy B - Graceful Degradation:** If data is optional (e.g., product recommendations), route to a fallback method returning `200 OK` with safe defaults, empty collections, or cached data.
>
> **Key Nuance / Trade-off:**
> - Failing fast preserves backend system stability, whereas graceful degradation preserves frontend user experience; the choice depends on whether the failing microservice is a critical or non-critical business dependency.
>
> **Reference Concept:** `Microservice Resilience & Circuit Breakers (Resilience4j)`

---

### What are Spring Boot Actuator endpoints, and how would you expose and secure them in production?

> **Direct Answer:**
> - **Operational Purpose:** `spring-boot-starter-actuator` provides built-in production endpoints under `/actuator` for health checks (`/actuator/health`), build/Git metadata (`/actuator/info`), and Micrometer telemetry (`/actuator/metrics` and `/actuator/prometheus`, the latter requiring `micrometer-registry-prometheus`).
> - **Production Monitoring Setup:**
>   1. **Explicit Allow-List:** Keep Spring Boot 3's safe default (`health` only) and explicitly allow-list only `health,info,prometheus` via `management.endpoints.web.exposure.include`—never use `*` in production.
>   2. **Health Probes:** Permit unauthenticated access to `/actuator/health` (and `/liveness` / `/readiness` probes) for load balancers and Kubernetes while keeping `management.endpoint.health.show-details=when_authorized` so internal dependency details are not leaked publicly.
>   3. **Authorized Metrics Scraping:** Protect `/actuator/prometheus` and `/actuator/info` in `SecurityFilterChain` using `EndpointRequest` matchers requiring a dedicated monitoring role/token (or bind management endpoints to an internal `management.server.port` behind a firewall).
>   4. **Keep Diagnostic Endpoints Restricted:** Leave `heapdump`, `threaddump`, `env`, `configprops`, `beans`, `mappings`, `loggers`, and `shutdown` unexposed over HTTP to prevent secret exposure, reconnaissance, and JVM pause DoS risks.
>
> **Key Nuance / Trade-off:**
> - **Three Separate Gates:** An endpoint must pass **availability/access** (bean enabled in context; `management.endpoint.<id>.access` in Boot 3.4+ vs. `enabled` in Boot 2.x–3.3), **HTTP exposure** (`management.endpoints.web.exposure.include`), and **Spring Security authorization**. Exposing an endpoint without configuring Spring Security locks it down to authenticated users (or rejects it), whereas exposing `*` with `permitAll()` creates a critical security leak.
>
> **Reference Concept:** [Spring Boot Actuator: Endpoints, Exposure & Security](<./Notes - Spring - 2. Auto Config, Components.md#spring-boot-actuator-endpoints-exposure--security>)

---

### Your application needs to fetch weather data from an external API (example: OpenWeather). How do you make HTTP requests?

> **Direct Answer:**
> - **`RestClient` (Spring 6.1+ Standard):** Use the modern, synchronous, fluent `RestClient` for standard Spring Boot MVC applications, avoiding deprecated `RestTemplate`.
> - **Builder Injection:** Inject `RestClient.Builder` to preserve distributed tracing contexts, set base URLs, and retrieve response bodies into DTOs (`.retrieve().body(WeatherResponse.class)`).
> - **`WebClient` for High Concurrency:** Use reactive `WebClient` if the application handles high concurrency or runs on Spring WebFlux.
> - **Spring Cloud OpenFeign:** Use OpenFeign for declarative, interface-driven HTTP clients where Spring auto-generates the HTTP implementation at runtime.
>
> **Key Nuance / Trade-off:**
> - `RestTemplate` is officially in maintenance mode and should not be used for greenfield development; `RestClient` offers the modern fluent syntax of `WebClient` without requiring a reactive WebFlux dependency.
>
> **Reference Concept:** `External HTTP Service Integration (RestClient & WebClient)`

---

### Your API returns JSON responses, but you need to hide some fields (like passwords) while exposing others. How do you do it?

> **Direct Answer:**
> - **The DTO Pattern (Industry Standard):** Never return database entities directly; map entities to Response DTOs that physically lack sensitive fields (e.g., `UserResponseDto` without a password field), guaranteeing they cannot be leaked.
> - **Jackson `@JsonIgnore`:** Annotate sensitive fields with `@JsonIgnore` to exclude them entirely from both serialization and deserialization.
> - **Jackson Write-Only Access:** If the client must send a password during registration but the API must hide it in response payloads, annotate the field with `@JsonProperty(access = Access.WRITE_ONLY)`.
>
> **Key Nuance / Trade-off:**
> - While Jackson annotations are convenient for shared DTOs, dedicated Response and Request DTOs provide compile-time safety and prevent accidental field exposure during model refactoring.
>
> **Reference Concept:** `Hiding JSON Fields & DTO Serialization`

---

### You need to implement an API that allows users to upload large files. How do you handle it?

> **Direct Answer:**
> - **Avoid Heap Buffering:** Never read `MultipartFile.getBytes()` directly into memory, as large files will instantly crash the JVM with `OutOfMemoryError`.
> - **Configure Servlet Limits:** Configure max request and file size limits in `application.properties` (`spring.servlet.multipart.max-file-size=2GB`).
> - **Stream Payloads:** Capture the `MultipartFile` in the controller and stream its `InputStream` directly to local disk or cloud object storage without holding the full byte array in heap.
> - **Architectural Solution (Pre-Signed URLs):** In cloud architectures, bypass the Spring backend entirely by authenticating the user and generating an AWS S3 Pre-Signed URL, allowing the client browser to upload large files directly to S3.
>
> **Key Nuance / Trade-off:**
> - Direct backend streaming still consumes backend bandwidth and worker threads; generating Pre-Signed URLs offloads 100% of network I/O and storage routing to cloud infrastructure.
>
> **Reference Concept:** `Large File Uploads (Streaming & Pre-Signed URLs)`
