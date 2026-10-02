# Part 079: Terraform State Advanced Operations (ขั้นตอนที่ 781-790)

## บทนำ (Introduction)

การทำ advanced state operations เป็นทักษะสำคัญสำหรับ Terraform practitioners ระดับ senior
บทนี้จะครอบคลุมการ manipulate state อย่างปลอดภัยและมีประสิทธิภาพ

---

## ขั้นตอนที่ 781: State File JSON Structure

### โครงสร้าง terraform.tfstate

```json
{
  "version": 4,                              // State format version
  "terraform_version": "1.7.0",
  "serial": 47,                              // ตัวเลขนับเพิ่มทุก apply
  "lineage": "abc123-def456-ghi789",         // Unique identifier ของ state
  "outputs": {
    "vpc_id": {
      "value": "vpc-0abc123def456",
      "type": "string",
      "sensitive": false
    },
    "database_password": {
      "value": "s3cr3t",
      "type": "string",
      "sensitive": true
    }
  },
  "resources": [
    {
      "module": "module.networking",          // null ถ้าอยู่ root
      "mode": "managed",                      // managed | data
      "type": "aws_vpc",
      "name": "main",
      "provider": "provider[\"registry.terraform.io/hashicorp/aws\"]",
      "instances": [
        {
          "schema_version": 1,
          "attributes": {
            "arn":                     "arn:aws:ec2:us-east-1:123456789012:vpc/vpc-0abc123",
            "cidr_block":              "10.0.0.0/16",
            "enable_dns_hostnames":    true,
            "enable_dns_support":      true,
            "id":                      "vpc-0abc123def456",
            "instance_tenancy":        "default",
            "tags": {
              "Environment": "production",
              "Name":        "main-vpc"
            },
            "tags_all": {
              "Environment": "production",
              "Name":        "main-vpc"
            }
          },
          "sensitive_attributes": [],
          "private":                 "eyJlMmJmYjk...",  // base64 encoded private state
          "dependencies": [
            "aws_vpc.main"
          ]
        }
      ]
    }
  ],
  "check_results": null
}
```

---

## ขั้นตอนที่ 782: State Versioning

### Serial Number

```
Serial Number เพิ่มขึ้นทุกครั้งที่มีการเปลี่ยนแปลง state:
- Create resource: +1
- Update resource: +1
- Delete resource: +1
- Import resource: +1

ถ้า serial ใน remote state > serial ที่ local มี = Conflict!
Terraform จะ refuse ถ้า serial ไม่ตรงกัน
```

### State Lineage

```bash
# Lineage คือ UUID ที่ generate ครั้งแรก
# ไม่เคย change ตลอดอายุของ state

# ดู lineage
cat terraform.tfstate | python3 -c "
import json, sys
state = json.load(sys.stdin)
print('Lineage:', state['lineage'])
print('Serial:', state['serial'])
print('Version:', state['version'])
"
```

### State Pull และ Push

```bash
# ดึง state จาก remote backend
terraform state pull > local-backup.tfstate

# Push state ขึ้น remote (ระวัง! ใช้เฉพาะกรณีจำเป็น)
terraform state push local-backup.tfstate

# Push state ที่แก้ไขแล้ว (เพิ่ม serial ก่อน push)
# จะ fail ถ้า serial ไม่ใหม่กว่า
```

---

## ขั้นตอนที่ 783: terraform state mv Complex Patterns

### ย้าย count Resources ไป for_each

```bash
# State ก่อน (count):
# aws_instance.servers[0]
# aws_instance.servers[1]
# aws_instance.servers[2]

# หลัง migration ต้องการ (for_each):
# aws_instance.servers["web-1"]
# aws_instance.servers["web-2"]
# aws_instance.servers["web-3"]

# รัน state mv commands
terraform state mv \
  'aws_instance.servers[0]' \
  'aws_instance.servers["web-1"]'

terraform state mv \
  'aws_instance.servers[1]' \
  'aws_instance.servers["web-2"]'

terraform state mv \
  'aws_instance.servers[2]' \
  'aws_instance.servers["web-3"]'
```

### Script สำหรับ Bulk count → for_each Migration

