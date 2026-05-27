# AWS Messaging & Streaming — Practice Questions

> Topics: Amazon SQS, Amazon SNS, Amazon Kinesis, Amazon MQ
> 30 scenario-based questions for SAA-C03 preparation

---

## Amazon SQS (Q1–Q10)

**Question 1:**
A web application receives sudden traffic spikes during flash sales, causing the backend order-processing service to become overwhelmed and drop requests. The team wants to ensure that no orders are lost, even if the processing layer is temporarily slower than the ingestion rate. Which architecture best solves this problem?

- [ ] Add more EC2 instances to the backend with a fixed Auto Scaling group
- [ ] Use Amazon SNS to push orders directly to the backend
- [x] Place an Amazon SQS queue between the web tier and the order-processing service to buffer requests
- [ ] Use Amazon Kinesis Data Firehose to deliver orders to the backend

> **Explanation:** SQS acts as a buffer between producers and consumers. During traffic spikes, messages queue up and the backend processes them at its own pace. No orders are lost because messages persist in the queue (up to 14 days). This is the textbook decoupling pattern.

---

**Question 2:**
A financial trading platform must process trade settlement messages in the exact order they were submitted, and each message must be processed exactly once. The system handles around 200 messages per second. Which SQS configuration meets these requirements?

- [ ] Standard Queue with message deduplication IDs
- [ ] Standard Queue with a Lambda consumer that tracks order
- [x] FIFO Queue with Message Group ID set to the trading account ID
- [ ] Standard Queue with visibility timeout set to 0

> **Explanation:** FIFO queues guarantee exactly-once delivery and strict ordering within a Message Group. At 200 msg/s, this is well within the FIFO limit of 300 msg/s (or 3,000 with batching). Setting the Message Group ID to the account ID ensures per-account ordering while allowing parallel processing across different accounts.

---

**Question 3:**
A development team reports that some SQS messages are being processed twice by their consumer fleet. The application uses a Standard Queue, and each message takes approximately 2 minutes to process. The default visibility timeout is in place. What is the most likely cause, and how should it be fixed?

- [ ] The queue is misconfigured as FIFO — switch to Standard
- [x] The default visibility timeout (30 seconds) is shorter than the processing time (2 minutes) — increase it to at least 3 minutes
- [ ] The consumers are polling too fast — enable long polling
- [ ] The messages are too large — use the SQS Extended Client Library

> **Explanation:** With the default 30-second visibility timeout, a message becomes visible again before the consumer finishes processing (2 minutes). Another consumer then picks it up, causing duplicate processing. The fix is to increase the visibility timeout beyond the processing time. Consumers can also call ChangeMessageVisibility API to extend the timeout mid-processing.

---

**Question 4:**
An application sends image metadata messages to SQS, but some messages contain high-resolution image payloads of 5 MB each. SQS rejects these messages. How should the architecture be modified to handle large payloads?

- [ ] Increase the SQS message size limit to 10 MB in the queue settings
- [ ] Compress the images to under 256 KB before sending
- [x] Use the SQS Extended Client Library to store the image payload in S3 and send a reference pointer in the SQS message
- [ ] Split each message into 20 smaller messages of 256 KB each

> **Explanation:** SQS has a hard limit of 256 KB per message. The SQS Extended Client Library stores payloads larger than 256 KB in S3 and places a pointer/reference in the SQS message. The consumer library transparently retrieves the large payload from S3.

---

**Question 5:**
A company wants to scale its EC2 consumer fleet based on the number of messages waiting in an SQS queue. Which CloudWatch metric and AWS service combination enables this automatic scaling?

- [ ] NumberOfMessagesSent metric → AWS Lambda auto-invocation
- [ ] CPUUtilization of consumers → EC2 Auto Scaling
- [x] ApproximateNumberOfMessagesVisible metric → CloudWatch Alarm → EC2 Auto Scaling Group policy
- [ ] ApproximateNumberOfMessagesDeleted metric → EventBridge → Step Functions

> **Explanation:** ApproximateNumberOfMessagesVisible represents the queue depth (messages waiting to be processed). A CloudWatch Alarm monitors this metric and triggers an Auto Scaling policy to add or remove EC2 consumers. This is the classic SQS + ASG scaling pattern.

