# Part 30: Terraform Debugging & Troubleshooting (ขั้นตอนที่ 291-300)

## ภาพรวม (Overview)

Debugging Terraform เป็นทักษะที่สำคัญมาก การรู้จักเครื่องมือและวิธีอ่าน error messages จะช่วยประหยัดเวลาได้มาก คู่มือนี้ครอบคลุมทุกแง่มุมของการ debug และ troubleshoot Terraform

---

## Step 291: TF_LOG - Debug Logging

### TF_LOG Levels

```bash
# กำหนด log level ด้วย environment variable
export TF_LOG=TRACE   # Verbose ที่สุด - ทุก operation
export TF_LOG=DEBUG   # Detailed information
export TF_LOG=INFO    # Informational messages (default)
export TF_LOG=WARN    # Warnings only
export TF_LOG=ERROR   # Errors only
export TF_LOG=OFF     # ปิด logging (default)

# รัน Terraform พร้อม logging
TF_LOG=DEBUG terraform plan

# ยกเลิก logging
unset TF_LOG
```

### TF_LOG_PATH - Save Logs ไฟล์

```bash
# Save logs ไปไฟล์
export TF_LOG=DEBUG
export TF_LOG_PATH="/tmp/terraform-debug.log"
terraform apply

# ดู logs
cat /tmp/terraform-debug.log
tail -f /tmp/terraform-debug.log  # Follow real-time

# Save logs พร้อม timestamp
export TF_LOG_PATH="terraform-$(date +%Y%m%d-%H%M%S).log"
terraform plan
```

### TF_LOG_CORE vs TF_LOG_PROVIDER

```bash
# TF_LOG_CORE: Log สำหรับ Terraform core เท่านั้น
export TF_LOG_CORE=DEBUG
export TF_LOG_PROVIDER=ERROR

# TF_LOG_PROVIDER: Log สำหรับ provider เท่านั้น
export TF_LOG_CORE=ERROR
export TF_LOG_PROVIDER=DEBUG

# ทั้งคู่พร้อมกัน
export TF_LOG_CORE=INFO
export TF_LOG_PROVIDER=DEBUG
terraform plan
```

### อ่าน Debug Output

```bash
# ตัวอย่าง TRACE output (มาก!)
$ TF_LOG=TRACE terraform plan 2>&1 | head -50

2024-01-15T10:30:00.123+0700 [INFO]  Terraform version: 1.6.0
2024-01-15T10:30:00.124+0700 [DEBUG] using github.com/zclconf/go-cty v1.14.1
2024-01-15T10:30:00.125+0700 [INFO]  Go runtime version: go1.21.5
2024-01-15T10:30:00.126+0700 [INFO]  CLI args: []string{"terraform", "plan"}
2024-01-15T10:30:00.127+0700 [TRACE] Preserving existing state lineage "uuid..."
2024-01-15T10:30:00.128+0700 [DEBUG] checking for provider in "."
2024-01-15T10:30:00.129+0700 [TRACE] Loading state...
2024-01-15T10:30:00.130+0700 [DEBUG] provider: starting provider plugin...
2024-01-15T10:30:00.200+0700 [TRACE] aws: configuring AWS SDK...
2024-01-15T10:30:00.201+0700 [TRACE] aws: initializing region us-east-1

# ส่วนที่น่าสนใจ:
# - Provider API calls
# - HTTP requests/responses
# - State operations
# - Resource refresh details
```

### กรองเฉพาะส่วนที่ต้องการ

```bash
# กรอง error เท่านั้น
TF_LOG=DEBUG terraform apply 2>&1 | grep -E "ERROR|Error|error"

# กรอง AWS API calls
TF_LOG=DEBUG terraform plan 2>&1 | grep -E "GET|POST|PUT|DELETE|HTTP"

# กรอง specific resource
TF_LOG=DEBUG terraform plan 2>&1 | grep "aws_instance.web"

# Save และ filter
TF_LOG=DEBUG TF_LOG_PATH=debug.log terraform plan
grep "Error\|Warning" debug.log
```

