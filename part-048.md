# Part 048: AWS SQS, SNS & Event-Driven Architecture
## การสร้าง Event-Driven Systems ด้วย Terraform (Steps 471-480)

---

## บทนำ (Introduction)

Event-driven architecture ช่วยให้ components ทำงานแบบ loosely coupled ผ่าน messaging services
AWS มี SQS (Simple Queue Service) และ SNS (Simple Notification Service) เป็น building blocks หลัก

**หัวข้อที่จะเรียนรู้:**
- SQS Standard และ FIFO Queues
- Dead Letter Queues (DLQ)
- SNS Topics และ Subscriptions
- SNS Filtering Policies
- EventBridge Rules และ Targets
- Fan-out Pattern (SNS -> SQS)
- Encryption สำหรับ Messaging
- Security Best Practices

---

## Step 471: SQS Standard Queue

### aws_sqs_queue

```hcl
# ✅ Secure: SQS Standard Queue
resource "aws_sqs_queue" "standard" {
  name = "${var.project_name}-standard-queue"

  # ✅ Visibility Timeout (ควรเป็น 6x Lambda timeout)
  visibility_timeout_seconds = 180  # 3 minutes

  # Message retention (1 min - 14 days)
  message_retention_seconds = 86400  # 1 day

  # Delay delivery
  delay_seconds = 0

  # Max message size (1KB - 256KB)
  max_message_size = 262144  # 256KB

  # ✅ Long polling
  receive_wait_time_seconds = 20  # Long polling

  # ✅ Dead Letter Queue
  redrive_policy = jsonencode({
    deadLetterTargetArn = aws_sqs_queue.standard_dlq.arn
    maxReceiveCount     = 3
  })

  # ✅ Encryption
  sqs_managed_sse_enabled = true  # SSE-SQS (ฟรี)
  # หรือใช้ KMS:
  # kms_master_key_id                 = aws_kms_key.sqs.arn
  # kms_data_key_reuse_period_seconds = 300

  tags = {
    Name        = "${var.project_name}-standard-queue"
    Environment = var.environment
    ManagedBy   = "terraform"
  }
}

# ✅ Dead Letter Queue
resource "aws_sqs_queue" "standard_dlq" {
  name = "${var.project_name}-standard-dlq"

  message_retention_seconds = 1209600  # 14 days - เก็บ failed messages นานขึ้น

  # ✅ DLQ ก็ควร encrypt
  sqs_managed_sse_enabled = true

  tags = {
    Name      = "${var.project_name}-standard-dlq"
    ManagedBy = "terraform"
  }
}

# ✅ Redrive Allow Policy สำหรับ DLQ
resource "aws_sqs_queue_redrive_allow_policy" "standard_dlq" {
  queue_url = aws_sqs_queue.standard_dlq.id

  redrive_allow_policy = jsonencode({
    redrivePermission = "byQueue"
    sourceQueueArns   = [aws_sqs_queue.standard.arn]
  })
}

# ❌ Insecure Queue
resource "aws_sqs_queue" "insecure" {
  name = "insecure-queue"
  # ❌ ไม่มี encryption
  # ❌ ไม่มี DLQ
  # ❌ ไม่มี Long polling
}
```

---

## Step 472: SQS FIFO Queue

