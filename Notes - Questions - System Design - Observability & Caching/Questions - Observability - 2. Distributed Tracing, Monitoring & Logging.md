# Observability - Questions - Chapter 2: Distributed Tracing, Monitoring & Logging

## Distributed Tracing: Request Flow and Context Propagation

### Core Tracing Mechanics: Trace ID, Span ID, and Span Trees

#### How do you implement distributed tracing?

> **Direct Answer:** Distributed tracing is implemented by generating a globally unique Trace ID at the entry gateway and propagating it across all downstream HTTP/gRPC boundaries using standard headers such as W3C `traceparent` or B3. Within each microservice, distinct units of work are demarcated as Spans (containing a Span ID, Parent Span ID, timestamps, and metadata tags), which are asynchronously dispatched to a centralized collector (Jaeger, Zipkin, or OpenTelemetry Collector). Modern implementations leverage Spring Boot 3 Micrometer Tracing or the OpenTelemetry SDK to automatically inject Trace IDs into the logging Mapped Diagnostic Context (MDC), tightly coupling traces to application log output.

> **Key Nuance / Trade-off:** High-throughput production environments cannot afford 100% trace ingestion due to network saturation and astronomical APM storage costs. Architects implement tail-based sampling in the OpenTelemetry Collector—retaining 100% of failed requests (HTTP 5xx) and latency anomalies while dropping routine high-speed transactions—rather than blind head-based sampling that risks dropping critical incident traces.

> **Reference Concept:** [Core Tracing Mechanics: Trace ID, Span ID, and Span Trees](./Notes%20-%20Observability%20-%202.%20Distributed%20Tracing%2C%20Monitoring%20%26%20Logging.md#core-tracing-mechanics-trace-id-span-id-and-span-trees)

---

### Network Context Propagation: W3C TraceContext and B3 Specifications

### OpenTelemetry Standard and Vendor-Neutral Instrumentation

### Telemetry Sampling Strategies: Head-Based vs Tail-Based Sampling

### Log Correlation via Mapped Diagnostic Context and Spring Boot 3 Micrometer

---

## Real-Time Microservice Monitoring: The Three Pillars and Golden Signals

### The Three Pillars of Distributed Observability: Metrics, Logs, and Traces

### Prometheus Pull-Based Metrics Architecture and Scrape Mechanics

### Correlated Observability and Unified Grafana Visualization

### Alerting on the RED Method: Rate, Errors, and Duration

#### How to monitor microservices in real time?

> **Direct Answer:** Real-time microservice monitoring shifts from isolated server resource tracking to unified distributed observability by consolidating the Three Pillars: Metrics (numerical time-series scraped via Prometheus), Logs (structured JSON streamed from container stdout via Fluent Bit/Loki), and Traces (request path tracking via OpenTelemetry). Production systems visualize these streams on unified Grafana dashboards and base on-call alerting strictly on the RED method (Rate, Errors, Duration) representing direct user experience. Crucially, the Trace ID is injected into the application MDC to enable seamless pivot navigation from metric latency spikes to specific traces and correlated log entries.

> **Key Nuance / Trade-off:** The single greatest operational hazard in Prometheus monitoring is the High Cardinality Trap—adding dynamic labels like `user_id` or `order_id` to metrics. Each unique permutation of labels instantiates an independent in-memory time-series array, triggering catastrophic Prometheus out-of-memory (OOM) crashes. Dynamic identifiers must reside exclusively in structured logs and distributed trace spans.

> **Reference Concept:** [Alerting on the RED Method: Rate, Errors, and Duration](./Notes%20-%20Observability%20-%202.%20Distributed%20Tracing%2C%20Monitoring%20%26%20Logging.md#alerting-on-the-red-method-rate-errors-and-duration)

---

### The High Cardinality Trap in Time-Series Datastores

---

## Centralized Logging and Metrics Infrastructure: Tooling and Architecture

### Modern Observability Stacks: Generation, Collection, Storage, and UI

### The Indexing Tax: Elasticsearch (ELK) vs Grafana Loki (PLG)

#### What tools do you use for logs and metrics?

> **Direct Answer:** Modern production stacks decouple telemetry generation, shipping, storage, and visualization: Micrometer and OpenTelemetry SDKs generate metrics and traces, Prometheus stores metrics via its active pull scraping model, Fluent Bit or Promtail ship container stdout logs, and Grafana serves as the unified visualization and PromQL querying engine. For log storage, organizations choose between the full-text search capabilities of Elasticsearch (ELK Stack) and the lightweight, label-indexed architecture of Grafana Loki (PLG Stack). An OpenTelemetry Collector operates within the cluster as a vendor-neutral universal router, exporting telemetry to self-hosted backends or SaaS platforms without application code changes.

> **Key Nuance / Trade-off:** Elasticsearch imposes a massive "indexing tax" because it constructs inverted indexes for every token across terabytes of log data, requiring heavy JVM heap allocations and expensive enterprise storage. Grafana Loki eliminates this tax by indexing only Kubernetes metadata labels while storing raw compressed logs in object storage (e.g., AWS S3), cutting log infrastructure costs by 70–80% at the expense of slower unstructured text grep performance.

> **Reference Concept:** [The Indexing Tax: Elasticsearch (ELK) vs Grafana Loki (PLG)](./Notes%20-%20Observability%20-%202.%20Distributed%20Tracing%2C%20Monitoring%20%26%20Logging.md#the-indexing-tax-elasticsearch-elk-vs-grafana-loki-plg)

---

### OpenTelemetry Collector as a Universal Telemetry Router

### Build vs Buy: Self-Hosted Infrastructure vs Managed SaaS Cost Control
