# Part 29: Terraform Workspaces (ขั้นตอนที่ 281-290)

## ภาพรวม (Overview)

Terraform Workspaces ช่วยให้จัดการหลาย environments (dev/staging/prod) จาก codebase เดียวกัน โดยแต่ละ workspace มี state file แยกจากกัน เหมาะสำหรับ scenarios ที่ต้องการ infrastructure เหมือนกัน แต่มีค่าต่างกัน

---

## Step 281: What Are Workspaces?

### Concept

```
Without workspaces:
├── dev infrastructure -> dev/terraform.tfstate
├── staging infrastructure -> staging/terraform.tfstate
└── prod infrastructure -> prod/terraform.tfstate

With workspaces (from single directory):
├── workspace: dev     -> terraform.tfstate.d/dev/terraform.tfstate
├── workspace: staging -> terraform.tfstate.d/staging/terraform.tfstate
└── workspace: prod    -> terraform.tfstate.d/prod/terraform.tfstate
```

### Default Workspace

```bash
# เมื่อ init ใหม่ จะอยู่ใน "default" workspace เสมอ
$ terraform workspace show
default

# State ของ default workspace:
# Local: ./terraform.tfstate
# S3: s3://bucket/key/terraform.tfstate
```

---

## Step 282: terraform workspace Commands

### terraform workspace new

```bash
# สร้าง workspace ใหม่และ switch ไป
terraform workspace new dev
# Created and switched to workspace "dev"!

terraform workspace new staging
terraform workspace new production
terraform workspace new feature-user-auth
terraform workspace new hotfix-security-patch

# สร้างและ copy state จาก workspace ปัจจุบัน
terraform workspace new dev -state=existing-state.tfstate
```

### terraform workspace list

```bash
# แสดงรายการ workspaces ทั้งหมด
terraform workspace list

# ตัวอย่าง output:
  default
* dev       <- asterisk หมายถึง workspace ปัจจุบัน
  staging
  production
```

### terraform workspace select

```bash
# Switch ไป workspace อื่น
terraform workspace select dev
# Switched to workspace "dev".

terraform workspace select production
# Switched to workspace "production".

# ตรวจสอบ workspace ปัจจุบัน
terraform workspace show
# production
```

### terraform workspace show

```bash
# แสดง workspace ปัจจุบัน
terraform workspace show
# dev
```

### terraform workspace delete

```bash
# ลบ workspace
terraform workspace delete dev

# ⚠️ ไม่สามารถลบ workspace ปัจจุบันได้
# ต้อง switch ออกก่อน

# ⚠️ ไม่สามารถลบ workspace ที่มี resources อยู่ใน state
# ต้อง destroy ก่อน

# Force delete (ถ้า state ว่าง)
terraform workspace delete -force dev
```

---

## Step 283: terraform.workspace Reference

### ใช้ workspace name ใน Configuration

```hcl
# terraform.workspace คือ built-in value
# Returns: "default", "dev", "staging", "production", etc.

# ใช้ใน resource names
resource "aws_vpc" "main" {
  cidr_block = "10.0.0.0/16"
  
  tags = {
    Name        = "${terraform.workspace}-vpc"
    Environment = terraform.workspace
  }
}

resource "aws_s3_bucket" "data" {
  bucket = "my-app-${terraform.workspace}-data"
  
  tags = {
    Environment = terraform.workspace
  }
}

# ใช้ใน local values
locals {
  env = terraform.workspace
  is_prod = terraform.workspace == "production"
}

resource "aws_instance" "app" {
  instance_type = local.is_prod ? "t3.large" : "t3.micro"
  
  tags = {
    Environment = local.env
    Name        = "${local.env}-app-server"
  }
}
```

---

## Step 284: Workspace-Based Resource Configuration

### Pattern 1: Workspace-Based Variable Selection

