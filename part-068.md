# Part 068: Lifecycle Rules & Resource Control
## Lifecycle Rules และการควบคุม Resources
### Steps 671-680

---

## บทนำ (Introduction)

`lifecycle` block ใน Terraform ช่วยให้ควบคุมพฤติกรรมการสร้าง อัพเดต และลบ resources ได้อย่างละเอียด การเข้าใจ lifecycle rules จะช่วยป้องกันข้อผิดพลาดที่อาจเกิดขึ้นกับ production resources

---

## Step 671: Lifecycle Block Basics

### โครงสร้างของ lifecycle block

```hcl
# ==========================================
# LIFECYCLE BLOCK POSITION
# ==========================================

resource "aws_instance" "web" {
  ami           = "ami-12345"
  instance_type = "t3.micro"

  tags = {
    Name = "web-server"
  }

  # lifecycle block ต้องอยู่ภายใน resource block
  lifecycle {
    create_before_destroy = false
    prevent_destroy       = false
    ignore_changes        = []
    # replace_triggered_by (Terraform 1.2+)
  }
}

# lifecycle arguments ที่ใช้ได้:
# - create_before_destroy: bool
# - prevent_destroy: bool
# - ignore_changes: list of attributes
# - replace_triggered_by: list of references (1.2+)
# - precondition: validation block (1.2+)
# - postcondition: validation block (1.2+)
```

---

## Step 672: create_before_destroy

### การสร้างก่อนทำลาย

```hcl
# ==========================================
# CREATE_BEFORE_DESTROY PATTERNS
# ==========================================

# Pattern 1: Blue-Green Deployment
# EC2 instance ที่อยู่ใน Auto Scaling Group
resource "aws_launch_template" "app" {
  name_prefix   = "app-"
  image_id      = var.ami_id
  instance_type = var.instance_type

  # เมื่อ AMI เปลี่ยน ต้องสร้าง launch template ใหม่ก่อน
  # แล้วค่อยลบอันเก่า
  lifecycle {
    create_before_destroy = true
  }
}

# Pattern 2: SSL Certificate - Zero Downtime
resource "aws_acm_certificate" "main" {
  domain_name       = var.domain_name
  validation_method = "DNS"

  # สร้าง cert ใหม่ก่อน validate แล้วค่อย attach กับ ALB
  # จากนั้นลบ cert เก่า
  lifecycle {
    create_before_destroy = true
  }
}

# Pattern 3: Security Group - ที่ ALB ใช้อยู่
resource "aws_security_group" "alb" {
  name        = "alb-sg-${var.environment}"
  description = "Security group for ALB"
  vpc_id      = var.vpc_id

  # ALB ต้องมี SG ตลอดเวลา ถ้าลบก่อนสร้างใหม่จะ error
  lifecycle {
    create_before_destroy = true
  }
}

# Pattern 4: RDS Subnet Group
resource "aws_db_subnet_group" "main" {
  name       = "${var.project}-db-subnet-group"
  subnet_ids = var.subnet_ids

  lifecycle {
    create_before_destroy = true
  }
}

# ==========================================
# WHY create_before_destroy IS NEEDED
# ==========================================

# ปกติ Terraform จะ: DESTROY เก่า -> CREATE ใหม่
# ถ้า resource อื่นขึ้นอยู่กับมันอยู่ -> ERROR

# ด้วย create_before_destroy: CREATE ใหม่ -> UPDATE references -> DESTROY เก่า
# ไม่มี downtime!

# ==========================================
# create_before_destroy กับ ALB + ASG
# ==========================================

# ALB Listener -> Target Group
resource "aws_lb_target_group" "blue" {
  name     = "${var.project}-blue-tg"
  port     = 80
  protocol = "HTTP"
  vpc_id   = var.vpc_id

  # ต้องสร้างก่อน update listener rule
  lifecycle {
    create_before_destroy = true
  }
}

resource "aws_lb_listener_rule" "main" {
  listener_arn = aws_lb_listener.main.arn

  action {
    type             = "forward"
    target_group_arn = aws_lb_target_group.blue.arn
  }

  condition {
    path_pattern {
      values = ["/api/*"]
    }
  }
}

# ==========================================
# create_before_destroy กับ DEPENDENCIES
# ==========================================

# เมื่อ resource A มี create_before_destroy = true
# resources ที่ขึ้นอยู่กับ A จะต้องมี create_before_destroy = true ด้วย

resource "aws_security_group" "app" {
  lifecycle {
    create_before_destroy = true
  }
}

resource "aws_network_interface" "app" {
  security_groups = [aws_security_group.app.id]

  # ต้องมีด้วยเพราะขึ้นอยู่กับ SG ที่มี create_before_destroy
  lifecycle {
    create_before_destroy = true
  }
}
```

