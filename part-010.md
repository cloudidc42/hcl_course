# Part 010: HCL Template Syntax & Heredocs
## Template Syntax และ Heredocs ใน HCL (Steps 91-100)

---

## Step 91: Heredoc Syntax พื้นฐาน

### Basic Heredoc: <<EOF ... EOF

```hcl
# Basic heredoc syntax
# ข้อความเริ่มต้นหลัง <<EOF และจบก่อน EOF
resource "local_file" "config" {
  filename = "/tmp/config.txt"
  
  content = <<EOF
This is a basic heredoc.
All content between <<EOF and EOF
is treated as a string.
  Leading spaces ARE preserved here.
    Indentation IS part of the content.
EOF
}

# ใช้ IAM policy
resource "aws_iam_policy" "s3_access" {
  name = "s3-access"
  
  policy = <<EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": ["s3:GetObject"],
      "Resource": "arn:aws:s3:::my-bucket/*"
    }
  ]
}
EOF
}

# Heredoc กับ string interpolation
variable "bucket_name" {
  default = "my-bucket"
}

resource "aws_iam_policy" "dynamic" {
  name = "dynamic-s3-access"
  
  policy = <<EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": ["s3:GetObject"],
      "Resource": "arn:aws:s3:::${var.bucket_name}/*"
    }
  ]
}
EOF
}
```

### Custom EOF Markers

```hcl
# สามารถใช้ markers อื่นได้ (ต้องเป็น uppercase ตัวเดียวกัน)
resource "aws_instance" "web" {
  ami           = "ami-12345"
  instance_type = "t3.micro"
  
  # ใช้ EOT (End Of Text)
  user_data = <<EOT
    #!/bin/bash
    echo "Hello World"
  EOT
  
  # ใช้ BASH
  user_data = <<BASH
    #!/bin/bash
    yum update -y
  BASH
  
  # ใช้ SCRIPT
  user_data = <<SCRIPT
    #!/bin/bash
    # Setup script
  SCRIPT
  
  # ใช้ JSON
  # metadata = <<JSON
  #   { "key": "value" }
  # JSON
}
```

---

## Step 92: Indented Heredoc: <<-EOF ... EOF

### ความแตกต่างจาก Basic Heredoc

```hcl
# <<EOF: ไม่ลบ leading whitespace
resource "local_file" "basic" {
  filename = "/tmp/basic.txt"
  content  = <<EOF
    Line 1 (has 4 leading spaces)
    Line 2 (has 4 leading spaces)
EOF
  # Output:
  # "    Line 1 (has 4 leading spaces)\n    Line 2 (has 4 leading spaces)\n"
}

# <<-EOF: ลบ leading whitespace ตามการ indent น้อยที่สุด
resource "local_file" "indented" {
  filename = "/tmp/indented.txt"
  content  = <<-EOF
    Line 1 (no leading spaces in output)
    Line 2 (no leading spaces in output)
  EOF
  # Output:
  # "Line 1 (no leading spaces in output)\nLine 2 (no leading spaces in output)\n"
}

# ✅ แนะนำ: ใช้ <<-EOT เพื่อ code ที่อ่านง่าย
resource "aws_instance" "web" {
  ami           = "ami-12345"
  instance_type = "t3.micro"
  
  user_data = <<-BASH
    #!/bin/bash
    
    # Update system
    apt-get update -y
    
    # Install nginx
    apt-get install -y nginx
    
    # Start nginx
    systemctl enable nginx
    systemctl start nginx
    
    echo "Setup complete!"
  BASH
}
```

### Indented Heredoc Rules

```
Rules for <<-EOF:
┌────────────────────────────────────────────────────────────┐
│ - ลบ whitespace ที่ leading ในทุกบรรทัด                   │
│ - ลบตามจำนวน minimum indentation                          │
│ - Relative indentation ยังคงเก็บไว้                       │
│ - EOF marker ต้องมี indentation น้อยกว่าหรือเท่ากับ content│
└────────────────────────────────────────────────────────────┘
```

