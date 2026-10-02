# Part 100: World-class Architecture Patterns (Steps 991-1000)

## ยอดวิชาความรู้ Terraform - Enterprise-grade Architecture

---

## บทนำ: ความสำเร็จใน 100 บทเรียน

ยินดีที่คุณมาถึงบทสุดท้าย! เราได้เรียนรู้เรื่อง Terraform มาตลอด 100 steps ตั้งแต่พื้นฐานจนถึง advanced patterns ในบทนี้เราจะรวมทุกอย่างเข้าด้วยกันและนำเสนอ patterns ระดับ Enterprise ที่ใช้ใน production จริงๆ

---

## Step 991: Multi-Account AWS Architecture (Landing Zone)

### ทำไมต้อง Multi-Account?

```
Single Account (ไม่แนะนำ):
─────────────────────────────────────────────────────────────
┌────────────────────────────────────────────────────────────┐
│                    AWS Account                              │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐  │
│  │   Dev    │  │ Staging  │  │   Prod   │  │ Security  │  │
│  │  VPC     │  │  VPC     │  │  VPC     │  │ Tools    │  │
│  └──────────┘  └──────────┘  └──────────┘  └──────────┘  │
└────────────────────────────────────────────────────────────┘
ปัญหา:
❌ ไม่มี blast radius isolation
❌ IAM permissions ซับซ้อน
❌ Cost tracking ยาก
❌ ไม่สามารถ enforce guardrails ได้
```

```
Multi-Account (AWS Landing Zone):
─────────────────────────────────────────────────────────────
                    AWS Organizations
                         │
          ┌──────────────┼──────────────────┐
          │              │                  │
   Management        Security Root          Infrastructure
     Account           Account              OU
     (root)         ┌──────────┐           ┌──────────┐
                    │  Log     │           │  Shared  │
                    │ Archive  │           │ Services │
                    └──────────┘           └──────────┘
                    ┌──────────┐                │
                    │ Security │         Workloads OU
                    │  Tooling │        ┌─────────────┐
                    └──────────┘        │  Team A OU  │
                                        │  ┌────────┐  │
                                        │  │Dev Acc │  │
                                        │  ├────────┤  │
                                        │  │Stg Acc │  │
                                        │  ├────────┤  │
                                        │  │Prd Acc │  │
                                        │  └────────┘  │
                                        └─────────────┘
```

### Terraform สำหรับ AWS Organizations

```hcl
# organizations/main.tf

# ===== AWS Organizations =====
resource "aws_organizations_organization" "main" {
  aws_service_access_principals = [
    "cloudtrail.amazonaws.com",
    "config.amazonaws.com",
    "guardduty.amazonaws.com",
    "securityhub.amazonaws.com",
    "sso.amazonaws.com",
    "controltower.amazonaws.com",
  ]
  
  feature_set                   = "ALL"
  enabled_policy_types = [
    "SERVICE_CONTROL_POLICY",
    "TAG_POLICY",
    "BACKUP_POLICY",
  ]
}

# ===== Organizational Units =====
resource "aws_organizations_organizational_unit" "workloads" {
  name      = "Workloads"
  parent_id = aws_organizations_organization.main.roots[0].id
}

resource "aws_organizations_organizational_unit" "security" {
  name      = "Security"
  parent_id = aws_organizations_organization.main.roots[0].id
}

resource "aws_organizations_organizational_unit" "infrastructure" {
  name      = "Infrastructure"
  parent_id = aws_organizations_organization.main.roots[0].id
}

# ===== Team OUs under Workloads =====
resource "aws_organizations_organizational_unit" "teams" {
  for_each = var.teams
  
  name      = each.key
  parent_id = aws_organizations_organizational_unit.workloads.id
}

# ===== Accounts =====
resource "aws_organizations_account" "workloads" {
  for_each = {
    for item in flatten([
      for team, envs in var.teams : [
        for env in envs.environments : {
          key     = "${team}-${env}"
          name    = "${var.org_name}-${team}-${env}"
          email   = "${team}-${env}@${var.domain}"
          team    = team
          env     = env
          ou_id   = aws_organizations_organizational_unit.teams[team].id
        }
      ]
    ]) : item.key => item
  }
  
  name      = each.value.name
  email     = each.value.email
  parent_id = each.value.ou_id
  
  tags = {
    Team        = each.value.team
    Environment = each.value.env
    ManagedBy   = "terraform"
  }
  
  lifecycle {
    # ป้องกัน accidental deletion
    prevent_destroy = true
    
    # email ไม่สามารถเปลี่ยนได้ใน Organizations
    ignore_changes = [email]
  }
}
```

### Service Control Policies (SCPs)

