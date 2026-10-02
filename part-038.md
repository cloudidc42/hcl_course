# Part 038: AWS VPC & Networking
# AWS VPC และ Networking กับ Terraform

## Steps 371-380: สร้าง VPC ที่สมบูรณ์และปลอดภัย

---

## Step 371: aws_vpc Resource

### VPC พื้นฐาน

```hcl
# vpc.tf

resource "aws_vpc" "main" {
  cidr_block = "10.0.0.0/16"
  
  # ─── DNS Settings ─────────────────────────────────────────
  enable_dns_hostnames = true   # ✅ จำเป็นสำหรับ private hosted zones
  enable_dns_support   = true   # ✅ จำเป็นสำหรับ DNS resolution
  
  # ─── Instance Tenancy ────────────────────────────────────
  # default    = shared tenancy (ปกติ)
  # dedicated  = dedicated tenancy (ราคาสูงกว่า)
  # host       = dedicated host
  instance_tenancy = "default"
  
  tags = {
    Name        = "${var.project_name}-${var.environment}-vpc"
    Environment = var.environment
    ManagedBy   = "terraform"
  }
}

# ─── Secondary CIDR Block (optional) ─────────────────────

resource "aws_vpc_ipv4_cidr_block_association" "secondary" {
  vpc_id     = aws_vpc.main.id
  cidr_block = "100.64.0.0/16"  # Secondary CIDR
}
```

### VPC Attributes ที่ Export

```hcl
# Useful VPC attributes:
# aws_vpc.main.id              - VPC ID
# aws_vpc.main.arn             - VPC ARN
# aws_vpc.main.cidr_block      - Primary CIDR block
# aws_vpc.main.default_route_table_id    - Default route table
# aws_vpc.main.default_security_group_id - Default security group
# aws_vpc.main.default_network_acl_id    - Default NACL
# aws_vpc.main.main_route_table_id       - Main route table
# aws_vpc.main.dhcp_options_id           - DHCP options
```

---

## Step 372: Subnets (Public & Private)

### CIDR Design RFC 1918

```
RFC 1918 Private Address Ranges:
- 10.0.0.0/8      (10.x.x.x)       - ใช้บ่อยที่สุด
- 172.16.0.0/12   (172.16.x.x - 172.31.x.x)
- 192.168.0.0/16  (192.168.x.x)

Best Practice VPC CIDR Design:
- ใช้ /16 สำหรับ VPC (65534 hosts)
- แบ่ง subnet ด้วย /24 (254 hosts) สำหรับแต่ละ subnet
- เว้นช่วง CIDR เผื่อ expansion

ตัวอย่าง Multi-environment:
- Development:  10.1.0.0/16
- Staging:      10.2.0.0/16
- Production:   10.3.0.0/16
- Shared:       10.0.0.0/16
```

### Subnet CIDR Calculation ด้วย cidrsubnet

```hcl
# locals.tf

locals {
  # VPC CIDR: 10.0.0.0/16
  vpc_cidr = "10.0.0.0/16"
  
  azs = ["ap-southeast-1a", "ap-southeast-1b", "ap-southeast-1c"]
  
  # Public Subnets:  10.0.0.0/24, 10.0.1.0/24, 10.0.2.0/24
  public_subnets = [
    for i, az in local.azs : cidrsubnet(local.vpc_cidr, 8, i)
  ]
  
  # Private Subnets: 10.0.10.0/24, 10.0.11.0/24, 10.0.12.0/24
  private_subnets = [
    for i, az in local.azs : cidrsubnet(local.vpc_cidr, 8, i + 10)
  ]
  
  # Database Subnets: 10.0.20.0/24, 10.0.21.0/24, 10.0.22.0/24
  database_subnets = [
    for i, az in local.azs : cidrsubnet(local.vpc_cidr, 8, i + 20)
  ]
  
  # Cache Subnets: 10.0.30.0/24, 10.0.31.0/24, 10.0.32.0/24
  cache_subnets = [
    for i, az in local.azs : cidrsubnet(local.vpc_cidr, 8, i + 30)
  ]
}
```

