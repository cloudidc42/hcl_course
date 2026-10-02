# Part 005: HCL Expressions และ Operators
## นิพจน์และตัวดำเนินการใน HCL (Steps 41-50)

---

## Step 41: Arithmetic Operators

### ตัวดำเนินการทางคณิตศาสตร์

```hcl
locals {
  a = 10
  b = 3
  
  # บวก (+)
  sum       = local.a + local.b    # 13
  
  # ลบ (-)
  diff      = local.a - local.b    # 7
  
  # คูณ (*)
  product   = local.a * local.b    # 30
  
  # หาร (/)
  quotient  = local.a / local.b    # 3.3333...
  
  # เศษจากการหาร (%)
  remainder = local.a % local.b    # 1
  
  # ลบ unary
  negative  = -local.a             # -10
}
```

### Arithmetic ใน Real-world

```hcl
variable "base_port" {
  type    = number
  default = 8000
}

variable "service_count" {
  type    = number
  default = 5
}

variable "storage_gb" {
  type    = number
  default = 100
}

locals {
  # คำนวณ ports
  api_port     = var.base_port + 0    # 8000
  admin_port   = var.base_port + 1    # 8001
  metrics_port = var.base_port + 2    # 8002
  
  # คำนวณ storage
  storage_bytes = var.storage_gb * 1024 * 1024 * 1024  # bytes
  storage_mb    = var.storage_gb * 1024                  # MB
  
  # คำนวณ costs (ตัวอย่าง)
  hourly_cost   = 0.023  # per GB per month → per hour
  monthly_cost  = local.storage_gb * local.hourly_cost * 24 * 30
  
  # CIDR calculations
  vpc_cidr_parts = split(".", split("/", var.vpc_cidr)[0])
  vpc_first_octet = tonumber(local.vpc_cidr_parts[0])
  
  # Subnet index calculations
  subnet_offsets = [for i in range(var.service_count) : i * 256]
}

# Auto-scaling calculations
resource "aws_autoscaling_policy" "scale_out" {
  name                   = "scale-out"
  autoscaling_group_name = aws_autoscaling_group.web.name
  adjustment_type        = "ChangeInCapacity"
  scaling_adjustment     = ceil(aws_autoscaling_group.web.desired_capacity * 0.25)  # 25% increase
  cooldown               = 300
}
```

### Integer Division และ Modulo

```hcl
locals {
  # Integer division (HCL ทำ float division)
  ten_div_three  = 10 / 3     # 3.3333... (float)
  floor_division = floor(10 / 3)  # 3 (integer)
  
  # Modulo สำหรับ even/odd check
  is_even = 10 % 2 == 0  # true
  is_odd  = 7 % 2 == 1   # true
  
  # Modulo สำหรับ cycle ผ่าน list
  items = ["a", "b", "c"]
  # รับ element ด้วย circular index
  # element_at_index_4 = local.items[4 % length(local.items)]  # "b"
  
  # Percentage calculations
  cpu_cores   = 8
  reserve_pct = 20
  reserved    = floor(local.cpu_cores * local.reserve_pct / 100)  # 1 core
  available   = local.cpu_cores - local.reserved  # 7 cores
}
```

---

## Step 42: Comparison Operators

### ตัวดำเนินการเปรียบเทียบ

```hcl
locals {
  a = 10
  b = 20
  x = "hello"
  y = "world"
  
  # Equal (==)
  eq_nums    = local.a == local.b   # false
  eq_strs    = local.x == "hello"   # true
  
  # Not Equal (!=)
  ne_nums    = local.a != local.b   # true
  ne_strs    = local.x != local.y   # true
  
  # Less Than (<)
  lt         = local.a < local.b    # true
  
  # Greater Than (>)
  gt         = local.a > local.b    # false
  
  # Less Than or Equal (<=)
  lte        = local.a <= 10        # true
  
  # Greater Than or Equal (>=)
  gte        = local.a >= 10        # true
}
```

