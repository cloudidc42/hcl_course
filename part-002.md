# Part 002: HCL Syntax โครงสร้างพื้นฐาน
## HCL Syntax and Structure (Steps 11-20)

---

## Step 11: HCL File Naming Conventions

### Convention มาตรฐาน

```
Standard Terraform File Naming:
├── main.tf          → Main configuration (resources หลัก)
├── variables.tf     → Input variable declarations
├── outputs.tf       → Output value declarations
├── locals.tf        → Local value declarations
├── versions.tf      → Terraform and provider version constraints
├── providers.tf     → Provider configurations
├── data.tf          → Data source declarations
├── backend.tf       → Backend configuration (บางครั้งอยู่ใน versions.tf)
└── terraform.tfvars → Default variable values
```

### ตัวอย่างการแบ่งไฟล์สำหรับโปรเจกต์ใหญ่

```
large-project/
├── main.tf              # Entry point - module calls
├── versions.tf          # Required versions
├── variables.tf         # All input variables
├── outputs.tf           # All output values
├── locals.tf            # Computed local values
├── networking.tf        # VPC, subnets, routing
├── compute.tf           # EC2, ECS, Lambda
├── database.tf          # RDS, DynamoDB
├── storage.tf           # S3, EFS
├── security.tf          # IAM, Security Groups, KMS
└── monitoring.tf        # CloudWatch, SNS
```

### Naming Rules

```hcl
# ✅ ถูกต้อง - ใช้ lowercase และ underscore
resource "aws_instance" "web_server" { }
resource "aws_s3_bucket" "static_assets" { }
variable "instance_type" { }
locals {
  common_tags = {}
}

# ❌ ผิด - ห้ามใช้ uppercase, dash, หรือ เริ่มด้วยตัวเลข
resource "aws_instance" "WebServer" { }      # uppercase
resource "aws_instance" "web-server" { }     # dash ใน identifier
resource "aws_instance" "1st_server" { }     # เริ่มด้วยตัวเลข
```

---

## Step 12: Block Syntax

### โครงสร้าง Block พื้นฐาน

```
ASCII Diagram - Block Structure:
┌────────────────────────────────────────────┐
│  block_type "label1" "label2" {            │
│  │          │        │        │            │
│  │          │        │        └── Body {}  │
│  │          │        └──────────── Label 2 │
│  │          └───────────────────── Label 1 │
│  └──────────────────────────────── Type    │
│    argument_name = argument_value          │
│                                            │
│    nested_block {                          │
│      nested_argument = value               │
│    }                                       │
│  }                                         │
└────────────────────────────────────────────┘
```

### ประเภทของ Blocks

```hcl
# 1. terraform block - ไม่มี labels
terraform {
  required_version = ">= 1.5.0"
}

# 2. provider block - มี 1 label (provider name)
provider "aws" {
  region = "ap-southeast-1"
}

# 3. resource block - มี 2 labels (type, name)
resource "aws_instance" "web" {
  ami           = "ami-12345"
  instance_type = "t3.micro"
}

# 4. data block - มี 2 labels (type, name)
data "aws_ami" "ubuntu" {
  most_recent = true
}

# 5. variable block - มี 1 label (variable name)
variable "region" {
  type    = string
  default = "ap-southeast-1"
}

# 6. output block - มี 1 label (output name)
output "instance_id" {
  value = aws_instance.web.id
}

# 7. locals block - ไม่มี labels
locals {
  env = "production"
}

# 8. module block - มี 1 label (module name)
module "vpc" {
  source = "./modules/vpc"
}
```

### Nested Blocks

```hcl
# Nested blocks - blocks ภายใน blocks
resource "aws_security_group" "web" {
  name        = "web-sg"
  description = "Web server security group"
  vpc_id      = aws_vpc.main.id

  # ingress เป็น nested block
  ingress {
    description = "HTTPS from internet"
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  # สามารถมีหลาย ingress blocks
  ingress {
    description = "HTTP for redirect"
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = {
    Name = "web-sg"
  }
}
```

### Block ที่ซ้อนกันหลายชั้น

