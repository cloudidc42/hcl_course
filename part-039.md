# Part 039: AWS Security Groups & NACLs
# AWS Security Groups และ Network ACLs กับ Terraform

## Steps 381-390: การจัดการ Network Security อย่างละเอียด

---

## Step 381: aws_security_group Resource

### Security Group พื้นฐาน

```hcl
# security_group_basic.tf

resource "aws_security_group" "web" {
  name        = "${var.project_name}-${var.environment}-web-sg"
  description = "Security group สำหรับ web servers"
  vpc_id      = aws_vpc.main.id

  # ─── Ingress Rules ────────────────────────────────────────
  
  ingress {
    description = "HTTP จาก Internet"
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  ingress {
    description      = "HTTPS จาก Internet"
    from_port        = 443
    to_port          = 443
    protocol         = "tcp"
    cidr_blocks      = ["0.0.0.0/0"]
    ipv6_cidr_blocks = ["::/0"]  # IPv6 support
  }

  # ─── Egress Rules ─────────────────────────────────────────

  egress {
    description = "All outbound traffic"
    from_port   = 0
    to_port     = 0
    protocol    = "-1"           # -1 = all protocols
    cidr_blocks = ["0.0.0.0/0"]
  }

  # ─── Tags ────────────────────────────────────────────────

  tags = {
    Name        = "${var.project_name}-${var.environment}-web-sg"
    Environment = var.environment
  }

  # ─── Lifecycle ───────────────────────────────────────────
  
  lifecycle {
    create_before_destroy = true  # ✅ สำคัญสำหรับ SG ที่ใช้ใน EC2
  }
}
```

### ทุก Attribute ที่สำคัญ

```hcl
resource "aws_security_group" "app" {
  # ─── Identity ─────────────────────────────────────────────
  name        = "${var.project_name}-app-sg"    # ชื่อ SG
  name_prefix = null                             # ใช้ name_prefix แทน name ได้
  description = "App tier security group"        # จำเป็น
  vpc_id      = aws_vpc.main.id                 # จำเป็น
  
  # ─── Rules (inline) ───────────────────────────────────────
  # สามารถใช้ inline rules หรือ aws_security_group_rule แยก
  
  ingress {
    from_port   = 8080
    to_port     = 8080
    protocol    = "tcp"
    
    # ระบุ source ได้หลายแบบ:
    cidr_blocks      = ["10.0.0.0/8"]         # IPv4 CIDRs
    ipv6_cidr_blocks = []                      # IPv6 CIDRs
    security_groups  = [aws_security_group.alb.id]  # Other SGs
    self             = false                   # ให้ SG ตัวเองเข้าถึงกันได้
    prefix_list_ids  = []                      # Prefix list IDs
    description      = "App port from ALB"
  }

  # ─── Tags ─────────────────────────────────────────────────
  tags = { Name = "${var.project_name}-app-sg" }
  
  # ─── Lifecycle ────────────────────────────────────────────
  lifecycle {
    create_before_destroy = true
  }
  
  # ─── Timeouts ─────────────────────────────────────────────
  timeouts {
    create = "10m"
    delete = "15m"
  }
}
```

---

## Step 382: Ingress และ Egress Rules

### Common Ingress Patterns

