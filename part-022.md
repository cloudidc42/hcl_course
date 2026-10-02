# Part 22: Terraform Data Sources (ขั้นตอนที่ 211-220)

## ภาพรวม (Overview)

Data Sources ใน Terraform ช่วยให้เราดึงข้อมูลจาก provider หรือ infrastructure ที่มีอยู่แล้ว โดยไม่ต้องสร้างใหม่ ต่างจาก resource ที่ create/update/delete data source แค่ **อ่านข้อมูล** เท่านั้น

**Read vs Create**:
- `resource` block → Terraform manages lifecycle (create, update, destroy)
- `data` block → Terraform only reads (query existing infrastructure)

---

## Step 211: Data Block Syntax

### Syntax พื้นฐาน

```hcl
data "<TYPE>" "<NAME>" {
  # filter/query arguments
}

# ใช้ข้อมูลที่ได้:
# data.<TYPE>.<NAME>.<ATTRIBUTE>
```

### ตัวอย่างเบื้องต้น

```hcl
# อ่านข้อมูล AMI ล่าสุด
data "aws_ami" "ubuntu" {
  most_recent = true
  owners      = ["099720109477"]  # Canonical

  filter {
    name   = "name"
    values = ["ubuntu/images/hvm-ssd/ubuntu-*-22.04-amd64-server-*"]
  }
  
  filter {
    name   = "virtualization-type"
    values = ["hvm"]
  }
}

# ใช้ AMI ใน resource
resource "aws_instance" "web" {
  ami           = data.aws_ami.ubuntu.id  # ใช้ data source
  instance_type = "t3.micro"
}
```

---

## Step 212: aws_ami - Finding Latest AMI

### Amazon Linux 2023

```hcl
data "aws_ami" "amazon_linux_2023" {
  most_recent = true
  owners      = ["amazon"]

  filter {
    name   = "name"
    values = ["al2023-ami-*-x86_64"]
  }

  filter {
    name   = "architecture"
    values = ["x86_64"]
  }

  filter {
    name   = "virtualization-type"
    values = ["hvm"]
  }
}

output "al2023_ami_id" {
  value = data.aws_ami.amazon_linux_2023.id
}

output "al2023_ami_name" {
  value = data.aws_ami.amazon_linux_2023.name
}
```

### Ubuntu 22.04 LTS

```hcl
data "aws_ami" "ubuntu_22_04" {
  most_recent = true
  owners      = ["099720109477"]  # Canonical

  filter {
    name   = "name"
    values = ["ubuntu/images/hvm-ssd/ubuntu-jammy-22.04-amd64-server-*"]
  }

  filter {
    name   = "state"
    values = ["available"]
  }
}
```

### Windows Server 2022

```hcl
data "aws_ami" "windows_2022" {
  most_recent = true
  owners      = ["amazon"]

  filter {
    name   = "name"
    values = ["Windows_Server-2022-English-Full-Base-*"]
  }

  filter {
    name   = "platform"
    values = ["windows"]
  }
}
```

### Custom/Organization AMI

```hcl
data "aws_ami" "app_base" {
  most_recent = true
  owners      = [var.aws_account_id]  # องค์กรเอง

  filter {
    name   = "name"
    values = ["my-app-base-*"]
  }

  filter {
    name   = "tag:Environment"
    values = ["production"]
  }

  filter {
    name   = "tag:Team"
    values = ["platform"]
  }
}
```

### หา AMI เฉพาะ version

```hcl
data "aws_ami" "specific_version" {
  owners = ["amazon"]

  filter {
    name   = "name"
    values = ["amzn2-ami-hvm-2.0.20230320.0-x86_64-gp2"]
  }
}

# หรือระบุ AMI ID ตรงๆ (แต่ไม่ flexible)
data "aws_ami" "by_id" {
  owners = ["amazon"]
  
  filter {
    name   = "image-id"
    values = ["ami-0c55b159cbfafe1f0"]
  }
}
```

### ใช้ AMI data source กับ multiple regions

