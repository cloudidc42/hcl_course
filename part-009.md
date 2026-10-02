# Part 009: HCL Dynamic Blocks
## Dynamic Blocks ใน HCL (Steps 81-90)

---

## Step 81: Dynamic Blocks คืออะไรและทำไมต้องใช้?

### ปัญหาที่ Dynamic Blocks แก้ไข

```hcl
# ❌ BEFORE: ต้อง repeat blocks manually
resource "aws_security_group" "web" {
  name   = "web-sg"
  vpc_id = aws_vpc.main.id
  
  # ถ้ามี 10 rules ต้องเขียน 10 ingress blocks!
  ingress {
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
    description = "HTTP"
  }
  
  ingress {
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
    description = "HTTPS"
  }
  
  ingress {
    from_port   = 8080
    to_port     = 8080
    protocol    = "tcp"
    cidr_blocks = ["10.0.0.0/8"]
    description = "App port"
  }
  
  # ... 7 more rules ...
}

# ✅ AFTER: ใช้ dynamic block
variable "ingress_rules" {
  type = list(object({
    port        = number
    description = string
    cidr        = string
  }))
  
  default = [
    { port = 80,   description = "HTTP",     cidr = "0.0.0.0/0"  },
    { port = 443,  description = "HTTPS",    cidr = "0.0.0.0/0"  },
    { port = 8080, description = "App port", cidr = "10.0.0.0/8" },
  ]
}

resource "aws_security_group" "web" {
  name   = "web-sg"
  vpc_id = aws_vpc.main.id
  
  dynamic "ingress" {
    for_each = var.ingress_rules
    content {
      from_port   = ingress.value.port
      to_port     = ingress.value.port
      protocol    = "tcp"
      cidr_blocks = [ingress.value.cidr]
      description = ingress.value.description
    }
  }
}
```

### ประโยชน์ของ Dynamic Blocks

```
Dynamic Block Benefits:
├── DRY (Don't Repeat Yourself) - ไม่ต้อง copy-paste
├── Configurable via variables - เปลี่ยนได้จาก tfvars
├── Conditional blocks - สร้างหรือข้ามตาม condition
├── Reusable modules - module ที่ flexible มากขึ้น
└── Maintainable - เปลี่ยนในที่เดียวได้ทั้งหมด
```

---

## Step 82: Basic Dynamic Block Syntax

### โครงสร้าง Dynamic Block

```
ASCII Diagram - Dynamic Block Structure:
┌────────────────────────────────────────────────┐
│  dynamic "block_type" {                        │
│  │        │                                    │
│  │        └── Block type ที่จะ iterate         │
│  │                                             │
│  │  for_each = collection                      │
│  │  │          └── List, set, or map           │
│  │  │                                          │
│  │  iterator = alias  (optional)              │
│  │  │          └── Custom name for each item  │
│  │  │                                         │
│  │  content {                                  │
│  │  │  arg = block_type.value.attr            │
│  │  │  └── Access via block_type.key/.value   │
│  │  }                                          │
│  }                                             │
└────────────────────────────────────────────────┘
```

### ตัวอย่าง Basic Dynamic Block

```hcl
# ตัวอย่าง 1: Dynamic block กับ list
locals {
  dns_servers = ["8.8.8.8", "8.8.4.4", "1.1.1.1"]
}

resource "aws_vpc_dhcp_options" "main" {
  # Static argument
  domain_name = "example.internal"
  
  # Dynamic ไม่จำเป็นสำหรับ list argument ง่ายๆ
  # แต่ถ้าเป็น nested block ต้องใช้ dynamic
  domain_name_servers = local.dns_servers
}

# ตัวอย่าง 2: Dynamic block ที่จำเป็น
resource "aws_security_group" "app" {
  name   = "app-sg"
  vpc_id = aws_vpc.main.id
  
  dynamic "ingress" {
    for_each = [80, 443, 8080]  # iterate over simple list
    
    content {
      # ingress.value = current port number
      from_port   = ingress.value
      to_port     = ingress.value
      protocol    = "tcp"
      cidr_blocks = ["0.0.0.0/0"]
    }
  }
  
  # Single egress (no dynamic needed)
  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}
```

