# Part 012: HCL Output Values (ค่า Output)
## Steps 111-120: การใช้งาน Output Values ใน Terraform

---

## บทนำ (Introduction)

Output Values คือกลไกที่ทำให้ Terraform configurations สามารถส่งข้อมูลออกมาเพื่อ:
1. **แสดงผลให้ผู้ใช้เห็น** หลังจาก apply
2. **ส่งค่าระหว่าง modules** (module outputs)
3. **ใช้ใน remote state** - configurations อื่นอ่านค่าจาก state file
4. **Debug** - ดูค่าที่ Terraform คำนวณได้

Outputs เป็นส่วน "interface" ของ module ที่ expose ค่าออกมาให้ภายนอกใช้

---

## Step 111: Output Block Syntax พื้นฐาน

### โครงสร้างพื้นฐาน

```hcl
output "output_name" {
  description = "คำอธิบาย output"
  value       = <expression>
}
```

### ตัวอย่าง Output พื้นฐาน

```hcl
# outputs.tf

# Output ค่าจาก resource attribute
output "instance_id" {
  description = "ID of the EC2 instance"
  value       = aws_instance.web.id
}

output "instance_public_ip" {
  description = "Public IP address of the EC2 instance"
  value       = aws_instance.web.public_ip
}

output "instance_private_ip" {
  description = "Private IP address of the EC2 instance"
  value       = aws_instance.web.private_ip
}

output "vpc_id" {
  description = "ID of the VPC"
  value       = aws_vpc.main.id
}
```

---

## Step 112: Output Attributes

### value, description, sensitive

```hcl
# description attribute - คำอธิบาย output
output "rds_endpoint" {
  description = "The connection endpoint for the RDS instance in address:port format"
  value       = aws_db_instance.main.endpoint
}

# sensitive attribute - ซ่อนค่าใน terminal output
output "database_password" {
  description = "The master password for the RDS instance"
  value       = var.database_password
  sensitive   = true  # จะแสดงเป็น <sensitive> ใน terminal
}

output "connection_string" {
  description = "Full database connection string"
  value       = "postgresql://${var.db_user}:${var.db_password}@${aws_db_instance.main.endpoint}/${var.db_name}"
  sensitive   = true  # ต้องใส่ถ้าใช้ค่า sensitive
}

# depends_on - กำหนด dependency ให้ output
output "load_balancer_dns" {
  description = "DNS name of the load balancer"
  value       = aws_lb.main.dns_name
  
  depends_on = [
    aws_lb_listener.http,
    aws_lb_listener.https
  ]
}
```

---

## Step 113: Precondition in Outputs

### precondition - ตรวจสอบเงื่อนไขก่อน output (Terraform 1.2+)

```hcl
output "api_base_url" {
  description = "Base URL of the API"
  value       = "https://${aws_lb.main.dns_name}/api"

  precondition {
    condition     = aws_lb.main.state.code == "active"
    error_message = "Load balancer must be in active state before outputting URL."
  }
}

output "database_endpoint" {
  description = "Database endpoint"
  value       = aws_db_instance.main.endpoint

  precondition {
    condition     = aws_db_instance.main.status == "available"
    error_message = "Database must be in available status."
  }
}

output "certificate_arn" {
  description = "ACM certificate ARN"
  value       = aws_acm_certificate.main.arn

  precondition {
    condition     = aws_acm_certificate.main.status == "ISSUED"
    error_message = "Certificate must be issued/validated before use. Status: ${aws_acm_certificate.main.status}"
  }
}
```

---

## Step 114: Outputting Primitive Values

### String, Number, Bool outputs

```hcl
# String output
output "bucket_name" {
  description = "Name of the S3 bucket"
  value       = aws_s3_bucket.main.bucket
}

output "hosted_zone_id" {
  description = "Route53 hosted zone ID"
  value       = aws_route53_zone.main.zone_id
}

output "region" {
  description = "AWS region where resources are deployed"
  value       = var.region
}

# Number output
output "alb_zone_id" {
  description = "Canonical hosted zone ID of the ALB"
  value       = aws_lb.main.zone_id
}

# Boolean output (computed)
output "is_multi_az" {
  description = "Whether the RDS instance uses Multi-AZ deployment"
  value       = aws_db_instance.main.multi_az
}

# Computed string output
output "name_prefix" {
  description = "Naming prefix used for all resources"
  value       = "${var.project_name}-${var.environment}"
}

output "arn" {
  description = "ARN of the S3 bucket"
  value       = aws_s3_bucket.main.arn
}
```

