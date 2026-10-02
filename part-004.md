# Part 004: HCL Data Types: Complex Types
## ประเภทข้อมูลซับซ้อนใน HCL (Steps 31-40)

---

## Step 31: list(type) - Ordered Sequence

### พื้นฐาน List

```hcl
# List declaration ใน variables
variable "availability_zones" {
  description = "รายการ Availability Zones"
  type        = list(string)
  default     = ["ap-southeast-1a", "ap-southeast-1b", "ap-southeast-1c"]
}

variable "allowed_ports" {
  description = "Ports ที่อนุญาต"
  type        = list(number)
  default     = [80, 443, 8080, 8443]
}

variable "enable_features" {
  description = "Feature flags"
  type        = list(bool)
  default     = [true, false, true]
}
```

### List Operations

```hcl
locals {
  fruits = ["apple", "banana", "cherry", "date", "elderberry"]
  nums   = [5, 3, 1, 4, 2]
  
  # ความยาว
  count = length(local.fruits)  # 5
  
  # Index access (0-based)
  first  = local.fruits[0]   # "apple"
  second = local.fruits[1]   # "banana"
  last   = local.fruits[4]   # "elderberry"
  
  # Sort
  sorted_fruits = sort(local.fruits)  # alphabetically sorted
  sorted_nums   = sort(local.nums)    # [1, 2, 3, 4, 5]
  
  # Reverse
  reversed = reverse(local.fruits)  # ["elderberry", "date", "cherry", "banana", "apple"]
  
  # Contains
  has_apple  = contains(local.fruits, "apple")   # true
  has_grape  = contains(local.fruits, "grape")   # false
  
  # Concat
  more_fruits = concat(local.fruits, ["fig", "grape"])
  
  # Distinct (remove duplicates)
  with_dups  = ["a", "b", "a", "c", "b", "d"]
  unique_vals = distinct(local.with_dups)  # ["a", "b", "c", "d"]
  
  # Flatten
  nested_list = [["a", "b"], ["c", "d"], ["e"]]
  flat_list   = flatten(local.nested_list)  # ["a", "b", "c", "d", "e"]
  
  # Slice (จาก index, ถึง index - ไม่รวมตัวสุดท้าย)
  first_three = slice(local.fruits, 0, 3)  # ["apple", "banana", "cherry"]
  
  # Chunklist
  chunked = chunklist(local.fruits, 2)  # [["apple", "banana"], ["cherry", "date"], ["elderberry"]]
  
  # Index ของ element
  banana_index = index(local.fruits, "banana")  # 1
}
```

### List ใน Real-world Examples

```hcl
# VPC Subnets
variable "public_subnets" {
  type    = list(string)
  default = ["10.0.1.0/24", "10.0.2.0/24", "10.0.3.0/24"]
}

variable "private_subnets" {
  type    = list(string)
  default = ["10.0.11.0/24", "10.0.12.0/24", "10.0.13.0/24"]
}

# สร้าง subnets จาก list
resource "aws_subnet" "public" {
  count             = length(var.public_subnets)
  vpc_id            = aws_vpc.main.id
  cidr_block        = var.public_subnets[count.index]
  availability_zone = var.availability_zones[count.index]
  
  tags = {
    Name = "public-subnet-${count.index + 1}"
    Type = "public"
  }
}

# Security Group rules จาก list
variable "allowed_ingress_ports" {
  type    = list(number)
  default = [80, 443]
}

resource "aws_security_group" "web" {
  name   = "web-sg"
  vpc_id = aws_vpc.main.id
  
  dynamic "ingress" {
    for_each = var.allowed_ingress_ports
    content {
      from_port   = ingress.value
      to_port     = ingress.value
      protocol    = "tcp"
      cidr_blocks = ["0.0.0.0/0"]
    }
  }
}
```

---

## Step 32: set(type) - Unique Unordered Collection

### พื้นฐาน Set

```hcl
# Set declaration
variable "enabled_regions" {
  description = "AWS regions ที่เปิดใช้งาน"
  type        = set(string)
  default     = ["ap-southeast-1", "us-east-1", "eu-west-1"]
}

variable "admin_usernames" {
  description = "รายชื่อ admin users"
  type        = set(string)
  default     = ["alice", "bob", "charlie"]
}
```

