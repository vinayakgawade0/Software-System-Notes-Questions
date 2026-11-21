# Notes - Cloud - 4. Serverless Java & AWS Lambda

## 1. Serverless Compute Spectrum & AWS Lambda Architecture

### Serverless Java Spectrum: Lambda, Fargate, App Runner & PaaS History

In enterprise cloud architecture, **Serverless** is not synonymous with Function-as-a-Service (FaaS). It encompasses any operational model where engineers manage zero underlying operating systems or virtual machine hardware, with automated elastic scaling and pay-for-use pricing:

| Serverless Model | Primary AWS Services | How Java / Spring Boot Runs | Best Architectural Fit |
| :--- | :--- | :--- | :--- |
| **Function-as-a-Service (FaaS)** | **AWS Lambda** | Single Java method implementing `RequestHandler` or Spring Cloud Function. Hard 15-minute timeout. | Asynchronous S3 file processing, SQS consumers, scheduled cron tasks, spiky webhooks. |
| **Serverless Containers (CaaS)** | **AWS Fargate** (Amazon ECS / EKS) | Standard Spring Boot application packaged as a Docker container. Continuous execution with zero VM management. | Core synchronous REST microservices, long-running batch workers, persistent database connections. |
| **Fully Managed Web Apps** | **AWS App Runner** | Direct deployment from Git repository or container image with automated load balancing and TLS. | Fast-to-market HTTP web APIs without VPC or load balancer orchestration overhead. |

#### Pre-2014 History & Platform-as-a-Service (PaaS) Ancestors
- **The Ancestors (2007–2011):** Heroku (2007), Google App Engine (2008), and AWS Elastic Beanstalk (2011) automated application deployments from `.jar` and `.war` packages.
- **Why PaaS Was Not "True" Serverless:** In Elastic Beanstalk, dedicated EC2 instances and load balancers ran 24/7 in the customer's account. Customers were billed for continuous capacity even when traffic was zero at 3:00 AM, and autoscaling required minutes to boot fresh virtual machines rather than scaling instantaneously to zero.
- **The FaaS Paradigm Shift (2014):** AWS Lambda introduced sub-second micro-VM provisioning, native event triggers, per-millisecond duration billing, and automatic scaling to zero when idle.

---

### Java FaaS Function Anatomy vs Traditional Spring Boot Controller

In Function-as-a-Service, a function is a single isolated Java method executed in response to a cloud event, fundamentally distinct from a monolithic web server:

```
[ Traditional Web Server ]               [ FaaS Cloud Function ]
+----------------------------+           +----------------------------+
| Spring Boot Web Server     |           | Cloud Event Trigger        |
| - Runs 24/7 on port 8080   |           | (S3 Upload / SQS / Cron)   |
| - Embedded Tomcat active   |           +--------------+-------------+
| - Billed continuously      |                          |
| - Thread pools listen loop |                          v
+----------------------------+           +----------------------------+
                                         | Single Java Method Invoked |
                                         | - RequestHandler interface |
                                         | - Executes to completion   |
                                         | - Suspends / Billed for ms |
                                         +----------------------------+
```

#### Code Comparison:

```java
// TRADITIONAL SPRING BOOT CONTROLLER (Always-on HTTP server listening on port 8080)
@RestController
public class UserController {

    private final UserService userService;

    public UserController(UserService userService) {
        this.userService = userService;
    }

    @PostMapping("/users")
    public ResponseEntity<UserResponse> createUser(@RequestBody UserRequest request) {
        return ResponseEntity.ok(userService.save(request));
    }
}
```

```java
// AWS LAMBDA FAAS FUNCTION (Single event-driven method implementing RequestHandler)
public class CreateUserFunction implements RequestHandler<UserRequest, Response> {

    // Initialized ONCE during cold start / SnapStart phase
    private static final UserService userService = new UserServiceImpl();

    @Override
    public Response handleRequest(UserRequest request, Context context) {
        // 1. Invoked directly by AWS when an event fires.
        // 2. No embedded Tomcat, no port 8080, no @RestController.
        return userService.save(request);
    }
}
```

---

### AWS Firecracker MicroVM Architecture & Lambda Execution Lifecycle

Under the hood, AWS Lambda does not execute Java functions inside a shared application server or multi-tenant JVM. It isolates every customer workload using **AWS Firecracker**:

