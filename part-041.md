# Part 041: AWS IAM Roles, Users & Policies
## การจัดการสิทธิ์การเข้าถึง AWS ด้วย Terraform (Steps 401-410)

---

## บทนำ (Introduction)

AWS Identity and Access Management (IAM) เป็นบริการที่ช่วยควบคุมการเข้าถึง AWS resources อย่างปลอดภัย
Terraform ช่วยให้เราสามารถจัดการ IAM resources ทั้งหมดในรูปแบบ Infrastructure as Code ได้

**หัวข้อที่จะเรียนรู้:**
- IAM Users, Groups, และ Access Keys
- IAM Roles พร้อม Trust Relationships
- IAM Policies (Managed และ Inline)
- OIDC Provider สำหรับ CI/CD
- Common IAM Patterns สำหรับ AWS Services
- Security Best Practices และ Least Privilege Principle

---

## Step 401: IAM Users และ Access Keys

### aws_iam_user

```hcl
# ✅ Secure: สร้าง IAM User สำหรับ Application
resource "aws_iam_user" "app_user" {
  name = "app-service-user"
  path = "/service-accounts/"

  # Force destroy จะลบ access keys และ policies ก่อน destroy
  force_destroy = true

  tags = {
    Name        = "app-service-user"
    Environment = "production"
    ManagedBy   = "terraform"
    Purpose     = "application-service"
  }
}

# ❌ Insecure: อย่าสร้าง root account credentials หรือ users โดยไม่มี MFA policy
resource "aws_iam_user" "bad_user" {
  name = "admin-user"
  # ไม่มี tags, ไม่มี path organization - ยากต่อการ audit
}
```

### aws_iam_access_key

```hcl
# ✅ Secure: Access Key พร้อม rotation reminder
resource "aws_iam_access_key" "app_user_key" {
  user = aws_iam_user.app_user.name

  # ✅ เก็บ secret ใน SSM Parameter Store ไม่ใช่ใน state โดยตรง
  # pgp_key ใช้สำหรับ encrypt secret
  # pgp_key = "keybase:username"  # หรือ base64 encoded PGP key
}

# ✅ เก็บ credentials ใน AWS Secrets Manager
resource "aws_secretsmanager_secret" "app_user_credentials" {
  name                    = "app-service-user-credentials"
  description             = "IAM access key for app-service-user"
  recovery_window_in_days = 7

  tags = {
    Name        = "app-service-user-credentials"
    Environment = "production"
    ManagedBy   = "terraform"
  }
}

resource "aws_secretsmanager_secret_version" "app_user_credentials" {
  secret_id = aws_secretsmanager_secret.app_user_credentials.id
  secret_string = jsonencode({
    access_key_id     = aws_iam_access_key.app_user_key.id
    secret_access_key = aws_iam_access_key.app_user_key.secret
  })
}

# Output แบบ sensitive (ไม่แสดงใน plan/apply output)
output "app_user_access_key_id" {
  value     = aws_iam_access_key.app_user_key.id
  sensitive = true
}

output "app_user_secret_access_key" {
  value     = aws_iam_access_key.app_user_key.secret
  sensitive = true
}
```

---

## Step 402: IAM Groups และ Group Membership

### aws_iam_group

```hcl
# IAM Group สำหรับ Developers
resource "aws_iam_group" "developers" {
  name = "developers"
  path = "/teams/"
}

# IAM Group สำหรับ DevOps
resource "aws_iam_group" "devops" {
  name = "devops"
  path = "/teams/"
}

# IAM Group สำหรับ Read-Only Access
resource "aws_iam_group" "read_only" {
  name = "read-only"
  path = "/teams/"
}

# IAM Group สำหรับ Security Team
resource "aws_iam_group" "security" {
  name = "security-team"
  path = "/teams/"
}
```

### aws_iam_group_membership

```hcl
# ✅ Secure: กำหนด Group Membership อย่างชัดเจน
resource "aws_iam_group_membership" "developer_team" {
  name  = "developer-team-membership"
  group = aws_iam_group.developers.name

  users = [
    aws_iam_user.developer1.name,
    aws_iam_user.developer2.name,
    aws_iam_user.developer3.name,
  ]
}

resource "aws_iam_group_membership" "devops_team" {
  name  = "devops-team-membership"
  group = aws_iam_group.devops.name

  users = [
    aws_iam_user.devops_engineer.name,
  ]
}

# Users สำหรับ Team Members
resource "aws_iam_user" "developer1" {
  name          = "dev-john-doe"
  path          = "/developers/"
  force_destroy = true

  tags = {
    Name        = "John Doe"
    Team        = "developers"
    Environment = "all"
  }
}

resource "aws_iam_user" "developer2" {
  name          = "dev-jane-smith"
  path          = "/developers/"
  force_destroy = true

  tags = {
    Name = "Jane Smith"
    Team = "developers"
  }
}

resource "aws_iam_user" "developer3" {
  name          = "dev-bob-wilson"
  path          = "/developers/"
  force_destroy = true

  tags = {
    Name = "Bob Wilson"
    Team = "developers"
  }
}

resource "aws_iam_user" "devops_engineer" {
  name          = "ops-alice-brown"
  path          = "/devops/"
  force_destroy = true

  tags = {
    Name = "Alice Brown"
    Team = "devops"
  }
}
```

---

## Step 403: IAM Policy Document และ Managed Policies

### aws_iam_policy_document (Data Source)

