# Part 006: HCL Built-in Functions
## ฟังก์ชันในตัวของ HCL (Steps 51-60)

---

## Step 51: String Functions

### format() และ formatlist()

```hcl
locals {
  # format() - C-style string formatting
  greeting    = format("Hello, %s!", "World")          # "Hello, World!"
  padded_num  = format("%05d", 42)                     # "00042"
  float_str   = format("%.2f", 3.14159)               # "3.14"
  hex_str     = format("0x%X", 255)                    # "0xFF"
  
  # Multiple args
  server_info = format("Server %s is at %s:%d", "web-01", "10.0.1.1", 80)
  # "Server web-01 is at 10.0.1.1:80"
  
  # formatlist() - format list elements
  server_names = formatlist("server-%02d", range(1, 6))
  # ["server-01", "server-02", "server-03", "server-04", "server-05"]
  
  cidr_blocks  = formatlist("10.0.%d.0/24", range(1, 4))
  # ["10.0.1.0/24", "10.0.2.0/24", "10.0.3.0/24"]
  
  # formatlist กับ 2 lists (zip operation)
  hosts        = ["web-01", "web-02", "api-01"]
  ips          = ["10.0.1.1", "10.0.1.2", "10.0.2.1"]
  host_entries = formatlist("%s\t%s", local.ips, local.hosts)
  # ["10.0.1.1\tweb-01", "10.0.1.2\tweb-02", "10.0.2.1\tapi-01"]
}
```

### join() และ split()

```hcl
locals {
  # join() - รวม list elements เป็น string
  words    = ["Hello", "World", "Terraform"]
  joined_space  = join(" ", local.words)    # "Hello World Terraform"
  joined_comma  = join(", ", local.words)   # "Hello, World, Terraform"
  joined_newline = join("\n", local.words)  # "Hello\nWorld\nTerraform"
  joined_empty  = join("", local.words)    # "HelloWorldTerraform"
  
  # Practical: join IPs for security group
  allowed_ips  = ["10.0.1.0/24", "10.0.2.0/24", "192.168.1.0/24"]
  cidrs_string = join(",", local.allowed_ips)
  # "10.0.1.0/24,10.0.2.0/24,192.168.1.0/24"
  
  # split() - แยก string เป็น list
  csv = "apple,banana,cherry,date"
  fruits_list = split(",", local.csv)       # ["apple", "banana", "cherry", "date"]
  
  arn = "arn:aws:s3:::my-bucket"
  arn_parts = split(":", local.arn)
  # ["arn", "aws", "s3", "", "", "my-bucket"]
  
  # Split path
  path = "/var/log/nginx/access.log"
  path_parts = split("/", local.path)
  # ["", "var", "log", "nginx", "access.log"]
  filename = local.path_parts[length(local.path_parts) - 1]  # "access.log"
}
```

### replace(), regex(), regexall()

```hcl
locals {
  text = "Hello World Hello Terraform"
  
  # replace() - replace string
  replaced = replace(local.text, "Hello", "Hi")
  # "Hi World Hi Terraform"
  
  # Replace with regex
  sanitized = replace("my resource name!", "/[^a-zA-Z0-9-]/", "-")
  # "my-resource-name-"
  
  # regex() - return first match
  email    = "Contact: admin@example.com for help"
  addr     = regex("[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\\.[a-zA-Z]{2,}", local.email)
  # "admin@example.com"
  
  # regex กับ capture groups
  version_str = "terraform-v1.5.0"
  version_match = regex("v(\\d+)\\.(\\d+)\\.(\\d+)", local.version_str)
  # ["1", "5", "0"] (capture groups)
  major = version_match[0]  # "1"
  minor = version_match[1]  # "5"
  patch = version_match[2]  # "0"
  
  # regexall() - return all matches
  log_line = "Error at line 10, warning at line 25, error at line 42"
  line_nums = regexall("line (\\d+)", local.log_line)
  # [["line 10", "10"], ["line 25", "25"], ["line 42", "42"]]
  
  only_nums = [for match in local.line_nums : match[1]]
  # ["10", "25", "42"]
}
```

### trim() Functions

```hcl
locals {
  # trimspace() - ลบ whitespace ทั้งสองด้าน
  spaced = "  Hello World  "
  clean  = trimspace(local.spaced)  # "Hello World"
  
  # trim() - ลบ characters ที่กำหนด
  path_with_slash = "/var/log/"
  clean_path = trim(local.path_with_slash, "/")  # "var/log"
  
  # trimleft() - ลบทางซ้าย
  left_trim = trimleft("---hello---", "-")  # "hello---"
  
  # trimright() - ลบทางขวา
  right_trim = trimright("---hello---", "-")  # "---hello"
  
  # trimprefix() - ลบ prefix
  full_name    = "aws_instance"
  without_aws  = trimprefix(local.full_name, "aws_")  # "instance"
  
  # trimsuffix() - ลบ suffix
  filename     = "main.tf"
  without_ext  = trimsuffix(local.filename, ".tf")  # "main"
}
```

