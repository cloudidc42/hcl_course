# Part 085: Encryption at Rest Misconfigurations
## ขั้นตอนที่ 841-850: การกำหนดค่า Encryption at Rest ที่ผิดพลาด

---

## ขั้นตอนที่ 841: ภาพรวม Encryption at Rest

### ทำไม Encryption at Rest จึงสำคัญ

```
Encryption at Rest ป้องกัน:
1. Physical theft of storage media
2. Unauthorized access to storage layer
3. Data breaches from storage misconfigurations
4. Compliance violations (GDPR, HIPAA, PCI-DSS)
5. Insider threats from storage administrators

ไม่ป้องกัน:
1. Authorized access (ต้อง use IAM + access controls)
2. In-transit attacks (ต้อง use TLS)
3. Memory attacks
```

### AWS Encryption Options

```
AWS Encryption at Rest Options:
┌─────────────────────────────────────────────────────┐
│  SSE-S3 / AWS-managed keys                          │
│  - AWS manages keys                                  │
│  - Automatic rotation                               │
│  - Less audit visibility                            │
│  - Good for: Non-sensitive data, cost optimization  │
├─────────────────────────────────────────────────────┤
│  SSE-KMS / Customer-managed keys (CMK)              │
│  - Customer controls key policy                     │
│  - CloudTrail audit trail                           │
│  - Key rotation control                             │
│  - Good for: Sensitive data, compliance             │
├─────────────────────────────────────────────────────┤
│  SSE-C / Customer-provided keys                     │
│  - Customer manages and provides keys               │
│  - AWS never stores key                             │
│  - Complex to manage                                │
│  - Good for: Maximum key control                    │
└─────────────────────────────────────────────────────┘
```

### KMS Key Rotation Best Practice

```hcl
# ✅ KMS Key with automatic rotation
resource "aws_kms_key" "secure" {
  description             = "Encryption key for sensitive data"
  deletion_window_in_days = 30
  
  # ✅ Always enable key rotation
  enable_key_rotation = true  # Rotates annually
  
  # ✅ Key policy
  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Sid    = "Enable Root Account"
        Effect = "Allow"
        Principal = {
          AWS = "arn:aws:iam::${var.account_id}:root"
        }
        Action   = "kms:*"
        Resource = "*"
      },
      {
        Sid    = "Allow Specific Services"
        Effect = "Allow"
        Principal = {
          Service = [
            "s3.amazonaws.com",
            "rds.amazonaws.com",
            "logs.amazonaws.com"
          ]
        }
        Action = [
          "kms:Decrypt",
          "kms:GenerateDataKey*",
          "kms:CreateGrant"
        ]
        Resource = "*"
        Condition = {
          StringEquals = {
            "kms:CallerAccount" = var.account_id
          }
        }
      }
    ]
  })
  
  tags = {
    Purpose = "data-encryption"
  }
}

resource "aws_kms_alias" "secure" {
  name          = "alias/secure-data-key"
  target_key_id = aws_kms_key.secure.key_id
}
```

---

## ขั้นตอนที่ 842: EC2 EBS Volume Encryption

### ❌ Vulnerable - Unencrypted EBS Volumes

```hcl
# ❌ VULNERABLE - Root volume not encrypted
resource "aws_instance" "unencrypted" {
  ami           = data.aws_ami.amazon_linux.id
  instance_type = "t3.micro"
  
  # ❌ Root volume without encryption
  root_block_device {
    volume_type = "gp3"
    volume_size = 20
    # ❌ encrypted = false (default)
  }
  
  # ❌ Additional volume not encrypted
  ebs_block_device {
    device_name = "/dev/sdb"
    volume_size = 100
    volume_type = "gp3"
    # ❌ No encryption
  }
}

# ❌ VULNERABLE - EBS Volume created separately
resource "aws_ebs_volume" "unencrypted" {
  availability_zone = "us-east-1a"
  size              = 100
  type              = "gp3"
  # ❌ encrypted = false (default)
}
```

### ✅ Secure - Encrypted EBS Volumes

