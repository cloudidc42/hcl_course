# Part 92: Terrascan Security Scanner (Steps 911-920)

## การใช้ Terrascan สำหรับตรวจสอบความปลอดภัย Infrastructure

---

## Step 911: Terrascan คืออะไร?

### ภาพรวม

**Terrascan** เป็น Static Code Analyzer สำหรับ Infrastructure as Code พัฒนาโดย **Tenable** (เดิมชื่อ Accurics) รองรับ:

- Terraform (AWS, Azure, GCP, GitHub, Kubernetes)
- Kubernetes (YAML manifests)
- Helm Charts
- Kustomize
- Docker
- AWS CloudFormation

### จุดเด่นของ Terrascan

```
Terrascan Features:
┌─────────────────────────────────────────────────────────────┐
│ ✅ 500+ Rego-based policies                                  │
│ ✅ Multi-platform: Terraform, K8s, Docker, Helm              │
│ ✅ Server mode (REST API สำหรับ enterprise integration)      │
│ ✅ Webhook mode สำหรับ GitOps                               │
│ ✅ Custom policies ด้วย Rego                                │
│ ✅ Output: human, json, yaml, xml, junit-xml, sarif          │
│ ✅ CI/CD integrations                                        │
│ ✅ Kubernetes Admission Controller                           │
└─────────────────────────────────────────────────────────────┘
```

### เปรียบเทียบกับ Tools อื่น

```
┌────────────────┬──────────────┬──────────────┬──────────────┐
│ Feature        │ Terrascan    │ TFSec        │ Checkov      │
├────────────────┼──────────────┼──────────────┼──────────────┤
│ Company        │ Tenable      │ Aqua Security│ Prisma Cloud │
│ License        │ Apache 2.0   │ MIT          │ Apache 2.0   │
│ Policies       │ 500+         │ 200+         │ 1000+        │
│ Language       │ Go           │ Go           │ Python       │
│ Custom rules   │ Rego         │ Rego/YAML    │ Python/YAML  │
│ Server mode    │ ✅           │ ❌           │ ❌           │
│ K8s Admission  │ ✅           │ ❌           │ ❌           │
│ Multi-IaC      │ ✅ ดีมาก    │ ✅           │ ✅ ดีมาก    │
└────────────────┴──────────────┴──────────────┴──────────────┘
```

---

## Step 912: การติดตั้ง Terrascan

### วิธีที่ 1: Binary Download (แนะนำ)

```bash
# Linux AMD64
TERRASCAN_VERSION="1.19.1"
curl -L "https://github.com/tenable/terrascan/releases/download/v${TERRASCAN_VERSION}/terrascan_${TERRASCAN_VERSION}_Linux_x86_64.tar.gz" \
  -o terrascan.tar.gz
tar -xzf terrascan.tar.gz terrascan
sudo install terrascan /usr/local/bin/
rm terrascan terrascan.tar.gz

# macOS (Intel)
curl -L "https://github.com/tenable/terrascan/releases/download/v${TERRASCAN_VERSION}/terrascan_${TERRASCAN_VERSION}_Darwin_x86_64.tar.gz" \
  -o terrascan.tar.gz
tar -xzf terrascan.tar.gz terrascan
sudo install terrascan /usr/local/bin/

# macOS (Apple Silicon)
curl -L "https://github.com/tenable/terrascan/releases/download/v${TERRASCAN_VERSION}/terrascan_${TERRASCAN_VERSION}_Darwin_arm64.tar.gz" \
  -o terrascan.tar.gz
tar -xzf terrascan.tar.gz terrascan
sudo install terrascan /usr/local/bin/

# Windows PowerShell
$Version = "1.19.1"
Invoke-WebRequest -Uri "https://github.com/tenable/terrascan/releases/download/v${Version}/terrascan_${Version}_Windows_x86_64.zip" -OutFile "terrascan.zip"
Expand-Archive -Path "terrascan.zip" -DestinationPath "."
Move-Item -Path "terrascan.exe" -Destination "C:\Windows\System32\"
```

### วิธีที่ 2: Homebrew

```bash
brew install terrascan

# ตรวจสอบ
terrascan version
```

### วิธีที่ 3: Docker

