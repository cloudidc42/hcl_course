# Part 23: Terraform State Basics (ขั้นตอนที่ 221-230)

## ภาพรวม (Overview)

Terraform State คือหัวใจสำคัญของ Terraform ที่เก็บข้อมูล mapping ระหว่าง Terraform configuration กับ infrastructure จริงที่มีอยู่ใน cloud provider การเข้าใจ state อย่างลึกซึ้งเป็นสิ่งจำเป็นสำหรับการใช้งาน Terraform อย่างมืออาชีพ

---

## Step 221: What is Terraform State?

### State ทำหน้าที่อะไร?

```
[Terraform Config] <----> [Terraform State] <----> [Real Infrastructure]
    main.tf                terraform.tfstate         AWS/GCP/Azure
```

State เก็บข้อมูล 3 อย่างหลัก:

1. **Resource Mapping**: เชื่อม Terraform resource address กับ real resource ID
2. **Metadata**: ข้อมูล dependency และ providers
3. **Performance Cache**: cache ข้อมูลล่าสุดของ resources เพื่อลด API calls

### ทำไม State จึงจำเป็น?

```hcl
# สมมติเรา apply configuration นี้:
resource "aws_instance" "web" {
  ami           = "ami-xxx"
  instance_type = "t3.micro"
  tags = { Name = "web-server" }
}

# Terraform เก็บใน state ว่า:
# - aws_instance.web -> i-1234567890abcdef0
# - ami = "ami-xxx"
# - instance_type = "t3.micro"
# - etc.

# ครั้งต่อไปที่ plan:
# Terraform เปรียบเทียบ config กับ state
# ถ้า config เปลี่ยน -> propose changes
# ถ้า config เหมือนเดิม -> "No changes"
```

---

## Step 222: terraform.tfstate File Structure

### ตัวอย่าง State File จริง

```json
{
  "version": 4,
  "terraform_version": "1.6.0",
  "serial": 42,
  "lineage": "550e8400-e29b-41d4-a716-446655440000",
  "outputs": {
    "instance_id": {
      "value": "i-1234567890abcdef0",
      "type": "string"
    },
    "instance_ip": {
      "value": "10.0.1.100",
      "type": "string",
      "sensitive": false
    },
    "db_password": {
      "value": "secret123",
      "type": "string",
      "sensitive": true
    }
  },
  "resources": [
    {
      "module": "module.vpc",
      "mode": "managed",
      "type": "aws_vpc",
      "name": "main",
      "provider": "provider[\"registry.terraform.io/hashicorp/aws\"]",
      "instances": [
        {
          "schema_version": 1,
          "attributes": {
            "arn": "arn:aws:ec2:us-east-1:123456789012:vpc/vpc-12345678",
            "assign_generated_ipv6_cidr_block": false,
            "cidr_block": "10.0.0.0/16",
            "default_network_acl_id": "acl-12345678",
            "default_route_table_id": "rtb-12345678",
            "default_security_group_id": "sg-12345678",
            "dhcp_options_id": "dopt-12345678",
            "enable_classiclink": false,
            "enable_classiclink_dns_support": false,
            "enable_dns_hostnames": true,
            "enable_dns_support": true,
            "id": "vpc-12345678",
            "instance_tenancy": "default",
            "ipv6_association_id": "",
            "ipv6_cidr_block": "",
            "main_route_table_id": "rtb-12345678",
            "owner_id": "123456789012",
            "tags": {
              "Environment": "production",
              "Name": "main-vpc"
            },
            "tags_all": {
              "Environment": "production",
              "Name": "main-vpc",
              "ManagedBy": "terraform"
            }
          },
          "sensitive_attributes": [],
          "private": "eyJlMnNfc2NoZW1h..."
        }
      ]
    },
    {
      "mode": "managed",
      "type": "aws_instance",
      "name": "web",
      "provider": "provider[\"registry.terraform.io/hashicorp/aws\"]",
      "instances": [
        {
          "index_key": 0,
          "schema_version": 1,
          "attributes": {
            "ami": "ami-0c55b159cbfafe1f0",
            "arn": "arn:aws:ec2:us-east-1:123456789012:instance/i-1234567890abcdef0",
            "id": "i-1234567890abcdef0",
            "instance_type": "t3.micro",
            "private_ip": "10.0.1.100",
            "public_ip": "52.1.2.3",
            "subnet_id": "subnet-12345678",
            "vpc_security_group_ids": ["sg-12345678"],
            "tags": {
              "Name": "web-server-1"
            }
          },
          "sensitive_attributes": [],
          "dependencies": [
            "aws_subnet.public",
            "aws_security_group.web"
          ]
        }
      ]
    },
    {
      "mode": "data",
      "type": "aws_ami",
      "name": "ubuntu",
      "provider": "provider[\"registry.terraform.io/hashicorp/aws\"]",
      "instances": [
        {
          "schema_version": 0,
          "attributes": {
            "id": "ami-0c55b159cbfafe1f0",
            "name": "ubuntu/images/hvm-ssd/ubuntu-jammy-22.04-amd64-server-20230208",
            "owner_id": "099720109477"
          }
        }
      ]
    }
  ]
}
```

