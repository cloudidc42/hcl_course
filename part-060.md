# Part 060: Multi-Cloud Patterns
## ขั้นตอนที่ 591-600: รูปแบบและ Best Practices สำหรับ Multi-Cloud Architecture

---

## ขั้นตอนที่ 591: ทำไมต้องใช้ Multi-Cloud?

### เหตุผลสำหรับ Multi-Cloud

```
ข้อดี (Benefits):
┌─────────────────────────────────────────────────────────────┐
│ 1. Vendor Lock-in Avoidance                                  │
│    - ไม่พึ่งพา cloud provider เดียว                         │
│    - Negotiating power กับ vendors                          │
│                                                              │
│ 2. Best-of-Breed Services                                    │
│    - ใช้ AWS สำหรับ AI/ML (SageMaker)                       │
│    - ใช้ GCP สำหรับ Data Analytics (BigQuery)               │
│    - ใช้ Azure สำหรับ Enterprise (AD Integration)           │
│                                                              │
│ 3. Geographic Distribution                                   │
│    - บาง regions มีเฉพาะบาง cloud                          │
│    - Data residency requirements                            │
│                                                              │
│ 4. Disaster Recovery                                         │
│    - Cloud-level failure protection                          │
│    - Business continuity                                     │
│                                                              │
│ 5. Cost Optimization                                         │
│    - ใช้ cloud ที่ถูกกว่าสำหรับ workloads บางอย่าง         │
│    - Spot/Preemptible instances across clouds               │
└─────────────────────────────────────────────────────────────┘

ข้อเสีย (Challenges):
┌─────────────────────────────────────────────────────────────┐
│ 1. Increased Complexity                                      │
│    - หลาย tools และ APIs ต้องเรียนรู้                       │
│    - Security ซับซ้อนขึ้น                                   │
│                                                              │
│ 2. Data Transfer Costs                                       │
│    - Egress charges ระหว่าง clouds แพงมาก                   │
│                                                              │
│ 3. Operational Overhead                                      │
│    - หลาย monitoring systems                                 │
│    - หลาย IAM systems                                        │
│                                                              │
│ 4. Skill Requirements                                        │
│    - Team ต้องรู้หลาย cloud platforms                       │
└─────────────────────────────────────────────────────────────┘
```

---

## ขั้นตอนที่ 592: Multi-Cloud Provider Configuration

```hcl
# multi-cloud-providers.tf

terraform {
  required_version = ">= 1.5.0"
  
  required_providers {
    # AWS Provider
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
    
    # Azure Provider
    azurerm = {
      source  = "hashicorp/azurerm"
      version = "~> 3.80"
    }
    
    # GCP Provider
    google = {
      source  = "hashicorp/google"
      version = "~> 5.0"
    }
    
    # Additional providers
    cloudflare = {
      source  = "cloudflare/cloudflare"
      version = "~> 4.0"
    }
    
    datadog = {
      source  = "DataDog/datadog"
      version = "~> 3.0"
    }
    
    pagerduty = {
      source  = "PagerDuty/pagerduty"
      version = "~> 3.0"
    }
  }
  
  # Multi-cloud backend (Terraform Cloud)
  # cloud {
  #   organization = "my-org"
  #   workspaces {
  #     name = "multi-cloud-prod"
  #   }
  # }
}

# ============================================================
# AWS Providers
# ============================================================

provider "aws" {
  alias  = "primary"
  region = "ap-southeast-1"  # Singapore
  
  default_tags {
    tags = local.common_tags
  }
}

provider "aws" {
  alias  = "secondary"
  region = "ap-northeast-1"  # Tokyo
  
  default_tags {
    tags = local.common_tags
  }
}

provider "aws" {
  alias  = "us_east"
  region = "us-east-1"
  
  default_tags {
    tags = local.common_tags
  }
}

# ============================================================
# Azure Providers
# ============================================================

provider "azurerm" {
  alias = "southeast_asia"
  features {
    resource_group {
      prevent_deletion_if_contains_resources = true
    }
  }
  subscription_id = var.azure_subscription_id
}

provider "azurerm" {
  alias = "east_asia"
  features {}
  subscription_id = var.azure_subscription_id
  # location ระบุใน resources
}

# ============================================================
# GCP Providers
# ============================================================

provider "google" {
  alias   = "asia"
  project = var.gcp_project_id
  region  = "asia-southeast1"
}

provider "google" {
  alias   = "us"
  project = var.gcp_project_id
  region  = "us-central1"
}

# ============================================================
# Cloudflare (DNS)
# ============================================================

provider "cloudflare" {
  api_token = var.cloudflare_api_token
}
```

