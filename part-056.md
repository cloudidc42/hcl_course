# Part 056: Azure Storage Accounts
## ขั้นตอนที่ 551-560: การจัดการ Azure Storage Services

---

## ขั้นตอนที่ 551: Storage Account พื้นฐาน

Azure Storage Account เป็น container ระดับบนสุดสำหรับ storage services ทั้งหมด

```hcl
# storage-account-basic.tf

# Storage Account พื้นฐาน (General Purpose v2)
resource "azurerm_storage_account" "main" {
  name                     = "stmyappprod001"  # globally unique, 3-24 chars, lowercase+numbers only
  resource_group_name      = azurerm_resource_group.main.name
  location                 = azurerm_resource_group.main.location
  account_kind             = "StorageV2"    # StorageV2 (แนะนำ), BlobStorage, Storage
  account_tier             = "Standard"    # Standard, Premium
  account_replication_type = "GRS"         # LRS, ZRS, GRS, GZRS, RA-GRS, RA-GZRS
  
  # Access tier (สำหรับ Blob)
  access_tier = "Hot"  # Hot, Cool
  
  # TLS
  min_tls_version = "TLS1_2"
  
  # Allow Azure services only
  allow_nested_items_to_be_public = false
  
  # Secure transfer required
  enable_https_traffic_only = true
  
  # Shared Key access (ควรปิดถ้าใช้ Azure AD)
  shared_access_key_enabled = false  # ใช้ Azure AD RBAC แทน
  
  # Hierarchical namespace (สำหรับ Azure Data Lake Storage Gen2)
  is_hns_enabled = false
  
  # Large file shares (สำหรับ File Shares > 5 TiB)
  large_file_share_enabled = false
  
  tags = local.common_tags
}

# Storage Account สำหรับ Data Lake
resource "azurerm_storage_account" "datalake" {
  name                     = "stdatalakeprod001"
  resource_group_name      = azurerm_resource_group.main.name
  location                 = azurerm_resource_group.main.location
  account_kind             = "StorageV2"
  account_tier             = "Standard"
  account_replication_type = "GZRS"
  access_tier              = "Hot"
  
  # Enable HNS สำหรับ Data Lake Storage Gen2
  is_hns_enabled = true
  
  # Enable NFS 3.0 (ต้องการ HNS)
  nfsv3_enabled = false
  
  min_tls_version           = "TLS1_2"
  enable_https_traffic_only = true
}

# Storage Account สำหรับ Premium Blob
resource "azurerm_storage_account" "premium_blob" {
  name                     = "stpremblob001"
  resource_group_name      = azurerm_resource_group.main.name
  location                 = azurerm_resource_group.main.location
  account_kind             = "BlockBlobStorage"  # สำหรับ Premium Blob
  account_tier             = "Premium"
  account_replication_type = "LRS"  # Premium ต้องใช้ LRS หรือ ZRS
}

# Storage Account สำหรับ Premium Files
resource "azurerm_storage_account" "premium_files" {
  name                     = "stpremfiles001"
  resource_group_name      = azurerm_resource_group.main.name
  location                 = azurerm_resource_group.main.location
  account_kind             = "FileStorage"  # สำหรับ Premium Files
  account_tier             = "Premium"
  account_replication_type = "ZRS"
  
  large_file_share_enabled = true
}
```

---

## ขั้นตอนที่ 552: Account Kinds และ Replication Types

