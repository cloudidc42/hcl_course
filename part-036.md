# Part 036: AWS Provider Setup & Authentication
# AWS Provider และวิธีการ Authentication

## Steps 351-360: การ Setup และ Configure AWS Provider อย่างถูกต้องและปลอดภัย

---

## Step 351: AWS Provider Versions

### required_providers Configuration

```hcl
# versions.tf

terraform {
  required_version = ">= 1.5.0"

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}
```

### ตรวจสอบ Provider Version

```bash
# ดู provider versions ที่ใช้อยู่
terraform providers

# Output:
# Providers required by configuration:
# . provider[registry.terraform.io/hashicorp/aws] ~> 5.0
#
# Providers required by state:
# provider[registry.terraform.io/hashicorp/aws] 5.20.0

# ดู version ที่ถูก lock
cat .terraform.lock.hcl | grep "version ="
```

### AWS Provider Changelog สำคัญ

```
AWS Provider Major Versions:
- v5.x (Current): Provider framework ใหม่, รองรับ AWS SDK v2
- v4.x (Previous): AWS SDK v1, deprecated
- v3.x (Legacy): อย่าใช้ใน projects ใหม่

สิ่งที่เปลี่ยนใน v5:
- S3 bucket resources แยกออกเป็น sub-resources
- IAM policy documents เปลี่ยน format
- default_tags มีประสิทธิภาพดีขึ้น
```

---

## Step 352: Authentication Methods (เรียงตาม Best Practice)

### ภาพรวม Authentication Methods

```
Authentication Priority (Terraform AWS Provider):

1. ✅✅✅ IAM Roles (Best - ไม่มี static credentials!)
   - EC2 Instance Profile
   - ECS Task Role
   - Lambda Execution Role
   - EKS Pod Identity
   - OIDC Federation (GitHub Actions, GitLab CI)

2. ✅✅  Environment Variables (Good - ไม่มีใน code)
   - AWS_ACCESS_KEY_ID
   - AWS_SECRET_ACCESS_KEY
   - AWS_SESSION_TOKEN (สำหรับ assumed role)

3. ✅    AWS Config File (~/.aws/credentials)
   - [default] profile
   - Named profiles

4. ✅    AWS SSO
   - aws sso login
   - Named profiles

5. ⚠️   Hardcoded in provider (NEVER DO!)
   - access_key = "..."
   - secret_key = "..."
```

---

## Step 353: IAM Roles (Best Method)

### 1. EC2 Instance Profile

```hcl
# ✅ EC2: สร้าง IAM Role และ Instance Profile สำหรับ Terraform

# IAM Role
resource "aws_iam_role" "terraform" {
  name = "terraform-execution-role"
  
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
    Name    = "terraform-execution-role"
    Purpose = "terraform-infrastructure-management"
  }
}

# IAM Policy สำหรับ Terraform operations
resource "aws_iam_policy" "terraform" {
  name        = "terraform-execution-policy"
  description = "Permissions for Terraform to manage infrastructure"

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Action = [
          "ec2:*",
          "s3:*",
          "rds:*",
          "iam:CreateRole",
          "iam:DeleteRole",
          "iam:GetRole",
          "iam:ListRolePolicies",
          "iam:AttachRolePolicy",
          "iam:DetachRolePolicy",
          "iam:PassRole",
        ]
        Resource = "*"
      }
    ]
  })
}

# Attach Policy to Role
resource "aws_iam_role_policy_attachment" "terraform" {
  role       = aws_iam_role.terraform.name
  policy_arn = aws_iam_policy.terraform.arn
}

# Instance Profile
resource "aws_iam_instance_profile" "terraform" {
  name = "terraform-instance-profile"
  role = aws_iam_role.terraform.name
}

# EC2 ที่รัน Terraform
resource "aws_instance" "terraform_runner" {
  ami                  = data.aws_ami.amazon_linux.id
  instance_type        = "t3.medium"
  iam_instance_profile = aws_iam_instance_profile.terraform.name
  
  # ✅ ไม่ต้องมี credentials ใน config - ใช้ instance profile
  
  tags = {
    Name = "terraform-runner"
  }
}
```

