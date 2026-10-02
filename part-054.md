# Part 054: Azure Virtual Machines
## ขั้นตอนที่ 531-540: การสร้างและจัดการ Azure Virtual Machines

---

## ขั้นตอนที่ 531: Network Interface

Network Interface (NIC) คือ component แรกที่ต้องสร้างก่อน VM

```hcl
# network-interface.tf

# NIC พื้นฐาน
resource "azurerm_network_interface" "app_server" {
  name                = "nic-app-server-001"
  resource_group_name = azurerm_resource_group.compute.name
  location            = azurerm_resource_group.compute.location
  
  ip_configuration {
    name                          = "internal"
    subnet_id                     = azurerm_subnet.app.id
    private_ip_address_allocation = "Dynamic"
  }
  
  tags = local.common_tags
}

# NIC พร้อม Static Private IP
resource "azurerm_network_interface" "db_server" {
  name                = "nic-db-server-001"
  resource_group_name = azurerm_resource_group.compute.name
  location            = azurerm_resource_group.compute.location
  
  ip_configuration {
    name                          = "internal"
    subnet_id                     = azurerm_subnet.data.id
    private_ip_address_allocation = "Static"
    private_ip_address            = "10.0.3.10"
  }
}

# NIC พร้อม Public IP (สำหรับ development เท่านั้น)
resource "azurerm_network_interface" "dev_server" {
  name                = "nic-dev-server-001"
  resource_group_name = azurerm_resource_group.compute.name
  location            = azurerm_resource_group.compute.location
  
  ip_configuration {
    name                          = "internal"
    subnet_id                     = azurerm_subnet.web.id
    private_ip_address_allocation = "Dynamic"
    public_ip_address_id          = azurerm_public_ip.dev.id  # อย่าใช้ใน production!
  }
}

# NIC พร้อม Multiple IP configurations
resource "azurerm_network_interface" "multi_ip" {
  name                = "nic-multi-ip-001"
  resource_group_name = azurerm_resource_group.compute.name
  location            = azurerm_resource_group.compute.location
  
  # Primary IP
  ip_configuration {
    name                          = "primary"
    subnet_id                     = azurerm_subnet.app.id
    private_ip_address_allocation = "Static"
    private_ip_address            = "10.0.2.10"
    primary                       = true
  }
  
  # Secondary IP
  ip_configuration {
    name                          = "secondary"
    subnet_id                     = azurerm_subnet.app.id
    private_ip_address_allocation = "Static"
    private_ip_address            = "10.0.2.11"
    primary                       = false
  }
}

# Associate NSG กับ NIC (แทนที่ subnet)
resource "azurerm_network_interface_security_group_association" "app_server" {
  network_interface_id      = azurerm_network_interface.app_server.id
  network_security_group_id = azurerm_network_security_group.app.id
}
```

---

## ขั้นตอนที่ 532: Linux Virtual Machine

