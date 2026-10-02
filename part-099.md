# Part 99: Cost Optimization with Terraform (Steps 981-990)

## การจัดการต้นทุน Cloud Infrastructure ด้วย Terraform

---

## Step 981: ทำไม Cost Optimization จำคัญ?

### ปัญหาต้นทุน Cloud ที่พบบ่อย

```
Cloud Cost Waste Statistics:
────────────────────────────────────────────────────────────
💸 30% ของ cloud spending เป็น waste (Gartner 2023)
💸 Dev environments รันตลอด 24/7 ทั้งที่ไม่ต้องการ
💸 Oversized instances เพราะ "ขอให้ใหญ่ไว้ก่อน"
💸 Unattached EBS volumes จาก deleted EC2 instances
💸 Data transfer costs ที่ไม่ได้คาดไว้
💸 Forgotten test resources
────────────────────────────────────────────────────────────
```

### ToolsหLัก สำหรับ Cost Optimization

```
1. Infracost    - ประเมิน cost ของ Terraform changes
2. AWS Cost Explorer - วิเคราะห์ cost จริง
3. AWS Budgets  - ตั้ง budget alerts
4. Cost Allocation Tags - แยก cost ตาม team/project
5. AWS Compute Optimizer - แนะนำ rightsizing
```

---

## Step 982: Infracost - ประเมินต้นทุนก่อน Apply

### ติดตั้ง Infracost

```bash
# macOS
brew install infracost

# Linux
curl -fsSL https://raw.githubusercontent.com/infracost/infracost/master/scripts/install.sh | sh

# Windows (Chocolatey)
choco install infracost

# Docker
docker pull infracost/infracost:ci-0.10

# ตรวจสอบ version
infracost --version

# Register (ฟรี - ต้องมี API key)
infracost auth login
# หรือ
infracost configure set api_key <your-api-key>
```

### คำสั่งพื้นฐาน

```bash
# ===== Breakdown - ดู full cost breakdown =====
infracost breakdown --path .
# Output:
#  Name                                   Monthly Qty  Unit           Monthly Cost
# ─────────────────────────────────────────────────────────────────────────────
#  aws_instance.web
#  ├─ Instance usage (Linux/UNIX, on-demand, t3.xlarge)    730 hours    $150.56
#  └─ root_block_device
#     └─ Storage (general purpose SSD, gp3)               50 GB         $4.00
#  
#  aws_db_instance.main
#  ├─ Database instance (on-demand, db.r5.2xlarge)        730 hours    $876.00
#  ├─ Storage (gp2)                                       100 GB        $11.50
#  └─ Backup storage                                      100 GB         $9.50
# ─────────────────────────────────────────────────────────────────────────────
#  MONTHLY TOTAL                                                      $1,051.56

# ===== Diff - เปรียบเทียบ cost ก่อน/หลัง change =====
# ต้องมี git history
infracost diff --path .

# ===== Output formats =====
infracost breakdown --path . --format table     # Default
infracost breakdown --path . --format json > infracost.json
infracost breakdown --path . --format html > infracost.html

# ===== ดู cost ต่อ resource =====
infracost breakdown --path . --show-skipped

# ===== ระบุ terraform variables =====
infracost breakdown \
  --path . \
  --terraform-var-file=terraform.tfvars \
  --terraform-var="environment=prod"

# ===== Scan หลาย projects =====
infracost breakdown \
  --path environments/dev \
  --path environments/staging \
  --path environments/prod
```

### Infracost JSON Output

```bash
infracost breakdown --path . --format json > infracost.json
cat infracost.json | jq '{
  total_monthly_cost: .totalMonthlyCost,
  resources: [.projects[].breakdown.resources[] | {
    name: .name,
    monthly_cost: .monthlyCost
  }] | sort_by(.monthly_cost | tonumber) | reverse
}'
```

### Infracost diff สำหรับ CI/CD

```bash
# สร้าง JSON สำหรับ baseline (current state)
git stash
infracost breakdown --path . --format json > /tmp/infracost-base.json
git stash pop

# สร้าง JSON สำหรับ new state (changes)
infracost breakdown --path . --format json > /tmp/infracost-new.json

# Compare
infracost diff \
  --path . \
  --compare-to /tmp/infracost-base.json \
  --format json \
  > /tmp/infracost-diff.json

# แสดงผล
infracost diff \
  --path . \
  --compare-to /tmp/infracost-base.json
```

