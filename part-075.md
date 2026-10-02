# Part 075: Terraform Enterprise Features (ขั้นตอนที่ 741-750)

## บทนำ (Introduction)

Terraform Enterprise (TFE) คือ self-hosted version ของ Terraform Cloud
เหมาะสำหรับองค์กรที่ต้องการ:
- ควบคุม data ทั้งหมด (compliance requirements)
- Air-gapped environments (ไม่มี internet)
- Custom integration กับ enterprise systems
- Advanced security features

---

## ขั้นตอนที่ 741: Terraform Enterprise vs Terraform Cloud

### Feature Comparison

```
Feature                    | TFC Free | TFC Plus | TFE
---------------------------|----------|----------|-----
Remote state               | ✓        | ✓        | ✓
Remote runs                | ✓        | ✓        | ✓
VCS integration            | ✓        | ✓        | ✓
Team management            | ✓        | ✓        | ✓
Sentinel (Advisory)        | ✓        | ✓        | ✓
Sentinel (Soft-Mandatory)  | ✗        | ✓        | ✓
Sentinel (Hard-Mandatory)  | ✗        | ✓        | ✓
Cost estimation            | ✓        | ✓        | ✓
Audit logging              | ✗        | ✓        | ✓
SAML SSO                   | ✗        | ✓        | ✓
Self-hosted                | ✗        | ✗        | ✓
Air-gapped                 | ✗        | ✗        | ✓
Private module registry    | ✓        | ✓        | ✓
No-code provisioning       | ✗        | ✓        | ✓
Dynamic credentials (OIDC) | ✓        | ✓        | ✓
Custom agents              | ✓        | ✓        | ✓
SLA                        | ✗        | 99.9%    | 99.9%
Resources managed          | 500 free | Unlimited| Unlimited
```

---

## ขั้นตอนที่ 742: TFE Deployment Options

### Option 1: Mounted Disk (Simplest)

```bash
# สำหรับ single-node deployment
# ข้อมูลเก็บใน local disk หรือ NFS mount

# System Requirements:
# - CPU: 8+ cores
# - RAM: 32+ GB
# - Disk: 40+ GB for application + space for data
# - OS: Ubuntu 20.04/22.04, RHEL 8/9, CentOS 8

# ดาวน์โหลด TFE installer
curl -o install.sh https://install.terraform.io/ptfe/stable
sudo bash install.sh
```

```bash
# ตั้งค่า via replicated.conf
cat > /etc/replicated.conf << 'EOF'
{
  "DaemonAuthenticationType": "password",
  "DaemonAuthenticationPassword": "CHANGE_ME",
  "TlsBootstrapType": "self-signed",
  "TlsBootstrapHostname": "tfe.mycompany.com",
  "BypassPreflightChecks": true
}
EOF
```

### Option 2: Podman/Docker Compose (Flexible External Services)

```yaml
# docker-compose.yml สำหรับ TFE
version: "3.8"

services:
  tfe:
    image: hashicorp/terraform-enterprise:v202405-1
    restart: unless-stopped
    environment:
      # License
      TFE_LICENSE: "${TFE_LICENSE}"

      # Hostname
      TFE_HOSTNAME: "tfe.mycompany.com"

      # Database (External PostgreSQL)
      TFE_DATABASE_USER: "tfe"
      TFE_DATABASE_PASSWORD: "${DB_PASSWORD}"
      TFE_DATABASE_HOST: "postgres.internal:5432"
      TFE_DATABASE_NAME: "tfe"
      TFE_DATABASE_PARAMETERS: "sslmode=require"

      # Object Storage (External S3 or MinIO)
      TFE_OBJECT_STORAGE_TYPE: "s3"
      TFE_OBJECT_STORAGE_S3_BUCKET: "tfe-storage"
      TFE_OBJECT_STORAGE_S3_REGION: "us-east-1"
      TFE_OBJECT_STORAGE_S3_USE_INSTANCE_PROFILE: "true"

      # Redis (for caching)
      TFE_REDIS_URL: "redis://redis.internal:6379"

      # TLS
      TFE_TLS_CERT_FILE: "/etc/ssl/tfe/cert.pem"
      TFE_TLS_KEY_FILE: "/etc/ssl/tfe/key.pem"
      TFE_TLS_CA_BUNDLE_FILE: "/etc/ssl/tfe/ca.pem"

      # Operational mode
      TFE_OPERATIONAL_MODE: "external"

      # Encryption password
      TFE_ENCRYPTION_PASSWORD: "${TFE_ENCRYPTION_PASSWORD}"

    volumes:
      - type: bind
        source: /etc/ssl/tfe
        target: /etc/ssl/tfe
      - type: volume
        source: tfe-logs
        target: /var/log/terraform-enterprise

    ports:
      - "443:443"
      - "80:80"

volumes:
  tfe-logs:
```