### Subnet Resources

```hcl
# subnets.tf

# ─── Public Subnets ───────────────────────────────────────

resource "aws_subnet" "public" {
  count = length(local.azs)

  vpc_id                  = aws_vpc.main.id
  cidr_block              = local.public_subnets[count.index]
  availability_zone       = local.azs[count.index]
  
  # ✅ ไม่ auto-assign public IP (ใช้ EIP เมื่อต้องการ)
  map_public_ip_on_launch = false

  tags = {
    Name                                          = "${var.project_name}-public-${local.azs[count.index]}"
    "kubernetes.io/cluster/${var.cluster_name}"   = "shared"  # optional: for EKS
    "kubernetes.io/role/elb"                      = "1"       # optional: for EKS ALB
  }
}

# ─── Private Subnets ─────────────────────────────────────

resource "aws_subnet" "private" {
  count = length(local.azs)

  vpc_id            = aws_vpc.main.id
  cidr_block        = local.private_subnets[count.index]
  availability_zone = local.azs[count.index]

  tags = {
    Name                                          = "${var.project_name}-private-${local.azs[count.index]}"
    "kubernetes.io/cluster/${var.cluster_name}"   = "shared"
    "kubernetes.io/role/internal-elb"             = "1"
  }
}

# ─── Database Subnets ────────────────────────────────────

resource "aws_subnet" "database" {
  count = length(local.azs)

  vpc_id            = aws_vpc.main.id
  cidr_block        = local.database_subnets[count.index]
  availability_zone = local.azs[count.index]

  tags = {
    Name = "${var.project_name}-database-${local.azs[count.index]}"
  }
}

# ─── RDS Subnet Group ────────────────────────────────────

resource "aws_db_subnet_group" "main" {
  name        = "${var.project_name}-db-subnet-group"
  description = "Subnet group สำหรับ RDS instances"
  subnet_ids  = aws_subnet.database[*].id

  tags = {
    Name = "${var.project_name}-db-subnet-group"
  }
}

# ─── ElastiCache Subnet Group ────────────────────────────

resource "aws_elasticache_subnet_group" "main" {
  name        = "${var.project_name}-cache-subnet-group"
  description = "Subnet group สำหรับ ElastiCache"
  subnet_ids  = aws_subnet.private[*].id
}
```

---

## Step 373: Internet Gateway และ Route Tables

```hcl
# internet_gateway.tf

resource "aws_internet_gateway" "main" {
  vpc_id = aws_vpc.main.id

  tags = {
    Name = "${var.project_name}-igw"
  }
}

# ─── Route Tables ─────────────────────────────────────────

# Public Route Table - traffic ออกไป Internet
resource "aws_route_table" "public" {
  vpc_id = aws_vpc.main.id

  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.main.id
  }

  # IPv6 route (optional)
  # route {
  #   ipv6_cidr_block = "::/0"
  #   gateway_id      = aws_internet_gateway.main.id
  # }

  tags = {
    Name = "${var.project_name}-public-rt"
  }
}

# Associate Public Subnets กับ Public Route Table
resource "aws_route_table_association" "public" {
  count = length(aws_subnet.public)

  subnet_id      = aws_subnet.public[count.index].id
  route_table_id = aws_route_table.public.id
}

# Private Route Tables - traffic ออกผ่าน NAT
resource "aws_route_table" "private" {
  count = length(local.azs)

  vpc_id = aws_vpc.main.id

  # Route ไป NAT Gateway ใน AZ เดียวกัน (high availability)
  route {
    cidr_block     = "0.0.0.0/0"
    nat_gateway_id = aws_nat_gateway.main[count.index].id
  }

  tags = {
    Name = "${var.project_name}-private-rt-${local.azs[count.index]}"
  }
}

resource "aws_route_table_association" "private" {
  count = length(aws_subnet.private)

  subnet_id      = aws_subnet.private[count.index].id
  route_table_id = aws_route_table.private[count.index].id
}

# Database Route Table - ไม่มี Internet access
resource "aws_route_table" "database" {
  vpc_id = aws_vpc.main.id

  tags = {
    Name = "${var.project_name}-database-rt"
  }
}

resource "aws_route_table_association" "database" {
  count = length(aws_subnet.database)

  subnet_id      = aws_subnet.database[count.index].id
  route_table_id = aws_route_table.database.id
}
```