---

## ขั้นตอนที่ 593: Code Organization

```hcl
# ============================================================
# โครงสร้าง Directory สำหรับ Multi-Cloud
# ============================================================

/*
infrastructure/
├── modules/
│   ├── shared/                    # Cloud-agnostic modules
│   │   ├── dns/
│   │   ├── monitoring/
│   │   └── security/
│   │
│   ├── aws/                       # AWS-specific modules
│   │   ├── vpc/
│   │   ├── eks/
│   │   ├── rds/
│   │   └── s3/
│   │
│   ├── azure/                     # Azure-specific modules
│   │   ├── vnet/
│   │   ├── aks/
│   │   ├── sql/
│   │   └── storage/
│   │
│   └── gcp/                       # GCP-specific modules
│       ├── vpc/
│       ├── gke/
│       ├── cloudsql/
│       └── gcs/
│
├── environments/
│   ├── dev/
│   │   ├── main.tf                # ใช้ modules
│   │   ├── variables.tf
│   │   └── terraform.tfvars
│   │
│   ├── staging/
│   │   └── main.tf
│   │
│   └── prod/
│       ├── aws/                   # Per-cloud separation
│       │   └── main.tf
│       ├── azure/
│       │   └── main.tf
│       └── gcp/
│           └── main.tf
│
├── global/                        # Cross-cloud global resources
│   ├── dns.tf                     # Cloudflare DNS
│   ├── monitoring.tf              # Datadog/PagerDuty
│   └── certificates.tf
│
└── cross-cloud/                   # Cross-cloud connectivity
    ├── networking.tf              # VPN/Interconnects
    └── dns-failover.tf
*/

# ============================================================
# Multi-Cloud Variables
# ============================================================

# variables.tf
variable "environment" {
  type = string
}

variable "primary_cloud" {
  description = "Primary cloud provider"
  type        = string
  validation {
    condition     = contains(["aws", "azure", "gcp"], var.primary_cloud)
    error_message = "primary_cloud must be aws, azure, or gcp."
  }
}

variable "aws_region" {
  type    = string
  default = "ap-southeast-1"
}

variable "azure_location" {
  type    = string
  default = "Southeast Asia"
}

variable "gcp_region" {
  type    = string
  default = "asia-southeast1"
}

variable "aws_account_id" {
  type = string
}

variable "azure_subscription_id" {
  type = string
}

variable "gcp_project_id" {
  type = string
}

# Common tags/labels
locals {
  common_tags = {
    Environment  = var.environment
    ManagedBy    = "Terraform"
    Architecture = "multi-cloud"
    Team         = var.team
  }
}
```

---

## ขั้นตอนที่ 594: Module Per Cloud Pattern

