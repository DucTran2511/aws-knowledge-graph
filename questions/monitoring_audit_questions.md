---
tags: [questions, monitoring, cloudwatch, cloudtrail, config, audit, compliance]
date: 2026-06-06
---

# Monitoring, Audit & Compliance — Practice Questions

30 scenario-based questions covering [[Amazon CloudWatch]], [[AWS CloudTrail]], and [[AWS Config]].

---

## Amazon CloudWatch (Q1–Q12)

### Q1: EC2 Memory Monitoring
A development team notices their application on EC2 instances is running out of memory, but CloudWatch shows no memory-related metrics. What should they do?

<details>
<summary>Answer</summary>

**Install the CloudWatch Unified Agent** on the EC2 instances and attach an IAM role with `CloudWatchAgentServerPolicy`. Memory utilization is NOT a default EC2 metric — it requires the CloudWatch Agent because AWS can only see hypervisor-level metrics (CPU, network, disk I/O), not OS-level metrics (memory, disk space, swap).
</details>

---

### Q2: Faster Auto Scaling Response
An Auto Scaling Group is responding too slowly to traffic spikes. Metrics are available every 5 minutes. The team needs 1-minute granularity to enable faster scaling. What should they do?

<details>
<summary>Answer</summary>

**Enable Detailed Monitoring** on the EC2 instances. Basic Monitoring collects metrics at 5-minute intervals (default). Detailed Monitoring collects at 1-minute intervals, allowing Auto Scaling policies to react faster.
</details>

---

### Q3: Reduce Alarm Noise
A company has CloudWatch Alarms for CPU utilization, disk I/O, and network traffic on their EC2 fleet. The operations team is overwhelmed with individual alarm notifications. They want to be alerted only when CPU AND disk I/O alarms fire simultaneously. What should they use?

<details>
<summary>Answer</summary>

**Composite Alarms**. Create a composite alarm that combines the CPU alarm AND the disk I/O alarm using AND logic. The composite alarm will only trigger when both child alarms are in the ALARM state, reducing alarm noise.
</details>

---

### Q4: Automated EC2 Recovery
An EC2 instance running a critical application experiences a system status check failure due to underlying hardware issues. The company wants to automatically recover the instance without manual intervention. What should they configure?

<details>
<summary>Answer</summary>

Create a **CloudWatch Alarm** on the `StatusCheckFailed_System` metric with an **EC2 Recovery action**. When the system status check fails, CloudWatch will automatically recover the instance by migrating it to new hardware while preserving its instance ID, private IP, Elastic IP, and all EBS attachments. The instance must use an EBS-backed root volume.
</details>

---

### Q5: Real-Time Log Processing
A company wants to stream application logs from CloudWatch Logs to an Amazon OpenSearch cluster in near real-time for analysis. What is the best approach?

<details>
<summary>Answer</summary>

Create a **Subscription Filter** on the CloudWatch Logs log group that sends data to an **AWS Lambda function**, which then indexes the logs into OpenSearch. Alternatively, use a Subscription Filter to **Amazon Kinesis Data Firehose** with OpenSearch as the destination. Do NOT use S3 export — that is batch-based with up to 12 hours of delay.
</details>

---

### Q6: Stream Metrics to Third-Party
A company uses Datadog for centralized monitoring and wants to send CloudWatch metrics to Datadog in near real-time. They want to avoid polling the CloudWatch API. What should they use?

<details>
<summary>Answer</summary>

**CloudWatch Metric Streams** with **Amazon Data Firehose**. Metric Streams continuously push metrics to Firehose, which can deliver them to third-party partners (Datadog, Splunk, New Relic) in near real-time without API polling.
</details>

---

### Q7: Cross-Region Dashboard
A company has resources deployed in us-east-1, eu-west-1, and ap-southeast-1. They want a single dashboard showing metrics from all three regions. Is this possible?

<details>
<summary>Answer</summary>

Yes. **CloudWatch Dashboards are global** and can display metrics from multiple regions and even multiple AWS accounts in a single view. Create a dashboard and add metric widgets from each region.
</details>

---

### Q8: API Endpoint Monitoring
A company wants to proactively monitor their public-facing REST API for availability, latency, and broken links on a schedule, even when there is no real user traffic. What CloudWatch feature should they use?

