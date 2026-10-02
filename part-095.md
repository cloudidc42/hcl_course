# Part 95: Security CI/CD Integration (Steps 941-950)

## การรวม Security Tools เข้ากับ CI/CD Pipeline

---

## Step 941: Overview - Security Pipeline Architecture

### หลักการ DevSecOps

```
Traditional:                    DevSecOps:
Dev → Build → Test → Deploy    Dev → Sec → Build → Sec → Test → Sec → Deploy

Security checks ที่ทุก stage:
┌─────────────────────────────────────────────────────────────┐
│ 1. Pre-commit (local)                                       │
│    - terraform fmt                                          │
│    - terraform validate                                     │
│    - TFSec/Trivy (fast check)                              │
│    - credential scanning (gitleaks, detect-secrets)        │
│                                                             │
│ 2. PR/CI Stage                                             │
│    - Full security scan (Checkov + TFSec + Snyk)           │
│    - terraform plan                                        │
│    - Cost estimation (infracost)                           │
│    - SARIF upload to GitHub Security                       │
│                                                             │
│ 3. Post-merge (main branch)                               │
│    - terraform apply                                       │
│    - Runtime security monitoring                          │
│    - Drift detection                                       │
│                                                             │
│ 4. Scheduled                                               │
│    - Weekly full security scan                            │
│    - Drift detection vs production                        │
│    - Compliance report generation                         │
└─────────────────────────────────────────────────────────────┘
```

---

## Step 942: Pre-commit Hooks

### ติดตั้ง Pre-commit

```bash
# macOS
brew install pre-commit

# Python pip
pip install pre-commit

# หรือ pipx (recommended)
pipx install pre-commit

# ตรวจสอบ
pre-commit --version
```

### .pre-commit-config.yaml สมบูรณ์

```yaml
# .pre-commit-config.yaml
# Pre-commit configuration for Terraform security

repos:
  # ====================================================
  # 1. Terraform formatting and validation
  # ====================================================
  - repo: https://github.com/antonbabenko/pre-commit-terraform
    rev: v1.88.0
    hooks:
      # Format Terraform files
      - id: terraform_fmt
        args:
          - --args=-recursive
          - --args=-diff
      
      # Validate Terraform
      - id: terraform_validate
        args:
          - --args=-no-color
        # ต้องการ terraform init ก่อน
        pass_filenames: false
      
      # TFLint
      - id: terraform_tflint
        args:
          - --args=--only=terraform_deprecated_interpolation
          - --args=--only=terraform_deprecated_index
          - --args=--only=terraform_unused_declarations
          - --args=--only=terraform_comment_syntax
          - --args=--only=terraform_documented_outputs
          - --args=--only=terraform_documented_variables
          - --args=--only=terraform_typed_variables
          - --args=--only=terraform_module_pinned_source
          - --args=--only=terraform_naming_convention
          - --args=--only=terraform_required_version
          - --args=--only=terraform_required_providers
      
      # TFSec - Security scan
      - id: terraform_tfsec
        args:
          - --args=--minimum-severity=HIGH
          - --args=--format=lovely
        # ทำงานเฉพาะเมื่อ .tf files เปลี่ยน
      
      # Trivy config scan (alternative to tfsec)
      - id: terraform_trivy
        args:
          - --args=--severity=HIGH,CRITICAL
          - --args=--exit-code=1
      
      # Checkov
      - id: terraform_checkov
        args:
          - --args=--quiet
          - --args=--compact
          - --args=--framework=terraform
          - --args=--skip-check=CKV_AWS_145  # ตัวอย่าง skip
        # ติดตั้ง checkov: pip install checkov
      
      # Terraform docs
      - id: terraform_docs
        args:
          - --hook-config=--path-to-file=README.md
          - --hook-config=--add-to-existing-file=true
          - --hook-config=--create-file-if-not-exist=true
  
  # ====================================================
  # 2. Credential Scanning
  # ====================================================
  
  # Detect-secrets - ตรวจจับ secrets ใน code
  - repo: https://github.com/Yelp/detect-secrets
    rev: v1.4.0
    hooks:
      - id: detect-secrets
        args:
          - '--baseline'
          - '.secrets.baseline'
        exclude: '(\.lock$|\.sum$|package-lock\.json)'
  
  # Gitleaks - ตรวจจับ secrets ที่ commit แล้ว
  - repo: https://github.com/gitleaks/gitleaks
    rev: v8.18.0
    hooks:
      - id: gitleaks
        args:
          - '--verbose'
  
  # ====================================================
  # 3. General Code Quality
  # ====================================================
  
  - repo: https://github.com/pre-commit/pre-commit-hooks
    rev: v4.5.0
    hooks:
      # ห้าม commit ไฟล์ขนาดใหญ่
      - id: check-added-large-files
        args: ['--maxkb=500']
      
      # ห้าม commit binary files ที่ไม่จำเป็น
      - id: check-case-conflict
      
      # ตรวจสอบ YAML syntax
      - id: check-yaml
        args: ['--unsafe']
      
      # ตรวจสอบ JSON syntax
      - id: check-json
      
      # ห้าม trailing whitespace
      - id: trailing-whitespace
        args: ['--markdown-linebreak-ext=md']
      
      # ต้องมี newline ท้ายไฟล์
      - id: end-of-file-fixer
      
      # ห้าม merge conflicts ที่ยังไม่แก้
      - id: check-merge-conflict
      
      # ตรวจหา private keys
      - id: detect-private-key
  
  # ====================================================
  # 4. Markdown linting
  # ====================================================
  
  - repo: https://github.com/igorshubovych/markdownlint-cli
    rev: v0.39.0
    hooks:
      - id: markdownlint
        args: ['--fix']
        files: '\.md$'
```

