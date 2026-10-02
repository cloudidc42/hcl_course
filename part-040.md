# Part 040: AWS S3 Buckets
# AWS S3 Buckets กับ Terraform - ครบทุกด้าน

## Steps 391-400: การสร้างและจัดการ S3 Buckets อย่างปลอดภัย

---

## Step 391: aws_s3_bucket Resource

### S3 Bucket พื้นฐาน

```hcl
# s3_basic.tf

resource "aws_s3_bucket" "main" {
  bucket = "${var.project_name}-${var.environment}-data"
  
  # ✅ Force destroy เฉพาะ development
  # Production ควรตั้งเป็น false
  force_destroy = var.environment != "production"

  tags = {
    Name        = "${var.project_name}-${var.environment}-data"
    Environment = var.environment
    ManagedBy   = "terraform"
  }
}
```

### S3 Bucket Naming Rules

```
S3 Bucket Naming Requirements:
- ยาว 3-63 characters
- ตัวพิมพ์เล็ก, ตัวเลข, และ hyphen เท่านั้น
- ต้องขึ้นต้นและลงท้ายด้วยตัวพิมพ์เล็กหรือตัวเลข
- ไม่สามารถมี consecutive hyphen
- ต้อง globally unique ทั่วโลก!

❌ ไม่ได้:
- MyBucket       (uppercase)
- my_bucket      (underscore)
- -mybucket      (ขึ้นต้นด้วย -)
- 192.168.1.1    (เหมือน IP address)

✅ ได้:
- my-bucket-123
- myproject-production-data-2024
- terraform-state-abcd1234
```

### Random Suffix สำหรับ Unique Naming

```hcl
# ─── Unique bucket names ──────────────────────────────────

resource "random_id" "bucket_suffix" {
  byte_length = 4
}

resource "aws_s3_bucket" "unique" {
  bucket = "${var.project_name}-${var.environment}-${random_id.bucket_suffix.hex}"
  
  tags = {
    Name = "${var.project_name}-${var.environment}-data"
  }
}

# หรือใช้ Account ID เพื่อ uniqueness
data "aws_caller_identity" "current" {}

resource "aws_s3_bucket" "with_account_id" {
  bucket = "${var.project_name}-${var.environment}-${data.aws_caller_identity.current.account_id}"
  
  tags = { Name = "${var.project_name}-data" }
}
```

---

## Step 392: aws_s3_bucket_versioning

### เปิดใช้ Versioning

```hcl
# s3_versioning.tf

resource "aws_s3_bucket_versioning" "main" {
  bucket = aws_s3_bucket.main.id

  versioning_configuration {
    # Enabled  - เปิด versioning
    # Suspended - หยุดสร้าง versions ใหม่ (versions เก่ายังอยู่)
    # Disabled  - ปิด versioning (default สำหรับ new buckets)
    status = "Enabled"

    # MFA Delete - ต้องใช้ MFA เพื่อลบ version หรือ suspend versioning
    # mfa_delete = "Enabled"  # ต้องใช้ MFA (optional, ยาก configure)
  }
}
```

### Versioning Use Cases

```hcl
# ─── Production Bucket: Versioning + MFA Delete ──────────

resource "aws_s3_bucket" "production_data" {
  bucket = "${var.project_name}-production-critical-data"
  
  tags = { Classification = "critical" }
}

resource "aws_s3_bucket_versioning" "production_data" {
  bucket = aws_s3_bucket.production_data.id

  versioning_configuration {
    status = "Enabled"
    # mfa_delete = "Enabled"  # ✅ Enable สำหรับ critical data
  }
}

# ─── Terraform State Bucket: Versioning จำเป็น! ──────────

resource "aws_s3_bucket" "terraform_state" {
  bucket = "terraform-state-${data.aws_caller_identity.current.account_id}"
  
  # ✅ ห้ามลบ bucket ที่เก็บ terraform state
  lifecycle {
    prevent_destroy = true
  }
}

resource "aws_s3_bucket_versioning" "terraform_state" {
  bucket = aws_s3_bucket.terraform_state.id

  versioning_configuration {
    status = "Enabled"  # ✅ จำเป็นสำหรับ state bucket
  }
}
```

