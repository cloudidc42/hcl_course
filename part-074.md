# Part 074: Terraform Cloud & Remote Operations (ขั้นตอนที่ 731-740)

## บทนำ (Introduction)

Terraform Cloud (TFC) เป็น SaaS platform ของ HashiCorp ที่ให้บริการ:
- Remote state storage
- Remote plan & apply
- Policy enforcement (Sentinel)
- Private module registry
- Team management และ RBAC
- Cost estimation

---

## ขั้นตอนที่ 731: Terraform Cloud Overview

### ความแตกต่างระหว่าง Terraform CLI, Cloud, และ Enterprise

```
┌─────────────────────────────────────────────────────────┐
│                    Terraform Ecosystem                   │
├───────────────┬──────────────────┬──────────────────────┤
│  Terraform    │  Terraform       │  Terraform           │
│  CLI (Free)   │  Cloud (Free*)   │  Enterprise (Paid)   │
├───────────────┼──────────────────┼──────────────────────┤
│ Local state   │ Remote state     │ Self-hosted          │
│ Local plans   │ Remote plans     │ Air-gapped           │
│ Manual runs   │ VCS integration  │ SAML SSO             │
│               │ Cost estimation  │ Audit logs           │
│               │ Basic Sentinel   │ Advanced Sentinel    │
│               │ Team mgmt        │ Custom agents        │
└───────────────┴──────────────────┴──────────────────────┘

* Free tier: 500 resources managed
```

### สมัคร Terraform Cloud

```
1. ไปที่ https://app.terraform.io
2. Click "Create account"
3. ใส่ email, username, password
4. ยืนยัน email
5. สร้าง Organization
```

---

## ขั้นตอนที่ 732: Organizations และ Workspaces

### Organization Structure

```
Organization: mycompany
├── Teams
│   ├── infrastructure    (admin)
│   ├── developers        (plan only)
│   └── ops               (apply)
├── Variable Sets
│   ├── aws-credentials   (global)
│   └── common-tags       (workspace-specific)
├── Workspaces
│   ├── networking-prod
│   ├── networking-staging
│   ├── app-prod
│   ├── app-staging
│   └── database-prod
└── Private Module Registry
    ├── terraform-aws-vpc
    ├── terraform-aws-eks
    └── terraform-aws-rds
```

### สร้าง Workspace ผ่าน CLI

```bash
# Login ไปยัง Terraform Cloud
terraform login

# สร้างไฟล์ backend configuration
cat > backend.tf << 'EOF'
terraform {
  cloud {
    organization = "mycompany"

    workspaces {
      name = "networking-prod"
    }
  }
}
EOF

# Initialize (จะสร้าง workspace ถ้ายังไม่มี)
terraform init
```

---

## ขั้นตอนที่ 733: TFC Workspace Configuration

### Cloud Block (แนะนำ - Terraform 1.1+)

```hcl
# main.tf หรือ backend.tf
terraform {
  # Required providers
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }

  # Terraform Cloud configuration
  cloud {
    organization = "mycompany"

    # Option 1: Single workspace
    workspaces {
      name = "networking-prod"
    }

    # Option 2: Multiple workspaces by tags (workspace set)
    # workspaces {
    #   tags = ["networking", "production"]
    # }
  }
}

provider "aws" {
  region = var.aws_region
}
```

### workspace-specific variables

```hcl
# ใช้ terraform.workspace ใน configuration
locals {
  environment = {
    "networking-prod"    = "prod"
    "networking-staging" = "staging"
    "networking-dev"     = "dev"
  }

  current_env = lookup(local.environment, terraform.workspace, "dev")
}

resource "aws_vpc" "main" {
  cidr_block = var.vpc_cidr

  tags = {
    Name        = "vpc-${local.current_env}"
    Environment = local.current_env
    Workspace   = terraform.workspace
  }
}
```

---

## ขั้นตอนที่ 734: VCS-Driven Workflow

### เชื่อมต่อ GitHub กับ TFC