### Option 3: Kubernetes (Production Recommended)

```yaml
# values.yaml สำหรับ TFE Helm chart
replicaCount: 2

image:
  repository: hashicorp/terraform-enterprise
  tag: "v202405-1"

tfe:
  license: "${TFE_LICENSE}"
  hostname: "tfe.mycompany.com"
  
  # Operational mode
  operationalMode: "external"
  
  # Encryption
  encryptionPassword: "${ENCRYPTION_PASSWORD}"
  
  database:
    user: "tfe"
    password: "${DB_PASSWORD}"
    host: "postgres.internal"
    port: "5432"
    name: "tfe"
    sslMode: "require"
    
  objectStorage:
    type: "s3"
    bucket: "tfe-storage"
    region: "us-east-1"
    useInstanceProfile: true
    
  redis:
    url: "redis://redis.internal:6379"

service:
  type: LoadBalancer
  annotations:
    service.beta.kubernetes.io/aws-load-balancer-type: "nlb"

ingress:
  enabled: true
  className: "nginx"
  annotations:
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
  hosts:
    - host: tfe.mycompany.com
      paths:
        - path: /
          pathType: Prefix
  tls:
    - secretName: tfe-tls
      hosts:
        - tfe.mycompany.com

resources:
  requests:
    memory: "4Gi"
    cpu: "2"
  limits:
    memory: "8Gi"
    cpu: "4"

podAnnotations:
  iam.amazonaws.com/role: "tfe-role"  # สำหรับ kube2iam/IRSA
```

```bash
# Deploy TFE บน Kubernetes
helm repo add hashicorp https://helm.releases.hashicorp.com
helm repo update

kubectl create namespace tfe
kubectl create secret generic tfe-secrets \
  --from-literal=TFE_LICENSE="$TFE_LICENSE" \
  --from-literal=TFE_ENCRYPTION_PASSWORD="$ENCRYPTION_PASSWORD" \
  -n tfe

helm install terraform-enterprise hashicorp/terraform-enterprise \
  -n tfe \
  -f values.yaml
```

---

## ขั้นตอนที่ 743: TFE Air-Gapped Installation

### เตรียม Air-Gapped Bundle

```bash
# บน machine ที่มี internet access:

# 1. ดาวน์โหลด TFE package
curl -o tfe-airgap.tar.gz \
  "https://releases.hashicorp.com/terraform-enterprise/v202405-1/terraform-enterprise_v202405-1_linux_amd64.tar.gz"

# 2. ดาวน์โหลด Replicated (สำหรับ legacy deployment)
curl -o replicated.tar.gz \
  "https://s3.amazonaws.com/replicated-airgap-work/stable.tar.gz"

# 3. ดาวน์โหลด Container images
docker pull hashicorp/terraform-enterprise:v202405-1
docker save hashicorp/terraform-enterprise:v202405-1 \
  -o tfe-image.tar

# 4. สร้าง bundle
tar -czf tfe-airgap-bundle.tar.gz \
  tfe-airgap.tar.gz \
  tfe-image.tar \
  replicated.tar.gz

# 5. Transfer ไปยัง air-gapped environment
# (ผ่าน USB, secure transfer, etc.)
scp tfe-airgap-bundle.tar.gz tfe-server:/tmp/
```

