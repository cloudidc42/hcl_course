# Part 043: AWS EKS Kubernetes Clusters
## การสร้างและจัดการ Kubernetes Cluster ด้วย Terraform (Steps 421-430)

---

## บทนำ (Introduction)

Amazon EKS (Elastic Kubernetes Service) เป็น managed Kubernetes service ที่ช่วยให้เราสามารถ
รัน Kubernetes workloads บน AWS ได้โดยไม่ต้องจัดการ control plane เอง

**หัวข้อที่จะเรียนรู้:**
- EKS Cluster และ IAM Roles
- Node Groups (Managed Nodes)
- Fargate Profiles
- EKS Add-ons
- IRSA (IAM Roles for Service Accounts)
- Security Groups สำหรับ EKS
- Kubernetes Provider Configuration
- Helm Provider Integration
- Production EKS Setup

---

## Step 421: EKS Cluster พื้นฐาน

### VPC Configuration สำหรับ EKS

```hcl
# ✅ VPC พร้อม tags ที่จำเป็นสำหรับ EKS
resource "aws_vpc" "eks_vpc" {
  cidr_block           = "10.0.0.0/16"
  enable_dns_hostnames = true  # ✅ จำเป็นสำหรับ EKS
  enable_dns_support   = true  # ✅ จำเป็นสำหรับ EKS

  tags = {
    Name                                            = "${var.cluster_name}-vpc"
    "kubernetes.io/cluster/${var.cluster_name}"     = "shared"  # ✅ Required tag
    ManagedBy                                       = "terraform"
  }
}

# ✅ Public Subnets (สำหรับ Load Balancers)
resource "aws_subnet" "public" {
  count             = length(var.availability_zones)
  vpc_id            = aws_vpc.eks_vpc.id
  cidr_block        = "10.0.${count.index}.0/24"
  availability_zone = var.availability_zones[count.index]

  map_public_ip_on_launch = true  # ✅ จำเป็นสำหรับ public ALB

  tags = {
    Name                                            = "${var.cluster_name}-public-${var.availability_zones[count.index]}"
    "kubernetes.io/cluster/${var.cluster_name}"     = "shared"  # ✅ EKS tag
    "kubernetes.io/role/elb"                        = "1"       # ✅ External ALB tag
    ManagedBy                                       = "terraform"
  }
}

# ✅ Private Subnets (สำหรับ Worker Nodes)
resource "aws_subnet" "private" {
  count             = length(var.availability_zones)
  vpc_id            = aws_vpc.eks_vpc.id
  cidr_block        = "10.0.${count.index + 10}.0/24"
  availability_zone = var.availability_zones[count.index]

  tags = {
    Name                                            = "${var.cluster_name}-private-${var.availability_zones[count.index]}"
    "kubernetes.io/cluster/${var.cluster_name}"     = "owned"  # ✅ Private subnet = "owned"
    "kubernetes.io/role/internal-elb"               = "1"      # ✅ Internal ALB tag
    ManagedBy                                       = "terraform"
  }
}
```

### aws_eks_cluster

```hcl
# ✅ EKS Cluster IAM Role
data "aws_iam_policy_document" "eks_cluster_trust" {
  statement {
    effect = "Allow"
    principals {
      type        = "Service"
      identifiers = ["eks.amazonaws.com"]
    }
    actions = ["sts:AssumeRole"]
  }
}

resource "aws_iam_role" "eks_cluster" {
  name               = "${var.cluster_name}-cluster-role"
  assume_role_policy = data.aws_iam_policy_document.eks_cluster_trust.json

  tags = {
    Name      = "${var.cluster_name}-cluster-role"
    ManagedBy = "terraform"
  }
}

resource "aws_iam_role_policy_attachment" "eks_cluster_policy" {
  policy_arn = "arn:aws:iam::aws:policy/AmazonEKSClusterPolicy"
  role       = aws_iam_role.eks_cluster.name
}

# ✅ EKS Cluster
resource "aws_eks_cluster" "main" {
  name     = var.cluster_name
  version  = var.kubernetes_version  # e.g., "1.28"
  role_arn = aws_iam_role.eks_cluster.arn

  vpc_config {
    subnet_ids = concat(
      aws_subnet.private[*].id,
      aws_subnet.public[*].id,
    )

    security_group_ids = [aws_security_group.eks_cluster.id]

    # ✅ Endpoint access configuration
    endpoint_private_access = true   # ✅ Private access เปิด
    endpoint_public_access  = true   # สำหรับ initial setup (จะปิดหลังตั้งค่า)

    # ✅ จำกัด public access เฉพาะ IP ที่รู้จัก
    public_access_cidrs = var.allowed_public_cidrs  # ["YOUR_IP/32"]
  }

  # ✅ Enable cluster logging
  enabled_cluster_log_types = [
    "api",
    "audit",
    "authenticator",
    "controllerManager",
    "scheduler",
  ]

  # ✅ Encryption config สำหรับ Kubernetes secrets
  encryption_config {
    provider {
      key_arn = aws_kms_key.eks.arn
    }
    resources = ["secrets"]
  }

  # ✅ รอให้ IAM role policy attachment เสร็จก่อน
  depends_on = [
    aws_iam_role_policy_attachment.eks_cluster_policy,
  ]

  tags = {
    Name        = var.cluster_name
    Environment = var.environment
    ManagedBy   = "terraform"
  }
}

# ✅ KMS Key สำหรับ EKS secrets encryption
resource "aws_kms_key" "eks" {
  description             = "KMS key for EKS secrets encryption - ${var.cluster_name}"
  deletion_window_in_days = 7
  enable_key_rotation     = true

  tags = {
    Name      = "${var.cluster_name}-eks-kms"
    ManagedBy = "terraform"
  }
}

resource "aws_kms_alias" "eks" {
  name          = "alias/${var.cluster_name}-eks"
  target_key_id = aws_kms_key.eks.key_id
}
```