---

## Step 983: Infracost ใน GitHub Actions

### Complete Infracost Workflow

```yaml
# .github/workflows/infracost.yml
name: Infracost Cost Estimation

on:
  pull_request:
    paths:
      - '**.tf'
      - '**.tfvars'

permissions:
  contents: read
  pull-requests: write

jobs:
  infracost:
    name: Cost Estimation
    runs-on: ubuntu-latest
    
    env:
      TF_ROOT: ./environments
      INFRACOST_API_KEY: ${{ secrets.INFRACOST_API_KEY }}
    
    steps:
      - name: Setup Infracost
        uses: infracost/actions/setup@v3
        with:
          api-key: ${{ secrets.INFRACOST_API_KEY }}
      
      - name: Checkout base branch (for comparison)
        uses: actions/checkout@v4
        with:
          ref: ${{ github.event.pull_request.base.ref }}
      
      - name: Generate base Infracost data
        run: |
          infracost breakdown \
            --path ${{ env.TF_ROOT }} \
            --format json \
            --out-file /tmp/infracost-base.json
      
      - name: Checkout PR branch
        uses: actions/checkout@v4
      
      - name: Generate PR Infracost data
        run: |
          infracost breakdown \
            --path ${{ env.TF_ROOT }} \
            --format json \
            --out-file /tmp/infracost-pr.json
      
      - name: Post Infracost comment
        run: |
          infracost comment github \
            --path=/tmp/infracost-pr.json \
            --repo=$GITHUB_REPOSITORY \
            --github-token=${{ github.token }} \
            --pull-request=${{ github.event.pull_request.number }} \
            --behavior=update \
            --compare-to=/tmp/infracost-base.json
      
      - name: Check Cost Threshold
        run: |
          # ดึง cost เปรียบเทียบ
          DIFF=$(infracost diff \
            --path /tmp/infracost-pr.json \
            --compare-to /tmp/infracost-base.json \
            --format json | jq '.diffTotalMonthlyCost // "0"' -r)
          
          echo "Monthly cost difference: $${DIFF}"
          
          # ถ้าเพิ่มขึ้นเกิน $500/เดือน → warning
          THRESHOLD=500
          DIFF_NUM=$(echo "$DIFF" | sed 's/[^0-9.-]//g')
          
          if (( $(echo "$DIFF_NUM > $THRESHOLD" | bc -l) )); then
            echo "::warning::Monthly cost will increase by \$${DIFF_NUM} which exceeds threshold of \$${THRESHOLD}"
          fi
```

### Infracost Comment ตัวอย่าง

```
## 💰 Infracost Cost Estimation

Monthly cost will increase by $+234.56 (+18%)

| Resource | Base | New | Diff |
|----------|------|-----|------|
| aws_instance.web | $150.56/mo | $301.12/mo | +$150.56 |
| aws_db_instance.main | $876.00/mo | $960.00/mo | +$84.00 |
| New: aws_elasticache_cluster.redis | - | $100.00/mo | +$100.00 |

**Old monthly cost**: $1,026.56
**New monthly cost**: $1,361.12
**Difference**: +$334.56 (+32.6%)

> Costs shown are estimates. Actual costs depend on usage.
```

---

## Step 984: Cost-aware Terraform Patterns

### Pattern 1: Right-sizing Instances

```hcl
# variables.tf
variable "instance_type_map" {
  description = "Approved instance types per environment"
  type = map(string)
  
  default = {
    dev     = "t3.micro"
    staging = "t3.medium"
    prod    = "m5.large"
  }
}

# main.tf
resource "aws_instance" "app" {
  instance_type = var.instance_type_map[var.environment]
  ami           = data.aws_ami.app.id
  
  tags = {
    Name        = "${var.environment}-app"
    Environment = var.environment
  }
}

# terraform.tfvars สำหรับ prod
environment = "prod"
# instance_type จะ = m5.large (ราคาสมเหตุสมผล)
```

### Pattern 2: Spot Instances with On-Demand Fallback

