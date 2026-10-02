# Part 062: Complex Variable Types & Structures
## ประเภทตัวแปรที่ซับซ้อนและโครงสร้างข้อมูล
### Steps 611-620

---

## บทนำ (Introduction)

ในบทนี้เราจะเรียนรู้การออกแบบ variable types ที่ซับซ้อน เพื่อทำให้ module interfaces สะอาดและใช้งานง่าย การใช้ complex types อย่างถูกต้องจะช่วยลด configuration errors และทำให้ code อ่านง่ายขึ้น

---

## Step 611: Object Types สำหรับ Server Configurations

### การออกแบบ object type สำหรับ server configuration

```hcl
# ==========================================
# SIMPLE OBJECT - Server Configuration
# ==========================================

variable "web_server" {
  type = object({
    instance_type = string
    ami_id        = string
    key_name      = string
    disk_size_gb  = number
    enable_eip    = bool
  })
  description = "Web server configuration"

  default = {
    instance_type = "t3.medium"
    ami_id        = "ami-0c55b159cbfafe1f0"
    key_name      = "my-key"
    disk_size_gb  = 50
    enable_eip    = false
  }
}

# การใช้งาน
resource "aws_instance" "web" {
  ami           = var.web_server.ami_id
  instance_type = var.web_server.instance_type
  key_name      = var.web_server.key_name

  root_block_device {
    volume_size = var.web_server.disk_size_gb
  }
}

# ==========================================
# NESTED OBJECT - Database Server Configuration
# ==========================================

variable "database_server" {
  type = object({
    instance = object({
      class         = string
      engine        = string
      engine_version = string
    })
    storage = object({
      allocated_gb     = number
      max_allocated_gb = number
      type             = string
      encrypted        = bool
    })
    network = object({
      multi_az            = bool
      publicly_accessible = bool
      backup_window       = string
      maintenance_window  = string
    })
    backup = object({
      retention_days  = number
      skip_final_snapshot = bool
    })
  })
  description = "Complete database server configuration"

  default = {
    instance = {
      class          = "db.t3.medium"
      engine         = "postgres"
      engine_version = "15.4"
    }
    storage = {
      allocated_gb     = 100
      max_allocated_gb = 1000
      type             = "gp3"
      encrypted        = true
    }
    network = {
      multi_az            = true
      publicly_accessible = false
      backup_window       = "03:00-04:00"
      maintenance_window  = "Mon:04:00-Mon:05:00"
    }
    backup = {
      retention_days      = 7
      skip_final_snapshot = false
    }
  }
}

# การใช้งาน
resource "aws_db_instance" "main" {
  instance_class    = var.database_server.instance.class
  engine            = var.database_server.instance.engine
  engine_version    = var.database_server.instance.engine_version

  allocated_storage     = var.database_server.storage.allocated_gb
  max_allocated_storage = var.database_server.storage.max_allocated_gb
  storage_type          = var.database_server.storage.type
  storage_encrypted     = var.database_server.storage.encrypted

  multi_az            = var.database_server.network.multi_az
  publicly_accessible = var.database_server.network.publicly_accessible
  backup_window       = var.database_server.network.backup_window
  maintenance_window  = var.database_server.network.maintenance_window

  backup_retention_period = var.database_server.backup.retention_days
  skip_final_snapshot     = var.database_server.backup.skip_final_snapshot
}
```

---

## Step 612: Map of Objects สำหรับ Multiple Resources

### การใช้ map of objects เพื่อสร้างหลาย resources

