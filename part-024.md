# Part 24: State Storage & Backends (ขั้นตอนที่ 231-240)

## ภาพรวม (Overview)

Backend ใน Terraform กำหนดว่า state จะถูกเก็บที่ไหนและ operations จะถูกรันอย่างไร การเลือก backend ที่เหมาะสมเป็นสิ่งสำคัญสำหรับ team collaboration, security, และ reliability

---

## Step 231: Local Backend (Default)

### พื้นฐาน Local Backend

```hcl
# Local backend (default เมื่อไม่กำหนด)
terraform {
  backend "local" {
    path = "terraform.tfstate"
  }
}

# หรือกำหนด path เฉพาะ
terraform {
  backend "local" {
    path = "relative/path/to/terraform.tfstate"
  }
}
```

### เมื่อไรควรใช้ Local Backend

```bash
# ✅ เหมาะสำหรับ:
# - Learning/development/testing
# - Single developer projects
# - Local modules ที่ไม่ต้องการ collaboration

# ❌ ไม่เหมาะสำหรับ:
# - Team projects
# - Production infrastructure
# - CI/CD pipelines
```

---

## Step 232: S3 Backend - Complete Setup

### สร้าง S3 Bucket และ DynamoDB Table ก่อน

```hcl
# backend-setup/main.tf
# ⚠️ ต้องสร้างด้วย local backend ก่อน แล้วค่อย migrate

terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

provider "aws" {
  region = "us-east-1"
}

# S3 Bucket สำหรับเก็บ state
resource "aws_s3_bucket" "terraform_state" {
  bucket = "my-company-terraform-state-${data.aws_caller_identity.current.account_id}"

  # ป้องกันการลบโดยบังเอิญ
  lifecycle {
    prevent_destroy = true
  }

  tags = {
    Name        = "Terraform State Storage"
    Environment = "global"
    ManagedBy   = "terraform"
  }
}

# Enable Versioning (สำคัญมาก!)
resource "aws_s3_bucket_versioning" "terraform_state" {
  bucket = aws_s3_bucket.terraform_state.id

  versioning_configuration {
    status = "Enabled"
  }
}

# Enable Encryption
resource "aws_s3_bucket_server_side_encryption_configuration" "terraform_state" {
  bucket = aws_s3_bucket.terraform_state.id

  rule {
    apply_server_side_encryption_by_default {
      sse_algorithm     = "aws:kms"
      kms_master_key_id = aws_kms_key.terraform_state.arn
    }
    bucket_key_enabled = true
  }
}

# Block Public Access (สำคัญมากสำหรับ security!)
resource "aws_s3_bucket_public_access_block" "terraform_state" {
  bucket = aws_s3_bucket.terraform_state.id

  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}

# Enforce HTTPS only
resource "aws_s3_bucket_policy" "terraform_state" {
  bucket = aws_s3_bucket.terraform_state.id
  
  depends_on = [aws_s3_bucket_public_access_block.terraform_state]

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Sid       = "DenyHTTP"
        Effect    = "Deny"
        Principal = "*"
        Action    = "s3:*"
        Resource = [
          aws_s3_bucket.terraform_state.arn,
          "${aws_s3_bucket.terraform_state.arn}/*"
        ]
        Condition = {
          Bool = {
            "aws:SecureTransport" = "false"
          }
        }
      },
      {
        Sid    = "AllowTerraformAccess"
        Effect = "Allow"
        Principal = {
          AWS = "arn:aws:iam::${data.aws_caller_identity.current.account_id}:root"
        }
        Action = [
          "s3:GetObject",
          "s3:PutObject",
          "s3:ListBucket",
          "s3:DeleteObject"
        ]
        Resource = [
          aws_s3_bucket.terraform_state.arn,
          "${aws_s3_bucket.terraform_state.arn}/*"
        ]
      }
    ]
  })
}

# Lifecycle policy เก็บ old versions 90 วัน
resource "aws_s3_bucket_lifecycle_configuration" "terraform_state" {
  bucket = aws_s3_bucket.terraform_state.id

  rule {
    id     = "state-versioning-cleanup"
    status = "Enabled"

    noncurrent_version_transition {
      noncurrent_days = 30
      storage_class   = "STANDARD_IA"
    }

    noncurrent_version_expiration {
      noncurrent_days = 90
    }
  }
}

# KMS Key สำหรับ encryption
resource "aws_kms_key" "terraform_state" {
  description             = "KMS key for Terraform state encryption"
  deletion_window_in_days = 10
  enable_key_rotation     = true

  tags = {
    Name = "terraform-state-key"
  }
}

resource "aws_kms_alias" "terraform_state" {
  name          = "alias/terraform-state"
  target_key_id = aws_kms_key.terraform_state.key_id
}

# DynamoDB Table สำหรับ State Locking
resource "aws_dynamodb_table" "terraform_state_lock" {
  name         = "terraform-state-locks"
  billing_mode = "PAY_PER_REQUEST"
  hash_key     = "LockID"

  attribute {
    name = "LockID"
    type = "S"
  }

  # Enable encryption
  server_side_encryption {
    enabled     = true
    kms_key_arn = aws_kms_key.terraform_state.arn
  }

  # Enable Point-in-Time Recovery
  point_in_time_recovery {
    enabled = true
  }

  lifecycle {
    prevent_destroy = true
  }

  tags = {
    Name        = "Terraform State Lock Table"
    Environment = "global"
    ManagedBy   = "terraform"
  }
}

data "aws_caller_identity" "current" {}

# Outputs
output "state_bucket_name" {
  value = aws_s3_bucket.terraform_state.bucket
}

output "state_bucket_arn" {
  value = aws_s3_bucket.terraform_state.arn
}

output "lock_table_name" {
  value = aws_dynamodb_table.terraform_state_lock.name
}

output "kms_key_arn" {
  value = aws_kms_key.terraform_state.arn
}
```