```bash
# บน Air-Gapped Server:

# Extract bundle
tar -xzf /tmp/tfe-airgap-bundle.tar.gz

# Load Docker image
docker load -i tfe-image.tar

# Install TFE
tar -xzf tfe-airgap.tar.gz
./install.sh airgap
```

### Private Terraform Registry ใน Air-Gapped Environment

```bash
# ดาวน์โหลด provider binaries สำหรับ offline use
mkdir -p /var/tfe/providers/registry.terraform.io/hashicorp

# ดาวน์โหลด AWS provider
curl -o aws-provider.zip \
  "https://releases.hashicorp.com/terraform-provider-aws/5.50.0/terraform-provider-aws_5.50.0_linux_amd64.zip"

# สร้าง mirror structure
# /var/tfe/providers/registry.terraform.io/hashicorp/aws/5.50.0/linux_amd64/
```

```hcl
# network_mirror.tf - สำหรับ air-gapped environment
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

# .terraformrc หรือ terraform.rc
provider_installation {
  network_mirror {
    url     = "https://tfe.mycompany.com/api/v2/provider-versions/"
    include = ["hashicorp/*"]
  }

  direct {
    exclude = ["hashicorp/*"]
  }
}
```

---

## ขั้นตอนที่ 744: SAML SSO Integration

### SAML Configuration ใน TFE

```
TFE Admin Console → Settings → Authentication → SAML

Settings:
- Single Sign On URL: https://mycompany.okta.com/app/terraform/sso/saml
- Identity Provider Metadata XML: (paste XML from IdP)
- Attribute Mapping:
  - Username: user.username หรือ user.email
  - Site Admin: memberOf = "terraform-admins"

SP Metadata (ให้กับ IdP):
- Entity ID: https://tfe.mycompany.com/users/saml/metadata
- ACS URL:   https://tfe.mycompany.com/users/saml/auth
```

### Okta Integration

```xml
<!-- Okta SAML Configuration -->
<samlp:Response xmlns:samlp="urn:oasis:names:tc:SAML:2.0:protocol">
  <saml:Assertion>
    <saml:AttributeStatement>
      <!-- Username attribute -->
      <saml:Attribute Name="username">
        <saml:AttributeValue>john.doe@mycompany.com</saml:AttributeValue>
      </saml:Attribute>
      
      <!-- Groups for Team mapping -->
      <saml:Attribute Name="memberOf">
        <saml:AttributeValue>terraform-admins</saml:AttributeValue>
        <saml:AttributeValue>infrastructure-team</saml:AttributeValue>
      </saml:Attribute>
    </saml:AttributeStatement>
  </saml:Assertion>
</samlp:Response>
```

```hcl
# Terraform configuration สำหรับ SAML team mapping
resource "tfe_saml_settings" "main" {
  idp_cert      = file("okta-cert.pem")
  sso_api_token_session_timeout = 1209600  # 14 days
  sso_url       = "https://mycompany.okta.com/app/tfe/sso/saml"
  
  attr_username = "username"
  attr_groups   = "memberOf"
  attr_site_admin_role = "terraform-super-admins"
}
```

---

## ขั้นตอนที่ 745: Team และ RBAC Management

### Team Hierarchy

```
Organization
├── Owners (full admin)
│   └── Permission: Admin
├── Infrastructure Team
│   └── Permission: Manage workspaces
├── Developers
│   └── Permission: Plan only
└── Auditors
    └── Permission: Read state
```

### Workspace-level RBAC

```
Workspace Access Levels:
1. Read     - อ่าน state, plan output
2. Plan     - สร้าง speculative plan
3. Write    - Apply changes (ต้อง confirm)
4. Apply    - Apply โดยไม่ต้อง confirm
5. Admin    - ตั้งค่า workspace ทั้งหมด
```

