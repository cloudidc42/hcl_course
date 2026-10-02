# Part 071: Moved Blocks & Refactoring (ขั้นตอนที่ 701-710)

## บทนำ (Introduction)

การ Refactoring โครงสร้าง Terraform เป็นสิ่งจำเป็นในการพัฒนา Infrastructure as Code ระยะยาว
Terraform 1.1+ มี `moved` block ที่ช่วยให้สามารถเปลี่ยนชื่อ resource หรือย้าย resource ระหว่าง module
ได้โดยไม่ต้องทำลาย resource เดิมและสร้างใหม่

---

## ขั้นตอนที่ 701: ทำไมถึงต้องการ moved blocks?

### ปัญหาก่อนมี moved blocks

ก่อน Terraform 1.1 ถ้าต้องการเปลี่ยนชื่อ resource จาก `aws_instance.web` เป็น `aws_instance.web_server`:

```hcl
# เดิม
resource "aws_instance" "web" {
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t3.micro"
}

# หากเปลี่ยนเป็น
resource "aws_instance" "web_server" {
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t3.micro"
}
```

Terraform จะทำการ:
1. **Destroy** `aws_instance.web` (ลบ EC2 instance เก่า)
2. **Create** `aws_instance.web_server` (สร้าง EC2 instance ใหม่)

นี่คือ downtime ที่ไม่จำเป็น!

### วิธีแก้ปัญหาเดิม (Manual state move)

```bash
# วิธีเดิมก่อน Terraform 1.1
terraform state mv aws_instance.web aws_instance.web_server
```

ปัญหาของวิธีนี้:
- ต้องทำ manual step ทุกครั้ง
- ไม่มีใน version control
- เพื่อนร่วมทีมไม่รู้ต้องทำ step นี้
- ใน CI/CD pipeline ทำยาก

### วิธีแก้ปัญหาใหม่ด้วย moved block

```hcl
# main.tf
resource "aws_instance" "web_server" {  # ชื่อใหม่
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t3.micro"
}

# moved.tf (หรือในไฟล์เดียวกัน)
moved {
  from = aws_instance.web       # ชื่อเดิม
  to   = aws_instance.web_server # ชื่อใหม่
}
```

ตอนนี้ `terraform plan` จะแสดงว่า:
```
# aws_instance.web has moved to aws_instance.web_server
```

ไม่มีการ destroy และ create ใหม่!

---

## ขั้นตอนที่ 702: Moved Block Syntax

### โครงสร้างพื้นฐาน

```hcl
moved {
  from = <source_address>
  to   = <destination_address>
}
```

### กฎของ moved block

1. `from` และ `to` ต้องเป็น resource address ที่ถูกต้อง
2. Resource ที่ `to` ชี้ไปต้องมีอยู่ใน configuration ปัจจุบัน
3. Resource ที่ `from` ชี้ไปต้องมีอยู่ใน state (หรือเคยมี)
4. moved block ไม่มี lifecycle, provider, หรือ depends_on

### ตัวอย่าง Syntax ต่างๆ

```hcl
# การย้าย resource ธรรมดา
moved {
  from = aws_s3_bucket.old_name
  to   = aws_s3_bucket.new_name
}

# การย้าย resource ที่มี count
moved {
  from = aws_instance.servers[0]
  to   = aws_instance.web_servers[0]
}

# การย้าย resource ที่ใช้ for_each (key เป็น string)
moved {
  from = aws_subnet.old["us-east-1a"]
  to   = aws_subnet.new["us-east-1a"]
}

# การย้าย resource เข้า module
moved {
  from = aws_vpc.main
  to   = module.networking.aws_vpc.main
}

# การย้าย resource ออกจาก module
moved {
  from = module.networking.aws_vpc.main
  to   = aws_vpc.main
}

# การย้าย resource ระหว่าง module
moved {
  from = module.old_module.aws_vpc.main
  to   = module.new_module.aws_vpc.main
}

# การย้าย module ทั้งหมด
moved {
  from = module.old_networking
  to   = module.new_networking
}
```

---

## ขั้นตอนที่ 703: การย้าย Resource ไปยังชื่อใหม่

### Scenario: เปลี่ยนชื่อ S3 Bucket resource

