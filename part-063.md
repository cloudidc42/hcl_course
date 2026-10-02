# Part 063: Output Values: Advanced Patterns
## Output Values: รูปแบบขั้นสูง
### Steps 621-630

---

## บทนำ (Introduction)

Output values ใน Terraform ทำหน้าที่เป็น interface ระหว่าง modules และระหว่าง Terraform configurations ต่างๆ การออกแบบ outputs ที่ดีจะทำให้ module reusable และ maintainable

---

## Step 621: Outputs as Module Interfaces

### Outputs ในฐานะ interface ของ module

```hcl
# ==========================================
# MODULE OUTPUT INTERFACE DESIGN
# ==========================================
# ไฟล์: modules/vpc/outputs.tf

# Output: VPC ID - ค่าที่ consumer ต้องการบ่อยที่สุด
output "vpc_id" {
  description = "ID ของ VPC"
  value       = aws_vpc.main.id
}

output "vpc_arn" {
  description = "ARN ของ VPC"
  value       = aws_vpc.main.arn
}

output "vpc_cidr_block" {
  description = "CIDR block ของ VPC"
  value       = aws_vpc.main.cidr_block
}

# Subnet outputs - return as lists for easy use
output "public_subnet_ids" {
  description = "IDs ของ public subnets"
  value       = aws_subnet.public[*].id
}

output "private_subnet_ids" {
  description = "IDs ของ private subnets"
  value       = aws_subnet.private[*].id
}

output "database_subnet_ids" {
  description = "IDs ของ database subnets"
  value       = aws_subnet.database[*].id
}

# Output เป็น map เพื่อให้ access ได้ง่ายขึ้น
output "public_subnets_by_az" {
  description = "Map ของ public subnet ID ต่อ AZ"
  value = {
    for subnet in aws_subnet.public :
    subnet.availability_zone => subnet.id
  }
}

# Route table outputs
output "public_route_table_id" {
  description = "ID ของ public route table"
  value       = aws_route_table.public.id
}

output "private_route_table_ids" {
  description = "IDs ของ private route tables"
  value       = aws_route_table.private[*].id
}

# NAT Gateway
output "nat_gateway_ids" {
  description = "IDs ของ NAT Gateways"
  value       = aws_nat_gateway.main[*].id
}

output "nat_public_ips" {
  description = "Public IPs ของ NAT Gateways"
  value       = aws_eip.nat[*].public_ip
}

# Internet Gateway
output "internet_gateway_id" {
  description = "ID ของ Internet Gateway"
  value       = aws_internet_gateway.main.id
}
```

---

## Step 622: Designing Clean Output Interfaces

### หลักการออกแบบ output ที่ดี

```hcl
# ==========================================
# CLEAN OUTPUT INTERFACE PRINCIPLES
# ==========================================

# Principle 1: Output ทุกอย่างที่ consumer อาจต้องการ
# modules/rds/outputs.tf

output "db_instance_id" {
  description = "ID ของ RDS instance"
  value       = aws_db_instance.main.id
}

output "db_instance_arn" {
  description = "ARN ของ RDS instance"
  value       = aws_db_instance.main.arn
}

output "db_endpoint" {
  description = "Connection endpoint ของ database (hostname:port)"
  value       = aws_db_instance.main.endpoint
}

output "db_host" {
  description = "Hostname ของ database (ไม่มี port)"
  value       = aws_db_instance.main.address
}

output "db_port" {
  description = "Port ของ database"
  value       = aws_db_instance.main.port
}

output "db_name" {
  description = "ชื่อ database"
  value       = aws_db_instance.main.db_name
}

output "db_username" {
  description = "Username สำหรับ database"
  value       = aws_db_instance.main.username
}

# Principle 2: Output structured objects สำหรับ convenience
output "db_connection_info" {
  description = "Object รวบรวมข้อมูลการเชื่อมต่อ database"
  value = {
    host     = aws_db_instance.main.address
    port     = aws_db_instance.main.port
    name     = aws_db_instance.main.db_name
    username = aws_db_instance.main.username
    endpoint = aws_db_instance.main.endpoint
  }
}

# Principle 3: ให้ outputs ที่ชัดเจน สื่อความหมาย
output "security_group_id" {
  description = "ID ของ security group ที่ protect database"
  value       = aws_security_group.rds.id
}

output "subnet_group_id" {
  description = "ID ของ DB subnet group"
  value       = aws_db_subnet_group.main.id
}

output "parameter_group_name" {
  description = "ชื่อของ DB parameter group"
  value       = aws_db_parameter_group.main.name
}
```