---

## Step 292: Common Error #1 - Configuration Errors

### "Error: No configuration files"

```bash
# Error:
$ terraform plan
Error: No configuration files

There is no configuration in "/path/to/directory". 
A "backend" configuration must be present in at least one
configuration file in this directory.

# สาเหตุ: ไม่มีไฟล์ .tf ในโฟลเดอร์ปัจจุบัน

# วิธีแก้:
ls -la  # ตรวจสอบมีไฟล์ .tf หรือไม่
pwd     # ตรวจสอบอยู่โฟลเดอร์ถูกต้องหรือไม่
cd /correct/terraform/directory
terraform plan
```

### "Error: Provider configuration not present"

```bash
# Error:
Error: Provider configuration not present

To work with aws_instance.web its original provider configuration at
provider["registry.terraform.io/hashicorp/aws"] is required,
but it has been removed. This occurs when a provider configuration 
is removed while objects created by that provider still exist in the state.

# สาเหตุ: ลบ provider block ออกจาก config แต่ยังมี resources ใน state

# วิธีแก้:
# 1. เพิ่ม provider block กลับมา
provider "aws" {
  region = "us-east-1"
}

# 2. Destroy resources ก่อนลบ provider
terraform destroy

# 3. หรือลบออกจาก state
terraform state rm aws_instance.web
```

### "Error: Reference to undeclared resource"

```bash
# Error:
Error: Reference to undeclared resource

on main.tf line 15, in resource "aws_instance" "web":
15:   subnet_id = aws_subnet.nonexistent.id

A managed resource "aws_subnet" "nonexistent" has not been declared
in the root module.

# วิธีแก้:
# 1. ตรวจสอบชื่อ resource ว่าถูกต้อง
terraform state list  # ดู resources ที่มีอยู่

# 2. เพิ่ม resource ที่ขาดหายไป
resource "aws_subnet" "nonexistent" {
  # ...
}

# 3. หรือแก้ reference ให้ถูกต้อง
subnet_id = aws_subnet.correct_name.id
```

---

## Step 293: Common Error #2 - State Lock Errors

### "Error acquiring the state lock"

```bash
# Error:
Error: Error acquiring the state lock

Error message: ConditionalCheckFailedException: 
The conditional request failed

Lock Info:
  ID:        12345678-1234-1234-1234-1234567890ab
  Path:      s3://my-bucket/state.tfstate
  Operation: OperationTypeApply
  Who:       user@hostname
  Version:   1.6.0
  Created:   2024-01-15 10:30:00.000000000 +0000 UTC
  Info:      

Terraform acquires a state lock to protect the state from being
written by multiple users at the same time.

# วิธีแก้:
# ตรวจสอบว่า operation ก่อนหน้า ยังรันอยู่หรือไม่
# ถ้าแน่ใจว่าไม่มีคนอื่น:

# Force unlock (ต้องระบุ Lock ID จาก error)
terraform force-unlock 12345678-1234-1234-1234-1234567890ab

# ถ้า lock อยู่ใน DynamoDB:
aws dynamodb delete-item \
  --table-name terraform-state-locks \
  --key '{"LockID": {"S": "my-bucket/state.tfstate"}}'
```

### ป้องกัน Lock Issues

```bash
# ตรวจสอบก่อน run ว่ามี lock อยู่หรือไม่
aws dynamodb get-item \
  --table-name terraform-state-locks \
  --key '{"LockID": {"S": "my-bucket/path/terraform.tfstate"}}'

# Script ที่ตรวจสอบ lock ก่อน apply
check_state_lock() {
  LOCK=$(aws dynamodb get-item \
    --table-name terraform-state-locks \
    --key "{\"LockID\": {\"S\": \"$TF_STATE_KEY\"}}" \
    2>/dev/null)
  
  if [ -n "$LOCK" ]; then
    echo "⚠️  State is currently locked!"
    echo "$LOCK" | jq '.Item.Info.S' | jq -r '.'
    return 1
  fi
  return 0
}
```

