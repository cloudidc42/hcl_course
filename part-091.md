# Part 91: TFSec Static Analysis (Steps 901-910)

## การวิเคราะห์ความปลอดภัยแบบ Static Analysis ด้วย TFSec

---

## Step 901: TFSec คืออะไร และทำไมต้องใช้?

### ความเป็นมาของ TFSec

TFSec เป็นเครื่องมือ Static Analysis Security Testing (SAST) ที่ถูกออกแบบมาเพื่อตรวจสอบโค้ด Terraform โดยเฉพาะ พัฒนาโดย Aqua Security และต่อมาได้รวมเข้าเป็นส่วนหนึ่งของ **Trivy** ซึ่งเป็น All-in-One security scanner ชื่อดัง

**Timeline ของ TFSec:**
- 2019: TFSec เริ่มต้นเป็น standalone tool
- 2022: Aqua Security ประกาศรวม TFSec เข้ากับ Trivy
- 2023+: `trivy config` เป็น command หลัก แต่ `tfsec` ยังคงทำงานได้เพื่อ backward compatibility

### ทำไมต้องใช้ Static Analysis สำหรับ Terraform?

```
ปัญหาที่พบบ่อยใน Infrastructure as Code:
┌─────────────────────────────────────────┐
│ 1. S3 bucket เปิดสาธารณะโดยไม่ตั้งใจ   │
│ 2. Security Group อนุญาต 0.0.0.0/0      │
│ 3. RDS ไม่เข้ารหัสข้อมูล               │
│ 4. IAM policy มีสิทธิ์มากเกินไป        │
│ 5. ไม่มี logging/auditing               │
│ 6. Secrets ใน plain text                │
└─────────────────────────────────────────┘
```

TFSec ช่วยตรวจจับปัญหาเหล่านี้ **ก่อน** ที่จะ deploy จริงๆ

### เปรียบเทียบแนวทางการตรวจสอบความปลอดภัย

| แนวทาง | เมื่อไหร่ตรวจ | ค่าใช้จ่าย | ความเสี่ยง |
|--------|--------------|-----------|-----------|
| Manual Review | Pre-deploy | สูง (เวลา) | ผิดพลาดได้ |
| Static Analysis (TFSec) | Pre-deploy | ต่ำ | น้อย |
| Runtime Scanning | Post-deploy | ปานกลาง | มีแล้ว! |
| Penetration Testing | เป็นระยะ | สูงมาก | ช้า |

**หลักการ Shift Left Security**: ตรวจสอบความปลอดภัยให้เร็วที่สุดในกระบวนการ development

---

## Step 902: การติดตั้ง TFSec/Trivy

### วิธีที่ 1: Homebrew (macOS/Linux)

```bash
# ติดตั้ง Trivy (รวม TFSec functionality)
brew install trivy

# ตรวจสอบ version
trivy --version

# ติดตั้ง TFSec แบบ standalone (legacy)
brew install tfsec
tfsec --version
```

### วิธีที่ 2: Binary Download (Linux/Windows)

```bash
# Linux AMD64 - ดาวน์โหลด Trivy
TRIVY_VERSION="0.50.0"
curl -sfL https://raw.githubusercontent.com/aquasecurity/trivy/main/contrib/install.sh | sh -s -- -b /usr/local/bin v${TRIVY_VERSION}

# หรือดาวน์โหลด binary โดยตรง
wget https://github.com/aquasecurity/trivy/releases/download/v${TRIVY_VERSION}/trivy_${TRIVY_VERSION}_Linux-64bit.tar.gz
tar -xzf trivy_${TRIVY_VERSION}_Linux-64bit.tar.gz
sudo mv trivy /usr/local/bin/

# TFSec standalone binary
TFSEC_VERSION="1.28.5"
curl -s https://raw.githubusercontent.com/aquasecurity/tfsec/master/scripts/install_linux.sh | bash
# หรือ
wget https://github.com/aquasecurity/tfsec/releases/download/v${TFSEC_VERSION}/tfsec-linux-amd64
chmod +x tfsec-linux-amd64
sudo mv tfsec-linux-amd64 /usr/local/bin/tfsec
```

### วิธีที่ 3: Docker