```hcl
# variables.tf
variable "workspace_config" {
  description = "Configuration per workspace"
  type = map(object({
    instance_type   = string
    instance_count  = number
    db_instance_class = string
    min_capacity    = number
    max_capacity    = number
    enable_monitoring = bool
  }))
  
  default = {
    default = {
      instance_type      = "t3.micro"
      instance_count     = 1
      db_instance_class  = "db.t3.micro"
      min_capacity       = 1
      max_capacity       = 2
      enable_monitoring  = false
    }
    dev = {
      instance_type      = "t3.micro"
      instance_count     = 1
      db_instance_class  = "db.t3.micro"
      min_capacity       = 1
      max_capacity       = 2
      enable_monitoring  = false
    }
    staging = {
      instance_type      = "t3.small"
      instance_count     = 2
      db_instance_class  = "db.t3.small"
      min_capacity       = 2
      max_capacity       = 4
      enable_monitoring  = true
    }
    production = {
      instance_type      = "t3.medium"
      instance_count     = 3
      db_instance_class  = "db.r5.large"
      min_capacity       = 3
      max_capacity       = 10
      enable_monitoring  = true
    }
  }
}

# main.tf
locals {
  # เลือก config ตาม workspace
  config = var.workspace_config[terraform.workspace]
}

resource "aws_instance" "app" {
  count         = local.config.instance_count
  ami           = data.aws_ami.ubuntu.id
  instance_type = local.config.instance_type

  tags = {
    Name        = "${terraform.workspace}-app-${count.index + 1}"
    Environment = terraform.workspace
  }
}

resource "aws_db_instance" "main" {
  identifier     = "${terraform.workspace}-database"
  engine         = "postgres"
  instance_class = local.config.db_instance_class
  
  tags = {
    Environment = terraform.workspace
  }
}

resource "aws_cloudwatch_metric_alarm" "high_cpu" {
  count = local.config.enable_monitoring ? 1 : 0
  
  alarm_name          = "${terraform.workspace}-high-cpu"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = "2"
  metric_name         = "CPUUtilization"
  namespace           = "AWS/EC2"
  period              = "120"
  statistic           = "Average"
  threshold           = "80"
}
```

### Pattern 2: Workspace-Based Resource Naming

```hcl
locals {
  prefix     = terraform.workspace
  is_prod    = terraform.workspace == "production"
  is_staging = terraform.workspace == "staging"
  is_dev     = terraform.workspace == "dev" || terraform.workspace == "default"
  
  # Resource names
  vpc_name           = "${local.prefix}-vpc"
  cluster_name       = "${local.prefix}-eks"
  db_identifier      = "${local.prefix}-postgres"
  lb_name            = "${local.prefix}-alb"
  
  # CIDR blocks per workspace
  vpc_cidr = {
    default    = "10.0.0.0/16"
    dev        = "10.1.0.0/16"
    staging    = "10.2.0.0/16"
    production = "10.0.0.0/16"
  }
}

resource "aws_vpc" "main" {
  cidr_block = local.vpc_cidr[terraform.workspace]

  tags = {
    Name        = local.vpc_name
    Environment = terraform.workspace
    ManagedBy   = "terraform"
  }
}

resource "aws_eks_cluster" "main" {
  name     = local.cluster_name
  role_arn = aws_iam_role.eks.arn

  vpc_config {
    subnet_ids = aws_subnet.private[*].id
  }
}
```

### Pattern 3: Conditional Resources per Workspace

```hcl
locals {
  # Workspace-based flags
  create_nat_gateway   = terraform.workspace == "production" || terraform.workspace == "staging"
  multi_az_db         = terraform.workspace == "production"
  enable_backup       = terraform.workspace != "dev" && terraform.workspace != "default"
  deletion_protection = terraform.workspace == "production"
}

resource "aws_nat_gateway" "main" {
  count         = local.create_nat_gateway ? 1 : 0
  allocation_id = aws_eip.nat[0].id
  subnet_id     = aws_subnet.public[0].id

  tags = { Name = "${terraform.workspace}-nat-gw" }
}

resource "aws_db_instance" "main" {
  identifier  = "${terraform.workspace}-db"
  engine      = "postgres"
  multi_az    = local.multi_az_db
  
  backup_retention_period = local.enable_backup ? 7 : 0
  deletion_protection     = local.deletion_protection

  skip_final_snapshot = !local.enable_backup

  tags = { Environment = terraform.workspace }
}
```

---

## Step 285: Workspaces and S3 Backend

### S3 Backend State Key Structure

