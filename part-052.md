# Part 052: Azure Resource Groups & Management
## ขั้นตอนที่ 511-520: การจัดการ Resource Groups และโครงสร้าง Azure

---

## ขั้นตอนที่ 511: Azure Resource Groups พื้นฐาน

Resource Group คือ container สำหรับ Azure resources ทุกอย่างใน Azure ต้องอยู่ใน Resource Group เสมอ

```hcl
# resource-group-basic.tf

# Resource Group พื้นฐาน
resource "azurerm_resource_group" "main" {
  name     = "rg-myapp-prod-sea"
  location = "Southeast Asia"
  
  tags = {
    Environment = "Production"
    Application = "MyApp"
    ManagedBy   = "Terraform"
    Owner       = "platform-team@company.com"
  }
}

# Resource Group สำหรับ networking
resource "azurerm_resource_group" "networking" {
  name     = "rg-networking-prod-sea"
  location = "Southeast Asia"
  
  tags = {
    Environment = "Production"
    Purpose     = "Networking"
    ManagedBy   = "Terraform"
  }
}

# Resource Group สำหรับ security
resource "azurerm_resource_group" "security" {
  name     = "rg-security-prod-sea"
  location = "Southeast Asia"
  
  tags = {
    Environment = "Production"
    Purpose     = "Security"
    ManagedBy   = "Terraform"
  }
}

# Resource Group สำหรับ monitoring
resource "azurerm_resource_group" "monitoring" {
  name     = "rg-monitoring-prod-sea"
  location = "Southeast Asia"
  
  tags = {
    Environment = "Production"
    Purpose     = "Monitoring"
    ManagedBy   = "Terraform"
  }
}
```

### Azure Locations (Regions)

```hcl
# azure-regions.tf
# รายชื่อ Azure Regions ที่ใช้บ่อย

locals {
  azure_regions = {
    # Asia Pacific
    southeast_asia  = "Southeast Asia"      # Singapore
    east_asia       = "East Asia"           # Hong Kong
    australia_east  = "Australia East"      # New South Wales
    japan_east      = "Japan East"          # Tokyo
    korea_central   = "Korea Central"       # Seoul
    
    # Europe
    west_europe     = "West Europe"         # Netherlands
    north_europe    = "North Europe"        # Ireland
    uk_south        = "UK South"            # London
    
    # Americas
    east_us         = "East US"             # Virginia
    east_us_2       = "East US 2"           # Virginia
    west_us_2       = "West US 2"           # Washington
    central_us      = "Central US"          # Iowa
    
    # Middle East & Africa
    uae_north       = "UAE North"           # Dubai
    south_africa_n  = "South Africa North"  # Johannesburg
  }
  
  primary_location   = "Southeast Asia"
  secondary_location = "East Asia"
}

# ตรวจสอบ location ที่ valid ด้วย data source
data "azurerm_location" "primary" {
  location = local.primary_location
}

output "primary_location_display_name" {
  value = data.azurerm_location.primary.display_name
}
```

---

## ขั้นตอนที่ 512: Azure Naming Conventions

