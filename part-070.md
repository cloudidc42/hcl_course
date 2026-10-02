# Part 070: Custom Conditions & Preconditions
## Custom Conditions และ Preconditions
### Steps 691-700

---

## บทนำ (Introduction)

Terraform 1.2+ นำเสนอ `precondition` และ `postcondition` blocks และ Terraform 1.5+ นำเสนอ `check` blocks สิ่งเหล่านี้ช่วยให้เราสร้าง assertions ที่ครอบคลุมมากกว่า variable validation โดยสามารถอ้างอิงค่าจาก resources ที่สร้างแล้วได้

---

## Step 691: Check Blocks (Terraform 1.5+)

### Check blocks สำหรับ continuous assertion

```hcl
# ==========================================
# CHECK BLOCKS (Terraform 1.5+)
# ==========================================

# Check blocks ทำงานหลัง apply และบน terraform plan
# ไม่ block การ apply - แค่แสดง warning

# Check 1: ตรวจสอบว่า ALB พร้อมใช้งาน
check "alb_state" {
  data "aws_lb" "main" {
    arn = aws_lb.main.arn
  }

  assert {
    condition     = data.aws_lb.main.state == "active"
    error_message = "ALB '${aws_lb.main.name}' is not in active state."
  }
}

# Check 2: ตรวจสอบ RDS status
check "rds_status" {
  data "aws_db_instance" "main" {
    db_instance_identifier = aws_db_instance.main.identifier
  }

  assert {
    condition     = data.aws_db_instance.main.db_instance_status == "available"
    error_message = "RDS instance '${aws_db_instance.main.identifier}' is not available."
  }
}

# Check 3: ตรวจสอบว่า S3 bucket ไม่ public
check "s3_not_public" {
  data "aws_s3_bucket_public_access_block" "main" {
    bucket = aws_s3_bucket.data.bucket
  }

  assert {
    condition = (
      data.aws_s3_bucket_public_access_block.main.block_public_acls &&
      data.aws_s3_bucket_public_access_block.main.block_public_policy &&
      data.aws_s3_bucket_public_access_block.main.ignore_public_acls &&
      data.aws_s3_bucket_public_access_block.main.restrict_public_buckets
    )
    error_message = "S3 bucket '${aws_s3_bucket.data.bucket}' must have all public access blocked."
  }
}

# Check 4: ตรวจสอบ ECS Service health
check "ecs_service_running" {
  data "aws_ecs_service" "app" {
    cluster_arn  = aws_ecs_cluster.main.arn
    service_name = aws_ecs_service.app.name
  }

  assert {
    condition     = data.aws_ecs_service.app.running_count >= data.aws_ecs_service.app.desired_count
    error_message = "ECS service has fewer running tasks (${data.aws_ecs_service.app.running_count}) than desired (${data.aws_ecs_service.app.desired_count})."
  }
}

# Check 5: ตรวจสอบ External dependencies
check "external_api_available" {
  data "http" "api_health" {
    url = "https://api.example.com/health"
  }

  assert {
    condition     = data.http.api_health.status_code == 200
    error_message = "External API at api.example.com is not healthy (status: ${data.http.api_health.status_code})."
  }
}
```

---

## Step 692: Precondition in Resources

### Precondition ใน resource blocks

