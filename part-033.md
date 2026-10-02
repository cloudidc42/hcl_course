# Part 033: Terraform Console & Expressions
# Terraform Console และการทดสอบ Expressions

## Steps 321-330: การใช้ terraform console อย่างเชี่ยวชาญ

---

## Step 321: terraform console Command

### ความหมายและการใช้งาน

`terraform console` เป็น **interactive REPL (Read-Eval-Print Loop)** สำหรับ Terraform expressions ช่วยให้เราทดสอบ expressions, functions, และค่าต่างๆ ก่อนนำไปใช้ใน configuration จริง

### วิธีเปิด Console

```bash
# เปิด interactive console
terraform console

# Console จะแสดง:
# >  (prompt รอรับ input)

# ออกจาก console
> exit
# หรือ Ctrl+D
# หรือ Ctrl+C
```

### ข้อกำหนดก่อนใช้ Console

```bash
# 1. ต้อง init ก่อน (เพื่อ download providers)
terraform init

# 2. ถ้าต้องการเข้าถึง state values ต้องมี state file
terraform init
# แล้วเปิด console

# 3. ถ้า workspace ต่างกัน ต้อง select workspace ก่อน
terraform workspace select production
terraform console
```

---

## Step 322: Interactive Expression Evaluation

### การใช้งานพื้นฐาน

```hcl
# เปิด console แล้วทดสอบ expressions ต่างๆ

# ─── Arithmetic Expressions ────────────────────────────────

> 2 + 2
4

> 10 * 5
50

> 100 / 4
25

> 7 % 3
1

> 2 ^ 8
256

# ─── String Operations ─────────────────────────────────────

> "Hello" + " " + "World"
"Hello World"

> length("Hello World")
11

> upper("hello")
"HELLO"

> lower("HELLO")
"hello"

> trimspace("  hello  ")
"hello"

> replace("hello-world", "-", "_")
"hello_world"

# ─── Boolean Operations ────────────────────────────────────

> true && false
false

> true || false
true

> !true
false

> 5 > 3
true

> "a" == "a"
true

> "a" != "b"
true
```

### String Interpolation ใน Console

```hcl
# ทดสอบ string interpolation

> "Hello ${var.name}!"
# Error: ถ้าไม่มี variable definition

# ต้องมี variables.tf ที่ define ไว้ก่อน
# และ terraform.tfvars หรือ -var flag

> "Region: ${"ap-southeast-1"}"
"Region: ap-southeast-1"

> "Count: ${5 + 3}"
"Count: 8"

> "Project: ${"myapp"}-${"production"}"
"Project: myapp-production"
```

---

## Step 323: Testing Functions ใน Console

### String Functions

```hcl
# ─── String Functions ──────────────────────────────────────

> format("Hello, %s! You are %d years old.", "Alice", 30)
"Hello, Alice! You are 30 years old."

> format("%05d", 42)
"00042"

> format("%.2f", 3.14159)
"3.14"

> formatlist("Item: %s", ["a", "b", "c"])
tolist([
  "Item: a",
  "Item: b",
  "Item: c",
])

> join(", ", ["apple", "banana", "cherry"])
"apple, banana, cherry"

> join("-", ["web", "server", "01"])
"web-server-01"

> split(",", "a,b,c,d")
tolist([
  "a",
  "b",
  "c",
  "d",
])

> split("/", "ap-southeast-1/production/web")
tolist([
  "ap-southeast-1",
  "production",
  "web",
])

> startswith("hello-world", "hello")
true

> endswith("hello-world", "world")
true

> contains(["a", "b", "c"], "b")
true

> contains(["a", "b", "c"], "z")
false

> substr("hello world", 6, 5)
"world"

> substr("hello world", 0, 5)
"hello"

> trimprefix("hello-world", "hello-")
"world"

> trimsuffix("hello-world", "-world")
"hello"

> trim("  hello  ", " ")
"hello"

> indent(4, "line1\nline2\nline3")
"    line1\n    line2\n    line3"

> chomp("hello\n")
"hello"
```

### Numeric Functions

