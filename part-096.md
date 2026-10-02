# Part 96: GitOps with Terraform (Steps 951-960)

## การนำ GitOps มาใช้กับ Terraform

---

## Step 951: GitOps คืออะไร?

### ความหมายของ GitOps

**GitOps** เป็นแนวทาง (methodology) สำหรับจัดการ infrastructure และ application deployments โดยใช้ **Git เป็น single source of truth** และทำให้ทุกอย่างเป็น declarative

### หลักการ GitOps (The 4 Principles)

```
1. DECLARATIVE
   ─────────────
   ระบุ desired state ไม่ใช่ steps
   Terraform declarative อยู่แล้ว ✅

2. VERSIONED & IMMUTABLE  
   ─────────────
   ทุก change เก็บใน Git
   History สมบูรณ์
   Rollback ทำได้ง่าย

3. PULLED AUTOMATICALLY
   ─────────────
   System ดึง state จาก Git (pull-based)
   ไม่ใช่ push จากนอก (push-based)
   
4. CONTINUOUSLY RECONCILED
   ─────────────
   ตรวจสอบ actual state vs desired state ตลอดเวลา
   แก้ไข drift อัตโนมัติ
```

### GitOps vs Traditional Operations

```
Traditional (Push-based):
  Developer → CI/CD Server → terraform apply → Cloud

GitOps (Pull-based):
  Developer → Git PR → Review → Merge → Operator watches Git → terraform apply → Cloud
              ↑                                              ↑
         Code Review                                  Automated & Auditable
```

---

## Step 952: GitOps สำหรับ Terraform

### Git เป็น Single Source of Truth

```
Git Repository ประกอบด้วย:
┌─────────────────────────────────────────────────────────────┐
│ ✅ Terraform code (.tf files)                               │
│ ✅ Variable values (non-sensitive, via .tfvars)             │
│ ✅ State backend configuration                              │
│ ✅ Module references และ versions                           │
│ ✅ CI/CD pipeline configuration                             │
│ ✅ Security policies                                        │
│ ✅ Documentation                                            │
│                                                             │
│ ❌ Secrets และ passwords (ใช้ Vault/Secrets Manager)       │
│ ❌ terraform.tfstate (ใช้ remote backend)                  │
│ ❌ .terraform/ directory                                    │
└─────────────────────────────────────────────────────────────┘
```

### Repository Structure สำหรับ GitOps

```
infrastructure/
├── .github/
│   ├── workflows/
│   │   ├── pr.yml           # PR validation + plan
│   │   ├── apply.yml        # Apply on merge to main
│   │   └── drift.yml        # Scheduled drift detection
│   └── CODEOWNERS
├── environments/
│   ├── dev/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   ├── outputs.tf
│   │   └── terraform.tfvars
│   ├── staging/
│   │   └── ...
│   └── prod/
│       └── ...
├── modules/
│   ├── networking/
│   ├── compute/
│   └── database/
├── policies/
│   ├── checkov/
│   └── sentinel/
├── .tfsec/
├── .pre-commit-config.yaml
├── .gitignore
└── README.md
```

---

## Step 953: GitOps Workflow สำหรับ Infrastructure

### End-to-End Workflow

```
GitOps Infrastructure Workflow:
─────────────────────────────────────────────────────────────

Step 1: Developer สร้าง Feature Branch
  git checkout -b feature/add-new-rds-instance

Step 2: เขียน Terraform Code
  vim environments/dev/main.tf

Step 3: Local Validation (Pre-commit)
  pre-commit run --all-files
  terraform fmt
  terraform validate

Step 4: สร้าง PR
  git push origin feature/add-new-rds-instance
  gh pr create --title "Add RDS instance for API service"

Step 5: Automated CI Checks (PR Validation)
  - terraform fmt --check ✅
  - terraform validate ✅
  - TFSec/Checkov security scan ✅
  - terraform plan (post as PR comment) ✅

Step 6: Human Review
  - Review terraform plan output
  - Review security scan results
  - Approve PR

Step 7: Merge to main
  → Triggers automated apply

Step 8: Automated Apply
  - terraform apply (auto-approved, plan already reviewed)
  - Notify on Slack

Step 9: Verification
  - Smoke tests
  - Drift detection check
```