---

## Step 115: Outputting Complex Values

### List Output

```hcl
# List of resource IDs
output "public_subnet_ids" {
  description = "List of IDs of public subnets"
  value       = aws_subnet.public[*].id
}

output "private_subnet_ids" {
  description = "List of IDs of private subnets"
  value       = aws_subnet.private[*].id
}

output "all_subnet_ids" {
  description = "List of all subnet IDs (public and private)"
  value       = concat(aws_subnet.public[*].id, aws_subnet.private[*].id)
}

# List of resource attributes
output "instance_public_ips" {
  description = "Public IP addresses of all instances"
  value       = aws_instance.web[*].public_ip
}

output "availability_zones_used" {
  description = "Availability zones where subnets were created"
  value       = aws_subnet.public[*].availability_zone
}

# Sorted list
output "security_group_ids" {
  description = "List of security group IDs"
  value       = sort([
    aws_security_group.web.id,
    aws_security_group.app.id,
    aws_security_group.db.id,
  ])
}
```

### Map Output

```hcl
# Map output
output "subnet_id_by_az" {
  description = "Map of availability zone to subnet ID"
  value = {
    for subnet in aws_subnet.public : subnet.availability_zone => subnet.id
  }
}

output "instance_details" {
  description = "Map of instance name to details"
  value = {
    for instance in aws_instance.web : instance.tags["Name"] => {
      id         = instance.id
      public_ip  = instance.public_ip
      private_ip = instance.private_ip
      az         = instance.availability_zone
    }
  }
}

output "endpoints" {
  description = "Map of service endpoints"
  value = {
    web      = aws_lb.main.dns_name
    api      = aws_lb.api.dns_name
    database = aws_db_instance.main.endpoint
    cache    = aws_elasticache_cluster.main.cache_nodes[0].address
  }
}
```

### Object Output

```hcl
# Structured object output
output "vpc_info" {
  description = "VPC information"
  value = {
    id              = aws_vpc.main.id
    cidr_block      = aws_vpc.main.cidr_block
    public_subnets  = aws_subnet.public[*].id
    private_subnets = aws_subnet.private[*].id
    nat_gateway_ids = aws_nat_gateway.main[*].id
  }
}

output "database_info" {
  description = "Database connection information"
  value = {
    host     = aws_db_instance.main.address
    port     = aws_db_instance.main.port
    name     = aws_db_instance.main.db_name
    username = aws_db_instance.main.username
    endpoint = aws_db_instance.main.endpoint
  }
  sensitive = false  # ไม่ sensitive เพราะไม่มี password
}

output "load_balancer_info" {
  description = "Load balancer information"
  value = {
    arn         = aws_lb.main.arn
    dns_name    = aws_lb.main.dns_name
    zone_id     = aws_lb.main.zone_id
    https_url   = "https://${aws_lb.main.dns_name}"
  }
}
```

---

## Step 116: Outputting Resource Attributes

### Computed Attributes ที่สำคัญ

```hcl
# EC2 Instance attributes
output "ec2_details" {
  description = "EC2 instance details"
  value = {
    id               = aws_instance.web.id           # i-1234567890abcdef0
    arn              = aws_instance.web.arn           # arn:aws:ec2:...
    public_ip        = aws_instance.web.public_ip     # filled after create
    private_ip       = aws_instance.web.private_ip    # filled after create
    public_dns       = aws_instance.web.public_dns
    private_dns      = aws_instance.web.private_dns
    availability_zone = aws_instance.web.availability_zone
    vpc_id           = aws_instance.web.vpc_id
    subnet_id        = aws_instance.web.subnet_id
  }
}

# S3 Bucket attributes
output "s3_bucket_details" {
  description = "S3 bucket details"
  value = {
    id           = aws_s3_bucket.main.id           # bucket name
    arn          = aws_s3_bucket.main.arn
    region       = aws_s3_bucket.main.region
    domain_name  = aws_s3_bucket.main.bucket_domain_name
    website_endpoint = aws_s3_bucket_website_configuration.main.website_endpoint
  }
}

# RDS Instance attributes
output "rds_details" {
  description = "RDS instance details"
  value = {
    id               = aws_db_instance.main.id
    arn              = aws_db_instance.main.arn
    endpoint         = aws_db_instance.main.endpoint   # host:port
    address          = aws_db_instance.main.address    # host only
    port             = aws_db_instance.main.port
    availability_zone = aws_db_instance.main.availability_zone
    status           = aws_db_instance.main.status
    resource_id      = aws_db_instance.main.resource_id
  }
}

# EKS Cluster attributes
output "eks_cluster_info" {
  description = "EKS cluster connection info"
  value = {
    name                    = aws_eks_cluster.main.name
    endpoint                = aws_eks_cluster.main.endpoint
    certificate_authority   = aws_eks_cluster.main.certificate_authority[0].data
    version                 = aws_eks_cluster.main.version
    arn                     = aws_eks_cluster.main.arn
    oidc_issuer             = aws_eks_cluster.main.identity[0].oidc[0].issuer
  }
  sensitive = false
}
```