```hcl
resource "aws_ecs_task_definition" "app" {
  family                   = "app"
  network_mode             = "awsvpc"
  requires_compatibilities = ["FARGATE"]
  cpu                      = 256
  memory                   = 512
  
  # Level 1: container_definitions (เป็น JSON string ไม่ใช่ block)
  container_definitions = jsonencode([
    {
      name  = "app"
      image = "nginx:latest"
      portMappings = [
        {
          containerPort = 80
          hostPort      = 80
          protocol      = "tcp"
        }
      ]
    }
  ])
  
  # Level 1: volume block
  volume {
    name = "app-storage"
    
    # Level 2: efs_volume_configuration nested block
    efs_volume_configuration {
      file_system_id = aws_efs_file_system.app.id
      root_directory = "/app"
      
      # Level 3: authorization_config nested block
      authorization_config {
        access_point_id = aws_efs_access_point.app.id
        iam             = "ENABLED"
      }
    }
  }
}
```

---

## Step 13: Argument Syntax

### พื้นฐาน Arguments

```
ASCII Diagram - Argument Syntax:
┌─────────────────────────────────────────┐
│  argument_name = expression             │
│  │              │                       │
│  │              └── Value (any expr)    │
│  └─────────────────── Identifier        │
└─────────────────────────────────────────┘
```

### ประเภทของ Argument Values

```hcl
resource "example" "demo" {
  # String literal
  name = "my-resource"
  
  # Number literal  
  port = 8080
  
  # Boolean literal
  enabled = true
  
  # Null value
  optional_param = null
  
  # List/Tuple
  availability_zones = ["us-east-1a", "us-east-1b", "us-east-1c"]
  
  # Map/Object
  tags = {
    Name        = "my-resource"
    Environment = "production"
  }
  
  # Reference to another resource
  vpc_id = aws_vpc.main.id
  
  # Reference to variable
  instance_type = var.instance_type
  
  # Reference to local value
  common_name = local.resource_name
  
  # Expression
  full_name = "${var.project}-${var.environment}-resource"
  
  # Function call
  upper_name = upper(var.project)
  
  # Conditional expression
  size = var.env == "prod" ? "large" : "small"
}
```

### Argument Alignment (Formatting)

```hcl
# ✅ ถูกต้อง - ใช้ terraform fmt จะ align ให้อัตโนมัติ
resource "aws_instance" "web" {
  ami           = "ami-12345"
  instance_type = "t3.micro"
  subnet_id     = aws_subnet.public.id
  
  tags = {
    Name        = "web"
    Environment = "production"
    ManagedBy   = "terraform"
  }
}

# ❌ ไม่ดี - ไม่ align (ทำงานได้แต่อ่านยาก)
resource "aws_instance" "web" {
  ami = "ami-12345"
  instance_type = "t3.micro"
  subnet_id = aws_subnet.public.id
  tags = {
    Name = "web"
    Environment = "production"
    ManagedBy = "terraform"
  }
}
```

---

## Step 14: Identifiers และ Naming Rules

### Identifier Rules

```
Valid Identifier Characters:
├── Letters (a-z, A-Z)
├── Digits (0-9) - ไม่ได้ใช้เป็นตัวแรก
├── Underscore (_)
└── Dash (-) - ใช้ได้ใน some contexts

Invalid:
├── Cannot start with a digit
├── Cannot contain spaces
├── Cannot contain special chars (@, #, $, etc.)
└── Cannot be a reserved keyword
```

### ตัวอย่าง Valid Identifiers

```hcl
# ✅ Valid identifiers
variable "instance_type" { }
variable "vpc_cidr_block" { }
variable "enable_dns" { }
variable "max_size_99" { }
variable "_private_var" { }  # underscore prefix allowed

locals {
  region_name = "ap-southeast-1"
  is_prod     = true
  count_99    = 99
}

resource "aws_instance" "web_server_v2" { }
resource "aws_s3_bucket" "static_assets_2024" { }
```

### Reserved Keywords ใน HCL

