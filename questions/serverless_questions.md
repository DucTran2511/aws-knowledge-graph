---
tags: [questions, serverless, lambda, dynamodb, api-gateway, step-functions, cognito, SAA-C03]
date: 2026-05-29
---

# Serverless Practice Questions (SAA-C03)

30 scenario-based questions covering [[AWS Lambda]], [[Amazon DynamoDB]], [[Amazon API Gateway]], [[AWS Step Functions]], and [[Amazon Cognito]].

---

## AWS Lambda (Questions 1–10)

### Q1: Lambda Cold Start — Latency-Sensitive API
A company runs a customer-facing REST API using API Gateway and Lambda (Java runtime). Users report intermittent high-latency responses (~5 seconds) during low-traffic periods. During peak hours, responses are consistently fast (~200ms). What should the solutions architect do to eliminate the latency spikes?

<details><summary>Answer</summary>

**B) Configure Provisioned Concurrency on the Lambda function's published alias.**

Provisioned Concurrency pre-initializes execution environments, eliminating cold starts entirely. Java is notorious for slow cold starts. Set Provisioned Concurrency on a **Version or Alias** (not $LATEST). SnapStart is also an option for Java specifically, but Provisioned Concurrency is the definitive cold-start elimination answer.

**Why not:**
- A) Reserved Concurrency guarantees capacity but does NOT eliminate cold starts.
- C) Increasing memory improves overall performance but doesn't eliminate cold starts.
- D) Moving to a lighter runtime (Python) would reduce cold start time but requires rewriting the application.
</details>

---

### Q2: Lambda + SQS — Duplicate Processing
A company processes orders using SQS Standard Queue → Lambda. The Lambda function takes up to 3 minutes to process each message. Operations reports that some orders are being processed twice. What is the most likely cause and fix?

<details><summary>Answer</summary>

**C) The SQS visibility timeout is too short. Increase it to at least 6× the Lambda timeout (18 minutes).**

When Lambda receives a message from SQS, the message becomes invisible. If Lambda doesn't finish processing before the visibility timeout expires, the message becomes visible again and is processed by another invocation. AWS recommends setting visibility timeout to **6× the Lambda function timeout**.

**Why not:**
- A) Switching to FIFO prevents duplicates from being *sent* but doesn't address visibility timeout issues.
- B) Increasing Lambda memory makes it faster but doesn't guarantee it finishes before visibility timeout.
- D) Adding idempotency checks is a valid mitigation but doesn't fix the root cause.
</details>

---

### Q3: Lambda VPC — Can't Reach Internet
A Lambda function is deployed in a VPC private subnet to access an RDS database. The function successfully queries RDS but fails when trying to call an external third-party API over the internet. What should the solutions architect do?

<details><summary>Answer</summary>

**A) Deploy a NAT Gateway in a public subnet and update the private subnet route table to route 0.0.0.0/0 to the NAT Gateway.**

Lambda in a VPC private subnet has no internet access by default. A NAT Gateway in a public subnet provides outbound internet access for resources in private subnets.

**Why not:**
- B) Moving Lambda to a public subnet doesn't help — Lambda doesn't get a public IP even in public subnets.
- C) An Internet Gateway alone isn't sufficient — Lambda in a private subnet can't use it directly.
- D) VPC Endpoints only work for AWS services, not external third-party APIs.
</details>

---

### Q4: Lambda + RDS — Connection Exhaustion
A serverless application uses Lambda to handle API requests, with each invocation opening a new connection to an Amazon Aurora MySQL database. During traffic spikes, the database reaches its maximum connection limit and new Lambda invocations fail. What is the most operationally efficient solution?

<details><summary>Answer</summary>

**B) Use Amazon RDS Proxy to pool and share database connections across Lambda invocations.**

RDS Proxy sits between Lambda and the database, pooling connections and sharing them across concurrent Lambda invocations. It requires Lambda to be in the same VPC and supports IAM authentication.

**Why not:**
- A) Increasing the RDS instance size raises the connection limit but doesn't solve the fundamental problem of Lambda's stateless, high-concurrency nature creating too many connections.
- C) Reserving Lambda concurrency would cap the number of concurrent connections but also throttles your API.
- D) Caching connections in Lambda's init code (/tmp) helps within warm starts but doesn't help during scale-out with many cold starts.
</details>

