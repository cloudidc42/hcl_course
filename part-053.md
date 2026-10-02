# Part 053: Azure Virtual Networks
## ขั้นตอนที่ 521-530: การสร้างและจัดการ Azure Networking

---

## ขั้นตอนที่ 521: Virtual Network และ Subnet พื้นฐาน

```hcl
# virtual-network-basic.tf

resource "azurerm_resource_group" "network" {
  name     = "rg-network-prod-sea"
  location = "Southeast Asia"
  tags = {
    Environment = "Production"
    Purpose     = "Networking"
    ManagedBy   = "Terraform"
  }
}

# Virtual Network หลัก
resource "azurerm_virtual_network" "main" {
  name                = "vnet-myapp-prod-sea"
  resource_group_name = azurerm_resource_group.network.name
  location            = azurerm_resource_group.network.location
  address_space       = ["10.0.0.0/16"]
  
  # DNS servers (ใช้ Azure DNS ถ้าไม่กำหนด)
  dns_servers = ["10.0.0.4", "10.0.0.5"]  # Custom DNS (optional)
  
  tags = {
    Environment = "Production"
    ManagedBy   = "Terraform"
  }
}

# Subnets สำหรับ 3-tier architecture
# Tier 1: Web/Frontend
resource "azurerm_subnet" "web" {
  name                 = "snet-web-prod"
  resource_group_name  = azurerm_resource_group.network.name
  virtual_network_name = azurerm_virtual_network.main.name
  address_prefixes     = ["10.0.1.0/24"]
  
  # Service Endpoints
  service_endpoints = [
    "Microsoft.Storage",
    "Microsoft.KeyVault",
    "Microsoft.Sql"
  ]
}

# Tier 2: Application/Backend
resource "azurerm_subnet" "app" {
  name                 = "snet-app-prod"
  resource_group_name  = azurerm_resource_group.network.name
  virtual_network_name = azurerm_virtual_network.main.name
  address_prefixes     = ["10.0.2.0/24"]
  
  service_endpoints = [
    "Microsoft.Storage",
    "Microsoft.KeyVault",
    "Microsoft.Sql",
    "Microsoft.ServiceBus",
    "Microsoft.EventHub"
  ]
  
  # Delegation สำหรับ services เช่น AKS, App Service
  delegation {
    name = "delegation"
    service_delegation {
      name    = "Microsoft.ContainerService/managedClusters"
      actions = ["Microsoft.Network/virtualNetworks/subnets/join/action"]
    }
  }
}

# Tier 3: Data
resource "azurerm_subnet" "data" {
  name                 = "snet-data-prod"
  resource_group_name  = azurerm_resource_group.network.name
  virtual_network_name = azurerm_virtual_network.main.name
  address_prefixes     = ["10.0.3.0/24"]
  
  service_endpoints = [
    "Microsoft.Storage",
    "Microsoft.Sql"
  ]
}

# Management Subnet
resource "azurerm_subnet" "management" {
  name                 = "snet-mgmt-prod"
  resource_group_name  = azurerm_resource_group.network.name
  virtual_network_name = azurerm_virtual_network.main.name
  address_prefixes     = ["10.0.4.0/24"]
}

# Private Endpoints Subnet
resource "azurerm_subnet" "private_endpoints" {
  name                 = "snet-pe-prod"
  resource_group_name  = azurerm_resource_group.network.name
  virtual_network_name = azurerm_virtual_network.main.name
  address_prefixes     = ["10.0.5.0/24"]
  
  # Private Endpoints ต้องการ disable private endpoint network policies
  private_endpoint_network_policies_enabled     = false
  private_link_service_network_policies_enabled = false
}

# Gateway Subnet (สำหรับ VPN/ExpressRoute)
resource "azurerm_subnet" "gateway" {
  name                 = "GatewaySubnet"  # ต้องใช้ชื่อนี้เสมอ
  resource_group_name  = azurerm_resource_group.network.name
  virtual_network_name = azurerm_virtual_network.main.name
  address_prefixes     = ["10.0.255.0/27"]
}

# Azure Firewall Subnet
resource "azurerm_subnet" "firewall" {
  name                 = "AzureFirewallSubnet"  # ต้องใช้ชื่อนี้เสมอ
  resource_group_name  = azurerm_resource_group.network.name
  virtual_network_name = azurerm_virtual_network.main.name
  address_prefixes     = ["10.0.254.0/26"]
}

# Bastion Subnet
resource "azurerm_subnet" "bastion" {
  name                 = "AzureBastionSubnet"  # ต้องใช้ชื่อนี้เสมอ
  resource_group_name  = azurerm_resource_group.network.name
  virtual_network_name = azurerm_virtual_network.main.name
  address_prefixes     = ["10.0.253.0/27"]
}
```

