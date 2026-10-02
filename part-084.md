# Part 084: VPC & Network Misconfigurations
## ขั้นตอนที่ 831-840: การกำหนดค่า VPC และ Network ที่ผิดพลาด

---

## ขั้นตอนที่ 831: ภาพรวม Network Security

### Network Security Defense in Depth

```
Network Security Layers:
┌─────────────────────────────────────────────────────────┐
│                    Internet                              │
└──────────────────────┬──────────────────────────────────┘
                       │
┌──────────────────────▼──────────────────────────────────┐
│                  AWS Shield / WAF                         │
│  (DDoS Protection, Web Application Firewall)             │
└──────────────────────┬──────────────────────────────────┘
                       │
┌──────────────────────▼──────────────────────────────────┐
│                  Internet Gateway                         │
└──────────────────────┬──────────────────────────────────┘
                       │
┌──────────────────────▼──────────────────────────────────┐
│            Public Subnet (DMZ)                           │
│  ALB/NLB, Bastion Host, NAT Gateway                      │
│  NACL: Allow 80/443 inbound                              │
└──────────────────────┬──────────────────────────────────┘
                       │
┌──────────────────────▼──────────────────────────────────┐
│           Private Subnet (Application Tier)              │
│  EC2 Instances, ECS Tasks, Lambda in VPC                 │
│  Security Groups: Allow from ALB SG only                 │
└──────────────────────┬──────────────────────────────────┘
                       │
┌──────────────────────▼──────────────────────────────────┐
│           Isolated Subnet (Data Tier)                    │
│  RDS, ElastiCache, DynamoDB VPC Endpoints                │
│  Security Groups: Allow from App SG only                 │
└─────────────────────────────────────────────────────────┘
```

### CIS Benchmarks สำหรับ VPC

```
CIS AWS Foundations Benchmark - Network Controls:
- CIS 5.1: Ensure no security groups allow ingress from 0.0.0.0/0 to port 22
- CIS 5.2: Ensure no security groups allow ingress from 0.0.0.0/0 to port 3389
- CIS 5.3: Ensure default security group restricts all traffic
- CIS 5.4: Ensure routing tables for VPC peering are "least access"
```

---

## ขั้นตอนที่ 832: Misconfiguration #1 - Open Ports to Internet

### SSH (Port 22) Open to Internet

**CIS Control:** 5.1
**CVSS Score:** 9.8 (Critical) - Brute force, credential theft
**Real-World:** TeamViewer breach, multiple crypto mining attacks

#### ❌ Vulnerable - SSH Open

```hcl
# ❌ VULNERABLE - SSH open to entire internet
resource "aws_security_group" "web_server_vulnerable" {
  name   = "web-server-sg"
  vpc_id = aws_vpc.main.id
  
  # ❌ SSH open to world
  ingress {
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]   # ❌ 전 세계
    description = "SSH access"
  }
  
  # ❌ RDP open to world
  ingress {
    from_port   = 3389
    to_port     = 3389
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]   # ❌
    description = "RDP access"
  }
  
  # ❌ MySQL open to world
  ingress {
    from_port   = 3306
    to_port     = 3306
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]   # ❌ Database exposed!
    description = "MySQL access"
  }
  
  # ❌ ALL traffic open
  ingress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]   # ❌ Everything!
    description = "All traffic"
  }
  
  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
    description = "All outbound"
  }
}
```

#### ✅ Secure - Restricted Access

