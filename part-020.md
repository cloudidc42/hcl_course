# Part 020: Terraform Resources (Terraform Resources)
## Steps 191-200: การใช้งาน Resource Blocks ใน Terraform

---

## บทนำ (Introduction)

Resources คือ building blocks ที่สำคัญที่สุดใน Terraform Configuration แต่ละ resource block อธิบาย infrastructure object หนึ่งชิ้น เช่น EC2 instance, S3 bucket, VPC, หรือ Route53 record

---

## Step 191: Resource Block Syntax

### โครงสร้างพื้นฐาน

```hcl
resource "<RESOURCE_TYPE>" "<LOCAL_NAME>" {
  # Arguments (configuration)
  argument_name = argument_value
  
  # Nested blocks
  nested_block {
    nested_argument = value
  }
}
```

### ส่วนประกอบของ Resource Block

```hcl
# RESOURCE_TYPE = <provider>_<service>
# LOCAL_NAME = ชื่อที่ใช้อ้างอิงใน Terraform code

resource "aws_instance" "web_server" {
  # ^^^^^^^^^^^^^^^^^^^  ^^^^^^^^^^^
  # resource type        local name

  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t3.micro"
  
  tags = {
    Name = "web-server"
  }
}

# อ้างอิง: aws_instance.web_server.id
#           ^^^^^^^^^^^^^^^^^^^  ^^^
#           resource type.name   attribute
```

---

## Step 192: Resource Type Naming Convention

### Provider_Service Pattern

```hcl
# รูปแบบ: <provider>_<service_name>

# AWS Provider Resources
resource "aws_vpc" "main" {}              # aws + vpc
resource "aws_subnet" "public" {}        # aws + subnet
resource "aws_instance" "web" {}         # aws + instance (EC2)
resource "aws_s3_bucket" "assets" {}     # aws + s3_bucket
resource "aws_db_instance" "mysql" {}    # aws + db_instance (RDS)
resource "aws_lb" "main" {}              # aws + lb (ALB/NLB)
resource "aws_ecs_cluster" "main" {}     # aws + ecs_cluster
resource "aws_eks_cluster" "main" {}     # aws + eks_cluster
resource "aws_lambda_function" "main" {} # aws + lambda_function
resource "aws_iam_role" "app" {}         # aws + iam_role
resource "aws_route53_record" "www" {}   # aws + route53_record

# Google Cloud Provider Resources
resource "google_compute_instance" "vm" {}    # google + compute_instance
resource "google_container_cluster" "k8s" {}  # google + container_cluster
resource "google_sql_database_instance" {} {}  # google + sql_database_instance

# Azure Provider Resources
resource "azurerm_virtual_machine" "vm" {}         # azurerm + virtual_machine
resource "azurerm_kubernetes_cluster" "k8s" {}     # azurerm + kubernetes_cluster
resource "azurerm_sql_server" "db" {}              # azurerm + sql_server
```

---

## Step 193: Resource Arguments

### Required vs Optional Arguments

```hcl
# aws_instance - ตัวอย่าง required vs optional

resource "aws_instance" "example" {
  # ===== Required Arguments =====
  ami           = "ami-0c55b159cbfafe1f0"  # REQUIRED
  instance_type = "t3.micro"               # REQUIRED
  
  # ===== Optional Arguments (มี default) =====
  associate_public_ip_address = true   # default: depends on subnet
  monitoring                  = false  # default: false
  ebs_optimized               = false  # default: false
  
  # Optional - ถ้าไม่ระบุ ใช้ default VPC
  # subnet_id                = aws_subnet.public.id
  # vpc_security_group_ids   = [aws_security_group.web.id]
  
  # Optional list/set
  security_groups = []  # legacy
  
  # Optional nested blocks
  root_block_device {
    volume_type           = "gp3"   # optional, default: gp2
    volume_size           = 20      # optional, default: depends on AMI
    encrypted             = true    # optional, default: false
    delete_on_termination = true    # optional, default: true
  }
  
  tags = {
    Name = "example"
  }
}
```

