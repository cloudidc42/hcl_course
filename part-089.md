# Part 089: Compliance Frameworks: CIS, NIST, SOC2
## ขั้นตอนที่ 881-890: การใช้ Terraform สำหรับ Compliance Frameworks

---

## ขั้นตอนที่ 881: ภาพรวม Compliance Frameworks

### ทำไม Compliance สำคัญสำหรับ IaC

```
Compliance Requirements ใน Cloud:
1. Legal requirements (GDPR, PDPA)
2. Industry regulations (HIPAA, PCI-DSS, SOX)
3. Customer requirements
4. Cyber insurance requirements
5. Government contracts

Compliance + IaC:
✅ Automate compliance checks
✅ Version-controlled compliance policies
✅ Consistent enforcement across environments
✅ Audit trail through git history
✅ Repeatable compliance evidence

Without IaC Compliance:
❌ Manual checks = human error
❌ Inconsistent across environments
❌ No audit trail
❌ Expensive audit preparation
```

### Compliance Framework Mapping

```
CIS AWS Foundations Benchmark v1.5 → Technical controls
NIST SP 800-53 → Federal standards
SOC 2 Type II → Service organization controls
PCI-DSS → Payment card security
HIPAA → Healthcare data
ISO 27001 → Information security management
GDPR/PDPA → Privacy regulations
```

---

## ขั้นตอนที่ 882: CIS AWS Foundations Benchmark

### Section 1: IAM Controls

```hcl
# CIS 1.1: Maintain current contact details
# (Manual - สำหรับ AWS Account)

# CIS 1.4: Ensure no root account access key exists
resource "aws_cloudwatch_log_metric_filter" "root_access_key_usage" {
  name           = "RootAccessKeyUsage"
  pattern        = "{$.userIdentity.type = \"Root\" && $.userIdentity.invokedBy NOT EXISTS && $.eventType != \"AwsServiceEvent\"}"
  log_group_name = aws_cloudwatch_log_group.cloudtrail.name
  
  metric_transformation {
    name      = "RootAccessKeyUsage"
    namespace = "CISBenchmark"
    value     = "1"
  }
}

resource "aws_cloudwatch_metric_alarm" "cis_1_4" {
  alarm_name          = "CIS-1.4-RootAccountUsage"
  alarm_description   = "CIS 1.4: Root account activity detected"
  comparison_operator = "GreaterThanOrEqualToThreshold"
  evaluation_periods  = 1
  metric_name         = "RootAccessKeyUsage"
  namespace           = "CISBenchmark"
  period              = 300
  statistic           = "Sum"
  threshold           = 1
  treat_missing_data  = "notBreaching"
  alarm_actions       = [aws_sns_topic.security_alerts.arn]
}

# CIS 1.5: Ensure MFA is enabled for the root account
# (Verify via AWS Console - cannot configure via Terraform)

# CIS 1.8: Ensure IAM password policy requires minimum length of 14
resource "aws_iam_account_password_policy" "cis_compliant" {
  minimum_password_length        = 14   # CIS 1.8
  require_uppercase_characters   = true  # CIS 1.9
  require_lowercase_characters   = true  # CIS 1.10
  require_numbers                = true  # CIS 1.11
  require_symbols                = true  # CIS 1.12
  allow_users_to_change_password = true  # CIS 1.7
  max_password_age               = 365   # CIS 1.13 (some interpret as needed)
  password_reuse_prevention      = 24    # CIS 1.14
  hard_expiry                    = false
}

# CIS 1.16: Ensure IAM policies are attached only to groups or roles
resource "aws_config_config_rule" "cis_1_16" {
  name        = "cis-1-16-iam-no-direct-user-policy"
  description = "CIS 1.16: IAM policies should not be attached directly to users"
  
  source {
    owner             = "AWS"
    source_identifier = "IAM_USER_NO_POLICIES_CHECK"
  }
}

# CIS 1.17: Ensure a support role has been created
resource "aws_iam_role" "support_access" {
  name        = "SupportAccess"
  description = "CIS 1.17: Support access role"
  
  assume_role_policy = jsonencode({
    Statement = [{
      Effect = "Allow"
      Principal = {
        AWS = "arn:aws:iam::${var.account_id}:root"
      }
      Action = "sts:AssumeRole"
      Condition = {
        Bool = {
          "aws:MultiFactorAuthPresent" = "true"
        }
      }
    }]
  })
}

resource "aws_iam_role_policy_attachment" "support_access" {
  role       = aws_iam_role.support_access.name
  policy_arn = "arn:aws:iam::aws:policy/AWSSupportAccess"  # CIS 1.17
}

# CIS 1.19: Ensure that expired SSL/TLS certificates are removed
resource "aws_cloudwatch_metric_alarm" "cis_1_19_cert_expiry" {
  alarm_name          = "CIS-1.19-CertificateExpiring"
  comparison_operator = "LessThanThreshold"
  evaluation_periods  = 1
  metric_name         = "DaysToExpiry"
  namespace           = "AWS/CertificateManager"
  period              = 86400
  statistic           = "Minimum"
  threshold           = 30
  
  dimensions = {
    CertificateArn = aws_acm_certificate.main.arn
  }
  
  alarm_description = "CIS 1.19: SSL certificate expiring within 30 days"
  alarm_actions     = [aws_sns_topic.security_alerts.arn]
}

# CIS 1.20: Ensure that IAM Access analyzer is enabled
resource "aws_accessanalyzer_analyzer" "cis_1_20" {
  analyzer_name = "cis-access-analyzer"
  type          = "ACCOUNT"
  
  tags = {
    CISControl = "1.20"
  }
}
```

