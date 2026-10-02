# Part 081: Introduction to IaC Security
## ขั้นตอนที่ 801-810: รู้จักกับความปลอดภัยของ Infrastructure as Code

---

## ขั้นตอนที่ 801: ทำไม IaC Security ถึงสำคัญ (Why IaC Security Matters)

### ภาพรวม (Overview)

Infrastructure as Code (IaC) ได้เปลี่ยนวิธีที่องค์กรจัดการโครงสร้างพื้นฐานคลาวด์อย่างสิ้นเชิง แต่พร้อมกับความสะดวกและความเร็ว ก็มาพร้อมกับความเสี่ยงด้านความปลอดภัยที่เพิ่มขึ้นเช่นกัน

**สถิติที่น่าตกใจ (Alarming Statistics):**
- 70% ของ cloud security incidents เกิดจาก misconfiguration (Gartner, 2024)
- ช่องโหว่ IaC โดยเฉลี่ยใช้เวลา 212 วันกว่าจะถูกตรวจพบ (IBM Security)
- Organizations ที่ใช้ IaC มีโอกาสเกิด misconfiguration สูงกว่า manual configuration 3 เท่า
- Data breach จาก S3 misconfiguration สร้างความเสียหายเฉลี่ย $3.86M ต่อเหตุการณ์

### ทำไม IaC จึงสร้างความเสี่ยงเพิ่มขึ้น (Why IaC Increases Risk)

```
Traditional Manual Infrastructure:
- Configuration เปลี่ยนแปลงช้า
- Human review ทุกขั้นตอน
- Limited blast radius
- Harder to replicate mistakes

IaC Infrastructure:
- Configuration เปลี่ยนแปลงเร็วมาก
- Automated deployments
- Code replication = mistake replication
- One bad module → ทุก environment ได้รับผลกระทบ
```

### Real-World Incidents

**Capital One Breach (2019)**
- Attacker ใช้ SSRF vulnerability เข้าถึง IAM role
- IAM role มี permissions กว้างเกินไป
- ข้อมูลลูกค้า 100+ ล้านคนถูกเข้าถึง
- ค่าใช้จ่าย: $80+ million

**Twitch Source Code Leak (2021)**
- S3 bucket misconfiguration
- Internal source code, revenue data ถูก expose
- Root cause: Missing bucket access controls ใน IaC

**Toyota Supplier Portal (2023)**
- Environment variable ที่มี credentials ถูก push ขึ้น GitHub
- Terraform state file ที่มี sensitive data ถูก expose
- ข้อมูลของ Toyota suppliers กว่า 300,000 ราย

### ความสำคัญในบริบทของ Terraform (Importance in Terraform Context)

```hcl
# ❌ ตัวอย่างที่อันตราย - credentials hardcoded
provider "aws" {
  access_key = "AKIAIOSFODNN7EXAMPLE"
  secret_key = "wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY"
  region     = "us-east-1"
}

# ❌ ตัวอย่างที่อันตราย - S3 bucket public
resource "aws_s3_bucket" "data" {
  bucket = "company-sensitive-data"
  acl    = "public-read"  # ❌ ทุกคนอ่านได้
}

# ✅ ตัวอย่างที่ถูกต้อง
provider "aws" {
  region = "us-east-1"
  # ใช้ environment variables หรือ IAM roles
}

resource "aws_s3_bucket" "data" {
  bucket = "company-sensitive-data"
}

resource "aws_s3_bucket_public_access_block" "data" {
  bucket = aws_s3_bucket.data.id
  
  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}
```

---

## ขั้นตอนที่ 802: หลักการ "Shift Left" Security

### คืออะไร? (What is Shift Left?)

"Shift Left" หมายถึงการย้ายกิจกรรมด้านความปลอดภัยไปอยู่ในระยะเริ่มต้นของวงจรการพัฒนา (SDLC) แทนที่จะตรวจสอบหลังจาก deploy แล้ว

```
Traditional Security Model (Shift Right):
Developer → Code → Build → Test → Deploy → SECURITY CHECK

Shift Left Security Model:
SECURITY CHECK → Developer → Code → Build → Test → Deploy
        ↑                ↑         ↑       ↑
   IDE Scanning    Pre-commit   CI/CD   Runtime
```

### ต้นทุนของการค้นพบช่องโหว่ในแต่ละช่วง (Cost of Finding Vulnerabilities)

```
Development Phase:    $1     (แก้ได้ทันที)
Pre-commit:           $10    (แก้ได้เร็ว)
CI/CD Pipeline:       $100   (ต้องรื้อ pipeline)
Staging:              $1,000  (ต้องทดสอบใหม่)
Production:           $10,000+ (อาจมี breach)
```