```hcl
# ตัวอย่าง: relative indentation preserved
resource "local_file" "script" {
  filename = "/tmp/script.sh"
  
  content = <<-BASH
    #!/bin/bash
    
    # This is at level 0 (after stripping 4 spaces)
    if [ "$1" == "start" ]; then
      # This is at level 1 (after stripping 4 spaces)
      echo "Starting..."
      service app start
        # This is at level 2
        echo "Started!"
    fi
  BASH
  # Output ยัง indent ถูกต้อง เพียงแค่ลบ 4 spaces ออกจากทุก line
}
```

---

## Step 93: Template Directives - %{ if condition }

### Conditional Template Directives

```hcl
# %{ if condition } ... %{ endif }
variable "environment" {
  default = "production"
}

variable "debug_mode" {
  type    = bool
  default = false
}

resource "local_file" "nginx_config" {
  filename = "/tmp/nginx.conf"
  
  content = <<-EOT
    server {
        listen 80;
        server_name example.com;
        
        %{ if var.environment == "production" }
        # Production settings
        access_log /var/log/nginx/access.log combined;
        error_log /var/log/nginx/error.log warn;
        %{ else }
        # Development settings
        access_log /dev/stdout;
        error_log /dev/stderr debug;
        %{ endif }
        
        %{ if var.debug_mode }
        # Debug configuration
        error_log /var/log/nginx/debug.log debug;
        rewrite_log on;
        %{ endif }
        
        location / {
            proxy_pass http://app:8080;
        }
    }
  EOT
}
```

### if-else Template

```hcl
variable "enable_ssl" {
  type    = bool
  default = true
}

variable "ssl_cert_path" {
  type    = string
  default = "/etc/ssl/certs/app.crt"
}

variable "ssl_key_path" {
  type    = string
  default = "/etc/ssl/private/app.key"
}

locals {
  nginx_ssl_config = <<-EOT
    server {
        %{ if var.enable_ssl }
        listen 443 ssl;
        ssl_certificate ${var.ssl_cert_path};
        ssl_certificate_key ${var.ssl_key_path};
        ssl_protocols TLSv1.2 TLSv1.3;
        ssl_ciphers HIGH:!aNULL:!MD5;
        %{ else }
        listen 80;
        # SSL disabled - not recommended for production
        %{ endif }
        
        server_name _;
        
        location / {
            proxy_pass http://localhost:8080;
        }
    }
    
    %{ if var.enable_ssl }
    # Redirect HTTP to HTTPS
    server {
        listen 80;
        return 301 https://$host$request_uri;
    }
    %{ endif }
  EOT
}
```

---

## Step 94: Template For Loops

### %{ for item in list } ... %{ endfor }

```hcl
variable "dns_servers" {
  type    = list(string)
  default = ["8.8.8.8", "8.8.4.4", "1.1.1.1"]
}

variable "hostnames" {
  type = list(object({
    ip       = string
    hostname = string
    aliases  = list(string)
  }))
  
  default = [
    {
      ip       = "10.0.1.10"
      hostname = "web-01.internal"
      aliases  = ["web01", "www"]
    },
    {
      ip       = "10.0.1.11"
      hostname = "web-02.internal"
      aliases  = ["web02"]
    },
    {
      ip       = "10.0.2.10"
      hostname = "api-01.internal"
      aliases  = ["api"]
    }
  ]
}

# สร้าง /etc/hosts file
resource "local_file" "etc_hosts" {
  filename = "/tmp/hosts"
  
  content = <<-EOT
    # Generated by Terraform
    # DO NOT EDIT MANUALLY
    
    # Loopback
    127.0.0.1 localhost
    ::1       localhost
    
    # DNS Servers
    %{ for dns in var.dns_servers ~}
    # nameserver ${dns}
    %{ endfor ~}
    
    # Application Hosts
    %{ for host in var.hostnames ~}
    ${host.ip} ${host.hostname} ${join(" ", host.aliases)}
    %{ endfor ~}
  EOT
}

# Output จะเป็น:
# # Generated by Terraform
# # DO NOT EDIT MANUALLY
# 
# # Loopback
# 127.0.0.1 localhost
# ::1       localhost
# 
# # DNS Servers
# # nameserver 8.8.8.8
# # nameserver 8.8.4.4
# # nameserver 1.1.1.1
# 
# # Application Hosts
# 10.0.1.10 web-01.internal web01 www
# 10.0.1.11 web-02.internal web02
# 10.0.2.10 api-01.internal api
```

