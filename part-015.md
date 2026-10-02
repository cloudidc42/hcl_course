# Part 015: HCL Best Practices (แนวปฏิบัติที่ดีที่สุด)
## Steps 141-150: Terraform Best Practices ที่ควรปฏิบัติตาม

---

## บทนำ (Introduction)

Best Practices ใน Terraform ช่วยให้ code มีคุณภาพ maintainable และ scalable การปฏิบัติตาม standards เหล่านี้ลด technical debt และทำให้ทีมทำงานร่วมกันได้ง่ายขึ้น

---

## Step 141: File Organization

### โครงสร้างไฟล์มาตรฐาน

```
project/
├── environments/
│   ├── dev/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   ├── outputs.tf
│   │   └── terraform.tfvars
│   ├── staging/
│   │   └── ...
│   └── prod/
│       └── ...
├── modules/
│   ├── vpc/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   ├── outputs.tf
│   │   ├── locals.tf
│   │   ├── versions.tf
│   │   └── README.md
│   └── ec2/
│       └── ...
└── README.md
```

### ไฟล์มาตรฐานและหน้าที่

```hcl
# main.tf - Resources หลัก, module calls, data sources
resource "aws_vpc" "main" { ... }
module "vpc" { ... }
data "aws_ami" "main" { ... }

# variables.tf - Input variable declarations เท่านั้น
variable "project_name" { ... }

# outputs.tf - Output value declarations เท่านั้น
output "vpc_id" { ... }

# locals.tf - Local value computations
locals { ... }

# versions.tf - Terraform และ provider version constraints
terraform {
  required_version = ">= 1.3.0"
  required_providers { ... }
}

# backend.tf - Backend configuration (แยกออกมา)
terraform {
  backend "s3" { ... }
}

# data.tf - Data sources (ถ้ามีมาก)
data "aws_caller_identity" "current" {}
```

### ✅ Do / ❌ Don't

```hcl
# ✅ แยกไฟล์ตาม purpose
# main.tf, variables.tf, outputs.tf, locals.tf, versions.tf

# ❌ รวมทุกอย่างใน main.tf
# main.tf ยาว 1000+ บรรทัด ทุกอย่างอยู่ในไฟล์เดียว

# ✅ ชื่อไฟล์สื่อความหมาย
# security_groups.tf, iam.tf, s3.tf (ถ้า resources มีมาก)

# ❌ ชื่อไฟล์ที่ไม่สื่อ
# file1.tf, stuff.tf, misc.tf
```

---

## Step 142: Naming Conventions

### Resource Naming

```hcl
# ✅ ชื่อ resource: lowercase + underscores
resource "aws_instance" "web_server" { ... }
resource "aws_security_group" "app_servers" { ... }
resource "aws_s3_bucket" "static_assets" { ... }
resource "aws_db_instance" "primary" { ... }

# ❌ ชื่อที่ไม่ดี
resource "aws_instance" "WebServer" { ... }  # CamelCase
resource "aws_instance" "web-server" { ... }  # hyphen ไม่รองรับ
resource "aws_instance" "ws" { ... }  # abbreviation ไม่ชัดเจน

# ✅ ใช้ "this" สำหรับ single resource ใน module
resource "aws_vpc" "this" { ... }  # เพราะ module มี VPC เดียว

# ✅ ใช้ descriptive name สำหรับ multiple resources
resource "aws_subnet" "public" { ... }
resource "aws_subnet" "private" { ... }
resource "aws_subnet" "database" { ... }
```

### Variable Naming

```hcl
# ✅ ชื่อ variable: lowercase + underscores, noun phrases
variable "vpc_cidr_block" { ... }
variable "instance_count" { ... }
variable "enable_monitoring" { ... }  # booleans: enable_/create_/is_
variable "environment_name" { ... }

# ❌ ชื่อที่ไม่ดี
variable "CIDR" { ... }          # uppercase
variable "cidrBlock" { ... }     # camelCase
variable "the_vpc_cidr" { ... }  # unnecessary article
variable "x" { ... }             # ไม่สื่อความหมาย

# ✅ Boolean variables ใช้ prefix
variable "enable_nat_gateway" { ... }
variable "create_security_group" { ... }
variable "is_production" { ... }
variable "use_existing_vpc" { ... }
```

### Output Naming

