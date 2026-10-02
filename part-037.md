# Part 037: AWS EC2 Instances
# AWS EC2 Instances กับ Terraform

## Steps 361-370: การสร้างและจัดการ EC2 Instances

---

## Step 361: aws_instance Resource

### EC2 Instance พื้นฐาน

```hcl
# ec2_basic.tf

resource "aws_instance" "web" {
  # ─── Required ────────────────────────────────────────────
  ami           = "ami-0c02fb55956c7d316"  # Amazon Linux 2
  instance_type = "t3.micro"

  # ─── Network ─────────────────────────────────────────────
  subnet_id                   = aws_subnet.public.id
  vpc_security_group_ids      = [aws_security_group.web.id]
  associate_public_ip_address = true

  # ─── Storage ─────────────────────────────────────────────
  root_block_device {
    volume_type           = "gp3"
    volume_size           = 20
    delete_on_termination = true
    encrypted             = true  # ✅ Encrypt EBS
  }

  # ─── User Data ───────────────────────────────────────────
  user_data = base64encode(<<-EOF
    #!/bin/bash
    yum update -y
    yum install -y httpd
    systemctl start httpd
    systemctl enable httpd
    echo "<h1>Hello from $(hostname)</h1>" > /var/www/html/index.html
  EOF
  )

  # ─── IAM ─────────────────────────────────────────────────
  iam_instance_profile = aws_iam_instance_profile.ec2_profile.name

  # ─── SSH Key ─────────────────────────────────────────────
  key_name = aws_key_pair.deployer.key_name

  # ─── Metadata ────────────────────────────────────────────
  metadata_options {
    http_endpoint               = "enabled"
    http_tokens                 = "required"   # ✅ Enforce IMDSv2
    http_put_response_hop_limit = 1
    instance_metadata_tags      = "enabled"
  }

  # ─── Monitoring ──────────────────────────────────────────
  monitoring = true  # Enable detailed monitoring

  # ─── Tags ────────────────────────────────────────────────
  tags = {
    Name        = "${var.project_name}-${var.environment}-web"
    Role        = "web"
    Environment = var.environment
  }

  # ─── Lifecycle ───────────────────────────────────────────
  lifecycle {
    create_before_destroy = true
    ignore_changes = [
      user_data,  # ไม่ recreate เมื่อ user_data เปลี่ยน
    ]
  }
}
```

### ทุก Attribute ของ aws_instance