### Section 2: Storage Controls

```hcl
# CIS 2.1.1: Ensure that S3 Buckets have Block public access enabled
module "s3_public_access_block" {
  for_each = var.s3_buckets
  
  source = "./modules/secure-s3-bucket"
  
  bucket_name = each.value
}

# CIS 2.1.2: Ensure S3 Bucket Policy is set to deny HTTP requests
resource "aws_s3_bucket_policy" "cis_2_1_2" {
  for_each = var.s3_buckets
  
  bucket = each.value
  
  policy = jsonencode({
    Statement = [{
      Sid       = "DenyHTTP"
      Effect    = "Deny"
      Principal = "*"
      Action    = "s3:*"
      Resource  = ["arn:aws:s3:::${each.value}", "arn:aws:s3:::${each.value}/*"]
      Condition = {
        Bool = {
          "aws:SecureTransport" = "false"
        }
      }
    }]
  })
}

# CIS 2.1.3: Ensure MFA Delete is enabled on S3 buckets
# (Manual - ต้องใช้ root MFA)

# CIS 2.2.1: Ensure EBS Volume Encryption is Enabled
resource "aws_ebs_encryption_by_default" "cis_2_2_1" {
  enabled = true  # CIS 2.2.1
}

# CIS 2.3.1: Ensure that encryption is enabled for RDS Instances
resource "aws_config_config_rule" "cis_2_3_1" {
  name        = "cis-2-3-1-rds-storage-encrypted"
  description = "CIS 2.3.1: RDS storage encryption"
  
  source {
    owner             = "AWS"
    source_identifier = "RDS_STORAGE_ENCRYPTED"
  }
}
```

### Section 3: Logging Controls

```hcl
# CIS 3.1: Ensure CloudTrail is enabled in all regions
resource "aws_cloudtrail" "cis_3_1" {
  name                          = "cis-audit-trail"
  s3_bucket_name               = aws_s3_bucket.cloudtrail.id
  kms_key_id                   = aws_kms_key.cloudtrail.arn
  is_multi_region_trail         = true   # CIS 3.1
  include_global_service_events = true   # CIS 3.1
  enable_log_file_validation    = true   # CIS 3.2
  
  cloud_watch_logs_group_arn = "${aws_cloudwatch_log_group.cloudtrail.arn}:*"
  cloud_watch_logs_role_arn  = aws_iam_role.cloudtrail.arn
  
  event_selector {
    read_write_type           = "All"
    include_management_events = true
  }
}

# CIS 3.3: Ensure CloudTrail logs are encrypted at rest using KMS CMKs
# (Done via kms_key_id in cloudtrail above)

# CIS 3.4: Ensure CloudTrail log file validation is enabled
# (Done via enable_log_file_validation = true above)

# CIS 3.5: Ensure AWS Config is enabled in all regions
resource "aws_config_configuration_recorder" "cis_3_5" {
  name     = "default"
  role_arn = aws_iam_role.config.arn
  
  recording_group {
    all_supported                 = true
    include_global_resource_types = true
  }
}

# CIS 3.6: Ensure S3 bucket access logging is enabled
resource "aws_config_config_rule" "cis_3_6" {
  name        = "cis-3-6-s3-bucket-logging-enabled"
  description = "CIS 3.6: S3 bucket logging enabled"
  
  source {
    owner             = "AWS"
    source_identifier = "S3_BUCKET_LOGGING_ENABLED"
  }
}

# CIS 3.7-3.14: CloudWatch Metric Filters and Alarms
# (ดูใน Part 087 สำหรับรายละเอียด)
```

### Section 4: Monitoring Controls

```hcl
# CIS 4.1: Ensure a log metric filter and alarm exist for unauthorized API calls
resource "aws_cloudwatch_log_metric_filter" "cis_4_1" {
  name           = "CIS-4.1-UnauthorizedAPICalls"
  pattern        = "{($.errorCode = \"*UnauthorizedAccess\") || ($.errorCode = \"AccessDenied\")}"
  log_group_name = aws_cloudwatch_log_group.cloudtrail.name
  
  metric_transformation {
    name      = "UnauthorizedAPICalls"
    namespace = "CISBenchmark"
    value     = "1"
  }
}

resource "aws_cloudwatch_metric_alarm" "cis_4_1" {
  alarm_name          = "CIS-4.1-UnauthorizedAPICalls"
  alarm_description   = "CIS 4.1: Unauthorized API calls"
  comparison_operator = "GreaterThanOrEqualToThreshold"
  evaluation_periods  = 1
  metric_name         = "UnauthorizedAPICalls"
  namespace           = "CISBenchmark"
  period              = 300
  statistic           = "Sum"
  threshold           = 1
  alarm_actions       = [aws_sns_topic.security_alerts.arn]
}

# CIS 4.2: Console signin without MFA
resource "aws_cloudwatch_log_metric_filter" "cis_4_2" {
  name           = "CIS-4.2-NoMFAConsoleSignin"
  pattern        = "{$.eventName = \"ConsoleLogin\" && $.additionalEventData.MFAUsed != \"Yes\"}"
  log_group_name = aws_cloudwatch_log_group.cloudtrail.name
  
  metric_transformation {
    name      = "NoMFAConsoleSignin"
    namespace = "CISBenchmark"
    value     = "1"
  }
}

resource "aws_cloudwatch_metric_alarm" "cis_4_2" {
  alarm_name          = "CIS-4.2-NoMFAConsoleSignin"
  comparison_operator = "GreaterThanOrEqualToThreshold"
  evaluation_periods  = 1
  metric_name         = "NoMFAConsoleSignin"
  namespace           = "CISBenchmark"
  period              = 300
  statistic           = "Sum"
  threshold           = 1
  alarm_actions       = [aws_sns_topic.security_alerts.arn]
}
```

