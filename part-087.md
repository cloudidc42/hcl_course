# Part 087: Logging & Monitoring Misconfigurations
## ขั้นตอนที่ 861-870: การกำหนดค่า Logging และ Monitoring ที่ผิดพลาด

---

## ขั้นตอนที่ 861: ภาพรวม Logging & Monitoring Security

### ทำไม Logging สำคัญสำหรับ Security

```
Logging & Monitoring ช่วย:
1. DETECT security incidents in real-time
2. INVESTIGATE breaches after the fact
3. COMPLY with regulations (GDPR, HIPAA, PCI-DSS)
4. PROVE compliance to auditors
5. IDENTIFY misconfigured resources

CIS AWS Foundations Benchmark - Logging:
Section 3: Logging
- 3.1: CloudTrail enabled in all regions
- 3.2: CloudTrail log file validation
- 3.3: CloudTrail logs encrypted with KMS
- 3.4: CloudWatch log metric filter for root usage
- 3.5: VPC Flow Logs enabled
- 3.6: AWS Config enabled
- 3.7-3.15: CloudWatch metric filters and alarms
```

### Security Monitoring Architecture

```
┌──────────────────────────────────────────────────────────────┐
│                  Event Sources                               │
│  CloudTrail | VPC Flow Logs | ALB Logs | CloudFront Logs     │
│  RDS Logs | S3 Access Logs | Lambda Logs | EKS Logs          │
└────────────────────┬─────────────────────────────────────────┘
                     │
┌────────────────────▼─────────────────────────────────────────┐
│               Collection & Storage                           │
│  CloudWatch Logs | S3 | OpenSearch (ES)                      │
└────────────────────┬─────────────────────────────────────────┘
                     │
┌────────────────────▼─────────────────────────────────────────┐
│               Analysis & Correlation                         │
│  CloudWatch Insights | Athena | AWS Detective                │
└────────────────────┬─────────────────────────────────────────┘
                     │
┌────────────────────▼─────────────────────────────────────────┐
│               Alerting & Response                            │
│  SNS | PagerDuty | Slack | AWS Incident Manager              │
└──────────────────────────────────────────────────────────────┘
```

---

## ขั้นตอนที่ 862: CloudTrail Misconfigurations

### ❌ Vulnerable - CloudTrail Issues

```hcl
# ❌ Issue 1: CloudTrail not enabled at all
# (ไม่มี aws_cloudtrail resource)

# ❌ Issue 2: Single-region only
resource "aws_cloudtrail" "single_region" {
  name           = "audit-trail"
  s3_bucket_name = aws_s3_bucket.cloudtrail.id
  
  is_multi_region_trail = false  # ❌ เฉพาะ region เดียว
  # ❌ กิจกรรมใน regions อื่นไม่ถูกบันทึก
}

# ❌ Issue 3: No log file validation
resource "aws_cloudtrail" "no_validation" {
  name           = "audit-trail"
  s3_bucket_name = aws_s3_bucket.cloudtrail.id
  
  enable_log_file_validation = false  # ❌ Logs สามารถถูก tamper ได้
}

# ❌ Issue 4: No CloudWatch integration
resource "aws_cloudtrail" "no_cloudwatch" {
  name           = "audit-trail"
  s3_bucket_name = aws_s3_bucket.cloudtrail.id
  
  # ❌ ไม่มี cloud_watch_logs_group_arn
  # ❌ ไม่สามารถ set CloudWatch alarms ได้
}

# ❌ Issue 5: S3 data events not configured
resource "aws_cloudtrail" "no_data_events" {
  name                          = "audit-trail"
  s3_bucket_name                = aws_s3_bucket.cloudtrail.id
  include_global_service_events = true
  is_multi_region_trail         = true
  
  # ❌ ไม่มี event_selector with data_resource
  # ❌ S3 object access ไม่ถูกบันทึก
}
```

### ✅ Secure - Complete CloudTrail Configuration

