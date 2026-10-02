# Part 98: Terraform Patterns & Anti-patterns (Steps 971-980)

## Best Practices และสิ่งที่ควรหลีกเลี่ยงใน Terraform

---

## Step 971: Introduction - ทำไม Patterns สำคัญ?

### ปัญหาที่พบในโปรเจคจริง

```
ปัญหาที่ทีม Infrastructure เผชิญ:
─────────────────────────────────────────────────────────────
❌ "Terraform state file ขนาด 50MB มี resource 2,000+ ชิ้น"
❌ "แก้ไข main.tf แล้ว plan ใช้เวลา 30 นาที"
❌ "ไม่มีใครรู้ว่าไฟล์ไหน manage resource อะไร"
❌ "Copy-paste module เหมือนกัน 5 ที่ แต่ maintain ยากมาก"
❌ "AWS credentials hardcoded ใน terraform.tfvars"
─────────────────────────────────────────────────────────────

Patterns ช่วยแก้ปัญหาเหล่านี้ได้!
```

---

## Step 972: PATTERN 1 - Module-per-Component

### ปัญหา

```hcl
# ❌ Anti-pattern: Everything in one directory
# main.tf - 2,000 lines of everything!
resource "aws_vpc" "main" { ... }
resource "aws_subnet" "public_1" { ... }
resource "aws_subnet" "public_2" { ... }
resource "aws_internet_gateway" "main" { ... }
resource "aws_db_instance" "primary" { ... }
resource "aws_elasticache_cluster" "redis" { ... }
resource "aws_eks_cluster" "main" { ... }
# ... 200 more resources
```

### Solution: Module-per-Component

```hcl
# ✅ Pattern: แต่ละ component เป็น module แยก
# environments/prod/main.tf
module "networking" {
  source  = "../../modules/networking"
  version = "2.0.0"
  
  environment = var.environment
  vpc_cidr    = var.vpc_cidr
  
  tags = local.common_tags
}

module "database" {
  source  = "../../modules/rds"
  version = "1.5.0"
  
  environment    = var.environment
  vpc_id         = module.networking.vpc_id
  subnet_ids     = module.networking.private_subnet_ids
  instance_class = var.db_instance_class
  
  tags = local.common_tags
}

module "eks_cluster" {
  source  = "../../modules/eks"
  version = "3.0.0"
  
  environment    = var.environment
  vpc_id         = module.networking.vpc_id
  subnet_ids     = module.networking.private_subnet_ids
  
  tags = local.common_tags
}
```

```
modules/
├── networking/
│   ├── main.tf        # VPC, Subnets, IGW, NAT
│   ├── variables.tf   # Input variables
│   ├── outputs.tf     # VPC ID, Subnet IDs
│   └── README.md
├── rds/
│   ├── main.tf        # DB instance, SG, subnet group
│   ├── variables.tf
│   └── outputs.tf
└── eks/
    ├── main.tf
    ├── variables.tf
    └── outputs.tf
```

---

## Step 973: PATTERN 2 - Environment-specific Variable Files

### ปัญหา

```hcl
# ❌ Anti-pattern: hardcoded values per environment
# environments/prod/main.tf
resource "aws_instance" "web" {
  instance_type = "m5.xlarge"    # hardcoded!
  ami           = "ami-prod1234"  # hardcoded!
}

# environments/dev/main.tf (copy-paste!)
resource "aws_instance" "web" {
  instance_type = "t3.micro"     # hardcoded!
  ami           = "ami-dev5678"  # hardcoded!
}
```

### Solution: Variable Files per Environment

```hcl
# modules/compute/main.tf - Generic module
resource "aws_instance" "web" {
  ami           = var.ami_id
  instance_type = var.instance_type
  
  tags = merge(var.tags, {
    Name = "${var.environment}-web-server-001"
  })
}

# modules/compute/variables.tf
variable "instance_type" {
  description = "EC2 instance type"
  type        = string
  
  validation {
    condition = contains([
      "t3.micro", "t3.small", "t3.medium",
      "m5.large", "m5.xlarge"
    ], var.instance_type)
    error_message = "Must use approved instance types."
  }
}
```