---

## Step 294: Common Error #3 - Provider Errors

### Provider Authentication Errors

```bash
# AWS Authentication Error:
Error: configuring Terraform AWS Provider: no valid credential sources found

Please see https://registry.terraform.io/providers/hashicorp/aws/latest/docs
for information about providing credentials.

Error: failed to get shared config profile, myprofile

# วิธีแก้:
# 1. Set environment variables
export AWS_ACCESS_KEY_ID="AKIAIOSFODNN7EXAMPLE"
export AWS_SECRET_ACCESS_KEY="wJalrXUtnFEMI/K7MDENG/bPxRfiCYzEXAMPLEKEY"
export AWS_DEFAULT_REGION="us-east-1"

# 2. ใช้ AWS Profile
aws configure --profile myprofile
export AWS_PROFILE=myprofile
# หรือใน provider:
provider "aws" {
  profile = "myprofile"
  region  = "us-east-1"
}

# 3. ตรวจสอบ credentials
aws sts get-caller-identity

# 4. ถ้าใช้ IAM Role
aws sts assume-role \
  --role-arn "arn:aws:iam::123456789012:role/TerraformRole" \
  --role-session-name "terraform"
```

### Provider Version Conflict

```bash
# Error:
Error: Failed to query available provider packages

Could not retrieve the list of available versions for provider
hashicorp/aws: locked provider registry.terraform.io/hashicorp/aws
5.0.0 does not match configured version constraint ~> 4.0.

# วิธีแก้:
# อัพเดต lock file
terraform init -upgrade

# หรือแก้ version constraint
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"  # แก้ version
    }
  }
}
```

### "Error: Unsupported attribute"

```bash
# Error:
Error: Unsupported attribute

on main.tf line 10, in resource "aws_instance" "web":
10:   nonexistent_attribute = "value"

An argument named "nonexistent_attribute" is not expected here.

# วิธีแก้:
# 1. ตรวจสอบ documentation ของ provider
# 2. ตรวจสอบ version ของ provider (attribute อาจมาจาก version ใหม่)
# 3. แก้ชื่อ attribute ให้ถูกต้อง
```

---

## Step 295: Common Error #4 - Resource Errors

### "Error: Invalid index"

```bash
# Error:
Error: Invalid index

on main.tf line 20:
20:   subnet_id = aws_subnet.public[3].id

The given key does not identify an element in this collection value:
the collection has no elements starting at index 3.

# วิธีแก้:
# 1. ตรวจสอบ count/for_each
resource "aws_subnet" "public" {
  count = 3  # indices: 0, 1, 2 (ไม่มี 3!)
}

# 2. ใช้ index ที่ valid
subnet_id = aws_subnet.public[2].id  # index สูงสุดคือ count-1

# 3. ใช้ length() เพื่อป้องกัน
subnet_id = aws_subnet.public[min(var.subnet_index, length(aws_subnet.public) - 1)].id
```

### State Inconsistency Errors

```bash
# Error:
Error: Provider produced inconsistent result after apply

When applying changes to aws_instance.web, provider
"registry.terraform.io/hashicorp/aws" produced an unexpected new
value: Root object was present, but now absent.

# สาเหตุ: Provider bug หรือ resource ถูกลบนอก Terraform

# วิธีแก้:
# 1. Refresh state
terraform apply -refresh-only

# 2. Import resource กลับมา (ถ้ายังมีอยู่ใน AWS)
terraform import aws_instance.web i-1234567890abcdef0

# 3. ลบออกจาก state แล้วสร้างใหม่
terraform state rm aws_instance.web
terraform apply
```

---

