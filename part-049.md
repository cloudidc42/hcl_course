# Part 049: AWS ECR & ECS Container Services
## การจัดการ Container Infrastructure ด้วย Terraform (Steps 481-490)

---

## บทนำ (Introduction)

AWS ECR (Elastic Container Registry) และ ECS (Elastic Container Service) ช่วยให้เราสามารถ
build, store, และรัน containerized applications บน AWS ได้อย่างมีประสิทธิภาพ

**หัวข้อที่จะเรียนรู้:**
- ECR Repository พร้อม Lifecycle Policies
- ECS Cluster (Fargate + EC2)
- Task Definitions พร้อม containerDefinitions
- ECS Services
- Load Balancer Integration
- Service Discovery
- Auto Scaling
- Secrets Management
- CloudWatch Logs

---

## Step 481: ECR Repository

### aws_ecr_repository

```hcl
# ✅ Secure: ECR Repository
resource "aws_ecr_repository" "app" {
  name                 = "${var.project_name}/app"
  image_tag_mutability = "IMMUTABLE"  # ✅ ป้องกัน overwrite existing tags

  # ✅ Enable image scanning
  image_scanning_configuration {
    scan_on_push = true  # Scan ทุกครั้งที่ push
  }

  # ✅ Encryption
  encryption_configuration {
    encryption_type = "KMS"
    kms_key         = aws_kms_key.ecr.arn
  }

  tags = {
    Name        = "${var.project_name}/app"
    Environment = var.environment
    ManagedBy   = "terraform"
  }
}

# ❌ Insecure ECR
resource "aws_ecr_repository" "insecure" {
  name                 = "insecure-repo"
  image_tag_mutability = "MUTABLE"  # ❌ Tags สามารถ overwrite ได้

  image_scanning_configuration {
    scan_on_push = false  # ❌ ไม่ scan
  }
  # ❌ ไม่มี encryption
}

# ✅ KMS Key สำหรับ ECR
resource "aws_kms_key" "ecr" {
  description             = "KMS key for ECR - ${var.project_name}"
  deletion_window_in_days = 14
  enable_key_rotation     = true

  tags = {
    Name      = "${var.project_name}-ecr-kms"
    ManagedBy = "terraform"
  }
}
```

### ECR Lifecycle Policy

```hcl
# ✅ Lifecycle Policy - ลบ images เก่า
resource "aws_ecr_lifecycle_policy" "app" {
  repository = aws_ecr_repository.app.name

  policy = jsonencode({
    rules = [
      {
        rulePriority = 1
        description  = "Keep last 30 tagged images"
        selection = {
          tagStatus     = "tagged"
          tagPrefixList = ["v", "release"]
          countType     = "imageCountMoreThan"
          countNumber   = 30
        }
        action = {
          type = "expire"
        }
      },
      {
        rulePriority = 2
        description  = "Remove untagged images after 7 days"
        selection = {
          tagStatus   = "untagged"
          countType   = "sinceImagePushed"
          countUnit   = "days"
          countNumber = 7
        }
        action = {
          type = "expire"
        }
      },
      {
        rulePriority = 3
        description  = "Keep only last 10 dev images"
        selection = {
          tagStatus     = "tagged"
          tagPrefixList = ["dev", "feature"]
          countType     = "imageCountMoreThan"
          countNumber   = 10
        }
        action = {
          type = "expire"
        }
      },
    ]
  })
}
```

### ECR Repository Policy

```hcl
# ✅ ECR Repository Policy - Cross-Account Access
data "aws_iam_policy_document" "ecr_policy" {
  statement {
    sid    = "AllowPushPullFromCICD"
    effect = "Allow"

    principals {
      type        = "AWS"
      identifiers = [
        "arn:aws:iam::${var.cicd_account_id}:root",
        aws_iam_role.github_actions.arn,
      ]
    }

    actions = [
      "ecr:BatchCheckLayerAvailability",
      "ecr:BatchGetImage",
      "ecr:CompleteLayerUpload",
      "ecr:GetDownloadUrlForLayer",
      "ecr:InitiateLayerUpload",
      "ecr:PutImage",
      "ecr:UploadLayerPart",
    ]
  }

  statement {
    sid    = "AllowPullFromECS"
    effect = "Allow"

    principals {
      type        = "AWS"
      identifiers = [
        aws_iam_role.ecs_task_execution.arn,
      ]
    }

    actions = [
      "ecr:BatchCheckLayerAvailability",
      "ecr:BatchGetImage",
      "ecr:GetDownloadUrlForLayer",
    ]
  }
}

resource "aws_ecr_repository_policy" "app" {
  repository = aws_ecr_repository.app.name
  policy     = data.aws_iam_policy_document.ecr_policy.json
}
```

