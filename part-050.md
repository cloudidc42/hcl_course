# Part 050: AWS Load Balancers (ALB/NLB/CLB)
## การจัดการ Load Balancers ด้วย Terraform (Steps 491-500)

---

## บทนำ (Introduction)

AWS Load Balancers ช่วยกระจาย traffic ไปยัง multiple targets เพื่อเพิ่ม availability และ scalability
Terraform รองรับ ALB (Application), NLB (Network), และ CLB (Classic) ด้วย `aws_lb` resource

**หัวข้อที่จะเรียนรู้:**
- Application Load Balancer (ALB) - Layer 7
- Network Load Balancer (NLB) - Layer 4
- Classic Load Balancer (CLB) - Legacy
- Listeners และ Rules
- Target Groups และ Health Checks
- HTTPS/SSL Configuration
- WAF Integration
- Access Logs
- Auto Scaling Integration

---

## Step 491: Application Load Balancer (ALB)

### aws_lb

```hcl
# ✅ Security Group สำหรับ ALB
resource "aws_security_group" "alb" {
  name        = "${var.project_name}-alb-sg"
  description = "Security group for Application Load Balancer"
  vpc_id      = aws_vpc.main.id

  # ✅ อนุญาต HTTP/HTTPS จาก internet
  ingress {
    description = "HTTP from internet"
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
    ipv6_cidr_blocks = ["::/0"]
  }

  ingress {
    description = "HTTPS from internet"
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
    ipv6_cidr_blocks = ["::/0"]
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
    description = "Allow all outbound"
  }

  tags = {
    Name      = "${var.project_name}-alb-sg"
    ManagedBy = "terraform"
  }
}

# ✅ S3 Bucket สำหรับ ALB Access Logs
resource "aws_s3_bucket" "alb_logs" {
  bucket = "${var.project_name}-alb-logs-${data.aws_caller_identity.current.account_id}"

  tags = {
    Name      = "${var.project_name}-alb-logs"
    ManagedBy = "terraform"
  }
}

resource "aws_s3_bucket_lifecycle_configuration" "alb_logs" {
  bucket = aws_s3_bucket.alb_logs.id

  rule {
    id     = "expire-logs"
    status = "Enabled"

    expiration {
      days = 90
    }
  }
}

# ✅ S3 Bucket Policy สำหรับ ELB Logs (จำเป็นต้องมี)
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
        Resource = "${aws_s3_bucket.alb_logs.arn}/alb/AWSLogs/${data.aws_caller_identity.current.account_id}/*"
      },
      {
        Effect = "Allow"
        Principal = {
          Service = "delivery.logs.amazonaws.com"
        }
        Action   = "s3:PutObject"
        Resource = "${aws_s3_bucket.alb_logs.arn}/alb/AWSLogs/${data.aws_caller_identity.current.account_id}/*"
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

# ✅ Application Load Balancer
resource "aws_lb" "app" {
  name               = "${var.project_name}-alb"
  internal           = false  # Public-facing ALB
  load_balancer_type = "application"

  security_groups = [aws_security_group.alb.id]
  subnets         = aws_subnet.public[*].id  # ✅ ALB อยู่ใน public subnets

  # ✅ Deletion protection
  enable_deletion_protection = true

  # ✅ Cross-zone load balancing (default เปิดอยู่แล้วสำหรับ ALB)
  enable_cross_zone_load_balancing = true

  # ✅ HTTP/2 support
  enable_http2 = true

  # ✅ WAF desync mitigation
  desync_mitigation_mode = "defensive"

  # ✅ Drop invalid headers
  drop_invalid_header_fields = true

  # ✅ Access logs
  access_logs {
    bucket  = aws_s3_bucket.alb_logs.id
    prefix  = "alb"
    enabled = true
  }

  # ✅ Connection draining
  idle_timeout = 60

  tags = {
    Name        = "${var.project_name}-alb"
    Environment = var.environment
    ManagedBy   = "terraform"
  }
}

# ❌ Insecure ALB
resource "aws_lb" "insecure" {
  name               = "insecure-alb"
  internal           = false
  load_balancer_type = "application"
  security_groups    = [aws_security_group.alb.id]
  subnets            = aws_subnet.public[*].id

  # ❌ ไม่มี deletion protection
  enable_deletion_protection = false
  # ❌ ไม่มี access logs
  # ❌ ไม่มี drop invalid headers
}
```