```bash
# Pull image
docker pull tenable/terrascan:latest

# รัน scan
docker run --rm \
  -v $(pwd):/workspace \
  tenable/terrascan:latest \
  scan -t aws -d /workspace

# สร้าง alias สะดวกขึ้น
alias terrascan='docker run --rm -v $(pwd):/workspace tenable/terrascan:latest'
terrascan scan -t aws -d /workspace
```

### วิธีที่ 4: Go Install

```bash
go install github.com/tenable/terrascan@latest
```

### ตรวจสอบ Version

```bash
terrascan version
# Output:
# Terrascan
# Version:   1.19.1
# Git Commit: abc123
# Build Time: 2024-01-15
# Go Version: go1.21
```

---

## Step 913: คำสั่งพื้นฐาน

### Scan Commands หลัก

```bash
# สแกน AWS Terraform
terrascan scan -t aws

# สแกน Azure Terraform
terrascan scan -t azure

# สแกน GCP Terraform
terrascan scan -t gcp

# สแกน Kubernetes manifests
terrascan scan -t k8s

# สแกน Docker
terrascan scan -t docker

# สแกน GitHub
terrascan scan -t github

# สแกน directory เฉพาะ
terrascan scan -t aws -d ./terraform/aws/

# สแกน file เฉพาะ
terrascan scan -t aws -f main.tf

# สแกน หลาย type พร้อมกัน
terrascan scan -t aws -t k8s
```

### เลือก IaC Type ด้วย -i

```bash
# Auto-detect (default)
terrascan scan

# ระบุ IaC type
terrascan scan -i terraform
terrascan scan -i k8s
terrascan scan -i helm
terrascan scan -i kustomize
terrascan scan -i dockerfile
terrascan scan -i cfntemplate  # CloudFormation
```

### Options สำคัญ

```bash
# Output format
terrascan scan -t aws --output json
terrascan scan -t aws --output yaml
terrascan scan -t aws --output xml
terrascan scan -t aws --output human     # default
terrascan scan -t aws --output junit-xml
terrascan scan -t aws --output sarif
terrascan scan -t aws --output github-sarif

# Save output to file
terrascan scan -t aws --output json > results.json

# Filter by severity
terrascan scan -t aws --severity HIGH
terrascan scan -t aws --severity CRITICAL

# Verbose output
terrascan scan -t aws --log-level debug

# ไม่รัน specific policies
terrascan scan -t aws --skip-rules AC_AWS_0214

# รัน specific policies เท่านั้น
terrascan scan -t aws --use-policies "AC_AWS_0214,AC_AWS_0209"

# Config file
terrascan scan --config-path .terrascan.toml

# Terraform variable file
terrascan scan -t aws --var-file terraform.tfvars

# Non-recursive (ไม่ scan subdirectories)
terrascan scan -t aws --non-recursive
```

### Exit Codes

```bash
# Exit codes:
# 0 = ผ่าน (ไม่มี violations)
# 3 = มี violations

terrascan scan -t aws .
echo "Exit code: $?"

# ใช้ใน CI
terrascan scan -t aws . || exit 1
```

---

## Step 914: Output Formats

### Human Format (Default)

```bash
terrascan scan -t aws
```

Output:
```
Violation Details -
    
    Description    :    Ensure that all data stored in the S3 bucket is securely encrypted
    File           :    main.tf
    Module Name    :    root
    Plan Root      :    ./
    Line           :    1
    Severity       :    HIGH
    Rule Name      :    s3BucketSSEEnabled
    Rule ID        :    AC_AWS_0214
    Resource Name  :    aws_s3_bucket.example
    Resource Type  :    aws_s3_bucket
    Tags           :    (none)
    
Scan Summary -

    File/Folder         :    ./
    IaC Type            :    terraform
    Scanned At          :    2024-01-15 10:30:00 +0000 UTC
    Policies Validated  :    143
    Violated Policies   :    5
    Low                 :    0
    Medium              :    2
    High                :    3
    Critical            :    0
    Skipped             :    0
```

### JSON Format

```bash
terrascan scan -t aws --output json
```

