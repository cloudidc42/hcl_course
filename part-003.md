# Part 003: HCL Data Types: Primitives
## ประเภทข้อมูลพื้นฐานใน HCL (Steps 21-30)

---

## Step 21: String Type - ทุกอย่างเกี่ยวกับ String

### String Basics

```hcl
# String declarations
variable "project_name" {
  type    = string
  default = "my-project"
}

variable "environment" {
  type    = string
  default = "production"
}

# String ใน locals
locals {
  simple_string    = "Hello, World!"
  empty_string     = ""
  string_with_num  = "server-01"
  string_with_dash = "my-company-app"
  thai_string      = "สวัสดีชาวโลก"
  unicode_string   = "Hello 🌍 World"
}
```

### String Operations

```hcl
locals {
  name = "terraform"
  
  # Length
  name_length = length(local.name)  # 9
  
  # Uppercase/Lowercase
  upper_name = upper(local.name)     # "TERRAFORM"
  lower_name = lower(local.name)     # "terraform"
  title_name = title(local.name)     # "Terraform"
  
  # Trim
  spaced = "  hello world  "
  trimmed       = trimspace(local.spaced)           # "hello world"
  trim_left     = trimleft(local.spaced, " ")       # "hello world  "
  trim_right    = trimright(local.spaced, " ")      # "  hello world"
  trim_prefix   = trimprefix("myapp-prod", "myapp-") # "prod"
  trim_suffix   = trimsuffix("myapp-prod", "-prod")  # "myapp"
  
  # Replace
  replaced  = replace("hello world", "world", "terraform")  # "hello terraform"
  
  # Substring
  first_five = substr("hello world", 0, 5)  # "hello"
  from_six   = substr("hello world", 6, 5)  # "world"
  
  # Split and Join
  csv_string = "a,b,c,d,e"
  parts      = split(",", local.csv_string)  # ["a", "b", "c", "d", "e"]
  joined     = join("-", local.parts)         # "a-b-c-d-e"
  
  # Contains/Starts/Ends
  has_hello   = strcontains("hello world", "hello")  # true
  starts_with = startswith("hello world", "hello")   # true
  ends_with   = endswith("hello world", "world")     # true
}
```

### String Interpolation แบบต่างๆ

```hcl
variable "env" {
  default = "production"
}

variable "region" {
  default = "ap-southeast-1"
}

locals {
  project = "myapp"
  
  # Basic interpolation
  bucket_name = "${local.project}-${var.env}-data"
  
  # Interpolation กับ function
  upper_env = "${upper(var.env)}-bucket"
  
  # Interpolation กับ conditional
  env_tag = "${var.env == "production" ? "PROD" : "NON-PROD"}"
  
  # Multi-level interpolation
  full_arn = "arn:aws:s3:::${local.project}-${var.env}-${var.region}"
  
  # String formatting ด้วย format()
  formatted = format("%.2f", 3.14159)  # "3.14"
  padded    = format("%05d", 42)        # "00042"
  
  # formatlist() สำหรับ lists
  server_names = formatlist("server-%02d", range(1, 6))
  # ["server-01", "server-02", "server-03", "server-04", "server-05"]
}
```

### Regex ใน HCL

```hcl
locals {
  email = "user@example.com"
  
  # regex() - return first match or error
  domain = regex("[^@]+$", local.email)  # "example.com"
  
  # regexall() - return list of all matches
  text   = "IP: 192.168.1.1 and 10.0.0.1"
  ips    = regexall("\\d+\\.\\d+\\.\\d+\\.\\d+", local.text)
  # ["192.168.1.1", "10.0.0.1"]
  
  # ตรวจสอบ format ด้วย can()
  is_valid_email = can(regex("^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\\.[a-zA-Z]{2,}$", local.email))
  # true
}
```

---

## Step 22: String Concatenation

### วิธีต่าง ๆ ในการต่อ String