### Whitespace Control ใน Template Loops

```hcl
# ~ (tilde) ลบ newline หลัง directive
# ช่วยควบคุม whitespace ใน output

locals {
  items = ["apple", "banana", "cherry"]
  
  # ❌ Without ~: มี empty lines
  without_tilde = <<-EOT
    Items:
    %{ for item in local.items }
    - ${item}
    %{ endfor }
  EOT
  # Output:
  # Items:
  # 
  # - apple
  # 
  # - banana
  # 
  # - cherry
  # 
  
  # ✅ With ~: clean output
  with_tilde = <<-EOT
    Items:
    %{ for item in local.items ~}
    - ${item}
    %{ endfor ~}
  EOT
  # Output:
  # Items:
  # - apple
  # - banana
  # - cherry
}
```

---

## Step 95: Template Expressions กับ ${expression}

### Expressions ใน Templates

```hcl
variable "app_config" {
  type = object({
    name        = string
    version     = string
    port        = number
    environment = string
    replicas    = number
  })
  
  default = {
    name        = "myapp"
    version     = "1.0.0"
    port        = 8080
    environment = "production"
    replicas    = 3
  }
}

resource "local_file" "app_info" {
  filename = "/tmp/app-info.txt"
  
  content = <<-EOT
    Application Information
    =======================
    Name:        ${var.app_config.name}
    Version:     ${var.app_config.version}
    Port:        ${var.app_config.port}
    Environment: ${upper(var.app_config.environment)}
    Replicas:    ${var.app_config.replicas}
    
    Deployment Status:
    - Health Check URL: http://localhost:${var.app_config.port}/health
    - API Endpoint:     http://localhost:${var.app_config.port}/api
    - Metrics:         http://localhost:${var.app_config.port + 1}/metrics
    
    Configuration:
    - Is Production: ${var.app_config.environment == "production" ? "YES" : "NO"}
    - HA Mode:       ${var.app_config.replicas > 1 ? "Enabled (${var.app_config.replicas} replicas)" : "Disabled"}
    - Memory Limit:  ${var.app_config.environment == "production" ? "2GB" : "512MB"}
  EOT
}
```

### Function Calls ใน Templates

```hcl
variable "server_list" {
  type    = list(string)
  default = ["web-01", "web-02", "api-01", "db-01"]
}

variable "maintenance_window" {
  type = object({
    start = string
    end   = string
    days  = list(string)
  })
  
  default = {
    start = "02:00"
    end   = "04:00"
    days  = ["Saturday", "Sunday"]
  }
}

resource "local_file" "maintenance_doc" {
  filename = "/tmp/maintenance.md"
  
  content = <<-EOT
    # Maintenance Window Documentation
    
    **Generated:** ${formatdate("YYYY-MM-DD HH:mm:ss", timestamp())}
    
    ## Server Inventory
    
    Total servers: ${length(var.server_list)}
    
    Servers:
    %{ for server in sort(var.server_list) ~}
    - ${upper(server)}
    %{ endfor ~}
    
    ## Maintenance Schedule
    
    - **Window:** ${var.maintenance_window.start} - ${var.maintenance_window.end} UTC
    - **Days:** ${join(", ", var.maintenance_window.days)}
    - **Duration:** ${var.maintenance_window.end} - ${var.maintenance_window.start} hours
    
    ## Procedures
    
    %{ for i, server in sort(var.server_list) ~}
    ${i + 1}. Update ${server}
    %{ endfor ~}
  EOT
}
```

---

## Step 96: templatefile() Function

### การใช้ templatefile()

```hcl
# syntax: templatefile(path, variables)
# - path: path ของ template file (.tftpl extension แนะนำ)
# - variables: map ของ variables ที่จะส่งเข้า template

# สร้าง template file ก่อน: templates/nginx.conf.tftpl
# จากนั้นใช้ templatefile() ใน Terraform

resource "aws_instance" "web" {
  ami           = var.ami_id
  instance_type = var.instance_type
  
  user_data = templatefile("${path.module}/templates/user_data.sh.tftpl", {
    # Variables ที่ส่งเข้า template
    hostname    = "web-${count.index + 1}"
    environment = var.environment
    app_version = var.app_version
    
    # Complex variables
    packages    = ["nginx", "nodejs", "pm2", "git"]
    env_vars    = {
      APP_PORT   = tostring(var.app_port)
      NODE_ENV   = var.environment
      LOG_LEVEL  = var.environment == "production" ? "warn" : "debug"
    }
    
    # Service configurations
    nginx_config = {
      worker_processes  = var.environment == "production" ? "auto" : "1"
      worker_connections = var.environment == "production" ? 1024 : 256
    }
  })
}
```

