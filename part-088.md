# Part 088: Public Exposure Vulnerabilities
## ขั้นตอนที่ 871-880: ช่องโหว่จากการ expose ทรัพยากรสู่สาธารณะ

---

## ขั้นตอนที่ 871: ภาพรวม Public Exposure

### ทำไม Public Exposure จึงอันตราย

```
Public Exposure Attack Scenarios:
1. Database exposed publicly → Direct SQL injection/brute force
2. EC2 with public IP → Port scanning, exploitation
3. Lambda URL without auth → Unauthorized invocations
4. API Gateway without auth → Data theft, abuse
5. ElasticSearch public → Data exposure
6. S3 bucket public → Data breach
7. K8s API public → Cluster takeover
8. Redis public → Data theft, ransomware

ต้นทุนเฉลี่ย:
- Public database breach: $4.24M (IBM 2023)
- Exposed API key: Average $1.3M in unauthorized charges
- Public S3 bucket: $3.86M data breach cost
```

### Attack Surface Reduction Principles

```
Defense in Depth for Public Exposure:
1. Never expose directly - use load balancers
2. Private subnets for everything backend
3. Bastion/VPN for admin access
4. VPC endpoints instead of internet
5. WAF before any public endpoints
6. Authentication on everything public
7. Rate limiting to prevent abuse
8. Regular exposure scanning
```

---

## ขั้นตอนที่ 872: EC2 Public IP Exposure

### ❌ Vulnerable - EC2 with Public IP

```hcl
# ❌ VULNERABLE - Direct public IP
resource "aws_instance" "public_server" {
  ami           = data.aws_ami.amazon_linux.id
  instance_type = "t3.micro"
  
  # ❌ Public IP assigned automatically
  associate_public_ip_address = true
  subnet_id                   = aws_subnet.public.id  # ❌ Public subnet
  
  # ❌ Security group allows SSH from anywhere
  vpc_security_group_ids = [aws_security_group.allow_all.id]
  
  # Server ไม่มี WAF ป้องกัน
  # Server รับ traffic ตรงจาก internet
}

# ❌ VULNERABLE - EIP directly on instance
resource "aws_eip" "direct_server" {
  instance = aws_instance.public_server.id
  domain   = "vpc"
  # ❌ Direct internet access to server
}
```

### ✅ Secure - EC2 Behind ALB in Private Subnet

```hcl
# ✅ SECURE - EC2 in private subnet, no public IP
resource "aws_instance" "private_server" {
  ami           = data.aws_ami.amazon_linux.id
  instance_type = "t3.micro"
  
  associate_public_ip_address = false  # ✅ No public IP
  subnet_id                   = aws_subnet.private.id  # ✅ Private subnet
  
  # ✅ Only allow traffic from ALB
  vpc_security_group_ids = [aws_security_group.app_server.id]
  
  # ✅ IAM role for SSM (no SSH needed)
  iam_instance_profile = aws_iam_instance_profile.ssm.name
  
  # ✅ Encrypted root volume
  root_block_device {
    encrypted   = true
    volume_type = "gp3"
  }
  
  tags = {
    Name = "private-app-server"
  }
}

# ✅ ALB in public subnet handles external traffic
resource "aws_lb" "main" {
  name               = "main-alb"
  load_balancer_type = "application"
  subnets            = aws_subnet.public[*].id
  
  # ✅ WAF attached
  # (done via aws_wafv2_web_acl_association)
  
  enable_deletion_protection = true
  
  access_logs {
    bucket  = aws_s3_bucket.alb_logs.id
    enabled = true
  }
}
```

---

## ขั้นตอนที่ 873: RDS Public Accessibility

### ❌ Vulnerable - RDS Publicly Accessible

```hcl
# ❌ VULNERABLE - RDS accessible from internet
resource "aws_db_instance" "public_rds" {
  identifier     = "main-db"
  engine         = "mysql"
  instance_class = "db.t3.micro"
  
  publicly_accessible = true  # ❌ Internet accessible!
  
  # Even with security groups, public = risk
  # CVE scans will find it, brute force attempts
  # One misconfigured SG = game over
}
```