### upper(), lower(), title(), substr()

```hcl
locals {
  mixed = "Hello World Terraform"
  
  # Case conversion
  upper  = upper(local.mixed)  # "HELLO WORLD TERRAFORM"
  lower  = lower(local.mixed)  # "hello world terraform"
  title  = title(local.mixed)  # "Hello World Terraform" (already title case)
  
  # substr() - get substring
  str = "Hello, World!"
  
  from_start    = substr(local.str, 0, 5)   # "Hello"
  from_middle   = substr(local.str, 7, 5)   # "World"
  
  # Negative length = til end
  from_seven    = substr(local.str, 7, -1)  # "World!"
  
  # length() - string length
  str_len = length(local.str)  # 13
  
  # Practical: truncate string
  max_len     = 20
  long_str    = "This is a very long string that needs to be truncated"
  truncated   = substr(local.long_str, 0, local.max_len)
  # "This is a very long "
}
```

### startswith(), endswith(), strcontains()

```hcl
locals {
  resource_name = "aws_instance_web_server"
  
  # startswith()
  is_aws     = startswith(local.resource_name, "aws_")       # true
  is_gcp     = startswith(local.resource_name, "google_")    # false
  
  # endswith()
  is_server  = endswith(local.resource_name, "_server")      # true
  is_bucket  = endswith(local.resource_name, "_bucket")      # false
  
  # strcontains()
  has_web    = strcontains(local.resource_name, "web")        # true
  has_db     = strcontains(local.resource_name, "database")   # false
  
  # Practical: validate naming convention
  names = [
    "aws_instance_web",
    "aws_s3_bucket",
    "google_compute_instance",  # invalid
    "aws_rds_cluster",
  ]
  
  aws_resources = [
    for name in local.names : name
    if startswith(name, "aws_")
  ]
  # ["aws_instance_web", "aws_s3_bucket", "aws_rds_cluster"]
}
```

---

## Step 52: Collection Functions

### length(), concat(), flatten()

```hcl
locals {
  list1 = [1, 2, 3]
  list2 = [4, 5, 6]
  list3 = [7, 8, 9]
  
  # length() - count elements
  count1 = length(local.list1)  # 3
  
  # string length
  str_len = length("hello")  # 5
  
  # map length
  map_len = length({ a = 1, b = 2 })  # 2
  
  # concat() - combine lists
  combined = concat(local.list1, local.list2, local.list3)
  # [1, 2, 3, 4, 5, 6, 7, 8, 9]
  
  # flatten() - flatten nested lists
  nested = [
    [1, 2, 3],
    [4, 5],
    [[6, 7], [8, 9]],
  ]
  flat = flatten(local.nested)
  # [1, 2, 3, 4, 5, 6, 7, 8, 9]
  
  # Practical: flatten subnets from multiple VPCs
  vpc_subnets = {
    vpc1 = ["10.0.1.0/24", "10.0.2.0/24"]
    vpc2 = ["10.1.1.0/24", "10.1.2.0/24"]
    vpc3 = ["10.2.1.0/24"]
  }
  
  all_subnets = flatten(values(local.vpc_subnets))
  # ["10.0.1.0/24", "10.0.2.0/24", "10.1.1.0/24", "10.1.2.0/24", "10.2.1.0/24"]
}
```

### merge(), keys(), values(), zipmap()

```hcl
locals {
  defaults = {
    instance_type = "t3.micro"
    disk_size     = 20
    monitoring    = false
  }
  
  overrides = {
    instance_type = "t3.large"   # override
    backup        = true          # new key
  }
  
  # merge() - merge maps (later maps win)
  merged = merge(local.defaults, local.overrides)
  # {
  #   instance_type = "t3.large"  ← from overrides
  #   disk_size     = 20           ← from defaults
  #   monitoring    = false        ← from defaults
  #   backup        = true         ← from overrides
  # }
  
  config = { host = "db.example.com", port = "5432", name = "mydb" }
  
  # keys() - get keys as sorted list
  config_keys = keys(local.config)   # ["host", "name", "port"]
  
  # values() - get values (in same order as keys())
  config_vals = values(local.config) # ["db.example.com", "mydb", "5432"]
  
  # zipmap() - create map from 2 lists
  k = ["name", "age", "city"]
  v = ["Alice", "30", "Bangkok"]
  person = zipmap(local.k, local.v)
  # { name = "Alice", age = "30", city = "Bangkok" }
  
  # Reverse: map keys/values
  reversed = zipmap(values(local.config), keys(local.config))
  # { "db.example.com" = "host", "5432" = "port", "mydb" = "name" }
}
```

### toset(), tolist(), tomap(), range()