**Before (State เดิม):**
```hcl
# main.tf - version เดิม
resource "aws_s3_bucket" "data" {
  bucket = "my-company-data-bucket"

  tags = {
    Name        = "Data Bucket"
    Environment = "production"
  }
}

resource "aws_s3_bucket_versioning" "data" {
  bucket = aws_s3_bucket.data.id

  versioning_configuration {
    status = "Enabled"
  }
}
```

**After (Config ใหม่พร้อม moved block):**
```hcl
# main.tf - version ใหม่
resource "aws_s3_bucket" "company_data_storage" {  # ชื่อใหม่ที่ชัดเจนกว่า
  bucket = "my-company-data-bucket"

  tags = {
    Name        = "Data Bucket"
    Environment = "production"
  }
}

resource "aws_s3_bucket_versioning" "company_data_storage" {  # ต้องเปลี่ยนชื่อนี้ด้วย
  bucket = aws_s3_bucket.company_data_storage.id  # อ้างอิงชื่อใหม่

  versioning_configuration {
    status = "Enabled"
  }
}

# moved blocks
moved {
  from = aws_s3_bucket.data
  to   = aws_s3_bucket.company_data_storage
}

moved {
  from = aws_s3_bucket_versioning.data
  to   = aws_s3_bucket_versioning.company_data_storage
}
```

**ตรวจสอบด้วย terraform plan:**
```bash
$ terraform plan

Terraform will perform the following actions:

  # aws_s3_bucket.data has moved to aws_s3_bucket.company_data_storage
    resource "aws_s3_bucket" "company_data_storage" {
        id     = "my-company-data-bucket"
        ...
    }

  # aws_s3_bucket_versioning.data has moved to aws_s3_bucket_versioning.company_data_storage
    resource "aws_s3_bucket_versioning" "company_data_storage" {
        id     = "my-company-data-bucket"
        ...
    }

Plan: 0 to add, 0 to change, 0 to destroy.
```

---

## ขั้นตอนที่ 704: การย้าย Resource เข้า Module

### Scenario: นำ networking resources เข้า module

**Before (flat structure):**
```hcl
# main.tf - โครงสร้างเดิมที่ไม่มี module
resource "aws_vpc" "main" {
  cidr_block           = "10.0.0.0/16"
  enable_dns_hostnames = true
  enable_dns_support   = true

  tags = {
    Name = "main-vpc"
  }
}

resource "aws_subnet" "public" {
  vpc_id            = aws_vpc.main.id
  cidr_block        = "10.0.1.0/24"
  availability_zone = "us-east-1a"

  tags = {
    Name = "public-subnet"
    Type = "public"
  }
}

resource "aws_subnet" "private" {
  vpc_id            = aws_vpc.main.id
  cidr_block        = "10.0.2.0/24"
  availability_zone = "us-east-1b"

  tags = {
    Name = "private-subnet"
    Type = "private"
  }
}

resource "aws_internet_gateway" "main" {
  vpc_id = aws_vpc.main.id

  tags = {
    Name = "main-igw"
  }
}
```

**Step 1: สร้าง module structure:**
```
modules/
  networking/
    main.tf
    variables.tf
    outputs.tf
```

```hcl
# modules/networking/variables.tf
variable "vpc_cidr" {
  type        = string
  description = "CIDR block for VPC"
}

variable "public_subnet_cidr" {
  type        = string
  description = "CIDR for public subnet"
}

variable "private_subnet_cidr" {
  type        = string
  description = "CIDR for private subnet"
}

variable "availability_zone_public" {
  type    = string
  default = "us-east-1a"
}

variable "availability_zone_private" {
  type    = string
  default = "us-east-1b"
}

variable "tags" {
  type    = map(string)
  default = {}
}
```

```hcl
# modules/networking/main.tf
resource "aws_vpc" "main" {
  cidr_block           = var.vpc_cidr
  enable_dns_hostnames = true
  enable_dns_support   = true

  tags = merge(var.tags, {
    Name = "main-vpc"
  })
}

resource "aws_subnet" "public" {
  vpc_id            = aws_vpc.main.id
  cidr_block        = var.public_subnet_cidr
  availability_zone = var.availability_zone_public

  tags = merge(var.tags, {
    Name = "public-subnet"
    Type = "public"
  })
}

resource "aws_subnet" "private" {
  vpc_id            = aws_vpc.main.id
  cidr_block        = var.private_subnet_cidr
  availability_zone = var.availability_zone_private

  tags = merge(var.tags, {
    Name = "private-subnet"
    Type = "private"
  })
}

resource "aws_internet_gateway" "main" {
  vpc_id = aws_vpc.main.id

  tags = merge(var.tags, {
    Name = "main-igw"
  })
}
```

