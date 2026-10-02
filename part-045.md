# Part 045: AWS CloudFront CDN
## การใช้งาน Content Delivery Network ด้วย Terraform (Steps 441-450)

---

## บทนำ (Introduction)

AWS CloudFront เป็น CDN (Content Delivery Network) ที่กระจาย content ไปยัง Edge Locations ทั่วโลก
ช่วยลด latency, เพิ่ม performance, และป้องกัน DDoS attacks

**หัวข้อที่จะเรียนรู้:**
- CloudFront Distribution พื้นฐาน
- Origins: S3, ALB, Custom
- Origin Access Control (OAC) - วิธีใหม่
- Cache Behaviors และ Cache Policies
- SSL/TLS Certificates ด้วย ACM
- Custom Domains (Aliases)
- WAF Integration
- CloudFront Functions
- Logging และ Invalidation
- Static Website Hosting

---

## Step 441: CloudFront Distribution พื้นฐาน

```hcl
# ✅ S3 Bucket สำหรับ Static Website
resource "aws_s3_bucket" "website" {
  bucket = "${var.project_name}-website"

  tags = {
    Name        = "${var.project_name}-website"
    Environment = var.environment
    ManagedBy   = "terraform"
  }
}

resource "aws_s3_bucket_public_access_block" "website" {
  bucket = aws_s3_bucket.website.id

  # ✅ Block ทั้งหมด - CloudFront จะเข้าถึงด้วย OAC
  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}

resource "aws_s3_bucket_versioning" "website" {
  bucket = aws_s3_bucket.website.id
  versioning_configuration {
    status = "Enabled"
  }
}

resource "aws_s3_bucket_server_side_encryption_configuration" "website" {
  bucket = aws_s3_bucket.website.id
  rule {
    apply_server_side_encryption_by_default {
      sse_algorithm = "AES256"
    }
  }
}
```

---

## Step 442: Origin Access Control (OAC) - วิธีใหม่

```hcl
# ✅ Origin Access Control (OAC) - แทนที่ OAI เก่า
resource "aws_cloudfront_origin_access_control" "website" {
  name                              = "${var.project_name}-oac"
  description                       = "OAC for ${var.project_name} website"
  origin_access_control_origin_type = "s3"
  signing_behavior                  = "always"
  signing_protocol                  = "sigv4"
}

# ✅ S3 Bucket Policy สำหรับ OAC
data "aws_iam_policy_document" "website_bucket_policy" {
  statement {
    sid    = "AllowCloudFrontOAC"
    effect = "Allow"

    principals {
      type        = "Service"
      identifiers = ["cloudfront.amazonaws.com"]
    }

    actions = ["s3:GetObject"]

    resources = ["${aws_s3_bucket.website.arn}/*"]

    condition {
      test     = "StringEquals"
      variable = "AWS:SourceArn"
      values   = [aws_cloudfront_distribution.website.arn]
    }
  }
}

resource "aws_s3_bucket_policy" "website" {
  bucket = aws_s3_bucket.website.id
  policy = data.aws_iam_policy_document.website_bucket_policy.json

  depends_on = [
    aws_s3_bucket_public_access_block.website,
    aws_cloudfront_distribution.website,
  ]
}
```

### Origin Access Identity (OAI) - วิธีเก่า (Legacy)

```hcl
# ⚠️ OAI - วิธีเก่า (ยังใช้งานได้แต่ AWS แนะนำให้ใช้ OAC)
resource "aws_cloudfront_origin_access_identity" "website" {
  comment = "OAI for ${var.project_name} website"
}

# OAI Bucket Policy
data "aws_iam_policy_document" "website_oai_policy" {
  statement {
    actions   = ["s3:GetObject"]
    resources = ["${aws_s3_bucket.website.arn}/*"]

    principals {
      type        = "AWS"
      identifiers = [aws_cloudfront_origin_access_identity.website.iam_arn]
    }
  }
}
```

---

## Step 443: CloudFront Distribution หลัก

