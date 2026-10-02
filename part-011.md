# Part 011: HCL Input Variables (ตัวแปร Input)
## Steps 101-110: การใช้งาน Input Variables ใน Terraform

---

## บทนำ (Introduction)

Input Variables คือกลไกหลักในการทำให้ Terraform configurations มีความยืดหยุ่น (flexible) และนำกลับมาใช้ใหม่ได้ (reusable) แทนที่จะ hardcode ค่าต่างๆ ลงใน configuration โดยตรง เราสามารถกำหนดให้เป็น variable เพื่อให้ผู้ใช้สามารถกำหนดค่าได้เองในภายหลัง

Variables ทำให้ modules ของเราเป็น "interface" ที่ชัดเจน สิ่งที่ต้องการ input ก็คือ variables และสิ่งที่ส่งออกคือ outputs

---

## Step 101: Variable Block Syntax พื้นฐาน

### โครงสร้างพื้นฐานของ variable block

```hcl
# รูปแบบพื้นฐาน
variable "variable_name" {
  description = "คำอธิบาย variable นี้"
  type        = string
  default     = "default_value"
}
```

### ตัวอย่างการประกาศ variable แบบง่าย

```hcl
# variables.tf

# String variable
variable "environment" {
  description = "Environment name (dev, staging, prod)"
  type        = string
  default     = "dev"
}

# Number variable
variable "instance_count" {
  description = "Number of EC2 instances to create"
  type        = number
  default     = 1
}

# Boolean variable
variable "enable_monitoring" {
  description = "Whether to enable detailed monitoring"
  type        = bool
  default     = false
}
```

### การใช้งาน variable ใน resource

```hcl
# main.tf

resource "aws_instance" "web" {
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t2.micro"

  # อ้างอิง variable ด้วย var.variable_name
  monitoring = var.enable_monitoring
  
  tags = {
    Environment = var.environment
    Name        = "web-${var.environment}"
  }
}
```

---

## Step 102: ประเภทของ Variable Types

Terraform รองรับ type system ที่หลากหลาย แบ่งออกเป็น primitive types และ complex types

### 2.1 Primitive Types

#### String Type

```hcl
# ประเภท string - ข้อความธรรมดา
variable "region" {
  description = "AWS region to deploy resources"
  type        = string
  default     = "ap-southeast-1"
}

variable "instance_type" {
  description = "EC2 instance type"
  type        = string
  default     = "t3.micro"
}

variable "ami_id" {
  description = "AMI ID for EC2 instance"
  type        = string
  # ไม่มี default - ต้องระบุค่าทุกครั้ง
}

variable "key_name" {
  description = "SSH key pair name"
  type        = string
  default     = null  # optional - ไม่จำเป็นต้องระบุ
}
```

#### Number Type

```hcl
# ประเภท number - ตัวเลข (integer หรือ decimal)
variable "instance_count" {
  description = "Number of instances to launch"
  type        = number
  default     = 2
}

variable "disk_size_gb" {
  description = "Root volume size in GB"
  type        = number
  default     = 20
}

variable "cpu_threshold" {
  description = "CPU utilization threshold for auto scaling"
  type        = number
  default     = 80.0  # decimal ได้
}

variable "port" {
  description = "Application port number"
  type        = number
  default     = 8080
}
```

#### Bool Type

```hcl
# ประเภท bool - true/false
variable "enable_deletion_protection" {
  description = "Enable deletion protection for the database"
  type        = bool
  default     = true
}

variable "multi_az" {
  description = "Enable Multi-AZ deployment"
  type        = bool
  default     = false
}

variable "publicly_accessible" {
  description = "Make the database publicly accessible"
  type        = bool
  default     = false
}

variable "skip_final_snapshot" {
  description = "Skip final snapshot before deletion"
  type        = bool
  default     = false
}
```

### 2.2 Complex Types - List

```hcl
# ประเภท list - ordered collection
variable "availability_zones" {
  description = "List of availability zones"
  type        = list(string)
  default     = ["ap-southeast-1a", "ap-southeast-1b", "ap-southeast-1c"]
}

variable "allowed_ports" {
  description = "List of allowed ports"
  type        = list(number)
  default     = [80, 443, 8080]
}

variable "public_subnet_cidrs" {
  description = "CIDR blocks for public subnets"
  type        = list(string)
  default     = [
    "10.0.1.0/24",
    "10.0.2.0/24",
    "10.0.3.0/24"
  ]
}

variable "private_subnet_cidrs" {
  description = "CIDR blocks for private subnets"
  type        = list(string)
  default     = [
    "10.0.11.0/24",
    "10.0.12.0/24",
    "10.0.13.0/24"
  ]
}
```