```json
{
  "results": {
    "violations": [
      {
        "rule_name": "s3BucketSSEEnabled",
        "description": "Ensure that all data stored in the S3 bucket is securely encrypted",
        "rule_id": "AC_AWS_0214",
        "severity": "HIGH",
        "category": "ENCRYPTION",
        "resource_name": "aws_s3_bucket.example",
        "resource_type": "aws_s3_bucket",
        "file": "main.tf",
        "line": 1,
        "plan_root": "./",
        "source_type": "terraform",
        "tags": []
      }
    ],
    "skipped_violations": [],
    "summary": {
      "low": 0,
      "medium": 2,
      "high": 3,
      "critical": 0,
      "total": 5
    }
  }
}
```

### SARIF Format

```bash
terrascan scan -t aws --output sarif > terrascan-results.sarif
```

### GitHub SARIF

```bash
terrascan scan -t aws --output github-sarif > terrascan-results.sarif
# ปรับ format ให้เหมาะกับ GitHub Code Scanning
```

### JUnit XML

```bash
terrascan scan -t aws --output junit-xml > terrascan-junit.xml
```

```xml
<?xml version="1.0" encoding="UTF-8"?>
<testsuites>
  <testsuite name="terrascan" failures="5" tests="143">
    <testcase classname="AC_AWS_0214" name="s3BucketSSEEnabled">
      <failure>
        File: main.tf, Line: 1
        Resource: aws_s3_bucket.example
        Severity: HIGH
      </failure>
    </testcase>
  </testsuite>
</testsuites>
```

---

## Step 915: Terrascan Policies

### Policy Categories

```
Policy Categories:
├── ENCRYPTION         - การเข้ารหัสข้อมูล
├── LOGGING            - การ logging และ auditing
├── NETWORK_SECURITY   - ความปลอดภัยเครือข่าย
├── IAM                - Identity and Access Management
├── INFRASTRUCTURE     - ความปลอดภัยโครงสร้างพื้นฐาน
├── COMPLIANCE         - การปฏิบัติตามกฎระเบียบ
├── AVAILABILITY       - ความพร้อมใช้งาน
└── DATA_PROTECTION    - การปกป้องข้อมูล
```

### Key Terrascan Rules

**AC_AWS_0214: S3 Bucket Encryption**
```hcl
# ❌ ผิด
resource "aws_s3_bucket" "bad" {
  bucket = "my-bucket"
}

# ✅ ถูก
resource "aws_s3_bucket" "good" {
  bucket = "my-bucket"
}

resource "aws_s3_bucket_server_side_encryption_configuration" "good" {
  bucket = aws_s3_bucket.good.id
  rule {
    apply_server_side_encryption_by_default {
      sse_algorithm = "AES256"
    }
  }
}
```

**AC_AWS_0209: IAM Admin Policy**
```hcl
# ❌ ผิด - Admin privileges
resource "aws_iam_user_policy" "bad" {
  name = "policy"
  user = aws_iam_user.user.name
  
  policy = jsonencode({
    Statement = [{
      Effect   = "Allow"
      Action   = "*"
      Resource = "*"
    }]
  })
}

# ✅ ถูก - Least privilege
resource "aws_iam_user_policy" "good" {
  name = "policy"
  user = aws_iam_user.user.name
  
  policy = jsonencode({
    Statement = [{
      Effect   = "Allow"
      Action   = ["s3:GetObject", "s3:PutObject"]
      Resource = "arn:aws:s3:::specific-bucket/*"
    }]
  })
}
```

**AC_AWS_0369: Security Group Open to 0.0.0.0/0**
```hcl
# ❌ ผิด
resource "aws_security_group" "bad" {
  ingress {
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]  # ❌
  }
}

# ✅ ถูก
resource "aws_security_group" "good" {
  ingress {
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = ["10.0.0.0/8"]  # ✅ Private only
  }
}
```

**AC_AWS_0226: RDS Not Encrypted**
```hcl
# ❌ ผิด
resource "aws_db_instance" "bad" {
  storage_encrypted = false  # ❌
}

# ✅ ถูก
resource "aws_db_instance" "good" {
  storage_encrypted = true   # ✅
  kms_key_id        = aws_kms_key.rds.arn
}
```

