# Part 032: Terraform Refresh & Reconciliation
# Terraform Refresh และการ Reconcile State กับ Reality

## Steps 311-320: ทำความเข้าใจการ Refresh และจัดการ Drift

---

## Step 311: terraform refresh คืออะไร? (และทำไมถึง Deprecated)

### ความหมายของ terraform refresh

`terraform refresh` เป็นคำสั่งที่ **อ่านสถานะจริงของ Infrastructure** แล้วอัปเดต state file ให้ตรงกับความเป็นจริง โดยไม่ได้ทำการเปลี่ยนแปลง infrastructure จริงๆ

### ทำไม terraform refresh ถึง Deprecated?

ใน Terraform v0.15.4+ คำสั่ง `terraform refresh` ถูก **deprecated** เพราะ:

1. **ไม่ปลอดภัย**: อาจ overwrite state โดยไม่ตั้งใจ
2. **ไม่มี review step**: ไม่แสดงการเปลี่ยนแปลงก่อนทำ
3. **อันตราย**: ถ้า provider มี bug อาจลบ resources ออกจาก state

### แทนที่ด้วยอะไร?

```bash
# ❌ Deprecated (อย่าใช้)
terraform refresh

# ✅ ใช้แทน: plan -refresh-only (ดูก่อน)
terraform plan -refresh-only

# ✅ ใช้แทน: apply -refresh-only (ทำจริงหลังดูแล้ว)
terraform apply -refresh-only
```

---

## Step 312: terraform plan -refresh-only

### การทำงานของ plan -refresh-only

```bash
# รัน refresh-only plan
terraform plan -refresh-only

# Output ตัวอย่าง:
# aws_instance.web: Refreshing state... [id=i-0abc123def456]
# aws_vpc.main: Refreshing state... [id=vpc-0abc123]
#
# Note: Objects have changed outside of Terraform
#
# aws_instance.web has been modified:
#   ~ resource "aws_instance" "web" {
#       ~ instance_state = "running" -> "stopped"
#         ...
#     }
#
# This plan will update the Terraform state to reflect changes made
# to these objects outside of Terraform.
```

### วิธีใช้ plan -refresh-only

```bash
# พื้นฐาน
terraform plan -refresh-only

# บันทึก plan สำหรับ review
terraform plan -refresh-only -out=refresh.tfplan

# ดู plan ที่บันทึกไว้
terraform show refresh.tfplan

# ดู plan แบบ JSON
terraform show -json refresh.tfplan | jq .

# กำหนด target
terraform plan -refresh-only -target=aws_instance.web
```

### ตัวอย่าง Workflow

```bash
# 1. สร้าง refresh plan
terraform plan -refresh-only -out=refresh.tfplan

# 2. ตรวจสอบว่ามีอะไรเปลี่ยนแปลง
terraform show refresh.tfplan

# 3. ถ้า OK ทำการ apply
terraform apply refresh.tfplan

# 4. ตรวจสอบ state หลัง refresh
terraform show
```

---

## Step 313: terraform apply -refresh-only

### การใช้ apply -refresh-only

```bash
# Apply refresh-only (อัปเดต state เพื่อให้ตรงกับ reality)
terraform apply -refresh-only

# Output ตัวอย่าง:
# aws_instance.web: Refreshing state... [id=i-0abc123def456]
#
# Note: Objects have changed outside of Terraform
#
# ~ aws_instance.web
#   ~ tags = {
#       + "ManualTag" = "AddedManually"
#     }
#
# Do you want to update the Terraform state to reflect these changes?
# Terraform will update the state without modifying the infrastructure.
# Only 'yes' will be accepted to confirm.
#
# Enter a value: yes
#
# Apply complete! Resources: 0 added, 0 changed, 0 destroyed.
```

### ตัวอย่าง apply -refresh-only ใน Automation

