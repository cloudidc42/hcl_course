# Part 001: Introduction to HCL & Infrastructure as Code
## บทนำสู่ HCL และ Infrastructure as Code (Steps 1-10)

---

## Step 1: Infrastructure as Code คืออะไร?

### ความหมายและความสำคัญ

**Infrastructure as Code (IaC)** คือแนวทางการจัดการโครงสร้างพื้นฐาน (infrastructure) ผ่านการเขียนโค้ด แทนที่จะต้องคลิกผ่าน GUI หรือทำด้วยมือ

```
Traditional Approach (แบบเก่า):
├── เปิด AWS Console
├── คลิก Create EC2
├── เลือก AMI manually
├── กำหนด instance type
├── สร้าง Security Group
└── ทำซ้ำทุกครั้งที่ต้องการ environment ใหม่

IaC Approach (แบบใหม่):
├── เขียนโค้ดครั้งเดียว
├── run terraform apply
└── ได้ infrastructure เหมือนกันทุกครั้ง
```

### ปัญหาที่ IaC แก้ไข

| ปัญหา | ไม่ใช้ IaC | ใช้ IaC |
|-------|-----------|---------|
| Reproducibility | ทำซ้ำยาก อาจต่างกัน | เหมือนกัน 100% ทุกครั้ง |
| Documentation | ต้องเขียนแยก มักล้าสมัย | โค้ดคือ documentation |
| Version Control | ไม่มี history | Git track ทุก change |
| Collaboration | ยากต่อการทำงานร่วมกัน | Review ผ่าน Pull Request |
| Disaster Recovery | ใช้เวลานาน | Deploy ใหม่ได้ภายในนาที |
| Cost Management | มักมี orphaned resources | Destroy ทุกอย่างได้ง่าย |
| Testing | ทดสอบยาก | Unit test, integration test ได้ |

### ตัวอย่างที่จับต้องได้

สมมติว่าต้องการสร้าง development environment เหมือน production:

```hcl
# ไม่ใช้ IaC: ต้องทำมือ 2-3 วัน
# ใช้ IaC: รันคำสั่งเดียว ได้ผลลัพธ์เหมือนกัน

# main.tf - Terraform configuration
resource "aws_vpc" "development" {
  cidr_block = "10.0.0.0/16"
  
  tags = {
    Name        = "development-vpc"
    Environment = "development"
    ManagedBy   = "terraform"
  }
}

resource "aws_subnet" "public" {
  vpc_id            = aws_vpc.development.id
  cidr_block        = "10.0.1.0/24"
  availability_zone = "ap-southeast-1a"
  
  tags = {
    Name = "public-subnet"
  }
}

resource "aws_instance" "web_server" {
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t3.micro"
  subnet_id     = aws_subnet.public.id
  
  tags = {
    Name = "web-server"
  }
}
```

---

## Step 2: HCL คืออะไร?

### HashiCorp Configuration Language

**HCL (HashiCorp Configuration Language)** คือภาษาที่ HashiCorp สร้างขึ้นสำหรับเขียน configuration files โดยเฉพาะ ออกแบบให้:

1. **Human-friendly** - อ่านและเขียนง่ายสำหรับมนุษย์
2. **Machine-friendly** - Parse ได้ง่ายสำหรับเครื่อง
3. **Expressive** - มี expressions, functions, conditionals
4. **Modular** - แยก code เป็น modules ได้

### HCL ถูกใช้ใน HashiCorp tools ต่างๆ

```
HashiCorp Ecosystem:
├── Terraform     → infrastructure provisioning
├── Vault         → secrets management  
├── Consul        → service discovery
├── Nomad         → workload orchestration
├── Packer        → machine image building
└── Waypoint      → application deployment
```

### โครงสร้าง HCL พื้นฐาน

```hcl
# Block - หน่วยพื้นฐานของ HCL
block_type "label_1" "label_2" {
  # Arguments - key = value pairs
  argument_name = "argument_value"
  number_arg    = 42
  bool_arg      = true
  
  # Nested block
  nested_block {
    nested_arg = "value"
  }
}
```

---

## Step 3: เปรียบเทียบ HCL vs JSON vs YAML

### Comparison Table