### Shift Left กับ Terraform (Shift Left with Terraform)

```
Phase 1: IDE/Editor
├── Terraform VS Code Extension
├── HashiCorp Sentinel (policy as code)
└── IntelliJ Terraform Plugin

Phase 2: Pre-commit hooks
├── checkov
├── tfsec
├── terraform validate
└── terraform fmt

Phase 3: CI/CD Pipeline
├── Checkov GitHub Actions
├── TFSec in GitLab CI
├── Snyk IaC
└── Terrascan

Phase 4: Deployment Gate
├── OPA (Open Policy Agent)
├── HashiCorp Sentinel
└── AWS Config Rules

Phase 5: Runtime Monitoring
├── AWS Security Hub
├── Amazon GuardDuty
└── Prisma Cloud
```

### การตั้งค่า Pre-commit Hooks

```yaml
# .pre-commit-config.yaml
repos:
  - repo: https://github.com/antonbabenko/pre-commit-terraform
    rev: v1.83.5
    hooks:
      - id: terraform_fmt
      - id: terraform_validate
      - id: terraform_docs
      - id: checkov
        args:
          - --args=--skip-check CKV_AWS_136,CKV2_AWS_6
      - id: terraform_tfsec
```

---

## ขั้นตอนที่ 803: IaC Attack Surface

### พื้นที่การโจมตี IaC (IaC Attack Surface)

IaC มีพื้นที่การโจมตีที่หลากหลายซึ่งต้องพิจารณา:

```
IaC Attack Surface Map:
┌─────────────────────────────────────────────────────────────┐
│                     IaC Security Attack Surface              │
├─────────────────┬───────────────────────────────────────────┤
│   Source Code   │  - Hardcoded credentials                  │
│   Repository    │  - Secrets in .tfvars                     │
│                 │  - Sensitive output values                 │
│                 │  - Infrastructure vulnerabilities in code  │
├─────────────────┼───────────────────────────────────────────┤
│  Terraform      │  - Sensitive data in plain text            │
│  State Files    │  - Resource IDs, IPs, secrets             │
│                 │  - Unencrypted remote state               │
│                 │  - State file access controls             │
├─────────────────┼───────────────────────────────────────────┤
│  Pipeline &     │  - Exposed CI/CD environment variables    │
│  CI/CD          │  - Webhook secrets                        │
│                 │  - Build artifacts with secrets           │
│                 │  - Pipeline injection attacks             │
├─────────────────┼───────────────────────────────────────────┤
│  Provider       │  - Over-permissive provider credentials   │
│  Credentials    │  - Long-lived access keys                 │
│                 │  - Shared credentials                     │
├─────────────────┼───────────────────────────────────────────┤
│  Modules &      │  - Malicious module sources               │
│  Dependencies   │  - Unpinned module versions               │
│                 │  - Supply chain attacks                   │
│                 │  - Third-party provider risks             │
├─────────────────┼───────────────────────────────────────────┤
│  Deployed       │  - Misconfigured resources                │
│  Resources      │  - Over-permissive IAM                    │
│                 │  - Public exposure                        │
│                 │  - Missing encryption                     │
└─────────────────┴───────────────────────────────────────────┘
```

### Terraform State File Security

```hcl
# ❌ Local state - อันตรายมาก
# terraform.tfstate มี sensitive data ทั้งหมด

# ✅ Remote state with encryption
terraform {
  backend "s3" {
    bucket         = "terraform-state-bucket"
    key            = "prod/terraform.tfstate"
    region         = "us-east-1"
    encrypt        = true              # ✅ Encrypt at rest
    kms_key_id     = "arn:aws:kms:..."  # ✅ Customer managed key
    dynamodb_table = "terraform-locks" # ✅ State locking
    
    # ✅ Access control
    acl = "bucket-owner-full-control"
  }
}

# ✅ S3 bucket for state with strict controls
resource "aws_s3_bucket" "terraform_state" {
  bucket = "company-terraform-state"
  
  lifecycle {
    prevent_destroy = true  # ✅ Never accidentally delete
  }
}

resource "aws_s3_bucket_versioning" "terraform_state" {
  bucket = aws_s3_bucket.terraform_state.id
  versioning_configuration {
    status = "Enabled"  # ✅ Version history
  }
}

resource "aws_s3_bucket_public_access_block" "terraform_state" {
  bucket = aws_s3_bucket.terraform_state.id
  
  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}
```

### Module Supply Chain Security

