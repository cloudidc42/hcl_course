# Part 061: Variables: Deep Dive & Validation
## ตัวแปร: เจาะลึกและการตรวจสอบความถูกต้อง
### Steps 601-610

---

## บทนำ (Introduction)

ในบทนี้เราจะเรียนรู้เรื่อง Terraform Variables อย่างลึกซึ้ง โดยเฉพาะ **validation blocks** ที่ช่วยให้เราสามารถตรวจสอบความถูกต้องของค่า input ก่อนที่ Terraform จะทำการ apply จริง

Variables ที่มี validation ที่ดีจะช่วย:
- จับ error ได้เร็ว (fail fast)
- ให้ error message ที่อ่านง่ายและมีประโยชน์
- ป้องกัน infrastructure ที่ผิดพลาดจาก misconfiguration

---

## Step 601: ทบทวน Variable Types (Variable Types Review)

### ประเภทของ Variable ทั้งหมดใน Terraform

```hcl
# ==========================================
# PRIMITIVE TYPES - ประเภทพื้นฐาน
# ==========================================

# string - ข้อความ
variable "environment" {
  type        = string
  description = "สภาพแวดล้อม (dev, staging, prod)"
  default     = "dev"
}

# number - ตัวเลข (integer หรือ float)
variable "instance_count" {
  type        = number
  description = "จำนวน instance ที่ต้องการสร้าง"
  default     = 1
}

# bool - ค่า true/false
variable "enable_monitoring" {
  type        = bool
  description = "เปิดใช้งาน monitoring หรือไม่"
  default     = true
}

# ==========================================
# COLLECTION TYPES - ประเภทที่เก็บหลายค่า
# ==========================================

# list(string) - รายการของ string
variable "availability_zones" {
  type        = list(string)
  description = "รายการ Availability Zones"
  default     = ["ap-southeast-1a", "ap-southeast-1b"]
}

# list(number) - รายการของตัวเลข
variable "allowed_ports" {
  type        = list(number)
  description = "พอร์ตที่อนุญาต"
  default     = [80, 443, 8080]
}

# map(string) - แผนที่ key-value ของ string
variable "common_tags" {
  type        = map(string)
  description = "Tags ทั่วไปสำหรับทุก resource"
  default = {
    Project   = "MyProject"
    ManagedBy = "Terraform"
  }
}

# map(number) - แผนที่ key-value ของตัวเลข
variable "instance_counts" {
  type        = map(number)
  description = "จำนวน instance ต่อ environment"
  default = {
    dev     = 1
    staging = 2
    prod    = 5
  }
}

# set(string) - เซตของ string (ไม่ซ้ำ ไม่มีลำดับ)
variable "enabled_regions" {
  type        = set(string)
  description = "Regions ที่เปิดใช้งาน"
  default     = ["ap-southeast-1", "us-east-1"]
}

# ==========================================
# STRUCTURAL TYPES - ประเภทที่มีโครงสร้าง
# ==========================================

# object - object ที่มี schema กำหนดไว้
variable "server_config" {
  type = object({
    instance_type = string
    disk_size_gb  = number
    enable_backup = bool
    tags          = map(string)
  })
  description = "การกำหนดค่า server"
  default = {
    instance_type = "t3.micro"
    disk_size_gb  = 20
    enable_backup = true
    tags          = {}
  }
}

# tuple - tuple ที่มี type กำหนดสำหรับแต่ละตำแหน่ง
variable "aws_account_info" {
  type        = tuple([string, string, number])
  description = "ข้อมูล AWS account: [account_id, region, vpc_count]"
  default     = ["123456789012", "ap-southeast-1", 3]
}

# any - ยอมรับทุก type
variable "flexible_config" {
  type        = any
  description = "การกำหนดค่าที่ยืดหยุ่น"
  default     = {}
}
```

---

## Step 602: Validation Blocks พื้นฐาน (Basic Validation Blocks)

### โครงสร้างของ validation block

```hcl
variable "variable_name" {
  type        = <type>
  description = "<description>"

  validation {
    condition     = <expression_that_returns_bool>
    error_message = "<message shown when condition is false>"
  }
}
```

### กฎสำคัญของ validation:
- `condition` ต้องคืนค่า `true` หรือ `false`
- ถ้า `condition` คืน `true` = ค่านั้นถูกต้อง (valid)
- ถ้า `condition` คืน `false` = Terraform จะแสดง `error_message`
- `error_message` ต้องขึ้นต้นด้วย capital letter และลงท้ายด้วย period หรือ exclamation

```hcl
# ตัวอย่างพื้นฐาน - validation สำหรับ environment
variable "environment" {
  type        = string
  description = "สภาพแวดล้อม (dev, staging, prod)"

  validation {
    condition     = contains(["dev", "staging", "prod"], var.environment)
    error_message = "Environment must be one of: dev, staging, prod."
  }
}

# ตัวอย่าง - validation สำหรับ port number
variable "app_port" {
  type        = number
  description = "Port ของ application"

  validation {
    condition     = var.app_port >= 1 && var.app_port <= 65535
    error_message = "Port must be between 1 and 65535."
  }
}
```

---

## Step 603: Multiple Validations per Variable (หลาย validation ต่อตัวแปร)

