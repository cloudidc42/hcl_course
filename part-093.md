# Part 93: Snyk IaC Security (Steps 921-930)

## การใช้ Snyk สำหรับตรวจสอบความปลอดภัย Infrastructure as Code

---

## Step 921: Snyk IaC Overview

### Snyk คืออะไร?

**Snyk** เป็น Developer Security Platform ที่ครอบคลุม:
- **Snyk Open Source**: ตรวจจับ vulnerabilities ใน dependencies
- **Snyk Code**: SAST สำหรับ application code
- **Snyk Container**: Security scanning สำหรับ Docker images
- **Snyk IaC**: Security scanning สำหรับ Infrastructure as Code ✅
- **Snyk Cloud**: Runtime scanning สำหรับ Cloud environments

### Snyk IaC รองรับ

```
Snyk IaC ตรวจสอบ:
├── Terraform (.tf files)
├── CloudFormation (JSON/YAML)
├── Kubernetes manifests
├── Helm charts
├── Kustomize
├── Azure Resource Manager (ARM)
└── Serverless (serverless.yml)
```

### จุดเด่นของ Snyk

```
┌────────────────────────────────────────────────────────────┐
│ ✅ 1000+ IaC security rules                               │
│ ✅ Drift Detection (snyk iac describe)                    │
│ ✅ Auto-fix suggestions                                    │
│ ✅ PR checks integration                                   │
│ ✅ Web dashboard                                          │
│ ✅ Team collaboration                                     │
│ ✅ Organization-level policies                            │
│ ✅ SARIF output                                           │
│ ✅ VS Code extension                                      │
└────────────────────────────────────────────────────────────┘
```

---

## Step 922: การติดตั้งและ Authentication

### ติดตั้ง Snyk CLI

```bash
# npm (recommended)
npm install -g snyk

# หรือ standalone binary
# macOS
brew install snyk

# Linux
curl https://static.snyk.io/cli/latest/snyk-linux -o snyk
chmod +x ./snyk
sudo mv ./snyk /usr/local/bin/

# Windows (Chocolatey)
choco install snyk

# ตรวจสอบ version
snyk version
# snyk/1.1234.0 (linux-x64) node/v20.0.0
```

### Authentication

```bash
# ===== วิธีที่ 1: Browser-based auth =====
snyk auth
# เปิด browser และ login
# คัดลอก token กลับมา

# ===== วิธีที่ 2: API Token =====
# ไปที่ https://app.snyk.io/account
# คัดลอก API token
export SNYK_TOKEN="your-api-token-here"
snyk auth $SNYK_TOKEN

# ===== วิธีที่ 3: Service Account (สำหรับ CI/CD) =====
# สร้าง Service Account ใน Snyk dashboard
# Organization Settings > Service accounts
export SNYK_TOKEN="service-account-token"

# ===== ตรวจสอบการ auth =====
snyk whoami
# ✓ Authenticated as: your.name@company.com
# ✓ Organization: your-org
```

---

## Step 923: snyk iac test - Basic Scanning

### คำสั่งพื้นฐาน

```bash
# สแกน directory ปัจจุบัน
snyk iac test .

# สแกน file เฉพาะ
snyk iac test main.tf

# สแกน directory เฉพาะ
snyk iac test ./terraform/

# สแกน หลาย files
snyk iac test main.tf variables.tf outputs.tf

# สแกน แบบ recursive
snyk iac test --recursive

# แสดง all issues (รวม LOW)
snyk iac test --severity-threshold=low .

# แสดงเฉพาะ HIGH และ CRITICAL
snyk iac test --severity-threshold=high .

# Report แบบ JSON
snyk iac test --json .

# Report แบบ SARIF
snyk iac test --sarif .

# Save output to file
snyk iac test --json . > snyk-report.json
snyk iac test --sarif . > snyk-results.sarif

# Fail บน specific severity
snyk iac test --severity-threshold=critical . || exit 1
```

### ตัวอย่าง Output