---

## Step 374: NAT Gateway

```hcl
# nat_gateway.tf

# ─── EIP สำหรับ NAT Gateway ─────────────────────────────

resource "aws_eip" "nat" {
  count = var.enable_nat_gateway ? (var.single_nat_gateway ? 1 : length(local.azs)) : 0
  
  domain     = "vpc"
  depends_on = [aws_internet_gateway.main]

  tags = {
    Name = "${var.project_name}-nat-eip-${count.index + 1}"
  }
}

# ─── NAT Gateway ─────────────────────────────────────────

resource "aws_nat_gateway" "main" {
  count = var.enable_nat_gateway ? (var.single_nat_gateway ? 1 : length(local.azs)) : 0

  allocation_id = aws_eip.nat[count.index].id
  subnet_id     = aws_subnet.public[count.index].id  # ต้องอยู่ใน public subnet

  tags = {
    Name = "${var.project_name}-nat-${count.index + 1}"
  }

  depends_on = [aws_internet_gateway.main]
}
```

### Single NAT vs Multiple NAT

```hcl
# variables.tf

variable "enable_nat_gateway" {
  description = "สร้าง NAT Gateway"
  type        = bool
  default     = true
}

variable "single_nat_gateway" {
  description = <<-EOT
    ใช้ NAT Gateway เดียว (ประหยัดค่าใช้จ่าย แต่ไม่ HA)
    
    false = NAT Gateway หนึ่งตัวต่อ AZ (HA, ราคาสูงกว่า)
    true  = NAT Gateway เดียวสำหรับทุก AZ (ประหยัด, ไม่ HA)
    
    แนะนำ:
    - Development: true (ประหยัด ~$30/เดือน)
    - Production:  false (HA, ~$90/เดือน สำหรับ 3 AZs)
  EOT
  type        = bool
  default     = false
}
```

---

## Step 375: Multi-AZ Design

### Complete Multi-AZ VPC

