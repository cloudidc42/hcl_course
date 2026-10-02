# Part 086: Encryption in Transit Issues
## ขั้นตอนที่ 851-860: ปัญหาการเข้ารหัสข้อมูลระหว่างการส่ง

---

## ขั้นตอนที่ 851: ภาพรวม Encryption in Transit

### ทำไม TLS จึงสำคัญ

```
Encryption in Transit ป้องกัน:
1. Man-in-the-Middle (MITM) attacks
2. Eavesdropping บน network
3. Data tampering ระหว่าง transmission
4. Credential theft ผ่าน plain text protocols

ตัวอย่าง attacks:
- HTTP over coffee shop WiFi = credentials ถูกดัก
- Weak TLS cipher = BEAST, POODLE attacks
- Self-signed certificates = No validation
- TLS 1.0/1.1 = Legacy vulnerabilities
```

### TLS Version Comparison

```
TLS Versions:
TLS 1.0 (1999): ❌ DEPRECATED - Multiple vulnerabilities
TLS 1.1 (2006): ❌ DEPRECATED - POODLE attack
TLS 1.2 (2008): ✅ Acceptable - Strong when configured correctly
TLS 1.3 (2018): ✅ RECOMMENDED - Best performance + security

AWS Security Policies for ALB:
- ELBSecurityPolicy-2016-08:         TLS 1.0+ ❌ Too permissive
- ELBSecurityPolicy-TLS-1-1-2017-01: TLS 1.1+ ❌ Still old
- ELBSecurityPolicy-TLS-1-2-2017-01: TLS 1.2+ ✅ Good
- ELBSecurityPolicy-TLS-1-2-Ext-2018-06: TLS 1.2+ ✅ More ciphers
- ELBSecurityPolicy-FS-1-2-2019-08: TLS 1.2+ Forward Secrecy ✅✅
- ELBSecurityPolicy-TLS13-1-2-2021-06: TLS 1.2 + 1.3 ✅✅ Recommended
```

---

## ขั้นตอนที่ 852: ALB HTTP vs HTTPS

### ❌ Vulnerable - ALB HTTP Only

```hcl
# ❌ VULNERABLE - HTTP listener without redirect
resource "aws_lb_listener" "http_only" {
  load_balancer_arn = aws_lb.main.arn
  port              = "80"
  protocol          = "HTTP"
  
  default_action {
    type             = "forward"
    target_group_arn = aws_lb_target_group.app.arn
    # ❌ ส่ง traffic โดยตรง โดยไม่ redirect ไป HTTPS
  }
}

# ❌ ALSO VULNERABLE - HTTPS with weak policy
resource "aws_lb_listener" "weak_https" {
  load_balancer_arn = aws_lb.main.arn
  port              = "443"
  protocol          = "HTTPS"
  
  ssl_policy      = "ELBSecurityPolicy-2016-08"  # ❌ TLS 1.0 allowed!
  certificate_arn = aws_acm_certificate.main.arn
  
  default_action {
    type             = "forward"
    target_group_arn = aws_lb_target_group.app.arn
  }
}
```

### ✅ Secure - HTTPS with Redirect and Strong Policy

