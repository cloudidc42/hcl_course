# Part 046: AWS Route53 DNS
## การจัดการ DNS และ Domain ด้วย Terraform (Steps 451-460)

---

## บทนำ (Introduction)

AWS Route53 เป็น DNS service ที่รองรับทั้ง public และ private hosted zones
พร้อม routing policies หลายรูปแบบสำหรับ high availability และ global traffic management

**หัวข้อที่จะเรียนรู้:**
- Public และ Private Hosted Zones
- Record Types ต่างๆ (A, AAAA, CNAME, MX, TXT, etc.)
- Alias Records สำหรับ AWS resources
- Routing Policies (Simple, Weighted, Latency, Failover, Geolocation)
- Health Checks
- Route53 Resolver
- ACM Certificate Validation
- Multi-Region DNS Patterns

---

## Step 451: Hosted Zones

### Public Hosted Zone

```hcl
# ✅ Public Hosted Zone
resource "aws_route53_zone" "main" {
  name    = var.domain_name
  comment = "Public hosted zone for ${var.domain_name}"

  tags = {
    Name        = var.domain_name
    Environment = var.environment
    ManagedBy   = "terraform"
  }
}

# ✅ Subdomain Zone
resource "aws_route53_zone" "api" {
  name    = "api.${var.domain_name}"
  comment = "API subdomain zone"

  tags = {
    Name      = "api.${var.domain_name}"
    ManagedBy = "terraform"
  }
}

# ✅ Delegate Subdomain ด้วย NS records
resource "aws_route53_record" "api_ns" {
  zone_id = aws_route53_zone.main.zone_id
  name    = "api.${var.domain_name}"
  type    = "NS"
  ttl     = "30"

  records = aws_route53_zone.api.name_servers
}
```

### Private Hosted Zone

```hcl
# ✅ Private Hosted Zone (สำหรับ internal DNS)
resource "aws_route53_zone" "private" {
  name    = "${var.project_name}.internal"
  comment = "Private zone for internal services"

  # ✅ Associate กับ VPC
  vpc {
    vpc_id = aws_vpc.main.id
  }

  tags = {
    Name      = "${var.project_name}.internal"
    ManagedBy = "terraform"
  }

  # ✅ อย่า delete zone ถ้ายังมี records
  lifecycle {
    ignore_changes = [vpc]
  }
}

# ✅ Private Zone - Associate กับ multiple VPCs
resource "aws_route53_zone_association" "private_secondary_vpc" {
  zone_id = aws_route53_zone.private.zone_id
  vpc_id  = aws_vpc.secondary.id
}

# ✅ Private DNS Records
resource "aws_route53_record" "internal_api" {
  zone_id = aws_route53_zone.private.zone_id
  name    = "api.${aws_route53_zone.private.name}"
  type    = "A"
  ttl     = "300"

  records = [aws_instance.api_server.private_ip]
}

resource "aws_route53_record" "internal_db" {
  zone_id = aws_route53_zone.private.zone_id
  name    = "db.${aws_route53_zone.private.name}"
  type    = "CNAME"
  ttl     = "300"

  records = [aws_db_instance.main.address]
}
```

---

## Step 452: A Records และ AAAA Records

