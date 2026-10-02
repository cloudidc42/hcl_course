# Part 065: Module Versioning & Registry
## การจัดการ Version และ Registry ของ Module
### Steps 641-650

---

## บทนำ (Introduction)

Terraform Registry เป็น marketplace สำหรับ Terraform modules และ providers การเข้าใจวิธีการใช้ registry modules และการ publish modules ของตัวเองจะช่วยให้ทำงานได้อย่างมีประสิทธิภาพ

---

## Step 641: Terraform Registry Structure

### โครงสร้างของ Terraform Registry

```
registry.terraform.io
├── modules/
│   ├── hashicorp/          # Official (HashiCorp)
│   │   └── consul/aws
│   ├── terraform-aws-modules/  # Verified
│   │   ├── vpc/aws
│   │   ├── eks/aws
│   │   └── rds/aws
│   └── community-org/      # Community
│       └── mymodule/aws
└── providers/
    ├── hashicorp/aws
    ├── hashicorp/google
    └── datadog/datadog
```

### ประเภท Module ใน Registry:

**1. Official Modules** - สร้างโดย HashiCorp
```hcl
module "consul" {
  source  = "hashicorp/consul/aws"
  version = "~> 0.9"
}
```

**2. Verified Modules** - สร้างโดย Partners ที่ HashiCorp ตรวจสอบแล้ว
```hcl
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "~> 5.0"
}
```

**3. Community Modules** - สร้างโดย Community
```hcl
module "my_module" {
  source  = "community-org/module-name/aws"
  version = "1.0.0"
}
```

---

## Step 642: Module Naming Convention

### รูปแบบชื่อ Module: `namespace/name/provider`

```
terraform-aws-modules/vpc/aws
├── namespace: terraform-aws-modules (GitHub organization)
├── name: vpc (module name)
└── provider: aws (cloud provider)

mycompany/networking/aws
├── namespace: mycompany
├── name: networking
└── provider: aws
```

### Git Repository Naming:
```
GitHub repo name: terraform-<provider>-<name>
ตัวอย่าง:
- terraform-aws-vpc
- terraform-aws-rds
- terraform-google-gke
- terraform-azurerm-aks
```

---

## Step 643: Using Registry Modules

### การใช้งาน modules จาก registry

```hcl
# ==========================================
# TERRAFORM REGISTRY MODULES
# ==========================================

# Official AWS VPC Module
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "~> 5.0"

  name = "my-vpc"
  cidr = "10.0.0.0/16"

  azs             = ["ap-southeast-1a", "ap-southeast-1b", "ap-southeast-1c"]
  private_subnets = ["10.0.1.0/24", "10.0.2.0/24", "10.0.3.0/24"]
  public_subnets  = ["10.0.101.0/24", "10.0.102.0/24", "10.0.103.0/24"]

  enable_nat_gateway = true
  enable_vpn_gateway = false

  tags = {
    Terraform   = "true"
    Environment = "dev"
  }
}

# Official EKS Module
module "eks" {
  source  = "terraform-aws-modules/eks/aws"
  version = "~> 20.0"

  cluster_name    = "my-cluster"
  cluster_version = "1.28"

  vpc_id                         = module.vpc.vpc_id
  subnet_ids                     = module.vpc.private_subnets
  cluster_endpoint_public_access = true

  eks_managed_node_groups = {
    default = {
      min_size     = 1
      max_size     = 3
      desired_size = 2
      instance_types = ["t3.medium"]
    }
  }
}

# Official RDS Module
module "rds" {
  source  = "terraform-aws-modules/rds/aws"
  version = "~> 6.0"

  identifier = "my-database"

  engine            = "postgres"
  engine_version    = "15"
  instance_class    = "db.t3.micro"
  allocated_storage = 20

  db_name  = "mydb"
  username = "admin"

  vpc_security_group_ids = [module.vpc.default_security_group_id]
  db_subnet_group_name   = module.vpc.database_subnet_group

  tags = {
    Owner       = "myteam"
    Environment = "dev"
  }
}

# S3 Module
module "s3_bucket" {
  source  = "terraform-aws-modules/s3-bucket/aws"
  version = "~> 4.0"

  bucket = "my-unique-bucket-name"
  acl    = "private"

  versioning = {
    enabled = true
  }

  server_side_encryption_configuration = {
    rule = {
      apply_server_side_encryption_by_default = {
        sse_algorithm = "AES256"
      }
    }
  }
}
```

---

## Step 644: Version Constraints

### การกำหนด version constraints

