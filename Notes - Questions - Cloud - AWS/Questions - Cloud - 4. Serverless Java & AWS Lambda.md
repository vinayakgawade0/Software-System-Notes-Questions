# Questions - Cloud - 4. Serverless Java & AWS Lambda

### Is AWS Lambda the only serverless option for Java, and what existed before Lambda?

> **Direct Answer:**
> - No—AWS Lambda is only Function-as-a-Service (FaaS). For enterprise Java/Spring Boot applications, AWS also provides **AWS Fargate** (Serverless Containers / CaaS) and **AWS App Runner** (fully managed containerized web applications).
> - Before Lambda launched in November 2014, Platform-as-a-Service (PaaS) offerings like Heroku (2007), Google App Engine (2008), and AWS Elastic Beanstalk (2011) automated application deployments from JAR/WAR packages.
>
> **Key Nuance / Trade-off:**
> - PaaS was not "true" serverless because underlying virtual machines ran 24/7 in the customer's account (billing continuously while idle), and autoscaling took minutes to provision VMs rather than milliseconds. For core Spring Boot REST microservices with steady traffic, enterprise architects prefer **AWS Fargate** (no cold starts, no 15-minute limits, standard HikariCP connection pooling), reserving **AWS Lambda** for event-driven asynchronous processing.
>
> **Reference Concept:** `Serverless Java Spectrum: Lambda, Fargate, App Runner & PaaS History`

---

### What is meant by a "function" in FaaS (Function-as-a-Service) for a Java developer?

> **Direct Answer:**
> - In Java FaaS, a "function" is a single, stateless, isolated Java class implementing a cloud runtime interface (such as AWS Lambda's `RequestHandler<I, O>`). It contains a single entry-point method (`handleRequest()`) executed only when triggered by a cloud event.
> - Unlike a traditional Spring Boot `@RestController` that runs continuously 24/7 on an embedded Tomcat server listening to port 8080, a FaaS function has no always-on server loop, costs $0 when idle, and is natively invoked by cloud events (S3 uploads, SQS messages, EventBridge cron schedules, API Gateway calls).
>
> **Key Nuance / Trade-off:**
> - FaaS functions must remain strictly stateless. Because execution environments can be frozen or destroyed at any moment by AWS, user session state must never be stored in local memory and must be offloaded to external persistence services (Amazon S3, DynamoDB, Redis).
>
> **Reference Concept:** `Java FaaS Function Anatomy vs Traditional Spring Boot Controller`

---

### Under the hood, how does AWS execute serverless Java functions—do they run a shared Java application server?

> **Direct Answer:**
> - No—AWS does not run a shared Tomcat instance or multi-tenant JVM. It isolates every function execution environment using **AWS Firecracker**, a lightweight open-source Virtual Machine Monitor (VMM) based on Linux KVM.
> - Firecracker provisions a dedicated, hardware-isolated MicroVM in single-digit milliseconds containing its own minimal Linux guest kernel, user space, and dedicated JVM process, guaranteeing that customer workloads never share memory or operating system kernels.
>
> **Key Nuance / Trade-off:**
> - After `handleRequest()` completes, AWS freezes the MicroVM in host memory instead of destroying it. Subsequent requests arriving shortly after thaw the existing JVM for millisecond-level warm starts. If the function remains idle past a timeout window, AWS terminates the MicroVM and reclaims physical host capacity.
>
> **Reference Concept:** `AWS Firecracker MicroVM Architecture & Lambda Execution Lifecycle`

---

### How do you migrate a Spring Boot microservice to AWS Lambda using Spring Cloud Function, why do Java Lambdas suffer from cold starts, and how do you mitigate them?

> **Direct Answer:**
> - Migrate by adding `spring-cloud-function-adapter-aws`, setting the Lambda handler to `org.springframework.cloud.function.adapter.aws.FunctionInvoker::handleRequest`, and declaring business logic as standard functional `@Bean`s (`Function`, `Supplier`, `Consumer`).
> - Java cold starts (2–6+ seconds) occur because AWS must allocate a MicroVM, boot the JVM, load classes, and initialize Spring's `ApplicationContext`.
> - Mitigate cold starts using **AWS Lambda SnapStart (Java 21)** (taking a Firecracker memory snapshot at publish time to restore in <200ms at $0 cost), **Provisioned Concurrency**, **GraalVM Native Images**, and **static SDK initialization**.
>
> **Key Nuance / Trade-off:**
> - Lambda scales CPU proportionally to allocated Memory (1,769 MB = 1 full vCPU); fix slow CPU-bound functions by increasing memory. Always place **Amazon RDS Proxy** between Lambda and relational databases to prevent thousands of bursting execution sandboxes from exhausting PostgreSQL connection pools.
>
> **Reference Concept:** `Spring Cloud Function Migration, Cold Starts & SnapStart Optimization`

---
