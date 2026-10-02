# Part 007: HCL Conditional Expressions
## นิพจน์เงื่อนไขใน HCL (Steps 61-70)

---

## Step 61: Ternary Operator พื้นฐาน

### syntax: condition ? true_val : false_val

```hcl
# โครงสร้างพื้นฐาน
# condition ? value_if_true : value_if_false

locals {
  environment = "production"
  
  # Basic ternary
  instance_type = local.environment == "production" ? "t3.large" : "t3.micro"
  # "t3.large"
  
  # Boolean condition
  is_prod   = true
  log_level = local.is_prod ? "ERROR" : "DEBUG"
  # "ERROR"
  
  # Number condition
  count = 5
  msg   = local.count > 3 ? "many" : "few"
  # "many"
  
  # Null condition
  override = null
  effective = local.override != null ? local.override : "default-value"
  # "default-value"
}
```

### Ternary ใน Resource Arguments

```hcl
variable "environment" {
  type    = string
  default = "development"
}

variable "enable_deletion_protection" {
  type    = bool
  default = null  # null = use default based on environment
}

locals {
  is_prod = var.environment == "production"
  
  # ใช้ effective protection: use override if set, else env-based default
  deletion_protected = (
    var.enable_deletion_protection != null
    ? var.enable_deletion_protection
    : local.is_prod
  )
}

resource "aws_db_instance" "main" {
  engine         = "postgres"
  instance_class = local.is_prod ? "db.r5.large" : "db.t3.micro"
  
  # Production gets multi-AZ
  multi_az = local.is_prod ? true : false
  
  # Production has deletion protection
  deletion_protection = local.deletion_protected
  
  # Backup retention: 30 days prod, 7 days dev
  backup_retention_period = local.is_prod ? 30 : 7
  
  # Storage size
  allocated_storage = local.is_prod ? 100 : 20
  
  # Performance insights
  performance_insights_enabled = local.is_prod ? true : false
  
  tags = {
    Environment = var.environment
    Critical    = local.is_prod ? "true" : "false"
  }
}
```

---

## Step 62: Nested Conditionals

### Chained Ternary (if-elseif-else)

```hcl
locals {
  environment = "staging"
  
  # Nested ternary (if-elseif-else pattern)
  instance_type = (
    local.environment == "production"
    ? "t3.xlarge"
    : local.environment == "staging"
    ? "t3.small"
    : "t3.micro"
  )
  # "t3.small"
  
  # Multi-level nested
  log_retention = (
    local.environment == "production"  ? 365 :
    local.environment == "staging"     ? 90  :
    local.environment == "development" ? 7   :
    1  # default/unknown
  )
  # 90
}
```

### ตัวอย่าง Nested Conditionals จริง

```hcl
variable "instance_size" {
  description = "Size category: small, medium, large, xlarge"
  type        = string
  default     = "medium"
}

variable "workload_type" {
  description = "Type: compute, memory, balanced"
  type        = string
  default     = "balanced"
}

locals {
  # Multi-dimension instance type selection
  instance_type = (
    var.workload_type == "compute" ? (
      var.instance_size == "small"  ? "c5.large"   :
      var.instance_size == "medium" ? "c5.xlarge"  :
      var.instance_size == "large"  ? "c5.2xlarge" :
      "c5.4xlarge"
    ) :
    var.workload_type == "memory" ? (
      var.instance_size == "small"  ? "r5.large"   :
      var.instance_size == "medium" ? "r5.xlarge"  :
      var.instance_size == "large"  ? "r5.2xlarge" :
      "r5.4xlarge"
    ) : (
      # default: balanced
      var.instance_size == "small"  ? "t3.medium"  :
      var.instance_size == "medium" ? "t3.large"   :
      var.instance_size == "large"  ? "t3.xlarge"  :
      "t3.2xlarge"
    )
  )
  
  # Nested condition สำหรับ storage class
  storage_class = (
    var.environment == "production" ? (
      var.data_type == "archive" ? "GLACIER"     :
      var.data_type == "infreq"  ? "STANDARD_IA" :
      "STANDARD"
    ) : (
      "STANDARD"  # Non-prod always uses standard
    )
  )
}
```

---

## Step 63: Conditionals กับ Complex Types

### Conditional กับ List