```hcl
# ✅ ชื่อ output ให้ match กับ resource attribute
output "vpc_id" {          # ตรงกับ aws_vpc.main.id
  value = aws_vpc.main.id
}

output "instance_public_ip" {   # ชัดเจน
  value = aws_instance.web.public_ip
}

# ✅ Module outputs ให้ consistent กับ input variable ของ module ที่จะรับ
# module "compute" ต้องการ vpc_id → module "vpc" expose output "vpc_id"

# ❌ ไม่ดี
output "id" { ... }       # ไม่ชัดเจนว่า id ของอะไร
output "the_ip" { ... }   # ไม่จำเป็นต้องมี article
```

### Module Naming

```hcl
# ✅ ชื่อ module local name: lowercase + underscores
module "vpc" { ... }
module "web_servers" { ... }
module "database_cluster" { ... }

# ✅ Module directory/repository naming convention
# terraform-<provider>-<name>  (official)
# modules/vpc/
# modules/ecs-service/
# modules/rds-mysql/
```

---

## Step 143: Code Formatting

### terraform fmt

```bash
# Format code อัตโนมัติ
terraform fmt

# Format แบบ recursive (ทุก subdirectory)
terraform fmt -recursive

# ดูว่าไฟล์ไหนต้อง format (ไม่แก้)
terraform fmt -check

# ดู diff ของการ format
terraform fmt -diff

# ใช้ใน pre-commit hook
terraform fmt -check -recursive || exit 1
```

### ตัวอย่าง Formatting Rules

```hcl
# ❌ ก่อน terraform fmt
resource "aws_instance" "web" {
ami="ami-0c55b159cbfafe1f0"
  instance_type  =   "t3.micro"
  tags={Name="web",Environment="prod"}
}

# ✅ หลัง terraform fmt
resource "aws_instance" "web" {
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t3.micro"
  
  tags = {
    Name        = "web"
    Environment = "prod"
  }
}
```

### Alignment Rules

```hcl
# ✅ Align ค่าของ argument ที่อยู่ติดกัน
resource "aws_security_group_rule" "example" {
  type        = "ingress"
  from_port   = 443
  to_port     = 443
  protocol    = "tcp"
  cidr_blocks = ["0.0.0.0/0"]
}

# ✅ ใส่ blank lines แยก logical groups
resource "aws_instance" "web" {
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t3.micro"
  
  # Network configuration
  subnet_id              = aws_subnet.public.id
  vpc_security_group_ids = [aws_security_group.web.id]
  
  # Storage configuration
  root_block_device {
    volume_type = "gp3"
    volume_size = 20
  }
  
  tags = local.compute_tags
}
```

---

## Step 144: Code Validation

### terraform validate

```bash
# ตรวจสอบ syntax และ internal consistency
terraform validate

# ตัวอย่าง output เมื่อถูก:
# Success! The configuration is valid.

# ตัวอย่าง error:
# Error: Reference to undeclared resource
# 
# on main.tf line 15:
#   vpc_id = aws_vpc.nonexistent.id
```

### tflint - Linting Tool

```bash
# ติดตั้ง tflint
brew install tflint  # macOS
# หรือ download binary จาก GitHub

# ติดตั้ง AWS ruleset
tflint --init

# Run tflint
tflint

# ตัวอย่าง errors:
# Error: aws_instance_invalid_type
#   on main.tf line 5:
#   instance_type = "t2.xlarge"  # deprecated instance type
```

```hcl
# .tflint.hcl - configuration file
plugin "aws" {
  enabled = true
  version = "0.24.0"
  source  = "github.com/terraform-linters/tflint-ruleset-aws"
}

rule "terraform_deprecated_index" {
  enabled = true
}

rule "terraform_unused_declarations" {
  enabled = true
}

rule "terraform_comment_syntax" {
  enabled = true
}

rule "terraform_documented_outputs" {
  enabled = true
}

rule "terraform_documented_variables" {
  enabled = true
}
```

### checkov - Security Scanning

```bash
# ติดตั้ง checkov
pip install checkov

# Scan Terraform code
checkov -d .

# ดู specific checks
checkov -d . --check CKV_AWS_79  # ตรวจ IMDSv2
checkov -d . --framework terraform

# เพิ่ม checkov annotation เพื่อ skip
```

```hcl
# checkov annotations
resource "aws_s3_bucket" "logs" {
  #checkov:skip=CKV_AWS_18:Access logging is enabled on the main bucket
  bucket = "${var.project}-logs"
}
```

---

## Step 145: Documentation with terraform-docs

### ติดตั้งและใช้งาน

```bash
# ติดตั้ง
brew install terraform-docs  # macOS

# สร้าง README.md
terraform-docs markdown . > README.md

# สร้างแบบ pretty
terraform-docs markdown table . > README.md

# JSON format
terraform-docs json .

# ใช้กับ module
terraform-docs markdown ./modules/vpc > ./modules/vpc/README.md
```