```bash
#!/bin/bash
# scripts/refresh_state.sh

set -e

echo "🔄 Starting Terraform state refresh..."

# Initialize
terraform init -input=false

# Create refresh plan
terraform plan -refresh-only -out=refresh.tfplan -input=false

# Check if there are any changes
CHANGES=$(terraform show -json refresh.tfplan | jq '.resource_changes | length')

if [ "$CHANGES" -gt 0 ]; then
  echo "⚠️  Found $CHANGES resources with drift"
  echo "📋 Reviewing changes..."
  terraform show refresh.tfplan
  
  # Apply refresh (อัปเดต state ให้ตรงกับ reality)
  terraform apply -input=false refresh.tfplan
  echo "✅ State refresh complete"
else
  echo "✅ No drift detected - state is up to date"
fi

# Cleanup
rm -f refresh.tfplan
```

---

## Step 314: What Refresh Does (Queries Real Infrastructure)

### กระบวนการ Refresh

```
┌─────────────────────────────────────────────────────────┐
│                   terraform refresh process               │
│                                                           │
│  1. Read State File                                       │
│     └─ ดู resources ที่ Terraform รู้จัก                  │
│                                                           │
│  2. Query Real Infrastructure (ผ่าน Provider API)        │
│     └─ เรียก AWS API เพื่อดูสถานะจริง                    │
│                                                           │
│  3. Compare                                               │
│     └─ เปรียบเทียบ state กับ reality                     │
│                                                           │
│  4. Update State                                          │
│     └─ อัปเดต state file ให้ตรงกับ reality              │
│                                                           │
│  ⚠️  ไม่ได้เปลี่ยน Infrastructure จริงๆ                  │
└─────────────────────────────────────────────────────────┘
```

### Provider API Calls ที่เกิดขึ้น

```
ตัวอย่าง AWS Provider Refresh:

aws_instance.web:
  → AWS API: DescribeInstances(i-0abc123)
  → ได้รับ: instance state, tags, security groups, etc.
  → อัปเดต state

aws_vpc.main:
  → AWS API: DescribeVpcs(vpc-0abc123)
  → ได้รับ: CIDR, DNS settings, etc.
  → อัปเดต state

aws_s3_bucket.data:
  → AWS API: GetBucketLocation, GetBucketVersioning, etc.
  → ได้รับ: bucket configuration
  → อัปเดต state
```

### Attributes ที่ Refresh อัปเดต

```hcl
# ตัวอย่าง EC2 Instance attributes ที่อาจเปลี่ยนแปลง

resource "aws_instance" "web" {
  ami           = "ami-0c02fb55956c7d316"
  instance_type = "t3.micro"
  
  # Attributes ที่ Terraform manage:
  # - ami ✓
  # - instance_type ✓
  # - tags ✓
  
  # Attributes ที่อาจเปลี่ยนนอก Terraform:
  # - instance_state (running/stopped/terminated)
  # - public_ip (Elastic IP อาจเปลี่ยน)
  # - tags (ถ้ามีคนเพิ่ม manual)
  # - security_groups (ถ้ามีคนเปลี่ยน manual)
}
```

---

## Step 315: When State Diverges From Reality (Drift)

### State Drift คืออะไร?

**State Drift** (หรือ Configuration Drift) เกิดขึ้นเมื่อ:
- สถานะใน **Terraform state file** ≠ สถานะจริงใน **Cloud infrastructure**

### สาเหตุของ Drift

```
สาเหตุที่พบบ่อย:

1. การเปลี่ยนแปลงด้วยมือ (Manual Changes)
   - เปิด AWS Console แล้วเปลี่ยน settings
   - ใช้ AWS CLI โดยตรง
   - ใช้ SDK โดยตรง

2. การเปลี่ยนแปลงโดย AWS เอง (AWS-managed changes)
   - Auto Scaling เพิ่ม/ลด instances
   - AWS อัปเดต AMI
   - Certificate renewal

3. External Automation
   - Scripts อื่นๆ ที่ไม่ใช่ Terraform
   - Config management tools (Ansible, Chef)
   - CI/CD ที่ไม่ใช้ Terraform

4. Terraform ทำงานไม่สำเร็จ
   - Apply หยุดกลางคัน
   - Network timeout ระหว่าง apply
```

