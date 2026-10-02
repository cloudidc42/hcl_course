# Part 28: Terraform State Commands (ขั้นตอนที่ 271-280)

## ภาพรวม (Overview)

Terraform state commands เป็นเครื่องมือที่ช่วยจัดการ state โดยตรง ต้องใช้ด้วยความระมัดระวัง เพราะการแก้ไข state ที่ผิดพลาดอาจทำให้ infrastructure ไม่ sync กัน

---

## Step 271: terraform state list

### Syntax และตัวอย่าง

```bash
# แสดงรายการ resources ทั้งหมดใน state
terraform state list

# ตัวอย่าง output:
data.aws_ami.ubuntu
data.aws_availability_zones.available
data.aws_caller_identity.current
aws_db_instance.production
aws_instance.web[0]
aws_instance.web[1]
aws_instance.web[2]
aws_security_group.alb
aws_security_group.web
aws_vpc.main
module.networking.aws_subnet.private[0]
module.networking.aws_subnet.private[1]
module.networking.aws_subnet.public[0]
module.networking.aws_subnet.public[1]
module.rds.aws_db_subnet_group.main
module.rds.aws_db_instance.main
```

### Filter ด้วย Pattern

```bash
# Filter ด้วย address pattern (glob)
terraform state list 'aws_instance.*'
# aws_instance.web[0]
# aws_instance.web[1]
# aws_instance.web[2]
# aws_instance.bastion

terraform state list 'module.networking.*'
# module.networking.aws_subnet.private[0]
# module.networking.aws_subnet.private[1]
# module.networking.aws_subnet.public[0]

terraform state list 'aws_security_group*'
# aws_security_group.alb
# aws_security_group.web
# aws_security_group.db

# Filter ด้วย -id (resource ID)
terraform state list -id=i-1234567890abcdef0
# aws_instance.web[0]

terraform state list -id=vpc-12345678
# aws_vpc.main
```

### ใช้ state list ใน Scripts

```bash
#!/bin/bash
# count resources by type

terraform state list | \
  awk -F'.' '{print $1}' | \
  sort | uniq -c | sort -rn

# Output:
#   3 aws_instance
#   2 aws_security_group
#   2 module
#   1 aws_vpc
#   1 data
```

---

## Step 272: terraform state show

### ดู Details ของ Resource

```bash
# แสดง attributes ทั้งหมดของ resource
terraform state show aws_instance.web

# ตัวอย่าง output:
# aws_instance.web:
resource "aws_instance" "web" {
    ami                                  = "ami-0c55b159cbfafe1f0"
    arn                                  = "arn:aws:ec2:us-east-1:123456789012:instance/i-1234567890abcdef0"
    associate_public_ip_address          = true
    availability_zone                    = "us-east-1a"
    cpu_core_count                       = 1
    cpu_threads_per_core                 = 1
    disable_api_stop                     = false
    disable_api_termination              = false
    ebs_optimized                        = false
    get_password_data                    = false
    hibernation                          = false
    id                                   = "i-1234567890abcdef0"
    instance_initiated_shutdown_behavior = "stop"
    instance_state                       = "running"
    instance_type                        = "t3.micro"
    ipv6_address_count                   = 0
    ipv6_addresses                       = []
    monitoring                           = false
    private_dns                          = "ip-10-0-1-100.us-east-1.compute.internal"
    private_ip                           = "10.0.1.100"
    public_dns                           = "ec2-52-1-2-3.compute-1.amazonaws.com"
    public_ip                            = "52.1.2.3"
    secondary_private_ips                = []
    security_groups                      = []
    source_dest_check                    = true
    subnet_id                            = "subnet-12345678"
    tags                                 = {
        "Environment" = "production"
        "Name"        = "web-server"
    }
    tags_all                             = {
        "Environment" = "production"
        "ManagedBy"   = "terraform"
        "Name"        = "web-server"
    }
    vpc_security_group_ids               = [
        "sg-12345678",
    ]

    capacity_reservation_specification {
        capacity_reservation_preference = "open"
    }

    credit_specification {
        cpu_credits = "standard"
    }

    enclave_options {
        enabled = false
    }

    maintenance_options {
        auto_recovery = "default"
    }

    metadata_options {
        http_endpoint               = "enabled"
        http_put_response_hop_limit = 1
        http_tokens                 = "optional"
        instance_metadata_tags      = "disabled"
    }

    private_dns_name_options {
        enable_resource_name_dns_a_record    = false
        enable_resource_name_dns_aaaa_record = false
        hostname_type                        = "ip-name"
    }

    root_block_device {
        delete_on_termination = true
        device_name           = "/dev/xvda"
        encrypted             = false
        iops                  = 100
        tags                  = {}
        throughput            = 0
        volume_id             = "vol-12345678"
        volume_size           = 8
        volume_type           = "gp2"
    }
}
```