```hcl
# ✅ FIFO Queue (First-In-First-Out)
resource "aws_sqs_queue" "fifo" {
  # ✅ FIFO queues ต้องลงท้ายด้วย .fifo
  name = "${var.project_name}-orders.fifo"

  fifo_queue                  = true
  content_based_deduplication = true  # ✅ Auto dedup based on body hash

  # FIFO Settings
  deduplication_scope        = "messageGroup"  # หรือ "queue"
  fifo_throughput_limit      = "perMessageGroupId"  # หรือ "perQueue"

  visibility_timeout_seconds = 30
  message_retention_seconds  = 86400
  receive_wait_time_seconds  = 20

  # ✅ Dead Letter Queue (ก็ต้องเป็น FIFO)
  redrive_policy = jsonencode({
    deadLetterTargetArn = aws_sqs_queue.fifo_dlq.arn
    maxReceiveCount     = 3
  })

  # ✅ Encryption
  sqs_managed_sse_enabled = true

  tags = {
    Name      = "${var.project_name}-orders-fifo"
    ManagedBy = "terraform"
  }
}

# ✅ FIFO Dead Letter Queue
resource "aws_sqs_queue" "fifo_dlq" {
  name      = "${var.project_name}-orders-dlq.fifo"
  fifo_queue = true

  message_retention_seconds = 1209600
  sqs_managed_sse_enabled   = true

  tags = {
    Name      = "${var.project_name}-orders-dlq"
    ManagedBy = "terraform"
  }
}
```

---

## Step 473: SQS Queue Policies

```hcl
# ✅ SQS Queue Policy - อนุญาต SNS publish
data "aws_iam_policy_document" "sqs_policy" {
  statement {
    sid    = "AllowSNSPublish"
    effect = "Allow"

    principals {
      type        = "Service"
      identifiers = ["sns.amazonaws.com"]
    }

    actions   = ["sqs:SendMessage"]
    resources = [aws_sqs_queue.standard.arn]

    # ✅ จำกัดเฉพาะ SNS topic ที่รู้จัก
    condition {
      test     = "ArnEquals"
      variable = "aws:SourceArn"
      values   = [aws_sns_topic.events.arn]
    }
  }

  # ✅ อนุญาตเฉพาะ HTTPS
  statement {
    sid    = "DenyNonHTTPS"
    effect = "Deny"

    principals {
      type        = "*"
      identifiers = ["*"]
    }

    actions   = ["sqs:*"]
    resources = [aws_sqs_queue.standard.arn]

    condition {
      test     = "Bool"
      variable = "aws:SecureTransport"
      values   = ["false"]
    }
  }
}

resource "aws_sqs_queue_policy" "standard" {
  queue_url = aws_sqs_queue.standard.id
  policy    = data.aws_iam_policy_document.sqs_policy.json
}

# ✅ SQS Policy สำหรับ Cross-Account Access
data "aws_iam_policy_document" "sqs_cross_account_policy" {
  statement {
    sid    = "AllowCrossAccountAccess"
    effect = "Allow"

    principals {
      type        = "AWS"
      identifiers = ["arn:aws:iam::${var.producer_account_id}:root"]
    }

    actions = ["sqs:SendMessage"]

    resources = [aws_sqs_queue.standard.arn]

    # ✅ จำกัดเฉพาะ specific role
    condition {
      test     = "ArnLike"
      variable = "aws:PrincipalArn"
      values   = ["arn:aws:iam::${var.producer_account_id}:role/producer-role"]
    }
  }
}
```

---

## Step 474: SNS Topics

### aws_sns_topic

```hcl
# ✅ KMS Key สำหรับ SNS
resource "aws_kms_key" "sns" {
  description             = "KMS key for SNS - ${var.project_name}"
  deletion_window_in_days = 14
  enable_key_rotation     = true

  tags = {
    Name      = "${var.project_name}-sns-kms"
    ManagedBy = "terraform"
  }
}

# ✅ SNS Standard Topic
resource "aws_sns_topic" "events" {
  name              = "${var.project_name}-events"
  display_name      = "${var.project_name} Events"
  kms_master_key_id = aws_kms_key.sns.id  # ✅ Encryption

  # ✅ Delivery policy
  delivery_policy = jsonencode({
    http = {
      defaultHealthyRetryPolicy = {
        minDelayTarget      = 20
        maxDelayTarget      = 20
        numRetries          = 3
        numMaxDelayRetries  = 0
        numNoDelayRetries   = 0
        numMinDelayRetries  = 0
        backoffFunction     = "linear"
      }
      disableSubscriptionOverrides = false
    }
  })

  tags = {
    Name      = "${var.project_name}-events"
    ManagedBy = "terraform"
  }
}

# ✅ SNS FIFO Topic
resource "aws_sns_topic" "ordered_events" {
  # ✅ FIFO topics ต้องลงท้ายด้วย .fifo
  name                        = "${var.project_name}-ordered.fifo"
  fifo_topic                  = true
  content_based_deduplication = true
  kms_master_key_id           = aws_kms_key.sns.id

  tags = {
    Name      = "${var.project_name}-ordered-events"
    ManagedBy = "terraform"
  }
}
```

