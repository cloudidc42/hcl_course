# Part 055: Azure AKS Kubernetes Service
## ขั้นตอนที่ 541-550: การสร้างและจัดการ AKS Cluster

---

## ขั้นตอนที่ 541: AKS Cluster พื้นฐาน

Azure Kubernetes Service (AKS) คือ managed Kubernetes service บน Azure ที่ลดภาระในการจัดการ control plane

```hcl
# aks-basic.tf

# Resource Group สำหรับ AKS
resource "azurerm_resource_group" "aks" {
  name     = "rg-aks-prod-sea"
  location = "Southeast Asia"
  tags = {
    Environment = "Production"
    ManagedBy   = "Terraform"
  }
}

# Log Analytics Workspace สำหรับ Container Insights
resource "azurerm_log_analytics_workspace" "aks" {
  name                = "law-aks-prod-sea"
  resource_group_name = azurerm_resource_group.aks.name
  location            = azurerm_resource_group.aks.location
  sku                 = "PerGB2018"
  retention_in_days   = 30
}

# AKS Cluster พื้นฐาน
resource "azurerm_kubernetes_cluster" "main" {
  name                = "aks-myapp-prod-sea"
  resource_group_name = azurerm_resource_group.aks.name
  location            = azurerm_resource_group.aks.location
  dns_prefix          = "aks-myapp-prod"
  kubernetes_version  = "1.28.3"
  
  # Default Node Pool
  default_node_pool {
    name                = "system"
    vm_size             = "Standard_D4s_v3"
    node_count          = 3
    min_count           = 3
    max_count           = 10
    enable_auto_scaling = true
    
    # OS Disk
    os_disk_size_gb      = 128
    os_disk_type         = "Managed"
    
    # Availability Zones
    zones = ["1", "2", "3"]
    
    # Subnet
    vnet_subnet_id = azurerm_subnet.aks_nodes.id
    
    # Node Labels
    node_labels = {
      "nodepool-type" = "system"
      "environment"   = "production"
    }
    
    # Taints (system pools often tainted)
    # only_critical_addons_enabled = true  # Taint system pool for critical addons only
    
    upgrade_settings {
      max_surge = "10%"
    }
  }
  
  # Identity (Managed Identity)
  identity {
    type = "SystemAssigned"
  }
  
  # Network Profile
  network_profile {
    network_plugin    = "azure"     # azure หรือ kubenet
    network_policy    = "azure"     # azure หรือ calico
    service_cidr      = "10.100.0.0/16"
    dns_service_ip    = "10.100.0.10"
    load_balancer_sku = "standard"
    outbound_type     = "loadBalancer"
  }
  
  # Azure AD Integration (RBAC)
  azure_active_directory_role_based_access_control {
    managed                = true
    azure_rbac_enabled     = true
    admin_group_object_ids = [var.aks_admin_group_id]
  }
  
  # Addons
  oms_agent {
    log_analytics_workspace_id = azurerm_log_analytics_workspace.aks.id
  }
  
  # Key Vault secrets store CSI driver
  key_vault_secrets_provider {
    secret_rotation_enabled  = true
    secret_rotation_interval = "2m"
  }
  
  # Auto Upgrade
  automatic_channel_upgrade = "stable"
  
  # Maintenance Window
  maintenance_window {
    allowed {
      day   = "Sunday"
      hours = [0, 1, 2, 3]  # 00:00-04:00 Sunday
    }
    not_allowed {
      start = "2024-12-20T00:00:00Z"
      end   = "2025-01-02T00:00:00Z"
    }
  }
  
  # SKU Tier
  sku_tier = "Standard"  # Free หรือ Standard (SLA)
  
  tags = {
    Environment = "Production"
    ManagedBy   = "Terraform"
  }
}
```

---

## ขั้นตอนที่ 542: Node Pools เพิ่มเติม