### Comparison ใน Variables Validation

```hcl
variable "instance_count" {
  type    = number
  default = 3
  
  validation {
    condition     = var.instance_count >= 1 && var.instance_count <= 100
    error_message = "Instance count must be between 1 and 100."
  }
}

variable "port" {
  type = number
  
  validation {
    condition     = var.port > 1024 && var.port <= 65535
    error_message = "Port must be between 1025 and 65535."
  }
}

variable "environment" {
  type = string
  
  validation {
    condition     = var.environment == "prod" || var.environment == "dev" || var.environment == "staging"
    error_message = "Environment must be prod, dev, or staging."
  }
}
```

### String Comparison

```hcl
locals {
  env = "production"
  
  # String equality
  is_prod    = local.env == "production"   # true
  is_dev     = local.env == "development"  # false
  not_prod   = local.env != "production"   # false
  
  # String comparison (lexicographic)
  # "apple" < "banana" → true
  # "z" > "a" → true
  
  # Case-insensitive comparison
  lower_env  = lower(local.env)
  is_prod_ci = lower(local.env) == "production"  # case-insensitive
}
```

---

## Step 43: Logical Operators

### ตัวดำเนินการตรรกะ

```hcl
locals {
  is_prod    = true
  is_staging = false
  debug_mode = false
  
  # AND (&&)
  prod_no_debug = local.is_prod && !local.debug_mode  # true
  
  # OR (||)
  any_env    = local.is_prod || local.is_staging  # true
  
  # NOT (!)
  not_prod   = !local.is_prod   # false
  
  # Complex logical
  needs_ha         = local.is_prod || (local.is_staging && var.force_ha)
  enable_verbose   = local.debug_mode || local.is_staging
  skip_tests       = !var.run_tests && !local.is_prod
}
```

### alltrue() และ anytrue()

```hcl
locals {
  checks = [true, true, true, false, true]
  
  # alltrue() - true ถ้าทุก element เป็น true
  all_pass = alltrue(local.checks)  # false (มี false อยู่)
  
  # anytrue() - true ถ้ามีอย่างน้อยหนึ่ง element เป็น true
  any_pass = anytrue(local.checks)  # true
  
  # ตัวอย่างจริง: ตรวจสอบ subnets ทั้งหมด
  subnets = [
    { cidr = "10.0.1.0/24", valid = true },
    { cidr = "10.0.2.0/24", valid = true },
    { cidr = "invalid-cidr", valid = false },
  ]
  
  all_valid = alltrue([for s in local.subnets : s.valid])  # false
  any_invalid = anytrue([for s in local.subnets : !s.valid])  # true
}

# Validation ที่ใช้ alltrue()
variable "tags_list" {
  type = list(object({
    key   = string
    value = string
  }))
  
  validation {
    condition = alltrue([
      for tag in var.tags_list :
      length(tag.key) > 0 && length(tag.key) <= 128
    ])
    error_message = "All tag keys must be 1-128 characters."
  }
}
```

---

## Step 44: Operator Precedence

### ลำดับการประเมิน Operators

```
Operator Precedence (สูง → ต่ำ):
┌────────────────────────────────────────────────┐
│ 1. ! (logical NOT), - (unary minus)            │
│ 2. * / % (multiplication, division, modulo)    │
│ 3. + - (addition, subtraction)                 │
│ 4. > >= < <= (comparison)                      │
│ 5. == != (equality)                            │
│ 6. && (logical AND)                            │
│ 7. || (logical OR)                             │
│ 8. ? : (ternary/conditional)                   │
└────────────────────────────────────────────────┘
```

### ตัวอย่าง Precedence