```hcl
# ✅ SECURE - HTTP redirects to HTTPS
resource "aws_lb_listener" "http_redirect" {
  load_balancer_arn = aws_lb.main.arn
  port              = "80"
  protocol          = "HTTP"
  
  default_action {
    type = "redirect"   # ✅ Redirect to HTTPS
    
    redirect {
      port        = "443"
      protocol    = "HTTPS"
      status_code = "HTTP_301"  # ✅ Permanent redirect
    }
  }
}

# ✅ SECURE - HTTPS with strong TLS policy
resource "aws_lb_listener" "https" {
  load_balancer_arn = aws_lb.main.arn
  port              = "443"
  protocol          = "HTTPS"
  
  # ✅ TLS 1.2 + 1.3 with Forward Secrecy
  ssl_policy      = "ELBSecurityPolicy-TLS13-1-2-2021-06"
  certificate_arn = aws_acm_certificate.main.arn
  
  default_action {
    type             = "forward"
    target_group_arn = aws_lb_target_group.app.arn
  }
}

# ✅ ACM Certificate
resource "aws_acm_certificate" "main" {
  domain_name               = var.domain_name
  subject_alternative_names = ["*.${var.domain_name}"]  # ✅ Wildcard
  validation_method         = "DNS"
  
  lifecycle {
    create_before_destroy = true
  }
  
  tags = {
    Name = "main-cert"
  }
}

# ✅ Auto-validate with Route53
resource "aws_route53_record" "cert_validation" {
  for_each = {
    for dvo in aws_acm_certificate.main.domain_validation_options : dvo.domain_name => {
      name   = dvo.resource_record_name
      record = dvo.resource_record_value
      type   = dvo.resource_record_type
    }
  }
  
  zone_id = data.aws_route53_zone.main.zone_id
  name    = each.value.name
  type    = each.value.type
  records = [each.value.record]
  ttl     = 60
}

resource "aws_acm_certificate_validation" "main" {
  certificate_arn         = aws_acm_certificate.main.arn
  validation_record_fqdns = [for record in aws_route53_record.cert_validation : record.fqdn]
}

# ✅ WAF for ALB (bonus security)
resource "aws_wafv2_web_acl_association" "alb" {
  resource_arn = aws_lb.main.arn
  web_acl_arn  = aws_wafv2_web_acl.main.arn
}
```

---

## ขั้นตอนที่ 853: CloudFront TLS Configuration

### ❌ Vulnerable - CloudFront with Weak TLS

```hcl
# ❌ VULNERABLE - CloudFront allowing HTTP and old TLS
resource "aws_cloudfront_distribution" "vulnerable" {
  enabled = true
  
  origin {
    domain_name = aws_lb.main.dns_name
    origin_id   = "ALB"
    
    custom_origin_config {
      http_port                = 80
      https_port               = 443
      origin_protocol_policy   = "http-only"    # ❌ HTTP to origin!
      origin_ssl_protocols     = ["TLSv1", "TLSv1.1", "TLSv1.2"]  # ❌ Old versions
    }
  }
  
  default_cache_behavior {
    target_origin_id       = "ALB"
    viewer_protocol_policy = "allow-all"  # ❌ Allow HTTP!
    allowed_methods        = ["GET", "HEAD", "OPTIONS", "PUT", "POST", "PATCH", "DELETE"]
    cached_methods         = ["GET", "HEAD"]
    
    forwarded_values {
      query_string = true
      cookies { forward = "all" }
    }
  }
  
  viewer_certificate {
    cloudfront_default_certificate = true  # ❌ Default cert, not custom domain
    # ❌ ไม่มี minimum_protocol_version
    # ❌ ไม่มี ssl_support_method
  }
  
  restrictions {
    geo_restriction { restriction_type = "none" }
  }
}
```

### ✅ Secure - CloudFront with Strong TLS