---

## Step 492: ALB Target Groups

### aws_lb_target_group

```hcl
# ✅ Target Group สำหรับ ECS Fargate
resource "aws_lb_target_group" "app" {
  name        = "${var.project_name}-app-tg"
  port        = 8080
  protocol    = "HTTP"
  vpc_id      = aws_vpc.main.id
  target_type = "ip"  # ✅ "ip" สำหรับ Fargate, "instance" สำหรับ EC2

  # ✅ Health check configuration
  health_check {
    enabled             = true
    healthy_threshold   = 2
    unhealthy_threshold = 3
    interval            = 30
    path                = "/health"
    port                = "traffic-port"
    protocol            = "HTTP"
    timeout             = 10
    matcher             = "200"
  }

  # ✅ Deregistration delay
  deregistration_delay = 30  # 30 seconds (default 300)

  # ✅ Stickiness สำหรับ session-based apps
  stickiness {
    type            = "lb_cookie"
    cookie_duration = 86400  # 1 day
    enabled         = false  # ปิด default (stateless apps)
  }

  # ✅ Load balancing algorithm
  load_balancing_algorithm_type = "least_outstanding_requests"

  tags = {
    Name      = "${var.project_name}-app-tg"
    ManagedBy = "terraform"
  }

  lifecycle {
    create_before_destroy = true
  }
}

# ✅ Target Group สำหรับ EC2 instances
resource "aws_lb_target_group" "ec2_app" {
  name        = "${var.project_name}-ec2-tg"
  port        = 8080
  protocol    = "HTTP"
  vpc_id      = aws_vpc.main.id
  target_type = "instance"  # ✅ "instance" สำหรับ EC2

  health_check {
    enabled             = true
    healthy_threshold   = 2
    unhealthy_threshold = 3
    interval            = 30
    path                = "/health"
    protocol            = "HTTP"
    timeout             = 10
    matcher             = "200"
  }

  deregistration_delay = 300  # Default 300 seconds

  tags = {
    Name      = "${var.project_name}-ec2-tg"
    ManagedBy = "terraform"
  }
}

# ✅ Target Group Attachment สำหรับ EC2 instances
resource "aws_lb_target_group_attachment" "ec2_app" {
  count            = length(aws_instance.app_servers)
  target_group_arn = aws_lb_target_group.ec2_app.arn
  target_id        = aws_instance.app_servers[count.index].id
  port             = 8080
}

# ✅ Target Group สำหรับ Lambda
resource "aws_lb_target_group" "lambda" {
  name        = "${var.project_name}-lambda-tg"
  target_type = "lambda"
  vpc_id      = aws_vpc.main.id

  health_check {
    enabled  = true
    path     = "/health"
    matcher  = "200"
    interval = 35
    timeout  = 30
  }

  tags = {
    Name      = "${var.project_name}-lambda-tg"
    ManagedBy = "terraform"
  }
}

resource "aws_lambda_permission" "alb" {
  statement_id  = "AllowALBInvoke"
  action        = "lambda:InvokeFunction"
  function_name = aws_lambda_function.api_handler.function_name
  principal     = "elasticloadbalancing.amazonaws.com"
  source_arn    = aws_lb_target_group.lambda.arn
}

resource "aws_lb_target_group_attachment" "lambda" {
  target_group_arn = aws_lb_target_group.lambda.arn
  target_id        = aws_lambda_function.api_handler.arn
  depends_on       = [aws_lambda_permission.alb]
}
```

---

## Step 493: ALB Listeners

### aws_lb_listener