```hcl
# storage-types-reference.tf

locals {
  # Account Kind comparison
  account_kinds = {
    StorageV2 = {
      description = "General-purpose v2 (recommended)"
      services    = ["Blob", "File", "Queue", "Table", "Data Lake Gen2"]
      tiers       = ["Standard", "Premium (Block Blob)"]
      access_tiers = ["Hot", "Cool", "Archive (blob level)"]
    }
    BlobStorage = {
      description = "Blob-only storage (legacy)"
      services    = ["Blob"]
      tiers       = ["Standard"]
      access_tiers = ["Hot", "Cool"]
    }
    Storage = {
      description = "General-purpose v1 (legacy)"
      services    = ["Blob", "File", "Queue", "Table"]
      tiers       = ["Standard", "Premium"]
    }
    BlockBlobStorage = {
      description = "Premium block blob storage"
      services    = ["Blob (Block and Append only)"]
      tiers       = ["Premium"]
    }
    FileStorage = {
      description = "Premium file storage"
      services    = ["File"]
      tiers       = ["Premium"]
    }
  }
  
  # Replication types
  replication_types = {
    LRS   = "Locally Redundant Storage - 3 copies in same datacenter"
    ZRS   = "Zone Redundant Storage - 3 copies across zones in same region"
    GRS   = "Geo Redundant Storage - LRS + 3 copies in secondary region"
    GZRS  = "Geo Zone Redundant Storage - ZRS + 3 copies in secondary region"
    RAGRS = "Read-Access GRS - GRS + read access to secondary"
    RAGZRS = "Read-Access GZRS - GZRS + read access to secondary"
  }
}

# Choose replication based on requirements
variable "storage_replication" {
  type        = string
  description = "Storage replication type"
  
  validation {
    condition     = contains(["LRS", "ZRS", "GRS", "GZRS", "RAGRS", "RAGZRS"], var.storage_replication)
    error_message = "Invalid replication type."
  }
}

# Recommended configurations by use case
locals {
  storage_configs = {
    # Development/testing - cheapest
    dev = {
      replication  = "LRS"
      access_tier  = "Cool"
    }
    
    # Production - high availability
    prod = {
      replication  = "GZRS"
      access_tier  = "Hot"
    }
    
    # Backup/archive - cheapest durable option
    backup = {
      replication  = "GRS"
      access_tier  = "Cool"
    }
    
    # Critical with read access during failover
    critical = {
      replication  = "RAGZRS"
      access_tier  = "Hot"
    }
  }
}
```

---

## ขั้นตอนที่ 553: Blob Containers

```hcl
# blob-containers.tf

# Public container (สำหรับ static website assets)
resource "azurerm_storage_container" "public_assets" {
  name                  = "public-assets"
  storage_account_name  = azurerm_storage_account.main.name
  container_access_type = "blob"  # private, blob, container
}

# Private container
resource "azurerm_storage_container" "app_data" {
  name                  = "app-data"
  storage_account_name  = azurerm_storage_account.main.name
  container_access_type = "private"
}

# Terraform state container
resource "azurerm_storage_container" "tfstate" {
  name                  = "tfstate"
  storage_account_name  = azurerm_storage_account.main.name
  container_access_type = "private"
}

# Logs container
resource "azurerm_storage_container" "logs" {
  name                  = "logs"
  storage_account_name  = azurerm_storage_account.main.name
  container_access_type = "private"
}

# Backup container
resource "azurerm_storage_container" "backups" {
  name                  = "backups"
  storage_account_name  = azurerm_storage_account.main.name
  container_access_type = "private"
}

# Multiple containers with for_each
locals {
  containers = {
    "raw-data"      = "private"
    "processed"     = "private"
    "exports"       = "private"
    "public-files"  = "blob"
  }
}

resource "azurerm_storage_container" "multiple" {
  for_each = local.containers
  
  name                  = each.key
  storage_account_name  = azurerm_storage_account.main.name
  container_access_type = each.value
}
```

---

## ขั้นตอนที่ 554: Blob Storage Operations

```hcl
# blob-operations.tf

# Upload a file
resource "azurerm_storage_blob" "app_config" {
  name                   = "config/app-config.json"
  storage_account_name   = azurerm_storage_account.main.name
  storage_container_name = azurerm_storage_container.app_data.name
  type                   = "Block"  # Block, Append, Page
  source                 = "${path.module}/config/app-config.json"
  content_type           = "application/json"
  
  # Metadata
  metadata = {
    environment = "production"
    version     = "1.0.0"
    created_by  = "terraform"
  }
}

# Upload multiple files
locals {
  static_files = fileset("${path.module}/static", "**")
}

resource "azurerm_storage_blob" "static_files" {
  for_each = local.static_files
  
  name                   = each.value
  storage_account_name   = azurerm_storage_account.main.name
  storage_container_name = azurerm_storage_container.public_assets.name
  type                   = "Block"
  source                 = "${path.module}/static/${each.value}"
  content_type           = lookup({
    "html" = "text/html"
    "css"  = "text/css"
    "js"   = "application/javascript"
    "png"  = "image/png"
    "jpg"  = "image/jpeg"
    "svg"  = "image/svg+xml"
  }, split(".", each.value)[length(split(".", each.value)) - 1], "application/octet-stream")
}

# Append blob (สำหรับ logs)
resource "azurerm_storage_blob" "log_file" {
  name                   = "app.log"
  storage_account_name   = azurerm_storage_account.main.name
  storage_container_name = azurerm_storage_container.logs.name
  type                   = "Append"
  source_content         = ""  # Empty initial content
}

# Page blob (สำหรับ VM disks, VHD files)
resource "azurerm_storage_blob" "vhd_disk" {
  name                   = "vm-disk.vhd"
  storage_account_name   = azurerm_storage_account.main.name
  storage_container_name = azurerm_storage_container.app_data.name
  type                   = "Page"
  size                   = 10737418240  # 10 GB in bytes
}
```