```hcl
# ec2_complete.tf - ทุก attribute สำคัญ

resource "aws_instance" "complete" {
  # ─── Core ────────────────────────────────────────────────
  ami                    = data.aws_ami.amazon_linux.id
  instance_type          = var.instance_type
  availability_zone      = "ap-southeast-1a"  # optional, ไม่ระบุก็ได้

  # ─── Network ─────────────────────────────────────────────
  subnet_id                   = aws_subnet.private.id
  vpc_security_group_ids      = [
    aws_security_group.web.id,
    aws_security_group.bastion.id,
  ]
  associate_public_ip_address = false          # ไม่ต้องการ public IP
  private_ip                  = "10.0.1.50"   # optional: กำหนด private IP เอง
  secondary_private_ips       = ["10.0.1.51"] # optional: secondary private IPs
  source_dest_check           = true          # ปกติ true, false สำหรับ NAT/VPN
  ipv6_address_count          = 1             # optional: กำหนด IPv6
  
  # ─── IAM ─────────────────────────────────────────────────
  iam_instance_profile = aws_iam_instance_profile.app.name
  
  # ─── SSH Key ─────────────────────────────────────────────
  key_name = aws_key_pair.main.key_name
  
  # ─── Storage ─────────────────────────────────────────────
  root_block_device {
    volume_type           = "gp3"
    volume_size           = 30          # GB
    iops                  = 3000        # สำหรับ gp3
    throughput            = 125         # MB/s สำหรับ gp3
    delete_on_termination = true
    encrypted             = true
    kms_key_id            = aws_kms_key.ebs.arn  # custom KMS key
    
    tags = {
      Name = "root-volume"
    }
  }
  
  # ─── EBS Optimization ────────────────────────────────────
  ebs_optimized = true
  
  # ─── User Data ───────────────────────────────────────────
  user_data                   = filebase64("user_data.sh")
  user_data_replace_on_change = false  # ถ้า true จะ recreate instance เมื่อ user_data เปลี่ยน
  
  # ─── Metadata ────────────────────────────────────────────
  metadata_options {
    http_endpoint               = "enabled"
    http_tokens                 = "required"  # IMDSv2
    http_put_response_hop_limit = 2           # สำหรับ containers บน EC2
    instance_metadata_tags      = "enabled"
  }
  
  # ─── Monitoring ──────────────────────────────────────────
  monitoring = true  # CloudWatch detailed monitoring
  
  # ─── Tenancy ─────────────────────────────────────────────
  tenancy = "default"  # default | dedicated | host
  
  # ─── Placement ───────────────────────────────────────────
  placement_group            = aws_placement_group.app.id  # optional
  placement_partition_number = 1                           # optional
  
  # ─── Hibernation ─────────────────────────────────────────
  hibernation = false  # optional
  
  # ─── Tags ────────────────────────────────────────────────
  tags = local.instance_tags
  
  volume_tags = {  # Tags สำหรับ EBS volumes
    Environment = var.environment
    ManagedBy   = "terraform"
  }
  
  # ─── Lifecycle ───────────────────────────────────────────
  lifecycle {
    create_before_destroy = true
    prevent_destroy       = false
    ignore_changes = [
      ami,           # ไม่ update เมื่อ AMI เปลี่ยน (จัดการแยก)
      user_data,     # ไม่ recreate เมื่อ user_data เปลี่ยน
    ]
  }
  
  # ─── Dependencies ────────────────────────────────────────
  depends_on = [
    aws_iam_role_policy_attachment.app_policy
  ]
}
```

---

## Step 362: AMI Selection Strategies

### 1. ใช้ Data Source (แนะนำ)

```hcl
# data.tf

# Amazon Linux 2
data "aws_ami" "amazon_linux_2" {
  most_recent = true
  owners      = ["amazon"]

  filter {
    name   = "name"
    values = ["amzn2-ami-hvm-*-x86_64-gp2"]
  }

  filter {
    name   = "virtualization-type"
    values = ["hvm"]
  }
  
  filter {
    name   = "root-device-type"
    values = ["ebs"]
  }
}

# Amazon Linux 2023
data "aws_ami" "amazon_linux_2023" {
  most_recent = true
  owners      = ["amazon"]

  filter {
    name   = "name"
    values = ["al2023-ami-*-x86_64"]
  }
}

# Ubuntu 22.04 LTS
data "aws_ami" "ubuntu_22_04" {
  most_recent = true
  owners      = ["099720109477"]  # Canonical

  filter {
    name   = "name"
    values = ["ubuntu/images/hvm-ssd/ubuntu-jammy-22.04-amd64-server-*"]
  }

  filter {
    name   = "virtualization-type"
    values = ["hvm"]
  }
}

# Debian 12
data "aws_ami" "debian_12" {
  most_recent = true
  owners      = ["136693071363"]  # Debian

  filter {
    name   = "name"
    values = ["debian-12-amd64-*"]
  }
}

# Windows Server 2022
data "aws_ami" "windows_2022" {
  most_recent = true
  owners      = ["amazon"]

  filter {
    name   = "name"
    values = ["Windows_Server-2022-English-Full-Base-*"]
  }

  filter {
    name   = "virtualization-type"
    values = ["hvm"]
  }
}

# Custom AMI จาก Account ของเรา
data "aws_ami" "custom_app" {
  most_recent = true
  owners      = ["self"]

  filter {
    name   = "name"
    values = ["myapp-*"]
  }
  
  filter {
    name   = "tag:Environment"
    values = ["production"]
  }
  
  filter {
    name   = "state"
    values = ["available"]
  }
}
```

### 2. ใช้ SSM Parameter Store สำหรับ AMI IDs