### Set vs List ความแตกต่าง

```hcl
locals {
  # List - ลำดับสำคัญ, อาจมี duplicate
  list_example = ["a", "b", "a", "c"]  # ["a", "b", "a", "c"]
  
  # Set - ไม่มี duplicate, ไม่มีลำดับที่แน่นอน
  set_example  = toset(["a", "b", "a", "c"])  # {"a", "b", "c"}
  
  # แปลง list → set (ลบ duplicates)
  tags_list     = ["web", "prod", "web", "app", "prod"]
  tags_set      = toset(local.tags_list)  # {"web", "prod", "app"}
  
  # แปลง set → list
  regions_set   = toset(["eu-west-1", "us-east-1", "ap-southeast-1"])
  regions_list  = tolist(local.regions_set)  # list แต่ลำดับไม่แน่นอน
}
```

### Set Operations

```hcl
locals {
  set_a = toset(["apple", "banana", "cherry"])
  set_b = toset(["banana", "cherry", "date"])
  
  # Intersection - elements ที่อยู่ทั้งคู่
  intersection = setintersection(local.set_a, local.set_b)
  # {"banana", "cherry"}
  
  # Union - elements ทั้งหมดจากทั้งคู่
  union = setunion(local.set_a, local.set_b)
  # {"apple", "banana", "cherry", "date"}
  
  # Subtract - elements ใน a แต่ไม่ใน b
  difference = setsubtract(local.set_a, local.set_b)
  # {"apple"}
  
  # Symmetric difference
  all = setunion(local.set_a, local.set_b)
  both = setintersection(local.set_a, local.set_b)
  sym_diff = setsubtract(local.all, local.both)
  # {"apple", "date"} - อยู่ใน set หนึ่ง แต่ไม่ใช่ทั้งคู่
}
```

### Set ใน for_each

```hcl
# Set เหมาะมากสำหรับใช้กับ for_each
variable "s3_buckets" {
  description = "S3 buckets ที่ต้องสร้าง"
  type        = set(string)
  default     = ["data", "logs", "backups", "artifacts"]
}

resource "aws_s3_bucket" "buckets" {
  for_each = var.s3_buckets
  
  bucket = "my-company-${each.key}"  # each.key = bucket name
  
  tags = {
    Name    = each.key
    Purpose = each.key
  }
}

# ผลลัพธ์:
# aws_s3_bucket.buckets["data"]
# aws_s3_bucket.buckets["logs"]
# aws_s3_bucket.buckets["backups"]
# aws_s3_bucket.buckets["artifacts"]
```

---

## Step 33: map(type) - Key-Value Pairs

### พื้นฐาน Map

```hcl
# Map declaration
variable "tags" {
  description = "Resource tags"
  type        = map(string)
  default = {
    Environment = "production"
    Project     = "myapp"
    Team        = "platform"
  }
}

variable "instance_types" {
  description = "Instance types สำหรับแต่ละ environment"
  type        = map(string)
  default = {
    dev     = "t3.micro"
    staging = "t3.small"
    prod    = "t3.large"
  }
}

variable "az_config" {
  description = "Configuration สำหรับแต่ละ AZ"
  type        = map(number)
  default = {
    "ap-southeast-1a" = 1
    "ap-southeast-1b" = 2
    "ap-southeast-1c" = 3
  }
}
```

### Map Operations