---

## Step 482: ECS Cluster

### aws_ecs_cluster

```hcl
# ✅ ECS Cluster
resource "aws_ecs_cluster" "main" {
  name = "${var.project_name}-cluster"

  # ✅ Container Insights สำหรับ monitoring
  setting {
    name  = "containerInsights"
    value = "enabled"
  }

  tags = {
    Name        = "${var.project_name}-cluster"
    Environment = var.environment
    ManagedBy   = "terraform"
  }
}

# ✅ Capacity Providers
resource "aws_ecs_cluster_capacity_providers" "main" {
  cluster_name = aws_ecs_cluster.main.name

  capacity_providers = ["FARGATE", "FARGATE_SPOT"]

  default_capacity_provider_strategy {
    base              = 1
    weight            = 100
    capacity_provider = "FARGATE"
  }
}

# ✅ EC2 Capacity Provider (สำหรับ ECS with EC2)
resource "aws_ecs_capacity_provider" "ec2" {
  name = "${var.project_name}-ec2-capacity"

  auto_scaling_group_provider {
    auto_scaling_group_arn = aws_autoscaling_group.ecs.arn

    managed_scaling {
      maximum_scaling_step_size = 5
      minimum_scaling_step_size = 1
      status                    = "ENABLED"
      target_capacity           = 80  # Target 80% utilization
    }

    managed_termination_protection = "ENABLED"
  }

  tags = {
    Name      = "${var.project_name}-ec2-capacity"
    ManagedBy = "terraform"
  }
}
```

---

## Step 483: Task Definitions

### aws_ecs_task_definition (Fargate)

```hcl
# ✅ Task Definition สำหรับ Fargate
resource "aws_ecs_task_definition" "app" {
  family                   = "${var.project_name}-app"
  requires_compatibilities = ["FARGATE"]
  network_mode             = "awsvpc"
  cpu                      = "512"   # 0.5 vCPU
  memory                   = "1024"  # 1GB RAM

  execution_role_arn = aws_iam_role.ecs_task_execution.arn
  task_role_arn      = aws_iam_role.ecs_task.arn

  container_definitions = jsonencode([
    {
      name  = "app"
      image = "${aws_ecr_repository.app.repository_url}:${var.app_image_tag}"

      # Resource limits
      cpu    = 512
      memory = 1024

      # ✅ Port mapping
      portMappings = [
        {
          containerPort = 8080
          hostPort      = 8080
          protocol      = "tcp"
          name          = "http"
        }
      ]

      # ✅ Environment variables (non-sensitive)
      environment = [
        {
          name  = "ENVIRONMENT"
          value = var.environment
        },
        {
          name  = "AWS_REGION"
          value = var.aws_region
        },
        {
          name  = "APP_PORT"
          value = "8080"
        },
      ]

      # ✅ Secrets จาก Secrets Manager (sensitive values)
      secrets = [
        {
          name      = "DB_PASSWORD"
          valueFrom = "${aws_secretsmanager_secret.db_credentials.arn}:password::"
        },
        {
          name      = "DB_HOST"
          valueFrom = "${aws_secretsmanager_secret.db_credentials.arn}:host::"
        },
        {
          name      = "REDIS_AUTH_TOKEN"
          valueFrom = aws_secretsmanager_secret.redis_auth_token.arn
        },
      ]

      # ✅ CloudWatch Logs
      logConfiguration = {
        logDriver = "awslogs"
        options = {
          "awslogs-group"         = aws_cloudwatch_log_group.app.name
          "awslogs-region"        = var.aws_region
          "awslogs-stream-prefix" = "app"
        }
      }

      # ✅ Health check
      healthCheck = {
        command     = ["CMD-SHELL", "curl -f http://localhost:8080/health || exit 1"]
        interval    = 30
        timeout     = 5
        retries     = 3
        startPeriod = 60
      }

      # ✅ Read-only root filesystem
      readonlyRootFilesystem = true

      # ✅ ห้าม privilege escalation
      privileged             = false
      user                   = "1000"  # Non-root user

      essential = true

      # Mount points สำหรับ tmp directory
      mountPoints = [
        {
          sourceVolume  = "tmp"
          containerPath = "/tmp"
          readOnly      = false
        }
      ]
    },

    # ✅ Sidecar container สำหรับ logging
    {
      name  = "log-router"
      image = "public.ecr.aws/aws-observability/aws-for-fluent-bit:stable"

      cpu    = 64
      memory = 128

      essential = false

      logConfiguration = {
        logDriver = "awslogs"
        options = {
          "awslogs-group"         = aws_cloudwatch_log_group.app.name
          "awslogs-region"        = var.aws_region
          "awslogs-stream-prefix" = "fluent-bit"
        }
      }

      environment = [
        {
          name  = "AWS_REGION"
          value = var.aws_region
        }
      ]
    }
  ])

  volume {
    name = "tmp"
  }

  # ✅ Ephemeral storage
  ephemeral_storage {
    size_in_gib = 21  # Minimum 21GB
  }

  tags = {
    Name        = "${var.project_name}-app-task"
    Environment = var.environment
    ManagedBy   = "terraform"
  }
}
```