```hcl
# linux-vm.tf

# SSH Key
resource "azurerm_ssh_public_key" "admin" {
  name                = "ssh-key-admin"
  resource_group_name = azurerm_resource_group.compute.name
  location            = azurerm_resource_group.compute.location
  public_key          = file("~/.ssh/id_rsa.pub")
  
  tags = local.common_tags
}

# หรือสร้าง SSH key ด้วย Terraform
resource "tls_private_key" "vm_key" {
  algorithm = "RSA"
  rsa_bits  = 4096
}

resource "azurerm_ssh_public_key" "vm" {
  name                = "ssh-key-vm-prod"
  resource_group_name = azurerm_resource_group.compute.name
  location            = azurerm_resource_group.compute.location
  public_key          = tls_private_key.vm_key.public_key_openssh
}

# บันทึก private key
resource "local_sensitive_file" "private_key" {
  content         = tls_private_key.vm_key.private_key_pem
  filename        = "${path.module}/.ssh/vm_private_key.pem"
  file_permission = "0600"
}

# Linux VM - Ubuntu
resource "azurerm_linux_virtual_machine" "app_server" {
  name                = "vm-app-prod-001"
  resource_group_name = azurerm_resource_group.compute.name
  location            = azurerm_resource_group.compute.location
  size                = "Standard_D2s_v3"
  
  # Admin credentials
  admin_username                  = "adminuser"
  disable_password_authentication = true
  
  admin_ssh_key {
    username   = "adminuser"
    public_key = tls_private_key.vm_key.public_key_openssh
  }
  
  # Network
  network_interface_ids = [
    azurerm_network_interface.app_server.id
  ]
  
  # OS Disk
  os_disk {
    name                 = "osdisk-app-prod-001"
    caching              = "ReadWrite"
    storage_account_type = "Premium_LRS"
    disk_size_gb         = 128
  }
  
  # Source Image - Ubuntu 22.04 LTS
  source_image_reference {
    publisher = "Canonical"
    offer     = "0001-com-ubuntu-server-jammy"
    sku       = "22_04-lts-gen2"
    version   = "latest"
  }
  
  # Cloud-init (custom data)
  custom_data = base64encode(templatefile("${path.module}/cloud-init/app-server.yaml", {
    app_version = var.app_version
    db_host     = azurerm_postgresql_flexible_server.main.fqdn
  }))
  
  # Availability Zone
  zone = "1"
  
  # Managed Identity
  identity {
    type = "SystemAssigned"
  }
  
  # Boot diagnostics
  boot_diagnostics {
    storage_account_uri = azurerm_storage_account.diagnostics.primary_blob_endpoint
  }
  
  # Patch configuration
  patch_assessment_mode = "AutomaticByPlatform"
  patch_mode            = "AutomaticByPlatform"
  
  tags = merge(local.common_tags, {
    Role = "AppServer"
    OS   = "Ubuntu22.04"
  })
}

# Cloud-init example
# cloud-init/app-server.yaml
```

```yaml
# cloud-init/app-server.yaml
#cloud-config
package_update: true
package_upgrade: true

packages:
  - nginx
  - curl
  - jq
  - unzip
  - docker.io

runcmd:
  - systemctl enable nginx
  - systemctl start nginx
  - systemctl enable docker
  - systemctl start docker
  - usermod -aG docker adminuser
  - curl -o /tmp/app.zip "https://releases.example.com/app-${app_version}.zip"
  - unzip /tmp/app.zip -d /opt/app
  - systemctl start app

write_files:
  - path: /etc/app/config.env
    content: |
      DB_HOST=${db_host}
      DB_PORT=5432
      APP_ENV=production
    permissions: '0640'
```

---

## ขั้นตอนที่ 533: Windows Virtual Machine

```hcl
# windows-vm.tf

# Generate random password
resource "random_password" "windows_admin" {
  length           = 20
  special          = true
  override_special = "!@#$%&*()-_=+[]{}<>:?"
}

# Store password in Key Vault
resource "azurerm_key_vault_secret" "windows_admin_password" {
  name         = "vm-windows-admin-password"
  value        = random_password.windows_admin.result
  key_vault_id = azurerm_key_vault.main.id
}

# Windows Server VM
resource "azurerm_windows_virtual_machine" "iis_server" {
  name                = "vm-iis-prod-001"
  resource_group_name = azurerm_resource_group.compute.name
  location            = azurerm_resource_group.compute.location
  size                = "Standard_D4s_v3"
  
  admin_username = "adminuser"
  admin_password = random_password.windows_admin.result
  
  network_interface_ids = [
    azurerm_network_interface.iis_server.id
  ]
  
  os_disk {
    name                 = "osdisk-iis-prod-001"
    caching              = "ReadWrite"
    storage_account_type = "Premium_LRS"
    disk_size_gb         = 256
  }
  
  # Windows Server 2022
  source_image_reference {
    publisher = "MicrosoftWindowsServer"
    offer     = "WindowsServer"
    sku       = "2022-datacenter-azure-edition"
    version   = "latest"
  }
  
  # Time zone
  timezone = "SE Asia Standard Time"
  
  # Windows Update
  patch_mode            = "AutomaticByPlatform"
  enable_automatic_updates = true
  
  # Winrm listener (สำหรับ remote management)
  winrm_listener {
    protocol = "Http"
  }
  
  # Custom data (PowerShell script encoded as base64)
  custom_data = base64encode(<<-EOT
    <powershell>
    Install-WindowsFeature -name Web-Server -IncludeManagementTools
    Install-WindowsFeature -name NET-Framework-45-ASPNET
    Set-ExecutionPolicy Bypass -Scope Process -Force
    </powershell>
  EOT
  )
  
  identity {
    type = "SystemAssigned"
  }
  
  boot_diagnostics {
    storage_account_uri = azurerm_storage_account.diagnostics.primary_blob_endpoint
  }
  
  zone = "1"
  
  tags = merge(local.common_tags, {
    Role = "WebServer-IIS"
    OS   = "WindowsServer2022"
  })
}
```

