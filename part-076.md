# Part 076: Sentinel Policy as Code (ขั้นตอนที่ 751-760)

## บทนำ (Introduction)

Sentinel เป็น Policy as Code framework ของ HashiCorp
ช่วยให้องค์กรสามารถกำหนด governance policies ที่ enforce ได้อัตโนมัติ
ใน Terraform Cloud/Enterprise

---

## ขั้นตอนที่ 751: What is Sentinel?

### Sentinel ทำงานอย่างไร?

```
┌─────────────────────────────────────────────────────┐
│                  Terraform Run Flow                  │
│                                                     │
│  Plan → Cost Estimation → Sentinel Policies → Apply │
│                              │                      │
│                    ┌─────────┴──────────┐           │
│                    │  Policy Results:   │           │
│                    │  ✓ Pass → Apply    │           │
│                    │  ✗ Fail → Block    │           │
│                    │  ⚠ Advisory → Warn │           │
│                    └────────────────────┘           │
└─────────────────────────────────────────────────────┘
```

### Sentinel vs OPA vs Manual Review

| Aspect | Sentinel | OPA | Manual Review |
|--------|----------|-----|---------------|
| Language | Sentinel | Rego | N/A |
| Integration | TFC/TFE native | Requires setup | Human |
| Speed | Automatic | Automatic | Slow |
| Consistency | 100% | 100% | Variable |
| Cost | Plus+ plan | Free | Time cost |

---

## ขั้นตอนที่ 752: Sentinel Language Basics

### ไวยากรณ์พื้นฐาน

```python
# sentinel basics

# 1. Import
import "tfplan/v2" as tfplan
import "strings"

# 2. Variables
environment = "production"
allowed_regions = ["us-east-1", "us-west-2", "eu-west-1"]

# 3. Conditionals
is_production = environment is "production"

# 4. Functions
is_allowed_region = func(region) {
  return region in allowed_regions
}

# 5. Filter
resources_in_wrong_region = filter tfplan.resource_changes as _, rc {
  not is_allowed_region(rc.change.after.region)
}

# 6. Map
resource_names = map tfplan.resource_changes as _, rc {
  rc.address
}

# 7. All / Any
all_encrypted = all tfplan.resource_changes as _, rc {
  rc.type is "aws_ebs_volume" implies rc.change.after.encrypted is true
}

any_public = any tfplan.resource_changes as _, rc {
  rc.type is "aws_s3_bucket" and
  rc.change.after.acl is "public-read"
}

# 8. Main rule (required)
main = rule {
  all_encrypted and not any_public
}
```

### Sentinel Types

```python
# String
name = "production"
greeting = "Hello, " + name

# Number
max_instances = 10
current = 5
within_limit = current <= max_instances

# Boolean
is_encrypted = true

# List
allowed_types = ["t3.micro", "t3.small", "t3.medium"]
first_type = allowed_types[0]

# Map
config = {
  "region": "us-east-1",
  "env":    "prod",
}
region = config["region"]

# Null check
has_value = config["key"] is not null

# Type checking
is_string = config["region"] is string
```

---

## ขั้นตอนที่ 753: Sentinel ใน Terraform Cloud/Enterprise

### Policy Set Structure

```
policy-sets/
├── global/                    # บังคับใช้กับทุก workspace
│   ├── sentinel.hcl
│   ├── require-tags.sentinel
│   ├── no-public-s3.sentinel
│   └── approved-regions.sentinel
├── production/                # บังคับใช้กับ production workspaces
│   ├── sentinel.hcl
│   ├── require-encryption.sentinel
│   ├── restrict-instance-types.sentinel
│   └── cost-limit.sentinel
└── security/                  # สำหรับ security team review
    ├── sentinel.hcl
    └── iam-permissions.sentinel
```

### sentinel.hcl configuration

```hcl
# global/sentinel.hcl

# กำหนด parameters สำหรับ policies
module "tfplan-functions" {
  source = "./common-functions/tfplan-functions.sentinel"
}

# กำหนด policies
policy "require-mandatory-tags" {
  source            = "./require-tags.sentinel"
  enforcement_level = "soft-mandatory"
}

policy "no-public-s3-buckets" {
  source            = "./no-public-s3.sentinel"
  enforcement_level = "hard-mandatory"
}

policy "approved-regions-only" {
  source            = "./approved-regions.sentinel"
  enforcement_level = "advisory"

  params = {
    allowed_regions = ["us-east-1", "us-west-2"]
  }
}
```

