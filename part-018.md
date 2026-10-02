# Part 018: Terraform Providers (Terraform Providers)
## Steps 171-180: ทำความเข้าใจ Provider ใน Terraform

---

## บทนำ (Introduction)

Providers คือ plugins ที่ทำให้ Terraform สามารถ interact กับ APIs ต่างๆ ได้ ทั้ง cloud providers (AWS, GCP, Azure), SaaS services (Datadog, GitHub), และ local resources (DNS, TLS) Providers เป็นสิ่งที่แยก Terraform Core ออกจาก implementation รายละเอียด

---

## Step 171: What is a Provider?

### นิยามและบทบาทของ Provider

```
Provider = Plugin ที่:
1. แปลง Terraform resource definitions เป็น API calls
2. จัดการ authentication กับ API
3. Map resource attributes กับ API response
4. Handle CRUD operations

ตัวอย่าง:
- aws provider → เรียก AWS APIs
- google provider → เรียก Google Cloud APIs
- azurerm provider → เรียก Azure APIs
- github provider → เรียก GitHub API
- datadog provider → เรียก Datadog API
```

### Provider Architecture

```
┌─────────────────────────────────────┐
│           Terraform Core            │
│                                     │
│  resource "aws_instance" "web" {   │
│    ami = "ami-123"                  │
│  }                                  │
└─────────────────┬───────────────────┘
                  │ gRPC Plugin Protocol
                  ▼
┌─────────────────────────────────────┐
│       AWS Provider Plugin           │
│                                     │
│  - Handles auth (access_key, etc.) │
│  - Knows aws_instance attributes   │
│  - Calls EC2 API                   │
│  - Returns resource state          │
└─────────────────┬───────────────────┘
                  │ HTTPS
                  ▼
┌─────────────────────────────────────┐
│           AWS EC2 API               │
│  ec2.amazonaws.com                  │
└─────────────────────────────────────┘
```

---

## Step 172: Provider Registry

### Terraform Registry

```
registry.terraform.io = ศูนย์กลาง provider distribution

URL: https://registry.terraform.io/providers

Structure:
registry.terraform.io/<NAMESPACE>/<PROVIDER_NAME>

ตัวอย่าง:
registry.terraform.io/hashicorp/aws
registry.terraform.io/hashicorp/google
registry.terraform.io/hashicorp/azurerm
registry.terraform.io/datadog/datadog
```

### Provider Tiers

```
1. Official (Verified by HashiCorp)
   ✅ Maintained by HashiCorp
   ✅ Highest quality standards
   ✅ Full test coverage
   Examples: aws, google, azurerm, kubernetes

2. Partner (Verified by Technology Partners)
   ✅ Maintained by technology company
   ✅ Verified by HashiCorp
   Examples: datadog/datadog, pagerduty/pagerduty, mongodb/mongodbatlas

3. Community
   - Maintained by community
   - Quality varies
   Examples: custom providers

4. Internal/Private
   - Used within organizations
   - Not published to public registry
```

---

## Step 173: required_providers Block

### การประกาศ Providers

```hcl
# versions.tf

terraform {
  required_version = ">= 1.3.0"
  
  required_providers {
    # Official AWS provider
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
    
    # Official Google Cloud provider
    google = {
      source  = "hashicorp/google"
      version = "~> 5.0"
    }
    
    # Partner provider
    datadog = {
      source  = "DataDog/datadog"
      version = "~> 3.0"
    }
    
    # Random provider (utilities)
    random = {
      source  = "hashicorp/random"
      version = "~> 3.5"
    }
    
    # TLS provider (certificates)
    tls = {
      source  = "hashicorp/tls"
      version = "~> 4.0"
    }
    
    # Null provider
    null = {
      source  = "hashicorp/null"
      version = "~> 3.2"
    }
    
    # Local provider
    local = {
      source  = "hashicorp/local"
      version = "~> 2.4"
    }
    
    # External provider
    external = {
      source  = "hashicorp/external"
      version = "~> 2.3"
    }
    
    # HTTP provider
    http = {
      source  = "hashicorp/http"
      version = "~> 3.4"
    }
  }
}
```

---

## Step 174: Provider Version Constraints

### Syntax ของ Version Constraints