```hcl
# provider.tf - บน EC2 ไม่ต้องระบุ credentials
provider "aws" {
  region = "ap-southeast-1"
  # ✅ credentials จะอ่านจาก instance metadata service (IMDS) อัตโนมัติ
}
```

### 2. ECS Task Role

```hcl
# ECS Task ที่รัน Terraform

resource "aws_iam_role" "ecs_terraform" {
  name = "ecs-terraform-task-role"
  
  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Action    = "sts:AssumeRole"
      Effect    = "Allow"
      Principal = {
        Service = "ecs-tasks.amazonaws.com"
      }
    }]
  })
}

resource "aws_ecs_task_definition" "terraform" {
  family                   = "terraform-task"
  network_mode             = "awsvpc"
  requires_compatibilities = ["FARGATE"]
  cpu                      = "512"
  memory                   = "1024"
  task_role_arn            = aws_iam_role.ecs_terraform.arn
  execution_role_arn       = aws_iam_role.ecs_execution.arn

  container_definitions = jsonencode([{
    name  = "terraform"
    image = "hashicorp/terraform:1.6.4"
    
    # ✅ Task role จะถูก inject โดยอัตโนมัติ
    # ไม่ต้องใส่ credentials ใน container definition
    
    logConfiguration = {
      logDriver = "awslogs"
      options = {
        awslogs-group         = "/ecs/terraform"
        awslogs-region        = "ap-southeast-1"
        awslogs-stream-prefix = "terraform"
      }
    }
  }])
}
```

### 3. OIDC สำหรับ GitHub Actions (Recommended for CI/CD)

```hcl
# setup_github_oidc.tf - Setup OIDC trust กับ GitHub

# OIDC Provider สำหรับ GitHub
resource "aws_iam_openid_connect_provider" "github" {
  url = "https://token.actions.githubusercontent.com"
  
  client_id_list = ["sts.amazonaws.com"]
  
  # Thumbprint ของ GitHub's OIDC provider
  thumbprint_list = [
    "6938fd4d98bab03faadb97b34396831e3780aea1",
    "1c58a3a8518e8759bf075b76b750d4f2df264fcd"
  ]
}

# IAM Role สำหรับ GitHub Actions
resource "aws_iam_role" "github_actions" {
  name = "github-actions-terraform-role"
  
  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Action    = "sts:AssumeRoleWithWebIdentity"
      Effect    = "Allow"
      Principal = {
        Federated = aws_iam_openid_connect_provider.github.arn
      }
      Condition = {
        StringLike = {
          # ✅ จำกัดเฉพาะ repository และ branch ที่อนุญาต
          "token.actions.githubusercontent.com:sub" = [
            "repo:myorg/infrastructure:ref:refs/heads/main",
            "repo:myorg/infrastructure:environment:production"
          ]
        }
        StringEquals = {
          "token.actions.githubusercontent.com:aud" = "sts.amazonaws.com"
        }
      }
    }]
  })
}

# Policy สำหรับ GitHub Actions
resource "aws_iam_role_policy_attachment" "github_actions" {
  role       = aws_iam_role.github_actions.name
  policy_arn = "arn:aws:iam::aws:policy/AdministratorAccess"
  # ⚠️ ใน production ควรใช้ least-privilege policy แทน
}
```

```yaml
# .github/workflows/terraform.yml - ใช้ OIDC
name: Terraform

on:
  push:
    branches: [main]

permissions:
  id-token: write   # ← จำเป็นสำหรับ OIDC
  contents: read

jobs:
  terraform:
    runs-on: ubuntu-latest
    
    steps:
      - uses: actions/checkout@v4

      - name: Configure AWS Credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          # ✅ ไม่มี secrets! ใช้ OIDC แทน
          role-to-assume: arn:aws:iam::123456789012:role/github-actions-terraform-role
          aws-region: ap-southeast-1

      - name: Setup Terraform
        uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: "1.6.4"

      - name: Terraform Apply
        run: |
          terraform init
          terraform apply -auto-approve
```

