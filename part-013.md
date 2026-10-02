# Part 013: HCL Local Values (ค่า Local)
## Steps 121-130: การใช้งาน Local Values ใน Terraform

---

## บทนำ (Introduction)

Local Values (หรือเรียกสั้นๆ ว่า "locals") คือค่าที่คำนวณและกำหนดชื่อภายใน module เพื่อให้สามารถนำกลับมาใช้ซ้ำได้โดยไม่ต้องเขียนซ้ำ ต่างจาก variables (รับจากภายนอก) และ outputs (ส่งออกภายนอก) locals มีไว้สำหรับใช้งานภายใน module เท่านั้น

### เมื่อไหรควรใช้ Locals?

| ใช้ `variable` เมื่อ | ใช้ `local` เมื่อ | ใช้ `output` เมื่อ |
|---------------------|------------------|-------------------|
| รับค่าจากผู้ใช้ | คำนวณค่าจาก vars/data | ส่งค่าออกไปให้ module อื่น |
| ค่าต่างกันแต่ละ environment | ค่าคงที่ที่คำนวณได้ | แสดงผลให้ผู้ใช้เห็น |
| ค่าที่ user ควร override | Expression ที่ใช้ซ้ำ | Cross-module references |
| Secrets/credentials | Tag merging | Remote state data |
| Configuration options | Name generation | |

---

## Step 121: Locals Block Syntax

### โครงสร้างพื้นฐาน

```hcl
# locals block (plural - ไม่ใช่ local)
locals {
  local_name_1 = <expression>
  local_name_2 = <expression>
  # ...
}

# สามารถมีหลาย locals blocks ได้ในไฟล์เดียวกัน
locals {
  # Network locals
  vpc_name  = "${var.project}-${var.env}-vpc"
  igw_name  = "${var.project}-${var.env}-igw"
}

locals {
  # Compute locals
  app_name  = "${var.project}-${var.env}-app"
  web_name  = "${var.project}-${var.env}-web"
}
```

### การอ้างอิง local value

```hcl
# อ้างอิงด้วย local.<name> (singular)
resource "aws_vpc" "main" {
  cidr_block = var.vpc_cidr
  
  tags = {
    Name = local.vpc_name  # อ้างอิง local
  }
}
```

---

## Step 122: Simple Value Locals

### Constants และ Static Values

```hcl
# locals.tf

locals {
  # Constants ที่ใช้ซ้ำหลายที่
  http_port   = 80
  https_port  = 443
  ssh_port    = 22
  
  # String constants
  company_name   = "MyCompany"
  department     = "Engineering"
  managed_by     = "terraform"
  
  # Protocol constants
  tcp_protocol   = "tcp"
  udp_protocol   = "udp"
  all_protocols  = "-1"
  
  # CIDR constants  
  any_ipv4       = "0.0.0.0/0"
  any_ipv6       = "::/0"
  
  # Region/AZ info
  account_id     = data.aws_caller_identity.current.account_id
  region         = data.aws_region.current.name
  
  # Timestamp (evaluated once at plan time)
  timestamp      = timestamp()
  date_suffix    = formatdate("YYYYMMDD", timestamp())
}
```

### การใช้งาน

```hcl
resource "aws_security_group_rule" "http_ingress" {
  type        = "ingress"
  from_port   = local.http_port   # ใช้ local แทน 80
  to_port     = local.http_port
  protocol    = local.tcp_protocol
  cidr_blocks = [local.any_ipv4]
  
  security_group_id = aws_security_group.web.id
}

resource "aws_security_group_rule" "https_ingress" {
  type        = "ingress"
  from_port   = local.https_port  # ใช้ local แทน 443
  to_port     = local.https_port
  protocol    = local.tcp_protocol
  cidr_blocks = [local.any_ipv4]
  
  security_group_id = aws_security_group.web.id
}
```

---

## Step 123: Computed Locals

### Locals ที่ใช้ Expressions