```hcl
# ✅ Account-level default encryption
resource "aws_ebs_encryption_by_default" "enable" {
  enabled = true  # ✅ All new EBS volumes encrypted by default
}

resource "aws_ebs_default_kms_key" "custom" {
  key_arn = aws_kms_key.ebs.arn  # ✅ Customer managed key
  
  depends_on = [aws_ebs_encryption_by_default.enable]
}

# ✅ EC2 with encrypted volumes
resource "aws_instance" "encrypted" {
  ami           = data.aws_ami.amazon_linux.id
  instance_type = "t3.micro"
  
  # ✅ Encrypted root volume
  root_block_device {
    volume_type = "gp3"
    volume_size = 20
    encrypted   = true             # ✅ Encrypt
    kms_key_id  = aws_kms_key.ebs.arn  # ✅ Customer key
  }
  
  # ✅ Encrypted additional volume
  ebs_block_device {
    device_name = "/dev/sdb"
    volume_size = 100
    volume_type = "gp3"
    encrypted   = true             # ✅ Encrypt
    kms_key_id  = aws_kms_key.ebs.arn  # ✅ Customer key
  }
  
  tags = {
    Name = "encrypted-server"
  }
}

# ✅ KMS key for EBS
resource "aws_kms_key" "ebs" {
  description             = "EBS volume encryption key"
  deletion_window_in_days = 30
  enable_key_rotation     = true
  
  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Sid    = "Enable Root"
        Effect = "Allow"
        Principal = { AWS = "arn:aws:iam::${var.account_id}:root" }
        Action   = "kms:*"
        Resource = "*"
      },
      {
        Sid    = "Allow EC2"
        Effect = "Allow"
        Principal = { Service = "ec2.amazonaws.com" }
        Action = [
          "kms:Decrypt",
          "kms:GenerateDataKey*",
          "kms:CreateGrant",
          "kms:DescribeKey"
        ]
        Resource = "*"
      }
    ]
  })
}

# ✅ Verify encryption with AWS Config
resource "aws_config_config_rule" "encrypted_volumes" {
  name        = "encrypted-volumes"
  description = "Checks whether EBS volumes are encrypted"
  
  source {
    owner             = "AWS"
    source_identifier = "ENCRYPTED_VOLUMES"
  }
}
```

---

## ขั้นตอนที่ 843: RDS Instance Encryption

### ❌ Vulnerable - Unencrypted RDS

```hcl
# ❌ VULNERABLE - RDS without encryption
resource "aws_db_instance" "unencrypted_rds" {
  identifier     = "production-db"
  engine         = "mysql"
  engine_version = "8.0"
  instance_class = "db.t3.micro"
  
  allocated_storage = 20
  # ❌ storage_encrypted = false (default)
  
  username = "admin"
  password = var.db_password
  
  # ❌ ไม่มี encryption = ข้อมูลใน disk ไม่ถูกปกป้อง
  # ❌ Snapshots ก็ไม่ encrypted
}

# ⚠️ IMPORTANT: ไม่สามารถ enable encryption บน existing instance
# ต้อง: Create snapshot → Copy encrypted → Restore
```

### ✅ Secure - Encrypted RDS

```hcl
# ✅ KMS Key for RDS
resource "aws_kms_key" "rds" {
  description             = "RDS encryption key"
  deletion_window_in_days = 30
  enable_key_rotation     = true
  
  policy = jsonencode({
    Statement = [
      {
        Sid    = "Enable Root"
        Effect = "Allow"
        Principal = { AWS = "arn:aws:iam::${var.account_id}:root" }
        Action   = "kms:*"
        Resource = "*"
      },
      {
        Sid    = "Allow RDS"
        Effect = "Allow"
        Principal = { Service = "rds.amazonaws.com" }
        Action = [
          "kms:Decrypt",
          "kms:GenerateDataKey*",
          "kms:CreateGrant",
          "kms:DescribeKey"
        ]
        Resource = "*"
      }
    ]
  })
}

# ✅ Encrypted RDS instance
resource "aws_db_instance" "encrypted_rds" {
  identifier     = "production-db"
  engine         = "mysql"
  engine_version = "8.0"
  instance_class = "db.t3.micro"
  
  allocated_storage     = 20
  storage_type          = "gp3"
  storage_encrypted     = true              # ✅ Enable encryption
  kms_key_id            = aws_kms_key.rds.arn  # ✅ Customer key
  
  db_subnet_group_name   = aws_db_subnet_group.isolated.name
  vpc_security_group_ids = [aws_security_group.rds.id]
  
  username = "admin"
  password = random_password.rds.result    # ✅ Random password
  
  # ✅ Encryption applies to:
  # - Database storage
  # - Automated backups
  # - Read replicas
  # - Snapshots
  
  backup_retention_period = 7
  backup_window           = "03:00-04:00"
  maintenance_window      = "sun:04:00-sun:05:00"
  
  deletion_protection = true   # ✅ Prevent accidental deletion
  
  # ✅ Enhanced monitoring
  monitoring_interval = 60
  
  # ✅ CloudWatch logs
  enabled_cloudwatch_logs_exports = ["general", "error", "slowquery"]
  
  tags = {
    Environment = var.environment
  }
}

# ✅ Encrypted Read Replica
resource "aws_db_instance" "encrypted_replica" {
  identifier          = "production-db-replica"
  replicate_source_db = aws_db_instance.encrypted_rds.identifier
  instance_class      = "db.t3.micro"
  
  # ✅ Replica inherits encryption from source
  # But can specify different KMS key
  kms_key_id = aws_kms_key.rds.arn
  
  storage_encrypted = true
  
  tags = {
    Environment = var.environment
    Role        = "replica"
  }
}

# ✅ Parameter group with SSL required
resource "aws_db_parameter_group" "mysql_ssl" {
  name   = "mysql-ssl-required"
  family = "mysql8.0"
  
  parameter {
    name  = "require_secure_transport"
    value = "ON"   # ✅ Require SSL connections
  }
}
```