```hcl
# ✅ ACM Certificate (ต้องสร้างใน us-east-1 สำหรับ CloudFront)
provider "aws" {
  alias  = "us_east_1"
  region = "us-east-1"
}

resource "aws_acm_certificate" "website" {
  provider          = aws.us_east_1
  domain_name       = var.domain_name
  validation_method = "DNS"

  subject_alternative_names = [
    "www.${var.domain_name}",
    "*.${var.domain_name}",
  ]

  lifecycle {
    create_before_destroy = true
  }

  tags = {
    Name      = "${var.project_name}-cert"
    ManagedBy = "terraform"
  }
}

# ✅ Complete CloudFront Distribution
resource "aws_cloudfront_distribution" "website" {
  enabled             = true
  is_ipv6_enabled     = true
  comment             = "${var.project_name} website distribution"
  default_root_object = "index.html"
  price_class         = "PriceClass_100"  # US, Canada, Europe เท่านั้น (ประหยัดกว่า)
  # PriceClass_All = ทุก edge location
  # PriceClass_200 = US, Canada, Europe, Asia, Africa
  # PriceClass_100 = US, Canada, Europe

  # ✅ Custom domain
  aliases = [
    var.domain_name,
    "www.${var.domain_name}",
  ]

  # ✅ S3 Origin ด้วย OAC
  origin {
    domain_name              = aws_s3_bucket.website.bucket_regional_domain_name
    origin_id                = "S3-${aws_s3_bucket.website.bucket}"
    origin_access_control_id = aws_cloudfront_origin_access_control.website.id

    # S3 ไม่ต้องใช้ custom_origin_config
  }

  # ✅ Default Cache Behavior
  default_cache_behavior {
    allowed_methods  = ["GET", "HEAD", "OPTIONS"]
    cached_methods   = ["GET", "HEAD"]
    target_origin_id = "S3-${aws_s3_bucket.website.bucket}"

    # ✅ ใช้ Cache Policy
    cache_policy_id          = aws_cloudfront_cache_policy.static_assets.id
    origin_request_policy_id = data.aws_cloudfront_origin_request_policy.cors_s3.id

    # ✅ HTTPS เท่านั้น
    viewer_protocol_policy = "redirect-to-https"

    compress = true  # ✅ Gzip/Brotli compression

    # ✅ Response headers policy
    response_headers_policy_id = aws_cloudfront_response_headers_policy.security.id
  }

  # ✅ Additional Cache Behavior สำหรับ API calls
  ordered_cache_behavior {
    path_pattern     = "/api/*"
    allowed_methods  = ["DELETE", "GET", "HEAD", "OPTIONS", "PATCH", "POST", "PUT"]
    cached_methods   = ["GET", "HEAD"]
    target_origin_id = "ALB-${var.project_name}"

    cache_policy_id          = data.aws_cloudfront_cache_policy.caching_disabled.id
    origin_request_policy_id = data.aws_cloudfront_origin_request_policy.all_viewer.id

    viewer_protocol_policy = "https-only"
    compress               = true
  }

  # ✅ Custom error pages
  custom_error_response {
    error_caching_min_ttl = 300
    error_code            = 403
    response_code         = 200
    response_page_path    = "/index.html"  # SPA routing
  }

  custom_error_response {
    error_caching_min_ttl = 300
    error_code            = 404
    response_code         = 200
    response_page_path    = "/index.html"  # SPA routing
  }

  # ✅ SSL Certificate
  viewer_certificate {
    acm_certificate_arn      = aws_acm_certificate.website.arn
    ssl_support_method       = "sni-only"
    minimum_protocol_version = "TLSv1.2_2021"  # ✅ Modern TLS only
  }

  # ✅ Geo Restriction
  restrictions {
    geo_restriction {
      restriction_type = "none"
      # หรือ whitelist/blacklist
      # restriction_type = "whitelist"
      # locations        = ["TH", "SG", "JP"]
    }
  }

  # ✅ Access logging
  logging_config {
    include_cookies = false
    bucket          = "${aws_s3_bucket.cloudfront_logs.bucket_domain_name}"
    prefix          = "cloudfront-logs/${var.project_name}/"
  }

  # ✅ WAF Web ACL
  web_acl_id = aws_wafv2_web_acl.cloudfront.arn

  tags = {
    Name        = "${var.project_name}-distribution"
    Environment = var.environment
    ManagedBy   = "terraform"
  }

  depends_on = [
    aws_acm_certificate_validation.website,
  ]
}
```

---

## Step 444: Multiple Origins

### ALB Origin