---

## Step 422: Security Groups สำหรับ EKS

```hcl
# ✅ Cluster Security Group
resource "aws_security_group" "eks_cluster" {
  name        = "${var.cluster_name}-cluster-sg"
  description = "EKS cluster security group"
  vpc_id      = aws_vpc.eks_vpc.id

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
    description = "Allow all outbound"
  }

  tags = {
    Name                                        = "${var.cluster_name}-cluster-sg"
    "kubernetes.io/cluster/${var.cluster_name}" = "owned"
    ManagedBy                                   = "terraform"
  }
}

# ✅ Node Security Group
resource "aws_security_group" "eks_nodes" {
  name        = "${var.cluster_name}-nodes-sg"
  description = "EKS worker nodes security group"
  vpc_id      = aws_vpc.eks_vpc.id

  # ✅ Nodes communicate กัน
  ingress {
    description = "Allow nodes to communicate with each other"
    from_port   = 0
    to_port     = 65535
    protocol    = "tcp"
    self        = true
  }

  # ✅ Control plane ติดต่อกับ nodes
  ingress {
    description     = "Allow control plane to communicate with nodes"
    from_port       = 1025
    to_port         = 65535
    protocol        = "tcp"
    security_groups = [aws_security_group.eks_cluster.id]
  }

  ingress {
    description     = "Allow control plane to communicate with nodes (HTTPS)"
    from_port       = 443
    to_port         = 443
    protocol        = "tcp"
    security_groups = [aws_security_group.eks_cluster.id]
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
    description = "Allow all outbound"
  }

  tags = {
    Name                                        = "${var.cluster_name}-nodes-sg"
    "kubernetes.io/cluster/${var.cluster_name}" = "owned"
    ManagedBy                                   = "terraform"
  }
}

# ✅ Security Group Rule - Allow nodes to control plane
resource "aws_security_group_rule" "nodes_to_cluster" {
  description              = "Allow nodes to communicate with cluster API server"
  from_port                = 443
  to_port                  = 443
  protocol                 = "tcp"
  security_group_id        = aws_security_group.eks_cluster.id
  source_security_group_id = aws_security_group.eks_nodes.id
  type                     = "ingress"
}
```

---

## Step 423: EKS Managed Node Groups

### Node Group IAM Role

```hcl
# ✅ Node Group IAM Role
data "aws_iam_policy_document" "eks_node_trust" {
  statement {
    effect = "Allow"
    principals {
      type        = "Service"
      identifiers = ["ec2.amazonaws.com"]
    }
    actions = ["sts:AssumeRole"]
  }
}

resource "aws_iam_role" "eks_nodes" {
  name               = "${var.cluster_name}-node-role"
  assume_role_policy = data.aws_iam_policy_document.eks_node_trust.json

  tags = {
    Name      = "${var.cluster_name}-node-role"
    ManagedBy = "terraform"
  }
}

# ✅ 3 Required policies สำหรับ EKS Node Groups
resource "aws_iam_role_policy_attachment" "eks_worker_node_policy" {
  policy_arn = "arn:aws:iam::aws:policy/AmazonEKSWorkerNodePolicy"
  role       = aws_iam_role.eks_nodes.name
}

resource "aws_iam_role_policy_attachment" "eks_cni_policy" {
  policy_arn = "arn:aws:iam::aws:policy/AmazonEKS_CNI_Policy"
  role       = aws_iam_role.eks_nodes.name
}

resource "aws_iam_role_policy_attachment" "ecr_read_only" {
  policy_arn = "arn:aws:iam::aws:policy/AmazonEC2ContainerRegistryReadOnly"
  role       = aws_iam_role.eks_nodes.name
}

# ✅ Optional: SSM for node management
resource "aws_iam_role_policy_attachment" "ssm_core" {
  policy_arn = "arn:aws:iam::aws:policy/AmazonSSMManagedInstanceCore"
  role       = aws_iam_role.eks_nodes.name
}
```