### ✅ Secure - RDS in Isolated Subnet

```hcl
# ✅ SECURE - RDS completely private
resource "aws_db_instance" "private_rds" {
  identifier     = "main-db"
  engine         = "mysql"
  instance_class = "db.t3.micro"
  
  publicly_accessible  = false  # ✅ Private only
  
  db_subnet_group_name = aws_db_subnet_group.isolated.name  # ✅ Isolated subnet
  
  vpc_security_group_ids = [aws_security_group.rds.id]
  
  # ✅ Encryption
  storage_encrypted = true
  kms_key_id        = aws_kms_key.rds.arn
  
  # ✅ IAM authentication (no password needed!)
  iam_database_authentication_enabled = true
  
  tags = {
    Name = "private-database"
  }
}

# ✅ Use RDS Proxy for application access
resource "aws_db_proxy" "main" {
  name           = "main-proxy"
  engine_family  = "MYSQL"
  role_arn       = aws_iam_role.rds_proxy.arn
  require_tls    = true  # ✅ TLS required
  
  vpc_subnet_ids         = aws_subnet.private[*].id  # ✅ Private
  vpc_security_group_ids = [aws_security_group.rds_proxy.id]
  
  auth {
    auth_scheme = "SECRETS"
    iam_auth    = "REQUIRED"  # ✅ IAM authentication
    secret_arn  = aws_secretsmanager_secret.db_credentials.arn
  }
}

# ✅ AWS Config rule to detect public RDS
resource "aws_config_config_rule" "rds_not_public" {
  name        = "rds-instance-public-access-check"
  description = "Checks if RDS instances are not publicly accessible"
  
  source {
    owner             = "AWS"
    source_identifier = "RDS_INSTANCE_PUBLIC_ACCESS_CHECK"
  }
}
```

---

## ขั้นตอนที่ 874: OpenSearch/ElasticSearch Public Access

### ❌ Vulnerable - OpenSearch Public

```hcl
# ❌ VULNERABLE - OpenSearch public access
resource "aws_opensearch_domain" "public" {
  domain_name    = "search-cluster"
  engine_version = "OpenSearch_2.3"
  
  cluster_config {
    instance_type = "t3.small.search"
  }
  
  # ❌ ไม่มี vpc_options = public access!
  # ❌ ไม่มี advanced_security_options = no auth
  # ❌ ทุกคน query ได้
  
  # access_policies ที่ผิดพลาด
  access_policies = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect    = "Allow"
      Principal = { AWS = "*" }  # ❌ Everyone!
      Action    = "es:*"
      Resource  = "arn:aws:es:*:*:domain/search-cluster/*"
    }]
  })
}
```

### ✅ Secure - OpenSearch Private

```hcl
# ✅ SECURE - OpenSearch in VPC with fine-grained access
resource "aws_opensearch_domain" "private" {
  domain_name    = "search-cluster-secure"
  engine_version = "OpenSearch_2.9"
  
  cluster_config {
    instance_type          = "t3.small.search"
    instance_count         = 2
    zone_awareness_enabled = true
  }
  
  # ✅ VPC deployment
  vpc_options {
    subnet_ids         = [aws_subnet.private[0].id, aws_subnet.private[1].id]
    security_group_ids = [aws_security_group.opensearch.id]
  }
  
  # ✅ Fine-grained access control
  advanced_security_options {
    enabled                        = true
    anonymous_auth_enabled         = false  # ✅ No anonymous
    internal_user_database_enabled = false  # ✅ Use IAM only
    
    master_user_options {
      master_user_arn = aws_iam_role.opensearch_admin.arn
    }
  }
  
  # ✅ Encryption
  encrypt_at_rest { enabled = true }
  node_to_node_encryption { enabled = true }
  
  domain_endpoint_options {
    enforce_https       = true
    tls_security_policy = "Policy-Min-TLS-1-2-2019-07"
  }
  
  # ✅ Restrictive access policy
  access_policies = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect = "Allow"
      Principal = {
        AWS = [
          aws_iam_role.app_role.arn,  # ✅ Specific roles
          aws_iam_role.opensearch_admin.arn
        ]
      }
      Action   = ["es:ESHttp*"]
      Resource = "arn:aws:es:${var.region}:${var.account_id}:domain/search-cluster-secure/*"
    }]
  })
}

# ✅ OpenSearch Security Group
resource "aws_security_group" "opensearch" {
  name   = "opensearch-sg"
  vpc_id = aws_vpc.main.id
  
  # ✅ Only from app servers
  ingress {
    from_port                = 443
    to_port                  = 443
    protocol                 = "tcp"
    source_security_group_id = aws_security_group.app_server.id
  }
}
```

