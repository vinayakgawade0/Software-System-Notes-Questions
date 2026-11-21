# Questions - Cloud - 5. Messaging & Storage (SQS, SNS, S3)

### How do AWS SNS and SQS differ, how do SQS Standard and FIFO compare, and how does `@SqsListener` in Spring Cloud AWS 3.x manage message consumption?

> **Direct Answer:**
> - **Amazon SNS** is a push-based pub/sub service that fans out one published message to multiple subscribers; **Amazon SQS** is a pull-based, point-to-point queue where each message is consumed and deleted by a single worker.
> - **SQS Standard** provides nearly unlimited throughput with at-least-once delivery (requiring idempotent Java consumers); **SQS FIFO** guarantees strict message ordering and exactly-once processing up to 3,000 messages/second with batching.
> - In Spring Cloud AWS 3.x, `@SqsListener` automatically manages 20-second long-polling, deserialization, visibility timeout extensions, error retries, and message deletion upon successful execution.
>
> **Key Nuance / Trade-off:**
> - Long-polling (20 seconds) holds the HTTP request open until a message arrives, eliminating costly empty `ReceiveMessage` API polling fees. If a consumer throws an unhandled exception, the message's visibility timeout expires and its `receiveCount` increments; once it exceeds `maxReceiveCount`, SQS automatically routes the poison-pill message to a configured Dead Letter Queue (DLQ) for isolated triage.
>
> **Reference Concept:** `Amazon SQS & SNS Messaging Architecture & Spring Cloud AWS Integration`

---

### How do you handle large file uploads/downloads in Spring Boot with Amazon S3 using Presigned URLs, Multipart Uploads, Event Notifications, and Storage Classes?

> **Direct Answer:**
> - Use the AWS SDK for Java v2 `S3Presigner` to generate short-lived (e.g., 15-minute) **Presigned URLs** so frontend clients upload or download files directly to/from S3, completely bypassing the Spring Boot API to prevent JVM heap exhaustion and network bandwidth saturation.
> - For files >100 MB, utilize **Multipart Uploads** to stream chunks in parallel with resumable failure recovery.
> - Configure **S3 Event Notifications** (`s3:ObjectCreated:*`) to trigger asynchronous AWS Lambda workers for background image processing or virus scanning.
> - Define **S3 Lifecycle Policies** to transition aging objects automatically from S3 Standard $\to$ S3 Standard-IA (Infrequent Access) $\to$ S3 Glacier Deep Archive to optimize long-term storage costs.
>
> **Key Nuance / Trade-off:**
> - Presigned URLs provide temporary, cryptographically signed access without requiring clients to possess AWS credentials or broad bucket permissions. When using presigned upload URLs, enforce restrictive expiration windows and strict HTTP header validation (e.g., exact `Content-Type` matching) to prevent unauthorized file uploads.
>
> **Reference Concept:** `Amazon S3 Enterprise Patterns: Presigned URLs, Multipart & Lifecycle Tiers`

---