```hcl
# ─── Numeric Functions ─────────────────────────────────────

> abs(-5)
5

> abs(5)
5

> ceil(4.1)
5

> ceil(4.9)
5

> floor(4.9)
4

> floor(4.1)
4

> max(1, 2, 3)
3

> max(100, 50, 75)
100

> min(1, 2, 3)
1

> min(100, 50, 75)
50

> pow(2, 10)
1024

> signum(-5)
-1

> signum(0)
0

> signum(5)
1

> log(100, 10)
2

> parseint("FF", 16)
255

> parseint("100", 2)
4
```

### Collection Functions

```hcl
# ─── List Functions ────────────────────────────────────────

> length(["a", "b", "c"])
3

> length({a = 1, b = 2})
2

> concat(["a", "b"], ["c", "d"])
tolist([
  "a",
  "b",
  "c",
  "d",
])

> flatten(["a", ["b", "c"], ["d", ["e"]]])
tolist([
  "a",
  "b",
  "c",
  "d",
  "e",
])

> distinct(["a", "b", "a", "c", "b"])
tolist([
  "a",
  "b",
  "c",
])

> compact(["a", "", "b", "", "c"])
tolist([
  "a",
  "b",
  "c",
])

> slice(["a", "b", "c", "d", "e"], 1, 3)
tolist([
  "b",
  "c",
])

> reverse(["a", "b", "c"])
tolist([
  "c",
  "b",
  "a",
])

> sort(["cherry", "apple", "banana"])
tolist([
  "apple",
  "banana",
  "cherry",
])

> index(["a", "b", "c"], "b")
1

> element(["a", "b", "c"], 1)
"b"

> element(["a", "b", "c"], 5)  # wraps around
"c"

> chunklist(["a", "b", "c", "d", "e"], 2)
tolist([
  tolist([
    "a",
    "b",
  ]),
  tolist([
    "c",
    "d",
  ]),
  tolist([
    "e",
  ]),
])

# ─── Map Functions ─────────────────────────────────────────

> keys({b = 2, a = 1, c = 3})
tolist([
  "a",
  "b",
  "c",
])

> values({b = 2, a = 1, c = 3})
tolist([
  1,
  2,
  3,
])

> lookup({a = "apple", b = "banana"}, "a", "default")
"apple"

> lookup({a = "apple", b = "banana"}, "z", "default")
"default"

> merge({a = 1}, {b = 2}, {c = 3})
{
  "a" = 1
  "b" = 2
  "c" = 3
}

> merge({a = 1, b = 2}, {b = 99, c = 3})  # b ถูก override
{
  "a" = 1
  "b" = 99
  "c" = 3
}

> zipmap(["a", "b", "c"], [1, 2, 3])
{
  "a" = 1
  "b" = 2
  "c" = 3
}

> toset(["a", "b", "a", "c"])  # removes duplicates
toset([
  "a",
  "b",
  "c",
])

> tolist(toset(["c", "a", "b"]))  # convert set to sorted list
tolist([
  "a",
  "b",
  "c",
])
```

---

## Step 324: Testing Complex For Expressions

### For Expressions ใน Console

```hcl
# ─── For expressions กับ lists ────────────────────────────

> [for s in ["hello", "world"] : upper(s)]
tolist([
  "HELLO",
  "WORLD",
])

> [for i, v in ["a", "b", "c"] : "${i}: ${v}"]
tolist([
  "0: a",
  "1: b",
  "2: c",
])

# Filter ด้วย if
> [for s in ["apple", "banana", "cherry", "apricot"] : s if startswith(s, "a")]
tolist([
  "apple",
  "apricot",
])

# For expression กับ numbers
> [for n in range(1, 6) : n * n]
tolist([
  1,
  4,
  9,
  16,
  25,
])

# ─── For expressions กับ maps ──────────────────────────────

> {for k, v in {a = 1, b = 2, c = 3} : k => v * 2}
{
  "a" = 2
  "b" = 4
  "c" = 6
}

> {for k, v in {a = 1, b = 2, c = 3} : upper(k) => v}
{
  "A" = 1
  "B" = 2
  "C" = 3
}

# Filter map entries
> {for k, v in {a = 1, b = 2, c = 3, d = 4} : k => v if v > 2}
{
  "c" = 3
  "d" = 4
}

# ─── Real-world For Expression Examples ───────────────────

# แปลง list of objects เป็น map
> {for subnet in [{name = "web", cidr = "10.0.1.0/24"}, {name = "app", cidr = "10.0.2.0/24"}] : subnet.name => subnet.cidr}
{
  "app" = "10.0.2.0/24"
  "web" = "10.0.1.0/24"
}

# สร้าง ingress rules จาก list
> [for port in [80, 443, 8080] : {from_port = port, to_port = port, protocol = "tcp"}]
tolist([
  {
    "from_port" = 80
    "protocol" = "tcp"
    "to_port" = 80
  },
  {
    "from_port" = 443
    "protocol" = "tcp"
    "to_port" = 443
  },
  {
    "from_port" = 8080
    "protocol" = "tcp"
    "to_port" = 8080
  },
])

# สร้าง tag map จากหลาย sources
> merge(
    {for k, v in {env = "prod", team = "platform"} : k => v},
    {Name = "web-server"}
  )
{
  "Name" = "web-server"
  "env" = "prod"
  "team" = "platform"
}
```

