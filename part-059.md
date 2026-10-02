# Part 059: GCP Compute Engine & GKE
## ขั้นตอนที่ 581-590: การสร้าง Compute Resources และ GKE บน GCP

---

## ขั้นตอนที่ 581: VPC Networks

```hcl
# gcp-networking.tf

# VPC Network
resource "google_compute_network" "main" {
  name                    = "vpc-myapp-prod"
  auto_create_subnetworks = false  # Custom mode (แนะนำ)
  routing_mode            = "REGIONAL"  # REGIONAL หรือ GLOBAL
  
  mtu = 1460  # 1460 (default), 1500 (jumbo frames on some VM types)
  
  description = "Main VPC for production environment"
  
  project = var.project_id
}

# Subnets
resource "google_compute_subnetwork" "private" {
  name          = "subnet-private-prod-sea"
  ip_cidr_range = "10.0.1.0/24"
  region        = "asia-southeast1"
  network       = google_compute_network.main.id
  project       = var.project_id
  
  # Private Google Access - ให้ VMs ไม่มี public IP เข้าถึง Google APIs ได้
  private_ip_google_access = true
  
  # Secondary ranges สำหรับ GKE pods และ services
  secondary_ip_range {
    range_name    = "pods"
    ip_cidr_range = "10.100.0.0/16"
  }
  
  secondary_ip_range {
    range_name    = "services"
    ip_cidr_range = "10.200.0.0/20"
  }
  
  # Flow logs
  log_config {
    aggregation_interval = "INTERVAL_10_MIN"
    flow_sampling        = 0.5
    metadata             = "INCLUDE_ALL_METADATA"
  }
  
  description = "Private subnet for production workloads"
}

resource "google_compute_subnetwork" "public" {
  name          = "subnet-public-prod-sea"
  ip_cidr_range = "10.0.2.0/24"
  region        = "asia-southeast1"
  network       = google_compute_network.main.id
  project       = var.project_id
  
  private_ip_google_access = false
  
  log_config {
    aggregation_interval = "INTERVAL_10_MIN"
    flow_sampling        = 0.5
    metadata             = "INCLUDE_ALL_METADATA"
  }
}

# Proxy-only subnet สำหรับ Internal Application Load Balancer
resource "google_compute_subnetwork" "proxy_only" {
  name          = "subnet-proxy-only-prod"
  ip_cidr_range = "10.0.3.0/24"
  region        = "asia-southeast1"
  network       = google_compute_network.main.id
  purpose       = "REGIONAL_MANAGED_PROXY"
  role          = "ACTIVE"
}

# PSC (Private Service Connect) subnet
resource "google_compute_subnetwork" "psc" {
  name          = "subnet-psc-prod"
  ip_cidr_range = "10.0.4.0/24"
  region        = "asia-southeast1"
  network       = google_compute_network.main.id
  purpose       = "PRIVATE_SERVICE_CONNECT"
}
```

---

## ขั้นตอนที่ 582: Firewall Rules

```hcl
# gcp-firewall.tf

# Allow internal traffic within VPC
resource "google_compute_firewall" "allow_internal" {
  name    = "fw-allow-internal-prod"
  network = google_compute_network.main.name
  project = var.project_id
  
  direction   = "INGRESS"
  priority    = 1000
  
  allow {
    protocol = "tcp"
  }
  allow {
    protocol = "udp"
  }
  allow {
    protocol = "icmp"
  }
  
  source_ranges = ["10.0.0.0/8"]
  
  log_config {
    metadata = "INCLUDE_ALL_METADATA"
  }
}

# Allow HTTP/HTTPS from Load Balancer (GFE)
resource "google_compute_firewall" "allow_lb" {
  name    = "fw-allow-lb-prod"
  network = google_compute_network.main.name
  project = var.project_id
  
  allow {
    protocol = "tcp"
    ports    = ["80", "443", "8080", "8443"]
  }
  
  # GCP Load Balancer health check IP ranges
  source_ranges = [
    "130.211.0.0/22",
    "35.191.0.0/16"
  ]
  
  target_tags = ["web-server", "allow-lb"]
}

# Allow SSH from IAP (Identity-Aware Proxy)
resource "google_compute_firewall" "allow_iap_ssh" {
  name    = "fw-allow-iap-ssh-prod"
  network = google_compute_network.main.name
  project = var.project_id
  
  allow {
    protocol = "tcp"
    ports    = ["22"]
  }
  
  # IAP IP range
  source_ranges = ["35.235.240.0/20"]
  
  target_tags = ["allow-ssh"]
}

# Allow RDP from IAP
resource "google_compute_firewall" "allow_iap_rdp" {
  name    = "fw-allow-iap-rdp-prod"
  network = google_compute_network.main.name
  project = var.project_id
  
  allow {
    protocol = "tcp"
    ports    = ["3389"]
  }
  
  source_ranges = ["35.235.240.0/20"]
  
  target_tags = ["allow-rdp"]
}

# Deny all ingress (default deny)
resource "google_compute_firewall" "deny_all_ingress" {
  name    = "fw-deny-all-ingress-prod"
  network = google_compute_network.main.name
  project = var.project_id
  
  priority  = 65534
  direction = "INGRESS"
  
  deny {
    protocol = "all"
  }
  
  source_ranges = ["0.0.0.0/0"]
  
  log_config {
    metadata = "INCLUDE_ALL_METADATA"
  }
}

# Allow egress to Google APIs
resource "google_compute_firewall" "allow_google_apis" {
  name    = "fw-allow-google-apis-prod"
  network = google_compute_network.main.name
  project = var.project_id
  
  direction = "EGRESS"
  priority  = 1000
  
  allow {
    protocol = "tcp"
    ports    = ["443"]
  }
  
  destination_ranges = [
    "199.36.153.4/30",   # restricted.googleapis.com
    "199.36.153.8/30"    # private.googleapis.com
  ]
  
  target_tags = ["allow-google-apis"]
}
```