<details>
<summary>Answer</summary>

**CloudWatch Synthetics Canaries**. These are configurable scripts (Node.js or Python) that run on a schedule to test endpoint availability, measure latency, and detect broken links. Results integrate with CloudWatch Alarms for alerting.
</details>

---

### Q9: Container Monitoring
A team running microservices on Amazon ECS with Fargate needs to monitor CPU and memory utilization at the task and service level. What should they enable?

<details>
<summary>Answer</summary>

**CloudWatch Container Insights**. This provides metrics at the cluster, service, task, and container level for ECS (and EKS). It collects CPU, memory, disk, and network utilization automatically.
</details>

---

### Q10: Log Metric Alert
An operations team wants to be alerted every time the word "ERROR" appears more than 10 times within 5 minutes in their application logs stored in CloudWatch Logs. What should they configure?

<details>
<summary>Answer</summary>

1. Create a **Metric Filter** on the log group that matches the pattern `"ERROR"` and publishes a custom metric (e.g., `ErrorCount`).
2. Create a **CloudWatch Alarm** on that metric: threshold > 10, period = 5 minutes.
3. Configure the alarm action to send an **SNS notification**.
</details>

---

### Q11: Anomaly-Based Alerting
A company's application has variable traffic patterns — high on weekdays and low on weekends. Static threshold alarms keep triggering false positives on Monday mornings. What should they use instead?

<details>
<summary>Answer</summary>

**CloudWatch Anomaly Detection**. It uses machine learning to create a dynamic baseline that accounts for seasonal patterns and trends. Set up an anomaly detection alarm that triggers only when the metric goes outside the expected band, eliminating false positives from predictable traffic changes.
</details>

---

### Q12: High-Resolution Metrics
A trading application needs to push custom metrics to CloudWatch with 1-second granularity. Is this possible, and how?

<details>
<summary>Answer</summary>

Yes. Use the **PutMetricData API** with `StorageResolution` set to `1` (high-resolution). High-resolution custom metrics support data at 1-second intervals. Note: this costs more than standard 60-second resolution, and high-resolution data is retained for only 3 hours before being aggregated.
</details>

---

## AWS CloudTrail (Q13–Q22)

### Q13: S3 Object Access Audit
A security team needs to audit which IAM users are accessing specific objects in an S3 bucket. CloudTrail is enabled with a multi-region trail, but they see no S3 object-level activity. What's wrong?

<details>
<summary>Answer</summary>

**Data Events are OFF by default**. S3 object-level operations (GetObject, PutObject, DeleteObject) are data plane events and must be explicitly enabled on the trail. The team needs to configure the trail to log **S3 data events** for the specific bucket or all buckets.
</details>

---

### Q14: Tamper-Proof Logs
A compliance officer requires that CloudTrail log files stored in S3 cannot be modified or deleted by anyone, including administrators, for 7 years. What should the architect implement?

<details>
<summary>Answer</summary>

1. Enable **CloudTrail Log File Integrity Validation** (detects tampering with SHA-256 digests).
2. Enable **S3 Object Lock in Compliance mode** with a 7-year retention period (prevents even the root user from deleting).
3. Optionally enable **MFA Delete** on the S3 bucket for additional protection.
</details>

---

### Q15: Root Account Login Alert
A company's security policy requires an immediate alert whenever the AWS root account signs in to the console. What is the most effective approach?

<details>
<summary>Answer</summary>

1. Configure the CloudTrail trail to deliver logs to **CloudWatch Logs**.
2. Create a **Metric Filter** with the pattern: `{ $.userIdentity.type = "Root" && $.eventName = "ConsoleLogin" }`.
3. Create a **CloudWatch Alarm** that triggers on this metric (threshold ≥ 1).
4. Set the alarm action to **SNS** to notify the security team immediately.
</details>

---

### Q16: Multi-Account Audit
An enterprise with 50 AWS accounts in an AWS Organization needs centralized API activity logging for all accounts. What is the recommended approach?

<details>
<summary>Answer</summary>

Create an **Organization Trail** from the **management account**. This automatically logs events from all member accounts across all regions into a single S3 bucket. Member accounts can see the trail but cannot modify or delete it.
</details>

---

