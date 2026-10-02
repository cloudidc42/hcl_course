# Part 019: Provider Configuration Deep Dive (เจาะลึก Provider Configuration)
## Steps 181-190: AWS Provider Configuration อย่างละเอียด

---

## บทนำ (Introduction)

บทนี้เจาะลึก AWS Provider configuration ทุก option รวมถึง authentication methods, multi-account setups, cross-account role assumption และ advanced configurations สำหรับ production environments

---

## Step 181: AWS Provider Full Configuration Options

### ตัวเลือกทั้งหมดของ AWS Provider

```hcl
# provider.tf - Complete AWS provider configuration

provider "aws" {
  # ===== Region =====
  region = "ap-southeast-1"
  
  # ===== Credentials (explicit - ไม่แนะนำสำหรับ production) =====
  # access_key = var.aws_access_key_id
  # secret_key = var.aws_secret_access_key
  # token      = var.aws_session_token  # สำหรับ temporary credentials
  
  # ===== Profile =====
  # profile = "my-profile"  # จาก ~/.aws/credentials
  
  # ===== Assume Role =====
  # assume_role {
  #   role_arn     = "arn:aws:iam::123456789012:role/TerraformRole"
  #   session_name = "TerraformSession"
  #   external_id  = "my-external-id"
  # }
  
  # ===== Default Tags (apply to all resources) =====
  default_tags {
    tags = {
      ManagedBy   = "terraform"
      Project     = var.project_name
      Environment = var.environment
      Owner       = var.owner_email
    }
  }
  
  # ===== Custom Endpoints (LocalStack/testing) =====
  # endpoints {
  #   ec2      = "http://localhost:4566"
  #   s3       = "http://localhost:4566"
  #   dynamodb = "http://localhost:4566"
  # }
  
  # ===== HTTP Settings =====
  # http_proxy  = "http://proxy.example.com:8080"
  # https_proxy = "http://proxy.example.com:8080"
  # no_proxy    = "localhost,127.0.0.1"
  
  # ===== Retry Settings =====
  # max_retries = 3  # default is 25
  
  # ===== Shared Configuration =====
  # shared_config_files      = ["/path/to/config"]
  # shared_credentials_files = ["/path/to/credentials"]
  
  # ===== Allowed Accounts =====
  # allowed_account_ids     = ["123456789012", "987654321098"]
  # forbidden_account_ids   = ["111111111111"]  # prevent accidental deploys
  
  # ===== S3 Force Path Style =====
  # s3_use_path_style = false  # ใช้ true สำหรับ MinIO/LocalStack
  
  # ===== Ignore Tags =====
  # ignore_tags {
  #   keys         = ["kubernetes.io/cluster/"]
  #   key_prefixes = ["kubernetes.io/", "k8s.io/"]
  # }
}
```

---

## Step 182: Authentication Methods อย่างละเอียด

### Method 1: Environment Variables (แนะนำ)

```bash
# Static credentials
export AWS_ACCESS_KEY_ID="AKIAIOSFODNN7EXAMPLE"
export AWS_SECRET_ACCESS_KEY="wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY"
export AWS_DEFAULT_REGION="ap-southeast-1"

# Session token (MFA หรือ temporary credentials)
export AWS_SESSION_TOKEN="FwoGZXIvYXdzEJr..."

# Profile
export AWS_PROFILE="my-profile"
```

```hcl
# provider.tf - ใช้ environment variables โดยไม่ระบุ
provider "aws" {
  region = "ap-southeast-1"
  # credentials อ่านจาก env vars อัตโนมัติ
}
```

### Method 2: AWS Profile (~/.aws/credentials)

```ini
# ~/.aws/credentials

[default]
aws_access_key_id     = AKIAIOSFODNN7EXAMPLE
aws_secret_access_key = wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY

[dev]
aws_access_key_id     = AKIAI44QH8DHBEXAMPLE
aws_secret_access_key = je7MtGbClwBF/2Zp9Utk/h3yCo8nvbEXAMPLEKEY

[prod]
aws_access_key_id     = AKIAIOSFODNN7EXAMPLE2
aws_secret_access_key = wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY2
```