สามารถใส่ validation หลาย block ในตัวแปรเดียวได้:

```hcl
variable "app_name" {
  type        = string
  description = "ชื่อของ application"

  # Validation 1: ตรวจสอบความยาว
  validation {
    condition     = length(var.app_name) >= 3
    error_message = "App name must be at least 3 characters long."
  }

  # Validation 2: ตรวจสอบความยาวสูงสุด
  validation {
    condition     = length(var.app_name) <= 20
    error_message = "App name must not exceed 20 characters."
  }

  # Validation 3: ตรวจสอบ format (lowercase, numbers, hyphens only)
  validation {
    condition     = can(regex("^[a-z0-9-]+$", var.app_name))
    error_message = "App name may only contain lowercase letters, numbers, and hyphens."
  }

  # Validation 4: ต้องไม่ขึ้นต้นหรือลงท้ายด้วย hyphen
  validation {
    condition     = !startswith(var.app_name, "-") && !endswith(var.app_name, "-")
    error_message = "App name must not start or end with a hyphen."
  }
}

# ตัวอย่างที่ซับซ้อนขึ้น - validating an instance type
variable "instance_type" {
  type        = string
  description = "EC2 instance type"
  default     = "t3.micro"

  # Validation 1: ต้องอยู่ใน allowed families
  validation {
    condition = contains(
      ["t2", "t3", "t3a", "m5", "m5a", "c5", "c5a", "r5"],
      split(".", var.instance_type)[0]
    )
    error_message = "Instance type family must be one of: t2, t3, t3a, m5, m5a, c5, c5a, r5."
  }

  # Validation 2: ต้องมี size ที่ถูกต้อง
  validation {
    condition = contains(
      ["nano", "micro", "small", "medium", "large", "xlarge", "2xlarge", "4xlarge"],
      split(".", var.instance_type)[1]
    )
    error_message = "Instance size must be one of: nano, micro, small, medium, large, xlarge, 2xlarge, 4xlarge."
  }
}
```

---

## Step 604: CIDR Validation Patterns

### การตรวจสอบ CIDR range

```hcl
# ==========================================
# CIDR Validation - การตรวจสอบ CIDR Block
# ==========================================

# validation พื้นฐาน - ตรวจสอบว่าเป็น valid CIDR
variable "vpc_cidr" {
  type        = string
  description = "CIDR block สำหรับ VPC"
  default     = "10.0.0.0/16"

  validation {
    condition     = can(cidrhost(var.vpc_cidr, 0))
    error_message = "VPC CIDR block must be a valid IPv4 CIDR notation (e.g., 10.0.0.0/16)."
  }
}

# validation ที่ซับซ้อน - ตรวจสอบ prefix length
variable "subnet_cidr" {
  type        = string
  description = "CIDR block สำหรับ subnet"

  # ตรวจสอบว่าเป็น valid CIDR
  validation {
    condition     = can(cidrhost(var.subnet_cidr, 0))
    error_message = "Subnet CIDR must be a valid IPv4 CIDR notation."
  }

  # ตรวจสอบว่า prefix ไม่เล็กเกินไป (ต้องมีอย่างน้อย /28 = 16 IPs)
  validation {
    condition     = tonumber(split("/", var.subnet_cidr)[1]) <= 28
    error_message = "Subnet prefix must be /28 or larger (e.g., /24, /20, /16)."
  }

  # ตรวจสอบว่า prefix ไม่ใหญ่เกินไป (ไม่เกิน /8)
  validation {
    condition     = tonumber(split("/", var.subnet_cidr)[1]) >= 8
    error_message = "Subnet prefix must be /8 or smaller (e.g., /16, /20, /24)."
  }
}

# validation สำหรับ private CIDR ranges เท่านั้น
variable "private_cidr" {
  type        = string
  description = "CIDR block สำหรับ private network (ต้องเป็น RFC1918)"

  validation {
    condition     = can(cidrhost(var.private_cidr, 0))
    error_message = "CIDR must be a valid IPv4 CIDR notation."
  }

  validation {
    condition = (
      # 10.0.0.0/8 range
      startswith(var.private_cidr, "10.") ||
      # 172.16.0.0/12 range
      can(regex("^172\\.(1[6-9]|2[0-9]|3[01])\\.", var.private_cidr)) ||
      # 192.168.0.0/16 range
      startswith(var.private_cidr, "192.168.")
    )
    error_message = "CIDR must be a private RFC1918 address range (10.x.x.x, 172.16-31.x.x, or 192.168.x.x)."
  }
}

# validation สำหรับ list ของ CIDRs
variable "allowed_cidr_blocks" {
  type        = list(string)
  description = "รายการ CIDR blocks ที่อนุญาต"
  default     = []

  validation {
    condition     = alltrue([for cidr in var.allowed_cidr_blocks : can(cidrhost(cidr, 0))])
    error_message = "All CIDR blocks must be valid IPv4 CIDR notation."
  }
}

# ตัวอย่างครบถ้วน - VPC CIDR configuration
variable "vpc_network_config" {
  type = object({
    vpc_cidr         = string
    public_subnets   = list(string)
    private_subnets  = list(string)
    database_subnets = list(string)
  })

  validation {
    condition     = can(cidrhost(var.vpc_network_config.vpc_cidr, 0))
    error_message = "VPC CIDR must be a valid IPv4 CIDR notation."
  }

  validation {
    condition     = alltrue([for s in var.vpc_network_config.public_subnets : can(cidrhost(s, 0))])
    error_message = "All public subnet CIDRs must be valid IPv4 CIDR notation."
  }

  validation {
    condition     = alltrue([for s in var.vpc_network_config.private_subnets : can(cidrhost(s, 0))])
    error_message = "All private subnet CIDRs must be valid IPv4 CIDR notation."
  }

  validation {
    condition     = alltrue([for s in var.vpc_network_config.database_subnets : can(cidrhost(s, 0))])
    error_message = "All database subnet CIDRs must be valid IPv4 CIDR notation."
  }
}
```