### aws_eks_node_group

```hcl
# ✅ System Node Group (สำหรับ system workloads)
resource "aws_eks_node_group" "system" {
  cluster_name    = aws_eks_cluster.main.name
  node_group_name = "${var.cluster_name}-system"
  node_role_arn   = aws_iam_role.eks_nodes.arn
  subnet_ids      = aws_subnet.private[*].id

  # ✅ ใช้ on-demand instances สำหรับ system nodes
  capacity_type  = "ON_DEMAND"
  instance_types = ["t3.medium"]

  scaling_config {
    desired_size = 2
    min_size     = 2
    max_size     = 4
  }

  update_config {
    max_unavailable = 1
  }

  # ✅ Node Group labels
  labels = {
    role        = "system"
    environment = var.environment
  }

  # ✅ Taints เพื่อป้องกัน user workloads บน system nodes
  taint {
    key    = "CriticalAddonsOnly"
    value  = "true"
    effect = "NO_SCHEDULE"
  }

  # ✅ AMI release version (pin version)
  # release_version = "1.28.3-20231116"

  depends_on = [
    aws_iam_role_policy_attachment.eks_worker_node_policy,
    aws_iam_role_policy_attachment.eks_cni_policy,
    aws_iam_role_policy_attachment.ecr_read_only,
  ]

  tags = {
    Name      = "${var.cluster_name}-system-node-group"
    ManagedBy = "terraform"
  }

  lifecycle {
    ignore_changes = [scaling_config[0].desired_size]
  }
}

# ✅ Application Node Group
resource "aws_eks_node_group" "application" {
  cluster_name    = aws_eks_cluster.main.name
  node_group_name = "${var.cluster_name}-application"
  node_role_arn   = aws_iam_role.eks_nodes.arn
  subnet_ids      = aws_subnet.private[*].id

  capacity_type  = "ON_DEMAND"
  instance_types = ["m5.large", "m5a.large", "m5n.large"]

  scaling_config {
    desired_size = 3
    min_size     = 1
    max_size     = 20
  }

  update_config {
    max_unavailable_percentage = 25  # อนุญาต 25% unavailable ระหว่าง update
  }

  labels = {
    role        = "application"
    environment = var.environment
  }

  depends_on = [
    aws_iam_role_policy_attachment.eks_worker_node_policy,
    aws_iam_role_policy_attachment.eks_cni_policy,
    aws_iam_role_policy_attachment.ecr_read_only,
  ]

  tags = {
    Name      = "${var.cluster_name}-app-node-group"
    ManagedBy = "terraform"
    # ✅ Karpenter/Cluster Autoscaler tags
    "k8s.io/cluster-autoscaler/enabled"             = "true"
    "k8s.io/cluster-autoscaler/${var.cluster_name}" = "owned"
  }

  lifecycle {
    ignore_changes = [scaling_config[0].desired_size]
  }
}

# ✅ Spot Node Group (ประหยัดค่าใช้จ่าย)
resource "aws_eks_node_group" "spot" {
  cluster_name    = aws_eks_cluster.main.name
  node_group_name = "${var.cluster_name}-spot"
  node_role_arn   = aws_iam_role.eks_nodes.arn
  subnet_ids      = aws_subnet.private[*].id

  # ✅ SPOT instances - ประหยัด 60-90%
  capacity_type = "SPOT"
  instance_types = [
    "m5.large", "m5a.large", "m5n.large",
    "m4.large",
    "m6i.large", "m6a.large",
  ]

  scaling_config {
    desired_size = 2
    min_size     = 0
    max_size     = 50
  }

  update_config {
    max_unavailable = 2
  }

  labels = {
    role             = "spot"
    capacity-type    = "spot"
    "spot-instance"  = "true"
  }

  # ✅ Taint สำหรับ spot nodes (workloads ต้อง tolerate)
  taint {
    key    = "spot"
    value  = "true"
    effect = "NO_SCHEDULE"
  }

  depends_on = [
    aws_iam_role_policy_attachment.eks_worker_node_policy,
    aws_iam_role_policy_attachment.eks_cni_policy,
    aws_iam_role_policy_attachment.ecr_read_only,
  ]

  tags = {
    Name      = "${var.cluster_name}-spot-node-group"
    ManagedBy = "terraform"
  }

  lifecycle {
    ignore_changes = [scaling_config[0].desired_size]
  }
}
```