```hcl
locals {
  config = {
    host     = "db.example.com"
    port     = "5432"
    database = "myapp"
    ssl      = "true"
  }
  
  # Get keys
  config_keys = keys(local.config)    # ["database", "host", "port", "ssl"]
  
  # Get values
  config_vals = values(local.config)  # ["myapp", "db.example.com", "5432", "true"]
  
  # Lookup with default
  host    = lookup(local.config, "host", "localhost")     # "db.example.com"
  timeout = lookup(local.config, "timeout", "30")         # "30" (default)
  
  # Merge maps
  base_tags = {
    ManagedBy   = "terraform"
    Environment = "prod"
  }
  
  extra_tags = {
    Project = "myapp"
    Team    = "platform"
  }
  
  merged = merge(local.base_tags, local.extra_tags)
  # {
  #   ManagedBy   = "terraform"
  #   Environment = "prod"
  #   Project     = "myapp"
  #   Team        = "platform"
  # }
  
  # Override keys (keys ใน map ทีหลังจะ override)
  overridden = merge(
    { Environment = "staging" },
    { Environment = "prod" }  # นี่จะ override
  )
  # { Environment = "prod" }
  
  # zipmap - สร้าง map จาก 2 lists
  names  = ["alice", "bob", "charlie"]
  ages   = ["30", "25", "35"]
  people = zipmap(local.names, local.ages)
  # { alice = "30", bob = "25", charlie = "35" }
}
```

### Map ใน Real-world

```hcl
# Environment-specific configuration
variable "env_config" {
  type = map(object({
    instance_type = string
    min_size      = number
    max_size      = number
    desired_size  = number
  }))
  
  default = {
    dev = {
      instance_type = "t3.micro"
      min_size      = 1
      max_size      = 2
      desired_size  = 1
    }
    staging = {
      instance_type = "t3.small"
      min_size      = 2
      max_size      = 4
      desired_size  = 2
    }
    prod = {
      instance_type = "t3.large"
      min_size      = 3
      max_size      = 10
      desired_size  = 5
    }
  }
}

# ใช้ map สำหรับ current environment
locals {
  current_config = var.env_config[var.environment]
}

resource "aws_autoscaling_group" "web" {
  name             = "web-asg-${var.environment}"
  max_size         = local.current_config.max_size
  min_size         = local.current_config.min_size
  desired_capacity = local.current_config.desired_size
}
```

---

## Step 34: object({...}) - Structured Attributes

### พื้นฐาน Object

```hcl
# Object type - กำหนด structure ชัดเจน
variable "database" {
  description = "Database configuration"
  type = object({
    engine         = string
    version        = string
    instance_class = string
    storage_gb     = number
    multi_az       = bool
  })
  
  default = {
    engine         = "postgres"
    version        = "14.9"
    instance_class = "db.t3.micro"
    storage_gb     = 20
    multi_az       = false
  }
}

variable "networking" {
  description = "Network configuration"
  type = object({
    vpc_cidr        = string
    public_subnets  = list(string)
    private_subnets = list(string)
    enable_nat      = bool
    enable_vpn      = bool
  })
}
```

### Optional Fields ใน Object

```hcl
# Optional fields (Terraform 1.3+)
variable "server_config" {
  type = object({
    name          = string
    instance_type = string
    ami_id        = string
    
    # Optional fields with defaults
    key_pair      = optional(string, null)
    user_data     = optional(string, "")
    monitoring    = optional(bool, false)
    
    # Optional nested object
    tags = optional(map(string), {})
    
    # Optional with nested object
    ebs_config = optional(object({
      volume_size = optional(number, 20)
      volume_type = optional(string, "gp3")
      encrypted   = optional(bool, true)
    }), null)
  })
}
```

### Object Attribute Access

```hcl
locals {
  server = {
    name          = "web-server"
    instance_type = "t3.micro"
    ami_id        = "ami-12345"
    tags = {
      Name = "web-server"
    }
  }
  
  # Access attributes
  server_name = local.server.name           # "web-server"
  server_type = local.server.instance_type  # "t3.micro"
  
  # Nested access
  name_tag = local.server.tags.Name  # "web-server"
  
  # Dynamic attribute access ด้วย lookup
  dynamic_attr = lookup(local.server, "ami_id", "unknown")  # "ami-12345"
}
```

### Complex Object Example