### ติดตั้งและใช้งาน

```bash
# ติดตั้ง hooks
pre-commit install

# ติดตั้ง commit-msg hook (ตรวจสอบ commit message format)
pre-commit install --hook-type commit-msg

# รัน hooks บน all files (ครั้งแรก)
pre-commit run --all-files

# รัน hook เฉพาะ
pre-commit run terraform_tfsec --all-files
pre-commit run detect-secrets --all-files

# อัพเดต hooks เป็น version ล่าสุด
pre-commit autoupdate

# ข้าม hook ชั่วคราว (ไม่แนะนำ แต่มีกรณีจำเป็น)
git commit -m "WIP" --no-verify

# สร้าง baseline สำหรับ detect-secrets
detect-secrets scan > .secrets.baseline
git add .secrets.baseline
```

---

## Step 943: GitHub Actions - Complete Security Pipeline

### Complete Terraform Security Workflow

```yaml
# .github/workflows/terraform-security-pipeline.yml
name: Terraform Security Pipeline

on:
  pull_request:
    branches: [main, develop, 'release/**']
    paths:
      - '**.tf'
      - '**.tfvars'
      - '**.json'
      - '.github/workflows/terraform-security-pipeline.yml'
  push:
    branches: [main]
    paths:
      - '**.tf'
  schedule:
    # รัน weekly security scan ทุกวันจันทร์ 8am UTC
    - cron: '0 8 * * 1'

permissions:
  contents: read
  security-events: write
  pull-requests: write
  id-token: write  # สำหรับ OIDC auth กับ AWS

env:
  TF_VERSION: "1.7.0"
  PYTHON_VERSION: "3.11"
  AWS_REGION: "ap-southeast-1"

jobs:
  # ====================================================
  # Job 1: Code Format Check
  # ====================================================
  terraform-fmt:
    name: "Terraform Format Check"
    runs-on: ubuntu-latest
    
    steps:
      - uses: actions/checkout@v4
      
      - uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: ${{ env.TF_VERSION }}
      
      - name: Terraform Format Check
        id: fmt
        run: terraform fmt -check -recursive -diff
      
      - name: Comment on PR if fmt fails
        if: failure() && github.event_name == 'pull_request'
        uses: actions/github-script@v7
        with:
          script: |
            await github.rest.issues.createComment({
              issue_number: context.issue.number,
              owner: context.repo.owner,
              repo: context.repo.repo,
              body: '❌ **Terraform fmt check failed!**\n\nPlease run `terraform fmt -recursive` to fix formatting.'
            });
  
  # ====================================================
  # Job 2: Terraform Validate
  # ====================================================
  terraform-validate:
    name: "Terraform Validate"
    runs-on: ubuntu-latest
    needs: terraform-fmt
    
    strategy:
      matrix:
        terraform_dir:
          - ./environments/dev
          - ./environments/staging
          - ./environments/prod
          - ./modules/networking
          - ./modules/compute
    
    steps:
      - uses: actions/checkout@v4
      
      - uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: ${{ env.TF_VERSION }}
      
      - name: Terraform Init (no backend)
        run: |
          cd ${{ matrix.terraform_dir }}
          terraform init -backend=false
      
      - name: Terraform Validate
        run: |
          cd ${{ matrix.terraform_dir }}
          terraform validate -no-color
  
  # ====================================================
  # Job 3: TFLint
  # ====================================================
  tflint:
    name: "TFLint"
    runs-on: ubuntu-latest
    needs: terraform-validate
    
    steps:
      - uses: actions/checkout@v4
      
      - uses: terraform-linters/setup-tflint@v4
        with:
          tflint_version: latest
      
      - name: Init TFLint
        run: tflint --init
        env:
          GITHUB_TOKEN: ${{ github.token }}
      
      - name: Run TFLint
        run: |
          tflint \
            --format compact \
            --recursive \
            --no-color
  
  # ====================================================
  # Job 4: TFSec/Trivy Security Scan
  # ====================================================
  tfsec:
    name: "TFSec Security Scan"
    runs-on: ubuntu-latest
    needs: terraform-fmt
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Run TFSec
        uses: aquasecurity/tfsec-action@v1.0.0
        with:
          working_directory: '.'
          format: 'sarif'
          output: 'tfsec-results.sarif'
          soft_fail: 'true'
          additional_args: '--minimum-severity MEDIUM'
      
      - name: Upload TFSec SARIF
        uses: github/codeql-action/upload-sarif@v3
        if: always()
        with:
          sarif_file: 'tfsec-results.sarif'
          category: 'tfsec'
      
      - name: Run Trivy Config Scan
        uses: aquasecurity/trivy-action@master
        with:
          scan-type: 'config'
          scan-ref: '.'
          format: 'sarif'
          output: 'trivy-results.sarif'
          severity: 'HIGH,CRITICAL'
          exit-code: '0'
      
      - name: Upload Trivy SARIF
        uses: github/codeql-action/upload-sarif@v3
        if: always()
        with:
          sarif_file: 'trivy-results.sarif'
          category: 'trivy-config'
      
      - name: Fail on Critical TFSec Findings
        run: |
          docker run --rm \
            -v $(pwd):/src \
            aquasec/tfsec:latest \
            --minimum-severity CRITICAL \
            --exit-code 1 \
            /src
  
  # ====================================================
  # Job 5: Checkov Security Scan
  # ====================================================
  checkov:
    name: "Checkov Security Scan"
    runs-on: ubuntu-latest
    needs: terraform-fmt
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Run Checkov
        id: checkov
        uses: bridgecrewio/checkov-action@master
        with:
          directory: .
          framework: terraform
          output_format: sarif
          output_file_path: checkov-results.sarif
          soft_fail: true
          skip_check: >-
            CKV_AWS_79,
            CKV_AWS_126
          download_external_modules: false
      
      - name: Upload Checkov SARIF
        uses: github/codeql-action/upload-sarif@v3
        if: always()
        with:
          sarif_file: checkov-results.sarif
          category: 'checkov'
      
      # Run again to fail on critical
      - name: Checkov Critical Check
        uses: bridgecrewio/checkov-action@master
        with:
          directory: .
          framework: terraform
          check: >-
            CKV_AWS_57,
            CKV_AWS_24,
            CKV_AWS_23
          soft_fail: false
  
  # ====================================================
  # Job 6: Snyk IaC
  # ====================================================
  snyk:
    name: "Snyk IaC Scan"
    runs-on: ubuntu-latest
    needs: terraform-fmt
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Run Snyk IaC
        uses: snyk/actions/iac@master
        continue-on-error: true
        env:
          SNYK_TOKEN: ${{ secrets.SNYK_TOKEN }}
        with:
          file: .
          args: --severity-threshold=medium
      
      - name: Upload Snyk SARIF
        uses: github/codeql-action/upload-sarif@v3
        if: always()
        with:
          sarif_file: snyk.sarif
          category: 'snyk-iac'
  
  # ====================================================
  # Job 7: Terraform Plan (with security gate)
  # ====================================================
  terraform-plan:
    name: "Terraform Plan"
    runs-on: ubuntu-latest
    needs: [tfsec, checkov]
    if: github.event_name == 'pull_request'
    
    # ต้องการ AWS credentials
    environment: 'plan'
    
    steps:
      - uses: actions/checkout@v4
      
      - uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: ${{ env.TF_VERSION }}
          cli_config_credentials_token: ${{ secrets.TF_API_TOKEN }}
      
      - name: Configure AWS Credentials (OIDC)
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ secrets.AWS_PLAN_ROLE_ARN }}
          aws-region: ${{ env.AWS_REGION }}
      
      - name: Terraform Init
        run: |
          terraform init \
            -backend-config="bucket=${{ secrets.TF_STATE_BUCKET }}" \
            -backend-config="key=terraform.tfstate" \
            -backend-config="region=${{ env.AWS_REGION }}"
      
      - name: Terraform Plan
        id: plan
        run: |
          terraform plan \
            -no-color \
            -out=tfplan.binary \
            -var-file="environments/dev/terraform.tfvars" \
            2>&1 | tee plan-output.txt
          
          # Save plan as JSON for further processing
          terraform show -json tfplan.binary > tfplan.json
          
          echo "PLAN_EXIT_CODE=$?" >> $GITHUB_ENV
        continue-on-error: true
      
      - name: Post Plan to PR
        if: always()
        uses: actions/github-script@v7
        with:
          script: |
            const fs = require('fs');
            const planOutput = fs.readFileSync('plan-output.txt', 'utf8');
            
            // Truncate ถ้ายาวเกินไป
            const maxLength = 60000;
            const truncatedPlan = planOutput.length > maxLength
              ? planOutput.substring(0, maxLength) + '\n... (truncated)'
              : planOutput;
            
            const body = `## 📋 Terraform Plan

            <details>
            <summary>Show Plan Output</summary>

            \`\`\`hcl
            ${truncatedPlan}
            \`\`\`

            </details>

            **Plan Status**: ${process.env.PLAN_EXIT_CODE === '0' ? '✅ Success' : '❌ Failed'}
            `;
            
            await github.rest.issues.createComment({
              issue_number: context.issue.number,
              owner: context.repo.owner,
              repo: context.repo.repo,
              body
            });
      
      - name: Upload Plan Artifact
        uses: actions/upload-artifact@v4
        with:
          name: terraform-plan
          path: |
            tfplan.binary
            tfplan.json
          retention-days: 30
  
  # ====================================================
  # Job 8: Security Summary Report
  # ====================================================
  security-summary:
    name: "Security Summary"
    runs-on: ubuntu-latest
    needs: [tfsec, checkov, snyk]
    if: always() && github.event_name == 'pull_request'
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Create Security Summary
        uses: actions/github-script@v7
        with:
          script: |
            const tfsecStatus = '${{ needs.tfsec.result }}';
            const checkovStatus = '${{ needs.checkov.result }}';
            const snykStatus = '${{ needs.snyk.result }}';
            
            const icon = (status) => status === 'success' ? '✅' : 
                                     status === 'failure' ? '❌' : '⚠️';
            
            const body = `## 🔒 Security Scan Summary

            | Tool | Status | Details |
            |------|--------|---------|
            | TFSec/Trivy | ${icon(tfsecStatus)} ${tfsecStatus} | [View in Security tab](https://github.com/${{ github.repository }}/security/code-scanning) |
            | Checkov | ${icon(checkovStatus)} ${checkovStatus} | [View in Security tab](https://github.com/${{ github.repository }}/security/code-scanning) |
            | Snyk IaC | ${icon(snykStatus)} ${snykStatus} | [View in Snyk Dashboard](https://app.snyk.io) |

            **Merge Policy**:
            - 🚫 Critical findings: **Blocked**
            - ⚠️ High findings: **Review required**  
            - 📋 Medium/Low findings: **FYI only**
            `;
            
            await github.rest.issues.createComment({
              issue_number: context.issue.number,
              owner: context.repo.owner,
              repo: context.repo.repo,
              body
            });