---

## Step 623: Output All vs Specific Attributes

```hcl
# ==========================================
# OUTPUT: ALL vs SPECIFIC ATTRIBUTES
# ==========================================

# แนวทาง 1: Output ทั้ง resource (ไม่แนะนำ)
# output "ec2_instance" {
#   value = aws_instance.web  # ดึงออกมาทั้งหมด
# }
# ปัญหา: ผู้ใช้ module ไม่รู้ว่า attribute ไหนใช้ได้
# และ sensitive attributes อาจโดน expose

# แนวทาง 2: Output specific attributes ที่จำเป็น (แนะนำ)
output "instance_id" {
  description = "EC2 Instance ID"
  value       = aws_instance.web.id
}

output "instance_public_ip" {
  description = "Public IP address"
  value       = aws_instance.web.public_ip
}

output "instance_private_ip" {
  description = "Private IP address"
  value       = aws_instance.web.private_ip
}

# แนวทาง 3: Output object ของ key attributes
output "instance_info" {
  description = "ข้อมูลหลักของ EC2 instance"
  value = {
    id          = aws_instance.web.id
    public_ip   = aws_instance.web.public_ip
    private_ip  = aws_instance.web.private_ip
    dns_name    = aws_instance.web.public_dns
    az          = aws_instance.web.availability_zone
    subnet_id   = aws_instance.web.subnet_id
  }
}

# Output for multiple resources (for_each)
output "all_instance_ids" {
  description = "Map ของ instance IDs keyed by name"
  value = {
    for k, v in aws_instance.servers :
    k => v.id
  }
}

output "all_instance_ips" {
  description = "Map ของ private IPs keyed by name"
  value = {
    for k, v in aws_instance.servers :
    k => v.private_ip
  }
}
```

---

## Step 624: Sensitive Outputs

### การจัดการ sensitive outputs

```hcl
# ==========================================
# SENSITIVE OUTPUTS
# ==========================================

# sensitive = true: Terraform จะ mask ค่าใน output
output "db_password" {
  description = "Database master password"
  value       = random_password.db.result
  sensitive   = true
  # ใน terraform output จะแสดงเป็น (sensitive value)
  # แต่ยังสามารถอ้างอิงได้ใน code
}

output "api_key" {
  description = "API Key สำหรับ application"
  value       = aws_secretsmanager_secret_version.api_key.secret_string
  sensitive   = true
}

# การ consume sensitive output จาก module
# module.rds.db_password จะถูก treat เป็น sensitive
# ถ้าจะใช้ต้องระวัง:

resource "aws_ssm_parameter" "db_password" {
  name  = "/myapp/db-password"
  type  = "SecureString"
  value = module.rds.db_password  # ใช้ได้แต่จะถูก mask ใน plans
}

# ==========================================
# NON-SENSITIVE OVERRIDE
# ==========================================

# บางครั้งต้องการ "de-sensitize" output
# ใช้ nonsensitive() function (ด้วยความระมัดระวัง!)

output "public_cert_arn" {
  description = "ARN ของ ACM certificate (public, ไม่ sensitive)"
  value       = nonsensitive(aws_acm_certificate.main.arn)
  # ใช้เฉพาะเมื่อมั่นใจ 100% ว่าค่านี้ไม่ sensitive
}
```

---

## Step 625: Conditional Outputs

### Output แบบมีเงื่อนไข