```hcl
# ============================================================
# Shared Module Interface (Abstract Layer)
# ============================================================

# modules/abstract/kubernetes/variables.tf
variable "cluster_name" {
  type = string
}

variable "node_count" {
  type    = number
  default = 3
}

variable "node_size" {
  description = "Cloud-specific node size (mapped internally)"
  type        = string
  default     = "medium"  # small, medium, large
}

variable "kubernetes_version" {
  type    = string
  default = "1.28"
}

variable "cloud" {
  type = string
  validation {
    condition     = contains(["aws", "azure", "gcp"], var.cloud)
    error_message = "cloud must be aws, azure, or gcp."
  }
}

# ============================================================
# AWS EKS Module
# ============================================================

# modules/aws/kubernetes/main.tf
locals {
  node_sizes = {
    small  = "t3.medium"
    medium = "m5.large"
    large  = "m5.xlarge"
  }
}

resource "aws_eks_cluster" "main" {
  name     = var.cluster_name
  version  = var.kubernetes_version
  role_arn = aws_iam_role.eks.arn
  
  vpc_config {
    subnet_ids         = var.subnet_ids
    security_group_ids = [aws_security_group.eks.id]
  }
}

resource "aws_eks_node_group" "main" {
  cluster_name    = aws_eks_cluster.main.name
  node_group_name = "main"
  node_role_arn   = aws_iam_role.eks_nodes.arn
  subnet_ids      = var.subnet_ids
  
  instance_types = [local.node_sizes[var.node_size]]
  
  scaling_config {
    desired_size = var.node_count
    min_size     = var.node_count
    max_size     = var.node_count * 3
  }
}

# ============================================================
# Azure AKS Module
# ============================================================

# modules/azure/kubernetes/main.tf
locals {
  node_sizes = {
    small  = "Standard_B2s"
    medium = "Standard_D2s_v3"
    large  = "Standard_D4s_v3"
  }
}

resource "azurerm_kubernetes_cluster" "main" {
  name                = var.cluster_name
  resource_group_name = var.resource_group_name
  location            = var.location
  dns_prefix          = var.cluster_name
  kubernetes_version  = var.kubernetes_version
  
  default_node_pool {
    name                = "system"
    vm_size             = local.node_sizes[var.node_size]
    node_count          = var.node_count
    enable_auto_scaling = true
    min_count           = var.node_count
    max_count           = var.node_count * 3
  }
  
  identity {
    type = "SystemAssigned"
  }
}

# ============================================================
# GCP GKE Module
# ============================================================

# modules/gcp/kubernetes/main.tf
locals {
  node_sizes = {
    small  = "e2-medium"
    medium = "n2-standard-2"
    large  = "n2-standard-4"
  }
}

resource "google_container_cluster" "main" {
  name     = var.cluster_name
  location = var.region
  project  = var.project_id
  
  remove_default_node_pool = true
  initial_node_count       = 1
}

resource "google_container_node_pool" "main" {
  name       = "main"
  location   = var.region
  cluster    = google_container_cluster.main.name
  node_count = var.node_count
  
  autoscaling {
    min_node_count = var.node_count
    max_node_count = var.node_count * 3
  }
  
  node_config {
    machine_type = local.node_sizes[var.node_size]
    oauth_scopes = ["https://www.googleapis.com/auth/cloud-platform"]
  }
}
```

---

## ขั้นตอนที่ 595: Multi-Cloud Networking Patterns