| Feature | HCL | JSON | YAML |
|---------|-----|------|------|
| Human Readability | ⭐⭐⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐⭐ |
| Machine Parseable | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐ |
| Comments | ✅ Yes | ❌ No | ✅ Yes |
| Functions/Expressions | ✅ Yes | ❌ No | ❌ No |
| Conditionals | ✅ Yes | ❌ No | ❌ No |
| Loops | ✅ Yes | ❌ No | ❌ No |
| Multi-line strings | ✅ Heredoc | ❌ Difficult | ✅ Yes |
| Type Safety | ✅ Strong | ⚠️ Weak | ⚠️ Weak |
| Trailing Commas | ✅ Allowed | ❌ Not Allowed | N/A |
| Schema Validation | ✅ Built-in | ❌ External | ❌ External |

### ตัวอย่างเปรียบเทียบ: Define an EC2 Instance

**JSON Format:**
```json
{
  "resource": {
    "aws_instance": {
      "web": {
        "ami": "ami-0c55b159cbfafe1f0",
        "instance_type": "t3.micro",
        "tags": {
          "Name": "web-server",
          "Environment": "production"
        }
      }
    }
  }
}
```

**YAML Format:**
```yaml
resource:
  aws_instance:
    web:
      ami: ami-0c55b159cbfafe1f0
      instance_type: t3.micro
      tags:
        Name: web-server
        Environment: production
```

**HCL Format:**
```hcl
resource "aws_instance" "web" {
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t3.micro"
  
  tags = {
    Name        = "web-server"
    Environment = "production"
  }
}
```

### ทำไม HCL ถึงดีกว่าสำหรับ Infrastructure?

```hcl
# HCL สามารถใช้ expressions ได้
resource "aws_instance" "web" {
  # ใช้ variable
  ami           = var.ami_id
  # ใช้ conditional expression
  instance_type = var.environment == "production" ? "t3.large" : "t3.micro"
  # ใช้ function
  tags = merge(var.common_tags, {
    Name = format("%s-web-%s", var.project, var.environment)
  })
}

# JSON/YAML ไม่สามารถทำสิ่งนี้ได้โดยตรง
```

---

## Step 4: ทำไม Terraform/HCL ถึงเป็น Industry Standard

### Market Adoption

```
IaC Tool Popularity (2024):
┌─────────────────────────────────────┐
│ Terraform/OpenTofu    ████████ 78%  │
│ AWS CloudFormation    █████    45%  │
│ Pulumi                ███      28%  │
│ Ansible (IaC portion) ████     35%  │
│ Chef/Puppet           ██       15%  │
└─────────────────────────────────────┘
Source: HashiCorp State of Cloud Strategy Survey
```

### ข้อดีของ Terraform

1. **Cloud-agnostic** - ใช้กับ AWS, Azure, GCP, และอื่นๆ ได้
2. **Provider ecosystem** - มี providers มากกว่า 3,000+ ตัว
3. **State management** - track สถานะปัจจุบันของ infrastructure
4. **Plan before apply** - preview changes ก่อน apply
5. **Module system** - แบ่ง code เป็น reusable modules

```hcl
# ตัวอย่าง multi-cloud: deploy ทั้ง AWS และ Azure ในไฟล์เดียว
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
    azurerm = {
      source  = "hashicorp/azurerm"
      version = "~> 3.0"
    }
  }
}

# AWS Resource
resource "aws_s3_bucket" "data" {
  bucket = "my-company-data-aws"
}

# Azure Resource  
resource "azurerm_storage_account" "data" {
  name                = "mycompanydataazure"
  resource_group_name = azurerm_resource_group.main.name
  location            = "Southeast Asia"
  account_tier        = "Standard"
  account_replication_type = "LRS"
}
```

### Provider Registry - ecosystem ที่กว้างมาก

```
registry.terraform.io มี providers สำหรับ:
├── Cloud Providers:    AWS, Azure, GCP, Oracle, IBM, Alibaba
├── SaaS:               GitHub, Datadog, PagerDuty, Cloudflare
├── Database:           PostgreSQL, MySQL, MongoDB Atlas
├── Kubernetes:         Kubernetes, Helm
├── Networking:         Cisco, F5, Palo Alto
└── Many more...
```