```hcl
# ✅ Best Practice: ใช้ SSM Parameters แทน data sources โดยตรง
# AWS maintain AMI IDs ใน SSM Parameter Store

data "aws_ssm_parameter" "amazon_linux_2" {
  name = "/aws/service/ami-amazon-linux-latest/amzn2-ami-hvm-x86_64-gp2"
}

data "aws_ssm_parameter" "amazon_linux_2023" {
  name = "/aws/service/ami-amazon-linux-latest/al2023-ami-kernel-default-x86_64"
}

data "aws_ssm_parameter" "ubuntu_22_04" {
  name = "/aws/service/canonical/ubuntu/server/22.04/stable/current/amd64/hvm/ebs-gp2/ami-id"
}

resource "aws_instance" "web" {
  ami           = data.aws_ssm_parameter.amazon_linux_2.value
  instance_type = "t3.micro"
}
```

---

## Step 363: Key Pair (aws_key_pair)

### การจัดการ Key Pairs

```hcl
# ─── วิธีที่ 1: Import existing public key ────────────────

# ✅ แนะนำ: สร้าง key pair นอก Terraform แล้ว import public key
resource "aws_key_pair" "deployer" {
  key_name   = "${var.project_name}-${var.environment}-key"
  public_key = file("~/.ssh/id_rsa.pub")  # ✅ Import public key เท่านั้น
  
  tags = {
    Name = "${var.project_name}-${var.environment}-key"
  }
}

# ─── วิธีที่ 2: Generate ด้วย TLS Provider ───────────────

# ⚠️ Private key จะถูกเก็บใน state file!
resource "tls_private_key" "ec2_key" {
  algorithm = "RSA"
  rsa_bits  = 4096
}

resource "aws_key_pair" "generated" {
  key_name   = "${var.project_name}-generated-key"
  public_key = tls_private_key.ec2_key.public_key_openssh
}

# ⚠️ เก็บ private key ไว้ใน local file (ต้อง protect!)
resource "local_sensitive_file" "private_key" {
  content         = tls_private_key.ec2_key.private_key_pem
  filename        = "${path.module}/keys/${var.project_name}.pem"
  file_permission = "0600"
}

# ─── วิธีที่ 3: ใช้ Secrets Manager ──────────────────────

# ✅ แนะนำสำหรับ production: เก็บ private key ใน Secrets Manager
resource "tls_private_key" "secure_key" {
  algorithm = "RSA"
  rsa_bits  = 4096
}

resource "aws_key_pair" "secure" {
  key_name   = "${var.project_name}-secure-key"
  public_key = tls_private_key.secure_key.public_key_openssh
}

# ✅ เก็บ private key ใน Secrets Manager
resource "aws_secretsmanager_secret" "ec2_private_key" {
  name                    = "/${var.project_name}/${var.environment}/ec2-private-key"
  description             = "Private key สำหรับ EC2 instances"
  recovery_window_in_days = 7
}

resource "aws_secretsmanager_secret_version" "ec2_private_key" {
  secret_id     = aws_secretsmanager_secret.ec2_private_key.id
  secret_string = tls_private_key.secure_key.private_key_pem
}
```

---

## Step 364: Security Groups สำหรับ EC2

```hcl
# security_groups.tf

# ─── Web Server Security Group ────────────────────────────

resource "aws_security_group" "web" {
  name        = "${var.project_name}-${var.environment}-web-sg"
  description = "Security group สำหรับ web servers"
  vpc_id      = aws_vpc.main.id

  # HTTP
  ingress {
    description = "HTTP จาก Internet"
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  # HTTPS
  ingress {
    description = "HTTPS จาก Internet"
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  # SSH - จาก Bastion เท่านั้น
  ingress {
    description     = "SSH จาก Bastion"
    from_port       = 22
    to_port         = 22
    protocol        = "tcp"
    security_groups = [aws_security_group.bastion.id]
  }

  # All egress
  egress {
    description = "All outbound traffic"
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = {
    Name = "${var.project_name}-${var.environment}-web-sg"
  }
}

# ─── Bastion Host Security Group ──────────────────────────

resource "aws_security_group" "bastion" {
  name        = "${var.project_name}-${var.environment}-bastion-sg"
  description = "Security group สำหรับ Bastion host"
  vpc_id      = aws_vpc.main.id

  # SSH จาก IP ที่อนุญาต
  ingress {
    description = "SSH จาก office IP"
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = var.allowed_ssh_cidr_blocks  # เช่น ["203.0.113.0/24"]
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = {
    Name = "${var.project_name}-${var.environment}-bastion-sg"
  }
}
```