---

## Step 393: Encryption (aws_s3_bucket_server_side_encryption_configuration)

### SSE-S3 (Default AWS-managed keys)

```hcl
# s3_encryption.tf

# ─── SSE-S3: AWS-managed keys (เร็ว, ฟรี) ───────────────

resource "aws_s3_bucket_server_side_encryption_configuration" "sse_s3" {
  bucket = aws_s3_bucket.main.id

  rule {
    apply_server_side_encryption_by_default {
      sse_algorithm = "AES256"  # SSE-S3
    }
    bucket_key_enabled = true  # ลด API calls ไปยัง KMS
  }
}
```

### SSE-KMS (Customer-managed keys)

```hcl
# ─── KMS Key สำหรับ S3 ────────────────────────────────────

resource "aws_kms_key" "s3" {
  description             = "KMS key สำหรับ S3 encryption"
  deletion_window_in_days = 30
  enable_key_rotation     = true  # ✅ Rotate keys อัตโนมัติ

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
        Sid    = "Allow S3 Service"
        Effect = "Allow"
        Principal = {
          Service = "s3.amazonaws.com"
        }
        Action = [
          "kms:GenerateDataKey",
          "kms:Decrypt"
        ]
        Resource = "*"
      }
    ]
  })

  tags = { Name = "${var.project_name}-s3-kms" }
}

resource "aws_kms_alias" "s3" {
  name          = "alias/${var.project_name}-s3"
  target_key_id = aws_kms_key.s3.key_id
}

# ─── SSE-KMS Encryption ───────────────────────────────────

resource "aws_s3_bucket_server_side_encryption_configuration" "sse_kms" {
  bucket = aws_s3_bucket.secure.id

  rule {
    apply_server_side_encryption_by_default {
      sse_algorithm     = "aws:kms"          # SSE-KMS
      kms_master_key_id = aws_kms_key.s3.arn # ใช้ custom KMS key
    }
    bucket_key_enabled = true  # ✅ ลด KMS API costs อย่างมาก
  }
}
```

---

## Step 394: Public Access Block

### Block Public Access (Critical Security!)

```hcl
# s3_public_access_block.tf

# ✅ ALWAYS block public access (ยกเว้นมีเหตุผลชัดเจน)
resource "aws_s3_bucket_public_access_block" "main" {
  bucket = aws_s3_bucket.main.id

  # Block public ACLs - ห้าม set public ACLs
  block_public_acls       = true

  # Ignore public ACLs - ignore existing public ACLs
  ignore_public_acls      = true

  # Block public bucket policies - ห้าม bucket policy ที่ทำให้ public
  block_public_policy     = true

  # Restrict public buckets - restrict access ถ้ามี public policy
  restrict_public_buckets = true
}

# ─── Account-level Block Public Access ──────────────────
# ✅ แนะนำ: ตั้งค่าระดับ account ด้วย

resource "aws_s3_account_public_access_block" "main" {
  block_public_acls       = true
  ignore_public_acls      = true
  block_public_policy     = true
  restrict_public_buckets = true
}
```

### Static Website Exception

```hcl
# ─── เฉพาะ public website bucket ────────────────────────

resource "aws_s3_bucket" "website" {
  bucket = "${var.project_name}-website"
}

resource "aws_s3_bucket_public_access_block" "website" {
  bucket = aws_s3_bucket.website.id

  # ⚠️ ยกเว้น block สำหรับ public website
  block_public_acls       = false  # อนุญาต ACLs
  ignore_public_acls      = false  # อ่าน ACLs
  block_public_policy     = false  # อนุญาต public bucket policy
  restrict_public_buckets = false  # ไม่ restrict
}

resource "aws_s3_bucket_policy" "website" {
  bucket = aws_s3_bucket.website.id

  depends_on = [aws_s3_bucket_public_access_block.website]

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect    = "Allow"
      Principal = "*"
      Action    = "s3:GetObject"
      Resource  = "${aws_s3_bucket.website.arn}/*"
    }]
  })
}
```