```hcl
# ✅ SECURE - CloudFront with all security measures
resource "aws_cloudfront_distribution" "secure" {
  enabled             = true
  is_ipv6_enabled     = true
  comment             = "Secure CloudFront Distribution"
  default_root_object = "index.html"
  
  aliases = [var.domain_name, "www.${var.domain_name}"]
  
  # ✅ S3 Origin with OAI
  origin {
    domain_name = aws_s3_bucket.static.bucket_regional_domain_name
    origin_id   = "S3-${aws_s3_bucket.static.id}"
    
    s3_origin_config {
      origin_access_identity = aws_cloudfront_origin_access_identity.main.cloudfront_access_identity_path
    }
  }
  
  # ✅ ALB Origin with HTTPS only
  origin {
    domain_name = aws_lb.main.dns_name
    origin_id   = "ALB"
    
    custom_origin_config {
      http_port                = 80
      https_port               = 443
      origin_protocol_policy   = "https-only"   # ✅ HTTPS to origin only
      origin_ssl_protocols     = ["TLSv1.2"]    # ✅ TLS 1.2+ to origin
      origin_read_timeout      = 30
      origin_keepalive_timeout = 5
    }
  }
  
  # ✅ HTTPS only from viewers
  default_cache_behavior {
    target_origin_id       = "ALB"
    viewer_protocol_policy = "redirect-to-https"  # ✅ HTTPS only
    allowed_methods        = ["GET", "HEAD", "OPTIONS", "PUT", "POST", "PATCH", "DELETE"]
    cached_methods         = ["GET", "HEAD"]
    compress               = true  # ✅ Compress responses
    
    forwarded_values {
      query_string = true
      headers      = ["Host", "Authorization"]
      cookies { forward = "whitelist"
        whitelisted_names = ["session", "csrftoken"]
      }
    }
    
    # ✅ Security headers
    response_headers_policy_id = aws_cloudfront_response_headers_policy.security.id
  }
  
  # ✅ Static assets cache behavior
  ordered_cache_behavior {
    path_pattern           = "/static/*"
    target_origin_id       = "S3-${aws_s3_bucket.static.id}"
    viewer_protocol_policy = "redirect-to-https"  # ✅
    allowed_methods        = ["GET", "HEAD"]
    cached_methods         = ["GET", "HEAD"]
    
    forwarded_values {
      query_string = false
      cookies { forward = "none" }
    }
    
    min_ttl     = 86400    # 1 day
    default_ttl = 604800   # 7 days
    max_ttl     = 31536000 # 1 year
    compress    = true
  }
  
  # ✅ Strong TLS configuration
  viewer_certificate {
    acm_certificate_arn      = aws_acm_certificate.main.arn
    minimum_protocol_version = "TLSv1.2_2021"  # ✅ TLS 1.2 minimum
    ssl_support_method       = "sni-only"        # ✅ SNI (saves money)
  }
  
  # ✅ WAF integration
  web_acl_id = aws_wafv2_web_acl.main.arn
  
  # ✅ Logging
  logging_config {
    include_cookies = false
    bucket          = aws_s3_bucket.cloudfront_logs.bucket_domain_name
    prefix          = "cloudfront/"
  }
  
  restrictions {
    geo_restriction { restriction_type = "none" }
  }
  
  tags = {
    Name        = "secure-distribution"
    Environment = var.environment
  }
}

# ✅ Security Response Headers
resource "aws_cloudfront_response_headers_policy" "security" {
  name    = "security-headers-policy"
  comment = "Security headers for all responses"
  
  security_headers_config {
    # ✅ HSTS
    strict_transport_security {
      access_control_max_age_sec = 31536000  # 1 year
      include_subdomains         = true
      preload                    = true
      override                   = true
    }
    
    # ✅ Prevent clickjacking
    frame_options {
      frame_option = "DENY"
      override     = true
    }
    
    # ✅ Prevent MIME sniffing
    content_type_options {
      override = true
    }
    
    # ✅ XSS protection (older browsers)
    xss_protection {
      mode_block = true
      protection = true
      override   = true
    }
    
    # ✅ Referrer policy
    referrer_policy {
      referrer_policy = "strict-origin-when-cross-origin"
      override        = true
    }
    
    # ✅ Content Security Policy
    content_security_policy {
      content_security_policy = "default-src 'self'; img-src 'self' https:; script-src 'self' 'unsafe-inline'; style-src 'self' 'unsafe-inline'; object-src 'none'"
      override                = true
    }
  }
}
```

---

## ขั้นตอนที่ 854: ElastiCache Redis TLS

### ❌ Vulnerable - Redis Without TLS

```hcl
# ❌ VULNERABLE - Redis ไม่มี TLS
resource "aws_elasticache_replication_group" "no_tls" {
  replication_group_id = "my-redis"
  description          = "Redis without encryption"
  
  node_type  = "cache.t3.micro"
  engine     = "redis"
  
  # ❌ transit_encryption_enabled = false (default in older provider)
  # ❌ ข้อมูลส่งผ่าน network แบบ plain text
  # ❌ credentials อาจถูก intercept
}
```

### ✅ Secure - Redis with TLS

