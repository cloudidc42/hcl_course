# Part 069: Provider Aliasing & Multi-Region
## Provider Aliasing และ Multi-Region Deployment
### Steps 681-690

---

## บทนำ (Introduction)

Provider aliasing ช่วยให้เราสามารถ deploy resources ไปยัง หลาย regions, หลาย accounts, หรือหลาย cloud environments ได้ในครั้งเดียว การเข้าใจ provider aliasing เป็นสิ่งจำเป็นสำหรับ enterprise-grade Terraform configurations

---

## Step 681: Default vs Aliased Providers

### ความแตกต่างระหว่าง default และ aliased providers

```hcl
# ==========================================
# DEFAULT PROVIDER
# ==========================================

# Default provider - ใช้สำหรับทุก resource ที่ไม่ระบุ provider
provider "aws" {
  region = "ap-southeast-1"  # Default region
}

# Resources ที่ไม่ระบุ provider จะใช้ default
resource "aws_vpc" "main" {
  cidr_block = "10.0.0.0/16"
  # ใช้ provider "aws" default (ap-southeast-1)
}

# ==========================================
# ALIASED PROVIDER
# ==========================================

provider "aws" {
  alias  = "us_east_1"
  region = "us-east-1"
}

provider "aws" {
  alias  = "eu_west_1"
  region = "eu-west-1"
}

provider "aws" {
  alias  = "ap_northeast_1"
  region = "ap-northeast-1"
}

# Resources ที่ระบุ provider
resource "aws_vpc" "us_east" {
  provider   = aws.us_east_1  # ใช้ US East provider
  cidr_block = "10.1.0.0/16"
}

resource "aws_vpc" "eu_west" {
  provider   = aws.eu_west_1  # ใช้ EU West provider
  cidr_block = "10.2.0.0/16"
}

# ==========================================
# COMPLETE MULTI-REGION PROVIDER SETUP
# ==========================================

terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

# Primary: Singapore
provider "aws" {
  region = "ap-southeast-1"
  # ไม่มี alias = default provider

  default_tags {
    tags = {
      ManagedBy = "Terraform"
      Project   = var.project
    }
  }
}

# DR: US East
provider "aws" {
  alias  = "us_east_1"
  region = "us-east-1"

  default_tags {
    tags = {
      ManagedBy = "Terraform"
      Project   = var.project
      Region    = "us-east-1"
    }
  }
}

# APAC: Tokyo
provider "aws" {
  alias  = "ap_northeast_1"
  region = "ap-northeast-1"

  default_tags {
    tags = {
      ManagedBy = "Terraform"
      Project   = var.project
      Region    = "ap-northeast-1"
    }
  }
}

# Global: IAM, Route53, CloudFront (us-east-1 based)
provider "aws" {
  alias  = "global"
  region = "us-east-1"  # ACM for CloudFront ต้องอยู่ us-east-1

  default_tags {
    tags = {
      ManagedBy = "Terraform"
      Scope     = "global"
    }
  }
}
```

---

## Step 682: Passing Providers to Resources

### การส่ง provider ไปยัง resources

```hcl
# ==========================================
# PASSING PROVIDERS TO RESOURCES
# ==========================================

# Syntax: provider = <provider>.<alias>

# IAM (Global)
resource "aws_iam_role" "app" {
  provider = aws.global  # หรือ aws ถ้า global = default
  name     = "${var.project}-app-role"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Action = "sts:AssumeRole"
      Effect = "Allow"
      Principal = {
        Service = "ecs-tasks.amazonaws.com"
      }
    }]
  })
}

# Route 53 (Global)
resource "aws_route53_zone" "main" {
  provider = aws.global
  name     = "mycompany.com"
}

# ACM for CloudFront (must be us-east-1)
resource "aws_acm_certificate" "cloudfront" {
  provider          = aws.global  # us-east-1
  domain_name       = "*.mycompany.com"
  validation_method = "DNS"
}

# Regional resources
resource "aws_vpc" "primary" {
  # ไม่ระบุ provider = ใช้ default (ap-southeast-1)
  cidr_block = "10.0.0.0/16"
}

resource "aws_vpc" "dr" {
  provider   = aws.us_east_1  # DR region
  cidr_block = "10.1.0.0/16"
}

# CloudFront (Global)
resource "aws_cloudfront_distribution" "main" {
  provider = aws.global

  enabled = true

  origin {
    domain_name = aws_lb.primary.dns_name
    origin_id   = "primary-alb"

    custom_origin_config {
      http_port              = 80
      https_port             = 443
      origin_protocol_policy = "https-only"
      origin_ssl_protocols   = ["TLSv1.2"]
    }
  }

  viewer_certificate {
    acm_certificate_arn = aws_acm_certificate.cloudfront.arn
    ssl_support_method  = "sni-only"
  }

  # ต้อง depend_on cert validation
  depends_on = [aws_acm_certificate_validation.cloudfront]
}
```