---

## Step 605: Regex Validation Patterns

### การใช้ regex ในการตรวจสอบ

```hcl
# ==========================================
# REGEX VALIDATION PATTERNS
# ==========================================

# ตรวจสอบ naming convention ทั่วไป
variable "resource_name" {
  type        = string
  description = "ชื่อ resource (lowercase, numbers, hyphens เท่านั้น)"

  validation {
    condition     = can(regex("^[a-z0-9-]{3,20}$", var.resource_name))
    error_message = "Resource name must be 3-20 characters, lowercase letters, numbers, and hyphens only."
  }
}

# ตรวจสอบ email format
variable "admin_email" {
  type        = string
  description = "Email ของ administrator"

  validation {
    condition     = can(regex("^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\\.[a-zA-Z]{2,}$", var.admin_email))
    error_message = "Admin email must be a valid email address (e.g., admin@example.com)."
  }
}

# ตรวจสอบ AWS Region format
variable "aws_region" {
  type        = string
  description = "AWS Region (เช่น ap-southeast-1)"

  validation {
    condition     = can(regex("^[a-z]{2}-[a-z]+-[0-9]{1}$", var.aws_region))
    error_message = "AWS region must be in format like 'us-east-1' or 'ap-southeast-1'."
  }
}

# ตรวจสอบ semantic version
variable "app_version" {
  type        = string
  description = "Version ของ application ในรูปแบบ semver"

  validation {
    condition     = can(regex("^v?[0-9]+\\.[0-9]+\\.[0-9]+(-[a-zA-Z0-9.]+)?$", var.app_version))
    error_message = "App version must follow semantic versioning (e.g., 1.0.0 or v2.3.1-beta)."
  }
}

# ตรวจสอบ S3 bucket name
variable "s3_bucket_name" {
  type        = string
  description = "ชื่อ S3 bucket"

  # ต้อง lowercase, numbers, hyphens
  validation {
    condition     = can(regex("^[a-z0-9-]+$", var.s3_bucket_name))
    error_message = "S3 bucket name may only contain lowercase letters, numbers, and hyphens."
  }

  # ความยาว 3-63 characters
  validation {
    condition     = length(var.s3_bucket_name) >= 3 && length(var.s3_bucket_name) <= 63
    error_message = "S3 bucket name must be between 3 and 63 characters."
  }

  # ต้องไม่มี consecutive hyphens
  validation {
    condition     = !can(regex("--", var.s3_bucket_name))
    error_message = "S3 bucket name must not contain consecutive hyphens."
  }

  # ต้องไม่ขึ้นต้นหรือลงท้ายด้วย hyphen
  validation {
    condition     = !startswith(var.s3_bucket_name, "-") && !endswith(var.s3_bucket_name, "-")
    error_message = "S3 bucket name must not start or end with a hyphen."
  }

  # ต้องไม่เป็น IP address format
  validation {
    condition     = !can(regex("^[0-9]+\\.[0-9]+\\.[0-9]+\\.[0-9]+$", var.s3_bucket_name))
    error_message = "S3 bucket name must not be formatted as an IP address."
  }
}

# ตรวจสอบ AWS Account ID
variable "aws_account_id" {
  type        = string
  description = "AWS Account ID (12 digits)"

  validation {
    condition     = can(regex("^[0-9]{12}$", var.aws_account_id))
    error_message = "AWS account ID must be exactly 12 digits."
  }
}

# ตรวจสอบ IAM Role Name
variable "iam_role_name" {
  type        = string
  description = "ชื่อ IAM Role"

  validation {
    condition     = can(regex("^[a-zA-Z][a-zA-Z0-9_+=,.@-]{0,63}$", var.iam_role_name))
    error_message = "IAM role name must start with a letter and contain only alphanumeric characters and +=,.@-."
  }
}

# ตรวจสอบ Docker image tag
variable "docker_image" {
  type        = string
  description = "Docker image ในรูปแบบ repo:tag"

  validation {
    condition     = can(regex("^[a-zA-Z0-9./_-]+:[a-zA-Z0-9._-]+$", var.docker_image))
    error_message = "Docker image must be in format 'repository:tag' (e.g., nginx:1.21 or myrepo/myapp:latest)."
  }
}

# ตรวจสอบ Kubernetes namespace name
variable "k8s_namespace" {
  type        = string
  description = "Kubernetes namespace name"

  validation {
    condition     = can(regex("^[a-z0-9][a-z0-9-]{0,61}[a-z0-9]$", var.k8s_namespace))
    error_message = "Kubernetes namespace must start and end with alphanumeric characters, contain only lowercase letters, numbers, and hyphens, and be at most 63 characters."
  }
}
```