### ดู Policies ทั้งหมด

```bash
# ดู policy list
terrascan list policies

# ดู policy by cloud provider
terrascan list policies -t aws

# ดู policy details
terrascan list policies -t aws --output json | jq '.[] | select(.rule_id == "AC_AWS_0214")'
```

---

## Step 916: Writing Custom Terrascan Policies ด้วย Rego

### โครงสร้างไฟล์ Policy

```
custom-policies/
├── aws/
│   ├── s3/
│   │   ├── mandatory-tags.rego
│   │   └── mandatory-tags.json
│   ├── ec2/
│   │   ├── approved-instance-types.rego
│   │   └── approved-instance-types.json
│   └── general/
│       ├── naming-convention.rego
│       └── naming-convention.json
└── gcp/
    └── ...
```

### Metadata JSON File

```json
{
  "name": "mandatory_tags",
  "file": "mandatory-tags.rego",
  "policy_type": "terraform",
  "resource_type": "aws_s3_bucket",
  "template_args": {
    "prefix": "mandatory",
    "suffix": "tags"
  },
  "severity": "HIGH",
  "description": "Ensure S3 bucket has mandatory tags",
  "reference_id": "CUS-001",
  "category": "INFRASTRUCTURE",
  "version": 2
}
```

### Rego Policy File

```rego
# mandatory-tags.rego
package rules.mandatory_tags

import data.lib.tfplan as tfplan

resource_type := "MULTIPLE"

# Define required tags
required_tags := {"Owner", "Environment", "Project", "CostCenter"}

# Check all taggable resources
s3_bucket := tfplan.resources.aws_s3_bucket

deny[msg] {
    bucket := s3_bucket[_]
    existing_tags := {k | bucket.config.tags[k]}
    missing := required_tags - existing_tags
    count(missing) > 0
    msg := sprintf(
        "S3 bucket '%v' is missing required tags: %v",
        [bucket.name, missing]
    )
}
```

### ตัวอย่างที่ 2: Approved Instance Types

```rego
# approved-instance-types.rego
package rules.approved_instance_types

import data.lib.tfplan as tfplan

resource_type := "MULTIPLE"

approved_instance_types := {
    "t3.micro", "t3.small", "t3.medium",
    "t3.large", "t3.xlarge",
    "m5.large", "m5.xlarge", "m5.2xlarge",
    "c5.large", "c5.xlarge"
}

instances := tfplan.resources.aws_instance

deny[msg] {
    instance := instances[_]
    instance_type := instance.config.instance_type
    not approved_instance_types[instance_type]
    msg := sprintf(
        "EC2 instance '%v' uses unapproved instance type '%v'. Allowed: %v",
        [instance.name, instance_type, approved_instance_types]
    )
}
```

### ตัวอย่างที่ 3: Naming Convention

```rego
# naming-convention.rego
package rules.naming_convention

import data.lib.tfplan as tfplan
import future.keywords

resource_type := "MULTIPLE"

# Format: {env}-{service}-{resource}-{sequence}
# Example: prod-api-sg-001
valid_name_pattern := `^(dev|staging|prod)-[a-z0-9]+-[a-z0-9]+-[0-9]{3}$`

# Check Security Groups
security_groups := tfplan.resources.aws_security_group

deny[msg] {
    sg := security_groups[_]
    sg_name := sg.config.name
    not regex.match(valid_name_pattern, sg_name)
    msg := sprintf(
        "Security group '%v' (name: '%v') doesn't follow naming convention. Expected: {env}-{service}-sg-{seq}",
        [sg.name, sg_name]
    )
}
```

### รัน Custom Policies

```bash
# รัน พร้อม custom policy directory
terrascan scan -t aws \
  --policy-path ./custom-policies \
  -d ./terraform/

# รัน เฉพาะ custom policies (ไม่รัน built-in)
terrascan scan -t aws \
  --policy-path ./custom-policies \
  --skip-rules ALL \
  -d ./terraform/

# ดู custom policies ที่โหลด
terrascan list policies --policy-path ./custom-policies
```

### Testing Rego Policies