### Launch Templates สำหรับ Node Groups

```hcl
# ✅ Launch Template สำหรับ custom node configuration
resource "aws_launch_template" "eks_nodes" {
  name        = "${var.cluster_name}-node-template"
  description = "Launch template for EKS nodes"

  # ✅ Block device mapping สำหรับ root volume
  block_device_mappings {
    device_name = "/dev/xvda"
    ebs {
      volume_size           = 50
      volume_type           = "gp3"
      encrypted             = true
      kms_key_id            = aws_kms_key.eks_ebs.arn
      delete_on_termination = true
    }
  }

  # ✅ IMDSv2 required (security)
  metadata_options {
    http_endpoint               = "enabled"
    http_tokens                 = "required"  # ✅ IMDSv2 required!
    http_put_response_hop_limit = 2
  }

  # ✅ Enable EBS optimization
  ebs_optimized = true

  # ✅ Monitoring
  monitoring {
    enabled = true
  }

  tag_specifications {
    resource_type = "instance"
    tags = {
      Name        = "${var.cluster_name}-node"
      Environment = var.environment
      ManagedBy   = "terraform"
    }
  }

  tag_specifications {
    resource_type = "volume"
    tags = {
      Name        = "${var.cluster_name}-node-volume"
      Environment = var.environment
      ManagedBy   = "terraform"
    }
  }

  lifecycle {
    create_before_destroy = true
  }
}

# ✅ Node Group ที่ใช้ Launch Template
resource "aws_eks_node_group" "custom" {
  cluster_name    = aws_eks_cluster.main.name
  node_group_name = "${var.cluster_name}-custom"
  node_role_arn   = aws_iam_role.eks_nodes.arn
  subnet_ids      = aws_subnet.private[*].id

  instance_types = ["m5.xlarge"]

  launch_template {
    id      = aws_launch_template.eks_nodes.id
    version = aws_launch_template.eks_nodes.latest_version
  }

  scaling_config {
    desired_size = 2
    min_size     = 1
    max_size     = 10
  }

  depends_on = [
    aws_iam_role_policy_attachment.eks_worker_node_policy,
    aws_iam_role_policy_attachment.eks_cni_policy,
    aws_iam_role_policy_attachment.ecr_read_only,
  ]

  tags = {
    Name      = "${var.cluster_name}-custom-node-group"
    ManagedBy = "terraform"
  }

  lifecycle {
    ignore_changes = [scaling_config[0].desired_size]
  }
}
```

---

## Step 424: EKS Fargate Profiles

### aws_eks_fargate_profile

```hcl
# ✅ Fargate IAM Role
data "aws_iam_policy_document" "fargate_trust" {
  statement {
    effect = "Allow"
    principals {
      type        = "Service"
      identifiers = ["eks-fargate-pods.amazonaws.com"]
    }
    actions = ["sts:AssumeRole"]
  }
}

resource "aws_iam_role" "fargate" {
  name               = "${var.cluster_name}-fargate-role"
  assume_role_policy = data.aws_iam_policy_document.fargate_trust.json

  tags = {
    Name      = "${var.cluster_name}-fargate-role"
    ManagedBy = "terraform"
  }
}

resource "aws_iam_role_policy_attachment" "fargate_pod_execution" {
  policy_arn = "arn:aws:iam::aws:policy/AmazonEKSFargatePodExecutionRolePolicy"
  role       = aws_iam_role.fargate.name
}

# ✅ Fargate Profile สำหรับ kube-system namespace
resource "aws_eks_fargate_profile" "kube_system" {
  cluster_name           = aws_eks_cluster.main.name
  fargate_profile_name   = "kube-system"
  pod_execution_role_arn = aws_iam_role.fargate.arn
  subnet_ids             = aws_subnet.private[*].id

  selector {
    namespace = "kube-system"
  }

  tags = {
    Name      = "${var.cluster_name}-fargate-kube-system"
    ManagedBy = "terraform"
  }
}

# ✅ Fargate Profile สำหรับ application namespace
resource "aws_eks_fargate_profile" "app" {
  cluster_name           = aws_eks_cluster.main.name
  fargate_profile_name   = "app"
  pod_execution_role_arn = aws_iam_role.fargate.arn
  subnet_ids             = aws_subnet.private[*].id

  selector {
    namespace = "app"
    labels = {
      "fargate" = "true"
    }
  }

  tags = {
    Name      = "${var.cluster_name}-fargate-app"
    ManagedBy = "terraform"
  }
}
```

---

## Step 425: EKS Add-ons

