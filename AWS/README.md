# AWS Projects Documentation

## 1. Configure an EC2 instance with a custom bootstrap script

**Answer:**  
“I launched an EC2 instance using a suitable AMI and instance type based on the application requirements. I used the user data section to add a custom bootstrap script, which automatically installed packages, configured environment variables, and started services when the instance launched. After launching, I verified that the script ran correctly by checking system logs and ensuring services were running. This approach ensured the EC2 instance was fully configured and ready-to-use immediately, reducing manual intervention and standardizing deployment across environments.”

**Why:**  
Automates server setup, reduces human error, and ensures consistency.

---

## 2. Set up an S3 bucket with CloudFront as CDN

**Answer:**  
“I created an S3 bucket to store static content like HTML, CSS, JS, and images, and configured bucket policies to restrict public access. Then, I set up a CloudFront distribution with the S3 bucket as the origin, enabling low-latency content delivery globally. I configured caching policies for improved performance and set up an Origin Access Identity (OAI) so CloudFront could securely access the bucket without exposing it publicly. After deployment, I tested content delivery from different regions to verify latency improvements and security.”

**Why:**  
Ensures fast global delivery of static assets while securing storage.

**What is Origin Access Identity (OAI)?**  
- OAI is a special CloudFront feature that allows you to securely serve private content from an S3 bucket.  
- Without OAI, users could bypass CloudFront and access the S3 bucket directly if it’s public.  
- OAI creates a special CloudFront identity and configures the S3 bucket policy to allow only this identity to read objects.  
- CloudFront fetches content from S3 on behalf of the user.  
- Users can only access S3 content through CloudFront, ensuring security and enabling caching and HTTPS.

**Example Scenario:**  
- You have an S3 bucket with website images.  
- You don’t want anyone to access the images directly via S3 URLs.  
- By enabling OAI, attaching it to CloudFront, and updating the S3 bucket policy, users can only access images through the CloudFront URL.

---

## 3. Configure an Application Load Balancer with SSL (ACM certificate)

**Answer:**  
“I deployed an Application Load Balancer (ALB) in public subnets to distribute traffic across multiple backend EC2 instances. I requested an SSL certificate from AWS Certificate Manager (ACM) and attached it to the ALB to enable HTTPS traffic termination. I configured listeners and target groups, including health checks to monitor instance availability. Finally, I tested the ALB to ensure proper traffic routing and SSL functionality.”

**Why:**  
Provides secure, highly available access and offloads SSL termination from the instances.

---

## 4. Launch an Auto Scaling Group tied to CloudWatch alarms

**Answer:**  
“I created an Auto Scaling Group (ASG) using a launch template that included EC2 instance configuration and AMI. I set up CloudWatch alarms to monitor CPU and memory metrics. Using these alarms, I configured scaling policies to automatically increase or decrease the number of instances based on real-time load. I tested the scaling by simulating high traffic, ensuring the ASG added instances during spikes and removed them when demand decreased.”

**Why:**  
Ensures high availability, performance, and cost optimization.

---

## 5. Create a VPC with 2 private and 2 public subnets + NAT gateway

**Answer:**  
“I designed a custom VPC with a CIDR block suitable for the application. I created 2 public and 2 private subnets across two Availability Zones for high availability. I attached an Internet Gateway to public subnets for external access and deployed a NAT Gateway to enable instances in private subnets to access the internet securely. Route tables were configured appropriately, and connectivity was verified by testing access from both public and private subnets.”

**Why:**  
Provides secure, highly available network segmentation for application components.

---

## 6. Set up Route53 with a custom domain pointing to ALB

**Answer:**  
“I used Route53 to register or manage a custom domain and created a hosted zone for it. I added an alias A record pointing to the ALB DNS name to route traffic from the domain to the load balancer. I tested domain resolution and ensured traffic reached the ALB and backend instances correctly. I also set up TTL values and routing policies for reliability and fast DNS resolution.”

**Why:**  
Enables user-friendly domain routing with integration into AWS infrastructure and high availability.

---

## 7. Deploy a serverless app using Lambda + API Gateway

**Answer:**  
“I developed AWS Lambda functions to handle backend operations. Using API Gateway, I created REST endpoints and integrated them with the Lambda functions. I configured IAM roles to allow API Gateway to invoke Lambda securely and tested the endpoints using Postman to ensure proper functionality. Optional stage variables were used for environment separation. This architecture eliminated server management, and Lambda automatically scaled based on incoming requests.”

**Why:**  
Provides a scalable, cost-efficient, serverless solution for application endpoints.

---

## 8. Configure SSM Session Manager to access EC2 without SSH keys

**Answer:**  
“I installed or verified the SSM Agent on EC2 instances and attached IAM roles with AmazonSSMManagedInstanceCore permissions. Using SSM Session Manager, I connected to instances directly from the AWS console or CLI without SSH keys. All session activity was logged to CloudTrail, providing auditability and centralized access management. This approach also removed the need to manage SSH key rotation manually.”

**Why:**  
Improves security, compliance, and ease of management for accessing instances.

---

## 9. Enable CloudTrail for auditing and send logs to S3

**Answer:**  
“I enabled AWS CloudTrail across the AWS account to capture all API activity. I configured a secure S3 bucket to store CloudTrail logs with proper encryption and lifecycle policies. Optionally, I integrated CloudTrail with CloudWatch Logs for real-time monitoring and alerts. This setup allowed auditing of all changes, detecting suspicious activities, and supporting compliance and forensic investigations.”

**Why:**  
Ensures visibility, compliance, and security auditing across the AWS environment.

---

## 10. Implement AWS Backup for EC2 and RDS resources

**Answer:**  
“I created AWS Backup plans and assigned resource selections for EC2 volumes (EBS) and RDS instances. I defined backup schedules, retention periods, and lifecycle rules to meet RTO/RPO requirements. After backups completed, I tested restoration to ensure recoverability. Centralized backup management allowed consistent, automated protection of critical workloads across multiple AWS services.”

**Why:**  
Provides reliable, automated backup and disaster recovery for mission-critical resources.