---

## ขั้นตอนที่ 522: Network Security Groups

```hcl
# network-security-groups.tf

# NSG สำหรับ Web Tier
resource "azurerm_network_security_group" "web" {
  name                = "nsg-web-prod-sea"
  resource_group_name = azurerm_resource_group.network.name
  location            = azurerm_resource_group.network.location
  
  tags = {
    Environment = "Production"
    Purpose     = "Web Tier NSG"
    ManagedBy   = "Terraform"
  }
}

# Security Rules สำหรับ Web NSG
resource "azurerm_network_security_rule" "web_allow_https" {
  name                        = "Allow-HTTPS-Inbound"
  priority                    = 100
  direction                   = "Inbound"
  access                      = "Allow"
  protocol                    = "Tcp"
  source_port_range           = "*"
  destination_port_range      = "443"
  source_address_prefix       = "Internet"
  destination_address_prefix  = "VirtualNetwork"
  resource_group_name         = azurerm_resource_group.network.name
  network_security_group_name = azurerm_network_security_group.web.name
}

resource "azurerm_network_security_rule" "web_allow_http" {
  name                        = "Allow-HTTP-Inbound"
  priority                    = 110
  direction                   = "Inbound"
  access                      = "Allow"
  protocol                    = "Tcp"
  source_port_range           = "*"
  destination_port_range      = "80"
  source_address_prefix       = "Internet"
  destination_address_prefix  = "VirtualNetwork"
  resource_group_name         = azurerm_resource_group.network.name
  network_security_group_name = azurerm_network_security_group.web.name
}

resource "azurerm_network_security_rule" "web_allow_azure_lb" {
  name                        = "Allow-Azure-LB"
  priority                    = 120
  direction                   = "Inbound"
  access                      = "Allow"
  protocol                    = "*"
  source_port_range           = "*"
  destination_port_range      = "*"
  source_address_prefix       = "AzureLoadBalancer"
  destination_address_prefix  = "*"
  resource_group_name         = azurerm_resource_group.network.name
  network_security_group_name = azurerm_network_security_group.web.name
}

resource "azurerm_network_security_rule" "web_deny_all_inbound" {
  name                        = "Deny-All-Inbound"
  priority                    = 4096
  direction                   = "Inbound"
  access                      = "Deny"
  protocol                    = "*"
  source_port_range           = "*"
  destination_port_range      = "*"
  source_address_prefix       = "*"
  destination_address_prefix  = "*"
  resource_group_name         = azurerm_resource_group.network.name
  network_security_group_name = azurerm_network_security_group.web.name
}

# NSG สำหรับ App Tier
resource "azurerm_network_security_group" "app" {
  name                = "nsg-app-prod-sea"
  resource_group_name = azurerm_resource_group.network.name
  location            = azurerm_resource_group.network.location
  
  # Inline security rules (alternative to separate resources)
  security_rule {
    name                       = "Allow-From-Web-Tier"
    priority                   = 100
    direction                  = "Inbound"
    access                     = "Allow"
    protocol                   = "Tcp"
    source_port_range          = "*"
    destination_port_ranges    = ["8080", "8443"]
    source_address_prefix      = "10.0.1.0/24"  # Web subnet
    destination_address_prefix = "VirtualNetwork"
  }
  
  security_rule {
    name                       = "Allow-Management"
    priority                   = 110
    direction                  = "Inbound"
    access                     = "Allow"
    protocol                   = "Tcp"
    source_port_range          = "*"
    destination_port_range     = "22"
    source_address_prefix      = "10.0.4.0/24"  # Management subnet
    destination_address_prefix = "VirtualNetwork"
  }
  
  security_rule {
    name                       = "Deny-All-Inbound"
    priority                   = 4096
    direction                  = "Inbound"
    access                     = "Deny"
    protocol                   = "*"
    source_port_range          = "*"
    destination_port_range     = "*"
    source_address_prefix      = "*"
    destination_address_prefix = "*"
  }
  
  tags = {
    Environment = "Production"
    ManagedBy   = "Terraform"
  }
}

# NSG สำหรับ Data Tier
resource "azurerm_network_security_group" "data" {
  name                = "nsg-data-prod-sea"
  resource_group_name = azurerm_resource_group.network.name
  location            = azurerm_resource_group.network.location
  
  security_rule {
    name                       = "Allow-From-App-Tier-SQL"
    priority                   = 100
    direction                  = "Inbound"
    access                     = "Allow"
    protocol                   = "Tcp"
    source_port_range          = "*"
    destination_port_range     = "1433"  # SQL Server
    source_address_prefix      = "10.0.2.0/24"
    destination_address_prefix = "VirtualNetwork"
  }
  
  security_rule {
    name                       = "Allow-From-App-Tier-PG"
    priority                   = 110
    direction                  = "Inbound"
    access                     = "Allow"
    protocol                   = "Tcp"
    source_port_range          = "*"
    destination_port_range     = "5432"  # PostgreSQL
    source_address_prefix      = "10.0.2.0/24"
    destination_address_prefix = "VirtualNetwork"
  }
  
  security_rule {
    name                       = "Deny-All-Inbound"
    priority                   = 4096
    direction                  = "Inbound"
    access                     = "Deny"
    protocol                   = "*"
    source_port_range          = "*"
    destination_port_range     = "*"
    source_address_prefix      = "*"
    destination_address_prefix = "*"
  }
  
  tags = {
    Environment = "Production"
    ManagedBy   = "Terraform"
  }
}

# Associate NSG กับ Subnets
resource "azurerm_subnet_network_security_group_association" "web" {
  subnet_id                 = azurerm_subnet.web.id
  network_security_group_id = azurerm_network_security_group.web.id
}

resource "azurerm_subnet_network_security_group_association" "app" {
  subnet_id                 = azurerm_subnet.app.id
  network_security_group_id = azurerm_network_security_group.app.id
}

resource "azurerm_subnet_network_security_group_association" "data" {
  subnet_id                 = azurerm_subnet.data.id
  network_security_group_id = azurerm_network_security_group.data.id
}
```