---

## Step 83: Dynamic Blocks กับ for_each

### for_each กับ List

```hcl
variable "allowed_ports" {
  description = "Ports ที่อนุญาตเข้า"
  type        = list(number)
  default     = [22, 80, 443, 8080]
}

resource "aws_security_group" "flexible" {
  name   = "flexible-sg"
  vpc_id = aws_vpc.main.id
  
  dynamic "ingress" {
    for_each = var.allowed_ports
    
    content {
      # สำหรับ list: ingress.key = index, ingress.value = port
      description = "Allow port ${ingress.value}"
      from_port   = ingress.value
      to_port     = ingress.value
      protocol    = "tcp"
      cidr_blocks = ["0.0.0.0/0"]
    }
  }
}
```

### for_each กับ List of Objects

```hcl
variable "ingress_rules" {
  description = "Ingress rules definition"
  type = list(object({
    description = string
    from_port   = number
    to_port     = number
    protocol    = string
    cidr_blocks = list(string)
  }))
  
  default = [
    {
      description = "SSH from VPN"
      from_port   = 22
      to_port     = 22
      protocol    = "tcp"
      cidr_blocks = ["10.0.0.0/8"]
    },
    {
      description = "HTTP from internet"
      from_port   = 80
      to_port     = 80
      protocol    = "tcp"
      cidr_blocks = ["0.0.0.0/0"]
    },
    {
      description = "HTTPS from internet"
      from_port   = 443
      to_port     = 443
      protocol    = "tcp"
      cidr_blocks = ["0.0.0.0/0"]
    },
    {
      description = "Custom port range"
      from_port   = 8000
      to_port     = 8999
      protocol    = "tcp"
      cidr_blocks = ["192.168.1.0/24"]
    },
  ]
}

resource "aws_security_group" "web" {
  name        = "web-security-group"
  description = "Web server security group"
  vpc_id      = aws_vpc.main.id
  
  dynamic "ingress" {
    for_each = var.ingress_rules
    
    content {
      # ingress.value มี type = object
      description = ingress.value.description
      from_port   = ingress.value.from_port
      to_port     = ingress.value.to_port
      protocol    = ingress.value.protocol
      cidr_blocks = ingress.value.cidr_blocks
    }
  }
  
  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
    description = "Allow all outbound traffic"
  }
  
  tags = {
    Name = "web-sg"
  }
}
```

---

## Step 84: Dynamic Blocks กับ Maps

### for_each กับ Map

```hcl
variable "ingress_map" {
  description = "Named ingress rules"
  type = map(object({
    from_port   = number
    to_port     = number
    protocol    = string
    cidr_blocks = list(string)
    description = string
  }))
  
  default = {
    ssh = {
      from_port   = 22
      to_port     = 22
      protocol    = "tcp"
      cidr_blocks = ["10.0.0.0/8"]
      description = "SSH access"
    }
    http = {
      from_port   = 80
      to_port     = 80
      protocol    = "tcp"
      cidr_blocks = ["0.0.0.0/0"]
      description = "HTTP access"
    }
    https = {
      from_port   = 443
      to_port     = 443
      protocol    = "tcp"
      cidr_blocks = ["0.0.0.0/0"]
      description = "HTTPS access"
    }
  }
}

resource "aws_security_group" "named_rules" {
  name   = "named-rules-sg"
  vpc_id = aws_vpc.main.id
  
  dynamic "ingress" {
    for_each = var.ingress_map
    
    content {
      # ingress.key = rule name ("ssh", "http", "https")
      # ingress.value = rule object
      description = ingress.value.description
      from_port   = ingress.value.from_port
      to_port     = ingress.value.to_port
      protocol    = ingress.value.protocol
      cidr_blocks = ingress.value.cidr_blocks
    }
  }
}
```

### Dynamic กับ Conditional Map