```hcl
# aks-node-pools.tf

# User Node Pool สำหรับ Application workloads
resource "azurerm_kubernetes_cluster_node_pool" "app" {
  name                  = "app"
  kubernetes_cluster_id = azurerm_kubernetes_cluster.main.id
  vm_size               = "Standard_D4s_v3"
  node_count            = 3
  min_count             = 2
  max_count             = 20
  enable_auto_scaling   = true
  
  os_disk_size_gb = 128
  os_disk_type    = "Ephemeral"  # Ephemeral disk สำหรับ performance
  os_type         = "Linux"
  
  vnet_subnet_id = azurerm_subnet.aks_nodes.id
  zones          = ["1", "2", "3"]
  
  node_labels = {
    "nodepool-type" = "user"
    "workload-type" = "application"
  }
  
  node_taints = []  # ไม่มี taint สำหรับ app pool
  
  upgrade_settings {
    max_surge = "1"
  }
  
  tags = local.common_tags
}

# GPU Node Pool สำหรับ ML workloads
resource "azurerm_kubernetes_cluster_node_pool" "gpu" {
  name                  = "gpu"
  kubernetes_cluster_id = azurerm_kubernetes_cluster.main.id
  vm_size               = "Standard_NC6s_v3"  # GPU VM
  node_count            = 1
  min_count             = 0  # Scale to 0 when not needed
  max_count             = 4
  enable_auto_scaling   = true
  
  os_disk_size_gb = 256
  
  vnet_subnet_id = azurerm_subnet.aks_nodes.id
  zones          = ["1"]  # GPU VMs อาจมีเฉพาะบาง zones
  
  node_labels = {
    "nodepool-type"          = "gpu"
    "accelerator"            = "nvidia"
    "kubernetes.azure.com/scalesetpriority" = "regular"
  }
  
  # Taint สำหรับ GPU nodes - เฉพาะ pods ที่ต้องการ GPU เท่านั้น
  node_taints = [
    "nvidia.com/gpu=:NoSchedule"
  ]
}

# Spot Instance Node Pool สำหรับ batch workloads
resource "azurerm_kubernetes_cluster_node_pool" "spot" {
  name                  = "spot"
  kubernetes_cluster_id = azurerm_kubernetes_cluster.main.id
  vm_size               = "Standard_D8s_v3"
  node_count            = 0
  min_count             = 0
  max_count             = 50
  enable_auto_scaling   = true
  
  # Spot configuration
  priority        = "Spot"
  eviction_policy = "Delete"
  spot_max_price  = -1
  
  vnet_subnet_id = azurerm_subnet.aks_nodes.id
  
  node_labels = {
    "nodepool-type" = "spot"
    "workload-type" = "batch"
    "kubernetes.azure.com/scalesetpriority" = "spot"
  }
  
  # Taints สำหรับ spot nodes
  node_taints = [
    "kubernetes.azure.com/scalesetpriority=spot:NoSchedule"
  ]
}

# Windows Node Pool
resource "azurerm_kubernetes_cluster_node_pool" "windows" {
  name                  = "win"
  kubernetes_cluster_id = azurerm_kubernetes_cluster.main.id
  vm_size               = "Standard_D4s_v3"
  node_count            = 2
  min_count             = 2
  max_count             = 10
  enable_auto_scaling   = true
  os_type               = "Windows"
  os_sku                = "Windows2022"
  
  vnet_subnet_id = azurerm_subnet.aks_nodes.id
  
  node_labels = {
    "nodepool-type" = "windows"
  }
  
  node_taints = [
    "os=windows:NoSchedule"
  ]
}
```

---

## ขั้นตอนที่ 543: Network Profiles

