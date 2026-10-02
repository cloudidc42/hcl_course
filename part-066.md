# Part 066: Module Composition Patterns
## รูปแบบการประกอบ Modules
### Steps 651-660

---

## บทนำ (Introduction)

Module composition คือการนำ modules หลายๆ ตัวมาประกอบกันเพื่อสร้าง infrastructure ที่ซับซ้อน แทนที่จะสร้าง module ขนาดใหญ่ที่ทำทุกอย่าง เราจะสร้าง modules เล็กๆ แล้วนำมา compose

---

## Step 651: Single-Purpose Modules

### Module ที่ทำสิ่งเดียว

```hcl
# ==========================================
# SINGLE-PURPOSE MODULES
# ==========================================

# Module 1: VPC - ทำแค่ networking
# modules/networking/vpc/
# - aws_vpc
# - aws_subnet (public/private/database)
# - aws_internet_gateway
# - aws_nat_gateway
# - aws_route_table

# Module 2: ECS Cluster - ทำแค่ compute platform
# modules/compute/ecs-cluster/
# - aws_ecs_cluster
# - aws_ecs_cluster_capacity_providers
# - aws_cloudwatch_log_group

# Module 3: RDS - ทำแค่ database
# modules/database/rds/
# - aws_db_instance
# - aws_db_subnet_group
# - aws_security_group (for RDS)

# Module 4: ECS Service - ทำแค่ application service
# modules/compute/ecs-service/
# - aws_ecs_task_definition
# - aws_ecs_service
# - aws_lb
# - aws_lb_target_group
# - aws_lb_listener
```

---

## Step 652: Wrapper Modules

### Wrapper module ที่ wrap module อื่น

```hcl
# ==========================================
# WRAPPER MODULE PATTERN
# ==========================================
# modules/app-database/main.tf
# Wraps RDS module พร้อม company standards

module "rds" {
  source  = "terraform-aws-modules/rds/aws"
  version = "~> 6.0"

  identifier = var.identifier

  engine               = var.engine
  engine_version       = var.engine_version
  instance_class       = var.instance_class
  allocated_storage    = var.allocated_storage

  # Company standards (hard-coded defaults)
  storage_encrypted         = true     # Always encrypt
  deletion_protection       = true     # Always protect
  backup_retention_period   = var.environment == "prod" ? 30 : 7
  multi_az                  = var.environment == "prod" ? true : false
  performance_insights_enabled = true  # Always enable
  monitoring_interval       = 60       # Always monitor

  # Force required tags
  tags = merge(var.tags, {
    ManagedBy   = "Terraform"
    CostCenter  = var.cost_center
    Environment = var.environment
  })
}

# modules/app-database/variables.tf
variable "identifier" { type = string }
variable "environment" { type = string }
variable "engine" { type = string; default = "postgres" }
variable "engine_version" { type = string; default = "15" }
variable "instance_class" { type = string; default = "db.t3.micro" }
variable "allocated_storage" { type = number; default = 20 }
variable "cost_center" { type = string }
variable "tags" { type = map(string); default = {} }

# modules/app-database/outputs.tf
output "endpoint" { value = module.rds.db_instance_endpoint }
output "port" { value = module.rds.db_instance_port }
output "username" { value = module.rds.db_instance_username }
output "instance_id" { value = module.rds.db_instance_id }
```

---

## Step 653: App Module Using VPC, ECS, RDS Modules

### Pattern: Application Module

