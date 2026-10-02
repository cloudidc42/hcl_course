# Part 008: HCL For Expressions
## For Expressions ใน HCL (Steps 71-80)

---

## Step 71: For Expression พื้นฐาน

### syntax: [for item in collection : expression]

```hcl
# ============ LIST OUTPUT ============

# Basic for expression กับ list
locals {
  names = ["alice", "bob", "charlie"]
  
  # แปลงทุก element
  upper_names = [for name in local.names : upper(name)]
  # ["ALICE", "BOB", "CHARLIE"]
  
  # String transformation
  greetings = [for name in local.names : "Hello, ${name}!"]
  # ["Hello, alice!", "Hello, bob!", "Hello, charlie!"]
  
  # Numeric transformation
  numbers = [1, 2, 3, 4, 5]
  doubled = [for n in local.numbers : n * 2]
  # [2, 4, 6, 8, 10]
  
  squares = [for n in local.numbers : n * n]
  # [1, 4, 9, 16, 25]
}

# ============ MAP OUTPUT ============

# for expression ที่ return map
locals {
  servers = ["web-01", "web-02", "api-01"]
  
  # สร้าง map จาก list
  server_names = {
    for server in local.servers :
    server => upper(server)
  }
  # {
  #   "web-01" = "WEB-01"
  #   "web-02" = "WEB-02"
  #   "api-01" = "API-01"
  # }
  
  # สร้าง map จาก list ของ objects
  employees = [
    { name = "Alice", age = 30, dept = "Engineering" },
    { name = "Bob",   age = 25, dept = "Marketing" },
    { name = "Carol", age = 35, dept = "Engineering" },
  ]
  
  # name เป็น key, dept เป็น value
  emp_departments = {
    for emp in local.employees :
    emp.name => emp.dept
  }
  # { Alice = "Engineering", Bob = "Marketing", Carol = "Engineering" }
}
```

### For Expression กับ Maps

```hcl
locals {
  config = {
    DB_HOST = "localhost"
    DB_PORT = "5432"
    DB_NAME = "myapp"
    API_KEY = "secret123"
  }
  
  # Transform map values
  config_upper = {
    for key, val in local.config : key => upper(val)
  }
  # { DB_HOST = "LOCALHOST", DB_PORT = "5432", ... }
  
  # Transform map keys
  lower_keys = {
    for key, val in local.config : lower(key) => val
  }
  # { db_host = "localhost", db_port = "5432", ... }
  
  # Transform both keys and values
  env_format = {
    for key, val in local.config :
    lower(key) => {
      name  = key
      value = val
    }
  }
  
  # Filter specific keys
  db_config = {
    for key, val in local.config :
    key => val
    if startswith(key, "DB_")
  }
  # { DB_HOST = "localhost", DB_PORT = "5432", DB_NAME = "myapp" }
}
```

---

## Step 72: For Expression กับ Condition (if clause)

### syntax: [for item in list : expr if condition]

```hcl
locals {
  numbers = [1, 2, 3, 4, 5, 6, 7, 8, 9, 10]
  
  # Filter: เฉพาะ even numbers
  even_nums = [for n in local.numbers : n if n % 2 == 0]
  # [2, 4, 6, 8, 10]
  
  # Filter: เฉพาะค่า > 5
  large_nums = [for n in local.numbers : n if n > 5]
  # [6, 7, 8, 9, 10]
  
  # Transform และ filter พร้อมกัน
  # double เฉพาะ odd numbers
  doubled_odds = [for n in local.numbers : n * 2 if n % 2 != 0]
  # [2, 6, 10, 14, 18]
}

# Filter objects
locals {
  servers = [
    { name = "web-01", active = true,  env = "prod" },
    { name = "web-02", active = false, env = "prod" },
    { name = "api-01", active = true,  env = "prod" },
    { name = "dev-01", active = true,  env = "dev"  },
  ]
  
  # Filter: เฉพาะ active servers
  active_servers = [
    for s in local.servers : s
    if s.active
  ]
  # [web-01, api-01, dev-01 objects]
  
  # Filter: active prod servers
  active_prod = [
    for s in local.servers : s.name
    if s.active && s.env == "prod"
  ]
  # ["web-01", "api-01"]
  
  # Filter map และ transform
  prod_configs = {
    for s in local.servers :
    s.name => s
    if s.env == "prod"
  }
}
```

