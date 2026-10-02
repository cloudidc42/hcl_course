# Part 21: Resource Meta-Arguments (ขั้นตอนที่ 201-210)

## ภาพรวม (Overview)

Meta-arguments คือ arguments พิเศษที่ใช้ได้กับ resource block ทุกประเภทใน Terraform ไม่ว่าจะเป็น AWS, Azure, GCP หรือ provider อื่นๆ ต่างจาก arguments ปกติที่เป็น configuration ของ resource นั้นๆ โดยเฉพาะ

Meta-arguments ที่มีใน Terraform:
1. `depends_on` - กำหนด explicit dependencies
2. `count` - สร้าง resource หลายชิ้นจาก block เดียว
3. `for_each` - สร้าง resource จาก collection
4. `provider` - กำหนด provider ที่ใช้งาน
5. `lifecycle` - ควบคุม lifecycle behavior
6. `provisioner` - รัน script หลังสร้าง resource (deprecated pattern)

---

## Step 201: depends_on - Explicit Dependencies

### ทำไมต้องมี depends_on?

Terraform สร้าง dependency graph อัตโนมัติโดยดูจาก references ระหว่าง resources แต่บางครั้ง dependency ไม่ได้แสดงออกมาผ่าน reference จึงต้องใช้ `depends_on` เพื่อบอก Terraform ว่า resource A ต้อง apply หลัง resource B เสมอ

### Syntax พื้นฐาน

```hcl
resource "aws_instance" "app" {
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t2.micro"

  depends_on = [
    aws_iam_role_policy.app_policy,
    aws_s3_bucket.app_data
  ]
}
```

### ตัวอย่างที่ 1: EC2 ขึ้นอยู่กับ IAM Role Policy

```hcl
# สร้าง IAM Role
resource "aws_iam_role" "ec2_role" {
  name = "ec2-s3-access-role"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Action = "sts:AssumeRole"
        Effect = "Allow"
        Principal = {
          Service = "ec2.amazonaws.com"
        }
      }
    ]
  })
}

# สร้าง Policy
resource "aws_iam_role_policy" "s3_access" {
  name = "s3-access-policy"
  role = aws_iam_role.ec2_role.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Action = ["s3:GetObject", "s3:PutObject"]
        Resource = "arn:aws:s3:::my-app-bucket/*"
      }
    ]
  })
}

# Instance Profile
resource "aws_iam_instance_profile" "ec2_profile" {
  name = "ec2-instance-profile"
  role = aws_iam_role.ec2_role.name
}

# EC2 Instance ที่ depends on policy (ไม่ใช่แค่ role)
# เหตุผล: ถ้า EC2 สร้างก่อน policy อาจทำงานได้ไม่ครบ
resource "aws_instance" "app" {
  ami                  = "ami-0c55b159cbfafe1f0"
  instance_type        = "t2.micro"
  iam_instance_profile = aws_iam_instance_profile.ec2_profile.name

  # depends_on นี้จำเป็นเพราะ EC2 reference แค่ instance_profile
  # แต่เราต้องการให้ policy พร้อมก่อนด้วย
  depends_on = [aws_iam_role_policy.s3_access]

  tags = {
    Name = "app-server"
  }
}
```

### ตัวอย่างที่ 2: RDS ขึ้นอยู่กับ Security Group Rules

```hcl
resource "aws_security_group" "db_sg" {
  name   = "database-sg"
  vpc_id = aws_vpc.main.id
}

resource "aws_security_group_rule" "allow_app" {
  type                     = "ingress"
  from_port                = 5432
  to_port                  = 5432
  protocol                 = "tcp"
  source_security_group_id = aws_security_group.app_sg.id
  security_group_id        = aws_security_group.db_sg.id
}

resource "aws_db_instance" "postgres" {
  identifier        = "production-db"
  engine            = "postgres"
  engine_version    = "14.7"
  instance_class    = "db.t3.micro"
  allocated_storage = 20
  
  db_name  = "appdb"
  username = "admin"
  password = var.db_password
  
  vpc_security_group_ids = [aws_security_group.db_sg.id]
  
  # ต้องรอ security group rules พร้อมก่อน
  depends_on = [aws_security_group_rule.allow_app]
  
  skip_final_snapshot = true
}
```

### ตัวอย่างที่ 3: Lambda ขึ้นอยู่กับ CloudWatch Log Group

```hcl
resource "aws_cloudwatch_log_group" "lambda_logs" {
  name              = "/aws/lambda/my-function"
  retention_in_days = 14
}

resource "aws_lambda_function" "my_function" {
  filename         = "function.zip"
  function_name    = "my-function"
  role             = aws_iam_role.lambda_role.arn
  handler          = "index.handler"
  runtime          = "nodejs18.x"
  source_code_hash = filebase64sha256("function.zip")

  # Lambda จะสร้าง log group เองถ้าไม่มี
  # แต่เราสร้างไว้ก่อนเพื่อควบคุม retention
  # ต้องให้แน่ใจว่า log group พร้อมก่อน
  depends_on = [
    aws_cloudwatch_log_group.lambda_logs,
    aws_iam_role_policy_attachment.lambda_basic
  ]
}
```

### ตัวอย่างที่ 4: VPC Endpoints ขึ้นอยู่กับ Route Tables

```hcl
resource "aws_vpc" "main" {
  cidr_block = "10.0.0.0/16"
}

resource "aws_route_table" "private" {
  vpc_id = aws_vpc.main.id
  
  tags = { Name = "private-rt" }
}

resource "aws_route_table_association" "private_subnet_1" {
  subnet_id      = aws_subnet.private_1.id
  route_table_id = aws_route_table.private.id
}

resource "aws_route_table_association" "private_subnet_2" {
  subnet_id      = aws_subnet.private_2.id
  route_table_id = aws_route_table.private.id
}

# VPC Endpoint ต้องรอ route table associations พร้อมก่อน
resource "aws_vpc_endpoint" "s3" {
  vpc_id            = aws_vpc.main.id
  service_name      = "com.amazonaws.${var.region}.s3"
  vpc_endpoint_type = "Gateway"
  route_table_ids   = [aws_route_table.private.id]

  depends_on = [
    aws_route_table_association.private_subnet_1,
    aws_route_table_association.private_subnet_2
  ]
}
```

### Use Cases สำหรับ depends_on