---

## ขั้นตอนที่ 844: DynamoDB Encryption

### ❌ Vulnerable - Default DynamoDB Encryption

```hcl
# ❌ VULNERABLE - DynamoDB with default encryption (SSE-owned by AWS)
resource "aws_dynamodb_table" "default_encryption" {
  name         = "users-table"
  billing_mode = "PAY_PER_REQUEST"
  hash_key     = "user_id"
  
  attribute {
    name = "user_id"
    type = "S"
  }
  
  # ❌ ไม่มี server_side_encryption block
  # AWS เปิด encryption ให้อัตโนมัติ แต่ใช้ AWS-owned key
  # ไม่สามารถ audit ได้ผ่าน CloudTrail
  # ไม่สามารถ revoke access ได้
}
```

### ✅ Secure - DynamoDB with KMS

```hcl
# ✅ SECURE - DynamoDB with customer managed KMS key
resource "aws_kms_key" "dynamodb" {
  description             = "DynamoDB encryption key"
  deletion_window_in_days = 30
  enable_key_rotation     = true
}

resource "aws_dynamodb_table" "encrypted" {
  name         = "users-table"
  billing_mode = "PAY_PER_REQUEST"
  hash_key     = "user_id"
  range_key    = "created_at"
  
  attribute {
    name = "user_id"
    type = "S"
  }
  
  attribute {
    name = "created_at"
    type = "N"
  }
  
  # ✅ Customer managed KMS key
  server_side_encryption {
    enabled     = true
    kms_key_arn = aws_kms_key.dynamodb.arn  # ✅ CMK
  }
  
  # ✅ Point-in-time recovery
  point_in_time_recovery {
    enabled = true  # ✅ 35-day recovery window
  }
  
  # ✅ TTL for data lifecycle
  ttl {
    attribute_name = "expiry_time"
    enabled        = true
  }
  
  tags = {
    Environment = var.environment
    DataClass   = "sensitive"
  }
}
```

---

## ขั้นตอนที่ 845: ElastiCache & EFS Encryption

### ElastiCache Encryption at Rest

```hcl
# ❌ VULNERABLE - ElastiCache without encryption
resource "aws_elasticache_replication_group" "no_encryption" {
  replication_group_id = "my-redis"
  description          = "Redis cluster"
  
  node_type            = "cache.t3.micro"
  engine               = "redis"
  
  # ❌ ไม่มี at_rest_encryption_enabled
  # ❌ ไม่มี transit_encryption_enabled
  # ❌ ข้อมูลใน disk และ in-transit ไม่ encrypted
}

# ✅ SECURE - ElastiCache with full encryption
resource "aws_kms_key" "elasticache" {
  description             = "ElastiCache encryption key"
  deletion_window_in_days = 30
  enable_key_rotation     = true
}

resource "aws_elasticache_replication_group" "encrypted" {
  replication_group_id       = "my-redis-secure"
  description                = "Encrypted Redis cluster"
  
  node_type                  = "cache.t3.micro"
  engine                     = "redis"
  engine_version             = "7.0"
  num_cache_clusters         = 2    # ✅ Multi-AZ
  
  # ✅ Encryption at rest
  at_rest_encryption_enabled = true
  kms_key_id                 = aws_kms_key.elasticache.arn
  
  # ✅ Encryption in transit
  transit_encryption_enabled = true
  auth_token                 = random_password.redis_auth.result  # ✅ AUTH token
  
  # ✅ Security group
  security_group_ids = [aws_security_group.elasticache.id]
  subnet_group_name  = aws_elasticache_subnet_group.isolated.name
  
  # ✅ Auto failover
  automatic_failover_enabled = true
  multi_az_enabled           = true
  
  # ✅ Snapshot
  snapshot_retention_limit   = 7
  snapshot_window            = "03:00-04:00"
  
  tags = {
    Environment = var.environment
  }
}

# ✅ SECURE - ElastiCache Memcached
resource "aws_elasticache_cluster" "memcached_encrypted" {
  cluster_id           = "my-memcached"
  engine               = "memcached"
  node_type            = "cache.t3.micro"
  num_cache_nodes      = 2
  parameter_group_name = "default.memcached1.6"
  
  # Note: Memcached doesn't support at-rest encryption
  # Use Redis if encryption is required
  
  security_group_ids = [aws_security_group.elasticache.id]
  subnet_group_name  = aws_elasticache_subnet_group.isolated.name
}
```