```hcl
# ==========================================
# CONDITIONAL OUTPUTS
# ==========================================

variable "enable_monitoring" {
  type    = bool
  default = false
}

variable "create_eip" {
  type    = bool
  default = false
}

# Conditional output - คืน null ถ้าไม่ได้สร้าง
output "monitoring_dashboard_url" {
  description = "URL ของ monitoring dashboard (null ถ้าไม่ได้เปิดใช้)"
  value = var.enable_monitoring ? (
    "https://${var.region}.console.aws.amazon.com/cloudwatch/home"
  ) : null
}

# Conditional output สำหรับ resource ที่อาจไม่ถูกสร้าง
output "elastic_ip" {
  description = "Elastic IP address (null ถ้าไม่ได้ create)"
  value       = var.create_eip ? aws_eip.main[0].public_ip : null
}

# Output จาก count resource
resource "aws_eip" "nat" {
  count  = var.enable_nat_gateway ? length(var.azs) : 0
  domain = "vpc"
}

output "nat_gateway_ips" {
  description = "IPs ของ NAT Gateways (empty list ถ้าไม่มี)"
  value       = aws_eip.nat[*].public_ip
  # จะเป็น [] ถ้า count = 0
}

# Output จาก for_each resource
resource "aws_route53_record" "aliases" {
  for_each = var.domain_aliases

  zone_id = aws_route53_zone.main.zone_id
  name    = each.key
  type    = "CNAME"
  ttl     = 300
  records = [each.value]
}

output "dns_records_created" {
  description = "Map ของ DNS records ที่สร้าง"
  value = {
    for domain, record in aws_route53_record.aliases :
    domain => record.fqdn
  }
  # จะเป็น {} ถ้าไม่มี aliases
}

# ==========================================
# TRY() FOR SAFE CONDITIONAL OUTPUTS
# ==========================================

output "load_balancer_dns" {
  description = "DNS name ของ load balancer"
  # try() ป้องกัน error ถ้า resource ไม่ถูกสร้าง
  value = try(aws_lb.main.dns_name, null)
}

output "certificate_arn" {
  description = "ARN ของ ACM certificate"
  value = try(aws_acm_certificate_validation.main.certificate_arn, null)
}
```

---

## Step 626: Outputs with depends_on

### การใช้ depends_on ใน outputs

```hcl
# ==========================================
# OUTPUTS WITH DEPENDS_ON
# ==========================================

# โดยปกติ output ไม่ต้องการ depends_on
# แต่บางกรณีที่ Terraform ไม่รู้ dependency จาก value เพียงอย่างเดียว

# ตัวอย่าง: output URL ที่ต้องรอให้ DNS propagate
output "app_url" {
  description = "URL ของ application"
  value       = "https://${var.domain_name}"

  depends_on = [
    aws_route53_record.app,
    aws_acm_certificate_validation.main
  ]
  # บอก Terraform ว่า URL นี้ใช้ได้แต่หลังจาก DNS และ cert พร้อม
}

# ตัวอย่าง: output ที่ต้องรอ IAM propagation
output "role_arn" {
  description = "ARN ของ IAM role"
  value       = aws_iam_role.main.arn

  depends_on = [
    aws_iam_role_policy_attachment.main
    # รอให้ policy attachment เสร็จก่อน
  ]
}

# ตัวอย่าง: output ที่ต้องรอ module เสร็จ
output "database_endpoint" {
  description = "Database endpoint"
  value       = module.rds.endpoint

  depends_on = [
    module.rds,
    aws_security_group_rule.allow_app_to_db
  ]
}
```

---

## Step 627: Precondition in Outputs (Check Blocks)

### การใช้ precondition ใน outputs

```hcl
# ==========================================
# PRECONDITION IN OUTPUTS
# ==========================================

output "api_endpoint" {
  description = "API endpoint URL"
  value       = "https://${aws_api_gateway_domain_name.main.domain_name}"

  precondition {
    condition     = aws_api_gateway_domain_name.main.certificate_arn != ""
    error_message = "API Gateway domain must have a valid certificate before outputting endpoint."
  }
}

output "alb_dns_name" {
  description = "DNS name ของ Application Load Balancer"
  value       = aws_lb.main.dns_name

  precondition {
    condition     = aws_lb.main.state == "active"
    error_message = "Load balancer must be in 'active' state before it can be used."
  }
}

output "database_connection_string" {
  description = "Database connection string"
  value       = "postgresql://${aws_db_instance.main.username}@${aws_db_instance.main.endpoint}/${aws_db_instance.main.db_name}"

  precondition {
    condition     = aws_db_instance.main.status == "available"
    error_message = "Database must be in 'available' status before generating connection string."
  }
}

# ==========================================
# CHECK BLOCKS (Terraform 1.5+)
# ==========================================

# Check blocks ทำงานหลัง apply - validate state ที่ควรจะเป็น
check "alb_healthy" {
  data "aws_lb" "main" {
    arn = aws_lb.main.arn
  }

  assert {
    condition     = data.aws_lb.main.state == "active"
    error_message = "Application Load Balancer is not in active state after creation."
  }
}

check "rds_available" {
  data "aws_db_instance" "main" {
    db_instance_identifier = aws_db_instance.main.identifier
  }

  assert {
    condition     = data.aws_db_instance.main.db_instance_status == "available"
    error_message = "RDS instance is not available after creation."
  }
}
```