```hcl
# ✅ HTTP Listener - Redirect ไป HTTPS
resource "aws_lb_listener" "http" {
  load_balancer_arn = aws_lb.app.arn
  port              = 80
  protocol          = "HTTP"

  # ✅ Redirect HTTP ไป HTTPS
  default_action {
    type = "redirect"

    redirect {
      port        = "443"
      protocol    = "HTTPS"
      status_code = "HTTP_301"  # Permanent redirect
    }
  }

  tags = {
    Name      = "${var.project_name}-http-listener"
    ManagedBy = "terraform"
  }
}

# ✅ HTTPS Listener
resource "aws_lb_listener" "https" {
  load_balancer_arn = aws_lb.app.arn
  port              = 443
  protocol          = "HTTPS"
  ssl_policy        = "ELBSecurityPolicy-TLS13-1-2-2021-06"  # ✅ Modern TLS policy
  certificate_arn   = aws_acm_certificate_validation.main.certificate_arn

  default_action {
    type             = "forward"
    target_group_arn = aws_lb_target_group.app.arn
  }

  tags = {
    Name      = "${var.project_name}-https-listener"
    ManagedBy = "terraform"
  }
}

# ✅ เพิ่ม SSL Certificates (สำหรับ multiple domains)
resource "aws_lb_listener_certificate" "additional" {
  listener_arn    = aws_lb_listener.https.arn
  certificate_arn = aws_acm_certificate_validation.additional.certificate_arn
}
```

---

## Step 494: ALB Listener Rules

### aws_lb_listener_rule

```hcl
# ✅ Path-based Routing Rules
resource "aws_lb_listener_rule" "api" {
  listener_arn = aws_lb_listener.https.arn
  priority     = 100

  condition {
    path_pattern {
      values = ["/api/*", "/api"]
    }
  }

  action {
    type             = "forward"
    target_group_arn = aws_lb_target_group.api.arn
  }

  tags = {
    Name      = "${var.project_name}-api-rule"
    ManagedBy = "terraform"
  }
}

# ✅ Host-based Routing Rules
resource "aws_lb_listener_rule" "api_subdomain" {
  listener_arn = aws_lb_listener.https.arn
  priority     = 50

  condition {
    host_header {
      values = ["api.${var.domain_name}"]
    }
  }

  action {
    type             = "forward"
    target_group_arn = aws_lb_target_group.api.arn
  }

  tags = {
    Name      = "${var.project_name}-api-host-rule"
    ManagedBy = "terraform"
  }
}

# ✅ Header-based Routing (Canary Deployment)
resource "aws_lb_listener_rule" "canary" {
  listener_arn = aws_lb_listener.https.arn
  priority     = 10

  condition {
    http_header {
      http_header_name = "X-Canary"
      values           = ["true"]
    }
  }

  action {
    type             = "forward"
    target_group_arn = aws_lb_target_group.app_canary.arn
  }

  tags = {
    Name      = "${var.project_name}-canary-rule"
    ManagedBy = "terraform"
  }
}

# ✅ Query String-based Routing
resource "aws_lb_listener_rule" "beta" {
  listener_arn = aws_lb_listener.https.arn
  priority     = 20

  condition {
    query_string {
      key   = "version"
      value = "beta"
    }
  }

  action {
    type             = "forward"
    target_group_arn = aws_lb_target_group.app_beta.arn
  }
}

# ✅ Weighted Target Groups (Canary/Blue-Green)
resource "aws_lb_listener_rule" "weighted" {
  listener_arn = aws_lb_listener.https.arn
  priority     = 90

  condition {
    path_pattern {
      values = ["/*"]
    }
  }

  action {
    type = "forward"

    forward {
      target_group {
        arn    = aws_lb_target_group.app.arn
        weight = 90  # 90% traffic
      }

      target_group {
        arn    = aws_lb_target_group.app_new.arn
        weight = 10  # 10% traffic (canary)
      }

      stickiness {
        enabled  = true
        duration = 3600  # Sticky ใน 1 ชั่วโมง
      }
    }
  }
}

# ✅ Authentication Rules (Cognito)
resource "aws_lb_listener_rule" "authenticated" {
  listener_arn = aws_lb_listener.https.arn
  priority     = 200

  condition {
    path_pattern {
      values = ["/admin/*"]
    }
  }

  action {
    type = "authenticate-cognito"

    authenticate_cognito {
      user_pool_arn             = aws_cognito_user_pool.main.arn
      user_pool_client_id       = aws_cognito_user_pool_client.alb.id
      user_pool_domain          = aws_cognito_user_pool_domain.main.domain
      on_unauthenticated_request = "authenticate"
    }
  }

  # ✅ After authentication, forward to target
  action {
    type             = "forward"
    target_group_arn = aws_lb_target_group.admin.arn
  }
}

# ✅ Fixed Response
resource "aws_lb_listener_rule" "health_check" {
  listener_arn = aws_lb_listener.https.arn
  priority     = 1

  condition {
    path_pattern {
      values = ["/ping"]
    }
  }

  action {
    type = "fixed-response"

    fixed_response {
      content_type = "text/plain"
      message_body = "pong"
      status_code  = "200"
    }
  }
}
```