### Section 5: Networking Controls

```hcl
# CIS 5.1: No SGs allow SSH from 0.0.0.0/0
resource "aws_config_config_rule" "cis_5_1" {
  name        = "cis-5-1-restricted-ssh"
  description = "CIS 5.1: SSH restricted"
  
  source {
    owner             = "AWS"
    source_identifier = "INCOMING_SSH_DISABLED"
  }
}

# CIS 5.2: No SGs allow RDP from 0.0.0.0/0
resource "aws_config_config_rule" "cis_5_2" {
  name        = "cis-5-2-restricted-rdp"
  description = "CIS 5.2: RDP restricted"
  
  source {
    owner             = "AWS"
    source_identifier = "RESTRICTED_INCOMING_TRAFFIC"
  }
  
  input_parameters = jsonencode({
    blockedPort1 = "3389"
  })
}

# CIS 5.3: Default security group restricts all traffic
resource "aws_default_security_group" "cis_5_3" {
  vpc_id = aws_vpc.main.id
  
  # ✅ No ingress/egress rules = deny all
  
  tags = {
    Name       = "default-restricted"
    CISControl = "5.3"
  }
}

# CIS 5.4: VPC peering routing tables are least access
# (Manual review + AWS Config)

# CIS 5.5: Ensure routing tables for VPC peering are "least access"
resource "aws_config_config_rule" "cis_5_5" {
  name        = "cis-5-5-vpc-flow-logs"
  description = "CIS 5.5: VPC Flow Logs"
  
  source {
    owner             = "AWS"
    source_identifier = "VPC_FLOW_LOGS_ENABLED"
  }
}
```

---

## ขั้นตอนที่ 883: NIST SP 800-53 Implementation

### Control Families