---

## Step 395: Bucket Policies

### Common Bucket Policy Patterns

```hcl
# s3_policies.tf

# ─── Pattern 1: Force SSL ─────────────────────────────────

resource "aws_s3_bucket_policy" "force_ssl" {
  bucket = aws_s3_bucket.main.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Sid       = "DenyNonHTTPS"
        Effect    = "Deny"
        Principal = "*"
        Action    = "s3:*"
        Resource = [
          aws_s3_bucket.main.arn,
          "${aws_s3_bucket.main.arn}/*"
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

# ─── Pattern 2: Cross-account Access ─────────────────────

resource "aws_s3_bucket_policy" "cross_account" {
  bucket = aws_s3_bucket.shared.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Sid    = "AllowCrossAccountAccess"
        Effect = "Allow"
        Principal = {
          AWS = [
            "arn:aws:iam::${var.staging_account_id}:root",
            "arn:aws:iam::${var.production_account_id}:root"
          ]
        }
        Action = [
          "s3:GetObject",
          "s3:ListBucket"
        ]
        Resource = [
          aws_s3_bucket.shared.arn,
          "${aws_s3_bucket.shared.arn}/*"
        ]
      },
      {
        Sid    = "DenyNonHTTPS"
        Effect = "Deny"
        Principal = "*"
        Action    = "s3:*"
        Resource = [
          aws_s3_bucket.shared.arn,
          "${aws_s3_bucket.shared.arn}/*"
        ]
        Condition = {
          Bool = { "aws:SecureTransport" = "false" }
        }
      }
    ]
  })
}

# ─── Pattern 3: CloudFront OAC (Origin Access Control) ───

resource "aws_cloudfront_origin_access_control" "s3" {
  name                              = "${var.project_name}-s3-oac"
  origin_access_control_origin_type = "s3"
  signing_behavior                  = "always"
  signing_protocol                  = "sigv4"
}

data "aws_iam_policy_document" "cloudfront_oac" {
  statement {
    principals {
      type        = "Service"
      identifiers = ["cloudfront.amazonaws.com"]
    }

    actions   = ["s3:GetObject"]
    resources = ["${aws_s3_bucket.website.arn}/*"]

    condition {
      test     = "StringEquals"
      variable = "AWS:SourceArn"
      values   = [aws_cloudfront_distribution.main.arn]
    }
  }
}

resource "aws_s3_bucket_policy" "cloudfront_oac" {
  bucket = aws_s3_bucket.website.id
  policy = data.aws_iam_policy_document.cloudfront_oac.json
}

# ─── Pattern 4: Terraform State Bucket Policy ────────────

data "aws_iam_policy_document" "terraform_state" {
  # Force SSL
  statement {
    sid     = "DenyNonHTTPS"
    effect  = "Deny"
    actions = ["s3:*"]
    resources = [
      aws_s3_bucket.terraform_state.arn,
      "${aws_s3_bucket.terraform_state.arn}/*"
    ]
    principals {
      type        = "*"
      identifiers = ["*"]
    }
    condition {
      test     = "Bool"
      variable = "aws:SecureTransport"
      values   = ["false"]
    }
  }

  # Allow only terraform roles
  statement {
    sid     = "AllowTerraformRoles"
    effect  = "Allow"
    actions = [
      "s3:GetObject",
      "s3:PutObject",
      "s3:DeleteObject",
      "s3:ListBucket",
      "s3:GetBucketVersioning"
    ]
    resources = [
      aws_s3_bucket.terraform_state.arn,
      "${aws_s3_bucket.terraform_state.arn}/*"
    ]
    principals {
      type        = "AWS"
      identifiers = [
        "arn:aws:iam::${data.aws_caller_identity.current.account_id}:role/TerraformRole",
        aws_iam_role.github_actions.arn
      ]
    }
  }
}

resource "aws_s3_bucket_policy" "terraform_state" {
  bucket = aws_s3_bucket.terraform_state.id
  policy = data.aws_iam_policy_document.terraform_state.json
  
  depends_on = [aws_s3_bucket_public_access_block.terraform_state]
}
```