---

## Step 673: prevent_destroy

### การป้องกันการลบ resources

```hcl
# ==========================================
# PREVENT_DESTROY - PROTECTING CRITICAL RESOURCES
# ==========================================

# Pattern 1: Database Protection
resource "aws_db_instance" "production" {
  identifier     = "prod-database"
  engine         = "postgres"
  instance_class = "db.r6g.large"
  # ...

  lifecycle {
    prevent_destroy = true
    # ถ้าใครพยายาม terraform destroy หรือ remove resource block นี้
    # Terraform จะ error:
    # Error: Instance cannot be destroyed
    # Resource ... has lifecycle.prevent_destroy set
  }
}

# Pattern 2: S3 State Bucket
resource "aws_s3_bucket" "terraform_state" {
  bucket = "mycompany-terraform-state"

  lifecycle {
    prevent_destroy = true
  }
}

# Pattern 3: KMS Key
resource "aws_kms_key" "main" {
  description             = "Main encryption key"
  deletion_window_in_days = 30  # AWS requires 7-30 days

  lifecycle {
    prevent_destroy = true
    # KMS keys ที่ใช้ encrypt data สำคัญต้องป้องกันการลบ
  }
}

# Pattern 4: Production VPC
resource "aws_vpc" "production" {
  cidr_block = "10.0.0.0/16"

  lifecycle {
    prevent_destroy = var.environment == "prod"
    # ป้องกันเฉพาะ prod
  }
}

# ==========================================
# PREVENT_DESTROY + ENVIRONMENT CHECK
# ==========================================

variable "environment" {
  type    = string
  default = "dev"
}

resource "aws_elasticache_cluster" "main" {
  cluster_id           = "my-cache"
  engine               = "redis"
  node_type            = "cache.t3.micro"
  num_cache_nodes      = 1
  parameter_group_name = "default.redis7"

  lifecycle {
    prevent_destroy = var.environment == "prod"
    # ป้องกันเฉพาะ prod, dev/staging สามารถลบได้
  }
}

# ==========================================
# HOW TO BYPASS PREVENT_DESTROY (เมื่อจำเป็น)
# ==========================================

# 1. เอา prevent_destroy ออกจาก resource block
# 2. terraform apply (เพื่อ update state)
# 3. terraform destroy (ทำได้แล้ว)

# หรือ
# 1. เอา resource block ออก และเพิ่ม removed block (Terraform 1.5+)
removed {
  from = aws_db_instance.production

  lifecycle {
    destroy = false  # Remove from state without destroying
  }
}
```

---

## Step 674: ignore_changes

### การ ignore การเปลี่ยนแปลงจาก outside Terraform