---

## Step 354: Environment Variables

### การ Set Environment Variables

```bash
# ─── Linux/macOS ────────────────────────────────────────────

# Set สำหรับ session ปัจจุบัน
export AWS_ACCESS_KEY_ID="AKIAIOSFODNN7EXAMPLE"
export AWS_SECRET_ACCESS_KEY="wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY"
export AWS_DEFAULT_REGION="ap-southeast-1"

# Set สำหรับ assumed role (ชั่วคราว)
export AWS_SESSION_TOKEN="AQoDYXdzEJr..."

# ✅ ดีกว่า: ใช้ direnv เพื่อ auto-load per project
# .envrc (ไม่ commit เข้า git!)
export AWS_PROFILE=development
export AWS_DEFAULT_REGION=ap-southeast-1

# ─── Windows PowerShell ───────────────────────────────────

$env:AWS_ACCESS_KEY_ID = "AKIAIOSFODNN7EXAMPLE"
$env:AWS_SECRET_ACCESS_KEY = "wJalrXUtnFEMI/K7MDENG/..."
$env:AWS_DEFAULT_REGION = "ap-southeast-1"

# ─── Terraform Variables (TF_VAR_*) ──────────────────────

# เพิ่ม TF_VAR_ prefix สำหรับ terraform variables
export TF_VAR_db_password="super-secret-password"
export TF_VAR_environment="production"
```

### Environment Variables สำหรับ CI/CD

```yaml
# GitHub Actions - ใช้ Secrets
env:
  AWS_DEFAULT_REGION: ap-southeast-1

steps:
  - name: Set AWS Credentials
    env:
      AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
      AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
    run: terraform apply
```

---

## Step 355: AWS Profile (~/.aws/credentials)

### การ Setup AWS Profiles

```ini
# ~/.aws/credentials

[default]
aws_access_key_id = AKIAIOSFODNN7EXAMPLE
aws_secret_access_key = wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY

[development]
aws_access_key_id = AKIAI44QH8DHBEXAMPLE
aws_secret_access_key = je7MtGbClwBF/2Zp9Utk/h3yCo8nvbEXAMPLEKEY

[staging]
aws_access_key_id = AKIAI44QH8DHBEXAMPLE2
aws_secret_access_key = je7MtGbClwBF/2Zp9Utk/h3yCo8nvbEXAMPLEKEY2

[production]
# ✅ ใช้ assume role แทน long-term credentials
role_arn = arn:aws:iam::987654321098:role/ProductionTerraformRole
source_profile = default
```

```ini
# ~/.aws/config

[default]
region = ap-southeast-1
output = json

[profile development]
region = ap-southeast-1
output = json

[profile production]
region = ap-southeast-1
role_arn = arn:aws:iam::987654321098:role/ProductionTerraformRole
source_profile = default
mfa_serial = arn:aws:iam::123456789012:mfa/myuser
```

### ใช้ Profile ใน Provider

```hcl
# provider.tf

provider "aws" {
  region  = var.aws_region
  profile = var.aws_profile  # ระบุ profile
}

# variables.tf
variable "aws_profile" {
  description = "AWS CLI profile"
  type        = string
  default     = "default"
}
```

```bash
# ใช้ผ่าน environment variable
export AWS_PROFILE=production
terraform apply

# หรือผ่าน -var
terraform apply -var="aws_profile=production"
```

---

## Step 356: Web Identity Token (OIDC)

### GitLab CI กับ OIDC

```hcl
# setup_gitlab_oidc.tf

resource "aws_iam_openid_connect_provider" "gitlab" {
  url             = "https://gitlab.com"
  client_id_list  = ["https://gitlab.com"]
  thumbprint_list = ["b3dd7606d2b5a8b4a13771dbecc9ee1cecafa38a"]
}

resource "aws_iam_role" "gitlab_ci" {
  name = "gitlab-ci-terraform-role"
  
  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Action    = "sts:AssumeRoleWithWebIdentity"
      Effect    = "Allow"
      Principal = {
        Federated = aws_iam_openid_connect_provider.gitlab.arn
      }
      Condition = {
        StringLike = {
          "gitlab.com:sub" = "project_path:mygroup/infrastructure:ref_type:branch:ref:main"
        }
      }
    }]
  })
}
```

