# Part 080: Terraform Drift Detection (ขั้นตอนที่ 791-800)

## บทนำ (Introduction)

Infrastructure Drift คือสภาวะที่ actual infrastructure แตกต่างจาก state ที่ Terraform manage
การตรวจจับและจัดการ drift เป็นสิ่งสำคัญมากในองค์กรที่ใช้ Infrastructure as Code

---

## ขั้นตอนที่ 791: What is Infrastructure Drift?

### นิยามของ Drift

```
Drift = ความแตกต่างระหว่าง:
- Terraform configuration (desired state)
- Actual infrastructure (current state)

เกิดเมื่อ:
1. มีการ change infrastructure โดยตรง (AWS Console, CLI)
2. Infrastructure เปลี่ยนเองอัตโนมัติ (Auto Scaling, patches)
3. Partial apply (apply ไม่สำเร็จครบ)
4. External automation เปลี่ยนแปลง
```

### ตัวอย่าง Drift Scenarios

```
Scenario 1: Emergency Manual Change
- Production หยุดทำงาน
- Engineer เพิ่ม security group rule ผ่าน Console
- ลืม update Terraform code
→ Security group rule ใน state ≠ ใน AWS

Scenario 2: Auto Scaling
- ASG เพิ่ม instance จาก 2 → 5 ตอน traffic สูง
- Terraform ยังคิดว่ามี 2 instances
→ Instance count ใน state ≠ ใน AWS

Scenario 3: AWS-Managed Changes
- AWS update security patches บน managed services
- RDS minor version เปลี่ยน
→ RDS version ใน state ≠ ใน AWS

Scenario 4: Incomplete Apply
- Network error ระหว่าง apply
- บาง resources create สำเร็จ บางตัวไม่สำเร็จ
→ State ไม่ตรงกับ actual state
```

---

## ขั้นตอนที่ 792: Types of Drift

### 1. Manual Changes Outside Terraform

```bash
# ตัวอย่าง: แก้ไข Security Group ผ่าน AWS Console
# ใน Console: เพิ่ม inbound rule port 8080

# Terraform ตรวจพบ:
$ terraform plan

  # aws_security_group.web will be updated in-place
  ~ resource "aws_security_group" "web" {
      ~ ingress = [
          - {                                 # ← Terraform ต้องการลบ rule นี้
              - from_port   = 8080
              - to_port     = 8080
              - protocol    = "tcp"
              - cidr_blocks = ["0.0.0.0/0"]
            },
            # ... existing rules
        ]
    }
```

### 2. Auto-Scaling Changes

```hcl
# Configuration
resource "aws_autoscaling_group" "app" {
  desired_capacity = 3  # กำหนดไว้ 3
  min_size         = 1
  max_size         = 10
}

# แต่ ASG ปรับเป็น 7 ตอน peak hours
# Terraform plan จะเห็น:
# desired_capacity: 7 → 3 (Terraform ต้องการลดลง)
```

```hcl
# แก้ปัญหาด้วย ignore_changes
resource "aws_autoscaling_group" "app" {
  desired_capacity = 3
  min_size         = 1
  max_size         = 10

  lifecycle {
    ignore_changes = [desired_capacity]  # ไม่ revert การ scale
  }
}
```

### 3. System-Generated Changes

```hcl
# AWS managed RDS - version อาจ update อัตโนมัติ
resource "aws_db_instance" "main" {
  engine         = "mysql"
  engine_version = "8.0.32"  # อาจ auto-update เป็น 8.0.36

  # แก้ปัญหา:
  lifecycle {
    ignore_changes = [engine_version]
  }
}

# AWS Certificate Manager - validation status เปลี่ยน
resource "aws_acm_certificate" "main" {
  domain_name = "myapp.com"
  # status เปลี่ยนจาก PENDING_VALIDATION → ISSUED

  lifecycle {
    ignore_changes = [status]
  }
}
```

### 4. Partial Applies

