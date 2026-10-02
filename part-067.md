# Part 067: Count & For_each Mastery
## Count & For_each: ความเชี่ยวชาญ
### Steps 661-670

---

## บทนำ (Introduction)

`count` และ `for_each` เป็น meta-arguments ที่ช่วยให้สร้าง resources หลายตัวได้โดยไม่ต้องเขียน resource block ซ้ำ การเข้าใจความแตกต่างและรู้ว่าควรใช้อันไหนเมื่อไรเป็นทักษะสำคัญ

---

## Step 661: Count Basics

### การใช้ count พื้นฐาน

```hcl
# ==========================================
# COUNT BASICS
# ==========================================

# สร้าง 3 EC2 instances
resource "aws_instance" "web" {
  count = 3

  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t3.micro"

  tags = {
    # count.index = 0, 1, 2
    Name = "web-server-${count.index + 1}"
  }
}

# อ้างอิง count resources
output "instance_ids" {
  value = aws_instance.web[*].id  # Splat expression
  # Result: ["id-1", "id-2", "id-3"]
}

# อ้างอิง specific index
output "first_instance_id" {
  value = aws_instance.web[0].id
}

# ==========================================
# COUNT WITH VARIABLE
# ==========================================

variable "instance_count" {
  type    = number
  default = 2
}

resource "aws_instance" "app" {
  count = var.instance_count

  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t3.micro"

  tags = {
    Name = "${var.project}-app-${count.index + 1}"
  }
}

# ==========================================
# COUNT WITH LIST LENGTH
# ==========================================

variable "availability_zones" {
  type    = list(string)
  default = ["ap-southeast-1a", "ap-southeast-1b"]
}

# สร้าง subnet per AZ
resource "aws_subnet" "public" {
  count = length(var.availability_zones)

  vpc_id            = aws_vpc.main.id
  cidr_block        = cidrsubnet(var.vpc_cidr, 8, count.index)
  availability_zone = var.availability_zones[count.index]
  # count.index ใช้ access element ใน list

  tags = {
    Name = "public-subnet-${var.availability_zones[count.index]}"
  }
}

# ==========================================
# COUNT.INDEX EXAMPLES
# ==========================================

variable "server_names" {
  type    = list(string)
  default = ["web", "app", "cache"]
}

resource "aws_instance" "servers" {
  count = length(var.server_names)

  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t3.micro"

  tags = {
    Name = var.server_names[count.index]
    Index = count.index
    Role  = var.server_names[count.index]
  }
}

# Access all instances:
output "all_server_ids" {
  value = aws_instance.servers[*].id
  # ["id-0", "id-1", "id-2"]
}

# Access specific:
output "web_server_id" {
  value = aws_instance.servers[0].id  # index 0 = web
}
```

---

## Step 662: Count = 0 for Conditional Resources

### การใช้ count สำหรับ conditional resource creation