### Task Definition สำหรับ EC2 Launch Type

```hcl
# ✅ Task Definition สำหรับ EC2
resource "aws_ecs_task_definition" "app_ec2" {
  family                   = "${var.project_name}-app-ec2"
  requires_compatibilities = ["EC2"]
  network_mode             = "bridge"  # หรือ awsvpc
  cpu                      = "1024"
  memory                   = "2048"

  execution_role_arn = aws_iam_role.ecs_task_execution.arn
  task_role_arn      = aws_iam_role.ecs_task.arn

  container_definitions = jsonencode([
    {
      name  = "app"
      image = "${aws_ecr_repository.app.repository_url}:${var.app_image_tag}"

      cpu    = 1024
      memory = 2048

      portMappings = [
        {
          containerPort = 8080
          # hostPort ไม่ระบุ = dynamic port mapping
          protocol = "tcp"
        }
      ]

      environment = [
        {
          name  = "ENVIRONMENT"
          value = var.environment
        },
      ]

      secrets = [
        {
          name      = "DB_SECRET"
          valueFrom = aws_secretsmanager_secret.db_credentials.arn
        },
      ]

      logConfiguration = {
        logDriver = "awslogs"
        options = {
          "awslogs-group"         = aws_cloudwatch_log_group.app.name
          "awslogs-region"        = var.aws_region
          "awslogs-stream-prefix" = "app-ec2"
        }
      }

      healthCheck = {
        command     = ["CMD-SHELL", "curl -f http://localhost:8080/health || exit 1"]
        interval    = 30
        timeout     = 5
        retries     = 3
        startPeriod = 60
      }

      essential = true
    }
  ])

  tags = {
    Name      = "${var.project_name}-app-ec2-task"
    ManagedBy = "terraform"
  }
}
```

---

## Step 484: IAM Roles สำหรับ ECS

