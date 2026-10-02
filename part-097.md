# Part 97: Atlantis PR Automation (Steps 961-970)

## Atlantis - Terraform Pull Request Automation

---

## Step 961: Atlantis คืออะไร?

### ภาพรวม

**Atlantis** เป็น Open Source tool สำหรับ Terraform Pull Request Automation พัฒนาโดย Hootsuite และต่อมาดูแลโดย community

```
Atlantis Flow:
────────────────────────────────────────────────────────────
                    GitHub/GitLab/Bitbucket
                           │
           ┌───────────────┴───────────────┐
           │                               │
    Developer อยาก           Reviewer เห็น plan
    deploy Terraform          และ approve ใน PR
           │                               │
           ▼                               │
    git push → PR                         │
           │                               │
           ▼                               ▼
    Atlantis รับ webhook    atlantis apply
    atlantis plan           (รันหลัง approve)
           │                               │
           ▼                               ▼
    Post plan comment    terraform apply
    บน PR                  → cloud
```

### ทำไมต้องใช้ Atlantis?

```
ปัญหาโดยไม่มี Atlantis:
❌ ใครก็รัน terraform apply ได้
❌ ไม่มี peer review สำหรับ infrastructure changes
❌ ต้องมี AWS credentials ใน local machine ทุกคน
❌ ไม่มี audit trail ว่าใครรัน apply เมื่อไหร่
❌ State locking ขาดหาย

ด้วย Atlantis:
✅ ทุก apply ต้องผ่าน PR และ review
✅ Plan auto-runs เมื่อ push
✅ Credentials อยู่บน Atlantis server เท่านั้น
✅ Audit trail สมบูรณ์ใน PR comments
✅ State locking อัตโนมัติ
```

---

## Step 962: Atlantis Architecture

```
Atlantis Architecture:
────────────────────────────────────────────────────────────

GitHub/GitLab ←──── Webhook ────→ Atlantis Server
     ↑                              │
     │                              │
     │ PR Comments                  ├── terraform init
     │ Plan output                  ├── terraform plan
     │ Apply results                └── terraform apply
     │                                      │
                                            ▼
                                     AWS/Azure/GCP
                                     (via instance role
                                      or assumed role)
```

### Components

```
Atlantis Server ประกอบด้วย:
├── HTTP Server (port 4141)
│   └── รับ webhooks จาก Git providers
├── Worker
│   └── รัน terraform commands
├── Locking System
│   └── ป้องกัน concurrent applies
└── Configuration
    ├── server.yaml (server config)
    └── atlantis.yaml (repo config)
```

---

## Step 963: Installation Options

### วิธีที่ 1: Docker

```bash
# รัน Atlantis ด้วย Docker
docker run -d \
  --name atlantis \
  -p 4141:4141 \
  -e ATLANTIS_GH_USER="atlantis-bot" \
  -e ATLANTIS_GH_TOKEN="ghp_xxxxxxxxxxxxx" \
  -e ATLANTIS_GH_WEBHOOK_SECRET="mysecret" \
  -e ATLANTIS_REPO_ALLOWLIST="github.com/myorg/*" \
  -e ATLANTIS_AWS_REGION="ap-southeast-1" \
  -v /path/to/repo-config.yaml:/etc/atlantis/repos.yaml \
  ghcr.io/runatlantis/atlantis:latest server \
  --repo-config=/etc/atlantis/repos.yaml

# ดู logs
docker logs -f atlantis
```

### วิธีที่ 2: Binary

```bash
# ดาวน์โหลด binary
ATLANTIS_VERSION="0.28.0"
curl -LO https://github.com/runatlantis/atlantis/releases/download/v${ATLANTIS_VERSION}/atlantis_linux_amd64.zip
unzip atlantis_linux_amd64.zip
chmod +x atlantis
sudo mv atlantis /usr/local/bin/

# รัน
atlantis server \
  --gh-user=atlantis-bot \
  --gh-token=ghp_xxxxxxxxx \
  --gh-webhook-secret=mysecret \
  --repo-allowlist=github.com/myorg/* \
  --port=4141
```

### วิธีที่ 3: Kubernetes (Helm) - แนะนำ