### EFS File System Encryption

```hcl
# ❌ VULNERABLE - EFS without encryption
resource "aws_efs_file_system" "no_encryption" {
  # ❌ encrypted = false (default on some older configs)
  # ❌ ไม่มี kms_key_id
  
  tags = {
    Name = "app-storage"
  }
}

# ✅ SECURE - EFS with KMS encryption
resource "aws_kms_key" "efs" {
  description             = "EFS encryption key"
  deletion_window_in_days = 30
  enable_key_rotation     = true
}

resource "aws_efs_file_system" "encrypted" {
  encrypted  = true              # ✅ Enable encryption
  kms_key_id = aws_kms_key.efs.arn  # ✅ Customer key
  
  # ✅ Performance mode
  performance_mode = "generalPurpose"
  throughput_mode  = "bursting"
  
  # ✅ Lifecycle policy
  lifecycle_policy {
    transition_to_ia = "AFTER_30_DAYS"
  }
  
  lifecycle_policy {
    transition_to_primary_storage_class = "AFTER_1_ACCESS"
  }
  
  tags = {
    Name        = "app-storage"
    Environment = var.environment
    Encrypted   = "true"
  }
}

# ✅ Mount Target with security group
resource "aws_efs_mount_target" "app" {
  count = length(var.availability_zones)
  
  file_system_id  = aws_efs_file_system.encrypted.id
  subnet_id       = aws_subnet.private[count.index].id
  security_groups = [aws_security_group.efs.id]
}

# ✅ EFS Access Policy
resource "aws_efs_file_system_policy" "strict" {
  file_system_id = aws_efs_file_system.encrypted.id
  
  policy = jsonencode({
    Statement = [
      {
        Sid    = "DenyNonTLS"
        Effect = "Deny"
        Principal = { AWS = "*" }
        Action   = "*"
        Condition = {
          Bool = {
            "aws:SecureTransport" = "false"  # ✅ Require TLS
          }
        }
      },
      {
        Sid    = "AllowSpecificRoles"
        Effect = "Allow"
        Principal = {
          AWS = [
            aws_iam_role.app_role.arn
          ]
        }
        Action = [
          "elasticfilesystem:ClientMount",
          "elasticfilesystem:ClientWrite",
          "elasticfilesystem:ClientRootAccess"
        ]
        Condition = {
          Bool = {
            "aws:SecureTransport"  = "true"
            "elasticfilesystem:AccessedViaMountTarget" = "true"
          }
        }
      }
    ]
  })
}
```

---

## ขั้นตอนที่ 846: Secrets Manager & CloudTrail Encryption

### Secrets Manager KMS Encryption