```hcl
# common_rules.tf

# ─── HTTP/HTTPS ───────────────────────────────────────────

resource "aws_security_group" "public_web" {
  name   = "public-web-sg"
  vpc_id = aws_vpc.main.id

  # HTTP
  ingress {
    description = "HTTP"
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  # HTTPS
  ingress {
    description = "HTTPS"
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  # All outbound
  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}

# ─── Database Ports ──────────────────────────────────────

resource "aws_security_group" "database" {
  name   = "database-sg"
  vpc_id = aws_vpc.main.id

  # PostgreSQL
  ingress {
    description     = "PostgreSQL"
    from_port       = 5432
    to_port         = 5432
    protocol        = "tcp"
    security_groups = [aws_security_group.app.id]
  }

  # MySQL/MariaDB
  ingress {
    description     = "MySQL"
    from_port       = 3306
    to_port         = 3306
    protocol        = "tcp"
    security_groups = [aws_security_group.app.id]
  }

  # Redis
  ingress {
    description     = "Redis"
    from_port       = 6379
    to_port         = 6379
    protocol        = "tcp"
    security_groups = [aws_security_group.app.id]
  }

  # MongoDB
  ingress {
    description     = "MongoDB"
    from_port       = 27017
    to_port         = 27017
    protocol        = "tcp"
    security_groups = [aws_security_group.app.id]
  }

  # ไม่มี egress rules จาก database tier (ปลอดภัยกว่า)
  egress {
    description     = "Response to app tier only"
    from_port       = 0
    to_port         = 0
    protocol        = "-1"
    security_groups = [aws_security_group.app.id]
  }
}

# ─── SSH/RDP ─────────────────────────────────────────────

resource "aws_security_group" "bastion" {
  name   = "bastion-sg"
  vpc_id = aws_vpc.main.id

  # ✅ จำกัด SSH เฉพาะ IP ที่รู้จัก
  ingress {
    description = "SSH จาก Office"
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = var.office_ip_ranges  # เช่น ["203.0.113.0/24"]
  }

  # ❌ ไม่ควรทำ: SSH จาก 0.0.0.0/0
  # ingress {
  #   from_port   = 22
  #   to_port     = 22
  #   protocol    = "tcp"
  #   cidr_blocks = ["0.0.0.0/0"]  # ❌ อันตราย!
  # }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}
```

---

## Step 383: Dynamic Blocks สำหรับ Rules

### Dynamic Ingress Rules

```hcl
# dynamic_sg.tf

# ─── Dynamic blocks กับ list of ports ────────────────────

variable "web_ingress_ports" {
  description = "Ports ที่อนุญาตเข้า web servers"
  type = list(object({
    port        = number
    protocol    = string
    description = string
    cidr_blocks = list(string)
  }))
  default = [
    {
      port        = 80
      protocol    = "tcp"
      description = "HTTP"
      cidr_blocks = ["0.0.0.0/0"]
    },
    {
      port        = 443
      protocol    = "tcp"
      description = "HTTPS"
      cidr_blocks = ["0.0.0.0/0"]
    },
    {
      port        = 8080
      protocol    = "tcp"
      description = "App port"
      cidr_blocks = ["10.0.0.0/8"]
    },
  ]
}

resource "aws_security_group" "web_dynamic" {
  name   = "${var.project_name}-web-dynamic-sg"
  vpc_id = aws_vpc.main.id

  # Dynamic ingress rules
  dynamic "ingress" {
    for_each = var.web_ingress_ports
    content {
      description = ingress.value.description
      from_port   = ingress.value.port
      to_port     = ingress.value.port
      protocol    = ingress.value.protocol
      cidr_blocks = ingress.value.cidr_blocks
    }
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = { Name = "${var.project_name}-web-dynamic-sg" }
}

# ─── Dynamic blocks กับ port ranges ─────────────────────

variable "app_port_ranges" {
  type = list(object({
    from_port   = number
    to_port     = number
    protocol    = string
    source_sg   = string
    description = string
  }))
  default = []
}

resource "aws_security_group" "app_dynamic" {
  name   = "${var.project_name}-app-sg"
  vpc_id = aws_vpc.main.id

  dynamic "ingress" {
    for_each = var.app_port_ranges
    content {
      description     = ingress.value.description
      from_port       = ingress.value.from_port
      to_port         = ingress.value.to_port
      protocol        = ingress.value.protocol
      security_groups = [ingress.value.source_sg]
    }
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}
```

---

## Step 384: aws_security_group_rule (Separate Resource)

### เมื่อไหรที่ใช้ Separate Rules