```hcl
locals {
  # ชื่อ resource ที่ consistent
  name_prefix    = "${var.project_name}-${var.environment}"
  vpc_name       = "${local.name_prefix}-vpc"
  igw_name       = "${local.name_prefix}-igw"
  alb_name       = "${local.name_prefix}-alb"
  
  # ขึ้นอยู่กับ environment
  is_production  = var.environment == "prod"
  is_development = var.environment == "dev"
  
  # Computed instance type
  instance_type = local.is_production ? "t3.medium" : "t3.micro"
  
  # Computed สำหรับ RDS
  db_instance_class = local.is_production ? "db.t3.small" : "db.t3.micro"
  db_multi_az       = local.is_production ? true : false
  
  # Compute subnet count
  subnet_count = length(var.availability_zones)
  
  # Compute CIDR blocks programmatically
  public_subnet_cidrs  = [for i in range(local.subnet_count) : cidrsubnet(var.vpc_cidr, 8, i)]
  private_subnet_cidrs = [for i in range(local.subnet_count) : cidrsubnet(var.vpc_cidr, 8, i + 10)]
  
  # Compute domain
  full_domain      = "${var.subdomain}.${var.domain}"
  api_domain       = "api.${var.domain}"
  static_domain    = "static.${var.domain}"
}
```

---

## Step 124: Locals Referencing Other Locals

### Locals สามารถอ้างอิง Locals อื่นได้

```hcl
locals {
  # Level 1: base values
  project     = var.project_name
  environment = var.environment
  region      = var.region
  
  # Level 2: ใช้ level 1 locals
  name_prefix = "${local.project}-${local.environment}"
  resource_id = "${local.project}-${local.environment}-${local.region}"
  
  # Level 3: ใช้ level 2 locals
  vpc_name    = "${local.name_prefix}-vpc"
  alb_name    = "${local.name_prefix}-alb"
  ecs_name    = "${local.name_prefix}-ecs"
  rds_name    = "${local.name_prefix}-rds"
  
  # Level 4: ใช้ level 2 + 3
  s3_bucket_name  = "${local.resource_id}-assets"
  log_bucket_name = "${local.resource_id}-logs"
  
  # Tags ที่ใช้ name_prefix
  common_tags = {
    Project     = local.project
    Environment = local.environment
    Region      = local.region
    Prefix      = local.name_prefix
    ManagedBy   = "terraform"
  }
}
```

### ตัวอย่าง Chain ที่ซับซ้อน

```hcl
locals {
  # Base configuration
  base_cidr = var.vpc_cidr  # "10.0.0.0/16"
  
  # Compute network structure
  # Public: 10.0.0.0/24, 10.0.1.0/24, 10.0.2.0/24
  # Private: 10.0.10.0/24, 10.0.11.0/24, 10.0.12.0/24
  # Database: 10.0.20.0/24, 10.0.21.0/24, 10.0.22.0/24
  
  az_count = length(var.availability_zones)
  
  public_cidrs   = [for i in range(local.az_count) : cidrsubnet(local.base_cidr, 8, i)]
  private_cidrs  = [for i in range(local.az_count) : cidrsubnet(local.base_cidr, 8, i + 10)]
  database_cidrs = [for i in range(local.az_count) : cidrsubnet(local.base_cidr, 8, i + 20)]
  
  # All subnet CIDRs combined
  all_cidrs = concat(local.public_cidrs, local.private_cidrs, local.database_cidrs)
  
  # Count total subnets
  total_subnets = length(local.all_cidrs)
}
```

---

## Step 125: Common Tagging Pattern

### Tag Strategy ด้วย Locals

```hcl
# locals.tf

locals {
  # Required tags - ต้องมีทุก resource
  required_tags = {
    Project     = var.project_name
    Environment = var.environment
    ManagedBy   = "terraform"
    Repository  = "github.com/${var.github_org}/${var.github_repo}"
    TeamOwner   = var.team_name
  }
  
  # Optional tags - เพิ่มตาม use case
  optional_tags = {
    CostCenter   = var.cost_center
    ApplicationId = var.application_id
    Backup       = var.enable_backup ? "daily" : "none"
    DataClass    = var.data_classification
  }
  
  # Combined tags
  common_tags = merge(local.required_tags, local.optional_tags, var.additional_tags)
  
  # Specific resource tags
  compute_tags = merge(local.common_tags, {
    Tier = "Compute"
  })
  
  network_tags = merge(local.common_tags, {
    Tier = "Network"
  })
  
  database_tags = merge(local.common_tags, {
    Tier            = "Database"
    DataClass       = "Confidential"
    BackupPolicy    = "standard"
  })
}
```

### การใช้งาน Tags Pattern