```hcl
# ==========================================
# PRECONDITION IN RESOURCES
# ==========================================

# precondition ทำงานก่อนสร้าง resource (during plan)
# ถ้า condition = false -> plan fails

resource "aws_db_instance" "main" {
  identifier     = var.db_identifier
  engine         = var.engine
  instance_class = var.instance_class

  lifecycle {
    # Precondition 1: Production ต้องมี Multi-AZ
    precondition {
      condition     = var.environment != "prod" || var.multi_az
      error_message = "Production databases must have multi_az = true."
    }

    # Precondition 2: Production ต้องมี deletion protection
    precondition {
      condition     = var.environment != "prod" || var.deletion_protection
      error_message = "Production databases must have deletion_protection = true."
    }

    # Precondition 3: Storage ต้องมากพอสำหรับ Production
    precondition {
      condition = (
        var.environment != "prod" ||
        var.allocated_storage >= 100
      )
      error_message = "Production databases must have at least 100 GB of allocated storage."
    }

    # Precondition 4: ตรวจสอบ engine + instance class compatibility
    precondition {
      condition = !(
        var.engine == "aurora-mysql" &&
        startswith(var.instance_class, "db.t2.")
      )
      error_message = "Aurora MySQL does not support db.t2 instance classes. Use db.t3.medium or larger."
    }
  }
}

# ==========================================
# PRECONDITION กับ DATA SOURCES
# ==========================================

data "aws_ami" "app" {
  most_recent = true
  owners      = ["self"]

  filter {
    name   = "name"
    values = ["myapp-*"]
  }

  lifecycle {
    # ตรวจสอบว่า AMI ที่ได้เป็น AMI ของเรา ไม่ใช่ public AMI
    postcondition {
      condition     = self.owner_id == data.aws_caller_identity.current.account_id
      error_message = "Selected AMI is not owned by this AWS account. Verify AMI ownership."
    }

    # ตรวจสอบว่า AMI ไม่เก่าเกิน 30 วัน
    postcondition {
      condition = timecmp(
        self.creation_date,
        timeadd(timestamp(), "-720h")  # 30 days
      ) > 0
      error_message = "AMI is older than 30 days. Please build a fresh AMI."
    }
  }
}

resource "aws_instance" "app" {
  ami           = data.aws_ami.app.id
  instance_type = var.instance_type

  lifecycle {
    # ตรวจสอบว่า AMI ที่เลือกมี correct architecture
    precondition {
      condition = contains(
        ["x86_64", "arm64"],
        data.aws_ami.app.architecture
      )
      error_message = "AMI architecture must be x86_64 or arm64."
    }
  }
}
```

---

## Step 693: Postcondition in Resources

### Postcondition - ตรวจสอบหลัง apply

```hcl
# ==========================================
# POSTCONDITION IN RESOURCES
# ==========================================

# postcondition ทำงานหลังสร้าง/อัพเดต resource
# อ้างอิง self.<attribute> เพื่อ access resource values

resource "aws_instance" "web" {
  ami           = var.ami_id
  instance_type = var.instance_type
  subnet_id     = var.subnet_id

  lifecycle {
    # ตรวจสอบหลัง create: ต้องได้รับ private IP
    postcondition {
      condition     = self.private_ip != ""
      error_message = "EC2 instance must have a private IP after creation."
    }

    # ตรวจสอบ: instance ต้องอยู่ใน correct AZ
    postcondition {
      condition = contains(
        var.allowed_azs,
        self.availability_zone
      )
      error_message = "Instance was placed in ${self.availability_zone} which is not in the allowed AZs: ${join(", ", var.allowed_azs)}."
    }

    # ตรวจสอบ: instance state ต้องเป็น running
    postcondition {
      condition     = self.instance_state == "running"
      error_message = "Instance is not in 'running' state after creation."
    }
  }
}

resource "aws_lb" "main" {
  name    = "${var.project}-${var.environment}-alb"
  subnets = var.subnet_ids

  lifecycle {
    # ตรวจสอบ: ALB ต้องมี DNS name
    postcondition {
      condition     = self.dns_name != ""
      error_message = "ALB must have a DNS name after creation."
    }

    # ตรวจสอบ: ALB ต้องใช้ correct subnet count
    postcondition {
      condition     = length(self.subnets) >= 2
      error_message = "ALB must be deployed in at least 2 subnets for high availability."
    }
  }
}

resource "aws_s3_bucket" "data" {
  bucket = "${var.project}-${var.environment}-data"

  lifecycle {
    # ตรวจสอบ: bucket region ต้องถูกต้อง
    postcondition {
      condition = self.region == var.expected_region
      error_message = "S3 bucket was created in ${self.region} but expected ${var.expected_region}."
    }
  }
}

# ==========================================
# POSTCONDITION กับ ENCRYPTION
# ==========================================

resource "aws_ebs_volume" "data" {
  availability_zone = var.az
  size              = var.size_gb
  encrypted         = true
  kms_key_id        = var.kms_key_id

  lifecycle {
    postcondition {
      condition     = self.encrypted == true
      error_message = "EBS volume must be encrypted."
    }

    postcondition {
      condition     = self.kms_key_id != ""
      error_message = "EBS volume must use a customer-managed KMS key."
    }
  }
}
```