```hcl
# multi-cloud-networking.tf

# ============================================================
# AWS-to-Azure Site-to-Site VPN
# ============================================================

# AWS Side
resource "aws_vpn_gateway" "to_azure" {
  vpc_id = aws_vpc.main.id
  
  tags = {
    Name = "vpn-gw-to-azure"
    ManagedBy = "Terraform"
  }
}

resource "aws_customer_gateway" "azure" {
  bgp_asn    = 65000  # Azure ASN
  ip_address = azurerm_public_ip.vpn.ip_address
  type       = "ipsec.1"
  
  tags = {
    Name = "cgw-azure"
  }
}

resource "aws_vpn_connection" "to_azure" {
  vpn_gateway_id      = aws_vpn_gateway.to_azure.id
  customer_gateway_id = aws_customer_gateway.azure.id
  type                = "ipsec.1"
  static_routes_only  = false
  
  tags = {
    Name = "vpn-aws-to-azure"
  }
}

# Azure Side
resource "azurerm_public_ip" "vpn" {
  name                = "pip-vpn-gw"
  resource_group_name = azurerm_resource_group.network.name
  location            = azurerm_resource_group.network.location
  allocation_method   = "Static"
  sku                 = "Standard"
}

resource "azurerm_virtual_network_gateway" "main" {
  name                = "vng-main-prod"
  resource_group_name = azurerm_resource_group.network.name
  location            = azurerm_resource_group.network.location
  
  type     = "Vpn"
  vpn_type = "RouteBased"
  sku      = "VpnGw1"
  
  active_active = false
  enable_bgp    = true
  
  bgp_settings {
    asn = 65000
  }
  
  ip_configuration {
    name                 = "vnetGatewayConfig"
    public_ip_address_id = azurerm_public_ip.vpn.id
    subnet_id            = azurerm_subnet.gateway.id
  }
}

resource "azurerm_local_network_gateway" "aws" {
  name                = "lng-aws"
  resource_group_name = azurerm_resource_group.network.name
  location            = azurerm_resource_group.network.location
  
  gateway_address = aws_vpn_connection.to_azure.tunnel1_address
  
  address_space = ["10.0.0.0/8"]  # AWS VPC CIDR
  
  bgp_settings {
    asn                 = 64512
    bgp_peering_address = aws_vpn_connection.to_azure.tunnel1_vgw_inside_address
  }
}

resource "azurerm_virtual_network_gateway_connection" "to_aws" {
  name                = "conn-to-aws"
  resource_group_name = azurerm_resource_group.network.name
  location            = azurerm_resource_group.network.location
  
  type                       = "IPsec"
  virtual_network_gateway_id = azurerm_virtual_network_gateway.main.id
  local_network_gateway_id   = azurerm_local_network_gateway.aws.id
  
  shared_key             = var.vpn_shared_key
  enable_bgp             = true
  connection_protocol    = "IKEv2"
  
  ipsec_policy {
    dh_group         = "DHGroup14"
    ike_encryption   = "AES256"
    ike_integrity    = "SHA256"
    ipsec_encryption = "AES256"
    ipsec_integrity  = "SHA256"
    pfs_group        = "PFS14"
    sa_lifetime      = 27000
    sa_datasize      = 102400000
  }
}

# ============================================================
# GCP to AWS VPN
# ============================================================

# GCP Side
resource "google_compute_vpn_gateway" "to_aws" {
  name    = "vpn-gw-to-aws"
  network = google_compute_network.main.id
  region  = var.gcp_region
}

resource "google_compute_address" "vpn_gw_ip" {
  name   = "ip-vpn-gw"
  region = var.gcp_region
}

resource "google_compute_forwarding_rule" "vpn_esp" {
  name        = "vpn-esp"
  ip_address  = google_compute_address.vpn_gw_ip.address
  ip_protocol = "ESP"
  target      = google_compute_vpn_gateway.to_aws.id
}

resource "google_compute_forwarding_rule" "vpn_udp500" {
  name        = "vpn-udp-500"
  ip_address  = google_compute_address.vpn_gw_ip.address
  ip_protocol = "UDP"
  port_range  = "500"
  target      = google_compute_vpn_gateway.to_aws.id
}

resource "google_compute_vpn_tunnel" "to_aws" {
  name               = "vpn-tunnel-to-aws"
  peer_ip            = aws_vpn_connection.to_gcp.tunnel1_address
  shared_secret      = aws_vpn_connection.to_gcp.tunnel1_preshared_key
  target_vpn_gateway = google_compute_vpn_gateway.to_aws.id
  
  ike_version = 2
  
  local_traffic_selector  = ["0.0.0.0/0"]
  remote_traffic_selector = ["0.0.0.0/0"]
}
```

---

## ขั้นตอนที่ 596: Multi-Cloud DNS