---

## ขั้นตอนที่ 875: EKS API Server Exposure

### ❌ Vulnerable - EKS Public Endpoint Only

```hcl
# ❌ VULNERABLE - EKS API server public
resource "aws_eks_cluster" "public_api" {
  name     = "production"
  role_arn = aws_iam_role.eks.arn
  
  vpc_config {
    subnet_ids = aws_subnet.private[*].id
    
    endpoint_public_access  = true   # ❌ Public API endpoint
    endpoint_private_access = false  # ❌ No private access
    
    # ❌ ไม่มี public_access_cidrs = ทุก IP เข้าได้
  }
}
```

### ✅ Secure - EKS Private Endpoint

```hcl
# ✅ SECURE - EKS with private endpoint
resource "aws_eks_cluster" "private_api" {
  name     = "production"
  role_arn = aws_iam_role.eks.arn
  
  vpc_config {
    subnet_ids = aws_subnet.private[*].id
    
    endpoint_public_access  = false  # ✅ No public API
    endpoint_private_access = true   # ✅ Private access only
    
    security_group_ids = [aws_security_group.eks_cluster.id]
  }
  
  # ✅ Allow public access only from specific CIDRs (if needed)
  # endpoint_public_access = true
  # public_access_cidrs    = var.allowed_cidr_blocks
  
  # ✅ Secrets encryption
  encryption_config {
    resources = ["secrets"]
    provider { key_arn = aws_kms_key.eks.arn }
  }
  
  enabled_cluster_log_types = ["api", "audit", "authenticator", "controllerManager", "scheduler"]
}

# ✅ VPC endpoint for EKS (allows worker nodes in different VPC)
resource "aws_vpc_endpoint" "eks_api" {
  vpc_id              = aws_vpc.main.id
  service_name        = "com.amazonaws.${var.region}.eks"
  vpc_endpoint_type   = "Interface"
  subnet_ids          = aws_subnet.private[*].id
  security_group_ids  = [aws_security_group.vpc_endpoints.id]
  private_dns_enabled = true
}

# ✅ RBAC least privilege
# kubectl apply -f least-privilege-rbac.yaml
# apiVersion: rbac.authorization.k8s.io/v1
# kind: ClusterRoleBinding
# metadata:
#   name: developer-binding
# subjects:
# - kind: User
#   name: developer-alice
# roleRef:
#   kind: ClusterRole
#   name: view  # Read-only
```

---

## ขั้นตอนที่ 876: Lambda Function URL & API Auth

### ❌ Vulnerable - Lambda Without Auth

```hcl
# ❌ VULNERABLE - Lambda URL without auth
resource "aws_lambda_function_url" "no_auth" {
  function_name      = aws_lambda_function.api.function_name
  authorization_type = "NONE"  # ❌ No authentication
  
  # ❌ Anyone can call this URL:
  # https://xxxx.lambda-url.us-east-1.on.aws/
}

# ❌ VULNERABLE - Lambda with CORS allowing all
resource "aws_lambda_function_url" "bad_cors" {
  function_name      = aws_lambda_function.api.function_name
  authorization_type = "NONE"
  
  cors {
    allow_credentials = true
    allow_origins     = ["*"]  # ❌ Any origin
    allow_methods     = ["*"]  # ❌ Any method
    allow_headers     = ["*"]  # ❌ Any header
  }
}
```

