# Part 083: IAM Security Issues & Privilege Escalation
## ขั้นตอนที่ 821-830: ปัญหาความปลอดภัย IAM และการยกระดับสิทธิ์

---

## ขั้นตอนที่ 821: ภาพรวม IAM Security

### ทำไม IAM จึงเป็น Priority #1

IAM (Identity and Access Management) เป็นชั้นการป้องกันที่สำคัญที่สุดใน AWS:

**สถิติ IAM Breaches:**
- 80% ของ cloud breaches เกี่ยวข้องกับ IAM misconfiguration
- ค่าใช้จ่ายเฉลี่ยของ privilege escalation attack: $4.24M (IBM, 2023)
- Capital One, Uber, LastPass ล้วนเกี่ยวข้องกับ IAM issues

### หลัก Least Privilege

```
Principle of Least Privilege (PoLP):
- Users/Services ควรมีสิทธิ์เท่าที่จำเป็นเท่านั้น
- ไม่มีสิทธิ์ที่ "spare" หรือ "just in case"
- Review และ revoke สิทธิ์ที่ไม่ใช้อีกต่อไป

ตัวอย่าง:
❌ Lambda function ที่ read S3 ไม่ควรมี: ec2:*, iam:*, etc.
✅ Lambda function ที่ read S3 ควรมี: s3:GetObject สำหรับ bucket เฉพาะเท่านั้น
```

### IAM Security Checklist

```
IAM Security Checklist:
□ No wildcard actions (Action: "*")
□ No wildcard resources (Resource: "*") 
□ MFA required for all users
□ No root account access keys
□ No hardcoded access keys
□ Unused access keys rotated/deleted
□ Access keys rotated every 90 days
□ Strong password policy
□ Permissions boundaries on roles
□ Trust policies scoped properly
□ External ID for cross-account
□ No privilege escalation paths
□ IAM Access Analyzer enabled
□ No service accounts with admin rights
□ Regular access reviews
```

---

## ขั้นตอนที่ 822: Misconfiguration #1 - Wildcard Actions

### ข้อมูล (Info)
- **CIS Control:** 1.16 (Ensure IAM policies are attached only to groups or roles)
- **CVSS Score:** 9.9 (Critical)
- **Checkov Rule:** CKV_AWS_40, CKV_AWS_274

### ❌ Vulnerable - Action: "*"

```hcl
# ❌ VULNERABLE - AdministratorAccess equivalent
resource "aws_iam_policy" "admin_all" {
  name = "AdminAllPolicy"
  
  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect   = "Allow"
      Action   = "*"      # ❌ ทุก AWS action
      Resource = "*"      # ❌ ทุก resource
    }]
  })
}

# ❌ VULNERABLE - Partial wildcard ก็อันตราย
resource "aws_iam_policy" "iam_all" {
  name = "IAMWildcardPolicy"
  
  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect   = "Allow"
      Action   = "iam:*"  # ❌ ทุก IAM action = full IAM control
      Resource = "*"
    }]
  })
}

# ❌ VULNERABLE - Managed policy with wildcard
resource "aws_iam_role_policy_attachment" "lambda_admin" {
  role       = aws_iam_role.lambda.name
  policy_arn = "arn:aws:iam::aws:policy/AdministratorAccess"  # ❌
}
```

### ✅ Secure - Specific Actions

```hcl
# ✅ SECURE - Only what the service needs
resource "aws_iam_policy" "lambda_s3_reader" {
  name        = "LambdaS3ReaderPolicy"
  description = "Allow Lambda to read from specific S3 bucket"
  
  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Sid    = "S3ReadAccess"
        Effect = "Allow"
        Action = [
          "s3:GetObject",           # ✅ Read objects
          "s3:GetObjectVersion",    # ✅ Read versions
          "s3:ListBucket",          # ✅ List bucket contents
          "s3:GetBucketLocation"    # ✅ Get bucket region
        ]
        Resource = [
          aws_s3_bucket.data.arn,           # ✅ Specific bucket
          "${aws_s3_bucket.data.arn}/*"     # ✅ Objects in bucket
        ]
      },
      {
        Sid    = "CloudWatchLogs"
        Effect = "Allow"
        Action = [
          "logs:CreateLogGroup",
          "logs:CreateLogStream",
          "logs:PutLogEvents"
        ]
        Resource = "arn:aws:logs:${var.region}:${var.account_id}:log-group:/aws/lambda/${var.function_name}:*"
      }
    ]
  })
}

# ✅ Use AWS managed read-only policies where appropriate
resource "aws_iam_role_policy_attachment" "lambda_s3_read_only" {
  role       = aws_iam_role.lambda.name
  policy_arn = "arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess"  # ✅ Scoped managed policy
}
```

---

## ขั้นตอนที่ 823: Misconfiguration #2 - Wildcard Resources

### ❌ Vulnerable - Resource: "*" with Dangerous Actions

