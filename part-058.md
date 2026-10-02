# Part 058: GCP Provider Setup & Authentication
## ขั้นตอนที่ 571-580: การตั้งค่า Google Cloud Provider

---

## ขั้นตอนที่ 571: GCP Provider Versions

Google Cloud Provider (google) เป็น Terraform provider สำหรับจัดการ Google Cloud Platform resources

```hcl
# terraform.tf - GCP Provider configuration

terraform {
  required_version = ">= 1.5.0"
  
  required_providers {
    google = {
      source  = "hashicorp/google"
      version = "~> 5.0"  # Latest stable 5.x
    }
    
    # Google Beta provider สำหรับ preview/beta features
    google-beta = {
      source  = "hashicorp/google-beta"
      version = "~> 5.0"
    }
    
    # Google Cloud Storage backend
    # (ใช้ backend block แยก)
  }
  
  backend "gcs" {
    bucket  = "tf-state-myproject-prod"
    prefix  = "terraform/state"
    
    # Impersonate service account (optional)
    # impersonate_service_account = "terraform@myproject.iam.gserviceaccount.com"
  }
}

# Provider พื้นฐาน
provider "google" {
  project = var.project_id
  region  = var.region
  zone    = var.zone
}

# Beta provider (สำหรับ beta features)
provider "google-beta" {
  project = var.project_id
  region  = var.region
  zone    = var.zone
}
```

### Variables สำหรับ GCP

```hcl
# variables.tf

variable "project_id" {
  description = "GCP Project ID"
  type        = string
}

variable "region" {
  description = "GCP region"
  type        = string
  default     = "asia-southeast1"  # Singapore
}

variable "zone" {
  description = "GCP zone"
  type        = string
  default     = "asia-southeast1-a"
}

# GCP Regions ที่ใช้บ่อย
locals {
  gcp_regions = {
    # Asia Pacific
    singapore        = "asia-southeast1"
    jakarta          = "asia-southeast2"
    tokyo            = "asia-northeast1"
    osaka            = "asia-northeast2"
    seoul            = "asia-northeast3"
    mumbai           = "asia-south1"
    delhi            = "asia-south2"
    taiwan           = "asia-east1"
    hongkong         = "asia-east2"
    sydney           = "australia-southeast1"
    
    # Americas
    us_central       = "us-central1"        # Iowa
    us_east          = "us-east1"           # South Carolina
    us_east4         = "us-east4"           # Northern Virginia
    us_west          = "us-west1"           # Oregon
    us_west2         = "us-west2"           # Los Angeles
    
    # Europe
    eu_west          = "europe-west1"       # Belgium
    eu_west2         = "europe-west2"       # London
    eu_west3         = "europe-west3"       # Frankfurt
    eu_north         = "europe-north1"      # Finland
    
    # Middle East
    me_west          = "me-west1"           # Tel Aviv
    me_central       = "me-central1"        # Doha
  }
}
```

---

## ขั้นตอนที่ 572: Application Default Credentials (ADC)

วิธีที่ง่ายที่สุดสำหรับ Local Development

```bash
# ติดตั้ง gcloud CLI
# Linux
curl https://sdk.cloud.google.com | bash
exec -l $SHELL

# macOS
brew install google-cloud-sdk

# Windows
winget install Google.CloudSDK

# =====================================================
# Application Default Credentials (ADC) Setup
# =====================================================

# Login ด้วย user account
gcloud auth application-default login

# หรือ login พร้อมกำหนด scopes
gcloud auth application-default login \
  --scopes=https://www.googleapis.com/auth/cloud-platform

# ดู credentials ที่ active
gcloud auth application-default print-access-token

# ดูรายการ accounts
gcloud auth list

# กำหนด project default
gcloud config set project my-project-id

# ดู current config
gcloud config list

# =====================================================
# Config profiles
# =====================================================

# สร้าง profile ใหม่
gcloud config configurations create production
gcloud config set project prod-project-id
gcloud config set compute/region asia-southeast1

# สลับ profile
gcloud config configurations activate production

# รายการ profiles
gcloud config configurations list
```

```hcl
# auth-adc.tf
# เมื่อใช้ ADC ไม่ต้องกำหนด credentials ใน provider
provider "google" {
  project = "my-project-id"
  region  = "asia-southeast1"
  zone    = "asia-southeast1-a"
}

# ดึงข้อมูล project ปัจจุบัน
data "google_project" "current" {}

output "project_info" {
  value = {
    project_id   = data.google_project.current.project_id
    project_name = data.google_project.current.name
    project_number = data.google_project.current.number
  }
}
```