#### การใช้งาน list variable

```hcl
# ใช้งาน list ใน resource
resource "aws_subnet" "public" {
  count             = length(var.public_subnet_cidrs)
  vpc_id            = aws_vpc.main.id
  cidr_block        = var.public_subnet_cidrs[count.index]
  availability_zone = var.availability_zones[count.index]

  tags = {
    Name = "public-subnet-${count.index + 1}"
  }
}
```

### 2.3 Complex Types - Map

```hcl
# ประเภท map - key-value pairs (all values same type)
variable "tags" {
  description = "Common tags to apply to all resources"
  type        = map(string)
  default     = {
    Project     = "my-project"
    ManagedBy   = "terraform"
    Environment = "dev"
  }
}

variable "instance_types_by_env" {
  description = "Instance type mapping by environment"
  type        = map(string)
  default     = {
    dev     = "t3.micro"
    staging = "t3.small"
    prod    = "t3.medium"
  }
}

variable "asg_capacity" {
  description = "Auto Scaling Group capacity settings"
  type        = map(number)
  default     = {
    min     = 1
    max     = 10
    desired = 2
  }
}
```

#### การใช้งาน map variable

```hcl
resource "aws_instance" "app" {
  ami           = "ami-0c55b159cbfafe1f0"
  # ใช้ map lookup
  instance_type = var.instance_types_by_env[var.environment]

  # merge tags
  tags = merge(var.tags, {
    Name = "app-server"
  })
}
```

### 2.4 Complex Types - Set

```hcl
# ประเภท set - unordered unique collection
variable "security_group_ids" {
  description = "Set of security group IDs"
  type        = set(string)
  default     = []
}

variable "enabled_regions" {
  description = "Set of AWS regions to enable"
  type        = set(string)
  default     = ["us-east-1", "us-west-2", "ap-southeast-1"]
}

variable "admin_users" {
  description = "Set of admin usernames (unique)"
  type        = set(string)
  default     = ["admin1", "admin2", "admin3"]
}
```

#### ความแตกต่างระหว่าง list และ set

```hcl
# list - ordered, ใช้ index ได้ แต่อาจมีค่าซ้ำ
variable "subnet_ids_list" {
  type    = list(string)
  default = ["subnet-1234", "subnet-5678"]
}

# set - unordered, ไม่มี index แต่ค่าไม่ซ้ำ
variable "subnet_ids_set" {
  type    = set(string)
  default = ["subnet-1234", "subnet-5678"]
}

# ใช้ set ใน for_each
resource "aws_route_table_association" "private" {
  for_each = var.subnet_ids_set
  
  subnet_id      = each.value
  route_table_id = aws_route_table.private.id
}
```

### 2.5 Complex Types - Object

```hcl
# ประเภท object - structured type with named attributes
variable "database_config" {
  description = "Database configuration settings"
  type = object({
    engine         = string
    engine_version = string
    instance_class = string
    storage_gb     = number
    multi_az       = bool
    username       = string
  })
  default = {
    engine         = "mysql"
    engine_version = "8.0"
    instance_class = "db.t3.micro"
    storage_gb     = 20
    multi_az       = false
    username       = "admin"
  }
}

variable "vpc_config" {
  description = "VPC configuration"
  type = object({
    cidr_block           = string
    enable_dns_support   = bool
    enable_dns_hostnames = bool
    tags                 = map(string)
  })
  default = {
    cidr_block           = "10.0.0.0/16"
    enable_dns_support   = true
    enable_dns_hostnames = true
    tags                 = {}
  }
}

# Object with optional fields (Terraform 1.3+)
variable "s3_bucket_config" {
  description = "S3 bucket configuration"
  type = object({
    bucket_name    = string
    force_destroy  = optional(bool, false)
    versioning     = optional(bool, true)
    encryption     = optional(string, "AES256")
    acl            = optional(string, "private")
  })
}
```