```hcl
locals {
  a = 5
  b = 3
  c = 2
  
  # Multiplication before addition
  result1 = local.a + local.b * local.c  # 5 + (3 * 2) = 11
  
  # Parentheses override precedence
  result2 = (local.a + local.b) * local.c  # (5 + 3) * 2 = 16
  
  # Logical operators
  x = true
  y = false
  z = true
  
  # AND before OR
  logic1 = local.x || local.y && local.z  # x || (y && z) = true || false = true
  
  # NOT has highest precedence
  logic2 = !local.x || local.y  # (!x) || y = false || false = false
  
  # Comparison before logical
  n = 10
  comparison = local.n > 5 && local.n < 20  # (n > 5) && (n < 20) = true
  
  # Complex expression with precedence
  # ✅ ดีกว่า: ใช้ parentheses เพื่อความชัดเจน
  clear = (local.a > 3) && (local.b < 10) || (local.c == 2)
  
  # ⚠️ ไม่ชัดเจน: ต้องรู้ precedence
  unclear = local.a > 3 && local.b < 10 || local.c == 2
}
```

### Best Practices

```hcl
# ✅ ใช้ parentheses เพื่อความชัดเจน
locals {
  is_valid = (var.count >= 1) && (var.count <= 100)
  is_ready = (var.environment == "prod") || (var.force_deploy == true)
  
  # Conditional ที่ซับซ้อน
  instance_type = (
    var.environment == "prod" && var.high_performance
    ? "c5.4xlarge"
    : var.environment == "prod"
    ? "t3.large"
    : "t3.micro"
  )
}
```

---

## Step 45: String Operators

### String-specific Operations

```hcl
locals {
  # String comparison
  name = "terraform"
  
  is_terraform = local.name == "terraform"    # true
  not_empty    = local.name != ""             # true
  
  # String functions as operators
  has_prefix   = startswith(local.name, "terra")  # true
  has_suffix   = endswith(local.name, "form")     # true
  has_substr   = strcontains(local.name, "rraf")  # false
  
  # String length comparison
  is_long      = length(local.name) > 5  # true
  is_short     = length(local.name) < 5  # false
  
  # Regex matching
  is_lowercase = can(regex("^[a-z]+$", local.name))  # true
  is_alnum     = can(regex("^[a-zA-Z0-9]+$", local.name))  # true
}
```

### String Interpolation vs Concatenation

```hcl
locals {
  project = "myapp"
  env     = "production"
  region  = "ap-southeast-1"
  
  # ✅ Interpolation (แนะนำ)
  bucket_v1 = "${local.project}-${local.env}-${local.region}"
  
  # ✅ join() สำหรับ list
  bucket_v2 = join("-", [local.project, local.env, local.region])
  
  # ✅ format() สำหรับ complex formatting
  bucket_v3 = format("%s-%s-%s", local.project, local.env, local.region)
  
  # ทั้งหมดได้ผลเหมือนกัน: "myapp-production-ap-southeast-1"
}
```

---

## Step 46: Index Expressions

### การ Access Elements ด้วย Index

```hcl
locals {
  # List index (0-based)
  fruits = ["apple", "banana", "cherry"]
  
  first_fruit  = local.fruits[0]   # "apple"
  second_fruit = local.fruits[1]   # "banana"
  third_fruit  = local.fruits[2]   # "cherry"
  
  # Negative index ไม่ได้รับการ support ใน HCL
  # last_fruit = local.fruits[-1]  # Error!
  # ใช้ length - 1 แทน
  last_fruit   = local.fruits[length(local.fruits) - 1]  # "cherry"
  
  # Map index ด้วย key
  config = {
    host = "localhost"
    port = 5432
  }
  
  host_v1 = local.config["host"]   # "localhost" (bracket notation)
  host_v2 = local.config.host      # "localhost" (dot notation)
  
  port_v1 = local.config["port"]   # 5432
  port_v2 = local.config.port      # 5432
}
```

### Safe Index Access