### Real-world Filtering

```hcl
variable "subnets" {
  type = map(object({
    cidr    = string
    az      = string
    type    = string  # public, private, database
  }))
}

locals {
  # Filter subnets by type
  public_subnets = {
    for name, subnet in var.subnets :
    name => subnet
    if subnet.type == "public"
  }
  
  private_subnets = {
    for name, subnet in var.subnets :
    name => subnet
    if subnet.type == "private"
  }
  
  database_subnets = {
    for name, subnet in var.subnets :
    name => subnet
    if subnet.type == "database"
  }
  
  # Get CIDRs for security groups
  public_cidrs   = [for _, s in local.public_subnets : s.cidr]
  private_cidrs  = [for _, s in local.private_subnets : s.cidr]
}
```

---

## Step 73: For Expression กับ Map (key => value)

### สร้าง Map จาก For Expression

```hcl
# syntax: {for key_expr, val_expr in collection : new_key => new_value}
# หรือ: {for item in list : key_expr => val_expr}

locals {
  # สร้าง map จาก list
  fruits = ["apple", "banana", "cherry"]
  
  fruit_lengths = {
    for fruit in local.fruits :
    fruit => length(fruit)
  }
  # { apple = 5, banana = 6, cherry = 6 }
  
  fruit_upper = {
    for fruit in local.fruits :
    upper(fruit) => fruit
  }
  # { APPLE = "apple", BANANA = "banana", CHERRY = "cherry" }
  
  # สร้าง map จาก map (transform)
  prices = {
    apple  = 10
    banana = 5
    cherry = 20
  }
  
  discounted = {
    for item, price in local.prices :
    item => price * 0.9  # 10% discount
  }
  # { apple = 9, banana = 4.5, cherry = 18 }
  
  # invert map (swap keys and values)
  id_to_name = {
    "i-001" = "web-01"
    "i-002" = "web-02"
    "i-003" = "api-01"
  }
  
  name_to_id = {
    for id, name in local.id_to_name :
    name => id
  }
  # { web-01 = "i-001", web-02 = "i-002", api-01 = "i-003" }
}
```

### Map Grouping ด้วย ... (Grouping Mode)

```hcl
locals {
  employees = [
    { name = "Alice", dept = "Engineering" },
    { name = "Bob",   dept = "Marketing"   },
    { name = "Carol", dept = "Engineering" },
    { name = "Dave",  dept = "Marketing"   },
    { name = "Eve",   dept = "Engineering" },
  ]
  
  # Grouping mode: ใช้ ... เพื่อ group values ที่มี key เดียวกัน
  by_department = {
    for emp in local.employees :
    emp.dept => emp.name...
  }
  # {
  #   Engineering = ["Alice", "Carol", "Eve"]
  #   Marketing   = ["Bob", "Dave"]
  # }
}
```

---

## Step 74: For Expression กับ Index

### ใช้ Index ใน For Expression

```hcl
locals {
  items = ["a", "b", "c", "d"]
  
  # for กับ index (enumerate-like)
  with_index = [
    for i, v in local.items :
    "${i}: ${v}"
  ]
  # ["0: a", "1: b", "2: c", "3: d"]
  
  # สร้าง map ด้วย index
  indexed_map = {
    for i, v in local.items :
    i => v
  }
  # { 0 = "a", 1 = "b", 2 = "c", 3 = "d" }
  
  # สร้าง 1-based numbering
  numbered = [
    for i, v in local.items :
    "${i + 1}. ${v}"
  ]
  # ["1. a", "2. b", "3. c", "4. d"]
}
```

### Index ใน Map Iteration