---

## Step 694: Precondition in Outputs

### Precondition ใน output blocks

```hcl
# ==========================================
# PRECONDITION IN OUTPUTS
# ==========================================

output "db_connection_string" {
  description = "Database connection string"
  value = "postgresql://${aws_db_instance.main.username}@${aws_db_instance.main.endpoint}/${aws_db_instance.main.db_name}"

  precondition {
    condition     = aws_db_instance.main.status == "available"
    error_message = "Database must be in 'available' status before generating connection string."
  }
}

output "app_url" {
  description = "Application URL"
  value       = "https://${var.domain_name}"

  precondition {
    condition     = aws_acm_certificate_validation.main.id != ""
    error_message = "SSL certificate must be validated before the app URL can be used securely."
  }
}

output "api_endpoint" {
  description = "API Gateway endpoint"
  value       = aws_api_gateway_stage.main.invoke_url

  precondition {
    condition     = aws_api_gateway_stage.main.stage_name != ""
    error_message = "API Gateway stage must be deployed before outputting endpoint."
  }
}

# ==========================================
# MULTIPLE PRECONDITIONS IN OUTPUT
# ==========================================

output "production_db_endpoint" {
  description = "Production database endpoint"
  value       = aws_db_instance.main.endpoint

  # ตรวจสอบว่าเป็น production และปลอดภัย
  precondition {
    condition     = var.environment == "prod"
    error_message = "This output is only available in production environment."
  }

  precondition {
    condition     = aws_db_instance.main.storage_encrypted
    error_message = "Database must be encrypted before exposing the endpoint."
  }

  precondition {
    condition     = !aws_db_instance.main.publicly_accessible
    error_message = "Database must not be publicly accessible in production."
  }
}
```

---

## Step 695: Assertion vs Precondition vs Validation

### ความแตกต่างระหว่าง assertion types

```hcl
# ==========================================
# THREE TYPES OF ASSERTIONS
# ==========================================

# TYPE 1: variable validation
# - ทำงานก่อน plan
# - ตรวจสอบ input values เท่านั้น
# - ไม่สามารถอ้างอิง resources
variable "instance_type" {
  type = string
  validation {
    condition     = can(regex("^[a-z][0-9][a-z]?\\.[a-z0-9]+$", var.instance_type))
    error_message = "Invalid EC2 instance type format."
  }
}

# TYPE 2: precondition (in resource/output lifecycle)
# - ทำงานระหว่าง plan
# - สามารถอ้างอิง other variables และ data sources
# - ถ้าล้มเหลว: plan/apply fails
resource "aws_instance" "app" {
  ami           = var.ami_id
  instance_type = var.instance_type

  lifecycle {
    precondition {
      # สามารถอ้างอิง data sources และ variables ได้
      condition     = data.aws_ami.app.architecture == "x86_64"
      error_message = "AMI must be x86_64 architecture."
    }
  }
}

# TYPE 3: postcondition (in resource/output lifecycle)
# - ทำงานหลัง apply
# - สามารถอ้างอิง self (resource ที่สร้าง)
# - ถ้าล้มเหลว: apply fails with helpful message
resource "aws_lb" "main" {
  lifecycle {
    postcondition {
      # อ้างอิง self เพื่อ verify resource values
      condition     = self.dns_name != ""
      error_message = "ALB must have a DNS name."
    }
  }
}

# TYPE 4: check blocks (Terraform 1.5+)
# - ทำงานหลัง apply
# - ไม่ block apply (แค่ warning)
# - สามารถใช้ data sources ใน assert
check "alb_healthy" {
  data "aws_lb" "main" {
    arn = aws_lb.main.arn
  }

  assert {
    condition     = data.aws_lb.main.state == "active"
    error_message = "ALB must be in active state."
    # ถ้าล้มเหลว: แสดง warning แต่ไม่ fail
  }
}

# ==========================================
# WHEN TO USE EACH
# ==========================================

# variable validation: ตรวจสอบ user input format
# precondition: ตรวจสอบ config ก่อน apply
# postcondition: ยืนยัน resource state หลัง apply
# check: continuous monitoring ที่ไม่ block
```

