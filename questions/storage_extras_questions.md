# AWS Storage Extras Questions

**Question 1:**
A company needs to migrate their Windows-based file shares to AWS. The file server uses SMB protocol and integrates with Microsoft Active Directory for user authentication. Which AWS service should they use?

- [ ] Amazon EFS
- [ ] Amazon S3 with S3 File Gateway
- [x] Amazon FSx for Windows File Server
- [ ] Amazon EBS with Multi-Attach

**Question 2:**
A machine learning team needs to process 500 TB of training data stored in S3 with sub-millisecond latency and hundreds of GB/s throughput on a Linux compute cluster. Which file system should they use?

- [ ] Amazon EFS
- [ ] Amazon FSx for Windows File Server
- [x] Amazon FSx for Lustre
- [ ] Amazon EBS io2 Block Express

**Question 3:**
An FSx for Lustre file system is linked to an S3 bucket as a data repository. When does data from S3 get loaded into the Lustre file system?

- [ ] Immediately when the file system is created — all S3 data is copied
- [ ] According to a schedule you configure
- [x] Lazily — data is loaded from S3 only when it is first accessed
- [ ] Data is never loaded from S3; it must be manually copied

**Question 4:**
Which FSx for Lustre deployment type should you use for a short-term data processing job where the source data is in S3 and can be re-loaded if lost?

- [x] Scratch — optimized for temporary workloads with highest burst throughput
- [ ] Persistent — replicated within the same AZ
- [ ] Multi-AZ — replicated across AZs
- [ ] One Zone — lowest cost

**Question 5:**
A company is migrating from an on-premises NetApp storage appliance and needs a file system that supports NFS, SMB, and iSCSI protocols simultaneously. Which FSx variant should they use?

- [ ] FSx for Windows File Server
- [ ] FSx for Lustre
- [x] FSx for NetApp ONTAP
- [ ] FSx for OpenZFS

**Question 6:**
Multiple Linux EC2 instances across three Availability Zones need shared file access using NFS. The access pattern is simple file reads and writes. Which service is the best fit?

- [x] Amazon EFS
- [ ] Amazon FSx for Lustre
- [ ] Amazon FSx for Windows File Server
- [ ] Amazon FSx for NetApp ONTAP

---

**Question 7:**
A company needs to migrate 60 TB of data from their on-premises data center to Amazon S3. Their internet connection is 100 Mbps, which would take approximately 70 days. What is the most practical solution?

- [ ] Use S3 Transfer Acceleration
- [ ] Set up AWS Direct Connect
- [x] Order an AWS Snowball Edge Storage Optimized device
- [ ] Use AWS DataSync over the internet

**Question 8:**
Which AWS Snow Family device is the smallest and lightest, weighing only 4.5 lbs, and is designed for edge locations with constrained space and power?

- [x] AWS Snowcone
- [ ] AWS Snowball Edge Storage Optimized
- [ ] AWS Snowball Edge Compute Optimized
- [ ] AWS Snowmobile

**Question 9:**
A company needs to perform machine learning inference at a remote oil rig with no internet connectivity. Which Snow Family device should they use?

- [ ] AWS Snowcone
- [ ] AWS Snowball Edge Storage Optimized
- [x] AWS Snowball Edge Compute Optimized (with optional GPU)
- [ ] AWS Snowmobile

**Question 10:**
A company is shutting down three data centers containing a total of 50 PB of data. What is the most efficient way to migrate this data to AWS?

- [ ] Use 625 Snowball Edge devices (80 TB each)
- [ ] Set up 10 Gbps Direct Connect links
- [x] Use AWS Snowmobile — each Snowmobile handles up to 100 PB
- [ ] Use S3 Transfer Acceleration

**Question 11:**
After receiving a Snowball Edge device and loading your data, you want the data to end up in S3 Glacier Deep Archive for long-term compliance storage. How should you configure this?

- [ ] Select S3 Glacier Deep Archive as the destination when ordering the Snowball
- [ ] Configure the Snowball to write directly to Glacier
- [x] Import data to S3 Standard first, then use S3 Lifecycle Rules to transition to Glacier Deep Archive
- [ ] Use AWS DataSync to move from Snowball to Glacier

**Question 12:**
Which AWS Snow Family device has AWS DataSync agent pre-installed, allowing you to send data back to AWS over the network instead of physically shipping the device?

- [x] AWS Snowcone
- [ ] AWS Snowball Edge Storage Optimized
- [ ] AWS Snowball Edge Compute Optimized
- [ ] All Snow Family devices

---

**Question 13:**
An on-premises application needs to access files stored in Amazon S3 using the NFS protocol, with frequently accessed files cached locally for low-latency access. Which solution should you use?