```bash
# สร้าง test file
cat > test-mandatory-tags_test.rego << 'EOF'
package rules.mandatory_tags

test_bucket_with_all_tags {
  not deny[_] with input as {
    "aws_s3_bucket": {
      "my_bucket": {
        "config": {
          "tags": {
            "Owner": "team-platform",
            "Environment": "prod",
            "Project": "my-project",
            "CostCenter": "CC-001"
          }
        }
      }
    }
  }
}

test_bucket_missing_tags {
  deny[_] with input as {
    "aws_s3_bucket": {
      "my_bucket": {
        "config": {
          "tags": {
            "Owner": "team-platform"
          }
        }
      }
    }
  }
}
EOF

# รัน tests ด้วย OPA
opa test . -v
```

---

## Step 917: Terrascan Server Mode

### ทำไมต้องใช้ Server Mode?

Server mode ทำให้ Terrascan ทำงานเป็น REST API server ซึ่งมีประโยชน์สำหรับ:
- Integration กับ CI/CD pipelines
- Kubernetes Admission Controller
- Custom tooling
- Central policy enforcement

### เริ่มต้น Terrascan Server

```bash
# เริ่ม server บน port 9010 (default)
terrascan server

# เริ่ม server บน custom port
terrascan server --port 8080

# เริ่ม server พร้อม TLS
terrascan server \
  --cert-path /path/to/cert.pem \
  --key-path /path/to/key.pem

# ด้วย Docker
docker run -d \
  --name terrascan-server \
  -p 9010:9010 \
  tenable/terrascan:latest \
  server
```

### API Endpoints

```bash
# Health check
curl http://localhost:9010/health

# Scan Terraform
curl -X POST \
  http://localhost:9010/v1/terraform12/aws/local/file/scan \
  -H 'Content-Type: multipart/form-data' \
  -F 'file=@main.tf'

# Scan ด้วย archive (.zip)
zip -r terraform.zip .
curl -X POST \
  http://localhost:9010/v1/terraform12/aws/local/zip/scan \
  -F 'file=@terraform.zip'

# Response
{
  "violations": [...],
  "summary": {
    "low": 0,
    "medium": 2,
    "high": 3,
    "critical": 0
  }
}
```

### Kubernetes Admission Controller

```yaml
# admission-controller.yaml
apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingWebhookConfiguration
metadata:
  name: terrascan-webhook
  annotations:
    cert-manager.io/inject-ca-from: terrascan-system/terrascan-webhook-cert
webhooks:
  - name: validate.terrascan.io
    clientConfig:
      service:
        name: terrascan-webhook-service
        namespace: terrascan-system
        path: /v1/terraform12/k8s/webhook
    rules:
      - operations: ["CREATE", "UPDATE"]
        apiGroups: ["*"]
        apiVersions: ["*"]
        resources: ["pods", "deployments", "services"]
    admissionReviewVersions: ["v1"]
    sideEffects: None
    failurePolicy: Fail
```

### Helm Chart Deployment

```bash
# เพิ่ม Terrascan Helm repo
helm repo add terrascan https://tenable.github.io/terrascan

# ติดตั้ง
helm install terrascan terrascan/terrascan \
  --namespace terrascan-system \
  --create-namespace \
  --set server.port=9010 \
  --set policyUrl="https://github.com/tenable/terrascan/archive/master.tar.gz"
```

---

## Step 918: Terrascan ใน CI/CD

### GitHub Actions - Complete Workflow