```yaml
# .gitlab-ci.yml
terraform:
  stage: deploy
  image: hashicorp/terraform:1.6.4
  
  id_tokens:
    AWS_TOKEN:
      aud: https://gitlab.com
  
  before_script:
    - export $(printf "AWS_ACCESS_KEY_ID=%s AWS_SECRET_ACCESS_KEY=%s AWS_SESSION_TOKEN=%s"
        $(aws sts assume-role-with-web-identity
          --role-arn arn:aws:iam::123456789012:role/gitlab-ci-terraform-role
          --role-session-name GitLabCI
          --web-identity-token $AWS_TOKEN
          --query 'Credentials.[AccessKeyId,SecretAccessKey,SessionToken]'
          --output text))
  
  script:
    - terraform init
    - terraform apply -auto-approve
```

---

## Step 357: Assume Role

### Cross-account Role Assumption

```hcl
# ─── Single Account: Assume Role ─────────────────────────

provider "aws" {
  region = "ap-southeast-1"

  assume_role {
    role_arn     = "arn:aws:iam::123456789012:role/TerraformRole"
    session_name = "TerraformSession"
    duration     = "1h"
    
    # Optional: External ID (สำหรับ third-party access)
    external_id = var.assume_role_external_id
    
    # Optional: Tags ที่ติดมากับ assumed role
    tags = {
      ManagedBy = "terraform"
    }
  }
}

# ─── Multiple Accounts: Multi-provider Setup ─────────────

# Account 1: Shared Services
provider "aws" {
  alias  = "shared"
  region = "ap-southeast-1"
  
  assume_role {
    role_arn = "arn:aws:iam::111111111111:role/TerraformRole"
  }
}

# Account 2: Development
provider "aws" {
  alias  = "dev"
  region = "ap-southeast-1"
  
  assume_role {
    role_arn = "arn:aws:iam::222222222222:role/TerraformRole"
  }
}

# Account 3: Production
provider "aws" {
  alias  = "prod"
  region = "ap-southeast-1"
  
  assume_role {
    role_arn = "arn:aws:iam::333333333333:role/TerraformRole"
  }
}

# ใช้ provider alias ใน resources
resource "aws_s3_bucket" "shared_artifacts" {
  provider = aws.shared  # ← ระบุ provider
  bucket   = "shared-artifacts-bucket"
}

resource "aws_s3_bucket" "dev_data" {
  provider = aws.dev
  bucket   = "dev-data-bucket"
}
```

---

## Step 358: Provider Configuration

### default_tags Block

```hcl
# provider.tf - Complete provider configuration

provider "aws" {
  region  = var.aws_region
  profile = var.aws_profile

  # ✅ default_tags: tags ที่ apply กับ resources ทุกอัน
  default_tags {
    tags = {
      Project     = var.project_name
      Environment = var.environment
      ManagedBy   = "terraform"
      Repository  = var.git_repository
      Owner       = var.team_name
    }
  }

  # Retry configuration
  retry_mode                  = "standard"
  max_retries                 = 3
  
  # Custom endpoints (สำหรับ LocalStack หรือ custom endpoints)
  # endpoints {
  #   s3  = "http://localhost:4566"
  #   ec2 = "http://localhost:4566"
  # }

  # Ignore tags ที่มีคนเพิ่ม manual (เช่น security scans)
  ignore_tags {
    key_prefixes = ["security-scan-"]
    keys         = ["LastScanned"]
  }
}
```

### Multiple Regions