```hcl
# ==========================================
# VERSION CONSTRAINT OPERATORS
# ==========================================

# Exact version - ใช้ specific version เท่านั้น
module "vpc_exact" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "5.1.2"   # ต้องเป็น 5.1.2 เท่านั้น
}

# Greater than or equal - ใช้ version นี้หรือใหม่กว่า
module "vpc_gte" {
  source  = "terraform-aws-modules/vpc/aws"
  version = ">= 5.0.0"  # 5.0.0 หรือใหม่กว่า
}

# Greater than - ใช้ version ที่ใหม่กว่า
module "vpc_gt" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "> 5.0.0"  # ใหม่กว่า 5.0.0
}

# Less than or equal
module "vpc_lte" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "<= 5.9.9"  # 5.9.9 หรือเก่ากว่า
}

# Pessimistic constraint (~>) - Recommended!
# ~> 5.0 = >= 5.0, < 6.0 (ยอม minor updates)
module "vpc_tilde_minor" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "~> 5.0"  # 5.x.x แต่ไม่ถึง 6.0
}

# ~> 5.1 = >= 5.1, < 5.2 (ยอมแค่ patch updates)
module "vpc_tilde_patch" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "~> 5.1"  # 5.1.x เท่านั้น
}

# ~> 5.1.2 = >= 5.1.2, < 5.2.0 (ยอมแค่ patch)
module "vpc_tilde_specific" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "~> 5.1.2"  # 5.1.2, 5.1.3, 5.1.4 ...
}

# Combined constraints
module "vpc_range" {
  source  = "terraform-aws-modules/vpc/aws"
  version = ">= 5.0, < 6.0"  # 5.x.x เท่านั้น
}

module "vpc_range2" {
  source  = "terraform-aws-modules/vpc/aws"
  version = ">= 4.0, != 4.5.0"  # ยกเว้น 4.5.0 ที่มี bug
}
```

---

## Step 645: Module Source Types

### ประเภทของ module sources

```hcl
# ==========================================
# 1. TERRAFORM REGISTRY
# ==========================================
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "~> 5.0"
}

# ==========================================
# 2. GITHUB
# ==========================================

# GitHub HTTPS
module "vpc_github" {
  source = "github.com/terraform-aws-modules/terraform-aws-vpc"
}

# GitHub with specific ref (tag/branch/commit)
module "vpc_github_tag" {
  source = "github.com/terraform-aws-modules/terraform-aws-vpc?ref=v5.1.2"
}

# GitHub with subdirectory
module "vpc_github_subdir" {
  source = "github.com/myorg/terraform-modules//modules/vpc?ref=v1.0.0"
  #                                              ^^ double slash = subdirectory
}

# GitHub SSH
module "vpc_github_ssh" {
  source = "git@github.com:myorg/terraform-modules.git//modules/vpc?ref=v1.0.0"
}

# ==========================================
# 3. GENERIC GIT
# ==========================================

# HTTPS Git
module "vpc_git" {
  source = "git::https://github.com/myorg/terraform-modules.git//modules/vpc?ref=v1.0.0"
}

# SSH Git
module "vpc_git_ssh" {
  source = "git::ssh://git@github.com/myorg/terraform-modules.git//modules/vpc?ref=v1.0.0"
}

# ==========================================
# 4. GITLAB
# ==========================================
module "vpc_gitlab" {
  source = "git::https://gitlab.com/myorg/terraform-modules.git//modules/vpc?ref=v1.0.0"
}

# ==========================================
# 5. BITBUCKET
# ==========================================
module "vpc_bitbucket" {
  source = "bitbucket.org/myorg/terraform-modules//modules/vpc?ref=v1.0.0"
}

# ==========================================
# 6. HTTP/HTTPS ARCHIVE
# ==========================================

# HTTP zip archive
module "vpc_http" {
  source = "https://example.com/modules/vpc.zip//vpc"
}

# ==========================================
# 7. S3 BUCKET
# ==========================================
module "vpc_s3" {
  source = "s3::https://s3-ap-southeast-1.amazonaws.com/my-terraform-modules/vpc/v1.0.0.zip"
}

# ==========================================
# 8. GCS BUCKET (Google Cloud Storage)
# ==========================================
module "vpc_gcs" {
  source = "gcs::https://www.googleapis.com/storage/v1/my-terraform-modules/vpc.zip"
}

# ==========================================
# 9. LOCAL PATH (สำหรับ development)
# ==========================================

# Relative path
module "vpc_local" {
  source = "./modules/vpc"
}

# Relative path (parent directory)
module "vpc_parent" {
  source = "../shared-modules/vpc"
}

# Absolute path (ไม่แนะนำ - ใช้ relative แทน)
module "vpc_absolute" {
  source = "/home/user/terraform-modules/vpc"
}
```