---

## ขั้นตอนที่ 523: Route Tables

```hcl
# route-tables.tf

# Route Table สำหรับ Web Tier (traffic ผ่าน Firewall)
resource "azurerm_route_table" "web" {
  name                          = "rt-web-prod"
  resource_group_name           = azurerm_resource_group.network.name
  location                      = azurerm_resource_group.network.location
  disable_bgp_route_propagation = false
  
  tags = {
    Environment = "Production"
    ManagedBy   = "Terraform"
  }
}

# Route: ส่ง traffic ทั้งหมดผ่าน Azure Firewall
resource "azurerm_route" "web_to_firewall" {
  name                   = "route-to-firewall"
  resource_group_name    = azurerm_resource_group.network.name
  route_table_name       = azurerm_route_table.web.name
  address_prefix         = "0.0.0.0/0"
  next_hop_type          = "VirtualAppliance"
  next_hop_in_ip_address = "10.0.254.4"  # Azure Firewall private IP
}

# Route: Internet traffic ผ่าน Firewall
resource "azurerm_route" "internet_via_firewall" {
  name                   = "route-internet-via-fw"
  resource_group_name    = azurerm_resource_group.network.name
  route_table_name       = azurerm_route_table.web.name
  address_prefix         = "10.0.0.0/8"
  next_hop_type          = "VirtualAppliance"
  next_hop_in_ip_address = "10.0.254.4"
}

# Route Table สำหรับ App Tier
resource "azurerm_route_table" "app" {
  name                          = "rt-app-prod"
  resource_group_name           = azurerm_resource_group.network.name
  location                      = azurerm_resource_group.network.location
  disable_bgp_route_propagation = true  # ป้องกัน BGP routes
  
  tags = {
    Environment = "Production"
    ManagedBy   = "Terraform"
  }
}

resource "azurerm_route" "app_default" {
  name                   = "route-default"
  resource_group_name    = azurerm_resource_group.network.name
  route_table_name       = azurerm_route_table.app.name
  address_prefix         = "0.0.0.0/0"
  next_hop_type          = "VirtualAppliance"
  next_hop_in_ip_address = "10.0.254.4"
}

# Associate Route Tables กับ Subnets
resource "azurerm_subnet_route_table_association" "web" {
  subnet_id      = azurerm_subnet.web.id
  route_table_id = azurerm_route_table.web.id
}

resource "azurerm_subnet_route_table_association" "app" {
  subnet_id      = azurerm_subnet.app.id
  route_table_id = azurerm_route_table.app.id
}
```

---

## ขั้นตอนที่ 524: Public IP และ NAT Gateway

