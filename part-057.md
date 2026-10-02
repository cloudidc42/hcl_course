# Part 057: Azure Database Services
## ขั้นตอนที่ 561-570: Azure SQL, PostgreSQL, MySQL, Cosmos DB, Redis

---

## ขั้นตอนที่ 561: Azure SQL Server และ Database

```hcl
# azure-sql.tf

# Azure SQL Server
resource "azurerm_mssql_server" "main" {
  name                         = "sql-myapp-prod-sea"
  resource_group_name          = azurerm_resource_group.data.name
  location                     = azurerm_resource_group.data.location
  version                      = "12.0"
  
  # Authentication
  administrator_login          = "sqladmin"
  administrator_login_password = random_password.sql_admin.result
  
  # Azure AD Authentication
  azuread_administrator {
    login_username = "AzureAD Admin"
    object_id      = var.sql_aad_admin_object_id
    tenant_id      = data.azurerm_client_config.current.tenant_id
    azuread_authentication_only = false  # Allow both SQL + AAD
  }
  
  # Minimum TLS version
  minimum_tls_version = "1.2"
  
  # Public access (สำหรับ dev เท่านั้น)
  public_network_access_enabled = false
  
  # Identity สำหรับ Transparent Data Encryption
  identity {
    type = "SystemAssigned"
  }
  
  tags = local.common_tags
}

# Random password สำหรับ SQL admin
resource "random_password" "sql_admin" {
  length           = 20
  special          = true
  override_special = "!@#$%&*()-_=+[]{}<>:?"
}

# Store password in Key Vault
resource "azurerm_key_vault_secret" "sql_admin_password" {
  name         = "sql-admin-password"
  value        = random_password.sql_admin.result
  key_vault_id = azurerm_key_vault.main.id
}

# SQL Database
resource "azurerm_mssql_database" "app" {
  name      = "db-myapp-prod"
  server_id = azurerm_mssql_server.main.id
  
  # Pricing tier
  sku_name = "GP_Gen5_4"  # General Purpose, Gen5, 4 vCores
  # Alternative: "BC_Gen5_2" (Business Critical), "HS_Gen5_2" (Hyperscale)
  # DTU-based: "S3", "P1", "Basic"
  
  # Collation
  collation = "SQL_Latin1_General_CP1_CI_AS"
  
  # Max size
  max_size_gb = 100
  
  # Backup retention
  short_term_retention_policy {
    retention_days           = 35
    backup_interval_in_hours = 24
  }
  
  long_term_retention_policy {
    weekly_retention  = "P4W"   # Keep weekly backups for 4 weeks
    monthly_retention = "P12M"  # Keep monthly backups for 12 months
    yearly_retention  = "P5Y"   # Keep yearly backups for 5 years
    week_of_year      = 1
  }
  
  # Transparent Data Encryption (TDE)
  transparent_data_encryption_enabled = true
  
  # Auto Pause (สำหรับ serverless เท่านั้น)
  # auto_pause_delay_in_minutes = 60
  
  # Read Scale
  read_scale = false
  
  # Zone Redundant (Business Critical และ Premium เท่านั้น)
  # zone_redundant = true
  
  # Geo-Backup
  geo_backup_enabled = true
  
  # Elastic pool
  # elastic_pool_id = azurerm_mssql_elasticpool.main.id
  
  tags = local.common_tags
}

# SQL Serverless Database (scale compute to zero)
resource "azurerm_mssql_database" "serverless" {
  name      = "db-serverless-prod"
  server_id = azurerm_mssql_server.main.id
  sku_name  = "GP_S_Gen5_1"  # Serverless: S = Serverless
  
  auto_pause_delay_in_minutes = 60  # Pause after 60 min inactivity
  
  min_capacity = 0.5  # Minimum vCores
  max_size_gb  = 50
}

# Elastic Pool
resource "azurerm_mssql_elasticpool" "main" {
  name                = "ep-myapp-prod"
  resource_group_name = azurerm_resource_group.data.name
  location            = azurerm_resource_group.data.location
  server_name         = azurerm_mssql_server.main.name
  
  sku {
    name     = "GP_Gen5"
    tier     = "GeneralPurpose"
    family   = "Gen5"
    capacity = 8  # 8 vCores
  }
  
  per_database_settings {
    min_capacity = 0
    max_capacity = 4
  }
  
  max_size_gb = 200
}
```