---

**Question 6:**
A team notices excessive API costs from their SQS consumers. Investigation reveals that consumers make thousands of ReceiveMessage calls per minute, most of which return empty responses. What is the most cost-effective fix?

- [ ] Reduce the number of consumer instances
- [ ] Switch from Standard to FIFO queue
- [x] Enable Long Polling by setting WaitTimeSeconds to 20 seconds on the queue
- [ ] Increase the visibility timeout to reduce polls

> **Explanation:** Short Polling (default) returns immediately even if no messages exist, generating many empty responses that incur API charges. Long Polling waits up to 20 seconds for messages to arrive, dramatically reducing the number of empty API calls and lowering costs. Long Polling is always recommended.

---

**Question 7:**
A message processing application has a bug that causes certain messages to fail repeatedly. The team wants to isolate these poison messages for later analysis without losing them. What SQS feature should they configure?

- [ ] Enable message encryption with KMS
- [ ] Create a CloudWatch alarm on FailedMessages metric
- [x] Configure a Dead-Letter Queue (DLQ) with a Redrive Policy, setting maxReceiveCount to a suitable threshold (e.g., 3)
- [ ] Set the message retention to 14 days

> **Explanation:** A Dead-Letter Queue receives messages that fail to be processed after exceeding the maxReceiveCount threshold. The Redrive Policy connects the source queue to the DLQ. Once the bug is fixed, the team can use Redrive to Source to replay the failed messages back to the original queue.

---

**Question 8:**
A company needs to allow an S3 bucket in Account A to send event notifications to an SQS queue in Account B. Which access mechanism enables this cross-service, cross-account integration?

- [ ] IAM Role in Account A with sqs:SendMessage permission
- [ ] VPC Peering between the two accounts
- [x] SQS Access Policy (resource-based policy) on the queue in Account B, granting the S3 service principal in Account A permission to send messages
- [ ] Create the SQS queue in Account A instead

> **Explanation:** SQS Access Policies are resource-based policies (similar to S3 bucket policies) that grant permissions to other AWS accounts or services. To allow S3 to send notifications to SQS, the queue's Access Policy must allow the S3 service principal (s3.amazonaws.com) with the appropriate condition for the source bucket.

---

**Question 9:**
An order processing system must introduce a 10-minute delay before processing new orders to allow customers to cancel. Which SQS feature achieves this without modifying consumer logic?

- [ ] Set the visibility timeout to 10 minutes
- [ ] Use a Step Functions Wait state before sending to SQS
- [x] Configure a Delay Queue by setting DelaySeconds to 600 seconds (10 minutes) on the queue
- [ ] Use Long Polling with WaitTimeSeconds set to 600

> **Explanation:** Delay Queues postpone the delivery of new messages for a configured period (up to 15 minutes). Setting DelaySeconds to 600 makes messages invisible to consumers for 10 minutes after being sent. This is different from visibility timeout, which applies after a consumer receives the message.

---

**Question 10:**
A FIFO queue is being used to process financial transactions. The queue name is `transactions-queue`. Producers report receiving errors when trying to send messages. What is the most likely issue?

- [ ] FIFO queues cannot be used for financial transactions
- [x] The queue name does not end in `.fifo` — FIFO queue names must end with `.fifo`
- [ ] The producers are exceeding the 256 KB message size limit
- [ ] FIFO queues require KMS encryption to be enabled

> **Explanation:** FIFO queue names **must** end in `.fifo` (e.g., `transactions-queue.fifo`). This is a hard requirement — without the `.fifo` suffix, the queue is treated as a Standard Queue, and FIFO-specific parameters like Message Group ID and Deduplication ID will be rejected.

---

## Amazon SNS (Q11–Q16)

**Question 11:**
An e-commerce platform needs to notify multiple downstream services (inventory, shipping, analytics, and email) whenever a new order is placed. Currently, the order service calls each downstream service directly, creating tight coupling. What is the best solution?

- [ ] Use SQS with multiple consumers polling the same queue
- [ ] Use Amazon EventBridge to route events
- [x] Publish the order event to an SNS Topic, with each downstream service subscribing via its own SQS queue (SNS + SQS Fan-Out pattern)
- [ ] Use Kinesis Data Streams with enhanced fan-out