```hcl
# ===== Spot Instance + On-Demand Mixed Fleet =====
resource "aws_autoscaling_group" "app" {
  name               = "${var.environment}-app-asg"
  min_size           = var.min_size
  max_size           = var.max_size
  desired_capacity   = var.desired_capacity
  vpc_zone_identifier = var.private_subnet_ids
  
  mixed_instances_policy {
    instances_distribution {
      # 70% Spot, 30% On-Demand = ประหยัด ~50-70%!
      on_demand_base_capacity                  = var.on_demand_base_count
      on_demand_percentage_above_base_capacity = 30
      spot_allocation_strategy                 = "capacity-optimized"
      
      # หลาย instance types = เพิ่มโอกาสได้ Spot
      spot_max_price = ""  # ใช้ on-demand price เป็น max
    }
    
    launch_template {
      launch_template_specification {
        launch_template_id = aws_launch_template.app.id
        version            = "$Latest"
      }
      
      override {
        instance_type = "m5.large"
        weighted_capacity = "1"
      }
      override {
        instance_type = "m5a.large"
        weighted_capacity = "1"
      }
      override {
        instance_type = "m4.large"
        weighted_capacity = "1"
      }
      override {
        instance_type = "t3.large"
        weighted_capacity = "1"
      }
    }
  }
  
  # ใช้ instance refresh สำหรับ zero-downtime
  instance_refresh {
    strategy = "Rolling"
    preferences {
      min_healthy_percentage = 90
    }
  }
  
  tag {
    key                 = "SpotEnabled"
    value               = "true"
    propagate_at_launch = true
  }
}
```

### Pattern 3: S3 Intelligent Tiering

```hcl
# ประหยัด cost สำหรับ S3 ที่ access pattern ไม่แน่นอน
resource "aws_s3_bucket" "data" {
  bucket = "${var.environment}-data-${random_id.bucket_suffix.hex}"
}

resource "aws_s3_bucket_intelligent_tiering_configuration" "data" {
  bucket = aws_s3_bucket.data.id
  name   = "EntireS3Bucket"
  status = "Enabled"
  
  tiering {
    access_tier = "DEEP_ARCHIVE_ACCESS"
    days        = 180  # ข้อมูลที่ไม่ได้ access 180 วัน → Deep Archive (ถูกสุด)
  }
  
  tiering {
    access_tier = "ARCHIVE_ACCESS"
    days        = 90   # ข้อมูลที่ไม่ได้ access 90 วัน → Archive
  }
}

resource "aws_s3_bucket_lifecycle_configuration" "data" {
  bucket = aws_s3_bucket.data.id
  
  rule {
    id     = "transition-old-data"
    status = "Enabled"
    
    transition {
      days          = 30
      storage_class = "STANDARD_IA"  # หลัง 30 วัน → Infrequent Access (40% ถูกกว่า)
    }
    
    transition {
      days          = 90
      storage_class = "GLACIER"  # หลัง 90 วัน → Glacier (80% ถูกกว่า)
    }
    
    expiration {
      days = 2555  # ลบหลัง 7 ปี (compliance)
    }
    
    noncurrent_version_expiration {
      noncurrent_days = 90  # ลบ old versions หลัง 90 วัน
    }
  }
}
```

### Pattern 4: RDS Stop/Start Scheduling (Dev/Staging)

