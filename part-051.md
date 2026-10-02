# Part 051: Azure Provider Setup & Authentication
## ขั้นตอนที่ 501-510: การตั้งค่า Azure Provider และการยืนยันตัวตน

---

## ขั้นตอนที่ 501: Azure Provider (azurerm) และเวอร์ชัน

Azure Provider หรือ `azurerm` คือ Terraform provider หลักสำหรับการจัดการทรัพยากรบน Microsoft Azure การเลือกเวอร์ชันที่ถูกต้องมีความสำคัญมากเพื่อความเสถียรและการใช้งาน features ใหม่ๆ

### เวอร์ชันปัจจุบันและการเลือกใช้

```hcl
# terraform.tf - Basic provider version configuration
terraform {
  required_version = ">= 1.5.0"
  
  required_providers {
    azurerm = {
      source  = "hashicorp/azurerm"
      version = "~> 3.80"  # Use latest 3.x, compatible updates only
    }
  }
}

# Provider block - minimal configuration
provider "azurerm" {
  features {}
}
```

### การกำหนด Version Constraints

```hcl
# version-constraints.tf
terraform {
  required_version = ">= 1.5.0, < 2.0.0"
  
  required_providers {
    azurerm = {
      source  = "hashicorp/azurerm"
      # ~> 3.80 หมายถึง >= 3.80.0, < 4.0.0 (patch updates allowed)
      version = "~> 3.80"
    }
    
    # Additional providers
    azuread = {
      source  = "hashicorp/azuread"
      version = "~> 2.45"
    }
    
    random = {
      source  = "hashicorp/random"
      version = "~> 3.5"
    }
    
    tls = {
      source  = "hashicorp/tls"
      version = "~> 4.0"
    }
  }
}
```

---

## ขั้นตอนที่ 502: Required Providers Configuration

การกำหนด `required_providers` อย่างละเอียดช่วยให้ทีมใช้งาน provider versions ที่สอดคล้องกัน

```hcl
# providers.tf - Complete provider setup
terraform {
  required_version = ">= 1.5.0"
  
  required_providers {
    azurerm = {
      source  = "hashicorp/azurerm"
      version = "~> 3.80"
    }
    azuread = {
      source  = "hashicorp/azuread"
      version = "~> 2.45"
    }
    azapi = {
      source  = "azure/azapi"  # For new/preview Azure APIs
      version = "~> 1.10"
    }
  }
  
  # Backend configuration (Azure Blob Storage)
  backend "azurerm" {
    resource_group_name  = "rg-terraform-state"
    storage_account_name = "stterraformstate001"
    container_name       = "tfstate"
    key                  = "prod.terraform.tfstate"
  }
}

# azurerm provider with full features configuration
provider "azurerm" {
  features {
    # Key Vault settings
    key_vault {
      purge_soft_delete_on_destroy    = true
      recover_soft_deleted_key_vaults = true
    }
    
    # Virtual Machine settings
    virtual_machine {
      delete_os_disk_on_deletion     = true
      graceful_shutdown              = false
      skip_shutdown_and_force_delete = false
    }
    
    # Resource Group settings
    resource_group {
      prevent_deletion_if_contains_resources = true
    }
    
    # Cognitive Account
    cognitive_account {
      purge_soft_delete_on_destroy = true
    }
    
    # API Management
    api_management {
      purge_soft_delete_on_destroy = true
      recover_soft_deleted          = true
    }
    
    # App Configuration
    app_configuration {
      purge_soft_delete_on_destroy = true
      recover_soft_deleted          = true
    }
    
    # Application Insights
    application_insights {
      disable_generated_rule = false
    }
    
    # Log Analytics Workspace
    log_analytics_workspace {
      permanently_delete_on_destroy = true
    }
    
    # Template Deployment
    template_deployment {
      delete_nested_items_during_deletion = true
    }
  }
}
```

---

## ขั้นตอนที่ 503: วิธีการยืนยันตัวตน - Azure CLI

วิธีที่ง่ายที่สุดสำหรับ Local Development คือการใช้ Azure CLI

```bash
# ติดตั้ง Azure CLI
# Ubuntu/Debian
curl -sL https://aka.ms/InstallAzureCLIDeb | sudo bash

# macOS
brew install azure-cli

# Windows
winget install Microsoft.AzureCLI

# ล็อกอินด้วย Azure CLI
az login

# ดูรายการ subscriptions
az account list --output table

# เลือก subscription ที่ต้องการ
az account set --subscription "YOUR-SUBSCRIPTION-ID"

# ตรวจสอบ subscription ปัจจุบัน
az account show

# ล็อกอินด้วย Service Principal (สำหรับ automation)
az login --service-principal \
  --username $ARM_CLIENT_ID \
  --password $ARM_CLIENT_SECRET \
  --tenant $ARM_TENANT_ID
```