```bash
# รัน Trivy ผ่าน Docker
docker pull aquasec/trivy:latest

# รัน scan ผ่าน Docker
docker run --rm \
  -v $(pwd):/workspace \
  -w /workspace \
  aquasec/trivy:latest config .

# รัน TFSec ผ่าน Docker
docker pull aquasec/tfsec:latest
docker run --rm \
  -v $(pwd):/src \
  aquasec/tfsec /src
```

### วิธีที่ 4: GitHub Actions (ไม่ต้องติดตั้ง)

```yaml
# ใน GitHub Actions ใช้ action โดยตรง
- name: Run Trivy vulnerability scanner
  uses: aquasecurity/trivy-action@master
  with:
    scan-type: 'config'
    scan-ref: '.'
```

### วิธีที่ 5: Windows (Chocolatey / Scoop)

```powershell
# Chocolatey
choco install trivy

# Scoop
scoop install trivy

# หรือ WSL2 แล้วใช้วิธี Linux
```

### ตรวจสอบการติดตั้ง

```bash
# ตรวจสอบ Trivy
trivy --version
# Output: Version: 0.50.0

# ตรวจสอบ TFSec
tfsec --version
# Output: v1.28.5

# อัพเดต database
trivy image --download-db-only
```

---

## Step 903: คำสั่งพื้นฐาน - trivy config vs tfsec

### คำสั่งใหม่: `trivy config`

```bash
# สแกน directory ปัจจุบัน
trivy config .

# สแกน directory เฉพาะเจาะจง
trivy config ./terraform/

# สแกน file เดียว
trivy config main.tf

# Output พร้อม detail เพิ่มเติม
trivy config --severity HIGH,CRITICAL .

# ดู all findings รวม LOW
trivy config --severity LOW,MEDIUM,HIGH,CRITICAL .

# Output format แบบ JSON
trivy config --format json .

# Output format แบบ SARIF (สำหรับ GitHub)
trivy config --format sarif --output results.sarif .

# ไม่แสดง progress bar
trivy config --quiet .

# Exit code ตาม severity
trivy config --exit-code 1 --severity HIGH,CRITICAL .
```

### คำสั่งเก่า: `tfsec` (Legacy แต่ยังใช้งานได้)

```bash
# สแกนพื้นฐาน
tfsec .

# สแกน directory เฉพาะ
tfsec ./aws/

# Output format ต่างๆ
tfsec --format json .
tfsec --format csv .
tfsec --format checkstyle .
tfsec --format junit .
tfsec --format sarif .
tfsec --format markdown .
tfsec --format text .
tfsec --format lovely .  # format สวยงาม (default)

# Filter ตาม severity
tfsec --minimum-severity HIGH .

# รัน specific checks เท่านั้น
tfsec --include-checks AVD-AWS-0089,AVD-AWS-0057 .

# ไม่รัน specific checks
tfsec --exclude-checks AVD-AWS-0089 .

# รัน แล้ว fail ถ้ามี CRITICAL
tfsec --severity-overrides AVD-AWS-0089:CRITICAL . && echo "PASSED" || echo "FAILED"
```

### ความแตกต่างระหว่าง trivy config และ tfsec

```
┌──────────────────┬────────────────────────────┬───────────────────────────┐
│ Feature          │ trivy config               │ tfsec (legacy)            │
├──────────────────┼────────────────────────────┼───────────────────────────┤
│ Status           │ Active development         │ Maintenance mode          │
│ Scan types       │ Terraform, K8s, Docker,    │ Terraform only            │
│                  │ Helm, CloudFormation        │                           │
│ Rule format      │ Rego + built-in            │ Rego + built-in           │
│ Output formats   │ json, sarif, table, etc.   │ json, csv, sarif, etc.    │
│ CI/CD support    │ Excellent                  │ Good                      │
│ Recommendation   │ ✅ ใช้สำหรับโปรเจคใหม่    │ ⚠️ Legacy support only    │
└──────────────────┴────────────────────────────┴───────────────────────────┘
```

---

## Step 904: Output Formats ทั้งหมด

### 1. Beautiful/Lovely Format (Default - อ่านง่าย)

```bash
tfsec --format lovely .
# หรือ
tfsec .
```

ตัวอย่าง output:
```
Result #1 CRITICAL Bucket does not have encryption enabled
───────────────────────────────────────────────────────────────────────────────
  aws/s3.tf:10
───────────────────────────────────────────────────────────────────────────────
    7 | resource "aws_s3_bucket" "example" {
    8 |   bucket = "my-bucket"
    9 |   acl    = "private"
   10 | }
───────────────────────────────────────────────────────────────────────────────
          ID aws-s3-enable-bucket-encryption
      Impact The bucket objects could be read if compromised
  Resolution Enable AES-256 or aws:kms encryption of S3 bucket
  More info https://aquasecurity.github.io/tfsec/...
```