---

## ขั้นตอนที่ 573: Service Account Key File

```bash
# =====================================================
# สร้าง Service Account
# =====================================================

PROJECT_ID="my-project-id"
SA_NAME="terraform-sa"
SA_EMAIL="${SA_NAME}@${PROJECT_ID}.iam.gserviceaccount.com"

# สร้าง Service Account
gcloud iam service-accounts create "$SA_NAME" \
  --display-name "Terraform Service Account" \
  --description "Used for Terraform infrastructure management" \
  --project "$PROJECT_ID"

# ให้ roles กับ SA
gcloud projects add-iam-policy-binding "$PROJECT_ID" \
  --member "serviceAccount:${SA_EMAIL}" \
  --role "roles/editor"

# หรือใช้ custom role
gcloud projects add-iam-policy-binding "$PROJECT_ID" \
  --member "serviceAccount:${SA_EMAIL}" \
  --role "roles/compute.admin"

gcloud projects add-iam-policy-binding "$PROJECT_ID" \
  --member "serviceAccount:${SA_EMAIL}" \
  --role "roles/container.admin"

gcloud projects add-iam-policy-binding "$PROJECT_ID" \
  --member "serviceAccount:${SA_EMAIL}" \
  --role "roles/storage.admin"

gcloud projects add-iam-policy-binding "$PROJECT_ID" \
  --member "serviceAccount:${SA_EMAIL}" \
  --role "roles/iam.serviceAccountUser"

# สร้าง Key File
gcloud iam service-accounts keys create \
  "./terraform-sa-key.json" \
  --iam-account "${SA_EMAIL}" \
  --project "$PROJECT_ID"

echo "Service Account: $SA_EMAIL"
echo "Key file: terraform-sa-key.json"

# !! อย่า commit key file ไปใน git !!
echo "terraform-sa-key.json" >> .gitignore
echo "*.json" >> .gitignore  # หรือเฉพาะ key files
```

```hcl
# auth-service-account-key.tf

variable "credentials_file" {
  description = "Path to GCP service account key file"
  type        = string
  default     = "terraform-sa-key.json"
  sensitive   = true
}

provider "google" {
  credentials = file(var.credentials_file)
  project     = var.project_id
  region      = var.region
}

# หรือใช้ JSON string โดยตรง
provider "google" {
  credentials = var.google_credentials_json  # JSON string
  project     = var.project_id
  region      = var.region
}

# Environment Variable method
# export GOOGLE_APPLICATION_CREDENTIALS="/path/to/key.json"
# แล้วไม่ต้องกำหนด credentials ใน provider
provider "google" {
  project = var.project_id
  region  = var.region
  # อ่าน GOOGLE_APPLICATION_CREDENTIALS อัตโนมัติ
}
```

---

## ขั้นตอนที่ 574: Service Account Impersonation

Service Account Impersonation ปลอดภัยกว่าการใช้ Key File

```bash
# =====================================================
# กำหนด Impersonation Permissions
# =====================================================

IMPERSONATING_SA="developer@myproject.iam.gserviceaccount.com"
TARGET_SA="terraform@myproject.iam.gserviceaccount.com"

# ให้ impersonating SA สามารถใช้ target SA ได้
gcloud iam service-accounts add-iam-policy-binding \
  "$TARGET_SA" \
  --member "serviceAccount:${IMPERSONATING_SA}" \
  --role "roles/iam.serviceAccountTokenCreator"

# หรือให้ user account impersonate
gcloud iam service-accounts add-iam-policy-binding \
  "$TARGET_SA" \
  --member "user:developer@company.com" \
  --role "roles/iam.serviceAccountTokenCreator"
```

```hcl
# auth-impersonation.tf

provider "google" {
  project = var.project_id
  region  = var.region
  
  # Impersonate service account
  impersonate_service_account = "terraform@myproject.iam.gserviceaccount.com"
}

# กำหนดใน environment variable
# export GOOGLE_IMPERSONATE_SERVICE_ACCOUNT="terraform@myproject.iam.gserviceaccount.com"

# หรือสำหรับ backend
terraform {
  backend "gcs" {
    bucket                      = "tf-state-bucket"
    prefix                      = "prod/state"
    impersonate_service_account = "terraform@myproject.iam.gserviceaccount.com"
  }
}
```