---

## Step 396: Lifecycle Configuration

### Lifecycle Rules

```hcl
# s3_lifecycle.tf

resource "aws_s3_bucket_lifecycle_configuration" "main" {
  bucket = aws_s3_bucket.data.id
  
  # ต้องรอ versioning ถ้ามีการใช้ noncurrent rules
  depends_on = [aws_s3_bucket_versioning.data]

  # ─── Rule 1: Transition to cheaper storage ────────────────

  rule {
    id     = "transition-old-objects"
    status = "Enabled"

    # Apply กับทุก objects (ไม่มี filter)
    filter {
      prefix = "data/"  # เฉพาะ objects ที่ path ขึ้นต้นด้วย data/
    }

    # After 30 days → Intelligent Tiering
    transition {
      days          = 30
      storage_class = "INTELLIGENT_TIERING"
    }

    # After 90 days → Standard-IA (Infrequent Access)
    transition {
      days          = 90
      storage_class = "STANDARD_IA"
    }

    # After 180 days → Glacier Instant Retrieval
    transition {
      days          = 180
      storage_class = "GLACIER_IR"
    }

    # After 365 days → Glacier (cheapest)
    transition {
      days          = 365
      storage_class = "GLACIER"
    }

    # After 730 days → Delete
    expiration {
      days = 730  # 2 years
    }
  }

  # ─── Rule 2: Old versions ─────────────────────────────────

  rule {
    id     = "cleanup-old-versions"
    status = "Enabled"

    filter {}  # Apply to all objects

    # Transition noncurrent versions
    noncurrent_version_transition {
      noncurrent_days = 30
      storage_class   = "STANDARD_IA"
    }

    noncurrent_version_transition {
      noncurrent_days = 60
      storage_class   = "GLACIER"
    }

    # Delete old versions after 90 days
    noncurrent_version_expiration {
      noncurrent_days           = 90
      newer_noncurrent_versions = 5  # เก็บ 5 versions ล่าสุดไว้
    }
  }

  # ─── Rule 3: Delete incomplete multipart uploads ──────────

  rule {
    id     = "cleanup-incomplete-multipart"
    status = "Enabled"

    filter {}

    abort_incomplete_multipart_upload {
      days_after_initiation = 7  # ลบ incomplete uploads หลัง 7 วัน
    }
  }

  # ─── Rule 4: Logs expiration ─────────────────────────────

  rule {
    id     = "expire-logs"
    status = "Enabled"

    filter {
      prefix = "logs/"
    }

    expiration {
      days = 90
    }
  }
}
```

---

## Step 397: Website Configuration, CORS, Logging

### Website Configuration

```hcl
# s3_website.tf

resource "aws_s3_bucket_website_configuration" "main" {
  bucket = aws_s3_bucket.website.id

  index_document {
    suffix = "index.html"
  }

  error_document {
    key = "error.html"
  }

  # Routing rules (optional)
  routing_rule {
    condition {
      key_prefix_equals = "docs/"
    }
    redirect {
      replace_key_prefix_with = "documentation/"
    }
  }
}

output "website_endpoint" {
  value = aws_s3_bucket_website_configuration.main.website_endpoint
}
```

### CORS Configuration

```hcl
# s3_cors.tf

resource "aws_s3_bucket_cors_configuration" "api" {
  bucket = aws_s3_bucket.api_uploads.id

  cors_rule {
    allowed_headers = ["*"]
    allowed_methods = ["GET", "PUT", "POST", "DELETE", "HEAD"]
    allowed_origins = [
      "https://${var.domain_name}",
      "https://www.${var.domain_name}",
    ]
    expose_headers  = ["ETag", "Content-Length"]
    max_age_seconds = 3000
  }

  # Development: allow localhost
  cors_rule {
    allowed_headers = ["*"]
    allowed_methods = ["GET", "PUT", "POST"]
    allowed_origins = ["http://localhost:3000", "http://localhost:8080"]
    max_age_seconds = 0
  }
}
```