### แสดง Resources ใน Module

```bash
# Resource ที่ใช้ count
terraform state show 'aws_instance.web[0]'
terraform state show 'aws_instance.web[1]'

# Resource ที่ใช้ for_each
terraform state show 'aws_instance.servers["web"]'
terraform state show 'aws_instance.servers["api"]'

# Resource ใน module
terraform state show 'module.networking.aws_vpc.main'
terraform state show 'module.rds.aws_db_instance.main'
```

---

## Step 273: terraform state mv - Moving/Renaming Resources

### Rename Resource

```bash
# Rename โดยไม่ destroy
# OLD: aws_instance.web -> NEW: aws_instance.web_server
terraform state mv aws_instance.web aws_instance.web_server

# ⚠️ ต้องอัพเดต code ด้วย!
# ถ้า code ยังใช้ aws_instance.web อยู่ -> plan จะ show:
# - destroy aws_instance.web (ไม่มีใน state)
# + create aws_instance.web (ใหม่)
```

### Move Resource ระหว่าง Files

```bash
# ถ้าย้าย resource config จาก main.tf ไป web.tf
# ชื่อ resource ยังเหมือนเดิม -> ไม่ต้อง state mv
# ถ้าเปลี่ยนชื่อ -> ต้อง state mv
```

### Move Resource เข้า Module

```bash
# ย้าย resource จาก root module ไป submodule

# เดิม: aws_instance.web (ใน root)
# ใหม่: module.web.aws_instance.main (ใน module)

terraform state mv \
  aws_instance.web \
  module.web.aws_instance.main

# อัพเดต code:
# ลบ resource "aws_instance" "web" จาก root
# เพิ่ม module "web" { ... } แทน
# ใน modules/web/main.tf: สร้าง resource "aws_instance" "main"
```

### Move Resource ออกจาก Module

```bash
# ย้าย resource จาก module มา root

# เดิม: module.networking.aws_vpc.main
# ใหม่: aws_vpc.main

terraform state mv \
  'module.networking.aws_vpc.main' \
  aws_vpc.main
```

### Move ใน for_each

```bash
# Rename key ใน for_each

# เดิม: aws_instance.servers["old_key"]
# ใหม่: aws_instance.servers["new_key"]

terraform state mv \
  'aws_instance.servers["old_key"]' \
  'aws_instance.servers["new_key"]'
```

### Move Resource ระหว่าง State Files

```bash
# ต้องการย้าย resource จาก state A ไป state B

# Step 1: Move ออกจาก source state (pull state)
cd /path/to/source-project
terraform state mv \
  -state-out=/tmp/resource.tfstate \
  aws_instance.web \
  aws_instance.web

# Step 2: Push ไปยัง destination state
cd /path/to/dest-project
terraform state mv \
  -state=/tmp/resource.tfstate \
  aws_instance.web \
  aws_instance.migrated_web
```

---

## Step 274: terraform state rm - Removing from State

### ลบ Resource ออกจาก State (ไม่ Destroy!)

```bash
# ลบออกจาก state เท่านั้น - resource ยังอยู่ใน cloud
terraform state rm aws_instance.web

# ตัวอย่าง output:
# Removed aws_instance.web
# Successfully removed 1 resource instance(s).

# ⚠️ หลังจาก rm:
# - Resource ยังอยู่ใน AWS
# - Terraform ไม่รู้จัก resource นี้อีกต่อไป
# - terraform plan จะ show "create" ถ้ายังมีใน config
# - ถ้าลบออกจาก config ด้วย -> Terraform จะไม่ touch resource นั้น
```

### Use Cases สำหรับ state rm

```bash
# Use Case 1: ย้าย resource ออกไปจัดการด้วยวิธีอื่น
# เช่น จาก Terraform ไปจัดการ manually
terraform state rm aws_s3_bucket.old_bucket

# Use Case 2: Resource ที่ถูกลบนอก Terraform แล้ว
# เพื่อให้ state sync กัน
terraform state rm aws_instance.already_deleted

# Use Case 3: Clean up orphaned resources ใน state
# Resources ที่ถูกสร้างโดย process อื่นและถูก import โดยผิดพลาด
terraform state rm aws_resource.mistake

# Use Case 4: Fix state inconsistency
terraform state rm 'aws_subnet.private[2]'  # ลบออกก่อน recreate
```