```hcl
# ✅ SECURE - No direct SSH/RDP access
# Use Systems Manager Session Manager instead

# ✅ Bastion Host Security Group (if needed)
resource "aws_security_group" "bastion" {
  name        = "bastion-sg"
  description = "Security group for bastion host"
  vpc_id      = aws_vpc.main.id
  
  # ✅ SSH only from specific corporate IPs
  ingress {
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = var.corporate_ip_ranges  # ✅ Known IPs only
    description = "SSH from corporate network"
  }
  
  # ✅ Egress: allow SSH to private subnets only
  egress {
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = [var.private_subnet_cidr]  # ✅ Private subnet only
    description = "SSH to private instances"
  }
  
  tags = {
    Name = "bastion-sg"
  }
}

# ✅ Application Security Group - No SSH at all
resource "aws_security_group" "app_server" {
  name        = "app-server-sg"
  description = "Security group for application servers"
  vpc_id      = aws_vpc.main.id
  
  # ✅ HTTPS from ALB only
  ingress {
    from_port                = 443
    to_port                  = 443
    protocol                 = "tcp"
    source_security_group_id = aws_security_group.alb.id  # ✅ From ALB only
    description              = "HTTPS from ALB"
  }
  
  # ✅ App port from ALB only
  ingress {
    from_port                = 8080
    to_port                  = 8080
    protocol                 = "tcp"
    source_security_group_id = aws_security_group.alb.id
    description              = "App port from ALB"
  }
  
  # ✅ SSH from bastion only (if needed)
  ingress {
    from_port                = 22
    to_port                  = 22
    protocol                 = "tcp"
    source_security_group_id = aws_security_group.bastion.id  # ✅ Bastion only
    description              = "SSH from bastion"
  }
  
  # ✅ Restricted egress
  egress {
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
    description = "HTTPS outbound"
  }
  
  egress {
    from_port                = 3306
    to_port                  = 3306
    protocol                 = "tcp"
    source_security_group_id = aws_security_group.rds.id  # ✅ DB only
    description              = "MySQL to RDS"
  }
}

# ✅ Database Security Group
resource "aws_security_group" "rds" {
  name        = "rds-sg"
  description = "Security group for RDS database"
  vpc_id      = aws_vpc.main.id
  
  # ✅ MySQL only from app servers
  ingress {
    from_port                = 3306
    to_port                  = 3306
    protocol                 = "tcp"
    source_security_group_id = aws_security_group.app_server.id  # ✅ App only
    description              = "MySQL from application servers"
  }
  
  # ✅ No egress for database
  # RDS doesn't need outbound access typically
  
  tags = {
    Name = "rds-sg"
  }
}

# ✅ Use SSM Session Manager instead of SSH
resource "aws_iam_role_policy_attachment" "ssm_policy" {
  role       = aws_iam_role.ec2_role.name
  policy_arn = "arn:aws:iam::aws:policy/AmazonSSMManagedInstanceCore"
}

# ✅ VPC Endpoint for SSM (ไม่ต้อง internet)
resource "aws_vpc_endpoint" "ssm" {
  vpc_id            = aws_vpc.main.id
  service_name      = "com.amazonaws.${var.region}.ssm"
  vpc_endpoint_type = "Interface"
  
  subnet_ids         = aws_subnet.private[*].id
  security_group_ids = [aws_security_group.vpc_endpoints.id]
  
  private_dns_enabled = true
}
```

---

## ขั้นตอนที่ 833: Misconfiguration #2 - Unrestricted Egress

### ❌ Vulnerable - Unrestricted Outbound

```hcl
# ❌ VULNERABLE - ทุกอย่างออกได้
resource "aws_security_group" "unrestricted_egress" {
  name   = "app-sg"
  vpc_id = aws_vpc.main.id
  
  # ❌ All egress = data exfiltration risk
  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}

# Risks:
# - Malware can "phone home" to C&C servers
# - Data exfiltration to attacker's server
# - Crypto mining to pool servers
# - DNS tunneling
```

### ✅ Secure - Restricted Egress

```hcl
# ✅ SECURE - Minimal egress rules
resource "aws_security_group" "restricted_egress" {
  name        = "app-sg-secure"
  description = "Restricted egress for app servers"
  vpc_id      = aws_vpc.main.id
  
  # ✅ HTTPS to internet (for updates, APIs)
  egress {
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
    description = "HTTPS outbound"
  }
  
  # ✅ HTTP (for redirects only, ideally also HTTPS)
  egress {
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
    description = "HTTP for redirects"
  }
  
  # ✅ DNS
  egress {
    from_port   = 53
    to_port     = 53
    protocol    = "udp"
    cidr_blocks = [var.vpc_cidr]
    description = "DNS queries"
  }
  
  # ✅ Database access
  egress {
    from_port                = 5432
    to_port                  = 5432
    protocol                 = "tcp"
    source_security_group_id = aws_security_group.rds.id
    description              = "PostgreSQL to RDS"
  }
  
  # ✅ Cache access
  egress {
    from_port                = 6379
    to_port                  = 6379
    protocol                 = "tcp"
    source_security_group_id = aws_security_group.elasticache.id
    description              = "Redis to ElastiCache"
  }
}

# ✅ AWS Network Firewall สำหรับ deep packet inspection
resource "aws_networkfirewall_rule_group" "block_malicious" {
  capacity = 100
  name     = "block-malicious-domains"
  type     = "STATEFUL"
  
  rule_group {
    rules_source {
      rules_source_list {
        generated_rules_type = "DENYLIST"
        target_types         = ["HTTP_HOST", "TLS_SNI"]
        
        targets = [
          "malware-c2-server.example.com",
          "*.known-bad-domain.com"
        ]
      }
    }
  }
}
```

