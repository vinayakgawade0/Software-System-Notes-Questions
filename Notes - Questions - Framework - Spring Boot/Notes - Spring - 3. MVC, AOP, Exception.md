# Spring MVC, Web Clients, AOP & Exception Handling

## Spring MVC & Web Layer Architecture

### DispatcherServlet Request Lifecycle Flow
- **Front Controller Pattern:** Spring MVC funnels every incoming HTTP request through a central entry point called the `DispatcherServlet`.
- **Step-by-Step Execution Sequence:**
  1. **Receipt:** The `DispatcherServlet` receives the raw HTTP request from the embedded web server (e.g., Tomcat).
  2. **Routing:** It queries the `HandlerMapping` (the Router) to determine which Controller method maps to the requested URL and HTTP method.
  3. **Execution:** The selected `Controller` executes business logic and returns model data along with a logical view name (e.g., `"user-list"`).
  4. **Resolution:** The `DispatcherServlet` passes the logical view name to a `ViewResolver` (the Translator) to locate the physical template path (e.g., `/WEB-INF/views/user-list.jsp` or Thymeleaf template).
  5. **Rendering:** The `DispatcherServlet` binds the model data to the template and renders the final HTML response sent to the client.
- **Modern REST API Bypass:** Annotating a class with `@RestController` (meta-annotation combining `@Controller` and `@ResponseBody`) completely bypasses the `ViewResolver`. The `DispatcherServlet` delivers the returned Java object directly to an `HttpMessageConverter` (like Jackson) to serialize it into JSON.

### Imperative vs. Reactive Web Stacks (starter-web vs. starter-webflux)
- **`spring-boot-starter-web` (Imperative / Blocking):**
  - **Server Engine:** Apache Tomcat by default.
  - **Thread-Per-Request Model:** Each incoming HTTP connection is allocated a dedicated OS-level worker thread.
  - **I/O Blocking:** When querying a database or external microservice, the CPU scheduler physically suspends the thread until I/O completes. Under high concurrent traffic, thread pools become exhausted, causing thread starvation, high memory footprints, and latency spikes.
- **`spring-boot-starter-webflux` (Reactive / Non-Blocking):**
  - **Server Engine:** Netty by default.
  - **Event Loop Model:** Operates on an Event Loop architecture driven by a small, fixed pool of worker threads (typically equal to CPU cores).
  - **Non-Blocking I/O:** When waiting for I/O, the Event Loop registers an asynchronous OS kernel interrupt callback (e.g., Linux `epoll`) and immediately frees the thread to serve other requests. No thread ever sits idle waiting for I/O.
- **Client Execution Differences:**
  - **`RestClient`:** Operates in the imperative, blocking stack. Calling `.retrieve()` halts the calling thread until network packets arrive.
  - **`WebClient`:** Operates in the reactive stack. Immediately returns a reactive publisher (`Mono` or `Flux`) without blocking. Reactor Netty asynchronous channels stream data when the remote server responds.

### WebClient: Asynchronous Resilient API Calls & Legacy Integration (.block() Trap)
- **Implementation:** Injected via the auto-configured `WebClient.Builder` to preserve distributed tracing contexts (Micrometer/Zipkin/Brave) and returns reactive streams (`Mono<T>` for single items, `Flux<T>` for multi-item streams).
- **Resilience Chaining:** Unchecked HTTP calls fail abruptly if downstream services fail. Defensive operators are chained directly onto the stream:
  ```java
  public Mono<UserDto> fetchUserResilient(String userId) {
      return webClient.get()
          .uri("/users/{id}", userId)
          .retrieve()
          // 1. Handle HTTP errors dynamically
          .onStatus(HttpStatusCode::is4xxClientError, response -> Mono.error(new UserNotFoundException("Invalid ID")))
          .onStatus(HttpStatusCode::is5xxServerError, response -> Mono.error(new ExternalServiceException("Downstream dead")))
          .bodyToMono(UserDto.class)
          // 2. Network timeout (Abandon if too slow)
          .timeout(Duration.ofSeconds(3))
          // 3. Retry on transient failures with Exponential Backoff
          .retryWhen(Retry.backoff(3, Duration.ofSeconds(1)))
          // 4. Fallback (Return default if everything fails)
          .onErrorReturn(new UserDto("default_id", "Guest User"));
  }
  ```