---

## Step 606: Enum and Contains Validation

### การตรวจสอบว่าค่าอยู่ในรายการที่กำหนด

```hcl
# ==========================================
# ENUM / CONTAINS VALIDATION
# ==========================================

# ตรวจสอบ environment
variable "environment" {
  type        = string
  description = "สภาพแวดล้อมการทำงาน"

  validation {
    condition     = contains(["dev", "staging", "prod"], var.environment)
    error_message = "Environment must be one of: dev, staging, prod."
  }
}

# ตรวจสอบ AWS Region จาก allowed list
variable "deployment_region" {
  type        = string
  description = "AWS Region สำหรับ deployment"

  validation {
    condition = contains([
      "us-east-1",
      "us-west-2",
      "eu-west-1",
      "ap-southeast-1",
      "ap-northeast-1"
    ], var.deployment_region)
    error_message = "Deployment region must be one of the approved regions: us-east-1, us-west-2, eu-west-1, ap-southeast-1, ap-northeast-1."
  }
}

# ตรวจสอบ log level
variable "log_level" {
  type        = string
  description = "ระดับการ log"
  default     = "INFO"

  validation {
    condition     = contains(["DEBUG", "INFO", "WARN", "ERROR", "FATAL"], var.log_level)
    error_message = "Log level must be one of: DEBUG, INFO, WARN, ERROR, FATAL."
  }
}

# ตรวจสอบ database engine
variable "db_engine" {
  type        = string
  description = "Database engine"

  validation {
    condition     = contains(["mysql", "postgres", "mariadb", "aurora", "aurora-mysql", "aurora-postgresql"], var.db_engine)
    error_message = "Database engine must be one of: mysql, postgres, mariadb, aurora, aurora-mysql, aurora-postgresql."
  }
}

# ตรวจสอบ storage class
variable "storage_class" {
  type        = string
  description = "S3 storage class"
  default     = "STANDARD"

  validation {
    condition = contains([
      "STANDARD",
      "REDUCED_REDUNDANCY",
      "STANDARD_IA",
      "ONEZONE_IA",
      "INTELLIGENT_TIERING",
      "GLACIER",
      "DEEP_ARCHIVE"
    ], var.storage_class)
    error_message = "Storage class must be a valid S3 storage class."
  }
}

# ตรวจสอบ list ทุก element อยู่ใน allowed values
variable "enabled_features" {
  type        = list(string)
  description = "รายการ features ที่เปิดใช้งาน"
  default     = []

  validation {
    condition = alltrue([
      for feature in var.enabled_features :
      contains(["caching", "compression", "encryption", "monitoring", "backup"], feature)
    ])
    error_message = "Each feature must be one of: caching, compression, encryption, monitoring, backup."
  }
}

# ตรวจสอบ map values อยู่ใน allowed values
variable "service_tiers" {
  type        = map(string)
  description = "Tier สำหรับแต่ละ service"
  default     = {}

  validation {
    condition = alltrue([
      for service, tier in var.service_tiers :
      contains(["free", "basic", "premium", "enterprise"], tier)
    ])
    error_message = "Each service tier must be one of: free, basic, premium, enterprise."
  }
}
```

---

## Step 607: Length Validation

### การตรวจสอบความยาว

```hcl
# ==========================================
# LENGTH VALIDATION - การตรวจสอบความยาว
# ==========================================

# ตรวจสอบความยาว string
variable "project_name" {
  type        = string
  description = "ชื่อโปรเจค"

  validation {
    condition     = length(var.project_name) >= 3 && length(var.project_name) <= 20
    error_message = "Project name must be between 3 and 20 characters."
  }
}

# ตรวจสอบความยาว list - ต้องมีอย่างน้อย N elements
variable "availability_zones" {
  type        = list(string)
  description = "รายการ Availability Zones (ต้องมีอย่างน้อย 2)"

  validation {
    condition     = length(var.availability_zones) >= 2
    error_message = "At least 2 availability zones must be specified for high availability."
  }

  validation {
    condition     = length(var.availability_zones) <= 3
    error_message = "At most 3 availability zones may be specified."
  }
}

# ตรวจสอบความยาว map - ต้องมี key อย่างน้อย N ตัว
variable "required_tags" {
  type        = map(string)
  description = "Tags ที่จำเป็น"

  validation {
    condition     = length(var.required_tags) >= 3
    error_message = "At least 3 tags must be provided: Environment, Project, and Owner."
  }
}

# ตรวจสอบว่า required keys มีอยู่ใน map
variable "resource_tags" {
  type        = map(string)
  description = "Tags สำหรับ resource"

  validation {
    condition = alltrue([
      for required_key in ["Environment", "Project", "Owner", "CostCenter"] :
      contains(keys(var.resource_tags), required_key)
    ])
    error_message = "Tags must include: Environment, Project, Owner, and CostCenter."
  }
}

# ตรวจสอบ password complexity
variable "db_password" {
  type        = string
  description = "Database password"
  sensitive   = true

  validation {
    condition     = length(var.db_password) >= 12
    error_message = "Database password must be at least 12 characters long."
  }

  validation {
    condition     = can(regex("[A-Z]", var.db_password))
    error_message = "Database password must contain at least one uppercase letter."
  }

  validation {
    condition     = can(regex("[a-z]", var.db_password))
    error_message = "Database password must contain at least one lowercase letter."
  }

  validation {
    condition     = can(regex("[0-9]", var.db_password))
    error_message = "Database password must contain at least one number."
  }

  validation {
    condition     = can(regex("[^a-zA-Z0-9]", var.db_password))
    error_message = "Database password must contain at least one special character."
  }
}
```