```hcl
# ==========================================
# COUNT = 0 FOR CONDITIONAL RESOURCES
# ==========================================

variable "create_monitoring" {
  type    = bool
  default = false
}

# Create monitoring resources only if enabled
resource "aws_cloudwatch_log_group" "app" {
  count = var.create_monitoring ? 1 : 0

  name              = "/aws/app/myapp"
  retention_in_days = 30
}

# อ้างอิง conditional resource
output "log_group_arn" {
  value = var.create_monitoring ? aws_cloudwatch_log_group.app[0].arn : null
}

# ==========================================
# CONDITIONAL RESOURCE BASED ON ENV
# ==========================================

variable "environment" {
  type = string
}

# Bastion host: สร้างเฉพาะ dev/staging
resource "aws_instance" "bastion" {
  count = var.environment != "prod" ? 1 : 0

  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t3.micro"
  subnet_id     = aws_subnet.public[0].id

  tags = {
    Name = "bastion-${var.environment}"
    Role = "bastion"
  }
}

# WAF: สร้างเฉพาะ prod
resource "aws_wafv2_web_acl" "main" {
  count = var.environment == "prod" ? 1 : 0

  name  = "prod-waf"
  scope = "REGIONAL"
  # ...
}

# ==========================================
# MULTIPLE CONDITIONAL RESOURCES
# ==========================================

variable "enable_features" {
  type = object({
    nat_gateway    = bool
    flow_logs      = bool
    vpn_gateway    = bool
    vpc_endpoints  = bool
  })
  default = {
    nat_gateway   = true
    flow_logs     = false
    vpn_gateway   = false
    vpc_endpoints = true
  }
}

resource "aws_nat_gateway" "main" {
  count = var.enable_features.nat_gateway ? length(var.azs) : 0

  allocation_id = aws_eip.nat[count.index].id
  subnet_id     = aws_subnet.public[count.index].id
}

resource "aws_flow_log" "vpc" {
  count = var.enable_features.flow_logs ? 1 : 0

  vpc_id       = aws_vpc.main.id
  traffic_type = "ALL"
  iam_role_arn = aws_iam_role.flow_logs[0].arn
  log_destination = aws_cloudwatch_log_group.flow_logs[0].arn
}
```

---

## Step 663: Pitfalls of Count with Lists

### ปัญหาของ count กับ lists (Index Shifting)

```hcl
# ==========================================
# THE INDEX SHIFTING PROBLEM
# ==========================================

variable "server_names" {
  type    = list(string)
  default = ["web", "app", "db"]
}

resource "aws_instance" "servers" {
  count = length(var.server_names)
  ami   = "ami-12345"
  tags  = { Name = var.server_names[count.index] }
}

# State ปัจจุบัน:
# aws_instance.servers[0] -> "web"
# aws_instance.servers[1] -> "app"
# aws_instance.servers[2] -> "db"

# ถ้าลบ "app" ออกจาก list:
# variable "server_names" = ["web", "db"]
# 
# Terraform จะ:
# aws_instance.servers[0] = "web" (no change)
# aws_instance.servers[1] = "app" -> DESTROY + CREATE "db"  !!!
# aws_instance.servers[2] -> DESTROY !!!
#
# เกิด index shifting - "db" ที่ index 2 ต้องถูกสร้างใหม่!
# นี่คือปัญหาหลักของ count + list

# ==========================================
# SOLUTION: ใช้ for_each แทน count
# ==========================================

# ใช้ for_each ด้วย toset()
resource "aws_instance" "servers_freach" {
  for_each = toset(["web", "app", "db"])

  ami           = "ami-12345"
  instance_type = "t3.micro"

  tags = {
    Name = each.key  # "web", "app", "db"
  }
}

# State:
# aws_instance.servers_freach["web"] -> "web"
# aws_instance.servers_freach["app"] -> "app"
# aws_instance.servers_freach["db"]  -> "db"
#
# ถ้าลบ "app":
# for_each = toset(["web", "db"])
# 
# Terraform จะ:
# aws_instance.servers_freach["web"] -> no change
# aws_instance.servers_freach["app"] -> DESTROY (only "app")
# aws_instance.servers_freach["db"]  -> no change !!!

# ==========================================
# WHEN COUNT IS APPROPRIATE
# ==========================================

# 1. สร้าง identical resources (ไม่มี name)
resource "aws_iam_access_key" "app" {
  count = 2

  user = aws_iam_user.app.name
}
# ไม่มีชื่อ unique - count เหมาะ

# 2. สร้าง resource เดียว (count = 1 หรือ 0)
resource "aws_cloudwatch_alarm" "cpu" {
  count = var.enable_monitoring ? 1 : 0

  alarm_name = "high-cpu"
  # ...
}

# 3. Resources ที่ต้องการ ordering
resource "null_resource" "migrations" {
  count = length(var.sql_files)

  triggers = {
    script = var.sql_files[count.index]
  }
  # ทำตาม index order
}
```

---