```hcl
# กับ S3 backend, workspaces เก็บ state ที่:
# env:/<workspace_name>/<key>

terraform {
  backend "s3" {
    bucket = "my-terraform-state"
    key    = "myapp/terraform.tfstate"
    region = "us-east-1"
  }
}

# State paths:
# default workspace:  s3://my-terraform-state/myapp/terraform.tfstate
# dev workspace:      s3://my-terraform-state/env:/dev/myapp/terraform.tfstate
# staging workspace:  s3://my-terraform-state/env:/staging/myapp/terraform.tfstate
# production workspace: s3://my-terraform-state/env:/production/myapp/terraform.tfstate
```

### ดู State Files ใน S3

```bash
# List state files สำหรับทุก workspaces
aws s3 ls s3://my-terraform-state/ --recursive | grep terraform.tfstate

# Output:
# 2024-01-15 10:00:00       12345 myapp/terraform.tfstate
# 2024-01-15 10:00:00       12345 env:/dev/myapp/terraform.tfstate
# 2024-01-15 10:00:00       12345 env:/staging/myapp/terraform.tfstate
# 2024-01-15 10:00:00       12345 env:/production/myapp/terraform.tfstate
```

---

## Step 286: Complete Dev/Staging/Prod Setup

### Project Structure

```
my-terraform-project/
├── main.tf
├── variables.tf
├── outputs.tf
├── providers.tf
├── locals.tf
├── terraform.tfvars          # common defaults
├── workspace_vars/
│   ├── dev.tfvars            # dev overrides
│   ├── staging.tfvars        # staging overrides
│   └── production.tfvars     # production overrides
├── modules/
│   ├── vpc/
│   ├── eks/
│   └── rds/
└── scripts/
    ├── init.sh
    ├── plan.sh
    └── apply.sh
```

### providers.tf

```hcl
terraform {
  required_version = ">= 1.6"
  
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }

  backend "s3" {
    bucket         = "my-company-terraform-state"
    key            = "myapp/terraform.tfstate"
    region         = "us-east-1"
    encrypt        = true
    dynamodb_table = "terraform-state-locks"
  }
}

provider "aws" {
  region = var.aws_region
  
  default_tags {
    tags = {
      Environment = terraform.workspace
      ManagedBy   = "terraform"
      Project     = "my-app"
      Workspace   = terraform.workspace
    }
  }
}
```

### variables.tf

```hcl
variable "aws_region" {
  default = "us-east-1"
}

# Workspace-specific configurations
variable "workspace_vars" {
  type = map(object({
    # Networking
    vpc_cidr           = string
    public_subnet_cidrs  = list(string)
    private_subnet_cidrs = list(string)
    
    # Compute
    instance_type    = string
    min_size         = number
    max_size         = number
    desired_capacity = number
    
    # Database
    db_instance_class   = string
    db_allocated_storage = number
    db_multi_az          = bool
    db_backup_retention  = number
    
    # Features
    enable_nat_gateway  = bool
    enable_monitoring   = bool
    deletion_protection = bool
  }))
  
  default = {
    dev = {
      vpc_cidr             = "10.1.0.0/16"
      public_subnet_cidrs  = ["10.1.0.0/24", "10.1.1.0/24"]
      private_subnet_cidrs = ["10.1.10.0/24", "10.1.11.0/24"]
      
      instance_type    = "t3.micro"
      min_size         = 1
      max_size         = 2
      desired_capacity = 1
      
      db_instance_class    = "db.t3.micro"
      db_allocated_storage = 20
      db_multi_az          = false
      db_backup_retention  = 1
      
      enable_nat_gateway  = false
      enable_monitoring   = false
      deletion_protection = false
    }
    
    staging = {
      vpc_cidr             = "10.2.0.0/16"
      public_subnet_cidrs  = ["10.2.0.0/24", "10.2.1.0/24"]
      private_subnet_cidrs = ["10.2.10.0/24", "10.2.11.0/24"]
      
      instance_type    = "t3.small"
      min_size         = 2
      max_size         = 4
      desired_capacity = 2
      
      db_instance_class    = "db.t3.small"
      db_allocated_storage = 50
      db_multi_az          = false
      db_backup_retention  = 3
      
      enable_nat_gateway  = true
      enable_monitoring   = true
      deletion_protection = false
    }
    
    production = {
      vpc_cidr             = "10.0.0.0/16"
      public_subnet_cidrs  = ["10.0.0.0/24", "10.0.1.0/24", "10.0.2.0/24"]
      private_subnet_cidrs = ["10.0.10.0/24", "10.0.11.0/24", "10.0.12.0/24"]
      
      instance_type    = "t3.medium"
      min_size         = 3
      max_size         = 10
      desired_capacity = 3
      
      db_instance_class    = "db.r5.large"
      db_allocated_storage = 100
      db_multi_az          = true
      db_backup_retention  = 7
      
      enable_nat_gateway  = true
      enable_monitoring   = true
      deletion_protection = true
    }
  }
}
```