```hcl
variable "enable_ssh" {
  type    = bool
  default = false
}

variable "ssh_allowed_cidrs" {
  type    = list(string)
  default = []
}

locals {
  # สร้าง map ที่มี rules ตาม conditions
  security_rules = merge(
    # Always: HTTP and HTTPS
    {
      http = {
        from_port   = 80
        to_port     = 80
        protocol    = "tcp"
        cidr_blocks = ["0.0.0.0/0"]
      }
      https = {
        from_port   = 443
        to_port     = 443
        protocol    = "tcp"
        cidr_blocks = ["0.0.0.0/0"]
      }
    },
    
    # Conditional: SSH only if enabled
    var.enable_ssh ? {
      ssh = {
        from_port   = 22
        to_port     = 22
        protocol    = "tcp"
        cidr_blocks = length(var.ssh_allowed_cidrs) > 0 ? var.ssh_allowed_cidrs : ["10.0.0.0/8"]
      }
    } : {}
  )
}

resource "aws_security_group" "conditional" {
  name   = "conditional-sg"
  vpc_id = aws_vpc.main.id
  
  dynamic "ingress" {
    for_each = local.security_rules
    
    content {
      description = "Allow ${ingress.key}"
      from_port   = ingress.value.from_port
      to_port     = ingress.value.to_port
      protocol    = ingress.value.protocol
      cidr_blocks = ingress.value.cidr_blocks
    }
  }
}
```

---

## Step 85: Nested Dynamic Blocks

### Dynamic Block ภายใน Dynamic Block

```hcl
variable "policies" {
  description = "IAM policies with statements"
  type = list(object({
    policy_name = string
    statements  = list(object({
      effect    = string
      actions   = list(string)
      resources = list(string)
      conditions = optional(list(object({
        test     = string
        variable = string
        values   = list(string)
      })), [])
    }))
  }))
}

resource "aws_iam_policy" "policies" {
  for_each = {
    for policy in var.policies :
    policy.policy_name => policy
  }
  
  name = each.key
  
  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      for stmt in each.value.statements : merge(
        {
          Effect   = stmt.effect
          Action   = stmt.actions
          Resource = stmt.resources
        },
        length(stmt.conditions) > 0 ? {
          Condition = {
            for cond in stmt.conditions :
            cond.test => {
              (cond.variable) = cond.values
            }
          }
        } : {}
      )
    ]
  })
}

# ตัวอย่างที่ 2: Nested dynamic blocks ใน resource
variable "waf_rules" {
  type = list(object({
    name     = string
    priority = number
    action   = string  # "allow" or "block"
    
    # Each rule can have multiple conditions
    conditions = list(object({
      type     = string  # "ip_set", "regex", "size"
      negated  = bool
      field    = optional(string)
      values   = optional(list(string))
      operator = optional(string)
      size     = optional(number)
    }))
  }))
}

resource "aws_wafv2_rule_group" "custom" {
  name     = "${var.project}-rules"
  scope    = "REGIONAL"
  capacity = 100
  
  # Dynamic rules
  dynamic "rule" {
    for_each = var.waf_rules
    
    content {
      name     = rule.value.name
      priority = rule.value.priority
      
      action {
        # Nested: dynamic action block
        dynamic "allow" {
          for_each = rule.value.action == "allow" ? [1] : []
          content {}
        }
        
        dynamic "block" {
          for_each = rule.value.action == "block" ? [1] : []
          content {}
        }
      }
      
      statement {
        # Nested dynamic for multiple conditions
        dynamic "ip_set_reference_statement" {
          for_each = [
            for cond in rule.value.conditions :
            cond
            if cond.type == "ip_set"
          ]
          
          content {
            arn = ip_set_reference_statement.value.values[0]
          }
        }
      }
      
      visibility_config {
        cloudwatch_metrics_enabled = true
        metric_name                = rule.value.name
        sampled_requests_enabled   = true
      }
    }
  }
  
  visibility_config {
    cloudwatch_metrics_enabled = true
    metric_name                = "${var.project}-rules"
    sampled_requests_enabled   = true
  }
}
```

---

## Step 86: Dynamic Blocks ใน Resources (AWS Examples)

### AWS Security Group - Dynamic Rules