```hcl
variable "enable_extra_monitoring" {
  type    = bool
  default = false
}

variable "environment" {
  type    = string
  default = "development"
}

locals {
  is_prod = var.environment == "production"
  
  # Conditional list
  base_monitoring_actions = [
    "arn:aws:sns:ap-southeast-1:123:alarm-topic"
  ]
  
  extra_monitoring_actions = [
    "arn:aws:sns:ap-southeast-1:123:critical-topic",
    "arn:aws:sns:ap-southeast-1:123:pagerduty-topic"
  ]
  
  # Combine lists based on condition
  alarm_actions = concat(
    local.base_monitoring_actions,
    local.is_prod || var.enable_extra_monitoring ? local.extra_monitoring_actions : []
  )
  
  # Conditional tags list
  base_tags = {
    Environment = var.environment
    ManagedBy   = "terraform"
  }
  
  prod_tags = local.is_prod ? {
    CriticalLevel = "high"
    SLA           = "99.9%"
    BackupPolicy  = "daily"
  } : {}
  
  # Merge: prod_tags จะเป็น empty object ถ้าไม่ใช่ prod
  all_tags = merge(local.base_tags, local.prod_tags)
}
```

### Conditional กับ Object

```hcl
# Conditional object content
variable "database_config" {
  type    = map(string)
  default = {}
}

locals {
  has_custom_db = length(var.database_config) > 0
  
  # ใช้ conditional เพื่อเลือก database config
  effective_db_config = local.has_custom_db ? var.database_config : {
    host     = "localhost"
    port     = "5432"
    database = "app"
  }
  
  # Conditional nested object
  monitoring_config = local.is_prod ? {
    enabled          = true
    detailed_metrics = true
    alarm_threshold  = 80
    alarm_actions    = ["arn:aws:sns:ap-southeast-1:123:prod-alerts"]
  } : {
    enabled          = false
    detailed_metrics = false
    alarm_threshold  = 90
    alarm_actions    = []
  }
}
```

---

## Step 64: Conditionals ใน Resource Arguments

### count = 0 or 1 Pattern

```hcl
variable "create_nat_gateway" {
  description = "สร้าง NAT Gateway สำหรับ private subnets"
  type        = bool
  default     = false
}

variable "create_vpn_gateway" {
  description = "สร้าง VPN Gateway"
  type        = bool
  default     = false
}

variable "enable_flow_logs" {
  description = "เปิด VPC flow logs"
  type        = bool
  default     = true
}

# สร้าง resource เฉพาะเมื่อ condition เป็น true
resource "aws_eip" "nat" {
  count  = var.create_nat_gateway ? 1 : 0
  domain = "vpc"
}

resource "aws_nat_gateway" "main" {
  count = var.create_nat_gateway ? 1 : 0
  
  allocation_id = aws_eip.nat[0].id
  subnet_id     = aws_subnet.public[0].id
  
  depends_on = [aws_internet_gateway.main]
}

resource "aws_vpn_gateway" "main" {
  count  = var.create_vpn_gateway ? 1 : 0
  vpc_id = aws_vpc.main.id
}

resource "aws_flow_log" "vpc" {
  count = var.enable_flow_logs ? 1 : 0
  
  iam_role_arn    = aws_iam_role.flow_log.arn
  log_destination = aws_cloudwatch_log_group.flow_log.arn
  traffic_type    = "ALL"
  vpc_id          = aws_vpc.main.id
}
```

### Conditional กับ depends_on

```hcl
resource "aws_route" "private_nat" {
  count = var.create_nat_gateway ? length(aws_route_table.private) : 0
  
  route_table_id         = aws_route_table.private[count.index].id
  destination_cidr_block = "0.0.0.0/0"
  nat_gateway_id         = aws_nat_gateway.main[0].id  # ต้องมี NAT GW
}
```

---

## Step 65: Conditionals ใน Variable Defaults

### Default Values ที่ขึ้นกับ Conditions

```hcl
variable "environment" {
  type    = string
  default = "development"
}

# ✅ วิธีที่ 1: ใช้ locals สำหรับ computed defaults
locals {
  is_prod = var.environment == "production"
  
  # Computed defaults ขึ้นกับ environment
  default_instance_type    = local.is_prod ? "t3.large"  : "t3.micro"
  default_min_capacity     = local.is_prod ? 3            : 1
  default_max_capacity     = local.is_prod ? 10           : 3
  default_retention_days   = local.is_prod ? 90           : 7
  default_backup_enabled   = local.is_prod ? true         : false
  default_multi_az         = local.is_prod ? true         : false
  default_deletion_protect = local.is_prod ? true         : false
}

variable "instance_type" {
  type    = string
  default = null  # null = use computed default
}

# Final effective values
locals {
  effective_instance_type = coalesce(var.instance_type, local.default_instance_type)
}
```

