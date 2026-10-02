# Part 26: Terraform CLI - plan & apply (ขั้นตอนที่ 251-260)

## ภาพรวม (Overview)

`terraform plan` และ `terraform apply` เป็นคำสั่งหลักในการใช้งาน Terraform ทุกวัน การเข้าใจอย่างลึกซึ้งทำให้ทำงานได้ปลอดภัยและมีประสิทธิภาพ

---

## Step 251: terraform plan - What It Does

### Plan ทำงานอย่างไร?

```
terraform plan ทำ 3 ขั้นตอน:
1. Refresh: อ่าน state ปัจจุบัน + query provider
2. Diff: เปรียบเทียบ desired state (config) กับ current state
3. Output: แสดง changes ที่จะเกิดขึ้น
```

### ตัวอย่าง Plan Output

```bash
$ terraform plan

Terraform used the selected providers to generate the following execution plan.
Resource actions are indicated with the following symbols:
  + create
  ~ update in-place
  - destroy
-/+ destroy and then create replacement

Terraform will perform the following actions:

  # aws_instance.web will be created
  + resource "aws_instance" "web" {
      + ami                          = "ami-0c55b159cbfafe1f0"
      + arn                          = (known after apply)
      + availability_zone            = (known after apply)
      + id                           = (known after apply)
      + instance_type                = "t3.micro"
      + private_ip                   = (known after apply)
      + public_ip                    = (known after apply)
      + tags                         = {
          + "Name" = "web-server"
        }
    }

  # aws_security_group.web will be updated in-place
  ~ resource "aws_security_group" "web" {
        id   = "sg-12345678"
        name = "web-sg"
      ~ description = "Old description" -> "New description"
        # (7 unchanged attributes hidden)
    }

  # aws_s3_bucket.old will be destroyed
  - resource "aws_s3_bucket" "old" {
      - bucket = "my-old-bucket" -> null
      - id     = "my-old-bucket" -> null
      - arn    = "arn:aws:s3:::my-old-bucket" -> null
    }

  # aws_instance.app must be replaced
-/+ resource "aws_instance" "app" {
      ~ id                           = "i-1234567890" -> (known after apply) # forces replacement
      + ami                          = "ami-new" # forces replacement
      ~ private_ip                   = "10.0.1.5" -> (known after apply)
    }

Plan: 1 to add, 1 to change, 1 to destroy, 1 to replace.
```

---

## Step 252: Plan Symbols Explained

### รู้จักสัญลักษณ์ใน Plan

```
+ (green)  : จะสร้าง resource ใหม่ (create)
~ (yellow) : จะ update resource in-place
- (red)    : จะ destroy resource
-/+ (red)  : จะ destroy แล้วสร้างใหม่ (replace/recreate)
<= (cyan)  : จะอ่าน data source
! (orange) : warning (เช่น resource ถูก tainted)
```

### ตัวอย่าง Plan สำหรับแต่ละ Symbol

```bash
# + Create
+ resource "aws_instance" "new" {
    + ami           = "ami-xxx"
    + instance_type = "t3.micro"
    + id            = (known after apply)
  }

# ~ Update in-place
~ resource "aws_instance" "existing" {
    id            = "i-1234567890"
  ~ instance_type = "t3.micro" -> "t3.small"
    # (10 unchanged attributes hidden)
  }

# - Destroy
- resource "aws_s3_bucket" "to_delete" {
    - bucket = "my-bucket" -> null
    - id     = "my-bucket" -> null
  }

# -/+ Replace (destroy and recreate)
-/+ resource "aws_instance" "replace" {
    ~ id  = "i-old" -> (known after apply) # forces replacement
    ~ ami = "ami-old" -> "ami-new"         # forces replacement
  }

# <= Read data source
<= data "aws_ami" "ubuntu" {
    + id            = (known after apply)
    + most_recent   = true
    + name          = (known after apply)
  }
```

### "(known after apply)" หมายถึงอะไร

```hcl
resource "aws_instance" "web" {
  ami           = "ami-xxx"
  instance_type = "t3.micro"
}

# หลัง plan จะเห็น:
# + id         = (known after apply)  <- AWS generate หลัง create
# + public_ip  = (known after apply)  <- AWS assign หลัง create
# + arn        = (known after apply)  <- computed จาก id + region
# + private_ip = (known after apply)  <- assigned ตอน launch

# ค่าพวกนี้รู้ได้หลัง apply เสร็จเท่านั้น
```

