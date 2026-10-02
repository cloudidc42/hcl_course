# Part 014: HCL Modules Overview (ภาพรวม Modules)
## Steps 131-140: การใช้งาน Terraform Modules

---

## บทนำ (Introduction)

Modules คือกลุ่มของ Terraform resources ที่ถูก package ไว้ด้วยกันเพื่อนำกลับมาใช้ซ้ำ (reusable) เปรียบได้กับ "functions" หรือ "libraries" ในภาษา programming ทั่วไป

### ประโยชน์ของ Modules

1. **Reusability** - เขียนครั้งเดียว ใช้ได้หลายที่
2. **Abstraction** - ซ่อน complexity ภายใน
3. **Consistency** - enforce standards ได้
4. **Collaboration** - แบ่งงานกันทำได้
5. **Testing** - test module แยกกันได้
6. **Version Control** - จัดการ version ของ infrastructure ได้

---

## Step 131: Root Module vs Child Modules

### Root Module

Root module คือ Terraform configuration ใน working directory ที่ run `terraform apply`

```
my-infrastructure/
├── main.tf         <- root module
├── variables.tf
├── outputs.tf
├── locals.tf
└── versions.tf
```

### Child Modules

Child modules คือ module ที่ถูกเรียกใช้จาก root module หรือ module อื่น

```
my-infrastructure/
├── main.tf              <- root module ที่เรียก child modules
├── variables.tf
├── outputs.tf
└── modules/             <- child modules directory
    ├── vpc/
    │   ├── main.tf
    │   ├── variables.tf
    │   └── outputs.tf
    ├── ec2/
    │   ├── main.tf
    │   ├── variables.tf
    │   └── outputs.tf
    └── rds/
        ├── main.tf
        ├── variables.tf
        └── outputs.tf
```

```hcl
# root module: main.tf
module "vpc" {
  source = "./modules/vpc"
  # ...
}

module "ec2" {
  source = "./modules/ec2"
  # ...
}
```

---

## Step 132: Module Directory Structure

### โครงสร้างไฟล์มาตรฐาน

```
module-name/
├── main.tf          # resources หลักของ module
├── variables.tf     # input variables
├── outputs.tf       # output values
├── locals.tf        # local values (optional)
├── versions.tf      # terraform/provider requirements
├── README.md        # documentation
└── examples/        # ตัวอย่างการใช้งาน (optional)
    └── basic/
        ├── main.tf
        └── README.md
```

### main.tf - Resources หลัก

```hcl
# modules/vpc/main.tf

terraform {
  required_version = ">= 1.0"
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = ">= 4.0"
    }
  }
}

resource "aws_vpc" "main" {
  cidr_block           = var.cidr_block
  enable_dns_support   = var.enable_dns_support
  enable_dns_hostnames = var.enable_dns_hostnames

  tags = merge(var.tags, {
    Name = var.name
  })
}

resource "aws_internet_gateway" "main" {
  count  = var.create_igw ? 1 : 0
  vpc_id = aws_vpc.main.id

  tags = merge(var.tags, {
    Name = "${var.name}-igw"
  })
}
```

### variables.tf - Input Variables

```hcl
# modules/vpc/variables.tf

variable "name" {
  description = "Name prefix for all resources"
  type        = string
  nullable    = false
}

variable "cidr_block" {
  description = "CIDR block for the VPC"
  type        = string
  default     = "10.0.0.0/16"

  validation {
    condition     = can(cidrhost(var.cidr_block, 0))
    error_message = "Must be a valid CIDR block."
  }
}

variable "enable_dns_support" {
  description = "Enable DNS support in the VPC"
  type        = bool
  default     = true
}

variable "enable_dns_hostnames" {
  description = "Enable DNS hostnames in the VPC"
  type        = bool
  default     = true
}

variable "create_igw" {
  description = "Whether to create an Internet Gateway"
  type        = bool
  default     = true
}

variable "tags" {
  description = "Tags to apply to all resources"
  type        = map(string)
  default     = {}
}
```