---

## Step 696: Writing Effective Condition Expressions

### การเขียน condition expression ที่ดี

```hcl
# ==========================================
# EFFECTIVE CONDITION EXPRESSIONS
# ==========================================

# Pattern 1: Simple boolean
lifecycle {
  precondition {
    condition     = var.encrypted == true
    error_message = "Encryption must be enabled."
  }
}

# Pattern 2: Contains check
lifecycle {
  precondition {
    condition = contains(
      ["mysql", "postgres", "mariadb"],
      var.engine
    )
    error_message = "Engine must be mysql, postgres, or mariadb."
  }
}

# Pattern 3: Regex check
lifecycle {
  precondition {
    condition     = can(regex("^[a-z0-9-]+$", var.name))
    error_message = "Name must contain only lowercase letters, numbers, and hyphens."
  }
}

# Pattern 4: Comparison
lifecycle {
  precondition {
    condition     = var.storage_gb >= 20
    error_message = "Storage must be at least 20 GB."
  }
}

# Pattern 5: Environment-based
lifecycle {
  precondition {
    condition = (
      var.environment != "prod" ||
      (var.multi_az && var.deletion_protection)
    )
    error_message = "Production requires both multi_az and deletion_protection."
  }
}

# Pattern 6: CIDR validation
lifecycle {
  precondition {
    condition     = can(cidrhost(var.vpc_cidr, 0))
    error_message = "Invalid VPC CIDR block format."
  }
}

# Pattern 7: List validation
lifecycle {
  precondition {
    condition     = length(var.subnet_ids) >= 2
    error_message = "At least 2 subnets required."
  }
}

# Pattern 8: Complex AND/OR
lifecycle {
  precondition {
    condition = (
      (var.environment == "dev" && var.instance_class == "db.t3.micro") ||
      (var.environment == "staging" && contains(["db.t3.micro", "db.t3.medium"], var.instance_class)) ||
      (var.environment == "prod" && startswith(var.instance_class, "db.r"))
    )
    error_message = "Instance class does not match environment requirements. Dev: t3.micro, Staging: t3 series, Prod: r-series."
  }
}

# Pattern 9: Self reference (postcondition)
lifecycle {
  postcondition {
    condition     = self.arn != ""
    error_message = "Resource ARN must not be empty after creation."
  }
}

# Pattern 10: Numeric comparison with expression
lifecycle {
  postcondition {
    condition     = length(self.availability_zones) >= 2
    error_message = "Resource must span at least 2 availability zones."
  }
}
```

---

## Step 697: Error Messages Best Practices

### การเขียน error messages ที่ดี