---

## Step 253: terraform plan - Saving Plan Files

### -out Flag

```bash
# Save plan ไปไฟล์
terraform plan -out=tfplan

# Apply จาก plan file (exact operations ที่ plan ไว้)
terraform apply tfplan

# Plan file เป็น binary format ไม่ใช่ text
# ดู content ด้วย:
terraform show tfplan
terraform show -json tfplan | jq '.'
```

### ทำไมควร Save Plan?

```bash
# ✅ Benefits ของ saved plan:
# 1. Apply ทำ exactly what was planned - ไม่มี surprises
# 2. ป้องกัน "plan-apply" drift ถ้า infrastructure เปลี่ยนระหว่างนั้น
# 3. CI/CD: plan ใน PR review, apply เมื่อ merge

# Workflow ที่แนะนำ:
terraform plan -out=tfplan.$(date +%Y%m%d-%H%M%S)
# Review plan...
terraform apply tfplan.20240115-143022

# Clean up plan files
rm tfplan.*
```

---

## Step 254: terraform plan - Variable Flags

### -var Flag

```bash
# Pass variable ผ่าน command line
terraform plan -var="instance_type=t3.large"
terraform plan -var="environment=production" -var="instance_count=3"

# สำหรับ complex types
terraform plan -var='tags={"Env":"prod","Team":"platform"}'
```

### -var-file Flag

```bash
# ใช้ var file
terraform plan -var-file="production.tfvars"

# หลาย var files (ลำดับสำคัญ: ไฟล์หลังทับไฟล์ก่อน)
terraform plan \
  -var-file="common.tfvars" \
  -var-file="production.tfvars" \
  -var-file="secrets.tfvars"

# Terraform auto-loads:
# - terraform.tfvars
# - terraform.tfvars.json
# - *.auto.tfvars
# - *.auto.tfvars.json
```

### ลำดับความสำคัญ (Priority Order)

```
Variable precedence (สูงสุดก่อน):
1. -var flag ใน command line
2. -var-file flag ใน command line
3. *.auto.tfvars files (alphabetical order)
4. terraform.tfvars.json
5. terraform.tfvars
6. TF_VAR_* environment variables
7. Variable default values
```

---

## Step 255: terraform plan - Target & Replace

### -target Flag

```bash
# Plan เฉพาะ resources ที่กำหนด
terraform plan -target=aws_instance.web
terraform plan -target=aws_security_group.app -target=aws_instance.app

# Module targeting
terraform plan -target=module.networking
terraform plan -target=module.networking.aws_vpc.main

# ⚠️ คำเตือน:
# -target ใช้สำหรับ emergency fixes เท่านั้น
# อาจทำให้ config และ state ไม่ sync กัน
# ไม่ควรใช้ใน normal workflow
```

### -replace Flag

```bash
# Force replacement ของ specific resource
terraform plan -replace=aws_instance.web

# ใช้เมื่อ:
# - Resource อยู่ใน bad state
# - ต้องการ recreate โดยไม่แก้ config
# - เปลี่ยน AMI โดยไม่แก้ config

# ตัวอย่าง: instance ที่ disk เสีย
terraform plan -replace=aws_instance.corrupted_node
terraform apply -replace=aws_instance.corrupted_node  # ทำได้โดยตรง
```

---

## Step 256: terraform plan - Other Flags

### -refresh-only

```bash
# Plan เพื่อ update state จาก real infrastructure เท่านั้น
# ไม่ propose changes จาก config
terraform plan -refresh-only

# ใช้เมื่อ:
# - ต้องการ sync state กับ real world
# - หลังจากมีการ change นอก Terraform

# ตัวอย่าง output:
# ~ resource "aws_instance" "web" {
#     # (the value of this field was read from the real resource, not the configuration)
#   ~ tags = {
#       + "Environment" = "production"  # added manually
#     }
#   }
# You can apply this plan to save these new output values
```

### -destroy Flag

