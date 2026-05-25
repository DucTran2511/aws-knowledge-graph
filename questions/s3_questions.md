# Amazon S3 Questions

**Question 1:**
What is the maximum size of a single object that can be uploaded to Amazon S3?

- [ ] 5 GB
- [ ] 10 GB
- [x] 5 TB
- [ ] Unlimited

**Question 2:**
You need to upload a 7 GB file to S3. Which upload method is required?

- [ ] Single PUT request
- [x] Multi-Part Upload
- [ ] S3 Transfer Acceleration
- [ ] AWS Snowball

**Question 3:**
S3 bucket names must be globally unique. At which level are buckets actually created?

- [ ] Global level (no region)
- [x] Region level
- [ ] Availability Zone level
- [ ] Account level

**Question 4:**
A developer uploads an object to an S3 bucket without specifying any encryption. What encryption is applied to the object?

- [ ] No encryption — objects are unencrypted by default
- [x] SSE-S3 (AES-256) — default encryption since January 2023
- [ ] SSE-KMS with the AWS-managed key
- [ ] Client-side encryption

**Question 5:**
Your security team requires that all S3 encryption key usage is logged in AWS CloudTrail for auditing purposes. Which encryption method should you use?

- [ ] SSE-S3
- [x] SSE-KMS
- [ ] SSE-C
- [ ] Client-Side Encryption

**Question 6:**
You are using SSE-KMS encryption on a heavily accessed S3 bucket and your application is experiencing throttling errors. What is the most cost-effective solution?

- [ ] Switch to SSE-S3 encryption
- [ ] Request a KMS quota increase
- [ ] Enable S3 Bucket Keys to reduce KMS API calls
- [ ] Use SSE-C instead

**Question 7:**
Which S3 encryption method requires you to send the encryption key with every request and mandates HTTPS?

- [ ] SSE-S3
- [ ] SSE-KMS
- [x] SSE-C
- [ ] Client-Side Encryption

**Question 8:**
You enable versioning on an S3 bucket and then delete an object without specifying a version ID. What happens?

- [ ] The object is permanently deleted
- [ ] The latest version is permanently deleted
- [x] A Delete Marker is added; the object appears deleted but all versions remain
- [ ] All versions of the object are deleted

**Question 9:**
After enabling versioning on an S3 bucket, you decide to stop versioning. What happens to existing object versions when you suspend versioning?

- [ ] All existing versions are permanently deleted
- [ ] Existing versions are merged into a single version
- [x] Existing versions remain and continue to consume storage; new versions are no longer created
- [ ] Existing versions are moved to Glacier automatically

**Question 10:**
You need to replicate objects from an S3 bucket in us-east-1 to a bucket in eu-west-1 for disaster recovery. Which feature should you use and what is a prerequisite?

- [ ] S3 Transfer Acceleration; no prerequisites
- [ ] S3 Cross-Region Replication (CRR); versioning must be enabled on both buckets
- [ ] S3 Same-Region Replication (SRR); versioning must be enabled on source only
- [ ] S3 Batch Operations; no prerequisites

**Question 11:**
After enabling S3 Cross-Region Replication on a bucket that already contains 10,000 objects, what happens to the existing objects?

- [ ] All existing objects are replicated immediately
- [ ] Existing objects are replicated within 24 hours
- [ ] Only new objects are replicated; use S3 Batch Replication for existing objects
- [ ] Existing objects are replicated but with lower priority

**Question 12:**
You enable CRR with delete marker replication disabled (the default). A user deletes an object in the source bucket. What happens in the destination bucket?

- [ ] The object is also deleted in the destination
- [ ] A delete marker is created in the destination
- [ ] The object remains intact in the destination — delete markers are not replicated by default
- [ ] The replication rule fails and generates an error

**Question 13:**
Which S3 storage class should you use when the access pattern is unpredictable and you want to avoid retrieval fees?

- [ ] S3 Standard
- [ ] S3 Intelligent-Tiering
- [ ] S3 Standard-IA
- [ ] S3 One Zone-IA

**Question 14:**
A company stores data that is accessed frequently for the first 30 days, infrequently for the next 60 days, and then rarely for compliance retention for 7 years. Which lifecycle configuration is most cost-effective?

- [ ] Store all data in S3 Standard permanently
- [ ] Store in Glacier Deep Archive from day 1
- [ ] S3 Standard → Transition to Standard-IA at 30 days → Transition to Glacier Flexible at 90 days → Expire after 7 years
- [ ] S3 Intelligent-Tiering for the entire duration

**Question 15:**
Which S3 storage class stores data in only one Availability Zone and is approximately 20% cheaper than Standard-IA?

- [ ] S3 Standard
- [ ] S3 Glacier Instant Retrieval
- [ ] S3 One Zone-IA
- [ ] S3 Reduced Redundancy (deprecated)

**Question 16:**
A hospital needs to archive patient records that must be retrievable within milliseconds but are accessed only once per quarter. Which storage class is the best fit?