### Validation กับ Conditional Defaults

```hcl
variable "ssl_certificate_arn" {
  description = "ARN ของ SSL Certificate (required in production)"
  type        = string
  default     = null
  
  validation {
    # ถ้าไม่ใช่ prod สามารถเป็น null ได้
    # ถ้าเป็น prod ต้องมีค่า (validation นี้ต้องทำใน locals)
    condition     = var.ssl_certificate_arn == null || can(regex("^arn:aws:acm:", var.ssl_certificate_arn))
    error_message = "ssl_certificate_arn must be a valid ACM ARN."
  }
}

# ใช้ local + validation สำหรับ cross-variable validation
locals {
  # Validate production requirements
  validate_prod_ssl = (
    var.environment != "production" || var.ssl_certificate_arn != null
    ? null  # OK
    : file("ERROR: ssl_certificate_arn is required for production")
  )
}
```

---

## Step 66: count กับ Conditionals

### count = var.create ? 1 : 0

```hcl
# Pattern 1: Simple enable/disable
variable "create_s3_bucket" {
  type    = bool
  default = true
}

resource "aws_s3_bucket" "optional" {
  count  = var.create_s3_bucket ? 1 : 0
  bucket = "${var.project}-${var.environment}-optional"
}

# เข้าถึง optional resource
output "optional_bucket_id" {
  value = var.create_s3_bucket ? aws_s3_bucket.optional[0].id : null
}

# Pattern 2: Conditional based on environment
variable "environment" {
  type    = string
  default = "development"
}

# WAF ใช้เฉพาะ production
resource "aws_wafv2_web_acl" "main" {
  count = var.environment == "production" ? 1 : 0
  
  name  = "${var.project}-waf"
  scope = "REGIONAL"
  
  default_action {
    allow {}
  }
  
  visibility_config {
    cloudwatch_metrics_enabled = true
    metric_name                = "${var.project}-waf"
    sampled_requests_enabled   = true
  }
}

# Bastion host เฉพาะ non-production
resource "aws_instance" "bastion" {
  count = var.environment != "production" ? 1 : 0
  
  ami           = var.bastion_ami
  instance_type = "t3.micro"
  subnet_id     = aws_subnet.public[0].id
  
  tags = {
    Name = "bastion-${var.environment}"
    Role = "bastion"
  }
}

# Pattern 3: count จาก variable
variable "nat_gateway_count" {
  description = "จำนวน NAT Gateways (0=ไม่สร้าง, 1=single, n=per-AZ)"
  type        = number
  default     = 1
  
  validation {
    condition     = var.nat_gateway_count >= 0
    error_message = "NAT gateway count must be 0 or greater."
  }
}

resource "aws_eip" "nat" {
  count  = var.nat_gateway_count
  domain = "vpc"
}

resource "aws_nat_gateway" "main" {
  count = var.nat_gateway_count
  
  allocation_id = aws_eip.nat[count.index].id
  subnet_id     = aws_subnet.public[count.index % length(aws_subnet.public)].id
}
```

---

## Step 67: for_each กับ Conditionals

### Filter ด้วย Conditional ก่อน for_each