---

## Step 365: IAM Instance Profile

```hcl
# iam.tf

# ─── IAM Role สำหรับ EC2 ──────────────────────────────────

resource "aws_iam_role" "ec2_app" {
  name = "${var.project_name}-${var.environment}-ec2-app-role"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Action    = "sts:AssumeRole"
      Effect    = "Allow"
      Principal = { Service = "ec2.amazonaws.com" }
    }]
  })

  tags = {
    Name = "${var.project_name}-ec2-app-role"
  }
}

# ─── Policies ────────────────────────────────────────────

# SSM Session Manager (แทน SSH)
resource "aws_iam_role_policy_attachment" "ssm" {
  role       = aws_iam_role.ec2_app.name
  policy_arn = "arn:aws:iam::aws:policy/AmazonSSMManagedInstanceCore"
}

# CloudWatch Agent
resource "aws_iam_role_policy_attachment" "cloudwatch" {
  role       = aws_iam_role.ec2_app.name
  policy_arn = "arn:aws:iam::aws:policy/CloudWatchAgentServerPolicy"
}

# S3 Access (custom policy)
resource "aws_iam_policy" "s3_access" {
  name        = "${var.project_name}-${var.environment}-s3-access"
  description = "S3 access policy สำหรับ EC2 app instances"

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Action = [
          "s3:GetObject",
          "s3:PutObject",
          "s3:DeleteObject",
          "s3:ListBucket"
        ]
        Resource = [
          "arn:aws:s3:::${var.project_name}-${var.environment}-data",
          "arn:aws:s3:::${var.project_name}-${var.environment}-data/*"
        ]
      },
      {
        Effect = "Allow"
        Action = [
          "secretsmanager:GetSecretValue",
          "secretsmanager:DescribeSecret"
        ]
        Resource = "arn:aws:secretsmanager:*:*:secret:/${var.project_name}/${var.environment}/*"
      }
    ]
  })
}

resource "aws_iam_role_policy_attachment" "s3_access" {
  role       = aws_iam_role.ec2_app.name
  policy_arn = aws_iam_policy.s3_access.arn
}

# ─── Instance Profile ────────────────────────────────────

resource "aws_iam_instance_profile" "ec2_app" {
  name = "${var.project_name}-${var.environment}-ec2-app-profile"
  role = aws_iam_role.ec2_app.name
}
```

---

## Step 366: User Data Scripts