## Step 296: Timeout Errors

### Timeout ระหว่าง Resource Creation

```bash
# Error:
Error: timeout while waiting for state to become 'created'

# สาเหตุ: Resource ใช้เวลานานกว่า timeout default

# วิธีแก้:
resource "aws_eks_cluster" "main" {
  name     = "my-cluster"
  role_arn = aws_iam_role.eks.arn

  vpc_config {
    subnet_ids = aws_subnet.private[*].id
  }

  # เพิ่ม timeouts
  timeouts {
    create = "30m"  # default: 25m
    update = "60m"  # default: 60m
    delete = "15m"  # default: 15m
  }
}

resource "aws_db_instance" "main" {
  # ...
  
  timeouts {
    create = "40m"
    update = "80m"
    delete = "40m"
  }
}
```

### Connection Timeout

```bash
# Error: timeout - last error: dial tcp xxx.xxx.xxx.xxx:22: i/o timeout

# สาเหตุ: SSH connection timeout สำหรับ remote-exec

resource "aws_instance" "web" {
  # ...
  
  connection {
    type    = "ssh"
    host    = self.public_ip
    user    = "ubuntu"
    timeout = "5m"  # เพิ่ม timeout
    
    # ใช้ bastion ถ้าจำเป็น
    bastion_host = aws_instance.bastion.public_ip
    bastion_user = "ubuntu"
  }
  
  provisioner "remote-exec" {
    inline = ["echo connected"]
  }
}
```

---

## Step 297: terraform console - Expression Testing

### พื้นฐาน terraform console

```bash
# เริ่ม interactive console
terraform console

# ตัวอย่าง:
> "hello ${var.name}"
"hello world"

> length(["a", "b", "c"])
3

> toset(["a", "b", "a"])
toset(["a", "b"])

> [for i in range(5) : i * 2]
[0, 2, 4, 6, 8]

> exit หรือ Ctrl+D
```

### Test Expressions

```bash
# Test expressions ที่จะใช้ใน config
$ terraform console

# Test string operations
> upper("hello")
"HELLO"

> format("server-%03d", 5)
"server-005"

> replace("us-east-1", "-", "_")
"us_east_1"

# Test conditionals
> var.environment == "prod" ? "t3.large" : "t3.micro"
"t3.micro"

# Test lists/maps
> merge({a = 1}, {b = 2})
{
  "a" = 1
  "b" = 2
}

> flatten([[1, 2], [3, 4], [5]])
[1, 2, 3, 4, 5]

# Test cidrsubnet
> cidrsubnet("10.0.0.0/16", 8, 1)
"10.0.1.0/24"

> cidrsubnet("10.0.0.0/16", 8, 2)
"10.0.2.0/24"

# Test jsonencode
> jsonencode({name = "test", values = [1, 2, 3]})
"{\"name\":\"test\",\"values\":[1,2,3]}"

# Reference resource values (ถ้า state มีข้อมูล)
> aws_instance.web.public_ip
"52.1.2.3"

> [for i in aws_instance.web : i.id]
["i-111", "i-222", "i-333"]
```

### Debug Complex Expressions

```bash
# ในไฟล์ outputs.tf:
output "debug_cidr" {
  value = [
    for i in range(var.subnet_count) :
    cidrsubnet(var.vpc_cidr, 8, i)
  ]
}

# ใน console:
$ terraform console
> var.vpc_cidr
"10.0.0.0/16"

> [for i in range(3) : cidrsubnet("10.0.0.0/16", 8, i)]
["10.0.0.0/24", "10.0.1.0/24", "10.0.2.0/24"]
```

---

## Step 298: terraform graph - Dependency Visualization

### สร้าง Dependency Graph