```hcl
# ❌ Unpinned module version - อันตราย
module "vpc" {
  source = "terraform-aws-modules/vpc/aws"
  # ไม่มี version = อาจได้ version ที่มีช่องโหว่
}

# ✅ Pinned version
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "5.1.2"  # ✅ Fixed version
}

# ✅ ดียิ่งขึ้น - ใช้ git ref
module "vpc" {
  source = "git::https://github.com/terraform-aws-modules/terraform-aws-vpc.git?ref=v5.1.2"
}

# ✅ Internal module registry (ปลอดภัยที่สุด)
module "vpc" {
  source  = "registry.terraform.io/company/vpc/aws"
  version = "~> 2.0"
}
```

---

## ขั้นตอนที่ 804: ปัญหาความปลอดภัย IaC ที่พบบ่อย

### 1. Hardcoded Credentials (ข้อมูลประจำตัวฝังในโค้ด)

**ความเสี่ยง:** ข้อมูลลับถูก expose ใน git history หรือ logs

```hcl
# ❌ VULNERABLE - อย่าทำแบบนี้เด็ดขาด
resource "aws_db_instance" "database" {
  identifier     = "my-database"
  engine         = "mysql"
  instance_class = "db.t3.micro"
  
  username = "admin"
  password = "SuperSecret123!"  # ❌ Hardcoded password
  
  db_name = "myapp"
}

# ✅ SECURE - ใช้ Secrets Manager
data "aws_secretsmanager_secret_version" "db_password" {
  secret_id = "prod/myapp/db-password"
}

resource "aws_db_instance" "database" {
  identifier     = "my-database"
  engine         = "mysql"
  instance_class = "db.t3.micro"
  
  username = "admin"
  password = jsondecode(data.aws_secretsmanager_secret_version.db_password.secret_string)["password"]
  
  db_name = "myapp"
}
```

### 2. Over-permissive IAM (IAM ที่มี permissions กว้างเกินไป)

```hcl
# ❌ VULNERABLE - AdministratorAccess
resource "aws_iam_role_policy" "app_policy" {
  name = "app-policy"
  role = aws_iam_role.app_role.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect   = "Allow"
        Action   = "*"          # ❌ ทุก action
        Resource = "*"          # ❌ ทุก resource
      }
    ]
  })
}

# ✅ SECURE - Least Privilege
resource "aws_iam_role_policy" "app_policy" {
  name = "app-policy"
  role = aws_iam_role.app_role.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Action = [
          "s3:GetObject",      # ✅ เฉพาะที่จำเป็น
          "s3:PutObject"
        ]
        Resource = [
          "${aws_s3_bucket.app_data.arn}/*"  # ✅ เฉพาะ bucket ที่ต้องการ
        ]
      }
    ]
  })
}
```

### 3. Unencrypted Storage (storage ที่ไม่เข้ารหัส)

```hcl
# ❌ VULNERABLE - No encryption
resource "aws_s3_bucket" "data" {
  bucket = "sensitive-data"
  # ไม่มี encryption configuration
}

# ✅ SECURE - Encrypted with KMS
resource "aws_s3_bucket" "data" {
  bucket = "sensitive-data"
}

resource "aws_s3_bucket_server_side_encryption_configuration" "data" {
  bucket = aws_s3_bucket.data.id

  rule {
    apply_server_side_encryption_by_default {
      sse_algorithm     = "aws:kms"
      kms_master_key_id = aws_kms_key.s3_key.arn
    }
    bucket_key_enabled = true
  }
}
```

### 4. Public Network Exposure (การ expose ทาง network)

```hcl
# ❌ VULNERABLE - SSH open to internet
resource "aws_security_group_rule" "ssh" {
  type        = "ingress"
  from_port   = 22
  to_port     = 22
  protocol    = "tcp"
  cidr_blocks = ["0.0.0.0/0"]  # ❌ internet ทั้งหมด
  
  security_group_id = aws_security_group.web.id
}

# ✅ SECURE - SSH only from bastion/VPN
resource "aws_security_group_rule" "ssh" {
  type      = "ingress"
  from_port = 22
  to_port   = 22
  protocol  = "tcp"
  
  source_security_group_id = aws_security_group.bastion.id  # ✅ จาก bastion เท่านั้น
  
  security_group_id = aws_security_group.web.id
}
```

### 5. Missing Logging (ไม่มีการบันทึก logs)

```hcl
# ❌ VULNERABLE - No CloudTrail
# (ไม่มี CloudTrail configuration เลย)

# ✅ SECURE - CloudTrail enabled
resource "aws_cloudtrail" "main" {
  name                          = "main-trail"
  s3_bucket_name               = aws_s3_bucket.cloudtrail_logs.id
  include_global_service_events = true      # ✅ Global events
  is_multi_region_trail         = true      # ✅ All regions
  enable_log_file_validation    = true      # ✅ Log validation
  
  cloud_watch_logs_group_arn = "${aws_cloudwatch_log_group.cloudtrail.arn}:*"
  cloud_watch_logs_role_arn  = aws_iam_role.cloudtrail.arn
  
  event_selector {
    read_write_type           = "All"
    include_management_events = true
    
    data_resource {
      type   = "AWS::S3::Object"
      values = ["arn:aws:s3:::"]  # ✅ S3 data events
    }
  }
}
```