### ใช้งาน S3 Backend

```hcl
# main project/terraform.tf
terraform {
  required_version = ">= 1.0"
  
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }

  backend "s3" {
    # S3 bucket ที่สร้างไว้
    bucket = "my-company-terraform-state-123456789012"
    
    # Path ของ state file ใน bucket
    # Best practice: <project>/<environment>/terraform.tfstate
    key = "myapp/production/terraform.tfstate"
    
    region = "us-east-1"
    
    # เปิดใช้ encryption
    encrypt = true
    
    # KMS key สำหรับ encryption
    kms_key_id = "arn:aws:kms:us-east-1:123456789012:key/xxxxx"
    
    # DynamoDB table สำหรับ locking
    dynamodb_table = "terraform-state-locks"
    
    # Assume role (ถ้าต้องการ)
    # role_arn = "arn:aws:iam::123456789012:role/TerraformStateRole"
    
    # Profile (ถ้าใช้ named profile)
    # profile = "production"
  }
}
```

### S3 Backend สำหรับหลาย Environments

```
State File Structure:
├── my-company-terraform-state/
│   ├── networking/
│   │   ├── dev/terraform.tfstate
│   │   ├── staging/terraform.tfstate
│   │   └── prod/terraform.tfstate
│   ├── databases/
│   │   ├── dev/terraform.tfstate
│   │   ├── staging/terraform.tfstate
│   │   └── prod/terraform.tfstate
│   └── applications/
│       ├── myapp/
│       │   ├── dev/terraform.tfstate
│       │   ├── staging/terraform.tfstate
│       │   └── prod/terraform.tfstate
│       └── anotherapp/
│           └── ...
```

```hcl
# Networking project - dev
terraform {
  backend "s3" {
    bucket         = "my-company-terraform-state-123456789012"
    key            = "networking/dev/terraform.tfstate"
    region         = "us-east-1"
    encrypt        = true
    dynamodb_table = "terraform-state-locks"
  }
}

# Application project - production
terraform {
  backend "s3" {
    bucket         = "my-company-terraform-state-123456789012"
    key            = "applications/myapp/production/terraform.tfstate"
    region         = "us-east-1"
    encrypt        = true
    dynamodb_table = "terraform-state-locks"
  }
}
```