### aws_eks_addon

```hcl
# ✅ CoreDNS Add-on
resource "aws_eks_addon" "coredns" {
  cluster_name = aws_eks_cluster.main.name
  addon_name   = "coredns"
  # addon_version = "v1.10.1-eksbuild.5"  # Pin version

  resolve_conflicts_on_create = "OVERWRITE"
  resolve_conflicts_on_update = "OVERWRITE"

  depends_on = [
    aws_eks_node_group.system,  # CoreDNS ต้องการ nodes
  ]

  tags = {
    ManagedBy = "terraform"
  }
}

# ✅ kube-proxy Add-on
resource "aws_eks_addon" "kube_proxy" {
  cluster_name = aws_eks_cluster.main.name
  addon_name   = "kube-proxy"

  resolve_conflicts_on_create = "OVERWRITE"
  resolve_conflicts_on_update = "OVERWRITE"

  tags = {
    ManagedBy = "terraform"
  }
}

# ✅ VPC CNI Add-on
resource "aws_eks_addon" "vpc_cni" {
  cluster_name             = aws_eks_cluster.main.name
  addon_name               = "vpc-cni"
  service_account_role_arn = aws_iam_role.vpc_cni.arn  # IRSA

  resolve_conflicts_on_create = "OVERWRITE"
  resolve_conflicts_on_update = "OVERWRITE"

  configuration_values = jsonencode({
    env = {
      ENABLE_PREFIX_DELEGATION = "true"  # ✅ เพิ่มจำนวน pods ต่อ node
      WARM_PREFIX_TARGET       = "1"
    }
  })

  tags = {
    ManagedBy = "terraform"
  }
}

# ✅ EBS CSI Driver Add-on (สำหรับ persistent volumes)
resource "aws_eks_addon" "ebs_csi_driver" {
  cluster_name             = aws_eks_cluster.main.name
  addon_name               = "aws-ebs-csi-driver"
  service_account_role_arn = aws_iam_role.ebs_csi_driver.arn  # IRSA

  resolve_conflicts_on_create = "OVERWRITE"
  resolve_conflicts_on_update = "OVERWRITE"

  depends_on = [
    aws_eks_node_group.system,
  ]

  tags = {
    ManagedBy = "terraform"
  }
}

# ✅ EFS CSI Driver Add-on (สำหรับ shared persistent volumes)
resource "aws_eks_addon" "efs_csi_driver" {
  cluster_name             = aws_eks_cluster.main.name
  addon_name               = "aws-efs-csi-driver"
  service_account_role_arn = aws_iam_role.efs_csi_driver.arn

  resolve_conflicts_on_create = "OVERWRITE"
  resolve_conflicts_on_update = "OVERWRITE"

  tags = {
    ManagedBy = "terraform"
  }
}
```

---

## Step 426: IRSA - IAM Roles for Service Accounts

```hcl
# ✅ OIDC Provider สำหรับ EKS IRSA
data "tls_certificate" "eks" {
  url = aws_eks_cluster.main.identity[0].oidc[0].issuer
}

resource "aws_iam_openid_connect_provider" "eks" {
  client_id_list  = ["sts.amazonaws.com"]
  thumbprint_list = [data.tls_certificate.eks.certificates[0].sha1_fingerprint]
  url             = aws_eks_cluster.main.identity[0].oidc[0].issuer

  tags = {
    Name      = "${var.cluster_name}-oidc"
    ManagedBy = "terraform"
  }
}

locals {
  oidc_issuer     = trimprefix(aws_eks_cluster.main.identity[0].oidc[0].issuer, "https://")
  oidc_issuer_arn = aws_iam_openid_connect_provider.eks.arn
}

# Helper function สร้าง trust policy สำหรับ service account
# ✅ VPC CNI IRSA Role
data "aws_iam_policy_document" "vpc_cni_trust" {
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
      values   = ["system:serviceaccount:kube-system:aws-node"]
    }

    condition {
      test     = "StringEquals"
      variable = "${local.oidc_issuer}:aud"
      values   = ["sts.amazonaws.com"]
    }
  }
}

resource "aws_iam_role" "vpc_cni" {
  name               = "${var.cluster_name}-vpc-cni"
  assume_role_policy = data.aws_iam_policy_document.vpc_cni_trust.json

  tags = {
    Name      = "${var.cluster_name}-vpc-cni"
    ManagedBy = "terraform"
  }
}

resource "aws_iam_role_policy_attachment" "vpc_cni" {
  policy_arn = "arn:aws:iam::aws:policy/AmazonEKS_CNI_Policy"
  role       = aws_iam_role.vpc_cni.name
}

# ✅ EBS CSI Driver IRSA Role
data "aws_iam_policy_document" "ebs_csi_trust" {
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
      values   = ["system:serviceaccount:kube-system:ebs-csi-controller-sa"]
    }

    condition {
      test     = "StringEquals"
      variable = "${local.oidc_issuer}:aud"
      values   = ["sts.amazonaws.com"]
    }
  }
}

resource "aws_iam_role" "ebs_csi_driver" {
  name               = "${var.cluster_name}-ebs-csi-driver"
  assume_role_policy = data.aws_iam_policy_document.ebs_csi_trust.json

  tags = {
    Name      = "${var.cluster_name}-ebs-csi-driver"
    ManagedBy = "terraform"
  }
}

resource "aws_iam_role_policy_attachment" "ebs_csi_driver" {
  policy_arn = "arn:aws:iam::aws:policy/service-role/AmazonEBSCSIDriverPolicy"
  role       = aws_iam_role.ebs_csi_driver.name
}
```