```bash
# apply ที่ไม่สำเร็จ
$ terraform apply
  aws_security_group.web: Creating... done
  aws_instance.web[0]: Creating... done
  aws_instance.web[1]: Creating... Error!  ← หยุดตรงนี้

# State แสดงว่า create แค่ [0] แต่ไม่มี [1]
# AWS มี instance [0] แต่ไม่มี [1]
# → Partial success = drift
```

---

## ขั้นตอนที่ 793: Detecting Drift

### terraform plan -refresh-only

```bash
# ตรวจสอบ drift โดยไม่ apply configuration changes
terraform plan -refresh-only

# ตัวอย่าง output เมื่อพบ drift:
Note: Objects have changed outside of Terraform

Terraform detected the following changes made outside of Terraform
since the last "terraform apply" which may have affected this plan:

  # aws_security_group.web has been changed
  ~ resource "aws_security_group" "web" {
        id     = "sg-0abc123def"
      ~ ingress = [
          + {
              + from_port   = 8080
              + to_port     = 8080
              + protocol    = "tcp"
              + cidr_blocks = ["0.0.0.0/0"]
            },
          # ...
        ]
    }

This is a drift detection run. No changes will be made to your infrastructure.
```

### ตัวอย่าง Script ตรวจสอบ Drift

```bash
#!/bin/bash
# detect_drift.sh
# ตรวจสอบ drift สำหรับทุก environments

ENVIRONMENTS=("dev" "staging" "prod")
DRIFT_FOUND=false

for ENV in "${ENVIRONMENTS[@]}"; do
  echo "=== Checking $ENV environment ==="
  
  cd "environments/$ENV"
  terraform init -input=false > /dev/null 2>&1
  
  # รัน refresh-only plan
  PLAN_OUTPUT=$(terraform plan -refresh-only -no-color 2>&1)
  EXIT_CODE=$?
  
  if echo "$PLAN_OUTPUT" | grep -q "Objects have changed outside of Terraform"; then
    echo "⚠️  DRIFT DETECTED in $ENV!"
    echo "$PLAN_OUTPUT" | grep -A 20 "has been changed"
    DRIFT_FOUND=true
  else
    echo "✓ No drift in $ENV"
  fi
  
  cd ../..
done

if [ "$DRIFT_FOUND" = true ]; then
  echo ""
  echo "❌ Drift was detected. Review and remediate."
  exit 1
else
  echo ""
  echo "✓ All environments are drift-free."
  exit 0
fi
```

---

## ขั้นตอนที่ 794: Handling Drift Strategies

### Strategy 1: Accept Drift (terraform apply -refresh-only)

```bash
# เมื่อ drift ถูกต้อง และต้องการ update state ให้ตรงกับ reality
# เช่น: มีการ manual fix ที่ถูกต้อง และต้องการ record ใน state

terraform apply -refresh-only

# Output:
# Terraform will update your state to reflect the current configuration
# of these objects, and make adjustments to the drift-detection baseline
# going forward.

# Plan: 0 to add, 0 to change, 0 to destroy.
# Changes to Outputs: (none)
# Would you like to update the Terraform state to reflect these detected changes?
#   Terraform will write these changes to the state without modifying any real infrastructure.
#   There is no undo. Only 'yes' will be accepted to confirm.

# Enter value: yes
```

### Strategy 2: Revert Drift (terraform apply)

```bash
# เมื่อ drift ไม่ถูกต้อง และต้องการให้ infrastructure ตรงกับ config
terraform apply

# Terraform จะ revert การ change ทั้งหมดที่ทำนอก Terraform
```

### Strategy 3: Managed Drift (ignore_changes)

```hcl
# เมื่อ drift เป็น "expected" เช่น Auto Scaling
resource "aws_autoscaling_group" "app" {
  desired_capacity = var.initial_desired_capacity
  min_size         = var.min_size
  max_size         = var.max_size

  lifecycle {
    # ไม่ revert การ scale ที่ทำโดย Auto Scaling
    ignore_changes = [
      desired_capacity,
      # ไม่ track tags ที่ AWS เพิ่มเอง
      tag,
    ]
  }
}

# RDS - ignore auto-managed attributes
resource "aws_db_instance" "main" {
  engine_version = "8.0"

  lifecycle {
    ignore_changes = [
      engine_version,           # allow minor version updates
      ca_cert_identifier,       # allow certificate rotation
      latest_restorable_time,   # AWS managed
    ]
  }
}
```