---

## ขั้นตอนที่ 562: SQL Firewall Rules และ Network Security

```hcl
# sql-security.tf

# Firewall Rules (สำหรับ dev เท่านั้น!)
resource "azurerm_mssql_firewall_rule" "office" {
  name             = "office-network"
  server_id        = azurerm_mssql_server.main.id
  start_ip_address = "203.0.113.0"
  end_ip_address   = "203.0.113.255"
}

# Allow Azure Services (dev เท่านั้น!)
resource "azurerm_mssql_firewall_rule" "azure_services" {
  name             = "AllowAzureServices"
  server_id        = azurerm_mssql_server.main.id
  start_ip_address = "0.0.0.0"
  end_ip_address   = "0.0.0.0"
}

# VNet Rule (Production)
resource "azurerm_mssql_virtual_network_rule" "app_subnet" {
  name      = "vnet-rule-app-subnet"
  server_id = azurerm_mssql_server.main.id
  subnet_id = azurerm_subnet.app.id
}

# Private Endpoint สำหรับ SQL Server
resource "azurerm_private_endpoint" "sql" {
  name                = "pe-sql-prod"
  resource_group_name = azurerm_resource_group.network.name
  location            = azurerm_resource_group.network.location
  subnet_id           = azurerm_subnet.private_endpoints.id
  
  private_service_connection {
    name                           = "psc-sql"
    private_connection_resource_id = azurerm_mssql_server.main.id
    subresource_names              = ["sqlServer"]
    is_manual_connection           = false
  }
  
  private_dns_zone_group {
    name = "sql-dns-group"
    private_dns_zone_ids = [azurerm_private_dns_zone.sql.id]
  }
}

# SQL Auditing
resource "azurerm_mssql_server_extended_auditing_policy" "main" {
  server_id                               = azurerm_mssql_server.main.id
  storage_endpoint                        = azurerm_storage_account.audit.primary_blob_endpoint
  storage_account_access_key              = azurerm_storage_account.audit.primary_access_key
  storage_account_access_key_is_secondary = false
  retention_in_days                       = 90
  
  # Log to Log Analytics
  log_monitoring_enabled = true
}

# Database-level auditing
resource "azurerm_mssql_database_extended_auditing_policy" "app" {
  database_id                             = azurerm_mssql_database.app.id
  storage_endpoint                        = azurerm_storage_account.audit.primary_blob_endpoint
  storage_account_access_key              = azurerm_storage_account.audit.primary_access_key
  retention_in_days                       = 90
  log_monitoring_enabled                  = true
}

# Microsoft Defender for SQL
resource "azurerm_mssql_server_security_alert_policy" "main" {
  resource_group_name = azurerm_resource_group.data.name
  server_name         = azurerm_mssql_server.main.name
  state               = "Enabled"
  
  email_addresses        = ["security@company.com"]
  email_account_admins   = true
  retention_days         = 90
  
  storage_endpoint           = azurerm_storage_account.audit.primary_blob_endpoint
  storage_account_access_key = azurerm_storage_account.audit.primary_access_key
  
  disabled_alerts = []
}

# Vulnerability Assessment
resource "azurerm_mssql_server_vulnerability_assessment" "main" {
  server_security_alert_policy_id = azurerm_mssql_server_security_alert_policy.main.id
  
  storage_container_path = "${azurerm_storage_account.audit.primary_blob_endpoint}vulnerability-assessment/"
  storage_account_access_key = azurerm_storage_account.audit.primary_access_key
  
  recurring_scans {
    enabled                   = true
    email_subscription_admins = true
    emails                    = ["security@company.com"]
  }
}
```

---

## ขั้นตอนที่ 563: PostgreSQL Flexible Server