```hcl
provider "aws" {
  alias  = "us_east_1"
  region = "us-east-1"
}

provider "aws" {
  alias  = "ap_southeast_1"
  region = "ap-southeast-1"
}

data "aws_ami" "ubuntu_us" {
  provider    = aws.us_east_1
  most_recent = true
  owners      = ["099720109477"]

  filter {
    name   = "name"
    values = ["ubuntu/images/hvm-ssd/ubuntu-jammy-22.04-amd64-server-*"]
  }
}

data "aws_ami" "ubuntu_sg" {
  provider    = aws.ap_southeast_1
  most_recent = true
  owners      = ["099720109477"]

  filter {
    name   = "name"
    values = ["ubuntu/images/hvm-ssd/ubuntu-jammy-22.04-amd64-server-*"]
  }
}
```

---

## Step 213: aws_availability_zones

```hcl
# ดึงรายการ AZ ที่ available ใน region ปัจจุบัน
data "aws_availability_zones" "available" {
  state = "available"
}

output "azs" {
  value = data.aws_availability_zones.available.names
  # ["us-east-1a", "us-east-1b", "us-east-1c", "us-east-1d", "us-east-1f"]
}

# ใช้กับ subnets
resource "aws_subnet" "public" {
  count             = 3
  vpc_id            = aws_vpc.main.id
  cidr_block        = "10.0.${count.index}.0/24"
  availability_zone = data.aws_availability_zones.available.names[count.index]

  map_public_ip_on_launch = true

  tags = {
    Name = "public-subnet-${count.index + 1}"
  }
}

# ใช้กับ for_each
locals {
  azs = slice(data.aws_availability_zones.available.names, 0, 3)  # เอาแค่ 3 AZ แรก
  
  subnet_map = {
    for i, az in local.azs :
    "subnet-${i + 1}" => {
      az   = az
      cidr = "10.0.${i}.0/24"
    }
  }
}

resource "aws_subnet" "private" {
  for_each          = local.subnet_map
  vpc_id            = aws_vpc.main.id
  cidr_block        = each.value.cidr
  availability_zone = each.value.az

  tags = { Name = each.key }
}

# exclude specific AZ
data "aws_availability_zones" "filtered" {
  state = "available"
  
  filter {
    name   = "opt-in-status"
    values = ["opt-in-not-required"]
  }
  
  # Exclude AZs ที่มีปัญหา
  exclude_names = ["us-east-1e"]
}
```

---

## Step 214: aws_caller_identity

```hcl
# ดึงข้อมูล AWS account ปัจจุบัน
data "aws_caller_identity" "current" {}

output "account_id" {
  value = data.aws_caller_identity.current.account_id
}

output "caller_arn" {
  value = data.aws_caller_identity.current.arn
}

output "caller_user_id" {
  value = data.aws_caller_identity.current.user_id
}

# ใช้ใน policy
resource "aws_s3_bucket_policy" "secure" {
  bucket = aws_s3_bucket.main.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Sid    = "AllowCurrentAccount"
        Effect = "Allow"
        Principal = {
          AWS = "arn:aws:iam::${data.aws_caller_identity.current.account_id}:root"
        }
        Action   = "s3:*"
        Resource = [
          aws_s3_bucket.main.arn,
          "${aws_s3_bucket.main.arn}/*"
        ]
      }
    ]
  })
}

# ป้องกัน deploy ไปผิด account
variable "expected_account_id" {
  description = "Expected AWS Account ID"
  type        = string
}

resource "null_resource" "account_check" {
  lifecycle {
    precondition {
      condition     = data.aws_caller_identity.current.account_id == var.expected_account_id
      error_message = "Wrong AWS account! Expected ${var.expected_account_id} but got ${data.aws_caller_identity.current.account_id}"
    }
  }
}
```

---

## Step 215: aws_region

```hcl
# ดึงข้อมูล region ปัจจุบัน
data "aws_region" "current" {}

output "region_name" {
  value = data.aws_region.current.name
  # "us-east-1"
}

output "region_description" {
  value = data.aws_region.current.description
  # "US East (N. Virginia)"
}

# ใช้ใน resource names
resource "aws_cloudwatch_log_group" "app" {
  name = "/app/${data.aws_region.current.name}/logs"
}

# ใช้สร้าง ARN ด้วยตัวเอง
locals {
  lambda_arn_prefix = "arn:aws:lambda:${data.aws_region.current.name}:${data.aws_caller_identity.current.account_id}"
}

# ใช้กับ provider alias
data "aws_region" "dr" {
  provider = aws.dr_region
}

output "dr_region" {
  value = data.aws_region.dr.name
}
```