```hcl
# ===== SCP: Deny High-Risk Actions in Production =====
resource "aws_organizations_policy" "deny_dangerous_actions" {
  name        = "DenyDangerousActionsInProd"
  description = "Prevent high-risk actions in production accounts"
  type        = "SERVICE_CONTROL_POLICY"
  
  content = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Sid    = "DenyRootAccountActions"
        Effect = "Deny"
        Action = ["*"]
        Resource = ["*"]
        Condition = {
          StringLike = {
            "aws:PrincipalArn" = ["arn:aws:iam::*:root"]
          }
        }
      },
      {
        Sid    = "DenyRegionOutsideApproved"
        Effect = "Deny"
        NotAction = [
          "iam:*",
          "sts:*",
          "route53:*",
          "cloudfront:*",
          "waf:*",
          "support:*",
          "trustedadvisor:*",
        ]
        Resource = ["*"]
        Condition = {
          StringNotEquals = {
            "aws:RequestedRegion" = var.approved_regions
          }
        }
      },
      {
        Sid    = "RequireMFAForSensitiveActions"
        Effect = "Deny"
        Action = [
          "iam:DeleteUser",
          "iam:DeleteRole",
          "iam:DeletePolicy",
          "organizations:LeaveOrganization",
        ]
        Resource = ["*"]
        Condition = {
          BoolIfExists = {
            "aws:MultiFactorAuthPresent" = "false"
          }
        }
      }
    ]
  })
}

# ===== SCP: Deny Disabling Security Services =====
resource "aws_organizations_policy" "deny_disable_security" {
  name = "DenyDisableSecurityServices"
  type = "SERVICE_CONTROL_POLICY"
  
  content = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Deny"
        Action = [
          "cloudtrail:StopLogging",
          "cloudtrail:DeleteTrail",
          "guardduty:DeleteDetector",
          "guardduty:DisassociateFromMasterAccount",
          "securityhub:DisableSecurityHub",
          "config:DeleteConfigRule",
          "config:DeleteConfigurationRecorder",
          "config:StopConfigurationRecorder",
        ]
        Resource = ["*"]
      }
    ]
  })
}
```

---

## Step 992: Zero-Trust Network Architecture

### ออกแบบ Network แบบ Zero-Trust

```
Zero-Trust Network Architecture:
─────────────────────────────────────────────────────────────
Internet
    │
    ▼
┌─────────────────────────────────────────────────────────┐
│  AWS WAF + CloudFront + Shield Advanced                 │
└─────────────────────────────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────────────────────────────┐
│  Public Subnet (Minimal)                                │
│  ├── Application Load Balancer (HTTPS only)             │
│  └── NAT Gateway (outbound only)                        │
└─────────────────────────────────────────────────────────┘
    │ (TLS 1.2+ only, mTLS between services)
    ▼
┌─────────────────────────────────────────────────────────┐
│  Private Subnet - Application Tier                      │
│  ├── ECS/EKS Services (no public IPs)                   │
│  ├── Service Mesh (AWS App Mesh / Istio)                │
│  └── Strict Security Groups (SG-to-SG refs only)       │
└─────────────────────────────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────────────────────────────┐
│  Private Subnet - Data Tier                             │
│  ├── RDS (Multi-AZ, encrypted)                          │
│  ├── ElastiCache (encrypted)                            │
│  └── No internet connectivity                           │
└─────────────────────────────────────────────────────────┘
    │
    ▼ (via VPC Endpoints - no internet!)
┌─────────────────────────────────────────────────────────┐
│  AWS Services (S3, DynamoDB, SSM, KMS, etc.)            │
│  Access via Private VPC Endpoints ONLY                  │
└─────────────────────────────────────────────────────────┘
```

### Terraform สำหรับ Zero-Trust

```hcl
# ===== VPC Endpoints สำหรับ AWS Services =====
# ทำให้ traffic ไม่ออกอินเทอร์เน็ต!

locals {
  # Services ที่ต้องการ Interface Endpoints
  interface_endpoints = {
    ssm = {
      service_name = "com.amazonaws.${var.aws_region}.ssm"
    }
    ssmmessages = {
      service_name = "com.amazonaws.${var.aws_region}.ssmmessages"
    }
    ec2messages = {
      service_name = "com.amazonaws.${var.aws_region}.ec2messages"
    }
    secretsmanager = {
      service_name = "com.amazonaws.${var.aws_region}.secretsmanager"
    }
    kms = {
      service_name = "com.amazonaws.${var.aws_region}.kms"
    }
    ecr_api = {
      service_name = "com.amazonaws.${var.aws_region}.ecr.api"
    }
    ecr_dkr = {
      service_name = "com.amazonaws.${var.aws_region}.ecr.dkr"
    }
    logs = {
      service_name = "com.amazonaws.${var.aws_region}.logs"
    }
    monitoring = {
      service_name = "com.amazonaws.${var.aws_region}.monitoring"
    }
    sts = {
      service_name = "com.amazonaws.${var.aws_region}.sts"
    }
  }
}

resource "aws_vpc_endpoint" "interface" {
  for_each = local.interface_endpoints
  
  vpc_id              = aws_vpc.main.id
  service_name        = each.value.service_name
  vpc_endpoint_type   = "Interface"
  subnet_ids          = aws_subnet.private[*].id
  security_group_ids  = [aws_security_group.vpc_endpoints.id]
  private_dns_enabled = true  # ✅ ใช้ private DNS
  
  tags = {
    Name        = "${var.environment}-endpoint-${each.key}"
    Environment = var.environment
  }
}

# S3 Gateway Endpoint (ฟรี)
resource "aws_vpc_endpoint" "s3" {
  vpc_id            = aws_vpc.main.id
  service_name      = "com.amazonaws.${var.aws_region}.s3"
  vpc_endpoint_type = "Gateway"
  route_table_ids   = concat(
    aws_route_table.private[*].id,
    aws_route_table.database[*].id
  )
  
  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect    = "Allow"
        Principal = "*"
        Action    = "s3:*"
        Resource  = [
          "arn:aws:s3:::my-app-${var.environment}-*",
          "arn:aws:s3:::my-app-${var.environment}-*/*",
        ]
      }
    ]
  })
}

# ===== Security Group - SG-to-SG references =====
# ไม่ใช้ CIDR blocks เลย!

resource "aws_security_group" "app" {
  name        = "${var.environment}-app-sg"
  description = "App tier security group"
  vpc_id      = aws_vpc.main.id
  
  # ALB → App (HTTPS)
  ingress {
    from_port       = 8080
    to_port         = 8080
    protocol        = "tcp"
    security_groups = [aws_security_group.alb.id]  # ✅ SG reference
    description     = "Allow HTTPS from ALB"
  }
  
  # App → DB
  egress {
    from_port       = 5432
    to_port         = 5432
    protocol        = "tcp"
    security_groups = [aws_security_group.database.id]  # ✅ SG reference
    description     = "Allow PostgreSQL to database tier"
  }
  
  # App → AWS Services (via VPC Endpoints)
  egress {
    from_port       = 443
    to_port         = 443
    protocol        = "tcp"
    security_groups = [aws_security_group.vpc_endpoints.id]
    description     = "Allow HTTPS to VPC endpoints"
  }
  
  tags = {
    Name        = "${var.environment}-app-sg"
    Environment = var.environment
  }
}
```