```hcl
# ❌ VULNERABLE - IAM actions on all resources
resource "aws_iam_policy" "dangerous" {
  name = "DangerousPolicy"
  
  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Action = [
          "iam:CreateUser",
          "iam:DeleteUser",
          "iam:AttachUserPolicy",
          "iam:CreateAccessKey"
        ]
        Resource = "*"  # ❌ 모든 IAM users
      },
      {
        Effect = "Allow"
        Action = [
          "ec2:TerminateInstances",
          "ec2:StopInstances"
        ]
        Resource = "*"  # ❌ สามารถหยุดทุก EC2 instances
      },
      {
        Effect = "Allow"
        Action = [
          "s3:DeleteObject",
          "s3:DeleteBucket"
        ]
        Resource = "*"  # ❌ ลบทุก S3 bucket ได้
      }
    ]
  })
}
```

### ✅ Secure - Scoped Resources with Conditions

```hcl
# ✅ SECURE - Scoped resources
resource "aws_iam_policy" "scoped" {
  name = "ScopedPolicy"
  
  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Sid    = "IAMUsersInPath"
        Effect = "Allow"
        Action = [
          "iam:CreateUser",
          "iam:DeleteUser"
        ]
        Resource = "arn:aws:iam::${var.account_id}:user/app-service/*"  # ✅ Specific path
      },
      {
        Sid    = "EC2InTeamVPC"
        Effect = "Allow"
        Action = [
          "ec2:StopInstances",
          "ec2:StartInstances"
        ]
        Resource = "*"
        Condition = {
          StringEquals = {
            "aws:ResourceTag/Team"       = var.team_name     # ✅ Tag condition
            "aws:ResourceTag/ManagedBy"  = "terraform"
          }
        }
      },
      {
        Sid    = "S3BucketAccess"
        Effect = "Allow"
        Action = [
          "s3:GetObject",
          "s3:PutObject"
        ]
        Resource = "${aws_s3_bucket.team_bucket.arn}/*"  # ✅ Specific bucket
      }
    ]
  })
}
```

---

## ขั้นตอนที่ 824: Misconfiguration #3 - Privilege Escalation Paths

### ภาพรวม Privilege Escalation ใน AWS IAM

```
Privilege Escalation เกิดขึ้นเมื่อ:
1. User/Role มี permission ที่สามารถ grant ตัวเองสิทธิ์เพิ่มขึ้น
2. User สามารถสร้าง credentials ให้ privileged user/role อื่น
3. User สามารถ modify policy ที่ apply กับตัวเอง

17 เส้นทาง Privilege Escalation หลัก:
```

### Escalation Path 1: iam:CreatePolicyVersion

```hcl
# ❌ VULNERABLE - สามารถสร้าง policy version ใหม่ที่ admin
resource "aws_iam_policy" "escalation_1" {
  name = "CreatePolicyVersionEscalation"
  
  policy = jsonencode({
    Statement = [{
      Effect   = "Allow"
      Action   = [
        "iam:CreatePolicyVersion",  # ❌ สามารถสร้าง version ใหม่ที่มี "*" permission
        "iam:SetDefaultPolicyVersion"
      ]
      Resource = "*"
    }]
  })
}

# Attack scenario:
# 1. User has iam:CreatePolicyVersion
# 2. User creates new version of existing policy with Action: "*"
# 3. User sets it as default = full admin access!

# ✅ SECURE - Never grant iam:CreatePolicyVersion without constraints
# ถ้าจำเป็นต้องมี - จำกัดด้วย condition
resource "aws_iam_policy" "controlled_policy_management" {
  policy = jsonencode({
    Statement = [{
      Effect = "Allow"
      Action = ["iam:CreatePolicyVersion"]
      Resource = [
        "arn:aws:iam::${var.account_id}:policy/app/*"  # ✅ เฉพาะ policies ของ app
      ]
      # ยังมีความเสี่ยง - ควรหลีกเลี่ยงถ้าเป็นไปได้
    }]
  })
}
```

### Escalation Path 2: iam:CreateAccessKey

```hcl
# ❌ VULNERABLE - สามารถสร้าง access key ให้ user อื่น
resource "aws_iam_policy" "create_access_key" {
  policy = jsonencode({
    Statement = [{
      Effect   = "Allow"
      Action   = ["iam:CreateAccessKey"]
      Resource = "*"  # ❌ สร้าง key ให้ admin user ได้
    }]
  })
}

# Attack:
# 1. User A (low-priv) has iam:CreateAccessKey
# 2. User A creates access key for User B (admin)
# 3. User A uses that key to act as admin!

# ✅ SECURE - Only for own user
resource "aws_iam_policy" "create_own_access_key" {
  policy = jsonencode({
    Statement = [{
      Effect   = "Allow"
      Action   = ["iam:CreateAccessKey"]
      Resource = "arn:aws:iam::${var.account_id}:user/${aws:username}"  # ✅ Only self
    }]
  })
}
```