### อธิบาย State File Fields

```
Fields หลัก:
├── version: version ของ state file format (ปัจจุบัน = 4)
├── terraform_version: version ของ Terraform ที่สร้าง state
├── serial: เพิ่มขึ้นทุกครั้งที่ state เปลี่ยน (ใช้ detect conflicts)
├── lineage: UUID ที่ unique สำหรับ state นี้ (ไม่เปลี่ยน)
├── outputs: outputs จาก configuration
└── resources: list ของ managed/data resources

แต่ละ resource มี:
├── module: module path (ถ้าอยู่ใน module)
├── mode: "managed" (resource) หรือ "data" (data source)
├── type: resource type เช่น "aws_instance"
├── name: resource name จาก config
├── provider: provider ที่ใช้
└── instances: list ของ instances (สำหรับ count/for_each)
    ├── index_key: 0, 1, 2... (count) หรือ "key" (for_each)
    ├── schema_version: version ของ resource schema
    ├── attributes: ค่าทุก attribute ของ resource
    ├── sensitive_attributes: list ของ sensitive attribute paths
    └── dependencies: dependencies ของ resource นี้
```

---

## Step 223: What State Stores

### Resource Metadata ที่เก็บใน State

```hcl
# เมื่อ apply resource นี้:
resource "aws_db_instance" "production" {
  identifier     = "prod-db"
  engine         = "postgres"
  engine_version = "14.7"
  instance_class = "db.r5.large"
  username       = "admin"
  password       = var.db_password  # sensitive!
  
  allocated_storage = 100
  storage_encrypted = true
}

# State จะเก็บ:
# - resource ID (เช่น "prod-db")
# - endpoint (เช่น "prod-db.xxx.us-east-1.rds.amazonaws.com")
# - arn
# - port
# - password (อาจอยู่ใน state เป็น plaintext!)
# - ทุก attribute รวมถึง computed values
```

### Sensitive Values ใน State

```hcl
# ⚠️ ระวัง: sensitive values เก็บใน state เป็น plaintext

# ตัวอย่าง sensitive data ใน state:
resource "aws_iam_access_key" "user" {
  user = aws_iam_user.developer.name
}

# State เก็บ:
# "secret": "wJalrXUtnFEMI/K7MDENG/bPxRfiCYzEXAMPLEKEY"  <- plaintext!

resource "aws_db_instance" "main" {
  password = var.db_password
}
# State เก็บ password เป็น plaintext

# ✅ วิธีป้องกัน:
# 1. เข้ารหัส state file (S3 with SSE, Vault backend)
# 2. จำกัด access to state file
# 3. ใช้ sensitive = true ใน output เพื่อซ่อนใน output
output "db_password" {
  value     = aws_db_instance.main.password
  sensitive = true  # ซ่อนจาก output display แต่ยังอยู่ใน state
}
```

### State Outputs

```hcl
# outputs ใน state
output "vpc_id" {
  value = aws_vpc.main.id
}

output "db_endpoint" {
  value     = aws_db_instance.main.endpoint
  sensitive = false
}

output "secret_key" {
  value     = aws_iam_access_key.user.secret
  sensitive = true  # จะแสดงเป็น <sensitive> ใน terminal
}

# อ่าน outputs:
# $ terraform output           # แสดงทั้งหมด
# $ terraform output vpc_id    # แสดงเฉพาะ vpc_id
# $ terraform output -json     # JSON format
# $ terraform output -raw vpc_id  # raw value (ไม่มี quotes)
```