```hcl
# postgresql.tf

# PostgreSQL Flexible Server
resource "azurerm_postgresql_flexible_server" "main" {
  name                   = "psql-myapp-prod-sea"
  resource_group_name    = azurerm_resource_group.data.name
  location               = azurerm_resource_group.data.location
  version                = "15"
  
  # Authentication
  administrator_login    = "pgadmin"
  administrator_password = random_password.postgres_admin.result
  
  # SKU / Compute
  sku_name = "GP_Standard_D4s_v3"  # General Purpose, 4 vCores
  # Tier options: B (Burstable), GP (General Purpose), MO (Memory Optimized)
  # B_Standard_B1ms - 1 vCore, 2 GB RAM (Dev)
  # GP_Standard_D4s_v3 - 4 vCores, 16 GB RAM
  # MO_Standard_E4s_v3 - 4 vCores, 32 GB RAM
  
  # Storage
  storage_mb                   = 131072  # 128 GB
  storage_tier                 = "P30"   # Premium SSD tier
  auto_grow_enabled            = true
  
  # Backup
  backup_retention_days        = 30
  geo_redundant_backup_enabled = true
  
  # Availability Zone
  zone = "1"
  
  high_availability {
    mode                      = "ZoneRedundant"  # หรือ SameZone
    standby_availability_zone = "2"
  }
  
  # Maintenance window
  maintenance_window {
    day_of_week  = 0  # Sunday
    start_hour   = 1
    start_minute = 0
  }
  
  # Network - Private access only
  delegated_subnet_id    = azurerm_subnet.postgresql.id
  private_dns_zone_id    = azurerm_private_dns_zone.postgresql.id
  
  # Authentication
  authentication {
    active_directory_auth_enabled = true
    password_auth_enabled         = true  # Keep true for migration phase
    tenant_id                     = data.azurerm_client_config.current.tenant_id
  }
  
  tags = local.common_tags
  
  depends_on = [azurerm_private_dns_zone_virtual_network_link.postgresql]
}

# PostgreSQL Private DNS Zone
resource "azurerm_private_dns_zone" "postgresql" {
  name                = "privatelink.postgres.database.azure.com"
  resource_group_name = azurerm_resource_group.network.name
}

resource "azurerm_private_dns_zone_virtual_network_link" "postgresql" {
  name                  = "link-postgresql"
  resource_group_name   = azurerm_resource_group.network.name
  private_dns_zone_name = azurerm_private_dns_zone.postgresql.name
  virtual_network_id    = azurerm_virtual_network.main.id
  registration_enabled  = false
}

# Subnet สำหรับ PostgreSQL (ต้องการ delegation)
resource "azurerm_subnet" "postgresql" {
  name                 = "snet-postgresql"
  resource_group_name  = azurerm_resource_group.network.name
  virtual_network_name = azurerm_virtual_network.main.name
  address_prefixes     = ["10.0.6.0/24"]
  
  service_endpoints = ["Microsoft.Storage"]
  
  delegation {
    name = "postgresql-delegation"
    service_delegation {
      name = "Microsoft.DBforPostgreSQL/flexibleServers"
      actions = [
        "Microsoft.Network/virtualNetworks/subnets/join/action"
      ]
    }
  }
}

# PostgreSQL Databases
resource "azurerm_postgresql_flexible_server_database" "app" {
  name      = "appdb"
  server_id = azurerm_postgresql_flexible_server.main.id
  collation = "en_US.utf8"
  charset   = "utf8"
}

resource "azurerm_postgresql_flexible_server_database" "analytics" {
  name      = "analyticsdb"
  server_id = azurerm_postgresql_flexible_server.main.id
  collation = "en_US.utf8"
  charset   = "utf8"
}

# PostgreSQL Configuration
resource "azurerm_postgresql_flexible_server_configuration" "max_connections" {
  name      = "max_connections"
  server_id = azurerm_postgresql_flexible_server.main.id
  value     = "500"
}

resource "azurerm_postgresql_flexible_server_configuration" "shared_preload" {
  name      = "shared_preload_libraries"
  server_id = azurerm_postgresql_flexible_server.main.id
  value     = "pg_stat_statements,pglogical"
}

resource "azurerm_postgresql_flexible_server_configuration" "log_min_duration" {
  name      = "log_min_duration_statement"
  server_id = azurerm_postgresql_flexible_server.main.id
  value     = "1000"  # Log queries taking > 1 second
}

resource "azurerm_postgresql_flexible_server_configuration" "ssl_mode" {
  name      = "ssl"
  server_id = azurerm_postgresql_flexible_server.main.id
  value     = "on"
}

# PostgreSQL Firewall Rules (dev only - use private endpoint for prod)
resource "azurerm_postgresql_flexible_server_firewall_rule" "office" {
  name             = "office-access"
  server_id        = azurerm_postgresql_flexible_server.main.id
  start_ip_address = "203.0.113.0"
  end_ip_address   = "203.0.113.255"
}

# Azure AD Admin
resource "azurerm_postgresql_flexible_server_active_directory_administrator" "admin" {
  server_name         = azurerm_postgresql_flexible_server.main.name
  resource_group_name = azurerm_resource_group.data.name
  tenant_id           = data.azurerm_client_config.current.tenant_id
  object_id           = var.pg_aad_admin_object_id
  principal_name      = "AzureAD Admin"
  principal_type      = "Group"
}
```

