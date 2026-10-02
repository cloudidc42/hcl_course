# Part 042: AWS RDS Databases
## การสร้างและจัดการ Database ด้วย Terraform (Steps 411-420)

---

## บทนำ (Introduction)

Amazon RDS (Relational Database Service) ช่วยให้เราจัดการ relational databases บน AWS ได้ง่ายขึ้น
โดยรองรับหลาย database engine และมี features สำหรับ high availability, backup, และ security

**หัวข้อที่จะเรียนรู้:**
- RDS Instance สำหรับ Engine ต่างๆ (MySQL, PostgreSQL, Oracle, SQL Server, MariaDB)
- Multi-AZ Deployment และ Read Replicas
- Subnet Groups และ Parameter Groups
- Storage Options และ Encryption
- Aurora Cluster สำหรับ High Performance
- Password Management ด้วย Secrets Manager
- Security Best Practices

---

## Step 411: พื้นฐาน RDS Instance

### aws_db_instance พื้นฐาน

```hcl
# ✅ Secure: PostgreSQL RDS Instance สำหรับ Production
resource "aws_db_instance" "postgres_prod" {
  identifier = "${var.project_name}-postgres-prod"

  # Engine configuration
  engine         = "postgres"
  engine_version = "15.4"
  instance_class = "db.t3.medium"

  # Storage configuration
  allocated_storage     = 100
  max_allocated_storage = 1000  # Auto scaling สูงสุด 1TB
  storage_type          = "gp3"
  storage_encrypted     = true  # ✅ เปิด encryption เสมอ
  kms_key_id            = aws_kms_key.rds.arn

  # Database configuration
  db_name  = "appdb"
  username = "dbadmin"
  # ✅ ดึง password จาก Secrets Manager
  password = random_password.db_password.result

  # Network configuration
  db_subnet_group_name   = aws_db_subnet_group.main.name
  vpc_security_group_ids = [aws_security_group.rds.id]
  publicly_accessible    = false  # ✅ ห้าม public access!

  # High Availability
  multi_az = true  # ✅ ใช้ Multi-AZ สำหรับ production

  # Backup configuration
  backup_retention_period = 7       # ✅ เก็บ backup 7 วัน
  backup_window           = "03:00-04:00"  # UTC
  maintenance_window      = "Mon:04:00-Mon:05:00"
  copy_tags_to_snapshot   = true

  # Monitoring
  performance_insights_enabled          = true
  performance_insights_retention_period = 7
  monitoring_interval                   = 60  # Enhanced monitoring
  monitoring_role_arn                   = aws_iam_role.rds_enhanced_monitoring.arn

  # Parameter group
  parameter_group_name = aws_db_parameter_group.postgres15.name

  # Deletion protection
  deletion_protection   = true   # ✅ ป้องกันการลบโดยไม่ตั้งใจ
  skip_final_snapshot   = false  # ✅ สร้าง snapshot ก่อน destroy
  final_snapshot_identifier = "${var.project_name}-postgres-final-snapshot"

  # Auto minor version upgrade
  auto_minor_version_upgrade = true

  tags = {
    Name        = "${var.project_name}-postgres-prod"
    Environment = "production"
    ManagedBy   = "terraform"
    Backup      = "required"
  }
}

# ❌ Insecure: อย่าทำแบบนี้!
resource "aws_db_instance" "bad_db" {
  identifier        = "bad-database"
  engine            = "mysql"
  engine_version    = "8.0"
  instance_class    = "db.t3.micro"
  allocated_storage = 20

  # ❌ ไม่มี encryption
  storage_encrypted = false

  # ❌ Public accessible!
  publicly_accessible = true

  # ❌ ไม่มี backup
  backup_retention_period = 0
  skip_final_snapshot     = true

  username = "admin"
  password = "password123"  # ❌ Hardcoded password!
}
```

---

## Step 412: DB Subnet Group และ Security Groups

### aws_db_subnet_group