```hcl
# ✅ SECURE - CloudTrail bucket
resource "aws_s3_bucket" "cloudtrail" {
  bucket = "company-cloudtrail-logs-${var.account_id}"
}

resource "aws_s3_bucket_public_access_block" "cloudtrail" {
  bucket                  = aws_s3_bucket.cloudtrail.id
  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}

# ✅ S3 bucket policy for CloudTrail
resource "aws_s3_bucket_policy" "cloudtrail" {
  bucket = aws_s3_bucket.cloudtrail.id
  
  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Sid    = "AWSCloudTrailAclCheck"
        Effect = "Allow"
        Principal = {
          Service = "cloudtrail.amazonaws.com"
        }
        Action   = "s3:GetBucketAcl"
        Resource = aws_s3_bucket.cloudtrail.arn
        Condition = {
          StringEquals = {
            "aws:SourceArn" = "arn:aws:cloudtrail:${var.region}:${var.account_id}:trail/main-trail"
          }
        }
      },
      {
        Sid    = "AWSCloudTrailWrite"
        Effect = "Allow"
        Principal = {
          Service = "cloudtrail.amazonaws.com"
        }
        Action   = "s3:PutObject"
        Resource = "${aws_s3_bucket.cloudtrail.arn}/AWSLogs/*"
        Condition = {
          StringEquals = {
            "s3:x-amz-acl" = "bucket-owner-full-control"
            "aws:SourceArn" = "arn:aws:cloudtrail:${var.region}:${var.account_id}:trail/main-trail"
          }
        }
      },
      {
        Sid       = "DenyNonSSL"
        Effect    = "Deny"
        Principal = "*"
        Action    = "s3:*"
        Resource = [
          aws_s3_bucket.cloudtrail.arn,
          "${aws_s3_bucket.cloudtrail.arn}/*"
        ]
        Condition = {
          Bool = {
            "aws:SecureTransport" = "false"
          }
        }
      }
    ]
  })
}

# ✅ CloudWatch Log Group
resource "aws_cloudwatch_log_group" "cloudtrail" {
  name              = "/aws/cloudtrail"
  retention_in_days = 365  # ✅ 1 year retention
  kms_key_id        = aws_kms_key.cloudtrail.arn
}

# ✅ Complete CloudTrail
resource "aws_cloudtrail" "main" {
  name                          = "main-audit-trail"
  s3_bucket_name               = aws_s3_bucket.cloudtrail.id
  
  # ✅ KMS encryption
  kms_key_id = aws_kms_key.cloudtrail.arn
  
  # ✅ All regions
  is_multi_region_trail = true
  
  # ✅ Include global services (IAM, STS, etc.)
  include_global_service_events = true
  
  # ✅ Log file integrity validation
  enable_log_file_validation = true
  
  # ✅ CloudWatch Logs integration
  cloud_watch_logs_group_arn = "${aws_cloudwatch_log_group.cloudtrail.arn}:*"
  cloud_watch_logs_role_arn  = aws_iam_role.cloudtrail.arn
  
  # ✅ Management events
  event_selector {
    read_write_type           = "All"   # ✅ Both read and write
    include_management_events = true    # ✅ Management API calls
    
    # ✅ S3 data events
    data_resource {
      type   = "AWS::S3::Object"
      values = ["arn:aws:s3:::"]  # All S3 buckets
    }
    
    # ✅ Lambda invocations
    data_resource {
      type   = "AWS::Lambda::Function"
      values = ["arn:aws:lambda"]
    }
  }
  
  # ✅ Insights events (anomaly detection)
  insight_selector {
    insight_type = "ApiCallRateInsight"
  }
  
  insight_selector {
    insight_type = "ApiErrorRateInsight"
  }
  
  tags = {
    Name        = "main-audit-trail"
    Environment = var.environment
  }
}

# ✅ CIS Benchmark CloudWatch Alarms (3.1-3.14)
locals {
  cis_alarms = {
    "3.1-RootUsage" = {
      description = "Root account usage"
      pattern     = "{$.userIdentity.type = \"Root\" && $.userIdentity.invokedBy NOT EXISTS && $.eventType != \"AwsServiceEvent\"}"
    }
    "3.2-UnauthorizedAPICalls" = {
      description = "Unauthorized API calls"
      pattern     = "{($.errorCode = \"*UnauthorizedAccess\") || ($.errorCode = \"AccessDenied\")}"
    }
    "3.3-NoMFAConsoleSignin" = {
      description = "Console signin without MFA"
      pattern     = "{$.eventName = \"ConsoleLogin\" && $.additionalEventData.MFAUsed != \"Yes\"}"
    }
    "3.4-IAMPolicyChanges" = {
      description = "IAM policy changes"
      pattern     = "{($.eventName=DeleteGroupPolicy)||($.eventName=DeleteRolePolicy)||($.eventName=DeleteUserPolicy)||($.eventName=PutGroupPolicy)||($.eventName=PutRolePolicy)||($.eventName=PutUserPolicy)||($.eventName=CreatePolicy)||($.eventName=DeletePolicy)||($.eventName=CreatePolicyVersion)||($.eventName=DeletePolicyVersion)||($.eventName=SetDefaultPolicyVersion)||($.eventName=AttachRolePolicy)||($.eventName=DetachRolePolicy)||($.eventName=AttachUserPolicy)||($.eventName=DetachUserPolicy)||($.eventName=AttachGroupPolicy)||($.eventName=DetachGroupPolicy)}"
    }
    "3.5-CloudTrailConfigChanges" = {
      description = "CloudTrail configuration changes"
      pattern     = "{($.eventName = CreateTrail) || ($.eventName = UpdateTrail) || ($.eventName = DeleteTrail) || ($.eventName = StartLogging) || ($.eventName = StopLogging)}"
    }
    "3.6-ConsoleAuthFailures" = {
      description = "AWS Management Console authentication failures"
      pattern     = "{($.eventName = ConsoleLogin) && ($.errorMessage = \"Failed authentication\")}"
    }
    "3.7-DisableDeleteKMSKey" = {
      description = "Disabling or scheduled deletion of CMK"
      pattern     = "{($.eventSource = kms.amazonaws.com) && (($.eventName=DisableKey)||($.eventName=ScheduleKeyDeletion))}"
    }
    "3.8-S3BucketPolicyChanges" = {
      description = "S3 bucket policy changes"
      pattern     = "{($.eventSource = s3.amazonaws.com) && (($.eventName = PutBucketAcl) || ($.eventName = PutBucketPolicy) || ($.eventName = PutBucketCors) || ($.eventName = PutBucketLifecycle) || ($.eventName = PutBucketReplication) || ($.eventName = DeleteBucketPolicy) || ($.eventName = DeleteBucketCors) || ($.eventName = DeleteBucketLifecycle) || ($.eventName = DeleteBucketReplication))}"
    }
    "3.9-AWSConfigChanges" = {
      description = "AWS Config configuration changes"
      pattern     = "{($.eventSource = config.amazonaws.com) && (($.eventName=StopConfigurationRecorder)||($.eventName=DeleteDeliveryChannel)||($.eventName=PutDeliveryChannel)||($.eventName=PutConfigurationRecorder))}"
    }
    "3.10-SecurityGroupChanges" = {
      description = "Security group changes"
      pattern     = "{($.eventName = AuthorizeSecurityGroupIngress) || ($.eventName = AuthorizeSecurityGroupEgress) || ($.eventName = RevokeSecurityGroupIngress) || ($.eventName = RevokeSecurityGroupEgress) || ($.eventName = CreateSecurityGroup) || ($.eventName = DeleteSecurityGroup)}"
    }
    "3.11-NetworkACLChanges" = {
      description = "Network ACL changes"
      pattern     = "{($.eventName = CreateNetworkAcl) || ($.eventName = CreateNetworkAclEntry) || ($.eventName = DeleteNetworkAcl) || ($.eventName = DeleteNetworkAclEntry) || ($.eventName = ReplaceNetworkAclEntry) || ($.eventName = ReplaceNetworkAclAssociation)}"
    }
    "3.12-NetworkGatewayChanges" = {
      description = "Network gateway changes"
      pattern     = "{($.eventName = CreateCustomerGateway) || ($.eventName = DeleteCustomerGateway) || ($.eventName = AttachInternetGateway) || ($.eventName = CreateInternetGateway) || ($.eventName = DeleteInternetGateway) || ($.eventName = DetachInternetGateway)}"
    }
    "3.13-RouteTableChanges" = {
      description = "Route table changes"
      pattern     = "{($.eventSource = ec2.amazonaws.com) && (($.eventName = CreateRoute) || ($.eventName = CreateRouteTable) || ($.eventName = ReplaceRoute) || ($.eventName = ReplaceRouteTableAssociation) || ($.eventName = DeleteRouteTable) || ($.eventName = DeleteRoute) || ($.eventName = DisassociateRouteTable))}"
    }
    "3.14-VPCChanges" = {
      description = "VPC changes"
      pattern     = "{($.eventName = CreateVpc) || ($.eventName = DeleteVpc) || ($.eventName = ModifyVpcAttribute) || ($.eventName = AcceptVpcPeeringConnection) || ($.eventName = CreateVpcPeeringConnection) || ($.eventName = DeleteVpcPeeringConnection) || ($.eventName = RejectVpcPeeringConnection) || ($.eventName = AttachClassicLinkVpc) || ($.eventName = DetachClassicLinkVpc) || ($.eventName = DisableVpcClassicLink) || ($.eventName = EnableVpcClassicLink)}"
    }
  }
}

resource "aws_cloudwatch_log_metric_filter" "cis_alarms" {
  for_each = local.cis_alarms
  
  name           = each.key
  pattern        = each.value.pattern
  log_group_name = aws_cloudwatch_log_group.cloudtrail.name
  
  metric_transformation {
    name      = each.key
    namespace = "CISAlarms"
    value     = "1"
  }
}

resource "aws_cloudwatch_metric_alarm" "cis_alarms" {
  for_each = local.cis_alarms
  
  alarm_name          = each.key
  alarm_description   = each.value.description
  comparison_operator = "GreaterThanOrEqualToThreshold"
  evaluation_periods  = 1
  metric_name         = each.key
  namespace           = "CISAlarms"
  period              = 300
  statistic           = "Sum"
  threshold           = 1
  treat_missing_data  = "notBreaching"
  
  alarm_actions = [aws_sns_topic.security_alerts.arn]
}
```