- **Legacy Integration & The `.block()` Trap:**
  - In a synchronous, blocking application (Tomcat), developers often call `.block()` to extract the raw Java object (e.g., `fetchUserResilient("123").block()`).
  - *The Trap:* Calling `.block()` physically freezes the executing Tomcat thread, completely forfeiting the non-blocking and throughput benefits of `WebClient`. It turns `WebClient` into an awkward blocking client and should only be used as a temporary bridge in legacy migrations.

### RestClient: The Modern Synchronous HTTP Client (RestTemplate Deprecation)
- **`RestTemplate` Deprecation:** In Spring 6.1 / Spring Boot 3.2, `RestTemplate` was placed into maintenance mode. Its API is overloaded, clunky, and inconsistent with modern functional style.
- **Historical Workaround:** Prior to Spring 6.1, developers were forced to import the entire reactive `spring-boot-starter-webflux` dependency solely to get `WebClient`'s clean builder API, only to call `.block()` in synchronous applications.
- **`RestClient` Direct Replacement:** Introduced in Spring 6.1 specifically for synchronous Spring MVC applications. It runs synchronously on standard blocking threads while adopting the fluent, functional builder API of `WebClient`:
  ```java
  @Service
  public class SyncUserService {
      private final RestClient restClient;

      public SyncUserService(RestClient.Builder builder) {
          this.restClient = builder.baseUrl("https://api.external.com").build();
      }

      public UserDto fetchUser(String userId) {
          // Modern, fluent syntax, but purely synchronous and blocking!
          return restClient.get()
              .uri("/users/{id}", userId)
              .retrieve()
              .body(UserDto.class);
      }
  }
  ```

---

## Spring AOP (Aspect-Oriented Programming)

### Cross-Cutting Concerns & AOP Core Problem
- **The Architectural Problem:** Core business services frequently get polluted by cross-cutting concerns—repetitive infrastructure logic such as logging, security authorization, auditing, metrics, and transaction boundaries duplicated across layers.
- **The Solution:** AOP abstracts infrastructure logic out of business services into dedicated modular units called **Aspects**. Spring injects this logic at runtime using the **Proxy Pattern** without modifying business classes.

### Core AOP Terminology (Aspect, Advice, Pointcut, JoinPoint)
- **Aspect:** A modular class encapsulating a cross-cutting concern (annotated with `@Aspect`).
- **Advice:** The action taken by an aspect at a particular join point. The actual Java method defining *what* code executes and *when* (`@Before`, `@After`, `@Around`, etc.).
- **Pointcut:** A predicate or expression defining *where* the advice applies (e.g., specific package paths, method execution patterns, or custom annotations like `@annotation(...)`).
- **JoinPoint:** A candidate point in program execution (in Spring AOP, always method executions). Provides runtime metadata to inspect the executing method signature, arguments, and target instance.

### Advice Types & @Around Mechanics
- **Standard Advice Types:**
  - `@Before`: Executes prior to join point invocation.
  - `@After`: Executes after method completion regardless of outcome (like a `finally` block).
  - `@AfterReturning`: Executes strictly after successful method completion.
  - `@AfterThrowing`: Executes strictly if the method throws an exception.
- **`@Around` Advice:**
  - The most comprehensive and powerful advice type. It surrounds the join point completely.
  - Requires a `ProceedingJoinPoint` parameter and an explicit call to `.proceed()` to pass execution to the actual target method.
  - Can inspect, modify, or replace method parameters and return values, or halt execution entirely (e.g., returning cached values without calling `.proceed()`).

### AOP Under the Hood: JDK Dynamic Proxies vs. CGLIB Subclasses
- **Proxy Interception:** When a bean with active AOP aspects is injected, Spring does not inject the raw bean instance. It injects a generated runtime **Proxy** that intercepts calls, invokes relevant advices, delegates to the target bean, and processes responses.
- **Proxy Implementation Strategies:**
  - **JDK Dynamic Proxies:** Used by default when target classes implement interfaces. The proxy implements the interface and wraps the target instance via reflection.
  - **CGLIB Proxies:** Generates dynamic bytecode subclasses at runtime. Used when the target class does not implement an interface, or when configured as default (Spring Boot uses CGLIB subclasses by default via `proxyTargetClass=true`).

### AOP Implementation: Annotation-Driven Performance Logging Interceptor
- **Step 1: Clean Business Service:**
  ```java
  // 1. The Clean Business Logic
  @Service
  public class AccountService {

      @LogExecutionTime // Custom annotation triggers the AOP interceptor
      @Transactional
      public void transferFunds(String fromAcc, String toAcc, double amount) {
          // Only core business logic remains!
          accountRepo.deduct(fromAcc, amount);
          accountRepo.add(toAcc, amount);
      }
  }
  ```
