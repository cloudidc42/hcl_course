# Part 044: AWS Lambda Functions
## การสร้างและจัดการ Serverless Functions ด้วย Terraform (Steps 431-440)

---

## บทนำ (Introduction)

AWS Lambda เป็น serverless compute service ที่ช่วยให้เรารันโค้ดโดยไม่ต้องจัดการ servers
Terraform ช่วยจัดการ Lambda functions, event sources, และ configurations ทั้งหมดเป็น code

**หัวข้อที่จะเรียนรู้:**
- Lambda Function พื้นฐาน
- Deployment Packages (zip, S3, container)
- Lambda Layers
- VPC Configuration
- Event Sources (SQS, API GW, EventBridge, S3)
- Lambda Concurrency
- Monitoring และ Tracing
- Lambda Function URLs
- Lambda@Edge

---

## Step 431: Lambda Function พื้นฐาน

### Lambda Execution Role

```hcl
# ✅ Lambda Execution Role
data "aws_iam_policy_document" "lambda_trust" {
  statement {
    effect = "Allow"
    principals {
      type        = "Service"
      identifiers = ["lambda.amazonaws.com"]
    }
    actions = ["sts:AssumeRole"]
  }
}

resource "aws_iam_role" "lambda_basic" {
  name               = "${var.function_name}-role"
  assume_role_policy = data.aws_iam_policy_document.lambda_trust.json

  tags = {
    Name      = "${var.function_name}-role"
    ManagedBy = "terraform"
  }
}

# ✅ Basic execution policy (CloudWatch Logs)
resource "aws_iam_role_policy_attachment" "lambda_basic_execution" {
  role       = aws_iam_role.lambda_basic.name
  policy_arn = "arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole"
}

# ✅ VPC access (ถ้า Lambda อยู่ใน VPC)
resource "aws_iam_role_policy_attachment" "lambda_vpc_access" {
  role       = aws_iam_role.lambda_basic.name
  policy_arn = "arn:aws:iam::aws:policy/service-role/AWSLambdaVPCAccessExecutionRole"
}

# ✅ X-Ray tracing
resource "aws_iam_role_policy_attachment" "lambda_xray" {
  role       = aws_iam_role.lambda_basic.name
  policy_arn = "arn:aws:iam::aws:policy/AWSXRayDaemonWriteAccess"
}
```

### Deployment แบบ Zip File

```hcl
# ✅ สร้าง Zip file จาก source code
data "archive_file" "lambda_zip" {
  type        = "zip"
  source_dir  = "${path.module}/src/lambda"
  output_path = "${path.module}/dist/lambda.zip"
  # หรือ source_file สำหรับไฟล์เดียว
}

# ✅ Lambda Function จาก Zip file โดยตรง
resource "aws_lambda_function" "api_handler" {
  filename      = data.archive_file.lambda_zip.output_path
  function_name = "${var.project_name}-api-handler"
  role          = aws_iam_role.lambda_basic.arn
  handler       = "index.handler"
  runtime       = "nodejs20.x"

  # ✅ ตรวจสอบ hash เพื่อ detect code changes
  source_code_hash = data.archive_file.lambda_zip.output_base64sha256

  # Resource configuration
  memory_size = 256   # MB
  timeout     = 30    # seconds

  # ✅ Environment variables
  environment {
    variables = {
      ENVIRONMENT = var.environment
      LOG_LEVEL   = "INFO"
      # ✅ ไม่ใส่ secrets โดยตรง - ใช้ Secrets Manager
      SECRET_ARN  = aws_secretsmanager_secret.app_secret.arn
    }
  }

  # ✅ X-Ray tracing
  tracing_config {
    mode = "Active"
  }

  # ✅ Dead letter queue
  dead_letter_config {
    target_arn = aws_sqs_queue.lambda_dlq.arn
  }

  tags = {
    Name        = "${var.project_name}-api-handler"
    Environment = var.environment
    ManagedBy   = "terraform"
  }
}
```

### Deployment จาก S3