---

## ขั้นตอนที่ 564: MySQL Flexible Server

```hcl
# mysql.tf

resource "azurerm_mysql_flexible_server" "main" {
  name                   = "mysql-myapp-prod-sea"
  resource_group_name    = azurerm_resource_group.data.name
  location               = azurerm_resource_group.data.location
  administrator_login    = "mysqladmin"
  administrator_password = random_password.mysql_admin.result
  
  # SKU
  sku_name = "GP_Standard_D4ds_v4"  # General Purpose
  
  version = "8.0.21"
  
  # Storage
  storage {
    auto_grow_enabled = true
    iops              = 5000
    size_gb           = 64
  }
  
  # Backup
  backup_retention_days        = 30
  geo_redundant_backup_enabled = true
  
  # High Availability
  high_availability {
    mode                      = "ZoneRedundant"
    standby_availability_zone = "2"
  }
  
  # Network
  delegated_subnet_id = azurerm_subnet.mysql.id
  private_dns_zone_id = azurerm_private_dns_zone.mysql.id
  
  # Zone
  zone = "1"
  
  tags = local.common_tags
  
  depends_on = [azurerm_private_dns_zone_virtual_network_link.mysql]
}

resource "azurerm_private_dns_zone" "mysql" {
  name                = "privatelink.mysql.database.azure.com"
  resource_group_name = azurerm_resource_group.network.name
}

resource "azurerm_private_dns_zone_virtual_network_link" "mysql" {
  name                  = "link-mysql"
  resource_group_name   = azurerm_resource_group.network.name
  private_dns_zone_name = azurerm_private_dns_zone.mysql.name
  virtual_network_id    = azurerm_virtual_network.main.id
}

resource "azurerm_subnet" "mysql" {
  name                 = "snet-mysql"
  resource_group_name  = azurerm_resource_group.network.name
  virtual_network_name = azurerm_virtual_network.main.name
  address_prefixes     = ["10.0.7.0/24"]
  
  delegation {
    name = "mysql-delegation"
    service_delegation {
      name = "Microsoft.DBforMySQL/flexibleServers"
      actions = ["Microsoft.Network/virtualNetworks/subnets/join/action"]
    }
  }
}

# MySQL Database
resource "azurerm_mysql_flexible_server_database" "app" {
  name      = "appdb"
  server_id = azurerm_mysql_flexible_server.main.id
  charset   = "utf8mb4"
  collation = "utf8mb4_unicode_ci"
}

# MySQL Configuration
resource "azurerm_mysql_flexible_server_configuration" "max_connections" {
  name      = "max_connections"
  server_id = azurerm_mysql_flexible_server.main.id
  value     = "200"
}

resource "azurerm_mysql_flexible_server_configuration" "slow_query" {
  name      = "slow_query_log"
  server_id = azurerm_mysql_flexible_server.main.id
  value     = "ON"
}

resource "azurerm_mysql_flexible_server_configuration" "long_query_time" {
  name      = "long_query_time"
  server_id = azurerm_mysql_flexible_server.main.id
  value     = "2"  # 2 seconds
}

# MySQL Firewall Rule
resource "azurerm_mysql_flexible_server_firewall_rule" "office" {
  name             = "office-access"
  server_id        = azurerm_mysql_flexible_server.main.id
  start_ip_address = "203.0.113.0"
  end_ip_address   = "203.0.113.255"
}
```

---

## ขั้นตอนที่ 565: Azure Cosmos DB