---

## Step 683: Passing Providers to Modules

### การส่ง provider ไปยัง modules

```hcl
# ==========================================
# PASSING PROVIDERS TO MODULES
# ==========================================

# modules/vpc/main.tf
# ไม่ต้อง declare provider configuration ใน module
# ใช้ implicit provider จาก caller

resource "aws_vpc" "main" {
  cidr_block = var.cidr_block
  # ใช้ provider ที่ถูก inherit มาจาก caller
}

# ==========================================
# MODULE PROVIDER REQUIREMENTS
# ==========================================

# modules/vpc/versions.tf
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = ">= 4.0"
    }
  }
}

# ==========================================
# EXPLICIT PROVIDER PASSING
# ==========================================

# root/main.tf
module "vpc_primary" {
  source = "./modules/vpc"

  cidr_block  = "10.0.0.0/16"
  project     = var.project
  environment = var.environment

  # ใช้ default provider (ap-southeast-1)
  # ไม่ต้อง providers block ถ้าใช้ default
}

module "vpc_dr" {
  source = "./modules/vpc"

  cidr_block  = "10.1.0.0/16"
  project     = var.project
  environment = "${var.environment}-dr"

  # ส่ง provider ไปยัง module
  providers = {
    aws = aws.us_east_1  # Override default provider
  }
}

module "vpc_tokyo" {
  source = "./modules/vpc"

  cidr_block  = "10.2.0.0/16"
  project     = var.project
  environment = "${var.environment}-apne1"

  providers = {
    aws = aws.ap_northeast_1
  }
}

# ==========================================
# MODULE ที่ใช้ MULTIPLE PROVIDERS
# ==========================================

# modules/cross-region-replication/main.tf
# Module นี้ต้องการ 2 providers: source และ destination

terraform {
  required_providers {
    aws = {
      source                = "hashicorp/aws"
      version               = ">= 4.0"
      configuration_aliases = [aws.source, aws.destination]
    }
  }
}

resource "aws_s3_bucket" "source" {
  provider = aws.source
  bucket   = "${var.project}-source-bucket"
}

resource "aws_s3_bucket" "destination" {
  provider = aws.destination
  bucket   = "${var.project}-destination-bucket"
}

resource "aws_s3_bucket_replication_configuration" "main" {
  provider = aws.source
  bucket   = aws_s3_bucket.source.id
  role     = aws_iam_role.replication.arn

  rule {
    status = "Enabled"

    destination {
      bucket = aws_s3_bucket.destination.arn
    }
  }
}

# root/main.tf - เรียกใช้ module
module "replication" {
  source = "./modules/cross-region-replication"

  project = var.project

  providers = {
    aws.source      = aws          # Default provider (primary region)
    aws.destination = aws.us_east_1  # DR region
  }
}
```

---

## Step 684: Multi-Region Patterns

### รูปแบบ Multi-Region