```hcl
# ==========================================
# ERROR MESSAGE BEST PRACTICES
# ==========================================

# BAD error messages
lifecycle {
  precondition {
    condition     = var.environment == "prod"
    error_message = "Wrong environment."  # ไม่บอกว่าผิดอะไร หรือต้องทำอะไร
  }
}

lifecycle {
  precondition {
    condition     = var.storage_gb >= 100
    error_message = "Storage too small"  # ไม่บอกว่าต้องเป็นเท่าไหร่
  }
}

# GOOD error messages - บอก: อะไรผิด, ค่าที่ได้, ค่าที่ต้องการ, วิธีแก้
lifecycle {
  precondition {
    condition = var.environment == "prod"
    error_message = "This module can only be used in 'prod' environment. Got '${var.environment}'. Change environment variable to 'prod'."
  }
}

lifecycle {
  precondition {
    condition = var.storage_gb >= 100
    error_message = "Production database storage must be at least 100 GB. Got ${var.storage_gb} GB. Increase storage_gb to at least 100."
  }
}

lifecycle {
  precondition {
    condition = contains(["us-east-1", "ap-southeast-1"], var.region)
    error_message = "Region '${var.region}' is not approved for deployment. Approved regions: us-east-1, ap-southeast-1. Contact the platform team to add new regions."
  }
}

# Pattern: Error message with dynamic context
lifecycle {
  postcondition {
    condition = self.instance_state == "running"
    error_message = "EC2 instance '${self.id}' is in state '${self.instance_state}' instead of 'running'. Check CloudWatch logs for startup errors."
  }
}

# ==========================================
# ERROR MESSAGE TEMPLATES
# ==========================================

# Template 1: Simple violation
# "Resource X must have Y enabled/set/configured."

# Template 2: With current value
# "X must be Y or more. Got Z."

# Template 3: With allowed values
# "X must be one of: a, b, c. Got 'z'."

# Template 4: With action
# "X is Y but must be Z. Set X to Z in your variables file."

# Template 5: Environment-specific
# "In production environment, X must be Y. Got Z."

# Template 6: Contact info for policies
# "X violates policy Y. Contact security@company.com for exceptions."
```

---

## Step 698: Compliance Conditions

### การใช้ conditions สำหรับ compliance