### ตัวอย่าง terraform-docs output

```markdown
## Requirements

| Name | Version |
|------|---------|
| terraform | >= 1.3.0 |
| aws | >= 5.0.0 |

## Inputs

| Name | Description | Type | Default | Required |
|------|-------------|------|---------|:--------:|
| name | Name prefix | `string` | n/a | yes |
| vpc_cidr | VPC CIDR block | `string` | `"10.0.0.0/16"` | no |

## Outputs

| Name | Description |
|------|-------------|
| vpc_id | The ID of the VPC |
```

### .terraform-docs.yml Configuration

```yaml
# .terraform-docs.yml
formatter: "markdown table"

output:
  file: "README.md"
  mode: inject
  template: |-
    <!-- BEGIN_TF_DOCS -->
    {{ .Content }}
    <!-- END_TF_DOCS -->

sections:
  show:
    - requirements
    - providers
    - inputs
    - outputs

sort:
  enabled: true
  by: required
```

---

## Step 146: Version Pinning

### Terraform Version Pinning

```hcl
# versions.tf

terraform {
  # ✅ Pin Terraform version
  required_version = ">= 1.3.0, < 2.0.0"
  # หรือ
  required_version = "~> 1.6"  # >= 1.6, < 2.0

  required_providers {
    # ✅ Pin Provider versions
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"  # >= 5.0, < 6.0
    }
    
    random = {
      source  = "hashicorp/random"
      version = "~> 3.5"
    }
    
    tls = {
      source  = "hashicorp/tls"
      version = "~> 4.0"
    }
  }
}
```

### .terraform.lock.hcl

```hcl
# .terraform.lock.hcl - lock file (ควร commit เข้า git)
provider "registry.terraform.io/hashicorp/aws" {
  version     = "5.31.0"
  constraints = "~> 5.0"
  hashes = [
    "h1:...",
    "zh:...",
  ]
}
```

### ทำไมต้อง Commit Lock File?

```bash
# ✅ Commit .terraform.lock.hcl
# - ทำให้ทุกคนใน team ใช้ provider version เดียวกัน
# - CI/CD pipeline ใช้ version เดียวกัน
# - ลด supply chain attacks

# Update lock file
terraform init -upgrade  # upgrade to latest matching version
```

---

## Step 147: DRY Principle

### Don't Repeat Yourself

```hcl
# ❌ ไม่ดี: ซ้ำซาก
resource "aws_security_group_rule" "http_web" {
  type        = "ingress"
  from_port   = 80
  to_port     = 80
  protocol    = "tcp"
  cidr_blocks = ["0.0.0.0/0"]
  security_group_id = aws_security_group.web.id
}

resource "aws_security_group_rule" "https_web" {
  type        = "ingress"
  from_port   = 443
  to_port     = 443
  protocol    = "tcp"
  cidr_blocks = ["0.0.0.0/0"]
  security_group_id = aws_security_group.web.id
}

# ✅ ดีกว่า: ใช้ for_each
locals {
  web_ingress_rules = {
    http  = { port = 80,  description = "HTTP" }
    https = { port = 443, description = "HTTPS" }
  }
}

resource "aws_security_group_rule" "web_ingress" {
  for_each = local.web_ingress_rules

  type        = "ingress"
  from_port   = each.value.port
  to_port     = each.value.port
  protocol    = "tcp"
  cidr_blocks = ["0.0.0.0/0"]
  description = each.value.description
  
  security_group_id = aws_security_group.web.id
}
```

### Avoid Hardcoded Values

```hcl
# ❌ ไม่ดี: hardcoded values
resource "aws_instance" "web" {
  ami           = "ami-0c55b159cbfafe1f0"  # hardcoded!
  instance_type = "t3.micro"               # hardcoded!
  
  tags = {
    Name        = "my-project-prod-web"    # hardcoded!
    Environment = "prod"                   # hardcoded!
  }
}

# ✅ ดี: ใช้ variables, locals, data sources
data "aws_ami" "amazon_linux" {
  most_recent = true
  owners      = ["amazon"]
  filter {
    name   = "name"
    values = ["amzn2-ami-hvm-*-x86_64-gp2"]
  }
}

resource "aws_instance" "web" {
  ami           = data.aws_ami.amazon_linux.id  # dynamic
  instance_type = var.instance_type             # variable
  
  tags = merge(local.common_tags, {
    Name = "${local.name_prefix}-web"  # computed
  })
}
```

---

## Step 148: Tag Strategy

### Mandatory Tags