---

## ขั้นตอนที่ 863: VPC Flow Logs

```hcl
# ❌ VULNERABLE - ไม่มี Flow Logs (ดู Part 084 สำหรับรายละเอียด)

# ✅ SECURE - Flow Logs ครบทั้ง VPC, subnet, ENI

# Flow logs for each subnet
resource "aws_flow_log" "subnet_logs" {
  for_each = { for s in aws_subnet.private : s.id => s }
  
  iam_role_arn    = aws_iam_role.flow_logs.arn
  log_destination = "${aws_cloudwatch_log_group.vpc_flow_logs.arn}"
  traffic_type    = "ALL"
  subnet_id       = each.key  # ✅ Per-subnet granularity
  
  tags = {
    Name = "flow-logs-${each.key}"
  }
}

# ✅ Athena table for flow log analysis
resource "aws_glue_catalog_table" "vpc_flow_logs" {
  name          = "vpc_flow_logs"
  database_name = aws_glue_catalog_database.security.name
  
  table_type = "EXTERNAL_TABLE"
  
  parameters = {
    EXTERNAL              = "TRUE"
    "parquet.compression" = "SNAPPY"
  }
  
  storage_descriptor {
    location      = "s3://${aws_s3_bucket.flow_logs.id}/vpc-flow-logs/"
    input_format  = "org.apache.hadoop.hive.ql.io.parquet.MapredParquetInputFormat"
    output_format = "org.apache.hadoop.hive.ql.io.parquet.MapredParquetOutputFormat"
    
    ser_de_info {
      name                  = "ParquetHiveSerDe"
      serialization_library = "org.apache.hadoop.hive.ql.io.parquet.serde.ParquetHiveSerDe"
      
      parameters = {
        "serialization.format" = "1"
      }
    }
    
    columns {
      name = "version"
      type = "int"
    }
    columns {
      name = "account_id"
      type = "string"
    }
    columns {
      name = "interface_id"
      type = "string"
    }
    columns {
      name = "srcaddr"
      type = "string"
    }
    columns {
      name = "dstaddr"
      type = "string"
    }
    columns {
      name = "srcport"
      type = "int"
    }
    columns {
      name = "dstport"
      type = "int"
    }
    columns {
      name = "protocol"
      type = "bigint"
    }
    columns {
      name = "packets"
      type = "bigint"
    }
    columns {
      name = "bytes"
      type = "bigint"
    }
    columns {
      name = "action"
      type = "string"
    }
    columns {
      name = "log_status"
      type = "string"
    }
  }
}
```