```hcl
# ==========================================
# MAP OF OBJECTS - Multiple Server Fleet
# ==========================================

variable "servers" {
  type = map(object({
    instance_type = string
    ami_id        = string
    subnet_id     = string
    disk_size_gb  = number
    tags          = optional(map(string), {})
  }))
  description = "Map ของ servers ที่ต้องการสร้าง"

  default = {
    "web-1" = {
      instance_type = "t3.medium"
      ami_id        = "ami-0c55b159cbfafe1f0"
      subnet_id     = "subnet-12345"
      disk_size_gb  = 50
    }
    "web-2" = {
      instance_type = "t3.medium"
      ami_id        = "ami-0c55b159cbfafe1f0"
      subnet_id     = "subnet-67890"
      disk_size_gb  = 50
    }
    "app-1" = {
      instance_type = "c5.large"
      ami_id        = "ami-0c55b159cbfafe1f0"
      subnet_id     = "subnet-12345"
      disk_size_gb  = 100
      tags = {
        Role = "app-server"
      }
    }
  }
}

# สร้าง instances จาก map
resource "aws_instance" "servers" {
  for_each = var.servers

  ami           = each.value.ami_id
  instance_type = each.value.instance_type
  subnet_id     = each.value.subnet_id

  root_block_device {
    volume_size = each.value.disk_size_gb
  }

  tags = merge(
    { Name = each.key },
    each.value.tags
  )
}

# ==========================================
# MAP OF OBJECTS - Security Group Rules
# ==========================================

variable "security_group_rules" {
  type = map(object({
    type        = string  # "ingress" or "egress"
    from_port   = number
    to_port     = number
    protocol    = string
    cidr_blocks = list(string)
    description = optional(string, "")
  }))
  description = "Security group rules"

  default = {
    "https-in" = {
      type        = "ingress"
      from_port   = 443
      to_port     = 443
      protocol    = "tcp"
      cidr_blocks = ["0.0.0.0/0"]
      description = "Allow HTTPS from anywhere"
    }
    "http-in" = {
      type        = "ingress"
      from_port   = 80
      to_port     = 80
      protocol    = "tcp"
      cidr_blocks = ["0.0.0.0/0"]
      description = "Allow HTTP from anywhere"
    }
    "all-out" = {
      type        = "egress"
      from_port   = 0
      to_port     = 0
      protocol    = "-1"
      cidr_blocks = ["0.0.0.0/0"]
      description = "Allow all outbound"
    }
  }
}

resource "aws_security_group_rule" "rules" {
  for_each = var.security_group_rules

  security_group_id = aws_security_group.main.id
  type              = each.value.type
  from_port         = each.value.from_port
  to_port           = each.value.to_port
  protocol          = each.value.protocol
  cidr_blocks       = each.value.cidr_blocks
  description       = each.value.description
}
```

---

## Step 613: List of Objects

### การใช้ list of objects สำหรับ ordered items

```hcl
# ==========================================
# LIST OF OBJECTS - Ordered Configuration
# ==========================================

# Listener rules (ordered list - ลำดับสำคัญ)
variable "alb_listener_rules" {
  type = list(object({
    priority    = number
    conditions  = list(object({
      field   = string  # "path-pattern", "host-header", "http-header"
      values  = list(string)
    }))
    actions     = list(object({
      type             = string
      target_group_arn = optional(string)
      redirect         = optional(object({
        port        = optional(string, "443")
        protocol    = optional(string, "HTTPS")
        status_code = string
      }))
    }))
  }))
  description = "ALB Listener rules (ordered by priority)"
  default     = []
}

# ตัวอย่างค่า
# alb_listener_rules = [
#   {
#     priority = 10
#     conditions = [{
#       field  = "path-pattern"
#       values = ["/api/*"]
#     }]
#     actions = [{
#       type             = "forward"
#       target_group_arn = "arn:aws:elasticloadbalancing:..."
#     }]
#   },
#   {
#     priority = 20
#     conditions = [{
#       field  = "host-header"
#       values = ["api.example.com"]
#     }]
#     actions = [{
#       type = "redirect"
#       redirect = {
#         port        = "443"
#         protocol    = "HTTPS"
#         status_code = "HTTP_301"
#       }
#     }]
#   }
# ]

# ==========================================
# LIST OF OBJECTS - DNS Records
# ==========================================

variable "dns_records" {
  type = list(object({
    name    = string
    type    = string
    ttl     = optional(number, 300)
    records = list(string)
  }))
  description = "รายการ DNS records"
  default     = []
}

resource "aws_route53_record" "records" {
  count = length(var.dns_records)

  zone_id = aws_route53_zone.main.zone_id
  name    = var.dns_records[count.index].name
  type    = var.dns_records[count.index].type
  ttl     = var.dns_records[count.index].ttl
  records = var.dns_records[count.index].records
}

# ==========================================
# LIST OF OBJECTS - IAM Policy Statements
# ==========================================

variable "iam_policy_statements" {
  type = list(object({
    sid       = optional(string)
    effect    = string  # "Allow" or "Deny"
    actions   = list(string)
    resources = list(string)
    conditions = optional(list(object({
      test     = string
      variable = string
      values   = list(string)
    })), [])
  }))
  description = "IAM policy statements"
  default     = []
}
```