---

## Step 233: S3 Backend - IAM Permissions

### IAM Policy สำหรับ Terraform State

```hcl
# IAM Policy สำหรับ Terraform user/role
data "aws_iam_policy_document" "terraform_state" {
  # S3 permissions
  statement {
    sid    = "TerraformStateS3"
    effect = "Allow"
    
    actions = [
      "s3:ListBucket",
      "s3:GetBucketVersioning"
    ]
    
    resources = ["arn:aws:s3:::my-company-terraform-state-*"]
  }

  statement {
    sid    = "TerraformStateS3Objects"
    effect = "Allow"
    
    actions = [
      "s3:GetObject",
      "s3:PutObject",
      "s3:DeleteObject"
    ]
    
    resources = ["arn:aws:s3:::my-company-terraform-state-*/*"]
  }

  # DynamoDB permissions
  statement {
    sid    = "TerraformStateDynamoDB"
    effect = "Allow"
    
    actions = [
      "dynamodb:GetItem",
      "dynamodb:PutItem",
      "dynamodb:DeleteItem",
      "dynamodb:DescribeTable"
    ]
    
    resources = ["arn:aws:dynamodb:us-east-1:*:table/terraform-state-locks"]
  }

  # KMS permissions
  statement {
    sid    = "TerraformStateKMS"
    effect = "Allow"
    
    actions = [
      "kms:GenerateDataKey",
      "kms:DescribeKey",
      "kms:Decrypt",
      "kms:Encrypt"
    ]
    
    resources = ["arn:aws:kms:us-east-1:*:key/*"]
    
    condition {
      test     = "StringLike"
      variable = "kms:RequestAlias"
      values   = ["alias/terraform-state"]
    }
  }
}

resource "aws_iam_policy" "terraform_state" {
  name        = "TerraformStateAccess"
  description = "Allow Terraform to manage state files"
  policy      = data.aws_iam_policy_document.terraform_state.json
}
```

---

## Step 234: Azure Backend (azurerm)

```hcl
# สร้าง Azure resources ก่อน
# Resource Group
resource "azurerm_resource_group" "tfstate" {
  name     = "terraform-state-rg"
  location = "East US"
}

# Storage Account
resource "azurerm_storage_account" "tfstate" {
  name                     = "tfstate${random_string.suffix.result}"
  resource_group_name      = azurerm_resource_group.tfstate.name
  location                 = azurerm_resource_group.tfstate.location
  account_tier             = "Standard"
  account_replication_type = "LRS"
  
  blob_properties {
    versioning_enabled = true
  }
  
  tags = {
    environment = "global"
    managed_by  = "terraform"
  }
}

# Storage Container
resource "azurerm_storage_container" "tfstate" {
  name                  = "tfstate"
  storage_account_name  = azurerm_storage_account.tfstate.name
  container_access_type = "private"
}

# ใช้ Azure backend
terraform {
  backend "azurerm" {
    resource_group_name  = "terraform-state-rg"
    storage_account_name = "tfstate12345678"
    container_name       = "tfstate"
    key                  = "production.terraform.tfstate"
    
    # Optional: subscription_id, tenant_id
    # subscription_id = "xxx"
    # tenant_id       = "xxx"
    
    # Authentication options:
    # use_msi = true  # Managed Service Identity
    # sas_token = var.sas_token
    # access_key = var.access_key
  }
}
```

---

## Step 235: GCS Backend (Google Cloud)

```hcl
# สร้าง GCS bucket ก่อน
resource "google_storage_bucket" "terraform_state" {
  name          = "my-project-terraform-state"
  location      = "US"
  force_destroy = false

  versioning {
    enabled = true
  }

  lifecycle_rule {
    action {
      type = "Delete"
    }
    condition {
      num_newer_versions = 10
    }
  }

  uniform_bucket_level_access = true
}

# ใช้ GCS backend
terraform {
  backend "gcs" {
    bucket  = "my-project-terraform-state"
    prefix  = "terraform/state"
    
    # Optional: credentials file
    # credentials = "path/to/credentials.json"
  }
}
```