---

## Step 216: aws_vpc & aws_subnet - Finding Existing

### หา VPC ที่มีอยู่

```hcl
# หา Default VPC
data "aws_vpc" "default" {
  default = true
}

# หา VPC ด้วย ID
data "aws_vpc" "production" {
  id = "vpc-12345678"
}

# หา VPC ด้วย tag
data "aws_vpc" "main" {
  tags = {
    Name        = "production-vpc"
    Environment = "prod"
  }
}

# หา VPC ด้วย filter
data "aws_vpc" "by_cidr" {
  filter {
    name   = "cidr"
    values = ["10.0.0.0/16"]
  }
}

output "vpc_id" {
  value = data.aws_vpc.main.id
}

output "vpc_cidr" {
  value = data.aws_vpc.main.cidr_block
}
```

### หา Subnets

```hcl
# หา subnet เดียวด้วย ID
data "aws_subnet" "specific" {
  id = "subnet-12345678"
}

# หา subnet ด้วย tag
data "aws_subnet" "web" {
  vpc_id = data.aws_vpc.main.id
  
  tags = {
    Name = "web-subnet-1"
    Tier = "public"
  }
}

# หา subnets หลายอัน (plural)
data "aws_subnets" "private" {
  filter {
    name   = "vpc-id"
    values = [data.aws_vpc.main.id]
  }

  tags = {
    Type = "private"
  }
}

output "private_subnet_ids" {
  value = data.aws_subnets.private.ids
  # ["subnet-xxx", "subnet-yyy", "subnet-zzz"]
}

# หา public subnets
data "aws_subnets" "public" {
  filter {
    name   = "vpc-id"
    values = [data.aws_vpc.main.id]
  }

  filter {
    name   = "map-public-ip-on-launch"
    values = ["true"]
  }
}

# ใช้ subnet IDs ที่หาได้
resource "aws_instance" "web" {
  ami           = data.aws_ami.ubuntu.id
  instance_type = "t3.micro"
  
  # ใช้ subnet แรกจากรายการ
  subnet_id = data.aws_subnets.public.ids[0]
}

resource "aws_lb" "main" {
  name               = "main-alb"
  internal           = false
  load_balancer_type = "application"
  security_groups    = [aws_security_group.alb.id]
  
  # ใช้ subnets ทั้งหมด
  subnets = data.aws_subnets.public.ids
}
```

### หา Multiple Subnets ด้วย for_each

```hcl
data "aws_subnets" "all_private" {
  filter {
    name   = "vpc-id"
    values = [var.vpc_id]
  }
  
  tags = { Tier = "private" }
}

data "aws_subnet" "private" {
  for_each = toset(data.aws_subnets.all_private.ids)
  id       = each.value
}

output "private_subnets_by_az" {
  value = {
    for id, subnet in data.aws_subnet.private :
    subnet.availability_zone => subnet.id
  }
}
```

---

## Step 217: aws_iam_policy_document