```hcl
# TFE RBAC configuration ด้วย Terraform
provider "tfe" {
  token    = var.tfe_token
  hostname = "tfe.mycompany.com"
}

# สร้าง teams
resource "tfe_team" "infrastructure" {
  name         = "infrastructure"
  organization = "mycompany"

  organization_access {
    manage_workspaces   = true
    manage_policies     = true
    manage_vcs_settings = true
    manage_providers    = true
    manage_modules      = true
    manage_run_tasks    = true
    read_workspaces     = true
    read_projects       = true
  }
}

resource "tfe_team" "developers" {
  name         = "developers"
  organization = "mycompany"

  organization_access {
    read_workspaces = true
    read_projects   = true
  }
}

resource "tfe_team" "ops" {
  name         = "ops"
  organization = "mycompany"

  organization_access {
    manage_workspaces = false
    read_workspaces   = true
  }
}

# กำหนด workspace access
resource "tfe_team_access" "infra_networking_prod" {
  access       = "admin"
  team_id      = tfe_team.infrastructure.id
  workspace_id = tfe_workspace.networking_prod.id
}

resource "tfe_team_access" "dev_networking_staging" {
  access       = "plan"
  team_id      = tfe_team.developers.id
  workspace_id = tfe_workspace.networking_staging.id
}

# Custom access permissions
resource "tfe_team_access" "ops_networking_prod" {
  team_id      = tfe_team.ops.id
  workspace_id = tfe_workspace.networking_prod.id

  permissions {
    runs              = "apply"  # สามารถ apply
    variables         = "write"  # แก้ไข variables ได้
    state_versions    = "read"   # อ่าน state
    sentinel_mocks    = "none"   # ไม่เข้าถึง sentinel mocks
    workspace_locking = false
    run_tasks         = false
  }
}

# เพิ่ม members ลงใน team
resource "tfe_team_members" "infrastructure_members" {
  team_id   = tfe_team.infrastructure.id
  usernames = ["alice", "bob", "charlie"]
}

resource "tfe_team_members" "developers_members" {
  team_id   = tfe_team.developers.id
  usernames = ["david", "eve", "frank"]
}
```

---

## ขั้นตอนที่ 746: Sentinel Policy Tiers

### Advisory Policy

```python
# policies/require-tags.sentinel (Advisory)
# แค่ warn ไม่ block

import "tfplan/v2" as tfplan

# ดึง EC2 instances ทั้งหมด
all_ec2_instances = filter tfplan.resource_changes as _, rc {
  rc.type is "aws_instance" and
  (rc.change.actions contains "create" or rc.change.actions contains "update")
}

# ตรวจสอบ required tags
required_tags = ["Environment", "Project", "Owner"]

missing_tags = filter all_ec2_instances as _, instance {
  any required_tags as tag {
    not (instance.change.after.tags[tag] is not null)
  }
}

# Advisory: แค่ print warning ไม่ fail
main = rule {
  if length(missing_tags) > 0 {
    print("WARNING: Some instances are missing required tags")
    true  # return true = advisory ไม่ fail
  } else {
    true
  }
}
```

### Soft-Mandatory Policy

```python
# policies/require-encryption.sentinel (Soft-Mandatory)
# Block แต่ organization owner override ได้

import "tfplan/v2" as tfplan

all_ebs_volumes = filter tfplan.resource_changes as _, rc {
  rc.type is "aws_ebs_volume" and
  (rc.change.actions contains "create")
}

unencrypted_volumes = filter all_ebs_volumes as _, vol {
  vol.change.after.encrypted is false or
  vol.change.after.encrypted is null
}

main = rule {
  length(unencrypted_volumes) == 0
}
```

### Hard-Mandatory Policy

