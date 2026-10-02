# Part 27: Terraform CLI - destroy & import (ขั้นตอนที่ 261-270)

## ภาพรวม (Overview)

`terraform destroy` สำหรับลบ infrastructure และ `terraform import` สำหรับนำ existing resources เข้าสู่ Terraform management เป็นสองคำสั่งสำคัญที่ต้องใช้อย่างระมัดระวัง

---

## Step 261: terraform destroy - Full Destroy

### Syntax พื้นฐาน

```bash
# Destroy ทุกอย่าง (interactive - ถาม confirm)
terraform destroy

# ตัวอย่าง output:
$ terraform destroy

aws_instance.web: Refreshing state... [id=i-1234567890abcdef0]
aws_security_group.web: Refreshing state... [id=sg-12345678]
aws_vpc.main: Refreshing state... [id=vpc-12345678]

Terraform used the selected providers to generate the following execution plan.
Resource actions are indicated with the following symbols:
  - destroy

Terraform will perform the following actions:

  # aws_instance.web will be destroyed
  - resource "aws_instance" "web" {
      - ami           = "ami-xxx" -> null
      - id            = "i-1234567890abcdef0" -> null
      - instance_type = "t3.micro" -> null
    }

  # aws_security_group.web will be destroyed
  - resource "aws_security_group" "web" {
      - id   = "sg-12345678" -> null
      - name = "web-sg" -> null
    }

Plan: 0 to add, 0 to change, 3 to destroy.

Do you really want to destroy all resources?
  Terraform will destroy all your managed infrastructure, as shown above.
  There is no undo. Only 'yes' will be accepted to confirm.

  Enter a value: yes

aws_instance.web: Destroying... [id=i-1234567890abcdef0]
aws_instance.web: Still destroying... [id=i-1234567890abcdef0, 10s elapsed]
aws_instance.web: Destruction complete after 32s
aws_security_group.web: Destroying... [id=sg-12345678]
aws_security_group.web: Destruction complete after 3s
aws_vpc.main: Destroying... [id=vpc-12345678]
aws_vpc.main: Destruction complete after 5s

Destroy complete! Resources: 3 destroyed.
```

---

## Step 262: terraform destroy - Flags

### -auto-approve

```bash
# ไม่ถาม confirm
terraform destroy -auto-approve

# ⚠️ ระวังมาก! destroy production โดยไม่มี confirm
# ✅ ใช้ใน CI/CD หรือ test environments เท่านั้น
```

### -target Flag

```bash
# Destroy เฉพาะ resources ที่กำหนด
terraform destroy -target=aws_instance.web
terraform destroy -target=aws_instance.web -target=aws_security_group.web

# ⚠️ อาจทิ้งสิ่งที่ depend on resource นี้ไว้ใน state
# ต้อง careful เรื่อง dependencies
```

### -var และ -var-file

```bash
# Pass variables
terraform destroy -var="environment=dev" -auto-approve
terraform destroy -var-file="dev.tfvars" -auto-approve
```

### destroy vs apply -destroy

```bash
# สองวิธีเหมือนกัน:
terraform destroy

# เหมือนกับ:
terraform apply -destroy

# ข้อดีของ apply -destroy:
# สามารถ -out plan ก่อนได้
terraform plan -destroy -out=destroy.tfplan
# Review plan...
terraform apply destroy.tfplan
```

---

## Step 263: Order of Destruction

### Terraform Destroys ตาม Reverse Dependency Order

```hcl
# Resources ที่ create ใน order นี้:
# 1. aws_vpc.main
# 2. aws_subnet.private[*]
# 3. aws_security_group.app
# 4. aws_instance.app

# Terraform จะ destroy ใน reverse order:
# 1. aws_instance.app (ก่อน เพราะ depend on subnet และ sg)
# 2. aws_security_group.app
# 3. aws_subnet.private[*]
# 4. aws_vpc.main (สุดท้าย เพราะเป็น dependency ของทุกอย่าง)
```

### ตัวอย่าง Destruction Order

```bash
$ terraform destroy -auto-approve

aws_instance.web: Destroying... [id=i-xxx]
aws_instance.app: Destroying... [id=i-yyy]
aws_instance.web: Destruction complete after 32s
aws_instance.app: Destruction complete after 35s
aws_security_group.web: Destroying... [id=sg-xxx]
aws_security_group.app: Destroying... [id=sg-yyy]
aws_security_group.web: Destruction complete after 3s
aws_security_group.app: Destruction complete after 3s
aws_db_subnet_group.main: Destroying... [id=main-db-subnet]
aws_db_subnet_group.main: Destruction complete after 5s
aws_subnet.private[0]: Destroying... [id=subnet-xxx]
aws_subnet.private[1]: Destroying... [id=subnet-yyy]
aws_subnet.private[0]: Destruction complete after 5s
aws_subnet.private[1]: Destruction complete after 5s
aws_vpc.main: Destroying... [id=vpc-xxx]
aws_vpc.main: Destruction complete after 5s
```