### Escalation Path 3: iam:AttachUserPolicy / AttachRolePolicy

```hcl
# ❌ VULNERABLE - สามารถ attach AdminPolicy ให้ตัวเอง
resource "aws_iam_policy" "attach_policy" {
  policy = jsonencode({
    Statement = [{
      Effect   = "Allow"
      Action   = [
        "iam:AttachUserPolicy",    # ❌
        "iam:AttachRolePolicy",    # ❌
        "iam:AttachGroupPolicy"    # ❌
      ]
      Resource = "*"
    }]
  })
}

# Attack:
# 1. Attacker has iam:AttachUserPolicy
# 2. Attacker does: aws iam attach-user-policy --user-name attacker --policy-arn arn:aws:iam::aws:policy/AdministratorAccess
# 3. Now attacker has full admin!

# ✅ SECURE - Limit with Permissions Boundary
resource "aws_iam_policy" "controlled_attach" {
  policy = jsonencode({
    Statement = [
      {
        Effect = "Allow"
        Action = ["iam:AttachUserPolicy", "iam:AttachRolePolicy"]
        Resource = "*"
        Condition = {
          StringEquals = {
            "iam:PermissionsBoundary" = aws_iam_policy.permissions_boundary.arn
          }
          # ✅ ต้องระบุ boundary เสมอ = ไม่สามารถ grant permissions เกิน boundary
        }
      }
    ]
  })
}
```

### Escalation Path 4: iam:PassRole (อันตรายมาก)

```hcl
# ❌ VULNERABLE - PassRole ไม่มี condition
resource "aws_iam_policy" "pass_role_dangerous" {
  policy = jsonencode({
    Statement = [{
      Effect   = "Allow"
      Action   = ["iam:PassRole"]  # ❌
      Resource = "*"               # ❌ สามารถ pass admin role ให้ service ได้
    }]
  })
}

# Attack:
# 1. Attacker has iam:PassRole on "*"
# 2. Attacker creates Lambda with AdminRole
# 3. Attacker invokes Lambda to do admin actions
# 4. Attacker has effective admin access!

# ✅ SECURE - PassRole only to specific services and roles
resource "aws_iam_policy" "pass_role_secure" {
  policy = jsonencode({
    Statement = [{
      Sid    = "PassRoleToLambdaOnly"
      Effect = "Allow"
      Action = ["iam:PassRole"]
      Resource = [
        "arn:aws:iam::${var.account_id}:role/lambda-execution-*"  # ✅ Specific roles
      ]
      Condition = {
        StringEquals = {
          "iam:PassedToService" = "lambda.amazonaws.com"  # ✅ Lambda only
        }
        StringLike = {
          "iam:AssociatedResourceArn" = "arn:aws:lambda:*:${var.account_id}:function:app-*"
        }
      }
    }]
  })
}
```

### Escalation Path 5: sts:AssumeRole (Role Chaining)

```hcl
# ❌ VULNERABLE - Trust policy too broad
resource "aws_iam_role" "admin_role" {
  name = "AdminRole"
  
  assume_role_policy = jsonencode({
    Statement = [{
      Effect    = "Allow"
      Principal = {
        AWS = "*"  # ❌ ทุกคนสามารถ assume role นี้ได้!
      }
      Action = "sts:AssumeRole"
    }]
  })
}

# ❌ ALSO VULNERABLE - ทั้ง account assume ได้
resource "aws_iam_role" "another_admin" {
  assume_role_policy = jsonencode({
    Statement = [{
      Effect = "Allow"
      Principal = {
        AWS = "arn:aws:iam::${var.account_id}:root"  # ❌ ทุก entity ใน account
      }
      Action = "sts:AssumeRole"
    }]
  })
}

# ✅ SECURE - Specific principals with conditions
resource "aws_iam_role" "secure_role" {
  name = "SecureAppRole"
  
  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Principal = {
          AWS = [
            aws_iam_role.app_server_role.arn,    # ✅ Specific role
            "arn:aws:iam::${var.account_id}:role/DevOpsTeam"  # ✅ Specific team
          ]
        }
        Action = "sts:AssumeRole"
        Condition = {
          Bool = {
            "aws:MultiFactorAuthPresent" = "true"  # ✅ MFA required
          }
          StringEquals = {
            "sts:ExternalId" = var.external_id   # ✅ External ID
          }
          IpAddress = {
            "aws:SourceIp" = var.office_ip_ranges  # ✅ IP restriction
          }
        }
      }
    ]
  })
}
```

### Escalation Path 6: iam:UpdateAssumeRolePolicy

