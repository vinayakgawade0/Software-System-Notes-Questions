# Notes - Cloud - 6. Relational Databases & Security (RDS, IAM, Secrets)

## 1. Relational Database Management & Connection Pooling

### Amazon RDS Connection Pool Tuning (HikariCP) & Multi-AZ vs Read Replicas

When deploying Spring Boot applications on EC2, ECS, or EKS against **Amazon RDS (PostgreSQL Engine)**, database connection exhaustion is the primary cause of production outages under heavy load.

#### 1. Diagnosing & Remedying "Too Many Connections" Outages
- **The Anti-Pattern:** When receiving PostgreSQL `FATAL: remaining connection slots are reserved for non-superuser connections`, inexperienced developers raise PostgreSQL's `max_connections` parameter. Because PostgreSQL spawns a dedicated operating system process per connection, each allocating RAM for buffers, setting `max_connections` too high causes severe host memory pressure, swapping, and kernel OOM crashes.
- **The Correct Architecture:** Mathematically tune the client-side connection pool (**HikariCP**) across all active application instances:

$$(	ext{Application Instances} 	imes 	ext{maximum-pool-size}) < 	ext{RDS max\_connections}$$

```properties
# application.properties (Production HikariCP Tuning)
spring.datasource.hikari.maximum-pool-size=15
spring.datasource.hikari.minimum-idle=5
spring.datasource.hikari.connection-timeout=30000
spring.datasource.hikari.idle-timeout=600000
spring.datasource.hikari.max-lifetime=1800000
```

- **`maximum-pool-size=15`:** Caps active database connections per JVM. Even during aggressive autoscaling, the aggregate connection count stays below the database threshold.
- **`minimum-idle=5`:** Maintains a small pool of warm connections to avoid connection handshake latency during traffic spikes.
- **`connection-timeout=30000`:** Enforces a 30-second fast-fail timeout when the pool is saturated, preventing HTTP threads from hanging indefinitely.
- *(Note: For highly dynamic, bursting serverless workloads like AWS Lambda, place **Amazon RDS Proxy** between the compute layer and RDS to pool and multiplex database connections).*

#### 2. Amazon RDS High Availability: Multi-AZ vs. Read Replicas

```
+---------------------------------------------------------------------------------+
|                              AMAZON RDS ARCHITECTURE                            |
|                                                                                 |
|     [ Primary RDS Instance (AZ 1) ] -------------> [ Read Replica (AZ 2/3) ]    |
|      (Active: Serves Reads & Writes)  Asynchronous  (Active: Serves Reads Only) |
|                    |                  Replication                               |
|                    | Synchronous Replication                                    |
|                    v                                                            |
|     [ Standby RDS Instance (AZ 2) ]                                             |
|      (Passive: Zero Traffic Until Failover)                                     |
+---------------------------------------------------------------------------------+
```

| Architectural Feature | Amazon RDS Multi-AZ Deployment | Amazon RDS Read Replicas |
| :--- | :--- | :--- |
| **Primary Architectural Role** | **High Availability (HA) & Disaster Recovery:** Synchronous replication for automated failover. | **Scalability & Performance:** Horizontal scaling for read-intensive (`SELECT`) query offloading. |
| **Replication Mechanism** | **Synchronous block-level replication** to a standby instance in a different Availability Zone. | **Asynchronous engine-native replication** (PostgreSQL WAL streaming). |
| **Serves Application Traffic?** | **No:** The standby instance is completely passive. It cannot serve read or write queries until failover. | **Yes:** Serves read queries directly via its dedicated database endpoint. |
| **Failover Behavior** | **Automated (<60 seconds):** AWS automatically updates the DNS record to point to the standby instance upon primary failure. | **Manual Promotion:** Can be promoted to a standalone primary database if disaster recovery requires it. |

---

## 2. Enterprise Cloud Security Architecture (IAM & Secrets)

### Enterprise Cloud Security: IAM Roles, Secrets Manager & AWS KMS

Production enterprise architectures enforce zero-trust security across credentials, encryption, and authorization:

#### 1. AWS IAM Roles & Instance Profiles (Infrastructure Authorization)
- **Zero Static Credentials:** Developers must never hardcode static access keys (`AWS_ACCESS_KEY_ID` / `AWS_SECRET_ACCESS_KEY`) in `application.properties`, environment variables, Docker images, or Git repositories.
- **Instance Profiles & Task Roles:** Attach an **IAM Role** directly to the EC2 Instance Profile, ECS Task Role, or Lambda Execution Role. The AWS SDK for Java v2 automatically queries the instance metadata service (IMDSv2) to retrieve temporary, auto-rotated STS security credentials.

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowS3InvoiceBucketAccess",
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:PutObject"
      ],
      "Resource": "arn:aws:s3:::enterprise-invoices-production/*"
    }
  ]
}
```

#### 2. AWS Secrets Manager (Runtime Credential Management)
- **Centralized Secrets Storage:** Stores sensitive database passwords, third-party API tokens, and private signing keys (analogous to HashiCorp Vault).
- **Automated Rotation:** Natively orchestrates periodic password rotation in Amazon RDS without application downtime.
- **Runtime Injection:** Spring Boot applications retrieve secrets dynamically at startup via Spring Cloud AWS Secrets Manager integration.

#### 3. AWS Key Management Service (AWS KMS)
- Provides centralized management of Customer Master Keys (CMKs) and hardware security modules (HSM).
- **Envelope Encryption:** Generates plaintext data keys to encrypt application data locally, while encrypting the data key under a KMS master key. KMS is natively integrated into Amazon S3 (SSE-KMS), Amazon EBS volume encryption, and Amazon RDS storage encryption.

---