---

## ขั้นตอนที่ 534: VM Sizes และ OS Disk

```hcl
# vm-sizes-reference.tf

locals {
  # General Purpose VM sizes
  vm_sizes = {
    # B-series (Burstable) - Dev/Test
    b1s  = "Standard_B1s"    # 1 vCPU, 1 GB RAM
    b2s  = "Standard_B2s"    # 2 vCPU, 4 GB RAM
    b4ms = "Standard_B4ms"   # 4 vCPU, 16 GB RAM
    
    # D-series (General Purpose)
    d2sv3  = "Standard_D2s_v3"   # 2 vCPU, 8 GB RAM
    d4sv3  = "Standard_D4s_v3"   # 4 vCPU, 16 GB RAM
    d8sv3  = "Standard_D8s_v3"   # 8 vCPU, 32 GB RAM
    d16sv3 = "Standard_D16s_v3"  # 16 vCPU, 64 GB RAM
    
    # E-series (Memory Optimized)
    e2sv3  = "Standard_E2s_v3"   # 2 vCPU, 16 GB RAM
    e4sv3  = "Standard_E4s_v3"   # 4 vCPU, 32 GB RAM
    e8sv3  = "Standard_E8s_v3"   # 8 vCPU, 64 GB RAM
    
    # F-series (Compute Optimized)
    f2sv2 = "Standard_F2s_v2"    # 2 vCPU, 4 GB RAM
    f4sv2 = "Standard_F4s_v2"    # 4 vCPU, 8 GB RAM
    
    # N-series (GPU)
    nc6  = "Standard_NC6"        # 6 vCPU, 56 GB RAM, K80 GPU
    nv6  = "Standard_NV6"        # 6 vCPU, 56 GB RAM, M60 GPU
  }
  
  # Storage Account Types
  disk_types = {
    standard_hdd  = "Standard_LRS"    # HDD
    standard_ssd  = "StandardSSD_LRS" # Standard SSD
    premium_ssd   = "Premium_LRS"     # Premium SSD
    ultra_ssd     = "UltraSSD_LRS"    # Ultra SSD (IOPS intensive)
  }
}

# VM พร้อม OS Disk configuration
resource "azurerm_linux_virtual_machine" "with_os_disk_config" {
  name                = "vm-config-demo"
  resource_group_name = azurerm_resource_group.compute.name
  location            = azurerm_resource_group.compute.location
  size                = "Standard_D4s_v3"
  admin_username      = "adminuser"
  
  admin_ssh_key {
    username   = "adminuser"
    public_key = tls_private_key.vm_key.public_key_openssh
  }
  
  network_interface_ids = [azurerm_network_interface.app_server.id]
  
  # OS Disk - detailed configuration
  os_disk {
    name                      = "osdisk-config-demo"
    caching                   = "ReadWrite"
    storage_account_type      = "Premium_LRS"
    disk_size_gb              = 128
    
    # Encryption at host (requires Azure feature registration)
    # security_encryption_type = "VMGuestStateOnly"
    
    # Write Accelerator (สำหรับ M-series เท่านั้น)
    write_accelerator_enabled = false
    
    # Disk Encryption Set (customer-managed keys)
    # disk_encryption_set_id = azurerm_disk_encryption_set.main.id
  }
  
  source_image_reference {
    publisher = "Canonical"
    offer     = "0001-com-ubuntu-server-jammy"
    sku       = "22_04-lts-gen2"
    version   = "latest"
  }
  
  # Ultra SSD support
  additional_capabilities {
    ultra_ssd_enabled = false
  }
}
```

---

## ขั้นตอนที่ 535: Managed Disks และ Data Disk Attachment

