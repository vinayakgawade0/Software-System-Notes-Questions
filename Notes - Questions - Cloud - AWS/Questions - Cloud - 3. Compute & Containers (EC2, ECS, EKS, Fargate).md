# Questions - Cloud - 3. Compute & Containers (EC2, ECS, EKS, Fargate)

### What is the difference between Amazon EC2 and AWS Lambda?

> **Direct Answer:**
> - Amazon EC2 is an Infrastructure-as-a-Service (IaaS) offering that provides always-on, customer-managed virtual Linux/Windows servers with unlimited execution durations, billed continuously for provisioned capacity.
> - AWS Lambda is a Function-as-a-Service (FaaS) offering that provides serverless, event-driven compute environments managed entirely by AWS, enforcing a strict 15-minute execution limit and billing strictly for execution duration (in milliseconds) and invocation counts ($0 cost when idle).
>
> **Key Nuance / Trade-off:**
> - Deploying a high-volume, synchronous Spring Boot REST API on Lambda can become more expensive than EC2/Fargate while introducing cold-start latency spikes and database connection pool exhaustion. EC2 is optimized for steady, long-running services with persistent database pools, whereas Lambda is optimized for asynchronous, spiky event processing.
>
> **Reference Concept:** `Amazon EC2 vs AWS Lambda Architectural Trade-offs`

---

### What must a Java developer know in detail about Amazon EC2?

> **Direct Answer:**
> - A senior Java developer must understand:
>   1. **Golden AMIs:** Pre-baking hardened Linux operating systems, security agents, and OpenJDK 21 into immutable images for fast, deterministic autoscaling.
>   2. **Instance Families:** Choosing the right hardware profile (General Purpose `m6i`, Compute Optimized `c6i`, Memory Optimized `r6i`) to prevent JVM OOMs or wasted compute spend.
>   3. **Security Groups vs. NACLs:** Leveraging stateful instance-level firewalls (SGs) for fine-grained application access and stateless subnet-level firewalls (NACLs) for boundary defense.
>   4. **High Availability (ALB + ASG):** Placing instances across multiple AZs and configuring the ALB to poll Spring Boot Actuator's `/actuator/health` endpoint for self-healing replacement.
>   5. **Storage Mechanics (EBS vs. Instance Store):** Using network-attached persistent EBS volumes for application state and physically attached ephemeral Instance Store SSDs for high-speed scratch caching.
>   6. **User Data Scripts:** Automating final-mile runtime configuration and systemd service startup upon initial instance boot.
>
> **Key Nuance / Trade-off:**
> - Security Group statefulness and Instance Store ephemerality on stop/terminate are guaranteed cloud behaviors. Baking dependencies into a Golden AMI rather than downloading them during User Data bootstrapping reduces instance launch latency from 15 minutes to under 90 seconds.
>
> **Reference Concept:** `Amazon EC2 Core Mechanics, Golden AMIs, Networking & Storage`

---

### How do Amazon ECS and Amazon EKS compare (and EC2 vs Fargate) for Spring Boot microservices?

> **Direct Answer:**
> - **Amazon ECS:** AWS's proprietary container orchestrator featuring zero control-plane fees, low operational complexity, and seamless native integration with AWS primitives (ALB, IAM, CloudWatch).
> - **Amazon EKS:** Managed upstream Kubernetes (CNCF certified) with a ~$73/month control-plane fee per cluster, delivering multi-cloud portability and access to the rich Kubernetes ecosystem (Helm, Istio, Prometheus) at the expense of higher operational complexity.
> - **EC2 vs. Fargate:** Both orchestrators can run on self-managed EC2 worker nodes (where you control the OS, kernel tuning, and caching) or AWS Fargate (serverless containers where AWS manages the underlying VM infrastructure).
>
> **Key Nuance / Trade-off:**
> - Choose ECS + Fargate when an organization wants rapid time-to-market without Kubernetes operational overhead. Choose EKS when an enterprise requires strict multi-cloud portability, cross-cloud deployment standardization, or complex service meshes. When running EKS on EC2 worker nodes, Linux systems skills (systemd, cgroups, memory limits, and networking) apply directly when debugging node pressure and Pod evictions.
>
> **Reference Concept:** `Container Orchestration: Amazon ECS vs Amazon EKS & EC2 vs Fargate`

---