```hcl
# naming-conventions.tf

variable "env" {
  description = "Environment name"
  type        = string
  validation {
    condition     = contains(["dev", "test", "staging", "prod"], var.env)
    error_message = "Environment must be dev, test, staging, or prod."
  }
}

variable "app_name" {
  description = "Application name (short, lowercase)"
  type        = string
  validation {
    condition     = can(regex("^[a-z0-9]{2,10}$", var.app_name))
    error_message = "App name must be 2-10 lowercase alphanumeric characters."
  }
}

variable "region_code" {
  description = "Short region code"
  type        = string
  default     = "sea"
  
  validation {
    condition     = contains(["sea", "eas", "aue", "weu", "eus", "wus"], var.region_code)
    error_message = "Invalid region code."
  }
}

locals {
  # Naming prefix
  prefix = "${var.app_name}-${var.env}-${var.region_code}"
  
  # Resource names following CAF (Cloud Adoption Framework)
  names = {
    # General
    resource_group     = "rg-${local.prefix}"
    management_group   = "mg-${var.app_name}"
    
    # Networking
    virtual_network    = "vnet-${local.prefix}"
    subnet_app         = "snet-app-${local.prefix}"
    subnet_data        = "snet-data-${local.prefix}"
    subnet_mgmt        = "snet-mgmt-${local.prefix}"
    nsg                = "nsg-${local.prefix}"
    route_table        = "rt-${local.prefix}"
    public_ip          = "pip-${local.prefix}"
    nat_gateway        = "ng-${local.prefix}"
    
    # Compute
    vm_linux           = "vm-linux-${local.prefix}-001"
    vm_windows         = "vm-win-${local.prefix}-001"
    vmss               = "vmss-${local.prefix}"
    aks_cluster        = "aks-${local.prefix}"
    
    # Storage (no hyphens, max 24 chars)
    storage_account    = "st${var.app_name}${var.env}${var.region_code}001"
    
    # Database
    sql_server         = "sql-${local.prefix}"
    postgres_server    = "psql-${local.prefix}"
    cosmos_account     = "cosmos-${local.prefix}"
    redis_cache        = "redis-${local.prefix}"
    
    # Security
    key_vault          = "kv-${local.prefix}"
    
    # Monitoring
    log_analytics      = "law-${local.prefix}"
    app_insights       = "appi-${local.prefix}"
    
    # Containers
    container_registry = "cr${var.app_name}${var.env}001"  # no hyphens
    
    # Identity
    managed_identity   = "mi-${local.prefix}"
    
    # App Services
    app_service_plan   = "asp-${local.prefix}"
    app_service        = "app-${local.prefix}"
    function_app       = "func-${local.prefix}"
  }
}
```

### Azure CAF Naming Module

```hcl
# ใช้ Azure CAF naming module (community)
module "naming" {
  source  = "Azure/naming/azurerm"
  version = "~> 0.3"
  
  suffix = [var.env, var.region_code]
  prefix = [var.app_name]
}

resource "azurerm_resource_group" "main" {
  name     = module.naming.resource_group.name
  location = "Southeast Asia"
}

resource "azurerm_virtual_network" "main" {
  name                = module.naming.virtual_network.name
  resource_group_name = azurerm_resource_group.main.name
  location            = azurerm_resource_group.main.location
  address_space       = ["10.0.0.0/16"]
}
```

---

## ขั้นตอนที่ 513: Resource Groups as Scope

```hcl
# resource-group-scope.tf

# Resource Group เป็น boundary สำหรับ RBAC, Policy, และ Cost
resource "azurerm_resource_group" "app_prod" {
  name     = "rg-myapp-prod"
  location = "Southeast Asia"
  tags = {
    Environment = "Production"
    CostCenter  = "CC-100"
  }
}

# Assign RBAC at Resource Group level
data "azuread_group" "devs" {
  display_name = "DevTeam-MyApp"
}

resource "azurerm_role_assignment" "dev_reader" {
  scope                = azurerm_resource_group.app_prod.id
  role_definition_name = "Reader"
  principal_id         = data.azuread_group.devs.object_id
}

# Assign Contributor to specific service account
resource "azurerm_role_assignment" "cicd_contributor" {
  scope                = azurerm_resource_group.app_prod.id
  role_definition_name = "Contributor"
  principal_id         = var.cicd_service_principal_object_id
}

# Custom Role Assignment
resource "azurerm_role_definition" "app_operator" {
  name        = "App Operator"
  scope       = azurerm_resource_group.app_prod.id
  description = "Custom role for application operators"
  
  permissions {
    actions = [
      "Microsoft.Compute/virtualMachines/start/action",
      "Microsoft.Compute/virtualMachines/restart/action",
      "Microsoft.Compute/virtualMachines/read",
      "Microsoft.Network/*/read",
      "Microsoft.Resources/*/read",
    ]
    not_actions = []
  }
  
  assignable_scopes = [azurerm_resource_group.app_prod.id]
}
```

---

## ขั้นตอนที่ 514: Management Groups

Management Groups ช่วยจัดการ subscriptions หลายๆ ตัวพร้อมกัน