```hcl
variable "microservices" {
  type = map(object({
    enabled  = bool
    port     = number
    replicas = number
  }))
  
  default = {
    api = {
      enabled  = true
      port     = 8080
      replicas = 3
    }
    worker = {
      enabled  = true
      port     = 9090
      replicas = 2
    }
    experimental = {
      enabled  = false  # ปิดการใช้งาน
      port     = 9999
      replicas = 1
    }
  }
}

# Filter ก่อน for_each - สร้างเฉพาะ enabled services
locals {
  enabled_services = {
    for name, config in var.microservices :
    name => config
    if config.enabled
  }
}

resource "aws_ecs_service" "services" {
  for_each = local.enabled_services
  
  name            = "${var.project}-${each.key}"
  cluster         = aws_ecs_cluster.main.id
  task_definition = aws_ecs_task_definition.services[each.key].arn
  desired_count   = each.value.replicas
}

# Filter สำหรับ security group rules
variable "security_rules" {
  type = map(object({
    port        = number
    protocol    = string
    cidr        = string
    enabled     = bool
    description = string
  }))
  
  default = {
    https = {
      port        = 443
      protocol    = "tcp"
      cidr        = "0.0.0.0/0"
      enabled     = true
      description = "HTTPS from internet"
    }
    http = {
      port        = 80
      protocol    = "tcp"
      cidr        = "0.0.0.0/0"
      enabled     = true
      description = "HTTP redirect"
    }
    admin = {
      port        = 8443
      protocol    = "tcp"
      cidr        = "10.0.0.0/8"
      enabled     = false  # disabled
      description = "Admin panel (disabled)"
    }
  }
}

resource "aws_security_group" "web" {
  name   = "web-sg"
  vpc_id = aws_vpc.main.id
  
  # สร้างเฉพาะ enabled rules
  dynamic "ingress" {
    for_each = {
      for name, rule in var.security_rules :
      name => rule
      if rule.enabled
    }
    
    content {
      description = ingress.value.description
      from_port   = ingress.value.port
      to_port     = ingress.value.port
      protocol    = ingress.value.protocol
      cidr_blocks = [ingress.value.cidr]
    }
  }
}
```

---

## Step 68: Conditional Module Inclusion

### Module สร้างตาม Condition

```hcl
variable "deploy_monitoring" {
  description = "Deploy monitoring stack"
  type        = bool
  default     = false
}

variable "deploy_waf" {
  description = "Deploy WAF"
  type        = bool
  default     = false
}

# ✅ Pattern 1: สร้าง module พร้อม count
# (ใช้ได้กับ modules ที่ design มาสำหรับ count)
module "monitoring" {
  count  = var.deploy_monitoring ? 1 : 0
  source = "./modules/monitoring"
  
  project     = var.project
  environment = var.environment
  vpc_id      = aws_vpc.main.id
}

# ✅ Pattern 2: Module ที่มี internal conditionals
module "vpc" {
  source = "./modules/vpc"
  
  project     = var.project
  environment = var.environment
  
  # ส่ง conditional config ไปยัง module
  create_nat_gateway = var.environment == "production"
  create_vpn_gateway = var.environment == "production" && var.needs_vpn
  enable_flow_logs   = var.enable_monitoring || var.environment == "production"
}

# เข้าถึง output จาก conditional module
locals {
  monitoring_endpoint = (
    var.deploy_monitoring
    ? module.monitoring[0].grafana_url
    : null
  )
}
```

### Module สำหรับ Multiple Environments

```hcl
# สร้าง module instances ตาม environments
variable "deploy_environments" {
  type    = set(string)
  default = ["dev", "staging"]
}

module "environment" {
  for_each = var.deploy_environments
  source   = "./modules/environment"
  
  environment = each.key
  project     = var.project
  
  # Config ต่างกันตาม environment
  instance_type    = each.key == "staging" ? "t3.small" : "t3.micro"
  instance_count   = each.key == "staging" ? 2 : 1
  enable_https     = each.key == "staging" ? true : false
}

output "environment_endpoints" {
  value = {
    for env, mod in module.environment :
    env => mod.endpoint
  }
}
```

---

## Step 69: Null Coalescing Patterns

### coalesce() Function

```hcl
# coalesce() - return ค่าแรกที่ไม่ใช่ null และไม่ใช่ empty string
locals {
  a = null
  b = ""
  c = "actual-value"
  d = "another-value"
  
  # coalesce ข้ามค่า null และ empty string
  result = coalesce(local.a, local.b, local.c, local.d)
  # "actual-value"
  
  # Practical: default value chain
  override_region  = null  # ไม่ได้ set
  preferred_region = "ap-southeast-1"
  
  effective_region = coalesce(
    var.override_region,      # 1st: explicit override
    var.preferred_region,     # 2nd: preferred
    "us-east-1"               # fallback
  )
}
```

### coalescelist() Function

```hcl
# coalescelist() - return list แรกที่ไม่ว่าง
locals {
  override_ports  = []              # empty
  default_ports   = [80, 443, 8080]
  fallback_ports  = [80]
  
  effective_ports = coalescelist(
    local.override_ports,   # skip empty
    local.default_ports,    # use this
    local.fallback_ports
  )
  # [80, 443, 8080]
}
```

