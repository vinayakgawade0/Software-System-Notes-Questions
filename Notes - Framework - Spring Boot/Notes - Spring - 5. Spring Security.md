# Spring Security, OAuth2 & Enterprise API Protection

## Core Security Architecture & Component Model

### Modern Component-Based Security Architecture (SecurityFilterChain vs. WebSecurityConfigurerAdapter)
- **Deprecation of `WebSecurityConfigurerAdapter`:** In Spring Boot 2.7 / Spring Security 5.7 and fully removed in Spring Boot 3 / Security 6, class inheritance via `WebSecurityConfigurerAdapter` is completely replaced by a component-based model.
- **Component Model:** Security configuration is declared via discrete `@Bean` definitions, primarily returning a `SecurityFilterChain`.
- **Lambda DSL Standard:** Configuration requires the Lambda DSL syntax (`csrf -> csrf.disable()`), providing explicit scoping and preventing chained configuration ambiguities.
- **Production Implementation:**
  ```java
  @Configuration
  @EnableWebSecurity
  public class SecurityConfig {

      @Bean
      public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
          http
              // 1. Disable CSRF for stateless REST APIs
              .csrf(csrf -> csrf.disable())

              // 2. Configure CORS
              .cors(cors -> cors.configure(http))

              // 3. Define Authorization Rules
              .authorizeHttpRequests(auth -> auth
                  .requestMatchers("/api/auth/login", "/api/public/**").permitAll()
                  .requestMatchers("/api/admin/**").hasRole("ADMIN")
                  .anyRequest().authenticated()
              )

              // 4. Force Stateless Sessions
              .sessionManagement(session -> session
                  .sessionCreationPolicy(SessionCreationPolicy.STATELESS)
              );

          return http.build();
      }
  }
  ```

### Spring Security Filter Chain & Stateless REST Execution
- **The Filter Chain Gauntlet:** Under the hood, Spring Security executes as a chain of standard Servlet Filters managed by `FilterChainProxy` (delegated via `DelegatingFilterProxy`).
- **Execution Order:** Incoming HTTP requests pass sequentially through filters (CORS -> CSRF -> Authentication -> Authorization) before dispatching to `@RestController`. If any filter rejects the request, execution halts immediately with an HTTP 401 (Unauthorized) or 403 (Forbidden).
- **Stateless Session Management (`SessionCreationPolicy.STATELESS`):**
  - By default, Spring Security creates an `HttpSession` and caches user authentication state in memory (`SecurityContext` in `HttpSession`).
  - In distributed microservices behind load balancers, shared server sessions create state bottlenecks.
  - Setting `STATELESS` guarantees the server never creates or stores session state in RAM; every incoming request must provide self-contained credentials (JWT).

### REST API & Microservice Protection Strategies (JWT, OAuth2, RBAC)
- **Core Pillars of Enterprise API Security:**
  1. **Authentication:** Verifying client identity via cryptographically signed tokens (JWT / OAuth2 access tokens).
  2. **Authorization:** Granting or restricting endpoint execution based on user permissions or roles (RBAC).
  3. **Stateless Tokens:** Decoupling authentication state from server memory to enable infinite horizontal scaling.
- **Secure Coding Practices:**
  - Enforce HTTPS across all environments to prevent token eavesdropping.
  - Apply the Principle of Least Privilege: expose only strictly necessary endpoints and operations.
  - Sanitize and validate all incoming request parameters (Jakarta Validation).
  - Mask error details to prevent internal stack trace leakage in API responses.

---

## Token-Based Authentication (JWT)

### JWT Anatomy (Header, Payload, Cryptographic Signature)
- A JSON Web Token (RFC 7519) is a compact, URL-safe string containing three distinct Base64URL-encoded components separated by dots (`Header.Payload.Signature`):
  1. **Header:** Identifies the token type (`JWT`) and signing algorithm (e.g., `HS256` or `RS256`).
  2. **Payload (Claims):** Contains statement claims about the user (e.g., `sub` (userId), `roles`, `exp` (expiration), `iat` (issued at)). *Crucial Rule:* The payload is strictly encoded, not encrypted; never store passwords, private keys, or sensitive PII in claims.
  3. **Signature:** The mathematical proof guaranteeing token integrity. Computed by hashing the header, payload, and a secret or private key using the designated algorithm:
     $$\text{Signature} = \text{HMAC-SHA256}(\text{Base64}(Header) + "." + \text{Base64}(Payload), \text{SecretKey})$$
     If an attacker modifies claims in the payload, the recalculated signature will not match, causing immediate rejection.

### End-to-End JWT Authentication Lifecycle (Login to Verification)
- **Phase 1: Token Issuance (Login):**
  1. Client sends a `POST /api/auth/login` with credentials.
  2. The Authentication Server validates credentials against the database.
  3. Upon success, the server generates a JWT containing user claims and expiration, signs it using its private secret key, and returns it in the response body.
  4. The client stores the JWT securely (preferably in an `HttpOnly`, `Secure`, `SameSite` cookie to mitigate XSS attacks).
