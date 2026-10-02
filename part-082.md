# Part 082: S3 Bucket Misconfigurations
## ขั้นตอนที่ 811-820: การกำหนดค่า S3 Bucket ที่ผิดพลาด

---

## ขั้นตอนที่ 811: ภาพรวม S3 Security

### S3 เป็นเป้าหมายหลักของ Data Breaches

Amazon S3 เป็นบริการที่มีการ misconfiguration บ่อยที่สุดใน AWS:

**สถิติ S3 Security (2019-2024):**
- 6% ของ S3 buckets ถูก expose สาธารณะ (UpGuard, 2021)
- 2,500+ ล้าน records ถูก expose จาก S3 misconfiguration ใน 2021
- Capital One, Twitch, GoDaddy ล้วนเป็นเหยื่อ S3 misconfiguration

**CIS AWS Foundations Benchmark - S3 Controls:**
- CIS 2.1.1: S3 Block Public Access
- CIS 2.1.2: S3 MFA Delete
- CIS 2.1.3: S3 Lifecycle policies
- CIS 2.1.4: S3 Access Logging
- CIS 2.1.5: S3 Server-side encryption

### S3 Security Checklist ภาพรวม

```
S3 Security Checklist:
□ Block Public Access (4 settings)
□ Server-Side Encryption (SSE-KMS)
□ Versioning enabled
□ MFA Delete enabled
□ Access Logging enabled
□ Force SSL (bucket policy)
□ Object Lock for compliance
□ Restrictive bucket policy
□ Cross-account access controlled
□ Replication encryption
□ Lifecycle policies
□ Event notifications for security
□ VPC endpoint for access
```

---

## ขั้นตอนที่ 812: Misconfiguration #1 - Public S3 Bucket

### ข้อมูล (Info)
- **CIS Control:** 2.1.1
- **CVSS Score:** 9.1 (Critical) - สามารถ expose ข้อมูลทั้ง bucket
- **Checkov Rule:** CKV_AWS_20, CKV2_AWS_6
- **TFSec Rule:** aws-s3-no-public-access-with-acl

### Real-World Breach
**Twitch Source Code Leak (October 2021)**
- 125GB ของ source code ถูก leak
- S3 bucket misconfiguration เป็นหนึ่งในปัจจัย
- ข้อมูล payout ของ streamers ถูก expose

### ❌ Vulnerable Configuration

```hcl
# ❌ VULNERABLE - Public read ACL
resource "aws_s3_bucket" "vulnerable" {
  bucket = "company-customer-data"
  
  # ❌ Deprecated but still works - public read access
  acl = "public-read"
  
  tags = {
    Environment = "production"
    DataClass   = "confidential"  # Ironically labeled confidential!
  }
}

# ❌ ALSO VULNERABLE - Public website without controls
resource "aws_s3_bucket_website_configuration" "vulnerable" {
  bucket = aws_s3_bucket.vulnerable.id
  
  index_document {
    suffix = "index.html"
  }
  
  error_document {
    key = "error.html"
  }
  # ❌ website config without access controls = public
}
```

### ✅ Secure Configuration

```hcl
# ✅ SECURE - Block all public access
resource "aws_s3_bucket" "secure" {
  bucket = "company-customer-data"
  
  # Note: acl argument removed - use bucket ownership controls instead
  
  tags = {
    Environment = "production"
    DataClass   = "confidential"
  }
}

# ✅ Block ALL public access
resource "aws_s3_bucket_public_access_block" "secure" {
  bucket = aws_s3_bucket.secure.id
  
  block_public_acls       = true  # ✅ Block public ACLs
  block_public_policy     = true  # ✅ Block public bucket policies
  ignore_public_acls      = true  # ✅ Ignore any public ACLs
  restrict_public_buckets = true  # ✅ Restrict public bucket access
}

# ✅ Bucket ownership - enforce bucket owner is sole owner
resource "aws_s3_bucket_ownership_controls" "secure" {
  bucket = aws_s3_bucket.secure.id
  
  rule {
    object_ownership = "BucketOwnerEnforced"  # ✅ Disable ACLs entirely
  }
}

# ✅ Block public access at account level too
resource "aws_s3_account_public_access_block" "account" {
  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}
```

---

## ขั้นตอนที่ 813: Misconfiguration #2 - Missing Block Public Access Settings

### ข้อมูล (Info)
- **CIS Control:** 2.1.1
- **CVSS Score:** 8.1 (High)
- **Checkov Rule:** CKV_AWS_53, CKV_AWS_54, CKV_AWS_55, CKV_AWS_56

### ความแตกต่างของแต่ละ Setting

```
block_public_acls:
  - ป้องกันการ upload objects ที่มี public ACL
  - ป้องกันการ set bucket ACL เป็น public
  - ผล: ใหม่ไม่ public แต่เก่ายังเป็น public

ignore_public_acls:
  - ละเว้น ACL ที่มีอยู่แล้ว
  - ผล: แม้ objects มี public ACL แต่ถูก ignore

block_public_policy:
  - ป้องกันการสร้าง bucket policy ที่ให้ public access
  - ผล: ไม่สามารถ set policy ที่ให้ anonymous access

restrict_public_buckets:
  - ป้องกัน cross-account access ที่ไม่ได้ระบุ
  - ผล: เฉพาะ authorized accounts เท่านั้น
```

### ❌ Vulnerable - Missing All 4 Settings