- [ ] S3 Standard
- [ ] S3 Standard-IA
- [ ] S3 Glacier Instant Retrieval
- [ ] S3 Glacier Flexible Retrieval

**Question 17:**
What is the retrieval time for S3 Glacier Deep Archive using the Standard retrieval tier?

- [ ] 1–5 minutes
- [ ] 3–5 hours
- [ ] 12 hours
- [ ] 48 hours

**Question 18:**
You want to host a static website on S3 with a custom domain and HTTPS. Which additional AWS service is required for HTTPS?

- [ ] An Application Load Balancer
- [ ] AWS WAF
- [ ] Amazon CloudFront with an ACM SSL certificate
- [ ] Amazon API Gateway

**Question 19:**
A user reports that their S3-hosted static website cannot load fonts from another S3 bucket. What is the most likely issue?

- [ ] The fonts bucket has encryption enabled
- [ ] The website bucket needs versioning enabled
- [ ] CORS (Cross-Origin Resource Sharing) is not configured on the fonts bucket
- [ ] The fonts are too large for S3

**Question 20:**
You need to give a third-party vendor temporary access to download a specific private object from your S3 bucket without creating an IAM user for them. What should you use?

- [ ] Make the bucket public temporarily
- [ ] Create a bucket policy allowing their IP address
- [ ] Generate a pre-signed URL with an expiration time
- [ ] Use S3 Access Points

**Question 21:**
What is the maximum expiration time for a pre-signed URL generated using the AWS CLI?

- [ ] 720 minutes (12 hours)
- [ ] 24 hours
- [ ] 168 hours (7 days)
- [ ] 30 days

**Question 22:**
Your S3 application needs to achieve more than 3,500 PUT requests per second. How can you increase throughput?

- [ ] Request a service limit increase from AWS
- [ ] Distribute objects across multiple prefixes (each prefix gets 3,500 PUT/s and 5,500 GET/s)
- [ ] Enable S3 Transfer Acceleration
- [ ] Switch to S3 One Zone-IA for faster writes

**Question 23:**
A company needs to accelerate file uploads from users in Asia to an S3 bucket in us-east-1. Which feature should they enable?

- [ ] S3 Cross-Region Replication
- [ ] CloudFront with a custom origin
- [ ] S3 Transfer Acceleration
- [ ] AWS Global Accelerator

**Question 24:**
You want to run SQL queries on CSV files stored in S3 without downloading them first. Which S3 feature enables server-side filtering to reduce data transfer?

- [ ] S3 Batch Operations
- [ ] S3 Access Points
- [ ] S3 Select
- [ ] S3 Inventory

**Question 25:**
A financial services company must ensure that certain S3 objects cannot be deleted or overwritten by anyone — including the AWS root user — for a retention period of 5 years. Which feature should they use?

- [ ] S3 Object Lock — Governance Mode
- [ ] S3 Object Lock — Compliance Mode
- [ ] S3 MFA Delete
- [ ] S3 Versioning with lifecycle rules

**Question 26:**
What is the difference between S3 Object Lock Governance Mode and Compliance Mode?

- [ ] Governance Mode protects against accidental deletes; Compliance Mode encrypts data
- [ ] Governance Mode allows users with special IAM permissions to override the lock; Compliance Mode prevents anyone (including root) from deleting during the retention period
- [ ] Governance Mode is for single objects; Compliance Mode is for entire buckets
- [ ] There is no difference — they are aliases for the same feature

**Question 27:**
You have enabled S3 access logging to monitor requests to your application bucket. You set the logging destination to the same bucket. What will happen?

- [ ] Logging works normally
- [ ] S3 returns an error and prevents this configuration
- [ ] It creates an infinite logging loop that grows the bucket size exponentially
- [ ] Only the first level of logs is captured

**Question 28:**
A data lake team has a single S3 bucket shared by multiple departments (Finance, Analytics, and Sales). Each department should only access their own prefix. What is the simplest way to manage access?

- [ ] Create a separate bucket for each department
- [ ] Write one complex bucket policy with conditions for each department
- [ ] Create S3 Access Points, each scoped to a department's prefix with its own access policy
- [ ] Use S3 Object Lambda to filter access per department

**Question 29:**
Your company uses S3 to store data and wants to enforce HTTPS-only access. How should you implement this?

- [ ] Disable the HTTP endpoint in the S3 console
- [ ] Enable SSE-S3 encryption, which automatically enforces HTTPS
- [ ] Add a bucket policy with a Deny statement conditioned on `aws:SecureTransport: false`
- [ ] Enable S3 Block Public Access

**Question 30:**
An application uploads images to S3 and needs to automatically generate thumbnails upon upload. Which architecture is most appropriate?

- [ ] Use a cron job on EC2 to poll S3 for new objects
- [ ] Enable S3 Intelligent-Tiering to process new objects
- [ ] Configure an S3 Event Notification to trigger a Lambda function on `s3:ObjectCreated:*`
- [ ] Use S3 Batch Operations to process uploads
