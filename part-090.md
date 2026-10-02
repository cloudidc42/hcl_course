# Part 090: Checkov Security Scanner
## ขั้นตอนที่ 891-900: Checkov - Static Analysis สำหรับ IaC

---

## ขั้นตอนที่ 891: ติดตั้ง Checkov

### วิธีการติดตั้ง

```bash
# 1. ติดตั้งด้วย pip (แนะนำสำหรับ development)
pip install checkov
pip install checkov --upgrade  # อัพเดท

# ติดตั้ง version เฉพาะ
pip install checkov==3.2.0

# 2. ติดตั้งด้วย pip3 (Python 3)
pip3 install checkov

# 3. ติดตั้งด้วย Homebrew (macOS)
brew install checkov

# 4. Docker (ไม่ต้องติดตั้ง Python)
docker pull bridgecrew/checkov:latest

# รัน Checkov ผ่าน Docker
docker run --tty \
  --volume "$(pwd)":/tf \
  --workdir /tf \
  bridgecrew/checkov \
  --directory /tf

# 5. ติดตั้งด้วย pipx (isolated environment)
pipx install checkov

# ตรวจสอบ version
checkov --version
# Output: Checkov version: 3.x.x

# ตรวจสอบว่า Checkov พร้อมใช้งาน
checkov --help
```

### ทดสอบการติดตั้ง

```bash
# สร้างไฟล์ทดสอบ
cat > /tmp/test.tf << 'EOF'
resource "aws_s3_bucket" "example" {
  bucket = "my-test-bucket"
}
EOF

# รัน Checkov บนไฟล์ทดสอบ
checkov -f /tmp/test.tf

# Output จะแสดง:
# Passed checks: X, Failed checks: Y, Skipped checks: 0
# Check: CKV_AWS_21: "Ensure all data stored in the S3 bucket have versioning enabled"
# FAILED for resource: aws_s3_bucket.example
```

---

## ขั้นตอนที่ 892: Checkov พื้นฐาน

### รันบน Directory และ Files

```bash
# สแกน directory ทั้งหมด
checkov -d .

# สแกนไฟล์เดียว
checkov -f main.tf

# สแกนหลายไฟล์
checkov -f main.tf variables.tf outputs.tf

# สแกน Terraform directory พร้อม recursive
checkov -d /path/to/terraform --recursive

# สแกน Terraform และ CloudFormation ใน directory เดียวกัน
checkov -d . --framework terraform cloudformation

# ดู list ของ checks ทั้งหมด
checkov --list-checks

# ดู checks สำหรับ framework เฉพาะ
checkov --list-checks --framework terraform
checkov --list-checks --framework cloudformation
checkov --list-checks --framework kubernetes
```

### Output Format

```bash
# CLI output (default)
checkov -d .

# JSON output
checkov -d . -o json

# JUnit XML (สำหรับ CI/CD)
checkov -d . -o junitxml

# SARIF (สำหรับ GitHub Code Scanning)
checkov -d . -o sarif

# CSV output
checkov -d . -o csv

# Markdown output
checkov -d . -o markdown

# บันทึกผลลัพธ์ลงไฟล์
checkov -d . -o json > checkov-results.json
checkov -d . -o junitxml > checkov-results.xml

# หลาย format พร้อมกัน
checkov -d . -o json -o junitxml
```

### ตัวอย่าง Output

```
Passed checks: 15, Failed checks: 8, Skipped checks: 2

Check: CKV_AWS_21: "Ensure all data stored in the S3 bucket have versioning enabled"
	PASSED for resource: aws_s3_bucket.secure_bucket
	File: /modules/s3/main.tf:1-10

Check: CKV_AWS_19: "Ensure all data stored in the S3 bucket is securely encrypted"
	FAILED for resource: aws_s3_bucket.example_bucket
	File: /main.tf:15-20
	Guide: https://docs.prismacloud.io/en/enterprise-edition/policy-reference/aws-policies/s3-policies/s3-14-data-encrypted-at-rest

Check: CKV_AWS_18: "Ensure the S3 bucket has access logging enabled"
	SKIPPED for resource: aws_s3_bucket.example_bucket
	Suppress comment: "CKV_AWS_18: Access logging not needed for temp bucket"
	File: /main.tf:15-20
```