```hcl
# ❌ VULNERABLE - สามารถเปลี่ยน trust policy
resource "aws_iam_policy" "update_trust" {
  policy = jsonencode({
    Statement = [{
      Effect   = "Allow"
      Action   = ["iam:UpdateAssumeRolePolicy"]
      Resource = "*"  # ❌ แก้ trust policy ของทุก role ได้
    }]
  })
}

# Attack:
# 1. Update admin role's trust policy to trust attacker's user
# 2. Now attacker can assume admin role!

# ✅ SECURE - Cannot fix this completely with condition
# แต่สามารถจำกัดให้เฉพาะ roles ของตัวเอง
resource "aws_iam_policy" "update_own_trust" {
  policy = jsonencode({
    Statement = [{
      Effect   = "Allow"
      Action   = ["iam:UpdateAssumeRolePolicy"]
      Resource = "arn:aws:iam::${var.account_id}:role/app/*"  # ✅ Scoped
    }]
  })
}
```

---

## ขั้นตอนที่ 825: Privilege Escalation Detection

### ตรวจจับ Privilege Escalation Paths

```python
# script: check_escalation_paths.py
# Tool: Cloudsplaining, Policyuniverse

import boto3
import json

# Escalation actions ที่อันตราย
ESCALATION_ACTIONS = {
    "iam:CreatePolicyVersion": "Create new policy version",
    "iam:SetDefaultPolicyVersion": "Set default policy version", 
    "iam:CreateAccessKey": "Create access keys for other users",
    "iam:CreateLoginProfile": "Create console password for other users",
    "iam:UpdateLoginProfile": "Update console password",
    "iam:AttachUserPolicy": "Attach admin policy to self",
    "iam:AttachGroupPolicy": "Attach admin policy to group",
    "iam:AttachRolePolicy": "Attach admin policy to role",
    "iam:PutUserPolicy": "Add inline policy to user",
    "iam:PutGroupPolicy": "Add inline policy to group", 
    "iam:PutRolePolicy": "Add inline policy to role",
    "iam:AddUserToGroup": "Add self to privileged group",
    "iam:UpdateAssumeRolePolicy": "Modify trust policy",
    "iam:PassRole": "Pass privileged role to service",
    "sts:AssumeRole": "Assume privileged role",
    "iam:CreateRole": "Create role with admin permissions",
    "iam:DeleteRolePermissionsBoundary": "Remove permissions boundary"
}

def check_policy_for_escalation(policy_document):
    """Check if a policy contains privilege escalation paths"""
    issues = []
    
    for statement in policy_document.get("Statement", []):
        if statement.get("Effect") != "Allow":
            continue
        
        actions = statement.get("Action", [])
        if isinstance(actions, str):
            actions = [actions]
        
        resources = statement.get("Resource", [])
        if isinstance(resources, str):
            resources = [resources]
        
        for action in actions:
            # Check for wildcard
            if action == "*":
                issues.append({
                    "severity": "CRITICAL",
                    "issue": "Wildcard action grants all permissions including escalation paths",
                    "action": action
                })
                break
            
            # Check specific escalation actions
            if action in ESCALATION_ACTIONS:
                has_resource_wildcard = "*" in resources
                
                issues.append({
                    "severity": "HIGH" if has_resource_wildcard else "MEDIUM",
                    "issue": ESCALATION_ACTIONS[action],
                    "action": action,
                    "resource": resources,
                    "wildcard_resource": has_resource_wildcard
                })
    
    return issues
```

```hcl
# Terraform: IAM Access Analyzer
resource "aws_accessanalyzer_analyzer" "main" {
  analyzer_name = "main-analyzer"
  type          = "ACCOUNT"  # ✅ หา external access

  tags = {
    Name = "IAM-Access-Analyzer"
  }
}

# Archive rule สำหรับ known-safe findings
resource "aws_accessanalyzer_archive_rule" "known_safe" {
  analyzer_name = aws_accessanalyzer_analyzer.main.analyzer_name
  rule_name     = "known-safe-cross-account"
  
  filter {
    criteria = "principal.AWS"
    eq       = ["arn:aws:iam::${var.trusted_account}:root"]
  }
  
  filter {
    criteria = "resourceType"
    eq       = ["AWS::S3::Bucket"]
  }
}

# CloudWatch alarm สำหรับ new findings
resource "aws_cloudwatch_metric_alarm" "access_analyzer_findings" {
  alarm_name          = "IAMAccessAnalyzerNewFindings"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = 1
  metric_name         = "AccessAnalyzerFindingsCount"
  namespace           = "AWS/AccessAnalyzer"
  period              = 300
  statistic           = "Sum"
  threshold           = 0
  alarm_description   = "New IAM Access Analyzer findings detected"
  
  alarm_actions = [aws_sns_topic.security_alerts.arn]
}
```

---

## ขั้นตอนที่ 826: Missing MFA for Console Access

### ❌ Vulnerable - No MFA Enforcement

```hcl
# ❌ VULNERABLE - User สามารถ login โดยไม่มี MFA
resource "aws_iam_user" "developer" {
  name = "alice-developer"
}

resource "aws_iam_user_login_profile" "developer" {
  user                    = aws_iam_user.developer.name
  password_reset_required = false
  # ❌ ไม่มี MFA requirement
}

# ❌ ALSO VULNERABLE - Attaching PowerUser without MFA
resource "aws_iam_user_policy_attachment" "developer" {
  user       = aws_iam_user.developer.name
  policy_arn = "arn:aws:iam::aws:policy/PowerUserAccess"
  # ❌ PowerUser access ไม่มี MFA = risk
}
```