```

---

## Step 944: GitLab CI Pipeline

### .gitlab-ci.yml

```yaml
# .gitlab-ci.yml
stages:
  - validate
  - security
  - plan
  - apply
  - notify

variables:
  TF_VERSION: "1.7.0"
  TF_ROOT: ${CI_PROJECT_DIR}
  TF_STATE_NAME: ${CI_PROJECT_NAME}

# ===== Templates =====
.terraform_setup: &terraform_setup
  image:
    name: hashicorp/terraform:${TF_VERSION}
    entrypoint: [""]
  before_script:
    - terraform --version
    - cd ${TF_ROOT}
    - terraform init -backend=true

# ====================================================
# Stage 1: Validate
# ====================================================

terraform-fmt:
  stage: validate
  image:
    name: hashicorp/terraform:${TF_VERSION}
    entrypoint: [""]
  script:
    - terraform fmt -check -recursive -diff
  rules:
    - if: '$CI_PIPELINE_SOURCE == "merge_request_event"'
    - if: '$CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH'

terraform-validate:
  stage: validate
  <<: *terraform_setup
  script:
    - terraform init -backend=false
    - terraform validate -no-color
  rules:
    - if: '$CI_PIPELINE_SOURCE == "merge_request_event"'
    - if: '$CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH'