```hcl
# ✅ ECS Task Execution Role (สำหรับ ECS control plane)
data "aws_iam_policy_document" "ecs_task_execution_trust" {
  statement {
    effect = "Allow"
    principals {
      type        = "Service"
      identifiers = ["ecs-tasks.amazonaws.com"]
    }
    actions = ["sts:AssumeRole"]
  }
}

resource "aws_iam_role" "ecs_task_execution" {
  name               = "${var.project_name}-ecs-task-execution"
  assume_role_policy = data.aws_iam_policy_document.ecs_task_execution_trust.json

  tags = {
    Name      = "${var.project_name}-ecs-task-execution"
    ManagedBy = "terraform"
  }
}

resource "aws_iam_role_policy_attachment" "ecs_task_execution_basic" {
  role       = aws_iam_role.ecs_task_execution.name
  policy_arn = "arn:aws:iam::aws:policy/service-role/AmazonECSTaskExecutionRolePolicy"
}

# ✅ Policy สำหรับ pull secrets และ ECR images
resource "aws_iam_role_policy" "ecs_task_execution_extras" {
  name = "task-execution-extras"
  role = aws_iam_role.ecs_task_execution.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Sid    = "SecretsManagerAccess"
        Effect = "Allow"
        Action = [
          "secretsmanager:GetSecretValue",
        ]
        Resource = [
          "arn:aws:secretsmanager:${var.aws_region}:${data.aws_caller_identity.current.account_id}:secret:${var.project_name}/*",
        ]
      },
      {
        Sid    = "SSMParameterAccess"
        Effect = "Allow"
        Action = [
          "ssm:GetParameters",
          "ssm:GetParameter",
        ]
        Resource = [
          "arn:aws:ssm:${var.aws_region}:${data.aws_caller_identity.current.account_id}:parameter/${var.project_name}/*",
        ]
      },
      {
        Sid    = "KMSDecrypt"
        Effect = "Allow"
        Action = [
          "kms:Decrypt",
          "kms:DescribeKey",
        ]
        Resource = [
          aws_kms_key.secrets.arn,
        ]
      },
    ]
  })
}

# ✅ ECS Task Role (สำหรับ application code)
resource "aws_iam_role" "ecs_task" {
  name               = "${var.project_name}-ecs-task"
  assume_role_policy = data.aws_iam_policy_document.ecs_task_execution_trust.json

  tags = {
    Name      = "${var.project_name}-ecs-task"
    ManagedBy = "terraform"
  }
}

# ✅ Application permissions
resource "aws_iam_role_policy" "ecs_task_app" {
  name = "app-permissions"
  role = aws_iam_role.ecs_task.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Sid    = "S3Access"
        Effect = "Allow"
        Action = [
          "s3:GetObject",
          "s3:PutObject",
          "s3:DeleteObject",
          "s3:ListBucket",
        ]
        Resource = [
          aws_s3_bucket.app_data.arn,
          "${aws_s3_bucket.app_data.arn}/*",
        ]
      },
      {
        Sid    = "SQSAccess"
        Effect = "Allow"
        Action = [
          "sqs:SendMessage",
          "sqs:ReceiveMessage",
          "sqs:DeleteMessage",
        ]
        Resource = [
          aws_sqs_queue.standard.arn,
        ]
      },
      {
        Sid    = "DynamoDBAccess"
        Effect = "Allow"
        Action = [
          "dynamodb:GetItem",
          "dynamodb:PutItem",
          "dynamodb:UpdateItem",
          "dynamodb:DeleteItem",
          "dynamodb:Query",
        ]
        Resource = [
          "arn:aws:dynamodb:${var.aws_region}:${data.aws_caller_identity.current.account_id}:table/${var.project_name}-*",
        ]
      },
    ]
  })
}
```

---

## Step 485: ECS Services

### aws_ecs_service

```hcl
# ✅ Security Group สำหรับ ECS Tasks
resource "aws_security_group" "ecs_tasks" {
  name        = "${var.project_name}-ecs-tasks-sg"
  description = "Security group for ECS tasks"
  vpc_id      = aws_vpc.main.id

  ingress {
    description     = "App port from ALB"
    from_port       = 8080
    to_port         = 8080
    protocol        = "tcp"
    security_groups = [aws_security_group.alb.id]
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
    description = "Allow all outbound"
  }

  tags = {
    Name      = "${var.project_name}-ecs-tasks-sg"
    ManagedBy = "terraform"
  }
}

# ✅ ECS Fargate Service
resource "aws_ecs_service" "app" {
  name            = "${var.project_name}-app"
  cluster         = aws_ecs_cluster.main.id
  task_definition = aws_ecs_task_definition.app.arn

  desired_count = var.app_desired_count

  # ✅ Launch type
  launch_type = "FARGATE"
  # หรือใช้ capacity_provider_strategy:
  # capacity_provider_strategy {
  #   capacity_provider = "FARGATE"
  #   weight            = 3
  # }
  # capacity_provider_strategy {
  #   capacity_provider = "FARGATE_SPOT"
  #   weight            = 1
  # }

  # ✅ Network configuration
  network_configuration {
    subnets          = aws_subnet.private[*].id
    security_groups  = [aws_security_group.ecs_tasks.id]
    assign_public_ip = false  # ✅ Private IP เท่านั้น
  }

  # ✅ Load Balancer integration
  load_balancer {
    target_group_arn = aws_lb_target_group.app.arn
    container_name   = "app"
    container_port   = 8080
  }

  # ✅ Deployment settings
  deployment_minimum_healthy_percent = 50   # ลด ไม่เกิน 50% ระหว่าง deploy
  deployment_maximum_percent         = 200  # เพิ่มได้สูงสุด 200%

  deployment_circuit_breaker {
    enable   = true   # ✅ Rollback เมื่อ deploy fail
    rollback = true
  }

  deployment_controller {
    type = "ECS"  # หรือ "CODE_DEPLOY" สำหรับ blue/green
  }

  # ✅ Health check grace period
  health_check_grace_period_seconds = 60

  # ✅ Service discovery (optional)
  service_registries {
    registry_arn = aws_service_discovery_service.app.arn
  }

  # ✅ Enable execute command (สำหรับ debugging)
  enable_execute_command = var.enable_ecs_exec

  # ✅ Wait for steady state
  wait_for_steady_state = false

  lifecycle {
    # ✅ ไม่ให้ Terraform reset desired_count เมื่อ auto scaling เปลี่ยน
    ignore_changes = [desired_count]
  }

  tags = {
    Name        = "${var.project_name}-app-service"
    Environment = var.environment
    ManagedBy   = "terraform"
  }

  depends_on = [
    aws_lb_listener.https,
    aws_iam_role_policy_attachment.ecs_task_execution_basic,
  ]
}
```

