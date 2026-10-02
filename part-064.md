# Part 064: Terraform Modules: Deep Dive
## Terraform Modules: เจาะลึก
### Steps 631-640

---

## บทนำ (Introduction)

Modules คือการจัดกลุ่ม Terraform code เพื่อ reuse และ organize infrastructure ที่ดี การเข้าใจ module architecture จะช่วยให้สร้าง infrastructure ที่ maintainable และ scalable ได้

---

## Step 631: Module Architecture Principles

### หลักการออกแบบ Module

```
# โครงสร้าง Module ที่ดี
modules/
├── vpc/                    # Single-purpose module
│   ├── main.tf
│   ├── variables.tf
│   ├── outputs.tf
│   ├── versions.tf
│   └── README.md
├── rds/
│   ├── main.tf
│   ├── variables.tf
│   ├── outputs.tf
│   ├── versions.tf
│   └── README.md
└── ecs-service/
    ├── main.tf
    ├── variables.tf
    ├── outputs.tf
    ├── versions.tf
    └── README.md
```

### หลักการสำคัญ:

**1. Single Responsibility**
- แต่ละ module ทำสิ่งเดียว
- ชัดเจนว่า module สร้างอะไร

**2. Module Interface (Input/Output)**
- Variables = Inputs
- Outputs = Outputs  
- ไม่ดึง data จาก outside โดยตรง (ส่งเข้ามาผ่าน variables)

**3. Composability**
- Modules ควรใช้งานร่วมกันได้
- ไม่ hard-code dependencies

---

## Step 632: Flat vs Nested Module Structures

```
# Flat Structure - แนะนำสำหรับ simple projects
project/
├── main.tf          # root module
├── variables.tf
├── outputs.tf
└── modules/
    ├── networking/
    ├── compute/
    └── database/

# Nested Structure - สำหรับ complex organizations
project/
├── main.tf
└── modules/
    ├── app/           # composite module
    │   ├── main.tf    # uses networking + compute
    │   ├── modules/
    │   │   ├── networking/
    │   │   └── compute/
    │   └── ...
    └── platform/
        ├── main.tf
        └── modules/
            └── database/
```

### Root Module Example:

```hcl
# main.tf (root module)
terraform {
  required_version = ">= 1.5.0"
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
  backend "s3" {
    bucket = "my-terraform-state"
    key    = "prod/terraform.tfstate"
    region = "ap-southeast-1"
  }
}

provider "aws" {
  region = var.region
}

module "vpc" {
  source = "./modules/vpc"

  project     = var.project
  environment = var.environment
  cidr_block  = var.vpc_cidr
}

module "rds" {
  source = "./modules/rds"

  project       = var.project
  environment   = var.environment
  vpc_id        = module.vpc.vpc_id
  subnet_ids    = module.vpc.private_subnet_ids
  instance_class = var.db_instance_class
}

module "ecs_cluster" {
  source = "./modules/ecs-cluster"

  project     = var.project
  environment = var.environment
  vpc_id      = module.vpc.vpc_id
}
```

---

## Step 633: Module Interface Design