```bash
# เพิ่ม Atlantis Helm repo
helm repo add runatlantis https://runatlantis.github.io/helm-charts
helm repo update

# สร้าง values file
cat > atlantis-values.yaml << 'EOF'
# atlantis-values.yaml
replicaCount: 1

image:
  repository: ghcr.io/runatlantis/atlantis
  tag: "v0.28.0"
  pullPolicy: IfNotPresent

atlantis:
  # GitHub config
  githubUser: "atlantis-bot"
  githubWebhookSecret: "my-webhook-secret"  # จาก Secret
  githubToken: ""  # จาก Secret
  
  # Repository allowlist
  orgAllowlist: "github.com/myorg/*"
  
  # ป้องกัน self-signed certs
  hidePrevPlanComments: true
  enableRegExpCmdAllowList: false

# Environment variables
environmentSecrets:
  - name: ATLANTIS_GH_TOKEN
    secretName: atlantis-secrets
    secretKey: github-token
  - name: ATLANTIS_GH_WEBHOOK_SECRET
    secretName: atlantis-secrets
    secretKey: webhook-secret
  - name: AWS_ROLE_ARN
    secretName: atlantis-secrets
    secretKey: aws-role-arn

# Service Account สำหรับ IRSA (AWS IAM Roles for Service Accounts)
serviceAccount:
  create: true
  annotations:
    eks.amazonaws.com/role-arn: "arn:aws:iam::123456789012:role/atlantis-role"

# Ingress
ingress:
  enabled: true
  className: nginx
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
  hosts:
    - host: atlantis.company.com
      paths:
        - path: /
          pathType: Prefix
  tls:
    - secretName: atlantis-tls
      hosts:
        - atlantis.company.com

# Resource limits
resources:
  requests:
    cpu: 100m
    memory: 256Mi
  limits:
    cpu: 500m
    memory: 512Mi

# Persistence สำหรับ Atlantis data
persistence:
  enabled: true
  storageClass: gp3
  size: 5Gi

# Repo config
repoConfig: |
  repos:
    - id: /.*/
      allow_custom_workflows: false
      allowed_overrides:
        - workflow
      apply_requirements:
        - approved
        - mergeable
      workflow: default
  workflows:
    default:
      plan:
        steps:
          - init:
              extra_args: ["-upgrade"]
          - plan:
              extra_args: ["-detailed-exitcode"]
      apply:
        steps:
          - apply
EOF

# สร้าง namespace
kubectl create namespace atlantis

# สร้าง secrets
kubectl create secret generic atlantis-secrets \
  --namespace atlantis \
  --from-literal=github-token=ghp_xxxxxxxxx \
  --from-literal=webhook-secret=mysecretwebhook \
  --from-literal=aws-role-arn=arn:aws:iam::123456789012:role/atlantis-role

# ติดตั้ง
helm install atlantis runatlantis/atlantis \
  --namespace atlantis \
  --values atlantis-values.yaml

# ตรวจสอบ
kubectl get pods -n atlantis
kubectl logs -f deployment/atlantis -n atlantis
```

---

## Step 964: atlantis.yaml Configuration

### atlantis.yaml แบบ Basic

```yaml
# atlantis.yaml
# ไฟล์นี้อยู่ใน root ของ repository
version: 3

# ===================================================
# Autoplanning - รัน plan อัตโนมัติเมื่อไฟล์เปลี่ยน
# ===================================================
automerge: false  # ไม่ merge PR อัตโนมัติหลัง apply
autodiscover:
  mode: enabled  # auto-discover projects ใน repo

# ===================================================
# Projects
# ===================================================
projects:
  # Project 1: Dev environment
  - name: dev
    dir: environments/dev
    workspace: dev
    terraform_version: 1.7.0
    autoplan:
      when_modified:
        - "*.tf"
        - "*.tfvars"
        - "../modules/**/*.tf"
      enabled: true
    apply_requirements:
      - approved      # ต้องมี PR approval
      - mergeable     # PR ต้องไม่มี conflicts
    workflow: standard
  
  # Project 2: Staging environment
  - name: staging
    dir: environments/staging
    workspace: staging
    terraform_version: 1.7.0
    autoplan:
      when_modified:
        - "*.tf"
        - "*.tfvars"
      enabled: true
    apply_requirements:
      - approved
      - mergeable
      - undiverged    # Branch ต้องเป็นปัจจุบัน
    workflow: standard
  
  # Project 3: Production environment
  - name: prod
    dir: environments/prod
    workspace: prod
    terraform_version: 1.7.0
    autoplan:
      when_modified:
        - "*.tf"
        - "*.tfvars"
      enabled: true
    apply_requirements:
      - approved        # ต้องมี approval
      - mergeable       # ไม่มี conflicts
      - undiverged      # Branch เป็นปัจจุบัน
    workflow: production  # ใช้ workflow เฉพาะ
    
  # Project 4: Modules (plan only, no apply from PR)
  - name: modules
    dir: modules
    autoplan:
      enabled: false  # ไม่ auto-plan (modules ไม่มี state)
```