```hcl
# ✅ Upload Lambda package ไปยัง S3
resource "aws_s3_object" "lambda_package" {
  bucket = aws_s3_bucket.lambda_packages.id
  key    = "functions/${var.function_name}/${var.function_version}.zip"
  source = data.archive_file.lambda_zip.output_path
  etag   = filemd5(data.archive_file.lambda_zip.output_path)

  tags = {
    Name      = "${var.function_name}-package"
    Version   = var.function_version
    ManagedBy = "terraform"
  }
}

# ✅ Lambda Function จาก S3
resource "aws_lambda_function" "from_s3" {
  s3_bucket        = aws_s3_bucket.lambda_packages.id
  s3_key           = aws_s3_object.lambda_package.key
  s3_object_version = aws_s3_object.lambda_package.version_id

  function_name = "${var.project_name}-${var.function_name}"
  role          = aws_iam_role.lambda_basic.arn
  handler       = "handler.main"
  runtime       = "python3.11"

  source_code_hash = data.archive_file.lambda_zip.output_base64sha256

  memory_size = 512
  timeout     = 60

  environment {
    variables = {
      ENVIRONMENT = var.environment
      REGION      = var.aws_region
    }
  }

  tracing_config {
    mode = "Active"
  }

  tags = {
    Name      = "${var.project_name}-${var.function_name}"
    ManagedBy = "terraform"
  }
}
```

### Container Image Deployment

```hcl
# ✅ Lambda Container Image (สำหรับ custom runtimes หรือ large packages)
resource "aws_lambda_function" "container_function" {
  function_name = "${var.project_name}-container-fn"
  role          = aws_iam_role.lambda_basic.arn

  # ✅ Package type Container
  package_type = "Image"
  image_uri    = "${aws_ecr_repository.lambda.repository_url}:${var.image_tag}"

  image_config {
    command = ["app.handler"]  # Override CMD
    # entry_point = ["/lambda-entrypoint.sh"]
    # working_directory = "/var/task"
  }

  memory_size                    = 1024
  timeout                        = 120
  reserved_concurrent_executions = 100

  environment {
    variables = {
      ENVIRONMENT = var.environment
    }
  }

  tracing_config {
    mode = "Active"
  }

  tags = {
    Name      = "${var.project_name}-container-fn"
    ManagedBy = "terraform"
  }
}
```

---

## Step 432: Lambda Layers

### aws_lambda_layer_version

```hcl
# ✅ สร้าง Lambda Layer สำหรับ shared libraries
data "archive_file" "dependencies_layer" {
  type        = "zip"
  source_dir  = "${path.module}/layers/dependencies"
  output_path = "${path.module}/dist/dependencies-layer.zip"
}

resource "aws_lambda_layer_version" "dependencies" {
  filename   = data.archive_file.dependencies_layer.output_path
  layer_name = "${var.project_name}-dependencies"

  compatible_runtimes = ["python3.11", "python3.10"]
  compatible_architectures = ["x86_64", "arm64"]

  source_code_hash = data.archive_file.dependencies_layer.output_base64sha256

  description = "Shared Python dependencies for ${var.project_name}"
}

# ✅ Lambda Layer สำหรับ Node.js
data "archive_file" "node_modules_layer" {
  type        = "zip"
  source_dir  = "${path.module}/layers/node_modules"
  output_path = "${path.module}/dist/node-modules-layer.zip"
}

resource "aws_lambda_layer_version" "node_modules" {
  filename   = data.archive_file.node_modules_layer.output_path
  layer_name = "${var.project_name}-node-modules"

  compatible_runtimes      = ["nodejs20.x", "nodejs18.x"]
  compatible_architectures = ["x86_64"]

  source_code_hash = data.archive_file.node_modules_layer.output_base64sha256
}

# ✅ Lambda Function ที่ใช้ Layer
resource "aws_lambda_function" "with_layer" {
  filename         = data.archive_file.lambda_zip.output_path
  function_name    = "${var.project_name}-with-layer"
  role             = aws_iam_role.lambda_basic.arn
  handler          = "app.handler"
  runtime          = "python3.11"
  source_code_hash = data.archive_file.lambda_zip.output_base64sha256

  memory_size = 256
  timeout     = 30

  # ✅ Attach layers
  layers = [
    aws_lambda_layer_version.dependencies.arn,
    # AWS managed layers
    # "arn:aws:lambda:ap-southeast-1:017000801446:layer:AWSLambdaPowertoolsPythonV2:46",
  ]

  tracing_config {
    mode = "Active"
  }

  tags = {
    Name      = "${var.project_name}-with-layer"
    ManagedBy = "terraform"
  }
}
```

---

## Step 433: Lambda VPC Configuration