---

## ขั้นตอนที่ 583: Cloud Router และ NAT

```hcl
# cloud-nat.tf

# Cloud Router
resource "google_compute_router" "main" {
  name    = "router-myapp-prod-sea"
  region  = "asia-southeast1"
  network = google_compute_network.main.id
  project = var.project_id
  
  bgp {
    asn = 64514
  }
}

# Cloud NAT
resource "google_compute_router_nat" "main" {
  name                               = "nat-myapp-prod-sea"
  router                             = google_compute_router.main.name
  region                             = google_compute_router.main.region
  nat_ip_allocate_option             = "AUTO_ONLY"  # หรือ MANUAL_ONLY
  source_subnetwork_ip_ranges_to_nat = "LIST_OF_SUBNETWORKS"
  project                            = var.project_id
  
  subnetwork {
    name                    = google_compute_subnetwork.private.id
    source_ip_ranges_to_nat = ["ALL_IP_RANGES"]
  }
  
  # Logging
  log_config {
    enable = true
    filter = "ERRORS_ONLY"  # ALL, ERRORS_ONLY, TRANSLATIONS_ONLY
  }
  
  # Timeouts
  tcp_established_idle_timeout_sec    = 1200
  tcp_transitory_idle_timeout_sec     = 30
  udp_idle_timeout_sec                = 30
  icmp_idle_timeout_sec               = 30
  tcp_time_wait_timeout_sec           = 120
  
  # Min ports per VM
  min_ports_per_vm = 64
  
  # Enable endpoint independent mapping
  enable_endpoint_independent_mapping = false
}

# Static external IP สำหรับ NAT (manual allocation)
resource "google_compute_address" "nat_ip" {
  count = 2
  
  name    = "ip-nat-prod-${count.index + 1}"
  region  = "asia-southeast1"
  project = var.project_id
}

# NAT with manual IP allocation
resource "google_compute_router_nat" "with_static_ip" {
  name                               = "nat-static-prod-sea"
  router                             = google_compute_router.main.name
  region                             = google_compute_router.main.region
  nat_ip_allocate_option             = "MANUAL_ONLY"
  source_subnetwork_ip_ranges_to_nat = "ALL_SUBNETWORKS_ALL_IP_RANGES"
  nat_ips                            = google_compute_address.nat_ip[*].self_link
  project                            = var.project_id
}
```

---

## ขั้นตอนที่ 584: Compute Engine Instances