### 2. JSON Format

```bash
tfsec --format json . > tfsec-results.json

# หรือ Trivy
trivy config --format json --output results.json .
```

โครงสร้าง JSON:
```json
{
  "results": [
    {
      "rule_id": "AVD-AWS-0089",
      "long_id": "aws-s3-enable-bucket-encryption",
      "rule_description": "Bucket does not have encryption enabled",
      "rule_provider": "aws",
      "rule_service": "s3",
      "impact": "The bucket objects could be read if compromised",
      "resolution": "Enable AES-256 or aws:kms encryption of S3 bucket",
      "links": ["https://aquasecurity.github.io/tfsec/..."],
      "description": "Bucket does not have encryption enabled",
      "severity": "HIGH",
      "warning": false,
      "status": 0,
      "resource": "aws_s3_bucket.example",
      "location": {
        "filename": "aws/s3.tf",
        "start_line": 7,
        "end_line": 11
      }
    }
  ],
  "summary": {
    "passed": 12,
    "warnings": 3,
    "failed": 5,
    "ignored": 2,
    "critical": 1,
    "high": 2,
    "medium": 1,
    "low": 1
  }
}
```

### 3. CSV Format

```bash
tfsec --format csv . > tfsec-results.csv
```

```csv
rule_id,long_id,rule_description,severity,filename,line
AVD-AWS-0089,aws-s3-enable-bucket-encryption,Bucket does not have encryption enabled,HIGH,aws/s3.tf,7
AVD-AWS-0057,aws-ec2-no-public-ingress-sgr,An ingress security group rule allows traffic from /0.,CRITICAL,aws/sg.tf,15
```

### 4. SARIF Format (Static Analysis Results Interchange Format)

```bash
tfsec --format sarif . > results.sarif
# หรือ
trivy config --format sarif --output results.sarif .
```

SARIF ถูกใช้ใน GitHub Code Scanning:
```json
{
  "version": "2.1.0",
  "$schema": "https://json.schemastore.org/sarif-2.1.0.json",
  "runs": [
    {
      "tool": {
        "driver": {
          "name": "tfsec",
          "version": "1.28.5",
          "rules": [...]
        }
      },
      "results": [
        {
          "ruleId": "AVD-AWS-0089",
          "level": "error",
          "message": {
            "text": "Bucket does not have encryption enabled"
          },
          "locations": [
            {
              "physicalLocation": {
                "artifactLocation": {
                  "uri": "aws/s3.tf"
                },
                "region": {
                  "startLine": 7,
                  "endLine": 11
                }
              }
            }
          ]
        }
      ]
    }
  ]
}
```

### 5. JUnit Format (สำหรับ CI tools)

```bash
tfsec --format junit . > tfsec-junit.xml
```

```xml
<?xml version="1.0" encoding="UTF-8"?>
<testsuites>
  <testsuite name="tfsec" tests="5" failures="3" errors="0">
    <testcase classname="aws.s3" name="Bucket does not have encryption enabled">
      <failure message="AVD-AWS-0089: Bucket does not have encryption enabled">
        File: aws/s3.tf, Line: 7
      </failure>
    </testcase>
    <testcase classname="aws.ec2" name="Security group allows 0.0.0.0/0">
      <failure message="AVD-AWS-0057: ...">
        File: aws/sg.tf, Line: 15
      </failure>
    </testcase>
  </testsuite>
</testsuites>
```

### 6. Checkstyle Format

```bash
tfsec --format checkstyle . > checkstyle.xml
```

### 7. Markdown Format

```bash
tfsec --format markdown . > SECURITY_REPORT.md
```

### 8. Text Format

```bash
tfsec --format text .
```

---

## Step 905: Severity Levels และการกรอง

### ระดับ Severity ของ TFSec