```hcl
# ❌ VULNERABLE - Secrets Manager using default key
resource "aws_secretsmanager_secret" "db_password_weak" {
  name = "prod/db/password"
  # ❌ ไม่มี kms_key_id = ใช้ AWS managed key
  # ❌ ไม่สามารถ revoke access แบบ granular
  # ❌ ไม่มี audit trail แบบ CMK
}

# ✅ SECURE - Secrets Manager with CMK
resource "aws_kms_key" "secrets" {
  description             = "Secrets Manager encryption key"
  deletion_window_in_days = 30
  enable_key_rotation     = true
  
  policy = jsonencode({
    Statement = [
      {
        Sid    = "Enable Root"
        Effect = "Allow"
        Principal = { AWS = "arn:aws:iam::${var.account_id}:root" }
        Action   = "kms:*"
        Resource = "*"
      },
      {
        Sid    = "Allow Secrets Manager"
        Effect = "Allow"
        Principal = { Service = "secretsmanager.amazonaws.com" }
        Action = [
          "kms:Decrypt",
          "kms:GenerateDataKey",
          "kms:CreateGrant",
          "kms:DescribeKey"
        ]
        Resource = "*"
      }
    ]
  })
}

resource "aws_secretsmanager_secret" "db_password" {
  name       = "prod/db/password"
  kms_key_id = aws_kms_key.secrets.arn  # ✅ CMK

  # ✅ Enable rotation
  rotation_rules {
    automatically_after_days = 30  # ✅ Auto-rotate every 30 days
  }
  
  # ✅ Recovery window
  recovery_window_in_days = 30  # ✅ 30 day recovery period before delete
  
  tags = {
    Environment = var.environment
    DataClass   = "confidential"
  }
}

resource "aws_secretsmanager_secret_version" "db_password" {
  secret_id = aws_secretsmanager_secret.db_password.id
  
  secret_string = jsonencode({
    username = "app_user"
    password = random_password.db.result
    host     = aws_db_instance.main.endpoint
    port     = 3306
    dbname   = "application_db"
  })
}
```

### CloudTrail Log Encryption

```hcl
# ❌ VULNERABLE - CloudTrail without KMS
resource "aws_cloudtrail" "no_kms" {
  name           = "audit-trail"
  s3_bucket_name = aws_s3_bucket.cloudtrail.id
  # ❌ ไม่มี kms_key_id = logs ไม่ encrypted with CMK
  # ❌ ไม่มี enable_log_file_validation
  # ❌ ไม่มี cloud_watch_logs_group_arn
}

# ✅ SECURE - CloudTrail with full security
resource "aws_kms_key" "cloudtrail" {
  description             = "CloudTrail encryption key"
  deletion_window_in_days = 30
  enable_key_rotation     = true
  
  policy = jsonencode({
    Statement = [
      {
        Sid    = "Enable Root"
        Effect = "Allow"
        Principal = { AWS = "arn:aws:iam::${var.account_id}:root" }
        Action   = "kms:*"
        Resource = "*"
      },
      {
        Sid    = "Allow CloudTrail"
        Effect = "Allow"
        Principal = { Service = "cloudtrail.amazonaws.com" }
        Action = [
          "kms:Decrypt",
          "kms:GenerateDataKey*",
          "kms:CreateGrant",
          "kms:DescribeKey"
        ]
        Resource = "*"
        Condition = {
          StringLike = {
            "kms:EncryptionContext:aws:cloudtrail:arn" = "arn:aws:cloudtrail:*:${var.account_id}:trail/*"
          }
        }
      }
    ]
  })
}

resource "aws_cloudtrail" "secure" {
  name                          = "main-audit-trail"
  s3_bucket_name               = aws_s3_bucket.cloudtrail.id
  
  # ✅ KMS encryption
  kms_key_id = aws_kms_key.cloudtrail.arn
  
  # ✅ Global events
  include_global_service_events = true
  is_multi_region_trail         = true
  
  # ✅ Log file validation (integrity check)
  enable_log_file_validation = true
  
  # ✅ CloudWatch integration
  cloud_watch_logs_group_arn = "${aws_cloudwatch_log_group.cloudtrail.arn}:*"
  cloud_watch_logs_role_arn  = aws_iam_role.cloudtrail.arn
  
  event_selector {
    read_write_type           = "All"
    include_management_events = true
    
    data_resource {
      type   = "AWS::S3::Object"
      values = ["arn:aws:s3:::"]
    }
    
    data_resource {
      type   = "AWS::Lambda::Function"
      values = ["arn:aws:lambda"]
    }
  }
  
  tags = {
    Name    = "main-audit-trail"
    Purpose = "Compliance Audit"
  }
}
```

---

## ขั้นตอนที่ 847: SNS & SQS Encryption

### SNS Encryption