```hcl
# ✅ CloudFront Distribution พร้อม Multiple Origins
resource "aws_cloudfront_distribution" "multi_origin" {
  enabled         = true
  is_ipv6_enabled = true
  comment         = "Multi-origin distribution"
  price_class     = "PriceClass_All"

  aliases = [var.domain_name]

  # Origin 1: S3 สำหรับ static assets
  origin {
    domain_name              = aws_s3_bucket.assets.bucket_regional_domain_name
    origin_id                = "S3-assets"
    origin_access_control_id = aws_cloudfront_origin_access_control.assets.id
  }

  # Origin 2: ALB สำหรับ dynamic content
  origin {
    domain_name = aws_lb.app.dns_name
    origin_id   = "ALB-app"

    custom_origin_config {
      http_port              = 80
      https_port             = 443
      origin_protocol_policy = "https-only"  # ✅ HTTPS เท่านั้น
      origin_ssl_protocols   = ["TLSv1.2"]

      # ✅ ตั้ง origin timeout
      origin_read_timeout      = 60
      origin_keepalive_timeout = 60
    }

    # ✅ Custom headers ให้ backend รู้ว่ามาจาก CloudFront
    custom_header {
      name  = "X-CloudFront-Secret"
      value = random_password.cloudfront_secret.result  # ✅ ตรวจสอบใน ALB
    }
  }

  # Origin 3: Custom API (external)
  origin {
    domain_name = "api.external-service.com"
    origin_id   = "External-API"

    custom_origin_config {
      http_port              = 443
      https_port             = 443
      origin_protocol_policy = "https-only"
      origin_ssl_protocols   = ["TLSv1.2"]
    }
  }

  # Default behavior: Static assets จาก S3
  default_cache_behavior {
    allowed_methods        = ["GET", "HEAD"]
    cached_methods         = ["GET", "HEAD"]
    target_origin_id       = "S3-assets"
    viewer_protocol_policy = "redirect-to-https"
    compress               = true

    cache_policy_id = aws_cloudfront_cache_policy.static_assets.id
  }

  # /api/* -> ALB
  ordered_cache_behavior {
    path_pattern           = "/api/*"
    allowed_methods        = ["DELETE", "GET", "HEAD", "OPTIONS", "PATCH", "POST", "PUT"]
    cached_methods         = ["GET", "HEAD"]
    target_origin_id       = "ALB-app"
    viewer_protocol_policy = "https-only"
    compress               = true

    cache_policy_id          = data.aws_cloudfront_cache_policy.caching_disabled.id
    origin_request_policy_id = data.aws_cloudfront_origin_request_policy.all_viewer.id
  }

  # /static/* -> S3 assets
  ordered_cache_behavior {
    path_pattern           = "/static/*"
    allowed_methods        = ["GET", "HEAD"]
    cached_methods         = ["GET", "HEAD"]
    target_origin_id       = "S3-assets"
    viewer_protocol_policy = "redirect-to-https"
    compress               = true

    cache_policy_id = aws_cloudfront_cache_policy.static_assets.id
  }

  viewer_certificate {
    acm_certificate_arn      = aws_acm_certificate.website.arn
    ssl_support_method       = "sni-only"
    minimum_protocol_version = "TLSv1.2_2021"
  }

  restrictions {
    geo_restriction {
      restriction_type = "none"
    }
  }

  tags = {
    Name      = "${var.project_name}-multi-origin"
    ManagedBy = "terraform"
  }
}
```

---

## Step 445: Cache Policies

### aws_cloudfront_cache_policy

```hcl
# ✅ Cache Policy สำหรับ Static Assets (Long TTL)
resource "aws_cloudfront_cache_policy" "static_assets" {
  name        = "${var.project_name}-static-assets"
  comment     = "Cache policy for static assets with long TTL"
  min_ttl     = 3600    # 1 hour minimum
  default_ttl = 86400   # 1 day default
  max_ttl     = 2592000 # 30 days maximum

  parameters_in_cache_key_and_forwarded_to_origin {
    cookies_config {
      cookie_behavior = "none"  # ไม่ cache ตาม cookies
    }

    headers_config {
      header_behavior = "none"  # ไม่ cache ตาม headers
    }

    query_strings_config {
      query_string_behavior = "whitelist"
      query_strings {
        items = ["version", "v"]  # Cache version-busting params
      }
    }

    enable_accept_encoding_brotli = true  # ✅ Brotli
    enable_accept_encoding_gzip   = true  # ✅ Gzip
  }
}

# ✅ Cache Policy สำหรับ API (No Cache)
resource "aws_cloudfront_cache_policy" "no_cache" {
  name        = "${var.project_name}-no-cache"
  comment     = "No caching policy for dynamic content"
  min_ttl     = 0
  default_ttl = 0
  max_ttl     = 0

  parameters_in_cache_key_and_forwarded_to_origin {
    cookies_config {
      cookie_behavior = "none"
    }

    headers_config {
      header_behavior = "none"
    }

    query_strings_config {
      query_string_behavior = "none"
    }
  }
}

# ✅ AWS Managed Cache Policies
data "aws_cloudfront_cache_policy" "caching_disabled" {
  name = "Managed-CachingDisabled"
}

data "aws_cloudfront_cache_policy" "caching_optimized" {
  name = "Managed-CachingOptimized"
}
```