```hcl
# AC - Access Control

# AC-2: Account Management
resource "aws_config_config_rule" "nist_ac_2" {
  name        = "nist-ac-2-iam-user-mfa-enabled"
  description = "NIST AC-2: MFA enabled for IAM users"
  
  source {
    owner             = "AWS"
    source_identifier = "MFA_ENABLED_FOR_IAM_CONSOLE_ACCESS"
  }
}

# AC-3: Access Enforcement (Least Privilege)
# Implemented via IAM policies with specific actions/resources

# AC-17: Remote Access
resource "aws_ssm_session_manager_preferences" "nist_ac_17" {
  # Session Manager for remote access (no SSH needed)
  document_format           = "JSON"
  document_name             = "SSM-SessionManagerRunShell"
  s3_bucket_name            = aws_s3_bucket.ssm_logs.id
  s3_key_prefix             = "session-manager-logs"
  kms_key_id                = aws_kms_key.ssm.id
  cloudwatch_log_group_name = aws_cloudwatch_log_group.ssm.name
}

# AU - Audit and Accountability

# AU-2: Event Logging
resource "aws_cloudtrail" "nist_au_2" {
  name                          = "nist-audit-trail"
  s3_bucket_name               = aws_s3_bucket.cloudtrail.id
  is_multi_region_trail         = true
  include_global_service_events = true
  enable_log_file_validation    = true
  kms_key_id                   = aws_kms_key.cloudtrail.arn
  
  # ✅ Log all events
  event_selector {
    read_write_type           = "All"
    include_management_events = true
    
    data_resource {
      type   = "AWS::S3::Object"
      values = ["arn:aws:s3:::"]
    }
  }
}

# AU-3: Content of Audit Records
# (CloudTrail provides required fields: who, what, when, where, outcome)

# AU-9: Protection of Audit Information
resource "aws_s3_bucket_policy" "nist_au_9" {
  bucket = aws_s3_bucket.cloudtrail.id
  
  # Restrict who can delete audit logs
  policy = jsonencode({
    Statement = [
      {
        Sid    = "DenyDeleteExceptSecurityAdmin"
        Effect = "Deny"
        Principal = { AWS = "*" }
        Action = [
          "s3:DeleteObject",
          "s3:DeleteObjectVersion",
          "s3:PutLifecycleConfiguration"
        ]
        Resource = "${aws_s3_bucket.cloudtrail.arn}/*"
        Condition = {
          ArnNotEquals = {
            "aws:PrincipalARN" = aws_iam_role.security_admin.arn
          }
        }
      }
    ]
  })
}

# CM - Configuration Management

# CM-2: Baseline Configuration
# Terraform IS the baseline configuration management

# CM-6: Configuration Settings
resource "aws_config_configuration_recorder" "nist_cm_6" {
  name     = "default"
  role_arn = aws_iam_role.config.arn
  
  recording_group {
    all_supported                 = true
    include_global_resource_types = true
  }
}

# CM-7: Least Functionality
# Implemented via security groups and VPC configuration

# IA - Identification and Authentication

# IA-2: Identification and Authentication (Users)
resource "aws_iam_policy" "nist_ia_2" {
  name = "NIST-IA-2-RequireMFA"
  
  policy = jsonencode({
    Statement = [
      {
        Sid    = "DenyWithoutMFA"
        Effect = "Deny"
        NotAction = [
          "iam:EnableMFADevice",
          "iam:CreateVirtualMFADevice",
          "iam:ListMFADevices",
          "sts:GetSessionToken"
        ]
        Resource = "*"
        Condition = {
          BoolIfExists = {
            "aws:MultiFactorAuthPresent" = "false"
          }
        }
      }
    ]
  })
}

# IA-5: Authenticator Management
resource "aws_iam_account_password_policy" "nist_ia_5" {
  minimum_password_length        = 14
  require_uppercase_characters   = true
  require_lowercase_characters   = true
  require_numbers                = true
  require_symbols                = true
  max_password_age               = 90
  password_reuse_prevention      = 24
  allow_users_to_change_password = true
}

# SC - System and Communications Protection

# SC-5: Denial of Service Protection
resource "aws_shield_protection" "nist_sc_5" {
  count = var.enable_shield_advanced ? 1 : 0
  
  name         = "main-alb-protection"
  resource_arn = aws_lb.main.arn
}

# SC-8: Transmission Confidentiality and Integrity
# Implemented via TLS enforcement in all load balancers, APIs, databases

# SC-28: Protection of Information at Rest
# Implemented via KMS encryption for all services

# SI - System and Information Integrity

# SI-2: Flaw Remediation
resource "aws_inspector2_enabler" "nist_si_2" {
  account_ids    = [var.account_id]
  resource_types = ["ECR", "EC2", "LAMBDA"]  # ✅ Vulnerability scanning
}

# SI-4: Information System Monitoring
resource "aws_guardduty_detector" "nist_si_4" {
  enable = true
  
  datasources {
    s3_logs { enable = true }
    kubernetes { audit_logs { enable = true } }
    malware_protection {
      scan_ec2_instance_with_findings {
        ebs_volumes { enable = true }
      }
    }
  }
}
```

---

## ขั้นตอนที่ 884: SOC 2 Type II Controls

### Trust Services Criteria (TSC)

```hcl
# CC6.1: Logical and Physical Access Controls

# CC6.1 - Point of access restricted
resource "aws_security_group" "cc6_1" {
  name        = "web-tier-sg"
  description = "SOC2 CC6.1: Restrict access points"
  vpc_id      = aws_vpc.main.id
  
  ingress {
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
    description = "HTTPS from internet"
  }
  
  # ✅ No other ingress = restricted access
}

# CC6.2 - Prior to issuing credentials
resource "aws_cognito_user_pool" "cc6_2" {
  name = "soc2-user-pool"
  
  # ✅ Email verification required
  auto_verified_attributes = ["email"]
  
  # ✅ MFA required
  mfa_configuration = "ON"
  software_token_mfa_configuration { enabled = true }
  
  # ✅ Strong password
  password_policy {
    minimum_length    = 14
    require_lowercase = true
    require_numbers   = true
    require_symbols   = true
    require_uppercase = true
  }
  
  # ✅ Advanced security (adaptive auth)
  user_pool_add_ons {
    advanced_security_mode = "ENFORCED"
  }
}

# CC6.3 - Access roles reviewed
resource "aws_iam_policy" "cc6_3_access_review" {
  name = "AccessReviewPolicy"
  
  # Policy for periodic access reviews
  policy = jsonencode({
    Statement = [{
      Effect = "Allow"
      Action = [
        "iam:ListUsers",
        "iam:ListRoles",
        "iam:ListGroups",
        "iam:GetUser",
        "iam:GetRole",
        "iam:ListAccessKeys",
        "iam:GetAccessKeyLastUsed",
        "iam:ListAttachedUserPolicies",
        "iam:ListAttachedRolePolicies"
      ]
      Resource = "*"
    }]
  })
}

# CC6.6 - Security controls for system boundaries
resource "aws_wafv2_web_acl" "cc6_6" {
  name  = "soc2-waf"
  scope = "REGIONAL"
  
  default_action { allow {} }
  
  rule {
    name     = "AWSManagedRulesCommonRuleSet"
    priority = 1
    override_action { none {} }
    statement {
      managed_rule_group_statement {
        name        = "AWSManagedRulesCommonRuleSet"
        vendor_name = "AWS"
      }
    }
    visibility_config {
      cloudwatch_metrics_enabled = true
      metric_name                = "CC6-6-WAF"
      sampled_requests_enabled   = true
    }
  }
  
  visibility_config {
    cloudwatch_metrics_enabled = true
    metric_name                = "SOC2WAF"
    sampled_requests_enabled   = true
  }
}

# CC6.7 - Transmission of data
resource "aws_lb_listener" "cc6_7_https" {
  load_balancer_arn = aws_lb.main.arn
  port              = "443"
  protocol          = "HTTPS"
  ssl_policy        = "ELBSecurityPolicy-TLS13-1-2-2021-06"
  certificate_arn   = aws_acm_certificate.main.arn
  
  default_action {
    type             = "forward"
    target_group_arn = aws_lb_target_group.app.arn
  }
}

# CC6.8 - Prevent or detect unauthorized software
resource "aws_inspector2_enabler" "cc6_8" {
  account_ids    = [var.account_id]
  resource_types = ["ECR", "EC2", "LAMBDA"]
}

# CC7.1 - Detect and monitor components
resource "aws_cloudwatch_dashboard" "cc7_1" {
  dashboard_name = "SOC2-Monitoring"
  
  dashboard_body = jsonencode({
    widgets = [
      {
        type = "metric"
        properties = {
          title   = "API Error Rate"
          metrics = [["AWS/ApplicationELB", "HTTPCode_Target_5XX_Count"]]
        }
      },
      {
        type = "metric"
        properties = {
          title   = "GuardDuty Findings"
          metrics = [["AWS/GuardDuty", "FindingCount"]]
        }
      }
    ]
  })
}

# CC8.1 - Change management
# Terraform + Git = Change management by design
# Every change:
# 1. Written in code
# 2. Peer reviewed (Pull Request)
# 3. Tested in staging
# 4. Approved before production
# 5. Auditable git history
```