---

## Step 325: Accessing Variables ใน Console

### เข้าถึง Variables ผ่าน Console

```bash
# ต้องมี variables ที่ defined และ set ค่าไว้
# ผ่าน terraform.tfvars หรือ -var flag
```

```hcl
# variables.tf
variable "environment" {
  type    = string
  default = "development"
}

variable "region" {
  type    = string
  default = "ap-southeast-1"
}

variable "instance_count" {
  type    = number
  default = 3
}

variable "tags" {
  type = map(string)
  default = {
    Project = "myapp"
    Team    = "platform"
  }
}

variable "allowed_ports" {
  type    = list(number)
  default = [80, 443, 8080]
}
```

```bash
# เปิด console แล้วทดสอบ variables
terraform console

> var.environment
"development"

> var.region
"ap-southeast-1"

> var.instance_count
3

> var.tags
{
  "Project" = "myapp"
  "Team" = "platform"
}

> var.allowed_ports
tolist([
  80,
  443,
  8080,
])

# ใช้ variables ใน expressions
> "app-${var.environment}"
"app-development"

> "${var.region}-${var.environment}"
"ap-southeast-1-development"

> [for port in var.allowed_ports : "0.0.0.0/0:${port}"]
tolist([
  "0.0.0.0/0:80",
  "0.0.0.0/0:443",
  "0.0.0.0/0:8080",
])

# Conditional expression กับ variable
> var.environment == "production" ? "t3.large" : "t3.micro"
"t3.micro"

# ใช้ merge กับ var.tags
> merge(var.tags, {Environment = var.environment})
{
  "Environment" = "development"
  "Project" = "myapp"
  "Team" = "platform"
}
```

---

## Step 326: Accessing State Values ใน Console

### เข้าถึง State Values

```bash
# ต้องมี state file (หลัง terraform apply)
# เปิด console แล้วเข้าถึง resources

terraform console

# ─── EC2 Instance State ────────────────────────────────────

> aws_instance.web.id
"i-0abc123def456789"

> aws_instance.web.public_ip
"54.251.123.45"

> aws_instance.web.private_ip
"10.0.1.100"

> aws_instance.web.instance_type
"t3.micro"

> aws_instance.web.ami
"ami-0c02fb55956c7d316"

> aws_instance.web.tags
{
  "Name" = "web-server"
  "Environment" = "development"
}

# ─── VPC State ─────────────────────────────────────────────

> aws_vpc.main.id
"vpc-0abc123def"

> aws_vpc.main.cidr_block
"10.0.0.0/16"

# ─── For resources with count ──────────────────────────────

# ถ้ามี count = 3
> aws_instance.web[0].id
"i-0abc123def456789"

> aws_instance.web[1].id
"i-0def456ghi789012"

> aws_instance.web[*].id
tolist([
  "i-0abc123def456789",
  "i-0def456ghi789012",
  "i-0ghi789jkl012345",
])

> aws_instance.web[*].private_ip
tolist([
  "10.0.1.100",
  "10.0.1.101",
  "10.0.1.102",
])

# ─── For resources with for_each ───────────────────────────

# ถ้ามี for_each = {web = ..., app = ...}
> aws_security_group.this["web"].id
"sg-0abc123"

> {for k, v in aws_security_group.this : k => v.id}
{
  "app" = "sg-0def456"
  "web" = "sg-0abc123"
}

# ─── Module Outputs ────────────────────────────────────────

> module.vpc.vpc_id
"vpc-0abc123def"

> module.vpc.private_subnet_ids
tolist([
  "subnet-0abc123",
  "subnet-0def456",
])

> module.app.instance_ids
tolist([
  "i-0abc123def456789",
  "i-0def456ghi789012",
])
```