### ✅ Secure - Lambda with IAM Auth

```hcl
# ✅ SECURE - Lambda URL with IAM auth
resource "aws_lambda_function_url" "with_auth" {
  function_name      = aws_lambda_function.api.function_name
  authorization_type = "AWS_IAM"  # ✅ IAM authentication required
  
  # ✅ Specific CORS origins
  cors {
    allow_credentials = true
    allow_origins     = ["https://${var.domain_name}"]  # ✅ Specific origin
    allow_methods     = ["GET", "POST"]                  # ✅ Specific methods
    allow_headers     = ["Content-Type", "Authorization"]
    expose_headers    = ["X-Amz-Date", "X-Api-Key"]
    max_age           = 86400
  }
}

# ✅ Resource policy to allow specific principals
resource "aws_lambda_permission" "function_url" {
  statement_id           = "FunctionURLAllowPublicAccess"
  action                 = "lambda:InvokeFunctionUrl"
  function_name          = aws_lambda_function.api.function_name
  principal              = aws_iam_role.caller_role.arn  # ✅ Specific role
  function_url_auth_type = "AWS_IAM"
}

# ✅ Better: Use API Gateway instead of direct Lambda URL
resource "aws_api_gateway_rest_api" "secure" {
  name = "secure-api"
}

resource "aws_api_gateway_authorizer" "cognito" {
  name          = "CognitoAuthorizer"
  rest_api_id   = aws_api_gateway_rest_api.secure.id
  type          = "COGNITO_USER_POOLS"  # ✅ Cognito auth
  provider_arns = [aws_cognito_user_pool.main.arn]
}
```

---

## ขั้นตอนที่ 877: Cognito & CloudFront WAF

### ❌ Vulnerable - Cognito Without MFA

```hcl
# ❌ VULNERABLE - Cognito ไม่บังคับ MFA
resource "aws_cognito_user_pool" "no_mfa" {
  name = "user-pool"
  
  # ❌ ไม่มี mfa_configuration
  # Default: MFA optional
  
  password_policy {
    minimum_length    = 8         # ❌ Too short
    require_lowercase = false
    require_numbers   = false
    require_symbols   = false
    require_uppercase = false
    # ❌ Weak password policy
  }
}
```

### ✅ Secure - Cognito with MFA

```hcl
# ✅ SECURE - Cognito with MFA and strong policy
resource "aws_cognito_user_pool" "secure" {
  name = "secure-user-pool"
  
  # ✅ Strong password policy
  password_policy {
    minimum_length                   = 14   # ✅
    require_lowercase                = true  # ✅
    require_numbers                  = true  # ✅
    require_symbols                  = true  # ✅
    require_uppercase                = true  # ✅
    temporary_password_validity_days = 7    # ✅
  }
  
  # ✅ MFA required
  mfa_configuration = "ON"  # ✅ Always required
  
  software_token_mfa_configuration {
    enabled = true  # ✅ TOTP (Authenticator app)
  }
  
  # ✅ Account recovery (not SMS, use email)
  account_recovery_setting {
    recovery_mechanism {
      name     = "verified_email"
      priority = 1
    }
  }
  
  # ✅ Email verification required
  auto_verified_attributes = ["email"]
  
  # ✅ User attribute schema
  schema {
    attribute_data_type = "String"
    name                = "email"
    required            = true
    mutable             = true
    
    string_attribute_constraints {
      min_length = 5
      max_length = 256
    }
  }
  
  # ✅ Advanced security
  user_pool_add_ons {
    advanced_security_mode = "ENFORCED"  # ✅ ML-based risk detection
  }
  
  # ✅ Device tracking
  device_configuration {
    challenge_required_on_new_device      = true   # ✅
    device_only_remembered_on_user_prompt = true
  }
  
  # ✅ Brute force protection via Cognito
  # (built-in with advanced security mode)
  
  tags = {
    Environment = var.environment
  }
}
```