---

## ขั้นตอนที่ 885: PCI-DSS Implementation

### PCI-DSS Requirements สำหรับ Cloud Infrastructure

```hcl
# Requirement 1: Network security controls

# PCI 1.3: Prohibit direct public access to CHD environment
resource "aws_security_group" "pci_1_3" {
  name   = "cardholder-data-sg"
  vpc_id = aws_vpc.main.id
  
  # ✅ No direct internet access
  # Only from application tier
  ingress {
    from_port                = 443
    to_port                  = 443
    protocol                 = "tcp"
    source_security_group_id = aws_security_group.app.id
    description              = "HTTPS from app tier only"
  }
  
  # ✅ Very restricted egress
  egress {
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = [var.vpc_cidr]
    description = "Internal HTTPS only"
  }
}

# Requirement 3: Protect stored account data

# PCI 3.4: Mask PAN when displayed
# (Application-level control)

# PCI 3.5: Protect encryption keys
resource "aws_kms_key" "pci_3_5" {
  description             = "PCI-DSS 3.5: Cardholder data encryption key"
  deletion_window_in_days = 30
  enable_key_rotation     = true  # ✅ Annual rotation
  
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
        Sid    = "Allow CHD application"
        Effect = "Allow"
        Principal = { AWS = aws_iam_role.chd_app.arn }
        Action = ["kms:Decrypt", "kms:GenerateDataKey"]
        Resource = "*"
        Condition = {
          StringEquals = {
            "kms:EncryptionContext:purpose" = "cardholder-data"
          }
        }
      }
    ]
  })
}

# Requirement 6: Develop and maintain secure systems

# PCI 6.5: Protect web-facing applications
resource "aws_wafv2_web_acl" "pci_6_5" {
  name  = "pci-waf"
  scope = "REGIONAL"
  
  default_action { allow {} }
  
  # ✅ SQL Injection protection (PCI 6.5.1)
  rule {
    name     = "SQLiProtection"
    priority = 1
    action { block {} }
    statement {
      sqli_match_statement {
        field_to_match { all_query_arguments {} }
        text_transformation {
          priority = 1
          type     = "URL_DECODE"
        }
      }
    }
    visibility_config {
      cloudwatch_metrics_enabled = true
      metric_name                = "SQLi"
      sampled_requests_enabled   = true
    }
  }
  
  # ✅ XSS protection (PCI 6.5.7)
  rule {
    name     = "XSSProtection"
    priority = 2
    action { block {} }
    statement {
      xss_match_statement {
        field_to_match { all_query_arguments {} }
        text_transformation {
          priority = 1
          type     = "HTML_ENTITY_DECODE"
        }
      }
    }
    visibility_config {
      cloudwatch_metrics_enabled = true
      metric_name                = "XSS"
      sampled_requests_enabled   = true
    }
  }
  
  visibility_config {
    cloudwatch_metrics_enabled = true
    metric_name                = "PCIWAF"
    sampled_requests_enabled   = true
  }
}

# Requirement 8: Identify users and authenticate access to system components

# PCI 8.3: Secure authentication for all users
resource "aws_iam_account_password_policy" "pci_8_3" {
  minimum_password_length        = 14
  require_uppercase_characters   = true
  require_lowercase_characters   = true
  require_numbers                = true
  require_symbols                = true
  max_password_age               = 90   # PCI: Quarterly change
  password_reuse_prevention      = 12   # PCI: 12 previous passwords
  allow_users_to_change_password = true
}

# PCI 8.6: MFA for remote access
# (Implemented via IAM MFA policies)

# Requirement 10: Log and monitor all access

# PCI 10.1: Implement audit trails
resource "aws_cloudtrail" "pci_10_1" {
  name                          = "pci-audit-trail"
  s3_bucket_name               = aws_s3_bucket.pci_logs.id
  is_multi_region_trail         = true
  include_global_service_events = true
  enable_log_file_validation    = true
  kms_key_id                   = aws_kms_key.pci_3_5.arn
  
  event_selector {
    read_write_type           = "All"
    include_management_events = true
    
    data_resource {
      type   = "AWS::S3::Object"
      values = ["arn:aws:s3:::${aws_s3_bucket.chd_storage.id}/"]  # CHD bucket only
    }
  }
}

# PCI 10.5: Secure audit logs from modification
resource "aws_s3_bucket_object_lock_configuration" "pci_10_5" {
  bucket = aws_s3_bucket.pci_logs.id
  
  rule {
    default_retention {
      mode = "COMPLIANCE"  # Cannot be overridden
      days = 365           # 1 year retention (PCI requires minimum 12 months)
    }
  }
}

# PCI 10.6: Review logs at least daily
resource "aws_cloudwatch_metric_alarm" "pci_10_6" {
  alarm_name          = "PCI-10.6-SecurityEvents"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = 1
  metric_name         = "UnauthorizedAPICalls"
  namespace           = "CISBenchmark"
  period              = 300
  statistic           = "Sum"
  threshold           = 0
  alarm_actions       = [aws_sns_topic.pci_alerts.arn]
}

# Requirement 11: Test security of systems and networks regularly

# PCI 11.2: Vulnerability scans
resource "aws_inspector2_enabler" "pci_11_2" {
  account_ids    = [var.account_id]
  resource_types = ["EC2", "ECR", "LAMBDA"]
}

# PCI 11.3: Penetration testing
# (Manual process - cannot configure via Terraform)
# But document the scope in Terraform tags:
resource "aws_s3_bucket" "pci_pentest_scope" {
  tags = {
    PCIScope  = "true"
    PenTestRequired = "quarterly"
  }
}
```