```hcl
# ✅ Use Case 1: Hidden dependency ผ่าน external system
resource "aws_instance" "worker" {
  ami           = data.aws_ami.amazon_linux.id
  instance_type = "t3.medium"
  
  user_data = <<-EOF
    #!/bin/bash
    aws s3 cp s3://config-bucket/app.conf /etc/app.conf
  EOF

  # EC2 ต้องการไฟล์ใน S3 ตอน boot
  # แต่ไม่ได้ reference bucket โดยตรง
  depends_on = [aws_s3_object.app_config]
}

# ✅ Use Case 2: Database schema migration ต้องรอ DB พร้อม
resource "null_resource" "db_migration" {
  provisioner "local-exec" {
    command = "psql ${aws_db_instance.main.endpoint} -f migrations/001_initial.sql"
  }
  
  depends_on = [aws_db_instance.main]
}

# ✅ Use Case 3: EKS Addons ต้องรอ node group พร้อม
resource "aws_eks_addon" "coredns" {
  cluster_name = aws_eks_cluster.main.name
  addon_name   = "coredns"
  
  depends_on = [aws_eks_node_group.main]
}
```

### Anti-patterns ของ depends_on

```hcl
# ❌ Anti-pattern 1: depends_on ที่ไม่จำเป็น
# (Terraform track dependency อัตโนมัติแล้วผ่าน reference)
resource "aws_instance" "bad" {
  ami           = "ami-xxx"
  instance_type = "t2.micro"
  subnet_id     = aws_subnet.main.id  # มี reference แล้ว

  # ❌ ไม่จำเป็น! มี reference อยู่แล้ว
  depends_on = [aws_subnet.main]
}

# ✅ ที่ถูกต้อง: ไม่ต้องใส่ depends_on เพราะมี reference
resource "aws_instance" "good" {
  ami           = "ami-xxx"
  instance_type = "t2.micro"
  subnet_id     = aws_subnet.main.id  # Terraform รู้ dependency แล้ว
}

# ❌ Anti-pattern 2: depends_on กับ module ที่ไม่ควร
# (ทำให้ plan/apply ช้าลงโดยไม่จำเป็น)
module "networking" {
  source = "./modules/networking"
}

module "compute" {
  source = "./modules/compute"
  
  # ❌ ถ้า compute module reference outputs จาก networking
  # Terraform จะรู้ dependency เองแล้ว ไม่ต้อง depends_on
  depends_on = [module.networking]
}
```

---

## Step 202: count - Resource Multiplication

### Syntax พื้นฐาน

```hcl
resource "aws_instance" "server" {
  count         = 3
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t2.micro"

  tags = {
    Name = "server-${count.index}"
  }
}
```

### count.index

`count.index` เริ่มต้นที่ 0 และเพิ่มขึ้นทีละ 1

```hcl
# สร้าง 5 subnets
resource "aws_subnet" "private" {
  count             = 5
  vpc_id            = aws_vpc.main.id
  cidr_block        = "10.0.${count.index + 1}.0/24"
  availability_zone = data.aws_availability_zones.available.names[count.index % 3]

  tags = {
    Name = "private-subnet-${count.index + 1}"
    Type = "private"
  }
}

# Reference แบบ index
output "subnet_ids" {
  value = aws_subnet.private[*].id  # ได้ list ของทุก subnet
}

# Reference subnet เดียว
output "first_subnet" {
  value = aws_subnet.private[0].id
}
```

### ตัวอย่าง count กับ variables

```hcl
variable "instance_count" {
  description = "จำนวน web servers"
  type        = number
  default     = 2
}

variable "environments" {
  description = "รายชื่อ environment"
  type        = list(string)
  default     = ["dev", "staging", "prod"]
}

# ใช้ variable สำหรับ count
resource "aws_instance" "web" {
  count         = var.instance_count
  ami           = data.aws_ami.ubuntu.id
  instance_type = var.instance_type

  tags = {
    Name        = "web-server-${count.index + 1}"
    Environment = var.environment
  }
}

# ใช้ length() กับ list
resource "aws_s3_bucket" "env_buckets" {
  count  = length(var.environments)
  bucket = "my-app-${var.environments[count.index]}-${random_id.bucket_suffix.hex}"

  tags = {
    Environment = var.environments[count.index]
  }
}
```

### Conditional Creation ด้วย count

```hcl
variable "enable_monitoring" {
  description = "เปิดใช้ monitoring หรือไม่"
  type        = bool
  default     = false
}

variable "enable_nat_gateway" {
  type    = bool
  default = true
}

variable "environment" {
  type    = string
  default = "dev"
}

# Conditional resource: สร้างถ้า enable_monitoring = true
resource "aws_cloudwatch_metric_alarm" "high_cpu" {
  count = var.enable_monitoring ? 1 : 0

  alarm_name          = "high-cpu-utilization"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = "2"
  metric_name         = "CPUUtilization"
  namespace           = "AWS/EC2"
  period              = "120"
  statistic           = "Average"
  threshold           = "80"
  alarm_description   = "CPU utilization is too high"
  
  dimensions = {
    InstanceId = aws_instance.app[0].id
  }
}

# สร้าง NAT Gateway เฉพาะ production
resource "aws_nat_gateway" "main" {
  count         = var.environment == "prod" ? 1 : 0
  allocation_id = aws_eip.nat[0].id
  subnet_id     = aws_subnet.public[0].id

  tags = { Name = "main-nat-gw" }
}

# หรือใช้ count = var.enable_nat_gateway ? 1 : 0
resource "aws_nat_gateway" "optional" {
  count         = var.enable_nat_gateway ? 1 : 0
  allocation_id = aws_eip.nat_optional[0].id
  subnet_id     = aws_subnet.public[0].id

  tags = { Name = "optional-nat-gw" }
}
```

### ตัวอย่าง Count กับ Security Groups

```hcl
variable "allowed_ports" {
  description = "Ports ที่อนุญาต"
  type        = list(number)
  default     = [80, 443, 8080, 8443]
}

resource "aws_security_group" "web" {
  name   = "web-sg"
  vpc_id = aws_vpc.main.id
}

resource "aws_security_group_rule" "ingress" {
  count = length(var.allowed_ports)

  type              = "ingress"
  from_port         = var.allowed_ports[count.index]
  to_port           = var.allowed_ports[count.index]
  protocol          = "tcp"
  cidr_blocks       = ["0.0.0.0/0"]
  security_group_id = aws_security_group.web.id

  description = "Allow port ${var.allowed_ports[count.index]}"
}
```