### atlantis.yaml แบบ Advanced

```yaml
# atlantis.yaml - Advanced configuration
version: 3

automerge: false
parallel_plan: true    # Plan หลาย projects พร้อมกัน
parallel_apply: false  # Apply ทีละ project (safe)

projects:
  # ===== Production with strict requirements =====
  - name: prod-networking
    dir: environments/prod/networking
    workspace: prod
    terraform_version: 1.7.0
    
    autoplan:
      when_modified:
        - "*.tf"
        - "*.tfvars"
        - "../../modules/networking/**/*.tf"
      enabled: true
    
    apply_requirements:
      - approved
      - mergeable
    
    workflow: prod-workflow
    
    # ป้องกัน destroy ใน production
    delete_source_branch_on_merge: false

# ===================================================
# Custom Workflows
# ===================================================
workflows:
  standard:
    plan:
      steps:
        - env:
            name: TF_VAR_git_commit
            command: 'echo $HEAD_COMMIT'
        - init:
            extra_args: ["-upgrade", "-reconfigure"]
        - plan:
            extra_args: ["-var-file=terraform.tfvars", "-detailed-exitcode"]
    apply:
      steps:
        - apply:
            extra_args: ["-auto-approve"]
  
  prod-workflow:
    plan:
      steps:
        # ===== Pre-plan hooks =====
        - run: echo "=== Starting pre-plan checks ==="
        
        # Security scan ก่อน plan
        - run: |
            tfsec . \
              --minimum-severity HIGH \
              --format text || exit 1
            echo "TFSec: PASSED"
        
        # Checkov
        - run: |
            checkov -d . \
              --framework terraform \
              --compact \
              --quiet || exit 1
            echo "Checkov: PASSED"
        
        - init:
            extra_args: ["-upgrade"]
        
        - plan:
            extra_args: ["-var-file=terraform.tfvars", "-out=tfplan.binary"]
        
        # ===== Post-plan hooks =====
        - run: |
            # Cost estimation
            if command -v infracost &> /dev/null; then
              infracost breakdown \
                --path . \
                --format json \
                > infracost.json
              echo "=== Cost Estimate ==="
              infracost output \
                --path infracost.json \
                --format table
            fi
    
    apply:
      steps:
        # Pre-apply: สร้าง backup ของ current state
        - run: |
            aws s3 cp \
              s3://my-terraform-state/prod/terraform.tfstate \
              s3://my-terraform-state/backups/prod-before-$(date +%Y%m%d-%H%M%S).tfstate \
              || echo "Backup failed - continuing"
        
        - apply:
            extra_args: ["-auto-approve"]
        
        # Post-apply: notification
        - run: |
            curl -X POST \
              -H 'Content-type: application/json' \
              --data "{\"text\": \"✅ Production apply completed by ${ATLANTIS_USER}\"}" \
              ${SLACK_WEBHOOK_URL}
```

---

## Step 965: Webhook Configuration

### GitHub Webhook Setup

```bash
# GitHub: Repository Settings → Webhooks → Add webhook

# Payload URL: https://atlantis.company.com/events
# Content type: application/json
# Secret: <same as ATLANTIS_GH_WEBHOOK_SECRET>
# Events:
#   ✅ Pull requests
#   ✅ Push
#   ✅ Issue comments
```

### โดยใช้ GitHub CLI