```hcl
# ❌ VULNERABLE - ไม่มี Block Public Access เลย
resource "aws_s3_bucket" "no_public_block" {
  bucket = "sensitive-data-bucket"
}

# ❌ ALSO VULNERABLE - มีแค่บางอัน
resource "aws_s3_bucket_public_access_block" "partial" {
  bucket = aws_s3_bucket.no_public_block.id
  
  block_public_acls       = true   # ✅
  block_public_policy     = false  # ❌ ลืม
  ignore_public_acls      = true   # ✅
  restrict_public_buckets = false  # ❌ ลืม
}
```

### ✅ Secure - All 4 Settings

```hcl
# ✅ SECURE - ครบทั้ง 4 settings
resource "aws_s3_bucket" "secure" {
  bucket = "sensitive-data-bucket"
}

resource "aws_s3_bucket_public_access_block" "secure" {
  bucket = aws_s3_bucket.secure.id
  
  block_public_acls       = true  # ✅ 1. Block new public ACLs
  block_public_policy     = true  # ✅ 2. Block new public policies
  ignore_public_acls      = true  # ✅ 3. Ignore existing public ACLs
  restrict_public_buckets = true  # ✅ 4. Restrict public bucket access
}

# ✅ เพิ่มเติม: Account-level block (ป้องกัน bucket ใหม่ด้วย)
resource "aws_s3_account_public_access_block" "account_wide" {
  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}
```

---

## ขั้นตอนที่ 814: Misconfiguration #3 - Unencrypted S3 Bucket

### ข้อมูล (Info)
- **CIS Control:** 2.1.1 
- **CVSS Score:** 7.5 (High) - ข้อมูล at-rest ไม่ถูกปกป้อง
- **Checkov Rule:** CKV_AWS_19, CKV2_AWS_67
- **Regulation:** GDPR Article 32, HIPAA §164.312, PCI-DSS Requirement 3.5

### ประเภท Encryption

```
SSE-S3 (Server-Side Encryption with S3 Managed Keys):
  - Key managed โดย AWS
  - AES-256
  - ไม่มีค่าใช้จ่ายเพิ่ม
  - จำกัดการ audit

SSE-KMS (Server-Side Encryption with KMS):
  - Key managed ใน AWS KMS
  - สามารถ audit ผ่าน CloudTrail
  - รองรับ customer managed keys (CMK)
  - มีค่าใช้จ่าย KMS API calls

SSE-C (Server-Side Encryption with Customer Keys):
  - Customer ส่ง key ทุก request
  - AWS ไม่เก็บ key
  - ซับซ้อนในการ manage
  - ไม่รองรับ pre-signed URLs
```

### ❌ Vulnerable - No Encryption

```hcl
# ❌ VULNERABLE - ไม่มี encryption
resource "aws_s3_bucket" "unencrypted" {
  bucket = "medical-records-bucket"
}

# ❌ ALSO VULNERABLE - encryption ที่อ่อนแอ
resource "aws_s3_bucket_server_side_encryption_configuration" "weak" {
  bucket = aws_s3_bucket.unencrypted.id
  
  rule {
    apply_server_side_encryption_by_default {
      sse_algorithm = "AES256"  # ⚠️ S3 managed keys - ดีกว่าไม่มี แต่ไม่ดีเท่า KMS
      # ❌ ไม่มี kms_master_key_id = ใช้ default AWS key
    }
  }
}
```

### ✅ Secure - KMS Encryption

```hcl
# ✅ Step 1: สร้าง KMS Key
resource "aws_kms_key" "s3_key" {
  description             = "S3 bucket encryption key"
  deletion_window_in_days = 30
  
  enable_key_rotation = true  # ✅ Auto-rotate annually
  
  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Sid    = "Enable IAM User Permissions"
        Effect = "Allow"
        Principal = {
          AWS = "arn:aws:iam::${var.account_id}:root"
        }
        Action   = "kms:*"
        Resource = "*"
      },
      {
        Sid    = "Allow S3 Service"
        Effect = "Allow"
        Principal = {
          Service = "s3.amazonaws.com"
        }
        Action = [
          "kms:Decrypt",
          "kms:GenerateDataKey"
        ]
        Resource = "*"
      }
    ]
  })
  
  tags = {
    Name    = "s3-encryption-key"
    Purpose = "S3 Bucket Encryption"
  }
}

resource "aws_kms_alias" "s3_key" {
  name          = "alias/s3-medical-records"
  target_key_id = aws_kms_key.s3_key.key_id
}

# ✅ Step 2: S3 Bucket
resource "aws_s3_bucket" "secure_encrypted" {
  bucket = "medical-records-bucket"
}

# ✅ Step 3: Enable KMS Encryption
resource "aws_s3_bucket_server_side_encryption_configuration" "secure" {
  bucket = aws_s3_bucket.secure_encrypted.id
  
  rule {
    apply_server_side_encryption_by_default {
      sse_algorithm     = "aws:kms"           # ✅ KMS encryption
      kms_master_key_id = aws_kms_key.s3_key.arn  # ✅ Customer managed key
    }
    bucket_key_enabled = true  # ✅ Reduces KMS API calls and costs
  }
}

# ✅ Step 4: Block public access
resource "aws_s3_bucket_public_access_block" "secure_encrypted" {
  bucket                  = aws_s3_bucket.secure_encrypted.id
  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}
```

