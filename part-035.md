# Part 035: Terraform Configuration Best Practices
# Best Practices สำหรับ Terraform Configuration

## Steps 341-350: คู่มือ Best Practices ที่ครอบคลุมทุกด้าน

---

## Step 341: Standard Module Structure

### โครงสร้าง Module ที่แนะนำ

```
my-terraform-project/
├── environments/
│   ├── development/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   ├── outputs.tf
│   │   ├── versions.tf
│   │   ├── backend.tf
│   │   └── terraform.tfvars
│   ├── staging/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   ├── outputs.tf
│   │   ├── versions.tf
│   │   ├── backend.tf
│   │   └── terraform.tfvars
│   └── production/
│       ├── main.tf
│       ├── variables.tf
│       ├── outputs.tf
│       ├── versions.tf
│       ├── backend.tf
│       └── terraform.tfvars
│
├── modules/
│   ├── vpc/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   ├── outputs.tf
│   │   ├── versions.tf
│   │   ├── locals.tf
│   │   ├── data.tf
│   │   └── README.md
│   ├── ec2/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   ├── outputs.tf
│   │   ├── versions.tf
│   │   ├── locals.tf
│   │   └── README.md
│   ├── rds/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   ├── outputs.tf
│   │   ├── versions.tf
│   │   └── README.md
│   └── security-groups/
│       ├── main.tf
│       ├── variables.tf
│       ├── outputs.tf
│       └── README.md
│
├── .github/
│   └── workflows/
│       ├── terraform-plan.yml
│       ├── terraform-apply.yml
│       └── drift-detection.yml
│
├── scripts/
│   ├── validate.sh
│   └── format.sh
│
├── .pre-commit-config.yaml
├── .gitignore
├── .terraform-version
└── README.md
```

---

## Step 342: File Organization

### การจัดระเบียบ Files

#### main.tf - Resources หลัก

```hcl
# modules/vpc/main.tf

# VPC
resource "aws_vpc" "this" {
  cidr_block           = var.vpc_cidr
  enable_dns_hostnames = true
  enable_dns_support   = true

  tags = local.vpc_tags
}

# Public Subnets
resource "aws_subnet" "public" {
  count = length(var.public_subnet_cidrs)

  vpc_id                  = aws_vpc.this.id
  cidr_block              = var.public_subnet_cidrs[count.index]
  availability_zone       = var.availability_zones[count.index]
  map_public_ip_on_launch = true

  tags = merge(local.common_tags, {
    Name = "${local.name_prefix}-public-${var.availability_zones[count.index]}"
    Tier = "public"
  })
}

# Private Subnets
resource "aws_subnet" "private" {
  count = length(var.private_subnet_cidrs)

  vpc_id            = aws_vpc.this.id
  cidr_block        = var.private_subnet_cidrs[count.index]
  availability_zone = var.availability_zones[count.index]

  tags = merge(local.common_tags, {
    Name = "${local.name_prefix}-private-${var.availability_zones[count.index]}"
    Tier = "private"
  })
}

# Internet Gateway
resource "aws_internet_gateway" "this" {
  vpc_id = aws_vpc.this.id

  tags = merge(local.common_tags, {
    Name = "${local.name_prefix}-igw"
  })
}

# NAT Gateway EIP
resource "aws_eip" "nat" {
  count = var.create_nat_gateway ? 1 : 0

  domain     = "vpc"
  depends_on = [aws_internet_gateway.this]

  tags = merge(local.common_tags, {
    Name = "${local.name_prefix}-nat-eip"
  })
}

# NAT Gateway
resource "aws_nat_gateway" "this" {
  count = var.create_nat_gateway ? 1 : 0

  allocation_id = aws_eip.nat[0].id
  subnet_id     = aws_subnet.public[0].id

  tags = merge(local.common_tags, {
    Name = "${local.name_prefix}-nat"
  })

  depends_on = [aws_internet_gateway.this]
}
```

#### variables.tf - Input Variables