---

## ขั้นตอนที่ 864: ALB & CloudFront Access Logs

### ❌ Vulnerable - No ALB Access Logs

```hcl
# ❌ VULNERABLE - ALB ไม่มี access logs
resource "aws_lb" "no_logs" {
  name               = "main-alb"
  internal           = false
  load_balancer_type = "application"
  subnets            = aws_subnet.public[*].id
  
  # ❌ ไม่มี access_logs block
}
```

### ✅ Secure - ALB Access Logs

```hcl
# ✅ SECURE - ALB with complete logging
resource "aws_s3_bucket" "alb_logs" {
  bucket = "company-alb-logs-${var.account_id}"
}

# ✅ ALB logs bucket policy (required by AWS)
data "aws_elb_service_account" "main" {}

resource "aws_s3_bucket_policy" "alb_logs" {
  bucket = aws_s3_bucket.alb_logs.id
  
  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Principal = {
          AWS = data.aws_elb_service_account.main.arn
        }
        Action   = "s3:PutObject"
        Resource = "${aws_s3_bucket.alb_logs.arn}/alb/*"
      },
      {
        Effect = "Allow"
        Principal = {
          Service = "delivery.logs.amazonaws.com"
        }
        Action   = "s3:PutObject"
        Resource = "${aws_s3_bucket.alb_logs.arn}/alb/*"
        Condition = {
          StringEquals = {
            "s3:x-amz-acl" = "bucket-owner-full-control"
          }
        }
      },
      {
        Effect = "Allow"
        Principal = {
          Service = "delivery.logs.amazonaws.com"
        }
        Action   = "s3:GetBucketAcl"
        Resource = aws_s3_bucket.alb_logs.arn
      }
    ]
  })
}

resource "aws_lb" "with_logs" {
  name               = "main-alb"
  internal           = false
  load_balancer_type = "application"
  subnets            = aws_subnet.public[*].id
  
  # ✅ Enable access logs
  access_logs {
    bucket  = aws_s3_bucket.alb_logs.id
    prefix  = "alb"
    enabled = true   # ✅ Must explicitly enable
  }
  
  # ✅ Deletion protection
  enable_deletion_protection = true
  
  # ✅ WAF association
  # (done separately with aws_wafv2_web_acl_association)
}
```