```python
# policies/no-public-s3.sentinel (Hard-Mandatory)
# ไม่มีทางข้ามได้ แม้แต่ admin ก็ override ไม่ได้

import "tfplan/v2" as tfplan
import "tfstate/v2" as tfstate

# ดึง S3 bucket ACLs ทั้งหมด
public_acls = ["public-read", "public-read-write", "authenticated-read"]

all_s3_acl_changes = filter tfplan.resource_changes as _, rc {
  rc.type is "aws_s3_bucket_acl"
}

public_s3_buckets = filter all_s3_acl_changes as _, acl {
  any public_acls as public_acl {
    acl.change.after.acl is public_acl
  }
}

main = rule {
  length(public_s3_buckets) == 0
}
```

---

## ขั้นตอนที่ 747: No-Code Provisioning

### สร้าง No-Code Module

```
TFE Feature: No-Code Provisioning
- ให้ users ที่ไม่รู้ Terraform สร้าง infrastructure ได้
- ผ่าน UI form แทน code
- ใช้ private registry modules
```

```hcl
# modules/s3-website/variables.tf
# Variables เหล่านี้จะกลายเป็น UI form fields

variable "bucket_name" {
  type        = string
  description = "Name for the S3 bucket (must be globally unique)"

  validation {
    condition     = length(var.bucket_name) >= 3 && length(var.bucket_name) <= 63
    error_message = "Bucket name must be 3-63 characters"
  }
}

variable "website_index_document" {
  type        = string
  description = "Index document filename"
  default     = "index.html"
}

variable "enable_versioning" {
  type        = bool
  description = "Enable S3 versioning"
  default     = false
}

variable "environment" {
  type        = string
  description = "Deployment environment"

  validation {
    condition     = contains(["dev", "staging", "prod"], var.environment)
    error_message = "Environment must be dev, staging, or prod"
  }
}
```

---

## ขั้นตอนที่ 748: Continuous Validation

```
TFE/TFC Feature: Continuous Validation
- ตรวจสอบว่า state ยังตรงกับ configuration อยู่เสมอ
- รัน health checks ตามกำหนดเวลา
- แจ้งเตือนเมื่อพบ drift
```

```hcl
# เพิ่ม check blocks ใน configuration
resource "aws_s3_bucket" "main" {
  bucket = "my-important-bucket"
}

# Check block - ตรวจสอบ health
check "bucket_exists" {
  data "aws_s3_bucket" "check" {
    bucket = aws_s3_bucket.main.bucket
  }

  assert {
    condition     = data.aws_s3_bucket.check.id == aws_s3_bucket.main.bucket
    error_message = "S3 bucket does not exist or was deleted outside Terraform"
  }
}

check "bucket_versioning_enabled" {
  data "aws_s3_bucket_versioning" "check" {
    bucket = aws_s3_bucket.main.bucket
  }

  assert {
    condition     = data.aws_s3_bucket_versioning.check.versioning_configuration[0].status == "Enabled"
    error_message = "Versioning was disabled outside Terraform"
  }
}
```

```
TFC/TFE Workspace Settings:
- Health → Continuous Validation: Enabled
- Assessment frequency: Every 24 hours (หรือ custom)
- Notify on: Any issue
- Notification channel: Slack
```

---

## ขั้นตอนที่ 749: TFE API

### Common API Operations