```hcl
# public-ip-nat.tf

# Public IP สำหรับ Load Balancer
resource "azurerm_public_ip" "lb" {
  name                = "pip-lb-prod-sea"
  resource_group_name = azurerm_resource_group.network.name
  location            = azurerm_resource_group.network.location
  allocation_method   = "Static"
  sku                 = "Standard"  # Standard required for zones
  
  # Availability Zones
  zones = ["1", "2", "3"]
  
  # Idle timeout
  idle_timeout_in_minutes = 4
  
  # Domain name label (creates DNS: <label>.<region>.cloudapp.azure.com)
  domain_name_label = "myapp-prod"
  
  tags = {
    Environment = "Production"
    ManagedBy   = "Terraform"
  }
}

# Public IP สำหรับ Application Gateway
resource "azurerm_public_ip" "app_gateway" {
  name                = "pip-agw-prod-sea"
  resource_group_name = azurerm_resource_group.network.name
  location            = azurerm_resource_group.network.location
  allocation_method   = "Static"
  sku                 = "Standard"
  zones               = ["1", "2", "3"]
  
  tags = {
    Environment = "Production"
    Purpose     = "Application Gateway"
    ManagedBy   = "Terraform"
  }
}

# Public IPs สำหรับ NAT Gateway
resource "azurerm_public_ip" "nat" {
  count = 2  # 2 IPs สำหรับ high availability
  
  name                = "pip-nat-prod-${count.index + 1}"
  resource_group_name = azurerm_resource_group.network.name
  location            = azurerm_resource_group.network.location
  allocation_method   = "Static"
  sku                 = "Standard"
  zones               = ["1"]
  
  tags = {
    Environment = "Production"
    Purpose     = "NAT Gateway"
    ManagedBy   = "Terraform"
  }
}

# Public IP Prefix (block ของ IPs)
resource "azurerm_public_ip_prefix" "nat_prefix" {
  name                = "ippre-nat-prod"
  resource_group_name = azurerm_resource_group.network.name
  location            = azurerm_resource_group.network.location
  prefix_length       = 31  # 2 IP addresses
  sku                 = "Standard"
  zones               = ["1"]
  
  tags = {
    Environment = "Production"
    ManagedBy   = "Terraform"
  }
}

# NAT Gateway
resource "azurerm_nat_gateway" "main" {
  name                    = "ng-prod-sea"
  resource_group_name     = azurerm_resource_group.network.name
  location                = azurerm_resource_group.network.location
  sku_name                = "Standard"
  idle_timeout_in_minutes = 10
  zones                   = ["1"]
  
  tags = {
    Environment = "Production"
    ManagedBy   = "Terraform"
  }
}

# Associate Public IPs กับ NAT Gateway
resource "azurerm_nat_gateway_public_ip_association" "main" {
  count = 2
  
  nat_gateway_id       = azurerm_nat_gateway.main.id
  public_ip_address_id = azurerm_public_ip.nat[count.index].id
}

# Associate Public IP Prefix กับ NAT Gateway
resource "azurerm_nat_gateway_public_ip_prefix_association" "main" {
  nat_gateway_id      = azurerm_nat_gateway.main.id
  public_ip_prefix_id = azurerm_public_ip_prefix.nat_prefix.id
}

# Associate NAT Gateway กับ Subnets
resource "azurerm_subnet_nat_gateway_association" "web" {
  subnet_id      = azurerm_subnet.web.id
  nat_gateway_id = azurerm_nat_gateway.main.id
}

resource "azurerm_subnet_nat_gateway_association" "app" {
  subnet_id      = azurerm_subnet.app.id
  nat_gateway_id = azurerm_nat_gateway.main.id
}
```

---

## ขั้นตอนที่ 525: VNet Peering