---

### Q5: Lambda vs Step Functions — Long Processing
A company needs to process large video files uploaded to S3. Processing involves 5 sequential steps: validation, transcoding, thumbnail generation, metadata extraction, and notification. Total processing time is approximately 45 minutes. What architecture should the solutions architect recommend?

<details><summary>Answer</summary>

**C) Use AWS Step Functions Standard workflow to orchestrate the 5 steps, with each step running on ECS Fargate tasks (or Lambda for short steps).**

Lambda has a 15-minute max timeout, making it unsuitable for a 45-minute pipeline. Step Functions Standard supports workflows up to 1 year. Each step can use the most appropriate compute (Lambda for short tasks, ECS Fargate for long-running transcoding).

**Why not:**
- A) A single Lambda function can't run for 45 minutes (15-min limit).
- B) Chaining Lambda functions via SQS works but lacks visual debugging, built-in retry/catch, and is harder to maintain.
- D) Step Functions Express has a 5-minute max duration.
</details>

---

### Q6: Lambda@Edge vs CloudFront Functions
A company serves a global web application through CloudFront and needs to redirect users to country-specific URLs based on the `CloudFront-Viewer-Country` header. The redirect logic is a simple header check with no external API calls. Which solution provides the lowest latency and cost?

<details><summary>Answer</summary>

**A) Use CloudFront Functions on the Viewer Request event to perform the URL redirect.**

CloudFront Functions run at all 225+ edge locations, execute in less than 1ms, and are 1/6 the cost of Lambda@Edge. Simple header manipulation and URL redirects are ideal use cases.

**Why not:**
- B) Lambda@Edge works but is more expensive and slower (runs at Regional Edge Caches, not all POPs).
- C) Handling redirects in the origin application adds latency (request must travel to origin).
- D) Using Route 53 Geolocation routing solves a different problem (DNS-level routing, not URL redirects).
</details>

---

### Q7: Lambda Concurrency — Protecting Critical Function
A company runs multiple Lambda functions in the same account. A batch-processing function occasionally consumes all available concurrency (1,000), causing the customer-facing API Lambda function to be throttled. How should the architect protect the API function?

<details><summary>Answer</summary>

**B) Set Reserved Concurrency on the API Lambda function to guarantee it always has capacity.**

Reserved Concurrency carves out a dedicated portion of the account's concurrency pool for a specific function. This both guarantees capacity AND prevents other functions from consuming it. It also caps the function at that limit.

**Why not:**
- A) Provisioned Concurrency eliminates cold starts but doesn't protect against throttling from other functions.
- C) Setting Reserved Concurrency on the batch function would cap it, but doesn't guarantee the API function has capacity.
- D) Increasing account concurrency limit helps but doesn't prevent the batch function from consuming it all again.
</details>

---

### Q8: Lambda Event Source Mapping — DLQ Placement
A company uses SQS → Lambda event source mapping. Some messages consistently fail processing. The team wants to inspect failed messages. Where should the Dead-Letter Queue be configured?

<details><summary>Answer</summary>

**A) Configure the DLQ on the SQS source queue (not on the Lambda function).**

For event source mappings, failed messages are returned to the SQS queue. After exceeding `maxReceiveCount`, they are sent to the DLQ configured on the **SQS queue** via the Redrive Policy. Lambda's own DLQ/Destination only applies to asynchronous invocations (S3, SNS, EventBridge).

**Why not:**
- B) Configuring a DLQ on the Lambda function only works for asynchronous invocations, not event source mappings.
- C) Lambda Destinations also only apply to asynchronous invocations.
- D) CloudWatch Logs capture errors but don't preserve the failed message payload.
</details>

---

### Q9: Lambda Layers — Shared Dependencies
A team maintains 15 Lambda functions that all use the same set of Python libraries (boto3 extensions, pandas, numpy). Currently, each function includes these libraries in its deployment package, making deployments slow and packages large. What should the architect recommend?

<details><summary>Answer</summary>

**B) Package the shared libraries as a Lambda Layer and attach it to all 15 functions.**

Lambda Layers allow you to share common libraries across functions. Each function can use up to 5 layers. The total unzipped size (code + layers) must not exceed 250 MB. Layers are versioned and can be updated independently.