```hcl
# ✅ Security Group สำหรับ Lambda ใน VPC
resource "aws_security_group" "lambda" {
  name        = "${var.project_name}-lambda-sg"
  description = "Security group for Lambda functions in VPC"
  vpc_id      = aws_vpc.main.id

  # ✅ Lambda ต้องการ outbound สำหรับ AWS services
  egress {
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
    description = "HTTPS to AWS services"
  }

  egress {
    from_port       = 5432
    to_port         = 5432
    protocol        = "tcp"
    security_groups = [aws_security_group.rds.id]
    description     = "PostgreSQL to RDS"
  }

  egress {
    from_port       = 6379
    to_port         = 6379
    protocol        = "tcp"
    security_groups = [aws_security_group.redis.id]
    description     = "Redis to ElastiCache"
  }

  tags = {
    Name      = "${var.project_name}-lambda-sg"
    ManagedBy = "terraform"
  }
}

# ✅ Lambda Function ใน VPC
resource "aws_lambda_function" "vpc_function" {
  filename         = data.archive_file.lambda_zip.output_path
  function_name    = "${var.project_name}-vpc-function"
  role             = aws_iam_role.lambda_vpc.arn
  handler          = "index.handler"
  runtime          = "nodejs20.x"
  source_code_hash = data.archive_file.lambda_zip.output_base64sha256

  memory_size = 256
  timeout     = 30

  # ✅ VPC configuration
  vpc_config {
    subnet_ids         = aws_subnet.private[*].id
    security_group_ids = [aws_security_group.lambda.id]
  }

  environment {
    variables = {
      DB_HOST     = aws_db_instance.postgres.address
      DB_PORT     = aws_db_instance.postgres.port
      DB_NAME     = aws_db_instance.postgres.db_name
      SECRET_ARN  = aws_secretsmanager_secret.db_credentials.arn
      REDIS_HOST  = aws_elasticache_replication_group.redis.primary_endpoint_address
    }
  }

  tracing_config {
    mode = "Active"
  }

  tags = {
    Name      = "${var.project_name}-vpc-function"
    ManagedBy = "terraform"
  }
}

# ✅ Lambda Role สำหรับ VPC access
resource "aws_iam_role" "lambda_vpc" {
  name               = "${var.project_name}-lambda-vpc-role"
  assume_role_policy = data.aws_iam_policy_document.lambda_trust.json
}

resource "aws_iam_role_policy_attachment" "lambda_vpc_execution" {
  role       = aws_iam_role.lambda_vpc.name
  policy_arn = "arn:aws:iam::aws:policy/service-role/AWSLambdaVPCAccessExecutionRole"
}
```

---

## Step 434: Lambda Concurrency

```hcl
# ✅ Reserved Concurrency - จำกัด instances สูงสุด
resource "aws_lambda_function_event_invoke_config" "api_handler" {
  function_name = aws_lambda_function.api_handler.function_name

  maximum_event_age_in_seconds = 300
  maximum_retry_attempts       = 2

  destination_config {
    on_failure {
      destination = aws_sqs_queue.lambda_dlq.arn
    }
    on_success {
      destination = aws_sns_topic.lambda_success.arn
    }
  }
}

resource "aws_lambda_function_event_invoke_config" "batch_processor" {
  function_name                = aws_lambda_function.batch_processor.function_name
  maximum_event_age_in_seconds = 3600  # 1 hour สำหรับ async
  maximum_retry_attempts       = 3
}

# ✅ Reserved Concurrency - แยก capacity สำหรับ critical function
resource "aws_lambda_function" "critical_function" {
  filename         = data.archive_file.lambda_zip.output_path
  function_name    = "${var.project_name}-critical"
  role             = aws_iam_role.lambda_basic.arn
  handler          = "handler.main"
  runtime          = "python3.11"
  source_code_hash = data.archive_file.lambda_zip.output_base64sha256

  memory_size = 512
  timeout     = 30

  # ✅ Reserved concurrency - จำกัดสูงสุด 50 concurrent executions
  reserved_concurrent_executions = 50

  tracing_config {
    mode = "Active"
  }

  tags = {
    Name      = "${var.project_name}-critical"
    ManagedBy = "terraform"
  }
}

# ✅ Provisioned Concurrency - ลด cold start
resource "aws_lambda_alias" "prod" {
  name             = "prod"
  function_name    = aws_lambda_function.api_handler.function_name
  function_version = aws_lambda_function.api_handler.version
}

resource "aws_lambda_provisioned_concurrency_config" "api_handler" {
  function_name                  = aws_lambda_function.api_handler.function_name
  qualifier                      = aws_lambda_alias.prod.name
  provisioned_concurrent_executions = 5  # Keep 5 instances warm

  # ✅ Auto scaling provisioned concurrency
  depends_on = [aws_lambda_alias.prod]
}
```

---

## Step 435: Event Sources

### SQS Event Source