---

## ขั้นตอนที่ 575: Workload Identity Federation

```bash
# =====================================================
# Setup Workload Identity Federation สำหรับ GitHub Actions
# =====================================================

PROJECT_ID="my-project-id"
PROJECT_NUMBER=$(gcloud projects describe $PROJECT_ID --format='value(projectNumber)')
POOL_ID="github-actions-pool"
PROVIDER_ID="github-actions-provider"
SA_EMAIL="terraform@${PROJECT_ID}.iam.gserviceaccount.com"
GITHUB_ORG="my-org"
GITHUB_REPO="my-repo"

# สร้าง Workload Identity Pool
gcloud iam workload-identity-pools create "$POOL_ID" \
  --location="global" \
  --display-name="GitHub Actions Pool" \
  --description="Pool for GitHub Actions" \
  --project="$PROJECT_ID"

# สร้าง OIDC Provider สำหรับ GitHub Actions
gcloud iam workload-identity-pools providers create-oidc "$PROVIDER_ID" \
  --location="global" \
  --workload-identity-pool="$POOL_ID" \
  --display-name="GitHub Actions OIDC" \
  --attribute-mapping="google.subject=assertion.sub,attribute.actor=assertion.actor,attribute.repository=assertion.repository,attribute.repository_owner=assertion.repository_owner" \
  --attribute-condition="assertion.repository_owner == '${GITHUB_ORG}'" \
  --issuer-uri="https://token.actions.githubusercontent.com" \
  --project="$PROJECT_ID"

# ให้ Service Account ถูก impersonate ผ่าน Workload Identity
gcloud iam service-accounts add-iam-policy-binding "$SA_EMAIL" \
  --role="roles/iam.workloadIdentityUser" \
  --member="principalSet://iam.googleapis.com/projects/${PROJECT_NUMBER}/locations/global/workloadIdentityPools/${POOL_ID}/attribute.repository/${GITHUB_ORG}/${GITHUB_REPO}" \
  --project="$PROJECT_ID"

# ดึง Provider Resource Name
PROVIDER_NAME=$(gcloud iam workload-identity-pools providers describe "$PROVIDER_ID" \
  --location="global" \
  --workload-identity-pool="$POOL_ID" \
  --format="value(name)" \
  --project="$PROJECT_ID")

echo "Workload Identity Provider: $PROVIDER_NAME"
echo "Service Account: $SA_EMAIL"
```

```yaml
# .github/workflows/terraform.yml
name: Terraform on GCP

on:
  push:
    branches: [main]
  pull_request:

permissions:
  id-token: write
  contents: read

jobs:
  terraform:
    runs-on: ubuntu-latest
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Authenticate to GCP
        uses: google-github-actions/auth@v2
        with:
          workload_identity_provider: "projects/123456789/locations/global/workloadIdentityPools/github-actions-pool/providers/github-actions-provider"
          service_account: "terraform@my-project.iam.gserviceaccount.com"
      
      - name: Set up Cloud SDK
        uses: google-github-actions/setup-gcloud@v2
      
      - name: Setup Terraform
        uses: hashicorp/setup-terraform@v3
      
      - name: Terraform Init
        run: terraform init
        env:
          GOOGLE_PROJECT: ${{ vars.GCP_PROJECT_ID }}
      
      - name: Terraform Plan
        run: terraform plan
        env:
          GOOGLE_PROJECT: ${{ vars.GCP_PROJECT_ID }}
      
      - name: Terraform Apply
        if: github.ref == 'refs/heads/main' && github.event_name == 'push'
        run: terraform apply -auto-approve
```

```hcl
# auth-workload-identity.tf
# เมื่อใช้ Workload Identity ไม่ต้องกำหนด credentials
provider "google" {
  project = var.project_id
  region  = var.region
  # Credentials ถูก set โดย google-github-actions/auth action อัตโนมัติ
}
```

---

## ขั้นตอนที่ 576: Metadata Server (VM, Cloud Run)