---

## Step 628: Remote State Patterns

### การใช้ outputs กับ remote state

```hcl
# ==========================================
# TERRAFORM_REMOTE_STATE DATA SOURCE
# ==========================================

# Configuration 1: VPC stack (creates networking)
# File: networking/outputs.tf

output "vpc_id" {
  description = "VPC ID"
  value       = aws_vpc.main.id
}

output "private_subnet_ids" {
  description = "Private subnet IDs"
  value       = aws_subnet.private[*].id
}

output "public_subnet_ids" {
  description = "Public subnet IDs"
  value       = aws_subnet.public[*].id
}

# Configuration 2: Application stack (uses networking)
# File: application/main.tf

data "terraform_remote_state" "networking" {
  backend = "s3"
  config = {
    bucket = "my-terraform-state"
    key    = "networking/terraform.tfstate"
    region = "ap-southeast-1"
  }
}

# ใช้ outputs จาก remote state
resource "aws_instance" "app" {
  ami           = "ami-12345"
  instance_type = "t3.medium"

  # ใช้ VPC outputs จาก networking stack
  subnet_id              = data.terraform_remote_state.networking.outputs.private_subnet_ids[0]
  vpc_security_group_ids = [aws_security_group.app.id]
}

resource "aws_security_group" "app" {
  name   = "app-sg"
  vpc_id = data.terraform_remote_state.networking.outputs.vpc_id
}

# ==========================================
# MULTI-LEVEL REMOTE STATE
# ==========================================

# Level 1: Foundation (accounts, organizations)
data "terraform_remote_state" "foundation" {
  backend = "s3"
  config = {
    bucket = "tf-state-foundation"
    key    = "foundation/terraform.tfstate"
    region = "us-east-1"
  }
}

# Level 2: Networking (VPCs, subnets)
data "terraform_remote_state" "networking" {
  backend = "s3"
  config = {
    bucket = "tf-state-${var.environment}"
    key    = "networking/terraform.tfstate"
    region = var.region
  }
}

# Level 3: Platform (ECS cluster, RDS, etc.)
data "terraform_remote_state" "platform" {
  backend = "s3"
  config = {
    bucket = "tf-state-${var.environment}"
    key    = "platform/terraform.tfstate"
    region = var.region
  }
}

# Level 4: Application - uses all above
locals {
  # Access outputs from different layers
  vpc_id           = data.terraform_remote_state.networking.outputs.vpc_id
  private_subnets  = data.terraform_remote_state.networking.outputs.private_subnet_ids
  ecs_cluster_arn  = data.terraform_remote_state.platform.outputs.ecs_cluster_arn
  rds_endpoint     = data.terraform_remote_state.platform.outputs.rds_endpoint
  kms_key_arn      = data.terraform_remote_state.foundation.outputs.kms_key_arn
}

# ==========================================
# SAFER REMOTE STATE ACCESS WITH DEFAULTS
# ==========================================

locals {
  # ใช้ try() เพื่อป้องกัน error ถ้า output ไม่มี
  optional_feature_flag = try(
    data.terraform_remote_state.platform.outputs.feature_enabled,
    false  # default ถ้า output ไม่มี
  )

  monitoring_endpoint = try(
    data.terraform_remote_state.platform.outputs.monitoring_endpoint,
    ""
  )
}
```

---

