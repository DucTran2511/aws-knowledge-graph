# RDS, Aurora & ElastiCache Questions

**Question 1:**
Which of the following is NOT a database engine supported by Amazon RDS?

- [ ] PostgreSQL
- [ ] MariaDB
- [x] MongoDB
- [] Oracle

**Question 2:**
You have an RDS database in a single AZ. You want to improve availability with automatic failover while keeping the same connection endpoint. What should you enable?

- [ ] RDS Read Replica in another AZ
- [x] RDS Multi-AZ (Instance deployment)
- [ ] RDS Proxy
- [ ] Enable Storage Auto Scaling

**Question 3:**
Your Lambda functions are opening too many connections to your RDS database and exhausting the connection pool. What is the recommended AWS solution?

- [ ] Increase the RDS instance size
- [x] Use RDS Proxy for connection pooling
- [ ] Switch to Aurora Serverless v2
- [ ] Enable Multi-AZ Cluster deployment

**Question 4:**
Amazon Aurora stores data in how many copies across how many Availability Zones?

- [ ] 3 copies across 2 AZs
- [ ] 4 copies across 3 AZs
- [x] 6 copies across 3 AZs
- [ ] 6 copies across 2 AZs

**Question 5:**
What is the maximum number of Read Replicas that Amazon Aurora supports?

- [ ] 5
- [x] 15
- [ ] 20

**Question 6:**
You need a cross-region disaster recovery solution for your Aurora MySQL database with less than 1 second of replication lag and an RTO of less than 1 minute. Which feature should you use?

- [ ] Aurora Read Replicas in another region
- [x] Aurora Global Database
- [ ] Aurora Cloning
- [ ] Aurora Backtrack

**Question 7:**
An Aurora feature allows you to rewind the database to a specific point in time without restoring from a backup. What is this feature called?

- [ ] Aurora PITR (Point-in-Time Recovery)
- [x] Aurora Backtrack
- [ ] Aurora Cloning
- [ ] Aurora Snapshots

**Question 8:**
RDS Read Replicas use which type of replication?

- [ ] Synchronous replication
- [x] Asynchronous replication
- [ ] Semi-synchronous replication
- [ ] Logical streaming replication

**Question 9:**
You create an RDS Read Replica in the **same AWS Region** as the primary instance but in a **different AZ**. Will you be charged for network data transfer?

- [ ] Yes, cross-AZ data transfer is always charged
- [x] No, same-region replication is free
- [ ] Yes, but only for the first 1 TB
- [ ] No, but only if both instances are the same instance type

**Question 10:**
What happens when you promote an RDS Read Replica to a standalone database?

- [x] It becomes a new independent DB instance with read/write capability and stops replicating
- [ ] It replaces the primary instance automatically
- [ ] It becomes a Multi-AZ standby instance
- [ ] It continues to replicate from the primary but also accepts writes

**Question 11:**
RDS Multi-AZ **Instance** deployment uses which type of replication to the standby?

- [ ] Asynchronous replication
- [ ] Semi-synchronous replication
- [x] Synchronous replication
- [ ] Logical replication

**Question 12:**
Which statement is TRUE about RDS Multi-AZ deployments?

- [ ] The standby instance can serve read traffic to offload the primary
- [ ] Multi-AZ provides both high availability and read scaling
- [x] The standby is NOT accessible for reads or writes; it is only for failover
- [ ] Multi-AZ requires a manual failover process

**Question 13:**
You need to offload analytics/reporting queries from your production RDS database without impacting performance. What should you use?

- [ ] Enable Multi-AZ and query the standby
- [x] Create a Read Replica and point reporting tools to it
- [ ] Enable RDS Proxy
- [ ] Use RDS Custom

**Question 14:**
RDS Multi-AZ **Cluster** deployment mode offers which advantage over traditional Multi-AZ Instance deployment?

- [ ] Up to 15 readable standby instances
- [x] Two readable standby instances with semi-synchronous replication
- [ ] Cross-region failover capability
- [ ] Zero replication lag guarantee

**Question 15:**
Which ElastiCache engine supports Multi-AZ with automatic failover, persistence, and complex data structures like Sorted Sets?

- [ ] Memcached
- [x] Redis
- [ ] Both Redis and Memcached
- [ ] Neither — ElastiCache doesn't support these features

**Question 16:**
Your application reads the same data from the database repeatedly, and the data changes infrequently. You implement a caching layer. Which caching strategy loads data into the cache only when there is a cache miss?

- [ ] Write-Through
- [ ] Write-Behind (Write-Back)
- [x] Lazy Loading (Cache-Aside)
- [ ] Read-Through with TTL