---

## Step 614: Optional() Attributes ใน Objects

### การใช้ optional() function (Terraform 1.3+)

```hcl
# ==========================================
# OPTIONAL ATTRIBUTES - Terraform 1.3+
# ==========================================

# การใช้ optional() พื้นฐาน
variable "app_config" {
  type = object({
    # Required attributes
    name    = string
    image   = string

    # Optional attributes with defaults
    replicas    = optional(number, 1)
    port        = optional(number, 8080)
    memory_mb   = optional(number, 256)
    cpu_units   = optional(number, 256)
    environment = optional(map(string), {})
    secrets     = optional(map(string), {})

    # Optional nested object
    health_check = optional(object({
      path                = optional(string, "/health")
      port                = optional(number, 8080)
      interval_seconds    = optional(number, 30)
      timeout_seconds     = optional(number, 5)
      healthy_threshold   = optional(number, 2)
      unhealthy_threshold = optional(number, 3)
    }), {})

    # Optional list
    volumes = optional(list(object({
      name       = string
      mount_path = string
      read_only  = optional(bool, false)
    })), [])
  })
  description = "Application configuration"
}

# ตัวอย่างการใช้งาน - minimal config (ใช้ defaults)
# app_config = {
#   name  = "myapp"
#   image = "nginx:latest"
#   # replicas = 1 (default)
#   # port = 8080 (default)
# }

# ตัวอย่างการใช้งาน - full config
# app_config = {
#   name     = "myapp"
#   image    = "nginx:latest"
#   replicas = 3
#   port     = 80
#   health_check = {
#     path     = "/ping"
#     interval_seconds = 15
#   }
#   volumes = [
#     {
#       name       = "config"
#       mount_path = "/etc/config"
#       read_only  = true
#     }
#   ]
# }

# ==========================================
# OPTIONAL IN MAP OF OBJECTS
# ==========================================

variable "services" {
  type = map(object({
    image    = string
    replicas = optional(number, 1)
    ports    = optional(list(number), [])
    env      = optional(map(string), {})
    labels   = optional(map(string), {})
    resources = optional(object({
      cpu_limit    = optional(string, "500m")
      memory_limit = optional(string, "512Mi")
      cpu_request  = optional(string, "100m")
      memory_request = optional(string, "128Mi")
    }), {})
  }))
  description = "Service configurations"
  default     = {}
}

# ==========================================
# defaults() FUNCTION FOR OPTIONAL ATTRS
# ==========================================

# ใช้ defaults() function เพื่อ fill in defaults
locals {
  # Fill in defaults for server config
  server_with_defaults = defaults(var.app_config, {
    replicas  = 1
    port      = 8080
    memory_mb = 256
    cpu_units = 256
  })
}
```

---

## Step 615: Type Constraints Deep Dive

### การเข้าใจ Type Constraints อย่างลึกซึ้ง