```hcl
locals {
  # toset() - convert to set (remove duplicates)
  with_dups  = ["a", "b", "a", "c", "b"]
  unique_set = toset(local.with_dups)   # {"a", "b", "c"}
  
  # tolist() - convert to list
  my_set  = toset(["c", "a", "b"])
  my_list = tolist(local.my_set)        # ["a", "b", "c"] (sorted)
  
  # tomap() - convert to map
  my_obj  = { a = 1, b = 2 }
  my_map  = tomap(local.my_obj)         # map(number) type
  
  # range() - generate sequence
  zero_to_four  = range(5)              # [0, 1, 2, 3, 4]
  one_to_five   = range(1, 6)           # [1, 2, 3, 4, 5]
  evens_to_ten  = range(0, 11, 2)       # [0, 2, 4, 6, 8, 10]
  countdown     = range(5, 0, -1)       # [5, 4, 3, 2, 1]
  
  # Practical: generate ports
  app_ports = range(8000, 8010)
  # [8000, 8001, 8002, 8003, 8004, 8005, 8006, 8007, 8008, 8009]
  
  # Generate IP last octets
  ip_octets = range(1, 11)
  private_ips = formatlist("10.0.0.%d", local.ip_octets)
  # ["10.0.0.1", "10.0.0.2", ..., "10.0.0.10"]
}
```

### sort(), reverse(), distinct(), chunklist()

```hcl
locals {
  unsorted = ["banana", "apple", "cherry", "date"]
  nums     = [5, 3, 1, 4, 2]
  
  # sort() - alphabetic for strings, numeric for numbers
  sorted_strings = sort(local.unsorted)       # ["apple", "banana", "cherry", "date"]
  sorted_nums    = sort(local.nums)           # [1, 2, 3, 4, 5]
  
  # reverse() - reverse a list
  reversed = reverse(local.sorted_strings)    # ["date", "cherry", "banana", "apple"]
  
  # distinct() - remove duplicates (preserve order)
  with_dups    = ["a", "b", "a", "c", "b", "d"]
  unique       = distinct(local.with_dups)    # ["a", "b", "c", "d"]
  
  # chunklist() - split list into chunks
  items      = [1, 2, 3, 4, 5, 6, 7, 8, 9, 10]
  chunks_3   = chunklist(local.items, 3)
  # [[1, 2, 3], [4, 5, 6], [7, 8, 9], [10]]
  
  chunks_4   = chunklist(local.items, 4)
  # [[1, 2, 3, 4], [5, 6, 7, 8], [9, 10]]
  
  # Practical: batch process items
  all_servers = ["s1", "s2", "s3", "s4", "s5", "s6", "s7"]
  batch_size  = 3
  batches     = chunklist(local.all_servers, local.batch_size)
  # [["s1","s2","s3"], ["s4","s5","s6"], ["s7"]]
}
```

### setintersection(), setunion(), setsubtract()

```hcl
locals {
  # Set operations
  team_a_access  = toset(["s3", "ec2", "rds", "lambda"])
  team_b_access  = toset(["ec2", "rds", "sqs", "sns"])
  
  # Intersection - resources ที่ทั้งสอง teams มี access
  common_access = setintersection(local.team_a_access, local.team_b_access)
  # {"ec2", "rds"}
  
  # Union - resources ที่ team ใดก็ตามมี access
  all_access = setunion(local.team_a_access, local.team_b_access)
  # {"ec2", "lambda", "rds", "s3", "sns", "sqs"}
  
  # Subtract - resources ที่ team a มีแต่ team b ไม่มี
  exclusive_a = setsubtract(local.team_a_access, local.team_b_access)
  # {"lambda", "s3"}
  
  exclusive_b = setsubtract(local.team_b_access, local.team_a_access)
  # {"sns", "sqs"}
}
```

---

## Step 53: Numeric Functions

### abs(), ceil(), floor(), max(), min()

```hcl
locals {
  # abs() - absolute value
  abs_neg = abs(-42)     # 42
  abs_pos = abs(42)      # 42
  abs_flt = abs(-3.14)   # 3.14
  
  # ceil() - round up
  ceil_up   = ceil(4.1)   # 5
  ceil_down = ceil(4.9)   # 5
  ceil_neg  = ceil(-4.1)  # -4 (towards zero when negative)
  
  # floor() - round down
  floor_up   = floor(4.9)   # 4
  floor_down = floor(4.1)   # 4
  floor_neg  = floor(-4.1)  # -5 (away from zero when negative)
  
  # max() - maximum value
  max_val = max(3, 1, 4, 1, 5, 9, 2, 6)  # 9
  
  # min() - minimum value
  min_val = min(3, 1, 4, 1, 5, 9, 2, 6)  # 1
  
  # Practical usage
  count = 3
  min_ha_count = max(local.count, 2)  # ต้องมีอย่างน้อย 2 สำหรับ HA
  # max(3, 2) = 3
  
  cpu_usage = 85.7
  rounded_cpu = ceil(local.cpu_usage)  # 86 (round up สำหรับ capacity planning)
}
```

### pow(), signum(), log(), parseint()