### Computed Attributes

```hcl
# Computed = ค่าที่ AWS กำหนดให้ หลังจาก resource ถูกสร้าง
# ไม่สามารถ set ค่าเองได้

resource "aws_instance" "web" {
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t3.micro"
}

# หลัง apply จะมี computed attributes:
# id                = "i-1234567890abcdef0"  <- computed
# arn               = "arn:aws:ec2:..."      <- computed
# public_ip         = "1.2.3.4"             <- computed (ถ้า associate_public_ip)
# private_ip        = "10.0.1.100"          <- computed
# public_dns        = "ec2-1-2-3-4..."      <- computed
# private_dns       = "ip-10-0-1-100..."    <- computed
# availability_zone = "ap-southeast-1a"     <- computed

# ใช้ computed attribute ใน expressions
output "instance_public_ip" {
  value = aws_instance.web.public_ip  # computed attribute
}

resource "aws_eip" "web" {
  instance = aws_instance.web.id  # ใช้ computed id
}
```

---

## Step 194: Attribute Reference Syntax

### การอ้างอิง Resource Attributes

```hcl
# resource_type.local_name.attribute

# ตัวอย่างการอ้างอิง attributes
resource "aws_vpc" "main" {
  cidr_block = "10.0.0.0/16"
}

resource "aws_subnet" "public" {
  vpc_id     = aws_vpc.main.id           # อ้างอิง vpc id
  cidr_block = cidrsubnet(aws_vpc.main.cidr_block, 8, 0)  # ใช้ vpc cidr
  availability_zone = "ap-southeast-1a"
}

resource "aws_internet_gateway" "main" {
  vpc_id = aws_vpc.main.id  # อ้างอิง vpc
}

resource "aws_route_table" "public" {
  vpc_id = aws_vpc.main.id
  
  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.main.id  # อ้างอิง igw
  }
}

resource "aws_security_group" "web" {
  vpc_id = aws_vpc.main.id
  name   = "web-sg"
}

resource "aws_instance" "web" {
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t3.micro"
  subnet_id     = aws_subnet.public.id    # อ้างอิง subnet
  
  vpc_security_group_ids = [
    aws_security_group.web.id  # อ้างอิง security group
  ]
}

# อ้างอิง list resource (count)
resource "aws_subnet" "private" {
  count  = 3
  vpc_id = aws_vpc.main.id
  cidr_block = cidrsubnet("10.0.0.0/16", 8, count.index + 10)
}

# อ้างอิง specific item
resource "aws_db_subnet_group" "main" {
  subnet_ids = aws_subnet.private[*].id  # all private subnets
  # หรือ
  # subnet_ids = [
  #   aws_subnet.private[0].id,
  #   aws_subnet.private[1].id,
  # ]
}
```

---

## Step 195: Resource Dependencies

### Implicit Dependencies

```hcl
# Implicit dependency - Terraform ตรวจจับ reference อัตโนมัติ

resource "aws_vpc" "main" {
  cidr_block = "10.0.0.0/16"
}

# aws_subnet ขึ้นกับ aws_vpc.main โดย implicit
# เพราะ reference aws_vpc.main.id
resource "aws_subnet" "public" {
  vpc_id     = aws_vpc.main.id  # <- implicit dependency
  cidr_block = "10.0.1.0/24"
}

# Terraform จะ:
# 1. Create aws_vpc.main ก่อน
# 2. จากนั้น Create aws_subnet.public
```

### Explicit Dependencies (depends_on)

