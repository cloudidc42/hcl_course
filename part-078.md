# Part 078: Terraform Performance & Scale (ขั้นตอนที่ 771-780)

## บทนำ (Introduction)

เมื่อ infrastructure เติบโตขึ้น Terraform อาจเริ่มช้าและยากต่อการจัดการ
บทนี้จะครอบคลุมกลยุทธ์การ optimize performance และ scale

---

## ขั้นตอนที่ 771: Terraform Performance Challenges at Scale

### สัญญาณที่บ่งบอกว่ามีปัญหา Performance

```
❌ Terraform plan ใช้เวลา > 5 นาที
❌ State file ใหญ่กว่า 10MB
❌ terraform state list แสดง > 500 resources
❌ ทีมหลายทีมรอ unlock state ของกันและกัน
❌ Apply ใน production ใช้เวลา > 30 นาที
❌ Provider initialization ใช้เวลานาน
```

### Root Causes

```
1. State file ใหญ่เกินไป
   - ทุกอย่างอยู่ใน state เดียว
   - ไม่มีการแบ่ง state ตาม component

2. Refresh ทุก resource ทุกครั้ง
   - Terraform ต้อง API call ไปยัง provider
   - AWS: describe-instances, describe-vpcs, etc.

3. Provider ไม่มี cache
   - ดาวน์โหลด providers ใหม่ทุกครั้ง

4. Parallelism ต่ำเกินไป (default 10)

5. Large, complex modules
```

---

## ขั้นตอนที่ 772: Parallelism Tuning

### -parallelism flag

```bash
# Default: 10 parallel operations
terraform apply

# เพิ่ม parallelism สำหรับ resources ที่ independent กัน
terraform apply -parallelism=20

# ลด parallelism ถ้า API rate limits เป็นปัญหา
terraform apply -parallelism=5

# สำหรับ AWS: โดยทั่วไป 20-50 ปลอดภัย
terraform plan -parallelism=50
terraform apply -parallelism=50
```

### Parallelism vs API Rate Limits

```bash
# AWS Rate Limits ที่ควรระวัง:
# EC2: 100 req/sec (แต่บาง API น้อยกว่า)
# IAM: 100 req/sec
# S3: ไม่มี limit ชัดเจน

# ถ้าเจอ "RequestLimitExceeded" ให้ลด parallelism
# หรือเพิ่ม retry configuration ใน provider

provider "aws" {
  region = "us-east-1"

  # Retry configuration
  retry_mode   = "adaptive"  # หรือ "standard"
  max_retries  = 10
}
```

---

## ขั้นตอนที่ 773: -refresh=false สำหรับ Speed

### ทำความเข้าใจ refresh

```bash
# ปกติ Terraform ทำ 2 สิ่งก่อน plan:
# 1. Refresh state (API calls ไปยัง provider)
# 2. Compare refreshed state กับ config

# -refresh=false: ข้าม step 1
# เร็วขึ้นมาก แต่ state อาจไม่ตรงกับความจริง

# ใช้เมื่อ:
# - ทราบว่าไม่มีใคร change infrastructure นอก Terraform
# - ต้องการ plan เร็วๆ สำหรับ review
# - ใน CI/CD ที่ infrastructure controlled

terraform plan -refresh=false
terraform apply -refresh=false
```

### Refresh Only สำหรับ Drift Detection

```bash
# ตรวจสอบว่ามี drift หรือไม่ โดยไม่ apply changes
terraform plan -refresh-only

# Apply เฉพาะ refresh (อัพเดท state ให้ตรงกับ reality)
terraform apply -refresh-only
```

---

## ขั้นตอนที่ 774: Partial Target Applies

### -target flag

```bash
# Apply เฉพาะ resource ที่ระบุ
terraform apply -target=aws_instance.web_server

# Apply เฉพาะ module
terraform apply -target=module.networking

# Apply หลาย targets
terraform apply \
  -target=aws_vpc.main \
  -target=aws_subnet.public_1 \
  -target=aws_subnet.public_2

# Plan เฉพาะ target
terraform plan -target=module.database
```

### ควรใช้ -target เมื่อไหร่?

```bash
# ✓ ใช้ได้:
# - Debug specific resource
# - Apply urgent hotfix สำหรับ resource เดียว
# - Bootstrap infrastructure ทีละ step

# ✗ ไม่ควรใช้:
# - ทำ routine operations (อาจ miss dependencies)
# - ใน automated CI/CD
# - เพื่อ bypass Sentinel policies
```

---

## ขั้นตอนที่ 775: State Splitting Strategies