---

## ขั้นตอนที่ 555: File Shares

```hcl
# file-shares.tf

# File Share
resource "azurerm_storage_share" "app_share" {
  name                 = "appshare"
  storage_account_name = azurerm_storage_account.main.name
  quota                = 100  # GB
  
  access_tier = "Hot"  # Hot, Cool, TransactionOptimized
  
  # ACL สำหรับ access control
  acl {
    id = "app-read-write"
    
    access_policy {
      permissions = "rwd"  # r=read, w=write, d=delete, l=list, c=create
      start       = "2024-01-01T00:00:00Z"
      expiry      = "2025-12-31T23:59:59Z"
    }
  }
}

# Large File Share (> 5 TiB)
resource "azurerm_storage_share" "large_share" {
  name                 = "largeshare"
  storage_account_name = azurerm_storage_account.premium_files.name
  quota                = 5120  # 5 TiB = 5120 GiB
  
  enabled_protocol = "SMB"  # SMB หรือ NFS (ต้องการ Premium tier)
}

# File Share Directory
resource "azurerm_storage_share_directory" "configs" {
  name                 = "configs"
  share_name           = azurerm_storage_share.app_share.name
  storage_account_name = azurerm_storage_account.main.name
}

resource "azurerm_storage_share_directory" "logs" {
  name                 = "logs"
  share_name           = azurerm_storage_share.app_share.name
  storage_account_name = azurerm_storage_account.main.name
}

# File in Share
resource "azurerm_storage_share_file" "app_config" {
  name             = "app.conf"
  path             = azurerm_storage_share_directory.configs.name
  storage_share_id = azurerm_storage_share.app_share.id
  source           = "${path.module}/config/app.conf"
}
```

---

## ขั้นตอนที่ 556: Queue Storage

```hcl
# queue-storage.tf

# Storage Queue
resource "azurerm_storage_queue" "main" {
  name                 = "main-queue"
  storage_account_name = azurerm_storage_account.main.name
}

# Multiple Queues
locals {
  queues = [
    "orders",
    "notifications",
    "email-jobs",
    "dead-letter"
  ]
}

resource "azurerm_storage_queue" "multiple" {
  for_each = toset(local.queues)
  
  name                 = each.value
  storage_account_name = azurerm_storage_account.main.name
}

# Table Storage
resource "azurerm_storage_table" "app_metadata" {
  name                 = "AppMetadata"
  storage_account_name = azurerm_storage_account.main.name
}

resource "azurerm_storage_table" "user_sessions" {
  name                 = "UserSessions"
  storage_account_name = azurerm_storage_account.main.name
}

# Table Entity
resource "azurerm_storage_table_entity" "config_entry" {
  storage_account_name = azurerm_storage_account.main.name
  table_name           = azurerm_storage_table.app_metadata.name
  
  partition_key = "config"
  row_key       = "app-version"
  
  entity = {
    Value       = "1.2.3"
    UpdatedAt   = "2024-01-15"
    UpdatedBy   = "terraform"
  }
}
```

---

## ขั้นตอนที่ 557: Storage Networking และ Security