```
CRITICAL  ════════════════  ปัญหาที่ต้องแก้ไขทันที!
          ตัวอย่าง: Security group อนุญาต 0.0.0.0/0 บน port สำคัญ
          
HIGH      ════════════════  ปัญหาสำคัญที่ควรแก้ไขโดยเร็ว
          ตัวอย่าง: S3 ไม่มีการเข้ารหัส
          
MEDIUM    ════════════════  ปัญหาควรพิจารณาแก้ไข
          ตัวอย่าง: ไม่มี access logging
          
LOW       ════════════════  ปัญหาเล็กน้อย แต่ควรรับทราบ
          ตัวอย่าง: ไม่มี description ใน security group rules
```

### การกรองตาม Severity

```bash
# แสดงเฉพาะ CRITICAL
tfsec --minimum-severity CRITICAL .

# แสดง HIGH และ CRITICAL
tfsec --minimum-severity HIGH .

# แสดงทั้งหมด (รวม LOW)
tfsec --minimum-severity LOW .

# ด้วย trivy
trivy config --severity CRITICAL .
trivy config --severity HIGH,CRITICAL .
trivy config --severity LOW,MEDIUM,HIGH,CRITICAL .

# Exit code based on severity
# exit 0 = ผ่าน, exit 1 = มี findings
tfsec --minimum-severity HIGH . ; echo "Exit code: $?"
```

### ตั้งค่า Exit Codes

```bash
# สำหรับ CI/CD - fail เมื่อมี CRITICAL หรือ HIGH
trivy config \
  --exit-code 1 \
  --severity HIGH,CRITICAL \
  .

# ถ้าต้องการ ignore exit code (ดูผลเฉยๆ)
trivy config . || true
```

---

## Step 906: การ Ignore Findings

### วิธีที่ 1: Inline Comment ใน Terraform

```hcl
# การใช้ tfsec:ignore comment
resource "aws_s3_bucket" "logs" {
  bucket = "my-logs-bucket"
  
  # tfsec:ignore:AVD-AWS-0089
  # เหตุผล: bucket นี้ใช้เก็บ log ที่ไม่มีข้อมูลสำคัญ
  # reviewed by: john.doe@company.com on 2024-01-15
}

# หรือ ignore หลาย rules พร้อมกัน
resource "aws_s3_bucket" "backup" {
  bucket = "backup-bucket"
  
  # tfsec:ignore:AVD-AWS-0089
  # tfsec:ignore:AVD-AWS-0132
}

# Ignore ใน attribute เฉพาะ
resource "aws_security_group_rule" "allow_http" {
  type        = "ingress"
  from_port   = 80
  to_port     = 80
  protocol    = "tcp"
  cidr_blocks = ["0.0.0.0/0"] # tfsec:ignore:AVD-AWS-0057
  description = "Allow HTTP from internet (public endpoint)"
  
  security_group_id = aws_security_group.web.id
}
```

### วิธีที่ 2: Configuration File (.tfsec/config.json)

```bash
mkdir -p .tfsec
cat > .tfsec/config.json << 'EOF'
{
  "minimum_severity": "HIGH",
  "exclude": [
    "AVD-AWS-0089",
    "AVD-AWS-0132"
  ],
  "include": [],
  "exclude_ignores": false,
  "ignore_hcl_errors": false
}
EOF
```

### วิธีที่ 3: Configuration File YAML

```bash
cat > .tfsec/config.yml << 'EOF'
# TFSec Configuration
minimum_severity: MEDIUM

# Rules ที่ต้องการ exclude ทั้งหมด
exclude:
  - AVD-AWS-0089  # S3 encryption - handled by default encryption
  - AVD-AWS-0132  # S3 versioning - not needed for temp files

# Override severity สำหรับ specific rules
severity_overrides:
  AVD-AWS-0057: CRITICAL  # ยกระดับ Security Group rule เป็น CRITICAL

# Ignore specific resources
ignore_checks:
  - resource: "aws_s3_bucket.logs"
    checks:
      - AVD-AWS-0089
      - AVD-AWS-0132
EOF
```

### วิธีที่ 4: Trivy config file (.trivy.yaml หรือ trivy.yaml)

```yaml
# trivy.yaml หรือ .trivy.yaml
scan:
  security-checks:
    - config

format: table

severity:
  - HIGH
  - CRITICAL

misconfiguration:
  # File patterns ที่ต้องการ ignore
  ignore-policy: ".trivyignore"
```

### .trivyignore file

```bash
cat > .trivyignore << 'EOF'
# Ignore specific findings
# Format: <avd-id> [exp:YYYY-MM-DD] [<reason>]
AVD-AWS-0089 exp:2024-12-31 Encrypted at bucket policy level
AVD-AWS-0132 Versioning not required for ephemeral data
EOF
```