```hcl
# multi_az_vpc.tf - Complete 3-tier, 3-AZ VPC

locals {
  azs = slice(data.aws_availability_zones.available.names, 0, 3)
  
  public_subnet_cidrs  = [for i, az in local.azs : cidrsubnet(var.vpc_cidr, 8, i)]
  private_subnet_cidrs = [for i, az in local.azs : cidrsubnet(var.vpc_cidr, 8, i + 10)]
  db_subnet_cidrs      = [for i, az in local.azs : cidrsubnet(var.vpc_cidr, 8, i + 20)]
}

data "aws_availability_zones" "available" {
  state = "available"
}

# VPC
resource "aws_vpc" "main" {
  cidr_block           = var.vpc_cidr
  enable_dns_hostnames = true
  enable_dns_support   = true
  
  tags = merge(local.common_tags, {
    Name = "${local.name_prefix}-vpc"
  })
}

# Internet Gateway
resource "aws_internet_gateway" "main" {
  vpc_id = aws_vpc.main.id
  tags   = merge(local.common_tags, { Name = "${local.name_prefix}-igw" })
}

# Public Subnets (3x - one per AZ)
resource "aws_subnet" "public" {
  count             = length(local.azs)
  vpc_id            = aws_vpc.main.id
  cidr_block        = local.public_subnet_cidrs[count.index]
  availability_zone = local.azs[count.index]
  
  tags = merge(local.common_tags, {
    Name = "${local.name_prefix}-public-${local.azs[count.index]}"
    Tier = "public"
  })
}

# Private Subnets (3x - one per AZ)
resource "aws_subnet" "private" {
  count             = length(local.azs)
  vpc_id            = aws_vpc.main.id
  cidr_block        = local.private_subnet_cidrs[count.index]
  availability_zone = local.azs[count.index]
  
  tags = merge(local.common_tags, {
    Name = "${local.name_prefix}-private-${local.azs[count.index]}"
    Tier = "private"
  })
}

# Database Subnets (3x - one per AZ)
resource "aws_subnet" "database" {
  count             = length(local.azs)
  vpc_id            = aws_vpc.main.id
  cidr_block        = local.db_subnet_cidrs[count.index]
  availability_zone = local.azs[count.index]
  
  tags = merge(local.common_tags, {
    Name = "${local.name_prefix}-database-${local.azs[count.index]}"
    Tier = "database"
  })
}

# NAT Gateway EIPs (1 per AZ)
resource "aws_eip" "nat" {
  count      = length(local.azs)
  domain     = "vpc"
  depends_on = [aws_internet_gateway.main]
  
  tags = merge(local.common_tags, {
    Name = "${local.name_prefix}-nat-eip-${local.azs[count.index]}"
  })
}

# NAT Gateways (1 per AZ for HA)
resource "aws_nat_gateway" "main" {
  count         = length(local.azs)
  allocation_id = aws_eip.nat[count.index].id
  subnet_id     = aws_subnet.public[count.index].id
  
  tags = merge(local.common_tags, {
    Name = "${local.name_prefix}-nat-${local.azs[count.index]}"
  })
  
  depends_on = [aws_internet_gateway.main]
}

# Public Route Table
resource "aws_route_table" "public" {
  vpc_id = aws_vpc.main.id
  
  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.main.id
  }
  
  tags = merge(local.common_tags, { Name = "${local.name_prefix}-public-rt" })
}

resource "aws_route_table_association" "public" {
  count          = length(aws_subnet.public)
  subnet_id      = aws_subnet.public[count.index].id
  route_table_id = aws_route_table.public.id
}

# Private Route Tables (1 per AZ)
resource "aws_route_table" "private" {
  count  = length(local.azs)
  vpc_id = aws_vpc.main.id
  
  route {
    cidr_block     = "0.0.0.0/0"
    nat_gateway_id = aws_nat_gateway.main[count.index].id
  }
  
  tags = merge(local.common_tags, {
    Name = "${local.name_prefix}-private-rt-${local.azs[count.index]}"
  })
}

resource "aws_route_table_association" "private" {
  count          = length(aws_subnet.private)
  subnet_id      = aws_subnet.private[count.index].id
  route_table_id = aws_route_table.private[count.index].id
}

# Database Route Table (no internet access)
resource "aws_route_table" "database" {
  vpc_id = aws_vpc.main.id
  tags   = merge(local.common_tags, { Name = "${local.name_prefix}-database-rt" })
}

resource "aws_route_table_association" "database" {
  count          = length(aws_subnet.database)
  subnet_id      = aws_subnet.database[count.index].id
  route_table_id = aws_route_table.database.id
}
```

---

## Step 376: VPC Endpoints

### Gateway Endpoints (Free - S3 และ DynamoDB)