```hcl
# auth-azure-cli.tf
# เมื่อใช้ Azure CLI ไม่ต้องกำหนด credentials ใน provider
provider "azurerm" {
  features {}
  
  # Optional: specify subscription explicitly
  subscription_id = "00000000-0000-0000-0000-000000000000"
  tenant_id       = "00000000-0000-0000-0000-000000000001"
  
  # Optional: skip specific provider registrations for faster init
  skip_provider_registration = false
}

# ตรวจสอบ current user
data "azurerm_client_config" "current" {}

output "current_client_id" {
  value = data.azurerm_client_config.current.client_id
}

output "current_tenant_id" {
  value = data.azurerm_client_config.current.tenant_id
}

output "current_subscription_id" {
  value = data.azurerm_client_config.current.subscription_id
}

output "current_object_id" {
  value = data.azurerm_client_config.current.object_id
}
```

---

## ขั้นตอนที่ 504: Service Principal with Client Secret

Service Principal คือ "identity" สำหรับ application ที่ใช้ใน CI/CD pipelines

### การสร้าง Service Principal

```bash
# สร้าง Service Principal พร้อม Contributor role
az ad sp create-for-rbac \
  --name "sp-terraform-prod" \
  --role Contributor \
  --scopes "/subscriptions/YOUR-SUBSCRIPTION-ID" \
  --output json

# Output จะได้:
# {
#   "appId": "CLIENT_ID",
#   "displayName": "sp-terraform-prod",
#   "password": "CLIENT_SECRET",
#   "tenant": "TENANT_ID"
# }

# สร้าง SP สำหรับหลาย resource groups
az ad sp create-for-rbac \
  --name "sp-terraform-app1" \
  --role Contributor \
  --scopes \
    "/subscriptions/SUB-ID/resourceGroups/rg-app1-prod" \
    "/subscriptions/SUB-ID/resourceGroups/rg-app1-dev" \
  --output json

# ดู SP ที่มีอยู่
az ad sp list --display-name "sp-terraform" --output table

# ดูรายละเอียด SP
az ad sp show --id CLIENT_ID

# Reset credentials ของ SP
az ad sp credential reset --id CLIENT_ID --output json
```

### Terraform Configuration with Client Secret

```hcl
# auth-service-principal.tf
variable "arm_client_id" {
  description = "Azure Service Principal Client ID"
  type        = string
  sensitive   = true
}

variable "arm_client_secret" {
  description = "Azure Service Principal Client Secret"
  type        = string
  sensitive   = true
}

variable "arm_tenant_id" {
  description = "Azure Tenant ID"
  type        = string
}

variable "arm_subscription_id" {
  description = "Azure Subscription ID"
  type        = string
}

provider "azurerm" {
  features {}
  
  client_id       = var.arm_client_id
  client_secret   = var.arm_client_secret
  tenant_id       = var.arm_tenant_id
  subscription_id = var.arm_subscription_id
}
```

### Environment Variables สำหรับ Service Principal

```bash
# กำหนด environment variables
export ARM_CLIENT_ID="00000000-0000-0000-0000-000000000000"
export ARM_CLIENT_SECRET="your-client-secret-value"
export ARM_TENANT_ID="00000000-0000-0000-0000-000000000001"
export ARM_SUBSCRIPTION_ID="00000000-0000-0000-0000-000000000002"

# ตรวจสอบ environment variables
printenv | grep ARM_

# สร้าง .env file (อย่า commit ไปใน git!)
cat > .env << 'EOF'
ARM_CLIENT_ID=00000000-0000-0000-0000-000000000000
ARM_CLIENT_SECRET=your-client-secret-value
ARM_TENANT_ID=00000000-0000-0000-0000-000000000001
ARM_SUBSCRIPTION_ID=00000000-0000-0000-0000-000000000002
EOF

# โหลด .env file
set -a && source .env && set +a

# เพิ่ม .env ใน .gitignore
echo ".env" >> .gitignore
echo "*.tfvars" >> .gitignore
```

```hcl
# เมื่อใช้ environment variables ไม่ต้องกำหนดใน provider block
provider "azurerm" {
  features {}
  # ARM_* environment variables จะถูกอ่านอัตโนมัติ
}
```