```hcl
locals {
  # pow() - power
  square = pow(5, 2)    # 25
  cube   = pow(2, 10)   # 1024 (2^10)
  sqrt_  = pow(16, 0.5) # 4 (square root)
  
  # signum() - sign of number
  positive = signum(42)   # 1
  negative = signum(-42)  # -1
  zero_val = signum(0)    # 0
  
  # log() - logarithm
  log10_100 = log(100, 10)   # 2 (log base 10 of 100)
  log2_1024 = log(1024, 2)   # 10 (log base 2 of 1024)
  natural   = log(2.71828, 2.71828)  # ≈ 1 (natural log)
  
  # parseint() - parse string to integer
  hex_num    = parseint("FF", 16)      # 255
  bin_num    = parseint("1010", 2)     # 10
  oct_num    = parseint("17", 8)       # 15
  dec_num    = parseint("42", 10)      # 42
  
  # Practical: convert binary string to number
  binary = "11001010"
  decimal = parseint(local.binary, 2)  # 202
}
```

### sum() function

```hcl
locals {
  nums = [1, 2, 3, 4, 5]
  
  # sum() - sum of list
  total    = sum(local.nums)   # 15
  average  = sum(local.nums) / length(local.nums)  # 3.0
  
  # Practical: sum costs
  instance_costs = [0.023, 0.046, 0.023, 0.092]
  total_cost = sum(local.instance_costs)  # 0.184
  
  # Sum specific fields from objects
  servers = [
    { name = "web-01", vcpu = 2, ram_gb = 4  },
    { name = "web-02", vcpu = 2, ram_gb = 4  },
    { name = "api-01", vcpu = 4, ram_gb = 8  },
    { name = "db-01",  vcpu = 8, ram_gb = 16 },
  ]
  
  total_vcpu = sum([for s in local.servers : s.vcpu])   # 16
  total_ram  = sum([for s in local.servers : s.ram_gb]) # 32
}
```

---

## Step 54: Date/Time Functions

### timestamp(), formatdate(), timeadd()

```hcl
locals {
  # timestamp() - current time in RFC3339 format
  now = timestamp()
  # "2024-01-15T10:30:00Z"
  
  # formatdate() - format timestamp
  iso_date     = formatdate("YYYY-MM-DD", local.now)
  # "2024-01-15"
  
  human_date   = formatdate("DD MMMM YYYY", local.now)
  # "15 January 2024"
  
  time_only    = formatdate("HH:mm:ss", local.now)
  # "10:30:00"
  
  with_tz      = formatdate("YYYY-MM-DD'T'hh:mm:ssZ", local.now)
  # "2024-01-15T10:30:00+0000"
  
  # timeadd() - add duration to timestamp
  one_hour_later   = timeadd(local.now, "1h")
  one_day_later    = timeadd(local.now, "24h")
  one_week_later   = timeadd(local.now, "168h")  # 7 * 24h
  minus_one_hour   = timeadd(local.now, "-1h")
  thirty_minutes   = timeadd(local.now, "30m")
  
  # Practical: certificate expiry
  cert_start   = timestamp()
  cert_end     = timeadd(local.cert_start, "8760h")  # 1 year = 8760h
  cert_expiry  = formatdate("YYYY-MM-DD", local.cert_end)
}
```

### formatdate Format Codes

```
formatdate Format Codes:
┌────────────────────────────────────────────────────────────┐
│ YYYY  → 4-digit year (2024)                               │
│ YY    → 2-digit year (24)                                 │
│ MMMM  → Full month name (January)                         │
│ MMM   → Short month name (Jan)                            │
│ MM    → 2-digit month (01-12)                             │
│ M     → Month (1-12)                                      │
│ DD    → 2-digit day (01-31)                               │
│ D     → Day (1-31)                                        │
│ HH    → 24-hour hour (00-23)                              │
│ hh    → 12-hour hour (01-12)                              │
│ mm    → Minutes (00-59)                                   │
│ ss    → Seconds (00-59)                                   │
│ AA    → AM/PM                                             │
│ Z     → Timezone offset (±HHMM)                           │
│ ZZZZ  → Timezone name (UTC, +07:00)                       │
└────────────────────────────────────────────────────────────┘
```

---

## Step 55: Filesystem Functions

### file(), filebase64(), templatefile()

```hcl
# file() - read file content as string
data "aws_iam_policy_document" "custom" {
  source_policy_documents = [file("${path.module}/policies/base-policy.json")]
}

resource "aws_key_pair" "deployer" {
  key_name   = "deployer-key"
  public_key = file("~/.ssh/id_rsa.pub")  # read SSH public key
}

# filebase64() - read file as base64 encoded string
resource "aws_lambda_function" "app" {
  filename         = "${path.module}/lambda.zip"
  function_name    = "my-function"
  role             = aws_iam_role.lambda.arn
  handler          = "index.handler"
  runtime          = "nodejs18.x"
  source_code_hash = filebase64sha256("${path.module}/lambda.zip")
}

# Embed base64 file content
resource "aws_instance" "web" {
  ami           = var.ami_id
  instance_type = "t3.micro"
  
  # filebase64 สำหรับ binary data
  user_data_base64 = filebase64("${path.module}/scripts/init.sh")
}
```

