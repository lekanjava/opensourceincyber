---
layout: page
title: Terraform Security Bootcamp
permalink: /tutorials/terraform-bootcamp/
---

# Terraform Security Bootcamp

This bootcamp provides a comprehensive guide to implementing secure infrastructure as code using Terraform, with a focus on security best practices for cloud environments.

## Prerequisites

Before you begin, ensure you have the following:

- Terraform (version 1.0.0 or later) installed
- AWS CLI, Azure CLI, or GCP CLI configured (depending on your cloud provider)
- Basic understanding of infrastructure as code concepts
- A code editor (VS Code recommended with HashiCorp Terraform extension)
- Git for version control

## Module 1: Terraform Fundamentals with Security Focus

### Setting Up a Secure Terraform Environment

```bash
# Create a dedicated directory for your Terraform projects
mkdir -p ~/terraform-projects/secure-infrastructure
cd ~/terraform-projects/secure-infrastructure

# Initialize version control
git init

# Create a .gitignore file to prevent sensitive information from being committed
cat > .gitignore << EOF
# Local .terraform directories
**/.terraform/*

# .tfstate files
*.tfstate
*.tfstate.*

# Crash log files
crash.log

# Sensitive variables files
*.tfvars
*.tfvars.json

# Override files
override.tf
override.tf.json
*_override.tf
*_override.tf.json

# CLI configuration files
.terraformrc
terraform.rc
EOF

# Create a basic directory structure
mkdir -p {modules,environments/{dev,prod},policies}
```

### Creating Your First Secure Terraform Configuration

Create a `main.tf` file with secure defaults:

```hcl
# Configure the AWS Provider with secure defaults
provider "aws" {
  region = "us-west-2"
  
  default_tags {
    tags = {
      Environment = "dev"
      Project     = "terraform-bootcamp"
      ManagedBy   = "terraform"
    }
  }
}

# Configure S3 backend for secure state management
terraform {
  backend "s3" {
    bucket         = "your-terraform-state-bucket"
    key            = "dev/terraform.tfstate"
    region         = "us-west-2"
    encrypt        = true
    dynamodb_table = "terraform-state-lock"
  }
}

# Create a secure VPC
module "vpc" {
  source = "terraform-aws-modules/vpc/aws"
  version = "3.14.0"

  name = "secure-vpc"
  cidr = "10.0.0.0/16"

  azs             = ["us-west-2a", "us-west-2b", "us-west-2c"]
  private_subnets = ["10.0.1.0/24", "10.0.2.0/24", "10.0.3.0/24"]
  public_subnets  = ["10.0.101.0/24", "10.0.102.0/24", "10.0.103.0/24"]

  enable_nat_gateway = true
  single_nat_gateway = true

  # Enable VPC Flow Logs
  enable_flow_log                      = true
  flow_log_destination_type            = "cloud-watch-logs"
  flow_log_destination_arn             = aws_cloudwatch_log_group.flow_log.arn
  flow_log_traffic_type                = "ALL"
  flow_log_max_aggregation_interval    = 60
}

# CloudWatch Log Group for VPC Flow Logs
resource "aws_cloudwatch_log_group" "flow_log" {
  name = "vpc-flow-logs"
  retention_in_days = 30
  
  # Enable encryption for logs
  kms_key_id = aws_kms_key.log_encryption.arn
}

# KMS Key for log encryption
resource "aws_kms_key" "log_encryption" {
  description             = "KMS key for VPC flow logs encryption"
  deletion_window_in_days = 10
  enable_key_rotation     = true
}
```

## Module 2: Implementing Security Controls

### Network Security with Terraform

Create a `network_security.tf` file:

```hcl
# Create a security group with secure defaults
resource "aws_security_group" "web_server" {
  name        = "web-server-sg"
  description = "Security group for web servers"
  vpc_id      = module.vpc.vpc_id

  # No ingress rules defined by default - least privilege principle
  
  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
    description = "Allow all outbound traffic"
  }

  lifecycle {
    create_before_destroy = true
  }

  tags = {
    Name = "web-server-sg"
  }
}

# Add specific ingress rules based on requirements
resource "aws_security_group_rule" "web_https_ingress" {
  type              = "ingress"
  from_port         = 443
  to_port           = 443
  protocol          = "tcp"
  cidr_blocks       = ["0.0.0.0/0"]
  description       = "Allow HTTPS inbound traffic"
  security_group_id = aws_security_group.web_server.id
}

# Create Network ACLs for additional security
resource "aws_network_acl" "private" {
  vpc_id     = module.vpc.vpc_id
  subnet_ids = module.vpc.private_subnets

  egress {
    protocol   = "tcp"
    rule_no    = 100
    action     = "allow"
    cidr_block = "0.0.0.0/0"
    from_port  = 443
    to_port    = 443
  }

  egress {
    protocol   = "tcp"
    rule_no    = 110
    action     = "allow"
    cidr_block = "0.0.0.0/0"
    from_port  = 80
    to_port    = 80
  }

  ingress {
    protocol   = "tcp"
    rule_no    = 100
    action     = "allow"
    cidr_block = "10.0.0.0/16"
    from_port  = 1024
    to_port    = 65535
  }

  tags = {
    Name = "private-nacl"
  }
}
```

