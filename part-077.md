# Part 077: OPA (Open Policy Agent) กับ Terraform (ขั้นตอนที่ 761-770)

## บทนำ (Introduction)

Open Policy Agent (OPA) เป็น general-purpose policy engine แบบ open-source
ใช้ภาษา Rego สำหรับเขียน policies และสามารถ integrate กับ Terraform ผ่าน conftest tool

---

## ขั้นตอนที่ 761: What is OPA?

### OPA Architecture

```
┌─────────────────────────────────────────────────────────┐
│                     OPA Engine                           │
│                                                         │
│  Input (JSON) + Policy (Rego) → Decision (JSON)         │
│                                                         │
│  Input ตัวอย่าง:                                         │
│  - Terraform plan JSON                                  │
│  - Kubernetes admission requests                        │
│  - HTTP API requests                                    │
│  - Docker image tags                                    │
└─────────────────────────────────────────────────────────┘
```

### OPA vs Sentinel เปรียบเทียบ

| Aspect | OPA/Rego | Sentinel |
|--------|----------|----------|
| License | Open Source | Commercial |
| Language | Rego | Sentinel |
| Integration | Any platform | HashiCorp only |
| TFC/TFE Native | No | Yes |
| Kubernetes | Yes (Gatekeeper) | No |
| Docker | Yes | No |
| Learning Curve | Steeper | Easier |
| Community | Larger | Smaller |

---

## ขั้นตอนที่ 762: Rego Language Basics

### Rego Fundamentals

```rego
# hello.rego
package hello

# Simple rule
greeting = "Hello, World!"

# Conditional rule
allow {
  input.method == "GET"
  input.path[0] == "public"
}

# Default rule
default allow = false

# Rule with multiple conditions
allow {
  input.user.role == "admin"
}

allow {
  input.user.role == "developer"
  input.method == "GET"
}
```

### Rego Data Types

```rego
package example

# String
name := "production"

# Number
max_count := 10

# Boolean
is_enabled := true

# Array
allowed_regions := ["us-east-1", "us-west-2", "eu-west-1"]

# Object/Map
config := {
  "environment": "prod",
  "region": "us-east-1",
}

# Set
allowed_methods := {"GET", "POST", "PUT"}

# Null
empty_value := null
```

### Rego Comprehensions

```rego
package example

# Array comprehension
ec2_instances := [instance | 
  instance := input.resource_changes[_]
  instance.type == "aws_instance"
]

# Object comprehension
instance_types := {address: type |
  resource := input.resource_changes[_]
  resource.type == "aws_instance"
  address := resource.address
  type := resource.change.after.instance_type
}

# Set comprehension
unique_regions := {region |
  resource := input.resource_changes[_]
  region := resource.change.after.region
}
```

---

## ขั้นตอนที่ 763: conftest Tool

### ติดตั้ง conftest

```bash
# macOS
brew install conftest

# Linux
wget https://github.com/open-policy-agent/conftest/releases/download/v0.50.0/conftest_0.50.0_Linux_x86_64.tar.gz
tar xzf conftest_0.50.0_Linux_x86_64.tar.gz
mv conftest /usr/local/bin/

# Verify
conftest --version
# conftest version 0.50.0
```

### conftest Project Structure

```
project/
├── main.tf
├── variables.tf
├── policy/           ← Rego policies
│   ├── main.rego
│   ├── aws/
│   │   ├── s3.rego
│   │   ├── ec2.rego
│   │   ├── iam.rego
│   │   └── vpc.rego
│   └── general/
│       ├── tags.rego
│       └── naming.rego
└── conftest.yaml     ← optional configuration
```

### conftest.yaml configuration

```yaml
# conftest.yaml
policy:
  - policy/
  - https://raw.githubusercontent.com/mycompany/policies/main/aws/

namespace: terraform

output: table
```

---

## ขั้นตอนที่ 764: การสร้าง Terraform Plan JSON สำหรับ OPA

### Generate Plan JSON

```bash
# Step 1: สร้าง plan file
terraform plan -out=tfplan.binary

# Step 2: แปลง binary plan เป็น JSON
terraform show -json tfplan.binary > tfplan.json

# Step 3: รัน conftest
conftest test tfplan.json -p policy/

# ทำทั้งหมดใน pipeline:
#!/bin/bash
terraform init
terraform plan -out=tfplan.binary
terraform show -json tfplan.binary > tfplan.json
conftest test tfplan.json --policy policy/ --namespace main
```