```yaml
# .github/workflows/terrascan.yml
name: Terrascan Security Scan

on:
  pull_request:
    branches: [main, develop]
  push:
    branches: [main]

permissions:
  contents: read
  security-events: write
  pull-requests: write

jobs:
  terrascan:
    name: Terrascan IaC Security
    runs-on: ubuntu-latest
    
    steps:
      - name: Checkout Code
        uses: actions/checkout@v4
      
      # ===== Run Terrascan =====
      - name: Run Terrascan
        id: terrascan
        uses: tenable/terrascan-action@main
        with:
          iac_type: 'terraform'
          iac_version: 'v14'
          policy_type: 'aws'
          only_warn: false
          sarif_upload: true
          non_recursive: false
          verbose: true
          find_vulnerabilities: false
          policy_path: './custom-policies'  # ถ้ามี custom policies
          skip_rules: ''
          config_path: '.terrascan.toml'
      
      # ===== Upload SARIF =====
      - name: Upload SARIF file
        uses: github/codeql-action/upload-sarif@v3
        if: always()
        with:
          sarif_file: terrascan.sarif
          category: 'terrascan'
      
      # ===== Generate Report =====
      - name: Generate JSON Report
        if: always()
        run: |
          docker run --rm \
            -v $(pwd):/workspace \
            tenable/terrascan:latest \
            scan -t aws \
            -d /workspace \
            --output json \
            > terrascan-report.json || true
          
          # Extract counts
          CRITICAL=$(jq '.results.summary.critical // 0' terrascan-report.json)
          HIGH=$(jq '.results.summary.high // 0' terrascan-report.json)
          MEDIUM=$(jq '.results.summary.medium // 0' terrascan-report.json)
          LOW=$(jq '.results.summary.low // 0' terrascan-report.json)
          
          echo "CRITICAL=$CRITICAL" >> $GITHUB_ENV
          echo "HIGH=$HIGH" >> $GITHUB_ENV
          echo "MEDIUM=$MEDIUM" >> $GITHUB_ENV
          echo "LOW=$LOW" >> $GITHUB_ENV
      
      # ===== PR Comment =====
      - name: Comment on PR
        if: github.event_name == 'pull_request'
        uses: actions/github-script@v7
        with:
          script: |
            const critical = process.env.CRITICAL || '0';
            const high = process.env.HIGH || '0';
            const medium = process.env.MEDIUM || '0';
            const low = process.env.LOW || '0';
            
            const icon = parseInt(critical) > 0 ? '🚨' : 
                        parseInt(high) > 0 ? '⚠️' : '✅';
            
            await github.rest.issues.createComment({
              issue_number: context.issue.number,
              owner: context.repo.owner,
              repo: context.repo.repo,
              body: `## ${icon} Terrascan Security Report
              
              | Severity | Count |
              |----------|-------|
              | 🔴 Critical | ${critical} |
              | 🟠 High | ${high} |
              | 🟡 Medium | ${medium} |
              | 🔵 Low | ${low} |
              
              ${parseInt(critical) > 0 ? '> ⛔ **CRITICAL findings must be fixed before merge!**' : ''}
              `
            });
      
      # ===== Fail on Critical =====
      - name: Fail on Critical Findings
        if: env.CRITICAL != '0'
        run: |
          echo "::error::Found $CRITICAL critical findings. Cannot merge!"
          exit 1
```

### GitLab CI Pipeline

```yaml
# .gitlab-ci.yml
terrascan:
  stage: security
  image: tenable/terrascan:latest
  script:
    - |
      terrascan scan -t aws \
        -d . \
        --output json \
        > terrascan-report.json
      
      # แสดงผล human-readable ด้วย
      terrascan scan -t aws -d .
  artifacts:
    reports:
      sast: terrascan-report.json
    when: always
    expire_in: 30 days
  rules:
    - if: '$CI_PIPELINE_SOURCE == "merge_request_event"'
    - if: '$CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH'
```

### Jenkins Pipeline

```groovy
pipeline {
    agent any
    
    stages {
        stage('Terrascan') {
            steps {
                script {
                    sh '''
                        docker run --rm \
                          -v ${WORKSPACE}:/workspace \
                          tenable/terrascan:latest \
                          scan -t aws \
                          -d /workspace \
                          --output json \
                          > terrascan-report.json || true
                    '''
                    
                    def report = readJSON file: 'terrascan-report.json'
                    def critical = report.results.summary.critical ?: 0
                    def high = report.results.summary.high ?: 0
                    
                    if (critical > 0) {
                        error("Terrascan found ${critical} CRITICAL findings!")
                    }
                    
                    if (high > 0) {
                        unstable("Terrascan found ${high} HIGH findings")
                    }
                }
            }
            post {
                always {
                    archiveArtifacts artifacts: 'terrascan-report.json'
                }
            }
        }
    }
}
```

---

## Step 919: Terrascan Configuration File

### .terrascan.toml