### 6. Weak Authentication (การยืนยันตัวตนที่อ่อนแอ)

```hcl
# ❌ VULNERABLE - No MFA for IAM users
resource "aws_iam_user" "developer" {
  name = "developer-alice"
}

resource "aws_iam_user_login_profile" "developer" {
  user                    = aws_iam_user.developer.name
  password_reset_required = false  # ❌ ไม่ต้องเปลี่ยน password
}

# ✅ SECURE - MFA Required
resource "aws_iam_policy" "require_mfa" {
  name        = "RequireMFA"
  description = "Policy that requires MFA for all actions"

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Sid    = "DenyWithoutMFA"
        Effect = "Deny"
        NotAction = [
          "iam:CreateVirtualMFADevice",
          "iam:EnableMFADevice",
          "iam:ListMFADevices",
          "sts:GetSessionToken"
        ]
        Resource = "*"
        Condition = {
          BoolIfExists = {
            "aws:MultiFactorAuthPresent" = "false"
          }
        }
      }
    ]
  })
}
```

### 7. Insecure Defaults (ค่าเริ่มต้นที่ไม่ปลอดภัย)

```hcl
# ❌ VULNERABLE - Default security group
# Default VPC security group allows all inbound/outbound traffic

# ✅ SECURE - Restrictive default security group
resource "aws_default_security_group" "default" {
  vpc_id = aws_vpc.main.id

  # ✅ ไม่มี ingress rules = deny all inbound
  # ✅ ไม่มี egress rules = deny all outbound
  
  tags = {
    Name = "default-restricted"
  }
}
```

### 8. Missing Resource Constraints (ขาด resource constraints)

```hcl
# ❌ VULNERABLE - No limits on resource creation
resource "aws_iam_policy" "lambda_policy" {
  policy = jsonencode({
    Statement = [{
      Effect   = "Allow"
      Action   = "ec2:RunInstances"
      Resource = "*"
      # ❌ ไม่มี condition - สามารถสร้าง EC2 instance ขนาดใดก็ได้
    }]
  })
}

# ✅ SECURE - With constraints
resource "aws_iam_policy" "lambda_policy" {
  policy = jsonencode({
    Statement = [{
      Effect   = "Allow"
      Action   = "ec2:RunInstances"
      Resource = "*"
      Condition = {
        StringEquals = {
          "ec2:InstanceType" = ["t3.micro", "t3.small"]  # ✅ จำกัด instance type
        }
        StringLikeIfExists = {
          "ec2:Region" = "us-east-1"  # ✅ จำกัด region
        }
      }
    }]
  })
}
```

---

## ขั้นตอนที่ 805: IaC Security Threat Model

### STRIDE Framework สำหรับ IaC

```
STRIDE Analysis for IaC:

S - Spoofing (การปลอมแปลงตัวตน)
    - Fake AWS credentials
    - Compromised service accounts
    - Module source spoofing

T - Tampering (การแก้ไขข้อมูล)
    - State file manipulation
    - Module version tampering
    - Policy document injection

R - Repudiation (การปฏิเสธการกระทำ)
    - No audit trail for IaC changes
    - Missing CloudTrail
    - No IaC change logging

I - Information Disclosure (การเปิดเผยข้อมูล)
    - Secrets in state files
    - Sensitive outputs
    - Public state backends

D - Denial of Service (การปฏิเสธบริการ)
    - Resource quotas exhaustion
    - Cost attacks
    - Accidental resource deletion

E - Elevation of Privilege (การยกระดับสิทธิ์)
    - IAM privilege escalation
    - Role chaining attacks
    - PassRole abuse
```

### Data Flow Diagram สำหรับ IaC Security

```
Developer Workstation
       │
       ▼
[IDE with Security Plugin] ──→ Pre-commit Hooks
       │                              │
       ▼                              ▼
  Git Repository ──────────→ [tfsec, checkov]
       │
       ▼
   CI/CD Pipeline
  ┌────┴─────┐
  ▼          ▼
[Security  [Terraform
  Scan]      Plan]
  │          │
  └────┬─────┘
       ▼
  Gate Review
  (fail on HIGH)
       │
       ▼
  Terraform Apply
       │
       ▼
  AWS Resources
       │
       ▼
  Runtime Monitoring
  (GuardDuty, Config)
```