### SNS Topic Policy

```hcl
# ✅ SNS Topic Policy
data "aws_iam_policy_document" "sns_topic_policy" {
  # ✅ Allow services to publish
  statement {
    sid    = "AllowPublishFromAccount"
    effect = "Allow"

    principals {
      type        = "AWS"
      identifiers = ["arn:aws:iam::${data.aws_caller_identity.current.account_id}:root"]
    }

    actions   = ["sns:Publish", "sns:Subscribe"]
    resources = [aws_sns_topic.events.arn]
  }

  # ✅ Allow EventBridge to publish
  statement {
    sid    = "AllowEventBridgePublish"
    effect = "Allow"

    principals {
      type        = "Service"
      identifiers = ["events.amazonaws.com"]
    }

    actions   = ["sns:Publish"]
    resources = [aws_sns_topic.events.arn]

    condition {
      test     = "ArnEquals"
      variable = "AWS:SourceAccount"
      values   = [data.aws_caller_identity.current.account_id]
    }
  }

  # ✅ Allow CloudWatch Alarms to publish
  statement {
    sid    = "AllowCloudWatchAlarms"
    effect = "Allow"

    principals {
      type        = "Service"
      identifiers = ["cloudwatch.amazonaws.com"]
    }

    actions   = ["sns:Publish"]
    resources = [aws_sns_topic.events.arn]
  }

  # ✅ Deny non-HTTPS
  statement {
    sid    = "DenyNonHTTPS"
    effect = "Deny"

    principals {
      type        = "*"
      identifiers = ["*"]
    }

    actions   = ["sns:*"]
    resources = [aws_sns_topic.events.arn]

    condition {
      test     = "Bool"
      variable = "aws:SecureTransport"
      values   = ["false"]
    }
  }
}

resource "aws_sns_topic_policy" "events" {
  arn    = aws_sns_topic.events.arn
  policy = data.aws_iam_policy_document.sns_topic_policy.json
}
```

---

## Step 475: SNS Subscriptions

### aws_sns_topic_subscription

```hcl
# ✅ Subscribe SQS ไปยัง SNS
resource "aws_sns_topic_subscription" "events_to_sqs" {
  topic_arn = aws_sns_topic.events.arn
  protocol  = "sqs"
  endpoint  = aws_sqs_queue.standard.arn

  # ✅ ตั้ง raw message delivery (ไม่ wrap ด้วย SNS envelope)
  raw_message_delivery = true

  # ✅ Filter policy - รับเฉพาะ event types ที่ต้องการ
  filter_policy = jsonencode({
    eventType = ["ORDER_CREATED", "ORDER_UPDATED"]
  })

  filter_policy_scope = "MessageAttributes"  # หรือ "MessageBody"
}

# ✅ Subscribe Lambda ไปยัง SNS
resource "aws_sns_topic_subscription" "events_to_lambda" {
  topic_arn = aws_sns_topic.events.arn
  protocol  = "lambda"
  endpoint  = aws_lambda_function.event_processor.arn

  filter_policy = jsonencode({
    eventType = ["PAYMENT_COMPLETED"]
  })
}

resource "aws_lambda_permission" "sns_invoke" {
  statement_id  = "AllowSNSInvoke"
  action        = "lambda:InvokeFunction"
  function_name = aws_lambda_function.event_processor.function_name
  principal     = "sns.amazonaws.com"
  source_arn    = aws_sns_topic.events.arn
}

# ✅ Subscribe Email ไปยัง SNS (สำหรับ alerts)
resource "aws_sns_topic_subscription" "alerts_email" {
  topic_arn = aws_sns_topic.alerts.arn
  protocol  = "email"
  endpoint  = var.alert_email
}

# ✅ Subscribe HTTPS Webhook
resource "aws_sns_topic_subscription" "webhook" {
  topic_arn              = aws_sns_topic.events.arn
  protocol               = "https"
  endpoint               = "https://webhook.example.com/sns"
  endpoint_auto_confirms = true

  delivery_policy = jsonencode({
    healthyRetryPolicy = {
      numRetries         = 20
      minDelayTarget     = 20
      maxDelayTarget     = 600
      numMaxDelayRetries = 5
      backoffFunction    = "exponential"
    }
  })
}
```