```hcl
# locals.tf - Define tagging strategy
locals {
  # Mandatory tags ที่ทุก resource ต้องมี
  mandatory_tags = {
    # Identity
    Project     = var.project_name
    Environment = var.environment
    
    # Ownership
    Team        = var.team_name
    Owner       = var.owner_email
    
    # Operations
    ManagedBy   = "terraform"
    Repository  = var.git_repository
    
    # Cost allocation
    CostCenter  = var.cost_center
    BillingCode = var.billing_code
  }
  
  # เพิ่ม optional tags
  common_tags = merge(local.mandatory_tags, var.additional_tags)
  
  # Resource-specific tags
  compute_tags = merge(local.common_tags, {
    Tier        = "Compute"
    PatchGroup  = "AmazonLinux2"
  })
  
  data_tags = merge(local.common_tags, {
    Tier        = "Data"
    DataClass   = var.data_classification
    BackupPolicy = local.is_production ? "standard" : "none"
  })
}
```

### Enforce Tags ด้วย Validation

```hcl
variable "tags" {
  description = "Resource tags"
  type        = map(string)
  default     = {}

  validation {
    condition     = contains(keys(var.tags), "Project")
    error_message = "Tags must include 'Project' key."
  }
  
  validation {
    condition     = contains(keys(var.tags), "Owner")
    error_message = "Tags must include 'Owner' key with email address."
  }
  
  validation {
    condition = contains(
      ["dev", "staging", "prod"],
      lookup(var.tags, "Environment", "")
    )
    error_message = "Tags must include 'Environment' key with value dev, staging, or prod."
  }
}
```

---

## Step 149: Secret Management

### ❌ อย่าทำ

```hcl
# ❌ อย่า hardcode secrets
resource "aws_db_instance" "main" {
  username = "admin"
  password = "MyPassword123!"  # NEVER DO THIS!
}

# ❌ อย่าเก็บใน .tfvars ที่ commit เข้า git
# terraform.tfvars:
# database_password = "MyPassword123!"  # NEVER!

# ❌ อย่า output secrets ที่ไม่ sensitive
output "db_password" {
  value = var.database_password  # จะ error ถ้าไม่ mark sensitive
}
```

### ✅ วิธีที่ถูกต้อง

```hcl
# ✅ วิธี 1: ใช้ Environment Variables
# export TF_VAR_database_password="$(aws secretsmanager get-secret-value ...)"

# ✅ วิธี 2: AWS Secrets Manager
data "aws_secretsmanager_secret_version" "db" {
  secret_id = "prod/myapp/database"
}

locals {
  db_credentials = jsondecode(data.aws_secretsmanager_secret_version.db.secret_string)
}

resource "aws_db_instance" "main" {
  username = local.db_credentials.username
  password = local.db_credentials.password
}

# ✅ วิธี 3: HashiCorp Vault
provider "vault" {
  address = var.vault_address
}

data "vault_generic_secret" "db" {
  path = "secret/myapp/db"
}

resource "aws_db_instance" "main" {
  password = data.vault_generic_secret.db.data["password"]
}

# ✅ วิธี 4: AWS SSM Parameter Store
data "aws_ssm_parameter" "db_password" {
  name            = "/myapp/${var.environment}/db/password"
  with_decryption = true
}

resource "aws_db_instance" "main" {
  password = data.aws_ssm_parameter.db_password.value
}
```

### .gitignore สำหรับ Terraform

```gitignore
# .gitignore

# Local .terraform directories
**/.terraform/*

# .tfstate files
*.tfstate
*.tfstate.*
*.tfstate.backup

# Crash log files
crash.log
crash.*.log

# Exclude all .tfvars files (may contain secrets)
*.tfvars
*.tfvars.json

# Override files (local customization)
override.tf
override.tf.json
*_override.tf
*_override.tf.json

# Include example tfvars
!terraform.tfvars.example

# Terraform plan files
*.tfplan

# Terraform lock file (should be committed!)
# .terraform.lock.hcl  <- อย่า gitignore นี้!

# SSH keys
*.pem
*.key

# Sensitive files
secrets.tf
sensitive.tfvars
```

---

## Step 150: Pre-commit Hooks

### ติดตั้ง pre-commit

```bash
# ติดตั้ง
pip install pre-commit
brew install pre-commit  # macOS

# สร้าง .pre-commit-config.yaml
pre-commit install
```

### .pre-commit-config.yaml