## Step 664: For_each with Sets and Maps

### การใช้ for_each

```hcl
# ==========================================
# FOR_EACH WITH SET OF STRINGS
# ==========================================

resource "aws_iam_user" "developers" {
  for_each = toset(["alice", "bob", "charlie"])

  name = each.key  # "alice", "bob", "charlie"
  # each.value == each.key สำหรับ set
}

# IAM Group membership
resource "aws_iam_user_group_membership" "dev_group" {
  for_each = toset(["alice", "bob", "charlie"])

  user = aws_iam_user.developers[each.key].name

  groups = ["developers"]
}

# ==========================================
# FOR_EACH WITH MAP OF STRINGS
# ==========================================

variable "buckets" {
  type = map(string)
  default = {
    "app-data"    = "us-east-1"
    "app-backups" = "ap-southeast-1"
    "app-logs"    = "eu-west-1"
  }
}

resource "aws_s3_bucket" "app" {
  for_each = var.buckets

  bucket = each.key    # "app-data", "app-backups", etc.
  region = each.value  # "us-east-1", "ap-southeast-1", etc.
}

# ==========================================
# FOR_EACH WITH MAP OF OBJECTS
# ==========================================

variable "environments" {
  type = map(object({
    instance_type = string
    count         = number
    tags          = map(string)
  }))
  default = {
    "dev" = {
      instance_type = "t3.micro"
      count         = 1
      tags          = { Environment = "dev" }
    }
    "staging" = {
      instance_type = "t3.medium"
      count         = 2
      tags          = { Environment = "staging" }
    }
    "prod" = {
      instance_type = "c5.large"
      count         = 5
      tags          = { Environment = "prod" }
    }
  }
}

# Create ASG per environment
resource "aws_autoscaling_group" "env" {
  for_each = var.environments

  name             = "${each.key}-asg"
  desired_capacity = each.value.count
  min_size         = 1
  max_size         = each.value.count * 2

  launch_template {
    id      = aws_launch_template.app[each.key].id
    version = "$Latest"
  }

  tag {
    key                 = "Environment"
    value               = each.key
    propagate_at_launch = true
  }
}

resource "aws_launch_template" "app" {
  for_each = var.environments

  name_prefix   = "${each.key}-"
  image_id      = data.aws_ami.ubuntu.id
  instance_type = each.value.instance_type
}
```

---

## Step 665: each.key and each.value

### การใช้ each.key และ each.value

```hcl
# ==========================================
# EACH.KEY AND EACH.VALUE PATTERNS
# ==========================================

# Pattern 1: Map of objects
variable "services" {
  type = map(object({
    port        = number
    protocol    = string
    cidr_blocks = list(string)
  }))
  default = {
    "http" = {
      port        = 80
      protocol    = "tcp"
      cidr_blocks = ["0.0.0.0/0"]
    }
    "https" = {
      port        = 443
      protocol    = "tcp"
      cidr_blocks = ["0.0.0.0/0"]
    }
    "api" = {
      port        = 8080
      protocol    = "tcp"
      cidr_blocks = ["10.0.0.0/8"]
    }
  }
}

resource "aws_security_group_rule" "ingress" {
  for_each = var.services

  security_group_id = aws_security_group.app.id
  type              = "ingress"
  from_port         = each.value.port    # 80, 443, 8080
  to_port           = each.value.port    # same
  protocol          = each.value.protocol # "tcp"
  cidr_blocks       = each.value.cidr_blocks
  description       = "Allow ${each.key} traffic"
}

# Pattern 2: Named outputs using each.key
output "service_rules" {
  value = {
    for k, v in aws_security_group_rule.ingress :
    k => v.id
  }
  # Result: { "http" = "sgr-xxx", "https" = "sgr-yyy", "api" = "sgr-zzz" }
}

# Pattern 3: Using each.key for naming
variable "databases" {
  type = map(object({
    engine        = string
    instance_type = string
  }))
  default = {
    "primary" = { engine = "postgres", instance_type = "db.t3.medium" }
    "replica" = { engine = "postgres", instance_type = "db.t3.small" }
    "analytics" = { engine = "mysql",   instance_type = "db.r6g.large" }
  }
}

resource "aws_db_instance" "databases" {
  for_each = var.databases

  identifier     = "${var.project}-${each.key}-db"  # "myapp-primary-db"
  engine         = each.value.engine
  instance_class = each.value.instance_type

  tags = {
    Name = "${var.project}-${each.key}"
    Role = each.key  # "primary", "replica", "analytics"
  }
}

# Pattern 4: Conditional on each.key
resource "aws_db_instance_role_association" "main" {
  for_each = {
    for k, v in var.databases : k => v
    if each.key == "primary"  # ทำเฉพาะ primary
  }

  db_instance_identifier = aws_db_instance.databases[each.key].identifier
  feature_name           = "s3Export"
  role_arn               = aws_iam_role.rds_s3.arn
}
```