```hcl
# modules/networking/outputs.tf
output "vpc_id" {
  value = aws_vpc.main.id
}

output "public_subnet_id" {
  value = aws_subnet.public.id
}

output "private_subnet_id" {
  value = aws_subnet.private.id
}
```

**Step 2: อัพเดท root main.tf:**
```hcl
# main.tf - version ใหม่ที่ใช้ module
module "networking" {
  source = "./modules/networking"

  vpc_cidr             = "10.0.0.0/16"
  public_subnet_cidr   = "10.0.1.0/24"
  private_subnet_cidr  = "10.0.2.0/24"
}

# moved blocks สำหรับย้าย resources เข้า module
moved {
  from = aws_vpc.main
  to   = module.networking.aws_vpc.main
}

moved {
  from = aws_subnet.public
  to   = module.networking.aws_subnet.public
}

moved {
  from = aws_subnet.private
  to   = module.networking.aws_subnet.private
}

moved {
  from = aws_internet_gateway.main
  to   = module.networking.aws_internet_gateway.main
}
```

**ผลลัพธ์ terraform plan:**
```
# aws_internet_gateway.main has moved to module.networking.aws_internet_gateway.main
# aws_subnet.private has moved to module.networking.aws_subnet.private
# aws_subnet.public has moved to module.networking.aws_subnet.public
# aws_vpc.main has moved to module.networking.aws_vpc.main

Plan: 0 to add, 0 to change, 0 to destroy.
```

---

## ขั้นตอนที่ 705: การย้าย Resource ออกจาก Module

### Scenario: นำ resource ออกจาก module มาอยู่ root level

```hcl
# ก่อน - resource อยู่ใน module
module "security" {
  source = "./modules/security"
}

# หลัง - ย้าย IAM role ออกมาที่ root
resource "aws_iam_role" "app_role" {
  name = "app-execution-role"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Action = "sts:AssumeRole"
      Effect = "Allow"
      Principal = {
        Service = "ec2.amazonaws.com"
      }
    }]
  })
}

# moved block - ย้ายออกจาก module
moved {
  from = module.security.aws_iam_role.app_role
  to   = aws_iam_role.app_role
}
```

---

## ขั้นตอนที่ 706: การย้าย Resource ระหว่าง Modules

### Scenario: ย้าย database resource จาก module หนึ่งไปอีก module หนึ่ง

```hcl
# โครงสร้างเดิม
module "application" {
  source = "./modules/application"
  # มี RDS instance อยู่ภายใน
}

# โครงสร้างใหม่ - แยก database ออกมาเป็น module ของตัวเอง
module "application" {
  source = "./modules/application"
  # ลบ RDS ออก
}

module "database" {
  source = "./modules/database"
  # มี RDS instance ใหม่
}

# moved block
moved {
  from = module.application.aws_db_instance.main
  to   = module.database.aws_db_instance.main
}

moved {
  from = module.application.aws_db_subnet_group.main
  to   = module.database.aws_db_subnet_group.main
}

moved {
  from = module.application.aws_security_group.rds
  to   = module.database.aws_security_group.rds
}
```

### ตัวอย่างที่ซับซ้อนกว่า - ย้ายพร้อมเปลี่ยน module instance

```hcl
# เดิม: module เดี่ยว
module "app_v1" {
  source = "./modules/app"
}

# ใหม่: module ที่มี for_each
module "app" {
  source   = "./modules/app"
  for_each = toset(["blue", "green"])
}

# moved blocks
moved {
  from = module.app_v1.aws_instance.server
  to   = module.app["blue"].aws_instance.server
}
```

---

## ขั้นตอนที่ 707: Chaining Moved Blocks

### การ chain หลาย moved blocks

บางครั้งต้องทำหลาย refactoring steps ต่อกัน:

```hcl
# Step 1 (เคยทำไปแล้ว - still needed in config):
moved {
  from = aws_instance.server
  to   = aws_instance.web_server
}

# Step 2 (refactoring ครั้งถัดมา):
moved {
  from = aws_instance.web_server
  to   = module.compute.aws_instance.web_server
}
```