#### การใช้งาน object variable

```hcl
resource "aws_db_instance" "main" {
  engine            = var.database_config.engine
  engine_version    = var.database_config.engine_version
  instance_class    = var.database_config.instance_class
  allocated_storage = var.database_config.storage_gb
  multi_az          = var.database_config.multi_az
  username          = var.database_config.username
  password          = var.db_password  # separate sensitive variable
}
```

### 2.6 Complex Types - Tuple

```hcl
# ประเภท tuple - ordered collection with different types
variable "server_config" {
  description = "Server configuration: [name, count, type]"
  type        = tuple([string, number, string])
  default     = ["web-server", 3, "t3.micro"]
}

variable "health_check" {
  description = "Health check config: [path, interval, timeout]"
  type        = tuple([string, number, number])
  default     = ["/health", 30, 5]
}
```

---

## Step 103: Default Values

### การกำหนด default value

```hcl
# มี default - optional variable
variable "environment" {
  type    = string
  default = "dev"
}

# ไม่มี default - required variable
variable "vpc_id" {
  type        = string
  description = "ID of the VPC where resources will be created"
}

# default เป็น null - optional แต่ไม่มีค่าเริ่มต้น
variable "kms_key_id" {
  type    = string
  default = null
}

# complex default values
variable "common_tags" {
  type = map(string)
  default = {
    ManagedBy  = "terraform"
    Repository = "github.com/org/repo"
  }
}

variable "ingress_rules" {
  type = list(object({
    port     = number
    protocol = string
    cidr     = string
  }))
  default = [
    {
      port     = 80
      protocol = "tcp"
      cidr     = "0.0.0.0/0"
    },
    {
      port     = 443
      protocol = "tcp"
      cidr     = "0.0.0.0/0"
    }
  ]
}
```

### ใช้ conditional expression กับ default

```hcl
locals {
  # ใช้ local แทน default ที่ต้องการ dynamic value
  effective_instance_type = var.instance_type != null ? var.instance_type : (
    var.environment == "prod" ? "t3.medium" : "t3.micro"
  )
}
```

---

## Step 104: Description Attribute

### ทำไมต้อง description สำคัญ

description เป็น documentation ที่สำคัญมาก เพราะ:
1. แสดงใน `terraform plan` และ `terraform apply`
2. ใช้ใน `terraform-docs` สร้าง documentation อัตโนมัติ
3. ช่วยให้ผู้ใช้ module เข้าใจว่าต้องใส่ค่าอะไร

```hcl
# ❌ ไม่ดี - ไม่มี description
variable "db_pass" {
  type = string
}

# ✅ ดี - มี description ชัดเจน
variable "database_password" {
  description = "Password for the master database user. Must be at least 8 characters and contain uppercase, lowercase, numbers and special characters."
  type        = string
  sensitive   = true
}

# ตัวอย่าง description ที่ดี
variable "vpc_cidr" {
  description = "CIDR block for the VPC. Must be a valid IPv4 CIDR notation (e.g., 10.0.0.0/16). Cannot overlap with existing VPCs in the region."
  type        = string
  default     = "10.0.0.0/16"
}

variable "enable_nat_gateway" {
  description = "Whether to create NAT Gateway for private subnets. When true, creates one NAT Gateway per AZ (high_availability_nat_gateway=true) or a single NAT Gateway (false). Incurs hourly costs."
  type        = bool
  default     = true
}

variable "instance_type" {
  description = <<-EOT
    EC2 instance type for the application servers.
    
    Recommended values by environment:
    - dev:     t3.micro (0.5 vCPU, 1 GB RAM)
    - staging: t3.small (1 vCPU, 2 GB RAM)
    - prod:    t3.medium or larger (2 vCPU, 4 GB RAM)
    
    Full list: https://aws.amazon.com/ec2/instance-types/
  EOT
  type        = string
  default     = "t3.micro"
}
```

---

## Step 105: Validation Blocks

### โครงสร้าง validation block

```hcl
variable "variable_name" {
  type = string
  
  validation {
    condition     = <expression that evaluates to bool>
    error_message = "Error message shown when condition is false."
  }
}
```

### ตัวอย่าง validation พื้นฐาน