```bash
#!/bin/bash
# migrate_count_to_for_each.sh

RESOURCE_TYPE="aws_instance"
RESOURCE_NAME="servers"
KEYS=("web-1" "web-2" "web-3")

echo "Migrating ${RESOURCE_TYPE}.${RESOURCE_NAME} from count to for_each..."

# Backup state ก่อน
echo "Creating state backup..."
terraform state pull > "state-backup-$(date +%Y%m%d_%H%M%S).tfstate"

# ทำ migration
for i in "${!KEYS[@]}"; do
  OLD_ADDRESS="${RESOURCE_TYPE}.${RESOURCE_NAME}[$i]"
  NEW_ADDRESS="${RESOURCE_TYPE}.${RESOURCE_NAME}[\"${KEYS[$i]}\"]"
  
  echo "Moving: $OLD_ADDRESS → $NEW_ADDRESS"
  terraform state mv "$OLD_ADDRESS" "$NEW_ADDRESS"
  
  if [ $? -ne 0 ]; then
    echo "ERROR: Failed to move $OLD_ADDRESS"
    echo "State may be in inconsistent state. Restore from backup!"
    exit 1
  fi
done

echo "Migration complete. Running plan to verify..."
terraform plan
```

### ย้าย Resources ระหว่าง Modules

```bash
# ย้าย resource จาก root ไป module
terraform state mv \
  aws_vpc.main \
  module.networking.aws_vpc.main

# ย้าย resource จาก module ไป root
terraform state mv \
  module.networking.aws_vpc.main \
  aws_vpc.main

# ย้าย resource ระหว่าง modules
terraform state mv \
  module.app.aws_iam_role.execution \
  module.iam.aws_iam_role.app_execution

# ย้าย module ทั้งหมด (rename module)
terraform state mv \
  module.old_name \
  module.new_name
```

---

## ขั้นตอนที่ 784: terraform state rm Patterns

### ลบ Resources ออกจาก State

```bash
# ลบ resource เดี่ยว
terraform state rm aws_instance.legacy

# ลบ resource ใน module
terraform state rm module.networking.aws_subnet.old

# ลบ resource ที่ใช้ count
terraform state rm 'aws_instance.servers[0]'
terraform state rm 'aws_instance.servers[1]'

# ลบ resource ที่ใช้ for_each
terraform state rm 'aws_instance.servers["web-1"]'

# ลบ data source
terraform state rm data.aws_ami.latest

# ลบ module ทั้งหมด (ระวัง!)
terraform state rm module.old_application
```

### Bulk Removal

```bash
#!/bin/bash
# bulk_state_rm.sh - ลบหลาย resources พร้อมกัน

# รายการ resources ที่ต้องการลบ
RESOURCES_TO_REMOVE=(
  "aws_instance.legacy_web"
  "aws_instance.legacy_db"
  "aws_eip.legacy_web"
  "aws_security_group.legacy_web"
  "module.old_app"
)

# Backup ก่อน
echo "Creating backup..."
terraform state pull > "state-backup-$(date +%Y%m%d_%H%M%S).tfstate"

# ลบแต่ละ resource
for resource in "${RESOURCES_TO_REMOVE[@]}"; do
  echo "Removing: $resource"
  terraform state rm "$resource"
done

echo "Done. Run 'terraform plan' to see the changes."
```

### ลบ Resources ที่ถูก Auto-Created

```bash
# บางครั้ง resources ถูก create โดย AWS automatically
# เช่น default VPC, default security groups
# ต้องการ untrack เหล่านี้ออกจาก Terraform

# 1. ดูว่ามี resources อะไรที่อยากลบออก
terraform state list | grep "default"

# 2. ลบออกจาก state (ไม่ delete จาก AWS)
terraform state rm aws_vpc.default
terraform state rm aws_security_group.default
terraform state rm aws_internet_gateway.default
```

---

## ขั้นตอนที่ 785: State Surgery - Merging Two States

### Scenario: รวม 2 states เป็น 1

```bash
#!/bin/bash
# merge_states.sh
# รวม state จาก 2 directories เข้าด้วยกัน

STATE_A_DIR="./old-networking"
STATE_B_DIR="./old-compute"
MERGED_DIR="./merged"

mkdir -p "$MERGED_DIR"

# Step 1: Backup ทั้งสอง states
cp "$STATE_A_DIR/terraform.tfstate" "./backup-state-a.tfstate"
cp "$STATE_B_DIR/terraform.tfstate" "./backup-state-b.tfstate"

# Step 2: Copy state A เป็น base
cp "$STATE_A_DIR/terraform.tfstate" "$MERGED_DIR/terraform.tfstate"

# Step 3: รัน terraform init ใน merged dir
cd "$MERGED_DIR"
terraform init

# Step 4: Import resources จาก state B ทีละ resource
# วิธีที่ง่ายที่สุดคือใช้ terraform state push กับ state ที่ merge manual

echo "Manual merge required. Use 'terraform state push' to complete."
echo "See merge_states_manual.py for automated merging."
```