```hcl
# cosmosdb.tf

# Cosmos DB Account
resource "azurerm_cosmosdb_account" "main" {
  name                = "cosmos-myapp-prod-sea"
  resource_group_name = azurerm_resource_group.data.name
  location            = azurerm_resource_group.data.location
  offer_type          = "Standard"
  kind                = "GlobalDocumentDB"  # GlobalDocumentDB, MongoDB, Parse
  
  # Consistency level
  consistency_policy {
    consistency_level       = "BoundedStaleness"
    max_interval_in_seconds = 300   # 5 minutes
    max_staleness_prefix    = 100000
  }
  
  # Primary region
  geo_location {
    location          = "Southeast Asia"
    failover_priority = 0
  }
  
  # Secondary region
  geo_location {
    location          = "East Asia"
    failover_priority = 1
    zone_redundant    = true
  }
  
  # Serverless or Provisioned
  # capacityMode = "Serverless"  # สำหรับ variable workloads
  
  # Network access
  public_network_access_enabled = false
  is_virtual_network_filter_enabled = true
  
  virtual_network_rule {
    id                                   = azurerm_subnet.app.id
    ignore_missing_vnet_service_endpoint = false
  }
  
  # IP rules (เพิ่ม Azure Portal: 104.42.195.92, 40.76.54.131, etc.)
  ip_range_filter = "203.0.113.0/24"
  
  # Automatic failover
  enable_automatic_failover = true
  
  # Multi-write (Active-Active)
  # enable_multiple_write_locations = true
  
  # Free tier (เฉพาะ 1 account ต่อ subscription)
  # free_tier_enabled = true
  
  # Analytical Storage (Azure Synapse Link)
  analytical_storage_enabled = false
  
  # Backup
  backup {
    type                = "Periodic"
    interval_in_minutes = 240   # 4 hours
    retention_in_hours  = 720   # 30 days
    storage_redundancy  = "Geo"
  }
  
  # Identity
  identity {
    type = "SystemAssigned"
  }
  
  tags = local.common_tags
}

# SQL API Database
resource "azurerm_cosmosdb_sql_database" "app" {
  name                = "AppDatabase"
  resource_group_name = azurerm_resource_group.data.name
  account_name        = azurerm_cosmosdb_account.main.name
  
  # Shared throughput at database level
  throughput = 400  # RU/s - ลดลงได้ถ้าใช้ Autoscale
  
  # หรือใช้ Autoscale
  # autoscale_settings {
  #   max_throughput = 4000
  # }
}

# SQL API Container
resource "azurerm_cosmosdb_sql_container" "users" {
  name                = "Users"
  resource_group_name = azurerm_resource_group.data.name
  account_name        = azurerm_cosmosdb_account.main.name
  database_name       = azurerm_cosmosdb_sql_database.app.name
  
  partition_key_path    = "/userId"  # Partition key
  partition_key_version = 2          # 1 = single, 2 = hierarchical
  
  # Container-level throughput (override database)
  throughput = 400
  
  # TTL
  default_ttl = 86400  # 24 hours (set -1 to use document-level TTL)
  
  # Unique keys
  unique_key {
    paths = ["/email"]
  }
  
  # Indexing policy
  indexing_policy {
    indexing_mode = "consistent"  # consistent, none
    
    included_path {
      path = "/*"
    }
    
    excluded_path {
      path = "/largeData/*"
    }
    
    # Composite indexes สำหรับ ORDER BY
    composite_index {
      index {
        path  = "/userId"
        order = "ascending"
      }
      index {
        path  = "/createdAt"
        order = "descending"
      }
    }
    
    # Spatial indexes
    spatial_index {
      path = "/location/??"
      types = ["Point", "LineString", "Polygon"]
    }
  }
  
  # Conflict resolution policy
  conflict_resolution_policy {
    mode                          = "LastWriterWins"
    conflict_resolution_path      = "/_ts"
  }
  
  # Analytical TTL (สำหรับ Synapse Link)
  analytical_storage_ttl = -1
}

resource "azurerm_cosmosdb_sql_container" "orders" {
  name                = "Orders"
  resource_group_name = azurerm_resource_group.data.name
  account_name        = azurerm_cosmosdb_account.main.name
  database_name       = azurerm_cosmosdb_sql_database.app.name
  
  partition_key_path    = "/customerId"
  partition_key_version = 2
  
  # Autoscale throughput
  autoscale_settings {
    max_throughput = 10000  # Max 10,000 RU/s
  }
}

# MongoDB API Database
resource "azurerm_cosmosdb_account" "mongo" {
  name                = "cosmos-mongo-prod-sea"
  resource_group_name = azurerm_resource_group.data.name
  location            = azurerm_resource_group.data.location
  offer_type          = "Standard"
  kind                = "MongoDB"
  
  consistency_policy {
    consistency_level = "Session"
  }
  
  geo_location {
    location          = "Southeast Asia"
    failover_priority = 0
  }
  
  capabilities {
    name = "EnableMongo"
  }
  
  # MongoDB API version
  mongo_server_version = "6.0"
}

resource "azurerm_cosmosdb_mongo_database" "app" {
  name                = "appdb"
  resource_group_name = azurerm_resource_group.data.name
  account_name        = azurerm_cosmosdb_account.mongo.name
  throughput          = 400
}

resource "azurerm_cosmosdb_mongo_collection" "products" {
  name                = "products"
  resource_group_name = azurerm_resource_group.data.name
  account_name        = azurerm_cosmosdb_account.mongo.name
  database_name       = azurerm_cosmosdb_mongo_database.app.name
  
  # Shard key (partition key in MongoDB)
  shard_key = "category"
  
  # Indexes
  index {
    keys   = ["_id"]
    unique = true
  }
  
  index {
    keys   = ["category", "price"]
    unique = false
  }
  
  default_ttl_seconds = -1  # Disabled
  throughput          = 400
}

# Private Endpoint สำหรับ Cosmos DB
resource "azurerm_private_endpoint" "cosmosdb" {
  name                = "pe-cosmos-prod"
  resource_group_name = azurerm_resource_group.network.name
  location            = azurerm_resource_group.network.location
  subnet_id           = azurerm_subnet.private_endpoints.id
  
  private_service_connection {
    name                           = "psc-cosmos"
    private_connection_resource_id = azurerm_cosmosdb_account.main.id
    subresource_names              = ["Sql"]
    is_manual_connection           = false
  }
  
  private_dns_zone_group {
    name = "cosmos-dns-group"
    private_dns_zone_ids = [azurerm_private_dns_zone.cosmos.id]
  }
}
```