### ตัวอย่าง Drift ที่พบบ่อย

```hcl
# Terraform state บอกว่า:
resource "aws_instance" "web" {
  instance_type = "t3.micro"    # ใน state: t3.micro
  
  tags = {
    Name = "web-server"         # ใน state: มีแค่ Name tag
  }
}

# Reality (ใน AWS console):
# instance_type = "t3.small"   ← มีคนเปลี่ยนผ่าน console!
# tags = {
#   Name       = "web-server"
#   Department = "Engineering"  ← มีคนเพิ่ม tag!
#   CostCenter = "IT-001"       ← มีคนเพิ่ม tag!
# }
```

---

## Step 316: Detecting Drift

### วิธีตรวจจับ Drift

```bash
# วิธีที่ 1: terraform plan (วิธีพื้นฐาน)
terraform plan
# ถ้ามี drift จะแสดง changes ที่ Terraform จะทำเพื่อ fix it

# วิธีที่ 2: terraform plan -refresh-only (ดู drift อย่างเดียว)
terraform plan -refresh-only
# แสดงเฉพาะ drift ไม่แสดง configuration changes

# วิธีที่ 3: terraform show (ดู current state)
terraform show

# วิธีที่ 4: terraform state show (ดู specific resource)
terraform state show aws_instance.web
```

### Script สำหรับ Automated Drift Detection

```bash
#!/bin/bash
# scripts/detect_drift.sh - ตรวจจับ drift อัตโนมัติ

set -e

WORKSPACE=${1:-default}
NOTIFY_SLACK=${NOTIFY_SLACK:-false}
SLACK_WEBHOOK=${SLACK_WEBHOOK:-""}

echo "🔍 Detecting infrastructure drift for workspace: $WORKSPACE"

# Switch to workspace
terraform workspace select $WORKSPACE

# Initialize
terraform init -input=false -no-color > /dev/null 2>&1

# Run refresh-only plan
PLAN_OUTPUT=$(terraform plan -refresh-only -no-color 2>&1)
EXIT_CODE=$?

# Check for drift
if echo "$PLAN_OUTPUT" | grep -q "Objects have changed outside of Terraform"; then
  echo "⚠️  DRIFT DETECTED in workspace: $WORKSPACE"
  
  # Count changed resources
  CHANGED=$(echo "$PLAN_OUTPUT" | grep -c "has been modified\|has been deleted\|has been created" || true)
  echo "📊 Number of drifted resources: $CHANGED"
  
  # Show details
  echo ""
  echo "Drift details:"
  echo "$PLAN_OUTPUT" | grep -A 20 "Objects have changed" | head -50
  
  # Notify Slack if configured
  if [ "$NOTIFY_SLACK" = "true" ] && [ -n "$SLACK_WEBHOOK" ]; then
    curl -X POST "$SLACK_WEBHOOK" \
      -H 'Content-type: application/json' \
      --data "{
        \"text\": \"⚠️ Infrastructure Drift Detected!\",
        \"blocks\": [
          {
            \"type\": \"section\",
            \"text\": {
              \"type\": \"mrkdwn\",
              \"text\": \"*⚠️ Drift Detected in Workspace: $WORKSPACE*\n$CHANGED resources have drifted from expected state.\"
            }
          }
        ]
      }"
  fi
  
  exit 1
else
  echo "✅ No drift detected - infrastructure matches Terraform state"
  exit 0
fi
```

### Drift Detection ใน GitHub Actions