### ✅ Secure - MFA Required Policy

```hcl
# ✅ SECURE - MFA Enforcement Policy
resource "aws_iam_policy" "require_mfa" {
  name        = "EnforceMFA"
  description = "Deny all actions unless MFA is present, except MFA setup"
  
  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      # Allow users to manage their own MFA device
      {
        Sid    = "AllowManageMFADevice"
        Effect = "Allow"
        Action = [
          "iam:CreateVirtualMFADevice",
          "iam:EnableMFADevice",
          "iam:GetUser",
          "iam:ListMFADevices",
          "iam:ListVirtualMFADevices",
          "iam:ResyncMFADevice",
          "sts:GetSessionToken"
        ]
        Resource = "*"
      },
      # Allow viewing account info
      {
        Sid    = "AllowViewAccountInfo"
        Effect = "Allow"
        Action = [
          "iam:GetAccountPasswordPolicy",
          "iam:GetAccountSummary",
          "iam:ListVirtualMFADevices"
        ]
        Resource = "*"
      },
      # Deny everything else without MFA
      {
        Sid    = "DenyWithoutMFA"
        Effect = "Deny"
        NotAction = [
          "iam:CreateVirtualMFADevice",
          "iam:EnableMFADevice",
          "iam:GetUser",
          "iam:ListMFADevices",
          "iam:ListVirtualMFADevices",
          "iam:ResyncMFADevice",
          "sts:GetSessionToken"
        ]
        Resource = "*"
        Condition = {
          BoolIfExists = {
            "aws:MultiFactorAuthPresent" = "false"  # ✅ Deny if no MFA
          }
        }
      }
    ]
  })
}

# ✅ Attach MFA policy to all users via group
resource "aws_iam_group" "all_users" {
  name = "AllUsers"
}

resource "aws_iam_group_policy_attachment" "mfa_required" {
  group      = aws_iam_group.all_users.name
  policy_arn = aws_iam_policy.require_mfa.arn
}

# ✅ Add all users to the group
resource "aws_iam_user_group_membership" "developers" {
  for_each = toset(var.developer_usernames)
  
  user   = each.value
  groups = [aws_iam_group.all_users.name]
}
```

---

## ขั้นตอนที่ 827: Root Account Security

### ❌ Vulnerable - Root Account Usage

```hcl
# ❌ VULNERABLE - ใช้ root credentials ใน Terraform
provider "aws" {
  access_key = var.root_access_key     # ❌ ROOT CREDENTIALS!
  secret_key = var.root_secret_key
  region     = "us-east-1"
}

# ❌ VULNERABLE - Root account has access keys
# (ไม่สามารถ configure ผ่าน Terraform แต่ต้อง monitor)
```

### ✅ Secure - Monitoring Root Account Usage

```hcl
# ✅ CloudWatch alarm for root account usage
resource "aws_cloudwatch_metric_alarm" "root_account_usage" {
  alarm_name          = "RootAccountUsage"
  alarm_description   = "Triggers when root account is used"
  comparison_operator = "GreaterThanOrEqualToThreshold"
  evaluation_periods  = 1
  metric_name         = "RootAccountUsageCount"
  namespace           = "SecurityMetrics"
  period              = 300
  statistic           = "Sum"
  threshold           = 1
  treat_missing_data  = "notBreaching"
  
  alarm_actions = [aws_sns_topic.security_alerts.arn]
}

# ✅ CloudWatch metric filter for root usage
resource "aws_cloudwatch_log_metric_filter" "root_account_usage" {
  name           = "RootAccountUsage"
  pattern        = "{$.userIdentity.type = \"Root\" && $.userIdentity.invokedBy NOT EXISTS && $.eventType != \"AwsServiceEvent\"}"
  log_group_name = aws_cloudwatch_log_group.cloudtrail.name
  
  metric_transformation {
    name      = "RootAccountUsageCount"
    namespace = "SecurityMetrics"
    value     = "1"
  }
}

# ✅ IAM Password Policy (ป้องกัน root password reuse)
resource "aws_iam_account_password_policy" "strict" {
  minimum_password_length        = 14           # ✅ Minimum 14 chars
  require_lowercase_characters   = true          # ✅ Lowercase
  require_numbers                = true          # ✅ Numbers
  require_uppercase_characters   = true          # ✅ Uppercase
  require_symbols                = true          # ✅ Symbols
  allow_users_to_change_password = true          # ✅ Self-service
  max_password_age               = 90            # ✅ 90 day expiry
  password_reuse_prevention      = 24            # ✅ 24 previous passwords
  hard_expiry                    = false         # ✅ Allow expired password change
}
```

---

## ขั้นตอนที่ 828: Missing Permissions Boundaries