### Access Logging

```hcl
# s3_logging.tf

# Bucket สำหรับ เก็บ logs
resource "aws_s3_bucket" "access_logs" {
  bucket = "${var.project_name}-${var.environment}-access-logs"

  tags = { Purpose = "s3-access-logs" }
}

resource "aws_s3_bucket_lifecycle_configuration" "access_logs" {
  bucket = aws_s3_bucket.access_logs.id

  rule {
    id     = "expire-access-logs"
    status = "Enabled"

    filter {}

    expiration {
      days = 90
    }
  }
}

# เปิด Access Logging สำหรับ buckets อื่น
resource "aws_s3_bucket_logging" "main" {
  bucket = aws_s3_bucket.main.id

  target_bucket = aws_s3_bucket.access_logs.id
  target_prefix = "s3-logs/${aws_s3_bucket.main.id}/"
}
```

---

## Step 398: Notifications, Replication, Ownership Controls

### Notifications

```hcl
# s3_notifications.tf

# SNS Topic สำหรับรับ notifications
resource "aws_sns_topic" "s3_events" {
  name = "${var.project_name}-s3-events"
}

resource "aws_sns_topic_policy" "s3_events" {
  arn = aws_sns_topic.s3_events.arn

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect    = "Allow"
      Principal = { Service = "s3.amazonaws.com" }
      Action    = "SNS:Publish"
      Resource  = aws_sns_topic.s3_events.arn
      Condition = {
        ArnLike = {
          "aws:SourceArn" = aws_s3_bucket.data.arn
        }
      }
    }]
  })
}

resource "aws_s3_bucket_notification" "data" {
  bucket = aws_s3_bucket.data.id

  # SNS Notification
  topic {
    topic_arn     = aws_sns_topic.s3_events.arn
    events        = ["s3:ObjectCreated:*"]
    filter_prefix = "uploads/"
    filter_suffix = ".csv"
  }

  # SQS Queue Notification
  queue {
    queue_arn     = aws_sqs_queue.s3_events.arn
    events        = ["s3:ObjectCreated:*", "s3:ObjectRemoved:*"]
    filter_prefix = "data/"
  }

  # Lambda Notification
  lambda_function {
    lambda_function_arn = aws_lambda_function.process_upload.arn
    events              = ["s3:ObjectCreated:Put"]
    filter_prefix       = "raw/"
    filter_suffix       = ".json"
  }
}
```

### Replication Configuration