```hcl
# aks-networking.tf

# Subnet สำหรับ AKS nodes
resource "azurerm_subnet" "aks_nodes" {
  name                 = "snet-aks-nodes"
  resource_group_name  = azurerm_resource_group.network.name
  virtual_network_name = azurerm_virtual_network.main.name
  address_prefixes     = ["10.1.0.0/20"]  # /20 = 4096 IPs สำหรับ nodes
  
  service_endpoints = [
    "Microsoft.ContainerRegistry",
    "Microsoft.Storage",
    "Microsoft.KeyVault"
  ]
}

# Subnet สำหรับ pods (Azure CNI Overlay)
resource "azurerm_subnet" "aks_pods" {
  name                 = "snet-aks-pods"
  resource_group_name  = azurerm_resource_group.network.name
  virtual_network_name = azurerm_virtual_network.main.name
  address_prefixes     = ["10.2.0.0/16"]  # /16 = 65536 IPs สำหรับ pods
  
  delegation {
    name = "aks-delegation"
    service_delegation {
      name = "Microsoft.ContainerService/managedClusters"
      actions = [
        "Microsoft.Network/virtualNetworks/subnets/join/action"
      ]
    }
  }
}

# AKS พร้อม Azure CNI (Advanced Networking)
resource "azurerm_kubernetes_cluster" "azure_cni" {
  name                = "aks-cni-prod"
  resource_group_name = azurerm_resource_group.aks.name
  location            = azurerm_resource_group.aks.location
  dns_prefix          = "aks-cni-prod"
  
  default_node_pool {
    name                = "system"
    vm_size             = "Standard_D4s_v3"
    node_count          = 3
    enable_auto_scaling = true
    min_count           = 3
    max_count           = 10
    vnet_subnet_id      = azurerm_subnet.aks_nodes.id
    pod_subnet_id       = azurerm_subnet.aks_pods.id  # Azure CNI Overlay
  }
  
  identity {
    type = "SystemAssigned"
  }
  
  network_profile {
    network_plugin      = "azure"
    network_plugin_mode = "overlay"  # Azure CNI Overlay
    network_policy      = "cilium"   # Cilium network policy
    ebpf_data_plane     = "cilium"
    service_cidr        = "10.200.0.0/16"
    dns_service_ip      = "10.200.0.10"
  }
}

# AKS พร้อม kubenet (Basic Networking)
resource "azurerm_kubernetes_cluster" "kubenet" {
  name                = "aks-kubenet-prod"
  resource_group_name = azurerm_resource_group.aks.name
  location            = azurerm_resource_group.aks.location
  dns_prefix          = "aks-kubenet-prod"
  
  default_node_pool {
    name                = "system"
    vm_size             = "Standard_D2s_v3"
    node_count          = 3
    vnet_subnet_id      = azurerm_subnet.aks_nodes.id
  }
  
  identity {
    type = "SystemAssigned"
  }
  
  network_profile {
    network_plugin    = "kubenet"
    pod_cidr          = "10.244.0.0/16"   # Pod CIDR
    service_cidr      = "10.100.0.0/16"
    dns_service_ip    = "10.100.0.10"
  }
}
```

---

## ขั้นตอนที่ 544: RBAC และ Azure AD Integration

```hcl
# aks-rbac.tf

# AKS พร้อม Azure AD RBAC
resource "azurerm_kubernetes_cluster" "with_aad" {
  name                = "aks-aad-prod"
  resource_group_name = azurerm_resource_group.aks.name
  location            = azurerm_resource_group.aks.location
  dns_prefix          = "aks-aad-prod"
  
  default_node_pool {
    name       = "system"
    vm_size    = "Standard_D4s_v3"
    node_count = 3
  }
  
  identity {
    type = "UserAssigned"
    identity_ids = [azurerm_user_assigned_identity.aks.id]
  }
  
  # Azure AD RBAC
  azure_active_directory_role_based_access_control {
    managed                = true
    azure_rbac_enabled     = true
    
    # Admin groups
    admin_group_object_ids = [
      var.aks_admin_group_id
    ]
  }
}

# Azure AD Groups
data "azuread_group" "aks_admins" {
  display_name     = "AKS-Admins-Prod"
  security_enabled = true
}

data "azuread_group" "aks_devops" {
  display_name     = "AKS-DevOps-Prod"
  security_enabled = true
}

data "azuread_group" "aks_developers" {
  display_name     = "AKS-Developers-Prod"
  security_enabled = true
}

# Cluster Admin role to admin group
resource "azurerm_role_assignment" "aks_cluster_admin" {
  scope                = azurerm_kubernetes_cluster.main.id
  role_definition_name = "Azure Kubernetes Service Cluster Admin Role"
  principal_id         = data.azuread_group.aks_admins.object_id
}

# Cluster User role to DevOps
resource "azurerm_role_assignment" "aks_cluster_user" {
  scope                = azurerm_kubernetes_cluster.main.id
  role_definition_name = "Azure Kubernetes Service Cluster User Role"
  principal_id         = data.azuread_group.aks_devops.object_id
}

# RBAC Reader for developers
resource "azurerm_role_assignment" "aks_rbac_reader" {
  scope                = azurerm_kubernetes_cluster.main.id
  role_definition_name = "Azure Kubernetes Service RBAC Reader"
  principal_id         = data.azuread_group.aks_developers.object_id
}
```