```hcl
# String length validation
variable "project_name" {
  description = "Name of the project"
  type        = string

  validation {
    condition     = length(var.project_name) >= 3 && length(var.project_name) <= 20
    error_message = "Project name must be between 3 and 20 characters."
  }
}

# Allowed values validation
variable "environment" {
  description = "Deployment environment"
  type        = string
  default     = "dev"

  validation {
    condition     = contains(["dev", "staging", "prod"], var.environment)
    error_message = "Environment must be one of: dev, staging, prod."
  }
}

# Instance type validation
variable "instance_type" {
  description = "EC2 instance type"
  type        = string

  validation {
    condition = contains([
      "t2.micro", "t2.small", "t2.medium",
      "t3.micro", "t3.small", "t3.medium", "t3.large",
      "m5.large", "m5.xlarge", "m5.2xlarge"
    ], var.instance_type)
    error_message = "Instance type must be one of the approved types."
  }
}
```

### Regex Validation

```hcl
# CIDR block validation ด้วย regex
variable "vpc_cidr" {
  description = "CIDR block for VPC"
  type        = string
  default     = "10.0.0.0/16"

  validation {
    condition = can(regex(
      "^([0-9]{1,3}\\.){3}[0-9]{1,3}/([0-9]|[1-2][0-9]|3[0-2])$",
      var.vpc_cidr
    ))
    error_message = "VPC CIDR must be a valid IPv4 CIDR block (e.g., 10.0.0.0/16)."
  }
}

# Email format validation
variable "alert_email" {
  description = "Email address for alerts"
  type        = string

  validation {
    condition = can(regex(
      "^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\\.[a-zA-Z]{2,}$",
      var.alert_email
    ))
    error_message = "Alert email must be a valid email address."
  }
}

# Resource name with allowed characters
variable "bucket_name" {
  description = "S3 bucket name"
  type        = string

  validation {
    condition = can(regex("^[a-z0-9][a-z0-9-]*[a-z0-9]$", var.bucket_name)) && length(var.bucket_name) >= 3 && length(var.bucket_name) <= 63
    error_message = "Bucket name must be 3-63 characters, start/end with lowercase letter or number, and contain only lowercase letters, numbers, or hyphens."
  }
}

# AWS Account ID validation (12 digits)
variable "aws_account_id" {
  description = "AWS Account ID"
  type        = string

  validation {
    condition     = can(regex("^[0-9]{12}$", var.aws_account_id))
    error_message = "AWS Account ID must be exactly 12 digits."
  }
}
```

### Advanced Validation - Multiple Conditions

```hcl
# Multiple validation blocks
variable "database_port" {
  description = "Database port number"
  type        = number
  default     = 5432

  validation {
    condition     = var.database_port > 0
    error_message = "Database port must be greater than 0."
  }

  validation {
    condition     = var.database_port <= 65535
    error_message = "Database port must be less than or equal to 65535."
  }

  validation {
    condition     = !contains([22, 25, 110, 143], var.database_port)
    error_message = "Database port must not use reserved ports (22, 25, 110, 143)."
  }
}

# CIDR with prefix length constraint
variable "subnet_cidr" {
  description = "Subnet CIDR block"
  type        = string

  validation {
    condition = can(cidrhost(var.subnet_cidr, 0))
    error_message = "Subnet CIDR must be a valid CIDR block."
  }

  validation {
    condition = tonumber(split("/", var.subnet_cidr)[1]) >= 16 && tonumber(split("/", var.subnet_cidr)[1]) <= 28
    error_message = "Subnet CIDR prefix length must be between /16 and /28."
  }
}
```

### Validation กับ Complex Types

```hcl
# List validation
variable "availability_zones" {
  description = "List of availability zones"
  type        = list(string)

  validation {
    condition     = length(var.availability_zones) >= 2
    error_message = "At least 2 availability zones must be specified for high availability."
  }

  validation {
    condition     = length(var.availability_zones) <= 3
    error_message = "Maximum 3 availability zones are supported."
  }
}

# Map validation
variable "tags" {
  description = "Resource tags"
  type        = map(string)

  validation {
    condition     = contains(keys(var.tags), "Environment")
    error_message = "Tags must include 'Environment' key."
  }

  validation {
    condition     = contains(keys(var.tags), "Project")
    error_message = "Tags must include 'Project' key."
  }
}
```

---

## Step 106: Sensitive Variables