```bash
# Base URL
TFE_URL="https://tfe.mycompany.com"
TOKEN="your-api-token"

# Headers
AUTH_HEADER="Authorization: Bearer $TOKEN"
CONTENT_TYPE="Content-Type: application/vnd.api+json"

# List workspaces
curl -s \
  -H "$AUTH_HEADER" \
  "$TFE_URL/api/v2/organizations/mycompany/workspaces" \
  | jq '.data[].attributes.name'

# Trigger a run
curl -s \
  -X POST \
  -H "$AUTH_HEADER" \
  -H "$CONTENT_TYPE" \
  -d '{
    "data": {
      "type": "runs",
      "attributes": {
        "is-destroy": false,
        "message": "Triggered via API"
      },
      "relationships": {
        "workspace": {
          "data": {
            "type": "workspaces",
            "id": "ws-xxxxxxxxxxxxxxxx"
          }
        }
      }
    }
  }' \
  "$TFE_URL/api/v2/runs"

# Approve a run
RUN_ID="run-xxxxxxxxxxxxxxxx"
curl -s \
  -X POST \
  -H "$AUTH_HEADER" \
  -H "$CONTENT_TYPE" \
  -d '{"comment": "Approved via API"}' \
  "$TFE_URL/api/v2/runs/$RUN_ID/actions/apply"

# Get workspace outputs
curl -s \
  -H "$AUTH_HEADER" \
  "$TFE_URL/api/v2/workspaces/$WORKSPACE_ID/current-state-version-outputs" \
  | jq '.data[] | {name: .attributes.name, value: .attributes.value}'
```

### TFE API สำหรับ Module Registry

```bash
# Upload module to private registry
# Step 1: Create module
curl -s \
  -X POST \
  -H "$AUTH_HEADER" \
  -H "$CONTENT_TYPE" \
  -d '{
    "data": {
      "type": "registry-modules",
      "attributes": {
        "name": "vpc",
        "provider": "aws"
      }
    }
  }' \
  "$TFE_URL/api/v2/organizations/mycompany/registry-modules"

# Step 2: Create version
curl -s \
  -X POST \
  -H "$AUTH_HEADER" \
  -H "$CONTENT_TYPE" \
  -d '{
    "data": {
      "type": "registry-module-versions",
      "attributes": {
        "version": "1.0.0"
      }
    }
  }' \
  "$TFE_URL/api/v2/registry-modules/private/mycompany/vpc/aws/versions"

# Step 3: Upload module source (ต้องได้รับ upload URL จาก Step 2)
UPLOAD_URL="..." # จาก Step 2 response
curl -T module.tar.gz "$UPLOAD_URL"
```

---

## ขั้นตอนที่ 750: TFE Architecture Overview

### High-Availability Architecture

```
                          ┌─────────────────────────────┐
                          │         Load Balancer        │
                          │     (ALB/NLB - HTTPS 443)    │
                          └──────────┬──────────────────┘
                                     │
                    ┌────────────────┼───────────────────┐
                    │                │                   │
             ┌──────┴──────┐ ┌──────┴──────┐ ┌─────────┴─────┐
             │   TFE Node 1│ │  TFE Node 2 │ │  TFE Node 3   │
             │  (Primary)  │ │  (Standby)  │ │  (Standby)    │
             └──────┬──────┘ └──────┬──────┘ └─────────┬─────┘
                    │               │                   │
        ┌───────────┼───────────────┼───────────────────┼───────┐
        │           │               │                   │       │
   ┌────┴────┐  ┌───┴────┐  ┌───────┴──────┐  ┌────────┴────┐  │
   │PostgreSQL│  │  Redis  │  │  S3/MinIO    │  │  Vault      │  │
   │(Primary+│  │(Cluster)│  │ (Object Store│  │  (Secrets)  │  │
   │Replica) │  │         │  │              │  │             │  │
   └─────────┘  └─────────┘  └──────────────┘  └─────────────┘  │
        │                                                         │
   ┌────┴─────────────────────────────────────────────────────┐   │
   │                    Monitoring Stack                       │   │
   │            (Prometheus, Grafana, AlertManager)            │   │
   └───────────────────────────────────────────────────────────┘   │
```

### Terraform สำหรับ Deploy TFE บน AWS