```hcl
# locals.tf
locals {
  # Define all rules
  all_ingress_rules = {
    https_public = {
      description = "HTTPS from internet"
      from_port   = 443
      to_port     = 443
      protocol    = "tcp"
      cidr_blocks = ["0.0.0.0/0"]
    }
    http_redirect = {
      description = "HTTP for redirect"
      from_port   = 80
      to_port     = 80
      protocol    = "tcp"
      cidr_blocks = ["0.0.0.0/0"]
    }
    app_internal = {
      description = "App port internal"
      from_port   = 8080
      to_port     = 8080
      protocol    = "tcp"
      cidr_blocks = ["10.0.0.0/8"]
    }
    monitoring = {
      description = "Prometheus metrics"
      from_port   = 9090
      to_port     = 9090
      protocol    = "tcp"
      cidr_blocks = ["10.0.0.0/8"]
    }
  }
  
  # Filter by environment
  ingress_rules = var.environment == "production" ? (
    local.all_ingress_rules
  ) : (
    # Dev: add extra ports
    merge(local.all_ingress_rules, {
      debug_port = {
        description = "Debug port (non-prod only)"
        from_port   = 9229
        to_port     = 9229
        protocol    = "tcp"
        cidr_blocks = ["10.0.0.0/8"]
      }
    })
  )
}

resource "aws_security_group" "app" {
  name        = "${var.project}-${var.environment}-app-sg"
  description = "Application security group"
  vpc_id      = aws_vpc.main.id
  
  dynamic "ingress" {
    for_each = local.ingress_rules
    
    content {
      description = ingress.value.description
      from_port   = ingress.value.from_port
      to_port     = ingress.value.to_port
      protocol    = ingress.value.protocol
      cidr_blocks = ingress.value.cidr_blocks
    }
  }
  
  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
    description = "Allow all outbound"
  }
  
  tags = merge(local.common_tags, {
    Name = "${var.project}-${var.environment}-app-sg"
  })
}
```

### AWS ALB Listener Rules - Dynamic

```hcl
variable "routing_rules" {
  description = "ALB routing rules"
  type = list(object({
    priority        = number
    path_pattern    = string
    target_group_arn = string
  }))
  
  default = [
    {
      priority         = 10
      path_pattern     = "/api/*"
      target_group_arn = "arn:aws:elasticloadbalancing:..."
    },
    {
      priority         = 20
      path_pattern     = "/admin/*"
      target_group_arn = "arn:aws:elasticloadbalancing:..."
    },
    {
      priority         = 100
      path_pattern     = "/*"
      target_group_arn = "arn:aws:elasticloadbalancing:..."
    },
  ]
}

resource "aws_lb_listener" "https" {
  load_balancer_arn = aws_lb.main.arn
  port              = 443
  protocol          = "HTTPS"
  ssl_policy        = "ELBSecurityPolicy-2016-08"
  certificate_arn   = var.ssl_certificate_arn
  
  # Default action
  default_action {
    type             = "forward"
    target_group_arn = var.default_target_group_arn
  }
}

resource "aws_lb_listener_rule" "routing" {
  for_each = {
    for rule in var.routing_rules :
    tostring(rule.priority) => rule
  }
  
  listener_arn = aws_lb_listener.https.arn
  priority     = each.value.priority
  
  action {
    type             = "forward"
    target_group_arn = each.value.target_group_arn
  }
  
  condition {
    path_pattern {
      values = [each.value.path_pattern]
    }
  }
}
```

---

## Step 87: Dynamic Blocks vs count/for_each

### เมื่อไหรใช้อะไร

```
Decision Guide:
┌───────────────────────────────────────────────────────────┐
│                                                           │
│  ต้องการสร้าง nested blocks หลายๆ อัน?                   │
│  └─ ใช้ dynamic block                                     │
│                                                           │
│  ต้องการสร้าง top-level resources หลายๆ อัน?             │
│  └─ ใช้ count หรือ for_each                              │
│                                                           │
│  Nested block: block ที่อยู่ภายใน resource block         │
│  Example: ingress {} ใน aws_security_group               │
│                                                           │
│  Top-level resource: resource block เอง                  │
│  Example: aws_security_group สร้างหลายอัน               │
│                                                           │
└───────────────────────────────────────────────────────────┘
```

### ตัวอย่างเปรียบเทียบ

