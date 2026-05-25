# AWS Storage Extras — Practice Questions (Set 2)

> Topics: Snow Family, FSx, Storage Gateway, Transfer Family, DataSync, Storage Comparison
> 30 scenario-based questions for SAA-C03 preparation

---

## Snow Family (Q1–Q6)

**Question 1:**
A research vessel at sea collects 6 TB of ocean sensor data over a 3-month expedition. Internet connectivity is satellite-based at 5 Mbps. The team needs to transfer the data to S3 when they return to port. Which is the MOST cost-effective and practical solution?

- [ ] Upload data to S3 using S3 Transfer Acceleration over satellite
- [ ] Use AWS Direct Connect at the port
- [x] Use an AWS Snowcone device — small, portable, and fits the data size
- [ ] Use a Snowball Edge Storage Optimized device

> **Explanation:** Snowcone is the smallest Snow device (8–14 TB), weighing only 4.5 lbs — perfect for small, portable field deployments. 6 TB fits within its capacity. A full Snowball Edge (80 TB) is overkill. Satellite upload at 5 Mbps would take ~111 days.

---

**Question 2:**
A company is decommissioning an on-premises data center with 40 PB of data. They need to migrate everything to Amazon S3 within 6 months. Their available network bandwidth is 10 Gbps. What is the recommended migration approach?

- [ ] Use AWS DataSync over the 10 Gbps link
- [ ] Order 500 Snowball Edge Storage Optimized devices
- [x] Use AWS Snowmobile — designed for migrations exceeding 10 PB
- [ ] Use S3 multi-part upload with Transfer Acceleration

> **Explanation:** At 40 PB, Snowmobile (100 PB capacity per truck) is the recommended option. The rule of thumb is: > 10 PB → Snowmobile. Even at 10 Gbps, 40 PB would take ~370 days to transfer over the network.

---

**Question 3:**
An edge computing application at a mining site needs to run ML inference models on Snowball Edge. The team wants to cluster multiple devices together for increased storage and compute. How many Snowball Edge devices can be clustered?

- [ ] 2–3 devices
- [x] 5–15 devices
- [ ] Up to 50 devices
- [ ] Snowball Edge does not support clustering

> **Explanation:** Snowball Edge supports clustering of 5–15 devices to increase storage capacity and compute power at edge locations.

---

**Question 4:**
A company uses AWS Snowcone at a remote weather station. After collecting data, they realize the station has intermittent but usable internet connectivity (50 Mbps). Instead of shipping the device back, what alternative does Snowcone uniquely offer?

- [ ] Upload data directly to S3 using the AWS CLI on Snowcone
- [ ] Use AWS Storage Gateway agent on Snowcone
- [x] Use the pre-installed AWS DataSync agent to transfer data over the network
- [ ] Stream data to Amazon Kinesis from Snowcone

> **Explanation:** Snowcone is the only Snow device with AWS DataSync pre-installed, allowing data transfer over the network instead of physically shipping the device.

---

**Question 5:**
After loading 60 TB of archival records onto a Snowball Edge Storage Optimized device, the data needs to end up in S3 Glacier Deep Archive for compliance. What is the correct workflow?

- [ ] Configure the Snowball Edge to write directly to Glacier Deep Archive
- [ ] Select Glacier Deep Archive as the import destination in the AWS Console
- [x] Import the data to S3 Standard, then create an S3 Lifecycle Rule to transition to Glacier Deep Archive
- [ ] Use AWS DataSync to move data from Snowball to Glacier

> **Explanation:** Snow devices cannot import directly into Glacier. Data must first land in S3 (any non-Glacier class), then S3 Lifecycle Rules handle the transition to Glacier or Deep Archive.

---

**Question 6:**
What software tool does AWS provide to manage Snow Family devices through a graphical interface, replacing the previous CLI-only approach?

- [ ] AWS Systems Manager
- [ ] AWS CloudFormation
- [ ] Snow Device Management Console in AWS
- [x] AWS OpsHub — a desktop GUI application installed on your laptop

> **Explanation:** AWS OpsHub is a graphical application installed on your local computer. It lets you manage Snow devices, transfer data, launch EC2 instances, and monitor device status through a visual interface.

---

## FSx (Q7–Q12)

**Question 7:**
A financial services company runs HPC simulations that require sub-millisecond latency access to 200 TB of market data stored in S3. The compute cluster runs on Linux EC2 instances. They need the highest possible throughput. Which solution provides the best performance?