```hcl
# ✅ SQS Queue สำหรับ Lambda trigger
resource "aws_sqs_queue" "lambda_trigger" {
  name                       = "${var.project_name}-lambda-trigger"
  visibility_timeout_seconds = 300  # ✅ ควรเท่ากับ Lambda timeout * 6
  message_retention_seconds  = 86400  # 1 day
  receive_wait_time_seconds  = 20  # Long polling

  # ✅ Dead Letter Queue
  redrive_policy = jsonencode({
    deadLetterTargetArn = aws_sqs_queue.lambda_dlq.arn
    maxReceiveCount     = 3
  })

  kms_master_key_id = "alias/${var.project_name}-sqs"

  tags = {
    Name      = "${var.project_name}-lambda-trigger"
    ManagedBy = "terraform"
  }
}

# ✅ Dead Letter Queue
resource "aws_sqs_queue" "lambda_dlq" {
  name                      = "${var.project_name}-lambda-dlq"
  message_retention_seconds = 1209600  # 14 days

  tags = {
    Name      = "${var.project_name}-lambda-dlq"
    ManagedBy = "terraform"
  }
}

# ✅ Lambda Event Source Mapping - SQS
resource "aws_lambda_event_source_mapping" "sqs_trigger" {
  event_source_arn = aws_sqs_queue.lambda_trigger.arn
  function_name    = aws_lambda_function.api_handler.arn

  batch_size                         = 10
  maximum_batching_window_in_seconds = 5

  # ✅ Filter messages
  filter_criteria {
    filter {
      pattern = jsonencode({
        body = {
          eventType = ["ORDER_CREATED", "ORDER_UPDATED"]
        }
      })
    }
  }

  # ✅ ปิด trigger เมื่อ error rate สูง
  function_response_types = ["ReportBatchItemFailures"]

  depends_on = [
    aws_iam_role_policy_attachment.lambda_sqs_policy,
  ]
}

# ✅ Policy สำหรับ SQS access
resource "aws_iam_role_policy" "lambda_sqs" {
  name = "sqs-access"
  role = aws_iam_role.lambda_basic.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Action = [
          "sqs:ReceiveMessage",
          "sqs:DeleteMessage",
          "sqs:GetQueueAttributes",
          "sqs:ChangeMessageVisibility",
        ]
        Resource = [
          aws_sqs_queue.lambda_trigger.arn,
        ]
      },
    ]
  })
}
```

### DynamoDB Streams

```hcl
# ✅ Lambda Event Source Mapping - DynamoDB Streams
resource "aws_lambda_event_source_mapping" "dynamodb_stream" {
  event_source_arn  = aws_dynamodb_table.orders.stream_arn
  function_name     = aws_lambda_function.stream_processor.arn
  starting_position = "LATEST"

  batch_size = 100
  bisect_batch_on_function_error = true

  # ✅ Retry settings
  maximum_retry_attempts = 3
  maximum_record_age_in_seconds = 3600

  # ✅ DLQ สำหรับ failed items
  destination_config {
    on_failure {
      destination_arn = aws_sqs_queue.dynamodb_stream_dlq.arn
    }
  }
}
```

### Kinesis Event Source

```hcl
# ✅ Lambda Event Source Mapping - Kinesis
resource "aws_lambda_event_source_mapping" "kinesis_trigger" {
  event_source_arn  = aws_kinesis_stream.events.arn
  function_name     = aws_lambda_function.kinesis_processor.arn
  starting_position = "TRIM_HORIZON"

  batch_size = 100
  maximum_batching_window_in_seconds = 5

  # ✅ Parallel processing
  parallelization_factor = 5  # 5 concurrent processors per shard

  bisect_batch_on_function_error = true
  maximum_retry_attempts = 3

  destination_config {
    on_failure {
      destination_arn = aws_sqs_queue.kinesis_dlq.arn
    }
  }
}
```

### Lambda Permission สำหรับ API Gateway

```hcl
# ✅ Lambda Permission - API Gateway
resource "aws_lambda_permission" "api_gateway" {
  statement_id  = "AllowAPIGatewayInvoke"
  action        = "lambda:InvokeFunction"
  function_name = aws_lambda_function.api_handler.function_name
  principal     = "apigateway.amazonaws.com"

  # ✅ จำกัด source ARN
  source_arn = "${aws_api_gateway_rest_api.main.execution_arn}/*/*"
}

# ✅ Lambda Permission - S3
resource "aws_lambda_permission" "s3_trigger" {
  statement_id  = "AllowS3Invoke"
  action        = "lambda:InvokeFunction"
  function_name = aws_lambda_function.s3_processor.function_name
  principal     = "s3.amazonaws.com"
  source_arn    = aws_s3_bucket.uploads.arn
  # ✅ source_account ป้องกัน confused deputy
  source_account = data.aws_caller_identity.current.account_id
}

# ✅ S3 trigger
resource "aws_s3_bucket_notification" "upload_trigger" {
  bucket = aws_s3_bucket.uploads.id

  lambda_function {
    lambda_function_arn = aws_lambda_function.s3_processor.arn
    events              = ["s3:ObjectCreated:*"]
    filter_prefix       = "uploads/"
    filter_suffix       = ".jpg"
  }

  depends_on = [aws_lambda_permission.s3_trigger]
}
```