---

## ขั้นตอนที่ 505: Service Principal with Certificate

การใช้ Certificate แทน Password ปลอดภัยกว่าเพราะ certificate สามารถ rotate ได้

```bash
# สร้าง self-signed certificate
openssl req -newkey rsa:4096 \
  -keyout sp-terraform.key \
  -x509 -days 365 \
  -out sp-terraform.crt \
  -subj "/CN=sp-terraform-prod"

# รวม certificate และ key เป็น PEM file
cat sp-terraform.key sp-terraform.crt > sp-terraform.pem

# แปลงเป็น PKCS12 format (สำหรับ Windows)
openssl pkcs12 -export \
  -in sp-terraform.crt \
  -inkey sp-terraform.key \
  -out sp-terraform.p12 \
  -passout pass:

# สร้าง SP พร้อม certificate
az ad sp create-for-rbac \
  --name "sp-terraform-cert" \
  --role Contributor \
  --scopes "/subscriptions/YOUR-SUBSCRIPTION-ID" \
  --cert @sp-terraform.crt \
  --output json

# หรือ upload certificate ให้ SP ที่มีอยู่
az ad sp credential reset \
  --id CLIENT_ID \
  --cert @sp-terraform.crt \
  --output json
```

```hcl
# auth-certificate.tf
provider "azurerm" {
  features {}
  
  client_id                   = var.arm_client_id
  client_certificate_path     = "/path/to/sp-terraform.pfx"
  client_certificate_password = var.certificate_password  # optional
  tenant_id                   = var.arm_tenant_id
  subscription_id             = var.arm_subscription_id
}

# หรือใช้ environment variables
# ARM_CLIENT_CERTIFICATE_PATH=/path/to/certificate.pfx
# ARM_CLIENT_CERTIFICATE_PASSWORD=certificate-password
```

---

## ขั้นตอนที่ 506: Managed Identity

Managed Identity ให้ Azure VM/Container จัดการ credentials โดยอัตโนมัติ

### System-Assigned Managed Identity

```hcl
# managed-identity-system.tf

# VM with system-assigned managed identity
resource "azurerm_linux_virtual_machine" "terraform_runner" {
  name                = "vm-terraform-runner"
  resource_group_name = azurerm_resource_group.main.name
  location            = azurerm_resource_group.main.location
  size                = "Standard_B2s"
  
  # Enable system-assigned managed identity
  identity {
    type = "SystemAssigned"
  }
  
  # ... other VM configuration
}

# Assign Contributor role to the VM's managed identity
resource "azurerm_role_assignment" "terraform_runner_contributor" {
  scope                = "/subscriptions/${var.subscription_id}"
  role_definition_name = "Contributor"
  principal_id         = azurerm_linux_virtual_machine.terraform_runner.identity[0].principal_id
}

# Provider using system-assigned managed identity
provider "azurerm" {
  features {}
  use_msi         = true
  subscription_id = var.subscription_id
  tenant_id       = var.tenant_id
}
```

### User-Assigned Managed Identity

```hcl
# managed-identity-user.tf

# สร้าง User-Assigned Managed Identity
resource "azurerm_user_assigned_identity" "terraform" {
  name                = "mi-terraform-prod"
  resource_group_name = azurerm_resource_group.identity.name
  location            = azurerm_resource_group.identity.location
  
  tags = {
    Purpose = "Terraform automation"
    ManagedBy = "Terraform"
  }
}

# Assign permissions
resource "azurerm_role_assignment" "terraform_contributor" {
  scope                = "/subscriptions/${var.subscription_id}"
  role_definition_name = "Contributor"
  principal_id         = azurerm_user_assigned_identity.terraform.principal_id
}

# VM with user-assigned managed identity
resource "azurerm_linux_virtual_machine" "runner" {
  name                = "vm-runner"
  resource_group_name = azurerm_resource_group.main.name
  location            = azurerm_resource_group.main.location
  size                = "Standard_B2s"
  
  identity {
    type         = "UserAssigned"
    identity_ids = [azurerm_user_assigned_identity.terraform.id]
  }
  
  # ... other configuration
}

# Provider using user-assigned managed identity
provider "azurerm" {
  features {}
  use_msi                    = true
  client_id                  = azurerm_user_assigned_identity.terraform.client_id
  subscription_id            = var.subscription_id
}

# Environment variable method
# ARM_USE_MSI=true
# ARM_CLIENT_ID=user-assigned-identity-client-id (optional)
# ARM_SUBSCRIPTION_ID=subscription-id
```