---

## Step 666: for_each Derived from Other Resources

### for_each ที่ derived จาก resources อื่น

```hcl
# ==========================================
# FOR_EACH FROM DATA SOURCES
# ==========================================

# ดึง AZs จาก data source แล้วใช้ใน for_each
data "aws_availability_zones" "available" {
  state = "available"
}

# สร้าง subnet per AZ
resource "aws_subnet" "private" {
  for_each = toset(data.aws_availability_zones.available.names)

  vpc_id            = aws_vpc.main.id
  availability_zone = each.key

  cidr_block = cidrsubnet(
    var.vpc_cidr,
    4,
    index(data.aws_availability_zones.available.names, each.key)
  )

  tags = {
    Name = "private-${each.key}"
  }
}

# ==========================================
# FOR_EACH FROM CREATED RESOURCES
# ==========================================

variable "bucket_names" {
  type    = set(string)
  default = ["logs", "data", "backups"]
}

resource "aws_s3_bucket" "main" {
  for_each = var.bucket_names
  bucket   = "${var.project}-${each.key}"
}

# ใช้ created buckets ใน for_each อื่น
resource "aws_s3_bucket_versioning" "main" {
  for_each = aws_s3_bucket.main  # iterate over created buckets

  bucket = each.value.id

  versioning_configuration {
    status = "Enabled"
  }
}

resource "aws_s3_bucket_server_side_encryption_configuration" "main" {
  for_each = aws_s3_bucket.main  # same

  bucket = each.value.id

  rule {
    apply_server_side_encryption_by_default {
      sse_algorithm = "AES256"
    }
  }
}

# ==========================================
# FOR_EACH WITH COMPLEX EXPRESSION
# ==========================================

variable "vpc_config" {
  type = map(object({
    cidr    = string
    subnets = list(string)
  }))
  default = {
    "vpc-1" = { cidr = "10.0.0.0/16", subnets = ["10.0.1.0/24", "10.0.2.0/24"] }
    "vpc-2" = { cidr = "10.1.0.0/16", subnets = ["10.1.1.0/24", "10.1.2.0/24"] }
  }
}

# Flatten for for_each
locals {
  all_subnets = merge([
    for vpc_name, vpc in var.vpc_config : {
      for i, subnet in vpc.subnets :
      "${vpc_name}-subnet-${i}" => {
        vpc_name = vpc_name
        cidr     = subnet
        index    = i
      }
    }
  ]...)
  # Result:
  # {
  #   "vpc-1-subnet-0" = { vpc_name = "vpc-1", cidr = "10.0.1.0/24", index = 0 }
  #   "vpc-1-subnet-1" = { vpc_name = "vpc-1", cidr = "10.0.2.0/24", index = 1 }
  #   "vpc-2-subnet-0" = { vpc_name = "vpc-2", cidr = "10.1.1.0/24", index = 0 }
  #   "vpc-2-subnet-1" = { vpc_name = "vpc-2", cidr = "10.1.2.0/24", index = 1 }
  # }
}

resource "aws_subnet" "all" {
  for_each = local.all_subnets

  vpc_id     = aws_vpc.main[each.value.vpc_name].id
  cidr_block = each.value.cidr

  tags = {
    Name = each.key
  }
}
```