```bash
# Plan การ destroy ทุกอย่าง
terraform plan -destroy

# ดู destroy plan ก่อน
terraform plan -destroy -out=destroy.tfplan
terraform apply destroy.tfplan

# เหมือนกับ terraform destroy แต่ดู plan ก่อนได้
```

### -compact-warnings

```bash
# ลด verbose ของ warnings
terraform plan -compact-warnings
```

### -parallelism Flag

```bash
# กำหนดจำนวน operations ที่ run พร้อมกัน (default: 10)
terraform plan -parallelism=20  # เพิ่ม parallelism

# ⚠️ เพิ่มมากเกินไปอาจ hit API rate limits
```

### -json Output

```bash
# Plan output แบบ JSON (machine-readable)
terraform plan -json > plan.json

# หรือ
terraform plan -out=tfplan
terraform show -json tfplan > plan.json

# Parse ด้วย jq
cat plan.json | jq '.resource_changes[] | select(.change.actions[] == "create")'
cat plan.json | jq '.resource_changes[] | select(.change.actions[] == "delete") | .address'
```

### Plan JSON Format

```json
{
  "format_version": "1.2",
  "terraform_version": "1.6.0",
  "variables": {
    "environment": { "value": "production" }
  },
  "planned_values": {
    "root_module": {
      "resources": [...]
    }
  },
  "resource_changes": [
    {
      "address": "aws_instance.web",
      "module_address": null,
      "mode": "managed",
      "type": "aws_instance",
      "name": "web",
      "change": {
        "actions": ["create"],
        "before": null,
        "after": {
          "ami": "ami-xxx",
          "instance_type": "t3.micro"
        },
        "after_unknown": {
          "id": true,
          "public_ip": true
        }
      }
    }
  ],
  "configuration": {...},
  "prior_state": {...}
}
```

---

## Step 257: terraform plan - CI/CD Integration

### GitHub Actions Plan

```yaml
# .github/workflows/terraform-plan.yml
name: Terraform Plan

on:
  pull_request:
    branches: [main]

jobs:
  plan:
    name: Terraform Plan
    runs-on: ubuntu-latest
    
    permissions:
      contents: read
      pull-requests: write
    
    env:
      AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
      AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
    
    steps:
      - uses: actions/checkout@v4
      
      - uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: "~1.6"
      
      - name: Terraform Init
        id: init
        run: terraform init -input=false
        
      - name: Terraform Format
        id: fmt
        run: terraform fmt -check
        continue-on-error: true
        
      - name: Terraform Validate
        id: validate
        run: terraform validate -no-color
        
      - name: Terraform Plan
        id: plan
        run: terraform plan -no-color -out=tfplan 2>&1 | tee plan.log
        continue-on-error: true
        
      - name: Update PR with Plan
        uses: actions/github-script@v7
        with:
          github-token: ${{ secrets.GITHUB_TOKEN }}
          script: |
            const fs = require('fs');
            const plan = fs.readFileSync('plan.log', 'utf8');
            const maxLen = 65536;
            const truncated = plan.length > maxLen 
              ? plan.substring(0, maxLen) + '\n... (truncated)' 
              : plan;
            
            const body = `## Terraform Plan Results
            
            #### 🖊 Format: \`${{ steps.fmt.outcome }}\`
            #### ✅ Validate: \`${{ steps.validate.outcome }}\`
            #### 📖 Plan: \`${{ steps.plan.outcome }}\`
            
            <details><summary>Show Plan</summary>
            
            \`\`\`hcl
            ${truncated}
            \`\`\`
            
            </details>`;
            
            github.rest.issues.createComment({
              issue_number: context.issue.number,
              owner: context.repo.owner,
              repo: context.repo.repo,
              body: body
            });
```

### Atlantis (GitOps for Terraform)

```yaml
# atlantis.yaml - Atlantis configuration
version: 3

projects:
  - name: networking
    dir: ./networking
    workspace: default
    terraform_version: v1.6.0
    autoplan:
      when_modified: ["*.tf", "*.tfvars"]
      enabled: true
    apply_requirements:
      - mergeable
      - approved

  - name: applications-dev
    dir: ./applications
    workspace: dev
    terraform_version: v1.6.0
    apply_requirements:
      - approved

  - name: applications-prod
    dir: ./applications
    workspace: prod
    terraform_version: v1.6.0
    apply_requirements:
      - mergeable
      - approved
      # ต้องการ 2 approvals สำหรับ prod
      - approved_by_count: 2