---

## Step 236: Terraform Cloud Backend

```hcl
# Terraform Cloud/Enterprise backend
terraform {
  cloud {
    organization = "my-organization"
    
    workspaces {
      name = "my-app-production"
    }
  }
}

# หรือใช้กับ workspace tags
terraform {
  cloud {
    organization = "my-organization"
    
    workspaces {
      tags = ["app:myapp", "env:production"]
    }
  }
}

# Terraform Cloud backend (เก่ากว่า)
terraform {
  backend "remote" {
    organization = "my-organization"
    
    workspaces {
      name = "my-workspace"
    }
    
    # สำหรับ Enterprise
    # hostname = "terraform.company.com"
  }
}
```

### Terraform Cloud Configuration

```bash
# ตั้งค่า token สำหรับ Terraform Cloud
# ~/.terraform.d/credentials.tfrc.json
{
  "credentials": {
    "app.terraform.io": {
      "token": "xxxxx.atlasv1.xxxxxxxxxxxxx"
    }
  }
}

# หรือใช้ environment variable
export TFE_TOKEN="xxxxx.atlasv1.xxxxxxxxxxxxx"

# Login ด้วย CLI
terraform login
```

---

## Step 237: HTTP Backend และ Consul Backend

### HTTP Backend

```hcl
# HTTP backend - ใช้กับ custom HTTP server
terraform {
  backend "http" {
    address        = "https://my-server.example.com/terraform/state/production"
    lock_address   = "https://my-server.example.com/terraform/state/production/lock"
    unlock_address = "https://my-server.example.com/terraform/state/production/lock"
    
    lock_method   = "LOCK"
    unlock_method = "UNLOCK"
    
    username = "terraform"
    password = var.state_server_password
    
    # TLS
    # skip_cert_verification = false
    # client_certificate_pem = file("client.pem")
    # client_private_key_pem = file("client-key.pem")
  }
}
```

### Consul Backend

```hcl
terraform {
  backend "consul" {
    address = "consul.example.com:8500"
    scheme  = "https"
    path    = "terraform/state/production"
    
    access_token = var.consul_token
    
    # TLS
    # ca_file   = "ca.crt"
    # cert_file = "client.crt"
    # key_file  = "client.key"
    
    # ล็อคด้วย Consul sessions
    lock = true
    
    datacenter = "dc1"
  }
}
```

---

## Step 238: Partial Backend Configuration

### ทำไมต้องใช้ Partial Configuration?

```bash
# ❌ ปัญหา: ถ้าเก็บ credentials ใน backend config
terraform {
  backend "s3" {
    bucket     = "my-state-bucket"
    key        = "production/terraform.tfstate"
    region     = "us-east-1"
    access_key = "AKIAIOSFODNN7EXAMPLE"  # ❌ อย่าเก็บใน code!
    secret_key = "wJalrXUtnFEMI/K7MDENG..."  # ❌ อย่าเก็บใน code!
  }
}
```

### วิธีที่ 1: Backend Config File

```hcl
# terraform.tf (commit ไปได้)
terraform {
  backend "s3" {
    bucket = "my-state-bucket"
    key    = "production/terraform.tfstate"
    region = "us-east-1"
    # ไม่มี credentials
  }
}
```

```bash
# backend-secrets.hcl (ไม่ commit! เพิ่มใน .gitignore)
access_key = "AKIAIOSFODNN7EXAMPLE"
secret_key = "wJalrXUtnFEMI/K7MDENG..."
```

```bash
# Init พร้อม secret config
terraform init -backend-config="backend-secrets.hcl"
```

### วิธีที่ 2: Command Line Flags