---

## ขั้นตอนที่ 834: Misconfiguration #3 - Default VPC Usage

### ❌ Vulnerable - Using Default VPC

```hcl
# ❌ VULNERABLE - ใช้ default VPC
data "aws_vpc" "default" {
  default = true  # ❌ Default VPC
}

resource "aws_instance" "server" {
  ami           = data.aws_ami.amazon_linux.id
  instance_type = "t3.micro"
  
  subnet_id = data.aws_subnet.default.id  # ❌ Default subnet
  
  # ❌ Default VPC ปัญหา:
  # - Default security group allows all traffic
  # - Subnets have public IPs by default
  # - Less isolation between services
  # - All resources share same VPC
}

# ❌ ALSO VULNERABLE - ไม่ปิด default VPC
# Default VPC ยังคงมีอยู่ = attack surface
```

### ✅ Secure - Custom VPC

```hcl
# ✅ SECURE - Custom VPC with proper CIDR design
resource "aws_vpc" "main" {
  cidr_block = "10.0.0.0/16"  # ✅ Private CIDR
  
  # ✅ DNS settings
  enable_dns_hostnames = true
  enable_dns_support   = true
  
  tags = {
    Name        = "main-vpc"
    Environment = var.environment
  }
}

# ✅ Internet Gateway (for public subnets only)
resource "aws_internet_gateway" "main" {
  vpc_id = aws_vpc.main.id
  
  tags = {
    Name = "main-igw"
  }
}

# ✅ Public Subnets (for ALB, NAT Gateway)
resource "aws_subnet" "public" {
  count = length(var.availability_zones)
  
  vpc_id            = aws_vpc.main.id
  cidr_block        = cidrsubnet(var.public_subnet_cidr, 4, count.index)
  availability_zone = var.availability_zones[count.index]
  
  map_public_ip_on_launch = false  # ✅ ไม่ auto-assign public IP
  
  tags = {
    Name = "public-subnet-${var.availability_zones[count.index]}"
    Tier = "public"
  }
}

# ✅ Private Subnets (for application servers)
resource "aws_subnet" "private" {
  count = length(var.availability_zones)
  
  vpc_id            = aws_vpc.main.id
  cidr_block        = cidrsubnet(var.private_subnet_cidr, 4, count.index)
  availability_zone = var.availability_zones[count.index]
  
  map_public_ip_on_launch = false  # ✅ No public IPs
  
  tags = {
    Name = "private-subnet-${var.availability_zones[count.index]}"
    Tier = "private"
  }
}

# ✅ Isolated Subnets (for databases - no internet access)
resource "aws_subnet" "isolated" {
  count = length(var.availability_zones)
  
  vpc_id            = aws_vpc.main.id
  cidr_block        = cidrsubnet(var.isolated_subnet_cidr, 4, count.index)
  availability_zone = var.availability_zones[count.index]
  
  map_public_ip_on_launch = false  # ✅ No public IPs
  
  tags = {
    Name = "isolated-subnet-${var.availability_zones[count.index]}"
    Tier = "isolated"
  }
}

# ✅ Restrict default security group
resource "aws_default_security_group" "restrict_default" {
  vpc_id = aws_vpc.main.id
  
  # ✅ No ingress rules = deny all inbound
  # ✅ No egress rules = deny all outbound
  
  tags = {
    Name    = "default-restricted"
    Purpose = "Restricted default SG - do not use"
  }
}
```

---

## ขั้นตอนที่ 835: Misconfiguration #4 - Resources in Wrong Subnets

### ❌ Vulnerable - Database in Public Subnet

```hcl
# ❌ VULNERABLE - Database ใน public subnet
resource "aws_db_subnet_group" "public_db" {
  name       = "public-db-subnet-group"
  subnet_ids = aws_subnet.public[*].id  # ❌ PUBLIC SUBNETS!
  
  tags = {
    Name = "public-database-subnets"
  }
}

resource "aws_db_instance" "vulnerable_db" {
  identifier = "main-database"
  
  db_subnet_group_name   = aws_db_subnet_group.public_db.name
  publicly_accessible    = true  # ❌ Accessible from internet
  
  engine         = "mysql"
  instance_class = "db.t3.micro"
  
  username = "admin"
  password = var.db_password
  
  # ❌ Security group allows all DB access
  vpc_security_group_ids = [aws_security_group.open_db.id]
}
```