```
+-------------------------------------------------------------------------+
|                  Physical Bare-Metal Host (Linux Kernel)                 |
|  +-----------------------------------+-------------------------------+  |
|  | Firecracker MicroVM (Customer A)  | Firecracker MicroVM (Cust. B) |  |
|  | - Dedicated Guest Linux OS        | - Dedicated Guest Linux OS    |  |
|  | - Dedicated HotSpot JVM Process   | - Dedicated HotSpot JVM Proc. |  |
|  | - Stripped-down device drivers    | - Stripped-down device drivers|  |
|  | - Millisecond launch time         | - Millisecond launch time     |  |
|  +-----------------------------------+-------------------------------+  |
+-------------------------------------------------------------------------+
```

#### Detailed Execution Lifecycle:
1. **Bare-Metal Fleet:** AWS manages fleets of high-performance bare-metal servers running Linux KVM.
2. **Firecracker MicroVM Creation:** Firecracker strips out legacy PC peripherals and BIOS emulations, booting a minimal, hardware-isolated virtual machine in single-digit milliseconds.
3. **Execution Environment Initialization (Cold Start):** The guest OS mounts the application payload, boots the Java Virtual Machine, loads bytecode, and initializes static variables.
4. **Invocation:** AWS passes the event payload into `handleRequest()`.
5. **Freeze & Thaw (Warm Starts):** After `handleRequest()` completes, the micro-VM is frozen in memory rather than destroyed. Subsequent requests arriving shortly after thaw the existing JVM, executing in single-digit milliseconds.
6. **Reclamation:** If no invocations arrive within an idle window (~15–45 minutes), the micro-VM is terminated.

*(Multi-cloud note: Google Cloud Functions / Cloud Run enforce isolation using **gVisor** syscall interception; Azure Functions utilizes pre-warmed container worker pools).*

---

### Spring Cloud Function Migration, Cold Starts & SnapStart Optimization

#### 1. Migrating Spring Boot to AWS Lambda via Spring Cloud Function
Rather than rewriting an entire Spring application, organizations use `spring-cloud-function-adapter-aws`:

```xml
<!-- pom.xml -->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-function-adapter-aws</artifactId>
</dependency>
```

- **Configure Lambda Handler:** Set the function handler to `org.springframework.cloud.function.adapter.aws.FunctionInvoker::handleRequest`.
- **Expose Functional Beans:** Replace `@RestController` classes with standard Java functional beans (`Function`, `Consumer`, `Supplier`):

```java
@Configuration
public class PaymentProcessingConfig {

    @Bean
    public Function<PaymentRequest, PaymentResponse> processPayment(PaymentService service) {
        return request -> service.execute(request);
    }
}
```

#### 2. Java Cold Starts & Mitigation Strategies
Because booting a MicroVM, starting the JVM, and initializing Spring's `ApplicationContext` can take **2 to 6+ seconds**, enterprise architectures employ specific mitigations:

- **AWS Lambda SnapStart (Java 11, 17, 21+):**
  - Enabled at zero additional cost.
  - When a function version is published, SnapStart boots the function, runs static initializers, and initializes the Spring context ahead of time.
  - AWS takes an encrypted memory and disk snapshot of the Firecracker MicroVM and caches it.
  - Cold starts restore directly from the cached snapshot, reducing latency from ~5 seconds down to **under 200 milliseconds**.
- **Static / Class-Level SDK Initialization:**
  - Initialize heavy dependencies (AWS SDK clients, JSON mappers) as `static final` fields outside `handleRequest()`, allowing warm invocations to reuse established connection pools:

```java
public class S3ProcessorLambda implements RequestHandler<S3Event, Void> {

    // Initialized ONCE during cold start / SnapStart phase
    private static final S3Client s3Client = S3Client.create();
    private static final ObjectMapper objectMapper = new ObjectMapper();

    @Override
    public Void handleRequest(S3Event event, Context context) {
        // Reuses existing s3Client connection pool
        return null;
    }
}
```

- **GraalVM Native Image:** Ahead-of-Time (AOT) compilation into standalone native machine code, achieving <50ms startup times at the expense of build-time reflection configuration.
- **Provisioned Concurrency:** Pre-warms a fixed number of execution environments ready to serve requests immediately, billed continuously for provisioned capacity.

#### 3. Core Lambda Constraints & Database Connection Management
- **Memory-to-CPU Proportionality:** You cannot configure vCPU directly. Allocating **1,769 MB of Memory** grants exactly **1 full vCPU**. Fix slow CPU-bound Java functions by increasing memory.
- **Hard Execution Limits:** Strict **15-minute (900 seconds)** execution timeout; default soft limit of **1,000 concurrent executions** per region.
- **Amazon RDS Proxy:** Because Lambda can burst to 1,000 concurrent sandboxes in seconds, direct connections will instantly exhaust PostgreSQL connection pools. **Amazon RDS Proxy** must be placed in front of RDS to pool and multiplex database connections.

---