```hcl
# modules/vpc/variables.tf

# ─── Required Variables ────────────────────────────────────

variable "project_name" {
  description = "ชื่อโปรเจกต์ (ใช้ใน naming)"
  type        = string

  validation {
    condition     = can(regex("^[a-z][a-z0-9-]*[a-z0-9]$", var.project_name))
    error_message = "project_name ต้องขึ้นต้นด้วยตัวพิมพ์เล็ก ตามด้วยตัวพิมพ์เล็ก ตัวเลข หรือ - เท่านั้น"
  }
}

variable "environment" {
  description = "ชื่อ environment (development, staging, production)"
  type        = string

  validation {
    condition     = contains(["development", "staging", "production"], var.environment)
    error_message = "environment ต้องเป็น development, staging, หรือ production เท่านั้น"
  }
}

variable "vpc_cidr" {
  description = "CIDR block สำหรับ VPC"
  type        = string
  default     = "10.0.0.0/16"

  validation {
    condition     = can(cidrhost(var.vpc_cidr, 0))
    error_message = "vpc_cidr ต้องเป็น valid CIDR notation"
  }
}

variable "availability_zones" {
  description = "List ของ Availability Zones ที่ต้องการใช้"
  type        = list(string)

  validation {
    condition     = length(var.availability_zones) >= 2
    error_message = "ต้องระบุอย่างน้อย 2 Availability Zones"
  }
}

variable "public_subnet_cidrs" {
  description = "CIDR blocks สำหรับ public subnets"
  type        = list(string)

  validation {
    condition     = length(var.public_subnet_cidrs) == length(var.availability_zones)
    error_message = "จำนวน public_subnet_cidrs ต้องเท่ากับจำนวน availability_zones"
  }
}

variable "private_subnet_cidrs" {
  description = "CIDR blocks สำหรับ private subnets"
  type        = list(string)

  validation {
    condition     = length(var.private_subnet_cidrs) == length(var.availability_zones)
    error_message = "จำนวน private_subnet_cidrs ต้องเท่ากับจำนวน availability_zones"
  }
}

# ─── Optional Variables ────────────────────────────────────

variable "create_nat_gateway" {
  description = "สร้าง NAT Gateway สำหรับ private subnets"
  type        = bool
  default     = true
}

variable "tags" {
  description = "Tags เพิ่มเติมที่ต้องการเพิ่มใน resources ทุกอัน"
  type        = map(string)
  default     = {}
}

variable "enable_vpc_flow_logs" {
  description = "เปิดใช้ VPC Flow Logs"
  type        = bool
  default     = true
}

variable "flow_log_retention_days" {
  description = "จำนวนวันที่เก็บ VPC Flow Logs"
  type        = number
  default     = 30

  validation {
    condition     = contains([1, 3, 5, 7, 14, 30, 60, 90, 120, 150, 180, 365, 400, 545, 731, 1827, 3653], var.flow_log_retention_days)
    error_message = "flow_log_retention_days ต้องเป็นค่าที่ CloudWatch Logs รองรับ"
  }
}
```

#### outputs.tf - Output Values

```hcl
# modules/vpc/outputs.tf

output "vpc_id" {
  description = "ID ของ VPC"
  value       = aws_vpc.this.id
}

output "vpc_arn" {
  description = "ARN ของ VPC"
  value       = aws_vpc.this.arn
}

output "vpc_cidr_block" {
  description = "CIDR block ของ VPC"
  value       = aws_vpc.this.cidr_block
}

output "public_subnet_ids" {
  description = "IDs ของ public subnets"
  value       = aws_subnet.public[*].id
}

output "private_subnet_ids" {
  description = "IDs ของ private subnets"
  value       = aws_subnet.private[*].id
}

output "public_subnet_cidrs" {
  description = "CIDR blocks ของ public subnets"
  value       = aws_subnet.public[*].cidr_block
}

output "private_subnet_cidrs" {
  description = "CIDR blocks ของ private subnets"
  value       = aws_subnet.private[*].cidr_block
}

output "internet_gateway_id" {
  description = "ID ของ Internet Gateway"
  value       = aws_internet_gateway.this.id
}

output "nat_gateway_id" {
  description = "ID ของ NAT Gateway (null ถ้าไม่ได้สร้าง)"
  value       = var.create_nat_gateway ? aws_nat_gateway.this[0].id : null
}

output "nat_gateway_public_ip" {
  description = "Public IP ของ NAT Gateway"
  value       = var.create_nat_gateway ? aws_eip.nat[0].public_ip : null
}
```

#### versions.tf - Provider Requirements

```hcl
# modules/vpc/versions.tf

terraform {
  required_version = ">= 1.5.0"

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = ">= 5.0.0"
    }
  }
}
```

#### locals.tf - Local Values