---

## ขั้นตอนที่ 795: Automated Drift Detection Pipeline

### GitHub Actions Scheduled Workflow

```yaml
# .github/workflows/drift-detection.yml
name: Terraform Drift Detection

on:
  # รันทุกวัน เวลา 06:00 UTC
  schedule:
    - cron: '0 6 * * *'
  
  # รันได้ manual
  workflow_dispatch:
    inputs:
      environment:
        description: 'Environment to check'
        required: false
        default: 'all'
        type: choice
        options:
          - all
          - dev
          - staging
          - prod

jobs:
  detect-drift:
    name: Check ${{ matrix.environment }} for Drift
    runs-on: ubuntu-latest
    
    strategy:
      fail-fast: false
      matrix:
        environment: [dev, staging, prod]
    
    permissions:
      id-token: write
      contents: read
      issues: write     # สำหรับสร้าง GitHub Issue เมื่อพบ drift

    steps:
      - uses: actions/checkout@v4

      - name: Configure AWS Credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::${{ vars.AWS_ACCOUNT_ID }}:role/github-terraform-drift
          aws-region: us-east-1

      - name: Setup Terraform
        uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: "1.7.0"

      - name: Terraform Init
        working-directory: environments/${{ matrix.environment }}
        run: terraform init -input=false

      - name: Check for Drift
        id: drift-check
        working-directory: environments/${{ matrix.environment }}
        run: |
          set +e  # ไม่ fail ทันทีถ้า exit code ไม่ใช่ 0
          
          PLAN_OUTPUT=$(terraform plan -refresh-only -no-color 2>&1)
          PLAN_EXIT_CODE=$?
          
          echo "plan_output<<EOF" >> $GITHUB_OUTPUT
          echo "$PLAN_OUTPUT" >> $GITHUB_OUTPUT
          echo "EOF" >> $GITHUB_OUTPUT
          
          if echo "$PLAN_OUTPUT" | grep -q "Objects have changed outside of Terraform"; then
            echo "drift_detected=true" >> $GITHUB_OUTPUT
            echo "DRIFT DETECTED in ${{ matrix.environment }}"
          elif [ $PLAN_EXIT_CODE -eq 0 ]; then
            echo "drift_detected=false" >> $GITHUB_OUTPUT
            echo "No drift in ${{ matrix.environment }}"
          else
            echo "drift_detected=error" >> $GITHUB_OUTPUT
            echo "Error checking ${{ matrix.environment }}"
          fi

      - name: Create GitHub Issue on Drift
        if: steps.drift-check.outputs.drift_detected == 'true'
        uses: actions/github-script@v7
        with:
          script: |
            const environment = '${{ matrix.environment }}';
            const planOutput = `${{ steps.drift-check.outputs.plan_output }}`;
            
            // ตรวจสอบว่ามี issue เปิดอยู่แล้วหรือไม่
            const issues = await github.rest.issues.listForRepo({
              owner: context.repo.owner,
              repo: context.repo.repo,
              labels: [`drift-${environment}`],
              state: 'open'
            });
            
            if (issues.data.length === 0) {
              await github.rest.issues.create({
                owner: context.repo.owner,
                repo: context.repo.repo,
                title: `🚨 Infrastructure Drift Detected: ${environment}`,
                body: `
              ## Infrastructure Drift Alert
              
              **Environment:** ${environment}
              **Detected:** ${new Date().toISOString()}
              **Run:** ${{ github.server_url }}/${{ github.repository }}/actions/runs/${{ github.run_id }}
              
              ### Drift Details
              \`\`\`
              ${planOutput.substring(0, 3000)}
              \`\`\`
              
              ### Remediation Steps
              1. Review the drift details above
              2. Decide: Accept drift (\`terraform apply -refresh-only\`) or Revert drift (\`terraform apply\`)
              3. Update Terraform code if needed
              4. Close this issue after remediation
                `,
                labels: [`drift-${environment}`, 'infrastructure', 'urgent']
              });
            }

      - name: Send Slack Notification on Drift
        if: steps.drift-check.outputs.drift_detected == 'true'
        uses: slackapi/slack-github-action@v1.26.0
        with:
          payload: |
            {
              "text": "⚠️ Infrastructure Drift Detected",
              "blocks": [
                {
                  "type": "section",
                  "text": {
                    "type": "mrkdwn",
                    "text": "*⚠️ Infrastructure Drift Detected!*\n*Environment:* ${{ matrix.environment }}\n*Run:* <${{ github.server_url }}/${{ github.repository }}/actions/runs/${{ github.run_id }}|View Details>"
                  }
                }
              ]
            }
        env:
          SLACK_WEBHOOK_URL: ${{ secrets.SLACK_DRIFT_WEBHOOK }}