```
Inline Rules (ภายใน aws_security_group):
✅ ใช้เมื่อ rules ทั้งหมดรู้ตั้งแต่แรก
✅ Simple configurations
❌ ทำให้เกิด circular dependencies ระหว่าง SGs

Separate Rules (aws_security_group_rule):
✅ แก้ circular dependencies
✅ Rules ที่สร้างทีหลัง
✅ Module ต้องการ add rules ไปยัง SG ของ consumer
⚠️ ต้องระวัง: อย่าผสม inline และ separate rules ในหัวข้อเดียวกัน
```

```hcl
# separate_rules.tf

# สร้าง SGs แค่โครง ไม่มี rules
resource "aws_security_group" "app" {
  name   = "${var.project_name}-app-sg"
  vpc_id = aws_vpc.main.id
  
  # ✅ ไม่มี inline rules เพราะใช้ aws_security_group_rule แทน
  
  lifecycle {
    create_before_destroy = true
  }
}

resource "aws_security_group" "db" {
  name   = "${var.project_name}-db-sg"
  vpc_id = aws_vpc.main.id
  
  lifecycle {
    create_before_destroy = true
  }
}

# ─── Ingress Rules ────────────────────────────────────────

resource "aws_security_group_rule" "app_ingress_http" {
  type              = "ingress"
  from_port         = 8080
  to_port           = 8080
  protocol          = "tcp"
  security_group_id = aws_security_group.app.id
  cidr_blocks       = ["10.0.0.0/8"]
  description       = "App port from internal"
}

resource "aws_security_group_rule" "app_ingress_from_alb" {
  type                     = "ingress"
  from_port                = 8080
  to_port                  = 8080
  protocol                 = "tcp"
  security_group_id        = aws_security_group.app.id
  source_security_group_id = aws_security_group.alb.id  # ← SG Reference
  description              = "App port from ALB"
}

resource "aws_security_group_rule" "db_ingress_from_app" {
  type                     = "ingress"
  from_port                = 5432
  to_port                  = 5432
  protocol                 = "tcp"
  security_group_id        = aws_security_group.db.id
  source_security_group_id = aws_security_group.app.id  # ← SG Reference
  description              = "PostgreSQL from app tier"
}

# ─── Egress Rules ─────────────────────────────────────────

resource "aws_security_group_rule" "app_egress_db" {
  type                     = "egress"
  from_port                = 5432
  to_port                  = 5432
  protocol                 = "tcp"
  security_group_id        = aws_security_group.app.id
  source_security_group_id = aws_security_group.db.id
  description              = "App to PostgreSQL"
}

resource "aws_security_group_rule" "app_egress_internet" {
  type              = "egress"
  from_port         = 443
  to_port           = 443
  protocol          = "tcp"
  security_group_id = aws_security_group.app.id
  cidr_blocks       = ["0.0.0.0/0"]
  description       = "HTTPS to internet (for AWS APIs)"
}
```

---

## Step 385: Security Group References

### Source Security Group Reference

```hcl
# sg_references.tf

# ─── ALB → App → DB pattern ──────────────────────────────

# Tier 1: ALB Security Group
resource "aws_security_group" "alb" {
  name   = "${var.project_name}-alb-sg"
  vpc_id = aws_vpc.main.id

  ingress {
    description = "HTTP จาก Internet"
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  ingress {
    description = "HTTPS จาก Internet"
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  egress {
    description     = "ไปยัง App tier เท่านั้น"
    from_port       = 8080
    to_port         = 8080
    protocol        = "tcp"
    security_groups = [aws_security_group.app.id]  # ← SG reference
  }
}

# Tier 2: App Security Group
resource "aws_security_group" "app" {
  name   = "${var.project_name}-app-sg"
  vpc_id = aws_vpc.main.id

  ingress {
    description     = "จาก ALB เท่านั้น"
    from_port       = 8080
    to_port         = 8080
    protocol        = "tcp"
    security_groups = [aws_security_group.alb.id]  # ← SG reference
  }

  egress {
    description     = "ไปยัง DB tier"
    from_port       = 5432
    to_port         = 5432
    protocol        = "tcp"
    security_groups = [aws_security_group.db.id]   # ← SG reference
  }

  egress {
    description = "HTTPS ออก Internet สำหรับ AWS APIs"
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }
}

# Tier 3: Database Security Group
resource "aws_security_group" "db" {
  name   = "${var.project_name}-db-sg"
  vpc_id = aws_vpc.main.id

  ingress {
    description     = "PostgreSQL จาก App tier เท่านั้น"
    from_port       = 5432
    to_port         = 5432
    protocol        = "tcp"
    security_groups = [aws_security_group.app.id]  # ← SG reference
  }

  # ❌ ไม่มี egress ออก Internet สำหรับ database tier (ปลอดภัยกว่า)
}
```