```hcl
# ❌ VULNERABLE - SNS without KMS
resource "aws_sns_topic" "no_encryption" {
  name = "payment-notifications"
  # ❌ ไม่มี kms_master_key_id
  # ❌ Messages ไม่ encrypted at rest
}

# ✅ SECURE - SNS with KMS
resource "aws_kms_key" "sns" {
  description             = "SNS encryption key"
  deletion_window_in_days = 30
  enable_key_rotation     = true
  
  policy = jsonencode({
    Statement = [
      {
        Sid    = "Enable Root"
        Effect = "Allow"
        Principal = { AWS = "arn:aws:iam::${var.account_id}:root" }
        Action   = "kms:*"
        Resource = "*"
      },
      {
        Sid    = "Allow SNS"
        Effect = "Allow"
        Principal = { Service = "sns.amazonaws.com" }
        Action = [
          "kms:Decrypt",
          "kms:GenerateDataKey*",
          "kms:CreateGrant",
          "kms:DescribeKey"
        ]
        Resource = "*"
      },
      # Allow CloudWatch to publish alerts
      {
        Sid    = "Allow CloudWatch"
        Effect = "Allow"
        Principal = { Service = "cloudwatch.amazonaws.com" }
        Action = [
          "kms:Decrypt",
          "kms:GenerateDataKey*"
        ]
        Resource = "*"
      }
    ]
  })
}

resource "aws_sns_topic" "encrypted" {
  name              = "payment-notifications"
  kms_master_key_id = aws_kms_key.sns.arn  # ✅ Customer key
  
  # ✅ FIFO topic for ordered delivery (if needed)
  # fifo_topic = true
  
  tags = {
    Environment = var.environment
    DataClass   = "sensitive"
  }
}

# ✅ SNS Topic Policy
resource "aws_sns_topic_policy" "encrypted" {
  arn = aws_sns_topic.encrypted.arn
  
  policy = jsonencode({
    Statement = [
      {
        Sid    = "DenyNonSSL"
        Effect = "Deny"
        Principal = { AWS = "*" }
        Action   = "sns:Publish"
        Condition = {
          Bool = {
            "aws:SecureTransport" = "false"
          }
        }
      },
      {
        Sid    = "AllowSpecificPublishers"
        Effect = "Allow"
        Principal = {
          AWS = [aws_iam_role.app_role.arn]
        }
        Action = ["sns:Publish"]
      }
    ]
  })
}
```

### SQS Queue Encryption

```hcl
# ❌ VULNERABLE - SQS without encryption
resource "aws_sqs_queue" "no_encryption" {
  name = "payment-processing"
  # ❌ ไม่มี kms_master_key_id
  # ❌ Messages ไม่ encrypted
}

# ✅ SECURE - SQS with KMS
resource "aws_kms_key" "sqs" {
  description             = "SQS encryption key"
  deletion_window_in_days = 30
  enable_key_rotation     = true
}

resource "aws_sqs_queue" "encrypted" {
  name                       = "payment-processing"
  kms_master_key_id         = aws_kms_key.sqs.arn  # ✅ Customer key
  kms_data_key_reuse_period_seconds = 300  # ✅ Reuse data key 5 min
  
  # ✅ Message retention
  message_retention_seconds = 86400  # 1 day
  
  # ✅ Visibility timeout
  visibility_timeout_seconds = 30
  
  # ✅ DLQ for failed messages
  redrive_policy = jsonencode({
    deadLetterTargetArn = aws_sqs_queue.dead_letter.arn
    maxReceiveCount     = 3
  })
  
  tags = {
    Environment = var.environment
    DataClass   = "sensitive"
  }
}

# ✅ Dead Letter Queue (also encrypted)
resource "aws_sqs_queue" "dead_letter" {
  name              = "payment-processing-dlq"
  kms_master_key_id = aws_kms_key.sqs.arn
  
  message_retention_seconds = 1209600  # 14 days
}

# ✅ SQS Queue Policy
resource "aws_sqs_queue_policy" "encrypted" {
  queue_url = aws_sqs_queue.encrypted.id
  
  policy = jsonencode({
    Statement = [
      {
        Sid    = "DenyNonSSL"
        Effect = "Deny"
        Principal = { AWS = "*" }
        Action   = "sqs:*"
        Condition = {
          Bool = {
            "aws:SecureTransport" = "false"
          }
        }
      },
      {
        Sid    = "AllowSNS"
        Effect = "Allow"
        Principal = { Service = "sns.amazonaws.com" }
        Action   = "sqs:SendMessage"
        Condition = {
          ArnEquals = {
            "aws:SourceArn" = aws_sns_topic.encrypted.arn
          }
        }
      }
    ]
  })
}
```

---

## ขั้นตอนที่ 848: EKS Secrets Encryption

### ❌ Vulnerable - EKS Without Secrets Encryption