---

## Step 646: Private Module Registry

### Terraform Cloud/Enterprise Private Registry

```hcl
# ==========================================
# PRIVATE REGISTRY (Terraform Cloud)
# ==========================================

# รูปแบบ: <HOSTNAME>/<NAMESPACE>/<MODULE NAME>/<PROVIDER>

# Terraform Cloud Private Registry
module "vpc_private" {
  source  = "app.terraform.io/mycompany/vpc/aws"
  version = "~> 1.0"

  project     = "myapp"
  environment = "prod"
  cidr_block  = "10.0.0.0/16"
}

# Terraform Enterprise (self-hosted)
module "vpc_enterprise" {
  source  = "terraform.mycompany.com/platform-team/vpc/aws"
  version = "~> 2.0"

  project     = "myapp"
  environment = "prod"
}
```

### การ Configure Private Registry:

```hcl
# terraform.rc หรือ .terraformrc
credentials "app.terraform.io" {
  token = "YOUR_TERRAFORM_CLOUD_TOKEN"
}

credentials "terraform.mycompany.com" {
  token = "YOUR_TFE_TOKEN"
}
```

```bash
# หรือใช้ environment variable
export TF_TOKEN_app_terraform_io="YOUR_TFC_TOKEN"
export TF_TOKEN_terraform_mycompany_com="YOUR_TFE_TOKEN"
```

---

## Step 647: Publishing Modules to Registry

### การ publish module ไปยัง Terraform Registry

```
Requirements สำหรับ Terraform Registry:
1. GitHub repository (public)
2. Repository name: terraform-<PROVIDER>-<NAME>
3. Module structure:
   ├── main.tf
   ├── variables.tf
   ├── outputs.tf
   └── README.md
4. Semantic versioning tags (v1.0.0)
5. Required files

Optional แต่แนะนำ:
- examples/ directory
- modules/ subdirectory
- tests/ directory
```

### Module Structure ที่ถูกต้อง:

```
terraform-aws-vpc/
├── README.md           # Required - terraform-docs generate ได้
├── main.tf             # Required
├── variables.tf        # Required  
├── outputs.tf          # Required
├── versions.tf         # Recommended
├── CHANGELOG.md        # Recommended
├── LICENSE             # Recommended
├── examples/           # Recommended
│   ├── simple/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   └── outputs.tf
│   └── complete/
│       ├── main.tf
│       ├── variables.tf
│       └── outputs.tf
├── modules/            # Optional - submodules
│   └── subnets/
│       ├── main.tf
│       ├── variables.tf
│       └── outputs.tf
└── test/               # Recommended
    └── vpc_test.go
```

### Steps การ Publish:

```bash
# 1. สร้าง repository บน GitHub
# Repository name: terraform-aws-vpc

# 2. Push code
git init
git add .
git commit -m "Initial commit: VPC module v1.0.0"
git remote add origin git@github.com:myorg/terraform-aws-vpc.git
git push -u origin main

# 3. Create tag
git tag -a v1.0.0 -m "Release v1.0.0"
git push origin v1.0.0

# 4. ไปที่ registry.terraform.io
# Click "Publish" -> "Module"
# Connect GitHub และเลือก repository

# 5. Module จะพร้อมใช้งาน:
# registry.terraform.io/myorg/vpc/aws
```

---

## Step 648: Module Documentation Requirements

### การเขียน README.md ที่สมบูรณ์

```markdown
# Terraform AWS VPC Module

สร้าง VPC พร้อม public/private subnets, NAT Gateway, และ Internet Gateway

## Usage

```hcl
module "vpc" {
  source  = "myorg/vpc/aws"
  version = "~> 1.0"

  project     = "myapp"
  environment = "prod"
  cidr_block  = "10.0.0.0/16"
}
```

## Features

- สร้าง VPC พร้อม custom CIDR
- Public, Private, และ Database subnets
- NAT Gateway (single หรือ multi-AZ)
- VPC Flow Logs
- Auto-calculated subnet CIDRs

## Examples

- [Simple VPC](examples/simple)
- [Complete VPC with all options](examples/complete)

## Requirements

| Name | Version |
|------|---------|
| terraform | >= 1.3 |
| aws | >= 5.0 |

## Providers

| Name | Version |
|------|---------|
| aws | >= 5.0 |

## Inputs

| Name | Description | Type | Default | Required |
|------|-------------|------|---------|----------|
| project | ชื่อโปรเจค | string | n/a | yes |
| environment | สภาพแวดล้อม | string | n/a | yes |
| cidr_block | CIDR block สำหรับ VPC | string | n/a | yes |
| enable_nat_gateway | สร้าง NAT Gateway | bool | true | no |

## Outputs

| Name | Description |
|------|-------------|
| vpc_id | ID ของ VPC |
| private_subnet_ids | IDs ของ private subnets |
| public_subnet_ids | IDs ของ public subnets |

## License

MIT
```