```hcl
locals {
  config_map = {
    host = "localhost"
    port = "5432"
    name = "mydb"
  }
  
  # keys() ให้ index เป็น key ชื่อ
  # ถ้าต้องการ numeric index ต้องใช้ keys() + range()
  
  config_list = [
    for key in sort(keys(local.config_map)) :
    "${key}=${local.config_map[key]}"
  ]
  # ["host=localhost", "name=mydb", "port=5432"]
  
  # สร้าง numbered configuration
  config_numbered = {
    for i, key in sort(keys(local.config_map)) :
    "${i + 1}_${key}" => local.config_map[key]
  }
  # {
  #   "1_host" = "localhost"
  #   "2_name" = "mydb"
  #   "3_port" = "5432"
  # }
}
```

---

## Step 75: Nested For Expressions

### For ซ้อนกัน

```hcl
locals {
  # List ของ lists
  matrix = [[1, 2, 3], [4, 5, 6], [7, 8, 9]]
  
  # Flatten matrix
  flat = [
    for row in local.matrix :
    for item in row :
    item
  ]
  # [1, 2, 3, 4, 5, 6, 7, 8, 9]
  
  # สร้าง coordinate pairs
  rows = [1, 2, 3]
  cols = ["a", "b", "c"]
  
  coords = [
    for row in local.rows :
    for col in local.cols :
    "${row}${col}"
  ]
  # ["1a", "1b", "1c", "2a", "2b", "2c", "3a", "3b", "3c"]
}
```

### Nested For กับ Real Data

```hcl
locals {
  # environments และ services
  environments = ["dev", "staging", "prod"]
  services     = ["api", "worker", "frontend"]
  
  # สร้าง deployment matrix
  deployments = [
    for env in local.environments :
    for svc in local.services :
    {
      environment = env
      service     = svc
      name        = "${env}-${svc}"
    }
  ]
  # 9 deployment configs
  
  # สร้างเป็น map สำหรับ for_each
  deployment_map = {
    for d in local.deployments :
    d.name => d
  }
  
  # ตัวอย่างที่ซับซ้อนขึ้น: VPCs และ Subnets
  vpc_configs = {
    vpc1 = {
      cidr = "10.0.0.0/16"
      subnets = {
        public1  = "10.0.1.0/24"
        public2  = "10.0.2.0/24"
        private1 = "10.0.10.0/24"
      }
    }
    vpc2 = {
      cidr = "10.1.0.0/16"
      subnets = {
        public1  = "10.1.1.0/24"
        private1 = "10.1.10.0/24"
      }
    }
  }
  
  # Flatten: สร้าง list ของ subnet objects
  all_subnets = flatten([
    for vpc_name, vpc in local.vpc_configs : [
      for subnet_name, cidr in vpc.subnets : {
        vpc_name    = vpc_name
        subnet_name = subnet_name
        cidr        = cidr
        vpc_cidr    = vpc.cidr
        full_name   = "${vpc_name}-${subnet_name}"
      }
    ]
  ])
}
```

---

## Step 76: For Expressions กับ Functions

### ใช้ Functions ใน For

```hcl
locals {
  names = ["  Alice  ", "bob", "CHARLIE", " dave "]
  
  # Normalize: trim และ title case
  normalized = [
    for name in local.names :
    title(trimspace(name))
  ]
  # ["Alice", "Bob", "Charlie", "Dave"]
  
  # Format และ filter
  ports = [80, 443, 8080, 9090, -1, 0]
  
  valid_ports = [
    for port in local.ports :
    port
    if port > 0 && port <= 65535
  ]
  # [80, 443, 8080, 9090]
  
  port_strings = [
    for port in local.valid_ports :
    format(":%d", port)
  ]
  # [":80", ":443", ":8080", ":9090"]
  
  # Complex transformation
  servers = [
    { hostname = "web-01.prod.internal", port = 80 },
    { hostname = "api-01.prod.internal", port = 8080 },
    { hostname = "db-01.prod.internal",  port = 5432 },
  ]
  
  # Extract domain từ hostname
  service_domains = {
    for s in local.servers :
    split(".", s.hostname)[0] => {
      host = s.hostname
      port = s.port
      url  = format("http://%s:%d", s.hostname, s.port)
    }
  }
}
```