```hcl
# vnet-peering.tf

# Hub VNet (ใน Hub-Spoke topology)
resource "azurerm_virtual_network" "hub" {
  name                = "vnet-hub-shared"
  resource_group_name = azurerm_resource_group.network_hub.name
  location            = azurerm_resource_group.network_hub.location
  address_space       = ["10.100.0.0/16"]
}

# Spoke VNet 1 - Production
resource "azurerm_virtual_network" "spoke_prod" {
  name                = "vnet-spoke-prod"
  resource_group_name = azurerm_resource_group.network_prod.name
  location            = azurerm_resource_group.network_prod.location
  address_space       = ["10.1.0.0/16"]
}

# Spoke VNet 2 - Development
resource "azurerm_virtual_network" "spoke_dev" {
  name                = "vnet-spoke-dev"
  resource_group_name = azurerm_resource_group.network_dev.name
  location            = azurerm_resource_group.network_dev.location
  address_space       = ["10.2.0.0/16"]
}

# Peering: Hub -> Spoke Prod
resource "azurerm_virtual_network_peering" "hub_to_prod" {
  name                         = "peer-hub-to-prod"
  resource_group_name          = azurerm_resource_group.network_hub.name
  virtual_network_name         = azurerm_virtual_network.hub.name
  remote_virtual_network_id    = azurerm_virtual_network.spoke_prod.id
  
  allow_virtual_network_access = true
  allow_forwarded_traffic      = true
  allow_gateway_transit        = true  # Hub shares its gateway
  use_remote_gateways          = false
}

# Peering: Spoke Prod -> Hub
resource "azurerm_virtual_network_peering" "prod_to_hub" {
  name                         = "peer-prod-to-hub"
  resource_group_name          = azurerm_resource_group.network_prod.name
  virtual_network_name         = azurerm_virtual_network.spoke_prod.name
  remote_virtual_network_id    = azurerm_virtual_network.hub.id
  
  allow_virtual_network_access = true
  allow_forwarded_traffic      = true
  allow_gateway_transit        = false
  use_remote_gateways          = true  # Use hub's gateway
  
  depends_on = [azurerm_virtual_network_peering.hub_to_prod]
}

# Peering: Hub -> Spoke Dev
resource "azurerm_virtual_network_peering" "hub_to_dev" {
  name                         = "peer-hub-to-dev"
  resource_group_name          = azurerm_resource_group.network_hub.name
  virtual_network_name         = azurerm_virtual_network.hub.name
  remote_virtual_network_id    = azurerm_virtual_network.spoke_dev.id
  
  allow_virtual_network_access = true
  allow_forwarded_traffic      = true
  allow_gateway_transit        = true
  use_remote_gateways          = false
}

# Peering: Spoke Dev -> Hub
resource "azurerm_virtual_network_peering" "dev_to_hub" {
  name                         = "peer-dev-to-hub"
  resource_group_name          = azurerm_resource_group.network_dev.name
  virtual_network_name         = azurerm_virtual_network.spoke_dev.name
  remote_virtual_network_id    = azurerm_virtual_network.hub.id
  
  allow_virtual_network_access = true
  allow_forwarded_traffic      = true
  allow_gateway_transit        = false
  use_remote_gateways          = true
  
  depends_on = [azurerm_virtual_network_peering.hub_to_dev]
}
```

---

## ขั้นตอนที่ 526: Private DNS Zone

