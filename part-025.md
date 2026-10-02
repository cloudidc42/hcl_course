# Part 25: Terraform CLI - init, fmt, validate (ขั้นตอนที่ 241-250)

## ภาพรวม (Overview)

สามคำสั่งพื้นฐานที่ใช้ก่อนเริ่ม development workflow ทุกครั้ง:
- `terraform init` - เตรียม working directory
- `terraform fmt` - จัดรูปแบบ code
- `terraform validate` - ตรวจสอบ configuration

---

## Step 241: terraform init - What It Does

### init ทำอะไรบ้าง?

```
terraform init ทำ 4 อย่างหลัก:
1. Backend Initialization - configure state backend
2. Provider Installation - download providers
3. Module Installation - download modules  
4. .terraform.lock.hcl - สร้าง/อัพเดต lock file
```

### ตัวอย่าง Output ของ terraform init

```bash
$ terraform init

Initializing the backend...

Successfully configured the backend "s3"! Terraform will automatically
use this backend unless the backend configuration changes.

Initializing provider plugins...
- Finding hashicorp/aws versions matching "~> 5.0"...
- Finding hashicorp/random versions matching "~> 3.0"...
- Installing hashicorp/aws v5.31.0...
- Installed hashicorp/aws v5.31.0 (signed by HashiCorp)
- Installing hashicorp/random v3.6.0...
- Installed hashicorp/random v3.6.0 (signed by HashiCorp)

Terraform has created a lock file .terraform.lock.hcl to record the provider
selections it made above. Include this file in your version control repository
so that Terraform can guarantee to make the same selections by default when
you run "terraform init" in the future.

Terraform has been successfully initialized!

You may now begin working with Terraform. Try running "terraform plan" to see
any changes that are required for your infrastructure.
```

---

## Step 242: terraform init - Flags & Options

### -upgrade Flag

```bash
# Upgrade providers ให้เป็น version ล่าสุดที่ตรงกับ constraints
terraform init -upgrade

# ตัวอย่าง output:
# - Upgrading hashicorp/aws... (was 5.0.0, now 5.31.0)

# เมื่อไรใช้:
# - ต้องการ upgrade providers
# - หลังจากเปลี่ยน version constraints ใน required_providers
```

### -reconfigure Flag

```bash
# Reconfigure backend โดยไม่ migrate state
terraform init -reconfigure

# ใช้เมื่อ:
# - เปลี่ยน backend configuration
# - ต้องการ force reinitialize
# - Switch workspace ที่ใช้ backend ต่างกัน

# ⚠️ -reconfigure ไม่ migrate state เดิม!
# ถ้าต้องการ migrate ใช้ -migrate-state แทน
```

### -migrate-state Flag

```bash
# Initialize พร้อม migrate state ไป backend ใหม่
terraform init -migrate-state

# Terraform จะถาม:
# "Do you want to copy existing state to the new backend?"
# ตอบ yes เพื่อ migrate

# ตัวอย่าง: migrate จาก local ไป S3
# 1. เพิ่ม S3 backend config
# 2. Run:
terraform init -migrate-state
```

### -backend=false Flag

```bash
# Initialize โดยไม่ configure backend
# ใช้สำหรับ modules ที่ไม่มี backend
terraform init -backend=false

# ใช้เมื่อ:
# - Testing modules locally
# - ไม่ต้องการ state management
# - Running validate only
```

### -backend-config Flag

```bash
# Partial backend configuration
terraform init \
  -backend-config="bucket=my-state-bucket" \
  -backend-config="key=production/terraform.tfstate" \
  -backend-config="region=us-east-1"

# หรือจาก file
terraform init -backend-config="backend.hcl"

# ผสม file และ flags
terraform init \
  -backend-config="backend.hcl" \
  -backend-config="access_key=${AWS_ACCESS_KEY_ID}"
```

### -input Flag