### templatefile()

```hcl
# สร้างไฟล์ template ก่อน: templates/user_data.sh.tpl
# #!/bin/bash
# export APP_NAME="${app_name}"
# export ENVIRONMENT="${environment}"
# export DB_HOST="${db_host}"
# export DB_PORT="${db_port}"
# 
# yum update -y
# %{ for pkg in packages ~}
# yum install -y ${pkg}
# %{ endfor ~}
# 
# systemctl start ${app_name}

resource "aws_instance" "web" {
  ami           = var.ami_id
  instance_type = var.instance_type
  
  # templatefile() - render template with variables
  user_data = templatefile("${path.module}/templates/user_data.sh.tpl", {
    app_name    = var.app_name
    environment = var.environment
    db_host     = aws_db_instance.main.endpoint
    db_port     = aws_db_instance.main.port
    packages    = ["nginx", "nodejs", "pm2"]
  })
}

# templatefile กับ JSON template
# templates/config.json.tpl:
# {
#   "app": "${app_name}",
#   "env": "${environment}",
#   "services": ${jsonencode(services)}
# }

resource "aws_ssm_parameter" "config" {
  name  = "/app/${var.environment}/config"
  type  = "String"
  value = templatefile("${path.module}/templates/config.json.tpl", {
    app_name    = var.app_name
    environment = var.environment
    services    = ["api", "worker", "scheduler"]
  })
}
```

### fileexists(), fileset(), dirname(), basename()

```hcl
locals {
  # fileexists() - check if file exists
  has_custom_policy = fileexists("${path.module}/policies/custom.json")
  
  # Conditional based on file existence
  policy_file = local.has_custom_policy ? (
    "${path.module}/policies/custom.json"
  ) : (
    "${path.module}/policies/default.json"
  )
  
  # fileset() - get files matching pattern
  lambda_functions = fileset("${path.module}/lambdas", "*.zip")
  # {"function1.zip", "function2.zip", "function3.zip"}
  
  template_files  = fileset("${path.module}/templates", "**/*.tpl")
  # Set of all .tpl files recursively
  
  # dirname() / basename() - path manipulation
  full_path = "/var/app/config/production.json"
  dir       = dirname(local.full_path)   # "/var/app/config"
  file_name = basename(local.full_path)  # "production.json"
}

# สร้าง Lambda functions จาก directory
locals {
  lambda_files = fileset("${path.module}/lambdas", "*.zip")
  
  lambda_configs = {
    for f in local.lambda_files :
    trimsuffix(f, ".zip") => {
      filename = "${path.module}/lambdas/${f}"
      name     = trimsuffix(f, ".zip")
    }
  }
}

resource "aws_lambda_function" "functions" {
  for_each = local.lambda_configs
  
  filename      = each.value.filename
  function_name = "${var.project}-${each.value.name}"
  role          = aws_iam_role.lambda.arn
  handler       = "index.handler"
  runtime       = "nodejs18.x"
}
```

---

## Step 56: Hash Functions

### base64encode(), base64decode()

```hcl
locals {
  # base64encode() - encode string to base64
  original      = "Hello, World! This is a secret."
  encoded       = base64encode(local.original)
  # "SGVsbG8sIFdvcmxkISBUaGlzIGlzIGEgc2VjcmV0Lg=="
  
  # base64decode() - decode base64 to string
  decoded = base64decode(local.encoded)
  # "Hello, World! This is a secret."
  
  # Practical: encode configuration
  config_json   = jsonencode({
    server = "db.example.com"
    port   = 5432
    name   = "myapp"
  })
  config_b64    = base64encode(local.config_json)
}

# ส่ง encoded data ไปยัง EC2 instance
resource "aws_instance" "web" {
  ami           = var.ami_id
  instance_type = "t3.micro"
  
  # user_data ต้อง base64 encoded เมื่อใช้ user_data_base64
  user_data_base64 = base64encode(<<-EOF
    #!/bin/bash
    echo "Starting setup..."
    export APP_ENV=${var.environment}
  EOF
  )
}
```

### md5(), sha256(), bcrypt()