```hcl
# ==========================================
# IGNORE_CHANGES PATTERNS
# ==========================================

# Pattern 1: AMI managed outside Terraform
resource "aws_instance" "app" {
  ami           = var.ami_id
  instance_type = "t3.medium"

  lifecycle {
    ignore_changes = [
      ami,  # AMI อาจถูก update โดย patch management system
    ]
  }
}

# Pattern 2: Tags added by other tools (Cost allocation, compliance)
resource "aws_instance" "web" {
  ami           = var.ami_id
  instance_type = "t3.micro"

  tags = {
    Name        = "web-server"
    Environment = "prod"
  }

  lifecycle {
    ignore_changes = [
      tags,  # Tags อาจถูกเพิ่มโดย AWS Config, Security Hub, ฯลฯ
      # หรือระบุ specific tags:
      # tags["CostCenter"],
      # tags["CreatedBy"],
    ]
  }
}

# Pattern 3: Auto-scaled count
resource "aws_autoscaling_group" "web" {
  name             = "web-asg"
  desired_capacity = var.desired_count  # Initial value
  min_size         = 1
  max_size         = 10

  lifecycle {
    ignore_changes = [
      desired_capacity,  # Auto Scaling อาจปรับ desired_count
    ]
  }
}

# Pattern 4: Password managed by rotation
resource "aws_db_instance" "main" {
  identifier = "my-db"
  password   = var.db_password  # Initial password

  lifecycle {
    ignore_changes = [
      password,  # Password rotation managed by Secrets Manager
    ]
  }
}

# Pattern 5: Multiple attributes
resource "aws_eks_node_group" "main" {
  cluster_name    = aws_eks_cluster.main.name
  node_group_name = "main-nodes"
  node_role_arn   = aws_iam_role.node.arn
  subnet_ids      = var.private_subnet_ids

  scaling_config {
    desired_size = var.node_desired
    min_size     = var.node_min
    max_size     = var.node_max
  }

  lifecycle {
    ignore_changes = [
      scaling_config[0].desired_size,  # Managed by Cluster Autoscaler
      # labels,  # Labels managed by other tools
    ]
  }
}

# ==========================================
# IGNORE_CHANGES = ALL
# ==========================================

# ใช้เมื่อ resource ทั้งหมดถูก manage นอก Terraform
resource "aws_instance" "imported" {
  ami           = "ami-12345"
  instance_type = "t3.micro"

  lifecycle {
    ignore_changes = all
    # Terraform จะไม่ plan changes ใดๆ กับ resource นี้
    # ใช้เป็น "adopt" resource ที่ existing
  }
}

# ==========================================
# WHEN TO USE IGNORE_CHANGES
# ==========================================

# USE: เมื่อ resource attribute ถูก manage โดย system อื่น
# AVOID: เมื่อต้องการให้ Terraform มี full control
# CAREFUL: ถ้า ignore_changes = all อาจทำให้ drift ใหญ่

resource "aws_ecs_service" "app" {
  name            = "my-service"
  cluster         = aws_ecs_cluster.main.id
  task_definition = aws_ecs_task_definition.app.arn
  desired_count   = var.desired_count

  lifecycle {
    ignore_changes = [
      task_definition,  # Deployed via CI/CD pipeline
      desired_count,    # Scaled by Application Auto Scaling
    ]
  }
}
```

---

## Step 675: replace_triggered_by

### บังคับ replacement เมื่อ dependency เปลี่ยน (Terraform 1.2+)

```hcl
# ==========================================
# REPLACE_TRIGGERED_BY (Terraform 1.2+)
# ==========================================

# Pattern 1: EC2 instance ต้องถูกสร้างใหม่เมื่อ user data เปลี่ยน
resource "aws_instance" "web" {
  ami           = var.ami_id
  instance_type = "t3.micro"
  user_data     = file("${path.module}/userdata.sh")

  lifecycle {
    # สร้างใหม่เมื่อ user_data เปลี่ยน (แทนที่จะ in-place update)
    replace_triggered_by = [
      null_resource.user_data_hash
    ]
  }
}

resource "null_resource" "user_data_hash" {
  triggers = {
    hash = filesha256("${path.module}/userdata.sh")
  }
}

# Pattern 2: ECS Service restart เมื่อ config เปลี่ยน
resource "aws_ecs_service" "app" {
  name            = "my-service"
  cluster         = aws_ecs_cluster.main.id
  task_definition = aws_ecs_task_definition.app.arn
  desired_count   = 2

  lifecycle {
    # Force redeploy เมื่อ task definition เปลี่ยน
    replace_triggered_by = [
      aws_ecs_task_definition.app.revision
    ]
  }
}

# Pattern 3: Rolling replace เมื่อ certificate เปลี่ยน
resource "aws_lb_listener" "https" {
  load_balancer_arn = aws_lb.main.arn
  port              = 443
  protocol          = "HTTPS"
  ssl_policy        = "ELBSecurityPolicy-TLS13-1-2-2021-06"
  certificate_arn   = aws_acm_certificate_validation.main.certificate_arn

  default_action {
    type             = "forward"
    target_group_arn = aws_lb_target_group.main.arn
  }

  lifecycle {
    replace_triggered_by = [
      aws_acm_certificate.main
    ]
  }
}

# Pattern 4: Replace เมื่อ variable เปลี่ยน
variable "app_version" {
  type = string
}

resource "terraform_data" "version_trigger" {
  input = var.app_version  # Terraform 1.4+
}

resource "aws_instance" "app" {
  ami           = var.ami_id
  instance_type = "t3.micro"

  lifecycle {
    replace_triggered_by = [
      terraform_data.version_trigger
    ]
  }
}
```

