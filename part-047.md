# Part 047: AWS ElastiCache & Database Caching
## การใช้งาน Caching Layer ด้วย Terraform (Steps 461-470)

---

## บทนำ (Introduction)

AWS ElastiCache เป็น managed in-memory caching service ที่รองรับทั้ง Redis และ Memcached
การใช้ caching ช่วยลด latency, ลด database load, และเพิ่ม performance ของ applications

**หัวข้อที่จะเรียนรู้:**
- Redis Replication Group (High Availability)
- Memcached Cluster
- Subnet Groups และ Parameter Groups
- Encryption at Rest และ In Transit
- Auth Token สำหรับ Redis
- ElastiCache Users และ User Groups
- CloudWatch Metrics
- Security Best Practices

---

## Step 461: ElastiCache Subnet Group และ Security Groups

```hcl
# ✅ ElastiCache Subnet Group (private subnets)
resource "aws_elasticache_subnet_group" "main" {
  name        = "${var.project_name}-cache-subnet-group"
  description = "ElastiCache subnet group for ${var.project_name}"

  subnet_ids = [
    aws_subnet.private_1a.id,
    aws_subnet.private_1b.id,
    aws_subnet.private_1c.id,
  ]

  tags = {
    Name      = "${var.project_name}-cache-subnet-group"
    ManagedBy = "terraform"
  }
}

# ✅ Security Group สำหรับ ElastiCache Redis
resource "aws_security_group" "redis" {
  name        = "${var.project_name}-redis-sg"
  description = "Security group for ElastiCache Redis"
  vpc_id      = aws_vpc.main.id

  # ✅ อนุญาตเฉพาะ application servers
  ingress {
    description     = "Redis from application servers"
    from_port       = 6379
    to_port         = 6379
    protocol        = "tcp"
    security_groups = [aws_security_group.app.id]
  }

  # ✅ TLS Redis port
  ingress {
    description     = "Redis TLS from application servers"
    from_port       = 6380
    to_port         = 6380
    protocol        = "tcp"
    security_groups = [aws_security_group.app.id]
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = {
    Name      = "${var.project_name}-redis-sg"
    ManagedBy = "terraform"
  }
}

# ✅ Security Group สำหรับ Memcached
resource "aws_security_group" "memcached" {
  name        = "${var.project_name}-memcached-sg"
  description = "Security group for ElastiCache Memcached"
  vpc_id      = aws_vpc.main.id

  ingress {
    description     = "Memcached from application servers"
    from_port       = 11211
    to_port         = 11211
    protocol        = "tcp"
    security_groups = [aws_security_group.app.id]
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = {
    Name      = "${var.project_name}-memcached-sg"
    ManagedBy = "terraform"
  }
}
```

---

## Step 462: Redis Replication Group

### aws_elasticache_replication_group