```hcl
# สร้าง IAM policy document แบบ structured
data "aws_iam_policy_document" "s3_access" {
  statement {
    sid    = "AllowS3ListBucket"
    effect = "Allow"
    
    actions = [
      "s3:ListBucket",
      "s3:GetBucketLocation"
    ]
    
    resources = [
      "arn:aws:s3:::${var.bucket_name}"
    ]
    
    condition {
      test     = "StringLike"
      variable = "s3:prefix"
      values   = ["${var.app_name}/*"]
    }
  }

  statement {
    sid    = "AllowS3Objects"
    effect = "Allow"
    
    actions = [
      "s3:GetObject",
      "s3:PutObject",
      "s3:DeleteObject"
    ]
    
    resources = [
      "arn:aws:s3:::${var.bucket_name}/${var.app_name}/*"
    ]
  }
}

# ใช้ใน IAM policy
resource "aws_iam_policy" "s3_access" {
  name        = "s3-access-policy"
  description = "Allow access to S3 bucket"
  policy      = data.aws_iam_policy_document.s3_access.json
}

# Assume Role Policy
data "aws_iam_policy_document" "ec2_assume_role" {
  statement {
    actions = ["sts:AssumeRole"]
    
    principals {
      type        = "Service"
      identifiers = ["ec2.amazonaws.com"]
    }
  }
}

resource "aws_iam_role" "ec2" {
  name               = "ec2-role"
  assume_role_policy = data.aws_iam_policy_document.ec2_assume_role.json
}

# Cross-account assume role
data "aws_iam_policy_document" "cross_account" {
  statement {
    actions = ["sts:AssumeRole"]
    
    principals {
      type        = "AWS"
      identifiers = [
        "arn:aws:iam::${var.trusted_account_id}:root",
        "arn:aws:iam::${var.trusted_account_id}:role/DeployRole"
      ]
    }
    
    condition {
      test     = "Bool"
      variable = "aws:MultiFactorAuthPresent"
      values   = ["true"]
    }
  }
}
```

### Policy Document สำหรับ KMS

```hcl
data "aws_iam_policy_document" "kms_key_policy" {
  statement {
    sid    = "Enable IAM User Permissions"
    effect = "Allow"
    
    principals {
      type        = "AWS"
      identifiers = ["arn:aws:iam::${data.aws_caller_identity.current.account_id}:root"]
    }
    
    actions   = ["kms:*"]
    resources = ["*"]
  }

  statement {
    sid    = "Allow CloudWatch Logs"
    effect = "Allow"
    
    principals {
      type        = "Service"
      identifiers = ["logs.${data.aws_region.current.name}.amazonaws.com"]
    }
    
    actions = [
      "kms:Encrypt*",
      "kms:Decrypt*",
      "kms:ReEncrypt*",
      "kms:GenerateDataKey*",
      "kms:Describe*"
    ]
    
    resources = ["*"]
  }
}

resource "aws_kms_key" "cloudwatch" {
  description             = "KMS key for CloudWatch Logs"
  deletion_window_in_days = 7
  enable_key_rotation     = true
  policy                  = data.aws_iam_policy_document.kms_key_policy.json
}
```

---

## Step 218: aws_s3_bucket, aws_route53_zone, aws_acm_certificate

### aws_s3_bucket (existing)

```hcl
# อ่านข้อมูล S3 bucket ที่มีอยู่แล้ว
data "aws_s3_bucket" "existing" {
  bucket = "my-existing-bucket"
}

output "bucket_arn" {
  value = data.aws_s3_bucket.existing.arn
}

output "bucket_domain" {
  value = data.aws_s3_bucket.existing.bucket_domain_name
}

output "bucket_regional_domain" {
  value = data.aws_s3_bucket.existing.bucket_regional_domain_name
}

# ใช้ reference existing bucket ใน CloudFront
resource "aws_cloudfront_distribution" "main" {
  origin {
    domain_name = data.aws_s3_bucket.existing.bucket_regional_domain_name
    origin_id   = "S3-${data.aws_s3_bucket.existing.bucket}"
    
    s3_origin_config {
      origin_access_identity = aws_cloudfront_origin_access_identity.main.cloudfront_access_identity_path
    }
  }

  enabled = true
  
  default_cache_behavior {
    viewer_protocol_policy = "redirect-to-https"
    target_origin_id       = "S3-${data.aws_s3_bucket.existing.bucket}"
    
    forwarded_values {
      query_string = false
      cookies { forward = "none" }
    }
    
    allowed_methods = ["GET", "HEAD"]
    cached_methods  = ["GET", "HEAD"]
  }
  
  restrictions {
    geo_restriction { restriction_type = "none" }
  }
  
  viewer_certificate {
    acm_certificate_arn = data.aws_acm_certificate.main.arn
    ssl_support_method  = "sni-only"
  }
}
```

### aws_route53_zone