---

## ขั้นตอนที่ 815: Misconfiguration #4 - Missing Versioning

### ข้อมูล (Info)
- **CIS Control:** ไม่มีเฉพาะเจาะจง แต่ best practice
- **CVSS Score:** 6.5 (Medium) - Risk การสูญเสียข้อมูล
- **Checkov Rule:** CKV_AWS_21
- **TFSec Rule:** aws-s3-enable-versioning

### ทำไม Versioning สำคัญ

```
ประโยชน์ของ S3 Versioning:
1. ป้องกัน Ransomware attacks
   - แม้ attacker encrypt objects คุณสามารถ restore เวอร์ชันก่อนหน้าได้

2. ป้องกัน Accidental deletion
   - Delete เป็นแค่ "delete marker" ไม่ใช่การลบจริง

3. Compliance requirements
   - HIPAA, PCI-DSS ต้องการ data integrity

4. Recovery options
   - Point-in-time recovery
   - Audit trail สำหรับ objects
```

### ❌ Vulnerable - No Versioning

```hcl
# ❌ VULNERABLE - ไม่มี versioning
resource "aws_s3_bucket" "no_versioning" {
  bucket = "company-contracts"
  # ❌ ไม่มี versioning = ลบแล้วหายถาวร
}

# ❌ ALSO VULNERABLE - versioning suspended
resource "aws_s3_bucket_versioning" "suspended" {
  bucket = aws_s3_bucket.no_versioning.id
  versioning_configuration {
    status = "Suspended"  # ❌ ปิด versioning
  }
}
```

### ✅ Secure - Versioning with Lifecycle

```hcl
# ✅ SECURE - Enable versioning
resource "aws_s3_bucket" "with_versioning" {
  bucket = "company-contracts"
}

resource "aws_s3_bucket_versioning" "enabled" {
  bucket = aws_s3_bucket.with_versioning.id
  versioning_configuration {
    status = "Enabled"  # ✅ Enable versioning
  }
}

# ✅ MFA Delete สำหรับ critical buckets
resource "aws_s3_bucket_versioning" "with_mfa_delete" {
  bucket = aws_s3_bucket.with_versioning.id
  
  # ต้องใช้ AWS CLI หรือ SDK กับ MFA credentials
  # ไม่สามารถ enable ผ่าน Terraform ได้โดยตรง
  # แต่สามารถ verify ได้
  
  versioning_configuration {
    status     = "Enabled"
    mfa_delete = "Enabled"  # ⚠️ ต้อง apply ด้วย root MFA
  }
}

# ✅ Lifecycle policy จัดการ old versions
resource "aws_s3_bucket_lifecycle_configuration" "versioning_lifecycle" {
  bucket = aws_s3_bucket.with_versioning.id
  
  rule {
    id     = "expire-old-versions"
    status = "Enabled"
    
    filter {
      prefix = ""  # Apply ทุก objects
    }
    
    # Keep current version ไปเรื่อยๆ
    
    # Non-current versions: move to IA after 30 days
    noncurrent_version_transition {
      noncurrent_days = 30
      storage_class   = "STANDARD_IA"
    }
    
    # Non-current versions: move to Glacier after 90 days
    noncurrent_version_transition {
      noncurrent_days = 90
      storage_class   = "GLACIER"
    }
    
    # Delete non-current versions after 365 days
    noncurrent_version_expiration {
      noncurrent_days = 365
    }
    
    # Delete expired delete markers
    expiration {
      expired_object_delete_marker = true
    }
    
    # Clean up incomplete multipart uploads
    abort_incomplete_multipart_upload {
      days_after_initiation = 7
    }
  }
}
```

---

## ขั้นตอนที่ 816: Misconfiguration #5 - Missing Access Logging

### ข้อมูล (Info)
- **CIS Control:** 2.6 (Ensure S3 bucket access logging is enabled)
- **CVSS Score:** 5.3 (Medium) - ไม่มี audit trail
- **Checkov Rule:** CKV_AWS_18
- **TFSec Rule:** aws-s3-enable-bucket-logging

### Real-World Impact
Capital One breach ทำให้ตระหนักถึงความสำคัญของ logging:
- ถ้ามี S3 access logging จะตรวจจับการเข้าถึงผิดปกติได้เร็วขึ้น
- SSRF attack ผ่าน metadata service จะเห็นใน logs

### ❌ Vulnerable - No Access Logging

```hcl
# ❌ VULNERABLE - ไม่มี access logging
resource "aws_s3_bucket" "app_data" {
  bucket = "app-customer-data"
  # ❌ ไม่มี logging = ไม่รู้ว่าใคร access อะไรเมื่อไหร่
}
```

### ✅ Secure - Access Logging