---

## ขั้นตอนที่ 507: OIDC Authentication

OIDC (OpenID Connect) ให้ CI/CD systems ยืนยันตัวตนกับ Azure โดยไม่ต้องเก็บ credentials

### GitHub Actions + OIDC

```bash
# สร้าง App Registration สำหรับ GitHub Actions OIDC
APP_ID=$(az ad app create --display-name "github-actions-terraform" --query appId -o tsv)

# สร้าง Service Principal
az ad sp create --id $APP_ID

# กำหนด Federated Credentials สำหรับ GitHub Actions
az ad app federated-credential create \
  --id $APP_ID \
  --parameters '{
    "name": "github-actions-prod",
    "issuer": "https://token.actions.githubusercontent.com",
    "subject": "repo:YOUR-ORG/YOUR-REPO:environment:production",
    "description": "GitHub Actions OIDC for production",
    "audiences": ["api://AzureADTokenExchange"]
  }'

# Grant Contributor role
az role assignment create \
  --role Contributor \
  --assignee $APP_ID \
  --scope "/subscriptions/YOUR-SUBSCRIPTION-ID"

echo "APP_ID (ARM_CLIENT_ID): $APP_ID"
echo "TENANT_ID: $(az account show --query tenantId -o tsv)"
echo "SUBSCRIPTION_ID: $(az account show --query id -o tsv)"
```

```yaml
# .github/workflows/terraform.yml
name: Terraform Deploy

on:
  push:
    branches: [main]
  pull_request:

permissions:
  id-token: write  # Required for OIDC
  contents: read

jobs:
  terraform:
    name: Terraform
    runs-on: ubuntu-latest
    environment: production
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Azure Login with OIDC
        uses: azure/login@v1
        with:
          client-id: ${{ secrets.AZURE_CLIENT_ID }}
          tenant-id: ${{ secrets.AZURE_TENANT_ID }}
          subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}
      
      - name: Setup Terraform
        uses: hashicorp/setup-terraform@v3
      
      - name: Terraform Init
        env:
          ARM_CLIENT_ID: ${{ secrets.AZURE_CLIENT_ID }}
          ARM_TENANT_ID: ${{ secrets.AZURE_TENANT_ID }}
          ARM_SUBSCRIPTION_ID: ${{ secrets.AZURE_SUBSCRIPTION_ID }}
          ARM_USE_OIDC: "true"
        run: terraform init
      
      - name: Terraform Plan
        env:
          ARM_CLIENT_ID: ${{ secrets.AZURE_CLIENT_ID }}
          ARM_TENANT_ID: ${{ secrets.AZURE_TENANT_ID }}
          ARM_SUBSCRIPTION_ID: ${{ secrets.AZURE_SUBSCRIPTION_ID }}
          ARM_USE_OIDC: "true"
        run: terraform plan
      
      - name: Terraform Apply
        if: github.ref == 'refs/heads/main'
        env:
          ARM_CLIENT_ID: ${{ secrets.AZURE_CLIENT_ID }}
          ARM_TENANT_ID: ${{ secrets.AZURE_TENANT_ID }}
          ARM_SUBSCRIPTION_ID: ${{ secrets.AZURE_SUBSCRIPTION_ID }}
          ARM_USE_OIDC: "true"
        run: terraform apply -auto-approve
```

```hcl
# auth-oidc.tf - Provider configuration for OIDC
provider "azurerm" {
  features {}
  
  use_oidc        = true
  client_id       = var.arm_client_id
  tenant_id       = var.arm_tenant_id
  subscription_id = var.arm_subscription_id
  
  # Optional: specify OIDC token file path
  # oidc_token_file_path = "/path/to/token"
  # Or use environment variable: ARM_OIDC_TOKEN_FILE_PATH
}
```

### GitLab CI + OIDC

```bash
# GitLab CI OIDC federated credential
az ad app federated-credential create \
  --id $APP_ID \
  --parameters '{
    "name": "gitlab-ci-prod",
    "issuer": "https://gitlab.com",
    "subject": "project_path:your-group/your-project:ref_type:branch:ref:main",
    "description": "GitLab CI OIDC",
    "audiences": ["https://gitlab.com"]
  }'
```