---

## Step 907: TFSec Rules Catalog - AWS Rules สำคัญ

### AVD-AWS-0089: S3 Bucket Encryption

```hcl
# ❌ ผิด - ไม่มี encryption
resource "aws_s3_bucket" "bad" {
  bucket = "my-bucket"
}

# ✅ ถูก - มี encryption
resource "aws_s3_bucket" "good" {
  bucket = "my-bucket"
}

resource "aws_s3_bucket_server_side_encryption_configuration" "good" {
  bucket = aws_s3_bucket.good.id

  rule {
    apply_server_side_encryption_by_default {
      sse_algorithm     = "aws:kms"
      kms_master_key_id = aws_kms_key.s3.arn
    }
  }
}
```

### AVD-AWS-0132: S3 Bucket Versioning

```hcl
# ❌ ผิด
resource "aws_s3_bucket" "bad" {
  bucket = "my-bucket"
}

# ✅ ถูก
resource "aws_s3_bucket" "good" {
  bucket = "my-bucket"
}

resource "aws_s3_bucket_versioning" "good" {
  bucket = aws_s3_bucket.good.id
  versioning_configuration {
    status = "Enabled"
  }
}
```

### AVD-AWS-0028: IAM Policies Too Permissive

```hcl
# ❌ ผิด - wildcard actions และ resources
resource "aws_iam_policy" "bad" {
  name = "overly-permissive"
  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect   = "Allow"
        Action   = "*"        # ❌ ไม่ควรใช้ wildcard
        Resource = "*"        # ❌ ไม่ควรใช้ wildcard
      }
    ]
  })
}

# ✅ ถูก - specific actions และ resources
resource "aws_iam_policy" "good" {
  name = "least-privilege"
  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Action = [
          "s3:GetObject",
          "s3:PutObject"
        ]
        Resource = "arn:aws:s3:::my-bucket/*"
      }
    ]
  })
}
```

### AVD-AWS-0057: Security Group Ingress 0.0.0.0/0

```hcl
# ❌ ผิด - open to world
resource "aws_security_group" "bad" {
  name = "bad-sg"

  ingress {
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]  # ❌ SSH from anywhere!
  }
}

# ✅ ถูก - restrict access
resource "aws_security_group" "good" {
  name = "good-sg"

  ingress {
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = ["10.0.0.0/8"]  # ✅ VPN/Private network only
    description = "SSH from internal network via VPN"
  }
}
```

### AVD-AWS-0104: RDS Not Encrypted

```hcl
# ❌ ผิด
resource "aws_db_instance" "bad" {
  identifier        = "mydb"
  engine            = "mysql"
  instance_class    = "db.t3.micro"
  username          = "admin"
  password          = "password"
  storage_encrypted = false  # ❌
}

# ✅ ถูก
resource "aws_db_instance" "good" {
  identifier        = "mydb"
  engine            = "mysql"
  instance_class    = "db.t3.micro"
  username          = "admin"
  password          = var.db_password
  
  storage_encrypted = true              # ✅
  kms_key_id        = aws_kms_key.rds.arn
  
  # Additional security
  multi_az               = true
  deletion_protection    = true
  backup_retention_period = 7
  
  enabled_cloudwatch_logs_exports = ["audit", "error", "general", "slowquery"]
}
```

### AVD-AWS-0077: ElastiCache Not Encrypted

```hcl
# ❌ ผิด
resource "aws_elasticache_replication_group" "bad" {
  replication_group_id          = "my-redis"
  replication_group_description = "Redis cluster"
  
  # ขาด at_rest_encryption_enabled และ transit_encryption_enabled
}

# ✅ ถูก
resource "aws_elasticache_replication_group" "good" {
  replication_group_id          = "my-redis"
  replication_group_description = "Redis cluster"
  
  at_rest_encryption_enabled = true   # ✅ Encrypt data at rest
  transit_encryption_enabled = true   # ✅ Encrypt data in transit
  auth_token                 = var.redis_auth_token
  kms_key_id                 = aws_kms_key.redis.arn
}
```

### AVD-AZU-* (Azure Rules)

