# Part 016: Introduction to Terraform (บทนำ Terraform)
## Steps 151-160: ทำความรู้จัก Terraform และระบบนิเวศ

---

## บทนำ (Introduction)

Terraform เป็นเครื่องมือ Infrastructure as Code (IaC) แบบ open-source พัฒนาโดย HashiCorp ที่ช่วยให้เราสามารถกำหนดและจัดการ infrastructure ผ่าน code แทนการ click ผ่าน UI หรือใช้ scripts แบบ imperative

---

## Step 151: What is Terraform?

### นิยามและประวัติ

**Terraform** เปิดตัวในปี 2014 โดย Mitchell Hashimoto และ HashiCorp ชื่อมาจาก "terraform" หมายความว่าการแปลงพื้นผิวดาวเคราะห์ให้เหมาะสมกับการอยู่อาศัย ซึ่งสื่อถึงการ "แปลง" cloud resources ให้กลายเป็น infrastructure

### Key Concepts

```
Infrastructure as Code (IaC) คืออะไร?
- แทนที่จะ click ผ่าน AWS Console
- เขียน code ที่อธิบาย infrastructure ที่ต้องการ
- Terraform อ่าน code แล้ว create/update/delete resources

ประโยชน์:
✅ Reproducible - deploy เหมือนกันทุกครั้ง
✅ Version Control - track การเปลี่ยนแปลงใน git
✅ Automation - เป็นส่วนหนึ่งของ CI/CD pipeline
✅ Documentation - code คือ documentation
✅ Collaboration - review ด้วย pull requests
✅ Consistency - dev/staging/prod ใช้ code เดียวกัน
```

### Timeline ของ Terraform

```
2014 - Terraform 0.1 เปิดตัว
2017 - Terraform 0.10 (แยก providers ออกมา)
2019 - Terraform 0.12 (HCL2, type system ปรับปรุง)
2020 - Terraform 0.13 (module registry ปรับปรุง)
2021 - Terraform 1.0 (stable API, backward compat)
2023 - Terraform 1.5+ (improved checks, imports)
2023 - HashiCorp เปลี่ยน license เป็น BSL 1.1
2023 - OpenTofu fork ถูกสร้างขึ้น (MPL 2.0 license)
```

---

## Step 152: Terraform vs Other IaC Tools

### เปรียบเทียบ IaC Tools

| Feature | Terraform | CloudFormation | Pulumi | Ansible | Chef/Puppet |
|---------|-----------|----------------|--------|---------|-------------|
| **Type** | Declarative | Declarative | Imperative/Declarative | Procedural | Declarative |
| **Language** | HCL | JSON/YAML | Python/TS/Go | YAML | Ruby DSL |
| **State** | Remote state | Managed by AWS | Remote state | Stateless | Stateless |
| **Multi-Cloud** | ✅ ทุก cloud | ❌ AWS only | ✅ | ✅ | ✅ |
| **Providers** | 1000+ | AWS only | 60+ | Modules | Cookbooks |
| **Learning Curve** | Medium | Low-Medium | High | Low | High |
| **Dry Run** | terraform plan | Change sets | Preview | Check mode | --noop |
| **Rollback** | Manual/partial | Automatic | Manual | Manual | Manual |
| **Cost** | Free/Commercial | Free | Free/Commercial | Free/Commercial | Free/Commercial |

### Terraform vs CloudFormation (AWS)

```
Terraform ดีกว่า CloudFormation เมื่อ:
✅ Multi-cloud (AWS + GCP + Azure ในที่เดียว)
✅ Cleaner syntax (HCL vs JSON/YAML)
✅ Better state management
✅ Community modules (registry)
✅ Preview changes (plan)
✅ Import existing resources

CloudFormation ดีกว่า Terraform เมื่อ:
✅ AWS-only infrastructure
✅ Automatic rollback
✅ ไม่ต้องจัดการ state
✅ Native AWS integration (StackSets, Drift detection)
✅ ไม่ต้องติดตั้งอะไร (มีใน AWS Console)
```

### Terraform vs Pulumi

```
Terraform ดีกว่า Pulumi เมื่อ:
✅ ทีมไม่ต้องการเรียน programming language
✅ Ecosystem ใหญ่กว่า
✅ Documentation ครบกว่า
✅ Community support มากกว่า

Pulumi ดีกว่า Terraform เมื่อ:
✅ ต้องการ logic ซับซ้อน (loops, conditions ใน code)
✅ ทีมเป็น developers ที่ถนัด Python/TypeScript
✅ Testing โดยใช้ unit tests ของ programming language
```