```hcl
# management-groups.tf

# Root management group (สร้างอัตโนมัติ แต่สามารถ import ได้)
data "azurerm_management_group" "root" {
  name = "your-root-management-group-id"
}

# Management Group hierarchy
resource "azurerm_management_group" "company" {
  name         = "mg-company"
  display_name = "Company Root"
}

resource "azurerm_management_group" "production" {
  name              = "mg-production"
  display_name      = "Production"
  parent_management_group_id = azurerm_management_group.company.id
}

resource "azurerm_management_group" "development" {
  name              = "mg-development"
  display_name      = "Development"
  parent_management_group_id = azurerm_management_group.company.id
}

resource "azurerm_management_group" "sandbox" {
  name              = "mg-sandbox"
  display_name      = "Sandbox"
  parent_management_group_id = azurerm_management_group.company.id
}

# Assign subscriptions to management groups
resource "azurerm_management_group_subscription_association" "prod_sub" {
  management_group_id = azurerm_management_group.production.id
  subscription_id     = "/subscriptions/${var.prod_subscription_id}"
}

resource "azurerm_management_group_subscription_association" "dev_sub" {
  management_group_id = azurerm_management_group.development.id
  subscription_id     = "/subscriptions/${var.dev_subscription_id}"
}

# Policy at Management Group level
resource "azurerm_management_group_policy_assignment" "deny_public_ip" {
  name                 = "deny-public-ip"
  management_group_id  = azurerm_management_group.production.id
  policy_definition_id = azurerm_policy_definition.deny_public_ip.id
  display_name         = "Deny Public IP addresses in Production"
  description          = "Prevents creation of public IP addresses"
  
  enforce = true
}
```

---

## ขั้นตอนที่ 515: Azure Subscriptions

```hcl
# subscriptions.tf

# ดึงข้อมูล subscription ปัจจุบัน
data "azurerm_subscription" "current" {}

# ดึงข้อมูล subscription อื่น
data "azurerm_subscription" "production" {
  subscription_id = var.prod_subscription_id
}

output "current_subscription_info" {
  value = {
    id           = data.azurerm_subscription.current.subscription_id
    display_name = data.azurerm_subscription.current.display_name
    state        = data.azurerm_subscription.current.state
    tenant_id    = data.azurerm_subscription.current.tenant_id
  }
}

# List subscriptions (ใช้ azuread provider)
data "azurerm_subscriptions" "available" {
  display_name_prefix = "company-"  # Filter by prefix
}

output "available_subscriptions" {
  value = [
    for sub in data.azurerm_subscriptions.available.subscriptions : {
      id   = sub.subscription_id
      name = sub.display_name
    }
  ]
}

# Client config - current authenticated user/SP
data "azurerm_client_config" "current" {}

output "current_user_info" {
  value = {
    client_id       = data.azurerm_client_config.current.client_id
    tenant_id       = data.azurerm_client_config.current.tenant_id
    subscription_id = data.azurerm_client_config.current.subscription_id
    object_id       = data.azurerm_client_config.current.object_id
  }
}
```

---

## ขั้นตอนที่ 516: Resource Group Tagging Strategy