- [ ] Mount S3 directly using S3 File Gateway
- [ ] Use Amazon EFS with Max I/O performance mode
- [x] Create FSx for Lustre linked to the S3 bucket as a data repository
- [ ] Copy all data to EBS io2 Block Express volumes

> **Explanation:** FSx for Lustre provides 100s GB/s throughput with sub-ms latency and integrates transparently with S3 as a data repository — purpose-built for HPC workloads on Linux.

---

**Question 8:**
A company is migrating from an on-premises NetApp ONTAP storage system. Their applications use a mix of NFS, SMB, and iSCSI protocols simultaneously. Which AWS service provides the smoothest migration path?

- [ ] Amazon EFS for NFS + FSx for Windows for SMB
- [ ] Use three separate Storage Gateway types
- [x] FSx for NetApp ONTAP — supports NFS, SMB, and iSCSI simultaneously
- [ ] FSx for OpenZFS with SMB client installed

> **Explanation:** FSx for NetApp ONTAP is the only FSx variant supporting all three protocols (NFS + SMB + iSCSI) simultaneously. It also provides ONTAP-specific features like SnapMirror, FlexClone, auto-tiering, and deduplication — ideal for NetApp migrations.

---

**Question 9:**
A company runs FSx for Lustre with a **Scratch** deployment type. A hardware failure on one of the underlying file servers causes data loss. The operations team is surprised. Why did data loss occur?

- [ ] FSx for Lustre does not encrypt data at rest
- [ ] They forgot to enable Cross-Region replication
- [x] Scratch deployment type does not replicate data — it prioritizes burst throughput over durability
- [ ] Lustre does not support Linux workloads

> **Explanation:** FSx for Lustre Scratch deployment does NOT replicate data. If the underlying server fails, data is lost. It's designed for temporary processing workloads where the source data can be re-loaded from S3. For durable storage, use Persistent deployment.

---

**Question 10:**
Which FSx file system variant supports **Multi-AZ deployment** with automatic failover for high availability?

- [ ] FSx for Lustre
- [x] FSx for Windows File Server
- [ ] FSx for OpenZFS
- [ ] All FSx file systems support Multi-AZ

> **Explanation:** Only FSx for Windows File Server and FSx for NetApp ONTAP support Multi-AZ deployment. FSx for Lustre and FSx for OpenZFS are single-AZ only.

---

**Question 11:**
A development team currently runs ZFS-based file servers on-premises for their CI/CD pipelines and internal databases. They want to migrate to AWS with minimal changes. Which service should they use?

- [ ] Amazon EFS
- [ ] FSx for Windows File Server
- [ ] FSx for Lustre
- [x] FSx for OpenZFS — designed for migrating ZFS workloads to AWS

> **Explanation:** FSx for OpenZFS is purpose-built for migrating existing ZFS workloads. It supports NFS protocol, point-in-time snapshots, data compression, instant cloning, and copy-on-write — matching on-premises ZFS feature sets.

---

**Question 12:**
A data engineering team uses FSx for Lustre linked to an S3 data repository. They notice that when a compute job first accesses a file, there is a slight delay, but subsequent accesses are instant. What explains this behavior?

- [ ] FSx for Lustre is throttling read requests
- [ ] The S3 bucket has requester-pays enabled
- [x] FSx for Lustre uses lazy loading — data is fetched from S3 only on first access, then cached in the file system
- [ ] The EBS volumes backing Lustre need to be warmed up

> **Explanation:** FSx for Lustre with S3 data repository uses lazy loading. Files are not pre-copied from S3; they are loaded into the Lustre file system only when first accessed. After that, subsequent reads are served from the high-performance Lustre storage.

---

## Storage Gateway (Q13–Q18)

**Question 13:**
An on-premises application writes data to a local NFS share. The company wants to archive this data to S3 Glacier for long-term retention. The application cannot be modified to use the S3 API. Which architecture achieves this with minimal changes?

- [ ] Deploy Tape Gateway and modify the application to use iSCSI
- [x] Deploy S3 File Gateway — the application writes to NFS, data goes to S3, then S3 Lifecycle Rules transition to Glacier
- [ ] Use AWS DataSync to sync the NFS share to Glacier directly
- [ ] Use AWS Snowball to periodically ship data to Glacier

> **Explanation:** S3 File Gateway presents S3 as an NFS mount point. The application writes files via NFS (no changes needed), files are stored as S3 objects, and Lifecycle Rules handle the transition to Glacier. The gateway cannot write directly to Glacier.

---