```hcl
# ==========================================
# WELL-DESIGNED MODULE INTERFACE
# ==========================================
# modules/rds/variables.tf

# ============
# REQUIRED
# ============
variable "project" {
  type        = string
  description = "(Required) ชื่อโปรเจค"

  validation {
    condition     = can(regex("^[a-z0-9-]+$", var.project))
    error_message = "Project must be lowercase alphanumeric with hyphens."
  }
}

variable "environment" {
  type        = string
  description = "(Required) สภาพแวดล้อม"

  validation {
    condition     = contains(["dev", "staging", "prod"], var.environment)
    error_message = "Environment must be dev, staging, or prod."
  }
}

variable "vpc_id" {
  type        = string
  description = "(Required) ID ของ VPC"

  validation {
    condition     = can(regex("^vpc-[a-f0-9]+$", var.vpc_id))
    error_message = "VPC ID must be a valid AWS VPC ID."
  }
}

variable "subnet_ids" {
  type        = list(string)
  description = "(Required) IDs ของ subnets (ต้องมีอย่างน้อย 2)"

  validation {
    condition     = length(var.subnet_ids) >= 2
    error_message = "At least 2 subnet IDs required for multi-AZ."
  }
}

# ============
# OPTIONAL
# ============
variable "instance_class" {
  type        = string
  description = "(Optional) RDS instance class"
  default     = "db.t3.micro"
}

variable "engine" {
  type        = string
  description = "(Optional) Database engine"
  default     = "postgres"

  validation {
    condition     = contains(["mysql", "postgres", "mariadb"], var.engine)
    error_message = "Engine must be mysql, postgres, or mariadb."
  }
}

variable "engine_version" {
  type        = string
  description = "(Optional) Database engine version"
  default     = "15.4"
}

variable "allocated_storage" {
  type        = number
  description = "(Optional) Storage ขนาด GB"
  default     = 20

  validation {
    condition     = var.allocated_storage >= 20
    error_message = "Minimum storage is 20 GB."
  }
}

variable "max_allocated_storage" {
  type        = number
  description = "(Optional) Maximum storage GB (สำหรับ autoscaling)"
  default     = 100
}

variable "multi_az" {
  type        = bool
  description = "(Optional) เปิด Multi-AZ"
  default     = false
}

variable "backup_retention_period" {
  type        = number
  description = "(Optional) จำนวนวันเก็บ backup"
  default     = 7
}

variable "deletion_protection" {
  type        = bool
  description = "(Optional) ป้องกันการลบ database"
  default     = true
}

variable "tags" {
  type        = map(string)
  description = "(Optional) Tags เพิ่มเติม"
  default     = {}
}
```

---

## Step 634: Keeping Modules Small and Focused

```hcl
# ==========================================
# GOOD: Small, focused module
# ==========================================
# modules/rds-instance/main.tf
# ทำแค่ RDS instance และ security group

resource "aws_db_instance" "main" {
  identifier        = "${var.project}-${var.environment}-db"
  engine            = var.engine
  engine_version    = var.engine_version
  instance_class    = var.instance_class
  allocated_storage = var.allocated_storage

  db_subnet_group_name   = aws_db_subnet_group.main.name
  vpc_security_group_ids = [aws_security_group.rds.id]

  multi_az                = var.multi_az
  backup_retention_period = var.backup_retention_period
  deletion_protection     = var.deletion_protection

  tags = merge(
    {
      Name        = "${var.project}-${var.environment}-db"
      Project     = var.project
      Environment = var.environment
    },
    var.tags
  )
}

resource "aws_db_subnet_group" "main" {
  name       = "${var.project}-${var.environment}-db-subnet-group"
  subnet_ids = var.subnet_ids

  tags = {
    Name        = "${var.project}-${var.environment}-db-subnet-group"
    Project     = var.project
    Environment = var.environment
  }
}

resource "aws_security_group" "rds" {
  name        = "${var.project}-${var.environment}-rds-sg"
  description = "Security group for RDS - ${var.project} ${var.environment}"
  vpc_id      = var.vpc_id

  tags = {
    Name        = "${var.project}-${var.environment}-rds-sg"
    Project     = var.project
    Environment = var.environment
  }
}

# ==========================================
# BAD: Overly broad module (ห้ามทำ)
# ==========================================
# module "everything" - สร้างทั้ง VPC, EC2, RDS, S3, IAM, ฯลฯ
# ปัญหา: ยาก maintain, ยาก test, ยาก reuse

# ==========================================
# modules/rds-instance/outputs.tf
# ==========================================

output "db_instance_id" {
  value = aws_db_instance.main.id
}

output "db_endpoint" {
  value = aws_db_instance.main.endpoint
}

output "db_port" {
  value = aws_db_instance.main.port
}

output "security_group_id" {
  value = aws_security_group.rds.id
}

output "subnet_group_name" {
  value = aws_db_subnet_group.main.name
}
```

