# EBS and EC2 Questions

**Question 3:**
You can use an AMI in N.Virginia Region `us-east-1` to launch an EC2 instance in any AWS Region.

- [ ] True
- [ ] False

**Question 4:**
Which of the following EBS volume types can be used as boot volumes when you create EC2 instances?

- [ ] `gp2`, `gp3`, `st1`, `sc1`
- [ ] `gp2`, `gp3`, `io1`, `io2`
- [ ] `io1`, `io2`, `st1`, `sc1`

**Question 5:**
What is EBS Multi-Attach?

- [ ] Attach the same EBS volume to multiple EC2 instances in multiple AZs
- [ ] Attach multiple EBS volumes in the same AZ to the same EC2 instance
- [ ] Attach the same EBS volume to multiple EC2 instances in the same AZ
- [ ] Attach multiple EBS volumes in multiple AZs to the same EC2 instance

**Question 6:**
You would like to encrypt an unencrypted EBS volume attached to your EC2 instance. What should you do?

- [ ] Create an EBS snapshot of your EBS volume. Copy the snapshot and tick the option to encrypt the copied snapshot. Then, use the encrypted snapshot to create a new EBS volume
- [ ] Select your EBS volume, choose **Edit Attributes**, then tick the **Encrypt using KMS** option
- [ ] Create a new encrypted EBS volume, then copy data from your unencrypted EBS volume to the new EBS volume.
- [ ] Submit a request to AWS Support to encrypt your EBS volume

**Question 9:**
You are running a high-performance database that requires an IOPS of 310,000 for its underlying storage. What do you recommend?

- [ ] Use an EBS `gp2` drive
- [ ] Use an EBS `io1` drive
- [ ] Use an EC2 Instance Store
- [ ] Use an EBS `io2` Block Express drive