```hcl
# environments/dev/terraform.tfvars
environment    = "dev"
instance_type  = "t3.micro"
ami_id         = "ami-dev1234"
db_instance_class = "db.t3.micro"
min_capacity   = 1
max_capacity   = 3

# environments/prod/terraform.tfvars
environment    = "prod"
instance_type  = "m5.xlarge"
ami_id         = "ami-prod5678"
db_instance_class = "db.r5.2xlarge"
min_capacity   = 3
max_capacity   = 20
```

---

## Step 974: PATTERN 3 - Remote State Data Source

### Pattern: Cross-Stack Data Sharing

```hcl
# ===== networking stack outputs =====
# environments/prod/networking/outputs.tf
output "vpc_id" {
  value = aws_vpc.main.id
}

output "private_subnet_ids" {
  value = aws_subnet.private[*].id
}

# ===== compute stack ใช้ data source =====
# environments/prod/compute/main.tf

# ดึง outputs จาก networking stack
data "terraform_remote_state" "networking" {
  backend = "s3"
  
  config = {
    bucket = "my-terraform-state"
    key    = "prod/networking/terraform.tfstate"
    region = "ap-southeast-1"
  }
}

# ใช้ค่าจาก networking stack
resource "aws_instance" "app" {
  ami           = var.ami_id
  instance_type = var.instance_type
  
  # ✅ ดึงค่าจาก remote state - ไม่ต้อง hardcode!
  subnet_id = data.terraform_remote_state.networking.outputs.private_subnet_ids[0]
  
  vpc_security_group_ids = [
    data.terraform_remote_state.networking.outputs.app_security_group_id
  ]
}
```

### Alternative: SSM Parameter Store

```hcl
# ===== networking stack เก็บค่าใน SSM =====
resource "aws_ssm_parameter" "vpc_id" {
  name  = "/prod/networking/vpc_id"
  type  = "String"
  value = aws_vpc.main.id
}

# ===== compute stack ดึงค่าจาก SSM =====
data "aws_ssm_parameter" "vpc_id" {
  name = "/prod/networking/vpc_id"
}

resource "aws_instance" "app" {
  subnet_id = data.aws_ssm_parameter.vpc_id.value
}
```

---

## Step 975: PATTERN 4 - Factory Pattern (for_each on Map)

### Pattern: สร้าง Resources จาก Map

```hcl
# ===== Factory Pattern - สร้าง S3 buckets จาก config =====
# variables.tf
variable "buckets" {
  description = "S3 bucket configurations"
  type = map(object({
    versioning = bool
    lifecycle_days = number
    tags = map(string)
  }))
  
  default = {
    "app-assets" = {
      versioning     = true
      lifecycle_days = 90
      tags = { Purpose = "Static assets" }
    }
    "app-logs" = {
      versioning     = false
      lifecycle_days = 30
      tags = { Purpose = "Application logs" }
    }
    "app-backups" = {
      versioning     = true
      lifecycle_days = 365
      tags = { Purpose = "Database backups" }
    }
  }
}

# main.tf
resource "aws_s3_bucket" "buckets" {
  for_each = var.buckets
  
  bucket = "${var.environment}-${each.key}"
  
  tags = merge(var.common_tags, each.value.tags, {
    Name = "${var.environment}-${each.key}"
  })
}

resource "aws_s3_bucket_versioning" "buckets" {
  for_each = { for k, v in var.buckets : k => v if v.versioning }
  
  bucket = aws_s3_bucket.buckets[each.key].id
  
  versioning_configuration {
    status = "Enabled"
  }
}

resource "aws_s3_bucket_lifecycle_configuration" "buckets" {
  for_each = var.buckets
  
  bucket = aws_s3_bucket.buckets[each.key].id
  
  rule {
    id     = "lifecycle"
    status = "Enabled"
    
    expiration {
      days = each.value.lifecycle_days
    }
  }
}
```

### Factory Pattern สำหรับ IAM Users

```hcl
# iam-users.auto.tfvars
iam_users = {
  "john.doe" = {
    groups  = ["developers", "readonly"]
    console = true
    tags    = { Department = "Engineering" }
  }
  "jane.smith" = {
    groups  = ["developers", "terraform-ops"]
    console = true
    tags    = { Department = "Platform" }
  }
  "ci-bot" = {
    groups  = ["ci-cd"]
    console = false
    tags    = { Purpose = "CI/CD automation" }
  }
}
```