```hcl
locals {
  list = ["a", "b", "c"]
  
  # ✅ ปลอดภัย - ตรวจสอบก่อน
  safe_first = length(local.list) > 0 ? local.list[0] : null
  
  # ✅ ใช้ try() สำหรับ safe access
  try_fourth = try(local.list[3], "default")  # "default" (index 3 ไม่มี)
  
  # ✅ element() function - circular index
  element_0 = element(local.list, 0)  # "a"
  element_4 = element(local.list, 4)  # "b" (4 % 3 = 1)
  
  # ✅ lookup() สำหรับ maps
  map = { key = "value", other = "data" }
  safe_lookup = lookup(local.map, "missing_key", "default")  # "default"
}
```

### Index ใน Resources

```hcl
variable "availability_zones" {
  type    = list(string)
  default = ["ap-southeast-1a", "ap-southeast-1b", "ap-southeast-1c"]
}

resource "aws_subnet" "public" {
  count = length(var.availability_zones)
  
  vpc_id            = aws_vpc.main.id
  cidr_block        = "10.0.${count.index}.0/24"
  availability_zone = var.availability_zones[count.index]
  
  tags = {
    Name = "public-subnet-${var.availability_zones[count.index]}"
  }
}

# สร้าง ECS tasks ที่ต้องการ AZ เฉพาะ
locals {
  az_count = length(var.availability_zones)
  
  # Round-robin assignment
  task_az_assignments = [
    for i in range(6) : var.availability_zones[i % local.az_count]
  ]
  # ["ap-southeast-1a", "ap-southeast-1b", "ap-southeast-1c",
  #  "ap-southeast-1a", "ap-southeast-1b", "ap-southeast-1c"]
}
```

---

## Step 47: Attribute Access

### การ Access Attributes ด้วย Dot Notation

```hcl
# Resource attribute access
resource "aws_vpc" "main" {
  cidr_block = "10.0.0.0/16"
}

resource "aws_subnet" "public" {
  vpc_id = aws_vpc.main.id  # dot notation
  # เหมือนกับ: vpc_id = aws_vpc.main["id"]
}

# Nested attribute access
locals {
  config = {
    database = {
      host = "db.example.com"
      port = 5432
      credentials = {
        username = "admin"
        password = "secret"
      }
    }
  }
  
  # Chain ของ dot notation
  db_host     = local.config.database.host
  db_port     = local.config.database.port
  db_username = local.config.database.credentials.username
}
```

### Attribute Access vs Index Access

```hcl
locals {
  obj = {
    name     = "test"
    "special-key" = "value"  # key ที่มี dash
    "123start"    = "data"   # key เริ่มด้วยตัวเลข
  }
  
  # ✅ Dot notation (ถ้า key เป็น valid identifier)
  name_v1 = local.obj.name
  
  # ✅ Bracket notation (ใช้ได้เสมอ)
  name_v2 = local.obj["name"]
  
  # ✅ ต้องใช้ bracket ถ้า key ไม่ใช่ valid identifier
  special = local.obj["special-key"]
  start   = local.obj["123start"]
}
```

### Dynamic Attribute Access

```hcl
variable "attribute_name" {
  default = "host"
}

locals {
  config = {
    host = "localhost"
    port = 5432
  }
  
  # Dynamic attribute access ด้วย lookup()
  dynamic_value = lookup(local.config, var.attribute_name, "unknown")
  
  # หรือใช้ bracket notation กับ variable
  # dynamic_attr = local.config[var.attribute_name]
  # ⚠️ นี่จะ error ถ้า key ไม่มีใน map
}
```

---

## Step 48: Splat Expressions

### Legacy Splat (.*) 

```hcl
# Legacy splat - ใช้กับ list ของ objects
resource "aws_instance" "web" {
  count = 3
  
  ami           = "ami-12345"
  instance_type = "t3.micro"
}

# ดึง attribute จากทุก instances ด้วย legacy splat
output "instance_ids" {
  value = aws_instance.web[*].id  
  # ["i-001", "i-002", "i-003"]
}

output "instance_public_ips" {
  value = aws_instance.web[*].public_ip
  # ["1.1.1.1", "2.2.2.2", "3.3.3.3"]
}
```