```hcl
# multi-cloud-dns.tf - Cloudflare as authoritative DNS

# Cloudflare Zone
data "cloudflare_zone" "main" {
  name = "company.com"
}

# Primary: AWS ALB
resource "cloudflare_record" "api_aws" {
  zone_id = data.cloudflare_zone.main.id
  name    = "api"
  value   = aws_lb.main.dns_name
  type    = "CNAME"
  ttl     = 60
  proxied = true  # Go through Cloudflare proxy
}

# DNS Failover using Cloudflare Load Balancing
resource "cloudflare_load_balancer_pool" "aws" {
  account_id = var.cloudflare_account_id
  name       = "aws-primary"
  
  origins {
    name    = "aws-singapore"
    address = aws_lb.main.dns_name
    enabled = true
  }
  
  health_threshold   = 1
  enabled            = true
  minimum_origins    = 1
  notification_email = "ops@company.com"
  
  load_shedding {
    default_percent = 0
    default_policy  = "random"
  }
}

resource "cloudflare_load_balancer_pool" "azure" {
  account_id = var.cloudflare_account_id
  name       = "azure-failover"
  
  origins {
    name    = "azure-sea"
    address = azurerm_public_ip.lb.fqdn
    enabled = true
  }
  
  health_threshold   = 1
  enabled            = true
  minimum_origins    = 1
  notification_email = "ops@company.com"
}

resource "cloudflare_load_balancer_pool" "gcp" {
  account_id = var.cloudflare_account_id
  name       = "gcp-failover"
  
  origins {
    name    = "gcp-sea"
    address = google_compute_global_address.lb.address
    enabled = true
  }
  
  health_threshold   = 1
  enabled            = true
  minimum_origins    = 1
}

# Health Check
resource "cloudflare_load_balancer_monitor" "https" {
  account_id     = var.cloudflare_account_id
  type           = "https"
  path           = "/health"
  expected_codes = "200"
  interval       = 60
  retries        = 2
  timeout        = 5
  
  header {
    header = "Host"
    values = ["api.company.com"]
  }
}

# Load Balancer with DNS Failover
resource "cloudflare_load_balancer" "api" {
  zone_id          = data.cloudflare_zone.main.id
  name             = "api.company.com"
  description      = "Multi-cloud API load balancer with failover"
  proxied          = true
  session_affinity = "none"
  ttl              = 30
  
  default_pool_ids = [cloudflare_load_balancer_pool.aws.id]
  
  fallback_pool_id = cloudflare_load_balancer_pool.azure.id
  
  # Geo-routing
  pop_pools {
    pop      = "SIN"  # Singapore
    pool_ids = [cloudflare_load_balancer_pool.aws.id]
  }
  
  pop_pools {
    pop      = "HKG"  # Hong Kong
    pool_ids = [cloudflare_load_balancer_pool.gcp.id]
  }
  
  # Region-based routing
  region_pools {
    region   = "SEAS"  # Southeast Asia
    pool_ids = [cloudflare_load_balancer_pool.aws.id]
  }
  
  rules {
    name      = "failover-rule"
    condition = "cf.region_code == \"SEAS\""
    
    overrides {
      default_pools = [cloudflare_load_balancer_pool.aws.id]
      
      fallback_pool = cloudflare_load_balancer_pool.azure.id
    }
  }
  
  adaptive_routing {
    failover_across_pools = true
  }
}
```

---

## ขั้นตอนที่ 597: Multi-Cloud Identity Federation

```hcl
# multi-cloud-identity.tf

# ============================================================
# AWS - Azure Identity Federation
# ============================================================

# Azure App Registration สำหรับ AWS Workload
resource "azuread_application" "aws_workload" {
  display_name = "aws-workload-federation"
  
  web {
    redirect_uris = ["https://sts.amazonaws.com/"]
  }
}

resource "azuread_service_principal" "aws_workload" {
  client_id = azuread_application.aws_workload.client_id
}

# Federated credentials สำหรับ AWS OIDC
resource "azuread_application_federated_identity_credential" "aws" {
  application_id = "/applications/${azuread_application.aws_workload.object_id}"
  display_name   = "aws-oidc"
  
  audiences = ["sts.amazonaws.com"]
  issuer    = "https://token.actions.githubusercontent.com"
  subject   = "repo:${var.github_org}/${var.github_repo}:environment:production"
}

# ============================================================
# Unified Secrets Management
# ============================================================

# AWS Secrets Manager -> Azure Key Vault sync (via Lambda)
resource "aws_secretsmanager_secret" "db_password" {
  name        = "prod/database/password"
  description = "Database password synchronized across clouds"
  
  replica {
    region = "ap-northeast-1"
  }
}

resource "aws_secretsmanager_secret_version" "db_password" {
  secret_id = aws_secretsmanager_secret.db_password.id
  secret_string = jsonencode({
    password = random_password.db.result
  })
}

# Azure Key Vault - same secret
resource "azurerm_key_vault_secret" "db_password" {
  name         = "database-password"
  value        = random_password.db.result
  key_vault_id = azurerm_key_vault.main.id
}

# GCP Secret Manager - same secret
resource "google_secret_manager_secret" "db_password" {
  secret_id = "database-password"
  project   = var.gcp_project_id
  
  replication {
    user_managed {
      replicas {
        location = "asia-southeast1"
      }
      replicas {
        location = "asia-northeast1"
      }
    }
  }
}

resource "google_secret_manager_secret_version" "db_password" {
  secret      = google_secret_manager_secret.db_password.id
  secret_data = random_password.db.result
}
```

---

## ขั้นตอนที่ 598: Multi-Cloud Kubernetes

