# Spring MVC, Web Clients, AOP & Exception Handling - Interview Q&A

### Explain Spring MVC Architecture: DispatcherServlet → HandlerMapping → Controller → ViewResolver flow

> **Direct Answer:**
> - **Front Controller Pattern:** Spring MVC funnels all incoming HTTP traffic through a central gateway called the `DispatcherServlet`.
> - **HandlerMapping & Controller:** The `DispatcherServlet` queries `HandlerMapping` to route the URL to the target Controller, which executes domain logic and returns a Model and logical view name.
> - **ViewResolver Rendering:** For traditional MVC, the `DispatcherServlet` consults a `ViewResolver` to convert the logical name into a physical view template (e.g., JSP/Thymeleaf) and render the HTML response.
>
> **Key Nuance / Trade-off:**
> - **Modern API Bypass:** Annotating a controller with `@RestController` applies `@ResponseBody` across all methods, which completely bypasses the `ViewResolver` and routes Java return objects directly to `HttpMessageConverter` (Jackson) for JSON serialization.
>
> **Reference Concept:** `DispatcherServlet Request Lifecycle Flow`

---

### What is the low-level difference between spring-boot-starter-web and spring-boot-starter-webflux, and how do RestClient and WebClient operate within them?

> **Direct Answer:**
> - **Imperative vs. Reactive Stacks:** `starter-web` runs on a blocking Tomcat server using a Thread-Per-Request model, whereas `starter-webflux` runs on an asynchronous Netty event loop using a small, fixed thread pool.
> - **Low-Level Scheduling:** When waiting for I/O in `starter-web`, the OS physically blocks and suspends the thread. In `starter-webflux`, worker threads never wait; they register OS interrupt callbacks (e.g., Linux `epoll`) and immediately serve other traffic.
> - **Client Operations:** `RestClient` operates synchronously on blocking threads, halting execution until remote packets return. `WebClient` operates asynchronously, immediately returning a reactive publisher (`Mono` or `Flux`) that streams data via non-blocking Netty channels.
>
> **Key Nuance / Trade-off:**
> - `starter-web` provides simple debugging and thread-local traceability but exhausts thread pools under high-latency I/O loads. `starter-webflux` scales efficiently under high concurrent I/O but requires completely non-blocking drivers across the entire stack.
>
> **Reference Concept:** `Imperative vs. Reactive Web Stacks (starter-web vs. starter-webflux)`

---

### How do you implement asynchronous service calls using Spring WebClient, ensure resilience, and how do you integrate it into a legacy synchronous application?

> **Direct Answer:**
> - **Asynchronous Implementation:** Inject the auto-configured `WebClient.Builder` to preserve tracing contexts, define target endpoints, and return reactive streams (`Mono` or `Flux`) without blocking worker threads.
> - **Resilience Pipeline:** Chain reactive operators directly onto the pipeline—dynamically handle HTTP error statuses with `.onStatus()`, apply network timeouts with `.timeout()`, handle transient drops with `.retryWhen(Retry.backoff(...))`, and return graceful fallbacks with `.onErrorReturn()`.
>
> **Key Nuance / Trade-off:**
> - **The `.block()` Trap:** Calling `.block()` to extract raw objects in synchronous Tomcat applications freezes the caller thread, completely defeating the non-blocking performance advantages of WebClient and degrading it into an inefficient blocking client.
>
> **Reference Concept:** `WebClient: Asynchronous Resilient API Calls & Legacy Integration (.block() Trap)`

---

### Why is RestTemplate considered obsolete, and how does the modern RestClient serve as its direct replacement in synchronous Spring applications?

> **Direct Answer:**
> - **`RestTemplate` Obsolescence:** `RestTemplate` is clunky, heavily overloaded with overlapping method signatures, and based on legacy design principles; Spring officially placed it in maintenance mode in Spring 6.1.
> - **The Historical Dilemma:** Developers were previously forced to import the entire reactive `spring-boot-starter-webflux` stack just to access `WebClient`'s clean builder API in synchronous applications.
> - **`RestClient` Direct Replacement:** Introduced in Spring 6.1 / Spring Boot 3.2, `RestClient` runs purely synchronously on blocking threads while adopting the fluent, functional, builder-style API of `WebClient`.
>
> **Key Nuance / Trade-off:**
> - `RestClient` is not non-blocking; it uses standard blocking HTTP client libraries (e.g., JDK `HttpClient` or Apache `HttpComponents`) underneath, providing modern syntax without the complexity of a reactive runtime.
>
> **Reference Concept:** `RestClient: The Modern Synchronous HTTP Client (RestTemplate Deprecation)`