```hcl
locals {
  password = "my-super-secret-password"
  content  = "important configuration data"
  
  # md5() - MD5 hash (ใช้สำหรับ checksums ไม่ใช่ security)
  content_hash = md5(local.content)
  # "d8e8fca2dc0f896fd7cb4cb0031ba249"
  
  # sha1() - SHA1 hash
  sha1_hash = sha1(local.content)
  
  # sha256() - SHA256 hash
  sha256_hash  = sha256(local.content)
  # "a63ac...sha256 hash..."
  
  # sha512() - SHA512 hash
  sha512_hash  = sha512(local.content)
  
  # bcrypt() - bcrypt hash (สำหรับ passwords)
  # ⚠️ cost parameter 10-31 (higher = slower = more secure)
  hashed_pwd = bcrypt(local.password, 10)
  # "$2a$10$..."
}

# Practical: ใช้ hash สำหรับ cache busting
locals {
  config_content = templatefile("${path.module}/templates/config.tpl", var.config)
  config_hash    = md5(local.config_content)
}

resource "aws_ssm_parameter" "config" {
  name  = "/app/config-${local.config_hash}"  # unique per content
  type  = "String"
  value = local.config_content
}

# filemd5(), filesha256() - hash file content
resource "aws_lambda_function" "app" {
  filename         = "${path.module}/function.zip"
  function_name    = "my-function"
  role             = aws_iam_role.lambda.arn
  handler          = "index.handler"
  runtime          = "nodejs18.x"
  
  # เปลี่ยน hash เมื่อ file เปลี่ยน
  source_code_hash = filebase64sha256("${path.module}/function.zip")
}
```

---

## Step 57: IP Network Functions

### cidrhost(), cidrnetmask(), cidrsubnet()

```hcl
locals {
  vpc_cidr = "10.0.0.0/16"
  
  # cidrhost() - get specific IP from CIDR
  network_addr   = cidrhost(local.vpc_cidr, 0)    # "10.0.0.0" (network)
  first_host     = cidrhost(local.vpc_cidr, 1)    # "10.0.0.1"
  second_host    = cidrhost(local.vpc_cidr, 2)    # "10.0.0.2"
  broadcast      = cidrhost(local.vpc_cidr, -1)   # "10.0.255.255" (broadcast)
  last_usable    = cidrhost(local.vpc_cidr, -2)   # "10.0.255.254"
  
  # cidrnetmask() - get subnet mask
  netmask_16     = cidrnetmask("10.0.0.0/16")     # "255.255.0.0"
  netmask_24     = cidrnetmask("10.0.0.0/24")     # "255.255.255.0"
  netmask_28     = cidrnetmask("10.0.0.0/28")     # "255.255.255.240"
  
  # cidrsubnet() - calculate subnet CIDR
  # cidrsubnet(prefix, newbits, netnum)
  subnet_0       = cidrsubnet(local.vpc_cidr, 8, 0)  # "10.0.0.0/24"
  subnet_1       = cidrsubnet(local.vpc_cidr, 8, 1)  # "10.0.1.0/24"
  subnet_10      = cidrsubnet(local.vpc_cidr, 8, 10) # "10.0.10.0/24"
  
  # /20 subnets from /16
  large_subnet_0 = cidrsubnet(local.vpc_cidr, 4, 0)  # "10.0.0.0/20"
  large_subnet_1 = cidrsubnet(local.vpc_cidr, 4, 1)  # "10.0.16.0/20"
}
```

### cidrsubnets() และ cidrcontains()

```hcl
locals {
  vpc_cidr = "10.0.0.0/16"
  
  # cidrsubnets() - calculate multiple subnets at once
  # cidrsubnets(prefix, newbits...)
  auto_subnets = cidrsubnets(local.vpc_cidr, 8, 8, 8, 4)
  # [
  #   "10.0.0.0/24"   (8 new bits → /24)
  #   "10.0.1.0/24"   (8 new bits → /24)
  #   "10.0.2.0/24"   (8 new bits → /24)
  #   "10.0.16.0/20"  (4 new bits → /20)
  # ]
  
  # Generate subnets for multiple AZs
  az_count = 3
  public_subnets = [
    for i in range(local.az_count) :
    cidrsubnet(local.vpc_cidr, 8, i)  # 10.0.0.0/24, 10.0.1.0/24, 10.0.2.0/24
  ]
  
  private_subnets = [
    for i in range(local.az_count) :
    cidrsubnet(local.vpc_cidr, 8, i + 10)  # 10.0.10.0/24, 10.0.11.0/24, 10.0.12.0/24
  ]
  
  # cidrcontains() - check if IP is in CIDR (Terraform 1.5+)
  is_in_vpc    = cidrcontains(local.vpc_cidr, "10.0.5.1")    # true
  is_not_in_vpc = cidrcontains(local.vpc_cidr, "192.168.1.1") # false
}

# สร้าง subnets จาก calculation
resource "aws_subnet" "public" {
  count = length(local.public_subnets)
  
  vpc_id            = aws_vpc.main.id
  cidr_block        = local.public_subnets[count.index]
  availability_zone = var.availability_zones[count.index]
  
  tags = {
    Name = "public-${var.availability_zones[count.index]}"
    Type = "public"
  }
}
```

---

## Step 58: Type Conversion Functions

### tostring(), tonumber(), tobool()