---

## ขั้นตอนที่ 566: Azure Cache for Redis

```hcl
# redis-cache.tf

# Redis Cache
resource "azurerm_redis_cache" "main" {
  name                = "redis-myapp-prod-sea"
  resource_group_name = azurerm_resource_group.data.name
  location            = azurerm_resource_group.data.location
  
  # SKU
  sku_name = "Premium"  # Basic, Standard, Premium
  family   = "P"        # C (Basic/Standard), P (Premium)
  capacity = 1          # 0-6 for Standard, 1-5 for Premium
  
  # P1 = 6GB, P2 = 13GB, P3 = 26GB, P4 = 53GB, P5 = 120GB
  
  # Redis version
  redis_version = "6"
  
  # TLS settings
  minimum_tls_version = "1.2"
  non_ssl_port_enabled = false
  
  # Premium features
  shard_count = 1  # Clustering (Premium only)
  
  # Zones (Premium only)
  zones = ["1", "2"]
  
  # Backup (Premium only)
  redis_configuration {
    maxmemory_policy   = "allkeys-lru"
    maxmemory_reserved = 100    # MB
    maxfragmentationmemory_reserved = 100
    
    # AOF persistence
    aof_backup_enabled              = false
    
    # RDB snapshot
    rdb_backup_enabled              = true
    rdb_backup_frequency            = 60    # Minutes: 15, 30, 60, 360, 720, 1440
    rdb_backup_max_snapshot_count   = 7
    rdb_storage_connection_string   = azurerm_storage_account.redis_backup.primary_blob_connection_string
    
    # Authentication
    enable_authentication = true
    
    # Active geo-replication
    # active_directory_authentication_enabled = true
    
    # Notify keyspace events
    notify_keyspace_events = "KEx"
    
    # Additional settings
    maxmemory_delta = 100
  }
  
  # Network access
  public_network_access_enabled = false
  
  # Private endpoint (จะ override subnet_id)
  # subnet_id = azurerm_subnet.redis.id  # Premium only (classic VNET injection)
  
  tags = local.common_tags
}

# Private Endpoint สำหรับ Redis (แนะนำกว่า VNET injection)
resource "azurerm_private_endpoint" "redis" {
  name                = "pe-redis-prod"
  resource_group_name = azurerm_resource_group.network.name
  location            = azurerm_resource_group.network.location
  subnet_id           = azurerm_subnet.private_endpoints.id
  
  private_service_connection {
    name                           = "psc-redis"
    private_connection_resource_id = azurerm_redis_cache.main.id
    subresource_names              = ["redisCache"]
    is_manual_connection           = false
  }
  
  private_dns_zone_group {
    name = "redis-dns-group"
    private_dns_zone_ids = [azurerm_private_dns_zone.redis.id]
  }
}

resource "azurerm_private_dns_zone" "redis" {
  name                = "privatelink.redis.cache.windows.net"
  resource_group_name = azurerm_resource_group.network.name
}

resource "azurerm_private_dns_zone_virtual_network_link" "redis" {
  name                  = "link-redis"
  resource_group_name   = azurerm_resource_group.network.name
  private_dns_zone_name = azurerm_private_dns_zone.redis.name
  virtual_network_id    = azurerm_virtual_network.main.id
}

# Redis Enterprise (Enterprise sku)
resource "azurerm_redis_enterprise_cluster" "main" {
  name                = "redisenterprise-myapp-prod"
  resource_group_name = azurerm_resource_group.data.name
  location            = azurerm_resource_group.data.location
  
  sku_name = "Enterprise_E10-2"
  zones    = ["1", "2", "3"]
}

resource "azurerm_redis_enterprise_database" "app" {
  cluster_id          = azurerm_redis_enterprise_cluster.main.id
  
  client_protocol     = "Encrypted"  # Encrypted, Plaintext
  clustering_policy   = "OSSCluster"  # EnterpriseCluster, OSSCluster
  eviction_policy     = "AllKeysLRU"
  
  module {
    name = "RedisJSON"
  }
  
  module {
    name = "RediSearch"
    args = "MAXDOCTABLESIZE 10000"
  }
}
```