### ❌ Vulnerable - CloudFront Without WAF

```hcl
# ❌ VULNERABLE - CloudFront ไม่มี WAF
resource "aws_cloudfront_distribution" "no_waf" {
  # ❌ ไม่มี web_acl_id
  # ❌ ไม่มี protection จาก:
  # - SQL injection
  # - XSS
  # - Rate limiting
  # - IP blocking
  # - Known bad IPs
  
  enabled = true
  # ...
}
```

### ✅ Secure - CloudFront with WAF

```hcl
# ✅ SECURE - WAF for CloudFront (must be us-east-1)
resource "aws_wafv2_web_acl" "cloudfront" {
  provider = aws.us_east_1
  
  name        = "cloudfront-waf"
  description = "WAF for CloudFront"
  scope       = "CLOUDFRONT"
  
  default_action {
    allow {}  # Default allow, specific deny rules
  }
  
  # ✅ Rate limiting
  rule {
    name     = "RateLimitRule"
    priority = 1
    
    action {
      block {}
    }
    
    statement {
      rate_based_statement {
        limit              = 2000  # ✅ 2000 requests per 5 minutes
        aggregate_key_type = "IP"
      }
    }
    
    visibility_config {
      cloudwatch_metrics_enabled = true
      metric_name                = "RateLimitRule"
      sampled_requests_enabled   = true
    }
  }
  
  # ✅ AWS Managed Rule - Core Rule Set
  rule {
    name     = "AWSManagedRulesCommonRuleSet"
    priority = 2
    
    override_action {
      none {}  # Use managed rule's action
    }
    
    statement {
      managed_rule_group_statement {
        name        = "AWSManagedRulesCommonRuleSet"
        vendor_name = "AWS"
      }
    }
    
    visibility_config {
      cloudwatch_metrics_enabled = true
      metric_name                = "AWSManagedRulesCommonRuleSet"
      sampled_requests_enabled   = true
    }
  }
  
  # ✅ AWS Managed Rule - Known Bad Inputs
  rule {
    name     = "AWSManagedRulesKnownBadInputsRuleSet"
    priority = 3
    
    override_action {
      none {}
    }
    
    statement {
      managed_rule_group_statement {
        name        = "AWSManagedRulesKnownBadInputsRuleSet"
        vendor_name = "AWS"
      }
    }
    
    visibility_config {
      cloudwatch_metrics_enabled = true
      metric_name                = "KnownBadInputs"
      sampled_requests_enabled   = true
    }
  }
  
  # ✅ SQL Injection protection
  rule {
    name     = "SQLiProtection"
    priority = 4
    
    action {
      block {}
    }
    
    statement {
      sqli_match_statement {
        field_to_match {
          all_query_arguments {}
        }
        text_transformation {
          priority = 1
          type     = "URL_DECODE"
        }
        text_transformation {
          priority = 2
          type     = "HTML_ENTITY_DECODE"
        }
      }
    }
    
    visibility_config {
      cloudwatch_metrics_enabled = true
      metric_name                = "SQLiProtection"
      sampled_requests_enabled   = true
    }
  }
  
  # ✅ IP Block list
  rule {
    name     = "BlockBadIPs"
    priority = 10
    
    action {
      block {}
    }
    
    statement {
      ip_set_reference_statement {
        arn = aws_wafv2_ip_set.blocked_ips.arn
      }
    }
    
    visibility_config {
      cloudwatch_metrics_enabled = true
      metric_name                = "BlockedIPs"
      sampled_requests_enabled   = true
    }
  }
  
  visibility_config {
    cloudwatch_metrics_enabled = true
    metric_name                = "CloudFrontWAF"
    sampled_requests_enabled   = true
  }
  
  tags = {
    Environment = var.environment
  }
}

# ✅ IP block list
resource "aws_wafv2_ip_set" "blocked_ips" {
  provider = aws.us_east_1
  
  name               = "blocked-ips"
  description        = "Known malicious IP addresses"
  scope              = "CLOUDFRONT"
  ip_address_version = "IPV4"
  
  addresses = var.blocked_ip_addresses  # Maintained list
}

# ✅ Associate WAF with CloudFront
resource "aws_cloudfront_distribution" "with_waf" {
  enabled = true
  
  web_acl_id = aws_wafv2_web_acl.cloudfront.arn  # ✅ WAF attached
  
  # ... other configuration
}
```