---

## Step 5: การตั้งค่า Development Environment

### สิ่งที่ต้องติดตั้ง

```
Required Tools:
├── Terraform CLI      → เครื่องมือหลัก
├── VS Code            → Text Editor
├── HashiCorp HCL ext  → Syntax highlighting + autocomplete
└── Git                → Version control

Optional but Recommended:
├── tfenv              → Manage multiple Terraform versions
├── tflint             → Linter for Terraform
├── terraform-docs     → Auto-generate documentation
└── pre-commit         → Git hooks for validation
```

### ติดตั้ง Terraform CLI

**macOS (Homebrew):**
```bash
# ติดตั้ง via Homebrew
brew tap hashicorp/tap
brew install hashicorp/tap/terraform

# ตรวจสอบการติดตั้ง
terraform version
# Output: Terraform v1.6.x
```

**Ubuntu/Debian:**
```bash
# Add HashiCorp GPG key
wget -O- https://apt.releases.hashicorp.com/gpg | sudo gpg --dearmor -o /usr/share/keyrings/hashicorp-archive-keyring.gpg

# Add repository
echo "deb [signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] https://apt.releases.hashicorp.com $(lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/hashicorp.list

# ติดตั้ง
sudo apt update && sudo apt install terraform

# ตรวจสอบ
terraform version
```

**Windows (Chocolatey):**
```powershell
# ติดตั้ง via Chocolatey
choco install terraform

# หรือ download binary จาก
# https://developer.hashicorp.com/terraform/downloads
```

**ใช้ tfenv (แนะนำ):**
```bash
# ติดตั้ง tfenv
git clone https://github.com/tfutils/tfenv.git ~/.tfenv
echo 'export PATH="$HOME/.tfenv/bin:$PATH"' >> ~/.bash_profile

# ติดตั้ง Terraform version ที่ต้องการ
tfenv install 1.6.0
tfenv use 1.6.0

# ตรวจสอบ
terraform version
```

### ตั้งค่า VS Code

```json
// settings.json for VS Code
{
  "editor.formatOnSave": true,
  "editor.defaultFormatter": "hashicorp.terraform",
  "[terraform]": {
    "editor.defaultFormatter": "hashicorp.terraform",
    "editor.formatOnSave": true,
    "editor.formatOnSaveMode": "file"
  },
  "[terraform-vars]": {
    "editor.defaultFormatter": "hashicorp.terraform",
    "editor.formatOnSave": true
  },
  "terraform.languageServer": {
    "enable": true
  }
}
```

Extensions ที่ควรติดตั้ง:
```
Extensions:
├── HashiCorp Terraform (hashicorp.terraform)
│   ├── Syntax highlighting
│   ├── Auto-complete
│   ├── Hover documentation  
│   └── Format on save
├── GitLens
├── Error Lens
└── indent-rainbow
```

---

## Step 6: HCL File แรก - Hello World Equivalent

### สร้าง Project Structure

```bash
# สร้าง directory
mkdir my-first-terraform && cd my-first-terraform

# โครงสร้างไฟล์
.
├── main.tf          # Main configuration
├── variables.tf     # Variable definitions
├── outputs.tf       # Output definitions
└── terraform.tfvars # Variable values (ไม่ commit ถ้า sensitive)
```

### main.tf - Configuration หลัก

```hcl
# main.tf

# บอก Terraform ว่าต้องการ version อะไร
terraform {
  required_version = ">= 1.0"
  
  required_providers {
    local = {
      source  = "hashicorp/local"
      version = "~> 2.4"
    }
  }
}

# ใช้ local provider (ไม่ต้องมี cloud credentials)
resource "local_file" "hello_world" {
  filename = "${path.module}/hello_world.txt"
  content  = "Hello, Terraform! This is my first HCL file.\n"
}

resource "local_file" "greeting" {
  filename = "${path.module}/greeting.txt"
  content  = <<-EOT
    Hello from HCL!
    ----------------
    Project: ${var.project_name}
    Environment: ${var.environment}
    Created at: ${timestamp()}
  EOT
}
```

### variables.tf - Variable Definitions