```
TFC Web UI:
1. Organization Settings → Version Control Providers
2. Add a VCS Provider → GitHub
3. Authorize Terraform Cloud GitHub App
4. ระบุ repos ที่อนุญาต
```

### VCS-driven workspace configuration

```
Workspace Settings (VCS-driven):
- Repository: github.com/mycompany/infrastructure
- Branch: main
- Working Directory: environments/production/networking
- Terraform Working Directory: environments/production/networking
- Auto Apply: disabled (manual review before apply)
- Speculative Plans: enabled (for PRs)
```

### GitHub Repository Structure

```
infrastructure/
├── environments/
│   ├── production/
│   │   ├── networking/     ← workspace: networking-prod
│   │   │   ├── main.tf
│   │   │   └── terraform.tfvars
│   │   ├── compute/        ← workspace: compute-prod
│   │   └── database/       ← workspace: database-prod
│   └── staging/
│       ├── networking/     ← workspace: networking-staging
│       └── compute/        ← workspace: compute-staging
└── modules/
    ├── networking/
    └── compute/
```

### GitOps Flow

```
1. Developer สร้าง PR
   └→ TFC สร้าง Speculative Plan (plan-only)
      └→ GitHub PR แสดงผล plan ว่ามีอะไรเปลี่ยน

2. Code Review + Plan Approval
   └→ Merge PR ไป main branch

3. TFC auto-trigger run
   └→ Plan → (Sentinel policy check) → Apply
```

---

## ขั้นตอนที่ 735: CLI-Driven Workflow

### CLI Workflow Steps

```bash
# 1. Login
terraform login
# จะเปิด browser ให้ generate token

# 2. Initialize workspace
terraform init

# 3. Plan (remote)
terraform plan
# Plan รันบน TFC, output แสดงใน terminal

# 4. Apply (remote)
terraform apply
# Apply รันบน TFC

# 5. Show workspace
terraform workspace show

# 6. List workspaces
terraform workspace list
```

### CLI Configuration สำหรับ Team

```hcl
# ~/.terraform.d/credentials.tfrc.json (จะถูกสร้างโดย terraform login)
{
  "credentials": {
    "app.terraform.io": {
      "token": "your-api-token-here"
    }
  }
}
```

### tfrc file สำหรับ CI/CD

```hcl
# .terraformrc
credentials "app.terraform.io" {
  token = "$TF_TOKEN_app_terraform_io"
}
```

```bash
# ใน CI/CD pipeline
export TF_TOKEN_app_terraform_io="your-token"
terraform init
terraform plan
```

---

## ขั้นตอนที่ 736: Remote State Storage

### ทำไม Remote State ถึงดีกว่า Local State?

```
Local State ปัญหา:
- ไม่มี locking (concurrent apply = corruption)
- ไม่มี versioning
- ไม่มี encryption
- ทีมต้องแชร์ไฟล์

TFC Remote State ดีกว่า:
✓ Automatic locking
✓ State versioning (unlimited)
✓ Encrypted at rest
✓ Access control (RBAC)
✓ ดึง state จาก workspace อื่นได้
```

### อ่าน Output จาก Remote State

```hcl
# การดึงข้อมูลจาก workspace อื่นใน TFC
data "terraform_remote_state" "networking" {
  backend = "remote"

  config = {
    organization = "mycompany"
    workspaces = {
      name = "networking-prod"
    }
  }
}

# ใช้ข้อมูล
resource "aws_instance" "app" {
  ami           = var.ami_id
  instance_type = "t3.micro"
  subnet_id     = data.terraform_remote_state.networking.outputs.private_subnet_id

  tags = {
    Name = "app-server"
  }
}
```

---

## ขั้นตอนที่ 737: Variables ใน TFC

### ประเภท Variables ใน TFC