---

## Step 993: Platform Engineering with Terraform

### Internal Developer Platform (IDP)

```
Platform Engineering:
─────────────────────────────────────────────────────────────
Goal: ให้ Developer สามารถ provision infrastructure เองได้อย่างปลอดภัย

Traditional:
  Dev → Ticket to Ops → Wait 2 weeks → Get infrastructure

Platform Engineering (IDP):
  Dev → Self-service portal → Select template → Click deploy
  → Automated provisioning (via Terraform modules) → Done in minutes
─────────────────────────────────────────────────────────────
```

### Golden Path Templates

```hcl
# modules/golden-paths/microservice/main.tf
# "Golden Path" สำหรับ deploy microservice ใหม่
# Developer ไม่ต้องรู้เรื่อง infrastructure ลึกๆ

module "microservice" {
  source  = "registry.company.com/platform/microservice/aws"
  version = "~> 2.0"
  
  # ===== Developer inputs (simple) =====
  service_name = "payment-service"
  team         = "payments"
  environment  = "prod"
  
  # Resource sizing (T-shirt sizes)
  size = "medium"  # small, medium, large, xlarge
  
  # Docker image
  container_image = "123456789012.dkr.ecr.ap-southeast-1.amazonaws.com/payment-service:v2.1.0"
  
  # Scaling
  min_replicas = 2
  max_replicas = 10
  
  # Dependencies
  needs_database    = true
  needs_redis       = true
  needs_sqs         = true
}

# ===== Module ทำสิ่งเหล่านี้อัตโนมัติ =====
# ✅ ECS Task Definition + Service
# ✅ Load Balancer + Target Group
# ✅ Auto Scaling policies
# ✅ CloudWatch alarms
# ✅ IAM roles + policies (least privilege)
# ✅ Security Groups
# ✅ Service discovery
# ✅ RDS instance (if needs_database = true)
# ✅ ElastiCache (if needs_redis = true)
# ✅ SQS queues (if needs_sqs = true)
# ✅ Cost allocation tags
# ✅ Compliance tags
```

### Backstage Integration

```hcl
# backstage-catalog-info.yaml (ไม่ใช่ Terraform แต่เกี่ยวข้อง)
# เก็บใน root ของ service repository
apiVersion: backstage.io/v1alpha1
kind: Component
metadata:
  name: payment-service
  annotations:
    github.com/project-slug: myorg/payment-service
    backstage.io/techdocs-ref: dir:.
    # Terraform integration
    terraform.io/workspace: prod-payment-service
    infracost.io/project: payment-service
spec:
  type: service
  owner: team-payments
  lifecycle: production
  system: payments-platform
  dependsOn:
    - resource:payment-database
    - resource:payment-redis
```

---

## Step 994: GitOps at Enterprise Scale

### Monorepo Strategy

```
Enterprise Monorepo:
─────────────────────────────────────────────────────────────
infrastructure/
├── platform/
│   ├── organizations/        # AWS Organizations, SCPs
│   ├── security-baseline/    # GuardDuty, SecurityHub, Config
│   ├── shared-services/      # DNS, ACM, Transit Gateway
│   └── identity/             # SSO, IAM Identity Center
│
├── workloads/
│   ├── team-payments/
│   │   ├── dev/
│   │   ├── staging/
│   │   └── prod/
│   ├── team-orders/
│   │   └── ...
│   └── team-users/
│       └── ...
│
├── modules/
│   ├── networking/
│   ├── eks/
│   ├── rds/
│   ├── golden-paths/         # Platform team modules
│   │   ├── microservice/
│   │   └── data-pipeline/
│   └── compliance/           # Compliance modules
│
├── policies/
│   ├── checkov/
│   ├── sentinel/
│   └── opa/
│
├── .github/
│   └── workflows/
│       ├── validate.yml
│       ├── plan.yml
│       ├── apply.yml
│       ├── security.yml
│       └── drift.yml
│
└── docs/
    ├── runbooks/
    └── architecture/
```

### Environment Promotion Pipeline