```hcl
# main.tf
resource "aws_iam_user" "users" {
  for_each = var.iam_users
  
  name = each.key
  tags = merge(var.common_tags, each.value.tags)
}

resource "aws_iam_user_group_membership" "users" {
  for_each = var.iam_users
  
  user   = aws_iam_user.users[each.key].name
  groups = each.value.groups
}
```

---

## Step 976: PATTERN 5 - Config-driven Infrastructure (YAML + for_each)

### YAML Config Files สำหรับ Infrastructure

```yaml
# config/microservices.yaml
services:
  api-gateway:
    cpu:    512
    memory: 1024
    port:   8080
    replicas: 2
    health_check: /health
    env:
      NODE_ENV: production
      LOG_LEVEL: info
  
  user-service:
    cpu:    256
    memory: 512
    port:   3001
    replicas: 3
    health_check: /health
    env:
      DB_HOST: "${db_endpoint}"
      
  payment-service:
    cpu:    1024
    memory: 2048
    port:   3002
    replicas: 2
    health_check: /health
    env:
      STRIPE_KEY: "${ssm:/prod/payment/stripe-key}"
```

```hcl
# main.tf
locals {
  # อ่าน YAML file
  services = yamldecode(file("${path.module}/config/microservices.yaml")).services
}

# สร้าง ECS Task Definition สำหรับแต่ละ service
resource "aws_ecs_task_definition" "services" {
  for_each = local.services
  
  family                   = "${var.environment}-${each.key}"
  requires_compatibilities = ["FARGATE"]
  network_mode             = "awsvpc"
  cpu                      = each.value.cpu
  memory                   = each.value.memory
  
  container_definitions = jsonencode([{
    name  = each.key
    image = "${aws_ecr_repository.services[each.key].repository_url}:latest"
    
    portMappings = [{
      containerPort = each.value.port
      protocol      = "tcp"
    }]
    
    environment = [
      for k, v in each.value.env : {
        name  = k
        value = v
      }
    ]
    
    healthCheck = {
      command  = ["CMD-SHELL", "curl -f http://localhost:${each.value.port}${each.value.health_check} || exit 1"]
      interval = 30
      timeout  = 5
      retries  = 3
    }
    
    logConfiguration = {
      logDriver = "awslogs"
      options = {
        "awslogs-group"         = "/ecs/${var.environment}/${each.key}"
        "awslogs-region"        = var.aws_region
        "awslogs-stream-prefix" = "ecs"
      }
    }
  }])
}

# ECS Service
resource "aws_ecs_service" "services" {
  for_each = local.services
  
  name            = "${var.environment}-${each.key}"
  cluster         = aws_ecs_cluster.main.id
  task_definition = aws_ecs_task_definition.services[each.key].arn
  desired_count   = each.value.replicas
  launch_type     = "FARGATE"
  
  network_configuration {
    subnets          = var.private_subnet_ids
    security_groups  = [aws_security_group.ecs_tasks.id]
    assign_public_ip = false
  }
}
```

---

## Step 977: PATTERN 6 - Secrets Management Pattern

### Pattern: ไม่เก็บ Secrets ใน Code

```hcl
# ❌ Anti-pattern: Secrets ใน code
resource "aws_db_instance" "main" {
  username = "admin"
  password = "SuperSecret123!"  # ❌ เห็นใน Git!
}

# ❌ Anti-pattern: Secrets ใน tfvars (แม้ใน .gitignore)
# terraform.tfvars
db_password = "SuperSecret123!"  # ❌ อาจ leak โดยไม่ตั้งใจ
```

```hcl
# ✅ Pattern 1: AWS Secrets Manager
data "aws_secretsmanager_secret_version" "db_password" {
  secret_id = "prod/database/master-password"
}

resource "aws_db_instance" "main" {
  username = "admin"
  password = data.aws_secretsmanager_secret_version.db_password.secret_string
}
```

```hcl
# ✅ Pattern 2: Random Password + Secrets Manager
resource "random_password" "db" {
  length           = 32
  special          = true
  override_special = "!#$%&*()-_=+[]{}<>:?"
}

resource "aws_secretsmanager_secret" "db_password" {
  name                    = "${var.environment}/database/master-password"
  recovery_window_in_days = 7
}

resource "aws_secretsmanager_secret_version" "db_password" {
  secret_id     = aws_secretsmanager_secret.db_password.id
  secret_string = random_password.db.result
}

resource "aws_db_instance" "main" {
  username = "admin"
  password = aws_secretsmanager_secret_version.db_password.secret_string
}
```

