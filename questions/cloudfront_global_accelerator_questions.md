# CloudFront & Global Accelerator Questions

**Question 1:**
Your company hosts a static website in an S3 bucket in `us-east-1`. Users in Asia report slow page load times. You want to improve latency globally while keeping the S3 bucket private. Which architecture should you use?

- [ ] Enable S3 Transfer Acceleration
- [ ] Create S3 Cross-Region Replication to `ap-southeast-1`
- [x] Create a CloudFront distribution with the S3 bucket as origin, using Origin Access Control (OAC)
- [ ] Deploy an additional copy of the website to an S3 bucket in `ap-southeast-1`

**Question 2:**
You are setting up a CloudFront distribution with a custom domain `cdn.example.com`. Where must you provision the ACM SSL certificate?

- [ ] In the same region as the S3 origin bucket
- [ ] In any region — ACM certificates are global
- [ ] In `us-east-1` (N. Virginia)
- [x] In the region closest to your largest user base

**Question 3:**
Your application returns different HTML based on the user's `Accept-Language` header. After deploying CloudFront, all users see the same language regardless of their browser settings. What is the most likely cause?

- [ ] CloudFront does not support HTTP headers
- [x] The `Accept-Language` header is not included in the Cache Key — all users hit the same cached response
- [ ] The origin is ignoring the header
- [ ] CloudFront strips all custom headers by default

**Question 4:**
You need to forward the `User-Agent` header to your origin for analytics, but you do NOT want it to affect caching (since it would dramatically reduce your cache hit ratio). How should you configure this?

- [ ] Add `User-Agent` to the Cache Policy
- [ ] Disable caching entirely for this behavior
- [x] Add `User-Agent` to the Origin Request Policy (forwarded but not part of cache key)
- [ ] Use a CloudFront Function to log the header and remove it

**Question 5:**
A developer updated images in the S3 origin bucket, but CloudFront continues to serve the old versions. What are TWO effective solutions? (Select TWO)

- [x] Create a cache invalidation for `/images/*`
- [ ] Delete and recreate the CloudFront distribution
- [ ] Reduce the S3 bucket's replication lag
- [x] Use file versioning in the URL (e.g., `/images/logo_v2.png`) — best practice
- [ ] Restart the CloudFront edge locations

**Question 6:**
You want to implement simple URL rewrites (e.g., append `index.html` to directory URLs) at the CloudFront edge for millions of requests per second. The logic is lightweight — just string manipulation. Which compute option is most cost-effective?

- [ ] Lambda@Edge with a Viewer Request trigger
- [x] CloudFront Functions
- [ ] An ALB with path-based routing rules
- [ ] A Lambda function behind API Gateway

**Question 7:**
Your application needs to make an external API call at the edge to validate a JWT token and fetch user attributes before the request reaches the origin. The validation logic takes about 50ms and requires network access. Which edge compute option should you use?

- [ ] CloudFront Functions
- [x] Lambda@Edge
- [ ] AWS WAF custom rules
- [ ] Origin Request Policy with custom headers

**Question 8:**
Which of the following is TRUE about CloudFront Functions vs Lambda@Edge?

- [ ] CloudFront Functions can be triggered on Origin Request and Origin Response events
- [ ] Lambda@Edge runs at all 450+ edge locations
- [x] CloudFront Functions execute in less than 1ms and support only Viewer Request/Response triggers
- [ ] Lambda@Edge supports only JavaScript runtime

**Question 9:**
You want to restrict access to your CloudFront distribution so that users in certain countries cannot access the content. What CloudFront feature should you use?

- [ ] AWS WAF IP reputation lists
- [ ] Origin Access Control with country-based policies
- [x] CloudFront Geo Restriction (Geo-Blocking)
- [ ] Route 53 Geolocation Routing

**Question 10:**
A premium video streaming service wants to provide time-limited access to individual video files through CloudFront. Each user should get a unique URL that expires after 24 hours. What should you use?

- [ ] S3 Pre-Signed URLs
- [x] CloudFront Signed URLs
- [ ] CloudFront Signed Cookies
- [ ] AWS WAF time-based rules

**Question 11:**
The same streaming service now wants to give premium subscribers access to an entire library of videos (`/premium/*`) without generating individual signed URLs for each file. What should you use?