```hcl
# ==========================================
# COMPLIANCE: ENSURE ENCRYPTION
# ==========================================

resource "aws_s3_bucket_server_side_encryption_configuration" "compliance" {
  bucket = aws_s3_bucket.data.id

  rule {
    apply_server_side_encryption_by_default {
      sse_algorithm     = "aws:kms"
      kms_master_key_id = var.kms_key_id
    }
    bucket_key_enabled = true
  }

  lifecycle {
    precondition {
      condition     = var.kms_key_id != ""
      error_message = "S3 bucket must use customer-managed KMS key for compliance. Provide kms_key_id."
    }
  }
}

check "encryption_compliance" {
  data "aws_s3_bucket_server_side_encryption_configuration" "check" {
    bucket = aws_s3_bucket.data.bucket
  }

  assert {
    condition = (
      length(data.aws_s3_bucket_server_side_encryption_configuration.check.rule) > 0 &&
      data.aws_s3_bucket_server_side_encryption_configuration.check.rule[0].apply_server_side_encryption_by_default[0].sse_algorithm == "aws:kms"
    )
    error_message = "S3 bucket '${aws_s3_bucket.data.bucket}' must use KMS encryption for compliance."
  }
}

# ==========================================
# COMPLIANCE: ENSURE PUBLIC ACCESS BLOCKED
# ==========================================

resource "aws_s3_bucket_public_access_block" "compliance" {
  bucket = aws_s3_bucket.data.id

  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}

check "public_access_compliance" {
  data "aws_s3_bucket_public_access_block" "check" {
    bucket = aws_s3_bucket.data.bucket
  }

  assert {
    condition = (
      data.aws_s3_bucket_public_access_block.check.block_public_acls &&
      data.aws_s3_bucket_public_access_block.check.block_public_policy &&
      data.aws_s3_bucket_public_access_block.check.ignore_public_acls &&
      data.aws_s3_bucket_public_access_block.check.restrict_public_buckets
    )
    error_message = "S3 bucket '${aws_s3_bucket.data.bucket}' must have all public access blocked for compliance."
  }
}

# ==========================================
# COMPLIANCE: REQUIRED TAGS
# ==========================================

locals {
  required_tags = ["Environment", "Project", "Owner", "CostCenter", "DataClassification"]
}

resource "aws_instance" "compliant" {
  ami           = var.ami_id
  instance_type = var.instance_type
  tags          = var.tags

  lifecycle {
    precondition {
      condition = alltrue([
        for required_tag in local.required_tags :
        contains(keys(var.tags), required_tag)
      ])
      error_message = "Instance must have all required tags: ${join(", ", local.required_tags)}. Missing: ${join(", ", [for t in local.required_tags : t if !contains(keys(var.tags), t)])}."
    }
  }
}

# ==========================================
# COMPLIANCE: SECURE CIDR RANGES
# ==========================================

variable "management_cidrs" {
  type    = list(string)
  default = ["10.0.0.0/8"]
}

resource "aws_security_group_rule" "ssh" {
  security_group_id = aws_security_group.bastion.id
  type              = "ingress"
  from_port         = 22
  to_port           = 22
  protocol          = "tcp"
  cidr_blocks       = var.management_cidrs

  lifecycle {
    # ห้ามเปิด SSH จาก internet
    precondition {
      condition = !contains(var.management_cidrs, "0.0.0.0/0")
      error_message = "SSH must not be open to the internet (0.0.0.0/0). Use VPN or bastion host CIDR."
    }

    # ตรวจสอบว่า CIDRs เป็น private ranges
    precondition {
      condition = alltrue([
        for cidr in var.management_cidrs :
        (
          startswith(cidr, "10.") ||
          can(regex("^172\\.(1[6-9]|2[0-9]|3[01])\\.", cidr)) ||
          startswith(cidr, "192.168.")
        )
      ])
      error_message = "SSH access must be restricted to private (RFC1918) IP ranges only."
    }
  }
}

# ==========================================
# COMPLIANCE: ENSURE PROPER BACKUP
# ==========================================

resource "aws_db_instance" "compliant" {
  identifier              = var.db_identifier
  engine                  = var.engine
  instance_class          = var.instance_class
  backup_retention_period = var.backup_retention

  lifecycle {
    precondition {
      condition = (
        var.environment != "prod" ||
        var.backup_retention >= 14
      )
      error_message = "Production databases must retain backups for at least 14 days per compliance policy."
    }

    postcondition {
      condition     = self.backup_retention_period >= 7
      error_message = "Database backup retention is ${self.backup_retention_period} days, but must be at least 7 days."
    }
  }
}
```

---

## Step 699: Conditions for Modules and Data Sources

### Conditions ในส่วนต่างๆ