```bash
# Init พร้อม -backend-config flags
terraform init \
  -backend-config="bucket=my-state-bucket" \
  -backend-config="key=production/terraform.tfstate" \
  -backend-config="region=us-east-1" \
  -backend-config="access_key=$TF_STATE_ACCESS_KEY" \
  -backend-config="secret_key=$TF_STATE_SECRET_KEY"
```

### วิธีที่ 3: Environment Variables

```bash
# AWS credentials จาก environment variables
export AWS_ACCESS_KEY_ID="AKIAIOSFODNN7EXAMPLE"
export AWS_SECRET_ACCESS_KEY="wJalrXUtnFEMI/K7MDENG..."
export AWS_DEFAULT_REGION="us-east-1"

# Terraform จะใช้ env vars อัตโนมัติ
terraform init
```

### วิธีที่ 4: AWS Profile

```hcl
terraform {
  backend "s3" {
    bucket  = "my-state-bucket"
    key     = "production/terraform.tfstate"
    region  = "us-east-1"
    profile = "production"  # AWS CLI profile
  }
}
```

### Pattern สำหรับ Multiple Environments

```hcl
# terraform.tf - ไม่มี key เลย
terraform {
  backend "s3" {
    bucket         = "my-state-bucket"
    region         = "us-east-1"
    encrypt        = true
    dynamodb_table = "terraform-locks"
    # ไม่มี key - จะ pass ตอน init
  }
}
```

```bash
# dev init
terraform init -backend-config="key=myapp/dev/terraform.tfstate"

# staging init
terraform init -backend-config="key=myapp/staging/terraform.tfstate"

# prod init
terraform init -backend-config="key=myapp/prod/terraform.tfstate"
```

---

## Step 239: Migrating State Between Backends

### Migrate จาก Local ไป S3

```bash
# ขั้นตอน:
# 1. Add S3 backend config ไปใน terraform.tf
# 2. Run terraform init
# 3. Terraform จะถามว่าต้องการ migrate หรือไม่

$ terraform init

Initializing the backend...
Do you want to copy existing state to the new backend?
  Pre-existing state was found while migrating the old "local" backend
  to the newly configured "s3" backend. No existing state was found
  in the newly configured "s3" backend.
  
  Do you want to copy this state to the new "s3" backend?
  Enter a value: yes

Successfully configured the backend "s3"!
```

### Migrate จาก S3 ไป S3 (เปลี่ยน bucket/key)

```bash
# วิธีที่ 1: ใช้ -migrate-state
# แก้ไข backend config ก่อน แล้วรัน:
terraform init -migrate-state

# วิธีที่ 2: Manual migration
# 1. Download state จาก source
terraform state pull > terraform.tfstate.backup

# 2. เปลี่ยน backend config
# 3. Reconfigure
terraform init -reconfigure

# 4. Upload state ไป destination
terraform state push terraform.tfstate.backup
```

### Migrate ไป Terraform Cloud

```bash
# 1. Login ไป Terraform Cloud
terraform login

# 2. เพิ่ม cloud block ใน terraform.tf
# terraform {
#   cloud {
#     organization = "my-org"
#     workspaces { name = "my-workspace" }
#   }
# }

# 3. Run init (จะถาม migrate)
terraform init

# 4. ยืนยัน migration
# Enter a value: yes
```

---

## Step 240: Backend Best Practices

### Complete Production S3 Backend Setup