### ลบหลาย Resources ในครั้งเดียว

```bash
# ลบทีละอัน
terraform state rm aws_instance.web
terraform state rm aws_security_group.web

# ลบหลายอันพร้อมกัน
terraform state rm aws_instance.web aws_security_group.web aws_vpc.old

# ลบทุก resource ใน module
terraform state rm 'module.old_module'
# จะลบทุก resource ภายใต้ module นั้น
```

---

## Step 275: terraform state pull & push

### terraform state pull

```bash
# Download current state จาก remote backend
terraform state pull

# Save ไปไฟล์
terraform state pull > current-state.json

# ดู content
terraform state pull | jq '.resources[].type' | sort | uniq -c

# ดู specific resource
terraform state pull | jq '.resources[] | select(.type == "aws_instance")'

# ดู outputs
terraform state pull | jq '.outputs'
```

### terraform state push

```bash
# ⚠️ อันตราย! ใช้ด้วยความระมัดระวังอย่างยิ่ง

# Upload state file ไปยัง backend
terraform state push modified-state.json

# ใช้เมื่อ:
# - Emergency state recovery
# - Manual state correction
# - Migrating state ระหว่าง environments

# Best practice: backup ก่อน push
terraform state pull > state-backup-$(date +%Y%m%d-%H%M%S).json
# แก้ไข state...
terraform state push modified-state.json
```

### Manual State Correction Workflow

```bash
#!/bin/bash
# emergency-state-fix.sh

set -e

echo "=== Emergency State Fix ==="
echo "⚠️  WARNING: This directly modifies Terraform state!"
echo ""

# 1. Backup ก่อนทำอะไร
BACKUP_FILE="state-backup-$(date +%Y%m%d-%H%M%S).json"
echo "Step 1: Backing up current state to $BACKUP_FILE"
terraform state pull > "$BACKUP_FILE"
echo "✅ Backup saved: $BACKUP_FILE"

# 2. Edit state (ถ้าจำเป็น)
# cp "$BACKUP_FILE" modified-state.json
# vim modified-state.json

# 3. Validate modified state
echo "Step 2: Validating modified state..."
python3 -c "import json,sys; json.load(open('modified-state.json'))" && \
  echo "✅ Valid JSON" || \
  echo "❌ Invalid JSON!"

# 4. Push (ถ้า validate pass)
echo "Step 3: Pushing modified state..."
# terraform state push modified-state.json

echo "=== Done ==="
```

---

## Step 276: terraform state replace-provider

### เปลี่ยน Provider สำหรับ Resources ใน State

```bash
# ใช้เมื่อ:
# - Provider namespace เปลี่ยน (เช่น hashicorp/ -> registry/)
# - ย้ายจาก community provider ไป official provider
# - Provider ถูก fork

# ตัวอย่าง: เปลี่ยน provider namespace
terraform state replace-provider \
  registry.terraform.io/hashicorp/aws \
  registry.terraform.io/neworg/aws

# ตัวอย่าง: เปลี่ยน provider version ใน state
terraform state replace-provider \
  "registry.terraform.io/hashicorp/kubernetes" \
  "registry.terraform.io/hashicorp/kubernetes"

# Confirm:
# Terraform will make the following changes:
# - Replace provider registry.terraform.io/hashicorp/aws 
#   with registry.terraform.io/neworg/aws
#   for all resources currently in state

# Enter a value: yes
```

---

## Step 277: terraform show

### Show State หรือ Plan

```bash
# แสดง state ทั้งหมด
terraform show

# แสดง plan file
terraform show tfplan
terraform show myplan.tfplan

# JSON output
terraform show -json
terraform show -json tfplan

# No color (สำหรับ CI)
terraform show -no-color
```

### ใช้ terraform show -json สำหรับ Automation

```bash
# ดึงข้อมูล instance IDs ทั้งหมด
terraform show -json | \
  jq '[.values.root_module.resources[] 
       | select(.type == "aws_instance") 
       | {name: .name, id: .values.id, ip: .values.private_ip}]'

# Output:
# [
#   {"name": "web", "id": "i-111", "ip": "10.0.1.100"},
#   {"name": "app", "id": "i-222", "ip": "10.0.1.101"}
# ]

# ดู outputs
terraform show -json | jq '.values.outputs'

# ดู module resources
terraform show -json | jq '
  .values.root_module.child_modules[] 
  | {module: .address, resources: [.resources[].address]}'
```

