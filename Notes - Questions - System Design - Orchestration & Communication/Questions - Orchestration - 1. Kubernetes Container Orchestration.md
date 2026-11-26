# Kubernetes Container Orchestration - Interview Q&A

## Kubernetes Cluster Architecture

### Control Plane Components

#### What are the core components of the Kubernetes Control Plane and how do they coordinate cluster state?

> **Direct Answer:**
> The Kubernetes Control Plane is the cluster's brain, consisting of `kube-apiserver`, `etcd`, `kube-scheduler`, `kube-controller-manager`, and `cloud-controller-manager`. The `kube-apiserver` acts as the sole gateway and front door, exposing the REST API, authenticating requests, and reading/writing cluster state to `etcd`. Controllers in `kube-controller-manager` execute reconciliation loops to converge actual state to declared state, while `kube-scheduler` assigns unscheduled pods to eligible worker nodes.

> **Key Nuance / Trade-off:**
> `etcd` is the single source of truth using the Raft consensus algorithm; no component except `kube-apiserver` communicates directly with `etcd`. Loss of an `etcd` quorum halts all cluster administrative and scheduling operations, though already running worker node workloads continue executing uninterrupted.

> **Reference Concept:** [Control Plane Components](./Notes%20-%20Orchestration%20-%201.%20Kubernetes%20Container%20Orchestration.md#control-plane-components)

---

### Worker Node Components

#### What are the core components of a Kubernetes Worker Node and how does kubelet interact with the container runtime?

> **Direct Answer:**
> A Kubernetes Worker Node executes containerized workloads and consists of the `kubelet`, `kube-proxy`, and a CRI-compliant Container Runtime (e.g., `containerd` or `CRI-O`). The `kubelet` is the primary node agent that receives `PodSpecs` from `kube-apiserver` and instructs the container runtime over the Container Runtime Interface (CRI) gRPC protocol to pull images, create namespaces, and manage container lifecycles. `kube-proxy` configures Linux `iptables` or `IPVS` rules to manage Layer 4 networking and Service IP translation.

> **Key Nuance / Trade-off:**
> Docker Engine is no longer used directly as a runtime since Kubernetes removed Dockershim in v1.24; runtimes must adhere directly to CRI. If `kubelet` stops running or loses network connectivity, running pods continue executing on the node, but the control plane marks the node as `NotReady` after a timeout and initiates pod eviction to other nodes.

> **Reference Concept:** [Worker Node Components](./Notes%20-%20Orchestration%20-%201.%20Kubernetes%20Container%20Orchestration.md#worker-node-components)

---

### Pods and Multi-Container Patterns

#### What is a Kubernetes Pod, why does Kubernetes deploy Pods instead of individual containers, and what are common multi-container patterns?

> **Direct Answer:**
> A Pod is the smallest deployable computing unit in Kubernetes, encapsulating one or more tightly coupled Linux containers that share storage volumes, an IP address, and a Linux network namespace. Kubernetes deploys Pods rather than raw containers to support composite co-located processes that communicate over `localhost` and share lifecycle bindings. Common multi-container patterns include the **Sidecar** (augmenting functionality like log shipping or service mesh proxies), **Ambassador** (proxying external connections like DB routing), and **Adapter** (normalizing application metrics and monitoring output).

> **Key Nuance / Trade-off:**
> All containers in a single Pod share resource constraints, fate, and network ports; two containers in the same Pod cannot bind to the same network port. Overusing multi-container pods for loosely coupled services couples their deployments and scaling lifecycles, violating microservice independence.

> **Reference Concept:** [Pods and Multi-Container Patterns](./Notes%20-%20Orchestration%20-%201.%20Kubernetes%20Container%20Orchestration.md#pods-and-multi-container-patterns)

---

## Microservice Deployment Lifecycle

### Microservice Deployment Pipeline

#### How do you deploy microservices on Kubernetes (K8s)?

> **Direct Answer:**
> Deploying a microservice to Kubernetes follows a five-step declarative pipeline: (1) **Containerize** the application using a multi-stage Dockerfile packaging the runtime and compiled artifact into an immutable OCI image; (2) **Push** the tagged image to a secure centralized container registry (e.g., ECR, GCR, Docker Hub); (3) **Author Declarative Manifests** defining a `Deployment` (declaring replica count, pod template, resource requests/limits, probes) and a `Service` (providing internal stable DNS and load balancing); (4) **Apply Configuration** via `kubectl apply -f deployment.yaml` to the `kube-apiserver`, which reconciles the state via the scheduler and node kubelets; (5) **Expose Traffic** externally by defining an `Ingress` resource or cloud `LoadBalancer`.

> **Key Nuance / Trade-off:**
> Always decouple configuration from the container image using `ConfigMaps` and `Secrets` rather than hardcoding environment variables into Dockerfiles. Never use the `:latest` image tag in production manifests, as it circumvents rollout versioning and causes inconsistent node image caching.

> **Reference Concept:** [Microservice Deployment Pipeline](./Notes%20-%20Orchestration%20-%201.%20Kubernetes%20Container%20Orchestration.md#microservice-deployment-pipeline)

---

## Networking and Service Discovery

### Kubernetes Services and Networking

#### How do Kubernetes Services work, and what are the differences between ClusterIP, NodePort, LoadBalancer, and Headless services?

> **Direct Answer:**
> A Kubernetes Service provides a single, stable IP address and DNS endpoint that abstracts away the volatile, ephemeral IP addresses of underlying Pods selected via label selectors. **ClusterIP** is the default internal-only virtual IP used for east-west service-to-service communication. **NodePort** exposes the service externally on a high-range static port (`30000-32767`) across every worker node. **LoadBalancer** automatically provisions a cloud provider external Layer 4/Layer 7 load balancer routing into NodePorts. **Headless** (`clusterIP: None`) returns the direct A/AAAA records of matching Pod IPs without virtual IP routing, required for peer-to-peer stateful topologies like Kafka or Cassandra.

> **Key Nuance / Trade-off:**
> ClusterIPs are virtual IPs implemented in the Linux kernel via `kube-proxy` iptables/IPVS rules, meaning pinging a ClusterIP directly with ICMP usually fails. NodePort exposes node port security risks and port-exhaustion limits, whereas LoadBalancer incurs cloud infrastructure provider costs for every provisioned service.

> **Reference Concept:** [Kubernetes Services and Networking](./Notes%20-%20Orchestration%20-%201.%20Kubernetes%20Container%20Orchestration.md#kubernetes-services-and-networking)

---

### Ingress and Ingress Controllers

#### What is the difference between a Kubernetes Service and an Ingress, and when would you use an Ingress Controller?

> **Direct Answer:**
> A Kubernetes Service operates at Layer 4 (TCP/UDP transport layer), directing traffic to pod endpoints using IP addresses and ports. An **Ingress** operates at Layer 7 (Application layer), providing intelligent HTTP/HTTPS routing, host/domain-based routing (e.g., `api.example.com`), URL path matching (e.g., `/orders` vs `/catalog`), and centralized SSL/TLS termination. An Ingress Controller (such as Ingress-NGINX or AWS ALB Controller) is the reverse-proxy implementation daemon running inside the cluster that translates Ingress YAML definitions into live routing configurations.

> **Key Nuance / Trade-off:**
> Creating an Ingress resource without an active Ingress Controller running in the cluster does nothing; the resource remains an inert record in `etcd`. Using an Ingress Controller dramatically reduces cloud infrastructure costs by consolidating dozens of microservices behind a single external load balancer instead of provisioning separate cloud LoadBalancers per service.

> **Reference Concept:** [Ingress and Ingress Controllers](./Notes%20-%20Orchestration%20-%201.%20Kubernetes%20Container%20Orchestration.md#ingress-and-ingress-controllers)

---

## Health Checks and Zero-Downtime Operations

### Health Checks: Liveness, Readiness, and Startup Probes

#### What is the difference between Liveness, Readiness, and Startup probes, and what happens when each fails?

> **Direct Answer:**
> A **Liveness Probe** determines if the container process is alive and healthy; failure triggers `kubelet` to immediately kill the container and restart it according to its `restartPolicy`. A **Readiness Probe** determines if the container is ready to accept incoming network traffic; failure temporarily removes the Pod IP from the Service `EndpointSlice` so no traffic is routed to it, without killing the container. A **Startup Probe** disables both liveness and readiness checks during slow application initialization; failure restarts the container only after exhausting its configured `failureThreshold`.

> **Key Nuance / Trade-off:**
> A catastrophic anti-pattern is checking downstream databases or message brokers in a Liveness probe. If the database experiences transient latency or downtime, all microservice pods will fail their liveness probes simultaneously, creating an unrecoverable cluster-wide reboot storm that exacerbates the database outage.

> **Reference Concept:** [Health Checks: Liveness, Readiness, and Startup Probes](./Notes%20-%20Orchestration%20-%201.%20Kubernetes%20Container%20Orchestration.md#health-checks-liveness-readiness-and-startup-probes)

---

### Zero-Downtime Rolling Updates and Rollback Strategies

#### How do Kubernetes Rolling Updates achieve zero downtime, and how do maxSurge and maxUnavailable control pod replacement?

> **Direct Answer:**
> Kubernetes achieves zero-downtime rolling updates by incrementally creating new Pods with updated container images while keeping existing Pods active until the new ones pass their Readiness probes. The process is governed by two parameters in the Deployment strategy: `maxSurge` (how many extra pods can be created above the desired replica count) and `maxUnavailable` (how many pods can be unavailable relative to the desired count). By configuring `maxUnavailable: 0` and `maxSurge: 25%`, Kubernetes guarantees that service capacity never drops below 100% during a deployment.

> **Key Nuance / Trade-off:**
> Achieving true zero-downtime requires coordinating with `preStop` lifecycle hooks. Because `kube-proxy` takes a few seconds to remove terminating Pod IPs from iptables rules across all worker nodes, a container that terminates immediately upon receiving `SIGTERM` will drop in-flight requests. Adding a `preStop: exec: command: ["sh", "-c", "sleep 10"]` prevents request drops by delaying container shutdown until endpoint caches synchronize.

> **Reference Concept:** [Zero-Downtime Rolling Updates and Rollback Strategies](./Notes%20-%20Orchestration%20-%201.%20Kubernetes%20Container%20Orchestration.md#zero-downtime-rolling-updates-and-rollback-strategies)

---

## Configuration Management and Scaling

### Configuration Management: ConfigMaps and Secrets

#### How do ConfigMaps and Secrets decouple configuration in Kubernetes, and what happens when you update a mounted ConfigMap?

> **Direct Answer:**
> ConfigMaps and Secrets decouple runtime configuration and confidential credentials from container images, enabling the identical image artifact to run across development, staging, and production environments. When injected as **Environment Variables**, values are fixed at pod startup and will **not** update if the underlying ConfigMap/Secret changes until the pod is recreated. When injected as **Volume Mounts**, Kubernetes updates the projected files inside the container automatically via symlink updates without requiring pod restarts.

> **Key Nuance / Trade-off:**
> Secrets in Kubernetes are only `base64` encoded by default, not encrypted; they must be backed by `etcd` encryption-at-rest or external operators (e.g., HashiCorp Vault or External Secrets Operator). Furthermore, applications reading mounted volume files must implement dynamic file-watch listeners to hot-reload configuration in memory without a pod restart.

> **Reference Concept:** [Configuration Management: ConfigMaps and Secrets](./Notes%20-%20Orchestration%20-%201.%20Kubernetes%20Container%20Orchestration.md#configuration-management-configmaps-and-secrets)

---

### Autoscaling: Horizontal Pod Autoscaler (HPA)

#### How does the Horizontal Pod Autoscaler (HPA) scale microservices, and what metrics does it evaluate?

> **Direct Answer:**
> The Horizontal Pod Autoscaler (HPA) continuously queries the Kubernetes Metrics API (every 15 seconds by default) and adjusts the replica count of Deployments or StatefulSets to match target utilization thresholds. It natively evaluates resource metrics like CPU and Memory utilization against requested resources, and can evaluate custom application metrics (e.g., HTTP request rate or Kafka consumer lag) via Prometheus adapters. The desired replica count is computed as: $\lceil \text{Current Replicas} \times (\text{Current Metric} / \text{Target Metric}) \rceil$.

> **Key Nuance / Trade-off:**
> HPA requires explicit resource `requests` defined in the container spec; without resource requests, HPA cannot calculate percentage utilization and will fail to scale. To prevent "flapping" or thrashing (rapid scaling up and down during spiky traffic), configure stabilization windows (`stabilizationWindowSeconds`) in the HPA `behavior` specification.

> **Reference Concept:** [Autoscaling: Horizontal Pod Autoscaler (HPA)](./Notes%20-%20Orchestration%20-%201.%20Kubernetes%20Container%20Orchestration.md#autoscaling-horizontal-pod-autoscaler-hpa)