### Full Splat ([*])

```hcl
# Full splat - ทำงานกับ lists และ tuples
locals {
  servers = [
    { name = "web-01", ip = "10.0.1.1", port = 80 },
    { name = "web-02", ip = "10.0.1.2", port = 80 },
    { name = "api-01", ip = "10.0.2.1", port = 8080 },
  ]
  
  # ดึง names ด้วย full splat
  server_names = local.servers[*].name  # ["web-01", "web-02", "api-01"]
  server_ips   = local.servers[*].ip    # ["10.0.1.1", "10.0.1.2", "10.0.2.1"]
  server_ports = local.servers[*].port  # [80, 80, 8080]
}
```

### Legacy vs Full Splat ความแตกต่าง

```hcl
# Legacy splat (.*) - เก่ากว่า
# - ใช้ได้เฉพาะกับ list ของ objects
# - ทำงานกับ null โดย return null

# Full splat ([*]) - ใหม่กว่า  
# - ใช้ได้กับ list, sets, tuples
# - สามารถ chain ได้
# - ทำงานกับ null โดย return empty list

locals {
  maybe_null = null  # อาจเป็น list หรือ null
  
  # Legacy splat กับ null: อาจ return null
  # legacy = maybe_null.*.id  # ⚠️ behavior ไม่แน่นอน
  
  # Full splat กับ null: return empty list []
  # full = maybe_null[*].id  # ✅ return []
  
  # ตัวอย่างจริง
  optional_instances = var.create_servers ? aws_instance.web : null
  instance_ids = optional_instances[*].id  # [] หรือ ["i-001", ...]
}
```

### Splat ใน Real-world

```hcl
# ALB Target Group Registration
resource "aws_lb_target_group_attachment" "web" {
  count = length(aws_instance.web)
  
  target_group_arn = aws_lb_target_group.web.arn
  target_id        = aws_instance.web[count.index].id
  port             = 80
}

# Route53 Records สำหรับทุก instances
resource "aws_route53_record" "web" {
  count = length(aws_instance.web)
  
  zone_id = aws_route53_zone.main.id
  name    = "web-${count.index + 1}.example.com"
  type    = "A"
  ttl     = 300
  records = [aws_instance.web[count.index].public_ip]
}

# Outputs ที่ใช้ splat
output "all_instance_ids" {
  value = aws_instance.web[*].id
}

output "all_public_ips" {
  value = aws_instance.web[*].public_ip
}

output "all_private_dns" {
  value = aws_instance.web[*].private_dns
}
```

---

## Step 49: Expression Evaluation

### การประเมิน Expressions ใน HCL

```hcl
# Expressions ประเมินจาก left ไป right
# และ short-circuit evaluation

locals {
  # Short-circuit AND: ถ้า left เป็น false ไม่ต้องประเมิน right
  a = false
  result_and = local.a && expensive_function()  # expensive_function() ไม่ถูกเรียก
  
  # Short-circuit OR: ถ้า left เป็น true ไม่ต้องประเมิน right
  b = true
  result_or = local.b || expensive_function()   # expensive_function() ไม่ถูกเรียก
  
  # ในทางปฏิบัติ: ใช้สำหรับ null checks
  config = null
  # safe access: ถ้า config เป็น null จะ short-circuit
  db_host = local.config != null ? local.config.host : "localhost"
}
```

### Lazy Evaluation

```hcl
# HCL ประเมิน expressions เมื่อจำเป็น
# ไม่ใช่ตอน declare

locals {
  # นี่จะไม่ error ถ้า enable_feature = false
  # เพราะ HCL จะไม่ประเมิน expression ถ้าไม่ต้องการ
  
  # ✅ Safe: ternary ช่วยหลีกเลี่ยง error
  resource_name = var.enable_feature ? "my-resource" : null
}
```