---

## ขั้นตอนที่ 567: Database Security Best Practices

```hcl
# database-security.tf

# ============================================================
# Transparent Data Encryption (TDE) with Customer-Managed Keys
# ============================================================

# Key Vault Key สำหรับ SQL TDE
resource "azurerm_key_vault_key" "sql_tde" {
  name         = "key-sql-tde-prod"
  key_vault_id = azurerm_key_vault.main.id
  key_type     = "RSA"
  key_size     = 2048
  
  key_opts = ["decrypt", "encrypt", "sign", "unwrapKey", "verify", "wrapKey"]
  
  rotation_policy {
    automatic {
      time_before_expiry = "P30D"
    }
    expire_after         = "P1Y"
    notify_before_expiry = "P30D"
  }
}

# Grant SQL Server access to Key Vault
resource "azurerm_role_assignment" "sql_kv_access" {
  scope                = azurerm_key_vault.main.id
  role_definition_name = "Key Vault Crypto Service Encryption User"
  principal_id         = azurerm_mssql_server.main.identity[0].principal_id
}

# SQL TDE with Customer-Managed Key
resource "azurerm_mssql_server_transparent_data_encryption" "main" {
  server_id        = azurerm_mssql_server.main.id
  key_vault_key_id = azurerm_key_vault_key.sql_tde.id
  
  depends_on = [azurerm_role_assignment.sql_kv_access]
}

# ============================================================
# Azure SQL - Advanced Threat Protection
# ============================================================

resource "azurerm_mssql_server_microsoft_support_auditing_policy" "main" {
  server_id                       = azurerm_mssql_server.main.id
  blob_storage_endpoint           = azurerm_storage_account.audit.primary_blob_endpoint
  storage_account_access_key      = azurerm_storage_account.audit.primary_access_key
  log_to_audit_log_analytics_enabled = true
  enabled                         = true
}

# ============================================================
# Connection Strings (stored in Key Vault)
# ============================================================

resource "azurerm_key_vault_secret" "sql_connection_string" {
  name         = "sql-connection-string"
  value        = "Server=${azurerm_mssql_server.main.fully_qualified_domain_name};Database=${azurerm_mssql_database.app.name};Authentication=Active Directory Managed Identity;"
  key_vault_id = azurerm_key_vault.main.id
}

resource "azurerm_key_vault_secret" "postgres_connection_string" {
  name  = "postgres-connection-string"
  value = "Host=${azurerm_postgresql_flexible_server.main.fqdn};Database=${azurerm_postgresql_flexible_server_database.app.name};Username=${azurerm_postgresql_flexible_server.main.administrator_login};Password=${random_password.postgres_admin.result};SslMode=Require"
  key_vault_id = azurerm_key_vault.main.id
}

resource "azurerm_key_vault_secret" "redis_connection_string" {
  name         = "redis-connection-string"
  value        = "${azurerm_redis_cache.main.hostname}:${azurerm_redis_cache.main.ssl_port},password=${azurerm_redis_cache.main.primary_access_key},ssl=True,abortConnect=False"
  key_vault_id = azurerm_key_vault.main.id
}
```