---

## Step 635: Module Documentation with terraform-docs

### การสร้าง documentation อัตโนมัติ

```yaml
# .terraform-docs.yml
formatter: "markdown table"

version: ""

header-from: main.tf
footer-from: ""

recursive:
  enabled: false
  path: modules

sections:
  hide: []
  show: []

content: |-
  {{ .Header }}

  ## Usage

  ```hcl
  module "vpc" {
    source  = "registry.terraform.io/myorg/vpc/aws"
    version = "~> 2.0"

    project     = "myapp"
    environment = "prod"
    cidr_block  = "10.0.0.0/16"
  }
  ```

  {{ .Inputs }}

  {{ .Outputs }}

  {{ .Providers }}

  {{ .Requirements }}

  {{ .Resources }}

output:
  file: README.md
  mode: replace
  template: |-
    <!-- BEGIN_TF_DOCS -->
    {{ .Content }}
    <!-- END_TF_DOCS -->

sort:
  enabled: true
  by: name

settings:
  anchor: true
  color: true
  default: true
  description: true
  escape: true
  hide-empty: false
  html: true
  indent: 2
  lockfile: true
  read-comments: true
  required: true
  sensitive: true
  type: true
```

```bash
# ติดตั้ง terraform-docs
brew install terraform-docs

# หรือ
curl -Lo ./terraform-docs.tar.gz https://github.com/terraform-docs/terraform-docs/releases/download/v0.17.0/terraform-docs-v0.17.0-linux-amd64.tar.gz
tar -xzf terraform-docs.tar.gz
chmod +x terraform-docs
mv terraform-docs /usr/local/bin/

# Generate README
terraform-docs markdown table --output-file README.md .

# Generate สำหรับทุก modules
for dir in modules/*/; do
  terraform-docs markdown table --output-file "${dir}README.md" "$dir"
done

# หรือใช้กับ pre-commit
# .pre-commit-config.yaml
repos:
  - repo: https://github.com/terraform-docs/gh-actions
    hooks:
      - id: terraform-docs-go
        args: ["markdown", "table", "--output-file", "README.md", "./"]
```

---

## Step 636: Module Testing Strategies

### Unit Testing ด้วย terraform test (Terraform 1.6+)

```hcl
# tests/vpc_test.tftest.hcl

# Test 1: Basic VPC creation
run "basic_vpc_creation" {
  command = plan

  variables {
    project     = "test-project"
    environment = "dev"
    cidr_block  = "10.0.0.0/16"
  }

  assert {
    condition     = aws_vpc.main.cidr_block == "10.0.0.0/16"
    error_message = "VPC CIDR block should be 10.0.0.0/16"
  }

  assert {
    condition     = aws_vpc.main.enable_dns_hostnames == true
    error_message = "DNS hostnames should be enabled"
  }
}

# Test 2: VPC with custom options
run "vpc_with_custom_options" {
  command = apply

  variables {
    project              = "test-project"
    environment          = "dev"
    cidr_block           = "10.1.0.0/16"
    enable_nat_gateway   = false
    enable_vpn_gateway   = false
  }

  assert {
    condition     = output.vpc_id != ""
    error_message = "VPC ID should not be empty"
  }

  assert {
    condition     = length(output.private_subnet_ids) >= 2
    error_message = "Should have at least 2 private subnets"
  }
}

# Test 3: Cleanup
run "cleanup" {
  command = destroy
}
```