---

## ขั้นตอนที่ 865: RDS & Lambda Logs

### ❌ Vulnerable - RDS Without CloudWatch Logs

```hcl
# ❌ VULNERABLE - RDS ไม่ส่ง logs ไป CloudWatch
resource "aws_db_instance" "no_logs" {
  identifier = "main-database"
  engine     = "mysql"
  
  # ❌ ไม่มี enabled_cloudwatch_logs_exports
  # ❌ ไม่สามารถ query logs ใน CloudWatch
  # ❌ ไม่มี audit trail สำหรับ database
}
```

### ✅ Secure - RDS with CloudWatch Logs

```hcl
# ✅ SECURE - RDS with full logging
resource "aws_db_instance" "with_logs" {
  identifier     = "main-database"
  engine         = "mysql"
  engine_version = "8.0"
  instance_class = "db.t3.micro"
  
  # ✅ MySQL CloudWatch logs
  enabled_cloudwatch_logs_exports = [
    "general",    # ✅ General query log
    "error",      # ✅ Error log
    "slowquery",  # ✅ Slow query log (performance)
    "audit"       # ✅ Audit log (security)
  ]
  
  # ✅ Parameter group for slow query log
  parameter_group_name = aws_db_parameter_group.mysql_logging.name
  
  # ✅ Enhanced monitoring
  monitoring_interval = 60
  monitoring_role_arn = aws_iam_role.rds_monitoring.arn
  
  # ✅ Performance Insights
  performance_insights_enabled          = true
  performance_insights_retention_period = 7
}

# ✅ Parameter group สำหรับ MySQL logging
resource "aws_db_parameter_group" "mysql_logging" {
  name   = "mysql-logging"
  family = "mysql8.0"
  
  parameter {
    name  = "slow_query_log"
    value = "1"  # ✅ Enable slow query log
  }
  
  parameter {
    name  = "long_query_time"
    value = "2"  # ✅ Log queries > 2 seconds
  }
  
  parameter {
    name  = "general_log"
    value = "0"  # ⚠️ Disable general log in production (too verbose)
    # Enable only for debugging
  }
  
  parameter {
    name  = "server_audit_logging"
    value = "1"
  }
  
  parameter {
    name  = "server_audit_events"
    value = "CONNECT,QUERY_DDL,QUERY_DCL"  # ✅ Audit connections and DDL
  }
}

# ✅ Lambda CloudWatch Log Group
resource "aws_cloudwatch_log_group" "lambda" {
  name              = "/aws/lambda/${var.function_name}"
  retention_in_days = 90      # ✅ 90-day retention
  kms_key_id        = aws_kms_key.logs.arn  # ✅ Encrypted
  
  tags = {
    Environment = var.environment
  }
}

# ✅ Lambda function with log group
resource "aws_lambda_function" "with_logging" {
  filename      = "function.zip"
  function_name = var.function_name
  role          = aws_iam_role.lambda.arn
  handler       = "index.handler"
  runtime       = "nodejs18.x"
  
  # ✅ CloudWatch logs configuration
  logging_config {
    log_format = "JSON"  # ✅ Structured logging
    log_group  = aws_cloudwatch_log_group.lambda.name
  }
  
  # ✅ X-Ray tracing
  tracing_config {
    mode = "Active"  # ✅ Enable X-Ray
  }
  
  depends_on = [aws_cloudwatch_log_group.lambda]
}
```

---

## ขั้นตอนที่ 866: EKS Control Plane Logs

### ❌ Vulnerable - EKS Without Control Plane Logs

```hcl
# ❌ VULNERABLE - EKS ไม่มี control plane logs
resource "aws_eks_cluster" "no_logs" {
  name     = "production"
  role_arn = aws_iam_role.eks.arn
  
  vpc_config {
    subnet_ids = aws_subnet.private[*].id
  }
  
  # ❌ ไม่มี enabled_cluster_log_types
  # ❌ ไม่รู้ว่าใคร access API Server
  # ❌ ไม่มี audit trail สำหรับ Kubernetes actions
}
```

### ✅ Secure - EKS with All Log Types