### ตัวอย่าง Count กับ IAM Users

```hcl
variable "team_members" {
  description = "รายชื่อ team members"
  type        = list(string)
  default     = ["alice", "bob", "charlie", "diana"]
}

resource "aws_iam_user" "team" {
  count = length(var.team_members)
  name  = var.team_members[count.index]
  path  = "/team/"

  tags = {
    Department = "Engineering"
    Member     = var.team_members[count.index]
  }
}

resource "aws_iam_user_group_membership" "team_membership" {
  count = length(var.team_members)
  user  = aws_iam_user.team[count.index].name
  groups = ["Developers"]

  depends_on = [aws_iam_group.developers]
}
```

### ปัญหาของ count กับ list ordering

```hcl
# ⚠️ ปัญหา: ถ้าเอา element ออกจาก list กลางๆ
# Terraform จะ destroy และสร้างใหม่หลาย resources

# สมมติมี list = ["a", "b", "c"]
# index:           0    1    2

# ถ้าเอา "b" ออก กลายเป็น ["a", "c"]
# index:                    0    1
# Terraform จะ: destroy index 2, modify index 1 ให้เป็น "c"
# นี่อาจไม่ใช่สิ่งที่ต้องการ!

# ✅ แก้ไขด้วยการใช้ for_each แทน (ดู Step 203)
```

---

## Step 203: for_each - Advanced Resource Creation

### for_each กับ Set of Strings

```hcl
# ใช้ toset() แปลง list เป็น set
resource "aws_iam_user" "developers" {
  for_each = toset(["alice", "bob", "charlie"])
  
  name = each.key  # "alice", "bob", "charlie"
  path = "/developers/"

  tags = {
    Name = each.key
  }
}

# Reference resources
output "developer_arns" {
  value = {
    for username, user in aws_iam_user.developers :
    username => user.arn
  }
}
```

### for_each กับ Map of Objects

```hcl
variable "servers" {
  description = "Server configurations"
  type = map(object({
    instance_type = string
    ami           = string
    environment   = string
  }))
  default = {
    web = {
      instance_type = "t3.small"
      ami           = "ami-0c55b159cbfafe1f0"
      environment   = "prod"
    }
    api = {
      instance_type = "t3.medium"
      ami           = "ami-0c55b159cbfafe1f0"
      environment   = "prod"
    }
    worker = {
      instance_type = "t3.large"
      ami           = "ami-0c55b159cbfafe1f0"
      environment   = "prod"
    }
  }
}

resource "aws_instance" "servers" {
  for_each      = var.servers
  
  ami           = each.value.ami
  instance_type = each.value.instance_type

  tags = {
    Name        = each.key              # "web", "api", "worker"
    Environment = each.value.environment
    Role        = each.key
  }
}

# Reference เฉพาะ server
output "web_server_ip" {
  value = aws_instance.servers["web"].public_ip
}

output "all_server_ids" {
  value = {
    for k, v in aws_instance.servers : k => v.id
  }
}
```

### for_each กับ S3 Buckets

```hcl
variable "buckets" {
  description = "S3 bucket configurations"
  type = map(object({
    versioning  = bool
    encryption  = bool
    public      = bool
    lifecycle_days = number
  }))
  default = {
    "assets" = {
      versioning     = false
      encryption     = true
      public         = true
      lifecycle_days = 90
    }
    "backups" = {
      versioning     = true
      encryption     = true
      public         = false
      lifecycle_days = 365
    }
    "logs" = {
      versioning     = false
      encryption     = true
      public         = false
      lifecycle_days = 30
    }
  }
}

resource "aws_s3_bucket" "buckets" {
  for_each = var.buckets
  bucket   = "${var.project_name}-${each.key}-${var.environment}"

  tags = {
    Name        = each.key
    Environment = var.environment
    Project     = var.project_name
  }
}

resource "aws_s3_bucket_versioning" "buckets" {
  for_each = { for k, v in var.buckets : k => v if v.versioning }
  
  bucket = aws_s3_bucket.buckets[each.key].id
  versioning_configuration {
    status = "Enabled"
  }
}

resource "aws_s3_bucket_server_side_encryption_configuration" "buckets" {
  for_each = { for k, v in var.buckets : k => v if v.encryption }
  
  bucket = aws_s3_bucket.buckets[each.key].id
  rule {
    apply_server_side_encryption_by_default {
      sse_algorithm = "AES256"
    }
  }
}

resource "aws_s3_bucket_lifecycle_configuration" "buckets" {
  for_each = var.buckets
  
  bucket = aws_s3_bucket.buckets[each.key].id

  rule {
    id     = "cleanup"
    status = "Enabled"
    
    expiration {
      days = each.value.lifecycle_days
    }
  }
}
```

### for_each กับ Security Groups - Complex Pattern

```hcl
variable "security_groups" {
  type = map(object({
    description = string
    ingress_rules = list(object({
      from_port   = number
      to_port     = number
      protocol    = string
      cidr_blocks = list(string)
      description = string
    }))
  }))
  default = {
    "web" = {
      description = "Web server security group"
      ingress_rules = [
        {
          from_port   = 80
          to_port     = 80
          protocol    = "tcp"
          cidr_blocks = ["0.0.0.0/0"]
          description = "HTTP"
        },
        {
          from_port   = 443
          to_port     = 443
          protocol    = "tcp"
          cidr_blocks = ["0.0.0.0/0"]
          description = "HTTPS"
        }
      ]
    }
    "app" = {
      description = "Application server security group"
      ingress_rules = [
        {
          from_port   = 8080
          to_port     = 8080
          protocol    = "tcp"
          cidr_blocks = ["10.0.0.0/8"]
          description = "App port"
        }
      ]
    }
  }
}

resource "aws_security_group" "groups" {
  for_each    = var.security_groups
  name        = "${var.project}-${each.key}-sg"
  description = each.value.description
  vpc_id      = aws_vpc.main.id

  dynamic "ingress" {
    for_each = each.value.ingress_rules
    content {
      from_port   = ingress.value.from_port
      to_port     = ingress.value.to_port
      protocol    = ingress.value.protocol
      cidr_blocks = ingress.value.cidr_blocks
      description = ingress.value.description
    }
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = { Name = "${var.project}-${each.key}-sg" }
}
```

### Complex for_each patterns