```hcl
# ✅ Random Auth Token สำหรับ Redis
resource "random_password" "redis_auth_token" {
  length           = 64
  special          = false  # Redis auth token ไม่อนุญาต special chars บางตัว
  override_special = "!#$%&*()-_=+[]{}<>:"
  min_lower        = 10
  min_upper        = 10
  min_numeric      = 10
}

# ✅ เก็บ Auth Token ใน Secrets Manager
resource "aws_secretsmanager_secret" "redis_auth_token" {
  name                    = "${var.project_name}/redis/auth-token"
  description             = "Redis auth token for ${var.project_name}"
  recovery_window_in_days = 7
  kms_key_id              = aws_kms_key.secrets.arn

  tags = {
    Name      = "${var.project_name}-redis-auth-token"
    ManagedBy = "terraform"
  }
}

resource "aws_secretsmanager_secret_version" "redis_auth_token" {
  secret_id     = aws_secretsmanager_secret.redis_auth_token.id
  secret_string = random_password.redis_auth_token.result
}

# ✅ Production Redis Replication Group
resource "aws_elasticache_replication_group" "redis" {
  replication_group_id = "${var.project_name}-redis"
  description          = "Redis cluster for ${var.project_name}"

  # Engine
  engine               = "redis"
  engine_version       = "7.1"
  node_type            = "cache.r6g.large"

  # Cluster settings
  num_cache_clusters         = 2  # 1 primary + 1 replica
  automatic_failover_enabled = true  # ✅ ต้องมีอย่างน้อย 2 nodes
  multi_az_enabled           = true  # ✅ Multi-AZ

  # ✅ Encryption
  at_rest_encryption_enabled = true   # ✅ Encrypt at rest
  transit_encryption_enabled = true   # ✅ Encrypt in transit
  kms_key_id                 = aws_kms_key.elasticache.arn

  # ✅ Auth Token
  auth_token                    = random_password.redis_auth_token.result
  auth_token_update_strategy    = "ROTATE"  # สำหรับ rotation

  # Network
  subnet_group_name  = aws_elasticache_subnet_group.main.name
  security_group_ids = [aws_security_group.redis.id]

  # ✅ Port (TLS port)
  port = 6379

  # Parameter group
  parameter_group_name = aws_elasticache_parameter_group.redis7.name

  # Maintenance
  maintenance_window = "sun:05:00-sun:06:00"
  snapshot_window    = "04:00-05:00"

  # ✅ Backup
  snapshot_retention_limit = 7  # 7 วัน

  # Apply changes immediately (ระวัง: อาจทำให้ downtime)
  apply_immediately = false

  # Auto minor version upgrade
  auto_minor_version_upgrade = true

  # ✅ Notification
  notification_topic_arn = aws_sns_topic.elasticache_alerts.arn

  tags = {
    Name        = "${var.project_name}-redis"
    Environment = var.environment
    ManagedBy   = "terraform"
  }
}

# ❌ Insecure Redis - อย่าทำแบบนี้!
resource "aws_elasticache_replication_group" "insecure_redis" {
  replication_group_id = "insecure-redis"
  description          = "Insecure Redis"
  node_type            = "cache.t3.micro"
  num_cache_clusters   = 1

  # ❌ ไม่มี encryption
  at_rest_encryption_enabled = false
  transit_encryption_enabled = false

  # ❌ ไม่มี auth token
  # ❌ ไม่มี backup
  snapshot_retention_limit = 0

  # ❌ ไม่มี Multi-AZ
  automatic_failover_enabled = false
}
```

### Redis Cluster Mode

```hcl
# ✅ Redis Cluster Mode Enabled (ขนาดใหญ่, sharding)
resource "aws_elasticache_replication_group" "redis_cluster" {
  replication_group_id = "${var.project_name}-redis-cluster"
  description          = "Redis cluster with sharding"

  engine         = "redis"
  engine_version = "7.1"
  node_type      = "cache.r6g.xlarge"

  # ✅ Cluster mode: num_node_groups = number of shards
  num_node_groups         = 3  # 3 shards
  replicas_per_node_group = 2  # 2 replicas per shard

  automatic_failover_enabled = true
  multi_az_enabled           = true

  at_rest_encryption_enabled = true
  transit_encryption_enabled = true
  kms_key_id                 = aws_kms_key.elasticache.arn
  auth_token                 = random_password.redis_auth_token.result

  subnet_group_name  = aws_elasticache_subnet_group.main.name
  security_group_ids = [aws_security_group.redis.id]

  parameter_group_name     = aws_elasticache_parameter_group.redis7_cluster.name
  snapshot_retention_limit = 7
  maintenance_window       = "sun:05:00-sun:06:00"

  tags = {
    Name      = "${var.project_name}-redis-cluster"
    ManagedBy = "terraform"
  }
}
```

---

## Step 463: Memcached Cluster

### aws_elasticache_cluster (Memcached)