```hcl
# vpc_endpoints.tf

# ─── S3 Gateway Endpoint (Free) ───────────────────────────

resource "aws_vpc_endpoint" "s3" {
  vpc_id       = aws_vpc.main.id
  service_name = "com.amazonaws.${var.aws_region}.s3"
  
  vpc_endpoint_type = "Gateway"
  route_table_ids   = concat(
    aws_route_table.private[*].id,
    [aws_route_table.database.id]
  )

  tags = {
    Name = "${var.project_name}-s3-endpoint"
  }
}

# ─── DynamoDB Gateway Endpoint (Free) ────────────────────

resource "aws_vpc_endpoint" "dynamodb" {
  vpc_id       = aws_vpc.main.id
  service_name = "com.amazonaws.${var.aws_region}.dynamodb"
  
  vpc_endpoint_type = "Gateway"
  route_table_ids   = aws_route_table.private[*].id

  tags = {
    Name = "${var.project_name}-dynamodb-endpoint"
  }
}

# ─── Interface Endpoints (มีค่าใช้จ่าย) ─────────────────

# SSM Endpoints (สำหรับ EC2 ที่ไม่มี public IP)
resource "aws_vpc_endpoint" "ssm" {
  vpc_id              = aws_vpc.main.id
  service_name        = "com.amazonaws.${var.aws_region}.ssm"
  vpc_endpoint_type   = "Interface"
  subnet_ids          = aws_subnet.private[*].id
  security_group_ids  = [aws_security_group.vpc_endpoint.id]
  private_dns_enabled = true

  tags = { Name = "${var.project_name}-ssm-endpoint" }
}

resource "aws_vpc_endpoint" "ssmmessages" {
  vpc_id              = aws_vpc.main.id
  service_name        = "com.amazonaws.${var.aws_region}.ssmmessages"
  vpc_endpoint_type   = "Interface"
  subnet_ids          = aws_subnet.private[*].id
  security_group_ids  = [aws_security_group.vpc_endpoint.id]
  private_dns_enabled = true

  tags = { Name = "${var.project_name}-ssmmessages-endpoint" }
}

resource "aws_vpc_endpoint" "ec2messages" {
  vpc_id              = aws_vpc.main.id
  service_name        = "com.amazonaws.${var.aws_region}.ec2messages"
  vpc_endpoint_type   = "Interface"
  subnet_ids          = aws_subnet.private[*].id
  security_group_ids  = [aws_security_group.vpc_endpoint.id]
  private_dns_enabled = true

  tags = { Name = "${var.project_name}-ec2messages-endpoint" }
}

# Security Group สำหรับ VPC Endpoints
resource "aws_security_group" "vpc_endpoint" {
  name        = "${var.project_name}-vpc-endpoint-sg"
  description = "Security group สำหรับ VPC Interface Endpoints"
  vpc_id      = aws_vpc.main.id

  ingress {
    description = "HTTPS จาก VPC"
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = [aws_vpc.main.cidr_block]
  }

  tags = { Name = "${var.project_name}-vpc-endpoint-sg" }
}
```

---

## Step 377: VPC Flow Logs

```hcl
# vpc_flow_logs.tf

# ─── CloudWatch Log Group ─────────────────────────────────

resource "aws_cloudwatch_log_group" "vpc_flow_logs" {
  name              = "/aws/vpc/flowlogs/${var.project_name}-${var.environment}"
  retention_in_days = var.flow_log_retention_days

  tags = {
    Name = "${var.project_name}-vpc-flow-logs"
  }
}

# ─── IAM Role สำหรับ VPC Flow Logs ───────────────────────

resource "aws_iam_role" "vpc_flow_logs" {
  name = "${var.project_name}-vpc-flow-logs-role"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Action    = "sts:AssumeRole"
      Effect    = "Allow"
      Principal = { Service = "vpc-flow-logs.amazonaws.com" }
    }]
  })
}

resource "aws_iam_role_policy" "vpc_flow_logs" {
  name = "vpc-flow-logs-policy"
  role = aws_iam_role.vpc_flow_logs.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect = "Allow"
      Action = [
        "logs:CreateLogGroup",
        "logs:CreateLogStream",
        "logs:PutLogEvents",
        "logs:DescribeLogGroups",
        "logs:DescribeLogStreams"
      ]
      Resource = "*"
    }]
  })
}

# ─── VPC Flow Log ─────────────────────────────────────────

resource "aws_flow_log" "main" {
  vpc_id          = aws_vpc.main.id
  traffic_type    = "ALL"   # ACCEPT | REJECT | ALL
  iam_role_arn    = aws_iam_role.vpc_flow_logs.arn
  log_destination = aws_cloudwatch_log_group.vpc_flow_logs.arn

  tags = {
    Name = "${var.project_name}-vpc-flow-log"
  }
}

# ─── Flow Log ไปยัง S3 (แนะนำสำหรับ long-term) ──────────

resource "aws_flow_log" "s3" {
  vpc_id               = aws_vpc.main.id
  traffic_type         = "ALL"
  log_destination_type = "s3"
  log_destination      = "${aws_s3_bucket.flow_logs.arn}/vpc-flow-logs/"
  
  destination_options {
    file_format                = "parquet"  # Parquet สำหรับ Athena queries
    hive_compatible_partitions = true
    per_hour_partition         = true
  }

  tags = {
    Name = "${var.project_name}-vpc-flow-log-s3"
  }
}
```