```hcl
# ✅ Secure: ใช้ aws_iam_policy_document สำหรับสร้าง JSON policy อย่างปลอดภัย
data "aws_iam_policy_document" "s3_read_only" {
  statement {
    sid    = "AllowS3ReadAccess"
    effect = "Allow"

    actions = [
      "s3:GetObject",
      "s3:GetObjectVersion",
      "s3:ListBucket",
      "s3:ListBucketVersions",
    ]

    resources = [
      "arn:aws:s3:::my-app-bucket",
      "arn:aws:s3:::my-app-bucket/*",
    ]
  }

  # ✅ เพิ่ม condition เพื่อจำกัดการเข้าถึง
  statement {
    sid    = "DenyPublicAccess"
    effect = "Deny"

    actions = ["s3:*"]

    resources = ["*"]

    condition {
      test     = "Bool"
      variable = "aws:SecureTransport"
      values   = ["false"]
    }
  }
}

# ✅ Developer Policy - อ่านและเขียน S3, CloudWatch Logs
data "aws_iam_policy_document" "developer_policy" {
  # S3 access สำหรับ specific bucket
  statement {
    sid    = "S3DeveloperAccess"
    effect = "Allow"

    actions = [
      "s3:GetObject",
      "s3:PutObject",
      "s3:DeleteObject",
      "s3:ListBucket",
    ]

    resources = [
      "arn:aws:s3:::${var.project_name}-*",
      "arn:aws:s3:::${var.project_name}-*/*",
    ]
  }

  # CloudWatch Logs - อ่านได้เท่านั้น
  statement {
    sid    = "CloudWatchLogsRead"
    effect = "Allow"

    actions = [
      "logs:DescribeLogGroups",
      "logs:DescribeLogStreams",
      "logs:GetLogEvents",
      "logs:FilterLogEvents",
    ]

    resources = ["*"]
  }

  # EC2 Describe permissions (read-only)
  statement {
    sid    = "EC2ReadOnly"
    effect = "Allow"

    actions = [
      "ec2:Describe*",
      "ec2:Get*",
    ]

    resources = ["*"]
  }

  # SSM Parameter Store - อ่านได้เฉพาะ prefix ของ project
  statement {
    sid    = "SSMParameterRead"
    effect = "Allow"

    actions = [
      "ssm:GetParameter",
      "ssm:GetParameters",
      "ssm:GetParametersByPath",
    ]

    resources = [
      "arn:aws:ssm:${var.aws_region}:${data.aws_caller_identity.current.account_id}:parameter/${var.project_name}/*",
    ]
  }
}

# ✅ CI/CD Pipeline Policy
data "aws_iam_policy_document" "cicd_policy" {
  # ECR - push/pull images
  statement {
    sid    = "ECRAccess"
    effect = "Allow"

    actions = [
      "ecr:GetAuthorizationToken",
      "ecr:BatchCheckLayerAvailability",
      "ecr:GetDownloadUrlForLayer",
      "ecr:BatchGetImage",
      "ecr:PutImage",
      "ecr:InitiateLayerUpload",
      "ecr:UploadLayerPart",
      "ecr:CompleteLayerUpload",
    ]

    resources = ["*"]
  }

  # ECS - deploy services
  statement {
    sid    = "ECSDeployAccess"
    effect = "Allow"

    actions = [
      "ecs:RegisterTaskDefinition",
      "ecs:DeregisterTaskDefinition",
      "ecs:DescribeTaskDefinition",
      "ecs:UpdateService",
      "ecs:DescribeServices",
      "ecs:DescribeClusters",
    ]

    resources = ["*"]
  }

  # PassRole - เฉพาะ ECS task roles
  statement {
    sid    = "PassRoleForECS"
    effect = "Allow"

    actions = ["iam:PassRole"]

    resources = [
      "arn:aws:iam::${data.aws_caller_identity.current.account_id}:role/${var.project_name}-ecs-task-*",
    ]
  }

  # S3 - deploy artifacts
  statement {
    sid    = "S3DeployAccess"
    effect = "Allow"

    actions = [
      "s3:GetObject",
      "s3:PutObject",
      "s3:ListBucket",
    ]

    resources = [
      "arn:aws:s3:::${var.project_name}-artifacts",
      "arn:aws:s3:::${var.project_name}-artifacts/*",
    ]
  }
}
```

### aws_iam_policy (Managed Policy)

```hcl
# สร้าง Managed Policy จาก policy document
resource "aws_iam_policy" "developer_policy" {
  name        = "${var.project_name}-developer-policy"
  path        = "/policies/"
  description = "Developer access policy for ${var.project_name}"
  policy      = data.aws_iam_policy_document.developer_policy.json

  tags = {
    Name        = "${var.project_name}-developer-policy"
    Environment = var.environment
    ManagedBy   = "terraform"
  }
}

resource "aws_iam_policy" "cicd_policy" {
  name        = "${var.project_name}-cicd-policy"
  path        = "/policies/"
  description = "CI/CD pipeline policy for ${var.project_name}"
  policy      = data.aws_iam_policy_document.cicd_policy.json

  tags = {
    Name      = "${var.project_name}-cicd-policy"
    ManagedBy = "terraform"
  }
}

resource "aws_iam_policy" "s3_read_only_policy" {
  name        = "${var.project_name}-s3-read-only"
  path        = "/policies/"
  description = "Read-only S3 access policy"
  policy      = data.aws_iam_policy_document.s3_read_only.json

  tags = {
    Name      = "${var.project_name}-s3-read-only"
    ManagedBy = "terraform"
  }
}
```

---

## Step 404: IAM Roles พร้อม Trust Relationships

### Trust Policy สำหรับ EC2