```ini
# ~/.aws/config

[profile default]
region = ap-southeast-1
output = json

[profile dev]
region = ap-southeast-1
output = json

[profile prod]
region = ap-southeast-1
output = json
mfa_serial = arn:aws:iam::123456789012:mfa/my-mfa-device

[profile cross-account]
role_arn = arn:aws:iam::999999999999:role/CrossAccountRole
source_profile = default
region = ap-southeast-1
```

```hcl
provider "aws" {
  region  = "ap-southeast-1"
  profile = "prod"  # ใช้ profile ที่ระบุ
}
```

### Method 3: EC2 Instance Profile / ECS Task Role

```hcl
# สำหรับ EC2 instances หรือ ECS tasks
# ไม่ต้องระบุ credentials เลย
provider "aws" {
  region = "ap-southeast-1"
  # Terraform จะ auto-detect IAM role จาก instance metadata
}
```

```hcl
# IAM Role สำหรับ EC2 instance ที่จะ run Terraform
resource "aws_iam_role" "terraform_runner" {
  name = "terraform-runner"
  
  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Action = "sts:AssumeRole"
      Effect = "Allow"
      Principal = {
        Service = "ec2.amazonaws.com"
      }
    }]
  })
}

resource "aws_iam_role_policy_attachment" "terraform_runner" {
  role       = aws_iam_role.terraform_runner.name
  policy_arn = "arn:aws:iam::aws:policy/PowerUserAccess"
}

resource "aws_iam_instance_profile" "terraform_runner" {
  name = "terraform-runner"
  role = aws_iam_role.terraform_runner.name
}
```

---

## Step 183: Assume Role Configuration

### Cross-Account Role Assumption

```hcl
# assume_role configuration
provider "aws" {
  region = "ap-southeast-1"
  
  assume_role {
    # Role ARN ที่ต้องการ assume
    role_arn = "arn:aws:iam::123456789012:role/TerraformDeployRole"
    
    # Session name สำหรับ audit logs
    session_name = "TerraformDeploy-${var.environment}"
    
    # External ID (optional security measure)
    external_id = var.external_id
    
    # Session duration (seconds)
    duration_seconds = 3600  # 1 hour
    
    # Tags สำหรับ assumed session
    tags = {
      Project     = var.project_name
      Environment = var.environment
    }
    
    # Inline policy sจำกัด permissions
    policy = jsonencode({
      Version = "2012-10-17"
      Statement = [{
        Effect   = "Allow"
        Action   = ["ec2:*", "vpc:*"]
        Resource = "*"
      }]
    })
  }
}
```

### Trust Policy สำหรับ Assume Role

```hcl
# สร้าง role ใน target account
resource "aws_iam_role" "terraform_deploy" {
  name = "TerraformDeployRole"
  
  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        # Allow từ source account
        Effect = "Allow"
        Principal = {
          AWS = "arn:aws:iam::SOURCE_ACCOUNT_ID:root"
        }
        Action = "sts:AssumeRole"
        Condition = {
          StringEquals = {
            "sts:ExternalId" = var.external_id
          }
        }
      },
      {
        # Allow from specific role in source account
        Effect = "Allow"
        Principal = {
          AWS = "arn:aws:iam::SOURCE_ACCOUNT_ID:role/TerraformRunner"
        }
        Action = "sts:AssumeRole"
      }
    ]
  })
}
```

---

## Step 184: default_tags Configuration

### ใช้ default_tags ลด Repetition

```hcl
# ✅ ดี: ใช้ default_tags ใน provider
provider "aws" {
  region = "ap-southeast-1"
  
  default_tags {
    tags = {
      ManagedBy   = "terraform"
      Project     = var.project_name
      Environment = var.environment
      Repository  = "github.com/org/repo"
      Team        = "platform-engineering"
    }
  }
}

# Resources ไม่ต้องระบุ tags ซ้ำ
resource "aws_vpc" "main" {
  cidr_block = "10.0.0.0/16"
  # default_tags ถูก merge เข้ามาอัตโนมัติ
  tags = {
    Name = "main-vpc"  # เพิ่มเฉพาะ resource-specific tags
  }
}

resource "aws_instance" "web" {
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t3.micro"
  tags = {
    Name    = "web-server"
    Service = "frontend"
  }
  # Final tags = default_tags + resource-specific tags
}
```