```hcl
# auth-metadata-server.tf
# เมื่อ run บน GCE/Cloud Run/Cloud Functions ไม่ต้องกำหนด credentials
# Metadata server จัดการ authentication อัตโนมัติ

provider "google" {
  project = var.project_id
  region  = var.region
  # ไม่ต้องกำหนด credentials - อ่านจาก metadata server อัตโนมัติ
}

# ตัวอย่าง: Cloud Build pipeline
# cloudbuild.yaml
```

```yaml
# cloudbuild.yaml
steps:
  - name: 'hashicorp/terraform:latest'
    entrypoint: 'sh'
    args:
      - '-c'
      - |
        terraform init
        terraform plan
        terraform apply -auto-approve
    env:
      - 'TF_VAR_project_id=$PROJECT_ID'
```

---

## ขั้นตอนที่ 577: Environment Variables

```bash
# =====================================================
# GCP Environment Variables Reference
# =====================================================

# === Project ===
export GOOGLE_PROJECT="my-project-id"
export GOOGLE_CLOUD_PROJECT="my-project-id"  # Alternative
export GCLOUD_PROJECT="my-project-id"        # Alternative

# === Credentials ===
export GOOGLE_APPLICATION_CREDENTIALS="/path/to/key.json"
export GOOGLE_CREDENTIALS="/path/to/key.json"         # Alternative
export GOOGLE_OAUTH_ACCESS_TOKEN="ya29...."            # Direct token

# === Region/Zone ===
export GOOGLE_REGION="asia-southeast1"
export GOOGLE_ZONE="asia-southeast1-a"
export GOOGLE_COMPUTE_REGION="asia-southeast1"  # For compute
export GOOGLE_COMPUTE_ZONE="asia-southeast1-a"  # For compute

# === Impersonation ===
export GOOGLE_IMPERSONATE_SERVICE_ACCOUNT="terraform@project.iam.gserviceaccount.com"

# === Terraform-specific ===
export TF_VAR_project_id="my-project-id"
export TF_VAR_region="asia-southeast1"
export TF_VAR_zone="asia-southeast1-a"

# =====================================================
# .env file สำหรับ local development
# =====================================================
cat > .env.gcp << 'EOF'
# GCP Configuration - LOCAL DEVELOPMENT
# DO NOT COMMIT
GOOGLE_APPLICATION_CREDENTIALS=./terraform-sa-key.json
GOOGLE_PROJECT=my-dev-project
GOOGLE_REGION=asia-southeast1
GOOGLE_ZONE=asia-southeast1-a
TF_VAR_project_id=my-dev-project
TF_VAR_region=asia-southeast1
EOF

# โหลด env
set -a && source .env.gcp && set +a
```

---

## ขั้นตอนที่ 578: GCP Project Structure

```hcl
# project-structure.tf

# ============================================================
# Google Project Data Sources
# ============================================================

# ดึงข้อมูล project ปัจจุบัน
data "google_project" "current" {}

output "project_details" {
  value = {
    id     = data.google_project.current.project_id
    name   = data.google_project.current.name
    number = data.google_project.current.number
  }
}

# ดึงข้อมูล project อื่น
data "google_project" "shared_vpc" {
  project_id = "shared-vpc-project"
}

# ดึงข้อมูล Organization
data "google_organization" "main" {
  domain = "company.com"
}

# ============================================================
# Enabling APIs
# ============================================================

locals {
  required_apis = [
    "compute.googleapis.com",
    "container.googleapis.com",
    "iam.googleapis.com",
    "cloudresourcemanager.googleapis.com",
    "cloudbilling.googleapis.com",
    "sqladmin.googleapis.com",
    "servicenetworking.googleapis.com",
    "redis.googleapis.com",
    "storage.googleapis.com",
    "cloudkms.googleapis.com",
    "secretmanager.googleapis.com",
    "monitoring.googleapis.com",
    "logging.googleapis.com",
    "cloudtrace.googleapis.com",
    "cloudbuild.googleapis.com",
    "artifactregistry.googleapis.com",
    "dns.googleapis.com",
    "pubsub.googleapis.com",
    "run.googleapis.com",
    "cloudfunctions.googleapis.com",
    "vpcaccess.googleapis.com"
  ]
}

resource "google_project_service" "apis" {
  for_each = toset(local.required_apis)
  
  project = var.project_id
  service = each.value
  
  disable_on_destroy         = false  # ไม่ disable API เมื่อ destroy
  disable_dependent_services = false
}

# Enable APIs แบบ module
module "project_services" {
  source  = "terraform-google-modules/project-factory/google//modules/project_services"
  version = "~> 14.0"
  
  project_id = var.project_id
  
  activate_apis = local.required_apis
  
  disable_services_on_destroy = false
}
```