---

## Step 327: Batch Input to Console

### การใช้ Console แบบ Non-interactive (Batch Mode)

```bash
# ─── echo แบบ single expression ────────────────────────────

echo 'upper("hello world")' | terraform console
# Output: "HELLO WORLD"

echo 'cidrsubnet("10.0.0.0/16", 8, 1)' | terraform console
# Output: "10.0.1.0/24"

echo 'length(["a", "b", "c"])' | terraform console
# Output: 3

# ─── Multiple expressions ───────────────────────────────────

cat << 'EOF' | terraform console
upper("hello")
lower("WORLD")
format("%s-%s", "web", "01")
EOF
# Output:
# "HELLO"
# "world"
# "web-01"

# ─── ใน shell scripts ──────────────────────────────────────

#!/bin/bash
# ทดสอบ cidr calculations
RESULT=$(echo 'cidrsubnet("10.0.0.0/16", 8, 5)' | terraform console)
echo "Subnet CIDR: $RESULT"

# ─── Store output ──────────────────────────────────────────

echo '[for i in range(10) : cidrsubnet("10.0.0.0/16", 8, i)]' \
  | terraform console \
  | tee subnet_list.txt

# ─── ใน Python script ──────────────────────────────────────

import subprocess

def terraform_eval(expression, cwd="."):
    result = subprocess.run(
        ["terraform", "console"],
        input=expression,
        capture_output=True,
        text=True,
        cwd=cwd
    )
    return result.stdout.strip()

# ทดสอบ
print(terraform_eval('cidrsubnet("10.0.0.0/16", 8, 1)'))
print(terraform_eval('upper("hello")'))
```

---

## Step 328: Testing CIDR Functions

### CIDR Functions ที่สำคัญ

```hcl
# ─── cidrsubnet ────────────────────────────────────────────

# cidrsubnet(prefix, newbits, netnum)
# prefix: CIDR block เดิม
# newbits: จำนวน bits ที่เพิ่ม
# netnum: หมายเลข subnet

> cidrsubnet("10.0.0.0/16", 8, 0)
"10.0.0.0/24"

> cidrsubnet("10.0.0.0/16", 8, 1)
"10.0.1.0/24"

> cidrsubnet("10.0.0.0/16", 8, 10)
"10.0.10.0/24"

> cidrsubnet("10.0.0.0/16", 8, 255)
"10.0.255.0/24"

> cidrsubnet("10.0.0.0/8", 16, 1)
"10.0.1.0/24"

# สร้าง subnets หลายอัน
> [for i in range(6) : cidrsubnet("10.0.0.0/16", 8, i)]
tolist([
  "10.0.0.0/24",
  "10.0.1.0/24",
  "10.0.2.0/24",
  "10.0.3.0/24",
  "10.0.4.0/24",
  "10.0.5.0/24",
])

# ─── cidrhost ──────────────────────────────────────────────

# cidrhost(prefix, hostnum)
> cidrhost("10.0.1.0/24", 1)
"10.0.1.1"

> cidrhost("10.0.1.0/24", 10)
"10.0.1.10"

> cidrhost("10.0.1.0/24", 254)
"10.0.1.254"

# ─── cidrnetmask ───────────────────────────────────────────

> cidrnetmask("10.0.0.0/24")
"255.255.255.0"

> cidrnetmask("10.0.0.0/16")
"255.255.0.0"

> cidrnetmask("10.0.0.0/8")
"255.0.0.0"

> cidrnetmask("10.0.0.0/22")
"255.255.252.0"

# ─── cidrcontains ──────────────────────────────────────────

> cidrcontains("10.0.0.0/8", "10.5.0.0/16")
true

> cidrcontains("10.0.0.0/16", "10.1.0.0/24")
false

# ─── Real-world Example: Multi-AZ Subnet Calculation ──────

# สำหรับ VPC 10.0.0.0/16 ใน 3 AZs:
# - Public subnets:  10.0.0.0/24, 10.0.1.0/24, 10.0.2.0/24
# - Private subnets: 10.0.10.0/24, 10.0.11.0/24, 10.0.12.0/24
# - DB subnets:      10.0.20.0/24, 10.0.21.0/24, 10.0.22.0/24

> {
    public  = [for i in range(3) : cidrsubnet("10.0.0.0/16", 8, i)],
    private = [for i in range(3) : cidrsubnet("10.0.0.0/16", 8, i + 10)],
    db      = [for i in range(3) : cidrsubnet("10.0.0.0/16", 8, i + 20)]
  }
{
  "db" = tolist([
    "10.0.20.0/24",
    "10.0.21.0/24",
    "10.0.22.0/24",
  ])
  "private" = tolist([
    "10.0.10.0/24",
    "10.0.11.0/24",
    "10.0.12.0/24",
  ])
  "public" = tolist([
    "10.0.0.0/24",
    "10.0.1.0/24",
    "10.0.2.0/24",
  ])
}
```