---

## Step 264: Pre-Destroy Checks & Protection

### prevent_destroy Lifecycle

```hcl
resource "aws_db_instance" "production" {
  identifier = "prod-db"
  engine     = "postgres"
  # ...

  lifecycle {
    prevent_destroy = true
  }
}

# $ terraform destroy
# Error: Instance cannot be destroyed
#
# Resource aws_db_instance.production has lifecycle.prevent_destroy set,
# but the plan calls for this resource to be destroyed.
# To avoid this error and continue with the destroy plan,
# either disable the prevent_destroy flag or remove this resource from
# the configuration.
```

### deletion_protection สำหรับ AWS Resources

```hcl
resource "aws_db_instance" "production" {
  identifier          = "prod-db"
  deletion_protection = true  # AWS-level protection
  
  lifecycle {
    prevent_destroy = true  # Terraform-level protection
  }
}

resource "aws_rds_cluster" "production" {
  cluster_identifier  = "prod-cluster"
  deletion_protection = true
}

# ต้อง disable ก่อน destroy:
# 1. แก้ deletion_protection = false
# 2. terraform apply
# 3. ลบ lifecycle { prevent_destroy = true } หรือ set false
# 4. terraform destroy
```

### Pre-Destroy Checks Script

```bash
#!/bin/bash
# pre-destroy-check.sh

ENV=${1:-dev}

if [ "$ENV" = "prod" ]; then
  echo "⚠️  WARNING: You are about to destroy PRODUCTION infrastructure!"
  echo ""
  echo "Please confirm you want to destroy PRODUCTION by typing: destroy production"
  read -r CONFIRM
  
  if [ "$CONFIRM" != "destroy production" ]; then
    echo "❌ Confirmation failed. Aborting."
    exit 1
  fi
  
  echo "Sending notification to team..."
  # Send Slack/PagerDuty notification
fi

echo "Running destroy..."
terraform destroy -var-file="environments/${ENV}/terraform.tfvars"
```

---

## Step 265: When Destroy Fails

### ปัญหาที่พบบ่อยเมื่อ Destroy ล้มเหลว

```bash
# Error 1: Dependency error
# Error: deleting VPC (vpc-xxx): DependencyViolation: The vpc 'vpc-xxx' 
# has dependencies and cannot be deleted.

# แก้ไข:
# Terraform ควร handle dependencies อัตโนมัติ
# แต่ถ้าล้มเหลว: ต้องลบ resources ที่ depend on VPC ก่อน manually
# หรือใช้ -target เพื่อลบ specific resources

# Error 2: Still in use
# Error: error deleting Security Group (sg-xxx): DependencyViolation
# The security group is in use

# แก้ไข:
# ตรวจสอบว่ามี resource ไหนที่ใช้ SG นั้นอยู่แต่ไม่อยู่ใน state
# ลบ manually จาก AWS Console แล้ว terraform refresh

# Error 3: S3 bucket not empty
# Error: error deleting S3 Bucket (my-bucket): BucketNotEmpty

# แก้ไข:
resource "aws_s3_bucket" "data" {
  bucket = "my-data-bucket"

  # เพิ่ม force_destroy เพื่อลบ bucket แม้ไม่ว่าง
  force_destroy = true
}
```

### Handle Stuck Destroy

```bash
# ถ้า destroy ค้าง หรือ fail:

# 1. ดู state ปัจจุบัน
terraform state list

# 2. Destroy ทีละอัน
terraform destroy -target=aws_instance.web
terraform destroy -target=aws_security_group.web

# 3. ถ้า resource ถูกลบแล้วแต่ยังอยู่ใน state
terraform state rm aws_instance.already_deleted

# 4. Force unlock ถ้า state locked
terraform force-unlock LOCK_ID
```

---

## Step 266: terraform import - Import Workflow

### ทำไมต้องใช้ import?

```
Use cases สำหรับ terraform import:
1. Existing infrastructure ที่สร้าง manually
2. Migration จาก CLI/Console ไป Terraform
3. Resource ที่สร้างโดย process อื่น
4. Recovery หลังจาก state loss
```

### วิธีที่ 1: import block (Terraform 1.5+ - Declarative)