Terraform สามารถ chain หลาย moved blocks ได้ในครั้งเดียว
ถ้า state มี `aws_instance.server` Terraform จะ:
1. ย้ายจาก `aws_instance.server` → `aws_instance.web_server`
2. จากนั้นย้าย `aws_instance.web_server` → `module.compute.aws_instance.web_server`

**ข้อควรระวัง:** Cycle ใน moved blocks จะทำให้ error!

```hcl
# ERROR: ห้ามทำแบบนี้ - circular dependency
moved {
  from = aws_instance.a
  to   = aws_instance.b
}

moved {
  from = aws_instance.b
  to   = aws_instance.a  # circular!
}
```

---

## ขั้นตอนที่ 708: การลบ Moved Blocks หลัง Apply

### เมื่อไหรควรลบ moved blocks?

**ลบได้เมื่อ:** ทุกคนที่ใช้ configuration นี้ได้ run `terraform apply` ไปแล้ว

**เหตุผล:**
- moved blocks ที่ไม่มี resource ใน state ที่ตรงกับ `from` จะถูก ignore
- แต่ถ้า state ยังมีชื่อเก่า จะต้องมี moved block ไว้

**Workflow:**

```bash
# Step 1: เพิ่ม moved block และ config ใหม่
git add .
git commit -m "refactor: rename web to web_server"

# Step 2: ทีมทุกคน apply
terraform apply

# Step 3: ลบ moved blocks (แต่เก็บ config ใหม่)
# แก้ไข main.tf ลบ moved {} blocks ออก
git add .
git commit -m "cleanup: remove moved blocks after refactor"
```

### Script สำหรับ cleanup moved blocks

```bash
#!/bin/bash
# cleanup_moved_blocks.sh
# ลบ moved blocks จากไฟล์ .tf ทั้งหมด

# ตรวจสอบว่า apply แล้วหรือยัง
if ! terraform plan -detailed-exitcode 2>/dev/null; then
  echo "WARNING: Plan shows changes. Apply first before removing moved blocks."
  exit 1
fi

# ลบ moved blocks (ใช้ awk)
for file in *.tf; do
  awk '
    /^moved \{/{in_block=1; next}
    in_block && /^\}/{in_block=0; next}
    in_block{next}
    {print}
  ' "$file" > "${file}.tmp" && mv "${file}.tmp" "$file"
done

echo "Moved blocks removed. Run terraform plan to verify no changes."
```

---

## ขั้นตอนที่ 709: Removed Block (Terraform 1.7+)

### ทำไมต้องมี removed block?

ก่อน Terraform 1.7: ถ้าลบ resource ออกจาก config แต่ไม่อยากให้ Terraform destroy มัน
ต้องใช้:
```bash
terraform state rm aws_instance.legacy
```

Terraform 1.7+ มี `removed` block ที่ทำให้สิ่งนี้เป็น declarative:

### Syntax ของ removed block

```hcl
removed {
  from = aws_instance.legacy

  lifecycle {
    destroy = false  # ไม่ destroy resource จริง, แค่ลบออกจาก state
  }
}
```

หรือถ้าต้องการ destroy จริง:
```hcl
removed {
  from = aws_instance.old_server

  lifecycle {
    destroy = true  # destroy resource เมื่อ apply
  }
}
```

### ตัวอย่างการใช้งาน

**Scenario 1: ลบ resource ออกจาก Terraform management (ไม่ destroy)**
```hcl
# ต้องการหยุด manage S3 bucket นี้ด้วย Terraform
# แต่ไม่อยากลบ bucket จริงๆ

# ลบ resource block นี้ออกจาก config:
# resource "aws_s3_bucket" "legacy_data" { ... }

# เพิ่ม removed block แทน:
removed {
  from = aws_s3_bucket.legacy_data

  lifecycle {
    destroy = false
  }
}
```

```bash
$ terraform plan

Terraform will perform the following actions:

  # aws_s3_bucket.legacy_data will no longer be managed by Terraform
  # (destroy = false)
  # (nothing will happen to the actual bucket)

Plan: 0 to add, 0 to change, 0 to destroy.
Changes to Outputs: (none)

Note: The plan removed 1 resource instance from the state.
```