```hcl
# compute-instances.tf

# Service Account สำหรับ VM
resource "google_service_account" "vm_sa" {
  account_id   = "vm-app-sa"
  display_name = "VM Application Service Account"
  project      = var.project_id
}

resource "google_project_iam_member" "vm_logging" {
  project = var.project_id
  role    = "roles/logging.logWriter"
  member  = "serviceAccount:${google_service_account.vm_sa.email}"
}

resource "google_project_iam_member" "vm_monitoring" {
  project = var.project_id
  role    = "roles/monitoring.metricWriter"
  member  = "serviceAccount:${google_service_account.vm_sa.email}"
}

# GCE Instance
resource "google_compute_instance" "app_server" {
  name         = "vm-app-prod-001"
  machine_type = "n2-standard-4"  # 4 vCPUs, 16 GB RAM
  zone         = "asia-southeast1-a"
  project      = var.project_id
  
  # Tags สำหรับ firewall rules
  tags = ["allow-ssh", "allow-lb", "web-server"]
  
  # Labels (GCP equivalent ของ tags)
  labels = local.common_labels
  
  # Boot disk
  boot_disk {
    auto_delete = true
    initialize_params {
      image  = "debian-cloud/debian-11"  # หรือ "ubuntu-os-cloud/ubuntu-2204-lts"
      size   = 50   # GB
      type   = "pd-ssd"  # pd-standard, pd-balanced, pd-ssd, pd-extreme
      labels = local.common_labels
    }
  }
  
  # Scratch disk (local SSD)
  # scratch_disk {
  #   interface = "NVME"
  # }
  
  # Data disk
  attached_disk {
    source      = google_compute_disk.app_data.id
    device_name = "data-disk"
    mode        = "READ_WRITE"
  }
  
  # Network interface
  network_interface {
    subnetwork = google_compute_subnetwork.private.id
    
    # ไม่กำหนด access_config = ไม่มี public IP (แนะนำ)
    # access_config {
    #   nat_ip = google_compute_address.vm_ip.address
    # }
  }
  
  # Service account
  service_account {
    email  = google_service_account.vm_sa.email
    scopes = ["cloud-platform"]  # ให้ access ทุก GCP services
  }
  
  # Metadata (startup script)
  metadata = {
    startup-script = <<-EOT
      #!/bin/bash
      apt-get update
      apt-get install -y nginx
      systemctl enable nginx
      systemctl start nginx
    EOT
    
    # หรือ reference script file
    # startup-script-url = "gs://my-bucket/startup.sh"
    
    # Enable OS Login (ใช้ Google account แทน SSH keys)
    enable-oslogin = "TRUE"
    
    # SSH keys (ถ้าไม่ใช้ OS Login)
    # ssh-keys = "adminuser:${file("~/.ssh/id_rsa.pub")}"
  }
  
  # Scheduling
  scheduling {
    on_host_maintenance = "MIGRATE"  # MIGRATE, TERMINATE
    automatic_restart   = true
    preemptible         = false  # true = Spot VM (ราคาถูก แต่ interruptions)
    
    # Spot VM
    # preemptible = true
    # provisioning_model = "SPOT"
  }
  
  # Shielded VM
  shielded_instance_config {
    enable_secure_boot          = true
    enable_vtpm                 = true
    enable_integrity_monitoring = true
  }
  
  # Confidential Computing (N2D machines only)
  # confidential_instance_config {
  #   enable_confidential_compute = true
  # }
  
  # Allow stopping for update
  allow_stopping_for_update = true
  
  # Deletion protection
  deletion_protection = true
  
  # Resource policy (schedule)
  resource_policies = [
    google_compute_resource_policy.instance_schedule.id
  ]
}

# Compute Disk
resource "google_compute_disk" "app_data" {
  name  = "disk-app-data-001"
  type  = "pd-ssd"
  zone  = "asia-southeast1-a"
  size  = 200  # GB
  
  # Snapshot schedule
  resource_policies = [
    google_compute_resource_policy.snapshot_schedule.id
  ]
  
  labels = local.common_labels
}

# Static External IP
resource "google_compute_address" "vm_ip" {
  name    = "ip-vm-app-001"
  region  = "asia-southeast1"
  project = var.project_id
}

# Resource Policy - Instance Schedule
resource "google_compute_resource_policy" "instance_schedule" {
  name    = "policy-instance-schedule"
  region  = "asia-southeast1"
  project = var.project_id
  
  instance_schedule_policy {
    vm_start_schedule {
      schedule = "0 8 * * 1-5"  # Start at 8:00 AM Mon-Fri
    }
    vm_stop_schedule {
      schedule = "0 20 * * 1-5"  # Stop at 8:00 PM Mon-Fri
    }
    time_zone = "Asia/Bangkok"
  }
}

# Resource Policy - Snapshot Schedule
resource "google_compute_resource_policy" "snapshot_schedule" {
  name    = "policy-snapshot-daily"
  region  = "asia-southeast1"
  project = var.project_id
  
  snapshot_schedule_policy {
    schedule {
      daily_schedule {
        days_in_cycle = 1
        start_time    = "04:00"
      }
    }
    retention_policy {
      max_retention_days    = 14
      on_source_disk_delete = "KEEP_AUTO_SNAPSHOTS"
    }
    snapshot_properties {
      labels            = local.common_labels
      storage_locations = ["asia-southeast1"]
      chain_name        = ""
    }
  }
}
```

---

## ขั้นตอนที่ 585: Instance Templates และ MIG