### outputs.tf - Output Values

```hcl
# modules/vpc/outputs.tf

output "vpc_id" {
  description = "ID of the VPC"
  value       = aws_vpc.main.id
}

output "vpc_arn" {
  description = "ARN of the VPC"
  value       = aws_vpc.main.arn
}

output "vpc_cidr_block" {
  description = "CIDR block of the VPC"
  value       = aws_vpc.main.cidr_block
}

output "internet_gateway_id" {
  description = "ID of the Internet Gateway"
  value       = var.create_igw ? aws_internet_gateway.main[0].id : null
}
```

### versions.tf - Version Constraints

```hcl
# modules/vpc/versions.tf

terraform {
  required_version = ">= 1.3.0"

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = ">= 5.0.0, < 6.0.0"
    }
  }
}
```

---

## Step 133: Module Sources

### 1. Local Path

```hcl
# ใช้ relative path
module "vpc" {
  source = "./modules/vpc"
}

module "compute" {
  source = "../shared-modules/compute"
}

# ใช้ absolute path (ไม่แนะนำ)
module "network" {
  source = "/home/user/modules/network"
}
```

### 2. Git Source

```hcl
# GitHub (HTTPS)
module "vpc" {
  source = "github.com/org/terraform-aws-vpc"
}

# GitHub (SSH)  
module "vpc" {
  source = "git@github.com:org/terraform-aws-vpc.git"
}

# Specific branch
module "vpc" {
  source = "github.com/org/terraform-aws-vpc//modules/vpc?ref=main"
}

# Specific tag
module "vpc" {
  source = "github.com/org/terraform-aws-vpc//modules/vpc?ref=v2.0.0"
}

# Specific commit
module "vpc" {
  source = "github.com/org/terraform-aws-vpc//modules/vpc?ref=abc1234"
}

# Subdirectory (// คือ subdirectory separator)
module "vpc" {
  source = "github.com/org/terraform-modules//aws/vpc?ref=v1.0.0"
}
```

### 3. Terraform Registry

```hcl
# รูปแบบ: <NAMESPACE>/<MODULE>/<PROVIDER>
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "~> 5.0"
}

module "eks" {
  source  = "terraform-aws-modules/eks/aws"
  version = "~> 20.0"
}

module "rds" {
  source  = "terraform-aws-modules/rds/aws"
  version = "~> 6.0"
}

module "s3_bucket" {
  source  = "terraform-aws-modules/s3-bucket/aws"
  version = "~> 4.0"
}

# Private registry (Terraform Cloud/Enterprise)
module "vpc" {
  source  = "app.terraform.io/my-org/vpc/aws"
  version = "~> 2.0"
}
```

### 4. HTTP URL

```hcl
# Download from HTTP
module "vpc" {
  source = "https://example.com/modules/vpc.zip"
}

# S3 bucket
module "vpc" {
  source = "s3::https://s3-eu-west-1.amazonaws.com/my-bucket/modules/vpc.zip"
}

# Google Cloud Storage
module "vpc" {
  source = "gcs::https://www.googleapis.com/storage/v1/my-bucket/modules/vpc.zip"
}
```

---

## Step 134: Module Block Syntax

### การเรียกใช้ Module

```hcl
module "<local_name>" {
  source  = "<source>"
  version = "<version_constraint>"  # เฉพาะ registry/git

  # Input variables ของ module
  variable_name_1 = value1
  variable_name_2 = value2
  
  # Meta-arguments
  count      = <number>
  for_each   = <map or set>
  depends_on = [<dependencies>]
  providers  = { <provider aliases> }
}
```

### ตัวอย่างการเรียกใช้ Module