```hcl
# ✅ Memcached Cluster
resource "aws_elasticache_cluster" "memcached" {
  cluster_id           = "${var.project_name}-memcached"
  engine               = "memcached"
  engine_version       = "1.6.22"
  node_type            = "cache.t3.medium"

  # ✅ Multiple nodes สำหรับ horizontal scaling
  num_cache_nodes      = 3

  # ✅ AZ distribution สำหรับ HA
  az_mode              = "cross-az"
  preferred_availability_zones = [
    "${var.aws_region}a",
    "${var.aws_region}b",
    "${var.aws_region}c",
  ]

  # Network
  subnet_group_name  = aws_elasticache_subnet_group.main.name
  security_group_ids = [aws_security_group.memcached.id]

  port = 11211

  # Parameter group
  parameter_group_name = aws_elasticache_parameter_group.memcached.name

  maintenance_window     = "sun:05:00-sun:06:00"
  apply_immediately      = false

  tags = {
    Name      = "${var.project_name}-memcached"
    ManagedBy = "terraform"
  }
}

# ✅ Memcached Parameter Group
resource "aws_elasticache_parameter_group" "memcached" {
  name        = "${var.project_name}-memcached-params"
  family      = "memcached1.6"
  description = "Memcached 1.6 parameter group"

  parameter {
    name  = "max_item_size"
    value = "10485760"  # 10MB
  }

  tags = {
    Name      = "${var.project_name}-memcached-params"
    ManagedBy = "terraform"
  }
}
```

---

## Step 464: Parameter Groups

### Redis Parameter Groups

```hcl
# ✅ Redis 7.x Parameter Group
resource "aws_elasticache_parameter_group" "redis7" {
  name        = "${var.project_name}-redis7-params"
  family      = "redis7"
  description = "Redis 7 parameter group for ${var.project_name}"

  # Performance tuning
  parameter {
    name  = "maxmemory-policy"
    value = "allkeys-lru"  # Evict LRU keys when maxmemory hit
  }

  parameter {
    name  = "activerehashing"
    value = "yes"
  }

  parameter {
    name  = "lazyfree-lazy-eviction"
    value = "yes"
  }

  parameter {
    name  = "lazyfree-lazy-expire"
    value = "yes"
  }

  parameter {
    name  = "lazyfree-lazy-server-del"
    value = "yes"
  }

  # ✅ Slow log สำหรับ monitoring
  parameter {
    name  = "slowlog-log-slower-than"
    value = "10000"  # microseconds
  }

  parameter {
    name  = "slowlog-max-len"
    value = "1024"
  }

  # ✅ Notify keyspace events (สำหรับ pub/sub)
  parameter {
    name  = "notify-keyspace-events"
    value = "Ex"  # Expired events
  }

  tags = {
    Name      = "${var.project_name}-redis7-params"
    ManagedBy = "terraform"
  }
}

# ✅ Redis 7 Cluster Mode Parameter Group
resource "aws_elasticache_parameter_group" "redis7_cluster" {
  name        = "${var.project_name}-redis7-cluster-params"
  family      = "redis7"
  description = "Redis 7 cluster mode parameter group"

  parameter {
    name  = "cluster-enabled"
    value = "yes"  # ✅ เปิด cluster mode
  }

  parameter {
    name  = "maxmemory-policy"
    value = "allkeys-lru"
  }

  parameter {
    name  = "slowlog-log-slower-than"
    value = "10000"
  }

  tags = {
    Name      = "${var.project_name}-redis7-cluster-params"
    ManagedBy = "terraform"
  }
}
```

---

## Step 465: ElastiCache Users และ User Groups