```hcl
locals {
  first_name = "John"
  last_name  = "Doe"
  domain     = "example.com"
  
  # Method 1: String interpolation (แนะนำ)
  full_name_1 = "${local.first_name} ${local.last_name}"
  
  # Method 2: join() function
  full_name_2 = join(" ", [local.first_name, local.last_name])
  
  # Method 3: format() function
  email = format("%s.%s@%s", lower(local.first_name), lower(local.last_name), local.domain)
  
  # Method 4: concat() สำหรับ lists (ไม่ใช่ strings)
  # ต้องใช้ join() เพื่อแปลงกลับ
  parts     = concat(["Hello"], [", "], ["World"])
  sentence  = join("", local.parts)  # "Hello, World"
}
```

### String Concatenation ใน Resource Arguments

```hcl
variable "project" {
  default = "myapp"
}

variable "env" {
  default = "production"
}

locals {
  prefix = "${var.project}-${var.env}"
}

resource "aws_s3_bucket" "data" {
  bucket = "${local.prefix}-data"    # "myapp-production-data"
}

resource "aws_s3_bucket" "logs" {
  bucket = "${local.prefix}-logs"    # "myapp-production-logs"
}

resource "aws_iam_role" "app" {
  name = "${local.prefix}-role"      # "myapp-production-role"
}

resource "aws_iam_policy" "app" {
  name = "${local.prefix}-policy"    # "myapp-production-policy"
}
```

---

## Step 23: Multi-line Strings

### Heredoc Syntax

```hcl
# Basic heredoc (<<EOF)
resource "aws_iam_policy" "s3_access" {
  name = "s3-access-policy"
  
  policy = <<EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:PutObject",
        "s3:DeleteObject"
      ],
      "Resource": "arn:aws:s3:::my-bucket/*"
    }
  ]
}
EOF
}

# Indented heredoc (<<-EOF) - ลบ leading whitespace
resource "aws_instance" "web" {
  ami           = "ami-12345"
  instance_type = "t3.micro"
  
  user_data = <<-BASH
    #!/bin/bash
    
    # Update system
    sudo apt-get update -y
    sudo apt-get upgrade -y
    
    # Install nginx
    sudo apt-get install -y nginx
    
    # Start nginx
    sudo systemctl enable nginx
    sudo systemctl start nginx
    
    # Create homepage
    cat > /var/www/html/index.html << 'HTML'
    <!DOCTYPE html>
    <html>
    <body>
      <h1>Deployed by Terraform!</h1>
    </body>
    </html>
    HTML
  BASH
}
```

### Heredoc กับ Interpolation

```hcl
variable "db_host" {
  default = "db.example.com"
}

variable "db_port" {
  default = 5432
}

resource "local_file" "config" {
  filename = "/tmp/app.conf"
  
  # Heredoc ที่ใช้ interpolation
  content = <<-EOT
    # Application Configuration
    # Generated by Terraform on ${timestamp()}
    
    [database]
    host = ${var.db_host}
    port = ${var.db_port}
    name = myapp_db
    
    [server]
    host = 0.0.0.0
    port = 8080
    
    [logging]
    level = INFO
    file  = /var/log/myapp/app.log
  EOT
}
```

---

## Step 24: Raw Strings

### Escaping ใน Strings

```hcl
locals {
  # ปกติ $ เริ่ม interpolation
  # ใช้ $$ เพื่อ escape เป็น literal $
  shell_script = "echo $${HOME}"    # จะ output: echo ${HOME}
  
  # ปกติ % เริ่ม template directive
  # ใช้ %% เพื่อ escape เป็น literal %
  percentage   = "Usage: 80%%"      # จะ output: Usage: 80%
  
  # Escape sequences ปกติ
  tab_char    = "col1\tcol2"        # Tab
  newline     = "line1\nline2"      # Newline
  backslash   = "path\\to\\file"    # Backslash
  quote       = "say \"hello\""     # Double quote
  
  # Unicode escape
  heart       = "\u2764"            # ❤
  check_mark  = "\u2705"            # ✅
}
```

### ตัวอย่างการใช้งานจริง