---

## Step 608: List Element Validation

### การตรวจสอบ elements ใน list

```hcl
# ==========================================
# LIST ELEMENT VALIDATION
# ==========================================

# ตรวจสอบทุก element ใน list ว่าเป็น valid CIDR
variable "security_group_cidr_blocks" {
  type        = list(string)
  description = "CIDR blocks สำหรับ security group rules"

  validation {
    condition     = alltrue([for cidr in var.security_group_cidr_blocks : can(cidrhost(cidr, 0))])
    error_message = "All CIDR blocks must be valid IPv4 CIDR notation."
  }
}

# ตรวจสอบทุก element ว่าไม่ซ้ำกัน (unique)
variable "subnet_cidrs" {
  type        = list(string)
  description = "CIDR blocks สำหรับ subnets (ต้องไม่ซ้ำ)"

  validation {
    condition     = length(var.subnet_cidrs) == length(toset(var.subnet_cidrs))
    error_message = "Subnet CIDRs must be unique."
  }
}

# ตรวจสอบทุก element ว่า match regex
variable "allowed_ssh_keys" {
  type        = list(string)
  description = "รายการ SSH public keys"

  validation {
    condition = alltrue([
      for key in var.allowed_ssh_keys :
      can(regex("^(ssh-rsa|ssh-ed25519|ecdsa-sha2-nistp256) [A-Za-z0-9+/=]+ ?.+$", key))
    ])
    error_message = "All SSH keys must be valid public keys in OpenSSH format."
  }
}

# ตรวจสอบ list ของ ports
variable "allowed_ports" {
  type        = list(number)
  description = "รายการ port ที่อนุญาต"

  validation {
    condition     = alltrue([for port in var.allowed_ports : port >= 1 && port <= 65535])
    error_message = "All ports must be between 1 and 65535."
  }

  validation {
    condition     = length(var.allowed_ports) == length(toset([for p in var.allowed_ports : tostring(p)]))
    error_message = "Duplicate ports are not allowed."
  }
}

# ตรวจสอบ list ของ email addresses
variable "notification_emails" {
  type        = list(string)
  description = "รายการ email สำหรับการแจ้งเตือน"

  validation {
    condition = alltrue([
      for email in var.notification_emails :
      can(regex("^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\\.[a-zA-Z]{2,}$", email))
    ])
    error_message = "All notification emails must be valid email addresses."
  }

  validation {
    condition     = length(var.notification_emails) <= 10
    error_message = "At most 10 notification email addresses may be specified."
  }
}

# ตรวจสอบ list ของ IAM roles
variable "iam_roles" {
  type        = list(string)
  description = "รายการ IAM Role ARNs"

  validation {
    condition = alltrue([
      for arn in var.iam_roles :
      can(regex("^arn:aws:iam::[0-9]{12}:role/[a-zA-Z0-9+=,.@_/-]+$", arn))
    ])
    error_message = "All IAM role ARNs must be valid (format: arn:aws:iam::123456789012:role/RoleName)."
  }
}
```

---

## Step 609: Map Key Validation & Combined Conditions

### การตรวจสอบ Map keys และ Combined Conditions