```hcl
# Exact version (ไม่แนะนำ - ทำให้ update ยาก)
version = "5.31.0"

# Greater than or equal
version = ">= 5.0.0"

# Less than
version = "< 6.0.0"

# Not equal
version = "!= 5.1.0"

# Pessimistic constraint operator (~>) - แนะนำ!
version = "~> 5.0"    # >= 5.0.0, < 6.0.0  (minor updates allowed)
version = "~> 5.31"   # >= 5.31.0, < 5.32.0 (patch updates only)

# Range
version = ">= 5.0.0, < 6.0.0"
version = ">= 5.0.0, != 5.1.0, < 6.0.0"

# Any version (ไม่แนะนำ!)
version = "*"  # NEVER use this in production
```

### ตัวอย่างการเลือก Version Constraints

```hcl
required_providers {
  # ✅ Production: ใช้ ~> major version
  aws = {
    source  = "hashicorp/aws"
    version = "~> 5.0"
  }
  
  # ✅ ต้องการ stability สูง: lock minor version
  kubernetes = {
    source  = "hashicorp/kubernetes"
    version = "~> 2.24"
  }
  
  # ✅ ต้องการ specific features: >= constraint
  helm = {
    source  = "hashicorp/helm"
    version = ">= 2.12.0"
  }
  
  # ✅ Skip buggy version: != constraint
  azurerm = {
    source  = "hashicorp/azurerm"
    version = ">= 3.75.0, != 3.80.0"  # skip buggy version
  }
}
```

---

## Step 175: Provider Configuration Block

### AWS Provider Configuration

```hcl
# provider.tf

provider "aws" {
  region = "ap-southeast-1"
  
  # Optional: explicit credentials (ไม่แนะนำ hardcode)
  # access_key = var.aws_access_key
  # secret_key = var.aws_secret_key
  
  # Default tags (apply to all resources)
  default_tags {
    tags = {
      ManagedBy   = "terraform"
      Project     = var.project_name
      Environment = var.environment
    }
  }
}
```

### Google Cloud Provider

```hcl
provider "google" {
  project = var.gcp_project_id
  region  = var.gcp_region
  zone    = var.gcp_zone
}

provider "google-beta" {
  project = var.gcp_project_id
  region  = var.gcp_region
}
```

### Azure Provider

```hcl
provider "azurerm" {
  features {
    key_vault {
      purge_soft_delete_on_destroy = false
    }
    virtual_machine {
      delete_os_disk_on_deletion = true
    }
  }
  
  subscription_id = var.azure_subscription_id
  tenant_id       = var.azure_tenant_id
}
```

### GitHub Provider

```hcl
provider "github" {
  owner = var.github_organization
  token = var.github_token
}
```

### Datadog Provider

```hcl
provider "datadog" {
  api_key = var.datadog_api_key
  app_key = var.datadog_app_key
  api_url = "https://api.datadoghq.com/"
}
```

---

## Step 176: Multiple Provider Instances with Alias

### ทำไมต้องใช้ Provider Aliases?

```
Use Cases:
1. Multi-region deployment (AWS us-east-1 + ap-southeast-1)
2. Multi-account AWS (dev account + prod account)
3. Multi-cloud (AWS + GCP ใน configuration เดียว)
4. Different credentials สำหรับ different resources
```

### การใช้ Provider Alias

```hcl
# provider.tf

# Default provider (ไม่ต้องระบุ alias ตอนใช้)
provider "aws" {
  region = "ap-southeast-1"
}

# Provider with alias
provider "aws" {
  alias  = "us_east_1"
  region = "us-east-1"
}

provider "aws" {
  alias  = "eu_west_1"
  region = "eu-west-1"
}

# Multi-account
provider "aws" {
  alias  = "dev"
  region = "ap-southeast-1"
  
  assume_role {
    role_arn = "arn:aws:iam::111111111111:role/TerraformRole"
  }
}

provider "aws" {
  alias  = "prod"
  region = "ap-southeast-1"
  
  assume_role {
    role_arn = "arn:aws:iam::222222222222:role/TerraformRole"
  }
}
```

### การใช้งาน Provider Alias