---

## Step 278: terraform output

### ดู Output Values

```bash
# แสดง outputs ทั้งหมด
terraform output

# ตัวอย่าง:
# instance_id = "i-1234567890abcdef0"
# public_ip = "52.1.2.3"
# vpc_id = "vpc-12345678"

# แสดง output เฉพาะ
terraform output vpc_id
# "vpc-12345678"

# Raw value (ไม่มี quotes)
terraform output -raw vpc_id
# vpc-12345678

# JSON format
terraform output -json
# {
#   "instance_id": {"value": "i-xxx", "type": "string"},
#   "vpc_id": {"value": "vpc-xxx", "type": "string"}
# }

# Specific output เป็น JSON
terraform output -json public_ips

# No color
terraform output -no-color
```

### ใช้ Output ใน Scripts

```bash
#!/bin/bash
# deploy-app.sh

# ดึง values จาก Terraform outputs
DB_ENDPOINT=$(terraform output -raw db_endpoint)
APP_SUBNET=$(terraform output -raw app_subnet_id)
APP_SG=$(terraform output -raw app_security_group_id)

echo "Database: $DB_ENDPOINT"
echo "Subnet: $APP_SUBNET"

# ใช้ค่าเหล่านี้ใน deployment script
aws ecs create-service \
  --cluster my-cluster \
  --service-name my-app \
  --task-definition my-app:latest \
  --network-configuration "awsvpcConfiguration={subnets=[$APP_SUBNET],securityGroups=[$APP_SG]}"

# ดึง list values
INSTANCE_IDS=$(terraform output -json instance_ids | jq -r '.[]')
for id in $INSTANCE_IDS; do
  echo "Instance: $id"
  aws ec2 describe-instance-status --instance-id $id
done
```

---

## Step 279: Practical State Operations

### Use Case 1: Rename Resource โดยไม่ Destroy

```bash
# Scenario: ต้องการ rename จาก aws_instance.web ไป aws_instance.web_server

# Step 1: state mv
terraform state mv aws_instance.web aws_instance.web_server

# Step 2: อัพเดต code
# เปลี่ยน resource "aws_instance" "web" { ... }
# เป็น resource "aws_instance" "web_server" { ... }
# และอัพเดต references ทั้งหมด (เช่น aws_instance.web.id -> aws_instance.web_server.id)

# Step 3: Verify
terraform plan
# ควร show "No changes"
```

### Use Case 2: Move Resource เข้า Module

```bash
# Scenario: refactor ย้าย aws_vpc.main เข้า module.networking

# Step 1: state mv
terraform state mv \
  aws_vpc.main \
  module.networking.aws_vpc.main

# Step 2: อัพเดต code
# ลบ resource "aws_vpc" "main" จาก root
# สร้าง module "networking" { source = "./modules/networking" }
# ใน modules/networking/main.tf: resource "aws_vpc" "main" { ... }

# Step 3: อัพเดต references
# aws_vpc.main.id -> module.networking.vpc_id (output)

# Step 4: Verify
terraform plan
```

### Use Case 3: Fix State Corruption

```bash
# Scenario: state เสีย มี resource ที่ไม่มีอยู่จริงใน AWS

# Step 1: ตรวจสอบ
terraform state list

# Step 2: ตรวจสอบว่า resource มีอยู่จริงหรือไม่
aws ec2 describe-instances --instance-ids i-deleted123
# Error: instance not found

# Step 3: ลบออกจาก state
terraform state rm aws_instance.deleted_instance

# Step 4: Verify
terraform plan
# ถ้า resource ยังอยู่ใน config -> จะ show "create"
# ถ้าลบออกจาก config ด้วย -> No changes
```

### Use Case 4: Split Monolithic State

```bash
# Scenario: state ใหญ่มาก ต้องการแยกเป็น networking, databases, applications

# State ปัจจุบัน:
# aws_vpc.main
# aws_subnet.* (6 subnets)
# aws_db_instance.main
# aws_instance.web[0..2]
# aws_s3_bucket.assets

# Step 1: Download state
terraform state pull > monolithic.tfstate

# Step 2: สร้าง networking project
mkdir networking-project && cd networking-project

# สร้าง backend config สำหรับ networking
terraform {
  backend "s3" {
    bucket = "my-state-bucket"
    key    = "networking/terraform.tfstate"
    region = "us-east-1"
  }
}

# Init
terraform init

# Step 3: Pull empty state จาก networking backend
terraform state pull > networking.tfstate
# จะได้ empty state

# Step 4: import networking resources
cd networking-project
terraform import aws_vpc.main vpc-12345678
terraform import 'aws_subnet.public[0]' subnet-11111111
# ... import ทุก networking resource

# Step 5: ลบ networking resources ออกจาก monolithic state
cd monolithic-project
terraform state rm aws_vpc.main
terraform state rm 'aws_subnet.public[0]'
# ...
```