### โครงสร้าง tfplan.json

```json
{
  "format_version": "1.2",
  "terraform_version": "1.7.0",
  "variables": {
    "environment": { "value": "production" }
  },
  "planned_values": {
    "root_module": {
      "resources": [...]
    }
  },
  "resource_changes": [
    {
      "address": "aws_instance.web",
      "module_address": null,
      "type": "aws_instance",
      "name": "web",
      "provider_name": "registry.terraform.io/hashicorp/aws",
      "change": {
        "actions": ["create"],
        "before": null,
        "after": {
          "ami": "ami-12345",
          "instance_type": "t3.micro",
          "tags": {
            "Environment": "production",
            "Name": "web-server"
          }
        },
        "after_unknown": {}
      }
    }
  ],
  "configuration": { ... }
}
```

---

## ขั้นตอนที่ 765: Writing Rego Policies สำหรับ Terraform

### Policy: Deny Public S3 Buckets

```rego
# policy/aws/s3.rego
package main

import future.keywords.contains
import future.keywords.if
import future.keywords.in

# Public ACLs ที่ไม่อนุญาต
public_acls := {"public-read", "public-read-write", "authenticated-read"}

# ดึง S3 bucket ACL changes
s3_acl_changes[rc] {
  rc := input.resource_changes[_]
  rc.type == "aws_s3_bucket_acl"
  rc.change.actions[_] in {"create", "update"}
}

# Violations: S3 buckets ที่มี public ACL
deny contains msg if {
  rc := s3_acl_changes[_]
  rc.change.after.acl in public_acls
  msg := sprintf("S3 bucket ACL '%s' uses public ACL '%s'. Public S3 buckets are not allowed.",
    [rc.address, rc.change.after.acl])
}

# ตรวจสอบ Public Access Block
deny contains msg if {
  rc := input.resource_changes[_]
  rc.type == "aws_s3_bucket_public_access_block"
  rc.change.actions[_] in {"create", "update"}
  not rc.change.after.block_public_acls == true
  msg := sprintf("S3 Public Access Block '%s' must have block_public_acls = true",
    [rc.address])
}

deny contains msg if {
  rc := input.resource_changes[_]
  rc.type == "aws_s3_bucket_public_access_block"
  rc.change.actions[_] in {"create", "update"}
  not rc.change.after.block_public_policy == true
  msg := sprintf("S3 Public Access Block '%s' must have block_public_policy = true",
    [rc.address])
}
```

### Policy: Require Encryption

```rego
# policy/aws/encryption.rego
package main

import future.keywords.contains
import future.keywords.if
import future.keywords.in

# EBS Volume encryption
deny contains msg if {
  rc := input.resource_changes[_]
  rc.type == "aws_ebs_volume"
  rc.change.actions[_] == "create"
  not rc.change.after.encrypted == true
  msg := sprintf("EBS volume '%s' must be encrypted. Add 'encrypted = true'.",
    [rc.address])
}

# RDS encryption
deny contains msg if {
  rc := input.resource_changes[_]
  rc.type == "aws_db_instance"
  rc.change.actions[_] == "create"
  not rc.change.after.storage_encrypted == true
  msg := sprintf("RDS instance '%s' must have storage_encrypted = true",
    [rc.address])
}

# S3 server-side encryption
deny contains msg if {
  s3_buckets := {addr |
    rc := input.resource_changes[_]
    rc.type == "aws_s3_bucket"
    rc.change.actions[_] == "create"
    addr := rc.address
  }
  
  encrypted_buckets := {addr |
    rc := input.resource_changes[_]
    rc.type == "aws_s3_bucket_server_side_encryption_configuration"
    rc.change.actions[_] == "create"
    addr := rc.address
  }
  
  bucket := s3_buckets[_]
  not bucket in encrypted_buckets
  msg := sprintf("S3 bucket '%s' must have server-side encryption configured", [bucket])
}

# ElastiCache encryption
deny contains msg if {
  rc := input.resource_changes[_]
  rc.type == "aws_elasticache_replication_group"
  rc.change.actions[_] == "create"
  not rc.change.after.at_rest_encryption_enabled == true
  msg := sprintf("ElastiCache '%s' must have at_rest_encryption_enabled = true",
    [rc.address])
}
```