```hcl
# ✅ Secure: DB Subnet Group ใน private subnets
resource "aws_db_subnet_group" "main" {
  name        = "${var.project_name}-db-subnet-group"
  description = "Database subnet group for ${var.project_name}"

  # ✅ ใช้ private subnets เท่านั้น
  subnet_ids = [
    aws_subnet.private_1a.id,
    aws_subnet.private_1b.id,
    aws_subnet.private_1c.id,
  ]

  tags = {
    Name      = "${var.project_name}-db-subnet-group"
    ManagedBy = "terraform"
  }
}

# ✅ Security Group สำหรับ RDS
resource "aws_security_group" "rds" {
  name        = "${var.project_name}-rds-sg"
  description = "Security group for RDS instances"
  vpc_id      = aws_vpc.main.id

  # ✅ อนุญาตเฉพาะ application servers เท่านั้น
  ingress {
    description     = "PostgreSQL from app servers"
    from_port       = 5432
    to_port         = 5432
    protocol        = "tcp"
    security_groups = [aws_security_group.app.id]
  }

  # ✅ ห้าม inbound จาก internet
  # (ไม่มี 0.0.0.0/0 ingress rule)

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = {
    Name      = "${var.project_name}-rds-sg"
    ManagedBy = "terraform"
  }
}

# ✅ Security Group สำหรับ MySQL
resource "aws_security_group" "mysql_rds" {
  name        = "${var.project_name}-mysql-rds-sg"
  description = "Security group for MySQL RDS"
  vpc_id      = aws_vpc.main.id

  ingress {
    description     = "MySQL from app servers"
    from_port       = 3306
    to_port         = 3306
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
    Name      = "${var.project_name}-mysql-rds-sg"
    ManagedBy = "terraform"
  }
}
```

---

## Step 413: Engine Types ต่างๆ

### MySQL RDS

```hcl
# ✅ MySQL 8.0 Production Database
resource "aws_db_instance" "mysql_prod" {
  identifier = "${var.project_name}-mysql-prod"

  engine         = "mysql"
  engine_version = "8.0.35"
  instance_class = "db.r6g.large"

  allocated_storage     = 200
  max_allocated_storage = 2000
  storage_type          = "gp3"
  iops                  = 3000  # สำหรับ gp3
  storage_encrypted     = true
  kms_key_id            = aws_kms_key.rds.arn

  db_name  = "appdb"
  username = "admin"
  password = random_password.mysql_password.result

  db_subnet_group_name   = aws_db_subnet_group.main.name
  vpc_security_group_ids = [aws_security_group.mysql_rds.id]
  publicly_accessible    = false

  multi_az                = true
  backup_retention_period = 14
  backup_window           = "02:00-03:00"
  maintenance_window      = "Sun:03:00-Sun:04:00"

  parameter_group_name = aws_db_parameter_group.mysql80.name

  performance_insights_enabled          = true
  performance_insights_retention_period = 7
  monitoring_interval                   = 60
  monitoring_role_arn                   = aws_iam_role.rds_enhanced_monitoring.arn

  deletion_protection       = true
  skip_final_snapshot       = false
  final_snapshot_identifier = "${var.project_name}-mysql-final"

  tags = {
    Name        = "${var.project_name}-mysql-prod"
    Environment = "production"
    Engine      = "mysql"
    ManagedBy   = "terraform"
  }
}
```

### MariaDB RDS

```hcl
resource "aws_db_instance" "mariadb" {
  identifier = "${var.project_name}-mariadb"

  engine         = "mariadb"
  engine_version = "10.11.6"
  instance_class = "db.t3.large"

  allocated_storage     = 100
  max_allocated_storage = 500
  storage_type          = "gp3"
  storage_encrypted     = true
  kms_key_id            = aws_kms_key.rds.arn

  db_name  = "appdb"
  username = "admin"
  password = random_password.mariadb_password.result

  db_subnet_group_name   = aws_db_subnet_group.main.name
  vpc_security_group_ids = [aws_security_group.mysql_rds.id]
  publicly_accessible    = false

  multi_az                = true
  backup_retention_period = 7

  parameter_group_name = aws_db_parameter_group.mariadb.name

  deletion_protection = true
  skip_final_snapshot = false

  tags = {
    Name      = "${var.project_name}-mariadb"
    Engine    = "mariadb"
    ManagedBy = "terraform"
  }
}
```

### Oracle RDS

```hcl
resource "aws_db_instance" "oracle" {
  identifier = "${var.project_name}-oracle"

  engine                      = "oracle-ee"  # oracle-ee, oracle-se2, oracle-se2-cdb
  engine_version              = "19.0.0.0.ru-2023-10.rur-2023-10.r1"
  instance_class              = "db.r6g.xlarge"
  license_model               = "bring-your-own-license"  # หรือ "license-included"

  allocated_storage     = 500
  max_allocated_storage = 2000
  storage_type          = "gp3"
  storage_encrypted     = true
  kms_key_id            = aws_kms_key.rds.arn

  # Oracle ใช้ SID แทน db_name
  db_name  = "ORCL"  # Oracle SID
  username = "admin"
  password = random_password.oracle_password.result

  db_subnet_group_name   = aws_db_subnet_group.main.name
  vpc_security_group_ids = [aws_security_group.oracle_rds.id]
  publicly_accessible    = false

  multi_az                = true
  backup_retention_period = 14

  option_group_name    = aws_db_option_group.oracle.name
  parameter_group_name = aws_db_parameter_group.oracle19.name

  deletion_protection = true
  skip_final_snapshot = false

  tags = {
    Name      = "${var.project_name}-oracle"
    Engine    = "oracle-ee"
    ManagedBy = "terraform"
  }
}
```