**Question 14:**
A hospital runs backup software (Veritas NetBackup) that writes to physical tape drives using the iSCSI protocol. The company wants to eliminate physical tape infrastructure while keeping the same backup software and workflows. Which Storage Gateway type should they use?

- [ ] S3 File Gateway
- [ ] Volume Gateway — Cached mode
- [ ] FSx File Gateway
- [x] Tape Gateway — presents virtual tapes via iSCSI-VTL, backed by S3 and Glacier

> **Explanation:** Tape Gateway emulates a Virtual Tape Library (VTL) over iSCSI. The backup software (NetBackup) continues to "write tapes" as before, but the tapes are virtual and stored in S3 (active) and S3 Glacier/Deep Archive (archived). No application changes needed.

---

**Question 15:**
A company uses Volume Gateway in Cached Volumes mode. They want to create EC2 instances from this data for disaster recovery. How can they achieve this?

- [ ] Directly attach the Volume Gateway to EC2
- [x] Create EBS Snapshots from the Volume Gateway volumes, then create EBS volumes from the snapshots and attach to EC2
- [ ] Use AWS DataSync to copy Volume Gateway data to EBS
- [ ] Mount the Volume Gateway as an NFS share on EC2

> **Explanation:** Volume Gateway backs up data as EBS Snapshots in S3. These snapshots can be used to create EBS volumes, which can then be attached to EC2 instances — enabling disaster recovery or cloud migration from on-premises block storage.

---

**Question 16:**
A company needs their on-premises Windows servers to access an FSx for Windows File Server in AWS with low-latency performance for frequently accessed files. Users authenticate via Active Directory. Which solution provides local caching?

- [ ] S3 File Gateway with SMB
- [ ] Volume Gateway — Cached mode
- [x] FSx File Gateway — caches frequently accessed data locally, connects to FSx for Windows via VPN/Direct Connect
- [ ] AWS DataSync with scheduled transfers

> **Explanation:** FSx File Gateway is specifically designed for this use case: providing on-premises Windows users with low-latency SMB access to FSx for Windows File Server, with a local cache for hot data. Requires VPN or Direct Connect connectivity.

---

**Question 17:**
A Storage Gateway hardware appliance fails at a remote office with no VMware, Hyper-V, or KVM infrastructure. The office has limited IT staff. What is the recommended replacement?

- [ ] Deploy a new VM-based gateway on a cloud server
- [ ] Ship a Snowball Edge to act as a gateway
- [x] Order another AWS Storage Gateway Hardware Appliance — designed for sites without virtualization infrastructure
- [ ] Use AWS DataSync instead

> **Explanation:** The Storage Gateway Hardware Appliance is a physical device from AWS designed for locations without existing virtualization infrastructure. It's the only option when you can't deploy a VM-based gateway.

---

**Question 18:**
An on-premises application requires low-latency access to the FULL dataset (not just frequently accessed data), while also needing asynchronous backups to AWS for disaster recovery. Which Volume Gateway mode should they use?

- [ ] Volume Gateway — Cached Volumes (data in S3, cache local)
- [x] Volume Gateway — Stored Volumes (full data local, async backups to S3 as EBS Snapshots)
- [ ] S3 File Gateway with large local cache
- [ ] FSx File Gateway

> **Explanation:** Stored Volumes keeps the **entire dataset on-premises** for low-latency access to all data, with asynchronous point-in-time snapshots uploaded to S3 as EBS Snapshots. Cached Volumes only keeps hot data locally — the primary copy lives in S3.

---

## Transfer Family (Q19–Q22)

**Question 19:**
A healthcare company exchanges EDI (Electronic Data Interchange) documents with insurance partners. They need a fully managed solution that supports the **AS2 protocol** for B2B data exchange, storing files in S3. Which service should they use?

- [ ] Amazon MQ with AMQP
- [ ] AWS AppSync
- [x] AWS Transfer Family with AS2 protocol
- [ ] Amazon SQS with S3 event notifications

> **Explanation:** AWS Transfer Family supports SFTP, FTPS, FTP, and AS2 protocols. AS2 (Applicability Statement 2) is specifically designed for B2B data exchange commonly used in healthcare (HIPAA), retail, and finance for EDI workflows.

---

**Question 20:**
A company uses AWS Transfer Family for SFTP. They want each external partner to have different S3 access permissions — Partner A can only write to `s3://bucket/partner-a/` and Partner B can only access `s3://bucket/partner-b/`. How is this configured?