---

## ขั้นตอนที่ 806: OWASP IaC Security Top 10

### OWASP Top 10 IaC Security Risks

**IaC-01: Overly Permissive Identity and Access Management (IAM)**

```hcl
# ❌ VULNERABLE - IaC-01
resource "aws_iam_role" "lambda_role" {
  assume_role_policy = jsonencode({
    Statement = [{
      Principal = { Service = "lambda.amazonaws.com" }
      Action    = "sts:AssumeRole"
    }]
  })
}

resource "aws_iam_role_policy_attachment" "lambda_admin" {
  role       = aws_iam_role.lambda_role.name
  policy_arn = "arn:aws:iam::aws:policy/AdministratorAccess"  # ❌
}

# ✅ SECURE - Least privilege
resource "aws_iam_role_policy" "lambda_policy" {
  role = aws_iam_role.lambda_role.id
  policy = jsonencode({
    Statement = [{
      Effect   = "Allow"
      Action   = ["s3:GetObject", "s3:PutObject"]
      Resource = "${aws_s3_bucket.app.arn}/*"
    }]
  })
}
```

**IaC-02: Exposed Secrets and Credentials**

```hcl
# ❌ VULNERABLE - IaC-02
variable "db_password" {
  default = "MyPassword123!"  # ❌ Default password
}

resource "aws_ssm_parameter" "db_password" {
  name  = "/prod/db/password"
  type  = "String"            # ❌ Plain text
  value = var.db_password
}

# ✅ SECURE
variable "db_password" {
  description = "Database password"
  type        = string
  sensitive   = true  # ✅ Mark as sensitive
  # No default - must be provided
}

resource "aws_ssm_parameter" "db_password" {
  name   = "/prod/db/password"
  type   = "SecureString"    # ✅ Encrypted
  value  = var.db_password
  key_id = aws_kms_key.ssm_key.arn
}
```

**IaC-03: Lack of Immutable Infrastructure**

```hcl
# ❌ VULNERABLE - Mutable infrastructure
resource "aws_instance" "web" {
  ami           = "ami-12345678"
  instance_type = "t3.micro"
  
  provisioner "remote-exec" {
    inline = [
      "sudo apt-get update",
      "sudo apt-get install -y nginx"
    ]
    # ❌ Mutable - state drifts over time
  }
}

# ✅ SECURE - Immutable via AMI
data "aws_ami" "web_server" {
  most_recent = true
  owners      = ["self"]  # ✅ Own AMI
  
  filter {
    name   = "name"
    values = ["web-server-nginx-*"]
  }
}

resource "aws_instance" "web" {
  ami           = data.aws_ami.web_server.id  # ✅ Pre-built AMI
  instance_type = "t3.micro"
  # ✅ No provisioners - immutable
}
```

**IaC-04: Insecure Defaults**
**IaC-05: Missing Audit Logging**
**IaC-06: Unencrypted Data**
**IaC-07: Network Misconfigurations**
**IaC-08: Supply Chain Vulnerabilities**
**IaC-09: Lack of Change Management**
**IaC-10: Weak Authentication and Authorization**

---

## ขั้นตอนที่ 807: Security Scanning Tools Landscape

### เปรียบเทียบ Tools (Tools Comparison Matrix)

| Feature | Checkov | TFSec | Terrascan | Snyk IaC | KICS | Trivy |
|---------|---------|-------|-----------|----------|------|-------|
| **License** | Apache 2.0 | MIT | Apache 2.0 | Commercial | Apache 2.0 | Apache 2.0 |
| **Terraform Support** | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| **CloudFormation** | ✅ | ❌ | ✅ | ✅ | ✅ | ✅ |
| **Kubernetes** | ✅ | ❌ | ✅ | ✅ | ✅ | ✅ |
| **Dockerfile** | ✅ | ❌ | ❌ | ✅ | ✅ | ✅ |
| **Custom Rules** | ✅ Python/YAML | ✅ Rego | ✅ Rego | ✅ | ✅ YAML | ✅ |
| **IDE Plugin** | ✅ VS Code | ✅ | ❌ | ✅ | ✅ VS Code | ❌ |
| **CI/CD** | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| **SARIF Output** | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| **Fix Suggestions** | ✅ | ✅ | ❌ | ✅ | ✅ | ✅ |
| **Speed** | Medium | Fast | Medium | Slow | Medium | Fast |
| **False Positives** | Low | Low | Medium | Low | Medium | Low |

### 1. Checkov