- **Step 2: The Interceptor Aspect:**
  ```java
  // 2. The Aspect (The Interceptor)
  @Aspect
  @Component
  @Slf4j
  public class PerformanceLoggingAspect {

      // The @Around advice completely wraps the target method
      @Around("@annotation(com.example.LogExecutionTime)")
      public Object logExecutionTime(ProceedingJoinPoint joinPoint) throws Throwable {
          long start = System.currentTimeMillis();

          // proceed() is explicitly called to execute the real transferFunds() method
          Object result = joinPoint.proceed();

          long executionTime = System.currentTimeMillis() - start;
          log.info("{} executed in {}ms", joinPoint.getSignature().getName(), executionTime);

          return result;
      }
  }
  ```

### Real-World AOP Use Case: Caching with @Cacheable
- **Declarative Caching:** Placing `@Cacheable("users")` on a data access method activates Spring's internal caching aspect.
- **Execution Mechanism:**
  - The generated AOP proxy intercepts the invocation and constructs a key from method arguments.
  - **Cache Hit:** The proxy short-circuits the call, returns the cached result (e.g., from Redis/Caffeine), and never executes the expensive database query.
  - **Cache Miss:** The proxy invokes `.proceed()`, queries the database, puts the fresh result into the cache, and returns it to the caller.
- **Architectural Advantage:** Strictly enforces separation of concerns—business and persistence layers remain completely agnostic of caching technologies and invalidation boilerplate.

---

## Global Exception Handling & REST API Architecture

### Controller-Level vs. Global Exception Handling (@ExceptionHandler vs. @ControllerAdvice)
- **Controller-Level (`@ExceptionHandler`):** Handlers placed inside individual controller classes handle exceptions thrown strictly within that specific controller. Useful for narrow, controller-specific logic, but leads to duplication across an enterprise API.
- **Global Centralized Handling (`@RestControllerAdvice`):**
  - Consolidates error handling into a centralized global interceptor.
  - Catches domain exceptions thrown anywhere across controllers, logs details, and transforms them into standard, structured JSON error responses with consistent HTTP status codes.

### RFC 7807 / RFC 9457 Problem Details & Modern Error Contracts
- **`ProblemDetail` (Spring Boot 3 / Spring 6):**
  - Standardized representation for HTTP API errors conforming to RFC 7807 / RFC 9457.
  - Eliminates the need to maintain custom `ErrorResponse` DTO classes across microservices.
  - Automatically provides standardized JSON fields: `type`, `title`, `status`, `detail`, and `instance`.
- **Enabling Globally:**
  - Extend `ResponseEntityExceptionHandler` in `@RestControllerAdvice` to automatically format all standard Spring MVC framework exceptions (404, 405, 400).
  - Enable framework integration in `application.properties`:
    ```properties
    spring.mvc.problemdetails.enabled=true
    ```

### @ControllerAdvice vs. @RestControllerAdvice Mechanics
- **`@ControllerAdvice` (HTML / MVC Views):**
  - Standard web controller advice. When an exception handler returns an object or string, it passes the data to a `ViewResolver` to render a physical HTML template (e.g., an error JSP or Thymeleaf error view).
- **`@RestControllerAdvice` (REST APIs / JSON):**
  - Meta-annotation combining `@ControllerAdvice` and `@ResponseBody`.
  - Completely bypasses the `ViewResolver` and routes the returned Java error object directly to an `HttpMessageConverter` (like Jackson) to serialize into raw JSON written directly to the HTTP response stream.

## REST API Design Evolution and Governance

### Resource Naming and URI Modeling
- **Resource-Oriented Nouns:** Model URIs around plural domain nouns representing resources, never RPC-style action verbs:
  - Good: `GET /api/v1/orders/{orderId}/items`
  - Avoid: `POST /api/v1/getOrderItems` or `POST /api/v1/deleteOrder`
- **Hierarchical Nesting & Query Parameters:**
  - Use path segments for strict parent-child ownership (`/customers/{customerId}/accounts`), limiting nesting depth to 2–3 levels.
  - Use query parameters for filtering, sorting, and pagination (`GET /api/v1/orders?status=CONFIRMED&sort=createdAt,desc&page=0&size=20`).