```hcl
# ─── Simple User Data ──────────────────────────────────────

resource "aws_instance" "simple" {
  ami           = data.aws_ami.amazon_linux_2.id
  instance_type = "t3.micro"

  user_data = base64encode(<<-EOF
    #!/bin/bash
    set -ex
    
    # Update system
    yum update -y
    
    # Install web server
    yum install -y httpd
    systemctl start httpd
    systemctl enable httpd
    
    # Create test page
    cat > /var/www/html/index.html << 'HTML'
    <!DOCTYPE html>
    <html>
    <body><h1>Hello from $(hostname)</h1></body>
    </html>
    HTML
    
    # Install CloudWatch agent
    yum install -y amazon-cloudwatch-agent
    systemctl start amazon-cloudwatch-agent
    systemctl enable amazon-cloudwatch-agent
  EOF
  )
}

# ─── User Data จากไฟล์ templatefile ───────────────────────

# user_data.sh.tftpl
# #!/bin/bash
# set -ex
#
# # Install required packages
# yum update -y
# yum install -y ${packages}
#
# # Configure application
# export APP_ENV="${environment}"
# export DB_HOST="${db_endpoint}"
# export DB_NAME="${db_name}"
# export S3_BUCKET="${s3_bucket}"
# export REGION="${region}"
#
# # Download app from S3
# aws s3 cp s3://${s3_bucket}/app/latest/app.tar.gz /tmp/
# tar -xzf /tmp/app.tar.gz -C /opt/app
#
# # Start application
# systemctl start myapp
# systemctl enable myapp

resource "aws_instance" "with_template" {
  ami           = data.aws_ami.amazon_linux_2.id
  instance_type = var.instance_type
  subnet_id     = aws_subnet.private.id

  user_data = base64encode(templatefile("${path.module}/templates/user_data.sh.tftpl", {
    environment  = var.environment
    packages     = "httpd amazon-cloudwatch-agent jq"
    db_endpoint  = aws_db_instance.main.endpoint
    db_name      = var.db_name
    s3_bucket    = aws_s3_bucket.app.id
    region       = var.aws_region
  }))

  iam_instance_profile = aws_iam_instance_profile.ec2_app.name
}

# ─── Cloud-init Multi-part ────────────────────────────────

data "cloudinit_config" "app" {
  gzip          = true
  base64_encode = true

  # Parte 1: Cloud Config (YAML)
  part {
    filename     = "cloud-config.yaml"
    content_type = "text/cloud-config"
    content = yamlencode({
      packages = ["httpd", "jq", "git"]
      runcmd = [
        "systemctl start httpd",
        "systemctl enable httpd"
      ]
    })
  }

  # Part 2: Shell Script
  part {
    filename     = "setup.sh"
    content_type = "text/x-shellscript"
    content = templatefile("${path.module}/templates/setup.sh.tftpl", {
      environment = var.environment
      app_version = var.app_version
    })
  }
}

resource "aws_instance" "cloudinit" {
  ami           = data.aws_ami.amazon_linux_2.id
  instance_type = "t3.small"
  user_data     = data.cloudinit_config.app.rendered
}
```

---

## Step 367: EBS Volumes

```hcl
# ebs_volumes.tf

# ─── Root Volume (ใน aws_instance) ────────────────────────

resource "aws_instance" "with_volumes" {
  ami           = data.aws_ami.amazon_linux_2.id
  instance_type = "t3.large"

  # Root Volume
  root_block_device {
    volume_type           = "gp3"
    volume_size           = 50      # 50 GB
    iops                  = 3000    # Default สำหรับ gp3
    throughput            = 125     # MB/s default สำหรับ gp3
    delete_on_termination = true
    encrypted             = true
    
    tags = {
      Name = "app-root-volume"
    }
  }

  # Additional EBS Volume (ใน aws_instance block)
  ebs_block_device {
    device_name           = "/dev/xvdf"
    volume_type           = "gp3"
    volume_size           = 100     # 100 GB สำหรับ data
    iops                  = 3000
    throughput            = 125
    delete_on_termination = false   # ✅ ไม่ลบเมื่อ terminate instance
    encrypted             = true
    
    tags = {
      Name    = "app-data-volume"
      Purpose = "application-data"
    }
  }
}

# ─── Separate EBS Volume Resource ─────────────────────────

resource "aws_ebs_volume" "data" {
  availability_zone = aws_instance.app.availability_zone
  type              = "gp3"
  size              = 200    # 200 GB
  iops              = 6000   # เพิ่ม IOPS สำหรับ high performance
  throughput        = 250    # MB/s
  encrypted         = true
  kms_key_id        = aws_kms_key.ebs.arn

  tags = {
    Name        = "app-data-volume"
    Application = var.project_name
  }
}

# Attach EBS Volume
resource "aws_volume_attachment" "data" {
  device_name                    = "/dev/xvdf"
  volume_id                      = aws_ebs_volume.data.id
  instance_id                    = aws_instance.app.id
  force_detach                   = false   # ไม่ force detach
  stop_instance_before_detaching = true    # stop instance ก่อน detach
}

# ─── EBS Snapshot ────────────────────────────────────────

resource "aws_ebs_snapshot" "data_backup" {
  volume_id   = aws_ebs_volume.data.id
  description = "Backup ของ data volume"

  tags = {
    Name        = "data-volume-backup"
    CreatedAt   = timestamp()
    Application = var.project_name
  }
}

# ─── KMS Key สำหรับ EBS Encryption ──────────────────────

resource "aws_kms_key" "ebs" {
  description             = "KMS key สำหรับ EBS encryption"
  deletion_window_in_days = 10
  enable_key_rotation     = true

  tags = {
    Name = "${var.project_name}-ebs-kms-key"
  }
}

resource "aws_kms_alias" "ebs" {
  name          = "alias/${var.project_name}-ebs"
  target_key_id = aws_kms_key.ebs.key_id
}
```