## Step 629: Output Naming Conventions & Structured Outputs

### ข้อตกลงในการตั้งชื่อ

```hcl
# ==========================================
# OUTPUT NAMING CONVENTIONS
# ==========================================

# Pattern: resource_type + attribute
output "vpc_id" { ... }
output "vpc_arn" { ... }
output "subnet_ids" { ... }
output "security_group_id" { ... }

# Pattern: สำหรับ for_each resources - ใช้ map
output "instance_ids" {
  value = { for k, v in aws_instance.web : k => v.id }
}

# Pattern: สำหรับ count resources - ใช้ list
output "instance_ip_addresses" {
  value = aws_instance.web[*].private_ip
}

# ==========================================
# STRUCTURED OUTPUTS (Object)
# ==========================================

# แทนที่จะ output หลาย outputs แยกกัน
# ใช้ structured object แทน

output "vpc" {
  description = "VPC information"
  value = {
    id         = aws_vpc.main.id
    arn        = aws_vpc.main.arn
    cidr_block = aws_vpc.main.cidr_block
    subnets = {
      public   = aws_subnet.public[*].id
      private  = aws_subnet.private[*].id
      database = aws_subnet.database[*].id
    }
    nat_gateway_ips = aws_eip.nat[*].public_ip
    flow_logs_group = try(aws_cloudwatch_log_group.vpc_flow_logs.name, null)
  }
}

output "database" {
  description = "Database connection information"
  sensitive   = true  # ถ้า username/password รวมอยู่ด้วย
  value = {
    host     = aws_db_instance.main.address
    port     = aws_db_instance.main.port
    name     = aws_db_instance.main.db_name
    username = aws_db_instance.main.username
    endpoint = aws_db_instance.main.endpoint
    arn      = aws_db_instance.main.arn
    id       = aws_db_instance.main.id
  }
}

output "load_balancer" {
  description = "Load balancer details"
  value = {
    id       = aws_lb.main.id
    arn      = aws_lb.main.arn
    dns_name = aws_lb.main.dns_name
    zone_id  = aws_lb.main.zone_id
  }
}
```

---

## Step 630: Complete Module with Well-Designed Outputs

### ตัวอย่างสมบูรณ์ - ECS Module