```hcl
# คำเหล่านี้เป็น reserved keywords
# ห้ามใช้เป็น identifier ของตัวเอง

# Block type keywords:
# terraform, provider, resource, data, variable, output, locals, module

# Expression keywords:
# true, false, null

# ถ้าจำเป็นต้องใช้ชื่อที่ชนกับ keyword ใช้ quote
variable "true_value" { }    # ✅ OK ถ้าใช้ใน context ที่ถูกต้อง

# In maps, quoted keys allow any string
locals {
  config = {
    "true"  = "yes"    # ✅ quoted key
    "false" = "no"     # ✅ quoted key
    "null"  = "none"   # ✅ quoted key
  }
}
```

### Naming Conventions Best Practices

```hcl
# ตัวอย่าง naming convention สำหรับโปรเจกต์จริง

# Variables: lowercase_with_underscores
variable "aws_region" { }
variable "instance_type" { }
variable "db_password" { }

# Resources: descriptive_name ที่บอกว่าเป็นอะไร
resource "aws_vpc" "main" { }           # หรือ "primary"
resource "aws_subnet" "public_1a" { }   # บอก AZ
resource "aws_instance" "web_app" { }   # บอก role
resource "aws_rds_cluster" "postgres" { }

# Data sources: ใช้ชื่อบอกว่าจะเอาไปทำอะไร
data "aws_ami" "amazon_linux_2" { }
data "aws_vpc" "existing" { }
data "aws_iam_policy" "admin" { }

# Outputs: ชื่อชัดเจนว่า output อะไร
output "vpc_id" { }
output "public_subnet_ids" { }
output "web_server_public_ip" { }
```

---

## Step 15: String Literals

### Quoted Strings

```hcl
# Simple quoted strings
variable "name" {
  default = "John Doe"
}

resource "aws_s3_bucket" "example" {
  # String literals
  bucket = "my-company-assets-2024"
  
  # String interpolation
  bucket = "my-company-${var.environment}-assets"
  
  # Multiple interpolations
  bucket = "${var.company}-${var.project}-${var.environment}"
}
```

### String Escape Sequences

```hcl
locals {
  # Tab character
  tabbed   = "Column1\tColumn2\tColumn3"
  
  # Newline
  newlined = "Line1\nLine2\nLine3"
  
  # Double quote inside string
  quoted   = "She said \"Hello World\""
  
  # Backslash
  path     = "C:\\Users\\admin\\documents"
  
  # Dollar sign (to avoid interpolation)
  literal_dollar = "Price: $${price}"  # ${price} ไม่ถูก interpolate
  
  # Percent sign (in templatefile)
  literal_percent = "100%%"  # %% ใน template = literal %
}
```

### Heredoc Strings

```hcl
# Basic heredoc - <<EOF
resource "aws_iam_policy" "example" {
  name   = "example-policy"
  policy = <<EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": "s3:GetObject",
      "Resource": "arn:aws:s3:::my-bucket/*"
    }
  ]
}
EOF
}

# Indented heredoc - <<-EOF (removes leading whitespace)
resource "aws_instance" "web" {
  ami           = "ami-12345"
  instance_type = "t3.micro"
  
  user_data = <<-EOT
    #!/bin/bash
    yum update -y
    yum install -y httpd
    systemctl start httpd
    systemctl enable httpd
    echo "<h1>Hello from Terraform!</h1>" > /var/www/html/index.html
  EOT
}
```

### String Interpolation

```hcl
locals {
  project     = "myapp"
  environment = "production"
  region      = "ap-southeast-1"
  
  # Basic interpolation
  bucket_name = "${local.project}-${local.environment}"
  
  # Function in interpolation
  upper_name  = "${upper(local.project)}-config"
  
  # Conditional in interpolation
  env_prefix  = "${local.environment == "production" ? "prod" : "dev"}"
  
  # Complex expression
  full_name   = "${local.project}-${local.environment}-${local.region}"
  
  # Template literal (เมื่อมีแค่ interpolation เดียว ไม่ต้องใส่ "")
  # ✅ ดีกว่า: ไม่ต้องใช้ interpolation ถ้ามีแค่ reference เดียว
  bucket_arn = "arn:aws:s3:::${local.bucket_name}"
}
```