```hcl
# Pattern 1: for_each จาก locals ที่ computed
locals {
  # สร้าง map จาก combination
  subnet_configs = {
    for pair in setproduct(["public", "private"], ["a", "b", "c"]) :
    "${pair[0]}-${pair[1]}" => {
      type = pair[0]
      az   = pair[1]
      cidr = pair[0] == "public" ? 
        "10.0.${index(["a","b","c"], pair[1])}.0/24" :
        "10.0.${index(["a","b","c"], pair[1]) + 10}.0/24"
    }
  }
}

resource "aws_subnet" "all" {
  for_each          = local.subnet_configs
  vpc_id            = aws_vpc.main.id
  cidr_block        = each.value.cidr
  availability_zone = "${var.region}${each.value.az}"

  map_public_ip_on_launch = each.value.type == "public"

  tags = {
    Name = "subnet-${each.key}"
    Type = each.value.type
  }
}

# Pattern 2: for_each ที่ filter บาง items ออก
variable "all_features" {
  type = map(object({
    enabled = bool
    config  = map(string)
  }))
}

resource "aws_feature_resource" "enabled_only" {
  for_each = {
    for k, v in var.all_features : k => v
    if v.enabled  # filter เฉพาะ enabled = true
  }
  
  name   = each.key
  config = each.value.config
}
```

### for_each กับ IAM Role Policies

```hcl
variable "lambda_policies" {
  type = map(list(string))
  default = {
    "s3_read" = [
      "s3:GetObject",
      "s3:ListBucket"
    ]
    "dynamodb_rw" = [
      "dynamodb:GetItem",
      "dynamodb:PutItem",
      "dynamodb:UpdateItem",
      "dynamodb:DeleteItem",
      "dynamodb:Query",
      "dynamodb:Scan"
    ]
    "cloudwatch_logs" = [
      "logs:CreateLogGroup",
      "logs:CreateLogStream",
      "logs:PutLogEvents"
    ]
  }
}

resource "aws_iam_role_policy" "lambda_policies" {
  for_each = var.lambda_policies
  
  name = "${each.key}-policy"
  role = aws_iam_role.lambda.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect   = "Allow"
        Action   = each.value
        Resource = "*"
      }
    ]
  })
}
```

---

## Step 204: provider - Explicit Provider Assignment

### Multi-Region Setup

```hcl
# กำหนด provider หลัก
provider "aws" {
  region = "us-east-1"
}

# กำหนด provider เพิ่มเติมสำหรับ region อื่น
provider "aws" {
  alias  = "us-west-2"
  region = "us-west-2"
}

provider "aws" {
  alias  = "eu-west-1"
  region = "eu-west-1"
}

provider "aws" {
  alias  = "ap-southeast-1"
  region = "ap-southeast-1"
}
```

### ใช้ provider meta-argument

```hcl
# S3 bucket ใน us-east-1 (default)
resource "aws_s3_bucket" "us_east" {
  bucket = "my-bucket-us-east"
}

# S3 bucket ใน us-west-2
resource "aws_s3_bucket" "us_west" {
  provider = aws.us-west-2
  bucket   = "my-bucket-us-west"
}

# EC2 ใน ap-southeast-1
resource "aws_instance" "singapore" {
  provider      = aws.ap-southeast-1
  ami           = "ami-0d4ae09ec9361d8ac"  # Singapore AMI
  instance_type = "t3.micro"

  tags = { Name = "singapore-server" }
}

# CloudFront ต้องการ certificate ใน us-east-1 เสมอ
resource "aws_acm_certificate" "main" {
  provider          = aws.us-east-1  # CloudFront ต้องการ us-east-1
  domain_name       = "*.example.com"
  validation_method = "DNS"
}
```

### Multi-Account Setup

```hcl
provider "aws" {
  alias  = "prod"
  region = "us-east-1"
  
  assume_role {
    role_arn = "arn:aws:iam::PRODUCTION_ACCOUNT_ID:role/TerraformDeployRole"
  }
}

provider "aws" {
  alias  = "staging"
  region = "us-east-1"
  
  assume_role {
    role_arn = "arn:aws:iam::STAGING_ACCOUNT_ID:role/TerraformDeployRole"
  }
}

provider "aws" {
  alias  = "shared_services"
  region = "us-east-1"
  
  assume_role {
    role_arn = "arn:aws:iam::SHARED_SERVICES_ACCOUNT_ID:role/TerraformDeployRole"
  }
}

# Resources ใน prod account
resource "aws_vpc" "prod_vpc" {
  provider   = aws.prod
  cidr_block = "10.0.0.0/16"
  tags       = { Name = "prod-vpc" }
}

# Resources ใน staging account  
resource "aws_vpc" "staging_vpc" {
  provider   = aws.staging
  cidr_block = "10.1.0.0/16"
  tags       = { Name = "staging-vpc" }
}

# Shared resources
resource "aws_ecr_repository" "app" {
  provider = aws.shared_services
  name     = "my-application"
}
```

### provider ใน modules

```hcl
# Root module
provider "aws" {
  alias  = "primary"
  region = "us-east-1"
}

provider "aws" {
  alias  = "dr"
  region = "us-west-2"
}

module "primary_infra" {
  source = "./modules/vpc"
  
  providers = {
    aws = aws.primary
  }
  
  environment = "prod"
  cidr_block  = "10.0.0.0/16"
}

module "dr_infra" {
  source = "./modules/vpc"
  
  providers = {
    aws = aws.dr
  }
  
  environment = "prod-dr"
  cidr_block  = "10.1.0.0/16"
}
```

---

## Step 205: lifecycle - Controlling Resource Behavior

### create_before_destroy

```hcl
# ปัญหา default: Terraform destroy ก่อน แล้วค่อย create
# ถ้า resource อื่นอ้างอิง resource นี้อยู่จะเกิด error

resource "aws_security_group" "app" {
  name   = "app-security-group"
  vpc_id = aws_vpc.main.id

  lifecycle {
    create_before_destroy = true
  }
}

# ตัวอย่างจริง: SSL Certificate replacement
resource "aws_acm_certificate" "main" {
  domain_name       = var.domain_name
  validation_method = "DNS"

  # เมื่อ certificate ต้อง replace, สร้างอันใหม่ก่อน
  # แล้วค่อย destroy อันเก่า
  lifecycle {
    create_before_destroy = true
  }
}

# Launch Template replacement
resource "aws_launch_template" "app" {
  name_prefix   = "app-"
  image_id      = data.aws_ami.app.id
  instance_type = var.instance_type

  lifecycle {
    create_before_destroy = true
  }
}

# Auto Scaling Group ที่ใช้ launch template
resource "aws_autoscaling_group" "app" {
  name                = "app-asg-${aws_launch_template.app.latest_version}"
  desired_capacity    = 2
  max_size            = 5
  min_size            = 1
  vpc_zone_identifier = aws_subnet.private[*].id

  launch_template {
    id      = aws_launch_template.app.id
    version = "$Latest"
  }

  lifecycle {
    create_before_destroy = true
  }
}
```