**Scenario 2: ลบ resource บาง instance จาก for_each**
```hcl
# เดิมมี 3 environments
resource "aws_s3_bucket" "env_buckets" {
  for_each = toset(["dev", "staging", "prod"])
  bucket   = "mycompany-${each.key}-data"
}

# ต้องการลบ staging ออกจาก management (ไม่ destroy bucket จริง)
# ลบ "staging" ออกจาก for_each:
resource "aws_s3_bucket" "env_buckets" {
  for_each = toset(["dev", "prod"])  # ลบ staging ออก
  bucket   = "mycompany-${each.key}-data"
}

# เพิ่ม removed block:
removed {
  from = aws_s3_bucket.env_buckets["staging"]

  lifecycle {
    destroy = false
  }
}
```

**Scenario 3: removed block กับ module**
```hcl
# ต้องการหยุด manage module ทั้งหมด
removed {
  from = module.legacy_application

  lifecycle {
    destroy = false
  }
}
```

---

## ขั้นตอนที่ 710: Refactoring Patterns

### Pattern 1: Splitting Large Configuration

**ปัญหา:** main.tf มีขนาดใหญ่มาก ต้องการแยกออกเป็นหลายไฟล์/modules

```hcl
# main.tf เดิม (ใหญ่มาก - 1000+ บรรทัด)
resource "aws_vpc" "main" { ... }
resource "aws_subnet" "public_1" { ... }
resource "aws_subnet" "public_2" { ... }
resource "aws_subnet" "private_1" { ... }
resource "aws_subnet" "private_2" { ... }
resource "aws_instance" "app_server" { ... }
resource "aws_instance" "db_server" { ... }
resource "aws_rds_cluster" "main" { ... }
resource "aws_elasticache_cluster" "main" { ... }
# ... อีก 50 resources
```

**Step 1: สร้าง module structure:**
```
.
├── main.tf
├── networking.tf     (หรือ modules/networking/)
├── compute.tf        (หรือ modules/compute/)
├── database.tf       (หรือ modules/database/)
└── cache.tf
```

**Step 2: แยก networking ออกมา:**
```hcl
# networking.tf (ใหม่)
resource "aws_vpc" "main" {
  cidr_block = "10.0.0.0/16"
  # ...
}

resource "aws_subnet" "public_1" {
  vpc_id     = aws_vpc.main.id
  cidr_block = "10.0.1.0/24"
  # ...
}

# ถ้าแค่ย้ายไฟล์ ไม่เปลี่ยนชื่อ - ไม่ต้องใช้ moved block!
# Terraform ไม่สนใจว่า resource อยู่ไฟล์ไหน
```

**Step 3: เปลี่ยนเป็น module พร้อม moved blocks:**
```hcl
# main.tf
module "networking" {
  source = "./modules/networking"
  vpc_cidr = "10.0.0.0/16"
}

module "compute" {
  source    = "./modules/compute"
  vpc_id    = module.networking.vpc_id
  subnet_id = module.networking.public_subnet_id
}

# networking_moved.tf (แยกไฟล์เพื่อความชัดเจน)
moved {
  from = aws_vpc.main
  to   = module.networking.aws_vpc.main
}

moved {
  from = aws_subnet.public_1
  to   = module.networking.aws_subnet.public_1
}

moved {
  from = aws_subnet.public_2
  to   = module.networking.aws_subnet.public_2
}

# compute_moved.tf
moved {
  from = aws_instance.app_server
  to   = module.compute.aws_instance.app_server
}
```

### Pattern 2: การเปลี่ยนจาก count เป็น for_each

นี่คือ refactoring ที่ทำบ่อยมาก เพราะ `for_each` ยืดหยุ่นกว่า `count`

**Before (count):**
```hcl
variable "server_names" {
  type    = list(string)
  default = ["web-1", "web-2", "web-3"]
}

resource "aws_instance" "servers" {
  count         = length(var.server_names)
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t3.micro"

  tags = {
    Name = var.server_names[count.index]
  }
}

# State addresses:
# aws_instance.servers[0]
# aws_instance.servers[1]
# aws_instance.servers[2]
```

**After (for_each):**
```hcl
variable "servers" {
  type = map(object({
    instance_type = string
  }))
  default = {
    "web-1" = { instance_type = "t3.micro" }
    "web-2" = { instance_type = "t3.micro" }
    "web-3" = { instance_type = "t3.small" }
  }
}

resource "aws_instance" "servers" {
  for_each      = var.servers
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = each.value.instance_type

  tags = {
    Name = each.key
  }
}

# moved blocks สำหรับ count -> for_each
moved {
  from = aws_instance.servers[0]
  to   = aws_instance.servers["web-1"]
}

moved {
  from = aws_instance.servers[1]
  to   = aws_instance.servers["web-2"]
}

moved {
  from = aws_instance.servers[2]
  to   = aws_instance.servers["web-3"]
}
```