```
Testing main.tf...

Infrastructure as code issues:

  ✗ S3 Bucket is not encrypted [High Severity] [SNYK-CC-TF-45] in resource:aws_s3_bucket[main]
    introduced by input > aws_s3_bucket[main] > server_side_encryption_configuration

  ✗ Security Group allows ingress from 0.0.0.0/0 on port 22 [Critical Severity] [SNYK-CC-TF-73] in resource:aws_security_group[web]
    introduced by input > aws_security_group[web] > ingress

  ✗ RDS instance is not encrypted [High Severity] [SNYK-CC-AWS-415] in resource:aws_db_instance[main]
    introduced by input > aws_db_instance[main] > storage_encrypted

-------------------------------------------------------

Test Summary

  Organization:      my-org
  Project name:      my-terraform-project

✔ Files without issues: 2
✗ Files with issues: 1
  Ignored issues: 0
  Total issues: 3 [ 0 critical, 2 high, 1 medium, 0 low ]

-------------------------------------------------------
```

### Remediation Guidance

Snyk IaC มีคำแนะนำการแก้ไข:
```
  ✗ S3 Bucket is not encrypted [High Severity] [SNYK-CC-TF-45]
  
  Impact: The S3 bucket contents could be read if compromised.
  
  Remediation: Add server_side_encryption_configuration block:
  
    resource "aws_s3_bucket" "main" {
      ...
      server_side_encryption_configuration {
        rule {
          apply_server_side_encryption_by_default {
            sse_algorithm = "AES256"
          }
        }
      }
    }
  
  References:
    - https://docs.aws.amazon.com/AmazonS3/latest/userguide/bucket-encryption.html
    - https://snyk.io/security-rules/SNYK-CC-TF-45
```

---

## Step 924: Snyk IaC Rules

### High Severity Examples

```hcl
# SNYK-CC-TF-45: S3 bucket not encrypted
resource "aws_s3_bucket" "bad" {
  # ❌ ขาด server_side_encryption_configuration
}

# SNYK-CC-TF-73: Security group port 22 open to world
resource "aws_security_group" "bad" {
  ingress {
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]  # ❌
  }
}

# SNYK-CC-AWS-415: RDS not encrypted
resource "aws_db_instance" "bad" {
  storage_encrypted = false  # ❌
}

# SNYK-CC-AWS-422: CloudTrail not encrypted
resource "aws_cloudtrail" "bad" {
  # ❌ ไม่มี kms_key_id
}
```

### Medium Severity Examples

```hcl
# SNYK-CC-TF-4: S3 bucket not versioned
resource "aws_s3_bucket" "bad" {
  # ❌ ไม่มี versioning block
}

# SNYK-CC-AWS-417: RDS no backup
resource "aws_db_instance" "bad" {
  backup_retention_period = 0  # ❌
}

# SNYK-CC-TF-20: ELB no access logs
resource "aws_lb" "bad" {
  # ❌ ไม่มี access_logs block
}

# SNYK-CC-AWS-427: Lambda no tracing
resource "aws_lambda_function" "bad" {
  # ❌ ไม่มี tracing_config
}
```

### Low Severity Examples

```hcl
# ขาด description ใน security group rules
resource "aws_security_group_rule" "bad" {
  description = ""  # ❌ ควรมี description
}

# ไม่มี deletion_protection ใน RDS
resource "aws_db_instance" "bad" {
  deletion_protection = false  # ❌ ในสภาพแวดล้อม production
}
```

---

## Step 925: .snyk File สำหรับ Ignoring

### สร้าง .snyk file

```yaml
# .snyk
version: v1.25.0
ignore:
  # ====================================================
  # Format: {rule-id}:
  #   {resource-path}:
  #     reason: 'เหตุผล'
  #     expires: 'YYYY-MM-DDT00:00:00.000Z'  (optional)
  #     created: 'YYYY-MM-DDT00:00:00.000Z'
  # ====================================================
  
  SNYK-CC-TF-45:  # S3 not encrypted
    - aws_s3_bucket.logs:
        reason: >
          Log bucket uses S3 default encryption via bucket policy.
          Reviewed by: security-team@company.com
        expires: "2025-01-01T00:00:00.000Z"
        created: "2024-01-15T00:00:00.000Z"
  
  SNYK-CC-TF-4:   # S3 not versioned
    - aws_s3_bucket.temp:
        reason: >
          Temporary bucket for ephemeral data, versioning not needed.
          Expiry controlled by lifecycle rules.
        created: "2024-01-15T00:00:00.000Z"
  
  SNYK-CC-TF-73:  # Security group open 22
    - aws_security_group.bastion:
        reason: >
          Bastion host is only accessible from VPN (controlled by NACL).
          IP is further restricted by IAM policy conditions.
          Approved by CISO on 2024-01-10.
        expires: "2024-06-01T00:00:00.000Z"
        created: "2024-01-15T00:00:00.000Z"

exclude:
  - tests/**  # ไม่สแกน test directory
  - examples/**
```