```bash
# สร้าง webhook ด้วย GitHub CLI
gh api \
  repos/myorg/my-terraform-repo/hooks \
  --method POST \
  --field name=web \
  --field config[url]=https://atlantis.company.com/events \
  --field config[content_type]=json \
  --field config[secret]=mywebhooksecret \
  --field config[insecure_ssl]=0 \
  --field events[]=push \
  --field events[]=pull_request \
  --field events[]=issue_comment
```

### GitLab Webhook

```bash
# GitLab: Project Settings → Webhooks

URL: https://atlantis.company.com/events
Secret Token: <same as ATLANTIS_GITLAB_WEBHOOK_SECRET>
Trigger:
  ✅ Push events
  ✅ Comments
  ✅ Merge request events
```

---

## Step 966: Atlantis Commands ใน PR Comments

### คำสั่งพื้นฐาน

```bash
# ============================================================
# Plan commands
# ============================================================

# รัน plan บน project ที่เปลี่ยน
atlantis plan

# รัน plan บน specific project
atlantis plan -p dev

# รัน plan บน specific directory
atlantis plan -d environments/dev

# Plan พร้อม workspace
atlantis plan -p prod -w prod

# Plan แบบ verbose
atlantis plan -- -refresh-only

# ============================================================
# Apply commands
# ============================================================

# Apply ทุก planned projects
atlantis apply

# Apply specific project
atlantis apply -p dev

# Apply specific directory
atlantis apply -d environments/dev

# Apply แบบ manual (ข้าม workflow บาง steps)
atlantis apply -- -target=aws_instance.web

# ============================================================
# Management commands
# ============================================================

# Unlock (ล้าง locks ทั้งหมดใน PR)
atlantis unlock

# รัน plan อีกครั้ง (re-plan)
atlantis plan -p dev

# ดู help
atlantis help

# Import resource (experimental)
atlantis import aws_instance.web i-1234567890abcdef0

# State manipulation (dangerous!)
atlantis state rm aws_instance.old
```

### ตัวอย่าง PR Comment Flow

```
Developer: @atlantis-bot atlantis plan -p prod
─────────────────────────────────────────────────────────────

Atlantis Bot: 
  Running Plan: environments/prod

  ✅ Plan succeeded
  
  Terraform will perform the following actions:
  
    + aws_instance.new_api_server
        ami:           "ami-0123456789"
        instance_type: "t3.medium"
        tags.Name:     "prod-api-server-001"
  
  Plan: 1 to add, 0 to change, 0 to destroy.
  
  ─────────────────────────────────────────────────────────────
  To apply: comment `atlantis apply -p prod`
  To discard: comment `atlantis unlock`

─────────────────────────────────────────────────────────────

Reviewer: LGTM! Approved. (clicks GitHub approve button)

─────────────────────────────────────────────────────────────

Developer: @atlantis-bot atlantis apply -p prod

─────────────────────────────────────────────────────────────

Atlantis Bot:
  Applying Plan: environments/prod
  
  ✅ Apply succeeded
  
  Apply complete! Resources: 1 added, 0 changed, 0 destroyed.
  
  Outputs:
    instance_id = "i-0123456789abcdef0"
    public_ip   = "1.2.3.4"
```

---

## Step 967: Pull Request Locking

### ทำงานอย่างไร?

```
PR Locking Mechanism:
────────────────────────────────────────────────────────────

PR #42 (feature/add-rds) รัน atlantis plan → ล็อค environments/prod

PR #43 (fix/security-group) พยายามรัน atlantis plan บน environments/prod
  → ❌ Error: Repo environments/prod/workspace: default is currently
            locked by PR #42

PR #42 Merge หรือ unlock → ล็อคหลุด

PR #43 สามารถรัน plan ได้
```

### การจัดการ Locks

```bash
# ดู locks ทั้งหมด
# ไปที่ https://atlantis.company.com/locks

# Unlock จาก PR comment
atlantis unlock

# Force unlock (admin only)
# ไปที่ Atlantis web UI > Locks > Delete
```

---

## Step 968: Server-side Configuration (repos.yaml)

### server.yaml