```hcl
# ===== EventBridge Scheduler สำหรับ RDS Stop/Start =====
# หยุด RDS ตอนเย็น เริ่มตอนเช้า = ประหยัด ~60%

resource "aws_scheduler_schedule" "rds_stop" {
  count = var.environment != "prod" ? 1 : 0
  
  name       = "${var.environment}-rds-stop"
  group_name = "default"
  
  flexible_time_window {
    mode = "OFF"
  }
  
  # หยุดทุกวันจันทร์-ศุกร์ เวลา 20:00 ICT (13:00 UTC)
  schedule_expression          = "cron(0 13 ? * MON-FRI *)"
  schedule_expression_timezone = "Asia/Bangkok"
  
  target {
    arn      = "arn:aws:scheduler:::aws-sdk:rds:stopDBInstance"
    role_arn = aws_iam_role.scheduler.arn
    
    input = jsonencode({
      DbInstanceIdentifier = aws_db_instance.main.id
    })
  }
}

resource "aws_scheduler_schedule" "rds_start" {
  count = var.environment != "prod" ? 1 : 0
  
  name       = "${var.environment}-rds-start"
  group_name = "default"
  
  flexible_time_window {
    mode = "OFF"
  }
  
  # เริ่มทุกวันจันทร์-ศุกร์ เวลา 08:00 ICT (01:00 UTC)
  schedule_expression          = "cron(0 1 ? * MON-FRI *)"
  schedule_expression_timezone = "Asia/Bangkok"
  
  target {
    arn      = "arn:aws:scheduler:::aws-sdk:rds:startDBInstance"
    role_arn = aws_iam_role.scheduler.arn
    
    input = jsonencode({
      DbInstanceIdentifier = aws_db_instance.main.id
    })
  }
}

# ===== EC2 Stop/Start Scheduling =====
resource "aws_scheduler_schedule" "ec2_stop" {
  count = var.environment == "dev" ? 1 : 0
  
  name = "${var.environment}-ec2-stop"
  
  flexible_time_window {
    mode = "OFF"
  }
  
  schedule_expression          = "cron(0 13 ? * MON-FRI *)"
  schedule_expression_timezone = "Asia/Bangkok"
  
  target {
    arn      = "arn:aws:scheduler:::aws-sdk:ec2:stopInstances"
    role_arn = aws_iam_role.scheduler.arn
    
    input = jsonencode({
      InstanceIds = [aws_instance.dev_server.id]
    })
  }
}
```

---

## Step 985: AWS Cost Allocation Tags

### Terraform-enforced Tagging

```hcl
# ===== ตั้งค่า Cost Allocation Tags ใน AWS =====
resource "aws_ce_cost_allocation_tag" "tags" {
  for_each = toset([
    "Environment",
    "Team",
    "Project",
    "CostCenter",
    "ManagedBy",
  ])
  
  tag_key = each.value
  status  = "Active"
}

# ===== Default Tags ด้วย Provider =====
provider "aws" {
  region = var.aws_region
  
  default_tags {
    tags = {
      Environment = var.environment
      Team        = var.team
      Project     = var.project
      CostCenter  = var.cost_center
      ManagedBy   = "terraform"
      Repository  = var.repository
    }
  }
}

# ===== Custom Tag Policy =====
resource "aws_organizations_policy" "tagging" {
  name        = "RequiredTaggingPolicy"
  description = "Enforce required tags on all resources"
  type        = "TAG_POLICY"
  
  content = jsonencode({
    tags = {
      Environment = {
        tag_key = {
          "@@assign" = "Environment"
        }
        tag_value = {
          "@@assign" = ["dev", "staging", "prod", "sandbox"]
        }
        enforced_for = {
          "@@assign" = [
            "ec2:instance",
            "rds:db",
            "s3:bucket"
          ]
        }
      }
      CostCenter = {
        tag_key = {
          "@@assign" = "CostCenter"
        }
        enforced_for = {
          "@@assign" = ["ec2:instance", "rds:db"]
        }
      }
    }
  })
}
```

---

## Step 986: Unused Resource Detection

### Lambda สำหรับหา Resources ที่ไม่ได้ใช้

