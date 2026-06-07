# AWS Containers — Practice Questions

> Topics: Amazon ECS, AWS Fargate, Amazon ECR, Amazon EKS
> 30 scenario-based questions for SAA-C03 preparation

---

## Amazon ECS Core Concepts (Q1–Q10)

**Question 1:**
A company wants to run Docker containers on AWS without managing the underlying infrastructure. They need automatic scaling, no OS patching, and a pay-per-use pricing model. They have no Kubernetes expertise. Which combination of services should they use?

- [ ] Amazon EKS with Managed Node Groups
- [ ] Amazon ECS with EC2 Launch Type and Auto Scaling Group
- [x] Amazon ECS with Fargate Launch Type
- [ ] EC2 instances with Docker Engine installed manually

> **Explanation:** ECS with Fargate is the ideal choice — it's AWS-native (no Kubernetes knowledge needed), fully serverless (no EC2 instances to manage, no OS patching), and you pay only for the vCPU and memory your tasks consume per second. EKS would be overkill without Kubernetes requirements, and EC2 Launch Type requires managing the underlying instances.

---

**Question 2:**
A solutions architect is designing an ECS architecture. The application has three microservices: an API gateway, an order processor, and a notification service. Each microservice needs different IAM permissions — the API reads from DynamoDB, the order processor writes to S3, and the notification service publishes to SNS. How should IAM permissions be configured?

- [ ] Attach all permissions to the EC2 Instance Profile so all containers inherit them
- [ ] Create a single ECS Task Role with all three sets of permissions
- [x] Define a separate ECS Task Role for each Task Definition, granting only the permissions that specific microservice needs
- [ ] Use IAM users with access keys embedded as environment variables in each container

> **Explanation:** Each ECS Task Definition can be assigned its own **Task Role**, which follows the principle of least privilege. The API gateway task gets DynamoDB read permissions, the order processor gets S3 write permissions, and the notification service gets SNS publish permissions. This is fundamentally different from the EC2 Instance Profile (shared by all containers on the host) and the Task Execution Role (infrastructure-level permissions).

---

**Question 3:**
An operations team is confused about the difference between the ECS Task Role and the ECS Task Execution Role. Which statement correctly distinguishes the two?

- [ ] The Task Role is used by the ECS Agent; the Task Execution Role is used by the application
- [ ] Both roles serve the same purpose and are interchangeable
- [ ] The Task Execution Role grants application-level permissions like accessing S3 and DynamoDB
- [x] The Task Role grants permissions to the application running inside the container (e.g., S3, DynamoDB). The Task Execution Role grants permissions to ECS/Fargate to pull container images from ECR and write logs to CloudWatch.

> **Explanation:** **Task Role** = what your application code can do (access S3, DynamoDB, SNS, etc.). **Task Execution Role** = what the ECS infrastructure needs to launch your task (pull images from ECR, push logs to CloudWatch, retrieve secrets from SSM/Secrets Manager). Both are defined per Task Definition but serve completely different purposes.

---

**Question 4:**
A company runs an ECS cluster with the EC2 Launch Type. They deploy a web application service with a desired count of 4 tasks. During a deployment, they want zero downtime with the ability to run up to 8 tasks simultaneously during the transition. What deployment configuration should they set?

- [ ] minimumHealthyPercent = 100%, maximumPercent = 100%
- [ ] minimumHealthyPercent = 0%, maximumPercent = 200%
- [x] minimumHealthyPercent = 50%, maximumPercent = 200%
- [ ] minimumHealthyPercent = 100%, maximumPercent = 150%

> **Explanation:** With `maximumPercent = 200%`, ECS can run up to 8 tasks (200% of 4) during the update — launching new v2 tasks alongside existing v1 tasks. `minimumHealthyPercent = 50%` means at least 2 tasks must always be running. This allows ECS to perform a rolling update: start new tasks → verify health → drain old tasks → zero downtime.

---