### SNS Filtering Policies

```hcl
# ✅ Advanced Filtering - Order Processing System
resource "aws_sqs_queue" "orders_processing" {
  name                    = "${var.project_name}-orders-processing"
  sqs_managed_sse_enabled = true
}

resource "aws_sns_topic_subscription" "orders_processing" {
  topic_arn = aws_sns_topic.events.arn
  protocol  = "sqs"
  endpoint  = aws_sqs_queue.orders_processing.arn

  # ✅ Complex filter policy
  filter_policy = jsonencode({
    eventType = ["ORDER_CREATED"]
    status    = ["pending", "confirmed"]
    amount    = [{ numeric = [">=", 100] }]  # Amount >= 100
  })

  filter_policy_scope = "MessageBody"  # Filter ใน message body
}

# ✅ Filter โดยใช้ MessageBody
resource "aws_sns_topic_subscription" "high_value_orders" {
  topic_arn = aws_sns_topic.events.arn
  protocol  = "sqs"
  endpoint  = aws_sqs_queue.high_value.arn

  filter_policy = jsonencode({
    eventType = [{ "prefix" = "ORDER" }]  # ทุก event ที่ขึ้นต้นด้วย "ORDER"
    amount    = [{ numeric = [">=", 10000] }]  # High value orders
  })

  filter_policy_scope = "MessageBody"
}
```

---

## Step 476: EventBridge (CloudWatch Events)

### aws_cloudwatch_event_bus

```hcl
# ✅ Custom Event Bus
resource "aws_cloudwatch_event_bus" "main" {
  name = "${var.project_name}-event-bus"

  tags = {
    Name      = "${var.project_name}-event-bus"
    ManagedBy = "terraform"
  }
}

# ✅ Event Bus Policy
data "aws_iam_policy_document" "event_bus_policy" {
  statement {
    sid    = "AllowPublishFromAccount"
    effect = "Allow"

    principals {
      type        = "AWS"
      identifiers = ["arn:aws:iam::${data.aws_caller_identity.current.account_id}:root"]
    }

    actions   = ["events:PutEvents"]
    resources = [aws_cloudwatch_event_bus.main.arn]
  }

  # ✅ Allow cross-account publishing
  statement {
    sid    = "AllowCrossAccountPublish"
    effect = "Allow"

    principals {
      type        = "AWS"
      identifiers = ["arn:aws:iam::${var.producer_account_id}:root"]
    }

    actions   = ["events:PutEvents"]
    resources = [aws_cloudwatch_event_bus.main.arn]
  }
}

resource "aws_cloudwatch_event_bus_policy" "main" {
  event_bus_name = aws_cloudwatch_event_bus.main.name
  policy         = data.aws_iam_policy_document.event_bus_policy.json
}
```

### Scheduled Events