### Expression Chaining

```hcl
locals {
  raw_data = [
    { name = "Alice", score = 85, active = true },
    { name = "Bob",   score = 72, active = false },
    { name = "Carol", score = 91, active = true },
    { name = "Dave",  score = 68, active = true },
  ]
  
  # Chain expressions
  # 1. Filter active users
  # 2. ดึง scores
  # 3. คำนวณ average
  
  active_scores = [
    for u in local.raw_data : u.score
    if u.active
  ]
  
  total_active    = length(local.active_scores)
  sum_of_scores   = sum(local.active_scores)
  average_score   = local.total_active > 0 ? (
    local.sum_of_scores / local.total_active
  ) : 0
  
  # 85 + 91 + 68 = 244, 244/3 = 81.33...
}
```

---

## Step 50: Real-world Expressions สมบูรณ์

### Complete Example: Infrastructure Sizing

```hcl
# variables.tf
variable "environment" {
  type = string
}

variable "workload_type" {
  description = "Type of workload: web, api, batch, cache"
  type        = string
  default     = "web"
  
  validation {
    condition     = contains(["web", "api", "batch", "cache"], var.workload_type)
    error_message = "Workload type must be web, api, batch, or cache."
  }
}

variable "expected_rps" {
  description = "Expected requests per second"
  type        = number
  default     = 100
}

variable "high_availability" {
  description = "Enable high availability (minimum 2 instances)"
  type        = bool
  default     = false
}

# locals.tf
locals {
  is_production = var.environment == "prod"
  is_high_load  = var.expected_rps > 1000
  needs_ha      = var.high_availability || local.is_production
  
  # Instance type calculation
  base_instance_type = (
    var.workload_type == "batch"  ? "c5.xlarge"   :
    var.workload_type == "cache"  ? "r5.large"    :
    local.is_high_load            ? "t3.large"    :
    "t3.micro"
  )
  
  # Production gets upgraded instance types
  instance_type = local.is_production ? (
    local.base_instance_type == "t3.micro"  ? "t3.small"   :
    local.base_instance_type == "t3.small"  ? "t3.medium"  :
    local.base_instance_type == "t3.large"  ? "t3.xlarge"  :
    local.base_instance_type
  ) : local.base_instance_type
  
  # Instance count
  min_count = local.needs_ha ? 2 : 1
  
  max_count = (
    local.is_high_load && local.is_production ? 20 :
    local.is_high_load                        ? 10 :
    local.is_production                       ? 5  :
    3
  )
  
  desired_count = (
    local.is_production ? max(local.min_count, ceil(local.max_count * 0.4)) :
    local.min_count
  )
  
  # CPU and memory
  cpu    = contains(["c5.xlarge", "t3.xlarge", "c5.4xlarge"], local.instance_type) ? 4096 : 2048
  memory = local.cpu * 2  # 2x CPU in MB
  
  # Scaling policy
  scale_out_threshold = local.is_production ? 60 : 80
  scale_in_threshold  = local.is_production ? 30 : 40
  
  # Cost estimation (rough)
  hourly_rates = {
    "t3.micro"   = 0.0104
    "t3.small"   = 0.0208
    "t3.medium"  = 0.0416
    "t3.large"   = 0.0832
    "t3.xlarge"  = 0.1664
    "c5.xlarge"  = 0.1700
    "r5.large"   = 0.1260
    "c5.4xlarge" = 0.6800
  }
  
  hourly_cost   = lookup(local.hourly_rates, local.instance_type, 0.1) * local.desired_count
  monthly_cost  = local.hourly_cost * 24 * 30
  
  # Summary tags
  sizing_tags = {
    InstanceType  = local.instance_type
    DesiredCount  = tostring(local.desired_count)
    MaxCount      = tostring(local.max_count)
    WorkloadType  = var.workload_type
    HighAvail     = tostring(local.needs_ha)
    MonthlyCost   = format("$%.2f", local.monthly_cost)
  }
}

# outputs.tf
output "infrastructure_plan" {
  value = {
    instance_type  = local.instance_type
    min_count      = local.min_count
    desired_count  = local.desired_count
    max_count      = local.max_count
    cpu            = local.cpu
    memory_mb      = local.memory
    ha_enabled     = local.needs_ha
    estimated_cost = format("$%.2f/month", local.monthly_cost)
    
    scaling = {
      scale_out_at = "${local.scale_out_threshold}% CPU"
      scale_in_at  = "${local.scale_in_threshold}% CPU"
    }
  }
}
```