### Null Coalescing Patterns ใน Real-world

```hcl
# Pattern 1: Optional variable with computed default
variable "instance_type" {
  type    = string
  default = null  # null = compute based on environment
}

variable "environment" {
  type    = string
  default = "development"
}

locals {
  computed_instance_type = var.environment == "production" ? "t3.large" : "t3.micro"
  
  effective_instance_type = coalesce(var.instance_type, local.computed_instance_type)
}

# Pattern 2: Override chain สำหรับ tags
variable "global_tags" {
  type    = map(string)
  default = {}
}

variable "project_tags" {
  type    = map(string)
  default = {}
}

variable "resource_tags" {
  type    = map(string)
  default = {}
}

locals {
  # Merge order: global → project → resource
  # resource_tags override project_tags override global_tags
  final_tags = merge(
    {
      ManagedBy   = "terraform"
      Environment = var.environment
    },
    local.global_tags,
    local.project_tags,
    local.resource_tags
  )
}

# Pattern 3: Database connection string
variable "db_host" {
  type    = string
  default = null
}

variable "db_replica_host" {
  type    = string
  default = null
}

locals {
  # Use replica if available, else primary
  effective_db_host = coalesce(var.db_replica_host, var.db_host, "localhost")
  
  # Connection string
  db_connection = format(
    "postgres://%s:%d/%s",
    local.effective_db_host,
    coalesce(var.db_port, 5432),
    coalesce(var.db_name, "app")
  )
}
```

---

## Step 70: Complex Conditional Logic Patterns

### Multi-condition Resource Configuration

```hcl
# variables.tf
variable "environment" {
  type    = string
  default = "development"
}

variable "region" {
  type    = string
  default = "ap-southeast-1"
}

variable "compliance_requirements" {
  description = "Compliance frameworks: pci, hipaa, soc2"
  type        = set(string)
  default     = []
}

variable "performance_tier" {
  description = "Performance tier: standard, high, ultra"
  type        = string
  default     = "standard"
}

# locals.tf
locals {
  is_prod         = var.environment == "production"
  is_staging      = var.environment == "staging"
  is_dev          = var.environment == "development"
  
  has_pci         = contains(var.compliance_requirements, "pci")
  has_hipaa       = contains(var.compliance_requirements, "hipaa")
  has_soc2        = contains(var.compliance_requirements, "soc2")
  has_compliance  = length(var.compliance_requirements) > 0
  
  is_high_perf    = contains(["high", "ultra"], var.performance_tier)
  is_ultra_perf   = var.performance_tier == "ultra"
  
  # Derived conditions
  needs_encryption  = local.is_prod || local.has_compliance
  needs_audit_log   = local.is_prod || local.has_pci || local.has_hipaa
  needs_multi_az    = local.is_prod && local.is_high_perf
  needs_dedicated   = local.has_pci || local.is_ultra_perf
  
  # Database configuration
  db_config = {
    instance_class = (
      local.is_ultra_perf   ? "db.r5.4xlarge" :
      local.is_high_perf    ? "db.r5.xlarge"  :
      local.is_prod         ? "db.t3.large"   :
      local.is_staging      ? "db.t3.small"   :
      "db.t3.micro"
    )
    
    storage_gb = (
      local.has_pci    ? 500 :
      local.is_prod    ? 200 :
      local.is_staging ? 50  :
      20
    )
    
    storage_type = local.is_high_perf || local.has_compliance ? "io1" : "gp3"
    storage_iops = local.is_high_perf ? 10000 : null
    
    backup_retention = (
      local.has_pci    ? 365 :
      local.has_hipaa  ? 365 :
      local.is_prod    ? 30  :
      local.is_staging ? 7   :
      1
    )
    
    multi_az            = local.needs_multi_az
    storage_encrypted   = local.needs_encryption
    deletion_protection = local.is_prod
    
    performance_insights_enabled          = local.is_prod || local.is_high_perf
    performance_insights_retention_period = local.is_prod ? 731 : 7  # 2 years or 1 week
    
    enabled_cloudwatch_logs_exports = concat(
      ["postgresql"],
      local.needs_audit_log ? ["upgrade"] : []
    )
  }
  
  # Auto-scaling configuration
  asg_config = {
    min_size         = local.is_prod ? (local.needs_multi_az ? 2 : 1) : 1
    max_size         = (
      local.is_ultra_perf ? 50 :
      local.is_high_perf  ? 20 :
      local.is_prod       ? 10 :
      5
    )
    desired_capacity = (
      local.is_prod    ? max(2, ceil(local.asg_config.max_size * 0.3)) :
      local.is_staging ? 2 :
      1
    )
    
    health_check_type = local.is_prod ? "ELB" : "EC2"
    health_check_grace_period = local.is_prod ? 300 : 60
  }
}

# outputs สรุป
output "infrastructure_decision" {
  value = {
    environment         = var.environment
    compliance_required = local.has_compliance
    compliance_types    = var.compliance_requirements
    
    decisions = {
      needs_encryption  = local.needs_encryption
      needs_audit_log   = local.needs_audit_log
      needs_multi_az    = local.needs_multi_az
      needs_dedicated   = local.needs_dedicated
    }
    
    db_instance_class   = local.db_config.instance_class
    db_storage_gb       = local.db_config.storage_gb
    db_backup_days      = local.db_config.backup_retention
    
    asg_min_size        = local.asg_config.min_size
    asg_max_size        = local.asg_config.max_size
    asg_desired         = local.asg_config.desired_capacity
  }
}
```