---

## Step 386: Self-referencing Security Groups

```hcl
# self_reference.tf

# ─── Self-referencing: members สามารถ communicate กันได้ ─

resource "aws_security_group" "cluster" {
  name   = "${var.project_name}-cluster-sg"
  vpc_id = aws_vpc.main.id

  # Members ใน SG เดียวกันสามารถ communicate กันได้ (cluster)
  ingress {
    description = "All traffic จาก SG เดียวกัน (cluster members)"
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    self        = true  # ← self-reference
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = { Name = "${var.project_name}-cluster-sg" }
}

# ─── EKS Node Group: ต้องการ self-reference ────────────

resource "aws_security_group" "eks_nodes" {
  name   = "${var.project_name}-eks-nodes-sg"
  vpc_id = aws_vpc.main.id
  
  description = "Security group สำหรับ EKS worker nodes"

  # Nodes communicate กันเอง
  ingress {
    description = "Node-to-node communication"
    from_port   = 0
    to_port     = 65535
    protocol    = "tcp"
    self        = true
  }

  ingress {
    description = "Node-to-node UDP"
    from_port   = 0
    to_port     = 65535
    protocol    = "udp"
    self        = true
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}
```

---

## Step 387: Network ACLs (NACLs)

### NACL vs Security Group

```
┌─────────────────────┬───────────────────────────────────────────────┐
│ Feature             │ Security Group    │ NACL                      │
├─────────────────────┼───────────────────┼───────────────────────────┤
│ Scope               │ Instance-level    │ Subnet-level              │
│ State               │ Stateful          │ Stateless                 │
│ Rules               │ Allow only        │ Allow and Deny            │
│ Rule Evaluation     │ All rules         │ Number order (lowest first)│
│ Default             │ Deny all inbound  │ Allow all                 │
│ Use case            │ ทุก resource      │ Additional layer of defense│
└─────────────────────┴───────────────────┴───────────────────────────┘
```

### aws_network_acl Resource

```hcl
# nacl.tf

# ─── Public Subnet NACL ────────────────────────────────────

resource "aws_network_acl" "public" {
  vpc_id     = aws_vpc.main.id
  subnet_ids = aws_subnet.public[*].id

  # ─── Inbound Rules ─────────────────────────────────────────

  # Allow HTTP (100)
  ingress {
    rule_no    = 100
    action     = "allow"
    from_port  = 80
    to_port    = 80
    protocol   = "tcp"
    cidr_block = "0.0.0.0/0"
  }

  # Allow HTTPS (110)
  ingress {
    rule_no    = 110
    action     = "allow"
    from_port  = 443
    to_port    = 443
    protocol   = "tcp"
    cidr_block = "0.0.0.0/0"
  }

  # Allow ephemeral ports (return traffic) (120)
  # Stateless NACLs ต้องอนุญาต return traffic
  ingress {
    rule_no    = 120
    action     = "allow"
    from_port  = 1024
    to_port    = 65535
    protocol   = "tcp"
    cidr_block = "0.0.0.0/0"
  }

  # Allow SSH จาก specific IPs (130)
  ingress {
    rule_no    = 130
    action     = "allow"
    from_port  = 22
    to_port    = 22
    protocol   = "tcp"
    cidr_block = "203.0.113.0/24"  # Office IP
  }

  # Deny everything else (implicit, but explicit is clearer)
  # NACL มี implicit deny ที่ rule 32767

  # ─── Outbound Rules ────────────────────────────────────────

  # Allow HTTP out (100)
  egress {
    rule_no    = 100
    action     = "allow"
    from_port  = 80
    to_port    = 80
    protocol   = "tcp"
    cidr_block = "0.0.0.0/0"
  }

  # Allow HTTPS out (110)
  egress {
    rule_no    = 110
    action     = "allow"
    from_port  = 443
    to_port    = 443
    protocol   = "tcp"
    cidr_block = "0.0.0.0/0"
  }

  # Allow ephemeral ports out (120) - return traffic
  egress {
    rule_no    = 120
    action     = "allow"
    from_port  = 1024
    to_port    = 65535
    protocol   = "tcp"
    cidr_block = "0.0.0.0/0"
  }

  tags = {
    Name = "${var.project_name}-public-nacl"
  }
}
```