---

## Step 676: precondition and postcondition

### การตรวจสอบ conditions ใน resources

```hcl
# ==========================================
# PRECONDITION IN RESOURCES (Terraform 1.2+)
# ==========================================

# precondition ทำงานตอน plan phase
# ถ้า condition = false -> plan fails with error_message

resource "aws_db_instance" "main" {
  identifier     = var.db_identifier
  engine         = var.db_engine
  instance_class = var.db_instance_class
  # ...

  lifecycle {
    # ตรวจสอบก่อน apply: production ต้องใช้ multi_az
    precondition {
      condition     = var.environment != "prod" || var.multi_az == true
      error_message = "Production databases must use Multi-AZ for high availability."
    }

    # ตรวจสอบ: production ต้องใช้ deletion protection
    precondition {
      condition     = var.environment != "prod" || var.deletion_protection == true
      error_message = "Production databases must have deletion_protection enabled."
    }
  }
}

resource "aws_s3_bucket_policy" "main" {
  bucket = aws_s3_bucket.main.id
  policy = data.aws_iam_policy_document.bucket_policy.json

  lifecycle {
    # ตรวจสอบว่าไม่ใช้ wildcard principal ใน production
    precondition {
      condition = !(
        var.environment == "prod" &&
        can(regex("\"Principal\": \"\\*\"", data.aws_iam_policy_document.bucket_policy.json))
      )
      error_message = "Production S3 bucket policies must not use wildcard (*) principals."
    }
  }
}

# ==========================================
# POSTCONDITION IN RESOURCES
# ==========================================

# postcondition ทำงานหลัง apply
# ตรวจสอบว่า resource ที่สร้างมีค่าถูกต้อง

resource "aws_instance" "app" {
  ami           = var.ami_id
  instance_type = var.instance_type

  lifecycle {
    # ตรวจสอบหลัง create: ต้องมี private IP
    postcondition {
      condition     = self.private_ip != ""
      error_message = "EC2 instance must have a private IP address after creation."
    }

    # ตรวจสอบ: instance ต้องอยู่ใน VPC ที่ถูกต้อง
    postcondition {
      condition     = self.subnet_id != ""
      error_message = "EC2 instance must be in a subnet."
    }
  }
}

resource "aws_lb" "main" {
  name    = "${var.project}-alb"
  subnets = var.subnet_ids

  lifecycle {
    # ตรวจสอบหลัง create: ALB ต้องมี DNS name
    postcondition {
      condition     = self.dns_name != ""
      error_message = "Load balancer must have a DNS name after creation."
    }
  }
}

# ==========================================
# PRECONDITION IN DATA SOURCES
# ==========================================

data "aws_ami" "ubuntu" {
  most_recent = true
  owners      = ["099720109477"]

  filter {
    name   = "name"
    values = ["ubuntu/images/hvm-ssd/ubuntu-jammy-22.04-amd64-server-*"]
  }

  lifecycle {
    # ตรวจสอบว่า AMI ที่ได้ไม่เก่าเกิน 90 วัน
    postcondition {
      condition = timecmp(
        self.creation_date,
        timeadd(timestamp(), "-2160h")  # 90 days = 2160 hours
      ) > 0
      error_message = "The selected Ubuntu AMI is more than 90 days old. Please update your AMI filter."
    }
  }
}
```