```hcl
# ใช้ default provider (ไม่ต้องระบุ)
resource "aws_vpc" "main" {
  cidr_block = "10.0.0.0/16"
}

# ใช้ provider alias ที่ระบุ
resource "aws_vpc" "us_east" {
  provider   = aws.us_east_1  # ระบุ alias
  cidr_block = "10.1.0.0/16"
}

resource "aws_vpc" "eu_west" {
  provider   = aws.eu_west_1
  cidr_block = "10.2.0.0/16"
}

# ACM certificate ต้อง create ใน us-east-1 สำหรับ CloudFront
resource "aws_acm_certificate" "main" {
  provider          = aws.us_east_1
  domain_name       = var.domain_name
  validation_method = "DNS"
}

# ส่ง alias ไปยัง module
module "vpc_us" {
  source = "./modules/vpc"
  
  providers = {
    aws = aws.us_east_1
  }
  
  name = "vpc-us"
  cidr = "10.1.0.0/16"
}

module "vpc_ap" {
  source = "./modules/vpc"
  
  providers = {
    aws = aws  # default provider
  }
  
  name = "vpc-ap"
  cidr = "10.0.0.0/16"
}
```

---

## Step 177: Implicit vs Explicit Provider Configuration

### Implicit Provider Configuration

```hcl
# ไม่ต้องระบุ provider block ถ้า default config เพียงพอ
# Terraform จะ auto-configure จาก environment variables

# AWS: ใช้ environment variables
# AWS_ACCESS_KEY_ID, AWS_SECRET_ACCESS_KEY, AWS_DEFAULT_REGION

resource "aws_instance" "web" {
  # ไม่มี explicit provider = aws
  # Terraform ใช้ default aws provider
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t3.micro"
}
```

### Explicit Provider Configuration

```hcl
# ระบุ provider อย่างชัดเจน

provider "aws" {
  region = "ap-southeast-1"
}

resource "aws_instance" "web" {
  provider      = aws  # explicit
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t3.micro"
}
```

---

## Step 178: Provider Authentication Methods

### AWS Authentication Methods

```hcl
# Method 1: Environment Variables (แนะนำสำหรับ CI/CD)
# export AWS_ACCESS_KEY_ID="xxx"
# export AWS_SECRET_ACCESS_KEY="xxx"
# export AWS_DEFAULT_REGION="ap-southeast-1"
provider "aws" {}  # อ่านจาก environment

# Method 2: AWS Profile
provider "aws" {
  region  = "ap-southeast-1"
  profile = "my-aws-profile"  # จาก ~/.aws/credentials
}

# Method 3: EC2 Instance Profile / ECS Task Role (แนะนำสำหรับ AWS)
provider "aws" {
  region = "ap-southeast-1"
  # ไม่ต้องระบุ credentials - ใช้ IAM role ที่ attach กับ instance/task
}

# Method 4: Assume Role
provider "aws" {
  region = "ap-southeast-1"
  
  assume_role {
    role_arn     = "arn:aws:iam::123456789012:role/TerraformRole"
    session_name = "TerraformSession"
    external_id  = var.external_id  # optional security
  }
}

# Method 5: Web Identity Token (EKS IRSA)
provider "aws" {
  region = "ap-southeast-1"
  
  assume_role_with_web_identity {
    role_arn                = "arn:aws:iam::123456789012:role/TerraformRole"
    web_identity_token_file = "/var/run/secrets/eks.amazonaws.com/serviceaccount/token"
    session_name            = "TerraformSession"
  }
}
```

### GCP Authentication Methods

```hcl
# Method 1: Service Account Key File
provider "google" {
  project     = var.project_id
  region      = var.region
  credentials = file("service-account-key.json")
}

# Method 2: Environment Variable
# export GOOGLE_APPLICATION_CREDENTIALS="/path/to/key.json"
provider "google" {
  project = var.project_id
  region  = var.region
}

# Method 3: Application Default Credentials (gcloud auth)
# gcloud auth application-default login
provider "google" {
  project = var.project_id
  region  = var.region
}

# Method 4: Workload Identity (แนะนำสำหรับ GKE)
provider "google" {
  project = var.project_id
  region  = var.region
  # ใช้ Workload Identity โดยอัตโนมัติ
}
```

### Azure Authentication Methods