---

## Step 378: VPC Peering

```hcl
# vpc_peering.tf

# ─── VPC Peering Connection ─────────────────────────────

# สร้าง peering request
resource "aws_vpc_peering_connection" "app_to_shared" {
  vpc_id      = aws_vpc.app.id      # Requester VPC
  peer_vpc_id = aws_vpc.shared.id   # Accepter VPC
  
  # ถ้าเป็น account เดียวกัน: auto_accept = true
  auto_accept = true

  tags = {
    Name = "${var.project_name}-app-to-shared-peering"
  }
}

# ─── Cross-account Peering ────────────────────────────────

# Account A: Requester
resource "aws_vpc_peering_connection" "cross_account" {
  provider = aws.account_a
  
  vpc_id        = aws_vpc.account_a.id
  peer_vpc_id   = var.account_b_vpc_id
  peer_owner_id = var.account_b_id
  peer_region   = var.account_b_region
  auto_accept   = false  # ต้อง accept จาก Account B

  tags = {
    Name = "cross-account-peering"
  }
}

# Account B: Accepter
resource "aws_vpc_peering_connection_accepter" "cross_account" {
  provider = aws.account_b
  
  vpc_peering_connection_id = aws_vpc_peering_connection.cross_account.id
  auto_accept               = true

  tags = {
    Name = "cross-account-peering-accepter"
  }
}

# Route จาก VPC A ไป VPC B
resource "aws_route" "a_to_b" {
  route_table_id            = aws_route_table.private_a.id
  destination_cidr_block    = aws_vpc.account_b.cidr_block
  vpc_peering_connection_id = aws_vpc_peering_connection.cross_account.id
}

# Route จาก VPC B ไป VPC A
resource "aws_route" "b_to_a" {
  route_table_id            = aws_route_table.private_b.id
  destination_cidr_block    = aws_vpc.account_a.cidr_block
  vpc_peering_connection_id = aws_vpc_peering_connection.cross_account.id
}
```

---

## Step 379: Outputs สำหรับ VPC Module

```hcl
# modules/vpc/outputs.tf - Complete outputs

output "vpc_id" {
  description = "ID ของ VPC"
  value       = aws_vpc.main.id
}

output "vpc_arn" {
  description = "ARN ของ VPC"
  value       = aws_vpc.main.arn
}

output "vpc_cidr_block" {
  description = "CIDR block ของ VPC"
  value       = aws_vpc.main.cidr_block
}

# ─── Internet Gateway ────────────────────────────────────

output "internet_gateway_id" {
  description = "ID ของ Internet Gateway"
  value       = aws_internet_gateway.main.id
}

# ─── Subnets ─────────────────────────────────────────────

output "public_subnet_ids" {
  description = "IDs ของ public subnets"
  value       = aws_subnet.public[*].id
}

output "private_subnet_ids" {
  description = "IDs ของ private subnets"
  value       = aws_subnet.private[*].id
}

output "database_subnet_ids" {
  description = "IDs ของ database subnets"
  value       = aws_subnet.database[*].id
}

output "public_subnet_cidrs" {
  description = "CIDR blocks ของ public subnets"
  value       = aws_subnet.public[*].cidr_block
}

output "private_subnet_cidrs" {
  description = "CIDR blocks ของ private subnets"
  value       = aws_subnet.private[*].cidr_block
}

output "database_subnet_group_name" {
  description = "ชื่อของ RDS subnet group"
  value       = aws_db_subnet_group.main.name
}

# ─── NAT Gateway ─────────────────────────────────────────

output "nat_gateway_ids" {
  description = "IDs ของ NAT Gateways"
  value       = aws_nat_gateway.main[*].id
}

output "nat_gateway_public_ips" {
  description = "Public IPs ของ NAT Gateways"
  value       = aws_eip.nat[*].public_ip
}

# ─── Route Tables ─────────────────────────────────────────

output "public_route_table_id" {
  description = "ID ของ public route table"
  value       = aws_route_table.public.id
}

output "private_route_table_ids" {
  description = "IDs ของ private route tables"
  value       = aws_route_table.private[*].id
}

# ─── AZs ──────────────────────────────────────────────────

output "availability_zones" {
  description = "Availability Zones ที่ใช้"
  value       = local.azs
}
```