```hcl
# ==========================================
# PATTERN 1: PRIMARY + DR REGION
# ==========================================

# Providers
provider "aws" {
  region = "ap-southeast-1"  # Primary
}

provider "aws" {
  alias  = "dr"
  region = "us-east-1"  # DR
}

# Primary VPC
module "vpc_primary" {
  source = "./modules/vpc"

  cidr_block  = "10.0.0.0/16"
  environment = "prod"
}

# DR VPC (smaller)
module "vpc_dr" {
  source = "./modules/vpc"

  providers = { aws = aws.dr }

  cidr_block  = "10.1.0.0/16"
  environment = "prod-dr"
}

# Primary RDS
module "rds_primary" {
  source = "./modules/rds"

  vpc_id         = module.vpc_primary.vpc_id
  subnet_ids     = module.vpc_primary.database_subnet_ids
  instance_class = "db.r6g.large"
  multi_az       = true
}

# RDS Read Replica in DR region
resource "aws_db_instance" "dr_replica" {
  provider = aws.dr

  identifier             = "prod-dr-replica"
  replicate_source_db    = module.rds_primary.db_instance_arn
  instance_class         = "db.r6g.medium"  # Smaller in DR
  publicly_accessible    = false
  skip_final_snapshot    = true
  deletion_protection    = true
}

# ==========================================
# PATTERN 2: ACTIVE-ACTIVE MULTI-REGION
# ==========================================

locals {
  regions = {
    "ap-southeast-1" = {
      provider = "aws"
      vpc_cidr = "10.0.0.0/16"
      weight   = 50
    }
    "us-east-1" = {
      provider = "aws.us_east_1"
      vpc_cidr = "10.1.0.0/16"
      weight   = 50
    }
  }
}

# Route 53 Latency-based routing
resource "aws_route53_record" "app_apac" {
  zone_id = aws_route53_zone.main.id
  name    = "app.mycompany.com"
  type    = "A"

  alias {
    name                   = module.alb_primary.dns_name
    zone_id                = module.alb_primary.zone_id
    evaluate_target_health = true
  }

  set_identifier = "apac"
  latency_routing_policy {
    region = "ap-southeast-1"
  }
}

resource "aws_route53_record" "app_us" {
  zone_id = aws_route53_zone.main.id
  name    = "app.mycompany.com"
  type    = "A"

  alias {
    name                   = module.alb_us.dns_name
    zone_id                = module.alb_us.zone_id
    evaluate_target_health = true
  }

  set_identifier = "us"
  latency_routing_policy {
    region = "us-east-1"
  }
}
```

---

## Step 685: Multi-Account Patterns

### รูปแบบ Multi-Account

```hcl
# ==========================================
# MULTI-ACCOUNT: HUB/SPOKE
# ==========================================

# Hub Account Provider (Shared Services)
provider "aws" {
  alias  = "hub"
  region = "ap-southeast-1"

  assume_role {
    role_arn     = "arn:aws:iam::111111111111:role/TerraformExecutor"
    session_name = "terraform-hub"
  }
}

# Spoke 1: Production Account
provider "aws" {
  alias  = "prod"
  region = "ap-southeast-1"

  assume_role {
    role_arn     = "arn:aws:iam::222222222222:role/TerraformExecutor"
    session_name = "terraform-prod"
  }
}

# Spoke 2: Staging Account
provider "aws" {
  alias  = "staging"
  region = "ap-southeast-1"

  assume_role {
    role_arn     = "arn:aws:iam::333333333333:role/TerraformExecutor"
    session_name = "terraform-staging"
  }
}

# Spoke 3: Dev Account
provider "aws" {
  alias  = "dev"
  region = "ap-southeast-1"

  assume_role {
    role_arn     = "arn:aws:iam::444444444444:role/TerraformExecutor"
    session_name = "terraform-dev"
  }
}

# Hub resources
resource "aws_vpc" "hub" {
  provider   = aws.hub
  cidr_block = "10.100.0.0/16"

  tags = { Name = "hub-vpc" }
}

# Spoke resources
resource "aws_vpc" "prod" {
  provider   = aws.prod
  cidr_block = "10.0.0.0/16"

  tags = { Name = "prod-vpc" }
}

resource "aws_vpc" "staging" {
  provider   = aws.staging
  cidr_block = "10.1.0.0/16"

  tags = { Name = "staging-vpc" }
}

# VPC Peering: Hub <-> Prod
resource "aws_vpc_peering_connection" "hub_prod" {
  provider = aws.hub

  vpc_id        = aws_vpc.hub.id
  peer_vpc_id   = aws_vpc.prod.id
  peer_owner_id = "222222222222"  # Prod account ID
  peer_region   = "ap-southeast-1"
  auto_accept   = false
}

resource "aws_vpc_peering_connection_accepter" "hub_prod" {
  provider = aws.prod

  vpc_peering_connection_id = aws_vpc_peering_connection.hub_prod.id
  auto_accept               = true
}

# ==========================================
# ASSUME_ROLE CONFIGURATION
# ==========================================

provider "aws" {
  alias  = "cross_account"
  region = "ap-southeast-1"

  assume_role {
    role_arn     = var.cross_account_role_arn
    session_name = "terraform-${var.project}"
    external_id  = var.external_id  # Security best practice

    # กำหนด scope
    policy = jsonencode({
      Version = "2012-10-17"
      Statement = [{
        Effect   = "Allow"
        Action   = ["ec2:*", "iam:*"]
        Resource = "*"
      }]
    })
  }
}
```