---

## ขั้นตอนที่ 545: Azure Container Registry

```hcl
# acr.tf

# Azure Container Registry
resource "azurerm_container_registry" "main" {
  name                = "crMyAppProd001"  # no hyphens, globally unique
  resource_group_name = azurerm_resource_group.aks.name
  location            = azurerm_resource_group.aks.location
  sku                 = "Premium"  # Basic, Standard, Premium
  
  # Admin access (สำหรับ development เท่านั้น)
  admin_enabled = false  # ใช้ Managed Identity แทน
  
  # Geo-replication (Premium only)
  georeplications {
    location                = "East Asia"
    zone_redundancy_enabled = true
  }
  
  # Network rules
  network_rule_set {
    default_action = "Deny"
    
    ip_rule {
      action   = "Allow"
      ip_range = "203.0.113.0/24"  # Office IP range
    }
  }
  
  # Private endpoint support
  public_network_access_enabled = false
  
  # Encryption
  encryption {
    enabled            = true
    key_vault_key_id   = azurerm_key_vault_key.acr.id
    identity_client_id = azurerm_user_assigned_identity.acr.client_id
  }
  
  # Retention policy (Premium only)
  retention_policy {
    days    = 30
    enabled = true
  }
  
  # Trust policy (Content Trust)
  trust_policy {
    enabled = true
  }
  
  tags = local.common_tags
}

# Private endpoint สำหรับ ACR
resource "azurerm_private_endpoint" "acr" {
  name                = "pe-acr-prod"
  resource_group_name = azurerm_resource_group.network.name
  location            = azurerm_resource_group.network.location
  subnet_id           = azurerm_subnet.private_endpoints.id
  
  private_service_connection {
    name                           = "psc-acr"
    private_connection_resource_id = azurerm_container_registry.main.id
    subresource_names              = ["registry"]
    is_manual_connection           = false
  }
  
  private_dns_zone_group {
    name = "acr-dns-group"
    private_dns_zone_ids = [azurerm_private_dns_zone.acr.id]
  }
}

# Grant AKS permission to pull from ACR
resource "azurerm_role_assignment" "aks_acr_pull" {
  scope                = azurerm_container_registry.main.id
  role_definition_name = "AcrPull"
  principal_id         = azurerm_kubernetes_cluster.main.kubelet_identity[0].object_id
}

# Grant specific identity AcrPush for CI/CD
resource "azurerm_role_assignment" "cicd_acr_push" {
  scope                = azurerm_container_registry.main.id
  role_definition_name = "AcrPush"
  principal_id         = var.cicd_service_principal_object_id
}
```

---

## ขั้นตอนที่ 546: Application Gateway Ingress Controller (AGIC)