```hcl
# หา Route53 Zone ด้วยชื่อ domain
data "aws_route53_zone" "main" {
  name         = "example.com."  # Note: trailing dot
  private_zone = false
}

output "zone_id" {
  value = data.aws_route53_zone.main.zone_id
}

# หา Private Hosted Zone
data "aws_route53_zone" "internal" {
  name         = "internal.example.com."
  private_zone = true
  vpc_id       = aws_vpc.main.id
}

# สร้าง DNS record ใน zone ที่มีอยู่
resource "aws_route53_record" "app" {
  zone_id = data.aws_route53_zone.main.zone_id
  name    = "app.${data.aws_route53_zone.main.name}"
  type    = "A"

  alias {
    name                   = aws_lb.main.dns_name
    zone_id                = aws_lb.main.zone_id
    evaluate_target_health = true
  }
}

# Certificate validation record
resource "aws_route53_record" "cert_validation" {
  for_each = {
    for dvo in aws_acm_certificate.main.domain_validation_options : dvo.domain_name => {
      name   = dvo.resource_record_name
      record = dvo.resource_record_value
      type   = dvo.resource_record_type
    }
  }

  zone_id = data.aws_route53_zone.main.zone_id
  name    = each.value.name
  type    = each.value.type
  records = [each.value.record]
  ttl     = 60
}
```

### aws_acm_certificate

```hcl
# หา Certificate ที่ issued แล้ว
data "aws_acm_certificate" "main" {
  domain   = "*.example.com"
  statuses = ["ISSUED"]
}

output "certificate_arn" {
  value = data.aws_acm_certificate.main.arn
}

# หา Certificate ใน region เฉพาะ (CloudFront ต้องการ us-east-1)
data "aws_acm_certificate" "cloudfront" {
  provider = aws.us_east_1
  domain   = "*.example.com"
  statuses = ["ISSUED"]
  
  most_recent = true  # ถ้ามีหลายอัน เอาล่าสุด
}

# หา Certificate พร้อม key type
data "aws_acm_certificate" "rsa" {
  domain      = "example.com"
  types       = ["AMAZON_ISSUED"]
  statuses    = ["ISSUED"]
  key_types   = ["RSA_2048"]
  most_recent = true
}

# ใช้ใน ALB
resource "aws_lb_listener" "https" {
  load_balancer_arn = aws_lb.main.arn
  port              = "443"
  protocol          = "HTTPS"
  ssl_policy        = "ELBSecurityPolicy-TLS13-1-2-2021-06"
  certificate_arn   = data.aws_acm_certificate.main.arn

  default_action {
    type             = "forward"
    target_group_arn = aws_lb_target_group.app.arn
  }
}
```

---

## Step 219: terraform_remote_state Data Source

```hcl
# อ่าน outputs จาก Terraform state อื่น
# ใช้เมื่อ infrastructure แบ่งเป็นหลาย state files

# networking/outputs.tf
output "vpc_id" {
  value = aws_vpc.main.id
}

output "private_subnet_ids" {
  value = aws_subnet.private[*].id
}

output "public_subnet_ids" {
  value = aws_subnet.public[*].id
}

# ---

# application/main.tf
data "terraform_remote_state" "networking" {
  backend = "s3"
  
  config = {
    bucket = "my-terraform-state"
    key    = "networking/terraform.tfstate"
    region = "us-east-1"
  }
}

# ใช้ output จาก networking state
resource "aws_instance" "app" {
  ami           = data.aws_ami.ubuntu.id
  instance_type = "t3.medium"
  
  # ใช้ VPC และ subnet จาก networking state
  subnet_id = data.terraform_remote_state.networking.outputs.private_subnet_ids[0]
  
  vpc_security_group_ids = [aws_security_group.app.id]
  
  tags = { Name = "app-server" }
}

resource "aws_security_group" "app" {
  vpc_id = data.terraform_remote_state.networking.outputs.vpc_id
  name   = "app-sg"
  
  # ...
}

# ใช้กับ Terraform Cloud/Enterprise backend
data "terraform_remote_state" "shared" {
  backend = "remote"
  
  config = {
    organization = "my-org"
    workspaces = {
      name = "shared-networking"
    }
  }
}

# ใช้กับ local state (สำหรับ testing)
data "terraform_remote_state" "local_test" {
  backend = "local"
  
  config = {
    path = "../networking/terraform.tfstate"
  }
}
```

### ⚠️ ข้อควรระวัง terraform_remote_state