```hcl
# tagging-strategy.tf

# ============================================================
# Tag Variables
# ============================================================
variable "environment" {
  type = string
}

variable "application" {
  type = string
}

variable "cost_center" {
  type    = string
  default = "Unknown"
}

variable "owner" {
  type    = string
  default = "unknown@company.com"
}

variable "business_unit" {
  type    = string
  default = "Unknown"
}

variable "criticality" {
  type    = string
  default = "Low"
  validation {
    condition     = contains(["Critical", "High", "Medium", "Low"], var.criticality)
    error_message = "Criticality must be Critical, High, Medium, or Low."
  }
}

variable "data_classification" {
  type    = string
  default = "Internal"
  validation {
    condition     = contains(["Public", "Internal", "Confidential", "Restricted"], var.data_classification)
    error_message = "Data classification must be Public, Internal, Confidential, or Restricted."
  }
}

# ============================================================
# Common Tags Local
# ============================================================
locals {
  common_tags = {
    # Mandatory tags
    Environment        = var.environment
    Application        = var.application
    CostCenter         = var.cost_center
    Owner              = var.owner
    BusinessUnit       = var.business_unit
    ManagedBy          = "Terraform"
    TerraformWorkspace = terraform.workspace
    TerraformRepo      = "github.com/company/infrastructure"
    
    # Optional but recommended
    Criticality        = var.criticality
    DataClassification = var.data_classification
    LastUpdated        = formatdate("YYYY-MM-DD", timestamp())
  }
  
  # Security-specific tags
  security_tags = merge(local.common_tags, {
    Compliance  = "PCI-DSS"
    SensitiveData = "Yes"
    AuditRequired = "Yes"
  })
  
  # Cost allocation tags
  cost_tags = merge(local.common_tags, {
    BillingProject = var.application
    CostAllocation  = "Auto"
  })
}

# Resource Groups with proper tagging
resource "azurerm_resource_group" "application" {
  name     = "rg-${var.application}-${var.environment}"
  location = "Southeast Asia"
  tags     = local.common_tags
}

resource "azurerm_resource_group" "networking" {
  name     = "rg-network-${var.application}-${var.environment}"
  location = "Southeast Asia"
  tags = merge(local.common_tags, {
    Purpose = "Networking"
  })
}

resource "azurerm_resource_group" "security" {
  name     = "rg-security-${var.application}-${var.environment}"
  location = "Southeast Asia"
  tags = merge(local.security_tags, {
    Purpose = "Security"
  })
}
```

---

## ขั้นตอนที่ 517: Resource Group Locks

Locks ป้องกันการลบหรือแก้ไข resources โดยไม่ตั้งใจ

```hcl
# resource-locks.tf

# ============================================================
# Management Locks
# ============================================================

# CanNotDelete Lock - ป้องกันการลบ แต่อนุญาตให้แก้ไขได้
resource "azurerm_management_lock" "production_rg_lock" {
  name       = "lock-rg-prod-nodelete"
  scope      = azurerm_resource_group.production.id
  lock_level = "CanNotDelete"
  notes      = "Production resources - deletion requires approval"
}

# ReadOnly Lock - ป้องกันทั้งการแก้ไขและการลบ
resource "azurerm_management_lock" "critical_storage_lock" {
  name       = "lock-storage-readonly"
  scope      = azurerm_storage_account.critical.id
  lock_level = "ReadOnly"
  notes      = "Critical storage account - read-only access only"
}

# Lock ที่ Key Vault level
resource "azurerm_management_lock" "key_vault_lock" {
  name       = "lock-kv-nodelete"
  scope      = azurerm_key_vault.main.id
  lock_level = "CanNotDelete"
  notes      = "Key Vault contains critical secrets"
}

# Lock ที่ Virtual Network level
resource "azurerm_management_lock" "vnet_lock" {
  name       = "lock-vnet-production"
  scope      = azurerm_virtual_network.production.id
  lock_level = "CanNotDelete"
  notes      = "Production Virtual Network"
}

# ============================================================
# Conditional Locks (production only)
# ============================================================
variable "enable_resource_locks" {
  description = "Enable resource locks (recommended for production)"
  type        = bool
  default     = false
}

resource "azurerm_management_lock" "conditional_lock" {
  count = var.enable_resource_locks ? 1 : 0
  
  name       = "lock-rg-conditional"
  scope      = azurerm_resource_group.main.id
  lock_level = "CanNotDelete"
  notes      = "Lock enabled for environment: ${var.environment}"
}
```

---

## ขั้นตอนที่ 518: Azure Policy