```bash
# ติดตั้ง (Installation)
pip install checkov

# scan directory
checkov -d /path/to/terraform

# scan single file
checkov -f main.tf

# ดู output แบบ JSON
checkov -d . -o json

# skip specific checks
checkov -d . --skip-check CKV_AWS_20,CKV_AWS_57

# ใช้เฉพาะ framework
checkov -d . --framework terraform

# output สำหรับ CI
checkov -d . -o junitxml > checkov-results.xml
```

### 2. TFSec

```bash
# ติดตั้ง
brew install tfsec
# หรือ
go install github.com/aquasecurity/tfsec/cmd/tfsec@latest

# scan
tfsec .

# พร้อม format
tfsec . --format json

# ข้าม check
tfsec . --exclude aws-s3-enable-bucket-encryption

# minimum severity
tfsec . --minimum-severity HIGH
```

```hcl
# TFSec inline skip
resource "aws_s3_bucket" "example" {
  #tfsec:ignore:aws-s3-enable-bucket-versioning
  bucket = "example-bucket"
}
```

### 3. Terrascan

```bash
# ติดตั้ง
brew install terrascan
# หรือ Docker
docker pull tenable/terrascan

# scan
terrascan scan -t aws -d /path/to/terraform

# scan with policy
terrascan scan -t aws -p /path/to/policies

# output
terrascan scan -t aws -o json
```

### 4. Snyk IaC

```bash
# ติดตั้ง
npm install -g snyk

# authenticate
snyk auth

# scan
snyk iac test /path/to/terraform

# scan with severity threshold
snyk iac test --severity-threshold=high

# ignore issues
snyk ignore --id=SNYK-CC-TF-1
```

### 5. KICS (Keeping Infrastructure as Code Secure)

```bash
# ติดตั้ง
docker pull checkmarx/kics

# scan
docker run -v /path/to/terraform:/path checkmarx/kics scan -p /path -o /path/results

# output format
kics scan -p . -o . --report-formats "html,json"
```

### 6. Trivy (IaC Scanning)

```bash
# ติดตั้ง
brew install trivy

# scan terraform
trivy config /path/to/terraform

# scan with format
trivy config --format json /path/to/terraform

# severity filter
trivy config --severity HIGH,CRITICAL /path/to/terraform
```

---

## ขั้นตอนที่ 808: Integration Points

### IDE Integration

```json
// VS Code settings.json สำหรับ Checkov
{
  "checkov.token": "your-bridgecrew-token",
  "checkov.skipChecks": ["CKV_AWS_136"],
  "checkov.frameworks": ["terraform"],
  "checkov.externalChecksDir": ["./custom-checks"]
}
```

```json
// VS Code settings.json สำหรับ TFLint
{
  "tflint.executablePath": "/usr/local/bin/tflint",
  "tflint.autoInstall": true
}
```

### Pre-commit Hooks

```yaml
# .pre-commit-config.yaml - สมบูรณ์
repos:
  # Terraform formatting
  - repo: https://github.com/antonbabenko/pre-commit-terraform
    rev: v1.83.5
    hooks:
      - id: terraform_fmt
        args:
          - --args=-recursive
      
      - id: terraform_validate
        args:
          - --args=-json
      
      - id: terraform_docs
        args:
          - --hook-config=--path-to-file=README.md
          - --hook-config=--add-to-existing-file=true
      
      # Checkov scan
      - id: checkov
        args:
          - --args=--config-file .checkov.yml
      
      # TFSec scan
      - id: terraform_tfsec
        args:
          - --args=--minimum-severity HIGH
  
  # Secrets detection
  - repo: https://github.com/Yelp/detect-secrets
    rev: v1.4.0
    hooks:
      - id: detect-secrets
        args: ['--baseline', '.secrets.baseline']
  
  # General security
  - repo: https://github.com/zricethezav/gitleaks
    rev: v8.18.0
    hooks:
      - id: gitleaks
```

### GitHub Actions CI/CD Integration

```yaml
# .github/workflows/terraform-security.yml
name: Terraform Security Scan

on:
  pull_request:
    branches: [main, develop]
  push:
    branches: [main]

jobs:
  security-scan:
    name: IaC Security Scanning
    runs-on: ubuntu-latest
    
    steps:
      - name: Checkout code
        uses: actions/checkout@v4
      
      # Checkov Scan
      - name: Checkov Scan
        uses: bridgecrewio/checkov-action@master
        with:
          directory: .
          framework: terraform
          output_format: cli,sarif
          output_file_path: console,results.sarif
          soft_fail: false
          skip_check: CKV_AWS_136
      
      # Upload SARIF to GitHub Security tab
      - name: Upload SARIF
        uses: github/codeql-action/upload-sarif@v3
        with:
          sarif_file: results.sarif
        if: always()
      
      # TFSec Scan
      - name: TFSec
        uses: aquasecurity/tfsec-action@v1.0.0
        with:
          soft_fail: true
      
      # Snyk IaC
      - name: Snyk IaC
        uses: snyk/actions/iac@master
        env:
          SNYK_TOKEN: ${{ secrets.SNYK_TOKEN }}
        with:
          args: --severity-threshold=high
  
  terraform-plan:
    name: Terraform Plan
    runs-on: ubuntu-latest
    needs: security-scan
    
    steps:
      - name: Checkout
        uses: actions/checkout@v4
      
      - name: Setup Terraform
        uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: 1.6.0
      
      - name: Terraform Init
        run: terraform init
        env:
          AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
          AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
      
      - name: Terraform Plan
        run: terraform plan -out=tfplan
      
      - name: Terraform Security Check
        run: checkov -f tfplan --framework terraform_plan
```