```hcl
# ===== Lambda สำหรับตรวจหา unattached EBS volumes =====
resource "aws_lambda_function" "unused_resources_detector" {
  function_name = "${var.environment}-unused-resources-detector"
  role          = aws_iam_role.lambda_cost_optimizer.arn
  
  filename         = data.archive_file.detector.output_path
  source_code_hash = data.archive_file.detector.output_base64sha256
  
  runtime = "python3.11"
  handler = "detector.lambda_handler"
  timeout = 300
  
  environment {
    variables = {
      SNS_TOPIC_ARN = aws_sns_topic.cost_alerts.arn
      ENVIRONMENT   = var.environment
    }
  }
}

data "archive_file" "detector" {
  type        = "zip"
  output_path = "/tmp/detector.zip"
  
  source {
    content  = <<-PYTHON
import boto3
import os
import json

def lambda_handler(event, context):
    ec2 = boto3.client('ec2')
    findings = []
    
    # ===== Check 1: Unattached EBS volumes =====
    volumes = ec2.describe_volumes(
        Filters=[{'Name': 'status', 'Values': ['available']}]
    )['Volumes']
    
    for vol in volumes:
        findings.append({
            'type': 'unattached_ebs',
            'id': vol['VolumeId'],
            'size_gb': vol['Size'],
            'monthly_cost': vol['Size'] * 0.10,  # ~$0.10/GB for gp2
            'tags': {t['Key']: t['Value'] for t in vol.get('Tags', [])}
        })
    
    # ===== Check 2: Unused Elastic IPs =====
    eips = ec2.describe_addresses()['Addresses']
    for eip in eips:
        if 'InstanceId' not in eip and 'NetworkInterfaceId' not in eip:
            findings.append({
                'type': 'unused_eip',
                'id': eip.get('AllocationId', eip.get('PublicIp')),
                'ip': eip.get('PublicIp'),
                'monthly_cost': 3.65,  # ~$0.005/hour = $3.65/month
                'tags': {t['Key']: t['Value'] for t in eip.get('Tags', [])}
            })
    
    # ===== Check 3: Old Snapshots (>30 days) =====
    import datetime
    snapshots = ec2.describe_snapshots(OwnerIds=['self'])['Snapshots']
    cutoff = datetime.datetime.now(datetime.timezone.utc) - datetime.timedelta(days=30)
    
    for snap in snapshots:
        if snap['StartTime'] < cutoff:
            findings.append({
                'type': 'old_snapshot',
                'id': snap['SnapshotId'],
                'size_gb': snap['VolumeSize'],
                'age_days': (datetime.datetime.now(datetime.timezone.utc) - snap['StartTime']).days,
                'monthly_cost': snap['VolumeSize'] * 0.05,  # ~$0.05/GB for snapshots
            })
    
    if findings:
        total_waste = sum(f.get('monthly_cost', 0) for f in findings)
        
        sns = boto3.client('sns')
        sns.publish(
            TopicArn=os.environ['SNS_TOPIC_ARN'],
            Subject=f"💸 Unused AWS Resources Detected - ${total_waste:.2f}/month waste",
            Message=json.dumps({
                'environment': os.environ['ENVIRONMENT'],
                'total_monthly_waste': total_waste,
                'findings': findings
            }, default=str, indent=2)
        )
    
    return {'findings': len(findings), 'total_waste': sum(f.get('monthly_cost', 0) for f in findings)}
    PYTHON
    filename = "detector.py"
  }
}

# ===== EventBridge ให้ Lambda รันทุกวัน =====
resource "aws_cloudwatch_event_rule" "daily_detector" {
  name                = "${var.environment}-daily-unused-resource-check"
  description         = "Daily check for unused AWS resources"
  schedule_expression = "cron(0 2 * * ? *)"  # 2am UTC ทุกวัน
}

resource "aws_cloudwatch_event_target" "detector_lambda" {
  rule      = aws_cloudwatch_event_rule.daily_detector.name
  target_id = "UnusedResourceDetector"
  arn       = aws_lambda_function.unused_resources_detector.arn
}
```

---

## Step 987: AWS Budget Alerts