```hcl
# ============ count ============
# ใช้สำหรับสร้าง resources หลายอัน

resource "aws_instance" "web" {
  count = 3  # สร้าง 3 instances
  
  ami           = var.ami_id
  instance_type = "t3.micro"
  
  tags = {
    Name = "web-${count.index + 1}"
  }
}

# ============ for_each ============
# ใช้สำหรับสร้าง resources หลายอัน จาก map/set

resource "aws_s3_bucket" "buckets" {
  for_each = toset(["data", "logs", "backups"])
  
  bucket = "${var.project}-${each.key}"
}

# ============ dynamic ============
# ใช้สำหรับสร้าง nested blocks หลายอัน ภายใน resource เดียว

resource "aws_security_group" "web" {
  name   = "web-sg"
  vpc_id = aws_vpc.main.id
  
  # dynamic สร้าง ingress blocks หลายๆ อัน
  dynamic "ingress" {
    for_each = var.allowed_ports
    content {
      from_port   = ingress.value
      to_port     = ingress.value
      protocol    = "tcp"
      cidr_blocks = ["0.0.0.0/0"]
    }
  }
}
```

### Hybrid: ใช้ทั้ง count/for_each และ dynamic

```hcl
# สร้างหลาย security groups แต่ละ group มีหลาย rules

variable "service_security_groups" {
  type = map(object({
    description   = string
    ingress_rules = list(object({
      port     = number
      protocol = string
      cidr     = string
    }))
  }))
  
  default = {
    web = {
      description = "Web servers"
      ingress_rules = [
        { port = 80,  protocol = "tcp", cidr = "0.0.0.0/0" },
        { port = 443, protocol = "tcp", cidr = "0.0.0.0/0" },
      ]
    }
    app = {
      description = "App servers"
      ingress_rules = [
        { port = 8080, protocol = "tcp", cidr = "10.0.0.0/8" },
        { port = 9090, protocol = "tcp", cidr = "10.0.0.0/8" },
      ]
    }
    db = {
      description = "Database servers"
      ingress_rules = [
        { port = 5432, protocol = "tcp", cidr = "10.0.0.0/8" },
      ]
    }
  }
}

# for_each สร้างหลาย security groups
resource "aws_security_group" "services" {
  for_each = var.service_security_groups
  
  name        = "${var.project}-${each.key}-sg"
  description = each.value.description
  vpc_id      = aws_vpc.main.id
  
  # dynamic สร้างหลาย ingress rules ภายใน แต่ละ sg
  dynamic "ingress" {
    for_each = each.value.ingress_rules
    
    content {
      from_port   = ingress.value.port
      to_port     = ingress.value.port
      protocol    = ingress.value.protocol
      cidr_blocks = [ingress.value.cidr]
    }
  }
  
  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
  
  tags = {
    Name    = "${var.project}-${each.key}-sg"
    Service = each.key
  }
}
```

---

## Step 88: iterator Argument

### Custom Iterator Name

```hcl
# ปกติ: ใช้ชื่อของ block type เป็น iterator
resource "aws_security_group" "web" {
  name   = "web-sg"
  vpc_id = aws_vpc.main.id
  
  dynamic "ingress" {
    for_each = var.ingress_rules
    
    content {
      # Default: ใช้ "ingress" เป็น iterator name
      from_port   = ingress.value.port
      to_port     = ingress.value.port
      protocol    = ingress.value.protocol
      cidr_blocks = ingress.value.cidr_blocks
    }
  }
}

# ใช้ iterator เพื่อกำหนดชื่อ custom
resource "aws_security_group" "web_custom" {
  name   = "web-custom-sg"
  vpc_id = aws_vpc.main.id
  
  dynamic "ingress" {
    for_each = var.ingress_rules
    iterator = rule  # custom name!
    
    content {
      # ใช้ "rule" แทน "ingress"
      from_port   = rule.value.port
      to_port     = rule.value.port
      protocol    = rule.value.protocol
      cidr_blocks = rule.value.cidr_blocks
      description = "Rule ${rule.key}"  # rule.key = index
    }
  }
}
```

### Iterator ใน Nested Dynamic Blocks