### sensitive = true

```hcl
# ตัวแปรที่มี sensitive = true จะไม่แสดงค่าใน terraform plan/apply
variable "database_password" {
  description = "Master password for the RDS instance"
  type        = string
  sensitive   = true
}

variable "api_key" {
  description = "API key for third-party service"
  type        = string
  sensitive   = true
}

variable "private_key_pem" {
  description = "PEM-encoded private key"
  type        = string
  sensitive   = true
}

variable "aws_secret_access_key" {
  description = "AWS Secret Access Key"
  type        = string
  sensitive   = true
  # ⚠️ ไม่ควรใส่ credentials ใน variables - ใช้ environment variables แทน
}
```

### Sensitive output จาก sensitive variable

```hcl
# ถ้า output ใช้ sensitive variable จะต้อง mark เป็น sensitive ด้วย
output "db_connection_string" {
  description = "Database connection string"
  value       = "mysql://${var.db_username}:${var.database_password}@${aws_db_instance.main.endpoint}/${var.db_name}"
  sensitive   = true  # จำเป็นต้องใส่เพราะใช้ sensitive variable
}
```

### การจัดการ sensitive variables อย่างปลอดภัย

```hcl
# ✅ วิธีที่แนะนำ: ใช้ environment variables
# export TF_VAR_database_password="your-secure-password"

# ✅ ใช้ AWS Secrets Manager
data "aws_secretsmanager_secret_version" "db_password" {
  secret_id = "prod/myapp/database"
}

locals {
  db_password = jsondecode(data.aws_secretsmanager_secret_version.db_password.secret_string)["password"]
}

# ✅ ใช้ HashiCorp Vault
data "vault_generic_secret" "db" {
  path = "secret/myapp/db"
}

resource "aws_db_instance" "main" {
  password = data.vault_generic_secret.db.data["password"]
}
```

---

## Step 107: Nullable Variables

### nullable = false

```hcl
# โดย default variables สามารถ null ได้
# nullable = false ทำให้ variable ต้องมีค่าเสมอ (ไม่สามารถเป็น null ได้)

variable "project_name" {
  description = "Name of the project"
  type        = string
  nullable    = false  # ต้องมีค่าเสมอ ห้ามเป็น null
}

variable "environment" {
  description = "Deployment environment"
  type        = string
  default     = "dev"
  nullable    = false  # แม้มี default แต่ก็ห้ามส่ง null เข้ามา
}

# เมื่อ nullable = false และมี default
# - ถ้าไม่ระบุ: ใช้ default
# - ถ้าระบุ null: error
# - ถ้าระบุค่า: ใช้ค่านั้น

variable "subnet_count" {
  description = "Number of subnets to create"
  type        = number
  default     = 3
  nullable    = false

  validation {
    condition     = var.subnet_count >= 1 && var.subnet_count <= 6
    error_message = "Subnet count must be between 1 and 6."
  }
}
```

---

## Step 108: Variable Precedence Order

### ลำดับความสำคัญของ Variable Values (น้อยไปมาก)

```
1. default value (ต่ำสุด)
2. terraform.tfvars
3. terraform.tfvars.json
4. *.auto.tfvars / *.auto.tfvars.json
5. -var-file flag
6. TF_VAR_* environment variables
7. -var flag (สูงสุด)
```

### ตัวอย่าง Precedence

```hcl
# variables.tf
variable "environment" {
  type    = string
  default = "dev"  # ค่า default
}
```

```bash
# terraform.tfvars
environment = "staging"  # override default

# แต่ถ้าใช้ -var flag:
terraform apply -var="environment=prod"  # override ทุกอย่าง
```

### ตัวอย่างสถานการณ์จริง

```bash
# 1. ค่า default จาก variable declaration
# environment = "dev"

# 2. Override ด้วย terraform.tfvars
# terraform.tfvars:
# environment = "staging"

# 3. Override ด้วย environment variable
export TF_VAR_environment=prod
# environment = "prod"

# 4. Override ด้วย -var flag (ใช้ได้ทุกวิธี)
terraform apply -var="environment=custom"
# environment = "custom"

# 5. ใช้หลาย -var flags
terraform apply \
  -var="environment=prod" \
  -var="region=ap-southeast-1" \
  -var="instance_type=t3.medium"
```