---

## Step 224: State and The Real World (Drift)

### Terraform Drift คืออะไร?

Drift เกิดขึ้นเมื่อ state ไม่ตรงกับ real infrastructure:

```
Scenario 1: Manual changes
- State: EC2 t3.micro
- Real: EC2 t3.small (someone changed via console)
- Result: Drift! Terraform will want to change back to t3.micro

Scenario 2: External deletion
- State: S3 bucket "my-bucket"  
- Real: bucket deleted manually
- Result: Drift! terraform plan will show "add" to recreate

Scenario 3: Out-of-band creation
- State: no security group rule for port 8080
- Real: rule added manually
- Result: Terraform doesn't know about it, won't affect state
```

### ตรวจสอบ Drift

```bash
# Refresh state จาก real infrastructure
terraform refresh  # deprecated แต่ยังใช้ได้

# วิธีใหม่ที่แนะนำ:
terraform apply -refresh-only

# ดู plan ที่ includes drift detection
terraform plan -refresh=true  # default

# Plan โดยไม่ refresh (เร็วกว่า แต่อาจไม่ reflect reality)
terraform plan -refresh=false
```

### Handle Drift

```bash
# Option 1: Apply เพื่อกลับไปตาม configuration
terraform apply

# Option 2: Import resource ที่สร้างนอก Terraform
terraform import aws_instance.web i-1234567890abcdef0

# Option 3: Remove resource จาก state (ถ้าไม่ต้องการจัดการอีก)
terraform state rm aws_instance.old_server

# Option 4: Update state ให้ตรงกับ real world (refresh-only)
terraform apply -refresh-only
```

---

## Step 225: Why State is Critical

### State ทำให้ Terraform ทำงานได้

```hcl
# ไม่มี state: Terraform ไม่รู้ว่า resource ไหนถูกสร้างแล้ว
# จะสร้างใหม่ทุกครั้ง หรือ error!

# มี state: Terraform รู้ว่า:
# 1. resource นี้มีอยู่แล้ว -> check for changes
# 2. resource ใหม่ -> create
# 3. resource ถูกลบจาก config -> destroy

# ตัวอย่าง: ถ้าไม่มี state
# Plan ครั้งที่ 1: + create aws_vpc.main
# Apply ครั้งที่ 1: create vpc-12345678
# Plan ครั้งที่ 2: + create aws_vpc.main (สร้างอีก! เพราะไม่รู้ว่ามีแล้ว)
# Apply ครั้งที่ 2: error! หรือสร้าง vpc ซ้ำ
```

### State สำหรับ Performance

```hcl
# Terraform ใช้ state cache เพื่อ:
# - ไม่ต้อง API call ทุก attribute ทุกครั้ง
# - รู้ attribute ที่ทำ computed (เช่น ARN, IP) โดยไม่ต้อง query

resource "aws_instance" "web" {
  ami           = data.aws_ami.ubuntu.id
  instance_type = "t3.micro"
}

# ถ้าไม่มี state:
# Terraform ต้อง query AWS ทุกครั้งว่า instance ไหนเป็น aws_instance.web
# หากมี 1000 instances ต้อง iterate ทั้งหมด!

# มี state:
# Terraform รู้ทันทีว่า aws_instance.web = i-1234567890abcdef0
```

---

## Step 226: State File Security

### ความเสี่ยงด้าน Security

```
State File อาจมีข้อมูลสำคัญ:
├── Database passwords
├── IAM access keys
├── Private keys (SSH, TLS)
├── API tokens
├── Connection strings
└── Encryption keys
```

### Best Practices ด้าน Security

