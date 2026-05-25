# ELB & Auto Scaling Group Questions

**Question 1:**
Your company is deploying a web application that routes `/api/*` requests to a backend microservice and `/web/*` requests to a frontend service. Both services are containerized on ECS. Which load balancer should you use?

- [ ] Network Load Balancer
- [x] Application Load Balancer
- [ ] Gateway Load Balancer
- [ ] Classic Load Balancer

**Question 2:**
A partner company requires you to provide a **static IP address** for whitelisting in their firewall. Your application still needs path-based HTTP routing. What architecture should you use?

- [ ] ALB with an Elastic IP attached
- [ ] CLB with an Elastic IP attached
- [x] NLB (with Elastic IP) in front of an ALB
- [ ] GWLB in front of an ALB

**Question 3:**
Your security team requires that **all traffic** entering your VPC be inspected by a fleet of third-party firewall appliances before reaching your application. The inspection must be transparent to both the client and the application. Which load balancer is designed for this?

- [ ] Application Load Balancer
- [ ] Network Load Balancer
- [x] Gateway Load Balancer
- [ ] Classic Load Balancer

**Question 4:**
Which tunneling protocol and port does the Gateway Load Balancer (GWLB) use to encapsulate traffic when sending it to virtual appliances?

- [ ] VXLAN on port 4789
- [ ] GENEVE on port 6081
- [ ] GRE on port 47
- [ ] IPsec on port 500

**Question 5:**
An application behind an ALB needs to know the client's real IP address for rate limiting. The backend only sees the ALB's private IP in the network packet. How should the application retrieve the client's actual IP?

- [ ] Read the `Host` header
- [ ] Read the `Origin` header
- [x] Read the `X-Forwarded-For` header
- [ ] Use VPC Flow Logs to determine the client IP

**Question 6:**
You have an ALB with cross-zone load balancing enabled (the default). There are 10 instances in AZ-a and 2 instances in AZ-b. How will traffic be distributed?

- [ ] AZ-a instances each get 5%, AZ-b instances each get 25%
- [ ] AZ-a gets 50% and AZ-b gets 50%, split within each AZ
- [x] Each of the 12 instances gets approximately 8.3% of the total traffic
- [ ] Traffic is only sent to AZ-a because it has more instances

**Question 7:**
Which of the following statements about cross-zone load balancing defaults is correct?

- [ ] ALB: disabled by default, free. NLB: enabled by default, paid.
- [x] ALB: enabled by default, free. NLB: disabled by default, paid.
- [ ] Both ALB and NLB: enabled by default, free.
- [ ] Both ALB and NLB: disabled by default, paid.

**Question 8:**
Your NLB serves 10 TB/day across 3 AZs. After enabling cross-zone load balancing, your AWS bill increased significantly. Why?

- [ ] NLB charges extra per hour when cross-zone is enabled
- [ ] NLB incurs inter-AZ data transfer charges when cross-zone sends traffic across AZ boundaries
- [ ] Enabling cross-zone requires upgrading to a larger NLB tier
- [ ] This is a billing error — cross-zone is free for all load balancers

**Question 9:**
Users of your web application lose their shopping cart contents when their requests are routed to a different instance behind the ALB. You need a quick fix. What should you enable?

- [ ] Cross-Zone Load Balancing
- [x] Sticky Sessions (Session Affinity)
- [ ] Connection Draining
- [ ] SSL Termination

**Question 10:**
What is the **best practice** long-term solution for the session-loss problem described in Question 9, rather than relying on sticky sessions?

- [ ] Increase the number of instances to reduce per-instance load
- [ ] Disable cross-zone load balancing
- [x] Externalize session state to ElastiCache (Redis) or DynamoDB
- [ ] Switch from ALB to NLB

**Question 11:**
Which of the following is a reserved ALB cookie name that your application must **never** use for custom cookies?

- [ ] `SESSION_ID`
- [ ] `MY_APP_COOKIE`
- [x] `AWSALBTG`
- [ ] `JSESSIONID`

**Question 12:**
How does NLB implement sticky sessions?

- [ ] Using the `AWSNLB` cookie
- [ ] Using the `AWSALB` cookie forwarded from ALB
- [x] Using source IP-based affinity (no cookie)
- [ ] NLB does not support sticky sessions

**Question 13:**
Your application is hosted behind an ALB and serves multiple domains (`api.example.com` and `shop.example.com`), each with its own SSL certificate. How can you serve both domains on the same ALB listener (port 443)?

- [ ] Upload a single wildcard certificate and use it for both
- [ ] Create two ALB listeners on port 443
- [x] Attach multiple certificates and use Server Name Indication (SNI)
- [ ] This is not possible — you need two separate ALBs