```hcl
variable "microservices" {
  description = "Microservices configuration"
  type = map(object({
    image         = string
    cpu           = number
    memory        = number
    port          = number
    replicas      = number
    health_check  = object({
      path     = string
      interval = number
      timeout  = number
    })
    env_vars = optional(map(string), {})
    secrets  = optional(list(string), [])
  }))
  
  default = {
    api = {
      image    = "my-api:latest"
      cpu      = 256
      memory   = 512
      port     = 8080
      replicas = 3
      health_check = {
        path     = "/health"
        interval = 30
        timeout  = 5
      }
      env_vars = {
        LOG_LEVEL = "info"
        PORT      = "8080"
      }
    }
    
    worker = {
      image    = "my-worker:latest"
      cpu      = 512
      memory   = 1024
      port     = 9090
      replicas = 2
      health_check = {
        path     = "/status"
        interval = 60
        timeout  = 10
      }
    }
  }
}
```

---

## Step 35: tuple([type, ...]) - Fixed-length Sequence

### พื้นฐาน Tuple

```hcl
# Tuple - ความยาวคงที่, แต่ละ element มี type ต่างกันได้
locals {
  # Tuple ไม่มี named type declaration แบบ variable
  # แต่สามารถกำหนดใน variable type
  
  # Simple tuple
  point    = [1.5, 2.7]       # [number, number]
  
  # Mixed type tuple
  server_info = ["web-01", "t3.micro", 8080, true]
  # tuple([string, string, number, bool])
}

variable "cidr_range" {
  description = "CIDR range as [network, prefix_length]"
  type        = tuple([string, number])
  default     = ["10.0.0.0", 16]
}
```

### Tuple vs List

```
Comparison:
┌────────────────────────────────────────────────────────────┐
│  Tuple                          │  List                    │
│─────────────────────────────────┼──────────────────────────│
│  Fixed length                   │  Variable length         │
│  Mixed types allowed            │  Single type             │
│  Access by index                │  Access by index         │
│  [string, number, bool]         │  list(string)            │
│  Cannot add/remove elements     │  Can concat, etc.        │
└────────────────────────────────────────────────────────────┘
```

### Tuple ใน HCL

```hcl
# Tuple ใน for expressions
locals {
  server_configs = [
    ["web-01", "t3.micro", 80],
    ["web-02", "t3.micro", 80],
    ["api-01", "t3.small", 8080],
  ]
  
  # Destructure tuple
  servers = [for config in local.server_configs : {
    name          = config[0]
    instance_type = config[1]
    port          = config[2]
  }]
}
```

---

## Step 36: any type

### การใช้ any

```hcl
# any - ยอมรับ type ใดก็ได้
variable "flexible_config" {
  description = "Flexible configuration value"
  type        = any
  default     = null
}

variable "mixed_values" {
  description = "Can be string or number"
  type        = any
}
```

### ตัวอย่างการใช้ any

```hcl
# Module ที่รับ config แบบยืดหยุ่น
variable "module_config" {
  type = object({
    name     = string
    settings = any  # ยอมรับ config ใดก็ได้
  })
}

# การส่ง any ไปยัง module
module "flexible_app" {
  source = "./modules/app"
  
  module_config = {
    name = "my-app"
    settings = {
      debug   = true
      port    = 8080
      origins = ["https://app.example.com"]
    }
  }
}
```

### ข้อควรระวัง any

```hcl
# ⚠️ any ทำให้ lose type safety
# ✅ ใช้ any เมื่อ:
# - Type ไม่แน่นอน หรือ vary ตาม context
# - Module ต้องรับ config หลายรูปแบบ
# - ต้องการ backward compatibility

# ❌ หลีกเลี่ยง any เมื่อ:
# - รู้ type แน่นอน → ใช้ type นั้นไปเลย
# - ต้องการ validation → ใช้ specific types + validation blocks
```

---

## Step 37: Type Nesting Examples

### Nested Complex Types