```hcl
# storage-security.tf

# Storage Account พร้อม Network Rules
resource "azurerm_storage_account" "secure" {
  name                     = "stsecureprod001"
  resource_group_name      = azurerm_resource_group.main.name
  location                 = azurerm_resource_group.main.location
  account_kind             = "StorageV2"
  account_tier             = "Standard"
  account_replication_type = "GRS"
  
  # ปิด public access โดย default
  public_network_access_enabled   = false
  allow_nested_items_to_be_public = false
  
  min_tls_version           = "TLS1_2"
  enable_https_traffic_only = true
  
  # Disable shared access key
  shared_access_key_enabled = false
  
  # Identity สำหรับ encryption
  identity {
    type = "UserAssigned"
    identity_ids = [azurerm_user_assigned_identity.storage.id]
  }
  
  # Customer-managed key encryption
  customer_managed_key {
    key_vault_key_id          = azurerm_key_vault_key.storage.id
    user_assigned_identity_id = azurerm_user_assigned_identity.storage.id
  }
  
  # Blob properties
  blob_properties {
    # Versioning
    versioning_enabled = true
    
    # Soft delete สำหรับ blobs
    delete_retention_policy {
      days = 30
    }
    
    # Soft delete สำหรับ containers
    container_delete_retention_policy {
      days = 30
    }
    
    # Change feed
    change_feed_enabled            = true
    change_feed_retention_in_days  = 30
    
    # Last access time tracking
    last_access_time_enabled = true
    
    # CORS
    cors_rule {
      allowed_headers    = ["*"]
      allowed_methods    = ["GET", "HEAD", "POST", "OPTIONS"]
      allowed_origins    = ["https://myapp.com", "https://www.myapp.com"]
      exposed_headers    = ["*"]
      max_age_in_seconds = 3600
    }
  }
  
  # Queue properties
  queue_properties {
    logging {
      delete                = true
      read                  = true
      write                 = true
      version               = "1.0"
      retention_policy_days = 10
    }
    
    hour_metrics {
      enabled               = true
      version               = "1.0"
      include_apis          = true
      retention_policy_days = 10
    }
  }
  
  # Share properties
  share_properties {
    cors_rule {
      allowed_headers    = ["*"]
      allowed_methods    = ["GET", "HEAD", "OPTIONS"]
      allowed_origins    = ["https://myapp.com"]
      exposed_headers    = ["*"]
      max_age_in_seconds = 3600
    }
    
    retention_policy {
      days = 7
    }
    
    smb {
      versions                        = ["SMB3.0", "SMB3.1.1"]
      authentication_types            = ["NTLMv2", "Kerberos"]
      kerberos_ticket_encryption_type = ["AES-256"]
      channel_encryption_type         = ["AES-128-CCM", "AES-128-GCM", "AES-256-GCM"]
      multichannel_enabled            = false
    }
  }
  
  tags = local.common_tags
}

# Network rules สำหรับ Storage Account
resource "azurerm_storage_account_network_rules" "secure" {
  storage_account_id = azurerm_storage_account.secure.id
  
  default_action = "Deny"
  
  # Allow specific IP ranges
  ip_rules = [
    "203.0.113.0/24",  # Office network
    "198.51.100.1"     # VPN endpoint
  ]
  
  # Allow specific VNet subnets
  virtual_network_subnet_ids = [
    azurerm_subnet.app.id,
    azurerm_subnet.web.id,
    azurerm_subnet.aks_nodes.id
  ]
  
  # Bypass rules
  bypass = ["AzureServices", "Logging", "Metrics"]
  
  # Private Link access
  private_link_access {
    endpoint_resource_id = azurerm_search_service.main.id
    endpoint_tenant_id   = data.azurerm_client_config.current.tenant_id
  }
}
```

---

## ขั้นตอนที่ 558: Lifecycle Management Policies

```hcl
# lifecycle-management.tf

resource "azurerm_storage_management_policy" "main" {
  storage_account_id = azurerm_storage_account.main.id
  
  rule {
    name    = "move-to-cool-after-30-days"
    enabled = true
    
    filters {
      prefix_match = ["app-data/"]
      blob_types   = ["blockBlob"]
      
      match_blob_index_tag {
        name      = "tier"
        operation = "=="
        value     = "moveable"
      }
    }
    
    actions {
      base_blob {
        # Move to Cool tier after 30 days of no access
        tier_to_cool_after_days_since_last_access_time_greater_than = 30
        
        # Move to Archive after 90 days
        tier_to_archive_after_days_since_last_access_time_greater_than = 90
        
        # Delete after 365 days
        delete_after_days_since_last_access_time_greater_than = 365
      }
      
      snapshot {
        # Delete snapshots older than 30 days
        delete_after_days_since_creation_greater_than = 30
      }
      
      version {
        # Delete non-current versions after 30 days
        delete_after_days_since_creation = 30
      }
    }
  }
  
  rule {
    name    = "archive-logs"
    enabled = true
    
    filters {
      prefix_match = ["logs/"]
      blob_types   = ["appendBlob", "blockBlob"]
    }
    
    actions {
      base_blob {
        tier_to_cool_after_days_since_modification_greater_than    = 7
        tier_to_archive_after_days_since_modification_greater_than = 30
        delete_after_days_since_modification_greater_than          = 90
      }
    }
  }
  
  rule {
    name    = "delete-temp-files"
    enabled = true
    
    filters {
      prefix_match = ["temp/"]
      blob_types   = ["blockBlob"]
    }
    
    actions {
      base_blob {
        delete_after_days_since_creation_greater_than = 7
      }
    }
  }
}
```