**Question 5:**
An ECS service runs 10 tasks on Fargate. The team wants to auto-scale the number of tasks based on incoming traffic from an Application Load Balancer. Which metric and scaling mechanism should they use?

- [ ] CPUUtilization of the Fargate instances → EC2 Auto Scaling Group
- [x] ALBRequestCountPerTarget metric → ECS Service Auto Scaling with Target Tracking policy
- [ ] ApproximateNumberOfMessagesVisible → CloudWatch Alarm → Lambda to update desired count
- [ ] NetworkIn metric → Step Scaling policy on the ASG

> **Explanation:** **ECS Service Auto Scaling** scales the number of tasks (not EC2 instances — there are none with Fargate). The `ALBRequestCountPerTarget` metric is ideal for scaling based on HTTP request load. Target Tracking automatically adjusts the desired task count to maintain the specified requests-per-target value. With Fargate, there is NO infrastructure layer to scale — only tasks.

---

**Question 6:**
A company uses ECS with the EC2 Launch Type. They have 10 EC2 instances in an Auto Scaling Group, but ECS reports insufficient capacity to place new tasks. What is the recommended AWS feature to automatically manage EC2 capacity based on ECS task placement needs?

- [ ] ECS Service Auto Scaling with CPU Target Tracking
- [ ] CloudWatch Alarm on MemoryReservation → ASG scaling policy
- [x] ECS Cluster Capacity Provider — it automatically scales the underlying ASG based on pending task placement demands
- [ ] AWS Lambda function triggered by ECS events to manually resize the ASG

> **Explanation:** **ECS Cluster Capacity Provider** is the recommended approach for managing EC2 capacity in ECS. It monitors pending tasks that cannot be placed due to insufficient resources and automatically scales the linked ASG up or down. This replaces the older pattern of CloudWatch alarm → ASG scaling, providing tighter integration with ECS task placement.

---

**Question 7:**
A containerized application running on ECS Fargate needs to share persistent data across multiple tasks in different Availability Zones. The data must survive task restarts and be accessible to all tasks simultaneously. Which storage option should the architect use?

- [ ] Amazon EBS volumes mounted to each Fargate task
- [ ] S3 mounted as a file system using s3fs
- [x] Amazon EFS (Elastic File System) mounted as a volume in the Task Definition
- [ ] Docker bind mounts with a shared host directory

> **Explanation:** **EFS is the ONLY persistent shared storage option for Fargate tasks.** EBS volumes cannot be mounted to Fargate tasks. S3 is object storage, not a file system (s3fs is not supported in Fargate). Bind mounts only share data between containers within the same task, not across tasks. EFS is multi-AZ by design and works with both EC2 and Fargate launch types.

---

**Question 8:**
A company deploys multiple ECS services behind an Application Load Balancer. They need to route `/api/*` requests to the API service and `/web/*` requests to the frontend service. Both services may run multiple tasks on the same EC2 instances. How does this work?

- [ ] Each service must run on separate EC2 instances with different ports
- [ ] Use an NLB with TCP-based routing rules
- [x] ALB path-based routing directs traffic to different Target Groups, and ALB supports dynamic port mapping so multiple tasks can run on the same EC2 instance
- [ ] Deploy two separate ALBs, one per service

> **Explanation:** ALB supports **path-based routing** (route by URL path) and **dynamic port mapping** (each task gets a random host port registered in the Target Group). This allows multiple tasks from different services to coexist on the same EC2 instance. ECS automatically registers/deregisters tasks with the ALB Target Group. This is why ALB is the recommended load balancer for ECS.

---

**Question 9:**
A development team wants to inject database credentials into their ECS containers at runtime without hardcoding them in the Docker image or environment variables in the Task Definition. What is the AWS-recommended approach?