### locals.tf

```hcl
locals {
  workspace = terraform.workspace
  
  # Get workspace-specific config
  # Fallback to "dev" if workspace not found
  config = lookup(
    var.workspace_vars,
    terraform.workspace,
    var.workspace_vars["dev"]
  )
  
  # Convenience flags
  is_prod    = local.workspace == "production"
  is_staging = local.workspace == "staging"
  is_dev     = local.workspace == "dev" || local.workspace == "default"
  
  # Resource naming prefix
  prefix = local.workspace
  
  # Tags
  common_tags = {
    Environment = local.workspace
    ManagedBy   = "terraform"
    Project     = "my-app"
  }
}
```

### main.tf

```hcl
# VPC
module "vpc" {
  source = "./modules/vpc"
  
  name                 = "${local.prefix}-vpc"
  cidr_block           = local.config.vpc_cidr
  public_subnet_cidrs  = local.config.public_subnet_cidrs
  private_subnet_cidrs = local.config.private_subnet_cidrs
  enable_nat_gateway   = local.config.enable_nat_gateway
  
  tags = local.common_tags
}

# Database
resource "aws_db_instance" "main" {
  identifier     = "${local.prefix}-database"
  engine         = "postgres"
  engine_version = "14.7"
  instance_class = local.config.db_instance_class
  
  allocated_storage       = local.config.db_allocated_storage
  multi_az                = local.config.db_multi_az
  backup_retention_period = local.config.db_backup_retention
  deletion_protection     = local.config.deletion_protection
  
  skip_final_snapshot = !local.is_prod
  
  vpc_security_group_ids = [aws_security_group.db.id]
  db_subnet_group_name   = aws_db_subnet_group.main.name

  tags = merge(local.common_tags, { Name = "${local.prefix}-database" })

  lifecycle {
    prevent_destroy = false  # Override per workspace ถ้าต้องการ
  }
}
```

---

## Step 287: Workspace Deployment Workflow

### Init Script

```bash
#!/bin/bash
# scripts/init.sh

WORKSPACE=${1:-dev}

echo "=== Initializing workspace: $WORKSPACE ==="

# Init
terraform init

# Create workspace if not exists
if ! terraform workspace list | grep -q "^  $WORKSPACE$\|^\* $WORKSPACE$"; then
  echo "Creating workspace: $WORKSPACE"
  terraform workspace new $WORKSPACE
else
  echo "Selecting workspace: $WORKSPACE"
  terraform workspace select $WORKSPACE
fi

echo "=== Current workspace: $(terraform workspace show) ==="
```

### Plan Script

```bash
#!/bin/bash
# scripts/plan.sh

WORKSPACE=${1:-dev}
PLAN_FILE="plan-${WORKSPACE}-$(date +%Y%m%d-%H%M%S).tfplan"

echo "=== Planning workspace: $WORKSPACE ==="
terraform workspace select $WORKSPACE

terraform plan \
  -out="$PLAN_FILE" \
  -var="environment=$WORKSPACE"

echo ""
echo "Plan saved to: $PLAN_FILE"
echo "To apply: terraform apply $PLAN_FILE"
```

### Apply Script