### prevent_destroy

```hcl
# ป้องกัน Production Database จากการถูกลบโดยบังเอิญ
resource "aws_db_instance" "production" {
  identifier     = "production-database"
  engine         = "postgres"
  engine_version = "14.7"
  instance_class = "db.r5.large"
  
  allocated_storage     = 100
  storage_encrypted     = true
  multi_az              = true
  deletion_protection   = true  # AWS-level protection

  lifecycle {
    prevent_destroy = true  # Terraform-level protection
  }
}

# ป้องกัน S3 bucket ที่มีข้อมูลสำคัญ
resource "aws_s3_bucket" "critical_data" {
  bucket = "company-critical-data-prod"

  lifecycle {
    prevent_destroy = true
  }
}

# ป้องกัน VPC production
resource "aws_vpc" "production" {
  cidr_block = "10.0.0.0/16"

  lifecycle {
    prevent_destroy = true
  }
}
```

### ignore_changes

```hcl
# ตัวอย่าง 1: ignore AMI changes (update manually)
resource "aws_instance" "app" {
  ami           = data.aws_ami.app.id
  instance_type = "t3.medium"

  lifecycle {
    # ไม่ต้องการให้ Terraform replace instance ทุกครั้ง AMI update
    ignore_changes = [ami]
  }
}

# ตัวอย่าง 2: ignore tags ที่ถูก update จากระบบอื่น
resource "aws_instance" "managed" {
  ami           = "ami-xxx"
  instance_type = "t3.micro"

  tags = {
    Name = "managed-instance"
  }

  lifecycle {
    # ระบบ CMDB update tags บางตัว ไม่ต้องการให้ Terraform override
    ignore_changes = [
      tags["LastUpdated"],
      tags["ManagedBy"],
      tags["CostCenter"]
    ]
  }
}

# ตัวอย่าง 3: EKS Cluster - ignore logging configuration (set externally)
resource "aws_eks_cluster" "main" {
  name     = "my-cluster"
  role_arn = aws_iam_role.eks.arn

  vpc_config {
    subnet_ids = aws_subnet.private[*].id
  }

  lifecycle {
    ignore_changes = [
      kubernetes_network_config  # Updated by EKS itself
    ]
  }
}

# ตัวอย่าง 4: RDS password managed by Secrets Manager
resource "aws_db_instance" "main" {
  identifier     = "app-database"
  engine         = "postgres"
  instance_class = "db.t3.medium"
  
  username = "admin"
  password = var.initial_db_password  # Set initially

  lifecycle {
    # Password changed via AWS Console/Secrets Manager
    # ไม่ต้องการให้ Terraform reset password
    ignore_changes = [password]
  }
}

# ตัวอย่าง 5: ignore all tags
resource "aws_instance" "external_tags" {
  ami           = "ami-xxx"
  instance_type = "t3.micro"

  lifecycle {
    ignore_changes = [tags]  # Ignore ทุก tag
  }
}
```

### replace_triggered_by

```hcl
# Terraform 1.2+ feature
# Force replacement ของ resource เมื่อ resource อื่นเปลี่ยน

resource "aws_launch_template" "app" {
  name_prefix   = "app-"
  image_id      = var.ami_id
  instance_type = var.instance_type
  user_data     = base64encode(var.user_data)
}

resource "aws_autoscaling_group" "app" {
  min_size            = 1
  max_size            = 3
  desired_capacity    = 2
  vpc_zone_identifier = var.subnet_ids

  launch_template {
    id      = aws_launch_template.app.id
    version = "$Latest"
  }

  lifecycle {
    # Force replace ASG เมื่อ launch template เปลี่ยน
    # เพื่อให้ instances ใหม่ถูก launch
    replace_triggered_by = [
      aws_launch_template.app
    ]
  }
}

# ตัวอย่างอื่น: replace API Gateway deployment เมื่อ config เปลี่ยน
resource "aws_api_gateway_rest_api" "main" {
  name = "my-api"
  body = jsonencode(local.api_spec)
}

resource "aws_api_gateway_deployment" "main" {
  rest_api_id = aws_api_gateway_rest_api.main.id

  lifecycle {
    create_before_destroy = true
    
    # Deploy ใหม่เสมอเมื่อ API definition เปลี่ยน
    replace_triggered_by = [
      aws_api_gateway_rest_api.main.body
    ]
  }
}
```

### รวม lifecycle options หลายอัน

```hcl
resource "aws_db_instance" "production" {
  identifier     = "production-db"
  engine         = "postgres"
  instance_class = "db.r5.xlarge"
  
  username = "admin"
  password = var.db_password
  
  apply_immediately   = false
  deletion_protection = true

  lifecycle {
    # ป้องกันการลบโดยบังเอิญ
    prevent_destroy = true
    
    # สร้างใหม่ก่อนลบ (zero-downtime replacement)
    create_before_destroy = true
    
    # Password managed externally
    # Engine version managed via maintenance window
    ignore_changes = [
      password,
      engine_version,
      latest_restorable_time
    ]
  }
}
```

---

## Step 206: provisioner - Running Scripts (Deprecated Pattern)

### ⚠️ คำเตือน

Terraform แนะนำให้หลีกเลี่ยง provisioners เพราะ:
1. ทำให้ idempotency เสีย
2. ยากต่อการ debug
3. ต้องการ network connectivity
4. ถ้า fail จะทำให้ resource อยู่ใน "tainted" state

แนะนำให้ใช้:
- `user_data` สำหรับ EC2 initialization
- Packer สำหรับ build AMI
- Configuration management tools (Ansible, Chef, Puppet)

### local-exec provisioner