---

## Step 954: Terraform GitOps Tools Comparison

### ตารางเปรียบเทียบ GitOps Tools

```
┌─────────────────┬──────────┬──────────┬──────────┬──────────┬──────────┐
│ Feature         │ Atlantis │ Env0     │ Scalr    │Spacelift │ TFC/TFE  │
├─────────────────┼──────────┼──────────┼──────────┼──────────┼──────────┤
│ Open Source     │ ✅ Free  │ ❌ Paid  │ ❌ Paid  │ ❌ Paid  │ ❌ Paid  │
│ Self-hosted     │ ✅       │ ✅       │ ✅       │ ✅       │ TFE only │
│ SaaS option     │ ❌       │ ✅       │ ✅       │ ✅       │ ✅ (TFC) │
│ PR automation   │ ✅       │ ✅       │ ✅       │ ✅       │ ✅       │
│ Policy engine   │ OPA      │ OPA/Rego │ OPA/Rego │Rego/OPA  │ Sentinel │
│ RBAC            │ Basic    │ Advanced │ Advanced │ Advanced │ Advanced │
│ Cost estimation │ ❌       │ ✅       │ ✅       │ ✅       │ ❌       │
│ Drift detection │ ❌       │ ✅       │ ✅       │ ✅       │ ❌       │
│ Module registry │ ❌       │ ✅       │ ✅       │ ✅       │ ✅       │
│ Setup complexity│ Medium   │ Easy     │ Easy     │ Easy     │ Easy     │
├─────────────────┼──────────┼──────────┼──────────┼──────────┼──────────┤
│ Best for        │ Teams    │SMB/Mid   │Enterprise│Enterprise│Enterprise│
│                 │ on budget│ market   │          │          │          │
└─────────────────┴──────────┴──────────┴──────────┴──────────┴──────────┘
```

---

## Step 955: GitHub Actions GitOps Pipeline (DIY)

### PR Workflow