---

## Step 109: tfvars Files

### terraform.tfvars - ไฟล์หลัก

```hcl
# terraform.tfvars (โหลดอัตโนมัติ)

# Primitive types
environment   = "production"
region        = "ap-southeast-1"
instance_type = "t3.medium"
instance_count = 3
enable_monitoring = true

# String multi-line
description = <<-EOT
  Production environment
  Managed by Terraform
EOT

# List
availability_zones = ["ap-southeast-1a", "ap-southeast-1b", "ap-southeast-1c"]
allowed_ips = ["1.2.3.4/32", "5.6.7.8/32"]

# Map
tags = {
  Environment = "prod"
  Project     = "myapp"
  ManagedBy   = "terraform"
  CostCenter  = "12345"
}

# Object
database_config = {
  engine         = "mysql"
  engine_version = "8.0"
  instance_class = "db.t3.small"
  storage_gb     = 100
  multi_az       = true
  username       = "admin"
}
```

### *.auto.tfvars - โหลดอัตโนมัติ

```hcl
# prod.auto.tfvars (โหลดอัตโนมัติตาม alphabetical order)

environment = "prod"
instance_type = "t3.large"
min_capacity = 3
max_capacity = 20
```

### -var-file flag - ระบุไฟล์เอง

```bash
# ใช้ไฟล์ tfvars ที่กำหนดเอง
terraform apply -var-file="environments/prod.tfvars"
terraform apply -var-file="environments/prod.tfvars" -var-file="secrets/prod.tfvars"

# ตัวอย่างโครงสร้างไฟล์
# environments/
# ├── dev.tfvars
# ├── staging.tfvars
# └── prod.tfvars
```

```hcl
# environments/dev.tfvars
environment    = "dev"
instance_type  = "t3.micro"
instance_count = 1
enable_monitoring = false

database_config = {
  engine         = "mysql"
  engine_version = "8.0"
  instance_class = "db.t3.micro"
  storage_gb     = 20
  multi_az       = false
  username       = "admin"
}
```

```hcl
# environments/prod.tfvars
environment    = "prod"
instance_type  = "t3.large"
instance_count = 5
enable_monitoring = true

database_config = {
  engine         = "mysql"
  engine_version = "8.0"
  instance_class = "db.r5.large"
  storage_gb     = 500
  multi_az       = true
  username       = "admin"
}
```

### tfvars.json format

```json
// terraform.tfvars.json
{
  "environment": "prod",
  "instance_type": "t3.medium",
  "tags": {
    "Environment": "prod",
    "Project": "myapp"
  },
  "availability_zones": ["ap-southeast-1a", "ap-southeast-1b"],
  "enable_monitoring": true
}
```

---

## Step 110: TF_VAR_* Environment Variables

### การใช้งาน Environment Variables

```bash
# รูปแบบ: TF_VAR_<variable_name>=<value>

# String variables
export TF_VAR_environment=prod
export TF_VAR_region=ap-southeast-1
export TF_VAR_instance_type=t3.medium

# Number variables
export TF_VAR_instance_count=3

# Boolean variables
export TF_VAR_enable_monitoring=true

# Secret variables (วิธีที่แนะนำสำหรับ secrets)
export TF_VAR_database_password="super-secret-password"
export TF_VAR_api_key="my-api-key-12345"

# จากนั้น run terraform
terraform plan
terraform apply
```

### Complex Types ใน Environment Variables

```bash
# List (ต้องใช้ format พิเศษ)
export TF_VAR_availability_zones='["ap-southeast-1a","ap-southeast-1b","ap-southeast-1c"]'

# Map
export TF_VAR_tags='{"Environment":"prod","Project":"myapp"}'

# Object
export TF_VAR_database_config='{"engine":"mysql","engine_version":"8.0","instance_class":"db.t3.micro","storage_gb":20,"multi_az":false,"username":"admin"}'
```

### ใน CI/CD Pipeline

```yaml
# GitHub Actions ตัวอย่าง
name: Terraform Deploy

on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    env:
      TF_VAR_environment: prod
      TF_VAR_region: ap-southeast-1
      TF_VAR_database_password: ${{ secrets.DB_PASSWORD }}
      TF_VAR_api_key: ${{ secrets.API_KEY }}

    steps:
      - uses: actions/checkout@v3
      - uses: hashicorp/setup-terraform@v2
      - run: terraform init
      - run: terraform plan
      - run: terraform apply -auto-approve
```