```yaml
# .pre-commit-config.yaml
repos:
  - repo: https://github.com/antonbabenko/pre-commit-terraform
    rev: v1.83.5
    hooks:
      # Format Terraform code
      - id: terraform_fmt
        args:
          - --args=-recursive
      
      # Validate Terraform code
      - id: terraform_validate
        args:
          - --init-args=-backend=false
      
      # Lint with tflint
      - id: terraform_tflint
        args:
          - --args=--config=__GIT_WORKING_DIR__/.tflint.hcl
      
      # Security scan with checkov
      - id: terraform_checkov
        args:
          - --args=--quiet
          - --args=--compact
      
      # Update docs
      - id: terraform_docs
        args:
          - --hook-config=--path-to-file=README.md
          - --hook-config=--add-to-existing-file=true
          - --hook-config=--create-file-if-not-exist=true
  
  - repo: https://github.com/pre-commit/pre-commit-hooks
    rev: v4.5.0
    hooks:
      # ป้องกัน commit ไฟล์ขนาดใหญ่
      - id: check-added-large-files
        args: ['--maxkb=1024']
      
      # ป้องกัน commit ไปที่ main/master
      - id: no-commit-to-branch
        args: ['--branch', 'main', '--branch', 'master']
      
      # ตรวจ trailing whitespace
      - id: trailing-whitespace
      
      # ตรวจ YAML syntax
      - id: check-yaml
      
      # ตรวจ JSON syntax
      - id: check-json
      
      # ป้องกัน commit private keys
      - id: detect-private-key
```

---

## Code Review Checklist

### Terraform Code Review

```markdown
## Terraform Code Review Checklist

### Structure ✅
- [ ] ไฟล์แยกตาม purpose (main.tf, variables.tf, outputs.tf)
- [ ] Module structure สมบูรณ์
- [ ] versions.tf มี required_version และ required_providers

### Variables ✅
- [ ] ทุก variable มี description
- [ ] ใช้ validation blocks สำหรับ critical variables
- [ ] sensitive = true สำหรับ secrets
- [ ] ไม่มี hardcoded values

### Resources ✅
- [ ] Resource names ตาม naming convention
- [ ] tags ครบตาม tagging strategy
- [ ] ไม่มี hardcoded secrets
- [ ] lifecycle blocks เหมาะสม

### Security ✅
- [ ] IAM policies มี least privilege
- [ ] Security groups ไม่ open 0.0.0.0/0 โดยไม่จำเป็น
- [ ] Encryption enabled สำหรับ data at rest
- [ ] Secrets ไม่อยู่ใน code

### Performance ✅
- [ ] ใช้ count/for_each แทน copy-paste
- [ ] Locals สำหรับ computed values ที่ใช้ซ้ำ
- [ ] ไม่มี circular dependencies

### Documentation ✅
- [ ] terraform fmt ผ่าน
- [ ] terraform validate ผ่าน
- [ ] README.md อัพเดท
- [ ] CHANGELOG อัพเดท (ถ้ามี)
```

---

## สรุป (Summary)

### Best Practices สรุป

| Category | Practice |
|----------|----------|
| Structure | แยกไฟล์ตาม purpose |
| Naming | lowercase_underscore ทั้งหมด |
| Format | ใช้ terraform fmt เสมอ |
| Version | Pin ทั้ง terraform และ providers |
| DRY | ใช้ for_each, locals, modules |
| Tags | Mandatory tags ทุก resource |
| Secrets | ไม่ hardcode ใน .tf files |
| Docs | description ทุก variable/output |
| CI/CD | pre-commit hooks |

### ✅ Do / ❌ Don't Summary

```
✅ Do:
- terraform fmt -recursive ก่อน commit
- Pin versions ด้วย ~> operator  
- ใส่ description ทุก variable/output
- ใช้ sensitive = true สำหรับ secrets
- Commit .terraform.lock.hcl
- gitignore *.tfvars (อาจมี secrets)

❌ Don't:
- Hardcode secrets ใน .tf files
- ใช้ version = "*" (no pinning)
- ออก output sensitive value โดยไม่ mark sensitive
- Copy-paste resources แทนการใช้ for_each
- Commit state files (.tfstate)
- ใช้ count บน resources ที่มี identifier ที่เปลี่ยนได้
```

### 💡 Pro Tips

1. **ใช้ `terraform-docs`** สร้าง README อัตโนมัติ
2. **ทำ `terraform fmt -check`** ใน CI/CD pipeline
3. **ทำ `terraform validate`** ก่อน plan
4. **ใช้ checkov/tfsec** scan security issues
5. **สร้าง .tfvars.example** เป็น template สำหรับทีม

---

*จบ Part 015 - HCL Best Practices*