```hcl
# Shell script ที่มี $ variables
resource "aws_instance" "web" {
  ami           = "ami-12345"
  instance_type = "t3.micro"
  
  # user_data ที่มี bash variables
  user_data = <<-BASH
    #!/bin/bash
    
    # Bash variable (ต้องใช้ $$ เพื่อ escape)
    INSTANCE_ID=$$(curl -s http://169.254.169.254/latest/meta-data/instance-id)
    REGION=$$(curl -s http://169.254.169.254/latest/meta-data/placement/region)
    
    echo "Instance: $${INSTANCE_ID}"
    echo "Region: $${REGION}"
    
    # Terraform variable (ไม่ต้อง escape)
    echo "Project: ${var.project}"
    echo "Environment: ${var.environment}"
    
    # Write config
    cat > /etc/myapp.conf << EOF
    PROJECT=${var.project}
    ENVIRONMENT=${var.environment}
    INSTANCE_ID=$${INSTANCE_ID}
    REGION=$${REGION}
    EOF
  BASH
}
```

---

## Step 25: Number Type - Integer และ Float

### Number Operations

```hcl
locals {
  # Integer operations
  a = 10
  b = 3
  
  add      = local.a + local.b   # 13
  subtract = local.a - local.b   # 7
  multiply = local.a * local.b   # 30
  divide   = local.a / local.b   # 3.3333...
  modulo   = local.a % local.b   # 1
  
  # Float operations
  x = 3.14
  y = 2.0
  
  float_add  = local.x + local.y  # 5.14
  float_mult = local.x * local.y  # 6.28
  
  # Math functions
  abs_val    = abs(-42)            # 42
  ceil_val   = ceil(4.1)           # 5
  floor_val  = floor(4.9)          # 4
  max_val    = max(1, 5, 3, 2, 4) # 5
  min_val    = min(1, 5, 3, 2, 4) # 1
  pow_val    = pow(2, 10)          # 1024
  sqrt_val   = sqrt(144)           # 12
  log_val    = log(100, 10)        # 2
}
```

### ตัวอย่างการใช้งานจริง

```hcl
variable "instance_count" {
  type    = number
  default = 3
}

variable "base_port" {
  type    = number
  default = 8000
}

# คำนวณ CIDR blocks
locals {
  vpc_cidr    = "10.0.0.0/16"
  subnet_bits = 8  # /24 subnets
  
  # สร้าง subnet CIDRs ด้วย cidrsubnet
  public_subnets = [
    cidrsubnet(local.vpc_cidr, local.subnet_bits, 0),  # 10.0.0.0/24
    cidrsubnet(local.vpc_cidr, local.subnet_bits, 1),  # 10.0.1.0/24
    cidrsubnet(local.vpc_cidr, local.subnet_bits, 2),  # 10.0.2.0/24
  ]
  
  private_subnets = [
    cidrsubnet(local.vpc_cidr, local.subnet_bits, 10), # 10.0.10.0/24
    cidrsubnet(local.vpc_cidr, local.subnet_bits, 11), # 10.0.11.0/24
    cidrsubnet(local.vpc_cidr, local.subnet_bits, 12), # 10.0.12.0/24
  ]
}

# คำนวณ port numbers
resource "aws_lb_target_group" "app" {
  count = var.instance_count
  
  name     = "tg-${count.index}"
  port     = var.base_port + count.index  # 8000, 8001, 8002
  protocol = "HTTP"
  vpc_id   = aws_vpc.main.id
}
```

---

## Step 26: Bool Type

### Boolean Operations

```hcl
locals {
  # Boolean values
  is_production = true
  is_staging    = false
  debug_mode    = false
  
  # Logical operators
  and_result  = local.is_production && !local.debug_mode  # true
  or_result   = local.is_staging || local.is_production    # true
  not_result  = !local.is_production                       # false
  
  # Boolean ใน conditions
  enable_monitoring   = local.is_production || var.force_monitoring
  skip_final_snapshot = !local.is_production
  
  # Comparison operators return boolean
  is_large    = var.instance_count > 5   # true if count > 5
  is_small    = var.instance_count <= 2  # true if count <= 2
  is_equal    = var.environment == "production"
  is_not_equal = var.environment != "development"
}
```

### Boolean Variable Patterns