---

## Step 427: Kubernetes Provider Configuration

```hcl
# ✅ Kubernetes Provider - ใช้ EKS token
data "aws_eks_cluster" "main" {
  name = aws_eks_cluster.main.name
}

data "aws_eks_cluster_auth" "main" {
  name = aws_eks_cluster.main.name
}

provider "kubernetes" {
  host                   = data.aws_eks_cluster.main.endpoint
  cluster_ca_certificate = base64decode(data.aws_eks_cluster.main.certificate_authority[0].data)
  token                  = data.aws_eks_cluster_auth.main.token
}

# ✅ Helm Provider
provider "helm" {
  kubernetes {
    host                   = data.aws_eks_cluster.main.endpoint
    cluster_ca_certificate = base64decode(data.aws_eks_cluster.main.certificate_authority[0].data)
    token                  = data.aws_eks_cluster_auth.main.token
  }
}

# ✅ แบบที่ดีกว่า: ใช้ exec สำหรับ dynamic token refresh
provider "kubernetes" {
  host                   = aws_eks_cluster.main.endpoint
  cluster_ca_certificate = base64decode(aws_eks_cluster.main.certificate_authority[0].data)

  exec {
    api_version = "client.authentication.k8s.io/v1beta1"
    args        = ["eks", "get-token", "--cluster-name", aws_eks_cluster.main.name]
    command     = "aws"
  }
}

provider "helm" {
  kubernetes {
    host                   = aws_eks_cluster.main.endpoint
    cluster_ca_certificate = base64decode(aws_eks_cluster.main.certificate_authority[0].data)

    exec {
      api_version = "client.authentication.k8s.io/v1beta1"
      args        = ["eks", "get-token", "--cluster-name", aws_eks_cluster.main.name]
      command     = "aws"
    }
  }
}
```

---

## Step 428: Helm Deployments

```hcl
# ✅ Install AWS Load Balancer Controller ด้วย Helm
resource "helm_release" "aws_load_balancer_controller" {
  name       = "aws-load-balancer-controller"
  repository = "https://aws.github.io/eks-charts"
  chart      = "aws-load-balancer-controller"
  namespace  = "kube-system"
  version    = "1.6.2"

  set {
    name  = "clusterName"
    value = aws_eks_cluster.main.name
  }

  set {
    name  = "serviceAccount.create"
    value = "true"
  }

  set {
    name  = "serviceAccount.annotations.eks\\.amazonaws\\.com/role-arn"
    value = aws_iam_role.alb_controller.arn
  }

  set {
    name  = "replicaCount"
    value = "2"
  }

  depends_on = [
    aws_eks_addon.coredns,
    aws_iam_role_policy_attachment.alb_controller,
  ]
}

# ✅ Install External DNS ด้วย Helm
resource "helm_release" "external_dns" {
  name       = "external-dns"
  repository = "https://kubernetes-sigs.github.io/external-dns/"
  chart      = "external-dns"
  namespace  = "kube-system"
  version    = "1.13.1"

  values = [
    yamlencode({
      provider = "aws"
      aws = {
        region = var.aws_region
      }
      serviceAccount = {
        annotations = {
          "eks.amazonaws.com/role-arn" = aws_iam_role.external_dns.arn
        }
      }
      domainFilters = [var.domain_name]
      policy        = "sync"
      txtOwnerId    = aws_eks_cluster.main.name
    })
  ]

  depends_on = [
    aws_eks_addon.coredns,
  ]
}

# ✅ Install Cluster Autoscaler
resource "helm_release" "cluster_autoscaler" {
  name       = "cluster-autoscaler"
  repository = "https://kubernetes.github.io/autoscaler"
  chart      = "cluster-autoscaler"
  namespace  = "kube-system"
  version    = "9.29.3"

  set {
    name  = "autoDiscovery.clusterName"
    value = aws_eks_cluster.main.name
  }

  set {
    name  = "awsRegion"
    value = var.aws_region
  }

  set {
    name  = "rbac.serviceAccount.annotations.eks\\.amazonaws\\.com/role-arn"
    value = aws_iam_role.cluster_autoscaler.arn
  }

  depends_on = [
    aws_eks_node_group.system,
  ]
}

# ✅ Install Metrics Server
resource "helm_release" "metrics_server" {
  name       = "metrics-server"
  repository = "https://kubernetes-sigs.github.io/metrics-server/"
  chart      = "metrics-server"
  namespace  = "kube-system"
  version    = "3.11.0"

  set {
    name  = "replicas"
    value = "2"
  }

  depends_on = [
    aws_eks_node_group.system,
  ]
}
```