```yaml
# server.yaml - Main server configuration
repos:
  # Allow all repos ใน organization
  - id: "github.com/myorg/.*"
    branch: /.*/
    
    # Repository-level defaults
    allowed_overrides:
      - workflow
      - apply_requirements
    
    allow_custom_workflows: false  # ป้องกัน custom workflows จาก repos
    
    apply_requirements:
      - approved
      - mergeable
    
    # Webhook secrets
    webhook_secret: ${ATLANTIS_GH_WEBHOOK_SECRET}
  
  # Production repositories - stricter settings
  - id: "github.com/myorg/prod-infrastructure"
    apply_requirements:
      - approved
      - mergeable
      - undiverged
    
    # ต้องการ 2 approvals สำหรับ production
    # (ตั้งค่าใน GitHub Branch Protection)

# Default workflow
workflows:
  default:
    plan:
      steps:
        - init
        - plan
    apply:
      steps:
        - apply
  
  # Strict workflow สำหรับ production
  strict:
    plan:
      steps:
        - run: echo "Pre-plan security check..."
        - run: tfsec . --minimum-severity HIGH --format text
        - init
        - plan
    apply:
      steps:
        - run: echo "Creating state backup..."
        - run: |
            aws s3 cp \
              s3://${TF_STATE_BUCKET}/terraform.tfstate \
              s3://${TF_STATE_BUCKET}/backups/backup-$(date +%Y%m%d%H%M%S).tfstate
        - apply
        - run: echo "Apply complete! Sending notifications..."
        - run: |
            curl -s -X POST \
              -H "Content-type: application/json" \
              --data "{\"text\":\"✅ Production deployment completed by ${ATLANTIS_USER}\"}" \
              ${SLACK_WEBHOOK_URL}
```

### Environment Variables

```bash
# ===== GitHub =====
ATLANTIS_GH_USER=atlantis-bot
ATLANTIS_GH_TOKEN=ghp_xxxxxxxxxxxxx
ATLANTIS_GH_WEBHOOK_SECRET=mysecretwebhook

# ===== GitLab =====
ATLANTIS_GITLAB_USER=atlantis-bot
ATLANTIS_GITLAB_TOKEN=glpat-xxxxxxxxxxxxx
ATLANTIS_GITLAB_WEBHOOK_SECRET=mysecretwebhook

# ===== AWS =====
AWS_REGION=ap-southeast-1
AWS_ROLE_ARN=arn:aws:iam::123456789012:role/atlantis-role

# หรือใช้ static credentials (ไม่แนะนำ)
# AWS_ACCESS_KEY_ID=AKIA...
# AWS_SECRET_ACCESS_KEY=...

# ===== Server =====
ATLANTIS_PORT=4141
ATLANTIS_ATLANTIS_URL=https://atlantis.company.com

# ===== Notifications =====
SLACK_WEBHOOK_URL=https://hooks.slack.com/services/...

# ===== Terraform Cloud =====
TFE_TOKEN=xxxxxxxxxxxxx
```

---

## Step 969: Atlantis with Vault

### Vault Integration

```yaml
# atlantis.yaml - ใช้ Vault สำหรับ secrets
workflows:
  vault-workflow:
    plan:
      steps:
        # ดึง credentials จาก Vault ก่อน plan
        - run: |
            # ดึง AWS credentials
            export VAULT_TOKEN=$(vault write auth/aws/login \
              role=atlantis \
              pkcs7=$(curl -s http://169.254.169.254/latest/dynamic/instance-identity/pkcs7) \
              | jq -r '.auth.client_token')
            
            AWS_CREDS=$(vault read -format=json aws/creds/terraform-role)
            export AWS_ACCESS_KEY_ID=$(echo $AWS_CREDS | jq -r '.data.access_key')
            export AWS_SECRET_ACCESS_KEY=$(echo $AWS_CREDS | jq -r '.data.secret_key')
            
            echo "AWS credentials obtained from Vault"
        
        - init
        - plan
    
    apply:
      steps:
        # ดึง credentials อีกครั้งสำหรับ apply
        - run: |
            # Same vault auth as plan
            AWS_CREDS=$(vault read -format=json aws/creds/terraform-role)
            export AWS_ACCESS_KEY_ID=$(echo $AWS_CREDS | jq -r '.data.access_key')
            export AWS_SECRET_ACCESS_KEY=$(echo $AWS_CREDS | jq -r '.data.secret_key')
        - apply
```