### ✅ Secure - Database in Isolated Subnet

```hcl
# ✅ SECURE - Database ใน isolated subnet
resource "aws_db_subnet_group" "isolated" {
  name        = "isolated-db-subnet-group"
  description = "Subnet group for RDS in isolated subnets"
  subnet_ids  = aws_subnet.isolated[*].id  # ✅ ISOLATED SUBNETS
  
  tags = {
    Name = "isolated-database-subnets"
  }
}

resource "aws_db_instance" "secure_db" {
  identifier = "main-database"
  
  db_subnet_group_name   = aws_db_subnet_group.isolated.name
  publicly_accessible    = false  # ✅ Not publicly accessible
  
  engine             = "mysql"
  engine_version     = "8.0"
  instance_class     = "db.t3.micro"
  allocated_storage  = 20
  storage_encrypted  = true         # ✅ Encrypted
  
  username = "admin"
  password = random_password.db.result  # ✅ Random password
  
  # ✅ Strict security group
  vpc_security_group_ids = [aws_security_group.rds.id]
  
  # ✅ Multi-AZ for availability
  multi_az = true
  
  # ✅ Backup
  backup_retention_period = 7
  backup_window           = "03:00-04:00"
  
  # ✅ Enhanced monitoring
  monitoring_interval = 60
  monitoring_role_arn = aws_iam_role.rds_monitoring.arn
  
  # ✅ Performance Insights
  performance_insights_enabled = true
  
  # ✅ Auto minor version upgrade
  auto_minor_version_upgrade = true
  
  # ✅ Deletion protection
  deletion_protection = true
  
  # ✅ CloudWatch logs
  enabled_cloudwatch_logs_exports = ["general", "error", "slowquery", "audit"]
  
  tags = {
    Name        = "main-database"
    Environment = var.environment
  }
}
```

---

## ขั้นตอนที่ 836: Misconfiguration #5 - Missing NAT Gateway

### ❌ Vulnerable - No NAT Gateway

```hcl
# ❌ VULNERABLE - Private instances ต้อง public IP เพื่อ internet
resource "aws_subnet" "private_no_nat" {
  vpc_id                  = aws_vpc.main.id
  cidr_block              = "10.0.1.0/24"
  map_public_ip_on_launch = true  # ❌ Public IP เพราะไม่มี NAT
}

# Route table goes directly to IGW
resource "aws_route" "direct_internet" {
  route_table_id         = aws_route_table.private.id
  destination_cidr_block = "0.0.0.0/0"
  gateway_id             = aws_internet_gateway.main.id  # ❌ Direct to IGW
}
```

### ✅ Secure - NAT Gateway

```hcl
# ✅ SECURE - NAT Gateway in public subnet
resource "aws_eip" "nat" {
  count  = length(var.availability_zones)
  domain = "vpc"
  
  tags = {
    Name = "nat-eip-${var.availability_zones[count.index]}"
  }
}

resource "aws_nat_gateway" "main" {
  count = length(var.availability_zones)
  
  allocation_id = aws_eip.nat[count.index].id
  subnet_id     = aws_subnet.public[count.index].id  # ✅ In public subnet
  
  depends_on = [aws_internet_gateway.main]
  
  tags = {
    Name = "nat-gateway-${var.availability_zones[count.index]}"
  }
}

# ✅ Route table for private subnets - via NAT Gateway
resource "aws_route_table" "private" {
  count  = length(var.availability_zones)
  vpc_id = aws_vpc.main.id
  
  route {
    cidr_block     = "0.0.0.0/0"
    nat_gateway_id = aws_nat_gateway.main[count.index].id  # ✅ Via NAT
  }
  
  tags = {
    Name = "private-rt-${var.availability_zones[count.index]}"
  }
}

resource "aws_route_table_association" "private" {
  count = length(var.availability_zones)
  
  subnet_id      = aws_subnet.private[count.index].id
  route_table_id = aws_route_table.private[count.index].id
}

# ✅ Route table for isolated subnets - NO internet
resource "aws_route_table" "isolated" {
  vpc_id = aws_vpc.main.id
  
  # ✅ No routes to internet - databases don't need it
  
  tags = {
    Name = "isolated-rt"
  }
}

resource "aws_route_table_association" "isolated" {
  count = length(var.availability_zones)
  
  subnet_id      = aws_subnet.isolated[count.index].id
  route_table_id = aws_route_table.isolated.id
}
```