```

---

## ขั้นตอนที่ 796: CloudWatch + Lambda สำหรับ Drift Detection

### Lambda Function สำหรับ Trigger Drift Check

```python
# lambda/drift_detector/handler.py
import boto3
import json
import os
import subprocess

def handler(event, context):
    """
    Lambda function ที่ trigger Terraform drift check
    เรียกผ่าน EventBridge scheduled rule
    """
    
    environment = os.environ.get('ENVIRONMENT', 'dev')
    s3_bucket   = os.environ.get('STATE_BUCKET')
    state_key   = os.environ.get('STATE_KEY')
    
    # ดึง Terraform state จาก S3
    s3 = boto3.client('s3')
    response = s3.get_object(Bucket=s3_bucket, Key=state_key)
    state_content = json.loads(response['Body'].read())
    
    # ดึง resources จาก state
    resources = state_content.get('resources', [])
    
    # ตรวจสอบแต่ละ resource กับ AWS API
    drift_findings = []
    
    for resource in resources:
        if resource.get('mode') != 'managed':
            continue
            
        resource_type = resource.get('type')
        
        for instance in resource.get('instances', []):
            attrs = instance.get('attributes', {})
            resource_id = attrs.get('id')
            
            if resource_type == 'aws_security_group':
                drift = check_security_group_drift(resource_id, attrs)
                if drift:
                    drift_findings.append({
                        'resource': f"{resource_type}.{resource.get('name')}",
                        'id': resource_id,
                        'drift': drift
                    })
    
    # ส่ง notification ถ้าพบ drift
    if drift_findings:
        send_drift_notification(environment, drift_findings)
        return {
            'statusCode': 200,
            'drift_detected': True,
            'findings': len(drift_findings)
        }
    
    return {
        'statusCode': 200,
        'drift_detected': False,
        'findings': 0
    }


def check_security_group_drift(sg_id, expected_attrs):
    """ตรวจสอบว่า Security Group ตรงกับ state หรือไม่"""
    ec2 = boto3.client('ec2')
    
    try:
        response = ec2.describe_security_groups(GroupIds=[sg_id])
        actual_sg = response['SecurityGroups'][0]
        
        # เปรียบเทียบ ingress rules
        actual_ingress = actual_sg.get('IpPermissions', [])
        expected_ingress = expected_attrs.get('ingress', [])
        
        if len(actual_ingress) != len(expected_ingress):
            return {
                'type': 'ingress_rule_count',
                'expected': len(expected_ingress),
                'actual': len(actual_ingress)
            }
    except Exception as e:
        return {'type': 'error', 'message': str(e)}
    
    return None


def send_drift_notification(environment, findings):
    """ส่ง notification ผ่าน SNS"""
    sns = boto3.client('sns')
    topic_arn = os.environ.get('DRIFT_SNS_TOPIC_ARN')
    
    message = f"""
Infrastructure Drift Detected!
Environment: {environment}
Findings: {len(findings)}

Details:
"""
    
    for finding in findings:
        message += f"- {finding['resource']} ({finding['id']}): {finding['drift']}\n"
    
    sns.publish(
        TopicArn=topic_arn,
        Subject=f"Terraform Drift Alert: {environment}",
        Message=message
    )