### For กับ String Functions

```hcl
locals {
  resource_names = [
    "aws_instance_web_server",
    "aws_s3_bucket_data",
    "aws_rds_cluster_postgres",
    "aws_lambda_function_processor",
  ]
  
  # Extract resource type (ลบ "aws_" prefix)
  resource_types = [
    for name in local.resource_names :
    trimprefix(name, "aws_")
  ]
  # ["instance_web_server", "s3_bucket_data", ...]
  
  # สร้าง human-readable names
  readable_names = {
    for name in local.resource_names :
    name => join(" ", [
      for word in split("_", trimprefix(name, "aws_")) :
      title(word)
    ])
  }
  # {
  #   aws_instance_web_server = "Instance Web Server"
  #   aws_s3_bucket_data = "S3 Bucket Data"
  #   ...
  # }
}
```

---

## Step 77: Converting Between Types ด้วย For

### List ↔ Map Conversion

```hcl
locals {
  # List of objects → Map (by key field)
  users = [
    { id = "u001", name = "Alice", email = "alice@example.com" },
    { id = "u002", name = "Bob",   email = "bob@example.com"   },
    { id = "u003", name = "Carol", email = "carol@example.com" },
  ]
  
  # Index by id
  users_by_id = {
    for user in local.users :
    user.id => user
  }
  # {
  #   u001 = { id = "u001", name = "Alice", ... }
  #   u002 = { id = "u002", name = "Bob", ... }
  # }
  
  # Index by name (หลาย fields)
  users_by_name = {
    for user in local.users :
    user.name => user.email
  }
  # { Alice = "alice@example.com", Bob = "bob@example.com", ... }
  
  # Map → List of objects
  config = {
    HOST    = "localhost"
    PORT    = "5432"
    DB_NAME = "myapp"
  }
  
  config_as_list = [
    for key, val in local.config : {
      name  = key
      value = val
    }
  ]
  # [{ name = "DB_NAME", value = "myapp" }, ...]
  
  # Suitable for ECS environment variables format
  ecs_env_vars = [
    for key, val in local.config : {
      name  = key
      value = val
    }
  ]
}
```

### Type Restructuring

```hcl
# เปลี่ยน structure ของข้อมูล
locals {
  # Original: list of {name, env, value}
  parameters = [
    { name = "api_url",    env = "dev",  value = "http://localhost:8080" },
    { name = "api_url",    env = "prod", value = "https://api.example.com" },
    { name = "db_host",    env = "dev",  value = "localhost" },
    { name = "db_host",    env = "prod", value = "db.example.com" },
  ]
  
  # Restructure to: { env → { name → value } }
  # ไม่สามารถทำได้โดยตรงด้วย for ครั้งเดียว
  # ต้องใช้ 2 รอบ
  
  # รอบแรก: group by env
  by_env_raw = {
    for param in local.parameters :
    param.env => {
      name  = param.name
      value = param.value
    }...  # grouping mode
  }
  
  # Note: by_env_raw จะเป็น { env = [list of objects] }
  # ต้องแปลงอีกรอบ
  
  # Alternative: สร้าง flat map ด้วย composite key
  flat_params = {
    for param in local.parameters :
    "${param.env}/${param.name}" => param.value
  }
  # {
  #   "dev/api_url"  = "http://localhost:8080"
  #   "prod/api_url" = "https://api.example.com"
  #   "dev/db_host"  = "localhost"
  #   "prod/db_host" = "db.example.com"
  # }
}
```

---

## Step 78: Grouping ด้วย For Expressions

### Grouping Mode (... syntax)