### Q17: Real-Time Security Response
A security team wants to automatically revert any changes to security groups that open port 22 (SSH) to the internet (0.0.0.0/0). What architecture should they use?

<details>
<summary>Answer</summary>

CloudTrail captures the `AuthorizeSecurityGroupIngress` API call → **EventBridge rule** matches this event pattern → triggers a **Lambda function** that checks if the rule allows 0.0.0.0/0 on port 22 and automatically revokes it. EventBridge reacts in near real-time (seconds), much faster than S3 log delivery (5-15 min).

*Note: This can also be achieved with **AWS Config** rule (`restricted-ssh`) + auto-remediation, which is the more "Config-native" approach.*
</details>

---

### Q18: CloudTrail Delivery Timing
An incident responder is investigating a security breach that happened 10 minutes ago. They check the CloudTrail S3 bucket but the relevant log files haven't appeared yet. Is this expected?

<details>
<summary>Answer</summary>

Yes, this is **expected behavior**. CloudTrail log delivery to S3 takes approximately **5-15 minutes**. For near-real-time visibility, the team should use **CloudWatch Logs** (if CloudTrail is configured to deliver there) or check **EventBridge** events. For ad-hoc investigation of recent events, they can use the **CloudTrail Event History** in the console (available immediately for management events, retained 90 days).
</details>

---

### Q19: Event History Limitations
A security analyst needs to look up API calls from 6 months ago but the CloudTrail Event History only shows the last 90 days. What should they do?

<details>
<summary>Answer</summary>

Event History only retains the **last 90 days** of management events. For longer retention, the analyst should:
1. Set up a **Trail** that delivers logs to an **S3 bucket** (unlimited retention with lifecycle rules).
2. Use **Amazon Athena** to query the S3 logs using SQL.
3. Alternatively, use **CloudTrail Lake** which supports up to 7 years of retention with built-in SQL queries.
</details>

---

### Q20: Insights Detection
A company notices an unusual spike in `TerminateInstances` API calls over the weekend. They want CloudTrail to automatically detect such anomalies in the future. What should they enable?

<details>
<summary>Answer</summary>

Enable **CloudTrail Insights** on the trail. Insights uses machine learning to establish a baseline of normal API call patterns and automatically detects anomalies like unusual spikes in write management events (e.g., TerminateInstances). When detected, it generates an Insights event that can trigger alerts.
</details>

---

### Q21: Trail vs Event History
What are the key differences between CloudTrail Event History and a Trail?

<details>
<summary>Answer</summary>

| Feature | Event History | Trail |
|---|---|---|
| Setup | None (always on) | Must create |
| Retention | 90 days | Unlimited (S3) |
| Event types | Management only | Management + Data + Insights |
| Delivery | Console only | S3, CloudWatch Logs, EventBridge, SNS |
| Cost | Free | First mgmt copy free, data events extra |
| Multi-account | ❌ | ✅ (Organization Trail) |

</details>

---

### Q22: Lambda Invocation Tracking
A developer wants to see every invocation of a specific Lambda function, including who called it. CloudTrail shows management events for Lambda (CreateFunction, UpdateFunctionConfiguration) but not individual invocations. Why?

<details>
<summary>Answer</summary>

Lambda **Invoke** events are **Data Events**, which are OFF by default due to their high volume. The developer must edit the trail and add **Lambda data events** for the specific function (or all functions) to start capturing invocations.
</details>

---

## AWS Config (Q23–Q30)

### Q23: S3 Encryption Compliance
A company policy requires that all S3 buckets must have server-side encryption enabled. They need a way to continuously check compliance and get notified when a new unencrypted bucket is created. What should they use?

<details>
<summary>Answer</summary>

Create an **AWS Config Rule** using the managed rule `s3-bucket-server-side-encryption-enabled` with a **configuration change trigger**. When any S3 bucket is created or modified, Config evaluates it and marks it COMPLIANT or NON_COMPLIANT. Configure **EventBridge** to match compliance change events and send to **SNS** for notification.
</details>

---

### Q24: Auto-Remediate Open SSH
A security team wants any security group that allows inbound SSH (port 22) from 0.0.0.0/0 to be automatically fixed by removing the offending rule. What should they implement?

<details>
<summary>Answer</summary>