---

## ขั้นตอนที่ 878: Secrets in SSM Parameter Store

### ❌ Vulnerable - Secrets in Plain Text SSM

```hcl
# ❌ VULNERABLE - Sensitive value as String (not SecureString)
resource "aws_ssm_parameter" "db_password_plain" {
  name  = "/prod/db/password"
  type  = "String"          # ❌ Plain text! base64 encoding only
  value = var.db_password
  
  # ❌ Anyone with ssm:GetParameter can see it
  # ❌ Visible in AWS Console without KMS access
}

# ❌ ALSO VULNERABLE - API key in String type
resource "aws_ssm_parameter" "api_key_plain" {
  name  = "/prod/stripe/api-key"
  type  = "String"          # ❌ Plain text
  value = var.stripe_api_key
}

# ❌ ALSO VULNERABLE - StringList for sensitive data
resource "aws_ssm_parameter" "credentials_list" {
  name  = "/prod/credentials"
  type  = "StringList"      # ❌ Plain text list
  value = join(",", [var.user, var.password])
}
```

### ✅ Secure - Secrets in SecureString

```hcl
# ✅ SECURE - Use SecureString with CMK
resource "aws_kms_key" "ssm" {
  description             = "SSM Parameter Store encryption key"
  deletion_window_in_days = 30
  enable_key_rotation     = true
}

resource "aws_ssm_parameter" "db_password_secure" {
  name        = "/prod/db/password"
  description = "Database password for production"
  type        = "SecureString"         # ✅ Encrypted
  value       = var.db_password
  key_id      = aws_kms_key.ssm.arn    # ✅ Customer managed key
  
  # ✅ Tier
  tier = "Standard"
  
  tags = {
    Environment = "production"
    DataClass   = "confidential"
  }
  
  lifecycle {
    ignore_changes = [value]  # ✅ Don't track changes in state
  }
}

# ✅ BETTER: Use Secrets Manager for sensitive data
resource "aws_secretsmanager_secret" "db_credentials" {
  name        = "prod/db/credentials"
  description = "Database credentials for production"
  
  kms_key_id = aws_kms_key.secrets.arn
  
  rotation_rules {
    automatically_after_days = 30  # ✅ Auto-rotate
  }
  
  recovery_window_in_days = 30
}

# ✅ IAM Policy - limit who can read secrets
resource "aws_iam_policy" "read_db_secret" {
  name = "ReadDBSecret"
  
  policy = jsonencode({
    Statement = [
      {
        Effect = "Allow"
        Action = [
          "secretsmanager:GetSecretValue",
          "secretsmanager:DescribeSecret"
        ]
        Resource = aws_secretsmanager_secret.db_credentials.arn
      },
      {
        Effect = "Allow"
        Action = [
          "kms:Decrypt",
          "kms:DescribeKey"
        ]
        Resource = aws_kms_key.secrets.arn
        Condition = {
          StringEquals = {
            "kms:ViaService" = "secretsmanager.${var.region}.amazonaws.com"
          }
        }
      }
    ]
  })
}

# ✅ Detect plain text secrets via AWS Config
resource "aws_config_config_rule" "ssm_no_plain_secrets" {
  name        = "ssm-parameter-no-plain-text"
  description = "Check that SSM parameters of type String don't contain sensitive data"
  
  source {
    owner             = "AWS"
    source_identifier = "SSM_PARAMETER_NOT_PLAINTEXT"
  }
}
```