---

## ขั้นตอนที่ 754: Sentinel Imports สำหรับ Terraform

### tfplan/v2 - ข้อมูล plan ที่กำลังจะ apply

```python
import "tfplan/v2" as tfplan

# โครงสร้างข้อมูล:
# tfplan.variables        - Input variables
# tfplan.resource_changes - Resources ที่จะเปลี่ยนแปลง
# tfplan.output_changes   - Outputs ที่จะเปลี่ยน

# resource_changes แต่ละ entry มี:
# .address         - full resource address
# .module_address  - module address
# .type            - resource type
# .name            - resource name
# .change.actions  - ["create", "update", "delete", "no-op"]
# .change.before   - current state
# .change.after    - planned state
# .change.after_unknown - unknown values (ยังไม่รู้)

# ตัวอย่าง:
all_resources = tfplan.resource_changes

# filter เฉพาะที่กำลังจะ create
new_resources = filter all_resources as _, rc {
  rc.change.actions contains "create"
}

# filter เฉพาะ aws_instance
ec2_instances = filter all_resources as _, rc {
  rc.type is "aws_instance"
}

# อ่านค่า attribute
instance_type = ec2_instances["aws_instance.web"].change.after.instance_type
```

### tfstate/v2 - ข้อมูล current state

```python
import "tfstate/v2" as tfstate

# ตรวจสอบ existing resources
all_s3_buckets = filter tfstate.resources as _, r {
  r.type is "aws_s3_bucket"
}

# ดึงค่าจาก state
bucket_names = map all_s3_buckets as _, bucket {
  bucket.values.bucket
}
```

### tfconfig/v2 - ข้อมูล configuration

```python
import "tfconfig/v2" as tfconfig

# ดู module calls
module_calls = tfconfig.module_calls

# ตรวจสอบว่า specific module ถูกใช้
uses_approved_vpc_module = any module_calls as _, call {
  call.source is "app.terraform.io/mycompany/vpc/aws"
}

# ตรวจสอบ variable definitions
variables = tfconfig.variables

# ดู resource configurations
resources = tfconfig.resources
```

### tfrun - ข้อมูล run context

```python
import "tfrun"

# ข้อมูล workspace
workspace_name = tfrun.workspace.name
org_name       = tfrun.organization.name

# ข้อมูล run
is_destroy = tfrun.is_destroy
message    = tfrun.message

# ข้อมูล cost (ถ้า cost estimation enabled)
proposed_monthly_cost = tfrun.cost_estimate.proposed_monthly_cost

# ตรวจสอบ workspace tags
workspace_tags = tfrun.workspace.tags
is_production  = "production" in workspace_tags
```

---

## ขั้นตอนที่ 755: Common Policy Patterns

### Pattern 1: Require Mandatory Tags

```python
# policies/require-mandatory-tags.sentinel

import "tfplan/v2" as tfplan
import "strings"

# กำหนด required tags
required_tags = [
  "Environment",
  "Project",
  "Owner",
  "CostCenter",
]

# Resource types ที่ต้องมี tags
taggable_resource_types = [
  "aws_instance",
  "aws_s3_bucket",
  "aws_vpc",
  "aws_subnet",
  "aws_security_group",
  "aws_db_instance",
  "aws_elasticache_cluster",
]

# ดึง resources ที่จะ create/update และต้องมี tags
all_taggable_resources = filter tfplan.resource_changes as _, rc {
  rc.type in taggable_resource_types and
  (rc.change.actions contains "create" or
   rc.change.actions contains "update")
}

# ตรวจสอบว่ามีครบทุก required tag
resources_missing_tags = filter all_taggable_resources as address, rc {
  tags = rc.change.after.tags
  
  any required_tags as tag {
    tags is null or
    tags[tag] is null or
    strings.trim(tags[tag], " ") is ""
  }
}

# แสดง error message ที่มีประโยชน์
violations = map resources_missing_tags as address, rc {
  tags = rc.change.after.tags else {}
  
  missing = filter required_tags as tag {
    tags[tag] is null or strings.trim(tags[tag] else "", " ") is ""
  }
  
  address + " is missing required tags: " + strings.join(missing, ", ")
}

if length(violations) > 0 {
  print("Tag violations found:")
  for violations as v {
    print("  -", v)
  }
}

main = rule {
  length(resources_missing_tags) == 0
}
```