```hcl
# ❌ VULNERABLE - EKS without secrets encryption
resource "aws_eks_cluster" "no_secrets_encryption" {
  name     = "production-cluster"
  role_arn = aws_iam_role.eks.arn
  
  vpc_config {
    subnet_ids = aws_subnet.private[*].id
  }
  
  # ❌ ไม่มี encryption_config
  # ❌ Kubernetes Secrets ไม่ encrypted at rest
  # ❌ Secret values stored as base64 (not encrypted!) in etcd
}
```

### ✅ Secure - EKS with Full Encryption

```hcl
# ✅ KMS Key for EKS
resource "aws_kms_key" "eks" {
  description             = "EKS cluster encryption key"
  deletion_window_in_days = 30
  enable_key_rotation     = true
}

resource "aws_eks_cluster" "secure" {
  name     = "production-cluster"
  role_arn = aws_iam_role.eks.arn
  version  = "1.28"
  
  vpc_config {
    subnet_ids              = aws_subnet.private[*].id
    endpoint_private_access = true   # ✅ Private endpoint
    endpoint_public_access  = false  # ✅ No public endpoint
    
    security_group_ids = [aws_security_group.eks_cluster.id]
  }
  
  # ✅ Secrets encryption
  encryption_config {
    resources = ["secrets"]  # ✅ Encrypt Kubernetes secrets
    
    provider {
      key_arn = aws_kms_key.eks.arn  # ✅ CMK
    }
  }
  
  # ✅ Enable all logging
  enabled_cluster_log_types = [
    "api",
    "audit",          # ✅ Audit logs
    "authenticator",
    "controllerManager",
    "scheduler"
  ]
  
  tags = {
    Environment = var.environment
  }
  
  depends_on = [
    aws_iam_role_policy_attachment.eks_cluster_policy,
    aws_iam_role_policy_attachment.eks_vpc_resource_controller
  ]
}
```

---

## ขั้นตอนที่ 849: KMS Best Practices

### KMS Key Rotation & Monitoring

```hcl
# ❌ VULNERABLE - KMS key without rotation
resource "aws_kms_key" "no_rotation" {
  description = "Encryption key"
  # ❌ enable_key_rotation = false (default)
  # ❌ Key never rotates = risk if key material compromised
}

# ✅ SECURE - KMS with all best practices
resource "aws_kms_key" "best_practice" {
  description             = "Secure encryption key"
  deletion_window_in_days = 30  # ✅ Minimum 7, max 30 days
  
  # ✅ Auto rotation annually
  enable_key_rotation = true
  
  # ✅ Cannot delete without waiting
  # (deletion_window_in_days provides protection)
  
  tags = {
    Environment = var.environment
    Purpose     = "data-encryption"
    Rotation    = "enabled"
  }
}

# ✅ Monitor KMS key usage
resource "aws_cloudwatch_metric_alarm" "kms_key_deletion" {
  alarm_name          = "KMSKeyDeletion"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = 1
  metric_name         = "KMSKeyDeletion"
  namespace           = "SecurityMetrics"
  period              = 300
  statistic           = "Sum"
  threshold           = 0
  
  alarm_description = "KMS key scheduled for deletion detected"
  alarm_actions     = [aws_sns_topic.security_alerts.arn]
}

resource "aws_cloudwatch_log_metric_filter" "kms_key_deletion" {
  name           = "KMSKeyDeletion"
  pattern        = "{$.eventSource = \"kms.amazonaws.com\" && ($.eventName = \"DisableKey\" || $.eventName = \"ScheduleKeyDeletion\")}"
  log_group_name = aws_cloudwatch_log_group.cloudtrail.name
  
  metric_transformation {
    name      = "KMSKeyDeletion"
    namespace = "SecurityMetrics"
    value     = "1"
  }
}

# ✅ KMS Key Grants (instead of key policy changes)
resource "aws_kms_grant" "lambda_decrypt" {
  name              = "lambda-decrypt-grant"
  key_id            = aws_kms_key.best_practice.key_id
  grantee_principal = aws_iam_role.lambda.arn
  
  operations = [
    "Decrypt",
    "GenerateDataKey",
    "DescribeKey"
  ]
  
  # ✅ Grant expires (if needed)
  constraints {
    encryption_context_equals = {
      Environment = "production"
    }
  }
}
```

---

## ขั้นตอนที่ 850: Complete Encryption Compliance Check

### AWS Config Rules for Encryption