```hcl
# ✅ SECURE - EKS with full logging
resource "aws_eks_cluster" "with_logs" {
  name     = "production"
  role_arn = aws_iam_role.eks.arn
  version  = "1.28"
  
  vpc_config {
    subnet_ids              = aws_subnet.private[*].id
    endpoint_private_access = true
    endpoint_public_access  = false
  }
  
  # ✅ Enable ALL control plane log types
  enabled_cluster_log_types = [
    "api",              # ✅ API Server logs
    "audit",            # ✅ Audit logs (who did what)
    "authenticator",    # ✅ IAM authentication
    "controllerManager", # ✅ Controller manager
    "scheduler"         # ✅ Scheduler
  ]
  
  # ✅ Secrets encryption
  encryption_config {
    resources = ["secrets"]
    provider {
      key_arn = aws_kms_key.eks.arn
    }
  }
}

# ✅ CloudWatch Log Group for EKS
resource "aws_cloudwatch_log_group" "eks" {
  name              = "/aws/eks/${var.cluster_name}/cluster"
  retention_in_days = 90
  kms_key_id        = aws_kms_key.logs.arn
}

# ✅ Query EKS audit logs
# Use CloudWatch Insights query:
# fields @timestamp, @message
# | filter @logStream = "kube-apiserver-audit"
# | filter @message like /verb.*delete/
# | sort @timestamp desc
# | limit 100
```

---

## ขั้นตอนที่ 867: GuardDuty & AWS Config

### ❌ Vulnerable - GuardDuty Not Enabled

```hcl
# ❌ VULNERABLE - ไม่มี GuardDuty
# GuardDuty ให้ threat detection จาก:
# - VPC Flow Logs
# - DNS logs
# - CloudTrail events
# - EKS audit logs
# - RDS login events
# - S3 data events
# ถ้าไม่เปิด = ไม่รู้ว่ามีการโจมตีเกิดขึ้น
```

### ✅ Secure - GuardDuty Fully Enabled

```hcl
# ✅ SECURE - GuardDuty with all data sources
resource "aws_guardduty_detector" "main" {
  enable = true
  
  # ✅ Anomaly detection sensitivity
  finding_publishing_frequency = "FIFTEEN_MINUTES"
  
  datasources {
    # ✅ S3 protection
    s3_logs {
      enable = true
    }
    
    # ✅ Kubernetes protection
    kubernetes {
      audit_logs {
        enable = true
      }
    }
    
    # ✅ EC2 malware protection
    malware_protection {
      scan_ec2_instance_with_findings {
        ebs_volumes {
          enable = true
        }
      }
    }
    
    # ✅ RDS protection
    rds_login_events {
      enable = true
    }
    
    # ✅ Lambda protection
    lambda {
      network_logs {
        enable = true
      }
    }
  }
}

# ✅ GuardDuty SNS alerting
resource "aws_cloudwatch_event_rule" "guardduty_high" {
  name        = "guardduty-high-severity"
  description = "GuardDuty HIGH/CRITICAL findings"
  
  event_pattern = jsonencode({
    source      = ["aws.guardduty"]
    detail-type = ["GuardDuty Finding"]
    detail = {
      severity = [{ numeric = [">=", 7.0] }]
    }
  })
}

resource "aws_cloudwatch_event_target" "guardduty_sns" {
  rule = aws_cloudwatch_event_rule.guardduty_high.name
  arn  = aws_sns_topic.security_alerts.arn
  
  input_transformer {
    input_paths = {
      severity    = "$.detail.severity"
      title       = "$.detail.title"
      description = "$.detail.description"
      type        = "$.detail.type"
      region      = "$.region"
      account     = "$.account"
    }
    
    input_template = <<EOF
{
  "alert": "GuardDuty High Severity Finding",
  "severity": <severity>,
  "title": <title>,
  "description": <description>,
  "type": <type>,
  "region": <region>,
  "account": <account>
}
EOF
  }
}

# ✅ GuardDuty in Organization (all member accounts)
resource "aws_guardduty_organization_configuration" "main" {
  auto_enable = "ALL"  # ✅ Auto-enable for new accounts
  detector_id = aws_guardduty_detector.main.id
  
  datasources {
    s3_logs { auto_enable = true }
    
    kubernetes {
      audit_logs { enable = true }
    }
    
    malware_protection {
      scan_ec2_instance_with_findings {
        ebs_volumes { auto_enable = true }
      }
    }
  }
}

# ✅ AWS Config for compliance monitoring
resource "aws_config_configuration_recorder" "main" {
  name     = "default"
  role_arn = aws_iam_role.config.arn
  
  recording_group {
    all_supported                 = true  # ✅ Record all resource types
    include_global_resource_types = true  # ✅ IAM etc.
    
    # ✅ Exclusions (optional, for performance)
    exclusion_by_resource_types {
      resource_types = []  # No exclusions for security
    }
  }
  
  recording_mode {
    recording_frequency = "CONTINUOUS"  # ✅ Real-time
    
    # ✅ Daily for less important resources
    recording_mode_override {
      description         = "Daily for EC2 instance volume"
      resource_types      = ["AWS::EC2::Volume"]
      recording_frequency = "DAILY"
    }
  }
}

resource "aws_config_delivery_channel" "main" {
  name           = "default"
  s3_bucket_name = aws_s3_bucket.config.id
  
  # ✅ SNS for change notifications
  sns_topic_arn = aws_sns_topic.config_changes.arn
  
  snapshot_delivery_properties {
    delivery_frequency = "TwentyFour_Hours"
  }
  
  depends_on = [aws_config_configuration_recorder.main]
}

resource "aws_config_configuration_recorder_status" "main" {
  name       = aws_config_configuration_recorder.main.name
  is_enabled = true
  
  depends_on = [aws_config_delivery_channel.main]
}
```