---

## Complex Validation Examples

### CIDR Validation แบบละเอียด

```hcl
variable "vpc_cidr" {
  description = "VPC CIDR block"
  type        = string
  default     = "10.0.0.0/16"

  validation {
    # ตรวจสอบว่าเป็น valid CIDR
    condition = can(cidrhost(var.vpc_cidr, 0))
    error_message = "VPC CIDR must be a valid CIDR block."
  }

  validation {
    # ตรวจสอบ prefix length
    condition = tonumber(split("/", var.vpc_cidr)[1]) >= 8 && tonumber(split("/", var.vpc_cidr)[1]) <= 28
    error_message = "VPC CIDR prefix must be between /8 and /28."
  }

  validation {
    # ตรวจสอบว่าอยู่ใน private IP ranges (RFC 1918)
    condition = (
      can(regex("^10\\.", var.vpc_cidr)) ||
      can(regex("^172\\.(1[6-9]|2[0-9]|3[01])\\.", var.vpc_cidr)) ||
      can(regex("^192\\.168\\.", var.vpc_cidr))
    )
    error_message = "VPC CIDR must be in private IP range: 10.0.0.0/8, 172.16.0.0/12, or 192.168.0.0/16."
  }
}
```

### Password Validation

```hcl
variable "master_password" {
  description = "Master password (min 8 chars, must have upper, lower, number, special)"
  type        = string
  sensitive   = true

  validation {
    condition     = length(var.master_password) >= 8
    error_message = "Password must be at least 8 characters long."
  }

  validation {
    condition     = can(regex("[A-Z]", var.master_password))
    error_message = "Password must contain at least one uppercase letter."
  }

  validation {
    condition     = can(regex("[a-z]", var.master_password))
    error_message = "Password must contain at least one lowercase letter."
  }

  validation {
    condition     = can(regex("[0-9]", var.master_password))
    error_message = "Password must contain at least one number."
  }

  validation {
    condition     = can(regex("[!@#$%^&*()_+]", var.master_password))
    error_message = "Password must contain at least one special character (!@#$%^&*()_+)."
  }
}
```

### AWS ARN Validation

```hcl
variable "iam_role_arn" {
  description = "IAM Role ARN to assume"
  type        = string

  validation {
    condition = can(regex(
      "^arn:aws:iam::[0-9]{12}:role/[a-zA-Z0-9+=,.@_/-]+$",
      var.iam_role_arn
    ))
    error_message = "IAM Role ARN must be a valid AWS ARN in format: arn:aws:iam::ACCOUNT_ID:role/ROLE_NAME."
  }
}

variable "kms_key_arn" {
  description = "KMS Key ARN for encryption"
  type        = string
  default     = null

  validation {
    condition = var.kms_key_arn == null || can(regex(
      "^arn:aws:kms:[a-z0-9-]+:[0-9]{12}:key/[a-f0-9-]{36}$",
      var.kms_key_arn
    ))
    error_message = "KMS Key ARN must be a valid AWS KMS key ARN or null."
  }
}
```

---

## Variable Dependencies และ Cross-References

```hcl
# variables.tf - ตัวแปรที่มีความเกี่ยวข้องกัน
variable "environment" {
  type    = string
  default = "dev"

  validation {
    condition     = contains(["dev", "staging", "prod"], var.environment)
    error_message = "Environment must be dev, staging, or prod."
  }
}

variable "region" {
  type    = string
  default = "ap-southeast-1"
}

variable "availability_zones" {
  type    = list(string)
  default = ["ap-southeast-1a", "ap-southeast-1b"]
}
```

```hcl
# locals.tf - ใช้ locals เพื่อ compute values จาก variables
locals {
  # ชื่อ prefix ที่ใช้ทั่วทั้ง infrastructure
  name_prefix = "${var.project_name}-${var.environment}-${var.region}"

  # Instance type ตาม environment
  effective_instance_type = lookup(
    {
      dev     = "t3.micro"
      staging = "t3.small"
      prod    = "t3.medium"
    },
    var.environment,
    "t3.micro"
  )

  # Tags รวม
  common_tags = merge(
    var.tags,
    {
      Environment = var.environment
      Region      = var.region
      ManagedBy   = "terraform"
    }
  )
}
```