---

## Step 649: Semantic Versioning for Modules

### Semantic Versioning (SemVer)

```
vMAJOR.MINOR.PATCH

MAJOR - Breaking changes (ไม่ backward compatible)
MINOR - New features (backward compatible)
PATCH - Bug fixes (backward compatible)

ตัวอย่าง:
v1.0.0 - Initial release
v1.1.0 - Added new optional feature
v1.1.1 - Fixed bug in subnet calculation
v2.0.0 - Breaking change: renamed variable
```

### CHANGELOG.md:

```markdown
# Changelog

All notable changes to this module will be documented here.

## [2.0.0] - 2024-01-15

### BREAKING CHANGES
- Renamed variable `cidr` to `cidr_block`
- Removed deprecated `enable_classiclink` variable
- `azs` default changed from `["ap-southeast-1a", "ap-southeast-1b"]` to `[]`

### Added
- Auto-calculated subnet CIDRs when `azs` is empty
- VPC Flow Logs support

### Migration Guide

```hcl
# Before (v1.x)
module "vpc" {
  source = "myorg/vpc/aws"
  version = "~> 1.0"
  cidr = "10.0.0.0/16"  # OLD variable name
}

# After (v2.x)
module "vpc" {
  source = "myorg/vpc/aws"
  version = "~> 2.0"
  cidr_block = "10.0.0.0/16"  # NEW variable name
}
```

## [1.2.0] - 2023-12-01

### Added
- Single NAT Gateway option (`single_nat_gateway`)
- Database subnet support
- Tags propagation

## [1.1.0] - 2023-10-15

### Added
- NAT Gateway support
- Custom route tables

## [1.0.0] - 2023-09-01

### Added
- Initial release
- Basic VPC with public/private subnets
```

---

## Step 650: Lock File and Module Versions

### .terraform.lock.hcl สำหรับ modules

```hcl
# .terraform.lock.hcl
# ไฟล์นี้ถูกสร้างอัตโนมัติโดย terraform init
# ควร commit ไปกับ code เพื่อ reproducible builds

provider "registry.terraform.io/hashicorp/aws" {
  version     = "5.31.0"
  constraints = "~> 5.0"
  hashes = [
    "h1:...",
    "zh:...",
  ]
}
```

```bash
# อัพเดท lock file
terraform init -upgrade

# Lock file สำหรับ specific platforms
terraform providers lock \
  -platform=linux_amd64 \
  -platform=darwin_amd64 \
  -platform=darwin_arm64

# ตรวจสอบ version ที่ lock ไว้
terraform version
terraform providers
```

### การจัดการ Module Versions ในทีม:

```hcl
# versions.tf - กำหนด version ที่ทีมทั้งหมดใช้
terraform {
  required_version = ">= 1.5.0"

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

# เพิ่ม module versions ใน README หรือ MODULES.md:
```

```markdown
## Module Versions

| Module | Source | Version |
|--------|--------|---------|
| VPC | terraform-aws-modules/vpc/aws | ~> 5.0 |
| EKS | terraform-aws-modules/eks/aws | ~> 20.0 |
| RDS | terraform-aws-modules/rds/aws | ~> 6.0 |
| S3 | terraform-aws-modules/s3-bucket/aws | ~> 4.0 |
```

---

## ตัวอย่าง Complete: ใช้ Registry Modules