---

### What is Aspect-Oriented Programming (AOP), what specific architectural problem does it solve in Spring, and how do you implement it?

> **Direct Answer:**
> - **Cross-Cutting Concerns:** AOP eliminates infrastructure pollution (logging, security, auditing, transaction demarcation) duplicated across service layers by encapsulating it into modular classes called Aspects.
> - **Proxy-Based Interception:** Spring injects cross-cutting logic at runtime using the Proxy Pattern, intercepting method executions without modifying the underlying business source code.
> - **Implementation Model:** Define business services cleanly with target annotations, then implement an `@Aspect` class featuring an advice (e.g., `@Around`) targeted at that annotation's execution.
>
> **Key Nuance / Trade-off:**
> - In Spring AOP, join points are strictly limited to method executions on Spring-managed beans. Calls within the same target bean instance (self-invocation) bypass the proxy entirely.
>
> **Reference Concept:** `Cross-Cutting Concerns & AOP Core Problem`

---

### Explain the core terminology of Spring AOP: Aspect, Advice, Pointcut, and JoinPoint.

> **Direct Answer:**
> - **Aspect:** The class encapsulating the cross-cutting concern (annotated with `@Aspect`).
> - **Advice:** The action taken by an aspect at a join point; the method defining *what* runs and *when* (e.g., `@Before`, `@After`, `@Around`).
> - **Pointcut:** The predicate expression defining *where* the advice applies (e.g., execution of methods with `@annotation(LogExecutionTime)`).
> - **JoinPoint:** The specific point during program execution representing method invocation, exposing execution metadata, signatures, and runtime parameters.
>
> **Key Nuance / Trade-off:**
> - While AspectJ supports field modifications, constructor calls, and static initializers as join points, Spring AOP supports strictly public and protected bean method execution join points.
>
> **Reference Concept:** `Core AOP Terminology (Aspect, Advice, Pointcut, JoinPoint)`

---

### What are the different types of Advice in Spring AOP, and when would you specifically use @Around?

> **Direct Answer:**
> - **Advice Types:** Spring AOP provides `@Before`, `@After` (finally), `@AfterReturning` (success), `@AfterThrowing` (exception), and `@Around`.
> - **`@Around` Capabilities:** The most powerful advice type; it completely surrounds target method execution, taking a `ProceedingJoinPoint` parameter.
> - **Primary Use Cases:** Use `@Around` when you must control whether the target executes (e.g., caching, where cached values bypass `.proceed()`) or when tracking start-to-finish execution (e.g., stopwatch performance profiling).
>
> **Key Nuance / Trade-off:**
> - `@Around` methods must declare `throws Throwable` and explicitly return the output of `joinPoint.proceed()`, otherwise the target method's return value will be swallowed and replaced with null.
>
> **Reference Concept:** `Advice Types & @Around Mechanics`

---

### How does Spring AOP actually work under the hood? Explain the Proxy pattern.

> **Direct Answer:**
> - **Dynamic Proxy Generation:** When a bean requires AOP interception, Spring does not inject the raw instance into dependents. It generates and injects a dynamic runtime Proxy wrapping the target bean.
> - **Interception Pipeline:** The caller invokes the proxy, which executes `@Before` advices, delegates execution to the actual target bean, executes `@After` / `@AfterReturning` / `@AfterThrowing` advices, and returns the result.
> - **Proxy Mechanics:** Spring uses **JDK Dynamic Proxies** when the target class implements an interface, or **CGLIB Subclasses** when targeting concrete classes without interfaces (Spring Boot defaults to CGLIB via `proxyTargetClass=true`).
>
> **Key Nuance / Trade-off:**
> - Because CGLIB generates runtime subclasses, target classes and target methods must not be declared `final`, otherwise the proxy generator cannot override them.
>
> **Reference Concept:** `AOP Under the Hood: JDK Dynamic Proxies vs. CGLIB Subclasses`

---

### Your application fetches data from a slow database. How do you cache results to improve performance?