```hcl
# s3_replication.tf

# ─── Cross-region Replication ─────────────────────────────

# Source bucket (ap-southeast-1)
resource "aws_s3_bucket" "source" {
  provider = aws.ap_southeast_1
  bucket   = "${var.project_name}-source-${data.aws_caller_identity.current.account_id}"
}

resource "aws_s3_bucket_versioning" "source" {
  provider = aws.ap_southeast_1
  bucket   = aws_s3_bucket.source.id
  versioning_configuration { status = "Enabled" }
}

# Destination bucket (us-east-1)
resource "aws_s3_bucket" "destination" {
  provider = aws.us_east_1
  bucket   = "${var.project_name}-destination-${data.aws_caller_identity.current.account_id}"
}

resource "aws_s3_bucket_versioning" "destination" {
  provider = aws.us_east_1
  bucket   = aws_s3_bucket.destination.id
  versioning_configuration { status = "Enabled" }
}

# IAM Role สำหรับ replication
resource "aws_iam_role" "replication" {
  name = "${var.project_name}-s3-replication-role"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Action    = "sts:AssumeRole"
      Effect    = "Allow"
      Principal = { Service = "s3.amazonaws.com" }
    }]
  })
}

resource "aws_iam_role_policy" "replication" {
  name = "replication-policy"
  role = aws_iam_role.replication.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Action = ["s3:GetReplicationConfiguration", "s3:ListBucket"]
        Resource = [aws_s3_bucket.source.arn]
      },
      {
        Effect = "Allow"
        Action = ["s3:GetObjectVersionForReplication", "s3:GetObjectVersionAcl", "s3:GetObjectVersionTagging"]
        Resource = ["${aws_s3_bucket.source.arn}/*"]
      },
      {
        Effect = "Allow"
        Action = ["s3:ReplicateObject", "s3:ReplicateDelete", "s3:ReplicateTags"]
        Resource = ["${aws_s3_bucket.destination.arn}/*"]
      }
    ]
  })
}

# Replication Configuration
resource "aws_s3_bucket_replication_configuration" "main" {
  provider = aws.ap_southeast_1
  bucket   = aws_s3_bucket.source.id
  role     = aws_iam_role.replication.arn

  rule {
    id     = "replicate-all"
    status = "Enabled"

    filter {}  # Replicate all objects

    destination {
      bucket        = aws_s3_bucket.destination.arn
      storage_class = "STANDARD_IA"  # ลด cost ใน destination

      encryption_configuration {
        replica_kms_key_id = aws_kms_key.s3_dest.arn
      }

      replication_time {
        status  = "Enabled"
        time {
          minutes = 15  # RTO ไม่เกิน 15 นาที
        }
      }

      metrics {
        status = "Enabled"
        event_threshold {
          minutes = 15
        }
      }
    }

    source_selection_criteria {
      sse_kms_encrypted_objects {
        status = "Enabled"
      }
    }

    delete_marker_replication {
      status = "Enabled"
    }
  }

  depends_on = [
    aws_s3_bucket_versioning.source,
    aws_s3_bucket_versioning.destination
  ]
}
```

### Ownership Controls

```hcl
# s3_ownership.tf

# ✅ แนะนำ: Object Ownership เพื่อ disable ACLs
resource "aws_s3_bucket_ownership_controls" "main" {
  bucket = aws_s3_bucket.main.id

  rule {
    # BucketOwnerEnforced - ปิด ACLs, bucket owner owns all objects
    # BucketOwnerPreferred - bucket owner owns เมื่อ uploaded with bucket-owner-full-control
    # ObjectWriter - object uploader owns objects (legacy)
    object_ownership = "BucketOwnerEnforced"
  }
}
```

---

## Step 399: aws_s3_object สำหรับ Upload Files

```hcl
# s3_objects.tf

# ─── Single file ──────────────────────────────────────────

resource "aws_s3_object" "index_html" {
  bucket       = aws_s3_bucket.website.id
  key          = "index.html"
  source       = "${path.module}/files/index.html"
  content_type = "text/html"
  etag         = filemd5("${path.module}/files/index.html")  # Track changes

  tags = { ManagedBy = "terraform" }
}

# ─── Upload Configuration files ──────────────────────────

resource "aws_s3_object" "app_config" {
  bucket  = aws_s3_bucket.config.id
  key     = "config/${var.environment}/app-config.json"
  content = jsonencode({
    environment   = var.environment
    database_host = aws_db_instance.main.endpoint
    region        = var.aws_region
    features = {
      new_ui      = var.environment != "production"
      debug_mode  = var.environment == "development"
    }
  })
  content_type = "application/json"

  # ✅ Encrypt sensitive config
  server_side_encryption = "aws:kms"
  kms_key_id             = aws_kms_key.s3.arn
}

# ─── Upload multiple files ────────────────────────────────

locals {
  static_files = fileset("${path.module}/static", "**/*")
}

resource "aws_s3_object" "static" {
  for_each = local.static_files

  bucket = aws_s3_bucket.website.id
  key    = each.value
  source = "${path.module}/static/${each.value}"
  etag   = filemd5("${path.module}/static/${each.value}")

  content_type = lookup({
    "html" = "text/html"
    "css"  = "text/css"
    "js"   = "application/javascript"
    "png"  = "image/png"
    "jpg"  = "image/jpeg"
    "ico"  = "image/x-icon"
  }, split(".", each.value)[length(split(".", each.value)) - 1], "application/octet-stream")
}
```