```hcl
# instance-template-mig.tf

# Instance Template
resource "google_compute_instance_template" "web" {
  name         = "it-web-prod"
  machine_type = "n2-standard-2"
  project      = var.project_id
  
  tags   = ["allow-lb", "web-server"]
  labels = local.common_labels
  
  disk {
    source_image = "debian-cloud/debian-11"
    auto_delete  = true
    boot         = true
    disk_size_gb = 50
    disk_type    = "pd-balanced"
  }
  
  network_interface {
    subnetwork = google_compute_subnetwork.private.id
  }
  
  service_account {
    email  = google_service_account.vm_sa.email
    scopes = ["cloud-platform"]
  }
  
  metadata = {
    startup-script   = file("${path.module}/scripts/startup.sh")
    enable-oslogin   = "TRUE"
  }
  
  scheduling {
    on_host_maintenance = "MIGRATE"
    automatic_restart   = true
  }
  
  shielded_instance_config {
    enable_secure_boot = true
    enable_vtpm        = true
  }
  
  lifecycle {
    create_before_destroy = true
  }
}

# Managed Instance Group (MIG)
resource "google_compute_region_instance_group_manager" "web" {
  name               = "mig-web-prod"
  base_instance_name = "web"
  region             = "asia-southeast1"
  project            = var.project_id
  
  version {
    instance_template = google_compute_instance_template.web.id
    name              = "primary"
  }
  
  # Target Pools (for HTTP LB)
  # target_pools = [google_compute_target_pool.web.id]
  
  # Named ports
  named_port {
    name = "http"
    port = 80
  }
  
  # Autohealing
  auto_healing_policies {
    health_check      = google_compute_health_check.http.id
    initial_delay_sec = 300
  }
  
  # Update policy
  update_policy {
    type                         = "PROACTIVE"  # PROACTIVE, OPPORTUNISTIC
    minimal_action               = "REPLACE"    # REPLACE, RESTART
    most_disruptive_allowed_action = "REPLACE"
    max_unavailable_fixed        = 1
    max_surge_fixed              = 1
    replacement_method           = "SUBSTITUTE"
  }
  
  # Distribution policy
  distribution_policy_target_shape = "EVEN"  # EVEN, ANY
  distribution_policy_zones = [
    "asia-southeast1-a",
    "asia-southeast1-b",
    "asia-southeast1-c"
  ]
  
  stateful_disk {
    device_name = "data-disk"
    delete_rule = "ON_PERMANENT_INSTANCE_DELETION"
  }
}

# Autoscaler
resource "google_compute_region_autoscaler" "web" {
  name    = "autoscaler-web-prod"
  region  = "asia-southeast1"
  target  = google_compute_region_instance_group_manager.web.id
  project = var.project_id
  
  autoscaling_policy {
    max_replicas    = 20
    min_replicas    = 3
    cooldown_period = 60
    
    # CPU utilization
    cpu_utilization {
      target = 0.7  # 70%
    }
    
    # Load balancing utilization
    load_balancing_utilization {
      target = 0.8
    }
    
    # Custom metrics
    metric {
      name   = "pubsub.googleapis.com/subscription/num_undelivered_messages"
      target = 1000
      type   = "GAUGE"
    }
    
    # Scale in control
    scale_in_control {
      max_scaled_in_replicas {
        fixed = 2
      }
      time_window_sec = 300
    }
  }
}

# Health Check
resource "google_compute_health_check" "http" {
  name    = "hc-http-web"
  project = var.project_id
  
  timeout_sec         = 5
  check_interval_sec  = 10
  healthy_threshold   = 2
  unhealthy_threshold = 3
  
  http_health_check {
    port         = 80
    request_path = "/health"
  }
  
  log_config {
    enable = true
  }
}
```

---

## ขั้นตอนที่ 586: Cloud SQL