```hcl
# Pattern 1: Feature flags
variable "enable_nat_gateway" {
  description = "Create NAT Gateway for private subnets"
  type        = bool
  default     = false
}

variable "enable_vpn_gateway" {
  description = "Create VPN Gateway"
  type        = bool
  default     = false
}

variable "enable_flow_logs" {
  description = "Enable VPC flow logs"
  type        = bool
  default     = true
}

# Pattern 2: Boolean ใน count
resource "aws_nat_gateway" "main" {
  count = var.enable_nat_gateway ? 1 : 0
  
  allocation_id = aws_eip.nat[0].id
  subnet_id     = aws_subnet.public[0].id
}

resource "aws_vpn_gateway" "main" {
  count  = var.enable_vpn_gateway ? 1 : 0
  vpc_id = aws_vpc.main.id
}

resource "aws_flow_log" "vpc" {
  count = var.enable_flow_logs ? 1 : 0
  
  iam_role_arn    = aws_iam_role.flow_log.arn
  log_destination = aws_cloudwatch_log_group.flow_log.arn
  traffic_type    = "ALL"
  vpc_id          = aws_vpc.main.id
}
```

---

## Step 27: Type Conversions

### tostring(), tonumber(), tobool()

```hcl
locals {
  # tostring() - แปลงเป็น string
  num_to_str    = tostring(42)         # "42"
  bool_to_str   = tostring(true)       # "true"
  float_to_str  = tostring(3.14)       # "3.14"
  
  # tonumber() - แปลงเป็น number
  str_to_num    = tonumber("42")       # 42
  str_to_float  = tonumber("3.14")     # 3.14
  
  # tobool() - แปลงเป็น boolean
  str_true      = tobool("true")       # true
  str_false     = tobool("false")      # false
  # tobool("yes")  จะ error! ต้องเป็น "true" หรือ "false" เท่านั้น
}
```

### Type Conversion ใน Variables

```hcl
# บางครั้งต้องแปลง type
variable "port_number" {
  type    = string
  default = "8080"
}

resource "aws_lb_target_group" "app" {
  name     = "app-tg"
  port     = tonumber(var.port_number)  # แปลง string เป็น number
  protocol = "HTTP"
  vpc_id   = aws_vpc.main.id
}

# ตัวอย่าง: environment variable ที่เป็น string แต่ต้องการ bool
variable "create_resources" {
  type    = string
  default = "true"
}

resource "aws_s3_bucket" "optional" {
  count  = tobool(var.create_resources) ? 1 : 0
  bucket = "optional-bucket"
}
```

### Implicit Type Conversion

```hcl
locals {
  # HCL ทำ implicit conversion บางอย่างให้อัตโนมัติ
  
  # number → string (automatic ใน interpolation)
  msg = "Port is: ${42}"  # "Port is: 42"
  
  # bool → string (automatic ใน interpolation)
  flag = "Enabled: ${true}"  # "Enabled: true"
}
```

---

## Step 28: Type Checking ด้วย can() และ try()

### can() Function

```hcl
# can() ทดสอบว่า expression จะ error หรือไม่
# Return true ถ้า expression สำเร็จ, false ถ้า error

locals {
  # ตรวจสอบว่า string เป็น valid number หรือไม่
  valid_num   = can(tonumber("42"))     # true
  invalid_num = can(tonumber("abc"))    # false
  
  # ตรวจสอบ regex match
  email       = "user@example.com"
  is_valid_email = can(regex("^[^@]+@[^@]+\\.[^@]+$", local.email))  # true
  
  # ตรวจสอบ attribute access
  # จะ return false ถ้า object ไม่มี attribute นั้น
  obj = { name = "test" }
  has_name  = can(local.obj.name)    # true
  has_other = can(local.obj.other)   # false
}
```

### try() Function

```hcl
# try() ลองทำ expression แต่ละอัน จนกว่าจะสำเร็จ
# คล้าย try-catch ใน programming languages

locals {
  # try() กับ default value
  port_from_config = try(
    tonumber(var.config_port),  # ลองแปลงเป็น number
    8080                         # ถ้า error ใช้ default
  )
  
  # try() กับ nested attribute access
  config_value = try(
    var.complex_config.database.host,  # ลองอ่าน nested attribute
    "localhost"                          # default ถ้าไม่มี
  )
  
  # try() สำหรับ optional list element
  first_az = try(
    var.availability_zones[0],  # element แรก
    "ap-southeast-1a"            # default ถ้า list ว่าง
  )
  
  # try() กับ regex
  extracted_region = try(
    regex("[a-z]+-[a-z]+-[0-9]+", var.arn),
    "unknown-region"
  )
}
```