- [ ] Store credentials in an S3 file and download them at container startup
- [ ] Use EC2 Instance Metadata to retrieve credentials
- [x] Store credentials in AWS Secrets Manager or SSM Parameter Store and reference them in the Task Definition using the `secrets` field — ECS injects them at task launch via the Task Execution Role
- [ ] Embed credentials in a custom AMI used by the EC2 instances

> **Explanation:** ECS natively integrates with **Secrets Manager** and **SSM Parameter Store**. The Task Definition's `secrets` section references the secret ARN, and ECS injects the value as an environment variable when the task starts. The **Task Execution Role** must have permission to read the secret. This keeps secrets out of the image and Task Definition plaintext.

---

**Question 10:**
A company runs ECS tasks and wants centralized logging. Each container should automatically send its stdout/stderr to CloudWatch Logs without installing a logging agent inside the container. How should this be configured?

- [ ] Install the CloudWatch Agent in each container image
- [ ] Mount an EFS volume and write logs to a shared file
- [x] Configure the `awslogs` log driver in the Task Definition — ECS automatically sends container stdout/stderr to CloudWatch Logs using the Task Execution Role
- [ ] Deploy a sidecar Fluentd container in each task

> **Explanation:** The `awslogs` log driver is built into ECS and sends container stdout/stderr directly to CloudWatch Logs without any additional agent installation. You configure the log group, region, and stream prefix in the Task Definition. The **Task Execution Role** must have `logs:CreateLogStream` and `logs:PutLogEvents` permissions.

---

## EC2 vs Fargate Launch Types (Q11–Q16)

**Question 11:**
A machine learning team needs to run GPU-accelerated inference containers on ECS. They require access to NVIDIA GPUs and need to install custom CUDA drivers on the host. Which ECS launch type must they use?

- [x] EC2 Launch Type — Fargate does not support GPU instances, custom drivers, or SSH access to the host OS
- [ ] Fargate Launch Type with GPU task definition parameters
- [ ] Either launch type — GPU support is configured at the Task Definition level
- [ ] EKS with Fargate profile configured for GPU workloads

> **Explanation:** **Fargate does not support GPU workloads.** GPU instances (p3, p4, g4, g5 families), custom AMIs, and host-level access (SSH, custom drivers) require the **EC2 Launch Type**. You provision GPU-enabled EC2 instances in your cluster and configure the Task Definition with GPU resource requirements.

---

**Question 12:**
A startup with a small DevOps team runs a variable workload — containers are idle most of the day but spike during business hours. They want to minimize operational overhead and avoid paying for idle infrastructure. Which launch type best fits their needs?

- [ ] EC2 Launch Type with Reserved Instances for baseline, Spot Instances for spikes
- [ ] EC2 Launch Type with an aggressively scaled-down ASG
- [x] Fargate Launch Type — pay per task (vCPU + memory per second) with no idle infrastructure costs, and zero server management
- [ ] Lambda functions instead of containers

> **Explanation:** Fargate charges only for the vCPU and memory consumed while tasks are running. When no tasks run, the cost is zero. This is ideal for variable workloads and small teams that don't want to manage, patch, or right-size EC2 instances. EC2 Launch Type would incur costs for idle instances during off-peak hours.

---

**Question 13:**
A regulated financial institution requires full OS-level audit logging, custom security agents installed on every container host, and the ability to SSH into hosts for compliance investigations. They want to run containers on ECS. Which launch type meets these requirements?

- [ ] Fargate with enhanced monitoring enabled
- [x] EC2 Launch Type — provides SSH access, custom AMIs, and the ability to install host-level security agents and audit tooling
- [ ] Either launch type with CloudWatch Container Insights enabled
- [ ] Fargate with ECS Exec enabled

> **Explanation:** Compliance scenarios requiring **OS-level access, custom security agents, and SSH** mandate the **EC2 Launch Type**. Fargate provides no access to the underlying host OS. While ECS Exec allows interactive command execution inside a running container, it does not provide host-level OS access or the ability to install host agents.

---