---

## Step 495: WAF Integration

```hcl
# ✅ WAF Web ACL
resource "aws_wafv2_web_acl" "alb" {
  name  = "${var.project_name}-alb-waf"
  scope = "REGIONAL"  # REGIONAL สำหรับ ALB, CLOUDFRONT สำหรับ CloudFront

  default_action {
    allow {}
  }

  # ✅ AWS Managed Rules
  rule {
    name     = "AWSManagedRulesCommonRuleSet"
    priority = 1

    override_action {
      none {}  # ใช้ action จาก rule group
    }

    statement {
      managed_rule_group_statement {
        name        = "AWSManagedRulesCommonRuleSet"
        vendor_name = "AWS"
      }
    }

    visibility_config {
      cloudwatch_metrics_enabled = true
      metric_name                = "CommonRuleSet"
      sampled_requests_enabled   = true
    }
  }

  rule {
    name     = "AWSManagedRulesKnownBadInputsRuleSet"
    priority = 2

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

  rule {
    name     = "AWSManagedRulesSQLiRuleSet"
    priority = 3

    override_action {
      none {}
    }

    statement {
      managed_rule_group_statement {
        name        = "AWSManagedRulesSQLiRuleSet"
        vendor_name = "AWS"
      }
    }

    visibility_config {
      cloudwatch_metrics_enabled = true
      metric_name                = "SQLiRuleSet"
      sampled_requests_enabled   = true
    }
  }

  # ✅ Rate Limiting
  rule {
    name     = "RateLimit"
    priority = 10

    action {
      block {}
    }

    statement {
      rate_based_statement {
        limit              = 2000  # 2000 requests per 5 minutes
        aggregate_key_type = "IP"
      }
    }

    visibility_config {
      cloudwatch_metrics_enabled = true
      metric_name                = "RateLimit"
      sampled_requests_enabled   = true
    }
  }

  # ✅ IP Allowlist (สำหรับ internal APIs)
  rule {
    name     = "AllowInternalIPs"
    priority = 0

    action {
      allow {}
    }

    statement {
      ip_set_reference_statement {
        arn = aws_wafv2_ip_set.internal.arn
      }
    }

    visibility_config {
      cloudwatch_metrics_enabled = true
      metric_name                = "InternalIPAllowlist"
      sampled_requests_enabled   = false
    }
  }

  visibility_config {
    cloudwatch_metrics_enabled = true
    metric_name                = "${var.project_name}-alb-waf"
    sampled_requests_enabled   = true
  }

  tags = {
    Name      = "${var.project_name}-alb-waf"
    ManagedBy = "terraform"
  }
}

# ✅ IP Set สำหรับ Allowlist
resource "aws_wafv2_ip_set" "internal" {
  name               = "${var.project_name}-internal-ips"
  scope              = "REGIONAL"
  ip_address_version = "IPV4"

  addresses = var.internal_ip_cidrs

  tags = {
    Name      = "${var.project_name}-internal-ips"
    ManagedBy = "terraform"
  }
}

# ✅ Associate WAF กับ ALB
resource "aws_wafv2_web_acl_association" "alb" {
  resource_arn = aws_lb.app.arn
  web_acl_arn  = aws_wafv2_web_acl.alb.arn
}
```

---

## Step 496: Network Load Balancer (NLB)