---

## Step 329: Testing Regex Functions

### Regex Functions ใน Console

```hcl
# ─── regex ─────────────────────────────────────────────────

# regex(pattern, string) - returns first match
> regex("[0-9]+", "abc123def456")
"123"

> regex("[a-z]+", "ABC123def")
"def"

> regex("^([^-]+)-([^-]+)-(.+)$", "web-server-01")
tolist([
  "web",
  "server",
  "01",
])

# Extract IP from string
> regex("(\\d+\\.\\d+\\.\\d+\\.\\d+)", "Server at 10.0.1.100 port 80")
tolist([
  "10.0.1.100",
])

# ─── regexall ──────────────────────────────────────────────

# regexall(pattern, string) - returns all matches
> regexall("[0-9]+", "abc123def456ghi789")
tolist([
  "123",
  "456",
  "789",
])

> regexall("[a-z]+", "Hello World foo bar")
tolist([
  "ello",
  "orld",
  "foo",
  "bar",
])

# ─── can (ใช้กับ regex validation) ────────────────────────

# can(expression) - returns true if expression succeeds
> can(regex("^[a-z][a-z0-9-]*$", "valid-name-123"))
true

> can(regex("^[a-z][a-z0-9-]*$", "Invalid-Name!"))
false

> can(regex("^[a-z][a-z0-9-]*$", "123invalid"))
false

# ─── Real-world: Validating Names ──────────────────────────

# Validate S3 bucket naming rules
> can(regex("^[a-z0-9][a-z0-9.-]{1,61}[a-z0-9]$", "my-bucket-name"))
true

> can(regex("^[a-z0-9][a-z0-9.-]{1,61}[a-z0-9]$", "My-BUCKET"))
false

# Validate environment name
> can(regex("^(development|staging|production)$", "staging"))
true

> can(regex("^(development|staging|production)$", "dev"))
false

# ─── ใช้ใน variable validation ────────────────────────────

# ตัวอย่างใน variables.tf:
variable "environment" {
  type = string
  
  validation {
    condition     = can(regex("^(development|staging|production)$", var.environment))
    error_message = "Environment must be development, staging, or production."
  }
}
```

---

## Step 330: Testing templatefile() และ Functions อื่นๆ

### templatefile() ใน Console

```bash
# สร้าง template file ก่อน
cat > /tmp/test.tftpl << 'EOF'
#!/bin/bash
hostname "${hostname}"
export APP_ENV="${environment}"
export DB_HOST="${db_host}"
export REGION="${region}"

%{ for port in ports ~}
firewall-cmd --add-port=${port}/tcp
%{ endfor ~}
EOF
```

```hcl
# ทดสอบ templatefile ใน console
> templatefile("/tmp/test.tftpl", {
    hostname    = "web-server-01"
    environment = "production"
    db_host     = "rds.example.com"
    region      = "ap-southeast-1"
    ports       = [80, 443, 8080]
  })
<<EOT
#!/bin/bash
hostname "web-server-01"
export APP_ENV="production"
export DB_HOST="rds.example.com"
export REGION="ap-southeast-1"

firewall-cmd --add-port=80/tcp
firewall-cmd --add-port=443/tcp
firewall-cmd --add-port=8080/tcp

EOT
```