- **Consistency Conventions:** Use lowercase kebab-case (`/order-items`), consistent pluralization, and stable opaque identifiers (UUIDs or public IDs rather than internal auto-increment sequence numbers).

### HTTP Methods Idempotency and Status Codes
- **Method Semantics & Idempotency:**
  | HTTP Method | Operation | Safe | Idempotent | Expected Success Status |
  | :--- | :--- | :--- | :--- | :--- |
  | **`GET`** | Retrieve resource(s) | Yes | Yes | `200 OK` |
  | **`POST`** | Create resource / non-idempotent action | No | No | `201 Created` (`Location` header) or `202 Accepted` |
  | **`PUT`** | Full replacement of an existing resource | No | Yes | `200 OK` or `204 No Content` |
  | **`PATCH`** | Partial update (JSON Patch / Merge Patch) | No | Conditional | `200 OK` |
  | **`DELETE`** | Remove resource | No | Yes | `204 No Content` |
  - *Idempotency Key Pattern:* For non-idempotent `POST` endpoints (e.g., payment creation), accept an `Idempotency-Key` header so client retries safely return the original response without duplicate execution.
- **Client & Server Error Status Codes:**
  - `400 Bad Request`: Malformed JSON syntax, invalid parameter type, or domain validation failure.
  - `401 Unauthorized`: Missing, expired, or invalid authentication credentials/JWT.
  - `403 Forbidden`: Authenticated identity lacks required role/scope/ownership permissions.
  - `404 Not Found`: Target resource does not exist.
  - `409 Conflict`: Optimistic locking version mismatch (`@Version`) or duplicate unique key.
  - `422 Unprocessable Entity`: Syntactically valid payload violating business rules.
  - `429 Too Many Requests`: Rate-limit quota exceeded (include `Retry-After` header).
  - `500 Internal Server Error` / `503 Service Unavailable`: Unhandled server fault or temporary downstream dependency outage.

### Backward-Compatible Evolution and Versioning Strategies
- **Non-Breaking (Additive) Changes — No Version Bump Required:**
  - Adding optional fields to JSON response payloads (clients must ignore unknown properties).
  - Adding new optional query parameters or new endpoints.
  - Loosening validation constraints on existing request fields.
- **Breaking Changes — Require New Major Version:**
  - Removing or renaming existing request/response fields, changing field data types, tightening validation rules, or altering error codes/semantics.
- **Versioning Approaches:**
  1. **URI Path Versioning (`/api/v1/orders` → `/api/v2/orders`):** Most explicit, easily routable at API gateways/CDNs, and unambiguous in OpenAPI specs and access logs.
  2. **Custom Header Versioning (`X-API-Version: 2`):** Preserves clean resource URIs while routing via Spring MVC `@GetMapping(headers = "X-API-Version=2")` or gateway rules.
  3. **Media Type / Content Negotiation (`Accept: application/vnd.company.order.v2+json`):** Adheres strictly to REST hypermedia/representation principles without altering URIs.
- **Implementation & Deprecation Hygiene:** Keep version differences isolated in controller DTOs and mappers while sharing the underlying service layer; announce deprecation via `Deprecation` and `Sunset` HTTP response headers before retiring old versions.

### Standardized Error Contract RFC 7807 ProblemDetail
- **Uniform Error Payload:** Return a consistent JSON schema across all controllers and security filters using Spring Framework's native `ProblemDetail` (RFC 7807 / RFC 9457):
  ```json
  {
    "type": "[https://api.example.com/errors/validation-failed](https://api.example.com/errors/validation-failed)",
    "title": "Request Validation Failed",
    "status": 400,
    "detail": "One or more request fields failed validation.",
    "instance": "/api/v1/orders",
    "timestamp": "2026-11-27T08:00:00Z",
    "traceId": "4bf92f3577b34da6",
    "fieldErrors": [
      { "field": "quantity", "message": "must be greater than 0" }
    ]
  }
  ```
- **Centralized Enforcement in Spring Boot:**
  - Handle controller and validation exceptions (`MethodArgumentNotValidException`, `HttpMessageNotReadableException`, custom domain exceptions) in a single `@RestControllerAdvice` class.
  - Handle filter-level Spring Security exceptions (`401`/`403`) via custom `AuthenticationEntryPoint` and `AccessDeniedHandler` so security failures return the exact same `ProblemDetail` structure.
  - Never expose stack traces, SQL errors, or internal class names; attach a `traceId` / correlation ID for log lookup.