---

## Step 677: Custom Conditions in Resources

### การสร้าง custom conditions ที่ซับซ้อน

```hcl
# ==========================================
# COMPLEX CUSTOM CONDITIONS
# ==========================================

# 1. ตรวจสอบ production readiness
resource "aws_elasticache_replication_group" "main" {
  replication_group_id = "${var.project}-${var.environment}-cache"
  description          = "Redis cache"

  node_type            = var.cache_node_type
  num_node_groups      = var.cache_shard_count
  replicas_per_node_group = var.cache_replicas

  lifecycle {
    # Production ต้องมีอย่างน้อย 2 replicas
    precondition {
      condition = (
        var.environment != "prod" ||
        var.cache_replicas >= 2
      )
      error_message = "Production Redis must have at least 2 replicas per shard."
    }

    # Production ต้องมีอย่างน้อย 2 shards
    precondition {
      condition = (
        var.environment != "prod" ||
        var.cache_shard_count >= 2
      )
      error_message = "Production Redis must have at least 2 shards for high availability."
    }

    # ตรวจสอบ encryption
    precondition {
      condition = (
        var.environment != "prod" ||
        var.at_rest_encryption_enabled
      )
      error_message = "Production Redis must have at-rest encryption enabled."
    }
  }
}

# 2. ตรวจสอบ network configuration
resource "aws_db_instance" "secure" {
  identifier     = "${var.project}-${var.environment}-db"
  engine         = "postgres"
  instance_class = var.db_class

  lifecycle {
    # ห้าม publicly_accessible ใน production
    precondition {
      condition = !(
        var.environment == "prod" &&
        var.publicly_accessible
      )
      error_message = "Production databases must not be publicly accessible."
    }

    # ต้องมี subnet group
    precondition {
      condition     = var.db_subnet_group_name != ""
      error_message = "Database must use a DB subnet group (not default VPC)."
    }

    # ต้องมี security groups
    precondition {
      condition     = length(var.security_group_ids) > 0
      error_message = "Database must be associated with at least one security group."
    }
  }
}

# 3. Business rule validation
resource "aws_s3_bucket" "data" {
  bucket = "${var.project}-${var.environment}-data"

  lifecycle {
    # Production ต้องมี versioning prefix
    precondition {
      condition = (
        var.environment != "prod" ||
        startswith(var.project, "prod-") ||
        contains(["billing", "analytics", "compliance"], var.data_classification)
      )
      error_message = "Production data buckets must be classified as billing, analytics, or compliance."
    }
  }
}
```

---

## Step 678: Lifecycle and count/for_each

### ปฏิสัมพันธ์ระหว่าง lifecycle และ count/for_each

```hcl
# ==========================================
# LIFECYCLE WITH COUNT
# ==========================================

resource "aws_instance" "web" {
  count = var.instance_count

  ami           = var.ami_id
  instance_type = "t3.micro"

  lifecycle {
    create_before_destroy = true
    prevent_destroy       = var.environment == "prod"

    ignore_changes = [
      ami,
      tags["LastUpdated"],
    ]
  }
}

# ==========================================
# LIFECYCLE WITH FOR_EACH
# ==========================================

variable "databases" {
  type = map(object({
    engine       = string
    protect      = bool
  }))
}

resource "aws_db_instance" "dbs" {
  for_each = var.databases

  identifier     = each.key
  engine         = each.value.engine
  instance_class = "db.t3.micro"

  lifecycle {
    # prevent_destroy สามารถเป็น expression ที่อ้างอิง each.value
    prevent_destroy = each.value.protect

    create_before_destroy = true

    ignore_changes = [
      password,
      engine_version,
    ]
  }
}

# ==========================================
# LIFECYCLE WITH DEPENDS_ON
# ==========================================

resource "aws_security_group" "app" {
  name   = "app-sg"
  vpc_id = var.vpc_id

  lifecycle {
    create_before_destroy = true
  }
}

resource "aws_instance" "app" {
  ami                    = var.ami_id
  instance_type          = "t3.micro"
  vpc_security_group_ids = [aws_security_group.app.id]

  depends_on = [aws_security_group.app]

  lifecycle {
    # ต้องมี create_before_destroy ด้วยเนื่องจาก SG มี
    create_before_destroy = true
  }
}
```