---

## ขั้นตอนที่ 886: Resource Tagging for Compliance

### Mandatory Resource Tagging

```hcl
# variables.tf
variable "mandatory_tags" {
  description = "Mandatory tags for all resources"
  type        = map(string)
  
  validation {
    condition = alltrue([
      contains(keys(var.mandatory_tags), "Environment"),
      contains(keys(var.mandatory_tags), "Owner"),
      contains(keys(var.mandatory_tags), "CostCenter"),
      contains(keys(var.mandatory_tags), "DataClassification")
    ])
    error_message = "Mandatory tags: Environment, Owner, CostCenter, DataClassification"
  }
}

# ✅ Default tags for all resources in provider
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

provider "aws" {
  region = var.region
  
  # ✅ Default tags applied to ALL resources
  default_tags {
    tags = {
      Environment        = var.environment
      ManagedBy          = "terraform"
      Repository         = var.git_repository
      LastApplied        = timestamp()
      DataClassification = var.data_classification
      Owner              = var.team_name
      CostCenter         = var.cost_center
      ComplianceScope    = var.compliance_scope  # e.g., "PCI-DSS", "HIPAA", "SOX"
    }
  }
}

# ✅ Compliance tagging policy (Organization SCP)
resource "aws_organizations_policy" "mandatory_tags" {
  name        = "MandatoryTags"
  description = "Require mandatory tags on all resources"
  type        = "TAG_POLICY"
  
  content = jsonencode({
    tags = {
      Environment = {
        tag_key = { "@@assign" = "Environment" }
        tag_value = {
          "@@assign" = ["production", "staging", "development", "test"]
        }
        enforced_for = {
          "@@assign" = [
            "ec2:instance",
            "rds:db",
            "s3:bucket",
            "lambda:function"
          ]
        }
      }
      DataClassification = {
        tag_key = { "@@assign" = "DataClassification" }
        tag_value = {
          "@@assign" = ["public", "internal", "confidential", "restricted"]
        }
      }
    }
  })
}

# ✅ AWS Config rule for tag compliance
resource "aws_config_config_rule" "required_tags" {
  name        = "required-tags"
  description = "Ensure resources have required tags"
  
  source {
    owner             = "AWS"
    source_identifier = "REQUIRED_TAGS"
  }
  
  input_parameters = jsonencode({
    tag1Key   = "Environment"
    tag2Key   = "Owner"
    tag3Key   = "CostCenter"
    tag4Key   = "DataClassification"
  })
}

# ✅ Tag-based access control
resource "aws_iam_policy" "prod_access_with_tag" {
  name = "ProdAccessWithTag"
  
  policy = jsonencode({
    Statement = [
      {
        Effect = "Allow"
        Action = ["ec2:StartInstances", "ec2:StopInstances"]
        Resource = "*"
        Condition = {
          StringEquals = {
            "aws:ResourceTag/Environment" = var.environment  # ✅ Tag-based
            "aws:ResourceTag/Team"        = var.team_name
          }
        }
      }
    ]
  })
}
```