### Pattern 2: Restrict Instance Types

```python
# policies/restrict-instance-types.sentinel

import "tfplan/v2" as tfplan

# กำหนด allowed instance types
allowed_instance_types = [
  "t3.nano",
  "t3.micro",
  "t3.small",
  "t3.medium",
  "t3.large",
  "t3.xlarge",
  "m5.large",
  "m5.xlarge",
  "m5.2xlarge",
]

# สำหรับ production อาจอนุญาตขนาดใหญ่กว่า
# import "tfrun"
# is_production = "production" in tfrun.workspace.tags
# production_allowed = ["m5.4xlarge", "c5.4xlarge"]

all_new_ec2 = filter tfplan.resource_changes as _, rc {
  rc.type is "aws_instance" and
  rc.change.actions contains "create"
}

violating_instances = filter all_new_ec2 as _, instance {
  not (instance.change.after.instance_type in allowed_instance_types)
}

if length(violating_instances) > 0 {
  print("The following instances use disallowed types:")
  for violating_instances as _, instance {
    print("  -", instance.address, 
          "uses", instance.change.after.instance_type,
          "(allowed:", allowed_instance_types, ")")
  }
}

main = rule {
  length(violating_instances) == 0
}
```

### Pattern 3: Require EBS Encryption

```python
# policies/require-ebs-encryption.sentinel

import "tfplan/v2" as tfplan

# ตรวจสอบ EBS volumes ที่สร้างใหม่
all_new_ebs = filter tfplan.resource_changes as _, rc {
  rc.type is "aws_ebs_volume" and
  rc.change.actions contains "create"
}

# ตรวจสอบ EC2 instances ที่มี root block device
all_new_ec2 = filter tfplan.resource_changes as _, rc {
  rc.type is "aws_instance" and
  rc.change.actions contains "create"
}

# EBS volumes ที่ไม่ได้ encrypt
unencrypted_ebs = filter all_new_ebs as _, vol {
  vol.change.after.encrypted is not true
}

# EC2 instances ที่มี root volume ไม่ encrypt
ec2_unencrypted_root = filter all_new_ec2 as _, instance {
  root = instance.change.after.root_block_device
  root is not null and
  length(root) > 0 and
  root[0].encrypted is not true
}

# EC2 instances ที่มี additional volumes ไม่ encrypt
ec2_unencrypted_ebs = filter all_new_ec2 as _, instance {
  ebs_blocks = instance.change.after.ebs_block_device else []
  
  any ebs_blocks as block {
    block.encrypted is not true
  }
}

if length(unencrypted_ebs) > 0 {
  print("Unencrypted EBS volumes:")
  for unencrypted_ebs as address, _ {
    print(" -", address)
  }
}

if length(ec2_unencrypted_root) > 0 {
  print("EC2 instances with unencrypted root volumes:")
  for ec2_unencrypted_root as address, _ {
    print(" -", address)
  }
}

main = rule {
  length(unencrypted_ebs) == 0 and
  length(ec2_unencrypted_root) == 0 and
  length(ec2_unencrypted_ebs) == 0
}
```

### Pattern 4: Deny Public S3 Buckets

```python
# policies/deny-public-s3.sentinel

import "tfplan/v2" as tfplan

# ACLs ที่ถือว่า public
public_acls = [
  "public-read",
  "public-read-write",
  "authenticated-read",
]

# ตรวจสอบ S3 bucket ACL resource
s3_acl_changes = filter tfplan.resource_changes as _, rc {
  rc.type is "aws_s3_bucket_acl" and
  (rc.change.actions contains "create" or
   rc.change.actions contains "update")
}

public_acl_violations = filter s3_acl_changes as _, acl {
  acl.change.after.acl in public_acls
}

# ตรวจสอบ Public Access Block (ต้องมี)
s3_public_block_changes = filter tfplan.resource_changes as _, rc {
  rc.type is "aws_s3_bucket_public_access_block" and
  rc.change.actions contains "create"
}

# ตรวจสอบว่า public access block ถูก configure ถูกต้อง
misconfigured_public_access_blocks = filter s3_public_block_changes as _, block {
  block.change.after.block_public_acls is not true or
  block.change.after.block_public_policy is not true or
  block.change.after.ignore_public_acls is not true or
  block.change.after.restrict_public_buckets is not true
}

if length(public_acl_violations) > 0 {
  print("Public S3 ACLs detected (not allowed):")
  for public_acl_violations as address, _ {
    print(" -", address)
  }
}

if length(misconfigured_public_access_blocks) > 0 {
  print("S3 Public Access Blocks not fully configured:")
  for misconfigured_public_access_blocks as address, _ {
    print(" -", address, "(all 4 settings must be true)")
  }
}

main = rule {
  length(public_acl_violations) == 0 and
  length(misconfigured_public_access_blocks) == 0
}
```