### คืออะไร Permissions Boundary

```
Permissions Boundary คือ:
- Policy ที่กำหนด maximum permissions ที่ entity สามารถมีได้
- ถึงแม้จะ attach policies อื่น effective permissions จะไม่เกิน boundary
- ใช้เพื่อ delegate IAM management อย่างปลอดภัย

Analogy:
- Permissions Boundary = กำแพงสูงสุด
- Attached Policies = พื้นที่ภายในกำแพง
- Effective Permissions = intersection ของทั้งสอง
```

### ❌ Vulnerable - No Permissions Boundary

```hcl
# ❌ VULNERABLE - Developer สามารถสร้าง role ที่มี admin permissions
resource "aws_iam_policy" "developer_iam" {
  name = "DeveloperIAMPolicy"
  
  policy = jsonencode({
    Statement = [{
      Effect = "Allow"
      Action = [
        "iam:CreateRole",
        "iam:AttachRolePolicy",
        "iam:CreatePolicy"
      ]
      Resource = "*"
      # ❌ ไม่มี boundary requirement = สามารถสร้าง admin role ได้
    }]
  })
}
```

### ✅ Secure - Permissions Boundary

```hcl
# ✅ SECURE - Permissions Boundary Definition
resource "aws_iam_policy" "developer_boundary" {
  name        = "DeveloperPermissionsBoundary"
  description = "Maximum permissions that developer-created roles can have"
  
  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      # Allow common service permissions
      {
        Sid    = "AllowCommonServices"
        Effect = "Allow"
        Action = [
          "s3:*",
          "dynamodb:*",
          "lambda:*",
          "logs:*",
          "ec2:Describe*",
          "cloudwatch:*"
        ]
        Resource = "*"
      },
      # Explicitly deny dangerous actions
      {
        Sid    = "DenyPrivilegeEscalation"
        Effect = "Deny"
        Action = [
          "iam:CreateUser",
          "iam:DeleteUser",
          "iam:AttachUserPolicy",
          "iam:PutUserPolicy",
          "organizations:*",
          "account:*"
        ]
        Resource = "*"
      },
      # Deny removing boundaries
      {
        Sid    = "DenyRemovingBoundary"
        Effect = "Deny"
        Action = [
          "iam:DeleteRolePermissionsBoundary",
          "iam:DeleteUserPermissionsBoundary"
        ]
        Resource = "*"
      }
    ]
  })
}

# ✅ Policy that requires boundary when creating roles
resource "aws_iam_policy" "developer_iam_with_boundary" {
  name = "DeveloperIAMWithBoundary"
  
  policy = jsonencode({
    Statement = [
      {
        Sid    = "CreateRolesWithBoundary"
        Effect = "Allow"
        Action = [
          "iam:CreateRole",
          "iam:PutRolePolicy",
          "iam:AttachRolePolicy"
        ]
        Resource = "*"
        Condition = {
          StringEquals = {
            "iam:PermissionsBoundary" = aws_iam_policy.developer_boundary.arn
            # ✅ MUST set boundary = cannot exceed boundary
          }
        }
      },
      {
        Sid    = "ManageBoundedRoles"
        Effect = "Allow"
        Action = [
          "iam:DeleteRole",
          "iam:DetachRolePolicy",
          "iam:DeleteRolePolicy"
        ]
        Resource = "*"
        Condition = {
          StringEquals = {
            "iam:PermissionsBoundary" = aws_iam_policy.developer_boundary.arn
          }
        }
      }
    ]
  })
}

# ✅ Example of creating a role with boundary
resource "aws_iam_role" "app_role_with_boundary" {
  name                 = "app-service-role"
  permissions_boundary = aws_iam_policy.developer_boundary.arn  # ✅ Set boundary
  
  assume_role_policy = jsonencode({
    Statement = [{
      Effect    = "Allow"
      Principal = { Service = "lambda.amazonaws.com" }
      Action    = "sts:AssumeRole"
    }]
  })
}
```

---

## ขั้นตอนที่ 829: IAM Password Policy & Access Key Management

### ❌ Vulnerable - Weak Password Policy

```hcl
# ❌ VULNERABLE - Weak or default password policy
resource "aws_iam_account_password_policy" "weak" {
  minimum_password_length        = 8    # ❌ Too short
  require_lowercase_characters   = false # ❌ No complexity
  require_numbers                = false # ❌ No complexity
  require_uppercase_characters   = false # ❌ No complexity
  require_symbols                = false # ❌ No complexity
  allow_users_to_change_password = true
  max_password_age               = 0    # ❌ Never expires
  password_reuse_prevention      = 0    # ❌ Can reuse
}
```

### ✅ Secure - Strong Password Policy