```hcl
# main.tf - Root module

# VPC Module
module "vpc" {
  source = "./modules/vpc"

  name         = "${var.project_name}-${var.environment}"
  cidr_block   = var.vpc_cidr
  
  azs                 = var.availability_zones
  public_subnets      = var.public_subnet_cidrs
  private_subnets     = var.private_subnet_cidrs
  
  enable_nat_gateway = true
  single_nat_gateway = var.environment != "prod"
  
  tags = local.common_tags
}

# EC2 Module - ใช้ outputs จาก vpc module
module "web_servers" {
  source = "./modules/ec2"

  name          = "${var.project_name}-web"
  instance_type = var.web_instance_type
  instance_count = var.web_instance_count
  
  vpc_id     = module.vpc.vpc_id           # จาก vpc module output
  subnet_ids = module.vpc.public_subnet_ids # จาก vpc module output
  
  tags = local.compute_tags
}

# RDS Module
module "database" {
  source = "./modules/rds"
  
  identifier     = "${var.project_name}-${var.environment}"
  engine         = "mysql"
  engine_version = "8.0"
  
  vpc_id     = module.vpc.vpc_id
  subnet_ids = module.vpc.private_subnet_ids
  
  db_name  = var.db_name
  username = var.db_username
  password = var.db_password
  
  tags = local.database_tags
  
  depends_on = [module.vpc]
}
```

---

## Step 135: Module Version Constraints

### Version Constraint Syntax

```hcl
# Exact version
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "5.1.2"
}

# Greater than or equal
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = ">= 5.0.0"
}

# Pessimistic constraint (~>) - แนะนำสำหรับ semver
# ~> 5.0 = >= 5.0, < 6.0
# ~> 5.1 = >= 5.1, < 5.2
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "~> 5.0"  # ยอมรับ 5.x.x
}

module "eks" {
  source  = "terraform-aws-modules/eks/aws"
  version = "~> 20.1"  # ยอมรับ 20.1.x
}

# Range
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = ">= 5.0.0, < 6.0.0"
}

# Not equal
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = ">= 5.0.0, != 5.1.0"  # skip buggy version
}
```

---

## Step 136: module.name.output_name Syntax

### การอ้างอิง Module Outputs

```hcl
# module.<module_name>.<output_name>

resource "aws_security_group" "app" {
  vpc_id = module.vpc.vpc_id  # อ้างอิง output ของ vpc module
  
  ingress {
    from_port   = 8080
    to_port     = 8080
    protocol    = "tcp"
    cidr_blocks = [module.vpc.vpc_cidr_block]  # ใช้ CIDR ของ VPC
  }
}

resource "aws_db_subnet_group" "main" {
  subnet_ids = module.vpc.private_subnet_ids  # list output
}

# ใช้ output ใน local
locals {
  web_urls = [
    for ip in module.web_servers.instance_public_ips :
      "http://${ip}:80"
  ]
}
```

### Multiple Instances ของ Module

```hcl
# count บน module
module "app_server" {
  count  = var.instance_count
  source = "./modules/ec2"
  
  name      = "${var.project}-app-${count.index + 1}"
  subnet_id = module.vpc.private_subnet_ids[count.index % length(module.vpc.private_subnet_ids)]
  # ...
}

# อ้างอิง module output เมื่อใช้ count
output "app_server_ids" {
  value = module.app_server[*].instance_id  # list ของ outputs
}

# for_each บน module
module "environments" {
  for_each = toset(["dev", "staging", "prod"])
  source   = "./modules/environment"
  
  environment = each.key
  # ...
}

# อ้างอิง module output เมื่อใช้ for_each
output "environment_vpc_ids" {
  value = {
    for env, mod in module.environments : env => mod.vpc_id
  }
}
```

---

## Step 137: When to Create a Module

### เกณฑ์การตัดสินใจ

```
สร้าง Module เมื่อ:
✅ ใช้ resource group เดิมซ้ำมากกว่า 2 ครั้ง
✅ ต้องการ enforce standards (naming, tagging, security)
✅ ต้องการ abstract ความซับซ้อน
✅ ทีมต่างกันต้องการใช้ infrastructure เดียวกัน
✅ ต้องการ test infrastructure pattern
✅ มี configuration ที่เปลี่ยนแปลงบ่อยตาม environment

อย่าสร้าง Module เมื่อ:
❌ ใช้แค่ครั้งเดียว
❌ ซับซ้อนเกินจำเป็น (over-engineering)
❌ Module มีแค่ resource เดียว
❌ ทำให้ harder to understand แทนที่จะ easier
```