---

## Step 16: Number Literals

### Integer และ Float

```hcl
locals {
  # Integer literals
  port         = 8080
  max_size     = 10
  timeout      = 300
  year         = 2024
  
  # Float literals
  cpu_units    = 0.5
  memory_gb    = 2.5
  threshold    = 99.9
  
  # Large numbers (no separator ใน HCL)
  million      = 1000000
  bytes        = 1073741824  # 1 GB in bytes
  
  # Negative numbers
  temperature  = -10
  offset       = -5.5
  
  # Scientific notation (HCL ไม่ support แบบนี้โดยตรง)
  # ใช้ function แทน
  large_num    = pow(10, 6)  # 1,000,000
}
```

### ตัวเลขใน Resource Arguments

```hcl
resource "aws_autoscaling_group" "web" {
  name                = "web-asg"
  max_size            = 10        # integer
  min_size            = 2         # integer
  desired_capacity    = 3         # integer
  health_check_grace_period = 300 # seconds (integer)
  
  tag {
    key                 = "Name"
    value               = "web-server"
    propagate_at_launch = true  # boolean (ไม่ใช่ number)
  }
}

resource "aws_cloudwatch_metric_alarm" "cpu" {
  alarm_name          = "high-cpu"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = 2          # integer
  metric_name         = "CPUUtilization"
  namespace           = "AWS/EC2"
  period              = 120         # seconds (integer)
  statistic           = "Average"
  threshold           = 80          # integer (80%)
  
  # float ใน some cases
  # datapoints_to_alarm = 1
}
```

---

## Step 17: Boolean Literals

### true/false Values

```hcl
resource "aws_vpc" "main" {
  cidr_block = "10.0.0.0/16"
  
  # Boolean arguments
  enable_dns_hostnames = true   # เปิด DNS hostnames
  enable_dns_support   = true   # เปิด DNS support
  
  # Boolean ที่เป็น false (default)
  assign_generated_ipv6_cidr_block = false
}

resource "aws_s3_bucket_versioning" "example" {
  bucket = aws_s3_bucket.example.id
  
  versioning_configuration {
    status = "Enabled"  # บางที boolean เก็บเป็น string "Enabled"/"Disabled"
  }
}

variable "enable_monitoring" {
  description = "เปิดใช้ monitoring หรือไม่"
  type        = bool
  default     = false
}

variable "create_nat_gateway" {
  description = "สร้าง NAT Gateway หรือไม่"
  type        = bool
  default     = true
}
```

### Boolean ใน Conditional Expressions

```hcl
# ใช้ boolean variable สำหรับ conditional resource creation
resource "aws_nat_gateway" "main" {
  # สร้าง NAT Gateway เฉพาะเมื่อ create_nat_gateway = true
  count = var.create_nat_gateway ? 1 : 0
  
  allocation_id = aws_eip.nat[0].id
  subnet_id     = aws_subnet.public[0].id
}

locals {
  # Boolean operations
  is_production = var.environment == "production"
  needs_backup  = local.is_production || var.force_backup
  skip_tests    = !var.run_tests
  
  # Boolean ใน conditional
  instance_type = local.is_production ? "t3.large" : "t3.micro"
}
```

---

## Step 18: Null Values

### การใช้ Null

```hcl
# Null ใน variable default
variable "db_snapshot_id" {
  description = "Snapshot ID สำหรับ restore (null = สร้างใหม่)"
  type        = string
  default     = null  # Optional variable
}

# Null ใน conditional
resource "aws_db_instance" "main" {
  # ถ้า snapshot_id เป็น null จะสร้าง DB ใหม่
  # ถ้าไม่ใช่ null จะ restore จาก snapshot
  snapshot_identifier = var.db_snapshot_id
  
  engine         = "mysql"
  instance_class = "db.t3.micro"
  
  # Final snapshot: null หมายความว่าจะไม่ทำ final snapshot
  final_snapshot_identifier = var.environment == "production" ? "prod-final-snapshot" : null
  skip_final_snapshot       = var.environment != "production"
}

# Null coalescing pattern
locals {
  # ใช้ coalesce() เพื่อ handle null
  effective_region = coalesce(var.override_region, var.default_region)
  
  # ตรวจสอบ null ด้วย try()
  optional_config = try(var.optional_config.setting, "default_value")
}
```