### aws_network_acl_rule (Separate Rules)

```hcl
# nacl_rules.tf

# NACL สำหรับ Private Subnet
resource "aws_network_acl" "private" {
  vpc_id     = aws_vpc.main.id
  subnet_ids = aws_subnet.private[*].id

  tags = { Name = "${var.project_name}-private-nacl" }
}

# Inbound rules
resource "aws_network_acl_rule" "private_ingress_app" {
  network_acl_id = aws_network_acl.private.id
  rule_number    = 100
  egress         = false  # ingress
  protocol       = "tcp"
  rule_action    = "allow"
  cidr_block     = aws_vpc.main.cidr_block
  from_port      = 8080
  to_port        = 8080
}

resource "aws_network_acl_rule" "private_ingress_ephemeral" {
  network_acl_id = aws_network_acl.private.id
  rule_number    = 200
  egress         = false
  protocol       = "tcp"
  rule_action    = "allow"
  cidr_block     = "0.0.0.0/0"
  from_port      = 1024
  to_port        = 65535
}

resource "aws_network_acl_rule" "private_ingress_deny_all" {
  network_acl_id = aws_network_acl.private.id
  rule_number    = 32766  # ก่อน implicit 32767 deny
  egress         = false
  protocol       = "-1"
  rule_action    = "deny"
  cidr_block     = "0.0.0.0/0"
  from_port      = 0
  to_port        = 0
}

# Outbound rules
resource "aws_network_acl_rule" "private_egress_db" {
  network_acl_id = aws_network_acl.private.id
  rule_number    = 100
  egress         = true  # egress
  protocol       = "tcp"
  rule_action    = "allow"
  cidr_block     = aws_vpc.main.cidr_block
  from_port      = 5432
  to_port        = 5432
}

resource "aws_network_acl_rule" "private_egress_https" {
  network_acl_id = aws_network_acl.private.id
  rule_number    = 110
  egress         = true
  protocol       = "tcp"
  rule_action    = "allow"
  cidr_block     = "0.0.0.0/0"
  from_port      = 443
  to_port        = 443
}
```

---

## Step 388: Complete 3-tier Architecture Security Groups