```hcl
# Explicit dependency - ใช้เมื่อ dependency ไม่สามารถ detect ด้วย reference

resource "aws_s3_bucket" "data" {
  bucket = "my-data-bucket"
}

resource "aws_s3_bucket_policy" "data" {
  bucket = aws_s3_bucket.data.id
  policy = data.aws_iam_policy_document.s3_policy.json
}

resource "aws_lambda_function" "processor" {
  function_name = "data-processor"
  # ...
  
  # Lambda ต้องการ bucket policy ก่อน แต่ไม่มี direct reference
  depends_on = [
    aws_s3_bucket_policy.data  # explicit dependency
  ]
}

# ใช้ depends_on บน module
module "application" {
  source = "./modules/application"
  
  depends_on = [
    module.network,   # network ต้องสร้างก่อน
    module.security,  # security groups ต้องมีก่อน
  ]
}
```

---

## Step 196: Resource Lifecycle Overview

### lifecycle Meta-argument

```hcl
resource "aws_instance" "web" {
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t3.medium"
  
  lifecycle {
    # สร้าง resource ใหม่ก่อน destroy ของเก่า
    create_before_destroy = true
    
    # ป้องกันการ destroy
    prevent_destroy = true
    
    # Ignore changes บาง attributes
    ignore_changes = [
      ami,       # ไม่ update เมื่อ AMI เปลี่ยน
      user_data, # ไม่ update เมื่อ user_data เปลี่ยน
      tags["LastUpdated"],  # ไม่ track tag บางตัว
    ]
    
    # Precondition (Terraform 1.2+)
    precondition {
      condition     = var.instance_type != "t2.micro"
      error_message = "t2.micro is deprecated. Please use t3.micro or higher."
    }
    
    # Postcondition (Terraform 1.2+)
    postcondition {
      condition     = self.public_ip != ""
      error_message = "Instance must have a public IP address."
    }
  }
}
```

### create_before_destroy Pattern

```hcl
# ใช้ create_before_destroy เมื่อ update จะ force replacement
# ป้องกัน downtime

resource "aws_launch_template" "web" {
  name_prefix   = "web-lt-"
  image_id      = var.ami_id
  instance_type = "t3.micro"
  
  lifecycle {
    create_before_destroy = true
  }
}

# Auto Scaling Group อ้างอิง launch template
resource "aws_autoscaling_group" "web" {
  name = "web-asg"
  
  launch_template {
    id      = aws_launch_template.web.id
    version = aws_launch_template.web.latest_version
  }
  
  lifecycle {
    create_before_destroy = true
  }
}
```

### prevent_destroy

```hcl
# ป้องกันการ destroy resources สำคัญ
resource "aws_db_instance" "production" {
  identifier = "prod-database"
  engine     = "mysql"
  
  lifecycle {
    prevent_destroy = true  # จะ error ถ้าพยายาม destroy
  }
}

# ถ้าต้องการ destroy จริงๆ ต้องเอา prevent_destroy ออกก่อน
# แล้ว terraform apply จากนั้น terraform destroy
```

---

## Step 197: Resource Targeting

### -target Flag

```bash
# Apply เฉพาะ resource ที่ระบุ
terraform apply -target=aws_instance.web

# Target หลาย resources
terraform apply \
  -target=aws_instance.web \
  -target=aws_security_group.web

# Target module
terraform apply -target=module.vpc

# Target resource ใน module
terraform apply -target=module.vpc.aws_subnet.public

# Destroy เฉพาะ resource
terraform destroy -target=aws_instance.old_server

# Plan เฉพาะ resource
terraform plan -target=aws_instance.web
```

### ⚠️ คำเตือนการใช้ -target

```bash
# ⚠️ -target ควรใช้เฉพาะ exceptional circumstances
# เหตุผล:
# - Terraform อาจไม่รู้ว่ามี dependency ที่ต้องอัพเดทด้วย
# - State อาจ inconsistent
# - ใช้เป็น quick fix เท่านั้น ไม่ใช่ workflow ปกติ

# ✅ ใช้เมื่อ:
# - ต้องการ destroy resource เดียว
# - Debugging
# - Emergency patch

# ❌ ไม่ควรใช้เมื่อ:
# - Regular deployments
# - มี dependencies ระหว่าง resources
```