---

## Step 117: Sensitive Outputs

### จัดการ Sensitive Outputs

```hcl
# sensitive = true ซ่อนค่าใน terminal
output "rds_password" {
  description = "RDS master password"
  value       = random_password.db.result
  sensitive   = true
}

# Output ที่ใช้ sensitive variable ต้อง mark เป็น sensitive
output "full_connection_string" {
  description = "Full database connection string with credentials"
  value       = "postgresql://${var.db_user}:${var.db_pass}@${aws_db_instance.main.endpoint}/${var.db_name}"
  sensitive   = true
}

# private key
output "private_key_pem" {
  description = "Private key in PEM format"
  value       = tls_private_key.main.private_key_pem
  sensitive   = true
}

# JWT secret
output "jwt_secret" {
  description = "JWT signing secret"
  value       = random_password.jwt.result
  sensitive   = true
}
```

### การอ่าน Sensitive Outputs

```bash
# ค่า sensitive จะถูก mask ใน terminal
$ terraform output rds_password
<sensitive>

# ใช้ -json flag เพื่อดูค่าจริง (ระวัง! อาจถูก log)
$ terraform output -json rds_password
"P@ssw0rd123!"

# ใช้ -raw flag สำหรับ script
$ terraform output -raw rds_password
P@ssw0rd123!

# ✅ วิธีที่ปลอดภัยกว่า: เก็บใน secret manager แทน output
```

---

## Step 118: Cross-Module Outputs

### Module Output Pattern

```hcl
# modules/vpc/outputs.tf

output "vpc_id" {
  description = "ID of the VPC"
  value       = aws_vpc.main.id
}

output "public_subnet_ids" {
  description = "IDs of public subnets"
  value       = aws_subnet.public[*].id
}

output "private_subnet_ids" {
  description = "IDs of private subnets"
  value       = aws_subnet.private[*].id
}

output "vpc_cidr" {
  description = "CIDR block of the VPC"
  value       = aws_vpc.main.cidr_block
}
```

```hcl
# main.tf - root module ใช้ VPC module

module "vpc" {
  source = "./modules/vpc"

  project_name = var.project_name
  environment  = var.environment
  vpc_cidr     = var.vpc_cidr
  azs          = var.availability_zones
}

# อ้างอิง module output ด้วย module.<module_name>.<output_name>
module "ec2" {
  source = "./modules/ec2"

  vpc_id            = module.vpc.vpc_id              # ใช้ output จาก vpc module
  subnet_ids        = module.vpc.private_subnet_ids  # ใช้ output list
  allowed_cidrs     = [module.vpc.vpc_cidr]         # ใช้ output ใน expression
}

module "rds" {
  source = "./modules/rds"

  vpc_id     = module.vpc.vpc_id
  subnet_ids = module.vpc.private_subnet_ids
}

# Root outputs expose module outputs ขึ้นมา
output "vpc_id" {
  description = "VPC ID"
  value       = module.vpc.vpc_id
}

output "web_server_ips" {
  description = "Web server IPs"
  value       = module.ec2.instance_ips
}
```

### Multi-level Module Outputs