```hcl
# three_tier_security.tf - Complete 3-tier architecture

# ─── ALB (Public-facing) ──────────────────────────────────

resource "aws_security_group" "alb" {
  name        = "${var.project_name}-${var.environment}-alb"
  description = "Application Load Balancer"
  vpc_id      = aws_vpc.main.id

  ingress {
    description = "HTTP from Internet"
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  ingress {
    description = "HTTPS from Internet"
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  egress {
    description = "To App tier"
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = [aws_vpc.main.cidr_block]
  }

  tags = { Name = "${var.project_name}-alb-sg" }
  lifecycle { create_before_destroy = true }
}

# ─── Web Tier ─────────────────────────────────────────────

resource "aws_security_group" "web" {
  name        = "${var.project_name}-${var.environment}-web"
  description = "Web servers"
  vpc_id      = aws_vpc.main.id
  
  lifecycle { create_before_destroy = true }
}

resource "aws_security_group_rule" "web_from_alb" {
  type                     = "ingress"
  from_port                = 80
  to_port                  = 80
  protocol                 = "tcp"
  security_group_id        = aws_security_group.web.id
  source_security_group_id = aws_security_group.alb.id
  description              = "HTTP from ALB"
}

resource "aws_security_group_rule" "web_from_bastion" {
  type                     = "ingress"
  from_port                = 22
  to_port                  = 22
  protocol                 = "tcp"
  security_group_id        = aws_security_group.web.id
  source_security_group_id = aws_security_group.bastion.id
  description              = "SSH from Bastion"
}

resource "aws_security_group_rule" "web_egress" {
  type              = "egress"
  from_port         = 0
  to_port           = 0
  protocol          = "-1"
  security_group_id = aws_security_group.web.id
  cidr_blocks       = ["0.0.0.0/0"]
  description       = "All outbound"
}

# ─── App Tier ─────────────────────────────────────────────

resource "aws_security_group" "app" {
  name        = "${var.project_name}-${var.environment}-app"
  description = "Application servers"
  vpc_id      = aws_vpc.main.id
  
  lifecycle { create_before_destroy = true }
}

resource "aws_security_group_rule" "app_from_web" {
  type                     = "ingress"
  from_port                = 8080
  to_port                  = 8080
  protocol                 = "tcp"
  security_group_id        = aws_security_group.app.id
  source_security_group_id = aws_security_group.web.id
  description              = "App port from Web tier"
}

resource "aws_security_group_rule" "app_from_bastion" {
  type                     = "ingress"
  from_port                = 22
  to_port                  = 22
  protocol                 = "tcp"
  security_group_id        = aws_security_group.app.id
  source_security_group_id = aws_security_group.bastion.id
  description              = "SSH from Bastion"
}

resource "aws_security_group_rule" "app_egress" {
  type              = "egress"
  from_port         = 0
  to_port           = 0
  protocol          = "-1"
  security_group_id = aws_security_group.app.id
  cidr_blocks       = ["0.0.0.0/0"]
  description       = "All outbound"
}

# ─── DB Tier ──────────────────────────────────────────────

resource "aws_security_group" "db" {
  name        = "${var.project_name}-${var.environment}-db"
  description = "Database servers"
  vpc_id      = aws_vpc.main.id
  
  lifecycle { create_before_destroy = true }
}

resource "aws_security_group_rule" "db_from_app" {
  type                     = "ingress"
  from_port                = 5432
  to_port                  = 5432
  protocol                 = "tcp"
  security_group_id        = aws_security_group.db.id
  source_security_group_id = aws_security_group.app.id
  description              = "PostgreSQL from App tier"
}

resource "aws_security_group_rule" "db_from_bastion" {
  type                     = "ingress"
  from_port                = 5432
  to_port                  = 5432
  protocol                 = "tcp"
  security_group_id        = aws_security_group.db.id
  source_security_group_id = aws_security_group.bastion.id
  description              = "PostgreSQL from Bastion (maintenance)"
}

# ─── Bastion Host ─────────────────────────────────────────

resource "aws_security_group" "bastion" {
  name        = "${var.project_name}-${var.environment}-bastion"
  description = "Bastion host"
  vpc_id      = aws_vpc.main.id

  ingress {
    description = "SSH จาก Office IP"
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = var.allowed_ssh_cidrs
  }

  egress {
    description = "SSH ไปยัง internal servers"
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = [aws_vpc.main.cidr_block]
  }

  tags = { Name = "${var.project_name}-bastion-sg" }
  lifecycle { create_before_destroy = true }
}

# ─── Management ───────────────────────────────────────────

resource "aws_security_group" "management" {
  name        = "${var.project_name}-${var.environment}-management"
  description = "Management access (monitoring, etc)"
  vpc_id      = aws_vpc.main.id

  # Prometheus/Grafana
  ingress {
    description = "Prometheus scrape"
    from_port   = 9090
    to_port     = 9090
    protocol    = "tcp"
    cidr_blocks = [aws_vpc.main.cidr_block]
  }

  # Node exporter
  ingress {
    description = "Node exporter"
    from_port   = 9100
    to_port     = 9100
    protocol    = "tcp"
    cidr_blocks = [aws_vpc.main.cidr_block]
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = { Name = "${var.project_name}-management-sg" }
}
```