```hcl
# main.tf
# Step 1: เขียน resource configuration
resource "aws_instance" "imported" {
  # จะ fill details หลัง import
}

# Step 2: เพิ่ม import block
import {
  to = aws_instance.imported
  id = "i-1234567890abcdef0"  # AWS Instance ID
}

# Step 3: รัน plan
# terraform plan จะแสดง import + potential changes

# Step 4: Apply
# terraform apply
```

### วิธีที่ 2: terraform import Command (Old Way)

```bash
# Syntax:
terraform import <resource_address> <resource_id>

# ตัวอย่าง:
terraform import aws_instance.web i-1234567890abcdef0
terraform import aws_s3_bucket.data my-existing-bucket
terraform import aws_vpc.main vpc-12345678
terraform import aws_security_group.web sg-12345678
terraform import aws_iam_role.app my-existing-role

# ตัวอย่าง output:
# aws_instance.web: Importing from ID "i-1234567890abcdef0"...
# aws_instance.web: Import prepared!
#   Prepared aws_instance for import
# aws_instance.web: Refreshing state... [id=i-1234567890abcdef0]
# 
# Import successful!
# 
# The resources that were imported are shown above. These resources are now in
# your Terraform state and will henceforth be managed by Terraform.
```

---

## Step 267: Import Workflow - Complete Example

### Import Existing S3 Bucket

```bash
# ขั้นตอนที่ 1: ตรวจสอบ bucket ที่มีอยู่
aws s3api get-bucket-location --bucket my-existing-bucket
aws s3api get-bucket-versioning --bucket my-existing-bucket
aws s3api get-bucket-tags --bucket my-existing-bucket
```

```hcl
# ขั้นตอนที่ 2: เขียน configuration (เดาก่อน)
# main.tf
resource "aws_s3_bucket" "existing" {
  bucket = "my-existing-bucket"
}
```

```bash
# ขั้นตอนที่ 3: Import
terraform import aws_s3_bucket.existing my-existing-bucket

# Output:
# aws_s3_bucket.existing: Importing from ID "my-existing-bucket"...
# aws_s3_bucket.existing: Import prepared!
# aws_s3_bucket.existing: Refreshing state... [id=my-existing-bucket]
# Import successful!
```

```bash
# ขั้นตอนที่ 4: ดู state เพื่อรู้ configuration
terraform state show aws_s3_bucket.existing

# Output (ตัวอย่าง):
# # aws_s3_bucket.existing:
# resource "aws_s3_bucket" "existing" {
#     bucket                      = "my-existing-bucket"
#     bucket_domain_name          = "my-existing-bucket.s3.amazonaws.com"
#     hosted_zone_id              = "Z3AQBSTGFYJSTF"
#     id                          = "my-existing-bucket"
#     object_lock_enabled         = false
#     region                      = "us-east-1"
#     request_payer               = "BucketOwner"
#     tags                        = {
#         "Environment" = "production"
#         "Team"        = "platform"
#     }
# }
```

```hcl
# ขั้นตอนที่ 5: อัพเดต configuration ให้ตรงกับ state
resource "aws_s3_bucket" "existing" {
  bucket = "my-existing-bucket"

  tags = {
    Environment = "production"
    Team        = "platform"
  }
}
```

```bash
# ขั้นตอนที่ 6: Plan เพื่อดูว่ามี drift หรือไม่
terraform plan
# ควร show "No changes" ถ้าทำถูก
```

### Import Existing VPC

```hcl
# Step 1: เขียน resource
resource "aws_vpc" "existing" {
  cidr_block = "10.0.0.0/16"
}
```

```bash
# Step 2: Import
terraform import aws_vpc.existing vpc-12345678

# Step 3: ดู state
terraform state show aws_vpc.existing
```

```hcl
# Step 4: อัพเดต config ให้ match
resource "aws_vpc" "existing" {
  cidr_block           = "10.0.0.0/16"
  enable_dns_hostnames = true
  enable_dns_support   = true

  tags = {
    Name        = "production-vpc"
    Environment = "production"
  }
}
```

---

## Step 268: Generating Configuration with terraform plan

### -generate-config-out Flag (Terraform 1.5+)

```bash
# Generate configuration file จาก import block
# ไม่ต้องเขียน resource config เอง!

# Step 1: เพิ่มแค่ import block (ไม่ต้องมี resource)
```

```hcl
# imports.tf
import {
  to = aws_instance.legacy
  id = "i-1234567890abcdef0"
}

import {
  to = aws_security_group.legacy
  id = "sg-12345678"
}
```

```bash
# Step 2: Generate configuration
terraform plan -generate-config-out=generated.tf

# Step 3: Terraform สร้าง generated.tf พร้อม resource config
cat generated.tf
```