### SQL Server RDS

```hcl
resource "aws_db_instance" "sqlserver" {
  identifier = "${var.project_name}-sqlserver"

  engine         = "sqlserver-se"  # sqlserver-ee, sqlserver-se, sqlserver-ex, sqlserver-web
  engine_version = "15.00.4345.5.v1"  # SQL Server 2019
  instance_class = "db.r6i.xlarge"
  license_model  = "license-included"

  allocated_storage     = 200
  max_allocated_storage = 2000
  storage_type          = "gp3"
  storage_encrypted     = true
  kms_key_id            = aws_kms_key.rds.arn

  # SQL Server ไม่ใช้ db_name (ใช้ SQL Server Management Studio แทน)
  username = "admin"
  password = random_password.sqlserver_password.result

  db_subnet_group_name   = aws_db_subnet_group.main.name
  vpc_security_group_ids = [aws_security_group.sqlserver_rds.id]
  publicly_accessible    = false

  multi_az                = true
  backup_retention_period = 7

  timezone = "UTC"

  deletion_protection = true
  skip_final_snapshot = false

  tags = {
    Name      = "${var.project_name}-sqlserver"
    Engine    = "sqlserver-se"
    ManagedBy = "terraform"
  }
}
```

---

## Step 414: Parameter Groups

### aws_db_parameter_group

```hcl
# ✅ PostgreSQL 15 Parameter Group
resource "aws_db_parameter_group" "postgres15" {
  name        = "${var.project_name}-postgres15-params"
  family      = "postgres15"
  description = "PostgreSQL 15 parameter group for ${var.project_name}"

  # Performance tuning parameters
  parameter {
    name  = "shared_preload_libraries"
    value = "pg_stat_statements,auto_explain"
  }

  parameter {
    name  = "log_min_duration_statement"
    value = "1000"  # Log queries > 1 second
  }

  parameter {
    name  = "log_statement"
    value = "ddl"
  }

  parameter {
    name  = "max_connections"
    value = "200"
  }

  parameter {
    name  = "work_mem"
    value = "65536"  # 64MB
  }

  parameter {
    name  = "maintenance_work_mem"
    value = "524288"  # 512MB
  }

  parameter {
    name  = "effective_cache_size"
    value = "3145728"  # 3GB (75% of RAM)
  }

  parameter {
    name  = "wal_compression"
    value = "on"
  }

  tags = {
    Name      = "${var.project_name}-postgres15-params"
    ManagedBy = "terraform"
  }
}

# ✅ MySQL 8.0 Parameter Group
resource "aws_db_parameter_group" "mysql80" {
  name        = "${var.project_name}-mysql80-params"
  family      = "mysql8.0"
  description = "MySQL 8.0 parameter group for ${var.project_name}"

  parameter {
    name  = "character_set_server"
    value = "utf8mb4"
  }

  parameter {
    name  = "collation_server"
    value = "utf8mb4_unicode_ci"
  }

  parameter {
    name  = "max_connections"
    value = "500"
  }

  parameter {
    name  = "innodb_buffer_pool_size"
    value = "{DBInstanceClassMemory*3/4}"
  }

  parameter {
    name  = "slow_query_log"
    value = "1"
  }

  parameter {
    name  = "long_query_time"
    value = "1"  # 1 second
  }

  parameter {
    name  = "general_log"
    value = "0"  # ปิด general log ใน production
  }

  parameter {
    name         = "binlog_format"
    value        = "ROW"
    apply_method = "pending-reboot"
  }

  tags = {
    Name      = "${var.project_name}-mysql80-params"
    ManagedBy = "terraform"
  }
}

# ✅ MariaDB Parameter Group
resource "aws_db_parameter_group" "mariadb" {
  name        = "${var.project_name}-mariadb-params"
  family      = "mariadb10.11"
  description = "MariaDB 10.11 parameter group"

  parameter {
    name  = "character_set_server"
    value = "utf8mb4"
  }

  parameter {
    name  = "collation_server"
    value = "utf8mb4_unicode_ci"
  }

  parameter {
    name  = "slow_query_log"
    value = "1"
  }

  parameter {
    name  = "long_query_time"
    value = "1"
  }

  tags = {
    Name      = "${var.project_name}-mariadb-params"
    ManagedBy = "terraform"
  }
}
```