**Why not:**
- A) A container image supports larger packages (10 GB) but is overkill for shared libraries.
- C) Duplicating libraries in each function is the current problem — not a solution.
- D) An EFS mount could work but adds VPC complexity and is operationally heavier.
</details>

---

### Q10: Lambda — Async Invocation Retries
An application uses S3 event notifications to trigger a Lambda function for image processing. Occasionally, the Lambda function fails due to a transient dependency. The team notices that failed events are eventually lost. What should be configured to capture failed events?

<details><summary>Answer</summary>

**C) Configure a Lambda Destination for failure events (send to SQS or SNS).**

S3 → Lambda is an **asynchronous** invocation. Lambda automatically retries twice on failure. After all retries fail, the event is discarded unless you configure a **DLQ** or a **Lambda Destination**. Destinations are preferred (more flexible — support SQS, SNS, Lambda, EventBridge).

**Why not:**
- A) SQS DLQ on a queue only applies to event source mapping (SQS → Lambda), not async invocations.
- B) Increasing Lambda timeout helps if the function is timing out, but doesn't capture permanently failed events.
- D) S3 event notifications don't support DLQs directly.
</details>

---

## Amazon DynamoDB (Questions 11–18)

### Q11: DynamoDB — Capacity Mode Selection
A startup is launching a new mobile game. They expect a massive spike of users on launch day, followed by unpredictable usage patterns for the first few months. The game stores player profiles and scores in DynamoDB. Which capacity mode should they use initially?

<details><summary>Answer</summary>

**A) On-Demand capacity mode.**

On-Demand mode instantly accommodates unpredictable traffic without capacity planning. It's ideal for new applications with unknown workload patterns. After usage patterns stabilize, they can switch to Provisioned with Auto Scaling for cost optimization (switchable once every 24 hours).

**Why not:**
- B) Provisioned mode requires estimating RCU/WCU, which is impossible for a new launch with unknown traffic.
- C) Provisioned with Auto Scaling still requires initial capacity estimates and has scaling lag.
- D) DynamoDB Reserved Capacity is for long-term cost savings on predictable workloads — not suitable for a new launch.
</details>

---

### Q12: DynamoDB — DAX vs ElastiCache
An e-commerce application stores product catalog data in DynamoDB. During flash sales, read traffic spikes 100× and the application experiences throttling on hot partition keys. The team needs to reduce DynamoDB read load with minimal code changes. What should the architect recommend?

<details><summary>Answer</summary>

**B) Deploy DynamoDB Accelerator (DAX) in front of the DynamoDB table.**

DAX provides microsecond read latency, uses the same DynamoDB API (minimal code changes — just change the endpoint), and specifically solves the hot-partition read problem by caching frequently accessed items.

**Why not:**
- A) ElastiCache (Redis) works but requires significant application changes (caching logic, different API).
- C) Increasing provisioned RCU helps but is expensive during spikes and doesn't solve hot partition keys.
- D) Adding a GSI doesn't reduce read load — it creates an additional index to query, not a cache.
</details>

---

### Q13: DynamoDB — Global Tables
A multinational company requires a database that supports active-active writes in us-east-1, eu-west-1, and ap-southeast-1 with sub-second replication. They use a NoSQL data model. What should the architect recommend?

<details><summary>Answer</summary>

**A) Use DynamoDB Global Tables with replicas in all three regions.**

Global Tables provide multi-region, multi-active (read AND write in any region) with sub-second replication. Requires DynamoDB Streams enabled.

**Why not:**
- B) Aurora Global Database is active-passive for writes (one writer region, read replicas in others).
- C) DynamoDB with cross-region replication via Lambda is custom/complex — Global Tables does this natively.
- D) Running separate DynamoDB tables with application-level sync is error-prone and not recommended.
</details>

---

### Q14: DynamoDB — RCU Calculation
A DynamoDB table stores items averaging 6 KB. The application requires 100 strongly consistent reads per second. How many RCUs should be provisioned?

<details><summary>Answer</summary>

**C) 200 RCU.**

Calculation: Each strongly consistent read of 6 KB requires `ceil(6 KB / 4 KB) = 2 RCU`. For 100 reads/sec: `100 × 2 = 200 RCU`.

If these were eventually consistent reads: `100 × 2 / 2 = 100 RCU` (half the cost).
</details>