```hcl
# tests/rds_test.tftest.hcl

provider "aws" {
  region = "ap-southeast-1"
}

# Setup: สร้าง VPC ก่อน
run "setup_vpc" {
  command = apply
  module {
    source = "./tests/fixtures/vpc"
  }
}

# Test RDS module
run "test_rds_creation" {
  command = apply

  variables {
    project             = "test"
    environment         = "dev"
    vpc_id              = run.setup_vpc.output.vpc_id
    subnet_ids          = run.setup_vpc.output.private_subnet_ids
    instance_class      = "db.t3.micro"
    deletion_protection = false
  }

  assert {
    condition     = output.db_endpoint != ""
    error_message = "Database endpoint should not be empty"
  }

  assert {
    condition     = output.db_port == 5432
    error_message = "PostgreSQL should use port 5432"
  }
}

# Cleanup
run "cleanup" {
  command = destroy
}
```

### Integration Testing ด้วย Terratest (Go)

```go
// test/vpc_test.go
package test

import (
    "testing"
    "github.com/gruntwork-io/terratest/modules/terraform"
    "github.com/gruntwork-io/terratest/modules/aws"
    "github.com/stretchr/testify/assert"
)

func TestVPCModule(t *testing.T) {
    t.Parallel()

    terraformOptions := terraform.WithDefaultRetryableErrors(t, &terraform.Options{
        TerraformDir: "../modules/vpc",
        Vars: map[string]interface{}{
            "project":     "test-vpc",
            "environment": "dev",
            "cidr_block":  "10.0.0.0/16",
        },
    })

    defer terraform.Destroy(t, terraformOptions)
    terraform.InitAndApply(t, terraformOptions)

    // Get outputs
    vpcID := terraform.Output(t, terraformOptions, "vpc_id")
    privateSubnetIDs := terraform.OutputList(t, terraformOptions, "private_subnet_ids")
    publicSubnetIDs := terraform.OutputList(t, terraformOptions, "public_subnet_ids")

    // Assertions
    assert.NotEmpty(t, vpcID, "VPC ID should not be empty")
    assert.GreaterOrEqual(t, len(privateSubnetIDs), 2, "Should have at least 2 private subnets")
    assert.GreaterOrEqual(t, len(publicSubnetIDs), 2, "Should have at least 2 public subnets")

    // Verify VPC actually exists in AWS
    vpc := aws.GetVpcById(t, vpcID, "ap-southeast-1")
    assert.Equal(t, "10.0.0.0/16", vpc.CidrBlock)
}

func TestRDSModule(t *testing.T) {
    t.Parallel()

    // First create VPC
    vpcOptions := terraform.WithDefaultRetryableErrors(t, &terraform.Options{
        TerraformDir: "./fixtures/vpc",
    })
    defer terraform.Destroy(t, vpcOptions)
    terraform.InitAndApply(t, vpcOptions)

    vpcID := terraform.Output(t, vpcOptions, "vpc_id")
    subnetIDs := terraform.OutputList(t, vpcOptions, "private_subnet_ids")

    // Then test RDS
    rdsOptions := terraform.WithDefaultRetryableErrors(t, &terraform.Options{
        TerraformDir: "../modules/rds",
        Vars: map[string]interface{}{
            "project":             "test",
            "environment":         "dev",
            "vpc_id":              vpcID,
            "subnet_ids":          subnetIDs,
            "deletion_protection": false,
        },
    })

    defer terraform.Destroy(t, rdsOptions)
    terraform.InitAndApply(t, rdsOptions)

    endpoint := terraform.Output(t, rdsOptions, "db_endpoint")
    assert.NotEmpty(t, endpoint, "Database endpoint should not be empty")
}
```

---

## Step 637: Module Versioning in Git

```bash
# Module versioning ด้วย Git tags

# สร้าง tag สำหรับ module version
git tag -a "modules/vpc/v1.0.0" -m "Initial VPC module release"
git tag -a "modules/vpc/v1.1.0" -m "Add NAT gateway support"
git tag -a "modules/vpc/v2.0.0" -m "Breaking change: rename cidr to cidr_block"

# Push tags
git push origin --tags

# การใช้งาน module จาก Git tag
module "vpc" {
  source = "git::https://github.com/myorg/terraform-modules.git//modules/vpc?ref=modules/vpc/v1.1.0"
  
  project    = "myapp"
  cidr_block = "10.0.0.0/16"
}
```