---

## Step 686: Workarounds for Dynamic Providers

### วิธีแก้ปัญหา Dynamic Providers (ที่ไม่ support natively)

```hcl
# ==========================================
# PROBLEM: ไม่สามารถใช้ for_each กับ providers
# ==========================================

# ไม่สามารถทำแบบนี้ได้:
# provider "aws" {
#   for_each = var.regions  # ERROR - providers ไม่รองรับ for_each
#   alias    = each.key
#   region   = each.key
# }

# ==========================================
# WORKAROUND 1: Declare providers statically
# ==========================================

# แก้ด้วยการ declare providers ทั้งหมดที่จำเป็น
provider "aws" { region = "ap-southeast-1" }
provider "aws" { alias = "r2"; region = "us-east-1" }
provider "aws" { alias = "r3"; region = "eu-west-1" }

# สร้าง resource ด้วย for_each สำหรับแต่ละ provider
module "infra_r1" {
  source = "./modules/regional"
  region = "ap-southeast-1"
  providers = { aws = aws }
}

module "infra_r2" {
  source = "./modules/regional"
  region = "us-east-1"
  providers = { aws = aws.r2 }
}

# ==========================================
# WORKAROUND 2: Separate Root Modules per Region
# ==========================================

# deployments/
# ├── ap-southeast-1/
# │   ├── main.tf
# │   └── terraform.tfvars
# ├── us-east-1/
# │   ├── main.tf
# │   └── terraform.tfvars
# └── eu-west-1/
#     ├── main.tf
#     └── terraform.tfvars

# deployments/ap-southeast-1/main.tf
provider "aws" {
  region = "ap-southeast-1"
}

module "infra" {
  source = "../../modules/regional"
  region = "ap-southeast-1"
}

# ==========================================
# WORKAROUND 3: Terragrunt (DRY approach)
# ==========================================

# terragrunt.hcl (root)
# locals {
#   regions = ["ap-southeast-1", "us-east-1", "eu-west-1"]
# }
#
# generate "provider" {
#   path      = "provider.tf"
#   if_exists = "overwrite_terragrunt"
#   contents  = <<EOF
# provider "aws" {
#   region = "${local.region}"
# }
# EOF
# }
```

---

## Step 687: Multi-Region Module Design

### การออกแบบ module สำหรับ multi-region