### default_tags กับ Multiple Providers

```hcl
provider "aws" {
  region = "ap-southeast-1"
  
  default_tags {
    tags = {
      ManagedBy   = "terraform"
      Environment = "prod"
      Region      = "ap-southeast-1"
    }
  }
}

provider "aws" {
  alias  = "us_east"
  region = "us-east-1"
  
  default_tags {
    tags = {
      ManagedBy   = "terraform"
      Environment = "prod"
      Region      = "us-east-1"  # ต่างกัน!
    }
  }
}
```

---

## Step 185: Custom Endpoints สำหรับ Local Testing

### LocalStack Setup

```hcl
# provider-localstack.tf
# สำหรับ development/testing ด้วย LocalStack

provider "aws" {
  region                      = "us-east-1"
  access_key                  = "mock_access_key"
  secret_key                  = "mock_secret_key"
  skip_credentials_validation = true
  skip_requesting_account_id  = true
  skip_metadata_api_check     = true
  s3_use_path_style           = true  # จำเป็นสำหรับ LocalStack
  
  endpoints {
    # Override endpoints ให้ชี้ไป LocalStack
    ec2          = "http://localhost:4566"
    s3           = "http://localhost:4566"
    iam          = "http://localhost:4566"
    sts          = "http://localhost:4566"
    rds          = "http://localhost:4566"
    dynamodb     = "http://localhost:4566"
    lambda       = "http://localhost:4566"
    sqs          = "http://localhost:4566"
    sns          = "http://localhost:4566"
    secretsmanager = "http://localhost:4566"
    ssm          = "http://localhost:4566"
    cloudwatch   = "http://localhost:4566"
    elasticache  = "http://localhost:4566"
    kinesis      = "http://localhost:4566"
  }
}
```

### Docker Compose สำหรับ LocalStack

```yaml
# docker-compose.yml
version: '3.8'

services:
  localstack:
    image: localstack/localstack:latest
    ports:
      - "4566:4566"
    environment:
      - SERVICES=ec2,s3,iam,sts,rds,dynamodb,lambda,sqs,sns,secretsmanager,ssm
      - DEFAULT_REGION=us-east-1
      - DOCKER_HOST=unix:///var/run/docker.sock
    volumes:
      - "/var/run/docker.sock:/var/run/docker.sock"
      - "./localstack:/var/lib/localstack"
```

```bash
# ใช้งาน LocalStack
docker-compose up -d
terraform init
terraform apply

# ตรวจสอบ resources ใน LocalStack
aws --endpoint-url=http://localhost:4566 ec2 describe-instances
aws --endpoint-url=http://localhost:4566 s3 ls
```

---

## Step 186: Multiple AWS Accounts ด้วย Aliases

### Multi-Account Strategy

```hcl
# provider.tf - Multi-account setup

# Management account (สำหรับ shared services)
provider "aws" {
  region  = "ap-southeast-1"
  profile = "management"
  
  default_tags {
    tags = { Account = "management" }
  }
}

# Development account
provider "aws" {
  alias   = "dev"
  region  = "ap-southeast-1"
  profile = "dev"
  
  allowed_account_ids = ["111111111111"]  # ป้องกัน deploy ผิด account
  
  default_tags {
    tags = { Account = "dev", Environment = "dev" }
  }
}

# Staging account
provider "aws" {
  alias   = "staging"
  region  = "ap-southeast-1"
  
  assume_role {
    role_arn     = "arn:aws:iam::222222222222:role/TerraformRole"
    session_name = "terraform-staging"
  }
  
  allowed_account_ids = ["222222222222"]
  
  default_tags {
    tags = { Account = "staging", Environment = "staging" }
  }
}

# Production account
provider "aws" {
  alias   = "prod"
  region  = "ap-southeast-1"
  
  assume_role {
    role_arn     = "arn:aws:iam::333333333333:role/TerraformRole"
    session_name = "terraform-prod"
    external_id  = var.prod_external_id  # extra security
  }
  
  allowed_account_ids = ["333333333333"]
  
  default_tags {
    tags = { Account = "prod", Environment = "prod" }
  }
}
```