```hcl
# agic.tf

# Public IP สำหรับ Application Gateway
resource "azurerm_public_ip" "agw" {
  name                = "pip-agw-aks-prod"
  resource_group_name = azurerm_resource_group.aks.name
  location            = azurerm_resource_group.aks.location
  allocation_method   = "Static"
  sku                 = "Standard"
  zones               = ["1", "2", "3"]
}

# Subnet สำหรับ Application Gateway
resource "azurerm_subnet" "agw" {
  name                 = "snet-agw"
  resource_group_name  = azurerm_resource_group.network.name
  virtual_network_name = azurerm_virtual_network.main.name
  address_prefixes     = ["10.0.10.0/24"]
}

# Application Gateway
resource "azurerm_application_gateway" "aks" {
  name                = "agw-aks-prod"
  resource_group_name = azurerm_resource_group.aks.name
  location            = azurerm_resource_group.aks.location
  
  sku {
    name     = "WAF_v2"
    tier     = "WAF_v2"
    capacity = 2
  }
  
  # Zones
  zones = ["1", "2", "3"]
  
  gateway_ip_configuration {
    name      = "gateway-ip-config"
    subnet_id = azurerm_subnet.agw.id
  }
  
  frontend_port {
    name = "http"
    port = 80
  }
  
  frontend_port {
    name = "https"
    port = 443
  }
  
  frontend_ip_configuration {
    name                 = "public"
    public_ip_address_id = azurerm_public_ip.agw.id
  }
  
  # Managed identity for SSL cert from Key Vault
  identity {
    type = "UserAssigned"
    identity_ids = [azurerm_user_assigned_identity.agw.id]
  }
  
  ssl_certificate {
    name                = "wildcard-cert"
    key_vault_secret_id = azurerm_key_vault_certificate.wildcard.secret_id
  }
  
  backend_address_pool {
    name = "default-backend"
  }
  
  backend_http_settings {
    name                  = "default-http-settings"
    cookie_based_affinity = "Disabled"
    protocol              = "Http"
    port                  = 80
    request_timeout       = 60
  }
  
  http_listener {
    name                           = "http-listener"
    frontend_ip_configuration_name = "public"
    frontend_port_name             = "http"
    protocol                       = "Http"
  }
  
  request_routing_rule {
    name                       = "default-rule"
    rule_type                  = "Basic"
    priority                   = 100
    http_listener_name         = "http-listener"
    backend_address_pool_name  = "default-backend"
    backend_http_settings_name = "default-http-settings"
  }
  
  # WAF Configuration
  waf_configuration {
    enabled          = true
    firewall_mode    = "Prevention"
    rule_set_type    = "OWASP"
    rule_set_version = "3.2"
    
    disabled_rule_group {
      rule_group_name = "REQUEST-931-APPLICATION-ATTACK-RFI"
    }
    
    file_upload_limit_mb     = 100
    request_body_check       = true
    max_request_body_size_kb = 128
  }
}

# AKS พร้อม AGIC add-on
resource "azurerm_kubernetes_cluster" "with_agic" {
  name                = "aks-agic-prod"
  resource_group_name = azurerm_resource_group.aks.name
  location            = azurerm_resource_group.aks.location
  dns_prefix          = "aks-agic-prod"
  
  default_node_pool {
    name                = "system"
    vm_size             = "Standard_D4s_v3"
    node_count          = 3
    vnet_subnet_id      = azurerm_subnet.aks_nodes.id
  }
  
  identity {
    type = "SystemAssigned"
  }
  
  network_profile {
    network_plugin = "azure"
    network_policy = "azure"
    service_cidr   = "10.100.0.0/16"
    dns_service_ip = "10.100.0.10"
  }
  
  # AGIC Add-on
  ingress_application_gateway {
    gateway_id = azurerm_application_gateway.aks.id
  }
}
```

---

## ขั้นตอนที่ 547: Monitoring - Container Insights