---

## Step 679: Lifecycle and Dependencies

### ความสัมพันธ์ระหว่าง lifecycle และ dependencies

```hcl
# ==========================================
# LIFECYCLE DEPENDENCY CHAIN
# ==========================================

# Level 1: Security Group
resource "aws_security_group" "alb" {
  name   = "${var.project}-alb-sg"
  vpc_id = var.vpc_id

  lifecycle {
    create_before_destroy = true
    # ALB SG เปลี่ยนไม่ได้โดยตรง ต้องสร้างใหม่
  }
}

# Level 2: ALB ขึ้นอยู่กับ SG
resource "aws_lb" "main" {
  name            = "${var.project}-alb"
  security_groups = [aws_security_group.alb.id]
  subnets         = var.public_subnet_ids

  lifecycle {
    create_before_destroy = true  # ต้องมีด้วยเพราะ alb_sg มี
  }
}

# Level 3: Listener ขึ้นอยู่กับ ALB
resource "aws_lb_listener" "https" {
  load_balancer_arn = aws_lb.main.arn
  port              = 443
  protocol          = "HTTPS"

  default_action {
    type = "fixed-response"
    fixed_response {
      content_type = "text/plain"
      status_code  = "404"
    }
  }

  lifecycle {
    create_before_destroy = true  # chain ต้องสอดคล้องกัน
  }
}

# ==========================================
# LIFECYCLE ANTI-PATTERN
# ==========================================

# PROBLEM: SG มี create_before_destroy แต่ instance ไม่มี
resource "aws_security_group" "bad_sg" {
  lifecycle { create_before_destroy = true }
}

resource "aws_instance" "bad_instance" {
  vpc_security_group_ids = [aws_security_group.bad_sg.id]
  # lifecycle ไม่มี create_before_destroy
  # จะ ERROR เมื่อ SG ต้องถูก recreate!
}

# SOLUTION: ทุก resource ใน chain ต้องมี
resource "aws_instance" "good_instance" {
  vpc_security_group_ids = [aws_security_group.bad_sg.id]
  lifecycle { create_before_destroy = true }  # เพิ่มด้วย
}
```

---

## Step 680: Complete Lifecycle Examples

### ตัวอย่างสมบูรณ์