```hcl
# provider.tf - Multi-region setup

# Primary region
provider "aws" {
  region = "ap-southeast-1"
  
  default_tags {
    tags = {
      Project   = var.project_name
      ManagedBy = "terraform"
    }
  }
}

# US East (สำหรับ global services เช่น Route53, CloudFront, IAM)
provider "aws" {
  alias  = "us_east_1"
  region = "us-east-1"
}

# EU West (สำหรับ GDPR compliance)
provider "aws" {
  alias  = "eu_west_1"
  region = "eu-west-1"
}

# ใช้ CloudFront (ต้องการ us-east-1 สำหรับ ACM)
resource "aws_acm_certificate" "cert" {
  provider = aws.us_east_1  # ← CloudFront ต้องการ certificate ใน us-east-1
  
  domain_name       = var.domain_name
  validation_method = "DNS"
}
```

---

## Step 359: Multiple AWS Accounts Setup

### Hub-and-Spoke Architecture

```hcl
# multi_account_setup.tf

# ─── Variables ────────────────────────────────────────────

variable "account_ids" {
  description = "AWS Account IDs"
  type        = map(string)
  default = {
    management  = "111111111111"
    shared      = "222222222222"
    development = "333333333333"
    staging     = "444444444444"
    production  = "555555555555"
  }
}

# ─── Providers ─────────────────────────────────────────────

provider "aws" {
  alias  = "management"
  region = "ap-southeast-1"
  assume_role {
    role_arn = "arn:aws:iam::${var.account_ids.management}:role/TerraformRole"
  }
}

provider "aws" {
  alias  = "development"
  region = "ap-southeast-1"
  assume_role {
    role_arn = "arn:aws:iam::${var.account_ids.development}:role/TerraformRole"
  }
}

provider "aws" {
  alias  = "production"
  region = "ap-southeast-1"
  assume_role {
    role_arn = "arn:aws:iam::${var.account_ids.production}:role/TerraformRole"
  }
}

# ─── Resources across accounts ────────────────────────────

# SNS Topic ใน Management account
resource "aws_sns_topic" "alerts" {
  provider = aws.management
  name     = "infrastructure-alerts"
}

# VPC ใน Development account
module "dev_vpc" {
  source = "./modules/vpc"
  
  providers = {
    aws = aws.development
  }
  
  project_name = var.project_name
  environment  = "development"
  vpc_cidr     = "10.1.0.0/16"
  
  availability_zones   = ["ap-southeast-1a", "ap-southeast-1b"]
  public_subnet_cidrs  = ["10.1.1.0/24", "10.1.2.0/24"]
  private_subnet_cidrs = ["10.1.10.0/24", "10.1.11.0/24"]
}

# VPC ใน Production account
module "prod_vpc" {
  source = "./modules/vpc"
  
  providers = {
    aws = aws.production
  }
  
  project_name = var.project_name
  environment  = "production"
  vpc_cidr     = "10.2.0.0/16"
  
  availability_zones   = ["ap-southeast-1a", "ap-southeast-1b", "ap-southeast-1c"]
  public_subnet_cidrs  = ["10.2.1.0/24", "10.2.2.0/24", "10.2.3.0/24"]
  private_subnet_cidrs = ["10.2.10.0/24", "10.2.11.0/24", "10.2.12.0/24"]
}
```

---

## Step 360: Testing กับ LocalStack

### LocalStack Setup

```hcl
# localstack.tf - Provider config สำหรับ LocalStack

provider "aws" {
  region                      = "us-east-1"
  access_key                  = "test"        # LocalStack ไม่ต้องการ credentials จริง
  secret_key                  = "test"
  skip_credentials_validation = true
  skip_metadata_api_check     = true
  skip_requesting_account_id  = true
  s3_use_path_style           = true          # จำเป็นสำหรับ LocalStack S3

  endpoints {
    s3             = "http://localhost:4566"
    ec2            = "http://localhost:4566"
    iam            = "http://localhost:4566"
    sts            = "http://localhost:4566"
    lambda         = "http://localhost:4566"
    dynamodb       = "http://localhost:4566"
    rds            = "http://localhost:4566"
    sqs            = "http://localhost:4566"
    sns            = "http://localhost:4566"
    secretsmanager = "http://localhost:4566"
    ssm            = "http://localhost:4566"
  }
}
```