```hcl
# ✅ A Record - IPv4
resource "aws_route53_record" "app_a_record" {
  zone_id = aws_route53_zone.main.zone_id
  name    = "app.${var.domain_name}"
  type    = "A"
  ttl     = "300"

  records = ["203.0.113.1", "203.0.113.2"]  # Multiple IPs = round robin
}

# ✅ AAAA Record - IPv6
resource "aws_route53_record" "app_aaaa_record" {
  zone_id = aws_route53_zone.main.zone_id
  name    = "app.${var.domain_name}"
  type    = "AAAA"
  ttl     = "300"

  records = ["2001:db8::1", "2001:db8::2"]
}

# ✅ A Record Alias (ไม่มี TTL) - สำหรับ AWS resources
resource "aws_route53_record" "app_alias" {
  zone_id = aws_route53_zone.main.zone_id
  name    = var.domain_name  # Apex domain
  type    = "A"

  alias {
    name                   = aws_lb.app.dns_name
    zone_id                = aws_lb.app.zone_id
    evaluate_target_health = true  # ✅ Health check integration
  }
}

# ✅ Alias ชี้ไป CloudFront
resource "aws_route53_record" "cdn_alias" {
  zone_id = aws_route53_zone.main.zone_id
  name    = "cdn.${var.domain_name}"
  type    = "A"

  alias {
    name                   = aws_cloudfront_distribution.main.domain_name
    zone_id                = aws_cloudfront_distribution.main.hosted_zone_id
    evaluate_target_health = false  # CloudFront ไม่รองรับ
  }
}

# ✅ Alias ชี้ไป S3 Website
resource "aws_route53_record" "s3_website" {
  zone_id = aws_route53_zone.main.zone_id
  name    = "static.${var.domain_name}"
  type    = "A"

  alias {
    name                   = aws_s3_bucket_website_configuration.main.website_domain
    zone_id                = aws_s3_bucket.website.hosted_zone_id
    evaluate_target_health = false
  }
}

# ✅ Alias ชี้ไป API Gateway
resource "aws_route53_record" "api_gateway" {
  zone_id = aws_route53_zone.main.zone_id
  name    = "api.${var.domain_name}"
  type    = "A"

  alias {
    name                   = aws_api_gateway_domain_name.main.cloudfront_domain_name
    zone_id                = aws_api_gateway_domain_name.main.cloudfront_zone_id
    evaluate_target_health = false
  }
}
```

---

## Step 453: CNAME, MX, TXT Records

### CNAME Records

```hcl
# ✅ CNAME Record
resource "aws_route53_record" "www_cname" {
  zone_id = aws_route53_zone.main.zone_id
  name    = "www.${var.domain_name}"
  type    = "CNAME"
  ttl     = "300"

  records = [var.domain_name]
}

# ✅ CNAME สำหรับ third-party services
resource "aws_route53_record" "mailchimp_tracking" {
  zone_id = aws_route53_zone.main.zone_id
  name    = "mail.${var.domain_name}"
  type    = "CNAME"
  ttl     = "300"

  records = ["mailchimp.example.com"]
}
```

### MX Records (Email)

```hcl
# ✅ MX Records สำหรับ Email
resource "aws_route53_record" "mx" {
  zone_id = aws_route53_zone.main.zone_id
  name    = var.domain_name
  type    = "MX"
  ttl     = "300"

  # Format: priority mailserver
  records = [
    "10 mail1.example.com.",
    "20 mail2.example.com.",
    "30 mail3.example.com.",
  ]
}

# ✅ Google Workspace MX Records
resource "aws_route53_record" "google_workspace_mx" {
  zone_id = aws_route53_zone.main.zone_id
  name    = var.domain_name
  type    = "MX"
  ttl     = "300"

  records = [
    "1 ASPMX.L.GOOGLE.COM.",
    "5 ALT1.ASPMX.L.GOOGLE.COM.",
    "5 ALT2.ASPMX.L.GOOGLE.COM.",
    "10 ALT3.ASPMX.L.GOOGLE.COM.",
    "10 ALT4.ASPMX.L.GOOGLE.COM.",
  ]
}
```

### TXT Records

```hcl
# ✅ TXT Records สำหรับ Domain Verification
resource "aws_route53_record" "spf" {
  zone_id = aws_route53_zone.main.zone_id
  name    = var.domain_name
  type    = "TXT"
  ttl     = "300"

  records = [
    "v=spf1 include:_spf.google.com ~all",
  ]
}

# ✅ DKIM Record
resource "aws_route53_record" "dkim" {
  zone_id = aws_route53_zone.main.zone_id
  name    = "google._domainkey.${var.domain_name}"
  type    = "TXT"
  ttl     = "300"

  records = [
    "v=DKIM1; k=rsa; p=MIGfMA0GCSqGSIb3DQEBA...",
  ]
}

# ✅ DMARC Record
resource "aws_route53_record" "dmarc" {
  zone_id = aws_route53_zone.main.zone_id
  name    = "_dmarc.${var.domain_name}"
  type    = "TXT"
  ttl     = "300"

  records = [
    "v=DMARC1; p=quarantine; rua=mailto:dmarc@${var.domain_name}",
  ]
}

# ✅ SES Verification Record
resource "aws_ses_domain_identity" "main" {
  domain = var.domain_name
}

resource "aws_route53_record" "ses_verification" {
  zone_id = aws_route53_zone.main.zone_id
  name    = "_amazonses.${var.domain_name}"
  type    = "TXT"
  ttl     = "600"

  records = [aws_ses_domain_identity.main.verification_token]
}
```