---

## ขั้นตอนที่ 837: Misconfiguration #6 - VPC Flow Logs Disabled

### ❌ Vulnerable - No Flow Logs

```hcl
# ❌ VULNERABLE - ไม่มี VPC Flow Logs
resource "aws_vpc" "no_flow_logs" {
  cidr_block = "10.0.0.0/16"
  # ❌ ไม่มี flow logs = ไม่รู้ traffic patterns
  # ❌ ไม่สามารถ investigate security incidents
  # ❌ ไม่มี network audit trail
}
```

### ✅ Secure - VPC Flow Logs

```hcl
# ✅ SECURE - VPC Flow Logs to CloudWatch and S3

# CloudWatch Log Group
resource "aws_cloudwatch_log_group" "vpc_flow_logs" {
  name              = "/aws/vpc/flow-logs"
  retention_in_days = 90  # ✅ 90-day retention
  
  kms_key_id = aws_kms_key.logs.arn  # ✅ Encrypted
}

# IAM Role for Flow Logs
resource "aws_iam_role" "flow_logs" {
  name = "vpc-flow-logs-role"
  
  assume_role_policy = jsonencode({
    Statement = [{
      Effect    = "Allow"
      Principal = { Service = "vpc-flow-logs.amazonaws.com" }
      Action    = "sts:AssumeRole"
    }]
  })
}

resource "aws_iam_role_policy" "flow_logs" {
  name = "vpc-flow-logs-policy"
  role = aws_iam_role.flow_logs.id
  
  policy = jsonencode({
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

# ✅ VPC Flow Logs to CloudWatch
resource "aws_flow_log" "cloudwatch" {
  iam_role_arn    = aws_iam_role.flow_logs.arn
  log_destination = aws_cloudwatch_log_group.vpc_flow_logs.arn
  traffic_type    = "ALL"   # ✅ ACCEPT, REJECT, ALL
  vpc_id          = aws_vpc.main.id
  
  # ✅ Custom format with more fields
  log_format = "$${version} $${account-id} $${interface-id} $${srcaddr} $${dstaddr} $${srcport} $${dstport} $${protocol} $${packets} $${bytes} $${start} $${end} $${action} $${log-status} $${vpc-id} $${subnet-id} $${instance-id} $${tcp-flags} $${type} $${pkt-srcaddr} $${pkt-dstaddr}"
  
  tags = {
    Name = "vpc-flow-logs-cloudwatch"
  }
}

# ✅ VPC Flow Logs to S3 (for long-term storage)
resource "aws_flow_log" "s3" {
  log_destination      = "${aws_s3_bucket.flow_logs.arn}/vpc-flow-logs/"
  log_destination_type = "s3"
  traffic_type         = "ALL"
  vpc_id               = aws_vpc.main.id
  
  destination_options {
    file_format        = "parquet"  # ✅ Efficient format
    per_hour_partition = true        # ✅ Partitioned for Athena
  }
  
  tags = {
    Name = "vpc-flow-logs-s3"
  }
}

# ✅ Metric filter for rejected connections
resource "aws_cloudwatch_metric_filter" "rejected_connections" {
  name           = "RejectedConnections"
  pattern        = "[version, account_id, interface_id, srcaddr, dstaddr, srcport, dstport, protocol, packets, bytes, start, end, action=\"REJECT\", log_status]"
  log_group_name = aws_cloudwatch_log_group.vpc_flow_logs.name
  
  metric_transformation {
    name      = "RejectedConnectionsCount"
    namespace = "VPCFlowMetrics"
    value     = "1"
  }
}

# ✅ Alarm for high rejection rate (potential scan)
resource "aws_cloudwatch_metric_alarm" "high_rejection" {
  alarm_name          = "HighConnectionRejection"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = 1
  metric_name         = "RejectedConnectionsCount"
  namespace           = "VPCFlowMetrics"
  period              = 300
  statistic           = "Sum"
  threshold           = 100  # Alert if >100 rejections in 5 min
  
  alarm_actions = [aws_sns_topic.security_alerts.arn]
  alarm_description = "High rate of rejected connections - possible port scan"
}
```

---

## ขั้นตอนที่ 838: Misconfiguration #7 & #8 - NACLs & Security Groups

### Network ACLs (NACLs) - Line of Defense