```bash
# ป้องกัน interactive prompts
terraform init -input=false

# ใช้ใน CI/CD:
terraform init \
  -input=false \
  -backend-config="bucket=${TF_STATE_BUCKET}" \
  -backend-config="key=${TF_STATE_KEY}" \
  -backend-config="region=${AWS_REGION}"
```

---

## Step 243: .terraform Directory

### โครงสร้าง .terraform/

```
.terraform/
├── providers/
│   └── registry.terraform.io/
│       ├── hashicorp/
│       │   ├── aws/
│       │   │   └── 5.31.0/
│       │   │       └── linux_amd64/
│       │   │           └── terraform-provider-aws_v5.31.0_x5
│       │   └── random/
│       │       └── 3.6.0/
│       │           └── linux_amd64/
│       │               └── terraform-provider-random_v3.6.0_x5
├── modules/
│   ├── vpc/
│   │   └── ... (downloaded module files)
│   └── modules.json  (module registry)
└── terraform.tfstate  (backend configuration)
```

### ไม่ควร Commit .terraform/ ไปใน Git

```bash
# .gitignore
.terraform/
*.tfstate
*.tfstate.backup
*.tfplan
.terraform.lock.hcl  # optional: อาจ commit ก็ได้
```

### Provider Installation Directory

```bash
# Default: .terraform/providers/
# Custom: ใช้ TF_PLUGIN_DIR environment variable

export TF_PLUGIN_DIR=/path/to/custom/plugins
terraform init

# หรือใช้ CLI config (~/.terraformrc)
# provider_installation {
#   filesystem_mirror {
#     path    = "/usr/share/terraform/providers"
#     include = ["registry.terraform.io/*/*"]
#   }
#   direct {
#     exclude = ["registry.terraform.io/*/*"]
#   }
# }
```

---

## Step 244: .terraform.lock.hcl Explained

### Lock File คืออะไร?

```hcl
# .terraform.lock.hcl - auto-generated, should be committed to git
provider "registry.terraform.io/hashicorp/aws" {
  version     = "5.31.0"
  constraints = "~> 5.0"
  
  hashes = [
    "h1:abcdef1234567890...",  # hash ของ binary
    "zh:1234567890abcdef...",  # hash ของ zip
  ]
}

provider "registry.terraform.io/hashicorp/random" {
  version     = "3.6.0"
  constraints = "~> 3.0"
  
  hashes = [
    "h1:xyz789...",
    "zh:abc123...",
  ]
}
```

### Lock File ทำอะไร?

```bash
# Lock file รับประกันว่าทุกคนใน team ใช้ provider version เดียวกัน

# Developer A: terraform init -> ติดตั้ง aws 5.31.0, lock ไว้
# Developer B: terraform init -> ใช้ version จาก lock file (5.31.0)
# CI/CD: terraform init -> ใช้ version จาก lock file (5.31.0)

# ✅ Commit .terraform.lock.hcl ไปใน git
# เพื่อให้ทุกคนใช้ version เดียวกัน
```

### อัพเดต Lock File

```bash
# อัพเดต provider versions (เปลี่ยน lock file)
terraform init -upgrade

# Lock provider สำหรับหลาย platforms
terraform providers lock \
  -platform=linux_amd64 \
  -platform=darwin_amd64 \
  -platform=darwin_arm64 \
  -platform=windows_amd64
```

---

## Step 245: Module Installation

### ดาวน์โหลด Modules จาก Registry

```hcl
# main.tf
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "~> 5.0"
  
  name = "my-vpc"
  cidr = "10.0.0.0/16"
  
  azs             = ["us-east-1a", "us-east-1b", "us-east-1c"]
  private_subnets = ["10.0.1.0/24", "10.0.2.0/24", "10.0.3.0/24"]
  public_subnets  = ["10.0.101.0/24", "10.0.102.0/24", "10.0.103.0/24"]
  
  enable_nat_gateway = true
}
```