### Template File (.tftpl)

```bash
# templates/user_data.sh.tftpl
#!/bin/bash
set -euo pipefail

# ================================================
# User Data Script
# Generated by Terraform
# Environment: ${environment}
# Hostname: ${hostname}
# ================================================

# Set hostname
hostnamectl set-hostname ${hostname}

# Update system
apt-get update -y
apt-get upgrade -y

# Install packages
%{ for pkg in packages ~}
echo "Installing ${pkg}..."
apt-get install -y ${pkg}
%{ endfor ~}

# Set environment variables
%{ for key, val in env_vars ~}
export ${key}="${val}"
echo "export ${key}=${val}" >> /etc/environment
%{ endfor ~}

# Configure nginx
cat > /etc/nginx/nginx.conf << 'NGINX'
worker_processes ${nginx_config.worker_processes};
events {
    worker_connections ${nginx_config.worker_connections};
}

http {
    include /etc/nginx/mime.types;
    default_type application/octet-stream;
    
    %{ if environment == "production" ~}
    access_log /var/log/nginx/access.log combined;
    error_log /var/log/nginx/error.log warn;
    %{ else ~}
    access_log /dev/stdout;
    error_log /dev/stderr debug;
    %{ endif ~}
    
    include /etc/nginx/conf.d/*.conf;
}
NGINX

# Start services
systemctl enable nginx
systemctl start nginx

echo "=== Setup complete for ${hostname} ==="
```

---

## Step 97: Template Files (.tftpl)

### .tftpl File Format

```
Template File Extension:
├── .tftpl = Terraform Template file
├── สามารถ preview ด้วย `terraform console`
├── Supports: ${}, %{ if }, %{ for }, %{ endif }, %{ endfor }
└── ใช้กับ templatefile() function เท่านั้น
```

### ตัวอย่าง: Kubernetes Manifest Template

```yaml
# templates/deployment.yaml.tftpl
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ${app_name}
  namespace: ${namespace}
  labels:
    app: ${app_name}
    version: ${app_version}
    environment: ${environment}
  annotations:
    managed-by: terraform
    deploy-time: ${deploy_time}
spec:
  replicas: ${replicas}
  selector:
    matchLabels:
      app: ${app_name}
  template:
    metadata:
      labels:
        app: ${app_name}
        version: ${app_version}
    spec:
      containers:
      - name: ${app_name}
        image: ${docker_image}:${app_version}
        ports:
        - containerPort: ${container_port}
        
        %{ if length(env_vars) > 0 ~}
        env:
        %{ for key, val in env_vars ~}
        - name: ${key}
          value: "${val}"
        %{ endfor ~}
        %{ endif ~}
        
        %{ if length(secret_refs) > 0 ~}
        envFrom:
        %{ for secret in secret_refs ~}
        - secretRef:
            name: ${secret}
        %{ endfor ~}
        %{ endif ~}
        
        resources:
          requests:
            cpu: ${cpu_request}
            memory: ${memory_request}
          limits:
            cpu: ${cpu_limit}
            memory: ${memory_limit}
        
        livenessProbe:
          httpGet:
            path: ${health_check_path}
            port: ${container_port}
          initialDelaySeconds: 30
          periodSeconds: 10
        
        readinessProbe:
          httpGet:
            path: ${health_check_path}
            port: ${container_port}
          initialDelaySeconds: 5
          periodSeconds: 5
```