---

## Step 153: Terraform Use Cases

### Use Cases หลัก

```
1. Cloud Infrastructure Provisioning
   - VPC, subnets, security groups
   - EC2, ECS, EKS clusters
   - RDS, DynamoDB, ElastiCache
   - S3, CloudFront

2. Multi-Cloud Management
   - AWS + GCP + Azure ในโปรเจคเดียวกัน
   - Consistent tooling ทุก cloud

3. SaaS and Third-party Services
   - GitHub repositories, teams
   - Datadog monitors, dashboards
   - PagerDuty services
   - Cloudflare DNS records

4. Kubernetes
   - EKS, GKE, AKS clusters
   - Helm chart deployments
   - Kubernetes resources

5. CI/CD Infrastructure
   - GitHub Actions runners
   - Jenkins infrastructure
   - Artifact repositories

6. Security and Compliance
   - IAM policies, roles
   - Security groups, NACLs
   - KMS keys
   - Compliance enforcement
```

### Real-World Use Case Example

```hcl
# ตัวอย่าง: Complete web application infrastructure

# 1. Network layer
module "vpc" {
  source = "terraform-aws-modules/vpc/aws"
  version = "~> 5.0"
  
  name = "myapp-prod"
  cidr = "10.0.0.0/16"
  # ...
}

# 2. Compute layer
module "ecs" {
  source = "terraform-aws-modules/ecs/aws"
  version = "~> 5.0"
  
  cluster_name = "myapp-prod"
  # ...
}

# 3. Database layer
module "rds" {
  source  = "terraform-aws-modules/rds/aws"
  version = "~> 6.0"
  
  identifier = "myapp-prod-mysql"
  # ...
}

# 4. CDN layer
resource "aws_cloudfront_distribution" "main" {
  # ...
}

# 5. DNS
resource "aws_route53_record" "app" {
  # ...
}

# 6. Monitoring
resource "datadog_monitor" "app_errors" {
  # ...
}
```

---

## Step 154: Terraform Architecture

### Core Components

```
┌─────────────────────────────────────────────────┐
│              Terraform Core                       │
│                                                   │
│  ┌───────────┐  ┌───────────┐  ┌─────────────┐  │
│  │   HCL     │  │  State    │  │   Planner   │  │
│  │  Parser   │  │  Manager  │  │   Engine    │  │
│  └───────────┘  └───────────┘  └─────────────┘  │
│                                                   │
│  ┌─────────────────────────────────────────────┐ │
│  │            Plugin System (RPC)              │ │
│  └─────────────────────────────────────────────┘ │
└──────────────────────────┬──────────────────────┘
                           │ gRPC
           ┌───────────────┼──────────────────┐
           │               │                  │
    ┌──────▼──────┐ ┌──────▼──────┐  ┌───────▼─────┐
    │  AWS        │ │  Azure      │  │  GCP        │
    │  Provider   │ │  Provider   │  │  Provider   │
    └──────┬──────┘ └──────┬──────┘  └──────┬──────┘
           │               │                │
    ┌──────▼──────┐ ┌──────▼──────┐  ┌──────▼──────┐
    │  AWS API    │ │  Azure API  │  │  GCP API    │
    └─────────────┘ └─────────────┘  └─────────────┘
```

### State File

```
Terraform State คืออะไร?
- ไฟล์ JSON ที่ track ว่า resources ที่ Terraform จัดการมีอะไรบ้าง
- เปรียบเทียบ desired state (code) กับ actual state (state file)
- Default: terraform.tfstate ใน local directory
- Production: เก็บใน remote backend (S3, GCS, Azure Blob, Terraform Cloud)

ทำไม State สำคัญ?
- ช่วย map resource ใน code กับ resource จริงๆ ใน cloud
- Track resource metadata
- เพิ่ม performance (ไม่ต้อง query API ทุก attribute)
- Dependency tracking
```

### State Backends

```hcl
# Local backend (default - ไม่แนะนำสำหรับ team)
terraform {
  backend "local" {
    path = "terraform.tfstate"
  }
}

# S3 backend (แนะนำสำหรับ AWS)
terraform {
  backend "s3" {
    bucket         = "my-terraform-state"
    key            = "prod/terraform.tfstate"
    region         = "ap-southeast-1"
    dynamodb_table = "terraform-state-lock"
    encrypt        = true
  }
}

# Terraform Cloud (managed service)
terraform {
  cloud {
    organization = "my-org"
    workspaces {
      name = "my-workspace"
    }
  }
}
```