---

## Step 667: Converting Count to For_each

### วิธี migrate จาก count ไป for_each

```hcl
# ==========================================
# MIGRATING FROM COUNT TO FOR_EACH
# ==========================================

# BEFORE: ใช้ count
resource "aws_instance" "web_old" {
  count = 3

  ami           = "ami-12345"
  instance_type = "t3.micro"

  tags = {
    Name = "web-${count.index}"
  }
}
# State: aws_instance.web_old[0], [1], [2]

# AFTER: ใช้ for_each
resource "aws_instance" "web_new" {
  for_each = toset(["web-0", "web-1", "web-2"])

  ami           = "ami-12345"
  instance_type = "t3.micro"

  tags = {
    Name = each.key
  }
}
# State: aws_instance.web_new["web-0"], ["web-1"], ["web-2"]

# ==========================================
# HOW TO MIGRATE WITHOUT DESTROYING
# ==========================================

# ขั้นตอน:
# 1. เพิ่ม moved block (Terraform 1.1+)
moved {
  from = aws_instance.web_old[0]
  to   = aws_instance.web_new["web-0"]
}

moved {
  from = aws_instance.web_old[1]
  to   = aws_instance.web_new["web-1"]
}

moved {
  from = aws_instance.web_old[2]
  to   = aws_instance.web_new["web-2"]
}

# 2. terraform plan - ตรวจสอบว่าไม่มี destroy/create
# 3. terraform apply
# 4. ลบ moved blocks ออก
# 5. terraform plan ครั้งสุดท้ายเพื่อยืนยัน

# ==========================================
# TOSET() FUNCTION
# ==========================================

# toset() แปลง list เป็น set (remove duplicates)
locals {
  names = ["alice", "bob", "alice", "charlie"]  # duplicate "alice"
  unique_names = toset(local.names)
  # Result: {"alice", "bob", "charlie"} - ไม่มี order, ไม่มี duplicate
}

# ใช้กับ for_each
resource "aws_iam_user" "users" {
  for_each = toset(["alice", "bob", "charlie"])
  name = each.key
}
```

---

## Step 668: Splat Expressions

### Splat expressions กับ count และ for_each

```hcl
# ==========================================
# SPLAT EXPRESSIONS
# ==========================================

# COUNT SPLAT: resource[*].attribute
resource "aws_instance" "web" {
  count = 3
  ami   = "ami-12345"
  instance_type = "t3.micro"
}

output "all_ids" {
  value = aws_instance.web[*].id
  # ["i-111", "i-222", "i-333"]
}

output "all_private_ips" {
  value = aws_instance.web[*].private_ip
  # ["10.0.1.1", "10.0.1.2", "10.0.1.3"]
}

# สร้าง map จาก count resources
output "id_map" {
  value = {
    for i, inst in aws_instance.web :
    "web-${i}" => inst.id
  }
}

# ==========================================
# FOR_EACH SPLAT
# ==========================================

resource "aws_instance" "servers" {
  for_each = toset(["web", "app", "db"])

  ami           = "ami-12345"
  instance_type = "t3.micro"
}

# FOR_EACH ไม่มี [*] splat โดยตรง - ต้องใช้ values()
output "server_ids" {
  value = values(aws_instance.servers)[*].id
  # ["id-app", "id-db", "id-web"] (alphabetical order)
}

# หรือใช้ for expression ที่ readable กว่า
output "server_id_map" {
  value = {
    for name, instance in aws_instance.servers :
    name => instance.id
  }
  # { "web" = "i-111", "app" = "i-222", "db" = "i-333" }
}

output "all_server_ids_list" {
  value = [
    for name, instance in aws_instance.servers :
    instance.id
  ]
  # ["i-222", "i-333", "i-111"] (alphabetical order by key)
}

# ==========================================
# PRACTICAL PATTERNS
# ==========================================

# Get all subnet IDs for use in other resources
resource "aws_subnet" "private" {
  for_each = var.subnet_configs

  vpc_id     = aws_vpc.main.id
  cidr_block = each.value.cidr
  availability_zone = each.value.az
}

# Use in ALB
resource "aws_lb" "main" {
  name    = "my-alb"
  subnets = values(aws_subnet.private)[*].id
  # or
  # subnets = [for s in aws_subnet.private : s.id]
}

# Use in Auto Scaling Group
resource "aws_autoscaling_group" "main" {
  vpc_zone_identifier = [
    for k, v in aws_subnet.private : v.id
  ]
}
```