```hcl
# multi-cloud-kubernetes.tf

# ============================================================
# AWS EKS
# ============================================================

module "eks" {
  source  = "terraform-aws-modules/eks/aws"
  version = "~> 20.0"
  
  cluster_name    = "eks-myapp-prod"
  cluster_version = "1.28"
  
  vpc_id     = module.vpc.vpc_id
  subnet_ids = module.vpc.private_subnets
  
  eks_managed_node_groups = {
    main = {
      instance_types = ["m5.large"]
      min_size       = 3
      max_size       = 20
      desired_size   = 3
    }
  }
  
  tags = local.common_tags
}

# ============================================================
# Azure AKS
# ============================================================

resource "azurerm_kubernetes_cluster" "main" {
  name                = "aks-myapp-prod"
  resource_group_name = azurerm_resource_group.aks.name
  location            = azurerm_resource_group.aks.location
  dns_prefix          = "aks-myapp-prod"
  
  default_node_pool {
    name                = "system"
    vm_size             = "Standard_D4s_v3"
    node_count          = 3
    enable_auto_scaling = true
    min_count           = 3
    max_count           = 20
  }
  
  identity {
    type = "SystemAssigned"
  }
}

# ============================================================
# GCP GKE
# ============================================================

resource "google_container_cluster" "main" {
  name                     = "gke-myapp-prod"
  location                 = "asia-southeast1"
  project                  = var.gcp_project_id
  remove_default_node_pool = true
  initial_node_count       = 1
}

resource "google_container_node_pool" "main" {
  name       = "main"
  cluster    = google_container_cluster.main.name
  location   = "asia-southeast1"
  node_count = 3
  
  autoscaling {
    min_node_count = 3
    max_node_count = 20
  }
  
  node_config {
    machine_type = "n2-standard-4"
    oauth_scopes = ["https://www.googleapis.com/auth/cloud-platform"]
  }
}

# ============================================================
# Outputs สำหรับ kubectl config
# ============================================================

output "kubeconfig_commands" {
  value = {
    eks   = "aws eks update-kubeconfig --name ${module.eks.cluster_name} --region ap-southeast-1"
    aks   = "az aks get-credentials --name ${azurerm_kubernetes_cluster.main.name} --resource-group ${azurerm_resource_group.aks.name}"
    gke   = "gcloud container clusters get-credentials ${google_container_cluster.main.name} --region asia-southeast1 --project ${var.gcp_project_id}"
  }
}
```

---

## ขั้นตอนที่ 599: Multi-Cloud Storage

```hcl
# multi-cloud-storage.tf

# ============================================================
# Multi-Cloud Object Storage
# ============================================================

# AWS S3
resource "aws_s3_bucket" "primary" {
  bucket = "${var.app_name}-${var.environment}-primary"
  
  tags = local.common_tags
}

resource "aws_s3_bucket_versioning" "primary" {
  bucket = aws_s3_bucket.primary.id
  
  versioning_configuration {
    status = "Enabled"
  }
}

# Replicate S3 to GCS
resource "aws_s3_bucket_replication_configuration" "to_gcs" {
  depends_on = [aws_s3_bucket_versioning.primary]
  
  role   = aws_iam_role.replication.arn
  bucket = aws_s3_bucket.primary.id
  
  rule {
    id     = "to-gcs"
    status = "Enabled"
    
    destination {
      bucket        = "arn:aws:s3:::interop-bucket"  # GCS bucket via interop
      storage_class = "STANDARD"
    }
  }
}

# Azure Blob Storage
resource "azurerm_storage_account" "secondary" {
  name                     = "st${var.app_name}${var.environment}001"
  resource_group_name      = azurerm_resource_group.main.name
  location                 = azurerm_resource_group.main.location
  account_kind             = "StorageV2"
  account_tier             = "Standard"
  account_replication_type = "GRS"
}

# GCP Cloud Storage
resource "google_storage_bucket" "tertiary" {
  name          = "${var.gcp_project_id}-${var.app_name}-${var.environment}"
  location      = "ASIA"
  storage_class = "STANDARD"
  project       = var.gcp_project_id
  
  versioning {
    enabled = true
  }
  
  uniform_bucket_level_access = true
}

# ============================================================
# Multi-Cloud Database Comparison
# ============================================================

locals {
  database_endpoints = {
    aws_rds = {
      engine   = "PostgreSQL"
      endpoint = aws_db_instance.main.endpoint
      port     = 5432
    }
    azure_postgresql = {
      engine   = "PostgreSQL"
      endpoint = azurerm_postgresql_flexible_server.main.fqdn
      port     = 5432
    }
    gcp_cloudsql = {
      engine   = "PostgreSQL"
      endpoint = google_sql_database_instance.main.private_ip_address
      port     = 5432
    }
  }
}
```