---

## ขั้นตอนที่ 868: Route53 & S3 Access Logs

### Route53 Query Logging

```hcl
# ❌ VULNERABLE - Route53 ไม่มี query logs
data "aws_route53_zone" "main" {
  name = var.domain_name
}

# ❌ ไม่มี aws_route53_query_log = ไม่รู้ว่ามี DNS enumeration

# ✅ SECURE - Route53 query logging
resource "aws_cloudwatch_log_group" "route53" {
  # Route53 query logs MUST be in us-east-1
  provider = aws.us_east_1
  
  name              = "/aws/route53/${var.domain_name}"
  retention_in_days = 90
}

resource "aws_cloudwatch_log_resource_policy" "route53" {
  provider    = aws.us_east_1
  policy_name = "route53-query-logging"
  
  policy_document = jsonencode({
    Statement = [{
      Effect = "Allow"
      Principal = {
        Service = "route53.amazonaws.com"
      }
      Action = [
        "logs:CreateLogStream",
        "logs:PutLogEvents"
      ]
      Resource = "arn:aws:logs:us-east-1:${var.account_id}:log-group:/aws/route53/*"
      Condition = {
        ArnLike = {
          "aws:SourceArn" = "arn:aws:route53:::hostedzone/${data.aws_route53_zone.main.zone_id}"
        }
      }
    }]
  })
}

resource "aws_route53_query_log" "main" {
  depends_on = [aws_cloudwatch_log_resource_policy.route53]
  
  cloudwatch_log_group_arn = aws_cloudwatch_log_group.route53.arn
  zone_id                  = data.aws_route53_zone.main.zone_id
}
```

---

## ขั้นตอนที่ 869: Security Hub Centralized Monitoring

### ❌ Vulnerable - No Centralized Security View

```hcl
# ❌ VULNERABLE - ไม่มี Security Hub
# ไม่รู้ comprehensive view ของ security posture
```

### ✅ Secure - Security Hub with All Standards

```hcl
# ✅ SECURE - AWS Security Hub
resource "aws_securityhub_account" "main" {
  enable_default_standards  = false  # ✅ Manually control which standards
  control_finding_generator = "SECURITY_CONTROL"  # ✅ New format
  
  auto_enable_controls = true
}

# ✅ Enable CIS Benchmark
resource "aws_securityhub_standards_subscription" "cis_14" {
  standards_arn = "arn:aws:securityhub:${var.region}::standards/cis-aws-foundations-benchmark/v/1.4.0"
  depends_on    = [aws_securityhub_account.main]
}

# ✅ Enable AWS Foundational Security
resource "aws_securityhub_standards_subscription" "aws_fsbp" {
  standards_arn = "arn:aws:securityhub:${var.region}::standards/aws-foundational-security-best-practices/v/1.0.0"
  depends_on    = [aws_securityhub_account.main]
}

# ✅ Enable PCI-DSS
resource "aws_securityhub_standards_subscription" "pci_dss" {
  standards_arn = "arn:aws:securityhub:${var.region}::standards/pci-dss/v/3.2.1"
  depends_on    = [aws_securityhub_account.main]
}

# ✅ Security Hub organization settings
resource "aws_securityhub_organization_configuration" "main" {
  auto_enable           = true  # ✅ Auto-enable for new accounts
  auto_enable_standards = "NONE"
}

# ✅ Custom action for incident response
resource "aws_securityhub_action_target" "disable_iam_user" {
  name        = "Disable IAM User"
  identifier  = "DisableIAMUser"
  description = "Disables compromised IAM user"
}

# ✅ EventBridge for Security Hub findings → Lambda remediation
resource "aws_cloudwatch_event_rule" "security_hub_critical" {
  name        = "security-hub-critical"
  description = "Critical Security Hub findings"
  
  event_pattern = jsonencode({
    source      = ["aws.securityhub"]
    detail-type = ["Security Hub Findings - Imported"]
    detail = {
      findings = {
        Severity = {
          Label = ["CRITICAL", "HIGH"]
        }
        Compliance = {
          Status = ["FAILED"]
        }
        Workflow = {
          Status = ["NEW"]
        }
      }
    }
  })
}

resource "aws_cloudwatch_event_target" "security_hub_lambda" {
  rule = aws_cloudwatch_event_rule.security_hub_critical.name
  arn  = aws_lambda_function.auto_remediate.arn
}
```