```hcl
# main.tf - ใช้ template
resource "local_file" "k8s_deployment" {
  filename = "${path.module}/output/${var.app_name}-deployment.yaml"
  
  content = templatefile("${path.module}/templates/deployment.yaml.tftpl", {
    app_name         = var.app_name
    namespace        = var.kubernetes_namespace
    app_version      = var.app_version
    environment      = var.environment
    deploy_time      = timestamp()
    replicas         = var.environment == "production" ? 3 : 1
    docker_image     = "${var.ecr_repository_url}/${var.app_name}"
    container_port   = var.app_port
    health_check_path = "/health"
    
    env_vars = {
      APP_ENV   = var.environment
      APP_PORT  = tostring(var.app_port)
      LOG_LEVEL = var.environment == "production" ? "warn" : "debug"
    }
    
    secret_refs = ["app-secrets", "db-credentials"]
    
    cpu_request    = "100m"
    memory_request = "256Mi"
    cpu_limit      = var.environment == "production" ? "1000m" : "500m"
    memory_limit   = var.environment == "production" ? "1Gi" : "512Mi"
  })
}
```

---

## Step 98: Multi-line String Formatting

### Formatting Best Practices

```hcl
# ✅ Good: ใช้ <<-EOT สำหรับ indented heredoc
locals {
  well_formatted = <<-EOT
    Line 1
    Line 2
    Line 3
  EOT
  
  # ✅ JSON ที่ formatted อ่านง่าย
  policy_json = <<-JSON
    {
      "Version": "2012-10-17",
      "Statement": [
        {
          "Effect": "Allow",
          "Action": "s3:*",
          "Resource": "*"
        }
      ]
    }
  JSON
}

# ❌ Avoid: ยาวใน single line (ยากอ่าน)
locals {
  bad_format = "{\"Version\": \"2012-10-17\", \"Statement\": [{\"Effect\": \"Allow\", \"Action\": \"s3:*\", \"Resource\": \"*\"}]}"
}

# ✅ Better: jsonencode() สำหรับ structured data
locals {
  good_format = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect   = "Allow"
        Action   = "s3:*"
        Resource = "*"
      }
    ]
  })
}
```

### Multi-line กับ Escape Characters

```hcl
# ✅ Heredoc vs String concat
locals {
  # Heredoc (recommended for multi-line)
  heredoc_msg = <<-EOT
    Dear ${var.user_name},
    
    Your account has been created.
    
    Username: ${var.username}
    Password: (see separate email)
    
    Best regards,
    The Team
  EOT
  
  # String with \n (harder to read)
  newline_msg = "Dear ${var.user_name},\n\nYour account has been created.\n\nUsername: ${var.username}\nPassword: (see separate email)\n\nBest regards,\nThe Team"
  
  # Both are equivalent but heredoc is more readable
}
```

---

## Step 99: JSON/YAML Generation จาก Templates

### JSON Generation

```hcl
# Method 1: jsonencode() - แนะนำสำหรับ structured data
locals {
  app_config = jsonencode({
    application = {
      name    = var.app_name
      version = var.app_version
      port    = var.app_port
    }
    database = {
      host     = aws_db_instance.main.endpoint
      port     = aws_db_instance.main.port
      name     = var.db_name
    }
    features = {
      for feature, enabled in var.feature_flags :
      feature => enabled
    }
  })
}

# Method 2: templatefile() กับ .json.tftpl
# templates/config.json.tftpl:
# {
#   "app": {
#     "name": "${app_name}",
#     "version": "${app_version}",
#     "port": ${app_port}
#   },
#   "database": {
#     "host": "${db_host}",
#     "port": ${db_port}
#   },
#   "features": {
# %{ for feature, enabled in feature_flags ~}
#     "${feature}": ${enabled},
# %{ endfor ~}
#   }
# }

resource "aws_ssm_parameter" "app_config" {
  name  = "/${var.project}/${var.environment}/config"
  type  = "String"
  value = local.app_config
  
  tags = local.common_tags
}
```

### YAML Generation สำหรับ Kubernetes