---

## Step 389: Security Group Outputs

```hcl
# outputs.tf - Security Group outputs

output "alb_security_group_id" {
  description = "ID ของ ALB security group"
  value       = aws_security_group.alb.id
}

output "web_security_group_id" {
  description = "ID ของ Web tier security group"
  value       = aws_security_group.web.id
}

output "app_security_group_id" {
  description = "ID ของ App tier security group"
  value       = aws_security_group.app.id
}

output "db_security_group_id" {
  description = "ID ของ DB tier security group"
  value       = aws_security_group.db.id
}

output "bastion_security_group_id" {
  description = "ID ของ Bastion security group"
  value       = aws_security_group.bastion.id
}

output "all_security_group_ids" {
  description = "Map ของ security group IDs ทั้งหมด"
  value = {
    alb     = aws_security_group.alb.id
    web     = aws_security_group.web.id
    app     = aws_security_group.app.id
    db      = aws_security_group.db.id
    bastion = aws_security_group.bastion.id
  }
}
```

---

## Step 390: Security Group Best Practices

### Checklist

```
Security Group Best Practices:

✅ Least Privilege
   - เฉพาะ ports ที่จำเป็น
   - จำกัด source IPs/SGs
   - ไม่ใช้ 0.0.0.0/0 สำหรับ SSH/RDP

✅ Descriptive Names
   - ชื่อที่อธิบาย purpose ชัดเจน
   - ใส่ environment ใน name

✅ SG References ไม่ใช่ CIDR
   - ใช้ source_security_group_id แทน cidr_blocks เมื่อเป็นไปได้

✅ No Circular Dependencies
   - แยก resource definition กับ rules
   - ใช้ aws_security_group_rule แยก

✅ Separate Rules สำหรับ cross-SG dependencies
   - ป้องกัน circular dependencies

✅ lifecycle.create_before_destroy
   - ป้องกัน downtime เมื่อ update SG

✅ Default Deny
   - ลบ default egress rule ถ้าต้องการ strict egress

✅ Tags
   - ติด tags ทุก SG
   - Name tag ที่อ่านเข้าใจได้
```

### Security Anti-patterns

```hcl
# ❌ Anti-patterns ที่ควรหลีกเลี่ยง

# 1. ❌ SSH/RDP จาก Internet
resource "aws_security_group" "bad_example" {
  ingress {
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]  # ❌ อันตราย!
  }
}

# 2. ❌ ไม่มี description ใน rules
resource "aws_security_group" "no_desc" {
  ingress {
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
    # ❌ ไม่มี description = ไม่รู้ว่าทำไมถึงมี rule นี้
  }
}

# 3. ❌ Over-permissive egress
resource "aws_security_group" "overpermissive" {
  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]  # สำหรับ DB tier ไม่ควรทำ
  }
}

# 4. ❌ Using default security group
# resource "aws_instance" "using_default" {
#   # ไม่ระบุ vpc_security_group_ids = ใช้ default SG
#   # ❌ Default SG อนุญาต all traffic ระหว่าง members
# }
```

---

## สรุป: NACLs vs Security Groups

```
เมื่อใช้อะไร:

Security Groups:
✅ ใช้เสมอ (primary defense)
✅ Instance/resource level protection
✅ Stateful (ไม่ต้องกังวล return traffic)
✅ SG references ทำให้ rules ง่ายขึ้น

Network ACLs:
✅ ใช้เป็น additional layer (optional แต่แนะนำ)
✅ Subnet level protection
✅ Explicit DENY rules (block specific IPs)
✅ Compliance requirements
⚠️ ต้องระวัง return traffic (stateless!)
```

---

*จบ Part 039: AWS Security Groups & NACLs*

*ต่อไป: Part 040 - AWS S3 Buckets*