---

## Step 280: State Commands Summary & Best Practices

### Commands Reference

```bash
# Listing
terraform state list               # List all resources
terraform state list 'PATTERN'     # Filter by pattern
terraform state list -id=CLOUD_ID  # Filter by resource ID

# Showing
terraform state show ADDR          # Show resource details
terraform show                     # Show all state
terraform show -json               # JSON output
terraform output                   # Show outputs
terraform output OUTNAME           # Specific output
terraform output -raw OUTNAME      # Raw value
terraform output -json             # JSON format

# Moving
terraform state mv SRC DST         # Move/rename resource
terraform state mv -state-out=FILE SRC DST  # Move to different state

# Removing
terraform state rm ADDR            # Remove from state (not destroy)
terraform state rm ADDR1 ADDR2     # Remove multiple

# State I/O
terraform state pull               # Download state
terraform state push FILE          # Upload state (dangerous!)

# Provider management
terraform state replace-provider OLD NEW  # Replace provider
```

### ⚠️ State Command Precautions

```bash
# ก่อนทำ state operations:
# 1. BACKUP ก่อนเสมอ
terraform state pull > backup-$(date +%Y%m%d-%H%M%S).json

# 2. Lock state (manual check)
# ตรวจสอบว่าไม่มีคนอื่น running terraform ในขณะนี้

# 3. Verify หลัง operation
terraform plan  # ดูว่า plan ถูกต้อง

# 4. ถ้าใช้ remote backend: state operations จะ lock อัตโนมัติ

# 5. ทำใน staging ก่อน production
```

### Common Mistakes ที่ควรหลีกเลี่ยง

```bash
# ❌ 1. rm แล้วลืมลบออกจาก config
terraform state rm aws_instance.web
# ถ้าไม่ลบออกจาก config -> plan จะ create ใหม่!

# ❌ 2. mv โดยไม่ update code
terraform state mv aws_instance.web aws_instance.web_server
# ถ้า code ยังใช้ aws_instance.web -> plan จะ:
# + create aws_instance.web (สร้างใหม่)
# - destroy aws_instance.web_server (ลบของที่ mv มา)

# ❌ 3. push state โดยไม่ backup ก่อน
terraform state push wrong-state.json
# ไม่สามารถ undo ได้ง่ายๆ (ต้องใช้ S3 versioning)

# ❌ 4. rm resource ที่ dependency อื่นขึ้นอยู่
terraform state rm aws_vpc.main
# ถ้า subnet ยังอ้างถึง vpc -> plan จะ error หรือ unexpected behavior
```

---

## แบบฝึกหัด (Exercises)

### Exercise 1: State Inspection

```bash
# 1. Deploy infrastructure
terraform apply -auto-approve

# 2. List all resources
terraform state list

# 3. Show details of each resource type
terraform state show <resource_address>

# 4. Export outputs
terraform output -json > outputs.json
cat outputs.json | jq '.'
```

### Exercise 2: Resource Rename

1. สร้าง resource `aws_instance.old_name`
2. State mv เป็น `aws_instance.new_name`
3. อัพเดต config
4. Verify `terraform plan` shows no changes

### Exercise 3: Module Refactoring

1. สร้าง resources ใน root module
2. ย้าย code ไปใน submodule
3. ใช้ `terraform state mv` เพื่อ move resources เข้า module
4. Verify ไม่มี destroy/recreate

---

## Checklist

- [ ] สามารถใช้ `terraform state list` กับ filters ได้
- [ ] รู้วิธี inspect resource ด้วย `terraform state show`
- [ ] สามารถ rename resource ด้วย `state mv` โดยไม่ destroy ได้
- [ ] รู้วิธี move resource เข้า/ออก module
- [ ] เข้าใจความต่างระหว่าง `state rm` และ `destroy`
- [ ] รู้วิธี backup state ก่อน operations
- [ ] สามารถใช้ `terraform output` ใน scripts ได้
- [ ] เข้าใจ common mistakes และวิธีหลีกเลี่ยง