### Monorepo vs Separate Repos:

```hcl
# ==========================================
# MONOREPO APPROACH
# ==========================================
# ข้อดี: ทุก module อยู่ที่เดียว, ง่ายต่อการ develop
# ข้อเสีย: tag ต้อง specific ถึง module path

# structure:
# terraform-modules/ (single repo)
# ├── modules/
# │   ├── vpc/
# │   ├── rds/
# │   └── ecs/
# └── examples/

module "vpc" {
  source = "git::https://github.com/myorg/terraform-modules.git//modules/vpc?ref=v1.0.0"
}

# ==========================================
# SEPARATE REPOS APPROACH
# ==========================================
# ข้อดี: version ชัดเจน per module, independent release
# ข้อเสีย: หลาย repos ยากต่อการ manage

# repos:
# terraform-aws-vpc
# terraform-aws-rds
# terraform-aws-ecs

module "vpc" {
  source  = "git::https://github.com/myorg/terraform-aws-vpc.git?ref=v1.0.0"
}

module "rds" {
  source  = "git::https://github.com/myorg/terraform-aws-rds.git?ref=v2.1.0"
}
```

---

## Step 638: Complete VPC Module Example

```hcl
# ==========================================
# COMPLETE VPC MODULE
# modules/vpc/versions.tf
# ==========================================

terraform {
  required_version = ">= 1.5.0"
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = ">= 5.0"
    }
  }
}

# ==========================================
# modules/vpc/variables.tf
# ==========================================

variable "project" {
  type        = string
  description = "(Required) ชื่อโปรเจค"
}

variable "environment" {
  type        = string
  description = "(Required) สภาพแวดล้อม"

  validation {
    condition     = contains(["dev", "staging", "prod"], var.environment)
    error_message = "Environment must be dev, staging, or prod."
  }
}

variable "cidr_block" {
  type        = string
  description = "(Required) CIDR block สำหรับ VPC"

  validation {
    condition     = can(cidrhost(var.cidr_block, 0))
    error_message = "Must be a valid IPv4 CIDR block."
  }
}

variable "azs" {
  type        = list(string)
  description = "(Optional) Availability zones. Defaults to first 2 AZs in region."
  default     = []
}

variable "public_subnet_cidrs" {
  type        = list(string)
  description = "(Optional) CIDRs สำหรับ public subnets"
  default     = []
}

variable "private_subnet_cidrs" {
  type        = list(string)
  description = "(Optional) CIDRs สำหรับ private subnets"
  default     = []
}

variable "database_subnet_cidrs" {
  type        = list(string)
  description = "(Optional) CIDRs สำหรับ database subnets"
  default     = []
}

variable "enable_nat_gateway" {
  type        = bool
  description = "(Optional) สร้าง NAT Gateway"
  default     = true
}

variable "single_nat_gateway" {
  type        = bool
  description = "(Optional) ใช้ NAT Gateway เดียว"
  default     = false
}

variable "enable_flow_logs" {
  type        = bool
  description = "(Optional) เปิด VPC Flow Logs"
  default     = false
}

variable "flow_logs_retention_days" {
  type        = number
  description = "(Optional) จำนวนวันเก็บ Flow Logs"
  default     = 30
}

variable "tags" {
  type        = map(string)
  description = "(Optional) Tags เพิ่มเติม"
  default     = {}
}

# ==========================================
# modules/vpc/main.tf
# ==========================================

# Data: Get available AZs
data "aws_availability_zones" "available" {
  state = "available"
}

locals {
  # Use provided AZs or default to first 2
  azs = length(var.azs) > 0 ? var.azs : slice(data.aws_availability_zones.available.names, 0, 2)
  az_count = length(local.azs)

  # Calculate subnet CIDRs if not provided
  # Splits VPC CIDR into /24 subnets
  public_subnet_cidrs = length(var.public_subnet_cidrs) > 0 ? var.public_subnet_cidrs : [
    for i in range(local.az_count) : cidrsubnet(var.cidr_block, 8, i)
  ]

  private_subnet_cidrs = length(var.private_subnet_cidrs) > 0 ? var.private_subnet_cidrs : [
    for i in range(local.az_count) : cidrsubnet(var.cidr_block, 8, i + 10)
  ]

  database_subnet_cidrs = length(var.database_subnet_cidrs) > 0 ? var.database_subnet_cidrs : [
    for i in range(local.az_count) : cidrsubnet(var.cidr_block, 8, i + 20)
  ]

  # Common tags
  common_tags = merge(
    {
      Project     = var.project
      Environment = var.environment
      ManagedBy   = "Terraform"
    },
    var.tags
  )

  # NAT gateway count
  nat_gateway_count = var.enable_nat_gateway ? (var.single_nat_gateway ? 1 : local.az_count) : 0
}

# VPC
resource "aws_vpc" "main" {
  cidr_block           = var.cidr_block
  enable_dns_hostnames = true
  enable_dns_support   = true

  tags = merge(local.common_tags, {
    Name = "${var.project}-${var.environment}-vpc"
  })
}

# Internet Gateway
resource "aws_internet_gateway" "main" {
  vpc_id = aws_vpc.main.id

  tags = merge(local.common_tags, {
    Name = "${var.project}-${var.environment}-igw"
  })
}

# Public Subnets
resource "aws_subnet" "public" {
  count = local.az_count

  vpc_id                  = aws_vpc.main.id
  cidr_block              = local.public_subnet_cidrs[count.index]
  availability_zone       = local.azs[count.index]
  map_public_ip_on_launch = true

  tags = merge(local.common_tags, {
    Name = "${var.project}-${var.environment}-public-${local.azs[count.index]}"
    Tier = "public"
  })
}

# Private Subnets
resource "aws_subnet" "private" {
  count = local.az_count

  vpc_id            = aws_vpc.main.id
  cidr_block        = local.private_subnet_cidrs[count.index]
  availability_zone = local.azs[count.index]

  tags = merge(local.common_tags, {
    Name = "${var.project}-${var.environment}-private-${local.azs[count.index]}"
    Tier = "private"
  })
}

# Database Subnets
resource "aws_subnet" "database" {
  count = local.az_count

  vpc_id            = aws_vpc.main.id
  cidr_block        = local.database_subnet_cidrs[count.index]
  availability_zone = local.azs[count.index]

  tags = merge(local.common_tags, {
    Name = "${var.project}-${var.environment}-database-${local.azs[count.index]}"
    Tier = "database"
  })
}

# Elastic IPs for NAT Gateways
resource "aws_eip" "nat" {
  count  = local.nat_gateway_count
  domain = "vpc"

  depends_on = [aws_internet_gateway.main]

  tags = merge(local.common_tags, {
    Name = "${var.project}-${var.environment}-nat-eip-${count.index + 1}"
  })
}

# NAT Gateways
resource "aws_nat_gateway" "main" {
  count = local.nat_gateway_count

  allocation_id = aws_eip.nat[count.index].id
  subnet_id     = aws_subnet.public[count.index].id

  depends_on = [aws_internet_gateway.main]

  tags = merge(local.common_tags, {
    Name = "${var.project}-${var.environment}-nat-${count.index + 1}"
  })
}

# Public Route Table
resource "aws_route_table" "public" {
  vpc_id = aws_vpc.main.id

  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.main.id
  }

  tags = merge(local.common_tags, {
    Name = "${var.project}-${var.environment}-public-rt"
  })
}

# Associate Public Subnets with Public Route Table
resource "aws_route_table_association" "public" {
  count = local.az_count

  subnet_id      = aws_subnet.public[count.index].id
  route_table_id = aws_route_table.public.id
}

# Private Route Tables (one per NAT GW or one shared)
resource "aws_route_table" "private" {
  count  = local.az_count
  vpc_id = aws_vpc.main.id

  dynamic "route" {
    for_each = var.enable_nat_gateway ? [1] : []
    content {
      cidr_block     = "0.0.0.0/0"
      nat_gateway_id = var.single_nat_gateway ? aws_nat_gateway.main[0].id : aws_nat_gateway.main[count.index].id
    }
  }

  tags = merge(local.common_tags, {
    Name = "${var.project}-${var.environment}-private-rt-${local.azs[count.index]}"
  })
}

# Associate Private Subnets
resource "aws_route_table_association" "private" {
  count = local.az_count

  subnet_id      = aws_subnet.private[count.index].id
  route_table_id = aws_route_table.private[count.index].id
}

# Database Route Table (uses private route)
resource "aws_route_table_association" "database" {
  count = local.az_count

  subnet_id      = aws_subnet.database[count.index].id
  route_table_id = aws_route_table.private[count.index].id
}

# VPC Flow Logs
resource "aws_cloudwatch_log_group" "vpc_flow_logs" {
  count = var.enable_flow_logs ? 1 : 0

  name              = "/aws/vpc-flow-logs/${var.project}-${var.environment}"
  retention_in_days = var.flow_logs_retention_days

  tags = local.common_tags
}

resource "aws_iam_role" "vpc_flow_logs" {
  count = var.enable_flow_logs ? 1 : 0
  name  = "${var.project}-${var.environment}-vpc-flow-logs-role"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Action = "sts:AssumeRole"
      Effect = "Allow"
      Principal = {
        Service = "vpc-flow-logs.amazonaws.com"
      }
    }]
  })

  tags = local.common_tags
}

resource "aws_iam_role_policy" "vpc_flow_logs" {
  count = var.enable_flow_logs ? 1 : 0
  name  = "vpc-flow-logs-policy"
  role  = aws_iam_role.vpc_flow_logs[0].id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect = "Allow"
      Action = [
        "logs:CreateLogGroup",
        "logs:CreateLogStream",
        "logs:PutLogEvents",
        "logs:DescribeLogGroups",
        "logs:DescribeLogStreams"
      ]
      Resource = "*"
    }]
  })
}

resource "aws_flow_log" "main" {
  count = var.enable_flow_logs ? 1 : 0

  vpc_id          = aws_vpc.main.id
  traffic_type    = "ALL"
  iam_role_arn    = aws_iam_role.vpc_flow_logs[0].arn
  log_destination = aws_cloudwatch_log_group.vpc_flow_logs[0].arn

  tags = merge(local.common_tags, {
    Name = "${var.project}-${var.environment}-flow-logs"
  })
}

# ==========================================
# modules/vpc/outputs.tf
# ==========================================

output "vpc_id" {
  description = "ID ของ VPC"
  value       = aws_vpc.main.id
}

output "vpc_arn" {
  description = "ARN ของ VPC"
  value       = aws_vpc.main.arn
}

output "vpc_cidr_block" {
  description = "CIDR block ของ VPC"
  value       = aws_vpc.main.cidr_block
}

output "public_subnet_ids" {
  description = "IDs ของ public subnets"
  value       = aws_subnet.public[*].id
}

output "private_subnet_ids" {
  description = "IDs ของ private subnets"
  value       = aws_subnet.private[*].id
}

output "database_subnet_ids" {
  description = "IDs ของ database subnets"
  value       = aws_subnet.database[*].id
}

output "internet_gateway_id" {
  description = "ID ของ Internet Gateway"
  value       = aws_internet_gateway.main.id
}

output "nat_gateway_ids" {
  description = "IDs ของ NAT Gateways"
  value       = aws_nat_gateway.main[*].id
}

output "nat_public_ips" {
  description = "Public IPs ของ NAT Gateways"
  value       = aws_eip.nat[*].public_ip
}

output "public_route_table_id" {
  description = "ID ของ public route table"
  value       = aws_route_table.public.id
}

output "private_route_table_ids" {
  description = "IDs ของ private route tables"
  value       = aws_route_table.private[*].id
}

output "availability_zones" {
  description = "Availability zones ที่ใช้"
  value       = local.azs
}

output "vpc_summary" {
  description = "สรุปข้อมูล VPC"
  value = {
    id         = aws_vpc.main.id
    cidr       = aws_vpc.main.cidr_block
    azs        = local.azs
    public_subnets   = aws_subnet.public[*].id
    private_subnets  = aws_subnet.private[*].id
    database_subnets = aws_subnet.database[*].id
    nat_ips          = aws_eip.nat[*].public_ip
  }
}
```