### Python Script สำหรับ Merge States

```python
#!/usr/bin/env python3
# merge_states.py
# รวม 2 Terraform state files เข้าด้วยกัน

import json
import sys
import uuid
from datetime import datetime

def merge_states(state_a_path, state_b_path, output_path):
    """Merge two Terraform state files"""
    
    # อ่าน states
    with open(state_a_path) as f:
        state_a = json.load(f)
    
    with open(state_b_path) as f:
        state_b = json.load(f)
    
    # ตรวจสอบว่า versions ตรงกัน
    if state_a['version'] != state_b['version']:
        raise ValueError(f"State version mismatch: {state_a['version']} vs {state_b['version']}")
    
    # สร้าง merged state
    merged = {
        'version': state_a['version'],
        'terraform_version': state_a['terraform_version'],
        'serial': max(state_a['serial'], state_b['serial']) + 1,
        'lineage': str(uuid.uuid4()),  # New lineage for merged state
        'outputs': {},
        'resources': []
    }
    
    # Merge outputs
    merged['outputs'].update(state_a.get('outputs', {}))
    merged['outputs'].update(state_b.get('outputs', {}))
    
    # Merge resources
    all_resources = list(state_a.get('resources', []))
    
    # ตรวจสอบ duplicate resources
    existing_addresses = set()
    for r in all_resources:
        module = r.get('module', '')
        key = f"{module}.{r['type']}.{r['name']}"
        existing_addresses.add(key)
    
    conflicts = []
    for r in state_b.get('resources', []):
        module = r.get('module', '')
        key = f"{module}.{r['type']}.{r['name']}"
        
        if key in existing_addresses:
            conflicts.append(key)
        else:
            all_resources.append(r)
    
    if conflicts:
        print("WARNING: Duplicate resources found:")
        for c in conflicts:
            print(f"  - {c}")
        print("These resources from state_b will be skipped.")
    
    merged['resources'] = all_resources
    
    # บันทึก merged state
    with open(output_path, 'w') as f:
        json.dump(merged, f, indent=2)
    
    print(f"Merged state saved to: {output_path}")
    print(f"Total resources: {len(merged['resources'])}")
    print(f"Outputs: {list(merged['outputs'].keys())}")
    print(f"New serial: {merged['serial']}")
    print(f"New lineage: {merged['lineage']}")

if __name__ == '__main__':
    if len(sys.argv) != 4:
        print("Usage: python3 merge_states.py state_a.tfstate state_b.tfstate merged.tfstate")
        sys.exit(1)
    
    merge_states(sys.argv[1], sys.argv[2], sys.argv[3])
```

```bash
# ใช้ script
python3 merge_states.py \
  networking/terraform.tfstate \
  compute/terraform.tfstate \
  merged/terraform.tfstate

# Push merged state (ระวัง!)
cd merged
terraform init
terraform state push merged/terraform.tfstate

# Verify
terraform plan  # ควรเป็น no-op
```

---

## ขั้นตอนที่ 786: State Surgery - Splitting State

### แบ่ง State ใหญ่เป็น State เล็กๆ

```bash
#!/bin/bash
# split_state.sh
# แบ่ง state เป็น networking และ compute

SOURCE_DIR="./monolith"
NETWORKING_DIR="./networking"
COMPUTE_DIR="./compute"

# Backup
cp "$SOURCE_DIR/terraform.tfstate" "./backup-monolith.tfstate"

# --- สร้าง Networking State ---
mkdir -p "$NETWORKING_DIR"
cp "$SOURCE_DIR/terraform.tfstate" "$NETWORKING_DIR/terraform.tfstate"
cd "$NETWORKING_DIR"
terraform init

# ลบ resources ที่ไม่ใช่ networking ออก
COMPUTE_RESOURCES=(
  "aws_instance.app"
  "aws_instance.web"
  "aws_autoscaling_group.app"
  "aws_launch_template.app"
)

for resource in "${COMPUTE_RESOURCES[@]}"; do
  terraform state rm "$resource"
done

# --- สร้าง Compute State ---
cd ..
mkdir -p "$COMPUTE_DIR"
cp "$SOURCE_DIR/terraform.tfstate" "$COMPUTE_DIR/terraform.tfstate"
cd "$COMPUTE_DIR"
terraform init

# ลบ resources ที่ไม่ใช่ compute ออก
NETWORKING_RESOURCES=(
  "aws_vpc.main"
  "aws_subnet.public_1"
  "aws_subnet.public_2"
  "aws_subnet.private_1"
  "aws_subnet.private_2"
  "aws_internet_gateway.main"
  "aws_route_table.public"
  "aws_route_table.private"
)

for resource in "${NETWORKING_RESOURCES[@]}"; do
  terraform state rm "$resource"
done

echo "Split complete!"
echo "Verify each state with 'terraform plan'"
```