```hcl
# ✅ SECURE - Redis with TLS + AUTH token
resource "random_password" "redis_auth" {
  length  = 32
  special = false  # Redis AUTH token doesn't support some special chars
}

resource "aws_secretsmanager_secret" "redis_auth" {
  name = "prod/redis/auth-token"
}

resource "aws_secretsmanager_secret_version" "redis_auth" {
  secret_id     = aws_secretsmanager_secret.redis_auth.id
  secret_string = random_password.redis_auth.result
}

resource "aws_elasticache_replication_group" "secure_redis" {
  replication_group_id = "my-redis-secure"
  description          = "Redis with TLS"
  
  node_type            = "cache.t3.micro"
  engine               = "redis"
  engine_version       = "7.0"
  
  num_cache_clusters   = 2
  
  # ✅ TLS in transit
  transit_encryption_enabled = true
  
  # ✅ AUTH token required
  auth_token = random_password.redis_auth.result
  
  # ✅ Encryption at rest
  at_rest_encryption_enabled = true
  kms_key_id                 = aws_kms_key.elasticache.arn
  
  subnet_group_name  = aws_elasticache_subnet_group.isolated.name
  security_group_ids = [aws_security_group.elasticache.id]
  
  automatic_failover_enabled = true
  multi_az_enabled           = true
  
  tags = {
    Environment = var.environment
  }
}

# ✅ Application uses TLS when connecting
# Connection string: rediss://:<auth_token>@<endpoint>:6380
# Note: Port 6380 for TLS (vs 6379 for non-TLS)
```

---

## ขั้นตอนที่ 855: RDS SSL Configuration

### ❌ Vulnerable - RDS Without SSL Requirement

```hcl
# ❌ VULNERABLE - MySQL ไม่บังคับ SSL
resource "aws_db_parameter_group" "no_ssl" {
  name   = "mysql-no-ssl"
  family = "mysql8.0"
  
  # ❌ ไม่มี require_secure_transport parameter
  # Default: SSL is optional, not required
}

resource "aws_db_instance" "no_ssl_db" {
  identifier           = "database"
  engine               = "mysql"
  instance_class       = "db.t3.micro"
  parameter_group_name = aws_db_parameter_group.no_ssl.name
  
  # Application can connect without SSL
}
```

### ✅ Secure - RDS with Forced SSL

```hcl
# ✅ SECURE - MySQL requires SSL
resource "aws_db_parameter_group" "mysql_ssl_required" {
  name        = "mysql-ssl-required"
  family      = "mysql8.0"
  description = "MySQL parameter group requiring SSL"
  
  parameter {
    name         = "require_secure_transport"
    value        = "ON"         # ✅ Force SSL
    apply_method = "immediate"
  }
  
  parameter {
    name  = "tls_version"
    value = "TLSv1.2,TLSv1.3"  # ✅ TLS 1.2+
  }
}

# ✅ PostgreSQL SSL requirement
resource "aws_db_parameter_group" "postgres_ssl_required" {
  name        = "postgres-ssl-required"
  family      = "postgres15"
  description = "PostgreSQL parameter group requiring SSL"
  
  parameter {
    name  = "rds.force_ssl"
    value = "1"  # ✅ Force SSL for PostgreSQL
  }
  
  parameter {
    name  = "ssl_min_protocol_version"
    value = "TLSv1.2"  # ✅ Minimum TLS 1.2
  }
}

resource "aws_db_instance" "ssl_required_db" {
  identifier           = "secure-database"
  engine               = "mysql"
  engine_version       = "8.0"
  instance_class       = "db.t3.micro"
  
  parameter_group_name = aws_db_parameter_group.mysql_ssl_required.name  # ✅
  
  storage_encrypted  = true    # ✅ At rest encryption
  kms_key_id         = aws_kms_key.rds.arn
  
  publicly_accessible = false  # ✅ Private only
  
  # ✅ CA certificate for client verification
  ca_cert_identifier = "rds-ca-rsa2048-g1"
}

# ✅ Application connection example
# MySQL: mysql -h <endpoint> -u user -p --ssl-ca=rds-ca-2019-root.pem --ssl-mode=VERIFY_CA
# Postgres: psql "host=<endpoint> sslmode=verify-full sslrootcert=rds-ca.pem"
```

