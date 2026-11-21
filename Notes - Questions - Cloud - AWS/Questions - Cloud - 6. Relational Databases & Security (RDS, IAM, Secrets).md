# Questions - Cloud - 6. Relational Databases & Security (RDS, IAM, Secrets)

### If a Spring Boot app on EC2/EKS throws "Too many connections" against AWS RDS under load, how do you fix it, and how do RDS Multi-AZ and Read Replicas differ?

> **Direct Answer:**
> - Fix connection exhaustion by mathematically tuning **HikariCP** properties (`maximum-pool-size`, `minimum-idle`, `connection-timeout`) across your application instances so that `(aggregate instances × maximum-pool-size)` remains strictly below PostgreSQL's `max_connections` limit—rather than blindly increasing database connections (which exhausts database RAM).
> - **RDS Multi-AZ** provides synchronous high availability and automated disaster failover to a passive standby instance (which serves zero traffic).
> - **RDS Read Replicas** provide asynchronous replication to scale read-heavy (`SELECT`) query traffic horizontally via dedicated read endpoints.
>
> **Key Nuance / Trade-off:**
> - Raising PostgreSQL `max_connections` setting indiscriminately degrades database performance because each connection allocates dedicated operating system process memory and cache buffers. For bursty serverless workloads like AWS Lambda, place **Amazon RDS Proxy** between the application and database to pool and multiplex connections safely.
>
> **Reference Concept:** `Amazon RDS Connection Pool Tuning (HikariCP) & Multi-AZ vs Read Replicas`

---

### How should AWS permissions and application secrets be managed in a production Spring Boot architecture?

> **Direct Answer:**
> - Never hardcode AWS access keys, database passwords, or JWT signing keys in `application.properties` or source repositories.
> - Attach least-privilege **AWS IAM Roles** directly to compute resources (EC2 Instance Profiles, ECS Task Roles, Lambda Execution Roles) so the AWS SDK for Java v2 retrieves short-lived, auto-rotated STS credentials automatically from instance metadata.
> - Manage runtime application secrets (database credentials, third-party API keys) using **AWS Secrets Manager** with automated rotation, and protect sensitive data at rest across EBS, RDS, and S3 using envelope encryption managed by **AWS KMS**.
>
> **Key Nuance / Trade-off:**
> - Clearly separate authorization boundaries: AWS IAM governs infrastructure-level cloud authorization (controlling which AWS APIs an EC2 instance can call), whereas application-level user authentication is managed via JWT tokens. IMDSv2 (Instance Metadata Service Version 2) should always be enforced on EC2 to mitigate Server-Side Request Forgery (SSRF) vulnerabilities.
>
> **Reference Concept:** `Enterprise Cloud Security: IAM Roles, Secrets Manager & AWS KMS`

---