```

### EventBridge Rule สำหรับ Scheduled Check

```hcl
# drift-detection-infrastructure.tf

# Lambda function
resource "aws_lambda_function" "drift_detector" {
  filename         = "drift_detector.zip"
  function_name    = "terraform-drift-detector-${var.environment}"
  role             = aws_iam_role.drift_detector.arn
  handler          = "handler.handler"
  runtime          = "python3.12"
  timeout          = 300

  environment {
    variables = {
      ENVIRONMENT      = var.environment
      STATE_BUCKET     = var.state_bucket
      STATE_KEY        = "prod/terraform.tfstate"
      DRIFT_SNS_TOPIC_ARN = aws_sns_topic.drift_alerts.arn
    }
  }
}

# EventBridge rule - รันทุก 6 ชั่วโมง
resource "aws_cloudwatch_event_rule" "drift_check" {
  name                = "terraform-drift-check-${var.environment}"
  description         = "Trigger Terraform drift detection every 6 hours"
  schedule_expression = "rate(6 hours)"
  state               = "ENABLED"
}

resource "aws_cloudwatch_event_target" "drift_check" {
  rule      = aws_cloudwatch_event_rule.drift_check.name
  target_id = "TerraformDriftDetector"
  arn       = aws_lambda_function.drift_detector.arn
}

resource "aws_lambda_permission" "drift_check" {
  statement_id  = "AllowEventBridgeInvoke"
  action        = "lambda:InvokeFunction"
  function_name = aws_lambda_function.drift_detector.function_name
  principal     = "events.amazonaws.com"
  source_arn    = aws_cloudwatch_event_rule.drift_check.arn
}

# SNS Topic สำหรับ notifications
resource "aws_sns_topic" "drift_alerts" {
  name = "terraform-drift-alerts-${var.environment}"
}

resource "aws_sns_topic_subscription" "slack" {
  topic_arn = aws_sns_topic.drift_alerts.arn
  protocol  = "https"
  endpoint  = var.slack_webhook_url  # ต้องใช้ SNS-to-Slack Lambda
}

resource "aws_sns_topic_subscription" "email" {
  topic_arn = aws_sns_topic.drift_alerts.arn
  protocol  = "email"
  endpoint  = "infrastructure@mycompany.com"
}
```

---

## ขั้นตอนที่ 797: Terraform Cloud Continuous Validation

### ตั้งค่า Continuous Validation ใน TFC

```hcl
# main.tf - เพิ่ม check blocks
resource "aws_s3_bucket" "critical_data" {
  bucket = "mycompany-critical-data"
}

resource "aws_s3_bucket_versioning" "critical_data" {
  bucket = aws_s3_bucket.critical_data.id
  versioning_configuration {
    status = "Enabled"
  }
}

# Check blocks สำหรับ Continuous Validation
check "s3_bucket_versioning_enabled" {
  data "aws_s3_bucket" "check" {
    bucket = aws_s3_bucket.critical_data.bucket
  }

  assert {
    condition     = data.aws_s3_bucket.check.id == aws_s3_bucket.critical_data.bucket
    error_message = "Critical S3 bucket has been deleted or renamed outside Terraform!"
  }
}

check "s3_bucket_public_access_blocked" {
  data "aws_s3_bucket_public_access_block" "check" {
    bucket = aws_s3_bucket.critical_data.bucket
  }

  assert {
    condition     = data.aws_s3_bucket_public_access_block.check.block_public_acls == true
    error_message = "Public access block was disabled - potential security issue!"
  }
}

check "ec2_instances_running" {
  data "aws_instances" "app_servers" {
    filter {
      name   = "tag:Role"
      values = ["app-server"]
    }
    filter {
      name   = "instance-state-name"
      values = ["running"]
    }
  }

  assert {
    condition     = length(data.aws_instances.app_servers.ids) >= var.min_instance_count
    error_message = "Number of running instances (${length(data.aws_instances.app_servers.ids)}) is below minimum required (${var.min_instance_count})"
  }
}
```

---

## ขั้นตอนที่ 798: Drift Remediation Workflow

### Complete Drift Remediation Process

```bash
#!/bin/bash
# remediate_drift.sh
# Interactive workflow สำหรับ drift remediation