---

## Step 429: kubeconfig Generation

```hcl
# ✅ สร้าง kubeconfig file
resource "local_file" "kubeconfig" {
  filename = "${path.module}/kubeconfig"
  content  = <<-EOT
    apiVersion: v1
    kind: Config
    clusters:
    - cluster:
        certificate-authority-data: ${aws_eks_cluster.main.certificate_authority[0].data}
        server: ${aws_eks_cluster.main.endpoint}
      name: ${aws_eks_cluster.main.name}
    contexts:
    - context:
        cluster: ${aws_eks_cluster.main.name}
        user: ${aws_eks_cluster.main.name}
      name: ${aws_eks_cluster.main.name}
    current-context: ${aws_eks_cluster.main.name}
    users:
    - name: ${aws_eks_cluster.main.name}
      user:
        exec:
          apiVersion: client.authentication.k8s.io/v1beta1
          command: aws
          args:
            - eks
            - get-token
            - --cluster-name
            - ${aws_eks_cluster.main.name}
            - --region
            - ${var.aws_region}
  EOT

  file_permission = "0600"  # ✅ Restrict permissions

  depends_on = [aws_eks_cluster.main]
}

# ✅ หรือใช้ null_resource เพื่อ update kubeconfig
resource "null_resource" "update_kubeconfig" {
  triggers = {
    cluster_arn = aws_eks_cluster.main.arn
  }

  provisioner "local-exec" {
    command = "aws eks update-kubeconfig --region ${var.aws_region} --name ${aws_eks_cluster.main.name}"
  }

  depends_on = [aws_eks_cluster.main]
}
```

---

## Step 430: Complete Production EKS Module

### module/eks/main.tf

```hcl
# ✅ Complete Production EKS Setup
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
    kubernetes = {
      source  = "hashicorp/kubernetes"
      version = "~> 2.23"
    }
    helm = {
      source  = "hashicorp/helm"
      version = "~> 2.11"
    }
    tls = {
      source  = "hashicorp/tls"
      version = "~> 4.0"
    }
  }
}

# Locals
locals {
  cluster_name    = "${var.project_name}-${var.environment}"
  oidc_issuer     = trimprefix(aws_eks_cluster.main.identity[0].oidc[0].issuer, "https://")
  oidc_issuer_arn = aws_iam_openid_connect_provider.eks.arn

  common_tags = {
    Project     = var.project_name
    Environment = var.environment
    ManagedBy   = "terraform"
    Cluster     = local.cluster_name
  }
}

# EKS Cluster
resource "aws_eks_cluster" "main" {
  name     = local.cluster_name
  version  = var.kubernetes_version
  role_arn = aws_iam_role.eks_cluster.arn

  vpc_config {
    subnet_ids              = concat(var.private_subnet_ids, var.public_subnet_ids)
    security_group_ids      = [aws_security_group.eks_cluster.id]
    endpoint_private_access = true
    endpoint_public_access  = var.enable_public_endpoint
    public_access_cidrs     = var.public_access_cidrs
  }

  enabled_cluster_log_types = var.cluster_log_types

  encryption_config {
    provider {
      key_arn = aws_kms_key.eks.arn
    }
    resources = ["secrets"]
  }

  depends_on = [
    aws_iam_role_policy_attachment.eks_cluster_policy,
    aws_cloudwatch_log_group.eks,
  ]

  tags = local.common_tags
}

# CloudWatch Log Group สำหรับ EKS control plane logs
resource "aws_cloudwatch_log_group" "eks" {
  name              = "/aws/eks/${local.cluster_name}/cluster"
  retention_in_days = 30  # ✅ Log retention policy
  kms_key_id        = aws_kms_key.eks.arn

  tags = local.common_tags
}

# OIDC Provider
data "tls_certificate" "eks" {
  url = aws_eks_cluster.main.identity[0].oidc[0].issuer
}

resource "aws_iam_openid_connect_provider" "eks" {
  client_id_list  = ["sts.amazonaws.com"]
  thumbprint_list = [data.tls_certificate.eks.certificates[0].sha1_fingerprint]
  url             = aws_eks_cluster.main.identity[0].oidc[0].issuer

  tags = local.common_tags
}
```