---

## Step 486: Service Discovery

```hcl
# ✅ Service Discovery Namespace
resource "aws_service_discovery_private_dns_namespace" "main" {
  name        = "${var.project_name}.local"
  description = "Service discovery namespace for ${var.project_name}"
  vpc         = aws_vpc.main.id

  tags = {
    Name      = "${var.project_name}-namespace"
    ManagedBy = "terraform"
  }
}

# ✅ Service Discovery Service
resource "aws_service_discovery_service" "app" {
  name = "app"

  dns_config {
    namespace_id = aws_service_discovery_private_dns_namespace.main.id

    dns_records {
      ttl  = 10
      type = "A"
    }

    routing_policy = "MULTIVALUE"
  }

  health_check_custom_config {
    failure_threshold = 1
  }

  tags = {
    Name      = "${var.project_name}-app-discovery"
    ManagedBy = "terraform"
  }
}

# ✅ API Discovery
resource "aws_service_discovery_service" "api" {
  name = "api"

  dns_config {
    namespace_id = aws_service_discovery_private_dns_namespace.main.id

    dns_records {
      ttl  = 10
      type = "A"
    }
  }

  health_check_custom_config {
    failure_threshold = 1
  }

  tags = {
    Name      = "${var.project_name}-api-discovery"
    ManagedBy = "terraform"
  }
}

# ✅ Services สามารถ communicate กันด้วย DNS
# app.myproject.local -> app service
# api.myproject.local -> api service
```

---

## Step 487: Auto Scaling สำหรับ ECS

```hcl
# ✅ Auto Scaling Target
resource "aws_appautoscaling_target" "ecs_app" {
  max_capacity       = 20
  min_capacity       = 2
  resource_id        = "service/${aws_ecs_cluster.main.name}/${aws_ecs_service.app.name}"
  scalable_dimension = "ecs:service:DesiredCount"
  service_namespace  = "ecs"
}

# ✅ CPU-based Auto Scaling
resource "aws_appautoscaling_policy" "ecs_cpu" {
  name               = "${var.project_name}-cpu-scaling"
  policy_type        = "TargetTrackingScaling"
  resource_id        = aws_appautoscaling_target.ecs_app.resource_id
  scalable_dimension = aws_appautoscaling_target.ecs_app.scalable_dimension
  service_namespace  = aws_appautoscaling_target.ecs_app.service_namespace

  target_tracking_scaling_policy_configuration {
    target_value       = 60.0  # Target 60% CPU
    scale_in_cooldown  = 300   # 5 minutes
    scale_out_cooldown = 60    # 1 minute

    predefined_metric_specification {
      predefined_metric_type = "ECSServiceAverageCPUUtilization"
    }
  }
}

# ✅ Memory-based Auto Scaling
resource "aws_appautoscaling_policy" "ecs_memory" {
  name               = "${var.project_name}-memory-scaling"
  policy_type        = "TargetTrackingScaling"
  resource_id        = aws_appautoscaling_target.ecs_app.resource_id
  scalable_dimension = aws_appautoscaling_target.ecs_app.scalable_dimension
  service_namespace  = aws_appautoscaling_target.ecs_app.service_namespace

  target_tracking_scaling_policy_configuration {
    target_value       = 70.0  # Target 70% Memory
    scale_in_cooldown  = 300
    scale_out_cooldown = 60

    predefined_metric_specification {
      predefined_metric_type = "ECSServiceAverageMemoryUtilization"
    }
  }
}

# ✅ ALB Request Count-based Scaling
resource "aws_appautoscaling_policy" "ecs_alb_requests" {
  name               = "${var.project_name}-alb-scaling"
  policy_type        = "TargetTrackingScaling"
  resource_id        = aws_appautoscaling_target.ecs_app.resource_id
  scalable_dimension = aws_appautoscaling_target.ecs_app.scalable_dimension
  service_namespace  = aws_appautoscaling_target.ecs_app.service_namespace

  target_tracking_scaling_policy_configuration {
    target_value       = 100  # 100 requests per target
    scale_in_cooldown  = 300
    scale_out_cooldown = 60

    predefined_metric_specification {
      predefined_metric_type = "ALBRequestCountPerTarget"
      resource_label         = "${aws_lb.app.arn_suffix}/${aws_lb_target_group.app.arn_suffix}"
    }
  }
}

# ✅ Schedule-based Scaling (รองรับ traffic spikes ที่คาดการณ์ได้)
resource "aws_appautoscaling_scheduled_action" "scale_up_morning" {
  name               = "${var.project_name}-scale-up-morning"
  service_namespace  = aws_appautoscaling_target.ecs_app.service_namespace
  resource_id        = aws_appautoscaling_target.ecs_app.resource_id
  scalable_dimension = aws_appautoscaling_target.ecs_app.scalable_dimension

  schedule = "cron(0 8 * * MON-FRI)"  # 8 AM UTC วันจันทร์-ศุกร์

  scalable_target_action {
    min_capacity = 5
    max_capacity = 20
  }
}

resource "aws_appautoscaling_scheduled_action" "scale_down_night" {
  name               = "${var.project_name}-scale-down-night"
  service_namespace  = aws_appautoscaling_target.ecs_app.service_namespace
  resource_id        = aws_appautoscaling_target.ecs_app.resource_id
  scalable_dimension = aws_appautoscaling_target.ecs_app.scalable_dimension

  schedule = "cron(0 20 * * MON-FRI)"  # 8 PM UTC

  scalable_target_action {
    min_capacity = 2
    max_capacity = 10
  }
}
```