```hcl
# ✅ Pattern 3: HashiCorp Vault
provider "vault" {
  address = "https://vault.company.com:8200"
}

data "vault_generic_secret" "db" {
  path = "secret/prod/database"
}

resource "aws_db_instance" "main" {
  username = data.vault_generic_secret.db.data["username"]
  password = data.vault_generic_secret.db.data["password"]
}
```

---

## Step 978: PATTERN 7 - Zero-Downtime (create_before_destroy)

### Blue-Green Pattern

```hcl
# ===== Zero-Downtime AMI Update =====
resource "aws_launch_template" "app" {
  name_prefix   = "${var.environment}-app-"
  image_id      = var.ami_id  # เปลี่ยน AMI → สร้าง new launch template
  instance_type = var.instance_type
  
  # สร้าง new launch template ก่อน destroy เก่า
  lifecycle {
    create_before_destroy = true
  }
  
  user_data = base64encode(templatefile("${path.module}/user_data.sh", {
    environment = var.environment
    app_version = var.app_version
  }))
}

resource "aws_autoscaling_group" "app" {
  name = "${var.environment}-app-${aws_launch_template.app.latest_version}"
  
  desired_capacity = var.desired_capacity
  min_size         = var.min_size
  max_size         = var.max_size
  
  launch_template {
    id      = aws_launch_template.app.id
    version = "$Latest"
  }
  
  # Instance refresh สำหรับ zero-downtime update
  instance_refresh {
    strategy = "Rolling"
    preferences {
      min_healthy_percentage = 90
      instance_warmup        = 300
    }
  }
  
  lifecycle {
    create_before_destroy = true
    
    # ป้องกัน Terraform ลบ ASG (ทำเอง)
    ignore_changes = [desired_capacity]
  }
}
```

---

## Step 979: ANTI-PATTERNS ที่ควรหลีกเลี่ยง

### Anti-pattern 1: God Module

```hcl
# ❌ Anti-pattern: ทุกอย่างอยู่ใน module เดียว
# modules/everything/main.tf - 5,000 lines!
resource "aws_vpc" "main" { ... }
resource "aws_eks_cluster" "main" { ... }
resource "aws_rds_cluster" "main" { ... }
resource "aws_elasticache_cluster" "redis" { ... }
resource "aws_cloudfront_distribution" "main" { ... }
# ... ต่อไปเรื่อยๆ

# ✅ ถูก: แยก module ตาม component
modules/
├── networking/
├── eks/
├── rds/
├── elasticache/
└── cdn/
```

### Anti-pattern 2: Copy-Paste Modules

```hcl
# ❌ Anti-pattern: Copy-paste module
# เมื่อแก้ bug ต้องแก้ทุกที่!
modules/
├── rds-dev/
│   └── main.tf  # Copy ของ rds-prod
├── rds-staging/
│   └── main.tf  # Copy ของ rds-prod (ล้าหลัง)
└── rds-prod/
    └── main.tf  # ต้นฉบับ

# ✅ ถูก: Module เดียว, environment ต่างกันด้วย variables
modules/
└── rds/
    ├── main.tf
    └── variables.tf  # instance_class, storage, etc.

environments/
├── dev/
│   └── main.tf   # module.rds { instance_class = "db.t3.micro" }
└── prod/
    └── main.tf   # module.rds { instance_class = "db.r5.2xlarge" }
```

### Anti-pattern 3: Monolithic State

```
# ❌ Anti-pattern: State file เดียวสำหรับทั้งหมด
terraform.tfstate (100MB, 3,000 resources)

ปัญหา:
- Plan/Apply ช้ามาก
- Risk สูงมาก (mistake อาจทำลาย production)
- Team ต้องรอกัน (state locking)
- ยาก debug

# ✅ ถูก: State แยกตาม concern
states/
├── networking/terraform.tfstate      # VPC, Subnets
├── shared-services/terraform.tfstate  # DNS, ACM, etc.
├── eks/terraform.tfstate              # Kubernetes cluster
├── rds/terraform.tfstate              # Databases
└── applications/
    ├── api/terraform.tfstate
    └── web/terraform.tfstate
```