```yaml
# .gitlab-ci.yml
variables:
  ARM_USE_OIDC: "true"
  ARM_CLIENT_ID: $AZURE_CLIENT_ID
  ARM_TENANT_ID: $AZURE_TENANT_ID
  ARM_SUBSCRIPTION_ID: $AZURE_SUBSCRIPTION_ID

terraform:
  id_tokens:
    GITLAB_OIDC_TOKEN:
      aud: https://gitlab.com
  before_script:
    - export ARM_OIDC_TOKEN=$GITLAB_OIDC_TOKEN
  script:
    - terraform init
    - terraform plan
    - terraform apply -auto-approve
```

### Terraform Cloud + OIDC

```hcl
# auth-terraform-cloud-oidc.tf
provider "azurerm" {
  features {}
  
  use_oidc        = true
  client_id       = var.arm_client_id
  tenant_id       = var.arm_tenant_id
  subscription_id = var.arm_subscription_id
}

# ใน Terraform Cloud: กำหนด Workspace Variables
# ARM_CLIENT_ID = client_id (sensitive)
# ARM_TENANT_ID = tenant_id
# ARM_SUBSCRIPTION_ID = subscription_id
# ARM_USE_OIDC = true
```

---

## ขั้นตอนที่ 508: Environment Variables ทั้งหมด

```bash
# ============================================================
# ARM Environment Variables Reference
# ============================================================

# === Authentication ===
export ARM_CLIENT_ID="service-principal-app-id"
export ARM_CLIENT_SECRET="service-principal-password"
export ARM_CLIENT_CERTIFICATE_PATH="/path/to/cert.pfx"
export ARM_CLIENT_CERTIFICATE_PASSWORD="cert-password"

# === Identity ===
export ARM_TENANT_ID="azure-tenant-id"
export ARM_SUBSCRIPTION_ID="azure-subscription-id"

# === Managed Identity ===
export ARM_USE_MSI=true
export ARM_MSI_ENDPOINT="http://169.254.169.254/metadata/identity/oauth2/token"

# === OIDC ===
export ARM_USE_OIDC=true
export ARM_OIDC_TOKEN="eyJ..."            # Direct token
export ARM_OIDC_TOKEN_FILE_PATH="/var/run/secrets/azure/token"  # Token from file
export ARM_OIDC_REQUEST_URL="$ACTIONS_ID_TOKEN_REQUEST_URL"     # GitHub Actions
export ARM_OIDC_REQUEST_TOKEN="$ACTIONS_ID_TOKEN_REQUEST_TOKEN" # GitHub Actions

# === Azure Environment ===
export ARM_ENVIRONMENT="public"           # public, usgovernment, china, german
export ARM_METADATA_URL="https://management.azure.com/"

# === Provider Behavior ===
export ARM_SKIP_PROVIDER_REGISTRATION=false
export ARM_DISABLE_TERRAFORM_PARTNER_ID=false

# === Partner ID ===
export ARM_PARTNER_ID="partner-uuid"

# ============================================================
# ตัวอย่าง .env file สำหรับ local development
# ============================================================
cat > .env.local << 'EOF'
# Azure Authentication - LOCAL DEVELOPMENT ONLY
# DO NOT COMMIT THIS FILE
ARM_CLIENT_ID=00000000-0000-0000-0000-000000000000
ARM_CLIENT_SECRET=very-secret-value-here
ARM_TENANT_ID=00000000-0000-0000-0000-000000000001
ARM_SUBSCRIPTION_ID=00000000-0000-0000-0000-000000000002
EOF

# Script สำหรับโหลด credentials
cat > scripts/load-azure-env.sh << 'EOF'
#!/bin/bash
# Load Azure credentials from Azure Key Vault
VAULT_NAME="kv-terraform-dev"

export ARM_CLIENT_ID=$(az keyvault secret show --vault-name $VAULT_NAME --name sp-client-id --query value -o tsv)
export ARM_CLIENT_SECRET=$(az keyvault secret show --vault-name $VAULT_NAME --name sp-client-secret --query value -o tsv)
export ARM_TENANT_ID=$(az account show --query tenantId -o tsv)
export ARM_SUBSCRIPTION_ID=$(az account show --query id -o tsv)

echo "Azure credentials loaded from Key Vault"
EOF
chmod +x scripts/load-azure-env.sh
```

---

## ขั้นตอนที่ 509: Azure Provider Features Block

`features` block ควบคุม behavior ของ provider เมื่อ create/destroy resources