```hcl
# ==========================================
# COMPLETE EXAMPLE: Production Setup
# using Registry Modules
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
    bucket = "my-terraform-state"
    key    = "prod/main.tfstate"
    region = "ap-southeast-1"
  }
}

provider "aws" {
  region = var.region
}

# =========
# Variables
# =========
variable "region" {
  default = "ap-southeast-1"
}

variable "environment" {
  default = "prod"
}

# =========
# Modules
# =========

# VPC
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "~> 5.1"

  name = "prod-vpc"
  cidr = "10.0.0.0/16"

  azs              = ["${var.region}a", "${var.region}b", "${var.region}c"]
  private_subnets  = ["10.0.1.0/24", "10.0.2.0/24", "10.0.3.0/24"]
  public_subnets   = ["10.0.101.0/24", "10.0.102.0/24", "10.0.103.0/24"]
  database_subnets = ["10.0.201.0/24", "10.0.202.0/24", "10.0.203.0/24"]

  enable_nat_gateway     = true
  single_nat_gateway     = false  # One NAT per AZ
  enable_dns_hostnames   = true
  enable_dns_support     = true

  create_database_subnet_group = true

  tags = {
    Environment = var.environment
    Terraform   = "true"
  }
}

# EKS
module "eks" {
  source  = "terraform-aws-modules/eks/aws"
  version = "~> 20.0"

  cluster_name    = "prod-cluster"
  cluster_version = "1.28"

  vpc_id     = module.vpc.vpc_id
  subnet_ids = module.vpc.private_subnets

  cluster_endpoint_public_access = true

  eks_managed_node_groups = {
    general = {
      min_size     = 2
      max_size     = 10
      desired_size = 3

      instance_types = ["m5.large"]

      labels = {
        Environment = var.environment
        NodeGroup   = "general"
      }

      tags = {
        Environment = var.environment
      }
    }

    compute = {
      min_size     = 0
      max_size     = 20
      desired_size = 2

      instance_types = ["c5.xlarge", "c5.2xlarge"]

      labels = {
        Environment = var.environment
        NodeGroup   = "compute"
        Purpose     = "compute-intensive"
      }
    }
  }

  tags = {
    Environment = var.environment
  }
}

# RDS
module "rds" {
  source  = "terraform-aws-modules/rds/aws"
  version = "~> 6.0"

  identifier = "prod-db"

  engine               = "postgres"
  engine_version       = "15"
  family               = "postgres15"
  major_engine_version = "15"
  instance_class       = "db.r6g.large"

  allocated_storage     = 100
  max_allocated_storage = 1000
  storage_encrypted     = true

  db_name  = "appdb"
  username = "dbadmin"
  port     = 5432

  db_subnet_group_name   = module.vpc.database_subnet_group_name
  vpc_security_group_ids = [aws_security_group.rds.id]

  multi_az               = true
  backup_retention_period = 14
  backup_window          = "03:00-04:00"
  maintenance_window     = "Mon:04:00-Mon:05:00"

  deletion_protection = true

  parameters = [
    {
      name  = "autovacuum"
      value = 1
    },
    {
      name  = "log_connections"
      value = "1"
    }
  ]

  tags = {
    Environment = var.environment
  }
}

# S3 for application storage
module "app_storage" {
  source  = "terraform-aws-modules/s3-bucket/aws"
  version = "~> 4.0"

  bucket = "myapp-prod-storage"

  versioning = {
    enabled = true
  }

  server_side_encryption_configuration = {
    rule = {
      apply_server_side_encryption_by_default = {
        sse_algorithm = "aws:kms"
      }
    }
  }

  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true

  tags = {
    Environment = var.environment
  }
}

# Security Group for RDS
resource "aws_security_group" "rds" {
  name_prefix = "prod-rds-"
  description = "Security group for RDS"
  vpc_id      = module.vpc.vpc_id

  ingress {
    from_port       = 5432
    to_port         = 5432
    protocol        = "tcp"
    security_groups = [module.eks.node_security_group_id]
    description     = "Allow from EKS nodes"
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = {
    Name        = "prod-rds-sg"
    Environment = var.environment
  }
}

# =========
# Outputs
# =========
output "vpc_id" {
  value = module.vpc.vpc_id
}

output "eks_cluster_endpoint" {
  value = module.eks.cluster_endpoint
}

output "rds_endpoint" {
  value     = module.rds.db_instance_endpoint
  sensitive = false
}

output "s3_bucket_id" {
  value = module.app_storage.s3_bucket_id
}
```

---

## สรุป (Summary)

### Module Source Decision Tree:

```
ต้องการ module จากไหน?
├── Terraform Registry (public module)?
│   └── source = "namespace/name/provider"
│       version = "~> X.Y"
├── Internal/Private module?
│   ├── Terraform Cloud Private Registry
│   │   └── source = "app.terraform.io/org/name/provider"
│   └── Git Repository
│       └── source = "git::https://..."
└── Local development?
    └── source = "./modules/name"
```

### Version Constraint Best Practices:

| Scenario | Constraint | Example |
|----------|-----------|---------|
| Pin exact version | `=` | `= 5.1.2` |
| Allow patch updates | `~>` patch | `~> 5.1` |
| Allow minor updates | `~>` minor | `~> 5.0` |
| Exclude known-bad | `!= ` | `>= 5.0, != 5.1.0` |
| Range | `>=, <` | `>= 5.0, < 6.0` |

---

*จบ Part 065 - Module Versioning & Registry*