```bash
#!/bin/bash
# scripts/apply.sh

WORKSPACE=${1:-dev}
PLAN_FILE=$2

if [ -z "$PLAN_FILE" ]; then
  echo "Usage: $0 <workspace> <plan_file>"
  exit 1
fi

# Safety check สำหรับ production
if [ "$WORKSPACE" = "production" ]; then
  echo "⚠️  WARNING: Applying to PRODUCTION!"
  echo "Type 'yes' to continue:"
  read -r CONFIRM
  if [ "$CONFIRM" != "yes" ]; then
    echo "Aborted."
    exit 1
  fi
fi

echo "=== Applying plan: $PLAN_FILE ==="
terraform workspace select $WORKSPACE
terraform apply "$PLAN_FILE"
```

---

## Step 288: Workspace Limitations & Alternatives

### Workspace Limitations

```
❌ Workspace ไม่เหมาะสำหรับ:
1. Environments ที่ต้องการ different configurations ที่ซับซ้อนมาก
2. Different providers หรือ accounts
3. Different regions ที่มี different backends
4. Team ขนาดใหญ่ที่ต้องการ isolation จริงๆ

⚠️ ปัญหาของ Workspaces:
1. Code logic ซับซ้อน (if workspace == "prod" everywhere)
2. ยากต่อการ test workspace-specific changes
3. Accidental apply ผิด workspace
4. ไม่ enforce different permissions per workspace
```

### เมื่อไรควรใช้ Workspaces

```
✅ เหมาะสำหรับ:
1. Infrastructure เหมือนกัน แต่ขนาดต่างกัน (dev/staging/prod)
2. Team เล็ก ต้องการ solution ง่ายๆ
3. Feature branches ที่ต้องการ isolated environment ชั่วคราว
4. Short-lived environments

❌ ไม่เหมาะสำหรับ:
1. Different AWS accounts
2. Very different configurations
3. Large teams ที่ต้องการ strong isolation
4. Compliance requirements ที่ต้องการ clear separation
```

### ทางเลือก: Separate Configs per Environment

```
# Directory-based approach
terraform/
├── environments/
│   ├── dev/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   └── terraform.tfvars
│   ├── staging/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   └── terraform.tfvars
│   └── prod/
│       ├── main.tf
│       ├── variables.tf
│       └── terraform.tfvars
└── modules/
    ├── vpc/
    ├── eks/
    └── rds/
```

```hcl
# environments/prod/main.tf
module "vpc" {
  source = "../../modules/vpc"
  # prod-specific settings
}
```

### ทางเลือก: Terragrunt

```hcl
# terragrunt.hcl
locals {
  environment = basename(get_terragrunt_dir())
  
  env_vars = {
    dev = {
      instance_type = "t3.micro"
    }
    prod = {
      instance_type = "t3.large"
    }
  }
}

terraform {
  source = "../../modules//vpc"
}

inputs = {
  instance_type = local.env_vars[local.environment]["instance_type"]
  environment   = local.environment
}

remote_state {
  backend = "s3"
  config = {
    bucket = "my-state-bucket"
    key    = "${local.environment}/vpc/terraform.tfstate"
    region = "us-east-1"
  }
}
```

---

## Step 289: Workspace in CI/CD

### GitHub Actions กับ Workspaces

```yaml
# .github/workflows/terraform.yml
name: Terraform

on:
  push:
    branches:
      - main
      - 'env/**'

jobs:
  terraform:
    runs-on: ubuntu-latest
    
    env:
      AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
      AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
    
    steps:
      - uses: actions/checkout@v4
      
      - uses: hashicorp/setup-terraform@v3
      
      - name: Determine Workspace
        id: workspace
        run: |
          if [[ "${{ github.ref }}" == "refs/heads/main" ]]; then
            echo "workspace=production" >> $GITHUB_OUTPUT
          elif [[ "${{ github.ref }}" =~ refs/heads/env/(.*) ]]; then
            echo "workspace=${BASH_REMATCH[1]}" >> $GITHUB_OUTPUT
          else
            echo "workspace=dev" >> $GITHUB_OUTPUT
          fi
      
      - name: Terraform Init
        run: terraform init
      
      - name: Terraform Select/Create Workspace
        run: |
          terraform workspace select ${{ steps.workspace.outputs.workspace }} || \
          terraform workspace new ${{ steps.workspace.outputs.workspace }}
      
      - name: Terraform Plan
        run: |
          terraform plan \
            -var="environment=${{ steps.workspace.outputs.workspace }}" \
            -out=tfplan
      
      - name: Terraform Apply
        if: github.event_name == 'push'
        run: terraform apply tfplan
```