### การใช้งาน Multi-Account

```hcl
# main.tf - ใช้ provider aliases

# สร้าง VPC ใน dev account
resource "aws_vpc" "dev" {
  provider   = aws.dev
  cidr_block = "10.1.0.0/16"
  tags = { Name = "dev-vpc" }
}

# สร้าง VPC ใน prod account
resource "aws_vpc" "prod" {
  provider   = aws.prod
  cidr_block = "10.0.0.0/16"
  tags = { Name = "prod-vpc" }
}

# VPC Peering ระหว่าง accounts
resource "aws_vpc_peering_connection" "dev_to_prod" {
  provider    = aws.dev        # initiator ใน dev
  vpc_id      = aws_vpc.dev.id
  peer_vpc_id = aws_vpc.prod.id
  peer_owner_id = "333333333333"  # prod account ID
  peer_region   = "ap-southeast-1"
  auto_accept   = false
}

resource "aws_vpc_peering_connection_accepter" "prod" {
  provider                  = aws.prod  # accepter ใน prod
  vpc_peering_connection_id = aws_vpc_peering_connection.dev_to_prod.id
  auto_accept               = true
}
```

---

## Step 187: Cross-Account Role Assumption Patterns

### Hub-and-Spoke Pattern

```hcl
# Hub (Management Account) → Spoke (Multiple Accounts) Pattern

# Central management account
variable "account_ids" {
  description = "Map of account names to IDs"
  type        = map(string)
  default = {
    dev     = "111111111111"
    staging = "222222222222"
    prod    = "333333333333"
  }
}

# สร้าง provider สำหรับแต่ละ account
provider "aws" {
  for_each = var.account_ids
  alias    = each.key
  region   = "ap-southeast-1"
  
  assume_role {
    role_arn     = "arn:aws:iam::${each.value}:role/TerraformRole"
    session_name = "terraform-${each.key}"
  }
}
# หมายเหตุ: ใน Terraform provider ไม่รองรับ for_each โดยตรง
# ต้องสร้างแต่ละ provider แยกกัน
```

### Organization-level Deployment

```hcl
# data source: list ทุก accounts ใน Organization
data "aws_organizations_organization" "current" {}

# Terraform ใน management account
# Deploy resources ไปยัง member accounts

data "aws_caller_identity" "management" {}

locals {
  member_accounts = [
    for account in data.aws_organizations_organization.current.accounts :
      account if account.id != data.aws_caller_identity.management.account_id
  ]
}
```

---

## Step 188: Provider Meta-argument

### provider_meta (Provider-specific metadata)

```hcl
# provider_meta ส่งข้อมูลพิเศษไปยัง provider
# ใช้ได้เฉพาะเมื่อ provider รองรับ

terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

# ตัวอย่าง: ส่ง module attribution
resource "aws_instance" "example" {
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t3.micro"
}
```

### Configuration Options ขั้นสูง

```hcl
# provider.tf - Advanced options

provider "aws" {
  region = "ap-southeast-1"
  
  # Retry settings (ลด ThrottlingException)
  max_retries = 5
  
  # Custom HTTP client settings
  # Useful สำหรับ on-premises environments
  # http_proxy  = "http://corporate-proxy:8080"
  # https_proxy = "http://corporate-proxy:8080"
  # no_proxy    = "169.254.169.254,*.internal"  # metadata service
  
  # Ignore tags สำหรับ EKS-managed resources
  ignore_tags {
    key_prefixes = [
      "kubernetes.io/",
      "k8s.io/",
      "eks.amazonaws.com/",
    ]
  }
  
  # Security: ป้องกัน deploy ไปผิด account
  allowed_account_ids = [
    data.aws_caller_identity.current.account_id
  ]
}
```