```bash
# Init จะ download module
terraform init

# Output:
# Initializing modules...
# Downloading registry.terraform.io/terraform-aws-modules/vpc/aws 5.4.0 for vpc...
# - vpc in .terraform/modules/vpc
```

### Modules จาก Git

```hcl
module "my_module" {
  source = "git::https://github.com/myorg/terraform-modules.git//modules/vpc?ref=v1.2.0"
}

# หรือ SSH
module "private_module" {
  source = "git::ssh://git@github.com/myorg/private-modules.git//vpc?ref=main"
}
```

```bash
# ต้อง init ก่อนใช้ module ใดๆ
terraform init

# Force re-download modules
terraform init -upgrade
```

---

## Step 246: terraform fmt - Auto-Formatting

### Syntax พื้นฐาน

```bash
terraform fmt
```

### Formatting Rules

```hcl
# ❌ ก่อน fmt
resource "aws_instance" "web" {
ami = "ami-xxx"
instance_type="t3.micro"
  tags={
    Name="web"
  Environment   =   "prod"
  }
}

# ✅ หลัง fmt
resource "aws_instance" "web" {
  ami           = "ami-xxx"
  instance_type = "t3.micro"
  tags = {
    Name        = "web"
    Environment = "prod"
  }
}
```

### fmt Rules ที่ Terraform ปฏิบัติ:

1. **Indentation**: 2 spaces
2. **Alignment**: align = signs ใน argument groups
3. **Spacing**: space รอบ operators
4. **Newlines**: blank lines ระหว่าง blocks
5. **Brackets**: opening brace บน same line

### -recursive Flag

```bash
# Format ไฟล์ในโฟลเดอร์ปัจจุบัน
terraform fmt

# Format ทุกไฟล์ใน subdirectories ด้วย
terraform fmt -recursive

# ตัวอย่าง: format ทั้ง project
terraform fmt -recursive .
```

### -diff Flag

```bash
# แสดง diff แต่ไม่แก้ไขไฟล์
terraform fmt -diff

# ตัวอย่าง output:
# --- main.tf  (original)
# +++ main.tf  (formatted)
# @@ -1,6 +1,6 @@
#  resource "aws_instance" "web" {
# -  ami = "ami-xxx"
# -  instance_type = "t3.micro"
# +  ami           = "ami-xxx"
# +  instance_type = "t3.micro"
#  }
```

### -check Flag (สำหรับ CI)

```bash
# Check formatting โดยไม่แก้ไข
# Exit code 0: format ถูกต้อง
# Exit code 3: พบ formatting issues
terraform fmt -check

# ใช้ใน CI pipeline
terraform fmt -check -recursive
if [ $? -ne 0 ]; then
  echo "ERROR: Terraform files are not formatted!"
  echo "Run 'terraform fmt -recursive' to fix"
  exit 1
fi
```

### Integration กับ Editors

```bash
# VS Code: ติดตั้ง HashiCorp Terraform extension
# Format on save อัตโนมัติ

# Settings.json:
# {
#   "[terraform]": {
#     "editor.formatOnSave": true,
#     "editor.defaultFormatter": "hashicorp.terraform"
#   }
# }

# vim: ใช้ vim-terraform plugin
# พิมพ์ :TerraformFmt เพื่อ format

# JetBrains: ใช้ Terraform plugin
# Format with: Ctrl+Alt+L (Windows/Linux) หรือ Cmd+Alt+L (Mac)
```

### Pre-commit Hook สำหรับ fmt

```yaml
# .pre-commit-config.yaml
repos:
  - repo: https://github.com/antonbabenko/pre-commit-terraform
    rev: v1.83.5
    hooks:
      - id: terraform_fmt
        args:
          - --args=-recursive

  - repo: https://github.com/pre-commit/pre-commit-hooks
    rev: v4.5.0
    hooks:
      - id: end-of-file-fixer
      - id: trailing-whitespace
```

```bash
# ติดตั้ง pre-commit
pip install pre-commit
pre-commit install

# Test
pre-commit run terraform_fmt --all-files
```

---