---

## Step 415: Option Groups (สำหรับ Oracle และ SQL Server)

### aws_db_option_group

```hcl
# ✅ Oracle Option Group
resource "aws_db_option_group" "oracle" {
  name                 = "${var.project_name}-oracle-options"
  option_group_description = "Oracle options for ${var.project_name}"
  engine_name          = "oracle-ee"
  major_engine_version = "19"

  option {
    option_name = "STATSPACK"
  }

  option {
    option_name = "NATIVE_NETWORK_ENCRYPTION"
    option_settings {
      name  = "SQLNET.ENCRYPTION_SERVER"
      value = "REQUIRED"
    }
    option_settings {
      name  = "SQLNET.ENCRYPTION_TYPES_SERVER"
      value = "AES256"
    }
  }

  tags = {
    Name      = "${var.project_name}-oracle-options"
    ManagedBy = "terraform"
  }
}

# ✅ SQL Server Option Group
resource "aws_db_option_group" "sqlserver" {
  name                 = "${var.project_name}-sqlserver-options"
  option_group_description = "SQL Server options"
  engine_name          = "sqlserver-se"
  major_engine_version = "15.00"

  option {
    option_name = "SQLSERVER_BACKUP_RESTORE"

    option_settings {
      name  = "IAM_ROLE_ARN"
      value = aws_iam_role.rds_backup.arn
    }
  }

  tags = {
    Name      = "${var.project_name}-sqlserver-options"
    ManagedBy = "terraform"
  }
}
```

---

## Step 416: Read Replicas

```hcl
# ✅ Read Replica สำหรับ PostgreSQL
resource "aws_db_instance" "postgres_replica" {
  identifier = "${var.project_name}-postgres-replica"

  # Read replica ระบุ source database
  replicate_source_db = aws_db_instance.postgres_prod.identifier

  instance_class = "db.t3.medium"

  # Storage settings (inherit from source)
  storage_encrypted = true
  kms_key_id        = aws_kms_key.rds.arn

  # ไม่ต้องระบุ db_name, username, password (inherit จาก source)

  # ✅ replica อาจไม่ต้อง Multi-AZ
  multi_az = false

  # Network
  vpc_security_group_ids = [aws_security_group.rds.id]
  db_subnet_group_name   = aws_db_subnet_group.main.name
  publicly_accessible    = false

  # Performance monitoring
  performance_insights_enabled = true
  monitoring_interval          = 60
  monitoring_role_arn          = aws_iam_role.rds_enhanced_monitoring.arn

  # ✅ Backup ไม่จำเป็นสำหรับ replica
  backup_retention_period = 0
  skip_final_snapshot     = true

  # ✅ Auto minor upgrade
  auto_minor_version_upgrade = true

  tags = {
    Name        = "${var.project_name}-postgres-replica"
    Environment = "production"
    Type        = "read-replica"
    ManagedBy   = "terraform"
  }
}

# ✅ Cross-Region Read Replica (สำหรับ DR)
resource "aws_db_instance" "postgres_dr_replica" {
  provider   = aws.dr_region
  identifier = "${var.project_name}-postgres-dr"

  replicate_source_db = aws_db_instance.postgres_prod.arn

  instance_class    = "db.t3.medium"
  storage_encrypted = true
  kms_key_id        = aws_kms_key.rds_dr.arn

  vpc_security_group_ids = [aws_security_group.rds_dr.id]
  db_subnet_group_name   = aws_db_subnet_group.dr.name
  publicly_accessible    = false

  skip_final_snapshot = true

  tags = {
    Name      = "${var.project_name}-postgres-dr"
    Type      = "dr-replica"
    ManagedBy = "terraform"
  }
}
```

---

## Step 417: Aurora Cluster

### aws_rds_cluster