```hcl
# ==========================================
# APP MODULE - ประกอบ VPC + ECS + RDS
# modules/app/main.tf
# ==========================================

# Networking
module "vpc" {
  source = "../networking/vpc"

  project            = var.project
  environment        = var.environment
  cidr_block         = var.vpc_cidr
  enable_nat_gateway = var.environment == "prod" ? true : false
  single_nat_gateway = var.environment != "prod"
}

# Compute Platform
module "ecs_cluster" {
  source = "../compute/ecs-cluster"

  name        = "${var.project}-${var.environment}"
  project     = var.project
  environment = var.environment
}

# Database
module "rds" {
  source = "../database/rds"

  project        = var.project
  environment    = var.environment
  vpc_id         = module.vpc.vpc_id
  subnet_ids     = module.vpc.database_subnet_ids
  instance_class = var.db_instance_class
  db_name        = var.db_name
}

# Application Service
module "app_service" {
  source = "../compute/ecs-service"

  project      = var.project
  environment  = var.environment
  cluster_arn  = module.ecs_cluster.cluster_arn
  vpc_id       = module.vpc.vpc_id
  subnet_ids   = module.vpc.private_subnet_ids

  container_image = var.app_image
  container_port  = var.app_port
  desired_count   = var.app_desired_count

  # Pass database connection info to app
  environment_variables = merge(var.app_env, {
    DB_HOST = module.rds.db_host
    DB_PORT = tostring(module.rds.db_port)
    DB_NAME = module.rds.db_name
    DB_USER = module.rds.db_username
  })

  secrets = {
    DB_PASSWORD = module.rds.db_secret_arn
  }

  tags = var.tags
}

# ==========================================
# modules/app/variables.tf
# ==========================================

variable "project" { type = string }
variable "environment" { type = string }
variable "region" { type = string; default = "ap-southeast-1" }

variable "vpc_cidr" {
  type    = string
  default = "10.0.0.0/16"
}

variable "db_instance_class" {
  type    = string
  default = "db.t3.micro"
}

variable "db_name" {
  type    = string
  default = "appdb"
}

variable "app_image" { type = string }
variable "app_port" { type = number; default = 8080 }
variable "app_desired_count" { type = number; default = 1 }
variable "app_env" { type = map(string); default = {} }
variable "tags" { type = map(string); default = {} }

# ==========================================
# modules/app/outputs.tf
# ==========================================

output "vpc_id" {
  value = module.vpc.vpc_id
}

output "app_url" {
  value = module.app_service.load_balancer_dns
}

output "database_endpoint" {
  value = module.rds.db_endpoint
}

output "ecs_cluster_arn" {
  value = module.ecs_cluster.cluster_arn
}
```

---

## Step 654: Environment Module Composing App Modules

### Pattern: Environment Module

```hcl
# ==========================================
# ENVIRONMENT MODULE - ประกอบหลาย app modules
# environments/prod/main.tf
# ==========================================

terraform {
  required_version = ">= 1.5.0"
  required_providers {
    aws = { source = "hashicorp/aws"; version = "~> 5.0" }
  }
  backend "s3" {
    bucket = "mycompany-terraform-state"
    key    = "prod/main.tfstate"
    region = "ap-southeast-1"
  }
}

provider "aws" {
  region = var.region
}

locals {
  environment = "prod"
  project     = "mycompany"
  region      = var.region

  common_tags = {
    Environment = local.environment
    Project     = local.project
    ManagedBy   = "Terraform"
    Region      = local.region
  }
}

# ============================================================
# FRONTEND APP
# ============================================================
module "frontend" {
  source = "../../modules/app"

  project     = local.project
  environment = "${local.environment}-frontend"
  region      = local.region

  vpc_cidr  = "10.0.0.0/16"
  app_image = "myregistry/frontend:${var.frontend_version}"
  app_port  = 3000

  app_desired_count = 3
  db_instance_class = "db.t3.medium"

  tags = merge(local.common_tags, { App = "frontend" })
}

# ============================================================
# BACKEND API
# ============================================================
module "backend" {
  source = "../../modules/app"

  project     = local.project
  environment = "${local.environment}-backend"
  region      = local.region

  vpc_cidr  = "10.1.0.0/16"
  app_image = "myregistry/backend:${var.backend_version}"
  app_port  = 8080

  app_desired_count = 5
  db_instance_class = "db.r6g.large"

  app_env = {
    FRONTEND_URL = "https://app.mycompany.com"
    LOG_LEVEL    = "INFO"
  }

  tags = merge(local.common_tags, { App = "backend" })
}

# ============================================================
# ADMIN SERVICE
# ============================================================
module "admin" {
  source = "../../modules/app"

  project     = local.project
  environment = "${local.environment}-admin"
  region      = local.region

  vpc_cidr  = "10.2.0.0/16"
  app_image = "myregistry/admin:${var.admin_version}"
  app_port  = 8081

  app_desired_count = 2
  db_instance_class = "db.t3.small"

  tags = merge(local.common_tags, { App = "admin" })
}

# ============================================================
# SHARED SERVICES (Route53, CloudFront, etc.)
# ============================================================
module "shared" {
  source = "../../modules/shared"

  project      = local.project
  environment  = local.environment
  domain_name  = "mycompany.com"

  services = {
    frontend = {
      subdomain       = "app"
      alb_dns_name    = module.frontend.app_url
      alb_zone_id     = module.frontend.alb_zone_id
    }
    backend = {
      subdomain       = "api"
      alb_dns_name    = module.backend.app_url
      alb_zone_id     = module.backend.alb_zone_id
    }
  }

  tags = local.common_tags
}

# Outputs
output "frontend_url" { value = "https://app.mycompany.com" }
output "backend_api_url" { value = "https://api.mycompany.com" }
output "admin_url" { value = module.admin.app_url }
```