## Step 247: terraform validate

### Syntax พื้นฐาน

```bash
terraform validate

# Output ถ้าผ่าน:
# Success! The configuration is valid.

# Output ถ้าไม่ผ่าน:
# ╷
# │ Error: Reference to undeclared resource
# │
# │   on main.tf line 15, in resource "aws_instance" "web":
# │   15:   subnet_id = aws_subnet.nonexistent.id
# │
# │ A managed resource "aws_subnet" "nonexistent" has not been declared
# │ in the root module.
# ╵
```

### What Validate Checks

```
terraform validate ตรวจสอบ:
├── Syntax: HCL syntax ถูกต้องหรือไม่
├── References: variable/resource references ถูกต้องหรือไม่
├── Types: types ของ values ถูกต้องหรือไม่
├── Required arguments: มี required arguments ครบหรือไม่
├── Unknown attributes: มี attribute ที่ไม่มีอยู่หรือไม่
└── Expression logic: expressions ถูกต้องหรือไม่
```

```
terraform validate ไม่ตรวจสอบ:
├── Provider credentials
├── Resource existence ใน cloud
├── Quota/limits
└── Runtime values (computed attributes)
```

### ตัวอย่าง Validation Errors

```hcl
# Error 1: Missing required argument
resource "aws_instance" "bad" {
  # ❌ ขาด ami และ instance_type
  tags = { Name = "bad" }
}
# Error: Missing required argument
# The argument "ami" is required, but no definition was found.

# Error 2: Invalid type
variable "count_value" {
  type    = number
  default = "not-a-number"  # ❌ wrong type
}
# Error: Invalid default value for variable

# Error 3: Undeclared reference
resource "aws_instance" "web" {
  subnet_id = aws_subnet.nonexistent.id  # ❌ ไม่มี resource นี้
}
# Error: Reference to undeclared resource

# Error 4: Cycle
resource "aws_security_group" "a" {
  name = "sg-a"
  # ❌ circular reference
}

resource "aws_security_group_rule" "a_to_b" {
  security_group_id        = aws_security_group.a.id
  source_security_group_id = aws_security_group.b.id  # b depends on a
  # ...
}

resource "aws_security_group" "b" {
  name       = "sg-b"
  depends_on = [aws_security_group_rule.a_to_b]  # ❌ cycle!
}
```

### Validate กับ Variables

```bash
# validate ไม่ต้องการ actual values ของ variables
# แต่ต้องการ type information

# ✅ นี้ validate ได้
variable "instance_type" {
  type    = string
  default = "t3.micro"
}

resource "aws_instance" "web" {
  ami           = "ami-xxx"
  instance_type = var.instance_type
}

# ✅ validate ไม่ต้องการ provider credentials
# ไม่ต้อง AWS credentials เพื่อ validate
terraform validate
```

### -json Output

```bash
# Output แบบ JSON สำหรับ CI/CD processing
terraform validate -json

# ตัวอย่าง JSON output ถ้าผ่าน:
# {
#   "format_version": "1.0",
#   "valid": true,
#   "error_count": 0,
#   "warning_count": 0,
#   "diagnostics": []
# }

# ตัวอย่าง JSON output ถ้าไม่ผ่าน:
# {
#   "format_version": "1.0",
#   "valid": false,
#   "error_count": 2,
#   "warning_count": 0,
#   "diagnostics": [
#     {
#       "severity": "error",
#       "summary": "Missing required argument",
#       "detail": "The argument \"ami\" is required...",
#       "range": {
#         "filename": "main.tf",
#         "start": {"line": 5, "column": 1},
#         "end": {"line": 5, "column": 40}
#       }
#     }
#   ]
# }

# Parse ด้วย jq
terraform validate -json | jq '.valid'
terraform validate -json | jq '.diagnostics[] | .summary'
```

---

## Step 248: Validate in CI/CD

### GitHub Actions