```hcl
# ==========================================
# MULTI-REGION MODULE DESIGN
# ==========================================

# modules/regional-stack/versions.tf
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = ">= 5.0"
    }
  }
}

# modules/regional-stack/variables.tf
variable "region" {
  type        = string
  description = "AWS Region ที่ deploy"
}

variable "project" {
  type        = string
  description = "ชื่อโปรเจค"
}

variable "environment" {
  type        = string
  description = "สภาพแวดล้อม"
}

variable "is_primary" {
  type        = bool
  description = "เป็น primary region หรือไม่"
  default     = false
}

variable "vpc_cidr" {
  type        = string
  description = "CIDR สำหรับ VPC"
}

variable "primary_db_arn" {
  type        = string
  description = "ARN ของ primary database (สำหรับ DR replication)"
  default     = null
}

# modules/regional-stack/main.tf
module "vpc" {
  source = "../vpc"

  project     = var.project
  environment = var.environment
  cidr_block  = var.vpc_cidr
}

module "ecs_cluster" {
  source = "../ecs-cluster"

  project     = var.project
  environment = var.environment
  vpc_id      = module.vpc.vpc_id
  subnet_ids  = module.vpc.private_subnet_ids
}

# Primary: Create new RDS
resource "aws_db_instance" "primary" {
  count = var.is_primary ? 1 : 0

  identifier     = "${var.project}-${var.environment}-db"
  engine         = "postgres"
  instance_class = "db.r6g.large"
  # ...
}

# DR: Create read replica
resource "aws_db_instance" "replica" {
  count = var.is_primary ? 0 : 1

  identifier          = "${var.project}-${var.environment}-dr-db"
  replicate_source_db = var.primary_db_arn
  instance_class      = "db.r6g.medium"
  # ...
}

# modules/regional-stack/outputs.tf
output "vpc_id" { value = module.vpc.vpc_id }
output "ecs_cluster_arn" { value = module.ecs_cluster.cluster_arn }
output "db_endpoint" {
  value = var.is_primary ? aws_db_instance.primary[0].endpoint : aws_db_instance.replica[0].endpoint
}
```

---

## Step 688: Complete Multi-Region Example

### ตัวอย่างสมบูรณ์ Multi-Region