```hcl
# ===== Budget Alerts =====

# 1. Overall account budget
resource "aws_budgets_budget" "total_account" {
  name         = "${var.environment}-total-budget"
  budget_type  = "COST"
  limit_amount = var.monthly_budget_usd
  limit_unit   = "USD"
  time_unit    = "MONTHLY"
  
  # Alert ที่ 80%
  notification {
    comparison_operator        = "GREATER_THAN"
    threshold                  = 80
    threshold_type             = "PERCENTAGE"
    notification_type          = "ACTUAL"
    subscriber_email_addresses = ["finance@company.com", "platform-team@company.com"]
    subscriber_sns_topic_arns  = [aws_sns_topic.budget_alerts.arn]
  }
  
  # Alert ที่ 100%
  notification {
    comparison_operator        = "GREATER_THAN"
    threshold                  = 100
    threshold_type             = "PERCENTAGE"
    notification_type          = "ACTUAL"
    subscriber_email_addresses = ["cto@company.com", "finance@company.com"]
    subscriber_sns_topic_arns  = [aws_sns_topic.budget_alerts.arn]
  }
  
  # Forecasted > 120%
  notification {
    comparison_operator        = "GREATER_THAN"
    threshold                  = 120
    threshold_type             = "PERCENTAGE"
    notification_type          = "FORECASTED"
    subscriber_email_addresses = ["platform-team@company.com"]
  }
}

# 2. Budget per service
resource "aws_budgets_budget" "ec2_budget" {
  name         = "${var.environment}-ec2-budget"
  budget_type  = "COST"
  limit_amount = var.ec2_monthly_budget_usd
  limit_unit   = "USD"
  time_unit    = "MONTHLY"
  
  cost_filter {
    name   = "Service"
    values = ["Amazon Elastic Compute Cloud - Compute"]
  }
  
  notification {
    comparison_operator        = "GREATER_THAN"
    threshold                  = 90
    threshold_type             = "PERCENTAGE"
    notification_type          = "ACTUAL"
    subscriber_email_addresses = var.budget_alert_emails
  }
}

# 3. Budget per tag (per team/project)
resource "aws_budgets_budget" "per_team" {
  for_each = var.team_budgets
  
  name         = "${var.environment}-team-${each.key}-budget"
  budget_type  = "COST"
  limit_amount = each.value.monthly_usd
  limit_unit   = "USD"
  time_unit    = "MONTHLY"
  
  cost_filter {
    name   = "TagKeyValue"
    values = ["user:Team$${each.key}"]
  }
  
  notification {
    comparison_operator        = "GREATER_THAN"
    threshold                  = 80
    threshold_type             = "PERCENTAGE"
    notification_type          = "ACTUAL"
    subscriber_email_addresses = each.value.alert_emails
  }
}
```

---

## Step 988: Infracost ใน Atlantis

### atlantis.yaml พร้อม Infracost

```yaml
# atlantis.yaml
version: 3

projects:
  - name: prod
    dir: environments/prod
    workflow: cost-aware
    autoplan:
      when_modified:
        - "*.tf"
        - "*.tfvars"

workflows:
  cost-aware:
    plan:
      steps:
        - env:
            name: INFRACOST_API_KEY
            command: 'echo $INFRACOST_API_KEY'  # จาก env var
        
        - run: |
            # สร้าง cost estimate ก่อน plan
            if command -v infracost &>/dev/null; then
              infracost breakdown \
                --path . \
                --format json \
                > /tmp/infracost-current.json
              
              echo "=== Current Monthly Cost ==="
              infracost output \
                --path /tmp/infracost-current.json \
                --format table
            fi
        
        - init
        
        - plan:
            extra_args: ["-out=tfplan.binary"]
        
        - run: |
            # สร้าง cost estimate หลัง plan
            if command -v infracost &>/dev/null; then
              terraform show -json tfplan.binary > /tmp/tfplan.json
              
              infracost diff \
                --path /tmp/tfplan.json \
                --compare-to /tmp/infracost-current.json
              
              echo ""
              echo "=== Cost Summary ==="
              DIFF=$(infracost diff \
                --path /tmp/tfplan.json \
                --compare-to /tmp/infracost-current.json \
                --format json | jq '.diffTotalMonthlyCost' -r)
              echo "Monthly cost change: $${DIFF}"
            fi
    
    apply:
      steps:
        - apply
```

---

## Step 989: Reserved Instances & Savings Plans ด้วย Terraform Tags

```hcl
# ===== Tagging สำหรับ RI/Savings Plans tracking =====

locals {
  # Tags สำหรับ cost optimization
  cost_tags = {
    # สำหรับ RI matching
    "aws:cloudformation:stack-name" = "${var.environment}-infra"
    
    # Custom tags
    BillingTeam     = var.billing_team
    CostCenter      = var.cost_center
    Project         = var.project
    Environment     = var.environment
    
    # RI eligibility marker
    ReservedInstance = "eligible"  # ช่วยทีม finance identify RI candidates
  }
}

# ===== Compute Savings Plans (ยืดหยุ่นกว่า RI) =====
# ไม่สามารถ manage ผ่าน Terraform ได้โดยตรง
# แต่ tagging ช่วยให้ track ง่ายขึ้น

# ===== AWS Compute Optimizer Integration =====
resource "aws_computeoptimizer_enrollment_status" "main" {
  status                = "Active"
  include_member_accounts = true  # รวม accounts ใน organization
}

# ===== EC2 Instance ที่พร้อมสำหรับ Savings Plans =====
resource "aws_instance" "app" {
  ami           = var.ami_id
  instance_type = "m5.large"  # Savings Plans apply ได้
  
  # ต้องเป็น tenancy = default สำหรับ Compute Savings Plans
  tenancy = "default"
  
  tags = merge(local.cost_tags, {
    Name          = "${var.environment}-app-001"
    SavingsPlan   = "compute"  # ช่วย finance track
  })
  
  lifecycle {
    # ป้องกัน Terraform เปลี่ยน instance type โดยไม่ตั้งใจ
    # (กระทบ Savings Plans commitment)
    ignore_changes = [instance_type]
  }
}
```