1. Create an **AWS Config Rule** using the managed rule `restricted-ssh` (configuration change trigger).
2. Attach an **automatic remediation action** using an **SSM Automation Document** (e.g., `AWS-DisablePublicAccessForSecurityGroup`).
3. When a security group is flagged NON_COMPLIANT, Config automatically executes the SSM document to revoke the offending ingress rule.
</details>

---

### Q25: Configuration History Investigation
An EC2 instance's security group was changed last week, causing an outage. The team needs to see the exact previous configuration and who made the change. What combination of services should they use?

<details>
<summary>Answer</summary>

- **AWS Config** → view the **Configuration Timeline** of the security group to see exactly what changed (before and after configuration items).
- **AWS CloudTrail** → search for the `AuthorizeSecurityGroupIngress` or `RevokeSecurityGroupIngress` API call to identify **who** made the change (IAM user/role, source IP, timestamp).

Config tells you WHAT changed. CloudTrail tells you WHO changed it.
</details>

---

### Q26: Multi-Account Compliance View
An enterprise with 200 AWS accounts needs a centralized compliance dashboard showing all non-compliant resources across every account and region. What should they set up?

<details>
<summary>Answer</summary>

Create a **Config Aggregator** in a central security account. Authorize it to aggregate data from all accounts in the **AWS Organization** (no individual account authorization needed with Organizations). The aggregator provides a unified, read-only compliance view across all accounts and regions with advanced SQL-like query capabilities.
</details>

---

### Q27: Industry Compliance Framework
A healthcare company needs to deploy a set of compliance rules aligned with HIPAA requirements across all AWS accounts in their organization. They want to manage these rules as a single unit. What should they use?

<details>
<summary>Answer</summary>

Deploy a **Conformance Pack** using the pre-built `Operational-Best-Practices-for-HIPAA-Security` template. Conformance Packs bundle multiple Config rules and remediation actions into a single YAML template that can be deployed across an organization via AWS Organizations. They provide pack-level compliance scores.
</details>

---

### Q28: Preventive vs Detective
A solutions architect is told: "We need to PREVENT users from launching EC2 instances without encryption enabled on their EBS volumes." Is AWS Config the right tool?

<details>
<summary>Answer</summary>

**No.** AWS Config rules are **detective controls** — they evaluate AFTER a change occurs and flag non-compliance. They cannot prevent the action. For **prevention**, use:
- **IAM Policies** with conditions (e.g., deny `ec2:RunInstances` unless encrypted)
- **Service Control Policies (SCPs)** in AWS Organizations
- **AWS Config + auto-remediation** as a fallback to terminate or stop non-compliant instances after launch
</details>

---

### Q29: Custom Compliance Check
A company has a custom requirement: all EC2 instances must use a specific set of approved AMI IDs. There is no single AWS Managed Rule that fits exactly. What should they do?

<details>
<summary>Answer</summary>

Create a **Custom Config Rule** powered by a **Lambda function**. The Lambda evaluates each EC2 instance's AMI ID against the approved list and returns COMPLIANT or NON_COMPLIANT. Set the trigger to **configuration changes** for `AWS::EC2::Instance` resources.

*Note: AWS also offers the managed rule `approved-amis-by-id` which may fit this use case. Always check managed rules first before building custom ones.*
</details>

---

### Q30: Config vs CloudTrail vs CloudWatch
For each scenario, identify the correct service:

1. "CPU utilization on EC2 is spiking"
2. "Someone deleted an S3 bucket — who did it?"
3. "Is our RDS instance configured for Multi-AZ?"
4. "Alert me when API error rate exceeds 1%"
5. "Ensure all EBS volumes are encrypted"
6. "What was the configuration of this VPC yesterday?"

<details>
<summary>Answer</summary>

1. **CloudWatch** — performance metric (CPUUtilization)
2. **CloudTrail** — audit trail (who called DeleteBucket API)
3. **AWS Config** — configuration compliance (evaluate rds-multi-az-support rule)
4. **CloudWatch** — metric alarm on error rate
5. **AWS Config** — Config Rule (encrypted-volumes)
6. **AWS Config** — Configuration Timeline (historical Configuration Items)

**Memory aid:**
- CloudWatch = "What is happening NOW?" (performance)
- CloudTrail = "WHO did WHAT?" (audit)
- AWS Config = "Is it COMPLIANT?" (configuration)
</details>