---

## Step 198: Resource Taint and Replace

### terraform taint (deprecated ใน Terraform 1.0)

```bash
# terraform taint (deprecated)
terraform taint aws_instance.web

# Terraform 1.0+ ใช้ replace แทน
terraform apply -replace=aws_instance.web

# Force replacement สำหรับ specific resource
terraform plan -replace=aws_instance.web
terraform apply -replace=aws_instance.web

# Replace resource ใน module
terraform apply -replace=module.vpc.aws_nat_gateway.main[0]
```

### เมื่อไหรต้อง Replace?

```bash
# ใช้ replace เมื่อ resource มีปัญหาและต้องการสร้างใหม่
# เช่น:
# - EC2 instance มีปัญหา
# - Certificate หมดอายุ
# - Security group ถูก modified นอก Terraform

# ตัวอย่าง
terraform apply -replace=aws_instance.web_2
# Terraform จะ:
# 1. Create aws_instance.web_2_new
# 2. Destroy aws_instance.web_2 (ถ้า create_before_destroy = true)
# หรือ
# 1. Destroy aws_instance.web_2
# 2. Create aws_instance.web_2_new (default)
```

---

## Step 199: Resource Import

### import Block (Terraform 1.5+)

```hcl
# นำ existing resource เข้า Terraform management
# import block ใน code (Terraform 1.5+)

import {
  to = aws_instance.web
  id = "i-1234567890abcdef0"
}

import {
  to = aws_s3_bucket.existing
  id = "my-existing-bucket-name"
}

import {
  to = aws_vpc.main
  id = "vpc-0abc123def456"
}

# Import VPC subnet
import {
  to = aws_subnet.public[0]
  id = "subnet-0abc123"
}

# Import IAM role
import {
  to = aws_iam_role.app
  id = "my-app-role"
}
```

```bash
# Plan import
terraform plan

# Apply import
terraform apply

# Terraform 1.4 และก่อนหน้า ใช้ CLI command
terraform import aws_instance.web i-1234567890abcdef0
terraform import aws_s3_bucket.existing my-bucket-name
terraform import aws_vpc.main vpc-0abc123def456
```

### Import ID Formats

```bash
# ตัวอย่าง Import IDs สำหรับ AWS resources

# EC2 Instance
terraform import aws_instance.web i-1234567890abcdef0

# VPC
terraform import aws_vpc.main vpc-0123456789abcdef0

# Subnet
terraform import aws_subnet.public subnet-0abc123def456

# Security Group
terraform import aws_security_group.web sg-0abc123def456

# S3 Bucket
terraform import aws_s3_bucket.main my-bucket-name

# RDS Instance
terraform import aws_db_instance.main my-database-identifier

# IAM Role
terraform import aws_iam_role.app my-role-name

# IAM Policy
terraform import aws_iam_policy.main arn:aws:iam::123456789012:policy/my-policy

# Route53 Record
terraform import aws_route53_record.www Z4KAPRWWNC7JR_www_A

# ECS Service
terraform import aws_ecs_service.main cluster-name/service-name

# Lambda Function
terraform import aws_lambda_function.main my-function-name
```

---

## Step 200: Complete Resource Examples

### aws_instance - Full Example