---

## Step 189: Environment-specific Provider Configurations

### Pattern: ใช้ Variables เลือก Configuration

```hcl
# variables.tf
variable "environment" {
  type = string
  validation {
    condition     = contains(["dev", "staging", "prod"], var.environment)
    error_message = "Environment must be dev, staging, or prod."
  }
}

variable "aws_region" {
  type    = string
  default = "ap-southeast-1"
}

# locals.tf
locals {
  # Account ID ตาม environment
  account_map = {
    dev     = "111111111111"
    staging = "222222222222"
    prod    = "333333333333"
  }
  
  # Role ARN ตาม environment
  role_arn_map = {
    dev     = null  # ใช้ default credentials
    staging = "arn:aws:iam::222222222222:role/TerraformRole"
    prod    = "arn:aws:iam::333333333333:role/TerraformRole"
  }
  
  target_account_id = local.account_map[var.environment]
  target_role_arn   = local.role_arn_map[var.environment]
}
```

```hcl
# providers.tf - Environment-aware provider
# หมายเหตุ: Terraform ไม่รองรับ dynamic provider config โดยตรง
# ต้องใช้ pattern นี้:

provider "aws" {
  region = var.aws_region
  
  # Conditional assume_role - ต้องใช้ dynamic blocks pattern
  # หรือแยกเป็นหลาย provider files และ switch ด้วย TF_VAR_*
  
  dynamic "assume_role" {
    for_each = local.target_role_arn != null ? [1] : []
    content {
      role_arn     = local.target_role_arn
      session_name = "terraform-${var.environment}"
    }
  }
  
  allowed_account_ids = [local.target_account_id]
  
  default_tags {
    tags = {
      Environment = var.environment
      ManagedBy   = "terraform"
    }
  }
}
```

---

## Step 190: Provider Version Upgrade Strategies

### การอัพเกรด Provider Safely

```bash
# 1. ตรวจสอบ current versions
terraform providers

# 2. ดู available versions
# ไปที่ registry.terraform.io ดู changelog

# 3. อ่าน CHANGELOG สำหรับ breaking changes
# https://github.com/hashicorp/terraform-provider-aws/blob/main/CHANGELOG.md

# 4. อัพเดท version constraint
# เปลี่ยน ~> 5.0 เป็น ~> 5.31 (หรือ version ใหม่)

# 5. Run init -upgrade
terraform init -upgrade

# 6. Run plan ดู changes
terraform plan

# 7. ทดสอบใน dev ก่อน
# 8. Apply ใน dev
# 9. ทดสอบ functionality
# 10. Repeat สำหรับ staging และ prod
```

### Breaking Changes Handling

```hcl
# ตัวอย่าง: AWS Provider 4.x → 5.x migration

# AWS Provider 4.x
resource "aws_s3_bucket" "main" {
  bucket = "my-bucket"
  acl    = "private"  # deprecated ใน 4.x, removed ใน 5.x!
  
  versioning {  # deprecated, ใช้ aws_s3_bucket_versioning แทน
    enabled = true
  }
}

# AWS Provider 5.x
resource "aws_s3_bucket" "main" {
  bucket = "my-bucket"
  # ลบ acl และ versioning block ออก
}

resource "aws_s3_bucket_acl" "main" {  # แยก resource
  bucket = aws_s3_bucket.main.id
  acl    = "private"
}

resource "aws_s3_bucket_versioning" "main" {  # แยก resource
  bucket = aws_s3_bucket.main.id
  
  versioning_configuration {
    status = "Enabled"
  }
}
```

### Using Upgrade Tools

```bash
# awscc provider migration helper
# https://github.com/hashicorp/aws-sdk-go-v2

# Terraform AWS Provider Migration Guide
# https://registry.terraform.io/providers/hashicorp/aws/latest/docs/guides/version-5-upgrade

# tfupdate - อัพเดท version constraints อัตโนมัติ
brew install tfupdate
tfupdate provider aws --version "~> 5.0" .
tfupdate terraform --version "~> 1.7" .
```