```hcl
# modules/vpc/locals.tf

locals {
  # Naming prefix
  name_prefix = "${var.project_name}-${var.environment}"

  # Common tags ที่ใส่ใน resources ทุกอัน
  common_tags = merge(
    {
      Project     = var.project_name
      Environment = var.environment
      ManagedBy   = "terraform"
      CreatedAt   = timestamp()
    },
    var.tags
  )

  # VPC-specific tags
  vpc_tags = merge(local.common_tags, {
    Name = "${local.name_prefix}-vpc"
  })

  # คำนวณ subnet tiers
  subnet_count = length(var.availability_zones)
}
```

#### data.tf - Data Sources

```hcl
# modules/vpc/data.tf

# ดึง current AWS region
data "aws_region" "current" {}

# ดึง current caller identity
data "aws_caller_identity" "current" {}

# ดึง available AZs
data "aws_availability_zones" "available" {
  state = "available"
}
```

---

## Step 343: Naming Conventions

### หลักการตั้งชื่อ

```hcl
# ─── Resource Naming: snake_case ──────────────────────────

# ✅ ดี: snake_case, descriptive
resource "aws_vpc" "main" {}
resource "aws_instance" "web_server" {}
resource "aws_security_group" "alb_sg" {}
resource "aws_db_instance" "primary_postgres" {}

# ❌ ไม่ดี: camelCase
resource "aws_vpc" "mainVpc" {}

# ❌ ไม่ดี: abbreviations ที่ไม่ชัดเจน
resource "aws_vpc" "v" {}
resource "aws_instance" "i1" {}

# ─── Resource Names ใน AWS: kebab-case ────────────────────

# ✅ ดี: project-env-resource
# production-web-alb
# production-private-subnet-1a

locals {
  # Pattern: {project}-{environment}-{resource-type}-{identifier}
  alb_name        = "${var.project_name}-${var.environment}-alb"
  vpc_name        = "${var.project_name}-${var.environment}-vpc"
  public_subnet_1 = "${var.project_name}-${var.environment}-public-1a"
  private_subnet_1 = "${var.project_name}-${var.environment}-private-1a"
  
  # Database: {project}-{environment}-{engine}
  rds_identifier = "${var.project_name}-${var.environment}-postgres"
}

# ─── Variable Naming ───────────────────────────────────────

# ✅ ดี: descriptive, snake_case
variable "vpc_cidr_block" {}
variable "instance_type" {}
variable "enable_deletion_protection" {}
variable "database_password" {}

# ❌ ไม่ดี: too short or unclear
variable "cidr" {}
variable "type" {}
variable "pass" {}

# ─── Output Naming ─────────────────────────────────────────

# ✅ ดี: what the value is, not just "id" or "arn"
output "vpc_id" {}
output "alb_dns_name" {}
output "rds_endpoint" {}
output "private_subnet_ids" {}

# ❌ ไม่ดี: generic names
output "id" {}
output "endpoint" {}

# ─── Module Naming ─────────────────────────────────────────

# ✅ ดี: lowercase, kebab-case, descriptive
module "vpc" {}
module "web-servers" {}
module "rds-postgres" {}

# ❌ ไม่ดี
module "MyVPC" {}
module "vpcModule" {}
```

---

## Step 344: Version Constraints Strategy

### แนวทางการตั้ง Version Constraints

```hcl
# versions.tf

terraform {
  # ─── Terraform Version ──────────────────────────────────
  
  # ✅ แนะนำ: ใช้ >= ที่ minimum version ที่รองรับ features ที่ใช้
  required_version = ">= 1.5.0"

  # ✅ หรือระบุ range ชัดเจน
  # required_version = ">= 1.5.0, < 2.0.0"

  # ❌ ไม่แนะนำ: Exact version (ทำให้ upgrade ยาก)
  # required_version = "= 1.6.0"

  # ❌ ไม่แนะนำ: ไม่มี constraint
  # (ไม่มี required_version)

  required_providers {
    # ─── AWS Provider ─────────────────────────────────────
    
    # ✅ แนะนำ: Pessimistic operator - อนุญาต minor และ patch
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }

    # ✅ หรือ: กำหนด range ชัดเจน
    # aws = {
    #   source  = "hashicorp/aws"
    #   version = ">= 5.0, < 6.0"
    # }

    # ─── Community Providers ──────────────────────────────
    
    # ✅ แนะนำ: ระมัดระวังมากขึ้นกับ community providers
    datadog = {
      source  = "DataDog/datadog"
      version = "~> 3.30"  # lock ที่ minor version สำหรับ stability
    }

    # ─── Other Hashicorp Providers ────────────────────────
    
    random = {
      source  = "hashicorp/random"
      version = ">= 3.1.0"
    }

    null = {
      source  = "hashicorp/null"
      version = ">= 3.0.0"
    }

    archive = {
      source  = "hashicorp/archive"
      version = ">= 2.0.0"
    }

    tls = {
      source  = "hashicorp/tls"
      version = ">= 4.0.0"
    }
  }
}
```