```yaml
# .github/workflows/validate.yml
name: Terraform Validate

on:
  pull_request:
    paths:
      - '**.tf'
      - '**.tfvars'

jobs:
  validate:
    name: Validate Terraform
    runs-on: ubuntu-latest
    
    steps:
      - name: Checkout
        uses: actions/checkout@v4
      
      - name: Setup Terraform
        uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: "~1.6"
      
      - name: Terraform Format Check
        run: terraform fmt -check -recursive
      
      - name: Terraform Init
        run: terraform init -backend=false
        # -backend=false เพราะ CI ไม่มี credentials สำหรับ backend
        # validate ไม่ต้องการ backend
      
      - name: Terraform Validate
        run: terraform validate -json | tee /tmp/validate.json
      
      - name: Check Validation Result
        run: |
          VALID=$(cat /tmp/validate.json | jq -r '.valid')
          ERRORS=$(cat /tmp/validate.json | jq -r '.error_count')
          
          if [ "$VALID" != "true" ]; then
            echo "Terraform validation failed with $ERRORS errors!"
            cat /tmp/validate.json | jq '.diagnostics[]'
            exit 1
          fi
          
          echo "✅ Terraform configuration is valid!"
```

### GitLab CI

```yaml
# .gitlab-ci.yml
stages:
  - validate
  - plan
  - apply

variables:
  TF_VERSION: "1.6.0"

.terraform_before_script: &terraform_before_script
  before_script:
    - apk add --no-cache curl unzip
    - curl -fsSL "https://releases.hashicorp.com/terraform/${TF_VERSION}/terraform_${TF_VERSION}_linux_amd64.zip" -o terraform.zip
    - unzip terraform.zip
    - mv terraform /usr/local/bin/

terraform:validate:
  stage: validate
  <<: *terraform_before_script
  script:
    - terraform fmt -check -recursive
    - terraform init -backend=false
    - terraform validate
  only:
    - merge_requests
    - main
```

### Pre-commit Integration

```bash
# Complete .pre-commit-config.yaml
repos:
  - repo: https://github.com/antonbabenko/pre-commit-terraform
    rev: v1.83.5
    hooks:
      # Format check
      - id: terraform_fmt
        args: [--args=-recursive]
      
      # Validate
      - id: terraform_validate
        args:
          - --args=-no-color
        # ต้องการ init ก่อน:
        # pre-commit run terraform_validate --all-files
      
      # Lint ด้วย tflint
      - id: terraform_tflint
        args:
          - --args=--only=terraform_deprecated_interpolation
          - --args=--only=terraform_deprecated_index
          - --args=--only=terraform_unused_declarations
          - --args=--only=terraform_comment_syntax
          - --args=--only=terraform_documented_outputs
          - --args=--only=terraform_documented_variables
          - --args=--only=terraform_typed_variables
          - --args=--only=terraform_module_pinned_source
          - --args=--only=terraform_naming_convention
          - --args=--only=terraform_required_version
          - --args=--only=terraform_required_providers
      
      # Security scan
      - id: terraform_tfsec
```

---

## Step 249: Validation Errors vs Plan Errors

### Validation Errors (ตรวจได้โดยไม่ต้อง connect provider)

```hcl
# ❌ Validation Error: syntax error
resource "aws_instance" "web" {
  ami = "ami-xxx"  
  # ลืมปิด brace

# ❌ Validation Error: unknown attribute
resource "aws_instance" "web" {
  ami                = "ami-xxx"
  instance_type      = "t3.micro"
  nonexistent_field  = "value"  # ไม่มี attribute นี้
}

# ❌ Validation Error: type mismatch
variable "port" {
  type = number
}
resource "aws_security_group_rule" "web" {
  from_port = var.port
  to_port   = "not-a-number"  # ควรเป็น number
}
```

### Plan Errors (ตรวจได้เมื่อ connect provider)