```hcl
# ✅ Schedule - Cron expression
resource "aws_cloudwatch_event_rule" "daily_cleanup" {
  name                = "${var.project_name}-daily-cleanup"
  description         = "Daily cleanup job"
  schedule_expression = "cron(0 2 * * ? *)"  # ทุกวัน 2:00 AM UTC

  # ✅ หรือใช้ rate expression
  # schedule_expression = "rate(6 hours)"

  tags = {
    Name      = "${var.project_name}-daily-cleanup"
    ManagedBy = "terraform"
  }
}

resource "aws_cloudwatch_event_target" "cleanup_lambda" {
  rule      = aws_cloudwatch_event_rule.daily_cleanup.name
  target_id = "CleanupLambda"
  arn       = aws_lambda_function.cleanup.arn

  # ✅ Input transformation
  input_transformer {
    input_paths = {
      time = "$.time"
    }
    input_template = <<-EOT
      {
        "jobType": "CLEANUP",
        "timestamp": <time>,
        "environment": "${var.environment}"
      }
    EOT
  }
}

resource "aws_lambda_permission" "cleanup" {
  statement_id  = "AllowEventBridgeInvoke"
  action        = "lambda:InvokeFunction"
  function_name = aws_lambda_function.cleanup.function_name
  principal     = "events.amazonaws.com"
  source_arn    = aws_cloudwatch_event_rule.daily_cleanup.arn
}

# ✅ เพิ่ม retry settings
resource "aws_cloudwatch_event_target" "cleanup_lambda_with_retry" {
  rule      = aws_cloudwatch_event_rule.daily_cleanup.name
  target_id = "CleanupLambdaWithRetry"
  arn       = aws_lambda_function.cleanup.arn

  retry_policy {
    maximum_event_age_in_seconds = 3600  # 1 hour
    maximum_retry_attempts       = 3
  }

  dead_letter_config {
    arn = aws_sqs_queue.eventbridge_dlq.arn
  }
}
```

### Event Pattern Rules

```hcl
# ✅ Event Pattern - EC2 State Changes
resource "aws_cloudwatch_event_rule" "ec2_terminate" {
  name        = "${var.project_name}-ec2-terminate"
  description = "Trigger on EC2 termination"

  event_pattern = jsonencode({
    source      = ["aws.ec2"]
    detail-type = ["EC2 Instance State-change Notification"]
    detail = {
      state = ["terminated"]
    }
  })
}

# ✅ Event Pattern - CodePipeline
resource "aws_cloudwatch_event_rule" "pipeline_failed" {
  name        = "${var.project_name}-pipeline-failed"
  description = "Alert on pipeline failures"

  event_pattern = jsonencode({
    source      = ["aws.codepipeline"]
    detail-type = ["CodePipeline Pipeline Execution State Change"]
    detail = {
      state = ["FAILED"]
    }
    resources = [aws_codepipeline.main.arn]
  })
}

resource "aws_cloudwatch_event_target" "pipeline_alert" {
  rule      = aws_cloudwatch_event_rule.pipeline_failed.name
  target_id = "PipelineAlertSNS"
  arn       = aws_sns_topic.alerts.arn

  # ✅ IAM Role สำหรับ EventBridge -> SNS
  role_arn = aws_iam_role.eventbridge_sns.arn
}

# ✅ EventBridge IAM Role สำหรับ publish ไปยัง SNS
data "aws_iam_policy_document" "eventbridge_trust" {
  statement {
    effect = "Allow"
    principals {
      type        = "Service"
      identifiers = ["events.amazonaws.com"]
    }
    actions = ["sts:AssumeRole"]
  }
}

resource "aws_iam_role" "eventbridge_sns" {
  name               = "${var.project_name}-eventbridge-sns"
  assume_role_policy = data.aws_iam_policy_document.eventbridge_trust.json
}

resource "aws_iam_role_policy" "eventbridge_sns" {
  name = "sns-publish"
  role = aws_iam_role.eventbridge_sns.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect   = "Allow"
        Action   = ["sns:Publish"]
        Resource = [aws_sns_topic.alerts.arn]
      }
    ]
  })
}

# ✅ Custom Event Pattern - S3 events via EventBridge
resource "aws_s3_bucket_notification" "events" {
  bucket      = aws_s3_bucket.uploads.id
  eventbridge = true  # ✅ ส่ง S3 events ไปยัง EventBridge
}

resource "aws_cloudwatch_event_rule" "s3_upload" {
  name = "${var.project_name}-s3-upload"

  event_pattern = jsonencode({
    source      = ["aws.s3"]
    detail-type = ["Object Created"]
    detail = {
      bucket = {
        name = [aws_s3_bucket.uploads.bucket]
      }
      object = {
        key = [{ "prefix" = "uploads/" }]
      }
    }
  })
}
```