```yaml
# .github/workflows/drift-detection.yml
name: Drift Detection

on:
  schedule:
    - cron: '0 */6 * * *'  # รัน ทุก 6 ชั่วโมง
  workflow_dispatch:         # รัน manual ได้

jobs:
  detect-drift:
    name: Detect Infrastructure Drift
    runs-on: ubuntu-latest
    
    permissions:
      id-token: write
      contents: read
      issues: write
    
    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Configure AWS Credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ secrets.AWS_ROLE_ARN }}
          aws-region: ap-southeast-1

      - name: Setup Terraform
        uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: "1.6.0"
          terraform_wrapper: false

      - name: Terraform Init
        run: terraform init -input=false

      - name: Check for Drift
        id: drift_check
        run: |
          terraform plan -refresh-only -detailed-exitcode -no-color \
            -out=refresh.tfplan 2>&1 | tee drift_output.txt
          echo "exit_code=$?" >> $GITHUB_OUTPUT
        continue-on-error: true

      - name: Parse Drift Results
        id: parse_drift
        run: |
          if [ "${{ steps.drift_check.outputs.exit_code }}" = "2" ]; then
            echo "drift_detected=true" >> $GITHUB_OUTPUT
            CHANGED=$(grep -c "has been modified\|will be updated\|will be destroyed" drift_output.txt || echo "0")
            echo "changed_resources=$CHANGED" >> $GITHUB_OUTPUT
          else
            echo "drift_detected=false" >> $GITHUB_OUTPUT
            echo "changed_resources=0" >> $GITHUB_OUTPUT
          fi

      - name: Create Drift Alert Issue
        if: steps.parse_drift.outputs.drift_detected == 'true'
        uses: actions/github-script@v6
        with:
          script: |
            const changedResources = '${{ steps.parse_drift.outputs.changed_resources }}';
            const driftContent = require('fs').readFileSync('drift_output.txt', 'utf8');
            
            github.rest.issues.create({
              owner: context.repo.owner,
              repo: context.repo.repo,
              title: `⚠️ Infrastructure Drift Detected - ${new Date().toISOString().split('T')[0]}`,
              body: `## Infrastructure Drift Alert
              
**${changedResources} resources have drifted** from expected Terraform state.

### Details
\`\`\`
${driftContent.substring(0, 3000)}
\`\`\`

### Actions Required
1. Review the drift details above
2. Decide how to handle (accept/revert/ignore)
3. Run \`terraform apply -refresh-only\` to accept OR \`terraform apply\` to revert

### Commands
\`\`\`bash
# Accept drift (update state to match reality)
terraform apply -refresh-only

# Revert drift (apply Terraform config to override manual changes)  
terraform apply

# Ignore specific changes
# Add lifecycle.ignore_changes to resource
\`\`\`
`,
              labels: ['infrastructure-drift', 'terraform']
            });
```

---

## Step 317: Handling Drift - 3 Strategies

### กลยุทธ์ที่ 1: Accept (รับ drift เข้า state)

ใช้เมื่อ: การเปลี่ยนแปลงที่เกิดขึ้นนอก Terraform นั้น **ถูกต้องและต้องการ**

```bash
# ขั้นตอน:
# 1. ดูว่ามี drift อะไรบ้าง
terraform plan -refresh-only

# 2. ถ้า drift ที่พบนั้น OK
terraform apply -refresh-only

# ผลลัพธ์: State อัปเดตให้ตรงกับ reality
# Infrastructure: ไม่เปลี่ยนแปลง
```

```hcl
# ตัวอย่าง: Tags ที่เพิ่มโดย Security team
# Reality: instance มี tag Security=Scanned
# State: ไม่มี tag นี้

# หลัง terraform apply -refresh-only:
# State จะมี:
resource "aws_instance" "web" {
  # ...
  tags = {
    Name     = "web-server"
    Security = "Scanned"  # ← เพิ่มเข้า state แล้ว
  }
}
```

### กลยุทธ์ที่ 2: Revert (คืน infrastructure ให้ตรง config)

ใช้เมื่อ: การเปลี่ยนแปลงนั้น **ไม่ควรเกิดขึ้น** และต้องการยืนยัน configuration

```bash
# ขั้นตอน:
# 1. ดู drift
terraform plan -refresh-only