### .terraform-version File (tfenv)

```bash
# .terraform-version
1.6.4
# ใช้กับ tfenv: https://github.com/tfutils/tfenv
# ทีมทุกคนจะใช้ Terraform version นี้
```

---

## Step 345: Secret Management

### ❌ สิ่งที่ห้ามทำ

```hcl
# ❌ NEVER: Hardcode credentials ใน .tf files
provider "aws" {
  region     = "ap-southeast-1"
  access_key = "AKIAIOSFODNN7EXAMPLE"      # ❌ ห้ามทำ!
  secret_key = "wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY"  # ❌ ห้ามทำ!
}

# ❌ NEVER: Hardcode passwords
resource "aws_db_instance" "main" {
  password = "SuperSecretPassword123!"  # ❌ ห้ามทำ!
}

# ❌ NEVER: Hardcode API keys
resource "datadog_monitor" "cpu" {
  api_key = "abc123def456ghi789"  # ❌ ห้ามทำ!
}
```

### ✅ วิธีที่ถูกต้อง: Environment Variables

```bash
# ✅ ใช้ Environment Variables สำหรับ AWS credentials
export AWS_ACCESS_KEY_ID="AKIAIOSFODNN7EXAMPLE"
export AWS_SECRET_ACCESS_KEY="wJalrXUtnFEMI/K7MDENG/..."
export AWS_DEFAULT_REGION="ap-southeast-1"

# provider.tf - ไม่ต้องระบุ credentials ใน config
provider "aws" {
  region = var.aws_region
  # credentials จะอ่านจาก environment variables โดยอัตโนมัติ
}
```

### ✅ วิธีที่ถูกต้อง: IAM Roles (แนะนำที่สุด)

```hcl
# ✅ สำหรับ EC2, ECS, Lambda - ใช้ Instance/Task/Execution Role
provider "aws" {
  region = var.aws_region
  # credentials จะอ่านจาก instance metadata service (IMDS)
  # ไม่ต้องระบุ access_key หรือ secret_key เลย!
}

# ✅ Assume Role สำหรับ Cross-account access
provider "aws" {
  region = var.aws_region

  assume_role {
    role_arn     = "arn:aws:iam::123456789012:role/TerraformRole"
    session_name = "TerraformSession"
    external_id  = var.external_id  # ถ้าต้องการ
  }
}
```

### ✅ วิธีที่ถูกต้อง: AWS Secrets Manager

```hcl
# ✅ ดึง secrets จาก AWS Secrets Manager
data "aws_secretsmanager_secret_version" "db_password" {
  secret_id = "/${var.environment}/rds/password"
}

resource "aws_db_instance" "main" {
  identifier     = "${var.project_name}-${var.environment}"
  engine         = "postgres"
  instance_class = "db.t3.micro"
  
  # ✅ ดึง password จาก Secrets Manager
  username = "dbadmin"
  password = data.aws_secretsmanager_secret_version.db_password.secret_string
  
  # ✅ ไม่แสดง password ใน plan output
  lifecycle {
    ignore_changes = [password]
  }
}
```

### ✅ วิธีที่ถูกต้อง: HashiCorp Vault

```hcl
# ✅ ใช้ HashiCorp Vault สำหรับ secrets
provider "vault" {
  address = "https://vault.company.com"
  # Authentication ผ่าน VAULT_TOKEN environment variable
}

data "vault_generic_secret" "db" {
  path = "secret/data/database"
}

resource "aws_db_instance" "main" {
  username = data.vault_generic_secret.db.data["username"]
  password = data.vault_generic_secret.db.data["password"]
  
  lifecycle {
    ignore_changes = [password]
  }
}
```

### ✅ วิธีที่ถูกต้อง: Variable + Sensitive

```hcl
# variables.tf
variable "db_password" {
  description = "Password สำหรับ RDS instance"
  type        = string
  sensitive   = true  # ✅ ซ่อนค่าใน plan/apply output
}

# terraform.tfvars (อยู่ใน .gitignore!)
db_password = "my-secret-password"

# หรือ environment variable:
# export TF_VAR_db_password="my-secret-password"
```