---

## ขั้นตอนที่ 559: Static Website Hosting

```hcl
# static-website.tf

# Storage Account สำหรับ Static Website
resource "azurerm_storage_account" "website" {
  name                     = "stwebsite001"
  resource_group_name      = azurerm_resource_group.main.name
  location                 = azurerm_resource_group.main.location
  account_kind             = "StorageV2"
  account_tier             = "Standard"
  account_replication_type = "GRS"
  
  # Static website configuration
  static_website {
    index_document     = "index.html"
    error_404_document = "404.html"
  }
  
  # Public access required for static website
  allow_nested_items_to_be_public = true
  
  # CORS
  blob_properties {
    cors_rule {
      allowed_headers    = ["*"]
      allowed_methods    = ["GET", "HEAD", "OPTIONS"]
      allowed_origins    = ["*"]
      exposed_headers    = ["Content-Type"]
      max_age_in_seconds = 86400
    }
  }
}

# Upload website files
resource "azurerm_storage_blob" "index_html" {
  name                   = "index.html"
  storage_account_name   = azurerm_storage_account.website.name
  storage_container_name = "$web"  # Special container for static website
  type                   = "Block"
  source                 = "${path.module}/website/index.html"
  content_type           = "text/html"
}

resource "azurerm_storage_blob" "error_html" {
  name                   = "404.html"
  storage_account_name   = azurerm_storage_account.website.name
  storage_container_name = "$web"
  type                   = "Block"
  source                 = "${path.module}/website/404.html"
  content_type           = "text/html"
}

# Azure CDN สำหรับ Static Website
resource "azurerm_cdn_profile" "main" {
  name                = "cdn-myapp-prod"
  resource_group_name = azurerm_resource_group.main.name
  location            = "global"
  sku                 = "Standard_Microsoft"
}

resource "azurerm_cdn_endpoint" "website" {
  name                = "cdn-endpoint-myapp"
  profile_name        = azurerm_cdn_profile.main.name
  resource_group_name = azurerm_resource_group.main.name
  location            = "global"
  
  origin {
    name      = "storage-origin"
    host_name = azurerm_storage_account.website.primary_web_host
  }
  
  origin_host_header = azurerm_storage_account.website.primary_web_host
  
  # Compression
  is_compression_enabled = true
  content_types_to_compress = [
    "text/html",
    "text/css",
    "application/javascript",
    "application/json"
  ]
  
  # Query string caching
  querystring_caching_behaviour = "IgnoreQueryString"
}

output "website_url" {
  value = azurerm_storage_account.website.primary_web_endpoint
}

output "cdn_url" {
  value = "https://${azurerm_cdn_endpoint.website.host_name}"
}
```

---

## ขั้นตอนที่ 560: Terraform State Backend