```hcl
# ✅ Step 1: สร้าง logging bucket แยกต่างหาก
resource "aws_s3_bucket" "access_logs" {
  bucket = "app-access-logs-${var.account_id}"
}

# ✅ Ownership controls for logging bucket
resource "aws_s3_bucket_ownership_controls" "access_logs" {
  bucket = aws_s3_bucket.access_logs.id
  rule {
    object_ownership = "ObjectWriter"
    # Note: logging ต้องการ ObjectWriter สำหรับ log delivery
  }
}

# ✅ Encryption สำหรับ log bucket
resource "aws_s3_bucket_server_side_encryption_configuration" "logs" {
  bucket = aws_s3_bucket.access_logs.id
  rule {
    apply_server_side_encryption_by_default {
      sse_algorithm = "AES256"
    }
  }
}

# ✅ Lifecycle สำหรับ log retention
resource "aws_s3_bucket_lifecycle_configuration" "logs_lifecycle" {
  bucket = aws_s3_bucket.access_logs.id
  
  rule {
    id     = "log-retention"
    status = "Enabled"
    
    filter {}
    
    transition {
      days          = 30
      storage_class = "STANDARD_IA"
    }
    
    transition {
      days          = 90
      storage_class = "GLACIER"
    }
    
    expiration {
      days = 365  # ✅ Keep 1 year
    }
  }
}

# ✅ Block public access on log bucket
resource "aws_s3_bucket_public_access_block" "access_logs" {
  bucket                  = aws_s3_bucket.access_logs.id
  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}

# ✅ Step 2: Enable logging on source bucket
resource "aws_s3_bucket" "app_data" {
  bucket = "app-customer-data"
}

resource "aws_s3_bucket_logging" "app_data" {
  bucket = aws_s3_bucket.app_data.id
  
  target_bucket = aws_s3_bucket.access_logs.id  # ✅ Separate logging bucket
  target_prefix = "app-customer-data-logs/"     # ✅ Organized prefix
}

# ✅ CloudWatch Metric Filter สำหรับ anomaly detection
resource "aws_cloudwatch_log_group" "s3_logs" {
  name              = "/aws/s3/access-logs"
  retention_in_days = 90
}

resource "aws_cloudwatch_metric_filter" "s3_unauthorized" {
  name           = "S3UnauthorizedAccess"
  pattern        = "[bucket_owner, bucket, time, remote_ip, requester, request_id, operation, key, request_uri, status_code=403 || status_code=401, ...]"
  log_group_name = aws_cloudwatch_log_group.s3_logs.name

  metric_transformation {
    name      = "S3UnauthorizedAccess"
    namespace = "SecurityMetrics"
    value     = "1"
  }
}
```

---

## ขั้นตอนที่ 817: Misconfiguration #6 - Overly Permissive Bucket Policy

### ข้อมูล (Info)
- **CVSS Score:** 9.8 (Critical) - อาจทำให้ทุกคน access ได้
- **Checkov Rule:** CKV_AWS_70, CKV_AWS_135
- **Common Pattern:** "*" Principal หรือ Action: "*"

### ❌ Vulnerable - Wildcard Principal

```hcl
# ❌ VULNERABLE - ทุกคนสามารถ GetObject ได้
resource "aws_s3_bucket_policy" "too_permissive" {
  bucket = aws_s3_bucket.data.id
  
  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect    = "Allow"
        Principal = "*"           # ❌ ทุกคน (anonymous + authenticated)
        Action    = "s3:GetObject"
        Resource  = "${aws_s3_bucket.data.arn}/*"
      }
    ]
  })
}

# ❌ ALSO VULNERABLE - ทุก action
resource "aws_s3_bucket_policy" "full_access" {
  bucket = aws_s3_bucket.data.id
  
  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect    = "Allow"
        Principal = { AWS = "arn:aws:iam::${var.account_id}:root" }
        Action    = "*"           # ❌ ทุก action รวม Delete
        Resource = [
          aws_s3_bucket.data.arn,
          "${aws_s3_bucket.data.arn}/*"
        ]
      }
    ]
  })
}

# ❌ VULNERABLE - Cross-account ไม่มี condition
resource "aws_s3_bucket_policy" "cross_account_no_condition" {
  bucket = aws_s3_bucket.data.id
  
  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Principal = {
          AWS = "arn:aws:iam::PARTNER_ACCOUNT:root"  # ❌ ไม่มี condition
        }
        Action   = ["s3:GetObject", "s3:PutObject"]
        Resource = "${aws_s3_bucket.data.arn}/*"
      }
    ]
  })
}
```

### ✅ Secure - Least Privilege Bucket Policy

```hcl
# ✅ SECURE - Specific roles + conditions
resource "aws_s3_bucket_policy" "least_privilege" {
  bucket = aws_s3_bucket.data.id
  
  # ต้อง apply public access block ก่อน policy
  depends_on = [aws_s3_bucket_public_access_block.data]
  
  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      # ✅ SSL required
      {
        Sid       = "DenyNonSSL"
        Effect    = "Deny"
        Principal = "*"
        Action    = "s3:*"
        Resource = [
          aws_s3_bucket.data.arn,
          "${aws_s3_bucket.data.arn}/*"
        ]
        Condition = {
          Bool = {
            "aws:SecureTransport" = "false"
          }
        }
      },
      # ✅ Application service role only
      {
        Sid    = "AllowApplicationAccess"
        Effect = "Allow"
        Principal = {
          AWS = aws_iam_role.app_role.arn  # ✅ Specific role
        }
        Action = [
          "s3:GetObject",    # ✅ เฉพาะ read
          "s3:PutObject",    # ✅ เฉพาะ write
          "s3:DeleteObject"  # ✅ delete ถ้าจำเป็น
        ]
        Resource  = "${aws_s3_bucket.data.arn}/*"
        Condition = {
          StringEquals = {
            "s3:prefix"                = ["app-data/"]  # ✅ เฉพาะ prefix
            "aws:RequestedRegion"      = "us-east-1"    # ✅ เฉพาะ region
          }
          IpAddress = {
            "aws:SourceIp" = var.allowed_cidr_blocks  # ✅ IP restriction
          }
        }
      },
      # ✅ Cross-account with External ID
      {
        Sid    = "AllowPartnerAccess"
        Effect = "Allow"
        Principal = {
          AWS = "arn:aws:iam::${var.partner_account_id}:role/DataAccess"
        }
        Action   = ["s3:GetObject"]  # ✅ Read only
        Resource = "${aws_s3_bucket.data.arn}/partner-data/*"  # ✅ Specific prefix
        Condition = {
          StringEquals = {
            "sts:ExternalId" = var.partner_external_id  # ✅ Prevents confused deputy
          }
          DateGreaterThan = {
            "aws:CurrentTime" = "2024-01-01T00:00:00Z"  # ✅ Time bound
          }
          DateLessThan = {
            "aws:CurrentTime" = "2024-12-31T23:59:59Z"
          }
        }
      }
    ]
  })
}
```