- **Phase 2: Accessing Protected Endpoints:**
  1. The client attaches the JWT to requests in the `Authorization` header: `Authorization: Bearer <token>`.
  2. A custom `OncePerRequestFilter` or Spring Resource Server intercepts the request.
  3. The server validates the cryptographic signature using its secret/public key and checks expiration.
  4. If valid, Spring extracts claims, instantiates an `Authentication` token (`UsernamePasswordAuthenticationToken`), and places it into `SecurityContextHolder.getContext().setAuthentication(...)`.
  5. If invalid or expired, the filter halts execution with an HTTP 401 Unauthorized response.

### Stateless Horizontal Scalability vs. Session-Based Authentication
- **Traditional Session Bottlenecks:** Session-based authentication requires servers to store session state in JVM RAM or synchronize via distributed caches (e.g., Redis). Under high traffic, session replication consumes bandwidth and creates single points of failure.
- **Stateless Token Scaling:** Because the JWT carries verified claims within itself, any microservice instance can authenticate the user mathematically without touching a database or shared session cache, allowing effortless horizontal scaling across cloud instances.

---

## OAuth2 & Distributed Microservice Security

### OAuth2 Architecture: Authorization Server, Client Gateway & Resource Servers
- **Decoupled Responsibilities in Distributed Systems:**
  - **Authorization Server (Keycloak / Auth0 / Spring Authorization Server):** Owns user authentication, credentials, MFA, and token issuance. Microservices never touch login passwords directly.
  - **Client / API Gateway:** Directs users to the Authorization Server, exchanges authorization codes for access tokens, and passes tokens downstream.
  - **Resource Server (Microservices):** Backend services hosting domain business APIs. They never handle login forms; their sole responsibility is validating incoming JWT tokens and enforcing role permissions.

### Spring Boot 3 Resource Server Configuration (JWKS & Issuer URI)
- **Dependency:** `spring-boot-starter-oauth2-resource-server`.
- **Java Configuration:**
  ```java
  @Configuration
  @EnableWebSecurity
  public class SecurityConfig {

      @Bean
      public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
          http
              .csrf(csrf -> csrf.disable())
              .authorizeHttpRequests(auth -> auth
                  .requestMatchers("/api/public/**").permitAll()
                  .requestMatchers("/api/admin/**").hasRole("ADMIN")
                  .anyRequest().authenticated()
              )
              .oauth2ResourceServer(oauth2 -> oauth2
                  .jwt(Customizer.withDefaults())
              );

          return http.build();
      }
  }
  ```
- **YAML Configuration (JWKS Auto-Download):**
  ```yaml
  spring:
    security:
      oauth2:
        resourceserver:
          jwt:
            issuer-uri: https://my-auth-server.com/realms/production
  ```
  *Mechanism:* At startup, Spring Security contacts the `issuer-uri`, retrieves the OpenID Connect metadata, and fetches public keys from the JSON Web Key Set (JWKS) endpoint to validate token signatures statelessly without calling the auth server per request.

### Service-to-Service Token Relay Pattern (WebClient ServletBearerExchangeFilterFunction)
- **The Challenge:** When Microservice A receives an authenticated request and must call Microservice B, Service B needs to know the caller identity and permissions.
- **Token Relay Pattern:** Microservice A extracts the bearer token from its own incoming request's `SecurityContext` and propagates it in the outgoing request to Service B.
- **WebClient Auto-Relay Configuration:**
  ```java
  @Bean
  public WebClient webClient() {
      return WebClient.builder()
          .filter(new ServletBearerExchangeFilterFunction()) // Auto-propagates Bearer token from SecurityContext
          .build();
  }
  ```

### Keycloak & OAuth2 Custom Role Mapping (JwtAuthenticationConverter)
- **The Problem:** Third-party OAuth2 providers (like Keycloak) store roles in nested JSON claims (e.g., `realm_access.roles`), whereas Spring Security expects roles as a flat list of `GrantedAuthority` objects prefixed with `ROLE_`.
- **The Solution:** Configure a custom `JwtAuthenticationConverter`:
  ```java
  @Bean
  public JwtAuthenticationConverter jwtAuthenticationConverter() {
      JwtGrantedAuthoritiesConverter grantedAuthoritiesConverter = new JwtGrantedAuthoritiesConverter();
      grantedAuthoritiesConverter.setAuthorityPrefix("ROLE_");
      grantedAuthoritiesConverter.setAuthoritiesClaimName("roles");

      JwtAuthenticationConverter jwtAuthenticationConverter = new JwtAuthenticationConverter();
      jwtAuthenticationConverter.setJwtGrantedAuthoritiesConverter(grantedAuthoritiesConverter);
      return jwtAuthenticationConverter;
  }
  ```

---

## Web Vulnerabilities & Transport Security