```hcl
# private-dns.tf

# Private DNS Zone สำหรับ internal services
resource "azurerm_private_dns_zone" "internal" {
  name                = "internal.company.com"
  resource_group_name = azurerm_resource_group.network.name
  
  tags = {
    Environment = "Production"
    ManagedBy   = "Terraform"
  }
}

# Private DNS Zone สำหรับ Azure SQL
resource "azurerm_private_dns_zone" "sql" {
  name                = "privatelink.database.windows.net"
  resource_group_name = azurerm_resource_group.network.name
}

# Private DNS Zone สำหรับ Azure Storage (Blob)
resource "azurerm_private_dns_zone" "storage_blob" {
  name                = "privatelink.blob.core.windows.net"
  resource_group_name = azurerm_resource_group.network.name
}

# Private DNS Zone สำหรับ Azure Storage (File)
resource "azurerm_private_dns_zone" "storage_file" {
  name                = "privatelink.file.core.windows.net"
  resource_group_name = azurerm_resource_group.network.name
}

# Private DNS Zone สำหรับ Key Vault
resource "azurerm_private_dns_zone" "key_vault" {
  name                = "privatelink.vaultcore.azure.net"
  resource_group_name = azurerm_resource_group.network.name
}

# Private DNS Zone สำหรับ Azure Container Registry
resource "azurerm_private_dns_zone" "acr" {
  name                = "privatelink.azurecr.io"
  resource_group_name = azurerm_resource_group.network.name
}

# Private DNS Zone สำหรับ Cosmos DB
resource "azurerm_private_dns_zone" "cosmos" {
  name                = "privatelink.documents.azure.com"
  resource_group_name = azurerm_resource_group.network.name
}

# Link DNS Zones กับ VNet
resource "azurerm_private_dns_zone_virtual_network_link" "internal" {
  name                  = "link-internal-dns"
  resource_group_name   = azurerm_resource_group.network.name
  private_dns_zone_name = azurerm_private_dns_zone.internal.name
  virtual_network_id    = azurerm_virtual_network.main.id
  registration_enabled  = true  # Auto-register VM DNS names
  
  tags = {
    ManagedBy = "Terraform"
  }
}

resource "azurerm_private_dns_zone_virtual_network_link" "sql" {
  name                  = "link-sql-dns"
  resource_group_name   = azurerm_resource_group.network.name
  private_dns_zone_name = azurerm_private_dns_zone.sql.name
  virtual_network_id    = azurerm_virtual_network.main.id
  registration_enabled  = false
}

resource "azurerm_private_dns_zone_virtual_network_link" "storage_blob" {
  name                  = "link-storage-blob-dns"
  resource_group_name   = azurerm_resource_group.network.name
  private_dns_zone_name = azurerm_private_dns_zone.storage_blob.name
  virtual_network_id    = azurerm_virtual_network.main.id
  registration_enabled  = false
}

resource "azurerm_private_dns_zone_virtual_network_link" "key_vault" {
  name                  = "link-kv-dns"
  resource_group_name   = azurerm_resource_group.network.name
  private_dns_zone_name = azurerm_private_dns_zone.key_vault.name
  virtual_network_id    = azurerm_virtual_network.main.id
  registration_enabled  = false
}

# Custom DNS Records
resource "azurerm_private_dns_a_record" "app_server" {
  name                = "appserver"
  zone_name           = azurerm_private_dns_zone.internal.name
  resource_group_name = azurerm_resource_group.network.name
  ttl                 = 300
  records             = ["10.0.2.10"]
}

resource "azurerm_private_dns_cname_record" "api" {
  name                = "api"
  zone_name           = azurerm_private_dns_zone.internal.name
  resource_group_name = azurerm_resource_group.network.name
  ttl                 = 300
  record              = "appserver.internal.company.com"
}
```

---

## ขั้นตอนที่ 527: Private Endpoints

```hcl
# private-endpoints.tf

# Private Endpoint สำหรับ Storage Account
resource "azurerm_private_endpoint" "storage_blob" {
  name                = "pe-storage-blob-prod"
  resource_group_name = azurerm_resource_group.network.name
  location            = azurerm_resource_group.network.location
  subnet_id           = azurerm_subnet.private_endpoints.id
  
  private_service_connection {
    name                           = "psc-storage-blob"
    private_connection_resource_id = azurerm_storage_account.main.id
    subresource_names              = ["blob"]
    is_manual_connection           = false
  }
  
  private_dns_zone_group {
    name = "storage-blob-dns-group"
    private_dns_zone_ids = [
      azurerm_private_dns_zone.storage_blob.id
    ]
  }
  
  tags = {
    Environment = "Production"
    ManagedBy   = "Terraform"
  }
}

# Private Endpoint สำหรับ Key Vault
resource "azurerm_private_endpoint" "key_vault" {
  name                = "pe-kv-prod"
  resource_group_name = azurerm_resource_group.network.name
  location            = azurerm_resource_group.network.location
  subnet_id           = azurerm_subnet.private_endpoints.id
  
  private_service_connection {
    name                           = "psc-key-vault"
    private_connection_resource_id = azurerm_key_vault.main.id
    subresource_names              = ["vault"]
    is_manual_connection           = false
  }
  
  private_dns_zone_group {
    name = "kv-dns-group"
    private_dns_zone_ids = [
      azurerm_private_dns_zone.key_vault.id
    ]
  }
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
    private_dns_zone_ids = [
      azurerm_private_dns_zone.sql.id
    ]
  }
}

# Private Endpoint สำหรับ Azure Container Registry
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
    private_dns_zone_ids = [
      azurerm_private_dns_zone.acr.id
    ]
  }
}
```

---

## ขั้นตอนที่ 528: Hub-Spoke Network Topology