### Origin Request Policies

```hcl
# ✅ AWS Managed Origin Request Policies
data "aws_cloudfront_origin_request_policy" "cors_s3" {
  name = "Managed-CORS-S3Origin"
}

data "aws_cloudfront_origin_request_policy" "all_viewer" {
  name = "Managed-AllViewer"
}

data "aws_cloudfront_origin_request_policy" "cors_custom" {
  name = "Managed-CORS-CustomOrigin"
}

# ✅ Custom Origin Request Policy
resource "aws_cloudfront_origin_request_policy" "api" {
  name    = "${var.project_name}-api-origin-request"
  comment = "Forward specific headers and cookies to API origin"

  cookies_config {
    cookie_behavior = "whitelist"
    cookies {
      items = ["session", "auth"]
    }
  }

  headers_config {
    header_behavior = "whitelist"
    headers {
      items = [
        "Authorization",
        "Accept",
        "Content-Type",
        "X-Request-ID",
      ]
    }
  }

  query_strings_config {
    query_string_behavior = "all"  # Forward all query strings
  }
}
```

---

## Step 446: Response Headers Policy

```hcl
# ✅ Security Headers Policy
resource "aws_cloudfront_response_headers_policy" "security" {
  name    = "${var.project_name}-security-headers"
  comment = "Security headers for ${var.project_name}"

  # ✅ CORS Headers
  cors_config {
    access_control_allow_credentials = false

    access_control_allow_headers {
      items = ["Authorization", "Content-Type", "X-Requested-With"]
    }

    access_control_allow_methods {
      items = ["GET", "HEAD", "OPTIONS"]
    }

    access_control_allow_origins {
      items = ["https://${var.domain_name}"]
    }

    origin_override = false
  }

  # ✅ Security Headers
  security_headers_config {
    # Strict Transport Security
    strict_transport_security {
      access_control_max_age_sec = 31536000  # 1 year
      include_subdomains         = true
      preload                    = true
      override                   = true
    }

    # Content Security Policy
    content_security_policy {
      content_security_policy = "default-src 'self'; img-src 'self' data: https:; script-src 'self'; style-src 'self' 'unsafe-inline';"
      override                = true
    }

    # X-Content-Type-Options
    content_type_options {
      override = true
    }

    # X-Frame-Options
    frame_options {
      frame_option = "DENY"
      override     = true
    }

    # X-XSS-Protection
    xss_protection {
      mode_block = true
      protection = true
      override   = true
    }

    # Referrer-Policy
    referrer_policy {
      referrer_policy = "strict-origin-when-cross-origin"
      override        = true
    }
  }

  # ✅ Custom Headers
  custom_headers_config {
    items {
      header   = "X-Powered-By"
      value    = ""  # ✅ ลบ X-Powered-By
      override = true
    }

    items {
      header   = "Permissions-Policy"
      value    = "camera=(), microphone=(), geolocation=()"
      override = true
    }
  }
}
```

---

## Step 447: CloudFront Functions