```hcl
# ✅ Default User (จำเป็นต้องมี)
resource "aws_elasticache_user" "default" {
  user_id       = "default"
  user_name     = "default"
  access_string = "off ~* +@all"  # ✅ Disable default user!
  engine        = "REDIS"

  # ✅ Password สำหรับ default user (แม้จะ disabled)
  passwords = [random_password.redis_default_password.result]

  tags = {
    Name      = "elasticache-default-user"
    ManagedBy = "terraform"
  }
}

# ✅ Application User
resource "aws_elasticache_user" "app" {
  user_id       = "app-user"
  user_name     = "app"
  access_string = "on ~* +@read +@write -@dangerous"  # Read/Write แต่ไม่ dangerous commands
  engine        = "REDIS"

  passwords = [random_password.redis_app_password.result]

  tags = {
    Name      = "elasticache-app-user"
    ManagedBy = "terraform"
  }
}

# ✅ Read-Only User
resource "aws_elasticache_user" "readonly" {
  user_id       = "readonly-user"
  user_name     = "readonly"
  access_string = "on ~* +@read"  # Read only
  engine        = "REDIS"

  passwords = [random_password.redis_readonly_password.result]

  tags = {
    Name      = "elasticache-readonly-user"
    ManagedBy = "terraform"
  }
}

# ✅ User Group
resource "aws_elasticache_user_group" "app" {
  engine        = "REDIS"
  user_group_id = "${var.project_name}-user-group"

  user_ids = [
    aws_elasticache_user.default.user_id,
    aws_elasticache_user.app.user_id,
    aws_elasticache_user.readonly.user_id,
  ]

  tags = {
    Name      = "${var.project_name}-user-group"
    ManagedBy = "terraform"
  }
}

# ✅ Passwords
resource "random_password" "redis_default_password" {
  length  = 32
  special = false
}

resource "random_password" "redis_app_password" {
  length  = 32
  special = false
}

resource "random_password" "redis_readonly_password" {
  length  = 32
  special = false
}

# ✅ เก็บ credentials ใน Secrets Manager
resource "aws_secretsmanager_secret" "redis_credentials" {
  name                    = "${var.project_name}/redis/credentials"
  description             = "Redis user credentials"
  recovery_window_in_days = 7
  kms_key_id              = aws_kms_key.secrets.arn

  tags = {
    Name      = "${var.project_name}-redis-credentials"
    ManagedBy = "terraform"
  }
}

resource "aws_secretsmanager_secret_version" "redis_credentials" {
  secret_id = aws_secretsmanager_secret.redis_credentials.id
  secret_string = jsonencode({
    app_username  = aws_elasticache_user.app.user_name
    app_password  = random_password.redis_app_password.result
    host          = aws_elasticache_replication_group.redis.primary_endpoint_address
    port          = aws_elasticache_replication_group.redis.port
    reader_host   = aws_elasticache_replication_group.redis.reader_endpoint_address
  })
}
```

---

## Step 466: KMS สำหรับ ElastiCache

```hcl
# ✅ KMS Key สำหรับ ElastiCache
resource "aws_kms_key" "elasticache" {
  description             = "KMS key for ElastiCache encryption - ${var.project_name}"
  deletion_window_in_days = 14
  enable_key_rotation     = true  # ✅ Automatic key rotation

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Sid    = "Enable IAM User Permissions"
        Effect = "Allow"
        Principal = {
          AWS = "arn:aws:iam::${data.aws_caller_identity.current.account_id}:root"
        }
        Action   = "kms:*"
        Resource = "*"
      },
      {
        Sid    = "Allow ElastiCache to use key"
        Effect = "Allow"
        Principal = {
          Service = "elasticache.amazonaws.com"
        }
        Action = [
          "kms:Encrypt",
          "kms:Decrypt",
          "kms:ReEncrypt*",
          "kms:GenerateDataKey*",
          "kms:DescribeKey",
        ]
        Resource = "*"
      },
    ]
  })

  tags = {
    Name      = "${var.project_name}-elasticache-kms"
    ManagedBy = "terraform"
  }
}

resource "aws_kms_alias" "elasticache" {
  name          = "alias/${var.project_name}-elasticache"
  target_key_id = aws_kms_key.elasticache.key_id
}
```

---

## Step 467: CloudWatch Monitoring สำหรับ ElastiCache