---

## ขั้นตอนที่ 870: Complete Monitoring Dashboard

### CloudWatch Dashboard สำหรับ Security

```hcl
# ✅ Security Operations Dashboard
resource "aws_cloudwatch_dashboard" "security_ops" {
  dashboard_name = "SecurityOperations"
  
  dashboard_body = jsonencode({
    widgets = [
      # Row 1: Critical Alerts
      {
        type   = "alarm"
        x      = 0
        y      = 0
        width  = 24
        height = 3
        properties = {
          title  = "Security Alarms"
          alarms = [
            aws_cloudwatch_metric_alarm.cis_alarms["3.1-RootUsage"].arn,
            aws_cloudwatch_metric_alarm.cis_alarms["3.2-UnauthorizedAPICalls"].arn,
            aws_cloudwatch_metric_alarm.cis_alarms["3.3-NoMFAConsoleSignin"].arn
          ]
        }
      },
      # Row 2: GuardDuty Findings
      {
        type   = "metric"
        x      = 0
        y      = 3
        width  = 12
        height = 6
        properties = {
          metrics = [
            ["AWS/GuardDuty", "FindingCount", "DetectorId", aws_guardduty_detector.main.id, "Severity", "HIGH"],
            ["AWS/GuardDuty", "FindingCount", "DetectorId", aws_guardduty_detector.main.id, "Severity", "MEDIUM"]
          ]
          period = 300
          stat   = "Sum"
          title  = "GuardDuty Findings"
        }
      },
      # Row 2: Failed Auth Attempts
      {
        type   = "metric"
        x      = 12
        y      = 3
        width  = 12
        height = 6
        properties = {
          metrics = [
            ["CISAlarms", "3.6-ConsoleAuthFailures"],
            ["CISAlarms", "3.2-UnauthorizedAPICalls"]
          ]
          period = 300
          stat   = "Sum"
          title  = "Failed Authentication Attempts"
        }
      },
      # Row 3: VPC Flow Logs
      {
        type   = "metric"
        x      = 0
        y      = 9
        width  = 12
        height = 6
        properties = {
          metrics = [
            ["VPCFlowMetrics", "RejectedConnectionsCount"]
          ]
          period = 300
          stat   = "Sum"
          title  = "Rejected Network Connections"
        }
      }
    ]
  })
}

# ✅ Summary: logging resources
output "logging_resources" {
  description = "All logging resources"
  value = {
    cloudtrail_bucket       = aws_s3_bucket.cloudtrail.id
    cloudtrail_log_group    = aws_cloudwatch_log_group.cloudtrail.name
    vpc_flow_log_group      = aws_cloudwatch_log_group.vpc_flow_logs.name
    alb_logs_bucket         = aws_s3_bucket.alb_logs.id
    guardduty_detector_id   = aws_guardduty_detector.main.id
    security_hub_enabled    = true
    config_recorder         = aws_config_configuration_recorder.main.name
  }
}
```

---

## สรุป Logging & Monitoring

### Checkov Rules สำหรับ Logging

| Service | Issue | Checkov Rule | CIS Control |
|---------|-------|-------------|-------------|
| CloudTrail | Not enabled | CKV_AWS_35 | 3.1 |
| CloudTrail | Single region | CKV_AWS_67 | 3.1 |
| CloudTrail | No validation | CKV_AWS_36 | 3.2 |
| CloudTrail | No KMS | CKV_AWS_35 | 3.7 |
| VPC Flow Logs | Not enabled | CKV_AWS_73 | 2.9 |
| ALB | No access logs | CKV_AWS_91 | - |
| CloudFront | No access logs | CKV_AWS_86 | - |
| RDS | No CW logs | CKV_AWS_129 | - |
| EKS | No control logs | CKV_AWS_58 | - |
| GuardDuty | Not enabled | CKV_AWS_238 | - |
| Config | Not enabled | CKV_AWS_144 | - |

---

*Part 087 ครอบคลุม Logging & Monitoring ทั้งหมด - ต่อไปใน Part 088 จะเจาะลึก Public Exposure Vulnerabilities*