---

## Step 155: How Terraform Works Internally

### Workflow ภายใน

```
┌─────────────────────────────────────────────────────────────┐
│                    terraform apply flow                      │
│                                                              │
│  1. Parse HCL files                                          │
│     └─> Build resource graph                                 │
│                                                              │
│  2. Load state file                                          │
│     └─> Get current infrastructure state                     │
│                                                              │
│  3. Query providers                                          │
│     └─> Refresh resource attributes (unless -refresh=false) │
│                                                              │
│  4. Compute diff                                             │
│     └─> Desired state (code) vs Current state               │
│                                                              │
│  5. Create execution plan                                    │
│     └─> Determine order of operations (DAG)                  │
│                                                              │
│  6. Apply changes                                            │
│     ├─> Create resources
│     ├─> Update resources
│     └─> Delete resources                                     │
│                                                              │
│  7. Update state file                                        │
│     └─> Record new state                                     │
└─────────────────────────────────────────────────────────────┘
```

### Dependency Graph (DAG)

```
Terraform สร้าง Directed Acyclic Graph (DAG) ของ resources
เพื่อกำหนดลำดับการ create/destroy

ตัวอย่าง:
                    aws_vpc
                   /       \
          aws_subnet      aws_igw
               |               |
        aws_instance    aws_route_table
               |
        aws_eip

→ Terraform จะ create ตามลำดับ:
  1. aws_vpc (ไม่มี dependency)
  2. aws_subnet และ aws_igw (parallel - ทั้งสองขึ้นกับ vpc)
  3. aws_instance และ aws_route_table (parallel)
  4. aws_eip (ขึ้นกับ instance)
```

---

## Step 156: Terraform's Declarative Model

### Declarative vs Imperative

```bash
# Imperative (Bash script) - บอกวิธีทำ
#!/bin/bash
# ต้องตรวจสอบทุกอย่างเอง
if aws ec2 describe-vpcs --filters "Name=tag:Name,Values=my-vpc" | grep -q "VpcId"; then
  echo "VPC already exists"
else
  aws ec2 create-vpc --cidr-block 10.0.0.0/16
  aws ec2 create-tags --resources $VPC_ID --tags Key=Name,Value=my-vpc
fi

# ปัญหา:
# - ต้องจัดการ state เอง
# - ต้องเขียน idempotency logic เอง
# - ยากที่จะ update/delete
# - Error handling ซับซ้อน
```

```hcl
# Declarative (Terraform) - บอกสิ่งที่ต้องการ
resource "aws_vpc" "main" {
  cidr_block = "10.0.0.0/16"
  
  tags = {
    Name = "my-vpc"
  }
}

# ประโยชน์:
# ✅ Terraform จัดการ state เอง
# ✅ Idempotent โดย default
# ✅ Update ง่าย - แค่เปลี่ยน code
# ✅ Delete ง่าย - แค่ลบ resource block
```

---

## Step 157: Idempotency in Terraform

### Idempotency คืออะไร?

```
Idempotency = การ apply หลายครั้งให้ผลลัพธ์เหมือนกัน

ตัวอย่าง:
- Run terraform apply ครั้งแรก: creates VPC
- Run terraform apply ครั้งที่สอง (ไม่มีการเปลี่ยนแปลง): ไม่ทำอะไร
- เพิ่ม resource ใน code แล้ว apply: creates only new resource
- ลบ resource จาก code แล้ว apply: destroys only that resource
```

```bash
# ตัวอย่าง idempotent run
$ terraform apply

# ครั้งแรก:
# aws_vpc.main: Creating...
# aws_vpc.main: Creation complete after 2s [id=vpc-0abc123]
# Apply complete! Resources: 1 added, 0 changed, 0 destroyed.

# ครั้งที่สอง (ไม่มีการเปลี่ยน):
# No changes. Your infrastructure matches the configuration.
# Apply complete! Resources: 0 added, 0 changed, 0 destroyed.
```

### Drift Detection

```bash
# Refresh state จาก actual infrastructure
terraform refresh  # หรือ
terraform apply -refresh-only

# ดู drift ระหว่าง state และ actual
terraform plan -refresh=true
```

---

## Step 158: Terraform Versions and HCL 2

### HCL Version 2 Features (Terraform 0.12+)