---

## Step 477: Fan-Out Pattern

```hcl
# ✅ Complete Fan-Out Pattern: SNS -> Multiple SQS Queues

# 1. Central SNS Topic
resource "aws_sns_topic" "order_events" {
  name              = "${var.project_name}-order-events"
  kms_master_key_id = aws_kms_key.sns.id

  tags = {
    Name    = "${var.project_name}-order-events"
    Pattern = "fan-out"
  }
}

# 2. SQS Queues สำหรับแต่ละ subscriber

# Email Service Queue
resource "aws_sqs_queue" "email_service" {
  name                      = "${var.project_name}-email-service"
  visibility_timeout_seconds = 60
  message_retention_seconds = 86400
  receive_wait_time_seconds  = 20
  sqs_managed_sse_enabled   = true

  redrive_policy = jsonencode({
    deadLetterTargetArn = aws_sqs_queue.email_service_dlq.arn
    maxReceiveCount     = 5
  })

  tags = { Name = "${var.project_name}-email-service" }
}

resource "aws_sqs_queue" "email_service_dlq" {
  name                      = "${var.project_name}-email-service-dlq"
  message_retention_seconds = 1209600
  sqs_managed_sse_enabled   = true
  tags                      = { Name = "${var.project_name}-email-service-dlq" }
}

# Inventory Service Queue
resource "aws_sqs_queue" "inventory_service" {
  name                      = "${var.project_name}-inventory-service"
  visibility_timeout_seconds = 120
  message_retention_seconds = 86400
  receive_wait_time_seconds  = 20
  sqs_managed_sse_enabled   = true

  redrive_policy = jsonencode({
    deadLetterTargetArn = aws_sqs_queue.inventory_service_dlq.arn
    maxReceiveCount     = 3
  })

  tags = { Name = "${var.project_name}-inventory-service" }
}

resource "aws_sqs_queue" "inventory_service_dlq" {
  name                      = "${var.project_name}-inventory-service-dlq"
  message_retention_seconds = 1209600
  sqs_managed_sse_enabled   = true
  tags                      = { Name = "${var.project_name}-inventory-dlq" }
}

# Analytics Queue
resource "aws_sqs_queue" "analytics_service" {
  name                      = "${var.project_name}-analytics-service"
  visibility_timeout_seconds = 300
  message_retention_seconds = 86400
  receive_wait_time_seconds  = 20
  sqs_managed_sse_enabled   = true

  redrive_policy = jsonencode({
    deadLetterTargetArn = aws_sqs_queue.analytics_service_dlq.arn
    maxReceiveCount     = 3
  })

  tags = { Name = "${var.project_name}-analytics-service" }
}

resource "aws_sqs_queue" "analytics_service_dlq" {
  name                      = "${var.project_name}-analytics-dlq"
  message_retention_seconds = 1209600
  sqs_managed_sse_enabled   = true
  tags                      = { Name = "${var.project_name}-analytics-dlq" }
}

# 3. SQS Queue Policies
locals {
  fanout_queues = {
    email = {
      queue_arn = aws_sqs_queue.email_service.arn
      queue_url = aws_sqs_queue.email_service.id
    }
    inventory = {
      queue_arn = aws_sqs_queue.inventory_service.arn
      queue_url = aws_sqs_queue.inventory_service.id
    }
    analytics = {
      queue_arn = aws_sqs_queue.analytics_service.arn
      queue_url = aws_sqs_queue.analytics_service.id
    }
  }
}

resource "aws_sqs_queue_policy" "fanout" {
  for_each = local.fanout_queues

  queue_url = each.value.queue_url

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Sid    = "AllowSNSPublish"
        Effect = "Allow"
        Principal = {
          Service = "sns.amazonaws.com"
        }
        Action   = "sqs:SendMessage"
        Resource = each.value.queue_arn
        Condition = {
          ArnEquals = {
            "aws:SourceArn" = aws_sns_topic.order_events.arn
          }
        }
      }
    ]
  })
}

# 4. SNS Subscriptions พร้อม Filter Policies

# Email Service - รับทุก ORDER events
resource "aws_sns_topic_subscription" "email_service" {
  topic_arn            = aws_sns_topic.order_events.arn
  protocol             = "sqs"
  endpoint             = aws_sqs_queue.email_service.arn
  raw_message_delivery = false  # ✅ Email service ต้องการ SNS envelope

  filter_policy = jsonencode({
    eventType = ["ORDER_CREATED", "ORDER_SHIPPED", "ORDER_DELIVERED", "ORDER_CANCELLED"]
  })
}

# Inventory Service - รับเฉพาะ ORDER_CREATED, ORDER_CANCELLED
resource "aws_sns_topic_subscription" "inventory_service" {
  topic_arn            = aws_sns_topic.order_events.arn
  protocol             = "sqs"
  endpoint             = aws_sqs_queue.inventory_service.arn
  raw_message_delivery = true

  filter_policy = jsonencode({
    eventType = ["ORDER_CREATED", "ORDER_CANCELLED"]
  })
}

# Analytics - รับทุก events
resource "aws_sns_topic_subscription" "analytics_service" {
  topic_arn            = aws_sns_topic.order_events.arn
  protocol             = "sqs"
  endpoint             = aws_sqs_queue.analytics_service.arn
  raw_message_delivery = true
  # ไม่มี filter - รับทุก events
}
```