### Strategy 1: แบ่งตาม Environment

```
states/
├── dev/
│   └── terraform.tfstate    (dev infrastructure)
├── staging/
│   └── terraform.tfstate    (staging infrastructure)
└── prod/
    └── terraform.tfstate    (prod infrastructure)
```

```hcl
# environments/prod/backend.tf
terraform {
  backend "s3" {
    bucket         = "mycompany-terraform-state"
    key            = "prod/terraform.tfstate"
    region         = "us-east-1"
    encrypt        = true
    dynamodb_table = "terraform-state-lock"
  }
}
```

### Strategy 2: แบ่งตาม Component

```
states/
├── network/
│   └── terraform.tfstate     (VPC, subnets, routing)
├── security/
│   └── terraform.tfstate     (IAM, security groups, WAF)
├── compute/
│   └── terraform.tfstate     (EC2, ECS, Lambda)
├── database/
│   └── terraform.tfstate     (RDS, ElastiCache, DynamoDB)
├── storage/
│   └── terraform.tfstate     (S3, EFS)
└── monitoring/
    └── terraform.tfstate     (CloudWatch, alerts)
```

```hcl
# network/main.tf
terraform {
  backend "s3" {
    bucket = "mycompany-terraform-state"
    key    = "prod/network/terraform.tfstate"
    region = "us-east-1"
  }
}

# Outputs สำหรับ share กับ other components
output "vpc_id" {
  value = aws_vpc.main.id
}

output "private_subnet_ids" {
  value = aws_subnet.private[*].id
}
```

```hcl
# compute/main.tf
terraform {
  backend "s3" {
    bucket = "mycompany-terraform-state"
    key    = "prod/compute/terraform.tfstate"
    region = "us-east-1"
  }
}

# อ่าน outputs จาก network state
data "terraform_remote_state" "network" {
  backend = "s3"

  config = {
    bucket = "mycompany-terraform-state"
    key    = "prod/network/terraform.tfstate"
    region = "us-east-1"
  }
}

resource "aws_instance" "app" {
  ami           = var.ami_id
  instance_type = var.instance_type
  subnet_id     = data.terraform_remote_state.network.outputs.private_subnet_ids[0]
}
```

### Strategy 3: แบ่งตาม Team Ownership

```
states/
├── platform-team/
│   ├── networking/
│   ├── security-baseline/
│   └── shared-services/
├── app-team-a/
│   ├── frontend/
│   └── backend/
└── data-team/
    ├── warehouse/
    └── streaming/
```

### Strategy 4: แบ่งตาม Region

```
states/
├── us-east-1/
│   ├── network/
│   └── compute/
├── us-west-2/
│   ├── network/
│   └── compute/
└── eu-west-1/
    ├── network/
    └── compute/
```

---

## ขั้นตอนที่ 776: Provider Plugin Caching

### TF_PLUGIN_CACHE_DIR

```bash
# Set environment variable
export TF_PLUGIN_CACHE_DIR="$HOME/.terraform.d/plugin-cache"
mkdir -p "$TF_PLUGIN_CACHE_DIR"

# หรือใน ~/.terraformrc
provider_installation {
  filesystem_mirror {
    path    = "/home/user/.terraform.d/plugin-cache"
    include = ["registry.terraform.io/hashicorp/*"]
  }

  direct {
    exclude = ["registry.terraform.io/hashicorp/*"]
  }
}
```

### Shared Plugin Cache สำหรับ Team/CI

```bash
# ใน CI/CD pipeline (GitHub Actions)
# Cache provider plugins ระหว่าง runs

# .github/workflows/terraform.yml
      - name: Cache Terraform providers
        uses: actions/cache@v3
        with:
          path: ~/.terraform.d/plugin-cache
          key: terraform-providers-${{ hashFiles('**/.terraform.lock.hcl') }}
          restore-keys: |
            terraform-providers-

      - name: Set Terraform plugin cache
        run: echo "TF_PLUGIN_CACHE_DIR=$HOME/.terraform.d/plugin-cache" >> $GITHUB_ENV

      - name: Terraform Init
        run: terraform init
```

### Pre-install Providers สำหรับ Air-Gapped

```bash
# สร้าง local mirror
MIRROR_DIR="/opt/terraform/providers"
mkdir -p "$MIRROR_DIR"

# ดาวน์โหลด AWS provider
terraform providers mirror "$MIRROR_DIR"

# ใน CI สำหรับ air-gapped environment:
cat >> ~/.terraformrc << 'EOF'
provider_installation {
  filesystem_mirror {
    path = "/opt/terraform/providers"
  }

  direct {
    exclude = ["*/*/*"]
  }
}
EOF
```