```hcl
# ==========================================
# MAP KEY VALIDATION
# ==========================================

# ตรวจสอบว่า required keys มีอยู่ใน map
variable "environment_config" {
  type        = map(string)
  description = "Configuration สำหรับแต่ละ environment"

  validation {
    condition = alltrue([
      for env in ["dev", "staging", "prod"] :
      contains(keys(var.environment_config), env)
    ])
    error_message = "Environment config must contain keys for: dev, staging, prod."
  }
}

# ตรวจสอบ map keys ว่า match pattern
variable "service_endpoints" {
  type        = map(string)
  description = "Endpoints สำหรับแต่ละ service"

  validation {
    condition = alltrue([
      for service_name, endpoint in var.service_endpoints :
      can(regex("^[a-z0-9-]+$", service_name))
    ])
    error_message = "Service names (map keys) must contain only lowercase letters, numbers, and hyphens."
  }

  validation {
    condition = alltrue([
      for service_name, endpoint in var.service_endpoints :
      can(regex("^https?://", endpoint))
    ])
    error_message = "All service endpoints must start with http:// or https://."
  }
}

# ==========================================
# COMBINED CONDITIONS - การรวมหลาย conditions ด้วย &&
# ==========================================

# ตัวอย่าง - ตรวจสอบ EC2 instance configuration
variable "ec2_config" {
  type = object({
    instance_type = string
    min_count     = number
    max_count     = number
    desired_count = number
  })
  description = "EC2 Auto Scaling configuration"

  # ตรวจสอบว่า min <= desired <= max
  validation {
    condition = (
      var.ec2_config.min_count <= var.ec2_config.desired_count &&
      var.ec2_config.desired_count <= var.ec2_config.max_count
    )
    error_message = "Instance counts must satisfy: min_count <= desired_count <= max_count."
  }

  # ตรวจสอบว่า counts เป็นค่าบวก
  validation {
    condition = (
      var.ec2_config.min_count >= 0 &&
      var.ec2_config.max_count >= 1 &&
      var.ec2_config.desired_count >= 0
    )
    error_message = "Instance counts must be non-negative, and max_count must be at least 1."
  }
}

# ตัวอย่าง - ตรวจสอบ RDS configuration
variable "rds_config" {
  type = object({
    engine         = string
    engine_version = string
    instance_class = string
    storage_gb     = number
    multi_az       = bool
  })
  description = "RDS database configuration"

  validation {
    condition = contains(
      ["mysql", "postgres", "mariadb"],
      var.rds_config.engine
    )
    error_message = "RDS engine must be one of: mysql, postgres, mariadb."
  }

  validation {
    condition = (
      var.rds_config.storage_gb >= 20 &&
      var.rds_config.storage_gb <= 65536
    )
    error_message = "RDS storage must be between 20 GB and 65536 GB."
  }

  validation {
    condition     = can(regex("^db\\.[a-z0-9]+\\.[a-z0-9]+$", var.rds_config.instance_class))
    error_message = "RDS instance class must be in format 'db.family.size' (e.g., db.t3.micro)."
  }
}

# ==========================================
# COMPLEX COMBINED VALIDATION
# ==========================================

# ตัวอย่าง - ตรวจสอบ network configuration ที่ซับซ้อน
variable "network_config" {
  type = object({
    vpc_cidr           = string
    public_subnets     = list(string)
    private_subnets    = list(string)
    enable_nat_gateway = bool
    single_nat_gateway = bool
  })
  description = "Network configuration"

  validation {
    condition     = can(cidrhost(var.network_config.vpc_cidr, 0))
    error_message = "VPC CIDR must be a valid IPv4 CIDR."
  }

  validation {
    condition     = length(var.network_config.public_subnets) >= 2
    error_message = "At least 2 public subnets required for high availability."
  }

  validation {
    condition     = length(var.network_config.public_subnets) == length(var.network_config.private_subnets)
    error_message = "Number of public and private subnets must be equal."
  }

  # ถ้า single_nat_gateway = true แล้ว enable_nat_gateway ต้องเป็น true ด้วย
  validation {
    condition = (
      !var.network_config.single_nat_gateway ||
      var.network_config.enable_nat_gateway
    )
    error_message = "single_nat_gateway can only be true when enable_nat_gateway is also true."
  }
}
```

---

## Step 610: Sensitive, Nullable, Variable Files & Real-World Examples

### Sensitive Variables

```hcl
# ==========================================
# SENSITIVE VARIABLES
# ==========================================

# sensitive = true - ซ่อนค่าจาก output และ logs
variable "database_password" {
  type        = string
  description = "Database master password"
  sensitive   = true

  validation {
    condition     = length(var.database_password) >= 16
    error_message = "Database password must be at least 16 characters."
  }
}

variable "api_key" {
  type        = string
  description = "External API key"
  sensitive   = true
  # ไม่มี default - ต้องระบุเสมอ
}

variable "tls_private_key" {
  type        = string
  description = "TLS private key content"
  sensitive   = true

  validation {
    condition     = startswith(var.tls_private_key, "-----BEGIN")
    error_message = "TLS private key must be in PEM format."
  }
}
```

### Nullable Variables

```hcl
# ==========================================
# NULLABLE VARIABLES
# ==========================================

# nullable = false - ไม่อนุญาตให้เป็น null
variable "project_owner" {
  type        = string
  description = "เจ้าของโปรเจค - ต้องระบุเสมอ"
  nullable    = false  # ไม่สามารถ override ด้วย null ได้

  validation {
    condition     = length(var.project_owner) > 0
    error_message = "Project owner must not be empty."
  }
}

# nullable = true (default) - สามารถเป็น null ได้
variable "optional_kms_key_id" {
  type        = string
  description = "KMS Key ID (optional)"
  default     = null  # null = ไม่ใช้ KMS encryption
  nullable    = true
}
```

### Variable Files Organization

```hcl
# ==========================================
# VARIABLE FILES ORGANIZATION
# ==========================================
# โครงสร้างไฟล์ที่แนะนำ:
#
# project/
# ├── variables.tf          # ประกาศ variables ทั้งหมด
# ├── terraform.tfvars      # ค่า default (ควร commit)
# ├── dev.tfvars            # ค่าสำหรับ dev
# ├── staging.tfvars        # ค่าสำหรับ staging
# └── prod.tfvars           # ค่าสำหรับ prod
```