---

## ขั้นตอนที่ 893: การกรองและการข้าม Checks

### Filtering Checks

```bash
# รัน specific check เท่านั้น
checkov -d . --check CKV_AWS_21

# รัน หลาย specific checks
checkov -d . --check CKV_AWS_21,CKV_AWS_19,CKV_AWS_18

# ข้าม specific check
checkov -d . --skip-check CKV_AWS_21

# ข้ามหลาย checks
checkov -d . --skip-check CKV_AWS_21,CKV_AWS_19

# รัน checks ตาม severity
checkov -d . --check-threshold HIGH  # เฉพาะ HIGH และ CRITICAL
checkov -d . --check-threshold MEDIUM

# กรองตาม tag
checkov -d . --runner-filter-tags HIPAA,CIS

# Framework filtering
checkov -d . --framework terraform      # เฉพาะ Terraform
checkov -d . --framework cloudformation # เฉพาะ CloudFormation
checkov -d . --framework arm            # Azure Resource Manager
checkov -d . --framework kubernetes     # Kubernetes manifests
checkov -d . --framework dockerfile     # Dockerfiles
checkov -d . --framework secrets        # Secrets detection
checkov -d . --framework all            # ทุก framework

# External checks directory
checkov -d . --external-checks-dir /path/to/custom/checks
```

### Inline Skip Comments

```hcl
# ❌ ไม่ดี - ข้าม check โดยไม่มีเหตุผล
resource "aws_s3_bucket" "example" {
  #checkov:skip=CKV_AWS_21
  bucket = "my-bucket"
}

# ✅ ดี - ข้าม check พร้อมระบุเหตุผล
resource "aws_s3_bucket" "access_logs" {
  #checkov:skip=CKV_AWS_18:This IS the access logs bucket - no need to log its own access
  bucket = "my-app-access-logs"
}

resource "aws_instance" "bastion" {
  #checkov:skip=CKV_AWS_8:Detailed monitoring adds cost and is handled by CloudWatch Agent instead
  #checkov:skip=CKV_AWS_79:IMDSv2 enforced via launch template not instance resource
  ami           = "ami-0abcdef1234567890"
  instance_type = "t3.micro"
}

# ✅ ข้ามหลาย checks บน resource เดียวกัน
resource "aws_security_group" "internal_lb" {
  # CKV_AWS_25: Internal LB does not need 443-only ingress
  #checkov:skip=CKV_AWS_25:Internal ALB accepts HTTP within VPC only
  name = "internal-lb-sg"
  vpc_id = aws_vpc.main.id
}
```

---

## ขั้นตอนที่ 894: Checkov Rules สำหรับ AWS

### CKV_AWS_* สำหรับ S3

```bash
# S3 Checks
CKV_AWS_19  # Ensure all data stored in S3 is encrypted
CKV_AWS_20  # Ensure S3 bucket has versioning enabled
CKV_AWS_21  # Ensure all data stored in the S3 bucket have versioning enabled
CKV_AWS_52  # Ensure S3 bucket has MFA delete enabled
CKV_AWS_55  # Ensure S3 bucket has ignore public ACLs enabled
CKV_AWS_56  # Ensure S3 bucket has 'restrict_public_buckets' enabled
CKV_AWS_70  # Ensure S3 bucket has access logging configured
CKV_AWS_93  # Ensure S3 bucket policy does not allow public read access
CKV_AWS_145 # Ensure S3 bucket uses customer-managed KMS key

# ตัวอย่าง S3 ที่ผ่านทุก check
resource "aws_s3_bucket" "fully_compliant" {
  bucket = "my-compliant-bucket"
}

resource "aws_s3_bucket_versioning" "compliant" {    # CKV_AWS_21
  bucket = aws_s3_bucket.fully_compliant.id
  versioning_configuration { status = "Enabled" }
}

resource "aws_s3_bucket_server_side_encryption_configuration" "compliant" {  # CKV_AWS_145
  bucket = aws_s3_bucket.fully_compliant.id
  rule {
    apply_server_side_encryption_by_default {
      sse_algorithm     = "aws:kms"
      kms_master_key_id = aws_kms_key.s3.arn  # Customer-managed key
    }
    bucket_key_enabled = true
  }
}

resource "aws_s3_bucket_public_access_block" "compliant" {  # CKV_AWS_55, CKV_AWS_56
  bucket                  = aws_s3_bucket.fully_compliant.id
  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}

resource "aws_s3_bucket_logging" "compliant" {  # CKV_AWS_70
  bucket        = aws_s3_bucket.fully_compliant.id
  target_bucket = aws_s3_bucket.access_logs.id
  target_prefix = "my-compliant-bucket/"
}
```