### Policy: Check Instance Types

```rego
# policy/aws/ec2.rego
package main

import future.keywords.contains
import future.keywords.if
import future.keywords.in

# Allowed instance type families
allowed_instance_families := {"t3", "t3a", "m5", "m5a", "c5", "c5a", "r5"}

# Extract instance family from instance type
# e.g., "t3.micro" -> "t3"
instance_family(instance_type) := family if {
  parts := split(instance_type, ".")
  family := parts[0]
}

# Get all new EC2 instances
new_ec2_instances[rc] {
  rc := input.resource_changes[_]
  rc.type == "aws_instance"
  rc.change.actions[_] == "create"
}

# Deny non-approved instance types
deny contains msg if {
  rc := new_ec2_instances[_]
  instance_type := rc.change.after.instance_type
  family := instance_family(instance_type)
  not family in allowed_instance_families
  msg := sprintf(
    "EC2 instance '%s' uses disallowed instance type '%s'. Allowed families: %v",
    [rc.address, instance_type, allowed_instance_families]
  )
}

# Warn about large instances (soft check)
warn contains msg if {
  rc := new_ec2_instances[_]
  instance_type := rc.change.after.instance_type
  endswith(instance_type, ".4xlarge")
  msg := sprintf(
    "WARNING: EC2 instance '%s' uses large instance type '%s'. Please confirm this is necessary.",
    [rc.address, instance_type]
  )
}
```

### Policy: Enforce Tagging

```rego
# policy/general/tags.rego
package main

import future.keywords.contains
import future.keywords.if
import future.keywords.in

# Required tags
required_tags := {"Environment", "Project", "Owner", "CostCenter"}

# Resource types ที่ต้องมี tags
taggable_types := {
  "aws_instance",
  "aws_s3_bucket",
  "aws_vpc",
  "aws_subnet",
  "aws_security_group",
  "aws_db_instance",
  "aws_elasticache_replication_group",
  "aws_lb",
}

# Get all taggable resources being created/updated
taggable_resources[rc] {
  rc := input.resource_changes[_]
  rc.type in taggable_types
  rc.change.actions[_] in {"create", "update"}
}

# Check each required tag
deny contains msg if {
  rc := taggable_resources[_]
  required_tag := required_tags[_]
  
  # Tag missing entirely
  not rc.change.after.tags[required_tag]
  
  msg := sprintf(
    "Resource '%s' (type: %s) is missing required tag '%s'",
    [rc.address, rc.type, required_tag]
  )
}

deny contains msg if {
  rc := taggable_resources[_]
  required_tag := required_tags[_]
  
  # Tag exists but is empty
  rc.change.after.tags[required_tag] == ""
  
  msg := sprintf(
    "Resource '%s' has empty value for required tag '%s'",
    [rc.address, required_tag]
  )
}

# Warn about non-standard environment values
valid_environments := {"dev", "staging", "prod", "test", "sandbox"}

warn contains msg if {
  rc := taggable_resources[_]
  env := rc.change.after.tags["Environment"]
  env != null
  not env in valid_environments
  msg := sprintf(
    "Resource '%s' has non-standard Environment tag '%s'. Use one of: %v",
    [rc.address, env, valid_environments]
  )
}
```

### Policy: Network Access Restrictions