---

## Step 290: Workspace Best Practices

### ✅ Best Practices

```bash
# 1. ตั้งชื่อ workspace ให้ตรงกับ environment names
# dev, staging, production (หรือ prod)
# ไม่ใช้: test, uat, qe (อาจสับสน)

# 2. Check workspace ก่อนทำงานเสมอ
alias tf-show-workspace='echo "Current workspace: $(terraform workspace show)"'

# 3. บังคับ confirm ก่อน production operations
terraform_apply_safe() {
  WORKSPACE=$(terraform workspace show)
  if [ "$WORKSPACE" = "production" ]; then
    echo "⚠️  PRODUCTION - Type 'yes' to continue:"
    read -r CONFIRM
    [ "$CONFIRM" = "yes" ] || return 1
  fi
  terraform apply "$@"
}

# 4. ใช้ workspace-based naming สม่ำเสมอ
# "${terraform.workspace}-resource-name"

# 5. Validate workspace ใน configuration
variable "allowed_workspaces" {
  default = ["dev", "staging", "production"]
}

resource "null_resource" "workspace_check" {
  lifecycle {
    precondition {
      condition     = contains(var.allowed_workspaces, terraform.workspace)
      error_message = "Invalid workspace '${terraform.workspace}'. Must be one of: ${join(", ", var.allowed_workspaces)}"
    }
  }
}
```

### Workspace ใน Shell Profile

```bash
# ~/.bashrc หรือ ~/.zshrc
# แสดง current workspace ใน prompt
tf_workspace() {
  if [ -d ".terraform" ]; then
    ws=$(terraform workspace show 2>/dev/null)
    if [ -n "$ws" ] && [ "$ws" != "default" ]; then
      echo " [tf:$ws]"
    fi
  fi
}

# เพิ่มใน PS1 (bash)
PS1='${debian_chroot:+($debian_chroot)}\u@\h:\w$(tf_workspace)\$ '

# Helper functions
alias twl='terraform workspace list'
alias tws='terraform workspace show'
alias twn='terraform workspace new'
alias twsel='terraform workspace select'
```

---

## Command Reference

```bash
# === Workspace Commands ===
terraform workspace list            # List all workspaces
terraform workspace show            # Show current workspace
terraform workspace new NAME        # Create and switch to workspace
terraform workspace select NAME     # Switch to workspace
terraform workspace delete NAME     # Delete workspace
terraform workspace delete -force   # Force delete (empty state)

# === Common Patterns ===
terraform workspace select dev && terraform plan
terraform workspace select production && terraform apply -auto-approve

# Check workspace in script
WS=$(terraform workspace show)
echo "Running in workspace: $WS"
```

---

## แบบฝึกหัด (Exercises)

### Exercise 1: Basic Workspace Setup

1. สร้าง workspace dev, staging, production
2. เขียน config ที่ใช้ `terraform.workspace` สำหรับ resource naming
3. Deploy ไปแต่ละ workspace
4. Verify state files แยกจากกัน

### Exercise 2: Workspace-Based Configuration

1. สร้าง `workspace_vars` map สำหรับ 3 environments
2. สร้าง infrastructure ที่ scale ตาม workspace
3. ทดสอบ deploy ไปแต่ละ workspace

### Exercise 3: CI/CD Integration

สร้าง GitHub Actions workflow ที่:
1. Deploy ไป `dev` workspace เมื่อ push to `develop` branch
2. Deploy ไป `production` workspace เมื่อ push to `main`
3. มีการ confirm สำหรับ production

---

## Checklist

- [ ] เข้าใจ workspace concept และ default workspace
- [ ] สามารถสร้าง, switch, ลบ workspaces ได้
- [ ] รู้วิธีใช้ `terraform.workspace` ใน configuration
- [ ] เข้าใจ workspace state storage ใน S3
- [ ] สามารถสร้าง workspace-based variable selection ได้
- [ ] รู้ข้อจำกัดของ workspaces
- [ ] เข้าใจ alternatives (directory-based, terragrunt)
- [ ] สามารถ integrate workspaces กับ CI/CD ได้