```hcl
# รัน command บน local machine ที่รัน Terraform
resource "aws_instance" "app" {
  ami           = data.aws_ami.ubuntu.id
  instance_type = "t3.micro"

  provisioner "local-exec" {
    command = "echo 'Instance created: ${self.public_ip}' >> instances.log"
  }
}

# ตัวอย่างที่ใช้จริง: Trigger Ansible playbook
resource "aws_instance" "configured" {
  ami           = data.aws_ami.base.id
  instance_type = "t3.medium"
  key_name      = aws_key_pair.main.key_name

  vpc_security_group_ids = [aws_security_group.ssh.id]
  subnet_id              = aws_subnet.public[0].id

  provisioner "local-exec" {
    # รอให้ SSH พร้อมแล้วรัน Ansible
    command = <<-EOT
      sleep 30
      ansible-playbook -i '${self.public_ip},' \
        --private-key ${var.private_key_path} \
        -u ubuntu \
        playbook.yml
    EOT
  }

  tags = { Name = "configured-server" }
}

# local-exec เมื่อ destroy
resource "aws_instance" "tracked" {
  ami           = data.aws_ami.ubuntu.id
  instance_type = "t3.micro"

  provisioner "local-exec" {
    when    = destroy
    command = "deregister-instance.sh ${self.id}"
  }
}

# local-exec พร้อม environment variables
resource "aws_db_instance" "main" {
  identifier     = "mydb"
  engine         = "postgres"
  instance_class = "db.t3.micro"
  
  username = "admin"
  password = var.db_password

  provisioner "local-exec" {
    command = "run-migrations.sh"
    
    environment = {
      DB_HOST     = self.endpoint
      DB_NAME     = self.db_name
      DB_USER     = self.username
      DB_PASSWORD = var.db_password
    }
  }
}
```

### remote-exec provisioner

```hcl
# รัน commands บน remote resource
resource "aws_instance" "web" {
  ami           = data.aws_ami.ubuntu.id
  instance_type = "t3.micro"
  key_name      = aws_key_pair.main.key_name

  vpc_security_group_ids = [aws_security_group.ssh.id]

  # Connection block
  connection {
    type        = "ssh"
    user        = "ubuntu"
    private_key = file(var.private_key_path)
    host        = self.public_ip
  }

  # รัน commands บน instance
  provisioner "remote-exec" {
    inline = [
      "sudo apt-get update",
      "sudo apt-get install -y nginx",
      "sudo systemctl start nginx",
      "sudo systemctl enable nginx"
    ]
  }
}

# remote-exec ด้วย script file
resource "aws_instance" "app_server" {
  ami           = data.aws_ami.ubuntu.id
  instance_type = "t3.medium"
  key_name      = aws_key_pair.main.key_name

  connection {
    type        = "ssh"
    user        = "ubuntu"
    private_key = file(var.private_key_path)
    host        = self.public_ip
    timeout     = "5m"
  }

  provisioner "file" {
    source      = "scripts/setup.sh"
    destination = "/tmp/setup.sh"
  }

  provisioner "remote-exec" {
    inline = [
      "chmod +x /tmp/setup.sh",
      "/tmp/setup.sh"
    ]
  }
}
```

### connection block

```hcl
# SSH connection
resource "aws_instance" "linux" {
  ami           = data.aws_ami.ubuntu.id
  instance_type = "t3.micro"

  connection {
    type        = "ssh"
    user        = "ubuntu"
    private_key = file("~/.ssh/id_rsa")
    host        = self.public_ip
    port        = 22
    timeout     = "5m"
    
    # ถ้าผ่าน bastion host
    # bastion_host        = aws_instance.bastion.public_ip
    # bastion_user        = "ec2-user"
    # bastion_private_key = file("~/.ssh/bastion_key")
  }

  provisioner "remote-exec" {
    inline = ["echo 'connected!'"]
  }
}

# WinRM connection (Windows)
resource "aws_instance" "windows" {
  ami           = data.aws_ami.windows.id
  instance_type = "t3.medium"

  connection {
    type     = "winrm"
    user     = "Administrator"
    password = rsadecrypt(self.password_data, file("~/.ssh/private_key.pem"))
    host     = self.public_ip
    port     = 5986
    https    = true
    insecure = true  # skip TLS verification
    timeout  = "10m"
  }

  provisioner "remote-exec" {
    inline = [
      "Write-Host 'Connected to Windows instance'",
      "Install-WindowsFeature -Name Web-Server"
    ]
    interpreter = ["PowerShell", "-Command"]
  }
  
  get_password_data = true
}
```

---

## Step 207: Meta-Arguments Interaction

### count + lifecycle

```hcl
resource "aws_instance" "web" {
  count         = var.web_count
  ami           = data.aws_ami.ubuntu.id
  instance_type = "t3.small"

  tags = {
    Name  = "web-${count.index + 1}"
    Index = count.index
  }

  lifecycle {
    create_before_destroy = true
    ignore_changes        = [ami]
  }
}
```

### for_each + lifecycle

```hcl
resource "aws_security_group" "services" {
  for_each    = var.services
  name        = "${each.key}-sg"
  description = "Security group for ${each.key}"
  vpc_id      = aws_vpc.main.id

  lifecycle {
    create_before_destroy = true
    
    # ไม่ต้องการให้ Terraform update description
    ignore_changes = [description]
  }
}
```

### for_each + depends_on

```hcl
resource "aws_iam_role" "services" {
  for_each = var.service_roles
  name     = each.key
  
  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect    = "Allow"
      Principal = { Service = each.value.principal }
      Action    = "sts:AssumeRole"
    }]
  })
}

resource "aws_iam_role_policy_attachment" "services" {
  for_each = var.service_roles

  role       = aws_iam_role.services[each.key].name
  policy_arn = each.value.policy_arn

  depends_on = [aws_iam_role.services]
}
```

### provider + for_each (Multi-Region Resources)

```hcl
# Providers
provider "aws" {
  alias  = "us_east_1"
  region = "us-east-1"
}

provider "aws" {
  alias  = "eu_west_1"
  region = "eu-west-1"
}

# Route53 ต้องการ Health Checks ใน us-east-1
resource "aws_route53_health_check" "endpoints" {
  for_each = var.regional_endpoints
  provider = aws.us_east_1  # Health checks always in us-east-1

  fqdn              = each.value.fqdn
  port              = 443
  type              = "HTTPS"
  resource_path     = "/health"
  failure_threshold = "3"
  request_interval  = "30"

  tags = { Name = "health-check-${each.key}" }
}
```