```rego
# policy/aws/networking.rego
package main

import future.keywords.contains
import future.keywords.if
import future.keywords.in

# SSH (port 22) ไม่อนุญาต public access
deny contains msg if {
  rc := input.resource_changes[_]
  rc.type == "aws_security_group_rule"
  rc.change.actions[_] in {"create", "update"}
  rc.change.after.type == "ingress"
  rc.change.after.from_port <= 22
  rc.change.after.to_port >= 22
  rc.change.after.cidr_blocks[_] == "0.0.0.0/0"
  msg := sprintf(
    "Security group rule '%s' allows SSH (port 22) from 0.0.0.0/0. This is not allowed.",
    [rc.address]
  )
}

# RDP (port 3389) ไม่อนุญาต public access
deny contains msg if {
  rc := input.resource_changes[_]
  rc.type == "aws_security_group_rule"
  rc.change.actions[_] in {"create", "update"}
  rc.change.after.type == "ingress"
  rc.change.after.from_port <= 3389
  rc.change.after.to_port >= 3389
  rc.change.after.cidr_blocks[_] == "0.0.0.0/0"
  msg := sprintf(
    "Security group rule '%s' allows RDP (port 3389) from 0.0.0.0/0. This is not allowed.",
    [rc.address]
  )
}

# Database ports ไม่อนุญาต public access
database_ports := {3306, 5432, 1521, 1433, 27017}

deny contains msg if {
  rc := input.resource_changes[_]
  rc.type == "aws_security_group_rule"
  rc.change.actions[_] in {"create", "update"}
  rc.change.after.type == "ingress"
  port := database_ports[_]
  rc.change.after.from_port <= port
  rc.change.after.to_port >= port
  rc.change.after.cidr_blocks[_] == "0.0.0.0/0"
  msg := sprintf(
    "Security group rule '%s' allows database port %d from 0.0.0.0/0. Databases must not be publicly accessible.",
    [rc.address, port]
  )
}

# VPC ต้องมี VPC flow logs
warn contains msg if {
  rc := input.resource_changes[_]
  rc.type == "aws_vpc"
  rc.change.actions[_] == "create"
  
  vpc_flow_logs := {addr |
    flow_log := input.resource_changes[_]
    flow_log.type == "aws_flow_log"
    flow_log.change.actions[_] == "create"
    addr := flow_log.address
  }
  
  count(vpc_flow_logs) == 0
  
  msg := sprintf(
    "VPC '%s' does not have VPC Flow Logs configured. Consider adding aws_flow_log resource.",
    [rc.address]
  )
}
```

### Policy: IAM Permissions Validation

```rego
# policy/aws/iam.rego
package main

import future.keywords.contains
import future.keywords.if
import future.keywords.in

# Deny overly permissive IAM policies
deny contains msg if {
  rc := input.resource_changes[_]
  rc.type in {"aws_iam_policy", "aws_iam_user_policy", "aws_iam_role_policy"}
  rc.change.actions[_] in {"create", "update"}
  
  # Parse policy document
  policy_doc := json.unmarshal(rc.change.after.policy)
  statement := policy_doc.Statement[_]
  
  # Action * ทั้งหมด
  statement.Effect == "Allow"
  statement.Action == "*"
  statement.Resource == "*"
  
  msg := sprintf(
    "IAM policy '%s' grants Action:* on Resource:* which is too permissive. Use least privilege.",
    [rc.address]
  )
}

# ไม่อนุญาต inline policies บน users
deny contains msg if {
  rc := input.resource_changes[_]
  rc.type == "aws_iam_user_policy"  # inline policy
  rc.change.actions[_] == "create"
  msg := sprintf(
    "Inline IAM policy '%s' on user is not allowed. Use managed policies instead.",
    [rc.address]
  )
}

# IAM role ต้องมี description
deny contains msg if {
  rc := input.resource_changes[_]
  rc.type == "aws_iam_role"
  rc.change.actions[_] == "create"
  not rc.change.after.description
  msg := sprintf(
    "IAM role '%s' must have a description",
    [rc.address]
  )
}
```

---

## ขั้นตอนที่ 766: Policy Package Structure

### Multiple Packages

```rego
# policy/aws/main.rego - รวม all AWS policies
package main

# Import จาก sub-packages
import data.terraform.aws.s3
import data.terraform.aws.ec2
import data.terraform.aws.iam
import data.terraform.aws.networking

# Combine all denys
deny[msg] {
  s3.deny[msg]
}

deny[msg] {
  ec2.deny[msg]
}

deny[msg] {
  iam.deny[msg]
}

deny[msg] {
  networking.deny[msg]
}
```

### ใช้ conftest namespaces

```bash
# รัน tests ด้วย specific namespace
conftest test tfplan.json \
  --policy policy/ \
  --namespace terraform.aws

# รัน หลาย namespaces
conftest test tfplan.json \
  --policy policy/aws/ \
  --policy policy/general/ \
  --namespace main
```