ENVIRONMENT="$1"
COMPONENT="${2:-all}"

if [ -z "$ENVIRONMENT" ]; then
  echo "Usage: $0 <environment> [component]"
  echo "Example: $0 prod networking"
  exit 1
fi

echo "=== Drift Remediation for $ENVIRONMENT/$COMPONENT ==="

# Step 1: ตรวจสอบ drift
echo ""
echo "Step 1: Detecting drift..."
cd "environments/$ENVIRONMENT/$COMPONENT"
terraform init -input=false > /dev/null 2>&1

PLAN=$(terraform plan -refresh-only -no-color 2>&1)

if ! echo "$PLAN" | grep -q "Objects have changed"; then
  echo "✓ No drift detected. Exiting."
  exit 0
fi

echo "⚠ Drift detected!"
echo ""
echo "=== Drift Summary ==="
echo "$PLAN" | grep -A 5 "has been changed"
echo ""

# Step 2: เลือก remediation strategy
echo "Step 2: Choose remediation strategy:"
echo "  1) Accept drift (update state to match reality)"
echo "  2) Revert drift (apply config to fix infrastructure)"
echo "  3) Review full plan first"
echo "  4) Exit without changes"
echo ""
read -p "Enter choice (1-4): " CHOICE

case $CHOICE in
  1)
    echo "Accepting drift (updating state)..."
    terraform apply -refresh-only -auto-approve
    echo "✓ State updated to match actual infrastructure"
    echo ""
    echo "IMPORTANT: Update Terraform code to reflect these changes!"
    ;;
  
  2)
    echo "Reverting drift (applying config)..."
    echo "This will change infrastructure to match Terraform configuration."
    read -p "Are you sure? (yes/no): " CONFIRM
    if [ "$CONFIRM" = "yes" ]; then
      terraform apply -auto-approve
      echo "✓ Infrastructure reverted to Terraform configuration"
    else
      echo "Cancelled."
    fi
    ;;
  
  3)
    echo "Showing full plan..."
    terraform plan
    echo ""
    read -p "Press Enter to continue..."
    exec "$0" "$ENVIRONMENT" "$COMPONENT"  # Re-run to choose
    ;;
  
  4)
    echo "Exiting without changes."
    exit 0
    ;;
  
  *)
    echo "Invalid choice. Exiting."
    exit 1
    ;;
esac

# Step 3: Verify
echo ""
echo "Step 3: Verifying remediation..."
VERIFY_PLAN=$(terraform plan -refresh-only -no-color 2>&1)

if echo "$VERIFY_PLAN" | grep -q "Objects have changed"; then
  echo "⚠ Drift still detected after remediation!"
  echo "Manual investigation required."
  exit 1
else
  echo "✓ No drift remaining. Remediation successful!"
fi
```

---

## ขั้นตอนที่ 799: Preventing Drift

### AWS Organizations Service Control Policies (SCPs)

```json
// scp-restrict-console-access.json
// ป้องกัน manual changes ผ่าน Console

{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DenyManualEC2Changes",
      "Effect": "Deny",
      "Action": [
        "ec2:RunInstances",
        "ec2:TerminateInstances",
        "ec2:ModifyInstanceAttribute"
      ],
      "Resource": "*",
      "Condition": {
        "StringNotLike": {
          "aws:PrincipalArn": [
            "arn:aws:iam::*:role/terraform-*",
            "arn:aws:iam::*:role/emergency-break-glass"
          ]
        }
      }
    },
    {
      "Sid": "DenyManualSecurityGroupChanges",
      "Effect": "Deny",
      "Action": [
        "ec2:AuthorizeSecurityGroupIngress",
        "ec2:AuthorizeSecurityGroupEgress",
        "ec2:RevokeSecurityGroupIngress",
        "ec2:RevokeSecurityGroupEgress"
      ],
      "Resource": "*",
      "Condition": {
        "StringNotLike": {
          "aws:PrincipalArn": "arn:aws:iam::*:role/terraform-*"
        }
      }
    }
  ]
}
```

### Immutable Infrastructure Pattern

```hcl
# แทนที่จะ update instance (mutable)
# ให้ create instance ใหม่แล้ว destroy เก่า (immutable)