---

## ขั้นตอนที่ 787: State Disaster Recovery

### State Backup Strategy

```bash
# 1. Enable S3 versioning สำหรับ state bucket
aws s3api put-bucket-versioning \
  --bucket mycompany-terraform-state \
  --versioning-configuration Status=Enabled

# 2. ตรวจสอบ state versions
aws s3api list-object-versions \
  --bucket mycompany-terraform-state \
  --prefix "prod/networking/terraform.tfstate" \
  --query 'Versions[*].{VersionId:VersionId,LastModified:LastModified}'

# 3. สร้าง cross-region replication
aws s3api put-bucket-replication \
  --bucket mycompany-terraform-state \
  --replication-configuration file://replication-config.json
```

### Automatic State Backup Script

```bash
#!/bin/bash
# backup_terraform_states.sh
# Backup all Terraform states ทุกวัน

STATES_BUCKET="mycompany-terraform-state"
BACKUP_BUCKET="mycompany-terraform-state-backup"
DATE=$(date +%Y%m%d)

# List all state files
aws s3 ls "s3://$STATES_BUCKET/" --recursive \
  | grep "terraform.tfstate$" \
  | awk '{print $4}' \
  | while read STATE_KEY; do
    BACKUP_KEY="backups/$DATE/$STATE_KEY"
    echo "Backing up: $STATE_KEY → $BACKUP_KEY"
    aws s3 cp \
      "s3://$STATES_BUCKET/$STATE_KEY" \
      "s3://$BACKUP_BUCKET/$BACKUP_KEY"
  done

echo "Backup complete for $DATE"
```

### Restore State จาก S3 Version

```bash
# ดู versions ที่มี
aws s3api list-object-versions \
  --bucket mycompany-terraform-state \
  --prefix "prod/networking/terraform.tfstate"

# Restore specific version
VERSION_ID="abc123xyz"
aws s3api get-object \
  --bucket mycompany-terraform-state \
  --key "prod/networking/terraform.tfstate" \
  --version-id "$VERSION_ID" \
  restored-state.tfstate

# ตรวจสอบ restored state
cat restored-state.tfstate | python3 -m json.tool | head -20

# Push restored state
terraform state push restored-state.tfstate

# Verify
terraform plan
```

### Rebuilding State จาก Scratch

```bash
# กรณี state หาย และไม่มี backup
# ต้อง import resources ทั้งหมดใหม่

#!/bin/bash
# rebuild_state.sh

# 1. ดู resources ที่มีอยู่จริงใน AWS
echo "Existing VPCs:"
aws ec2 describe-vpcs \
  --filters "Name=tag:ManagedBy,Values=terraform" \
  --query 'Vpcs[*].{ID:VpcId,CIDR:CidrBlock,Name:Tags[?Key==`Name`]|[0].Value}'

echo "Existing EC2 instances:"
aws ec2 describe-instances \
  --filters "Name=tag:ManagedBy,Values=terraform" "Name=instance-state-name,Values=running" \
  --query 'Reservations[*].Instances[*].{ID:InstanceId,Type:InstanceType,Name:Tags[?Key==`Name`]|[0].Value}'

# 2. Import สำหรับ resource เหล่านั้น
echo "Importing VPC..."
terraform import aws_vpc.main vpc-0abc123def456

echo "Importing Subnets..."
terraform import 'aws_subnet.public[0]' subnet-0111111111
terraform import 'aws_subnet.public[1]' subnet-0222222222

echo "Importing EC2 instances..."
terraform import 'aws_instance.app["web-1"]' i-0aaaaaaaaaaaaaaaa

echo "Rebuilding complete. Run 'terraform plan' to verify."
```