---

## ขั้นตอนที่ 879: ALB Without WAF & Route53 Security

### ALB WAF Protection

```hcl
# ✅ WAF for ALB (Regional - same region as ALB)
resource "aws_wafv2_web_acl" "alb" {
  name  = "alb-waf"
  scope = "REGIONAL"
  
  default_action {
    allow {}
  }
  
  # ✅ Rate limiting
  rule {
    name     = "RateLimit"
    priority = 1
    
    action { block {} }
    
    statement {
      rate_based_statement {
        limit              = 1000
        aggregate_key_type = "IP"
        
        # ✅ Rate limit specific paths
        scope_down_statement {
          byte_match_statement {
            search_string         = "/api/login"
            positional_constraint = "STARTS_WITH"
            field_to_match { uri_path {} }
            text_transformation {
              priority = 0
              type     = "LOWERCASE"
            }
          }
        }
      }
    }
    
    visibility_config {
      cloudwatch_metrics_enabled = true
      metric_name                = "LoginRateLimit"
      sampled_requests_enabled   = true
    }
  }
  
  # ✅ Geo blocking (if needed)
  rule {
    name     = "GeoBlock"
    priority = 5
    
    action { block {} }
    
    statement {
      geo_match_statement {
        country_codes = var.blocked_countries  # e.g., ["CN", "RU"]
      }
    }
    
    visibility_config {
      cloudwatch_metrics_enabled = true
      metric_name                = "GeoBlock"
      sampled_requests_enabled   = true
    }
  }
  
  # ✅ AWS Managed Rules
  rule {
    name     = "CoreRuleSet"
    priority = 10
    override_action { none {} }
    statement {
      managed_rule_group_statement {
        name        = "AWSManagedRulesCommonRuleSet"
        vendor_name = "AWS"
      }
    }
    visibility_config {
      cloudwatch_metrics_enabled = true
      metric_name                = "CoreRuleSet"
      sampled_requests_enabled   = true
    }
  }
  
  visibility_config {
    cloudwatch_metrics_enabled = true
    metric_name                = "ALBWAF"
    sampled_requests_enabled   = true
  }
}

# ✅ Associate WAF with ALB
resource "aws_wafv2_web_acl_association" "alb" {
  resource_arn = aws_lb.main.arn
  web_acl_arn  = aws_wafv2_web_acl.alb.arn
}

# ✅ WAF Logging
resource "aws_wafv2_web_acl_logging_configuration" "alb" {
  log_destination_configs = [aws_kinesis_firehose_delivery_stream.waf.arn]
  resource_arn            = aws_wafv2_web_acl.alb.arn
  
  redacted_fields {
    single_header {
      name = "authorization"  # ✅ Redact auth headers from logs
    }
  }
}
```

### Route53 Zone Enumeration Prevention

```hcl
# ❌ Route53 public hosted zone can be enumerated
# Attacker: dig axfr @ns1.example.com example.com
# → Gets all DNS records

# ✅ SECURE - DNSSEC signing
resource "aws_route53_key_signing_key" "main" {
  hosted_zone_id             = aws_route53_zone.main.zone_id
  key_management_service_arn = aws_kms_key.dnssec.arn
  name                       = "main-ksk"
}

resource "aws_route53_hosted_zone_dnssec" "main" {
  hosted_zone_id = aws_route53_zone.main.zone_id
  
  depends_on = [aws_route53_key_signing_key.main]
}

# ✅ Health checks with failover
resource "aws_route53_health_check" "primary" {
  fqdn              = var.domain_name
  port              = 443
  type              = "HTTPS"
  resource_path     = "/health"
  failure_threshold = 3
  request_interval  = 30
  
  tags = {
    Name = "primary-health-check"
  }
}

# ✅ Private hosted zone for internal resources
resource "aws_route53_zone" "internal" {
  name = "internal.${var.domain_name}"
  
  vpc {
    vpc_id = aws_vpc.main.id
  }
  
  tags = {
    Name = "internal-zone"
    Type = "private"
  }
}
```