# 2. ดู plan ที่ Terraform จะทำเพื่อ revert
terraform plan

# 3. Apply เพื่อคืนค่าตาม configuration
terraform apply

# ผลลัพธ์: Infrastructure กลับมาตรงกับ Terraform config
# State: ตรงกับ Infrastructure ที่ตรงกับ Config
```

### กลยุทธ์ที่ 3: Ignore (ไม่สนใจ drift บางอย่าง)

ใช้เมื่อ: บาง attribute ที่เปลี่ยนแปลงนั้น **ไม่ต้องการ** ให้ Terraform manage

```hcl
# ใช้ lifecycle.ignore_changes

resource "aws_instance" "web" {
  ami           = "ami-0c02fb55956c7d316"
  instance_type = "t3.micro"

  tags = {
    Name = "web-server"
  }

  lifecycle {
    # ไม่สนใจการเปลี่ยนแปลง tags และ AMI
    # เช่น Security team เพิ่ม tags เอง
    # หรือมีการอัปเดต AMI ผ่านกลไกอื่น
    ignore_changes = [
      tags,
      ami,
    ]
  }
}
```

```hcl
# ตัวอย่างการใช้ ignore_changes ที่พบบ่อย

# 1. ASG ที่ Auto Scaling ปรับ desired_capacity
resource "aws_autoscaling_group" "app" {
  name                = "app-asg"
  desired_capacity    = 2
  max_size            = 10
  min_size            = 1
  vpc_zone_identifier = var.subnet_ids

  lifecycle {
    ignore_changes = [desired_capacity]  # ปล่อยให้ Auto Scaling จัดการ
  }
}

# 2. RDS ที่ AWS อาจอัปเดต engine_version minor version
resource "aws_db_instance" "main" {
  identifier     = "main-db"
  engine         = "postgres"
  engine_version = "15.4"
  
  lifecycle {
    ignore_changes = [engine_version]  # ปล่อยให้ AWS จัดการ minor updates
  }
}

# 3. ECS Service ที่ deployment pipeline อัปเดต task definition
resource "aws_ecs_service" "app" {
  name            = "app-service"
  cluster         = aws_ecs_cluster.main.id
  task_definition = aws_ecs_task_definition.app.arn
  desired_count   = 2

  lifecycle {
    ignore_changes = [task_definition, desired_count]  # CD pipeline manages these
  }
}

# 4. S3 Bucket ACL ที่ AWS อาจเปลี่ยน
resource "aws_s3_bucket" "data" {
  bucket = "my-data-bucket"
  
  lifecycle {
    # ป้องกันการลบ bucket โดยไม่ตั้งใจ
    prevent_destroy = true
  }
}

# 5. Launch Template ที่ต้องการ create_before_destroy
resource "aws_launch_template" "app" {
  name_prefix   = "app-"
  image_id      = data.aws_ami.amazon_linux.id
  instance_type = "t3.micro"

  lifecycle {
    create_before_destroy = true  # สร้างอันใหม่ก่อนลบอันเก่า
  }
}
```

---

## Step 318: Refresh และ Performance

### ปัญหา Performance ของ Refresh

เมื่อมี resources จำนวนมาก การ refresh จะช้าเพราะต้องเรียก API ทุก resource

```
ตัวอย่าง Performance:
- 10 resources:   ~5 วินาที
- 100 resources:  ~30 วินาที
- 500 resources:  ~2-3 นาที
- 1000 resources: ~5-10 นาที
```

### -refresh=false เพื่อเพิ่มความเร็ว

```bash
# Skip refresh เพื่อความเร็ว
terraform plan -refresh=false

# เมื่อไหรที่ใช้ -refresh=false?
# 1. ต้องการดู config diff อย่างเดียว (ไม่สนใจ drift)
# 2. ใน development ที่รู้ว่า state ตรง
# 3. เมื่อทำ testing configuration ก่อน apply จริง