```hcl
# aks-monitoring.tf

# Log Analytics Workspace
resource "azurerm_log_analytics_workspace" "aks" {
  name                = "law-aks-prod-sea"
  resource_group_name = azurerm_resource_group.aks.name
  location            = azurerm_resource_group.aks.location
  sku                 = "PerGB2018"
  retention_in_days   = 90
}

# Log Analytics Solution - ContainerInsights
resource "azurerm_log_analytics_solution" "container_insights" {
  solution_name         = "ContainerInsights"
  workspace_resource_id = azurerm_log_analytics_workspace.aks.id
  workspace_name        = azurerm_log_analytics_workspace.aks.name
  resource_group_name   = azurerm_resource_group.aks.name
  location              = azurerm_resource_group.aks.location
  
  plan {
    publisher = "Microsoft"
    product   = "OMSGallery/ContainerInsights"
  }
}

# AKS พร้อม monitoring
resource "azurerm_kubernetes_cluster" "with_monitoring" {
  name                = "aks-monitored-prod"
  resource_group_name = azurerm_resource_group.aks.name
  location            = azurerm_resource_group.aks.location
  dns_prefix          = "aks-monitored"
  
  default_node_pool {
    name       = "system"
    vm_size    = "Standard_D4s_v3"
    node_count = 3
  }
  
  identity {
    type = "SystemAssigned"
  }
  
  # OMS Agent (Container Insights)
  oms_agent {
    log_analytics_workspace_id      = azurerm_log_analytics_workspace.aks.id
    msi_auth_for_monitoring_enabled = true
  }
  
  # Prometheus monitoring
  monitor_metrics {
    annotations_allowed = null
    labels_allowed      = null
  }
}

# Azure Monitor workspace สำหรับ Prometheus metrics
resource "azurerm_monitor_workspace" "aks" {
  name                = "amw-aks-prod"
  resource_group_name = azurerm_resource_group.aks.name
  location            = azurerm_resource_group.aks.location
}

# Alert Rules
resource "azurerm_monitor_metric_alert" "aks_node_cpu" {
  name                = "alert-aks-node-cpu-high"
  resource_group_name = azurerm_resource_group.aks.name
  scopes              = [azurerm_kubernetes_cluster.main.id]
  description         = "AKS node CPU usage is high"
  severity            = 2
  frequency           = "PT5M"
  window_size         = "PT15M"
  
  criteria {
    metric_namespace = "Microsoft.ContainerService/managedClusters"
    metric_name      = "node_cpu_usage_percentage"
    aggregation      = "Average"
    operator         = "GreaterThan"
    threshold        = 80
  }
  
  action {
    action_group_id = azurerm_monitor_action_group.aks_alerts.id
  }
}

resource "azurerm_monitor_metric_alert" "aks_pod_count" {
  name                = "alert-aks-pods-failing"
  resource_group_name = azurerm_resource_group.aks.name
  scopes              = [azurerm_kubernetes_cluster.main.id]
  description         = "AKS pods in failed state"
  severity            = 1
  frequency           = "PT5M"
  window_size         = "PT10M"
  
  criteria {
    metric_namespace = "Microsoft.ContainerService/managedClusters"
    metric_name      = "kube_pod_status_phase"
    aggregation      = "Average"
    operator         = "GreaterThan"
    threshold        = 0
    
    dimension {
      name     = "phase"
      operator = "Include"
      values   = ["Failed"]
    }
  }
  
  action {
    action_group_id = azurerm_monitor_action_group.aks_alerts.id
  }
}

resource "azurerm_monitor_action_group" "aks_alerts" {
  name                = "ag-aks-alerts"
  resource_group_name = azurerm_resource_group.aks.name
  short_name          = "aks-alerts"
  
  email_receiver {
    name          = "platform-team"
    email_address = "platform@company.com"
  }
  
  webhook_receiver {
    name        = "pagerduty"
    service_uri = var.pagerduty_webhook_url
  }
}
```

---

## ขั้นตอนที่ 548: Key Vault Secrets Store CSI Driver

```hcl
# aks-keyvault-csi.tf

# Key Vault
resource "azurerm_key_vault" "aks" {
  name                = "kv-aks-prod-sea"
  resource_group_name = azurerm_resource_group.aks.name
  location            = azurerm_resource_group.aks.location
  tenant_id           = data.azurerm_client_config.current.tenant_id
  sku_name            = "standard"
  
  # Enable RBAC authorization
  enable_rbac_authorization = true
  
  # Soft delete
  soft_delete_retention_days = 90
  purge_protection_enabled   = true
}

# Secrets
resource "azurerm_key_vault_secret" "db_password" {
  name         = "db-password"
  value        = random_password.db.result
  key_vault_id = azurerm_key_vault.aks.id
}

resource "azurerm_key_vault_secret" "api_key" {
  name         = "external-api-key"
  value        = var.external_api_key
  key_vault_id = azurerm_key_vault.aks.id
}

# Grant AKS kubelet identity access to Key Vault
resource "azurerm_role_assignment" "aks_kv_reader" {
  scope                = azurerm_key_vault.aks.id
  role_definition_name = "Key Vault Secrets User"
  principal_id         = azurerm_kubernetes_cluster.main.kubelet_identity[0].object_id
}
```

```yaml
# Kubernetes SecretProviderClass (ใช้ใน Kubernetes manifests)
# secret-provider-class.yaml
apiVersion: secrets-store.csi.x-k8s.io/v1
kind: SecretProviderClass
metadata:
  name: azure-kvname
  namespace: default
spec:
  provider: azure
  parameters:
    usePodIdentity: "false"
    useVMManagedIdentity: "true"
    userAssignedIdentityID: ""  # Leave empty for system-assigned
    keyvaultName: "kv-aks-prod-sea"
    objects: |
      array:
        - |
          objectName: db-password
          objectType: secret
          objectVersion: ""
        - |
          objectName: external-api-key
          objectType: secret
    tenantId: "YOUR-TENANT-ID"
  secretObjects:
  - secretName: app-secrets
    type: Opaque
    data:
    - key: DB_PASSWORD
      objectName: db-password
    - key: API_KEY
      objectName: external-api-key
```