```hcl
# iterator ช่วยเมื่อมี nested dynamic blocks
# ป้องกัน name collision

variable "vpc_peering_configs" {
  type = list(object({
    name              = string
    peer_vpc_id       = string
    
    # Accepter routes
    accepter_routes = list(object({
      destination_cidr = string
      route_table_id   = string
    }))
    
    # Requester routes
    requester_routes = list(object({
      destination_cidr = string
      route_table_id   = string
    }))
  }))
}

resource "aws_vpc_peering_connection" "peerings" {
  for_each = {
    for config in var.vpc_peering_configs :
    config.name => config
  }
  
  vpc_id      = aws_vpc.main.id
  peer_vpc_id = each.value.peer_vpc_id
  auto_accept = true
}

resource "aws_route" "peering_routes" {
  # สำหรับ accepter routes
  for_each = {
    for item in flatten([
      for peering in var.vpc_peering_configs : [
        for route in peering.accepter_routes : {
          key              = "${peering.name}-accepter-${route.destination_cidr}"
          peering_name     = peering.name
          destination_cidr = route.destination_cidr
          route_table_id   = route.route_table_id
        }
      ]
    ]) : item.key => item
  }
  
  route_table_id            = each.value.route_table_id
  destination_cidr_block    = each.value.destination_cidr
  vpc_peering_connection_id = aws_vpc_peering_connection.peerings[each.value.peering_name].id
}
```

---

## Step 89: Content Block Structure

### Content Block Details

```hcl
# Content block คือ body ของ dynamic block
# มีสิทธิ์เข้าถึง:
# - block_type.key   → key (index สำหรับ list, key name สำหรับ map)
# - block_type.value → value (element value)

resource "aws_autoscaling_group" "web" {
  name = "web-asg"
  
  min_size         = 2
  max_size         = 10
  desired_capacity = 3
  
  launch_template {
    id      = aws_launch_template.web.id
    version = "$Latest"
  }
  
  # Dynamic block สำหรับ tag
  dynamic "tag" {
    for_each = merge(
      local.common_tags,
      { Name = "web-instance" }
    )
    
    content {
      # tag.key   = tag key name (e.g., "Environment")
      # tag.value = tag value (e.g., "production")
      key                 = tag.key
      value               = tag.value
      propagate_at_launch = true
    }
  }
  
  # Dynamic block สำหรับ availability_zones
  dynamic "availability_zone_subgroup_control" {
    for_each = var.create_mixed_instances ? [1] : []
    
    content {
      mixed_instances_policy {
        launch_template {
          launch_template_specification {
            launch_template_id = aws_launch_template.web.id
          }
        }
      }
    }
  }
}
```

### Content กับ Complex Expressions

```hcl
variable "container_definitions" {
  type = map(object({
    image      = string
    cpu        = number
    memory     = number
    port       = number
    env_vars   = map(string)
    secrets    = map(string)
  }))
}

resource "aws_ecs_task_definition" "services" {
  for_each = var.container_definitions
  
  family = "${var.project}-${each.key}"
  cpu    = each.value.cpu
  memory = each.value.memory
  
  container_definitions = jsonencode([
    {
      name  = each.key
      image = each.value.image
      
      portMappings = [
        {
          containerPort = each.value.port
          hostPort      = each.value.port
          protocol      = "tcp"
        }
      ]
      
      # แปลง map env_vars เป็น list format สำหรับ ECS
      environment = [
        for key, val in each.value.env_vars : {
          name  = key
          value = val
        }
      ]
      
      # แปลง map secrets เป็น list format สำหรับ ECS
      secrets = [
        for key, arn in each.value.secrets : {
          name      = key
          valueFrom = arn
        }
      ]
    }
  ])
  
  requires_compatibilities = ["FARGATE"]
  network_mode             = "awsvpc"
  execution_role_arn       = aws_iam_role.ecs_execution.arn
}
```

---

## Step 90: Real Examples - AWS Security Group กับ Dynamic Ingress

### Complete AWS Security Group Module