```yaml
# .github/workflows/pr.yml
name: PR Validation & Plan

on:
  pull_request:
    branches: [main]
    paths:
      - 'environments/**'
      - 'modules/**'

permissions:
  contents: read
  pull-requests: write
  id-token: write

env:
  TF_VERSION: "1.7.0"
  AWS_REGION: "ap-southeast-1"

jobs:
  # ===================================================
  # สแกนทั้ง repository แต่ plan เฉพาะที่เปลี่ยน
  # ===================================================
  detect-changes:
    name: Detect Changed Environments
    runs-on: ubuntu-latest
    outputs:
      environments: ${{ steps.detect.outputs.environments }}
    
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0  # Need full history for diff
      
      - name: Detect Changed Terraform Environments
        id: detect
        run: |
          # หา directories ที่มีการเปลี่ยนแปลง
          CHANGED_DIRS=$(git diff --name-only origin/${{ github.base_ref }}...HEAD \
            | grep -E '^environments/' \
            | cut -d'/' -f1-2 \
            | sort -u \
            | jq -R -s 'split("\n") | map(select(. != ""))')
          
          echo "Changed environments: $CHANGED_DIRS"
          echo "environments=$CHANGED_DIRS" >> $GITHUB_OUTPUT
  
  # ===================================================
  # Security Scan
  # ===================================================
  security-scan:
    name: Security Scan
    runs-on: ubuntu-latest
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Run TFSec
        uses: aquasecurity/tfsec-action@v1.0.0
        with:
          format: 'sarif'
          output: 'tfsec-results.sarif'
          soft_fail: 'true'
          additional_args: '--minimum-severity MEDIUM'
      
      - name: Upload SARIF
        uses: github/codeql-action/upload-sarif@v3
        if: always()
        with:
          sarif_file: tfsec-results.sarif
          category: tfsec
      
      - name: Run Checkov
        uses: bridgecrewio/checkov-action@master
        with:
          directory: environments/
          framework: terraform
          soft_fail: true
          output_format: sarif
          output_file_path: checkov-results.sarif
      
      - name: Upload Checkov SARIF
        uses: github/codeql-action/upload-sarif@v3
        if: always()
        with:
          sarif_file: checkov-results.sarif
          category: checkov
      
      - name: Check Critical Security Issues
        run: |
          # Run again with strict exit code
          docker run --rm -v $(pwd):/src aquasec/tfsec:latest \
            --minimum-severity CRITICAL \
            --exit-code 1 \
            /src
  
  # ===================================================
  # Terraform Plan สำหรับแต่ละ environment
  # ===================================================
  terraform-plan:
    name: "Plan: ${{ matrix.environment }}"
    runs-on: ubuntu-latest
    needs: [security-scan, detect-changes]
    if: needs.detect-changes.outputs.environments != '[]'
    
    strategy:
      matrix:
        environment: ${{ fromJSON(needs.detect-changes.outputs.environments) }}
      fail-fast: false  # Plan ทุก environment แม้มี failure
    
    environment: plan-${{ matrix.environment }}
    
    steps:
      - uses: actions/checkout@v4
      
      - uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: ${{ env.TF_VERSION }}
      
      - name: Configure AWS Credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ secrets[format('AWS_ROLE_{0}', matrix.environment)] }}
          aws-region: ${{ env.AWS_REGION }}
      
      - name: Terraform Init
        working-directory: ${{ matrix.environment }}
        run: |
          terraform init \
            -backend-config="key=${{ matrix.environment }}/terraform.tfstate"
      
      - name: Terraform Plan
        id: plan
        working-directory: ${{ matrix.environment }}
        run: |
          set +e
          terraform plan \
            -no-color \
            -input=false \
            -out=tfplan.binary \
            2>&1 | tee plan-output.txt
          
          PLAN_EXIT_CODE=$?
          echo "exit_code=$PLAN_EXIT_CODE" >> $GITHUB_OUTPUT
          
          # สร้าง plan summary
          ADDED=$(grep -c "will be created" plan-output.txt 2>/dev/null || echo "0")
          CHANGED=$(grep -c "will be updated in-place" plan-output.txt 2>/dev/null || echo "0")
          DESTROYED=$(grep -c "will be destroyed" plan-output.txt 2>/dev/null || echo "0")
          
          echo "added=$ADDED" >> $GITHUB_OUTPUT
          echo "changed=$CHANGED" >> $GITHUB_OUTPUT
          echo "destroyed=$DESTROYED" >> $GITHUB_OUTPUT
          
          exit $PLAN_EXIT_CODE
        continue-on-error: true
      
      - name: Post Plan Comment
        uses: actions/github-script@v7
        with:
          script: |
            const environment = '${{ matrix.environment }}';
            const added = '${{ steps.plan.outputs.added }}';
            const changed = '${{ steps.plan.outputs.changed }}';
            const destroyed = '${{ steps.plan.outputs.destroyed }}';
            const exitCode = '${{ steps.plan.outputs.exit_code }}';
            
            const fs = require('fs');
            const planOutput = fs.readFileSync('${{ matrix.environment }}/plan-output.txt', 'utf8');
            const maxLength = 50000;
            const truncatedPlan = planOutput.length > maxLength 
              ? planOutput.substring(0, maxLength) + '\n... (truncated - see full plan in artifacts)'
              : planOutput;
            
            const status = exitCode === '0' ? '✅' : '❌';
            const destructiveWarning = parseInt(destroyed) > 0 
              ? `\n> ⚠️ **Warning**: ${destroyed} resource(s) will be **DESTROYED**!`
              : '';
            
            const body = `## ${status} Terraform Plan: \`${environment}\`
            
            | Action | Count |
            |--------|-------|
            | ➕ Add | ${added} |
            | 🔄 Change | ${changed} |
            | 💥 Destroy | ${destroyed} |
            ${destructiveWarning}
            
            <details>
            <summary>📋 Show Plan Output</summary>
            
            \`\`\`hcl
            ${truncatedPlan}
            \`\`\`
            
            </details>
            `;
            
            // ลบ comment เก่า
            const comments = await github.rest.issues.listComments({
              issue_number: context.issue.number,
              owner: context.repo.owner,
              repo: context.repo.repo
            });
            
            for (const comment of comments.data) {
              if (comment.body.includes(`Terraform Plan: \`${environment}\``)) {
                await github.rest.issues.deleteComment({
                  owner: context.repo.owner,
                  repo: context.repo.repo,
                  comment_id: comment.id
                });
              }
            }
            
            await github.rest.issues.createComment({
              issue_number: context.issue.number,
              owner: context.repo.owner,
              repo: context.repo.repo,
              body
            });
      
      - name: Upload Plan Artifacts
        uses: actions/upload-artifact@v4
        if: always()
        with:
          name: plan-${{ matrix.environment }}-${{ github.run_number }}
          path: |
            ${{ matrix.environment }}/tfplan.binary
            ${{ matrix.environment }}/plan-output.txt
          retention-days: 30