```hcl
# ✅ Trust Policy สำหรับ EC2 instances
data "aws_iam_policy_document" "ec2_trust_policy" {
  statement {
    sid    = "AllowEC2AssumeRole"
    effect = "Allow"

    principals {
      type        = "Service"
      identifiers = ["ec2.amazonaws.com"]
    }

    actions = ["sts:AssumeRole"]
  }
}

resource "aws_iam_role" "ec2_instance_role" {
  name               = "${var.project_name}-ec2-role"
  path               = "/roles/"
  assume_role_policy = data.aws_iam_policy_document.ec2_trust_policy.json
  description        = "IAM role for EC2 instances in ${var.project_name}"

  # ✅ Max session duration (ปกติ 1 ชั่วโมง สำหรับ EC2)
  max_session_duration = 3600

  tags = {
    Name        = "${var.project_name}-ec2-role"
    Environment = var.environment
    ManagedBy   = "terraform"
  }
}
```

### Trust Policy สำหรับ Lambda

```hcl
# ✅ Trust Policy สำหรับ Lambda functions
data "aws_iam_policy_document" "lambda_trust_policy" {
  statement {
    sid    = "AllowLambdaAssumeRole"
    effect = "Allow"

    principals {
      type        = "Service"
      identifiers = ["lambda.amazonaws.com"]
    }

    actions = ["sts:AssumeRole"]
  }
}

resource "aws_iam_role" "lambda_execution_role" {
  name               = "${var.project_name}-lambda-role"
  path               = "/roles/"
  assume_role_policy = data.aws_iam_policy_document.lambda_trust_policy.json
  description        = "IAM role for Lambda functions"

  tags = {
    Name      = "${var.project_name}-lambda-role"
    ManagedBy = "terraform"
  }
}

# ✅ Lambda basic execution policy (CloudWatch Logs)
data "aws_iam_policy_document" "lambda_basic_execution" {
  statement {
    sid    = "CloudWatchLogs"
    effect = "Allow"

    actions = [
      "logs:CreateLogGroup",
      "logs:CreateLogStream",
      "logs:PutLogEvents",
    ]

    resources = [
      "arn:aws:logs:${var.aws_region}:${data.aws_caller_identity.current.account_id}:log-group:/aws/lambda/${var.project_name}-*",
    ]
  }
}

resource "aws_iam_role_policy" "lambda_basic_execution" {
  name   = "basic-execution"
  role   = aws_iam_role.lambda_execution_role.id
  policy = data.aws_iam_policy_document.lambda_basic_execution.json
}
```

### Trust Policy สำหรับ ECS

```hcl
# ✅ Trust Policy สำหรับ ECS Tasks
data "aws_iam_policy_document" "ecs_task_trust_policy" {
  statement {
    sid    = "AllowECSTaskAssumeRole"
    effect = "Allow"

    principals {
      type        = "Service"
      identifiers = ["ecs-tasks.amazonaws.com"]
    }

    actions = ["sts:AssumeRole"]
  }
}

# ECS Task Execution Role (สำหรับ ECS control plane)
resource "aws_iam_role" "ecs_task_execution_role" {
  name               = "${var.project_name}-ecs-task-execution-role"
  assume_role_policy = data.aws_iam_policy_document.ecs_task_trust_policy.json

  tags = {
    Name      = "${var.project_name}-ecs-task-execution-role"
    ManagedBy = "terraform"
  }
}

# ✅ Attach AWS managed policy สำหรับ ECS Task Execution
resource "aws_iam_role_policy_attachment" "ecs_task_execution_policy" {
  role       = aws_iam_role.ecs_task_execution_role.name
  policy_arn = "arn:aws:iam::aws:policy/service-role/AmazonECSTaskExecutionRolePolicy"
}

# ECS Task Role (สำหรับ application code)
resource "aws_iam_role" "ecs_task_role" {
  name               = "${var.project_name}-ecs-task-role"
  assume_role_policy = data.aws_iam_policy_document.ecs_task_trust_policy.json

  tags = {
    Name      = "${var.project_name}-ecs-task-role"
    ManagedBy = "terraform"
  }
}
```

### Trust Policy สำหรับ EKS

```hcl
# ✅ EKS Cluster Role
data "aws_iam_policy_document" "eks_cluster_trust_policy" {
  statement {
    effect = "Allow"

    principals {
      type        = "Service"
      identifiers = ["eks.amazonaws.com"]
    }

    actions = ["sts:AssumeRole"]
  }
}

resource "aws_iam_role" "eks_cluster_role" {
  name               = "${var.project_name}-eks-cluster-role"
  assume_role_policy = data.aws_iam_policy_document.eks_cluster_trust_policy.json

  tags = {
    Name      = "${var.project_name}-eks-cluster-role"
    ManagedBy = "terraform"
  }
}

resource "aws_iam_role_policy_attachment" "eks_cluster_policy" {
  policy_arn = "arn:aws:iam::aws:policy/AmazonEKSClusterPolicy"
  role       = aws_iam_role.eks_cluster_role.name
}

# ✅ EKS Node Group Role
data "aws_iam_policy_document" "eks_node_trust_policy" {
  statement {
    effect = "Allow"

    principals {
      type        = "Service"
      identifiers = ["ec2.amazonaws.com"]
    }

    actions = ["sts:AssumeRole"]
  }
}

resource "aws_iam_role" "eks_node_role" {
  name               = "${var.project_name}-eks-node-role"
  assume_role_policy = data.aws_iam_policy_document.eks_node_trust_policy.json

  tags = {
    Name      = "${var.project_name}-eks-node-role"
    ManagedBy = "terraform"
  }
}

# ✅ 3 Required policies สำหรับ EKS Node Groups
resource "aws_iam_role_policy_attachment" "eks_worker_node_policy" {
  policy_arn = "arn:aws:iam::aws:policy/AmazonEKSWorkerNodePolicy"
  role       = aws_iam_role.eks_node_role.name
}

resource "aws_iam_role_policy_attachment" "eks_cni_policy" {
  policy_arn = "arn:aws:iam::aws:policy/AmazonEKS_CNI_Policy"
  role       = aws_iam_role.eks_node_role.name
}

resource "aws_iam_role_policy_attachment" "ecr_read_only" {
  policy_arn = "arn:aws:iam::aws:policy/AmazonEC2ContainerRegistryReadOnly"
  role       = aws_iam_role.eks_node_role.name
}
```