```hcl
# generated.tf (auto-generated - ตัวอย่าง)
resource "aws_instance" "legacy" {
  ami                         = "ami-0c55b159cbfafe1f0"
  instance_type               = "t3.micro"
  key_name                    = "my-key"
  monitoring                  = false
  subnet_id                   = "subnet-12345678"
  vpc_security_group_ids      = ["sg-12345678"]
  
  root_block_device {
    delete_on_termination = true
    encrypted             = false
    volume_size           = 8
    volume_type           = "gp2"
  }

  tags = {
    Name = "legacy-server"
  }
}
```

```bash
# Step 4: Review generated config และแก้ไขตามต้องการ
# Step 5: Apply
terraform apply
```

---

## Step 269: Import Limitations & Complex Resources

### Import Limitations

```bash
# ❌ ไม่สามารถ import:
# - Resources ที่มีค่า sensitive ที่ Terraform ไม่ track
#   (เช่น password ที่ hash แล้ว)
# - Resources ที่ provider ไม่ support import
# - Sub-resources บางประเภท

# ตรวจสอบว่า resource support import:
# ดูใน provider documentation
# Ctrl+F "Import" ใน resource page

# ตัวอย่าง: AWS provider
# https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/instance#import
```

### Import Multiple Resources (Bulk Import)

```hcl
# Terraform 1.5+: หลาย import blocks ใน file เดียว
import {
  to = aws_subnet.public[0]
  id = "subnet-11111111"
}

import {
  to = aws_subnet.public[1]
  id = "subnet-22222222"
}

import {
  to = aws_subnet.public[2]
  id = "subnet-33333333"
}

import {
  to = aws_subnet.private[0]
  id = "subnet-44444444"
}

import {
  to = aws_subnet.private[1]
  id = "subnet-55555555"
}
```

```bash
# Apply all imports at once
terraform apply
```

### Import ด้วย for_each

```hcl
# สำหรับ resources ที่ใช้ for_each
locals {
  subnet_imports = {
    "public-1a" = "subnet-11111111"
    "public-1b" = "subnet-22222222"
    "private-1a" = "subnet-33333333"
    "private-1b" = "subnet-44444444"
  }
}

import {
  for_each = local.subnet_imports
  to       = aws_subnet.all[each.key]
  id       = each.value
}

resource "aws_subnet" "all" {
  for_each = var.subnet_configs
  # ...
}
```

---

## Step 270: Real-World Import Examples

### Import Existing RDS Instance

```bash
# ตรวจสอบ RDS ที่มีอยู่
aws rds describe-db-instances --db-instance-identifier my-production-db

# Import ID สำหรับ RDS = DB identifier
terraform import aws_db_instance.production my-production-db
```

```hcl
# Configuration หลัง import
resource "aws_db_instance" "production" {
  identifier             = "my-production-db"
  engine                 = "postgres"
  engine_version         = "14.7"
  instance_class         = "db.r5.large"
  allocated_storage      = 100
  storage_type           = "gp3"
  storage_encrypted      = true
  
  db_name  = "appdb"
  username = "admin"
  # password ไม่ต้องใส่ใน config หลัง import
  # ใช้ ignore_changes เพื่อไม่ให้ Terraform reset password
  
  multi_az               = true
  deletion_protection    = true
  backup_retention_period = 7
  
  vpc_security_group_ids = ["sg-12345678"]
  db_subnet_group_name   = "production-subnet-group"

  lifecycle {
    ignore_changes = [password]
    prevent_destroy = true
  }

  tags = {
    Environment = "production"
  }
}
```

### Import IAM Resources

```bash
# IAM User
terraform import aws_iam_user.alice alice

# IAM Role
terraform import aws_iam_role.app_role my-app-role

# IAM Policy (ใช้ ARN)
terraform import aws_iam_policy.custom arn:aws:iam::123456789012:policy/MyCustomPolicy

# IAM Role Policy Attachment
# Format: <role_name>/<policy_arn>
terraform import aws_iam_role_policy_attachment.app_policy \
  "my-app-role/arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess"

# IAM Group Membership
# Format: <group_name>/<user1>,<user2>,...
terraform import aws_iam_group_membership.team \
  "developers/alice,bob,charlie"
```

### Import EKS Resources

```bash
# EKS Cluster
terraform import aws_eks_cluster.main my-cluster

# EKS Node Group
# Format: <cluster_name>:<node_group_name>
terraform import aws_eks_node_group.workers my-cluster:worker-group-1

# EKS Addon
# Format: <cluster_name>:<addon_name>
terraform import aws_eks_addon.coredns my-cluster:coredns
```