```hcl
# ✅ CloudWatch Alarms สำหรับ Redis
resource "aws_cloudwatch_metric_alarm" "redis_cpu" {
  alarm_name          = "${var.project_name}-redis-cpu"
  comparison_operator = "GreaterThanOrEqualToThreshold"
  evaluation_periods  = 2
  metric_name         = "EngineCPUUtilization"
  namespace           = "AWS/ElastiCache"
  period              = 300
  statistic           = "Average"
  threshold           = 75
  alarm_description   = "Redis CPU usage is high"
  alarm_actions       = [aws_sns_topic.alerts.arn]

  dimensions = {
    CacheClusterId = "${aws_elasticache_replication_group.redis.id}-001"
  }

  tags = {
    Name      = "${var.project_name}-redis-cpu-alarm"
    ManagedBy = "terraform"
  }
}

resource "aws_cloudwatch_metric_alarm" "redis_memory" {
  alarm_name          = "${var.project_name}-redis-memory"
  comparison_operator = "GreaterThanOrEqualToThreshold"
  evaluation_periods  = 2
  metric_name         = "DatabaseMemoryUsagePercentage"
  namespace           = "AWS/ElastiCache"
  period              = 300
  statistic           = "Average"
  threshold           = 75
  alarm_description   = "Redis memory usage is high"
  alarm_actions       = [aws_sns_topic.alerts.arn]

  dimensions = {
    ReplicationGroupId = aws_elasticache_replication_group.redis.id
  }
}

resource "aws_cloudwatch_metric_alarm" "redis_connections" {
  alarm_name          = "${var.project_name}-redis-connections"
  comparison_operator = "GreaterThanOrEqualToThreshold"
  evaluation_periods  = 2
  metric_name         = "CurrConnections"
  namespace           = "AWS/ElastiCache"
  period              = 300
  statistic           = "Maximum"
  threshold           = 5000
  alarm_description   = "Too many Redis connections"
  alarm_actions       = [aws_sns_topic.alerts.arn]

  dimensions = {
    CacheClusterId = "${aws_elasticache_replication_group.redis.id}-001"
  }
}

resource "aws_cloudwatch_metric_alarm" "redis_cache_hits" {
  alarm_name          = "${var.project_name}-redis-cache-hit-ratio"
  comparison_operator = "LessThanThreshold"
  evaluation_periods  = 3
  metric_name         = "CacheHitRate"
  namespace           = "AWS/ElastiCache"
  period              = 300
  statistic           = "Average"
  threshold           = 0.5  # Alert ถ้า hit rate ต่ำกว่า 50%
  alarm_description   = "Redis cache hit rate is low"
  alarm_actions       = [aws_sns_topic.alerts.arn]

  dimensions = {
    ReplicationGroupId = aws_elasticache_replication_group.redis.id
  }
}

resource "aws_cloudwatch_metric_alarm" "redis_replication_lag" {
  alarm_name          = "${var.project_name}-redis-replication-lag"
  comparison_operator = "GreaterThanOrEqualToThreshold"
  evaluation_periods  = 2
  metric_name         = "ReplicationLag"
  namespace           = "AWS/ElastiCache"
  period              = 60
  statistic           = "Maximum"
  threshold           = 10  # Alert ถ้า lag > 10 วินาที
  alarm_description   = "Redis replication lag is too high"
  alarm_actions       = [aws_sns_topic.alerts.arn]

  dimensions = {
    CacheClusterId = "${aws_elasticache_replication_group.redis.id}-002"  # Replica node
  }
}
```

---

## Step 468: Redis vs Memcached เปรียบเทียบ

### เมื่อไหร่ควรใช้ Redis

```hcl
# ✅ Use Cases สำหรับ Redis:
# 1. Session caching
# 2. Pub/Sub messaging
# 3. Sorted sets (leaderboards)
# 4. Streams
# 5. Geospatial indexing
# 6. เมื่อต้องการ persistence

# ✅ Redis สำหรับ Session Store
resource "aws_elasticache_replication_group" "sessions" {
  replication_group_id = "${var.project_name}-sessions"
  description          = "Redis for session storage"

  engine         = "redis"
  engine_version = "7.1"
  node_type      = "cache.t3.medium"

  num_cache_clusters         = 2
  automatic_failover_enabled = true

  at_rest_encryption_enabled = true
  transit_encryption_enabled = true
  kms_key_id                 = aws_kms_key.elasticache.arn
  auth_token                 = random_password.redis_auth_token.result

  subnet_group_name  = aws_elasticache_subnet_group.main.name
  security_group_ids = [aws_security_group.redis.id]

  # ✅ Snapshot สำหรับ session persistence
  snapshot_retention_limit = 1

  parameter_group_name = aws_elasticache_parameter_group.redis_sessions.name

  tags = {
    Name    = "${var.project_name}-sessions"
    Purpose = "session-store"
  }
}

resource "aws_elasticache_parameter_group" "redis_sessions" {
  name   = "${var.project_name}-redis-sessions"
  family = "redis7"

  parameter {
    name  = "maxmemory-policy"
    value = "volatile-lru"  # Evict keys with TTL first
  }

  parameter {
    name  = "notify-keyspace-events"
    value = "Ex"  # Expired key notifications
  }
}
```