---

## Complete Real-World Example

```hcl
# variables.tf สำหรับ VPC Module

variable "project_name" {
  description = "Name of the project. Used as prefix for all resource names."
  type        = string
  nullable    = false

  validation {
    condition     = can(regex("^[a-z][a-z0-9-]{2,29}$", var.project_name))
    error_message = "Project name must start with lowercase letter, contain only lowercase letters, numbers, hyphens, and be 3-30 characters."
  }
}

variable "environment" {
  description = "Deployment environment (dev/staging/prod)"
  type        = string
  nullable    = false

  validation {
    condition     = contains(["dev", "staging", "prod"], var.environment)
    error_message = "Environment must be one of: dev, staging, prod."
  }
}

variable "vpc_cidr" {
  description = "CIDR block for the VPC (must be in private IP range)"
  type        = string
  default     = "10.0.0.0/16"

  validation {
    condition = can(cidrhost(var.vpc_cidr, 0))
    error_message = "VPC CIDR must be a valid CIDR block."
  }
}

variable "availability_zones" {
  description = "List of availability zones to use (minimum 2 for HA)"
  type        = list(string)

  validation {
    condition     = length(var.availability_zones) >= 2
    error_message = "At least 2 availability zones required for high availability."
  }

  validation {
    condition     = length(var.availability_zones) <= 3
    error_message = "Maximum 3 availability zones supported."
  }
}

variable "enable_nat_gateway" {
  description = "Create NAT Gateway for private subnets (incurs costs)"
  type        = bool
  default     = true
}

variable "single_nat_gateway" {
  description = "Use single NAT Gateway (cost saving) vs one per AZ (high availability)"
  type        = bool
  default     = false
}

variable "tags" {
  description = "Additional tags to apply to all resources"
  type        = map(string)
  default     = {}
}
```

---

## สรุป (Summary)

### Key Takeaways

| Feature | Description | Example |
|---------|-------------|---------|
| `type` | กำหนด type ของ variable | `type = string` |
| `default` | ค่าเริ่มต้น (optional variable) | `default = "dev"` |
| `description` | คำอธิบาย (documentation) | `description = "..."` |
| `sensitive` | ซ่อนค่าใน output | `sensitive = true` |
| `nullable` | ห้าม null | `nullable = false` |
| `validation` | ตรวจสอบค่า | `validation { condition = ... }` |

### Variable Precedence (ต่ำไปสูง)

```
default value
↓
terraform.tfvars
↓
*.auto.tfvars  
↓
-var-file flag
↓
TF_VAR_* environment variables
↓
-var flag (สูงสุด)
```

### ✅ Best Practices

1. **ใส่ description เสมอ** - ช่วย documentation
2. **ใช้ validation** - จับ error ก่อน apply
3. **sensitive = true สำหรับ secrets** - ป้องกันข้อมูลรั่ว
4. **ใช้ TF_VAR_* ใน CI/CD** - อย่า hardcode secrets ใน .tfvars
5. **nullable = false** สำหรับ required fields ที่ห้าม null

### ⚠️ Common Mistakes

```hcl
# ❌ Hardcode secrets ใน .tfvars
database_password = "MySecretPassword123"

# ✅ ใช้ environment variable แทน
# export TF_VAR_database_password="MySecretPassword123"

# ❌ ไม่มี validation บน critical variables
variable "vpc_cidr" {
  type = string
}

# ✅ เพิ่ม validation
variable "vpc_cidr" {
  type = string
  validation {
    condition     = can(cidrhost(var.vpc_cidr, 0))
    error_message = "Must be valid CIDR block."
  }
}
```

### 💡 Pro Tips

1. สร้างไฟล์ `variables.tf` แยกต่างหากจาก `main.tf` เสมอ
2. ใช้ `terraform.tfvars.example` เป็น template แล้ว gitignore `terraform.tfvars`
3. Group variables ที่เกี่ยวข้องกันเป็น object type
4. ใช้ `can()` function ใน validation condition เพื่อ catch invalid formats
5. เขียน `description` ราวกับว่าเป็น documentation สำหรับคนอื่น

---

*จบ Part 011 - HCL Input Variables*