---

## Step 990: Cost Dashboard และ Reporting

### AWS Cost Explorer Query

```hcl
# ===== AWS Cost Anomaly Detection =====
resource "aws_ce_anomaly_monitor" "main" {
  name              = "${var.environment}-cost-anomaly-monitor"
  monitor_type      = "DIMENSIONAL"
  
  monitor_dimension = "SERVICE"
}

resource "aws_ce_anomaly_subscription" "main" {
  name      = "${var.environment}-cost-anomaly-alerts"
  frequency = "DAILY"
  
  monitor_arn_list = [
    aws_ce_anomaly_monitor.main.arn
  ]
  
  subscriber {
    address = "platform-team@company.com"
    type    = "EMAIL"
  }
  
  # Alert ถ้า cost spike เกิน $50
  threshold_expression {
    dimension {
      key           = "ANOMALY_TOTAL_IMPACT_ABSOLUTE"
      match_options = ["GREATER_THAN_OR_EQUAL"]
      values        = ["50"]
    }
  }
}

# ===== Cost and Usage Report (CUR) =====
resource "aws_cur_report_definition" "main" {
  report_name                = "${var.environment}-cost-report"
  time_unit                  = "DAILY"
  format                     = "Parquet"
  compression                = "Parquet"
  additional_schema_elements = ["RESOURCES"]
  s3_bucket                  = aws_s3_bucket.cur_reports.id
  s3_prefix                  = "cost-reports"
  s3_region                  = var.aws_region
  report_versioning          = "OVERWRITE_REPORT"
  
  additional_artifacts = [
    "ATHENA",  # ใช้ Athena query reports
    "QUICKSIGHT",  # ใช้ QuickSight dashboard
  ]
}
```

### Cost Optimization Summary

```hcl
# ===== outputs.tf - Cost-related outputs =====
output "estimated_monthly_savings" {
  description = "Estimated monthly savings from cost optimizations"
  value = {
    spot_instances = "~50-70% vs On-Demand"
    s3_intelligent_tiering = "~40% on infrequently accessed data"
    rds_scheduled_stop = "~60% for non-prod (stops nights/weekends)"
    ec2_scheduled_stop = "~65% for dev environment"
    reserved_instances = "~30-60% vs On-Demand (1-3 year commitment)"
  }
}
```

---

## สรุป Cost Optimization Checklist

```
Cost Optimization ด้วย Terraform:
─────────────────────────────────────────────────────────────
✅ ใช้ Infracost ใน CI/CD เพื่อ estimate cost ก่อน apply
✅ ใช้ Spot instances สำหรับ workloads ที่ interrupt ได้
✅ ตั้ง RDS/EC2 schedule stop สำหรับ non-prod environments
✅ ใช้ S3 Intelligent Tiering หรือ Lifecycle rules
✅ ตั้ง Budget alerts ก่อนถึง 80% ของ budget
✅ Enable Cost Anomaly Detection
✅ ใช้ mandatory tags สำหรับ cost allocation
✅ รัน Lambda detector หาก unused resources เป็นประจำ
✅ ใช้ Auto Scaling แทน fixed capacity
✅ พิจารณา Compute Optimizer recommendations
✅ ใช้ Reserved Instances/Savings Plans สำหรับ stable workloads
✅ Monitor และ eliminate idle resources สม่ำเสมอ
```

---

*จบ Part 99: Cost Optimization with Terraform*