### CKV_AWS_* สำหรับ EC2 & IAM

```bash
# EC2 Checks
CKV_AWS_8   # Ensure detailed monitoring is enabled for EC2
CKV_AWS_79  # Ensure Instance Metadata Service Version 1 is not enabled
CKV_AWS_126 # Ensure EC2 instance does not have public IP
CKV_AWS_131 # Ensure EC2 instances does not have unrestricted outbound access
CKV_AWS_189 # Ensure EBS Volume is encrypted by KMS CMK

# IAM Checks
CKV_AWS_40  # Ensure IAM policies are attached only to groups or roles
CKV_AWS_41  # Ensure no hard coded AWS access key and secret key
CKV_AWS_62  # Ensure IAM role has a permissions boundary
CKV_AWS_274 # Disallow IAM roles, users, and groups from using the AWS AdministratorAccess policy
CKV_AWS_275 # Disallow IAM roles, users, and groups from using the AWS PowerUserAccess policy

# ตัวอย่าง EC2 ที่ผ่าน checks
resource "aws_instance" "compliant" {
  ami                     = "ami-0abcdef1234567890"
  instance_type           = "t3.micro"
  monitoring              = true    # CKV_AWS_8
  associate_public_ip_address = false  # CKV_AWS_126
  
  metadata_options {
    http_endpoint               = "enabled"
    http_tokens                 = "required"   # CKV_AWS_79 (IMDSv2)
    http_put_response_hop_limit = 1
  }
  
  root_block_device {
    encrypted   = true          # CKV_AWS_189
    kms_key_id  = aws_kms_key.ec2.arn
  }
}
```

### CKV_AWS_* สำหรับ RDS, EKS, Lambda

```bash
# RDS Checks
CKV_AWS_16  # Ensure all data stored in the RDS instance is securely encrypted
CKV_AWS_17  # Ensure all data stored in the RDS instance is not publicly accessible
CKV_AWS_23  # Ensure RDS database has automatic minor version upgrades enabled
CKV_AWS_133 # Ensure that RDS instances has backup policy
CKV_AWS_157 # Ensure RDS database cluster is Multi-AZ
CKV_AWS_211 # Ensure RDS uses a modern CaCert

# EKS Checks
CKV_AWS_37  # Ensure Amazon EKS control plane logging enabled
CKV_AWS_38  # Ensure Amazon EKS control plane API is not publicly accessible
CKV_AWS_39  # Ensure Amazon EKS public endpoint not accessible to 0.0.0.0/0
CKV_AWS_58  # Ensure EKS Cluster has Secrets Encryption enabled

# Lambda Checks
CKV_AWS_45  # Ensure no hard-coded credentials exist in lambda environment
CKV_AWS_50  # Ensure lambda functions with existing tracing capability have it enabled
CKV_AWS_272 # Ensure Lambda function should have X-Ray tracing enabled
CKV_AWS_116 # Ensure Lambda function is subscribed to an SQS dead-letter queue
CKV_AWS_117 # Ensure Lambda function is inside a VPC
```

---