### Identity and Access Management

Create an `iam.tf` file:

```hcl
# Create a custom IAM policy with least privilege
resource "aws_iam_policy" "s3_read_only" {
  name        = "s3-read-only"
  description = "Policy that grants read-only access to specific S3 buckets"

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Action = [
          "s3:GetObject",
          "s3:ListBucket",
        ]
        Effect   = "Allow"
        Resource = [
          "arn:aws:s3:::example-bucket",
          "arn:aws:s3:::example-bucket/*"
        ]
      },
    ]
  })
}

# Create an IAM role with the policy attached
resource "aws_iam_role" "app_role" {
  name = "app-role"
  
  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Action = "sts:AssumeRole"
        Effect = "Allow"
        Principal = {
          Service = "ec2.amazonaws.com"
        }
      },
    ]
  })

  # Force MFA for security-sensitive actions
  inline_policy {
    name = "require-mfa"
    policy = jsonencode({
      Version = "2012-10-17"
      Statement = [
        {
          Action = [
            "iam:ChangePassword",
            "iam:CreateAccessKey",
            "iam:UpdateAccessKey",
          ]
          Effect = "Deny"
          Resource = "*"
          Condition = {
            BoolIfExists = {
              "aws:MultiFactorAuthPresent": "false"
            }
          }
        }
      ]
    })
  }
}

# Attach the policy to the role
resource "aws_iam_role_policy_attachment" "app_s3_access" {
  role       = aws_iam_role.app_role.name
  policy_arn = aws_iam_policy.s3_read_only.arn
}
```

## Module 3: Implementing Security Scanning for Terraform

### Setting Up Terraform Security Scanning

```bash
# Install tfsec
brew install tfsec  # macOS
# OR
curl -s https://raw.githubusercontent.com/aquasecurity/tfsec/master/scripts/install_linux.sh | bash  # Linux

# Install checkov
pip install checkov

# Create a pre-commit hook for security scanning
cat > .git/hooks/pre-commit << EOF
#!/bin/bash
set -e

echo "Running Terraform security scanners..."

# Format check
terraform fmt -check -recursive

# Run tfsec
tfsec .

# Run checkov
checkov -d .

echo "Security scanning completed successfully!"
exit 0
EOF

chmod +x .git/hooks/pre-commit
```

### Creating a CI/CD Pipeline for Terraform Security

Create a `.github/workflows/terraform-security.yml` file:

```yaml
name: Terraform Security Scan

on:
  push:
    branches: [ main ]
  pull_request:
    branches: [ main ]

jobs:
  security-scan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v2
      
      - name: Setup Terraform
        uses: hashicorp/setup-terraform@v1
        with:
          terraform_version: 1.0.0
      
      - name: Terraform Format
        run: terraform fmt -check -recursive
      
      - name: tfsec
        uses: aquasecurity/tfsec-action@v1.0.0
      
      - name: Checkov
        uses: bridgecrewio/checkov-action@master
        with:
          directory: .
          framework: terraform
```

## Module 4: Advanced Security Patterns

### Implementing Secure Modules

Create a reusable security module in `modules/security-baseline/main.tf`:

```hcl
variable "environment" {
  description = "Environment name (e.g., dev, prod)"
  type        = string
}

variable "vpc_id" {
  description = "VPC ID where security resources will be created"
  type        = string
}

# AWS Config for continuous compliance monitoring
resource "aws_config_configuration_recorder" "this" {
  name     = "security-baseline-recorder"
  role_arn = aws_iam_role.config_role.arn
  
  recording_group {
    all_supported                 = true
    include_global_resource_types = true
  }
}

resource "aws_config_delivery_channel" "this" {
  name           = "security-baseline-delivery-channel"
  s3_bucket_name = aws_s3_bucket.config_logs.bucket
  
  snapshot_delivery_properties {
    delivery_frequency = "Six_Hours"
  }
  
  depends_on = [aws_config_configuration_recorder.this]
}

resource "aws_config_configuration_recorder_status" "this" {
  name       = aws_config_configuration_recorder.this.name
  is_enabled = true
  
  depends_on = [aws_config_delivery_channel.this]
}

# GuardDuty for threat detection
resource "aws_guardduty_detector" "this" {
  enable = true
  
  finding_publishing_frequency = "SIX_HOURS"
}

# Security Hub for security posture management
resource "aws_securityhub_account" "this" {}

# Enable security standards
resource "aws_securityhub_standards_subscription" "cis" {
  standards_arn = "arn:aws:securityhub:::ruleset/cis-aws-foundations-benchmark/v/1.2.0"
  
  depends_on = [aws_securityhub_account.this]
}

# S3 bucket for logs with secure configuration
resource "aws_s3_bucket" "config_logs" {
  bucket = "config-logs-${var.environment}-${data.aws_caller_identity.current.account_id}"
  
  lifecycle {
    prevent_destroy = true
  }
}

resource "aws_s3_bucket_server_side_encryption_configuration" "config_logs" {
  bucket = aws_s3_bucket.config_logs.bucket
  
  rule {
    apply_server_side_encryption_by_default {
      sse_algorithm = "AES256"
    }
  }
}

resource "aws_s3_bucket_public_access_block" "config_logs" {
  bucket = aws_s3_bucket.config_logs.id
  
  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}

# IAM role for AWS Config
resource "aws_iam_role" "config_role" {
  name = "aws-config-role-${var.environment}"
  
  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Action = "sts:AssumeRole"
        Effect = "Allow"
        Principal = {
          Service = "config.amazonaws.com"
        }
      },
    ]
  })
}

resource "aws_iam_role_policy_attachment" "config_policy" {
  role       = aws_iam_role.config_role.name
  policy_arn = "arn:aws:iam::aws:policy/service-role/AWS_ConfigRole"
}

data "aws_caller_identity" "current" {}

output "guardduty_detector_id" {
  value = aws_guardduty_detector.this.id
}

output "config_recorder_id" {
  value = aws_config_configuration_recorder.this.id
}
```

## Module 5: Putting It All Together

### Creating a Complete Secure Infrastructure

Update your `main.tf` to use the security baseline module:

```hcl
module "security_baseline" {
  source = "./modules/security-baseline"
  
  environment = "dev"
  vpc_id      = module.vpc.vpc_id
}

# Create a secure EC2 instance
resource "aws_instance" "web" {
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t3.micro"
  subnet_id     = module.vpc.private_subnets[0]
  
  # Use the IAM role created earlier
  iam_instance_profile = aws_iam_instance_profile.app_profile.name
  
  # Use the security group created earlier
  vpc_security_group_ids = [aws_security_group.web_server.id]
  
  # Enable detailed monitoring
  monitoring = true
  
  # Use encrypted EBS volumes
  root_block_device {
    encrypted   = true
    volume_size = 20
  }
  
  metadata_options {
    http_endpoint               = "enabled"
    http_tokens                 = "required"  # Require IMDSv2
    http_put_response_hop_limit = 1
  }
  
  tags = {
    Name = "secure-web-server"
  }
}

# Create an IAM instance profile
resource "aws_iam_instance_profile" "app_profile" {
  name = "app-profile"
  role = aws_iam_role.app_role.name
}
```

## Best Practices for Secure Terraform Usage

1. **Version Control**
   - Store Terraform code in version control systems
   - Use branch protection rules to prevent direct commits to main branches
   - Require peer reviews for all changes

2. **State Management**
   - Use remote state with encryption
   - Implement state locking to prevent concurrent modifications
   - Restrict access to state files

3. **Secret Management**
   - Never store secrets in Terraform code
   - Use secret management tools like HashiCorp Vault or AWS Secrets Manager
   - Inject secrets at runtime rather than storing them in configuration

4. **Least Privilege**
   - Use minimal permissions for Terraform execution
   - Create separate service accounts for different environments
   - Regularly audit and rotate credentials

5. **Continuous Validation**
   - Implement automated security scanning in CI/CD pipelines
   - Validate changes before applying them
   - Use policy as code to enforce security standards

## Conclusion

By following this bootcamp, you've learned how to implement secure infrastructure as code using Terraform. You've created a foundation for secure cloud deployments with proper network segmentation, identity management, and security monitoring. Continue to build on this foundation by exploring advanced security patterns and keeping up with evolving best practices in infrastructure security.

## Additional Resources

- [Terraform Security Best Practices](https://www.terraform.io/docs/cloud/guides/recommended-practices/index.html)
- [AWS Security Reference Architecture](https://docs.aws.amazon.com/prescriptive-guidance/latest/security-reference-architecture/welcome.html)
- [tfsec Documentation](https://aquasecurity.github.io/tfsec/latest/)
- [Checkov Documentation](https://www.checkov.io/)