---

## Step 669: for_each in Modules

### การใช้ for_each กับ modules

```hcl
# ==========================================
# FOR_EACH IN MODULES
# ==========================================

variable "environments" {
  type = map(object({
    vpc_cidr      = string
    instance_type = string
    region        = string
  }))
  default = {
    "dev" = {
      vpc_cidr      = "10.0.0.0/16"
      instance_type = "t3.micro"
      region        = "ap-southeast-1"
    }
    "staging" = {
      vpc_cidr      = "10.1.0.0/16"
      instance_type = "t3.medium"
      region        = "ap-southeast-1"
    }
  }
}

# Create VPC per environment
module "vpc" {
  for_each = var.environments
  source   = "./modules/vpc"

  project     = var.project
  environment = each.key  # "dev", "staging"
  cidr_block  = each.value.vpc_cidr
}

# Create ECS cluster per environment
module "ecs" {
  for_each = var.environments
  source   = "./modules/ecs-cluster"

  project     = var.project
  environment = each.key

  # ใช้ output จาก vpc module ที่ matching key
  vpc_id = module.vpc[each.key].vpc_id
}

# Output map of all VPC IDs
output "vpc_ids" {
  value = {
    for env, vpc in module.vpc :
    env => vpc.vpc_id
  }
}

# ==========================================
# FOR_EACH WITH CONDITIONAL MODULE
# ==========================================

variable "services" {
  type = map(object({
    enabled = bool
    image   = string
    port    = number
  }))
}

# สร้างแค่ services ที่ enabled = true
module "service" {
  for_each = {
    for name, config in var.services :
    name => config
    if config.enabled
  }
  source = "./modules/ecs-service"

  service_name = each.key
  image        = each.value.image
  port         = each.value.port
}
```

---

## Step 670: Complex Patterns - N VMs with Different Configs

### รูปแบบที่ซับซ้อน