---

## ขั้นตอนที่ 818: Misconfiguration #7 - HTTP Access Allowed

### ข้อมูล (Info)
- **CIS Control:** Ensure S3 buckets enforce SSL
- **CVSS Score:** 7.4 (High) - Man-in-the-middle attacks
- **Checkov Rule:** CKV_AWS_20
- **Regulation:** PCI-DSS Requirement 4.1

### ❌ Vulnerable - No SSL Enforcement

```hcl
# ❌ VULNERABLE - ไม่บังคับ SSL
resource "aws_s3_bucket" "no_ssl" {
  bucket = "api-responses-cache"
}

# ไม่มี bucket policy ที่ deny HTTP = HTTP access allowed
```

### ✅ Secure - Force SSL Bucket Policy

```hcl
resource "aws_s3_bucket" "force_ssl" {
  bucket = "api-responses-cache"
}

# ✅ Force SSL via bucket policy
resource "aws_s3_bucket_policy" "force_ssl" {
  bucket = aws_s3_bucket.force_ssl.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Sid       = "ForceSSLOnly"
        Effect    = "Deny"
        Principal = "*"
        Action    = "s3:*"
        Resource = [
          aws_s3_bucket.force_ssl.arn,
          "${aws_s3_bucket.force_ssl.arn}/*"
        ]
        Condition = {
          Bool = {
            "aws:SecureTransport" = "false"  # ✅ Deny if not HTTPS
          }
        }
      }
    ]
  })
}

# ✅ Also enforce minimum TLS version via CloudFront
resource "aws_cloudfront_distribution" "s3_distribution" {
  enabled = true
  
  origin {
    domain_name = aws_s3_bucket.force_ssl.bucket_regional_domain_name
    origin_id   = "S3-${aws_s3_bucket.force_ssl.id}"
    
    s3_origin_config {
      origin_access_identity = aws_cloudfront_origin_access_identity.oai.cloudfront_access_identity_path
    }
  }
  
  default_cache_behavior {
    viewer_protocol_policy = "redirect-to-https"  # ✅ Redirect HTTP to HTTPS
    allowed_methods        = ["GET", "HEAD"]
    cached_methods         = ["GET", "HEAD"]
    target_origin_id       = "S3-${aws_s3_bucket.force_ssl.id}"
    
    forwarded_values {
      query_string = false
      cookies { forward = "none" }
    }
  }
  
  viewer_certificate {
    minimum_protocol_version = "TLSv1.2_2021"  # ✅ TLS 1.2+ only
    ssl_support_method       = "sni-only"
  }
  
  restrictions {
    geo_restriction { restriction_type = "none" }
  }
}
```

---

## ขั้นตอนที่ 819: Misconfiguration #8-12

### Misconfiguration #8: MFA Delete Not Enabled

```hcl
# ❌ VULNERABLE
resource "aws_s3_bucket_versioning" "no_mfa_delete" {
  bucket = aws_s3_bucket.critical.id
  versioning_configuration {
    status = "Enabled"
    # ❌ ไม่มี mfa_delete
  }
}

# ✅ SECURE - MFA Delete (ต้อง apply ด้วย MFA credentials)
# Note: Terraform ไม่สามารถ enable MFA delete ได้โดยตรง
# ต้องใช้ AWS CLI:
# aws s3api put-bucket-versioning \
#   --bucket critical-bucket \
#   --versioning-configuration Status=Enabled,MFADelete=Enabled \
#   --mfa "arn:aws:iam::ACCOUNT-ID:mfa/USERNAME MFA-TOKEN"

# แต่สามารถ verify ด้วย data source:
data "aws_s3_bucket" "verify_mfa_delete" {
  bucket = aws_s3_bucket.critical.id
}

# ✅ Use aws_s3_bucket_versioning to document the expected state
resource "aws_s3_bucket_versioning" "with_mfa_delete" {
  bucket = aws_s3_bucket.critical.id
  versioning_configuration {
    status     = "Enabled"
    mfa_delete = "Enabled"  # ⚠️ ต้องใช้ MFA credentials ใน provider
  }
}
```

### Misconfiguration #9: Cross-account Access Too Broad