```hcl
# cloud-sql.tf

# Cloud SQL Instance (PostgreSQL)
resource "google_sql_database_instance" "postgres" {
  name             = "sql-myapp-prod-sea"
  database_version = "POSTGRES_15"
  region           = "asia-southeast1"
  project          = var.project_id
  
  # Deletion protection
  deletion_protection = true
  
  settings {
    tier              = "db-custom-4-15360"  # 4 vCPU, 15 GB RAM
    availability_type = "REGIONAL"            # ZONAL หรือ REGIONAL (HA)
    
    disk_autoresize       = true
    disk_autoresize_limit = 1000  # GB
    disk_size             = 100   # GB initial
    disk_type             = "PD_SSD"
    
    # Backup
    backup_configuration {
      enabled                        = true
      start_time                     = "02:00"  # UTC
      point_in_time_recovery_enabled = true
      location                       = "asia"
      
      backup_retention_settings {
        retained_backups = 30
        retention_unit   = "COUNT"
      }
    }
    
    # Maintenance window
    maintenance_window {
      day          = 7  # Sunday
      hour         = 2  # 2 AM UTC
      update_track = "stable"
    }
    
    # IP configuration
    ip_configuration {
      ipv4_enabled                                  = false  # No public IP
      private_network                               = google_compute_network.main.id
      require_ssl                                   = true
      enable_private_path_for_google_cloud_services = true
      
      # Authorized networks (สำหรับ dev เท่านั้น)
      # authorized_networks {
      #   value = "203.0.113.0/24"
      #   name  = "office"
      # }
    }
    
    # Password validation
    password_validation_policy {
      enable_password_policy = true
      min_length             = 12
      complexity              = "COMPLEXITY_DEFAULT"
      disallow_username_substring = true
      reuse_interval         = 5
      password_change_interval = "300s"
    }
    
    # Database flags
    database_flags {
      name  = "max_connections"
      value = "500"
    }
    
    database_flags {
      name  = "log_min_duration_statement"
      value = "1000"
    }
    
    database_flags {
      name  = "log_checkpoints"
      value = "on"
    }
    
    database_flags {
      name  = "log_connections"
      value = "on"
    }
    
    # Query Insights
    insights_config {
      query_insights_enabled  = true
      query_string_length     = 4500
      record_application_tags = true
      record_client_address   = true
    }
    
    # Labels
    user_labels = local.common_labels
  }
  
  depends_on = [google_service_networking_connection.private_vpc_connection]
}

# Private Service Connection สำหรับ Cloud SQL
resource "google_compute_global_address" "private_ip_range" {
  name          = "ip-range-private-services"
  purpose       = "VPC_PEERING"
  address_type  = "INTERNAL"
  prefix_length = 16
  network       = google_compute_network.main.id
  project       = var.project_id
}

resource "google_service_networking_connection" "private_vpc_connection" {
  network                 = google_compute_network.main.id
  service                 = "servicenetworking.googleapis.com"
  reserved_peering_ranges = [google_compute_global_address.private_ip_range.name]
}

# Cloud SQL Database
resource "google_sql_database" "app" {
  name     = "appdb"
  instance = google_sql_database_instance.postgres.name
  project  = var.project_id
}

# Cloud SQL User
resource "google_sql_user" "app_user" {
  name     = "appuser"
  instance = google_sql_database_instance.postgres.name
  password = random_password.sql_user.result
  project  = var.project_id
}

# Read Replica
resource "google_sql_database_instance" "read_replica" {
  name                 = "sql-myapp-prod-sea-replica"
  database_version     = "POSTGRES_15"
  region               = "asia-southeast1"
  project              = var.project_id
  master_instance_name = google_sql_database_instance.postgres.name
  
  deletion_protection = true
  
  settings {
    tier              = "db-custom-2-7680"  # Smaller for read replica
    availability_type = "ZONAL"
    
    ip_configuration {
      ipv4_enabled    = false
      private_network = google_compute_network.main.id
    }
    
    database_flags {
      name  = "max_connections"
      value = "300"
    }
  }
  
  replica_configuration {
    failover_target = false
  }
}
```

---

## ขั้นตอนที่ 587: Cloud Storage

```hcl
# cloud-storage.tf

# GCS Bucket
resource "google_storage_bucket" "app_data" {
  name          = "${var.project_id}-app-data-prod"  # globally unique
  location      = "ASIA-SOUTHEAST1"  # Regional
  storage_class = "STANDARD"
  project       = var.project_id
  
  # Versioning
  versioning {
    enabled = true
  }
  
  # Lifecycle rules
  lifecycle_rule {
    condition {
      age            = 30  # days
      with_state     = "ANY"
    }
    action {
      type          = "SetStorageClass"
      storage_class = "NEARLINE"
    }
  }
  
  lifecycle_rule {
    condition {
      age        = 90
      with_state = "ANY"
    }
    action {
      type          = "SetStorageClass"
      storage_class = "COLDLINE"
    }
  }
  
  lifecycle_rule {
    condition {
      age            = 365
      num_newer_versions = 3
    }
    action {
      type = "Delete"
    }
  }
  
  # Encryption
  encryption {
    default_kms_key_name = google_kms_crypto_key.storage.id
  }
  
  # Uniform bucket-level access
  uniform_bucket_level_access = true
  
  # Public access prevention
  public_access_prevention = "enforced"
  
  # CORS
  cors {
    origin          = ["https://myapp.com"]
    method          = ["GET", "HEAD", "POST"]
    response_header = ["*"]
    max_age_seconds = 3600
  }
  
  # Logging
  logging {
    log_bucket        = google_storage_bucket.logs.name
    log_object_prefix = "storage-access-logs/"
  }
  
  # Website configuration (สำหรับ static website)
  # website {
  #   main_page_suffix = "index.html"
  #   not_found_page   = "404.html"
  # }
  
  labels = local.common_labels
  
  # Prevent accidental deletion
  force_destroy = false
}

# IAM สำหรับ Bucket
resource "google_storage_bucket_iam_member" "app_writer" {
  bucket = google_storage_bucket.app_data.name
  role   = "roles/storage.objectAdmin"
  member = "serviceAccount:${google_service_account.app.email}"
}

resource "google_storage_bucket_iam_member" "public_viewer" {
  count  = var.make_bucket_public ? 1 : 0
  bucket = google_storage_bucket.app_data.name
  role   = "roles/storage.objectViewer"
  member = "allUsers"
}

# Bucket Notification (Pub/Sub)
resource "google_storage_notification" "app_data" {
  bucket         = google_storage_bucket.app_data.name
  payload_format = "JSON_API_V1"
  topic          = google_pubsub_topic.storage_events.id
  
  event_types = [
    "OBJECT_FINALIZE",
    "OBJECT_METADATA_UPDATE",
    "OBJECT_DELETE"
  ]
  
  custom_attributes = {
    source = "app-data-bucket"
  }
}
```