---

## Step 380: Complete VPC Module Example

```hcl
# ─── Module: modules/vpc/main.tf (Complete) ───────────────

terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = ">= 5.0.0"
    }
  }
}

data "aws_availability_zones" "available" {
  state = "available"
}

locals {
  azs = slice(
    data.aws_availability_zones.available.names,
    0,
    min(length(data.aws_availability_zones.available.names), var.az_count)
  )

  name_prefix = "${var.project_name}-${var.environment}"

  common_tags = merge(
    {
      Project     = var.project_name
      Environment = var.environment
      ManagedBy   = "terraform"
    },
    var.tags
  )

  public_subnet_cidrs  = [for i in range(length(local.azs)) : cidrsubnet(var.vpc_cidr, 8, i)]
  private_subnet_cidrs = [for i in range(length(local.azs)) : cidrsubnet(var.vpc_cidr, 8, i + 10)]
  database_subnet_cidrs = [for i in range(length(local.azs)) : cidrsubnet(var.vpc_cidr, 8, i + 20)]
}

# ─── VPC ──────────────────────────────────────────────────
resource "aws_vpc" "this" {
  cidr_block           = var.vpc_cidr
  enable_dns_hostnames = true
  enable_dns_support   = true
  tags = merge(local.common_tags, { Name = "${local.name_prefix}-vpc" })
}

# ─── Internet Gateway ─────────────────────────────────────
resource "aws_internet_gateway" "this" {
  vpc_id = aws_vpc.this.id
  tags   = merge(local.common_tags, { Name = "${local.name_prefix}-igw" })
}

# ─── Public Subnets ───────────────────────────────────────
resource "aws_subnet" "public" {
  count             = length(local.azs)
  vpc_id            = aws_vpc.this.id
  cidr_block        = local.public_subnet_cidrs[count.index]
  availability_zone = local.azs[count.index]
  tags = merge(local.common_tags, {
    Name = "${local.name_prefix}-public-${local.azs[count.index]}"
    Tier = "public"
  })
}

# ─── Private Subnets ─────────────────────────────────────
resource "aws_subnet" "private" {
  count             = length(local.azs)
  vpc_id            = aws_vpc.this.id
  cidr_block        = local.private_subnet_cidrs[count.index]
  availability_zone = local.azs[count.index]
  tags = merge(local.common_tags, {
    Name = "${local.name_prefix}-private-${local.azs[count.index]}"
    Tier = "private"
  })
}

# ─── Database Subnets ────────────────────────────────────
resource "aws_subnet" "database" {
  count             = length(local.azs)
  vpc_id            = aws_vpc.this.id
  cidr_block        = local.database_subnet_cidrs[count.index]
  availability_zone = local.azs[count.index]
  tags = merge(local.common_tags, {
    Name = "${local.name_prefix}-database-${local.azs[count.index]}"
    Tier = "database"
  })
}

# ─── EIPs and NAT Gateways ───────────────────────────────
resource "aws_eip" "nat" {
  count      = var.create_nat_gateway ? length(local.azs) : 0
  domain     = "vpc"
  depends_on = [aws_internet_gateway.this]
  tags = merge(local.common_tags, {
    Name = "${local.name_prefix}-nat-eip-${count.index + 1}"
  })
}

resource "aws_nat_gateway" "this" {
  count         = var.create_nat_gateway ? length(local.azs) : 0
  allocation_id = aws_eip.nat[count.index].id
  subnet_id     = aws_subnet.public[count.index].id
  depends_on    = [aws_internet_gateway.this]
  tags = merge(local.common_tags, {
    Name = "${local.name_prefix}-nat-${count.index + 1}"
  })
}

# ─── Route Tables ────────────────────────────────────────
resource "aws_route_table" "public" {
  vpc_id = aws_vpc.this.id
  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.this.id
  }
  tags = merge(local.common_tags, { Name = "${local.name_prefix}-public-rt" })
}

resource "aws_route_table_association" "public" {
  count          = length(aws_subnet.public)
  subnet_id      = aws_subnet.public[count.index].id
  route_table_id = aws_route_table.public.id
}

resource "aws_route_table" "private" {
  count  = length(local.azs)
  vpc_id = aws_vpc.this.id
  
  dynamic "route" {
    for_each = var.create_nat_gateway ? [1] : []
    content {
      cidr_block     = "0.0.0.0/0"
      nat_gateway_id = aws_nat_gateway.this[count.index].id
    }
  }
  
  tags = merge(local.common_tags, {
    Name = "${local.name_prefix}-private-rt-${local.azs[count.index]}"
  })
}

resource "aws_route_table_association" "private" {
  count          = length(aws_subnet.private)
  subnet_id      = aws_subnet.private[count.index].id
  route_table_id = aws_route_table.private[count.index].id
}

# ─── RDS Subnet Group ─────────────────────────────────────
resource "aws_db_subnet_group" "this" {
  name        = "${local.name_prefix}-db-subnet-group"
  subnet_ids  = aws_subnet.database[*].id
  tags = merge(local.common_tags, { Name = "${local.name_prefix}-db-subnet-group" })
}

# ─── S3 Endpoint (Free) ───────────────────────────────────
resource "aws_vpc_endpoint" "s3" {
  vpc_id            = aws_vpc.this.id
  service_name      = "com.amazonaws.${data.aws_region.current.name}.s3"
  vpc_endpoint_type = "Gateway"
  route_table_ids   = aws_route_table.private[*].id
  tags = merge(local.common_tags, { Name = "${local.name_prefix}-s3-endpoint" })
}

data "aws_region" "current" {}
```