```hcl
# azure-policy.tf

# ============================================================
# Policy Definitions
# ============================================================

# Policy: ต้องการ tags บน resources
resource "azurerm_policy_definition" "require_tags" {
  name         = "require-mandatory-tags"
  policy_type  = "Custom"
  mode         = "Indexed"
  display_name = "Require mandatory tags on resources"
  description  = "Ensures all resources have required tags"
  
  metadata = jsonencode({
    category = "Tags"
    version  = "1.0.0"
  })
  
  parameters = jsonencode({
    tagName = {
      type = "String"
      metadata = {
        displayName = "Tag Name"
        description = "The name of the required tag"
      }
    }
  })
  
  policy_rule = jsonencode({
    if = {
      allOf = [
        {
          field  = "type"
          notIn  = ["Microsoft.Resources/subscriptions", "Microsoft.Resources/resourceGroups"]
        },
        {
          field  = "[concat('tags[', parameters('tagName'), ']')]"
          exists = "false"
        }
      ]
    }
    then = {
      effect = "deny"
    }
  })
}

# Policy: ป้องกันการสร้าง Public IP
resource "azurerm_policy_definition" "deny_public_ip" {
  name         = "deny-public-ip-creation"
  policy_type  = "Custom"
  mode         = "All"
  display_name = "Deny Public IP address creation"
  description  = "Prevents creation of public IP addresses for security"
  
  metadata = jsonencode({
    category = "Network"
    version  = "1.0.0"
  })
  
  policy_rule = jsonencode({
    if = {
      field = "type"
      equals = "Microsoft.Network/publicIPAddresses"
    }
    then = {
      effect = "deny"
    }
  })
}

# Policy: กำหนด allowed locations
resource "azurerm_policy_definition" "allowed_locations" {
  name         = "allowed-azure-locations"
  policy_type  = "Custom"
  mode         = "Indexed"
  display_name = "Allowed Azure locations for resources"
  
  parameters = jsonencode({
    allowedLocations = {
      type = "Array"
      metadata = {
        displayName = "Allowed locations"
        description = "The list of allowed locations"
        strongType  = "location"
      }
    }
  })
  
  policy_rule = jsonencode({
    if = {
      allOf = [
        {
          field     = "location"
          notIn     = "[parameters('allowedLocations')]"
        },
        {
          field     = "location"
          notEquals = "global"
        },
        {
          field     = "type"
          notEquals = "Microsoft.Resources/resourceGroups"
        }
      ]
    }
    then = {
      effect = "deny"
    }
  })
}

# ============================================================
# Policy Assignments
# ============================================================

# Assign tag policy at subscription level
resource "azurerm_subscription_policy_assignment" "require_environment_tag" {
  name                 = "require-environment-tag"
  subscription_id      = data.azurerm_subscription.current.id
  policy_definition_id = azurerm_policy_definition.require_tags.id
  display_name         = "Require Environment tag"
  description          = "All resources must have an Environment tag"
  
  parameters = jsonencode({
    tagName = {
      value = "Environment"
    }
  })
}

# Assign location policy at Resource Group level
resource "azurerm_resource_group_policy_assignment" "allowed_locations_prod" {
  name                 = "allowed-locations-prod"
  resource_group_id    = azurerm_resource_group.production.id
  policy_definition_id = azurerm_policy_definition.allowed_locations.id
  display_name         = "Allowed locations for production"
  
  parameters = jsonencode({
    allowedLocations = {
      value = ["Southeast Asia", "East Asia"]
    }
  })
}

# Policy Initiative (Policy Set)
resource "azurerm_policy_set_definition" "security_baseline" {
  name         = "security-baseline"
  policy_type  = "Custom"
  display_name = "Security Baseline Initiative"
  description  = "Collection of security policies for baseline compliance"
  
  metadata = jsonencode({
    category = "Security"
    version  = "1.0.0"
  })
  
  policy_definition_reference {
    policy_definition_id = azurerm_policy_definition.deny_public_ip.id
    reference_id         = "deny-public-ip"
  }
  
  policy_definition_reference {
    policy_definition_id = azurerm_policy_definition.require_tags.id
    reference_id         = "require-environment-tag"
    
    parameter_values = jsonencode({
      tagName = {
        value = "Environment"
      }
    })
  }
  
  policy_definition_reference {
    policy_definition_id = azurerm_policy_definition.require_tags.id
    reference_id         = "require-owner-tag"
    
    parameter_values = jsonencode({
      tagName = {
        value = "Owner"
      }
    })
  }
}
```

---

## ขั้นตอนที่ 519: Resource Group Organization Patterns

### Pattern 1: Per Environment