### module/eks/variables.tf

```hcl
variable "project_name" {
  description = "Project name"
  type        = string
}

variable "environment" {
  description = "Environment name (production, staging, development)"
  type        = string
}

variable "aws_region" {
  description = "AWS region"
  type        = string
}

variable "kubernetes_version" {
  description = "Kubernetes version for EKS cluster"
  type        = string
  default     = "1.28"
}

variable "vpc_id" {
  description = "VPC ID"
  type        = string
}

variable "private_subnet_ids" {
  description = "Private subnet IDs for worker nodes"
  type        = list(string)
}

variable "public_subnet_ids" {
  description = "Public subnet IDs for load balancers"
  type        = list(string)
}

variable "node_groups" {
  description = "Node group configurations"
  type = map(object({
    instance_types = list(string)
    capacity_type  = string
    min_size       = number
    max_size       = number
    desired_size   = number
    labels         = map(string)
    taints = list(object({
      key    = string
      value  = string
      effect = string
    }))
  }))
  default = {
    system = {
      instance_types = ["t3.medium"]
      capacity_type  = "ON_DEMAND"
      min_size       = 2
      max_size       = 4
      desired_size   = 2
      labels = {
        role = "system"
      }
      taints = []
    }
  }
}

variable "enable_public_endpoint" {
  description = "Enable public API endpoint"
  type        = bool
  default     = true
}

variable "public_access_cidrs" {
  description = "CIDRs allowed for public API access"
  type        = list(string)
  default     = ["0.0.0.0/0"]  # ✅ ควรจำกัดให้แคบกว่านี้ใน production
}

variable "cluster_log_types" {
  description = "EKS cluster log types to enable"
  type        = list(string)
  default = [
    "api",
    "audit",
    "authenticator",
    "controllerManager",
    "scheduler",
  ]
}

variable "domain_name" {
  description = "Domain name for External DNS"
  type        = string
  default     = ""
}
```

### module/eks/outputs.tf

```hcl
output "cluster_id" {
  description = "EKS cluster ID"
  value       = aws_eks_cluster.main.id
}

output "cluster_name" {
  description = "EKS cluster name"
  value       = aws_eks_cluster.main.name
}

output "cluster_endpoint" {
  description = "EKS cluster API endpoint"
  value       = aws_eks_cluster.main.endpoint
}

output "cluster_certificate_authority_data" {
  description = "EKS cluster CA data"
  value       = aws_eks_cluster.main.certificate_authority[0].data
}

output "cluster_arn" {
  description = "EKS cluster ARN"
  value       = aws_eks_cluster.main.arn
}

output "oidc_provider_arn" {
  description = "OIDC provider ARN"
  value       = aws_iam_openid_connect_provider.eks.arn
}

output "oidc_provider_url" {
  description = "OIDC provider URL"
  value       = aws_eks_cluster.main.identity[0].oidc[0].issuer
}

output "node_role_arn" {
  description = "IAM role ARN for EKS nodes"
  value       = aws_iam_role.eks_nodes.arn
}

output "cluster_security_group_id" {
  description = "Security group ID for EKS cluster"
  value       = aws_security_group.eks_cluster.id
}

output "node_security_group_id" {
  description = "Security group ID for EKS nodes"
  value       = aws_security_group.eks_nodes.id
}
```

---

## EKS Security Checklist

### ✅ สิ่งที่ควรทำ

1. **Encryption**: เปิด secrets encryption ด้วย KMS
2. **Private Nodes**: Worker nodes ใน private subnets
3. **IMDSv2**: บังคับใช้ IMDSv2 ใน launch template
4. **IRSA**: ใช้ IRSA แทน node-level IAM roles
5. **Network Policy**: ติดตั้ง network policy enforcement
6. **Control Plane Logging**: เปิด log ทั้งหมด
7. **Node Groups**: แยก system/application node groups
8. **Taints**: ใช้ taints เพื่อจัดการ workload placement

### ❌ สิ่งที่ไม่ควรทำ

1. ❌ Public worker nodes
2. ❌ `endpoint_public_access = true` โดยไม่จำกัด CIDRs
3. ❌ ใช้ wildcard node IAM permissions
4. ❌ ไม่มี cluster autoscaler
5. ❌ ไม่มี secrets encryption

---

**Next Steps**: ไปต่อที่ Part 044 - AWS Lambda Functions