```hcl
locals {
  # tostring() - convert to string
  num_str   = tostring(42)       # "42"
  bool_str  = tostring(true)     # "true"
  float_str = tostring(3.14)     # "3.14"
  null_str  = tostring(null)     # null (null stays null)
  
  # tonumber() - convert to number
  str_int   = tonumber("42")     # 42
  str_float = tonumber("3.14")   # 3.14
  # tonumber("abc") → error!
  
  # Safe conversion
  safe_num = can(tonumber(var.port_str)) ? tonumber(var.port_str) : 8080
  
  # tobool() - convert to bool
  true_val  = tobool("true")    # true
  false_val = tobool("false")   # false
  # tobool("yes") → error! (only "true"/"false" allowed)
  
  # Safe bool conversion
  is_enabled = try(tobool(var.feature_flag), false)
}
```

### tolist(), toset(), tomap()

```hcl
locals {
  # tolist() - convert tuple/set to list
  tuple_val = ["a", "b", "c"]
  as_list   = tolist(local.tuple_val)  # list(string)
  
  set_val   = toset(["c", "a", "b"])
  set_as_list = tolist(local.set_val)  # ["a", "b", "c"] (sorted)
  
  # toset() - convert list to set (removes duplicates)
  list_val  = ["a", "b", "a", "c"]
  as_set    = toset(local.list_val)    # {"a", "b", "c"}
  
  # tomap() - convert object to map
  obj_val   = { key1 = "val1", key2 = "val2" }
  as_map    = tomap(local.obj_val)    # map(string)
  
  # Practical: ensure type consistency
  mixed_tags = {
    Name        = "web-server"
    Instance    = 1             # number
    Enabled     = true          # bool
  }
  
  # ต้องแปลงเป็น string ก่อนใช้เป็น tags
  string_tags = {
    for k, v in local.mixed_tags : k => tostring(v)
  }
}
```

---

## Step 59: Encoding Functions

### jsonencode(), jsondecode(), yamlencode(), yamldecode()

```hcl
locals {
  # jsonencode() - convert HCL to JSON string
  policy_data = {
    Version = "2012-10-17"
    Statement = [
      {
        Effect   = "Allow"
        Action   = ["s3:GetObject", "s3:PutObject"]
        Resource = "arn:aws:s3:::my-bucket/*"
      }
    ]
  }
  
  policy_json = jsonencode(local.policy_data)
  # '{"Statement":[{"Action":["s3:GetObject","s3:PutObject"],"Effect":"Allow","Resource":"arn:aws:s3:::my-bucket/*"}],"Version":"2012-10-17"}'
  
  # jsondecode() - parse JSON string to HCL
  json_str    = "{\"name\":\"test\",\"port\":8080,\"enabled\":true}"
  parsed      = jsondecode(local.json_str)
  # { name = "test", port = 8080, enabled = true }
  
  name_from_json = local.parsed.name   # "test"
  port_from_json = local.parsed.port   # 8080
  
  # yamlencode() - convert HCL to YAML string
  k8s_labels = {
    app     = "myapp"
    version = "1.0.0"
    env     = "production"
  }
  
  labels_yaml = yamlencode(local.k8s_labels)
  # "app: myapp\nenv: production\nversion: 1.0.0\n"
  
  # yamldecode() - parse YAML string to HCL
  yaml_str = "name: Alice\nage: 30\ncity: Bangkok\n"
  person   = yamldecode(local.yaml_str)
  # { age = 30, city = "Bangkok", name = "Alice" }
}
```

### Practical Encoding Examples

```hcl
# ECS Task Definition - ต้องการ JSON string
resource "aws_ecs_task_definition" "app" {
  family = "app"
  
  container_definitions = jsonencode([
    {
      name  = "app"
      image = "${var.ecr_repository_url}:${var.image_tag}"
      
      portMappings = [
        {
          containerPort = var.app_port
          hostPort      = var.app_port
          protocol      = "tcp"
        }
      ]
      
      environment = [
        for k, v in var.env_vars : {
          name  = k
          value = v
        }
      ]
      
      secrets = [
        for k, v in var.secret_arns : {
          name      = k
          valueFrom = v
        }
      ]
      
      logConfiguration = {
        logDriver = "awslogs"
        options = {
          "awslogs-group"         = "/ecs/${var.app_name}"
          "awslogs-region"        = data.aws_region.current.name
          "awslogs-stream-prefix" = "ecs"
        }
      }
      
      healthCheck = {
        command     = ["CMD-SHELL", "curl -f http://localhost:${var.app_port}/health || exit 1"]
        interval    = 30
        timeout     = 5
        retries     = 3
        startPeriod = 60
      }
    }
  ])
}
```

---

## Step 60: Combined Functions Example

### Real-world: Infrastructure Configuration Generator