---

## Step 639: Module Defaults vs Required Inputs

### แนวทางในการกำหนด defaults

```hcl
# ==========================================
# MODULE DEFAULTS STRATEGY
# ==========================================

# Rule 1: Required inputs - ไม่มี default
# สิ่งที่ต้องการข้อมูล specific ต่อ deployment

variable "project" { type = string }    # Required
variable "environment" { type = string } # Required
variable "vpc_id" { type = string }      # Required - specific resource

# Rule 2: Optional with sensible defaults
# สิ่งที่มีค่า "ปกติ" ที่ใช้ได้กับส่วนใหญ่

variable "enable_deletion_protection" {
  type    = bool
  default = true   # ปลอดภัยกว่า false
}

variable "backup_retention_days" {
  type    = number
  default = 7      # 7 วัน เป็น standard
}

variable "log_retention_days" {
  type    = number
  default = 30     # 30 วัน เป็น common
}

# Rule 3: Empty default - optional features
variable "extra_tags" {
  type    = map(string)
  default = {}     # ไม่ต้องมี tags เพิ่ม
}

variable "additional_security_group_ids" {
  type    = list(string)
  default = []     # ไม่มี additional SGs
}

# Rule 4: null default - explicitly optional
variable "kms_key_arn" {
  type    = string
  default = null   # null = ใช้ AWS managed key
}

variable "parameter_group_name" {
  type    = string
  default = null   # null = ใช้ default parameter group
}
```