```hcl
# modules/security_group/variables.tf

variable "name" {
  description = "Security group name"
  type        = string
}

variable "description" {
  description = "Security group description"
  type        = string
  default     = ""
}

variable "vpc_id" {
  description = "VPC ID"
  type        = string
}

variable "ingress_rules" {
  description = "Ingress rules"
  type = list(object({
    description              = optional(string, "")
    from_port                = number
    to_port                  = number
    protocol                 = string
    cidr_blocks              = optional(list(string), [])
    ipv6_cidr_blocks         = optional(list(string), [])
    source_security_group_id = optional(string, null)
    self                     = optional(bool, false)
  }))
  default = []
}

variable "egress_rules" {
  description = "Egress rules"
  type = list(object({
    description      = optional(string, "")
    from_port        = number
    to_port          = number
    protocol         = string
    cidr_blocks      = optional(list(string), [])
    ipv6_cidr_blocks = optional(list(string), [])
  }))
  
  default = [
    {
      from_port   = 0
      to_port     = 0
      protocol    = "-1"
      cidr_blocks = ["0.0.0.0/0"]
      description = "Allow all outbound"
    }
  ]
}

variable "tags" {
  type    = map(string)
  default = {}
}

# modules/security_group/main.tf

resource "aws_security_group" "this" {
  name        = var.name
  description = var.description != "" ? var.description : "Security group for ${var.name}"
  vpc_id      = var.vpc_id
  
  dynamic "ingress" {
    for_each = var.ingress_rules
    
    content {
      description              = ingress.value.description
      from_port                = ingress.value.from_port
      to_port                  = ingress.value.to_port
      protocol                 = ingress.value.protocol
      cidr_blocks              = ingress.value.cidr_blocks
      ipv6_cidr_blocks         = ingress.value.ipv6_cidr_blocks
      source_security_group_id = ingress.value.source_security_group_id
      self                     = ingress.value.self
    }
  }
  
  dynamic "egress" {
    for_each = var.egress_rules
    
    content {
      description      = egress.value.description
      from_port        = egress.value.from_port
      to_port          = egress.value.to_port
      protocol         = egress.value.protocol
      cidr_blocks      = egress.value.cidr_blocks
      ipv6_cidr_blocks = egress.value.ipv6_cidr_blocks
    }
  }
  
  tags = merge(var.tags, {
    Name = var.name
  })
  
  lifecycle {
    create_before_destroy = true
  }
}

# modules/security_group/outputs.tf

output "id" {
  description = "Security group ID"
  value       = aws_security_group.this.id
}

output "arn" {
  description = "Security group ARN"
  value       = aws_security_group.this.arn
}

output "name" {
  description = "Security group name"
  value       = aws_security_group.this.name
}
```

### การใช้ Module

```hcl
# main.tf - ใช้ security_group module

module "web_sg" {
  source = "./modules/security_group"
  
  name   = "${var.project}-${var.environment}-web-sg"
  vpc_id = module.vpc.vpc_id
  
  ingress_rules = [
    {
      description = "HTTPS from internet"
      from_port   = 443
      to_port     = 443
      protocol    = "tcp"
      cidr_blocks = ["0.0.0.0/0"]
    },
    {
      description = "HTTP for redirect"
      from_port   = 80
      to_port     = 80
      protocol    = "tcp"
      cidr_blocks = ["0.0.0.0/0"]
    }
  ]
  
  tags = local.common_tags
}

module "app_sg" {
  source = "./modules/security_group"
  
  name   = "${var.project}-${var.environment}-app-sg"
  vpc_id = module.vpc.vpc_id
  
  ingress_rules = [
    {
      description              = "Traffic from web tier"
      from_port                = 8080
      to_port                  = 8080
      protocol                 = "tcp"
      source_security_group_id = module.web_sg.id
    }
  ]
  
  tags = local.common_tags
}

module "db_sg" {
  source = "./modules/security_group"
  
  name   = "${var.project}-${var.environment}-db-sg"
  vpc_id = module.vpc.vpc_id
  
  ingress_rules = [
    {
      description              = "PostgreSQL from app tier"
      from_port                = 5432
      to_port                  = 5432
      protocol                 = "tcp"
      source_security_group_id = module.app_sg.id
    }
  ]
  
  egress_rules = []  # No outbound from DB
  
  tags = local.common_tags
}
```

---

## Before/After Comparison