```hcl
# ❌ VULNERABLE - Default NACL allows everything
# Default NACL ไม่ได้ป้องกันอะไรเลย

# ✅ SECURE - Custom NACLs
resource "aws_network_acl" "public" {
  vpc_id     = aws_vpc.main.id
  subnet_ids = aws_subnet.public[*].id
  
  # ✅ Inbound: Allow HTTP/HTTPS
  ingress {
    protocol   = "tcp"
    rule_no    = 100
    action     = "allow"
    cidr_block = "0.0.0.0/0"
    from_port  = 80
    to_port    = 80
  }
  
  ingress {
    protocol   = "tcp"
    rule_no    = 110
    action     = "allow"
    cidr_block = "0.0.0.0/0"
    from_port  = 443
    to_port    = 443
  }
  
  # ✅ Ephemeral ports for return traffic
  ingress {
    protocol   = "tcp"
    rule_no    = 900
    action     = "allow"
    cidr_block = "0.0.0.0/0"
    from_port  = 1024
    to_port    = 65535
  }
  
  # ✅ Deny everything else
  ingress {
    protocol   = "-1"
    rule_no    = 32766
    action     = "deny"
    cidr_block = "0.0.0.0/0"
    from_port  = 0
    to_port    = 0
  }
  
  # ✅ Outbound: Allow specific
  egress {
    protocol   = "tcp"
    rule_no    = 100
    action     = "allow"
    cidr_block = "0.0.0.0/0"
    from_port  = 80
    to_port    = 80
  }
  
  egress {
    protocol   = "tcp"
    rule_no    = 110
    action     = "allow"
    cidr_block = "0.0.0.0/0"
    from_port  = 443
    to_port    = 443
  }
  
  egress {
    protocol   = "tcp"
    rule_no    = 900
    action     = "allow"
    cidr_block = "0.0.0.0/0"
    from_port  = 1024
    to_port    = 65535
  }
  
  egress {
    protocol   = "-1"
    rule_no    = 32766
    action     = "deny"
    cidr_block = "0.0.0.0/0"
    from_port  = 0
    to_port    = 0
  }
  
  tags = {
    Name = "public-nacl"
  }
}

# ✅ NACL for private subnets
resource "aws_network_acl" "private" {
  vpc_id     = aws_vpc.main.id
  subnet_ids = aws_subnet.private[*].id
  
  # ✅ Inbound: Allow from ALB (port 8080)
  ingress {
    protocol   = "tcp"
    rule_no    = 100
    action     = "allow"
    cidr_block = var.public_subnet_cidr
    from_port  = 8080
    to_port    = 8080
  }
  
  # ✅ Inbound: Ephemeral from internet (return traffic via NAT)
  ingress {
    protocol   = "tcp"
    rule_no    = 900
    action     = "allow"
    cidr_block = "0.0.0.0/0"
    from_port  = 1024
    to_port    = 65535
  }
  
  # ✅ Outbound: HTTPS for downloads
  egress {
    protocol   = "tcp"
    rule_no    = 100
    action     = "allow"
    cidr_block = "0.0.0.0/0"
    from_port  = 443
    to_port    = 443
  }
  
  # ✅ Outbound: Return traffic to ALB
  egress {
    protocol   = "tcp"
    rule_no    = 200
    action     = "allow"
    cidr_block = var.public_subnet_cidr
    from_port  = 1024
    to_port    = 65535
  }
  
  tags = {
    Name = "private-nacl"
  }
}
```

---

## ขั้นตอนที่ 839: Misconfiguration #9-11 - Advanced Network Issues

### Missing VPC Endpoints