```hcl
# features-complete.tf
provider "azurerm" {
  features {
    # ============================================================
    # Key Vault
    # ============================================================
    key_vault {
      # ลบ Key Vault ถาวรเมื่อ destroy (ไม่รอ soft-delete expiry)
      purge_soft_delete_on_destroy = true
      
      # กู้คืน soft-deleted Key Vault อัตโนมัติเมื่อ create ใหม่
      recover_soft_deleted_key_vaults = true
      
      # กู้คืน soft-deleted secrets/keys/certs
      recover_soft_deleted_secrets          = true
      recover_soft_deleted_key_vault_keys   = true
      recover_soft_deleted_certificates     = true
    }
    
    # ============================================================
    # Virtual Machine
    # ============================================================
    virtual_machine {
      # ลบ OS disk เมื่อ VM ถูกลบ
      delete_os_disk_on_deletion = true
      
      # Graceful shutdown ก่อน delete
      graceful_shutdown = false
      
      # Force delete โดยไม่ shutdown (ระวัง! อาจทำให้ data loss)
      skip_shutdown_and_force_delete = false
    }
    
    # ============================================================
    # Virtual Machine Scale Set
    # ============================================================
    virtual_machine_scale_set {
      # Force delete VMSS
      force_delete                  = false
      roll_instances_when_required  = true
      scale_to_zero_before_deletion = true
    }
    
    # ============================================================
    # Resource Group
    # ============================================================
    resource_group {
      # ป้องกันการลบ Resource Group ที่ยังมี resources อยู่
      prevent_deletion_if_contains_resources = true
    }
    
    # ============================================================
    # API Management
    # ============================================================
    api_management {
      purge_soft_delete_on_destroy = true
      recover_soft_deleted         = true
    }
    
    # ============================================================
    # App Configuration
    # ============================================================
    app_configuration {
      purge_soft_delete_on_destroy = true
      recover_soft_deleted         = true
    }
    
    # ============================================================
    # Application Insights
    # ============================================================
    application_insights {
      # ปิดการสร้าง Smart Detection Rules อัตโนมัติ
      disable_generated_rule = false
    }
    
    # ============================================================
    # Cognitive Account (Azure AI Services)
    # ============================================================
    cognitive_account {
      purge_soft_delete_on_destroy = true
    }
    
    # ============================================================
    # Log Analytics Workspace
    # ============================================================
    log_analytics_workspace {
      permanently_delete_on_destroy = true
    }
    
    # ============================================================
    # Template Deployment
    # ============================================================
    template_deployment {
      delete_nested_items_during_deletion = true
    }
    
    # ============================================================
    # Managed Disk
    # ============================================================
    managed_disk {
      expand_without_downtime = true
    }
    
    # ============================================================
    # Recovery Services
    # ============================================================
    recovery_services {
      vm_backup_stop_protection_and_retain_data_on_destroy = false
      purge_protected_items_from_vault_on_destroy          = false
    }
    
    # ============================================================
    # Storage
    # ============================================================
    storage {
      data_plane_available = true
    }
  }
}
```

---

## ขั้นตอนที่ 510: Multiple Subscriptions & Special Configurations

### Multiple Subscriptions

```hcl
# multi-subscription.tf

# Production subscription
provider "azurerm" {
  alias           = "production"
  subscription_id = var.prod_subscription_id
  client_id       = var.prod_client_id
  client_secret   = var.prod_client_secret
  tenant_id       = var.tenant_id
  features {}
}

# Development subscription
provider "azurerm" {
  alias           = "development"
  subscription_id = var.dev_subscription_id
  client_id       = var.dev_client_id
  client_secret   = var.dev_client_secret
  tenant_id       = var.tenant_id
  features {}
}

# Shared services subscription
provider "azurerm" {
  alias           = "shared"
  subscription_id = var.shared_subscription_id
  client_id       = var.shared_client_id
  client_secret   = var.shared_client_secret
  tenant_id       = var.tenant_id
  features {}
}

# ใช้ provider aliases ใน resources
resource "azurerm_resource_group" "prod_rg" {
  provider = azurerm.production
  name     = "rg-myapp-prod"
  location = "Southeast Asia"
}

resource "azurerm_resource_group" "dev_rg" {
  provider = azurerm.development
  name     = "rg-myapp-dev"
  location = "Southeast Asia"
}

# Data sources with specific provider
data "azurerm_virtual_network" "shared_vnet" {
  provider            = azurerm.shared
  name                = "vnet-hub-shared"
  resource_group_name = "rg-network-shared"
}
```

### Azure Government Cloud