---

## Step 970: Atlantis vs Terraform Cloud Comparison

### ตารางเปรียบเทียบ

```
┌────────────────────────┬───────────────────────┬───────────────────────┐
│ Feature                │ Atlantis              │ Terraform Cloud (TFC) │
├────────────────────────┼───────────────────────┼───────────────────────┤
│ License                │ Apache 2.0 (Free)     │ Freemium/Paid         │
│ Hosting                │ Self-hosted           │ SaaS + Self-hosted    │
│ Setup complexity       │ Medium                │ Easy                  │
├────────────────────────┼───────────────────────┼───────────────────────┤
│ PR automation          │ ✅ Core feature       │ ✅ VCS-driven runs    │
│ Auto plan on PR        │ ✅                    │ ✅                    │
│ Apply on merge         │ ✅ via comment        │ ✅ auto/manual        │
│ PR locking             │ ✅ Built-in           │ ✅ State locking      │
├────────────────────────┼───────────────────────┼───────────────────────┤
│ State management       │ Remote backend only   │ ✅ Built-in state     │
│ Module registry        │ ❌                    │ ✅ Private registry   │
│ Policy engine          │ Sentinel/OPA          │ Sentinel (paid)       │
│ Cost estimation        │ ❌ (need plugin)      │ ✅ (Business+)        │
│ Drift detection        │ ❌                    │ ✅ Health assessments  │
├────────────────────────┼───────────────────────┼───────────────────────┤
│ RBAC                   │ PR-based (GitHub)     │ ✅ Advanced RBAC      │
│ Audit logging          │ Git history + logs    │ ✅ Detailed audit     │
│ API                    │ Limited               │ ✅ Full REST API      │
├────────────────────────┼───────────────────────┼───────────────────────┤
│ Multi-cloud            │ ✅ Any provider       │ ✅ Any provider       │
│ Kubernetes deploy      │ ✅ Helm available     │ ✅ Terraform          │
├────────────────────────┼───────────────────────┼───────────────────────┤
│ Best for               │ Teams ที่ต้องการ     │ Enterprise ที่ต้อง   │
│                        │ control + free        │ managed service       │
│                        │                       │ + advanced features   │
└────────────────────────┴───────────────────────┴───────────────────────┘
```

### เมื่อไหรควรใช้อะไร?

```
ใช้ Atlantis เมื่อ:
  ✅ ต้องการ self-hosted บน infrastructure ของตัวเอง
  ✅ ต้องการ flexibility สูงสุด
  ✅ ไม่ต้องการ vendor lock-in
  ✅ Team มีทรัพยากรในการดูแล infrastructure
  ✅ Budget จำกัด

ใช้ Terraform Cloud เมื่อ:
  ✅ ต้องการ managed service
  ✅ ต้องการ module registry
  ✅ ต้องการ Sentinel policies
  ✅ ต้องการ built-in state management
  ✅ มีงบประมาณ
```

---

## Complete Atlantis Kubernetes Deployment

### atlantis-deployment.yaml