---

## ขั้นตอนที่ 588: GKE Standard Cluster

```hcl
# gke-cluster.tf

# GKE Standard Cluster
resource "google_container_cluster" "main" {
  name     = "gke-myapp-prod-sea"
  location = "asia-southeast1"  # Regional cluster (HA)
  project  = var.project_id
  
  # Remove default node pool (create custom pool instead)
  remove_default_node_pool = true
  initial_node_count       = 1
  
  # Network
  network    = google_compute_network.main.id
  subnetwork = google_compute_subnetwork.private.id
  
  # VPC-Native (recommended)
  ip_allocation_policy {
    cluster_secondary_range_name  = "pods"
    services_secondary_range_name = "services"
  }
  
  # Private Cluster
  private_cluster_config {
    enable_private_nodes    = true
    enable_private_endpoint = false  # true = public endpoint ปิด
    master_ipv4_cidr_block  = "172.16.0.0/28"
  }
  
  # Master authorized networks
  master_authorized_networks_config {
    cidr_blocks {
      cidr_block   = "10.0.0.0/8"
      display_name = "internal"
    }
    cidr_blocks {
      cidr_block   = "203.0.113.0/24"
      display_name = "office"
    }
  }
  
  # Kubernetes version
  min_master_version = "1.28"
  
  # Release channel
  release_channel {
    channel = "REGULAR"  # RAPID, REGULAR, STABLE, UNSPECIFIED
  }
  
  # Addons
  addons_config {
    http_load_balancing {
      disabled = false
    }
    horizontal_pod_autoscaling {
      disabled = false
    }
    network_policy_config {
      disabled = false
    }
    gcp_filestore_csi_driver_config {
      enabled = true
    }
    gcs_fuse_csi_driver_config {
      enabled = true
    }
    gke_backup_agent_config {
      enabled = true
    }
  }
  
  # Network policy
  network_policy {
    enabled  = true
    provider = "CALICO"  # CALICO หรือ PROVIDER_UNSPECIFIED
  }
  
  # Workload Identity
  workload_identity_config {
    workload_pool = "${var.project_id}.svc.id.goog"
  }
  
  # Cluster Autoscaler
  cluster_autoscaling {
    enabled             = true
    autoscaling_profile = "BALANCED"  # BALANCED, OPTIMIZE_UTILIZATION
    
    resource_limits {
      resource_type = "cpu"
      minimum       = 4
      maximum       = 100
    }
    
    resource_limits {
      resource_type = "memory"
      minimum       = 16
      maximum       = 400
    }
    
    auto_provisioning_defaults {
      service_account = google_service_account.gke_nodes.email
      oauth_scopes    = ["https://www.googleapis.com/auth/cloud-platform"]
      
      disk_size = 100
      disk_type = "pd-balanced"
      
      management {
        auto_repair  = true
        auto_upgrade = true
      }
    }
  }
  
  # Logging and Monitoring
  logging_config {
    enable_components = [
      "SYSTEM_COMPONENTS",
      "WORKLOADS",
      "APISERVER",
      "SCHEDULER",
      "CONTROLLER_MANAGER"
    ]
  }
  
  monitoring_config {
    enable_components = [
      "SYSTEM_COMPONENTS",
      "WORKLOADS",
      "APISERVER",
      "SCHEDULER",
      "CONTROLLER_MANAGER",
      "STORAGE",
      "DAEMONSET",
      "DEPLOYMENT",
      "HPA"
    ]
    
    managed_prometheus {
      enabled = true
    }
  }
  
  # Security
  enable_shielded_nodes = true
  
  binary_authorization {
    evaluation_mode = "PROJECT_SINGLETON_POLICY_ENFORCE"
  }
  
  # Maintenance window
  maintenance_policy {
    recurring_window {
      start_time = "2024-01-01T02:00:00Z"
      end_time   = "2024-01-01T06:00:00Z"
      recurrence = "FREQ=WEEKLY;BYDAY=SU"
    }
    
    maintenance_exclusion {
      exclusion_name = "holiday-freeze"
      start_time     = "2024-12-20T00:00:00Z"
      end_time       = "2025-01-02T00:00:00Z"
      exclusion_options {
        scope = "MINOR_UPGRADE_ONLY"
      }
    }
  }
  
  # Gateway API
  gateway_api_config {
    channel = "CHANNEL_STANDARD"
  }
  
  # Fleet registration
  fleet {
    project = var.project_id
  }
  
  deletion_protection = true
  
  labels = local.common_labels
  
  depends_on = [
    google_project_service.apis["container.googleapis.com"]
  ]
}
```