---

## ขั้นตอนที่ 856: OpenSearch/Elasticsearch TLS

### ❌ Vulnerable - OpenSearch Without Node-to-Node Encryption

```hcl
# ❌ VULNERABLE - OpenSearch ไม่มี encryption
resource "aws_opensearch_domain" "no_encryption" {
  domain_name    = "my-search"
  engine_version = "OpenSearch_2.3"
  
  # ❌ ไม่มี encrypt_at_rest
  # ❌ ไม่มี node_to_node_encryption
  # ❌ ไม่มี domain_endpoint_options (HTTPS)
  # ❌ ไม่มี advanced_security_options (fine-grained access)
  
  cluster_config {
    instance_type = "t3.small.search"
  }
}
```

### ✅ Secure - OpenSearch with Full Encryption

```hcl
# ✅ SECURE - OpenSearch with all encryption
resource "aws_kms_key" "opensearch" {
  description             = "OpenSearch encryption key"
  deletion_window_in_days = 30
  enable_key_rotation     = true
}

resource "aws_opensearch_domain" "secure" {
  domain_name    = "my-search-secure"
  engine_version = "OpenSearch_2.9"
  
  cluster_config {
    instance_type          = "t3.small.search"
    instance_count         = 3  # ✅ Multi-node
    zone_awareness_enabled = true
    
    zone_awareness_config {
      availability_zone_count = 3
    }
  }
  
  # ✅ Encryption at rest
  encrypt_at_rest {
    enabled    = true
    kms_key_id = aws_kms_key.opensearch.arn  # ✅ CMK
  }
  
  # ✅ Node-to-node encryption
  node_to_node_encryption {
    enabled = true  # ✅ Internal cluster communication encrypted
  }
  
  # ✅ HTTPS only
  domain_endpoint_options {
    enforce_https       = true         # ✅ Force HTTPS
    tls_security_policy = "Policy-Min-TLS-1-2-2019-07"  # ✅ TLS 1.2
    
    custom_endpoint_enabled         = true
    custom_endpoint                 = "search.${var.domain_name}"
    custom_endpoint_certificate_arn = aws_acm_certificate.main.arn
  }
  
  # ✅ Fine-grained access control
  advanced_security_options {
    enabled                        = true  # ✅ Fine-grained access
    anonymous_auth_enabled         = false  # ✅ No anonymous access
    internal_user_database_enabled = true   # ✅ Or use SAML/Cognito
    
    master_user_options {
      master_user_arn = aws_iam_role.opensearch_admin.arn
    }
  }
  
  # ✅ VPC configuration
  vpc_options {
    subnet_ids         = [aws_subnet.private[0].id, aws_subnet.private[1].id]
    security_group_ids = [aws_security_group.opensearch.id]
  }
  
  # ✅ Log publishing
  log_publishing_options {
    log_type                 = "AUDIT_LOGS"
    cloudwatch_log_group_arn = aws_cloudwatch_log_group.opensearch_audit.arn
  }
  
  log_publishing_options {
    log_type                 = "INDEX_SLOW_LOGS"
    cloudwatch_log_group_arn = aws_cloudwatch_log_group.opensearch_slow.arn
  }
  
  tags = {
    Environment = var.environment
  }
}
```

---

## ขั้นตอนที่ 857: MSK (Kafka) TLS

### ❌ Vulnerable - MSK Without TLS

```hcl
# ❌ VULNERABLE - Kafka ไม่มี TLS
resource "aws_msk_cluster" "no_tls" {
  cluster_name           = "kafka-cluster"
  kafka_version          = "3.4.0"
  number_of_broker_nodes = 3
  
  broker_node_group_info {
    instance_type  = "kafka.t3.small"
    client_subnets = aws_subnet.private[*].id
    storage_info {
      ebs_storage_info {
        volume_size = 100
      }
    }
    security_groups = [aws_security_group.msk.id]
  }
  
  # ❌ ไม่มี client_authentication
  # ❌ ไม่มี encryption_info
  # Default: PLAINTEXT protocol, no encryption
}
```

### ✅ Secure - MSK with TLS