---

## ขั้นตอนที่ 788: State Consistency Checks

### ตรวจสอบ State Integrity

```bash
#!/bin/bash
# check_state_consistency.sh

echo "=== Terraform State Consistency Check ==="

# 1. ตรวจสอบ plan เป็น no-op
echo "1. Checking for drift..."
PLAN_OUTPUT=$(terraform plan -detailed-exitcode 2>&1)
EXIT_CODE=$?

case $EXIT_CODE in
  0)
    echo "✓ No changes detected - state is consistent"
    ;;
  1)
    echo "✗ Error running plan:"
    echo "$PLAN_OUTPUT"
    exit 1
    ;;
  2)
    echo "⚠ Changes detected - possible drift:"
    terraform plan -no-color | grep -E "(must be|will be|is tainted|Plan:)"
    ;;
esac

# 2. ตรวจสอบ state file syntax
echo ""
echo "2. Validating state file..."
STATE=$(terraform state pull)

if echo "$STATE" | python3 -m json.tool > /dev/null 2>&1; then
  echo "✓ State file is valid JSON"
else
  echo "✗ State file is corrupted!"
  exit 1
fi

# 3. ตรวจสอบ resource count
RESOURCE_COUNT=$(echo "$STATE" | python3 -c "
import json, sys
state = json.load(sys.stdin)
print(len(state.get('resources', [])))
")
echo "Total resources in state: $RESOURCE_COUNT"

# 4. ตรวจสอบ outputs
OUTPUT_COUNT=$(echo "$STATE" | python3 -c "
import json, sys
state = json.load(sys.stdin)
print(len(state.get('outputs', {})))
")
echo "Total outputs in state: $OUTPUT_COUNT"

# 5. ตรวจสอบ orphaned resources
echo ""
echo "3. Checking for orphaned resources (in state but not in config)..."
terraform state list > /tmp/state_resources.txt
grep -c "^" /tmp/state_resources.txt
```

---

## ขั้นตอนที่ 789: State File Access Control

### S3 Bucket Policy สำหรับ State Security

```json
// s3-state-bucket-policy.json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "EnforceTLS",
      "Effect": "Deny",
      "Principal": "*",
      "Action": "s3:*",
      "Resource": [
        "arn:aws:s3:::mycompany-terraform-state",
        "arn:aws:s3:::mycompany-terraform-state/*"
      ],
      "Condition": {
        "Bool": {
          "aws:SecureTransport": "false"
        }
      }
    },
    {
      "Sid": "AllowTerraformRoles",
      "Effect": "Allow",
      "Principal": {
        "AWS": [
          "arn:aws:iam::123456789012:role/terraform-prod-role",
          "arn:aws:iam::123456789012:role/terraform-staging-role",
          "arn:aws:iam::123456789012:role/github-actions-terraform"
        ]
      },
      "Action": [
        "s3:GetObject",
        "s3:PutObject",
        "s3:DeleteObject",
        "s3:ListBucket"
      ],
      "Resource": [
        "arn:aws:s3:::mycompany-terraform-state",
        "arn:aws:s3:::mycompany-terraform-state/*"
      ]
    },
    {
      "Sid": "DenyDirectAccess",
      "Effect": "Deny",
      "Principal": "*",
      "Action": "s3:*",
      "Resource": "arn:aws:s3:::mycompany-terraform-state/*",
      "Condition": {
        "StringNotLike": {
          "aws:PrincipalArn": [
            "arn:aws:iam::123456789012:role/terraform-*",
            "arn:aws:iam::123456789012:role/github-actions-*"
          ]
        }
      }
    }
  ]
}
```

### IAM Role สำหรับ Terraform