```hcl
# ❌ ปัญหา: tight coupling ระหว่าง state files
# ถ้า networking state เปลี่ยน outputs อาจกระทบ application state

# ✅ ทางเลือกที่ดีกว่า: ใช้ data sources โดยตรง
data "aws_vpc" "main" {
  tags = { Name = "production-vpc" }
}

# ✅ หรือใช้ SSM Parameter Store เป็น intermediary
data "aws_ssm_parameter" "vpc_id" {
  name = "/networking/vpc_id"
}

data "aws_ssm_parameter" "private_subnet_ids" {
  name = "/networking/private_subnet_ids"
}
```

---

## Step 220: http Data Source & Data Source Dependencies

### http data source

```hcl
# ต้อง configure provider ก่อน
terraform {
  required_providers {
    http = {
      source  = "hashicorp/http"
      version = "~> 3.0"
    }
  }
}

provider "http" {}

# ดึงข้อมูลจาก URL
data "http" "my_ip" {
  url = "https://ipv4.icanhazip.com"
}

output "my_public_ip" {
  value = trimspace(data.http.my_ip.response_body)
}

# ใช้ IP ของเครื่องที่รัน Terraform ใน security group
resource "aws_security_group_rule" "allow_my_ip" {
  type              = "ingress"
  from_port         = 22
  to_port           = 22
  protocol          = "tcp"
  cidr_blocks       = ["${trimspace(data.http.my_ip.response_body)}/32"]
  security_group_id = aws_security_group.bastion.id
  description       = "Allow SSH from my IP"
}

# ดึง JSON จาก API
data "http" "github_meta" {
  url = "https://api.github.com/meta"
  
  request_headers = {
    Accept = "application/json"
  }
}

locals {
  github_hooks_ips = jsondecode(data.http.github_meta.response_body).hooks
}

# อนุญาต GitHub Webhooks
resource "aws_security_group_rule" "github_webhooks" {
  count = length(local.github_hooks_ips)
  
  type              = "ingress"
  from_port         = 443
  to_port           = 443
  protocol          = "tcp"
  cidr_blocks       = [local.github_hooks_ips[count.index]]
  security_group_id = aws_security_group.api.id
  description       = "GitHub Webhook ${local.github_hooks_ips[count.index]}"
}
```

### Data Source Dependencies

```hcl
# Data sources ที่ depends on resource อื่น
resource "aws_vpc" "main" {
  cidr_block = "10.0.0.0/16"
  
  tags = { 
    Name = "main-vpc"
    Created = "terraform"
  }
}

# Data source ที่อ่าน resource ที่เพิ่งสร้าง
data "aws_vpc" "main" {
  id = aws_vpc.main.id  # dependency implicit
}

# หรือถ้าต้องการ explicit dependency
data "aws_vpc" "by_tag" {
  tags = { Name = "main-vpc" }
  
  # ⚠️ ถ้า VPC ถูกสร้างในรอบเดียวกัน ต้อง depends_on
  depends_on = [aws_vpc.main]
}

# ตัวอย่าง: สร้าง S3 bucket แล้วอ่านข้อมูลมันกลับ
resource "aws_s3_bucket" "config" {
  bucket = "my-config-bucket"
}

data "aws_s3_bucket" "config" {
  bucket = aws_s3_bucket.config.bucket
  
  depends_on = [aws_s3_bucket.config]
}

output "config_bucket_arn" {
  value = data.aws_s3_bucket.config.arn
}
```

### Data Source Refresh

```hcl
# Data sources ถูก refresh ทุกครั้งที่รัน terraform plan/apply
# ถ้าต้องการ force refresh:
# $ terraform refresh (deprecated)
# $ terraform apply -refresh-only

# ตัวอย่าง: data source ที่เปลี่ยนแปลงบ่อย
data "aws_ami" "latest" {
  most_recent = true  # จะได้ AMI ใหม่ทุกครั้งที่ plan
  owners      = ["amazon"]
  
  filter {
    name   = "name"
    values = ["amzn2-ami-hvm-*-x86_64-gp2"]
  }
}

# ⚠️ ระวัง: ถ้า AMI เปลี่ยน และไม่มี lifecycle.ignore_changes
# Terraform อาจ destroy และสร้าง instance ใหม่!

resource "aws_instance" "stable" {
  ami           = data.aws_ami.latest.id
  instance_type = "t3.micro"
  
  lifecycle {
    # ป้องกัน replace เมื่อ AMI ใหม่ออก
    ignore_changes = [ami]
  }
}
```