---

## Step 346: Tagging Strategy

### Mandatory Tags

```hcl
# locals.tf - กำหนด tags ที่บังคับ

locals {
  mandatory_tags = {
    # ─── Resource Identity ─────────────────────────────────
    Project     = var.project_name          # ชื่อโปรเจกต์
    Environment = var.environment           # development/staging/production
    
    # ─── Ownership ─────────────────────────────────────────
    Owner       = var.team_name             # ชื่อทีมที่ดูแล
    CostCenter  = var.cost_center           # สำหรับ cost allocation
    
    # ─── Management ────────────────────────────────────────
    ManagedBy   = "terraform"              # บอกว่าจัดการด้วย Terraform
    Repository  = var.git_repository       # URL ของ Git repository
    
    # ─── Compliance ─────────────────────────────────────────
    DataClass   = var.data_classification  # public/internal/confidential/restricted
  }

  # รวม mandatory tags กับ optional tags
  all_tags = merge(local.mandatory_tags, var.additional_tags)
}
```

### Default Tags กับ AWS Provider

```hcl
# provider.tf

provider "aws" {
  region = var.aws_region

  # ✅ default_tags ใส่ tags ใน resources ทุกอันอัตโนมัติ
  default_tags {
    tags = {
      Project     = var.project_name
      Environment = var.environment
      ManagedBy   = "terraform"
      Repository  = "https://github.com/myorg/infrastructure"
      Owner       = "platform-team"
    }
  }
}

# Resource ไม่ต้องระบุ tags ที่ซ้ำกัน
resource "aws_instance" "web" {
  ami           = data.aws_ami.amazon_linux.id
  instance_type = "t3.micro"

  # แค่ระบุ tags ที่เฉพาะเจาะจงสำหรับ resource นี้
  tags = {
    Name = "web-server-01"
    Role = "web"
  }
  # ← default_tags จะถูกเพิ่มโดยอัตโนมัติ
}
```

### Cost Allocation Tags

```hcl
# ตัวอย่าง Cost Allocation Tags

locals {
  cost_tags = {
    CostCenter    = var.cost_center       # "IT-001", "ENG-002"
    Project       = var.project_name      # "ecommerce-platform"
    Environment   = var.environment       # production ← costs more
    BusinessUnit  = var.business_unit     # "retail", "b2b"
    Application   = var.app_name          # "web-frontend", "api"
    Team          = var.team_name         # "frontend", "backend", "devops"
  }
}
```

---

## Step 347: Pre-commit Hooks Setup

### การติดตั้งและ Configure Pre-commit

```bash
# ติดตั้ง pre-commit
pip install pre-commit

# หรือ
brew install pre-commit

# ติดตั้ง hooks
pre-commit install
```

### .pre-commit-config.yaml

```yaml
# .pre-commit-config.yaml

repos:
  # ─── Terraform-specific hooks ──────────────────────────
  
  - repo: https://github.com/antonbabenko/pre-commit-terraform
    rev: v1.85.0
    hooks:
      # Format Terraform code
      - id: terraform_fmt
        args:
          - --args=-recursive
          - --args=-diff

      # Validate Terraform configurations
      - id: terraform_validate
        args:
          - --init-args=-backend=false

      # Run terraform docs
      - id: terraform_docs
        args:
          - --args=--config=.terraform-docs.yml

      # Run tflint
      - id: terraform_tflint
        args:
          - --args=--config=__GIT_WORKING_DIR__/.tflint.hcl

      # Run tfsec security scanner
      - id: terraform_tfsec
        args:
          - --args=--config-file=.tfsec.yml

      # Run checkov
      - id: terraform_checkov
        args:
          - --args=--config-file=.checkov.yml

      # Lock file maintenance
      - id: terraform_providers_lock
        args:
          - --args=-platform=linux_amd64
          - --args=-platform=darwin_arm64

  # ─── General hooks ─────────────────────────────────────
  
  - repo: https://github.com/pre-commit/pre-commit-hooks
    rev: v4.5.0
    hooks:
      # ไม่ให้ commit secrets
      - id: detect-aws-credentials
      - id: detect-private-key
      
      # File hygiene
      - id: trailing-whitespace
      - id: end-of-file-fixer
      - id: check-yaml
      - id: check-json
      - id: check-merge-conflict
      - id: check-case-conflict
      
      # Prevent large files
      - id: check-added-large-files
        args: ['--maxkb=500']
      
      # Don't commit .tfvars with real values
      - id: no-commit-to-branch
        args: ['--branch', 'main', '--branch', 'master']

  # ─── Secret detection ──────────────────────────────────
  
  - repo: https://github.com/Yelp/detect-secrets
    rev: v1.4.0
    hooks:
      - id: detect-secrets
        args: ['--baseline', '.secrets.baseline']
```