**Question 14:**
A Classic Load Balancer (CLB) currently serves HTTPS traffic for two different domains. Each domain requires a separate SSL certificate. What is the recommended solution?

- [ ] Enable SNI on the CLB
- [ ] Upload both certificates to the same CLB listener
- [x] Migrate to an ALB which supports SNI with multiple certificates
- [ ] Use a multi-domain (SAN) certificate and keep the CLB

**Question 15:**
Which AWS service provides **free, auto-renewing** SSL/TLS certificates for use with Elastic Load Balancers?

- [ ] AWS KMS
- [ ] AWS Certificate Manager (ACM)
- [ ] AWS Secrets Manager
- [ ] AWS IAM Certificate Store

**Question 16:**
Your compliance team requires that TLS encryption be maintained from the client all the way to the backend EC2 instances — not just to the load balancer. How do you achieve this with an ALB?

- [ ] Use a TCP listener on the ALB to pass TLS through
- [x] Configure the ALB to re-encrypt traffic (HTTPS listener + HTTPS target group) with a certificate on the backend
- [ ] This is not possible with ALB — use NLB
- [ ] Disable SSL termination on the ALB

**Question 17:**
You want the load balancer to forward encrypted TLS traffic directly to the backend without decrypting it (TLS passthrough). Which load balancer and listener type should you use?

- [ ] ALB with an HTTPS listener
- [ ] CLB with an SSL listener
- [x] NLB with a TCP listener
- [ ] GWLB with a TLS listener

**Question 18:**
During a rolling deployment, old instances are deregistered from the ALB but users report 502 errors. You discover that in-flight requests to the old instances are being dropped immediately. What should you configure?

- [ ] Enable Cross-Zone Load Balancing
- [ ] Increase the ALB idle timeout
- [x] Increase the Deregistration Delay (Connection Draining) from 0 to an appropriate value
- [ ] Enable Sticky Sessions

**Question 19:**
Your rolling deployment across 10 instances takes over 50 minutes, even though each instance handles only sub-second requests. What is the most likely cause and fix?

- [ ] Health checks are too slow — reduce the health check interval
- [ ] The default deregistration delay of 300 seconds is too long — reduce it to 30 seconds
- [ ] Cross-zone load balancing is disabled — enable it
- [ ] The instances are in a warm pool — remove the warm pool

**Question 20:**
What is the relationship between Connection Draining (CLB) and Deregistration Delay (ALB/NLB)?

- [ ] They are completely different features
- [x] They are the same feature with different names — CLB calls it Connection Draining, ALB/NLB call it Deregistration Delay
- [ ] Connection Draining is for inbound traffic, Deregistration Delay is for outbound traffic
- [ ] Connection Draining has a fixed 300-second timeout while Deregistration Delay is configurable

**Question 21:**
Your Auto Scaling Group has Minimum = 2, Desired = 4, Maximum = 8. The average CPU target tracking policy is set to 50%, and current CPU is 80%. What will the ASG do?

- [ ] Nothing — the ASG only reacts to CloudWatch Alarms
- [x] Launch additional instances (up to Maximum of 8) to bring average CPU closer to 50%
- [ ] Terminate instances because 80% exceeds the target
- [ ] Increase the instance type to a larger size

**Question 22:**
Your web application has predictable traffic patterns — high during business hours (9 AM–6 PM) and low overnight. Which scaling policy is most appropriate?

- [ ] Target Tracking Scaling
- [ ] Simple Scaling with CloudWatch Alarms
- [x] Scheduled Scaling
- [ ] Manual Scaling

**Question 23:**
You need to install custom monitoring agents and pull configuration from S3 on new EC2 instances **before** they start receiving traffic from the load balancer. Which ASG feature should you use?

- [ ] Instance Refresh
- [ ] Warm Pools
- [ ] Lifecycle Hooks (Pending:Wait state)
- [ ] User Data scripts only

**Question 24:**
You've updated your Launch Template with a new AMI. You want all existing instances in the ASG to be gradually replaced with instances using the new AMI, while maintaining at least 90% capacity throughout the process. What feature should you use?

- [ ] Terminate all instances and let the ASG re-launch them
- [ ] Create a new ASG and migrate traffic
- [ ] Instance Refresh with a Minimum Healthy Percentage of 90%
- [ ] Use Lifecycle Hooks to manually swap each instance

**Question 25:**
Your application takes 8 minutes to initialize after launch (loading caches, warming connections). Cold-launching new instances during scaling events causes delays before they can serve traffic. Which ASG feature can pre-initialize instances so they are ready in seconds?

- [ ] Predictive Scaling
- [ ] Increase the Health Check Grace Period
- [ ] Warm Pools
- [ ] Step Scaling with aggressive thresholds