### Null vs Empty String vs Not Set

```hcl
variable "tag_value" {
  type    = string
  default = null  # ไม่ได้ตั้งค่า
}

variable "empty_tag" {
  type    = string
  default = ""  # empty string (ต่างจาก null)
}

locals {
  # ตรวจสอบว่าเป็น null หรือ empty
  has_value    = var.tag_value != null && var.tag_value != ""
  
  # Conditional สำหรับ optional tags
  final_tags = var.tag_value != null ? {
    CustomTag = var.tag_value
  } : {}
}
```

---

## Step 19: Reference Syntax

### วิธีการอ้างถึง Resources

```
Reference Syntax:
┌────────────────────────────────────────────────────┐
│  resource_type.resource_name.attribute_name        │
│  │              │             │                    │
│  │              │             └── Attribute        │
│  │              └───────────── Resource Name (ID)  │
│  └──────────────────────────── Resource Type       │
│                                                    │
│  Examples:                                         │
│  aws_instance.web.id                               │
│  aws_vpc.main.cidr_block                           │
│  aws_subnet.public.availability_zone               │
└────────────────────────────────────────────────────┘
```

### ประเภทของ References

```hcl
# 1. Resource reference
resource "aws_subnet" "public" {
  vpc_id = aws_vpc.main.id  # อ้างถึง aws_vpc.main.id
  cidr_block = "10.0.1.0/24"
}

# 2. Variable reference
resource "aws_instance" "web" {
  instance_type = var.instance_type  # อ้างถึง variable
}

# 3. Local value reference
resource "aws_instance" "web" {
  tags = local.common_tags  # อ้างถึง local value
}

# 4. Data source reference
resource "aws_instance" "web" {
  ami = data.aws_ami.ubuntu.id  # อ้างถึง data source
}

# 5. Module output reference
resource "aws_instance" "web" {
  subnet_id = module.vpc.public_subnet_ids[0]  # อ้างถึง module output
}

# 6. Self reference (ใน provisioner)
resource "aws_instance" "web" {
  ami           = "ami-12345"
  instance_type = "t3.micro"
  
  provisioner "local-exec" {
    # self อ้างถึง resource นี้เอง
    command = "echo Instance IP: ${self.public_ip}"
  }
}

# 7. Count index reference
resource "aws_subnet" "private" {
  count      = 3
  vpc_id     = aws_vpc.main.id
  cidr_block = "10.0.${count.index + 1}.0/24"  # count.index = 0,1,2
}

# 8. For_each key reference
resource "aws_subnet" "az" {
  for_each   = var.availability_zones
  vpc_id     = aws_vpc.main.id
  cidr_block = each.value.cidr  # each.key = az name, each.value = az config
}
```

### Dependency และ Reference

```hcl
# Explicit dependency ผ่าน reference
resource "aws_vpc" "main" {
  cidr_block = "10.0.0.0/16"
}

resource "aws_internet_gateway" "main" {
  vpc_id = aws_vpc.main.id  # Explicit dependency: IGW ต้องสร้างหลัง VPC
}

resource "aws_route_table" "public" {
  vpc_id = aws_vpc.main.id

  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.main.id  # Dependency chain
  }
}

# Implicit dependency ผ่าน depends_on
resource "aws_iam_role_policy_attachment" "lambda" {
  role       = aws_iam_role.lambda.name
  policy_arn = aws_iam_policy.lambda.arn
  
  # ใช้ depends_on เมื่อ Terraform ไม่รู้ dependency จาก references
  depends_on = [
    aws_iam_role.lambda,
    aws_iam_policy.lambda
  ]
}
```

---

## Step 20: Terraform, Required_providers, Provider Blocks