# ====================================================
# Stage 2: Security
# ====================================================

tfsec:
  stage: security
  image: aquasec/tfsec:latest
  script:
    - tfsec . --minimum-severity HIGH --format sarif > tfsec-results.sarif
    - tfsec . --minimum-severity HIGH --format text
    - |
      # Check for critical findings
      CRITICAL=$(tfsec . --format json 2>/dev/null | jq '[.results[] | select(.severity == "CRITICAL")] | length')
      if [ "$CRITICAL" -gt "0" ]; then
        echo "CRITICAL: $CRITICAL critical findings found!"
        exit 1
      fi
  artifacts:
    reports:
      sast: tfsec-results.sarif
    paths:
      - tfsec-results.sarif
    expire_in: 30 days
  rules:
    - if: '$CI_PIPELINE_SOURCE == "merge_request_event"'
    - if: '$CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH'

checkov:
  stage: security
  image: bridgecrew/checkov:latest
  script:
    - |
      checkov \
        -d . \
        --framework terraform \
        --output sarif \
        --output-file checkov-results.sarif \
        --compact \
        --quiet
    - |
      checkov \
        -d . \
        --framework terraform \
        --compact \
        --quiet
  artifacts:
    reports:
      sast: checkov-results.sarif
    paths:
      - checkov-results.sarif
    expire_in: 30 days
  rules:
    - if: '$CI_PIPELINE_SOURCE == "merge_request_event"'
    - if: '$CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH'