---

## ขั้นตอนที่ 589: GKE Node Pools

```hcl
# gke-node-pools.tf

# Service Account สำหรับ GKE nodes
resource "google_service_account" "gke_nodes" {
  account_id   = "gke-nodes-sa"
  display_name = "GKE Nodes Service Account"
  project      = var.project_id
}

resource "google_project_iam_member" "gke_node_roles" {
  for_each = toset([
    "roles/logging.logWriter",
    "roles/monitoring.metricWriter",
    "roles/monitoring.viewer",
    "roles/storage.objectViewer",
    "roles/artifactregistry.reader"
  ])
  
  project = var.project_id
  role    = each.value
  member  = "serviceAccount:${google_service_account.gke_nodes.email}"
}

# Default Node Pool
resource "google_container_node_pool" "default" {
  name     = "default"
  location = "asia-southeast1"
  cluster  = google_container_cluster.main.name
  project  = var.project_id
  
  # Initial count per zone (3 zones = 3*3=9 nodes)
  initial_node_count = 3
  
  autoscaling {
    min_node_count  = 3
    max_node_count  = 10
    location_policy = "BALANCED"
  }
  
  management {
    auto_repair  = true
    auto_upgrade = true
  }
  
  upgrade_settings {
    max_surge       = 1
    max_unavailable = 0
    strategy        = "SURGE"
  }
  
  node_config {
    machine_type = "n2-standard-4"  # 4 vCPU, 16 GB
    disk_size_gb = 100
    disk_type    = "pd-balanced"
    
    service_account = google_service_account.gke_nodes.email
    oauth_scopes    = ["https://www.googleapis.com/auth/cloud-platform"]
    
    # Metadata
    metadata = {
      disable-legacy-endpoints = "true"
    }
    
    # Labels
    labels = merge(local.common_labels, {
      nodepool = "default"
    })
    
    # Shielded nodes
    shielded_instance_config {
      enable_secure_boot          = true
      enable_integrity_monitoring = true
    }
    
    # Workload Identity
    workload_metadata_config {
      mode = "GKE_METADATA"
    }
    
    # Spot/Preemptible (ราคาถูก)
    # spot = true
    
    tags = ["gke-node", "allow-lb"]
  }
  
  network_config {
    enable_private_nodes = true
  }
}

# Spot Node Pool
resource "google_container_node_pool" "spot" {
  name     = "spot-pool"
  location = "asia-southeast1"
  cluster  = google_container_cluster.main.name
  project  = var.project_id
  
  initial_node_count = 0
  
  autoscaling {
    min_node_count = 0
    max_node_count = 50
  }
  
  management {
    auto_repair  = true
    auto_upgrade = true
  }
  
  node_config {
    machine_type = "n2-standard-8"
    disk_size_gb = 100
    disk_type    = "pd-standard"
    
    # Spot instances
    spot = true
    
    service_account = google_service_account.gke_nodes.email
    oauth_scopes    = ["https://www.googleapis.com/auth/cloud-platform"]
    
    labels = merge(local.common_labels, {
      nodepool = "spot"
    })
    
    taint {
      key    = "cloud.google.com/gke-spot"
      value  = "true"
      effect = "NO_SCHEDULE"
    }
    
    metadata = {
      disable-legacy-endpoints = "true"
    }
    
    workload_metadata_config {
      mode = "GKE_METADATA"
    }
  }
}

# GPU Node Pool
resource "google_container_node_pool" "gpu" {
  name     = "gpu-pool"
  location = "asia-southeast1-a"  # Zonal - GPUs often in specific zones
  cluster  = google_container_cluster.main.name
  project  = var.project_id
  
  initial_node_count = 0
  
  autoscaling {
    min_node_count = 0
    max_node_count = 4
  }
  
  node_config {
    machine_type = "n1-standard-8"  # 8 vCPU, 30 GB
    disk_size_gb = 200
    disk_type    = "pd-ssd"
    
    # GPU accelerator
    guest_accelerator {
      type  = "nvidia-tesla-t4"
      count = 1
      
      gpu_driver_installation_config {
        gpu_driver_version = "LATEST"
      }
    }
    
    service_account = google_service_account.gke_nodes.email
    oauth_scopes    = ["https://www.googleapis.com/auth/cloud-platform"]
    
    labels = merge(local.common_labels, {
      nodepool    = "gpu"
      accelerator = "nvidia-t4"
    })
    
    taint {
      key    = "nvidia.com/gpu"
      value  = "present"
      effect = "NO_SCHEDULE"
    }
    
    metadata = {
      disable-legacy-endpoints = "true"
    }
    
    workload_metadata_config {
      mode = "GKE_METADATA"
    }
  }
}
```