```
1. Terraform Variables (var.*):
   - ค่าที่ส่งให้ Terraform configuration
   - Sensitive หรือ Non-sensitive

2. Environment Variables (env vars):
   - AWS_ACCESS_KEY_ID, AWS_SECRET_ACCESS_KEY
   - หรือ AWS credentials ผ่าน OIDC
   - สำคัญ: ต้อง mark เป็น Sensitive

3. Variable Sets:
   - กลุ่ม variables ที่ share ระหว่าง workspaces
```

### ตั้งค่า Variables ผ่าน TFC API

```bash
# ใช้ TFC API เพื่อ set variable
TFC_TOKEN="your-token"
ORG="mycompany"
WORKSPACE="networking-prod"

# ดึง workspace ID
WORKSPACE_ID=$(curl -s \
  -H "Authorization: Bearer $TFC_TOKEN" \
  -H "Content-Type: application/vnd.api+json" \
  "https://app.terraform.io/api/v2/organizations/$ORG/workspaces/$WORKSPACE" \
  | jq -r '.data.id')

# สร้าง variable
curl -s \
  -X POST \
  -H "Authorization: Bearer $TFC_TOKEN" \
  -H "Content-Type: application/vnd.api+json" \
  -d "{
    \"data\": {
      \"type\": \"vars\",
      \"attributes\": {
        \"key\": \"environment\",
        \"value\": \"production\",
        \"category\": \"terraform\",
        \"sensitive\": false
      },
      \"relationships\": {
        \"workspace\": {
          \"data\": {
            \"id\": \"$WORKSPACE_ID\",
            \"type\": \"workspaces\"
          }
        }
      }
    }
  }" \
  "https://app.terraform.io/api/v2/vars"
```

### Variable Sets สำหรับ AWS Credentials

```
TFC UI:
Organization → Settings → Variable Sets → New Variable Set

Variable Set: "AWS Credentials (Production)"
Apply to:
  - Specific workspaces: networking-prod, compute-prod
  - OR: All workspaces

Variables:
  AWS_ACCESS_KEY_ID     = (sensitive)
  AWS_SECRET_ACCESS_KEY = (sensitive)
  AWS_DEFAULT_REGION    = us-east-1
```

---

## ขั้นตอนที่ 738: Cost Estimation

### Cost Estimation ใน TFC

```
เปิดใช้:
Workspace Settings → General → Cost estimation: Enabled

ทุก run จะมี:
- Plan → Cost Estimation → (Sentinel) → Apply

ข้อมูล Cost:
- Monthly cost delta
- New resources cost
- Removed resources savings
```

### ตัวอย่าง Cost Estimation Output

```
Cost Estimation

Resources: 5 of 5 estimated

  + aws_instance.web (t3.large)         $60.74/mo
  + aws_db_instance.main (db.t3.medium) $49.64/mo
  + aws_nat_gateway.main                $32.85/mo
  + aws_eip.nat                          $3.65/mo
  ~ aws_instance.app (t3.micro → t3.small) +$8.76/mo

Monthly cost increase: +$155.64
```

---

## ขั้นตอนที่ 739: TFC Notifications และ Workspace Configuration

### Slack Notifications

```
TFC UI:
Workspace → Settings → Notifications → Create a Notification

Type: Slack
URL: https://hooks.slack.com/services/xxx/yyy/zzz
Events:
  ✓ Created
  ✓ Planning
  ✓ Needs Attention (waiting for confirmation)
  ✓ Applying
  ✓ Completed
  ✓ Errored
```

### Email Notifications

```
User Settings → Notifications:
- Organization-level notifications
- Workspace-level notifications
- Email/Slack per event type
```

### Run Triggers (ต่อ workspaces)

```
TFC UI:
Workspace → Settings → Run Triggers

Source Workspaces:
- networking-prod  → triggers → compute-prod
- networking-prod  → triggers → database-prod

เมื่อ networking-prod apply สำเร็จ จะ trigger plan ใน compute-prod และ database-prod
```

---

## ขั้นตอนที่ 740: TFC Agents

### TFC Agents สำหรับ Private Infrastructure