```hcl
# ✅ Comprehensive encryption compliance

locals {
  encryption_config_rules = {
    "encrypted-volumes"                    = "ENCRYPTED_VOLUMES"
    "rds-storage-encrypted"               = "RDS_STORAGE_ENCRYPTED"
    "s3-bucket-server-side-encryption-enabled" = "S3_BUCKET_SERVER_SIDE_ENCRYPTION_ENABLED"
    "dynamodb-table-encrypted-at-rest"    = "DYNAMODB_TABLE_ENCRYPTED_AT_REST"
    "efs-encrypted-check"                 = "EFS_ENCRYPTED_CHECK"
    "elasticsearch-encrypted-at-rest"     = "ELASTICSEARCH_ENCRYPTED_AT_REST"
    "elasticache-repl-grp-encrypted-at-rest" = "ELASTICACHE_REPL_GRP_ENCRYPTED_AT_REST"
    "sqs-queue-server-side-encryption-enabled" = "SQS_QUEUE_ENCRYPTION_CHECK"
  }
}

resource "aws_config_config_rule" "encryption" {
  for_each = local.encryption_config_rules
  
  name = each.key
  
  source {
    owner             = "AWS"
    source_identifier = each.value
  }
  
  depends_on = [aws_config_configuration_recorder.main]
}

# ✅ Security Hub - CIS Benchmark
resource "aws_securityhub_standards_subscription" "cis" {
  standards_arn = "arn:aws:securityhub:${var.region}::standards/cis-aws-foundations-benchmark/v/1.4.0"
}

# ✅ Encryption summary dashboard
resource "aws_cloudwatch_dashboard" "encryption" {
  dashboard_name = "EncryptionCompliance"
  
  dashboard_body = jsonencode({
    widgets = [
      {
        type = "metric"
        properties = {
          metrics = [
            ["AWS/Config", "CompliancePercentage", "ConfigRuleName", "encrypted-volumes"],
            ["AWS/Config", "CompliancePercentage", "ConfigRuleName", "rds-storage-encrypted"],
            ["AWS/Config", "CompliancePercentage", "ConfigRuleName", "s3-bucket-server-side-encryption-enabled"]
          ]
          period = 3600
          title  = "Encryption Compliance"
          view   = "timeSeries"
          stat   = "Average"
        }
        width  = 24
        height = 6
        x      = 0
        y      = 0
      }
    ]
  })
}
```

### Encryption Coverage Summary

```hcl
# Terraform output - encryption status
output "encryption_summary" {
  description = "Summary of encryption status"
  value = {
    ebs_encryption_by_default = aws_ebs_encryption_by_default.enable.enabled
    rds_encrypted             = aws_db_instance.encrypted_rds.storage_encrypted
    s3_encrypted              = "aws:kms"
    dynamodb_encrypted        = true
    elasticache_encrypted     = aws_elasticache_replication_group.encrypted.at_rest_encryption_enabled
    efs_encrypted             = aws_efs_file_system.encrypted.encrypted
    secrets_manager_key       = aws_kms_key.secrets.arn
    cloudtrail_key            = aws_kms_key.cloudtrail.arn
    sns_key                   = aws_kms_key.sns.arn
    sqs_key                   = aws_kms_key.sqs.arn
    eks_secrets_encrypted     = true
  }
}
```

---

## สรุป Encryption at Rest

### ตารางการ encrypt แต่ละ service

| Service | Default | Recommended | CIS Control | Checkov Rule |
|---------|---------|-------------|-------------|-------------|
| EBS Volumes | None | aws:kms CMK | - | CKV_AWS_8 |
| RDS | None | aws:kms CMK | - | CKV_AWS_17 |
| S3 Buckets | None | aws:kms CMK | 2.1.1 | CKV_AWS_19 |
| DynamoDB | AWS-owned | CMK | - | CKV_AWS_28 |
| ElastiCache | None | CMK | - | CKV_AWS_29 |
| EFS | None | CMK | - | CKV_AWS_42 |
| Secrets Manager | AWS-managed | CMK | - | CKV_AWS_149 |
| CloudTrail | None | CMK | 3.7 | CKV_AWS_36 |
| SNS | None | CMK | - | CKV_AWS_26 |
| SQS | None | CMK | - | CKV_AWS_27 |
| EKS Secrets | None | CMK | - | CKV_AWS_58 |
| KMS Keys | - | Rotation=true | - | CKV_AWS_7 |

---

*Part 085 ครอบคลุม Encryption at Rest ทั้งหมด - ต่อไปใน Part 086 จะเจาะลึก Encryption in Transit*