```hcl
# ✅ 1. เข้ารหัส State File ที่ S3
terraform {
  backend "s3" {
    bucket  = "my-terraform-state"
    key     = "production/terraform.tfstate"
    region  = "us-east-1"
    encrypt = true  # เข้ารหัสด้วย SSE
    
    kms_key_id = "arn:aws:kms:us-east-1:123456789012:key/xxxxx"
  }
}

# ✅ 2. จำกัด Access ด้วย IAM
resource "aws_s3_bucket_policy" "state" {
  bucket = aws_s3_bucket.terraform_state.id
  
  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect    = "Deny"
        Principal = "*"
        Action    = "s3:*"
        Resource  = [
          aws_s3_bucket.terraform_state.arn,
          "${aws_s3_bucket.terraform_state.arn}/*"
        ]
        Condition = {
          Bool = {
            "aws:SecureTransport" = "false"
          }
        }
      }
    ]
  })
}

# ✅ 3. Enable Bucket Versioning
resource "aws_s3_bucket_versioning" "state" {
  bucket = aws_s3_bucket.terraform_state.id
  
  versioning_configuration {
    status = "Enabled"
  }
}

# ✅ 4. Block Public Access
resource "aws_s3_bucket_public_access_block" "state" {
  bucket = aws_s3_bucket.terraform_state.id

  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}

# ✅ 5. Never store state in git
# .gitignore:
# terraform.tfstate
# terraform.tfstate.backup
# .terraform/
```

### การจัดการ Sensitive Values

```hcl
# ❌ อย่าเก็บ secrets ใน variables โดยตรง
variable "db_password" {
  # ถ้า default มี value จะอยู่ใน state และ config
  default = "mypassword123"  # ❌ อย่าทำ!
}

# ✅ ใช้ environment variable แทน
# export TF_VAR_db_password="$(aws secretsmanager get-secret-value ...)"

# ✅ ดึงจาก Secrets Manager โดยตรง
data "aws_secretsmanager_secret_version" "db_pass" {
  secret_id = "production/database/password"
}

resource "aws_db_instance" "main" {
  password = data.aws_secretsmanager_secret_version.db_pass.secret_string
}
# ⚠️ password ยังอยู่ใน state แต่อย่างน้อยไม่อยู่ใน config files
```

---

## Step 227: Local State Limitations

### ปัญหาของ Local State

```bash
# Local state อยู่ที่ ./terraform.tfstate

# ปัญหา 1: Team collaboration
# คน A: terraform apply -> สร้าง state ใน local
# คน B: terraform apply -> ไม่มี state ของ A -> พยายามสร้างซ้ำ!

# ปัญหา 2: No locking
# คน A: terraform apply (กำลัง apply อยู่)
# คน B: terraform apply (apply พร้อมกัน -> state corruption!)

# ปัญหา 3: No backup
# ถ้า disk พัง หรือลบไฟล์ state -> lost forever

# ปัญหา 4: Security
# State อยู่ใน filesystem ทุกคนที่เข้าถึง machine ได้ก็เข้าถึง state ได้

# ✅ แก้ไขด้วย Remote Backend (ดู Part 24)
```

---

## Step 228: State Locking

### State Locking คืออะไร?

```bash
# State locking ป้องกัน concurrent operations ที่อาจทำให้ state เสีย

# เมื่อ apply หรือ plan รัน:
# 1. Terraform ขอ lock
# 2. ถ้าได้ lock -> ดำเนิน operation
# 3. เมื่อเสร็จ -> release lock

# ถ้า lock ถูก held อยู่แล้ว:
# Error: Error acquiring the state lock
# Error message: ConditionalCheckFailedException
# Lock Info:
#   ID:        12345678-...
#   Path:      s3://my-bucket/state.tfstate
#   Operation: OperationTypeApply
#   Who:       user@host
#   Version:   1.6.0
#   Created:   2024-01-15 10:30:00
```

### Force Unlock

```bash
# ⚠️ ใช้เฉพาะเมื่อแน่ใจว่า lock ค้างอยู่จาก operation ที่ตายไปแล้ว
terraform force-unlock LOCK_ID

# ตรวจสอบว่า lock ID คืออะไร ดูจาก error message
```

### DynamoDB สำหรับ State Locking

```hcl
# สร้าง DynamoDB table สำหรับ state locking
resource "aws_dynamodb_table" "terraform_lock" {
  name         = "terraform-state-lock"
  billing_mode = "PAY_PER_REQUEST"
  hash_key     = "LockID"

  attribute {
    name = "LockID"
    type = "S"
  }

  tags = {
    Name        = "Terraform State Lock Table"
    Environment = "global"
  }
}

# ใช้ใน backend config
terraform {
  backend "s3" {
    bucket         = "my-terraform-state"
    key            = "production/terraform.tfstate"
    region         = "us-east-1"
    encrypt        = true
    dynamodb_table = "terraform-state-lock"  # Lock table
  }
}
```