---

## Step 655: Region Module Composing Environment Modules

### Pattern: Multi-Region

```hcl
# ==========================================
# REGION MODULE - ประกอบ environment modules
# deployments/global/main.tf
# ==========================================

terraform {
  required_version = ">= 1.5.0"
  required_providers {
    aws = { source = "hashicorp/aws"; version = "~> 5.0" }
  }
}

# Primary Region - Singapore
provider "aws" {
  alias  = "primary"
  region = "ap-southeast-1"
}

# DR Region - US East
provider "aws" {
  alias  = "dr"
  region = "us-east-1"
}

module "primary_region" {
  source = "../../environments/prod"

  providers = {
    aws = aws.primary
  }

  region             = "ap-southeast-1"
  frontend_version   = var.frontend_version
  backend_version    = var.backend_version
  admin_version      = var.admin_version
}

module "dr_region" {
  source = "../../environments/prod"

  providers = {
    aws = aws.dr
  }

  region             = "us-east-1"
  frontend_version   = var.frontend_version
  backend_version    = var.backend_version
  admin_version      = var.admin_version

  # DR: scale down
  frontend_desired_count = 1
  backend_desired_count  = 2
}

# Route 53 Global Load Balancing
module "global_routing" {
  source = "../../modules/global-routing"

  domain_name = "mycompany.com"

  endpoints = {
    primary = {
      region      = "ap-southeast-1"
      alb_dns     = module.primary_region.frontend_url
      weight      = 90
      health_check = true
    }
    dr = {
      region      = "us-east-1"
      alb_dns     = module.dr_region.frontend_url
      weight      = 10
      health_check = true
    }
  }
}
```

---

## Step 656: Passing Outputs Between Modules

### วิธีส่ง outputs ระหว่าง modules

```hcl
# ==========================================
# PASSING OUTPUTS BETWEEN MODULES
# ==========================================

# Step 1: Module A creates resources
module "vpc" {
  source = "./modules/vpc"

  project    = var.project
  cidr_block = "10.0.0.0/16"
}

# Step 2: Module B uses outputs from Module A
module "security_groups" {
  source = "./modules/security-groups"

  # รับ vpc_id จาก module.vpc.vpc_id
  vpc_id      = module.vpc.vpc_id
  environment = var.environment
}

# Step 3: Module C uses outputs from A and B
module "compute" {
  source = "./modules/compute"

  # จาก VPC module
  vpc_id     = module.vpc.vpc_id
  subnet_ids = module.vpc.private_subnet_ids

  # จาก Security Groups module
  security_group_ids = module.security_groups.app_security_group_ids

  instance_type = var.instance_type
}

# Step 4: Module D uses outputs from A, B, and C
module "alb" {
  source = "./modules/alb"

  # จาก VPC
  vpc_id             = module.vpc.vpc_id
  public_subnet_ids  = module.vpc.public_subnet_ids

  # จาก Security Groups
  alb_security_group_id = module.security_groups.alb_security_group_id

  # จาก Compute
  instance_ids = module.compute.instance_ids
}

# ==========================================
# PRACTICAL: 3-TIER APP COMPOSITION
# ==========================================

locals {
  project     = "myapp"
  environment = "prod"
}

# Tier 1: Networking
module "networking" {
  source = "./modules/networking"

  project     = local.project
  environment = local.environment
  cidr_block  = "10.0.0.0/16"
}

# Tier 2: Database  
module "database" {
  source = "./modules/database"

  project     = local.project
  environment = local.environment

  # รับจาก networking module
  vpc_id     = module.networking.vpc_id
  subnet_ids = module.networking.database_subnet_ids
}

# Tier 3: Application
module "application" {
  source = "./modules/application"

  project     = local.project
  environment = local.environment

  # รับจาก networking module
  vpc_id            = module.networking.vpc_id
  private_subnet_ids = module.networking.private_subnet_ids
  public_subnet_ids  = module.networking.public_subnet_ids

  # รับจาก database module
  db_host     = module.database.db_host
  db_port     = module.database.db_port
  db_name     = module.database.db_name
  db_username = module.database.db_username
  db_secret_arn = module.database.db_secret_arn
}
```