```hcl
# azure-government.tf
provider "azurerm" {
  features {}
  
  # กำหนด environment เป็น US Government
  environment     = "usgovernment"
  subscription_id = var.gov_subscription_id
  client_id       = var.gov_client_id
  client_secret   = var.gov_client_secret
  tenant_id       = var.gov_tenant_id
  
  # หรือใช้ environment variable:
  # ARM_ENVIRONMENT=usgovernment
}

# Azure China
provider "azurerm" {
  alias       = "china"
  environment = "china"
  features {}
  
  # ARM_ENVIRONMENT=china
}
```

### Provider สำหรับ Local Development vs CI/CD

```hcl
# locals-based auth switching
locals {
  # ตรวจสอบว่า running ใน CI/CD หรือ local
  is_ci = var.is_ci_environment
}

# variables.tf
variable "is_ci_environment" {
  description = "True when running in CI/CD pipeline"
  type        = bool
  default     = false
}

variable "arm_client_id" {
  description = "Client ID (used in CI/CD mode)"
  type        = string
  default     = ""
  sensitive   = true
}

variable "arm_client_secret" {
  description = "Client Secret (used in CI/CD mode)"
  type        = string
  default     = ""
  sensitive   = true
}
```

```bash
# dev.tfvars - Local development
is_ci_environment = false

# ci.tfvars - CI/CD pipeline
is_ci_environment = true
arm_client_id     = "CI-CLIENT-ID"
arm_client_secret = "CI-CLIENT-SECRET"
```

### Service Principal Creation Script

```bash
#!/bin/bash
# scripts/create-terraform-sp.sh

set -euo pipefail

SUBSCRIPTION_ID=$(az account show --query id -o tsv)
TENANT_ID=$(az account show --query tenantId -o tsv)
SP_NAME="sp-terraform-$(date +%Y%m)"

echo "Creating Service Principal: $SP_NAME"
echo "Subscription: $SUBSCRIPTION_ID"
echo "Tenant: $TENANT_ID"

# สร้าง SP
SP_OUTPUT=$(az ad sp create-for-rbac \
  --name "$SP_NAME" \
  --role "Contributor" \
  --scopes "/subscriptions/$SUBSCRIPTION_ID" \
  --output json)

CLIENT_ID=$(echo $SP_OUTPUT | jq -r '.appId')
CLIENT_SECRET=$(echo $SP_OUTPUT | jq -r '.password')

echo ""
echo "=== Service Principal Created ==="
echo "SP Name: $SP_NAME"
echo "Client ID: $CLIENT_ID"
echo "Client Secret: [HIDDEN - stored in vault]"
echo ""

# เพิ่ม Storage Blob Data Contributor สำหรับ Terraform state
az role assignment create \
  --role "Storage Blob Data Contributor" \
  --assignee "$CLIENT_ID" \
  --scope "/subscriptions/$SUBSCRIPTION_ID/resourceGroups/rg-terraform-state/providers/Microsoft.Storage/storageAccounts/stterraformstate001" \
  2>/dev/null && echo "Storage role assigned" || echo "Storage role assignment skipped"

# บันทึก credentials ไปที่ Key Vault
KV_NAME="kv-terraform-creds"
az keyvault secret set --vault-name $KV_NAME --name "sp-client-id" --value "$CLIENT_ID" -o none
az keyvault secret set --vault-name $KV_NAME --name "sp-client-secret" --value "$CLIENT_SECRET" -o none
az keyvault secret set --vault-name $KV_NAME --name "tenant-id" --value "$TENANT_ID" -o none
az keyvault secret set --vault-name $KV_NAME --name "subscription-id" --value "$SUBSCRIPTION_ID" -o none

echo "Credentials stored in Key Vault: $KV_NAME"
echo ""
echo "=== GitHub Secrets to Configure ==="
echo "AZURE_CLIENT_ID: $CLIENT_ID"
echo "AZURE_TENANT_ID: $TENANT_ID"
echo "AZURE_SUBSCRIPTION_ID: $SUBSCRIPTION_ID"
echo "AZURE_CLIENT_SECRET: [retrieve from Key Vault]"
```

### Azure Naming Conventions