snyk-iac:
  stage: security
  image: snyk/snyk:latest
  variables:
    SNYK_TOKEN: $SNYK_TOKEN
  script:
    - snyk iac test . --json > snyk-report.json || true
    - snyk iac test . --severity-threshold=high
  artifacts:
    reports:
      sast: snyk-report.json
    paths:
      - snyk-report.json
    expire_in: 30 days
  rules:
    - if: '$CI_PIPELINE_SOURCE == "merge_request_event"'
    - if: '$CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH'

# ====================================================
# Stage 3: Plan
# ====================================================

terraform-plan:
  stage: plan
  <<: *terraform_setup
  script:
    - terraform plan -no-color -out tfplan.binary
    - terraform show -json tfplan.binary > tfplan.json
  artifacts:
    paths:
      - ${TF_ROOT}/tfplan.binary
      - ${TF_ROOT}/tfplan.json
    expire_in: 1 week
  rules:
    - if: '$CI_PIPELINE_SOURCE == "merge_request_event"'
  environment:
    name: review/$CI_COMMIT_REF_NAME

# ====================================================
# Stage 4: Apply (manual trigger)
# ====================================================

terraform-apply:
  stage: apply
  <<: *terraform_setup
  dependencies:
    - terraform-plan
  script:
    - terraform apply -no-color -auto-approve tfplan.binary
  rules:
    - if: '$CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH'
      when: manual
  environment:
    name: production

# ====================================================
# Stage 5: Notifications
# ====================================================

notify-slack:
  stage: notify
  image: curlimages/curl:latest
  script:
    - |
      if [ "$CI_JOB_STATUS" == "failed" ]; then
        curl -X POST \
          -H 'Content-type: application/json' \
          --data "{
            \"text\": \"🚨 Security scan failed on branch \`$CI_COMMIT_BRANCH\`\",
            \"attachments\": [{
              \"color\": \"danger\",
              \"fields\": [
                {\"title\": \"Project\", \"value\": \"$CI_PROJECT_NAME\", \"short\": true},
                {\"title\": \"Branch\", \"value\": \"$CI_COMMIT_BRANCH\", \"short\": true},
                {\"title\": \"Pipeline\", \"value\": \"$CI_PIPELINE_URL\", \"short\": false}
              ]
            }]
          }" \
          $SLACK_WEBHOOK_URL
      fi
  rules:
    - when: on_failure