```hcl
# ✅ Aurora PostgreSQL Cluster (Production)
resource "aws_rds_cluster" "aurora_postgres" {
  cluster_identifier = "${var.project_name}-aurora-postgres"

  engine         = "aurora-postgresql"
  engine_version = "15.4"
  engine_mode    = "provisioned"  # สำหรับ Aurora Serverless v2 ใช้ provisioned ด้วย

  database_name   = "appdb"
  master_username = "dbadmin"
  master_password = random_password.aurora_password.result

  db_subnet_group_name   = aws_db_subnet_group.main.name
  vpc_security_group_ids = [aws_security_group.rds.id]

  # ✅ Storage encryption
  storage_encrypted = true
  kms_key_id        = aws_kms_key.rds.arn

  # Backup
  backup_retention_period = 14
  preferred_backup_window = "03:00-04:00"
  preferred_maintenance_window = "Mon:04:00-Mon:05:00"
  copy_tags_to_snapshot   = true

  # ✅ Deletion protection
  deletion_protection       = true
  skip_final_snapshot       = false
  final_snapshot_identifier = "${var.project_name}-aurora-final"

  # ✅ Enhanced monitoring
  enabled_cloudwatch_logs_exports = ["postgresql"]

  # ✅ ป้องกัน public access
  port = 5432

  # Parameter group
  db_cluster_parameter_group_name = aws_rds_cluster_parameter_group.aurora_postgres15.name

  tags = {
    Name        = "${var.project_name}-aurora-postgres"
    Environment = "production"
    Engine      = "aurora-postgresql"
    ManagedBy   = "terraform"
  }
}

# ✅ Aurora Cluster Instances
resource "aws_rds_cluster_instance" "aurora_writer" {
  count = 1

  identifier         = "${var.project_name}-aurora-writer-${count.index + 1}"
  cluster_identifier = aws_rds_cluster.aurora_postgres.id

  instance_class = "db.r6g.large"
  engine         = aws_rds_cluster.aurora_postgres.engine
  engine_version = aws_rds_cluster.aurora_postgres.engine_version

  db_parameter_group_name = aws_db_parameter_group.aurora_postgres_instance.name

  # ✅ Enhanced monitoring
  monitoring_interval = 60
  monitoring_role_arn = aws_iam_role.rds_enhanced_monitoring.arn

  # ✅ Performance Insights
  performance_insights_enabled          = true
  performance_insights_retention_period = 7

  # ✅ Minor version upgrade
  auto_minor_version_upgrade = true

  tags = {
    Name      = "${var.project_name}-aurora-writer-${count.index + 1}"
    Role      = "writer"
    ManagedBy = "terraform"
  }
}

resource "aws_rds_cluster_instance" "aurora_reader" {
  count = 2

  identifier         = "${var.project_name}-aurora-reader-${count.index + 1}"
  cluster_identifier = aws_rds_cluster.aurora_postgres.id

  instance_class = "db.r6g.large"
  engine         = aws_rds_cluster.aurora_postgres.engine
  engine_version = aws_rds_cluster.aurora_postgres.engine_version

  db_parameter_group_name = aws_db_parameter_group.aurora_postgres_instance.name

  monitoring_interval = 60
  monitoring_role_arn = aws_iam_role.rds_enhanced_monitoring.arn

  performance_insights_enabled = true

  auto_minor_version_upgrade = true

  tags = {
    Name      = "${var.project_name}-aurora-reader-${count.index + 1}"
    Role      = "reader"
    ManagedBy = "terraform"
  }
}
```

### Aurora Serverless v2

```hcl
# ✅ Aurora Serverless v2 (auto-scaling)
resource "aws_rds_cluster" "aurora_serverless_v2" {
  cluster_identifier = "${var.project_name}-aurora-serverless"

  engine         = "aurora-postgresql"
  engine_version = "15.4"
  engine_mode    = "provisioned"  # Serverless v2 ใช้ provisioned mode

  database_name   = "appdb"
  master_username = "dbadmin"
  master_password = random_password.aurora_serverless_password.result

  db_subnet_group_name   = aws_db_subnet_group.main.name
  vpc_security_group_ids = [aws_security_group.rds.id]

  storage_encrypted = true
  kms_key_id        = aws_kms_key.rds.arn

  backup_retention_period = 7
  deletion_protection     = true
  skip_final_snapshot     = false
  final_snapshot_identifier = "${var.project_name}-serverless-final"

  # ✅ Serverless v2 scaling configuration
  serverlessv2_scaling_configuration {
    min_capacity = 0.5   # 0.5 ACU = minimum
    max_capacity = 16.0  # 16 ACU = maximum
  }

  tags = {
    Name      = "${var.project_name}-aurora-serverless"
    Type      = "serverless-v2"
    ManagedBy = "terraform"
  }
}

# Aurora Serverless v2 instance ต้องการ instance
resource "aws_rds_cluster_instance" "aurora_serverless_writer" {
  identifier         = "${var.project_name}-serverless-writer"
  cluster_identifier = aws_rds_cluster.aurora_serverless_v2.id

  # ✅ ใช้ db.serverless class สำหรับ Serverless v2
  instance_class = "db.serverless"
  engine         = aws_rds_cluster.aurora_serverless_v2.engine
  engine_version = aws_rds_cluster.aurora_serverless_v2.engine_version

  tags = {
    Name = "${var.project_name}-serverless-writer"
    Role = "writer"
  }
}
```