resource "aws_instance" "app" {
  ami           = var.ami_id  # เปลี่ยน AMI = สร้าง instance ใหม่เสมอ
  instance_type = var.instance_type

  lifecycle {
    # สร้างใหม่ก่อน แล้วค่อย destroy เก่า
    create_before_destroy = true

    # ถ้า AMI เปลี่ยน = replace ทั้ง instance
    replace_triggered_by = [
      var.ami_id
    ]
  }
}

# Launch Template + ASG สำหรับ immutable approach
resource "aws_launch_template" "app" {
  name_prefix   = "app-"
  image_id      = var.ami_id
  instance_type = var.instance_type

  # Hash ของ config เป็นส่วนหนึ่งของ name
  # เมื่อ config เปลี่ยน จะสร้าง template ใหม่
  name_prefix = "app-${sha256(jsonencode({
    ami           = var.ami_id
    instance_type = var.instance_type
    user_data     = var.user_data
  }))}-"

  lifecycle {
    create_before_destroy = true
  }
}
```

---

## ขั้นตอนที่ 800: Complete Drift Detection System

### Full GitHub Actions Workflow

```yaml
# .github/workflows/drift-detection-complete.yml
name: Complete Drift Detection & Remediation

on:
  # ตรวจสอบทุกวันเวลา 02:00 UTC
  schedule:
    - cron: '0 2 * * *'
  
  workflow_dispatch:
    inputs:
      remediate:
        description: 'Auto-remediate drift?'
        required: false
        default: 'false'
        type: boolean

env:
  TF_VERSION: "1.7.0"
  SLACK_CHANNEL: "#infrastructure-alerts"

jobs:
  detect-and-report:
    name: Detect Drift
    runs-on: ubuntu-latest
    outputs:
      drift_summary: ${{ steps.report.outputs.summary }}
      has_drift: ${{ steps.report.outputs.has_drift }}

    strategy:
      fail-fast: false
      matrix:
        environment: [dev, staging, prod]
        component: [networking, compute, database]
        exclude:
          - environment: dev
            component: database  # dev ไม่มี database component

    permissions:
      id-token: write
      contents: read

    steps:
      - uses: actions/checkout@v4

      - name: Configure AWS
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::${{ vars.AWS_ACCOUNT_ID }}:role/drift-detection-role
          aws-region: us-east-1

      - name: Setup Terraform
        uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: ${{ env.TF_VERSION }}

      - name: Check for state directory
        id: check_dir
        run: |
          if [ -d "environments/${{ matrix.environment }}/${{ matrix.component }}" ]; then
            echo "exists=true" >> $GITHUB_OUTPUT
          else
            echo "exists=false" >> $GITHUB_OUTPUT
          fi

      - name: Terraform Init
        if: steps.check_dir.outputs.exists == 'true'
        working-directory: environments/${{ matrix.environment }}/${{ matrix.component }}
        run: terraform init -input=false -no-color

      - name: Detect Drift
        id: detect
        if: steps.check_dir.outputs.exists == 'true'
        working-directory: environments/${{ matrix.environment }}/${{ matrix.component }}
        run: |
          set +e
          OUTPUT=$(terraform plan -refresh-only -no-color -detailed-exitcode 2>&1)
          EXIT_CODE=$?
          
          echo "exit_code=$EXIT_CODE" >> $GITHUB_OUTPUT
          echo "output<<HEREDOC" >> $GITHUB_OUTPUT
          echo "$OUTPUT" >> $GITHUB_OUTPUT
          echo "HEREDOC" >> $GITHUB_OUTPUT
          
          # exit code 2 = changes detected (drift)
          if [ $EXIT_CODE -eq 2 ]; then
            echo "drift=true" >> $GITHUB_OUTPUT
          elif [ $EXIT_CODE -eq 0 ]; then
            echo "drift=false" >> $GITHUB_OUTPUT
          else
            echo "drift=error" >> $GITHUB_OUTPUT
          fi

      - name: Report Results
        id: report
        if: always()
        run: |
          DRIFT="${{ steps.detect.outputs.drift }}"
          ENV="${{ matrix.environment }}"
          COMP="${{ matrix.component }}"
          
          if [ "$DRIFT" = "true" ]; then
            echo "has_drift=true" >> $GITHUB_OUTPUT
            echo "summary=$ENV/$COMP: DRIFT DETECTED" >> $GITHUB_OUTPUT
          elif [ "$DRIFT" = "false" ]; then
            echo "has_drift=false" >> $GITHUB_OUTPUT
            echo "summary=$ENV/$COMP: Clean" >> $GITHUB_OUTPUT
          else
            echo "has_drift=unknown" >> $GITHUB_OUTPUT
            echo "summary=$ENV/$COMP: Error" >> $GITHUB_OUTPUT
          fi

  send-report:
    name: Send Drift Report
    runs-on: ubuntu-latest
    needs: detect-and-report
    if: always()

    steps:
      - name: Compile Report
        run: |
          echo "# Drift Detection Report" > report.md
          echo "Date: $(date -u)" >> report.md
          echo "" >> report.md
          echo "## Results" >> report.md
          echo "${{ needs.detect-and-report.outputs.drift_summary }}" >> report.md

      - name: Send Slack Summary
        uses: slackapi/slack-github-action@v1.26.0
        with:
          payload: |
            {
              "text": "Terraform Drift Detection Complete",
              "blocks": [
                {
                  "type": "header",
                  "text": {
                    "type": "plain_text",
                    "text": "🔍 Daily Drift Detection Report"
                  }
                },
                {
                  "type": "section",
                  "text": {
                    "type": "mrkdwn",
                    "text": "*Status:* ${{ needs.detect-and-report.outputs.has_drift == 'true' && '⚠️ Drift Detected' || '✅ No Drift' }}\n*Run:* <${{ github.server_url }}/${{ github.repository }}/actions/runs/${{ github.run_id }}|View Details>"
                  }
                }
              ]
            }
        env:
          SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK_URL }}