---

## ขั้นตอนที่ 887: Compliance Automation with Lambda

### Automated Compliance Remediation

```hcl
# Lambda สำหรับ auto-remediation

# ✅ Function: Auto-enable S3 encryption
resource "aws_lambda_function" "remediate_s3_encryption" {
  filename      = "remediate-s3-encryption.zip"
  function_name = "remediate-s3-encryption"
  role          = aws_iam_role.remediator.arn
  handler       = "index.handler"
  runtime       = "python3.12"
  timeout       = 60
  
  environment {
    variables = {
      KMS_KEY_ARN = aws_kms_key.s3_remediation.arn
    }
  }
}

# AWS Config Auto-remediation
resource "aws_config_remediation_configuration" "s3_encryption" {
  config_rule_name = aws_config_config_rule.s3_encryption.name
  
  resource_type    = "AWS::S3::Bucket"
  target_type      = "SSM_DOCUMENT"
  target_id        = "AWS-ConfigureS3BucketLogging"
  
  parameter {
    name         = "AutomationAssumeRole"
    static_value = aws_iam_role.remediator.arn
  }
  
  automatic                    = true
  maximum_automatic_attempts   = 3
  retry_attempt_seconds        = 60
  
  execution_controls {
    ssm_controls {
      concurrent_execution_rate_percentage = 25
      error_percentage                     = 20
    }
  }
}

# ✅ Compliance Reporting Lambda
resource "aws_lambda_function" "compliance_report" {
  filename      = "compliance-report.zip"
  function_name = "generate-compliance-report"
  role          = aws_iam_role.compliance_reporter.arn
  handler       = "index.handler"
  runtime       = "python3.12"
  timeout       = 300
  memory_size   = 512
  
  environment {
    variables = {
      REPORT_BUCKET = aws_s3_bucket.compliance_reports.id
      SNS_TOPIC_ARN = aws_sns_topic.compliance_alerts.arn
    }
  }
}

# ✅ Weekly compliance report
resource "aws_cloudwatch_event_rule" "weekly_compliance" {
  name                = "WeeklyComplianceReport"
  description         = "Generate weekly compliance report"
  schedule_expression = "cron(0 8 ? * MON *)"  # Every Monday 8am
}

resource "aws_cloudwatch_event_target" "compliance_report" {
  rule = aws_cloudwatch_event_rule.weekly_compliance.name
  arn  = aws_lambda_function.compliance_report.arn
}
```

---

## ขั้นตอนที่ 888: Compliance Dashboard

### Comprehensive Compliance Status

```hcl
# ✅ Security Hub Compliance Dashboard
resource "aws_cloudwatch_dashboard" "compliance" {
  dashboard_name = "ComplianceDashboard"
  
  dashboard_body = jsonencode({
    widgets = [
      # Security Hub Score
      {
        type   = "custom"
        x      = 0
        y      = 0
        width  = 8
        height = 6
        properties = {
          endpoint = "https://...lambda-url.../security-score"
          title    = "Security Hub Score"
        }
      },
      # CIS Benchmark Compliance
      {
        type   = "metric"
        x      = 8
        y      = 0
        width  = 8
        height = 6
        properties = {
          metrics = [
            ["AWS/Config", "CompliancePercentage", "ConfigRuleName", "cis-1-16-iam-no-direct-user-policy"],
            ["AWS/Config", "CompliancePercentage", "ConfigRuleName", "cis-5-1-restricted-ssh"],
            ["AWS/Config", "CompliancePercentage", "ConfigRuleName", "cis-5-2-restricted-rdp"]
          ]
          title  = "CIS Compliance Rate"
          period = 3600
          stat   = "Average"
        }
      },
      # GuardDuty Findings Trend
      {
        type   = "metric"
        x      = 16
        y      = 0
        width  = 8
        height = 6
        properties = {
          metrics = [
            ["AWS/GuardDuty", "FindingCount", "Severity", "HIGH"],
            ["AWS/GuardDuty", "FindingCount", "Severity", "CRITICAL"]
          ]
          title  = "High/Critical GuardDuty Findings"
          period = 86400
          stat   = "Sum"
        }
      }
    ]
  })
}

# ✅ Compliance report S3 bucket
resource "aws_s3_bucket" "compliance_reports" {
  bucket = "company-compliance-reports-${var.account_id}"
}

resource "aws_s3_bucket_lifecycle_configuration" "compliance_reports" {
  bucket = aws_s3_bucket.compliance_reports.id
  
  rule {
    id     = "compliance-archive"
    status = "Enabled"
    
    filter {}
    
    transition {
      days          = 90
      storage_class = "STANDARD_IA"
    }
    
    transition {
      days          = 365
      storage_class = "GLACIER"
    }
    
    # Keep 7 years for compliance (SOX requirement)
    expiration {
      days = 2555  # 7 years
    }
  }
}
```

---

## ขั้นตอนที่ 889: HashiCorp Sentinel Policies

### Policy as Code with Sentinel