```hcl
# naming-conventions.tf
# Azure แนะนำ naming convention ตาม Cloud Adoption Framework

locals {
  # Common prefix
  env    = "prod"    # dev, test, staging, prod
  region = "sea"     # Southeast Asia
  app    = "myapp"
  team   = "platform"
  
  # Resource naming patterns
  resource_group_name    = "rg-${local.app}-${local.env}"
  virtual_network_name   = "vnet-${local.app}-${local.env}-${local.region}"
  subnet_name            = "snet-${local.app}-${local.env}"
  nsg_name               = "nsg-${local.app}-${local.env}"
  vm_name                = "vm-${local.app}-${local.env}-001"
  storage_account_name   = "st${local.app}${local.env}${local.region}001"  # no hyphens
  key_vault_name         = "kv-${local.app}-${local.env}"
  aks_cluster_name       = "aks-${local.app}-${local.env}"
  container_registry_name = "cr${local.app}${local.env}001"  # no hyphens
  
  # Common tags
  common_tags = {
    Environment = local.env
    Application = local.app
    Team        = local.team
    ManagedBy   = "Terraform"
    CostCenter  = "CC-001"
    Owner       = "platform-team@company.com"
    CreatedDate = formatdate("YYYY-MM-DD", timestamp())
  }
}

# ตัวอย่างการใช้งาน naming
resource "azurerm_resource_group" "main" {
  name     = local.resource_group_name
  location = "Southeast Asia"
  tags     = local.common_tags
}

resource "azurerm_virtual_network" "main" {
  name                = local.virtual_network_name
  resource_group_name = azurerm_resource_group.main.name
  location            = azurerm_resource_group.main.location
  address_space       = ["10.0.0.0/16"]
  tags                = local.common_tags
}
```

### Resource Tagging Strategy

```hcl
# tagging-strategy.tf

# Tag variables สำหรับทุก environment
variable "tags" {
  description = "Resource tags"
  type        = map(string)
  default     = {}
}

locals {
  # Mandatory tags
  mandatory_tags = {
    Environment    = var.environment
    ManagedBy      = "Terraform"
    TerraformRepo  = "github.com/company/infrastructure"
    CostCenter     = var.cost_center
    Owner          = var.owner_email
    BusinessUnit   = var.business_unit
  }
  
  # Merge mandatory with custom tags
  all_tags = merge(local.mandatory_tags, var.tags)
}

# Output tags สำหรับ modules
output "common_tags" {
  value = local.all_tags
}

# ตัวอย่าง terraform.tfvars
/*
environment   = "production"
cost_center   = "CC-Engineering-001"
owner_email   = "platform@company.com"
business_unit = "Engineering"

tags = {
  Application = "MyApp"
  Criticality = "High"
  DataClass   = "Confidential"
}
*/

# Azure Policy สำหรับ enforce tags
resource "azurerm_policy_definition" "require_tags" {
  name         = "require-resource-tags"
  policy_type  = "Custom"
  mode         = "Indexed"
  display_name = "Require mandatory tags on resources"
  
  policy_rule = jsonencode({
    if = {
      anyOf = [
        {
          field  = "tags['Environment']"
          exists = "false"
        },
        {
          field  = "tags['Owner']"
          exists = "false"
        },
        {
          field  = "tags['CostCenter']"
          exists = "false"
        }
      ]
    }
    then = {
      effect = "deny"
    }
  })
}
```

---

## สรุป: Azure Provider Authentication Comparison

| Method | Use Case | Security Level | Complexity |
|--------|----------|----------------|------------|
| Azure CLI | Local Development | Medium | Low |
| Client Secret | CI/CD (simple) | Medium | Low |
| Certificate | CI/CD (enhanced) | High | Medium |
| System Managed Identity | Azure-hosted runners | Very High | Low |
| User Managed Identity | Azure-hosted (shared) | Very High | Medium |
| OIDC (GitHub/GitLab) | Modern CI/CD | Very High | Medium |
| Terraform Cloud OIDC | TFC Workspaces | Very High | Medium |

### Best Practices สรุป

```hcl
# best-practices.tf

# 1. ไม่ hardcode credentials ใน code
# BAD:
# provider "azurerm" {
#   client_id     = "hardcoded-id"
#   client_secret = "hardcoded-secret"  # อันตราย!
# }

# GOOD: ใช้ environment variables หรือ variables
provider "azurerm" {
  features {}
  # อ่านจาก ARM_* environment variables อัตโนมัติ
}

# 2. ใช้ Managed Identity สำหรับ Azure-hosted workloads
# 3. ใช้ OIDC สำหรับ GitHub Actions / GitLab CI
# 4. Rotate credentials สม่ำเสมอ
# 5. ใช้ least-privilege principle
# 6. เก็บ secrets ใน Azure Key Vault
# 7. Enable audit logging สำหรับ Service Principals
# 8. ใช้ Conditional Access policies
```

---

*จบ Part 051: Azure Provider Setup & Authentication*
*ต่อไป Part 052: Azure Resource Groups & Management*