```hcl
# Method 1: Service Principal with Client Secret
provider "azurerm" {
  features {}
  
  client_id       = var.azure_client_id
  client_secret   = var.azure_client_secret
  tenant_id       = var.azure_tenant_id
  subscription_id = var.azure_subscription_id
}

# Method 2: Environment Variables
# ARM_CLIENT_ID, ARM_CLIENT_SECRET, ARM_TENANT_ID, ARM_SUBSCRIPTION_ID
provider "azurerm" {
  features {}
}

# Method 3: Managed Identity (Azure VM/AKS)
provider "azurerm" {
  features {}
  use_msi = true
}

# Method 4: Azure CLI
# az login
provider "azurerm" {
  features {}
  use_cli = true
}
```

---

## Step 179: terraform init - Provider Initialization

### Provider Download Process

```bash
# terraform init downloads providers
terraform init

# Output:
# Initializing provider plugins...
# - Finding hashicorp/aws versions matching "~> 5.0"...
# - Finding hashicorp/random versions matching "~> 3.5"...
# - Installing hashicorp/aws v5.31.0...
# - Installed hashicorp/aws v5.31.0 (signed by HashiCorp)
# - Installing hashicorp/random v3.6.0...
# - Installed hashicorp/random v3.6.0 (signed by HashiCorp)

# Provider stored at:
ls .terraform/providers/registry.terraform.io/
# hashicorp/aws/5.31.0/linux_amd64/
# hashicorp/random/3.6.0/linux_amd64/
```

### .terraform.lock.hcl

```hcl
# .terraform.lock.hcl - generated automatically

# This file is maintained automatically by "terraform init".
# Manual edits may be lost in future updates.

provider "registry.terraform.io/hashicorp/aws" {
  version     = "5.31.0"
  constraints = "~> 5.0"
  hashes = [
    "h1:abc123...",
    "zh:def456...",
  ]
}

provider "registry.terraform.io/hashicorp/random" {
  version     = "3.6.0"
  constraints = "~> 3.5"
  hashes = [
    "h1:ghi789...",
    "zh:jkl012...",
  ]
}
```

```bash
# อัพเกรด providers
terraform init -upgrade

# Lock providers สำหรับ multiple platforms
terraform providers lock \
  -platform=linux_amd64 \
  -platform=darwin_amd64 \
  -platform=darwin_arm64
```

---

## Step 180: Provider Documentation Navigation

### วิธีอ่าน Provider Documentation

```
1. เปิด registry.terraform.io
2. ค้นหา provider (เช่น "aws")
3. เลือก provider และ version
4. Navigation structure:
   - Overview (Authentication, getting started)
   - Resources (all available resources)
   - Data Sources (all available data sources)
   - Guides (complex topics)

Resource Documentation Structure:
1. Example Usage
2. Argument Reference (inputs)
3. Attributes Reference (outputs/computed)
4. Timeouts
5. Import
```

### ตัวอย่าง Top 10 Providers

```hcl
# 1. AWS Provider
provider "aws" {
  region = "ap-southeast-1"
}

resource "aws_instance" "example" {
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t3.micro"
}

# 2. Google Cloud Provider
provider "google" {
  project = "my-project"
  region  = "asia-southeast1"
}

resource "google_compute_instance" "example" {
  name         = "example-vm"
  machine_type = "e2-micro"
  zone         = "asia-southeast1-a"
  
  boot_disk {
    initialize_params {
      image = "debian-cloud/debian-11"
    }
  }
  
  network_interface {
    network = "default"
  }
}

# 3. Azure Provider
provider "azurerm" {
  features {}
}

resource "azurerm_resource_group" "example" {
  name     = "example-rg"
  location = "Southeast Asia"
}

# 4. Kubernetes Provider
provider "kubernetes" {
  host                   = var.k8s_host
  cluster_ca_certificate = base64decode(var.k8s_ca_cert)
  token                  = var.k8s_token
}

resource "kubernetes_namespace" "example" {
  metadata {
    name = "my-namespace"
  }
}

# 5. Helm Provider
provider "helm" {
  kubernetes {
    host  = var.k8s_host
    token = var.k8s_token
  }
}

resource "helm_release" "nginx" {
  name       = "nginx-ingress"
  repository = "https://kubernetes.github.io/ingress-nginx"
  chart      = "ingress-nginx"
  version    = "4.9.0"
}

# 6. GitHub Provider
provider "github" {
  owner = "my-organization"
  token = var.github_token
}

resource "github_repository" "example" {
  name        = "example-repo"
  description = "Example repository managed by Terraform"
  visibility  = "private"
}

# 7. Datadog Provider
provider "datadog" {
  api_key = var.datadog_api_key
  app_key = var.datadog_app_key
}

resource "datadog_monitor" "cpu_high" {
  name    = "High CPU on production"
  type    = "metric alert"
  message = "CPU is too high! @pagerduty"
  query   = "avg(last_5m):avg:system.cpu.user{env:prod} > 90"
}

# 8. Cloudflare Provider
provider "cloudflare" {
  api_token = var.cloudflare_api_token
}

resource "cloudflare_record" "www" {
  zone_id = var.cloudflare_zone_id
  name    = "www"
  value   = aws_lb.main.dns_name
  type    = "CNAME"
  proxied = true
}

# 9. PagerDuty Provider
provider "pagerduty" {
  token = var.pagerduty_token
}

resource "pagerduty_service" "example" {
  name                    = "My Application"
  auto_resolve_timeout    = 14400
  acknowledgement_timeout = 600
  escalation_policy       = pagerduty_escalation_policy.example.id
}

# 10. MongoDB Atlas Provider
provider "mongodbatlas" {
  public_key  = var.mongodb_atlas_public_key
  private_key = var.mongodb_atlas_private_key
}

resource "mongodbatlas_cluster" "example" {
  project_id   = var.mongodb_project_id
  name         = "example-cluster"
  cluster_type = "REPLICASET"
  
  provider_name               = "AWS"
  provider_region_name        = "AP_SOUTHEAST_1"
  provider_instance_size_name = "M10"
}
```