```hcl
# variables.tf
variable "environment" {
  type    = string
  default = "dev"
}

variable "instance_type" {
  type    = string
  default = "t3.micro"
}

variable "replica_count" {
  type    = number
  default = 1
}
```

```hcl
# terraform.tfvars (default values)
environment   = "dev"
instance_type = "t3.micro"
replica_count = 1
```

```hcl
# prod.tfvars
environment   = "prod"
instance_type = "c5.xlarge"
replica_count = 5
```

```bash
# การใช้งาน
terraform plan -var-file="prod.tfvars"
terraform apply -var-file="prod.tfvars"
```

### Optional Object Attributes with Defaults

```hcl
# ==========================================
# OPTIONAL OBJECT ATTRIBUTES WITH DEFAULTS
# ==========================================
# Terraform 1.3+ รองรับ optional() ใน object types

variable "server_config" {
  type = object({
    instance_type = string
    # optional attributes พร้อม default values
    disk_size_gb    = optional(number, 20)
    enable_backup   = optional(bool, true)
    backup_schedule = optional(string, "0 2 * * *")
    tags            = optional(map(string), {})
    extra_volumes   = optional(list(object({
      size_gb   = number
      type      = optional(string, "gp3")
      encrypted = optional(bool, true)
    })), [])
  })
  description = "Server configuration with optional attributes"

  validation {
    condition     = var.server_config.disk_size_gb >= 8 && var.server_config.disk_size_gb <= 16384
    error_message = "Disk size must be between 8 and 16384 GB."
  }
}

# ตัวอย่างการใช้งาน - ระบุแค่ required attributes
# server_config = {
#   instance_type = "t3.medium"
#   # disk_size_gb จะเป็น 20 (default)
#   # enable_backup จะเป็น true (default)
# }
```

### 10+ Real-World Validation Examples