### Cross-Site Request Forgery (CSRF): Mechanics & Stateless API Immunity
- **Attack Mechanics:** A malicious website tricks an authenticated user's browser into executing unwanted actions on another site where the user is currently logged in. Because browsers automatically send ambient cookies (including session cookies) with cross-origin requests, the target server mistakenly treats the request as authenticated.
- **Stateless REST Immunity:** CSRF attacks rely strictly on browsers automatically attaching credentials (cookies). Stateless REST APIs authenticate via `Authorization: Bearer <JWT>` headers. Browsers never automatically attach custom `Authorization` headers to cross-site requests, making CSRF physically impossible.
- **Rule of Thumb:**
  - **Stateless REST APIs (JWTs / Bearer Tokens):** Disable CSRF (`csrf.disable()`) to eliminate processing overhead.
  - **Stateful Browser MVC Apps (Session Cookies):** Enable CSRF protection using `CookieCsrfTokenRepository.withHttpOnlyFalse()` to synchronize CSRF tokens.

### HTTP OPTIONS Pre-Flight Requests & CORS Performance Optimization (Max-Age Caching)
- **The Pre-Flight Mechanism:** Before sending non-simple cross-origin requests (e.g., `POST` with `Content-Type: application/json` or custom headers), browsers send an automatic HTTP `OPTIONS` pre-flight check asking the server whether the origin, method, and headers are permitted.
- **The Latency Trap:** Without caching, every single API invocation requires two round-trips (one for `OPTIONS`, one for the real request), doubling network latency and increasing backend load.
- **Performance Fix (CORS Max-Age Caching):** Configure `maxAge` on CORS configurations:
  ```java
  @Bean
  public CorsConfigurationSource corsConfigurationSource() {
      CorsConfiguration configuration = new CorsConfiguration();
      configuration.setAllowedOrigins(List.of("https://app.example.com"));
      configuration.setAllowedMethods(List.of("GET", "POST", "PUT", "DELETE"));
      configuration.setAllowedHeaders(List.of("Authorization", "Content-Type"));
      
      // Cache the pre-flight response in the browser for 1 hour (3600 seconds)
      configuration.setMaxAge(3600L);

      UrlBasedCorsConfigurationSource source = new UrlBasedCorsConfigurationSource();
      source.registerCorsConfiguration("/**", configuration);
      return source;
  }
  ```
  *Result:* The browser caches pre-flight approval, executing subsequent requests immediately without repeating `OPTIONS` handshakes.

---

## Method-Level Authorization & Access Control

### Role-Based Access Control (RBAC) & @EnableMethodSecurity
- **RBAC Principle:** Permissions are bound to functional Roles (e.g., `ROLE_ADMIN`, `ROLE_MANAGER`) rather than individual user identities. Users are assigned roles, decoupling business permissions from personnel changes.
- **Enabling Method Security:** Annotate configuration with `@EnableMethodSecurity` (replaces legacy `@EnableGlobalMethodSecurity` in Spring Boot 3 / Security 6).
- **SpEL Expressions:** Annotate methods using Spring Expression Language (`hasRole('ADMIN')`, `hasAuthority('SCOPE_read')`).

### @PreAuthorize vs. @PostAuthorize Mechanics & The Mutation Trap
- **`@PreAuthorize` (Pre-Execution Guard):**
  - Evaluates the security SpEL expression **before** the method runs.
  - If the check fails, Spring throws an `AccessDeniedException` (HTTP 403 Forbidden) and the method body is never executed.
  - Supports checking method arguments (e.g., `#userId == authentication.principal.id`).
  ```java
  @PreAuthorize("hasRole('ADMIN') or #userId == authentication.principal.id")
  public void updateProfile(Long userId, ProfileData data) {
      userRepository.update(userId, data);
  }
  ```
- **`@PostAuthorize` (Post-Execution Auditor):**
  - Evaluates the security SpEL expression **after** the method has executed, but before the return value is delivered to the caller.
  - Can inspect the return object using `returnObject` (e.g., verifying if the fetched entity's `ownerId` matches the authenticated user).
  ```java
  @PostAuthorize("returnObject.ownerId == authentication.principal.id")
  public Document getConfidentialDocument(Long documentId) {
      return documentRepository.findById(documentId).orElseThrow();
  }
  ```
- **The Mutation Trap:**
  - Because `@PostAuthorize` runs *after* method completion, any state mutations (database deletes, payment charges) have already been committed to the database! If the authorization check fails, Spring throws 403, but the underlying data mutation is not rolled back.
  - *Golden Rule:* Strictly restrict `@PostAuthorize` to read-only (`GET`) operations.

### Method Security Comparison Matrix (@PreAuthorize vs. @PostAuthorize)
| Feature | `@PreAuthorize` | `@PostAuthorize` |
| :--- | :--- | :--- |
| **Execution Point** | Before method body executes | After method body executes, before return |
| **Access Method Arguments** | Yes (e.g., `#userId`) | Yes |
| **Access Return Value** | No | Yes (via `returnObject`) |
| **Safe for Mutations (Writes/Deletes)** | **Yes** (Blocks execution completely) | **No** (Database actions execute regardless) |
| **Primary Use Case** | Role validation, input ownership checks | Data filtering when ownership is unknown prior to DB query |
