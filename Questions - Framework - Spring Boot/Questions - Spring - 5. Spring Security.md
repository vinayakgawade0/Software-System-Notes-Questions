# Spring Security, OAuth2 & Enterprise API Protection - Interview Q&A

### How do you secure REST APIs or microservices? (JWT, OAuth2, API Gateway)

> **Direct Answer:**
> - **Stateless Authentication:** Implement token-based authentication using JSON Web Tokens (JWT) or OAuth2 access tokens delivered in the `Authorization: Bearer <token>` header, eliminating server-side session state.
> - **Authorization & RBAC:** Enforce Role-Based Access Control (RBAC) at the endpoint level via `authorizeHttpRequests` and at the method level using `@PreAuthorize("hasRole('ADMIN')")`.
> - **Edge Gateway Validation:** Route traffic through an API Gateway (e.g., Spring Cloud Gateway) to validate tokens, enforce rate-limiting, and strip sensitive internal routing details before forwarding requests to backend microservices.
> - **Secure Coding Baselines:** Always enforce HTTPS encryption, sanitize request inputs, apply the principle of least privilege, and avoid leaking internal stack traces in responses.
>
> **Key Nuance / Trade-off:**
> - Centralizing authentication at an API Gateway simplifies downstream services, but service-to-service calls still require mutual TLS (mTLS) or the Token Relay Pattern to maintain end-to-end identity propagation.
>
> **Reference Concept:** `REST API & Microservice Protection Strategies (JWT, OAuth2, RBAC)`

---

### How do you configure Spring Security in a Spring Boot application?

> **Direct Answer:**
> - **Component-Based Architecture:** In modern Spring Boot 3+ / Spring Security 6, `WebSecurityConfigurerAdapter` is completely removed; security is configured by declaring discrete `@Bean` definitions, primarily a `SecurityFilterChain`.
> - **Lambda DSL Configuration:** Chain security rules using the mandatory Lambda DSL (`csrf -> csrf.disable()`, `session -> session.sessionCreationPolicy(STATELESS)`).
> - **Stateless REST Policy:** Disable CSRF (since stateless token APIs are immune to cookie-based forgery) and configure `SessionCreationPolicy.STATELESS` so the server never maintains in-memory HTTP sessions.
>
> **Key Nuance / Trade-off:**
> - Setting sessions to `STATELESS` ensures horizontal scalability across microservice pods behind a load balancer, but places the responsibility on clients to provide a valid token with every individual HTTP request.
>
> **Reference Concept:** `Modern Component-Based Security Architecture (SecurityFilterChain vs. WebSecurityConfigurerAdapter)`

---

### Explain JWT authentication flow in detail.

> **Direct Answer:**
> - **Phase 1 (Login & Issuance):** The client sends credentials via `POST /login`. The Auth Server verifies them against the database, generates a signed JWT containing claims (userId, roles, expiration), and returns it to the client for secure storage.
> - **Phase 2 (Subsequent Access):** The client attaches the token in the `Authorization: Bearer <token>` header on every subsequent request.
> - **Stateless Verification:** The Resource Server extracts the JWT, verifies the cryptographic signature mathematically using its secret/public key, validates expiration, and populates `SecurityContextHolder` without querying a database or shared session cache.
>
> **Key Nuance / Trade-off:**
> - JWTs cannot be easily revoked before expiration without maintaining a token blocklist cache (e.g., in Redis); keep access token lifetimes short (e.g., 15 minutes) and pair with secure refresh token rotation.
>
> **Reference Concept:** `End-to-End JWT Authentication Lifecycle (Login to Verification)`

---

### How do you implement OAuth2 in Spring Security for microservices?