## ขั้นตอนที่ 895: Checkov Rules สำหรับ Azure และ GCP

### CKV_AZURE_* Rules

```bash
# Azure Storage
CKV_AZURE_3   # Ensure that storage account access is not publicly accessible
CKV_AZURE_33  # Ensure Storage Account is using the latest version of TLS encryption
CKV_AZURE_36  # Ensure that Storage Blob Container is encrypted with CMK

# Azure Network
CKV_AZURE_10  # Ensure that Microsoft Antimalware is configured to automatically update
CKV_AZURE_12  # Ensure that Network Security Group Flow Log retention period is greater than 90 days
CKV_AZURE_13  # Ensure that RDP access is not allowed from the Internet

# Azure Key Vault
CKV_AZURE_42  # Ensure the key vault is recoverable
CKV_AZURE_110 # Ensure that Azure key vault disables public network access

# ตัวอย่าง Azure Storage ที่ compliant
resource "azurerm_storage_account" "compliant" {
  name                     = "mystorageaccount"
  resource_group_name      = azurerm_resource_group.main.name
  location                 = azurerm_resource_group.main.location
  account_tier             = "Standard"
  account_replication_type = "GRS"
  
  # CKV_AZURE_3: Deny public blob access
  allow_nested_items_to_be_public = false  # CKV_AZURE_3
  
  # CKV_AZURE_33: Minimum TLS 1.2
  min_tls_version = "TLS1_2"  # CKV_AZURE_33
  
  # Enable HTTPS traffic only
  enable_https_traffic_only = true
  
  # Infrastructure encryption
  infrastructure_encryption_enabled = true
  
  network_rules {
    default_action = "Deny"
    bypass         = ["AzureServices"]
  }
}
```

### CKV_GCP_* Rules

```bash
# GCP Compute
CKV_GCP_30  # Ensure firewall rule does not allow ingress from 0.0.0.0/0
CKV_GCP_32  # Ensure 'Block Project-Wide SSH Keys' is enabled for VM instances
CKV_GCP_38  # Ensure VM disks are encrypted with Customer-Managed Encryption Keys

# GCP Storage
CKV_GCP_28  # Ensure that Cloud Storage bucket is not anonymously or publicly accessible
CKV_GCP_62  # Ensure that logging is enabled for all Cloud Storage Buckets

# GCP Kubernetes
CKV_GCP_19  # Ensure GKE basic auth is not enabled
CKV_GCP_24  # Ensure PodSecurityPolicy is enabled on Kubernetes cluster
CKV_GCP_66  # Ensure GKE master authorized networks is enabled

# ตัวอย่าง GCP VM ที่ compliant
resource "google_compute_instance" "compliant" {
  name         = "my-instance"
  machine_type = "e2-medium"
  zone         = "us-central1-a"
  
  # CKV_GCP_32: Block project-wide SSH
  metadata = {
    block-project-ssh-keys = "true"  # CKV_GCP_32
    enable-oslogin         = "true"
  }
  
  # CKV_GCP_38: Customer-managed encryption key
  boot_disk {
    initialize_params {
      image = "debian-cloud/debian-11"
    }
    kms_key_self_link = google_kms_crypto_key.disk.id  # CKV_GCP_38
  }
  
  # Disable external IP
  network_interface {
    network = google_compute_network.main.id
    # ✅ No access_config block = no external IP
  }
}
```

---

## ขั้นตอนที่ 896: Custom Checks

### Python Custom Check