```hcl
# HCL 1 (เก่า) - ใน Terraform 0.11
variable "cidr_blocks" {
  default = ["10.0.0.0/24", "10.0.1.0/24"]
}
resource "aws_subnet" "example" {
  count      = "${length(var.cidr_blocks)}"  # ต้องใช้ ${}
  cidr_block = "${element(var.cidr_blocks, count.index)}"
}

# HCL 2 (ใหม่) - Terraform 0.12+
variable "cidr_blocks" {
  default = ["10.0.0.0/24", "10.0.1.0/24"]
}
resource "aws_subnet" "example" {
  count      = length(var.cidr_blocks)    # ไม่ต้องใช้ ${}
  cidr_block = var.cidr_blocks[count.index]  # cleaner syntax
}
```

### HCL 2 New Features

```hcl
# For expressions
locals {
  # ไม่มีใน HCL 1
  subnet_map = {
    for i, cidr in var.cidrs : "subnet-${i}" => cidr
  }
  
  filtered = [
    for subnet in var.subnets : subnet
    if subnet.public
  ]
}

# Dynamic blocks
resource "aws_security_group" "main" {
  dynamic "ingress" {
    for_each = var.ingress_rules
    content {
      from_port   = ingress.value.from_port
      to_port     = ingress.value.to_port
      protocol    = ingress.value.protocol
      cidr_blocks = ingress.value.cidr_blocks
    }
  }
}

# String templates
locals {
  message = "Hello, ${var.name}! You are ${var.age} years old."
  multiline = <<-EOT
    This is a
    multiline string
    with ${var.interpolation}
  EOT
}

# Type system
variable "complex_var" {
  type = object({
    name    = string
    count   = number
    enabled = bool
    tags    = map(string)
    cidrs   = list(string)
  })
}
```

---

## Step 159: Terraform Community vs Enterprise vs Cloud

### เปรียบเทียบ Editions

| Feature | Community (OSS) | Terraform Cloud (Free) | Terraform Cloud (Plus/Business) | TF Enterprise |
|---------|-----------------|------------------------|----------------------------------|---------------|
| **Core Features** | ✅ | ✅ | ✅ | ✅ |
| **Remote State** | Self-managed | Managed | Managed | Managed |
| **Remote Execution** | ❌ | ✅ (limited) | ✅ | ✅ |
| **Team Mgmt** | ❌ | ❌ | ✅ | ✅ |
| **SSO/SAML** | ❌ | ❌ | ✅ | ✅ |
| **Audit Logging** | ❌ | ❌ | ✅ | ✅ |
| **Self-hosted** | ✅ | ❌ | ❌ | ✅ |
| **Price** | Free | Free | $20+/user/month | Enterprise |

### Provider Ecosystem

```
Terraform Registry: registry.terraform.io

Provider Categories:
1. Official (by HashiCorp)
   - aws, azure, google, kubernetes
   
2. Partner (by technology companies)
   - datadog, mongodb, pagerduty
   
3. Community (by community)
   - 1000+ providers

จำนวน Providers ทั้งหมด: 3000+ (2024)
```

---

## Step 160: Terraform Workflow Deep Dive

### Complete Workflow

```bash
# Step 1: Initialize
terraform init
# Downloads providers, modules, sets up backend

# Step 2: Plan
terraform plan
# Shows what will be created/updated/destroyed
# ไม่ทำการเปลี่ยนแปลงจริง

# Step 3: Apply
terraform apply
# Shows plan อีกครั้ง แล้วถาม yes/no
terraform apply -auto-approve  # ข้าม confirmation (CI/CD)

# Step 4: Destroy (เมื่อต้องการ)
terraform destroy
terraform destroy -auto-approve  # CI/CD

# Additional commands:
terraform fmt          # Format code
terraform validate     # Validate syntax
terraform output       # Show outputs
terraform show         # Show state
terraform state list   # List resources in state
terraform state show   # Show specific resource
terraform import       # Import existing resources
terraform taint        # Mark resource for recreation
terraform refresh      # Update state from real infrastructure
terraform workspace    # Manage workspaces
```

### Plan Output ตัวอย่าง