### ระดับของ Module

```
Level 1: Basic Resources (ไม่ต้องเป็น module)
- single aws_s3_bucket
- single aws_security_group

Level 2: Resource Groups (ควรเป็น module)
- VPC + subnets + route tables + IGW + NAT
- ECS cluster + task definition + service
- RDS + subnet group + parameter group

Level 3: Application Pattern (module ระดับสูง)
- Complete web application stack
- Microservice pattern
- Data pipeline
```

---

## Step 138: Module Interface Design

### หลักการออกแบบ Module Interface

```hcl
# ✅ ดี: Interface ที่ชัดเจน มี defaults ที่เหมาะสม
variable "name" {
  description = "Name for all resources (required)"
  type        = string
}

variable "vpc_id" {
  description = "VPC ID (required)"
  type        = string
}

variable "instance_type" {
  description = "EC2 instance type"
  type        = string
  default     = "t3.micro"  # reasonable default
}

variable "instance_count" {
  description = "Number of instances"
  type        = number
  default     = 1  # reasonable default
}

variable "tags" {
  description = "Additional tags"
  type        = map(string)
  default     = {}  # empty is valid default
}
```

```hcl
# ❌ ไม่ดี: Interface ที่ expose internals ที่ไม่จำเป็น
variable "aws_instance_resource_name" {  # expose internal naming
  type = string
}

variable "sg_ingress_rule_1_from_port" {  # too granular
  type = number
}

variable "sg_ingress_rule_1_to_port" {  # should be part of object
  type = number
}
```

### Module Interface Best Practices

```hcl
# ✅ Group related variables เป็น object
variable "database" {
  description = "Database configuration"
  type = object({
    engine         = optional(string, "mysql")
    engine_version = optional(string, "8.0")
    instance_class = optional(string, "db.t3.micro")
    storage_gb     = optional(number, 20)
    username       = optional(string, "admin")
    multi_az       = optional(bool, false)
  })
  default = {}
}

# ✅ Provide enable/disable flags
variable "enable_cloudwatch_alarms" {
  description = "Whether to create CloudWatch alarms"
  type        = bool
  default     = false  # disabled by default, opt-in
}

variable "enable_enhanced_monitoring" {
  description = "Enable enhanced monitoring (interval in seconds, 0 to disable)"
  type        = number
  default     = 0
}
```

---

## Step 139: First Complete Module - VPC Module

### สมบูรณ์ VPC Module ตัวอย่าง