```python
# custom_checks/check_s3_lifecycle.py
from checkov.common.models.enums import CheckCategories, CheckResult
from checkov.terraform.checks.resource.base_resource_check import BaseResourceCheck

class S3LifecycleCheck(BaseResourceCheck):
    def __init__(self):
        name = "Ensure S3 bucket has lifecycle configuration enabled"
        id = "CKV_CUSTOM_1"
        supported_resources = ["aws_s3_bucket"]
        categories = [CheckCategories.STORAGE]
        
        super().__init__(
            name=name,
            id=id,
            categories=categories,
            supported_resources=supported_resources
        )
    
    def scan_resource_conf(self, conf):
        # ตรวจสอบว่ามี lifecycle_rule
        lifecycle_rules = conf.get("lifecycle_rule", [])
        
        if not lifecycle_rules:
            return CheckResult.FAILED
        
        # ตรวจสอบว่า lifecycle rule มี enabled = true
        for rule in lifecycle_rules:
            if isinstance(rule, dict):
                enabled = rule.get("enabled", [False])
                if isinstance(enabled, list):
                    enabled = enabled[0]
                if enabled:
                    return CheckResult.PASSED
        
        return CheckResult.FAILED

scanner = S3LifecycleCheck()
```

### YAML Custom Check

```yaml
# custom_checks/check_required_tags.yaml
metadata:
  name: "Ensure all EC2 instances have required tags"
  id: "CKV2_CUSTOM_1"
  category: "GENERAL_SECURITY"
  severity: "MEDIUM"
scope:
  provider: "terraform"

definition:
  and:
    - cond_type: "attribute"
      resource_types:
        - "aws_instance"
      attribute: "tags.Environment"
      operator: "exists"
    - cond_type: "attribute"
      resource_types:
        - "aws_instance"
      attribute: "tags.Owner"
      operator: "exists"
    - cond_type: "attribute"
      resource_types:
        - "aws_instance"
      attribute: "tags.CostCenter"
      operator: "exists"
```

```yaml
# custom_checks/check_sg_description.yaml
metadata:
  name: "Ensure security group rules have descriptions"
  id: "CKV2_CUSTOM_2"
  category: "NETWORKING"
  severity: "LOW"
scope:
  provider: "terraform"

definition:
  and:
    - cond_type: "attribute"
      resource_types:
        - "aws_security_group"
      attribute: "description"
      operator: "not_equals"
      value: "Managed by Terraform"
    - cond_type: "attribute"
      resource_types:
        - "aws_security_group"
      attribute: "description"
      operator: "not_equals"
      value: ""
```

```bash
# รัน Checkov พร้อม custom checks
checkov -d . \
  --external-checks-dir ./custom_checks \
  --include-all-checkov-policies

# รัน custom YAML checks
checkov -d . \
  --external-checks-dir ./custom_checks/yaml
```

---

## ขั้นตอนที่ 897: CI/CD Integration

### GitHub Actions Integration

```yaml
# .github/workflows/checkov.yml
name: Checkov Security Scan

on:
  push:
    branches: [main, develop]
    paths:
      - "**/*.tf"
      - "**/*.tfvars"
  pull_request:
    branches: [main]
    paths:
      - "**/*.tf"
      - "**/*.tfvars"

jobs:
  checkov:
    runs-on: ubuntu-latest
    name: Checkov Terraform Scan
    
    steps:
      - name: Checkout code
        uses: actions/checkout@v4
      
      # Method 1: Using official Checkov action
      - name: Run Checkov (Action)
        uses: bridgecrewio/checkov-action@master
        with:
          directory: .
          framework: terraform
          output_format: sarif
          output_file_path: checkov-results.sarif
          soft_fail: false
          skip_check: CKV_AWS_18,CKV_AWS_52  # Documented exceptions
      
      # Method 2: Using pip
      - name: Install Checkov
        run: pip install checkov
      
      - name: Run Checkov
        id: checkov
        run: |
          checkov \
            --directory . \
            --framework terraform \
            --output json \
            --output-file-path checkov-results.json \
            --skip-check CKV_AWS_18 \
            --soft-fail || true  # Don't fail pipeline yet
      
      - name: Upload SARIF to GitHub Security
        uses: github/codeql-action/upload-sarif@v3
        if: always()
        with:
          sarif_file: checkov-results.sarif
          category: checkov
      
      # Annotate PRs with findings
      - name: Check Checkov Results
        if: always()
        run: |
          FAILED=$(cat checkov-results.json | jq '.summary.failed')
          echo "Checkov found $FAILED failed checks"
          if [ "$FAILED" -gt 0 ]; then
            echo "❌ Checkov failed with $FAILED issues"
            cat checkov-results.json | jq '.results.failed_checks[] | {check_id: .check_id, resource: .resource, file: .file_path}'
            exit 1
          fi
          echo "✅ All Checkov checks passed"

  checkov-pr-comment:
    runs-on: ubuntu-latest
    if: github.event_name == 'pull_request'
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Run Checkov for PR annotation
        uses: bridgecrewio/checkov-action@master
        with:
          directory: .
          framework: terraform
          output_format: github_failed_only  # GitHub annotations only
          soft_fail: true
```