```hcl
# ❌ VULNERABLE - ทั้ง account สามารถ access ได้
resource "aws_s3_bucket_policy" "cross_account_broad" {
  bucket = aws_s3_bucket.shared.id
  
  policy = jsonencode({
    Statement = [{
      Effect = "Allow"
      Principal = {
        AWS = "arn:aws:iam::${var.partner_account}:root"  # ❌ ทั้ง account
      }
      Action   = ["s3:GetObject", "s3:PutObject", "s3:DeleteObject"]
      Resource = "${aws_s3_bucket.shared.arn}/*"
      # ❌ ไม่มี condition
    }]
  })
}

# ✅ SECURE - Specific role + External ID + limited scope
resource "aws_s3_bucket_policy" "cross_account_secure" {
  bucket = aws_s3_bucket.shared.id
  
  policy = jsonencode({
    Statement = [
      # Deny non-SSL
      {
        Sid       = "DenyNonSSL"
        Effect    = "Deny"
        Principal = "*"
        Action    = "s3:*"
        Resource = ["${aws_s3_bucket.shared.arn}/*"]
        Condition = {
          Bool = { "aws:SecureTransport" = "false" }
        }
      },
      # Allow specific role with conditions
      {
        Sid    = "AllowPartnerRole"
        Effect = "Allow"
        Principal = {
          AWS = "arn:aws:iam::${var.partner_account}:role/DataAccessRole"  # ✅ Specific role
        }
        Action   = ["s3:GetObject"]  # ✅ Read only
        Resource = "${aws_s3_bucket.shared.arn}/partner/${var.partner_id}/*"  # ✅ Specific prefix
        Condition = {
          StringEquals = {
            "sts:ExternalId"       = var.external_id   # ✅ External ID
            "s3:prefix"            = ["partner/"]
            "aws:RequestedRegion"  = "us-east-1"
          }
          DateLessThan = {
            "aws:TokenIssueTime" = "2025-12-31T23:59:59Z"  # ✅ Time bound
          }
        }
      }
    ]
  })
}
```

### Misconfiguration #10: Server Access Logging to Same Bucket

```hcl
# ❌ VULNERABLE - Circular logging
resource "aws_s3_bucket" "circular_logging" {
  bucket = "my-bucket"
}

resource "aws_s3_bucket_logging" "circular" {
  bucket        = aws_s3_bucket.circular_logging.id
  target_bucket = aws_s3_bucket.circular_logging.id  # ❌ Same bucket!
  target_prefix = "logs/"
  # ปัญหา: log อ่าน log อ่าน log... infinite loop
  # ค่าใช้จ่ายพุ่งสูง, storage ล้น
}

# ✅ SECURE - Separate logging bucket
resource "aws_s3_bucket" "data" {
  bucket = "my-data-bucket"
}

resource "aws_s3_bucket" "logs" {
  bucket = "my-logs-bucket-${var.account_id}"  # ✅ Separate bucket
}

resource "aws_s3_bucket_logging" "data_logs" {
  bucket        = aws_s3_bucket.data.id
  target_bucket = aws_s3_bucket.logs.id    # ✅ Different bucket
  target_prefix = "my-data-bucket-access-logs/"
}
```

### Misconfiguration #11: Object Lock Not Enabled for Compliance

```hcl
# ❌ VULNERABLE - No Object Lock for regulated data
resource "aws_s3_bucket" "financial_records" {
  bucket = "financial-records-7years"
  # ❌ ไม่มี Object Lock
  # ❌ ข้อมูลสามารถลบได้ก่อน retention period
}

# ✅ SECURE - Object Lock for WORM storage
resource "aws_s3_bucket" "financial_records" {
  bucket = "financial-records-7years"
  
  object_lock_enabled = true  # ✅ ต้อง enable ตอนสร้าง bucket
}

resource "aws_s3_bucket_versioning" "financial_records" {
  bucket = aws_s3_bucket.financial_records.id
  versioning_configuration {
    status = "Enabled"
    # Object Lock ต้องการ versioning
  }
}

resource "aws_s3_bucket_object_lock_configuration" "financial_records" {
  bucket = aws_s3_bucket.financial_records.id
  
  rule {
    default_retention {
      mode  = "COMPLIANCE"  # ✅ Compliance mode - ไม่สามารถ override ได้แม้ root
      years = 7             # ✅ 7 year retention (SOX requirement)
    }
  }
}

# ✅ Also encrypt
resource "aws_s3_bucket_server_side_encryption_configuration" "financial_records" {
  bucket = aws_s3_bucket.financial_records.id
  rule {
    apply_server_side_encryption_by_default {
      sse_algorithm     = "aws:kms"
      kms_master_key_id = aws_kms_key.financial.arn
    }
    bucket_key_enabled = true
  }
}
```

### Misconfiguration #12: S3 Replication Without Encryption