> **Explanation:** The SNS + SQS Fan-Out pattern is the gold standard for one-to-many messaging. The order service publishes once to an SNS topic, and SNS pushes the message to all subscribed SQS queues. Each downstream service has its own queue for independent processing, buffering, and retry. Adding new services requires only subscribing a new queue — no code changes.

---

**Question 12:**
An S3 bucket triggers event notifications on `ObjectCreated` events. The company needs to send these events to three different SQS queues for thumbnail generation, metadata extraction, and audit logging. However, S3 event notification rules only allow one destination per event type per prefix. How can this be achieved?

- [ ] Create three identical S3 buckets, each with its own notification rule
- [ ] Use three different S3 prefixes and one queue per prefix
- [x] Configure S3 to send the event to an SNS topic, then subscribe all three SQS queues to that topic (fan-out)
- [ ] Use AWS Lambda to read from one SQS queue and copy to two others

> **Explanation:** S3 event notifications support only one destination per event type per prefix. To send the same event to multiple SQS queues, use SNS as a multiplexer: S3 → SNS Topic → multiple SQS queues. This is the fan-out pattern specifically designed for this constraint.

---

**Question 13:**
An SNS topic receives order events with attributes like `orderType` (electronics, clothing, food) and `region` (us-east, eu-west). The analytics team only wants to receive electronics orders from eu-west. How should this be configured?

- [ ] Create a separate SNS topic for each orderType/region combination
- [ ] Use Lambda to filter messages before forwarding
- [x] Attach a Message Filter Policy to the analytics subscription: `{"orderType": ["electronics"], "region": ["eu-west"]}`
- [ ] Configure S3 event prefix filtering on the SNS topic

> **Explanation:** SNS Message Filtering allows each subscriber to define a JSON filter policy that selectively receives only matching messages. This eliminates the need for multiple topics or intermediate filtering logic. Subscribers without a filter policy receive ALL messages.

---

**Question 14:**
A company wants to use SNS FIFO topics to fan out ordered messages to multiple consumers. They plan to subscribe Lambda functions and HTTP endpoints. Will this architecture work?

- [ ] Yes — SNS FIFO topics support all subscriber types
- [ ] Yes — but messages won't be ordered at Lambda
- [x] No — SNS FIFO topics can ONLY deliver to SQS FIFO queues. Lambda and HTTP endpoints are not supported subscribers.
- [ ] No — SNS FIFO topics do not exist

> **Explanation:** SNS FIFO topics guarantee ordered, deduplicated delivery, but they can ONLY fan out to SQS FIFO queues. They cannot deliver to Lambda, HTTP, email, SMS, or any other subscriber type. If Lambda processing is needed, route through SQS FIFO → Lambda.

---

**Question 15:**
A company's CloudWatch alarm triggers when the CPU of a production database exceeds 90%. The operations team needs to receive email notifications AND have a Lambda function automatically scale the database. Which architecture implements this?

- [ ] CloudWatch Alarm → Lambda (Lambda sends email via SES)
- [ ] CloudWatch Alarm → SQS → Lambda + separate email
- [x] CloudWatch Alarm → SNS Topic → (Email Subscription + Lambda Subscription)
- [ ] CloudWatch Alarm → EventBridge → Step Functions

> **Explanation:** CloudWatch Alarms natively integrate with SNS. The SNS topic can have multiple subscribers — one for email notifications and one for a Lambda function that handles the auto-scaling action. This is a classic use of SNS's push-based one-to-many delivery.

---

**Question 16:**
A solutions architect compares SNS and SQS for a new application. Which statement CORRECTLY distinguishes the two services?

- [ ] SNS persists messages for up to 14 days; SQS is fire-and-forget
- [ ] SQS pushes messages to consumers; SNS requires consumers to poll
- [x] SNS is push-based (one-to-many, no persistence); SQS is pull-based (one consumer per message, messages persist until deleted or retained)
- [ ] SNS and SQS both support message replay