```python
# sentinel/policies/require-encryption.sentinel
# Ensure all S3 buckets have encryption enabled

import "tfplan/v2" as tfplan

# Get all S3 bucket encryption configurations
encryption_configs = filter tfplan.resource_changes as _, rc {
  rc.type is "aws_s3_bucket_server_side_encryption_configuration" and
  rc.change.actions contains "create" or
  rc.change.actions contains "update"
}

# Rule: All S3 buckets must have encryption
main = rule {
  length(filter encryption_configs as _, ec {
    ec.change.after.rule[0].apply_server_side_encryption_by_default[0].sse_algorithm in
      ["AES256", "aws:kms"]
  }) is length(encryption_configs)
}
```

```python
# sentinel/policies/require-mfa.sentinel
# Ensure MFA is enabled for all IAM users

import "tfplan/v2" as tfplan

# Check password policy
password_policies = filter tfplan.resource_changes as _, rc {
  rc.type is "aws_iam_account_password_policy"
}

main = rule when length(password_policies) > 0 {
  all password_policies as _, pp {
    pp.change.after.minimum_password_length >= 14 and
    pp.change.after.require_uppercase_characters is true and
    pp.change.after.require_lowercase_characters is true and
    pp.change.after.require_numbers is true and
    pp.change.after.require_symbols is true
  }
}
```

```hcl
# ✅ Sentinel policy set
# .terraform.d/policies/main.tf (Terraform Cloud)

# การ configure ผ่าน Terraform Cloud workspace
# Settings → Policy Sets → Create Policy Set
```

---

## ขั้นตอนที่ 890: Compliance Testing

### Terratest สำหรับ Compliance

```go
// compliance_test.go
package test

import (
    "testing"
    "github.com/gruntwork-io/terratest/modules/aws"
    "github.com/gruntwork-io/terratest/modules/terraform"
    "github.com/stretchr/testify/assert"
)

func TestS3BucketCompliance(t *testing.T) {
    t.Parallel()
    
    region := "us-east-1"
    bucketID := "my-test-bucket"
    
    // ✅ Test: S3 encryption enabled
    encryption, err := aws.GetS3BucketEncryptionE(t, region, bucketID)
    assert.Nil(t, err)
    assert.NotNil(t, encryption, "S3 bucket should have encryption")
    
    // ✅ Test: Public access blocked
    publicAccess, err := aws.GetS3BucketPublicAccessBlockE(t, region, bucketID)
    assert.Nil(t, err)
    assert.True(t, *publicAccess.BlockPublicAcls, "Block public ACLs should be enabled")
    assert.True(t, *publicAccess.BlockPublicPolicy, "Block public policy should be enabled")
    assert.True(t, *publicAccess.IgnorePublicAcls, "Ignore public ACLs should be enabled")
    assert.True(t, *publicAccess.RestrictPublicBuckets, "Restrict public buckets should be enabled")
    
    // ✅ Test: Versioning enabled
    versioningStatus := aws.GetS3BucketVersioning(t, region, bucketID)
    assert.Equal(t, "Enabled", versioningStatus, "Versioning should be enabled")
}

func TestIAMCompliance(t *testing.T) {
    t.Parallel()
    
    // ✅ Test: No direct user policies
    awsAccountID := aws.GetAccountId(t)
    passwordPolicy := aws.GetIamPasswordPolicy(t)
    
    assert.GreaterOrEqual(t, *passwordPolicy.MinimumPasswordLength, int64(14))
    assert.True(t, *passwordPolicy.RequireUppercaseCharacters)
    assert.True(t, *passwordPolicy.RequireLowercaseCharacters)
    assert.True(t, *passwordPolicy.RequireNumbers)
    assert.True(t, *passwordPolicy.RequireSymbols)
    assert.Equal(t, int64(24), *passwordPolicy.PasswordReusePrevention)
}
```

---

## สรุป Compliance Frameworks

### Framework Coverage Matrix

| Control | CIS | NIST 800-53 | SOC 2 | PCI-DSS | Implementation |
|---------|-----|-------------|-------|---------|----------------|
| IAM MFA | 1.5-1.6 | IA-2 | CC6.1 | 8.3.6 | IAM Policy |
| Encryption at rest | 2.1-2.3 | SC-28 | CC6.7 | 3.4 | KMS + service config |
| Encryption in transit | - | SC-8 | CC6.7 | 4.1 | TLS + HTTPS |
| CloudTrail | 3.1-3.7 | AU-2 | CC7.2 | 10.1 | aws_cloudtrail |
| GuardDuty | - | SI-4 | CC7.1 | 11.4 | aws_guardduty_detector |
| Security Groups | 5.1-5.2 | SC-7 | CC6.6 | 1.3 | aws_security_group |
| VPC Flow Logs | 5.5 | AU-2 | CC7.2 | 10.2 | aws_flow_log |
| AWS Config | 3.5 | CM-6 | CC7.1 | - | aws_config_* |
| Access Analyzer | 1.20 | AC-6 | CC6.3 | - | aws_accessanalyzer |
| WAF | - | SC-7 | CC6.6 | 6.5 | aws_wafv2_web_acl |

---

*Part 089 ครอบคลุม Compliance Frameworks ทั้งหมด - ต่อไปใน Part 090 จะเจาะลึก Checkov Security Scanner*