```hcl
# variables.tf

variable "project_name" {
  description = "ชื่อโปรเจกต์"
  type        = string
  default     = "my-first-project"
}

variable "environment" {
  description = "สภาพแวดล้อม (dev/staging/prod)"
  type        = string
  default     = "development"
  
  validation {
    condition     = contains(["development", "staging", "production"], var.environment)
    error_message = "Environment must be one of: development, staging, production."
  }
}
```

### outputs.tf - Output Definitions

```hcl
# outputs.tf

output "hello_file_path" {
  description = "Path ของไฟล์ hello world"
  value       = local_file.hello_world.filename
}

output "greeting_content" {
  description = "เนื้อหาของ greeting file"
  value       = local_file.greeting.content
}
```

### terraform.tfvars - Variable Values

```hcl
# terraform.tfvars
project_name = "hcl-learning"
environment  = "development"
```

---

## Step 7: Terraform Workflow - init/plan/apply/destroy

### The Four Core Commands

```
Terraform Workflow:
┌─────────────────────────────────────────┐
│                                         │
│  Code  →  init  →  plan  →  apply       │
│                              ↓          │
│                           Resources     │
│                           Created!      │
│                              ↓          │
│                           destroy       │
│                           (cleanup)     │
└─────────────────────────────────────────┘
```

### terraform init

```bash
# รัน init ก่อนเสมอ
terraform init

# Output:
# Initializing the backend...
# Initializing provider plugins...
# - Finding hashicorp/local versions matching "~> 2.4"...
# - Installing hashicorp/local v2.4.0...
# - Installed hashicorp/local v2.4.0 (signed by HashiCorp)
#
# Terraform has been successfully initialized!
```

init ทำอะไรบ้าง:
```
terraform init:
├── Download providers ที่ระบุใน required_providers
├── Initialize backend (ที่เก็บ state file)
├── สร้าง .terraform/ directory
└── สร้าง .terraform.lock.hcl (lock file สำหรับ provider versions)
```

### terraform plan

```bash
# Preview changes ก่อน apply
terraform plan

# Output:
# Terraform used the selected providers to generate the following execution plan.
# Resource actions are indicated with the following symbols:
#   + create
#
# Terraform will perform the following actions:
#
#   # local_file.greeting will be created
#   + resource "local_file" "greeting" {
#       + content              = <<-EOT
#             Hello from HCL!
#             ----------------
#             Project: hcl-learning
#             Environment: development
#         EOT
#       + filename             = "./greeting.txt"
#       + id                   = (known after apply)
#     }
#
# Plan: 2 to add, 0 to change, 0 to destroy.

# บันทึก plan ลงไฟล์
terraform plan -out=tfplan
```

### terraform apply

```bash
# Apply the changes
terraform apply

# หรือ apply โดยไม่ต้อง confirm (สำหรับ CI/CD)
terraform apply -auto-approve

# Apply จาก saved plan
terraform apply tfplan

# Output:
# local_file.hello_world: Creating...
# local_file.hello_world: Creation complete after 0s [id=abc123]
# local_file.greeting: Creating...
# local_file.greeting: Creation complete after 0s [id=def456]
#
# Apply complete! Resources: 2 added, 0 changed, 0 destroyed.
#
# Outputs:
# hello_file_path = "./hello_world.txt"
```

### terraform destroy

```bash
# ลบทรัพยากรทั้งหมด
terraform destroy

# Output:
# Terraform used the selected providers to generate the following execution plan.
# Resource actions are indicated with the following symbols:
#   - destroy
#
# Terraform will perform the following actions:
#   # local_file.greeting will be destroyed
#   - resource "local_file" "greeting" {
#       ...
#     }
#
# Plan: 0 to add, 0 to change, 2 to destroy.
#
# Do you really want to destroy all resources?
#   Enter a value: yes
#
# local_file.greeting: Destroying...
# local_file.hello_world: Destroying...
# Destroy complete! Resources: 2 destroyed.
```

### ตาราง Terraform Commands สรุป