```hcl
# ✅ SECURE - MSK with TLS and mTLS
resource "aws_kms_key" "msk" {
  description             = "MSK encryption key"
  deletion_window_in_days = 30
  enable_key_rotation     = true
}

resource "aws_msk_cluster" "secure" {
  cluster_name           = "kafka-cluster-secure"
  kafka_version          = "3.5.1"
  number_of_broker_nodes = 3
  
  broker_node_group_info {
    instance_type  = "kafka.t3.small"
    client_subnets = aws_subnet.private[*].id
    
    storage_info {
      ebs_storage_info {
        volume_size       = 100
        provisioned_throughput {
          enabled           = true
          volume_throughput = 250
        }
      }
    }
    
    security_groups = [aws_security_group.msk.id]
  }
  
  # ✅ Authentication
  client_authentication {
    # ✅ SASL/SCRAM authentication
    sasl {
      scram = true
    }
    
    # ✅ TLS client certificate (mTLS)
    tls {
      certificate_authority_arns = [aws_acmpca_certificate_authority.msk.arn]
    }
    
    unauthenticated = false  # ✅ No unauthenticated access
  }
  
  # ✅ Encryption
  encryption_info {
    encryption_in_transit {
      client_broker = "TLS"         # ✅ Client to broker: TLS only
      in_cluster    = true           # ✅ Broker to broker: encrypted
    }
    
    encryption_at_rest {
      data_volume_kms_key_id = aws_kms_key.msk.arn  # ✅ At rest encryption
    }
  }
  
  # ✅ Logging
  broker_logs {
    cloudwatch_logs {
      enabled   = true
      log_group = aws_cloudwatch_log_group.msk.name
    }
    s3 {
      enabled = true
      bucket  = aws_s3_bucket.msk_logs.id
      prefix  = "kafka-logs/"
    }
  }
  
  # ✅ Enhanced monitoring
  enhanced_monitoring = "PER_BROKER"
  
  tags = {
    Environment = var.environment
  }
}
```

---

## ขั้นตอนที่ 858: API Gateway HTTPS

### ❌ Vulnerable - API Gateway HTTP

```hcl
# ❌ VULNERABLE - HTTP API without HTTPS enforcement
resource "aws_api_gateway_rest_api" "no_https" {
  name = "my-api"
}

resource "aws_api_gateway_stage" "production" {
  deployment_id = aws_api_gateway_deployment.main.id
  rest_api_id   = aws_api_gateway_rest_api.no_https.id
  stage_name    = "prod"
  
  # ❌ ไม่มี client certificate requirement
  # ❌ ไม่มี access log setting
  # ❌ ไม่มี xray_tracing_enabled
}
```

### ✅ Secure - API Gateway with HTTPS