---

## Step 405: IAM Role Policy Attachments

### aws_iam_role_policy_attachment

```hcl
# ✅ Attach Managed Policy ไปยัง Role
resource "aws_iam_role_policy_attachment" "ec2_ssm_policy" {
  role       = aws_iam_role.ec2_instance_role.name
  policy_arn = "arn:aws:iam::aws:policy/AmazonSSMManagedInstanceCore"
}

resource "aws_iam_role_policy_attachment" "ec2_cloudwatch_policy" {
  role       = aws_iam_role.ec2_instance_role.name
  policy_arn = "arn:aws:iam::aws:policy/CloudWatchAgentServerPolicy"
}

resource "aws_iam_role_policy_attachment" "ec2_custom_policy" {
  role       = aws_iam_role.ec2_instance_role.name
  policy_arn = aws_iam_policy.developer_policy.arn
}

# ✅ Lambda - attach multiple policies
resource "aws_iam_role_policy_attachment" "lambda_vpc_policy" {
  role       = aws_iam_role.lambda_execution_role.name
  policy_arn = "arn:aws:iam::aws:policy/service-role/AWSLambdaVPCAccessExecutionRole"
}

resource "aws_iam_role_policy_attachment" "lambda_xray_policy" {
  role       = aws_iam_role.lambda_execution_role.name
  policy_arn = "arn:aws:iam::aws:policy/AWSXRayDaemonWriteAccess"
}
```

### aws_iam_user_policy_attachment

```hcl
# ✅ Attach Policy ไปยัง IAM Group (ดีกว่า attach กับ User โดยตรง)
resource "aws_iam_group_policy_attachment" "developer_group_policy" {
  group      = aws_iam_group.developers.name
  policy_arn = aws_iam_policy.developer_policy.arn
}

resource "aws_iam_group_policy_attachment" "developer_s3_policy" {
  group      = aws_iam_group.developers.name
  policy_arn = aws_iam_policy.s3_read_only_policy.arn
}

# ✅ Read-only access สำหรับ group
resource "aws_iam_group_policy_attachment" "read_only_group_policy" {
  group      = aws_iam_group.read_only.name
  policy_arn = "arn:aws:iam::aws:policy/ReadOnlyAccess"
}

# ⚠️ ในบางกรณีที่ต้อง attach policy กับ user โดยตรง
resource "aws_iam_user_policy_attachment" "app_user_policy" {
  user       = aws_iam_user.app_user.name
  policy_arn = aws_iam_policy.cicd_policy.arn
}
```

### aws_iam_role_policy (Inline Policy)

```hcl
# ✅ Inline Policy - ใช้เมื่อ policy ต้องการ dynamic values หรือเป็น one-off
resource "aws_iam_role_policy" "ecs_secrets_access" {
  name = "secrets-access"
  role = aws_iam_role.ecs_task_role.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Sid    = "SecretsManagerAccess"
        Effect = "Allow"
        Action = [
          "secretsmanager:GetSecretValue",
          "secretsmanager:DescribeSecret",
        ]
        Resource = [
          "arn:aws:secretsmanager:${var.aws_region}:${data.aws_caller_identity.current.account_id}:secret:${var.project_name}/*",
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
          aws_kms_key.app.arn,
        ]
      },
    ]
  })
}

# ✅ Inline policy สำหรับ Lambda ที่ต้องการ access DynamoDB
resource "aws_iam_role_policy" "lambda_dynamodb_access" {
  name = "dynamodb-access"
  role = aws_iam_role.lambda_execution_role.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Sid    = "DynamoDBAccess"
        Effect = "Allow"
        Action = [
          "dynamodb:GetItem",
          "dynamodb:PutItem",
          "dynamodb:UpdateItem",
          "dynamodb:DeleteItem",
          "dynamodb:Query",
          "dynamodb:Scan",
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

## Step 406: IAM Instance Profile

```hcl
# ✅ Instance Profile สำหรับ EC2
resource "aws_iam_instance_profile" "ec2_profile" {
  name = "${var.project_name}-ec2-profile"
  role = aws_iam_role.ec2_instance_role.name

  tags = {
    Name      = "${var.project_name}-ec2-profile"
    ManagedBy = "terraform"
  }
}

# ✅ Application Server Role พร้อม policies ที่จำเป็น
resource "aws_iam_role" "app_server_role" {
  name               = "${var.project_name}-app-server"
  assume_role_policy = data.aws_iam_policy_document.ec2_trust_policy.json

  tags = {
    Name      = "${var.project_name}-app-server"
    ManagedBy = "terraform"
  }
}

# ✅ SSM สำหรับ patching และ session manager
resource "aws_iam_role_policy_attachment" "app_server_ssm" {
  role       = aws_iam_role.app_server_role.name
  policy_arn = "arn:aws:iam::aws:policy/AmazonSSMManagedInstanceCore"
}