### terraform Block โดยละเอียด

```hcl
# versions.tf

terraform {
  # 1. Terraform version constraint
  required_version = ">= 1.5.0, < 2.0.0"
  
  # 2. Required providers
  required_providers {
    aws = {
      source  = "hashicorp/aws"    # registry.terraform.io/hashicorp/aws
      version = "~> 5.0"           # >= 5.0.0, < 6.0.0
    }
    
    kubernetes = {
      source  = "hashicorp/kubernetes"
      version = ">= 2.0, < 3.0"
    }
    
    random = {
      source  = "hashicorp/random"
      version = "~> 3.5"
    }
    
    # Third-party provider
    datadog = {
      source  = "DataDog/datadog"
      version = "~> 3.0"
    }
  }
  
  # 3. Backend configuration
  backend "s3" {
    bucket         = "my-terraform-state"
    key            = "prod/terraform.tfstate"
    region         = "ap-southeast-1"
    encrypt        = true
    dynamodb_table = "terraform-lock"
  }
  
  # 4. Cloud block (alternative to backend for HCP Terraform)
  # cloud {
  #   organization = "my-org"
  #   workspaces {
  #     name = "production"
  #   }
  # }
}
```

### Version Constraint Operators

```
Version Constraint Operators:
┌────────────────────────────────────────────────────┐
│  = 1.5.0      → Exactly version 1.5.0             │
│  != 1.5.0     → Any except 1.5.0                  │
│  > 1.5.0      → Greater than 1.5.0                │
│  >= 1.5.0     → Greater than or equal to 1.5.0    │
│  < 2.0.0      → Less than 2.0.0                   │
│  <= 2.0.0     → Less than or equal to 2.0.0       │
│  ~> 1.5.0     → >= 1.5.0, < 1.6.0 (patch only)   │
│  ~> 1.5       → >= 1.5, < 2.0 (minor + patch)     │
└────────────────────────────────────────────────────┘
```

### Provider Block

```hcl
# providers.tf

# Basic provider configuration
provider "aws" {
  region = "ap-southeast-1"
}

# Provider with authentication
provider "aws" {
  region     = "ap-southeast-1"
  access_key = var.aws_access_key  # ไม่แนะนำ hardcode
  secret_key = var.aws_secret_key  # ใช้ environment variables แทน
}

# Provider with profile (recommended)
provider "aws" {
  region  = "ap-southeast-1"
  profile = "my-company-profile"
}

# Provider with default_tags (best practice)
provider "aws" {
  region = "ap-southeast-1"
  
  default_tags {
    tags = {
      ManagedBy   = "terraform"
      Project     = var.project
      Environment = var.environment
      Owner       = var.team
    }
  }
}

# Multiple provider configurations (aliased)
provider "aws" {
  alias  = "us_east"
  region = "us-east-1"
}

provider "aws" {
  alias  = "ap_southeast"
  region = "ap-southeast-1"
}

# ใช้ aliased provider
resource "aws_instance" "us_web" {
  provider      = aws.us_east
  ami           = "ami-us-east-123"
  instance_type = "t3.micro"
}

resource "aws_instance" "ap_web" {
  provider      = aws.ap_southeast
  ami           = "ami-ap-sea-123"
  instance_type = "t3.micro"
}
```

### ตัวอย่างสมบูรณ์: Multi-environment Setup

```hcl
# versions.tf - ใช้กับทุก environment
terraform {
  required_version = ">= 1.5.0"
  
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
  
  backend "s3" {
    # ค่าเหล่านี้ set ผ่าน -backend-config หรือ environment variables
    # bucket = ""
    # key    = ""
    # region = ""
  }
}

# providers.tf
provider "aws" {
  region = var.aws_region
  
  assume_role {
    role_arn     = "arn:aws:iam::${var.account_id}:role/TerraformDeployRole"
    session_name = "terraform-${var.environment}"
  }
  
  default_tags {
    tags = {
      ManagedBy   = "terraform"
      Environment = var.environment
      Project     = var.project_name
    }
  }
}

# variables.tf
variable "aws_region" {
  description = "AWS Region"
  type        = string
  default     = "ap-southeast-1"
}

variable "environment" {
  description = "Environment name"
  type        = string
  
  validation {
    condition     = contains(["dev", "staging", "prod"], var.environment)
    error_message = "Environment must be dev, staging, or prod."
  }
}

variable "project_name" {
  description = "Project name"
  type        = string
}

variable "account_id" {
  description = "AWS Account ID"
  type        = string
}
```