```hcl
# ==========================================
# COMPLETE MULTI-REGION SETUP
# root/main.tf
# ==========================================

terraform {
  required_version = ">= 1.5.0"
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
  backend "s3" {
    bucket = "mycompany-terraform-state"
    key    = "global/main.tfstate"
    region = "ap-southeast-1"
  }
}

# ==========================================
# PROVIDERS
# ==========================================

# Primary: Singapore
provider "aws" {
  region = "ap-southeast-1"

  default_tags {
    tags = {
      Project   = var.project
      ManagedBy = "Terraform"
    }
  }
}

# DR: US East
provider "aws" {
  alias  = "us_east_1"
  region = "us-east-1"

  default_tags {
    tags = {
      Project   = var.project
      ManagedBy = "Terraform"
      Region    = "us-east-1"
    }
  }
}

# Global (ACM for CloudFront, Route53, IAM)
provider "aws" {
  alias  = "global"
  region = "us-east-1"

  default_tags {
    tags = {
      Project   = var.project
      ManagedBy = "Terraform"
      Scope     = "global"
    }
  }
}

# ==========================================
# GLOBAL RESOURCES (ต้องสร้างก่อน)
# ==========================================

# Route 53 Zone
resource "aws_route53_zone" "main" {
  provider = aws.global
  name     = var.domain_name
}

# Global ACM Certificate (for CloudFront)
resource "aws_acm_certificate" "global" {
  provider          = aws.global  # CloudFront requires us-east-1
  domain_name       = "*.${var.domain_name}"
  validation_method = "DNS"

  subject_alternative_names = [
    var.domain_name,
    "*.${var.domain_name}"
  ]

  lifecycle {
    create_before_destroy = true
  }
}

# Validate ACM Certificate
resource "aws_route53_record" "cert_validation" {
  provider = aws.global
  for_each = {
    for dvo in aws_acm_certificate.global.domain_validation_options :
    dvo.domain_name => {
      name   = dvo.resource_record_name
      record = dvo.resource_record_value
      type   = dvo.resource_record_type
    }
  }

  zone_id = aws_route53_zone.main.zone_id
  name    = each.value.name
  type    = each.value.type
  records = [each.value.record]
  ttl     = 60
}

resource "aws_acm_certificate_validation" "global" {
  provider                = aws.global
  certificate_arn         = aws_acm_certificate.global.arn
  validation_record_fqdns = [for r in aws_route53_record.cert_validation : r.fqdn]
}

# Regional ACM Certificate (for ALB in ap-southeast-1)
resource "aws_acm_certificate" "primary" {
  # ไม่ระบุ provider = ใช้ default (ap-southeast-1)
  domain_name       = "*.${var.domain_name}"
  validation_method = "DNS"

  lifecycle {
    create_before_destroy = true
  }
}

# Regional ACM Certificate (for ALB in us-east-1)
resource "aws_acm_certificate" "dr" {
  provider          = aws.us_east_1
  domain_name       = "*.${var.domain_name}"
  validation_method = "DNS"

  lifecycle {
    create_before_destroy = true
  }
}

# ==========================================
# PRIMARY REGION (AP-SOUTHEAST-1)
# ==========================================

module "primary" {
  source = "./modules/regional-stack"
  # ไม่ระบุ providers = ใช้ default

  project     = var.project
  environment = var.environment
  region      = "ap-southeast-1"
  is_primary  = true
  vpc_cidr    = "10.0.0.0/16"

  acm_certificate_arn = aws_acm_certificate.primary.arn
}

# Primary Route 53 records
resource "aws_route53_record" "primary_app" {
  provider = aws.global
  zone_id  = aws_route53_zone.main.id
  name     = "app.${var.domain_name}"
  type     = "A"

  alias {
    name                   = module.primary.alb_dns_name
    zone_id                = module.primary.alb_zone_id
    evaluate_target_health = true
  }

  # Failover routing
  set_identifier = "primary"
  failover_routing_policy {
    type = "PRIMARY"
  }

  health_check_id = aws_route53_health_check.primary.id
}

resource "aws_route53_health_check" "primary" {
  provider          = aws.global
  fqdn              = module.primary.alb_dns_name
  port              = 443
  type              = "HTTPS"
  resource_path     = "/health"
  failure_threshold = 3
  request_interval  = 30

  tags = { Name = "primary-health-check" }
}

# ==========================================
# DR REGION (US-EAST-1)
# ==========================================

module "dr" {
  source = "./modules/regional-stack"

  providers = {
    aws = aws.us_east_1
  }

  project        = var.project
  environment    = "${var.environment}-dr"
  region         = "us-east-1"
  is_primary     = false
  vpc_cidr       = "10.1.0.0/16"
  primary_db_arn = module.primary.db_arn

  acm_certificate_arn = aws_acm_certificate.dr.arn
}

# DR Route 53 records (Failover secondary)
resource "aws_route53_record" "dr_app" {
  provider = aws.global
  zone_id  = aws_route53_zone.main.id
  name     = "app.${var.domain_name}"
  type     = "A"

  alias {
    name                   = module.dr.alb_dns_name
    zone_id                = module.dr.alb_zone_id
    evaluate_target_health = true
  }

  set_identifier = "dr"
  failover_routing_policy {
    type = "SECONDARY"
  }

  health_check_id = aws_route53_health_check.dr.id
}

resource "aws_route53_health_check" "dr" {
  provider          = aws.global
  fqdn              = module.dr.alb_dns_name
  port              = 443
  type              = "HTTPS"
  resource_path     = "/health"
  failure_threshold = 3
  request_interval  = 30

  tags = { Name = "dr-health-check" }
}

# ==========================================
# S3 CROSS-REGION REPLICATION
# ==========================================

# Source bucket (Primary)
resource "aws_s3_bucket" "primary" {
  bucket = "${var.project}-${var.environment}-primary-data"
}

resource "aws_s3_bucket_versioning" "primary" {
  bucket = aws_s3_bucket.primary.id

  versioning_configuration {
    status = "Enabled"
  }
}

# Destination bucket (DR)
resource "aws_s3_bucket" "dr" {
  provider = aws.us_east_1
  bucket   = "${var.project}-${var.environment}-dr-data"
}

resource "aws_s3_bucket_versioning" "dr" {
  provider = aws.us_east_1
  bucket   = aws_s3_bucket.dr.id

  versioning_configuration {
    status = "Enabled"
  }
}

# Replication configuration
resource "aws_s3_bucket_replication_configuration" "main" {
  bucket = aws_s3_bucket.primary.id
  role   = aws_iam_role.s3_replication.arn

  rule {
    id     = "replicate-to-dr"
    status = "Enabled"

    destination {
      bucket        = aws_s3_bucket.dr.arn
      storage_class = "STANDARD_IA"
    }
  }

  depends_on = [
    aws_s3_bucket_versioning.primary,
    aws_s3_bucket_versioning.dr,
  ]
}

# ==========================================
# CLOUDFRONT (GLOBAL)
# ==========================================

resource "aws_cloudfront_distribution" "main" {
  provider = aws.global

  enabled             = true
  is_ipv6_enabled     = true
  default_root_object = "index.html"

  aliases = ["app.${var.domain_name}"]

  # Primary origin
  origin {
    domain_name = module.primary.alb_dns_name
    origin_id   = "primary"

    custom_origin_config {
      http_port              = 80
      https_port             = 443
      origin_protocol_policy = "https-only"
      origin_ssl_protocols   = ["TLSv1.2"]
    }
  }

  # DR origin
  origin {
    domain_name = module.dr.alb_dns_name
    origin_id   = "dr"

    custom_origin_config {
      http_port              = 80
      https_port             = 443
      origin_protocol_policy = "https-only"
      origin_ssl_protocols   = ["TLSv1.2"]
    }
  }

  # Origin group for failover
  origin_group {
    origin_id = "failover-group"

    failover_criteria {
      status_codes = [500, 502, 503, 504]
    }

    member {
      origin_id = "primary"
    }

    member {
      origin_id = "dr"
    }
  }

  default_cache_behavior {
    allowed_methods  = ["DELETE", "GET", "HEAD", "OPTIONS", "PATCH", "POST", "PUT"]
    cached_methods   = ["GET", "HEAD"]
    target_origin_id = "failover-group"

    viewer_protocol_policy = "redirect-to-https"
    compress               = true
    cache_policy_id        = "658327ea-f89d-4fab-a63d-7e88639e58f6"  # CachingOptimized
  }

  viewer_certificate {
    acm_certificate_arn = aws_acm_certificate_validation.global.certificate_arn
    ssl_support_method  = "sni-only"
    minimum_protocol_version = "TLSv1.2_2021"
  }

  restrictions {
    geo_restriction {
      restriction_type = "none"
    }
  }

  depends_on = [
    aws_acm_certificate_validation.global,
    module.primary,
    module.dr
  ]
}

# ==========================================
# OUTPUTS
# ==========================================

output "primary_url" {
  value = "https://app.${var.domain_name} (primary: ap-southeast-1)"
}

output "dr_url" {
  value = "https://app.${var.domain_name} (dr: us-east-1)"
}

output "cloudfront_domain" {
  value = aws_cloudfront_distribution.main.domain_name
}

output "route53_nameservers" {
  value = aws_route53_zone.main.name_servers
}
```

---

## Step 689 & 690: Variables and Summary

```hcl
# ==========================================
# variables.tf
# ==========================================

variable "project" {
  type        = string
  description = "ชื่อโปรเจค"
  default     = "mycompany"
}

variable "environment" {
  type        = string
  description = "สภาพแวดล้อม"
  default     = "prod"
}

variable "domain_name" {
  type        = string
  description = "Domain name หลัก"
  default     = "mycompany.com"
}
```

---

## สรุป (Summary)

### Provider Aliasing Pattern Summary:

| Use Case | Pattern |
|----------|---------|
| Multi-Region | Provider per region with alias |
| Multi-Account | Provider with assume_role per account |
| Global Resources | Provider aliased to us-east-1 |
| Module multi-provider | configuration_aliases in module |

### Provider Passing to Modules:
```hcl
# Module กำหนด required providers
terraform {
  required_providers {
    aws = {
      configuration_aliases = [aws.primary, aws.secondary]
    }
  }
}

# Caller ส่ง providers ผ่าน providers block
module "my_module" {
  providers = {
    aws.primary   = aws.prod
    aws.secondary = aws.dr
  }
}
```

---

*จบ Part 069 - Provider Aliasing & Multi-Region*