---

## ขั้นตอนที่ 568: Database Outputs และ Complete Example

```hcl
# database-outputs.tf

output "sql_server_info" {
  value = {
    fqdn = azurerm_mssql_server.main.fully_qualified_domain_name
    id   = azurerm_mssql_server.main.id
    name = azurerm_mssql_server.main.name
  }
}

output "postgresql_info" {
  value = {
    fqdn = azurerm_postgresql_flexible_server.main.fqdn
    id   = azurerm_postgresql_flexible_server.main.id
    name = azurerm_postgresql_flexible_server.main.name
  }
}

output "cosmosdb_info" {
  value = {
    endpoint            = azurerm_cosmosdb_account.main.endpoint
    id                  = azurerm_cosmosdb_account.main.id
    connection_strings  = azurerm_cosmosdb_account.main.connection_strings
  }
  sensitive = true
}

output "redis_info" {
  value = {
    hostname   = azurerm_redis_cache.main.hostname
    ssl_port   = azurerm_redis_cache.main.ssl_port
    id         = azurerm_redis_cache.main.id
  }
}

# ============================================================
# Complete Database Setup Summary
# ============================================================
/*
Production Database Architecture:

1. Azure SQL Server (sql-myapp-prod-sea)
   - Private Endpoint: pe-sql-prod
   - TDE with CMK
   - AAD Authentication
   - Auditing to Storage + Log Analytics
   - Advanced Threat Protection

2. PostgreSQL Flexible (psql-myapp-prod-sea)
   - Private VNet injection (snet-postgresql)
   - Zone-redundant HA
   - AAD Authentication + Password
   - Automated backups: 30 days

3. MySQL Flexible (mysql-myapp-prod-sea)
   - Private VNet injection (snet-mysql)
   - Zone-redundant HA
   - Automated backups: 30 days

4. Cosmos DB (cosmos-myapp-prod-sea)
   - SQL API + MongoDB API
   - Multi-region: SEA + East Asia
   - Automatic failover
   - Private Endpoint

5. Redis Cache (redis-myapp-prod-sea)
   - Premium P1 (6 GB)
   - Private Endpoint
   - RDB backup to storage
   - Clustering enabled

Security:
- All databases use Private Endpoints or VNET injection
- No public internet access
- TDE/encryption at rest
- Connection strings stored in Key Vault
- AAD authentication preferred
*/
```

---

## Database SKU Comparison

### Azure SQL

| SKU | vCores | RAM | IOPS | Use Case |
|-----|--------|-----|------|----------|
| Basic | - | 2GB | 5 DTU | Dev/Test |
| S0-S3 | - | 2-10GB | 10-100 DTU | Small Apps |
| GP_Gen5_2 | 2 | 10.4GB | 1024 | Production |
| GP_Gen5_8 | 8 | 41.5GB | 4096 | Large Apps |
| BC_Gen5_4 | 4 | 20.7GB | 25600 | OLTP Critical |

### PostgreSQL Flexible

| SKU | vCores | RAM | Notes |
|-----|--------|-----|-------|
| B_Standard_B1ms | 1 | 2GB | Dev/Test |
| GP_Standard_D2s_v3 | 2 | 8GB | Small Production |
| GP_Standard_D4s_v3 | 4 | 16GB | Medium Production |
| MO_Standard_E4s_v3 | 4 | 32GB | Memory-intensive |

---

*จบ Part 057: Azure Database Services*  
*ต่อไป Part 058: GCP Provider Setup & Authentication*