### เมื่อไหร่ควรใช้ Memcached

```hcl
# ✅ Use Cases สำหรับ Memcached:
# 1. Simple key-value caching
# 2. เมื่อต้องการ multi-threading
# 3. ไม่ต้องการ persistence
# 4. Simple horizontal scaling

# ✅ Memcached สำหรับ Object Cache
resource "aws_elasticache_cluster" "object_cache" {
  cluster_id     = "${var.project_name}-object-cache"
  engine         = "memcached"
  engine_version = "1.6.22"
  node_type      = "cache.r6g.large"

  num_cache_nodes = 4  # เพิ่ม nodes สำหรับ horizontal scaling

  az_mode = "cross-az"
  preferred_availability_zones = [
    "${var.aws_region}a",
    "${var.aws_region}b",
    "${var.aws_region}c",
    "${var.aws_region}a",  # ซ้ำ AZ ได้
  ]

  subnet_group_name  = aws_elasticache_subnet_group.main.name
  security_group_ids = [aws_security_group.memcached.id]

  parameter_group_name = aws_elasticache_parameter_group.memcached.name

  tags = {
    Name    = "${var.project_name}-object-cache"
    Purpose = "object-caching"
  }
}
```

---

## Step 469: Connection Patterns จาก Application

### Python Application

```hcl
# ✅ Environment Variables สำหรับ Application
resource "aws_lambda_function" "app_with_redis" {
  filename         = data.archive_file.app.output_path
  function_name    = "${var.project_name}-app"
  role             = aws_iam_role.lambda_app.arn
  handler          = "app.handler"
  runtime          = "python3.11"
  source_code_hash = data.archive_file.app.output_base64sha256

  memory_size = 512
  timeout     = 30

  environment {
    variables = {
      # ✅ Redis connection details
      REDIS_PRIMARY_ENDPOINT = aws_elasticache_replication_group.redis.primary_endpoint_address
      REDIS_READER_ENDPOINT  = aws_elasticache_replication_group.redis.reader_endpoint_address
      REDIS_PORT             = tostring(aws_elasticache_replication_group.redis.port)
      REDIS_AUTH_SECRET_ARN  = aws_secretsmanager_secret.redis_auth_token.arn

      # ✅ Memcached connection details
      MEMCACHED_ENDPOINT = aws_elasticache_cluster.object_cache.cluster_address
      MEMCACHED_PORT     = tostring(aws_elasticache_cluster.object_cache.port)

      ENVIRONMENT = var.environment
    }
  }

  vpc_config {
    subnet_ids         = aws_subnet.private[*].id
    security_group_ids = [aws_security_group.lambda.id]
  }

  tracing_config {
    mode = "Active"
  }
}
```

---

## Step 470: Complete Production Redis Setup