---

### Q15: DynamoDB — GSI vs LSI
A developer has a DynamoDB table with `user_id` as partition key and `order_date` as sort key. A new feature requires querying orders by `product_category`. The table has been in production for 6 months. What should the developer do?

<details><summary>Answer</summary>

**B) Create a Global Secondary Index (GSI) with `product_category` as the partition key.**

GSIs can be added at any time to an existing table. They have their own partition key and optional sort key, enabling queries on non-key attributes.

**Why not:**
- A) A Local Secondary Index (LSI) must be created at table creation time — cannot be added to an existing table.
- C) Scan with a filter works but is extremely inefficient and expensive (reads entire table).
- D) Recreating the table with an LSI is possible but causes downtime and data migration.
</details>

---

### Q16: DynamoDB Streams — Real-Time Processing
An application needs to send an email notification whenever a new user is created in a DynamoDB table. The notification must include the user's name and email. What is the most serverless, event-driven approach?

<details><summary>Answer</summary>

**A) Enable DynamoDB Streams with `NEW_IMAGE` view type, and trigger a Lambda function to send the email via SES.**

Streams capture item-level changes in real-time. `NEW_IMAGE` provides the full item after insertion, which contains the name and email. Lambda processes the stream record and sends the email.

**Why not:**
- B) Polling the table periodically (CloudWatch Events → Lambda → Scan) is not real-time and expensive.
- C) Using application-level code to send emails on write is tightly coupled and doesn't handle retries.
- D) `KEYS_ONLY` view type would only provide the key — not the user's name and email.
</details>

---

### Q17: DynamoDB — TTL for Expiring Data
A session management system stores user sessions in DynamoDB. Sessions should be automatically deleted after 24 hours. The solution must be cost-effective. What should the architect configure?

<details><summary>Answer</summary>

**B) Enable Time-to-Live (TTL) on a `session_expiry` attribute, setting it to the Unix timestamp 24 hours from creation.**

TTL automatically deletes expired items at no additional cost (no WCU consumed). Items may take up to 48 hours to be actually deleted, but they are filtered from query results immediately after expiry.

**Why not:**
- A) A scheduled Lambda function to scan and delete is expensive and operationally complex.
- C) DynamoDB Streams to trigger deletion on insert is overcomplicated.
- D) Setting a short backup retention period doesn't delete items from the live table.
</details>

---

### Q18: DynamoDB — S3 Export for Analytics
A data analytics team needs to run complex SQL queries against data stored in a DynamoDB table with billions of items. They want to avoid impacting production performance. What approach should the architect recommend?

<details><summary>Answer</summary>

**C) Use DynamoDB S3 Export to export the table to S3 in Parquet format, then query with Amazon Athena.**

S3 Export uses Point-in-Time Recovery (PITR) snapshots to export data without consuming any RCU. Exporting to Parquet enables efficient querying with Athena, Redshift Spectrum, or EMR.

**Why not:**
- A) Scanning the table with a Lambda function consumes massive RCU and impacts production.
- B) Creating a read replica isn't a DynamoDB feature (that's RDS).
- D) DAX reduces read latency but doesn't help with complex SQL analytics queries.
</details>

---

## Amazon API Gateway (Questions 19–23)

### Q19: API Gateway — REST vs HTTP API
A company is building a simple serverless API with Lambda backend. The API needs JWT-based authentication but does NOT need caching, WAF, or usage plans. Cost optimization is the top priority. Which API type should they choose?

<details><summary>Answer</summary>

**B) HTTP API.**

HTTP APIs are ~70% cheaper than REST APIs, support JWT authorization natively, and are ideal for simple Lambda/HTTP proxy workloads. REST APIs should be used only when features like caching, WAF, or usage plans are required.
</details>

---

### Q20: API Gateway — AWS Service Integration
A company wants to accept form submissions via a REST API and queue them in SQS for asynchronous processing. They want to minimize cost and operational overhead. What is the best approach?

<details><summary>Answer</summary>

**A) Use API Gateway with AWS Service integration to send messages directly to SQS, without a Lambda function.**

API Gateway can integrate directly with SQS (and other AWS services) using AWS Service integration. This eliminates the Lambda function entirely, reducing cost and latency.