**Question 17:**
A web application currently stores user session data in an RDS database, causing high read/write load. What is the recommended approach to improve performance and enable session sharing across multiple EC2 instances?

- [ ] Use sticky sessions on the ALB
- [x] Store session data in Amazon ElastiCache (Redis)
- [ ] Store session data in EBS volumes
- [ ] Use RDS Read Replicas for session reads

**Question 18:**
Which of the following is TRUE about ElastiCache for Memcached?

- [ ] It supports Multi-AZ with automatic failover
- [ ] It supports data persistence and backup/restore
- [x] It supports multi-threaded architecture and simple key-value caching
- [ ] It supports replication groups and Cluster Mode

**Question 19:**
You are designing a solution that needs a sub-millisecond in-memory cache specifically optimized for DynamoDB queries. Which service is the best fit?

- [ ] ElastiCache for Redis
- [ ] ElastiCache for Memcached
- [x] Amazon DAX
- [ ] CloudFront caching

**Question 20:**
Your company wants to use IAM-based authentication to connect to their RDS MySQL database instead of traditional username/password. Which feature enables this?

- [ ] RDS Proxy with IAM enforcement
- [x] IAM Database Authentication
- [ ] AWS SSO integration with RDS
- [ ] KMS encryption for RDS

**Question 21:**
We have an RDS database that struggles to keep up with the demand of requests from our website. Our million users mostly read news, and we don't post news very often. Which solution is NOT adapted to this problem?

- [ ] An ElastiCache Cluster
- [ ] RDS Multi-AZ
- [ ] RDS Read Replicas

**Question 22:**
You have set up read replicas on your RDS database, but users are complaining that upon updating their social media posts, they do not see their updated posts right away. What is a possible cause for this?

- [ ] There must be a bug in your application
- [ ] Read Replicas have Asynchronous Replication, therefore it's likely your users will only read Eventual Consistency
- [ ] You should have setup Multi-AZ instead

**Question 23:**
Which RDS (NOT Aurora) feature when used does not require you to change the SQL connection string?

- [ ] Multi-AZ
- [ ] Read Replicas

**Question 24:**
Your application running on a fleet of EC2 instances managed by an Auto Scaling Group behind an Application Load Balancer. Users have to constantly log back in and you don't want to enable Sticky Sessions on your ALB as you fear it will overload some EC2 instances. What should you do?

- [ ] Use your own custom Load Balancer on EC2 instances instead of using ALB
- [ ] Store session data in RDS
- [ ] Store session data in ElastiCache
- [ ] Store session data in a shared EBS volume

**Question 25:**
You would like to create a disaster recovery strategy for your RDS PostgreSQL database so that in case of a regional outage the database can be quickly made available for both read and write workloads in another AWS Region. The DR database must be highly available. What do you recommend?

- [ ] Create a Read Replica in the same region and enable Multi-AZ on the main database
- [ ] Create a Read Replica in a different region and enable Multi-AZ on the Read Replica
- [ ] Create a Read Replica in the same region and enable Multi-AZ on the Read Replica
- [ ] Enable Multi-Region option on the main database

**Question 26:**
How do you encrypt an unencrypted RDS DB instance?

- [ ] Do it straight from AWS Console, select your RDS DB instance, choose Actions then Encrypt using KMS
- [ ] Do it straight from AWS Console, after stopping the RDS DB instance
- [ ] Create a snapshot of the unencrypted RDS DB instance, copy the snapshot and tick "Enable encryption", then restore the RDS DB instance from the encrypted snapshot

**Question 27:**
You have an un-encrypted RDS DB instance and you want to create Read Replicas. Can you configure the RDS Read Replicas to be encrypted?

- [ ] No
- [ ] Yes

**Question 28:**
An application running in production is using an Aurora Cluster as its database. Your development team would like to run a version of the application in a scaled-down application with the ability to perform some heavy workload on a need-basis. Most of the time, the application will be unused. Your CIO has tasked you with helping the team to achieve this while minimizing costs. What do you suggest?

- [ ] Use an Aurora Global Database
- [ ] Use an RDS database
- [ ] Use Aurora Serverless
- [ ] Run Aurora on EC2, and write a script to shut down the EC2 instance at night

**Question 29:**
You need to store long-term backups for your Aurora database for disaster recovery and audit purposes. What do you recommend?

- [ ] Enable Automated Backups
- [ ] Perform On Demand Backups
- [ ] Use Aurora Database Cloning

**Question 30:**
You have 100 EC2 instances connected to your RDS database and you see that upon a maintenance of the database, all your applications take a lot of time to reconnect to RDS, due to poor application logic. How do you improve this?

- [ ] Fix all the applications
- [ ] Disable Multi-AZ
- [ ] Enable Multi-AZ
- [ ] Use an RDS Proxy