```hcl
# ✅ Network Load Balancer
resource "aws_lb" "network" {
  name               = "${var.project_name}-nlb"
  internal           = false
  load_balancer_type = "network"
  subnets            = aws_subnet.public[*].id

  # ✅ Static IPs ด้วย Elastic IP
  # (ถ้าต้องการ static IPs)
  # subnet_mapping {
  #   subnet_id     = aws_subnet.public_1a.id
  #   allocation_id = aws_eip.nlb_1a.id
  # }

  enable_deletion_protection       = true
  enable_cross_zone_load_balancing = true

  # ✅ Access logs สำหรับ NLB
  access_logs {
    bucket  = aws_s3_bucket.alb_logs.id
    prefix  = "nlb"
    enabled = true
  }

  tags = {
    Name      = "${var.project_name}-nlb"
    ManagedBy = "terraform"
  }
}

# ✅ NLB Target Group TCP
resource "aws_lb_target_group" "nlb_tcp" {
  name        = "${var.project_name}-nlb-tcp"
  port        = 443
  protocol    = "TCP"  # TCP, UDP, TLS
  vpc_id      = aws_vpc.main.id
  target_type = "ip"

  health_check {
    enabled             = true
    protocol            = "TCP"
    port                = "traffic-port"
    healthy_threshold   = 3
    unhealthy_threshold = 3
    interval            = 10
  }

  # ✅ Connection termination
  connection_termination = true

  preserve_client_ip = true  # ✅ รักษา client IP

  tags = {
    Name      = "${var.project_name}-nlb-tcp-tg"
    ManagedBy = "terraform"
  }
}

# ✅ NLB TCP Listener
resource "aws_lb_listener" "nlb_tcp" {
  load_balancer_arn = aws_lb.network.arn
  port              = 443
  protocol          = "TCP"

  default_action {
    type             = "forward"
    target_group_arn = aws_lb_target_group.nlb_tcp.arn
  }
}

# ✅ NLB TLS Listener (Terminate TLS at NLB)
resource "aws_lb_listener" "nlb_tls" {
  load_balancer_arn = aws_lb.network.arn
  port              = 443
  protocol          = "TLS"
  certificate_arn   = aws_acm_certificate_validation.main.certificate_arn
  ssl_policy        = "ELBSecurityPolicy-TLS13-1-2-2021-06"

  default_action {
    type             = "forward"
    target_group_arn = aws_lb_target_group.nlb_tcp.arn
  }

  alpn_policy = "HTTP2Preferred"
}

# ✅ UDP สำหรับ applications
resource "aws_lb_listener" "nlb_udp" {
  load_balancer_arn = aws_lb.network.arn
  port              = 514
  protocol          = "UDP"

  default_action {
    type             = "forward"
    target_group_arn = aws_lb_target_group.nlb_udp.arn
  }
}

resource "aws_lb_target_group" "nlb_udp" {
  name        = "${var.project_name}-nlb-udp"
  port        = 514
  protocol    = "UDP"
  vpc_id      = aws_vpc.main.id
  target_type = "instance"

  health_check {
    protocol = "TCP"
    port     = "514"
  }

  tags = {
    Name      = "${var.project_name}-nlb-udp-tg"
    ManagedBy = "terraform"
  }
}
```

---

## Step 497: Classic Load Balancer (CLB) - Legacy

```hcl
# ⚠️ CLB - Legacy (ใช้ ALB หรือ NLB แทน)
# ✅ ยังใช้ได้แต่ AWS แนะนำให้ migrate ไป ALB/NLB

resource "aws_elb" "legacy" {
  name = "${var.project_name}-legacy-elb"

  subnets         = aws_subnet.public[*].id
  security_groups = [aws_security_group.alb.id]

  # ✅ Access logs
  access_logs {
    bucket        = aws_s3_bucket.alb_logs.id
    bucket_prefix = "clb"
    interval      = 60
  }

  listener {
    instance_port     = 8080
    instance_protocol = "HTTP"
    lb_port           = 80
    lb_protocol       = "HTTP"
  }

  listener {
    instance_port      = 8080
    instance_protocol  = "HTTP"
    lb_port            = 443
    lb_protocol        = "HTTPS"
    ssl_certificate_id = aws_acm_certificate_validation.main.certificate_arn
  }

  health_check {
    healthy_threshold   = 2
    unhealthy_threshold = 2
    timeout             = 3
    target              = "HTTP:8080/health"
    interval            = 30
  }

  cross_zone_load_balancing   = true
  idle_timeout                = 400
  connection_draining         = true
  connection_draining_timeout = 400

  instances = aws_instance.app_servers[*].id

  tags = {
    Name      = "${var.project_name}-legacy-elb"
    ManagedBy = "terraform"
  }
}
```

