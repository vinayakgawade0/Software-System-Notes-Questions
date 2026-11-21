# Notes - Cloud - 3. Compute & Containers (EC2, ECS, EKS, Fargate)

## 1. Cloud Compute Paradigms & Virtual Machines

### Amazon EC2 vs AWS Lambda Architectural Trade-offs

Choosing the appropriate compute engine for Java applications on AWS requires balancing execution lifecycle, maintenance overhead, startup latency, and cost structure:

| Dimension | Amazon EC2 (Infrastructure-as-a-Service) | AWS Lambda (Function-as-a-Service) |
| :--- | :--- | :--- |
| **Core Abstraction** | Full virtual machines with dedicated Linux/Windows kernels running on AWS Nitro hypervisors. | Ephemeral MicroVM sandboxes (Firecracker) executing an isolated code handler. |
| **Execution Lifecycle** | **Always-On (24/7):** Runs continuously until explicitly stopped or terminated. | **Event-Driven:** Spins up on trigger, executes handler, and terminates/freezes when idle. |
| **Execution Duration Cap** | **Unlimited:** Can run processes, WebSocket servers, and batch jobs for days or months. | **Hard 15-Minute (900 seconds) Timeout** per invocation. |
| **OS & JRE Maintenance** | **Customer Managed:** Kernel patching, security hardening, OpenJDK installation, OS upgrades. | **AWS Managed:** AWS manages the underlying Linux OS, security patches, and runtime updates. |
| **Scaling Velocity** | **Minutes:** Auto Scaling Groups boot new virtual machines, initialize OS, and start JVM. | **Milliseconds:** Automatically spawns concurrent execution sandboxes up to account limits. |
| **Billing Model** | Provisioned capacity: Billed per second/hour regardless of CPU utilization or idle periods. | True Pay-per-Use: Billed per millisecond of execution duration and invocation count. **$0 when idle**. |
| **Primary Java Sweet Spot** | Enterprise Spring Boot microservices with predictable, steady traffic and persistent DB pools. | Asynchronous SQS consumer jobs, S3 event triggers, scheduled cron tasks, spiky/unpredictable webhooks. |

---

### Amazon EC2 Core Mechanics, Golden AMIs, Networking & Storage

Amazon Elastic Compute Cloud (EC2) provides resizable virtual computing capacity. For enterprise Java backends, EC2 management centers around seven core architectural pillars:

#### 1. The Golden AMI Strategy
- Rather than executing slow bootstrapping scripts (`dnf update`, downloading JREs) upon every instance launch, production environments utilize **Golden AMIs (Amazon Machine Images)**.
- **Implementation:** Built via automated pipelines (HashiCorp Packer or AWS EC2 Image Builder), baking a hardened AlmaLinux OS, security agents (CrowdStrike, Qualys), monitoring daemons (CloudWatch Agent), and OpenJDK 21 into an immutable image.
- **Enterprise Benefit:** Reduces Auto Scaling launch duration from 10–15 minutes down to under 90 seconds while guaranteeing identical, audited security baselines across all production nodes.

#### 2. EC2 Instance Families & Workload Mapping
- **General Purpose (`t3`, `m6i`, `m7g`):** Balanced 4:1 RAM-to-vCPU ratio. Standard for Spring Boot REST microservices and API gateways.
- **Compute Optimized (`c6i`, `c7g`):** High compute-to-memory ratio (2:1). Ideal for CPU-intensive algorithms, cryptography, batch processing, and proxy routing.
- **Memory Optimized (`r6i`, `r7g`):** High RAM ratio (8:1 to 16:1). Required for large JVM heaps (>32 GiB), distributed caches (Hazelcast, Redis), and in-memory analytics.
- **Storage Optimized (`i3en`, `im4gn`):** Directly attached high-speed NVMe SSD drives. Designed for high IOPS workloads like Apache Kafka brokers and Elasticsearch/OpenSearch clusters.

#### 3. Security Groups vs. Network Access Control Lists (NACLs)