---

## ขั้นตอนที่ 579: GCP Naming Conventions และ Labels

```hcl
# gcp-conventions.tf

# ============================================================
# GCP Naming Conventions
# ============================================================
# GCP resource names: lowercase, hyphens allowed, no spaces
# Labels: lowercase, underscores allowed for values

locals {
  env        = "prod"
  app        = "myapp"
  team       = "platform"
  region     = "asia-southeast1"
  region_code = "sea"
  
  # Resource naming convention
  names = {
    # Networking
    vpc               = "vpc-${local.app}-${local.env}"
    subnet_private    = "subnet-private-${local.app}-${local.env}-${local.region_code}"
    subnet_public     = "subnet-public-${local.app}-${local.env}-${local.region_code}"
    router            = "router-${local.app}-${local.env}-${local.region_code}"
    nat               = "nat-${local.app}-${local.env}-${local.region_code}"
    firewall_allow    = "fw-allow-${local.app}-${local.env}"
    firewall_deny     = "fw-deny-${local.app}-${local.env}"
    
    # Compute
    instance          = "${local.app}-${local.env}-vm-001"
    instance_template = "${local.app}-${local.env}-it"
    mig               = "${local.app}-${local.env}-mig"
    
    # GKE
    cluster           = "gke-${local.app}-${local.env}-${local.region_code}"
    nodepool_default  = "default"
    nodepool_app      = "app-pool"
    
    # Storage
    bucket            = "${local.app}-${local.env}-${local.region_code}-001"  # globally unique
    
    # Database
    sql_instance      = "${local.app}-${local.env}-sql"
    
    # Service Accounts
    sa_terraform      = "terraform-sa"
    sa_gke            = "gke-sa"
    sa_app            = "${local.app}-sa"
    
    # Secret Manager
    secret_db_pass    = "${local.app}-${local.env}-db-password"
    
    # Artifact Registry
    registry          = "${local.app}-${local.env}-registry"
  }
  
  # GCP Labels (equivalent to AWS/Azure tags)
  common_labels = {
    environment = local.env
    application = local.app
    team        = local.team
    managed_by  = "terraform"
    terraform_repo = "github_com_company_infra"  # underscores only, no dots/slashes
    cost_center    = "cc_001"
  }
}

# ============================================================
# GCP IAM
# ============================================================

# Service Account สำหรับ applications
resource "google_service_account" "app" {
  account_id   = local.names.sa_app
  display_name = "Application Service Account"
  description  = "Service account for ${local.app} application"
  project      = var.project_id
}

# Roles
resource "google_project_iam_member" "app_storage_reader" {
  project = var.project_id
  role    = "roles/storage.objectViewer"
  member  = "serviceAccount:${google_service_account.app.email}"
}

resource "google_project_iam_member" "app_secret_accessor" {
  project = var.project_id
  role    = "roles/secretmanager.secretAccessor"
  member  = "serviceAccount:${google_service_account.app.email}"
}

# Service Account สำหรับ Terraform
resource "google_service_account" "terraform" {
  account_id   = local.names.sa_terraform
  display_name = "Terraform Service Account"
  description  = "Service account for Terraform infrastructure management"
  project      = var.project_id
}

resource "google_project_iam_member" "terraform_editor" {
  project = var.project_id
  role    = "roles/editor"
  member  = "serviceAccount:${google_service_account.terraform.email}"
}

resource "google_project_iam_member" "terraform_security_admin" {
  project = var.project_id
  role    = "roles/iam.securityAdmin"
  member  = "serviceAccount:${google_service_account.terraform.email}"
}
```

---

## ขั้นตอนที่ 580: Complete Provider Setup