```hcl
# ==========================================
# TYPE CONSTRAINTS DEEP DIVE
# ==========================================

# any type - ยอมรับทุก type
variable "flexible_config" {
  type        = any
  description = "Configuration ที่ยืดหยุ่น"
  default     = {}
}

# เมื่อใช้ any จะไม่มี type checking
# ควรใช้เฉพาะเมื่อ type ไม่แน่นอนจริงๆ

# ตัวอย่างที่ควรใช้ any:
# - Meta-arguments ที่รับ config หลายรูปแบบ
# - Modules ที่รองรับหลาย cloud providers
# - Test utilities

# ==========================================
# TYPE CONVERSION PITFALLS
# ==========================================

# PITFALL 1: number ที่ให้เป็น string
variable "port_as_string" {
  type    = string
  default = "8080"
}

# Terraform จะ auto-convert เมื่อ type compatible
# แต่บางครั้งอาจทำให้สับสน

# PITFALL 2: bool conversion
variable "enabled_as_string" {
  type    = string
  default = "true"
}
# "true" (string) != true (bool)

# PITFALL 3: number ที่มี decimal
variable "instance_count" {
  type    = number
  default = 1
  # number ใน Terraform เป็น float64
  # 1 == 1.0 == 1.00 ทั้งหมดเท่ากัน
}

# PITFALL 4: list vs set
variable "az_list" {
  type    = list(string)
  default = ["ap-southeast-1a", "ap-southeast-1b", "ap-southeast-1a"]  # มี duplicate!
  # list เก็บ order และยอมให้ duplicate
}

variable "az_set" {
  type    = set(string)
  default = ["ap-southeast-1a", "ap-southeast-1b"]  # ต้อง unique
  # set ไม่เก็บ order แต่ต้อง unique
}

# ==========================================
# EXPLICIT TYPE CONVERSION
# ==========================================

locals {
  # tostring() - แปลงเป็น string
  port_string = tostring(8080)            # "8080"
  
  # tonumber() - แปลงเป็น number
  port_number = tonumber("8080")          # 8080
  
  # tobool() - แปลงเป็น bool
  enabled = tobool("true")               # true
  
  # tolist() - แปลงเป็น list
  az_list = tolist(toset(["a", "b", "a"]))  # ["a", "b"] (deduplicated)
  
  # toset() - แปลงเป็น set
  az_set = toset(["ap-southeast-1a", "ap-southeast-1b"])
  
  # tomap() - แปลงเป็น map
  config_map = tomap({ key = "value" })
}

# ==========================================
# COMPLEX TYPE EXAMPLES
# ==========================================

# VPC Configuration Object (Full Example)
variable "vpc_config" {
  type = object({
    cidr_block = string
    name       = string

    subnets = object({
      public = list(object({
        cidr              = string
        availability_zone = string
        name              = optional(string)
      }))
      private = list(object({
        cidr              = string
        availability_zone = string
        name              = optional(string)
      }))
      database = optional(list(object({
        cidr              = string
        availability_zone = string
        name              = optional(string)
      })), [])
    })

    nat_gateway = optional(object({
      enabled        = bool
      single_gateway = optional(bool, false)
    }), { enabled = false })

    flow_logs = optional(object({
      enabled          = bool
      retention_days   = optional(number, 30)
      traffic_type     = optional(string, "ALL")
    }), { enabled = false })

    tags = optional(map(string), {})
  })
  description = "Complete VPC configuration"

  validation {
    condition     = can(cidrhost(var.vpc_config.cidr_block, 0))
    error_message = "VPC CIDR must be valid IPv4 CIDR notation."
  }

  validation {
    condition     = length(var.vpc_config.subnets.public) >= 2
    error_message = "At least 2 public subnets required for high availability."
  }
}
```

---

## Step 616: Variables สำหรับ Modules

### การออกแบบ module interface

```hcl
# ==========================================
# MODULE INTERFACE DESIGN
# ==========================================
# ไฟล์: modules/vpc/variables.tf

# Required variables (ไม่มี default)
variable "vpc_cidr" {
  type        = string
  description = "(Required) CIDR block สำหรับ VPC"

  validation {
    condition     = can(cidrhost(var.vpc_cidr, 0))
    error_message = "VPC CIDR must be valid IPv4 CIDR."
  }
}

variable "project" {
  type        = string
  description = "(Required) ชื่อโปรเจค"

  validation {
    condition     = can(regex("^[a-z0-9-]+$", var.project))
    error_message = "Project name may only contain lowercase letters, numbers, and hyphens."
  }
}

variable "environment" {
  type        = string
  description = "(Required) สภาพแวดล้อม"

  validation {
    condition     = contains(["dev", "staging", "prod"], var.environment)
    error_message = "Environment must be dev, staging, or prod."
  }
}

# Optional variables พร้อม sensible defaults
variable "azs" {
  type        = list(string)
  description = "(Optional) Availability zones"
  default     = []
  # ถ้าว่าง จะใช้ data source ดึง AZs ของ region นั้น
}

variable "public_subnet_cidrs" {
  type        = list(string)
  description = "(Optional) CIDR blocks สำหรับ public subnets"
  default     = []
  # ถ้าว่าง จะคำนวณ CIDR จาก vpc_cidr อัตโนมัติ
}

variable "private_subnet_cidrs" {
  type        = list(string)
  description = "(Optional) CIDR blocks สำหรับ private subnets"
  default     = []
}

variable "enable_nat_gateway" {
  type        = bool
  description = "(Optional) เปิดใช้ NAT Gateway"
  default     = true
}

variable "single_nat_gateway" {
  type        = bool
  description = "(Optional) ใช้ NAT Gateway เดียวสำหรับทุก AZ (ลดค่าใช้จ่าย)"
  default     = false
}

variable "enable_flow_logs" {
  type        = bool
  description = "(Optional) เปิดใช้ VPC Flow Logs"
  default     = false
}

variable "tags" {
  type        = map(string)
  description = "(Optional) Tags เพิ่มเติม"
  default     = {}
}

# Feature flags
variable "features" {
  type = object({
    enable_ipv6         = optional(bool, false)
    enable_dns_support  = optional(bool, true)
    enable_dns_hostnames = optional(bool, true)
    enable_classiclink  = optional(bool, false)
  })
  description = "(Optional) Feature flags สำหรับ VPC"
  default     = {}
}
```