```
ปัญหา: TFC runners ไม่สามารถ reach private resources ได้
เช่น: databases ใน private subnet, on-premise systems

Solution: TFC Agents (self-hosted runners ใน VPC ของคุณ)
```

### ติดตั้ง TFC Agent

```bash
# Option 1: Docker
docker run -d \
  --name tfc-agent \
  -e TFC_AGENT_TOKEN="your-agent-token" \
  -e TFC_AGENT_NAME="prod-agent-1" \
  hashicorp/tfc-agent:latest

# Option 2: ไว้ใน EC2 instance
#!/bin/bash
# bootstrap_tfc_agent.sh

# ติดตั้ง dependencies
apt-get update && apt-get install -y curl unzip

# ดาวน์โหลด tfc-agent
TFC_AGENT_VERSION="1.14.0"
curl -O "https://releases.hashicorp.com/tfc-agent/${TFC_AGENT_VERSION}/tfc-agent_${TFC_AGENT_VERSION}_linux_amd64.zip"
unzip "tfc-agent_${TFC_AGENT_VERSION}_linux_amd64.zip"
mv tfc-agent /usr/local/bin/

# สร้าง service
cat > /etc/systemd/system/tfc-agent.service << 'EOF'
[Unit]
Description=Terraform Cloud Agent
After=network.target

[Service]
Type=simple
User=tfc-agent
Environment="TFC_AGENT_TOKEN=your-token"
Environment="TFC_AGENT_NAME=prod-agent-1"
ExecStart=/usr/local/bin/tfc-agent
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF

systemctl enable tfc-agent
systemctl start tfc-agent
```

### Terraform สำหรับ TFC Agent Pool

```hcl
# สร้าง agent pool และ agents ด้วย Terraform
terraform {
  required_providers {
    tfe = {
      source  = "hashicorp/tfe"
      version = "~> 0.51"
    }
  }
}

provider "tfe" {
  token = var.tfe_token
}

# สร้าง Agent Pool
resource "tfe_agent_pool" "production" {
  name         = "production-agents"
  organization = "mycompany"
}

# Generate agent token
resource "tfe_agent_token" "agent_1" {
  agent_pool_id = tfe_agent_pool.production.id
  description   = "Production Agent 1"
}

# กำหนด workspace ให้ใช้ agent pool นี้
resource "tfe_workspace" "networking_prod" {
  name         = "networking-prod"
  organization = "mycompany"
  agent_pool_id       = tfe_agent_pool.production.id
  execution_mode      = "agent"
}

output "agent_token" {
  value     = tfe_agent_token.agent_1.token
  sensitive = true
}
```

### EC2 Instance สำหรับ TFC Agent

```hcl
# สร้าง EC2 instance สำหรับ TFC Agent
resource "aws_instance" "tfc_agent" {
  ami           = data.aws_ami.amazon_linux_2023.id
  instance_type = "t3.small"
  subnet_id     = aws_subnet.private.id

  iam_instance_profile = aws_iam_instance_profile.tfc_agent.name

  user_data = base64encode(templatefile("${path.module}/tfc-agent-bootstrap.sh.tpl", {
    agent_token = var.tfc_agent_token
    agent_name  = "ec2-agent-${count.index + 1}"
  }))

  tags = {
    Name = "tfc-agent"
    Role = "terraform-cloud-agent"
  }
}

# IAM Role สำหรับ Agent (ใช้ assume role แทน access keys)
resource "aws_iam_role" "tfc_agent" {
  name = "tfc-agent-role"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Action = "sts:AssumeRole"
      Effect = "Allow"
      Principal = {
        Service = "ec2.amazonaws.com"
      }
    }]
  })
}

resource "aws_iam_role_policy_attachment" "tfc_agent_admin" {
  role       = aws_iam_role.tfc_agent.name
  policy_arn = "arn:aws:iam::aws:policy/AdministratorAccess"
  # ใน production ควรใช้ least privilege policy แทน
}
```

---

## Step-by-Step TFC Workspace Setup Guide