### Pattern 5: Cost Limit Enforcement

```python
# policies/cost-limit.sentinel

import "tfrun"

# กำหนด cost limits
max_monthly_increase = 500.0  # $500 per month

# ดึง cost estimate
proposed_cost = tfrun.cost_estimate.proposed_monthly_cost
prior_cost    = tfrun.cost_estimate.prior_monthly_cost
cost_delta    = tfrun.cost_estimate.delta_monthly_cost

# ตรวจสอบ cost delta
within_budget = float(cost_delta) <= max_monthly_increase

if not within_budget {
  print("Cost limit exceeded!")
  print("  Current monthly cost:  $" + string(prior_cost))
  print("  Proposed monthly cost: $" + string(proposed_cost))
  print("  Monthly cost increase: $" + string(cost_delta))
  print("  Maximum allowed:       $" + string(max_monthly_increase))
  print("  Contact FinOps team for approval: finops@mycompany.com")
}

main = rule {
  within_budget
}
```

### Pattern 6: Require MFA for IAM Users

```python
# policies/iam-mfa-required.sentinel

import "tfplan/v2" as tfplan

# ดึง IAM user changes
all_iam_users = filter tfplan.resource_changes as _, rc {
  rc.type is "aws_iam_user" and
  rc.change.actions contains "create"
}

# ดึง IAM policies
all_iam_policies = filter tfplan.resource_changes as _, rc {
  rc.type is "aws_iam_user_policy" or
  rc.type is "aws_iam_user_policy_attachment"
}

# ตรวจสอบว่ามี MFA policy attached
users_without_mfa_policy = filter all_iam_users as _, user {
  username = user.change.after.name
  
  # ตรวจสอบว่ามี policy ที่ require MFA
  not any all_iam_policies as _, policy {
    (policy.change.after.user is username) and
    (
      "force-mfa" in (policy.change.after.policy_arn else "") or
      "require-mfa" in (policy.change.after.name else "")
    )
  }
}

main = rule {
  length(users_without_mfa_policy) == 0
}
```

### Pattern 7: Approved Regions Only

```python
# policies/approved-regions.sentinel

import "tfplan/v2" as tfplan
import "tfrun"
import "param"

# กำหนด allowed regions (สามารถ override ด้วย parameters)
default_allowed_regions = ["us-east-1", "us-west-2", "eu-west-1"]

allowed_regions = param.allowed_regions else default_allowed_regions

# ดึง resources ทั้งหมดที่มี region
resources_with_region = filter tfplan.resource_changes as _, rc {
  rc.change.after.region is not null and
  rc.change.actions contains "create"
}

# ตรวจสอบ region
out_of_region_resources = filter resources_with_region as _, rc {
  not (rc.change.after.region in allowed_regions)
}

if length(out_of_region_resources) > 0 {
  print("Resources in disallowed regions:")
  for out_of_region_resources as address, rc {
    print(" -", address, "in region", rc.change.after.region)
  }
  print("Allowed regions:", allowed_regions)
}

main = rule {
  length(out_of_region_resources) == 0
}
```

### Pattern 8: Enforce Naming Convention

```python
# policies/naming-convention.sentinel

import "tfplan/v2" as tfplan
import "strings"
import "regex"

# Naming convention: <project>-<environment>-<resource-type>-<name>
# Example: myapp-prod-ec2-webserver

naming_pattern = "^[a-z][a-z0-9-]*$"  # lowercase, alphanumeric, hyphens

# Resources ที่ต้องตรวจสอบ naming
checkable_resources = ["aws_instance", "aws_s3_bucket", "aws_vpc", "aws_subnet"]

new_resources = filter tfplan.resource_changes as _, rc {
  rc.type in checkable_resources and
  rc.change.actions contains "create"
}

# ดึง name จาก tags หรือ attribute
get_name = func(rc) {
  if rc.change.after.tags is not null and
     rc.change.after.tags["Name"] is not null {
    return rc.change.after.tags["Name"]
  } else if rc.type is "aws_s3_bucket" {
    return rc.change.after.bucket
  } else {
    return ""
  }
}

# ตรวจสอบ naming convention
naming_violations = filter new_resources as _, rc {
  name = get_name(rc)
  name is "" or not regex.match(naming_pattern, name)
}

if length(naming_violations) > 0 {
  print("Naming convention violations:")
  for naming_violations as _, rc {
    print(" -", rc.address, "name:", get_name(rc))
    print("   Expected pattern:", naming_pattern)
  }
}

main = rule {
  length(naming_violations) == 0
}
```