### JSON Functions

```hcl
# ─── jsonencode / jsondecode ───────────────────────────────

> jsonencode({name = "Alice", age = 30})
"{\"age\":30,\"name\":\"Alice\"}"

> jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect   = "Allow"
      Action   = ["s3:GetObject"]
      Resource = "arn:aws:s3:::my-bucket/*"
    }]
  })
"{\"Statement\":[{\"Action\":[\"s3:GetObject\"],\"Effect\":\"Allow\",\"Resource\":\"arn:aws:s3:::my-bucket/*\"}],\"Version\":\"2012-10-17\"}"

> jsondecode("{\"name\":\"Alice\",\"age\":30}")
{
  "age" = 30
  "name" = "Alice"
}

> jsondecode("{\"name\":\"Alice\",\"age\":30}").name
"Alice"

# ─── yamlencode / yamldecode ───────────────────────────────

> yamlencode({name = "Alice", tags = ["web", "frontend"]})
<<EOT
name: Alice
tags:
- web
- frontend

EOT

> yamldecode("name: Alice\nage: 30\n")
{
  "age" = 30
  "name" = "Alice"
}
```

### Type Conversion Functions

```hcl
# ─── Type conversions ──────────────────────────────────────

> tostring(42)
"42"

> tostring(true)
"true"

> tonumber("42")
42

> tonumber("3.14")
3.14

> tobool("true")
true

> tobool("false")
false

> tolist(toset(["c", "a", "b"]))
tolist([
  "a",
  "b",
  "c",
])

> toset(["a", "b", "a", "c"])
toset([
  "a",
  "b",
  "c",
])

> tomap({a = 1, b = 2})
{
  "a" = 1
  "b" = 2
}

# ─── try และ can ───────────────────────────────────────────

# try(expression, fallback) - คืน fallback ถ้า expression error
> try(tonumber("not-a-number"), 0)
0

> try(tonumber("42"), 0)
42

> try(lookup({a = 1}, "z"), "default")
"default"

> try(lookup({a = 1}, "a"), "default")
1
```

### Encoding Functions

```hcl
# ─── Base64 Encoding ───────────────────────────────────────

> base64encode("Hello, World!")
"SGVsbG8sIFdvcmxkIQ=="

> base64decode("SGVsbG8sIFdvcmxkIQ==")
"Hello, World!"

> base64gzip("Hello, World!")
# Gzip then base64 (useful for user data)

# ─── Hash Functions ────────────────────────────────────────

> md5("hello")
"5d41402abc4b2a76b9719d911017c592"

> sha1("hello")
"aaf4c61ddcc5e8a2dabede0f3b482cd9aea9434d"

> sha256("hello")
"2cf24dba5fb0a30e26e83b2ac5b9e29e1b161e5c1fa7425e73043362938b9824"

> sha512("hello")
"9b71d224bd62f3785d96d46ad3ea3d73319bfbc2890caadae2dff72519673ca72323c3d99ba5c11d7c7acc6e14b8c5da0c4663475c2e5c3adef46f73bcdec043"

# ─── UUID / Random Functions ───────────────────────────────

> uuid()
"1b9d6bcd-bbfd-4b2d-9b5d-ab8dfbbd4bed"

> uuidv5("dns", "www.example.com")
"2ed6423a-d21a-5613-a4e3-4a8f0b8c10b0"
```

### Filesystem Functions

```hcl
# ─── File Functions ────────────────────────────────────────

# อ่านไฟล์ (ต้องมีไฟล์จริงอยู่)
> file("user_data.sh")
"#!/bin/bash\nyum update -y\n..."

> filebase64("user_data.sh")
"IyEvYmluL2Jhc2gKeXVtIHVwZGF0ZSAteQo..."

> filesha256("main.tf")
"2cf24dba5fb0a30e26e83b2ac5b9e29e1b161e5c1fa7425e73043362938b9824"

> fileexists("main.tf")
true

> fileexists("nonexistent.tf")
false

# ─── Path Functions ────────────────────────────────────────

> path.module
"."

> path.root
"."

> path.cwd
"/home/user/terraform-project"

> basename("/home/user/main.tf")
"main.tf"

> dirname("/home/user/main.tf")
"/home/user"
```