### GitLab CI Integration

```yaml
# .gitlab-ci.yml
stages:
  - security
  - validate
  - plan
  - apply

variables:
  TF_VERSION: "1.6.0"
  CHECKOV_VERSION: "3.0.0"

checkov-scan:
  stage: security
  image: bridgecrew/checkov:${CHECKOV_VERSION}
  script:
    - checkov -d . --framework terraform -o cli,junitxml --output-file-path console,checkov-results.xml
  artifacts:
    reports:
      junit: checkov-results.xml
    paths:
      - checkov-results.xml
    when: always
  rules:
    - if: '$CI_PIPELINE_SOURCE == "merge_request_event"'

tfsec-scan:
  stage: security
  image: aquasec/tfsec:latest
  script:
    - tfsec . --format junit --out tfsec-results.xml
  artifacts:
    reports:
      junit: tfsec-results.xml
    when: always
```

---

## ขั้นตอนที่ 809: Security vs Velocity Tension

### ความขัดแย้งระหว่าง Security กับ Velocity

```
Fast Development ←────────────────────→ High Security
       │                                      │
  Ship quickly                          Never deploy
  Low friction                          Many gates
  Developer friendly                    Complex process
  
ต้องหาจุดสมดุล (Balance Point)
```

### วิธีสร้างสมดุล (Balancing Security and Velocity)

**1. Security as Code - อย่าให้ security เป็น blocker**

```python
# Checkov custom policy - ยืดหยุ่น
from checkov.common.models.enums import CheckCategories
from checkov.terraform.checks.resource.base_resource_check import BaseResourceCheck

class S3EncryptionCheck(BaseResourceCheck):
    def __init__(self):
        name = "Ensure S3 bucket has encryption enabled"
        id = "CKV_CUSTOM_S3_1"
        supported_resources = ["aws_s3_bucket_server_side_encryption_configuration"]
        categories = [CheckCategories.ENCRYPTION]
        super().__init__(name=name, id=id, categories=categories, 
                        supported_resources=supported_resources)
    
    def scan_resource_conf(self, conf):
        # ยืดหยุ่น - อนุญาต AES256 หรือ KMS
        rules = conf.get("rule", [])
        if not rules:
            return CheckResult.FAILED
        
        for rule in rules:
            apply_default = rule.get("apply_server_side_encryption_by_default", [])
            if apply_default:
                sse_algo = apply_default[0].get("sse_algorithm", [""])[0]
                if sse_algo in ["AES256", "aws:kms"]:
                    return CheckResult.PASSED
        
        return CheckResult.FAILED
```

**2. Soft-fail สำหรับ non-critical**

```bash
# Hard fail เฉพาะ CRITICAL
checkov -d . --check-severity-level CRITICAL

# Soft fail สำหรับ HIGH (แจ้งเตือนแต่ไม่ block)
checkov -d . --check-severity-level HIGH --soft-fail

# Non-blocking สำหรับ MEDIUM/LOW
checkov -d . --check-severity-level MEDIUM --soft-fail-on-empty-scope
```

**3. Baseline - ยกเว้นของเก่า focus ของใหม่**

```bash
# สร้าง baseline จาก existing issues
checkov -d . -o json > checkov-baseline.json

# ตรวจสอบเฉพาะ issues ใหม่
checkov -d . --baseline checkov-baseline.json
```

**4. Developer-Friendly Security Feedback**

```yaml
# GitHub Actions ที่เป็น developer-friendly
- name: Security Scan with Annotations
  uses: bridgecrewio/checkov-action@master
  with:
    directory: .
    output_format: github_failed_only  # ✅ แสดงเฉพาะที่ fail
    output_file_path: console
    quiet: true  # ✅ ไม่ verbose เกินไป
```

---

## ขั้นตอนที่ 810: Security Posture Maturity Model

### Security Maturity Levels สำหรับ IaC