# ✅ CloudWatch Agent
resource "aws_iam_role_policy_attachment" "app_server_cw" {
  role       = aws_iam_role.app_server_role.name
  policy_arn = "arn:aws:iam::aws:policy/CloudWatchAgentServerPolicy"
}

# ✅ Application-specific policy
data "aws_iam_policy_document" "app_server_policy" {
  statement {
    sid    = "S3AppAccess"
    effect = "Allow"

    actions = [
      "s3:GetObject",
      "s3:PutObject",
      "s3:ListBucket",
    ]

    resources = [
      "arn:aws:s3:::${var.project_name}-app-data",
      "arn:aws:s3:::${var.project_name}-app-data/*",
    ]
  }

  statement {
    sid    = "SecretsManagerAccess"
    effect = "Allow"

    actions = [
      "secretsmanager:GetSecretValue",
    ]

    resources = [
      "arn:aws:secretsmanager:${var.aws_region}:${data.aws_caller_identity.current.account_id}:secret:${var.project_name}/*",
    ]
  }
}

resource "aws_iam_role_policy" "app_server_inline" {
  name   = "app-server-policy"
  role   = aws_iam_role.app_server_role.id
  policy = data.aws_iam_policy_document.app_server_policy.json
}

resource "aws_iam_instance_profile" "app_server_profile" {
  name = "${var.project_name}-app-server-profile"
  role = aws_iam_role.app_server_role.name
}

# ✅ ใช้ instance profile กับ EC2
resource "aws_instance" "app_server" {
  ami                  = data.aws_ami.amazon_linux_2.id
  instance_type        = "t3.medium"
  iam_instance_profile = aws_iam_instance_profile.app_server_profile.name
  # ... other config
}
```

---

## Step 407: OIDC Provider สำหรับ GitHub Actions

### aws_iam_openid_connect_provider

```hcl
# ✅ GitHub Actions OIDC Provider
resource "aws_iam_openid_connect_provider" "github_actions" {
  url = "https://token.actions.githubusercontent.com"

  client_id_list = [
    "sts.amazonaws.com",
  ]

  # ✅ Thumbprint ของ GitHub Actions OIDC
  thumbprint_list = [
    "6938fd4d98bab03faadb97b34396831e3780aea1",
    "1c58a3a8518e8759bf075b76b750d4f2df264fcd",
  ]

  tags = {
    Name      = "github-actions-oidc"
    ManagedBy = "terraform"
  }
}

# ✅ Trust Policy สำหรับ GitHub Actions Role
data "aws_iam_policy_document" "github_actions_trust_policy" {
  statement {
    sid    = "AllowGitHubActionsAssumeRole"
    effect = "Allow"

    principals {
      type        = "Federated"
      identifiers = [aws_iam_openid_connect_provider.github_actions.arn]
    }

    actions = ["sts:AssumeRoleWithWebIdentity"]

    # ✅ จำกัดเฉพาะ repository และ branch ที่ต้องการ
    condition {
      test     = "StringEquals"
      variable = "token.actions.githubusercontent.com:aud"
      values   = ["sts.amazonaws.com"]
    }

    condition {
      test     = "StringLike"
      variable = "token.actions.githubusercontent.com:sub"
      # ✅ เฉพาะ main branch ของ repository นี้เท่านั้น
      values = ["repo:${var.github_org}/${var.github_repo}:ref:refs/heads/main"]
    }
  }
}

resource "aws_iam_role" "github_actions_role" {
  name               = "${var.project_name}-github-actions"
  assume_role_policy = data.aws_iam_policy_document.github_actions_trust_policy.json
  description        = "IAM role for GitHub Actions CI/CD"

  max_session_duration = 3600

  tags = {
    Name      = "${var.project_name}-github-actions"
    ManagedBy = "terraform"
  }
}

# ✅ Policy สำหรับ GitHub Actions Deploy
data "aws_iam_policy_document" "github_actions_deploy" {
  # ECR
  statement {
    sid    = "ECRAuth"
    effect = "Allow"
    actions = ["ecr:GetAuthorizationToken"]
    resources = ["*"]
  }

  statement {
    sid    = "ECRPushPull"
    effect = "Allow"
    actions = [
      "ecr:BatchCheckLayerAvailability",
      "ecr:GetDownloadUrlForLayer",
      "ecr:BatchGetImage",
      "ecr:PutImage",
      "ecr:InitiateLayerUpload",
      "ecr:UploadLayerPart",
      "ecr:CompleteLayerUpload",
      "ecr:DescribeRepositories",
    ]
    resources = [
      "arn:aws:ecr:${var.aws_region}:${data.aws_caller_identity.current.account_id}:repository/${var.project_name}*",
    ]
  }

  # ECS Deploy
  statement {
    sid    = "ECSUpdate"
    effect = "Allow"
    actions = [
      "ecs:RegisterTaskDefinition",
      "ecs:DescribeTaskDefinition",
      "ecs:UpdateService",
      "ecs:DescribeServices",
    ]
    resources = ["*"]
  }

  statement {
    sid    = "IAMPassRole"
    effect = "Allow"
    actions = ["iam:PassRole"]
    resources = [
      aws_iam_role.ecs_task_execution_role.arn,
      aws_iam_role.ecs_task_role.arn,
    ]
  }
}