| Command | Purpose | ใช้เมื่อ |
|---------|---------|----------|
| `terraform init` | Initialize workspace | ครั้งแรก หรือเมื่อเพิ่ม provider |
| `terraform validate` | Check syntax | ก่อน plan เสมอ |
| `terraform plan` | Preview changes | ก่อน apply เสมอ |
| `terraform apply` | Create/Update resources | พร้อม deploy |
| `terraform destroy` | Delete all resources | ต้องการ cleanup |
| `terraform show` | Show current state | ตรวจสอบ state |
| `terraform output` | Show outputs | อ่าน output values |
| `terraform fmt` | Format code | ก่อน commit |
| `terraform state list` | List resources in state | Debug |
| `terraform import` | Import existing resources | Adopt existing infra |

---

## Step 8: HCL File Structure Overview

### Terraform Project Structure

```
terraform-project/
├── main.tf              # Main resources
├── variables.tf         # Input variables
├── outputs.tf           # Output values
├── locals.tf            # Local values
├── versions.tf          # Terraform + provider versions
├── data.tf              # Data sources
├── terraform.tfvars     # Variable values (สำหรับ local)
├── terraform.tfvars.json # Variable values (JSON format)
├── .terraform/          # Terraform internals (ไม่ต้อง commit)
│   ├── providers/
│   └── modules/
├── .terraform.lock.hcl  # Provider version lock (ต้อง commit)
└── terraform.tfstate    # State file (ระวัง! มี sensitive data)
```

### ประเภทของ Blocks ใน Terraform

```hcl
# 1. terraform block - Terraform configuration
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
    key    = "global/s3/terraform.tfstate"
    region = "ap-southeast-1"
  }
}

# 2. provider block - Provider configuration
provider "aws" {
  region  = "ap-southeast-1"
  profile = "default"
  
  default_tags {
    tags = {
      ManagedBy = "terraform"
    }
  }
}

# 3. resource block - Infrastructure resources
resource "aws_instance" "web" {
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t3.micro"
}

# 4. data block - Read existing resources
data "aws_ami" "ubuntu" {
  most_recent = true
  owners      = ["099720109477"]
  
  filter {
    name   = "name"
    values = ["ubuntu/images/hvm-ssd/ubuntu-*-22.04-amd64-server-*"]
  }
}

# 5. variable block - Input variables
variable "instance_type" {
  description = "EC2 instance type"
  type        = string
  default     = "t3.micro"
}

# 6. output block - Output values
output "instance_ip" {
  value = aws_instance.web.public_ip
}

# 7. locals block - Local computed values
locals {
  common_tags = {
    Project     = var.project
    Environment = var.environment
    ManagedBy   = "terraform"
  }
}

# 8. module block - Reuse modules
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "~> 5.0"
  
  name = "my-vpc"
  cidr = "10.0.0.0/16"
}
```

---

## Step 9: HCL Comments

### สามแบบของ Comments

```hcl
# Single-line comment ด้วย hash
# นี่คือ comment แบบที่ 1
# ใช้บ่อยที่สุด ใน Terraform

// Single-line comment ด้วย double slash
// นี่คือ comment แบบที่ 2
// ใช้บ้างแต่ไม่ค่อยนิยมใน Terraform

/* Multi-line comment
   นี่คือ comment แบบที่ 3
   สำหรับ comment หลายบรรทัด
   ใช้สำหรับ documentation blocks */

# ตัวอย่างการใช้ comment อย่างมีประสิทธิภาพ
resource "aws_vpc" "main" {
  # CIDR block สำหรับ VPC ทั้งหมด
  # /16 ให้ IP ได้ถึง 65,534 addresses
  cidr_block = "10.0.0.0/16"
  
  # เปิด DNS support เพื่อให้ instances resolve hostnames ได้
  enable_dns_hostnames = true
  enable_dns_support   = true
  
  tags = {
    Name = "main-vpc"
    /* 
     * Tag นี้ใช้สำหรับ cost allocation
     * ดู confluence page: https://wiki.company.com/aws-tagging
     */
    CostCenter = "engineering"
  }
}
```

### Comment Best Practices