---

## ขั้นตอนที่ 767: conftest ใน CI/CD

### GitHub Actions Pipeline

```yaml
# .github/workflows/terraform-policy.yml
name: Terraform Policy Check

on:
  pull_request:
    paths:
      - '**.tf'
      - '**.tfvars'

jobs:
  policy-check:
    runs-on: ubuntu-latest
    name: OPA Policy Check

    steps:
      - uses: actions/checkout@v4

      - name: Setup Terraform
        uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: "1.7.0"

      - name: Install conftest
        run: |
          CONFTEST_VERSION=0.50.0
          wget -q https://github.com/open-policy-agent/conftest/releases/download/v${CONFTEST_VERSION}/conftest_${CONFTEST_VERSION}_Linux_x86_64.tar.gz
          tar xzf conftest_${CONFTEST_VERSION}_Linux_x86_64.tar.gz
          sudo mv conftest /usr/local/bin/

      - name: Terraform Init
        run: terraform init
        env:
          AWS_ACCESS_KEY_ID:     ${{ secrets.AWS_ACCESS_KEY_ID }}
          AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}

      - name: Terraform Plan
        run: |
          terraform plan -out=tfplan.binary
          terraform show -json tfplan.binary > tfplan.json
        env:
          AWS_ACCESS_KEY_ID:     ${{ secrets.AWS_ACCESS_KEY_ID }}
          AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}

      - name: Run Policy Checks
        run: |
          conftest test tfplan.json \
            --policy policy/ \
            --namespace main \
            --output table \
            --no-fail  # แสดงผลแต่ไม่ fail CI (สำหรับ warning-only)

      - name: Strict Policy Check
        run: |
          conftest test tfplan.json \
            --policy policy/critical/ \
            --namespace main \
            --output table
        # ไม่มี --no-fail = fail ถ้าผ่าน policy ไม่ได้

      - name: Upload Policy Results
        if: always()
        uses: actions/upload-artifact@v3
        with:
          name: policy-results
          path: tfplan.json
```

### conftest Pull Results สำหรับ Shared Policies

```bash
# ดึง policies จาก OCI registry
conftest pull ghcr.io/mycompany/terraform-policies:latest

# หรือจาก GitHub
conftest pull github.com/mycompany/terraform-policies//aws

# หรือระบุใน conftest.yaml
```

```yaml
# conftest.yaml
policy:
  - policy/          # local policies
  - data/            # local data

# Remote policies (ดึงก่อน run)
# conftest pull จาก registry แล้วเก็บใน ~/.conftest/
```

---

## ขั้นตอนที่ 768: OPA Server Mode

### รัน OPA Server

```bash
# Start OPA server
opa run --server \
  --addr=:8181 \
  --log-format=json \
  --watch \
  policy/

# Test ด้วย curl
curl -X POST \
  http://localhost:8181/v1/data/main/deny \
  -H "Content-Type: application/json" \
  -d @tfplan.json

# Response:
# {
#   "result": [
#     "S3 bucket 'aws_s3_bucket.data' must be encrypted",
#     "EC2 instance 'aws_instance.web' is missing required tags"
#   ]
# }
```

### OPA Bundle สำหรับ Production

```bash
# สร้าง OPA bundle
opa build \
  --bundle \
  --output policies.tar.gz \
  policy/

# Upload ไป S3
aws s3 cp policies.tar.gz s3://mycompany-opa-policies/

# OPA server โหลดจาก bundle
opa run --server \
  --bundle s3://mycompany-opa-policies/policies.tar.gz
```

---

## ขั้นตอนที่ 769: Parsing Plan JSON ใน OPA

### ตัวอย่าง Advanced Plan JSON Parsing