```hcl
# modules/vpc/main.tf

resource "aws_vpc" "this" {
  cidr_block           = var.cidr
  enable_dns_support   = true
  enable_dns_hostnames = true

  tags = merge(var.tags, { Name = var.name })
}

resource "aws_internet_gateway" "this" {
  count  = var.enable_internet_gateway ? 1 : 0
  vpc_id = aws_vpc.this.id
  tags   = merge(var.tags, { Name = "${var.name}-igw" })
}

resource "aws_subnet" "public" {
  count = length(var.public_subnets)

  vpc_id                  = aws_vpc.this.id
  cidr_block              = var.public_subnets[count.index]
  availability_zone       = var.azs[count.index]
  map_public_ip_on_launch = true

  tags = merge(var.tags, {
    Name = "${var.name}-public-${count.index + 1}"
    Tier = "Public"
  })
}

resource "aws_subnet" "private" {
  count = length(var.private_subnets)

  vpc_id            = aws_vpc.this.id
  cidr_block        = var.private_subnets[count.index]
  availability_zone = var.azs[count.index]

  tags = merge(var.tags, {
    Name = "${var.name}-private-${count.index + 1}"
    Tier = "Private"
  })
}

resource "aws_eip" "nat" {
  count  = var.enable_nat_gateway ? (var.single_nat_gateway ? 1 : length(var.azs)) : 0
  domain = "vpc"
  tags   = merge(var.tags, { Name = "${var.name}-nat-eip-${count.index + 1}" })
}

resource "aws_nat_gateway" "this" {
  count = var.enable_nat_gateway ? (var.single_nat_gateway ? 1 : length(var.azs)) : 0

  allocation_id = aws_eip.nat[count.index].id
  subnet_id     = aws_subnet.public[count.index].id

  tags = merge(var.tags, { Name = "${var.name}-nat-${count.index + 1}" })

  depends_on = [aws_internet_gateway.this]
}

resource "aws_route_table" "public" {
  vpc_id = aws_vpc.this.id
  tags   = merge(var.tags, { Name = "${var.name}-public-rt" })
}

resource "aws_route" "public_internet" {
  count = var.enable_internet_gateway ? 1 : 0

  route_table_id         = aws_route_table.public.id
  destination_cidr_block = "0.0.0.0/0"
  gateway_id             = aws_internet_gateway.this[0].id
}

resource "aws_route_table_association" "public" {
  count = length(aws_subnet.public)

  subnet_id      = aws_subnet.public[count.index].id
  route_table_id = aws_route_table.public.id
}

resource "aws_route_table" "private" {
  count  = var.enable_nat_gateway ? length(var.azs) : 1
  vpc_id = aws_vpc.this.id
  tags   = merge(var.tags, { Name = "${var.name}-private-rt-${count.index + 1}" })
}

resource "aws_route" "private_nat" {
  count = var.enable_nat_gateway ? length(var.azs) : 0

  route_table_id         = aws_route_table.private[count.index].id
  destination_cidr_block = "0.0.0.0/0"
  nat_gateway_id = var.single_nat_gateway ? aws_nat_gateway.this[0].id : aws_nat_gateway.this[count.index].id
}

resource "aws_route_table_association" "private" {
  count = length(aws_subnet.private)

  subnet_id      = aws_subnet.private[count.index].id
  route_table_id = var.enable_nat_gateway ? aws_route_table.private[
    var.single_nat_gateway ? 0 : count.index
  ].id : aws_route_table.private[0].id
}
```

```hcl
# modules/vpc/variables.tf

variable "name" {
  description = "Name prefix for all VPC resources"
  type        = string
  nullable    = false
}

variable "cidr" {
  description = "CIDR block for the VPC"
  type        = string
  default     = "10.0.0.0/16"
}

variable "azs" {
  description = "Availability zones to use"
  type        = list(string)
  
  validation {
    condition     = length(var.azs) >= 2
    error_message = "At least 2 AZs required."
  }
}

variable "public_subnets" {
  description = "CIDR blocks for public subnets"
  type        = list(string)
  default     = []
}

variable "private_subnets" {
  description = "CIDR blocks for private subnets"
  type        = list(string)
  default     = []
}

variable "enable_internet_gateway" {
  description = "Create an Internet Gateway"
  type        = bool
  default     = true
}

variable "enable_nat_gateway" {
  description = "Create NAT Gateway(s) for private subnets"
  type        = bool
  default     = false
}

variable "single_nat_gateway" {
  description = "Use a single NAT Gateway (cost saving) vs one per AZ"
  type        = bool
  default     = false
}

variable "tags" {
  description = "Tags to apply to all resources"
  type        = map(string)
  default     = {}
}
```

```hcl
# modules/vpc/outputs.tf

output "vpc_id" {
  description = "ID of the VPC"
  value       = aws_vpc.this.id
}

output "vpc_cidr" {
  description = "CIDR block of the VPC"
  value       = aws_vpc.this.cidr_block
}

output "public_subnet_ids" {
  description = "IDs of public subnets"
  value       = aws_subnet.public[*].id
}

output "private_subnet_ids" {
  description = "IDs of private subnets"
  value       = aws_subnet.private[*].id
}

output "internet_gateway_id" {
  description = "ID of the Internet Gateway"
  value       = var.enable_internet_gateway ? aws_internet_gateway.this[0].id : null
}

output "nat_gateway_ids" {
  description = "IDs of NAT Gateways"
  value       = aws_nat_gateway.this[*].id
}

output "nat_public_ips" {
  description = "Public IPs of NAT Gateways"
  value       = aws_eip.nat[*].public_ip
}
```