```hcl
# AVD-AZU-0007: Azure Storage Account not encrypted with CMK
resource "azurerm_storage_account" "good" {
  name                     = "mystorageaccount"
  resource_group_name      = azurerm_resource_group.main.name
  location                 = azurerm_resource_group.main.location
  account_tier             = "Standard"
  account_replication_type = "GRS"

  customer_managed_key {
    key_vault_key_id          = azurerm_key_vault_key.storage.id
    user_assigned_identity_id = azurerm_user_assigned_identity.storage.id
  }
}
```

### AVD-GCP-* (Google Cloud Rules)

```hcl
# AVD-GCP-0062: GCS bucket not encrypted with CMK
resource "google_storage_bucket" "good" {
  name          = "my-bucket"
  location      = "US"
  force_destroy = false

  encryption {
    default_kms_key_name = google_kms_crypto_key.bucket_key.id
  }
}
```

---

## Step 908: Custom Checks ด้วย Rego

### โครงสร้าง Custom Check

```
.
├── main.tf
├── .tfsec/
│   ├── config.json
│   └── custom-checks/
│       ├── mandatory-tags.rego
│       └── approved-regions.rego
```

### ตัวอย่าง Custom Check: Mandatory Tags

```rego
# .tfsec/custom-checks/mandatory-tags.rego
package custom.terraform.aws.mandatory_tags

import future.keywords

deny[msg] {
    resource := input.aws_instance[_]
    required_tags := {"Owner", "Environment", "Project", "CostCenter"}
    existing_tags := {k | resource.tags[k]}
    missing := required_tags - existing_tags
    count(missing) > 0
    msg := sprintf(
        "EC2 instance '%v' is missing required tags: %v",
        [resource.id, missing]
    )
}
```

### ตัวอย่าง Custom Check: Approved Instance Types Only

```rego
# .tfsec/custom-checks/approved-instance-types.rego
package custom.terraform.aws.approved_instances

import future.keywords

approved_types := {
    "t3.micro",
    "t3.small", 
    "t3.medium",
    "t3.large",
    "m5.large",
    "m5.xlarge"
}

deny[msg] {
    instance := input.aws_instance[name]
    not approved_types[instance.instance_type]
    msg := sprintf(
        "EC2 instance '%v' uses unapproved instance type '%v'. Approved types: %v",
        [name, instance.instance_type, approved_types]
    )
}
```

### รัน Custom Checks

```bash
# รัน TFSec พร้อม custom checks
tfsec --custom-check-dir .tfsec/custom-checks .

# รัน Trivy พร้อม custom checks
trivy config --policy .tfsec/custom-checks .
```

### Custom Check Format ใหม่ (YAML)

```yaml
# .tfsec/custom-checks/check-mandatory-tags.yaml
---
checks:
  - code: CUS-001
    description: "Ensure mandatory tags are present on EC2 instances"
    impact: "Resources without proper tags cannot be tracked for cost/ownership"
    resolution: "Add Owner, Environment, Project, and CostCenter tags"
    requiredTypes:
      - resource
    requiredLabels:
      - aws_instance
    severity: HIGH
    errorMessage: "EC2 instance is missing required tags"
    relatedLinks:
      - "https://your-wiki.com/tagging-policy"
    matchSpec:
      name: tags
      action: contains
      value:
        Owner: ""
        Environment: ""
        Project: ""
        CostCenter: ""
    rule: |
      resource.tags.Owner != null &&
      resource.tags.Environment != null &&
      resource.tags.Project != null &&
      resource.tags.CostCenter != null
```

---

## Step 909: TFSec ใน GitHub Actions (Complete Workflow)

### .github/workflows/terraform-security.yml