```hcl
# tfe-infrastructure/main.tf

# VPC สำหรับ TFE
module "vpc" {
  source = "./modules/vpc"

  name       = "tfe-vpc"
  cidr_block = "10.0.0.0/16"

  private_subnets = ["10.0.1.0/24", "10.0.2.0/24", "10.0.3.0/24"]
  public_subnets  = ["10.0.101.0/24", "10.0.102.0/24"]
}

# RDS PostgreSQL
resource "aws_db_instance" "tfe" {
  identifier     = "tfe-database"
  engine         = "postgres"
  engine_version = "15"
  instance_class = "db.m5.xlarge"

  allocated_storage     = 100
  max_allocated_storage = 500
  storage_type          = "gp3"
  storage_encrypted     = true

  db_name  = "tfe"
  username = "tfe"
  password = random_password.db_password.result

  multi_az               = true
  db_subnet_group_name   = aws_db_subnet_group.tfe.name
  vpc_security_group_ids = [aws_security_group.rds.id]

  backup_retention_period   = 7
  backup_window             = "03:00-04:00"
  maintenance_window        = "sun:04:00-sun:05:00"
  skip_final_snapshot       = false
  final_snapshot_identifier = "tfe-final-snapshot"

  tags = {
    Name = "tfe-database"
  }
}

# S3 bucket สำหรับ TFE storage
resource "aws_s3_bucket" "tfe" {
  bucket = "mycompany-tfe-storage-${random_string.suffix.result}"

  tags = {
    Name = "tfe-storage"
  }
}

resource "aws_s3_bucket_versioning" "tfe" {
  bucket = aws_s3_bucket.tfe.id

  versioning_configuration {
    status = "Enabled"
  }
}

resource "aws_s3_bucket_server_side_encryption_configuration" "tfe" {
  bucket = aws_s3_bucket.tfe.id

  rule {
    apply_server_side_encryption_by_default {
      sse_algorithm = "aws:kms"
    }
  }
}

# ElastiCache Redis
resource "aws_elasticache_replication_group" "tfe" {
  replication_group_id       = "tfe-redis"
  description                = "TFE Redis cache"
  node_type                  = "cache.m5.large"
  port                       = 6379
  parameter_group_name       = "default.redis7"
  engine_version             = "7.0"
  num_cache_clusters         = 2
  automatic_failover_enabled = true

  subnet_group_name  = aws_elasticache_subnet_group.tfe.name
  security_group_ids = [aws_security_group.redis.id]

  at_rest_encryption_enabled = true
  transit_encryption_enabled = true
  auth_token                 = random_password.redis_password.result

  tags = {
    Name = "tfe-redis"
  }
}

# Application Load Balancer
resource "aws_lb" "tfe" {
  name               = "tfe-alb"
  internal           = false
  load_balancer_type = "application"
  security_groups    = [aws_security_group.alb.id]
  subnets            = module.vpc.public_subnet_ids

  tags = {
    Name = "tfe-alb"
  }
}

# EC2 Auto Scaling Group สำหรับ TFE nodes
resource "aws_autoscaling_group" "tfe" {
  name                = "tfe-asg"
  vpc_zone_identifier = module.vpc.private_subnet_ids
  target_group_arns   = [aws_lb_target_group.tfe.arn]
  health_check_type   = "ELB"

  min_size         = 2
  max_size         = 4
  desired_capacity = 2

  launch_template {
    id      = aws_launch_template.tfe.id
    version = "$Latest"
  }

  tag {
    key                 = "Name"
    value               = "tfe-node"
    propagate_at_launch = true
  }
}
```

---

## สรุป (Summary)

Terraform Enterprise ให้ความสามารถระดับ Enterprise:

1. **Self-hosted** - ควบคุม data ทั้งหมด
2. **Air-gapped** - สำหรับ high-security environments
3. **SAML SSO** - integrate กับ enterprise identity
4. **Advanced RBAC** - fine-grained access control
5. **Sentinel** - policy as code ที่ hard-enforce ได้
6. **Audit Logging** - track ทุก action
7. **HA Deployment** - high availability deployment
8. **Custom Agents** - deploy ใน private network

---

*จบ Part 075 - ในส่วนถัดไปจะเรียนรู้เรื่อง Sentinel Policy as Code*
