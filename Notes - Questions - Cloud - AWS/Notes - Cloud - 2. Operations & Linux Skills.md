# Notes - Cloud - 2. Operations & Linux Skills

## 1. Cloud Operations & Enterprise Management Modalities

### AWS Management Modalities: Console, CLI, and Infrastructure as Code (IaC)

In an enterprise cloud ecosystem, infrastructure management and operational interactions are categorized into three distinct modalities:

| Modality | Primary Tooling | Operational Role | Production Viability |
| :--- | :--- | :--- | :--- |
| **AWS Management Console** | Web Browser Graphical UI | Visual inspection, ad-hoc metric monitoring, exploratory learning, and emergency manual triage. | **Forbidden for Provisioning:** Manual creation ("ClickOps") introduces untracked configuration drift, human error, and non-reproducible environments. |
| **AWS CLI & SDKs** | Terminal (`aws`), Python (Boto3), Java (AWS SDK v2) | Ad-hoc administrative queries, shell-level automation, automated pipeline hooks, and programmatic integration. | **Standard for Operations:** Excellent for automated deployment verification, ephemeral queries, and CI/CD script execution. |
| **Infrastructure as Code (IaC)** | **Terraform**, **AWS CloudFormation**, **AWS CDK** | Declarative specification of infrastructure topologies versioned in Git repositories. | **Mandatory Enterprise Standard:** Enforces peer reviews, automated security linting, repeatable multi-environment deployments, and deterministic rollbacks. |

```bash
# Example AWS CLI query to inspect running production instances in JSON format
aws ec2 describe-instances \
    --region ap-south-1 \
    --filters "Name=instance-state-name,Values=running" "Name=tag:Environment,Values=Production" \
    --query "Reservations[*].Instances[*].[InstanceId,PrivateIpAddress,InstanceType,LaunchTime]" \
    --output table
```

---

### AWS Management Console Deep Dive & Operational Features

Senior cloud engineers must understand the underlying operational architecture of the AWS Management Console beyond surface-level navigation:

#### 1. Global vs. Regional Scope
- **Regional Context:** Most compute and data services—including **Amazon EC2**, **Amazon RDS**, **AWS Lambda**, and **Amazon SQS**—are strictly scoped to an AWS Region (e.g., `ap-south-1` for Mumbai, `us-east-1` for N. Virginia). Compute and storage resources provisioned in one region are completely invisible if the console view is set to another region.
- **Global Context:** Identity and domain services—including **AWS IAM**, **Amazon Route 53**, and the **Amazon S3** top-level console (which displays a globally unique bucket namespace)—operate with a global administrative plane.

#### 2. AWS Systems Manager (Session Manager)
- **Zero-Port Secure Shell:** Replaces legacy inbound SSH access (port 22) and static `.pem` private key management.
- **Architecture:** The lightweight SSM Agent running inside the EC2 instance connects outbound to the AWS Systems Manager endpoint over HTTPS (port 443).
- **Security & Compliance:** Administrative shell sessions are authenticated via AWS IAM, fully audited in AWS CloudTrail, and session keystrokes can be streamed encrypted into Amazon CloudWatch Logs or an Amazon S3 audit bucket.

#### 3. AWS CloudShell
- A browser-based Linux terminal pre-authenticated with the administrator's active IAM console credentials and pre-installed with the AWS CLI, Node.js, Python, and Git.
- Eliminates the need to configure local terminal access keys on developer laptops for temporary administration.

#### 4. Amazon CloudWatch Centralized Observability
- Aggregates operational telemetry across compute and application tiers:
  - **Metrics & Alarms:** Tracks CPU utilization, memory pressure (via CloudWatch Agent), network throughput, and disk I/O with automated alarm triggers (SNS notifications or ASG scaling actions).
  - **CloudWatch Logs:** Centralizes distributed Spring Boot application logs (`logback-spring.xml` streaming via CloudWatch agent or JSON appender), eliminating the need to SSH into individual server nodes to view log files.

#### 5. Tag Editor & Resource Groups
- Assigns uniform metadata key-value tags (e.g., `Environment=Production`, `Service=PaymentAPI`, `CostCenter=Engineering`).
- **Resource Groups:** Consolidates all disparate AWS resources (EC2, RDS, S3, SQS) sharing a specific tag set into a single, unified administrative dashboard within large multi-tenant enterprise accounts.

#### 6. AWS Cost Explorer & Billing Dashboard
- Provides multi-dimensional visualization of infrastructure spend:
  - Identifies anomalous spending surges by filtering costs by Service, Region, or Allocation Tag.
  - Highlights idle compute capacity (e.g., oversized EC2 instances or unattached EBS volumes).

---

### Operational Maturity: Progression from Console to IaC