```hcl
# ==========================================
# REAL-WORLD VALIDATION EXAMPLES
# ==========================================

# 1. EKS Cluster Configuration
variable "eks_config" {
  type = object({
    cluster_name    = string
    kubernetes_version = string
    node_count      = number
    node_type       = string
  })

  validation {
    condition     = can(regex("^[a-zA-Z][a-zA-Z0-9-]*$", var.eks_config.cluster_name))
    error_message = "EKS cluster name must start with a letter and contain only alphanumeric characters and hyphens."
  }

  validation {
    condition     = can(regex("^1\\.(2[4-9]|[3-9][0-9])$", var.eks_config.kubernetes_version))
    error_message = "Kubernetes version must be 1.24 or later (e.g., 1.28)."
  }

  validation {
    condition     = var.eks_config.node_count >= 2 && var.eks_config.node_count <= 100
    error_message = "EKS node count must be between 2 and 100."
  }
}

# 2. ALB Configuration
variable "alb_config" {
  type = object({
    name           = string
    internal       = bool
    ssl_policy     = string
    idle_timeout   = number
  })

  validation {
    condition     = can(regex("^[a-zA-Z0-9-]{1,32}$", var.alb_config.name))
    error_message = "ALB name must be 1-32 alphanumeric characters or hyphens."
  }

  validation {
    condition = contains([
      "ELBSecurityPolicy-TLS13-1-2-2021-06",
      "ELBSecurityPolicy-TLS13-1-2-Res-2021-06",
      "ELBSecurityPolicy-2016-08"
    ], var.alb_config.ssl_policy)
    error_message = "ALB SSL policy must be a valid AWS security policy."
  }

  validation {
    condition     = var.alb_config.idle_timeout >= 1 && var.alb_config.idle_timeout <= 4000
    error_message = "ALB idle timeout must be between 1 and 4000 seconds."
  }
}

# 3. CloudFront Distribution
variable "cloudfront_config" {
  type = object({
    price_class    = string
    min_ttl        = number
    default_ttl    = number
    max_ttl        = number
  })

  validation {
    condition = contains([
      "PriceClass_All",
      "PriceClass_200",
      "PriceClass_100"
    ], var.cloudfront_config.price_class)
    error_message = "CloudFront price class must be PriceClass_All, PriceClass_200, or PriceClass_100."
  }

  validation {
    condition = (
      var.cloudfront_config.min_ttl >= 0 &&
      var.cloudfront_config.default_ttl >= var.cloudfront_config.min_ttl &&
      var.cloudfront_config.max_ttl >= var.cloudfront_config.default_ttl
    )
    error_message = "CloudFront TTL values must satisfy: min_ttl <= default_ttl <= max_ttl."
  }
}

# 4. Route53 Record
variable "dns_record" {
  type = object({
    name    = string
    type    = string
    ttl     = number
    records = list(string)
  })

  validation {
    condition = contains([
      "A", "AAAA", "CNAME", "MX", "NS", "PTR", "SOA", "SPF", "SRV", "TXT"
    ], var.dns_record.type)
    error_message = "DNS record type must be a valid Route53 record type."
  }

  validation {
    condition     = var.dns_record.ttl >= 60 && var.dns_record.ttl <= 172800
    error_message = "DNS TTL must be between 60 seconds and 172800 seconds (48 hours)."
  }
}

# 5. SNS Topic Configuration
variable "sns_config" {
  type = object({
    name         = string
    is_fifo      = bool
    kms_key_id   = optional(string)
  })

  validation {
    condition = (
      !var.sns_config.is_fifo ||
      endswith(var.sns_config.name, ".fifo")
    )
    error_message = "FIFO SNS topics must have names ending with '.fifo'."
  }
}

# 6. Lambda Function Configuration
variable "lambda_config" {
  type = object({
    function_name = string
    runtime       = string
    memory_mb     = number
    timeout_sec   = number
    handler       = string
  })

  validation {
    condition = contains([
      "python3.9", "python3.10", "python3.11",
      "nodejs18.x", "nodejs20.x",
      "java11", "java17",
      "go1.x"
    ], var.lambda_config.runtime)
    error_message = "Lambda runtime must be a supported AWS Lambda runtime."
  }

  validation {
    condition = contains(
      [128, 256, 512, 1024, 2048, 3008, 4096, 5120, 6144, 7168, 8192, 9216, 10240],
      var.lambda_config.memory_mb
    )
    error_message = "Lambda memory must be one of the allowed values (128 to 10240 MB)."
  }

  validation {
    condition     = var.lambda_config.timeout_sec >= 1 && var.lambda_config.timeout_sec <= 900
    error_message = "Lambda timeout must be between 1 and 900 seconds."
  }
}

# 7. SQS Queue Configuration
variable "sqs_config" {
  type = object({
    queue_name                 = string
    is_fifo                    = bool
    visibility_timeout_seconds = number
    message_retention_seconds  = number
    max_message_size           = number
  })

  validation {
    condition     = var.sqs_config.visibility_timeout_seconds >= 0 && var.sqs_config.visibility_timeout_seconds <= 43200
    error_message = "SQS visibility timeout must be between 0 and 43200 seconds."
  }

  validation {
    condition     = var.sqs_config.message_retention_seconds >= 60 && var.sqs_config.message_retention_seconds <= 1209600
    error_message = "SQS message retention must be between 60 seconds (1 minute) and 1209600 seconds (14 days)."
  }

  validation {
    condition     = var.sqs_config.max_message_size >= 1024 && var.sqs_config.max_message_size <= 262144
    error_message = "SQS max message size must be between 1024 bytes (1 KB) and 262144 bytes (256 KB)."
  }
}

# 8. Backup Retention Policy
variable "backup_retention_days" {
  type        = number
  description = "จำนวนวันที่เก็บ backup"

  validation {
    condition     = var.backup_retention_days >= 1 && var.backup_retention_days <= 35
    error_message = "Backup retention must be between 1 and 35 days."
  }
}

# 9. IP Allowlist (with CIDR and plain IP support)
variable "ip_allowlist" {
  type        = list(string)
  description = "รายการ IP addresses หรือ CIDR blocks ที่อนุญาต"

  validation {
    condition = alltrue([
      for ip in var.ip_allowlist :
      can(cidrhost(ip, 0)) || can(regex("^(25[0-5]|2[0-4][0-9]|[01]?[0-9][0-9]?)\\.(25[0-5]|2[0-4][0-9]|[01]?[0-9][0-9]?)\\.(25[0-5]|2[0-4][0-9]|[01]?[0-9][0-9]?)\\.(25[0-5]|2[0-4][0-9]|[01]?[0-9][0-9]?)$", ip))
    ])
    error_message = "All entries in ip_allowlist must be valid IPv4 addresses or CIDR notation."
  }
}

# 10. Cost Center Tag (business requirement)
variable "cost_center" {
  type        = string
  description = "Cost center code สำหรับ billing"

  validation {
    condition     = can(regex("^CC-[0-9]{4}-[A-Z]{2,4}$", var.cost_center))
    error_message = "Cost center must be in format 'CC-XXXX-YY' where X is 4 digits and Y is 2-4 uppercase letters (e.g., CC-1234-ENG)."
  }
}
```

---

## สรุป (Summary)

### Best Practices สำหรับ Variable Validation:

1. **Fail Fast** - ตรวจสอบ input ให้เร็วที่สุด ก่อน apply
2. **Clear Error Messages** - error message ต้องบอกว่าต้องแก้ยังไง ไม่ใช่แค่บอกว่าผิด
3. **Multiple Validations** - แยก validation ออกเป็น concerns ย่อยๆ
4. **Use `can()` for risky expressions** - ใช้ `can()` เพื่อป้องกัน error จาก invalid values
5. **Test your validations** - ทดสอบทั้ง valid และ invalid inputs

### Validation Functions ที่ใช้บ่อย:

| Function | การใช้งาน |
|----------|-----------|
| `can(expr)` | คืน true ถ้า expression ไม่ error |
| `contains(list, val)` | ตรวจสอบว่า val อยู่ใน list |
| `length(val)` | ความยาวของ string/list/map |
| `regex(pattern, val)` | ตรวจสอบ regex pattern |
| `cidrhost(cidr, 0)` | ตรวจสอบ CIDR format |
| `alltrue(list)` | true ถ้าทุก element เป็น true |
| `startswith(str, prefix)` | ตรวจสอบ prefix |
| `endswith(str, suffix)` | ตรวจสอบ suffix |

---

*จบ Part 061 - Variables: Deep Dive & Validation*