### Expression Patterns ที่ใช้บ่อย

```hcl
locals {
  # Pattern 1: Null coalescing
  effective_timeout = coalesce(var.custom_timeout, var.default_timeout, 30)
  
  # Pattern 2: Environment-based config
  env_settings = {
    dev     = { replicas = 1, debug = true,  cache_ttl = 60   }
    staging = { replicas = 2, debug = false, cache_ttl = 300  }
    prod    = { replicas = 5, debug = false, cache_ttl = 3600 }
  }
  settings = local.env_settings[var.environment]
  
  # Pattern 3: Feature flag
  features = {
    monitoring = var.environment == "prod"
    tracing    = var.environment != "dev"
    profiling  = var.enable_profiling && var.environment == "prod"
  }
  
  # Pattern 4: Conditional resource naming
  resource_prefix = "${var.project}-${var.environment}"
  unique_suffix   = var.use_random_suffix ? "-${random_id.suffix.hex}" : ""
  resource_name   = "${local.resource_prefix}${local.unique_suffix}"
  
  # Pattern 5: Default value chain
  final_region = coalesce(
    var.override_region,    # 1st priority: explicit override
    var.preferred_region,   # 2nd priority: preferred region
    "ap-southeast-1"        # fallback: default region
  )
}
```

---

## สรุป Expressions และ Operators

### Quick Reference Table

| Operator | Syntax | Example | Result |
|----------|--------|---------|--------|
| Add | `+` | `5 + 3` | `8` |
| Subtract | `-` | `5 - 3` | `2` |
| Multiply | `*` | `5 * 3` | `15` |
| Divide | `/` | `10 / 3` | `3.333...` |
| Modulo | `%` | `10 % 3` | `1` |
| Equal | `==` | `"a" == "a"` | `true` |
| Not Equal | `!=` | `"a" != "b"` | `true` |
| Less Than | `<` | `3 < 5` | `true` |
| Greater Than | `>` | `5 > 3` | `true` |
| And | `&&` | `true && false` | `false` |
| Or | `\|\|` | `true \|\| false` | `true` |
| Not | `!` | `!true` | `false` |
| Ternary | `? :` | `true ? "a" : "b"` | `"a"` |
| Index | `[n]` | `list[0]` | first element |
| Attribute | `.` | `obj.key` | attribute value |
| Splat | `[*]` | `list[*].id` | list of ids |

💡 **Pro Tips:**
- ใช้ parentheses เพื่อความชัดเจน แม้ว่า operator precedence จะถูกต้อง
- ใช้ `try()` เพื่อป้องกัน runtime errors จาก null access
- ใช้ `coalesce()` สำหรับ null coalescing pattern
- ใช้ `alltrue()`/`anytrue()` สำหรับ validation lists

⚠️ **Common Mistakes:**
- สับสนระหว่าง `=` (assignment) กับ `==` (comparison)
- ลืมว่า HCL ทำ float division (10/3 = 3.333 ไม่ใช่ 3)
- ใช้ index ที่เกิน length โดยไม่มี bounds check
- ไม่ระวัง null ก่อน attribute access

---

*ก่อนหน้า: [Part 004 - HCL Data Types: Complex Types](part-004.md)*
*ต่อไป: [Part 006 - HCL Built-in Functions](part-006.md)*