```hcl
# complete-gcp-setup.tf

terraform {
  required_version = ">= 1.5.0"
  
  required_providers {
    google = {
      source  = "hashicorp/google"
      version = "~> 5.0"
    }
    google-beta = {
      source  = "hashicorp/google-beta"
      version = "~> 5.0"
    }
  }
  
  backend "gcs" {
    bucket = "tf-state-myproject-prod"
    prefix = "environments/prod"
  }
}

# ============================================================
# Multiple Projects
# ============================================================

provider "google" {
  alias   = "production"
  project = var.prod_project_id
  region  = var.prod_region
}

provider "google" {
  alias   = "staging"
  project = var.staging_project_id
  region  = var.staging_region
}

provider "google" {
  alias   = "shared"
  project = var.shared_project_id
  region  = var.primary_region
}

# Resources with provider aliases
resource "google_compute_network" "prod_vpc" {
  provider = google.production
  name     = "vpc-myapp-prod"
  project  = var.prod_project_id
}

resource "google_compute_network" "staging_vpc" {
  provider = google.staging
  name     = "vpc-myapp-staging"
  project  = var.staging_project_id
}

# ============================================================
# GCP Organization IAM
# ============================================================

data "google_organization" "main" {
  domain = "company.com"
}

# Organization-level roles
resource "google_organization_iam_member" "terraform_org_viewer" {
  org_id = data.google_organization.main.org_id
  role   = "roles/resourcemanager.organizationViewer"
  member = "serviceAccount:${google_service_account.terraform.email}"
}

# ============================================================
# Terraform State Bucket
# ============================================================

resource "google_storage_bucket" "terraform_state" {
  name          = "tf-state-${var.project_id}"
  location      = "ASIA"  # Multi-region
  force_destroy = false
  
  # Versioning สำคัญมากสำหรับ state file
  versioning {
    enabled = true
  }
  
  # Lifecycle - keep old versions
  lifecycle_rule {
    condition {
      num_newer_versions = 10  # Keep 10 newest versions
    }
    action {
      type = "Delete"
    }
  }
  
  # Prevent public access
  public_access_prevention = "enforced"
  
  uniform_bucket_level_access = true
  
  labels = local.common_labels
}

# Lock bucket against deletion
resource "google_storage_bucket_iam_member" "terraform_state_writer" {
  bucket = google_storage_bucket.terraform_state.name
  role   = "roles/storage.objectAdmin"
  member = "serviceAccount:${google_service_account.terraform.email}"
}

# ============================================================
# Complete Outputs
# ============================================================

output "provider_info" {
  value = {
    project        = var.project_id
    region         = var.region
    zone           = var.zone
    project_number = data.google_project.current.number
  }
}

output "service_accounts" {
  value = {
    terraform_email = google_service_account.terraform.email
    app_email       = google_service_account.app.email
  }
}
```

---

## GCP Authentication Methods Comparison

| Method | Use Case | Security | Setup |
|--------|----------|----------|-------|
| gcloud ADC | Local Development | High | Easy |
| Service Account Key | Legacy/Simple CI | Medium | Easy |
| SA Impersonation | Production CI | High | Medium |
| Workload Identity | GitHub/GitLab CI | Very High | Complex |
| Metadata Server | GCE/Cloud Run | Very High | Auto |
| Cloud Build | GCP Native CI | Very High | Easy |

### Best Practices Summary

```hcl
# security-best-practices.tf

# 1. ไม่ hardcode credentials
# BAD:
# provider "google" {
#   credentials = "{\"type\": \"service_account\", ...}"  # อันตราย!
# }

# GOOD: ใช้ environment variables
provider "google" {
  project = var.project_id
  region  = var.region
  # GOOGLE_APPLICATION_CREDENTIALS env var
}

# 2. ใช้ Workload Identity สำหรับ CI/CD
# 3. ใช้ Service Account Impersonation
# 4. Rotate service account keys สม่ำเสมอ
# 5. ใช้ Least Privilege principles
# 6. Enable Audit Logs
# 7. ใช้ Secret Manager สำหรับ secrets
# 8. Enable Organization Policies
```

### GCP Organization Structure

```
Google Organization (company.com)
├── Folders
│   ├── Production
│   │   ├── Project: app-prod (prod workloads)
│   │   ├── Project: shared-vpc-prod (networking)
│   │   └── Project: monitoring-prod (observability)
│   ├── Non-Production
│   │   ├── Project: app-dev
│   │   └── Project: app-staging
│   └── Shared
│       ├── Project: terraform-admin (Terraform state)
│       └── Project: cicd (CI/CD pipelines)
└── Billing Account
    └── (linked to all projects)
```

---

*จบ Part 058: GCP Provider Setup & Authentication*  
*ต่อไป Part 059: GCP Compute Engine & GKE*