Enterprise engineering adoption follows a standardized three-stage progression of operational maturity:

```
[ Stage 1: Foundational / Visual ] ---> [ Stage 2: Developer / SysOps ] ---> [ Stage 3: Enterprise / Architect ]
- AWS Management Console                - AWS CLI Terminal Scripting            - Infrastructure as Code (IaC)
- Visual mental models of VPCs          - AWS SDK for Java v2 APIs              - Terraform, CloudFormation, CDK
- Ad-hoc testing & inspection           - CI/CD build automation                - Git-driven CI/CD pipelines
```

- **Stage 1 (Console Exploration):** Used during initial architectural discovery and training (e.g., Cloud Practitioner) to visualize networking boundaries, subnet associations, and security rules.
- **Stage 2 (Programmatic Automation):** Developers shift to the AWS CLI and AWS SDK for Java v2 to script operational workflows, perform automated integration tests, and build programmatic application integrations.
- **Stage 3 (Declarative IaC):** Production systems enforce 100% Infrastructure as Code. Manual modifications in the AWS Console are actively prevented via IAM Service Control Policies (SCPs) to ensure zero configuration drift.

---

### Linux Administration & CentOS-to-AlmaLinux Migration Skills

For backend Java engineers, host operating system fluency inside EC2 virtual machines and container worker nodes is critical. Enterprise migrations from **CentOS 7 to AlmaLinux 8/9** demonstrate deep systems-level competence across four core areas:

#### 1. OS Migration & Package Management (`yum` $	o$ `dnf` Application Streams)
- CentOS 7 reached End-of-Life (EOL) in June 2024. Enterprise platforms migrated to binary-compatible RHEL-fork distributions such as **AlmaLinux 8/9**.
- AlmaLinux replaced legacy `yum` with `dnf` (Dandified YUM) and introduced **Application Streams (Modules)**, allowing independent lifecycle management of multiple runtime versions (e.g., JDKs, PostgreSQL clients, Node.js) on the same OS.

```bash
# Verify current OS release
cat /etc/os-release

# Execute in-place CentOS-to-AlmaLinux official migration script
curl -O https://raw.githubusercontent.com/AlmaLinux/almalinux-deploy/master/almalinux-deploy.sh
sudo bash almalinux-deploy.sh

# Re-validate OS release post-migration
cat /etc/os-release

# Inspect and manage runtime application stream modules (e.g., Java runtimes)
dnf module list
dnf module install -y javapackages-runtime
dnf install -y java-21-openjdk-devel
```

#### 2. Java Service Lifecycle Management with `systemd`
- Production Spring Boot applications on EC2 must never be executed interactively via raw `java -jar app.jar` (which dies when the SSH session closes). They are registered and supervised by `systemd`.

```ini
# /etc/systemd/system/payment-service.service
[Unit]
Description=Payment Gateway Spring Boot Microservice
After=syslog.target network.target

[Service]
Type=simple
User=springboot
Group=springboot
WorkingDirectory=/opt/app
ExecStart=/usr/bin/java -Xms2048m -Xmx2048m -XX:+UseG1GC -jar /opt/app/payment-service.jar
SuccessExitStatus=143
Restart=always
RestartSec=10
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
```

```bash
# Reload systemd configuration after unit modifications
systemctl daemon-reload

# Enable service to start automatically upon server reboot
systemctl enable payment-service.service

# Start or restart application after deploying an updated JAR artifact
systemctl restart payment-service.service

# Tail continuous real-time application logs and startup stack traces
journalctl -u payment-service.service -f --lines=100
```

#### 3. Host-Level Networking & Firewall Configuration (`firewalld` & `ss`)
- Even if an AWS Security Group permits inbound traffic on port 8080, an unconfigured host firewall inside the Linux OS will drop packets silently.

```bash
# Verify embedded Tomcat is actively bound and listening on TCP port 8080
ss -tulpn | grep 8080

# Check active status of firewalld service
systemctl status firewalld

# Permanently allow TCP port 8080 through the host OS firewall and reload rules
firewall-cmd --permanent --add-port=8080/tcp
firewall-cmd --reload

# Verify opened ports in firewalld
firewall-cmd --list-ports
```

#### 4. System Resource & JVM Triage (`free`, `top`, `df`)
- Diagnosing host performance degradation, memory pressure, and filesystem locks:

```bash
# 1. Inspect physical RAM and Swap allocation in megabytes
# Heavy swap usage indicates the JVM heap is forcing memory paging to disk, destroying GC performance
free -m

# 2. Monitor real-time CPU and Memory consumption per PID to identify runaway threads
top -b -n 1 | head -n 25

# 3. Check filesystem storage utilization across mount points
# A full root partition or full /var/log causes immediate application crashes and database lockups
df -h
```

---