---

## Step 488: CloudWatch Logs สำหรับ ECS

```hcl
# ✅ CloudWatch Log Group สำหรับ ECS
resource "aws_cloudwatch_log_group" "app" {
  name              = "/ecs/${var.project_name}/app"
  retention_in_days = 30

  # ✅ Encrypt logs
  kms_key_id = aws_kms_key.cloudwatch.arn

  tags = {
    Name      = "${var.project_name}-app-logs"
    ManagedBy = "terraform"
  }
}

resource "aws_cloudwatch_log_group" "app_worker" {
  name              = "/ecs/${var.project_name}/worker"
  retention_in_days = 30
  kms_key_id        = aws_kms_key.cloudwatch.arn

  tags = {
    Name      = "${var.project_name}-worker-logs"
    ManagedBy = "terraform"
  }
}

# ✅ CloudWatch Container Insights
resource "aws_cloudwatch_metric_alarm" "ecs_cpu_high" {
  alarm_name          = "${var.project_name}-ecs-cpu-high"
  comparison_operator = "GreaterThanOrEqualToThreshold"
  evaluation_periods  = 3
  metric_name         = "CPUUtilization"
  namespace           = "AWS/ECS"
  period              = 60
  statistic           = "Average"
  threshold           = 80
  alarm_description   = "ECS service CPU is high"
  alarm_actions       = [aws_sns_topic.alerts.arn]

  dimensions = {
    ClusterName = aws_ecs_cluster.main.name
    ServiceName = aws_ecs_service.app.name
  }

  tags = {
    Name      = "${var.project_name}-ecs-cpu-alarm"
    ManagedBy = "terraform"
  }
}

resource "aws_cloudwatch_metric_alarm" "ecs_memory_high" {
  alarm_name          = "${var.project_name}-ecs-memory-high"
  comparison_operator = "GreaterThanOrEqualToThreshold"
  evaluation_periods  = 3
  metric_name         = "MemoryUtilization"
  namespace           = "AWS/ECS"
  period              = 60
  statistic           = "Average"
  threshold           = 85
  alarm_description   = "ECS service Memory is high"
  alarm_actions       = [aws_sns_topic.alerts.arn]

  dimensions = {
    ClusterName = aws_ecs_cluster.main.name
    ServiceName = aws_ecs_service.app.name
  }
}
```

---

## Step 489: ECS with EC2 Launch Type

