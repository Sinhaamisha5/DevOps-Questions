# Terraform AWS Infrastructure

A comprehensive collection of Terraform configurations for deploying and managing AWS infrastructure, demonstrating best practices for production-ready deployments.

## 📋 Table of Contents

- [Overview](#overview)
- [Prerequisites](#prerequisites)
- [Installation](#installation)
- [Configuration Examples](#configuration-examples)
  - [1. EC2 Module](#1-ec2-module)
  - [2. S3 Bucket with Encryption](#2-s3-bucket-with-encryption)
  - [3. Workspace Management](#3-workspace-management)
  - [4. IAM Roles and Policies](#4-iam-roles-and-policies)
  - [5. Application Load Balancer](#5-application-load-balancer)
  - [6. Dynamic EC2 Provisioning](#6-dynamic-ec2-provisioning)
  - [7. Multi-AZ RDS Instance](#7-multi-az-rds-instance)
  - [8. Secrets Management](#8-secrets-management)
  - [9. EKS Cluster Deployment](#9-eks-cluster-deployment)
  - [10. CI/CD with GitHub Actions](#10-cicd-with-github-actions)
- [Best Practices](#best-practices)
- [Interview Tips](#interview-tips)
- [Contributing](#contributing)
- [License](#license)

## Overview

This repository contains production-ready Terraform configurations demonstrating:

- **Modularity:** Reusable modules for common infrastructure components
- **Security:** Encryption, IAM policies, and secrets management
- **High Availability:** Multi-AZ deployments, load balancing, and auto-scaling
- **Automation:** CI/CD integration with GitHub Actions
- **Cost Optimization:** Lifecycle policies and resource tagging

| Configuration | Purpose | Key Features |
|--------------|---------|--------------|
| EC2 Module | Reusable compute instances | Variables, tags, modularity |
| S3 Bucket | Secure object storage | Encryption, lifecycle rules |
| Workspaces | Environment isolation | Dev, QA, Prod separation |
| IAM Roles | Access management | Least privilege, assume roles |
| Load Balancer | Traffic distribution | Health checks, HA |
| Dynamic EC2 | Scalable instances | Count argument, indexing |
| Multi-AZ RDS | Database HA | Automatic failover |
| Secrets Manager | Credential security | Encrypted storage, rotation |
| EKS Cluster | Kubernetes orchestration | Node groups, auto-scaling |
| GitHub Actions | Automated deployments | CI/CD pipeline |

## Prerequisites

- **Terraform:** Version 1.5.0 or higher
- **AWS CLI:** Configured with appropriate credentials
- **AWS Account:** With sufficient permissions for resource creation
- **Git:** For version control
- **kubectl:** (Optional) For EKS cluster management

### AWS Credentials Setup

```bash
# Configure AWS CLI
aws configure

# Or export credentials
export AWS_ACCESS_KEY_ID="your-access-key"
export AWS_SECRET_ACCESS_KEY="your-secret-key"
export AWS_DEFAULT_REGION="us-east-1"
```

## Installation

1. **Clone the repository:**
```bash
git clone https://github.com/yourusername/terraform-aws-infrastructure.git
cd terraform-aws-infrastructure
```

2. **Initialize Terraform:**
```bash
terraform init
```

3. **Validate configuration:**
```bash
terraform validate
```

4. **Plan deployment:**
```bash
terraform plan
```

5. **Apply configuration:**
```bash
terraform apply
```

## Configuration Examples

### 1. EC2 Module

Create reusable EC2 instances with configurable parameters.

**Module Structure:**

```
modules/
└── ec2/
    ├── main.tf
    ├── variables.tf
    └── outputs.tf
```

**File:** `modules/ec2/main.tf`

```hcl
resource "aws_instance" "this" {
  ami           = var.ami
  instance_type = var.instance_type
  tags          = var.tags
}
```

**File:** `modules/ec2/variables.tf`

```hcl
variable "ami" {
  description = "AMI ID for the EC2 instance"
  type        = string
}

variable "instance_type" {
  description = "Instance type for the EC2 instance"
  type        = string
  default     = "t3.micro"
}

variable "tags" {
  description = "Tags to apply to the EC2 instance"
  type        = map(string)
  default     = {}
}
```

**Usage:**

```hcl
module "web_server" {
  source        = "./modules/ec2"
  ami           = "ami-123456"
  instance_type = "t3.micro"
  tags = {
    Name        = "web-server"
    Environment = "production"
    Owner       = "devops-team"
  }
}
```

**Key Concepts:**
- Modular design for reusability
- Input variables for flexibility
- DRY (Don't Repeat Yourself) principle
- Can output instance IDs or public IPs for reference

---

### 2. S3 Bucket with Encryption

Deploy secure S3 bucket with encryption and lifecycle management.

**File:** `s3_bucket.tf`

```hcl
resource "aws_s3_bucket" "my_bucket" {
  bucket = "my-encrypted-bucket"
}

resource "aws_s3_bucket_server_side_encryption_configuration" "sse" {
  bucket = aws_s3_bucket.my_bucket.id
  
  rule {
    apply_server_side_encryption_by_default {
      sse_algorithm = "AES256"
      # Or use KMS: sse_algorithm = "aws:kms"
      # kms_master_key_id = aws_kms_key.mykey.arn
    }
  }
}

resource "aws_s3_bucket_lifecycle_configuration" "lifecycle" {
  bucket = aws_s3_bucket.my_bucket.id
  
  rule {
    id      = "expire-old-objects"
    status  = "Enabled"
    
    expiration {
      days = 30
    }
  }
  
  rule {
    id      = "transition-to-glacier"
    status  = "Enabled"
    
    transition {
      days          = 90
      storage_class = "GLACIER"
    }
  }
}
```

**Features:**
- **Security:** AES256 server-side encryption
- **Cost Optimization:** Automatic expiration after 30 days
- **Archival:** Transition to Glacier for long-term storage
- **Compliance:** Meets data protection requirements

**Optional additions:**
```hcl
# Enable versioning
resource "aws_s3_bucket_versioning" "versioning" {
  bucket = aws_s3_bucket.my_bucket.id
  versioning_configuration {
    status = "Enabled"
  }
}

# Block public access
resource "aws_s3_bucket_public_access_block" "block" {
  bucket = aws_s3_bucket.my_bucket.id

  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}
```

---

### 3. Workspace Management

Create isolated environments using Terraform workspaces.

**Creating Workspaces:**

```bash
# Create workspaces for different environments
terraform workspace new dev
terraform workspace new qa
terraform workspace new prod

# List all workspaces
terraform workspace list

# Switch between workspaces
terraform workspace select dev

# Show current workspace
terraform workspace show
```

**Using Workspace in Configuration:**

```hcl
# Reference current workspace
output "current_workspace" {
  value = terraform.workspace
}

# Dynamic resource naming based on workspace
resource "aws_instance" "app" {
  ami           = var.ami
  instance_type = terraform.workspace == "prod" ? "t3.large" : "t3.micro"
  
  tags = {
    Name        = "app-${terraform.workspace}"
    Environment = terraform.workspace
  }
}

# Environment-specific variables
locals {
  env_config = {
    dev = {
      instance_count = 1
      instance_type  = "t3.micro"
    }
    qa = {
      instance_count = 2
      instance_type  = "t3.small"
    }
    prod = {
      instance_count = 3
      instance_type  = "t3.large"
    }
  }
}

resource "aws_instance" "web" {
  count         = local.env_config[terraform.workspace].instance_count
  instance_type = local.env_config[terraform.workspace].instance_type
  ami           = var.ami
  
  tags = {
    Name = "web-${terraform.workspace}-${count.index + 1}"
  }
}
```

**Benefits:**
- Isolated state files per environment
- Same configuration for multiple environments
- Reduced code duplication
- Easy environment switching

---

### 4. IAM Roles and Policies

Configure AWS IAM roles following least privilege principle.

**File:** `iam_roles.tf`

```hcl
# IAM Role for EC2
resource "aws_iam_role" "ec2_role" {
  name = "ec2-s3-access-role"
  
  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Action    = "sts:AssumeRole"
      Effect    = "Allow"
      Principal = {
        Service = "ec2.amazonaws.com"
      }
    }]
  })
  
  tags = {
    Name = "EC2 S3 Access Role"
  }
}

# Attach managed policy
resource "aws_iam_role_policy_attachment" "s3_full_access" {
  role       = aws_iam_role.ec2_role.name
  policy_arn = "arn:aws:iam::aws:policy/AmazonS3FullAccess"
}

# Custom inline policy
resource "aws_iam_role_policy" "custom_policy" {
  name = "custom-s3-policy"
  role = aws_iam_role.ec2_role.id
  
  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect = "Allow"
      Action = [
        "s3:GetObject",
        "s3:PutObject"
      ]
      Resource = "arn:aws:s3:::my-bucket/*"
    }]
  })
}

# Instance profile for EC2
resource "aws_iam_instance_profile" "ec2_profile" {
  name = "ec2-profile"
  role = aws_iam_role.ec2_role.name
}
```

**Usage with EC2:**

```hcl
resource "aws_instance" "app" {
  ami                  = var.ami
  instance_type        = "t3.micro"
  iam_instance_profile = aws_iam_instance_profile.ec2_profile.name
  
  tags = {
    Name = "app-server"
  }
}
```

**Security Best Practices:**
- Use least privilege principle
- Prefer managed policies for common tasks
- Use inline policies for specific requirements
- Avoid hardcoding credentials
- Enable MFA for sensitive operations

---

### 5. Application Load Balancer

Deploy highly available load balancer with health checks.

**File:** `load_balancer.tf`

```hcl
# Application Load Balancer
resource "aws_lb" "app" {
  name               = "app-load-balancer"
  internal           = false
  load_balancer_type = "application"
  security_groups    = [aws_security_group.lb_sg.id]
  subnets            = ["subnet-123abc", "subnet-456def"]
  
  enable_deletion_protection = false
  
  tags = {
    Name        = "app-lb"
    Environment = "production"
  }
}

# Target Group
resource "aws_lb_target_group" "app_tg" {
  name     = "app-target-group"
  port     = 80
  protocol = "HTTP"
  vpc_id   = "vpc-123456"
  
  health_check {
    enabled             = true
    path                = "/health"
    port                = "traffic-port"
    protocol            = "HTTP"
    interval            = 30
    timeout             = 5
    healthy_threshold   = 3
    unhealthy_threshold = 3
    matcher             = "200"
  }
  
  tags = {
    Name = "app-tg"
  }
}

# Listener
resource "aws_lb_listener" "http" {
  load_balancer_arn = aws_lb.app.arn
  port              = "80"
  protocol          = "HTTP"
  
  default_action {
    type             = "forward"
    target_group_arn = aws_lb_target_group.app_tg.arn
  }
}

# HTTPS Listener (optional)
resource "aws_lb_listener" "https" {
  load_balancer_arn = aws_lb.app.arn
  port              = "443"
  protocol          = "HTTPS"
  ssl_policy        = "ELBSecurityPolicy-2016-08"
  certificate_arn   = "arn:aws:acm:region:account:certificate/id"
  
  default_action {
    type             = "forward"
    target_group_arn = aws_lb_target_group.app_tg.arn
  }
}

# Target Group Attachment
resource "aws_lb_target_group_attachment" "app" {
  count            = length(aws_instance.app)
  target_group_arn = aws_lb_target_group.app_tg.arn
  target_id        = aws_instance.app[count.index].id
  port             = 80
}
```

**Features:**
- High availability across multiple AZs
- Health checks ensure only healthy instances receive traffic
- Automatic removal of unhealthy instances
- Support for HTTP and HTTPS
- SSL/TLS termination

---

### 6. Dynamic EC2 Provisioning

Use `count` to create multiple instances dynamically.

**File:** `dynamic_ec2.tf`

```hcl
# Create multiple instances
resource "aws_instance" "web" {
  count         = 3
  ami           = "ami-123456"
  instance_type = "t3.micro"
  
  tags = {
    Name = "web-server-${count.index + 1}"
    Index = count.index
  }
}

# Output instance details
output "instance_ids" {
  value = aws_instance.web[*].id
}

output "instance_public_ips" {
  value = aws_instance.web[*].public_ip
}
```

**Advanced Example with for_each:**

```hcl
# Define instances as a map
variable "instances" {
  type = map(object({
    instance_type = string
    ami           = string
  }))
  default = {
    web = {
      instance_type = "t3.micro"
      ami           = "ami-123456"
    }
    api = {
      instance_type = "t3.small"
      ami           = "ami-789012"
    }
    db = {
      instance_type = "t3.medium"
      ami           = "ami-345678"
    }
  }
}

# Create instances using for_each
resource "aws_instance" "app" {
  for_each      = var.instances
  ami           = each.value.ami
  instance_type = each.value.instance_type
  
  tags = {
    Name = "${each.key}-server"
    Type = each.key
  }
}
```

**Benefits:**
- Easy scaling by changing count value
- Unique identifiers using count.index
- More efficient than duplicating resources
- `for_each` provides better control for complex scenarios

---

### 7. Multi-AZ RDS Instance

Deploy highly available RDS database with automatic failover.

**File:** `rds_instance.tf`

```hcl
resource "aws_db_instance" "main" {
  identifier           = "mydb-instance"
  allocated_storage    = 20
  storage_type         = "gp3"
  engine               = "mysql"
  engine_version       = "8.0"
  instance_class       = "db.t3.micro"
  db_name              = "myappdb"
  username             = "admin"
  password             = var.db_password  # Use variables for sensitive data
  
  # High Availability
  multi_az             = true
  
  # Backup Configuration
  backup_retention_period = 7
  backup_window          = "03:00-04:00"
  maintenance_window     = "mon:04:00-mon:05:00"
  
  # Network Configuration
  db_subnet_group_name   = aws_db_subnet_group.main.name
  vpc_security_group_ids = [aws_security_group.db_sg.id]
  publicly_accessible    = false
  
  # Storage
  storage_encrypted      = true
  
  # Deletion Protection
  deletion_protection    = true
  skip_final_snapshot    = false
  final_snapshot_identifier = "mydb-final-snapshot-${formatdate("YYYY-MM-DD-hhmm", timestamp())}"
  
  tags = {
    Name        = "main-database"
    Environment = "production"
  }
}

# DB Subnet Group
resource "aws_db_subnet_group" "main" {
  name       = "main-db-subnet-group"
  subnet_ids = ["subnet-123abc", "subnet-456def"]
  
  tags = {
    Name = "Main DB Subnet Group"
  }
}

# Security Group for RDS
resource "aws_security_group" "db_sg" {
  name        = "rds-security-group"
  description = "Security group for RDS instance"
  vpc_id      = var.vpc_id
  
  ingress {
    from_port   = 3306
    to_port     = 3306
    protocol    = "tcp"
    cidr_blocks = ["10.0.0.0/16"]  # Only allow from VPC
  }
  
  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}

# Output
output "rds_endpoint" {
  value = aws_db_instance.main.endpoint
}
```

**Features:**
- Multi-AZ deployment for high availability
- Automatic failover to standby replica
- Encrypted storage for data security
- Automated backups with retention
- Maintenance windows for updates
- Network isolation within VPC

---

### 8. Secrets Management

Securely manage sensitive data using AWS Secrets Manager.

**File:** `secrets.tf`

```hcl
# Create secret
resource "aws_secretsmanager_secret" "db_password" {
  name                    = "prod/db/password"
  description             = "Database password for production"
  recovery_window_in_days = 7
  
  tags = {
    Environment = "production"
    ManagedBy   = "terraform"
  }
}

# Store secret value
resource "aws_secretsmanager_secret_version" "db_password" {
  secret_id = aws_secretsmanager_secret.db_password.id
  secret_string = jsonencode({
    username = "admin"
    password = var.db_password
    host     = aws_db_instance.main.endpoint
    port     = 3306
  })
}

# Retrieve secret in Terraform
data "aws_secretsmanager_secret_version" "db_creds" {
  secret_id = aws_secretsmanager_secret.db_password.id
}

locals {
  db_creds = jsondecode(data.aws_secretsmanager_secret_version.db_creds.secret_string)
}

# Use in RDS
resource "aws_db_instance" "app" {
  identifier = "app-database"
  username   = local.db_creds.username
  password   = local.db_creds.password
  # ... other configuration
}
```

**Automatic Rotation:**

```hcl
# Lambda function for rotation (simplified)
resource "aws_secretsmanager_secret_rotation" "db_password" {
  secret_id           = aws_secretsmanager_secret.db_password.id
  rotation_lambda_arn = aws_lambda_function.rotate_secret.arn
  
  rotation_rules {
    automatically_after_days = 30
  }
}
```

**Best Practices:**
- Never hardcode secrets in Terraform files
- Use AWS Secrets Manager or Parameter Store
- Enable automatic rotation
- Use IAM policies to restrict access
- Store secrets in JSON format for multiple values
- Use encryption at rest

---

### 9. EKS Cluster Deployment

Deploy Amazon EKS cluster with managed node groups.

**File:** `eks_cluster.tf`

```hcl
module "eks" {
  source  = "terraform-aws-modules/eks/aws"
  version = "~> 19.0"
  
  cluster_name    = "my-eks-cluster"
  cluster_version = "1.27"
  
  vpc_id     = var.vpc_id
  subnet_ids = ["subnet-123abc", "subnet-456def", "subnet-789ghi"]
  
  # Cluster endpoint access
  cluster_endpoint_public_access  = true
  cluster_endpoint_private_access = true
  
  # Enable IRSA (IAM Roles for Service Accounts)
  enable_irsa = true
  
  # Managed Node Groups
  eks_managed_node_groups = {
    general = {
      desired_size = 2
      min_size     = 1
      max_size     = 4
      
      instance_types = ["t3.medium"]
      capacity_type  = "ON_DEMAND"
      
      labels = {
        role = "general"
      }
      
      tags = {
        Environment = "production"
      }
    }
    
    spot = {
      desired_size = 1
      min_size     = 0
      max_size     = 3
      
      instance_types = ["t3.medium", "t3a.medium"]
      capacity_type  = "SPOT"
      
      labels = {
        role = "spot"
      }
    }
  }
  
  # Cluster security group rules
  cluster_security_group_additional_rules = {
    ingress_nodes_ephemeral_ports_tcp = {
      description                = "Nodes on ephemeral ports"
      protocol                   = "tcp"
      from_port                  = 1025
      to_port                    = 65535
      type                       = "ingress"
      source_node_security_group = true
    }
  }
  
  tags = {
    Environment = "production"
    ManagedBy   = "terraform"
  }
}

# Output kubeconfig
output "cluster_endpoint" {
  value = module.eks.cluster_endpoint
}

output "cluster_name" {
  value = module.eks.cluster_name
}

output "configure_kubectl" {
  description = "Configure kubectl: make sure you're logged in with the correct AWS profile and run the following command to update your kubeconfig"
  value       = "aws eks --region ${var.region} update-kubeconfig --name ${module.eks.cluster_name}"
}
```

**Configure kubectl:**

```bash
# Update kubeconfig
aws eks --region us-east-1 update-kubeconfig --name my-eks-cluster

# Verify connection
kubectl get nodes
```

**Features:**
- Managed node groups with auto-scaling
- IRSA for secure pod authentication
- Support for spot instances
- VPC integration for network security
- Modular design using community module

---

### 10. CI/CD with GitHub Actions

Automate Terraform deployments using GitHub Actions.

**File:** `.github/workflows/terraform.yml`

```yaml
name: Terraform CI/CD

on:
  push:
    branches: [ "main", "develop" ]
  pull_request:
    branches: [ "main" ]

env:
  AWS_REGION: us-east-1
  TF_VERSION: 1.5.0

jobs:
  terraform-plan:
    name: Terraform Plan
    runs-on: ubuntu-latest
    
    steps:
      - name: Checkout code
        uses: actions/checkout@v3
      
      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v2
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: ${{ env.AWS_REGION }}
      
      - name: Setup Terraform
        uses: hashicorp/setup-terraform@v2
        with:
          terraform_version: ${{ env.TF_VERSION }}
      
      - name: Terraform Format Check
        run: terraform fmt -check
      
      - name: Terraform Init
        run: terraform init
      
      - name: Terraform Validate
        run: terraform validate
      
      - name: Terraform Plan
        run: terraform plan -out=tfplan
      
      - name: Upload Plan
        uses: actions/upload-artifact@v3
        with:
          name: tfplan
          path: tfplan

  terraform-apply:
    name: Terraform Apply
    needs: terraform-plan
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main' && github.event_name == 'push'
    environment: production
    
    steps:
      - name: Checkout code
        uses: actions/checkout@v3
      
      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v2
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: ${{ env.AWS_REGION }}
      
      - name: Setup Terraform
        uses: hashicorp/setup-terraform@v2
        with:
          terraform_version: ${{ env.TF_VERSION }}
      
      - name: Terraform Init
        run: terraform init
      
      - name: Download Plan
        uses: actions/download-artifact@v3
        with:
          name: tfplan
      
      - name: Terraform Apply
        run: terraform apply -auto-approve tfplan
      
      - name: Notify Success
        if: success()
        run: echo "Terraform applied successfully!"
```

**Enhanced workflow with environment protection:**

```yaml
# Add manual approval for production
environment:
  name: production
  url: https://myapp.example.com
  approval-required: true
```

**Features:**
- Automated validation and planning on every push
- Manual approval for production deployments
- Secure credential management with GitHub Secrets
- Artifact storage for plan files
- Format checking and validation
- Separate jobs for plan and apply
- Environment-specific deployments

**GitHub Secrets Required:**
- `AWS_ACCESS_KEY_ID`
- `AWS_SECRET_ACCESS_KEY`

---

## Best Practices

### 1. State Management
```hcl
# Use remote backend
terraform {
  backend "s3" {
    bucket         = "my-terraform-state"
    key            = "prod/terraform.tfstate"
    region         = "us-east-1"
    encrypt        = true
    dynamodb_table = "terraform-locks"
  }
}
```

### 2. Variable Management
```hcl
# Use terraform.tfvars for environment-specific values
# terraform.tfvars
region          = "us-east-1"
environment     = "production"
instance_count  = 3
```

### 3. Resource Naming
```hcl
# Use consistent naming conventions
locals {
  name_prefix = "${var.project_name}-${var.environment}"
  
  common_tags = {
    Project     = var.project_name
    Environment = var.environment
    ManagedBy   = "Terraform"
    Owner       = var.owner
  }
}
```

### 4. Security
- Never commit sensitive data
- Use AWS Secrets Manager or Parameter Store
- Enable encryption at rest and in transit
- Implement least privilege IAM policies
- Use security groups with minimal access

### 5. Cost Optimization
- Use lifecycle policies for S3
- Implement auto-scaling
- Use spot instances where appropriate
- Tag resources for cost allocation
- Set up budget alerts

---

## Interview Tips

When discussing these Terraform configurations in interviews:

**1. EC2 Module:**
"I create reusable modules for EC2 that accept AMI, instance type, and tags as variables. This follows DRY principles and allows consistent deployment across environments. Modules can output instance IDs for reference by other resources."

**2. S3 with Encryption:**
"I implement S3 buckets with AES256 or KMS encryption for data at rest, and lifecycle policies to transition objects to Glacier or delete them after 30 days. This balances security with cost optimization."

**3. Workspaces:**
"I use Terraform workspaces to maintain separate state files for dev, QA, and prod environments. This allows the same configuration to deploy to multiple environments with environment-specific variables."

**4. IAM Roles:**
"I follow the principle of least privilege when creating IAM roles. EC2 instances assume roles to access AWS services without embedded credentials. I prefer managed policies for common tasks and inline policies for specific requirements."

**5. Load Balancer:**
"I deploy Application Load Balancers with target groups and health checks. The ALB distributes traffic only to healthy instances, ensuring high availability. Health check parameters ensure quick detection of failures."

**6. Dynamic Provisioning:**
"I use the count argument to dynamically create multiple resources. This makes scaling simple—just change the count value. Using count.index, I assign unique names and identifiers to each resource."

**7. Multi-AZ RDS:**
"I deploy RDS with Multi-AZ enabled for automatic failover. This ensures high availability with minimal downtime during failures. I configure backup retention, maintenance windows, and encryption for security and compliance."

**8. Secrets Management:**
"I never hardcode secrets. Instead, I use AWS Secrets Manager to store credentials securely. Terraform references these secrets, and automatic rotation can be enabled. This follows security best practices."

**9. EKS Cluster:**
"I use the community EKS module to deploy Kubernetes clusters with managed node groups. This includes IAM roles, security groups, and VPC configuration. IRSA enables secure pod authentication without long-term credentials."

**10. CI/CD Integration:**
"I integrate Terraform with GitHub Actions to automate deployments. On every push, the pipeline runs terraform plan. For production, manual approval is required before apply. AWS credentials are stored as GitHub Secrets."

---

## Contributing

Contributions are welcome! Please follow these guidelines:

1. Fork the repository
2. Create a feature branch
3. Make your changes
4. Run `terraform fmt` and `terraform validate`
5. Submit a pull request

---

## License

MIT License - feel free to use these configurations in your own projects.

---

**⚠️ Important Notes:**

- Always review `terraform plan` output before applying
- Use remote state with locking for team environments
- Implement proper backup and disaster recovery
- Test in non-production environments first
- Keep Terraform and provider versions updated
- Document custom modules thoroughly
- Use version constraints in module sources

**Security Reminder:** Never commit AWS credentials, private keys, or sensitive data to version control. Use `.gitignore` to exclude `*.tfvars`, `*.tfstate`, and `.terraform/` directories.