```hcl
# ==========================================
# COMPLEX PATTERN: N VMs WITH DIFFERENT CONFIGS
# ==========================================

variable "server_fleet" {
  type = map(object({
    count         = number
    instance_type = string
    subnet_tier   = string  # "public" or "private"
    extra_volumes = optional(list(number), [])
    userdata_path = optional(string)
  }))
  default = {
    "web" = {
      count         = 2
      instance_type = "t3.medium"
      subnet_tier   = "public"
    }
    "app" = {
      count         = 3
      instance_type = "c5.large"
      subnet_tier   = "private"
      extra_volumes = [100, 50]  # 2 extra volumes
    }
    "worker" = {
      count         = 5
      instance_type = "c5.xlarge"
      subnet_tier   = "private"
      extra_volumes = [200]
    }
  }
}

# Flatten: สร้าง map ของ instances ทั้งหมด
locals {
  # สร้าง individual instances จาก fleet definition
  all_instances = merge([
    for role, config in var.server_fleet : {
      for i in range(config.count) :
      "${role}-${i + 1}" => {
        role          = role
        index         = i + 1
        instance_type = config.instance_type
        subnet_tier   = config.subnet_tier
        extra_volumes = config.extra_volumes
        userdata_path = config.userdata_path
      }
    }
  ]...)
  # Result:
  # {
  #   "web-1"    = { role = "web",    index = 1, instance_type = "t3.medium",  ... }
  #   "web-2"    = { role = "web",    index = 2, instance_type = "t3.medium",  ... }
  #   "app-1"    = { role = "app",    index = 1, instance_type = "c5.large",   ... }
  #   "app-2"    = { role = "app",    index = 2, instance_type = "c5.large",   ... }
  #   "app-3"    = { role = "app",    index = 3, instance_type = "c5.large",   ... }
  #   "worker-1" = { role = "worker", index = 1, instance_type = "c5.xlarge",  ... }
  #   ...
  # }
}

# Create all instances
resource "aws_instance" "fleet" {
  for_each = local.all_instances

  ami           = data.aws_ami.ubuntu.id
  instance_type = each.value.instance_type

  subnet_id = each.value.subnet_tier == "public" ? (
    aws_subnet.public[each.value.index % length(aws_subnet.public)].id
  ) : (
    aws_subnet.private[each.value.index % length(aws_subnet.private)].id
  )

  tags = {
    Name  = each.key
    Role  = each.value.role
    Index = tostring(each.value.index)
  }

  lifecycle {
    ignore_changes = [ami]
  }
}

# Extra EBS volumes
locals {
  # สร้าง list ของ volumes สำหรับทุก instance
  all_volumes = merge([
    for instance_name, instance_config in local.all_instances : {
      for i, size in instance_config.extra_volumes :
      "${instance_name}-vol-${i + 1}" => {
        instance_name = instance_name
        volume_index  = i + 1
        size_gb       = size
      }
    }
    if length(instance_config.extra_volumes) > 0
  ]...)
}

resource "aws_ebs_volume" "extra" {
  for_each = local.all_volumes

  availability_zone = aws_instance.fleet[each.value.instance_name].availability_zone
  size              = each.value.size_gb
  type              = "gp3"
  encrypted         = true

  tags = {
    Name     = each.key
    Instance = each.value.instance_name
  }
}

resource "aws_volume_attachment" "extra" {
  for_each = local.all_volumes

  device_name = "/dev/sd${["f","g","h","i","j"][each.value.volume_index - 1]}"
  volume_id   = aws_ebs_volume.extra[each.key].id
  instance_id = aws_instance.fleet[each.value.instance_name].id
}

# ==========================================
# OUTPUT ORGANIZED BY ROLE
# ==========================================

output "fleet_summary" {
  value = {
    for role in distinct([for k, v in local.all_instances : v.role]) :
    role => {
      count = length([for k, v in local.all_instances : k if v.role == role])
      ids   = [
        for k, v in aws_instance.fleet :
        v.id
        if local.all_instances[k].role == role
      ]
      private_ips = [
        for k, v in aws_instance.fleet :
        v.private_ip
        if local.all_instances[k].role == role
      ]
    }
  }
}
```

---

## สรุป (Summary)

### Count vs For_each:

| Feature | count | for_each |
|---------|-------|----------|
| Address | `resource[0]` | `resource["key"]` |
| Change existing | Risky (index shift) | Safe (by key) |
| Use case | Identical resources | Named resources |
| Input | number | map or set |
| Delete middle | Recreates all after | Only deletes that key |

### Decision Guide:
```
ต้องการสร้างหลาย resources?
├── Resources identical (ไม่ต้อง identify by name)?
│   └── ใช้ count
├── Resources มีชื่อหรือ identity?
│   └── ใช้ for_each
│       ├── มี unique names? -> toset([...])
│       └── มี configs ต่างกัน? -> map(object({...}))
└── สร้าง resource เดียว (conditional)?
    └── ใช้ count = condition ? 1 : 0
```

---

*จบ Part 067 - Count & For_each Mastery*