### Aurora MySQL

```hcl
# ✅ Aurora MySQL Cluster
resource "aws_rds_cluster" "aurora_mysql" {
  cluster_identifier = "${var.project_name}-aurora-mysql"

  engine         = "aurora-mysql"
  engine_version = "8.0.mysql_aurora.3.04.1"
  engine_mode    = "provisioned"

  database_name   = "appdb"
  master_username = "admin"
  master_password = random_password.aurora_mysql_password.result

  db_subnet_group_name   = aws_db_subnet_group.main.name
  vpc_security_group_ids = [aws_security_group.mysql_rds.id]

  storage_encrypted = true
  kms_key_id        = aws_kms_key.rds.arn

  backup_retention_period = 14
  deletion_protection     = true
  skip_final_snapshot     = false
  final_snapshot_identifier = "${var.project_name}-aurora-mysql-final"

  enabled_cloudwatch_logs_exports = ["error", "slowquery", "audit"]

  tags = {
    Name      = "${var.project_name}-aurora-mysql"
    Engine    = "aurora-mysql"
    ManagedBy = "terraform"
  }
}

resource "aws_rds_cluster_instance" "aurora_mysql_instances" {
  count = 2

  identifier         = "${var.project_name}-aurora-mysql-${count.index + 1}"
  cluster_identifier = aws_rds_cluster.aurora_mysql.id
  instance_class     = "db.r6g.large"
  engine             = aws_rds_cluster.aurora_mysql.engine
  engine_version     = aws_rds_cluster.aurora_mysql.engine_version

  monitoring_interval                   = 60
  monitoring_role_arn                   = aws_iam_role.rds_enhanced_monitoring.arn
  performance_insights_enabled          = true
  performance_insights_retention_period = 7

  tags = {
    Name = "${var.project_name}-aurora-mysql-${count.index + 1}"
  }
}
```

---

## Step 418: Database Snapshots

### aws_db_snapshot

```hcl
# ✅ Manual snapshot
resource "aws_db_snapshot" "postgres_snapshot" {
  db_instance_identifier = aws_db_instance.postgres_prod.identifier
  db_snapshot_identifier = "${var.project_name}-manual-snapshot-${formatdate("YYYYMMDDhhmmss", timestamp())}"

  tags = {
    Name      = "${var.project_name}-manual-snapshot"
    ManagedBy = "terraform"
    CreatedBy = "terraform-manual"
  }

  lifecycle {
    # ✅ ป้องกัน snapshot ถูก recreate ทุก apply
    ignore_changes = [db_snapshot_identifier]
  }
}

# ✅ Aurora Cluster Snapshot
resource "aws_db_cluster_snapshot" "aurora_snapshot" {
  db_cluster_identifier          = aws_rds_cluster.aurora_postgres.id
  db_cluster_snapshot_identifier = "${var.project_name}-aurora-snapshot"

  tags = {
    Name      = "${var.project_name}-aurora-snapshot"
    ManagedBy = "terraform"
  }
}
```

---

## Step 419: Password Management ด้วย Secrets Manager