```hcl
# ❌ VULNERABLE - Replication without encryption
resource "aws_s3_bucket_replication_configuration" "no_encryption" {
  role   = aws_iam_role.replication.arn
  bucket = aws_s3_bucket.source.id
  
  rule {
    id     = "replicate-all"
    status = "Enabled"
    
    destination {
      bucket = aws_s3_bucket.destination.arn
      # ❌ ไม่มี encryption configuration
    }
  }
}

# ✅ SECURE - Replication with encryption
resource "aws_s3_bucket_replication_configuration" "with_encryption" {
  role   = aws_iam_role.replication.arn
  bucket = aws_s3_bucket.source.id
  
  rule {
    id     = "replicate-all-encrypted"
    status = "Enabled"
    
    filter {
      prefix = ""  # replicate everything
    }
    
    source_selection_criteria {
      sse_kms_encrypted_objects {
        status = "Enabled"  # ✅ Replicate encrypted objects
      }
    }
    
    destination {
      bucket = aws_s3_bucket.destination.arn
      
      encryption_configuration {
        replica_kms_key_id = aws_kms_key.destination_key.arn  # ✅ Encrypt at destination
      }
      
      access_control_translation {
        owner = "Destination"  # ✅ Destination account owns replicas
      }
      
      metrics {
        status = "Enabled"
        event_threshold {
          minutes = 15
        }
      }
      
      replication_time {
        status = "Enabled"
        time {
          minutes = 15  # ✅ SLA for replication
        }
      }
    }
    
    delete_marker_replication {
      status = "Enabled"  # ✅ Replicate deletes too
    }
  }
}

# ✅ IAM Role for replication with least privilege
resource "aws_iam_role" "replication" {
  name = "s3-replication-role"
  
  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect    = "Allow"
      Principal = { Service = "s3.amazonaws.com" }
      Action    = "sts:AssumeRole"
    }]
  })
}

resource "aws_iam_role_policy" "replication" {
  name = "s3-replication-policy"
  role = aws_iam_role.replication.id
  
  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Action = [
          "s3:GetReplicationConfiguration",
          "s3:ListBucket"
        ]
        Resource = aws_s3_bucket.source.arn
      },
      {
        Effect = "Allow"
        Action = [
          "s3:GetObjectVersionForReplication",
          "s3:GetObjectVersionAcl",
          "s3:GetObjectVersionTagging"
        ]
        Resource = "${aws_s3_bucket.source.arn}/*"
      },
      {
        Effect = "Allow"
        Action = [
          "s3:ReplicateObject",
          "s3:ReplicateDelete",
          "s3:ReplicateTags"
        ]
        Resource = "${aws_s3_bucket.destination.arn}/*"
      },
      {
        Effect = "Allow"
        Action = [
          "kms:Decrypt"
        ]
        Resource = aws_kms_key.source_key.arn
      },
      {
        Effect = "Allow"
        Action = [
          "kms:Encrypt"
        ]
        Resource = aws_kms_key.destination_key.arn
      }
    ]
  })
}
```

---

## ขั้นตอนที่ 820: S3 Security - Complete Secure Module

### S3 Secure Bucket Module

```hcl
# modules/secure-s3-bucket/main.tf
# ✅ Complete secure S3 bucket module

terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

variable "bucket_name" {
  description = "Name of the S3 bucket"
  type        = string
  
  validation {
    condition     = can(regex("^[a-z0-9][a-z0-9\\-]{1,61}[a-z0-9]$", var.bucket_name))
    error_message = "Bucket name must be lowercase alphanumeric and hyphens, 3-63 chars."
  }
}

variable "enable_versioning" {
  description = "Enable S3 versioning"
  type        = bool
  default     = true
}

variable "enable_object_lock" {
  description = "Enable S3 Object Lock (WORM)"
  type        = bool
  default     = false
}

variable "object_lock_retention_days" {
  description = "Object Lock retention in days"
  type        = number
  default     = 365
}

variable "kms_key_arn" {
  description = "ARN of KMS key for encryption"
  type        = string
  default     = null
}

variable "log_bucket_id" {
  description = "ID of the S3 bucket for access logs"
  type        = string
}

variable "allowed_principals" {
  description = "List of IAM principal ARNs allowed to access the bucket"
  type        = list(string)
  default     = []
}

variable "tags" {
  description = "Tags to apply to resources"
  type        = map(string)
  default     = {}
}

# Main bucket
resource "aws_s3_bucket" "this" {
  bucket = var.bucket_name
  
  object_lock_enabled = var.enable_object_lock
  
  tags = merge(var.tags, {
    Module   = "secure-s3-bucket"
    ManagedBy = "terraform"
  })
  
  lifecycle {
    prevent_destroy = false  # Set to true for production
  }
}

# Block ALL public access
resource "aws_s3_bucket_public_access_block" "this" {
  bucket                  = aws_s3_bucket.this.id
  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}

# Ownership controls
resource "aws_s3_bucket_ownership_controls" "this" {
  bucket = aws_s3_bucket.this.id
  rule {
    object_ownership = "BucketOwnerEnforced"
  }
}

# Versioning
resource "aws_s3_bucket_versioning" "this" {
  bucket = aws_s3_bucket.this.id
  versioning_configuration {
    status = var.enable_versioning ? "Enabled" : "Disabled"
  }
}

# Server-side encryption
resource "aws_s3_bucket_server_side_encryption_configuration" "this" {
  bucket = aws_s3_bucket.this.id
  
  rule {
    apply_server_side_encryption_by_default {
      sse_algorithm     = var.kms_key_arn != null ? "aws:kms" : "AES256"
      kms_master_key_id = var.kms_key_arn
    }
    bucket_key_enabled = true
  }
}

# Access logging
resource "aws_s3_bucket_logging" "this" {
  bucket        = aws_s3_bucket.this.id
  target_bucket = var.log_bucket_id
  target_prefix = "${var.bucket_name}/"
}

# Lifecycle rules
resource "aws_s3_bucket_lifecycle_configuration" "this" {
  bucket     = aws_s3_bucket.this.id
  depends_on = [aws_s3_bucket_versioning.this]
  
  rule {
    id     = "lifecycle-management"
    status = "Enabled"
    
    filter { prefix = "" }
    
    # Transition current versions
    transition {
      days          = 90
      storage_class = "STANDARD_IA"
    }
    transition {
      days          = 365
      storage_class = "GLACIER"
    }
    
    # Handle non-current versions
    dynamic "noncurrent_version_transition" {
      for_each = var.enable_versioning ? [1] : []
      content {
        noncurrent_days = 30
        storage_class   = "STANDARD_IA"
      }
    }
    
    dynamic "noncurrent_version_expiration" {
      for_each = var.enable_versioning ? [1] : []
      content {
        noncurrent_days = 90
      }
    }
    
    # Clean up incomplete multipart uploads
    abort_incomplete_multipart_upload {
      days_after_initiation = 7
    }
  }
}

# Object Lock configuration
resource "aws_s3_bucket_object_lock_configuration" "this" {
  count  = var.enable_object_lock ? 1 : 0
  bucket = aws_s3_bucket.this.id
  
  rule {
    default_retention {
      mode = "GOVERNANCE"  # Use COMPLIANCE for strict requirements
      days = var.object_lock_retention_days
    }
  }
}

# Bucket policy
resource "aws_s3_bucket_policy" "this" {
  bucket     = aws_s3_bucket.this.id
  depends_on = [aws_s3_bucket_public_access_block.this]
  
  policy = jsonencode({
    Version = "2012-10-17"
    Statement = concat(
      # Always deny non-SSL
      [{
        Sid       = "DenyNonSSL"
        Effect    = "Deny"
        Principal = "*"
        Action    = "s3:*"
        Resource = [
          aws_s3_bucket.this.arn,
          "${aws_s3_bucket.this.arn}/*"
        ]
        Condition = {
          Bool = {
            "aws:SecureTransport" = "false"
          }
        }
      }],
      # Allowed principals
      length(var.allowed_principals) > 0 ? [{
        Sid    = "AllowAuthorizedPrincipals"
        Effect = "Allow"
        Principal = {
          AWS = var.allowed_principals
        }
        Action = [
          "s3:GetObject",
          "s3:PutObject",
          "s3:DeleteObject",
          "s3:ListBucket"
        ]
        Resource = [
          aws_s3_bucket.this.arn,
          "${aws_s3_bucket.this.arn}/*"
        ]
      }] : []
    )
  })
}

# Outputs
output "bucket_id" {
  description = "ID of the S3 bucket"
  value       = aws_s3_bucket.this.id
}

output "bucket_arn" {
  description = "ARN of the S3 bucket"
  value       = aws_s3_bucket.this.arn
}

output "bucket_domain_name" {
  description = "Domain name of the S3 bucket"
  value       = aws_s3_bucket.this.bucket_regional_domain_name
}
```