---

## ขั้นตอนที่ 777: Terragrunt สำหรับ DRY State Management

### ทำไมต้อง Terragrunt?

```
ปัญหาของ pure Terraform ที่ scale:
- Backend configuration ต้อง copy ทุก directory
- Provider configuration ซ้ำกันมาก
- Variables ซ้ำกันระหว่าง environments
- ต้องรัน terraform apply ทีละ directory

Terragrunt แก้ปัญหา:
- DRY backend configuration
- Automatic dependency management
- Run-all commands
- Hierarchical configuration (inheritance)
```

### โครงสร้าง Terragrunt

```
infrastructure/
├── terragrunt.hcl            # root config
├── _envcommon/
│   ├── networking.hcl        # shared networking config
│   └── compute.hcl           # shared compute config
├── prod/
│   ├── account.hcl
│   ├── networking/
│   │   └── terragrunt.hcl
│   ├── compute/
│   │   └── terragrunt.hcl
│   └── database/
│       └── terragrunt.hcl
└── staging/
    ├── account.hcl
    ├── networking/
    │   └── terragrunt.hcl
    └── compute/
        └── terragrunt.hcl
```

### root terragrunt.hcl

```hcl
# infrastructure/terragrunt.hcl

# Generate backend.tf สำหรับทุก module โดยอัตโนมัติ
remote_state {
  backend = "s3"

  generate = {
    path      = "backend.tf"
    if_exists = "overwrite_terragrunt"
  }

  config = {
    bucket         = "mycompany-terraform-state-${local.account_id}"
    key            = "${path_relative_to_include()}/terraform.tfstate"
    region         = "us-east-1"
    encrypt        = true
    dynamodb_table = "terraform-state-lock"

    s3_bucket_tags = {
      Environment = local.environment
      ManagedBy   = "terragrunt"
    }
  }
}

# Generate provider.tf สำหรับทุก module
generate "provider" {
  path      = "provider.tf"
  if_exists = "overwrite_terragrunt"

  contents = <<EOF
provider "aws" {
  region = "${local.aws_region}"

  default_tags {
    tags = {
      Environment = "${local.environment}"
      ManagedBy   = "terraform"
      Account     = "${local.account_id}"
    }
  }
}
EOF
}

# Global inputs สำหรับทุก module
inputs = {
  aws_region  = local.aws_region
  environment = local.environment
  account_id  = local.account_id
}

# Local variables
locals {
  # อ่าน account-specific config
  account_vars = read_terragrunt_config(find_in_parent_folders("account.hcl"))

  # อ่าน environment-specific config
  environment_vars = read_terragrunt_config(find_in_parent_folders("env.hcl"))

  # Extract values
  account_id  = local.account_vars.locals.aws_account_id
  aws_region  = local.account_vars.locals.aws_region
  environment = local.environment_vars.locals.environment
}
```

### prod/networking/terragrunt.hcl

```hcl
# infrastructure/prod/networking/terragrunt.hcl

terraform {
  source = "git::https://github.com/mycompany/terraform-modules.git//networking?ref=v2.0.0"
}

# Inherit root config
include "root" {
  path = find_in_parent_folders()
}

# Include environment-specific common config
include "envcommon" {
  path   = "${dirname(find_in_parent_folders())}/_envcommon/networking.hcl"
  expose = true
}

# Module-specific inputs
inputs = {
  vpc_cidr             = "10.0.0.0/16"
  public_subnet_cidrs  = ["10.0.1.0/24", "10.0.2.0/24"]
  private_subnet_cidrs = ["10.0.10.0/24", "10.0.11.0/24"]
  enable_nat_gateway   = true
  single_nat_gateway   = false  # prod: one NAT per AZ
}
```

### prod/compute/terragrunt.hcl

```hcl
# infrastructure/prod/compute/terragrunt.hcl

terraform {
  source = "git::https://github.com/mycompany/terraform-modules.git//compute?ref=v2.0.0"
}

include "root" {
  path = find_in_parent_folders()
}

# Dependency management
dependency "networking" {
  config_path = "../networking"

  # Mock outputs สำหรับ plan โดยไม่ต้อง apply networking ก่อน
  mock_outputs_allowed_terraform_commands = ["validate", "plan"]
  mock_outputs = {
    vpc_id             = "vpc-00000000"
    private_subnet_ids = ["subnet-00000000", "subnet-00000001"]
  }
}

inputs = {
  vpc_id             = dependency.networking.outputs.vpc_id
  subnet_ids         = dependency.networking.outputs.private_subnet_ids
  instance_type      = "t3.medium"
  min_capacity       = 2
  max_capacity       = 10
  desired_capacity   = 3
}
```