```bash
# ❌ Plan Error: invalid AMI ID
resource "aws_instance" "web" {
  ami           = "ami-invalid"  # ผ่าน validate แต่ fail ตอน plan
  instance_type = "t3.micro"
}

# ❌ Plan Error: quota exceeded
# มีแค่ 5 Elastic IPs แต่จะสร้าง 6 อัน
resource "aws_eip" "web" {
  count = 6  # fail ตอน apply
}

# ❌ Plan Error: insufficient permissions
resource "aws_iam_role" "admin" {
  # ถ้า user ไม่มีสิทธิ์สร้าง IAM role
  name = "admin-role"
}
```

---

## Step 250: Common Init/Fmt/Validate Patterns

### Complete Init Script

```bash
#!/bin/bash
# scripts/init.sh

set -e  # Exit on error

ENV=${1:-dev}  # default: dev

echo "=== Initializing Terraform for environment: $ENV ==="

# Check required env vars
: "${AWS_ACCESS_KEY_ID:?Need AWS_ACCESS_KEY_ID}"
: "${AWS_SECRET_ACCESS_KEY:?Need AWS_SECRET_ACCESS_KEY}"

# Format check
echo "=== Checking formatting ==="
terraform fmt -check -recursive
echo "✅ Format check passed"

# Init
echo "=== Initializing ==="
terraform init \
  -backend-config="environments/${ENV}/backend.hcl" \
  -input=false

# Validate
echo "=== Validating ==="
terraform validate
echo "✅ Validation passed"

echo "=== Initialization complete! ==="
echo "Run: terraform plan -var-file=environments/${ENV}/terraform.tfvars"
```

### Makefile สำหรับ Terraform Workflow

```makefile
# Makefile
.PHONY: fmt fmt-check validate init plan apply destroy clean

ENV ?= dev
TF_VARS = -var-file="environments/$(ENV)/terraform.tfvars"

fmt:
	terraform fmt -recursive

fmt-check:
	terraform fmt -check -recursive

validate: init
	terraform validate

init:
	terraform init \
		-backend-config="environments/$(ENV)/backend.hcl" \
		-input=false

plan: init
	terraform plan $(TF_VARS) -out=tfplan.$(ENV)

apply:
	terraform apply tfplan.$(ENV)

apply-auto: init
	terraform apply -auto-approve $(TF_VARS)

destroy:
	terraform destroy $(TF_VARS)

clean:
	rm -rf .terraform/
	rm -f .terraform.lock.hcl
	rm -f *.tfplan

# CI targets
ci-validate: fmt-check validate
	echo "✅ All CI checks passed"

ci-plan: ci-validate
	terraform plan $(TF_VARS) -out=tfplan.$(ENV) -no-color 2>&1 | tee plan.log

ci-apply:
	terraform apply -auto-approve tfplan.$(ENV) -no-color 2>&1 | tee apply.log
```

### Terraform Wrapper Script

```bash
#!/bin/bash
# terraform-wrapper.sh
# ใช้แทน terraform command โดยตรง

COMMAND=$1
shift

case $COMMAND in
  init)
    echo "Running terraform init..."
    terraform init -input=false "$@"
    ;;
  
  fmt)
    echo "Formatting Terraform files..."
    terraform fmt -recursive
    echo "✅ Done"
    ;;
  
  validate)
    echo "Checking format..."
    terraform fmt -check -recursive || {
      echo "❌ Format check failed. Run: ./terraform-wrapper.sh fmt"
      exit 1
    }
    
    echo "Validating configuration..."
    terraform validate || {
      echo "❌ Validation failed"
      exit 1
    }
    
    echo "✅ All checks passed"
    ;;
  
  plan)
    ./terraform-wrapper.sh validate || exit 1
    echo "Running terraform plan..."
    terraform plan -out=tfplan "$@"
    ;;
  
  apply)
    if [ ! -f "tfplan" ]; then
      echo "❌ No plan file found. Run: ./terraform-wrapper.sh plan"
      exit 1
    fi
    echo "Running terraform apply..."
    terraform apply tfplan
    ;;
  
  *)
    echo "Usage: $0 {init|fmt|validate|plan|apply} [options]"
    exit 1
    ;;
esac
```