---

## Step 400: Terraform State S3 Backend (Secure Configuration)

### S3 State Backend Setup

```hcl
# backend.tf (ใน root module)

terraform {
  backend "s3" {
    # ─── S3 Configuration ─────────────────────────────────
    bucket         = "terraform-state-123456789012"
    key            = "production/terraform.tfstate"
    region         = "ap-southeast-1"
    encrypt        = true                                    # ✅ Encrypt state
    kms_key_id     = "arn:aws:kms:ap-southeast-1:123456789012:key/abc-123"

    # ─── DynamoDB Locking ─────────────────────────────────
    dynamodb_table = "terraform-state-lock"

    # ─── Access ──────────────────────────────────────────
    role_arn       = "arn:aws:iam::123456789012:role/TerraformRole"
    profile        = "terraform"  # AWS CLI profile
  }
}
```

### Setup สำหรับ State Backend

```hcl
# setup_state_backend.tf - สร้าง infrastructure สำหรับ state storage

# ─── S3 Bucket สำหรับ State ──────────────────────────────

resource "aws_s3_bucket" "terraform_state" {
  bucket = "terraform-state-${data.aws_caller_identity.current.account_id}-${data.aws_region.current.name}"

  lifecycle {
    prevent_destroy = true  # ✅ ป้องกันการลบโดยไม่ตั้งใจ
  }

  tags = {
    Name    = "terraform-state"
    Purpose = "terraform-remote-state"
  }
}

# Versioning (จำเป็นมาก!)
resource "aws_s3_bucket_versioning" "terraform_state" {
  bucket = aws_s3_bucket.terraform_state.id
  versioning_configuration { status = "Enabled" }
}

# Encryption
resource "aws_s3_bucket_server_side_encryption_configuration" "terraform_state" {
  bucket = aws_s3_bucket.terraform_state.id
  rule {
    apply_server_side_encryption_by_default {
      sse_algorithm     = "aws:kms"
      kms_master_key_id = aws_kms_key.terraform_state.arn
    }
    bucket_key_enabled = true
  }
}

# Block public access
resource "aws_s3_bucket_public_access_block" "terraform_state" {
  bucket                  = aws_s3_bucket.terraform_state.id
  block_public_acls       = true
  ignore_public_acls      = true
  block_public_policy     = true
  restrict_public_buckets = true
}

# Lifecycle - เก็บ state versions 90 วัน
resource "aws_s3_bucket_lifecycle_configuration" "terraform_state" {
  bucket = aws_s3_bucket.terraform_state.id
  depends_on = [aws_s3_bucket_versioning.terraform_state]

  rule {
    id     = "cleanup-old-state-versions"
    status = "Enabled"
    filter {}

    noncurrent_version_expiration {
      noncurrent_days = 90
    }

    abort_incomplete_multipart_upload {
      days_after_initiation = 7
    }
  }
}

# ─── DynamoDB สำหรับ State Locking ──────────────────────

resource "aws_dynamodb_table" "terraform_state_lock" {
  name         = "terraform-state-lock"
  billing_mode = "PAY_PER_REQUEST"
  hash_key     = "LockID"

  attribute {
    name = "LockID"
    type = "S"
  }

  point_in_time_recovery {
    enabled = true  # ✅ Enable PITR
  }

  server_side_encryption {
    enabled = true  # ✅ Encrypt
  }

  lifecycle {
    prevent_destroy = true
  }

  tags = {
    Name    = "terraform-state-lock"
    Purpose = "terraform-state-locking"
  }
}

# ─── KMS Key ─────────────────────────────────────────────

resource "aws_kms_key" "terraform_state" {
  description             = "KMS key สำหรับ Terraform state encryption"
  deletion_window_in_days = 30
  enable_key_rotation     = true

  lifecycle {
    prevent_destroy = true
  }

  tags = { Name = "terraform-state-kms" }
}

resource "aws_kms_alias" "terraform_state" {
  name          = "alias/terraform-state"
  target_key_id = aws_kms_key.terraform_state.key_id
}

# ─── Outputs ─────────────────────────────────────────────

output "state_bucket_name" {
  description = "S3 bucket สำหรับ Terraform state"
  value       = aws_s3_bucket.terraform_state.id
}

output "state_lock_table_name" {
  description = "DynamoDB table สำหรับ state locking"
  value       = aws_dynamodb_table.terraform_state_lock.name
}

output "state_kms_key_arn" {
  description = "KMS key ARN สำหรับ state encryption"
  value       = aws_kms_key.terraform_state.arn
}

output "backend_config" {
  description = "Backend configuration สำหรับ copy ไปใน backend.tf"
  value = <<-EOF
    terraform {
      backend "s3" {
        bucket         = "${aws_s3_bucket.terraform_state.id}"
        key            = "<workspace>/terraform.tfstate"
        region         = "${data.aws_region.current.name}"
        encrypt        = true
        kms_key_id     = "${aws_kms_key.terraform_state.arn}"
        dynamodb_table = "${aws_dynamodb_table.terraform_state_lock.name}"
      }
    }
  EOF
}
```