```

---

## Step 258: terraform apply

### Basic Apply

```bash
# Interactive apply (จะถาม confirm)
terraform apply

# ตัวอย่าง prompt:
# Do you want to perform these actions?
#   Terraform will perform the actions described above.
#   Only 'yes' will be accepted to approve.
#
#   Enter a value: yes

# Apply จาก saved plan (ไม่ถาม confirm)
terraform apply tfplan
```

### Apply Output

```bash
$ terraform apply

aws_vpc.main: Creating...
aws_vpc.main: Still creating... [10s elapsed]
aws_vpc.main: Creation complete after 12s [id=vpc-1234567890abcdef0]

aws_subnet.private[0]: Creating...
aws_subnet.private[1]: Creating...
aws_subnet.private[0]: Creation complete after 5s [id=subnet-12345678]
aws_subnet.private[1]: Creation complete after 5s [id=subnet-87654321]

aws_instance.web: Creating...
aws_instance.web: Still creating... [10s elapsed]
aws_instance.web: Still creating... [20s elapsed]
aws_instance.web: Creation complete after 28s [id=i-1234567890abcdef0]

Apply complete! Resources: 4 added, 0 changed, 0 destroyed.

Outputs:

instance_id = "i-1234567890abcdef0"
vpc_id = "vpc-1234567890abcdef0"
```

---

## Step 259: terraform apply - Flags

### -auto-approve

```bash
# ไม่ต้องถาม confirm (ใช้ใน CI/CD)
terraform apply -auto-approve

# ✅ ใช้ใน CI/CD
# ❌ ระวังใช้ใน production โดยไม่มีการ review

# Best practice: apply จาก saved plan
terraform plan -out=tfplan
terraform apply tfplan  # ไม่ต้อง -auto-approve เพราะ apply จาก plan file
```

### -input=false

```bash
# ป้องกัน interactive prompts
terraform apply -input=false -auto-approve
```

### -var และ -var-file

```bash
# ผ่าน variables ตอน apply
terraform apply -var="environment=production" -auto-approve
terraform apply -var-file="production.tfvars" -auto-approve

# หรือ apply จาก plan ที่ใช้ var files แล้ว
terraform plan -var-file="prod.tfvars" -out=tfplan
terraform apply tfplan
```

### -target Flag

```bash
# Apply เฉพาะ specific resources
terraform apply -target=aws_instance.web -auto-approve

# ⚠️ ระวัง: อาจทำให้ state ไม่สมบูรณ์
# ใช้เฉพาะ emergency เท่านั้น
```

### -replace Flag

```bash
# Force replace specific resource
terraform apply -replace=aws_instance.web -auto-approve

# Useful เมื่อ:
# - resource อยู่ใน bad state
# - ต้องการ cycle เพื่อ refresh configuration
```

### -parallelism Flag

```bash
# กำหนดจำนวน concurrent operations (default: 10)
terraform apply -parallelism=5   # ช้าลง แต่ลด API rate limit issues
terraform apply -parallelism=20  # เร็วขึ้น แต่อาจ hit rate limits

# สำหรับ providers ที่มี rate limits เข้มงวด
terraform apply -parallelism=2
```

### -refresh=false

```bash
# Skip state refresh ก่อน apply (เร็วขึ้น แต่ state อาจไม่ sync)
terraform apply -refresh=false -auto-approve

# ⚠️ ใช้เฉพาะเมื่อแน่ใจว่า infrastructure ไม่มีการเปลี่ยนแปลงนอก Terraform
```

---

## Step 260: Apply Error Handling & Partial Apply

### Error Handling

```bash
# เมื่อ apply ล้มเหลว:
$ terraform apply

aws_vpc.main: Creating...
aws_vpc.main: Creation complete after 12s [id=vpc-xxx]

aws_instance.web: Creating...
aws_instance.web: Still creating... [10s elapsed]

Error: Error launching source instance: InvalidAMIID.NotFound: The image id '[ami-invalid]' does not exist

  on main.tf line 15, in resource "aws_instance" "web":
  15:   ami = var.ami_id