terraform apply -refresh=false
```

### เปรียบเทียบ: refresh=true vs refresh=false

```
                  refresh=true          refresh=false
                  ─────────────         ─────────────
เวลา:             ช้ากว่า               เร็วกว่ามาก
ความแม่นยำ:      สูง (real state)       ต่ำกว่า (cached state)
API calls:        มาก                   น้อย
ใช้เมื่อ:         production apply      quick dev iteration
```

### Selective Refresh

```bash
# Refresh เฉพาะบาง resources
terraform plan -target=aws_instance.web
terraform apply -target=aws_instance.web

# ดู specific resource ใน state
terraform state show aws_instance.web

# Refresh specific resource manually
terraform refresh -target=aws_instance.web  # deprecated แต่ยังใช้ได้
```

---

## Step 319: Out-of-band Changes Handling

### Out-of-band Changes คืออะไร?

การเปลี่ยนแปลงที่เกิดขึ้น **นอก Terraform** เช่น:

1. **Manual changes** ผ่าน AWS Console
2. **AWS-initiated changes** (Security patches, auto-updates)
3. **Other tools** (CloudFormation, CDK, Pulumi)
4. **Automated processes** (Lambda functions, EventBridge rules)

### ตัวอย่างสถานการณ์และวิธีจัดการ

```bash
# สถานการณ์ที่ 1: Tags ถูกเพิ่มโดย Cloud Security tool

# ตรวจจับ:
terraform plan -refresh-only
# > aws_instance.web has been modified:
# >   ~ tags = {
# >       + "security-scanned" = "2024-01-15"
# >     }

# วิธีจัดการ: Accept - เพราะ security team ต้องการ tags เหล่านี้
terraform apply -refresh-only
```

```bash
# สถานการณ์ที่ 2: Instance type เปลี่ยนโดยไม่ได้รับอนุญาต

# ตรวจจับ:
terraform plan -refresh-only  
# > aws_instance.web has been modified:
# >   ~ instance_type = "t3.small" -> "t3.large"  # ← ใครเปลี่ยน?!

# วิธีจัดการ: Revert - ไม่ควรมีการเปลี่ยนแปลงนี้
terraform apply  # จะ revert กลับเป็น t3.micro
```

```bash
# สถานการณ์ที่ 3: Security Group rule ถูกเพิ่มฉุกเฉิน

# ตรวจจับ:
terraform plan -refresh-only
# > aws_security_group.web has been modified:
# >   ~ ingress rules include port 8080 (was not in config)

# วิธีจัดการ:
# Option A: ถ้าต้องการ rule นี้ถาวร -> เพิ่มใน Terraform config แล้ว apply
# Option B: ถ้าเป็น temporary -> ignore_changes หรือ apply เพื่อ revert
```

### การป้องกัน Out-of-band Changes

```hcl
# 1. ใช้ SCP (Service Control Policies) ใน AWS Organizations
# เพื่อป้องกัน manual changes

# 2. ใช้ IAM Policies ที่จำกัดการเข้าถึง
# เฉพาะ Terraform service role ที่สามารถเปลี่ยนแปลงได้

# 3. Enable AWS Config Rules
# เพื่อตรวจจับและ alert เมื่อมีการเปลี่ยนแปลง

# 4. CloudTrail monitoring
# บันทึกทุกการ API call

# 5. Terraform Lock File (ป้องกัน concurrent applies)
```

---

## Step 320: Automated Drift Detection ใน CI/CD

### ระบบ Automated Drift Detection

```yaml
# .github/workflows/drift-detection-advanced.yml
name: Advanced Drift Detection

on:
  schedule:
    - cron: '0 8,12,16,20 * * 1-5'  # ทุก 4 ชั่วโมงในวันทำงาน
  workflow_dispatch:
    inputs:
      environment:
        description: 'Environment to check'
        required: true
        default: 'production'
        type: choice
        options:
          - production
          - staging
          - development