```hcl
# pattern-per-environment.tf
# แบ่ง Resource Group ตาม environment

locals {
  environments = ["dev", "test", "staging", "prod"]
}

resource "azurerm_resource_group" "per_env" {
  for_each = toset(local.environments)
  
  name     = "rg-myapp-${each.key}"
  location = "Southeast Asia"
  
  tags = {
    Environment = each.key
    Application = "MyApp"
    ManagedBy   = "Terraform"
  }
}

output "environment_rg_ids" {
  value = {
    for env, rg in azurerm_resource_group.per_env :
    env => rg.id
  }
}
```

### Pattern 2: Per Application

```hcl
# pattern-per-application.tf
# แบ่ง Resource Group ตาม application/service

variable "applications" {
  type = map(object({
    team         = string
    cost_center  = string
    criticality  = string
  }))
  
  default = {
    "frontend" = {
      team        = "web-team"
      cost_center = "CC-001"
      criticality = "High"
    }
    "backend" = {
      team        = "api-team"
      cost_center = "CC-002"
      criticality = "Critical"
    }
    "data" = {
      team        = "data-team"
      cost_center = "CC-003"
      criticality = "High"
    }
  }
}

resource "azurerm_resource_group" "per_app" {
  for_each = var.applications
  
  name     = "rg-${each.key}-prod"
  location = "Southeast Asia"
  
  tags = {
    Application = each.key
    Team        = each.value.team
    CostCenter  = each.value.cost_center
    Criticality = each.value.criticality
    Environment = "prod"
    ManagedBy   = "Terraform"
  }
}
```

### Pattern 3: Per Team

```hcl
# pattern-per-team.tf
# แบ่ง Resource Group ตาม team

locals {
  team_environments = flatten([
    for team in ["platform", "devops", "security"] : [
      for env in ["dev", "prod"] : {
        key  = "${team}-${env}"
        team = team
        env  = env
      }
    ]
  ])
}

resource "azurerm_resource_group" "per_team" {
  for_each = {
    for item in local.team_environments :
    item.key => item
  }
  
  name     = "rg-${each.value.team}-${each.value.env}"
  location = "Southeast Asia"
  
  tags = {
    Team        = each.value.team
    Environment = each.value.env
    ManagedBy   = "Terraform"
  }
}
```

### Pattern 4: Layered Architecture

```hcl
# pattern-layered.tf
# แบ่งตาม layer (network, compute, data, security)

resource "azurerm_resource_group" "network_layer" {
  name     = "rg-network-${var.env}"
  location = "Southeast Asia"
  tags = merge(local.common_tags, { Layer = "Network" })
}

resource "azurerm_resource_group" "compute_layer" {
  name     = "rg-compute-${var.env}"
  location = "Southeast Asia"
  tags = merge(local.common_tags, { Layer = "Compute" })
}

resource "azurerm_resource_group" "data_layer" {
  name     = "rg-data-${var.env}"
  location = "Southeast Asia"
  tags = merge(local.common_tags, { Layer = "Data" })
}

resource "azurerm_resource_group" "security_layer" {
  name     = "rg-security-${var.env}"
  location = "Southeast Asia"
  tags = merge(local.common_tags, { Layer = "Security" })
}

resource "azurerm_resource_group" "monitoring_layer" {
  name     = "rg-monitoring-${var.env}"
  location = "Southeast Asia"
  tags = merge(local.common_tags, { Layer = "Monitoring" })
}
```

---

## ขั้นตอนที่ 520: Random Suffix และ Complete Examples

```hcl
# random-suffix.tf
# ใช้ random string สำหรับ globally unique names

resource "random_string" "suffix" {
  length  = 6
  special = false
  upper   = false
}

resource "random_id" "storage_suffix" {
  byte_length = 4
}

# Resource Group with random suffix
resource "azurerm_resource_group" "main" {
  name     = "rg-myapp-${var.env}-${random_string.suffix.result}"
  location = "Southeast Asia"
  
  tags = local.common_tags
}

# Storage account (globally unique, max 24 chars)
resource "azurerm_storage_account" "main" {
  name                     = "st${var.app_name}${var.env}${random_string.suffix.result}"
  resource_group_name      = azurerm_resource_group.main.name
  location                 = azurerm_resource_group.main.location
  account_tier             = "Standard"
  account_replication_type = "GRS"
  
  tags = local.common_tags
}

# Key Vault (globally unique, max 24 chars)
resource "azurerm_key_vault" "main" {
  name                = "kv-${var.app_name}-${random_string.suffix.result}"
  resource_group_name = azurerm_resource_group.main.name
  location            = azurerm_resource_group.main.location
  tenant_id           = data.azurerm_client_config.current.tenant_id
  sku_name            = "standard"
  
  tags = local.common_tags
}
```