```yaml
# docker-compose.yml สำหรับ LocalStack

version: '3.8'
services:
  localstack:
    image: localstack/localstack:latest
    ports:
      - "4566:4566"       # LocalStack Edge Port
      - "4510-4559:4510-4559"  # External services
    environment:
      - SERVICES=s3,ec2,iam,sts,lambda,dynamodb,rds,sqs,sns,ssm,secretsmanager
      - DEBUG=1
      - LAMBDA_EXECUTOR=docker
      - DOCKER_HOST=unix:///var/run/docker.sock
      - AWS_DEFAULT_REGION=us-east-1
    volumes:
      - "/var/run/docker.sock:/var/run/docker.sock"
      - "./localstack-data:/var/lib/localstack"
```

```bash
#!/bin/bash
# scripts/test_localstack.sh

# Start LocalStack
docker-compose up -d localstack

# Wait for LocalStack
echo "Waiting for LocalStack..."
until curl -s http://localhost:4566/_localstack/health | grep -q '"s3": "available"'; do
  sleep 2
done
echo "LocalStack is ready!"

# Run Terraform against LocalStack
export TF_VAR_environment="local"
terraform init
terraform apply -auto-approve \
  -var-file="localstack.tfvars"

echo "✅ LocalStack tests passed!"

# Cleanup
terraform destroy -auto-approve
docker-compose down
```