### ตัวอย่างการใช้งานจริง

```hcl
# ตัวอย่าง: Handle optional configuration
variable "database_config" {
  description = "Database configuration (optional)"
  type = object({
    host     = string
    port     = optional(number, 5432)
    name     = string
    ssl_mode = optional(string, "require")
  })
  default = null
}

locals {
  # ตรวจสอบว่ามี database config หรือไม่
  has_database = var.database_config != null
  
  # ดึงค่าอย่างปลอดภัย
  db_host = try(var.database_config.host, "localhost")
  db_port = try(var.database_config.port, 5432)
  db_name = try(var.database_config.name, "app")
  
  # สร้าง connection string ถ้ามี config
  connection_string = local.has_database ? format(
    "postgresql://%s:%d/%s?sslmode=%s",
    local.db_host,
    local.db_port,
    local.db_name,
    try(var.database_config.ssl_mode, "require")
  ) : null
}
```

---

## Step 29: String Escape Sequences สมบูรณ์

### ตารางสรุป Escape Sequences

| Sequence | Description | ตัวอย่าง |
|----------|-------------|----------|
| `\n` | Newline | `"line1\nline2"` |
| `\r` | Carriage return | `"line1\r\nline2"` |
| `\t` | Tab | `"col1\tcol2"` |
| `\"` | Double quote | `"say \"hello\""` |
| `\\` | Backslash | `"C:\\Users"` |
| `\uNNNN` | Unicode (4 hex) | `"\u2764"` (❤) |
| `\UNNNNNNNN` | Unicode (8 hex) | `"\U0001F600"` (😀) |
| `$$` | Literal $ in template | `"$${HOME}"` |
| `%%` | Literal % in template | `"100%%"` |

### ตัวอย่างการใช้งาน

```hcl
locals {
  # Path separators
  windows_path = "C:\\Program Files\\Terraform"
  unix_path    = "/usr/local/bin/terraform"
  
  # JSON ใน string (ต้อง escape quotes)
  inline_json  = "{\"key\": \"value\", \"number\": 42}"
  
  # Newlines ใน string
  multiline = "First Line\nSecond Line\nThird Line"
  
  # Tab-separated data
  tsv_header = "Name\tAge\tCity\tCountry"
  tsv_row    = "Alice\t30\tBangkok\tThailand"
  
  # Unicode characters
  checkmark   = "\u2713"   # ✓
  crossmark   = "\u2717"   # ✗
  star        = "\u2605"   # ★
  
  # Bash script ที่ต้องการ literal $
  user_data = <<-BASH
    #!/bin/bash
    # Get metadata (ใช้ $$ เพื่อ escape)
    INSTANCE_ID=$$(curl -s http://169.254.169.254/latest/meta-data/instance-id)
    
    # Use Terraform variable (ไม่ต้อง escape)
    echo "App: ${var.app_name}"
    echo "Instance: $${INSTANCE_ID}"
  BASH
}
```

---

## Step 30: ตัวอย่างสมบูรณ์ - Primitive Types ใน Project จริง

### Complete Example: Application Configuration