```hcl
# ✅ สร้าง Random Password
resource "random_password" "db_password" {
  length           = 32
  special          = true
  override_special = "!#$%&*()-_=+[]{}<>:?"
  # ✅ หลีกเลี่ยง characters ที่อาจมีปัญหากับ connection strings
  min_lower   = 4
  min_upper   = 4
  min_numeric = 4
  min_special = 4
}

resource "random_password" "mysql_password" {
  length           = 32
  special          = true
  override_special = "!#$%&*()-_=+[]{}<>:?"
  min_lower        = 4
  min_upper        = 4
  min_numeric      = 4
  min_special      = 4
}

resource "random_password" "aurora_password" {
  length           = 32
  special          = true
  override_special = "!#$%&*()-_=+[]{}<>:?"
  min_lower        = 4
  min_upper        = 4
  min_numeric      = 4
  min_special      = 4
}

# ✅ เก็บ DB credentials ใน Secrets Manager
resource "aws_secretsmanager_secret" "db_credentials" {
  name                    = "${var.project_name}/database/postgres/credentials"
  description             = "PostgreSQL database credentials for ${var.project_name}"
  recovery_window_in_days = 7
  kms_key_id              = aws_kms_key.secrets.arn

  tags = {
    Name        = "${var.project_name}-db-credentials"
    Environment = var.environment
    ManagedBy   = "terraform"
  }
}

resource "aws_secretsmanager_secret_version" "db_credentials" {
  secret_id = aws_secretsmanager_secret.db_credentials.id
  secret_string = jsonencode({
    username = aws_db_instance.postgres_prod.username
    password = random_password.db_password.result
    host     = aws_db_instance.postgres_prod.address
    port     = aws_db_instance.postgres_prod.port
    dbname   = aws_db_instance.postgres_prod.db_name
    # ✅ Connection string สำหรับ applications
    connection_string = "postgresql://${aws_db_instance.postgres_prod.username}:${random_password.db_password.result}@${aws_db_instance.postgres_prod.address}:${aws_db_instance.postgres_prod.port}/${aws_db_instance.postgres_prod.db_name}"
  })
}

# ✅ MySQL credentials
resource "aws_secretsmanager_secret" "mysql_credentials" {
  name                    = "${var.project_name}/database/mysql/credentials"
  description             = "MySQL database credentials"
  recovery_window_in_days = 7
  kms_key_id              = aws_kms_key.secrets.arn

  tags = {
    Name      = "${var.project_name}-mysql-credentials"
    ManagedBy = "terraform"
  }
}

resource "aws_secretsmanager_secret_version" "mysql_credentials" {
  secret_id = aws_secretsmanager_secret.mysql_credentials.id
  secret_string = jsonencode({
    username = aws_db_instance.mysql_prod.username
    password = random_password.mysql_password.result
    host     = aws_db_instance.mysql_prod.address
    port     = aws_db_instance.mysql_prod.port
    dbname   = aws_db_instance.mysql_prod.db_name
  })
}
```

---

## Step 420: Enhanced Monitoring Role และ KMS Keys

### IAM Role สำหรับ Enhanced Monitoring

```hcl
# ✅ Enhanced Monitoring Role
data "aws_iam_policy_document" "rds_monitoring_trust" {
  statement {
    effect = "Allow"
    principals {
      type        = "Service"
      identifiers = ["monitoring.rds.amazonaws.com"]
    }
    actions = ["sts:AssumeRole"]
  }
}

resource "aws_iam_role" "rds_enhanced_monitoring" {
  name               = "${var.project_name}-rds-monitoring-role"
  assume_role_policy = data.aws_iam_policy_document.rds_monitoring_trust.json

  tags = {
    Name      = "${var.project_name}-rds-monitoring-role"
    ManagedBy = "terraform"
  }
}

resource "aws_iam_role_policy_attachment" "rds_enhanced_monitoring" {
  role       = aws_iam_role.rds_enhanced_monitoring.name
  policy_arn = "arn:aws:iam::aws:policy/service-role/AmazonRDSEnhancedMonitoringRole"
}
```

### KMS Keys สำหรับ RDS Encryption

```hcl
# ✅ KMS Key สำหรับ RDS encryption
resource "aws_kms_key" "rds" {
  description             = "KMS key for RDS encryption - ${var.project_name}"
  deletion_window_in_days = 14
  enable_key_rotation     = true  # ✅ เปิด automatic key rotation

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
        Sid    = "Allow RDS to use key"
        Effect = "Allow"
        Principal = {
          Service = "rds.amazonaws.com"
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
    Name      = "${var.project_name}-rds-kms"
    ManagedBy = "terraform"
  }
}

resource "aws_kms_alias" "rds" {
  name          = "alias/${var.project_name}-rds"
  target_key_id = aws_kms_key.rds.key_id
}

# ✅ KMS Key สำหรับ Secrets Manager
resource "aws_kms_key" "secrets" {
  description             = "KMS key for Secrets Manager - ${var.project_name}"
  deletion_window_in_days = 14
  enable_key_rotation     = true

  tags = {
    Name      = "${var.project_name}-secrets-kms"
    ManagedBy = "terraform"
  }
}

resource "aws_kms_alias" "secrets" {
  name          = "alias/${var.project_name}-secrets"
  target_key_id = aws_kms_key.secrets.key_id
}
```