```hcl
# ✅ CloudFront Function สำหรับ URL Rewriting (Viewer Request)
resource "aws_cloudfront_function" "url_rewrite" {
  name    = "${var.project_name}-url-rewrite"
  runtime = "cloudfront-js-2.0"
  comment = "URL rewriting for SPA routing"
  publish = true

  code = <<-EOT
    function handler(event) {
      var request = event.request;
      var uri = request.uri;

      // Check for file extension
      if (!uri.includes('.')) {
        // No file extension - likely a SPA route
        request.uri = '/index.html';
      }

      return request;
    }
  EOT
}

# ✅ CloudFront Function สำหรับ Security Headers (Viewer Response)
resource "aws_cloudfront_function" "security_headers" {
  name    = "${var.project_name}-security-headers"
  runtime = "cloudfront-js-2.0"
  comment = "Add security headers to responses"
  publish = true

  code = <<-EOT
    function handler(event) {
      var response = event.response;
      var headers = response.headers;

      // Security headers
      headers['strict-transport-security'] = {
        value: 'max-age=63072000; includeSubdomains; preload'
      };
      headers['x-content-type-options'] = {value: 'nosniff'};
      headers['x-frame-options'] = {value: 'DENY'};
      headers['x-xss-protection'] = {value: '1; mode=block'};
      headers['referrer-policy'] = {value: 'strict-origin-when-cross-origin'};

      return response;
    }
  EOT
}

# ✅ ใช้ CloudFront Functions ใน distribution
resource "aws_cloudfront_distribution" "with_functions" {
  # ...

  default_cache_behavior {
    # ...
    function_association {
      event_type   = "viewer-request"
      function_arn = aws_cloudfront_function.url_rewrite.arn
    }

    function_association {
      event_type   = "viewer-response"
      function_arn = aws_cloudfront_function.security_headers.arn
    }
  }

  # ...
}
```

---

## Step 448: Logging Configuration

```hcl
# ✅ S3 Bucket สำหรับ CloudFront Access Logs
resource "aws_s3_bucket" "cloudfront_logs" {
  bucket = "${var.project_name}-cloudfront-logs"

  tags = {
    Name      = "${var.project_name}-cloudfront-logs"
    ManagedBy = "terraform"
  }
}

resource "aws_s3_bucket_ownership_controls" "cloudfront_logs" {
  bucket = aws_s3_bucket.cloudfront_logs.id

  rule {
    object_ownership = "BucketOwnerPreferred"
  }
}

# ✅ CloudFront ต้องการ ACL สำหรับ logging
resource "aws_s3_bucket_acl" "cloudfront_logs" {
  depends_on = [aws_s3_bucket_ownership_controls.cloudfront_logs]

  bucket = aws_s3_bucket.cloudfront_logs.id
  acl    = "log-delivery-write"
}

resource "aws_s3_bucket_lifecycle_configuration" "cloudfront_logs" {
  bucket = aws_s3_bucket.cloudfront_logs.id

  rule {
    id     = "expire-old-logs"
    status = "Enabled"

    expiration {
      days = 90  # เก็บ logs 90 วัน
    }

    noncurrent_version_expiration {
      noncurrent_days = 30
    }
  }
}

resource "aws_s3_bucket_server_side_encryption_configuration" "cloudfront_logs" {
  bucket = aws_s3_bucket.cloudfront_logs.id
  rule {
    apply_server_side_encryption_by_default {
      sse_algorithm = "AES256"
    }
  }
}
```

---

## Step 449: Cache Invalidation

```hcl
# ✅ Invalidation ด้วย null_resource (manual trigger)
resource "null_resource" "cloudfront_invalidation" {
  triggers = {
    # Trigger เมื่อ distribution หรือ source files เปลี่ยน
    distribution_id = aws_cloudfront_distribution.website.id
    # สร้าง hash จาก file contents
    source_hash = sha256(join("", [
      for f in fileset("${path.module}/dist", "**") :
      filesha256("${path.module}/dist/${f}")
    ]))
  }

  provisioner "local-exec" {
    command = <<-EOT
      aws cloudfront create-invalidation \
        --distribution-id ${aws_cloudfront_distribution.website.id} \
        --paths "/*"
    EOT
  }

  depends_on = [aws_cloudfront_distribution.website]
}

# ✅ Deploy S3 files พร้อม invalidation
resource "null_resource" "deploy_website" {
  triggers = {
    # Deploy เมื่อ source เปลี่ยน
    content_hash = md5(join("", [
      for f in fileset("${path.module}/build", "**") :
      filemd5("${path.module}/build/${f}")
    ]))
  }

  provisioner "local-exec" {
    command = <<-EOT
      # Upload files ไป S3
      aws s3 sync ${path.module}/build/ s3://${aws_s3_bucket.website.bucket}/ \
        --delete \
        --cache-control "max-age=31536000" \
        --exclude "index.html"

      # index.html ไม่ cache
      aws s3 cp ${path.module}/build/index.html s3://${aws_s3_bucket.website.bucket}/index.html \
        --cache-control "no-cache, no-store, must-revalidate"

      # Invalidate CloudFront cache
      aws cloudfront create-invalidation \
        --distribution-id ${aws_cloudfront_distribution.website.id} \
        --paths "/*"
    EOT
  }

  depends_on = [aws_cloudfront_distribution.website]
}
```