```hcl
# terraform-state-backend.tf

# Storage Account สำหรับ Terraform State
resource "azurerm_storage_account" "terraform_state" {
  name                     = "stterraformstate001"
  resource_group_name      = azurerm_resource_group.terraform.name
  location                 = azurerm_resource_group.terraform.location
  account_kind             = "StorageV2"
  account_tier             = "Standard"
  account_replication_type = "GRS"
  
  # Security
  min_tls_version           = "TLS1_2"
  enable_https_traffic_only = true
  allow_nested_items_to_be_public = false
  
  # Versioning สำคัญมากสำหรับ state file
  blob_properties {
    versioning_enabled = true
    
    delete_retention_policy {
      days = 30
    }
    
    container_delete_retention_policy {
      days = 30
    }
  }
  
  tags = {
    Purpose   = "Terraform State"
    ManagedBy = "Platform Team"
  }
}

# Container สำหรับ Terraform State
resource "azurerm_storage_container" "terraform_state" {
  name                  = "tfstate"
  storage_account_name  = azurerm_storage_account.terraform_state.name
  container_access_type = "private"
}

# Lock for Terraform State Storage
resource "azurerm_management_lock" "state_storage" {
  name       = "lock-terraform-state-storage"
  scope      = azurerm_storage_account.terraform_state.id
  lock_level = "CanNotDelete"
  notes      = "Terraform state storage - do not delete"
}

# Grant Contributor to CI/CD SP
resource "azurerm_role_assignment" "cicd_state_contributor" {
  scope                = azurerm_storage_account.terraform_state.id
  role_definition_name = "Storage Blob Data Contributor"
  principal_id         = var.cicd_service_principal_object_id
}

# Backend configuration (ใส่ใน terraform block)
/*
terraform {
  backend "azurerm" {
    resource_group_name  = "rg-terraform-state"
    storage_account_name = "stterraformstate001"
    container_name       = "tfstate"
    key                  = "prod/main.tfstate"
    
    # Authentication (ใช้ environment variables)
    # ARM_CLIENT_ID, ARM_CLIENT_SECRET, ARM_TENANT_ID, ARM_SUBSCRIPTION_ID
  }
}
*/

# ============================================================
# Role Assignments สำหรับ Storage
# ============================================================

# Allow specific users to read blobs
resource "azurerm_role_assignment" "data_reader" {
  scope                = azurerm_storage_account.main.id
  role_definition_name = "Storage Blob Data Reader"
  principal_id         = data.azuread_group.data_team.object_id
}

# Allow specific services to write blobs
resource "azurerm_role_assignment" "app_blob_contributor" {
  scope                = azurerm_storage_container.app_data.resource_manager_id
  role_definition_name = "Storage Blob Data Contributor"
  principal_id         = azurerm_linux_virtual_machine.app_server.identity[0].principal_id
}

# Storage Account Key Operator
resource "azurerm_role_assignment" "key_operator" {
  scope                = azurerm_storage_account.main.id
  role_definition_name = "Storage Account Key Operator Service Role"
  principal_id         = var.key_vault_object_id
}

# ============================================================
# Storage Account Complete Outputs
# ============================================================
output "storage_account_info" {
  value = {
    name                = azurerm_storage_account.main.name
    id                  = azurerm_storage_account.main.id
    primary_blob_endpoint = azurerm_storage_account.main.primary_blob_endpoint
    primary_file_endpoint = azurerm_storage_account.main.primary_file_endpoint
    primary_queue_endpoint = azurerm_storage_account.main.primary_queue_endpoint
    primary_table_endpoint = azurerm_storage_account.main.primary_table_endpoint
    primary_web_endpoint   = azurerm_storage_account.main.primary_web_endpoint
  }
}

output "storage_connection_string" {
  value     = azurerm_storage_account.main.primary_connection_string
  sensitive = true
}

output "storage_access_key" {
  value     = azurerm_storage_account.main.primary_access_key
  sensitive = true
}
```

### Shared Access Signature (SAS) Token

```hcl
# sas-tokens.tf

# SAS Token สำหรับ Blob
data "azurerm_storage_account_blob_container_sas" "blob_sas" {
  connection_string = azurerm_storage_account.main.primary_connection_string
  container_name    = azurerm_storage_container.app_data.name
  https_only        = true
  
  start  = "2024-01-01"
  expiry = "2024-12-31"
  
  permissions {
    read   = true
    add    = false
    create = false
    write  = false
    delete = false
    list   = true
  }
  
  ip_address = "203.0.113.0/24"  # Restrict to specific IP
}

output "blob_sas_token" {
  value     = data.azurerm_storage_account_blob_container_sas.blob_sas.sas
  sensitive = true
}

# SAS Token สำหรับ Storage Account
data "azurerm_storage_account_sas" "account_sas" {
  connection_string = azurerm_storage_account.main.primary_connection_string
  https_only        = true
  signed_version    = "2020-08-04"
  
  resource_types {
    service   = false
    container = false
    object    = true
  }
  
  services {
    blob  = true
    queue = false
    table = false
    file  = false
  }
  
  start  = "2024-01-01"
  expiry = "2024-12-31"
  
  permissions {
    read    = true
    write   = true
    delete  = false
    list    = true
    add     = true
    create  = true
    update  = false
    process = false
    tag     = false
    filter  = false
  }
}
```

---

## Storage Account Cost Comparison

| Replication | Cost | RPO | RTO | Use Case |
|-------------|------|-----|-----|----------|
| LRS | $ | 0 | N/A | Dev/Test, non-critical |
| ZRS | $$ | 0 | Low | Production in one region |
| GRS | $$$ | < 15 min | Hours | Production with DR |
| GZRS | $$$$ | < 15 min | Hours | Mission-critical |
| RA-GRS | $$$$ | < 15 min | Minutes | Critical + read DR |
| RA-GZRS | $$$$$ | < 15 min | Minutes | Highest availability |

---

*จบ Part 056: Azure Storage Accounts*  
*ต่อไป Part 057: Azure Database Services*