---

## ขั้นตอนที่ 549: Workload Identity

```hcl
# workload-identity.tf

# User Assigned Managed Identity สำหรับ workload
resource "azurerm_user_assigned_identity" "app_workload" {
  name                = "mi-app-workload-prod"
  resource_group_name = azurerm_resource_group.aks.name
  location            = azurerm_resource_group.aks.location
}

# Federated Credentials สำหรับ Workload Identity
resource "azurerm_federated_identity_credential" "app_workload" {
  name                = "fic-app-workload"
  resource_group_name = azurerm_resource_group.aks.name
  parent_id           = azurerm_user_assigned_identity.app_workload.id
  
  issuer    = azurerm_kubernetes_cluster.main.oidc_issuer_url
  subject   = "system:serviceaccount:default:app-service-account"
  audience  = ["api://AzureADTokenExchange"]
}

# Grant permissions to the workload identity
resource "azurerm_role_assignment" "app_kv_access" {
  scope                = azurerm_key_vault.aks.id
  role_definition_name = "Key Vault Secrets User"
  principal_id         = azurerm_user_assigned_identity.app_workload.principal_id
}

resource "azurerm_role_assignment" "app_storage_access" {
  scope                = azurerm_storage_account.app.id
  role_definition_name = "Storage Blob Data Contributor"
  principal_id         = azurerm_user_assigned_identity.app_workload.principal_id
}

# AKS ต้องเปิด OIDC issuer และ Workload Identity
resource "azurerm_kubernetes_cluster" "workload_identity" {
  name                = "aks-wi-prod"
  resource_group_name = azurerm_resource_group.aks.name
  location            = azurerm_resource_group.aks.location
  dns_prefix          = "aks-wi-prod"
  
  default_node_pool {
    name       = "system"
    vm_size    = "Standard_D4s_v3"
    node_count = 3
  }
  
  identity {
    type = "SystemAssigned"
  }
  
  # Enable OIDC Issuer
  oidc_issuer_enabled       = true
  
  # Enable Workload Identity
  workload_identity_enabled = true
}
```

```yaml
# Kubernetes ServiceAccount สำหรับ Workload Identity
# service-account.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: app-service-account
  namespace: default
  annotations:
    azure.workload.identity/client-id: "USER-ASSIGNED-IDENTITY-CLIENT-ID"
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
        azure.workload.identity/use: "true"  # Required label
    spec:
      serviceAccountName: app-service-account
      containers:
      - name: myapp
        image: crMyAppProd001.azurecr.io/myapp:latest
        env:
        - name: AZURE_CLIENT_ID
          value: "USER-ASSIGNED-IDENTITY-CLIENT-ID"
```

---

## ขั้นตอนที่ 550: Complete AKS Production Setup