```

---

## สรุป (Summary)

### Drift Management Decision Tree

```
ตรวจพบ Drift
    │
    ├─→ Drift ถูกต้อง (emergency fix, intentional change)?
    │       │
    │       ├─→ YES: terraform apply -refresh-only
    │       │         + Update Terraform code ให้ตรง
    │       │
    │       └─→ NO: terraform apply
    │                 (revert กลับสู่ desired state)
    │
    ├─→ Drift เป็น expected (Auto Scaling, AWS managed)?
    │       │
    │       └─→ YES: เพิ่ม ignore_changes lifecycle
    │
    └─→ Drift เกิดบ่อยจาก manual changes?
              │
              └─→ ใช้ SCPs/Azure Policy เพื่อ prevent
```

| กลยุทธ์ | Command | ใช้เมื่อ |
|---------|---------|----------|
| Detect | `terraform plan -refresh-only` | ตรวจสอบ drift |
| Accept | `terraform apply -refresh-only` | drift ถูกต้อง |
| Revert | `terraform apply` | drift ไม่ถูกต้อง |
| Ignore | `lifecycle { ignore_changes }` | drift expected |
| Prevent | SCPs, Org Policies | ป้องกัน manual changes |
| Monitor | Scheduled checks | detect drift อัตโนมัติ |

---

*จบ Part 080 - สิ้นสุด Series Terraform Advanced Topics (Part 071-080)*

## Course Summary: What We Covered

| Part | Topic |
|------|-------|
| 071 | Moved Blocks & Refactoring |
| 072 | Terraform Testing Framework |
| 073 | Terratest Integration Testing |
| 074 | Terraform Cloud & Remote Operations |
| 075 | Terraform Enterprise Features |
| 076 | Sentinel Policy as Code |
| 077 | OPA with Terraform |
| 078 | Terraform Performance & Scale |
| 079 | Terraform State Advanced Operations |
| 080 | Terraform Drift Detection |