### EventBridge (CloudWatch Events)

```hcl
# ✅ Scheduled Lambda ด้วย EventBridge
resource "aws_cloudwatch_event_rule" "daily_report" {
  name                = "${var.project_name}-daily-report"
  description         = "Trigger daily report generation"
  schedule_expression = "cron(0 8 * * ? *)"  # ทุกวัน 8:00 UTC

  tags = {
    Name      = "${var.project_name}-daily-report"
    ManagedBy = "terraform"
  }
}

resource "aws_cloudwatch_event_target" "daily_report_lambda" {
  rule      = aws_cloudwatch_event_rule.daily_report.name
  target_id = "DailyReportLambda"
  arn       = aws_lambda_function.report_generator.arn

  input = jsonencode({
    reportType = "DAILY"
    format     = "PDF"
  })
}

resource "aws_lambda_permission" "eventbridge" {
  statement_id  = "AllowEventBridgeInvoke"
  action        = "lambda:InvokeFunction"
  function_name = aws_lambda_function.report_generator.function_name
  principal     = "events.amazonaws.com"
  source_arn    = aws_cloudwatch_event_rule.daily_report.arn
}

# ✅ Event Pattern trigger
resource "aws_cloudwatch_event_rule" "ec2_state_change" {
  name        = "${var.project_name}-ec2-state-change"
  description = "Trigger on EC2 state changes"

  event_pattern = jsonencode({
    source      = ["aws.ec2"]
    detail-type = ["EC2 Instance State-change Notification"]
    detail = {
      state = ["running", "stopped", "terminated"]
    }
  })
}

resource "aws_cloudwatch_event_target" "ec2_state_lambda" {
  rule      = aws_cloudwatch_event_rule.ec2_state_change.name
  target_id = "EC2StateLambda"
  arn       = aws_lambda_function.ec2_monitor.arn
}

resource "aws_lambda_permission" "ec2_state_change" {
  statement_id  = "AllowEC2StateChangeTrigger"
  action        = "lambda:InvokeFunction"
  function_name = aws_lambda_function.ec2_monitor.function_name
  principal     = "events.amazonaws.com"
  source_arn    = aws_cloudwatch_event_rule.ec2_state_change.arn
}
```

---

## Step 436: Lambda Aliases และ Versions

```hcl
# ✅ Lambda Version (เปิดใช้งานด้วย publish = true)
resource "aws_lambda_function" "versioned_function" {
  filename         = data.archive_file.lambda_zip.output_path
  function_name    = "${var.project_name}-versioned"
  role             = aws_iam_role.lambda_basic.arn
  handler          = "handler.main"
  runtime          = "python3.11"
  source_code_hash = data.archive_file.lambda_zip.output_base64sha256

  memory_size = 256
  timeout     = 30

  # ✅ เปิด versioning
  publish = true

  tracing_config {
    mode = "Active"
  }

  tags = {
    Name      = "${var.project_name}-versioned"
    ManagedBy = "terraform"
  }
}

# ✅ Lambda Alias
resource "aws_lambda_alias" "production" {
  name             = "production"
  description      = "Production alias pointing to stable version"
  function_name    = aws_lambda_function.versioned_function.function_name
  function_version = aws_lambda_function.versioned_function.version

  # ✅ Canary deployment - ส่ง 10% traffic ไป new version
  # routing_config {
  #   additional_version_weights = {
  #     "2" = 0.1  # 10% to version 2
  #   }
  # }
}

resource "aws_lambda_alias" "staging" {
  name             = "staging"
  description      = "Staging alias for testing"
  function_name    = aws_lambda_function.versioned_function.function_name
  function_version = "$LATEST"
}

# ✅ CodeDeploy สำหรับ Gradual deployment
resource "aws_codedeploy_deployment_group" "lambda" {
  app_name               = aws_codedeploy_app.lambda.name
  deployment_group_name  = "${var.project_name}-lambda-deploy"
  service_role_arn       = aws_iam_role.codedeploy.arn
  deployment_config_name = "CodeDeployDefault.LambdaCanary10Percent5Minutes"

  deployment_style {
    deployment_option = "WITH_TRAFFIC_CONTROL"
    deployment_type   = "BLUE_GREEN"
  }
}
```