| Feature | Security Group (SG) | Network Access Control List (NACL) |
| :--- | :--- | :--- |
| **Boundary Level** | **Instance Level** (Virtual network interface / ENI). | **Subnet Level** (Boundary of the entire subnet). |
| **State Tracking** | **Stateful:** Permitting inbound traffic automatically allows return outbound traffic. | **Stateless:** Inbound and outbound traffic must be explicitly permitted. |
| **Rule Capabilities** | Supports **Allow rules only** (implicit deny). | Supports both **Allow and Deny rules** in numerical order. |
| **Primary Role** | Fine-grained application firewall (e.g., permit port 8080 only from the ALB's SG). | Coarse subnet defense (e.g., blocking malicious CIDR IP blocks). |

#### 4. High Availability: Auto Scaling Groups (ASG) & Application Load Balancers (ALB)
- Production EC2 instances are never deployed as standalone virtual machines. They are grouped in an Auto Scaling Group across multiple Availability Zones behind an Application Load Balancer.
- **Self-Healing Mechanics:** The ALB continuously sends health checks to Spring Boot Actuator's `/actuator/health` endpoint. If a JVM crashes, the ALB initiates connection draining (deregistration delay), while the ASG terminates the dead instance and provisions a fresh VM from the Golden AMI automatically.

#### 5. Storage Architecture: Elastic Block Store (EBS) vs. Instance Store
- **Amazon EBS:** Network-attached virtual disk. Data persists independently of the instance lifecycle (survives stop/start cycles) and can be snapshotted point-in-time to Amazon S3. Used for OS root volumes and persistent state.
- **Instance Store:** Physically attached NVMe SSD integrated directly into the host hardware chassis. Ultra-low microsecond latency and high IOPS, but **ephemeral**: data is permanently lost if the instance stops or terminates. Used for swap files and temporary caches.

#### 6. User Data Bootstrapping
- A shell script executed automatically with `root` privileges during the initial launch phase of an EC2 instance.
- Used for final-mile deployment: pulling the release `.jar` artifact from a private S3 bucket, setting file permissions, and executing `systemctl start my-app.service`.

---

### Container Orchestration: Amazon ECS vs Amazon EKS & EC2 vs Fargate

When deploying containerized Spring Boot microservices, AWS provides two primary orchestrators and two compute launch types:

| Orchestration Dimension | Amazon ECS (Elastic Container Service) | Amazon EKS (Elastic Kubernetes Service) |
| :--- | :--- | :--- |
| **Underlying Architecture** | AWS-proprietary container orchestration engine. | Upstream, CNCF-certified managed Kubernetes. |
| **Operational Overhead** | **Low:** Deeply integrated with native AWS services (IAM, ALB, CloudWatch). | **High:** Requires dedicated Kubernetes administration expertise and tooling. |
| **Control Plane Fee** | **$0 / month:** No charge for cluster management. | **$0.10 / hour (~$73 / month)** control plane fee per cluster. |
| **Portability** | Locked to AWS APIs and Task Definition schemas. | **Fully Portable:** Manifests deploy across AWS (EKS), Azure (AKS), GCP (GKE), and on-prem. |
| **Ecosystem & Tooling** | AWS native: Task Definitions, ECS Services, CloudWatch. | Open-source ecosystem: Helm, Istio, Prometheus, ArgoCD, Calico. |
| **Core Abstractions** | Task Definitions, Tasks, Services, Clusters. | Manifests, Pods, Deployments, Services, Ingress. |

#### Compute Launch Types: EC2 vs. AWS Fargate

```
                    +---------------------------------------------+
                    |    Container Orchestrator (ECS or EKS)      |
                    +---------------------------------------------+
                                   |                       |
                     [ EC2 Launch Type ]       [ Fargate Launch Type ]
                                   |                       |
                      Customer manages OS,      AWS provisions serverless
                      Docker daemon, systemd,   micro-VMs per container;
                      and host kernel tuning    zero OS management
```

1. **EC2 Launch Type (Self-Managed Worker Nodes):**
   - You provision and manage the underlying EC2 instances acting as worker nodes.
   - **Advantage:** Full control over Linux OS kernel parameters, local NVMe storage, custom networking drivers, and spot instance pricing optimizations.
   - **Linux Skills Alignment:** Private cloud systems experience (systemd, kernel limits, memory sizing, and network troubleshooting) applies directly when debugging node pressure, cgroup throttling, or OOM-killed pods.

2. **AWS Fargate Launch Type (Serverless Containers / CaaS):**
   - AWS manages the underlying virtual machine infrastructure. You declare required vCPU and memory (e.g., `2 vCPU, 4 GiB RAM`), and AWS runs the container inside an isolated micro-VM.
   - **Advantage:** Completely eliminates Linux OS maintenance, security patching, and cluster capacity planning.

---