---

## Step 617: Complex Variable Examples - Full Patterns

### ตัวอย่างที่ครบถ้วน

```hcl
# ==========================================
# EXAMPLE 1: FULL SERVER FLEET DEFINITION
# ==========================================

variable "server_fleet" {
  type = map(object({
    # Server basics
    role          = string  # "web", "app", "worker", "bastion"
    instance_type = string
    count         = optional(number, 1)

    # Storage
    root_volume = optional(object({
      size_gb   = optional(number, 20)
      type      = optional(string, "gp3")
      encrypted = optional(bool, true)
    }), {})

    extra_volumes = optional(list(object({
      device_name = string
      size_gb     = number
      type        = optional(string, "gp3")
      encrypted   = optional(bool, true)
    })), [])

    # Network
    subnet_tier    = optional(string, "private")  # "public" or "private"
    security_groups = optional(list(string), [])
    assign_eip     = optional(bool, false)

    # IAM
    instance_profile = optional(string)
    iam_policies     = optional(list(string), [])

    # Monitoring
    enable_detailed_monitoring = optional(bool, false)
    enable_ssm                 = optional(bool, true)

    # Tags
    tags = optional(map(string), {})
  }))

  description = "Server fleet configuration"
  default     = {}
}

# ตัวอย่างค่า
# server_fleet = {
#   "web" = {
#     role          = "web"
#     instance_type = "t3.medium"
#     count         = 2
#     subnet_tier   = "public"
#     assign_eip    = false
#     tags = {
#       Tier = "frontend"
#     }
#   }
#   "app" = {
#     role          = "app"
#     instance_type = "c5.large"
#     count         = 3
#     extra_volumes = [
#       {
#         device_name = "/dev/xvdb"
#         size_gb     = 100
#       }
#     ]
#   }
#   "bastion" = {
#     role          = "bastion"
#     instance_type = "t3.micro"
#     count         = 1
#     subnet_tier   = "public"
#     assign_eip    = true
#   }
# }

# ==========================================
# EXAMPLE 2: APPLICATION CONFIGURATION
# ==========================================

variable "applications" {
  type = map(object({
    # Container image
    image   = string
    version = string

    # Compute
    cpu     = optional(number, 256)  # CPU units (256 = 0.25 vCPU)
    memory  = optional(number, 512)  # Memory in MB

    # Scaling
    min_capacity = optional(number, 1)
    max_capacity = optional(number, 10)

    # Ports
    container_port = optional(number, 8080)
    protocol       = optional(string, "tcp")

    # Health check
    health_check = optional(object({
      path                = optional(string, "/health")
      interval            = optional(number, 30)
      timeout             = optional(number, 5)
      healthy_threshold   = optional(number, 2)
      unhealthy_threshold = optional(number, 3)
      matcher             = optional(string, "200")
    }), {})

    # Environment variables
    environment = optional(map(string), {})
    secrets     = optional(map(string), {})  # Secret Manager ARNs

    # Logging
    log_retention_days = optional(number, 30)

    # Auto-scaling triggers
    scaling = optional(object({
      cpu_threshold    = optional(number, 70)
      memory_threshold = optional(number, 80)
      scale_in_cooldown  = optional(number, 300)
      scale_out_cooldown = optional(number, 60)
    }), {})
  }))
  description = "Application configurations"
  default     = {}
}

# ==========================================
# EXAMPLE 3: FULL PIPELINE CONFIGURATION
# ==========================================

variable "pipelines" {
  type = map(object({
    # Source
    source = object({
      type       = string  # "github", "codecommit", "s3"
      repository = optional(string)
      branch     = optional(string, "main")
      bucket     = optional(string)
      object_key = optional(string)
    })

    # Build stages
    stages = list(object({
      name    = string
      actions = list(object({
        name     = string
        category = string  # "Build", "Test", "Deploy", "Approval"
        provider = string  # "CodeBuild", "Manual", "ECS", etc.
        config   = map(string)

        # Run order within stage
        run_order = optional(number, 1)

        # Input/Output artifacts
        input_artifacts  = optional(list(string), [])
        output_artifacts = optional(list(string), [])
      }))
    }))

    # Notifications
    notifications = optional(object({
      events   = optional(list(string), ["FAILED"])
      sns_arns = optional(list(string), [])
    }), {})
  }))
  description = "CI/CD Pipeline configurations"
  default     = {}
}
```