### การจัดการ .snyk ด้วย CLI

```bash
# Ignore finding แบบ interactive
snyk ignore \
  --id=SNYK-CC-TF-45 \
  --reason="Encrypted at organization level" \
  --policy-path=.snyk

# View ignored issues
snyk iac test --json . | jq '.[], .ignored | length'
```

---

## Step 926: Snyk IAC Describe - Drift Detection

### Drift Detection คืออะไร?

Drift เกิดขึ้นเมื่อ infrastructure จริงใน cloud ต่างจาก Terraform state:
- มีคนสร้าง resource ผ่าน console โดยตรง
- มีคนแก้ไข resource โดยไม่ผ่าน Terraform
- Resource ถูกลบโดยไม่ผ่าน Terraform

### ตั้งค่า snyk iac describe

```bash
# ต้องการ:
# 1. Snyk Account
# 2. Terraform state access
# 3. Cloud credentials

# สำหรับ AWS
export AWS_ACCESS_KEY_ID="your-key"
export AWS_SECRET_ACCESS_KEY="your-secret"
export AWS_DEFAULT_REGION="ap-southeast-1"

# สำหรับ Terraform state แบบ local
snyk iac describe \
  --from="tfstate://terraform.tfstate"

# สำหรับ Terraform state ใน S3
snyk iac describe \
  --from="tfstate+s3://my-terraform-state/path/to/terraform.tfstate"

# ตรวจสอบ resource เฉพาะ type
snyk iac describe \
  --from="tfstate://terraform.tfstate" \
  --filter="type=aws_s3_bucket"

# ทั้ง AWS account (ไม่ใช้ state)
snyk iac describe \
  --only-unmanaged
```

### ตัวอย่าง Output ของ Describe

```
Scanned states (1)
Found 12 resource(s) in state(s)
Found 4 resource(s) changed in real infrastructure

Changed Resources (4)
  
  aws_s3_bucket.assets:
    server_side_encryption_configuration:
      + rule:
          + apply_server_side_encryption_by_default:
              + sse_algorithm: AES256
    (Changed outside of Terraform - someone added encryption via console)
  
  aws_security_group.web:
    description: 
      - "Old description"
      + "Updated description"

Unmanaged Resources (2)
  (Resources in AWS but NOT in Terraform state)
  aws_s3_bucket: my-unknown-bucket
  aws_iam_user: orphan-user

Deleted Resources (1)
  (Resources in Terraform state but NOT in AWS)
  aws_elastic_ip.old: eipalloc-0123456789
```

---

## Step 927: Snyk Cloud (Runtime Scanning)

### Snyk Cloud vs Snyk IaC

```
Snyk IaC:
  - Scans Terraform CODE ก่อน deploy
  - Static analysis
  - Fast feedback loop
  - Free tier available

Snyk Cloud:
  - Scans LIVE cloud environment
  - Runtime detection
  - Continuous monitoring
  - Requires paid plan
```

### ตั้งค่า Snyk Cloud

```bash
# สร้าง environment ใน Snyk Cloud
snyk iac describe \
  --from="tfstate+s3://my-state-bucket/terraform.tfstate" \
  --cloud \
  --org="my-org-id"

# Dashboard: https://app.snyk.io/org/my-org-id/cloud
```

---

## Step 928: Snyk ใน CI/CD

### GitHub Actions - Complete Workflow