### IAM Permissions สำหรับ Common Operations

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "TerraformVPC",
      "Effect": "Allow",
      "Action": [
        "ec2:CreateVpc",
        "ec2:DeleteVpc",
        "ec2:DescribeVpcs",
        "ec2:ModifyVpcAttribute",
        "ec2:CreateSubnet",
        "ec2:DeleteSubnet",
        "ec2:DescribeSubnets",
        "ec2:CreateInternetGateway",
        "ec2:DeleteInternetGateway",
        "ec2:AttachInternetGateway",
        "ec2:DetachInternetGateway",
        "ec2:CreateRouteTable",
        "ec2:DeleteRouteTable",
        "ec2:CreateRoute",
        "ec2:DeleteRoute",
        "ec2:AssociateRouteTable",
        "ec2:DisassociateRouteTable",
        "ec2:CreateNatGateway",
        "ec2:DeleteNatGateway",
        "ec2:AllocateAddress",
        "ec2:ReleaseAddress",
        "ec2:DescribeAddresses",
        "ec2:DescribeAvailabilityZones",
        "ec2:DescribeInternetGateways",
        "ec2:DescribeRouteTables",
        "ec2:DescribeNatGateways",
        "ec2:CreateTags",
        "ec2:DeleteTags",
        "ec2:DescribeTags"
      ],
      "Resource": "*"
    },
    {
      "Sid": "TerraformEC2",
      "Effect": "Allow",
      "Action": [
        "ec2:RunInstances",
        "ec2:TerminateInstances",
        "ec2:StopInstances",
        "ec2:StartInstances",
        "ec2:DescribeInstances",
        "ec2:DescribeInstanceTypes",
        "ec2:DescribeImages",
        "ec2:DescribeKeyPairs",
        "ec2:CreateKeyPair",
        "ec2:DeleteKeyPair",
        "ec2:CreateSecurityGroup",
        "ec2:DeleteSecurityGroup",
        "ec2:DescribeSecurityGroups",
        "ec2:AuthorizeSecurityGroupIngress",
        "ec2:RevokeSecurityGroupIngress",
        "ec2:AuthorizeSecurityGroupEgress",
        "ec2:RevokeSecurityGroupEgress",
        "ec2:DescribeVolumes",
        "ec2:CreateVolume",
        "ec2:DeleteVolume",
        "ec2:AttachVolume",
        "ec2:DetachVolume",
        "ec2:DescribeInstanceAttribute",
        "ec2:ModifyInstanceAttribute",
        "ec2:DescribeInstanceStatus"
      ],
      "Resource": "*"
    },
    {
      "Sid": "TerraformS3",
      "Effect": "Allow",
      "Action": [
        "s3:CreateBucket",
        "s3:DeleteBucket",
        "s3:ListBucket",
        "s3:GetBucketLocation",
        "s3:GetBucketPolicy",
        "s3:PutBucketPolicy",
        "s3:DeleteBucketPolicy",
        "s3:GetBucketVersioning",
        "s3:PutBucketVersioning",
        "s3:GetEncryptionConfiguration",
        "s3:PutEncryptionConfiguration",
        "s3:GetBucketPublicAccessBlock",
        "s3:PutBucketPublicAccessBlock",
        "s3:GetBucketTagging",
        "s3:PutBucketTagging",
        "s3:GetObject",
        "s3:PutObject",
        "s3:DeleteObject",
        "s3:GetBucketLogging",
        "s3:PutBucketLogging",
        "s3:GetBucketOwnershipControls",
        "s3:PutBucketOwnershipControls"
      ],
      "Resource": "*"
    },
    {
      "Sid": "TerraformIAM",
      "Effect": "Allow",
      "Action": [
        "iam:CreateRole",
        "iam:DeleteRole",
        "iam:GetRole",
        "iam:ListRoles",
        "iam:UpdateRole",
        "iam:CreatePolicy",
        "iam:DeletePolicy",
        "iam:GetPolicy",
        "iam:ListPolicies",
        "iam:CreatePolicyVersion",
        "iam:DeletePolicyVersion",
        "iam:GetPolicyVersion",
        "iam:ListPolicyVersions",
        "iam:AttachRolePolicy",
        "iam:DetachRolePolicy",
        "iam:ListAttachedRolePolicies",
        "iam:CreateInstanceProfile",
        "iam:DeleteInstanceProfile",
        "iam:GetInstanceProfile",
        "iam:AddRoleToInstanceProfile",
        "iam:RemoveRoleFromInstanceProfile",
        "iam:PassRole",
        "iam:TagRole",
        "iam:UntagRole",
        "iam:ListRoleTags",
        "iam:CreateOpenIDConnectProvider",
        "iam:DeleteOpenIDConnectProvider",
        "iam:GetOpenIDConnectProvider"
      ],
      "Resource": "*"
    },
    {
      "Sid": "TerraformState",
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:PutObject",
        "s3:DeleteObject",
        "s3:ListBucket"
      ],
      "Resource": [
        "arn:aws:s3:::terraform-state-bucket",
        "arn:aws:s3:::terraform-state-bucket/*"
      ]
    },
    {
      "Sid": "TerraformStateLock",
      "Effect": "Allow",
      "Action": [
        "dynamodb:GetItem",
        "dynamodb:PutItem",
        "dynamodb:DeleteItem",
        "dynamodb:DescribeTable"
      ],
      "Resource": "arn:aws:dynamodb:*:*:table/terraform-state-lock"
    }
  ]
}
```

---

## สรุป: AWS Provider Authentication Best Practices

### Security Hierarchy

```
Most Secure ──────────────────────── Least Secure
     │                                     │
     ▼                                     ▼
IAM Role     Environment    AWS Profile   Hardcoded
(OIDC/IME)   Variables     (~/.aws)       ❌ NEVER
     │              │            │
 ✅✅✅           ✅✅          ✅
```

### Environment-specific Provider Config

```hcl
# ✅ สำหรับ local development
provider "aws" {
  region  = "ap-southeast-1"
  profile = "development"   # ← local profile
}

# ✅ สำหรับ CI/CD (GitHub Actions / GitLab CI)
provider "aws" {
  region = "ap-southeast-1"
  # credentials จาก OIDC role หรือ environment variables
}

# ✅ สำหรับ production infrastructure
provider "aws" {
  region = "ap-southeast-1"
  assume_role {
    role_arn = "arn:aws:iam::PRODUCTION_ACCOUNT_ID:role/TerraformRole"
  }
}
```

---

*จบ Part 036: AWS Provider Setup & Authentication*

*ต่อไป: Part 037 - AWS EC2 Instances*