```bash
# Generate DOT format graph
terraform graph > graph.dot

# Convert to PNG (ต้องการ graphviz)
terraform graph | dot -Tpng > graph.png

# Convert to SVG
terraform graph | dot -Tsvg > graph.svg

# ดู plan graph
terraform graph -type=plan | dot -Tpng > plan-graph.png

# ดู apply graph
terraform graph -type=apply | dot -Tpng > apply-graph.png

# ดู destroy graph
terraform graph -type=plan-destroy | dot -Tpng > destroy-graph.png
```

### ติดตั้ง Graphviz

```bash
# Ubuntu/Debian
sudo apt-get install -y graphviz

# macOS
brew install graphviz

# CentOS/RHEL
sudo yum install -y graphviz
```

### ใช้ Online Graph Viewer

```bash
# สร้าง DOT output
terraform graph > graph.dot

# Copy content ไปที่ https://dreampuf.github.io/GraphvizOnline/
cat graph.dot
```

### Filter Graph

```bash
# Filter เฉพาะ module
terraform graph | grep -v "provider\|meta"

# Simplify graph
terraform graph | \
  grep -v "provider\[\"registry" | \
  dot -Tpng > simplified-graph.png
```

---

## Step 299: Crash Log Analysis & Provider Debug

### Terraform Crash Logs

```bash
# ถ้า Terraform crash จะสร้าง crash.log
ls crash.log

# ตัวอย่าง crash.log:
# panic: runtime error: invalid memory address or nil pointer dereference
# [signal SIGSEGV: segmentation violation code=0x1 addr=0x0 pc=0x...]
# 
# goroutine 1 [running]:
# github.com/hashicorp/terraform/command.(*ApplyCommand).Run(...)
# ...
# 
# Terraform Version: 1.6.0
# Go Version: go1.21.5
```

### วิธีรายงาน Crash

```bash
# 1. บันทึก crash.log
cp crash.log crash-$(date +%Y%m%d-%H%M%S).log

# 2. เก็บ Terraform version
terraform version

# 3. เก็บ Provider versions
terraform providers

# 4. เก็บ debug log
TF_LOG=TRACE TF_LOG_PATH=debug.log terraform plan 2>&1

# 5. รายงานใน GitHub Issues ของ provider
```

### Provider Debug Mode

```bash
# AWS Provider debug
export TF_LOG=DEBUG
export TF_LOG_PROVIDER=DEBUG

# ดู AWS API calls
TF_LOG=DEBUG terraform plan 2>&1 | grep -E "Request\|Response\|HTTP"

# ตัวอย่าง AWS API debug output:
# 2024-01-15T10:30:00.000+0700 [DEBUG] provider.terraform-provider-aws: ...
# 2024-01-15T10:30:00.001+0700 [DEBUG] provider.terraform-provider-aws: 
#   AWS API Call: service=ec2, region=us-east-1, operation=DescribeInstances
# 2024-01-15T10:30:00.050+0700 [DEBUG] provider.terraform-provider-aws: 
#   Response: StatusCode=200, RequestID=abc123
```

---

## Step 300: Complete Troubleshooting Guide

### Troubleshooting Workflow

```
Problem Occurred
       |
       v
Check error message carefully
       |
       +---> Is it a config error?
       |         -> terraform validate
       |         -> Check syntax, types, references
       |
       +---> Is it a provider error?
       |         -> Check credentials
       |         -> Check API permissions
       |         -> Check provider version
       |
       +---> Is it a state error?
       |         -> terraform state list
       |         -> terraform show
       |         -> Check for drift
       |
       +---> Is it a network error?
       |         -> Check connectivity
       |         -> Check firewall/security groups
       |
       +---> Need more info?
                 -> TF_LOG=DEBUG
                 -> Check crash.log
```

### Quick Diagnosis Commands