---

## Step 640: Module Compatibility Matrix

```hcl
# ==========================================
# MODULE COMPATIBILITY
# ==========================================
# modules/vpc/versions.tf

terraform {
  required_version = ">= 1.3.0"  # ต้องการ 1.3+ สำหรับ optional()

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = ">= 4.0, < 6.0"  # Compatible ทั้ง v4 และ v5
    }
  }
}
```

```markdown
# COMPATIBILITY.md

## Version Compatibility Matrix

| Module Version | Terraform | AWS Provider | Notes |
|---------------|-----------|--------------|-------|
| v1.x          | >= 0.14   | >= 3.0, < 5.0 | Legacy |
| v2.x          | >= 1.0    | >= 4.0, < 6.0 | Current |
| v3.x (planned)| >= 1.5    | >= 5.0         | Future |

## Breaking Changes

### v2.0.0
- Renamed `cidr` to `cidr_block`
- Removed `enable_classiclink` (AWS deprecated ClassicLink)
- `azs` now defaults to empty list (uses data source)

### v1.1.0 -> v1.2.0 (non-breaking)
- Added `enable_flow_logs` option
- Added `flow_logs_retention_days` option
```

---

## สรุป (Summary)

### Module Design Checklist:

- [ ] Single responsibility - module ทำสิ่งเดียวชัดเจน
- [ ] Clean interface - variables + outputs ที่สื่อความหมาย
- [ ] Sensible defaults - optional inputs ที่มี default ที่ดี
- [ ] Validation - ตรวจสอบ input ก่อนใช้
- [ ] Documentation - README ที่ครบถ้วน
- [ ] Testing - terraform test หรือ Terratest
- [ ] Versioning - Git tags ที่ชัดเจน
- [ ] Outputs - ออก output ทุกอย่างที่ consumer อาจต้องการ

---

*จบ Part 064 - Terraform Modules: Deep Dive*