- [ ] Create separate Transfer Family servers for each partner
- [x] Map each user to a different IAM Role with scoped S3 permissions (bucket and prefix)
- [ ] Use S3 Bucket ACLs to restrict per-partner access
- [ ] Configure separate VPC endpoints per partner

> **Explanation:** In AWS Transfer Family, each user is mapped to an IAM Role that defines their specific S3/EFS permissions — which bucket, which prefix, read-only vs read-write. This enables fine-grained per-user access control.

---

**Question 21:**
A legacy on-premises system needs to push files to S3 using the FTP protocol (unencrypted). How should the AWS Transfer Family endpoint be configured for security?

- [ ] Public endpoint with TLS termination at the load balancer
- [ ] CloudFront distribution with HTTPS
- [x] VPC endpoint only — FTP is unencrypted and must not traverse the public internet
- [ ] FTP is not supported by AWS Transfer Family

> **Explanation:** AWS Transfer Family supports plain FTP, but since it is unencrypted, it should only be deployed as a VPC endpoint (internal). For internet-facing transfers, use SFTP or FTPS instead.

---

**Question 22:**
A company wants to authenticate AWS Transfer Family SFTP users against their existing corporate LDAP directory, which is not compatible with AWS Directory Service. What authentication method should they use?

- [ ] Service-managed SSH keys only
- [ ] AWS IAM users with access keys
- [x] Custom Identity Provider using an AWS Lambda function to authenticate against the LDAP directory
- [ ] Transfer Family does not support custom authentication

> **Explanation:** Transfer Family supports three auth methods: service-managed (SSH keys), AWS Directory Service (AD), and Custom Identity Provider (Lambda / API Gateway). For non-AD LDAP or any custom identity store, use a Lambda-based custom IdP.

---

## DataSync (Q23–Q26)

**Question 23:**
A company needs to perform a one-time migration of 50 TB of NFS file data from an on-premises data center to Amazon EFS. They have a 1 Gbps Direct Connect link. Which service is purpose-built for this high-speed, automated data transfer?

- [ ] AWS Storage Gateway — S3 File Gateway
- [ ] rsync over SSH on an EC2 instance
- [x] AWS DataSync — agent-based, automated, high-speed data transfer optimized for migrations
- [ ] AWS Snowball Edge

> **Explanation:** AWS DataSync is purpose-built for moving large amounts of data between on-premises storage and AWS (S3, EFS, FSx). It uses a software agent on-premises, handles scheduling, integrity validation, bandwidth throttling, and can saturate network links for maximum throughput.

---

**Question 24:**
What is the key architectural difference between AWS DataSync and AWS Storage Gateway?

- [ ] DataSync is for cloud-to-cloud transfers; Storage Gateway is for on-premises only
- [ ] Storage Gateway encrypts data; DataSync does not
- [x] DataSync is for **data migration/transfer tasks** (move data); Storage Gateway is for **ongoing hybrid access** (bridge on-premises apps to cloud storage with local caching)
- [ ] They are the same service with different names

> **Explanation:** DataSync = **move data** (migration, replication, sync). Storage Gateway = **use cloud storage** from on-premises (continuous hybrid access with local caching). DataSync is task-oriented; Storage Gateway is always-on.

---

**Question 25:**
AWS DataSync supports transferring data between which of the following location pairs? (Select TWO)

- [x] On-premises NFS/SMB server → Amazon S3
- [ ] Amazon RDS → Amazon DynamoDB
- [x] Amazon EFS in one region → Amazon EFS in another region
- [ ] Amazon Redshift → Amazon S3 Glacier

> **Explanation:** DataSync supports transfers between on-premises (NFS, SMB, HDFS, self-managed object storage) and AWS (S3, EFS, FSx). It also supports AWS-to-AWS transfers (e.g., EFS-to-EFS across regions, S3-to-S3 across accounts). It does NOT work with databases like RDS or Redshift.

---

**Question 26:**
A company runs a scheduled nightly sync of changed files from an on-premises NFS share to Amazon S3 using AWS DataSync. They want to minimize bandwidth usage. What DataSync feature helps with this?

- [ ] DataSync always copies all files regardless of changes
- [x] DataSync performs incremental transfers — only changed or new files are transferred after the initial full copy
- [ ] DataSync compresses all data using gzip before transfer
- [ ] DataSync requires manual identification of changed files

> **Explanation:** DataSync automatically detects changed files using checksums and timestamps, then transfers only the differences (incremental sync). This dramatically reduces bandwidth usage for scheduled, recurring sync tasks.

---

## Storage Comparison — Cross-Service (Q27–Q30)