---

## Step 450: Complete Static Website Hosting Example

```hcl
# ✅ Complete Static Website with CloudFront + S3 + Route53 + ACM

# 1. Providers
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

provider "aws" {
  region = "ap-southeast-1"
}

provider "aws" {
  alias  = "us_east_1"
  region = "us-east-1"
}

# 2. Variables
variable "domain_name" {
  description = "Main domain name"
  type        = string
  default     = "example.com"
}

variable "project_name" {
  description = "Project name"
  type        = string
  default     = "myapp"
}

# 3. Route53 Hosted Zone
data "aws_route53_zone" "main" {
  name         = var.domain_name
  private_zone = false
}

# 4. ACM Certificate (us-east-1 สำหรับ CloudFront)
resource "aws_acm_certificate" "main" {
  provider          = aws.us_east_1
  domain_name       = var.domain_name
  validation_method = "DNS"

  subject_alternative_names = ["www.${var.domain_name}"]

  lifecycle {
    create_before_destroy = true
  }

  tags = {
    Name      = "${var.project_name}-ssl-cert"
    ManagedBy = "terraform"
  }
}

# 5. DNS Validation Records
resource "aws_route53_record" "cert_validation" {
  for_each = {
    for dvo in aws_acm_certificate.main.domain_validation_options :
    dvo.domain_name => {
      name   = dvo.resource_record_name
      record = dvo.resource_record_value
      type   = dvo.resource_record_type
    }
  }

  allow_overwrite = true
  name            = each.value.name
  records         = [each.value.record]
  ttl             = 60
  type            = each.value.type
  zone_id         = data.aws_route53_zone.main.zone_id
}

resource "aws_acm_certificate_validation" "main" {
  provider                = aws.us_east_1
  certificate_arn         = aws_acm_certificate.main.arn
  validation_record_fqdns = [for record in aws_route53_record.cert_validation : record.fqdn]
}

# 6. S3 Website Bucket
resource "aws_s3_bucket" "website_main" {
  bucket = "${var.project_name}-website-${data.aws_caller_identity.current.account_id}"

  tags = {
    Name        = "${var.project_name}-website"
    Environment = "production"
    ManagedBy   = "terraform"
  }
}

resource "aws_s3_bucket_public_access_block" "website_main" {
  bucket                  = aws_s3_bucket.website_main.id
  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}

resource "aws_s3_bucket_versioning" "website_main" {
  bucket = aws_s3_bucket.website_main.id
  versioning_configuration {
    status = "Enabled"
  }
}

resource "aws_s3_bucket_server_side_encryption_configuration" "website_main" {
  bucket = aws_s3_bucket.website_main.id
  rule {
    apply_server_side_encryption_by_default {
      sse_algorithm = "AES256"
    }
  }
}

# 7. CloudFront OAC
resource "aws_cloudfront_origin_access_control" "main" {
  name                              = "${var.project_name}-oac"
  description                       = "OAC for ${var.project_name}"
  origin_access_control_origin_type = "s3"
  signing_behavior                  = "always"
  signing_protocol                  = "sigv4"
}

# 8. CloudFront Distribution
resource "aws_cloudfront_distribution" "main" {
  enabled             = true
  is_ipv6_enabled     = true
  default_root_object = "index.html"
  price_class         = "PriceClass_All"
  comment             = "${var.project_name} main distribution"

  aliases = [
    var.domain_name,
    "www.${var.domain_name}",
  ]

  origin {
    domain_name              = aws_s3_bucket.website_main.bucket_regional_domain_name
    origin_id                = "primary-s3"
    origin_access_control_id = aws_cloudfront_origin_access_control.main.id
  }

  default_cache_behavior {
    allowed_methods        = ["GET", "HEAD", "OPTIONS"]
    cached_methods         = ["GET", "HEAD"]
    target_origin_id       = "primary-s3"
    viewer_protocol_policy = "redirect-to-https"
    compress               = true

    cache_policy_id = data.aws_cloudfront_cache_policy.caching_optimized.id

    function_association {
      event_type   = "viewer-request"
      function_arn = aws_cloudfront_function.url_rewrite.arn
    }
  }

  # ✅ SPA: redirect 404/403 ไป index.html
  custom_error_response {
    error_caching_min_ttl = 10
    error_code            = 403
    response_code         = 200
    response_page_path    = "/index.html"
  }

  custom_error_response {
    error_caching_min_ttl = 10
    error_code            = 404
    response_code         = 200
    response_page_path    = "/index.html"
  }

  viewer_certificate {
    acm_certificate_arn      = aws_acm_certificate_validation.main.certificate_arn
    ssl_support_method       = "sni-only"
    minimum_protocol_version = "TLSv1.2_2021"
  }

  restrictions {
    geo_restriction {
      restriction_type = "none"
    }
  }

  logging_config {
    include_cookies = false
    bucket          = aws_s3_bucket.cloudfront_logs.bucket_domain_name
    prefix          = "website/"
  }

  tags = {
    Name        = "${var.project_name}-distribution"
    Environment = "production"
    ManagedBy   = "terraform"
  }

  depends_on = [aws_acm_certificate_validation.main]
}

# 9. S3 Bucket Policy
data "aws_iam_policy_document" "website_policy" {
  statement {
    sid    = "AllowCloudFront"
    effect = "Allow"

    principals {
      type        = "Service"
      identifiers = ["cloudfront.amazonaws.com"]
    }

    actions   = ["s3:GetObject"]
    resources = ["${aws_s3_bucket.website_main.arn}/*"]

    condition {
      test     = "StringEquals"
      variable = "AWS:SourceArn"
      values   = [aws_cloudfront_distribution.main.arn]
    }
  }
}

resource "aws_s3_bucket_policy" "website_main" {
  bucket = aws_s3_bucket.website_main.id
  policy = data.aws_iam_policy_document.website_policy.json

  depends_on = [aws_s3_bucket_public_access_block.website_main]
}

# 10. Route53 Records ชี้ไป CloudFront
resource "aws_route53_record" "website_apex" {
  zone_id = data.aws_route53_zone.main.zone_id
  name    = var.domain_name
  type    = "A"

  alias {
    name                   = aws_cloudfront_distribution.main.domain_name
    zone_id                = aws_cloudfront_distribution.main.hosted_zone_id
    evaluate_target_health = false
  }
}

resource "aws_route53_record" "website_www" {
  zone_id = data.aws_route53_zone.main.zone_id
  name    = "www.${var.domain_name}"
  type    = "A"

  alias {
    name                   = aws_cloudfront_distribution.main.domain_name
    zone_id                = aws_cloudfront_distribution.main.hosted_zone_id
    evaluate_target_health = false
  }
}

# 11. Outputs
output "cloudfront_distribution_id" {
  description = "CloudFront distribution ID"
  value       = aws_cloudfront_distribution.main.id
}

output "cloudfront_domain_name" {
  description = "CloudFront domain name"
  value       = aws_cloudfront_distribution.main.domain_name
}

output "website_url" {
  description = "Website URL"
  value       = "https://${var.domain_name}"
}

output "s3_bucket_name" {
  description = "S3 bucket name"
  value       = aws_s3_bucket.website_main.bucket
}
```