```hcl
resource "aws_vpc" "main" {
  cidr_block = var.vpc_cidr
  tags       = merge(local.network_tags, { Name = "${local.name_prefix}-vpc" })
}

resource "aws_instance" "app" {
  ami           = data.aws_ami.amazon_linux.id
  instance_type = local.instance_type
  tags          = merge(local.compute_tags, { Name = "${local.name_prefix}-app" })
}

resource "aws_db_instance" "main" {
  engine        = "mysql"
  instance_class = local.db_instance_class
  tags          = merge(local.database_tags, { Name = "${local.name_prefix}-db" })
}
```

---

## Step 126: Locals for Complex Transformations

### Transform Data ด้วย Locals

```hcl
locals {
  # Transform list of objects เป็น map for for_each
  # Input variable:
  # users = [
  #   { name = "alice", role = "admin" },
  #   { name = "bob",   role = "developer" }
  # ]
  
  users_map = {
    for user in var.users : user.name => user
  }
  
  # Transform security group rules
  # Input: list of rule objects
  # Output: map keyed by rule name
  sg_rules_map = {
    for rule in var.security_group_rules : "${rule.protocol}-${rule.from_port}-${rule.to_port}" => rule
  }
  
  # Flatten nested structure
  # Input: map of environment => list of subnets
  # Output: flat list of all subnets
  all_subnets = flatten([
    for env, subnets in var.subnet_config : subnets
  ])
  
  # Filter list
  prod_instances = [
    for instance in var.instances : instance
    if instance.environment == "prod"
  ]
  
  # Group by attribute
  instances_by_type = {
    for instance in var.instances :
      instance.type => instance...  # group with ellipsis
  }
}
```

### Transform AWS AMI Data

```hcl
data "aws_ami" "available" {
  most_recent = true
  owners      = ["amazon"]

  filter {
    name   = "name"
    values = ["amzn2-ami-hvm-*-x86_64-gp2"]
  }
}

locals {
  # แปลง AMI data เป็น simple map
  ami_info = {
    id           = data.aws_ami.available.id
    name         = data.aws_ami.available.name
    creation_date = data.aws_ami.available.creation_date
    description  = data.aws_ami.available.description
  }
  
  # AMI ID สำหรับ instance type
  ami_id = data.aws_ami.available.id
}
```

---

## Step 127: String Manipulation Locals

### String Operations

```hcl
locals {
  # toLowerCase / toUpper
  project_lower = lower(var.project_name)   # "myproject"
  project_upper = upper(var.project_name)   # "MYPROJECT"
  
  # Replace characters (สำหรับ S3 bucket names ที่ไม่รับ underscore)
  bucket_name_safe = replace(lower(var.project_name), "_", "-")
  
  # Trim whitespace
  trimmed_name = trimspace(var.raw_name)
  
  # Concatenation
  full_domain = "${var.subdomain}.${var.domain}"
  
  # Format string
  resource_name = format("%s-%s-%s", var.project, var.environment, var.region)
  
  # Split string
  domain_parts  = split(".", var.domain)  # ["example", "com"]
  tld           = local.domain_parts[length(local.domain_parts) - 1]  # "com"
  
  # Join
  subnet_list   = join(", ", aws_subnet.public[*].id)  # "subnet-1, subnet-2"
  
  # Regex replace
  clean_name    = replace(var.name, "/[^a-zA-Z0-9-]/", "")
  
  # Padding
  padded_index  = format("%03d", 1)  # "001"
  
  # Base64
  encoded_config = base64encode(jsonencode({
    key = "value"
    num = 42
  }))
}
```

---

## Step 128: List/Map Operation Locals

### Collection Operations