---

## Step 208: count vs for_each - Decision Guide

### เมื่อไรใช้ count

```hcl
# ✅ ใช้ count เมื่อ:

# 1. Resources เหมือนกันทุกอย่าง ต่างกันแค่ index
resource "aws_instance" "worker" {
  count         = var.worker_count
  ami           = var.worker_ami
  instance_type = var.worker_type

  tags = {
    Name  = "worker-${count.index}"
  }
}

# 2. สร้างหรือไม่สร้าง resource (conditional)
resource "aws_nat_gateway" "main" {
  count         = var.create_nat_gateway ? 1 : 0
  allocation_id = aws_eip.nat[0].id
  subnet_id     = aws_subnet.public[0].id
}

# 3. จำนวน resources ที่กำหนดตายตัวและไม่เปลี่ยนบ่อย
resource "aws_subnet" "public" {
  count             = 3
  vpc_id            = aws_vpc.main.id
  cidr_block        = "10.0.${count.index}.0/24"
  availability_zone = data.aws_availability_zones.available.names[count.index]
}
```

### เมื่อไรใช้ for_each

```hcl
# ✅ ใช้ for_each เมื่อ:

# 1. Resources แต่ละอันมี config ต่างกัน
resource "aws_iam_user" "team" {
  for_each = toset(var.team_members)
  name     = each.value
}

# 2. ต้องการ address resource ด้วยชื่อที่มีความหมาย
resource "aws_security_group" "services" {
  for_each = var.service_configs
  name     = "${each.key}-sg"
}

# 3. List ที่อาจมีการเพิ่ม/ลบ items (stable keys)
variable "environments" {
  type    = map(string)
  default = {
    dev  = "development"
    stg  = "staging"
    prod = "production"
  }
}

resource "aws_s3_bucket" "envs" {
  for_each = var.environments
  bucket   = "myapp-${each.key}"
  
  tags = {
    Environment = each.value
  }
}
```

### เปรียบเทียบ count vs for_each

```hcl
# ❌ ปัญหากับ count เมื่อ list เปลี่ยน:

# เดิม: ["alice", "bob", "charlie"]
resource "aws_iam_user" "users_count" {
  count = length(var.usernames)
  name  = var.usernames[count.index]
}
# users_count[0] = alice
# users_count[1] = bob
# users_count[2] = charlie

# ถ้าเอา "bob" ออก: ["alice", "charlie"]
# users_count[0] = alice (ไม่เปลี่ยน)
# users_count[1] = charlie (เปลี่ยน! Terraform จะ destroy bob แล้ว update charlie)
# users_count[2] = ถูก destroy

# ✅ for_each ไม่มีปัญหานี้:
resource "aws_iam_user" "users_foreach" {
  for_each = toset(var.usernames)
  name     = each.key
}
# users_foreach["alice"] = alice
# users_foreach["bob"] = bob  (ถูก destroy ถ้าเอาออก)
# users_foreach["charlie"] = charlie (ไม่ถูก affect)
```

---

## Step 209: for_each Keys Best Practices

### ใช้ stable, unique keys

```hcl
# ✅ Good: ใช้ identifier ที่ไม่เปลี่ยน
resource "aws_iam_user" "users" {
  for_each = toset(["alice@company.com", "bob@company.com"])
  name     = each.key  # email ไม่ค่อยเปลี่ยน
}

# ✅ Good: ใช้ meaningful business keys
variable "regions" {
  default = {
    "us-east-1"      = "N. Virginia"
    "eu-west-1"      = "Ireland"
    "ap-southeast-1" = "Singapore"
  }
}

resource "aws_cloudwatch_log_group" "regional" {
  for_each = var.regions
  
  provider = # ...
  name     = "/app/logs/${each.key}"
}

# ❌ Bad: ใช้ index-based key
resource "aws_instance" "bad" {
  for_each = { for i, v in var.instances : tostring(i) => v }
  # key คือ "0", "1", "2" - เหมือน count แต่แย่กว่า
}

# ❌ Bad: ใช้ key ที่มีการเปลี่ยนแปลงบ่อย
resource "aws_s3_bucket" "bad" {
  for_each = { for b in var.buckets : b.created_at => b }
  # timestamp เป็น key ที่ไม่ stable
}
```

### การจัดการ key conflicts

```hcl
# ถ้า key อาจซ้ำกัน ต้องแก้ไขก่อน
locals {
  # เพิ่ม suffix เพื่อป้องกัน conflicts
  bucket_configs = {
    for bucket in var.buckets :
    "${bucket.name}-${bucket.region}" => bucket
  }
}

resource "aws_s3_bucket" "multi_region" {
  for_each = local.bucket_configs
  
  bucket   = "${each.value.name}-${each.value.region}"
  provider = # เลือก provider ตาม each.value.region
}
```

### Converting List to Map สำหรับ for_each

```hcl
variable "instances" {
  type = list(object({
    name          = string
    instance_type = string
    subnet        = string
  }))
  default = [
    { name = "web-1",    instance_type = "t3.small",  subnet = "public" },
    { name = "api-1",    instance_type = "t3.medium", subnet = "private" },
    { name = "worker-1", instance_type = "t3.large",  subnet = "private" }
  ]
}

# แปลง list เป็น map โดยใช้ name เป็น key
locals {
  instances_map = { for i in var.instances : i.name => i }
}

resource "aws_instance" "all" {
  for_each      = local.instances_map
  ami           = data.aws_ami.ubuntu.id
  instance_type = each.value.instance_type
  
  subnet_id = each.value.subnet == "public" ? 
    aws_subnet.public[0].id : 
    aws_subnet.private[0].id

  tags = { Name = each.key }
}
```

---

## Step 210: Changing count/for_each Values - Risks & Procedures

### ความเสี่ยงของการเปลี่ยน count

```hcl
# ⚠️ การลด count อาจทำให้ resources ถูก destroy
# เช่น count จาก 5 เป็น 3 -> instances[3] และ [4] จะถูก destroy

# ขั้นตอนที่ปลอดภัย:
# 1. ทำ plan ก่อนเสมอ
# $ terraform plan -var="instance_count=3"

# 2. Review plan อย่างละเอียด
# 3. ถ้ายืนยัน apply
# $ terraform apply -var="instance_count=3"

# ✅ ถ้าต้องการลด count โดยไม่ destroy บาง resources:
# ใช้ terraform state mv เพื่อ restructure ก่อน
```