> **Explanation:** SNS pushes messages to all subscribers simultaneously (fire-and-forget, no persistence). SQS stores messages that consumers must poll for (pull model), with one consumer per message and retention up to 14 days. Neither supports replay — only Kinesis Data Streams supports replay.

---

## Amazon Kinesis (Q17–Q26)

**Question 17:**
A ride-sharing company needs to ingest GPS coordinates from 50,000 vehicles in real-time, with the ability to replay the last 7 days of location data for route optimization analysis. Which service meets both the real-time ingestion and replay requirements?

- [ ] Amazon SQS Standard Queue with 14-day retention
- [ ] Amazon SNS with SQS fan-out
- [x] Amazon Kinesis Data Streams with 7-day retention and Partition Key set to vehicle ID
- [ ] Amazon Kinesis Data Firehose

> **Explanation:** Kinesis Data Streams is the only service that supports both real-time data ingestion and data replay. Setting retention to 7 days allows consumers to re-read past data. Partition Key = vehicle ID ensures all data for a specific vehicle goes to the same shard, maintaining per-vehicle ordering. SQS deletes messages after consumption (no replay). Firehose has no storage/replay.

---

**Question 18:**
A company's Kinesis Data Stream has 10 shards. They have 5 consumer applications, each needing to read ALL data from the stream with low latency. Using standard consumers, they notice throughput bottlenecks. Why, and how should it be fixed?

- [ ] Add more shards to increase write throughput
- [ ] Switch to Kinesis Data Firehose for better delivery
- [x] Enable Enhanced Fan-Out — standard mode shares 2 MB/s per shard across ALL consumers, but Enhanced Fan-Out provides dedicated 2 MB/s per shard per consumer
- [ ] Reduce to 2 consumer applications

> **Explanation:** In standard mode, all consumers share 2 MB/s of read throughput per shard. With 5 consumers on 10 shards, each consumer effectively gets only 0.4 MB/s per shard (2 MB/s ÷ 5). Enhanced Fan-Out gives each consumer a dedicated 2 MB/s per shard using HTTP/2 push, with ~70ms latency instead of ~200ms.

---

**Question 19:**
A data engineering team needs to load streaming clickstream data into Amazon S3 in Parquet format for analytics. They do not want to manage any infrastructure or write consumer code. Which service should they use?

- [ ] Kinesis Data Streams with KCL consumer writing to S3
- [ ] Amazon SQS with Lambda writing to S3
- [ ] Amazon SNS with S3 as a subscriber
- [x] Kinesis Data Firehose with format conversion enabled (JSON → Parquet)

> **Explanation:** Kinesis Data Firehose is fully managed (serverless, no shards, no consumers to write). It natively supports format conversion from JSON to Parquet or ORC, and delivers directly to S3 (also Redshift, OpenSearch, and third-party destinations). No infrastructure management required.

---

**Question 20:**
A solutions architect is choosing between Kinesis Data Streams and Kinesis Data Firehose. The application requires sub-second processing latency with custom transformation logic. Which is the correct choice, and why?

- [x] Kinesis Data Streams — provides real-time latency (~200ms standard, ~70ms with Enhanced Fan-Out) and supports custom consumers (SDK, KCL, Lambda)
- [ ] Kinesis Data Firehose — it supports Lambda transformations and is real-time
- [ ] Either service works — they have the same latency characteristics
- [ ] Amazon SQS — it provides faster processing than Kinesis

> **Explanation:** Kinesis Data Streams is true real-time (sub-second). Firehose is near real-time with a minimum 60-second buffer interval, making it unsuitable for sub-second requirements. While Firehose supports Lambda transformations, its buffering introduces unacceptable latency for this use case.

---

**Question 21:**
A Kinesis Data Stream uses 4 shards. The producer needs to write 6 MB/s of data. What happens, and how should it be resolved?

- [ ] Kinesis automatically scales to handle the throughput
- [ ] Data is buffered internally until shards can catch up
- [x] Writes are throttled (ProvisionedThroughputExceededException) because 4 shards support only 4 MB/s write. The solution is to increase to at least 6 shards or switch to On-Demand mode.
- [ ] Kinesis drops excess data silently