```hcl
locals {
  # Grouping by department
  employees = [
    { name = "Alice",   dept = "Engineering", level = "senior"  },
    { name = "Bob",     dept = "Marketing",   level = "junior"  },
    { name = "Carol",   dept = "Engineering", level = "senior"  },
    { name = "Dave",    dept = "Marketing",   level = "senior"  },
    { name = "Eve",     dept = "Engineering", level = "mid"     },
    { name = "Frank",   dept = "Sales",       level = "junior"  },
  ]
  
  # Group by department (names only)
  by_dept_names = {
    for emp in local.employees :
    emp.dept => emp.name...
  }
  # {
  #   Engineering = ["Alice", "Carol", "Eve"]
  #   Marketing   = ["Bob", "Dave"]
  #   Sales       = ["Frank"]
  # }
  
  # Group by department (full objects)
  by_dept = {
    for emp in local.employees :
    emp.dept => emp...
  }
  
  # Count per department
  dept_counts = {
    for dept, members in local.by_dept_names :
    dept => length(members)
  }
  # { Engineering = 3, Marketing = 2, Sales = 1 }
  
  # Group by level
  by_level = {
    for emp in local.employees :
    emp.level => emp.name...
  }
  # { senior = ["Alice", "Carol", "Dave"], mid = ["Eve"], junior = ["Bob", "Frank"] }
}
```

### Practical Grouping

```hcl
variable "security_rules" {
  type = list(object({
    direction   = string  # ingress, egress
    port        = number
    protocol    = string
    description = string
    cidr        = string
  }))
  
  default = [
    { direction = "ingress", port = 80,   protocol = "tcp", description = "HTTP",    cidr = "0.0.0.0/0"   },
    { direction = "ingress", port = 443,  protocol = "tcp", description = "HTTPS",   cidr = "0.0.0.0/0"   },
    { direction = "ingress", port = 22,   protocol = "tcp", description = "SSH",     cidr = "10.0.0.0/8"  },
    { direction = "egress",  port = 443,  protocol = "tcp", description = "HTTPS",   cidr = "0.0.0.0/0"   },
    { direction = "egress",  port = 5432, protocol = "tcp", description = "Postgres",cidr = "10.0.0.0/8"  },
  ]
}

locals {
  # Group by direction
  grouped_rules = {
    for rule in var.security_rules :
    rule.direction => rule...
  }
  
  ingress_rules = lookup(local.grouped_rules, "ingress", [])
  egress_rules  = lookup(local.grouped_rules, "egress", [])
}
```

---

## Step 79: Object Construction กับ For

### สร้าง Objects ซับซ้อน

```hcl
locals {
  # สร้าง ECS container definitions จาก map
  services = {
    api = {
      image   = "api:1.0.0"
      port    = 8080
      cpu     = 256
      memory  = 512
    }
    worker = {
      image   = "worker:1.0.0"
      port    = 9090
      cpu     = 512
      memory  = 1024
    }
  }
  
  # แปลงเป็น ECS format
  container_definitions = [
    for name, config in local.services : {
      name    = name
      image   = config.image
      cpu     = config.cpu
      memory  = config.memory
      
      portMappings = [
        {
          containerPort = config.port
          hostPort      = config.port
          protocol      = "tcp"
        }
      ]
      
      environment = [
        {
          name  = "SERVICE_NAME"
          value = name
        },
        {
          name  = "SERVICE_PORT"
          value = tostring(config.port)
        }
      ]
      
      logConfiguration = {
        logDriver = "awslogs"
        options = {
          "awslogs-group"         = "/ecs/${var.project}/${name}"
          "awslogs-region"        = var.region
          "awslogs-stream-prefix" = "ecs"
        }
      }
    }
  ]
}
```

### สร้าง IAM Policy Statements

```hcl
variable "s3_bucket_permissions" {
  type = map(list(string))
  default = {
    read_bucket  = ["GetObject", "ListBucket"]
    write_bucket = ["PutObject", "DeleteObject"]
    manage_bucket = ["CreateBucket", "DeleteBucket", "PutBucketPolicy"]
  }
}

variable "s3_buckets" {
  type    = list(string)
  default = ["data-bucket", "logs-bucket", "artifacts-bucket"]
}

locals {
  # สร้าง policy statements สำหรับแต่ละ bucket
  policy_statements = flatten([
    for bucket in var.s3_buckets : [
      {
        Sid    = "ReadAccess${replace(title(replace(bucket, "-", " ")), " ", "")}"
        Effect = "Allow"
        Action = [for action in var.s3_bucket_permissions.read_bucket : "s3:${action}"]
        Resource = [
          "arn:aws:s3:::${bucket}",
          "arn:aws:s3:::${bucket}/*"
        ]
      }
    ]
  ])
  
  policy_document = jsonencode({
    Version   = "2012-10-17"
    Statement = local.policy_statements
  })
}
```