### รัน Terragrunt

```bash
# รัน apply สำหรับ single module
cd infrastructure/prod/networking
terragrunt apply

# รัน apply สำหรับทุก modules ใน prod (ตามลำดับ dependency)
cd infrastructure/prod
terragrunt run-all apply

# Plan ทุก modules
terragrunt run-all plan

# Destroy ทุก modules (ระวัง!)
terragrunt run-all destroy

# รัน เฉพาะ modules ที่เปลี่ยนแปลง
terragrunt run-all apply --terragrunt-include-dir "prod/networking" \
                         --terragrunt-include-dir "prod/compute"

# ดู dependency graph
terragrunt graph-dependencies
```

---

## ขั้นตอนที่ 778: State Locking และ Concurrency

### ปัญหา Concurrent Access

```bash
# ปัญหา: สองคนรัน terraform apply พร้อมกัน
# คนที่ 1: ได้ lock → apply สำเร็จ
# คนที่ 2: รอ lock → timeout หรือ error

Error: Error acquiring the state lock

Error message: ConditionalCheckFailedException: The conditional request failed
Lock Info:
  ID:        a7a5a4e7-3834-4b29-ab29-d4f4c39f6b56
  Path:      terraform-state/prod/terraform.tfstate
  Operation: OperationTypeApply
  Who:       alice@mycompany.com
  Version:   1.7.0
  Created:   2024-01-15 10:30:00 UTC
  Info:
```

### Force Unlock (ระวัง!)

```bash
# ใช้เฉพาะเมื่อ:
# 1. แน่ใจว่าไม่มีการ apply อยู่จริง
# 2. Lock ID ได้จาก error message

terraform force-unlock a7a5a4e7-3834-4b29-ab29-d4f4c39f6b56

# ยืนยันก่อน unlock
# Enter "yes" to confirm: yes
```

### Backend Configuration สำหรับ Locking

```hcl
# S3 + DynamoDB backend (แนะนำสำหรับ AWS)
terraform {
  backend "s3" {
    bucket         = "mycompany-terraform-state"
    key            = "prod/terraform.tfstate"
    region         = "us-east-1"
    encrypt        = true

    # DynamoDB สำหรับ locking
    dynamodb_table = "terraform-state-lock"
  }
}
```

```hcl
# สร้าง DynamoDB table สำหรับ state locking
resource "aws_dynamodb_table" "terraform_lock" {
  name         = "terraform-state-lock"
  billing_mode = "PAY_PER_REQUEST"
  hash_key     = "LockID"

  attribute {
    name = "LockID"
    type = "S"
  }

  tags = {
    Name    = "Terraform State Lock"
    Purpose = "terraform-state-locking"
  }
}
```

---

## ขั้นตอนที่ 779: Backend Performance Comparison

### เปรียบเทียบ Backend Options

| Backend | Locking | Versioning | Cost | Speed | Recommendation |
|---------|---------|------------|------|-------|----------------|
| local | No | No | Free | Fast | Dev only |
| S3 + DynamoDB | Yes | Yes | Low | Medium | AWS Teams |
| GCS | Yes | Yes | Low | Medium | GCP Teams |
| Azure Blob | Yes | Yes | Low | Medium | Azure Teams |
| Terraform Cloud | Yes | Yes | Free/Paid | Fast | Any team |
| PostgreSQL | Yes | No | Medium | Fast | Custom |
| Consul | Yes | No | Medium | Fast | HashiCorp Stack |

### S3 Backend Optimization

```hcl
terraform {
  backend "s3" {
    bucket = "mycompany-terraform-state"
    key    = "prod/networking/terraform.tfstate"
    region = "us-east-1"

    # ใช้ S3 Transfer Acceleration สำหรับ global teams
    # use_accelerate_endpoint = true  # ต้อง enable acceleration บน bucket

    # ใช้ specific endpoint สำหรับ VPC endpoint
    # endpoint = "https://bucket.vpce-xxx.s3.us-east-1.vpce.amazonaws.com"

    encrypt = true

    # KMS encryption
    kms_key_id = "arn:aws:kms:us-east-1:123456789012:key/xxx"

    # Role สำหรับ backend access (ดีกว่าใช้ access keys)
    role_arn = "arn:aws:iam::123456789012:role/terraform-state-role"

    dynamodb_table = "terraform-state-lock"
    dynamodb_endpoint = "https://dynamodb.us-east-1.amazonaws.com"
  }
}
```