```hcl
# managed-disks.tf

# Managed Disk สำหรับ data
resource "azurerm_managed_disk" "app_data" {
  name                 = "disk-app-data-001"
  resource_group_name  = azurerm_resource_group.compute.name
  location             = azurerm_resource_group.compute.location
  storage_account_type = "Premium_LRS"
  create_option        = "Empty"
  disk_size_gb         = 512
  
  # Availability Zone (ต้องตรงกับ VM)
  zone = "1"
  
  # Encryption
  # disk_encryption_set_id = azurerm_disk_encryption_set.main.id
  
  tags = merge(local.common_tags, {
    Purpose = "Application Data"
  })
}

# สร้าง disk จาก snapshot
resource "azurerm_managed_disk" "from_snapshot" {
  name                 = "disk-from-snapshot-001"
  resource_group_name  = azurerm_resource_group.compute.name
  location             = azurerm_resource_group.compute.location
  storage_account_type = "Standard_LRS"
  create_option        = "Copy"
  source_resource_id   = azurerm_snapshot.app_disk.id
  disk_size_gb         = 512
}

# สร้าง disk จาก image
resource "azurerm_managed_disk" "from_image" {
  name                 = "disk-from-image-001"
  resource_group_name  = azurerm_resource_group.compute.name
  location             = azurerm_resource_group.compute.location
  storage_account_type = "Premium_LRS"
  create_option        = "FromImage"
  image_reference_id   = data.azurerm_image.custom.id
  disk_size_gb         = 128
}

# Attach Data Disk ให้ VM
resource "azurerm_virtual_machine_data_disk_attachment" "app_data" {
  managed_disk_id    = azurerm_managed_disk.app_data.id
  virtual_machine_id = azurerm_linux_virtual_machine.app_server.id
  lun                = 0  # Logical Unit Number (0-63)
  caching            = "ReadWrite"
  
  # Write Accelerator (M-series only)
  write_accelerator_enabled = false
}

# Attach multiple data disks
locals {
  data_disks = [
    { name = "data-1", size = 256, lun = 0, caching = "ReadWrite" },
    { name = "data-2", size = 512, lun = 1, caching = "ReadOnly" },
    { name = "logs",   size = 128, lun = 2, caching = "None" },
  ]
}

resource "azurerm_managed_disk" "multiple" {
  for_each = {
    for disk in local.data_disks :
    disk.name => disk
  }
  
  name                 = "disk-vm-${each.key}"
  resource_group_name  = azurerm_resource_group.compute.name
  location             = azurerm_resource_group.compute.location
  storage_account_type = "Premium_LRS"
  create_option        = "Empty"
  disk_size_gb         = each.value.size
  zone                 = "1"
}

resource "azurerm_virtual_machine_data_disk_attachment" "multiple" {
  for_each = {
    for disk in local.data_disks :
    disk.name => disk
  }
  
  managed_disk_id    = azurerm_managed_disk.multiple[each.key].id
  virtual_machine_id = azurerm_linux_virtual_machine.app_server.id
  lun                = each.value.lun
  caching            = each.value.caching
}

# Disk Snapshot
resource "azurerm_snapshot" "app_disk" {
  name                = "snapshot-app-disk-${formatdate("YYYY-MM-DD", timestamp())}"
  resource_group_name = azurerm_resource_group.compute.name
  location            = azurerm_resource_group.compute.location
  create_option       = "Copy"
  source_uri          = azurerm_managed_disk.app_data.id
  
  tags = merge(local.common_tags, {
    SnapshotDate = formatdate("YYYY-MM-DD", timestamp())
  })
}
```

---

## ขั้นตอนที่ 536: Availability Sets และ Zones