---

## Step 657: Module Communication Anti-patterns

### สิ่งที่ไม่ควรทำ

```hcl
# ==========================================
# ANTI-PATTERNS TO AVOID
# ==========================================

# ANTI-PATTERN 1: Module ที่ hardcode dependencies
# BAD - module สร้าง VPC ตัวเองภายใน
module "compute_bad" {
  source = "./modules/compute-with-own-vpc"  # BAD! ทำทุกอย่างเอง
  # Module สร้าง VPC, subnet, security group ทั้งหมดเอง
  # ทำให้ reuse ยากและ test ยาก
}

# GOOD - แยก concerns ออกจากกัน
module "vpc_good" {
  source = "./modules/vpc"
  cidr_block = "10.0.0.0/16"
}

module "compute_good" {
  source = "./modules/compute"
  vpc_id     = module.vpc_good.vpc_id
  subnet_ids = module.vpc_good.private_subnet_ids
}

# ANTI-PATTERN 2: Circular dependencies
# BAD - A depends on B, B depends on A
# module "a" { output_from_b = module.b.output }  # ERROR!
# module "b" { output_from_a = module.a.output }  # ERROR!

# GOOD - ใช้ data source หรือ remote state แทน
data "aws_vpc" "main" {
  tags = { Name = "prod-vpc" }
}

module "compute" {
  vpc_id = data.aws_vpc.main.id  # ดึงจาก data source
}

# ANTI-PATTERN 3: Module รับ object ทั้งหมดแทนที่จะรับ specific attributes
# BAD
variable "vpc_module_output" {
  type = any  # รับทั้ง module output
}
resource "aws_instance" "bad" {
  subnet_id = var.vpc_module_output.private_subnet_ids[0]
}

# GOOD
variable "subnet_id" {
  type = string  # รับแค่ที่ต้องการ
}
resource "aws_instance" "good" {
  subnet_id = var.subnet_id
}

# ANTI-PATTERN 4: Deeply nested modules (3+ levels)
# BAD: root -> app -> service -> compute -> instance
# ยาก debug, ยาก maintain

# GOOD: Flat composition ที่ root
module "vpc" { source = "./modules/vpc" }
module "rds" { source = "./modules/rds"; vpc_id = module.vpc.vpc_id }
module "ecs" { source = "./modules/ecs"; vpc_id = module.vpc.vpc_id }
```

---

## Step 658: Cross-Module Data via Remote State

```hcl
# ==========================================
# CROSS-MODULE DATA VIA REMOTE STATE
# ==========================================

# File: data.tf ใน application layer
# ดึงข้อมูลจาก networking layer ที่ deploy แยก

data "terraform_remote_state" "networking" {
  backend = "s3"

  config = {
    bucket = "mycompany-terraform-state"
    key    = "${var.environment}/networking/terraform.tfstate"
    region = "ap-southeast-1"
  }
}

data "terraform_remote_state" "platform" {
  backend = "s3"

  config = {
    bucket = "mycompany-terraform-state"
    key    = "${var.environment}/platform/terraform.tfstate"
    region = "ap-southeast-1"
  }
}

# ใช้ข้อมูลจาก remote states
locals {
  vpc_id          = data.terraform_remote_state.networking.outputs.vpc_id
  private_subnets = data.terraform_remote_state.networking.outputs.private_subnet_ids
  ecs_cluster_arn = data.terraform_remote_state.platform.outputs.ecs_cluster_arn
}

# Deploy application using info from other stacks
resource "aws_ecs_service" "app" {
  name            = "myapp"
  cluster         = local.ecs_cluster_arn
  task_definition = aws_ecs_task_definition.app.arn
  desired_count   = var.desired_count

  network_configuration {
    subnets         = local.private_subnets
    security_groups = [aws_security_group.app.id]
  }
}

resource "aws_security_group" "app" {
  name   = "myapp-sg"
  vpc_id = local.vpc_id
}
```