---

## Step 454: Routing Policies

### Weighted Routing

```hcl
# ✅ Weighted Routing - Blue/Green Deployment
# Blue environment - 90% traffic
resource "aws_route53_record" "app_blue" {
  zone_id = aws_route53_zone.main.zone_id
  name    = "app.${var.domain_name}"
  type    = "A"

  weighted_routing_policy {
    weight = 90  # 90% traffic
  }

  set_identifier = "blue"

  alias {
    name                   = aws_lb.blue.dns_name
    zone_id                = aws_lb.blue.zone_id
    evaluate_target_health = true
  }
}

# Green environment - 10% traffic
resource "aws_route53_record" "app_green" {
  zone_id = aws_route53_zone.main.zone_id
  name    = "app.${var.domain_name}"
  type    = "A"

  weighted_routing_policy {
    weight = 10  # 10% traffic (testing new version)
  }

  set_identifier = "green"

  alias {
    name                   = aws_lb.green.dns_name
    zone_id                = aws_lb.green.zone_id
    evaluate_target_health = true
  }
}
```

### Latency Routing

```hcl
# ✅ Latency Routing - Route to nearest region
resource "aws_route53_record" "app_ap_southeast_1" {
  zone_id = aws_route53_zone.main.zone_id
  name    = "api.${var.domain_name}"
  type    = "A"

  latency_routing_policy {
    region = "ap-southeast-1"  # Singapore
  }

  set_identifier = "ap-southeast-1"

  alias {
    name                   = aws_lb.app_sg.dns_name
    zone_id                = aws_lb.app_sg.zone_id
    evaluate_target_health = true
  }
}

resource "aws_route53_record" "app_us_east_1" {
  zone_id = aws_route53_zone.main.zone_id
  name    = "api.${var.domain_name}"
  type    = "A"

  latency_routing_policy {
    region = "us-east-1"  # Virginia
  }

  set_identifier = "us-east-1"

  alias {
    name                   = aws_lb.app_us.dns_name
    zone_id                = aws_lb.app_us.zone_id
    evaluate_target_health = true
  }
}

resource "aws_route53_record" "app_eu_west_1" {
  zone_id = aws_route53_zone.main.zone_id
  name    = "api.${var.domain_name}"
  type    = "A"

  latency_routing_policy {
    region = "eu-west-1"  # Ireland
  }

  set_identifier = "eu-west-1"

  alias {
    name                   = aws_lb.app_eu.dns_name
    zone_id                = aws_lb.app_eu.zone_id
    evaluate_target_health = true
  }
}
```

### Failover Routing

```hcl
# ✅ Health Check สำหรับ Primary
resource "aws_route53_health_check" "primary" {
  fqdn              = "primary.${var.domain_name}"
  port              = 443
  type              = "HTTPS"
  resource_path     = "/health"
  failure_threshold = 3
  request_interval  = 30

  tags = {
    Name      = "${var.project_name}-primary-health-check"
    ManagedBy = "terraform"
  }
}

# ✅ Primary Record
resource "aws_route53_record" "app_primary" {
  zone_id = aws_route53_zone.main.zone_id
  name    = "app.${var.domain_name}"
  type    = "A"

  failover_routing_policy {
    type = "PRIMARY"  # Primary endpoint
  }

  set_identifier  = "primary"
  health_check_id = aws_route53_health_check.primary.id

  alias {
    name                   = aws_lb.primary.dns_name
    zone_id                = aws_lb.primary.zone_id
    evaluate_target_health = true
  }
}

# ✅ Secondary (Failover) Record
resource "aws_route53_record" "app_secondary" {
  zone_id = aws_route53_zone.main.zone_id
  name    = "app.${var.domain_name}"
  type    = "A"

  failover_routing_policy {
    type = "SECONDARY"  # Failover endpoint
  }

  set_identifier = "secondary"
  # ✅ Secondary ไม่ต้องมี health check (ใช้เมื่อ primary fail)

  alias {
    name                   = aws_lb.secondary.dns_name
    zone_id                = aws_lb.secondary.zone_id
    evaluate_target_health = true
  }
}
```

### Geolocation Routing