```yaml
# .github/workflows/promotion.yml
name: Environment Promotion Pipeline

on:
  workflow_dispatch:
    inputs:
      service:
        description: 'Service to promote'
        required: true
      version:
        description: 'Version/Tag to promote'
        required: true
      from_env:
        type: choice
        options: [dev, staging]
        default: staging
      to_env:
        type: choice
        options: [staging, prod]
        default: prod
      change_request:
        description: 'Change Request number (required for prod)'
        required: false

jobs:
  validate-promotion:
    name: Validate Promotion Request
    runs-on: ubuntu-latest
    
    steps:
      - name: Check CR for Production
        if: inputs.to_env == 'prod' && !inputs.change_request
        run: |
          echo "ERROR: Change Request number required for production promotion!"
          exit 1
      
      - name: Check Service Exists in Source
        run: |
          echo "Checking if ${{ inputs.service }}:${{ inputs.version }} exists in ${{ inputs.from_env }}"
          # Verify image exists in ECR, service is healthy, etc.
  
  promote:
    name: Promote ${{ inputs.service }} to ${{ inputs.to_env }}
    needs: validate-promotion
    runs-on: ubuntu-latest
    environment: ${{ inputs.to_env }}
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Update version
        run: |
          # อัพเดต image version ใน terraform.tfvars
          sed -i "s/image_tag = \".*\"/image_tag = \"${{ inputs.version }}\"/" \
            workloads/${{ inputs.service }}/${{ inputs.to_env }}/terraform.tfvars
      
      - name: Create PR for promotion
        uses: peter-evans/create-pull-request@v6
        with:
          title: "Promote ${{ inputs.service }} ${{ inputs.version }} to ${{ inputs.to_env }}"
          body: |
            ## Promotion Details
            
            - **Service**: ${{ inputs.service }}
            - **Version**: ${{ inputs.version }}
            - **From**: ${{ inputs.from_env }}
            - **To**: ${{ inputs.to_env }}
            - **Change Request**: ${{ inputs.change_request || 'N/A' }}
            - **Requested by**: ${{ github.actor }}
          branch: "promote/${{ inputs.service }}-${{ inputs.version }}-${{ inputs.to_env }}"
          labels: "promotion,${{ inputs.to_env }}"
```

---

## Step 995: Immutable Infrastructure

### Golden AMI Pipeline

```
Golden AMI Pipeline:
─────────────────────────────────────────────────────────────
                  Packer Build
                      │
                      ▼
Base AMI (Amazon Linux 2023)
          │
          ▼ hardening steps
┌─────────────────────────────────────────────────────────┐
│  Security Hardening:                                    │
│  - CIS Benchmark Level 2                               │
│  - Remove unnecessary packages                         │
│  - Enable auditd                                       │
│  - Configure SSH hardening                             │
│  - Install monitoring agents                           │
└─────────────────────────────────────────────────────────┘
          │
          ▼ security scanning
┌─────────────────────────────────────────────────────────┐
│  Vulnerability Scan:                                    │
│  - Trivy image scan                                    │
│  - AWS Inspector assessment                            │
│  - OVAL compliance check                               │
└─────────────────────────────────────────────────────────┘
          │
          ▼ if pass
Golden AMI tagged: golden-ami-2024-01-15-v1.2.3
          │
          ▼
Terraform uses this AMI
```

### Packer + Terraform Integration

```hcl
# data.tf - ดึง latest Golden AMI
data "aws_ami" "golden" {
  most_recent = true
  owners      = ["self"]  # AMIs ที่สร้างเอง
  
  filter {
    name   = "name"
    values = ["golden-ami-*"]
  }
  
  filter {
    name   = "tag:Status"
    values = ["approved"]  # ผ่าน security scan แล้ว
  }
  
  filter {
    name   = "tag:CISLevel"
    values = ["2"]
  }
}

# main.tf
resource "aws_launch_template" "app" {
  name_prefix   = "${var.environment}-app-"
  image_id      = data.aws_ami.golden.id  # ✅ ใช้ Golden AMI
  instance_type = var.instance_type
  
  # ไม่มี user data ที่ซับซ้อน - ทุกอย่างอยู่ใน AMI แล้ว
  user_data = base64encode(<<-EOT
    #!/bin/bash
    # Minimal startup - set environment-specific configs only
    echo "${var.environment}" > /etc/environment-name
    systemctl start app-service
  EOT
  )
  
  lifecycle {
    create_before_destroy = true
  }
  
  tag_specifications {
    resource_type = "instance"
    tags = {
      Name       = "${var.environment}-app"
      AMI        = data.aws_ami.golden.id
      AMIVersion = data.aws_ami.golden.tags["Version"]
    }
  }
}
```

---

## Step 996: Terraform at Scale (1000+ Resources)

### State Partitioning Strategy

```
State Partitioning สำหรับ Large Infrastructure:
─────────────────────────────────────────────────────────────
s3://my-terraform-state/
├── foundations/
│   ├── organizations/terraform.tfstate    # ~50 resources
│   ├── security-baseline/terraform.tfstate # ~30 resources
│   └── shared-services/terraform.tfstate   # ~40 resources
│
├── networking/
│   ├── dev/terraform.tfstate              # ~80 resources
│   ├── staging/terraform.tfstate          # ~80 resources
│   └── prod/terraform.tfstate             # ~100 resources
│
├── eks/
│   ├── dev/terraform.tfstate              # ~150 resources
│   ├── staging/terraform.tfstate          # ~150 resources
│   └── prod/terraform.tfstate             # ~200 resources
│
├── workloads/
│   ├── payments/
│   │   ├── dev/terraform.tfstate          # ~50 resources
│   │   └── prod/terraform.tfstate         # ~60 resources
│   └── orders/
│       └── ...
│
ทั้งหมดกว่า 1,000 resources แต่แต่ละ state file มี < 200 resources
```