```yaml
name: Terraform Security Scan

on:
  pull_request:
    branches: [main, develop]
    paths:
      - '**.tf'
      - '**.tfvars'
  push:
    branches: [main]
    paths:
      - '**.tf'

permissions:
  contents: read
  security-events: write    # สำหรับ upload SARIF
  pull-requests: write      # สำหรับ comment บน PR

jobs:
  # ===================================================
  # Job 1: TFSec Scan
  # ===================================================
  tfsec:
    name: TFSec Security Scan
    runs-on: ubuntu-latest
    
    steps:
      - name: Checkout code
        uses: actions/checkout@v4
        with:
          fetch-depth: 0  # ต้องการ full history สำหรับ diff
      
      # ===== วิธีที่ 1: ใช้ Official TFSec Action =====
      - name: Run TFSec
        uses: aquasecurity/tfsec-action@v1.0.0
        with:
          working_directory: '.'
          format: 'sarif'
          output: 'tfsec-results.sarif'
          soft_fail: 'true'  # ไม่ fail pipeline (upload SARIF ก่อน)
          additional_args: '--minimum-severity HIGH'
      
      # ===== Upload SARIF to GitHub Security Tab =====
      - name: Upload SARIF results
        uses: github/codeql-action/upload-sarif@v3
        if: always()
        with:
          sarif_file: 'tfsec-results.sarif'
          category: 'tfsec'
      
      # ===== แสดงผลใน PR Comment =====
      - name: Run TFSec for PR Comment
        if: github.event_name == 'pull_request'
        id: tfsec-pr
        run: |
          # ติดตั้ง TFSec
          curl -s https://raw.githubusercontent.com/aquasecurity/tfsec/master/scripts/install_linux.sh | bash
          
          # รัน scan
          tfsec . \
            --format json \
            --minimum-severity MEDIUM \
            --out tfsec-output.json || true
          
          # สร้าง summary
          CRITICAL=$(jq '[.results[] | select(.severity == "CRITICAL")] | length' tfsec-output.json)
          HIGH=$(jq '[.results[] | select(.severity == "HIGH")] | length' tfsec-output.json)
          MEDIUM=$(jq '[.results[] | select(.severity == "MEDIUM")] | length' tfsec-output.json)
          LOW=$(jq '[.results[] | select(.severity == "LOW")] | length' tfsec-output.json)
          
          echo "critical=$CRITICAL" >> $GITHUB_OUTPUT
          echo "high=$HIGH" >> $GITHUB_OUTPUT
          echo "medium=$MEDIUM" >> $GITHUB_OUTPUT
          echo "low=$LOW" >> $GITHUB_OUTPUT
      
      - name: Comment TFSec Results on PR
        if: github.event_name == 'pull_request'
        uses: actions/github-script@v7
        with:
          script: |
            const critical = '${{ steps.tfsec-pr.outputs.critical }}';
            const high = '${{ steps.tfsec-pr.outputs.high }}';
            const medium = '${{ steps.tfsec-pr.outputs.medium }}';
            const low = '${{ steps.tfsec-pr.outputs.low }}';
            
            const emoji = parseInt(critical) > 0 ? '🚨' : 
                         parseInt(high) > 0 ? '⚠️' : '✅';
            
            const body = `## ${emoji} TFSec Security Scan Results
            
            | Severity | Count |
            |----------|-------|
            | 🔴 CRITICAL | ${critical} |
            | 🟠 HIGH | ${high} |
            | 🟡 MEDIUM | ${medium} |
            | 🔵 LOW | ${low} |
            
            ${parseInt(critical) > 0 ? '**⛔ This PR has CRITICAL security findings that must be fixed before merging.**' : ''}
            ${parseInt(high) > 0 && parseInt(critical) === 0 ? '**⚠️ This PR has HIGH severity findings. Please review before merging.**' : ''}
            ${parseInt(critical) === 0 && parseInt(high) === 0 ? '**✅ No critical or high severity findings detected.**' : ''}
            
            > View detailed findings in the [Security tab](https://github.com/${{ github.repository }}/security/code-scanning)
            `;
            
            github.rest.issues.createComment({
              issue_number: context.issue.number,
              owner: context.repo.owner,
              repo: context.repo.repo,
              body: body
            });
      
      # ===== Fail ถ้ามี CRITICAL findings =====
      - name: Check for Critical Findings
        run: |
          if [ -f "tfsec-output.json" ]; then
            CRITICAL=$(jq '[.results[] | select(.severity == "CRITICAL")] | length' tfsec-output.json)
            if [ "$CRITICAL" -gt "0" ]; then
              echo "::error::Found $CRITICAL CRITICAL security findings. Please fix them before merging."
              exit 1
            fi
          fi
  
  # ===================================================
  # Job 2: Trivy Config Scan (Modern)
  # ===================================================
  trivy:
    name: Trivy Config Scan
    runs-on: ubuntu-latest
    
    steps:
      - name: Checkout code
        uses: actions/checkout@v4
      
      - name: Run Trivy vulnerability scanner
        uses: aquasecurity/trivy-action@master
        with:
          scan-type: 'config'
          scan-ref: '.'
          format: 'sarif'
          output: 'trivy-results.sarif'
          severity: 'CRITICAL,HIGH'
          exit-code: '0'  # Don't fail here, will fail after upload
      
      - name: Upload Trivy SARIF to GitHub
        uses: github/codeql-action/upload-sarif@v3
        if: always()
        with:
          sarif_file: 'trivy-results.sarif'
          category: 'trivy-config'
      
      - name: Check Trivy Results
        run: |
          # รัน อีกครั้งเพื่อ check exit code
          docker run --rm \
            -v $(pwd):/workspace \
            -w /workspace \
            aquasec/trivy:latest config \
            --severity CRITICAL \
            --exit-code 1 \
            . || (echo "CRITICAL findings found!" && exit 1)
```

### TFSec ใน GitLab CI

```yaml
# .gitlab-ci.yml
stages:
  - security

tfsec:
  stage: security
  image: aquasec/tfsec:latest
  script:
    - tfsec . --format sarif --out gl-sast-report.json
    - tfsec . --minimum-severity HIGH --format text
  artifacts:
    reports:
      sast: gl-sast-report.json
    paths:
      - gl-sast-report.json
    expire_in: 1 week
  rules:
    - if: '$CI_PIPELINE_SOURCE == "merge_request_event"'
    - if: '$CI_COMMIT_BRANCH == "main"'
```

---

## Step 910: TFSec กับ Pre-commit Hooks

### ติดตั้ง Pre-commit

```bash
# ติดตั้ง pre-commit
pip install pre-commit
# หรือ
brew install pre-commit
```

### .pre-commit-config.yaml

```yaml
repos:
  # TFSec pre-commit hook
  - repo: https://github.com/aquasecurity/tfsec
    rev: v1.28.5
    hooks:
      - id: tfsec
        args:
          - --minimum-severity=HIGH
          - --format=lovely
        # ทำงานเฉพาะเมื่อ .tf files เปลี่ยน
        files: \.tf$

  # Trivy config scan
  - repo: https://github.com/antonbabenko/pre-commit-terraform
    rev: v1.88.0
    hooks:
      - id: terraform_trivy
        args:
          - --args=--severity=HIGH,CRITICAL
          - --args=--exit-code=1
```

### รัน Pre-commit

```bash
# ติดตั้ง hooks
pre-commit install

# รันทุก hooks บน all files
pre-commit run --all-files

# รัน tfsec เฉพาะ
pre-commit run tfsec --all-files

# รัน บน specific files
pre-commit run tfsec --files main.tf
```

### Makefile สำหรับ Developer Workflow

```makefile
# Makefile
.PHONY: security lint validate

# Security scan
security:
	@echo "Running TFSec..."
	tfsec --minimum-severity HIGH .
	@echo "Running Trivy..."
	trivy config --severity HIGH,CRITICAL .
	@echo "Security scan complete!"

# Lint
lint:
	terraform fmt -check -recursive .
	tflint --recursive

# Validate
validate:
	terraform init -backend=false
	terraform validate

# Full check
check: lint validate security
	@echo "All checks passed!"

# Generate security report
security-report:
	tfsec --format json . > reports/tfsec-report.json
	trivy config --format json --output reports/trivy-report.json .
	@echo "Reports generated in ./reports/"
```

---

## สรุปเปรียบเทียบ TFSec vs Checkov

| Feature | TFSec/Trivy | Checkov |
|---------|------------|---------|
| ภาษาที่ใช้เขียน | Go | Python |
| Custom rules | Rego | Python/YAML |
| Output formats | หลายแบบ | หลายแบบ |
| Speed | เร็วมาก | เร็ว |
| AWS rules | ~200+ | ~400+ |
| Multi-cloud | AWS, Azure, GCP | AWS, Azure, GCP, K8s |
| Terraform versions | 0.12+ | 0.12+ |
| SARIF support | ✅ | ✅ |
| GitHub integration | ✅ | ✅ |
| Price | Free | Free (OSS) |

---

## แบบฝึกหัด

1. ติดตั้ง TFSec/Trivy บนเครื่อง local
2. สร้าง Terraform ที่มีความเสี่ยงด้านความปลอดภัย แล้วลองสแกนดู
3. สร้าง Custom Check สำหรับบังคับ mandatory tags
4. ตั้งค่า pre-commit hook ให้รัน TFSec อัตโนมัติ
5. ตั้งค่า GitHub Actions workflow ที่ upload SARIF ขึ้น GitHub Security tab

---

*จบ Part 91: TFSec Static Analysis*