```hcl
# ✅ Geolocation Routing - Route ตาม location ของ user
# Users ใน Asia -> Singapore
resource "aws_route53_record" "app_asia" {
  zone_id = aws_route53_zone.main.zone_id
  name    = "app.${var.domain_name}"
  type    = "A"

  geolocation_routing_policy {
    continent = "AS"  # Asia
  }

  set_identifier = "asia"

  alias {
    name                   = aws_lb.app_sg.dns_name
    zone_id                = aws_lb.app_sg.zone_id
    evaluate_target_health = true
  }
}

# Users ใน Europe -> Ireland
resource "aws_route53_record" "app_europe" {
  zone_id = aws_route53_zone.main.zone_id
  name    = "app.${var.domain_name}"
  type    = "A"

  geolocation_routing_policy {
    continent = "EU"  # Europe
  }

  set_identifier = "europe"

  alias {
    name                   = aws_lb.app_eu.dns_name
    zone_id                = aws_lb.app_eu.zone_id
    evaluate_target_health = true
  }
}

# Thailand เฉพาะ -> Bangkok region
resource "aws_route53_record" "app_thailand" {
  zone_id = aws_route53_zone.main.zone_id
  name    = "app.${var.domain_name}"
  type    = "A"

  geolocation_routing_policy {
    country = "TH"  # Thailand
  }

  set_identifier = "thailand"

  alias {
    name                   = aws_lb.app_bkk.dns_name
    zone_id                = aws_lb.app_bkk.zone_id
    evaluate_target_health = true
  }
}

# Default (ทุก location อื่น)
resource "aws_route53_record" "app_default" {
  zone_id = aws_route53_zone.main.zone_id
  name    = "app.${var.domain_name}"
  type    = "A"

  geolocation_routing_policy {
    country = "*"  # Default
  }

  set_identifier = "default"

  alias {
    name                   = aws_lb.app_us.dns_name
    zone_id                = aws_lb.app_us.zone_id
    evaluate_target_health = true
  }
}
```

---

## Step 455: Health Checks

### aws_route53_health_check

```hcl
# ✅ HTTP Health Check
resource "aws_route53_health_check" "http" {
  fqdn              = var.domain_name
  port              = 80
  type              = "HTTP"
  resource_path     = "/health"
  failure_threshold = 3
  request_interval  = 30

  tags = {
    Name      = "${var.project_name}-http-health"
    ManagedBy = "terraform"
  }
}

# ✅ HTTPS Health Check
resource "aws_route53_health_check" "https" {
  fqdn              = var.domain_name
  port              = 443
  type              = "HTTPS"
  resource_path     = "/health"
  failure_threshold = 3
  request_interval  = 30

  # ✅ Enable SNI สำหรับ HTTPS
  enable_sni = true

  # ✅ String matching ใน response
  search_string = "\"status\":\"healthy\""

  tags = {
    Name      = "${var.project_name}-https-health"
    ManagedBy = "terraform"
  }
}

# ✅ Calculated Health Check (Aggregate)
resource "aws_route53_health_check" "calculated" {
  type                   = "CALCULATED"
  child_healthchecks     = [
    aws_route53_health_check.http.id,
    aws_route53_health_check.https.id,
  ]
  child_health_threshold = 1  # ต้องมีอย่างน้อย 1 healthy

  tags = {
    Name      = "${var.project_name}-calculated-health"
    ManagedBy = "terraform"
  }
}

# ✅ CloudWatch Alarm Health Check
resource "aws_cloudwatch_metric_alarm" "app_health" {
  alarm_name          = "${var.project_name}-app-health"
  comparison_operator = "LessThanThreshold"
  evaluation_periods  = 2
  metric_name         = "HealthyHostCount"
  namespace           = "AWS/ApplicationELB"
  period              = 60
  statistic           = "Minimum"
  threshold           = 1
  alarm_description   = "No healthy hosts in target group"

  dimensions = {
    LoadBalancer = aws_lb.app.arn_suffix
    TargetGroup  = aws_lb_target_group.app.arn_suffix
  }
}

resource "aws_route53_health_check" "cloudwatch" {
  type                            = "CLOUDWATCH_METRIC"
  cloudwatch_alarm_name           = aws_cloudwatch_metric_alarm.app_health.alarm_name
  cloudwatch_alarm_region         = var.aws_region
  insufficient_data_health_status = "Unhealthy"

  tags = {
    Name      = "${var.project_name}-cloudwatch-health"
    ManagedBy = "terraform"
  }
}
```

---

## Step 456: ACM Certificate และ DNS Validation

### aws_acm_certificate