```bash
#!/bin/bash
# tf-diagnose.sh
# Quick diagnosis script

echo "=== Terraform Diagnosis ==="
echo ""

echo "1. Terraform Version:"
terraform version
echo ""

echo "2. Current Workspace:"
terraform workspace show
echo ""

echo "3. Backend Configuration:"
terraform init -backend=false 2>&1 | head -20
echo ""

echo "4. Configuration Validation:"
terraform validate 2>&1
echo ""

echo "5. State Overview:"
terraform state list 2>&1 | head -20
echo ""

echo "6. Provider Status:"
terraform providers
echo ""

echo "7. AWS Credentials:"
aws sts get-caller-identity 2>&1
echo ""

echo "=== Diagnosis Complete ==="
```

### Error Message Reference

```bash
# ================================
# CONFIGURATION ERRORS
# ================================

# Error: Unsupported block type
# Cause: Block type ไม่ valid สำหรับ resource นั้น
# Fix: ตรวจสอบ documentation, ลบ block ที่ไม่ support

# Error: Missing required argument
# Cause: ขาด required field
# Fix: เพิ่ม required argument

# Error: Incorrect attribute value type
# Cause: ใส่ type ผิด (string แทน number)
# Fix: แก้ type ให้ถูกต้อง

# ================================
# STATE ERRORS
# ================================

# Error: Error acquiring the state lock
# Cause: มี concurrent operation หรือ lock ค้าง
# Fix: terraform force-unlock LOCK_ID

# Error: state data in S3 does not have the expected content
# Cause: State file corrupt หรือถูกแก้ไขนอก Terraform
# Fix: ตรวจสอบ S3 versioning, restore จาก backup

# ================================
# PROVIDER ERRORS
# ================================

# Error: NoCredentialProviders
# Cause: ไม่มี AWS credentials
# Fix: Set AWS env vars หรือ AWS profile

# Error: UnauthorizedOperation
# Cause: ไม่มี permission
# Fix: เพิ่ม IAM permissions

# Error: InvalidParameterValue
# Cause: ค่าที่ส่งไป AWS ไม่ valid
# Fix: ตรวจสอบ parameter values

# ================================
# TIMEOUT ERRORS
# ================================

# Error: timeout while waiting for state to become 'available'
# Cause: Resource ใช้เวลานาน
# Fix: เพิ่ม timeout ใน resource config

# ================================
# DEPENDENCY ERRORS
# ================================

# Error: DependencyViolation
# Cause: มี resource ที่ depend on resource ที่กำลังจะลบ
# Fix: ลบ dependent resources ก่อน, หรือ terraform destroy ตามลำดับ

# Error: ResourceInUse
# Cause: Resource ถูก reference จาก resource อื่น
# Fix: ลบ reference ก่อน แล้วค่อยลบ resource
```

### Debug Specific Scenarios

```bash
# Scenario 1: Plan แสดง unexpected changes
# อาจเกิดจาก:
# - Configuration drift
# - Provider version change
# - Default values เปลี่ยน

# ตรวจสอบ:
terraform show -json | jq '.values.root_module.resources[] | select(.address == "aws_instance.web")'
terraform state show aws_instance.web

# Refresh state
terraform apply -refresh-only

# ================================

# Scenario 2: Apply ล้มเหลวกลางคัน
# ตรวจสอบ:
terraform state list  # ดูว่า resources ไหน apply แล้ว

# ลอง apply ใหม่ (Terraform จะ skip resources ที่ apply แล้ว)
terraform apply

# ================================

# Scenario 3: Import ทำแล้วแต่ plan ยังเห็น changes
# เหตุผล: config ไม่ match state

# ตรวจสอบ:
terraform state show <resource>  # ดูค่าใน state
# เปรียบเทียบกับ config แล้วแก้ให้ match

# ================================

# Scenario 4: "Cycle" dependency error
# Error: Cycle: A, B, A

# ตรวจสอบ:
terraform graph | dot -Tpng > graph.png
# ดู graph หา cycle

# แก้ไขโดย:
# - ลบ circular reference
# - ใช้ depends_on แทน implicit reference
# - Restructure resources

# ================================

# Scenario 5: for_each "values must be known"
resource "aws_instance" "web" {
  for_each = toset(data.aws_availability_zones.available.names)  # ❌
}

# Fix:
locals {
  # ใช้ hardcoded หรือ known values
  azs = ["us-east-1a", "us-east-1b", "us-east-1c"]
}

resource "aws_instance" "web" {
  for_each = toset(local.azs)  # ✅ known at plan time
}
```