resource "aws_iam_role_policy" "github_actions_deploy" {
  name   = "deploy-policy"
  role   = aws_iam_role.github_actions_role.id
  policy = data.aws_iam_policy_document.github_actions_deploy.json
}
```

### GitLab CI OIDC Provider

```hcl
# ✅ GitLab CI OIDC Provider
resource "aws_iam_openid_connect_provider" "gitlab_ci" {
  url = "https://gitlab.com"

  client_id_list = [
    "https://gitlab.com",
  ]

  thumbprint_list = [
    "b3dd7606d2b5a8b4a13771dbecc9ee1cecafa38a",
  ]

  tags = {
    Name      = "gitlab-ci-oidc"
    ManagedBy = "terraform"
  }
}

# ✅ Trust Policy สำหรับ GitLab CI
data "aws_iam_policy_document" "gitlab_ci_trust_policy" {
  statement {
    effect = "Allow"

    principals {
      type        = "Federated"
      identifiers = [aws_iam_openid_connect_provider.gitlab_ci.arn]
    }

    actions = ["sts:AssumeRoleWithWebIdentity"]

    condition {
      test     = "StringEquals"
      variable = "gitlab.com:aud"
      values   = ["https://gitlab.com"]
    }

    condition {
      test     = "StringLike"
      variable = "gitlab.com:sub"
      values   = ["project_path:${var.gitlab_namespace}/${var.gitlab_project}:ref_type:branch:ref:main"]
    }
  }
}
```

---

## Step 408: Cross-Account Access

```hcl
# ✅ Cross-Account Role (ใน Account A - Target Account)
data "aws_iam_policy_document" "cross_account_trust_policy" {
  statement {
    sid    = "AllowCrossAccountAccess"
    effect = "Allow"

    principals {
      type = "AWS"
      # Source account ที่จะ assume role นี้
      identifiers = ["arn:aws:iam::${var.source_account_id}:root"]
    }

    actions = ["sts:AssumeRole"]

    # ✅ ต้องการ MFA สำหรับ cross-account access
    condition {
      test     = "Bool"
      variable = "aws:MultiFactorAuthPresent"
      values   = ["true"]
    }

    # ✅ จำกัดเฉพาะ specific role ใน source account
    condition {
      test     = "ArnLike"
      variable = "aws:PrincipalArn"
      values   = ["arn:aws:iam::${var.source_account_id}:role/cross-account-*"]
    }
  }
}

resource "aws_iam_role" "cross_account_role" {
  name               = "cross-account-deploy-role"
  assume_role_policy = data.aws_iam_policy_document.cross_account_trust_policy.json
  description        = "Role สำหรับ cross-account deployment"

  # ✅ External ID เพิ่มความปลอดภัย (Confused Deputy Protection)
  # ใช้เมื่อ allow third-party account
  tags = {
    Name      = "cross-account-deploy-role"
    ManagedBy = "terraform"
  }
}

# ✅ Cross-Account Role (ใน Account B - Source Account)
data "aws_iam_policy_document" "cross_account_assume_policy" {
  statement {
    sid    = "AllowAssumeTargetRole"
    effect = "Allow"

    actions = ["sts:AssumeRole"]

    resources = [
      "arn:aws:iam::${var.target_account_id}:role/cross-account-deploy-role",
    ]
  }
}

resource "aws_iam_policy" "cross_account_policy" {
  name   = "cross-account-assume-policy"
  policy = data.aws_iam_policy_document.cross_account_assume_policy.json
}
```

---

## Step 409: IAM Boundary Policies (Permission Boundaries)

```hcl
# ✅ Permission Boundary - กำหนด maximum permissions ที่ role/user จะมีได้
data "aws_iam_policy_document" "developer_boundary" {
  # Developer ห้ามทำ IAM operations ที่ sensitive
  statement {
    sid    = "AllowedServices"
    effect = "Allow"

    actions = [
      "s3:*",
      "ec2:Describe*",
      "cloudwatch:*",
      "logs:*",
      "lambda:*",
      "dynamodb:*",
      "sqs:*",
      "sns:*",
      "ssm:GetParameter*",
    ]

    resources = ["*"]
  }

  # ❌ ห้ามใช้ IAM ยกเว้น read-only
  statement {
    sid    = "DenyIAMMutations"
    effect = "Deny"

    actions = [
      "iam:CreateUser",
      "iam:DeleteUser",
      "iam:CreateRole",
      "iam:DeleteRole",
      "iam:AttachRolePolicy",
      "iam:DetachRolePolicy",
      "iam:CreatePolicy",
      "iam:DeletePolicy",
      "iam:PutUserPolicy",
      "iam:PutRolePolicy",
    ]

    resources = ["*"]
  }

  # ❌ ห้าม disable security controls
  statement {
    sid    = "DenySecurityControls"
    effect = "Deny"

    actions = [
      "cloudtrail:StopLogging",
      "cloudtrail:DeleteTrail",
      "config:StopConfigurationRecorder",
      "config:DeleteConfigurationRecorder",
      "guardduty:DisassociateFromMasterAccount",
      "guardduty:DeleteDetector",
    ]

    resources = ["*"]
  }
}

resource "aws_iam_policy" "developer_boundary_policy" {
  name        = "${var.project_name}-developer-boundary"
  path        = "/boundaries/"
  description = "Permission boundary for developer roles"
  policy      = data.aws_iam_policy_document.developer_boundary.json
}