### Import Route53 Resources

```bash
# Hosted Zone
# Format: /hostedzone/<zone_id>
terraform import aws_route53_zone.main /hostedzone/Z1234567890ABC

# Route53 Record
# Format: <zone_id>_<record_name>_<record_type>
terraform import aws_route53_record.www \
  "Z1234567890ABC_www.example.com_A"
```

### Script สำหรับ Bulk Import

```bash
#!/bin/bash
# bulk-import.sh
# Import หลาย resources จาก CSV file

# Format ของ CSV: resource_type,resource_name,resource_id
# aws_s3_bucket,data_bucket,my-data-bucket
# aws_iam_role,app_role,my-app-role
# aws_instance,web_server,i-1234567890abcdef0

while IFS=',' read -r resource_type resource_name resource_id; do
  # Skip comments and empty lines
  [[ "$resource_type" =~ ^#.*$ ]] && continue
  [[ -z "$resource_type" ]] && continue
  
  echo "Importing: ${resource_type}.${resource_name} (ID: ${resource_id})"
  
  terraform import \
    "${resource_type}.${resource_name}" \
    "${resource_id}" || {
    echo "❌ Failed to import ${resource_type}.${resource_name}"
    # Continue แม้ fail
  }
  
  echo "✅ Done: ${resource_type}.${resource_name}"
done < resources_to_import.csv

echo ""
echo "Import complete. Running plan to check for drift..."
terraform plan
```

---

## Import Best Practices

```bash
# ✅ DO:
# 1. Always backup state ก่อน import
terraform state pull > state-backup-$(date +%Y%m%d).json

# 2. Import ทีละ resource แล้ว verify
terraform import aws_instance.web i-xxx
terraform plan  # ควรเห็น changes น้อยที่สุด

# 3. ใช้ -generate-config-out สำหรับ complex resources
terraform plan -generate-config-out=generated.tf

# 4. Test ใน dev/staging ก่อน import production

# 5. Document ทุก import ที่ทำ

# ❌ DON'T:
# - Import แล้วไม่ verify plan
# - Import โดยไม่ backup state ก่อน
# - Import resources ที่ provider ไม่ support
# - แก้ไข state file โดยตรงแทน import
```

### Import vs Adoption Pattern

```bash
# Pattern: "Import then Manage"
# 1. Import resource
# 2. ดู state
# 3. เขียน config ให้ match state
# 4. Plan - ควร "No changes"
# 5. เพิ่ม tags/settings ที่ต้องการ
# 6. Plan again - ดู changes
# 7. Apply
```

---

## Command Reference Summary

```bash
# === terraform destroy ===
terraform destroy                         # Interactive destroy
terraform destroy -auto-approve           # No confirmation
terraform destroy -target=RESOURCE        # Destroy specific resource
terraform destroy -var="key=value"        # Pass variables
terraform destroy -var-file="file.tfvars" # Use var file
terraform destroy -compact-warnings       # Compact output
terraform plan -destroy -out=plan         # Save destroy plan
terraform apply plan                      # Apply destroy plan

# === terraform import ===
terraform import RESOURCE_ADDR IMPORT_ID  # Import resource
terraform import -config=DIR RESOURCE ID  # From specific dir
# Import block in .tf file (TF 1.5+)
# terraform plan -generate-config-out=generated.tf
```

---

## แบบฝึกหัด (Exercises)

### Exercise 1: Import Existing Resource

1. สร้าง S3 bucket ด้วย AWS CLI
2. เขียน Terraform resource config
3. Import bucket เข้า Terraform
4. Verify plan แสดง "No changes"

### Exercise 2: Import Complex Infrastructure

1. สร้าง VPC, Subnets, Security Groups ด้วย AWS Console
2. Import ทั้งหมดเข้า Terraform
3. สร้าง complete configuration ที่ match existing infra

### Exercise 3: Destroy Safely

1. สร้าง infrastructure ด้วย count=3
2. ลบ 1 instance ด้วย `-target`
3. Verify state ที่เหลือถูกต้อง

---

## Checklist

- [ ] รู้วิธีใช้ terraform destroy อย่างปลอดภัย
- [ ] เข้าใจ destroy order ตาม dependencies
- [ ] รู้วิธีป้องกัน resources จาก accidental destroy
- [ ] เข้าใจ import workflow ทั้ง old และ new style
- [ ] สามารถ generate config จาก import ได้
- [ ] รู้ import ID format ของ common AWS resources
- [ ] เข้าใจ limitations ของ import
- [ ] สามารถทำ bulk import ได้