```hcl
# List of objects
variable "servers" {
  description = "Server configurations"
  type = list(object({
    name          = string
    instance_type = string
    tags          = map(string)
    subnets       = list(string)
  }))
  
  default = [
    {
      name          = "web-01"
      instance_type = "t3.micro"
      tags = {
        Role = "web"
        Zone = "public"
      }
      subnets = ["subnet-aaa", "subnet-bbb"]
    },
    {
      name          = "api-01"
      instance_type = "t3.small"
      tags = {
        Role = "api"
        Zone = "private"
      }
      subnets = ["subnet-ccc", "subnet-ddd"]
    }
  ]
}

# Map of lists
variable "security_group_rules" {
  description = "Security group ingress rules per environment"
  type        = map(list(number))
  default = {
    dev  = [80, 443, 8080, 3000]
    prod = [80, 443]
  }
}

# Map of objects
variable "environments" {
  type = map(object({
    cidr            = string
    instance_count  = number
    instance_type   = string
    enable_ha       = bool
  }))
  
  default = {
    dev = {
      cidr           = "10.1.0.0/16"
      instance_count = 1
      instance_type  = "t3.micro"
      enable_ha      = false
    }
    prod = {
      cidr           = "10.0.0.0/16"
      instance_count = 3
      instance_type  = "t3.large"
      enable_ha      = true
    }
  }
}

# Object with nested object and list
variable "application" {
  type = object({
    name    = string
    version = string
    
    database = object({
      engine   = string
      version  = string
      replicas = number
    })
    
    cache = optional(object({
      engine    = string
      node_type = string
      nodes     = number
    }), null)
    
    services = list(object({
      name     = string
      port     = number
      replicas = number
    }))
    
    allowed_origins = list(string)
    
    feature_flags = map(bool)
  })
}
```

---

## Step 38: Type Conversion ระหว่าง Complex Types

### แปลง List ↔ Set ↔ Map

```hcl
locals {
  # List with duplicates
  raw_list = ["us-east-1", "ap-southeast-1", "us-east-1", "eu-west-1"]
  
  # List → Set (remove duplicates)
  unique_regions = toset(local.raw_list)
  # {"ap-southeast-1", "eu-west-1", "us-east-1"}
  
  # Set → List
  regions_list = tolist(local.unique_regions)
  # ["ap-southeast-1", "eu-west-1", "us-east-1"] (sorted)
  
  # List → Map (using for expression)
  keys   = ["name", "age", "city"]
  values = ["Alice", "30", "Bangkok"]
  person_map = zipmap(local.keys, local.values)
  # { name = "Alice", age = "30", city = "Bangkok" }
  
  # Map → List of keys
  config = { a = 1, b = 2, c = 3 }
  key_list = keys(local.config)    # ["a", "b", "c"]
  val_list = values(local.config)  # [1, 2, 3]
  
  # Map → List of tuples (using for expression)
  map_as_tuples = [for k, v in local.config : [k, v]]
  # [["a", 1], ["b", 2], ["c", 3]]
  
  # List of objects → Map
  servers = [
    { name = "web-01", ip = "10.0.1.1" },
    { name = "web-02", ip = "10.0.1.2" },
    { name = "api-01", ip = "10.0.2.1" },
  ]
  
  server_map = {
    for server in local.servers :
    server.name => server.ip
  }
  # { web-01 = "10.0.1.1", web-02 = "10.0.1.2", api-01 = "10.0.2.1" }
}
```

### Flatten Nested Lists

```hcl
locals {
  # Nested list structure
  regions_and_azs = {
    "ap-southeast-1" = ["ap-southeast-1a", "ap-southeast-1b", "ap-southeast-1c"]
    "us-east-1"      = ["us-east-1a", "us-east-1b", "us-east-1c"]
  }
  
  # Flatten to list of objects
  all_azs = flatten([
    for region, azs in local.regions_and_azs : [
      for az in azs : {
        region = region
        az     = az
      }
    ]
  ])
  # [
  #   { region = "ap-southeast-1", az = "ap-southeast-1a" },
  #   { region = "ap-southeast-1", az = "ap-southeast-1b" },
  #   ...
  # ]
  
  # Convert to map for for_each
  az_map = {
    for obj in local.all_azs :
    "${obj.region}-${obj.az}" => obj
  }
}
```

---

## Step 39: Working with null in Complex Types

### Null ใน Complex Types