> **Direct Answer:**
> - **Decoupled Architecture:** Separate concerns between a central Authorization Server (Keycloak, Auth0) that authenticates users and issues tokens, and backend microservices that act as pure Resource Servers.
> - **Resource Server Setup:** Add `spring-boot-starter-oauth2-resource-server`, configure `.oauth2ResourceServer(oauth2 -> oauth2.jwt())`, and declare the `spring.security.oauth2.resourceserver.jwt.issuer-uri` in YAML.
> - **JWKS Key Caching:** Spring Boot automatically downloads public keys from the provider's JWKS endpoint at startup to validate token signatures statelessly without hitting the Auth Server per request.
>
> **Key Nuance / Trade-off:**
> - **Token Relay for Microservices:** For downstream service-to-service calls, configure `WebClient` with `ServletBearerExchangeFilterFunction` to automatically propagate the incoming Bearer token to the next microservice.
> - **Role Mapping:** Third-party providers often nest roles (e.g., `realm_access.roles` in Keycloak); a custom `JwtAuthenticationConverter` is required to extract and map them to Spring's `ROLE_` authorities.
>
> **Reference Concept:** `OAuth2 Architecture: Authorization Server, Client Gateway & Resource Servers`

---

### What is Cross-Site Request Forgery (CSRF), and how do you protect a Spring Boot application against it?

> **Direct Answer:**
> - **Attack Mechanics:** CSRF exploits ambient browser authentication by tricking an authenticated user into executing unwanted state-changing actions on a site where their session cookies are automatically sent with cross-site requests.
> - **Stateless REST Immunity:** Stateless REST APIs that use `Authorization: Bearer <token>` headers are immune to CSRF because browsers never automatically attach custom authorization headers to cross-site requests.
> - **Implementation Strategy:** Disable CSRF (`csrf.disable()`) for stateless REST APIs to eliminate overhead; enable CSRF using `CookieCsrfTokenRepository.withHttpOnlyFalse()` only for stateful browser MVC applications that rely on session cookies.
>
> **Key Nuance / Trade-off:**
> - If an API stores JWTs in browser cookies rather than headers without setting `SameSite=Strict` or `HttpOnly`, CSRF vulnerability is reintroduced, requiring CSRF token validation to remain active.
>
> **Reference Concept:** `Cross-Site Request Forgery (CSRF): Mechanics & Stateless API Immunity`

---

### What is the HTTP OPTIONS method, and why is it critical for modern API performance?

> **Direct Answer:**
> - **CORS Pre-Flight Check:** Browsers automatically dispatch an HTTP `OPTIONS` request before complex cross-origin requests (e.g., JSON `POST` or custom headers) to verify server permissions.
> - **The Latency Penalty:** Without caching, every API call requires two network roundtrips (one for `OPTIONS`, one for the actual payload), doubling network latency.
> - **Max-Age Caching Solution:** Configure `maxAge` on the server's `CorsConfiguration` (e.g., `configuration.setMaxAge(3600L)`). This instructs the browser to cache pre-flight approvals for 1 hour, cutting future call latency in half.
>
> **Key Nuance / Trade-off:**
> - If Spring Security is present, it will intercept requests before Spring MVC; ensure CORS is configured inside `SecurityFilterChain` so pre-flight `OPTIONS` requests are not rejected with 401/403 errors.
>
> **Reference Concept:** `HTTP OPTIONS Pre-Flight Requests & CORS Performance Optimization (Max-Age Caching)`

---

### What is Role-Based Access Control (RBAC)? Explain @PreAuthorize and @PostAuthorize in Spring Security?

> **Direct Answer:**
> - **RBAC Model:** Decouples permissions from individual user identities by binding access rights to abstract Roles (e.g., `ROLE_ADMIN`), enabled via `@EnableMethodSecurity`.
> - **`@PreAuthorize` (Pre-Execution Guard):** Evaluates SpEL expressions **before** the method runs (e.g., `@PreAuthorize("hasRole('ADMIN')")`). If false, throws an `AccessDeniedException` (403) and blocks method execution entirely.
> - **`@PostAuthorize` (Post-Execution Auditor):** Evaluates SpEL expressions **after** the method executes, allowing security checks against the returned object via `returnObject` (e.g., checking if `returnObject.ownerId == authentication.principal.id`).
>
> **Key Nuance / Trade-off:**
> - **The Mutation Trap:** Because `@PostAuthorize` executes after the method body, write/delete database operations have already been executed and committed before the security check runs. Strictly restrict `@PostAuthorize` to read-only (`GET`) queries.
>
> **Reference Concept:** `@PreAuthorize vs. @PostAuthorize Mechanics & The Mutation Trap`