```hcl
# Root module
# modules/
# ├── network/       <- level 1 module
# │   ├── main.tf
# │   ├── variables.tf
# │   └── outputs.tf
# └── application/   <- level 1 module
#     ├── main.tf
#     ├── variables.tf
#     └── outputs.tf

# modules/network/outputs.tf
output "vpc_id" {
  value = aws_vpc.main.id
}

# modules/application/variables.tf
variable "vpc_id" {
  type = string
}

# Root main.tf
module "network" {
  source = "./modules/network"
  # ...
}

module "application" {
  source = "./modules/application"
  vpc_id = module.network.vpc_id  # cross-module reference
  # ...
}
```

---

## Step 119: Remote State Outputs

### terraform_remote_state data source

```hcl
# สถาปัตยกรรม: infrastructure แบ่งเป็น layers
# Layer 1: Network (VPC, subnets, security groups)
# Layer 2: Compute (EC2, EKS) - อ่าน network state
# Layer 3: Application (ALB, RDS) - อ่าน compute state

# network/outputs.tf
output "vpc_id" {
  value = aws_vpc.main.id
}

output "private_subnet_ids" {
  value = aws_subnet.private[*].id
}

output "public_subnet_ids" {
  value = aws_subnet.public[*].id
}
```

```hcl
# compute/main.tf - อ่านค่าจาก network layer

# Data source: อ่าน outputs จาก remote state ของ network layer
data "terraform_remote_state" "network" {
  backend = "s3"
  config = {
    bucket = "my-terraform-state"
    key    = "network/terraform.tfstate"
    region = "ap-southeast-1"
  }
}

# ใช้ remote state outputs
resource "aws_instance" "app" {
  ami           = data.aws_ami.amazon_linux.id
  instance_type = "t3.micro"
  
  # อ้างอิง outputs จาก network remote state
  subnet_id = data.terraform_remote_state.network.outputs.private_subnet_ids[0]
  vpc_security_group_ids = [
    data.terraform_remote_state.network.outputs.app_security_group_id
  ]
  
  tags = {
    Name = "app-server"
    VPC  = data.terraform_remote_state.network.outputs.vpc_id
  }
}
```

```hcl
# application/main.tf - อ่านจากหลาย layers

data "terraform_remote_state" "network" {
  backend = "s3"
  config = {
    bucket = "my-terraform-state"
    key    = "network/terraform.tfstate"
    region = "ap-southeast-1"
  }
}

data "terraform_remote_state" "compute" {
  backend = "s3"
  config = {
    bucket = "my-terraform-state"
    key    = "compute/terraform.tfstate"
    region = "ap-southeast-1"
  }
}

resource "aws_db_instance" "main" {
  engine            = "mysql"
  instance_class    = "db.t3.micro"
  
  # จาก network layer
  db_subnet_group_name   = data.terraform_remote_state.network.outputs.db_subnet_group_name
  vpc_security_group_ids = [data.terraform_remote_state.network.outputs.db_security_group_id]
  
  # ตัวอย่างการใช้ compute outputs
  # parameter_group_name = data.terraform_remote_state.compute.outputs.db_parameter_group
}
```

---

## Step 120: Terraform Output Commands

### การใช้ terraform output CLI

```bash
# แสดง outputs ทั้งหมด
terraform output

# ตัวอย่าง output:
# instance_id = "i-1234567890abcdef0"
# public_ip = "1.2.3.4"
# vpc_id = "vpc-0abc123def456"

# แสดง output เฉพาะตัว
terraform output instance_id
# i-1234567890abcdef0

# -raw flag: แสดงค่าแบบ raw text (ไม่มี quotes)
terraform output -raw instance_id
# i-1234567890abcdef0

# -json flag: แสดงทุก output เป็น JSON
terraform output -json

# ตัวอย่าง JSON output:
# {
#   "instance_id": {
#     "sensitive": false,
#     "type": "string",
#     "value": "i-1234567890abcdef0"
#   },
#   "vpc_id": {
#     "sensitive": false,
#     "type": "string",
#     "value": "vpc-0abc123def456"
#   }
# }

# -json สำหรับ output เดียว
terraform output -json vpc_id
# "vpc-0abc123def456"

# เก็บ output ในตัวแปร shell
VPC_ID=$(terraform output -raw vpc_id)
echo "VPC ID: $VPC_ID"

# ใช้กับ jq สำหรับ complex outputs
terraform output -json endpoints | jq '.web'

# ดู outputs จาก workspace อื่น
terraform workspace select prod
terraform output
```