### Terragrunt สำหรับ DRY Configs

```hcl
# terragrunt.hcl (root)
locals {
  # อ่าน environment config
  account_vars = read_terragrunt_config(find_in_parent_folders("account.hcl"))
  region_vars  = read_terragrunt_config(find_in_parent_folders("region.hcl"))
  env_vars     = read_terragrunt_config(find_in_parent_folders("env.hcl"))
  
  account_id  = local.account_vars.locals.account_id
  aws_region  = local.region_vars.locals.aws_region
  environment = local.env_vars.locals.environment
}

# Remote state configuration
remote_state {
  backend = "s3"
  config = {
    bucket         = "terraform-state-${local.account_id}"
    key            = "${path_relative_to_include()}/terraform.tfstate"
    region         = local.aws_region
    encrypt        = true
    dynamodb_table = "terraform-state-lock"
  }
  generate = {
    path      = "backend.tf"
    if_exists = "overwrite_terraformignore"
  }
}

# Generate provider
generate "provider" {
  path      = "provider.tf"
  if_exists = "overwrite_terraformignore"
  contents = <<EOF
provider "aws" {
  region = "${local.aws_region}"
  
  assume_role {
    role_arn = "arn:aws:iam::${local.account_id}:role/terraform-execution"
  }
  
  default_tags {
    tags = {
      Environment = "${local.environment}"
      ManagedBy   = "terragrunt"
    }
  }
}
EOF
}

# Common inputs
inputs = merge(
  local.account_vars.locals,
  local.region_vars.locals,
  local.env_vars.locals,
)
```

```hcl
# prod/us-east-1/prod/vpc/terragrunt.hcl
include "root" {
  path = find_in_parent_folders()
}

terraform {
  source = "../../../../modules//networking"
}

inputs = {
  vpc_cidr           = "10.0.0.0/16"
  availability_zones = ["us-east-1a", "us-east-1b", "us-east-1c"]
  
  private_subnets = ["10.0.0.0/24", "10.0.1.0/24", "10.0.2.0/24"]
  public_subnets  = ["10.0.100.0/24", "10.0.101.0/24", "10.0.102.0/24"]
}
```

---

## Step 997: Compliance Automation at Scale

### AWS Config + Auto Remediation

```hcl
# ===== AWS Config Rules =====
resource "aws_config_config_rule" "s3_encryption" {
  name        = "s3-bucket-server-side-encryption-enabled"
  description = "Checks that your S3 buckets have encryption at rest"
  
  source {
    owner             = "AWS"
    source_identifier = "S3_BUCKET_SERVER_SIDE_ENCRYPTION_ENABLED"
  }
}

# ===== Auto Remediation =====
resource "aws_config_remediation_configuration" "s3_encryption" {
  config_rule_name = aws_config_config_rule.s3_encryption.name
  
  target_type       = "SSM_DOCUMENT"
  target_id         = "AWSConfigRemediation-EnableS3BucketDefaultEncryption"
  
  automatic = true  # ✅ Auto-remediate!
  
  maximum_automatic_attempts = 3
  retry_attempt_seconds      = 60
  
  parameter {
    name           = "AutomationAssumeRole"
    static_value   = aws_iam_role.config_remediation.arn
  }
  
  parameter {
    name           = "BucketName"
    resource_value = "RESOURCE_ID"
  }
  
  parameter {
    name         = "SSEAlgorithm"
    static_value = "aws:kms"
  }
}

# ===== Security Hub =====
resource "aws_securityhub_account" "main" {}

resource "aws_securityhub_standards_subscription" "cis" {
  standards_arn = "arn:aws:securityhub:${var.aws_region}::standards/cis-aws-foundations-benchmark/v/1.4.0"
  
  depends_on = [aws_securityhub_account.main]
}

resource "aws_securityhub_standards_subscription" "aws_foundational" {
  standards_arn = "arn:aws:securityhub:${var.aws_region}::standards/aws-foundational-security-best-practices/v/1.0.0"
  
  depends_on = [aws_securityhub_account.main]
}
```

### Compliance Dashboard

```hcl
# ===== Lambda สำหรับ Compliance Report =====
resource "aws_lambda_function" "compliance_report" {
  function_name = "${var.environment}-compliance-report"
  role          = aws_iam_role.lambda_compliance.arn
  
  filename         = data.archive_file.compliance_report.output_path
  source_code_hash = data.archive_file.compliance_report.output_base64sha256
  
  runtime = "python3.11"
  handler = "compliance_report.lambda_handler"
  timeout = 300
  
  environment {
    variables = {
      S3_BUCKET        = aws_s3_bucket.compliance_reports.id
      SLACK_WEBHOOK    = var.slack_webhook_url
    }
  }
}

# รัน Compliance Report ทุกวันจันทร์
resource "aws_cloudwatch_event_rule" "weekly_compliance" {
  name                = "${var.environment}-weekly-compliance"
  schedule_expression = "cron(0 8 ? * MON *)"  # ทุกวันจันทร์ 8am UTC
}
```