---

## Custom/Internal Providers

### เมื่อไหรต้องสร้าง Custom Provider?

```
ใช้ custom provider เมื่อ:
- Internal APIs ที่ไม่มี public provider
- Legacy systems
- Custom automation platforms
- Internal secrets management
```

```go
// ตัวอย่างโครงสร้าง Custom Provider (Go)
// main.go
package main

import (
  "github.com/hashicorp/terraform-plugin-sdk/v2/helper/schema"
  "github.com/hashicorp/terraform-plugin-sdk/v2/plugin"
)

func main() {
  plugin.Serve(&plugin.ServeOpts{
    ProviderFunc: func() *schema.Provider {
      return &schema.Provider{
        ResourcesMap: map[string]*schema.Resource{
          "custom_thing": resourceThing(),
        },
      }
    },
  })
}
```

```hcl
# การใช้งาน custom provider
terraform {
  required_providers {
    custom = {
      source  = "my-company/custom"
      version = "~> 1.0"
    }
  }
}

provider "custom" {
  endpoint = "https://api.mycompany.com"
  token    = var.api_token
}

resource "custom_thing" "example" {
  name = "my-thing"
}
```

---

## สรุป (Summary)

### Provider Quick Reference

| Category | Provider | Source |
|----------|----------|--------|
| Cloud | AWS | `hashicorp/aws` |
| Cloud | GCP | `hashicorp/google` |
| Cloud | Azure | `hashicorp/azurerm` |
| Container | Kubernetes | `hashicorp/kubernetes` |
| Container | Helm | `hashicorp/helm` |
| Code | GitHub | `integrations/github` |
| Monitoring | Datadog | `DataDog/datadog` |
| CDN | Cloudflare | `cloudflare/cloudflare` |
| Alert | PagerDuty | `pagerduty/pagerduty` |
| Database | MongoDB Atlas | `mongodb/mongodbatlas` |

### ✅ Best Practices

1. **Pin version** ด้วย `~>` operator เสมอ
2. **Commit .terraform.lock.hcl** ใน git
3. **ใช้ instance roles** แทน access keys สำหรับ cloud
4. **aliases** สำหรับ multi-region/account
5. **default_tags** ใน AWS provider ลด code ซ้ำ

### ⚠️ Common Mistakes

```hcl
# ❌ ไม่ pin version
required_providers {
  aws = {
    source = "hashicorp/aws"
    # ขาด version! ไม่ดี
  }
}

# ❌ Hardcode credentials
provider "aws" {
  access_key = "AKIAIOSFODNN7EXAMPLE"
  secret_key = "wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY"
}

# ✅ ใช้ environment variables หรือ IAM roles
provider "aws" {
  region = var.region
  # auth จาก env vars หรือ instance role
}
```

---

*จบ Part 018 - Terraform Providers*