```hcl
# ✅ Complete Production Redis Configuration

# 1. KMS Key
resource "aws_kms_key" "redis" {
  description             = "KMS key for Redis encryption"
  deletion_window_in_days = 14
  enable_key_rotation     = true

  tags = merge(local.common_tags, {
    Name = "${var.project_name}-redis-kms"
  })
}

# 2. Auth Token
resource "random_password" "redis" {
  length  = 64
  special = false
}

resource "aws_secretsmanager_secret" "redis" {
  name       = "${var.project_name}/redis/auth-token"
  kms_key_id = aws_kms_key.redis.arn
}

resource "aws_secretsmanager_secret_version" "redis" {
  secret_id     = aws_secretsmanager_secret.redis.id
  secret_string = random_password.redis.result
}

# 3. Parameter Group
resource "aws_elasticache_parameter_group" "redis_prod" {
  name   = "${var.project_name}-redis-prod"
  family = "redis7"

  parameter {
    name  = "maxmemory-policy"
    value = "allkeys-lru"
  }

  parameter {
    name  = "slowlog-log-slower-than"
    value = "10000"
  }

  parameter {
    name  = "lazyfree-lazy-eviction"
    value = "yes"
  }
}

# 4. Subnet Group
resource "aws_elasticache_subnet_group" "redis" {
  name       = "${var.project_name}-redis-subnet-group"
  subnet_ids = var.private_subnet_ids
}

# 5. Security Group
resource "aws_security_group" "redis_prod" {
  name   = "${var.project_name}-redis-prod-sg"
  vpc_id = var.vpc_id

  ingress {
    from_port       = 6379
    to_port         = 6379
    protocol        = "tcp"
    security_groups = [var.app_security_group_id]
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = merge(local.common_tags, {
    Name = "${var.project_name}-redis-prod-sg"
  })
}

# 6. Redis Replication Group
resource "aws_elasticache_replication_group" "prod" {
  replication_group_id = "${var.project_name}-prod"
  description          = "Production Redis"

  engine         = "redis"
  engine_version = "7.1"
  node_type      = var.redis_node_type

  num_cache_clusters         = var.redis_num_clusters
  automatic_failover_enabled = true
  multi_az_enabled           = true

  at_rest_encryption_enabled = true
  transit_encryption_enabled = true
  kms_key_id                 = aws_kms_key.redis.arn
  auth_token                 = random_password.redis.result

  subnet_group_name    = aws_elasticache_subnet_group.redis.name
  security_group_ids   = [aws_security_group.redis_prod.id]
  parameter_group_name = aws_elasticache_parameter_group.redis_prod.name

  snapshot_retention_limit = 7
  maintenance_window       = "sun:05:00-sun:06:00"
  snapshot_window          = "04:00-05:00"

  apply_immediately          = false
  auto_minor_version_upgrade = true

  notification_topic_arn = var.alerts_sns_topic_arn

  tags = merge(local.common_tags, {
    Name = "${var.project_name}-redis-prod"
  })
}

# 7. Outputs
output "redis_primary_endpoint" {
  description = "Redis primary endpoint"
  value       = aws_elasticache_replication_group.prod.primary_endpoint_address
}

output "redis_reader_endpoint" {
  description = "Redis reader endpoint"
  value       = aws_elasticache_replication_group.prod.reader_endpoint_address
}

output "redis_port" {
  description = "Redis port"
  value       = aws_elasticache_replication_group.prod.port
}

output "redis_auth_secret_arn" {
  description = "ARN of Secrets Manager secret containing Redis auth token"
  value       = aws_secretsmanager_secret.redis.arn
}
```

---

## Redis vs Memcached Comparison Table

| Feature | Redis | Memcached |
|---------|-------|-----------|
| Data Structures | Rich (strings, hashes, lists, sets, sorted sets, streams) | Simple key-value |
| Persistence | Yes (RDB, AOF) | No |
| Replication | Yes | No |
| Pub/Sub | Yes | No |
| Clustering | Yes (cluster mode) | Yes (multi-node) |
| Multi-threading | No (single-threaded) | Yes |
| Encryption | Yes (at rest + in transit) | No encryption at rest |
| Auth | Yes (AUTH token, ACL) | No |
| Max Value Size | 512MB | 1MB |
| Atomic Operations | Yes | Yes |

## ElastiCache Security Checklist

### ✅ สิ่งที่ควรทำ

1. **Encryption at Rest**: เปิด `at_rest_encryption_enabled = true`
2. **Encryption in Transit**: เปิด `transit_encryption_enabled = true`
3. **Auth Token**: ใช้ `auth_token` สำหรับ Redis
4. **Private Subnets**: ใส่ใน private subnets เท่านั้น
5. **Security Groups**: จำกัดการเข้าถึงเฉพาะ application layer
6. **Multi-AZ**: เปิด `multi_az_enabled = true`
7. **Auto Failover**: เปิด `automatic_failover_enabled = true`
8. **Backup**: ตั้ง `snapshot_retention_limit` > 0
9. **Secrets Manager**: เก็บ credentials อย่างปลอดภัย
10. **CloudWatch Alarms**: ตั้ง alarms สำหรับ CPU, Memory, Connections

### ❌ สิ่งที่ไม่ควรทำ

1. ❌ ไม่เปิด encryption
2. ❌ ไม่มี auth token
3. ❌ เปิด public access
4. ❌ Single node (ไม่มี HA)
5. ❌ ไม่มี backup
6. ❌ Hardcode auth token

---

**Next Steps**: ไปต่อที่ Part 048 - AWS SQS, SNS & Event-Driven Architecture