### Pattern 3: Renaming Module และ Internal Resources

```hcl
# เดิม
module "db" {
  source = "./modules/database"
}

# ใหม่
module "primary_database" {
  source = "./modules/database"
}

# ย้าย module ทั้งหมด (รวม sub-resources ทั้งหมดด้วย)
moved {
  from = module.db
  to   = module.primary_database
}
```

### Pattern 4: การแยก Module ที่ใหญ่เกินไป

```hcl
# เดิม - module เดี่ยวที่ทำทุกอย่าง
module "infrastructure" {
  source = "./modules/infrastructure"
}

# ใหม่ - แยกออกเป็น 3 modules
module "networking" {
  source = "./modules/networking"
}

module "compute" {
  source = "./modules/compute"
  vpc_id = module.networking.vpc_id
}

module "data" {
  source    = "./modules/data"
  vpc_id    = module.networking.vpc_id
  subnet_id = module.networking.private_subnet_id
}

# moved blocks
moved {
  from = module.infrastructure.aws_vpc.main
  to   = module.networking.aws_vpc.main
}

moved {
  from = module.infrastructure.aws_subnet.public
  to   = module.networking.aws_subnet.public
}

moved {
  from = module.infrastructure.aws_instance.app
  to   = module.compute.aws_instance.app
}

moved {
  from = module.infrastructure.aws_rds_cluster.main
  to   = module.data.aws_rds_cluster.main
}
```

---

## ขั้นตอนที่ 711 (Bonus): Import Block กับ Moved Block (Terraform 1.5+)

### การ import resource ที่สร้างนอก Terraform แล้วย้ายเข้า module

```hcl
# Step 1: Import resource เข้ามาก่อน
import {
  id = "my-existing-bucket"
  to = aws_s3_bucket.imported_bucket
}

resource "aws_s3_bucket" "imported_bucket" {
  bucket = "my-existing-bucket"
}

# Step 2: หลัง apply แล้ว ย้ายเข้า module
# (ใน Terraform apply ถัดมา)
module "storage" {
  source      = "./modules/storage"
  bucket_name = "my-existing-bucket"
}

moved {
  from = aws_s3_bucket.imported_bucket
  to   = module.storage.aws_s3_bucket.main
}
```

### ใช้ทั้ง import และ moved พร้อมกัน

```hcl
# Import resource พร้อมกับ moved block ได้ในครั้งเดียว
import {
  id = "existing-vpc-id"
  to = module.networking.aws_vpc.main
}

module "networking" {
  source   = "./modules/networking"
  vpc_cidr = "10.0.0.0/16"
}
```

---

## ขั้นตอนที่ 712: Automated Refactoring Workflow

### Complete Refactoring Checklist

```bash
#!/bin/bash
# refactor_check.sh - Script ตรวจสอบก่อน refactor

echo "=== Pre-Refactoring Checks ==="

# 1. ตรวจสอบว่า state ปัจจุบัน clean
echo "1. Checking current state..."
terraform plan -detailed-exitcode
if [ $? -eq 1 ]; then
  echo "ERROR: Current plan has errors. Fix before refactoring."
  exit 1
fi

if [ $? -eq 2 ]; then
  echo "WARNING: There are pending changes. Apply them first."
  read -p "Continue anyway? (y/N) " confirm
  if [ "$confirm" != "y" ]; then
    exit 1
  fi
fi

# 2. Backup state
echo "2. Creating state backup..."
BACKUP_FILE="terraform.tfstate.backup.$(date +%Y%m%d_%H%M%S)"
cp terraform.tfstate "$BACKUP_FILE" 2>/dev/null || \
  terraform state pull > "$BACKUP_FILE"
echo "State backed up to: $BACKUP_FILE"

# 3. List all resources in state
echo "3. Current state resources:"
terraform state list

echo "=== Ready to refactor ==="
echo "Remember: Add moved blocks BEFORE changing resource addresses"
```

### Refactoring Workflow Steps