---

## Step 618: Flattening Nested Variables with For Expressions

### การแปลง nested structures เป็น flat structures

```hcl
# ==========================================
# FLATTENING NESTED VARIABLES
# ==========================================

# ตัวแปรที่มี nested structure
variable "environments" {
  type = map(object({
    region    = string
    vpc_cidr  = string
    subnets   = list(string)
    instances = list(object({
      name = string
      type = string
    }))
  }))
  default = {
    "dev" = {
      region   = "ap-southeast-1"
      vpc_cidr = "10.0.0.0/16"
      subnets  = ["10.0.1.0/24", "10.0.2.0/24"]
      instances = [
        { name = "web-1", type = "t3.micro" },
        { name = "app-1", type = "t3.small" }
      ]
    }
    "prod" = {
      region   = "us-east-1"
      vpc_cidr = "10.1.0.0/16"
      subnets  = ["10.1.1.0/24", "10.1.2.0/24"]
      instances = [
        { name = "web-1", type = "c5.large" },
        { name = "web-2", type = "c5.large" },
        { name = "app-1", type = "c5.xlarge" }
      ]
    }
  }
}

locals {
  # Flatten nested instances across all environments
  # ผลลัพธ์: map ที่ key เป็น "env-name" และ value เป็น instance config
  all_instances = {
    for item in flatten([
      for env_name, env_config in var.environments : [
        for instance in env_config.instances : {
          key      = "${env_name}-${instance.name}"
          env      = env_name
          region   = env_config.region
          name     = instance.name
          type     = instance.type
        }
      ]
    ]) :
    item.key => item
  }
  # Result:
  # {
  #   "dev-web-1"  = { env = "dev", region = "ap-southeast-1", name = "web-1", type = "t3.micro" }
  #   "dev-app-1"  = { env = "dev", region = "ap-southeast-1", name = "app-1", type = "t3.small" }
  #   "prod-web-1" = { env = "prod", region = "us-east-1",     name = "web-1", type = "c5.large" }
  #   "prod-web-2" = { env = "prod", region = "us-east-1",     name = "web-2", type = "c5.large" }
  #   "prod-app-1" = { env = "prod", region = "us-east-1",     name = "app-1", type = "c5.xlarge" }
  # }

  # Flatten subnets across environments
  all_subnets = flatten([
    for env_name, env_config in var.environments : [
      for subnet in env_config.subnets : {
        env    = env_name
        region = env_config.region
        cidr   = subnet
      }
    ]
  ])
}

# ใช้ flattened data ใน resource
resource "aws_instance" "all" {
  for_each = local.all_instances

  ami           = data.aws_ami.ubuntu[each.value.region].id
  instance_type = each.value.type

  tags = {
    Name        = each.value.name
    Environment = each.value.env
  }
}

# ==========================================
# CONVERTING BETWEEN TYPES
# ==========================================

variable "routes_list" {
  type = list(object({
    destination = string
    gateway     = string
  }))
  default = [
    { destination = "0.0.0.0/0",    gateway = "igw-12345" },
    { destination = "10.0.0.0/8",   gateway = "vpn-12345" },
    { destination = "172.16.0.0/12", gateway = "tgw-12345" }
  ]
}

locals {
  # Convert list of objects to map (keyed by destination)
  routes_map = {
    for route in var.routes_list :
    route.destination => route.gateway
  }
  # Result:
  # {
  #   "0.0.0.0/0"      = "igw-12345"
  #   "10.0.0.0/8"     = "vpn-12345"
  #   "172.16.0.0/12"  = "tgw-12345"
  # }

  # Filter only specific routes
  internet_routes = {
    for dest, gw in local.routes_map :
    dest => gw
    if startswith(gw, "igw-")
  }
}
```