---

## ตัวอย่าง Real-World: Look Up Existing Infrastructure

### Pattern 1: Deploy แอพเข้า Existing VPC

```hcl
# ดึงข้อมูล infrastructure ที่สร้างโดย team อื่น
data "aws_vpc" "shared" {
  tags = {
    Name        = "shared-vpc"
    Environment = var.environment
  }
}

data "aws_subnets" "app" {
  filter {
    name   = "vpc-id"
    values = [data.aws_vpc.shared.id]
  }
  
  tags = { Tier = "application" }
}

data "aws_security_group" "shared_alb" {
  vpc_id = data.aws_vpc.shared.id
  
  tags = { Name = "shared-alb-sg" }
}

# Deploy แอพเข้า existing infrastructure
resource "aws_ecs_service" "app" {
  name            = var.app_name
  cluster         = aws_ecs_cluster.main.id
  task_definition = aws_ecs_task_definition.app.arn
  desired_count   = var.desired_count

  network_configuration {
    subnets          = data.aws_subnets.app.ids
    security_groups  = [aws_security_group.app.id]
    assign_public_ip = false
  }

  load_balancer {
    target_group_arn = aws_lb_target_group.app.arn
    container_name   = var.app_name
    container_port   = 8080
  }
}
```

### Pattern 2: Multi-Account Resource Lookup

```hcl
# Provider สำหรับ shared services account
provider "aws" {
  alias  = "shared"
  region = "us-east-1"
  
  assume_role {
    role_arn = "arn:aws:iam::${var.shared_account_id}:role/ReadOnlyAccess"
  }
}

# ดึงข้อมูลจาก shared account
data "aws_vpc" "shared" {
  provider = aws.shared
  
  tags = { Name = "shared-services-vpc" }
}

data "aws_route53_zone" "main" {
  provider = aws.shared
  name     = "internal.company.com."
  private_zone = true
}

# สร้าง VPC peering
resource "aws_vpc_peering_connection" "to_shared" {
  vpc_id        = aws_vpc.app.id
  peer_vpc_id   = data.aws_vpc.shared.id
  peer_owner_id = var.shared_account_id
  peer_region   = "us-east-1"
  auto_accept   = false
}
```

### Pattern 3: เชื่อม Lambda กับ Existing Resources

```hcl
# หา SQS Queue ที่มีอยู่
data "aws_sqs_queue" "input" {
  name = "order-processing-queue"
}

# หา DynamoDB table ที่มีอยู่
data "aws_dynamodb_table" "orders" {
  name = "orders"
}

# หา Secret จาก Secrets Manager
data "aws_secretsmanager_secret" "db_credentials" {
  name = "production/db/credentials"
}

data "aws_secretsmanager_secret_version" "db_credentials" {
  secret_id = data.aws_secretsmanager_secret.db_credentials.id
}

locals {
  db_credentials = jsondecode(data.aws_secretsmanager_secret_version.db_credentials.secret_string)
}

# Lambda function ที่ใช้ existing resources
resource "aws_lambda_function" "processor" {
  filename         = "processor.zip"
  function_name    = "order-processor"
  role             = aws_iam_role.lambda.arn
  handler          = "index.handler"
  runtime          = "nodejs18.x"
  source_code_hash = filebase64sha256("processor.zip")

  environment {
    variables = {
      QUEUE_URL  = data.aws_sqs_queue.input.url
      TABLE_NAME = data.aws_dynamodb_table.orders.name
      DB_HOST    = local.db_credentials["host"]
      DB_NAME    = local.db_credentials["dbname"]
    }
  }
}

# SQS trigger
resource "aws_lambda_event_source_mapping" "sqs" {
  event_source_arn = data.aws_sqs_queue.input.arn
  function_name    = aws_lambda_function.processor.arn
  batch_size       = 10
}
```