---

## Step 368: Instance Metadata Service (IMDSv2)

```hcl
# ✅ IMDSv2: บังคับใช้ IMDSv2 สำหรับ security

resource "aws_instance" "secure" {
  ami           = data.aws_ami.amazon_linux_2.id
  instance_type = "t3.micro"

  metadata_options {
    # Enable IMDS
    http_endpoint = "enabled"
    
    # ✅ บังคับใช้ IMDSv2 (ต้องมี session token)
    http_tokens = "required"
    
    # Limit hop count (1 สำหรับ EC2, 2 สำหรับ containers บน EC2)
    http_put_response_hop_limit = 1
    
    # ✅ Allow ดู instance tags ผ่าน IMDS
    instance_metadata_tags = "enabled"
  }
}

# ─── Account-level IMDSv2 Enforcement ─────────────────────

# ✅ บังคับ IMDSv2 สำหรับ instances ใหม่ทั้งหมดใน account
resource "aws_ec2_instance_metadata_defaults" "require_imdsv2" {
  http_tokens                 = "required"
  http_put_response_hop_limit = 1
}
```

### ทดสอบ IMDSv2 จากภายใน Instance

```bash
# ─── IMDSv2 Token Request ─────────────────────────────────

# Get token (IMDSv2)
TOKEN=$(curl -s -X PUT "http://169.254.169.254/latest/api/token" \
  -H "X-aws-ec2-metadata-token-ttl-seconds: 21600")

# Use token
curl -s -H "X-aws-ec2-metadata-token: $TOKEN" \
  http://169.254.169.254/latest/meta-data/

# Get instance ID
curl -s -H "X-aws-ec2-metadata-token: $TOKEN" \
  http://169.254.169.254/latest/meta-data/instance-id

# Get instance type
curl -s -H "X-aws-ec2-metadata-token: $TOKEN" \
  http://169.254.169.254/latest/meta-data/instance-type

# Get IAM credentials (temporary)
curl -s -H "X-aws-ec2-metadata-token: $TOKEN" \
  http://169.254.169.254/latest/meta-data/iam/security-credentials/
```

---

## Step 369: Elastic IP

```hcl
# elastic_ip.tf

# ─── Elastic IP ───────────────────────────────────────────

resource "aws_eip" "web" {
  domain = "vpc"
  
  # Associate กับ instance
  instance = aws_instance.web.id
  
  # หรือ associate กับ network interface
  # network_interface = aws_network_interface.web.id
  
  # ต้องรอ IGW ก่อน
  depends_on = [aws_internet_gateway.main]

  tags = {
    Name = "${var.project_name}-web-eip"
  }
}

# ─── EIP สำหรับ NAT Gateway ──────────────────────────────

resource "aws_eip" "nat" {
  count  = var.enable_nat_gateway ? length(var.availability_zones) : 0
  domain = "vpc"

  depends_on = [aws_internet_gateway.main]

  tags = {
    Name = "${var.project_name}-nat-eip-${count.index + 1}"
  }
}

# ─── Outputs ──────────────────────────────────────────────

output "web_public_ip" {
  description = "Public IP ของ web server"
  value       = aws_eip.web.public_ip
}

output "web_public_dns" {
  description = "Public DNS ของ web server"
  value       = aws_eip.web.public_dns
}
```

---

## Step 370: Launch Templates และ Complete Web Server Example

### Launch Template