### Complete Resource Group Module

```hcl
# modules/resource-group/main.tf

variable "name" {
  description = "Resource group name"
  type        = string
}

variable "location" {
  description = "Azure region"
  type        = string
  default     = "Southeast Asia"
}

variable "tags" {
  description = "Resource tags"
  type        = map(string)
  default     = {}
}

variable "lock_level" {
  description = "Management lock level (CanNotDelete, ReadOnly, or empty)"
  type        = string
  default     = ""
  validation {
    condition     = contains(["", "CanNotDelete", "ReadOnly"], var.lock_level)
    error_message = "lock_level must be empty, CanNotDelete, or ReadOnly."
  }
}

variable "policy_assignments" {
  description = "Policy assignments for the resource group"
  type = list(object({
    name                 = string
    policy_definition_id = string
    display_name         = string
    parameters           = optional(string, "{}")
  }))
  default = []
}

resource "azurerm_resource_group" "this" {
  name     = var.name
  location = var.location
  tags     = var.tags
}

resource "azurerm_management_lock" "this" {
  count = var.lock_level != "" ? 1 : 0
  
  name       = "lock-${var.name}"
  scope      = azurerm_resource_group.this.id
  lock_level = var.lock_level
  notes      = "Managed by Terraform"
}

resource "azurerm_resource_group_policy_assignment" "this" {
  for_each = {
    for policy in var.policy_assignments :
    policy.name => policy
  }
  
  name                 = each.value.name
  resource_group_id    = azurerm_resource_group.this.id
  policy_definition_id = each.value.policy_definition_id
  display_name         = each.value.display_name
  parameters           = each.value.parameters
}

output "id" {
  value = azurerm_resource_group.this.id
}

output "name" {
  value = azurerm_resource_group.this.name
}

output "location" {
  value = azurerm_resource_group.this.location
}
```

```hcl
# modules/resource-group/usage-example.tf

module "rg_production" {
  source = "./modules/resource-group"
  
  name     = "rg-myapp-prod-sea"
  location = "Southeast Asia"
  
  tags = {
    Environment = "Production"
    Application = "MyApp"
    ManagedBy   = "Terraform"
    Owner       = "platform@company.com"
    CostCenter  = "CC-100"
  }
  
  lock_level = "CanNotDelete"
  
  policy_assignments = [
    {
      name                 = "require-env-tag"
      policy_definition_id = azurerm_policy_definition.require_tags.id
      display_name         = "Require Environment tag"
      parameters = jsonencode({
        tagName = { value = "Environment" }
      })
    }
  ]
}

module "rg_dev" {
  source = "./modules/resource-group"
  
  name     = "rg-myapp-dev-sea"
  location = "Southeast Asia"
  
  tags = {
    Environment = "Development"
    Application = "MyApp"
    ManagedBy   = "Terraform"
  }
  
  # No lock for development
  lock_level = ""
}
```

### Cost Management Tags

```hcl
# cost-management.tf

# Budget alert สำหรับ resource group
resource "azurerm_consumption_budget_resource_group" "app_budget" {
  name              = "budget-myapp-prod"
  resource_group_id = azurerm_resource_group.application.id
  amount            = 5000  # $5,000 per month
  time_grain        = "Monthly"
  
  time_period {
    start_date = "2024-01-01T00:00:00Z"
    end_date   = "2025-12-31T00:00:00Z"
  }
  
  notification {
    enabled        = true
    threshold      = 80.0  # Alert at 80% of budget
    operator       = "GreaterThan"
    threshold_type = "Actual"
    
    contact_emails = [
      "finance@company.com",
      "platform@company.com"
    ]
    
    contact_roles = ["Owner", "Contributor"]
  }
  
  notification {
    enabled        = true
    threshold      = 100.0  # Alert at 100% (forecast)
    operator       = "GreaterThan"
    threshold_type = "Forecasted"
    
    contact_emails = [
      "management@company.com"
    ]
  }
  
  filter {
    tag {
      name = "Environment"
      values = ["Production"]
    }
  }
}
```