### Anti-pattern 4: ไม่มี Module Versioning

```hcl
# ❌ Anti-pattern: ใช้ latest โดยไม่ pin version
module "vpc" {
  source = "terraform-aws-modules/vpc/aws"
  # ไม่มี version! Breaking changes จะทำให้ break!
}

# ✅ ถูก: Pin module version
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "5.1.2"  # ✅ Pin version ชัดเจน
}

# หรือ range version
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "~> 5.1"  # ✅ Allow patch updates only
}
```

### Anti-pattern 5: Credentials ใน Code

```hcl
# ❌ Anti-pattern: Credentials ใน code
provider "aws" {
  region     = "us-east-1"
  access_key = "AKIAIOSFODNN7EXAMPLE"    # ❌ NEVER!
  secret_key = "wJalrXUtnFEMI/K7MDENG"  # ❌ NEVER!
}

# ❌ Anti-pattern: ใน tfvars
aws_access_key = "AKIAIOSFODNN7EXAMPLE"   # ❌

# ✅ ถูก: ใช้ environment variables หรือ instance roles
provider "aws" {
  region = var.aws_region
  # Credentials จาก environment variables หรือ instance role
  # AWS_ACCESS_KEY_ID, AWS_SECRET_ACCESS_KEY
  # หรือ IAM role (EC2/Lambda/ECS/EKS)
}

# ✅ ถูก: ใช้ assume_role
provider "aws" {
  region = var.aws_region
  
  assume_role {
    role_arn     = "arn:aws:iam::123456789012:role/terraform-execution"
    session_name = "TerraformSession"
  }
}
```

### Anti-pattern 6: ใช้ -auto-approve ใน CI โดยไม่มี Plan Review

```bash
# ❌ Anti-pattern: Apply โดยไม่มี plan review
terraform apply -auto-approve  # อันตรายมาก!

# ✅ ถูก: Plan → Review → Approve → Apply
# Step 1: Plan
terraform plan -out=tfplan.binary

# Step 2: Convert plan to readable format
terraform show -json tfplan.binary > tfplan.json

# Step 3: Human reviews plan (บน PR)

# Step 4: Apply only after approval
terraform apply tfplan.binary
```

### Anti-pattern 7: Manual State Manipulation

```bash
# ❌ Anti-pattern: แก้ไข state โดยตรง
vim terraform.tfstate  # อย่าทำ!

# ❌ Anti-pattern: ลบ state โดยไม่ระวัง
terraform state rm aws_instance.web  # ถ้าไม่แน่ใจ อย่าทำ!

# ✅ ถูก: ใช้ terraform state commands อย่างระวัง
# ดู state ก่อน
terraform state list
terraform state show aws_instance.web

# Backup ก่อนทำ
terraform state pull > backup-$(date +%Y%m%d).tfstate

# ทำการเปลี่ยนแปลง
terraform state mv aws_instance.web aws_instance.api_server

# ตรวจสอบผลลัพธ์
terraform plan  # ควร show no changes
```

---

## Step 980: PATTERN - Complete Production Example

### สรุป Pattern ที่ดีทั้งหมด