```hcl
# availability.tf

# Availability Set (Fault Domain protection)
resource "azurerm_availability_set" "web_servers" {
  name                         = "avset-web-prod"
  resource_group_name          = azurerm_resource_group.compute.name
  location                     = azurerm_resource_group.compute.location
  platform_fault_domain_count  = 2   # Max 3
  platform_update_domain_count = 5   # Max 20
  managed                      = true  # สำหรับ managed disks
  
  tags = local.common_tags
}

# VMs ใน Availability Set
resource "azurerm_linux_virtual_machine" "web_avset" {
  count = 2
  
  name                = "vm-web-avset-${count.index + 1}"
  resource_group_name = azurerm_resource_group.compute.name
  location            = azurerm_resource_group.compute.location
  size                = "Standard_D2s_v3"
  admin_username      = "adminuser"
  
  # Assign to Availability Set
  availability_set_id = azurerm_availability_set.web_servers.id
  
  admin_ssh_key {
    username   = "adminuser"
    public_key = tls_private_key.vm_key.public_key_openssh
  }
  
  network_interface_ids = [
    azurerm_network_interface.web[count.index].id
  ]
  
  os_disk {
    caching              = "ReadWrite"
    storage_account_type = "Premium_LRS"
  }
  
  source_image_reference {
    publisher = "Canonical"
    offer     = "0001-com-ubuntu-server-jammy"
    sku       = "22_04-lts-gen2"
    version   = "latest"
  }
}

# VMs ใน Availability Zones
resource "azurerm_linux_virtual_machine" "web_zones" {
  count = 3
  
  name                = "vm-web-zone${count.index + 1}"
  resource_group_name = azurerm_resource_group.compute.name
  location            = azurerm_resource_group.compute.location
  size                = "Standard_D2s_v3"
  admin_username      = "adminuser"
  
  # Each VM in different zone
  zone = tostring(count.index + 1)
  
  admin_ssh_key {
    username   = "adminuser"
    public_key = tls_private_key.vm_key.public_key_openssh
  }
  
  network_interface_ids = [
    azurerm_network_interface.web_zone[count.index].id
  ]
  
  os_disk {
    caching              = "ReadWrite"
    storage_account_type = "Premium_LRS"
  }
  
  source_image_reference {
    publisher = "Canonical"
    offer     = "0001-com-ubuntu-server-jammy"
    sku       = "22_04-lts-gen2"
    version   = "latest"
  }
}
```

---

## ขั้นตอนที่ 537: VM Extensions

```hcl
# vm-extensions.tf

# Custom Script Extension (Linux)
resource "azurerm_virtual_machine_extension" "custom_script_linux" {
  name                 = "CustomScript"
  virtual_machine_id   = azurerm_linux_virtual_machine.app_server.id
  publisher            = "Microsoft.Azure.Extensions"
  type                 = "CustomScript"
  type_handler_version = "2.1"
  
  settings = jsonencode({
    fileUris = [
      "https://stscripts.blob.core.windows.net/scripts/setup-app.sh"
    ]
    commandToExecute = "bash setup-app.sh"
  })
  
  protected_settings = jsonencode({
    storageAccountName = azurerm_storage_account.scripts.name
    storageAccountKey  = azurerm_storage_account.scripts.primary_access_key
  })
  
  tags = local.common_tags
}

# Azure Monitor Agent
resource "azurerm_virtual_machine_extension" "azure_monitor" {
  name                       = "AzureMonitorLinuxAgent"
  virtual_machine_id         = azurerm_linux_virtual_machine.app_server.id
  publisher                  = "Microsoft.Azure.Monitor"
  type                       = "AzureMonitorLinuxAgent"
  type_handler_version       = "1.0"
  auto_upgrade_minor_version = true
}

# Dependency Agent
resource "azurerm_virtual_machine_extension" "dependency_agent" {
  name                       = "DependencyAgentLinux"
  virtual_machine_id         = azurerm_linux_virtual_machine.app_server.id
  publisher                  = "Microsoft.Azure.Monitoring.DependencyAgent"
  type                       = "DependencyAgentLinux"
  type_handler_version       = "9.10"
  auto_upgrade_minor_version = true
}

# NVIDIA GPU Driver (สำหรับ GPU VMs)
resource "azurerm_virtual_machine_extension" "gpu_driver" {
  name                       = "NvidiaGpuDriverLinux"
  virtual_machine_id         = azurerm_linux_virtual_machine.gpu_server.id
  publisher                  = "Microsoft.HpcCompute"
  type                       = "NvidiaGpuDriverLinux"
  type_handler_version       = "1.9"
  auto_upgrade_minor_version = true
}

# Azure AD SSH Login (สำหรับ SSH ด้วย Azure AD credentials)
resource "azurerm_virtual_machine_extension" "aad_ssh" {
  name                       = "AADSSHLoginForLinux"
  virtual_machine_id         = azurerm_linux_virtual_machine.app_server.id
  publisher                  = "Microsoft.Azure.ActiveDirectory"
  type                       = "AADSSHLoginForLinux"
  type_handler_version       = "1.0"
  auto_upgrade_minor_version = true
}

# Network Watcher Agent
resource "azurerm_virtual_machine_extension" "network_watcher" {
  name                 = "NetworkWatcherAgentLinux"
  virtual_machine_id   = azurerm_linux_virtual_machine.app_server.id
  publisher            = "Microsoft.Azure.NetworkWatcher"
  type                 = "NetworkWatcherAgentLinux"
  type_handler_version = "1.4"
}
```