```hcl
# Data source สำหรับ AMI
data "aws_ami" "amazon_linux_2" {
  most_recent = true
  owners      = ["amazon"]

  filter {
    name   = "name"
    values = ["amzn2-ami-hvm-*-x86_64-gp2"]
  }

  filter {
    name   = "virtualization-type"
    values = ["hvm"]
  }
}

# EC2 Instance สมบูรณ์
resource "aws_instance" "web" {
  # Required
  ami           = data.aws_ami.amazon_linux_2.id
  instance_type = var.instance_type

  # Network
  subnet_id                   = aws_subnet.public.id
  vpc_security_group_ids      = [aws_security_group.web.id]
  associate_public_ip_address = true
  private_ip                  = null  # auto-assign

  # Key pair for SSH
  key_name = var.key_pair_name

  # IAM Role
  iam_instance_profile = aws_iam_instance_profile.web.name

  # Storage
  root_block_device {
    volume_type           = "gp3"
    volume_size           = 30
    iops                  = 3000
    throughput            = 125
    encrypted             = true
    kms_key_id            = aws_kms_key.ebs.arn
    delete_on_termination = true
  }

  # Additional EBS volume
  ebs_block_device {
    device_name           = "/dev/xvdf"
    volume_type           = "gp3"
    volume_size           = 100
    encrypted             = true
    delete_on_termination = true
  }

  # User data script (bootstrap)
  user_data = base64encode(templatefile("${path.module}/scripts/user-data.sh", {
    environment = var.environment
    app_name    = var.app_name
  }))

  # Monitoring
  monitoring = var.enable_detailed_monitoring

  # Placement
  availability_zone = var.availability_zone

  # Metadata
  metadata_options {
    http_endpoint               = "enabled"
    http_tokens                 = "required"  # IMDSv2 required
    http_put_response_hop_limit = 1
  }

  # Termination protection
  disable_api_termination = var.environment == "prod" ? true : false

  lifecycle {
    create_before_destroy = true
    ignore_changes = [
      ami,  # Allow AMI updates outside Terraform
      user_data,
    ]
  }

  tags = merge(local.compute_tags, {
    Name    = "${local.name_prefix}-web"
    Service = "frontend"
  })
}
```

### aws_s3_bucket - Full Example

```hcl
# S3 Bucket สมบูรณ์
resource "aws_s3_bucket" "assets" {
  bucket        = "${var.project_name}-${var.environment}-assets"
  force_destroy = var.environment != "prod"  # ป้องกันลบ prod bucket

  tags = merge(local.common_tags, {
    Name    = "${local.name_prefix}-assets"
    Purpose = "Static assets"
  })
}

# Versioning
resource "aws_s3_bucket_versioning" "assets" {
  bucket = aws_s3_bucket.assets.id

  versioning_configuration {
    status     = "Enabled"
    mfa_delete = "Disabled"
  }
}

# Encryption
resource "aws_s3_bucket_server_side_encryption_configuration" "assets" {
  bucket = aws_s3_bucket.assets.id

  rule {
    apply_server_side_encryption_by_default {
      sse_algorithm     = "aws:kms"
      kms_master_key_id = aws_kms_key.s3.arn
    }
    bucket_key_enabled = true
  }
}

# Block public access
resource "aws_s3_bucket_public_access_block" "assets" {
  bucket = aws_s3_bucket.assets.id

  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}

# Lifecycle rules
resource "aws_s3_bucket_lifecycle_configuration" "assets" {
  bucket = aws_s3_bucket.assets.id

  rule {
    id     = "transition-to-ia"
    status = "Enabled"

    filter {
      prefix = "uploads/"
    }

    transition {
      days          = 30
      storage_class = "STANDARD_IA"
    }

    transition {
      days          = 90
      storage_class = "GLACIER"
    }

    expiration {
      days = 365
    }

    noncurrent_version_expiration {
      noncurrent_days = 30
    }
  }
}

# CORS สำหรับ web application
resource "aws_s3_bucket_cors_configuration" "assets" {
  bucket = aws_s3_bucket.assets.id

  cors_rule {
    allowed_headers = ["*"]
    allowed_methods = ["GET", "PUT", "POST"]
    allowed_origins = ["https://${var.domain_name}"]
    expose_headers  = ["ETag"]
    max_age_seconds = 3000
  }
}

# Bucket policy
resource "aws_s3_bucket_policy" "assets" {
  bucket = aws_s3_bucket.assets.id
  policy = data.aws_iam_policy_document.assets_bucket.json
}

data "aws_iam_policy_document" "assets_bucket" {
  statement {
    sid    = "AllowCloudFrontAccess"
    effect = "Allow"

    principals {
      type        = "Service"
      identifiers = ["cloudfront.amazonaws.com"]
    }

    actions = ["s3:GetObject"]

    resources = ["${aws_s3_bucket.assets.arn}/*"]

    condition {
      test     = "StringEquals"
      variable = "aws:SourceArn"
      values   = [aws_cloudfront_distribution.main.arn]
    }
  }
}
```