---

## Production Setup สมบูรณ์

```hcl
# production/provider.tf

terraform {
  required_version = ">= 1.5.0, < 2.0.0"
  
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.31"
    }
    random = {
      source  = "hashicorp/random"
      version = "~> 3.6"
    }
  }
  
  backend "s3" {
    bucket         = "my-org-terraform-state"
    key            = "production/terraform.tfstate"
    region         = "ap-southeast-1"
    encrypt        = true
    dynamodb_table = "terraform-state-lock"
    
    # Assume role สำหรับ state bucket access
    role_arn = "arn:aws:iam::MGMT_ACCOUNT:role/TerraformStateRole"
  }
}

# Production provider (assume role)
provider "aws" {
  region = var.region
  
  assume_role {
    role_arn     = "arn:aws:iam::${var.prod_account_id}:role/TerraformDeployRole"
    session_name = "terraform-prod-${formatdate("YYYY-MM-DD", timestamp())}"
    external_id  = var.assume_role_external_id
    
    tags = {
      ManagedBy   = "terraform"
      Environment = "prod"
    }
  }
  
  allowed_account_ids = [var.prod_account_id]
  
  default_tags {
    tags = {
      ManagedBy        = "terraform"
      Environment      = "prod"
      Project          = var.project_name
      Team             = var.team_name
      CostCenter       = var.cost_center
      Repository       = var.git_repo
      TerraformVersion = ">=1.5.0"
    }
  }
  
  # ไม่ ignore tags สำหรับ EKS ถ้าไม่ได้ใช้ EKS
  # ignore_tags {
  #   key_prefixes = ["kubernetes.io/"]
  # }
}

# US East provider สำหรับ ACM (CloudFront requirement)
provider "aws" {
  alias  = "us_east_1"
  region = "us-east-1"
  
  assume_role {
    role_arn     = "arn:aws:iam::${var.prod_account_id}:role/TerraformDeployRole"
    session_name = "terraform-prod-us-east-1"
    external_id  = var.assume_role_external_id
  }
  
  allowed_account_ids = [var.prod_account_id]
  
  default_tags {
    tags = {
      ManagedBy   = "terraform"
      Environment = "prod"
      Region      = "us-east-1"
    }
  }
}
```

---

## สรุป (Summary)

### Authentication Priority Order (AWS Provider)

```
1. Static credentials ใน provider block (ไม่แนะนำ)
2. Environment variables (AWS_ACCESS_KEY_ID, etc.)
3. AWS Shared Credential File (~/.aws/credentials)
4. AWS Shared Configuration File (~/.aws/config)
5. Container credentials (ECS task role)
6. Instance Profile Credentials (EC2 instance role)
```

### ✅ Production Best Practices

1. **ไม่ hardcode credentials** - ใช้ IAM roles หรือ env vars
2. **allowed_account_ids** - ป้องกัน deploy ผิด account
3. **default_tags** - enforce tagging policy
4. **external_id** ใน assume_role - extra security
5. **Session name ที่ meaningful** - ช่วย audit logs
6. **ignore_tags** สำหรับ EKS-managed tags

### ⚠️ Security Warnings

```hcl
# ❌ อย่าทำ: hardcode credentials
provider "aws" {
  access_key = "AKIAIOSFODNN7EXAMPLE"
  secret_key = "wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY"
}

# ❌ อย่าทำ: ไม่ restrict accounts
provider "aws" {
  region = "ap-southeast-1"
  # ไม่มี allowed_account_ids - อันตราย!
}

# ✅ ทำอย่างนี้
provider "aws" {
  region = "ap-southeast-1"
  
  assume_role {
    role_arn = "arn:aws:iam::${var.account_id}:role/TerraformRole"
  }
  
  allowed_account_ids = [var.account_id]
}
```

---

*จบ Part 019 - Provider Configuration Deep Dive*