### .tflint.hcl Configuration

```hcl
# .tflint.hcl

config {
  module = true
  force  = false
}

plugin "aws" {
  enabled = true
  version = "0.27.0"
  source  = "github.com/terraform-linters/tflint-ruleset-aws"
}

# ─── Rules ─────────────────────────────────────────────────

# ห้ามใช้ deprecated resources
rule "aws_instance_invalid_type" {
  enabled = true
}

# ตรวจสอบ naming conventions
rule "terraform_naming_convention" {
  enabled = true
  
  variable {
    format = "snake_case"
  }
  
  locals {
    format = "snake_case"
  }
  
  output {
    format = "snake_case"
  }
  
  resource {
    format = "snake_case"
  }
}

# ห้ามใช้ wildcard ใน IAM
rule "aws_iam_policy_document_gov_friendly_arns" {
  enabled = true
}
```

---

## Step 348: CI/CD Pipeline Design

### GitHub Actions Pipeline (Complete)

```yaml
# .github/workflows/terraform.yml
name: Terraform CI/CD

on:
  push:
    branches: [main]
    paths:
      - 'environments/**'
      - 'modules/**'
  pull_request:
    branches: [main]
    paths:
      - 'environments/**'
      - 'modules/**'

env:
  TF_VERSION: "1.6.4"
  AWS_REGION: "ap-southeast-1"

jobs:
  # ─── Validate and Format ──────────────────────────────────
  validate:
    name: Validate
    runs-on: ubuntu-latest
    
    steps:
      - uses: actions/checkout@v4

      - name: Setup Terraform
        uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: ${{ env.TF_VERSION }}

      - name: Check Format
        run: terraform fmt -check -recursive

      - name: Validate (Development)
        run: |
          cd environments/development
          terraform init -backend=false
          terraform validate

  # ─── Security Scan ─────────────────────────────────────────
  security:
    name: Security Scan
    runs-on: ubuntu-latest
    needs: validate
    
    steps:
      - uses: actions/checkout@v4

      - name: Run tfsec
        uses: aquasecurity/tfsec-action@v1.0.0
        with:
          additional_args: --config-file .tfsec.yml

      - name: Run checkov
        uses: bridgecrewio/checkov-action@v12
        with:
          directory: environments/
          framework: terraform
          output_format: sarif
          output_file_path: reports/results.sarif

      - name: Upload SARIF
        uses: github/codeql-action/upload-sarif@v2
        if: always()
        with:
          sarif_file: reports/results.sarif

  # ─── Plan ──────────────────────────────────────────────────
  plan:
    name: Plan (${{ matrix.environment }})
    runs-on: ubuntu-latest
    needs: [validate, security]
    
    strategy:
      matrix:
        environment: [development, staging, production]
    
    permissions:
      id-token: write
      contents: read
      pull-requests: write
    
    steps:
      - uses: actions/checkout@v4

      - name: Configure AWS Credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ secrets[format('AWS_ROLE_{0}', matrix.environment)] }}
          aws-region: ${{ env.AWS_REGION }}

      - name: Setup Terraform
        uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: ${{ env.TF_VERSION }}

      - name: Terraform Init
        working-directory: environments/${{ matrix.environment }}
        run: terraform init -input=false

      - name: Terraform Plan
        id: plan
        working-directory: environments/${{ matrix.environment }}
        run: |
          terraform plan -input=false -no-color \
            -out=${{ matrix.environment }}.tfplan \
            2>&1 | tee plan_output.txt
          echo "exitcode=$?" >> $GITHUB_OUTPUT

      - name: Comment Plan on PR
        if: github.event_name == 'pull_request'
        uses: actions/github-script@v6
        with:
          script: |
            const fs = require('fs');
            const plan = fs.readFileSync('environments/${{ matrix.environment }}/plan_output.txt', 'utf8');
            
            github.rest.issues.createComment({
              issue_number: context.issue.number,
              owner: context.repo.owner,
              repo: context.repo.repo,
              body: `## Terraform Plan - ${{ matrix.environment }}
              