```hcl
# ==========================================
# CONDITIONS IN DATA SOURCES
# ==========================================

data "aws_kms_key" "main" {
  key_id = var.kms_key_id

  lifecycle {
    postcondition {
      condition     = self.enabled == true
      error_message = "KMS key '${self.key_id}' must be enabled."
    }

    postcondition {
      condition     = self.key_state == "Enabled"
      error_message = "KMS key is in state '${self.key_state}' instead of 'Enabled'."
    }

    postcondition {
      condition = (
        self.key_manager == "CUSTOMER" ||
        var.allow_aws_managed_keys
      )
      error_message = "Must use customer-managed KMS key (not AWS-managed) for this resource type."
    }
  }
}

data "aws_vpc" "selected" {
  id = var.vpc_id

  lifecycle {
    postcondition {
      condition     = self.state == "available"
      error_message = "VPC '${self.id}' must be in 'available' state."
    }

    postcondition {
      condition     = self.enable_dns_hostnames == true
      error_message = "VPC must have DNS hostnames enabled."
    }

    postcondition {
      condition     = self.enable_dns_support == true
      error_message = "VPC must have DNS support enabled."
    }
  }
}

# ==========================================
# CONDITIONS IN MODULES (PRECONDITIONS)
# ==========================================

# Module ที่มี preconditions ตรวจสอบ inputs
# modules/ecs-service/main.tf

resource "aws_ecs_service" "main" {
  name            = var.service_name
  cluster         = var.cluster_arn
  task_definition = aws_ecs_task_definition.main.arn
  desired_count   = var.desired_count

  lifecycle {
    precondition {
      condition     = can(regex("^arn:aws:ecs:", var.cluster_arn))
      error_message = "cluster_arn must be a valid ECS cluster ARN starting with 'arn:aws:ecs:'."
    }

    precondition {
      condition = (
        var.desired_count >= var.min_count &&
        var.desired_count <= var.max_count
      )
      error_message = "desired_count (${var.desired_count}) must be between min_count (${var.min_count}) and max_count (${var.max_count})."
    }
  }
}

# ==========================================
# TERRAFORM TEST WITH CONDITIONS
# ==========================================

# tests/compliance_test.tftest.hcl

run "test_encryption_compliance" {
  command = apply

  variables {
    environment = "prod"
    kms_key_id  = "arn:aws:kms:..."
    encrypted   = true
  }

  # Test ว่า resource ถูกสร้างด้วย encryption
  assert {
    condition     = aws_s3_bucket_server_side_encryption_configuration.compliance.rule[0].apply_server_side_encryption_by_default[0].sse_algorithm == "aws:kms"
    error_message = "S3 bucket must use KMS encryption in production."
  }
}

run "test_no_public_access" {
  command = apply

  assert {
    condition = (
      aws_s3_bucket_public_access_block.compliance.block_public_acls == true &&
      aws_s3_bucket_public_access_block.compliance.block_public_policy == true
    )
    error_message = "S3 bucket must block all public access."
  }
}
```

---

## Step 700: Conditions in CI/CD

### การใช้ conditions ใน CI/CD pipelines

```yaml
# ==========================================
# GITHUB ACTIONS WORKFLOW WITH CONDITIONS
# .github/workflows/terraform.yml
# ==========================================

name: Terraform Compliance Check

on:
  pull_request:
    branches: [main]
  push:
    branches: [main]

jobs:
  compliance:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3

      - name: Setup Terraform
        uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: "1.6.0"

      - name: Terraform Init
        run: terraform init

      - name: Terraform Validate
        run: terraform validate
        # ตรวจสอบ syntax และ type errors

      - name: Terraform Plan with Conditions
        run: terraform plan -var-file=prod.tfvars
        # preconditions และ postconditions จะ run ตอน plan

      - name: Check for Compliance Issues
        run: |
          # Run plan และ filter สำหรับ check block failures
          terraform plan -json | jq '.resource_changes[] | select(.change.actions | contains(["create"])) | .address'
```