```hcl
# variables.tf
variable "app_name" {
  description = "ชื่อ application"
  type        = string
  
  validation {
    condition     = can(regex("^[a-z][a-z0-9-]{1,28}[a-z0-9]$", var.app_name))
    error_message = "App name must be 3-30 chars, lowercase letters, numbers, hyphens."
  }
}

variable "environment" {
  description = "สภาพแวดล้อม"
  type        = string
  
  validation {
    condition     = contains(["dev", "staging", "prod"], var.environment)
    error_message = "Must be dev, staging, or prod."
  }
}

variable "instance_count" {
  description = "จำนวน instances"
  type        = number
  default     = 2
  
  validation {
    condition     = var.instance_count >= 1 && var.instance_count <= 20
    error_message = "Instance count must be between 1 and 20."
  }
}

variable "enable_https" {
  description = "เปิดใช้ HTTPS"
  type        = bool
  default     = true
}

variable "app_port" {
  description = "Application port"
  type        = number
  default     = 8080
  
  validation {
    condition     = var.app_port >= 1024 && var.app_port <= 65535
    error_message = "Port must be between 1024 and 65535."
  }
}

variable "ssl_certificate_arn" {
  description = "ARN ของ SSL certificate (optional)"
  type        = string
  default     = null
}

# locals.tf
locals {
  # String operations
  app_prefix    = "${var.app_name}-${var.environment}"
  app_fqdn      = "${local.app_prefix}.example.com"
  
  # Boolean logic
  is_production = var.environment == "prod"
  needs_https   = var.enable_https && var.ssl_certificate_arn != null
  
  # Number calculations
  desired_count = local.is_production ? var.instance_count * 2 : var.instance_count
  max_count     = ceil(local.desired_count * 1.5)  # 150% สำหรับ scaling
  min_count     = max(1, floor(local.desired_count * 0.5))  # 50% minimum
  
  # Type conversions
  port_string   = tostring(var.app_port)
  
  # String formatting
  health_check_url = format("http://localhost:%d/health", var.app_port)
  log_prefix       = format("[%s][%s]", upper(var.environment), var.app_name)
  
  # Conditional strings
  protocol      = local.needs_https ? "https" : "http"
  endpoint_url  = "${local.protocol}://${local.app_fqdn}:${local.port_string}"
  
  # Resource naming with type conversion
  resource_names = {
    ecs_cluster       = local.app_prefix
    ecs_service       = "${local.app_prefix}-svc"
    alb               = "${local.app_prefix}-alb"
    target_group      = "${local.app_prefix}-tg"
    security_group    = "${local.app_prefix}-sg"
    iam_role          = "${local.app_prefix}-role"
    log_group         = "/aws/ecs/${local.app_prefix}"
    ssm_param_prefix  = "/apps/${var.app_name}/${var.environment}"
  }
  
  # Tags
  common_tags = {
    Application = var.app_name
    Environment = var.environment
    ManagedBy   = "terraform"
    CostCenter  = upper(var.environment)
    Endpoint    = local.endpoint_url
  }
}

# outputs.tf
output "app_details" {
  description = "รายละเอียด application"
  value = {
    name         = var.app_name
    environment  = var.environment
    endpoint     = local.endpoint_url
    is_prod      = local.is_production
    desired_count = local.desired_count
    max_count    = local.max_count
    min_count    = local.min_count
  }
}

output "resource_names" {
  description = "ชื่อของ resources ทั้งหมด"
  value       = local.resource_names
}
```

---

## สรุป Primitive Types

### ตาราง Quick Reference

| Type | Example | Functions |
|------|---------|-----------|
| string | `"hello"` | `upper()`, `lower()`, `trim()`, `split()`, `join()`, `replace()`, `regex()` |
| number | `42`, `3.14` | `abs()`, `ceil()`, `floor()`, `max()`, `min()`, `pow()`, `sqrt()` |
| bool | `true`, `false` | ใช้ใน `&&`, `\|\|`, `!`, conditionals |

### Conversion Quick Reference

```hcl
# String → Number
tonumber("42")        # 42
tonumber("3.14")      # 3.14

# Number → String
tostring(42)          # "42"
tostring(3.14)        # "3.14"

# String → Bool
tobool("true")        # true
tobool("false")       # false

# Bool → String
tostring(true)        # "true"
tostring(false)       # "false"
```

💡 **Pro Tips:**
- ใช้ `can()` เพื่อตรวจสอบ validity ก่อน conversion
- ใช้ `try()` เพื่อ provide default value เมื่อ conversion อาจ fail
- ใช้ `format()` สำหรับ string formatting ที่ซับซ้อน
- ใช้ variable validation เพื่อ validate input ก่อนที่จะใช้

⚠️ **Common Mistakes:**
- `tobool("yes")` จะ error - ต้องใช้ `"true"` หรือ `"false"`
- `tonumber("1,000")` จะ error - ต้องลบ commas ก่อน
- ลืม escape `$` ใน bash scripts → `$${VAR}` ไม่ใช่ `${VAR}`
- ใช้ string แทน number ใน resource arguments

---

*ก่อนหน้า: [Part 002 - HCL Syntax](part-002.md)*
*ต่อไป: [Part 004 - HCL Data Types: Complex Types](part-004.md)*