- [ ] CloudFront Signed URLs for each file
- [x] CloudFront Signed Cookies
- [ ] S3 bucket policy with IP restrictions
- [ ] CloudFront Functions to check a subscription header

**Question 12:**
What is the key difference between a CloudFront Signed URL and an S3 Pre-Signed URL?

- [ ] CloudFront Signed URLs are fre e; S3 Pre-Signed URLs are paid
- [ ] S3 Pre-Signed URLs benefit from edge caching; CloudFront Signed URLs do not
- [x] CloudFront Signed URLs serve content through the CDN (cached at edge); S3 Pre-Signed URLs go directly to S3 (no caching)
- [ ] S3 Pre-Signed URLs support IP restrictions; CloudFront Signed URLs do not

**Question 13:**
Your EC2 instances serve as the origin for a CloudFront distribution. Users report intermittent 502 errors. You discover CloudFront cannot connect to some EC2 instances. What is the most likely cause?

- [ ] The EC2 instances are in a private subnet
- [x] The EC2 security group does not allow inbound traffic from CloudFront edge location IP ranges
- [ ] The CloudFront distribution is in the wrong region
- [ ] The EC2 instances need an Elastic IP

**Question 14:**
You want to reduce CloudFront costs by limiting which edge locations are used. Your users are primarily in North America and Europe. Which Price Class should you select?

- [ ] Price Class All
- [ ] Price Class 200
- [x] Price Class 100 (North America + Europe only)
- [ ] Price Class 50

**Question 15:**
Your CloudFront distribution uses an S3 bucket as the primary origin. You want automatic failover to a secondary S3 bucket in another region if the primary returns 5xx errors. How should you configure this?

- [ ] Use Route 53 Failover routing to switch between two CloudFront distributions
- [x] Create a CloudFront Origin Group with the primary and secondary S3 buckets, configured to failover on 5xx errors
- [ ] Use Lambda@Edge to catch 5xx errors and redirect to the secondary bucket
- [ ] Enable S3 Cross-Region Replication and use a single origin

---

**Question 16:**
A global financial trading application requires **static IP addresses** that partners can whitelist in their firewalls, and sub-second failover if a region becomes unhealthy. The application uses TCP connections (not HTTP). Which service should you use?

- [ ] Amazon CloudFront
- [ ] Route 53 Failover Routing
- [ ] AWS Global Accelerator
- [ ] Network Load Balancer with Elastic IPs

**Question 17:**
How does AWS Global Accelerator improve application performance for global users?

- [ ] By caching content at 450+ edge locations
- [ ] By compressing HTTP responses at the edge
- [x] By routing user traffic through the AWS private backbone network instead of the public internet, reducing latency and jitter
- [ ] By deploying application replicas to edge locations

**Question 18:**
Which of the following is NOT a supported endpoint type for a standard AWS Global Accelerator?

- [ ] Application Load Balancer (ALB)
- [ ] Network Load Balancer (NLB)
- [ ] EC2 Instance
- [x] Amazon S3 Bucket

**Question 19:**
Your Global Accelerator has endpoint groups in `us-east-1` (Traffic Dial: 80%) and `eu-west-1` (Traffic Dial: 20%). You want to gradually shift all traffic to `eu-west-1` during a migration. What should you do?

- [ ] Create a new accelerator pointing only to `eu-west-1`
- [ ] Delete the `us-east-1` endpoint group
- [x] Gradually adjust the traffic dials — decrease `us-east-1` and increase `eu-west-1` until it reaches 0/100
- [ ] Use Route 53 Weighted Routing to shift traffic between the two Anycast IPs

**Question 20:**
What is the difference between the **Traffic Dial** and **Endpoint Weight** in Global Accelerator?

- [ ] They are the same feature with different names
- [ ] Traffic Dial controls traffic within a region; Endpoint Weight controls traffic between regions
- [x] Traffic Dial controls traffic between endpoint groups (regions); Endpoint Weight controls traffic between endpoints within the same group
- [ ] Traffic Dial is for TCP only; Endpoint Weight is for UDP only

**Question 21:**
All endpoints in the `us-east-1` endpoint group become unhealthy. What happens to traffic automatically?