### ใช้ Output ใน Scripts

```bash
#!/bin/bash
# deploy.sh

cd infrastructure/

# Apply terraform
terraform apply -auto-approve

# ดึง outputs
DB_HOST=$(terraform output -raw db_host)
DB_PORT=$(terraform output -raw db_port)
ALB_DNS=$(terraform output -raw alb_dns_name)
INSTANCE_IDS=$(terraform output -json instance_ids | jq -r '.[]')

echo "Database: $DB_HOST:$DB_PORT"
echo "Load Balancer: $ALB_DNS"
echo "Instances:"
echo "$INSTANCE_IDS"

# ใช้ outputs ใน deploy commands
aws ecs update-service \
  --cluster my-cluster \
  --service my-service \
  --force-new-deployment
```

---

## Complete Module Example - VPC Module

```hcl
# modules/vpc/main.tf

terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

resource "aws_vpc" "main" {
  cidr_block           = var.cidr_block
  enable_dns_support   = var.enable_dns_support
  enable_dns_hostnames = var.enable_dns_hostnames

  tags = merge(var.tags, {
    Name = "${var.name}-vpc"
  })
}

resource "aws_internet_gateway" "main" {
  vpc_id = aws_vpc.main.id

  tags = merge(var.tags, {
    Name = "${var.name}-igw"
  })
}

resource "aws_subnet" "public" {
  count = length(var.public_subnet_cidrs)

  vpc_id                  = aws_vpc.main.id
  cidr_block              = var.public_subnet_cidrs[count.index]
  availability_zone       = var.availability_zones[count.index]
  map_public_ip_on_launch = true

  tags = merge(var.tags, {
    Name = "${var.name}-public-${count.index + 1}"
    Tier = "Public"
  })
}

resource "aws_subnet" "private" {
  count = length(var.private_subnet_cidrs)

  vpc_id            = aws_vpc.main.id
  cidr_block        = var.private_subnet_cidrs[count.index]
  availability_zone = var.availability_zones[count.index]

  tags = merge(var.tags, {
    Name = "${var.name}-private-${count.index + 1}"
    Tier = "Private"
  })
}
```

```hcl
# modules/vpc/outputs.tf

output "vpc_id" {
  description = "The ID of the VPC"
  value       = aws_vpc.main.id
}

output "vpc_arn" {
  description = "The ARN of the VPC"
  value       = aws_vpc.main.arn
}

output "vpc_cidr_block" {
  description = "The CIDR block of the VPC"
  value       = aws_vpc.main.cidr_block
}

output "internet_gateway_id" {
  description = "The ID of the Internet Gateway"
  value       = aws_internet_gateway.main.id
}

output "public_subnet_ids" {
  description = "List of IDs of public subnets"
  value       = aws_subnet.public[*].id
}

output "public_subnet_cidrs" {
  description = "List of CIDR blocks of public subnets"
  value       = aws_subnet.public[*].cidr_block
}

output "private_subnet_ids" {
  description = "List of IDs of private subnets"
  value       = aws_subnet.private[*].id
}

output "private_subnet_cidrs" {
  description = "List of CIDR blocks of private subnets"
  value       = aws_subnet.private[*].cidr_block
}

output "public_subnets" {
  description = "Map of public subnet details"
  value = {
    for i, subnet in aws_subnet.public : 
      subnet.availability_zone => {
        id         = subnet.id
        cidr_block = subnet.cidr_block
        arn        = subnet.arn
      }
  }
}

output "private_subnets" {
  description = "Map of private subnet details"
  value = {
    for i, subnet in aws_subnet.private : 
      subnet.availability_zone => {
        id         = subnet.id
        cidr_block = subnet.cidr_block
        arn        = subnet.arn
      }
  }
}

# Composite output สำหรับ convenience
output "vpc_summary" {
  description = "Summary of VPC resources"
  value = {
    vpc_id             = aws_vpc.main.id
    vpc_cidr           = aws_vpc.main.cidr_block
    public_subnet_ids  = aws_subnet.public[*].id
    private_subnet_ids = aws_subnet.private[*].id
    azs_used           = distinct(concat(
      aws_subnet.public[*].availability_zone,
      aws_subnet.private[*].availability_zone
    ))
  }
}
```