```hcl
# launch_template.tf

resource "aws_launch_template" "app" {
  name_prefix   = "${var.project_name}-${var.environment}-"
  image_id      = data.aws_ami.amazon_linux_2.id
  instance_type = var.instance_type
  key_name      = aws_key_pair.main.key_name

  # ─── Network ──────────────────────────────────────────────
  network_interfaces {
    associate_public_ip_address = false
    security_groups             = [aws_security_group.app.id]
    delete_on_termination       = true
  }

  # ─── IAM ──────────────────────────────────────────────────
  iam_instance_profile {
    name = aws_iam_instance_profile.ec2_app.name
  }

  # ─── Storage ──────────────────────────────────────────────
  block_device_mappings {
    device_name = "/dev/xvda"
    
    ebs {
      volume_type           = "gp3"
      volume_size           = 30
      encrypted             = true
      delete_on_termination = true
    }
  }

  # ─── Metadata ─────────────────────────────────────────────
  metadata_options {
    http_endpoint               = "enabled"
    http_tokens                 = "required"
    http_put_response_hop_limit = 2
  }

  # ─── Monitoring ───────────────────────────────────────────
  monitoring {
    enabled = true
  }

  # ─── User Data ────────────────────────────────────────────
  user_data = base64encode(templatefile("${path.module}/templates/user_data.sh.tftpl", {
    environment = var.environment
    app_version = var.app_version
  }))

  # ─── Tags ─────────────────────────────────────────────────
  tag_specifications {
    resource_type = "instance"
    tags = merge(local.common_tags, {
      Name = "${var.project_name}-${var.environment}-app"
      Role = "application"
    })
  }

  tag_specifications {
    resource_type = "volume"
    tags = merge(local.common_tags, {
      Name = "${var.project_name}-${var.environment}-app-volume"
    })
  }

  tags = local.common_tags

  # ─── Lifecycle ────────────────────────────────────────────
  lifecycle {
    create_before_destroy = true
  }
}
```

### Complete Web Server Example