```hcl
# ✅ DO: Comment ว่า "ทำไม" ไม่ใช่ "ทำอะไร"
resource "aws_security_group" "web" {
  name = "web-sg"
  
  # อนุญาต HTTPS เท่านั้น ไม่อนุญาต HTTP เพราะ security policy
  ingress {
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }
}

# ❌ DON'T: Comment ที่อธิบายสิ่งที่เห็นได้อยู่แล้ว
resource "aws_security_group" "web" {
  name = "web-sg"  # กำหนด name เป็น "web-sg"
  
  # สร้าง ingress rule
  ingress {
    from_port   = 443  # port 443
    to_port     = 443  # port 443
    protocol    = "tcp"  # protocol tcp
    cidr_blocks = ["0.0.0.0/0"]  # ทุก IP
  }
}
```

### Commenting Out Code

```hcl
# Temporarily disable a resource
# resource "aws_instance" "old_server" {
#   ami           = "ami-old123"
#   instance_type = "t2.micro"
# }

# TODO: เปลี่ยนเป็น RDS หลัง migration เสร็จ
resource "aws_db_instance" "legacy" {
  # FIXME: ต้องอัพเดท parameter group
  engine         = "mysql"
  instance_class = "db.t3.micro"
}
```

---

## Step 10: Hands-on Lab - สร้าง Terraform Configuration แรก

### Lab Objective

สร้าง Terraform configuration ที่จัดการ local files โดยใช้ local provider (ไม่ต้องมี cloud account)

### Lab Instructions

**Step 10.1: สร้าง Project Directory**

```bash
# สร้าง directory สำหรับ lab
mkdir terraform-lab-01
cd terraform-lab-01

# สร้างไฟล์ทั้งหมดที่ต้องการ
touch main.tf variables.tf outputs.tf terraform.tfvars
```

**Step 10.2: เขียน versions.tf**

```hcl
# versions.tf
terraform {
  required_version = ">= 1.5.0"
  
  required_providers {
    local = {
      source  = "hashicorp/local"
      version = "~> 2.4"
    }
    random = {
      source  = "hashicorp/random"
      version = "~> 3.5"
    }
  }
}
```

**Step 10.3: เขียน variables.tf**

```hcl
# variables.tf

variable "student_name" {
  description = "ชื่อนักเรียน"
  type        = string
}

variable "course_name" {
  description = "ชื่อหลักสูตร"
  type        = string
  default     = "HCL Fundamentals"
}

variable "lesson_count" {
  description = "จำนวน lessons"
  type        = number
  default     = 10
}

variable "topics" {
  description = "รายการหัวข้อที่เรียน"
  type        = list(string)
  default = [
    "Introduction to HCL",
    "HCL Syntax",
    "Data Types",
    "Expressions",
    "Functions"
  ]
}
```

**Step 10.4: เขียน main.tf**

```hcl
# main.tf

# Generate a random ID สำหรับ unique identifier
resource "random_id" "student_id" {
  byte_length = 4
}

# สร้าง welcome file
resource "local_file" "welcome" {
  filename = "${path.module}/output/welcome.txt"
  content  = <<-EOT
    ╔══════════════════════════════════════╗
    ║   Welcome to Terraform/HCL Course!  ║
    ╚══════════════════════════════════════╝
    
    Student: ${var.student_name}
    Student ID: ${random_id.student_id.hex}
    Course: ${var.course_name}
    Lessons: ${var.lesson_count}
    
    Topics to be covered:
    ${join("\n    ", [for i, topic in var.topics : "${i + 1}. ${topic}"])}
    
    Good luck with your learning journey!
  EOT
}

# สร้าง course syllabus file
resource "local_file" "syllabus" {
  filename = "${path.module}/output/syllabus.json"
  content  = jsonencode({
    course = {
      name     = var.course_name
      student  = var.student_name
      lessons  = var.lesson_count
      topics   = var.topics
    }
    metadata = {
      created_at = timestamp()
      version    = "1.0.0"
    }
  })
}

# สร้าง progress tracker
resource "local_file" "progress" {
  filename = "${path.module}/output/progress.md"
  content  = <<-EOT
    # Learning Progress Tracker
    
    **Student:** ${var.student_name}
    **Course:** ${var.course_name}
    
    ## Progress
    
    ${join("\n", [for topic in var.topics : "- [ ] ${topic}"])}
    
    ## Notes
    
    _Add your notes here as you progress through the course_
  EOT
}
```

**Step 10.5: เขียน outputs.tf**