---

## Common Syntax Errors และวิธีแก้

### Error 1: Missing required argument

```hcl
# ❌ ผิด
resource "aws_instance" "web" {
  instance_type = "t3.micro"
  # ลืม ami - required argument
}

# Error: Missing required argument
# The argument "ami" is required, but no definition was found.

# ✅ ถูก
resource "aws_instance" "web" {
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t3.micro"
}
```

### Error 2: Invalid reference

```hcl
# ❌ ผิด - อ้างถึง resource ที่ไม่มี
resource "aws_subnet" "public" {
  vpc_id     = aws_vpc.nonexistent.id  # vpc "nonexistent" ไม่มี
  cidr_block = "10.0.1.0/24"
}

# Error: Reference to undeclared resource
# A managed resource "aws_vpc" "nonexistent" has not been declared

# ✅ ถูก - สร้าง vpc ก่อนแล้วอ้างถึง
resource "aws_vpc" "main" {
  cidr_block = "10.0.0.0/16"
}

resource "aws_subnet" "public" {
  vpc_id     = aws_vpc.main.id  # อ้างถึง main vpc
  cidr_block = "10.0.1.0/24"
}
```

### Error 3: Invalid block definition

```hcl
# ❌ ผิด - ขาด opening brace
resource "aws_instance" "web"
  ami           = "ami-12345"
  instance_type = "t3.micro"

# Error: An argument or block definition is required here.

# ✅ ถูก
resource "aws_instance" "web" {
  ami           = "ami-12345"
  instance_type = "t3.micro"
}
```

### Error 4: Type mismatch

```hcl
# ❌ ผิด - ใส่ string แต่ต้องการ number
resource "aws_autoscaling_group" "web" {
  max_size = "10"  # string แต่ต้องการ number
}

# Error: Incorrect attribute value type
# Expected number, got string.

# ✅ ถูก
resource "aws_autoscaling_group" "web" {
  max_size = 10  # number
}
```

---

## สรุป (Summary)

### สิ่งที่เรียนรู้ใน Part 002

| หัวข้อ | สาระสำคัญ |
|--------|-----------|
| File naming | ใช้ main.tf, variables.tf, outputs.tf เป็น convention |
| Block syntax | `type "label1" "label2" { }` |
| Arguments | `key = value` format |
| Identifiers | lowercase_with_underscores เป็น convention |
| String literals | Quoted, interpolated, heredoc |
| Numbers | Integer และ float |
| Booleans | true/false literal |
| Null | null literal สำหรับ optional values |
| References | `resource_type.name.attribute` |
| Provider blocks | Configure cloud providers |

💡 **Pro Tips:**
- รัน `terraform fmt` เสมอ - มัน align code ให้อัตโนมัติ
- รัน `terraform validate` ก่อน plan เพื่อ catch syntax errors
- ใช้ `<<-EOT` (indented heredoc) แทน `<<EOT` เพื่อให้ code ดูสะอาด
- ตั้งชื่อ resource ให้สื่อความหมาย อย่าใช้ "this" หรือ "default"

⚠️ **Common Mistakes:**
- ใช้ dash (-) ใน identifier (ใช้ underscore แทน)
- ลืม `=` ใน argument (ใช้ block syntax แทน)
- Quote ตัวเลข: `max_size = "10"` แทนที่จะเป็น `max_size = 10`
- ไม่ close block ด้วย `}`

---

*ก่อนหน้า: [Part 001 - Introduction to HCL & IaC](part-001.md)*
*ต่อไป: [Part 003 - HCL Data Types: Primitives](part-003.md)*