```hcl
# 1. backend-setup/main.tf (run ครั้งเดียว)
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

provider "aws" {
  region = var.region
}

variable "region"      { default = "us-east-1" }
variable "company_name" { default = "mycompany" }
variable "account_id"   {}

locals {
  bucket_name = "${var.company_name}-terraform-state-${var.account_id}"
  table_name  = "terraform-state-locks"
}

# KMS Key
resource "aws_kms_key" "terraform" {
  description             = "Terraform State Encryption Key"
  deletion_window_in_days = 7
  enable_key_rotation     = true
  
  policy = data.aws_iam_policy_document.kms_policy.json

  lifecycle { prevent_destroy = true }
  
  tags = {
    Name = "terraform-state-kms"
    Use  = "terraform-state"
  }
}

data "aws_iam_policy_document" "kms_policy" {
  statement {
    sid     = "Enable IAM User Permissions"
    effect  = "Allow"
    principals {
      type        = "AWS"
      identifiers = ["arn:aws:iam::${var.account_id}:root"]
    }
    actions   = ["kms:*"]
    resources = ["*"]
  }
}

resource "aws_kms_alias" "terraform" {
  name          = "alias/terraform-state"
  target_key_id = aws_kms_key.terraform.key_id
}

# S3 Bucket
resource "aws_s3_bucket" "terraform_state" {
  bucket = local.bucket_name
  
  lifecycle { prevent_destroy = true }

  tags = {
    Name    = local.bucket_name
    Purpose = "Terraform State Storage"
  }
}

resource "aws_s3_bucket_versioning" "terraform_state" {
  bucket = aws_s3_bucket.terraform_state.id
  versioning_configuration { status = "Enabled" }
}

resource "aws_s3_bucket_server_side_encryption_configuration" "terraform_state" {
  bucket = aws_s3_bucket.terraform_state.id
  rule {
    apply_server_side_encryption_by_default {
      sse_algorithm     = "aws:kms"
      kms_master_key_id = aws_kms_key.terraform.arn
    }
    bucket_key_enabled = true
  }
}

resource "aws_s3_bucket_public_access_block" "terraform_state" {
  bucket                  = aws_s3_bucket.terraform_state.id
  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}

resource "aws_s3_bucket_logging" "terraform_state" {
  bucket        = aws_s3_bucket.terraform_state.id
  target_bucket = aws_s3_bucket.terraform_state.id
  target_prefix = "logs/"
}

resource "aws_s3_bucket_lifecycle_configuration" "terraform_state" {
  bucket = aws_s3_bucket.terraform_state.id
  
  rule {
    id     = "cleanup-old-versions"
    status = "Enabled"
    
    filter { prefix = "" }
    
    noncurrent_version_transition {
      noncurrent_days = 30
      storage_class   = "STANDARD_IA"
    }
    
    noncurrent_version_transition {
      noncurrent_days = 60
      storage_class   = "GLACIER"
    }
    
    noncurrent_version_expiration {
      noncurrent_days = 365
    }
  }
}

# DynamoDB Table
resource "aws_dynamodb_table" "terraform_locks" {
  name         = local.table_name
  billing_mode = "PAY_PER_REQUEST"
  hash_key     = "LockID"

  attribute {
    name = "LockID"
    type = "S"
  }

  server_side_encryption {
    enabled     = true
    kms_key_arn = aws_kms_key.terraform.arn
  }

  point_in_time_recovery { enabled = true }

  lifecycle { prevent_destroy = true }

  tags = {
    Name    = local.table_name
    Purpose = "Terraform State Locking"
  }
}

# Outputs
output "backend_config" {
  value = <<-EOT
    # Add this to your terraform.tf:
    
    terraform {
      backend "s3" {
        bucket         = "${aws_s3_bucket.terraform_state.bucket}"
        key            = "PROJECT/ENVIRONMENT/terraform.tfstate"
        region         = "${var.region}"
        encrypt        = true
        kms_key_id     = "${aws_kms_key.terraform.arn}"
        dynamodb_table = "${aws_dynamodb_table.terraform_locks.name}"
      }
    }
  EOT
}
```

### Backend Configuration per Environment

```bash
# environments/dev/backend.hcl
bucket         = "mycompany-terraform-state-123456789012"
key            = "myapp/dev/terraform.tfstate"
region         = "us-east-1"
encrypt        = true
dynamodb_table = "terraform-state-locks"

# environments/staging/backend.hcl
bucket         = "mycompany-terraform-state-123456789012"
key            = "myapp/staging/terraform.tfstate"
region         = "us-east-1"
encrypt        = true
dynamodb_table = "terraform-state-locks"

# environments/prod/backend.hcl
bucket         = "mycompany-terraform-state-123456789012"
key            = "myapp/prod/terraform.tfstate"
region         = "us-east-1"
encrypt        = true
kms_key_id     = "arn:aws:kms:us-east-1:123456789012:key/xxxxx"
dynamodb_table = "terraform-state-locks"
role_arn       = "arn:aws:iam::123456789012:role/TerraformProdRole"
```