```hcl
# Method: yamlencode()
locals {
  service_manifest = yamlencode({
    apiVersion = "v1"
    kind       = "Service"
    metadata = {
      name      = var.app_name
      namespace = var.kubernetes_namespace
      labels = {
        app         = var.app_name
        environment = var.environment
      }
    }
    spec = {
      selector = {
        app = var.app_name
      }
      type = "ClusterIP"
      ports = [
        {
          name       = "http"
          port       = 80
          targetPort = var.app_port
          protocol   = "TCP"
        }
      ]
    }
  })
}

resource "local_file" "k8s_service" {
  filename = "${path.module}/k8s/service.yaml"
  content  = local.service_manifest
}

# Helm values.yaml generation
locals {
  helm_values = yamlencode({
    replicaCount = var.environment == "production" ? 3 : 1
    
    image = {
      repository = "${var.ecr_repository_url}/${var.app_name}"
      tag        = var.app_version
      pullPolicy = "Always"
    }
    
    service = {
      type = "ClusterIP"
      port = var.app_port
    }
    
    ingress = {
      enabled   = true
      className = "nginx"
      hosts = [
        {
          host  = "${var.app_name}.${var.domain}"
          paths = [{ path = "/", pathType = "Prefix" }]
        }
      ]
    }
    
    resources = {
      requests = {
        cpu    = "100m"
        memory = "256Mi"
      }
      limits = {
        cpu    = var.environment == "production" ? "1000m" : "500m"
        memory = var.environment == "production" ? "1Gi" : "512Mi"
      }
    }
    
    autoscaling = {
      enabled                        = var.environment == "production"
      minReplicas                    = 2
      maxReplicas                    = 10
      targetCPUUtilizationPercentage = 70
    }
    
    env = [
      for key, val in var.app_env_vars :
      { name = key, value = val }
    ]
  })
}
```

---

## Step 100: Real-world Complete Examples

### Example 1: EC2 User Data Script

```hcl
# variables.tf
variable "server_config" {
  type = object({
    hostname      = string
    environment   = string
    app_port      = number
    db_host       = string
    db_port       = number
    db_name       = string
    extra_packages = optional(list(string), [])
    extra_env_vars = optional(map(string), {})
  })
}

# templates/ec2_user_data.sh.tftpl
# #!/bin/bash
# set -euo pipefail
# 
# # ============================================
# # EC2 Instance Initialization Script
# # Hostname: ${hostname}
# # Environment: ${environment}
# # Generated: $(date)
# # ============================================
# 
# # Set hostname
# hostnamectl set-hostname "${hostname}"
# 
# # System update
# yum update -y
# 
# # Install base packages
# yum install -y \
#     awscli \
#     jq \
#     curl \
#     wget
# 
# # Install extra packages
# %{ for pkg in extra_packages ~}
# yum install -y ${pkg}
# %{ endfor ~}
# 
# # Configure environment variables
# cat > /etc/profile.d/app.sh << 'ENVFILE'
# %{ for key, val in extra_env_vars ~}
# export ${key}="${val}"
# %{ endfor ~}
# export APP_PORT="${app_port}"
# export DATABASE_URL="postgres://${db_host}:${db_port}/${db_name}"
# export ENVIRONMENT="${environment}"
# ENVFILE
# 
# source /etc/profile.d/app.sh
# 
# %{ if environment == "production" ~}
# # Production-specific configuration
# 
# # Configure CloudWatch agent
# aws ssm get-parameter \
#     --name "/cloudwatch/${hostname}/config" \
#     --with-decryption \
#     --query Parameter.Value \
#     --output text > /tmp/cw-config.json
# 
# /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-ctl \
#     -a fetch-config \
#     -m ec2 \
#     -s \
#     -c file:/tmp/cw-config.json
# %{ else ~}
# # Non-production: simpler logging
# echo "Debug mode enabled for ${environment}" >> /var/log/user-data.log
# %{ endif ~}
# 
# echo "=== Initialization complete for ${hostname} ==="

# main.tf
resource "aws_instance" "app_server" {
  ami           = data.aws_ami.amazon_linux.id
  instance_type = var.server_config.environment == "production" ? "t3.large" : "t3.micro"
  
  user_data = templatefile("${path.module}/templates/ec2_user_data.sh.tftpl", {
    hostname      = var.server_config.hostname
    environment   = var.server_config.environment
    app_port      = var.server_config.app_port
    db_host       = var.server_config.db_host
    db_port       = var.server_config.db_port
    db_name       = var.server_config.db_name
    extra_packages = var.server_config.extra_packages
    extra_env_vars = var.server_config.extra_env_vars
  })
  
  tags = {
    Name        = var.server_config.hostname
    Environment = var.server_config.environment
  }
}
```