```rego
# policy/terraform_helpers.rego
package terraform

import future.keywords.if
import future.keywords.in

# Helper: ดึง resources ตาม type
resources_by_type(resource_type) := resources if {
  resources := [rc |
    rc := input.resource_changes[_]
    rc.type == resource_type
  ]
}

# Helper: ดึง resources ที่กำลัง create
creating_resources(resource_type) := resources if {
  resources := [rc |
    rc := input.resource_changes[_]
    rc.type == resource_type
    "create" in rc.change.actions
  ]
}

# Helper: ดึง resources ที่กำลัง destroy
destroying_resources(resource_type) := resources if {
  resources := [rc |
    rc := input.resource_changes[_]
    rc.type == resource_type
    "delete" in rc.change.actions
  ]
}

# Helper: ดึง tag value
get_tag(rc, tag_name) := value if {
  value := rc.change.after.tags[tag_name]
} else := "" if {
  true
}

# Helper: ตรวจสอบว่า resource อยู่ใน module
in_module(rc) if {
  rc.module_address != null
  rc.module_address != ""
}

# Helper: ดึง module path
module_path(rc) := path if {
  in_module(rc)
  path := rc.module_address
} else := "root"

# Helper: ตรวจสอบ CIDR
is_public_cidr(cidr) if {
  cidr == "0.0.0.0/0"
}

is_public_cidr(cidr) if {
  cidr == "::/0"
}
```

### ใช้ Helpers ใน Policies

```rego
# policy/aws/comprehensive.rego
package main

import data.terraform as tf
import future.keywords.contains
import future.keywords.if

# ใช้ helper functions
deny contains msg if {
  instances := tf.creating_resources("aws_instance")
  instance := instances[_]
  instance_type := instance.change.after.instance_type
  not startswith(instance_type, "t3")
  msg := sprintf("Only t3 instances allowed, got: %s", [instance_type])
}

deny contains msg if {
  volumes := tf.creating_resources("aws_ebs_volume")
  vol := volumes[_]
  not vol.change.after.encrypted == true
  msg := sprintf("EBS volume %s must be encrypted", [vol.address])
}
```

---

## ขั้นตอนที่ 770: Complete OPA Example

### Full Terraform + OPA Setup

```bash
# Directory structure
project/
├── main.tf
├── modules/
│   └── networking/
├── policy/
│   ├── aws/
│   │   ├── s3.rego
│   │   ├── ec2.rego
│   │   ├── iam.rego
│   │   └── networking.rego
│   ├── general/
│   │   ├── tags.rego
│   │   └── naming.rego
│   └── helpers/
│       └── terraform.rego
├── .github/
│   └── workflows/
│       └── terraform.yml
└── Makefile
```

```makefile
# Makefile
.PHONY: policy-check plan apply test

plan:
	terraform init
	terraform plan -out=tfplan.binary
	terraform show -json tfplan.binary > tfplan.json

policy-check: plan
	conftest test tfplan.json \
		--policy policy/ \
		--namespace main \
		--output table

apply: policy-check
	terraform apply tfplan.binary

test-policies:
	conftest verify --policy policy/

clean:
	rm -f tfplan.binary tfplan.json
```

### ตัวอย่าง Policy Output

```
$ conftest test tfplan.json --policy policy/ --output table

+---------+------+-------------------------------------------------------+
| Result  | File | Message                                               |
+---------+------+-------------------------------------------------------+
| failure |      | EBS volume 'aws_ebs_volume.data' must be encrypted    |
| failure |      | Resource 'aws_instance.web' missing tag 'CostCenter'  |
| warning |      | VPC 'aws_vpc.main' does not have VPC Flow Logs        |
| warning |      | Instance 'aws_instance.app' uses t3.xlarge             |
+---------+------+-------------------------------------------------------+

2 tests, 2 passed, 0 warnings, 2 failures
```

---

## สรุป (Summary)

OPA + conftest ให้:

1. **Open Source** - ฟรี ไม่ต้อง TFC/TFE Plus
2. **Flexible** - ใช้กับ Kubernetes, Docker, CI/CD ได้ด้วย
3. **Rego Language** - powerful query language
4. **CI/CD Integration** - รัน early ก่อน apply
5. **Custom Policies** - เขียนเองได้ตามต้องการ

| Command | Usage |
|---------|-------|
| `conftest test` | รัน policy tests |
| `conftest verify` | ตรวจสอบ policy syntax |
| `conftest pull` | ดึง policies จาก registry |
| `conftest push` | push policies ไป registry |

---

*จบ Part 077 - ในส่วนถัดไปจะเรียนรู้เรื่อง Terraform Performance & Scale*