\`\`\`hcl
${plan.substring(0, 3000)}
\`\`\`
`
            });

      - name: Save Plan Artifact
        uses: actions/upload-artifact@v3
        with:
          name: ${{ matrix.environment }}-plan
          path: environments/${{ matrix.environment }}/${{ matrix.environment }}.tfplan

  # ─── Apply (Development) - Auto on push to main ────────────
  apply-dev:
    name: Apply (development)
    runs-on: ubuntu-latest
    needs: plan
    if: github.ref == 'refs/heads/main' && github.event_name == 'push'
    environment: development
    
    permissions:
      id-token: write
      contents: read
    
    steps:
      - uses: actions/checkout@v4

      - name: Configure AWS Credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ secrets.AWS_ROLE_DEVELOPMENT }}
          aws-region: ${{ env.AWS_REGION }}

      - name: Setup Terraform
        uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: ${{ env.TF_VERSION }}

      - name: Download Plan
        uses: actions/download-artifact@v3
        with:
          name: development-plan
          path: environments/development/

      - name: Terraform Init
        working-directory: environments/development
        run: terraform init -input=false

      - name: Terraform Apply
        working-directory: environments/development
        run: terraform apply -input=false development.tfplan

  # ─── Apply (Production) - Manual approval required ─────────
  apply-prod:
    name: Apply (production)
    runs-on: ubuntu-latest
    needs: apply-dev
    if: github.ref == 'refs/heads/main' && github.event_name == 'push'
    environment: production  # ← ต้องการ manual approval
    
    permissions:
      id-token: write
      contents: read
    
    steps:
      - uses: actions/checkout@v4

      - name: Configure AWS Credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ secrets.AWS_ROLE_PRODUCTION }}
          aws-region: ${{ env.AWS_REGION }}

      - name: Setup Terraform
        uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: ${{ env.TF_VERSION }}

      - name: Download Plan
        uses: actions/download-artifact@v3
        with:
          name: production-plan
          path: environments/production/

      - name: Terraform Init
        working-directory: environments/production
        run: terraform init -input=false

      - name: Terraform Apply
        working-directory: environments/production
        run: terraform apply -input=false production.tfplan
```

---

## Step 349: Code Review Checklist

### Terraform Code Review Checklist

```markdown
## Terraform Code Review Checklist

### Security
- [ ] ไม่มี hardcoded credentials (passwords, API keys, access keys)
- [ ] ไม่มี sensitive data ใน .tfvars ที่ commit เข้า repo
- [ ] Sensitive variables มี `sensitive = true`
- [ ] S3 buckets มี public access block enabled
- [ ] Security groups ไม่มี 0.0.0.0/0 สำหรับ ingress ports ที่ไม่จำเป็น
- [ ] RDS/databases ไม่มี publicly_accessible = true (ยกเว้นมีเหตุผล)
- [ ] Encryption ถูก enable ใน EBS, RDS, S3
- [ ] IMDSv2 ถูก enforce ใน EC2 instances
- [ ] IAM policies ใช้ least privilege principle

### Code Quality  
- [ ] `terraform fmt` ผ่าน
- [ ] `terraform validate` ผ่าน
- [ ] Variable ทุกตัวมี `description`
- [ ] Output ทุกตัวมี `description`
- [ ] Resources มี `tags` ที่ครบตาม tagging standard
- [ ] ไม่มี hardcoded values ที่ควรเป็น variables
- [ ] ใช้ `locals` เพื่อลดการซ้ำ code

### Architecture
- [ ] Dependencies ระหว่าง resources ถูกต้อง
- [ ] ไม่มี circular dependencies
- [ ] Module design มี single responsibility
- [ ] Outputs เพียงพอสำหรับ consumers

### Operations
- [ ] `lifecycle.prevent_destroy = true` สำหรับ stateful resources
- [ ] Backup/retention policies กำหนดไว้
- [ ] Monitoring/alerting สำหรับ resources สำคัญ
- [ ] State backend มี locking configured
```

---

## Step 350: Testing Strategy

### ระดับของ Testing

```
Testing Pyramid สำหรับ Terraform:

                    ┌─────────────┐
                    │  E2E Tests  │  (ทดสอบ infrastructure จริง)
                    │   (Slow)    │
                   /┴─────────────┴\
                  / Integration Test \  (module tests with real AWS)
                 /    (Medium)        \
                /──────────────────────\
               /    Unit Tests          \  (terraform validate, fmt, tflint)
              /       (Fast)             \
             └─────────────────────────────┘