### GitLab CI Integration

```yaml
# .gitlab-ci.yml
stages:
  - validate
  - security
  - plan
  - apply

variables:
  TF_IN_AUTOMATION: "true"

checkov:
  stage: security
  image: bridgecrew/checkov:latest
  script:
    - checkov
        --directory .
        --framework terraform
        --output junitxml
        --output-file-path gl-sast-report.xml
        --skip-check CKV_AWS_18
        --soft-fail
  artifacts:
    reports:
      junit: gl-sast-report.xml
    paths:
      - gl-sast-report.xml
    expire_in: 1 week
  only:
    changes:
      - "**/*.tf"
      - "**/*.tfvars"

checkov-strict:
  stage: security
  image: bridgecrew/checkov:latest
  script:
    - checkov
        --directory .
        --framework terraform
        --check-threshold HIGH
        # ❌ จะ fail pipeline ถ้า HIGH/CRITICAL findings
  only:
    - main
    - production
```

---

## ขั้นตอนที่ 898: Baseline Files

### สร้างและใช้ Baseline

```bash
# 1. สร้าง baseline จาก existing state (acceptance ปัญหาที่มีอยู่แล้ว)
checkov -d . --create-baseline

# สร้างไฟล์ .checkov.baseline ใน directory
# เนื้อหาตัวอย่าง:
```

```json
{
    "results": {
        "passed_checks": [],
        "failed_checks": [
            {
                "check_id": "CKV_AWS_18",
                "bc_check_id": "BC_AWS_LOGGING_19",
                "check_name": "Ensure the S3 bucket has access logging enabled",
                "resource": "aws_s3_bucket.legacy_bucket",
                "file_path": "./main.tf",
                "file_line_range": [1, 10],
                "repo_file_path": "/main.tf",
                "resource_address": "aws_s3_bucket.legacy_bucket",
                "severity": "MEDIUM"
            }
        ]
    }
}
```

```bash
# 2. รัน Checkov ด้วย baseline (จะ ignore baseline findings)
checkov -d . --baseline .checkov.baseline

# ✅ เฉพาะ NEW findings เท่านั้นที่จะ fail
# Findings ใน baseline จะถูก skip

# 3. อัพเดท baseline เมื่อแก้ไข issues แล้ว
checkov -d . --create-baseline  # recreate baseline

# 4. ดูความแตกต่างจาก baseline
checkov -d . --baseline .checkov.baseline --output json | \
  jq '.results.failed_checks | length'
```

---

## ขั้นตอนที่ 899: Advanced Checkov Usage

### Soft Fail Mode

```bash
# soft-fail: แสดง findings แต่ไม่ fail pipeline (exit code 0)
checkov -d . --soft-fail

# soft-fail สำหรับ specific checks เท่านั้น
checkov -d . --soft-fail-on CKV_AWS_18,CKV_AWS_21

# ใช้ใน development/staging แต่ strict ใน production
ENVIRONMENT=${CI_ENVIRONMENT:-development}

if [ "$ENVIRONMENT" = "production" ]; then
  checkov -d . --framework terraform
else
  checkov -d . --framework terraform --soft-fail
fi
```

### Checkov Configuration File

```yaml
# .checkov.yaml
directory:
  - .
framework:
  - terraform
  - cloudformation
skip-check:
  - CKV_AWS_18  # S3 access logging - handled separately
  - CKV_AWS_52  # S3 MFA delete - requires manual setup
output:
  - cli
  - json
output-file-path: checkov-results
compact: true
quiet: false
log-level: WARNING
download-external-modules: false
evaluate-variables: true
```