```markdown
## Workflow สำหรับการ Refactor

### Phase 1: Preparation
1. [ ] Backup current state
2. [ ] Run terraform plan (should be no-op)
3. [ ] Create feature branch in git
4. [ ] Document what you're changing and why

### Phase 2: Implementation
1. [ ] Add moved blocks FIRST (ก่อนเปลี่ยน config)
2. [ ] Update resource configurations
3. [ ] Update references (other resources referencing moved resources)
4. [ ] Update outputs

### Phase 3: Validation
1. [ ] Run terraform validate
2. [ ] Run terraform plan
3. [ ] Verify plan shows ONLY moves (0 to add, 0 to change, 0 to destroy)
4. [ ] Review plan output carefully

### Phase 4: Apply
1. [ ] Get team approval
2. [ ] Run terraform apply
3. [ ] Verify post-apply plan is no-op

### Phase 5: Cleanup
1. [ ] Remove moved blocks (after everyone has applied)
2. [ ] Run final terraform plan (should be no-op)
3. [ ] Commit cleanup
4. [ ] Tag release
```

---

## ขั้นตอนที่ 713: Testing Refactors (Should Produce No-op Plan)

### Principle: Refactor ที่ดีต้องไม่มีการ create/destroy จริง

```bash
# Test script สำหรับ refactoring
#!/bin/bash
# test_refactor.sh

echo "Testing refactoring..."

# Run plan และ capture output
PLAN_OUTPUT=$(terraform plan -no-color 2>&1)

# ตรวจสอบว่าไม่มีการ add/change/destroy
if echo "$PLAN_OUTPUT" | grep -q "Plan: 0 to add, 0 to change, 0 to destroy"; then
  echo "✓ Refactoring is clean - no resource changes"
  exit 0
else
  echo "✗ Refactoring has unexpected changes:"
  echo "$PLAN_OUTPUT" | grep -E "(Plan:|must be|will be|is tainted)"
  exit 1
fi
```

### ตัวอย่าง Terraform Test สำหรับ Refactoring

```hcl
# tests/refactor_test.tftest.hcl
run "verify_refactor_is_no_op" {
  command = plan

  assert {
    condition     = length(planned_values.root_module.resources) > 0
    error_message = "No resources found in plan - something went wrong"
  }
}

# ตรวจสอบ output ยังคงมีค่าถูกต้อง
run "verify_outputs_unchanged" {
  command = plan

  assert {
    condition     = output.vpc_id != ""
    error_message = "VPC ID output should not be empty after refactor"
  }
}
```

---

## สรุป (Summary)

| Feature | Version | Use Case |
|---------|---------|----------|
| `moved` block | Terraform 1.1+ | เปลี่ยนชื่อ resource, ย้ายเข้า/ออก module |
| `removed` block | Terraform 1.7+ | ลบ resource ออกจาก management โดยไม่ destroy |
| `import` block | Terraform 1.5+ | Import existing infrastructure |
| `terraform state mv` | ทุก version | Manual state manipulation |

### Key Principles

1. **moved blocks อยู่ใน version control** - ทุกคนในทีมเห็นและ apply ได้
2. **ลบ moved blocks หลัง apply** - แต่ต้องแน่ใจว่าทุกคน apply แล้ว
3. **Refactor ที่ดี = no-op plan** - ไม่มีการ create/destroy จริง
4. **Backup ก่อนเสมอ** - โดยเฉพาะ production state
5. **ทำทีละขั้นตอน** - อย่า refactor หลายอย่างพร้อมกัน

---

## แบบฝึกหัด (Exercises)

### Exercise 1: Basic Rename
```hcl
# TODO: เปลี่ยนชื่อ resource นี้จาก "old" เป็น "new" โดยไม่ destroy
resource "aws_s3_bucket" "old" {
  bucket = "my-exercise-bucket-12345"
}
```

### Exercise 2: Module Introduction
```hcl
# TODO: ย้าย resources เหล่านี้เข้า module "compute"
resource "aws_instance" "web" {
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t3.micro"
}

resource "aws_eip" "web" {
  instance = aws_instance.web.id
}
```

### Exercise 3: Count to For_Each Migration
```hcl
# TODO: เปลี่ยนจาก count เป็น for_each โดยใช้ moved blocks
resource "aws_security_group" "servers" {
  count  = 3
  name   = "server-sg-${count.index}"
  vpc_id = var.vpc_id
}
```

---

*จบ Part 071 - ในส่วนถัดไปจะเรียนรู้เรื่อง Terraform Testing Framework*