> **Explanation:** Each shard supports 1 MB/s of write throughput. 4 shards = 4 MB/s maximum. At 6 MB/s, the producer receives ProvisionedThroughputExceededException errors. The fix is to split shards to at least 6, or switch to On-Demand mode which auto-scales based on observed throughput.

---

**Question 22:**
A logistics company ingests truck telemetry into Kinesis Data Streams using the truck ID as the Partition Key. They notice that data for truck "TRUCK-001" is always in order, but data across different trucks arrives in arbitrary order. Is this expected behavior?

- [x] Yes — Kinesis guarantees ordering ONLY within a shard. The Partition Key hash determines the shard, so the same truck ID always goes to the same shard (ordered), but different trucks may go to different shards (no cross-shard ordering).
- [ ] No — Kinesis guarantees global ordering across all shards
- [ ] No — this indicates a configuration error with the Partition Key
- [ ] Yes — but only if Enhanced Fan-Out is enabled

> **Explanation:** Kinesis provides per-shard ordering only. The Partition Key is hashed to determine which shard receives a record. Same Partition Key = same shard = ordered. Different Partition Keys may map to different shards, so there's no ordering guarantee across trucks. This is analogous to SQS FIFO's Message Group ID behavior.

---

**Question 23:**
A Kinesis Data Streams consumer application runs on 6 EC2 instances using the Kinesis Client Library (KCL). The stream has 4 shards. What is the effective consumer distribution?

- [ ] Each EC2 instance processes all 4 shards
- [x] Only 4 instances will be active (one per shard), and 2 instances will remain idle — KCL assigns one shard per instance maximum
- [ ] The 4 shards are split across 6 instances, with some instances sharing shards
- [ ] KCL will throw an error because there are more instances than shards

> **Explanation:** KCL assigns a maximum of one shard per instance. With 4 shards and 6 instances, only 4 instances get a shard assignment, and 2 sit idle. For optimal resource utilization, the number of KCL instances should be less than or equal to the number of shards.

---

**Question 24:**
A company runs Kinesis Data Analytics to detect anomalies in real-time IoT sensor data. They need to join the streaming data with a static lookup table stored in S3 (device metadata). Is this possible with Kinesis Data Analytics?

- [x] Yes — Kinesis Data Analytics supports reference data from S3 that can be joined with streaming data in SQL queries
- [ ] No — Kinesis Data Analytics only works with streaming data
- [ ] Yes — but only if using Apache Flink, not SQL
- [ ] No — you must pre-join the data before sending to Kinesis

> **Explanation:** Kinesis Data Analytics (SQL) supports reference data from S3. This allows you to enrich streaming records by joining them with static lookup tables (e.g., device metadata, user profiles). The reference data is loaded from S3 into the application and refreshed periodically.

---

**Question 25:**
A Kinesis Data Firehose delivery stream is configured to load data into Amazon Redshift. Some records fail to load due to schema mismatches. Where do the failed records end up?

- [ ] They are discarded permanently
- [ ] They are sent back to the source Kinesis Data Stream
- [ ] They trigger a CloudWatch alarm but are lost
- [x] ALL failed records are automatically saved to a configured backup S3 bucket

> **Explanation:** Kinesis Data Firehose always saves failed records to a backup S3 bucket. This applies to all destinations (Redshift, OpenSearch, HTTP endpoints, third-party). For Redshift specifically, Firehose first stages data in S3, then uses COPY command — if COPY fails, the data remains in the S3 staging area and errors are logged.

---

**Question 26:**
A company currently uses Apache Kafka on-premises for real-time event streaming. They want to migrate to AWS with minimal code changes to their Kafka producer and consumer applications. Which AWS service is the best fit?