env:
  TF_VERSION: "1.6.0"
  ENVIRONMENT: ${{ github.event.inputs.environment || 'production' }}

jobs:
  detect-drift:
    name: Detect Drift (${{ github.event.inputs.environment || 'production' }})
    runs-on: ubuntu-latest
    
    permissions:
      id-token: write
      contents: read
      issues: write
      pull-requests: write

    steps:
      - name: Checkout Repository
        uses: actions/checkout@v4

      - name: Configure AWS Credentials via OIDC
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ secrets[format('AWS_ROLE_{0}', env.ENVIRONMENT)] }}
          aws-region: ap-southeast-1

      - name: Setup Terraform
        uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: ${{ env.TF_VERSION }}
          terraform_wrapper: false

      - name: Terraform Init
        run: |
          cd environments/${{ env.ENVIRONMENT }}
          terraform init -input=false -no-color

      - name: Run Drift Detection
        id: drift
        run: |
          cd environments/${{ env.ENVIRONMENT }}
          
          # Run refresh-only plan
          set +e
          terraform plan -refresh-only \
            -detailed-exitcode \
            -no-color \
            -out=drift.tfplan \
            2>&1 | tee drift_output.txt
          EXIT_CODE=${PIPESTATUS[0]}
          set -e
          
          echo "exit_code=$EXIT_CODE" >> $GITHUB_OUTPUT
          
          case $EXIT_CODE in
            0) 
              echo "status=no_drift" >> $GITHUB_OUTPUT
              echo "✅ No drift detected"
              ;;
            1) 
              echo "status=error" >> $GITHUB_OUTPUT
              echo "❌ Terraform plan error"
              exit 1
              ;;
            2) 
              echo "status=drift_detected" >> $GITHUB_OUTPUT
              echo "⚠️ Drift detected!"
              
              # Count resources
              MODIFIED=$(grep -c "~ resource\|will be updated" drift_output.txt || echo "0")
              DELETED=$(grep -c "- resource\|will be destroyed" drift_output.txt || echo "0")
              echo "modified_count=$MODIFIED" >> $GITHUB_OUTPUT
              echo "deleted_count=$DELETED" >> $GITHUB_OUTPUT
              ;;
          esac

      - name: Generate Drift Report
        if: steps.drift.outputs.status == 'drift_detected'
        run: |
          cd environments/${{ env.ENVIRONMENT }}
          
          cat > drift_report.md << 'EOF'
          # 🚨 Infrastructure Drift Report
          
          **Environment**: ${{ env.ENVIRONMENT }}
          **Detected at**: $(date -u '+%Y-%m-%d %H:%M UTC')
          **Modified Resources**: ${{ steps.drift.outputs.modified_count }}
          **Deleted Resources**: ${{ steps.drift.outputs.deleted_count }}
          
          ## Drift Details
          
          \`\`\`
          $(cat drift_output.txt | grep -A 50 "Objects have changed" | head -100)
          \`\`\`
          
          ## Recommended Actions
          
          1. **Review** the drift details above
          2. **Investigate** who/what made these changes
          3. **Decide** on action:
             - Accept: `terraform apply -refresh-only`
             - Revert: `terraform apply`  
             - Ignore: Add `lifecycle.ignore_changes`
          EOF

      - name: Post Drift Summary to GitHub
        if: steps.drift.outputs.status == 'drift_detected'
        uses: actions/github-script@v6
        with:
          script: |
            const fs = require('fs');
            
            // Create issue for drift
            const issue = await github.rest.issues.create({
              owner: context.repo.owner,
              repo: context.repo.repo,
              title: `⚠️ Drift Detected: ${process.env.ENVIRONMENT} - ${new Date().toISOString().split('T')[0]}`,
              body: `## Infrastructure Drift Alert
              