### Complete Main.tf Example

```hcl
# complete-example/main.tf

terraform {
  required_version = ">= 1.5.0"
  
  required_providers {
    azurerm = {
      source  = "hashicorp/azurerm"
      version = "~> 3.80"
    }
    random = {
      source  = "hashicorp/random"
      version = "~> 3.5"
    }
  }
  
  backend "azurerm" {
    resource_group_name  = "rg-terraform-state"
    storage_account_name = "stterraformstate001"
    container_name       = "tfstate"
    key                  = "prod/main.tfstate"
  }
}

provider "azurerm" {
  features {
    resource_group {
      prevent_deletion_if_contains_resources = true
    }
  }
}

data "azurerm_client_config" "current" {}
data "azurerm_subscription" "current" {}

locals {
  env    = "prod"
  app    = "myapp"
  region = "sea"
  
  common_tags = {
    Environment = local.env
    Application = local.app
    ManagedBy   = "Terraform"
    Owner       = "platform@company.com"
    CostCenter  = "CC-Production-001"
    TerraformRepo = "github.com/company/infrastructure"
  }
}

# Resource Groups
resource "azurerm_resource_group" "main" {
  name     = "rg-${local.app}-${local.env}-${local.region}"
  location = "Southeast Asia"
  tags     = local.common_tags
}

resource "azurerm_resource_group" "networking" {
  name     = "rg-network-${local.app}-${local.env}-${local.region}"
  location = "Southeast Asia"
  tags = merge(local.common_tags, { Purpose = "Networking" })
}

resource "azurerm_resource_group" "security" {
  name     = "rg-security-${local.app}-${local.env}-${local.region}"
  location = "Southeast Asia"
  tags = merge(local.common_tags, { Purpose = "Security" })
}

resource "azurerm_resource_group" "monitoring" {
  name     = "rg-monitoring-${local.app}-${local.env}-${local.region}"
  location = "Southeast Asia"
  tags = merge(local.common_tags, { Purpose = "Monitoring" })
}

# Locks for Production
resource "azurerm_management_lock" "main_lock" {
  name       = "lock-${azurerm_resource_group.main.name}"
  scope      = azurerm_resource_group.main.id
  lock_level = "CanNotDelete"
  notes      = "Production Resource Group - Do Not Delete"
}

# Outputs
output "resource_group_ids" {
  value = {
    main       = azurerm_resource_group.main.id
    networking = azurerm_resource_group.networking.id
    security   = azurerm_resource_group.security.id
    monitoring = azurerm_resource_group.monitoring.id
  }
}

output "subscription_info" {
  value = {
    id           = data.azurerm_subscription.current.subscription_id
    display_name = data.azurerm_subscription.current.display_name
  }
}
```

---

## สรุป: Resource Group Best Practices

| Pattern | เหมาะสำหรับ | ข้อดี | ข้อเสีย |
|---------|------------|-------|---------|
| Per Environment | Small-medium projects | ง่ายต่อการจัดการ | RBAC ซับซ้อนเมื่อมีหลาย apps |
| Per Application | Multiple applications | ความรับผิดชอบชัดเจน | อาจมี RG มากเกินไป |
| Per Team | Large organizations | Team autonomy | Shared resources ยาก |
| Layered | Enterprise | Security boundaries ชัด | ซับซ้อนในการ coordinate |

### Resource Group Limits

- สูงสุด **980 Resource Groups** ต่อ subscription
- สูงสุด **800 resource deployments** ต่อ resource group
- Resource Groups สามารถมี resources ข้าม regions ได้

---

*จบ Part 052: Azure Resource Groups & Management*  
*ต่อไป Part 053: Azure Virtual Networks*