```

### Apply Workflow (Merge to Main)

```yaml
# .github/workflows/apply.yml
name: Terraform Apply (GitOps)

on:
  push:
    branches: [main]
    paths:
      - 'environments/**'
      - 'modules/**'

permissions:
  contents: read
  id-token: write
  deployments: write

env:
  TF_VERSION: "1.7.0"
  AWS_REGION: "ap-southeast-1"

jobs:
  # ===================================================
  # หา environments ที่เปลี่ยน
  # ===================================================
  detect-changes:
    name: Detect Changed Environments
    runs-on: ubuntu-latest
    outputs:
      environments: ${{ steps.detect.outputs.environments }}
    
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 2
      
      - name: Detect Changed Environments
        id: detect
        run: |
          CHANGED=$(git diff --name-only HEAD~1 HEAD \
            | grep -E '^environments/' \
            | cut -d'/' -f1-2 \
            | sort -u \
            | jq -R -s 'split("\n") | map(select(. != ""))')
          echo "environments=$CHANGED" >> $GITHUB_OUTPUT
  
  # ===================================================
  # Apply (ทีละ environment ตาม sequence)
  # ===================================================
  apply-dev:
    name: Apply Dev
    runs-on: ubuntu-latest
    needs: detect-changes
    if: contains(fromJSON(needs.detect-changes.outputs.environments), 'environments/dev')
    environment: production-dev
    
    steps:
      - uses: actions/checkout@v4
      
      - uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: ${{ env.TF_VERSION }}
      
      - name: Configure AWS Credentials (Dev)
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ secrets.AWS_ROLE_DEV }}
          aws-region: ${{ env.AWS_REGION }}
      
      - name: Terraform Init & Apply (Dev)
        working-directory: environments/dev
        run: |
          terraform init \
            -backend-config="key=dev/terraform.tfstate"
          
          terraform apply \
            -auto-approve \
            -input=false \
            -no-color \
            2>&1 | tee apply-output.txt
      
      - name: Notify Success
        if: success()
        uses: 8398a7/action-slack@v3
        with:
          status: success
          text: "✅ Dev environment deployed successfully!"
        env:
          SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK_URL }}
      
      - name: Notify Failure
        if: failure()
        uses: 8398a7/action-slack@v3
        with:
          status: failure
          text: "❌ Dev environment deployment FAILED!"
        env:
          SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK_URL }}
  
  apply-staging:
    name: Apply Staging
    runs-on: ubuntu-latest
    needs: apply-dev  # Sequential deployment
    if: |
      always() && 
      needs.apply-dev.result == 'success' && 
      contains(fromJSON(needs.detect-changes.outputs.environments), 'environments/staging')
    environment: production-staging
    
    steps:
      - uses: actions/checkout@v4
      # ... similar to dev but for staging
  
  apply-prod:
    name: Apply Production
    runs-on: ubuntu-latest
    needs: apply-staging
    if: |
      always() && 
      needs.apply-staging.result == 'success' &&
      contains(fromJSON(needs.detect-changes.outputs.environments), 'environments/prod')
    environment: production  # Requires manual approval!
    
    steps:
      - uses: actions/checkout@v4
      # ... similar but for production with extra safeguards
```

### Drift Detection Workflow

```yaml
# .github/workflows/drift-detection.yml
name: Drift Detection

on:
  schedule:
    # ทุกวัน 6am UTC
    - cron: '0 6 * * *'
  workflow_dispatch:

jobs:
  detect-drift:
    name: Detect Infrastructure Drift
    runs-on: ubuntu-latest
    
    strategy:
      matrix:
        environment: [dev, staging, prod]
    
    steps:
      - uses: actions/checkout@v4
      
      - uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: "1.7.0"
      
      - name: Configure AWS Credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ secrets[format('AWS_ROLE_{0}', matrix.environment)] }}
          aws-region: ap-southeast-1
      
      - name: Check for Drift
        id: drift
        working-directory: environments/${{ matrix.environment }}
        run: |
          terraform init -backend-config="key=${{ matrix.environment }}/terraform.tfstate"
          
          # plan -detailed-exitcode:
          # exit 0 = no changes
          # exit 1 = error
          # exit 2 = changes detected (drift!)
          set +e
          terraform plan \
            -detailed-exitcode \
            -no-color \
            -input=false \
            2>&1 | tee drift-output.txt
          
          DRIFT_CODE=$?
          echo "drift_code=$DRIFT_CODE" >> $GITHUB_OUTPUT
          echo "has_drift=$([ $DRIFT_CODE -eq 2 ] && echo 'true' || echo 'false')" >> $GITHUB_OUTPUT
      
      - name: Alert on Drift
        if: steps.drift.outputs.has_drift == 'true'
        run: |
          echo "::warning::Drift detected in ${{ matrix.environment }} environment!"
          
          # Send Slack alert
          DRIFT_SUMMARY=$(grep -E "^  [+~-]" drift-output.txt | head -20 || true)
          
          curl -X POST \
            -H 'Content-type: application/json' \
            --data "{
              \"text\": \"🚨 Infrastructure Drift Detected!\",
              \"attachments\": [{
                \"color\": \"danger\",
                \"fields\": [
                  {\"title\": \"Environment\", \"value\": \"${{ matrix.environment }}\", \"short\": true},
                  {\"title\": \"Repository\", \"value\": \"${{ github.repository }}\", \"short\": true},
                  {\"title\": \"Details\", \"value\": \"Changes found in infrastructure that don't match Terraform state\"}
                ]
              }]
            }" \
            ${{ secrets.SLACK_WEBHOOK_URL }}
      
      - name: Create Issue on Drift
        if: steps.drift.outputs.has_drift == 'true'
        uses: actions/github-script@v7
        with:
          script: |
            const fs = require('fs');
            const driftOutput = fs.readFileSync('environments/${{ matrix.environment }}/drift-output.txt', 'utf8');
            
            // สร้าง issue หรือ update issue ที่มีอยู่
            const issues = await github.rest.issues.listForRepo({
              owner: context.repo.owner,
              repo: context.repo.repo,
              labels: 'drift,${{ matrix.environment }}',
              state: 'open'
            });
            
            const body = `## 🚨 Infrastructure Drift Detected
            
            **Environment**: ${{ matrix.environment }}
            **Detected at**: ${new Date().toISOString()}
            
            Infrastructure in \`${{ matrix.environment }}\` has drifted from Terraform state.
            
            <details>
            <summary>Drift Details</summary>
            
            \`\`\`
            ${driftOutput.substring(0, 10000)}
            \`\`\`
            
            </details>
            
            **Resolution**: Run \`terraform apply\` for \`${{ matrix.environment }}\` to reconcile state.
            `;
            
            if (issues.data.length > 0) {
              // Update existing issue
              await github.rest.issues.createComment({
                owner: context.repo.owner,
                repo: context.repo.repo,
                issue_number: issues.data[0].number,
                body: `Drift still detected at ${new Date().toISOString()}`
              });
            } else {
              // Create new issue
              await github.rest.issues.create({
                owner: context.repo.owner,
                repo: context.repo.repo,
                title: `🚨 Infrastructure Drift: ${{ matrix.environment }}`,
                body,
                labels: ['drift', '${{ matrix.environment }}', 'infrastructure']
              });
            }
```

---

## Step 956: Branch Strategies

### Strategy 1: Environment Branches

```
Branch Strategy: Environment Branches
─────────────────────────────────────

main ─────────────────────────────────────── (production)
  │
  ├── staging ──────────────────────────── (staging)
  │
  └── develop ──────────────────────────── (dev)