---

## Step 619: Working with Complex Defaults

### การจัดการ defaults ที่ซับซ้อน

```hcl
# ==========================================
# COMPLEX DEFAULTS PATTERNS
# ==========================================

# Pattern 1: locals สำหรับ computed defaults
variable "environment" {
  type    = string
  default = "dev"
}

variable "instance_type" {
  type    = string
  default = null  # null = ใช้ default จาก environment
}

locals {
  # Default instance types per environment
  default_instance_types = {
    dev     = "t3.micro"
    staging = "t3.medium"
    prod    = "c5.large"
  }

  # ใช้ var หรือ default ตาม environment
  actual_instance_type = coalesce(
    var.instance_type,
    local.default_instance_types[var.environment]
  )
}

# Pattern 2: Merging defaults with user-provided values
variable "user_tags" {
  type    = map(string)
  default = {}
}

locals {
  required_tags = {
    ManagedBy   = "Terraform"
    Environment = var.environment
    LastUpdated = formatdate("YYYY-MM-DD", timestamp())
  }

  # user_tags ชนะ required_tags ถ้ามี key เดียวกัน
  final_tags = merge(local.required_tags, var.user_tags)
}

# Pattern 3: Deep merge สำหรับ nested configs
variable "custom_monitoring" {
  type    = map(any)
  default = {}
}

locals {
  default_monitoring = {
    enabled         = true
    metrics_interval = 60
    alarms = {
      cpu_threshold    = 80
      memory_threshold = 80
      disk_threshold   = 85
    }
  }

  # ต้อง merge manually สำหรับ nested objects
  monitoring = {
    enabled          = lookup(var.custom_monitoring, "enabled", local.default_monitoring.enabled)
    metrics_interval = lookup(var.custom_monitoring, "metrics_interval", local.default_monitoring.metrics_interval)
    alarms = merge(
      local.default_monitoring.alarms,
      lookup(var.custom_monitoring, "alarms", {})
    )
  }
}

# Pattern 4: Environment-specific variable files
# โครงสร้าง:
# ├── variables.tf
# ├── terraform.tfvars      <- shared defaults
# ├── environments/
# │   ├── dev.tfvars
# │   ├── staging.tfvars
# │   └── prod.tfvars

# เรียกใช้:
# terraform apply -var-file=environments/prod.tfvars
```

---

## Step 620: Type Conversion Patterns & Best Practices

### Pattern ที่ใช้งานจริง