### Example 2: Cloud-init Config สำหรับ Ubuntu

```hcl
locals {
  # Cloud-init YAML configuration
  cloud_init_config = <<-CLOUDINIT
    #cloud-config
    # ============================================
    # Cloud-init Configuration
    # Project: ${var.project}
    # Environment: ${var.environment}
    # ============================================
    
    hostname: ${var.hostname}
    fqdn: ${var.hostname}.${var.domain}
    manage_etc_hosts: true
    
    # User creation
    users:
      - name: deploy
        gecos: Deploy User
        groups: [sudo, docker]
        shell: /bin/bash
        sudo: ALL=(ALL) NOPASSWD:ALL
        ssh_authorized_keys:
    %{ for key in var.ssh_public_keys ~}
          - ${key}
    %{ endfor ~}
    
    # Package installation
    package_update: true
    package_upgrade: true
    packages:
    %{ for pkg in concat(local.base_packages, var.additional_packages) ~}
      - ${pkg}
    %{ endfor ~}
    
    # Write files
    write_files:
      - path: /etc/app/config.json
        permissions: '0644'
        content: |
          ${indent(10, jsonencode({
            app_name    = var.project
            environment = var.environment
            db_url      = "postgres://${var.db_host}:5432/${var.db_name}"
          }))}
      
      - path: /etc/systemd/system/app.service
        permissions: '0644'
        content: |
          [Unit]
          Description=${var.project} Application
          After=network.target
          
          [Service]
          Type=simple
          User=deploy
          WorkingDirectory=/opt/app
          ExecStart=/opt/app/bin/start.sh
          Restart=always
          RestartSec=5
          Environment=NODE_ENV=${var.environment}
          Environment=PORT=${var.app_port}
          
          [Install]
          WantedBy=multi-user.target
    
    # Run commands
    runcmd:
    %{ if var.environment == "production" ~}
      - [bash, -c, "aws s3 cp s3://${var.artifacts_bucket}/app.tar.gz /tmp/"]
      - [bash, -c, "tar -xzf /tmp/app.tar.gz -C /opt/app"]
    %{ else ~}
      - [echo, "Development mode - no artifact download"]
    %{ endif ~}
      - [systemctl, daemon-reload]
      - [systemctl, enable, app]
      - [systemctl, start, app]
      
    final_message: |
      =================================
      Cloud-init completed for ${var.hostname}
      Environment: ${var.environment}
      =================================
  CLOUDINIT
}

resource "aws_instance" "app" {
  ami           = data.aws_ami.ubuntu.id
  instance_type = var.instance_type
  
  user_data = local.cloud_init_config
  
  tags = {
    Name        = var.hostname
    Environment = var.environment
  }
}
```

### Example 3: IAM Policy Template

```hcl
# templates/s3_policy.json.tftpl
# {
#   "Version": "2012-10-17",
#   "Statement": [
#     %{ for i, stmt in statements ~}
#     {
#       "Sid": "${stmt.sid}",
#       "Effect": "${stmt.effect}",
#       "Action": ${jsonencode(stmt.actions)},
#       "Resource": [
#         %{ for j, resource in stmt.resources ~}
#         "${resource}"${j < length(stmt.resources) - 1 ? "," : ""}
#         %{ endfor ~}
#       ]
#       %{ if length(stmt.conditions) > 0 ~}
#       ,
#       "Condition": {
#         %{ for cond_test, cond_values in stmt.conditions ~}
#         "${cond_test}": ${jsonencode(cond_values)}
#         %{ endfor ~}
#       }
#       %{ endif ~}
#     }${i < length(statements) - 1 ? "," : ""}
#     %{ endfor ~}
#   ]
# }

# main.tf
locals {
  s3_policy = templatefile("${path.module}/templates/s3_policy.json.tftpl", {
    statements = [
      {
        sid     = "ReadAccess"
        effect  = "Allow"
        actions = ["s3:GetObject", "s3:ListBucket"]
        resources = [
          "arn:aws:s3:::${var.bucket_name}",
          "arn:aws:s3:::${var.bucket_name}/*"
        ]
        conditions = {}
      },
      {
        sid     = "WriteAccess"
        effect  = "Allow"
        actions = ["s3:PutObject", "s3:DeleteObject"]
        resources = [
          "arn:aws:s3:::${var.bucket_name}/uploads/*"
        ]
        conditions = {
          "StringEquals" = {
            "s3:prefix" = ["uploads/"]
          }
        }
      }
    ]
  })
}

resource "aws_iam_policy" "s3_access" {
  name        = "${var.project}-s3-policy"
  description = "S3 access policy for ${var.project}"
  policy      = local.s3_policy
}
```