---

## Step 498: Complete ALB Production Setup

```hcl
# ✅ Complete ALB Setup สำหรับ ECS Fargate Application

# 1. Application Load Balancer
resource "aws_lb" "production" {
  name               = "${var.project_name}-prod-alb"
  internal           = false
  load_balancer_type = "application"
  security_groups    = [aws_security_group.alb.id]
  subnets            = aws_subnet.public[*].id

  enable_deletion_protection       = true
  enable_cross_zone_load_balancing = true
  enable_http2                     = true
  drop_invalid_header_fields       = true
  desync_mitigation_mode           = "strictest"

  access_logs {
    bucket  = aws_s3_bucket.alb_logs.id
    prefix  = "production"
    enabled = true
  }

  tags = {
    Name        = "${var.project_name}-prod-alb"
    Environment = "production"
    ManagedBy   = "terraform"
  }
}

# 2. Target Groups
resource "aws_lb_target_group" "production_app" {
  name        = "${var.project_name}-prod-app-tg"
  port        = 8080
  protocol    = "HTTP"
  vpc_id      = aws_vpc.main.id
  target_type = "ip"

  health_check {
    enabled             = true
    healthy_threshold   = 2
    unhealthy_threshold = 3
    interval            = 30
    path                = "/health"
    protocol            = "HTTP"
    timeout             = 10
    matcher             = "200"
  }

  deregistration_delay              = 30
  load_balancing_algorithm_type     = "least_outstanding_requests"

  tags = {
    Name        = "${var.project_name}-prod-app-tg"
    Environment = "production"
    ManagedBy   = "terraform"
  }

  lifecycle {
    create_before_destroy = true
  }
}

resource "aws_lb_target_group" "production_api" {
  name        = "${var.project_name}-prod-api-tg"
  port        = 3000
  protocol    = "HTTP"
  vpc_id      = aws_vpc.main.id
  target_type = "ip"

  health_check {
    enabled             = true
    healthy_threshold   = 2
    unhealthy_threshold = 3
    interval            = 30
    path                = "/api/health"
    protocol            = "HTTP"
    timeout             = 10
    matcher             = "200"
  }

  deregistration_delay = 30

  tags = {
    Name        = "${var.project_name}-prod-api-tg"
    Environment = "production"
    ManagedBy   = "terraform"
  }

  lifecycle {
    create_before_destroy = true
  }
}

# 3. HTTP Redirect Listener
resource "aws_lb_listener" "production_http" {
  load_balancer_arn = aws_lb.production.arn
  port              = 80
  protocol          = "HTTP"

  default_action {
    type = "redirect"
    redirect {
      port        = "443"
      protocol    = "HTTPS"
      status_code = "HTTP_301"
    }
  }

  tags = {
    Name        = "${var.project_name}-prod-http"
    Environment = "production"
    ManagedBy   = "terraform"
  }
}

# 4. HTTPS Main Listener
resource "aws_lb_listener" "production_https" {
  load_balancer_arn = aws_lb.production.arn
  port              = 443
  protocol          = "HTTPS"
  ssl_policy        = "ELBSecurityPolicy-TLS13-1-2-2021-06"
  certificate_arn   = aws_acm_certificate_validation.main.certificate_arn

  default_action {
    type             = "forward"
    target_group_arn = aws_lb_target_group.production_app.arn
  }

  tags = {
    Name        = "${var.project_name}-prod-https"
    Environment = "production"
    ManagedBy   = "terraform"
  }
}

# 5. Listener Rules
resource "aws_lb_listener_rule" "production_api" {
  listener_arn = aws_lb_listener.production_https.arn
  priority     = 100

  condition {
    path_pattern {
      values = ["/api/*"]
    }
  }

  action {
    type             = "forward"
    target_group_arn = aws_lb_target_group.production_api.arn
  }

  tags = {
    Name      = "${var.project_name}-api-rule"
    ManagedBy = "terraform"
  }
}

resource "aws_lb_listener_rule" "production_health" {
  listener_arn = aws_lb_listener.production_https.arn
  priority     = 1

  condition {
    path_pattern {
      values = ["/health", "/ping"]
    }
  }

  action {
    type = "fixed-response"
    fixed_response {
      content_type = "application/json"
      message_body = "{\"status\":\"ok\"}"
      status_code  = "200"
    }
  }
}

# 6. WAF Association
resource "aws_wafv2_web_acl_association" "production" {
  resource_arn = aws_lb.production.arn
  web_acl_arn  = aws_wafv2_web_acl.alb.arn
}
```