- [ ] Mount S3 directly using NFS
- [x] Deploy an AWS Storage Gateway — S3 File Gateway
- [ ] Use AWS Transfer Family with SFTP
- [ ] Use AWS DataSync

**Question 14:**
A company uses Volume Gateway in Cached Volumes mode. Where is the primary copy of the data stored?

- [ ] On the on-premises gateway storage
- [x] In Amazon S3 — only frequently accessed data is cached on-premises
- [ ] On Amazon EBS volumes
- [ ] In Amazon EFS

**Question 15:**
A company uses Volume Gateway in Stored Volumes mode. Where is the primary copy of the data stored?

- [x] On-premises — the full dataset is local, with asynchronous backups to S3 as EBS snapshots
- [ ] In Amazon S3 — only frequently accessed data is cached on-premises
- [ ] On Amazon EBS volumes attached to EC2
- [ ] In Amazon Glacier

**Question 16:**
A company wants to replace their aging physical tape backup infrastructure with cloud storage, but their backup software (Veeam) requires an iSCSI Virtual Tape Library interface. Which AWS service should they use?

- [ ] S3 File Gateway
- [ ] Volume Gateway
- [x] Tape Gateway
- [ ] AWS Backup

**Question 17:**
When virtual tapes are archived through Tape Gateway, where are they stored?

- [ ] Amazon EBS snapshots
- [ ] Amazon EFS
- [x] S3 Glacier or S3 Glacier Deep Archive
- [ ] Amazon FSx

**Question 18:**
A company with on-premises Windows file servers wants to extend their storage to AWS with local caching for frequently accessed files. Their users access shares via SMB and authenticate with Active Directory. Which Storage Gateway type should they use?

- [ ] S3 File Gateway
- [x] FSx File Gateway (with FSx for Windows File Server in AWS)
- [ ] Volume Gateway (Cached)
- [ ] Tape Gateway

---

**Question 19:**
External business partners need to upload files to your Amazon S3 bucket using SFTP. You want a fully managed solution with no servers to maintain. Which service should you use?

- [ ] Deploy an EC2 instance running an SFTP server
- [ ] Use S3 pre-signed URLs
- [x] AWS Transfer Family with SFTP protocol
- [ ] AWS Storage Gateway — S3 File Gateway

**Question 20:**
AWS Transfer Family supports the FTP protocol (unencrypted). Where should this endpoint be deployed for security?

- [ ] Public internet endpoint
- [x] VPC endpoint only — FTP traffic is unencrypted and should not traverse the public internet
- [ ] CloudFront distribution
- [ ] Behind an Application Load Balancer

---

**Question 21:**
A company stores all corporate files on-premises and wants to move primary storage to the cloud while maintaining low-latency access. They want to minimize on-premises storage costs. Which solution is best?

- [ ] Volume Gateway — Stored Volumes
- [x] Volume Gateway — Cached Volumes (primary data in S3, hot data cached locally)
- [ ] S3 File Gateway
- [ ] AWS DataSync

**Question 22:**
Which Amazon FSx file system supports Multi-AZ deployment for high availability with automatic failover?

- [x] FSx for Windows File Server
- [ ] FSx for Lustre
- [ ] FSx for OpenZFS
- [ ] All FSx file systems

**Question 23:**
A genomics research lab needs to process S3 data at high throughput using a Linux HPC cluster. After processing, results should be written back to S3. They need sub-millisecond latency during processing. Which architecture is most appropriate?

- [ ] Mount S3 directly to the HPC cluster using S3 File Gateway
- [ ] Copy all data from S3 to EBS volumes attached to HPC instances
- [x] Create an FSx for Lustre file system linked to the S3 bucket as a data repository
- [ ] Use Amazon EFS mounted across HPC instances

**Question 24:**
A company needs to decide between multiple Snowball Edge devices and a single Snowmobile for their data migration. At approximately what threshold does Snowmobile become more practical than Snowball?

- [ ] 1 PB
- [ ] 5 PB
- [x] 10 PB
- [ ] 50 PB

**Question 25:**
An on-premises application currently writes backup data to a local NFS share. The company wants the data to eventually end up in S3 Glacier for long-term archival. Which architecture achieves this with minimal application changes?

- [ ] Rewrite the application to use the S3 API with Glacier storage class
- [x] Deploy S3 File Gateway — application writes to NFS share, data goes to S3, then S3 Lifecycle Rules transition to Glacier
- [ ] Deploy Tape Gateway — application writes to virtual tapes archived to Glacier
- [ ] Use AWS Snowball to periodically move data to Glacier