```hcl
locals {
  # Concat lists
  all_ips = concat(var.web_server_ips, var.app_server_ips, var.db_server_ips)
  
  # Flatten nested lists
  all_subnet_ids = flatten([
    aws_subnet.public[*].id,
    aws_subnet.private[*].id,
    aws_subnet.database[*].id
  ])
  
  # Unique values
  unique_azs = distinct(concat(
    aws_subnet.public[*].availability_zone,
    aws_subnet.private[*].availability_zone
  ))
  
  # Sort
  sorted_subnet_ids = sort(aws_subnet.public[*].id)
  
  # Length
  subnet_count = length(aws_subnet.public)
  
  # Keys/Values from map
  tag_keys   = keys(var.tags)
  tag_values = values(var.tags)
  
  # Merge maps
  all_tags = merge(
    var.common_tags,
    var.resource_tags,
    { CreatedAt = timestamp() }
  )
  
  # zipmap - สร้าง map จาก 2 lists
  az_subnet_map = zipmap(
    aws_subnet.public[*].availability_zone,
    aws_subnet.public[*].id
  )
  
  # lookup with default
  instance_type = lookup(
    {
      dev     = "t3.micro"
      staging = "t3.small"
      prod    = "t3.medium"
    },
    var.environment,
    "t3.micro"  # default
  )
  
  # contains check
  is_allowed_region = contains(var.allowed_regions, var.region)
  
  # slice
  first_two_subnets = slice(aws_subnet.public[*].id, 0, 2)
}
```

### For Expressions

```hcl
locals {
  # Map transformation
  uppercase_tags = {
    for k, v in var.tags : upper(k) => v
  }
  
  # Filter map
  non_empty_tags = {
    for k, v in var.tags : k => v
    if v != ""
  }
  
  # List to map
  subnet_id_by_az = {
    for subnet in aws_subnet.public :
      subnet.availability_zone => subnet.id
  }
  
  # Conditional inclusion
  optional_tags = {
    for k, v in {
      "BillingCode" = var.billing_code
      "CostCenter"  = var.cost_center
      "Owner"       = var.owner_email
    } : k => v
    if v != null && v != ""
  }
  
  # Nested for expression
  sg_rule_descriptions = {
    for rule in var.security_group_rules :
      "${rule.type}-${rule.protocol}-${rule.from_port}" => 
        "${rule.type} ${rule.protocol} port ${rule.from_port}-${rule.to_port}"
  }
}
```

---

## Step 129: Conditional Locals

### Ternary Operations

```hcl
locals {
  # Simple conditional
  is_prod = var.environment == "prod"
  
  # Conditional values
  instance_type     = local.is_prod ? "t3.medium" : "t3.micro"
  min_capacity      = local.is_prod ? 3 : 1
  max_capacity      = local.is_prod ? 20 : 5
  multi_az          = local.is_prod ? true : false
  enable_encryption = local.is_prod ? true : false
  backup_retention  = local.is_prod ? 30 : 7
  deletion_protection = local.is_prod ? true : false
  
  # Nested conditional
  instance_type_detail = (
    var.environment == "prod" ? "t3.large" : (
      var.environment == "staging" ? "t3.medium" : 
      "t3.micro"
    )
  )
  
  # Conditional list
  vpc_security_groups = local.is_prod ? [
    aws_security_group.prod_web.id,
    aws_security_group.prod_app.id,
    aws_security_group.prod_monitoring.id,
  ] : [
    aws_security_group.dev_web.id,
    aws_security_group.dev_app.id,
  ]
  
  # Conditional resource creation
  create_nat_gateway   = var.environment != "dev"
  create_bastion_host  = var.enable_bastion || local.is_prod
  create_waf           = local.is_prod && var.enable_waf
  
  # Null coalescing pattern
  effective_kms_key  = var.kms_key_id != null ? var.kms_key_id : "aws/s3"
  effective_log_path = var.log_path != null ? var.log_path : "/var/log/${var.app_name}"
}
```

---

## Step 130: Performance and Best Practices

### Locals ประเมินค่าครั้งเดียว

```hcl
locals {
  # ✅ ดี: คำนวณครั้งเดียว ใช้ได้หลายที่
  timestamp_suffix = formatdate("YYYYMMDD-hhmmss", timestamp())
  
  # ✅ ดี: complex computation ทำครั้งเดียว
  subnet_matrix = {
    for az_index, az in var.availability_zones : az => {
      public_cidr  = cidrsubnet(var.vpc_cidr, 8, az_index)
      private_cidr = cidrsubnet(var.vpc_cidr, 8, az_index + 10)
      db_cidr      = cidrsubnet(var.vpc_cidr, 8, az_index + 20)
    }
  }
}

# ❌ ไม่ดี: เขียน expression ซ้ำในหลายที่
resource "aws_instance" "web" {
  tags = { Name = "${var.project}-${var.env}-web" }  # ซ้ำ!
}

resource "aws_instance" "app" {
  tags = { Name = "${var.project}-${var.env}-app" }  # ซ้ำ!
}

# ✅ ดี: ใช้ local
locals {
  name_prefix = "${var.project}-${var.env}"
}

resource "aws_instance" "web" {
  tags = { Name = "${local.name_prefix}-web" }
}

resource "aws_instance" "app" {
  tags = { Name = "${local.name_prefix}-app" }
}
```