```hcl
# ✅ CIS Benchmark compliant password policy
resource "aws_iam_account_password_policy" "cis_compliant" {
  # CIS 1.8: Ensure IAM password policy requires minimum length of 14
  minimum_password_length = 14  # ✅

  # CIS 1.9-1.11: Complexity requirements
  require_lowercase_characters = true  # ✅
  require_numbers              = true  # ✅
  require_uppercase_characters = true  # ✅
  require_symbols              = true  # ✅

  # CIS 1.12: Allow users to change their own password
  allow_users_to_change_password = true  # ✅

  # CIS 1.13: No password expiration
  # (NIST 800-63B recommends no mandatory expiry unless compromised)
  max_password_age = 90  # ✅ Some orgs still require this

  # CIS 1.14: Prevent password reuse
  password_reuse_prevention = 24  # ✅ 24 previous passwords

  hard_expiry = false  # ✅ Allow login with expired password to change
}

# ✅ Monitor unused access keys
resource "aws_config_config_rule" "access_keys_rotated" {
  name        = "access-keys-rotated"
  description = "Checks whether IAM access keys are rotated within 90 days"
  
  source {
    owner             = "AWS"
    source_identifier = "ACCESS_KEYS_ROTATED"
  }
  
  input_parameters = jsonencode({
    maxAccessKeyAge = "90"
  })
}

# ✅ CloudWatch alarm for access key age
resource "aws_cloudwatch_metric_alarm" "old_access_keys" {
  alarm_name          = "IAMOldAccessKeys"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = 1
  metric_name         = "OldAccessKeyCount"
  namespace           = "SecurityMetrics"
  period              = 86400  # Daily check
  statistic           = "Maximum"
  threshold           = 0
  
  alarm_description = "IAM access keys older than 90 days detected"
  alarm_actions     = [aws_sns_topic.security_alerts.arn]
}
```

---

## ขั้นตอนที่ 830: Complete IAM Security Module

### IAM Security Baseline

```hcl
# modules/iam-security-baseline/main.tf

# ✅ GuardDuty for threat detection
resource "aws_guardduty_detector" "main" {
  enable = true
  
  datasources {
    s3_logs {
      enable = true
    }
    kubernetes {
      audit_logs {
        enable = true
      }
    }
    malware_protection {
      scan_ec2_instance_with_findings {
        ebs_volumes {
          enable = true
        }
      }
    }
  }
}

# ✅ IAM Access Analyzer
resource "aws_accessanalyzer_analyzer" "account" {
  analyzer_name = "account-analyzer"
  type          = "ACCOUNT"
}

# ✅ AWS Config for continuous compliance
resource "aws_config_configuration_recorder" "main" {
  name     = "default"
  role_arn = aws_iam_role.config.arn
  
  recording_group {
    all_supported                 = true
    include_global_resource_types = true
  }
}

# ✅ Config rules for IAM
resource "aws_config_config_rule" "iam_no_inline_policy" {
  name        = "iam-no-inline-policy"
  description = "Checks that users, groups, and roles do not have inline policies"
  
  source {
    owner             = "AWS"
    source_identifier = "IAM_NO_INLINE_POLICY_CHECK"
  }
  
  depends_on = [aws_config_configuration_recorder.main]
}

resource "aws_config_config_rule" "iam_policy_no_statements_with_admin" {
  name        = "iam-policy-no-admin"
  description = "Checks that no IAM policy allows admin access (*)"
  
  source {
    owner             = "AWS"
    source_identifier = "IAM_POLICY_NO_STATEMENTS_WITH_ADMIN_ACCESS"
  }
  
  depends_on = [aws_config_configuration_recorder.main]
}

resource "aws_config_config_rule" "mfa_enabled_for_iam_console" {
  name        = "mfa-enabled-for-iam-console"
  description = "Checks that MFA is enabled for all IAM users with console access"
  
  source {
    owner             = "AWS"
    source_identifier = "MFA_ENABLED_FOR_IAM_CONSOLE_ACCESS"
  }
  
  depends_on = [aws_config_configuration_recorder.main]
}

# ✅ Security Hub for aggregated findings
resource "aws_securityhub_account" "main" {}

resource "aws_securityhub_standards_subscription" "cis" {
  standards_arn = "arn:aws:securityhub:::ruleset/cis-aws-foundations-benchmark/v/1.4.0"
  depends_on    = [aws_securityhub_account.main]
}

# ✅ CloudWatch alarms for IAM events
locals {
  iam_alerts = {
    "CreateUser"            = "IAM user created"
    "DeleteUser"            = "IAM user deleted"
    "CreateAccessKey"       = "Access key created"
    "AttachUserPolicy"      = "Policy attached to user"
    "AttachRolePolicy"      = "Policy attached to role"
    "CreatePolicy"          = "IAM policy created"
    "DeletePolicy"          = "IAM policy deleted"
    "CreateRole"            = "IAM role created"
    "UpdateAssumeRolePolicy" = "Trust policy updated"
  }
}

resource "aws_cloudwatch_metric_filter" "iam_events" {
  for_each = local.iam_alerts
  
  name           = "IAMEvent-${each.key}"
  pattern        = "{$.eventSource = \"iam.amazonaws.com\" && $.eventName = \"${each.key}\"}"
  log_group_name = var.cloudtrail_log_group_name
  
  metric_transformation {
    name      = "IAMEvent${each.key}"
    namespace = "SecurityMetrics"
    value     = "1"
  }
}

resource "aws_cloudwatch_metric_alarm" "iam_events" {
  for_each = local.iam_alerts
  
  alarm_name          = "IAM${each.key}Detected"
  alarm_description   = each.value
  comparison_operator = "GreaterThanOrEqualToThreshold"
  evaluation_periods  = 1
  metric_name         = "IAMEvent${each.key}"
  namespace           = "SecurityMetrics"
  period              = 300
  statistic           = "Sum"
  threshold           = 1
  treat_missing_data  = "notBreaching"
  
  alarm_actions = [var.security_sns_topic_arn]
}

# ✅ CloudTrail for all IAM actions
resource "aws_cloudtrail" "iam_audit" {
  name                          = "iam-audit-trail"
  s3_bucket_name               = var.cloudtrail_bucket
  include_global_service_events = true
  is_multi_region_trail         = true
  enable_log_file_validation    = true
  
  cloud_watch_logs_group_arn = "${var.cloudtrail_log_group_arn}:*"
  cloud_watch_logs_role_arn  = var.cloudtrail_role_arn
  
  event_selector {
    read_write_type           = "All"
    include_management_events = true
  }
}
```