```

---

## Step 945: Jenkins Pipeline

### Jenkinsfile

```groovy
// Jenkinsfile
pipeline {
    agent {
        kubernetes {
            yaml """
apiVersion: v1
kind: Pod
spec:
  containers:
  - name: terraform
    image: hashicorp/terraform:1.7.0
    command: ['cat']
    tty: true
  - name: checkov
    image: bridgecrew/checkov:latest
    command: ['cat']
    tty: true
  - name: tfsec
    image: aquasec/tfsec:latest
    command: ['cat']
    tty: true
"""
        }
    }
    
    environment {
        AWS_DEFAULT_REGION = 'ap-southeast-1'
        SLACK_CHANNEL = '#terraform-alerts'
    }
    
    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }
        
        stage('Terraform Format') {
            steps {
                container('terraform') {
                    sh 'terraform fmt -check -recursive -diff'
                }
            }
        }
        
        stage('Terraform Validate') {
            steps {
                container('terraform') {
                    sh '''
                        terraform init -backend=false
                        terraform validate -no-color
                    '''
                }
            }
        }
        
        stage('Security Scans') {
            parallel {
                stage('TFSec') {
                    steps {
                        container('tfsec') {
                            script {
                                sh '''
                                    tfsec . \
                                      --format sarif \
                                      --out tfsec-results.sarif \
                                      --minimum-severity HIGH \
                                      || true
                                '''
                                
                                def tfsecOutput = sh(
                                    script: 'tfsec . --format json --minimum-severity CRITICAL',
                                    returnStdout: true
                                ).trim()
                                
                                def results = readJSON text: tfsecOutput
                                def criticalCount = results.results?.count { it.severity == 'CRITICAL' } ?: 0
                                
                                if (criticalCount > 0) {
                                    error("TFSec found ${criticalCount} CRITICAL findings!")
                                }
                            }
                        }
                    }
                    post {
                        always {
                            archiveArtifacts artifacts: 'tfsec-results.sarif', allowEmptyArchive: true
                        }
                    }
                }
                
                stage('Checkov') {
                    steps {
                        container('checkov') {
                            script {
                                def exitCode = sh(
                                    script: '''
                                        checkov \
                                          -d . \
                                          --framework terraform \
                                          --output json \
                                          --output-file checkov-results.json \
                                          --compact \
                                          --quiet
                                    ''',
                                    returnStatus: true
                                )
                                
                                if (exitCode != 0) {
                                    def report = readJSON file: 'checkov-results.json'
                                    def failedChecks = report.results?.failed_checks?.size() ?: 0
                                    
                                    unstable("Checkov found ${failedChecks} failed checks")
                                }
                            }
                        }
                    }
                    post {
                        always {
                            archiveArtifacts artifacts: 'checkov-results.json', allowEmptyArchive: true
                        }
                    }
                }
            }
        }
        
        stage('Terraform Plan') {
            when {
                anyOf {
                    changeRequest()
                    branch 'main'
                }
            }
            steps {
                container('terraform') {
                    withCredentials([
                        string(credentialsId: 'aws-access-key', variable: 'AWS_ACCESS_KEY_ID'),
                        string(credentialsId: 'aws-secret-key', variable: 'AWS_SECRET_ACCESS_KEY')
                    ]) {
                        sh '''
                            terraform init
                            terraform plan -no-color -out tfplan.binary 2>&1 | tee plan-output.txt
                            terraform show -json tfplan.binary > tfplan.json
                        '''
                    }
                }
            }
            post {
                always {
                    archiveArtifacts artifacts: 'tfplan.json,plan-output.txt', allowEmptyArchive: true
                }
            }
        }
        
        stage('Apply') {
            when {
                allOf {
                    branch 'main'
                    not { changeRequest() }
                }
            }
            input {
                message "Deploy to production?"
                ok "Deploy"
                parameters {
                    string(name: 'REASON', description: 'Reason for deployment', defaultValue: '')
                }
            }
            steps {
                container('terraform') {
                    withCredentials([
                        string(credentialsId: 'aws-access-key', variable: 'AWS_ACCESS_KEY_ID'),
                        string(credentialsId: 'aws-secret-key', variable: 'AWS_SECRET_ACCESS_KEY')
                    ]) {
                        sh 'terraform apply -no-color -auto-approve tfplan.binary'
                    }
                }
            }
        }
    }
    
    post {
        failure {
            slackSend(
                channel: env.SLACK_CHANNEL,
                color: 'danger',
                message: """
🚨 *Pipeline Failed*
*Project*: ${env.JOB_NAME}
*Branch*: ${env.BRANCH_NAME}
*Build*: <${env.BUILD_URL}|#${env.BUILD_NUMBER}>
"""
            )
        }
        success {
            slackSend(
                channel: env.SLACK_CHANNEL,
                color: 'good',
                message: """
✅ *Pipeline Passed*
*Project*: ${env.JOB_NAME}
*Branch*: ${env.BRANCH_NAME}
"""
            )
        }
    }
}
```

---

## Step 946: SARIF Upload และ GitHub Code Scanning

### SARIF Format

```json
// ตัวอย่าง SARIF file
{
  "version": "2.1.0",
  "$schema": "https://json.schemastore.org/sarif-2.1.0.json",
  "runs": [
    {
      "tool": {
        "driver": {
          "name": "tfsec",
          "version": "1.28.5",
          "informationUri": "https://aquasecurity.github.io/tfsec",
          "rules": [
            {
              "id": "AVD-AWS-0089",
              "name": "BucketEncryptionEnabled",
              "shortDescription": {
                "text": "Bucket does not have encryption enabled"
              },
              "defaultConfiguration": {
                "level": "error"
              }
            }
          ]
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
                  "uri": "terraform/s3.tf",
                  "uriBaseId": "%SRCROOT%"
                },
                "region": {
                  "startLine": 1,
                  "endLine": 5
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

### Upload SARIF จาก Multiple Tools

```yaml
# Merge multiple SARIF files
- name: Merge SARIF Files
  run: |
    # ติดตั้ง sarif-multitool
    npm install -g @microsoft/sarif-multitool
    
    # Merge ทุก SARIF ไฟล์
    sarif merge \
      tfsec-results.sarif \
      checkov-results.sarif \
      snyk.sarif \
      --output merged-results.sarif

- name: Upload Merged SARIF
  uses: github/codeql-action/upload-sarif@v3
  with:
    sarif_file: merged-results.sarif
    category: 'terraform-security'
```

---

## Step 947: Slack Notifications สำหรับ Critical Findings

### Slack Webhook Integration

```yaml
# GitHub Actions - Slack Notification
- name: Send Slack Notification
  if: failure() || contains(steps.tfsec.outputs.critical, 'true')
  uses: 8398a7/action-slack@v3
  with:
    status: ${{ job.status }}
    fields: repo,message,commit,author,eventName,workflow
    custom_payload: |
      {
        "attachments": [{
          "color": "${{ job.status == 'success' && 'good' || 'danger' }}",
          "blocks": [
            {
              "type": "header",
              "text": {
                "type": "plain_text",
                "text": "🚨 Security Scan Alert"
              }
            },
            {
              "type": "section",
              "fields": [
                {
                  "type": "mrkdwn",
                  "text": "*Repository:*\n${{ github.repository }}"
                },
                {
                  "type": "mrkdwn",
                  "text": "*Branch:*\n${{ github.ref_name }}"
                },
                {
                  "type": "mrkdwn",
                  "text": "*Critical Findings:*\n${{ env.CRITICAL_COUNT }}"
                },
                {
                  "type": "mrkdwn",
                  "text": "*High Findings:*\n${{ env.HIGH_COUNT }}"
                }
              ]
            },
            {
              "type": "actions",
              "elements": [
                {
                  "type": "button",
                  "text": {"type": "plain_text", "text": "View PR"},
                  "url": "${{ github.event.pull_request.html_url }}"
                },
                {
                  "type": "button",
                  "text": {"type": "plain_text", "text": "View Security Tab"},
                  "url": "https://github.com/${{ github.repository }}/security/code-scanning"
                }
              ]
            }
          ]
        }]
      }
  env:
    SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK_URL }}
```

---

## Step 948: PR-based Security Gates

### GitHub Branch Protection Rules

```
GitHub Repository Settings > Branches > Branch protection rules:

✅ Require status checks to pass before merging:
   - terraform-fmt
   - terraform-validate
   - tfsec
   - checkov
   
✅ Require a pull request before merging
✅ Require approvals: 1
✅ Dismiss stale pull request approvals when new commits are pushed
✅ Require review from Code Owners
✅ Require signed commits
```

### CODEOWNERS File

```
# .github/CODEOWNERS
# Security-related files require security team review

# Terraform files
*.tf @platform-team @security-team

# Security configurations
.tfsec/ @security-team
.checkov.yaml @security-team
.snyk @security-team
custom_checks/ @security-team

# CI/CD workflows
.github/workflows/ @platform-team @security-team

# Production environment
environments/prod/ @security-team @sre-team
```

### Auto-comment Security Findings

```yaml
# Action ที่ comment detailed findings ลงบน PR
- name: Comment Security Findings on PR
  uses: actions/github-script@v7
  if: github.event_name == 'pull_request'
  with:
    script: |
      const fs = require('fs');
      
      // Read TFSec JSON results
      let tfsecResults = { results: [] };
      try {
        tfsecResults = JSON.parse(fs.readFileSync('tfsec-output.json', 'utf8'));
      } catch(e) {}
      
      // Filter critical and high
      const critical = tfsecResults.results?.filter(r => r.severity === 'CRITICAL') || [];
      const high = tfsecResults.results?.filter(r => r.severity === 'HIGH') || [];
      
      if (critical.length === 0 && high.length === 0) return;
      
      // Build comment
      let body = '## 🔒 Security Findings Requiring Attention\n\n';
      
      if (critical.length > 0) {
        body += '### 🚨 CRITICAL Findings (Must Fix)\n\n';
        body += '| Rule | Resource | File | Line |\n';
        body += '|------|----------|------|------|\n';
        for (const finding of critical) {
          body += `| \`${finding.rule_id}\` | \`${finding.resource}\` | ${finding.location.filename} | ${finding.location.start_line} |\n`;
        }
        body += '\n';
      }
      
      if (high.length > 0) {
        body += '### ⚠️ HIGH Findings (Should Fix)\n\n';
        body += '| Rule | Resource | File | Line |\n';
        body += '|------|----------|------|------|\n';
        for (const finding of high) {
          body += `| \`${finding.rule_id}\` | \`${finding.resource}\` | ${finding.location.filename} | ${finding.location.start_line} |\n`;
        }
      }
      
      body += '\n> 💡 Tip: View detailed findings in the [Security tab](https://github.com/${{ github.repository }}/security/code-scanning)\n';
      
      // Delete previous bot comments
      const comments = await github.rest.issues.listComments({
        issue_number: context.issue.number,
        owner: context.repo.owner,
        repo: context.repo.repo,
      });
      
      for (const comment of comments.data) {
        if (comment.user.type === 'Bot' && comment.body.includes('Security Findings Requiring Attention')) {
          await github.rest.issues.deleteComment({
            owner: context.repo.owner,
            repo: context.repo.repo,
            comment_id: comment.id,
          });
        }
      }
      
      // Create new comment
      await github.rest.issues.createComment({
        issue_number: context.issue.number,
        owner: context.repo.owner,
        repo: context.repo.repo,
        body
      });
```

---

## Step 949: Compliance as Code Pipeline

### Scheduled Compliance Check

```yaml
# .github/workflows/compliance-check.yml
name: Scheduled Compliance Check

on:
  schedule:
    # ทุกวันจันทร์ 9am UTC
    - cron: '0 9 * * 1'
  workflow_dispatch:  # Manual trigger

jobs:
  compliance-check:
    name: Weekly Compliance Scan
    runs-on: ubuntu-latest
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Run Full Security Suite
        run: |
          # TFSec - All severities
          docker run --rm -v $(pwd):/src aquasec/tfsec:latest \
            --format json \
            --minimum-severity LOW \
            /src > tfsec-full.json || true
          
          # Checkov - All frameworks
          checkov \
            -d . \
            --output json \
            --compact \
            > checkov-full.json || true
          
          # Terrascan
          docker run --rm -v $(pwd):/workspace \
            tenable/terrascan:latest \
            scan -t aws --output json \
            > terrascan-full.json || true
      
      - name: Generate Compliance Report
        run: |
          python3 << 'EOF'
          import json
          from datetime import datetime
          
          report = {
              "timestamp": datetime.now().isoformat(),
              "repository": "${{ github.repository }}",
              "branch": "${{ github.ref_name }}",
              "tools": {}
          }
          
          # Process TFSec
          try:
              with open('tfsec-full.json') as f:
                  tfsec = json.load(f)
              report["tools"]["tfsec"] = {
                  "critical": len([r for r in tfsec.get("results", []) if r.get("severity") == "CRITICAL"]),
                  "high": len([r for r in tfsec.get("results", []) if r.get("severity") == "HIGH"]),
                  "medium": len([r for r in tfsec.get("results", []) if r.get("severity") == "MEDIUM"]),
                  "low": len([r for r in tfsec.get("results", []) if r.get("severity") == "LOW"]),
              }
          except:
              report["tools"]["tfsec"] = {"error": "Failed to parse results"}
          
          print(json.dumps(report, indent=2))
          
          with open('compliance-report.json', 'w') as f:
              json.dump(report, f, indent=2)
          EOF
      
      - name: Upload Compliance Report
        uses: actions/upload-artifact@v4
        with:
          name: compliance-report-${{ github.run_number }}
          path: compliance-report.json
          retention-days: 90  # เก็บ 3 เดือน
      
      - name: Send Weekly Report to Slack
        run: |
          python3 << 'EOF'
          import json
          import urllib.request
          import os
          
          with open('compliance-report.json') as f:
              report = json.load(f)
          
          tfsec = report.get("tools", {}).get("tfsec", {})
          
          message = {
              "text": f"📊 Weekly Terraform Security Compliance Report",
              "attachments": [{
                  "color": "danger" if tfsec.get("critical", 0) > 0 else "good",
                  "fields": [
                      {"title": "🔴 Critical", "value": str(tfsec.get("critical", 0)), "short": True},
                      {"title": "🟠 High", "value": str(tfsec.get("high", 0)), "short": True},
                      {"title": "🟡 Medium", "value": str(tfsec.get("medium", 0)), "short": True},
                      {"title": "🔵 Low", "value": str(tfsec.get("low", 0)), "short": True},
                  ]
              }]
          }
          
          webhook_url = os.environ.get('SLACK_WEBHOOK_URL', '')
          if webhook_url:
              data = json.dumps(message).encode('utf-8')
              req = urllib.request.Request(webhook_url, data=data, 
                                          headers={'Content-Type': 'application/json'})
              urllib.request.urlopen(req)
          EOF
        env:
          SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK_URL }}
```

---

## Step 950: Jira Integration

### สร้าง Jira Ticket อัตโนมัติ

```yaml
- name: Create Jira Tickets for Critical Findings
  if: env.CRITICAL_COUNT != '0'
  uses: atlassian/gajira-create@v3
  with:
    project: SEC
    issuetype: Bug
    summary: "Critical Security Finding in ${{ github.repository }}"
    description: |
      **Critical security findings detected** in Terraform code.
      
      **Repository**: ${{ github.repository }}
      **Branch**: ${{ github.ref_name }}
      **PR**: ${{ github.event.pull_request.html_url }}
      
      **Critical Count**: ${{ env.CRITICAL_COUNT }}
      
      Please review the [PR Security tab](${{ github.event.pull_request.html_url }}) 
      and the [GitHub Security tab](https://github.com/${{ github.repository }}/security/code-scanning)
      for detailed findings.
      
      **Action Required**: Fix all critical findings before merging.
    labels: security,terraform,critical
    priority: Highest
  env:
    JIRA_BASE_URL: ${{ secrets.JIRA_BASE_URL }}
    JIRA_USER_EMAIL: ${{ secrets.JIRA_USER_EMAIL }}
    JIRA_API_TOKEN: ${{ secrets.JIRA_API_TOKEN }}
```

---

## สรุป Security Pipeline Checklist

```
Pre-commit (local):
☐ terraform fmt --check
☐ terraform validate
☐ tflint
☐ tfsec --minimum-severity HIGH
☐ detect-secrets
☐ gitleaks

CI/CD Pipeline (PR):
☐ terraform fmt --check
☐ terraform validate
☐ tflint (lint check)
☐ TFSec/Trivy (SARIF upload)
☐ Checkov (SARIF upload)
☐ Snyk IaC (if budget allows)
☐ terraform plan (comment on PR)
☐ Security summary comment
☐ Block on CRITICAL findings
☐ Notify on HIGH findings

Post-merge:
☐ terraform apply
☐ Drift detection
☐ Runtime monitoring

Weekly:
☐ Full compliance scan
☐ Compliance report generation
☐ Slack/email weekly digest
```

---

*จบ Part 95: Security CI/CD Integration*