```hcl
# ==========================================
# COMPLETE ECS MODULE OUTPUTS
# ==========================================
# modules/ecs-service/outputs.tf

# Service basics
output "service_id" {
  description = "ID ของ ECS service"
  value       = aws_ecs_service.main.id
}

output "service_name" {
  description = "ชื่อของ ECS service"
  value       = aws_ecs_service.main.name
}

output "service_cluster" {
  description = "ARN ของ cluster ที่ service รันอยู่"
  value       = aws_ecs_service.main.cluster
}

output "task_definition_arn" {
  description = "ARN ของ task definition ปัจจุบัน"
  value       = aws_ecs_task_definition.main.arn
}

output "task_definition_family" {
  description = "Family ของ task definition"
  value       = aws_ecs_task_definition.main.family
}

output "task_definition_revision" {
  description = "Revision ของ task definition ปัจจุบัน"
  value       = aws_ecs_task_definition.main.revision
}

# IAM
output "task_role_arn" {
  description = "ARN ของ IAM role ที่ task ใช้"
  value       = aws_iam_role.task.arn
}

output "task_role_name" {
  description = "ชื่อ IAM role ที่ task ใช้"
  value       = aws_iam_role.task.name
}

output "execution_role_arn" {
  description = "ARN ของ ECS execution role"
  value       = aws_iam_role.execution.arn
}

# Security
output "security_group_id" {
  description = "ID ของ security group ของ service"
  value       = aws_security_group.service.id
}

# Logging
output "log_group_name" {
  description = "ชื่อ CloudWatch log group"
  value       = aws_cloudwatch_log_group.service.name
}

output "log_group_arn" {
  description = "ARN ของ CloudWatch log group"
  value       = aws_cloudwatch_log_group.service.arn
}

# Auto-scaling
output "autoscaling_target_resource_id" {
  description = "Resource ID ของ auto-scaling target"
  value = try(
    aws_appautoscaling_target.ecs_target[0].resource_id,
    null
  )
}

# Structured summary
output "service_info" {
  description = "ข้อมูลสรุปของ ECS service"
  value = {
    service = {
      id      = aws_ecs_service.main.id
      name    = aws_ecs_service.main.name
      cluster = aws_ecs_service.main.cluster
    }
    task_definition = {
      arn      = aws_ecs_task_definition.main.arn
      family   = aws_ecs_task_definition.main.family
      revision = aws_ecs_task_definition.main.revision
    }
    iam = {
      task_role_arn      = aws_iam_role.task.arn
      execution_role_arn = aws_iam_role.execution.arn
    }
    networking = {
      security_group_id = aws_security_group.service.id
    }
    logging = {
      log_group_name = aws_cloudwatch_log_group.service.name
      log_group_arn  = aws_cloudwatch_log_group.service.arn
    }
  }
}

# ==========================================
# CI/CD OUTPUTS - สำหรับการใช้กับ CI/CD pipelines
# ==========================================

output "deployment_config" {
  description = "ข้อมูลสำหรับ CI/CD deployment"
  value = {
    cluster_name         = split("/", aws_ecs_service.main.cluster)[1]
    service_name         = aws_ecs_service.main.name
    task_definition_arn  = aws_ecs_task_definition.main.arn
    container_name       = var.container_name
    container_port       = var.container_port
  }
}

# ==========================================
# USING OUTPUTS IN CI/CD
# ==========================================

# terraform output -json | jq

# bash script:
# #!/bin/bash
# OUTPUTS=$(terraform output -json)
# CLUSTER=$(echo $OUTPUTS | jq -r '.deployment_config.value.cluster_name')
# SERVICE=$(echo $OUTPUTS | jq -r '.deployment_config.value.service_name')
# aws ecs update-service --cluster $CLUSTER --service $SERVICE --force-new-deployment

# ==========================================
# OUTPUT DOCUMENTING BEST PRACTICES
# ==========================================

# ทุก output ควรมี:
# 1. description ที่อธิบายชัดเจน
# 2. sensitive = true สำหรับ credentials
# 3. ชื่อที่สื่อความหมาย

# ตัวอย่าง output documentation ที่ดี
output "rds_connection_details" {
  description = <<-EOT
    Database connection details for connecting application servers.
    Use these values to configure your application's database connection pool.
    
    Note: password is not included - retrieve from AWS Secrets Manager
    using the secret_arn output.
  EOT
  value = {
    host     = aws_db_instance.main.address
    port     = aws_db_instance.main.port
    dbname   = aws_db_instance.main.db_name
    username = aws_db_instance.main.username
  }
}

output "rds_secret_arn" {
  description = "ARN ของ Secrets Manager secret ที่เก็บ database password"
  value       = aws_secretsmanager_secret.db_password.arn
}

# ==========================================
# AVOIDING CIRCULAR OUTPUTS
# ==========================================

# WRONG - อย่าสร้าง circular dependency
# module A output -> module B -> module A input

# RIGHT - ใช้ data sources หรือ remote state แทน
# module A -> S3 state
# module B -> data.terraform_remote_state.a -> use A outputs
```

---

## สรุป (Summary)

### Output Best Practices:

| Best Practice | คำอธิบาย |
|---------------|----------|
| ให้ `description` ทุก output | ช่วย document module interface |
| Mark sensitive outputs | ป้องกัน credential leakage |
| Output structured objects | สะดวกกว่าการ output หลายค่าแยกกัน |
| ใช้ `try()` สำหรับ optional resources | ป้องกัน error กรณี resource ไม่ถูกสร้าง |
| หลีกเลี่ยง circular outputs | ทำให้ dependency graph ซับซ้อน |
| Output ทุกสิ่งที่ consumer อาจต้องการ | ลดการต้องแก้ module ภายหลัง |

### Output สำหรับ Remote State:
```hcl
# เข้าถึง output จาก remote state
data.terraform_remote_state.vpc.outputs.vpc_id
data.terraform_remote_state.vpc.outputs.private_subnet_ids[0]
data.terraform_remote_state.vpc.outputs.vpc.subnets.private
```

---

*จบ Part 063 - Output Values: Advanced Patterns*