```
Level 0: Unaware (ไม่รู้ตัว)
├── ไม่มี security scanning
├── Credentials hardcoded
├── ไม่มี access controls
└── ไม่มี monitoring

Level 1: Initial (เริ่มต้น)
├── Manual security reviews
├── Basic secret management
├── Some encryption
└── Basic IAM controls

Level 2: Developing (กำลังพัฒนา)
├── Automated scanning (Checkov/TFSec)
├── Pre-commit hooks
├── Secrets in Vault/Secrets Manager
├── Basic compliance checks
└── Security team involvement

Level 3: Defined (กำหนดแล้ว)
├── Security gates in CI/CD
├── Policy as Code (OPA/Sentinel)
├── All secrets managed
├── Encryption everywhere
├── Compliance automation
└── Security training

Level 4: Managed (จัดการได้)
├── Security metrics tracked
├── Automated remediation
├── Runtime protection
├── Threat modeling
├── Security champions
└── Regular pen tests

Level 5: Optimizing (เหมาะสมที่สุด)
├── Proactive threat hunting
├── ML-based anomaly detection
├── Continuous compliance
├── DevSecOps culture
├── Security innovation
└── Industry leadership
```

### NIST CSF Applied to IaC

```
NIST Cybersecurity Framework:

IDENTIFY (ระบุ)
├── IaC Asset Inventory (Terraform state)
├── Risk Assessment (tfsec, checkov baseline)
├── Governance (tagging strategy)
└── Supply Chain Risk (module sources)

PROTECT (ป้องกัน)
├── Access Control (IAM policies)
├── Data Security (encryption)
├── Information Protection (VPC, SGs)
├── Maintenance (patching via IaC)
└── Protective Technology (WAF, GuardDuty)

DETECT (ตรวจจับ)
├── Anomalies (CloudTrail, GuardDuty)
├── Continuous Monitoring (Config)
├── Detection Processes (SIEM)
└── IaC drift detection

RESPOND (ตอบสนอง)
├── Response Planning (runbooks)
├── Communications (alerting)
├── Analysis (log analysis)
├── Mitigation (auto-remediation)
└── Improvements (post-mortems)

RECOVER (ฟื้นฟู)
├── Recovery Planning (DR IaC)
├── Improvements (lessons learned)
├── Communications (stakeholders)
└── IaC-based restoration
```

### Building Security Culture for IaC Teams

```
Security Champion Program:
1. แต่ละ team มี Security Champion 1 คน
2. Champions ได้รับ training เพิ่มเติม
3. Champions review code ของ team
4. Champions เป็น bridge ระหว่าง Dev และ Security

Security Training Path:
Week 1: IaC Security Basics
Week 2: Terraform Security Best Practices
Week 3: Cloud-specific Security (AWS/Azure/GCP)
Week 4: Hands-on Security Scanning
Week 5: Incident Response
Week 6: Security as Code

Metrics to Track:
- Mean Time to Detect (MTTD) security issues
- Mean Time to Remediate (MTTR)
- % of IaC code with security scan
- False positive rate
- Security debt (age of open findings)
- Coverage (% of resources with security controls)
```

---

## สรุป (Summary)

### ประเด็นสำคัญที่ต้องจำ (Key Takeaways)

1. **IaC Security ไม่ใช่ option - เป็น requirement**: Misconfiguration เป็นสาเหตุหลักของ cloud breach

2. **Shift Left ประหยัดเงิน**: Bug ที่พบในช่วง dev ถูกกว่าใน production 10,000 เท่า

3. **Automate ทุกอย่างที่ทำได้**: Manual reviews ไม่เพียงพอสำหรับ IaC scale

4. **Security as Code**: ใช้ Checkov, TFSec, OPA เป็น standard ของ pipeline

5. **Balance Security and Velocity**: Security ไม่ควรเป็น blocker แต่ต้องเป็น enabler

6. **Culture สำคัญกว่า Tools**: Developer ที่ตระหนักด้านความปลอดภัยสำคัญกว่า tool ที่ดีที่สุด

### เครื่องมือแนะนำ (Recommended Tools)

```
ขั้นต่ำที่ควรมี:
✅ Checkov - pre-commit + CI/CD
✅ TFSec - fast additional check
✅ detect-secrets - prevent credentials

ขั้นสูง:
✅ Snyk IaC - comprehensive scanning
✅ Prisma Cloud / Bridgecrew - enterprise platform
✅ HashiCorp Sentinel - policy as code
✅ OPA - custom policies
```

---

*Part 081 ครอบคลุม IaC Security foundations ทั้งหมด - ต่อไปใน Part 082 จะเจาะลึก S3 Misconfigurations*