```bash
$ terraform plan

Terraform will perform the following actions:

  # aws_instance.web will be created
  + resource "aws_instance" "web" {
      + ami                                  = "ami-0c55b159cbfafe1f0"
      + arn                                  = (known after apply)
      + instance_type                        = "t3.micro"
      + private_ip                           = (known after apply)
      + public_ip                            = (known after apply)
      
      + tags = {
          + "Environment" = "prod"
          + "Name"        = "web-server"
        }
    }

  # aws_instance.web will be destroyed (old)
  - resource "aws_instance" "old" {
      - id            = "i-0123456789abcdef0" -> null
      - instance_type = "t2.micro" -> null
    }

  # aws_security_group.web will be updated in-place
  ~ resource "aws_security_group" "web" {
      ~ description = "Old description" -> "New description"
        id          = "sg-0abc123def"
        # (other unchanged attributes hidden)
    }

Plan: 1 to add, 1 to change, 1 to destroy.
```

---

## Real-World Adoption Story

### การนำ Terraform มาใช้ในองค์กร

```
Phase 1: ทดลองใช้ (2-4 สัปดาห์)
- เริ่มจาก non-critical infrastructure
- เรียนรู้ workflow พื้นฐาน
- สร้าง team conventions

Phase 2: Migrate infrastructure (2-3 เดือน)
- ค่อยๆ import existing resources
- สร้าง modules สำหรับ common patterns
- ตั้ง remote state

Phase 3: Standardize (3-6 เดือน)
- สร้าง module library
- enforce ด้วย policies (Sentinel/OPA)
- CI/CD integration

Phase 4: Scale (6+ เดือน)
- Multi-account strategy
- Module versioning
- Team training
```

### Cost of Terraform vs Manual

```
Manual Infrastructure Management:
- Engineer time: 4 hours per change
- Error rate: 15% (human errors)
- Audit: manual, inconsistent
- Rollback: difficult, often impossible
- Documentation: outdated quickly

Terraform:
- Engineer time: 30 min initial setup, 15 min changes
- Error rate: < 1% (validation, plan review)
- Audit: git history = full audit trail
- Rollback: git revert, apply previous version
- Documentation: code IS documentation
```

---

## ASCII Architecture Diagram

### Terraform Ecosystem

```
                          +------------------+
                          |   Developer      |
                          +--------+---------+
                                   |
                                   | Write HCL code
                                   v
                    +--------------+---------------+
                    |         Terraform            |
                    |                              |
                    |  +---------+  +-----------+  |
                    |  | Parser  |  |  Planner  |  |
                    |  +---------+  +-----------+  |
                    |       |             |         |
                    |  +---------+  +-----------+  |
                    |  | State   |  |  Executor |  |
                    |  | Manager |  |           |  |
                    |  +---------+  +-----------+  |
                    +--------+---------------------+
                             |
             +---------------+----------------+
             |               |                |
    +--------+------+ +------+------+ +-------+------+
    |   S3 Backend  | | TF Cloud   | | Azure Blob   |
    | (state file)  | |  Backend   | | (state file) |
    +---------------+ +------------+ +--------------+
             |
    +--------+-----------------------------------------------+
    |              Provider Plugins                          |
    |                                                        |
    | +-----------+ +-----------+ +-----------+ +--------+  |
    | | hashicorp | | hashicorp | | hashicorp | |  3rd   |  |
    | |    aws    | |   azure   | |  google   | | party  |  |
    | +-----------+ +-----------+ +-----------+ +--------+  |
    +--------+-------+---------------+----------------------+
             |       |               |
    +--------+  +----+------+  +-----+-----+
    | AWS API|  | Azure API |  |  GCP API  |
    +--------+  +-----------+  +-----------+
```

---

## สรุป (Summary)

### Key Takeaways

1. **Terraform เป็น IaC tool** ที่ declarative และ multi-cloud
2. **Architecture** ประกอบด้วย Core + Providers + State
3. **Workflow**: init → plan → apply → destroy
4. **Idempotent** - apply หลายครั้งให้ผลเหมือนกัน
5. **HCL 2** มี features ที่ทรงพลัง (for expressions, dynamic blocks)

### ✅ ข้อดีของ Terraform

- Open source + community ใหญ่
- Multi-cloud support
- Provider ecosystem ครบ
- Excellent planning (dry run)
- Good state management

### ⚠️ ข้อเสีย/ข้อควรระวัง

- State management ต้องดูแล
- Secrets management ต้องทำเอง
- Provider upgrades อาจ break
- Team coordination บน state

### 💡 Pro Tips

1. เริ่มจาก terraform.io/learn สำหรับ guided tutorials
2. ใช้ Terraform Cloud free tier สำหรับ remote state
3. Study existing modules ใน terraform-aws-modules organization
4. Join Terraform community ใน GitHub, Slack, Reddit

---

*จบ Part 016 - Introduction to Terraform*