```markdown
## Complete TFC Setup Walkthrough

### Step 1: สร้าง Organization
1. ไปที่ https://app.terraform.io
2. Create new organization: "mycompany"
3. เลือก plan tier (Free ได้ 500 resources)

### Step 2: เชื่อมต่อ GitHub
1. Organization Settings → VCS Providers → Add VCS Provider
2. เลือก GitHub
3. คลิก "Register a new OAuth Application" ใน GitHub
4. ใส่ callback URL ที่ TFC ให้
5. Copy Client ID และ Client Secret กลับมา TFC

### Step 3: สร้าง Workspace
1. Workspaces → New Workspace
2. เลือก "Version Control Workflow"
3. เลือก GitHub repo
4. ระบุ Working Directory
5. กำหนด Terraform version
6. เลือก Auto Apply: No (manual approval)

### Step 4: กำหนด Variables
1. เปิด Workspace
2. Variables tab
3. Add Terraform Variables:
   - environment = "production"
   - vpc_cidr = "10.0.0.0/16"
4. Add Environment Variables (Sensitive):
   - AWS_ACCESS_KEY_ID
   - AWS_SECRET_ACCESS_KEY
   OR ใช้ Dynamic Credentials (OIDC)

### Step 5: รันครั้งแรก
1. Queue Plan จาก UI
2. หรือ Push code ไปยัง configured branch
3. Review plan output
4. Confirm and Apply

### Step 6: ตั้งค่า Team Access
1. Organization → Teams → Create Team
2. ตั้งชื่อ: "developers"
3. กำหนด Organization Access
4. เพิ่ม members
5. กำหนด Workspace access:
   - developers: Plan access
   - ops-team: Apply access
   - infrastructure: Admin access
```

---

## Dynamic Provider Credentials (OIDC) - แนะนำมากกว่า Access Keys

```hcl
# Trust policy สำหรับ AWS
resource "aws_iam_role" "tfc_role" {
  name = "tfc-deployment-role"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect = "Allow"
      Principal = {
        Federated = "arn:aws:iam::${data.aws_caller_identity.current.account_id}:oidc-provider/app.terraform.io"
      }
      Action = "sts:AssumeRoleWithWebIdentity"
      Condition = {
        StringEquals = {
          "app.terraform.io:aud" = ["aws.workload.identity"]
        }
        StringLike = {
          "app.terraform.io:sub" = "organization:mycompany:project:*:workspace:*:run_phase:*"
        }
      }
    }]
  })
}

# กำหนด permissions
resource "aws_iam_role_policy_attachment" "tfc_policy" {
  role       = aws_iam_role.tfc_role.name
  policy_arn = "arn:aws:iam::aws:policy/PowerUserAccess"
}

# OIDC Provider ใน AWS
resource "aws_iam_openid_connect_provider" "tfc" {
  url = "https://app.terraform.io"

  client_id_list = ["aws.workload.identity"]

  thumbprint_list = [
    "9e99a48a9960b14926bb7f3b02e22da2b0ab7280"  # TFC certificate thumbprint
  ]
}
```

```
TFC Workspace Environment Variables (สำหรับ OIDC):
TFC_AWS_PROVIDER_AUTH = true
TFC_AWS_RUN_ROLE_ARN  = arn:aws:iam::123456789012:role/tfc-deployment-role
```

---

## สรุป (Summary)

Terraform Cloud ให้:

1. **Remote State** - ปลอดภัย, locked, versioned
2. **Remote Execution** - consistent environment
3. **VCS Integration** - GitOps workflow
4. **Team Management** - RBAC ที่ยืดหยุ่น
5. **Policy Enforcement** - Sentinel
6. **Cost Estimation** - รู้ค่าใช้จ่ายก่อน apply
7. **Notifications** - Slack, email alerts
8. **Private Registry** - share modules ภายในองค์กร

---

*จบ Part 074 - ในส่วนถัดไปจะเรียนรู้เรื่อง Terraform Enterprise Features*