---

## Step 659: Module Graph Design

### การออกแบบ dependency graph

```
# Module Dependency Graph ที่ดี (Acyclic - ไม่มี circular)
#
#     foundation
#         |
#      networking
#       /     \
#  platform   security
#       \     /
#      application
#
# Flow: foundation -> networking -> platform/security -> application

# ==========================================
# LAYER 1: Foundation (accounts, shared resources)
# ==========================================
# stacks/foundation/main.tf
resource "aws_kms_key" "main" { ... }
resource "aws_s3_bucket" "terraform_state" { ... }
resource "aws_iam_role" "terraform_executor" { ... }

output "kms_key_arn" { value = aws_kms_key.main.arn }
output "terraform_state_bucket" { value = aws_s3_bucket.terraform_state.bucket }

# ==========================================
# LAYER 2: Networking
# ==========================================
# stacks/networking/main.tf
data "terraform_remote_state" "foundation" { ... }

module "vpc" {
  source     = "./modules/vpc"
  kms_key_id = data.terraform_remote_state.foundation.outputs.kms_key_arn
}

output "vpc_id" { value = module.vpc.vpc_id }
output "private_subnets" { value = module.vpc.private_subnet_ids }

# ==========================================
# LAYER 3: Platform (ECS, RDS, ElastiCache)
# ==========================================
# stacks/platform/main.tf
data "terraform_remote_state" "networking" { ... }

module "ecs_cluster" {
  source  = "./modules/ecs-cluster"
  vpc_id  = data.terraform_remote_state.networking.outputs.vpc_id
  subnets = data.terraform_remote_state.networking.outputs.private_subnets
}

module "rds" {
  source  = "./modules/rds"
  vpc_id  = data.terraform_remote_state.networking.outputs.vpc_id
  subnets = data.terraform_remote_state.networking.outputs.private_subnets
}

output "ecs_cluster_arn" { value = module.ecs_cluster.cluster_arn }
output "rds_endpoint" { value = module.rds.endpoint }

# ==========================================
# LAYER 4: Application
# ==========================================
# stacks/application/main.tf
data "terraform_remote_state" "networking" { ... }
data "terraform_remote_state" "platform" { ... }

module "app" {
  source      = "./modules/app"
  vpc_id      = data.terraform_remote_state.networking.outputs.vpc_id
  subnets     = data.terraform_remote_state.networking.outputs.private_subnets
  cluster_arn = data.terraform_remote_state.platform.outputs.ecs_cluster_arn
  db_endpoint = data.terraform_remote_state.platform.outputs.rds_endpoint
}
```

---

## Step 660: Complete 3-Tier App Example

### ตัวอย่างสมบูรณ์ - 3-Tier Application