---

## Step 437: CloudWatch Logs สำหรับ Lambda

```hcl
# ✅ สร้าง Log Group ก่อน Lambda function
resource "aws_cloudwatch_log_group" "lambda_api_handler" {
  name              = "/aws/lambda/${aws_lambda_function.api_handler.function_name}"
  retention_in_days = 30  # ✅ ตั้ง retention เสมอ

  # ✅ Encrypt logs
  kms_key_id = aws_kms_key.cloudwatch.arn

  tags = {
    Name      = "${var.project_name}-lambda-logs"
    ManagedBy = "terraform"
  }
}

resource "aws_cloudwatch_log_group" "lambda_vpc_function" {
  name              = "/aws/lambda/${aws_lambda_function.vpc_function.function_name}"
  retention_in_days = 30
  kms_key_id        = aws_kms_key.cloudwatch.arn

  tags = {
    Name      = "${var.project_name}-vpc-fn-logs"
    ManagedBy = "terraform"
  }
}

# ✅ CloudWatch Metric Alarm สำหรับ Lambda errors
resource "aws_cloudwatch_metric_alarm" "lambda_errors" {
  alarm_name          = "${var.project_name}-lambda-errors"
  comparison_operator = "GreaterThanOrEqualToThreshold"
  evaluation_periods  = "2"
  metric_name         = "Errors"
  namespace           = "AWS/Lambda"
  period              = "60"
  statistic           = "Sum"
  threshold           = "5"
  alarm_description   = "Lambda function error rate is too high"
  alarm_actions       = [aws_sns_topic.alerts.arn]

  dimensions = {
    FunctionName = aws_lambda_function.api_handler.function_name
  }

  tags = {
    Name      = "${var.project_name}-lambda-errors-alarm"
    ManagedBy = "terraform"
  }
}

# ✅ Lambda Duration alarm
resource "aws_cloudwatch_metric_alarm" "lambda_duration" {
  alarm_name          = "${var.project_name}-lambda-duration"
  comparison_operator = "GreaterThanOrEqualToThreshold"
  evaluation_periods  = "3"
  metric_name         = "Duration"
  namespace           = "AWS/Lambda"
  period              = "60"
  statistic           = "p99"
  threshold           = "25000"  # 25 seconds (timeout = 30s)
  alarm_description   = "Lambda P99 duration approaching timeout"
  alarm_actions       = [aws_sns_topic.alerts.arn]

  dimensions = {
    FunctionName = aws_lambda_function.api_handler.function_name
  }
}
```

---

## Step 438: Lambda Function URL

```hcl
# ✅ Lambda Function URL (ไม่ต้องใช้ API Gateway)
resource "aws_lambda_function_url" "api_handler" {
  function_name      = aws_lambda_function.api_handler.function_name
  authorization_type = "AWS_IAM"  # ✅ ใช้ IAM auth เสมอ

  # สำหรับ public endpoint (ระวัง!)
  # authorization_type = "NONE"

  cors {
    allow_credentials = true
    allow_headers     = ["content-type", "authorization"]
    allow_methods     = ["GET", "POST", "PUT", "DELETE"]
    allow_origins     = ["https://app.example.com"]
    expose_headers    = ["x-request-id"]
    max_age           = 86400
  }
}

# ✅ Function URL สำหรับ Alias
resource "aws_lambda_function_url" "api_handler_prod" {
  function_name      = aws_lambda_function.api_handler.function_name
  qualifier          = aws_lambda_alias.production.name
  authorization_type = "AWS_IAM"
}

output "function_url" {
  description = "Lambda function URL"
  value       = aws_lambda_function_url.api_handler.function_url
}
```

---

## Step 439: Lambda Secrets Manager Integration