```hcl
# ✅ SECURE - REST API with all security
resource "aws_api_gateway_rest_api" "secure" {
  name        = "secure-api"
  description = "Secure REST API"
  
  # ✅ Endpoint type
  endpoint_configuration {
    types = ["REGIONAL"]  # REGIONAL or EDGE
  }
  
  # ✅ Minimum compression
  minimum_compression_size = 1024
}

# ✅ Custom domain with TLS
resource "aws_api_gateway_domain_name" "secure" {
  domain_name              = "api.${var.domain_name}"
  regional_certificate_arn = aws_acm_certificate.main.arn
  
  endpoint_configuration {
    types = ["REGIONAL"]
  }
  
  security_policy = "TLS_1_2"  # ✅ Minimum TLS 1.2
}

# ✅ Client certificate for mutual TLS
resource "aws_api_gateway_client_certificate" "mtls" {
  description = "Client certificate for mTLS"
  
  # Certificate for API Gateway to authenticate to backends
}

# ✅ Stage with security settings
resource "aws_api_gateway_stage" "secure" {
  deployment_id = aws_api_gateway_deployment.main.id
  rest_api_id   = aws_api_gateway_rest_api.secure.id
  stage_name    = "prod"
  
  # ✅ Client certificate for backend mTLS
  client_certificate_id = aws_api_gateway_client_certificate.mtls.id
  
  # ✅ X-Ray tracing
  xray_tracing_enabled = true
  
  # ✅ Access logging
  access_log_settings {
    destination_arn = aws_cloudwatch_log_group.api_gw.arn
    format = jsonencode({
      requestId      = "$context.requestId"
      ip             = "$context.identity.sourceIp"
      caller         = "$context.identity.caller"
      user           = "$context.identity.user"
      requestTime    = "$context.requestTime"
      httpMethod     = "$context.httpMethod"
      resourcePath   = "$context.resourcePath"
      status         = "$context.status"
      protocol       = "$context.protocol"
      responseLength = "$context.responseLength"
    })
  }
  
  # ✅ Default route throttling
  default_route_settings {
    throttling_burst_limit = 5000
    throttling_rate_limit  = 10000
  }
}

# ✅ HTTP to HTTPS redirect for API Gateway
# API Gateway HTTP API
resource "aws_apigatewayv2_api" "secure" {
  name          = "secure-http-api"
  protocol_type = "HTTP"
  
  # ✅ CORS configuration
  cors_configuration {
    allow_credentials = true
    allow_headers     = ["content-type", "x-amz-date", "authorization"]
    allow_methods     = ["GET", "POST", "PUT", "DELETE", "OPTIONS"]
    allow_origins     = ["https://${var.domain_name}"]  # ✅ Specific origin
    expose_headers    = ["x-amzn-requestid"]
    max_age           = 3600
  }
}

resource "aws_apigatewayv2_stage" "secure" {
  api_id      = aws_apigatewayv2_api.secure.id
  name        = "$default"
  auto_deploy = true
  
  # ✅ Access logging
  access_log_settings {
    destination_arn = aws_cloudwatch_log_group.api_gw_v2.arn
  }
  
  # ✅ Default throttling
  default_route_settings {
    throttling_burst_limit = 5000
    throttling_rate_limit  = 10000
    logging_level          = "INFO"
    data_trace_enabled     = true
    detailed_metrics_enabled = true
  }
}
```

---

## ขั้นตอนที่ 859: Certificate Management

### ❌ Vulnerable - Self-signed Certificates

```hcl
# ❌ VULNERABLE - Self-signed cert (no trust chain)
resource "tls_self_signed_cert" "bad_practice" {
  private_key_pem = tls_private_key.bad.private_key_pem
  
  subject {
    common_name  = "*.example.com"
    organization = "Example Corp"
  }
  
  validity_period_hours = 8760
  
  allowed_uses = ["key_encipherment", "digital_signature", "server_auth"]
  
  # ❌ Self-signed = Browsers/clients won't trust
  # ❌ Easy to spoof
  # ❌ No revocation mechanism
}
```

### ✅ Secure - ACM Managed Certificates

```hcl
# ✅ SECURE - ACM certificates (auto-renewed)
resource "aws_acm_certificate" "main" {
  domain_name               = var.domain_name
  subject_alternative_names = [
    "*.${var.domain_name}",
    "api.${var.domain_name}",
    "cdn.${var.domain_name}"
  ]
  validation_method = "DNS"
  
  lifecycle {
    create_before_destroy = true  # ✅ Zero downtime replacement
  }
  
  tags = {
    Name        = "main-certificate"
    Environment = var.environment
  }
}

# ✅ Auto-validate via Route53
resource "aws_route53_record" "cert_validation" {
  for_each = {
    for dvo in aws_acm_certificate.main.domain_validation_options : dvo.domain_name => {
      name   = dvo.resource_record_name
      record = dvo.resource_record_value
      type   = dvo.resource_record_type
    }
  }
  
  zone_id = data.aws_route53_zone.main.zone_id
  name    = each.value.name
  type    = each.value.type
  records = [each.value.record]
  ttl     = 60
  
  allow_overwrite = true
}

resource "aws_acm_certificate_validation" "main" {
  certificate_arn         = aws_acm_certificate.main.arn
  validation_record_fqdns = [for record in aws_route53_record.cert_validation : record.fqdn]
}

# ✅ Monitor certificate expiry
resource "aws_cloudwatch_metric_alarm" "cert_expiry" {
  alarm_name          = "ACMCertificateExpiry"
  comparison_operator = "LessThanThreshold"
  evaluation_periods  = 1
  metric_name         = "DaysToExpiry"
  namespace           = "AWS/CertificateManager"
  period              = 86400  # Daily
  statistic           = "Minimum"
  threshold           = 30     # Alert 30 days before expiry
  
  dimensions = {
    CertificateArn = aws_acm_certificate.main.arn
  }
  
  alarm_description = "SSL certificate expiring in 30 days"
  alarm_actions     = [aws_sns_topic.security_alerts.arn]
}

# ✅ Private CA for internal services (optional)
resource "aws_acmpca_certificate_authority" "internal" {
  certificate_authority_configuration {
    key_algorithm     = "RSA_4096"
    signing_algorithm = "SHA512WITHRSA"
    
    subject {
      common_name         = "Internal CA"
      organization        = "Company Name"
      organizational_unit = "IT Security"
      country             = "TH"
    }
  }
  
  permanent_deletion_time_in_days = 7
  type                            = "ROOT"
  
  tags = {
    Name = "internal-ca"
  }
}
```