```hcl
# hub-spoke-complete.tf
# Complete Hub-Spoke network architecture

locals {
  hub_address_space = "10.100.0.0/16"
  spokes = {
    prod = {
      address_space = "10.1.0.0/16"
      env           = "production"
    }
    dev = {
      address_space = "10.2.0.0/16"
      env           = "development"
    }
    test = {
      address_space = "10.3.0.0/16"
      env           = "testing"
    }
  }
}

# Hub Resource Group
resource "azurerm_resource_group" "hub" {
  name     = "rg-network-hub-shared"
  location = "Southeast Asia"
}

# Hub Virtual Network
resource "azurerm_virtual_network" "hub_network" {
  name                = "vnet-hub-shared"
  resource_group_name = azurerm_resource_group.hub.name
  location            = azurerm_resource_group.hub.location
  address_space       = [local.hub_address_space]
}

# Hub Subnets
resource "azurerm_subnet" "hub_gateway" {
  name                 = "GatewaySubnet"
  resource_group_name  = azurerm_resource_group.hub.name
  virtual_network_name = azurerm_virtual_network.hub_network.name
  address_prefixes     = ["10.100.255.0/27"]
}

resource "azurerm_subnet" "hub_firewall" {
  name                 = "AzureFirewallSubnet"
  resource_group_name  = azurerm_resource_group.hub.name
  virtual_network_name = azurerm_virtual_network.hub_network.name
  address_prefixes     = ["10.100.254.0/26"]
}

resource "azurerm_subnet" "hub_bastion" {
  name                 = "AzureBastionSubnet"
  resource_group_name  = azurerm_resource_group.hub.name
  virtual_network_name = azurerm_virtual_network.hub_network.name
  address_prefixes     = ["10.100.253.0/27"]
}

resource "azurerm_subnet" "hub_management" {
  name                 = "snet-management"
  resource_group_name  = azurerm_resource_group.hub.name
  virtual_network_name = azurerm_virtual_network.hub_network.name
  address_prefixes     = ["10.100.1.0/24"]
}

# Spoke Resource Groups and VNets
resource "azurerm_resource_group" "spoke" {
  for_each = local.spokes
  
  name     = "rg-network-spoke-${each.key}"
  location = "Southeast Asia"
}

resource "azurerm_virtual_network" "spoke" {
  for_each = local.spokes
  
  name                = "vnet-spoke-${each.key}"
  resource_group_name = azurerm_resource_group.spoke[each.key].name
  location            = azurerm_resource_group.spoke[each.key].location
  address_space       = [each.value.address_space]
}

# Spoke Subnets
resource "azurerm_subnet" "spoke_web" {
  for_each = local.spokes
  
  name                 = "snet-web"
  resource_group_name  = azurerm_resource_group.spoke[each.key].name
  virtual_network_name = azurerm_virtual_network.spoke[each.key].name
  address_prefixes = [
    cidrsubnet(each.value.address_space, 8, 1)  # /24 from /16
  ]
}

resource "azurerm_subnet" "spoke_app" {
  for_each = local.spokes
  
  name                 = "snet-app"
  resource_group_name  = azurerm_resource_group.spoke[each.key].name
  virtual_network_name = azurerm_virtual_network.spoke[each.key].name
  address_prefixes = [
    cidrsubnet(each.value.address_space, 8, 2)
  ]
}

# Hub-Spoke Peering
resource "azurerm_virtual_network_peering" "hub_to_spoke" {
  for_each = local.spokes
  
  name                         = "peer-hub-to-${each.key}"
  resource_group_name          = azurerm_resource_group.hub.name
  virtual_network_name         = azurerm_virtual_network.hub_network.name
  remote_virtual_network_id    = azurerm_virtual_network.spoke[each.key].id
  allow_virtual_network_access = true
  allow_forwarded_traffic      = true
  allow_gateway_transit        = true
}

resource "azurerm_virtual_network_peering" "spoke_to_hub" {
  for_each = local.spokes
  
  name                         = "peer-${each.key}-to-hub"
  resource_group_name          = azurerm_resource_group.spoke[each.key].name
  virtual_network_name         = azurerm_virtual_network.spoke[each.key].name
  remote_virtual_network_id    = azurerm_virtual_network.hub_network.id
  allow_virtual_network_access = true
  allow_forwarded_traffic      = true
  allow_gateway_transit        = false
  use_remote_gateways          = false  # Set to true if hub has gateway
  
  depends_on = [
    azurerm_virtual_network_peering.hub_to_spoke
  ]
}
```

---

## ขั้นตอนที่ 529: Azure Bastion