### aws_vpc - Full Example

```hcl
# VPC
resource "aws_vpc" "main" {
  cidr_block           = var.vpc_cidr
  enable_dns_support   = true
  enable_dns_hostnames = true
  instance_tenancy     = "default"

  tags = merge(local.network_tags, {
    Name = "${local.name_prefix}-vpc"
  })
}

# Internet Gateway
resource "aws_internet_gateway" "main" {
  vpc_id = aws_vpc.main.id

  tags = merge(local.network_tags, {
    Name = "${local.name_prefix}-igw"
  })
}

# Public Subnets
resource "aws_subnet" "public" {
  count = length(var.availability_zones)

  vpc_id                  = aws_vpc.main.id
  cidr_block              = cidrsubnet(var.vpc_cidr, 8, count.index)
  availability_zone       = var.availability_zones[count.index]
  map_public_ip_on_launch = true

  tags = merge(local.network_tags, {
    Name = "${local.name_prefix}-public-${count.index + 1}"
    Tier = "Public"
    "kubernetes.io/role/elb" = "1"  # สำหรับ EKS (ถ้าใช้)
  })
}

# Private Subnets
resource "aws_subnet" "private" {
  count = length(var.availability_zones)

  vpc_id            = aws_vpc.main.id
  cidr_block        = cidrsubnet(var.vpc_cidr, 8, count.index + 10)
  availability_zone = var.availability_zones[count.index]

  tags = merge(local.network_tags, {
    Name = "${local.name_prefix}-private-${count.index + 1}"
    Tier = "Private"
    "kubernetes.io/role/internal-elb" = "1"  # สำหรับ EKS
  })
}

# NAT Gateways (production: one per AZ)
resource "aws_eip" "nat" {
  count  = var.enable_nat_gateway ? length(var.availability_zones) : 0
  domain = "vpc"

  tags = merge(local.network_tags, {
    Name = "${local.name_prefix}-nat-eip-${count.index + 1}"
  })

  depends_on = [aws_internet_gateway.main]
}

resource "aws_nat_gateway" "main" {
  count = var.enable_nat_gateway ? length(var.availability_zones) : 0

  allocation_id = aws_eip.nat[count.index].id
  subnet_id     = aws_subnet.public[count.index].id

  tags = merge(local.network_tags, {
    Name = "${local.name_prefix}-nat-${count.index + 1}"
  })

  depends_on = [aws_internet_gateway.main]
}

# Route Tables
resource "aws_route_table" "public" {
  vpc_id = aws_vpc.main.id

  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.main.id
  }

  tags = merge(local.network_tags, {
    Name = "${local.name_prefix}-public-rt"
  })
}

resource "aws_route_table" "private" {
  count  = var.enable_nat_gateway ? length(var.availability_zones) : 1
  vpc_id = aws_vpc.main.id

  dynamic "route" {
    for_each = var.enable_nat_gateway ? [1] : []
    content {
      cidr_block     = "0.0.0.0/0"
      nat_gateway_id = aws_nat_gateway.main[count.index].id
    }
  }

  tags = merge(local.network_tags, {
    Name = "${local.name_prefix}-private-rt-${count.index + 1}"
  })
}

# Route Table Associations
resource "aws_route_table_association" "public" {
  count = length(aws_subnet.public)

  subnet_id      = aws_subnet.public[count.index].id
  route_table_id = aws_route_table.public.id
}

resource "aws_route_table_association" "private" {
  count = length(aws_subnet.private)

  subnet_id      = aws_subnet.private[count.index].id
  route_table_id = var.enable_nat_gateway ? aws_route_table.private[count.index].id : aws_route_table.private[0].id
}
```