---

## ขั้นตอนที่ 538: Azure Spot VMs

```hcl
# spot-vms.tf

# Spot VM (ราคาถูกกว่า แต่อาจถูก evict)
resource "azurerm_linux_virtual_machine" "spot" {
  name                = "vm-spot-batch-001"
  resource_group_name = azurerm_resource_group.compute.name
  location            = azurerm_resource_group.compute.location
  size                = "Standard_D4s_v3"
  admin_username      = "adminuser"
  
  # Spot VM settings
  priority        = "Spot"
  eviction_policy = "Deallocate"  # หรือ "Delete"
  max_bid_price   = -1            # -1 = Pay up to on-demand price
  
  admin_ssh_key {
    username   = "adminuser"
    public_key = tls_private_key.vm_key.public_key_openssh
  }
  
  network_interface_ids = [azurerm_network_interface.spot.id]
  
  os_disk {
    caching              = "ReadWrite"
    storage_account_type = "Standard_LRS"  # ใช้ Standard สำหรับ cost savings
  }
  
  source_image_reference {
    publisher = "Canonical"
    offer     = "0001-com-ubuntu-server-jammy"
    sku       = "22_04-lts-gen2"
    version   = "latest"
  }
  
  # Custom data สำหรับ handle spot interruption
  custom_data = base64encode(<<-EOT
    #!/bin/bash
    # Register IMDS endpoint สำหรับ spot interruption notifications
    curl -H Metadata:true "http://169.254.169.254/metadata/scheduledevents?api-version=2020-07-01" &
  EOT
  )
  
  tags = merge(local.common_tags, {
    VMType = "Spot"
    Purpose = "Batch Processing"
  })
}
```

---

## ขั้นตอนที่ 539: Virtual Machine Scale Sets

```hcl
# vmss.tf

# Linux VMSS
resource "azurerm_linux_virtual_machine_scale_set" "web" {
  name                = "vmss-web-prod"
  resource_group_name = azurerm_resource_group.compute.name
  location            = azurerm_resource_group.compute.location
  sku                 = "Standard_D2s_v3"
  instances           = 2
  admin_username      = "adminuser"
  
  admin_ssh_key {
    username   = "adminuser"
    public_key = tls_private_key.vm_key.public_key_openssh
  }
  
  source_image_reference {
    publisher = "Canonical"
    offer     = "0001-com-ubuntu-server-jammy"
    sku       = "22_04-lts-gen2"
    version   = "latest"
  }
  
  os_disk {
    storage_account_type = "Premium_LRS"
    caching              = "ReadWrite"
  }
  
  # Data Disk Template
  data_disk {
    lun                  = 0
    caching              = "ReadWrite"
    create_option        = "Empty"
    disk_size_gb         = 128
    storage_account_type = "Premium_LRS"
  }
  
  network_interface {
    name    = "nic-vmss"
    primary = true
    
    ip_configuration {
      name                                   = "ipconfig"
      primary                                = true
      subnet_id                              = azurerm_subnet.web.id
      load_balancer_backend_address_pool_ids = [azurerm_lb_backend_address_pool.web.id]
    }
  }
  
  # Upgrade mode
  upgrade_mode = "RollingUpgrade"  # Manual, Automatic, Rolling
  
  rolling_upgrade_policy {
    max_batch_instance_percent              = 20
    max_unhealthy_instance_percent          = 20
    max_unhealthy_upgraded_instance_percent = 5
    pause_time_between_batches              = "PT0S"
    cross_zone_upgrades_enabled             = true
    prioritize_unhealthy_instances_enabled  = false
  }
  
  # Health probe
  health_probe_id = azurerm_lb_probe.http.id
  
  # Automatic OS image upgrades
  automatic_os_upgrade_policy {
    disable_automatic_rollback  = false
    enable_automatic_os_upgrade = true
  }
  
  # Autoscale (configured separately)
  # ...
  
  # Zones for HA
  zones = ["1", "2", "3"]
  zone_balance = true
  
  # Identity
  identity {
    type = "SystemAssigned"
  }
  
  # Boot diagnostics
  boot_diagnostics {
    storage_account_uri = azurerm_storage_account.diagnostics.primary_blob_endpoint
  }
  
  # Custom data
  custom_data = base64encode(file("${path.module}/cloud-init/web-server.yaml"))
  
  tags = local.common_tags
}

# Autoscale settings สำหรับ VMSS
resource "azurerm_monitor_autoscale_setting" "web_vmss" {
  name                = "autoscale-vmss-web-prod"
  resource_group_name = azurerm_resource_group.compute.name
  location            = azurerm_resource_group.compute.location
  target_resource_id  = azurerm_linux_virtual_machine_scale_set.web.id
  
  profile {
    name = "defaultProfile"
    
    capacity {
      default = 2
      minimum = 2
      maximum = 10
    }
    
    # Scale out when CPU > 70%
    rule {
      metric_trigger {
        metric_name        = "Percentage CPU"
        metric_resource_id = azurerm_linux_virtual_machine_scale_set.web.id
        time_grain         = "PT1M"
        statistic          = "Average"
        time_window        = "PT5M"
        time_aggregation   = "Average"
        operator           = "GreaterThan"
        threshold          = 70
      }
      
      scale_action {
        direction = "Increase"
        type      = "ChangeCount"
        value     = "1"
        cooldown  = "PT5M"
      }
    }
    
    # Scale in when CPU < 30%
    rule {
      metric_trigger {
        metric_name        = "Percentage CPU"
        metric_resource_id = azurerm_linux_virtual_machine_scale_set.web.id
        time_grain         = "PT1M"
        statistic          = "Average"
        time_window        = "PT10M"
        time_aggregation   = "Average"
        operator           = "LessThan"
        threshold          = 30
      }
      
      scale_action {
        direction = "Decrease"
        type      = "ChangeCount"
        value     = "1"
        cooldown  = "PT10M"
      }
    }
  }
  
  notification {
    email {
      send_to_subscription_administrator    = true
      send_to_subscription_co_administrator = false
      custom_emails                         = ["platform@company.com"]
    }
  }
}
```