```hcl
# ✅ ACM Certificate สำหรับ ALB/CloudFront
resource "aws_acm_certificate" "main" {
  domain_name       = var.domain_name
  validation_method = "DNS"

  subject_alternative_names = [
    "*.${var.domain_name}",    # Wildcard
    "api.${var.domain_name}",  # Specific subdomain
    "www.${var.domain_name}",
  ]

  lifecycle {
    create_before_destroy = true
  }

  tags = {
    Name        = "${var.project_name}-certificate"
    Environment = var.environment
    ManagedBy   = "terraform"
  }
}

# ✅ DNS Validation Records - ใช้ for_each
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

# ✅ รอให้ certificate validated
resource "aws_acm_certificate_validation" "main" {
  certificate_arn         = aws_acm_certificate.main.arn
  validation_record_fqdns = [for record in aws_route53_record.cert_validation : record.fqdn]

  # ✅ อาจใช้เวลาสักครู่
  timeouts {
    create = "10m"
  }
}

# ✅ ACM Certificate สำหรับ CloudFront (us-east-1)
resource "aws_acm_certificate" "cloudfront" {
  provider          = aws.us_east_1
  domain_name       = var.domain_name
  validation_method = "DNS"

  subject_alternative_names = ["*.${var.domain_name}"]

  lifecycle {
    create_before_destroy = true
  }

  tags = {
    Name      = "${var.project_name}-cloudfront-cert"
    ManagedBy = "terraform"
  }
}

resource "aws_route53_record" "cloudfront_cert_validation" {
  provider = aws.us_east_1

  for_each = {
    for dvo in aws_acm_certificate.cloudfront.domain_validation_options :
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

resource "aws_acm_certificate_validation" "cloudfront" {
  provider                = aws.us_east_1
  certificate_arn         = aws_acm_certificate.cloudfront.arn
  validation_record_fqdns = [for record in aws_route53_record.cloudfront_cert_validation : record.fqdn]
}
```

---

## Step 457: Route53 Resolver

```hcl
# ✅ Route53 Resolver สำหรับ DNS resolution ระหว่าง VPC และ on-premises

# Inbound Endpoint (รับ DNS queries จาก on-premises)
resource "aws_route53_resolver_endpoint" "inbound" {
  name      = "${var.project_name}-inbound"
  direction = "INBOUND"

  security_group_ids = [aws_security_group.resolver.id]

  ip_address {
    subnet_id = aws_subnet.private_1a.id
    ip        = "10.0.1.10"  # ✅ กำหนด IP ชัดเจน
  }

  ip_address {
    subnet_id = aws_subnet.private_1b.id
    ip        = "10.0.2.10"
  }

  tags = {
    Name      = "${var.project_name}-resolver-inbound"
    ManagedBy = "terraform"
  }
}

# ✅ Outbound Endpoint (ส่ง DNS queries ไป on-premises)
resource "aws_route53_resolver_endpoint" "outbound" {
  name      = "${var.project_name}-outbound"
  direction = "OUTBOUND"

  security_group_ids = [aws_security_group.resolver.id]

  ip_address {
    subnet_id = aws_subnet.private_1a.id
  }

  ip_address {
    subnet_id = aws_subnet.private_1b.id
  }

  tags = {
    Name      = "${var.project_name}-resolver-outbound"
    ManagedBy = "terraform"
  }
}

# ✅ Resolver Rule สำหรับ forward DNS queries ไป on-premises
resource "aws_route53_resolver_rule" "onprem" {
  domain_name          = "corp.example.internal"  # Internal domain
  name                 = "forward-to-onprem"
  rule_type            = "FORWARD"
  resolver_endpoint_id = aws_route53_resolver_endpoint.outbound.id

  target_ip {
    ip   = "10.100.0.2"  # On-premises DNS server IP
    port = 53
  }

  target_ip {
    ip   = "10.100.0.3"  # Secondary DNS server
    port = 53
  }

  tags = {
    Name      = "${var.project_name}-onprem-resolver-rule"
    ManagedBy = "terraform"
  }
}

# ✅ Associate Resolver Rule กับ VPC
resource "aws_route53_resolver_rule_association" "onprem" {
  resolver_rule_id = aws_route53_resolver_rule.onprem.id
  vpc_id           = aws_vpc.main.id
}

# ✅ Security Group สำหรับ Resolver
resource "aws_security_group" "resolver" {
  name        = "${var.project_name}-resolver-sg"
  description = "Security group for Route53 Resolver endpoints"
  vpc_id      = aws_vpc.main.id

  ingress {
    from_port   = 53
    to_port     = 53
    protocol    = "tcp"
    cidr_blocks = [var.vpc_cidr, var.onprem_cidr]
  }

  ingress {
    from_port   = 53
    to_port     = 53
    protocol    = "udp"
    cidr_blocks = [var.vpc_cidr, var.onprem_cidr]
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = {
    Name      = "${var.project_name}-resolver-sg"
    ManagedBy = "terraform"
  }
}
```