---

## CloudFront Best Practices สรุป

### ✅ Security

1. **OAC แทน OAI**: ใช้ Origin Access Control (ใหม่กว่า)
2. **HTTPS Only**: บังคับ `viewer_protocol_policy = "redirect-to-https"`
3. **TLS 1.2+**: ตั้ง `minimum_protocol_version = "TLSv1.2_2021"`
4. **Security Headers**: ใช้ Response Headers Policy
5. **WAF**: เปิดใช้งาน WAF Web ACL
6. **Geo Restriction**: จำกัด locations ถ้าจำเป็น

### ✅ Performance

1. **Price Class**: เลือกตาม audience location
2. **Compression**: เปิด `compress = true`
3. **Cache Policy**: กำหนด TTL ที่เหมาะสม
4. **CloudFront Functions**: สำหรับ lightweight logic (ดีกว่า Lambda@Edge)
5. **Origin Shield**: เพิ่ม cache hit ratio (เพิ่มค่าใช้จ่าย)

### ✅ Cost Optimization

1. **Price Class**: `PriceClass_100` ถูกที่สุด
2. **Cache TTL**: TTL ยาว = request น้อยไป origin
3. **Logging**: เลือก log เฉพาะที่จำเป็น
4. **Origin Shield**: ลด origin requests

---

**Next Steps**: ไปต่อที่ Part 046 - AWS Route53 DNS