---

## Step 80: Real-world For Expression Patterns

### Pattern 1: Flatten Nested Lists

```hcl
variable "region_configs" {
  type = map(object({
    vpc_cidr = string
    azs      = list(string)
    subnets  = list(object({
      name = string
      cidr = string
      type = string
    }))
  }))
  
  default = {
    "ap-southeast-1" = {
      vpc_cidr = "10.0.0.0/16"
      azs      = ["ap-southeast-1a", "ap-southeast-1b", "ap-southeast-1c"]
      subnets = [
        { name = "public-1",  cidr = "10.0.1.0/24",  type = "public"  },
        { name = "private-1", cidr = "10.0.10.0/24", type = "private" },
      ]
    }
    "us-east-1" = {
      vpc_cidr = "10.1.0.0/16"
      azs      = ["us-east-1a", "us-east-1b"]
      subnets = [
        { name = "public-1",  cidr = "10.1.1.0/24",  type = "public"  },
        { name = "private-1", cidr = "10.1.10.0/24", type = "private" },
      ]
    }
  }
}

locals {
  # Flatten regions + subnets เป็น flat map
  all_subnets = {
    for item in flatten([
      for region, config in var.region_configs : [
        for subnet in config.subnets : {
          key    = "${region}-${subnet.name}"
          region = region
          name   = subnet.name
          cidr   = subnet.cidr
          type   = subnet.type
          vpc_cidr = config.vpc_cidr
        }
      ]
    ]) : item.key => item
  }
}
```

### Pattern 2: Inverting Maps

```hcl
locals {
  # Original: username → list of roles
  user_roles = {
    alice   = ["admin", "developer", "viewer"]
    bob     = ["developer", "viewer"]
    charlie = ["admin", "viewer"]
  }
  
  # Invert: role → list of users
  role_users = {
    for role, users in {
      for user, roles in local.user_roles :
      "" => "" ...  # placeholder
    } : role => users
  }
  
  # Better way using flatten + grouping
  flat_assignments = flatten([
    for user, roles in local.user_roles : [
      for role in roles : {
        user = user
        role = role
      }
    ]
  ])
  
  role_to_users = {
    for assignment in local.flat_assignments :
    assignment.role => assignment.user...
  }
  # {
  #   admin     = ["alice", "charlie"]
  #   developer = ["alice", "bob"]
  #   viewer    = ["alice", "bob", "charlie"]
  # }
}
```

### Pattern 3: Filtering และ Transforming

```hcl
variable "tags" {
  type = map(string)
  default = {
    Name        = "web-server"
    Environment = "production"
    Team        = "platform"
    CostCenter  = ""         # empty - should be removed
    Description = null       # null - should be removed
  }
}

locals {
  # Remove null และ empty string values จาก tags
  valid_tags = {
    for key, val in var.tags :
    key => val
    if val != null && val != ""
  }
  
  # Tag keys ต้องไม่เกิน 128 chars, values ไม่เกิน 256 chars
  truncated_tags = {
    for key, val in local.valid_tags :
    substr(key, 0, min(128, length(key))) => substr(val, 0, min(256, length(val)))
  }
}
```

### Pattern 4: Complex Data Transformation

```hcl
# ตัวอย่าง: แปลงข้อมูลจาก JSON ที่ดึงมาจาก data source
data "aws_ec2_instance_types" "available" {
  filter {
    name   = "vcpu-info.default-vcpus"
    values = ["2", "4"]
  }
  filter {
    name   = "memory-info.size-in-mib"
    values = ["4096", "8192"]
  }
}

locals {
  # แปลง instance types เป็น lookup map
  available_types = toset(data.aws_ec2_instance_types.available.instance_types)
  
  # Filter: เฉพาะ t3 family
  t3_types = [
    for type in local.available_types :
    type
    if startswith(type, "t3.")
  ]
  
  # Group by prefix
  type_families = {
    for type in local.available_types :
    split(".", type)[0] => type...
  }
}
```