---

## Step 499: Internal Load Balancer

```hcl
# ✅ Internal ALB สำหรับ internal services
resource "aws_lb" "internal" {
  name               = "${var.project_name}-internal-alb"
  internal           = true  # ✅ Internal - ไม่ expose สู่ internet
  load_balancer_type = "application"
  security_groups    = [aws_security_group.internal_alb.id]
  subnets            = aws_subnet.private[*].id  # ✅ Private subnets

  enable_deletion_protection = true
  drop_invalid_header_fields = true

  access_logs {
    bucket  = aws_s3_bucket.alb_logs.id
    prefix  = "internal"
    enabled = true
  }

  tags = {
    Name      = "${var.project_name}-internal-alb"
    ManagedBy = "terraform"
  }
}

# ✅ Security Group สำหรับ Internal ALB
resource "aws_security_group" "internal_alb" {
  name        = "${var.project_name}-internal-alb-sg"
  description = "Internal ALB security group"
  vpc_id      = aws_vpc.main.id

  # ✅ อนุญาตเฉพาะจาก VPC
  ingress {
    description = "HTTP from VPC"
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = [aws_vpc.main.cidr_block]
  }

  ingress {
    description = "HTTPS from VPC"
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = [aws_vpc.main.cidr_block]
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = {
    Name      = "${var.project_name}-internal-alb-sg"
    ManagedBy = "terraform"
  }
}
```

---

## Step 500: Outputs และ CloudWatch

```hcl
# ✅ CloudWatch Alarms สำหรับ ALB
resource "aws_cloudwatch_metric_alarm" "alb_5xx" {
  alarm_name          = "${var.project_name}-alb-5xx"
  comparison_operator = "GreaterThanOrEqualToThreshold"
  evaluation_periods  = 2
  metric_name         = "HTTPCode_Target_5XX_Count"
  namespace           = "AWS/ApplicationELB"
  period              = 60
  statistic           = "Sum"
  threshold           = 10
  alarm_description   = "ALB 5XX errors are high"
  alarm_actions       = [aws_sns_topic.alerts.arn]
  treat_missing_data  = "notBreaching"

  dimensions = {
    LoadBalancer = aws_lb.production.arn_suffix
  }

  tags = {
    Name      = "${var.project_name}-alb-5xx-alarm"
    ManagedBy = "terraform"
  }
}

resource "aws_cloudwatch_metric_alarm" "alb_unhealthy_hosts" {
  alarm_name          = "${var.project_name}-alb-unhealthy-hosts"
  comparison_operator = "GreaterThanOrEqualToThreshold"
  evaluation_periods  = 2
  metric_name         = "UnHealthyHostCount"
  namespace           = "AWS/ApplicationELB"
  period              = 60
  statistic           = "Average"
  threshold           = 1
  alarm_description   = "Unhealthy targets in ALB target group"
  alarm_actions       = [aws_sns_topic.alerts.arn]

  dimensions = {
    LoadBalancer = aws_lb.production.arn_suffix
    TargetGroup  = aws_lb_target_group.production_app.arn_suffix
  }
}

resource "aws_cloudwatch_metric_alarm" "alb_response_time" {
  alarm_name          = "${var.project_name}-alb-response-time"
  comparison_operator = "GreaterThanOrEqualToThreshold"
  evaluation_periods  = 3
  metric_name         = "TargetResponseTime"
  namespace           = "AWS/ApplicationELB"
  period              = 60
  extended_statistic  = "p99"
  threshold           = 3  # P99 > 3 seconds
  alarm_description   = "High ALB response time"
  alarm_actions       = [aws_sns_topic.alerts.arn]

  dimensions = {
    LoadBalancer = aws_lb.production.arn_suffix
  }
}

# ✅ Outputs
output "alb_dns_name" {
  description = "ALB DNS name"
  value       = aws_lb.production.dns_name
}

output "alb_zone_id" {
  description = "ALB Hosted Zone ID (สำหรับ Route53 Alias)"
  value       = aws_lb.production.zone_id
}

output "alb_arn" {
  description = "ALB ARN"
  value       = aws_lb.production.arn
}

output "target_group_arn" {
  description = "Main target group ARN"
  value       = aws_lb_target_group.production_app.arn
}

output "https_listener_arn" {
  description = "HTTPS listener ARN"
  value       = aws_lb_listener.production_https.arn
}

output "nlb_dns_name" {
  description = "NLB DNS name"
  value       = aws_lb.network.dns_name
}
```