```hcl
# ==========================================
# COMPLETE 3-TIER APP USING COMPOSED MODULES
# main.tf
# ==========================================

terraform {
  required_version = ">= 1.5.0"
  required_providers {
    aws = { source = "hashicorp/aws"; version = "~> 5.0" }
  }
}

locals {
  project     = "myapp"
  environment = var.environment
  region      = "ap-southeast-1"

  common_tags = {
    Project     = local.project
    Environment = local.environment
    ManagedBy   = "Terraform"
  }
}

# ===================
# NETWORKING MODULE
# ===================
module "networking" {
  source = "./modules/networking"

  project     = local.project
  environment = local.environment

  vpc_cidr = {
    dev     = "10.0.0.0/16"
    staging = "10.1.0.0/16"
    prod    = "10.2.0.0/16"
  }[local.environment]

  azs = ["${local.region}a", "${local.region}b"]

  enable_nat_gateway = local.environment == "prod"
  single_nat_gateway = local.environment != "prod"

  tags = local.common_tags
}

# ===================
# COMPUTE MODULE (ECS Cluster)
# ===================
module "compute" {
  source = "./modules/compute"

  project     = local.project
  environment = local.environment

  # From networking
  vpc_id     = module.networking.vpc_id
  subnet_ids = module.networking.private_subnet_ids

  tags = local.common_tags
}

# ===================
# DATABASE MODULE
# ===================
module "database" {
  source = "./modules/database"

  project     = local.project
  environment = local.environment
  db_name     = "appdb"

  # From networking
  vpc_id     = module.networking.vpc_id
  subnet_ids = module.networking.database_subnet_ids

  # Different sizing per environment
  instance_class = {
    dev     = "db.t3.micro"
    staging = "db.t3.medium"
    prod    = "db.r6g.large"
  }[local.environment]

  multi_az = local.environment == "prod"

  # Allow connections from app security group
  allowed_security_group_ids = [module.compute.app_security_group_id]

  tags = local.common_tags
}

# ===================
# APP MODULE (ECS Service)
# ===================
module "app" {
  source = "./modules/app"

  project     = local.project
  environment = local.environment
  app_name    = "api"

  # From compute
  cluster_arn = module.compute.ecs_cluster_arn

  # From networking
  vpc_id            = module.networking.vpc_id
  private_subnet_ids = module.networking.private_subnet_ids
  public_subnet_ids  = module.networking.public_subnet_ids

  container_image = var.api_image
  container_port  = 8080

  desired_count = {
    dev     = 1
    staging = 2
    prod    = 5
  }[local.environment]

  # Database connection
  environment_variables = {
    DB_HOST = module.database.db_host
    DB_PORT = tostring(module.database.db_port)
    DB_NAME = module.database.db_name
    DB_USER = module.database.db_username
    LOG_LEVEL = local.environment == "prod" ? "INFO" : "DEBUG"
  }

  secrets = {
    DB_PASSWORD = module.database.db_secret_arn
  }

  tags = local.common_tags
}

# ===================
# MONITORING MODULE
# ===================
module "monitoring" {
  source = "./modules/monitoring"

  project     = local.project
  environment = local.environment

  # Monitor all the resources
  ecs_cluster_name = module.compute.ecs_cluster_name
  ecs_service_name = module.app.service_name
  rds_identifier   = module.database.db_identifier

  alb_arn = module.app.alb_arn

  alarm_notifications_arn = var.alarm_sns_arn

  tags = local.common_tags
}

# ===================
# OUTPUTS
# ===================
output "app_url" {
  description = "URL ของ application"
  value       = "https://${module.app.alb_dns_name}"
}

output "vpc_id" {
  description = "VPC ID"
  value       = module.networking.vpc_id
}

output "db_endpoint" {
  description = "Database endpoint"
  value       = module.database.db_endpoint
}

output "ecs_cluster_arn" {
  description = "ECS Cluster ARN"
  value       = module.compute.ecs_cluster_arn
}

# ==========================================
# variables.tf
# ==========================================
variable "environment" {
  type    = string
  default = "dev"

  validation {
    condition     = contains(["dev", "staging", "prod"], var.environment)
    error_message = "Environment must be dev, staging, or prod."
  }
}

variable "api_image" {
  type        = string
  description = "Docker image สำหรับ API service"
}

variable "alarm_sns_arn" {
  type        = string
  description = "SNS Topic ARN สำหรับ alarm notifications"
  default     = null
}
```

---

## สรุป (Summary)

### Module Composition Patterns:

| Pattern | ใช้เมื่อ |
|---------|---------|
| Single-Purpose Module | ต้องการ reusability สูง |
| Wrapper Module | ต้องการ enforce company standards |
| App Module (Composite) | หลาย modules ที่เกี่ยวข้องกัน |
| Environment Module | รวม apps หลายตัวใน environment เดียว |
| Remote State | แยก lifecycle ของ modules ออกจากกัน |

### Best Practices:
1. **Loose Coupling** - modules ไม่ควรรู้จักกันโดยตรง
2. **Pass specific values** - ส่ง specific attributes ไม่ใช่ objects ทั้งก้อน
3. **Avoid circular deps** - ใช้ data sources หรือ remote state แทน
4. **Flat over deep** - 2 levels deep maximum
5. **Remote state** - ใช้ remote state สำหรับ cross-stack communication

---

*จบ Part 066 - Module Composition Patterns*