# ✅ Apply boundary ไปยัง role
resource "aws_iam_role" "developer_role_with_boundary" {
  name               = "${var.project_name}-developer-role"
  assume_role_policy = data.aws_iam_policy_document.ec2_trust_policy.json

  # ✅ Permission Boundary กำหนด maximum permissions
  permissions_boundary = aws_iam_policy.developer_boundary_policy.arn

  tags = {
    Name      = "${var.project_name}-developer-role"
    ManagedBy = "terraform"
  }
}
```

---

## Step 410: EKS IRSA (IAM Roles for Service Accounts)

```hcl
# ✅ EKS OIDC Provider (สร้างจาก cluster thumbprint)
data "tls_certificate" "eks" {
  url = aws_eks_cluster.main.identity[0].oidc[0].issuer
}

resource "aws_iam_openid_connect_provider" "eks" {
  client_id_list  = ["sts.amazonaws.com"]
  thumbprint_list = [data.tls_certificate.eks.certificates[0].sha1_fingerprint]
  url             = aws_eks_cluster.main.identity[0].oidc[0].issuer

  tags = {
    Name      = "${var.project_name}-eks-oidc"
    ManagedBy = "terraform"
  }
}

locals {
  oidc_issuer     = trimprefix(aws_eks_cluster.main.identity[0].oidc[0].issuer, "https://")
  oidc_issuer_arn = aws_iam_openid_connect_provider.eks.arn
}

# ✅ Trust Policy สำหรับ Kubernetes Service Account
data "aws_iam_policy_document" "eks_pod_trust_policy" {
  statement {
    effect = "Allow"

    principals {
      type        = "Federated"
      identifiers = [local.oidc_issuer_arn]
    }

    actions = ["sts:AssumeRoleWithWebIdentity"]

    # ✅ จำกัดเฉพาะ specific service account ใน namespace นั้น
    condition {
      test     = "StringEquals"
      variable = "${local.oidc_issuer}:sub"
      values   = ["system:serviceaccount:${var.k8s_namespace}:${var.k8s_service_account_name}"]
    }

    condition {
      test     = "StringEquals"
      variable = "${local.oidc_issuer}:aud"
      values   = ["sts.amazonaws.com"]
    }
  }
}

# ✅ IAM Role สำหรับ Kubernetes Pod (External DNS)
resource "aws_iam_role" "external_dns" {
  name               = "${var.project_name}-external-dns"
  assume_role_policy = data.aws_iam_policy_document.eks_pod_trust_policy.json

  tags = {
    Name      = "${var.project_name}-external-dns"
    ManagedBy = "terraform"
  }
}

data "aws_iam_policy_document" "external_dns_policy" {
  statement {
    effect = "Allow"

    actions = [
      "route53:ChangeResourceRecordSets",
    ]

    resources = ["arn:aws:route53:::hostedzone/*"]
  }

  statement {
    effect = "Allow"

    actions = [
      "route53:ListHostedZones",
      "route53:ListResourceRecordSets",
    ]

    resources = ["*"]
  }
}

resource "aws_iam_role_policy" "external_dns" {
  name   = "external-dns-policy"
  role   = aws_iam_role.external_dns.id
  policy = data.aws_iam_policy_document.external_dns_policy.json
}

# ✅ IAM Role สำหรับ AWS Load Balancer Controller
data "aws_iam_policy_document" "alb_controller_trust" {
  statement {
    effect = "Allow"

    principals {
      type        = "Federated"
      identifiers = [local.oidc_issuer_arn]
    }

    actions = ["sts:AssumeRoleWithWebIdentity"]

    condition {
      test     = "StringEquals"
      variable = "${local.oidc_issuer}:sub"
      values   = ["system:serviceaccount:kube-system:aws-load-balancer-controller"]
    }

    condition {
      test     = "StringEquals"
      variable = "${local.oidc_issuer}:aud"
      values   = ["sts.amazonaws.com"]
    }
  }
}

resource "aws_iam_role" "alb_controller" {
  name               = "${var.project_name}-alb-controller"
  assume_role_policy = data.aws_iam_policy_document.alb_controller_trust.json

  tags = {
    Name      = "${var.project_name}-alb-controller"
    ManagedBy = "terraform"
  }
}

# AWS managed policy สำหรับ ALB Controller
resource "aws_iam_role_policy_attachment" "alb_controller" {
  role       = aws_iam_role.alb_controller.name
  policy_arn = aws_iam_policy.alb_controller_policy.arn
}
```

---

## ตัวอย่าง Complete IAM Configuration

### variables.tf

```hcl
variable "aws_region" {
  description = "AWS region"
  type        = string
  default     = "ap-southeast-1"
}

variable "project_name" {
  description = "Project name prefix"
  type        = string
}

variable "environment" {
  description = "Environment name"
  type        = string
}

variable "github_org" {
  description = "GitHub organization name"
  type        = string
  default     = ""
}

variable "github_repo" {
  description = "GitHub repository name"
  type        = string
  default     = ""
}

variable "source_account_id" {
  description = "Source AWS account ID for cross-account access"
  type        = string
  default     = ""
}

variable "target_account_id" {
  description = "Target AWS account ID for cross-account access"
  type        = string
  default     = ""
}

variable "k8s_namespace" {
  description = "Kubernetes namespace for service account"
  type        = string
  default     = "default"
}

variable "k8s_service_account_name" {
  description = "Kubernetes service account name"
  type        = string
  default     = "default"
}
```

### data.tf

```hcl
data "aws_caller_identity" "current" {}
data "aws_partition" "current" {}