---

## Step 140: Using the Module

### การใช้ VPC Module ที่สร้าง

```hcl
# main.tf - Root module

module "vpc" {
  source = "./modules/vpc"

  name = "${var.project_name}-${var.environment}"
  cidr = var.vpc_cidr
  azs  = var.availability_zones

  public_subnets  = [for i, az in var.availability_zones : cidrsubnet(var.vpc_cidr, 8, i)]
  private_subnets = [for i, az in var.availability_zones : cidrsubnet(var.vpc_cidr, 8, i + 10)]

  enable_internet_gateway = true
  enable_nat_gateway      = var.environment == "prod" ? true : false
  single_nat_gateway      = var.environment != "prod"

  tags = {
    Project     = var.project_name
    Environment = var.environment
    ManagedBy   = "terraform"
  }
}

# ใช้ community module จาก registry
module "vpc_registry" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "~> 5.0"

  name = "${var.project_name}-${var.environment}"
  cidr = var.vpc_cidr

  azs             = var.availability_zones
  public_subnets  = var.public_subnet_cidrs
  private_subnets = var.private_subnet_cidrs

  enable_nat_gateway     = true
  single_nat_gateway     = var.environment != "prod"
  enable_dns_hostnames   = true
  enable_dns_support     = true

  tags = local.common_tags
}

# outputs.tf - Root module outputs
output "vpc_id" {
  description = "VPC ID"
  value       = module.vpc.vpc_id
}

output "private_subnet_ids" {
  description = "Private subnet IDs"
  value       = module.vpc.private_subnet_ids
}
```

---

## สรุป (Summary)

### Module Workflow

```
1. สร้าง module directory
   modules/vpc/
   ├── main.tf
   ├── variables.tf
   ├── outputs.tf
   └── versions.tf

2. เรียกใช้ใน root module
   module "vpc" {
     source = "./modules/vpc"
     name   = "my-vpc"
   }

3. terraform init (download module)
4. terraform plan
5. terraform apply
```

### Module Source Types สรุป

| Source | Format | Use Case |
|--------|--------|----------|
| Local | `./modules/vpc` | Internal modules |
| Git | `github.com/org/repo` | Team-shared modules |
| Registry | `hashicorp/vpc/aws` | Community modules |
| HTTP | `https://.../*.zip` | Artifact storage |

### ✅ Best Practices

1. **ไฟล์มาตรฐาน**: main.tf, variables.tf, outputs.tf, versions.tf
2. **Pin versions**: `version = "~> 5.0"` สำหรับ registry modules
3. **Document interface**: ทุก variable/output ต้องมี description
4. **Flat module structure**: อย่า nest modules ลึกเกินไป
5. **Test modules**: สร้าง examples/ directory

### ⚠️ Common Mistakes

```hcl
# ❌ ไม่ pin version ของ registry module
module "vpc" {
  source = "terraform-aws-modules/vpc/aws"
  # ขาด version!
}

# ✅ ถูกต้อง
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "~> 5.0"  # pin version
}

# ❌ Expose internal resource names
output "aws_vpc_resource_name" {  # implementation detail
  value = "main"
}

# ✅ ถูกต้อง - expose logical value
output "vpc_id" {
  value = aws_vpc.main.id
}
```

### 💡 Pro Tips

1. ใช้ `terraform-docs` สร้าง README.md อัตโนมัติ
2. `terraform get` อัพเดท module versions
3. `.terraform/modules/` directory เก็บ downloaded modules
4. ใช้ `//` ใน Git source เพื่อ reference subdirectory

---

*จบ Part 014 - HCL Modules Overview*