```hcl
# Optional nested objects
variable "monitoring_config" {
  description = "Monitoring configuration"
  type = object({
    enabled     = bool
    cloudwatch  = optional(object({
      log_group    = string
      retention    = number
    }), null)
    datadog = optional(object({
      api_key     = string
      site        = string
    }), null)
  })
  
  default = {
    enabled    = true
    cloudwatch = {
      log_group = "/app/myapp"
      retention = 30
    }
    datadog = null  # ไม่ใช้ Datadog
  }
}

# ตรวจสอบ null ก่อนใช้
locals {
  enable_cloudwatch = var.monitoring_config.enabled && var.monitoring_config.cloudwatch != null
  enable_datadog    = var.monitoring_config.enabled && var.monitoring_config.datadog != null
  
  # ใช้ try() สำหรับ safe access
  cw_log_group = try(var.monitoring_config.cloudwatch.log_group, "/default/log")
  dd_api_key   = try(var.monitoring_config.datadog.api_key, null)
}

# สร้าง resource เฉพาะเมื่อไม่ใช่ null
resource "aws_cloudwatch_log_group" "app" {
  count = local.enable_cloudwatch ? 1 : 0
  
  name              = local.cw_log_group
  retention_in_days = var.monitoring_config.cloudwatch.retention
}
```

### Null ใน Maps และ Lists

```hcl
locals {
  # Filter null values จาก list
  values_with_nulls = ["a", null, "b", null, "c"]
  
  # ใช้ compact() เพื่อลบ null strings (ใช้กับ list(string))
  # compact ลบ empty strings ด้วย
  non_null_values = compact(local.values_with_nulls)  # ["a", "b", "c"]
  
  # Filter null จาก list ด้วย for expression
  filtered = [
    for v in local.values_with_nulls : v
    if v != null
  ]  # ["a", "b", "c"]
  
  # Map ที่มี null values
  config_map = {
    host    = "db.example.com"
    port    = "5432"
    replica = null  # optional
    backup  = null  # optional
  }
  
  # Filter out null values จาก map
  active_config = {
    for k, v in local.config_map : k => v
    if v != null
  }
  # { host = "db.example.com", port = "5432" }
}
```

---

## Step 40: Complex Type Validation

### Variable Validation

```hcl
variable "subnets" {
  description = "Subnet configurations"
  type = list(object({
    cidr = string
    az   = string
    type = string
  }))
  
  validation {
    condition = length(var.subnets) >= 2
    error_message = "Must have at least 2 subnets for HA."
  }
  
  validation {
    condition = alltrue([
      for subnet in var.subnets :
      can(cidrnetmask(subnet.cidr))
    ])
    error_message = "All subnet CIDRs must be valid."
  }
  
  validation {
    condition = alltrue([
      for subnet in var.subnets :
      contains(["public", "private"], subnet.type)
    ])
    error_message = "Subnet type must be 'public' or 'private'."
  }
}

variable "tags" {
  type = map(string)
  
  validation {
    condition = contains(keys(var.tags), "Environment")
    error_message = "Tags must include 'Environment' key."
  }
  
  validation {
    condition = length(var.tags) <= 50
    error_message = "Cannot have more than 50 tags."
  }
  
  validation {
    condition = alltrue([
      for k, v in var.tags :
      length(k) <= 128 && length(v) <= 256
    ])
    error_message = "Tag keys max 128 chars, values max 256 chars."
  }
}

variable "microservices" {
  type = map(object({
    port     = number
    replicas = number
    cpu      = number
    memory   = number
  }))
  
  validation {
    condition = alltrue([
      for name, svc in var.microservices :
      svc.port >= 1024 && svc.port <= 65535
    ])
    error_message = "All service ports must be between 1024 and 65535."
  }
  
  validation {
    condition = alltrue([
      for name, svc in var.microservices :
      svc.replicas >= 1 && svc.replicas <= 100
    ])
    error_message = "Replicas must be between 1 and 100."
  }
  
  validation {
    condition = alltrue([
      for name, svc in var.microservices :
      contains([256, 512, 1024, 2048, 4096], svc.cpu)
    ])
    error_message = "CPU must be one of: 256, 512, 1024, 2048, 4096."
  }
}
```

### Complete Real-world Example