data "aws_ami" "amazon_linux_2" {
  most_recent = true
  owners      = ["amazon"]

  filter {
    name   = "name"
    values = ["amzn2-ami-hvm-*-x86_64-gp2"]
  }
}
```

---

## IAM Security Best Practices สรุป

### ✅ สิ่งที่ควรทำ (Do)

1. **Least Privilege Principle**: ให้สิทธิ์น้อยที่สุดที่จำเป็น
2. **ใช้ IAM Roles แทน Access Keys**: โดยเฉพาะสำหรับ EC2, Lambda, ECS
3. **Rotate Access Keys**: หมุนเวียน access keys อย่างสม่ำเสมอ
4. **เปิดใช้ MFA**: สำหรับ IAM users ที่ต้องการ console access
5. **ใช้ Permission Boundaries**: สำหรับ developer roles
6. **Monitor ด้วย CloudTrail**: ติดตาม IAM activities
7. **ใช้ OIDC สำหรับ CI/CD**: แทน long-lived access keys
8. **Resource-based Restrictions**: จำกัด resources ให้เฉพาะที่จำเป็น

### ❌ สิ่งที่ไม่ควรทำ (Don't)

1. **ห้ามใช้ root account**: สำหรับ day-to-day operations
2. **ห้ามแนบ AdministratorAccess**: กับ application roles
3. **ห้าม hardcode credentials**: ใน code หรือ Terraform config
4. **ห้ามใช้ wildcard resources**: ใน sensitive actions
5. **ห้ามสร้าง access keys**: สำหรับ EC2/Lambda (ใช้ roles แทน)

### ✅ Least Privilege Pattern ตัวอย่าง

```hcl
# ❌ Too Permissive - อย่าทำแบบนี้!
data "aws_iam_policy_document" "bad_policy" {
  statement {
    effect    = "Allow"
    actions   = ["*"]
    resources = ["*"]
  }
}

# ✅ Least Privilege - ทำแบบนี้!
data "aws_iam_policy_document" "good_policy" {
  statement {
    sid    = "AllowOnlyNeededActions"
    effect = "Allow"

    actions = [
      "s3:GetObject",
      "s3:PutObject",
    ]

    resources = [
      "arn:aws:s3:::my-specific-bucket/*",
    ]

    condition {
      test     = "StringEquals"
      variable = "aws:RequestedRegion"
      values   = ["ap-southeast-1"]
    }
  }
}
```

---

## IAM Access Analyzer

```hcl
# ✅ Enable IAM Access Analyzer สำหรับ detect unused access
resource "aws_accessanalyzer_analyzer" "account" {
  analyzer_name = "${var.project_name}-access-analyzer"
  type          = "ACCOUNT"

  tags = {
    Name      = "${var.project_name}-access-analyzer"
    ManagedBy = "terraform"
  }
}

# Organization-level analyzer (ต้องการ AWS Organizations)
resource "aws_accessanalyzer_analyzer" "organization" {
  count = var.is_organization_account ? 1 : 0

  analyzer_name = "${var.project_name}-org-analyzer"
  type          = "ORGANIZATION"

  tags = {
    Name      = "${var.project_name}-org-analyzer"
    ManagedBy = "terraform"
  }
}
```

---

## Service Control Policies (SCPs) Overview

SCPs ใช้ใน AWS Organizations เพื่อควบคุม maximum permissions ของทุก account ใน organization

```hcl
# ✅ SCP - บังคับใช้ encryption ทุกที่
# (ต้องการ AWS Organizations permissions)
data "aws_iam_policy_document" "require_encryption_scp" {
  # ห้ามสร้าง S3 bucket ที่ไม่มี encryption
  statement {
    sid    = "DenyS3WithoutEncryption"
    effect = "Deny"

    actions = ["s3:CreateBucket"]

    resources = ["*"]

    condition {
      test     = "StringNotEquals"
      variable = "s3:x-amz-server-side-encryption"
      values   = ["aws:kms", "AES256"]
    }
  }

  # ห้าม disable CloudTrail
  statement {
    sid    = "ProtectCloudTrail"
    effect = "Deny"

    actions = [
      "cloudtrail:DeleteTrail",
      "cloudtrail:StopLogging",
      "cloudtrail:UpdateTrail",
    ]

    resources = ["*"]
  }

  # ห้ามทำงานนอก approved regions
  statement {
    sid    = "DenyNonApprovedRegions"
    effect = "Deny"

    not_actions = [
      "iam:*",
      "sts:*",
      "cloudfront:*",
      "route53:*",
      "waf:*",
      "support:*",
    ]

    resources = ["*"]

    condition {
      test     = "StringNotEquals"
      variable = "aws:RequestedRegion"
      values   = ["ap-southeast-1", "ap-southeast-2", "us-east-1"]
    }
  }
}
```

---

## สรุปและแนวทางปฏิบัติ

### Pattern Summary

| Pattern | Use Case | Trust Principal |
|---------|----------|-----------------|
| EC2 Instance Profile | App servers, worker nodes | `ec2.amazonaws.com` |
| Lambda Execution Role | Serverless functions | `lambda.amazonaws.com` |
| ECS Task Role | Container applications | `ecs-tasks.amazonaws.com` |
| GitHub Actions OIDC | CI/CD pipelines | OIDC Federated |
| Cross-Account Role | Multi-account deployments | AWS Account ARN |
| EKS IRSA | Kubernetes pods | OIDC Federated (EKS) |

### IAM Hierarchy

```
AWS Organization
├── Management Account (SCPs)
├── Production Account
│   ├── IAM Roles (EC2, Lambda, ECS, EKS)
│   ├── IAM Users (Service accounts)
│   └── IAM Groups (Teams)
└── Development Account
    ├── IAM Roles
    └── IAM Users (Developers)
```

---

**Next Steps**: ไปต่อที่ Part 042 - AWS RDS Databases เพื่อเรียนรู้การ provision database ด้วย Terraform