---

## Step 998: Disaster Recovery with Terraform

### Multi-Region Active-Passive DR

```
DR Architecture:
─────────────────────────────────────────────────────────────
                         Route53 (Health Check)
                              │
              ┌───────────────┴───────────────┐
              │                               │
    Primary Region                      DR Region
    (ap-southeast-1)                    (ap-southeast-2)
              │                               │
    ┌─────────────────┐            ┌─────────────────┐
    │  Active Stack   │            │  Standby Stack  │
    │  - EKS cluster  │◄──────────►│  - EKS cluster  │
    │  - RDS Primary  │  Replication  - RDS Read      │
    │  - ElastiCache  │            │  - S3 (repl'd)  │
    └─────────────────┘            └─────────────────┘
    
    RPO: < 15 minutes
    RTO: < 1 hour (automated), < 30 min (with practice)
```

```hcl
# ===== Multi-Region Setup =====
# Primary region
provider "aws" {
  alias  = "primary"
  region = "ap-southeast-1"
}

# DR region
provider "aws" {
  alias  = "dr"
  region = "ap-southeast-2"
}

# ===== RDS with Read Replica for DR =====
resource "aws_db_instance" "primary" {
  provider = aws.primary
  
  identifier        = "${var.environment}-db-primary"
  engine            = "postgres"
  engine_version    = "15.3"
  instance_class    = "db.r5.2xlarge"
  
  multi_az              = true
  storage_encrypted     = true
  deletion_protection   = true
  backup_retention_period = 30
  
  # สำคัญ: เปิด cross-region replication
  backup_window         = "03:00-04:00"  # UTC
  maintenance_window    = "sun:04:00-sun:05:00"
  
  tags = { Region = "primary" }
}

resource "aws_db_instance_automated_backups_replication" "dr" {
  provider = aws.dr
  
  source_db_instance_arn  = aws_db_instance.primary.arn
  retention_period        = 14  # เก็บ backup 14 วัน ใน DR region
  kms_key_id              = aws_kms_key.dr_rds.arn
}

# ===== S3 Cross-Region Replication =====
resource "aws_s3_bucket_replication_configuration" "dr" {
  provider = aws.primary
  
  bucket = aws_s3_bucket.app_data.id
  role   = aws_iam_role.s3_replication.arn
  
  rule {
    id     = "replicate-to-dr"
    status = "Enabled"
    
    destination {
      bucket        = aws_s3_bucket.app_data_dr.arn
      storage_class = "STANDARD_IA"  # ถูกกว่าใน DR
      
      replication_time {
        status = "Enabled"
        time {
          minutes = 15  # SLA 15 นาที
        }
      }
      
      metrics {
        status = "Enabled"
        event_threshold {
          minutes = 15
        }
      }
    }
  }
}

# ===== Route53 Failover =====
resource "aws_route53_health_check" "primary" {
  fqdn              = "app.${var.environment}.company.com"
  port              = 443
  type              = "HTTPS"
  resource_path     = "/health"
  failure_threshold = 3
  request_interval  = 10
  
  regions = [
    "ap-southeast-1",
    "us-west-2",
    "eu-west-1",
  ]
  
  tags = { Name = "primary-health-check" }
}

resource "aws_route53_record" "primary" {
  zone_id = data.aws_route53_zone.main.zone_id
  name    = "app"
  type    = "A"
  
  set_identifier = "primary"
  
  failover_routing_policy {
    type = "PRIMARY"
  }
  
  health_check_id = aws_route53_health_check.primary.id
  
  alias {
    name                   = aws_lb.primary.dns_name
    zone_id                = aws_lb.primary.zone_id
    evaluate_target_health = true
  }
}

resource "aws_route53_record" "dr" {
  provider = aws.dr
  zone_id  = data.aws_route53_zone.main.zone_id
  name     = "app"
  type     = "A"
  
  set_identifier = "dr"
  
  failover_routing_policy {
    type = "SECONDARY"
  }
  
  alias {
    name                   = aws_lb.dr.dns_name
    zone_id                = aws_lb.dr.zone_id
    evaluate_target_health = true
  }
}
```

---

## Step 999: DR Runbook Automation

### Automated Failover Lambda