---

## Environment Variable Reference

```bash
# Logging
TF_LOG=TRACE|DEBUG|INFO|WARN|ERROR  # Log level
TF_LOG_PATH=/path/to/logfile         # Log file path
TF_LOG_CORE=LEVEL                    # Core log level
TF_LOG_PROVIDER=LEVEL                # Provider log level

# Variables
TF_VAR_name=value          # Pass variables
TF_CLI_ARGS="-no-color"    # Add CLI args globally
TF_CLI_ARGS_plan="-out=tfplan"  # Args for specific command
TF_CLI_ARGS_apply="-auto-approve"

# Data directories
TF_DATA_DIR=/path/to/dir   # .terraform directory location
TF_PLUGIN_DIR=/path/to/plugins  # Custom plugin directory

# Authentication
AWS_ACCESS_KEY_ID
AWS_SECRET_ACCESS_KEY
AWS_SESSION_TOKEN
AWS_PROFILE
AWS_DEFAULT_REGION

# Backend
TF_BACKEND_CONFIG=/path/to/backend.hcl  # Custom backend config

# Misc
TF_INPUT=false             # ปิด interactive input
TF_IN_AUTOMATION=1         # Signal CI/CD mode
```

---

## Quick Reference: Debug Commands

```bash
# ===================================
# ตรวจสอบ configuration
# ===================================
terraform validate            # Validate syntax
terraform fmt -check          # Check formatting
terraform providers           # Show providers

# ===================================
# ตรวจสอบ state
# ===================================
terraform state list          # List resources
terraform state show ADDR     # Show resource
terraform show                # Show all state
terraform output              # Show outputs

# ===================================
# Debug
# ===================================
TF_LOG=DEBUG terraform plan   # Debug logging
terraform console              # Interactive expression testing
terraform graph | dot -Tpng > g.png  # Visualize dependencies

# ===================================
# Fix common issues
# ===================================
terraform init -reconfigure   # Reset backend
terraform init -upgrade       # Upgrade providers
terraform apply -refresh-only  # Sync state with reality
terraform force-unlock LOCK_ID  # Remove stale lock
```

---

## แบบฝึกหัด (Exercises)

### Exercise 1: Debug Logging

1. Enable TF_LOG=DEBUG
2. รัน `terraform plan`
3. บันทึก log ไปไฟล์
4. หา HTTP API calls ที่ถูก made
5. หา timing information

### Exercise 2: Fix Common Errors

แก้ไข configuration ที่มี errors ต่อไปนี้:
1. Syntax error
2. Missing required argument
3. Type mismatch
4. Undeclared reference

### Exercise 3: Use terraform console

1. ทดสอบ cidrsubnet function ด้วย console
2. Debug complex for expression
3. ตรวจสอบค่าของ resource attributes ใน state

---

## Checklist

- [ ] รู้วิธีใช้ TF_LOG levels ต่างๆ
- [ ] สามารถ save logs ไปไฟล์ได้
- [ ] รู้วิธีแก้ common errors แต่ละประเภท
- [ ] สามารถแก้ state lock errors ได้
- [ ] รู้วิธีใช้ terraform console สำหรับ expression testing
- [ ] สามารถ generate และอ่าน dependency graph ได้
- [ ] เข้าใจ crash log และวิธีรายงาน
- [ ] รู้ environment variables ที่เกี่ยวข้องกับ debugging
- [ ] มี troubleshooting workflow ที่ชัดเจน