---

## Step 229: Reading State

### terraform show

```bash
# แสดง state ทั้งหมดในรูปแบบที่อ่านง่าย
terraform show

# ตัวอย่าง output:
# # aws_instance.web:
# resource "aws_instance" "web" {
#     ami                          = "ami-0c55b159cbfafe1f0"
#     arn                          = "arn:aws:ec2:us-east-1:123456789:instance/i-1234567890abcdef0"
#     instance_type                = "t3.micro"
#     private_ip                   = "10.0.1.100"
#     public_ip                    = "52.1.2.3"
#     ...
# }

# แสดง JSON format
terraform show -json

# แสดง plan file
terraform show myplan.tfplan
```

### terraform state list

```bash
# แสดงรายการ resources ทั้งหมดใน state
terraform state list

# ตัวอย่าง output:
# aws_instance.web
# aws_security_group.web
# aws_vpc.main
# module.networking.aws_subnet.public[0]
# module.networking.aws_subnet.public[1]
# module.networking.aws_subnet.private[0]
# aws_s3_bucket.logs
# data.aws_ami.ubuntu

# Filter ด้วย glob pattern
terraform state list 'aws_instance.*'
terraform state list 'module.networking.*'
```

### terraform state show

```bash
# แสดง details ของ resource เดียว
terraform state show aws_instance.web

# ตัวอย่าง output:
# # aws_instance.web:
# resource "aws_instance" "web" {
#     ami                          = "ami-0c55b159cbfafe1f0"
#     arn                          = "arn:aws:ec2:us-east-1:123:instance/i-xxx"
#     associate_public_ip_address  = true
#     availability_zone            = "us-east-1a"
#     disable_api_termination      = false
#     id                           = "i-1234567890abcdef0"
#     instance_state               = "running"
#     instance_type                = "t3.micro"
#     private_ip                   = "10.0.1.100"
#     public_ip                    = "52.1.2.3"
#     ...
# }

# Resource ที่ใช้ count
terraform state show 'aws_instance.web[0]'
terraform state show 'aws_instance.web[1]'

# Resource ที่ใช้ for_each
terraform state show 'aws_instance.web["prod"]'

# Resource ใน module
terraform state show 'module.vpc.aws_vpc.main'
```

---

## Step 230: State and Team Collaboration

### ปัญหาของ Local State ใน Team

```
Team Workflow ที่ผิด:
1. Alice: git clone repo
2. Alice: terraform apply -> สร้าง state.tfstate
3. Alice: git push (ถ้า push state -> ปัญหา!)
4. Bob: git pull (ถ้า state เก่าหน่อย)
5. Bob: terraform apply -> ใช้ state เก่า -> ปัญหา!

หรือถ้าไม่ commit state:
1. Alice: terraform apply (ในเครื่อง A)
2. Bob: terraform apply (ในเครื่อง B, ไม่มี state ของ Alice)
3. Bob พยายามสร้าง resources ที่ Alice สร้างไปแล้ว -> Error!
```

### ✅ Remote State สำหรับ Team

```hcl
# ทุกคนในทีมใช้ remote state เดียวกัน
terraform {
  backend "s3" {
    bucket         = "company-terraform-state"
    key            = "team-project/terraform.tfstate"
    region         = "us-east-1"
    encrypt        = true
    dynamodb_table = "terraform-state-locks"
  }
}

# Workflow ที่ถูกต้อง:
# 1. Alice: terraform apply -> lock state, apply, unlock
# 2. Bob: terraform apply (ขณะ Alice apply อยู่) -> Error: state locked
# 3. Bob รอ Alice เสร็จแล้วค่อย apply
```

### State File Backup