```hcl
# ==========================================
# PRACTICAL TYPE CONVERSION PATTERNS
# ==========================================

# Pattern 1: Converting instance_count to for_each compatible
variable "instance_names" {
  type    = list(string)
  default = ["web-1", "web-2", "app-1"]
}

locals {
  # Convert list to set for for_each
  instance_set = toset(var.instance_names)

  # Convert list to map with index
  instance_map = {
    for i, name in var.instance_names :
    name => {
      index = i
      name  = name
    }
  }
}

resource "aws_instance" "servers" {
  for_each = local.instance_set

  ami           = "ami-12345"
  instance_type = "t3.micro"

  tags = {
    Name = each.key
  }
}

# Pattern 2: Converting map to list
variable "environment_config" {
  type = map(object({
    vpc_cidr      = string
    instance_type = string
  }))
  default = {
    dev  = { vpc_cidr = "10.0.0.0/16", instance_type = "t3.micro" }
    prod = { vpc_cidr = "10.1.0.0/16", instance_type = "c5.large" }
  }
}

locals {
  # Get all VPC CIDRs as a list
  all_vpc_cidrs = values(var.environment_config)[*].vpc_cidr
  # Result: ["10.0.0.0/16", "10.1.0.0/16"]

  # Get all environments as a list of objects
  env_list = [
    for env_name, config in var.environment_config : {
      name          = env_name
      vpc_cidr      = config.vpc_cidr
      instance_type = config.instance_type
    }
  ]
}

# ==========================================
# BEST PRACTICES SUMMARY
# ==========================================

# 1. ใช้ object type แทน map(string) เมื่อโครงสร้างชัดเจน
# BAD:
variable "server_config_bad" {
  type = map(string)
  default = {
    instance_type = "t3.micro"
    disk_size_gb  = "20"  # ต้องใส่เป็น string แม้จะเป็น number
    enable_backup = "true" # ต้องใส่เป็น string แม้จะเป็น bool
  }
}

# GOOD:
variable "server_config_good" {
  type = object({
    instance_type = string
    disk_size_gb  = number
    enable_backup = bool
  })
  default = {
    instance_type = "t3.micro"
    disk_size_gb  = 20    # number จริงๆ
    enable_backup = true  # bool จริงๆ
  }
}

# 2. ใช้ optional() เพื่อลด boilerplate ในการระบุค่า
# 3. ใช้ validation blocks เพื่อ catch errors เร็ว
# 4. Document ทุก variable ด้วย description
# 5. ให้ default ที่สมเหตุสมผลเสมอ (เมื่อทำได้)

# ==========================================
# COMPLETE MODULE VARIABLE EXAMPLE
# ==========================================
# modules/ecs-service/variables.tf

variable "service_name" {
  type        = string
  description = "ชื่อของ ECS service"

  validation {
    condition     = can(regex("^[a-z0-9-]{3,32}$", var.service_name))
    error_message = "Service name must be 3-32 lowercase alphanumeric characters or hyphens."
  }
}

variable "cluster_arn" {
  type        = string
  description = "ARN ของ ECS cluster"

  validation {
    condition     = can(regex("^arn:aws:ecs:", var.cluster_arn))
    error_message = "Cluster ARN must be a valid ECS cluster ARN."
  }
}

variable "task_definition" {
  type = object({
    cpu    = number
    memory = number
    containers = list(object({
      name      = string
      image     = string
      essential = optional(bool, true)
      port_mappings = optional(list(object({
        container_port = number
        host_port      = optional(number)
        protocol       = optional(string, "tcp")
      })), [])
      environment = optional(map(string), {})
      secrets     = optional(map(string), {})
      log_group   = optional(string)
      command     = optional(list(string))
      health_check = optional(object({
        command     = list(string)
        interval    = optional(number, 30)
        timeout     = optional(number, 5)
        retries     = optional(number, 3)
        start_period = optional(number, 0)
      }))
    }))
  })
  description = "Task definition configuration"

  validation {
    condition     = var.task_definition.cpu >= 256
    error_message = "Task CPU must be at least 256 units (0.25 vCPU)."
  }

  validation {
    condition     = var.task_definition.memory >= 512
    error_message = "Task memory must be at least 512 MB."
  }

  validation {
    condition     = length([for c in var.task_definition.containers : c if c.essential]) >= 1
    error_message = "At least one container must be marked as essential."
  }
}

variable "desired_count" {
  type        = number
  description = "จำนวน task ที่ต้องการ"
  default     = 1

  validation {
    condition     = var.desired_count >= 0
    error_message = "Desired count must be non-negative."
  }
}

variable "load_balancer" {
  type = optional(object({
    target_group_arn = string
    container_name   = string
    container_port   = number
  }))
  description = "Load balancer configuration (optional)"
  default     = null
}

variable "tags" {
  type        = map(string)
  description = "Tags สำหรับ resources"
  default     = {}
}
```

---

## สรุป (Summary)

### เมื่อใช้ type แบบไหน:

| Scenario | Type ที่แนะนำ |
|----------|--------------|
| Fixed set ของ attributes | `object({...})` |
| Dynamic set ของ named items | `map(...)` |
| Ordered sequence ของ items | `list(...)` |
| Unique unordered set | `set(...)` |
| Mixed/unknown structure | `any` |
| Multiple resources ชนิดเดียวกัน | `map(object({...}))` |
| Ordered rules/policies | `list(object({...}))` |

### Optional Attributes (Terraform 1.3+):
- ใช้ `optional(type)` เพื่อทำให้ attribute ไม่จำเป็น
- ใช้ `optional(type, default)` เพื่อกำหนด default value
- ช่วยลด verbosity ในการกำหนดค่า

---

*จบ Part 062 - Complex Variable Types & Structures*