**Question 14:**
An ECS cluster uses the EC2 Launch Type with 5 EC2 instances. The team wants to scale both the tasks AND the underlying EC2 instances automatically. How many scaling layers are involved, and what controls each?

- [ ] One layer — ECS Service Auto Scaling handles both tasks and EC2 instances
- [ ] One layer — the ASG scales instances, and tasks fill automatically
- [x] Two layers — **Layer 1:** ECS Service Auto Scaling scales the number of tasks. **Layer 2:** ECS Cluster Capacity Provider (or ASG) scales the number of EC2 instances based on task placement needs.
- [ ] Two layers — CloudWatch scales tasks, and Lambda scales instances

> **Explanation:** EC2 Launch Type requires **two-layer scaling**: (1) **ECS Service Auto Scaling** adjusts the desired task count based on metrics like CPU, memory, or ALB requests. (2) **Cluster Capacity Provider** (or ASG policies) adjusts the EC2 fleet to ensure enough capacity for task placement. With **Fargate, only Layer 1 exists** — there's no infrastructure to scale.

---

**Question 15:**
A company wants to migrate from EC2 Launch Type to Fargate. Their tasks currently use EBS volumes for temporary storage and bind mounts to share data between containers in the same task. Which storage capabilities will they lose after migration?

- [ ] No storage capabilities are lost — Fargate supports all the same storage options
- [ ] They will lose EFS support
- [x] They will lose EBS volume mounts (Fargate doesn't support EBS). Bind mounts between containers in the same task are still supported via ephemeral storage. EFS mounts are also supported on Fargate.
- [ ] They will lose all persistent storage options on Fargate

> **Explanation:** Fargate supports **ephemeral storage** (shared between containers in the same task, up to 200 GB) and **EFS mounts** (persistent, shared across tasks). However, **EBS volumes cannot be mounted on Fargate tasks.** If the application requires EBS-level block storage, it must stay on EC2 Launch Type.

---

**Question 16:**
A solutions architect is comparing the pricing model of EC2 Launch Type vs Fargate for a workload that runs 100 containers 24/7 with predictable, steady resource consumption. Which is likely more cost-effective?

- [x] EC2 Launch Type with Reserved Instances — for predictable, always-on workloads, reserved EC2 pricing can be significantly cheaper than Fargate's per-task-per-second billing
- [ ] Fargate — it's always cheaper because you don't pay for idle capacity
- [ ] Both cost the same — AWS normalizes pricing across launch types
- [ ] EKS with Spot Instances

> **Explanation:** For **steady, predictable, 24/7 workloads**, EC2 with Reserved Instances (1-year or 3-year) typically provides significant cost savings (up to 40-70%) compared to Fargate's on-demand per-second pricing. Fargate excels for **variable, bursty, or infrequent workloads** where you'd otherwise pay for idle EC2 capacity. Cost optimization is a key exam topic.

---

## Amazon ECR (Q17–Q20)

**Question 17:**
A development team builds Docker images in their CI/CD pipeline and needs a private, fully managed registry on AWS to store and version these images. The images should be automatically scanned for known vulnerabilities on push. Which service and configuration meets these requirements?

- [ ] Docker Hub with IAM authentication
- [ ] Amazon S3 with versioning enabled to store Docker tar files
- [x] Amazon ECR with a private repository and image scanning on push enabled
- [ ] AWS CodeArtifact configured for container images

> **Explanation:** **Amazon ECR** is AWS's fully managed Docker registry. Private repositories are the default, secured by IAM. ECR supports **image scanning on push** using the Amazon Inspector integration (or basic scanning) to detect CVEs. Images are stored durably in S3 (managed by AWS) with lifecycle policies for automatic cleanup of old tags.

---

**Question 18:**
A company has ECR repositories in us-east-1 and wants their ECS clusters in eu-west-1 to pull images without cross-region latency and bandwidth costs. What ECR feature addresses this?

- [ ] CloudFront distribution in front of ECR
- [ ] Pull images from Docker Hub instead (global CDN)
- [x] ECR Cross-Region Replication — automatically replicates images to ECR repositories in eu-west-1, so ECS tasks pull from the local region
- [ ] S3 Cross-Region Replication for the underlying storage

> **Explanation:** ECR supports **cross-region replication** — you configure replication rules, and images are automatically replicated to ECR repositories in the target region. ECS tasks in eu-west-1 then pull from the local ECR repository, eliminating cross-region latency and reducing data transfer costs. ECR also supports **cross-account replication**.

---

**Question 19:**
An ECR repository has accumulated hundreds of old, untagged images that are consuming storage. The team wants to automatically clean up images that are older than 30 days or that exceed a count of 10 tagged images. What should they configure?

- [ ] A CloudWatch Events rule that triggers a Lambda to delete old images
- [ ] S3 Lifecycle policies on the underlying ECR storage
- [x] ECR Lifecycle Policies — rule-based automatic cleanup of untagged images, images older than a specified number of days, or images exceeding a count threshold
- [ ] Manual deletion via the AWS CLI on a cron schedule

> **Explanation:** **ECR Lifecycle Policies** allow you to define rules for automatic image cleanup based on age, count, or tag status. For example: "Keep only the 10 most recent tagged images" or "Delete untagged images older than 30 days." This is a managed feature — no Lambda or custom scripting required.

---

**Question 20:**
A company wants to share a set of base Docker images publicly with external partners and the open-source community. They want the images to be downloadable without requiring AWS credentials. Which ECR feature should they use?

- [ ] Private ECR repository with a resource-based policy granting public access
- [ ] Cross-account replication to partner AWS accounts
- [x] ECR Public Gallery (public.ecr.aws) — publicly accessible repositories that allow unauthenticated image pulls
- [ ] Upload images to Docker Hub instead

> **Explanation:** **ECR Public Gallery** (public.ecr.aws) allows you to create public repositories. Anyone can pull images without AWS credentials or authentication. Private ECR repositories (default) require IAM-based access. ECR Public is AWS's alternative to Docker Hub for public image distribution.

---

## Amazon EKS & Orchestrator Comparison (Q21–Q26)

**Question 21:**
A company migrating to AWS currently runs Kubernetes clusters on-premises. Their operations team has deep Kubernetes expertise and uses Helm charts, Istio service mesh, and custom Kubernetes operators. They want a managed Kubernetes service on AWS. Which service should they use?

- [ ] Amazon ECS — it's simpler and preferred on AWS
- [x] Amazon EKS — fully managed Kubernetes that is 100% API-compatible with upstream Kubernetes, so existing Helm charts, Istio, and custom operators work without modification
- [ ] Amazon ECS with Docker Compose integration
- [ ] AWS App Runner

> **Explanation:** **EKS** runs certified, conformant Kubernetes. All standard Kubernetes tooling (kubectl, Helm, Istio, Prometheus, custom CRDs/operators) works unchanged. The control plane (API server, etcd, scheduler, controller manager) is fully managed by AWS across multiple AZs. ECS uses a proprietary orchestration model that is not Kubernetes-compatible.

---

**Question 22:**
An EKS cluster needs worker nodes. The team wants AWS to handle node provisioning, patching, and scaling, while still having the ability to use Spot Instances for cost savings. Which EKS node type should they use?

- [ ] Self-Managed Nodes with a custom launch template
- [x] Managed Node Groups — AWS creates and manages the EC2 instances (ASG), handles OS patching, and supports both On-Demand and Spot Instances
- [ ] Fargate Profiles — serverless pods with Spot pricing
- [ ] EKS Anywhere with on-premises hardware

> **Explanation:** **EKS Managed Node Groups** provide the best balance of management and flexibility. AWS handles the EC2 lifecycle (provisioning, patching, draining during updates) while you maintain control over instance types, including Spot Instances for cost savings. Fargate is fully serverless but doesn't support Spot pricing. Self-Managed Nodes require you to handle all patching and scaling.

---

**Question 23:**
A company runs both ECS services and EKS clusters. They want serverless compute for both orchestrators — no EC2 instances to manage. Is this possible?

- [ ] No — Fargate only works with ECS
- [ ] No — Fargate only works with EKS
- [x] Yes — AWS Fargate is a serverless compute engine that works with BOTH ECS (Tasks) and EKS (Pods). No EC2 instances to manage in either case.
- [ ] Yes — but only if the ECS services and EKS clusters are in the same VPC

> **Explanation:** **Fargate is a compute engine, not an orchestrator.** It provides serverless compute for containers and works with both ECS (running Tasks) and EKS (running Pods). In both cases, you define CPU/memory requirements, and Fargate provisions the compute automatically. No EC2 instances, no ECS Agent, no kubelet to manage.

---

**Question 24:**
A solutions architect is choosing between ECS and EKS for a new project. The company is fully committed to AWS, has no existing Kubernetes expertise, and wants the simplest possible container orchestration with deep AWS service integration. Which should they choose and why?

- [x] Amazon ECS — it's AWS-native with a simpler learning curve, deep integration with ALB/CloudWatch/IAM, and doesn't require Kubernetes knowledge
- [ ] Amazon EKS — Kubernetes is the industry standard and should always be chosen
- [ ] Both — run ECS for simple services and EKS for complex ones in the same cluster
- [ ] Neither — use AWS Lambda for all container workloads

> **Explanation:** For **AWS-native, new projects without Kubernetes requirements**, ECS is the recommended choice. It has a simpler operational model, tighter AWS integration (Task Definitions are a straightforward JSON format), and no Kubernetes learning curve. EKS is preferred when you have existing Kubernetes workloads, need multi-cloud portability, or want the broader Kubernetes ecosystem.

---

**Question 25:**
A company wants to run ECS tasks on their on-premises servers while managing them from the AWS console. Their data center servers run Linux and are connected to AWS via Direct Connect. Which feature enables this?

- [ ] ECS with EC2 Launch Type using VPN-connected instances
- [ ] EKS Anywhere
- [x] ECS Anywhere — install the ECS Agent and SSM Agent on on-premises servers, register them as EXTERNAL instances in the ECS cluster, and manage tasks from the AWS ECS console/API
- [ ] AWS Outposts with ECS installed

> **Explanation:** **ECS Anywhere** extends ECS to on-premises infrastructure. You install the ECS Agent and SSM Agent on your servers, register them as EXTERNAL capacity in your ECS cluster, and then deploy and manage tasks using the standard ECS console, CLI, and APIs. This is different from EKS Anywhere (which runs the full Kubernetes stack on-premises).

---

**Question 26:**
An EKS cluster uses Fargate profiles for workloads in the `web` namespace and Managed Node Groups for workloads in the `gpu-compute` namespace. A new pod is scheduled in the `web` namespace. Where will it run?

- [ ] On one of the Managed Node Group EC2 instances
- [x] On Fargate — the Fargate profile matches pods in the `web` namespace, so EKS schedules them on Fargate compute
- [ ] Kubernetes randomly selects between Fargate and Node Group
- [ ] The pod will fail to schedule because mixed compute types are not supported

> **Explanation:** **EKS Fargate Profiles** define selectors (namespace + optional labels) that determine which pods run on Fargate. A pod in the `web` namespace matches the Fargate profile, so it runs serverlessly on Fargate. Pods in `gpu-compute` don't match the Fargate profile, so they're scheduled on the Managed Node Group EC2 instances. EKS fully supports mixed compute (Fargate + EC2) in the same cluster.

---

## Architecture Patterns & Integration (Q27–Q30)

**Question 27:**
A company wants to trigger ECS Fargate tasks on a schedule — for example, run a data processing container every night at 2 AM. The task runs for about 30 minutes and then exits. They don't want to keep any infrastructure running between executions. What is the best approach?

- [ ] Run an EC2 instance with a cron job that calls the ECS RunTask API
- [ ] Use a Lambda function on a schedule to run the processing logic
- [x] Create an Amazon EventBridge (CloudWatch Events) scheduled rule that triggers an ECS RunTask API call to launch the Fargate task
- [ ] Deploy an ECS Service with desired count = 1 and manually stop it after processing

> **Explanation:** **EventBridge scheduled rules** can directly invoke the ECS `RunTask` API to launch a Fargate task on a cron schedule. The task runs, completes its work, and exits — no infrastructure runs between executions, and no ECS Service is needed (services are for long-running tasks that need to maintain a desired count). This is the serverless batch pattern.

---

**Question 28:**
A three-tier application on ECS uses an ALB for the web tier. The web tier ECS service needs to communicate with a backend API tier ECS service. The backend should NOT be exposed to the internet. How should inter-service communication be designed?

- [ ] Expose the backend service through the same public ALB on a different path
- [ ] Use S3 as an intermediary between services
- [x] Deploy an internal (private) ALB for the backend service — the web tier tasks communicate with the backend via the internal ALB DNS name. Only the web tier ALB is internet-facing.
- [ ] Hardcode the IP addresses of backend tasks in the web tier configuration

> **Explanation:** An **internal ALB** (scheme: internal) is only accessible within the VPC — not from the internet. The web tier uses the internal ALB's DNS name to communicate with backend tasks. This provides load balancing, health checks, and service discovery for internal communication while keeping the backend private. ECS Service Connect and AWS Cloud Map are also options for service discovery.

---

**Question 29:**
A company runs a microservices application on ECS. They need to track the health and performance of individual containers, including CPU utilization, memory usage, network traffic, and the number of running/pending tasks per service. Which AWS feature provides these container-level insights?

- [ ] Basic CloudWatch EC2 metrics (CPUUtilization, NetworkIn)
- [ ] VPC Flow Logs for container network traffic
- [x] Amazon CloudWatch Container Insights — provides automated dashboards with container-level, task-level, and service-level metrics for ECS (and EKS)
- [ ] AWS X-Ray for container profiling

> **Explanation:** **CloudWatch Container Insights** collects, aggregates, and summarizes metrics and logs from containerized applications on ECS and EKS. It provides pre-built dashboards showing CPU/memory utilization, running/pending task counts, and network metrics at the cluster, service, task, and container level. It goes far beyond basic EC2 metrics, which don't understand container boundaries.

---

**Question 30:**
A solutions architect evaluates AWS container services. Match each scenario to the correct service or feature:

| Scenario | Service |
|---|---|
| A. Store Docker images with vulnerability scanning | ? |
| B. Run containers serverlessly with no EC2 management | ? |
| C. Migrate existing Kubernetes workloads to AWS | ? |
| D. Run ECS tasks on on-premises servers | ? |
| E. Orchestrate containers without Kubernetes knowledge | ? |

Which mapping is correct?

- [ ] A=Docker Hub, B=Lambda, C=ECS, D=EKS Anywhere, E=EKS
- [ ] A=ECR, B=EC2 Launch Type, C=ECS, D=AWS Outposts, E=EKS
- [x] A=Amazon ECR, B=AWS Fargate, C=Amazon EKS, D=ECS Anywhere, E=Amazon ECS
- [ ] A=S3, B=Fargate, C=EKS, D=ECS Anywhere, E=ECS

> **Explanation:** A → **ECR** (managed Docker registry with vulnerability scanning). B → **Fargate** (serverless compute for containers — no EC2 instances). C → **EKS** (managed Kubernetes — fully K8s API-compatible). D → **ECS Anywhere** (run ECS tasks on on-premises servers). E → **ECS** (AWS-native container orchestration — simpler than Kubernetes).