---

## ขั้นตอนที่ 780: Large Team Patterns และ Blast Radius

### Blast Radius Reduction

```
Blast Radius = ขนาดความเสียหายถ้า terraform apply ผิดพลาด

Small blast radius = ดี
- apply เปลี่ยนเฉพาะ network layer
- ถ้าพัง เฉพาะ network เสีย

Large blast radius = เสี่ยง
- apply เปลี่ยนทุกอย่างพร้อมกัน
- ถ้าพัง ทุกอย่างเสียหมด
```

### แบ่ง State เพื่อลด Blast Radius

```
Production State Structure (Optimal):
└── prod/
    ├── networking/      (1 state) - VPC, subnets, routing
    │   └── ~50 resources
    ├── security/        (1 state) - IAM, KMS, security groups
    │   └── ~100 resources
    ├── compute/         (1 state) - EC2, ASG, ALB
    │   └── ~150 resources
    ├── database/        (1 state) - RDS, ElastiCache
    │   └── ~30 resources
    ├── storage/         (1 state) - S3, EFS
    │   └── ~20 resources
    └── monitoring/      (1 state) - CloudWatch, alerts
        └── ~50 resources

Total: 6 states × ~70 resources each (manageable)
VS: 1 state × 400+ resources (slow, risky)
```

### Remote Runs สำหรับ Consistent Environment

```hcl
# Terraform Cloud - Remote execution
terraform {
  cloud {
    organization = "mycompany"

    workspaces {
      name = "networking-prod"
    }
  }
}
```

```
Remote Execution ดีกว่า Local Execution เพราะ:
1. Consistent environment (OS, Terraform version, providers)
2. No drift จาก developer laptop differences
3. Audit trail สำหรับ compliance
4. State access control
5. Notifications และ approvals
```

### Pattern: Mono-repo กับ Multiple States

```
infrastructure/
├── .github/
│   └── workflows/
│       ├── networking.yml    # trigger เฉพาะ path: environments/*/networking/
│       ├── compute.yml       # trigger เฉพาะ path: environments/*/compute/
│       └── database.yml
├── modules/
│   ├── networking/
│   └── compute/
└── environments/
    ├── prod/
    │   ├── networking/
    │   ├── compute/
    │   └── database/
    └── staging/
        ├── networking/
        └── compute/
```

```yaml
# .github/workflows/networking.yml
name: Networking Deployment

on:
  push:
    branches: [main]
    paths:
      - 'environments/*/networking/**'
      - 'modules/networking/**'

  pull_request:
    paths:
      - 'environments/*/networking/**'
      - 'modules/networking/**'

jobs:
  terraform:
    runs-on: ubuntu-latest

    strategy:
      matrix:
        environment: [staging, prod]

    steps:
      - uses: actions/checkout@v4

      - name: Setup Terraform
        uses: hashicorp/setup-terraform@v3

      - name: Terraform Plan
        working-directory: environments/${{ matrix.environment }}/networking
        run: |
          terraform init
          terraform plan -out=tfplan.binary

      - name: Terraform Apply
        if: github.ref == 'refs/heads/main' && github.event_name == 'push'
        working-directory: environments/${{ matrix.environment }}/networking
        run: terraform apply tfplan.binary
```

---

## Summary: Performance Checklist

```markdown
## Terraform Performance Checklist

### State Management
- [ ] State แบ่งตาม component/team/environment
- [ ] State size < 10MB ต่อ state
- [ ] Resources ต่อ state < 200
- [ ] Backend supports locking (DynamoDB, TFC, etc.)

### Execution Speed
- [ ] ใช้ -parallelism=20-50 (ทดสอบก่อน)
- [ ] Provider plugin cache enabled
- [ ] ใช้ -refresh=false เมื่อเหมาะสม
- [ ] Remote execution (TFC/TFE) สำหรับ consistency

### Architecture
- [ ] Modules ขนาดพอเหมาะ (ไม่ใหญ่เกินไป)
- [ ] State dependencies minimize ด้วย outputs
- [ ] Blast radius ลดด้วยการแบ่ง states
- [ ] Team ownership ชัดเจน

### Tools
- [ ] Terragrunt สำหรับ DRY configuration
- [ ] terraform.lock.hcl committed
- [ ] Provider versions pinned
```

---

*จบ Part 078 - ในส่วนถัดไปจะเรียนรู้เรื่อง Terraform State Advanced Operations*