### File Organization - locals.tf

```hcl
# locals.tf - ไฟล์สำหรับ locals โดยเฉพาะ

# ===== Naming Locals =====
locals {
  name_prefix = "${var.project_name}-${var.environment}"
  
  vpc_name    = "${local.name_prefix}-vpc"
  alb_name    = "${local.name_prefix}-alb"
  ecs_name    = "${local.name_prefix}-ecs"
  rds_name    = "${local.name_prefix}-rds"
  s3_name     = "${local.name_prefix}-assets"
}

# ===== Tag Locals =====
locals {
  required_tags = {
    Project     = var.project_name
    Environment = var.environment
    ManagedBy   = "terraform"
    Owner       = var.owner_email
  }
  
  common_tags = merge(local.required_tags, var.additional_tags)
}

# ===== Networking Locals =====
locals {
  az_count             = length(var.availability_zones)
  public_subnet_cidrs  = [for i in range(local.az_count) : cidrsubnet(var.vpc_cidr, 8, i)]
  private_subnet_cidrs = [for i in range(local.az_count) : cidrsubnet(var.vpc_cidr, 8, i + 10)]
}

# ===== Environment Config Locals =====
locals {
  is_production = var.environment == "prod"
  
  env_config = {
    dev = {
      instance_type      = "t3.micro"
      db_instance_class  = "db.t3.micro"
      min_capacity       = 1
      max_capacity       = 3
      multi_az           = false
    }
    staging = {
      instance_type      = "t3.small"
      db_instance_class  = "db.t3.small"
      min_capacity       = 1
      max_capacity       = 5
      multi_az           = false
    }
    prod = {
      instance_type      = "t3.medium"
      db_instance_class  = "db.t3.medium"
      min_capacity       = 3
      max_capacity       = 20
      multi_az           = true
    }
  }
  
  current_env_config = local.env_config[var.environment]
  instance_type      = local.current_env_config.instance_type
  db_instance_class  = local.current_env_config.db_instance_class
}
```

---

## Real-World Complete Example

### Infrastructure Locals สำหรับ Web Application

```hcl
# locals.tf - Complete production example

# Naming
locals {
  project     = lower(var.project_name)
  env         = lower(var.environment)
  region_short = {
    "ap-southeast-1" = "apse1"
    "us-east-1"      = "use1"
    "us-west-2"      = "usw2"
    "eu-west-1"      = "euw1"
  }[var.region]
  
  prefix       = "${local.project}-${local.env}"
  full_prefix  = "${local.project}-${local.env}-${local.region_short}"
  
  # Resource names
  vpc_name         = "${local.prefix}-vpc"
  alb_name         = "${local.prefix}-alb"
  ecs_cluster_name = "${local.prefix}-cluster"
  rds_identifier   = "${local.prefix}-mysql"
  redis_id         = "${local.prefix}-redis"
  s3_bucket_name   = "${local.full_prefix}-assets-${data.aws_caller_identity.current.account_id}"
  log_bucket_name  = "${local.full_prefix}-logs-${data.aws_caller_identity.current.account_id}"
}

# Tags
locals {
  base_tags = {
    Project     = var.project_name
    Environment = var.environment
    Region      = var.region
    ManagedBy   = "terraform"
    Repository  = var.repo_url
    UpdatedAt   = timestamp()
  }
  
  common_tags  = merge(local.base_tags, var.extra_tags)
  network_tags = merge(local.common_tags, { Layer = "Network" })
  compute_tags = merge(local.common_tags, { Layer = "Compute" })
  data_tags    = merge(local.common_tags, { Layer = "Data", DataClass = "Confidential" })
}

# Network config
locals {
  az_count = length(var.availability_zones)
  azs      = var.availability_zones
  
  public_cidrs   = [for i in range(local.az_count) : cidrsubnet(var.vpc_cidr, 4, i)]
  private_cidrs  = [for i in range(local.az_count) : cidrsubnet(var.vpc_cidr, 4, i + 4)]
  database_cidrs = [for i in range(local.az_count) : cidrsubnet(var.vpc_cidr, 4, i + 8)]
}

# Environment-specific config
locals {
  is_prod = var.environment == "prod"
  is_dev  = var.environment == "dev"
  
  # Scaling
  app_min_capacity     = local.is_prod ? 3 : 1
  app_max_capacity     = local.is_prod ? 50 : 5
  app_desired_capacity = local.is_prod ? 3 : 1
  
  # Instance sizes
  app_instance_type = local.is_prod ? "t3.medium" : "t3.micro"
  db_instance_class = local.is_prod ? "db.r5.large" : "db.t3.micro"
  cache_node_type   = local.is_prod ? "cache.r6g.large" : "cache.t3.micro"
  
  # High availability
  db_multi_az        = local.is_prod
  db_backup_days     = local.is_prod ? 30 : 1
  cache_num_nodes    = local.is_prod ? 3 : 1
  
  # Security
  deletion_protection       = local.is_prod
  enable_encryption         = local.is_prod || var.force_encryption
  enable_access_logs        = local.is_prod
  log_retention_days        = local.is_prod ? 90 : 7
  
  # URLs
  app_domain    = "${var.subdomain}.${var.domain}"
  api_domain    = "api.${var.domain}"
  admin_domain  = "admin.${var.domain}"
  
  # CORS origins
  allowed_origins = local.is_prod ? [
    "https://${local.app_domain}",
    "https://${local.api_domain}"
  ] : [
    "https://${local.app_domain}",
    "http://localhost:3000",
    "http://localhost:8080"
  ]
}

# Database connection info (non-sensitive)
locals {
  db_connection = {
    host     = aws_db_instance.main.address
    port     = aws_db_instance.main.port
    name     = var.db_name
    username = var.db_username
  }
  
  cache_connection = {
    host = aws_elasticache_cluster.main.cache_nodes[0].address
    port = aws_elasticache_cluster.main.cache_nodes[0].port
  }
}
```