```hcl
# aks-production-complete.tf

terraform {
  required_providers {
    azurerm = {
      source  = "hashicorp/azurerm"
      version = "~> 3.80"
    }
    azuread = {
      source  = "hashicorp/azuread"
      version = "~> 2.45"
    }
  }
}

# Complete AKS Production Cluster
resource "azurerm_kubernetes_cluster" "production" {
  name                = "aks-myapp-prod-sea"
  resource_group_name = azurerm_resource_group.aks.name
  location            = azurerm_resource_group.aks.location
  dns_prefix          = "aks-myapp-prod"
  kubernetes_version  = "1.28.3"
  
  # SLA tier
  sku_tier = "Standard"
  
  # Default System Node Pool
  default_node_pool {
    name                         = "system"
    vm_size                      = "Standard_D4s_v3"
    node_count                   = 3
    min_count                    = 3
    max_count                    = 10
    enable_auto_scaling          = true
    only_critical_addons_enabled = true  # System pool for critical addons
    
    os_disk_size_gb = 128
    os_disk_type    = "Ephemeral"
    
    vnet_subnet_id = azurerm_subnet.aks_nodes.id
    zones          = ["1", "2", "3"]
    
    node_labels = {
      "nodepool-type" = "system"
      "environment"   = "production"
    }
    
    upgrade_settings {
      max_surge = "33%"
    }
  }
  
  # User-Assigned Managed Identity
  identity {
    type         = "UserAssigned"
    identity_ids = [azurerm_user_assigned_identity.aks.id]
  }
  
  # Kubelet identity for ACR pull
  kubelet_identity {
    client_id                 = azurerm_user_assigned_identity.aks_kubelet.client_id
    object_id                 = azurerm_user_assigned_identity.aks_kubelet.principal_id
    user_assigned_identity_id = azurerm_user_assigned_identity.aks_kubelet.id
  }
  
  # Azure CNI Networking
  network_profile {
    network_plugin    = "azure"
    network_policy    = "azure"
    service_cidr      = "10.100.0.0/16"
    dns_service_ip    = "10.100.0.10"
    load_balancer_sku = "standard"
    outbound_type     = "userDefinedRouting"
  }
  
  # Azure AD RBAC
  azure_active_directory_role_based_access_control {
    managed                = true
    azure_rbac_enabled     = true
    admin_group_object_ids = [var.aks_admin_group_id]
  }
  
  # OIDC & Workload Identity
  oidc_issuer_enabled       = true
  workload_identity_enabled = true
  
  # Container Insights
  oms_agent {
    log_analytics_workspace_id      = azurerm_log_analytics_workspace.aks.id
    msi_auth_for_monitoring_enabled = true
  }
  
  # Azure Monitor Metrics (Managed Prometheus)
  monitor_metrics {
    annotations_allowed = null
    labels_allowed      = null
  }
  
  # Key Vault CSI Driver
  key_vault_secrets_provider {
    secret_rotation_enabled  = true
    secret_rotation_interval = "2m"
  }
  
  # Auto upgrade
  automatic_channel_upgrade = "patch"
  node_os_channel_upgrade   = "NodeImage"
  
  # Maintenance
  maintenance_window_auto_upgrade {
    frequency   = "Weekly"
    interval    = 1
    duration    = 4
    day_of_week = "Sunday"
    start_time  = "00:00"
    utc_offset  = "+07:00"
    
    not_allowed {
      start = "2024-12-22T00:00:00Z"
      end   = "2025-01-02T00:00:00Z"
    }
  }
  
  maintenance_window_node_os {
    frequency   = "Weekly"
    interval    = 1
    duration    = 4
    day_of_week = "Saturday"
    start_time  = "00:00"
    utc_offset  = "+07:00"
  }
  
  # Storage
  storage_profile {
    blob_driver_enabled         = true
    disk_driver_enabled         = true
    file_driver_enabled         = true
    snapshot_controller_enabled = true
  }
  
  tags = local.common_tags
  
  lifecycle {
    ignore_changes = [
      default_node_pool[0].node_count,  # Managed by autoscaler
      kubernetes_version,               # Managed by auto-upgrade
    ]
  }
}

# Outputs
output "aks_cluster_id" {
  value = azurerm_kubernetes_cluster.production.id
}

output "aks_kubeconfig" {
  value     = azurerm_kubernetes_cluster.production.kube_config_raw
  sensitive = true
}

output "aks_oidc_issuer_url" {
  value = azurerm_kubernetes_cluster.production.oidc_issuer_url
}

output "aks_kubelet_identity" {
  value = azurerm_kubernetes_cluster.production.kubelet_identity[0].object_id
}
```

---

## AKS Node Pool Sizing Guide

| Workload | VM Size | Min Nodes | Max Nodes | Notes |
|----------|---------|-----------|-----------|-------|
| System | Standard_D4s_v3 | 3 | 10 | Critical addons only |
| Web/API | Standard_D4s_v3 | 2 | 20 | General workloads |
| Data Processing | Standard_E8s_v3 | 2 | 10 | Memory intensive |
| ML/GPU | Standard_NC6s_v3 | 0 | 4 | Scale from 0 |
| Batch (Spot) | Standard_D8s_v3 | 0 | 50 | Cost saving |

---

*จบ Part 055: Azure AKS Kubernetes Service*  
*ต่อไป Part 056: Azure Storage Accounts*