# State จะ save ทุกอย่างที่ succeed ก่อน error
# aws_vpc.main ถูกสร้างแล้วและอยู่ใน state

# Fix the error แล้ว apply ใหม่:
# Terraform จะ skip resources ที่ apply แล้ว
# และ apply เฉพาะที่ยังไม่ได้ทำ
```

### Partial Apply State

```hcl
# เมื่อ apply ล้มเหลวกลางคัน:
# - Resources ที่ succeed: อยู่ใน state ปกติ
# - Resource ที่ fail: อาจอยู่ใน "tainted" state หรือไม่อยู่ใน state เลย

# Tainted resource คือ resource ที่สร้างแล้วแต่อาจไม่สมบูรณ์
# Terraform จะ destroy และ recreate ครั้งต่อไป

# ดู tainted resources:
terraform state list  # ดูว่ามี resource ไหนใน state บ้าง
```

### Resource Creation Order

```bash
# Terraform สร้าง resources ตาม dependency graph
# สามารถสร้างหลาย resources พร้อมกัน (parallel) ถ้าไม่ depend กัน

# ตัวอย่าง:
# aws_vpc.main -> สร้างก่อน
# aws_subnet.public[0] -\
# aws_subnet.public[1] -> สร้างพร้อมกัน (parallel) หลัง vpc พร้อม
# aws_subnet.public[2] -/
# aws_instance.web -> สร้างหลังสุด (depends on subnet)

# ดู dependency graph:
terraform graph | dot -Tpng > graph.png
```

---

## Apply ใน CI/CD Pipelines

### GitLab CI/CD - Full Pipeline

```yaml
# .gitlab-ci.yml
stages:
  - validate
  - plan
  - apply

variables:
  TF_ROOT: ${CI_PROJECT_DIR}
  TF_ADDRESS: ${CI_API_V4_URL}/projects/${CI_PROJECT_ID}/terraform/state/${CI_ENVIRONMENT_SLUG}

.terraform_init: &terraform_init
  before_script:
    - cd ${TF_ROOT}
    - terraform init
        -backend-config="address=${TF_ADDRESS}"
        -backend-config="lock_address=${TF_ADDRESS}/lock"
        -backend-config="unlock_address=${TF_ADDRESS}/lock"
        -backend-config="username=gitlab-ci-token"
        -backend-config="password=${CI_JOB_TOKEN}"
        -backend-config="lock_method=POST"
        -backend-config="unlock_method=DELETE"
        -backend-config="retry_wait_min=5"

validate:
  stage: validate
  image: hashicorp/terraform:1.6
  script:
    - terraform fmt -check -recursive
    - terraform validate
  rules:
    - if: $CI_PIPELINE_SOURCE == "merge_request_event"
    - if: $CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH

plan:
  stage: plan
  image: hashicorp/terraform:1.6
  <<: *terraform_init
  script:
    - terraform plan -out=plan.cache
  artifacts:
    paths:
      - plan.cache
    expire_in: 1 week
  rules:
    - if: $CI_PIPELINE_SOURCE == "merge_request_event"
    - if: $CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH
  environment:
    name: review/$CI_COMMIT_REF_SLUG

apply:
  stage: apply
  image: hashicorp/terraform:1.6
  <<: *terraform_init
  script:
    - terraform apply plan.cache
  dependencies:
    - plan
  rules:
    - if: $CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH
      when: manual
  environment:
    name: production
```

### Terraform Apply Notifications

```bash
# Script ที่ส่ง notification หลัง apply
#!/bin/bash

set -e

# Apply
terraform apply -auto-approve -input=false -no-color 2>&1 | tee apply.log
EXIT_CODE=${PIPESTATUS[0]}

# ส่ง Slack notification
if [ $EXIT_CODE -eq 0 ]; then
  STATUS="✅ SUCCESS"
  COLOR="good"
else
  STATUS="❌ FAILED"
  COLOR="danger"
fi

SUMMARY=$(tail -5 apply.log | tr '\n' '\\n')

curl -X POST "$SLACK_WEBHOOK_URL" \
  -H 'Content-type: application/json' \
  -d "{
    \"attachments\": [{
      \"color\": \"$COLOR\",
      \"title\": \"Terraform Apply $STATUS\",
      \"text\": \"Environment: $TF_ENVIRONMENT\\nBranch: $GITHUB_REF_NAME\",
      \"fields\": [{
        \"title\": \"Summary\",
        \"value\": \"$SUMMARY\"
      }]
    }]
  }"