```hcl
# iam-terraform-role.tf
resource "aws_iam_role" "terraform_prod" {
  name = "terraform-prod-role"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        # GitHub Actions (OIDC)
        Effect = "Allow"
        Principal = {
          Federated = "arn:aws:iam::${data.aws_caller_identity.current.account_id}:oidc-provider/token.actions.githubusercontent.com"
        }
        Action = "sts:AssumeRoleWithWebIdentity"
        Condition = {
          StringEquals = {
            "token.actions.githubusercontent.com:aud" = "sts.amazonaws.com"
            "token.actions.githubusercontent.com:sub" = "repo:mycompany/infrastructure:ref:refs/heads/main"
          }
        }
      }
    ]
  })
}

# State-specific S3 permissions
resource "aws_iam_role_policy" "terraform_state" {
  name = "terraform-state-access"
  role = aws_iam_role.terraform_prod.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Action = [
          "s3:ListBucket",
          "s3:GetBucketVersioning"
        ]
        Resource = "arn:aws:s3:::mycompany-terraform-state"
      },
      {
        Effect = "Allow"
        Action = [
          "s3:GetObject",
          "s3:PutObject",
          "s3:DeleteObject"
        ]
        Resource = "arn:aws:s3:::mycompany-terraform-state/prod/*"
        # จำกัดเฉพาะ prod/ path สำหรับ prod role
      },
      {
        Effect = "Allow"
        Action = [
          "dynamodb:GetItem",
          "dynamodb:PutItem",
          "dynamodb:DeleteItem"
        ]
        Resource = "arn:aws:dynamodb:us-east-1:123456789012:table/terraform-state-lock"
      }
    ]
  })
}
```

---

## ขั้นตอนที่ 790: Advanced State Commands

### Comprehensive State Command Reference

```bash
# 1. list - แสดง resources ทั้งหมด
terraform state list
terraform state list module.networking  # เฉพาะ module

# 2. show - แสดงรายละเอียด resource
terraform state show aws_instance.web
terraform state show 'aws_instance.servers["web-1"]'
terraform state show module.networking.aws_vpc.main

# 3. mv - ย้าย resource
terraform state mv SOURCE DESTINATION

# 4. rm - ลบ resource
terraform state rm RESOURCE_ADDRESS

# 5. pull - ดึง state ปัจจุบัน
terraform state pull

# 6. push - push state ขึ้น remote
terraform state push terraform.tfstate

# 7. replace-provider - เปลี่ยน provider
terraform state replace-provider \
  registry.terraform.io/hashicorp/aws \
  registry.terraform.io/mycompany/aws

# 8. import (via CLI - ไม่แนะนำ, ใช้ import block แทน)
terraform import aws_instance.web i-1234567890abcdef0
```

### State Taint (Deprecated ใน v0.15.2+)

```bash
# ใน Terraform 0.15.1 และก่อนหน้า
terraform taint aws_instance.web

# ใน Terraform 0.15.2+ ใช้ -replace แทน
terraform apply -replace=aws_instance.web

# ดู resource ที่ tainted
terraform state show aws_instance.web | grep "tainted"
```

### Complete State Management Workflow

```bash
#!/bin/bash
# complete_state_workflow.sh
# Script สำหรับ complex state operations

set -euo pipefail

OPERATION="${1:-help}"

case "$OPERATION" in
  "backup")
    echo "Creating state backup..."
    BACKUP_FILE="terraform.tfstate.backup.$(date +%Y%m%d_%H%M%S)"
    terraform state pull > "$BACKUP_FILE"
    echo "Backup created: $BACKUP_FILE"
    ;;

  "list")
    echo "Listing all resources..."
    terraform state list | sort
    ;;

  "check")
    echo "Checking state consistency..."
    if terraform plan -detailed-exitcode > /dev/null 2>&1; then
      echo "✓ State is consistent (no changes)"
    else
      echo "⚠ State may have drift, run 'terraform plan' to see details"
    fi
    ;;

  "count")
    echo "Resource count by type:"
    terraform state list \
      | sed 's/\[.*\]//' \
      | sed 's/^module\.[^.]*\.//' \
      | sed 's/\..*//' \
      | sort | uniq -c | sort -rn
    ;;

  "help"|*)
    echo "Usage: $0 {backup|list|check|count}"
    ;;
esac
```

---

## สรุป (Summary)

| Operation | Command | ใช้เมื่อ |
|-----------|---------|----------|
| Backup state | `terraform state pull > backup.tfstate` | ก่อนทุก major operation |
| List resources | `terraform state list` | ดู resources ใน state |
| Show resource | `terraform state show ADDR` | ดูรายละเอียด |
| Move resource | `terraform state mv OLD NEW` | Rename/reorganize |
| Remove resource | `terraform state rm ADDR` | Untrack (ไม่ delete จริง) |
| Import | `terraform import` หรือ import block | นำ existing resource เข้า state |
| Push state | `terraform state push FILE` | Restore จาก backup |

---

*จบ Part 079 - ในส่วนถัดไปจะเรียนรู้เรื่อง Terraform Drift Detection*