```hcl
# locals.tf สำหรับ complex infrastructure setup

locals {
  # === String Functions ===
  project_upper   = upper(var.project)
  env_prefix      = format("%s-%s", var.project, var.environment)
  sanitized_name  = replace(lower(var.project), "/[^a-z0-9]/", "-")
  
  # === Collection Functions ===
  all_azs = sort(var.availability_zones)
  az_count = length(local.all_azs)
  
  # ===  IP Functions ===
  public_subnets = [
    for i in range(local.az_count) :
    cidrsubnet(var.vpc_cidr, 8, i)
  ]
  
  private_subnets = [
    for i in range(local.az_count) :
    cidrsubnet(var.vpc_cidr, 8, i + 100)
  ]
  
  # เพิ่ม metadata ให้กับ subnets
  subnet_configs = flatten([
    [
      for i, subnet in local.public_subnets : {
        cidr = subnet
        az   = local.all_azs[i]
        type = "public"
        name = format("%s-public-%s", local.env_prefix, local.all_azs[i])
        index = i
      }
    ],
    [
      for i, subnet in local.private_subnets : {
        cidr = subnet
        az   = local.all_azs[i]
        type = "private"
        name = format("%s-private-%s", local.env_prefix, local.all_azs[i])
        index = i + local.az_count
      }
    ]
  ])
  
  # Convert to map for for_each
  subnet_map = {
    for s in local.subnet_configs :
    s.name => s
  }
  
  # === Numeric Functions ===
  total_subnets  = length(local.subnet_configs)
  instances_per_az = max(1, ceil(var.desired_capacity / local.az_count))
  total_capacity   = local.instances_per_az * local.az_count
  
  # === Hash Functions ===
  config_signature = md5(jsonencode({
    environment     = var.environment
    instance_type   = var.instance_type
    desired_capacity = var.desired_capacity
    vpc_cidr        = var.vpc_cidr
  }))
  
  # === Date Functions ===
  deployment_date = formatdate("YYYY-MM-DD", timestamp())
  
  # === Tags ===
  common_tags = merge(
    var.additional_tags,
    {
      Project          = var.project
      Environment      = var.environment
      ManagedBy        = "terraform"
      DeployedAt       = local.deployment_date
      ConfigSignature  = local.config_signature
    }
  )
}

# outputs.tf
output "infrastructure_summary" {
  value = {
    project         = local.project_upper
    environment     = var.environment
    vpc_cidr        = var.vpc_cidr
    az_count        = local.az_count
    total_subnets   = local.total_subnets
    public_subnets  = local.public_subnets
    private_subnets = local.private_subnets
    total_capacity  = local.total_capacity
    deployment_date = local.deployment_date
    config_version  = substr(local.config_signature, 0, 8)
  }
}
```

---

## Function Quick Reference

### String Functions Summary

| Function | Signature | Description |
|----------|-----------|-------------|
| `format` | `format(spec, args...)` | Printf-style formatting |
| `formatlist` | `formatlist(spec, list)` | Format each element |
| `join` | `join(sep, list)` | Join list with separator |
| `split` | `split(sep, str)` | Split string to list |
| `replace` | `replace(str, old, new)` | Replace substring |
| `regex` | `regex(pattern, str)` | First regex match |
| `regexall` | `regexall(pattern, str)` | All regex matches |
| `upper/lower` | `upper(str)` | Change case |
| `title` | `title(str)` | Title case |
| `trim/trimspace` | `trimspace(str)` | Remove whitespace |
| `trimprefix` | `trimprefix(str, prefix)` | Remove prefix |
| `trimsuffix` | `trimsuffix(str, suffix)` | Remove suffix |
| `substr` | `substr(str, offset, len)` | Substring |
| `length` | `length(str/list/map)` | Count items |
| `startswith` | `startswith(str, prefix)` | Prefix check |
| `endswith` | `endswith(str, suffix)` | Suffix check |
| `strcontains` | `strcontains(str, sub)` | Substring check |

### Collection Functions Summary

| Function | Description |
|----------|-------------|
| `concat` | Combine lists |
| `flatten` | Flatten nested lists |
| `merge` | Merge maps |
| `keys/values` | Map keys/values |
| `zipmap` | Create map from 2 lists |
| `sort/reverse` | Sort/reverse list |
| `distinct` | Remove duplicates |
| `chunklist` | Split into chunks |
| `toset/tolist/tomap` | Type conversion |
| `range` | Generate number sequence |
| `setintersection` | Set intersection |
| `setunion` | Set union |
| `setsubtract` | Set difference |

💡 **Pro Tips:**
- ใช้ `try()` กับทุก function ที่อาจ fail เช่น `try(tonumber(x), 0)`
- ใช้ `can()` เพื่อ validate ก่อน conversion
- ใช้ `cidrsubnets()` แทน loop เพื่อสร้าง multiple subnets
- ใช้ `templatefile()` แทน heredoc สำหรับ complex templates

⚠️ **Common Mistakes:**
- `tobool("yes")` จะ error - ใช้ `"true"` เท่านั้น
- `tonumber("1,000")` จะ error - ลบ comma ก่อน
- `file()` อ่าน relative path จาก cwd ไม่ใช่จาก module - ใช้ `${path.module}/`
- `timestamp()` return เวลา plan time ไม่ใช่ apply time - ใส่ใน resources จะ update ทุกครั้ง

---

*ก่อนหน้า: [Part 005 - HCL Expressions และ Operators](part-005.md)*
*ต่อไป: [Part 007 - HCL Conditional Expressions](part-007.md)*