**Why not:**
- B) API Gateway → Lambda → SQS works but adds unnecessary cost and latency (Lambda invocation).
- C) Using EC2 to receive submissions is not serverless and increases operational overhead.
- D) API Gateway → Kinesis is for streaming data, not simple message queuing.
</details>

---

### Q21: API Gateway — Throttling
A company exposes a public REST API through API Gateway. A single client is sending excessive requests, causing 429 errors for other clients. How should the architect limit per-client request rates?

<details><summary>Answer</summary>

**C) Create a Usage Plan with throttle limits and issue API Keys to each client.**

Usage Plans allow per-API-key throttling (requests/sec, burst, and monthly quota). Each client gets a unique API Key associated with a Usage Plan that defines their limits.

**Why not:**
- A) WAF rate-based rules can limit by IP but don't provide per-client API key-based throttling.
- B) Increasing the account-level throttle doesn't solve per-client abuse.
- D) Lambda-based rate limiting is operationally complex and defeats the purpose of API Gateway's built-in throttling.
</details>

---

### Q22: API Gateway — Private API
A company needs to expose an internal microservice API that should only be accessible from within their VPC. The API should not be accessible from the public internet. What should the architect configure?

<details><summary>Answer</summary>

**A) Create a Private REST API endpoint and access it via a VPC Interface Endpoint (AWS PrivateLink).**

Private endpoints are accessible only from within the VPC through an interface VPC endpoint. Combined with resource policies, you can restrict access to specific VPCs or VPC endpoints.

**Why not:**
- B) Regional endpoint with an IAM policy restricting by IP is still publicly accessible (just restricted).
- C) An internal ALB works but doesn't provide API Gateway features (throttling, caching, auth).
- D) Edge-Optimized with WAF IP restrictions is still a public endpoint.
</details>

---

### Q23: API Gateway — 29-Second Timeout
A company's API Gateway → Lambda API calls a slow third-party service that sometimes takes 60 seconds to respond. Requests are failing with 504 Gateway Timeout errors. What architecture change should the architect recommend?

<details><summary>Answer</summary>

**B) Change to an asynchronous pattern: API Gateway → SQS → Lambda. Return 202 Accepted immediately and process in the background.**

API Gateway has a hard 29-second integration timeout limit that cannot be increased. For operations longer than 29 seconds, use an async pattern with SQS or Step Functions. The client can poll for results or use WebSockets for notification.

**Why not:**
- A) Increasing Lambda timeout beyond 29 seconds won't help — the bottleneck is API Gateway's 29-second limit.
- C) Increasing API Gateway timeout is not possible — 29 seconds is a hard limit.
- D) Using CloudFront in front of API Gateway doesn't extend the timeout.
</details>

---

## AWS Step Functions (Questions 24–27)

### Q24: Step Functions — Standard vs Express
A company processes IoT sensor data from 500,000 devices. Each message requires a 3-step processing workflow that takes approximately 2 seconds total. The system must handle 50,000 messages per second during peak periods. Which Step Functions workflow type should they use?

<details><summary>Answer</summary>

**B) Express Workflow.**

Express Workflows support 100,000 executions/sec start rate, 5-minute max duration, and are priced per execution (cheaper for high-volume, short-duration workloads). The 2-second processing time is well within the 5-minute Express limit.

**Why not:**
- A) Standard Workflows are limited to 2,000 starts/sec and are priced per state transition — too expensive at this volume.
- C) Lambda without Step Functions doesn't provide workflow orchestration (retry, branching, visual debugging).
- D) SQS + Lambda works for simple queuing but lacks Step Functions' built-in orchestration.
</details>

---

### Q25: Step Functions — Error Handling
A Step Functions workflow processes insurance claims. The third step (fraud detection via Lambda) occasionally fails due to a third-party API timeout. The company wants to retry this step 3 times with exponential backoff, and if it still fails, route to a manual review queue. How should this be configured?

<details><summary>Answer</summary>

**A) Configure Retry with MaxAttempts=3 and BackoffRate=2.0 on the fraud detection Task state, and add a Catch block that routes to a "ManualReview" state on States.ALL errors.**

Step Functions' built-in Retry and Catch provide declarative error handling without custom code. Retry handles transient failures with exponential backoff. Catch provides a fallback path when retries are exhausted.
</details>