---

## ขั้นตอนที่ 756: Policy Testing

### สร้าง Test สำหรับ Sentinel Policies

```
policy-sets/
└── require-tags/
    ├── require-tags.sentinel
    ├── sentinel.hcl
    └── test/
        └── require-tags/
            ├── pass.json         # test ที่ควรผ่าน
            ├── fail-missing-env.json  # test ที่ควร fail
            └── fail-empty-tag.json   # test ที่ควร fail
```

```json
// test/require-tags/pass.json
{
  "mock": {
    "tfplan/v2": {
      "resource_changes": {
        "aws_instance.web": {
          "type": "aws_instance",
          "change": {
            "actions": ["create"],
            "before": null,
            "after": {
              "instance_type": "t3.micro",
              "ami": "ami-12345",
              "tags": {
                "Environment": "production",
                "Project": "myapp",
                "Owner": "platform-team",
                "CostCenter": "IT-001"
              }
            }
          }
        }
      }
    }
  },
  "test": {
    "main": true
  }
}
```

```json
// test/require-tags/fail-missing-env.json
{
  "mock": {
    "tfplan/v2": {
      "resource_changes": {
        "aws_instance.web": {
          "type": "aws_instance",
          "change": {
            "actions": ["create"],
            "before": null,
            "after": {
              "instance_type": "t3.micro",
              "tags": {
                "Project": "myapp",
                "Owner": "platform-team"
              }
            }
          }
        }
      }
    }
  },
  "test": {
    "main": false
  }
}
```

### รัน Sentinel Tests

```bash
# ติดตั้ง Sentinel CLI
wget https://releases.hashicorp.com/sentinel/0.26.3/sentinel_0.26.3_linux_amd64.zip
unzip sentinel_0.26.3_linux_amd64.zip
mv sentinel /usr/local/bin/

# รัน tests
cd policy-sets/require-tags
sentinel test

# Output:
# PASS - test/require-tags/pass.json
# PASS - test/require-tags/fail-missing-env.json

# รัน specific policy
sentinel apply require-tags.sentinel

# รัน ด้วย custom mock
sentinel apply -config=test/require-tags/pass.json require-tags.sentinel
```

---

## ขั้นตอนที่ 757: Sentinel Simulator

```bash
# Sentinel Playground - ทดสอบ policies แบบ interactive

# สร้าง mock data
cat > mock-tfplan.json << 'EOF'
{
  "resource_changes": {
    "aws_s3_bucket.sensitive": {
      "type": "aws_s3_bucket",
      "change": {
        "actions": ["create"],
        "after": {
          "bucket": "my-sensitive-data",
          "acl": "public-read"
        }
      }
    }
  }
}
EOF

# Apply policy กับ mock data
sentinel apply \
  -mock="tfplan/v2=mock-tfplan.json" \
  policies/deny-public-s3.sentinel

# ใช้ -trace เพื่อ debug
sentinel apply \
  -trace \
  -mock="tfplan/v2=mock-tfplan.json" \
  policies/deny-public-s3.sentinel
```

---

## ขั้นตอนที่ 758: Pre-Written Policy Libraries

### HashiCorp Sentinel Policy Libraries

```bash
# Clone pre-written AWS policies
git clone https://github.com/hashicorp/terraform-foundational-policies-library
cd terraform-foundational-policies-library

# โครงสร้าง:
# cis/
#   aws/
#     networking/
#       aws-cis-4.1-networking-no-public-default-sg.sentinel
#       aws-cis-4.2-networking-no-public-ssh.sentinel
#     iam/
#       aws-cis-1.22-iam-mfa-policy-attached.sentinel
#     storage/
#       aws-cis-2.1-storage-s3-no-public.sentinel
```

### ใช้ Community Policies