### การใช้งาน VPC Module

```hcl
# environments/production/main.tf

module "vpc" {
  source = "../../modules/vpc"

  project_name = var.project_name
  environment  = "production"
  vpc_cidr     = "10.3.0.0/16"
  az_count     = 3

  create_nat_gateway = true
  
  tags = {
    CostCenter = "production-infrastructure"
  }
}

# ใช้ outputs จาก module
resource "aws_instance" "app" {
  ami           = data.aws_ami.amazon_linux_2.id
  instance_type = "t3.medium"
  subnet_id     = module.vpc.private_subnet_ids[0]
  
  vpc_security_group_ids = [aws_security_group.app.id]
}
```

---

## สรุป: VPC Design Best Practices

| หัวข้อ | Best Practice |
|--------|--------------|
| **CIDR Design** | ใช้ /16 สำหรับ VPC, /24 สำหรับ subnets |
| **Availability Zones** | อย่างน้อย 2 AZ, Production ใช้ 3 AZ |
| **NAT Gateway** | 1 per AZ สำหรับ HA ใน Production |
| **DNS** | เปิด enable_dns_hostnames และ enable_dns_support |
| **VPC Endpoints** | ใช้ S3/DynamoDB Gateway Endpoints (ฟรี) |
| **Flow Logs** | เปิด VPC Flow Logs เสมอ |
| **Subnet Tiers** | แยก public, private, database subnets |
| **Route Tables** | Public: Internet, Private: NAT, DB: ไม่มี internet |

---

*จบ Part 038: AWS VPC & Networking*

*ต่อไป: Part 039 - AWS Security Groups & NACLs*