---

## Step 458: Multi-Region DNS Pattern

### Active-Active Pattern

```hcl
# ✅ Active-Active: ทั้งสอง region active พร้อมกัน ใช้ Latency routing
locals {
  regions = {
    ap_southeast_1 = {
      lb_dns_name = aws_lb.app_sg.dns_name
      lb_zone_id  = aws_lb.app_sg.zone_id
    }
    us_east_1 = {
      lb_dns_name = aws_lb.app_us.dns_name
      lb_zone_id  = aws_lb.app_us.zone_id
    }
    eu_west_1 = {
      lb_dns_name = aws_lb.app_eu.dns_name
      lb_zone_id  = aws_lb.app_eu.zone_id
    }
  }
}

# Health checks สำหรับแต่ละ region
resource "aws_route53_health_check" "regional" {
  for_each = local.regions

  fqdn              = "${each.key}.${var.domain_name}"
  port              = 443
  type              = "HTTPS"
  resource_path     = "/health"
  failure_threshold = 3
  request_interval  = 30

  tags = {
    Name   = "${var.project_name}-health-${each.key}"
    Region = each.key
  }
}

# Latency records สำหรับแต่ละ region
resource "aws_route53_record" "app_regional" {
  for_each = {
    ap-southeast-1 = {
      lb_dns_name = aws_lb.app_sg.dns_name
      lb_zone_id  = aws_lb.app_sg.zone_id
    }
    us-east-1 = {
      lb_dns_name = aws_lb.app_us.dns_name
      lb_zone_id  = aws_lb.app_us.zone_id
    }
    eu-west-1 = {
      lb_dns_name = aws_lb.app_eu.dns_name
      lb_zone_id  = aws_lb.app_eu.zone_id
    }
  }

  zone_id = aws_route53_zone.main.zone_id
  name    = "app.${var.domain_name}"
  type    = "A"

  latency_routing_policy {
    region = each.key
  }

  set_identifier = each.key

  alias {
    name                   = each.value.lb_dns_name
    zone_id                = each.value.lb_zone_id
    evaluate_target_health = true
  }
}
```

### Active-Passive Pattern

```hcl
# ✅ Active-Passive: Primary ใน Asia, Failover ไป US ถ้า fail
resource "aws_route53_health_check" "primary_region" {
  fqdn              = "primary.${var.domain_name}"
  port              = 443
  type              = "HTTPS"
  resource_path     = "/health"
  failure_threshold = 2
  request_interval  = 10  # ✅ Fast failover - check ทุก 10 วินาที
  measure_latency   = true

  tags = {
    Name      = "${var.project_name}-primary-health"
    ManagedBy = "terraform"
  }
}

resource "aws_route53_record" "app_active" {
  zone_id = aws_route53_zone.main.zone_id
  name    = "app.${var.domain_name}"
  type    = "A"

  failover_routing_policy {
    type = "PRIMARY"
  }

  set_identifier  = "primary-asia"
  health_check_id = aws_route53_health_check.primary_region.id

  alias {
    name                   = aws_lb.app_sg.dns_name
    zone_id                = aws_lb.app_sg.zone_id
    evaluate_target_health = true
  }
}

resource "aws_route53_record" "app_passive" {
  zone_id = aws_route53_zone.main.zone_id
  name    = "app.${var.domain_name}"
  type    = "A"

  failover_routing_policy {
    type = "SECONDARY"
  }

  set_identifier = "secondary-us"

  alias {
    name                   = aws_lb.app_us.dns_name
    zone_id                = aws_lb.app_us.zone_id
    evaluate_target_health = true
  }
}
```

---

## Step 459: Complete DNS Setup