**Question 27:**
Match each scenario to the correct AWS storage/transfer service:

| Scenario | Service |
|---|---|
| A. Linux EC2 fleet needs shared NFS storage across 3 AZs | ? |
| B. Windows servers need shared SMB storage with Active Directory | ? |
| C. HPC cluster needs sub-ms latency access to S3 data | ? |
| D. On-premises NFS share needs to be backed by S3 | ? |
| E. Partners upload files via SFTP into S3 | ? |

Which mapping is correct?

- [ ] A=FSx Windows, B=EFS, C=FSx Lustre, D=S3 File Gateway, E=Transfer Family
- [ ] A=EFS, B=EFS, C=FSx ONTAP, D=Volume Gateway, E=DataSync
- [x] A=EFS, B=FSx for Windows, C=FSx for Lustre, D=S3 File Gateway, E=AWS Transfer Family
- [ ] A=FSx Lustre, B=FSx Windows, C=EBS io2, D=DataSync, E=Transfer Family

> **Explanation:** A → EFS (Linux NFS, Multi-AZ). B → FSx Windows (SMB + AD). C → FSx Lustre (HPC + S3 integration). D → S3 File Gateway (on-premises NFS → S3). E → Transfer Family (SFTP/FTPS/FTP → S3).

---

**Question 28:**
A solutions architect needs to choose between AWS DataSync, Storage Gateway, Snow Family, and Transfer Family. Which statement correctly distinguishes ALL four services?

- [ ] They all provide ongoing hybrid storage access from on-premises
- [ ] They all require a physical device shipped to your data center
- [x] DataSync = bulk migration/sync, Storage Gateway = continuous hybrid access with caching, Snow Family = offline physical transfer + edge compute, Transfer Family = managed SFTP/FTPS/FTP/AS2 endpoint
- [ ] DataSync and Transfer Family are the same service; Snow Family and Storage Gateway are the same service

> **Explanation:** Each service has a distinct purpose: **DataSync** moves data at scale (online). **Storage Gateway** bridges on-premises to cloud with local caching (ongoing). **Snow Family** physically ships data + edge compute (offline). **Transfer Family** provides managed file transfer protocol endpoints (SFTP/FTPS/FTP/AS2) into S3/EFS.

---

**Question 29:**
A company has the following requirements:
1. 500 TB of on-premises data to migrate to S3 (one-time)
2. After migration, on-premises apps must continue accessing S3 data via NFS with local caching
3. External partners must upload new data via SFTP

Which combination of services satisfies ALL three requirements?

- [ ] AWS Snowball Edge for all three
- [ ] AWS DataSync for all three
- [x] Snowball Edge (migration) + S3 File Gateway (ongoing NFS access with cache) + AWS Transfer Family (SFTP for partners)
- [ ] S3 File Gateway (migration) + DataSync (NFS access) + Transfer Family (SFTP)

> **Explanation:** Each requirement maps to a different service: (1) 500 TB one-time migration → Snowball Edge (network transfer at 1 Gbps would take ~46 days). (2) Ongoing NFS access with local caching → S3 File Gateway. (3) SFTP for external partners → AWS Transfer Family. This is a classic "pick the right combination" exam question.

---

**Question 30:**
A company is evaluating AWS storage services. Which comparison table is CORRECT?

| Service | Protocol | Backend | On-Premises Agent/Device? | Primary Purpose |
|---|---|---|---|---|
| A | NFS/SMB | S3 | Yes (VM/HW appliance) | Hybrid cloud access |
| B | SFTP/FTPS/FTP/AS2 | S3 or EFS | No | File transfer endpoint |
| C | Agent-based | S3/EFS/FSx | Yes (software agent) | Data migration & sync |
| D | N/A (physical ship) | S3 | Yes (physical device) | Offline migration + edge compute |

- [ ] A=Transfer Family, B=Storage Gateway, C=Snow Family, D=DataSync
- [x] A=Storage Gateway (S3 File GW), B=Transfer Family, C=DataSync, D=Snow Family
- [ ] A=DataSync, B=Transfer Family, C=Storage Gateway, D=Snow Family
- [ ] A=Storage Gateway, B=DataSync, C=Transfer Family, D=Snow Family

> **Explanation:** A = Storage Gateway (on-premises VM/HW, NFS/SMB, backs to S3 with local caching). B = Transfer Family (managed SFTP/FTPS/FTP/AS2 endpoint, stores in S3/EFS). C = DataSync (agent-based, moves data to S3/EFS/FSx). D = Snow Family (physical devices shipped to/from AWS).