```hcl
# ==========================================
# COMPLIANCE MODULE สำหรับทุก environment
# modules/compliance-baseline/main.tf
# ==========================================

# ตรวจสอบว่า environment มี compliance requirements ครบ

check "s3_encryption" {
  for_each = toset(var.s3_bucket_ids)

  data "aws_s3_bucket_server_side_encryption_configuration" "check" {
    bucket = each.key
  }

  assert {
    condition = length(data.aws_s3_bucket_server_side_encryption_configuration.check.rule) > 0
    error_message = "S3 bucket '${each.key}' must have server-side encryption enabled."
  }
}

check "rds_encryption" {
  for_each = toset(var.rds_instance_ids)

  data "aws_db_instance" "check" {
    db_instance_identifier = each.key
  }

  assert {
    condition     = data.aws_db_instance.check.storage_encrypted
    error_message = "RDS instance '${each.key}' must have storage encryption enabled."
  }
}

check "security_groups_no_world_ssh" {
  for_each = toset(var.security_group_ids)

  data "aws_security_group" "check" {
    id = each.key
  }

  assert {
    condition = !anytrue([
      for rule in data.aws_security_group.check.ingress :
      rule.from_port <= 22 && rule.to_port >= 22 && contains(rule.cidr_blocks, "0.0.0.0/0")
    ])
    error_message = "Security group '${each.key}' must not allow SSH from 0.0.0.0/0."
  }
}

check "cloudtrail_enabled" {
  data "aws_cloudtrail" "main" {
    name = var.cloudtrail_name
  }

  assert {
    condition     = data.aws_cloudtrail.main.enable_logging == true
    error_message = "CloudTrail logging must be enabled for compliance."
  }

  assert {
    condition     = data.aws_cloudtrail.main.log_file_validation_enabled == true
    error_message = "CloudTrail log file validation must be enabled for integrity."
  }
}

# ==========================================
# COMPLETE COMPLIANCE EXAMPLE
# ==========================================

resource "aws_s3_bucket" "secure_data" {
  bucket = "${var.project}-${var.environment}-secure-data"

  lifecycle {
    # Pre-check: ชื่อ bucket ต้องมี environment
    precondition {
      condition     = can(regex(var.environment, "${var.project}-${var.environment}-secure-data"))
      error_message = "Bucket name must include environment: ${var.environment}."
    }

    # Post-check: ตรวจสอบ bucket region
    postcondition {
      condition     = self.region == var.region
      error_message = "Bucket was created in wrong region: ${self.region}."
    }
  }
}

# Check continuous compliance
check "data_bucket_compliance" {
  data "aws_s3_bucket" "check" {
    bucket = aws_s3_bucket.secure_data.bucket
  }

  assert {
    condition     = data.aws_s3_bucket.check.bucket_domain_name != ""
    error_message = "Data bucket must be properly configured."
  }
}
```

---

## สรุป (Summary)

### Condition Types Summary:

| Type | เมื่อ Evaluate | Block การ apply? | อ้างอิงได้ |
|------|---------------|-----------------|------------|
| `variable validation` | ก่อน plan | Yes | var เท่านั้น |
| `precondition` | ระหว่าง plan | Yes | var, data, local |
| `postcondition` | หลัง apply | Yes | self, var |
| `check assert` | หลัง apply | No (warning) | data sources |

### Best Practices สำหรับ Conditions:

1. **Error messages ที่ actionable** - บอกว่าต้องทำอะไร ไม่ใช่แค่ว่าผิด
2. **ใช้ precondition** สำหรับ environment-specific rules
3. **ใช้ postcondition** สำหรับ verify resource configuration
4. **ใช้ check blocks** สำหรับ non-blocking compliance monitoring
5. **ใช้ variable validation** สำหรับ format/type checking ของ inputs

### Decision Guide:
```
ต้องการตรวจสอบอะไร?
├── Format/type ของ user input?
│   └── variable validation
├── Business rule ก่อน apply?
│   └── lifecycle precondition
├── Resource state หลัง apply?
│   └── lifecycle postcondition
└── Ongoing compliance (non-blocking)?
    └── check block
```

---

*จบ Part 070 - Custom Conditions & Preconditions*

---

## บทสรุปรวม (Series Summary: Part 061-070)

ในบทเหล่านี้เราได้เรียนรู้:

| Part | หัวข้อ | Key Concepts |
|------|--------|--------------|
| 061 | Variables: Deep Dive | validation blocks, CIDR/regex/enum validation |
| 062 | Complex Variable Types | object, map, list, optional() |
| 063 | Output Values | module interfaces, remote state, sensitive |
| 064 | Modules: Deep Dive | architecture, testing, terraform-docs |
| 065 | Module Versioning | registry, semver, version constraints |
| 066 | Module Composition | patterns, composition, anti-patterns |
| 067 | Count & For_each | meta-arguments, index shifting, splat |
| 068 | Lifecycle Rules | create_before_destroy, prevent_destroy, conditions |
| 069 | Provider Aliasing | multi-region, multi-account, assume_role |
| 070 | Custom Conditions | check blocks, precondition, postcondition |