---

## ขั้นตอนที่ 880: Complete Exposure Scanning

### Automated Public Exposure Detection

```hcl
# ✅ IAM Access Analyzer - ตรวจจับ external access
resource "aws_accessanalyzer_analyzer" "main" {
  analyzer_name = "main-analyzer"
  type          = "ACCOUNT"
}

# ✅ AWS Config rules for exposure
locals {
  exposure_config_rules = {
    "restricted-ssh"             = "INCOMING_SSH_DISABLED"
    "restricted-rdp"             = "RESTRICTED_INCOMING_TRAFFIC"
    "rds-no-public-access"       = "RDS_INSTANCE_PUBLIC_ACCESS_CHECK"
    "ec2-no-public-ip"           = "EC2_INSTANCE_NO_PUBLIC_IP"
    "lambda-no-public-access"    = "LAMBDA_FUNCTION_PUBLIC_ACCESS_PROHIBITED"
    "opensearch-no-public"       = "OPENSEARCH_IN_VPC_ONLY"
    "redshift-no-public"         = "REDSHIFT_CLUSTER_PUBLIC_ACCESS_CHECK"
  }
}

resource "aws_config_config_rule" "exposure" {
  for_each = local.exposure_config_rules
  
  name = each.key
  
  source {
    owner             = "AWS"
    source_identifier = each.value
  }
}

# ✅ SNS notification for new public resources
resource "aws_cloudwatch_event_rule" "new_public_resources" {
  name        = "NewPublicResources"
  description = "Detect new publicly accessible resources"
  
  event_pattern = jsonencode({
    source = ["aws.accessanalyzer"]
    detail-type = ["Access Analyzer Finding"]
    detail = {
      type = [
        "AWS::S3::Bucket",
        "AWS::IAM::Role",
        "AWS::SQS::Queue",
        "AWS::Lambda::Function",
        "AWS::KMS::Key"
      ]
      isPublic = [true]
    }
  })
}

resource "aws_cloudwatch_event_target" "new_public_sns" {
  rule = aws_cloudwatch_event_rule.new_public_resources.name
  arn  = aws_sns_topic.security_alerts.arn
}

# ✅ Lambda for auto-remediation of public resources
resource "aws_lambda_function" "auto_remediate" {
  filename      = "auto-remediate.zip"
  function_name = "auto-remediate-public-resources"
  role          = aws_iam_role.auto_remediate.arn
  handler       = "index.handler"
  runtime       = "python3.12"
  
  environment {
    variables = {
      SNS_TOPIC_ARN = aws_sns_topic.security_alerts.arn
    }
  }
}
```

---

## สรุป Public Exposure Vulnerabilities

### Complete Exposure Security Matrix

| Resource | Vulnerability | Risk | Fix |
|----------|--------------|------|-----|
| EC2 | Public IP | High | Private subnet + ALB |
| RDS | publicly_accessible=true | Critical | private subnet |
| OpenSearch | No VPC | Critical | VPC + fine-grained |
| EKS API | Public endpoint | High | Private endpoint |
| Lambda URL | authorization_type=NONE | High | IAM auth or API GW |
| API Gateway | No auth | High | Cognito/IAM/Lambda auth |
| Cognito | No MFA | High | MFA required |
| CloudFront | No WAF | Medium | Attach WAF |
| ALB | No WAF | Medium | Attach WAF |
| S3 | Public access | Critical | Block public access |
| ElastiCache | No auth | High | AUTH token + TLS |
| SSM Parameter | String type | High | SecureString + KMS |

---

*Part 088 ครอบคลุม Public Exposure Vulnerabilities ทั้งหมด - ต่อไปใน Part 089 จะเจาะลึก Compliance Frameworks*