### Complete Example: AWS Multi-Region Deployment

```hcl
# ตัวอย่างสมบูรณ์: for expressions ใน multi-region setup

variable "regions" {
  type = map(object({
    primary        = bool
    instance_count = number
    instance_type  = string
    az_count       = number
  }))
  
  default = {
    "ap-southeast-1" = {
      primary        = true
      instance_count = 3
      instance_type  = "t3.large"
      az_count       = 3
    }
    "us-east-1" = {
      primary        = false
      instance_count = 2
      instance_type  = "t3.medium"
      az_count       = 2
    }
  }
}

locals {
  # Primary region
  primary_region = one([
    for region, config in var.regions :
    region
    if config.primary
  ])
  
  # All regions sorted
  all_regions = sort(keys(var.regions))
  
  # Total instance count
  total_instances = sum([
    for region, config in var.regions :
    config.instance_count
  ])
  
  # Generate subnet info per region
  region_subnets = {
    for region, config in var.regions : region => {
      public_subnets = [
        for i in range(config.az_count) :
        {
          index = i
          cidr  = cidrsubnet("10.${index(local.all_regions, region)}.0.0/16", 8, i)
        }
      ]
      
      private_subnets = [
        for i in range(config.az_count) :
        {
          index = i
          cidr  = cidrsubnet("10.${index(local.all_regions, region)}.0.0/16", 8, i + 100)
        }
      ]
    }
  }
  
  # Flat list of all subnets across all regions
  all_subnet_cidrs = flatten([
    for region, subnets in local.region_subnets : [
      [for s in subnets.public_subnets : s.cidr],
      [for s in subnets.private_subnets : s.cidr]
    ]
  ])
}

output "deployment_summary" {
  value = {
    primary_region   = local.primary_region
    total_regions    = length(var.regions)
    total_instances  = local.total_instances
    region_breakdown = {
      for region, config in var.regions :
      region => {
        is_primary       = config.primary
        instance_count   = config.instance_count
        instance_type    = config.instance_type
        subnet_count     = config.az_count * 2  # public + private per AZ
      }
    }
  }
}
```

---

## สรุป For Expressions

### Quick Reference

| Expression | Output | Example |
|------------|--------|---------|
| `[for x in list : f(x)]` | List | `[for n in nums : n * 2]` |
| `[for x in list : f(x) if cond]` | Filtered List | `[for n in nums : n if n > 0]` |
| `{for x in list : key => val}` | Map | `{for u in users : u.id => u.name}` |
| `{for k, v in map : k => f(v)}` | Transformed Map | `{for k, v in m : k => upper(v)}` |
| `{for k, v in map : k => v if cond}` | Filtered Map | `{for k, v in m : k => v if v != null}` |
| `[for i, v in list : ...]` | With Index | `[for i, v in l : "${i}: ${v}"]` |
| `{for x in list : x.key => x.val...}` | Grouped Map | `{for e in emps : e.dept => e.name...}` |

💡 **Pro Tips:**
- ใช้ `flatten()` กับ nested for เพื่อสร้าง flat list
- ใช้ grouping mode (`...`) เพื่อ aggregate values
- Filter ก่อนหรือหลัง transform ตามที่เหมาะสม
- ใช้ `for_each` กับ map ที่สร้างจาก for expression

⚠️ **Common Mistakes:**
- Map keys ต้องไม่ซ้ำกัน (ถ้าไม่ใช้ grouping mode)
- `for` expression ไม่สามารถมี side effects
- ระวัง type consistency ใน output list
- Nested for อาจทำให้ performance ช้าถ้าข้อมูลใหญ่มาก

---

*ก่อนหน้า: [Part 007 - HCL Conditional Expressions](part-007.md)*
*ต่อไป: [Part 009 - HCL Dynamic Blocks](part-009.md)*