```hcl
# ✅ Lambda ที่ใช้ Secrets Manager
resource "aws_lambda_function" "secure_function" {
  filename         = data.archive_file.lambda_zip.output_path
  function_name    = "${var.project_name}-secure-fn"
  role             = aws_iam_role.lambda_with_secrets.arn
  handler          = "index.handler"
  runtime          = "nodejs20.x"
  source_code_hash = data.archive_file.lambda_zip.output_base64sha256

  memory_size = 512
  timeout     = 30

  environment {
    variables = {
      # ✅ ใส่แค่ ARN, ไม่ใส่ secret value
      DB_SECRET_ARN   = aws_secretsmanager_secret.db_credentials.arn
      API_SECRET_ARN  = aws_secretsmanager_secret.api_key.arn
      ENVIRONMENT     = var.environment
    }
  }

  tracing_config {
    mode = "Active"
  }

  tags = {
    Name      = "${var.project_name}-secure-fn"
    ManagedBy = "terraform"
  }
}

# ✅ IAM Policy สำหรับ Secrets Manager access
resource "aws_iam_role" "lambda_with_secrets" {
  name               = "${var.project_name}-lambda-secrets-role"
  assume_role_policy = data.aws_iam_policy_document.lambda_trust.json
}

resource "aws_iam_role_policy_attachment" "lambda_secrets_basic" {
  role       = aws_iam_role.lambda_with_secrets.name
  policy_arn = "arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole"
}

resource "aws_iam_role_policy" "lambda_secrets_access" {
  name = "secrets-manager-access"
  role = aws_iam_role.lambda_with_secrets.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Action = [
          "secretsmanager:GetSecretValue",
        ]
        Resource = [
          aws_secretsmanager_secret.db_credentials.arn,
          aws_secretsmanager_secret.api_key.arn,
        ]
      },
      {
        Effect = "Allow"
        Action = [
          "kms:Decrypt",
        ]
        Resource = [
          aws_kms_key.secrets.arn,
        ]
      },
    ]
  })
}
```

---

## Step 440: Lambda@Edge

```hcl
# ✅ Lambda@Edge ต้อง deploy ใน us-east-1
provider "aws" {
  alias  = "us_east_1"
  region = "us-east-1"
}

resource "aws_iam_role" "lambda_edge" {
  provider           = aws.us_east_1
  name               = "${var.project_name}-lambda-edge-role"
  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Principal = {
          Service = [
            "lambda.amazonaws.com",
            "edgelambda.amazonaws.com",
          ]
        }
        Action = "sts:AssumeRole"
      }
    ]
  })
}

resource "aws_iam_role_policy_attachment" "lambda_edge_basic" {
  provider   = aws.us_east_1
  role       = aws_iam_role.lambda_edge.name
  policy_arn = "arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole"
}

# ✅ Lambda@Edge Function (ต้อง us-east-1)
resource "aws_lambda_function" "edge_viewer_request" {
  provider = aws.us_east_1

  filename         = data.archive_file.edge_zip.output_path
  function_name    = "${var.project_name}-edge-viewer-request"
  role             = aws_iam_role.lambda_edge.arn
  handler          = "index.handler"
  runtime          = "nodejs20.x"
  source_code_hash = data.archive_file.edge_zip.output_base64sha256

  # ✅ Lambda@Edge ต้องมี publish = true
  publish = true

  # Lambda@Edge constraints:
  # - memory_size: max 128 MB (viewer events) / 10240 MB (origin events)
  # - timeout: max 5s (viewer events) / 30s (origin events)
  memory_size = 128
  timeout     = 5

  tags = {
    Name      = "${var.project_name}-edge-viewer-request"
    ManagedBy = "terraform"
  }
}

# ✅ ใช้ Lambda@Edge ใน CloudFront distribution
resource "aws_cloudfront_distribution" "with_edge" {
  # ... other config

  default_cache_behavior {
    # ...

    lambda_function_association {
      event_type   = "viewer-request"
      lambda_arn   = aws_lambda_function.edge_viewer_request.qualified_arn
      include_body = false
    }

    # event_type options:
    # - viewer-request: ก่อน forward ไป origin (max 5s, 128MB)
    # - origin-request: ก่อน request ถึง origin (max 30s, 10GB)
    # - origin-response: หลัง origin ตอบ
    # - viewer-response: ก่อนส่งให้ viewer
  }

  # ...
}
```

---

## ตัวอย่าง Complete Event-Driven Pattern