- [ ] Traffic is dropped until `us-east-1` endpoints recover
- [ ] Global Accelerator returns a 503 error to clients
- [x] Traffic automatically fails over to the next closest healthy endpoint group (e.g., `eu-west-1`)
- [ ] The accelerator is automatically disabled

**Question 22:**
Why is Global Accelerator's failover faster than Route 53 DNS-based failover?

- [ ] Global Accelerator health checks are more frequent
- [ ] Global Accelerator uses faster health check protocols
- [x] Global Accelerator uses Anycast routing at the network level — failover doesn't depend on DNS TTL expiration and client DNS cache refresh
- [ ] Route 53 requires manual intervention to trigger failover

**Question 23:**
A multiplayer game company needs to route each player to a **specific** game server (EC2 instance) based on matchmaking results, while using Global Accelerator's Anycast IPs for low-latency entry. Which Global Accelerator feature should they use?

- [x] Standard Accelerator with endpoint weights set to route to specific instances
- [ ] Standard Accelerator with client affinity set to Source IP
- [ ] Custom Routing Accelerator
- [ ] Standard Accelerator with one endpoint group per game server

**Question 24:**
Which of the following scenarios is best suited for CloudFront rather than Global Accelerator?

- [ ] A VoIP application requiring low-latency UDP traffic
- [x] A media website serving cacheable images, CSS, and JavaScript to global users
- [ ] An application that requires two static Anycast IPs for firewall whitelisting
- [ ] A financial trading platform using persistent TCP connections

**Question 25:**
Your application is behind an ALB in `us-east-1`. You add the ALB as an endpoint in Global Accelerator. A security engineer asks whether the application will still see the real client IP address. What is the correct answer?

- [ ] No — the application will see Global Accelerator's IP as the source
- [x] Yes — Global Accelerator preserves the client IP for ALB endpoints (available in the `X-Forwarded-For` header)
- [ ] Only if you enable Proxy Protocol v2 on the ALB
- [ ] Only if you use an NLB instead of an ALB

**Question 26:**
Which statements about Global Accelerator security are TRUE? (Select TWO)

- [ ] AWS Shield Standard is automatically included at no extra cost
- [ ] AWS WAF can be directly attached to Global Accelerator
- [ ] AWS Shield Advanced can be enabled for enhanced DDoS protection
- [ ] Global Accelerator encrypts all traffic with TLS by default
- [ ] Security groups can be attached directly to the accelerator

**Question 27:**
Your company needs to optimize a REST API for global users. The API returns dynamic, non-cacheable responses. You also need static IP addresses for partner whitelisting. Which solution meets BOTH requirements?

- [ ] CloudFront with a custom domain and Elastic IP
- [ ] Route 53 Latency-Based Routing to regional ALBs
- [x] AWS Global Accelerator with ALB endpoints in multiple regions
- [ ] S3 Transfer Acceleration

**Question 28:**
You have a CloudFront distribution and also use Global Accelerator for different parts of your application. Which statement about their use of the AWS edge network is correct?

- [ ] They use completely separate edge infrastructure
- [ ] They both leverage the same AWS global edge network (400+ Points of Presence)
- [ ] Global Accelerator uses edge locations; CloudFront uses Regional Edge Caches only
- [ ] CloudFront has more edge locations than Global Accelerator

**Question 29:**
A solutions architect needs to host a static website with HTTPS, a custom domain `www.example.com`, and ensure the S3 bucket is not publicly accessible. Which combination of services is required? (Select FOUR)

- [x] S3 Bucket (private, Block Public Access enabled)
- [x] CloudFront Distribution with Origin Access Control (OAC)
- [ ] ACM Certificate in `us-east-1`
- [x] Global Accelerator with Anycast IPs
- [x] Route 53 Alias record pointing to the CloudFront distribution
- [ ] NAT Gateway for S3 access

**Question 30:**
A gaming company has servers in 3 AWS regions. Players worldwide need the lowest possible latency. The game uses UDP protocol. The company also requires DDoS protection. Which architecture should they use?

- [ ] CloudFront with UDP listener and AWS Shield
- [ ] Route 53 Latency-Based Routing with Shield Advanced on each region's NLB
- [x] AWS Global Accelerator (supports UDP) with Shield Advanced enabled
- [ ] ALB in each region with Route 53 Geoproximity Routing