```yaml
# .github/workflows/snyk-security.yml
name: Snyk IaC Security

on:
  pull_request:
    branches: [main, develop]
    paths:
      - '**.tf'
      - '**.tfvars'
  push:
    branches: [main]

permissions:
  contents: read
  security-events: write
  pull-requests: write

jobs:
  snyk-iac:
    name: Snyk IaC Scan
    runs-on: ubuntu-latest
    
    steps:
      - name: Checkout Code
        uses: actions/checkout@v4
      
      # ===== Snyk IaC Test =====
      - name: Run Snyk to check IaC files for issues
        uses: snyk/actions/iac@master
        continue-on-error: true  # Upload SARIF even if fails
        env:
          SNYK_TOKEN: ${{ secrets.SNYK_TOKEN }}
        with:
          file: .  # Scan all Terraform files
          args: >
            --severity-threshold=medium
            --report
            --org=${{ secrets.SNYK_ORG_ID }}
      
      # ===== Upload SARIF =====
      - name: Upload SARIF to GitHub
        uses: github/codeql-action/upload-sarif@v3
        if: always()
        with:
          sarif_file: snyk.sarif
          category: 'snyk-iac'
      
      # ===== Snyk Test แบบ Manual สำหรับ PR Comment =====
      - name: Run Snyk for PR Details
        if: github.event_name == 'pull_request'
        env:
          SNYK_TOKEN: ${{ secrets.SNYK_TOKEN }}
        run: |
          npm install -g snyk
          
          # รัน scan เก็บ results
          snyk iac test . \
            --json \
            --severity-threshold=low \
            --org=${{ secrets.SNYK_ORG_ID }} \
            > snyk-results.json || true
          
          # Extract counts
          CRITICAL=$(jq '[.[][] | select(.severity == "critical")] | length' snyk-results.json 2>/dev/null || echo "0")
          HIGH=$(jq '[.[][] | select(.severity == "high")] | length' snyk-results.json 2>/dev/null || echo "0")
          MEDIUM=$(jq '[.[][] | select(.severity == "medium")] | length' snyk-results.json 2>/dev/null || echo "0")
          LOW=$(jq '[.[][] | select(.severity == "low")] | length' snyk-results.json 2>/dev/null || echo "0")
          
          echo "SNYK_CRITICAL=$CRITICAL" >> $GITHUB_ENV
          echo "SNYK_HIGH=$HIGH" >> $GITHUB_ENV
          echo "SNYK_MEDIUM=$MEDIUM" >> $GITHUB_ENV
          echo "SNYK_LOW=$LOW" >> $GITHUB_ENV
      
      - name: Create PR Comment with Snyk Results
        if: github.event_name == 'pull_request'
        uses: actions/github-script@v7
        with:
          script: |
            const critical = process.env.SNYK_CRITICAL;
            const high = process.env.SNYK_HIGH;
            const medium = process.env.SNYK_MEDIUM;
            const low = process.env.SNYK_LOW;
            
            const icon = parseInt(critical) > 0 ? '🚨' :
                        parseInt(high) > 0 ? '⚠️' : '✅';
            
            const body = `## ${icon} Snyk IaC Security Scan
            
            | Severity | Count | Status |
            |----------|-------|--------|
            | 🔴 Critical | ${critical} | ${parseInt(critical) > 0 ? '❌ Must Fix' : '✅'} |
            | 🟠 High | ${high} | ${parseInt(high) > 0 ? '⚠️ Should Fix' : '✅'} |
            | 🟡 Medium | ${medium} | ${parseInt(medium) > 0 ? '📋 Review' : '✅'} |
            | 🔵 Low | ${low} | ℹ️ FYI |
            
            **Policy**: PRs with Critical findings will be blocked from merging.
            
            View full report: [Snyk Dashboard](https://app.snyk.io)
            `;
            
            await github.rest.issues.createComment({
              issue_number: context.issue.number,
              owner: context.repo.owner,
              repo: context.repo.repo,
              body
            });
      
      # ===== Block on Critical =====
      - name: Fail on Critical Findings
        env:
          SNYK_TOKEN: ${{ secrets.SNYK_TOKEN }}
        run: |
          snyk iac test . \
            --severity-threshold=critical \
            --org=${{ secrets.SNYK_ORG_ID }}
```

### GitLab CI

```yaml
# .gitlab-ci.yml
snyk-iac:
  stage: security
  image: snyk/snyk:latest
  variables:
    SNYK_TOKEN: $SNYK_TOKEN
  script:
    - snyk auth $SNYK_TOKEN
    - snyk iac test . --json > snyk-report.json || true
    - snyk iac test . --severity-threshold=high
  artifacts:
    reports:
      sast: snyk-report.json
    paths:
      - snyk-report.json
    when: always
    expire_in: 30 days
```

### Jenkins Pipeline

```groovy
pipeline {
    agent any
    
    environment {
        SNYK_TOKEN = credentials('snyk-token')
    }
    
    stages {
        stage('Snyk IaC Scan') {
            steps {
                sh 'npm install -g snyk'
                sh 'snyk auth $SNYK_TOKEN'
                
                script {
                    def snykStatus = sh(
                        script: 'snyk iac test . --severity-threshold=high --json > snyk-results.json',
                        returnStatus: true
                    )
                    
                    archiveArtifacts artifacts: 'snyk-results.json'
                    
                    if (snykStatus != 0) {
                        def report = readJSON file: 'snyk-results.json'
                        error("Snyk found HIGH/CRITICAL IaC security issues!")
                    }
                }
            }
        }
    }
}
```

---

## Step 929: Snyk CLI Options ครบถ้วน

### Options ทั้งหมด

```bash
# ===== Basic Options =====
snyk iac test [options] [<path>]

# Output format
--json                  # JSON output
--sarif                 # SARIF output  
--json-file-output=<filename>   # Save JSON to file
--sarif-file-output=<filename>  # Save SARIF to file

# Severity filtering
--severity-threshold=<level>  # low, medium, high, critical

# Organization
--org=<org-id>          # Specify Snyk organization

# Scan behavior
--detection-depth=<n>   # How deep to scan (default: ∞)
--exclude=<pattern>     # Exclude paths
--recursive             # Scan subdirectories

# Reporting
--report                # Share results with Snyk dashboard
--target-name=<name>    # Name for report
--target-reference=<ref>  # Git branch/tag for context

# Variables (for Terraform plan scanning)
--var-file=<filename>   # Terraform .tfvars file
```

### ตัวอย่าง Advanced Usage

```bash
# Scan พร้อม variable files
snyk iac test . \
  --var-file=terraform.tfvars \
  --var-file=secrets.auto.tfvars

# Scan แล้ว report ขึ้น dashboard
snyk iac test . \
  --report \
  --org=my-org-id \
  --target-name=my-terraform-project \
  --target-reference=$(git rev-parse --abbrev-ref HEAD)

# Scan แต่ exclude test directories
snyk iac test . \
  --exclude=tests \
  --exclude=examples

# ใช้ JSON output เพื่อ process ต่อ
snyk iac test . --json | jq '.[] | {
  file: .targetFile,
  critical: [.infrastructureAsCodeIssues[] | select(.severity == "critical")] | length,
  high: [.infrastructureAsCodeIssues[] | select(.severity == "high")] | length
}'
```

---

## Step 930: Snyk vs Checkov vs TFSec Comparison

### ตารางเปรียบเทียบโดยละเอียด

```
┌─────────────────────┬─────────────────┬──────────────────┬──────────────────┐
│ Feature             │ Snyk IaC        │ Checkov          │ TFSec/Trivy      │
├─────────────────────┼─────────────────┼──────────────────┼──────────────────┤
│ License             │ Freemium        │ Apache 2.0       │ MIT/Apache       │
│ Free tier           │ 300 tests/month │ Unlimited        │ Unlimited        │
│ Paid features       │ Yes             │ Prisma Cloud     │ No               │
├─────────────────────┼─────────────────┼──────────────────┼──────────────────┤
│ Rules count         │ 1000+           │ 1000+            │ 200+             │
│ Custom rules        │ Limited (paid)  │ Python/YAML      │ Rego/YAML        │
│ Rule format         │ N/A (managed)   │ Python + YAML    │ Rego             │
├─────────────────────┼─────────────────┼──────────────────┼──────────────────┤
│ IaC support         │                 │                  │                  │
│   Terraform         │ ✅              │ ✅               │ ✅               │
│   CloudFormation    │ ✅              │ ✅               │ ❌               │
│   Kubernetes        │ ✅              │ ✅               │ ✅ (Trivy)       │
│   Helm              │ ✅              │ ✅               │ ✅ (Trivy)       │
│   ARM               │ ✅              │ ✅               │ ❌               │
│   Dockerfile        │ ✅              │ ✅               │ ✅ (Trivy)       │
├─────────────────────┼─────────────────┼──────────────────┼──────────────────┤
│ Drift Detection     │ ✅ (snyk iac   │ ❌               │ ❌               │
│                     │   describe)     │                  │                  │
│ Runtime Scanning    │ ✅ (Snyk Cloud) │ ❌               │ ❌               │
│ Web Dashboard       │ ✅              │ ✅ (paid)        │ ❌               │
│ PR Integration      │ ✅ Native       │ ✅               │ ✅               │
├─────────────────────┼─────────────────┼──────────────────┼──────────────────┤
│ SARIF output        │ ✅              │ ✅               │ ✅               │
│ JSON output         │ ✅              │ ✅               │ ✅               │
│ JUnit output        │ ❌              │ ✅               │ ✅               │
├─────────────────────┼─────────────────┼──────────────────┼──────────────────┤
│ Speed               │ ปานกลาง        │ ปานกลาง          │ เร็วมาก          │
│ Install size        │ node_modules   │ Python packages  │ Single binary    │
├─────────────────────┼─────────────────┼──────────────────┼──────────────────┤
│ Best for            │ Teams ที่ต้องการ│ Custom rules    │ Fast CI/CD       │
│                     │ dashboard +    │ + many IaC types │ + Kubernetes     │
│                     │ drift detection│                  │ security         │
└─────────────────────┴─────────────────┴──────────────────┴──────────────────┘
```

### เมื่อไหรควรใช้อะไร?

```
ใช้ Snyk IaC เมื่อ:
  - ต้องการ web dashboard สำหรับทีม
  - ต้องการ drift detection
  - มีงบประมาณสำหรับ paid features
  - ต้องการ native PR integration

ใช้ Checkov เมื่อ:
  - ต้องการ custom rules ด้วย Python
  - รองรับหลาย IaC types
  - ต้องการ free unlimited scans
  - ต้องการ extensive rule library

ใช้ TFSec/Trivy เมื่อ:
  - ต้องการ speed สูงสุด
  - Kubernetes + Terraform ใน repo เดียว
  - ต้องการ single binary (ง่ายต่อการ deploy)
  - ทำงานกับ Trivy อยู่แล้ว
```

### ใช้ร่วมกัน (Best Practice)

```bash
# Run ทั้งสามพร้อมกันใน CI/CD
# แต่ละ tool จะตรวจจับสิ่งที่ tool อื่นอาจพลาด

make security-scan:
  # เร็วสุด (binary)
  trivy config --severity HIGH,CRITICAL . --exit-code 1
  
  # ครอบคลุมสุด (1000+ rules)  
  checkov -d . --compact --framework terraform
  
  # Drift detection + Dashboard
  snyk iac test . --severity-threshold=high
```

---

## Snyk Fix - Auto-Remediation

### Snyk Fix (Experimental)

```bash
# รัน snyk iac test แล้ว snyk fix
snyk iac test . --json > issues.json

# ดู fixable issues
snyk iac test . | grep "fixable"

# Apply auto-fix (experimental)
snyk fix
# Snyk จะแก้ไข Terraform files อัตโนมัติ
```

### Organization Policies

```bash
# Organization Policies ตั้งค่าใน Snyk Dashboard:
# Organization > Policies > IaC

# ตัวอย่าง Policies:
# - Block PR ถ้ามี CRITICAL findings
# - Require review ถ้ามี HIGH findings
# - Send Slack notification สำหรับ MEDIUM+
```

---

## แบบฝึกหัด

1. สร้าง Snyk account (free tier)
2. ติดตั้ง Snyk CLI และ auth ด้วย API token
3. รัน `snyk iac test .` บน Terraform project
4. สร้าง `.snyk` file เพื่อ ignore specific finding พร้อมเหตุผล
5. ตั้งค่า GitHub Actions workflow พร้อม SNYK_TOKEN secret
6. ลอง drift detection ถ้ามี cloud credentials
7. เปรียบเทียบ results ระหว่าง Snyk, Checkov, และ TFSec

---

*จบ Part 93: Snyk IaC Security*