```hcl
# ✅ Complete DNS Configuration สำหรับ Production Application

# Data source สำหรับ existing zone
data "aws_route53_zone" "production" {
  name         = var.domain_name
  private_zone = false
}

# ✅ Application Load Balancer Records
resource "aws_route53_record" "app_root" {
  zone_id = data.aws_route53_zone.production.zone_id
  name    = var.domain_name
  type    = "A"

  alias {
    name                   = aws_lb.app.dns_name
    zone_id                = aws_lb.app.zone_id
    evaluate_target_health = true
  }
}

resource "aws_route53_record" "app_www" {
  zone_id = data.aws_route53_zone.production.zone_id
  name    = "www.${var.domain_name}"
  type    = "A"

  alias {
    name                   = aws_lb.app.dns_name
    zone_id                = aws_lb.app.zone_id
    evaluate_target_health = true
  }
}

# ✅ API Subdomain
resource "aws_route53_record" "api" {
  zone_id = data.aws_route53_zone.production.zone_id
  name    = "api.${var.domain_name}"
  type    = "A"

  alias {
    name                   = aws_lb.api.dns_name
    zone_id                = aws_lb.api.zone_id
    evaluate_target_health = true
  }
}

# ✅ CDN Subdomain -> CloudFront
resource "aws_route53_record" "cdn" {
  zone_id = data.aws_route53_zone.production.zone_id
  name    = "cdn.${var.domain_name}"
  type    = "A"

  alias {
    name                   = aws_cloudfront_distribution.assets.domain_name
    zone_id                = aws_cloudfront_distribution.assets.hosted_zone_id
    evaluate_target_health = false
  }
}

# ✅ Email Records
resource "aws_route53_record" "email_mx" {
  zone_id = data.aws_route53_zone.production.zone_id
  name    = var.domain_name
  type    = "MX"
  ttl     = "300"

  records = [
    "10 inbound-smtp.${var.aws_region}.amazonaws.com",
  ]
}

# ✅ SPF สำหรับ SES
resource "aws_route53_record" "ses_spf" {
  zone_id = data.aws_route53_zone.production.zone_id
  name    = var.domain_name
  type    = "TXT"
  ttl     = "300"

  records = [
    "v=spf1 include:amazonses.com ~all",
  ]
}
```

---

## Step 460: Route53 Outputs

```hcl
# ✅ Outputs
output "zone_id" {
  description = "Route53 Zone ID"
  value       = aws_route53_zone.main.zone_id
}

output "zone_name_servers" {
  description = "Name servers สำหรับ delegate DNS"
  value       = aws_route53_zone.main.name_servers
}

output "certificate_arn" {
  description = "ACM Certificate ARN"
  value       = aws_acm_certificate_validation.main.certificate_arn
}

output "cloudfront_certificate_arn" {
  description = "ACM Certificate ARN สำหรับ CloudFront (us-east-1)"
  value       = aws_acm_certificate_validation.cloudfront.certificate_arn
}

output "website_url" {
  description = "Website URL"
  value       = "https://${var.domain_name}"
}
```

---

## DNS Best Practices สรุป

### ✅ สิ่งที่ควรทำ

1. **Alias Records**: ใช้ Alias แทน CNAME สำหรับ AWS resources (apex domain support)
2. **Health Checks**: ใส่ health checks กับ routing policies ทุกอัน
3. **evaluate_target_health**: เปิดสำหรับ ELB aliases
4. **Low TTL**: ระหว่าง deployment/migration ลด TTL ก่อน
5. **Private Zones**: ใช้สำหรับ internal services
6. **DNSSEC**: เปิดสำหรับ critical domains

### ❌ สิ่งที่ไม่ควรทำ

1. ❌ CNAME สำหรับ apex domain (ใช้ Alias แทน)
2. ❌ High TTL ระหว่าง migration
3. ❌ ไม่มี health checks กับ failover routing
4. ❌ ลืม create_before_destroy สำหรับ certificates

### Route53 Record Types สรุป

| Type | Use Case | หมายเหตุ |
|------|---------|---------|
| A | IPv4 address / Alias | Alias ไม่มี TTL |
| AAAA | IPv6 address | |
| CNAME | Canonical name | ไม่ใช้ที่ apex domain |
| MX | Mail server | มี priority |
| TXT | Text / verification | SPF, DKIM, DMARC |
| NS | Name servers | |
| SOA | Start of authority | Auto-created |
| SRV | Service records | |
| CAA | Certificate authority | Security |

---

**Next Steps**: ไปต่อที่ Part 047 - AWS ElastiCache & Database Caching