```hcl
# ✅ ECS AMI
data "aws_ami" "ecs_optimized" {
  most_recent = true
  owners      = ["amazon"]

  filter {
    name   = "name"
    values = ["amzn2-ami-ecs-hvm-*-x86_64-ebs"]
  }
}

# ✅ ECS Instance Role
resource "aws_iam_role" "ecs_instance" {
  name               = "${var.project_name}-ecs-instance-role"
  assume_role_policy = data.aws_iam_policy_document.ec2_trust.json
}

resource "aws_iam_role_policy_attachment" "ecs_instance_policy" {
  role       = aws_iam_role.ecs_instance.name
  policy_arn = "arn:aws:iam::aws:policy/service-role/AmazonEC2ContainerServiceforEC2Role"
}

resource "aws_iam_role_policy_attachment" "ecs_instance_ssm" {
  role       = aws_iam_role.ecs_instance.name
  policy_arn = "arn:aws:iam::aws:policy/AmazonSSMManagedInstanceCore"
}

resource "aws_iam_instance_profile" "ecs" {
  name = "${var.project_name}-ecs-profile"
  role = aws_iam_role.ecs_instance.name
}

# ✅ Launch Template สำหรับ ECS EC2
resource "aws_launch_template" "ecs" {
  name        = "${var.project_name}-ecs-launch-template"
  description = "Launch template for ECS EC2 instances"

  image_id      = data.aws_ami.ecs_optimized.id
  instance_type = "t3.medium"

  iam_instance_profile {
    name = aws_iam_instance_profile.ecs.name
  }

  # ✅ IMDSv2
  metadata_options {
    http_endpoint               = "enabled"
    http_tokens                 = "required"
    http_put_response_hop_limit = 2
  }

  # ✅ User data สำหรับ register cluster
  user_data = base64encode(<<-EOT
    #!/bin/bash
    echo ECS_CLUSTER=${aws_ecs_cluster.main.name} >> /etc/ecs/ecs.config
    echo ECS_ENABLE_CONTAINER_METADATA=true >> /etc/ecs/ecs.config
    echo ECS_ENABLE_SPOT_INSTANCE_DRAINING=true >> /etc/ecs/ecs.config
  EOT
  )

  block_device_mappings {
    device_name = "/dev/xvda"
    ebs {
      volume_size           = 30
      volume_type           = "gp3"
      encrypted             = true
      delete_on_termination = true
    }
  }

  network_interfaces {
    associate_public_ip_address = false
    security_groups             = [aws_security_group.ecs_ec2.id]
  }

  tag_specifications {
    resource_type = "instance"
    tags = {
      Name      = "${var.project_name}-ecs-node"
      ManagedBy = "terraform"
    }
  }

  lifecycle {
    create_before_destroy = true
  }
}

# ✅ Auto Scaling Group สำหรับ ECS EC2
resource "aws_autoscaling_group" "ecs" {
  name                = "${var.project_name}-ecs-asg"
  vpc_zone_identifier = aws_subnet.private[*].id
  min_size            = 2
  max_size            = 10
  desired_capacity    = 2

  launch_template {
    id      = aws_launch_template.ecs.id
    version = "$Latest"
  }

  lifecycle {
    create_before_destroy = true
    ignore_changes        = [desired_capacity]
  }

  tag {
    key                 = "AmazonECSManaged"
    value               = ""
    propagate_at_launch = true
  }

  tag {
    key                 = "Name"
    value               = "${var.project_name}-ecs-node"
    propagate_at_launch = true
  }
}
```

---

## Step 490: Complete Fargate Production Setup