exit $EXIT_CODE
```

---

## Plan & Apply Best Practices

```bash
# ✅ DO:
# 1. Always plan before apply
terraform plan -out=tfplan
terraform apply tfplan

# 2. Review plan output ก่อน apply เสมอ
# หา: destroy operations, replace operations

# 3. ใช้ -target เฉพาะ emergency
# 4. Save plans สำหรับ audit trail

# 5. CI/CD: plan ใน PR, apply เมื่อ merge
# 6. Production: ต้องการ approval ก่อน apply

# ❌ DON'T:
# - apply โดยไม่ plan ก่อน (ใน production)
# - ใช้ -auto-approve ใน production manually
# - apply ด้วย -target บ่อยๆ
# - ไม่ review destroy operations
```

### Plan Review Checklist

```bash
# ก่อน apply ตรวจสอบ:
# 1. จำนวน resources ที่จะ create/update/destroy
# 2. มี unexpected destroys หรือไม่?
# 3. มี -/+ (replace) ที่ไม่คาดไว้หรือไม่?
# 4. ค่า "forces replacement" fields
# 5. Resources ที่ sensitive (databases, etc.)

# Script ตรวจสอบ destroys
terraform plan -json | jq '
  .resource_changes[] 
  | select(.change.actions[] == "delete")
  | {
      resource: .address,
      type: .type
    }
'
```

---

## Command Summary

```bash
# === terraform plan ===
terraform plan                           # Basic plan
terraform plan -out=tfplan               # Save plan
terraform plan -var="key=value"          # Pass variable
terraform plan -var-file="prod.tfvars"   # Use var file
terraform plan -target=aws_instance.web  # Target specific resource
terraform plan -replace=aws_instance.web # Force replace
terraform plan -refresh-only             # Refresh state only
terraform plan -destroy                  # Plan destroy
terraform plan -compact-warnings         # Compact warnings
terraform plan -json                     # JSON output
terraform plan -no-color                 # No ANSI colors (CI)
terraform plan -parallelism=5            # Set parallelism

# === terraform apply ===
terraform apply                          # Interactive apply
terraform apply tfplan                   # Apply from plan file
terraform apply -auto-approve            # No confirmation
terraform apply -input=false             # No interactive input
terraform apply -var="key=value"         # Pass variable
terraform apply -var-file="prod.tfvars"  # Use var file
terraform apply -target=aws_instance.web # Target specific resource
terraform apply -replace=aws_instance.web # Force replace
terraform apply -parallelism=5           # Set parallelism
terraform apply -refresh=false           # Skip state refresh
terraform apply -no-color                # No ANSI colors (CI)
terraform apply -compact-warnings        # Compact warnings
```

---

## แบบฝึกหัด (Exercises)

### Exercise 1: Plan Analysis

1. สร้าง infrastructure ด้วย Terraform
2. เปลี่ยน instance_type
3. รัน `terraform plan` และ analyze output
4. หา resource ที่จะ update vs replace

### Exercise 2: Targeted Operations

1. สร้าง multiple resources
2. ใช้ `-target` เพื่อ update เฉพาะ 1 resource
3. Verify ว่า resources อื่นไม่ถูก affect

### Exercise 3: CI/CD Pipeline

สร้าง GitHub Actions workflow ที่:
1. Plan ใน PR
2. Comment plan result ใน PR
3. Apply อัตโนมัติเมื่อ merge ไป main

---

## Checklist

- [ ] เข้าใจ plan output symbols (+, -, ~, -/+)
- [ ] รู้ "(known after apply)" หมายถึงอะไร
- [ ] สามารถ save และ apply plan files ได้
- [ ] รู้วิธีใช้ -var และ -var-file
- [ ] เข้าใจ -target flag และข้อควรระวัง
- [ ] รู้วิธีใช้ -auto-approve ใน CI/CD
- [ ] เข้าใจ error handling ใน apply
- [ ] สามารถ integrate plan/apply ใน CI/CD ได้
- [ ] รู้วิธี review plan ก่อน apply