```hcl
# azure-bastion.tf

# Public IP สำหรับ Bastion
resource "azurerm_public_ip" "bastion" {
  name                = "pip-bastion-prod"
  resource_group_name = azurerm_resource_group.hub.name
  location            = azurerm_resource_group.hub.location
  allocation_method   = "Static"
  sku                 = "Standard"
}

# Azure Bastion Host
resource "azurerm_bastion_host" "main" {
  name                = "bastion-hub-prod"
  resource_group_name = azurerm_resource_group.hub.name
  location            = azurerm_resource_group.hub.location
  
  sku = "Standard"  # Basic หรือ Standard
  
  # Standard features
  tunneling_enabled       = true   # Native client tunneling
  file_copy_enabled       = true   # File copy via Bastion
  shareable_link_enabled  = false
  ip_connect_enabled      = true   # Connect by IP
  
  ip_configuration {
    name                 = "configuration"
    subnet_id            = azurerm_subnet.hub_bastion.id
    public_ip_address_id = azurerm_public_ip.bastion.id
  }
  
  tags = {
    Environment = "Production"
    ManagedBy   = "Terraform"
  }
}
```

---

## ขั้นตอนที่ 530: Complete Azure Network Example

```hcl
# complete-network.tf - Production-ready network configuration

terraform {
  required_providers {
    azurerm = {
      source  = "hashicorp/azurerm"
      version = "~> 3.80"
    }
  }
}

provider "azurerm" {
  features {}
}

locals {
  location = "Southeast Asia"
  env      = "prod"
  prefix   = "myapp-${local.env}"
  
  # Network Address Planning
  vnet_cidr    = "10.0.0.0/16"
  subnets = {
    web  = { cidr = "10.0.1.0/24", nsg = "web" }
    app  = { cidr = "10.0.2.0/24", nsg = "app" }
    data = { cidr = "10.0.3.0/24", nsg = "data" }
    pe   = { cidr = "10.0.5.0/24", nsg = null }
    mgmt = { cidr = "10.0.4.0/24", nsg = null }
  }
  
  common_tags = {
    Environment = local.env
    ManagedBy   = "Terraform"
    Project     = "myapp"
  }
}

# Resource Group
resource "azurerm_resource_group" "network" {
  name     = "rg-network-${local.prefix}"
  location = local.location
  tags     = local.common_tags
}

# Virtual Network
resource "azurerm_virtual_network" "main" {
  name                = "vnet-${local.prefix}"
  resource_group_name = azurerm_resource_group.network.name
  location            = azurerm_resource_group.network.location
  address_space       = [local.vnet_cidr]
  tags                = local.common_tags
}

# Subnets
resource "azurerm_subnet" "main" {
  for_each = local.subnets
  
  name                 = "snet-${each.key}"
  resource_group_name  = azurerm_resource_group.network.name
  virtual_network_name = azurerm_virtual_network.main.name
  address_prefixes     = [each.value.cidr]
}

# NSGs (only for subnets that have NSG defined)
resource "azurerm_network_security_group" "main" {
  for_each = {
    for k, v in local.subnets :
    k => v if v.nsg != null
  }
  
  name                = "nsg-${each.key}-${local.prefix}"
  resource_group_name = azurerm_resource_group.network.name
  location            = azurerm_resource_group.network.location
  tags                = local.common_tags
}

# NSG Associations
resource "azurerm_subnet_network_security_group_association" "main" {
  for_each = {
    for k, v in local.subnets :
    k => v if v.nsg != null
  }
  
  subnet_id                 = azurerm_subnet.main[each.key].id
  network_security_group_id = azurerm_network_security_group.main[each.key].id
}

# Outputs
output "vnet_id" {
  value = azurerm_virtual_network.main.id
}

output "subnet_ids" {
  value = {
    for k, v in azurerm_subnet.main :
    k => v.id
  }
}

output "nsg_ids" {
  value = {
    for k, v in azurerm_network_security_group.main :
    k => v.id
  }
}
```

---

## Network Address Planning Table

| Subnet | CIDR | Purpose | NSG | Notes |
|--------|------|---------|-----|-------|
| Web | 10.0.1.0/24 | Web servers, Load Balancers | Yes | Internet-facing |
| App | 10.0.2.0/24 | Application servers | Yes | Internal only |
| Data | 10.0.3.0/24 | Databases | Yes | Restricted access |
| Management | 10.0.4.0/24 | Jump servers, management | Optional | Admin access |
| Private Endpoints | 10.0.5.0/24 | PaaS private endpoints | No | PE network policies off |
| GatewaySubnet | 10.0.255.0/27 | VPN/ExpressRoute | No | Fixed name required |
| AzureFirewallSubnet | 10.0.254.0/26 | Azure Firewall | No | Fixed name, min /26 |
| AzureBastionSubnet | 10.0.253.0/27 | Azure Bastion | No | Fixed name, min /27 |

---

*จบ Part 053: Azure Virtual Networks*  
*ต่อไป Part 054: Azure Virtual Machines*