---

## ขั้นตอนที่ 540: Custom Images และ Marketplace Images

```hcl
# custom-images.tf

# ============================================================
# Marketplace Images - รายชื่อ Images ที่ใช้บ่อย
# ============================================================

locals {
  images = {
    ubuntu_22_04 = {
      publisher = "Canonical"
      offer     = "0001-com-ubuntu-server-jammy"
      sku       = "22_04-lts-gen2"
      version   = "latest"
    }
    ubuntu_20_04 = {
      publisher = "Canonical"
      offer     = "0001-com-ubuntu-server-focal"
      sku       = "20_04-lts-gen2"
      version   = "latest"
    }
    rhel_9 = {
      publisher = "RedHat"
      offer     = "RHEL"
      sku       = "9-gen2"
      version   = "latest"
    }
    centos_7 = {
      publisher = "OpenLogic"
      offer     = "CentOS"
      sku       = "7.7"
      version   = "latest"
    }
    debian_11 = {
      publisher = "Debian"
      offer     = "debian-11"
      sku       = "11-gen2"
      version   = "latest"
    }
    windows_server_2022 = {
      publisher = "MicrosoftWindowsServer"
      offer     = "WindowsServer"
      sku       = "2022-datacenter-azure-edition"
      version   = "latest"
    }
    windows_server_2019 = {
      publisher = "MicrosoftWindowsServer"
      offer     = "WindowsServer"
      sku       = "2019-datacenter-gensecond"
      version   = "latest"
    }
    windows_11 = {
      publisher = "MicrosoftWindowsDesktop"
      offer     = "windows-11"
      sku       = "win11-22h2-pro"
      version   = "latest"
    }
  }
}

# ค้นหา Marketplace Images ด้วย data source
data "azurerm_platform_image" "ubuntu" {
  location  = "Southeast Asia"
  publisher = "Canonical"
  offer     = "0001-com-ubuntu-server-jammy"
  sku       = "22_04-lts-gen2"
}

output "ubuntu_image_version" {
  value = data.azurerm_platform_image.ubuntu.version
}

# Custom Image จาก Packer
data "azurerm_image" "custom_app" {
  name                = "img-app-server-v1-20240101"
  resource_group_name = "rg-images-shared"
}

resource "azurerm_linux_virtual_machine" "from_custom_image" {
  name                = "vm-app-custom-001"
  resource_group_name = azurerm_resource_group.compute.name
  location            = azurerm_resource_group.compute.location
  size                = "Standard_D2s_v3"
  admin_username      = "adminuser"
  
  admin_ssh_key {
    username   = "adminuser"
    public_key = tls_private_key.vm_key.public_key_openssh
  }
  
  network_interface_ids = [azurerm_network_interface.app_server.id]
  
  os_disk {
    caching              = "ReadWrite"
    storage_account_type = "Premium_LRS"
  }
  
  # ใช้ custom image
  source_image_id = data.azurerm_image.custom_app.id
}

# Shared Image Gallery (Azure Compute Gallery)
resource "azurerm_shared_image_gallery" "main" {
  name                = "sig_company_images"
  resource_group_name = azurerm_resource_group.shared.name
  location            = azurerm_resource_group.shared.location
  description         = "Company shared image gallery"
}

resource "azurerm_shared_image" "app_server" {
  name                = "img-app-server"
  gallery_name        = azurerm_shared_image_gallery.main.name
  resource_group_name = azurerm_resource_group.shared.name
  location            = azurerm_resource_group.shared.location
  os_type             = "Linux"
  
  identifier {
    publisher = "CompanyInternal"
    offer     = "AppServer"
    sku       = "Production"
  }
}

resource "azurerm_shared_image_version" "app_server_v1" {
  name                = "1.0.0"
  gallery_name        = azurerm_shared_image_gallery.main.name
  image_name          = azurerm_shared_image.app_server.name
  resource_group_name = azurerm_resource_group.shared.name
  location            = azurerm_resource_group.shared.location
  
  managed_image_id = data.azurerm_image.custom_app.id
  
  target_region {
    name                   = "Southeast Asia"
    regional_replica_count = 1
    storage_account_type   = "Standard_LRS"
  }
  
  target_region {
    name                   = "East Asia"
    regional_replica_count = 1
    storage_account_type   = "Standard_LRS"
  }
}

# VM จาก Shared Image Gallery
resource "azurerm_linux_virtual_machine" "from_gallery" {
  name                = "vm-from-gallery-001"
  resource_group_name = azurerm_resource_group.compute.name
  location            = azurerm_resource_group.compute.location
  size                = "Standard_D2s_v3"
  admin_username      = "adminuser"
  
  admin_ssh_key {
    username   = "adminuser"
    public_key = tls_private_key.vm_key.public_key_openssh
  }
  
  network_interface_ids = [azurerm_network_interface.app_server.id]
  
  os_disk {
    caching              = "ReadWrite"
    storage_account_type = "Premium_LRS"
  }
  
  # ใช้ image จาก gallery
  source_image_id = azurerm_shared_image_version.app_server_v1.id
}

# ============================================================
# Outputs
# ============================================================
output "vm_ids" {
  value = {
    app_server = azurerm_linux_virtual_machine.app_server.id
    iis_server = azurerm_windows_virtual_machine.iis_server.id
  }
}

output "vm_private_ips" {
  value = {
    app_server = azurerm_network_interface.app_server.private_ip_address
  }
}

output "vmss_id" {
  value = azurerm_linux_virtual_machine_scale_set.web.id
}

output "managed_identity_principal_ids" {
  value = {
    app_server = azurerm_linux_virtual_machine.app_server.identity[0].principal_id
  }
}
```

---

## VM Sizes Quick Reference

| Series | Use Case | vCPU | RAM | Notes |
|--------|----------|------|-----|-------|
| B1s | Dev/test | 1 | 1 GB | Cheapest |
| B2s | Small apps | 2 | 4 GB | Good for dev |
| D2s_v3 | General | 2 | 8 GB | Most common |
| D4s_v3 | General | 4 | 16 GB | Web servers |
| E4s_v3 | Memory | 4 | 32 GB | DB servers |
| F4s_v2 | Compute | 4 | 8 GB | CPU intensive |
| N-series | ML/GPU | 6+ | 56+ GB | Deep learning |

---

*จบ Part 054: Azure Virtual Machines*  
*ต่อไป Part 055: Azure AKS Kubernetes Service*