---

## Output Formatting Examples

```hcl
# Format URLs
output "application_url" {
  description = "Application URL"
  value       = "https://${var.domain_name}"
}

output "api_endpoint" {
  description = "API endpoint URL"
  value       = "https://api.${var.domain_name}/v1"
}

# Format ARNs
output "ecs_service_arn" {
  description = "ECS Service ARN"
  value       = aws_ecs_service.main.id  # already an ARN
}

# Format connection strings
output "redis_url" {
  description = "Redis connection URL"
  value       = "redis://${aws_elasticache_cluster.main.cache_nodes[0].address}:${aws_elasticache_cluster.main.cache_nodes[0].port}"
}

# Conditional output
output "nat_gateway_ips" {
  description = "Elastic IPs of NAT Gateways (empty if NAT disabled)"
  value       = var.enable_nat_gateway ? aws_eip.nat[*].public_ip : []
}

# Formatted for copy-paste use
output "kubectl_config_command" {
  description = "Command to configure kubectl"
  value       = "aws eks update-kubeconfig --region ${var.region} --name ${aws_eks_cluster.main.name}"
}

output "ssh_command" {
  description = "SSH command to connect to bastion host"
  value       = "ssh -i ~/.ssh/${var.key_name}.pem ec2-user@${aws_instance.bastion.public_ip}"
}
```

---

## Debugging with Outputs

```hcl
# Temporary debug outputs (ลบออกก่อน production)
output "debug_vpc_data" {
  description = "DEBUG: Full VPC data object"
  value       = aws_vpc.main
}

output "debug_subnet_count" {
  description = "DEBUG: Number of subnets created"
  value = {
    public  = length(aws_subnet.public)
    private = length(aws_subnet.private)
    total   = length(aws_subnet.public) + length(aws_subnet.private)
  }
}

output "debug_computed_locals" {
  description = "DEBUG: Computed local values"
  value = {
    name_prefix    = local.name_prefix
    common_tags    = local.common_tags
    effective_azs  = local.availability_zones
  }
}
```

---

## สรุป (Summary)

### Output Attributes

| Attribute | Required | Description |
|-----------|----------|-------------|
| `value` | ✅ ใช่ | ค่าที่ output ส่งออก |
| `description` | แนะนำ | คำอธิบาย |
| `sensitive` | ไม่ | ซ่อนค่าใน terminal |
| `depends_on` | ไม่ | กำหนด dependency |
| `precondition` | ไม่ | ตรวจสอบเงื่อนไข |

### การอ้างอิง Outputs

```hcl
# Resource attribute
value = aws_instance.web.id

# Module output
value = module.vpc.vpc_id

# Remote state output
value = data.terraform_remote_state.network.outputs.vpc_id

# Expression
value = "https://${aws_lb.main.dns_name}"

# Collection
value = aws_subnet.public[*].id
```

### ✅ Best Practices

1. **ใส่ description เสมอ** - บอกว่า output คืออะไรและใช้ทำอะไร
2. **sensitive = true** สำหรับ passwords, keys, connection strings
3. **ใช้ precondition** ตรวจสอบ state ก่อน output
4. **สร้าง outputs.tf แยก** ไม่ปะปนกับ main.tf
5. **expose เฉพาะสิ่งจำเป็น** อย่า output ทุก attribute

### ⚠️ Common Mistakes

```hcl
# ❌ ไม่ mark sensitive output
output "db_password" {
  value = var.database_password  # จะ error! ต้อง mark sensitive
}

# ✅ ถูกต้อง
output "db_password" {
  value     = var.database_password
  sensitive = true
}

# ❌ อ้างอิง output ที่ไม่มีอยู่จริง
value = module.vpc.nonexistent_output  # error!

# ✅ ตรวจสอบ outputs ใน module ก่อน
```

### 💡 Pro Tips

1. ใช้ `terraform output -json | jq` สำหรับ process complex outputs ใน scripts
2. ตั้งชื่อ output ให้ consistent กับ input variable names ของ module ที่จะใช้
3. สร้าง "summary" output แบบ object ที่รวมหลายค่าเพื่อ convenience
4. ใช้ `-raw` flag ใน shell scripts เพื่อหลีกเลี่ยง quotes ใน string

---

*จบ Part 012 - HCL Output Values*