```bash
# รัน ด้วย config file
checkov --config-file .checkov.yaml

# หรือ Checkov จะหา .checkov.yaml อัตโนมัติถ้ามีใน directory
checkov -d .
```

### Terraform Variables Evaluation

```bash
# ✅ Evaluate variables (Checkov จะใช้ค่า variables จริง)
checkov -d . \
  --evaluate-variables \
  --var-file production.tfvars

# ✅ ใช้กับ Terraform plan output (แม่นยำที่สุด)
terraform plan -out=tfplan
terraform show -json tfplan > tfplan.json

checkov -f tfplan.json \
  --framework terraform_plan \
  --output json
```

---

## ขั้นตอนที่ 900: Checkov + Prisma Cloud / Bridgecrew

### API Integration

```bash
# ✅ รัน Checkov พร้อม Prisma Cloud API key
checkov -d . \
  --bc-api-key "${PRISMA_CLOUD_API_KEY}" \
  --repo-id "company/my-terraform-repo" \
  --branch "main"

# จะ:
# 1. รัน checks ทั้งหมด
# 2. ส่งผลลัพธ์ไป Prisma Cloud Dashboard
# 3. รับ custom policies จาก Prisma Cloud

# ✅ GitHub Actions พร้อม Prisma Cloud
name: Checkov + Prisma Cloud

on: [push, pull_request]

jobs:
  scan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Run Checkov with Prisma Cloud
        uses: bridgecrewio/checkov-action@master
        env:
          PRISMA_API_URL: ${{ secrets.PRISMA_API_URL }}
        with:
          api-key: ${{ secrets.PRISMA_CLOUD_API_KEY }}
          directory: .
          framework: terraform
          output_format: sarif
          output_file_path: results.sarif
          repo_root_for_plan_enrichment: .
```

### Pre-commit Hook

```yaml
# .pre-commit-config.yaml
repos:
  - repo: https://github.com/bridgecrewio/checkov
    rev: 3.2.0  # ใช้ version เฉพาะ
    hooks:
      - id: checkov
        name: Checkov
        description: Run Checkov on Terraform files
        entry: checkov
        language: python
        args:
          - --directory
          - .
          - --framework
          - terraform
          - --skip-check
          - CKV_AWS_18,CKV_AWS_52
        pass_filenames: false
        types_or: [terraform]
```

```bash
# ติดตั้ง pre-commit hooks
pip install pre-commit
pre-commit install

# รัน manually
pre-commit run checkov --all-files

# รัน ก่อน commit อัตโนมัติ
git commit -m "Add S3 bucket"
# Checkov จะรันก่อน commit สำเร็จ
```

### Checkov Summary Report

```bash
# Script สำหรับ generate comprehensive report

#!/bin/bash
# generate-checkov-report.sh

set -e

TIMESTAMP=$(date +%Y%m%d_%H%M%S)
REPORT_DIR="checkov-reports/${TIMESTAMP}"
mkdir -p "${REPORT_DIR}"

echo "🔍 Running Checkov Security Scan..."

# Run with multiple outputs
checkov \
  --directory . \
  --framework terraform \
  --output json \
  --output cli \
  --output-file-path "${REPORT_DIR}/results" \
  --skip-check CKV_AWS_18 \
  --soft-fail

# Parse JSON results
PASSED=$(jq '.summary.passed' "${REPORT_DIR}/results.json" 2>/dev/null || echo 0)
FAILED=$(jq '.summary.failed' "${REPORT_DIR}/results.json" 2>/dev/null || echo 0)
SKIPPED=$(jq '.summary.skipped' "${REPORT_DIR}/results.json" 2>/dev/null || echo 0)

echo ""
echo "📊 Checkov Results Summary"
echo "=========================="
echo "✅ Passed:  ${PASSED}"
echo "❌ Failed:  ${FAILED}"
echo "⏭️  Skipped: ${SKIPPED}"
echo ""

if [ "${FAILED}" -gt 0 ]; then
  echo "❌ Failed Checks:"
  jq -r '.results.failed_checks[] | "  [\(.check_id)] \(.check_name)\n    Resource: \(.resource)\n    File: \(.file_path):\(.file_line_range[0])"' \
    "${REPORT_DIR}/results.json" 2>/dev/null
  echo ""
  echo "Reports saved to: ${REPORT_DIR}/"
  exit 1
else
  echo "✅ All checks passed!"
  echo "Reports saved to: ${REPORT_DIR}/"
fi
```