### IAM Privilege Escalation Prevention Policy

```hcl
# ✅ Organization SCP ป้องกัน privilege escalation
resource "aws_organizations_policy" "prevent_escalation" {
  name        = "PreventPrivilegeEscalation"
  description = "Prevent common privilege escalation paths"
  
  content = jsonencode({
    Version = "2012-10-17"
    Statement = [
      # Prevent removing permission boundaries
      {
        Sid    = "DenyRemovePermBoundary"
        Effect = "Deny"
        Action = [
          "iam:DeleteRolePermissionsBoundary",
          "iam:DeleteUserPermissionsBoundary"
        ]
        Resource = "*"
        Condition = {
          ArnNotLike = {
            "aws:PrincipalARN" = [
              "arn:aws:iam::*:role/BreakGlassRole",  # Emergency access only
              "arn:aws:iam::*:role/SecurityAdminRole"
            ]
          }
        }
      },
      # Require boundary when creating roles
      {
        Sid    = "RequireBoundaryOnRoleCreation"
        Effect = "Deny"
        Action = [
          "iam:CreateRole",
          "iam:PutRolePermissionsBoundary"
        ]
        Resource = "*"
        Condition = {
          StringNotEquals = {
            "iam:PermissionsBoundary" = var.standard_boundary_arn
          }
          ArnNotLike = {
            "aws:PrincipalARN" = "arn:aws:iam::*:role/SecurityAdminRole"
          }
        }
      },
      # Prevent assume role to restricted roles
      {
        Sid    = "DenyAssumeRestrictedRoles"
        Effect = "Deny"
        Action = "sts:AssumeRole"
        Resource = [
          "arn:aws:iam::*:role/OrganizationAccountAccessRole"
        ]
        Condition = {
          ArnNotLike = {
            "aws:PrincipalARN" = "arn:aws:iam::*:role/AllowedCrossAccountRole"
          }
        }
      }
    ]
  })
}
```

---

## สรุป IAM Security

### Privilege Escalation Prevention Matrix

| Action | Risk | Prevention |
|--------|------|-----------|
| iam:CreatePolicyVersion | Critical | Never grant; if needed, scope to specific policies |
| iam:SetDefaultPolicyVersion | Critical | Never grant independently |
| iam:CreateAccessKey | High | Scope to own user only |
| iam:AttachUserPolicy | High | Require permissions boundary |
| iam:AttachRolePolicy | High | Require permissions boundary |
| iam:PassRole | High | Scope to specific services and roles |
| iam:UpdateAssumeRolePolicy | High | Scope to specific roles |
| sts:AssumeRole | Medium | MFA + IP condition + External ID |

### IAM Security Best Practices Summary

```
1. ✅ Least Privilege - ให้สิทธิ์เท่าที่จำเป็นเท่านั้น
2. ✅ MFA Everywhere - บังคับ MFA ทุก user
3. ✅ No Root Access Keys - ลบ root access keys
4. ✅ Permissions Boundaries - delegate safely
5. ✅ Access Analyzer - detect external access
6. ✅ Regular Reviews - audit access quarterly
7. ✅ Rotate Keys - max 90 days
8. ✅ Monitor Changes - CloudTrail + CloudWatch alerts
9. ✅ SCP for Organization - prevent escalation at org level
10. ✅ No Wildcards - specific actions and resources
```

---

*Part 083 ครอบคลุม IAM Security ทั้งหมด - ต่อไปใน Part 084 จะเจาะลึก VPC & Network Misconfigurations*