```

### Unit Tests - terraform validate และ fmt

```bash
#!/bin/bash
# scripts/unit_tests.sh

set -e

echo "🧪 Running Terraform unit tests..."

# Find all terraform directories
TF_DIRS=$(find . -name "*.tf" -not -path "./.terraform/*" \
  | xargs -I {} dirname {} | sort -u)

PASS=0
FAIL=0

for dir in $TF_DIRS; do
  echo ""
  echo "Testing: $dir"
  
  # Check format
  if terraform fmt -check "$dir" > /dev/null 2>&1; then
    echo "  ✅ Format: OK"
    ((PASS++))
  else
    echo "  ❌ Format: FAILED (run: terraform fmt $dir)"
    ((FAIL++))
  fi
  
  # Validate (skip if has backend config)
  if terraform -chdir="$dir" init -backend=false > /dev/null 2>&1; then
    if terraform -chdir="$dir" validate > /dev/null 2>&1; then
      echo "  ✅ Validate: OK"
      ((PASS++))
    else
      echo "  ❌ Validate: FAILED"
      terraform -chdir="$dir" validate
      ((FAIL++))
    fi
  fi
done

echo ""
echo "Results: $PASS passed, $FAIL failed"

if [ $FAIL -gt 0 ]; then
  exit 1
fi
```

### Integration Tests - Terratest

```go
// tests/vpc_test.go - Terratest example

package test

import (
    "testing"
    "github.com/gruntwork-io/terratest/modules/terraform"
    "github.com/gruntwork-io/terratest/modules/aws"
    "github.com/stretchr/testify/assert"
)

func TestVPCModule(t *testing.T) {
    t.Parallel()

    // Setup Terraform options
    terraformOptions := terraform.WithDefaultRetryableErrors(t, &terraform.Options{
        TerraformDir: "../modules/vpc",
        
        Vars: map[string]interface{}{
            "project_name":         "test",
            "environment":          "testing",
            "vpc_cidr":             "10.99.0.0/16",
            "availability_zones":   []string{"ap-southeast-1a", "ap-southeast-1b"},
            "public_subnet_cidrs":  []string{"10.99.1.0/24", "10.99.2.0/24"},
            "private_subnet_cidrs": []string{"10.99.10.0/24", "10.99.11.0/24"},
        },
        
        EnvVars: map[string]string{
            "AWS_DEFAULT_REGION": "ap-southeast-1",
        },
    })

    // Cleanup after test
    defer terraform.Destroy(t, terraformOptions)

    // Apply
    terraform.InitAndApply(t, terraformOptions)

    // Get outputs
    vpcId := terraform.Output(t, terraformOptions, "vpc_id")
    publicSubnetIds := terraform.OutputList(t, terraformOptions, "public_subnet_ids")
    privateSubnetIds := terraform.OutputList(t, terraformOptions, "private_subnet_ids")

    // Assertions
    assert.NotEmpty(t, vpcId)
    assert.Equal(t, 2, len(publicSubnetIds))
    assert.Equal(t, 2, len(privateSubnetIds))

    // Verify VPC exists in AWS
    vpc := aws.GetVpcById(t, vpcId, "ap-southeast-1")
    assert.Equal(t, "10.99.0.0/16", aws.GetCidrBlockOfVpc(t, vpc))
    assert.True(t, *vpc.EnableDnsHostnames)
    assert.True(t, *vpc.EnableDnsSupport)
}
```

---

## สรุป: Best Practices Summary

### Quick Reference

| หัวข้อ | Best Practice |
|--------|--------------|
| **File Organization** | main.tf, variables.tf, outputs.tf, versions.tf, locals.tf, data.tf |
| **Naming** | snake_case สำหรับ TF names, kebab-case สำหรับ AWS resource names |
| **Secrets** | ห้าม hardcode, ใช้ IAM roles หรือ Secrets Manager |
| **Tags** | mandatory tags: Project, Environment, ManagedBy, Owner, CostCenter |
| **Version** | Pessimistic constraint (~>) สำหรับ providers |
| **State** | Remote backend + locking เสมอ |
| **Testing** | fmt + validate + tflint + tfsec อย่างน้อย |
| **CI/CD** | Plan on PR, Apply on merge, Manual approval for prod |
| **Review** | Code review checklist ทุกครั้ง |

---

*จบ Part 035: Terraform Configuration Best Practices*

*ต่อไป: Part 036 - AWS Provider Setup & Authentication*