```hcl
# complete_web_server.tf - ตัวอย่าง Web Server ที่ครบสมบูรณ์

terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

provider "aws" {
  region = var.aws_region
  
  default_tags {
    tags = {
      Project     = var.project_name
      Environment = var.environment
      ManagedBy   = "terraform"
    }
  }
}

# ─── Data Sources ─────────────────────────────────────────

data "aws_ami" "amazon_linux_2" {
  most_recent = true
  owners      = ["amazon"]
  
  filter {
    name   = "name"
    values = ["amzn2-ami-hvm-*-x86_64-gp2"]
  }
}

data "aws_availability_zones" "available" {
  state = "available"
}

# ─── VPC ──────────────────────────────────────────────────

resource "aws_vpc" "main" {
  cidr_block           = "10.0.0.0/16"
  enable_dns_hostnames = true
  enable_dns_support   = true

  tags = { Name = "${var.project_name}-vpc" }
}

resource "aws_subnet" "public" {
  vpc_id                  = aws_vpc.main.id
  cidr_block              = "10.0.1.0/24"
  availability_zone       = data.aws_availability_zones.available.names[0]
  map_public_ip_on_launch = false  # ใช้ EIP แทน

  tags = { Name = "${var.project_name}-public" }
}

resource "aws_internet_gateway" "main" {
  vpc_id = aws_vpc.main.id
  tags   = { Name = "${var.project_name}-igw" }
}

resource "aws_route_table" "public" {
  vpc_id = aws_vpc.main.id

  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.main.id
  }

  tags = { Name = "${var.project_name}-public-rt" }
}

resource "aws_route_table_association" "public" {
  subnet_id      = aws_subnet.public.id
  route_table_id = aws_route_table.public.id
}

# ─── Security ─────────────────────────────────────────────

resource "aws_security_group" "web" {
  name        = "${var.project_name}-web-sg"
  description = "Security group สำหรับ web server"
  vpc_id      = aws_vpc.main.id

  ingress {
    description = "HTTP"
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  ingress {
    description = "HTTPS"
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = { Name = "${var.project_name}-web-sg" }
}

# ─── IAM ──────────────────────────────────────────────────

resource "aws_iam_role" "web" {
  name = "${var.project_name}-web-role"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Action    = "sts:AssumeRole"
      Effect    = "Allow"
      Principal = { Service = "ec2.amazonaws.com" }
    }]
  })
}

resource "aws_iam_role_policy_attachment" "ssm" {
  role       = aws_iam_role.web.name
  policy_arn = "arn:aws:iam::aws:policy/AmazonSSMManagedInstanceCore"
}

resource "aws_iam_instance_profile" "web" {
  name = "${var.project_name}-web-profile"
  role = aws_iam_role.web.name
}

# ─── EC2 Instance ─────────────────────────────────────────

resource "aws_instance" "web" {
  ami           = data.aws_ami.amazon_linux_2.id
  instance_type = "t3.micro"
  subnet_id     = aws_subnet.public.id

  vpc_security_group_ids = [aws_security_group.web.id]
  iam_instance_profile   = aws_iam_instance_profile.web.name

  root_block_device {
    volume_type           = "gp3"
    volume_size           = 20
    encrypted             = true
    delete_on_termination = true
  }

  metadata_options {
    http_endpoint               = "enabled"
    http_tokens                 = "required"
    http_put_response_hop_limit = 1
    instance_metadata_tags      = "enabled"
  }

  user_data = base64encode(<<-EOF
    #!/bin/bash
    yum update -y
    yum install -y httpd
    systemctl start httpd
    systemctl enable httpd
    
    cat > /var/www/html/index.html << 'HTML'
    <!DOCTYPE html>
    <html>
    <head><title>${var.project_name} Web Server</title></head>
    <body>
      <h1>🚀 ${var.project_name} Web Server</h1>
      <p>Environment: ${var.environment}</p>
      <p>Region: ${var.aws_region}</p>
    </body>
    </html>
    HTML
    
    echo "Setup complete at $(date)" > /tmp/setup.log
  EOF
  )

  monitoring = true

  tags = {
    Name = "${var.project_name}-${var.environment}-web"
    Role = "web"
  }

  depends_on = [
    aws_iam_role_policy_attachment.ssm
  ]
}

# ─── Elastic IP ───────────────────────────────────────────

resource "aws_eip" "web" {
  domain     = "vpc"
  instance   = aws_instance.web.id
  depends_on = [aws_internet_gateway.main]

  tags = { Name = "${var.project_name}-web-eip" }
}

# ─── Outputs ──────────────────────────────────────────────

output "web_public_ip" {
  description = "Public IP ของ web server"
  value       = aws_eip.web.public_ip
}

output "web_url" {
  description = "URL ของ web server"
  value       = "http://${aws_eip.web.public_ip}"
}

output "instance_id" {
  description = "Instance ID"
  value       = aws_instance.web.id
}

output "private_ip" {
  description = "Private IP"
  value       = aws_instance.web.private_ip
}

# ─── Variables ────────────────────────────────────────────

variable "project_name" {
  description = "ชื่อโปรเจกต์"
  type        = string
  default     = "myapp"
}

variable "environment" {
  description = "Environment"
  type        = string
  default     = "development"
}

variable "aws_region" {
  description = "AWS Region"
  type        = string
  default     = "ap-southeast-1"
}
```

---

## สรุป: EC2 Best Practices

| หัวข้อ | Best Practice |
|--------|--------------|
| **AMI** | ใช้ data source หรือ SSM parameter |
| **Instance Type** | เริ่มด้วย t3.micro แล้วปรับตามการใช้งาน |
| **Security** | Security Group, IMDSv2, Encrypted EBS |
| **IAM** | Instance Profile พร้อม least-privilege |
| **SSH** | ใช้ SSM Session Manager แทน SSH |
| **User Data** | ใช้ templatefile() สำหรับ complex scripts |
| **Monitoring** | Enable detailed monitoring |
| **Tags** | ติด tags ทุก resource ทั้ง instance และ volume |

---

*จบ Part 037: AWS EC2 Instances*

*ต่อไป: Part 038 - AWS VPC & Networking*