---

## สรุป (Summary)

### เมื่อไหรใช้ Locals

```
ใช้ locals สำหรับ:
✅ ค่าที่ใช้ซ้ำหลายครั้งใน module
✅ Expression ที่ซับซ้อนที่ต้องการชื่อที่อ่านง่าย
✅ Tag merging และ naming patterns
✅ Conditional configuration ตาม environment
✅ Data transformation (for expressions)
✅ Constants ที่ไม่ต้องการให้ user override

อย่าใช้ locals สำหรับ:
❌ ค่าที่ต้องการรับจาก user → ใช้ variable
❌ ค่าที่ต้องการส่งออกให้ module อื่น → ใช้ output
❌ Secrets → ใช้ external secret manager
```

### ✅ Best Practices

1. **แยก locals เป็นหมวดหมู่** - naming, tags, network, env config
2. **สร้าง locals.tf แยก** สำหรับ module ที่ซับซ้อน
3. **ชื่อ locals ต้องสื่อความหมาย** - `is_production` ดีกว่า `prod_flag`
4. **อย่าทำ circular references** - local A ไม่ควรอ้างอิง local B ที่อ้างอิง A
5. **Group ที่เกี่ยวข้องกัน** - ใช้หลาย `locals {}` blocks แต่ละ block มีหัวข้อ

### ⚠️ Common Mistakes

```hcl
# ❌ Circular reference (จะ error)
locals {
  a = local.b
  b = local.a  # error!
}

# ❌ ใช้ locals สำหรับสิ่งที่ควรเป็น variable
locals {
  environment = "prod"  # ❌ ไม่ดี ควรเป็น variable
}

# ✅ ถูกต้อง
variable "environment" {
  default = "prod"
}

# ❌ ไม่มี naming pattern
locals {
  x = "val1"
  y = "val2"
  z = "val3"
}

# ✅ ดีกว่า
locals {
  vpc_name = "val1"
  alb_name = "val2"
  ecs_name = "val3"
}
```

### 💡 Pro Tips

1. ใช้ locals เป็น "documentation" - ชื่อที่ดีช่วยให้อ่านง่าย
2. ใช้ `lookup()` และ map ใน locals สำหรับ environment-specific config
3. `cidrsubnet()` function ใน locals ช่วย generate CIDR blocks อัตโนมัติ
4. Locals ช่วย DRY principle - Don't Repeat Yourself

---

*จบ Part 013 - HCL Local Values*