### Code Reduction สำหรับ Security Groups

```
Code Comparison:
┌─────────────────────────────────────────────────────────┐
│ Without Dynamic (hardcoded 5 rules):                    │
│ ├── 50+ lines                                           │
│ ├── Duplicated boilerplate                              │
│ └── Not configurable                                    │
│                                                         │
│ With Dynamic (variable-driven):                         │
│ ├── 20 lines resource + 20 lines variables              │
│ └── Fully configurable, any number of rules             │
└─────────────────────────────────────────────────────────┘
```

```hcl
# BEFORE: 50+ lines, not flexible
resource "aws_security_group" "before" {
  name   = "hardcoded-sg"
  vpc_id = aws_vpc.main.id
  
  ingress {
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }
  
  ingress {
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }
  
  ingress {
    from_port   = 8080
    to_port     = 8080
    protocol    = "tcp"
    cidr_blocks = ["10.0.0.0/8"]
  }
  
  ingress {
    from_port   = 9090
    to_port     = 9090
    protocol    = "tcp"
    cidr_blocks = ["10.0.0.0/8"]
  }
  
  ingress {
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = ["10.0.0.0/8"]
  }
}

# AFTER: 20 lines, fully configurable
variable "sg_rules" {
  type = map(object({ from_port = number, to_port = number, protocol = string, cidr = string }))
  default = {
    http    = { from_port = 80,   to_port = 80,   protocol = "tcp", cidr = "0.0.0.0/0" }
    https   = { from_port = 443,  to_port = 443,  protocol = "tcp", cidr = "0.0.0.0/0" }
    app     = { from_port = 8080, to_port = 8080, protocol = "tcp", cidr = "10.0.0.0/8" }
    metrics = { from_port = 9090, to_port = 9090, protocol = "tcp", cidr = "10.0.0.0/8" }
    ssh     = { from_port = 22,   to_port = 22,   protocol = "tcp", cidr = "10.0.0.0/8" }
  }
}

resource "aws_security_group" "after" {
  name   = "dynamic-sg"
  vpc_id = aws_vpc.main.id
  
  dynamic "ingress" {
    for_each = var.sg_rules
    iterator = rule
    content {
      from_port   = rule.value.from_port
      to_port     = rule.value.to_port
      protocol    = rule.value.protocol
      cidr_blocks = [rule.value.cidr]
      description = "Allow ${rule.key}"
    }
  }
}
```

---

## สรุป Dynamic Blocks

### Quick Reference

```hcl
# Basic syntax
dynamic "block_name" {
  for_each = collection
  iterator = custom_name  # optional
  content {
    # block_name.key   = iteration key/index
    # block_name.value = iteration value
    arg = custom_name.value.attribute
  }
}
```

### เมื่อไหรใช้ Dynamic Blocks

| สถานการณ์ | วิธีแก้ |
|-----------|---------|
| Nested block หลายอัน (ingress rules) | `dynamic` block |
| Resources หลายอัน | `count` หรือ `for_each` |
| Optional single block | `dynamic` กับ `for_each = condition ? [1] : []` |
| Block ที่ขึ้นอยู่กับ map | `dynamic` กับ `for_each = map` |

💡 **Pro Tips:**
- ใช้ `iterator` เมื่อ block name ไม่ชัดเจน หรือมี nested dynamics
- Pre-process data ใน `locals` ก่อนส่งเข้า `dynamic`
- ใช้ `for_each = condition ? map : {}` สำหรับ optional blocks
- Test ด้วย `terraform plan` เพื่อดูว่า blocks ถูกสร้างถูกต้อง

⚠️ **Common Mistakes:**
- ลืม `content {}` block ภายใน dynamic
- ใช้ wrong iterator name (ใช้ `ingress.value` แต่ไม่ได้ set `iterator`)
- `for_each` กับ list ใช้ index เป็น key ซึ่ง unstable - ใช้ map แทน
- Dynamic block ไม่ support ทุก block type - บาง blocks เป็น computed

---

*ก่อนหน้า: [Part 008 - HCL For Expressions](part-008.md)*
*ต่อไป: [Part 010 - HCL Template Syntax & Heredocs](part-010.md)*