---

## ตัวอย่าง Complete Production RDS Configuration

### main.tf

```hcl
# ✅ Complete Production PostgreSQL Setup
module "rds_postgres" {
  source = "./modules/rds"

  project_name = var.project_name
  environment  = "production"
  aws_region   = var.aws_region

  # Engine
  engine         = "postgres"
  engine_version = "15.4"
  instance_class = "db.r6g.large"

  # Storage
  allocated_storage     = 200
  max_allocated_storage = 2000
  storage_type          = "gp3"
  storage_encrypted     = true

  # Network
  vpc_id             = module.vpc.vpc_id
  private_subnet_ids = module.vpc.private_subnet_ids
  app_security_group_id = aws_security_group.app.id

  # HA
  multi_az = true

  # Backup
  backup_retention_period = 14

  # Monitoring
  performance_insights_enabled = true
  monitoring_interval          = 60
}
```

### outputs.tf

```hcl
output "postgres_endpoint" {
  description = "PostgreSQL RDS endpoint"
  value       = aws_db_instance.postgres_prod.address
  sensitive   = false  # Endpoint ไม่ sensitive แต่ password sensitive
}

output "postgres_port" {
  description = "PostgreSQL RDS port"
  value       = aws_db_instance.postgres_prod.port
}

output "postgres_db_name" {
  description = "PostgreSQL database name"
  value       = aws_db_instance.postgres_prod.db_name
}

output "db_credentials_secret_arn" {
  description = "ARN of Secrets Manager secret containing DB credentials"
  value       = aws_secretsmanager_secret.db_credentials.arn
}

output "aurora_cluster_endpoint" {
  description = "Aurora cluster writer endpoint"
  value       = aws_rds_cluster.aurora_postgres.endpoint
}

output "aurora_cluster_reader_endpoint" {
  description = "Aurora cluster reader endpoint"
  value       = aws_rds_cluster.aurora_postgres.reader_endpoint
}
```

---

## Storage Type เปรียบเทียบ

| Storage Type | Use Case | Max IOPS | Max Throughput |
|-------------|---------|---------|---------------|
| gp2 | General purpose (เก่า) | 16,000 | 250 MB/s |
| gp3 | General purpose (ใหม่, ประหยัดกว่า) | 16,000 | 1,000 MB/s |
| io1 | High IOPS workloads | 64,000 | 1,000 MB/s |
| io2 | High IOPS with durability | 256,000 | 4,000 MB/s |

```hcl
# ✅ gp3 Storage (แนะนำสำหรับ general workloads)
resource "aws_db_instance" "gp3_example" {
  # ...
  storage_type      = "gp3"
  allocated_storage = 100
  iops              = 3000      # Baseline IOPS สำหรับ gp3
  storage_throughput = 125      # MB/s สำหรับ gp3
}

# ✅ io1/io2 Storage (สำหรับ high IOPS workloads)
resource "aws_db_instance" "io1_example" {
  # ...
  storage_type      = "io1"
  allocated_storage = 500
  iops              = 20000  # High IOPS
}
```

---

## RDS Security Checklist

### ✅ สิ่งที่ควรทำ

1. **Encryption**: เปิด `storage_encrypted = true` เสมอ
2. **Private Subnets**: ใช้ private subnets สำหรับ DB subnet group
3. **No Public Access**: `publicly_accessible = false`
4. **Security Groups**: จำกัดการเข้าถึงเฉพาะ application layer
5. **Backup Retention**: ตั้งค่า `backup_retention_period` >= 7
6. **Multi-AZ**: สำหรับ production environments
7. **Deletion Protection**: เปิด `deletion_protection = true`
8. **Secrets Manager**: เก็บ credentials ใน Secrets Manager
9. **Enhanced Monitoring**: เปิด monitoring_interval
10. **Performance Insights**: เปิดสำหรับ performance analysis

### ❌ สิ่งที่ไม่ควรทำ

1. ❌ `publicly_accessible = true` (เปิด public access)
2. ❌ `storage_encrypted = false` (ไม่ encrypt)
3. ❌ Hardcode passwords ใน Terraform
4. ❌ `backup_retention_period = 0` (ไม่มี backup)
5. ❌ `skip_final_snapshot = true` ใน production
6. ❌ `deletion_protection = false` ใน production
7. ❌ ใช้ default security group
8. ❌ ไม่ตั้ง maintenance window

---

**Next Steps**: ไปต่อที่ Part 043 - AWS EKS Kubernetes Clusters