```hcl
# ==========================================
# COMPLETE PRODUCTION DATABASE LIFECYCLE
# ==========================================

resource "aws_db_instance" "production" {
  identifier     = "prod-postgres-01"
  engine         = "postgres"
  engine_version = "15.4"
  instance_class = "db.r6g.large"

  allocated_storage     = 100
  max_allocated_storage = 1000
  storage_type          = "gp3"
  storage_encrypted     = true
  kms_key_id            = aws_kms_key.rds.arn

  db_name  = "appdb"
  username = "admin"
  password = var.db_password

  db_subnet_group_name   = aws_db_subnet_group.main.name
  vpc_security_group_ids = [aws_security_group.rds.id]

  multi_az                = true
  publicly_accessible     = false
  backup_retention_period = 30
  backup_window           = "03:00-04:00"
  maintenance_window      = "Mon:04:00-Mon:05:00"

  deletion_protection          = true
  skip_final_snapshot          = false
  final_snapshot_identifier    = "prod-final-snapshot"
  copy_tags_to_snapshot        = true

  performance_insights_enabled = true
  monitoring_interval          = 60
  monitoring_role_arn          = aws_iam_role.rds_monitoring.arn

  tags = {
    Name        = "prod-postgres-01"
    Environment = "prod"
    ManagedBy   = "Terraform"
    Protect     = "true"
  }

  lifecycle {
    # ป้องกันการลบโดยไม่ตั้งใจ
    prevent_destroy = true

    # ignore password changes (managed by Secrets Manager rotation)
    ignore_changes = [
      password,
      engine_version,  # patch managed separately
    ]

    # ตรวจสอบก่อน apply
    precondition {
      condition     = var.multi_az == true
      error_message = "Production database must use Multi-AZ."
    }

    precondition {
      condition     = var.backup_retention_period >= 14
      error_message = "Production database must retain backups for at least 14 days."
    }

    # ตรวจสอบหลัง apply
    postcondition {
      condition     = self.status == "available"
      error_message = "Database is not in 'available' status after creation."
    }

    postcondition {
      condition     = self.multi_az == true
      error_message = "Multi-AZ was not enabled on the database."
    }
  }
}

# ==========================================
# COMPLETE APPLICATION INSTANCE LIFECYCLE
# ==========================================

resource "aws_launch_template" "app" {
  name_prefix   = "${var.project}-${var.environment}-"
  image_id      = var.ami_id
  instance_type = var.instance_type

  vpc_security_group_ids = [aws_security_group.app.id]

  iam_instance_profile {
    arn = aws_iam_instance_profile.app.arn
  }

  user_data = base64encode(templatefile("${path.module}/userdata.sh", {
    environment = var.environment
    app_version = var.app_version
  }))

  tag_specifications {
    resource_type = "instance"
    tags = {
      Name        = "${var.project}-${var.environment}"
      Environment = var.environment
      Version     = var.app_version
    }
  }

  lifecycle {
    # สร้าง launch template ใหม่ก่อนลบอันเก่า
    create_before_destroy = true
  }
}

resource "aws_autoscaling_group" "app" {
  name_prefix         = "${var.project}-${var.environment}-"
  desired_capacity    = var.desired_count
  min_size            = var.min_count
  max_size            = var.max_count
  vpc_zone_identifier = var.private_subnet_ids
  target_group_arns   = [aws_lb_target_group.app.arn]

  launch_template {
    id      = aws_launch_template.app.id
    version = "$Latest"
  }

  health_check_type         = "ELB"
  health_check_grace_period = 300

  tag {
    key                 = "Name"
    value               = "${var.project}-${var.environment}"
    propagate_at_launch = true
  }

  lifecycle {
    # Auto Scaling manages desired_capacity
    ignore_changes = [desired_capacity]

    # Vet the config before applying
    precondition {
      condition     = var.min_count <= var.desired_count
      error_message = "Desired count must be >= min count."
    }

    precondition {
      condition     = var.desired_count <= var.max_count
      error_message = "Desired count must be <= max count."
    }

    # Force replace when app_version changes
    replace_triggered_by = [
      aws_launch_template.app
    ]
  }
}
```

---

## สรุป (Summary)

### Lifecycle Arguments Quick Reference:

| Argument | เมื่อใช้ |
|----------|---------|
| `create_before_destroy` | Resources ที่ต้องไม่มี downtime |
| `prevent_destroy` | Critical resources ที่ไม่ควรลบ |
| `ignore_changes` | Attributes ที่จัดการโดย system อื่น |
| `replace_triggered_by` | Force replace เมื่อ dependency เปลี่ยน |
| `precondition` | Validate config ก่อน apply |
| `postcondition` | Verify resource หลัง create/update |

### Best Practices:
1. ใช้ `prevent_destroy = true` กับทุก production databases
2. ใช้ `create_before_destroy = true` กับ resources ที่ ALB/ASG ใช้อยู่
3. ใช้ `ignore_changes` เฉพาะสำหรับ attributes ที่จัดการ externally
4. ใช้ `precondition` แทน variable validation เมื่อต้องอ้างอิง resource values

---

*จบ Part 068 - Lifecycle Rules & Resource Control*