```hcl
# ✅ Complete Production ECS Fargate Configuration

# Worker Task Definition
resource "aws_ecs_task_definition" "worker" {
  family                   = "${var.project_name}-worker"
  requires_compatibilities = ["FARGATE"]
  network_mode             = "awsvpc"
  cpu                      = "256"
  memory                   = "512"

  execution_role_arn = aws_iam_role.ecs_task_execution.arn
  task_role_arn      = aws_iam_role.ecs_task.arn

  container_definitions = jsonencode([
    {
      name  = "worker"
      image = "${aws_ecr_repository.app.repository_url}:${var.app_image_tag}"

      cpu    = 256
      memory = 512

      command = ["python", "-m", "worker"]

      environment = [
        {
          name  = "WORKER_TYPE"
          value = "background"
        },
        {
          name  = "SQS_QUEUE_URL"
          value = aws_sqs_queue.standard.url
        },
      ]

      secrets = [
        {
          name      = "DB_SECRET"
          valueFrom = aws_secretsmanager_secret.db_credentials.arn
        },
      ]

      logConfiguration = {
        logDriver = "awslogs"
        options = {
          "awslogs-group"         = aws_cloudwatch_log_group.app_worker.name
          "awslogs-region"        = var.aws_region
          "awslogs-stream-prefix" = "worker"
        }
      }

      essential = true
    }
  ])

  tags = {
    Name      = "${var.project_name}-worker-task"
    ManagedBy = "terraform"
  }
}

# Worker Service
resource "aws_ecs_service" "worker" {
  name            = "${var.project_name}-worker"
  cluster         = aws_ecs_cluster.main.id
  task_definition = aws_ecs_task_definition.worker.arn
  desired_count   = 2
  launch_type     = "FARGATE"

  network_configuration {
    subnets          = aws_subnet.private[*].id
    security_groups  = [aws_security_group.ecs_tasks.id]
    assign_public_ip = false
  }

  deployment_minimum_healthy_percent = 50
  deployment_maximum_percent         = 200

  deployment_circuit_breaker {
    enable   = true
    rollback = true
  }

  lifecycle {
    ignore_changes = [desired_count]
  }

  tags = {
    Name      = "${var.project_name}-worker-service"
    ManagedBy = "terraform"
  }
}

# Worker Auto Scaling
resource "aws_appautoscaling_target" "worker" {
  max_capacity       = 10
  min_capacity       = 1
  resource_id        = "service/${aws_ecs_cluster.main.name}/${aws_ecs_service.worker.name}"
  scalable_dimension = "ecs:service:DesiredCount"
  service_namespace  = "ecs"
}

resource "aws_appautoscaling_policy" "worker_sqs" {
  name               = "${var.project_name}-worker-sqs"
  policy_type        = "TargetTrackingScaling"
  resource_id        = aws_appautoscaling_target.worker.resource_id
  scalable_dimension = aws_appautoscaling_target.worker.scalable_dimension
  service_namespace  = aws_appautoscaling_target.worker.service_namespace

  target_tracking_scaling_policy_configuration {
    target_value       = 10  # 10 messages per worker
    scale_in_cooldown  = 300
    scale_out_cooldown = 60

    customized_metric_specification {
      metric_name = "ApproximateNumberOfMessagesVisible"
      namespace   = "AWS/SQS"
      statistic   = "Sum"
      unit        = "Count"

      dimensions {
        name  = "QueueName"
        value = aws_sqs_queue.standard.name
      }
    }
  }
}

# Outputs
output "ecs_cluster_name" {
  description = "ECS Cluster name"
  value       = aws_ecs_cluster.main.name
}

output "ecs_service_name" {
  description = "ECS Service name"
  value       = aws_ecs_service.app.name
}

output "ecr_repository_url" {
  description = "ECR Repository URL"
  value       = aws_ecr_repository.app.repository_url
}

output "service_discovery_namespace" {
  description = "Service Discovery namespace"
  value       = aws_service_discovery_private_dns_namespace.main.name
}
```

---

## ECS Best Practices สรุป

### ✅ Security

1. **Image Immutability**: ตั้ง `image_tag_mutability = "IMMUTABLE"`
2. **Image Scanning**: เปิด `scan_on_push = true`
3. **Non-Root User**: ตั้ง `user = "1000"` ใน container definition
4. **Read-Only Filesystem**: ตั้ง `readonlyRootFilesystem = true`
5. **No Privileged**: ตั้ง `privileged = false`
6. **Private Subnets**: รัน tasks ใน private subnets
7. **Task Role**: ให้สิทธิ์เฉพาะที่จำเป็น
8. **Secrets Manager**: ใช้ secrets injection ไม่ใช่ env vars

### ✅ Reliability

1. **Deployment Circuit Breaker**: เปิด rollback
2. **Health Checks**: ตั้ง container health check
3. **Auto Scaling**: ตั้ง scaling policies
4. **Multi-AZ**: กระจาย tasks หลาย AZs
5. **DLQ สำหรับ Workers**: รับมือ failed messages

### ✅ Performance

1. **Container Insights**: เปิดสำหรับ monitoring
2. **Fargate Spot**: ใช้สำหรับ batch workloads
3. **Resource Limits**: ตั้ง cpu/memory อย่างเหมาะสม
4. **Capacity Provider Strategy**: ใช้ FARGATE + FARGATE_SPOT

---

**Next Steps**: ไปต่อที่ Part 050 - AWS Load Balancers (ALB/NLB/CLB)