```bash
# terraform.tfstate.backup สร้างอัตโนมัติก่อน apply
# เก็บไว้เป็น backup ของ state version ก่อนหน้า

# ⚠️ terraform.tfstate.backup เก็บแค่ version ก่อนหน้า 1 version เท่านั้น

# ✅ ใช้ S3 versioning สำหรับ full history
resource "aws_s3_bucket_versioning" "state" {
  bucket = aws_s3_bucket.terraform_state.id
  
  versioning_configuration {
    status = "Enabled"
  }
}

# ✅ เพิ่ม lifecycle rule เก็บ versions ไว้ 90 วัน
resource "aws_s3_bucket_lifecycle_configuration" "state" {
  bucket = aws_s3_bucket.terraform_state.id

  rule {
    id     = "state-versioning"
    status = "Enabled"
    
    noncurrent_version_expiration {
      noncurrent_days = 90
    }
  }
}
```

### ⚠️ Never Edit State Manually

```bash
# ❌ อย่าแก้ไข terraform.tfstate โดยตรงด้วย text editor
# อาจทำให้ state เสีย (corrupt)

# ✅ ใช้ terraform state commands แทน:
terraform state mv    # ย้าย/rename resource
terraform state rm    # ลบออกจาก state (ไม่ destroy)
terraform state pull  # download remote state
terraform state push  # upload state

# ⚠️ ถ้าต้องแก้ state จริงๆ:
# 1. terraform state pull > current.tfstate  # backup ก่อน
# 2. แก้ไข current.tfstate
# 3. terraform state push current.tfstate  # upload กลับ
# ทำด้วยความระมัดระวัง!
```

---

## State File JSON: Deep Dive

### อ่าน State โดยตรง (สำหรับ debugging)

```bash
# ดู state แบบ raw JSON
terraform show -json | jq '.'

# หา resource เฉพาะ
terraform show -json | jq '.values.root_module.resources[] | select(.type == "aws_instance")'

# หา outputs
terraform show -json | jq '.values.outputs'

# หา resource ใน module
terraform show -json | jq '.values.root_module.child_modules[].resources[]'
```

### State Serial และ Lineage

```json
{
  "version": 4,
  "terraform_version": "1.6.0",
  "serial": 42,       // เพิ่มทุกครั้งที่ state เปลี่ยน
  "lineage": "uuid",  // ไม่เปลี่ยนตลอด lifetime ของ state
  ...
}
```

```bash
# serial ใช้ detect conflicts:
# ถ้า serial ใน state ไม่ตรงกับ serial ที่ backend expect
# Terraform จะ error: state conflict

# lineage ใช้ verify ว่าเป็น state เดียวกัน
# ถ้า lineage ต่างกัน = คนละ state
```

---

## Summary: State Concepts

```
Terraform State:
├── คือ source of truth ระหว่าง config และ real infrastructure
├── เก็บใน terraform.tfstate (JSON format)
├── ควรเก็บ remote (S3, GCS, etc.) สำหรับ team
├── ต้องมี locking ป้องกัน concurrent operations
├── มี sensitive data -> ต้องเข้ารหัสและ control access
├── อ่านได้ด้วย terraform show, terraform state list/show
└── แก้ไขผ่าน terraform state commands เท่านั้น
```

---

## แบบฝึกหัด (Exercises)

### Exercise 1: Inspect State

```bash
# 1. Deploy simple infrastructure
terraform apply

# 2. ตรวจสอบ state
terraform state list
terraform state show <resource>
terraform show -json | jq '.values.outputs'

# 3. แก้ไข resource ด้วย AWS Console
# 4. Run terraform plan เพื่อดู drift
```

### Exercise 2: State Backup

```bash
# 1. Backup state ก่อน operation
cp terraform.tfstate terraform.tfstate.manual-backup-$(date +%Y%m%d)

# 2. ทดลอง apply

# 3. ดู terraform.tfstate.backup
cat terraform.tfstate.backup
```

---

## Checklist

- [ ] เข้าใจว่า state คืออะไรและทำหน้าที่อะไร
- [ ] รู้โครงสร้าง JSON ของ state file
- [ ] เข้าใจ sensitive values ใน state
- [ ] รู้ปัญหาของ local state
- [ ] เข้าใจ state locking
- [ ] สามารถใช้ terraform show, state list, state show ได้
- [ ] เข้าใจเรื่อง drift detection
- [ ] รู้ best practices ด้าน security
- [ ] ไม่แก้ไข state file โดยตรง