```hcl
# outputs.tf

output "student_id" {
  description = "Student ID ที่ generate มา"
  value       = random_id.student_id.hex
}

output "files_created" {
  description = "รายการไฟล์ที่สร้าง"
  value = [
    local_file.welcome.filename,
    local_file.syllabus.filename,
    local_file.progress.filename
  ]
}

output "welcome_message" {
  description = "Welcome message"
  value       = "Hello ${var.student_name}! Your student ID is ${random_id.student_id.hex}"
}
```

**Step 10.6: เขียน terraform.tfvars**

```hcl
# terraform.tfvars
student_name  = "สมชาย ใจดี"
course_name   = "HCL and Terraform Fundamentals"
lesson_count  = 100
topics = [
  "Introduction to HCL & IaC",
  "HCL Syntax Basics",
  "Primitive Data Types",
  "Complex Data Types",
  "Expressions & Operators",
  "Built-in Functions",
  "Conditional Expressions",
  "For Expressions",
  "Dynamic Blocks",
  "Template Syntax"
]
```

**Step 10.7: รัน Terraform Commands**

```bash
# 1. Initialize
terraform init
# Expected: Terraform initialized successfully

# 2. Validate syntax
terraform validate
# Expected: Success! The configuration is valid.

# 3. Format code
terraform fmt
# Expected: Files reformatted (ถ้ามี formatting issues)

# 4. Plan
terraform plan
# Expected: Plan: 4 to add, 0 to change, 0 to destroy.

# 5. Apply
terraform apply
# Type 'yes' when prompted
# Expected: Apply complete! Resources: 4 added, 0 changed, 0 destroyed.

# 6. ดู outputs
terraform output
# Expected: แสดง student_id, files_created, welcome_message

# 7. ดูไฟล์ที่สร้าง
ls output/
cat output/welcome.txt
cat output/progress.md

# 8. Destroy เมื่อเสร็จ
terraform destroy
# Type 'yes' when prompted
```

**Step 10.8: ตรวจสอบ State File**

```bash
# ดู state
terraform show

# List resources ใน state
terraform state list
# Output:
# local_file.progress
# local_file.syllabus
# local_file.welcome
# random_id.student_id

# ดู details ของ resource
terraform state show local_file.welcome
```

### Lab Expected Output

```
Apply complete! Resources: 4 added, 0 changed, 0 destroyed.

Outputs:

files_created = [
  "./output/welcome.txt",
  "./output/syllabus.json",
  "./output/progress.md",
]
student_id = "a1b2c3d4"
welcome_message = "Hello สมชาย ใจดี! Your student ID is a1b2c3d4"
```

---

## สรุป (Summary)

### สิ่งที่เรียนรู้ใน Part 001

| หัวข้อ | สรุป |
|--------|------|
| IaC | การจัดการ infrastructure ผ่านโค้ด |
| HCL | Language ของ HashiCorp สำหรับ config files |
| Terraform Workflow | init → validate → plan → apply → destroy |
| File Structure | main.tf, variables.tf, outputs.tf, terraform.tfvars |
| Comments | #, //, /* */ |

### Key Takeaways

💡 **Pro Tips:**
- ใช้ `terraform fmt` ก่อน commit เสมอ
- ใช้ `terraform plan` ก่อน `terraform apply` ทุกครั้ง
- เก็บ `.terraform/` ไว้ใน `.gitignore`
- อย่า commit `terraform.tfstate` ไปยัง Git (ใช้ remote backend แทน)

⚠️ **Common Mistakes:**
- ลืมรัน `terraform init` หลังเพิ่ม provider ใหม่
- Apply โดยไม่ดู plan ก่อน
- เก็บ secrets ใน `.tfvars` แล้ว commit ขึ้น Git
- ไม่ใช้ version constraints ใน required_providers

### เตรียมพร้อมสำหรับ Part 002

ใน Part 002 เราจะเรียนรู้เกี่ยวกับ HCL Syntax โดยละเอียด:
- Block syntax ทุกรูปแบบ
- Argument types ต่างๆ
- Identifier naming rules
- String literals และ heredoc
- Provider และ resource blocks

---

*ต่อไป: [Part 002 - HCL Syntax โครงสร้างพื้นฐาน](part-002.md)*