### Conditional Error Handling

```hcl
# สร้าง validation error ด้วย conditional
locals {
  # ✅ Pattern: ใช้ conditional สำหรับ cross-variable validation
  
  # Production ต้องมี SSL certificate
  validate_ssl = (
    var.environment == "production" && var.ssl_certificate_arn == null
    ? tobool("ERROR: Production requires ssl_certificate_arn")
    : true
  )
  
  # Production ต้องมี backup enabled
  validate_backup = (
    var.environment == "production" && !var.enable_backup
    ? tobool("ERROR: Production requires enable_backup = true")
    : true
  )
  
  # ทั้งหมดต้อง pass
  all_validations = [
    local.validate_ssl,
    local.validate_backup,
  ]
}

# ⚠️ Note: วิธีนี้จะ error ตอน plan ถ้า condition เป็น true
# เพราะ tobool("ERROR: ...") จะ fail
# ทำให้ Terraform ไม่ plan ต่อ
```

---

## สรุป Conditional Expressions

### Quick Reference

| Pattern | Syntax | Use Case |
|---------|--------|----------|
| Basic ternary | `cond ? a : b` | Simple if-else |
| Nested ternary | `c1 ? a : c2 ? b : c` | if-elseif-else |
| Null coalesce | `coalesce(a, b, default)` | Default values |
| Count toggle | `var.create ? 1 : 0` | Optional resources |
| List filter | `[for x in list : x if cond]` | Conditional elements |
| Map filter | `{for k,v in map : k => v if cond}` | Conditional entries |
| Module toggle | `count = cond ? 1 : 0` | Optional modules |

### Decision Flowchart

```
ต้องการ conditional หรือไม่?
│
├─ ต้องการ resource หรือไม่?
│   └─ ใช้ count = condition ? 1 : 0
│
├─ ต้องการ value ที่ต่างกัน?
│   └─ ใช้ condition ? value_a : value_b
│
├─ ต้องการ default value?
│   └─ ใช้ coalesce(override, computed, hardcoded)
│
└─ ต้องการ filter list/map?
    └─ ใช้ for expression with if clause
```

💡 **Pro Tips:**
- ใช้ `locals` สำหรับ complex conditions เพื่อ reuse
- ตั้งชื่อ boolean locals ให้ชัดเจน: `is_production`, `has_compliance`
- ใช้ `coalesce()` แทน `var.x != null ? var.x : default`
- ทดสอบ conditions ด้วย `terraform console`

⚠️ **Common Mistakes:**
- ใช้ `aws_resource.name[0].id` โดยไม่ check `count > 0`
- ลืมว่า `coalesce()` ข้าม empty string ด้วย
- Nested ternary อ่านยาก - พิจารณา lookup map แทน
- ใช้ conditional กับ `for_each` แทนที่จะ filter map ก่อน

---

*ก่อนหน้า: [Part 006 - HCL Built-in Functions](part-006.md)*
*ต่อไป: [Part 008 - HCL For Expressions](part-008.md)*