---

## สรุป Checkov Commands Reference

### Quick Reference Card

```bash
# ===== Installation =====
pip install checkov                              # Install
pip install checkov --upgrade                    # Upgrade
docker pull bridgecrew/checkov                   # Docker

# ===== Basic Usage =====
checkov -d .                                     # Scan directory
checkov -f main.tf                               # Scan file
checkov --list-checks                            # List all checks
checkov --version                                # Show version

# ===== Output Formats =====
checkov -d . -o cli                              # CLI (default)
checkov -d . -o json                             # JSON
checkov -d . -o junitxml                         # JUnit XML (CI)
checkov -d . -o sarif                            # SARIF (GitHub)

# ===== Filtering =====
checkov -d . --check CKV_AWS_21                  # Run specific check
checkov -d . --skip-check CKV_AWS_18             # Skip specific check
checkov -d . --framework terraform               # Framework filter
checkov -d . --check-threshold HIGH              # Severity threshold

# ===== Advanced =====
checkov -d . --soft-fail                         # Don't fail on issues
checkov -d . --create-baseline                   # Create baseline
checkov -d . --baseline .checkov.baseline        # Use baseline
checkov -d . --evaluate-variables                # Evaluate variables
checkov -d . --external-checks-dir ./custom      # Custom checks
checkov -f tfplan.json --framework terraform_plan # Scan plan file

# ===== CI/CD =====
checkov -d . -o junitxml > results.xml           # JUnit for CI
checkov -d . -o sarif > results.sarif            # SARIF for GitHub
```

### Checkov Rule Categories

| Category | Check Count | Key Checks |
|----------|-------------|------------|
| S3 Security | 15+ | CKV_AWS_19, 21, 55, 56 |
| IAM Security | 20+ | CKV_AWS_40, 41, 62, 274 |
| EC2 Security | 15+ | CKV_AWS_8, 79, 126 |
| RDS Security | 10+ | CKV_AWS_16, 17, 133 |
| EKS Security | 8+ | CKV_AWS_37, 38, 58 |
| Networking | 12+ | CKV_AWS_24, 25, 131 |
| Encryption | 20+ | CKV_AWS_19, 189, 211 |
| Logging | 10+ | CKV_AWS_18, 50, 70 |
| CloudTrail | 5+ | CKV_AWS_36, 67, 252 |
| Lambda | 8+ | CKV_AWS_45, 50, 116, 117 |

---

## ภาพรวมหลักสูตร Security (Part 081-090)

| Part | หัวข้อ | Steps |
|------|--------|-------|
| 081 | IaC Security Introduction & Tools | 801-810 |
| 082 | S3 Bucket Misconfigurations | 811-820 |
| 083 | IAM Security & Privilege Escalation | 821-830 |
| 084 | VPC & Network Misconfigurations | 831-840 |
| 085 | Encryption at Rest | 841-850 |
| 086 | Encryption in Transit | 851-860 |
| 087 | Logging & Monitoring | 861-870 |
| 088 | Public Exposure Vulnerabilities | 871-880 |
| 089 | Compliance Frameworks (CIS/NIST/SOC2/PCI) | 881-890 |
| 090 | Checkov Security Scanner | 891-900 |

---

*จบส่วน Security & Misconfiguration Analysis (Steps 801-900) - ครอบคลุมการรักษาความปลอดภัยใน IaC อย่างครบถ้วน*