### VS Code Tasks Integration

```json
// .vscode/tasks.json
{
  "version": "2.0.0",
  "tasks": [
    {
      "label": "Terraform: Init",
      "type": "shell",
      "command": "terraform init",
      "group": "build",
      "presentation": {
        "reveal": "always",
        "panel": "new"
      }
    },
    {
      "label": "Terraform: Format",
      "type": "shell",
      "command": "terraform fmt -recursive",
      "group": "build"
    },
    {
      "label": "Terraform: Validate",
      "type": "shell",
      "command": "terraform validate",
      "group": "test",
      "dependsOn": ["Terraform: Init"]
    },
    {
      "label": "Terraform: Full Check",
      "type": "shell",
      "command": "terraform fmt -check -recursive && terraform validate",
      "group": "test",
      "dependsOn": ["Terraform: Init"]
    }
  ]
}
```

---

## Command Reference Summary

```bash
# === terraform init ===
terraform init                          # Basic init
terraform init -upgrade                 # Upgrade providers
terraform init -reconfigure             # Reconfigure backend
terraform init -migrate-state           # Migrate state to new backend
terraform init -backend=false           # Skip backend config
terraform init -input=false             # No interactive prompts
terraform init -backend-config=FILE     # Partial backend config
terraform init -get=false               # Skip module download

# === terraform fmt ===
terraform fmt                           # Format current directory
terraform fmt -recursive                # Format all subdirectories
terraform fmt -check                    # Check only (exit code)
terraform fmt -diff                     # Show diff without changing
terraform fmt -list=false               # Don't list changed files
terraform fmt -write=false              # Don't write changes

# === terraform validate ===
terraform validate                      # Validate configuration
terraform validate -json                # JSON output
terraform validate -no-color            # No ANSI colors (CI)
```

---

## Troubleshooting Common Issues

```bash
# Issue 1: "Error: Could not load plugin"
terraform init  # ต้อง init ก่อน

# Issue 2: "Error: Backend configuration changed"
terraform init -reconfigure

# Issue 3: "Error: Failed to query available provider packages"
# Network issues
export HTTPS_PROXY=http://proxy:8080
terraform init

# Issue 4: Provider version conflict
# เปลี่ยน version constraints แล้วรัน:
terraform init -upgrade

# Issue 5: Module not found after adding
terraform init  # ต้อง init อีกครั้ง

# Issue 6: fmt ไม่ทำงานบน Windows
# ตรวจสอบ line endings
git config core.autocrlf false
```

---

## แบบฝึกหัด (Exercises)

### Exercise 1: Setup Development Workflow

1. สร้าง Terraform project ใหม่
2. รัน `terraform init`
3. เพิ่มการ formatting ที่ผิด แล้วรัน `terraform fmt`
4. รัน `terraform validate` ดู result

### Exercise 2: CI Integration

สร้าง GitHub Actions workflow ที่:
1. Check formatting
2. Init (without backend)
3. Validate
4. แสดง PR comment ถ้า validation ล้มเหลว

### Exercise 3: Pre-commit Setup

1. ติดตั้ง pre-commit
2. สร้าง .pre-commit-config.yaml
3. ทดสอบ hooks

---

## Checklist

- [ ] รู้ว่า `terraform init` ทำอะไรบ้าง
- [ ] เข้าใจ .terraform directory structure
- [ ] เข้าใจ .terraform.lock.hcl
- [ ] รู้วิธีใช้ flags ต่างๆ ของ init
- [ ] สามารถ format Terraform files ได้
- [ ] รู้วิธี integrate fmt กับ CI
- [ ] เข้าใจสิ่งที่ validate ตรวจสอบและไม่ตรวจสอบ
- [ ] สามารถใช้ validate ใน CI/CD ได้
- [ ] รู้ความต่างระหว่าง validation errors และ plan errors