---

## สรุป: S3 Best Practices

### Security Checklist

```
✅ S3 Security Checklist:

Encryption:
  [ ] Server-side encryption เปิดอยู่เสมอ
  [ ] ใช้ SSE-KMS สำหรับ sensitive data
  [ ] bucket_key_enabled = true เพื่อลด costs

Public Access:
  [ ] block_public_acls = true
  [ ] ignore_public_acls = true
  [ ] block_public_policy = true
  [ ] restrict_public_buckets = true

Bucket Policies:
  [ ] Force SSL/TLS เสมอ (DenyNonHTTPS)
  [ ] Least-privilege access
  [ ] ไม่มี wildcard (*) ใน Principal ยกเว้นจำเป็น

Versioning:
  [ ] เปิด versioning สำหรับ state, config, critical data
  [ ] Lifecycle rules สำหรับลบ old versions

Logging:
  [ ] S3 Access Logging เปิดอยู่
  [ ] CloudTrail S3 events เปิดอยู่

Other:
  [ ] Object Ownership = BucketOwnerEnforced
  [ ] lifecycle.prevent_destroy สำหรับ important buckets
  [ ] Tags ครบถ้วน
```

### Cost Optimization

```hcl
# Cost Optimization: ใช้ Intelligent Tiering

resource "aws_s3_bucket_intelligent_tiering_configuration" "main" {
  bucket = aws_s3_bucket.data.id
  name   = "all-objects"

  tiering {
    access_tier = "DEEP_ARCHIVE_ACCESS"
    days        = 180
  }

  tiering {
    access_tier = "ARCHIVE_ACCESS"
    days        = 90
  }
}
```

---

*จบ Part 040: AWS S3 Buckets*

*จบ Part 031-040 ของ HCL/Terraform Course*

---

## รายการ Files ที่สร้าง

| File | หัวข้อ | Steps |
|------|--------|-------|
| part-031.md | Terraform Graph & Dependencies | 301-310 |
| part-032.md | Terraform Refresh & Reconciliation | 311-320 |
| part-033.md | Terraform Console & Expressions | 321-330 |
| part-034.md | Terraform Lock File | 331-340 |
| part-035.md | Configuration Best Practices | 341-350 |
| part-036.md | AWS Provider Setup & Authentication | 351-360 |
| part-037.md | AWS EC2 Instances | 361-370 |
| part-038.md | AWS VPC & Networking | 371-380 |
| part-039.md | AWS Security Groups & NACLs | 381-390 |
| part-040.md | AWS S3 Buckets | 391-400 |