```hcl
# variables.tf สำหรับ VPC Module

variable "vpc_config" {
  description = "VPC Configuration"
  type = object({
    cidr    = string
    name    = string
    region  = string
    
    public_subnets = list(object({
      cidr = string
      az   = string
      name = optional(string, null)
    }))
    
    private_subnets = list(object({
      cidr = string
      az   = string
      name = optional(string, null)
    }))
    
    enable_nat_gateway     = optional(bool, true)
    single_nat_gateway     = optional(bool, false)
    enable_vpn_gateway     = optional(bool, false)
    enable_flow_logs       = optional(bool, true)
    flow_logs_retention    = optional(number, 30)
    
    tags = optional(map(string), {})
  })
  
  validation {
    condition     = can(cidrnetmask(var.vpc_config.cidr))
    error_message = "VPC CIDR must be a valid CIDR block."
  }
  
  validation {
    condition     = length(var.vpc_config.public_subnets) >= 2
    error_message = "Must have at least 2 public subnets for ALB."
  }
  
  validation {
    condition     = length(var.vpc_config.private_subnets) >= 2
    error_message = "Must have at least 2 private subnets for HA."
  }
  
  validation {
    condition = alltrue([
      for subnet in concat(var.vpc_config.public_subnets, var.vpc_config.private_subnets) :
      can(cidrnetmask(subnet.cidr))
    ])
    error_message = "All subnet CIDRs must be valid."
  }
}

# ตัวอย่างค่าที่ส่งเข้า
# terraform.tfvars

vpc_config = {
  cidr   = "10.0.0.0/16"
  name   = "production-vpc"
  region = "ap-southeast-1"
  
  public_subnets = [
    { cidr = "10.0.1.0/24", az = "ap-southeast-1a", name = "public-1a" },
    { cidr = "10.0.2.0/24", az = "ap-southeast-1b", name = "public-1b" },
    { cidr = "10.0.3.0/24", az = "ap-southeast-1c", name = "public-1c" },
  ]
  
  private_subnets = [
    { cidr = "10.0.11.0/24", az = "ap-southeast-1a", name = "private-1a" },
    { cidr = "10.0.12.0/24", az = "ap-southeast-1b", name = "private-1b" },
    { cidr = "10.0.13.0/24", az = "ap-southeast-1c", name = "private-1c" },
  ]
  
  enable_nat_gateway  = true
  single_nat_gateway  = false
  enable_flow_logs    = true
  flow_logs_retention = 90
  
  tags = {
    CostCenter = "platform"
    Owner      = "infrastructure-team"
  }
}
```

---

## สรุป Complex Types

### Quick Reference

| Type | Syntax | Use Case |
|------|--------|----------|
| `list(T)` | `["a", "b", "c"]` | Ordered items, index access, count |
| `set(T)` | `toset(["a", "b"])` | Unique items, for_each |
| `map(T)` | `{key = "value"}` | Key-value lookup |
| `object({})` | `{name = "x", port = 80}` | Structured config |
| `tuple([])` | `["x", 80, true]` | Mixed-type fixed sequence |
| `any` | anything | Flexible, untyped |

### เมื่อไหรควรใช้อะไร

```
Decision Guide:
├── ต้องการ ordered? → list
├── ต้องการ unique? → set
├── ต้องการ key-value? → map
├── ต้องการ structure ชัดเจน? → object
├── Mixed types, fixed length? → tuple
└── ไม่รู้ type? → any (use sparingly)
```

💡 **Pro Tips:**
- ใช้ `object()` แทน `map(any)` เมื่อ fields ชัดเจน
- ใช้ `set` สำหรับ for_each เพื่อ stable ordering
- ใช้ `optional()` ใน object type สำหรับ backward compatibility
- ใช้ `compact()` เพื่อลบ empty strings จาก list

⚠️ **Common Mistakes:**
- ใช้ `list` แต่ควรใช้ `set` กับ `for_each`
- ไม่ validate complex types ที่รับ user input
- ใช้ `any` แทนที่จะ define proper types
- ลืม `optional()` ทำให้ต้องใส่ค่าทุก field

---

*ก่อนหน้า: [Part 003 - HCL Data Types: Primitives](part-003.md)*
*ต่อไป: [Part 005 - HCL Expressions และ Operators](part-005.md)*