Workflow:
  feature/* → develop (dev deploy)
  develop → staging (staging deploy)
  staging → main (production deploy, manual approval)

Pros:
  ✅ ชัดเจนว่า branch ไหน = environment ไหน
  ✅ ป้องกัน deploy production โดยไม่ผ่าน staging

Cons:
  ❌ Cherry-pick ยาก
  ❌ Long-lived branches = merge conflicts
```

### Strategy 2: Trunk-Based Development (แนะนำ)

```
Branch Strategy: Trunk-Based Development
─────────────────────────────────────────

main ──────────────────────────────────── (ALL environments)
  ↑
  feature/* (short-lived, < 2 days)

Deployment controlled by:
  - Variable: TF_VAR_environment = dev/staging/prod
  - Directory: environments/dev/, environments/prod/
  - Workspace: terraform workspace select prod

Workflow:
  feature/* → main PR (review + plan)
  Merge → Apply to dev
  Tag v1.2.3 → Apply to staging
  Manual approve tag → Apply to prod

Pros:
  ✅ Simple branching model
  ✅ ไม่มี long-lived branches
  ✅ Fast feedback loop

Cons:
  ❌ ต้องมี feature flags สำหรับ WIP features
```

### Strategy 3: Monorepo with Subdirectory

```
Monorepo Structure:
infrastructure/
  ├── environments/
  │   ├── dev/          → auto-deploy on merge
  │   ├── staging/      → auto-deploy on merge
  │   └── prod/         → manual approval required
  └── modules/          → shared modules

Path filters ใน GitHub Actions:
  - push to main + path changes in environments/dev/** → deploy dev
  - push to main + path changes in environments/staging/** → deploy staging
  - push to main + path changes in environments/prod/** → manual gate → deploy prod
```

---

## Step 957: PR Comments ด้วย terraform-comment

### ติดตั้ง terraform-comment

```bash
# ใช้ robburger/terraform-pr-commenter action
# หรือ hashicorp/setup-terraform ที่มี built-in plan comment
```

### Terraform PR Comment Action

```yaml
# ใช้ dflook/terraform-github-actions
- name: Terraform Plan
  uses: dflook/terraform-plan@v1
  id: plan
  with:
    path: environments/dev
    label: dev
    variables: |
      environment = "dev"
      aws_region  = "ap-southeast-1"
  env:
    AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
    AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
    GITHUB_TOKEN: ${{ github.token }}

# Plan จะถูก post เป็น PR comment อัตโนมัติ
```

---

## Step 958: Environment Promotion Workflow

### Promotion Pipeline

```yaml
# .github/workflows/promote.yml
name: Environment Promotion

on:
  workflow_dispatch:
    inputs:
      from_env:
        description: 'Source environment'
        required: true
        type: choice
        options: [dev, staging]
      to_env:
        description: 'Target environment'
        required: true
        type: choice
        options: [staging, prod]
      reason:
        description: 'Reason for promotion'
        required: true

jobs:
  promote:
    name: Promote ${{ inputs.from_env }} → ${{ inputs.to_env }}
    runs-on: ubuntu-latest
    environment: ${{ inputs.to_env }}  # Requires approval for prod
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Validate Promotion Path
        run: |
          # ห้าม promote dev → prod โดยตรง
          if [[ "${{ inputs.from_env }}" == "dev" && "${{ inputs.to_env }}" == "prod" ]]; then
            echo "Error: Cannot promote dev directly to prod. Must go through staging!"
            exit 1
          fi
      
      - name: Copy Configuration
        run: |
          # Copy variables from source to target
          # แต่เก็บ environment-specific values ไว้
          cp environments/${{ inputs.from_env }}/main.tf \
             environments/${{ inputs.to_env }}/main.tf
      
      - name: Create PR for Promotion
        uses: actions/github-script@v7
        with:
          script: |
            // สร้าง branch สำหรับ promotion
            const branch = `promote/${inputs.from_env}-to-${inputs.to_env}-${Date.now()}`;
            
            // Create PR
            const pr = await github.rest.pulls.create({
              owner: context.repo.owner,
              repo: context.repo.repo,
              title: `Promote ${inputs.from_env} → ${inputs.to_env}`,
              head: branch,
              base: 'main',
              body: `
              ## Environment Promotion
              
              **From**: ${inputs.from_env}
              **To**: ${inputs.to_env}
              **Reason**: ${inputs.reason}
              **Requested by**: ${context.actor}
              `
            });
```

---

## Step 959: Rollback Strategies

### Strategy 1: Git Revert

```bash
# Rollback: revert the problematic commit
git log --oneline -10
# a1b2c3d Apply: Add RDS instance (bad)
# e4f5g6h Apply: Update S3 policy (good)

# Revert ทันที
git revert a1b2c3d
git push origin main
# → triggers automatic apply with reverted state
```

### Strategy 2: Terraform State Manipulation

```bash
# Emergency: rollback ด้วย previous state
# (ใช้เมื่อ git revert ไม่เพียงพอ)

# Download state ก่อนหน้า
aws s3 cp s3://my-state-bucket/terraform.tfstate terraform.tfstate.current
aws s3 cp s3://my-state-bucket/terraform.tfstate.backup terraform.tfstate.previous

# ตรวจสอบความแตกต่าง
terraform state list  # ปัจจุบัน

# Plan ด้วย state เก่า
terraform plan -state=terraform.tfstate.previous

# Apply กลับ (Emergency only!)
terraform apply -state=terraform.tfstate.previous
```

### Strategy 3: Blue-Green Rollback

```hcl
# สลับ traffic กลับไปยัง blue environment
variable "active_environment" {
  description = "Active deployment environment (blue/green)"
  default     = "blue"  # เปลี่ยนเป็น "blue" เพื่อ rollback
}

resource "aws_lb_listener_rule" "active" {
  listener_arn = aws_lb_listener.main.arn
  
  condition {
    path_pattern {
      values = ["/*"]
    }
  }
  
  action {
    type             = "forward"
    target_group_arn = var.active_environment == "blue" \
      ? aws_lb_target_group.blue.arn \
      : aws_lb_target_group.green.arn
  }
}
```

---

## Step 960: Metrics และ Observability

### GitOps Metrics

```yaml
# GitHub Actions Metrics ที่ควรติดตาม:
metrics:
  - name: deployment_frequency
    description: กี่ครั้งต่อวันที่ deploy ไปแต่ละ environment
    query: count workflow runs with "apply" in main branch
    
  - name: lead_time_for_changes
    description: เวลาตั้งแต่ commit → production deploy
    target: < 1 day
    
  - name: change_failure_rate
    description: % ของ deployments ที่ต้อง rollback
    target: < 5%
    
  - name: time_to_restore
    description: เวลาตั้งแต่ตรวจพบปัญหา → restore service
    target: < 1 hour
    
  - name: drift_rate
    description: % ของ environments ที่มี drift ต่อสัปดาห์
    target: 0%
```

### Deployment Dashboard ด้วย GitHub Actions Summary

```yaml
- name: Create Deployment Summary
  run: |
    echo "# Terraform Deployment Summary" >> $GITHUB_STEP_SUMMARY
    echo "" >> $GITHUB_STEP_SUMMARY
    echo "| Environment | Status | Resources Changed | Apply Time |" >> $GITHUB_STEP_SUMMARY
    echo "|-------------|--------|------------------|------------|" >> $GITHUB_STEP_SUMMARY
    echo "| dev | ✅ Success | +2, ~1, -0 | 1m 23s |" >> $GITHUB_STEP_SUMMARY
    echo "| staging | ✅ Success | +2, ~1, -0 | 2m 05s |" >> $GITHUB_STEP_SUMMARY
    echo "| prod | ⏳ Pending Approval | - | - |" >> $GITHUB_STEP_SUMMARY
```

---

## สรุป GitOps Best Practices

```
GitOps Checklist สำหรับ Terraform:
─────────────────────────────────────────────────────────
✅ Git เป็น single source of truth
✅ ทุก change ผ่าน PR (ไม่มี direct push to main)
✅ Code review บน terraform plan
✅ Security scan ก่อน merge
✅ Automated apply หลัง merge
✅ Sequential deployment (dev → staging → prod)
✅ Manual approval gate สำหรับ production
✅ Rollback ทำได้ง่าย (git revert)
✅ Drift detection scheduled
✅ Audit trail สมบูรณ์ใน Git history
✅ Secrets ไม่อยู่ใน Git (ใช้ Vault/Secrets Manager)
✅ State ใน remote backend (ไม่อยู่ใน Git)
```

---

*จบ Part 96: GitOps with Terraform*