```toml
# .terrascan.toml - Terrascan configuration
[logging]
level = "info"
log_type = "json"

[notifications]
webhook_url = "https://hooks.slack.com/services/..."

[policy]
# ที่อยู่ policy directory
policy_path = ["./custom-policies"]

# Skip specific rules
skip_rules = [
  "AC_AWS_0214",  # S3 encryption - handled by default encryption policy
]

# Severity filter
severity = "MEDIUM"

# Repository policies
[policy.policy_update]
auto_update = false

[scan]
# Directories to scan
iac_dir = "./terraform"

# File types
iac_type = "terraform"

# Non-recursive
non_recursive = false

# Show only violations (no passed checks)
violations_only = true
```

### .terrascanignore

```
# ไฟล์นี้ระบุ resources ที่ต้องการ ignore

# Format: {resource_type}.{resource_name}:{rule_id}
# หรือ: {file_path}:{line_number}:{rule_id}

aws_s3_bucket.logs:AC_AWS_0214
aws_s3_bucket.temp:AC_AWS_0132
main.tf:15:AC_AWS_0369
```

---

## Step 920: Terrascan กับ Kubernetes

### Scan Kubernetes Manifests

```bash
# สแกน K8s manifests
terrascan scan -i k8s -d ./k8s-manifests/

# สแกน Helm chart
terrascan scan -i helm -d ./helm-charts/my-app/

# สแกน Kustomize
terrascan scan -i kustomize -d ./kustomize/

# สแกน เฉพาะไฟล์
terrascan scan -i k8s -f deployment.yaml
```

### K8s Policy Examples

```hcl
# Terrascan K8s rules:
# - AC_K8S_0001: Default namespace usage
# - AC_K8S_0002: Privileged containers
# - AC_K8S_0003: Root containers
# - AC_K8S_0004: Resources limits not set
```

```yaml
# ❌ ผิด
apiVersion: apps/v1
kind: Deployment
metadata:
  name: bad-app
  namespace: default  # AC_K8S_0001 - Don't use default namespace
spec:
  template:
    spec:
      containers:
        - name: app
          image: myapp:latest
          securityContext:
            privileged: true  # AC_K8S_0002 - No privileged containers!

# ✅ ถูก
apiVersion: apps/v1
kind: Deployment
metadata:
  name: good-app
  namespace: production  # ✅ ใช้ namespace เฉพาะ
spec:
  template:
    spec:
      containers:
        - name: app
          image: myapp:1.2.3  # ✅ Pin version
          securityContext:
            privileged: false    # ✅
            runAsNonRoot: true   # ✅
            runAsUser: 1000      # ✅
            readOnlyRootFilesystem: true  # ✅
          resources:
            limits:
              cpu: "500m"
              memory: "512Mi"
            requests:
              cpu: "100m"
              memory: "128Mi"
```

### IDE Integration

```bash
# VS Code Extension
# ค้นหา "Terrascan" ใน VS Code Extensions marketplace
# - เปิด .tf file
# - Auto-scan เมื่อ save
# - แสดง problems ใน Problems tab

# JetBrains Plugin (IntelliJ, GoLand)
# ค้นหา "Terrascan" ใน JetBrains Marketplace
```

---

## สรุป Terrascan Commands

```bash
# Quick Reference
terrascan version                          # Version
terrascan scan -t aws                      # Scan AWS Terraform
terrascan scan -t aws -d ./tf/             # Scan directory
terrascan scan -t aws --severity HIGH      # Filter severity
terrascan scan -t aws --output json        # JSON output
terrascan scan -t aws --output sarif       # SARIF output
terrascan list policies -t aws             # List policies
terrascan server                           # Start server
terrascan init                             # Initialize/update policies
```

---

## แบบฝึกหัด

1. ติดตั้ง Terrascan บนเครื่อง local
2. สแกน Terraform code ที่มีอยู่ด้วย `terrascan scan -t aws`
3. เขียน Custom Policy สำหรับ mandatory tags
4. ตั้งค่า `.terrascanignore` เพื่อ ignore findings บาง rule
5. ตั้งค่า GitHub Actions workflow พร้อม SARIF upload
6. ลองใช้ Server mode และเรียก REST API

---

*จบ Part 92: Terrascan Security Scanner*