```hcl
# ❌ VULNERABLE - S3 traffic goes over internet
# ทุก S3 request ออก internet แล้วกลับมา = แพงและ insecure

# ✅ SECURE - VPC Endpoints
# S3 Gateway Endpoint (ฟรี)
resource "aws_vpc_endpoint" "s3" {
  vpc_id            = aws_vpc.main.id
  service_name      = "com.amazonaws.${var.region}.s3"
  vpc_endpoint_type = "Gateway"
  
  route_table_ids = concat(
    aws_route_table.private[*].id,
    aws_route_table.isolated[*].id
  )
  
  # ✅ Policy restricting access
  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect    = "Allow"
        Principal = "*"
        Action    = ["s3:GetObject", "s3:PutObject", "s3:ListBucket"]
        Resource = [
          aws_s3_bucket.app_data.arn,
          "${aws_s3_bucket.app_data.arn}/*"
        ]
      }
    ]
  })
  
  tags = {
    Name = "s3-vpc-endpoint"
  }
}

# DynamoDB Gateway Endpoint (ฟรี)
resource "aws_vpc_endpoint" "dynamodb" {
  vpc_id            = aws_vpc.main.id
  service_name      = "com.amazonaws.${var.region}.dynamodb"
  vpc_endpoint_type = "Gateway"
  
  route_table_ids = concat(
    aws_route_table.private[*].id,
    aws_route_table.isolated[*].id
  )
  
  tags = {
    Name = "dynamodb-vpc-endpoint"
  }
}

# Interface Endpoints (มีค่าใช้จ่าย แต่จำเป็น)
locals {
  interface_endpoints = [
    "ec2",
    "ec2messages",
    "ssm",
    "ssmmessages",
    "kms",
    "secretsmanager",
    "ecr.api",
    "ecr.dkr",
    "logs",
    "monitoring",
    "sts",
    "elasticloadbalancing"
  ]
}

resource "aws_vpc_endpoint" "interface" {
  for_each = toset(local.interface_endpoints)
  
  vpc_id              = aws_vpc.main.id
  service_name        = "com.amazonaws.${var.region}.${each.value}"
  vpc_endpoint_type   = "Interface"
  
  subnet_ids = aws_subnet.private[*].id
  
  security_group_ids = [aws_security_group.vpc_endpoints.id]
  
  private_dns_enabled = true  # ✅ DNS resolves to private IP
  
  tags = {
    Name = "${each.value}-endpoint"
  }
}

# Security Group for VPC Endpoints
resource "aws_security_group" "vpc_endpoints" {
  name        = "vpc-endpoints-sg"
  description = "Security group for VPC endpoints"
  vpc_id      = aws_vpc.main.id
  
  ingress {
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = [aws_vpc.main.cidr_block]  # ✅ VPC CIDR only
    description = "HTTPS from VPC"
  }
}
```

### Direct Internet Access to Databases

```hcl
# ❌ VULNERABLE - Database directly accessible from internet
resource "aws_db_instance" "exposed_db" {
  identifier          = "exposed-database"
  publicly_accessible = true  # ❌ Internet accessible
  
  vpc_security_group_ids = [
    aws_security_group.allow_all_db.id  # ❌ Allow all
  ]
}

# ✅ SECURE - No public access + VPN/Bastion for admin
resource "aws_db_instance" "private_db" {
  identifier          = "private-database"
  publicly_accessible = false  # ✅ Private only
  
  db_subnet_group_name   = aws_db_subnet_group.isolated.name  # ✅ Isolated subnet
  vpc_security_group_ids = [aws_security_group.rds.id]         # ✅ Strict SG
}

# ✅ RDS Proxy for application access (connection pooling + auth)
resource "aws_db_proxy" "main" {
  name                   = "main-proxy"
  debug_logging          = false
  engine_family          = "MYSQL"
  idle_client_timeout    = 1800
  require_tls            = true  # ✅ TLS required
  role_arn               = aws_iam_role.rds_proxy.arn
  vpc_security_group_ids = [aws_security_group.rds_proxy.id]
  vpc_subnet_ids         = aws_subnet.private[*].id
  
  auth {
    auth_scheme = "SECRETS"
    iam_auth    = "REQUIRED"  # ✅ IAM auth required
    secret_arn  = aws_secretsmanager_secret.db_credentials.arn
  }
}
```

### Missing Egress Filtering with AWS Network Firewall

```hcl
# ✅ AWS Network Firewall for deep inspection
resource "aws_networkfirewall_firewall_policy" "main" {
  name = "main-firewall-policy"
  
  firewall_policy {
    stateless_default_actions          = ["aws:pass"]
    stateless_fragment_default_actions = ["aws:drop"]
    
    # ✅ Block known malicious IPs
    stateless_rule_group_reference {
      priority     = 100
      resource_arn = aws_networkfirewall_rule_group.block_ips.arn
    }
    
    # ✅ Allow only specific domains
    stateful_rule_group_reference {
      resource_arn = aws_networkfirewall_rule_group.allow_domains.arn
    }
  }
}

resource "aws_networkfirewall_rule_group" "allow_domains" {
  capacity = 100
  name     = "allow-specific-domains"
  type     = "STATEFUL"
  
  rule_group {
    rules_source {
      rules_source_list {
        generated_rules_type = "ALLOWLIST"
        target_types         = ["TLS_SNI", "HTTP_HOST"]
        
        targets = [
          ".amazonaws.com",            # ✅ AWS services
          ".cloudfront.net",
          "api.github.com",            # ✅ GitHub API
          "registry-1.docker.io",      # ✅ Docker Hub
          "pypi.org",                  # ✅ Python packages
          "npmjs.org"                  # ✅ npm packages
        ]
      }
    }
  }
}
```

