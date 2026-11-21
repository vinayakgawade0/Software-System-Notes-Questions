# Questions - Cloud - 1. Foundations & Architecture

### Why choose AWS over Microsoft Azure or Google Cloud Platform (GCP) for a Java backend role?

> **Direct Answer:**
> - AWS is the most mature, widely adopted cloud platform for enterprise Java microservices, offering deep operational control over compute, networking, and security primitives that integrate natively with the Spring ecosystem.
> - Key technical drivers include premier SDK support (**AWS SDK for Java v2**, **Spring Cloud AWS**), enterprise JVM performance innovations (AWS Lambda SnapStart, Amazon Corretto), and fine-grained VPC networking and IAM controls.
>
> **Key Nuance / Trade-off:**
> - Platform selection depends on enterprise alignment rather than dogmatic superiority: Azure is optimal for enterprises standardized on the Microsoft ecosystem (Active Directory / Entra ID, .NET, Windows Server), while GCP excels in big data analytics (BigQuery) and managed open-source Kubernetes (GKE). AWS provides the deepest operational control, but incurs higher configuration complexity compared to opinionated PaaS alternatives.
>
> **Reference Concept:** `Cloud Provider Evaluation & AWS Platform Selection`

---

### Even though AWS provides hundreds of services, which core services should a Java backend developer focus on, and how do they map to private cloud / on-premises technologies?

> **Direct Answer:**
> - Java developers should master six foundational pillars: Compute (**EC2**, **Lambda**, **ECS/EKS with Fargate**), Relational Databases (**RDS**), Messaging (**SQS**, **SNS**), Event Streaming (**MSK**, **Kinesis**), Object Storage (**S3**), and Security/Secrets (**IAM**, **Secrets Manager**, **KMS**).
> - These map directly to on-premises counterparts: Linux VMs (CentOS/AlmaLinux) $\to$ EC2/Fargate; self-managed PostgreSQL $\to$ RDS; RabbitMQ/ActiveMQ $\to$ SQS/SNS; Apache Kafka $\to$ Amazon MSK; NAS/SAN/Ceph $\to$ S3; HashiCorp Vault $\to$ Secrets Manager/KMS.
>
> **Key Nuance / Trade-off:**
> - Never conflate architectural layers: SQS is a point-to-point message queue, not an event streaming log like Apache Kafka (which maps to Amazon MSK). Similarly, JWT is an application-level identity token for end-user authentication, whereas AWS IAM provides infrastructure-level cloud authorization governing whether an EC2 instance can invoke AWS APIs.
>
> **Reference Concept:** `Core AWS Service Pillars & On-Premises Mapping`

---

### Can AWS architecture be drawn hierarchically, or is AWS just an umbrella of unrelated services?

> **Direct Answer:**
> - While AWS offers hundreds of standalone services, an enterprise Java backend must always be architected hierarchically as an $n$-tier request-flow pipeline wrapped by a unified security and observability boundary.
> - The standard 6-layer architecture flows from Layer 1 Entry Point (ALB/API Gateway) $\to$ Layer 2 Compute (Spring Boot on EC2/Fargate) $\to$ Layer 3 Decoupling (SQS/SNS) $\to$ Layer 4 Background Workers (Lambda/Worker nodes) $\to$ Layer 5 Persistence (RDS PostgreSQL and S3 Object Storage), wrapped by Layer 6 Security & Observability (IAM, Secrets Manager, CloudWatch).
>
> **Key Nuance / Trade-off:**
> - Decoupling synchronous REST APIs (Layer 2) from heavy background processing (Layer 4) via messaging queues (Layer 3) ensures that sudden traffic spikes or long-running tasks (e.g., report generation) do not saturate HTTP request threads or trigger client gateway timeouts.
>
> **Reference Concept:** `Six-Layer Hierarchical AWS Java Backend Architecture`

---