---

## ขั้นตอนที่ 860: S3 SSL Enforcement

### ❌ Vulnerable - S3 Without SSL Enforcement

```hcl
# ❌ VULNERABLE - ไม่บังคับ SSL สำหรับ S3
resource "aws_s3_bucket" "no_ssl" {
  bucket = "my-bucket"
}

# ไม่มี bucket policy = HTTP access allowed
```

### ✅ Secure - S3 with SSL Policy

```hcl
# ✅ SECURE - Force SSL for all S3 access
resource "aws_s3_bucket" "force_ssl" {
  bucket = "my-secure-bucket"
}

resource "aws_s3_bucket_policy" "force_ssl" {
  bucket = aws_s3_bucket.force_ssl.id
  
  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      # ✅ Deny all non-SSL access
      {
        Sid       = "DenyNonSSL"
        Effect    = "Deny"
        Principal = "*"
        Action    = "s3:*"
        Resource = [
          aws_s3_bucket.force_ssl.arn,
          "${aws_s3_bucket.force_ssl.arn}/*"
        ]
        Condition = {
          Bool = {
            "aws:SecureTransport" = "false"
          }
        }
      },
      # ✅ Deny old TLS versions
      {
        Sid       = "DenyOldTLS"
        Effect    = "Deny"
        Principal = "*"
        Action    = "s3:*"
        Resource = [
          aws_s3_bucket.force_ssl.arn,
          "${aws_s3_bucket.force_ssl.arn}/*"
        ]
        Condition = {
          NumericLessThan = {
            "s3:TlsVersion" = "1.2"  # ✅ Minimum TLS 1.2
          }
        }
      }
    ]
  })
}
```

---

## สรุป Encryption in Transit

### Checkov Rules สำหรับ Encryption in Transit

| Service | Issue | Checkov Rule | Fix |
|---------|-------|-------------|-----|
| ALB | HTTP listener | CKV_AWS_2 | Redirect to HTTPS |
| ALB | Weak TLS policy | CKV_AWS_103 | Use TLS 1.2+ policy |
| CloudFront | HTTP allowed | CKV_AWS_86 | viewer_protocol_policy = redirect-to-https |
| CloudFront | Old TLS | CKV_AWS_174 | minimum_protocol_version = TLSv1.2_2021 |
| ElastiCache | No TLS | CKV_AWS_31 | transit_encryption_enabled = true |
| RDS | No SSL param | CKV_AWS_17 | require_secure_transport = ON |
| OpenSearch | No HTTPS | CKV_AWS_84 | enforce_https = true |
| OpenSearch | No node-to-node | CKV_AWS_83 | node_to_node_encryption enabled |
| MSK | No TLS | CKV_AWS_80 | client_broker = TLS |
| S3 | HTTP allowed | CKV_AWS_20 | Force SSL bucket policy |

---

*Part 086 ครอบคลุม Encryption in Transit ทั้งหมด - ต่อไปใน Part 087 จะเจาะลึก Logging & Monitoring*