**Environment**: ${process.env.ENVIRONMENT}
**Modified Resources**: ${{ steps.drift.outputs.modified_count }}
**Deleted Resources**: ${{ steps.drift.outputs.deleted_count }}

### Quick Actions
- [View Workflow Run](${context.serverUrl}/${context.repo.owner}/${context.repo.repo}/actions/runs/${context.runId})

### How to Resolve
\`\`\`bash
# 1. Review drift
terraform plan -refresh-only

# 2a. Accept drift (if changes are valid)
terraform apply -refresh-only

# 2b. Revert drift (if changes should not have happened)
terraform apply

# 2c. Ignore specific attributes
# Add lifecycle.ignore_changes to affected resources
\`\`\`
`,
              labels: ['drift-detected', 'infrastructure', process.env.ENVIRONMENT]
            });
            
            console.log(`Issue created: ${issue.data.html_url}`);

      - name: Notify Slack on Drift
        if: steps.drift.outputs.status == 'drift_detected'
        run: |
          curl -X POST "${{ secrets.SLACK_WEBHOOK }}" \
            -H 'Content-type: application/json' \
            --data '{
              "blocks": [
                {
                  "type": "header",
                  "text": {
                    "type": "plain_text",
                    "text": "⚠️ Infrastructure Drift Detected"
                  }
                },
                {
                  "type": "section",
                  "fields": [
                    {
                      "type": "mrkdwn",
                      "text": "*Environment:*\n${{ env.ENVIRONMENT }}"
                    },
                    {
                      "type": "mrkdwn",
                      "text": "*Modified:*\n${{ steps.drift.outputs.modified_count }} resources"
                    }
                  ]
                }
              ]
            }'
```

### Drift Detection กับ Multiple Environments

```bash
#!/bin/bash
# scripts/drift_check_all_envs.sh

ENVIRONMENTS=("development" "staging" "production")
DRIFT_FOUND=false

for ENV in "${ENVIRONMENTS[@]}"; do
  echo "Checking $ENV..."
  
  cd "environments/$ENV"
  terraform init -input=false -no-color > /dev/null 2>&1
  
  set +e
  terraform plan -refresh-only -detailed-exitcode -no-color > /dev/null 2>&1
  EXIT_CODE=$?
  set -e
  
  if [ $EXIT_CODE -eq 2 ]; then
    echo "  ⚠️  DRIFT detected in $ENV"
    DRIFT_FOUND=true
  elif [ $EXIT_CODE -eq 0 ]; then
    echo "  ✅ No drift in $ENV"
  else
    echo "  ❌ Error checking $ENV"
  fi
  
  cd ../..
done

if $DRIFT_FOUND; then
  echo ""
  echo "❌ Drift detected in one or more environments!"
  exit 1
else
  echo ""
  echo "✅ All environments are in sync with Terraform state"
fi
```

---

## สรุป: Refresh & Reconciliation Best Practices

### เมื่อไหรที่ควรใช้คำสั่งไหน?

| สถานการณ์ | คำสั่ง |
|-----------|--------|
| ดู drift อย่างเดียว | `terraform plan -refresh-only` |
| รับ drift เข้า state | `terraform apply -refresh-only` |
| ยืนยัน configuration (revert drift) | `terraform apply` |
| ข้าม refresh เพื่อความเร็ว | `terraform plan -refresh=false` |
| ไม่สนใจ attribute บางอย่าง | `lifecycle.ignore_changes` |

### Workflow การจัดการ Drift

```
Scheduled Drift Detection
          │
          ▼
   Drift Detected?
    ├─── No ──→ ✅ All good!
    └─── Yes
          │
          ▼
   Analyze Changes
          │
    ┌─────┼──────┐
    │     │      │
    ▼     ▼      ▼
Accept  Revert  Ignore
  │       │       │
apply   apply  lifecycle
-refresh-       .ignore_
only           changes
```

---

*จบ Part 032: Terraform Refresh & Reconciliation*

*ต่อไป: Part 033 - Terraform Console & Expressions*