### ใช้งาน Secure S3 Module

```hcl
# main.tf - การใช้งาน secure module
module "app_data_bucket" {
  source = "./modules/secure-s3-bucket"
  
  bucket_name       = "myapp-customer-data-${var.environment}"
  enable_versioning = true
  enable_object_lock = false
  kms_key_arn       = aws_kms_key.app_data.arn
  log_bucket_id     = module.log_bucket.bucket_id
  
  allowed_principals = [
    aws_iam_role.app_role.arn,
    aws_iam_role.data_pipeline.arn
  ]
  
  tags = {
    Environment = var.environment
    Application = "myapp"
    DataClass   = "confidential"
    Owner       = "data-team"
  }
}

module "compliance_bucket" {
  source = "./modules/secure-s3-bucket"
  
  bucket_name                = "myapp-compliance-data-${var.environment}"
  enable_versioning          = true
  enable_object_lock         = true  # WORM for compliance
  object_lock_retention_days = 2555  # 7 years
  kms_key_arn                = aws_kms_key.compliance.arn
  log_bucket_id              = module.log_bucket.bucket_id
  
  tags = {
    Environment  = var.environment
    Application  = "compliance"
    DataClass    = "regulated"
    Regulation   = "SOX"
    RetentionYrs = "7"
  }
}
```

---

## สรุป S3 Security Misconfigurations

### CIS Benchmark Summary

| Control | Description | Checkov Rule |
|---------|-------------|-------------|
| CIS 2.1.1 | Block Public Access | CKV_AWS_53-56 |
| CIS 2.1.2 | MFA Delete | CKV_AWS_21 |
| CIS 2.1.3 | Lifecycle policies | CKV_AWS_21 |
| CIS 2.1.4 | Access Logging | CKV_AWS_18 |
| CIS 2.1.5 | SSE Encryption | CKV_AWS_19 |

### Security Matrix

| Risk | Severity | CVSS | Fix |
|------|----------|------|-----|
| Public bucket | Critical | 9.8 | Block Public Access |
| No encryption | High | 7.5 | SSE-KMS |
| No versioning | Medium | 5.5 | Enable versioning |
| No logging | Medium | 5.3 | Enable access logs |
| HTTP allowed | High | 7.4 | Force SSL policy |
| Wildcard policy | Critical | 9.1 | Least privilege |
| No MFA delete | Medium | 6.0 | Enable MFA delete |
| No Object Lock | Medium | 5.5 | Enable WORM |

---

*Part 082 ครอบคลุม S3 Misconfigurations ทั้งหมด - ต่อไปใน Part 083 จะเจาะลึก IAM Security*