```yaml
# atlantis-k8s-deployment.yaml
---
# Namespace
apiVersion: v1
kind: Namespace
metadata:
  name: atlantis
  labels:
    app: atlantis

---
# ServiceAccount
apiVersion: v1
kind: ServiceAccount
metadata:
  name: atlantis
  namespace: atlantis
  annotations:
    # AWS IAM Roles for Service Accounts (IRSA)
    eks.amazonaws.com/role-arn: "arn:aws:iam::123456789012:role/atlantis-role"

---
# Secrets
apiVersion: v1
kind: Secret
metadata:
  name: atlantis-secrets
  namespace: atlantis
type: Opaque
stringData:
  github-token: "ghp_xxxxxxxxxxxxx"
  webhook-secret: "mysecretwebhook"

---
# ConfigMap สำหรับ repos.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: atlantis-config
  namespace: atlantis
data:
  repos.yaml: |
    repos:
      - id: "github.com/myorg/.*"
        apply_requirements:
          - approved
          - mergeable
        allowed_overrides:
          - workflow
        workflow: default
    
    workflows:
      default:
        plan:
          steps:
            - init:
                extra_args: ["-upgrade"]
            - plan
        apply:
          steps:
            - apply

---
# StatefulSet
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: atlantis
  namespace: atlantis
spec:
  serviceName: atlantis
  replicas: 1  # Atlantis ไม่รองรับ horizontal scaling
  selector:
    matchLabels:
      app: atlantis
  template:
    metadata:
      labels:
        app: atlantis
    spec:
      serviceAccountName: atlantis
      
      securityContext:
        runAsUser: 100
        runAsGroup: 100
        fsGroup: 100
      
      containers:
        - name: atlantis
          image: ghcr.io/runatlantis/atlantis:v0.28.0
          imagePullPolicy: IfNotPresent
          
          command: ["atlantis", "server"]
          
          args:
            - --repo-config=/etc/atlantis/repos.yaml
          
          env:
            - name: ATLANTIS_GH_USER
              value: "atlantis-bot"
            
            - name: ATLANTIS_GH_TOKEN
              valueFrom:
                secretKeyRef:
                  name: atlantis-secrets
                  key: github-token
            
            - name: ATLANTIS_GH_WEBHOOK_SECRET
              valueFrom:
                secretKeyRef:
                  name: atlantis-secrets
                  key: webhook-secret
            
            - name: ATLANTIS_REPO_ALLOWLIST
              value: "github.com/myorg/*"
            
            - name: ATLANTIS_ATLANTIS_URL
              value: "https://atlantis.company.com"
            
            - name: ATLANTIS_PORT
              value: "4141"
            
            - name: ATLANTIS_DATA_DIR
              value: "/atlantis"
            
            - name: ATLANTIS_LOG_LEVEL
              value: "info"
            
            - name: ATLANTIS_HIDE_PREV_PLAN_COMMENTS
              value: "true"
            
            - name: ATLANTIS_ENABLE_DIFF_MARKDOWN_FORMAT
              value: "true"
            
            # Terraform version
            - name: ATLANTIS_DEFAULT_TF_VERSION
              value: "1.7.0"
            
            # Allow custom TF versions per project
            - name: ATLANTIS_TF_DOWNLOAD
              value: "true"
          
          ports:
            - containerPort: 4141
              name: http
          
          livenessProbe:
            httpGet:
              path: /healthz
              port: 4141
            initialDelaySeconds: 30
            periodSeconds: 10
            timeoutSeconds: 5
          
          readinessProbe:
            httpGet:
              path: /healthz
              port: 4141
            initialDelaySeconds: 10
            periodSeconds: 5
          
          resources:
            requests:
              cpu: 100m
              memory: 256Mi
            limits:
              cpu: 500m
              memory: 1Gi
          
          volumeMounts:
            - name: atlantis-data
              mountPath: /atlantis
            - name: atlantis-config
              mountPath: /etc/atlantis
              readOnly: true
      
      volumes:
        - name: atlantis-config
          configMap:
            name: atlantis-config
  
  volumeClaimTemplates:
    - metadata:
        name: atlantis-data
      spec:
        accessModes: ["ReadWriteOnce"]
        storageClassName: gp3
        resources:
          requests:
            storage: 5Gi

---
# Service
apiVersion: v1
kind: Service
metadata:
  name: atlantis
  namespace: atlantis
spec:
  selector:
    app: atlantis
  ports:
    - port: 80
      targetPort: 4141
      name: http
  type: ClusterIP

---
# Ingress
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: atlantis
  namespace: atlantis
  annotations:
    kubernetes.io/ingress.class: nginx
    cert-manager.io/cluster-issuer: letsencrypt-prod
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    nginx.ingress.kubernetes.io/force-ssl-redirect: "true"
spec:
  tls:
    - hosts:
        - atlantis.company.com
      secretName: atlantis-tls
  rules:
    - host: atlantis.company.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: atlantis
                port:
                  number: 80

---
# PodDisruptionBudget
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: atlantis
  namespace: atlantis
spec:
  maxUnavailable: 0  # ห้าม disrupt Atlantis
  selector:
    matchLabels:
      app: atlantis

---
# NetworkPolicy - จำกัด traffic
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: atlantis-network-policy
  namespace: atlantis
spec:
  podSelector:
    matchLabels:
      app: atlantis
  policyTypes:
    - Ingress
    - Egress
  ingress:
    # รับ webhooks จาก ingress controller
    - from:
        - namespaceSelector:
            matchLabels:
              name: ingress-nginx
      ports:
        - port: 4141
  egress:
    # อนุญาต outbound ทั้งหมด (GitHub API, AWS, etc.)
    - {}
```