---

## Step 478: EventBridge Archive และ Replay

```hcl
# ✅ EventBridge Archive
resource "aws_cloudwatch_event_archive" "main" {
  name             = "${var.project_name}-archive"
  event_source_arn = aws_cloudwatch_event_bus.main.arn
  description      = "Archive all events for replay"

  # ✅ Event Pattern - archive เฉพาะ order events
  event_pattern = jsonencode({
    source = ["app.orders"]
  })

  retention_days = 90  # เก็บ 90 วัน
}
```

---

## Step 479: Encryption สำหรับ Messaging

```hcl
# ✅ KMS Key สำหรับ SQS
resource "aws_kms_key" "sqs" {
  description             = "KMS key for SQS - ${var.project_name}"
  deletion_window_in_days = 14
  enable_key_rotation     = true

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Sid    = "Enable IAM"
        Effect = "Allow"
        Principal = {
          AWS = "arn:aws:iam::${data.aws_caller_identity.current.account_id}:root"
        }
        Action   = "kms:*"
        Resource = "*"
      },
      {
        Sid    = "Allow SNS to use key"
        Effect = "Allow"
        Principal = {
          Service = "sns.amazonaws.com"
        }
        Action = [
          "kms:GenerateDataKey",
          "kms:Decrypt",
        ]
        Resource = "*"
      },
    ]
  })

  tags = {
    Name      = "${var.project_name}-sqs-kms"
    ManagedBy = "terraform"
  }
}

# ✅ Encrypted SQS Queue
resource "aws_sqs_queue" "encrypted" {
  name = "${var.project_name}-encrypted"

  # ✅ ใช้ KMS สำหรับ enhanced security
  kms_master_key_id                 = aws_kms_key.sqs.arn
  kms_data_key_reuse_period_seconds = 300

  sqs_managed_sse_enabled = false  # ไม่ใช้ SSE-SQS เมื่อใช้ KMS

  tags = {
    Name      = "${var.project_name}-encrypted"
    ManagedBy = "terraform"
  }
}
```