---

## ขั้นตอนที่ 600: Complete Multi-Cloud Example

```hcl
# complete-multi-cloud.tf
# DNS Failover: Primary AWS, Failover Azure, DR GCP

terraform {
  required_version = ">= 1.5.0"
  
  required_providers {
    aws      = { source = "hashicorp/aws",      version = "~> 5.0" }
    azurerm  = { source = "hashicorp/azurerm",  version = "~> 3.80" }
    google   = { source = "hashicorp/google",   version = "~> 5.0" }
    cloudflare = { source = "cloudflare/cloudflare", version = "~> 4.0" }
  }
}

# ============================================================
# Terraform Abstraction Layer
# ============================================================

locals {
  # Cloud selection for each service
  services = {
    frontend = {
      primary   = "aws"
      secondary = "azure"
      dr        = "gcp"
    }
    api = {
      primary   = "aws"
      secondary = "azure"
      dr        = "gcp"
    }
    data = {
      primary   = "aws"    # RDS PostgreSQL
      secondary = "azure"  # Azure PostgreSQL
      dr        = "gcp"    # Cloud SQL
    }
  }
  
  # Cloud regions mapping
  cloud_regions = {
    aws = {
      primary   = "ap-southeast-1"
      secondary = "ap-northeast-1"
    }
    azure = {
      primary   = "Southeast Asia"
      secondary = "East Asia"
    }
    gcp = {
      primary   = "asia-southeast1"
      secondary = "asia-northeast1"
    }
  }
}

# ============================================================
# Monitoring: Datadog across all clouds
# ============================================================

resource "datadog_monitor" "multi_cloud_api" {
  name    = "Multi-Cloud API Health"
  type    = "metric alert"
  message = "API endpoint down! @pagerduty-critical"
  
  query = "min(last_5m):avg:trace.web.request.hits{env:production,service:api} < 1"
  
  thresholds {
    critical = 1
  }
  
  tags = ["env:production", "service:api", "cloud:multi"]
}

# ============================================================
# Cost Comparison Locals
# ============================================================

locals {
  # Monthly cost estimates (USD) สำหรับ similar configurations
  cost_estimates = {
    aws = {
      compute   = 280   # 3x m5.large
      database  = 150   # db.m5.large Multi-AZ
      storage   = 25    # 1 TB S3 + data transfer
      network   = 50    # NAT + LB
      total     = 505
    }
    azure = {
      compute   = 260   # 3x Standard_D2s_v3
      database  = 170   # GP_Standard_D2s_v3
      storage   = 20    # GRS storage
      network   = 45    # NAT + LB
      total     = 495
    }
    gcp = {
      compute   = 245   # 3x n2-standard-2
      database  = 145   # db-custom-2-7680
      storage   = 22    # Regional + operations
      network   = 40    # NAT + LB
      total     = 452
    }
  }
}

# ============================================================
# Anti-Patterns to Avoid
# ============================================================

/*
❌ Anti-Pattern 1: Same workload on multiple clouds simultaneously
   - เพิ่ม complexity โดยไม่จำเป็น
   - Data sync ยาก
   - ค่าใช้จ่าย egress สูง

✅ Pattern: Active-Passive failover ใช้ DNS

❌ Anti-Pattern 2: Hard coupling between clouds
   - Microservice ใน AWS เรียก microservice ใน GCP โดยตรง
   - ขึ้นอยู่กับ cross-cloud latency

✅ Pattern: Async communication (queues, events), loose coupling

❌ Anti-Pattern 3: Trying to abstract away everything
   - "Cloud-agnostic" layer ที่ซับซ้อนมาก
   - ไม่ใช้ strengths ของแต่ละ cloud

✅ Pattern: Accept cloud-specific services, use abstraction only where it makes sense

❌ Anti-Pattern 4: Replicating ALL data across clouds
   - Egress costs ระเบิด
   - Consistency issues

✅ Pattern: Replicate only necessary data, use CDN for static content

❌ Anti-Pattern 5: No governance model
   - Different teams use different clouds without coordination
   - No unified monitoring/security

✅ Pattern: Central platform team, unified tooling (Terraform, Datadog, etc.)
*/

# ============================================================
# Multi-Cloud Outputs Summary
# ============================================================

output "infrastructure_summary" {
  value = {
    aws = {
      region      = var.aws_region
      vpc_id      = module.aws_vpc.vpc_id
      eks_cluster = module.eks.cluster_name
      rds_endpoint = aws_db_instance.main.endpoint
    }
    azure = {
      location       = var.azure_location
      vnet_id        = azurerm_virtual_network.main.id
      aks_cluster    = azurerm_kubernetes_cluster.main.name
      sql_server     = azurerm_mssql_server.main.fully_qualified_domain_name
    }
    gcp = {
      project        = var.gcp_project_id
      region         = var.gcp_region
      gke_cluster    = google_container_cluster.main.name
      sql_instance   = google_sql_database_instance.main.connection_name
    }
    dns = {
      cloudflare_zone = data.cloudflare_zone.main.name
      api_endpoint    = "https://api.${data.cloudflare_zone.main.name}"
    }
  }
}

# ============================================================
# Cost Optimization Recommendations
# ============================================================

output "cost_recommendations" {
  value = {
    aws = {
      savings_plan       = "Purchase 1-year Compute Savings Plan for ~30% savings"
      reserved_instances = "Reserve RDS instances for ~40% savings"
      right_sizing       = "Review underutilized instances monthly"
    }
    azure = {
      reserved_vms    = "Purchase 1-year Reserved VM Instances"
      dev_shutdown    = "Auto-shutdown dev VMs after hours"
      spot_usage      = "Use Spot VMs for batch workloads"
    }
    gcp = {
      committed_use   = "1-year CUD for compute: 37% discount"
      preemptible     = "Use preemptible/spot for batch jobs"
      sustained_use   = "Automatic discount for sustained use"
    }
  }
}
```