### Deploy และทดสอบ

```bash
# Deploy
kubectl apply -f atlantis-k8s-deployment.yaml

# ตรวจสอบ
kubectl get all -n atlantis
kubectl logs -f statefulset/atlantis -n atlantis

# ทดสอบ health endpoint
kubectl port-forward svc/atlantis 4141:80 -n atlantis &
curl http://localhost:4141/healthz

# ตรวจสอบ version
curl http://localhost:4141/version
```

---

## Atlantis Security Hardening

### Security Best Practices

```yaml
# Security settings ใน repos.yaml
repos:
  - id: "github.com/myorg/.*"
    # ห้ามใช้ custom workflows ใน repos
    allow_custom_workflows: false
    
    # จำกัด commands ที่ใช้ได้
    allowed_overrides:
      - workflow
    
    # Apply ต้องมี approval
    apply_requirements:
      - approved
      - mergeable

# Disable features ที่ไม่ต้องการ
# ATLANTIS_ALLOW_COMMANDS=plan,apply,unlock  # Whitelist commands
# ATLANTIS_WRITE_GIT_CREDS=false              # ไม่เขียน git creds
# ATLANTIS_DISABLE_AUTOPLAN=false             # Enable autoplan
# ATLANTIS_SILENCE_ALLOWLIST_ERRORS=false     # แสดง allowlist errors
```

### IAM Policy สำหรับ Atlantis

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "sts:AssumeRole"
      ],
      "Resource": [
        "arn:aws:iam::111111111111:role/terraform-dev",
        "arn:aws:iam::222222222222:role/terraform-staging",
        "arn:aws:iam::333333333333:role/terraform-prod"
      ]
    }
  ]
}
```

### atlantis.yaml ที่ใช้ per-environment roles

```yaml
# atlantis.yaml
workflows:
  dev:
    plan:
      steps:
        - run: |
            export $(aws sts assume-role \
              --role-arn arn:aws:iam::111111111111:role/terraform-dev \
              --role-session-name atlantis-dev \
              --query 'Credentials.[AccessKeyId,SecretAccessKey,SessionToken]' \
              --output text | awk '{print "AWS_ACCESS_KEY_ID="$1" AWS_SECRET_ACCESS_KEY="$2" AWS_SESSION_TOKEN="$3}')
        - init
        - plan
  
  prod:
    plan:
      steps:
        - run: |
            export $(aws sts assume-role \
              --role-arn arn:aws:iam::333333333333:role/terraform-prod \
              --role-session-name atlantis-prod \
              --query 'Credentials.[AccessKeyId,SecretAccessKey,SessionToken]' \
              --output text | awk '{print "AWS_ACCESS_KEY_ID="$1" AWS_SECRET_ACCESS_KEY="$2" AWS_SESSION_TOKEN="$3}')
        - init
        - plan
```

---

## สรุป Atlantis Command Reference

```bash
# ใน PR Comments:

atlantis plan                    # Plan ทุก projects ที่เปลี่ยน
atlantis plan -p <project>       # Plan specific project
atlantis plan -d <dir>           # Plan specific directory
atlantis plan -w <workspace>     # Plan specific workspace
atlantis plan -- -target=<res>   # Plan specific resource

atlantis apply                   # Apply ทุก planned projects
atlantis apply -p <project>      # Apply specific project
atlantis apply -- -target=<res>  # Apply specific resource

atlantis unlock                  # Unlock PR
atlantis version                 # ดู Terraform version

# Admin commands (ใน UI หรือ API):
GET  /locks                      # List all locks
DELETE /locks/<id>               # Delete specific lock
GET  /status                     # Server status
GET  /events                     # Event log
```

---

*จบ Part 97: Atlantis PR Automation*