---

## Step 480: Outputs และ Monitoring

```hcl
# ✅ CloudWatch Alarms สำหรับ SQS
resource "aws_cloudwatch_metric_alarm" "sqs_dlq_messages" {
  alarm_name          = "${var.project_name}-dlq-messages"
  comparison_operator = "GreaterThanOrEqualToThreshold"
  evaluation_periods  = 1
  metric_name         = "ApproximateNumberOfMessagesVisible"
  namespace           = "AWS/SQS"
  period              = 60
  statistic           = "Sum"
  threshold           = 1
  alarm_description   = "Messages in DLQ - requires attention"
  alarm_actions       = [aws_sns_topic.alerts.arn]
  ok_actions          = [aws_sns_topic.alerts.arn]

  dimensions = {
    QueueName = aws_sqs_queue.standard_dlq.name
  }

  tags = {
    Name      = "${var.project_name}-dlq-alarm"
    ManagedBy = "terraform"
  }
}

resource "aws_cloudwatch_metric_alarm" "sqs_age" {
  alarm_name          = "${var.project_name}-sqs-message-age"
  comparison_operator = "GreaterThanOrEqualToThreshold"
  evaluation_periods  = 2
  metric_name         = "ApproximateAgeOfOldestMessage"
  namespace           = "AWS/SQS"
  period              = 300
  statistic           = "Maximum"
  threshold           = 600  # Alert ถ้า message เก่ากว่า 10 minutes
  alarm_description   = "Messages are aging - consumer may be stuck"
  alarm_actions       = [aws_sns_topic.alerts.arn]

  dimensions = {
    QueueName = aws_sqs_queue.standard.name
  }
}

# ✅ Outputs
output "sqs_standard_url" {
  description = "Standard SQS Queue URL"
  value       = aws_sqs_queue.standard.url
}

output "sqs_standard_arn" {
  description = "Standard SQS Queue ARN"
  value       = aws_sqs_queue.standard.arn
}

output "sns_topic_arn" {
  description = "SNS Topic ARN"
  value       = aws_sns_topic.events.arn
}

output "event_bus_arn" {
  description = "EventBridge Event Bus ARN"
  value       = aws_cloudwatch_event_bus.main.arn
}
```

---

## Event-Driven Architecture Patterns สรุป

### ✅ SQS Best Practices

1. **Long Polling**: ตั้ง `receive_wait_time_seconds = 20`
2. **Visibility Timeout**: ตั้งให้เหมาะกับ processing time (6x Lambda timeout)
3. **DLQ**: ใส่ Dead Letter Queue สำหรับทุก queue
4. **Encryption**: ใช้ SSE-SQS หรือ KMS
5. **FIFO**: ใช้เมื่อต้องการ ordering และ deduplication
6. **Monitoring**: ตั้ง alarm สำหรับ DLQ messages

### ✅ SNS Best Practices

1. **Encryption**: ใช้ KMS
2. **HTTPS Policy**: Deny non-HTTPS subscriptions
3. **Filter Policies**: ใช้เพื่อลด unnecessary processing
4. **Fan-out**: SNS -> SQS pattern สำหรับ loose coupling
5. **FIFO Topics**: สำหรับ ordered notifications

### ✅ EventBridge Best Practices

1. **Custom Event Bus**: แยก business events จาก AWS events
2. **DLQ**: ตั้ง Dead Letter Queue สำหรับ failed events
3. **Archive**: เก็บ events สำหรับ replay
4. **Event Pattern**: ใช้ specific patterns ไม่ catch-all
5. **IAM Role**: ตั้ง role สำหรับ EventBridge targets

---

**Next Steps**: ไปต่อที่ Part 049 - AWS ECR & ECS Container Services