```hcl
# sentinel.hcl สำหรับใช้ pre-written policies

policy "aws-cis-no-public-ssh" {
  source            = "https://raw.githubusercontent.com/hashicorp/terraform-foundational-policies-library/main/cis/aws/networking/aws-cis-4.2-networking-no-public-ssh.sentinel"
  enforcement_level = "hard-mandatory"
}

policy "aws-cis-s3-no-public" {
  source            = "https://raw.githubusercontent.com/hashicorp/terraform-foundational-policies-library/main/cis/aws/storage/aws-cis-2.1-storage-s3-no-public.sentinel"
  enforcement_level = "hard-mandatory"
}
```

---

## ขั้นตอนที่ 759: Complete Policy Set Example

### Production Policy Set

```python
# policies/production/main.sentinel
# Master policy ที่เรียก sub-policies

import "module" as module

# ตรวจสอบทุก policy
tagging_ok   = module.require_tags.main
encrypt_ok   = module.require_encryption.main
region_ok    = module.approved_regions.main
cost_ok      = module.cost_limit.main
s3_ok        = module.no_public_s3.main
instance_ok  = module.instance_types.main

# Main rule
main = rule {
  tagging_ok and
  encrypt_ok and
  region_ok  and
  cost_ok    and
  s3_ok      and
  instance_ok
}
```

```hcl
# policies/production/sentinel.hcl

# ใช้ common functions module
module "common-functions" {
  source = "./common-functions/common-functions.sentinel"
}

module "aws-functions" {
  source = "./aws-functions/aws-functions.sentinel"
}

policy "require-tags" {
  source            = "./require-tags.sentinel"
  enforcement_level = "hard-mandatory"
}

policy "require-encryption" {
  source            = "./require-encryption.sentinel"
  enforcement_level = "hard-mandatory"
}

policy "approved-regions" {
  source            = "./approved-regions.sentinel"
  enforcement_level = "soft-mandatory"

  params = {
    allowed_regions = ["us-east-1", "us-west-2"]
  }
}

policy "cost-limit" {
  source            = "./cost-limit.sentinel"
  enforcement_level = "soft-mandatory"

  params = {
    max_monthly_increase = 1000.0
  }
}

policy "no-public-s3" {
  source            = "./no-public-s3.sentinel"
  enforcement_level = "hard-mandatory"
}

policy "restrict-instance-types" {
  source            = "./instance-types.sentinel"
  enforcement_level = "soft-mandatory"

  params = {
    allowed_instance_families = ["t3", "m5", "c5"]
  }
}
```

---

## ขั้นตอนที่ 760: Policy as Code Best Practices

### Best Practices

```python
# 1. ใช้ functions เพื่อ reuse code

# common-functions.sentinel
get_tags = func(resource) {
  return resource.change.after.tags else {}
}

has_required_tag = func(resource, tag_name) {
  tags = get_tags(resource)
  return tags[tag_name] is not null and tags[tag_name] is not ""
}

# ใช้ใน policy อื่น:
import "module" as common
tags_ok = common.has_required_tag(instance, "Environment")

# 2. กำหนด parameters แทน hard-coded values
# sentinel.hcl
policy "approved-regions" {
  params = {
    allowed_regions = ["us-east-1", "us-west-2"]
  }
}

# 3. สร้าง helpful error messages
if length(violations) > 0 {
  print("=== Policy Violation: Encryption Required ===")
  print("The following resources must be encrypted:")
  for violations as address, _ {
    print("  -", address)
  }
  print("")
  print("To fix: add 'encrypted = true' to the resource")
  print("Documentation: https://wiki.mycompany.com/terraform-policies")
}

# 4. Test ทุก policy
# มี pass tests และ fail tests

# 5. Version control policies
# git tag v1.0.0
# TFC ใช้ specific version
```

---

## สรุป (Summary)

Sentinel ช่วยให้:

1. **Governance at Scale** - enforce policies อัตโนมัติทุก deployment
2. **Compliance** - CIS, SOC2, PCI-DSS requirements as code
3. **Cost Control** - block deployments ที่เกิน budget
4. **Security** - ป้องกัน misconfiguration
5. **Auditability** - ทุก policy check มี record

| Enforcement Level | Behavior |
|------------------|----------|
| Advisory | แสดง warning แต่ไม่ block |
| Soft-Mandatory | Block แต่ owner override ได้ |
| Hard-Mandatory | Block เด็ดขาด |

---

*จบ Part 076 - ในส่วนถัดไปจะเรียนรู้เรื่อง OPA (Open Policy Agent) กับ Terraform*