---

## Load Balancer Comparison

| Feature | ALB | NLB | CLB |
|---------|-----|-----|-----|
| Layer | 7 (HTTP/HTTPS) | 4 (TCP/UDP/TLS) | 4 & 7 |
| Protocols | HTTP, HTTPS, gRPC | TCP, UDP, TLS, TCP_UDP | HTTP, HTTPS, TCP, SSL |
| Target Types | Instance, IP, Lambda | Instance, IP, ALB | Instance |
| Host-based Routing | ✅ | ❌ | ❌ |
| Path-based Routing | ✅ | ❌ | ❌ |
| WebSocket | ✅ | ✅ | ✅ |
| Static IP | ❌ | ✅ | ❌ |
| Ultra-low Latency | ❌ | ✅ | ❌ |
| WAF Integration | ✅ | ❌ | ❌ |
| User Authentication | ✅ | ❌ | ❌ |
| Serverless (Lambda) | ✅ | ❌ | ❌ |

## Load Balancer Best Practices สรุป

### ✅ ALB Best Practices

1. **HTTPS Only**: HTTP listener redirect ไป HTTPS
2. **Modern TLS**: ใช้ `ELBSecurityPolicy-TLS13-1-2-2021-06`
3. **WAF**: เปิดใช้งาน WAF Web ACL
4. **Deletion Protection**: เปิด `enable_deletion_protection = true`
5. **Access Logs**: เปิดและเก็บใน S3
6. **Drop Invalid Headers**: เปิด `drop_invalid_header_fields = true`
7. **Deregistration Delay**: ลดลงสำหรับ faster deployments (30-60s)
8. **Health Checks**: ตั้ง path และ thresholds ที่เหมาะสม

### ✅ NLB Best Practices

1. **Static IPs**: ใช้ Elastic IPs เมื่อต้องการ fixed IPs
2. **Cross-zone**: เปิด cross-zone load balancing
3. **Client IP Preservation**: ใช้ `preserve_client_ip = true`
4. **TLS Termination**: ตัดสินใจว่าจะ terminate ที่ NLB หรือ targets

### ❌ สิ่งที่ไม่ควรทำ

1. ❌ HTTP listener ที่ไม่ redirect ไป HTTPS
2. ❌ ไม่มี health checks
3. ❌ ไม่มี access logs
4. ❌ ไม่มี deletion protection ใน production
5. ❌ Security group ที่อนุญาตทุก ports
6. ❌ ไม่มี WAF protection

---

## สรุป Course Part 041-050

เราได้เรียนรู้ AWS Services ที่สำคัญทั้งหมดแล้ว:

| Part | หัวข้อ |
|------|-------|
| 041 | IAM Roles, Users & Policies |
| 042 | RDS Databases |
| 043 | EKS Kubernetes |
| 044 | Lambda Functions |
| 045 | CloudFront CDN |
| 046 | Route53 DNS |
| 047 | ElastiCache |
| 048 | SQS, SNS & EventBridge |
| 049 | ECR & ECS |
| 050 | Load Balancers |

**Next Steps**: ต่อไปใน Part 051+ จะเรียนรู้เรื่อง Infrastructure Patterns, Multi-Region Deployment, และ Advanced Terraform Techniques