---

## Multi-Cloud Architecture Summary

```
                    ┌─────────────────────────────────┐
                    │    Cloudflare (DNS + CDN)        │
                    │    Global Load Balancer          │
                    └──────────┬──────────┬────────────┘
                               │          │
                    ┌──────────┴┐        ┌┴──────────┐
                    │  AWS SEA  │        │ Azure SEA │
                    │ (Primary) │◄──VPN──│ (Failover)│
                    │           │        │           │
                    │ EKS      │        │ AKS       │
                    │ RDS PG   │        │ Azure SQL │
                    │ S3       │        │ Blob      │
                    └──────────┘        └───────────┘
                               │
                    ┌──────────┴────────┐
                    │    GCP SEA        │
                    │    (DR/Backup)    │
                    │                  │
                    │    GKE           │
                    │    Cloud SQL     │
                    │    GCS           │
                    └──────────────────┘

                    ┌─────────────────────────────────┐
                    │         Observability            │
                    │  Datadog + PagerDuty + OpsGenie  │
                    └─────────────────────────────────┘
                    
                    ┌─────────────────────────────────┐
                    │         IaC Platform             │
                    │  Terraform Cloud / Atlantis      │
                    │  (Single pane of glass)          │
                    └─────────────────────────────────┘
```

---

## สรุป Multi-Cloud Best Practices

| หัวข้อ | คำแนะนำ |
|--------|---------|
| DNS | ใช้ Cloudflare/Route53 เป็น Global DNS |
| Monitoring | Unified platform (Datadog, Grafana Cloud) |
| Security | Zero-trust, consistent policies |
| Networking | Hub-spoke per cloud, VPN between |
| IaC | Terraform สำหรับทุก cloud |
| Cost | Track per cloud, per service |
| Identity | Workload Identity Federation |
| Data | Minimize cross-cloud data transfer |

---

*จบ Part 060: Multi-Cloud Patterns*  
*จบ Series Part 051-060: Azure, GCP, and Multi-Cloud Terraform*