### Null Resource และ terraform_data

```hcl
# terraform_data (Terraform 1.4+) - แทน null_resource
resource "terraform_data" "example" {
  # Triggers เมื่อ value เปลี่ยน
  triggers_replace = {
    script_hash = filemd5("${path.module}/scripts/setup.sh")
    ami_id      = var.ami_id
  }
  
  # Run script หลัง create
  provisioner "local-exec" {
    command = "echo 'Resource created: ${var.project_name}'"
  }
}

# null_resource (ยังใช้ได้ แต่ terraform_data แนะนำกว่า)
resource "null_resource" "setup" {
  triggers = {
    always_run = timestamp()  # run ทุก apply
  }
  
  provisioner "local-exec" {
    command = <<-EOT
      aws ssm send-command \
        --instance-ids ${aws_instance.web.id} \
        --document-name "AWS-RunShellScript" \
        --parameters 'commands=["sudo systemctl restart nginx"]'
    EOT
  }
  
  depends_on = [aws_instance.web]
}
```

---

## สรุป (Summary)

### Resource Block Structure สรุป

```hcl
resource "<TYPE>" "<NAME>" {
  # Required arguments
  required_arg = value
  
  # Optional arguments
  optional_arg = value
  
  # Nested block
  nested_block {
    nested_arg = value
  }
  
  # Meta-arguments
  count       = <number>
  for_each    = <map or set>
  provider    = <provider alias>
  depends_on  = [<dependencies>]
  
  lifecycle {
    create_before_destroy = bool
    prevent_destroy       = bool
    ignore_changes        = [attr, ...]
    precondition { ... }
    postcondition { ... }
  }
}
```

### Attribute Types

| Type | Description | Example |
|------|-------------|---------|
| Argument | ค่าที่เรา set | `instance_type = "t3.micro"` |
| Computed | ค่าที่ AWS set | `id = "i-123..."` |
| Reference | อ้างอิง resource อื่น | `vpc_id = aws_vpc.main.id` |

### ✅ Best Practices

1. **Implicit dependencies** ก่อน explicit (depends_on)
2. **lifecycle.create_before_destroy** สำหรับ zero-downtime
3. **lifecycle.prevent_destroy** สำหรับ critical data
4. **lifecycle.ignore_changes** สำหรับ external changes
5. **-target ใช้น้อยที่สุด** เฉพาะ emergencies
6. **import block** (TF 1.5+) สำหรับ existing resources

### ⚠️ Common Mistakes

```hcl
# ❌ Circular dependency
resource "aws_security_group" "a" {
  ingress {
    security_groups = [aws_security_group.b.id]  # อ้างอิง b
  }
}

resource "aws_security_group" "b" {
  ingress {
    security_groups = [aws_security_group.a.id]  # อ้างอิง a -> circular!
  }
}

# ✅ แก้ด้วย aws_security_group_rule แยก
resource "aws_security_group" "a" {}
resource "aws_security_group" "b" {}

resource "aws_security_group_rule" "a_from_b" {
  type                     = "ingress"
  security_group_id        = aws_security_group.a.id
  source_security_group_id = aws_security_group.b.id
}
```

### 💡 Pro Tips

1. ใช้ `terraform state show <resource>` ดู computed attributes
2. `terraform console` ทดสอบ expressions
3. `terraform graph | dot -Tsvg > graph.svg` visualize dependencies
4. ใช้ `moved` block สำหรับ rename resources โดยไม่ destroy

---

*จบ Part 020 - Terraform Resources*