---

### Q26: Step Functions — Human Approval
A company's deployment pipeline requires a manager to approve a deployment before it proceeds to production. The approval should happen via email, and the workflow should wait up to 7 days for a response. What Step Functions pattern should be used?

<details><summary>Answer</summary>

**B) Use a Task state with the `.waitForTaskToken` integration pattern. Send the task token via SNS email and wait for the manager to call `SendTaskSuccess` with the token.**

The waitForTaskToken pattern pauses the workflow until an external system calls back with the task token. Standard Workflows can wait up to 1 year, easily accommodating the 7-day approval window.

**Why not:**
- A) A Wait state with a fixed timer doesn't support dynamic approval — it always waits the full duration.
- C) Polling for approval via a Choice → Wait loop is operationally complex and wastes state transitions.
- D) Express Workflows max out at 5 minutes — can't wait 7 days.
</details>

---

### Q27: Step Functions — Distributed Map
A company needs to process 2 million files stored in an S3 bucket. Each file requires a Lambda function to parse and transform the data. The processing must complete within 4 hours. What Step Functions feature should the architect use?

<details><summary>Answer</summary>

**A) Use a Distributed Map state that reads the S3 inventory and processes items with up to 10,000 concurrent Lambda invocations.**

Distributed Map can iterate over millions of items from S3 with up to 10,000 concurrent child executions. It's designed for large-scale parallel processing.

**Why not:**
- B) Inline Map state supports only 40 concurrent iterations — too slow for 2 million files.
- C) Parallel state requires defining each branch statically — impractical for 2 million items.
- D) SQS → Lambda fan-out works but requires custom orchestration and lacks visual monitoring.
</details>

---

## Amazon Cognito (Questions 28–30)

### Q28: Cognito — User Pools vs Identity Pools
A mobile application needs to let users sign in with their Google account and then upload photos directly to their own private folder in S3. Which Cognito components are needed?

<details><summary>Answer</summary>

**C) Both Cognito User Pools (for Google sign-in federation) AND Identity Pools (for temporary AWS credentials to access S3).**

User Pools handle the authentication (Google federation → JWT tokens). Identity Pools exchange the JWT for temporary AWS credentials with an IAM policy that restricts S3 access to the user's own folder using `${cognito-identity.amazonaws.com:sub}`.

**Why not:**
- A) User Pools alone provide JWT tokens but not AWS credentials — can't access S3 directly.
- B) Identity Pools alone can accept Google tokens but it's better practice to use User Pools as the identity broker.
- D) IAM users for each mobile user is not scalable and not appropriate for end users.
</details>

---

### Q29: Cognito — API Gateway Auth
A company is building a REST API with API Gateway and Lambda backend. The API serves a web application where users sign in with email and password. The company wants the simplest, lowest-maintenance authentication solution. What should the architect recommend?

<details><summary>Answer</summary>

**A) Create a Cognito User Pool and configure a Cognito Authorizer on the API Gateway.**

Cognito User Pool Authorizer is fully managed — it validates JWT tokens automatically with no custom code. The web app authenticates users via the Cognito Hosted UI or SDK, receives JWT tokens, and includes them in API requests.

**Why not:**
- B) Lambda Authorizer requires writing and maintaining custom auth logic.
- C) IAM authentication with SigV4 is complex for web application users.
- D) API Keys are not an authentication mechanism — they're for usage tracking.
</details>

---

### Q30: Cognito — Enterprise Federation
A large enterprise wants to allow its 10,000 employees to access a web application using their existing corporate Active Directory credentials. They don't want employees to create new accounts. What should the architect configure?

<details><summary>Answer</summary>

**B) Configure a Cognito User Pool with SAML 2.0 federation to the corporate Active Directory (via AD FS or Azure AD as the SAML IdP).**

Cognito User Pools support SAML 2.0 federation, allowing employees to sign in with their corporate credentials via their existing Identity Provider. No new accounts needed — the corporate IdP handles authentication.

**Why not:**
- A) Importing all AD users into Cognito User Pool defeats the purpose of federation and creates a sync problem.
- C) Cognito Identity Pools alone provide AWS credentials but don't offer the full user management and hosted UI experience.
- D) Building a custom SAML integration without Cognito is operationally complex and reinvents the wheel.
</details>