---

## ขั้นตอนที่ 590: GKE Autopilot

```hcl
# gke-autopilot.tf

# GKE Autopilot Cluster
resource "google_container_cluster" "autopilot" {
  name     = "gke-autopilot-prod-sea"
  location = "asia-southeast1"
  project  = var.project_id
  
  # Autopilot mode
  enable_autopilot = true
  
  # Network
  network    = google_compute_network.main.id
  subnetwork = google_compute_subnetwork.private.id
  
  ip_allocation_policy {
    cluster_secondary_range_name  = "pods"
    services_secondary_range_name = "services"
  }
  
  private_cluster_config {
    enable_private_nodes    = true
    enable_private_endpoint = false
    master_ipv4_cidr_block  = "172.16.1.0/28"
  }
  
  master_authorized_networks_config {
    cidr_blocks {
      cidr_block   = "10.0.0.0/8"
      display_name = "internal"
    }
  }
  
  release_channel {
    channel = "REGULAR"
  }
  
  # Workload Identity
  workload_identity_config {
    workload_pool = "${var.project_id}.svc.id.goog"
  }
  
  # Monitoring
  monitoring_config {
    enable_components = ["SYSTEM_COMPONENTS", "WORKLOADS"]
    managed_prometheus {
      enabled = true
    }
  }
  
  maintenance_policy {
    recurring_window {
      start_time = "2024-01-01T02:00:00Z"
      end_time   = "2024-01-01T06:00:00Z"
      recurrence = "FREQ=WEEKLY;BYDAY=SU"
    }
  }
  
  deletion_protection = true
  labels              = local.common_labels
}

# ============================================================
# GKE Workload Identity
# ============================================================

# K8s ServiceAccount -> Google Service Account mapping
resource "google_service_account" "k8s_app" {
  account_id   = "k8s-app-sa"
  display_name = "Kubernetes App Service Account"
  project      = var.project_id
}

resource "google_service_account_iam_member" "workload_identity" {
  service_account_id = google_service_account.k8s_app.name
  role               = "roles/iam.workloadIdentityUser"
  member             = "serviceAccount:${var.project_id}.svc.id.goog[default/app-service-account]"
}

resource "google_project_iam_member" "k8s_app_storage" {
  project = var.project_id
  role    = "roles/storage.objectAdmin"
  member  = "serviceAccount:${google_service_account.k8s_app.email}"
}

# ============================================================
# Outputs
# ============================================================

output "gke_cluster_info" {
  value = {
    name            = google_container_cluster.main.name
    endpoint        = google_container_cluster.main.endpoint
    location        = google_container_cluster.main.location
    master_version  = google_container_cluster.main.master_version
  }
}

output "gke_connection_command" {
  value = "gcloud container clusters get-credentials ${google_container_cluster.main.name} --region ${google_container_cluster.main.location} --project ${var.project_id}"
}

output "sql_connection_string" {
  value     = "postgresql://${google_sql_user.app_user.name}:${random_password.sql_user.result}@${google_sql_database_instance.postgres.private_ip_address}/${google_sql_database.app.name}"
  sensitive = true
}
```

---

## GKE Standard vs Autopilot Comparison

| Feature | Standard | Autopilot |
|---------|---------|-----------|
| Node management | Manual | Google managed |
| Pricing | Per node | Per pod resource |
| Node pools | Custom | Automatic |
| Bin packing | Manual | Optimized |
| Node sizing | Manual | Automatic |
| Maintenance | Manual/auto | Automatic |
| Best for | Large teams, complex setups | Small teams, ease of use |
| Cost at low load | Higher (min nodes) | Lower (per pod) |
| Cost at high load | Lower (bin packing) | Higher |

---

*จบ Part 059: GCP Compute Engine & GKE*  
*ต่อไป Part 060: Multi-Cloud Patterns*