```hcl
# ===== DR Runbook Lambda =====
data "archive_file" "dr_runbook" {
  type        = "zip"
  output_path = "/tmp/dr_runbook.zip"
  
  source {
    content  = <<-PYTHON
import boto3
import json
import os
import time

def lambda_handler(event, context):
    """
    Automated DR failover runbook.
    Triggered by CloudWatch Alarm or manual invocation.
    """
    action = event.get('action', 'status')
    
    if action == 'failover':
        return execute_failover()
    elif action == 'failback':
        return execute_failback()
    else:
        return get_dr_status()

def execute_failover():
    """Execute DR failover to secondary region"""
    steps = []
    
    # Step 1: Verify primary is actually down
    steps.append(verify_primary_health())
    
    # Step 2: Promote RDS Read Replica to Primary
    rds = boto3.client('rds', region_name=os.environ['DR_REGION'])
    rds.promote_read_replica(
        DBInstanceIdentifier=os.environ['DR_DB_IDENTIFIER']
    )
    steps.append({'step': 'rds_promotion', 'status': 'initiated'})
    
    # Step 3: Scale up DR EKS cluster
    asg = boto3.client('autoscaling', region_name=os.environ['DR_REGION'])
    asg.update_auto_scaling_group(
        AutoScalingGroupName=os.environ['DR_ASG_NAME'],
        MinSize=int(os.environ['DR_MIN_SIZE']),
        DesiredCapacity=int(os.environ['DR_DESIRED_SIZE']),
        MaxSize=int(os.environ['DR_MAX_SIZE']),
    )
    steps.append({'step': 'scale_up_dr', 'status': 'complete'})
    
    # Step 4: Update Route53 to point to DR
    # (Manual step - requires approval)
    steps.append({
        'step': 'dns_failover',
        'status': 'awaiting_approval',
        'instructions': f"Run: aws route53 change-resource-record-sets..."
    })
    
    # Step 5: Notify team
    sns = boto3.client('sns')
    sns.publish(
        TopicArn=os.environ['ALERT_TOPIC'],
        Subject='🚨 DR FAILOVER INITIATED',
        Message=json.dumps({'steps': steps}, indent=2)
    )
    
    return {'status': 'failover_initiated', 'steps': steps}

def verify_primary_health():
    # ตรวจสอบว่า primary จริงๆ down
    import urllib.request
    try:
        urllib.request.urlopen(
            f"https://{os.environ['PRIMARY_ENDPOINT']}/health",
            timeout=5
        )
        return {'step': 'verify_primary', 'status': 'primary_is_up'}
    except:
        return {'step': 'verify_primary', 'status': 'primary_confirmed_down'}
    PYTHON
    filename = "dr_runbook.py"
  }
}

resource "aws_lambda_function" "dr_runbook" {
  function_name    = "${var.environment}-dr-runbook"
  role             = aws_iam_role.dr_lambda.arn
  filename         = data.archive_file.dr_runbook.output_path
  source_code_hash = data.archive_file.dr_runbook.output_base64sha256
  runtime          = "python3.11"
  handler          = "dr_runbook.lambda_handler"
  timeout          = 300
  
  environment {
    variables = {
      DR_REGION       = "ap-southeast-2"
      DR_DB_IDENTIFIER = "${var.environment}-db-dr"
      DR_ASG_NAME     = "${var.environment}-dr-asg"
      DR_MIN_SIZE     = "2"
      DR_DESIRED_SIZE = "5"
      DR_MAX_SIZE     = "20"
      ALERT_TOPIC     = aws_sns_topic.dr_alerts.arn
      PRIMARY_ENDPOINT = "app.${var.environment}.company.com"
    }
  }
}
```

---

## Step 1000: The Complete Picture - Enterprise Terraform Architecture

### สรุปสิ่งที่เราเรียนมาทั้ง 100 Steps

```
Terraform Enterprise Architecture Stack:
─────────────────────────────────────────────────────────────

Layer 8: Governance & Compliance
  ┌─────────────────────────────────────────────────────────┐
  │ AWS Organizations + SCPs + Tag Policies                 │
  │ AWS Config + Security Hub + GuardDuty                   │
  │ Compliance reports + Auto-remediation                  │
  └─────────────────────────────────────────────────────────┘

Layer 7: Developer Experience
  ┌─────────────────────────────────────────────────────────┐
  │ Backstage Developer Portal                              │
  │ Golden Path templates                                   │
  │ Self-service infrastructure                            │
  │ Module catalog + documentation                         │
  └─────────────────────────────────────────────────────────┘

Layer 6: GitOps Automation
  ┌─────────────────────────────────────────────────────────┐
  │ GitHub Actions / Atlantis                               │
  │ PR-based workflows                                      │
  │ Automated plan + apply                                  │
  │ Drift detection                                         │
  └─────────────────────────────────────────────────────────┘

Layer 5: Security Pipeline
  ┌─────────────────────────────────────────────────────────┐
  │ Pre-commit hooks (TFSec, detect-secrets)               │
  │ CI security scan (Checkov + Trivy + Snyk)              │
  │ Custom policies (Rego + Python)                        │
  │ SARIF upload to GitHub Security                        │
  └─────────────────────────────────────────────────────────┘

Layer 4: Cost Management
  ┌─────────────────────────────────────────────────────────┐
  │ Infracost in CI/CD                                      │
  │ Budget alerts + anomaly detection                       │
  │ Spot instances + scheduled stop/start                  │
  │ Cost allocation tags                                    │
  └─────────────────────────────────────────────────────────┘

Layer 3: Code Quality
  ┌─────────────────────────────────────────────────────────┐
  │ Terraform fmt + validate + tflint                       │
  │ Module versioning                                       │
  │ Patterns: Factory, YAML-driven, Secrets                │
  │ Testing: Terratest, Checkov, conftest                  │
  └─────────────────────────────────────────────────────────┘

Layer 2: Infrastructure Modules
  ┌─────────────────────────────────────────────────────────┐
  │ Reusable modules per component                         │
  │ Multi-account, multi-region setup                      │
  │ Zero-trust networking                                   │
  │ Immutable infrastructure                               │
  └─────────────────────────────────────────────────────────┘

Layer 1: Foundation
  ┌─────────────────────────────────────────────────────────┐
  │ Remote state (S3 + DynamoDB)                            │
  │ Workspaces per environment                              │
  │ Provider configuration                                  │
  │ Terraform version management                            │
  └─────────────────────────────────────────────────────────┘
```