> **Direct Answer:**
> - **Declarative Caching:** Annotate target service methods with `@Cacheable("cacheName")`, which automatically activates Spring's internal caching AOP interceptor.
> - **Cache Interception:** An AOP proxy intercepts the method call and inspects the cache provider (e.g., Redis, Caffeine) using method parameters as the cache key.
> - **Cache Hit vs. Miss:** On a Cache Hit, the proxy short-circuits the call and returns cached data instantly without touching the database. On a Cache Miss, it delegates to the real method, stores the return value in the cache, and returns it.
>
> **Key Nuance / Trade-off:**
> - Cached methods must ensure arguments implement reliable `equals()` and `hashCode()` contracts to avoid cache collisions or key misses. Methods called internally from the same class bypass the cache proxy.
>
> **Reference Concept:** `Real-World AOP Use Case: Caching with @Cacheable`

---

### How would you implement exception handling in a REST service? Explain @ControllerAdvice and @ExceptionHandler

> **Direct Answer:**
> - **Controller-Level (`@ExceptionHandler`):** Place methods annotated with `@ExceptionHandler(ExceptionClass.class)` inside controllers to handle errors thrown specifically within that class.
> - **Global Centralization (`@RestControllerAdvice`):** Consolidate error handling across the entire API by creating a dedicated global handler annotated with `@RestControllerAdvice`.
> - **Standardized JSON Contracts:** The global advice intercepts uncaught exceptions across all controllers, logs errors, and returns a predictable, structured error payload with an appropriate HTTP status code (e.g., 400 Bad Request, 404 Not Found).
>
> **Key Nuance / Trade-off:**
> - If an exception matches an `@ExceptionHandler` inside the calling `@Controller` and an `@ExceptionHandler` in `@RestControllerAdvice`, the local controller-level handler takes precedence.
>
> **Reference Concept:** `Controller-Level vs. Global Exception Handling (@ExceptionHandler vs. @ControllerAdvice)`

---

### How do you handle global exception handling in Spring Boot? Explain @RestControllerAdvice and ProblemDetail

> **Direct Answer:**
> - **Enterprise Architecture:** Annotate a global exception handler class with `@RestControllerAdvice` and extend `ResponseEntityExceptionHandler` to automatically format all standard Spring MVC framework exceptions.
> - **RFC 7807 / RFC 9457 ProblemDetail:** Return Spring Boot 3's `ProblemDetail` object instead of building custom error DTOs, standardizing API error responses across the industry with fields: `type`, `title`, `status`, `detail`, and `instance`.
> - **Global Property Activation:** Set `spring.mvc.problemdetails.enabled=true` in `application.properties` to ensure built-in framework validation and routing errors automatically match the RFC standard format.
>
> **Key Nuance / Trade-off:**
> - Extending `ResponseEntityExceptionHandler` allows overriding specific hooks (such as `handleMethodArgumentNotValid`) to customize field validation error arrays inside the `ProblemDetail` properties map.
>
> **Reference Concept:** `RFC 7807 / RFC 9457 Problem Details & Modern Error Contracts`

---

### What is the exact difference between @ControllerAdvice and @RestControllerAdvice in Spring exception handling, and how does Spring handle the data returned by each?

> **Direct Answer:**
> - **Rendering Target:** `@ControllerAdvice` is designed for server-side HTML template rendering, whereas `@RestControllerAdvice` is designed for RESTful JSON/XML API responses.
> - **ViewResolver vs. ResponseBody:** Returning an object from `@ControllerAdvice` delivers it to a `ViewResolver` to look up an HTML template. `@RestControllerAdvice` is a meta-annotation composed of `@ControllerAdvice` and `@ResponseBody`.
> - **JSON Serialization:** In `@RestControllerAdvice`, `@ResponseBody` bypasses the `ViewResolver` entirely and routes the error object directly through `HttpMessageConverter` (Jackson) to serialize into the HTTP response body.
>
> **Key Nuance / Trade-off:**
> - Using `@ControllerAdvice` for an API error will cause Spring to attempt view resolution on the returned error object or string, leading to a `Circular view path` error or 500 internal server error unless `@ResponseBody` is manually appended to every method.
>
> **Reference Concept:** `@ControllerAdvice vs. @RestControllerAdvice Mechanics`