```hcl
# environments/prod/main.tf - Production-ready example

terraform {
  required_version = ">= 1.7.0"
  
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
  
  backend "s3" {
    bucket         = "my-terraform-state-prod"
    key            = "prod/main/terraform.tfstate"
    region         = "ap-southeast-1"
    encrypt        = true
    kms_key_id     = "arn:aws:kms:ap-southeast-1:123456789012:key/mrk-xxx"
    dynamodb_table = "terraform-state-lock"
  }
}

provider "aws" {
  region = var.aws_region
  
  assume_role {
    role_arn     = "arn:aws:iam::123456789012:role/terraform-prod"
    session_name = "TerraformProd-${formatdate("YYYYMMDD-HHmmss", timestamp())}"
  }
  
  default_tags {
    tags = {
      Environment = "prod"
      ManagedBy   = "terraform"
      Repository  = "github.com/myorg/infrastructure"
    }
  }
}

# ===== Local Values =====
locals {
  # อ่าน YAML config
  services = yamldecode(file("${path.module}/config/services.yaml"))
  
  # Common tags
  common_tags = {
    Owner       = "platform-team"
    Environment = var.environment
    Project     = var.project
    CostCenter  = var.cost_center
  }
}

# ===== Networking Module =====
module "networking" {
  source  = "../../modules/networking"
  version = "2.1.0"
  
  environment        = var.environment
  vpc_cidr           = var.vpc_cidr
  availability_zones = data.aws_availability_zones.available.names
  
  tags = local.common_tags
}

# ===== EKS Module =====
module "eks" {
  source  = "../../modules/eks"
  version = "3.2.0"
  
  environment    = var.environment
  cluster_name   = "${var.environment}-${var.project}"
  vpc_id         = module.networking.vpc_id
  subnet_ids     = module.networking.private_subnet_ids
  cluster_version = "1.28"
  
  node_groups = {
    general = {
      instance_types = ["m5.large", "m5.xlarge"]
      min_size       = 3
      max_size       = 10
      desired_size   = 5
    }
  }
  
  tags = local.common_tags
}

# ===== RDS Module =====
module "database" {
  source  = "../../modules/rds"
  version = "1.3.0"
  
  environment    = var.environment
  vpc_id         = module.networking.vpc_id
  subnet_ids     = module.networking.database_subnet_ids
  instance_class = var.db_instance_class
  
  # ดึง password จาก Secrets Manager
  master_password = data.aws_secretsmanager_secret_version.db_master.secret_string
  
  # Backup และ maintenance
  backup_retention_period = 30
  deletion_protection     = true
  multi_az                = true
  
  tags = local.common_tags
}
```

---

## Pattern Summary Table

```
┌────────────────────────────────┬────────────────────────────────────────────┐
│ Pattern                        │ ใช้เมื่อ                                   │
├────────────────────────────────┼────────────────────────────────────────────┤
│ Module-per-Component           │ เสมอ (ห้าม God Module)                     │
│ Env-specific Variable Files    │ จัดการหลาย environments                    │
│ Remote State Data Source       │ Cross-stack dependencies                   │
│ Factory Pattern (for_each Map) │ สร้าง resources จาก config                │
│ Config-driven (YAML+for_each)  │ microservices, มี services หลายตัว        │
│ Secrets Management             │ เสมอ (ห้าม hardcode secrets!)              │
│ create_before_destroy          │ Zero-downtime updates                      │
│ Wrapper Module                 │ เพิ่ม company standards บน top ของ module  │
│ Cross-account Pattern          │ Multi-account AWS setup                    │
│ Blue-Green                     │ Zero-downtime deployments                  │
└────────────────────────────────┴────────────────────────────────────────────┘

┌────────────────────────────────┬────────────────────────────────────────────┐
│ Anti-pattern                   │ ปัญหา                                      │
├────────────────────────────────┼────────────────────────────────────────────┤
│ God Module                     │ ยาก maintain, slow plan/apply              │
│ Hardcoded Values               │ ไม่ reusable, copy-paste                  │
│ Copy-paste Modules             │ Bug fix ต้องแก้หลายที่                     │
│ Monolithic State               │ Slow, risky, team blocking                │
│ No Module Versioning           │ Breaking changes ไม่คาดคิด                 │
│ Credentials in Code            │ Security risk!                            │
│ -auto-approve without review   │ อันตราย! Destroy production โดยไม่ตั้งใจ  │
│ Manual State Manipulation      │ State corruption                          │
│ Ignoring State Locking         │ Concurrent runs = corrupted state         │
│ Excessive Module Nesting       │ Dependencies ซับซ้อน, debug ยาก           │
└────────────────────────────────┴────────────────────────────────────────────┘
```

---

## แบบฝึกหัด

1. Review Terraform code ที่มีอยู่แล้วหา anti-patterns
2. แปลง God Module เป็น Component-based modules
3. Implement Factory Pattern สำหรับ S3 buckets
4. ย้าย hardcoded secrets ไปยัง AWS Secrets Manager
5. ตั้งค่า state partitioning สำหรับ project ขนาดใหญ่
6. เพิ่ม `create_before_destroy` ให้กับ critical resources
7. สร้าง YAML-driven infrastructure configuration

---

*จบ Part 98: Terraform Patterns & Anti-patterns*