```bash
# ใช้งาน
terraform init -backend-config="environments/dev/backend.hcl"
terraform init -backend-config="environments/prod/backend.hcl"
```

### Backend Configuration ด้วย Makefile

```makefile
# Makefile
ENV ?= dev

init-dev:
	terraform init -backend-config="environments/dev/backend.hcl" -reconfigure

init-staging:
	terraform init -backend-config="environments/staging/backend.hcl" -reconfigure

init-prod:
	terraform init -backend-config="environments/prod/backend.hcl" -reconfigure

plan:
	terraform plan -var-file="environments/$(ENV)/terraform.tfvars"

apply:
	terraform apply -var-file="environments/$(ENV)/terraform.tfvars"
```

### Backend ใน CI/CD

```yaml
# .github/workflows/terraform.yml
name: Terraform

on:
  push:
    branches: [main]
  pull_request:

jobs:
  terraform:
    runs-on: ubuntu-latest
    
    env:
      AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
      AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
      TF_BACKEND_BUCKET: ${{ secrets.TF_STATE_BUCKET }}
      TF_BACKEND_KEY: "myapp/${{ github.ref_name }}/terraform.tfstate"
    
    steps:
      - uses: actions/checkout@v3
      
      - uses: hashicorp/setup-terraform@v2
        with:
          terraform_version: "1.6.0"
      
      - name: Terraform Init
        run: |
          terraform init \
            -backend-config="bucket=$TF_BACKEND_BUCKET" \
            -backend-config="key=$TF_BACKEND_KEY" \
            -backend-config="region=us-east-1" \
            -backend-config="encrypt=true" \
            -backend-config="dynamodb_table=terraform-state-locks"
      
      - name: Terraform Plan
        run: terraform plan -out=tfplan
      
      - name: Terraform Apply
        if: github.ref == 'refs/heads/main'
        run: terraform apply tfplan
```

---

## State Encryption Summary

```
Encryption Options:
├── S3 SSE-S3 (AES-256): เข้ารหัสที่ S3 ด้วย S3-managed keys
│   encrypt = true
│
├── S3 SSE-KMS: เข้ารหัสด้วย KMS key (แนะนำ)
│   encrypt = true
│   kms_key_id = "arn:aws:kms:..."
│
├── Azure Storage: เข้ารหัส at-rest โดยอัตโนมัติ
│
└── GCS: เข้ารหัส at-rest โดยอัตโนมัติ
```

---

## แบบฝึกหัด (Exercises)

### Exercise 1: Setup S3 Backend

1. สร้าง S3 bucket พร้อม versioning และ encryption
2. สร้าง DynamoDB table สำหรับ locking
3. Configure backend ใน terraform.tf
4. Run terraform init เพื่อ migrate state

### Exercise 2: Partial Configuration

1. แยก sensitive values ออกไปเป็น backend.hcl
2. เพิ่ม backend.hcl ใน .gitignore
3. สร้าง init script ที่ใช้ -backend-config

### Exercise 3: Multi-Environment

1. สร้าง backend config files สำหรับ dev/staging/prod
2. สร้าง Makefile commands สำหรับแต่ละ environment
3. ทดสอบ init ด้วยแต่ละ environment

---

## Checklist

- [ ] เข้าใจความแตกต่างระหว่าง backend types
- [ ] สามารถ setup S3 backend พร้อม DynamoDB locking ได้
- [ ] รู้วิธี migrate state ระหว่าง backends
- [ ] เข้าใจ partial backend configuration
- [ ] รู้วิธี secure state file
- [ ] สามารถ setup CI/CD กับ remote backend ได้
- [ ] เข้าใจ IAM permissions ที่จำเป็นสำหรับ S3 backend