---

## Template Quick Reference

### Template Syntax Summary

| Syntax | Description | Example |
|--------|-------------|---------|
| `${expression}` | Interpolation | `${var.name}` |
| `$${literal}` | Escape interpolation | `$${HOME}` |
| `%{if cond}` | Conditional start | `%{ if env == "prod" }` |
| `%{else}` | Else branch | `%{ else }` |
| `%{endif}` | End conditional | `%{ endif }` |
| `%{for x in list}` | Loop start | `%{ for pkg in packages }` |
| `%{endfor}` | End loop | `%{ endfor }` |
| `~` (strip) | Remove trailing newline | `%{ if cond ~}` |
| `<<EOF` | Basic heredoc | `<<EOF...EOF` |
| `<<-EOF` | Indented heredoc | `<<-EOT...EOT` |

### templatefile() vs Heredoc

| Feature | `templatefile()` | Heredoc |
|---------|-----------------|---------|
| Variables | Explicit map | Implicit (closure) |
| Reusability | High (separate file) | Low (inline only) |
| Editor support | Good (.tftpl) | Basic |
| Version control | Easy to diff | Mixed with HCL |
| Use case | Large templates | Small config snippets |

💡 **Pro Tips:**
- ใช้ `<<-EOT` เสมอ (indented) เพื่อ code ที่สะอาด
- ใช้ `templatefile()` สำหรับ templates ขนาดใหญ่ > 20 lines
- ใช้ `~` เพื่อ strip whitespace ใน template directives
- Preview templates ด้วย `terraform console` → `templatefile("...", {})`
- ใช้ `.tftpl` extension สำหรับ VS Code syntax highlighting

⚠️ **Common Mistakes:**
- ลืม `$$` เมื่อต้องการ literal `$` ใน bash scripts
- ใช้ `%{ if }` โดยไม่มี `%{ endif }`
- Indentation ไม่สม่ำเสมอใน heredoc ทำให้ output ผิด
- ใช้ heredoc ใน variable values (ต้องอยู่ใน resource/local)

---

## สรุปทั้งหมด Part 001-010

### สิ่งที่เรียนรู้ตลอด Course

| Part | หัวข้อ | Key Concepts |
|------|--------|--------------|
| 001 | Introduction | IaC, HCL vs JSON/YAML, Workflow |
| 002 | Syntax | Blocks, Arguments, Identifiers, Provider |
| 003 | Primitives | string, number, bool, null, conversions |
| 004 | Complex Types | list, set, map, object, tuple |
| 005 | Expressions | Arithmetic, Comparison, Logical, Splat |
| 006 | Functions | String, Collection, Numeric, IP, Hash |
| 007 | Conditionals | Ternary, Nested, count=0|1, coalesce |
| 008 | For Expressions | List/Map output, Filter, Grouping |
| 009 | Dynamic Blocks | Syntax, iterator, vs count/for_each |
| 010 | Templates | Heredoc, %{if}, %{for}, templatefile() |

### Next Steps

```
เมื่อเรียนจบ 10 parts นี้แล้ว ควรศึกษาต่อ:
├── Terraform Modules (การสร้าง reusable modules)
├── Terraform State Management (remote backend)
├── Terraform Workspaces (multiple environments)
├── Terraform Testing (terraform test, terratest)
├── CI/CD Integration (GitHub Actions, GitLab CI)
└── Cloud Provider specifics (AWS, Azure, GCP)
```

---

*จบ Part 010 - HCL Template Syntax & Heredocs*

*กลับไป: [Part 009 - HCL Dynamic Blocks](part-009.md)*

---

**Happy Terraforming! 🚀**