### การเพิ่ม count อย่างปลอดภัย

```hcl
# ✅ การเพิ่ม count ปลอดภัยกว่า (เพิ่ม resources ใหม่)
variable "subnet_count" {
  description = "จำนวน subnets (เพิ่มได้ ลดระวัง)"
  type        = number
  default     = 3
  
  validation {
    condition     = var.subnet_count >= 1 && var.subnet_count <= 6
    error_message = "Subnet count must be between 1 and 6."
  }
}
```

### การ migrate จาก count ไป for_each

```hcl
# เดิม: ใช้ count
resource "aws_subnet" "private" {
  count  = 3
  vpc_id = aws_vpc.main.id
  cidr_block = "10.0.${count.index + 10}.0/24"
}

# ใหม่: ต้องการใช้ for_each
# ถ้าเปลี่ยนตรงๆ Terraform จะ destroy และสร้างใหม่!

# ขั้นตอน migration:
# 1. รัน terraform state list เพื่อดู current state
# $ terraform state list
# aws_subnet.private[0]
# aws_subnet.private[1]
# aws_subnet.private[2]

# 2. ใช้ terraform state mv เพื่อ rename
# $ terraform state mv 'aws_subnet.private[0]' 'aws_subnet.private["private-1"]'
# $ terraform state mv 'aws_subnet.private[1]' 'aws_subnet.private["private-2"]'
# $ terraform state mv 'aws_subnet.private[2]' 'aws_subnet.private["private-3"]'

# 3. อัพเดต code ให้ใช้ for_each
resource "aws_subnet" "private" {
  for_each = {
    "private-1" = "10.0.11.0/24"
    "private-2" = "10.0.12.0/24"
    "private-3" = "10.0.13.0/24"
  }
  
  vpc_id     = aws_vpc.main.id
  cidr_block = each.value
  
  tags = { Name = each.key }
}

# 4. terraform plan ควร show "No changes" ถ้าทำถูกต้อง
```

### Best Practices สรุป

```hcl
# ✅ DO: ใช้ for_each สำหรับ resources ที่มี identifier ชัดเจน
resource "aws_iam_user" "users" {
  for_each = toset(var.usernames)
  name     = each.key
}

# ✅ DO: ใช้ count สำหรับ conditional resource
resource "aws_nat_gateway" "main" {
  count = var.create_nat ? 1 : 0
  # ...
}

# ✅ DO: ใช้ depends_on เมื่อมี hidden dependency เท่านั้น
resource "aws_instance" "app" {
  # ...
  depends_on = [aws_iam_role_policy.app]  # policy ไม่ได้ถูก reference โดยตรง
}

# ✅ DO: ใช้ lifecycle.prevent_destroy สำหรับ critical resources
resource "aws_rds_cluster" "prod" {
  # ...
  lifecycle {
    prevent_destroy = true
  }
}

# ✅ DO: ใช้ lifecycle.ignore_changes เมื่อมีการจัดการ attribute จากภายนอก
resource "aws_instance" "managed" {
  # ...
  lifecycle {
    ignore_changes = [user_data, ami]
  }
}

# ❌ DON'T: อย่าใช้ provisioner ถ้าหลีกเลี่ยงได้
# ❌ DON'T: อย่าใช้ depends_on กับ reference ที่มีอยู่แล้ว
# ❌ DON'T: อย่าลด count โดยไม่ review plan ก่อน
# ❌ DON'T: อย่าใช้ index เป็น key ของ for_each
```

---

## Summary Table: Meta-Arguments

| Meta-Argument | Purpose | When to Use |
|--------------|---------|-------------|
| `depends_on` | Explicit ordering | Hidden dependencies ที่ไม่มี reference |
| `count` | Multiply resources | N identical resources, conditional |
| `for_each` | Iterate collection | Resources ที่ต่างกัน, stable keys |
| `provider` | Override provider | Multi-region, multi-account |
| `lifecycle` | Control behavior | Protect resources, ignore external changes |
| `provisioner` | Run scripts | Last resort only (deprecated pattern) |

---

## แบบฝึกหัด (Exercises)

### Exercise 1: Multi-Environment Setup ด้วย for_each

สร้าง infrastructure สำหรับ 3 environments (dev/staging/prod) ที่มี:
- S3 bucket สำหรับแต่ละ environment
- IAM role สำหรับแต่ละ environment
- Resource naming ที่สม่ำเสมอ

```hcl
variable "environments" {
  type = map(object({
    instance_type = string
    min_capacity  = number
    max_capacity  = number
  }))
  default = {
    dev = {
      instance_type = "t3.micro"
      min_capacity  = 1
      max_capacity  = 2
    }
    staging = {
      instance_type = "t3.small"
      min_capacity  = 2
      max_capacity  = 4
    }
    prod = {
      instance_type = "t3.medium"
      min_capacity  = 3
      max_capacity  = 10
    }
  }
}

# TODO: สร้าง S3 buckets, IAM roles สำหรับแต่ละ environment
```

### Exercise 2: Conditional Resources

สร้าง infrastructure ที่:
- สร้าง NAT Gateway เฉพาะ prod environment
- สร้าง CloudWatch alarms เฉพาะเมื่อ enable_monitoring = true
- ป้องกัน prod database จากการถูกลบ

```hcl
variable "environment"       { default = "dev" }
variable "enable_monitoring" { default = false }

# TODO: Implement conditional resources
```

### Exercise 3: State Migration

Migrate resources จาก count ไป for_each โดยไม่ destroy:
1. Identify current state structure
2. Plan state mv commands
3. Update code
4. Verify no changes in plan

---

## Checklist

- [ ] เข้าใจความแตกต่างระหว่าง implicit vs explicit dependencies
- [ ] รู้ว่าเมื่อไรควรใช้ depends_on
- [ ] เข้าใจ count.index และการ reference resources
- [ ] รู้จัก conditional creation pattern ด้วย count
- [ ] เข้าใจ for_each กับ set และ map
- [ ] สามารถ migrate จาก count ไป for_each ได้
- [ ] เข้าใจ lifecycle options ทุกตัว
- [ ] รู้ว่าเมื่อไรควรใช้ prevent_destroy
- [ ] เข้าใจ ignore_changes และกรณีที่ใช้
- [ ] รู้ข้อดีข้อเสียของ provisioner