---

## ขั้นตอนที่ 840: VPC Security Monitoring

### VPC Security CloudWatch Dashboard

```hcl
# ✅ CloudWatch Dashboard for network security
resource "aws_cloudwatch_dashboard" "network_security" {
  dashboard_name = "NetworkSecurity"
  
  dashboard_body = jsonencode({
    widgets = [
      {
        type = "metric"
        properties = {
          metrics = [
            ["VPCFlowMetrics", "RejectedConnectionsCount"]
          ]
          period = 300
          title  = "Rejected Connections"
        }
      },
      {
        type = "metric"
        properties = {
          metrics = [
            ["AWS/NetworkFirewall", "DroppedPackets", "FirewallName", "main-firewall"]
          ]
          period = 300
          title  = "Dropped Packets"
        }
      }
    ]
  })
}

# ✅ Security alerts SNS
resource "aws_sns_topic" "security_alerts" {
  name              = "security-alerts"
  kms_master_key_id = aws_kms_key.sns.id
}

resource "aws_sns_topic_subscription" "security_email" {
  topic_arn = aws_sns_topic.security_alerts.arn
  protocol  = "email"
  endpoint  = var.security_email
}

# ✅ Amazon GuardDuty for network threats
resource "aws_guardduty_detector" "main" {
  enable = true
  
  datasources {
    vpc_flow_logs { status = "ENABLED" }
    dns_logs      { status = "ENABLED" }
  }
  
  finding_publishing_frequency = "FIFTEEN_MINUTES"
}

# ✅ GuardDuty findings → SNS
resource "aws_cloudwatch_event_rule" "guardduty_findings" {
  name        = "guardduty-findings"
  description = "Capture GuardDuty findings"
  
  event_pattern = jsonencode({
    source      = ["aws.guardduty"]
    detail-type = ["GuardDuty Finding"]
    detail = {
      severity = [{ numeric = [">=", 7] }]  # ✅ HIGH and CRITICAL only
    }
  })
}

resource "aws_cloudwatch_event_target" "guardduty_sns" {
  rule      = aws_cloudwatch_event_rule.guardduty_findings.name
  target_id = "SendToSNS"
  arn       = aws_sns_topic.security_alerts.arn
}
```

### Complete VPC Security Module Summary

```hcl
# outputs.tf - Network security outputs
output "vpc_id" {
  description = "VPC ID"
  value       = aws_vpc.main.id
}

output "private_subnet_ids" {
  description = "Private subnet IDs"
  value       = aws_subnet.private[*].id
}

output "isolated_subnet_ids" {
  description = "Isolated subnet IDs for databases"
  value       = aws_subnet.isolated[*].id
}

output "security_group_ids" {
  description = "Security group IDs"
  value = {
    alb          = aws_security_group.alb.id
    app          = aws_security_group.app_server.id
    rds          = aws_security_group.rds.id
    elasticache  = aws_security_group.elasticache.id
    bastion      = aws_security_group.bastion.id
    vpc_endpoints = aws_security_group.vpc_endpoints.id
  }
}
```

---

## สรุป VPC Security Misconfigurations

### Network Security Matrix

| Misconfiguration | Risk | CIS Control | Fix |
|-----------------|------|-------------|-----|
| SSH open to 0.0.0.0/0 | Critical | 5.1 | Restrict to bastion/VPN |
| RDP open to 0.0.0.0/0 | Critical | 5.2 | Restrict or use SSM |
| DB ports open to 0.0.0.0/0 | Critical | - | Private subnets only |
| Default VPC used | High | - | Custom VPC |
| DB in public subnet | Critical | - | Isolated subnet |
| No VPC Flow Logs | Medium | - | Enable flow logs |
| No NAT Gateway | Medium | - | NAT in public subnet |
| Default NACL | Medium | - | Custom NACLs |
| No VPC Endpoints | Medium | - | Gateway/Interface endpoints |
| Unrestricted egress | High | - | Restrict to required only |

---

*Part 084 ครอบคลุม VPC & Network Misconfigurations ทั้งหมด - ต่อไปใน Part 085 จะเจาะลึก Encryption at Rest*