- [ ] Kinesis Data Streams (it's "AWS's Kafka")
- [ ] Amazon SQS FIFO
- [x] Amazon MSK (Managed Streaming for Apache Kafka) — fully managed Kafka that is API-compatible
- [ ] Amazon MQ with RabbitMQ

> **Explanation:** While Kinesis Data Streams is conceptually similar to Kafka, it uses a completely different API. Amazon MSK (Managed Streaming for Apache Kafka) runs actual Apache Kafka, so existing Kafka producer and consumer code works without changes. If the question says "no code changes" + "Kafka" → MSK.

---

## Amazon MQ (Q27–Q30)

**Question 27:**
A manufacturing company runs an on-premises ActiveMQ broker that uses the MQTT protocol for factory floor IoT devices and the OpenWire protocol for Java-based MES (Manufacturing Execution System) applications. They want to move the broker to AWS with zero code changes. Which service should they use?

- [ ] Amazon SQS + SNS (cloud-native messaging)
- [ ] Amazon Kinesis Data Streams (real-time streaming)
- [x] Amazon MQ — managed ActiveMQ broker supporting MQTT, OpenWire, AMQP, STOMP, and WSS with no code changes required
- [ ] AWS IoT Core for MQTT + SQS for Java apps

> **Explanation:** Amazon MQ is purpose-built for migrating existing on-premises message brokers (ActiveMQ, RabbitMQ) to AWS without code changes. It supports all industry-standard protocols including MQTT, AMQP, STOMP, OpenWire, and WSS. SQS/SNS use AWS-proprietary APIs and would require rewriting all producer/consumer code.

---

**Question 28:**
A solutions architect is designing a new greenfield cloud-native application that requires both message queuing (point-to-point) and publish/subscribe (fan-out) messaging. The team is considering Amazon MQ because it supports both queues and topics natively. Is this the right choice?

- [ ] Yes — Amazon MQ is the best choice because it supports both queues and topics in one service
- [x] No — for new cloud-native applications, use SQS (queuing) + SNS (pub/sub). Amazon MQ is for migrating existing on-premises brokers and does not scale as well as SQS/SNS.
- [ ] Yes — Amazon MQ is serverless and scales better than SQS/SNS
- [ ] No — use Kinesis Data Streams for both use cases

> **Explanation:** Amazon MQ is specifically designed for migration scenarios where existing applications use standard protocols (MQTT, AMQP, etc.) and cannot be rewritten. For new applications, SQS + SNS is always preferred because they are serverless, virtually unlimited in scaling, and have lower operational overhead. Amazon MQ runs on provisioned instances and doesn't scale elastically.

---

**Question 29:**
A company deploys Amazon MQ with ActiveMQ for production workloads. They need high availability with automatic failover across Availability Zones. Which deployment architecture and storage configuration is correct?

- [ ] Single-instance broker with EBS in a Multi-AZ Auto Scaling group
- [x] Active/Standby broker deployment across two AZs with Amazon EFS for shared storage
- [ ] RabbitMQ cluster deployment across three AZs
- [ ] ActiveMQ cluster with EBS replication

> **Explanation:** ActiveMQ on Amazon MQ uses an Active/Standby deployment for high availability. The active broker handles traffic, while the standby broker in another AZ takes over during failover. Both brokers share storage via Amazon EFS (Elastic File System) to ensure message durability across failover. Note: RabbitMQ uses a different HA model — cluster deployment with queue replication across nodes.

---

**Question 30:**
A company evaluates AWS messaging services. Match each scenario to the correct service:

| Scenario | Service |
|---|---|
| A. New serverless app needs to decouple microservices | ? |
| B. Migrate on-premises RabbitMQ broker to AWS | ? |
| C. Real-time clickstream analytics with 7-day replay | ? |
| D. Fan-out notifications to email, SMS, and Lambda | ? |
| E. Load streaming IoT data into S3 in Parquet format | ? |

Which mapping is correct?

- [ ] A=Amazon MQ, B=SQS, C=Kinesis Firehose, D=SNS, E=Kinesis Data Streams
- [ ] A=SQS, B=SNS, C=Kinesis Data Streams, D=Amazon MQ, E=Kinesis Firehose
- [x] A=SQS, B=Amazon MQ, C=Kinesis Data Streams, D=SNS, E=Kinesis Data Firehose
- [ ] A=SNS, B=Amazon MQ, C=SQS FIFO, D=Kinesis Data Streams, E=Kinesis Firehose

> **Explanation:** A → SQS (serverless decoupling/buffering). B → Amazon MQ (migrate existing RabbitMQ — no code changes). C → Kinesis Data Streams (real-time + replay capability). D → SNS (push-based fan-out to multiple subscriber types). E → Kinesis Data Firehose (fully managed delivery to S3 with format conversion).