---

## Common Data Sources Reference

```hcl
# AWS Organizations
data "aws_organizations_organization" "main" {}

# EKS Cluster
data "aws_eks_cluster" "main" {
  name = "my-cluster"
}

data "aws_eks_cluster_auth" "main" {
  name = "my-cluster"
}

# SSM Parameters
data "aws_ssm_parameter" "db_password" {
  name            = "/app/prod/db_password"
  with_decryption = true  # สำหรับ SecureString
}

# Secrets Manager
data "aws_secretsmanager_secret_version" "app_secret" {
  secret_id = "my-app/production/config"
}

# KMS Key
data "aws_kms_key" "rds" {
  key_id = "alias/rds-encryption-key"
}

# IAM Role
data "aws_iam_role" "existing" {
  name = "existing-role-name"
}

# Elastic IP
data "aws_eip" "bastion" {
  filter {
    name   = "tag:Name"
    values = ["bastion-eip"]
  }
}

# ECR Repository
data "aws_ecr_repository" "app" {
  name = "my-app"
}

data "aws_ecr_image" "latest" {
  repository_name = data.aws_ecr_repository.app.name
  image_tag       = "latest"
}

output "latest_image_digest" {
  value = data.aws_ecr_image.latest.image_digest
}
```

---

## Best Practices สำหรับ Data Sources

```hcl
# ✅ DO: ใช้ data source แทนการ hardcode IDs
# ❌ DON'T
resource "aws_instance" "bad" {
  ami       = "ami-0c55b159cbfafe1f0"  # hardcoded - ใช้ได้แค่ region เดียว
  subnet_id = "subnet-12345678"         # hardcoded - ไม่ flexible
}

# ✅ DO
data "aws_ami" "ubuntu" {
  most_recent = true
  owners      = ["099720109477"]
  filter {
    name   = "name"
    values = ["ubuntu/images/hvm-ssd/ubuntu-jammy-22.04-amd64-server-*"]
  }
}

data "aws_subnet" "web" {
  tags = { Name = "web-subnet", Environment = var.environment }
}

resource "aws_instance" "good" {
  ami       = data.aws_ami.ubuntu.id
  subnet_id = data.aws_subnet.web.id
}

# ✅ DO: ตรวจสอบ data source ว่าเจอแค่ 1 ผลลัพธ์
# ถ้า filter match หลาย resources จะ error

# ✅ DO: ใช้ depends_on กับ data source ที่ depends on resource ที่เพิ่งสร้าง
data "aws_vpc" "newly_created" {
  id         = aws_vpc.main.id
  depends_on = [aws_vpc.main]
}

# ✅ DO: Cache ค่าใน locals เพื่อใช้ซ้ำ
locals {
  vpc_id         = data.aws_vpc.main.id
  private_subnets = data.aws_subnets.private.ids
  account_id     = data.aws_caller_identity.current.account_id
  region         = data.aws_region.current.name
}
```

---

## แบบฝึกหัด (Exercises)

### Exercise 1: AMI Lookup

เขียน data source เพื่อหา Amazon Linux 2023 AMI ล่าสุด และใช้สร้าง EC2 instance

### Exercise 2: Existing VPC Discovery

เขียน code เพื่อ:
1. หา production VPC ด้วย tag
2. หา private subnets ทั้งหมดใน VPC นั้น
3. Deploy EC2 instances ใน subnets เหล่านั้น

### Exercise 3: Cross-Stack References

เขียน code ที่:
1. อ่าน networking state จาก S3 backend
2. ใช้ VPC ID และ subnet IDs จาก state นั้น
3. Deploy application stack

---

## Checklist

- [ ] เข้าใจความแตกต่างระหว่าง resource และ data block
- [ ] รู้วิธีใช้ aws_ami data source กับ filter
- [ ] รู้วิธีหา VPC และ subnets ที่มีอยู่
- [ ] สามารถใช้ aws_caller_identity และ aws_region ได้
- [ ] เข้าใจ aws_iam_policy_document
- [ ] รู้วิธีใช้ terraform_remote_state
- [ ] สามารถใช้ http data source ได้
- [ ] รู้วิธีใช้ depends_on กับ data source
