# Notes - Cloud - 5. Messaging & Storage (SQS, SNS, S3)

## 1. Asynchronous Messaging Architecture (SQS & SNS)

### Amazon SQS & SNS Messaging Architecture & Spring Cloud AWS Integration

Enterprise distributed architectures rely on **Amazon SNS** and **Amazon SQS** to decouple synchronous microservices:

| Dimension | Amazon SNS (Simple Notification Service) | Amazon SQS (Simple Queue Service) |
| :--- | :--- | :--- |
| **Communication Paradigm** | **Publish / Subscribe (Pub/Sub):** Push-based fanout. | **Point-to-Point Queue:** Pull-based message consumption. |
| **Consumer Topology** | **One-to-Many:** One published message is dispatched to multiple subscribers (SQS queues, HTTP webhooks, Lambda). | **One-to-One:** Each message is processed and deleted by a single consuming worker. |
| **Message Persistence** | Ephemeral: If no subscribers exist, the message is dropped. | Durable: Messages persist in the queue (retention from 1 minute to 14 days; default 4 days). |

#### Amazon SQS: Standard vs. FIFO Queues

| Feature | Amazon SQS Standard | Amazon SQS FIFO (First-In-First-Out) |
| :--- | :--- | :--- |
| **Throughput Capacity** | Nearly unlimited transactions per second. | Up to **300 msg/sec** (up to **3,000 msg/sec** with batching of 10). |
| **Message Ordering** | Best-effort: Messages may occasionally arrive out of sequence. | **Strict Ordering:** Preserved strictly within each Message Group ID. |
| **Delivery Guarantee** | **At-least-once delivery:** Occasional duplicate messages occur. | **Exactly-once processing:** 5-minute deduplication window (via deduplication ID). |
| **Consumer Requirement** | Java consumer **must be idempotent**. | Safe for strict financial transactions, sequential ledgers, and inventory updates. |

#### Spring Cloud AWS 3.x Integration (`@SqsListener`)

```xml
<!-- pom.xml -->
<dependency>
    <groupId>io.awspring.cloud</groupId>
    <artifactId>spring-cloud-aws-starter-sqs</artifactId>
</dependency>
```

```java
@Service
public class OrderMessageConsumer {

    @SqsListener(value = "order-processing-queue")
    public void processOrderMessage(OrderEvent event) {
        // Spring Cloud AWS manages 20-second long-polling, deserialization,
        // visibility timeout extensions, and automatic message deletion upon success.
        log.info("Processing order event: {}", event.getOrderId());
        orderService.fulfillOrder(event);
    }
}
```

#### Core Operational Mechanics Managed by SQS & `@SqsListener`:
1. **Long-Polling (WaitTimeSeconds = 20):** Holds the polling HTTP connection open for up to 20 seconds waiting for messages, dramatically reducing empty `ReceiveMessage` API calls and lowering AWS bills.
2. **Visibility Timeout:** When a worker pulls a message, SQS marks it invisible to other workers for a configurable duration (e.g., 30 seconds). If the worker finishes successfully, Spring Cloud AWS deletes the message. If the worker crashes, the visibility timeout expires, and the message reappears on the queue.
3. **Dead Letter Queues (DLQ) & MaxReceiveCount:** If a poison-pill message causes unhandled exceptions, SQS increments its `receiveCount`. Once it exceeds `maxReceiveCount` (e.g., 3 retries), SQS routes the message automatically to a DLQ for isolated investigation without blocking the queue.

---

## 2. Enterprise Object Storage Architecture (Amazon S3)

### Amazon S3 Enterprise Patterns: Presigned URLs, Multipart & Lifecycle Tiers

Amazon Simple Storage Service (S3) provides highly durable (99.999999999% durability), HTTP REST-based object storage. Enterprise Java applications utilize four core S3 patterns:

#### 1. Presigned URLs (`S3Presigner` - AWS SDK for Java v2)
- **The Problem:** Routing large file uploads through a Spring Boot API server causes severe network bandwidth saturation, high thread consumption, and JVM OutOfMemory errors.
- **The Solution:** The Spring Boot backend generates a short-lived (e.g., 15-minute) **Presigned URL** signed with the backend's IAM credentials and returns it to the client. The frontend client uploads or downloads the binary directly to/from S3, completely bypassing the backend application servers.

```java
@Service
public class DocumentStorageService {

    public URL generateUploadPresignedUrl(String bucketName, String objectKey) {
        try (S3Presigner presigner = S3Presigner.create()) {
            PutObjectRequest objectRequest = PutObjectRequest.builder()
                    .bucket(bucketName)
                    .key(objectKey)
                    .contentType("application/pdf")
                    .build();

            PutObjectPresignRequest presignRequest = PutObjectPresignRequest.builder()
                    .signatureDuration(Duration.ofMinutes(15))
                    .putObjectRequest(objectRequest)
                    .build();

            return presigner.presignPutObject(presignRequest).url();
        }
    }
}
```

#### 2. Multipart Uploads (>100 MB)
- For files larger than 100 MB, Amazon S3 requires **Multipart Uploads**.
- Files are split into distinct byte chunks and uploaded in parallel. If an individual part fails over an unstable network, only that specific chunk is retried rather than restarting the multi-gigabyte upload from scratch.

#### 3. S3 Event Notifications $	o$ AWS Lambda
- S3 triggers asynchronous cloud events (e.g., `s3:ObjectCreated:*`) upon successful file uploads.
- An AWS Lambda function is invoked asynchronously to parse documents, extract metadata, generate image thumbnails, or execute virus scanning, keeping the primary API completely non-blocking.

#### 4. S3 Storage Classes & Cost Optimization Lifecycle Policies

| Storage Class | Data Access Latency | Minimum Storage Duration | Recommended Enterprise Use Case |
| :--- | :--- | :--- | :--- |
| **S3 Standard** | Milliseconds | None | Frequently accessed production data, active uploads, dynamic web assets. |
| **S3 Standard-IA (Infrequent Access)** | Milliseconds | 30 Days | Older files accessed <1x per month. Lower storage cost with a per-GB retrieval fee. |
| **S3 Glacier Flexible Retrieval** | Minutes to Hours | 90 Days | Regulatory compliance records, quarterly audit archives, long-term backups. |
| **S3 Glacier Deep Archive** | 12 to 48 Hours | 180 Days | Lowest-cost cloud storage tier ($1/TB/month). Multi-year compliance data accessed rarely. |

---