```hcl
# ✅ Complete: S3 Upload -> Lambda -> DynamoDB -> SNS

# 1. S3 Bucket สำหรับ uploads
resource "aws_s3_bucket" "uploads" {
  bucket = "${var.project_name}-uploads"

  tags = {
    Name      = "${var.project_name}-uploads"
    ManagedBy = "terraform"
  }
}

resource "aws_s3_bucket_server_side_encryption_configuration" "uploads" {
  bucket = aws_s3_bucket.uploads.id
  rule {
    apply_server_side_encryption_by_default {
      sse_algorithm = "aws:kms"
    }
  }
}

# 2. Lambda Function สำหรับ process uploads
resource "aws_lambda_function" "upload_processor" {
  filename         = data.archive_file.lambda_zip.output_path
  function_name    = "${var.project_name}-upload-processor"
  role             = aws_iam_role.upload_processor.arn
  handler          = "processor.handler"
  runtime          = "python3.11"
  source_code_hash = data.archive_file.lambda_zip.output_base64sha256

  memory_size = 1024  # เพิ่ม memory สำหรับ image processing
  timeout     = 300   # 5 minutes

  environment {
    variables = {
      DYNAMODB_TABLE = aws_dynamodb_table.uploads.name
      SNS_TOPIC_ARN  = aws_sns_topic.upload_complete.arn
      ENVIRONMENT    = var.environment
    }
  }

  tracing_config {
    mode = "Active"
  }

  tags = {
    Name      = "${var.project_name}-upload-processor"
    ManagedBy = "terraform"
  }
}

# 3. Lambda Permission
resource "aws_lambda_permission" "s3_upload_trigger" {
  statement_id  = "AllowS3UploadTrigger"
  action        = "lambda:InvokeFunction"
  function_name = aws_lambda_function.upload_processor.function_name
  principal     = "s3.amazonaws.com"
  source_arn    = aws_s3_bucket.uploads.arn
  source_account = data.aws_caller_identity.current.account_id
}

# 4. S3 Notification
resource "aws_s3_bucket_notification" "upload_trigger" {
  bucket = aws_s3_bucket.uploads.id

  lambda_function {
    lambda_function_arn = aws_lambda_function.upload_processor.arn
    events              = ["s3:ObjectCreated:*"]
  }

  depends_on = [aws_lambda_permission.s3_upload_trigger]
}

# 5. DynamoDB Table
resource "aws_dynamodb_table" "uploads" {
  name         = "${var.project_name}-uploads"
  billing_mode = "PAY_PER_REQUEST"
  hash_key     = "id"
  range_key    = "timestamp"

  attribute {
    name = "id"
    type = "S"
  }

  attribute {
    name = "timestamp"
    type = "S"
  }

  server_side_encryption {
    enabled = true
  }

  point_in_time_recovery {
    enabled = true
  }

  tags = {
    Name      = "${var.project_name}-uploads"
    ManagedBy = "terraform"
  }
}

# 6. SNS Topic
resource "aws_sns_topic" "upload_complete" {
  name              = "${var.project_name}-upload-complete"
  kms_master_key_id = aws_kms_key.sns.id

  tags = {
    Name      = "${var.project_name}-upload-complete"
    ManagedBy = "terraform"
  }
}

# 7. IAM Role สำหรับ upload processor
resource "aws_iam_role" "upload_processor" {
  name               = "${var.project_name}-upload-processor-role"
  assume_role_policy = data.aws_iam_policy_document.lambda_trust.json
}

resource "aws_iam_role_policy" "upload_processor" {
  name = "upload-processor-policy"
  role = aws_iam_role.upload_processor.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect   = "Allow"
        Action   = ["logs:CreateLogGroup", "logs:CreateLogStream", "logs:PutLogEvents"]
        Resource = ["arn:aws:logs:*:*:*"]
      },
      {
        Effect   = "Allow"
        Action   = ["s3:GetObject"]
        Resource = ["${aws_s3_bucket.uploads.arn}/*"]
      },
      {
        Effect   = "Allow"
        Action   = ["dynamodb:PutItem", "dynamodb:UpdateItem"]
        Resource = [aws_dynamodb_table.uploads.arn]
      },
      {
        Effect   = "Allow"
        Action   = ["sns:Publish"]
        Resource = [aws_sns_topic.upload_complete.arn]
      },
    ]
  })
}
```

---

## Lambda Best Practices สรุป

### ✅ Performance

1. **Memory Optimization**: Lambda CPU scale ตาม memory - เพิ่ม memory = เพิ่ม CPU
2. **Provisioned Concurrency**: ใช้กับ latency-sensitive functions
3. **Layer Reuse**: แยก dependencies เป็น layer
4. **Connection Pooling**: Reuse connections outside handler

### ✅ Security

1. **Least Privilege IAM**: ให้สิทธิ์เฉพาะที่จำเป็น
2. **VPC**: ใส่ Lambda ใน VPC เมื่อต้องการเข้าถึง private resources
3. **Secrets Manager**: อย่า hardcode secrets
4. **Environment Variables Encryption**: ใช้ KMS
5. **IMDSv2**: บังคับใช้สำหรับ EC2-backed resources

### ✅ Monitoring

1. **X-Ray Tracing**: เปิดเสมอ
2. **CloudWatch Logs**: ตั้ง retention period
3. **Custom Metrics**: ส่ง business metrics
4. **Alarms**: ตั้ง error rate, duration, throttle alarms
5. **DLQ**: ใช้สำหรับ async invocations

---

**Next Steps**: ไปต่อที่ Part 045 - AWS CloudFront CDN