### Technology Stack Summary

```
ภาษาและ Tools:
─────────────────────────────────────────────────────────────
Infrastructure as Code:
  ✅ Terraform (HCL)
  ✅ Terragrunt (DRY wrapper)
  ✅ Packer (Golden AMIs)

Security:
  ✅ TFSec/Trivy (IaC scanning)
  ✅ Checkov (Policy-as-Code)
  ✅ Snyk IaC (Developer security)
  ✅ Terrascan (Multi-IaC)
  ✅ Gitleaks (Secret scanning)
  ✅ OPA/Rego (Custom policies)
  ✅ Sentinel (TFC policies)
  ✅ conftest (Testing policies)

Automation:
  ✅ GitHub Actions (CI/CD)
  ✅ Atlantis (PR automation)
  ✅ Pre-commit hooks

Cost:
  ✅ Infracost (Cost estimation)
  ✅ AWS Budgets (Alerts)

Observability:
  ✅ AWS CloudWatch
  ✅ AWS Config
  ✅ Security Hub
  ✅ GuardDuty

Developer Experience:
  ✅ Backstage (IDP)
  ✅ Module Registry
  ✅ Golden Paths
```

---

## คำแนะนำสำหรับการ Learning ต่อ

### Road to Mastery

```
Junior (1-6 เดือน):
  ✅ Terraform basics (HCL syntax, resources, variables)
  ✅ Remote state
  ✅ Basic modules
  ✅ fmt, validate, plan, apply
  ✅ ทำ Project แรก (ใน dev environment)

Mid (6-18 เดือน):
  ✅ Advanced modules
  ✅ Workspaces
  ✅ CI/CD integration
  ✅ Security scanning (Checkov, TFSec)
  ✅ State management
  ✅ Testing (Terratest)

Senior (18-36 เดือน):
  ✅ Multi-account architecture
  ✅ Custom security policies
  ✅ GitOps workflows (Atlantis)
  ✅ Cost optimization
  ✅ Platform Engineering
  ✅ Disaster Recovery automation

Expert (3+ ปี):
  ✅ Enterprise Landing Zone
  ✅ Zero-Trust Architecture
  ✅ Compliance automation
  ✅ Internal Developer Platforms
  ✅ ดูแล Terraform modules ที่ใช้ใน organization
  ✅ Mentor developers อื่นๆ
```

### Certifications ที่แนะนำ

```
1. HashiCorp Terraform Associate (003)
   - เหมาะสำหรับ Junior → Mid level
   - ราคา: $70 USD
   - เตรียมตัว: 4-8 สัปดาห์

2. AWS Solutions Architect Associate
   - เข้าใจ AWS services ที่ Terraform manage
   - ราคา: $150 USD
   - เตรียมตัว: 2-3 เดือน

3. AWS DevOps Professional
   - CI/CD, Infrastructure automation
   - ราคา: $300 USD
   - เตรียมตัว: 3-4 เดือน

4. HashiCorp Terraform Professional (Pro)
   - ต้องมีประสบการณ์จริง
   - Advanced topics
```

---

## บทสรุปสุดท้าย

```
สิ่งที่สำคัญที่สุดที่เรียนรู้มาตลอด 100 Steps:
─────────────────────────────────────────────────────────────

1. Infrastructure as Code ไม่ใช่แค่ tool
   - เป็น practice และ mindset
   - ทุกอย่าง declarative, version-controlled

2. Security ต้องเป็น First-Class Citizen
   - Security ทุก stage: local → CI → runtime
   - Custom policies สำหรับ business requirements

3. Teams ต้องทำงานร่วมกันได้
   - GitOps + PR reviews
   - Clear ownership
   - Self-service แต่มี guardrails

4. Cost ต้องมองเห็นและควบคุมได้
   - Infracost ก่อน apply
   - Budget alerts
   - Regular cost review

5. Complexity ต้องจัดการ
   - State partitioning
   - Module design
   - Documentation

6. Infrastructure ต้องพร้อมรับมือกับ failure
   - Multi-AZ, Multi-Region
   - Automated DR
   - Tested runbooks

─────────────────────────────────────────────────────────────
"Infrastructure as Code is not about the code, 
 it's about enabling your team to move fast safely."
─────────────────────────────────────────────────────────────
```

---

## ขอบคุณที่เรียนมาถึงที่สุด!

คุณได้เรียนรู้ Terraform ตั้งแต่พื้นฐานจนถึง Enterprise-grade architecture ครบ 1,000 steps แล้ว ทักษะเหล่านี้จะช่วยให้คุณ:

1. **สร้าง Infrastructure ได้อย่างมั่นใจ** ด้วย best practices และ security
2. **ทำงานเป็น Team ได้** ด้วย GitOps และ Atlantis
3. **ประหยัด Cost** ด้วย Infracost และ optimization patterns
4. **รับมือกับ Scale** ด้วย state partitioning และ Terragrunt
5. **Comply กับ Enterprise requirements** ด้วย custom policies และ SCPs

---

*จบหลักสูตร HCL/Terraform - 100 Parts, 1000 Steps*

*"The best infrastructure is invisible - it just works."*

---

*จบ Part 100: World-class Architecture Patterns*