---

## Step 330b: Console สำหรับ Debugging

### Debugging Patterns

```hcl
# Pattern 1: Debug complex conditional
> var.environment == "production" ? 3 : 1
1  # (ถ้า environment = development)

# Pattern 2: Debug for expression output
> [for az in ["a", "b", "c"] : "ap-southeast-1${az}"]
tolist([
  "ap-southeast-1a",
  "ap-southeast-1b",
  "ap-southeast-1c",
])

# Pattern 3: Debug merge
> merge(
    {for k, v in {env = "prod"} : "tag_${k}" => v},
    {managed_by = "terraform"}
  )
{
  "managed_by" = "terraform"
  "tag_env" = "prod"
}

# Pattern 4: Debug cidrsubnet calculations
> {
    "public_a"  = cidrsubnet("10.0.0.0/16", 8, 1),
    "public_b"  = cidrsubnet("10.0.0.0/16", 8, 2),
    "private_a" = cidrsubnet("10.0.0.0/16", 8, 10),
    "private_b" = cidrsubnet("10.0.0.0/16", 8, 11)
  }
{
  "private_a" = "10.0.10.0/24"
  "private_b" = "10.0.11.0/24"
  "public_a" = "10.0.1.0/24"
  "public_b" = "10.0.2.0/24"
}

# Pattern 5: Debug state attribute
> aws_instance.web.public_ip
"54.251.123.45"

> "http://${aws_instance.web.public_ip}:${var.app_port}/health"
"http://54.251.123.45:8080/health"

# Pattern 6: Debug complex data transformation
> {
    for k, v in {
      web = {port = 80, protocol = "http"},
      api = {port = 443, protocol = "https"}
    } : "${k}-${v.protocol}" => v.port
  }
{
  "api-https" = 443
  "web-http" = 80
}
```

### Console vs terraform output

```bash
# terraform console - สำหรับ testing expressions
terraform console
> 2 + 2

# terraform output - ดู output values จาก state
terraform output
# web_url = "http://54.251.123.45:80"
# db_endpoint = "rds.example.com:5432"

# ดู output เฉพาะ
terraform output web_url

# ดู output แบบ raw (ไม่มี quotes)
terraform output -raw web_url

# ดู output แบบ JSON
terraform output -json
```

---

## สรุป: Terraform Console Best Practices

### เมื่อไหรที่ควรใช้ Console?

| สถานการณ์ | ใช้ Console? |
|-----------|-------------|
| ทดสอบ function syntax | ✅ ใช้ |
| Debug for expressions | ✅ ใช้ |
| ทดสอบ CIDR calculations | ✅ ใช้ |
| ดู state values | ✅ ใช้ |
| ทดสอบ regex patterns | ✅ ใช้ |
| ดู output values | ❌ ใช้ `terraform output` แทน |
| Apply configuration | ❌ ใช้ `terraform apply` แทน |

### Quick Reference: Functions ที่ใช้บ่อย

```bash
# String
echo 'format("%-10s %5d", "item", 42)' | terraform console
echo 'trimspace("  hello  ")' | terraform console
echo 'split(",", "a,b,c")' | terraform console

# Collection  
echo 'flatten([["a","b"],["c","d"]])' | terraform console
echo 'distinct(["a","b","a"])' | terraform console
echo 'compact(["a","","b"])' | terraform console

# CIDR
echo 'cidrsubnet("10.0.0.0/16", 8, 5)' | terraform console
echo 'cidrhost("10.0.1.0/24", 10)' | terraform console

# Encoding
echo 'base64encode("hello")' | terraform console
echo 'jsonencode({key = "value"})' | terraform console

# Type checking
echo 'try(tonumber("abc"), 0)' | terraform console
echo 'can(regex("^[0-9]+$", "123"))' | terraform console
```

---

*จบ Part 033: Terraform Console & Expressions*

*ต่อไป: Part 034 - Terraform Lock File (.terraform.lock.hcl)*
